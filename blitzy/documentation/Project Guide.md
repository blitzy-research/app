# Blitzy Project Guide — SimpleLogin Runtime-Behavior Investigation (`app_2cd6ee777f8c`)

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a single, evidence-backed technical document that answers **eight runtime-behavior questions (Q1–Q8)** about **SimpleLogin**, a Flask-based email-alias privacy service. The mandate was strictly read-only and *run-first*: every answer had to be derived by actually building, running, and observing the live system — not by reading source code. The deliverable, `blitzy/documentation/app_2cd6ee777f8c.md`, covers API authentication and sudo-gating, Redis-backed session serialization and session-fixation behavior, email-forwarding header rewriting, signed alias-suffix expiry, API-key usage counters, and login-failure handling. The audience is SimpleLogin's backend/security engineers. Business impact: a reproducible, citation-grounded reference that documents exact runtime behavior — including two security-relevant findings — without altering the application.

### 1.2 Completion Status

The completion percentage is calculated using AAP-scoped methodology (autonomous work delivered against the Agent Action Plan plus path-to-production activities). All autonomous AAP deliverables are complete and byte-for-byte validated; the remaining work is exclusively human review, acceptance, and merge.

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeWidth':'2px','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#111111','pieLegendTextColor':'#111111'}}}%%
pie showData title Project Completion — 89.1% Complete (AAP-Scoped)
    "Completed Work (AI)" : 45.0
    "Remaining Work" : 5.5
