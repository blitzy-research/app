# Blitzy Project Guide — SimpleLogin Empirical Investigation (Q1–Q8)

> Branch: `blitzy-e6bd4ef2-3871-4201-bbb0-d8a73e55fcc7` · Base: `app_2cd6ee777f8c` (`2cd6ee77`) · HEAD: `17a712a6`
> Rule set: `SWE-AtlasQnA-Repo` · Task type: Read-only empirical documentation

---

## 1. Executive Summary

### 1.1 Project Overview

This project answers **eight behavioral questions (Q1–Q8)** about the SimpleLogin email-aliasing Flask application by *running* the relevant code paths against a live runtime and capturing verbatim output — not by reading code alone. It spans four subsystems: API authentication/authorization, server-side Redis session storage, the inbound email-forwarding header pipeline, and signed-token expiry plus API-key usage tracking. The sole deliverable is one Markdown document, `blitzy/documentation/app_2cd6ee777f8c.md`, that pairs each answer with its exact observed output, a `file:line` citation, and rationale. The task is strictly **read-only**: no application source or dependency was changed. The audience is engineers who need verified, evidence-grounded knowledge of these behaviors.

### 1.2 Completion Status

The completion percentage is calculated using the AAP-scoped hours methodology (completed hours ÷ total hours). All completed work was performed autonomously by Blitzy agents; the only remaining activity is human review and acceptance of the deliverable.

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieTitleTextSize':'16px'}}}%%
pie showData
    title Completion — 94.9% (AAP-scoped)
    "Completed Work (AI)" : 37
    "Remaining Work" : 2
```

| Metric | Value |
|---|---|
| **Total Hours** | **39** |
| **Completed Hours (AI + Manual)** | **37** (37 AI + 0 Manual) |
| **Remaining Hours** | **2** |
| **Percent Complete** | **94.9%** |

> Legend — Completed Work `#5B39F3` · Remaining Work `#FFFFFF`

### 1.3 Key Accomplishments

- ✅ Full multi-service runtime stood up for observation: Flask/gunicorn (`wsgi:app` @ `127.0.0.1:7777`), PostgreSQL 13 (`:15432`, migrated to head `32f25cbf12f6`), Redis 6 (`:6379`, active `RedisSessionStore`), and the in-process `email_handler.handle` pipeline.
- ✅ All **8 questions answered with verbatim runtime evidence**, exact `file:line` citations (~101 references; 96 genuine repo-relative citations verified in-bounds), and rationale.
- ✅ Every answer **empirically re-verified** against a fresh runtime by the final validator — all 8 matched exactly (only per-run values such as timestamps/UUIDs differed).
- ✅ **Surprising results reported faithfully:** Q2 returns HTTP `500 {"error":"Internal error"}` via an `AttributeError`; Q4 shows the session identifier is **not** rotated across login.
- ✅ **Read-only rule fully satisfied:** `git diff` versus base shows exactly **1 file added** (+799/−0), zero source files modified, no dependency changes; working tree clean.
- ✅ Autonomous test validation: **639-test** full-suite baseline passing; **35-test** focused re-verification subset across all 8 subsystems passing (0 failures).

### 1.4 Critical Unresolved Issues

No issues block release or acceptance of the deliverable. The two application anomalies surfaced by the investigation are **findings that are intentionally documented and not fixed** (the read-only rule and AAP §0.3.2 place fixes out of scope); they are not defects in the delivered document.

| Issue | Impact | Owner | ETA |
|---|---|---|---|
| *(None blocking the deliverable)* | — | — | — |
| App finding — Q2: `DELETE /api/user` with a cookie-only session raises `AttributeError` → HTTP 500 (documented, out of scope to fix) | Informational; robustness gap on a sudo endpoint in the app under study | SimpleLogin maintainers | Out of scope (advisory) |
| App finding — Q4: session identifier not rotated on login (documented, out of scope to fix) | Informational; session-fixation hardening opportunity | SimpleLogin maintainers | Out of scope (advisory) |

