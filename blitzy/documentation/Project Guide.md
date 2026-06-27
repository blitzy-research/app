# Blitzy Project Guide

> **Project:** SimpleLogin Behavioral Investigation — Evidence-Backed Q&A
> **Branch:** `blitzy-d8553dc1-1eb7-4446-aafe-873833d03215` (base `app_2cd6ee777f8c` @ `2cd6ee77`)
> **Deliverable:** `blitzy/documentation/app_2cd6ee777f8c.md`
> **Brand colors:** Completed/AI = Dark Blue `#5B39F3` · Remaining = White `#FFFFFF` · Accents = Violet-Black `#B23AF2` · Highlight = Mint `#A8FDD9`

---

## 1. Executive Summary

### 1.1 Project Overview

This project is a **read-only, investigative Question-and-Answer documentation task** against the **SimpleLogin** Python/Flask email-aliasing application. The objective was to empirically determine and document the *actual, observed* behavior of eight authentication, session, email-forwarding, and API scenarios by **building and running** the system, capturing live runtime evidence (HTTP status codes, raw session bytes, before/after values, database rows, forwarded headers, log lines), and corroborating every finding against the source code as ground truth. The target users are engineers and security reviewers who need authoritative, reproducible answers rather than assumptions. The technical scope is cross-cutting in its *reach* (API authorization, server-side sessions, the email forward pipeline, login handling) but strictly **additive** in its *output*: exactly one new Markdown document, with no source file modified.

### 1.2 Completion Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeWidth':'2px','pieSectionTextColor':'#111111','pieLegendTextColor':'#111111','pieTitleTextSize':'16px'}}}%%
pie showData title Completion Status — 95.0% Complete (38h of 40h)
    "Completed (AI)" : 38
    "Remaining" : 2
