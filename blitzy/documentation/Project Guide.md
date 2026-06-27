# Blitzy Project Guide — SimpleLogin First‑Run Behavior: Code‑Grounded Verification

> **Color legend (Blitzy brand):** **Completed / AI Work** = Dark Blue `#5B39F3` · **Remaining / Not Completed** = White `#FFFFFF` · **Headings / Accents** = Violet‑Black `#B23AF2` · **Highlight** = Mint `#A8FDD9`.

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a single, **code‑grounded verification document** (`blitzy/documentation/app_2cd6ee777f8c.md`) that confirms and explains how the **SimpleLogin** email‑aliasing application (a full‑stack Python/Flask monorepo) behaves the first time it is built and run locally. It answers three runtime‑behavior questions — readiness signals (Q1), the new‑user register→verify→login→dashboard journey (Q2), and the behind‑the‑scenes background services that support email forwarding and identity verification (Q3) — with every claim traced to `file:line` citations and corroborated by live‑run log/HTTP evidence. The target audience is engineers onboarding to SimpleLogin. Critically, this is a **documentation deliverable**: the SimpleLogin source tree is strictly read‑only and remains byte‑for‑byte unchanged.

### 1.2 Completion Status

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieOuterStrokeWidth':'2px','pieTitleTextSize':'15px','pieSectionTextSize':'13px','pieLegendTextSize':'13px'}}}%%
pie showData title Completion — 85.7% Complete
    "Completed Work (AI)" : 30
    "Remaining Work (Human Review)" : 5