### 1.5 Access Issues

**No access issues identified.** The repository is committed and clean; the runtime is reproducible from the provided container image and `tests/test.env` configuration; PostgreSQL (`:15432`) and Redis (`:6379`) were reachable throughout the investigation.

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|---|---|---|---|---|
| Git repository | Read/Write | None | ✅ Resolved | — |
| PostgreSQL 13 / Redis 6 | Service access | None (both reachable) | ✅ Resolved | — |
| Container image / dependencies | Runtime | None (image + `poetry.lock` pinned) | ✅ Resolved | — |

### 1.6 Recommended Next Steps

1. **[High]** Perform the technical review and acceptance of `blitzy/documentation/app_2cd6ee777f8c.md`: read the eight answers, spot-verify a sample of the verbatim-output claims and `file:line` citations, confirm the original questions are fully answered, and approve/merge the PR. *(≈2 h — the only remaining in-scope work.)*
2. **[Medium]** *(Advisory, outside this read-only deliverable's scope)* Route the **Q2** finding to the SimpleLogin maintainers — add a null-`api_key` guard in `check_sudo_mode_is_active` (`app/api/base.py:47`) so a sudo call without an API key returns a proper `401/403/440` instead of a `500`.
3. **[Low]** *(Advisory)* Route the **Q4** finding to maintainers — consider rotating the session identifier on login as session-fixation hardening.
4. **[Low]** *(Optional)* Independently reproduce the observations using the Development Guide (Section 9) for extra assurance; not required for acceptance.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

All hours below were completed autonomously by Blitzy agents and trace to AAP requirements or path-to-production validation.

| Component | Hours | Description |
|---|---:|---|
| Runtime environment & service orchestration | 5.0 | Python 3.10 `.venv`; PostgreSQL 13 (`:15432`) + Redis 6 (`:6379`); `tests/test.env` config; `flask db upgrade` → head `32f25cbf12f6`; `RedisSessionStore` activation via `MEM_STORE_URI`. |
| Application entry points | 2.0 | Gunicorn `wsgi:app` @ `127.0.0.1:7777`; in-process `email_handler.handle` pipeline for the forwarding question. |
| Seed data provisioning | 1.5 | Activated User, Alias, and ApiKey created + committed via the `tests/conftest.py` / `init_app.py` pattern. |
| Q1 — Cookie-session API call | 1.5 | `GET /api/user_info` with no `Authentication` header → HTTP 200 + 8-key JSON body. |
| Q2 — Sudo op with browser session | 2.0 | `DELETE /api/user` → HTTP 500 `{"error":"Internal error"}` via `AttributeError`; captured server log + traceback. |
| Q3 — Redis session storage format | 2.5 | Raw bytes (len 300, `0x80`/`\x80\x04`), pickle protocol 4, 6 deserialized keys, `session:<uuid>` layout, TTL 604800. |
| Q4 — Session-fixation behavior | 2.0 | Session identifier decoded before/after login via the app's cookie signer; identifier unchanged. |
| Q5 — Email-forwarding header fate | 3.0 | Crafted email through the forward pipeline; custom `X-` + `Received` + original `Reply-To` stripped, `Reply-To` replaced by reverse-alias. |
| Q6 — Alias-suffix token expiry | 2.0 | `TimestampSigner` sign → validate immediately → past-boundary; exact 600-second window and `SignatureExpired` message. |
| Q7 — API-key usage statistics | 2.0 | Five authenticated calls; `times` 0→5, `last_used`/`updated_at` set, `sudo_mode_at` unchanged; before/after DB reads. |
| Q8 — Failed-login behavior | 1.5 | Wrong-password `POST /auth/login` → HTTP 200 + flash `Email or password incorrect`; log lines + `10/minute` rate limit. |
| Answer document authoring | 5.0 | ~800-line document: intro, versions table, eight formatted sections, final coverage pass; two commits (`7ec919a0`, `17a712a6`). |
| Read-only compliance & cleanup | 1.0 | Temp scripts kept in `/tmp/qna_probes` outside the repo; `git status` verified clean; exactly 1 file added. |
| Final empirical validation | 6.0 | 5 production-readiness gates; full runtime re-standup; re-verification of all 8 answers; 99-citation audit; 35-test subset + 639-test baseline. |
| **Total Completed** | **37.0** | |

### 2.2 Remaining Work Detail

Remaining work is limited to the single path-to-production activity for a documentation deliverable — human review and acceptance. Fixing the documented app anomalies and any code deployment are **out of AAP scope** (read-only task, no runtime to ship) and are therefore excluded from the completion math; they appear as advisory next steps in Section 1.6.

| Category | Hours | Priority |
|---|---:|---|
| Human review & acceptance of the answer document (read 8 answers, spot-verify claims/citations, confirm coverage of the original questions, approve/merge PR) | 2.0 | High |
| **Total Remaining** | **2.0** | |

---

## 3. Test Results

All tests below originate from Blitzy's autonomous validation logs for this project (framework **pytest 7.3.1**, Python 3.10.20, `CONFIG=tests/test.env`). Because the task is read-only documentation, no new tests were authored; the existing suite was used to confirm the cited code paths behave as documented. The subset and independent runs are **subsets of the full baseline**, not additive universes.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---|---|---:|---:|---:|---:|---|
| Full regression baseline | pytest 7.3.1 | 639 | 639 | 0 | Suite-wide | Autonomous full-suite baseline (setup logs). |
| Focused re-verification subset (all 8 subsystems) | pytest 7.3.1 | 35 | 35 | 0 | n/a (subset) | Validator GATE 1 — covers `api` user_info/user/sudo/auth, `auth` login, `api_to_cookie` session, `alias_suffixes`. |
| Alias-suffix expiry (Q6) — independent confirmation | pytest 7.3.1 | 5 | 5 | 0 | n/a (subset) | Re-run during this guide: `tests/test_alias_suffixes.py` → 5 passed in 1.45 s (exit 0). Part of the 35-test subset. |

> **Coverage note.** Running a *subset* trips the repository's suite-wide coverage gate (an addopts threshold). This is an expected artifact of subset execution — **not** a test failure — and is avoided with `-o addopts=""`. Zero test failures were observed at any tier.

---

## 4. Runtime Validation & UI Verification

**Runtime health (all exercised live during the investigation and re-verified):**

- ✅ **Operational** — Flask/gunicorn `wsgi:app` @ `127.0.0.1:7777` (health probe `GET /auth/login` → 200).
- ✅ **Operational** — PostgreSQL 13 @ `127.0.0.1:15432`, schema at head `32f25cbf12f6`.
- ✅ **Operational** — Redis 6 @ `127.0.0.1:6379`; `app.session_interface` confirmed as `RedisSessionStore`.
- ✅ **Operational** — Inbound `email_handler.handle(envelope, msg)` pipeline (Q5); forward result `250`.
- ✅ **Operational** — `itsdangerous.TimestampSigner` alias-suffix path (Q6); sign → expire transition observed.

**API integration outcomes (behavior confirmed exactly as documented):**

- ✅ `GET /api/user_info` (cookie session, no API key) → **200**, 8-key JSON (Q1).
- ✅ `DELETE /api/user` (cookie session) → **500** `{"error":"Internal error"}` — documented anomaly reported as observed (Q2).
- ✅ `POST /auth/login` (wrong password) → **200**, flash `Email or password incorrect` (Q8).
- ✅ `ApiKey` usage columns update as documented across 5 calls (Q7).

**UI verification:** ⚠ **Not applicable.** This is a read-only, back-end investigation with no user-interface component; no Figma frames or frontend changes were in scope (AAP §0.9). No UI verification was required or performed.

---

## 5. Compliance & Quality Review

The deliverable is cross-mapped to the binding `SWE-AtlasQnA-Repo` rule set and Blitzy quality benchmarks. The review-findings commit `17a712a6` (DOC-001..005) refined the document during autonomous validation; the final validator found no further fixes were required.

| Requirement / Benchmark | Status | Progress | Evidence / Notes |
|---|---|---|---|
| Deliverable at correct path & name | ✅ Pass | 100% | `blitzy/documentation/app_2cd6ee777f8c.md` (name derived from source branch). |
| Investigate by running the code (not reading) | ✅ Pass | 100% | Every question has an "Observed output (verbatim)" block from a live run. |
| Quote actual observed output verbatim | ✅ Pass | 100% | HTTP statuses, headers, raw bytes, log lines, timings pasted verbatim. |
| One claim, one piece of evidence | ✅ Pass | 100% | Each "Answer" bullet pairs a claim with its specific observed line. |
| Answer every part & every named item | ✅ Pass | 100% | Final coverage pass confirms all Q5 headers, all `ApiKey` fields, Q4 before/after, all status codes. |
| Exact & grounded (`file:line` citations) | ✅ Pass | 100% | ~101 references; 96 genuine repo-relative citations verified in-bounds across 23 files. |
| Report unexpected results faithfully | ✅ Pass | 100% | Q2 500, Q4 preserved id, Q7 auto `updated_at`, Q8 single access line reported as observed. |
| Read-only scope (no source/dep changes) | ✅ Pass | 100% | `git diff 2cd6ee77..HEAD` = 1 file added, +799/−0; zero source/dependency changes. |
| Temp scripts removed; tree clean | ✅ Pass | 100% | Scripts in `/tmp/qna_probes` outside repo; `git status` clean. |
| Cited modules compile cleanly (quality) | ✅ Pass | 100% | 19 cited source modules `py_compile` OK. |

---

## 6. Risk Assessment

Because Blitzy produced **no code** (read-only documentation), risks concern (a) documentation durability and (b) app-level findings the investigation surfaced — not defects in the delivered work.

| Risk | Category | Severity | Probability | Mitigation | Status |
|---|---|---|---|---|---|
| Citation line-number drift as upstream evolves | Technical | Low | Medium | Document is a point-in-time snapshot pinned to branch `app_2cd6ee777f8c`; header states the branch. | Accepted |
| Per-run value variance (timestamps/UUIDs/PIDs/IDs) | Technical | Low | Low | Ephemeral values labeled; only per-run values differ, no behavioral change. | Resolved |
| Q2 — `AttributeError` → 500 on `DELETE /api/user` (app finding) | Security | Medium | Low | Root cause + fix location documented (`app/api/base.py:47`); route to maintainers. Out of scope to fix. | Open (documented) |
| Q4 — no session-id rotation on login (app finding) | Security | Medium | Low | Documented; route to maintainers for session-fixation hardening. Out of scope to fix. | Open (documented) |
| Reproducibility needs full multi-service runtime | Operational | Low | Medium | "Environment & Runtime" section gives exact versions, config, startup commands, and image. | Mitigated |
| Ephemeral seed data not committed | Operational | Low | Low | Seeding pattern documented (`conftest`/`init_app`). | Accepted |
| Re-verification depends on external services/image + pinned deps | Integration | Low | Low | Exact versions and `poetry.lock` pins recorded in the document. | Mitigated |
| No CI/CD/deploy pipeline for a static doc | Integration | N/A | N/A | Deliverable is a Markdown file; traditional deploy risk does not apply. | N/A |

**Overall risk posture: LOW.** The deliverable itself carries negligible risk (complete, validated, committed clean). Residual items are point-in-time documentation drift (accepted) and two app-level findings correctly documented-not-fixed per the read-only mandate.

---

## 7. Visual Project Status

**Project hours breakdown** (Completed `#5B39F3` · Remaining `#FFFFFF`):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieTitleTextSize':'16px'}}}%%
pie showData
    title Project Hours — Completed vs Remaining
    "Completed Work" : 37
    "Remaining Work" : 2
```

**Remaining hours by category (Section 2.2):**

| Category | Hours | Bar |
|---|---:|---|
| Human review & acceptance | 2.0 | ████ |
| **Total** | **2.0** | |

**Completed hours by phase group:**

| Phase Group | Hours | Bar |
|---|---:|---|
| Runtime, entry points & seed data | 8.5 | █████████ |
| Per-question investigations (Q1–Q8) | 16.5 | █████████████████ |
| Document authoring | 5.0 | █████ |
| Compliance/cleanup + final validation | 7.0 | ███████ |
| **Total** | **37.0** | |

> Integrity: Section 7 "Remaining Work" (2) = Section 1.2 Remaining (2) = Section 2.2 total (2.0). "Completed Work" (37) = Section 2.1 total (37.0). 37 + 2 = 39 total.

---

## 8. Summary & Recommendations

**Achievements.** The project delivered a complete, evidence-grounded answer to all eight behavioral questions about SimpleLogin, produced entirely by running the real code paths against a live runtime. Each answer pairs a verbatim observation with an exact `file:line` citation and rationale, and every answer was independently re-verified against a fresh runtime with exact matches. The read-only mandate was honored precisely: a single documentation file was added and no source or dependency changed.

**Remaining gaps & critical path to production.** The project is **94.9% complete** (37 of 39 hours). The only remaining activity is **human review and acceptance** of the deliverable (≈2 hours) — the critical path is simply a technical read-through, a spot-check of a sample of the verbatim claims and citations, and PR approval. There is no code to deploy and no environment to provision for release, because the deliverable is a documentation artifact.

**Success metrics (all met).** Correct deliverable path ✔ · all 8 questions answered with verbatim evidence ✔ · ~101 citations with 96 genuine references verified in-bounds ✔ · read-only tree clean (1 file added) ✔ · 639-test baseline + 35-test subset passing ✔ · surprising results reported faithfully ✔.

**Production readiness assessment.** **Ready for review.** The document is accurate, complete, correctly formatted, and committed. Two application-level findings (Q2 `AttributeError`→500; Q4 no session-id rotation) are surfaced as advisory items for the SimpleLogin maintainers; addressing them is explicitly outside this read-only task and does not affect the deliverable's readiness.

| Metric | Value |
|---|---|
| Completion | 94.9% (37 / 39 h) |
| Blocking issues | 0 |
| Files changed vs base | 1 added (+799 / −0) |
| Autonomous tests passing | 639 baseline / 35 subset |
| Overall risk | Low |

---

## 9. Development Guide

This guide reproduces the runtime used for the investigation. All commands were tested in the project's container/host environment. Run from the repository root unless noted.

### 9.1 System Prerequisites

- **OS:** Linux (container image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`).
- **Python:** 3.10 (verified `3.10.20`).
- **Services:** PostgreSQL **13**, Redis **6** (Docker `postgres:13`, `redis:6`).
- **Tooling:** Docker Engine, Git, and either Poetry or the bundled `.venv`.
- **Disk/RAM:** ~2 GB free; 2 GB RAM comfortably sufficient.

