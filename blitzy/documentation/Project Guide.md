# Blitzy Project Guide — SimpleLogin Runtime Behavior Investigation (Q1–Q8)

> **Deliverable:** `blitzy/documentation/app_2cd6ee777f8c.md` (1,497 lines)
> **Task type:** Documentation / runtime-behavioral investigation (read-only)
> **Branch:** `blitzy-00d7b95c-a276-4103-8427-9f9a0ea344a7` · **Base:** `origin/app_2cd6ee777f8c` · **HEAD:** `86237593`

---

## 1. Executive Summary

### 1.1 Project Overview

This project empirically verifies **eight specific runtime behaviors** (Q1–Q8) of the SimpleLogin email-alias service's authentication, session-management, and email-forwarding subsystems, and records the observed evidence in a single new Markdown answer document. The user explicitly required answers grounded in *observed runtime behavior* — "what actually happens when you try it" — not code reading. Every claim is backed by the exact producing command, complete unedited output, a resolved answer, and a `file:line` citation, with observed values distinguished from inferred ones. The work is strictly **read-only**: zero SimpleLogin source files are modified, and exactly one documentation file is added. The target audience is engineers investigating SimpleLogin's auth/session/email internals.

### 1.2 Completion Status

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieLegendTextColor':'#B23AF2','pieStrokeWidth':'2px'}}}%%
pie showData title Completion — 90.7%
    "Completed Work (AI)" : 39
    "Remaining Work" : 4
```

| Metric | Value |
|--------|-------|
| **Total Hours** | **43** |
| **Completed Hours (AI + Manual)** | **39** (39 AI + 0 Manual) |
| **Remaining Hours** | **4** |
| **Percent Complete** | **90.7%** (39 ÷ 43) |

**Legend:** Completed = Dark Blue `#5B39F3` · Remaining = White `#FFFFFF`

### 1.3 Key Accomplishments

- ✅ Canonical stack stood up and exercised end-to-end: gunicorn web app (`:7777`), standalone SMTP `email_handler.py` (`:20381`), PostgreSQL 15, Redis 7 — all via documented commands.
- ✅ **All 8 questions (Q1–Q8) answered** with real captured output, the exact producing command, a resolved answer, and `file:line` references (82 citations total).
- ✅ **Q2 defect discovered and correctly reported (not patched):** browser-session call to a `@require_api_sudo` endpoint raises an unhandled `AttributeError` → `HTTP 500`, distinct from the expected `440`. Full traceback captured; destructive-endpoint safety proven (no account deleted).
- ✅ Magnitude/timing rigor: Q6 (token expiry) and Q7 (usage counters) observed across **two stable runs**; Q6 brackets the 600-second boundary with `595s`/`607s`.
- ✅ Session forensics: raw `pickle` protocol-4 bytes read directly from Redis (Q3); session identifier proven unchanged across login (Q4).
- ✅ Email forensics: a real `swaks` message with three test headers sent through the handler; forwarded output captured verbatim and each header's fate classified (Q5).
- ✅ **Read-only compliance proven** (before/after `git status` identical; 8 pre-existing image-baked files untouched) and **cleanup verified** (DB rows removed, Redis flushed, `.env` restored byte-identical, temp scripts removed).
- ✅ Autonomous validation re-ran the full stack and confirmed **8/8** behaviors reproduce; deliverable committed with a clean working tree.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| _None blocking._ All AAP-scoped requirements are delivered and validated. | — | — | — |
| (Informational, out-of-scope) Q2 product defect: unhandled `AttributeError` → `HTTP 500` on the browser-session sudo path | Robustness/error-handling gap on an auth route; **documented, not fixed** per read-only mandate | SimpleLogin product team | N/A (out of scope for this task) |

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-----------------|----------------|-------------------|-------------------|-------|
| SimpleLogin repository | Git read/write | None — branch checked out, single file committed | ✅ Resolved | Blitzy |
| PostgreSQL / Redis / web / SMTP | Runtime services | None — all bundled in the canonical image and started successfully | ✅ Resolved | Blitzy |
| Third-party APIs | External credentials | Not required — dev config uses `NOT_SEND_EMAIL=true`; Q5 outbound routed to a local sink | ✅ N/A | — |

