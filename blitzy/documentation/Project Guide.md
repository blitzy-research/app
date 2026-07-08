# Blitzy Project Guide — SimpleLogin Redis-Backed Flask Session Subsystem: Runtime-Grounded Q&A

> **Deliverable branch:** `blitzy-82163eff-1204-4fd2-99f5-0ba7521c2fbe` · **Base:** `app_2cd6ee777f8c` (`2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`) · **HEAD:** `ebb15e0f`
> **Task type:** Documentation (runtime-grounded security Q&A) · **Ruleset:** SWE-AtlasQnA-Repo · **Scope:** Isolated, additive, read-only w.r.t. source

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a single, runtime-grounded Q&A document that explains how SimpleLogin's custom, Redis-backed Flask session subsystem behaves at runtime, with a focused analysis of how stored session data is deserialized (`pickle.loads`) and exactly where the boundary lies between a harmless session reset and a genuine code-execution risk. The target audience is SimpleLogin's engineering and security stakeholders. Every behavioral claim is backed by actual, unedited output captured from running the real code paths in the canonical Python 3.10 build. The technical scope covers six named requirements (R1–R6) and eight runtime conditions (C1–C8), exercised through the real HTTP login/logout entry points plus direct Redis manipulation. The work is strictly read-only: exactly one new documentation file is added; no product code is modified.

### 1.2 Completion Status

The project is **91.1% complete** on an AAP-scoped, hours-based basis. All AAP-specified deliverables (R1–R6, C1–C8, methodology, research, read-only compliance, integrity proof) are fully delivered and byte-accurately validated. The remaining **5 hours** is path-to-production human acceptance (security-SME review, editorial review, publish/merge) — no autonomous engineering work remains.

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeWidth':'2px','pieSectionTextColor':'#FFFFFF','pieTitleTextSize':'18px','pieLegendTextColor':'#111111'}}}%%
pie showData title Completion Status — 91.1% Complete
    "Completed Work (h)" : 51
    "Remaining Work (h)" : 5
