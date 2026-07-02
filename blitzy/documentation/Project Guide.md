# Blitzy Project Guide — SimpleLogin First-Operator Runtime Verification Guide

> Branch: `blitzy-11c9c5fc-3d71-4f22-abc3-d2fb0eb5d4e5` · Base: `origin/app_2cd6ee777f8c` (`2cd6ee77`) · HEAD: `5dbaa66f`
> Brand palette — Completed/AI: **Dark Blue `#5B39F3`** · Remaining: **White `#FFFFFF`** · Headings/Accents: **Violet-Black `#B23AF2`** · Highlight: **Mint `#A8FDD9`**

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a single, evidence-grounded documentation artifact — a **first-operator runtime-verification guide** for **SimpleLogin**, an open-source Flask email alias/forwarding service. The guide answers three runtime-observability questions: what startup signals confirm the app is ready for **authentication** and **alias-based email** (Q1); what a new user sees when **registering, verifying, and logging in** through to the dashboard (Q2); and what **background jobs, email-forwarding internals, identity-verification services, and inter-process signals** reveal at runtime (Q3). Following a strict run-first, one-claim-one-evidence discipline, every behavioral claim is paired with verbatim captured output and an exact `file:line` citation. Target users are operators and engineers bringing SimpleLogin up locally. The source repository is treated strictly read-only; exactly one markdown file is added.

### 1.2 Completion Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeWidth':'2px','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieLegendTextColor':'#111111'}}}%%
pie showData title Completion Status — 90.0% Complete
    "Completed Work (AI)" : 27
    "Remaining Work" : 3
