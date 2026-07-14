# Blitzy Project Guide — SimpleLogin Self-Host Runtime Verification

> **Deliverable:** `blitzy/documentation/app_2cd6ee777f8c.md` — an evidence-backed Q&A report verifying, by live runtime observation, that a first-time self-hosted **SimpleLogin** deployment is healthy across user authentication and alias-based email activity.
> **Task class:** Read-only runtime-verification documentation (`SWE-AtlasQnA-Repo`). **Source changes: none.**

---

## 1. Executive Summary

### 1.1 Project Overview

SimpleLogin is a full-stack Python/Flask email-alias service organized into three cooperating runtime tiers — the Flask web application, the SMTP email handler, and background services — sharing a PostgreSQL database and Redis as their substrate. The Agent Action Plan (AAP) scoped a **read-only runtime-verification task**: boot the software in its default local configuration and produce one Markdown document answering three verification questions (startup readiness, the new-user register→verify→login→dashboard walkthrough, and behind-the-scenes background jobs/internal services). The target audience is an engineer standing up the platform for the first time who needs live confirmation it works. Every claim is grounded in captured runtime output plus `file:line` citations. No product feature was added and **no application source file was modified**.

### 1.2 Completion Status

Completion is measured strictly against AAP-scoped work plus path-to-production activities (PA1 methodology), using an hours-based formula: `Completed / (Completed + Remaining)`.

```mermaid
%%{init: {'theme':'base','themeVariables':{'pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pie1':'#5B39F3','pie2':'#FFFFFF'}}}%%
pie showData title Completion — 87.7%
    "Completed Work (h)" : 50
    "Remaining Work (h)" : 7
```

| Metric | Value |
|---|---|
| **Total Hours** | **57** |
| **Completed Hours (AI + Manual)** | **50** (AI: 50 · Manual: 0) |
| **Remaining Hours** | **7** |
| **Percent Complete** | **87.7%** (50 ÷ 57) |

> Color key — **Completed = Dark Blue `#5B39F3`**, **Remaining = White `#FFFFFF`**.

### 1.3 Key Accomplishments

- ✅ Canonical stack provisioned and booted (Python 3.10, PostgreSQL 13, Redis 6) exactly as a normal self-hoster would.
- ✅ **Objective 1** — startup readiness signals captured live across the web (:7777), email (:20381), and job-runner tiers.
- ✅ **Objective 2** — full `register → verify → login → dashboard` walkthrough exercised over real HTTP, including the `User.activated` False→True transition and **all** error/edge/boundary paths.
- ✅ **Objective 3** — background jobs (`ready→taken→done`), email forwarding via `swaks`, Postgres `LISTEN/NOTIFY`, default event no-op, and the yacron schedule all observed.
- ✅ 3,333-line answer document authored with actual unedited output, exact commands, `file:line` citations, an Inferred-vs-Observed ledger, and a coverage-pass matrix.
- ✅ Blitzy autonomous validation: **639/639 tests pass, 71.76% coverage**; all entry points boot; byte-compile & imports clean.
- ✅ Read-only guarantee upheld: `git diff` shows exactly one file added; all temporary test data removed via canonical DB reset.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|---|---|---|---|
| _None._ No code defects or blocking issues. The deliverable is complete and every runtime claim reproduced live. | N/A | N/A | N/A |

> Remaining items are human path-to-production activities (review, artifact disposition, merge), tracked in §1.6, §2.2, and §6 — not defects.

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|---|---|---|---|---|
| Source repository | Read/Write (branch) | Full access; deliverable committed by prior agents | ✅ Resolved | Blitzy |
| Canonical `python:3.10` image | Pull/build | Used to install locked deps cleanly (avoids `cbor2` sdist issue on newer OS) | ✅ Resolved | Blitzy |
| PostgreSQL 13 / Redis 6 | Local service | Provisioned as sidecars for boot & tests | ✅ Resolved | Blitzy |

**No access issues identified** that block build validation, integration, or deployment of this documentation deliverable.

