# Blitzy Project Guide — SimpleLogin First-Run / Initialization Q&A Investigation

> Branch: `app_2cd6ee777f8c` · Deliverable: `blitzy/documentation/app_2cd6ee777f8c.md`
> Brand legend — **Completed / AI Work:** Dark Blue `#5B39F3` · **Remaining / Not Completed:** White `#FFFFFF` · **Headings / Accents:** Violet-Black `#B23AF2` · **Highlight:** Mint `#A8FDD9`

---

## 1. Executive Summary

### 1.1 Project Overview

This project is a **read-only investigative Q&A task** against the SimpleLogin email-aliasing application (a Flask full-stack service). The objective was to **empirically answer five first-run / initialization questions by actually building and running the relevant code paths** on the project's canonical stack, then record grounded answers in a single new markdown document. Target users are the engineers and reviewers who need authoritative, reproducible answers about migrations, web-server readiness, the SMTP handler, registration/activation, and dynamic alias limits. The technical scope spans the migration subsystem, the Gunicorn web server, the aiosmtpd email handler, the API/auth layer, the data models, and the configuration module — all exercised through their real entry points. The source repository was required to remain byte-for-byte unchanged; the only permanent artifact is the answer document.

### 1.2 Completion Status

The completion percentage is calculated using the **AAP-scoped hours methodology (PA1)**: `Completed Hours ÷ Total Hours × 100 = 40 ÷ 44 = 90.9%`.

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStroke':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeWidth':'2px','pieTitleTextSize':'16px','pieSectionTextSize':'14px'}}}%%
pie showData
    title Completion — 90.9% Complete (40h of 44h)
    "Completed Work (AI) — #5B39F3" : 40
    "Remaining Work — #FFFFFF" : 4
```

| Metric | Hours |
|--------|-------|
| **Total Hours** | **44** |
| **Completed Hours (AI + Manual)** | **40** (AI: 40 · Manual: 0) |
| **Remaining Hours** | **4** |
| **Percent Complete** | **90.9%** |

### 1.3 Key Accomplishments

- ✅ Sole AAP deliverable authored: `blitzy/documentation/app_2cd6ee777f8c.md` (1,717 lines), one grounded section per question (Q1–Q5).
- ✅ **All five answers empirically reproduced on the canonical stack** — PostgreSQL 13.23 (`postgres:13`), Redis 6.2.22 (`redis:6`), Python 3.10.18, Gunicorn 20.0.4 — with **zero discrepancies**.
- ✅ Q1 — Migrations on an empty database: **77 tables**, last created = **`user_audit_log`**, stable across two independent reset→migrate runs.
- ✅ Q2 — Web-server readiness line **`Listening at: http://0.0.0.0:7777 (<pid>)`**, first→ready delta **≈ 0.20 ms** (0.187–0.231 ms) stable across four runs via an external millisecond timestamper.
- ✅ Q3 — Email handler on custom port confirms **`Listen for port 25025`** (INFO) and **`Start mail controller 0.0.0.0 25025`** (DEBUG).
- ✅ Q4 — Pre-activation login returns **`{"error":"Account not activated"}`**, HTTP **422**, with DB columns **`activated=f`, `notification=t`**.
- ✅ Q5 — `max_alias_free_plan` transitions **5 → 10** for **both** users after the restart, proving the limit is **global-live** (computed per request).
- ✅ Every factual claim is grounded with `file:line` references and complete unedited command output; a coverage checklist confirms every named sub-part.
- ✅ **Source repository left byte-for-byte unchanged** — `git diff` vs base shows exactly one added file; all ephemeral artifacts removed.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| _None._ All five answers were empirically reproduced (100% match); no compilation, test, or runtime failures; no blocking issues. | — | — | — |

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-----------------|----------------|-------------------|-------------------|-------|
| Source repository | Read/Write (git) | Full access; branch `blitzy-4ed19b60-2d4f-4271-8e00-f1af427ee793` checked out, working tree clean. | ✅ No issue | — |
| Canonical runtime (Docker `postgres:13`, `redis:6`, app image) | Container runtime | Docker Engine available; canonical stack stood up and torn down successfully during validation. | ✅ No issue | — |
| DNS / MX for `example.com` (Q4) | Network resolution | Anticipated MX-lookup blocker did **not** manifest on the canonical stack; reported honestly in the deliverable. | ✅ No issue | — |