```

| Metric | Value |
|--------|-------|
| **Total Hours** | **30** |
| **Completed Hours (AI + Manual)** | **27** (27 AI + 0 Manual) |
| **Remaining Hours** | **3** |
| **Percent Complete** | **90.0%** |

> Completion is computed per the AAP-scoped hours methodology: `Completed ÷ (Completed + Remaining) = 27 ÷ 30 = 90.0%`. The remaining 10% is standard path-to-production human review for a documentation deliverable (Section 2.2).

### 1.3 Key Accomplishments

- ✅ Authored the sole AAP deliverable `blitzy/documentation/app_2cd6ee777f8c.md` (725 lines) at the exact mandated path/name.
- ✅ Answered **all 11 named sub-parts** — Q1(a–b), Q2(a–e), Q3(a–d) — plus a run-setup recap, coverage-pass table, and honest-caveats section.
- ✅ Grounded the guide in a **real multi-process run** (webapp `:7777`, SMTP handler `:20381`, job runner) with captured PIDs, ports, and verbatim `SL` log lines.
- ✅ Applied **one-claim-one-evidence** discipline across **168 `file:line` citations spanning 29 distinct source files**.
- ✅ Reported local-environment nuances **honestly** (suppressed Werkzeug banner, `NOT_SEND_EMAIL`, `DISABLE_ONBOARDING`, New Relic vs. Postgres events, DMARC/SpamAssassin disabled).
- ✅ Corrected the one known discrepancy — `/auth/login` `Content-Length` — with a self-consistent same-request capture and a cited debug-toolbar variance note (`server.py:L588`).
- ✅ Preserved **strict read-only integrity**: cumulative diff vs base is exactly 1 file, `+725/-0`; **zero** source files touched; temp scripts and probe data removed; DB restored to seed baseline.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| _None._ No blocking issues remain. The single AAP deliverable is complete, evidence-grounded, and committed; the application builds, runs, and passes all environment-runnable tests. | None | — | — |

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-----------------|----------------|-------------------|-------------------|-------|
| Apple App Store API (`tests/api/test_apple.py`) | Outbound internet / Apple credentials | The autonomous sandbox is air-gapped, so the external Apple-payment test cannot reach Apple's API and times out. Not a regression and out of scope for this docs-only task. | Environmental — resolves automatically in an internet-enabled CI runner | Human reviewer / CI |

> No repository-permission, service-credential, or deployment access issues affect the deliverable itself.

### 1.6 Recommended Next Steps

1. **[High]** SME technical review of `blitzy/documentation/app_2cd6ee777f8c.md` for accuracy, completeness against every named sub-part, and first-operator usability.
2. **[Medium]** Independently bring up the stack and spot-verify a representative sample of the captured evidence (`/health`, `/auth/login` form, SMTP `220` banner, a register→activate→login walkthrough, a near-term `Take job` dispatch).
3. **[Medium]** Review and merge the PR (single file, `+725/-0`), confirming read-only integrity.
4. **[Low]** Optionally schedule periodic re-verification of observed values/citation line-numbers as SimpleLogin evolves (out of current AAP scope).

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|-------|-------------|
| Runtime foundation & environment bring-up | 4 | Provision PostgreSQL 15.13 + Redis 7.0.15; `.env` from `example.env`; `poetry install` (174 locked deps); `alembic upgrade head` (rev `32f25cbf12f6`, 77 tables); `flask dummy-data` (2 users, 11 aliases); launch `server.py`/`email_handler.py`/`job_runner.py`; verify liveness by PID/port. |
| Q1 readiness investigation & evidence capture | 3 | `>>> init logging <<<`; `/health` → `200 "success"` (CL 7); `/auth/login` CSRF `email`/`password` form; `/` → `302` to `/auth/login`; SMTP bind logs + live `EHLO` handshake (`220 … Python SMTP 1.4.2` + `SIZE`/`8BITMIME`/`SMTPUTF8`/`HELP`); job-runner poll; alias-domain query. |
| Q2 new-user journey walkthrough | 4 | Register + waiting screen; password-policy negative (`Length(min=8)`); activation-code DB read (len 30); `GET /auth/activate` → `302 /dashboard/` + `activated` false→true; welcome email; success flash; login success + wrong-password + not-activated paths; dashboard render (`Alias | SimpleLogin`) + 11 alias occurrences. |
| Q3 behind-the-scenes investigation | 7 | `DISABLE_ONBOARDING` idle proof; job selection-window computation; near-term job enqueue + live `Take job` dispatch + full lifecycle `0→2`; email-forward pipeline correlated under one `message_id` (handle → forward phase → contact → DMARC → routing → `EmailLog` → reverse-alias → send); disabled-alias negative; DMARC/SpamAssassin/DKIM/SPF/PGP each by name; mailbox verification live (code len 22); custom-domain DNS validation live; `NOTIFY/LISTEN` `simplelogin_sync_events` probe; 3-PID cross-process evidence; cron 15-job summary. |
| Answer document authoring | 5 | 725-line markdown; one-claim-one-evidence discipline; coverage-pass checklist; honest caveats; run-setup recap. |
| Citation audit & markdown validation | 2 | 168 citations verified against source; 29 distinct files; balanced fences (116). |
| Validation fix — `/auth/login` Content-Length correction | 1 | Debug-toolbar variance investigation (`server.py:L588`); self-consistent same-request capture (`235552`); cited honesty note; verified stale value now 0 occurrences. |
| Cleanup & repository restoration | 1 | Remove temp scripts (`walk_q2_dashboard.py`, `job_window.py`, `enqueue_job.py`); delete probe users/aliases; restore DB + `daily_metric` to seed baseline. |
| **Total** | **27** | **Sum of completed AAP-scoped work (matches Section 1.2 Completed Hours).** |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|-------|----------|
| SME technical review of the answer document (accuracy, completeness vs. named sub-parts, first-operator usability, citation spot-check) | 1.5 | High |
| Independent runtime spot-verification of sampled evidence (bring up stack, reproduce a representative sample) | 1.0 | Medium |
| PR review & merge (confirm read-only integrity; incorporate minor feedback) | 0.5 | Medium |
| **Total** | **3.0** | **Matches Section 1.2 Remaining Hours and Section 7 pie "Remaining Work".** |

> **Out-of-scope / optional (0h, excluded from the remaining total):** periodic re-verification of observed values as SimpleLogin evolves; an optional "production vs. local" delta appendix. These lie beyond the fully delivered AAP scope and are listed for awareness only.

### 2.3 Total Project Hours & Methodology

| Roll-up | Hours |
|---------|-------|
| Section 2.1 — Completed | 27 |
| Section 2.2 — Remaining | 3 |
| **Total Project Hours** | **30** |
| **Percent Complete** | **90.0%** |

Scope is defined exclusively by the AAP plus standard path-to-production activities. All 19 AAP-specified requirements are complete (autonomous work 100% done); the 3 remaining hours are the mandatory human sign-off before a documentation artifact is accepted as authoritative. Formula: `27 ÷ (27 + 3) = 90.0%`.

---

## 3. Test Results

All figures below originate from **Blitzy's autonomous validation run** using the project's documented command: `CONFIG=tests/test.env GITHUB_ACTIONS_TEST=true pytest -c pytest.ci.ini --timeout=60 --timeout-method=signal`.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|------------|-------|
| Full regression suite (unit + integration across `tests/`: `api`, `auth`, `cron`, `dashboard`, `email_tests`, `events`, `handler`, `jobs`, `models`, `monitor`, `oauth`, `proton`, `tasks`, `user_settings`) | pytest | 639 | 638 | 1 | 71.75% | Executed in `241.87s` with 53 warnings. Coverage gate is `fail_under = 55` (`coverage.ini`) — **exceeded** at 71.75%. |
| — of which: external Apple-payment test | pytest | 1 | 0 | 1 | — | `tests/api/test_apple.py::test_apple_process_payment` — **external Apple-API network dependency**, times out in the air-gapped sandbox. Not a regression; this docs-only task modified **0 test files**; the setup baseline recorded the identical result. Resolves in an internet-enabled runner. |

> **Integrity note:** This is a documentation-only task — no tests were authored or modified. The suite above is the regression run Blitzy's autonomous validation executed to confirm the application still builds and runs (which underpins the guide's live-evidence claims). The autonomous logs reported the aggregate figures shown; per-category numeric splits were not separately emitted and are therefore not fabricated here. Effective pass rate on environment-runnable tests: **638/638 = 100%**.

---

## 4. Runtime Validation & UI Verification

Captured by Blitzy's autonomous validation in the `sl-dev` container; every documented behavior reproduced live.

**Process & infrastructure health**
- ✅ **Webapp** — `python server.py` listening on `:7777`.
- ✅ **Email handler** — `python email_handler.py` (aiosmtpd) listening on `:20381`.
- ✅ **Job runner** — `python job_runner.py` polling every 10s.
- ✅ **PostgreSQL** — migration at head `32f25cbf12f6`; seed data present (2 users, 11 aliases).
- ✅ **Redis** — `7.0.15` reachable on `:6379`.

**Q1 — Readiness**
- ✅ `GET /health` → `HTTP 200`, body `success`, `Content-Length: 7`.
- ✅ `GET /` → `HTTP 302`, `Location: /auth/login`, `Content-Length: 229`.
- ✅ `GET /auth/login` → `HTTP 200` with CSRF-protected `email`/`password` form.
- ✅ SMTP `EHLO` → `220 … Python SMTP 1.4.2` plus advertised `SIZE`/`8BITMIME`/`SMTPUTF8`/`HELP`.

**Q2 — New-user journey**
- ✅ Register → title `Activation Email Sent | SimpleLogin`; `create user …` logged.
- ✅ Password policy → short password rejected (`Field must be between 8 and 100 characters long`); no row created.
- ✅ Verify → activation code length 30; `GET /auth/activate` → `302 /dashboard/`; `activated` false→true; success flash `Your account has been activated`.
- ✅ Login (success) → `302 /dashboard/`; `log user … in`.
- ⚠ Login (negative, expected) → wrong password `Email or password incorrect`; not-activated `Please check your inbox for the activation email…`. Marked ⚠ only because they are deliberate negative-path checks, both behaving correctly.
- ✅ Dashboard → title `Alias | SimpleLogin`; first alias `simplelogin-newsletter.hereon338@sl.local`.

**Q3 — Behind the scenes**
- ✅ `DISABLE_ONBOARDING=true` → zero onboarding jobs on signup (honest idle behavior).
- ✅ Near-term job → job runner `Take job …` within 10s; state `0→2`, `taken` f→t, `attempts` 0→1.
- ✅ Email forward → full pipeline correlated under one `message_id`; disabled alias `e0@sl.local` → `is disabled, do not forward`.
- ✅ Mailbox verification → code length 22 (`token_urlsafe(16)`), `verified` false→true.
- ✅ Custom-domain validation → expected ownership TXT / MX / SPF / DKIM records produced.
- ✅ Inter-process → `LISTEN simplelogin_sync_events` receives `NOTIFY` only after dispatcher `send()` + commit; 5 concurrent PostgreSQL backends observed across the processes.
- ✅ Scheduler → `crontab.yml` defines 15 jobs.

**UI verification artifacts:** 44 screenshots (`blitzy/screenshots/`) and 1 journey screencast (`blitzy/screen_recordings/final_acceptance_q2_journey.webm`) capture the register→activation→dashboard flow and negative paths. These are retained for review and intentionally not committed (per the AAP "single markdown file" constraint).

---

## 5. Compliance & Quality Review

Cross-map of the governing rule set **"SWE-AtlasQnA-Repo"** to observed outcomes.

| Compliance Item (AAP directive) | Benchmark | Status | Progress | Notes |
|---------------------------------|-----------|--------|----------|-------|
| Output location & naming | `blitzy/documentation/app_2cd6ee777f8c.md` | ✅ Pass | 100% | Correct path/name; committed. |
| Run-first methodology | Evidence from a real run | ✅ Pass | 100% | 3 processes live; PIDs/ports/verbatim logs captured. |
| One-claim-one-evidence | Evidence beside every claim | ✅ Pass | 100% | 168 evidence-backed claims; no batching. |
| Exact `file:line` citations | Cite literals with `file:line` | ✅ Pass | 100% | 168 citations / 29 files; spot-checks 100% accurate. |
| Answer every named item | Coverage pass | ✅ Pass | 100% | 37/37 named items present; coverage-pass table included. |
| Honest reporting of nuances | Report observed, even if unexpected | ✅ Pass | 100% | 5 caveats (banner, `NOT_SEND_EMAIL`, `DISABLE_ONBOARDING`, New Relic, DMARC/SpamAssassin). |
| Read-only source | No source file modified | ✅ Pass | 100% | 0 source files changed; diff = 1 file `+725/-0`. |
| Cleanup temp artifacts | Remove scripts/test data | ✅ Pass | 100% | Temp scripts absent; DB restored to seed. |
| Markdown quality | Well-formed | ✅ Pass | 100% | 116 balanced fences; 19 headings; tables render. |
| Value accuracy | No stale/paraphrased values | ✅ Pass | 100% | `Content-Length` corrected (`369826`→`235552`) + cited variance note. |

**Fixes applied during autonomous validation:** the `/auth/login` `Content-Length` capture was replaced with a self-consistent same-request value and an honest debug-toolbar variance note (`server.py:L588`). **Outstanding compliance items:** none.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Guide reflects local-dev behavior (`NOT_SEND_EMAIL`, `DISABLE_ONBOARDING`, DMARC/SpamAssassin off, debug-toolbar page size) that differs from production | Technical | Low | Medium | Explicit "Honest caveats" section + inline nuance notes describe each divergence | ✅ Mitigated |
| Citation drift — 168 `file:line` refs pinned to `app_2cd6ee777f8c` (`2cd6ee77`) may drift if source advances | Technical | Low | Medium | Guide states the exact branch + HEAD it was written against | ✅ Mitigated |
| `/auth/login` `Content-Length` varies per request (debug toolbar) | Technical | Low | Low | Self-consistent capture + variance note citing `server.py:L588` | ✅ Resolved |
| No security exposure — read-only docs task; only public seeded demo creds referenced | Security | None | N/A | No source/config/dependency changes; no real secrets in the guide | ✅ N/A |
| External Apple-API test times out without internet | Operational | Low (info) | N/A (environmental) | Documented as environmental/out-of-scope; 0 test files modified; resolves in internet-enabled CI | ✅ Accepted |
| Documentation may go stale as SimpleLogin evolves | Operational | Low | Medium (over time) | Optional periodic re-verification; guide is branch-pinned | ⚠ Open (future) |
| No integration surface — no code/interface/config/schema/dependency changes | Integration | None | N/A | Runtime components referenced strictly read-only | ✅ N/A |

Overall risk posture: **Low** — consistent with a validated, read-only documentation deliverable.

---

## 7. Visual Project Status

**Project hours breakdown** (Completed = Dark Blue `#5B39F3`, Remaining = White `#FFFFFF`):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeWidth':'2px','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieLegendTextColor':'#111111'}}}%%
pie showData title Project Hours Breakdown
    "Completed Work" : 27
    "Remaining Work" : 3