```

| Metric | Value |
|--------|-------|
| **Total Hours** | **35.0** |
| **Completed Hours (AI + Manual)** | **30.0** (AI 30.0 + Manual 0.0) |
| **Remaining Hours** | **5.0** |
| **Percent Complete** | **85.7%** |

> Completion is computed per the AAP‑scoped, hours‑based methodology: `Completed ÷ (Completed + Remaining) = 30.0 ÷ 35.0 = 85.7%`. The deliverable itself is fully authored, autonomously validated, and committed; the remaining 5.0 hours are **path‑to‑production human review/acceptance/merge** with no known defects.

### 1.3 Key Accomplishments

- ✅ **Single deliverable authored & committed** — `blitzy/documentation/app_2cd6ee777f8c.md` (456 lines, ~52 KB), at HEAD `46e0e9ac` by `Blitzy Agent <agent@blitzy.com>`.
- ✅ **All three questions answered with rationale** — Q1 readiness, Q2 new‑user journey, Q3 behind‑the‑scenes, each with evidence‑first explanations (20 explicit "Rationale" passages).
- ✅ **Code‑grounded** — 118+ `file:line` citations across ~30 source files; a full reference index grouped by section.
- ✅ **Live‑run evidence** — 9 "Observed" log/HTTP excerpt blocks captured from a running instance (web :7777, mail :20381, job runner loop).
- ✅ **Read‑only constraint honored** — `git diff 2cd6ee77..HEAD` = exactly one added file; zero source files modified.
- ✅ **Temporary test data removed** — register/forwarding/job artifacts torn down via the project's own ORM; net database impact = zero (seeded baseline restored).
- ✅ **Autonomously validated** — Final Validator passed all five production‑readiness gates; citation accuracy and live observation reproduction both 100%.
- ✅ **Independently corroborated** — 11/11 spot‑checked citations matched source exactly during this assessment.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| _None._ No defects, citation errors, or doc↔runtime mismatches remain. The deliverable passed autonomous validation with zero required edits. | No release‑blocking impact. | — | — |

> The only remaining work is standard human review/acceptance (see §1.6 and §2.2). There are **no critical unresolved issues**.

### 1.5 Access Issues

| System / Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-------------------|----------------|-------------------|-------------------|-------|
| _No access issues identified._ | — | The deliverable was authored and validated inside the provided prebuilt container with the source baked in; no external credentials, repository permissions, or third‑party API access were required. | N/A | — |

### 1.6 Recommended Next Steps

1. **[High]** Perform an SME technical review of the document — read it end‑to‑end and sample‑verify a subset of the 118+ citations against the source at commit `app_2cd6ee777f8c`.
2. **[Medium]** Reproduce the build/run recipe in your own environment (§9) to confirm portability; mind the `DB_URI` port reconciliation and the Flask dev‑server loopback‑binding note.
3. **[Medium]** Approve the pull request and merge/publish `blitzy/documentation/app_2cd6ee777f8c.md`.
4. **[Low]** Decide the disposition of the untracked `blitzy/screenshots/` directory (55 PNGs, ~11 MB) — confirm intentional exclusion (the committed doc embeds zero images) or selectively attach.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|------:|-------------|
| Environment setup & live stack bring‑up | 3.0 | Build/run the full stack to gather evidence: Poetry deps, Node asset build, PostgreSQL 13, `alembic upgrade head`, `flask dummy-data` seed, and booting webapp + `email_handler.py` + `job_runner.py`. |
| Doc §(a) — Reproducible build/run recipe | 2.5 | Architecture overview, runtime‑targets table, canonical `CONTRIBUTING.md` steps, env‑value table, and the investigated `DB_URI` port + Docker host‑browser/`URL` origin nuances. |
| Q1 — Readiness signals (analysis + live capture + authoring) | 4.0 | Five deterministic web‑tier markers + two mail‑tier markers, Werkzeug‑banner suppression caveat, with live log excerpts and citations to `app/log.py`, `server.py`, `email_handler.py`. |
| Q2 — New‑user journey (analysis + live walkthrough + authoring) | 5.0 | register→activate→login→dashboard with auto‑login proof (fresh cookie jar), seed‑account direct login, `NOT_SEND_EMAIL` activation‑code nuance, three UI‑quality observations, HTTP/log evidence. |
| Q3 — Behind‑the‑scenes services (analysis + live capture + authoring) | 5.0 | `job_runner` loop, `email_handler` forwarding + `message_id` correlation, `cron`/yacron schedule, event subsystems (`NOTIFY`/`LISTEN`), optional Redis, Mermaid coordination diagram, and the `DISABLE_ONBOARDING`⇒0‑jobs correction. |
| Code‑citation grounding, rationale & references index | 3.0 | 118+ `file:line` citations verified, evidence‑first rationale throughout, and the grouped reference index. |
| Temporary test‑data creation, ORM cleanup & DB verification | 1.5 | Created throwaway user/alias/contact/probe‑job to exercise Q2/Q3, then deleted all via the project ORM with post‑cleanup `psql` verification (net DB impact zero). |
| QA refinement cycles (3 follow‑up commits) | 3.0 | Addressed code‑review findings, corrected the Q3 `job_runner` self‑enqueue claim, and resolved 7 acceptance findings. |
| Final autonomous validation (live reproduction + citation sweep) | 3.0 | Independent re‑run of every documented observation in the live container plus a full citation in‑range sweep; all five gates passed with zero edits. |
| **Total Completed** | **30.0** | All autonomous (AI) work. |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|------:|----------|
| SME technical review & acceptance of the document | 2.0 | High |
| Reproduce build/run recipe in reviewer environment | 2.0 | Medium |
| PR approval & merge/publish | 0.5 | Medium |
| Disposition of untracked `blitzy/screenshots/` artifacts | 0.5 | Low |
| **Total Remaining** | **5.0** | — |

> **Integrity:** §2.1 (30.0) + §2.2 (5.0) = **35.0** total hours, matching §1.2. §2.2 total (5.0) equals the §1.2 Remaining Hours and the §7 pie "Remaining Work".

---

## 3. Test Results

This deliverable is a documentation/verification artifact; the AAP explicitly forbids adding or modifying tests, and the source tree is read‑only. Accordingly, the "test suite" here is **Blitzy's autonomous validation** — static citation verification plus live reproduction of every documented runtime observation. **All entries below originate from Blitzy's autonomous validation logs for this project** (container `sl-app-0`).

| Test Category | Framework / Method | Total Checks | Passed | Failed | Coverage | Notes |
|---------------|--------------------|-------------:|-------:|-------:|----------|-------|
| Citation Accuracy (static) | Blitzy citation sweep | 138 | 138 | 0 | 670 line refs across 30 files, 100% in‑range | Zero missing files, zero out‑of‑range; 19 key files sha256‑identical between repo and running container. |
| Q1 — Readiness reproduction | Live HTTP (curl) + process/log inspection | 8 | 8 | 0 | Web + mail tiers | `>>> init logging <<<`, `SL` logs, `/health`→200, `/`→302→`/auth/login`, `/auth/login`→200, `after_request` logs, `Listen for port 20381`, `Start mail controller`. |
| Q2 — New‑user journey reproduction | Live HTTP + read‑only ORM | 6 | 6 | 0 | register→activate→login→dashboard | `/auth/register`→200, 30‑char `ActivationCode` retrieved, activation→302→`/dashboard/`→200 (auto‑login), seeded login→302→`/dashboard/`→200. |
| Q3 — Behind‑the‑scenes reproduction | Live SMTP (smtplib) + ORM/psql + log inspection | 9 | 9 | 0 | jobs, forwarding, events, cache | `DISABLE_ONBOARDING`⇒0 jobs, probe `Job` dequeued (`Take job`)+done, inbound `New message`(INFO)+`Forward …` with shared `message_id`, `250` accepted, `Not sending events` guard, Redis `PONG`, `send_undelivered_mails */5`. |
| Cleanup & DB‑state verification | Read‑only `psql` assertions | 6 | 6 | 0 | Seeded baseline | users=2, job=0, activation_code=0, `e1@sl.local` contacts=0, seed `john@wick.com` intact/activated, temp users=0. |
| **Total** | — | **167** | **167** | **0** | **100% pass** | 29 live runtime checks + 138 static citation checks. |

> **Out of scope (not executed):** SimpleLogin's existing `pytest` suite (~120 test files) was **not** part of this documentation task and was not run — the AAP forbids modifying tests and keeps the source read‑only. No new tests were added.

---

## 4. Runtime Validation & UI Verification

**Status key:** ✅ Operational · ⚠ Partial / Observation · ❌ Failing

**Web tier (`python3 server.py`, :7777)**
- ✅ `GET /health` → **200** body `success`.
- ✅ `GET /` (unauthenticated) → **302** → `/auth/login`.
- ✅ `GET /auth/login` → **200**, renders `<title> Login | SimpleLogin`.
- ✅ `>>> init logging <<<` banner + colored `SL`‑prefixed logs at startup.
- ✅ Custom `after_request` access‑log line per request (Werkzeug default logger intentionally disabled).

**Mail tier (`python email_handler.py`, :20381)**
- ✅ `Listen for port 20381` (INFO) and `Start mail controller 0.0.0.0 20381` (DEBUG) — inbound SMTP listener bound and ready.

**Background workers**
- ✅ `job_runner.py` 10‑second loop dequeues a probe `Job` (`Take job …`) and marks it done.
- ✅ `email_handler.py` forwarding: `Forward <Contact> -> <Alias e1@sl.local> -> <Mailbox john@wick.com>` with a shared `message_id` across all correlated log lines, ending in `250 Message accepted`.
- ✅ Event guard `Not sending events …` fires as designed (no webhook/partner configured locally).
- ✅ Redis active in the validation environment (`redis-cli ping` → `PONG`, `session:*` keys present).

**UI verification (unmodified upstream templates)**
- ✅ Login page, `register_waiting_activation` ("Activation Email Sent"), and the alias dashboard render correctly for both the temporary user and the seeded account.
- ✅ Activation flash "Your account has been activated" with auto‑login into `/dashboard/`.
- ⚠ Three **observed UI‑quality notes** (properties of the read‑only upstream templates, **not** failures of the asked behaviors): low‑contrast bad‑password toast; auth inputs omit `autocomplete`/consistent programmatic labels; some absolute static‑asset URLs (avatar) assume the configured `URL` origin. These are documented as observations only — the source is left unchanged.

**API / integration outcomes**
- ✅ Identity verification (activation code → `activated=True` → session) and email‑forwarding pipeline both verified end‑to‑end via live HTTP/SMTP.

---

## 5. Compliance & Quality Review

Cross‑map of AAP deliverables/constraints to outcomes. Progress: 🟦 (Dark Blue `#5B39F3`) = complete; ⬜ (White `#FFFFFF`) = remaining.

