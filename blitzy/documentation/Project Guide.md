# Blitzy Project Guide — SimpleLogin Runtime-Behavior Investigation (Q1–Q8)

> **Deliverable:** `blitzy/documentation/app_2cd6ee777f8c.md` — a single, evidence-backed markdown document answering eight runtime-behavior questions about SimpleLogin's authentication and email-forwarding subsystems.
> **Brand color key:** Completed / AI Work = **Dark Blue `#5B39F3`** · Remaining / Not Completed = **White `#FFFFFF`**.

---

## 1. Executive Summary

### 1.1 Project Overview

This project is a **read-only empirical investigation-and-documentation task** against the open-source SimpleLogin email-aliasing platform. The objective was to answer eight precise questions (Q1–Q8) about the live runtime behavior of SimpleLogin's authentication, session, API-key, and email-forwarding subsystems — each answer derived from **running** the real code paths and capturing actual, unedited output with `file:line` grounding. The audience is engineers and reviewers who need authoritative, reproducible answers rather than source-reading guesses. The single deliverable is one markdown document. No source code was modified: the value is knowledge capture and behavioral verification, delivered as durable documentation.

### 1.2 Completion Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeWidth':'2px'}}}%%
pie showData title Completion — 93.3% Complete (28h of 30h)
    "Completed Work" : 28
    "Remaining Work" : 2