```

| Metric | Hours |
|--------|-------|
| **Total Hours** | **56** |
| **Completed Hours (AI + Manual)** | **51** (AI: 51, Manual: 0) |
| **Remaining Hours** | **5** |
| **Percent Complete** | **91.1%** |

> Calculation (PA1): `Completed / (Completed + Remaining) = 51 / (51 + 5) = 51 / 56 = 91.1%`.

### 1.3 Key Accomplishments

- ✅ **Single deliverable created** at the rule-mandated path `blitzy/documentation/app_2cd6ee777f8c.md` (1,291 lines, ~10,581 words, 64 balanced code fences).
- ✅ **R1 — Deserialization mechanism** answered: `pickle.loads(val)` at `app/session.py:76`, protocol-4 payloads (`\x80\x04`).
- ✅ **R2 — Normal lifecycle** answered across login, logout, unauthenticated CSRF-only, and `save_session` paths (session keys, TTL 300/604800, logout ID rotation + 4 `Set-Cookie`).
- ✅ **R3 — Malformed-data behavior** proven: corrupt/truncated bytes → `UnpicklingError` → silently swallowed → fresh session (HTTP 302/200, never 500).
- ✅ **R4 — Response/log surface** proven: no error surfaced, zero log lines on the failure path, no Sentry event.
- ✅ **R5 — Risk boundary** proven at runtime: a well-formed malicious pickle executes its `__reduce__` **during** `pickle.loads` (server-side code execution proven via a marker written by the gunicorn worker) — boundary is "reset vs. code execution," not "reset vs. error."
- ✅ **R6 — Attacker tampering without forging the ID** proven: the `itsdangerous.Signer` authenticates only the session-ID pointer, not the pickled payload; a tampered payload with a legitimately-signed cookie still reaches the vulnerable `pickle.loads`.
- ✅ **All 8 conditions (C1–C8)** exercised with before/during/after state capture.
- ✅ **Read-only mandate preserved**: `git diff` shows exactly one added file; zero source files changed; working tree clean; all temporary scripts and planted Redis keys removed.
- ✅ **Independently re-verified**: this assessment reproduced R1/R3/R5, dependency versions, session interface, and endpoint reachability in the canonical Python 3.10.18 container — all byte-identical to the document.

### 1.4 Critical Unresolved Issues

There are **no unresolved issues blocking the documentation deliverable** — it is complete and validated. The item below is a **product security finding surfaced by the deliverable**; remediation is explicitly out of scope for this task (AAP §0.5.2) and is tracked as a separate follow-on effort.

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| CWE-502 pickle deserialization RCE in `RedisSessionStore` (finding, not a deliverable defect) | Attacker with Redis write access can achieve remote code execution on the app server (proven at runtime) | SimpleLogin Security / Backend team | To be scheduled (separate remediation; out of scope for this doc) |

### 1.5 Access Issues

No access issues identified for producing or validating this deliverable. The canonical container, Redis, and PostgreSQL were all reachable during assessment.

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-----------------|----------------|-------------------|-------------------|-------|
| Canonical image `sl_app` | Container runtime | None — image present locally, container running | ✅ No issue | Blitzy env |
| Redis (`:6379`) | Session backend | None — `redis-cli ping` = PONG | ✅ No issue | Blitzy env |
| PostgreSQL (`:15432`) | Database | None — `pg_isready` = accepting connections | ✅ No issue | Blitzy env |
| Git remote (`origin`) | Repository | None — branch up to date, tree clean | ✅ No issue | Blitzy env |

### 1.6 Recommended Next Steps

1. **[High]** Have a security SME review and validate the CWE-502 RCE finding, assign a CVSS score, and confirm remediation ownership.
2. **[Medium]** Complete a technical & editorial peer review of the Q&A document (accuracy of `file:line` references, replayable commands, links, readability).
3. **[Medium]** Approve the PR, merge the document to the mainline, and distribute it to engineering/security stakeholders.
4. **[High — separate follow-on, out of scope here]** Plan CWE-502 remediation: replace `pickle` with JSON for session payloads (or add a restricted `Unpickler` allowlist + MAC over the stored bytes), and restrict/authenticate Redis network access.
5. **[Medium — separate follow-on, out of scope here]** Add logging/alerting on deserialization failure to close the observability gap identified in R4.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

All completed work was performed autonomously by Blitzy agents (Manual = 0h). Each component traces to a specific AAP requirement.

| Component | Hours | Description |
|-----------|-------|-------------|
| Canonical environment boot & runtime baseline | 4 | Boot SimpleLogin in the canonical Python 3.10 container; confirm Redis-backed `RedisSessionStore`, secret, cookie name, pickle protocol; endpoint reachability |
| Web research & risk-model validation | 3 | CWE-502 (Flask-Caching CVE-2021-33026 analog), `itsdangerous` Signer-vs-Serializer semantics, pickle `__reduce__` execution, remediation baseline |
| Backend wiring analysis (plain vs Sentinel) | 2 | `initialize_redis_services` read/write split: plain `redis://` same-connection vs `redis+sentinel://` master-write/slave-read |
| R1 — Deserialization mechanism | 2 | `pickle.loads(val)` at `app/session.py:76`; protocol-4 frame; grounding & causal reasoning |
| R2 — Normal session lifecycle | 8 | Real HTTP login/logout with CSRF handling; decode pickled Redis dict; keys, TTLs, cookie, ID-rotation-on-logout; unauth CSRF-only + `save_session` paths |
| R3 — Malformed-data behavior | 3 | Random-bytes and truncated-pickle poisoning; byte-exact `UnpicklingError` capture; silent-reset proof |
| R4 — Response/log surface | 2 | HTTP response inspection; log-window grep scan; Sentry (`SENTRY_DSN` unset) analysis |
| R5 — Risk boundary (PoC + code-exec proof) | 5 | CWE-502 proof-of-construct malicious pickle; server-side code execution proven via marker file with worker `ppid` match |
| R6 — Attacker tampering without forging ID | 4 | Tamper-without-forgery (reuse legit cookie) vs `BadSignature` forged-cookie contrast |
| Runtime driver script (8 conditions) | 4 | Full replayable driver exercising C1–C8 with before/during/after capture |
| Document authoring, structuring & formatting | 8 | 1,291-line Markdown: headline, methodology, tables, Mermaid decision diagram, coverage matrix |
| Coverage pass, integrity proof, cleanup | 2 | Named-item coverage table; repository integrity proof; temp-script/Redis-key cleanup; read-only compliance |
| Review-cycle iterations (3 follow-up commits) | 4 | Code-review findings, R5/R6 QA fixes, replayable-command hardening |
| **TOTAL COMPLETED** | **51** | |

