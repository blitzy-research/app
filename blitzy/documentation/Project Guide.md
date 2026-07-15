# Blitzy Project Guide — SimpleLogin Runtime Behavioral Investigation

> **Deliverable:** `blitzy/documentation/app_2cd6ee777f8c.md` · **Branch:** `blitzy-bbae9d68-fff2-457c-b3c7-71b46c6af08e` · **HEAD:** `64418a5d` · **Base:** `2cd6ee77`
>
> **Brand color legend** — <span style="color:#5B39F3">■</span> **Completed / AI Work = Dark Blue `#5B39F3`** · <span style="color:#FFFFFF">□</span> **Remaining / Not Completed = White `#FFFFFF`** · Headings accent = Violet-Black `#B23AF2` · Highlight = Mint `#A8FDD9`

---

## 1. Executive Summary

### 1.1 Project Overview

This project is a **read-only, evidence-based runtime behavioral investigation** of the open-source **SimpleLogin** email-aliasing application (a Flask/Python monolith). The objective was to empirically answer **eight discrete questions (Q1–Q8)** about SimpleLogin's authentication, session, email-forwarding, alias-token, and API-key subsystems by **actually building, running, and exercising the system** — capturing real runtime output rather than reading code. The target audience is the requesting engineer/security reviewer. The technical scope spans the API, auth, session, email-forwarding, and alias subsystems, yet produces exactly **one write**: a Markdown Q&A answer document. A strict constraint governed all work: **no existing source file may be modified**, and any temporary observation scripts must be cleaned up afterward.

### 1.2 Completion Status

The project is **93.9% complete** (62 of 66 hours). Completion is calculated using the PA1 AAP-scoped hours methodology: `Completed Hours ÷ (Completed + Remaining) × 100 = 62 ÷ 66 = 93.9%`. All 17 in-scope AAP requirements are delivered; the remaining 4 hours are inherent **human review-and-merge** path-to-production work.

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeWidth':'2px','pieTitleTextSize':'16px','pieSectionTextSize':'13px','pieLegendTextColor':'#222222'}}}%%
pie showData title Completion — 93.9% Complete (62 of 66 hrs)
    "Completed Work (AI)" : 62
    "Remaining Work" : 4
```

| Metric | Hours |
|---|---|
| **Total Hours** | **66** |
| **Completed Hours (AI + Manual)** | **62** (AI: 62 · Manual: 0) |
| **Remaining Hours** | **4** |
| **Percent Complete** | **93.9%** |

> Legend: <span style="color:#5B39F3">■</span> Completed = `#5B39F3` · <span style="color:#FFFFFF">□</span> Remaining = `#FFFFFF`.

### 1.3 Key Accomplishments

- ✅ **All eight questions (Q1–Q8) answered directly** with runtime evidence, each leading with a bold **Direct answer** and grounded in `file:line` references.
- ✅ **Canonical runtime stack stood up and exercised** — PostgreSQL 13.23 + Redis 6.2.22 + `gunicorn wsgi:app` on `:7777`, migrated (`alembic` head `32f25cbf12f6`) and seeded with a verified user, mailbox, alias, and API key.
- ✅ **Real canonical entry points only** — real HTTP requests, a real `aiosmtpd` SMTP delivery (Q5), the real alias-creation form (Q6), and direct inspection of Redis (Q3). No mocks, debug hooks, or synthetic tokens.
- ✅ **Byte-exact and stateful evidence** — Q3 raw pickle bytes reported exactly; before/during/after captured for Q4, Q6, and Q7; Q6 repeated across **two independent real-form cycles**.
- ✅ **Read-only mandate fully honored** — the only repository change is the single answer document; **zero source files modified**.
- ✅ **All temporary observation scripts cleaned up**; runtime residue removed; evidence sessions invalidated.
- ✅ **Independently re-verified** — every invariant signal reproduced exactly against the live system with **zero inaccuracies**; Report-7 findings F-P4-01..05 resolved.
- ✅ **Subsystem test suites green** — 83 passed / 0 failed across the Q1–Q8 areas.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|---|---|---|---|
| _None._ The deliverable is complete, committed, and re-verified with zero inaccuracies. | No release blockers. | — | — |