### 1.6 Recommended Next Steps

1. **[High]** SME technical review & acceptance of the 3,333-line answer document against the three AAP questions.
2. **[Medium]** Decide disposition of the untracked browser-evidence artifacts (`blitzy/lighthouse`, `blitzy/screen_recordings`, `blitzy/screenshots` — 252 files) left by prior agents: commit, relocate, or discard.
3. **[Medium]** Review and merge the documentation branch (single-file PR).
4. **[Low]** Optionally re-reproduce a sample of runtime claims in the canonical image to independently confirm reproducibility.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

Every row traces to an AAP objective, methodology requirement, or path-to-production gate. All hours are autonomous (AI) work.

| Component | Hours | Description |
|---|---:|---|
| Environment provisioning & canonical boot | 6 | `python:3.10` image + PG13/Redis6 sidecars; `sl-app:runtime` derived for the pyre2 0.3.6 fix; `cp example.env .env`; `alembic upgrade head` → `32f25cbf12f6` (77 tables); `flask dummy-data` (seeds `john@wick.com`) — AAP A1/E3/E7 |
| Objective 1 investigation + §3 authoring | 5 | Boot three entry points; capture `>>> init logging <<<`, Flask/debug banner, blueprint registration, `Listen for port 20381` / `Start mail controller` — AAP B1–B6 |
| Objective 2 investigation + §4 authoring | 9 | Real-HTTP `register→verify→login→dashboard`; `User.activated` transition; happy path + all edges (login-before-activation, wrong password, invalid/expired code, replay, password-length boundaries, resend) — AAP C1–C12 |
| Objective 3 investigation + §5 authoring | 9 | GDPR-export `Job` `ready→taken→done`; `swaks` forwarding through :20381; Postgres `NOTIFY simplelogin_sync_events`; default event no-op; yacron schedule — AAP D1–D8 |
| Document authoring (§1, §1.1, §2, §6, §7, §8) | 9 | HEALTHY verdict + security scope; environment & exact commands; Inferred-vs-Observed ledger; coverage matrix; cleanup section — AAP A1/E1–E7 |
| QA remediation across 6 commits | 6 | ~20+ code-review findings, citation fixes, reproducibility (DOC-MAJ-1), §1.1 security additions |
| Read-only guarantee + cleanup | 2 | Canonical `DROP SCHEMA public CASCADE` + `alembic upgrade head` + `flask dummy-data` reset; before/after SQL; verify tree unchanged — AAP F1/F2 |
| Autonomous validation gates | 4 | 639/639 tests, 71.76% coverage, entry-point boots, byte-compile & import checks — path-to-production |
| **Total Completed** | **50** | |

> **Validation:** total of the Hours column = **50** = Completed Hours in §1.2. ✔

### 2.2 Remaining Work Detail

Each category is a human path-to-production activity; no autonomous code work remains.

| Category | Hours | Priority |
|---|---:|---|
| Human SME technical review & acceptance of the answer document vs. the 3 questions | 3 | High |
| Disposition of untracked browser-evidence artifacts (252 files: lighthouse/recordings/screenshots) | 1 | Medium |
| PR review & merge of the documentation branch | 1 | Medium |
| Optional independent re-reproduction of sample runtime claims in the canonical image | 2 | Low |
| **Total Remaining** | **7** | |

> **Validation:** total of the Hours column = **7** = Remaining Hours in §1.2 = §7 pie "Remaining Work". ✔

### 2.3 Total Hours Reconciliation

| Reconciliation | Value |
|---|---:|
| Section 2.1 Completed | 50 |
| Section 2.2 Remaining | 7 |
| **Total (2.1 + 2.2)** | **57** |
| Completion % = 50 ÷ 57 × 100 | **87.7%** |

Cross-section integrity — **Rule 1** (§1.2 = §2.2 = §7 remaining): 7 = 7 = 7 ✔ · **Rule 2** (§2.1 + §2.2 = Total): 50 + 7 = 57 ✔.