```

**Remaining hours by category** (from Section 2.2, total = 3h):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#B23AF2','pie2':'#5B39F3','pie3':'#A8FDD9','pieTitleTextColor':'#B23AF2','pieLegendTextColor':'#111111'}}}%%
pie showData title Remaining Work by Category (3h)
    "SME technical review [High]" : 1.5
    "Runtime spot-verification [Medium]" : 1.0
    "PR review & merge [Medium]" : 0.5
```

> **Integrity check:** "Remaining Work" = **3h**, identical to Section 1.2 Remaining Hours and the Section 2.2 total. "Completed Work" = **27h**, identical to Section 1.2 Completed Hours and the Section 2.1 total.

---

## 8. Summary & Recommendations

**Achievements.** The project delivers exactly what the AAP scoped: a single, comprehensive, evidence-grounded runtime-verification guide for SimpleLogin. It answers all three question groups and every named sub-part (Q1 a–b, Q2 a–e, Q3 a–d), grounded in a real multi-process run with 168 verbatim-evidence-backed claims citing 29 source files. The read-only constraint is honored perfectly (diff = 1 file, `+725/-0`), and the one known discrepancy was corrected during validation.

**Remaining gaps.** No AAP-scoped autonomous work remains. The outstanding 3 hours are standard path-to-production human activities: SME technical review, independent runtime spot-verification, and PR merge.