| AAP Requirement / Benchmark | Status | Progress | Evidence |
|-----------------------------|--------|----------|----------|
| Single document created at correct path/name (`blitzy/documentation/app_2cd6ee777f8c.md`) | ✅ Pass | 🟦 | `git diff --name-status 2cd6ee77..HEAD` = `A blitzy/documentation/app_2cd6ee777f8c.md`. |
| Source tree strictly read‑only (byte‑for‑byte unchanged) | ✅ Pass | 🟦 | Zero source files in any of the 4 commits; only the doc added. |
| No code added to source repo besides the doc | ✅ Pass | 🟦 | No helper scripts/fixtures/modules added. |
| Every claim code‑cited (code = source of truth) | ✅ Pass | 🟦 | 118+ citations; reference index; 100% in‑range; 11/11 spot‑check exact. |
| Rationale provided ("why", not just "what") | ✅ Pass | 🟦 | 20 explicit "Rationale" passages. |
| Build & run methodology (live evidence) | ✅ Pass | 🟦 | 9 "Observed" log/HTTP excerpt blocks from a running instance. |
| Distinguish deterministic vs environment‑dependent signals | ✅ Pass | 🟦 | Q1 leads with deterministic markers; Werkzeug banner flagged env‑dependent with code rationale. |
| Explain local email nuance (`NOT_SEND_EMAIL`) | ✅ Pass | 🟦 | Documented; activation code obtained via read‑only ORM. |
| Temporary test data removed (seeded baseline restored) | ✅ Pass | 🟦 | ORM teardown + post‑cleanup `psql` verification; net DB impact zero. |
| No dependency / feature / refactor / CI‑CD / infra changes | ✅ Pass | 🟦 | None present in the diff. |
| Human SME review & acceptance | ⬜ Pending | ⬜ | Path‑to‑production; see §2.2 / §1.6. |