---

## 3. Test Results

All tests below originate exclusively from Blitzy's autonomous validation logs for this project (executed inside the derived `sl-app:runtime` image with `CONFIG=tests/test.env -c pytest.ci.ini` against a pre-migrated PostgreSQL 13 + Redis 6).

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---|---|---:|---:|---:|---:|---|
| Full suite (unit + integration) | pytest | 639 | 639 | 0 | 71.76% | 0 errors, 0 skipped. Coverage gate is `fail_under=55`; actual **71.76% > 55%**. Includes the 8 IPv6-dependent `tests/test_mail_sender.py` cases (sysctl fix applied). |

**Supplementary autonomous checks (non-pytest):**

- **Byte-compilation:** all 7 canonical entry points (`server.py`, `wsgi.py`, `email_handler.py`, `job_runner.py`, `cron.py`, `event_listener.py`, `init_app.py`) + `app/` → **EXIT 0**.
- **Import validation:** all 7 canonical modules import cleanly (`email_handler` imports because the runtime venv uses `pyre2 0.3.6` providing `re2.DOTALL`).
- **Migration:** `alembic upgrade head` → `32f25cbf12f6` (EXIT 0, 77 public tables).
- **Seed:** `flask dummy-data` (EXIT 0, seeds `john@wick.com`).

> Note: the `htmlcov` headline shows **72%** (rounded / `skip_covered`); the authoritative log figure is **71.76%**.

---

## 4. Runtime Validation & UI Verification

All three canonical tiers plus the auxiliary services were booted through their **real entry points** and observed serving live.

| Tier | Entry point | Readiness signal (observed) | Status |
|---|---|---|---|
| Web / auth | `python server.py` (:7777) | Flask banner + "Debug mode: on"; `GET /auth/login` → **200**; `GET /` → **302** login redirect; auth/dashboard blueprints mounted | ✅ Operational |
| Email / alias | `python email_handler.py` (:20381) | `Listen for port 20381` [email_handler.py:L2403] + `Start mail controller 0.0.0.0 20381` [L2386]; SMTP banner `220 <host> Python SMTP 1.4.2` | ✅ Operational |
| Background jobs | `python job_runner.py` | 10-second poll loop; `Take job <Job 1 send-user-report>` [job_runner.py:L334]; `Job.state` `ready→taken→done` | ✅ Operational |
| Scheduler | `python cron.py -j <job>` | Cronjob runs; 15-entry yacron schedule in `crontab.yml` (`send_undelivered_mails` every `*/5`) | ✅ Operational |
| Event listener | `python event_listener.py listener` | `Using PostgresEventSource`; `Starting to listen to events` | ✅ Operational |
| Datastore | PostgreSQL 13 | schema migrated to head `32f25cbf12f6`; 77 public tables | ✅ Operational |

**UI / product-flow verification (real HTTP):**

- ✅ `POST /auth/register` → 200, `register_waiting_activation.html` rendered; 30-char `ActivationCode` created; `NOT_SEND_EMAIL` log signal names subject "Just one more step to join SimpleLogin".
- ✅ `GET /auth/activate?code=…` → **302** → `/dashboard/` → 200; flash "Your account has been activated"; `User.activated` **False→True** [activate.py:L49]; one-time code deleted.
- ✅ `POST /auth/login` → **302** `/dashboard/`; authenticated dashboard renders 200.
- ✅ Edge paths: login-before-activation (200 + "Please check your inbox…"), wrong password (200 + "Email or password incorrect"), invalid code (**400** "cannot be found"), expired code (**400** "was expired"), replay-while-authenticated (**400** "You are already logged in"); password length 7→rejected / 8→accepted / 100→accepted / 101→rejected.

**Default-configuration nuances (correct behavior, not failures):**

- ⚠ Werkzeug "Running on…" line intentionally **absent** (logger disabled at [app/log.py:L70-71]) — the bind is still on :7777 via [server.py:L588].
- ⚠ `MEM_STORE_URI=None` — Redis is **not wired by the web app** under the default config [app/config.py:L568]; event dispatch is a documented no-op [app/events/event_dispatcher.py:L62].