**Critical path to production.** SME review (High) → independent spot-verification (Medium) → PR merge (Medium). None are blocking; each is a confidence/acceptance gate.

**Success metrics.** All 19 AAP-specified requirements complete; 37/37 named items covered; 168 citations accurate on spot-check; 638/638 environment-runnable tests passing; 71.75% coverage vs. a 55% gate.

**Production-readiness assessment.** The deliverable is **90.0% complete** on an AAP-scoped basis and is production-ready pending human sign-off. Confidence is **High**: the guide is complete, internally consistent, and independently reproducible via the documented run mechanism.

| Metric | Result |
|--------|--------|
| AAP-scoped completion | 90.0% (27h / 30h) |
| AAP requirements complete | 19 / 19 |
| Named items covered | 37 / 37 |
| Environment-runnable tests | 638 / 638 passing |
| Source files modified | 0 (read-only honored) |
| Overall risk | Low |

---

## 9. Development Guide

How to build, run, and reproduce the guide's observations. Commands come from `CONTRIBUTING.md §"Run the code locally"` and `example.env`. Stack-dependent commands were validated in Blitzy's `sl-dev` container; where a command was directly verifiable in this assessment environment it is noted.

### 9.1 System Prerequisites
- **Python 3.10** (`pyproject.toml`: `python = "^3.10"`). *(Do not use 3.13 — not the project baseline.)*
- **PostgreSQL 13+** (development baseline; the validated run used 15.13).
- **Redis 7.x** (rate limiting, sessions).
- **Poetry** (dependency management), **Docker** (to run the database), **Node.js + npm** (static assets).