**No access issues identified.** All resources required for the investigation were available inside the canonical container.

### 1.6 Recommended Next Steps

1. **[High]** Perform a human technical review of `blitzy/documentation/app_2cd6ee777f8c.md`: read the full document and spot-check at least two observations (recommend Q2's `500` defect and Q6's timing boundary) against a fresh stack.
2. **[High]** Confirm the Read-Only Compliance Proof and Cleanup Verification, and sample a few `file:line` citations against `/app` source.
3. **[Medium]** Review and merge the single-file PR to the base branch after verifying the additive, read-only diff.
4. **[Low / out-of-scope]** File a separate product-team backlog ticket to triage the documented Q2 defect (unhandled `AttributeError` → `500`). This requires a SimpleLogin source change and is intentionally excluded from this read-only task.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|-------|-------------|
| Environment & canonical stack bring-up | 5.0 | Start PostgreSQL 15 + Redis 7; run `alembic upgrade head` + `flask dummy-data`; launch gunicorn web (`:7777`) and `email_handler.py` (`:20381`); confirm `john@wick.com/password` login; document the run-setup preamble |
| Q1 — API auth via browser session (no key) | 1.5 | `GET /api/user_info` with session cookie only vs. no-cookie contrast; capture status + JSON body [app/api/base.py:L17,L20-25,L27] |
| Q2 — Privileged-op defect investigation | 3.5 | Resolve `500` (NoneType.`sudo_mode_at`) vs. `440` "Need sudo"; capture full traceback; destructive-endpoint safety before/after; create contrast key [app/api/base.py:L42,L46-49,L69] |
| Q3 — Session storage forensics | 2.5 | Unsign `slapp` cookie; read raw `pickle` bytes from Redis; `pickle.loads` to enumerate keys; capture TYPE/TTL [app/session.py:L18,L44-45,L91] |
| Q4 — Session-identifier fixation test | 2.0 | Capture cookie before + after login on the same jar; unsign both; compare UUIDs across two runs [app/session.py:L61-64,L68-80] |
| Q5 — Email-forwarding header forensics | 4.0 | Stand up aiosmtpd capture sink; temporary `.env` change (reverted byte-identical); send real `swaks` email with three headers; capture forwarded message; classify each header [email_handler.py:L793-810,L867,L873] |
| Q6 — Alias-token expiry timing | 3.0 | Fetch fresh signed suffix; submit at ~0s/595s (success) and 607s (failure); two cycles; bracket the 600s boundary [app/alias_suffix.py:L40] |
| Q7 — API-key usage statistics | 2.0 | Create key canonically; N=5 key-auth calls × 2 runs; full before/after DB row dumps; control key [app/api/base.py:L30-31; app/models.py:L2358-2360] |
| Q8 — Failed-login investigation | 2.0 | POST wrong credentials; snapshot log delta; capture `HTTP 200` re-render + flash; probe rate limiter to `429` [app/auth/views/login.py:L45,L49-50,L74] |
| Deliverable authoring | 7.0 | Write 1,497 lines: per-question evidence, Coverage Confirmation table, Security & Privacy Handling, Read-Only Compliance Proof, Cleanup Verification |
| Read-only compliance & cleanup discipline | 2.0 | Baseline `git status` capture; temp scripts kept outside repo; DB/Redis/`.env` teardown; final verification of clean tree |
| Autonomous validation re-run | 4.5 | Final Validator: full stack re-run, 8/8 behavior reproduction, citation verification, commit-state confirmation |
| **Total Completed** | **39.0** | Matches Completed Hours in Section 1.2 |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|-------|----------|
| Human technical review & acceptance of the deliverable (read full doc + verify 8 answers/coverage; re-run spot-check of Q2 + Q6 on a fresh stack; confirm read-only proof, cleanup, and sampled citations) | 3.0 | High |
| PR review & merge to base branch (verify single-file additive change + read-only compliance; approve; merge) | 1.0 | Medium |
| **Total Remaining** | **4.0** | Matches Remaining Hours in Section 1.2 and Section 7 pie chart |

> **Out-of-scope (excluded from all hours):** Fixing the Q2 defect (requires SimpleLogin source changes — forbidden by the read-only mandate), running SimpleLogin's pytest suite, and any SimpleLogin feature/deployment work.