```

| Metric | Hours |
|--------|-------|
| **Total Hours** | **50.5** |
| Completed Hours — AI (autonomous) | 45.0 |
| Completed Hours — Manual (human) | 0.0 |
| **Completed Hours (AI + Manual)** | **45.0** |
| **Remaining Hours** | **5.5** |
| **Percent Complete** | **89.1%** |

> Completion formula: 45.0 completed ÷ 50.5 total × 100 = **89.1%**. The 10.9% remaining reflects only the human path-to-production review/acceptance/merge of the answer document; no autonomous rework remains.

### 1.3 Key Accomplishments

- ✅ **Full canonical runtime stood up and observed live** — gunicorn web app on `:7777`, inbound SMTP handler `email_handler.py` on `:20381`, Redis 6, and PostgreSQL 13 (schema migrated to head `32f25cbf12f6`, 77 tables), seeded through real entry points (`flask dummy-data`).
- ✅ **All eight questions (Q1–Q8) answered from live runtime evidence** — each with the exact command, complete unedited output, a `file:line` + function citation, and a cause→effect rationale.
- ✅ **Exhaustive coverage of every named variant** — both Q2 paths (HTTP 500 `AttributeError` and HTTP 440 "Need sudo"), all three Q5 headers by name (`X-Custom-Test`, `Received`, `Reply-To`), before/after state for Q4 and Q7, and a ≥2-run stability confirmation for the Q6 600-second boundary (plus a real wall-clock run).
- ✅ **Q5 investigated beyond the minimum** — because the canonical `NOT_SEND_EMAIL=true` path only logs a summary, a second, wire-level capture (purpose-built SMTP sink) was added so the full forwarded header block is shown; the canonical path is presented alongside.
- ✅ **Two security-relevant findings documented honestly** — Q4 session-fixation (session id not rotated on login) and Q2's HTTP 500 `AttributeError` — reported, not remediated, per the read-only mandate.
- ✅ **Read-only guarantee upheld** — `git diff` versus the source branch shows exactly one added file; all transient probe scripts were removed; no source or configuration file was modified.
- ✅ **Autonomous validation passed all five production-readiness gates** — including the repository's existing `pytest` suite (639 passed) executed to confirm runtime health.

### 1.4 Critical Unresolved Issues

There are no unresolved issues that block delivery of the answer document. The two items below are **runtime findings the document reports** about the SimpleLogin application; remediating them is explicitly **out of scope** for this read-only investigation and they do **not** block acceptance of the deliverable.

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| Q4 — Session identifier not rotated on login (session-fixation) | Application-security weakness; documented with before/after evidence. Out of scope to fix here. | SimpleLogin backend/security team (follow-up) | Not scheduled (out of AAP scope) |
| Q2(a) — Browser-session-only `DELETE /api/user` returns HTTP 500 `AttributeError` instead of a clean authorization error | Application robustness defect; documented with verbatim traceback. Out of scope to fix here. | SimpleLogin backend team (follow-up) | Not scheduled (out of AAP scope) |

### 1.5 Access Issues

No access issues identified. The investigation ran entirely inside the provisioned canonical Docker environment using local backends (Redis, PostgreSQL) and local fixtures; no external credentials, third-party API access, or elevated repository permissions were required. All displayed secrets are throwaway local fixtures from `example.env` and `flask dummy-data`.

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-----------------|----------------|-------------------|-------------------|-------|
| Git repository (branch `blitzy-f78a0548-…`) | Read/Write | None — deliverable committed successfully | Resolved | — |
| PostgreSQL 13 / Redis 6 (local containers) | Local service | None — reachable, seeded, observed | Resolved | — |
| External email / third-party APIs | N/A | Not required (`NOT_SEND_EMAIL=true`; no outbound mail) | N/A | — |

### 1.6 Recommended Next Steps

1. **[High]** Assign a backend/security SME to review the answer document — validate the eight answers, confirm the `file:line` citations resolve, and check the coverage-pass table (3.5h).
2. **[Medium]** Independently spot-reproduce at least two probes in the canonical Docker image (recommended: the Q6 600-second boundary and the Q3 Redis pickle bytes), accepting run-to-run ephemeral values (1.5h).
3. **[Medium]** Accept and merge the pull request after confirming `git diff --name-status` reports exactly one added file and no source/config modification (0.5h).
4. **[Low — separate backlog, out of AAP scope]** Triage the two documented findings (Q4 session-fixation rotation; Q2 null-guard) as independent application changes if the team chooses to remediate them.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

All completed hours were delivered autonomously (AI). Each component traces to a specific AAP requirement (runtime foundation, the eight per-question probes, document authoring, cleanup, and autonomous validation).

| Component | Hours | Description |
|-----------|-------|-------------|
| Runtime foundation stand-up | 5.0 | Launch Redis 6 + PostgreSQL 13, derive `.env` from `example.env` (+`MEM_STORE_URI`, +`GNUPGHOME`), `alembic upgrade head` (→ head `32f25cbf12f6`, 77 tables), start gunicorn `:7777` and `email_handler.py` `:20381`, including config debugging |
| Seed fixtures via real entry points | 1.5 | `flask dummy-data` seeding user `john@wick.com`, alias, mailbox, and API keys `code`/`codeFF` |
| Q1 probe + write-up | 1.5 | Session + no `Authentication` header → HTTP 200; neither → HTTP 401 `{"error":"Wrong api key"}`; raw-socket capture |
| Q2 probe (both variants) + write-up | 2.5 | Browser-session-only → HTTP 500 `AttributeError` (+ traceback); API key without sudo → HTTP 440 "Need sudo" |
| Q3 probe + write-up | 3.0 | Redis raw bytes (`\x80\x04` pickle proto 4), `pickletools.dis`, 6 session keys, `session:<uuid4>` key structure, `slapp` cookie structure, TTLs |
| Q4 probe (before/after ×2 runs) + write-up | 2.0 | Capture `slapp` cookie before vs after login; identifier PRESERVED (session-fixation), stable across 2 runs |
| Q5 probe (3 headers, 2 demonstrations) + write-up | 5.0 | Craft message with all three headers; canonical `NOT_SEND_EMAIL` log demo + full wire capture via purpose-built SMTP sink + 2nd handler |
| Q6 probe (boundary + real clock) + write-up | 2.5 | Sign suffix, sweep the 600s boundary (≤600 VALID / ≥601 EXPIRED), stable across 2 backdated runs + real wall-clock (598.1s / 602.0s) |
| Q7 probe (before/during/after) + write-up | 2.0 | Read `api_key` row across N=5 API-key calls: `times` 0→5, `last_used` stamped; session-fallback no-op contrast |
| Q8 probe (both variants) + write-up | 2.0 | Wrong password & unknown user → HTTP 200 re-render (CL 7017/7026), flash "Email or password incorrect", access-log + telemetry note |
| Document assembly & structure | 8.0 | 2,100-line / 10,938-word document: intro, canonical-runtime section, six-part per-question format, 22-row coverage-pass table, cause→effect rationale, sensitive-value note |
| CP4-review remediation rewrite | 5.0 | Full rewrite resolving the checkpoint review (8 Major + 1 Minor): re-probe, re-capture evidence, restructure (commit `f594e983`, +1877/−569) |
| Cleanup + read-only guarantee | 1.0 | Remove `/tmp/qprobes`, stop transient sink/2nd handler, verify single-file `git diff` |
| Autonomous validation (Final Validator) | 4.0 | Re-run full stack, re-derive all 8 answers byte-for-byte, run `pytest` (639 passed), verify citations + well-formedness |
| **Total Completed** | **45.0** | |

### 2.2 Remaining Work Detail

All remaining work is human path-to-production for the deliverable. Each item traces to acceptance of the AAP deliverable. (The two documented findings are **not** listed here — remediating them is out of AAP scope; see Section 8.)

| Category | Hours | Priority |
|----------|-------|----------|
| SME technical review of the answer document (verify Q1–Q8 answers, citations, coverage-pass table) | 3.5 | High |
| Independent spot-reproduction of ≥2 probes in the canonical image (e.g., Q6 timing, Q3 Redis bytes) | 1.5 | Medium |
| PR acceptance & merge (confirm single-file diff, no source/config changes) | 0.5 | Medium |
| **Total Remaining** | **5.5** | |

### 2.3 Hours Summary

| Metric | Hours |
|--------|-------|
| Section 2.1 — Completed Work total | 45.0 |
| Section 2.2 — Remaining Work total | 5.5 |
| **Total Project Hours (2.1 + 2.2)** | **50.5** |
| Percent Complete (45.0 ÷ 50.5) | 89.1% |

> Cross-section integrity: Section 2.1 (45.0h) + Section 2.2 (5.5h) = 50.5h, matching the Total Hours in Section 1.2. The Remaining total (5.5h) matches Section 1.2 and the Section 7 pie chart.

---

## 3. Test Results

All results below originate from Blitzy's autonomous validation logs for this project. This was a read-only task that authored **no new tests**; the existing repository suite was executed by the Final Validator (GATE 1) to confirm runtime health, and the eight runtime probes (GATE 2) each reproduced byte-for-byte.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|------------|-------|
| Repository regression suite | pytest | 639 | 639 | 0 | Not measured | Existing SimpleLogin suite; run via `.venv/bin/pytest` → "639 passed, 51 warnings in 162.97s" (exit 0); isolated test DB on port 15432. Independently re-confirmed: 639 tests collected. |
| Runtime behavior probes (Q1–Q8) | Custom probe scripts (raw socket / `requests` / Redis / SMTP) | 8 | 8 | 0 | N/A | Each of the 8 documented answers re-derived byte-for-byte against the live stack in GATE 2; detailed in Section 4. |
| **Total** | | **647** | **647** | **0** | | 100% pass rate |

**Notes on coverage:** No coverage instrumentation was reported by the autonomous validation, and no coverage figure is fabricated here. The read-only mandate precluded authoring tests, so coverage of new code is not applicable (no application code was added or modified).

---

## 4. Runtime Validation & UI Verification

**Runtime health (canonical stack, observed live):**

- ✅ **PostgreSQL 13** — Operational. `pg_isready` → accepting connections; `alembic current` → `32f25cbf12f6 (head)`; 2 seed users present.
- ✅ **Redis 6** — Operational. `redis-cli ping` → `PONG`; server-side session keys present (`session:<uuid4>`), TTL 604800s (auth) / 300s (anon).
- ✅ **Gunicorn web app (`wsgi:app`, `:7777`)** — Operational. `GET /` → HTTP 302; `GET /auth/login` → HTTP 200.
- ✅ **Inbound SMTP handler (`email_handler.py`, `:20381`)** — Operational. Listening; processed the crafted Q5 message end-to-end.

**Per-question runtime validation (all reproduced byte-for-byte):**

- ✅ **Q1** — Session + no `Authentication` header → HTTP 200 (user_info JSON via `current_user` fallback); neither → HTTP 401 `{"error":"Wrong api key"}` (26 bytes).
- ✅ **Q2** — Browser-session-only → HTTP 500 `{"error":"Internal error"}` (`AttributeError: 'NoneType' object has no attribute 'sudo_mode_at'`); API key without sudo → HTTP 440 `{"error":"Need sudo"}` (22 bytes). Both seed users confirmed still present (no deletion occurred).
- ✅ **Q3** — `slapp = <uuid4>.<hmac-sig>`; storage key `session:<uuid4>`; Python pickle protocol 4 (`\x80\x04`); 6 authenticated-session keys; TTL 604800s.
- ✅ **Q4** — Session identifier PRESERVED (uuid4 and full signed cookie byte-identical before/after login), stable across 2 runs — session-fixation finding.
- ✅ **Q5** — `X-Custom-Test` STRIPPED, `Received` STRIPPED, `Reply-To` REPLACED with a reverse-alias address; confirmed via canonical log **and** independent wire capture.
- ✅ **Q6** — Expiry window 600s inclusive (age ≤600 VALID, ≥601 EXPIRED); stable across 2 backdated runs + real wall-clock (598.1s VALID / 602.0s EXPIRED).
- ✅ **Q7** — `times` +1 per call (0→5), `last_used` stamped fresh each call; `code` and `sudo_mode_at` unchanged; session-fallback path is a no-op on the row.
- ✅ **Q8** — Wrong credentials → HTTP 200 re-render (not 401/redirect); Content-Length 7017 (bad password) / 7026 (unknown user); flash "Email or password incorrect" (identical for both — no user enumeration).

**UI verification:** ⚠ **Partial (by design).** This is an API/backend runtime investigation; the only UI surface exercised is the `/auth/login` form, whose server-rendered HTML was captured verbatim for Q8 (byte-lengths 7017 / 7026). No broader UI verification was in scope, and none is required by the AAP.

---

## 5. Compliance & Quality Review

Cross-mapping of AAP deliverables and governing rules to their validated status.

| AAP / Rule Requirement | Benchmark | Status | Progress | Fixes Applied During Validation |
|------------------------|-----------|--------|----------|-------------------------------|
| Single deliverable `blitzy/documentation/app_2cd6ee777f8c.md` (CREATE) | One new Markdown file, correctly named after branch | ✅ Pass | 100% | None required |
| Run-first: answers from observed runtime, not reading | Live probes for all 8 questions | ✅ Pass | 100% | None required |
| No source/config file modified (read-only) | `git diff` = exactly one added file | ✅ Pass | 100% | None required |
| Transient scripts removed | `/tmp/qprobes` absent; repo clean | ✅ Pass | 100% | None required |
| Exact status codes / verbatim strings (Q1, Q2, Q8) | Byte-exact bodies, lengths, flash text | ✅ Pass | 100% | Q5 citation + Q4 prose corrected (commit `bcff3166`) |
| Both Q2 variants covered | HTTP 500 + HTTP 440 | ✅ Pass | 100% | None required |
| All three Q5 headers by name | `X-Custom-Test`, `Received`, `Reply-To` | ✅ Pass | 100% | Wire-capture demo added to overcome canonical logging limit |
| Before/during/after state (Q4, Q7) | Before/after tables present | ✅ Pass | 100% | None required |
| Q6 ≥2-run timing stability | 2 backdated runs + real wall-clock | ✅ Pass | 100% | None required |
| Evidence discipline (cmd + unedited output + citation + rationale) | Present for every probe | ✅ Pass | 100% | Full rewrite resolved CP4 review (8 Major, 1 Minor; commit `f594e983`) |
| Citations resolve to real source | `file:line` + function verified | ✅ Pass | 100% | Q5 `MIME_HEADERS` citation line corrected |
| Findings reported, not remediated | Q2-500 & Q4 fixation documented, unfixed | ✅ Pass | 100% | N/A (report-only by mandate) |
| Repository regression suite green | `pytest` 639 passed | ✅ Pass | 100% | None required (no code changed) |
| Document well-formedness | Balanced code fences, no placeholders | ✅ Pass | 100% | 78 balanced fences; 0 TODO/placeholder |

**Quality summary:** The deliverable satisfies every AAP rule. Quality issues raised during the checkpoint review (CP4: 8 Major + 1 Minor) were fully resolved via the rewrite in commit `f594e983` and the follow-up correction in `bcff3166`. The Final Validator required **zero additional corrections**.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Q4 session-fixation: session id not rotated on login (`app/session.py:68-80`) | Security (application finding) | High | High | Documented with before/after ×2 evidence; recommend app-side session rotation as a separate follow-up | Reported, not remediated (out of AAP scope) |
| Q2(a) HTTP 500 `AttributeError` on browser-session-only `DELETE /api/user` (`app/api/base.py:47`) | Security / Robustness (application finding) | Medium | High | Documented with verbatim traceback; recommend null-guard on `api_key` as a separate follow-up | Reported, not remediated (out of AAP scope) |
| Ephemeral run-to-run values (UUIDs, HMAC signatures, timestamps, reverse-alias local-parts) differ each run | Technical | Low | Medium | Prominent sensitive-value note states *structure/behavior* are the reproducible facts; exact bytes regenerate | Mitigated (documented) |
| Displayed fixture secrets could be mistaken for production secrets | Security | Low | Low | Note clarifies all are throwaway local fixtures from `example.env` + `flask dummy-data` | Mitigated (documented) |
| Reproducibility depends on exact Docker image + 2 required `.env` additions (`MEM_STORE_URI`, `GNUPGHOME`) | Operational | Low | Medium | Document states exact image, canonical versions (Py 3.10 / PG 13 / Redis 6), and the additions | Mitigated (documented) |
| Q5(b) wire capture used deliberately non-canonical config (SMTP sink + 2nd handler, `NOT_SEND_EMAIL` disabled) | Operational | Low | N/A | Explicitly labeled non-canonical; canonical Q5(a) provided alongside | Mitigated (labeled) |
| Citation line-number drift if source changes later | Technical | Low | Low | Source frozen on branch; all citations verified to resolve at current HEAD | Mitigated |
| No external/third-party integration exercised | Integration | Low | Low | Not applicable — local-only stack; `NOT_SEND_EMAIL=true` prevents outbound mail; deliverable is plain Markdown with zero runtime deps | N/A (no integration surface) |
| Human-review bottleneck (only remaining work is SME sign-off) | Process | Medium | Medium | Assign a security/backend SME; remaining scoped at 5.5h (Section 2.2) | Open (expected human step) |

**Note:** Because no application code was changed, there is no CI/deployment regression risk for the application itself — the 639-test suite remains green.

---

## 7. Visual Project Status

**Project hours breakdown (Completed = Dark Blue `#5B39F3`, Remaining = White `#FFFFFF`):**

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeWidth':'2px','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#111111','pieLegendTextColor':'#111111'}}}%%
pie showData title Project Hours — Completed vs Remaining
    "Completed Work" : 45.0
    "Remaining Work" : 5.5