### 9.2 Environment Setup

```bash
# From the repository root
cd /path/to/repo

# Use the pre-built virtualenv (Python 3.10.20) …
.venv/bin/python --version        # -> Python 3.10.20

# … or create/populate one with Poetry (poetry.lock-pinned):
# poetry env use 3.10 && poetry install

# The investigation configuration (activates the Redis session store):
export CONFIG=tests/test.env
grep -E 'DB_URI|FLASK_SECRET|MEM_STORE_URI|^URL|NOT_SEND_EMAIL' tests/test.env
# DB_URI=postgresql://test:test@localhost:15432/test
# FLASK_SECRET=secret
# MEM_STORE_URI=redis://localhost      # <- makes RedisSessionStore the active session backend
# URL=http://localhost
# NOT_SEND_EMAIL=true
```

### 9.3 Dependency Installation

```bash
# Preferred: reuse the bundled, poetry.lock-pinned virtualenv (no install needed).
.venv/bin/python - <<'PY'
import importlib.metadata as m
for p in ["flask","werkzeug","flask-login","flask-limiter","itsdangerous","redis","aiosmtpd","arrow","gunicorn","pytest"]:
    print(f"{p}=={m.version(p)}")
PY
# Expected: flask==1.1.2 werkzeug==1.0.1 flask-login==0.5.0 flask-limiter==1.4
#           itsdangerous==1.1.0 redis==4.6.0 aiosmtpd==1.4.2 arrow==0.16.0
#           gunicorn==20.0.4 pytest==7.3.1
```