_No access issues identified that block review or merge._

### 1.6 Recommended Next Steps

1. **[High]** Perform a technical review and sign-off of `blitzy/documentation/app_2cd6ee777f8c.md` — verify each direct answer against its embedded unedited output and spot-check the `file:line` grounding.
2. **[Medium]** Confirm source integrity (`git diff` still shows only the answer document) and approve/merge the PR to the target branch.
3. **[Low]** Optionally reproduce one or two code paths (e.g., Q1 migration count, Q4 login) on the canonical stack to independently confirm reproducibility.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

Every completed component traces to a specific AAP requirement or a required investigation activity. **Total = 40 h** (matches Completed Hours in §1.2).

| Component | Hours | Description |
|-----------|-------|-------------|
| Canonical runtime environment stand-up | 5 | Provision Python 3.10.18, PostgreSQL 13.23 (`postgres:13`), Redis 6.2.22 (`redis:6`); install pinned deps from `poetry.lock`; author ephemeral `.env` satisfying every var `app/config.py` reads at import; verify observed versions match CI. |
| Q1 — Migration / table-count investigation | 4 | Reset `public` schema, `alembic upgrade head` (255 steps) with SQL echo, `information_schema` count (=77), identify last `CREATE TABLE` (`user_audit_log`) from the migration chain; run twice; drop-cascade object analysis. |
| Q2 — Web-server readiness investigation | 4 | Launch canonical Gunicorn entry point, wrap stderr with an external millisecond timestamper, capture ready line + first→ready delta across four runs; web research to confirm Gunicorn 20.0.4 log-format. |
| Q3 — Email-handler custom-port investigation | 2 | Run `email_handler.py --port 25025`, capture the two startup log lines that echo the port. |
| Q4 — Registration / pre-activation-login investigation | 4 | Register `testuser@example.com`, attempt login before activation, capture full curl output + HTTP 422 + JSON error; direct DB query for `activated`/`notification`; in-process analysis of the MX-lookup gate. |
| Q5 — Dynamic alias-limit investigation | 5 | Full register→activate→login→`/api/user_info` flow; read `max_alias_free_plan` (5); set limit to 10, restart, create a second user, re-read for both; determine global-live vs per-user. |
| Answer document authoring | 8 | Author `app_2cd6ee777f8c.md` (1,717 lines): per-question Direct answer + Exact commands + Complete unedited output + `file:line` grounding + Rationale; environment preamble; coverage checklist; Appendix A (both full migration logs). |
| Code-review response iterations | 4 | Two rounds: commit `a4018549` addressed 7 code-review findings (+1,383/−362); commit `6c6f34fb` fixed the Q1 drop-cascade count and removed a non-deterministic metric. |
| Final empirical re-validation | 3 | Re-stand-up the canonical stack, re-run all five code paths through their real entry points, confirm 100% match; fixed a harness ordering bug (Alembic before CONFIG export). |
| Cleanup + repository integrity verification | 1 | Remove all ephemeral artifacts; verify byte-for-byte source integrity via `git status`/`git diff`. |
| **Total** | **40** | |

### 2.2 Remaining Work Detail

Every remaining item is a **path-to-production** gate for a documentation deliverable (there is no application deploy / CI-CD surface). **Total = 4 h** (matches Remaining Hours in §1.2 and the pie chart in §7).

| Category | Hours | Priority |
|----------|-------|----------|
| Human technical review & sign-off of the answer document | 3 | High |
| PR approval & merge to target branch + branch close-out | 1 | Medium |
| **Total** | **4** | |

### 2.3 Hours Reconciliation