```

**Remaining hours by category (from Section 2.2, total 5.5h):**

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#B23AF2','pie3':'#A8FDD9','pieStrokeColor':'#5B39F3','pieOuterStrokeColor':'#5B39F3','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#111111','pieLegendTextColor':'#111111'}}}%%
pie showData title Remaining Work by Category (Hours)
    "SME technical review [High]" : 3.5
    "Spot-reproduction of probes [Medium]" : 1.5
    "PR acceptance & merge [Medium]" : 0.5
```

> Integrity check: the pie "Remaining Work" value (5.5h) equals the Section 1.2 Remaining Hours and the sum of the Section 2.2 Hours column (3.5 + 1.5 + 0.5 = 5.5). "Completed Work" (45.0h) equals Section 1.2 Completed Hours and the Section 2.1 total.

---

## 8. Summary & Recommendations

**Achievements.** The project delivered a comprehensive, runtime-evidence answer document for all eight SimpleLogin questions, produced strictly by running and observing the live system. Every answer leads with the direct result and includes the exact command, complete unedited output, a `file:line` + function citation, and a cause→effect rationale. Coverage is exhaustive: both Q2 variants, all three Q5 headers by name, before/after state for Q4 and Q7, and a ≥2-run stability confirmation (plus a real wall-clock run) for the Q6 600-second boundary. The investigation exceeded the minimum where the canonical configuration was insufficient — adding a wire-level Q5 capture — and reported two security-relevant findings honestly rather than smoothing them over.