### 9.4 Application Startup

```bash
# 1) Ensure PostgreSQL 13 and Redis 6 are running (Docker), matching the CI stack:
#    docker run -d --name sl-postgres -p 15432:5432 -e POSTGRES_USER=test \
#      -e POSTGRES_PASSWORD=test -e POSTGRES_DB=test postgres:13
#    docker run -d --name sl-redis -p 6379:6379 redis:6

# 2) Apply the schema (migrates to head 32f25cbf12f6):
CONFIG=tests/test.env .venv/bin/flask db upgrade

# 3) Start the web app (gunicorn against wsgi:app):
CONFIG=tests/test.env PYTHONPATH=$PWD .venv/bin/gunicorn wsgi:app \
    -b 127.0.0.1:7777 --workers 1 --timeout 120 --log-level info &

# 4) The inbound SMTP pipeline (Q5) is exercised in-process via
#    email_handler.handle(envelope, msg) with NOT_SEND_EMAIL=true.
```

### 9.5 Verification Steps

```bash
# Services reachable?
python3 - <<'PY'
import socket
for name,host,port in [("PostgreSQL","127.0.0.1",15432),("Redis","127.0.0.1",6379)]:
    s=socket.socket(); s.settimeout(1.5)
    try: s.connect((host,port)); print(f"{name} {host}:{port} -> REACHABLE")
    except Exception as e: print(f"{name} {host}:{port} -> {e.__class__.__name__}")
    finally: s.close()
PY

# Web app health:
curl -s -o /dev/null -w "GET /auth/login -> %{http_code}\n" http://127.0.0.1:7777/auth/login   # -> 200

# Read the deliverable:
sed -n '1,60p' blitzy/documentation/app_2cd6ee777f8c.md

# Focused tests (subset; disable the suite-wide coverage gate to avoid a false FAIL):
CONFIG=tests/test.env .venv/bin/python -m pytest tests/test_alias_suffixes.py \
    -p no:cacheprovider -o addopts="" -q          # -> 5 passed

# Read-only guarantee:
git status --short          # -> empty (clean)
git diff --stat 2cd6ee77..HEAD   # -> 1 file changed, 799 insertions(+)
```