```

**Center label:** **93.3% Complete**

| Metric | Hours |
|--------|-------|
| **Total Hours** | **30.0** |
| **Completed Hours (AI + Manual)** | **28.0** |
| &nbsp;&nbsp;• Completed by Blitzy agents (AI) | 28.0 |
| &nbsp;&nbsp;• Completed by humans (manual) | 0.0 |
| **Remaining Hours** | **2.0** |
| **Percent Complete** | **93.3%** |

> Completion is computed with the PA1 AAP-scoped hours method: `Completed ÷ (Completed + Remaining) = 28.0 ÷ 30.0 = 93.3%`. The remaining 2.0h is entirely human path-to-production work (technical acceptance review + merge). The two behavioral findings surfaced during the investigation (Q2 error path, Q4 session non-rotation) are **explicitly out of scope** per the AAP and are **excluded** from the hours math — they are reported as product advisories only.

### 1.3 Key Accomplishments

- ✅ **All eight questions (Q1–Q8) answered from live runtime output**, not source reading — each with the exact command, complete unedited output, concrete observed values, and `file:line` citation.
- ✅ **Single deliverable produced** at `blitzy/documentation/app_2cd6ee777f8c.md` (1,173 lines / 7,422 words / ~64 KB, 25 balanced code blocks).
- ✅ **Read-only guarantee upheld** — `git diff` from base `2cd6ee77` to HEAD shows exactly one file added, zero source files touched, clean working tree.
- ✅ **Q2 ambiguity empirically resolved** — captured both the browser-session `HTTP 500` (unhandled `AttributeError` on a `None` API key) and the real-key `HTTP 440 "Need sudo"` paths with full tracebacks.
- ✅ **Timing rigor for Q6** — token expiry boundary established at 600s and confirmed stable across two runs (last valid `t=600.001s`, first expired `t=601.001s`).
- ✅ **Before/after state capture for Q4 and Q7** — session identifier byte-identical across login (not rotated); API-key `times` counter `0 → 5` with `last_used` `NULL → timestamp`.
- ✅ **Independent final validation** — all 8 investigations re-run in the live container; 8/8 reproduced exactly (only environmental diffs: port/UUID/timestamp); 100% of `file:line` citations verified against source.
- ✅ **Coverage pass** confirming every named sub-item of every question is present and answered.

### 1.4 Critical Unresolved Issues

There are **no unresolved issues that block release of the deliverable**. The document is complete, empirically accurate (8/8 verified), fully grounded, and committed. The two items below are **product-behavior findings** the investigation was asked to *report, not fix* (AAP §0.5.2); they do not block acceptance of this documentation deliverable.

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| Q2: `DELETE /api/user` with browser-session-only raises unhandled `AttributeError` → HTTP 500 | Product robustness gap; does **not** delete the account (fails before deletion). Report-only per AAP. | SimpleLogin product team | Backlog (not part of this deliverable) |
| Q4: Session identifier not rotated on login despite `session_protection="strong"` | Session-fixation consideration. Report-only per AAP. | SimpleLogin product team | Backlog (not part of this deliverable) |

### 1.5 Access Issues

**No access issues identified.** All prerequisites for build, runtime validation, and reproduction were available and exercised: the designated Docker container (`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`) is running, PostgreSQL 15 and Redis are healthy inside it, the repository is committed on the working branch, and no third-party credentials or external network services are required by a documentation deliverable.

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-----------------|----------------|-------------------|-------------------|-------|
| Source repository | Read/Write (git) | None — branch checked out, clean tree, deliverable committed | ✅ Resolved / N/A | Blitzy agent |
| Docker container `sl` | Exec/runtime | None — running and healthy (6h uptime) | ✅ Resolved / N/A | Blitzy agent |
| PostgreSQL 15 / Redis | Datastore | None — `pg_isready` accepting, `redis-cli ping → PONG` | ✅ Resolved / N/A | Blitzy agent |

### 1.6 Recommended Next Steps

1. **[High]** Subject-matter-expert **technical review & acceptance** of the eight answers — verify observed values against expectations, confirm evidence completeness, spot-check citations (~1.5h).
2. **[High]** **Review and merge** the pull request adding `blitzy/documentation/app_2cd6ee777f8c.md` to the target branch (~0.5h).
3. **[Medium]** *(Product backlog — out of scope here)* Add a `None`-guard in `check_sudo_mode_is_active` so `DELETE /api/user` with a browser session returns `440` rather than a `500` (~2–3h).
4. **[Medium]** *(Product backlog — out of scope here)* Rotate/regenerate the session identifier on login to close the Q4 session-fixation gap (~3–4h).
5. **[Low]** *(Product backlog — out of scope here)* Review the commented-out `HEADER_ALLOW_API_COOKIES` gate (`app/api/base.py:22-24`) governing the browser-session → API fallback (~2h).

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

Every completed component traces to a specific AAP requirement (runtime establishment, the eight investigations, authoring, remediation, and validation).

| Component | Hours | Description |
|-----------|------:|-------------|
| Runtime environment establishment | 3.0 | Boot SimpleLogin in the Docker container; verify PostgreSQL 15 (alembic head `32f25cbf12f6`) + Redis; set `FLASK_SECRET`; seed verified user `john@wick.com` + API keys; confirm `gunicorn wsgi:app` and test client. |
| Q1 — Session-auth API without key | 1.5 | Drive `GET /api/user_info` with session cookie / no `Authentication` header → HTTP 200 + 8-key JSON; exercise no-cookie 401 error path. |
| Q2 — Privileged op authorization code | 2.0 | Invoke `DELETE /api/user`; resolve 440-vs-500 empirically (browser-session `500` AttributeError; real-key `440 "Need sudo"`) with tracebacks. |
| Q3 — Session storage representation | 2.0 | Read raw Redis bytes; identify pickle proto-4 (300 bytes); enumerate session keys; decode `session:<uuid4>` key structure. |
| Q4 — Session identifier across login | 1.5 | Capture `slapp` cookie before vs after login; unsign and compare; determine identifier not rotated. |
| Q5 — Email-forward header handling | 3.0 | Craft `.eml` with custom `X-*`, `Received`, `Reply-To`; drive `email_handler.py` forward path; capture kept/stripped headers + `250` acceptance. |
| Q6 — Alias-suffix token expiry | 2.0 | Sign/verify suffix token; establish 600s boundary; confirm stable across two timed runs (~610s and ~603s). |
| Q7 — API-key usage statistics | 1.5 | Read `times`/`last_used` before and after N key-authed calls (`0 → 5`, `NULL → timestamp`); confirm session-fallback does not increment. |
| Q8 — Failed login behavior | 2.0 | POST wrong credentials → HTTP 200 re-render + flash; capture log output; exercise `429` rate-limit edge. |
| Document authoring | 4.5 | Compose the 1,173-line answer document; per-question (a) answer, (b) commands, (c) unedited output, (d) values, (e) `file:line`, (f) cause→effect, (g) OBSERVED/INFERRED; coverage pass; read-only guarantee. |
| Review remediation (commit `d5b8e17b`) | 2.0 | Address code-review findings on evidence fidelity (+341 / −130). |
| Final independent validation | 3.0 | Re-run all 8 investigations in the live container; verify 8/8 reproduction, 100% citation grounding, dependency pins, and clean repo. |
| **Total Completed** | **28.0** | **Matches Section 1.2 Completed Hours.** |

### 2.2 Remaining Work Detail

Each remaining category is human path-to-production work; together they sum to the Remaining Hours in Section 1.2.

| Category | Hours | Priority |
|----------|------:|----------|
| SME technical review & acceptance of the 8 answers (verify values, evidence completeness, citation spot-checks) | 1.5 | High |
| Pull-request review & merge of the deliverable to the target branch | 0.5 | High |
| **Total Remaining** | **2.0** | — |

> **Out-of-scope product-backlog follow-ups (NOT counted in the 2.0h remaining):** Q2 `None`-guard fix (~2–3h), Q4 session rotation (~3–4h), Q1 cookie→API fallback review (~2h). These are advisories for the SimpleLogin product team, explicitly excluded from this deliverable's hours per AAP §0.5.2.

### 2.3 Hours Reconciliation

- Section 2.1 total (Completed) = **28.0h**
- Section 2.2 total (Remaining) = **2.0h**
- Section 2.1 + Section 2.2 = **30.0h** = Total Project Hours in Section 1.2 ✅
- Completion = 28.0 ÷ 30.0 = **93.3%** ✅

---

## 3. Test Results

For this task the **eight empirical investigations _are_ the autonomous test suite** — the AAP mandates no permanent pytest additions; each question is a runtime experiment that must reproduce exact output. Blitzy's autonomous validation re-ran all eight in the live container and reproduced 8/8 exactly (only environmental diffs such as port, session UUID, and timestamp). The "coverage %" column reflects sub-item coverage (every named item in the question observed), not line coverage.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|------------:|-------:|-------:|-----------:|-------|
| API / session (Q1) | curl + gunicorn `wsgi:app` | 2 | 2 | 0 | 100% | 200 + 8-key JSON (CL 239); 401 `{"error":"Wrong api key"}` (CL 26). |
| API authorization (Q2) | curl + gunicorn | 2 | 2 | 0 | 100% | Browser-session `500` AttributeError; real-key `440 "Need sudo"`. |
| Session storage (Q3) | Redis + `pickle` introspection | 1 | 1 | 0 | 100% | pickle proto-4, 300 bytes, keys `[_fresh,_id,_permanent,_user_id,csrf_token,sudo_time]`, TTL 604800, key `session:<uuid4>`. |
| Session identity (Q4) | test client + cookie unsign | 1 | 1 | 0 | 100% | `slapp` byte-identical before/after → not rotated. |
| Email forwarding (Q5) | `email_handler.py` forward path | 1 | 1 | 0 | 100% | `X-Custom-Test` / `Received` / `Reply-To` all stripped (Reply-To then replaced); `250 Message accepted`. |
| Token expiry (Q6) | `app/alias_suffix.py` signer | 2 | 2 | 0 | 100% | 600s window; last valid `600.001s`, first expired `601.001s`; stable across 2 runs. |
| API-key usage (Q7) | psql + key-authed calls | 1 | 1 | 0 | 100% | `times 0 → 5`; `last_used NULL → timestamp`; session-fallback does not increment. |
| Failed login (Q8) | curl + gunicorn (rate on/off) | 2 | 2 | 0 | 100% | HTTP 200 re-render + flash "Email or password incorrect"; `429` after 10 failed logins. |
| **Total** | — | **12** | **12** | **0** | **100%** | All originate from Blitzy's autonomous validation logs; independently reproduced this session. |

---

## 4. Runtime Validation & UI Verification

**Runtime health (re-verified live in container `sl` this session):**

- ✅ **Web/API app** — `gunicorn wsgi:app` boots (v20.0.4, 2 sync workers); root `/` → **HTTP 302** with `Location: /auth/login`.
- ✅ **PostgreSQL 15** — `pg_isready` → "accepting connections"; alembic head `32f25cbf12f6`.
- ✅ **Redis** — `redis-cli ping` → `PONG`; session store + rate-limiter backend operational.
- ✅ **Email forward path** — `email_handler.py` `handle_forward(...)` returns `[(True, '250 Message accepted for delivery')]`.
- ✅ **Token signer** — `app/alias_suffix.signer.sign(...)` + `check_suffix_signature(...)` validates immediately (`'demo.suffix'`).

**API integration outcomes:**

- ✅ Q1 `GET /api/user_info` (no auth) → **HTTP 401** `{"error":"Wrong api key"}` (CL 26) — reproduced live.
- ✅ Q1 session-cookie path → **HTTP 200** with 8-key JSON (CL 239).
- ⚠ Q2 `DELETE /api/user` (browser session) → **HTTP 500** unhandled `AttributeError` — a genuine product edge (report-only; does not delete account).
- ✅ Q2 `DELETE /api/user` (real key, no sudo) → **HTTP 440** `{"error":"Need sudo"}`.

**UI verification:**

- ✅ Login form (`/auth/login`) renders; wrong credentials produce an **HTTP 200 re-render** carrying the flash message "Email or password incorrect" (not a 401).
- ⚠ Repeated failed logins trip the rate limiter → **HTTP 429** after 10 attempts (rate-limit-enabled boot).

Legend: ✅ Operational · ⚠ Partial / product edge (report-only) · ❌ Failing (none).

---

## 5. Compliance & Quality Review

The deliverable is cross-mapped to the AAP's hard requirements and rule set (`SWE-AtlasQnA-Repo`). Review remediation was applied in commit `d5b8e17b` (evidence fidelity, +341/−130).

| Requirement (AAP / Rules) | Benchmark | Status | Progress | Notes |
|---------------------------|-----------|--------|:--------:|-------|
| Single answer document `<branch>.md` in `blitzy/documentation/` | Exactly one CREATE | ✅ Pass | 100% | `blitzy/documentation/app_2cd6ee777f8c.md` only. |
| Run-first-then-write (evidence from running code) | All answers from runtime | ✅ Pass | 100% | Each answer paired with command + unedited output. |
| Read-only source repository | Zero source modifications | ✅ Pass | 100% | `git diff 2cd6ee77..HEAD` = 1 file added, 0 source changed. |
| Complete, unedited output + command per claim | No paraphrase/truncation | ✅ Pass | 100% | 25 balanced code blocks of raw output. |
| All conditions (not just happy path) | Primary + error + edge + transitional | ✅ Pass | 100% | Q1 401, Q2 440-vs-500, Q5 Reply-To replacement, Q8 200 + 429, Q4/Q7 before/after. |
| Exactness + `file:line` grounding | Cite source lines; label OBSERVED/INFERRED | ✅ Pass | 100% | 100% of citations independently verified. |
| Timing/magnitude rigor (Q6) | State duration; ≥2 runs; stable boundary | ✅ Pass | 100% | 600s boundary, two runs (~610s / ~603s). |
| Before/after capture (Q4, Q7) | Explicit boundary states | ✅ Pass | 100% | Session id + `times`/`last_used` captured pre/post. |
| Coverage pass | Every named sub-item answered | ✅ Pass | 100% | Dedicated Coverage Pass section. |
| Cleanup of temporary artifacts | `git status` shows only the doc | ✅ Pass | 100% | Temp scripts + throwaway keys removed; clean tree. |
| Markdown well-formedness | Balanced fences, complete structure | ✅ Pass | 100% | 50 fence lines = 25 balanced blocks. |

**Fixes applied during autonomous validation:** Review remediation commit `d5b8e17b` improved evidence fidelity across Q1–Q8. Final validation applied **no further fixes** — the document was already accurate; modifying an already-correct, committed deliverable would risk regression and violate the minimal-change / read-only principle. **Outstanding compliance items: none.**

---

## 6. Risk Assessment

Risks are overwhelmingly **product-behavior findings surfaced by the investigation**, not defects in the deliverable (which is validated and low-risk). Items marked *report-only* are out of scope to fix per AAP §0.5.2.

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| T1 — Q2 `DELETE /api/user` browser-session-only raises unhandled `AttributeError` (`app/api/base.py:47`) → HTTP 500 | Technical | Medium | High (deterministic) | Product team: add `None`-guard in `check_sudo_mode_is_active`. Does **not** delete the account. | Documented (report-only) |
| T2 — Evidence contains environment-specific values (UUIDs, timestamps, ports, throwaway key ids) | Technical | Low | Medium | Values labeled/redacted; commit `2cd6ee77` + dependency versions pinned for reproduction. | Mitigated / Disclosed |
| S1 — Q4 session id not rotated on login despite `session_protection="strong"` → session-fixation | Security | Medium | High (deterministic) | Product team: regenerate session id on login. | Documented (report-only) |
| S2 — Q1 `HEADER_ALLOW_API_COOKIES` gate commented out (`app/api/base.py:22-24`); browser session reaches API without a key | Security | Low–Medium | Medium | Product review of the cookie→API fallback policy. | Documented (report-only) |
| S3 — `FLASK_SECRET=secret` fake dev value | Security | Informational | N/A (dev container) | Ephemeral test container only; not production config. | Disclosed |
| O1 — Reproduction requires the specific swe-atlas container image | Operational | Low | Medium | Dev guide (Section 9) documents exact image + commands. | Mitigated |
| O2 — Documentation drift as SimpleLogin evolves | Operational | Low | Medium | Pinned to commit `2cd6ee77` + exact versions; OBSERVED/INFERRED labels. | Mitigated |
| I1 — Integration risk | Integration | None | N/A | Deliverable introduces no code, imports, dependencies, or external integrations. | N/A |

---

## 7. Visual Project Status

**Project hours breakdown** (Completed = Dark Blue `#5B39F3`, Remaining = White `#FFFFFF`):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeWidth':'2px'}}}%%
pie showData title Project Hours Breakdown (30h total)
    "Completed Work" : 28
    "Remaining Work" : 2