**Remaining gaps & critical path to production.** The project is **89.1% complete** (45.0 of 50.5 hours). The critical path is short and entirely human: (1) SME technical review of the document, (2) independent spot-reproduction of a couple of probes, and (3) PR acceptance and merge — 5.5 hours total. No autonomous rework remains; the deliverable is authored, byte-for-byte validated, committed, and the repository is clean with exactly one net change.

**Out-of-scope follow-ups (separate backlog).** The two documented findings — Q4 session-fixation (rotate the session id on login, ~4–6h if pursued) and Q2's HTTP 500 `AttributeError` (add a null-guard, ~2–3h if pursued) — are application changes that the read-only mandate explicitly prohibited. They are surfaced here for visibility and are **not** included in this project's remaining hours.

**Success metrics.** All eight answers reproduce byte-for-byte; 639/639 repository tests pass; 78/78 code fences balanced; 0 placeholders; 100% of sampled citations resolve; `git diff` = exactly one added file.

**Production-readiness assessment.** The deliverable is **production-ready** for a documentation artifact — accurate, complete, correctly cited, well-formed, and reproducible from the stated canonical configuration. It is recommended for acceptance and merge following SME review. Confidence: **High** (well-defined scope, independent validation, zero corrections required by the Final Validator).

| Metric | Value |
|--------|-------|
| Completion | 89.1% (45.0 / 50.5 h) |
| Deliverables committed | 1 file (`blitzy/documentation/app_2cd6ee777f8c.md`, 2,100 lines) |
| Repository regression tests | 639 / 639 passed |
| Runtime probes reproduced | 8 / 8 byte-for-byte |
| Net repository change | 1 file added, 0 modified, 0 deleted |
| Remaining effort | 5.5 h (human review + merge) |