```

| Metric | Hours |
|---|---|
| **Total Hours** | **40** |
| **Completed Hours (AI + Manual)** | **38** (AI: 38 · Manual: 0) |
| **Remaining Hours** | **2** |
| **Percent Complete** | **95.0%** |

> **Calculation (PA1, AAP-scoped):** Completion % = Completed ÷ (Completed + Remaining) = 38 ÷ (38 + 2) = 38 ÷ 40 = **95.0%**. All hours trace to AAP-scoped investigation work or path-to-production gates; no out-of-scope work is included.

### 1.3 Key Accomplishments

- ✅ **All 8 behavioral questions answered** with live runtime evidence + corroborating `path:line` code citations + explicit rationale.
- ✅ **System built, run, and seeded** in the prescribed container (web `127.0.0.1:7777`; inbound SMTP `:20381`; Postgres migrated to 77 tables; Redis session store; `john@wick.com`/`password` + API keys `code`/`codeFF`).
- ✅ **Single deliverable produced** — `blitzy/documentation/app_2cd6ee777f8c.md` (856 lines, ~5,500 words), structured as Environment & Method + Q1–Q8 + "Assumptions corrected" + Summary table.
- ✅ **Citation integrity:** 51 distinct `path:line` citations across 19 source files — independently re-verified at 100% accuracy against commit `2cd6ee77`.
- ✅ **Findings exceeded the AAP hypothesis** where reality differed: Q7 surfaced a **third** mutated DB column (`updated_at` via `onupdate`); Q2 captured the pure-session **HTTP 500** root cause (`NoneType.sudo_mode_at`).
- ✅ **Common assumption corrected:** session bytes are **Python `pickle`**, not PHP serialization (raw bytes begin `\x80\x04\x95`).
- ✅ **All constraints honored:** zero source files modified (clean tree), all ephemeral probes deleted, code-as-truth + rationale rules satisfied.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|---|---|---|---|
| *None — deliverable is complete and fully validated* | No blocker to release or validation; Final Validator found zero discrepancies and required no corrections | — | — |

> There are **no critical unresolved issues** in the deliverable. The security-relevant *findings* surfaced by the investigation (e.g., no session-id rotation on login, a latent pure-session HTTP 500) are **app behaviors documented by design** — remediation is explicitly **out of scope** for this task and is listed as optional, informational follow-up in §1.6 / §8.

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|---|---|---|---|---|
| — | — | No access issues identified | N/A | — |

**No access issues identified.** The investigation ran entirely inside the self-contained prescribed container (Python 3.10 + Postgres + Redis bundled). No external repository permissions, third-party API keys, or service credentials were required to build, run, seed, or probe the system.

### 1.6 Recommended Next Steps

1. **[High]** Conduct **SME review & acceptance** of the eight answers — read the deliverable and spot-verify a few high-value answers (suggested: Q3 pickle bytes/key set, Q5 header survival, Q7 three-field set) against the running container or source `@2cd6ee77`. *(~1.5h)*
2. **[Medium]** **Approve the PR, merge** to the target branch, and **publish/index** the document where team documentation lives. *(~0.5h)*
3. **[Low · out of scope]** Open a **separate backlog ticket** to triage the latent pure-session **HTTP 500** (`NoneType.sudo_mode_at`, `app/api/base.py:L47`) discovered in Q2.
4. **[Low · out of scope]** Evaluate adding **session-identifier rotation on login** (session-fixation hardening) surfaced in Q4, and review **pickle-based session storage** surfaced in Q3.
5. **[Low · out of scope]** Optionally add a lightweight **CI/periodic check** to re-verify the document's `path:line` citations against source if the document is retained long-term.

> Items 3–5 are **out of AAP scope** (this task documents behavior; it does not remediate it) and carry **zero hours** in the 40h project total.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|---|---:|---|
| Runtime environment establishment | 3.0 | Build/run prescribed container; `cp example.env .env`; `alembic upgrade head` (77 tables); `flask dummy-data`; launch `server.py` (web :7777) + `email_handler.py` (SMTP :20381); verify services |
| Q1 — API auth without key | 2.0 | Login/CSRF flow + cookie jar; session-only `GET /api/user_info` → 200 (8-key body); no-auth contrast → 401; corroborate `app/api/base.py:L16-43` |
| Q2 — Privileged-op status code | 2.5 | Capture HTTP **440** `{"error":"Need sudo"}` (valid key, sudo inactive) **and** pure-session **HTTP 500** root cause (`NoneType.sudo_mode_at`); `PATCH /api/sudo` 403; corroborate `app/api/base.py:L46-72`, `app/api/views/sudo.py` |
| Q3 — Session storage forensics | 3.0 | Read Redis `session:<uuid4>`; decode raw bytes → Python pickle proto-4; enumerate key set; unsign `slapp`; capture TTL; corroborate `app/session.py`, `server.py:L207`, `app/paddle_utils.py:L51` |
| Q4 — Session-id rotation before/after | 2.0 | Capture `slapp` before/after login → byte-identical (reused, not rotated); corroborate `app/session.py:L68-121`, `app/extensions.py:L8` |
| Q5 — Forwarded-email header handling | 4.0 | Real inbound SMTP to :20381; **dual captures** (NOT_SEND_EMAIL summary + disposable local sink via temp env) → X-header & Received **stripped**, Reply-To **rewritten**; corroborate `email_handler.py:L793-873`, `app/email/headers.py` |
| Q6 — Alias-token expiry boundary | 2.5 | Real-time prober (~10-min wait) + back-dated boundary sweep → **600s** window; corroborate `app/alias_suffix.py:L11,L40`, `app/config.py:L201` |
| Q7 — API-key usage statistics | 3.0 | Full-row `SELECT *` baseline + N calls + single-call + invalid-key controls → **3 fields** change (`times`, `last_used`, `updated_at`); corroborate `app/api/base.py:L30-32`, `app/models.py:L65,L2350-2360` |
| Q8 — Failed-login behavior | 2.0 | Wrong-creds POST → HTTP **200** re-render + flash "Email or password incorrect" + New Relic `LoginEvent.failed` (no error log line); corroborate `app/auth/views/login.py`, `app/events/auth_event.py:L22-25` |
| External web research | 1.5 | Characterize non-standard **HTTP 440** (Microsoft IIS "Login Time-out") for Q2; Flask/Flask-Login **session-fixation** framing for Q4 |
| Deliverable synthesis & authoring | 6.0 | Write 856-line Markdown: Environment & Method + 8 question sections (verbatim Q / method / evidence / citations / answer / rationale) + "Assumptions corrected" + Summary table |
| Citation integrity pass | 2.0 | Map every claim to exact `path:line`; 51 citations across 19 files; reconcile evidence ↔ code |
| Autonomous validation | 4.0 | Final Validator re-ran **all 8 probes** end-to-end and re-verified **100% of citations** @ `2cd6ee77`; structural Markdown checks; zero discrepancies |
| Constraint compliance & cleanup | 0.5 | Delete all ephemeral probes; verify read-only (no source modified); confirm clean tree |
| **Total Completed** | **38.0** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|---|---:|---|
| Human SME review & answer acceptance (read the deliverable; spot-verify select answers vs. code/runtime; sign off) | 1.5 | High |
| PR approval, merge & documentation publish (confirm only the deliverable is added; approve; merge; index/link) | 0.5 | Medium |
| **Total Remaining** | **2.0** | |

> **Cross-check:** §2.1 Completed (38.0) + §2.2 Remaining (2.0) = **40.0** Total Hours (matches §1.2). §2.2 Remaining (2.0) matches §1.2 Remaining and the §7 pie "Remaining Work".

### 2.3 Notes on Estimation

Estimates use the PA2 framework, anchored to actual investigation complexity rather than lines of code (this is a documentation task; the only code-volume metric is the 856-line deliverable). The heaviest items reflect genuine experimental effort: **Q5** required a real SMTP ingress→forward pipeline with two capture strategies; **Q3/Q7** required storage- and database-level forensics; **Q6** required a time-boundary experiment. Confidence is **High** — every estimate maps to a concrete, already-completed and independently re-validated artifact.

---

## 3. Test Results

For this investigative Q&A task, the AAP **explicitly relies on the live application plus disposable probes rather than the pytest suite**. Accordingly, Blitzy's autonomous validation vehicle is the **empirical reproduction of all eight questions** against the running system. Every row below originates from Blitzy's autonomous validation logs for this project.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---|---|---:|---:|---:|---:|---|
| Behavioral reproduction (Q1–Q8) | Disposable runtime probes (curl / redis-cli + pickle / smtplib / token-timing / psql) vs. live app | 8 | 8 | 0 | 100% of in-scope questions | Each probe re-run by the Final Validator; observed values match the deliverable exactly |
| Code-citation verification | Manual source cross-check @ `2cd6ee77` | 51 | 51 | 0 | 100% of citations | All `path:line` references resolve to the cited behavior; zero mismatches |
| Runtime service health | Live process checks | 2 | 2 | 0 | n/a | Web `server.py` (:7777) answered authenticated API 200; SMTP `email_handler.py` (:20381) announced `220 … Python SMTP 1.4.2` |
| Deliverable structural integrity | Markdown lint / fence + placeholder scan | 4 | 4 | 0 | n/a | 856 lines; 8 Q-sections; 56 balanced code fences; 0 TODO/FIXME/placeholder |
| **Total** | | **65** | **65** | **0** | **100%** | **Zero discrepancies across all autonomous validation** |

> **Integrity note:** No results are drawn from the project's `pytest` suite (`scripts/run-test.sh`, `pytest.ci.ini`); per the AAP that harness is available for reference only and is **not** the validation vehicle for this task. All figures above come strictly from Blitzy's autonomous reproduction and citation-verification logs.

---

## 4. Runtime Validation & UI Verification

**Runtime health (all observed live in the prescribed container):**

- ✅ **Web application** (`server.py`, `127.0.0.1:7777`) — **Operational.** Answered an authenticated API call with HTTP 200; served the login page and dashboard.
- ✅ **Inbound SMTP handler** (`email_handler.py`, `:20381`) — **Operational.** Announced `220 … Python SMTP 1.4.2`; processed a real ingress→forward transaction end-to-end.
- ✅ **Redis server-side session store** — **Operational.** `session:<uuid4>` key present; pickle payload decoded; `slapp` cookie unsigned to the matching UUID; TTL ~604,767s (~7 days).
- ✅ **PostgreSQL database** — **Operational.** 77 tables migrated via `alembic upgrade head`; seeded fixtures present; `api_key` row reads/updates observed.
- ✅ **Browser-session login flow** — **Operational.** `POST /auth/login` → 302 → `/dashboard/` (200) on valid credentials; re-render (200) + flash on invalid.

**API integration outcomes (observed status codes & bodies):**

- ✅ **Session-only API call (Q1)** — `GET /api/user_info` → **HTTP 200** with the expected 8-key JSON body.
- ✅ **No-auth API call (Q1)** — → **HTTP 401** `{"error":"Wrong api key"}`.
- ✅ **Sudo-gated endpoint, valid key + sudo inactive (Q2)** — → **HTTP 440** `{"error":"Need sudo"}`.
- ⚠ **Sudo-gated endpoint, pure browser session (Q2)** — → **HTTP 500** (latent `NoneType.sudo_mode_at`). *Captured and documented as a finding; remediation out of scope.*
- ✅ **Keyed API calls (Q7)** — repeated `GET /api/user_info` advance `times`, `last_used`, `updated_at`; invalid key → 401 with row unchanged.
- ✅ **Email forward pipeline (Q5)** — inbound message forwarded with custom `X-` header & `Received` **stripped**, `From`/`Reply-To` **rewritten** to reverse-alias.

**UI verification:**

- ✅ **Login page render & error flash** — **Operational** (exercised as part of Q1/Q8: page renders, accepts POST, flashes "Email or password incorrect").
- ➖ **No UI feature was built or modified** in this task. There are **no Figma frames, design-system references, or front-end deliverables** in scope, so visual/Design-System Compliance verification is **Not Applicable**.

---

## 5. Compliance & Quality Review

This task is governed by the user's **SWE-AtlasQnA-Repo** rule set. The matrix below cross-maps each directive to its outcome under Blitzy's autonomous validation.

| # | AAP / Rule Requirement | Benchmark | Status | Progress |
|---|---|---|---|---|
| 1 | Produce one answer doc named `<branch>.md` | `blitzy/documentation/app_2cd6ee777f8c.md` exists (856 lines) | ✅ Pass | 100% |
| 2 | Place it under `blitzy/documentation/` in the destination repo | Correct path confirmed | ✅ Pass | 100% |
| 3 | Build & run the system to observe real behavior | Web :7777 + SMTP :20381 run; DB migrated + seeded | ✅ Pass | 100% |
| 4 | Code is the source of truth (no assumptions) | 51 citations, 100% verified @ `2cd6ee77`; pickle/PHP & 440 assumptions explicitly corrected | ✅ Pass | 100% |
| 5 | Provide rationale for each answer | Every Q has a dedicated "Rationale" subsection | ✅ Pass | 100% |
| 6 | Do not modify existing source files | `git diff` shows only the deliverable added (status `A`); clean tree | ✅ Pass | 100% |
| 7 | Add no other code; delete ephemeral probes | No leftover probe scripts; no other tracked additions | ✅ Pass | 100% |
| 8 | Show evidence from actually running the system | Live status codes, raw bytes, before/after values, DB rows, log lines captured | ✅ Pass | 100% |
| 9 | Capture exact, observed values | Exact codes/messages/bytes/keys/headers/boundary/fields recorded verbatim | ✅ Pass | 100% |
| 10 | Answer all eight questions | Q1–Q8 each fully addressed + consolidated summary table | ✅ Pass | 100% |

**Fixes applied during autonomous validation:** The investigation self-corrected as evidence demanded — **Q7** was revised from a 2-field to a **3-field** answer after a full-row `SELECT *` revealed `updated_at` moving via `onupdate` (commit `9180efbf`); **Q5** evidence was upgraded to a real inbound-SMTP capture and the Postgres version was reconciled (commit `cc9d572a`); a Q7 summary-table citation path was corrected (commit `15b17a6f`). The Final Validator's independent re-run required **no further corrections**.

**Outstanding compliance items:** None. Quality posture is **production-ready** for a documentation deliverable.

---

## 6. Risk Assessment

Overall posture is **Low**: a read-only, fully-validated documentation deliverable with zero source changes. The notable items (R3–R5) are **app behaviors the investigation was designed to surface** — findings, not deliverable defects — and are out of scope to remediate.

| Risk | Category | Severity | Probability | Mitigation | Status |
|---|---|---|---|---|---|
| R1 — Findings pinned to commit `2cd6ee77` + legacy deps; `path:line` citations could drift if source evolves | Technical | Low | Medium | Every claim anchored to commit hash + `path:line`; re-verify if rebased onto newer code | Documented / Accepted |
| R2 — AAP prescribed Postgres 13 but runtime was Postgres 15.13 | Technical | Low | Low (already occurred) | Doc reconciles via `SHOW server_version`; no session/auth/email behavior depends on DB version | Resolved (documented) |
| R3 — Latent **HTTP 500** (`NoneType.sudo_mode_at`, `app/api/base.py:L47`) on pure-session call to a sudo-gated endpoint | Technical / Security | Medium | N/A (existing app behavior) | Documented as a finding; **remediation out of scope**; flag to maintainers for separate triage | Documented (out of scope) |
| R4 — App does **not rotate the session identifier on login** (session-fixation exposure) | Security | Medium | N/A (existing app behavior) | Reported with rationale + best-practice context; remediation out of scope | Documented (out of scope) |
| R5 — Session data stored via **Python `pickle`** (deserialization risk if Redis compromised) | Security | Low | Low | Reported as a finding; in-app mitigations (signed cookie pointer + server-side store) noted; change out of scope | Documented (out of scope) |
| R6 — Deliverable contains dev secret (`secret`) + seeded creds (`john@wick.com`/`password`) | Security | Low | Low | Confirmed these are **public** `example.env` / `app/fake_data.py` defaults — no production secret exposed | Mitigated |
| R7 — Byte-level evidence (UUIDs, timestamps) is environment-specific; literal values won't reproduce identically | Operational | Low | Medium | Full method + commands documented so conclusions reproduce even when literal captures differ | Mitigated |
| R8 — No CI check binds the doc's citations to source; future edits could silently invalidate line refs | Operational | Low | Low | Commit-pinned citations; recommend periodic re-verification if retained long-term | Accepted |
| R9 — Q8 New Relic telemetry inferred from code (no NR key configured), not observed on a live NR dashboard | Integration | Low | Low | Doc transparently states `LoginEvent.failed` is a NR custom event per code; runtime confirmed absence of a Python log line | Documented |
| R10 — Q5 full-header capture required a bespoke disposable SMTP sink + temp env harness | Integration | Low | Low | Harness documented step-by-step; repo `.env` never modified (temp copy used) | Mitigated |

---

## 7. Visual Project Status

**Project hours — Completed vs. Remaining** (Completed = `#5B39F3`, Remaining = `#FFFFFF`):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeWidth':'2px','pieSectionTextColor':'#111111','pieLegendTextColor':'#111111','pieTitleTextSize':'16px'}}}%%
pie showData title Project Hours Breakdown (Total 40h)
    "Completed Work" : 38
    "Remaining Work" : 2