---

## 5. Compliance & Quality Review

Cross-mapping AAP deliverables and `SWE-AtlasQnA-Repo` rules to Blitzy quality benchmarks. Fixes were applied across 6 autonomous commits.

| Requirement / Benchmark | Evidence | Status | Progress |
|---|---|---|---|
| Deliverable location & name (`blitzy/documentation/<branch>.md`) | `blitzy/documentation/app_2cd6ee777f8c.md` present | ✅ Pass | 100% |
| Investigate by RUNNING first, then write | Live captures for all 3 objectives; run-first methodology in §2 | ✅ Pass | 100% |
| Canonical entry points only | Web routes, aiosmtpd :20381, job_runner loop, yacron — non-canonical probes labeled | ✅ Pass | 100% |
| Default, canonical configuration | `example.env` defaults (`NOT_SEND_EMAIL=true`, `DISABLE_ONBOARDING=true`); exact commands stated | ✅ Pass | 100% |
| Exercise every condition (happy + edge) | All edges/boundaries reproduced (§4) | ✅ Pass | 100% |
| Actual, complete, unedited output | Raw output + producing command per claim | ✅ Pass | 100% |
| Inferred vs Observed labeling | §6 ledger; per-row evidence status (Observed / Observed-negative / Source-derived / Non-canonical) | ✅ Pass | 100% |
| Coverage pass over every named item | §7 coverage matrix in deliverable | ✅ Pass | 100% |
| `file:line` citations to named functions | 9/9 spot-checked citations accurate vs current source | ✅ Pass | 100% |
| Read-only source guarantee | `git diff 2cd6ee77 HEAD` = 1 file added; no tracked source/config edits | ✅ Pass | 100% |
| Cleanup of temporary artifacts | Canonical DB reset; users back to seed-only baseline | ✅ Pass | 100% |
| Immutable dependency manifests | `pyproject.toml` / `poetry.lock` unchanged; `cbor2` worked around via canonical image | ✅ Pass | 100% |

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|---|---|---|---|---|---|
| T1 — Env reproducibility depends on canonical `python:3.10` image + `pyre2 0.3.6` (`cbor2 5.2.0` sdist fails on newer OS; google-re2 substitution breaks `email_handler` import) | Technical | Medium | Medium | Use canonical image; runtime venv provides `re2.DOTALL`; documented in §2 of deliverable | Documented |
| T2 — Some Objective-3 signals are Source-derived / Non-canonical | Technical | Low | Low | Explicitly labeled in the Inferred-vs-Observed ledger; no overclaiming | Disclosed |
| T3 — Citation drift if source files change later | Technical | Low | Low | Citations pinned to current commit; re-verify on source change | Accepted |
| S1 — Default `example.env` is intentionally dev-insecure (`FLASK_SECRET=secret` [example.env:L77], Flask debug dev-server [server.py:L588], Debug Toolbar can leak `app.config`; demo creds `john@wick.com/password`) | Security | High (if local config exposed) | Low | Dev-only; prod uses `gunicorn wsgi:app` (no toolbar via `wsgi.py`); documented in §1.1 | Documented |
| S2 — New attack surface introduced by this task | Security | None | None | Zero source/dependency changes → zero new attack surface | Closed |
| O1 — HEALTHY verdict scoped to LOCAL DEV only (not a production security assessment) | Operational | Medium | Medium | Scope stated explicitly in §1.1 and §8 | Documented |
| O2 — 252 untracked browser-evidence files neither committed nor removed | Operational | Low | High | Human disposition decision (§1.6 / §2.2) | Open |
| I1 — Redis not wired by default (`MEM_STORE_URI=None`) | Integration | Low | Medium | Documented default behavior; sessions/rate-limit degrade gracefully locally | Documented |
| I2 — Event dispatch is a default no-op (no partner/webhook) | Integration | Low | Low | Observable negative indicator; part of the correct Objective-3 answer | Disclosed |
| I3 — Production integrations (DNS/MX/Postfix, TLS, S3, payments, captcha, APM) out of scope | Integration | Low | Low | Explicitly out of AAP scope | Accepted |