### 2.3 Hours Reconciliation

- Section 2.1 total (Completed) = **39.0 h**
- Section 2.2 total (Remaining) = **4.0 h**
- **39.0 + 4.0 = 43.0 h = Total Project Hours** (Section 1.2) ✓
- Completion = 39 ÷ 43 = **90.7%** (consistent across Sections 1.2, 7, 8) ✓

---

## 3. Test Results

The "tests" for this task are **Blitzy's autonomous runtime-behavior validations** — each question (Q1–Q8) was exercised against the live canonical stack and its documented observation re-confirmed. These originate entirely from Blitzy's autonomous validation logs for this project. (SimpleLogin's own `pytest` unit suite is **out of AAP scope** and was neither required nor run.)

| Test Category | Framework / Method | Total | Passed | Failed | Coverage % | Notes |
|---------------|--------------------|-------|--------|--------|------------|-------|
| API auth/authorization behaviors (Q1, Q2, Q7) | Live HTTP via `curl` against gunicorn (`:7777`) | 3 | 3 | 0 | 100% | Q1 `200`+JSON / `401` contrast; Q2 `500` defect + `440` contrast (safety proven); Q7 counter progression `0→5→10` |
| Session storage & lifecycle (Q3, Q4) | Redis inspection (`redis-cli`) + cookie unsign | 2 | 2 | 0 | 100% | Q3 pickle proto-4, 6 keys, `session:<uuid4>`, TTL 604800; Q4 identifier unchanged across login (2 runs) |
| Email forwarding end-to-end (Q5) | SMTP via `swaks` + `aiosmtpd` capture sink | 1 | 1 | 0 | 100% | X-Custom stripped, Received stripped, Reply-To rewritten; forwarded bytes captured verbatim |
| Timing / expiry boundary (Q6) | Time-bracketed HTTP observation, 2 cycles | 1 | 1 | 0 | 100% | `201`@0s/595s, `412`@607s; boundary `(595s,607s]` stable both runs |
| Auth-failure / login (Q8) | Form POST + server-log delta capture | 1 | 1 | 0 | 100% | `HTTP 200` re-render + flash; rate limiter `10/min` → `429` |
| **Total** | **Blitzy Autonomous Runtime Validation** | **8** | **8** | **0** | **100%** | 8/8 behaviors reproduced byte-/structurally-identical |

**Integrity note:** Per Cross-Section Rule 3, every entry above derives from Blitzy's autonomous validation execution logs (GATE 1 = 100% pass; 8/8 observations reproduced). Dynamic per-run values (UUIDs, timestamps, DB ids, HMAC signatures) were correctly treated as non-discrepancies.

---

## 4. Runtime Validation & UI Verification

**Runtime service health (canonical stack):**

- ✅ **Operational** — Web app (gunicorn `wsgi:app -b 0.0.0.0:7777 -w 2`): two sync workers booted; `7777 OPEN`.
- ✅ **Operational** — SMTP `email_handler.py` (`:20381`): "Listen for port 20381"; `20381 OPEN`.
- ✅ **Operational** — PostgreSQL 15 (`:5432`, db `simplelogin`): schema at head `32f25cbf12f6`; dev account `john@wick.com` present (`activated=t`).
- ✅ **Operational** — Redis 7.0.15 (`:6379`): `session:*` keys present, confirming the server-side `RedisSessionStore` path is active.

**Behavioral validation (per question):**

- ✅ **Operational** — Q1 session-fallback (`200` + user-info JSON; `401` no-cookie contrast).
- ✅ **Operational** — Q2 both paths captured (`500` browser-session defect; `440` key-without-sudo); account-deletion safety proven before/after.
- ✅ **Operational** — Q3 raw pickle bytes + key set + TTL read directly from Redis.
- ✅ **Operational** — Q4 identifier unchanged across login (byte-identical cookie), two runs.
- ✅ **Operational** — Q5 real email forwarded; three-header fate classified from captured bytes.
- ✅ **Operational** — Q6 expiry boundary bracketed `(595s, 607s]`, stable across two cycles.
- ✅ **Operational** — Q7 three DB fields updated (`times`, `last_used`, `updated_at`); control key NULL.
- ✅ **Operational** — Q8 `HTTP 200` re-render + flash; rate-limit `429` observed.

**UI verification:** Not applicable beyond login confirmation. This is a backend/API/documentation task with **no UI changes**; the SimpleLogin dashboard was touched only to confirm a successful login (`302 → /dashboard/`). No front-end artifacts were produced or modified.

---

## 5. Compliance & Quality Review

Cross-mapping the AAP's governing rule set (**SWE-AtlasQnA-Repo**) and Blitzy quality benchmarks against the delivered artifact:

| Benchmark / AAP Rule | Requirement | Status | Evidence |
|----------------------|-------------|--------|----------|
| Run-first methodology | Build & run before writing; answers from observation | ✅ Pass | Stack started before any answer; every Q embeds a producing command + captured output |
| Real canonical entry points | No mocks/bypasses; web `:7777`, SMTP `:20381` | ✅ Pass | gunicorn + `email_handler.py`; API via `curl`; login via real `/auth/login` |
| Complete output for every claim | Full unedited output + command | ✅ Pass | Fenced raw output blocks throughout (status lines, JSON, tracebacks, DB dumps, SMTP transcripts) |
| Exact & grounded (`file:line`) | Cite `file:line`, name function/method | ✅ Pass | 82 `file:line` citations; functions named (`authorize_request`, `check_sudo_mode_is_active`, etc.) |
| Observed vs. inferred | Distinguish observed from inferred | ✅ Pass | 31 "OBSERVED" / 11 "INFERRED" labels; e.g., Q6 boundary observed, exact `600` inferred |
| Magnitude/timing across ≥2 runs | State scale; confirm stability | ✅ Pass | Q6 two cycles; Q7 N=5 × two runs; scale stated explicitly |
| Answer every named sub-part | Decompose + final coverage pass | ✅ Pass | "Coverage Confirmation" table maps every Q and sub-part to its observed result |
| Report defects, do not fix | Observe-not-patch on read-only scope | ✅ Pass | Q2 defect reported with traceback, explicitly "NOT patched" |
| Read-only scope | No source file modified | ✅ Pass | `git diff` = single added file; 11 governing source files unchanged |
| Cleanup / temp scripts removed | Repo left unchanged | ✅ Pass | before/after `git status` identical; `.env` restored byte-identical; `/tmp/inv` removed |
| Correct deliverable location/naming | `blitzy/documentation/<branch>.md` | ✅ Pass | `blitzy/documentation/app_2cd6ee777f8c.md` committed |
| Security/privacy handling | Redact reusable secrets; invalidate | ✅ Pass | HMAC sigs, API-key codes, `csrf_token`/`_id` redacted; Redis flushed; keys deleted |

**Fixes applied during autonomous validation:** None required — the documentation was found fully accurate on re-validation (zero doc edits; zero observed/documented discrepancies). The most recent commit (`86237593`) refined Q2/Q8 `file:line` citations for precision.

**Outstanding compliance items:** None within scope.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Dynamic values (UUIDs, timestamps, ids, HMAC) are not byte-reproducible on re-run | Technical | Low | High | Doc keys answers to structure, not literals; dynamic vs. stable values called out | Documented |
| Evidence pinned to Python 3.10 / Flask 1.1.2 / Flask-Login 0.5.0 / itsdangerous 1.1.0 / redis 4.6.0; upgrades could shift Q3 pickle proto or Q4 rotation | Technical | Low | Low | Exact versions recorded; citations anchor to current source | Monitored |
| `file:line` citations can drift if source is later edited | Technical | Low | Medium | Citations also name the function/method, aiding relocation | Accepted |
| **Q2 product defect:** unhandled `AttributeError` → `HTTP 500` on browser-session sudo path | Security | Medium | N/A (doc) | Documented with traceback + root cause + safety proof; recommend product-team triage (out-of-scope to fix here) | Reported (not fixed) |
| Leakage of reusable auth material captured during investigation | Security | Low | Low | HMAC sigs, API-key codes, `csrf_token`/`_id` redacted; all live material invalidated at teardown (Redis `FLUSHALL`, keys deleted) | Mitigated |
| Dev credentials shown (`john@wick.com/password`, `FLASK_SECRET=secret`, DB creds) | Security | Low | Low | Published local-dev defaults, not prod secrets; classified with rationale in the doc | Accepted |
| Full reproduction needs whole stack + temp `.env` (Q5) + >600s wait (Q6) | Operational | Low | Medium | Each Q embeds its exact command; Section 9 documents the full procedure incl. cleanup | Mitigated |
| Q5 temporarily edits the git-ignored `.env`; drift if not restored | Operational | Low | Low | Exact byte-identical revert shown (`diff` empty) | Mitigated |
| No third-party integration (`NOT_SEND_EMAIL=true`; Q5 → local sink) | Integration | Low | Low | Fully local, self-contained investigation | N/A |
| Merge conflict risk on the deliverable | Integration | Low | Low | Single additive Markdown file; base diff = 1 file | Mitigated |

**Overall risk posture: LOW.** The single Medium-severity item is a product-level defect that is correctly documented and intentionally out-of-scope to remediate under the read-only mandate.

---

## 7. Visual Project Status

**Project hours breakdown** (Completed = `#5B39F3`, Remaining = `#FFFFFF`):

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieLegendTextColor':'#B23AF2','pieStrokeWidth':'2px'}}}%%
pie showData title Project Hours (Total 43h)
    "Completed Work" : 39
    "Remaining Work" : 4
```

**Remaining work by priority** (High = `#5B39F3`, Medium = `#A8FDD9`):

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#A8FDD9','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieLegendTextColor':'#B23AF2','pieStrokeWidth':'2px'}}}%%
pie showData title Remaining Work by Priority (Total 4h)
    "High — Technical review" : 3
    "Medium — PR merge" : 1
```

**Completed hours by area** (bar):

```mermaid
%%{init: {'theme':'base','themeVariables':{'xyChartBarColor':'#5B39F3'}}}%%
xychart-beta
    title "Completed Hours by Area"
    x-axis ["Env/Setup", "Q1-Q8 Investigation", "Authoring", "Cleanup", "Validation"]
    y-axis "Hours" 0 --> 25
    bar [5, 22.5, 7, 2, 4.5]
```

> Area totals: Env/Setup 5.0 · Q1–Q8 investigation 22.5 (1.5+3.5+2.5+2+4+3+2+2) · Authoring 7.0 · Cleanup 2.0 · Validation 4.5 = **39.0 h**.

**Integrity check:** "Remaining Work" = **4 h** here, in Section 1.2, and in Section 2.2 — identical across all three. ✓

---

## 8. Summary & Recommendations

**Achievements.** The project delivers a rigorous, evidence-backed answer document covering all eight runtime-behavior questions. Every answer is grounded in real captured output from the canonical stack, cross-referenced to a `file:line` location, with observed values cleanly separated from inferred ones. The investigation exercised the *real* entry points (gunicorn web app, standalone SMTP handler, PostgreSQL, Redis), used the canonical dev credentials, and honored a strict read-only posture — the git diff is exactly one added file.

**Remaining gaps.** With **90.7%** of AAP-scoped hours complete (39 of 43), the only remaining work is path-to-production human validation: a technical review of the document (with a recommended re-run spot-check of Q2 and Q6) and the PR merge — **4 hours** total.

**Critical path to production.** (1) Human technical review & acceptance → (2) PR review & merge to `origin/app_2cd6ee777f8c`. There is no build/deploy step because the artifact *is* the deliverable.

**Notable finding.** Q2 surfaced a genuine product defect: a browser-session call to a `@require_api_sudo` endpoint dereferences `None` (`g.api_key`) and raises an unhandled `AttributeError`, yielding `HTTP 500` instead of a clean `440 "Need sudo"`. This is documented with a full traceback and correctly reported **as-found, not fixed**, per the read-only mandate. It is recommended for a separate product-team backlog ticket.

**Success metrics.**

| Metric | Target | Actual |
|--------|--------|--------|
| Questions answered with runtime evidence | 8/8 | 8/8 ✅ |
| Autonomous validation pass rate | 100% | 100% (8/8) ✅ |
| Source files modified (read-only) | 0 | 0 ✅ |
| `file:line` citations | High | 82 ✅ |
| Working tree after cleanup | Clean | Clean ✅ |

**Production readiness assessment.** The deliverable is **production-ready pending human sign-off**. The content is complete, accurate on re-validation, compliant with every governing rule, and cleanly committed. Consistent with best practice, completion is capped below 100% to reserve the final human review gate.

---

## 9. Development Guide

> Reproduction runs **inside the canonical container** (image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`, id `ea242796bbce`), which ships SimpleLogin's pinned **Python 3.10** stack in `/app/venv`. The assessment host (Python 3.13) cannot host the pinned stack directly. Poetry is absent in the image; call venv binaries directly.