> There are **no critical unresolved issues**. The remaining work is limited to human review and merge (see §1.6 and §2.2). Product-behavior observations that a human may wish to act on separately are listed as optional follow-ups in §5 and §8; they are **out of scope** for this read-only documentation task.

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|---|---|---|---|---|
| Repository (`simple-login`) | Read/Write | None — repo accessed, branch committed successfully. | ✅ No issue | — |
| PostgreSQL 13 / Redis 6 (canonical) | Service | None — both stood up and reachable during observation. | ✅ No issue | — |
| Full-suite test DB on `:15432` | Service provisioning | Not an access/permission issue — the dedicated test database used by the *full* app test suite was simply **not provisioned** in the canonical containerized topology (only Q1–Q8 subsystem tests were run). | ⚠ Optional (out of scope) | Human (optional) |

> **No access issues identified.** The only note is an *infrastructure-provisioning* item (the `:15432` full-suite DB), which is out of scope for a read-only documentation deliverable and is listed for completeness only.

### 1.6 Recommended Next Steps

1. **[High]** Perform SME technical review and acceptance of the deliverable — read Q1–Q8, confirm each Direct answer resolves its question, and confirm evidence sufficiency and `[OBSERVED]`/`[INFERRED]` labelling.
2. **[Medium]** Approve and merge the pull request (single file: `blitzy/documentation/app_2cd6ee777f8c.md`).
3. **[Low]** Confirm the transparent disclosures (the `alembic` migration workaround; the `:15432` full-suite topology note) do not affect answer validity.
4. **[Low]** *(Optional, separate initiative)* Route the documented product observations — Q4 session-fixation, Q2b session-only-sudo `500`, Q3 pickle sessions — to the SimpleLogin product/security team for their own evaluation.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

Each completed component traces to a specific AAP requirement. **Total = 62 hours** (matches Completed Hours in §1.2).

| Component | Hours | Description |
|---|---:|---|
| Runtime environment stand-up | 6 | Canonical PG13 + Redis6 + gunicorn + aiosmtpd; Docker network/container wiring; `DB_URI`/`MEM_STORE_URI` override of the image's bundled PG15/Redis7. |
| Database migration + workaround | 3 | `alembic current`/`upgrade head` (head `32f25cbf12f6`); transparent schema-only `pg_dump` + `alembic stamp` workaround to avoid the `pg_trgm` rollback branch. |
| Seed data provisioning | 2 | Verified user, mailbox, alias, and API key via `flask dummy-data`; `psql` readback. |
| Methodology documentation | 3 | Exact stand-up commands, health check, worker processes, column-type disclosures, and conventions. |
| Q1 — API session fallback | 3 | Session-only `GET /api/aliases` → 200 + JSON; unauthenticated 401 control; `api_key` whole-table hash invariance. |
| Q2 — Sudo guard (both conditions) | 5 | Valid-key-no-sudo → 440 "Need sudo" (22 bytes); session-only edge → 500 `AttributeError` on `NoneType.sudo_mode_at`; server traceback + access-log correlation. |
| Q3 — Redis session format (byte-exact) | 3 | `redis-cli --no-raw` + `redis-py` pickle enumeration; key `session:<uuid>`; 6 keys; 300 bytes. |
| Q4 — Session id across login | 3 | Cookie before/after; `itsdangerous.Signer` decode; logout rotation + deletion; pre-logout replay → 401. |
| Q5 — Forward header survival | 6 | Real `aiosmtpd` `handle_DATA` + `smtplib` delivery; three-header verdicts; full forwarded header block + body; DB row readback. |
| Q6 — Alias token expiry | 5 | Two independent real-form cycles; suffix minted from real form HTML; 6 POSTs at 3 monotonic ages; ~10-min wall-clock waits; DB before/after. |
| Q7 — API-key usage stats | 3 | Two batches × 5 calls; `times`/`last_used` deltas; invalid-key control (unchanged); session-path contrast; column types. |
| Q8 — Failed-login logging + response | 4 | Wrong-password POST; bounded log window; negative grep for a dedicated log line; NewRelic no-op observation; success + nonexistent-user controls. |
| Document consolidation & authoring | 5 | 2,072-line document; direct answers; `[OBSERVED]`/`[INFERRED]` labelling; summary table. |
| QA re-verification vs live system | 8 | Re-verify all 8 answers; reproduce every invariant; resolve Report-7 F-P4-01..05; doc-integrity checks; 83 subsystem tests. |
| Cleanup & read-only verification | 3 | Delete temp scripts; remove residue; invalidate sessions; `git`-clean proof. |
| **Total** | **62** | |