| Check | Result |
|-------|--------|
| §2.1 Completed total | 40 h |
| §2.2 Remaining total | 4 h |
| §2.1 + §2.2 = §1.2 Total | 40 + 4 = **44 h** ✅ |
| Completion % = 40 ÷ 44 | **90.9%** ✅ |
| §1.2 ↔ §2.2 ↔ §7 remaining hours identical | **4 h** in all three ✅ |

---

## 3. Test Results

This is a **documentation Q&A deliverable (markdown) with no associated unit tests**. Per the read-only AAP, no source code was changed, so the repository's own pytest suite is explicitly **out of scope**. The substantive validation performed by Blitzy's autonomous systems is the **empirical reproduction of all five runtime answers** through their real entry points on the canonical stack. All results below originate from Blitzy's autonomous validation logs for this project.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|------------|-------|
| Empirical reproduction — Q1 (migrations on empty PG) | Alembic + psql (`information_schema`) | 1 | 1 | 0 | n/a | 255-step `upgrade head`; count **77** stable across **2** reset→migrate runs; last table `user_audit_log`. |
| Empirical reproduction — Q2 (web-server readiness) | Gunicorn 20.0.4 + ms timestamper | 1 | 1 | 0 | n/a | Ready line confirmed; first→ready **0.187–0.231 ms** stable across **4** runs. |
| Empirical reproduction — Q3 (email handler port) | aiosmtpd `Controller` | 1 | 1 | 0 | n/a | `Listen for port 25025` + `Start mail controller 0.0.0.0 25025` — port echoed verbatim. |
| Empirical reproduction — Q4 (register + pre-activation login) | Flask API + curl + psql | 1 | 1 | 0 | n/a | Register `200`; login `422` `{"error":"Account not activated"}`; DB `activated=f`, `notification=t`. |
| Empirical reproduction — Q5 (dynamic alias limit) | Flask API + curl + psql | 1 | 1 | 0 | n/a | `max_alias_free_plan` 5 → 10 for **both** users after restart (global-live). |
| **Totals** | — | **5** | **5** | **0** | — | **100% pass; zero discrepancies.** |

**Validation gates (autonomous logs):** Gate 1 (validation success) — 5/5 reproduced; Gate 2 (runtime) — Gunicorn, email handler, Alembic migrations, and the full register→activate→login→user_info flow all ran through real entry points; Gate 3 (zero unresolved errors) — every Python code path imported/ran cleanly; Gate 4 (all in-scope files validated) — the sole in-scope file (answer doc) verified accurate and complete.

---

## 4. Runtime Validation & UI Verification

This is a backend email-system Q&A task with **no UI component** and no Figma frames; runtime validation covers the exercised process entry points and API endpoints.

**Runtime health (real entry points):**
- ✅ **Operational** — Migration subsystem: `alembic upgrade head` (255 steps) → head `32f25cbf12f6`, exit 0.
- ✅ **Operational** — Web server: `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` reaches `Listening at: http://0.0.0.0:7777 (<pid>)`.
- ✅ **Operational** — Email handler: `python email_handler.py --port 25025` binds and echoes the custom port.
- ✅ **Operational** — Database: PostgreSQL 13.23; `information_schema` reports 77 tables post-migration.
- ✅ **Operational** — Redis 6.2.22: backs Flask-Limiter and caching.

**API integration outcomes:**
- ✅ **Operational** — `POST /api/auth/register` → HTTP 200 `{"msg":"User needs to confirm their account"}`.
- ✅ **Operational** — `POST /api/auth/login` (pre-activation) → HTTP 422 `{"error":"Account not activated"}`.
- ✅ **Operational** — `POST /api/auth/activate` and `POST /api/auth/login` (device) → API key minted.
- ✅ **Operational** — `GET /api/user_info` → `max_alias_free_plan` reflects live global config (5 → 10).

**UI verification:** ⚠ **Not applicable** — no user interface in scope for this investigation.

---

## 5. Compliance & Quality Review