```

**Remaining hours by priority** (from §2.2 — sums to 2.0h):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#A8FDD9','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieSectionTextColor':'#111111','pieLegendTextColor':'#111111','pieTitleTextSize':'16px'}}}%%
pie showData title Remaining Work by Priority (2.0h total)
    "High — SME review" : 1.5
    "Medium — merge & publish" : 0.5
```

> **Integrity check:** the pie "Remaining Work" (2) equals §1.2 Remaining Hours (2) and the §2.2 Hours sum (1.5 + 0.5 = 2). "Completed Work" (38) equals §1.2 Completed Hours (38) and the §2.1 total (38).

---

## 8. Summary & Recommendations

**Achievements.** The project delivered a complete, evidence-backed answer to all eight behavioral questions about SimpleLogin's authentication, session, email-forwarding, and API subsystems. Every answer pairs **observed runtime evidence** (actual status codes, raw pickle bytes, before/after `slapp` values, a real forwarded-email header set, the 600-second token boundary, live `api_key` rows, and the failed-login response/logs) with a precise **`path:line` code citation** and an explicit **rationale**. The work was independently re-validated end-to-end with **zero discrepancies**, and it honored every constraint: no source file was modified, and all ephemeral probes were removed.

**Remaining gaps & critical path to production.** The project is **95.0% complete** (38 of 40 hours). The only remaining work is the **path-to-production human gate**: an SME review/acceptance of the answers (~1.5h) followed by PR approval, merge, and publishing (~0.5h). There is **no rework** — the Final Validator confirmed the deliverable was already accurate and complete.