### 2.2 Remaining Work Detail

All remaining work is path-to-production human acceptance of a completed, validated document. (CWE-502 remediation is a **separate** out-of-scope effort — see §2.3 note — and is **not** included here.)

| Category | Hours | Priority |
|----------|-------|----------|
| Security Review & Validation (validate CWE-502 finding, assign severity, confirm ownership) | 2 | High |
| Technical & Editorial Review (verify R1–R6 accuracy, `file:line` refs, commands, formatting) | 2 | Medium |
| Publication & Merge (approve PR, merge, distribute to stakeholders) | 1 | Medium |
| **TOTAL REMAINING** | **5** | |

### 2.3 Hours Reconciliation & Methodology

- **Total Project Hours = 56** = Completed (51) + Remaining (5).
- **Completion % = 51 / 56 = 91.1%** (PA1 AAP-scoped, hours-based).
- **Cross-section anchors (identical everywhere):** Total 56h · Completed 51h · Remaining 5h · 91.1%.
- **Out-of-scope exclusion:** The pickle-RCE remediation (indicative ~16–24h), Redis hardening (~4–8h), and deserialization-failure logging (~2–4h) are follow-on product changes explicitly forbidden by the read-only mandate (AAP §0.5.2). They are surfaced as recommendations (§1.6, §8) and **excluded** from the 56h total to avoid scope creep.

---

## 3. Test Results

For a Markdown documentation deliverable there are no unit tests; the meaningful validation is **reproducing every runtime claim** against the canonical build. The results below originate entirely from Blitzy's autonomous validation logs — the Final Validator's exhaustive reproduction and this assessment's independent spot-check — plus the repository's own regression suite (context only).

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|------------|-------|
| Runtime condition reproduction (C1–C8) | Custom drivers: `requests.Session` + `redis` client + `pickle` on canonical Py 3.10 | 8 | 8 | 0 | 100% of AAP conditions | before/during/after captured; byte-accurate |
| Requirement verification (R1–R6) | Runtime drivers + standalone `pickle.loads` | 6 | 6 | 0 | 100% of AAP requirements | each answered with concrete value + `file:line` + evidence |
| Independent PM spot-check | Canonical 3.10 container (`sl_app`) | 8 | 8 | 0 | Key claims | protocol frame `\x80\x04`; `invalid load key, '\x00'.`; `pickle data was truncated`; malicious-pickle code exec; dep versions; `session_interface`; endpoint reachability |
| **Subtotal — task validation** | — | **22** | **22** | **0** | **100% of AAP scope** | All behavioral claims reproduced |
| Repository regression suite (context only, out of scope) | `pytest` | 639 | 638 | 1 | See note | 1 failure = external Apple receipt-server network call; environmental; unrelated to the session subsystem |

**Integrity note:** All task-validation tests derive from Blitzy's autonomous execution logs for this project. The `.coverage`/`htmlcov` artifacts in the repo are from the pre-existing repository suite, not from this documentation task; code coverage is **N/A** for a Markdown deliverable.

---

## 4. Runtime Validation & UI Verification

**Runtime health (canonical Python 3.10.18 container `sl_app`):**

- ✅ **Operational** — Application boots via `wsgi:app` (`gunicorn`, `CONFIG=tests/test.env`).
- ✅ **Operational** — `GET /health` → **200**.
- ✅ **Operational** — `GET /` → **302** (redirect to login).
- ✅ **Operational** — `GET /auth/login` → **200**; `Set-Cookie: slapp=<uuid>.<itsdangerous-signature>; HttpOnly; Path=/; SameSite=Lax`.
- ✅ **Operational** — Redis session backend reachable (`redis-cli ping` = PONG); PostgreSQL reachable (`pg_isready` = accepting connections).
- ✅ **Operational** — `app.session_interface` = `RedisSessionStore`; `pickle.DEFAULT_PROTOCOL` = 4; `secret_key` = `secret` (local test value); cookie name = `slapp`.

**API / behavioral integration outcomes:**