### 9.2 Environment Setup
```bash
# 1. Static assets
cd static && npm install && cd ..

# 2. Local settings from the template
cp example.env .env

# 3. Edit .env so DB_URI matches the exposed Postgres port (see step 4).
#    Default in example.env is :5432 (L75); align it with your host mapping:
#    DB_URI=postgresql://myuser:mypassword@localhost:15432/simplelogin
```
Key `.env` variables (inherited from `example.env`): `URL=http://localhost:7777` (L6), `NOT_SEND_EMAIL=true` (L19), `EMAIL_DOMAIN=sl.local` (L22), `FLASK_SECRET=secret` (L77), `DISABLE_ONBOARDING=true` (L150).

### 9.3 Start Backing Services
```bash
# PostgreSQL (host port 15432 -> container 5432)
docker run -d --name sl-postgres \
  -e POSTGRES_PASSWORD=mypassword -e POSTGRES_USER=myuser -e POSTGRES_DB=simplelogin \
  -p 15432:5432 postgres:13

# Redis
docker run -d --name sl-redis -p 6379:6379 redis:7
```

### 9.4 Dependency Installation
```bash
poetry install          # installs the 174 locked dependencies into a py3.10 virtualenv
poetry shell            # activate the environment (or prefix commands with `poetry run`)
```