**Success metrics.**

| Metric | Result |
|---|---|
| Questions answered | 8 / 8 (100%) |
| Behavioral reproductions passed | 8 / 8 (100%) |
| Code citations verified | 51 / 51 (100%) |
| Source files modified (must be 0) | 0 ✅ |
| Deliverables produced (target = 1) | 1 ✅ |
| AAP-scoped completion | **95.0%** |

**Production-readiness assessment.** The deliverable is **production-ready for a documentation artifact**: complete, internally consistent (summary table matches per-question answers), accurately cited, free of placeholders, and committed on a clean tree. It can be merged upon SME sign-off.

**Recommendations.** (1) Complete the SME review and merge (the only in-scope remaining work). (2) Separately — and **out of this task's scope** — triage the security-relevant *findings* the investigation surfaced: the latent pure-session HTTP 500 (R3), the absence of session-identifier rotation on login (R4), and pickle-based session storage (R5). These are app behaviors worth a dedicated hardening ticket but were intentionally documented, not remediated, here.

---

## 9. Development Guide

This guide reproduces the exact environment the investigation used. App-run commands come from `CONTRIBUTING.md` and the deliverable's (validator-confirmed) **Environment & Method** section; assessment-side verification commands were tested while preparing this guide.

### 9.1 System Prerequisites