### 2.2 Remaining Work Detail

Each category is human path-to-production work; **no incomplete AAP deliverable exists**. **Total = 4 hours** (matches Remaining Hours in §1.2 and §7).

| Category | Hours | Priority |
|---|---:|---|
| SME technical review & acceptance of the 13.7k-word deliverable (verify each Q1–Q8 Direct answer, evidence sufficiency, labels) | 2.5 | High |
| PR review, approval & merge of the single deliverable | 0.5 | Medium |
| Confirm transparent disclosures (migration workaround; `:15432` full-suite topology note) do not affect answer validity | 1.0 | Low |
| **Total** | **4.0** | |

> **Excluded from the hour totals (separate initiatives, out of AAP scope):** product-team follow-ups on Q4 session-fixation, Q2b session-only-sudo `500`, Q3 pickle sessions, and optional `:15432` full-suite provisioning. These are surfaced narratively in §5/§8 but are deliberately **not** counted in the 66-hour total to preserve cross-section arithmetic integrity.

### 2.3 Hours Reconciliation

| Check | Result |
|---|---|
| §2.1 Completed total | 62 h |
| §2.2 Remaining total | 4 h |
| §2.1 + §2.2 = §1.2 Total | 62 + 4 = **66 h** ✅ |
| Completion = 62 ÷ 66 | **93.9%** ✅ |
| Remaining identical in §1.2, §2.2, §7 | 4 h ✅ |

---

## 3. Test Results

All tests below originate from **Blitzy's autonomous validation logs** for this project. The subsystem suites were executed against the canonical `sl-pg13` + `sl-redis6` stack in read-only mode (the `conftest` wraps each test in `connection.begin()` / `transaction.rollback()`, so no DB mutation persists). The runtime behavioral verifications are the eight questions, each independently re-verified end-to-end against the live system.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---|---|---:|---:|---:|---:|---|
| Unit/Integration — subsystem suites | `pytest` | 83 | 83 | 0 | n/a | Files: `tests/api/test_sudo.py`, `test_user.py`, `test_user_info.py`, `test_auth.py`, `tests/auth/test_login.py`, `tests/test_alias_suffixes.py`, `tests/test_email_utils.py`. Runtime 18.07 s; 18 benign `DeprecationWarning`s only. |
| Runtime behavioral re-verification (Q1–Q8) | Live HTTP · SMTP · Redis · DB | 8 | 8 | 0 | n/a | Each question's invariant signal reproduced **exactly** against the running system; zero inaccuracies found. |
| **Total** | | **91** | **91** | **0** | | 100% pass rate. |

**Notes on coverage & scope.** Coverage instrumentation was **not** the objective of this task (it is a targeted subsystem + behavioral verification, not a coverage-gated build), so no coverage percentage is reported rather than a fabricated one. The **full** application test suite targets a dedicated test database on `localhost:15432` that is **not provisioned** in the canonical containerized topology; because zero source files changed (no regression is possible) and the task is a read-only documentation deliverable, running the full suite is out of scope. All seven Q1–Q8 subsystem suites pass 100%.

---

## 4. Runtime Validation & UI Verification

**Runtime health**
- ✅ **Operational** — Application live via `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15`; `GET http://localhost:7777/auth/login` → **HTTP 200**.
- ✅ **Operational** — PostgreSQL **13.23** (`sl-pg13`) reachable; schema migrated to head `32f25cbf12f6`.
- ✅ **Operational** — Redis **6.2.22** (`sl-redis6`) reachable; session store, rate limiting, and job backend live.
- ✅ **Operational** — Inbound SMTP path via real `aiosmtpd` `handle_DATA` exercised for Q5.

**API integration outcomes**
- ✅ **Operational** — Q1: session-fallback `GET /api/aliases` → **200** + 2,550-byte JSON alias list; unauthenticated control → **401** `{"error":"Wrong api key"}`.
- ✅ **Operational** — Q2a: valid API key without active sudo on `DELETE /api/user` → **440** `{"error":"Need sudo"}` (22 bytes), non-destructive.
- ⚠ **Partial (documented edge)** — Q2b: session-only `DELETE /api/user` (`g.api_key = None`) → **HTTP 500** `{"error":"Internal error"}` via an unhandled `AttributeError`. This is the *observed and documented* application behavior, faithfully captured per the AAP (it is **not** an error in the deliverable).
- ✅ **Operational** — Q7: API-key calls increment `times` (0→5→10) and set `last_used=arrow.now()`; invalid-key control leaves the row unchanged.