```

**Remaining work by priority** — both remaining items are **High** priority (path-to-production human gates):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#A8FDD9','pie3':'#FFFFFF'}}}%%
pie showData title Remaining Hours by Priority (2h total)
    "High — SME review & acceptance" : 1.5
    "High — PR review & merge" : 0.5
```

> **Integrity:** "Remaining Work" = **2.0h**, identical to Section 1.2 (Remaining Hours) and the Section 2.2 Hours total. "Completed Work" = **28.0h**, identical to Section 1.2 and the Section 2.1 total.

---

## 8. Summary & Recommendations

**Achievements.** The project is **93.3% complete** (28.0h of 30.0h). All eight runtime-behavior questions were answered from live execution with complete, unedited evidence and verified `file:line` grounding, and packaged into a single 1,173-line document. The read-only mandate was upheld exactly — the only repository change is the one added answer file. Blitzy's autonomous validation independently reproduced 8/8 investigations and confirmed 100% citation accuracy and matching dependency pins.

**Remaining gaps.** The residual 2.0h is purely human path-to-production: a subject-matter-expert acceptance review of the answers (1.5h) and PR review & merge (0.5h). There is no remaining engineering work inside the AAP scope; the deliverable is complete and committed.

**Critical path to production.** SME technical review → approve PR → merge. That is the entire critical path; there are no build, integration, or deployment steps for a documentation artifact.