---

## 9. Development Guide

This guide explains how to reproduce the investigation runtime so a reviewer can independently re-derive any of the eight answers. All commands were verified in the canonical environment.

### 9.1 System Prerequisites

- **OS/Runtime:** Linux container — canonical Docker image `andrewparkscaleai/coding-agent:simple-login__app__2cd6ee777f8c…`.
- **Language/tools (verified):** Python **3.10.20**, `gunicorn` **20.0.4**, `alembic` **1.4.3**, Docker Engine (for `postgres:13` and `redis:6`).
- **Dependencies:** the in-repo virtualenv `.venv` with all `poetry.lock` versions installed (Flask 1.1.2, flask-login 0.5.0, werkzeug 1.0.1, itsdangerous 1.1.0, redis 4.6.0, sqlalchemy 1.3.24, aiosmtpd 1.4.2, gunicorn 20.0.4, and others).

### 9.2 Environment Setup

```bash
# 1) Backends (canonical versions)
docker run -d --name sl-postgres -e POSTGRES_USER=myuser -e POSTGRES_PASSWORD=mypassword \
    -e POSTGRES_DB=simplelogin -p 5432:5432 -p 15432:5432 postgres:13
docker run -d --name sl-redis -p 6379:6379 redis:6

# 2) Configuration — derive .env from example.env, then add the two required knobs
cp example.env .env
printf 'MEM_STORE_URI=redis://localhost:6379\nGNUPGHOME=/tmp/gnupg\n' >> .env
# example.env already sets: URL=http://localhost:7777, EMAIL_DOMAIN=sl.local,
# NOT_SEND_EMAIL=true, DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin, FLASK_SECRET=secret
```