**Session & token behavior**
- ✅ **Verified** — Q3: authenticated session stored in Redis under `session:<uuid>` as a **pickle** stream (protocol 4, `\x80\x04`), **300 bytes**, **6 keys** (`_permanent, _fresh, csrf_token, _user_id, _id, sudo_time`).
- ✅ **Verified** — Q4: `slapp` cookie and decoded UUID **byte-identical** before/after login (no session-id rotation); logout deletes the key and rotates the UUID; pre-logout replay → 401.
- ✅ **Verified** — Q6: alias-creation token valid at ~0 s and ~593 s (DB rows created), expired at ~607 s (no row; `toastr.warning "Alias creation time is expired, please retry"`) → **600 s** window.

**Email forwarding**
- ✅ **Verified** — Q5: `X-Test-Custom` **stripped**, `Received` **stripped**, original `Reply-To` **stripped** then replaced by a reverse-alias `Reply-To`; forwarded message + DB rows captured.

**UI verification**
- ✅ **Operational** — The login form is the only UI exercised (to establish a browser session for Q1/Q2/Q4 and for Q8). Successful login → **302** redirect; wrong-password → **200** re-render (`Content-Length 7017`) with the flashed error `Email or password incorrect` rendered inline. No account enumeration (nonexistent user yields the same 200 + flash).

---

## 5. Compliance & Quality Review

This section cross-maps the AAP's governing rules and deliverable requirements to observed compliance, notes fixes applied during autonomous validation, and lists outstanding items.

| Benchmark (AAP Rule / Requirement) | Status | Progress | Evidence / Notes |
|---|---|---|---|
| **MainRule** — Create `<branch>.md` in `blitzy/documentation/`, answer all questions | ✅ Pass | 100% | `blitzy/documentation/app_2cd6ee777f8c.md` (2,072 lines) answers Q1–Q8. |
| **MainRule** — Do **not** modify any existing source file | ✅ Pass | 100% | `git diff --name-status 2cd6ee77 HEAD` = single `A` (the doc). Zero source changes. |
| **MainRule** — Temporary scripts removed | ✅ Pass | 100% | Cleanup section: temp scripts = 0; residue removed; sessions invalidated (`session:*` 8→0). |
| **Rule 1** — Run-first; canonical entry points; stable across ≥2 runs | ✅ Pass | 100% | All answers from runtime output; Q6 across two real-form cycles; Q7 across two batches. |
| **Rule 2** — Exhaustive condition & evidence coverage (primary + edge; before/during/after) | ✅ Pass | 100% | Q2 both paths; Q5 all three headers; Q7 each field before/after; byte-exact Q3. |
| **Rule 3** — Observed-output discipline; label inferences | ✅ Pass | 100% | 47 `[OBSERVED]` + 9 `[INFERRED]` labels; complete unedited command output next to each claim. |
| **Rule 4** — Complete, precise, grounded; `file:line` + named function | ✅ Pass | 100% | Direct answer per question; exact `file:line` (e.g., `app/api/base.py:70`, `app/alias_suffix.py:40`). |
| **Dependency policy** — no add/update/remove | ✅ Pass | 100% | `pyproject.toml`/`poetry.lock` unchanged. |
| **Doc integrity** — UTF-8, balanced fences, no placeholders | ✅ Pass | 100% | Valid UTF-8; 66 balanced fences (33 evidence blocks); no TODO/FIXME/truncation. |

**Fixes applied during autonomous validation:** the final validation cycle resolved Report-7 findings **F-P4-01..05** with complete runtime evidence (committed in `64418a5d`) and independently re-verified all eight answers against the live system, finding **zero inaccuracies** (so no further edits were required).

**Outstanding items:** none for the deliverable. **Optional product-team follow-ups** surfaced by the investigation (out of AAP scope, not counted in hours): (a) evaluate session-ID rotation on login (Q4); (b) handle `g.api_key = None` on the session-only sudo path to avoid the 500 (Q2b); (c) review pickle-serialized Redis sessions (Q3).