### 9.5 Application Startup
```bash
# Apply the schema, seed demo data, and launch the webapp
alembic upgrade head && flask dummy-data && python3 server.py     # webapp on :7777

# In separate shells (for Q1 email readiness and Q3 background jobs):
python email_handler.py   # SMTP handler on :20381
python job_runner.py      # background job poller (10s loop)
```
Then open `http://localhost:7777` and log in with **`john@wick.com` / `password`**.

### 9.6 Verification Steps
```bash
# Webapp liveness — expect: HTTP 200, body "success", Content-Length: 7
curl -sS -i http://localhost:7777/health

# Auth gating — expect: HTTP 302, Location: http://localhost:7777/auth/login
curl -sS -i http://localhost:7777/

# Auth UI — expect the three field names to print
curl -sS http://localhost:7777/auth/login | grep -oE 'name="(csrf_token|email|password)"'

# SMTP handshake — expect: 220 <host> Python SMTP 1.4.2  then 250-... capabilities
printf 'EHLO probe.local\r\nQUIT\r\n' | nc 127.0.0.1 20381

# Migration head — expect: 32f25cbf12f6 (head)
alembic current
```

### 9.7 Example Usage (reproduce the guide's evidence)
- **Q2 walkthrough:** `POST /auth/register` → see the "Activation Email Sent" screen and a `create user …` log line; read the 30-char code from the `activation_code` table (because `NOT_SEND_EMAIL` suppresses the printed link); `GET /auth/activate?code=<code>` → `302 /dashboard/` and `activated` flips to true; `POST /auth/login` with the seeded account → `302 /dashboard/`.
- **Q3 job dispatch:** enqueue a near-term `Job` (`run_at = now`); within 10s the job runner logs `Take job …` and the row transitions `state 0→2`.

### 9.8 Troubleshooting
- **No "Running on http://…" banner** — *expected.* The Werkzeug access logger is disabled (`app/log.py:L70-71`). Use `/health` for liveness instead.
- **Activation link not printed** — *expected.* `NOT_SEND_EMAIL=true` logs subject/from/to only (`app/mail_sender.py:L130-137`). Read the `activation_code` table, or point `POSTFIX_SERVER`/`POSTFIX_PORT` at a mail sink (e.g., MailHog `1025`/`1080`).
- **Job runner idle after signup** — *expected.* `DISABLE_ONBOARDING=true` enqueues zero jobs, and onboarding jobs schedule `+1/2/3 days`; the selection window only picks jobs due within ~10 minutes. Enqueue a near-term job to see a dispatch.
- **DB connection refused** — ensure `DB_URI`'s port matches the Docker `-p HOST:5432` mapping.
- **Python version mismatch** — use Python 3.10 via the Poetry environment.
- **`Field must be between 8 and 100 characters long`** on register — the password policy enforces a minimum length of 8 (`app/auth/views/register.py:L27`).

---

## 10. Appendices

### A. Command Reference
| Purpose | Command |
|---------|---------|
| Copy env template | `cp example.env .env` |
| Start Postgres | `docker run -d --name sl-postgres -e POSTGRES_PASSWORD=mypassword -e POSTGRES_USER=myuser -e POSTGRES_DB=simplelogin -p 15432:5432 postgres:13` |
| Start Redis | `docker run -d --name sl-redis -p 6379:6379 redis:7` |
| Install deps | `poetry install` |
| Migrate + seed + run webapp | `alembic upgrade head && flask dummy-data && python3 server.py` |
| Run SMTP handler | `python email_handler.py` |
| Run job runner | `python job_runner.py` |
| Health check | `curl -sS -i http://localhost:7777/health` |
| Run test suite | `CONFIG=tests/test.env GITHUB_ACTIONS_TEST=true pytest -c pytest.ci.ini --timeout=60 --timeout-method=signal` |