- ✅ **Operational** — R1 deserialization: benign pickle round-trip via `pickle.loads`; payloads carry the `\x80\x04` protocol-4 frame.
- ✅ **Operational** — R2 lifecycle: authenticated session keys `_user_id/_fresh/_id/_permanent/csrf_token/sudo_time`, TTL 604800; logout rotates the session ID and emits 4 `Set-Cookie` headers.
- ✅ **Operational** — R3 malformed: random and truncated payloads raise `UnpicklingError`, are swallowed, and produce a silent reset (302, never 500).
- ✅ **Operational** — R4 surface: no error line on the failure path; `SENTRY_DSN` unset → no Sentry event.
- ✅ **Operational** — R5/R6: a well-formed malicious pickle executes during `pickle.loads` (server-side, proven via worker-`ppid` marker); a forged cookie yields `BadSignature` and the Redis payload is never read.

**UI verification:** **Not applicable.** This task produces a backend/runtime analysis document; there is no user-facing interface, component library, or design system (AAP §0.3.3).

---

## 5. Compliance & Quality Review

Cross-map of AAP deliverables and the governing SWE-AtlasQnA-Repo rules against observed compliance.

| Benchmark / Rule | Requirement | Status | Evidence / Notes |
|------------------|-------------|--------|------------------|
| Deliverable location & name | `blitzy/documentation/app_2cd6ee777f8c.md` | ✅ Pass | File exists at exact path; branch-named per rule |
| Run-first methodology | Claims derived from executed output, not reading | ✅ Pass | 64 code fences of unedited runtime output; independently reproduced |
| Canonical build/config | Python 3.10 canonical Docker container | ✅ Pass | Python 3.10.18; deps byte-identical; non-canonical labeled |
| Exhaustive condition coverage | Every named condition (C1–C8) | ✅ Pass | Conditions-covered table with before/during/after |
| Evidence next to every claim | Complete, unedited output + command | ✅ Pass | Per-requirement evidence blocks + driver script |
| Answer every named item | R1–R6 + siblings (unauth, BadSignature, wiring) | ✅ Pass | Coverage-pass matrix confirms each |
| Exact & grounded | `file:line` + named function + causal reasoning | ✅ Pass | `pickle.loads@76`, `except@78`, signer `@37-41`, etc. verified against source |
| Read-only source mandate | No existing source file modified | ✅ Pass | `git diff` = 1 added file only; 10 reference files UNCHANGED |
| No remediation | Observe/explain only; do not fix | ✅ Pass | No code changes; remediation flagged as out-of-scope recommendation |
| Cleanup / no residue | Temp scripts & Redis keys removed | ✅ Pass | Working tree clean; no residue in re-check |
| Markdown quality | Balanced fences, trailing newline, no trailing ws | ✅ Pass | 64 balanced fences; ends-with-newline; 0 trailing-ws lines |
| Pre-commit (applicable hooks) | trailing-whitespace on `.md` | ✅ Pass | Manual equivalent check passed (no hook internet needed) |

**Fixes applied during autonomous validation:** None required by the Final Validator — the deliverable was already byte-accurate against the canonical runtime. Earlier review cycles (commits `d2b47391`, `3f2d00f7`, `ebb15e0f`) addressed code-review findings, R5/R6 QA refinements, and replayable-command hardening.

**Outstanding compliance items:** None for the deliverable. Human sign-off remains (see §2.2).

---

## 6. Risk Assessment