---

## 7. Visual Project Status

**Completion (hours):**

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2'}}}%%
pie showData title Project Hours — Completed vs Remaining
    "Completed Work" : 50
    "Remaining Work" : 7
```

**Remaining hours by task (Section 2.2):**

```mermaid
%%{init: {'theme':'base','themeVariables':{'xyChartBar0':'#5B39F3'}}}%%
xychart-beta
    title "Remaining Hours by Task"
    x-axis ["SME Review", "Artifacts", "PR Merge", "Re-repro"]
    y-axis "Hours" 0 --> 4
    bar [3, 1, 1, 2]
```

**Remaining work by priority:**

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#B23AF2','pie3':'#A8FDD9'}}}%%
pie showData title Remaining Hours by Priority
    "High" : 3
    "Medium" : 2
    "Low" : 2
```

> **Integrity:** "Remaining Work" = **7** here = §1.2 Remaining Hours = sum of §2.2 Hours column. Priority pie (3+2+2) and bar chart (3+1+1+2) each total **7**. Colors — Completed `#5B39F3`, Remaining `#FFFFFF`.

---

## 8. Summary & Recommendations

**Achievements.** The project is **87.7% complete** (50 of 57 AAP-scoped + path-to-production hours). Blitzy autonomously produced a comprehensive, evidence-backed 3,333-line answer document that verifies SimpleLogin's health across all three runtime tiers by **actually running the software** in its default configuration and capturing live output — never inferring from code alone. All three objectives (startup readiness, the new-user product walkthrough, and behind-the-scenes services) were reproduced live, including every error/edge/boundary path. Autonomous validation confirms a healthy build: **639/639 tests pass at 71.76% coverage**, all entry points boot, and byte-compile/import checks are clean.

**Remaining gaps.** The outstanding **7 hours** are entirely human path-to-production activities — SME technical review & acceptance, disposition of the untracked browser-evidence artifacts, PR review & merge, and optional independent re-reproduction. There are **no code defects and no blocking issues**.

**Critical path to production.** (1) SME reviews and accepts the document against the three questions → (2) decide artifact disposition → (3) merge the single-file PR. The optional re-reproduction step can run in parallel and is not on the critical path.

**Production-readiness assessment.** The deliverable is production-ready as a documentation artifact and requires only human sign-off. The HEALTHY verdict is explicitly scoped to **local development**; the default `example.env` is intentionally insecure and must not be interpreted as a production security clearance.

| Success Metric | Target | Actual | Status |
|---|---|---|---|
| AAP-specified requirements completed | 100% | 100% (37/37 autonomous items) | ✅ |
| Test pass rate | ≥ 95% | 100% (639/639) | ✅ |
| Coverage | ≥ 55% | 71.76% | ✅ |
| Citation accuracy (spot-check) | 100% | 100% (9/9) | ✅ |
| Source files modified | 0 | 0 | ✅ |
| Overall completion (hours) | — | **87.7%** | On track |

---

## 9. Development Guide

Commands are copy-pasteable and were validated during autonomous testing (tool versions verified: Docker 28.x running, Python 3.10 canonical image, PostgreSQL 13, Redis 6, `swaks`, `curl`, `git`).

### 9.1 System Prerequisites

- **Docker** (running) — used to run the canonical `python:3.10` image and PG13/Redis6 sidecars.
- **Python 3.10** — the canonical runtime (do **not** use a newer base OS to build; the pinned `cbor2 5.2.0` sdist fails there).
- **PostgreSQL 13** and **Redis 6**.
- CLI tools: `psql`, `redis-cli`, `swaks`, `curl`, `git`.

### 9.2 Environment Setup

