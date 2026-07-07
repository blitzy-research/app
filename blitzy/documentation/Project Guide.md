# Blitzy Project Guide — Runtime Verification of a Self-Hosted SimpleLogin Instance

> **Deliverable branch:** `blitzy-0ff7facd-952c-4d6b-a686-a283063f23f4` · **Base commit:** `2cd6ee77` · **HEAD:** `9fe5d7a0`
> **Task type:** Read-only runtime-verification **documentation (Q&A)** — the sole deliverable is one evidence-grounded Markdown answer document; **no source code was to be, or was, changed.**
> **Brand color key:** <span style="color:#5B39F3">■ Completed / AI Work = Dark Blue `#5B39F3`</span> · ▢ Remaining = White `#FFFFFF` · <span style="color:#B23AF2">Headings/Accents = Violet-Black `#B23AF2`</span> · <span style="color:#A8FDD9">Highlight = Mint `#A8FDD9`</span>

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a single, evidence-grounded runtime-verification document proving that a freshly self-hosted **SimpleLogin** instance (a Python/Flask email-alias platform) operates correctly. The operator — a first-time self-hoster — asked three questions: what startup/UI signals confirm readiness (Q1); what a new user experiences through register → verify → login → dashboard (Q2); and what behind-the-scenes services support email forwarding and identity verification (Q3). The work brings up the full multi-process topology (webapp, SMTP handler, job runner, PostgreSQL, Redis), observes real behavior, and authors `blitzy/documentation/app_2cd6ee777f8c.md` with actual unedited runtime output and `file:line` citations for every claim. It is a read-only investigation: exactly one artifact is created and nothing in the source tree is modified.

### 1.2 Completion Status

The completion percentage is computed with the PA1 hours-based methodology over **AAP-scoped work only**: `Completed ÷ (Completed + Remaining) × 100 = 38 ÷ 41 = 92.7%`.

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieTitleTextSize':'16px','pieSectionTextColor':'#B23AF2'}}}%%
pie showData title Completion Status — 92.7% Complete
    "Completed Work (AI)" : 38
    "Remaining Work" : 3