Risks are tagged **[Project]** (about the documentation deliverable) or **[Finding]** (a product security issue the document surfaces; remediation out of scope for this task).

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| **[Finding] CWE-502 pickle RCE** — `pickle.loads` on Redis session bytes with no MAC; Redis-write access → remote code execution | Security | Critical | Low–Medium (requires Redis write access) | Escalate to owners; replace pickle with JSON or restricted `Unpickler` + payload MAC; restrict Redis access | Documented & proven; remediation out of scope (separate effort) |
| **[Finding] No MAC over payload** — `itsdangerous.Signer` signs only the session-ID pointer, not the pickled data | Security | High | Low–Medium | MAC/authenticate the stored payload | Documented (R6) |
| **[Finding] Silent `except Exception: pass`** — all deserialization errors swallowed with no logging | Technical | High | Medium | Add logging/alerting on the failure path | Documented (R3/R4) |
| **[Finding] No log/Sentry on failure** — tampering/poisoning is invisible to operators/SOC | Operational | Medium | Medium | Emit a security log/metric on deserialization failure | Documented (R4) |
| **[Finding] Redis access = the security boundary** — network-exposed/shared Redis makes the RCE remotely reachable; depends on plain vs Sentinel topology | Integration | High (deployment-dependent) | Low–Medium | Restrict + authenticate Redis; network segmentation | Documented (Backend wiring) |
| **[Project] Canonical-only claims** — byte-level details (protocol, error text) may differ on non-canonical interpreters | Technical | Low | Low | Doc labels canonical vs NON-CANONICAL; verified in 3.10 container | Mitigated |
| **[Project] Documentation drift** — `file:line` refs pinned to base commit may stale as source evolves | Technical | Low | Medium | Point-in-time analysis explicitly pinned to `2cd6ee77` | Accepted |
| **[Project] Embedded PoC** — the doc contains a working malicious-pickle construct | Security | Low | Low | Standard, publicly-documented `os.system` marker construct; labeled as standard PoC, not user-supplied | Mitigated |
| **[Project] Test secret in doc** — `FLASK_SECRET=secret` appears | Security | Low | Low | Explicitly labeled a local test value from `tests/test.env`, not a real credential | Mitigated |
| **[Project] Reproduction needs container** — reviewer without the canonical stack cannot fully replay | Operational | Low | Medium | Exact replayable commands + image identity + full output embedded | Mitigated |
| **[Project] Repo coupling** — additive doc affecting imports/config/tests | Integration | None | N/A | No source/config/test/build changed (verified) | N/A (clean) |

**Overall:** Project-delivery risk is **LOW** (complete, validated, read-only preserved, all project risks mitigated/accepted). The document's central output is a **Critical** product finding (CWE-502 RCE) that requires human escalation and a separate remediation decision.

---

## 7. Visual Project Status

**Project hours breakdown (Completed = Dark Blue `#5B39F3`, Remaining = White `#FFFFFF`):**

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeWidth':'2px','pieSectionTextColor':'#FFFFFF','pieTitleTextSize':'18px','pieLegendTextColor':'#111111'}}}%%
pie showData title Project Hours Breakdown (Total 56h)
    "Completed Work" : 51
    "Remaining Work" : 5
```

**Remaining work by category (hours, from §2.2):**

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#A8FDD9','pie3':'#B23AF2','pieStrokeColor':'#111111','pieSectionTextColor':'#111111','pieLegendTextColor':'#111111'}}}%%
pie showData title Remaining Work by Category (Total 5h)
    "Security Review & Validation" : 2
    "Technical & Editorial Review" : 2
    "Publication & Merge" : 1
```

**Integrity check:** Pie "Remaining Work" = **5** = §1.2 Remaining Hours = §2.2 sum (2+2+1). Pie "Completed Work" = **51** = §1.2 Completed Hours = §2.1 sum. Total = 56h.

---

## 8. Summary & Recommendations

**Achievements.** The project delivers a complete, runtime-grounded Q&A document (1,291 lines) that answers all six named requirements (R1–R6) and exercises all eight conditions (C1–C8) with byte-accurate, unedited evidence from the canonical Python 3.10 build. The read-only mandate is fully preserved — exactly one file added, zero source files changed, working tree clean. This assessment independently reproduced the crux claims (R1 protocol frame, R3 error messages, R5 server-side code execution), the dependency versions, the session interface, and endpoint reachability inside the canonical container, all matching the document.

**Remaining gaps.** The project is **91.1% complete**. The only remaining work is **5 hours** of path-to-production human acceptance: security-SME validation of the CWE-502 finding, technical/editorial peer review, and PR approval/merge/publish. No autonomous engineering work remains on the deliverable.

**Critical path to production.** (1) Security review of the CWE-502 finding → (2) editorial/technical review → (3) merge & publish. These three human tasks unblock release of the document.

**Success metrics.** All AAP requirements answered (6/6), all conditions exercised (8/8), 22/22 behavioral reproductions passing, read-only preserved (1 added file), markdown quality gates passing (64 balanced fences, no trailing whitespace).