### 9.6 Example Usage (reproduce two answers)

```bash
# Q1 — cookie-session API call (expect HTTP 200 + 8-key JSON):
#   1. requests.Session() logs in via POST /auth/login (obtains 'slapp' cookie)
#   2. GET /api/user_info WITH NO Authentication header
#   -> 200, application/json, keys: can_create_reverse_alias, connected_proton_address,
#      email, in_trial, is_premium, max_alias_free_plan, name, profile_picture_url

# Q6 — alias suffix expiry (expect valid <=600 s, SignatureExpired after):
CONFIG=tests/test.env .venv/bin/python - <<'PY'
from app.config import CUSTOM_ALIAS_SECRET
import itsdangerous
s = itsdangerous.TimestampSigner(CUSTOM_ALIAS_SECRET)
tok = s.sign(b"demo-suffix")
print("immediate:", s.unsign(tok, max_age=600).decode())   # succeeds
try:
    s.unsign(tok, max_age=0)                                # forces expiry
except itsdangerous.SignatureExpired as e:
    print("expired:", e)                                    # "Signature age N > 0 seconds"
PY
```

### 9.7 Troubleshooting

- **Subset test run reports a coverage FAIL.** Expected — the suite-wide coverage gate fires on partial runs. Add `-o addopts=""` (as above). It is not a test failure.
- **Sessions not stored in Redis.** `MEM_STORE_URI` must be set (it is in `tests/test.env`); otherwise `initialize_redis_services` does not install `RedisSessionStore`. Confirm with `type(app.session_interface).__name__ == "RedisSessionStore"`.
- **Port already in use (7777 / 15432 / 6379).** Stop the previous process/container or choose another bind port for gunicorn.
- **`flask db upgrade` cannot connect.** Verify `DB_URI` host/port match the running PostgreSQL container (`localhost:15432`).
- **IPv6 loopback.** If localhost resolution fails, ensure IPv6 loopback is enabled (a setup note re-enabled it via `sysctl`).

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---|---|
| Python version | `.venv/bin/python --version` |
| Dependency versions | `.venv/bin/python -c "import importlib.metadata as m;print(m.version('flask'))"` |
| Apply DB schema | `CONFIG=tests/test.env .venv/bin/flask db upgrade` |
| Start web app | `CONFIG=tests/test.env PYTHONPATH=$PWD .venv/bin/gunicorn wsgi:app -b 127.0.0.1:7777 --workers 1 --timeout 120 --log-level info` |
| Health probe | `curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:7777/auth/login` |
| Focused tests | `CONFIG=tests/test.env .venv/bin/python -m pytest tests/test_alias_suffixes.py -p no:cacheprovider -o addopts="" -q` |
| Read-only check | `git status --short && git diff --stat 2cd6ee77..HEAD` |