**Success metrics.** 8/8 questions answered and independently reproduced; 0 source files modified; 100% of citations verified; markdown well-formed (25 balanced code blocks); coverage pass complete.

**Production-readiness assessment.** The deliverable is **ready for human acceptance and merge**. Completion is held at 93.3% (below the 99% ceiling) precisely because honest, remaining human review/acceptance of a knowledge artifact cannot be self-certified by the agent. The two behavioral findings (Q2 error path, Q4 session non-rotation) are correctly reported as product advisories and are out of scope to remediate here; they are recommended as separate product-backlog follow-ups.

| Metric | Value |
|--------|-------|
| Completion | 93.3% (28.0h / 30.0h) |
| Questions answered & reproduced | 8 / 8 |
| Source files modified | 0 |
| Citation accuracy | 100% |
| Remaining (human) hours | 2.0 |

---

## 9. Development Guide

All commands below were **tested live** inside the running container `sl` this session. Because PostgreSQL and Redis run *inside* the container, prefix host invocations with `docker exec sl bash -c '…'`.

### 9.1 System Prerequisites

- **Docker** (host) — the investigation container is `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`.
- **Container OS:** Debian GNU/Linux 12 (bookworm).
- **Python:** 3.10.18 (inside the container venv at `/app/venv`).
- **Datastores:** PostgreSQL 15 and Redis — both preinstalled and started inside the container.
- **Source:** SimpleLogin at `/app`, commit `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`.