**Fixes applied during autonomous validation:** code‑review findings (commit `6dd65063`), Q3 `job_runner` self‑enqueue correction (`3c74addf`), and 7 acceptance findings (`46e0e9ac`). The Final Validator required **zero further edits**.

---

## 6. Risk Assessment

Overall risk profile is **Low**: a read‑only documentation deliverable ships no runtime code, so there is no new attack surface or regression risk. Residual risks concern document durability, reproduction portability, and artifact disposition.

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Citation drift — `file:line` references pinned to commit `app_2cd6ee777f8c` may go stale as the source evolves | Technical | Low | Medium (over time) | Doc states it is pinned to the source branch; treat as a point‑in‑time snapshot; re‑anchor if re‑validated against a newer commit | Open (inherent) |
| Build/run recipe portability — validated in the specific prebuilt image; other hosts may hit `DB_URI` port or Flask loopback‑binding issues | Technical | Low | Medium | Doc explicitly documents the `DB_URI` port reconciliation and the `0.0.0.0`/`URL` host‑browser guidance | Mitigated in doc |
| Local dev credentials referenced (`john@wick.com/password`, `FLASK_SECRET=secret`) mistaken for production values | Security | Low | Low | Framed as local seed/dev values; verified to be public upstream defaults in `app/fake_data.py` — not real secrets | Mitigated in doc |
| No new attack surface (zero source/dependency changes) | Security | None | — | N/A — noted for completeness | N/A |
| Untracked `blitzy/screenshots/` (55 PNGs, ~11 MB) could be committed by a blind `git add -A`, violating the single‑file mandate | Operational | Low‑Med | Low | Flagged for human disposition; committed doc embeds zero images (verified) | Open (human decision) |
| Visual‑evidence confusion — prior‑agent screenshots not referenced by the doc | Operational | Low | Low | Documented as a disposition decision | Open (human decision) |
| External MX dependency for Q2 reproduction (`email_can_be_used_as_mailbox`) may fail in network‑restricted envs | Integration | Low | Low‑Med | Doc notes the throwaway domain needs valid MX; reviewer can use a real‑MX domain or the seeded account | Mitigated in doc |
| Optional services silent locally (Redis/New Relic/Plausible/event webhook) misread as defects | Integration | Low | Low | Doc explains each is optional/no‑op locally and that silence is by design | Mitigated in doc |