### B. Port Reference

| Port | Service | Notes |
|---|---|---|
| 7777 | Flask/gunicorn web app | `URL=http://localhost:7777` (`example.env:6`); bound `127.0.0.1:7777`. |
| 15432 | PostgreSQL 13 | `DB_URI=...localhost:15432/test` (`tests/test.env:17`). |
| 6379 | Redis 6 | `MEM_STORE_URI=redis://localhost` (`tests/test.env:78`). |
| 1025 | Postfix/SMTP (reference) | `POSTFIX_PORT=1025` (`example.env:154`); Q5 uses in-process `email_handler.handle`. |

### C. Key File Locations

| Path | Role |
|---|---|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The sole deliverable** (answer document). |
| `app/api/base.py` | `authorize_request` (Q1), sudo check (Q2), usage-stat update (Q7). |
| `app/api/views/user.py`, `app/api/views/user_info.py` | `DELETE /api/user` (Q2); `GET /api/user_info` (Q1). |
| `server.py` | Global exception → JSON 500 handler (Q2); session config. |
| `app/session.py`, `app/redis_services.py`, `app/extensions.py` | Redis session key layout/serialization/TTL (Q3); session store wiring; `session_protection` (Q4). |
| `app/auth/views/login_utils.py`, `app/auth/views/login.py`, `app/events/auth_event.py` | `after_login` no rotation (Q4); failed-credential branch + rate limit (Q8); New Relic event (Q8). |
| `email_handler.py`, `app/email/headers.py` | Header allowlist, stripping, Reply-To rewrite (Q5). |
| `app/alias_suffix.py` | `max_age=600` suffix expiry (Q6). |
| `app/models.py` | `ApiKey` usage columns (Q7). |
| `tests/conftest.py`, `tests/test.env` | App-bootstrap/seeding pattern; config that activates Redis sessions. |