### 9.2 Environment Setup

```bash
# Confirm the container is running (host shell)
docker ps --format '{{.Names}}\t{{.Image}}\t{{.Status}}'

# Enter the app dir, activate venv, and load the runtime env (DB_URI, FLASK_SECRET, ...)
docker exec sl bash -c 'cd /app && . venv/bin/activate && eval "$(grep "^export " /build.sh)" && echo "DB_URI=$DB_URI"'
# Expected: DB_URI=postgresql://test:test@localhost:5432/test
```

### 9.3 Dependency Verification

```bash
# Confirm the pinned dependency versions the answers depend on
docker exec sl bash -c 'cd /app && . venv/bin/activate && pip freeze | grep -iE "^(Flask|Flask-Login|itsdangerous|Werkzeug|redis|SQLAlchemy|psycopg2-binary|aiosmtpd|flanker|Flask-Limiter|Flask-WTF|gunicorn|arrow)==" | sort -f'
# Expected (exact): Flask==1.1.2, Flask-Login==0.5.0, itsdangerous==1.1.0, Werkzeug==1.0.1,
#   redis==4.6.0, SQLAlchemy==1.3.24, psycopg2-binary==2.9.3, aiosmtpd==1.4.2, flanker==0.9.11,
#   Flask-Limiter==1.4, Flask-WTF==0.14.3, gunicorn==20.0.4, arrow==0.16.0
```