---

## 6. Risk Assessment

Overall posture: **LOW.** Because **zero source files changed**, the dominant software risks (compilation failure, functional regression, security-of-changes) are effectively nil. Remaining risks concern reproducibility nuance and out-of-scope product observations.

| Risk | Category | Severity | Probability | Mitigation | Status |
|---|---|---|---|---|---|
| Migration workaround needed on a truly empty DB (`pg_trgm` rollback branch) | Technical | Low | Medium | Exact schema-only `pg_dump` + `alembic stamp` workaround disclosed in the doc; answers derive from a correctly migrated running system. | Documented / Accepted |
| Full app test suite not executed (needs test DB on `:15432`) | Technical | Low | Low | Zero source changed → no regression possible; all 7 Q1–Q8 subsystem suites pass 100%. | Accepted (out of scope) |
| Nine `[INFERRED]` findings could be imprecise | Technical | Low | Low | Each inference is transparently labelled and the underlying behavior is observed (e.g., 600 s bracketed by runtime; 440 observed, only its IIS provenance inferred). | Mitigated |
| Session/UUID values published as evidence | Security | Low | Low | Sessions invalidated (`session:*` 8→0); cookie **values** redacted in Q4/Q6/Q8 captures. | Mitigated |
| Product security behaviors documented, not fixed (Q4 no rotation; Q2b 500; Q3 pickle) | Security | Informational | N/A | Explicitly out of AAP read-only scope; surfaced for product-team follow-up. | Documented / Deferred |
| Topology-dependent cosmetic values (UUID, timestamps, `Content-Length`) | Operational | Low | Medium | Invariant signals (200/440/500/pickle/600 s/`times+1`) are environment-independent; exact commands + versions provided. | Mitigated |
| Ephemeral evidence (disposable fixtures, invalidated sessions no longer live) | Operational | Low | Low | Complete unedited output captured verbatim; re-run instructions provided. | Mitigated |
| NewRelic telemetry no-op (agent disabled) vs. enabled in prod | Integration | Low | Informational | Explicitly labelled + mechanism explained; the substantive HTTP-response answer is unaffected. | Documented |
| SMTP send-boundary capture (Q5) | Integration | Low | Low | Labelled as the canonical send-boundary interception (**not** a bypass); allow-list logic runs identically. | Mitigated / Documented |

---

## 7. Visual Project Status

**Project hours breakdown** — <span style="color:#5B39F3">■</span> Completed = `#5B39F3` · <span style="color:#FFFFFF">□</span> Remaining = `#FFFFFF`.

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeWidth':'2px','pieTitleTextSize':'16px','pieSectionTextSize':'13px','pieLegendTextColor':'#222222'}}}%%
pie showData title Project Hours — Completed vs Remaining
    "Completed Work" : 62
    "Remaining Work" : 4
```

**Remaining work by category (hours)** — from §2.2 (sums to 4 h):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#B23AF2','pie3':'#A8FDD9','pieStrokeColor':'#222222','pieStrokeWidth':'1px','pieSectionTextSize':'13px','pieLegendTextColor':'#222222'}}}%%
pie showData title Remaining Work by Category (4 h total)
    "SME review & acceptance" : 2.5
    "Disclosure-validity check" : 1.0
    "PR review & merge" : 0.5
```

> **Integrity:** the "Remaining Work" value (**4 h**) equals the Remaining Hours in §1.2 and the sum of the §2.2 Hours column. "Completed Work" (**62 h**) equals §2.1.

---

## 8. Summary & Recommendations

**Achievements.** The project fully delivers its AAP objective: a single, evidence-grounded Markdown document that answers all eight runtime questions about SimpleLogin from **actual system execution**. Every answer leads with a direct verdict, is backed by complete, unedited command output, and is grounded in exact `file:line` references. The canonical stack was stood up, migrated, and seeded; each question was exercised through its **real** entry point; stateful and timing-sensitive findings were captured before/during/after and repeated for stability; and the whole investigation was independently re-verified against the live system with **zero inaccuracies**. The read-only mandate was honored perfectly — the only repository change is the answer document, and all temporary scripts were cleaned up.