### D. Technology Versions

| Component | Version | Source of truth |
|---|---|---|
| Python | 3.10.20 | `pyproject.toml:61` (`python = "^3.10"`) |
| flask | 1.1.2 | `poetry.lock` |
| werkzeug | 1.0.1 | `poetry.lock` |
| flask-login | 0.5.0 | `poetry.lock` (`session_protection = "strong"`) |
| flask-limiter | 1.4 | `poetry.lock` (login rate limit `10/minute`) |
| itsdangerous | 1.1.0 | `poetry.lock` (session signer + suffix `TimestampSigner`) |
| redis | 4.6.0 | `poetry.lock` (session/rate-limit client) |
| aiosmtpd | 1.4.2 | `poetry.lock` (inbound SMTP handler) |
| arrow | 0.16.0 | `poetry.lock` (`last_used`, sudo window) |
| gunicorn | 20.0.4 | `poetry.lock` (WSGI server) |
| pytest | 7.3.1 | test runner |
| PostgreSQL server | 13 | Docker `postgres:13`, host port `15432` |
| Redis server | 6 | Docker `redis:6`, host port `6379` |

### E. Environment Variable Reference

| Variable | Value (investigation) | Purpose |
|---|---|---|
| `CONFIG` | `tests/test.env` | Selects the investigation configuration. |
| `DB_URI` | `postgresql://test:test@localhost:15432/test` | PostgreSQL connection. |
| `FLASK_SECRET` | `secret` | Signs the session cookie identifier. |
| `MEM_STORE_URI` | `redis://localhost` | Activates the server-side `RedisSessionStore`. |
| `URL` | `http://localhost` | Base application URL. |
| `NOT_SEND_EMAIL` | `true` | Captures outbound mail instead of delivering (Q5). |
| `DISABLE_RATE_LIMIT` | *(unset)* | Left unset so the `10/minute` login limit is active (Q8). |
| `PYTHONPATH` | `$PWD` | Ensures `wsgi:app` imports resolve for gunicorn. |