### B. Port Reference
| Port | Service |
|------|---------|
| 7777 | Webapp (`server.py`) |
| 20381 | SMTP email handler (`email_handler.py`, aiosmtpd) |
| 6379 | Redis |
| 15432 → 5432 | PostgreSQL (host → container mapping) |
| 1025 / 1080 | Optional local mail sink (MailHog) SMTP / UI |

### C. Key File Locations
| Path | Role |
|------|------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The deliverable** — the runtime-verification guide |
| `server.py` | Webapp app-factory, `/health`, dev server (`:7777`) |
| `email_handler.py` | SMTP handler / forwarding router (`:20381`) |
| `job_runner.py` | Background job poller |
| `cron.py` / `crontab.yml` | Scheduler + 15 recurring jobs |
| `app/log.py` | `SL` logger config; suppressed Werkzeug banner |
| `app/models.py` | ORM models; `User.create`, onboarding jobs |
| `app/config.py` | Environment-driven config keys |
| `app/mail_sender.py` | `NOT_SEND_EMAIL` behavior |
| `app/auth/views/{register,activate,login,login_utils}.py` | Q2 journey |
| `app/mailbox_utils.py` / `app/custom_domain_validation.py` | Q3 identity verification |
| `app/events/` + `events/` | Event dispatch / `NOTIFY-LISTEN` runner |

### D. Technology Versions
| Component | Version |
|-----------|---------|
| Python | 3.10 (baseline; validated on 3.10.18) |
| PostgreSQL | 13+ (validated 15.13) |
| Redis | 7.0.15 |
| Flask | ^1.1.2 |
| SQLAlchemy | 1.3.24 |
| aiosmtpd | ^1.2 (resolved 1.4.2) |
| redis (client) | ^4.5.3 |
| Alembic head revision | `32f25cbf12f6` |

### E. Environment Variable Reference
| Variable | Value (local) | Effect |
|----------|---------------|--------|
| `URL` | `http://localhost:7777` | Base URL used to build the activation link |
| `DB_URI` | `postgresql://myuser:mypassword@localhost:15432/simplelogin` | PostgreSQL connection (must match exposed port) |
| `NOT_SEND_EMAIL` | `true` | Logs mail (subject/from/to) instead of sending |
| `DISABLE_ONBOARDING` | `true` | Skips enqueuing onboarding jobs on registration |
| `EMAIL_DOMAIN` | `sl.local` | Domain for generated aliases |
| `FLASK_SECRET` | `secret` | Flask session signing key |

### F. Developer Tools Guide
- **pytest** — run the suite with the documented CI command (Appendix A); coverage HTML is written to `htmlcov/` (gitignored). Coverage gate: `fail_under = 55` (`coverage.ini`).
- **alembic** — `alembic current` (show head), `alembic upgrade head` (apply migrations).
- **flask dummy-data** — CLI seed command (`server.py`), creates demo users/aliases.
- **Docker** — run Postgres/Redis containers; `docker logs sl-postgres` for DB diagnostics.
- **curl / nc** — reproduce HTTP and SMTP evidence (Appendix A).

### G. Glossary
| Term | Meaning |
|------|---------|
| Alias | A generated forwarding address (e.g., `…@sl.local`) that relays to a user's real mailbox |
| Reverse-alias | Rewritten `From` address so replies route back through SimpleLogin |
| Activation code | 30-char random token emailed at registration to verify a user's address |
| Job runner | Background process that polls and executes queued `Job` rows every 10s |
| `NOTIFY/LISTEN` | PostgreSQL pub/sub channel (`simplelogin_sync_events`) used for inter-process event delivery |
| `NOT_SEND_EMAIL` | Local flag that logs outbound mail instead of sending it |
| `DISABLE_ONBOARDING` | Local flag that suppresses onboarding-job enqueue at registration |
| First operator | The intended reader — someone bringing SimpleLogin up and verifying health |