> **Important:** Redis-backed sessions are only enabled when `MEM_STORE_URI` is set — omitting it means Q3/Q4 will not reproduce. `GNUPGHOME` must point to a creatable directory or the app will error at boot.

### 9.3 Dependency Installation

Dependencies are pre-installed in `.venv`. To verify (no changes needed):

```bash
.venv/bin/python --version           # -> Python 3.10.20
.venv/bin/pip list | grep -Ei '^(Flask|redis|itsdangerous|SQLAlchemy|gunicorn|aiosmtpd) '
```

### 9.4 Schema, Seed & Application Startup

```bash
# Schema (app DB) -> head 32f25cbf12f6, 77 tables
.venv/bin/alembic upgrade head

# Seed fixtures through real entry points
FLASK_APP=wsgi.py .venv/bin/flask dummy-data
# Seeds: user john@wick.com / password; API keys code + codeFF;
#        alias begins_cashew220@sl.local -> mailbox john@wick.com

# Web application (canonical run command)
nohup .venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15 > /tmp/gunicorn.log 2>&1 &

# Inbound SMTP handler (separate process, listens on :20381)
nohup .venv/bin/python email_handler.py > /tmp/email_handler.log 2>&1 &
```

### 9.5 Verification Steps

```bash
docker exec sl-redis redis-cli ping                 # -> PONG
docker exec sl-postgres pg_isready                  # -> accepting connections
.venv/bin/alembic current                           # -> 32f25cbf12f6 (head)
curl -s -o /dev/null -w "HTTP %{http_code}\n" http://localhost:7777/   # -> HTTP 302
.venv/bin/pytest                                    # -> 639 passed (isolated test DB :15432)
```