**Production readiness.** The **documentation deliverable is production-ready** pending human sign-off. Separately, the document's headline finding — a **Critical CWE-502 pickle deserialization RCE** in `RedisSessionStore` — is a product security risk that must be escalated and remediated as a distinct, out-of-scope follow-on effort (replace pickle with JSON or add a restricted `Unpickler` + payload MAC; restrict/authenticate Redis; add failure-path logging).

| Metric | Value |
|--------|-------|
| Completion | 91.1% |
| Completed / Remaining / Total hours | 51 / 5 / 56 |
| AAP requirements answered | 6 / 6 |
| Conditions exercised | 8 / 8 |
| Behavioral reproductions | 22 / 22 passing |
| Source files modified | 0 (read-only preserved) |

---

## 9. Development Guide

All commands below were tested in the canonical container during this assessment.

### 9.1 System Prerequisites

- **Docker Engine** (the canonical multi-service runtime).
- **Canonical image:** `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (container name `sl_app`).
- **Python 3.10** (inside the container venv at `/app/venv`) — the canonical interpreter is **3.10.18**.
- **Services:** Redis 7 (`:6379`), PostgreSQL 15 (`:15432`) — started by the container's `/build.sh`.

> ⚠️ The host agent sandbox runs **Python 3.13 (non-canonical)**. Byte-level pickle details (protocol, error text) can differ; always reproduce inside the canonical 3.10 container.

### 9.2 Environment Setup

```bash
# Confirm the canonical container is running
docker ps --format '{{.Names}}\t{{.Image}}\t{{.Status}}'

# The container's /build.sh (already run during setup) creates the venv,
# runs `poetry install`, starts PostgreSQL + Redis, generates local keys,
# and applies the Alembic schema migration (`alembic upgrade head`).

# Verify backing services
docker exec sl_app bash -lc 'redis-cli ping'                 # -> PONG
docker exec sl_app bash -lc 'pg_isready -h localhost -p 15432'  # -> accepting connections
```

Runtime configuration is selected with `CONFIG=tests/test.env` (`FLASK_SECRET=secret` — a local test value; `MEM_STORE_URI=redis://localhost`; `DB_URI` → PostgreSQL on `:15432`).

### 9.3 Dependency Verification

```bash
docker exec sl_app bash -lc 'source /app/venv/bin/activate && python --version'
# -> Python 3.10.18

docker exec sl_app bash -lc 'source /app/venv/bin/activate && python -c "import importlib.metadata as m,sys;print(\"python:\",sys.version.split()[0]);[print(f\"{p}: {m.version(p)}\") for p in [\"Flask\",\"Flask-Login\",\"itsdangerous\",\"Werkzeug\",\"redis\",\"Flask-Limiter\",\"limits\"]]"'
# -> Flask 1.1.2 / Flask-Login 0.5.0 / itsdangerous 1.1.0 / Werkzeug 1.0.1 / redis 4.6.0 / Flask-Limiter 1.4 / limits 1.5.1
```

### 9.4 Application Startup

```bash
docker exec sl_app bash -lc 'source /app/venv/bin/activate && cd /app && \
  CONFIG=tests/test.env nohup gunicorn --bind 127.0.0.1:7788 --workers 1 --timeout 120 \
  --pid /tmp/gunicorn.pid wsgi:app > /tmp/gunicorn.log 2>&1 &'
```

> Entry point is **`wsgi:app`** (`wsgi.py` exposes `app = create_app()`), **not** `server:app`.

### 9.5 Verification Steps

```bash
# Endpoint reachability (expect 200 / 302 / 200)
docker exec sl_app bash -lc 'curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:7788/health'
docker exec sl_app bash -lc 'curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:7788/'
docker exec sl_app bash -lc 'curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:7788/auth/login'

# Confirm the Redis-backed session interface and pickle protocol
docker exec sl_app bash -lc 'source /app/venv/bin/activate && cd /app && CONFIG=tests/test.env python -c "
from wsgi import app
from app.session import RedisSessionStore
import pickle
print(\"session_interface:\", type(app.session_interface).__name__)
print(\"is RedisSessionStore:\", isinstance(app.session_interface, RedisSessionStore))
print(\"cookie:\", app.config.get(\"SESSION_COOKIE_NAME\"), \"| pickle protocol:\", pickle.DEFAULT_PROTOCOL)
"'
# -> session_interface: RedisSessionStore | is RedisSessionStore: True | cookie: slapp | pickle protocol: 4

# Stop the server cleanly (targeted pid; never use pkill)
docker exec sl_app bash -lc 'kill "$(cat /tmp/gunicorn.pid)" && rm -f /tmp/gunicorn.pid /tmp/gunicorn.log'
```