- **Preferred:** the prescribed container image — `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (bundles **Python 3.10**, Poetry-managed deps, **Postgres**, **Redis**).
- **Local alternative** (per `CONTRIBUTING.md`):
  - **Python 3.10** — *Python 3.12 is not supported* (`CONTRIBUTING.md:L236`).
  - **Poetry** (dependency management).
  - **PostgreSQL 13** (a 15.x server also works for this investigation).
  - **Redis** listening on **6379**.
  - **Node 10.17** for front-end assets (already built into the image).

### 9.2 Environment Setup

```bash
# From the repository root
cp example.env .env          # KEEP NOT_SEND_EMAIL=true (critical for Q5 email inspection)
```

Key values (defaults from `example.env`; `MEM_STORE_URI` enables the Redis session store):

| Key | Value | Why it matters |
|---|---|---|
| `URL` | `http://localhost:7777` | Web base URL |
| `NOT_SEND_EMAIL` | `true` | Outbound mail is not dispatched — enables local inspection (Q5) |
| `EMAIL_DOMAIN` | `sl.local` | Alias domain |
| `DB_URI` | `postgresql://myuser:mypassword@localhost:5432/simplelogin` | Dev database |
| `FLASK_SECRET` | `secret` | Base secret for the session cookie + derived secrets |
| `MEM_STORE_URI` | `redis://localhost` | Enables the server-side Redis session store |