```bash
# Start datastores (canonical ports)
docker run -d --name sl-pg -e POSTGRES_USER=myuser -e POSTGRES_PASSWORD=mypassword \
  -e POSTGRES_DB=simplelogin -p 15432:5432 postgres:13
docker run -d --name sl-redis -p 6379:6379 redis:6

# Configuration
cp example.env .env
# Ensure DB_URI points at the mapped port:
#   DB_URI=postgresql://myuser:mypassword@localhost:15432/simplelogin
# Default local flags to leave as-is:
#   NOT_SEND_EMAIL=true      (activation email is printed to the log, not sent)
#   DISABLE_ONBOARDING=true  (onboarding jobs suppressed by default)
```

### 9.3 Dependency Installation

```bash
# Inside the canonical python:3.10 environment
poetry config virtualenvs.create false
poetry install --no-interaction --no-ansi --no-root   # 181 packages install cleanly

# Frontend assets
cd static && npm install && cd ..
```

> If `email_handler.py` fails to import with a `re2` error, ensure the runtime venv has `pyre2 0.3.6` (provides `re2.DOTALL`) rather than `google-re2`. Never edit `poetry.lock` to fix this — use the canonical image.

### 9.4 Application Startup

```bash
alembic upgrade head        # migrates to 32f25cbf12f6 (77 public tables)
flask dummy-data            # seeds demo account john@wick.com / password

# Three canonical entry points (separate shells or backgrounded)
python server.py            # web app on http://localhost:7777
python email_handler.py     # SMTP on 0.0.0.0:20381
python job_runner.py        # async job loop (10s poll)

# Optional auxiliary services
python event_listener.py listener   # Postgres event listener
python cron.py -j stats             # run a single scheduled job
```

### 9.5 Verification Steps

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:7777/          # -> 302 (login redirect)
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:7777/auth/login # -> 200
printf 'QUIT\r\n' | nc -w2 127.0.0.1 20381                                # -> 220 ... Python SMTP 1.4.2
psql "postgresql://myuser:mypassword@localhost:15432/simplelogin" -c "\dt" | wc -l  # ~77 tables
redis-cli ping                                                            # -> PONG
```

Log signals confirming readiness: `>>> init logging <<<`, Flask "Debug mode: on", `Listen for port 20381`, `Start mail controller 0.0.0.0 20381`.

### 9.6 Example Usage

```bash
# End-to-end product flow: register -> activate -> login (browser at http://localhost:7777)
# Seed login: john@wick.com / password

# Exercise alias email forwarding through the SMTP entry point
swaks --to <newsletter-alias>@sl.local --from x@ext.com --server 127.0.0.1:20381
#   -> 250 Message accepted for delivery ; job_runner logs "Take job ..."