### 9.1 System Prerequisites

- Docker (to run the canonical container) — verified: `Docker version 28.5.2`.
- Inside the container: Python **3.10.18**, PostgreSQL **15** (`:5432`), Redis **7.0.15** (`:6379`).
- Observation tools (present in the image; **not** SimpleLogin dependencies): `curl`, `redis-cli`, `psql`, `swaks`.

### 9.2 Environment Setup

```bash
# All commands below run INSIDE the canonical container, from /app.
cd /app

# Confirm the pinned interpreter and repo state
/app/venv/bin/python --version          # -> Python 3.10.18
git -C /app rev-parse HEAD               # -> 2cd6ee777f8c... (detached HEAD)

# Confirm the relevant .env values (‘.env’ is a copy of example.env and is git-ignored)
grep -nE '^(URL|NOT_SEND_EMAIL|EMAIL_DOMAIN|DB_URI|FLASK_SECRET|MEM_STORE_URI)=' /app/.env
# Expected:
#   URL=http://localhost:7777
#   NOT_SEND_EMAIL=true
#   EMAIL_DOMAIN=sl.local
#   DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin
#   FLASK_SECRET=secret
#   MEM_STORE_URI=redis://localhost      # REQUIRED so the server-side RedisSessionStore is active (Q3/Q4)
```