### 9.3 Dependency Installation

```bash
# Inside the prescribed container the dependencies are prebuilt (pip check is clean).
# For a local environment:
poetry install --no-interaction --no-ansi --no-root
```

### 9.4 Application Startup

```bash
alembic upgrade head      # apply DB migrations (creates 77 tables)
flask dummy-data          # seed john@wick.com/password + API keys 'code' (Chrome) & 'codeFF' (Firefox)
python3 server.py         # web app -> 127.0.0.1:7777
python3 email_handler.py  # inbound SMTP (aiosmtpd) -> :20381   (start only when reproducing Q5)
```

### 9.5 Verification Steps

```bash
# 1) Web app health — authenticated API call with a seeded key should return HTTP 200 + JSON
curl -s -i -H "Authentication: code" http://127.0.0.1:7777/api/user_info | head -n 1
#    expected: HTTP/1.0 200 OK

# 2) No-auth contrast should return 401
curl -s -i http://127.0.0.1:7777/api/user_info | head -n 1
#    expected: HTTP/1.0 401 UNAUTHORIZED   ->   {"error":"Wrong api key"}

# 3) SMTP handler banner (when running email_handler.py)
#    expected announcement on connect: 220 ... Python SMTP 1.4.2  (port 20381)

# 4) Browser login: open http://localhost:7777 and log in with john@wick.com / password
```

**Assessment-side verification (tested while preparing this guide):**

````bash
# Confirm the single deliverable exists and is well-formed
test -f blitzy/documentation/app_2cd6ee777f8c.md && wc -l blitzy/documentation/app_2cd6ee777f8c.md   # 856
grep -cE '^## Q[0-9]' blitzy/documentation/app_2cd6ee777f8c.md                                        # 8
grep -c '```' blitzy/documentation/app_2cd6ee777f8c.md                                                # 56 (even => balanced)