### 9.6 Example Usage — Reproduce the Deserialization Behavior (R1/R3/R5)

```bash
docker exec sl_app bash -lc 'source /app/venv/bin/activate && python3 - <<PYEOF
import pickle, os
# R1: benign round-trip
b = pickle.dumps({"_user_id":"1","_fresh":True})
print("R1 frame:", b[:2], "| round-trip ok:", pickle.loads(b) == {"_user_id":"1","_fresh":True})
# R3: random + truncated -> UnpicklingError
for name, data in [("random", b"\x00\x01\x02not-a-pickle\xff"), ("truncated", pickle.dumps({"a":1,"b":2})[:6])]:
    try: pickle.loads(data)
    except Exception as e: print(f"R3 {name}:", type(e).__name__, "-", e)
# R5: well-formed malicious pickle EXECUTES during loads (does not raise)
m="/tmp/rce_marker.txt"; open(m,"w").close(); os.remove(m)
class X:
    def __reduce__(self): return (os.system, (f"echo executed > {m}",))
pickle.loads(pickle.dumps(X())); print("R5 code executed:", os.path.exists(m)); os.remove(m)
PYEOF'
# -> R1 frame: b\x27\x80\x04\x27 ... | R3 random: UnpicklingError - invalid load key, ...
#    R3 truncated: UnpicklingError - pickle data was truncated | R5 code executed: True
```

### 9.7 View & Validate the Deliverable

```bash
cd /tmp/blitzy/app/blitzy-82163eff-1204-4fd2-99f5-0ba7521c2fbe_7ec878

# Read-only proof: exactly one added file, clean tree
git diff --name-status 2cd6ee777f8c2d3531559588bcfb18627ffb5d2c..HEAD   # -> A  blitzy/documentation/app_2cd6ee777f8c.md
git status --porcelain                                                  # -> (empty)

# Markdown quality gates
python3 -c "l=open('blitzy/documentation/app_2cd6ee777f8c.md').read().splitlines();print(sum(x.startswith(chr(96)*3) for x in l))"  # even => balanced fences
grep -cE ' +$' blitzy/documentation/app_2cd6ee777f8c.md                 # -> 0 trailing whitespace
```

### 9.8 Troubleshooting

- **`module 'server' has no attribute 'app'`** → use `wsgi:app`, not `server:app`.
- **Redis not responding** → `docker exec sl_app bash -lc 'redis-cli ping'`; start `redis-server` if needed.
- **CSRF errors on `POST /auth/login`** → CSRF is enabled outside the test suite; `GET /auth/login` first, scrape the `csrf_token` hidden field, then POST with it (use a cookie-jar client such as `requests.Session`).
- **Different pickle bytes / error text** → you are likely on the non-canonical host interpreter (3.13); reproduce inside the canonical 3.10 container.
- **Port already in use** → pick a free port (the document uses `:7788`); stop any prior server via its pidfile.

---

## 10. Appendices

### Appendix A — Command Reference

| Purpose | Command |
|---------|---------|
| List containers | `docker ps --format '{{.Names}}\t{{.Image}}\t{{.Status}}'` |
| Interpreter version | `docker exec sl_app bash -lc 'source /app/venv/bin/activate && python --version'` |
| Boot app | `CONFIG=tests/test.env gunicorn --bind 127.0.0.1:7788 --workers 1 --timeout 120 wsgi:app` |
| Health check | `curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:7788/health` |
| Read-only proof | `git diff --name-status 2cd6ee777f8c..HEAD` |
| Clean-tree check | `git status --porcelain` |
| Balanced fences | `python3 -c "l=open('.../app_2cd6ee777f8c.md').read().splitlines();print(sum(x.startswith(chr(96)*3) for x in l))"` |

### Appendix B — Port Reference