### F. Developer Tools Guide

| Tool | Use |
|---|---|
| `gunicorn` | Serve `wsgi:app` for the HTTP questions (Q1, Q2, Q8). |
| `flask db` | Alembic migrations (`upgrade`, `current`) → head `32f25cbf12f6`. |
| `pytest` | Run focused subsets or the full suite; use `-o addopts=""` for partial runs. |
| `redis-cli` / Redis client | Inspect `session:<uuid>` keys and TTLs (Q3, Q4). |
| `psql` | Read the `ApiKey` row before/after calls (Q7). |
| `requests` | Drive cookie-session HTTP calls (Q1, Q2, Q8). |
| `itsdangerous` | Sign/decode session cookie and alias-suffix tokens (Q4, Q6). |

### G. Glossary

| Term | Meaning |
|---|---|
| **AAP** | Agent Action Plan — the primary directive defining scope and deliverables. |
| **Reverse alias** | A generated address that lets a recipient reply through SimpleLogin; substituted into `Reply-To` (Q5). |
| **Sudo mode** | Elevated API state guarded by `require_api_sudo`; the only such endpoint is `DELETE /api/user` (Q2). |
| **`RedisSessionStore`** | SimpleLogin's server-side session backend keyed as `session:<uuid>` with pickle-serialized values (Q3). |
| **`TimestampSigner`** | `itsdangerous` signer used for the alias-suffix token with `max_age=600` seconds (Q6). |
| **Session fixation** | A class of attack mitigated by rotating the session id on login; Q4 shows SimpleLogin does not rotate. |
| **`file:line` citation** | An exact source reference (e.g., `app/api/base.py:47`) grounding each factual claim. |
| **Coverage gate** | A suite-wide pytest addopts threshold that trips on subset runs; bypassed with `-o addopts=""`. |