# Trigger a canonical background Job (GDPR export) from the dashboard
#   POST /dashboard/account_setting (form-name=send-full-user-report)
#   -> Job.state ready(0) -> taken(1) -> done(2)
```

### 9.7 Running the Test Suite

```bash
CONFIG=tests/test.env poetry run pytest -c pytest.ci.ini
#   -> 639 passed, 71.76% coverage (requires PostgreSQL 13 + Redis 6 running)
```

### 9.8 Troubleshooting

| Symptom | Cause | Resolution |
|---|---|---|
| `cbor2` sdist build fails | Newer base OS drops `pkg_resources`; broken PEP 517 metadata | Use the canonical `python:3.10` image. **Never** edit `poetry.lock`. |
| `email_handler.py` import error on `re2` | `google-re2` in venv instead of `pyre2` | Ensure `pyre2 0.3.6` (`re2.DOTALL`) in the runtime venv only. |
| No activation email received | `NOT_SEND_EMAIL=true` (default) | Read the activation link from the log line, not an inbox. |
| `job_runner` never picks up onboarding jobs | `DISABLE_ONBOARDING=true` (default) logs "Disable onboarding emails" | Trigger the always-run GDPR export job instead. |
| 8 `test_mail_sender.py` tests fail | IPv6 sysctl disabled | Re-apply the IPv6 sysctl in the runtime environment. |

---

## 10. Appendices

### A. Command Reference

| Command | Purpose |
|---|---|
| `alembic upgrade head` | Migrate schema to `32f25cbf12f6` (77 tables) |
| `flask dummy-data` | Seed demo account `john@wick.com` |
| `python server.py` | Web app (:7777) |
| `python email_handler.py` | SMTP handler (:20381) |
| `python job_runner.py` | Background job loop |
| `python cron.py -j <job>` | Run a single yacron job |
| `python event_listener.py listener` | Postgres event listener |
| `CONFIG=tests/test.env poetry run pytest -c pytest.ci.ini` | Full test suite |
| `git diff 2cd6ee77 HEAD --stat` | Confirm exactly one file added |

### B. Port Reference

| Port | Service |
|---|---|
| 7777 | Flask web app (dev server) |
| 20381 | aiosmtpd SMTP email handler |
| 15432 → 5432 | PostgreSQL 13 (host → container) |
| 6379 | Redis 6 |

### C. Key File Locations

| Path | Role |
|---|---|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The deliverable** (3,333 lines) |
| `server.py` / `wsgi.py` | Web app / Gunicorn WSGI entry |
| `email_handler.py` | SMTP controller + forwarding pipeline |
| `job_runner.py` / `cron.py` / `event_listener.py` | Background services |
| `app/log.py` | Logging init banner (`>>> init logging <<<`, L67) |
| `app/models.py` | `User`, `ActivationCode`, `Job`, `JobState` |
| `example.env` | Default local configuration |
| `crontab.yml` | 15-entry yacron schedule |

### D. Technology Versions

| Component | Version |
|---|---|
| Python | 3.10 |
| PostgreSQL | 13 |
| Redis | 6 |
| Flask | 1.1.2 |
| SQLAlchemy | 1.3.24 |
| aiosmtpd | 1.4.2 |
| yacron | 0.11.2 |
| gunicorn | 20.0.4 |
| Alembic | 1.4.3 |

### E. Environment Variable Reference

| Variable | Default | Effect |
|---|---|---|
| `URL` | `http://localhost:7777` | Base app URL / activation links |
| `DB_URI` | `postgresql://myuser:mypassword@localhost:5432/simplelogin` | Datastore (set port 15432 for the mapped sidecar) |
| `NOT_SEND_EMAIL` | `true` | Print email to log instead of sending — activation-email verification signal |
| `DISABLE_ONBOARDING` | `true` | Suppress onboarding jobs |
| `EMAIL_DOMAIN` | `sl.local` | Alias domain |
| `FLASK_SECRET` | `secret` | Session secret (**dev-insecure**) |

### F. Developer Tools Guide

- **`swaks`** — scriptable SMTP client to exercise the :20381 forwarding pipeline.
- **`psql` / `redis-cli`** — inspect the shared substrate (`Job`, `User`, `SyncEvent` tables; Redis PING).
- **`curl` / `nc`** — verify HTTP status codes and the SMTP banner.
- **`alembic`** — schema migration; head is `32f25cbf12f6`.
- **Canonical DB reset (cleanup):** `DROP SCHEMA public CASCADE` → `alembic upgrade head` → `flask dummy-data`.

### G. Glossary

| Term | Definition |
|---|---|
| **AAP** | Agent Action Plan — the primary directive defining project scope |
| **Canonical entry point** | The real, user-facing path (HTTP route, SMTP controller, job loop) — as opposed to a mock or debug hook |
| **`NOT_SEND_EMAIL`** | Default local flag causing emails to be logged rather than transmitted |
| **`ready→taken→done`** | The `Job.state` lifecycle observed in `job_runner.py` |
| **`simplelogin_sync_events`** | Postgres `LISTEN/NOTIFY` channel used as the inter-process event substrate |
| **Observed-negative** | An evidence class where the correct behavior is the *absence* of a signal (e.g., default event no-op) |