---

## 7. Visual Project Status

**Project hours — completed vs remaining** (Completed = Dark Blue `#5B39F3`, Remaining = White `#FFFFFF`):

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieOuterStrokeWidth':'2px','pieTitleTextSize':'15px','pieSectionTextSize':'13px','pieLegendTextSize':'13px'}}}%%
pie showData title Project Hours (Total 35.0h — 85.7% Complete)
    "Completed Work" : 30
    "Remaining Work" : 5
```

**Remaining work by category (hours)** — from §2.2:

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#B23AF2','pie3':'#A8FDD9','pie4':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'1px','pieOuterStrokeColor':'#B23AF2'}}}%%
pie showData title Remaining Hours by Category (Total 5.0h)
    "SME technical review (High)" : 2.0
    "Reproduce build/run recipe (Medium)" : 2.0
    "PR approval & merge (Medium)" : 0.5
    "Screenshot disposition (Low)" : 0.5
```

> **Integrity:** the "Remaining Work" slice (5) equals §1.2 Remaining Hours and the §2.2 total. The category chart sums to 5.0.

---

## 8. Summary & Recommendations

**Achievements.** The project is **85.7% complete** (30.0 of 35.0 AAP‑scoped hours). The single required artifact — a code‑grounded, live‑run verification of SimpleLogin's first‑run behavior — is fully authored (456 lines), precisely cited (118+ `file:line` references; 100% citation accuracy), backed by real observed evidence, and committed. It answers all three questions (readiness, new‑user journey, behind‑the‑scenes services) with rationale, and it honors every constraint: the source tree is byte‑for‑byte unchanged, all temporary test data was removed (net DB impact zero), and no dependencies/features/tests/CI were touched.

**Remaining gaps (critical path to production).** The outstanding 5.0 hours are entirely **human review/acceptance**, with no known defects: (1) an SME technical review of the document, (2) reproducing the build/run recipe in the reviewer's environment, (3) PR approval and merge, and (4) deciding the disposition of the untracked screenshots directory. The critical path is short and low‑risk: SME review → optional reproduction → merge.

**Success metrics.** Citation accuracy 100% (167/167 autonomous checks passed, including 138 citation and 29 live‑runtime checks); zero doc↔runtime mismatches; zero source modifications; net‑zero database impact after cleanup.

**Production readiness assessment.** For a documentation deliverable, "production" means review‑and‑merge. The artifact is **ready for human review** and carries no release blockers. Recommendation: **proceed to SME review and merge.** Confidence is **High** — the autonomous validation passed all five gates and was independently corroborated during this assessment.

| Metric | Value |
|--------|-------|
| Completion | 85.7% |
| Completed / Total hours | 30.0 / 35.0 |
| Autonomous validation checks passed | 167 / 167 |
| Source files modified | 0 |
| Critical unresolved issues | 0 |

---

## 9. Development Guide

How to build, run, verify, and troubleshoot SimpleLogin locally so you can reproduce every observation in the deliverable. Commands are taken from `CONTRIBUTING.md` and were structurally verified during this assessment.

### 9.1 System Prerequisites

- **Python 3.10** (managed with Poetry) — `CONTRIBUTING.md:23`.
- **Node v10** — front‑end assets only — `CONTRIBUTING.md:24`.
- **PostgreSQL 13+** — `CONTRIBUTING.md:25`.
- **Docker** — to run PostgreSQL and/or the prebuilt image.
- **Redis** — optional locally (sessions + rate‑limit storage when `MEM_STORE_URI` is set).