### 9.3 Dependency Installation

Dependencies are pre-installed in `/app/venv` at their exact `poetry.lock` pins. Verify:

```bash
/app/venv/bin/pip freeze | grep -iE '^(Flask|Flask-Login|Werkzeug|itsdangerous|redis|SQLAlchemy|psycopg2-binary|arrow|bcrypt|gunicorn|aiosmtpd|alembic|Flask-Limiter)=='
# Flask==1.1.2  Flask-Login==0.5.0  Werkzeug==1.0.1  itsdangerous==1.1.0
# redis==4.6.0  SQLAlchemy==1.3.24  psycopg2-binary==2.9.3  arrow==0.16.0
# bcrypt==3.2.0  gunicorn==20.0.4  aiosmtpd==1.4.2  alembic==1.4.3  Flask-Limiter==1.4
```

### 9.4 Application Startup

```bash
# 1) Migrate the DB and seed the dev account (idempotent; DB ships at head 32f25cbf12f6)
cd /app && /app/venv/bin/alembic upgrade head && /app/venv/bin/flask dummy-data

# 2) Start the web app under gunicorn — IMPORTANT: debug OFF yields production-accurate
#    HTTP responses (required for Q2; Flask's dev debugger would mask the real HTTP 500).
cd /app && FLASK_APP=server.py /app/venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 30 > /tmp/inv/gunicorn.log 2>&1 &

# 3) Start the standalone SMTP email handler (:20381)
cd /app && FLASK_APP=server.py /app/venv/bin/python email_handler.py > /tmp/inv/eh.log 2>&1 &
```