**Remaining gaps & critical path.** The project is **93.9% complete (62 of 66 hours)**. The remaining **4 hours** are entirely **human path-to-production**: an SME technical review/acceptance of the 13.7k-word deliverable (2.5 h), PR review and merge (0.5 h), and confirmation that the transparent disclosures do not affect answer validity (1.0 h). There are no High-priority engineering blockers and no incomplete AAP deliverables.

**Success metrics.** 8/8 questions answered and re-verified · 17/17 AAP requirements completed · 83/83 subsystem tests passing · 0 source files modified · 33 evidence blocks · 47 `[OBSERVED]` / 9 `[INFERRED]` labels.

**Production readiness.** For a documentation deliverable, "production" means acceptance and merge. The deliverable is **ready for review**: it is complete, committed, internally consistent, and self-verifying (every command can be re-run). Recommendation: **proceed to SME review and merge.** Separately — and outside this task's scope — the product/security team may wish to evaluate the documented behaviors (session-fixation on login, the session-only-sudo `500`, and pickle-serialized sessions), which are informative findings rather than defects in this deliverable.

| Metric | Value |
|---|---|
| Completion | **93.9%** (62 / 66 h) |
| AAP requirements completed | 17 / 17 |
| Questions answered & re-verified | 8 / 8 |
| Subsystem tests | 83 passed / 0 failed |
| Source files modified | **0** |
| Critical unresolved issues | 0 |

---

## 9. Development Guide

This guide documents how to build, run, verify, and reproduce the investigation. Every command is copy-pasteable; the canonical (Docker) path mirrors exactly how the evidence was captured.

### 9.1 System Prerequisites

- **Recommended:** Docker 28.x (Docker-in-Docker verified: `docker --version` → 28.5.2).
- **Or, native:** Python **3.10** (required — `Dockerfile` uses `FROM python:3.10`, `pyproject.toml` pins `python = "^3.10"`), Poetry, PostgreSQL **13**, Redis **6**.
- **OS build dependencies** (native path): `libre2-dev`, `libpq-dev`, `gnupg`, `gcc`, `python3-dev`.

### 9.2 Environment Setup (canonical Docker topology)

```bash
# Backing services + app container on a shared network (matches the evidence capture)
docker network create sl-canon
docker run -d --name sl-pg13 --network sl-canon \
    -e POSTGRES_USER=test -e POSTGRES_PASSWORD=test -e POSTGRES_DB=test postgres:13
docker run -d --name sl-redis6 --network sl-canon redis:6

# App container: bind-mount the repo at /app, publish port 7777
docker run -d --name sl-app -w /app --shm-size=256m -p 7777:7777 \
    -v /tmp/blitzy/app/blitzy-bbae9d68-fff2-457c-b3c7-71b46c6af08e_aefa13:/app \
    -v sl-venv:/app/venv --entrypoint /bin/bash \
    ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0 -c 'sleep infinity'
docker network connect sl-canon sl-app
```

```bash
# Inside the app container: activate venv and point at the canonical services
cd /app && . venv/bin/activate
export DB_URI="postgresql://test:test@sl-pg13:5432/test"
export MEM_STORE_URI="redis://sl-redis6"
export PYTHONPATH=/app
# config.py auto-loads /app/.env via python-dotenv; FLASK_SECRET drives CUSTOM_ALIAS_SECRET.
```

### 9.3 Dependency Installation

Dependencies are pre-installed in the provided image's `venv`. For a native environment, install with Poetry (versions pinned in `pyproject.toml` / `poetry.lock`):

```bash
pip install -U pip
curl -sSL https://install.python-poetry.org | python3 -
poetry install --no-interaction --no-ansi --no-root
```

Key pinned versions: `flask ^1.1.2`, `flask_login ^0.5.0`, `gunicorn ^20.0.4`, `aiosmtpd ^1.2`, `redis ^4.5.3`, `sqlalchemy 1.3.24`, `arrow ^0.16.0`, `psycopg2-binary ^2.9.3`.

### 9.4 Application Startup

```bash
# 1) Migrate the database schema
alembic current           # expect: 32f25cbf12f6 (head)
alembic upgrade head      # clean no-op if already at head

# 2) Seed a verified user / mailbox / alias / API key
FLASK_APP=wsgi:app flask dummy-data     # runs fake_data() + add_sl_domains()

# 3) Start the canonical WSGI server (port 7777, 2 workers)
gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15 \
    --access-logfile /tmp/gunicorn_access.log \
    --error-logfile /tmp/gunicorn_error.log \
    > /tmp/gunicorn_boot.log 2>&1 &
```