```

| Metric | Hours |
|---|---|
| **Total Hours** | **41** |
| Completed Hours (AI + Manual) | 38 |
| &nbsp;&nbsp;• AI (Blitzy autonomous) | 38 |
| &nbsp;&nbsp;• Manual (human) | 0 |
| **Remaining Hours** | **3** |
| **Percent Complete** | **92.7%** |

### 1.3 Key Accomplishments

- ✅ Sole deliverable created and committed: `blitzy/documentation/app_2cd6ee777f8c.md` (825 lines / 88,387 bytes) across 5 commits.
- ✅ Full multi-process topology brought up and observed live: webapp (`server.py` :7777), SMTP forwarder (`email_handler.py` :20381), job runner (`job_runner.py`, 10 s poll), plus `cron.py -j` and `event_listener.py`, over PostgreSQL 15 + Redis 7.
- ✅ **Q1** answered with the `>>> init logging <<<` banner, `SL` logger identity/format/DEBUG, `:7777` bind, SMTP `Listen for port 20381`, and a **measured** ~10.017 s worker cadence stable across ≥2 runs.
- ✅ **Q2** answered end-to-end through real HTTP routes: register → activation email → activate (`users.activated` flips **False→True**, code deleted) → login → dashboard, with before/intermediate/after state tables.
- ✅ **Q3** answered: forwarding chain `Contact → Alias → Mailbox` (with an honest negative result for a disabled alias), background-job dispatch enumerated by name, a 17-job cron inventory, and a clean distinction between the two "event" subsystems — correctly attributing identity verification to the **synchronous activation path**.
- ✅ 8 edge/error paths exercised (7 observed + 1 correctly inferred) with before/after states.
- ✅ **179/179** `file:line` citations verified (exist + in-range + content-verified); all inferences and non-canonical aids explicitly labeled.
- ✅ Read-only guarantee upheld: **zero** source/template/config/migration changes; all temporary users/aliases/contacts/scripts removed; database restored to the exact seeded baseline.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|---|---|---|---|
| _None identified._ The Final Validator reproduced every documented claim live and reported zero remaining issues; the deliverable is complete, accurate, and committed. | — | — | — |

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|---|---|---|---|---|
| Deliverable repository (branch `blitzy-0ff7facd-…`) | Git write | None — the document was committed and the working tree is clean. | ✅ Resolved / No issue | — |
| Outbound DNS egress (canonical container) | Network egress | A **transient** DNS hang was diagnosed during observation (egress to `NAMESERVERS=['1.1.1.1']`); it recovered and did not affect the final deliverable. | ✅ Recovered (historical) | Human (reproduction only) |
| `google-re2` vs pinned `pyre2 0.3.6` | Python dependency | The shipped image installed `google-re2` (no module-level `DOTALL`) instead of the lockfile-pinned `pyre2 0.3.6`, blocking `email_handler.py` import until worked around. | ✅ Worked around read-only (PYTHONPATH shim outside `/app`); permanent fix = `pip install pyre2==0.3.6` | Human (reproduction only) |

> These are **environment-reproduction** notes, not blocking access issues for the delivered artifact. The document itself was produced and committed without impediment.

### 1.6 Recommended Next Steps

1. **[High]** Read `blitzy/documentation/app_2cd6ee777f8c.md` end-to-end and confirm it fully answers Q1 (readiness), Q2 (register→verify→login→dashboard), and Q3 (forwarding/jobs/scheduler/events) to your satisfaction. _(1 h)_
2. **[High]** Spot-check 3–5 `file:line` citations against the source at pinned commit `2cd6ee77`, and optionally re-run 1–2 quick observations (login page HTTP 200; job_runner ~10 s cadence) to confirm reproducibility in your environment. _(1 h)_
3. **[Low]** Accept/merge the PR, distribute the verification report to stakeholders, and archive it — noting the pinned commit so it can be re-verified after any upstream source bump. _(1 h)_
4. **[Low — out of scope]** _(No hours; context only.)_ If moving toward production self-hosting, plan a separate initiative for email hardening (DNS/MX, DKIM, SPF/DMARC, SSL/TLS, Postfix) and end-to-end verification of external integrations (New Relic, Proton webhook, real MTA, hCaptcha) — explicitly out of scope of this read-only verification per AAP §0.5.2.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

Each component traces to a specific AAP requirement group (see the classification in Section 5). All work was performed autonomously by Blitzy agents.

| Component | Hours | Description |
|---|---:|---|
| Runtime environment bring-up & multi-process topology orchestration | 5 | Derive `.env` from `example.env` (keep `NOT_SEND_EMAIL=true`), start PostgreSQL + Redis, `alembic upgrade head`, `flask dummy-data` seed, launch 5 processes; handle `re2` shim + transient DNS caveats (AAP §0.5.1 bring-up). |
| Q1 — Startup & readiness investigation + write-up (§2.1–2.6) | 4 | Capture logging banner/`SL` logger/format/DEBUG, build stamp, `:7777` bind + init routines (with 2 honest corrections), SMTP `Listen for port 20381`, measured 10.017 s cadence across ≥2 runs, login-page HTTP 200. |
| Q2 — New-user journey investigation + write-up (§3.1–3.6) | 5 | Drive register → activate → login → dashboard through **real** HTTP routes; capture flash messages, 302 redirects, and `users.activated` / `activation_code` before/intermediate/after state transitions. |
| Q3 — Behind-the-scenes services investigation + write-up (§4.1–4.4) | 7 | Full SMTP forward chain `Contact→Alias→Mailbox` + negative disabled-alias result; enumerate `process_job()` dispatch by name + observe onboarding; grep-verify 17-job cron inventory + run `sanity_check` live; distinguish New Relic analytics vs PostgreSQL Proton-sync and attribute identity verification to the synchronous activation path. |
| Edge/error path investigation (§5.1–5.8) | 5 | 8 cases: duplicate email, bad-mailbox, invalid code, forced-expiry code, wrong password, login-before-activation+resend, rate limiting (single + multi-worker variance), inferred hCaptcha — all with before/after states. |
| Answer-document authoring & structure (825 lines) | 5 | Compose the 7-section runtime-verification report with 44 unedited output blocks, an answer-at-a-glance, and a coverage checklist (AAP §0.4.2). |
| Citation grounding & verification (179 `file:line` refs) | 3 | Ground every claim in a `file:line` reference across ~30 files; label all inferences and non-canonical aids. |
| Cleanup, read-only verification & QA iteration | 4 | FK-safe deletion of temporary data, DB restore to seed baseline, script removal, `git status` verification; iterate across the 5-commit QA review cycle. |
| **Total Completed** | **38** | |

### 2.2 Remaining Work Detail

Because the deliverable is a validated documentation artifact, the only remaining work is the human review-and-acceptance path to production (per RG2, a document is never auto-100% before human sign-off).

| Category | Hours | Priority |
|---|---:|---|
| Human SME technical review & sign-off of the verification document | 2 | High |
| Stakeholder acceptance, distribution & archival | 1 | Low |
| **Total Remaining** | **3** | |

### 2.3 Hours Reconciliation

- **Completed (2.1) = 38 h** · **Remaining (2.2) = 3 h** · **Total = 41 h**.
- Completion % = `38 ÷ 41 × 100 = 92.7%` — identical to Section 1.2 and Section 7.
- Section 1.6 human tasks (1 + 1 + 1 = 3 h) reconcile exactly with Section 2.2 (3 h).

---

## 3. Test Results

For this task, "tests" are Blitzy's **autonomous runtime-reproduction validations** — every documented claim was re-run live against the real entry points (HTTP routes and the SMTP listener) and its unedited output re-observed. These are the applicable validations because the in-scope deliverable is a Markdown document with no unit-test suite.

| Test Category | Framework / Method | Total | Passed | Failed | Coverage % | Notes |
|---|---|---:|---:|---:|---:|---|
| Q1 — Startup/readiness reproduction | Live topology + stdout capture | 6 | 6 | 0 | 100% | §2.1–2.6: banner, logger, build stamp, bind, SMTP listen, 10 s cadence |
| Q2 — New-user flow reproduction | Live HTTP (real `/auth/*` + `/dashboard/`) | 4 | 4 | 0 | 100% | register → activate (False→True) → login → dashboard |
| Q3 — Behind-the-scenes reproduction | Live SMTP :20381 + worker + `cron -j` + listener | 4 | 4 | 0 | 100% | forwarding chain, jobs, scheduler, event subsystems |
| Edge/error paths | Live HTTP + SMTP | 8 | 8 | 0 | 100% | 7 observed + 1 correctly inferred (hCaptcha; `HCAPTCHA_SECRET=None`) |
| Citation audit | `file:line` resolver (exist/in-range/content) | 179 | 179 | 0 | 100% | across ~30 files; 0 drift/misattribution |
| Read-only + cleanup verification | `git` + `psql` | 2 | 2 | 0 | 100% | byte-clean trees; DB restored to seed baseline |
| **Total** | | **203** | **203** | **0** | **100%** | All reproduced-or-correctly-inferred |

> **Integrity note (Rule 3):** every row above originates from Blitzy's autonomous validation logs for this project. The repository's own `pytest` suite is **out of scope** (the AAP scopes this as read-only observation producing one document; no source changed, so the baseline test state is unaffected) and was intentionally **not run** — no fabricated unit-test numbers are reported.

---

## 4. Runtime Validation & UI Verification

**Runtime health (topology) — all ✅ Operational:**

- ✅ **Webapp** (`server.py`, `:7777`) — bound and serving; `/` 302-redirects to `/auth/login`, which returns HTTP **200**.
- ✅ **SMTP forwarder** (`email_handler.py`, `:20381`) — logged `Listen for port 20381` / `Start mail controller 0.0.0.0 20381`.
- ✅ **Job runner** (`job_runner.py`) — entered `while True` poll loop; measured cadence **10.017 s/cycle** (stable across ≥2 runs).
- ✅ **Cron scheduler** (`cron.py -j sanity_check`) — ran through its real `-j` entry point, exit code **0**.
- ✅ **Event listener** (`event_listener.py`) — started with `Using PostgresEventSource` / `Starting with HttpEventSink` / `Starting to listen to events`.
- ✅ **Data stores** — PostgreSQL 15.13 + Redis 7.0.15 reachable; state transitions queried directly.

**UI verification:**

- ✅ **Login page** (`templates/auth/login.html`) — renders at HTTP 200; demo login `john@wick.com` / `password` → dashboard.
- ✅ **Registration → waiting → activate → login → dashboard** surfaces all render as documented.
- ⚠ **`register_waiting_activation.html`** — emits a **non-fatal** JS `ReferenceError: plausible is not defined` (Plausible analytics global exists only in production); the page still renders and the flow is unaffected. Reported (not fixed) under the read-only constraint.
- ⚠ **Accessibility advisories** (DevTools) on auth forms — missing `<label for>` associations / `autocomplete`. Advisories, not JS errors; reported (not fixed) under read-only scope.

**API / integration outcomes:**

- ✅ **Email forwarding** — a real message to `:20381` forwarded `Contact → Alias → Mailbox`, `Finish … 250 Message accepted`; printed (not sent) because `NOT_SEND_EMAIL=true`.
- ⚠ **External integrations by design** — New Relic (`enabled=False`, no-op), Proton-sync (webhook guard fired, `sync_event=0`), real MTA/Postfix and hCaptcha are unconfigured locally. Correctly observed as no-op/guarded — **not** end-to-end verified (out of scope).

---

## 5. Compliance & Quality Review

AAP deliverables and the explicit "SWE-AtlasQnA-Repo" rule set (§0.7) cross-mapped to Blitzy's quality benchmarks. All items pass; fixes applied during autonomous validation are noted.

| AAP / Rule Benchmark | Requirement | Status | Progress | Notes / Fixes Applied |
|---|---|---|---|---|
| Deliverable (§0.4.2) | Create `blitzy/documentation/app_2cd6ee777f8c.md` + parent dir | ✅ Pass | 100% | 825 lines, committed (HEAD `9fe5d7a0`) |
| Run-first methodology (§0.7) | Observe by running before writing | ✅ Pass | 100% | All claims from live runs; 44 unedited output blocks |
| Unedited output (§0.7) | Actual output for every claim | ✅ Pass | 100% | Real log lines w/ timestamps, PIDs, `message_id` UUIDs |
| Grounded citations (§0.7) | `file:line` for every claim | ✅ Pass | 100% | 179/179 resolve (exist + in-range + content-verified) |
| Label inferences (§0.7) | Tag non-observed/non-canonical | ✅ Pass | 100% | 6 inference tags; forced-expiry & probe jobs labeled NON-CANONICAL |
| Q1 coverage | Startup/readiness (logs + UI) | ✅ Pass | 100% | §2.1–2.6 |
| Q2 coverage | register→verify→login→dashboard | ✅ Pass | 100% | §3.1–3.6 |
| Q3 coverage | forwarding/jobs/scheduler/events + identity verification | ✅ Pass | 100% | §4.1–4.4; two event subsystems distinguished |
| Edge paths (§0.7) | Exercise every condition | ✅ Pass | 100% | 8 cases §5.1–5.8 |
| Before/during/after (§0.7) | State transitions reported | ✅ Pass | 100% | §3.5 table; §5 per-case |
| Answer every named item (§0.7) | Final coverage pass | ✅ Pass | 100% | §7 coverage checklist |
| Read-only scope (§0.5.2, §0.7) | No source/template/config/migration change | ✅ Pass | 100% | `git diff 2cd6ee77 --name-status` = only the doc |
| Cleanup (§0.8.1) | Remove temp data/scripts; clean `git status` | ✅ Pass | 100% | FK-safe deletion; DB restored to seed baseline |
| Honest corrections | Report expectation mismatches | ✅ Pass | 100% | Werkzeug line suppressed; `add_sl_domains` not on webapp startup; §2.6 HTTP/1.0 (CP6 F1 fix) |

**Autonomous QA fixes applied during validation (5-commit arc):** initial report → resolve 6 review findings with runtime evidence → fix 2 Q1 citation findings (CP1) → address QA findings → fix CP6 F1 (§2.6 quotes HTTP/1.0 for the `python server.py` dev server).

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|---|---|---|---|---|---|
| Non-canonical observation aids (§5.4 forced expiry; §2.5 probe jobs) | Technical | Low | Low | Honestly labeled NON-CANONICAL in-doc; canonical reproduction available | Mitigated / Documented |
| hCaptcha edge (§5.8) is inferred, not observed (`HCAPTCHA_SECRET=None`) | Technical | Low | Low | Labeled "inferred from reading"; enable hCaptcha to observe | Mitigated / Documented |
| Point-in-time snapshot pinned to `2cd6ee77`; citations could drift if source changes | Technical | Low | Low | Commit pinned in-doc; re-verify after upstream bumps | Accepted |
| Local `.env` holds secrets on disk | Security | Low | Low | `.env` is git-ignored (not committed); ephemeral runtime artifact | Mitigated |
| Local config (`NOT_SEND_EMAIL`, DMARC disabled) mistaken for production-ready | Security | Medium | Low | Doc scopes local-only; prod hardening (DKIM/SSL/DNS-MX/SPF/DMARC) out of scope per §0.5.2 | Flagged / Out of scope |
| Canonical-environment dependency (Python 3.10 / PG15 / Redis7 container) | Operational | Low | Medium | Exact image + versions stated; recipe reproducible; diffs are cosmetic (timestamps/PIDs) | Documented |
| `re2`/DNS runtime caveats on reproduction | Operational | Low-Med | Medium | Doc + dev guide note `pip install pyre2==0.3.6` (or read-only shim) and DNS egress requirement | Documented / Workaround known |
| External integrations disabled locally (New Relic, Proton, MTA, hCaptcha) | Integration | Low | Low | Doc distinguishes local no-op/guarded behavior; e2e verification is separate future work | Out of scope / Documented |

**Summary:** all risks are **Low** or **Low-Medium**; the majority are already mitigated or explicitly documented by the deliverable itself — appropriate for a validated, read-only documentation task.

---

## 7. Visual Project Status

**Project hours breakdown** (Completed = Dark Blue `#5B39F3`, Remaining = White `#FFFFFF`). The "Remaining Work" value (3 h) equals Section 1.2 Remaining Hours and the sum of Section 2.2:

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieSectionTextColor':'#B23AF2'}}}%%
pie showData title Project Hours Breakdown (Total 41h)
    "Completed Work" : 38
    "Remaining Work" : 3
```

**Remaining hours by category (Section 2.2):**

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#A8FDD9','pieStrokeColor':'#B23AF2','pieSectionTextColor':'#B23AF2'}}}%%
pie showData title Remaining Work by Category (3h)
    "SME review & sign-off (High)" : 2
    "Stakeholder acceptance/distribution (Low)" : 1
```

| Priority | Remaining Hours | Share |
|---|---:|---:|
| High | 2 | 66.7% |
| Low | 1 | 33.3% |
| **Total** | **3** | **100%** |

---

## 8. Summary & Recommendations

**Achievements.** The project is **92.7% complete** on an AAP-scoped basis (38 of 41 hours). Every AAP requirement was delivered and independently validated: the single answer document `blitzy/documentation/app_2cd6ee777f8c.md` comprehensively answers Q1 (readiness), Q2 (the new-user journey), and Q3 (behind-the-scenes forwarding, background jobs, the cron scheduler, and the two event subsystems), each backed by unedited runtime output and `file:line` citations. The full multi-process topology was run live, all 8 edge/error paths were exercised, 179/179 citations were verified, and the read-only + cleanup guarantees were confirmed byte-clean.

**Remaining gaps (3 h).** The only outstanding work is the human review-and-acceptance path for a documentation deliverable: a SME technical review and sign-off (2 h) and stakeholder acceptance/distribution (1 h). There are **no** unresolved code, configuration, or bug-fix tasks.

**Critical path to production.** (1) SME reads the document and confirms coverage → (2) spot-check citations / optionally re-run a couple of observations → (3) accept/merge and distribute. Because the deliverable is a Markdown report, "production" means "reviewed, accepted, and archived."

**Success metrics.** ✅ One artifact created, nothing else changed (read-only intact); ✅ every question and named item answered (coverage checklist §7); ✅ 100% of documented runtime claims reproduced or correctly inferred; ✅ 179/179 citations resolve.

**Production readiness assessment.** The deliverable is **production-ready pending human sign-off**. Confidence is **High**: the scope is well-defined, the validator reproduced every claim live, and the read-only guarantee is verifiable. Per RG2, completion is capped below 100% until a human accepts the document — hence 92.7%. Note explicitly that production **self-hosting** hardening and external-integration end-to-end verification are **out of scope** of this AAP and are not reflected in the hour totals.

---

## 9. Development Guide

> The deliverable is a Markdown document. This guide covers **(A) consuming the document** (trivial) and **(B) reproducing the runtime observations** (bring up the multi-process topology). All commands are grounded in `CONTRIBUTING.md`, `example.env`, the `Dockerfile`, `scripts/reset_local_db.sh`, and the deliverable's own §1 recipe.

### 9.1 System Prerequisites

- **Canonical environment (recommended):** Docker image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` — Python **3.10.18**, PostgreSQL **15.13**, Redis **7.0.15**.
- **Manual environment:** Python `^3.10` (`pyproject.toml:61`), PostgreSQL 13+, Redis on `:6379`, Node.js `10.17.0` (front-end asset build), Poetry.
- **To only read the report:** any Markdown viewer — no build required.

### 9.2 Consume the Deliverable (fastest path)

```bash
# From the repository root
less blitzy/documentation/app_2cd6ee777f8c.md      # or open in any Markdown viewer

# Prove the read-only guarantee (both commands are copy-pasteable)
git status --porcelain -uall                         # expect: EMPTY (clean tree)
git diff 2cd6ee77 --name-status                      # expect exactly: A  blitzy/documentation/app_2cd6ee777f8c.md
```

### 9.3 Environment Setup (to reproduce observations)

```bash
# 1) Create local config from the shipped default (byte-identical to example.env)
cp example.env .env                                  # .env is git-ignored (.gitignore L4,L17)

# 2) Keep NOT_SEND_EMAIL=true so activation/forwarded emails PRINT to the log (evidence lever).
#    NOTE: NOT_SEND_EMAIL is a PRESENCE check (app/config.py:91) — even "=false" enables print mode;
#    to actually transmit mail you must REMOVE the line entirely.

# 3) Align DB_URI to your Postgres port. example.env ships :5432 (L75);
#    CONTRIBUTING.md / scripts/reset_local_db.sh use :15432 / :35432 (docker -p 15432:5432).
grep -nE '^(URL|NOT_SEND_EMAIL|EMAIL_DOMAIN|DISABLE_ONBOARDING|DB_URI)=' .env
```

### 9.4 Dependency Installation

```bash
# Front-end assets
cd static && npm install && cd ..

# Python dependencies (canonical: Poetry; 182 packages)
poetry install

# re2 caveat: if `google-re2` is installed instead of the lockfile-pinned `pyre2 0.3.6`,
# `python email_handler.py` aborts at import with:
#   AttributeError: module 're2' has no attribute 'DOTALL'  (app/spamassassin_utils.py:13)
# Permanent, lockfile-consistent remedy:
pip install pyre2==0.3.6
```

### 9.5 Data Stores, Migrations & Seed

```bash
# PostgreSQL (canonical CONTRIBUTING.md recipe)
docker run -e POSTGRES_PASSWORD=mypassword -e POSTGRES_USER=myuser \
  -e POSTGRES_DB=simplelogin -p 15432:5432 postgres:13

# Redis on :6379 (any standard instance)

# Apply migrations and seed the demo dataset (creates john@wick.com / password)
alembic upgrade head
flask dummy-data
```

### 9.6 Application Startup (multi-process topology)

```bash
# Developer webapp -> app.run(debug=True, port=7777)  (server.py:588)
CONFIG=/app/.env python server.py            &     # HTTP :7777

# SMTP forwarder (defaults --port 20381)             (email_handler.py:2399)
CONFIG=/app/.env python email_handler.py     &     # SMTP :20381

# Background worker (10-second poll loop)
CONFIG=/app/.env python job_runner.py        &

# Optional companions
CONFIG=/app/.env python event_listener.py listener &
CONFIG=/app/.env python cron.py -j sanity_check

# Production webapp (Dockerfile CMD; EXPOSE 7777)
# gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
```

### 9.7 Verification Steps

```bash
# Webapp readiness — expect HTTP 200
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:7777/auth/login

# Expected log signals:
#   webapp:        >>> init logging <<<   and the SL logger emitting DEBUG lines
#   email_handler: Listen for port 20381  /  Start mail controller 0.0.0.0 20381
#   job_runner:    Take job <...>  then  time.sleep(10)  (~10 s between cycles)

# UI: open http://localhost:7777 and log in with john@wick.com / password  -> dashboard
```

### 9.8 Example Usage (reproduce Q2 & Q3)

```bash
# Q2 — drive the new-user flow through the REAL routes:
#   POST /auth/register  -> activation email printed to the webapp log ("Just one more step to join SimpleLogin")
#   GET  /auth/activate?code=...  -> users.activated flips False->True, code deleted, 302 -> dashboard.index
#   POST /auth/login     -> 302 -> dashboard (HTTP 200)

# Q3 — send a test message to the SMTP listener and watch forwarding:
python - <<'PY'
import smtplib
from email.mime.text import MIMEText
m = MIMEText("probe"); m["From"]="sender@external.test"; m["To"]="e1@sl.local"; m["Subject"]="probe"
s = smtplib.SMTP("127.0.0.1", 20381); print(s.sendmail(m["From"], [m["To"]], m.as_string())); s.quit()
PY
# email_handler log shows: New message ... -> Forward <Contact> -> <Alias> -> <Mailbox> -> Finish ... 250 Message accepted
```

### 9.9 Troubleshooting

- **`AttributeError: module 're2' has no attribute 'DOTALL'`** → install the lockfile-pinned package: `pip install pyre2==0.3.6` (or set a read-only `PYTHONPATH` shim `sys.modules['re2']=re` outside the source tree).
- **DNS hang on startup** → ensure outbound DNS egress is available (the app resolves via `NAMESERVERS`, e.g. `1.1.1.1`).
- **DB connection refused** → align `.env`'s `DB_URI` port with your Postgres exposed port (`:5432` vs `:15432`/`:35432`).
- **Emails not printing to the log** → confirm `NOT_SEND_EMAIL` is present in `.env` (it is a presence check; the value is irrelevant).
- **No onboarding jobs at registration** → expected: `example.env` ships `DISABLE_ONBOARDING=true` (`app/config.py:401`), which disables onboarding-job scheduling by default.

---

## 10. Appendices

### Appendix A — Command Reference

| Command | Purpose |
|---|---|
| `git status --porcelain -uall` | Verify clean working tree (read-only guarantee) |
| `git diff 2cd6ee77 --name-status` | Confirm only the answer document was added |
| `cp example.env .env` | Create local config from shipped default |
| `alembic upgrade head` | Apply database migrations |
| `flask dummy-data` | Seed demo dataset (`john@wick.com` / `password`) |
| `CONFIG=/app/.env python server.py` | Start webapp on `:7777` (dev) |
| `CONFIG=/app/.env python email_handler.py` | Start SMTP forwarder on `:20381` |
| `CONFIG=/app/.env python job_runner.py` | Start 10 s background worker |
| `CONFIG=/app/.env python cron.py -j <job>` | Run a scheduled job on demand |
| `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` | Production webapp |
| `curl -s -o /dev/null -w "%{http_code}\n" http://localhost:7777/auth/login` | Check webapp readiness (expect 200) |
| `pip install pyre2==0.3.6` | Fix the `re2`/`DOTALL` import error |

### Appendix B — Port Reference

| Port | Service | Source |
|---|---|---|
| 7777 | Webapp (HTTP) | `server.py` (`app.run(port=7777)`), `Dockerfile:44 EXPOSE 7777` |
| 20381 | SMTP forwarder | `email_handler.py:2399` (`--port` default) |
| 5432 / 15432 / 35432 | PostgreSQL | `example.env:75` (`:5432`); `CONTRIBUTING.md` / `scripts/reset_local_db.sh` (`:15432`/`:35432`) |
| 6379 | Redis | `scripts/run-test.sh` |

### Appendix C — Key File Locations

| Path | Role |
|---|---|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The sole deliverable** — runtime-verification answer document |
| `server.py` / `wsgi.py` | Webapp entry points |
| `email_handler.py` | SMTP forwarding pipeline |
| `job_runner.py` | 10 s background worker |
| `cron.py` / `crontab.yml` / `crontab-all-hosts.yml` | Scheduler + yacron schedules |
| `event_listener.py` | Proton-sync event consumer |
| `app/log.py` | Shared `SL` logger, format, `>>> init logging <<<` banner |
| `app/auth/views/{register,activate,login,login_utils}.py` | Q2 auth flow |
| `app/email_utils.py` / `app/mail_sender.py` | Email dispatch + `NOT_SEND_EMAIL` print branch |
| `app/config.py` | `NOT_SEND_EMAIL`, `EMAIL_DOMAIN`, `DISABLE_ONBOARDING`, ports |
| `example.env` / `CONTRIBUTING.md` | Canonical local config + run recipe |

### Appendix D — Technology Versions

| Component | Version | Source |
|---|---|---|
| Python | `^3.10` (canonical 3.10.18) | `pyproject.toml:61` |
| Flask | `^1.1.2` | `pyproject.toml` |
| SQLAlchemy | `1.3.24` | `pyproject.toml` |
| aiosmtpd | `^1.2` (runtime 1.4.2) | `pyproject.toml` |
| gunicorn | `^20.0.4` | `pyproject.toml` |
| PostgreSQL | 13+ (canonical 15.13) | `CONTRIBUTING.md` / container |
| Redis | `^4.5.3` (server 7.0.15) | `pyproject.toml` / container |
| Node.js | `10.17.0` | `Dockerfile` |
| newrelic | `8.8.0` | `pyproject.toml` |

### Appendix E — Environment Variable Reference

| Variable | Default (`example.env`) | Effect |
|---|---|---|
| `URL` | `http://localhost:7777` | Base URL / webapp port |
| `NOT_SEND_EMAIL` | `true` (presence check, `app/config.py:91`) | Print emails to log instead of sending |
| `EMAIL_DOMAIN` | `sl.local` (`app/config.py:92`) | Alias domain |
| `DISABLE_ONBOARDING` | `true` (presence check, `app/config.py:401`) | Disables onboarding-job scheduling at registration |
| `DB_URI` | `postgresql://myuser:mypassword@localhost:5432/simplelogin` | PostgreSQL connection |
| `CONFIG` | `/app/.env` | Path to the config file each process loads |
| `COLOR_LOG` | (optional) `true` | Colored console logs |

### Appendix F — Developer Tools Guide

- **Evidence capture:** redirect each process's stdout/stderr to a log file; correlate an email's lifecycle across lines via the injected `message_id` UUID (`app/log.py:28-37`).
- **Database inspection:** `psql` for `users.activated`, `activation_code`, `job`, `sync_event`, `email_log` state before/after each step.
- **SMTP probing:** Python `smtplib` (or `swaks` where available) against `127.0.0.1:20381`.
- **Read-only enforcement:** run `git status --porcelain -uall` after every session; keep all temporary scripts in an untracked scratch dir outside the source tree and delete them afterward.

### Appendix G — Glossary

| Term | Meaning |
|---|---|
| **AAP** | Agent Action Plan — the authoritative scope for this task |
| **Activation code** | One-time code emailed at registration; consumed by `GET /auth/activate` to set `users.activated=true` |
| **Reverse alias** | Rewritten sender contact address so a mailbox owner can reply through SimpleLogin |
| **`NOT_SEND_EMAIL`** | Presence-checked flag that makes `MailSender.send()` print emails to the log |
| **yacron** | YAML-configured cron runner that schedules a subset of `cron.py -j <job>` jobs |
| **Proton-sync events** | PostgreSQL `SyncEvent` rows consumed by `event_listener.py` (partner integration) — distinct from New Relic analytics events |
| **Non-canonical aid** | An observation shortcut (e.g., forced code expiry) explicitly labeled because it bypasses the pure canonical path |

---

*Generated by the Blitzy Platform — AAP-scoped completion analysis. All hour figures, completion percentages, and test counts are internally consistent across Sections 1.2, 2, 3, 7, and 8.*