> **Fastest path (recommended):** use the prebuilt image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (Python 3.10.18, assets prebuilt, PostgreSQL + Redis preinstalled, source baked at commit `app_2cd6ee777f8c`). A generic host with a newer Python (e.g., 3.13) / Node (e.g., 20) will not match the pinned runtime.

### 9.2 Environment Setup

```bash
# Create the local env file from the template
cp example.env .env

# Reconcile DB_URI to the Postgres host port you actually publish.
# example.env ships :5432; CONTRIBUTING.md shows :35432; scripts/reset_local_db.sh uses :15432.
# Pick ONE and make .env's DB_URI match the published port (15432 is the most consistent).
#   DB_URI=postgresql://myuser:mypassword@localhost:15432/simplelogin
```

Key local environment values (`example.env`): `URL=http://localhost:7777` (L6), `NOT_SEND_EMAIL=true` (L19), `EMAIL_DOMAIN=sl.local` (L22), `FLASK_SECRET=secret` (L77), `DISABLE_ONBOARDING=true` (L150).

### 9.3 Dependency Installation

```bash
# Python dependencies (Poetry)
poetry sync                 # CONTRIBUTING.md:31

# Front-end assets (Node v10)
cd static && npm install    # CONTRIBUTING.md:82
cd ..
```

### 9.4 Application Startup

```bash
# 1) Start PostgreSQL 13 (publish to host port 15432 to match the DB_URI above)
docker run -e POSTGRES_PASSWORD=mypassword -e POSTGRES_USER=myuser \
  -e POSTGRES_DB=simplelogin -p 15432:5432 postgres:13

# 2) Migrate, seed, and run the webapp (in a second shell, from the repo root)
alembic upgrade head && flask dummy-data && python3 server.py   # web on :7777

# 3) (For Q3) run the mail handler and job worker as separate processes
python email_handler.py     # inbound SMTP on :20381
python job_runner.py        # 10-second job-draining loop
```

> **Production parity (host‑browser access in Docker):** `python3 server.py` runs the Flask dev server bound to container‑loopback only. To reach the UI from a host browser, serve on all interfaces — `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` — and set `URL` to the origin the browser actually uses.

### 9.5 Verification Steps

```bash
# Web tier readiness (run inside the app environment/container)
curl -s http://127.0.0.1:7777/health            # => "success" (HTTP 200)
curl -s -o /dev/null -w "%{http_code} %{redirect_url}\n" http://127.0.0.1:7777/   # => 302 .../auth/login
curl -s http://127.0.0.1:7777/auth/login | grep -o "<title>[^<]*</title>"          # => Login | SimpleLogin

# Logging smoke test (expect the banner on import)
python -c "import app.log"                       # stdout: >>> init logging <<<

# Mail tier readiness — look for these lines in the email_handler.py output:
#   ... SL - INFO  - Listen for port 20381
#   ... SL - DEBUG - Start mail controller 0.0.0.0 20381

# Inbound forwarding test (alias domain is sl.local)
python3 - <<'PY'
import smtplib
from email.message import EmailMessage
m = EmailMessage(); m["From"]="hey@google.com"; m["To"]="e1@sl.local"
m["Subject"]="inbound forwarding test"; m.set_content("hello")
with smtplib.SMTP("127.0.0.1", 20381, timeout=20) as s: s.send_message(m)
PY
# Expect in email_handler logs: "New message …" then "Forward <Contact> -> <Alias e1@sl.local> -> <Mailbox john@wick.com>"
```

Deliverable / repository verification (validated during this assessment):

```bash
git diff --name-status 2cd6ee77..HEAD                                   # => A blitzy/documentation/app_2cd6ee777f8c.md
git log --author="agent@blitzy.com" 2cd6ee77..HEAD --oneline            # => 4 doc commits
wc -l blitzy/documentation/app_2cd6ee777f8c.md                          # => 456
```

### 9.6 Example Usage