Cross-map of AAP deliverables and the governing **SWE-AtlasQnA-Repo** rules to their validation status.

| Deliverable / Rule (AAP) | Benchmark | Status | Progress |
|--------------------------|-----------|--------|----------|
| Single answer doc `blitzy/documentation/<branch>.md` | Exactly one CREATE | ✅ Pass | 100% |
| Source repository unchanged (read-only) | Zero source edits | ✅ Pass | 100% |
| Run-first methodology (observe, then write) | Every value from captured runtime output | ✅ Pass | 100% |
| Canonical build & invocation | Default config; exact commands stated | ✅ Pass | 100% |
| Q1 answered (count + last table) | 77 + `user_audit_log`, observed | ✅ Pass | 100% |
| Q2 answered (ready line + ms, ≥2 runs) | Ready line + 0.187–0.231 ms over 4 runs | ✅ Pass | 100% |
| Q3 answered (listening confirmation + exact msg) | Both log lines w/ port 25025 | ✅ Pass | 100% |
| Q4 answered (JSON + 422 + curl + DB cols) | `{"error":"Account not activated"}`, 422, `f`/`t` | ✅ Pass | 100% |
| Q5 answered (before/after + both-vs-only) | 5 → 10 for both users (global-live) | ✅ Pass | 100% |
| Complete unedited output + `file:line` grounding per Q | No elision; exact commands included | ✅ Pass | 100% |
| Coverage pass over every named sub-part | Q1a/b, Q2a/b, Q3a/b, Q4a/b/c/d, Q5 before/after/both | ✅ Pass | 100% |
| Non-canonical values labeled | MX-blocker reported honestly (did not fire) | ✅ Pass | 100% |
| Ephemeral artifacts removed | `.env` + scripts deleted; repo clean | ✅ Pass | 100% |
| Human technical review & sign-off | Peer review of deliverable | ⬜ Pending | 0% |
| PR approval & merge | Merge to target branch | ⬜ Pending | 0% |

**Fixes applied during autonomous validation:** 7 code-review findings addressed (commit `a4018549`); Q1 drop-cascade object count corrected 81 → 82 and a non-deterministic log-line metric removed (commit `6c6f34fb`); validation-harness ordering bug fixed (CONFIG exported before Alembic). **Outstanding:** human review + merge only.

---

## 6. Risk Assessment

Risk profile is **Low** overall — a read-only documentation task with no source changes, no new dependencies, and no deployment surface.

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| R1 — A documented answer contains a subtle factual error | Technical | High | Low | All 5 answers empirically reproduced 100% by the Final Validator + independent `file:line` re-verification by the assessor; human sign-off is the final gate. | Mitigated (review pending) |
| R2 — Q2 first→ready delta is sub-millisecond and hardware/environment-specific | Technical / Operational | Low | Medium | Reported as a range (0.187–0.231 ms) across 4 runs on the canonical stack; framed as environment-relative, not an absolute constant. | Accepted / Documented |
| R3 — Reproducibility depends on the canonical stack + Docker | Operational | Low | Medium | Exact observed versions + canonical CI references + copy-pasteable commands are in the deliverable; apt-installed PostgreSQL 13 noted as a Docker fallback. | Documented |
| R4 — Q4 MX-lookup behavior may differ in a fully offline / no-DNS environment | Integration / Operational | Low | Low | Observed behavior reported honestly (blocker did not fire; `example.com` MX non-empty `['']`); `SKIP_MX_LOOKUP_ON_CHECK` verified `False`; no non-canonical workaround used. | Accepted / Documented |
| R5 — Synthetic test credentials (`testpass123`) appear in the doc | Security | Low | Low | AAP-mandated verbatim test inputs, not real secrets; ephemeral `.env` removed during cleanup. | Accepted |
| R6 — No unit tests accompany the markdown deliverable | Technical / Quality | Low | n/a | Substantive validation is empirical reproduction of all 5 runtime answers (100%); repo pytest suite explicitly out of scope per the read-only AAP. | Accepted (by design) |