### 9.4 Service Verification

```bash
docker exec sl bash -c 'cd /app && . venv/bin/activate && eval "$(grep "^export " /build.sh)" && pg_isready && redis-cli ping && psql "$DB_URI" -tAc "SELECT version_num FROM alembic_version;"'
# Expected: /var/run/postgresql:5432 - accepting connections
#           PONG
#           32f25cbf12f6
```

### 9.5 Application Startup

```bash
# Boot the web/API app in the background (survives the exec session via nohup)
docker exec sl bash -c 'cd /app && . venv/bin/activate && eval "$(grep "^export " /build.sh)" && nohup gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 30 > /tmp/gunicorn.log 2>&1 & echo "pid=$!"'
sleep 8   # gunicorn needs a few seconds to boot before it answers

# Confirm boot + the documented root redirect
docker exec sl bash -c 'head -3 /tmp/gunicorn.log; curl -s -i http://localhost:7777/ | head -1; curl -s -i http://localhost:7777/ | grep -i "^Location"'
# Expected: HTTP/1.1 302 FOUND  and  Location: http://localhost:7777/auth/login
```

### 9.6 Verification Steps / Example Usage (reproduce the investigations)

```bash
# Q1 — API without auth returns 401
docker exec sl bash -c 'curl -s -i http://localhost:7777/api/user_info | grep -iE "^HTTP|Content-Length|Wrong api key"'
# Expected: HTTP/1.1 401 UNAUTHORIZED | Content-Length: 26 | {"error":"Wrong api key"}

# Q6 — sign a suffix token and validate it immediately
docker exec sl bash -c 'cd /app && . venv/bin/activate && eval "$(grep "^export " /build.sh)" && python -c "from app.alias_suffix import signer, check_suffix_signature; s=signer.sign(\"demo.suffix\").decode(); print(check_suffix_signature(s))"'
# Expected: demo.suffix

# Q7 — inspect API-key usage columns
docker exec sl bash -c 'cd /app && . venv/bin/activate && eval "$(grep "^export " /build.sh)" && psql "$DB_URI" -c "SELECT id,name,times,last_used,sudo_mode_at FROM api_key ORDER BY id LIMIT 5;"'
# Expected: rows including the times counter and last_used / sudo_mode_at columns

# Login credentials for interactive checks: john@wick.com / password
```