| Port | Service | Notes |
|------|---------|-------|
| 6379 | Redis 7 | Session backend (`MEM_STORE_URI=redis://localhost`) |
| 15432 | PostgreSQL 15 | Application database (`DB_URI`) |
| 7788 | gunicorn (`wsgi:app`) | Port used by the deliverable's documented runs |
| 7799 | gunicorn (validation) | Alternate port used during validation to avoid collision |

### Appendix C — Key File Locations

| Path | Role |
|------|------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The deliverable** (only added file) |
| `app/session.py` | `RedisSessionStore`; `pickle.loads@76`; `except@78`; signer `@37-41`; `save_session@91,97-101` |
| `app/redis_services.py` | Session-interface wiring (plain vs Sentinel) |
| `server.py` | `secret_key`, cookie config, lifetime, Sentry init |
| `app/config.py` | `FLASK_SECRET`, `SESSION_COOKIE_NAME="slapp"`, `MEM_STORE_URI` |
| `app/extensions.py` | Flask-Login `session_protection="strong"` |
| `app/auth/views/login.py`, `login_utils.py` | Login entry point; `login_user()`, `sudo_time` |
| `app/auth/views/logout.py`, `app/api/views/user_info.py` | Logout entry points |
| `app/log.py` | Logging configuration (evidence of no log line on failure) |
| `wsgi.py` | Exposes `app = create_app()` |

### Appendix D — Technology Versions (canonical build, observed)

| Component | Version |
|-----------|---------|
| Python | 3.10.18 |
| Flask | 1.1.2 |
| Flask-Login | 0.5.0 |
| itsdangerous | 1.1.0 |
| Werkzeug | 1.0.1 |
| redis (client) | 4.6.0 |
| Flask-Limiter | 1.4 |
| limits | 1.5.1 |
| gunicorn | 20.0.4 |
| Redis server | 7 |
| PostgreSQL | 15 |

### Appendix E — Environment Variable Reference

| Variable | Value (canonical local) | Purpose |
|----------|-------------------------|---------|
| `CONFIG` | `tests/test.env` | Selects the runtime configuration file |
| `FLASK_SECRET` | `secret` (local **test** value) | HMAC key for the `itsdangerous` session-ID signer |
| `MEM_STORE_URI` | `redis://localhost` | Redis session backend (plain branch) |
| `DB_URI` | `postgresql://test:test@localhost:15432/test` | Application database |
| `SENTRY_DSN` | (unset) | When unset, Sentry is never initialized (relevant to R4) |

### Appendix F — Developer Tools Guide

- **Reproduction drivers:** `requests.Session` (cookie-jar HTTP client for real login/logout with CSRF), the `redis` Python client (inspect/poison `session:<uuid>`), and standalone `pickle.loads` (canonical 3.10) for byte-level checks. The full 8-condition driver is embedded in the deliverable's *Runtime driver script* section.
- **Static reference:** `app/session.py` is the authoritative mechanism; `open_session` decision logic is diagrammed (Mermaid) in the deliverable.
- **Safety:** stop servers via their pidfile (`kill "$(cat /tmp/gunicorn.pid)"`); never use broad `pkill`/`killall`.

### Appendix G — Glossary

| Term | Definition |
|------|------------|
| **CWE-502** | Deserialization of Untrusted Data — deserializing attacker-controlled bytes can execute arbitrary code |
| **`pickle`** | Python's native object serializer; `loads` executes an object's `__reduce__` callable during load |
| **`itsdangerous.Signer`** | HMAC signer over **raw bytes**; here it signs only the session-ID string, not the pickled payload |
| **`RedisSessionStore`** | SimpleLogin's custom Flask `SessionInterface` storing pickled session dicts in Redis |
| **`slapp`** | The session cookie name; carries `<uuid>.<signature>` |
| **Silent reset** | On a deserialization error, `open_session` swallows the exception and returns a fresh empty session |
| **Tamper-without-forgery** | Modifying the Redis payload while reusing a legitimately-signed cookie (no signature forgery) |
| **Canonical build** | The project's default Python 3.10 Docker runtime; the only source of reported behavioral values |

---

*Generated by the Blitzy Platform · AAP-scoped completion 91.1% (51 completed / 5 remaining / 56 total hours) · Read-only mandate preserved (1 file added, 0 source files changed).*