- **Log in with the seeded account:** open `http://localhost:7777` → `john@wick.com / password` (pre‑activated by `flask dummy-data`) → lands on the alias dashboard.
- **Register a new user:** submit `/auth/register` → "Activation Email Sent" page. Because `NOT_SEND_EMAIL=true`, no email is delivered — the "send email with subject 'Just one more step…'" log line is the success signal. Obtain the code via read‑only ORM:
  ```python
  # flask shell  (DB_URI read from .env)
  from app.models import User, ActivationCode
  u = User.get_by(email="<your-temp>@gmail.com")
  print(ActivationCode.get_by(user_id=u.id).code)   # 30-char code
  ```
  Then visit `/auth/activate?code=<code>` → auto‑login → `/dashboard/`.
- **Clean up temporary data** (restore the seeded baseline) via `flask shell` using `User.delete(...)`, deleting the test `Contact`, and removing any probe `Job` — exactly as documented in §(e) of the deliverable.

### 9.7 Troubleshooting

- **`connection refused` on startup:** `.env` `DB_URI` port doesn't match the published Postgres host port — reconcile them (e.g., both `15432`).
- **UI unreachable from a host browser (`localhost:7777` reset):** the dev server binds container‑loopback only — use `gunicorn … -b 0.0.0.0:7777` and set `URL` to the browser‑facing origin.
- **No activation email arrives:** expected — `NOT_SEND_EMAIL=true`. The activation path logs the email; read the code from the DB/ORM.
- **`Uncaught ReferenceError: plausible is not defined` in the console:** benign — Plausible analytics is unset locally; registration completes normally.
- **No sync‑event traffic / New Relic silence:** expected — these are optional and no‑op without configuration.

---

## 10. Appendices

### Appendix A — Command Reference

| Purpose | Command |
|---------|---------|
| Install Python deps | `poetry sync` |
| Build front‑end assets | `cd static && npm install` |
| Create env file | `cp example.env .env` |
| Start PostgreSQL 13 | `docker run -e POSTGRES_PASSWORD=mypassword -e POSTGRES_USER=myuser -e POSTGRES_DB=simplelogin -p 15432:5432 postgres:13` |
| Migrate + seed + run web | `alembic upgrade head && flask dummy-data && python3 server.py` |
| Run mail handler | `python email_handler.py` |
| Run job worker | `python job_runner.py` |
| Production WSGI | `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` |
| Health check | `curl -s http://127.0.0.1:7777/health` |
| Full local DB reset | `scripts/reset_local_db.sh` |
| Verify single‑file diff | `git diff --name-status 2cd6ee77..HEAD` |

### Appendix B — Port Reference

| Port | Service | Notes |
|------|---------|-------|
| 7777 | Flask webapp (dev server / Gunicorn) | `app.run(debug=True, port=7777)`; prod `EXPOSE 7777` |
| 20381 | `email_handler.py` inbound SMTP (aiosmtpd) | bound `0.0.0.0`; default port `20381` |
| 15432 | PostgreSQL (host‑published) | most consistent choice (`scripts/reset_local_db.sh`) |
| 35432 / 5432 | PostgreSQL (alt. host ports) | `CONTRIBUTING.md` shows `35432`; `example.env` ships `5432` — reconcile with `DB_URI` |
| 6379 | Redis (optional) | active only when `MEM_STORE_URI` is set |

### Appendix C — Key File Locations