No High-probability or unmitigated risks. No security vulnerabilities introduced (zero new code, dependencies, or endpoints).

---

## 7. Visual Project Status

**Project hours breakdown** (`Completed Work` = Completed Hours in §1.2; `Remaining Work` = Remaining Hours in §1.2 and the sum of §2.2):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStroke':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeWidth':'2px','pieTitleTextSize':'16px','pieSectionTextSize':'14px'}}}%%
pie showData
    title Project Hours — 44h total (90.9% complete)
    "Completed Work — #5B39F3" : 40
    "Remaining Work — #FFFFFF" : 4
```

**Remaining work by priority** (from §2.2; sums to 4 h):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#B23AF2','pie2':'#A8FDD9','pieStroke':'#5B39F3','pieStrokeWidth':'2px','pieTitleTextSize':'16px','pieSectionTextSize':'14px'}}}%%
pie showData
    title Remaining Hours by Priority (4h)
    "High — Review & sign-off" : 3
    "Medium — PR approval & merge" : 1
```

**Remaining hours per category (bar view):**

| Category | Hours | Bar |
|----------|-------|-----|
| Review & sign-off (High) | 3 | ██████████████████████████████ |
| PR approval & merge (Medium) | 1 | ██████████ |
| **Total** | **4** | |

---

## 8. Summary & Recommendations

**Achievements.** The project is **90.9% complete** (40 of 44 hours). The sole AAP deliverable — `blitzy/documentation/app_2cd6ee777f8c.md` — is fully authored (1,717 lines) and every one of its five answers was **empirically reproduced on the canonical stack with zero discrepancies**. Each answer leads with a direct response, then supplies the exact command, complete unedited output, `file:line` grounding, and rationale, and a coverage checklist confirms every named sub-part. The source repository is verifiably **byte-for-byte unchanged**.

**Remaining gaps.** The only outstanding work is the **path-to-production for a documentation deliverable**: human technical review & sign-off (3 h) and PR approval & merge (1 h) — **4 hours total**. There are no code fixes, configuration tasks, integration tasks, or deployment tasks, because this is a read-only investigation with no source changes and no runtime surface to ship.

**Critical path to production.** Review the answer document → confirm source integrity → merge the PR. No blockers exist on this path.

**Success metrics.** 5/5 questions answered and reproduced (100%); 0 discrepancies; 0 source files modified; 1 file added; all validation gates passed.

**Production-readiness assessment.** The deliverable is **production-ready pending human sign-off**. Confidence is **High**: all items are well-defined, there are no failing tests or compilation issues (a read-only task), and the top risk (a subtle answer error) is mitigated by dual validation with human review as the final gate.

| Metric | Value |
|--------|-------|
| Completion | 90.9% (40 / 44 h) |
| Questions reproduced | 5 / 5 (100%) |
| Discrepancies | 0 |
| Source files modified | 0 |
| Files added | 1 (answer document) |
| Remaining effort | 4 h (human review + merge) |

---

## 9. Development Guide

This guide covers **(A)** reviewing the deliverable and **(B)** reproducing the five investigations on the canonical stack. Every command below was validated against the repository.

### 9.1 System Prerequisites

- **Docker Engine 28.x** (used to stand up the canonical stack via official images).
- **Git + Git LFS** (repository is Git-LFS enabled).
- **~2 GB free disk** for images and the build.
- Canonical target versions (from `.github/workflows/main.yml`): **Python 3.10**, **PostgreSQL 13**, **Redis 6**.

### 9.2 Environment Setup

The application's `app/config.py` reads several variables **at import time** — the process will not start if any are missing (a missing `URL` raises `KeyError: 'URL'`). Create an ephemeral `.env` (never committed):

```bash
# run_base.env  (ephemeral — do NOT commit)
URL=http://localhost:7777
DB_URI=postgresql://test:test@localhost:5432/test
FLASK_SECRET=secret
NOT_SEND_EMAIL=true       # suppress outbound email during register/activate
DISABLE_RATE_LIMIT=1      # allow repeated login attempts (login is 10/minute)
# MAX_NB_EMAIL_FREE_PLAN is intentionally unset here -> defaults to 5

# run_after.env  (for Q5 "after" phase) — identical plus:
# MAX_NB_EMAIL_FREE_PLAN=10
```