# Confirm the read-only constraint: ONLY the deliverable is added vs. base
git diff --name-status origin/app_2cd6ee777f8c...HEAD
#    expected single line: A   blitzy/documentation/app_2cd6ee777f8c.md
````

### 9.6 Example Usage — Reproducing the Eight Probes

Each question's full method, commands, and captured evidence live in the deliverable. In brief:

- **Q1 / Q2 / Q7** — drive `curl` against `:7777` with/without the `Authentication: code` header (and against a sudo-gated endpoint) to observe 200 / 401 / 440 / 500 and the `api_key` field updates.
- **Q3 / Q4** — log in to obtain the `slapp` cookie, then read `session:<uuid>` from Redis (`redis-cli`) and decode it with a short `pickle`/`itsdangerous` reader; compare the `slapp` value before vs. after login.
- **Q5** — start `email_handler.py` and send a crafted message over `smtplib` to `e1@sl.local`; inspect the forwarded headers (use a disposable local SMTP sink + temp env for the full header set).
- **Q6** — sign an alias suffix and probe `check_suffix_signature` over time (real-time and/or back-dated) to find the 600s boundary.
- **Q8** — `POST /auth/login` with wrong credentials; observe the HTTP 200 re-render, the flashed message, and the access-log line.

> **Cleanup:** delete every probe script/file after capture (the investigation did). Never commit probes.

### 9.7 Troubleshooting

- **`python3 server.py` fails to import / behaves oddly on Python 3.12** → use **Python 3.10** (`CONTRIBUTING.md:L236`).
- **Sessions not persisting / Q3 key absent** → ensure **Redis** is running and `MEM_STORE_URI=redis://localhost` is set.
- **Q5 prints only a one-line summary** → that is expected under `NOT_SEND_EMAIL=true`; to see full forwarded headers, relay outbound to a disposable local sink via a **temporary** env copy (never edit the repo `.env`).
- **DB connection errors / port mismatch** → align `DB_URI` with your Postgres port (`example.env` uses `5432`; `CONTRIBUTING.md` shows a `15432`-mapped container). Postgres 15.x is acceptable for this investigation.

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---|---|
| Configure runtime | `cp example.env .env` |
| Apply migrations | `alembic upgrade head` |
| Seed fixtures | `flask dummy-data` |
| Start web app | `python3 server.py` |
| Start inbound SMTP | `python3 email_handler.py` |
| Production server (per Dockerfile) | `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` |
| Authenticated API check | `curl -s -i -H "Authentication: code" http://127.0.0.1:7777/api/user_info` |
| Read-only verification | `git diff --name-status origin/app_2cd6ee777f8c...HEAD` |

### B. Port Reference

| Service | Port | Notes |
|---|---|---|
| SimpleLogin web app | `7777` | `server.py` / `gunicorn wsgi:app` (`EXPOSE 7777`) |
| Inbound SMTP handler | `20381` | `email_handler.py` (aiosmtpd); used for Q5 |
| PostgreSQL | `5432` | `example.env` default (`CONTRIBUTING.md` maps a container to `15432`) |
| Redis | `6379` | Server-side session store (`MEM_STORE_URI`) |

### C. Key File Locations

| File | Role in the investigation |
|---|---|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The deliverable** (only persisted artifact) |
| `app/api/base.py` | Q1 session-fallback auth; Q2 `require_api_sudo` → 440; Q7 `last_used`/`times` updates |
| `app/api/views/sudo.py` | Q2 `PATCH /api/sudo` password re-check |
| `app/session.py` | Q3 pickle + `session:<uuid>` key + `slapp` signing; Q4 identifier lifecycle |
| `app/extensions.py` | Q4 Flask-Login `session_protection="strong"` |
| `app/config.py` | `SESSION_COOKIE_NAME="slapp"`; `CUSTOM_ALIAS_SECRET`; sudo window |
| `server.py` | Session-store wiring; 7-day lifetime; `dummy-data` CLI |
| `app/alias_suffix.py` | Q6 `TimestampSigner` `max_age=600` |
| `app/models.py` | Q7 `ApiKey` columns + `ModelMixin.updated_at` `onupdate` |
| `email_handler.py` | Q5 forward-phase header whitelist + From/Reply-To rewrite |
| `app/email/headers.py` | Q5 header constants + `MIME_HEADERS` set |
| `app/auth/views/login.py` | Q8 failed-login flash + event + HTTP 200 re-render |
| `app/events/auth_event.py` | Q8 `LoginEvent` → New Relic custom event |
| `app/fake_data.py` | Seed fixtures (`john@wick.com`, API keys) |