### 9.5 Verification Steps

```bash
# Ports up
nc -z -w2 localhost 7777 && echo "7777 OPEN"
nc -z -w2 localhost 20381 && echo "20381 OPEN"

# Dev account present
PGPASSWORD=mypassword psql -h localhost -U myuser -d simplelogin \
  -c "SELECT id,email,activated FROM users WHERE email='john@wick.com';"   # -> 1 | john@wick.com | t

# Authenticate (CSRF-aware) and store the slapp cookie
JAR=/tmp/inv/auth_jar.txt
CSRF=$(curl -s -c "$JAR" http://localhost:7777/auth/login \
  | grep -o 'name="csrf_token"[^>]*value="[^"]*"' | grep -o 'value="[^"]*"' | cut -d'"' -f2)
curl -s -i -c "$JAR" -b "$JAR" \
  --data-urlencode "email=john@wick.com" --data-urlencode "password=password" \
  --data-urlencode "csrf_token=$CSRF" http://localhost:7777/auth/login \
  | grep -iE '^HTTP|^Location'          # -> HTTP/1.1 302 FOUND ; Location: .../dashboard/
```

### 9.6 Example Usage (reproduce Q1)

```bash
# Session cookie only, NO Authentication header -> session fallback
curl -sS -i -b /tmp/inv/auth_jar.txt http://localhost:7777/api/user_info
# -> HTTP/1.1 200 OK ; {"email":"john@wick.com","is_premium":true, ...}

# Contrast: no cookie, no key
curl -sS -i http://localhost:7777/api/user_info
# -> HTTP/1.1 401 UNAUTHORIZED ; {"error":"Wrong api key"}
```