### 9.7 Shutdown / Cleanup

```bash
# Stop only the gunicorn you started (targets the specific port pattern)
docker exec sl bash -c 'pids=$(ps aux | grep "[g]unicorn" | grep "7777" | awk "{print \$2}"); for p in $pids; do kill "$p"; done; rm -f /tmp/gunicorn.log; echo cleaned'
```

### 9.8 Troubleshooting

- **`curl` returns nothing right after boot** → gunicorn needs ~8s; add `sleep 8` before probing.
- **Background gunicorn dies when the exec returns** → use `nohup … &` (as above) so it outlives the `docker exec` session.
- **`DB_URI`/`FLASK_SECRET` unset** → you forgot `eval "$(grep '^export ' /build.sh)"`; always source it after activating the venv.
- **Q8 `429` not observed** → the default boot disables rate limiting; `unset DISABLE_RATE_LIMIT` before starting gunicorn, then the 11th failed login returns `429`.
- **Port already in use** → pick a fresh port (e.g., `-b 0.0.0.0:7898`) for scratch runs and clean it up afterward.

---

## 10. Appendices

### Appendix A — Command Reference

| Purpose | Command |
|---------|---------|
| List running containers | `docker ps --format '{{.Names}}\t{{.Image}}\t{{.Status}}'` |
| Enter app + load env | `docker exec sl bash -c 'cd /app && . venv/bin/activate && eval "$(grep "^export " /build.sh)"'` |
| Verify services | `pg_isready && redis-cli ping` |
| Alembic head | `psql "$DB_URI" -tAc "SELECT version_num FROM alembic_version;"` |
| Boot web/API | `nohup gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 30 &` |
| Root redirect check | `curl -s -i http://localhost:7777/` |
| Deliverable diff | `git diff 2cd6ee77..HEAD --name-status` |
| Repo cleanliness | `git status --porcelain` |

### Appendix B — Port Reference

| Port | Service | Notes |
|------|---------|-------|
| 7777 | gunicorn `wsgi:app` (canonical) | Web/API; root → 302 `/auth/login`. |
| 7778 | gunicorn (rate-limit-enabled) | Used to reproduce the Q8 `429` edge (`unset DISABLE_RATE_LIMIT`). |
| 5432 | PostgreSQL 15 | Inside container; `DB_URI=postgresql://test:test@localhost:5432/test`. |
| 6379 | Redis | Session store + rate-limiter backend. |

### Appendix C — Key File Locations