### 9.5 Verification Steps

```bash
# Containers & network
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}'

# App health — expect HTTP 200
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:7777/auth/login

# Migration head — expect 32f25cbf12f6
alembic current

# Read-only mandate proof — expect exactly one line: "A blitzy/documentation/app_2cd6ee777f8c.md"
git diff --name-status 2cd6ee77 HEAD

# Deliverable integrity
wc -l blitzy/documentation/app_2cd6ee777f8c.md          # 2072
file blitzy/documentation/app_2cd6ee777f8c.md           # UTF-8
grep -cE '^## Q[0-9]' blitzy/documentation/app_2cd6ee777f8c.md   # 8
```

### 9.6 Reproduce the Investigation (per question)

- **Q1** — With an authenticated cookie jar (no `Authentication` header): `curl -b jar.txt 'http://localhost:7777/api/aliases?page_id=0'` → **200** + JSON.
- **Q2** — `curl -X DELETE` on `/api/user`: with a valid key but no sudo → **440** `{"error":"Need sudo"}`; with a session cookie only → **500** `{"error":"Internal error"}`.
- **Q3** — `redis-cli -h sl-redis6 --no-raw GET session:<uuid>`; then `pickle.loads(...)` to enumerate the 6 keys.
- **Q4** — Capture the `slapp` cookie before and after `POST /auth/login` on the same jar; compare (byte-identical).
- **Q5** — Start a real `aiosmtpd` controller, deliver via `smtplib` with `X-Test-Custom` + `Received` + `Reply-To`; inspect the forwarded headers.
- **Q6** — Extract a signed suffix from `GET /dashboard/custom_alias`, `POST` it immediately (success) and again after >600 s (expired).
- **Q7** — Read `api_key.times`/`last_used` before/after N authenticated `GET /api/user_info` calls.
- **Q8** — `POST /auth/login` with a wrong password → **200** re-render + flash `Email or password incorrect`; inspect the bounded server-log window.

### 9.7 Troubleshooting