### 9.7 Verify the Deliverable (runs anywhere with the repo)

```bash
test -f blitzy/documentation/app_2cd6ee777f8c.md && wc -l blitzy/documentation/app_2cd6ee777f8c.md   # 1497
git diff --name-status origin/app_2cd6ee777f8c...HEAD    # A  blitzy/documentation/app_2cd6ee777f8c.md
git status --porcelain                                   # (empty = clean tree)
grep -nE '^## Q[0-9]' blitzy/documentation/app_2cd6ee777f8c.md   # Q1..Q8 present
```

### 9.8 Troubleshooting

- **Q2 returns an HTML debugger page instead of `500` JSON** → the web app was started in Flask debug mode. Always run under **gunicorn** (as in 9.4) so `error_handler()` returns `{"error":"Internal error"}` with status `500`.
- **Q3/Q4 show no `session:*` keys in Redis** → `MEM_STORE_URI` is unset; ensure `.env` has `MEM_STORE_URI=redis://localhost` so `RedisSessionStore` replaces Flask's default cookie session.
- **Q5 forwarded message not captured** → dev config uses `NOT_SEND_EMAIL=true` (prints only). Temporarily comment it and set `POSTFIX_SERVER=localhost` / `POSTFIX_PORT=<sink port>`, restart `email_handler.py`, then **restore `.env` byte-identical** afterward.
- **Q6 token still valid after waiting** → the window is 600s; wait past it (test at ~607s) and use a **distinct alias prefix** so a `412` cannot be confused with a `409` duplicate.
- **Cleanup** → delete investigation DB rows (FK-safe), `redis-cli FLUSHALL`, restore `.env`, stop processes by their exact pid, and `rm -rf /tmp/inv` so the tree stays clean.

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---------|---------|
| Migrate + seed | `cd /app && /app/venv/bin/alembic upgrade head && /app/venv/bin/flask dummy-data` |
| Start web (gunicorn) | `FLASK_APP=server.py /app/venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 30` |
| Start SMTP handler | `FLASK_APP=server.py /app/venv/bin/python email_handler.py` |
| Login (get cookie) | `curl -s -c jar /auth/login` → extract `csrf_token` → `curl -s -b jar -c jar --data-urlencode ... /auth/login` |
| Q1 session fallback | `curl -sS -i -b jar http://localhost:7777/api/user_info` |
| Q3 Redis inspect | `redis-cli TYPE session:<uuid>` · `redis-cli TTL session:<uuid>` |
| Q5 send test email | `swaks --to e1@sl.local --from hey@google.com --server 127.0.0.1:20381` |
| Q7 DB row read | `psql -h localhost -U myuser -d simplelogin -x -c "SELECT ... FROM api_key WHERE id=<id>;"` |
| Read-only check | `git diff --name-status origin/app_2cd6ee777f8c...HEAD` |

### B. Port Reference

| Port | Service |
|------|---------|
| 7777 | SimpleLogin web app (gunicorn) |
| 20381 | Standalone SMTP `email_handler.py` |
| 5432 | PostgreSQL 15 (db `simplelogin`) |
| 6379 | Redis 7.0.15 (session store + rate limiter) |

### C. Key File Locations