### 9.6 Example Usage (reproducing a probe)

API probes must use the **non-standard `Authentication:` header** (not `Authorization:`) per `docs/api.md:73`. Write ephemeral probe scripts **outside** the repository tree and delete them afterward (the read-only mandate forbids committing anything but the answer document):

```bash
# Q1 example — an unauthenticated call (no session, no header) returns 401
curl -s -o /dev/null -w "no-auth, no-session -> HTTP %{http_code}\n" http://localhost:7777/api/user_info
# -> HTTP 401  (body: {"error":"Wrong api key"})

# Recommended reviewer reproductions:
#  - Q6: sign an alias suffix and re-verify across the 600s boundary (app/alias_suffix.py:40, max_age=600)
#  - Q3: docker exec sl-redis redis-cli --scan --pattern 'session:*' ; then GET one key to inspect pickle bytes
```

### 9.7 Troubleshooting

- **Sessions not appearing in Redis (Q3/Q4 fail):** confirm `MEM_STORE_URI=redis://localhost:6379` is present in `.env`.
- **App fails at boot with GPG error:** ensure `GNUPGHOME=/tmp/gnupg` is set and the directory is creatable.
- **`:7777` bind error:** another process holds the port — free it or change `-b`.
- **No forwarded email appears for Q5:** `NOT_SEND_EMAIL=true` prints a summary instead of delivering — this is the intended canonical observation hook; use the handler DEBUG log for whitelist decisions, or an SMTP sink for the full wire message.
- **Test DB conflicts:** the suite uses the isolated DB on port 15432; keep it distinct from the app DB (5432).

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---------|---------|
| Start PostgreSQL 13 | `docker run -d --name sl-postgres -e POSTGRES_USER=myuser -e POSTGRES_PASSWORD=mypassword -e POSTGRES_DB=simplelogin -p 5432:5432 -p 15432:5432 postgres:13` |
| Start Redis 6 | `docker run -d --name sl-redis -p 6379:6379 redis:6` |
| Derive config | `cp example.env .env && printf 'MEM_STORE_URI=redis://localhost:6379\nGNUPGHOME=/tmp/gnupg\n' >> .env` |
| Migrate schema | `.venv/bin/alembic upgrade head` |
| Seed fixtures | `FLASK_APP=wsgi.py .venv/bin/flask dummy-data` |
| Run web app | `.venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` |
| Run SMTP handler | `.venv/bin/python email_handler.py` |
| Run tests | `.venv/bin/pytest` |
| Confirm read-only diff | `git diff --name-status 2cd6ee77 HEAD` |

### B. Port Reference

| Port | Service |
|------|---------|
| 7777 | Gunicorn web application (`wsgi:app`) |
| 20381 | Inbound SMTP handler (`email_handler.py`) |
| 5432 | PostgreSQL 13 (application database) |
| 15432 | PostgreSQL 13 (isolated test database used by `pytest`) |
| 6379 | Redis 6 (session store) |

### C. Key File Locations