| Path | Role |
|------|------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The deliverable** (only added file). |
| `app/api/base.py` | Q1 session fallback (`:17-27`), Q2 sudo `440`/`check_sudo_mode_is_active` (`:46-49,66-70`), Q7 usage update (`:30-32`). |
| `app/api/views/user.py` | Q2 sole `@require_api_sudo` endpoint `DELETE /api/user` (`:12-14`). |
| `app/session.py` | Q3 pickle serialization (`:91`) + `session:<uuid4>` key (`:18,44-45`); Q4 id minting. |
| `app/auth/views/login.py` / `login_utils.py` | Q8 failed-login branch (`:45-50`); Q4 `after_login()`→`login_user()` (`:36-37`). |
| `app/extensions.py` | Q4 `session_protection = "strong"` (`:8`). |
| `email_handler.py` | Q5 `headers_to_keep` + `delete_all_headers_except` (`:793-810`), Reply-To rewrite (`:871-872`). |
| `app/alias_suffix.py` | Q6 `TimestampSigner` (`:11`) + `unsign(max_age=600)` (`:37-42`). |
| `app/models.py` | Q7 `ApiKey` columns `last_used`/`times`/`sudo_mode_at` (`:2358-2360`). |
| `app/config.py` | `FLASK_SECRET` (`:196`), `SESSION_COOKIE_NAME="slapp"` (`:199`), `CUSTOM_ALIAS_SECRET` (`:201`). |

### Appendix D — Technology Versions

| Package | Version | Package | Version |
|---------|---------|---------|---------|
| Python | 3.10.18 | redis | 4.6.0 |
| Flask | 1.1.2 | SQLAlchemy | 1.3.24 |
| Flask-Login | 0.5.0 | psycopg2-binary | 2.9.3 |
| itsdangerous | 1.1.0 | aiosmtpd | 1.4.2 |
| Werkzeug | 1.0.1 | flanker | 0.9.11 |
| gunicorn | 20.0.4 | Flask-Limiter | 1.4 |
| arrow | 0.16.0 | Flask-WTF | 0.14.3 |
| PostgreSQL | 15 (alembic head `32f25cbf12f6`) | Redis | running (`PONG`) |

### Appendix E — Environment Variable Reference

| Variable | Value / Source | Purpose |
|----------|----------------|---------|
| `DB_URI` | `postgresql://test:test@localhost:5432/test` (from `/build.sh`) | PostgreSQL connection for Q7 reads and ORM. |
| `FLASK_SECRET` | dev value in container (from `/build.sh`) | Session-cookie signer + `CUSTOM_ALIAS_SECRET` seed. |
| `DISABLE_RATE_LIMIT` | set by default; `unset` to enable limiter | Toggles the Q8 `429` rate-limit edge. |
| `REDIS_URL` | container default | Session store + rate-limiter backend. |

### Appendix F — Developer Tools Guide

- **Reproduce any investigation:** boot gunicorn (Section 9.5) or import the target module in a `python -c` one-liner under the activated venv with `/build.sh` env loaded.
- **Inspect a session in Redis:** `redis-cli --scan --pattern 'session:*'` then `redis-cli get <key>` (raw pickle bytes; deserialize with `pickle.loads`).
- **Read DB state:** `psql "$DB_URI" -c '<SQL>'`.
- **Drive the email forward path (Q5):** call `email_handler.handle_forward(envelope, msg, rcpt_to)` with a crafted `.eml` under the venv.
- **Verify repo integrity:** `git diff 2cd6ee77..HEAD --name-status` (expect a single `A blitzy/documentation/app_2cd6ee777f8c.md`).

### Appendix G — Glossary

| Term | Meaning |
|------|---------|
| **AAP** | Agent Action Plan — the primary directive defining scope and constraints. |
| **`slapp`** | SimpleLogin's session cookie name (`SESSION_COOKIE_NAME`). |
| **`require_api_sudo`** | Decorator gating privileged API ops; returns HTTP `440 "Need sudo"`. |
| **HTTP 440** | Non-standard status (originally Microsoft IIS "Login Time-out"), re-purposed here with a `"Need sudo"` payload. |
| **`TimestampSigner`** | `itsdangerous` signer whose `unsign(max_age=600)` enforces the Q6 token expiry window. |
| **Session fixation** | Risk where the session id is not rotated on privilege change (login) — the Q4 finding. |
| **OBSERVED / INFERRED** | Evidence labels: measured at runtime vs. deduced from reading (to be confirmed). |
| **Coverage pass** | Final re-read confirming every named sub-item of every question is answered. |