| Path | Role |
|------|------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The deliverable** (this project's single created artifact) |
| `server.py` | Flask app factory, `/health`, index redirect, `flask dummy-data` CLI |
| `wsgi.py` | Production WSGI entrypoint (Gunicorn) |
| `email_handler.py` | aiosmtpd inbound SMTP server + forwarding |
| `job_runner.py` | 10‑second job‑draining loop |
| `cron.py` + `crontab.yml` | yacron‑scheduled maintenance jobs |
| `event_listener.py` | `NOTIFY`/`LISTEN` sync‑event consumer |
| `app/log.py` | `SL` logger, `>>> init logging <<<`, Werkzeug logger disabled |
| `app/config.py` | `NOT_SEND_EMAIL`, `URL`, `MEM_STORE_URI`, etc. |
| `app/auth/views/{register,activate,login,login_utils}.py` | Q2 journey handlers |
| `app/dashboard/views/index.py` | Dashboard landing route |
| `CONTRIBUTING.md` / `example.env` / `Dockerfile` | Build/run recipe + env + runtime pins |

### Appendix D — Technology Versions

| Component | Version | Source |
|-----------|---------|--------|
| Python | 3.10 (image: 3.10.18) | `CONTRIBUTING.md:23`; `Dockerfile` `FROM python:3.10` |
| Node.js | 10.17.0‑alpine (assets only) | `Dockerfile:2` |
| PostgreSQL | 13+ | `CONTRIBUTING.md:25` |
| Redis | optional | `app/config.py:568` |
| Flask | ^1.1.2 | `pyproject.toml` |
| SQLAlchemy | 1.3.24 (pinned) | `pyproject.toml` |
| aiosmtpd | ^1.2 | `pyproject.toml` |
| gunicorn | ^20.0.4 | `pyproject.toml`; `Dockerfile` |
| Poetry | dependency manager | `CONTRIBUTING.md` |

### Appendix E — Environment Variable Reference

| Variable | Local value | Purpose |
|----------|-------------|---------|
| `URL` | `http://localhost:7777` | Absolute base origin for generated links and some absolute asset URLs |
| `NOT_SEND_EMAIL` | `true` | Outbound email is logged, not transmitted (presence‑check) |
| `EMAIL_DOMAIN` | `sl.local` | Alias domain; inbound test mail targets `@sl.local` |
| `FLASK_SECRET` | `secret` | Signs the session cookie (local dev value only) |
| `DB_URI` | `postgresql://myuser:mypassword@localhost:<port>/simplelogin` | Postgres connection — reconcile `<port>` to the published host port |
| `DISABLE_ONBOARDING` | `true` | Short‑circuits onboarding‑email jobs (so a normal registration enqueues 0 jobs) |
| `MEM_STORE_URI` | unset in `example.env` (set to `redis://localhost` in the validation image) | When set, enables Redis sessions + rate‑limit storage |
| `DISABLE_REGISTRATION` | commented out → registration enabled | Allows the Q2 walkthrough to register |

### Appendix F — Developer Tools Guide

- **Live readiness checks:** `curl` the `/health`, `/`, and `/auth/login` endpoints (see §9.5).
- **Log inspection:** every app log line is `SL`‑prefixed and includes `pathname:lineno`, function, and a `message_id` field — use the `message_id` to trace one email across many lines.
- **Database inspection (read‑only):** `psql "$DB_URI" -c "select count(*) from users;"` and similar; or `flask shell` with the project ORM (`User.get_by`, `ActivationCode.get_by`).
- **Inbound mail testing:** `swaks --to e1@sl.local --from hey@google.com --server 127.0.0.1:20381` or the `smtplib` snippet in §9.5.
- **Job loop demonstration:** enqueue a throwaway `Job` and watch `job_runner.py` log `Take job …` within ~10 seconds.

### Appendix G — Glossary

| Term | Meaning |
|------|---------|
| **AAP** | Agent Action Plan — the authoritative scope for this project. |
| **Alias** | A SimpleLogin email address that forwards to a real mailbox. |
| **Activation code** | One‑time, 1‑hour `ActivationCode` proving control of an address. |
| **`NOT_SEND_EMAIL`** | Local flag making outbound email log‑only (no SMTP delivery). |
| **`SL` logger** | SimpleLogin's single project logger (`app/log.py`). |
| **job_runner** | Worker that dequeues `Job` rows from PostgreSQL every 10 seconds. |
| **NOTIFY/LISTEN** | PostgreSQL pub/sub channel used for partner sync events. |
| **Reverse alias** | Generated contact address so replies route back through SimpleLogin. |
| **Seed account** | `john@wick.com / password`, pre‑activated by `flask dummy-data`. |

---

*This Project Guide reflects the AAP‑scoped completion of a documentation/verification deliverable. Completed work (30.0h) is autonomous; remaining work (5.0h) is human review/acceptance. Source tree unchanged; net database impact zero.*