Stand up the canonical services (official images, matching CI):

```bash
docker network create sl-canon-net
docker run -d --name sl-pg13 --network sl-canon-net \
  -e POSTGRES_USER=test -e POSTGRES_PASSWORD=test -e POSTGRES_DB=test \
  postgres:13 -c fsync=off -c full_page_writes=off
docker run -d --name sl-redis6 --network sl-canon-net redis:6
```

> **Docker unavailable?** Per the AAP, replace the `postgres:13` container with an apt-installed PostgreSQL 13 for local runtime; point `DB_URI` at it.

### 9.3 Dependency Installation

Dependencies are pinned exactly in `poetry.lock`. Reproduce the canonical Python 3.10 environment:

```bash
# Option A (canonical): use the project app image (ships the pinned venv), repo bind-mounted read-only.
# Option B (local): a Python 3.10 virtualenv with the pinned dependencies:
python3.10 -m venv .venv && source .venv/bin/activate
pip install poetry && poetry install         # resolves from poetry.lock (gunicorn 20.0.4, flask 1.1.2, alembic 1.4.3, ...)
```

### 9.4 Application Startup & Verification (per code path)

```bash
export CONFIG=run_base.env            # config.py loads this at import — export BEFORE any app/alembic command
DB="postgresql://test:test@localhost:5432/test"

# ---- Q1: migrations on an empty database -> expect 77 tables, last = user_audit_log ----
echo "drop schema public cascade; create schema public;" | psql "$DB"
alembic upgrade head                                       # 255 steps -> head 32f25cbf12f6
psql "$DB" -c "SELECT count(*) FROM information_schema.tables WHERE table_schema='public';"   # -> 77

# ---- Q2: web-server readiness -> expect "Listening at: http://0.0.0.0:7777 (<pid>)" ----
gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15        # canonical entry (Dockerfile:47)

# ---- Q3: email handler on a custom port -> expect the port echoed in two log lines ----
timeout --signal=INT 5 python email_handler.py --port 25025 ; echo "exit=$?"

# ---- Q4: register then login BEFORE activation -> expect HTTP 422 {"error":"Account not activated"} ----
curl -sS -i -X POST http://localhost:7777/api/auth/register -H 'Content-Type: application/json' \
     -d '{"email":"testuser@example.com","password":"testpass123"}'
curl -sS -i -X POST http://localhost:7777/api/auth/login    -H 'Content-Type: application/json' \
     -d '{"email":"testuser@example.com","password":"testpass123"}'
psql "$DB" -c "SELECT activated, notification FROM users WHERE email='testuser@example.com';"   # -> f | t

# ---- Q5: dynamic alias limit -> expect 5 before, 10 for BOTH users after restart ----
curl -sS -i http://localhost:7777/api/user_info -H 'Authentication: <api_key>'   # "max_alias_free_plan": 5
# set MAX_NB_EMAIL_FREE_PLAN=10 (run_after.env), restart gunicorn, create user #2, re-read both -> 10 & 10
```

### 9.5 Example Usage (full Q5 API flow)

```bash
# register -> read activation code from DB (email suppressed) -> activate -> login (device) mints api_key -> user_info
curl -sS -X POST http://localhost:7777/api/auth/register -H 'Content-Type: application/json' \
     -d '{"email":"q5user2@example.com","password":"testpass123"}'
psql "$DB" -c "SELECT aa.code FROM account_activation aa JOIN users u ON u.id=aa.user_id WHERE u.email='q5user2@example.com';"
curl -sS -X POST http://localhost:7777/api/auth/activate -H 'Content-Type: application/json' \
     -d '{"email":"q5user2@example.com","code":"<code>"}'
curl -sS -X POST http://localhost:7777/api/auth/login -H 'Content-Type: application/json' \
     -d '{"email":"q5user2@example.com","password":"testpass123","device":"cli"}'
curl -sS http://localhost:7777/api/user_info -H 'Authentication: <api_key>'
```