| Path | Role |
|------|------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The deliverable** — runtime-evidence Q&A (2,100 lines) |
| `app/api/base.py` | `authorize_request()`, `require_api_sudo` (Q1, Q2, Q7) |
| `app/api/views/user.py` | `DELETE /api/user` sudo-gated endpoint (Q2) |
| `app/session.py` | `RedisSessionStore` — serialization, key structure, session id (Q3, Q4) |
| `app/config.py` | `SESSION_COOKIE_NAME = "slapp"`, `CUSTOM_ALIAS_SECRET` (Q3, Q6) |
| `email_handler.py` / `app/email_utils.py` / `app/email/headers.py` | Forwarding header whitelist & `Reply-To` re-add (Q5) |
| `app/alias_suffix.py` | `TimestampSigner`, `max_age=600` (Q6) |
| `app/models.py` | `ApiKey` model — `times`, `last_used` (Q7) |
| `app/auth/views/login.py` / `app/events/auth_event.py` | Login-failure branch & `LoginEvent` (Q8) |
| `docs/api.md` | Non-standard `Authentication` header contract |

### D. Technology Versions

| Component | Version |
|-----------|---------|
| Python | 3.10.20 |
| PostgreSQL | 13 |
| Redis | 6 |
| Flask | 1.1.2 |
| Flask-Login | 0.5.0 |
| Flask-WTF | 0.14.3 |
| Werkzeug | 1.0.1 |
| itsdangerous | 1.1.0 |
| redis (client) | 4.6.0 |
| SQLAlchemy | 1.3.24 |
| psycopg2-binary | 2.9.3 |
| arrow | 0.16.0 |
| aiosmtpd | 1.4.2 |
| dkimpy | 1.0.5 |
| gunicorn | 20.0.4 |
| alembic | 1.4.3 |

### E. Environment Variable Reference

| Variable | Value (canonical) | Purpose |
|----------|-------------------|---------|
| `URL` | `http://localhost:7777` | Base application URL |
| `EMAIL_DOMAIN` | `sl.local` | Alias/email domain |
| `NOT_SEND_EMAIL` | `true` | Print outbound mail instead of sending (Q5 observation hook) |
| `DB_URI` | `postgresql://myuser:mypassword@localhost:5432/simplelogin` | PostgreSQL connection |
| `MEM_STORE_URI` | `redis://localhost:6379` | Enables Redis-backed sessions (required for Q3/Q4) |
| `GNUPGHOME` | `/tmp/gnupg` | GPG home; required at boot |
| `FLASK_SECRET` | `secret` (fixture) | Signs `slapp` cookie; derives `CUSTOM_ALIAS_SECRET` |

> All values above are local, throwaway fixtures — not production secrets.

### F. Developer Tools Guide

- **Inspect Redis session bytes:** `docker exec sl-redis redis-cli --scan --pattern 'session:*'` then `GET <key>` (raw pickle bytes begin with `\x80\x04`).
- **Disassemble pickle:** `python -c "import pickletools,sys; pickletools.dis(open(sys.argv[1],'rb').read())"`.
- **Read `api_key` usage (Q7):** `docker exec sl-postgres psql -U myuser -d simplelogin -c "SELECT code, times, last_used, sudo_mode_at FROM api_key;"`.
- **Tail runtime logs:** `tail -f /tmp/gunicorn.log /tmp/email_handler.log` (access-log lines are emitted by `after_request` at `server.py:284`).
- **Confirm read-only guarantee:** `git status --porcelain` (clean) and `git diff --name-status 2cd6ee77 HEAD` (one added file).

### G. Glossary

| Term | Definition |
|------|------------|
| AAP | Agent Action Plan — the governing specification for this task |
| `slapp` | SimpleLogin's session cookie name (`app/config.py:199`) |
| Session fixation | A vulnerability where the session id is not rotated after login (Q4 finding) |
| Sudo mode | A time-limited elevated state required for privileged API operations (Q2) |
| Reverse-alias | The SimpleLogin address that replaces `From`/`Reply-To` so replies route back through the service (Q5) |
| `TimestampSigner` | `itsdangerous` signer that embeds a timestamp, enabling `max_age` expiry (Q6, `max_age=600`) |
| HTTP 440 | Non-standard status (originally IIS "Login Timeout") repurposed by SimpleLogin with a "Need sudo" body (Q2) |
| Byte-for-byte | Output reproduced exactly, modulo declared ephemeral values (UUIDs, signatures, timestamps) |