### D. Technology Versions

| Component | Version | Source |
|---|---|---|
| Python | 3.10 (runtime 3.10.18) | `pyproject.toml` (`python ^3.10`), Dockerfile |
| Flask | 1.1.2 | `poetry.lock` |
| Flask-Login | 0.5.0 | `poetry.lock` |
| itsdangerous | 1.1.0 | `poetry.lock` |
| Werkzeug | 1.0.1 | `poetry.lock` |
| redis (client) | 4.6.0 | `poetry.lock` |
| SQLAlchemy | 1.3.24 | `poetry.lock` |
| aiosmtpd | 1.4.2 | `poetry.lock` (SMTP banner) |
| gunicorn | 20.0.4 | `poetry.lock` / Dockerfile |
| PostgreSQL | 15.13 at runtime (AAP nominal 13) | `SHOW server_version` |
| Redis (server) | 7.0.15 | runtime |

### E. Environment Variable Reference

| Variable | Value | Purpose |
|---|---|---|
| `URL` | `http://localhost:7777` | Web base URL |
| `NOT_SEND_EMAIL` | `true` (presence-based; `app/config.py:L91`) | Suppresses outbound dispatch; enables local email inspection (Q5) |
| `EMAIL_DOMAIN` | `sl.local` | Alias domain (resolves `FIRST_ALIAS_DOMAIN`) |
| `DB_URI` | `postgresql://myuser:mypassword@localhost:5432/simplelogin` | Dev database connection |
| `FLASK_SECRET` | `secret` | Base secret → `slapp` cookie signing + derived secrets (e.g., `CUSTOM_ALIAS_SECRET`) |
| `MEM_STORE_URI` | `redis://localhost` | Enables the Redis server-side session store |
| `CONFIG` | path to env file | Used for Q5's temporary env copy (`/tmp/q5.env`) — repo `.env` never modified |

### F. Developer Tools Guide

- **`curl`** — HTTP probes for Q1/Q2/Q7 (status codes, JSON bodies, header presence).
- **`redis-cli` + a short `pickle`/`itsdangerous` reader** — Q3/Q4 session bytes, key listing, `slapp` unsign.
- **`smtplib` client + a disposable local SMTP sink** — Q5 real inbound email and full forwarded-header capture.
- **A token-timing prober** — Q6 (real-time and back-dated boundary sweeps).
- **`psql`** — Q7 `api_key` row inspection (`SELECT *`).
- All of the above are **ephemeral** observation tools — created to capture evidence, then deleted; never committed.

### G. Glossary

| Term | Meaning |
|---|---|
| **AAP** | Agent Action Plan — the authoritative requirement set for this task |
| **`slapp`** | SimpleLogin's session cookie name (`app/config.py:L199`); carries only the signed session identifier |
| **HTTP 440** | Non-standard status (Microsoft IIS "Login Time-out"); SimpleLogin repurposes it to mean "sudo mode required" |
| **sudo mode** | A re-authenticated state (valid 5 min) required for privileged API operations; entered via `PATCH /api/sudo` |
| **`require_api_sudo`** | Decorator returning `440 {"error":"Need sudo"}` when sudo mode is inactive |
| **Reverse-alias** | A SimpleLogin-generated address that masks the real sender; the forward path rewrites `From`/`Reply-To` to it |
| **pickle (proto 4)** | Python serialization format used for session payloads (raw bytes begin `\x80\x04\x95`) — *not* PHP serialization |
| **`onupdate`** | SQLAlchemy column hook (`ModelMixin.updated_at`) that auto-stamps `updated_at` on every row UPDATE (Q7's third field) |
| **Ephemeral probe** | A disposable script used only to capture runtime evidence, deleted after use |