### 9.6 Reviewing the Deliverable

```bash
sed -n '1,120p' blitzy/documentation/app_2cd6ee777f8c.md          # environment preamble + Q1
wc -l blitzy/documentation/app_2cd6ee777f8c.md                    # 1717 lines
git diff --name-status origin/app_2cd6ee777f8c..HEAD              # MUST show only: A blitzy/documentation/app_2cd6ee777f8c.md
```

### 9.7 Troubleshooting

- **`KeyError: 'URL'` (or config import fails):** the `.env` is missing a required variable; ensure `URL`, `DB_URI`, and a non-empty `FLASK_SECRET` are set and `CONFIG` is exported **before** running Alembic or the app (this was the exact validation-harness pitfall — Alembic was run before `CONFIG` was exported).
- **Registration rejected offline (MX lookup):** `SKIP_MX_LOOKUP_ON_CHECK` is hardcoded `False` (`app/config.py:600`) and not env-overridable; provide DNS/MX for the domain, or note the value as non-canonical if the flag is toggled in-process as the tests do.
- **HTTP 429 on repeated login:** the route is rate-limited `10/minute`; set `DISABLE_RATE_LIMIT=1` in the `.env`.
- **`psql` not on host:** exec into the database container, e.g. `docker exec -i sl-pg13 psql -U test -d test`.
- **Gunicorn timestamps look second-resolution:** that is expected; the millisecond first→ready delta requires an external high-resolution timestamper wrapping the stderr stream.

---

## 10. Appendices

### Appendix A — Command Reference

| Purpose | Command |
|---------|---------|
| Reset empty schema (Q1) | `echo "drop schema public cascade; create schema public;" \| psql "$DB"` |
| Run migrations (Q1) | `alembic upgrade head` |
| Table count (Q1) | `psql "$DB" -c "SELECT count(*) FROM information_schema.tables WHERE table_schema='public';"` |
| Start web server (Q2/Q4/Q5) | `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` |
| Start email handler on custom port (Q3) | `timeout --signal=INT 5 python email_handler.py --port 25025` |
| Register (Q4/Q5) | `curl -sS -i -X POST .../api/auth/register -d '{"email":"...","password":"..."}'` |
| Login (Q4/Q5) | `curl -sS -i -X POST .../api/auth/login -d '{"email":"...","password":"..."}'` |
| Read alias limit (Q5) | `curl -sS -i .../api/user_info -H 'Authentication: <api_key>'` |
| Verify source integrity | `git diff --name-status origin/app_2cd6ee777f8c..HEAD` |

### Appendix B — Port Reference

| Port | Service | Reference |
|------|---------|-----------|
| 7777 | Web server (Gunicorn / Flask) | `Dockerfile:47`, `example.env:6` |
| 25025 | Email handler — custom (Q3) | AAP Q3 |
| 20381 | Email handler — default | `email_handler.py:2399` |
| 5432 | PostgreSQL 13 | canonical stack |
| 6379 | Redis 6 | canonical stack |

### Appendix C — Key File Locations

| Path | Role |
|------|------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The deliverable** (answer document, 1,717 lines) |
| `app/config.py` | Configuration read at import (`URL`, `DB_URI`, `FLASK_SECRET`, `MAX_NB_EMAIL_FREE_PLAN`, `SKIP_MX_LOOKUP_ON_CHECK`) |
| `app/models.py` | 76 ORM tables; `max_alias_for_free_account()` (L858–865); `activated`/`notification` defaults |
| `app/api/views/auth.py` | Register/login/activate; `{"error":"Account not activated"}`/`422` (L75–77) |
| `app/api/views/user_info.py` | `GET /api/user_info` → `max_alias_free_plan` |
| `email_handler.py` | `--port` default 20381 (L2399); `Listen for port` (L2403); `Start mail controller` (L2386) |
| `wsgi.py` / `server.py` | Gunicorn app object / dev entry point |
| `Dockerfile` | Canonical Gunicorn `CMD` on port 7777 (L47) |
| `migrations/versions/…7d7b84779837_user_audit_log.py` | Last `CREATE TABLE` in execution order (Q1) |
| `.github/workflows/main.yml` | Canonical service versions (PostgreSQL 13, Redis 6, Python 3.10) |