- **`alembic upgrade head` hits the `pg_trgm` rollback branch on an empty DB** — load a schema-only dump from a reference DB already at head, then `alembic stamp head` so `upgrade head` is a clean no-op (this workaround is disclosed transparently in the deliverable's Methodology section).
- **App can't reach DB/Redis** — confirm `DB_URI` and `MEM_STORE_URI` point at `sl-pg13:5432` / `sl-redis6`, and that `sl-app` is connected to the `sl-canon` network.
- **Port 7777 already in use** — stop the prior gunicorn (`kill <pid>` for the process you started) or publish a different host port.
- **Full app test suite fails to connect** — it expects a dedicated test DB on `localhost:15432`, which is not part of the canonical topology; provision it separately if full-suite runs are desired (out of scope here).

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---|---|
| Start canonical services | `docker run -d --name sl-pg13 --network sl-canon -e POSTGRES_USER=test -e POSTGRES_PASSWORD=test -e POSTGRES_DB=test postgres:13` · `docker run -d --name sl-redis6 --network sl-canon redis:6` |
| Migrate DB | `alembic upgrade head` |
| Seed data | `FLASK_APP=wsgi:app flask dummy-data` |
| Run app | `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` |
| Health check | `curl -s -o /dev/null -w '%{http_code}\n' http://localhost:7777/auth/login` |
| Read-only proof | `git diff --name-status 2cd6ee77 HEAD` |
| Subsystem tests | `python -m pytest tests/api/test_sudo.py tests/api/test_user_info.py tests/api/test_auth.py tests/auth/test_login.py tests/test_alias_suffixes.py tests/test_email_utils.py -q --timeout=60` |

### B. Port Reference

| Port | Service |
|---|---|
| 7777 | SimpleLogin Flask app (gunicorn) |
| 5432 | PostgreSQL 13 (`sl-pg13`) |
| 6379 | Redis 6 (`sl-redis6`) |
| 15432 | Dedicated **test** DB for the *full* suite (not provisioned in canonical topology) |
| 20381 | Ephemeral local `aiosmtpd` listener used for the Q5 SMTP delivery |

### C. Key File Locations

| Area | Path | Question(s) |
|---|---|---|
| **Deliverable** | `blitzy/documentation/app_2cd6ee777f8c.md` | All |
| API authorization decorators | `app/api/base.py` | Q1, Q2, Q7 |
| Sudo endpoint | `app/api/views/user.py` | Q2 |
| Alias endpoints | `app/api/views/alias.py` | Q1 |
| `ApiKey` model | `app/models.py` | Q7 |
| Redis session store | `app/session.py` | Q3, Q4 |
| Login manager config | `app/extensions.py` | Q4 |
| Email forward pipeline | `email_handler.py` | Q5 |
| Header constants | `app/email/headers.py` | Q5 |
| Alias-suffix signer | `app/alias_suffix.py` | Q6 |
| Alias-creation form | `app/dashboard/views/custom_alias.py` | Q6 |
| Login view | `app/auth/views/login.py` | Q8 |
| Login telemetry | `app/events/auth_event.py` | Q8 |
| Entry / config | `wsgi.py`, `server.py`, `app/config.py` | All |

### D. Technology Versions

| Component | Version |
|---|---|
| Python (app) | 3.10.18 |
| Flask | 1.1.2 |
| Flask-Login | 0.5.0 |
| Flask-WTF | 0.14.3 |
| aiosmtpd | 1.4.2 |
| redis (client) | 4.6.0 |
| itsdangerous | 1.1.0 |
| SQLAlchemy | 1.3.24 |
| arrow | 0.16.0 |
| gunicorn | 20.0.4 |
| newrelic | 8.8.0 |
| psycopg2-binary | 2.9.3 |
| PostgreSQL | 13.23 |
| Redis | 6.2.22 |
| Alembic head | 32f25cbf12f6 |

### E. Environment Variable Reference

| Variable | Example / Value | Purpose |
|---|---|---|
| `DB_URI` | `postgresql://test:test@sl-pg13:5432/test` | PostgreSQL connection string |
| `MEM_STORE_URI` | `redis://sl-redis6` | Redis (session store, rate limiting, jobs) |
| `PYTHONPATH` | `/app` | Import root |
| `FLASK_APP` | `wsgi:app` | Flask CLI entry (for `flask dummy-data`) |
| `FLASK_SECRET` | *(from `/app/.env`)* | Secret; derives `CUSTOM_ALIAS_SECRET` (Q6) |
| `URL` | `http://localhost:7777` | App base URL |
| `DISABLE_RATE_LIMIT` | `1` | Used in Q8 to stay below the `10/minute` login limit |

### F. Developer Tools Guide

| Tool | Use |
|---|---|
| `curl` (with cookie jar `-b`/`-c`) | Exercise API/auth endpoints for Q1, Q2, Q4, Q7, Q8 |
| `redis-cli --no-raw` / `redis-py` | Inspect raw session bytes and enumerate keys (Q3) |
| `psql` | Read `alias` / `api_key` rows before/after (Q6, Q7) |
| `aiosmtpd` + `smtplib` | Real inbound SMTP delivery and forwarded-message capture (Q5) |
| `alembic` | Schema migration / version inspection |
| `pytest` | Subsystem verification (read-only, transaction-rolled-back) |
| `git diff --name-status` | Prove the read-only mandate |

### G. Glossary

| Term | Meaning |
|---|---|
| **AAP** | Agent Action Plan — the primary directive defining scope and requirements. |
| **Session fallback** | The API authorization branch where a request lacking the `Authentication` header falls back to the Flask-Login `current_user` (Q1). |
| **Sudo guard** | The `require_api_sudo` decorator returning `440 "Need sudo"` for privileged operations without active sudo mode (Q2). |
| **HTTP 440** | A non-standard status code (Microsoft IIS "Login Time-out"), repurposed by SimpleLogin to signal "Need sudo". |
| **Allow-list (header)** | `delete_all_headers_except(...)` — keeps only an enumerated set of headers on the forwarded message (Q5). |
| **Reverse alias** | The generated `Reply-To`/`From` address that routes replies back through SimpleLogin (Q5). |
| **`[OBSERVED]` / `[INFERRED]`** | Labels distinguishing a directly-captured runtime value from a value reasoned from source/semantics. |
| **`session:<uuid>`** | The Redis key format under which authenticated sessions are stored (Q3, Q4). |
| **`times` / `last_used`** | `ApiKey` columns updated on each authenticated API-key call (Q7). |