| Path | Role |
|------|------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The deliverable** (single added file) |
| `app/api/base.py` | Q1 session fallback; Q2 `require_api_sudo`/`440`; Q7 `times`/`last_used` |
| `app/session.py` | Q3 pickle serialization + `session:<uuid4>`; Q4 identifier reuse |
| `email_handler.py` | Q5 forward-path header whitelist + `Reply-To`/`From` rewrite |
| `app/alias_suffix.py` | Q6 `TimestampSigner.unsign(..., max_age=600)` |
| `app/models.py` | Q7 `ApiKey` model (`times`, `last_used`, `sudo_mode_at`) |
| `app/auth/views/login.py` | Q8 failed-login flash + `LoginEvent` + `200` re-render |
| `server.py` / `app/config.py` / `app/extensions.py` | Session cookie config, Redis wiring, `SESSION_COOKIE_NAME="slapp"` |

### D. Technology Versions

| Component | Version |
|-----------|---------|
| Python | 3.10.18 |
| Flask / Flask-Login / Werkzeug | 1.1.2 / 0.5.0 / 1.0.1 |
| itsdangerous | 1.1.0 |
| redis-py / SQLAlchemy / psycopg2-binary | 4.6.0 / 1.3.24 / 2.9.3 |
| arrow / bcrypt / gunicorn / aiosmtpd | 0.16.0 / 3.2.0 / 20.0.4 / 1.4.2 |
| PostgreSQL / Redis | 15 / 7.0.15 |

### E. Environment Variable Reference

| Variable | Value | Relevance |
|----------|-------|-----------|
| `URL` | `http://localhost:7777` | Web app base URL |
| `NOT_SEND_EMAIL` | `true` | Dev prints instead of sending mail (temporarily disabled for Q5) |
| `EMAIL_DOMAIN` | `sl.local` | Alias domain used in Q5 |
| `DB_URI` | `postgresql://myuser:mypassword@localhost:5432/simplelogin` | PostgreSQL connection |
| `FLASK_SECRET` | `secret` | Session HMAC + `CUSTOM_ALIAS_SECRET` derivation (Q3/Q6) |
| `MEM_STORE_URI` | `redis://localhost` | **Enables** the server-side `RedisSessionStore` (Q3/Q4) |

### F. Developer Tools Guide

- **`curl`** — HTTP observation for Q1, Q2, Q6, Q7, Q8 (use `-i` for headers, `-b/-c` for the cookie jar).
- **`redis-cli`** — `TYPE`/`TTL`/`GET` on `session:<uuid>` for Q3/Q4; `FLUSHALL` at teardown.
- **`psql`** — read `users` and `api_key` rows for Q2/Q7 (use `-x` expanded mode for full rows).
- **`swaks`** — send the real inbound test email for Q5 to `127.0.0.1:20381`.
- **`aiosmtpd`** — pinned project dep; also used in Q5 to stand up a throwaway local capture sink.
- **`git`** — `diff --name-status` / `status --porcelain` to prove read-only compliance.

### G. Glossary

| Term | Meaning |
|------|---------|
| `slapp` | SimpleLogin session cookie name (`app/config.py:L199`); value is `<uuid4>.<hmac-sig>` |
| `RedisSessionStore` | Bespoke server-side session interface storing pickled dicts under `session:<uuid4>` |
| `require_api_sudo` | Decorator gating privileged API ops; returns `440 "Need sudo"` when sudo inactive |
| Reverse-alias | Rewritten `From`/`Reply-To` address enabling replies to route back through SimpleLogin |
| `440` | Non-standard HTTP status ("Need sudo"); Werkzeug reason phrase renders as `UNKNOWN` |
| Signed suffix | `TimestampSigner`-signed alias suffix valid for 600s (`app/alias_suffix.py:L40`) |
| OBSERVED / INFERRED | Values seen at runtime vs. values derived from source and labeled accordingly |

---

*Cross-section integrity verified: Remaining hours = **4** in Sections 1.2, 2.2, and 7 (identical). Section 2.1 (39) + Section 2.2 (4) = **43** = Total Project Hours (Section 1.2). Completion = 39 ÷ 43 = **90.7%**, consistent across Sections 1.2, 7, and 8. All Section 3 tests originate from Blitzy's autonomous validation logs. Colors: Completed `#5B39F3`, Remaining `#FFFFFF`.*