### Appendix D — Technology Versions (observed on the canonical stack)

| Component | Version | Canonical reference |
|-----------|---------|---------------------|
| Python | 3.10.18 | `Dockerfile`, `pyproject.toml` `^3.10`, CI `:17`/`:40` |
| PostgreSQL | 13.23 (`postgres:13`) | CI `:47` |
| Redis | 6.2.22 (`redis:6`) | CI `:94` |
| Gunicorn | 20.0.4 | `poetry.lock:1447-1448` |
| Flask / Werkzeug | 1.1.2 / 1.0.1 | `poetry.lock` |
| SQLAlchemy / Alembic / Flask-Migrate | 1.3.24 / 1.4.3 / 2.5.3 | `poetry.lock` |
| psycopg2-binary | 2.9.3 | `poetry.lock` |
| aiosmtpd | 1.4.2 | `poetry.lock` |

### Appendix E — Environment Variable Reference

| Variable | Purpose | Source (`app/config.py`) |
|----------|---------|--------------------------|
| `URL` | Base URL, read at import (missing → `KeyError`) | L79 |
| `DB_URI` | PostgreSQL connection string | L192 |
| `FLASK_SECRET` | Flask secret (must be non-empty) | L196–198 |
| `NOT_SEND_EMAIL` | Suppress outbound email (register/activate) | L91 |
| `DISABLE_RATE_LIMIT` | Disable Flask-Limiter (repeated login) | L602 |
| `MAX_NB_EMAIL_FREE_PLAN` | Free-plan alias limit (Q5), default 5 | L120–124 |
| `SKIP_MX_LOOKUP_ON_CHECK` | Hardcoded `False`, not env-overridable (Q4 gate) | L600 |

### Appendix F — Developer Tools Guide

| Tool | Use |
|------|-----|
| `git diff --name-status origin/app_2cd6ee777f8c..HEAD` | Confirm only the answer document was added (source integrity). |
| `psql` (via `docker exec sl-pg13`) | Direct DB inspection: table count (Q1), `activated`/`notification` (Q4). |
| External ms timestamper (`time.perf_counter()` wrapper) | Measure Gunicorn first→ready delta at millisecond resolution (Q2). |
| `curl -sS -i` | Capture full HTTP response (status + headers + body) for Q4/Q5. |
| `alembic current` / `alembic upgrade head` | Migration state and execution (Q1). |

### Appendix G — Glossary

| Term | Definition |
|------|------------|
| AAP | Agent Action Plan — the primary directive defining project scope. |
| Canonical stack | The exact service/interpreter versions the repository declares (Python 3.10, PostgreSQL 13, Redis 6). |
| Global-live limit | A limit computed per request from global config (no per-user snapshot) — the Q5 finding. |
| Ready line | The Gunicorn log line signaling readiness: `Listening at: http://0.0.0.0:7777 (<pid>)`. |
| MX-lookup gate | Registration check (`email_can_be_used_as_mailbox`) that rejects domains with no MX unless `SKIP_MX_LOOKUP_ON_CHECK` is true. |
| Ephemeral artifact | A temporary `.env`/script created for observation and removed afterward — never committed. |
| Path-to-production | Standard activities to move a validated deliverable to production; here, human review + merge. |

---

_Completion basis: AAP-scoped hours (PA1). Completed 40 h ÷ Total 44 h = 90.9%. Remaining 4 h is identical across §1.2, §2.2, and §7. Brand colors — Completed `#5B39F3`, Remaining `#FFFFFF` — applied throughout._