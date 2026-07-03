# Blitzy Project Guide — SimpleLogin First-Time Operator Runtime Q&A

> **Deliverable:** `blitzy/documentation/app_2cd6ee777f8c.md` — a single, evidence-grounded Q&A documentation artifact for a self-hosted SimpleLogin instance.
> **Task type:** Read-only investigation → additive documentation (zero source-code changes).

---

## 1. Executive Summary

### 1.1 Project Overview

This project produces one evidence-grounded Q&A documentation artifact answering a first-time operator's questions about a freshly launched, self-hosted **SimpleLogin** instance. Written entirely from **observed runtime output** (verbatim log lines, HTTP responses, measured timings) rather than code reading, it explains: (Q1) what signals confirm the app is ready for authentication and alias/email activity; (Q2) what a new user sees across register → verify → login → dashboard; and (Q3) how the background jobs and internal services operate and inter-communicate. Target users are self-hosting operators and integrators. The technical scope spans five cooperating processes — the Flask web app, the SMTP handler, the job runner, the cron scheduler, and the event listener — observed strictly read-only, yielding a single additive Markdown file.

### 1.2 Completion Status

The project is **90.3% complete**, calculated on AAP-scoped hours: **28 completed / 31 total** autonomous+path-to-production hours. All AAP deliverable work is complete, validated, and committed; the remaining 3 hours is human path-to-production (technical-accuracy review and editorial/stakeholder sign-off).

```mermaid
%%{init: {"theme":"base","themeVariables":{"pie1":"#5B39F3","pie2":"#FFFFFF","pieStrokeColor":"#B23AF2","pieStrokeWidth":"2px","pieOuterStrokeWidth":"2px","pieTitleTextSize":"18px","pieSectionTextSize":"15px","pieLegendTextSize":"14px"}}}%%
pie showData title Completion — 90.3% (hours)
    "Completed Work (AI)" : 28
    "Remaining Work" : 3
```

| Metric | Hours |
|--------|-------|
| **Total Hours** | **31** |
| **Completed Hours (AI + Manual)** | **28** (28 AI autonomous + 0 Manual to date) |
| **Remaining Hours** | **3** |
| **Percent Complete** | **90.3%** |

> Color key — **Completed = Dark Blue `#5B39F3`**, **Remaining = White `#FFFFFF`**.

### 1.3 Key Accomplishments

- ✅ Single in-scope deliverable created and committed: `blitzy/documentation/app_2cd6ee777f8c.md` (653 lines, ~68 KB).
- ✅ **Q1 (Readiness)** answered with verbatim startup evidence for all five process entry points, the `GET /health → "success", 200` probe, and the UI landing state.
- ✅ **Q2 (New-user flow)** answered by exercising register → verify → login → dashboard through the **real HTTP entry points** on port 7777, with status codes, redirects, flashes, DB effects, and negative/edge branches.
- ✅ **Q3 (Behind-the-scenes)** answered with the job-runner 10-second cadence (measured across 5 cycles), the full SMTP forwarding lifecycle under one message id, and the `SyncEvent → NOTIFY → listener → HttpEventSink` pipeline.
- ✅ Evidence discipline upheld — one claim → one adjacent verbatim line; inferred/non-canonical/non-default values explicitly labeled.
- ✅ **27 cited source files / 88 `file:line` citations** — all resolve at HEAD; literal content independently spot-checked at 100% accuracy.
- ✅ Two decisive default-config flags disclosed (`NOT_SEND_EMAIL=true`, `DISABLE_ONBOARDING=true`) plus the `EVENT_WEBHOOK`-unset behavior.
- ✅ Coverage pass over every named item; all temporary artifacts cleaned up; repository left byte-for-byte unchanged apart from the deliverable.

### 1.4 Critical Unresolved Issues

There are **no critical unresolved issues** blocking release of this documentation deliverable. All AAP requirements are complete, validated, and committed. The items below are honest **disclosures about the software's shipped defaults** that the document already surfaces — they are not defects introduced by this work and do not block the deliverable.

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| Activation link not present in default logs (`NOT_SEND_EMAIL=true`) | Documentation-only; already labeled and captured via a non-default MTA | Documented (no action needed) | N/A — accurately reported |
| Canonical alias-creation event pipeline dormant/gated by default (`EVENT_WEBHOOK` unset + partner-user gate) | Documentation-only; transport proven via a labeled non-canonical trigger | Documented (no action needed) | N/A — accurately reported |
| Benign browser-console `ReferenceError: plausible is not defined` on the register waiting page | Cosmetic; non-blocking; out of scope for read-only task | Deferred to a future non-read-only task (optional) | N/A — reported, not fixed |

### 1.5 Access Issues

**No access issues identified.** The investigation ran entirely inside the provided canonical Docker runtime, which had PostgreSQL, Redis, the seeded dummy data, and all dependencies pre-provisioned. No external repository permissions, service credentials, or third-party API access were required or blocked.

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-----------------|----------------|-------------------|-------------------|-------|
| Canonical Docker runtime | Container execution | None — pre-provisioned and healthy | ✅ Resolved (no issue) | Blitzy |
| PostgreSQL / Redis | Local service | None — pre-provisioned and seeded | ✅ Resolved (no issue) | Blitzy |
| Host git repository | Read/write | None — clean, deliverable committed | ✅ Resolved (no issue) | Blitzy |

### 1.6 Recommended Next Steps

1. **[High]** Perform a human technical-accuracy review: re-run a sample of the documented observations on a fresh canonical instance (`/health`, register→verify→login→dashboard, job cadence, one SMTP forward) and spot-check a sample of the 88 citations. *(~1.5 h)*
2. **[High]** Confirm the coverage pass: verify every named item in the original Q1/Q2/Q3 prompt is addressed, including the two honestly-⚠ items. *(~0.5 h)*
3. **[Medium]** Editorial/readability review: proofread and confirm Markdown renders correctly in the target viewer (tables, fenced blocks, ⚠/✅ markers). *(~0.5 h)*
4. **[Medium]** Stakeholder sign-off and merge acceptance of the deliverable. *(~0.5 h)*
5. **[Low]** *(Optional, out-of-AAP-scope follow-ups — not counted in remaining hours)* Guard the unconditional `plausible()` call; document `EVENT_WEBHOOK`/partner-user requirements for integrators; configure an MTA for real email delivery.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

All completed work traces to specific AAP requirements (R1–R28). Total = **28 hours**.

| Component | Hours | Description |
|-----------|-------|-------------|
| Runtime foundation & environment provisioning | 3 | Start the five process entry points; verify PostgreSQL 15.13 / Redis 7.0.15 / seeded data; document run-environment (documented-vs-observed) and provisioning/migration/seed commands. [R1, R25] |
| Q1 — Readiness investigation & authoring | 4 | Capture verbatim startup of all 5 processes; `curl -i /health`; UI landing states; real yacron scheduled dispatch of `cron.py -j stats`; author 7 subsections + summary table. [R2–R8, R19, R20] |
| Q2 — New-user flow investigation & authoring | 6 | Exercise register→verify→login→dashboard via real HTTP; capture 3 labeled activation-link evidence levels (incl. a non-default MTA catcher + second web instance); verification & login edge branches; rate-limit multi-run analysis; author 4 steps + tables. [R9–R14, R18] |
| Q3 — Behind-the-scenes investigation & authoring | 6 | Measure job-runner cadence across 5 cycles via enqueued jobs; capture full SMTP forward lifecycle; investigate the event pipeline (discovering 2 gates) + prove transport via webhook receiver + extra listener; author 3 subsections + summary. [R15–R17] |
| Evidence discipline, `file:line` citations (27 files) & coverage pass | 3 | Place each observed line adjacent to its claim; ground 88 citations; label inferred/non-canonical/non-default; author the coverage-pass checklist. [R22–R24] |
| Cleanup & repository-hygiene verification | 2 | Delete temporary users/aliases/jobs/contacts/email-logs/SyncEvents (verified 0); stop temporary services; restore default config; confirm clean tree. [R27, R28] |
| QA refinement cycles (4 commits / 3 review rounds) | 4 | Address review + CP3 QA findings across iterative commits (~half the document rewritten during review). [R21–R24 iteration] |
| **Total Completed** | **28** | **Matches Completed Hours in Section 1.2** |

### 2.2 Remaining Work Detail

All remaining work is human-side path-to-production (no incomplete or broken autonomous work). Total = **3 hours**.

| Category | Hours | Priority |
|----------|-------|----------|
| Human technical-accuracy review & runtime re-verification (re-run sampled observations on a live instance; spot-check sampled citations at HEAD) | 2 | High |
| Editorial review & stakeholder sign-off / merge acceptance | 1 | Medium |
| **Total Remaining** | **3** | **Matches Remaining Hours in Section 1.2 and Section 7 pie** |

### 2.3 Total Project Hours

| Line | Hours |
|------|-------|
| Section 2.1 — Completed | 28 |
| Section 2.2 — Remaining | 3 |
| **Total Project Hours** | **31** |
| **Percent Complete** = 28 / 31 | **90.3%** |

---

## 3. Test Results

Per the AAP, the repository's own **pytest suite was intentionally NOT run as the source of evidence** (AAP §0.5.2 — "The existing test suite … is not run as the source of the answer's evidence"). For this read-only documentation deliverable, the applicable autonomous validation consists of the citation, runtime-reproduction, structural, coverage, and hygiene checks performed by Blitzy's autonomous validation systems (and independently re-verified during this assessment). **All checks below originate from Blitzy's autonomous validation logs.**

| Test Category | Framework / Method | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|--------------------|-------------|--------|--------|-----------|-------|
| Citation resolution (`file:line`) | `python3` regex + filesystem check | 88 | 88 | 0 | 100% | 27 distinct files; 0 missing, 0 out-of-range at HEAD |
| Citation literal-content match | Manual `grep`/`sed` vs source @ HEAD | 28 | 28 | 0 | 100% | Incl. verbatim `rctp tos` typo + trailing-space format string preserved |
| Q1 — Readiness runtime reproduction | Live process startup + `curl` | 7 | 7 | 0 | 100% | web, `/health`, UI landing, SMTP, job runner, event listener, cron |
| Q2 — New-user flow reproduction | Real HTTP entry points (`:7777`) | 11 | 11 | 0 | 100% | register, duplicate, activate(302), invalid(400), expired(400), login(302), wrong-pw, not-activated, dashboard, rate-limit ×2 |
| Q3 — Behind-the-scenes reproduction | Live runtime observation | 3 | 3 | 0 | 100% | job cadence (5 cycles), SMTP lifecycle (1 msg-id), event pipeline (3 levels) |
| Markdown structural validation | `grep`/`wc` | 4 | 4 | 0 | 100% | 62 balanced fences, 43 headings, 0 placeholders, consistent tables |
| Coverage-pass completeness | Checklist audit | 27 | 27 | 0 | 100% | Every named item addressed; 2 honestly labeled ⚠ (shipped default behavior, not failures) |
| Repo-hygiene & cleanup | `git` + `psql` | 6 | 6 | 0 | 100% | Clean tree; only deliverable added; DB baseline (users=2, alias=11, sync_event=0) |
| **TOTAL** | | **174** | **174** | **0** | **100%** | All from Blitzy autonomous validation logs |

---

## 4. Runtime Validation & UI Verification

All five process entry points were started and observed live through their canonical interfaces; the new-user journey was driven end-to-end through the real HTTP endpoints.

**Process / runtime health**
- ✅ **Operational** — Web app: Gunicorn 20.0.4, 2 workers, `Listening at: http://0.0.0.0:7777`; `>>> init logging <<<` banner emitted.
- ✅ **Operational** — Health probe: `GET /health → HTTP/1.1 200 OK`, body `success`, `Server: gunicorn/20.0.4`.
- ✅ **Operational** — SMTP handler: `Listen for port 20381` / `Start mail controller 0.0.0.0 20381`.
- ✅ **Operational** — Job runner: banner then stable ~10-second idle/poll loop.
- ✅ **Operational** — Event listener: `Using PostgresEventSource` / `Starting with HttpEventSink` / `Starting to listen to events`.
- ✅ **Operational** — Cron: real yacron dispatch of `cron.py -j stats`, exit code 0, "reporting success".

**UI & new-user journey**
- ✅ **Operational** — UI landing: `GET /auth/login → 200`, `GET / → 302` (redirect to login), `GET /auth/register → 200`.
- ✅ **Operational** — Register: `POST /auth/register → 200`, `create user …` logged, `Activation Email Sent` waiting page rendered; duplicate registration correctly flashes `Email … already used`.
- ✅ **Operational** — Verify: `GET /auth/activate?code=… → 302`, `Your account has been activated`, `redirect user to dashboard`.
- ✅ **Operational** — Login: `POST /auth/login → 302`, `log user … in`, `LoginEvent.success`.
- ✅ **Operational** — Dashboard: `GET /dashboard/ → 200`, `Show intro to <User …>`, `dashboard/index.html` rendered.

**Behind-the-scenes**
- ✅ **Operational** — Job cadence measured 10.018 / 10.025 / 10.020 / 10.016 s across 5 cycles; `process_job` name dispatch observed via `Unknown job name` fallback.
- ✅ **Operational** — SMTP forward lifecycle: `New message` → `Forward phase` → `Create <EmailLog>` → `Finish … return code '250 Message accepted for delivery'`.
- ✅ **Operational** — Event transport: `Got NOTIFY` → `Event … sent successfully to webhook` → `Marked … as done` (receiver logged `application/x-protobuf`).

**Partial / honestly labeled (shipped default behavior, not failures)**
- ⚠ **Partial** — Activation link is **not** in the default logs (`NOT_SEND_EMAIL=true` logs metadata only); real link captured via a labeled non-default MTA.
- ⚠ **Partial** — Canonical alias-creation event delivery is **dormant/gated** by default (`EVENT_WEBHOOK` unset; partner-user gate); transport proven via a labeled non-canonical trigger.
- ⚠ **Partial** — Register waiting page shows a benign console `ReferenceError: plausible is not defined` (out of scope; reported, not changed).

---

## 5. Compliance & Quality Review

Cross-mapping of AAP deliverables and the governing rule set (`SWE-AtlasQnA-Repo`) to Blitzy quality/compliance benchmarks. Fixes applied during autonomous validation are noted.

| Benchmark / AAP Rule | Requirement | Status | Progress | Notes / Fixes Applied |
|----------------------|-------------|--------|----------|------------------------|
| Output artifact | Single Markdown answer at `blitzy/documentation/app_2cd6ee777f8c.md` | ✅ Pass | 100% | Created and committed (653 lines) |
| Read-only mandate | No modification/addition/deletion of any existing file | ✅ Pass | 100% | `git diff` shows only the deliverable added |
| Run-first investigation | Build/run code paths; write from observation | ✅ Pass | 100% | All 5 entry points run; evidence captured |
| Real entry points only | Exercise HTTP `:7777` and SMTP `:20381` | ✅ Pass | 100% | Bypasses labeled non-canonical where used |
| Default canonical config | Run defaults; state commands; disclose decisive flags | ✅ Pass | 100% | `NOT_SEND_EMAIL`, `DISABLE_ONBOARDING`, `EVENT_WEBHOOK` disclosed |
| Verbatim quoting | Quote actual log/console/HTTP output | ✅ Pass | 100% | Verbatim fences incl. source typo preserved |
| One claim → one evidence | Adjacent evidence per behavioral claim | ✅ Pass | 100% | Enforced throughout |
| Exact literals + `file:line` | Cite exact values with references | ✅ Pass | 100% | 88 citations across 27 files; all resolve |
| Magnitude/frequency/timing | Observe across ≥2 cycles; state duration | ✅ Pass | 100% | Job cadence ×5 cycles; rate-limit ≥2 runs |
| Coverage pass | Address every named item by name | ✅ Pass | 100% | 27-item checklist; 2 honestly ⚠ |
| Label inferred/non-canonical | Distinguish observed vs inferred/bypass | ✅ Pass | 100% | 3-level labeling on activation link & events |
| Cleanup | Remove temp users/aliases/scripts/services | ✅ Pass | 100% | Verified counts = 0; DB baseline intact |
| No dependency changes | Do not alter deps/lockfiles | ✅ Pass | 100% | None touched |
| Structural quality | Valid Markdown, no placeholders | ✅ Pass | 100% | Balanced fences; 0 TODO/FIXME/TBD |

**Quality highlight:** The deliverable exceeds the AAP baseline by *correcting* an initial expectation (that the activation link is logged) with observed reality, and by honestly documenting the gated event pipeline — both with clearly labeled evidence levels, fulfilling the "report exactly what is observed even if unexpected" rule.

---

## 6. Risk Assessment

All risks are **Low** severity: this is a read-only, additive-Markdown deliverable that introduces zero code, dependency, config, or migration changes and no new runtime surface. Risks I1/I2/O2 are disclosures *about the software's shipped defaults* that the document surfaces for operators, not defects introduced by this work.

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| T1 — Session-specific evidence (PIDs, timings, session cookie) won't reproduce identically | Technical | Low | High | Document labels session-specific values; grounds portable claims in source | ✅ Mitigated |
| T2 — Citation line-number drift if source files are later edited | Technical | Low | Medium | Document states "verified against HEAD"; re-verify on source change | ⚠ Open (informational) |
| T3 — Two ⚠ items misread as bugs | Technical | Low | Low | Document explicitly explains these are shipped default behavior | ✅ Mitigated |
| S1 — Secret exposure in captured output | Security | Low | Low | Session cookie shown as `[REDACTED]`; only throwaway temp credentials used | ✅ Mitigated |
| S2 — Seeded demo credential (`john@wick.com/password`) shown | Security | Low | Low | Public fixture from `CONTRIBUTING.md`, not a real secret | ✅ N/A |
| O1 — Reproducing evidence requires provisioning PG+Redis+seed+5 processes+aux services | Operational | Low | Medium | Document states exact commands (see Section 9) | ✅ Documented |
| O2 — Benign browser-console `plausible` ReferenceError persists in app | Operational | Low | High | Root-caused in document; fix requires a separate non-read-only task | ⚠ Documented/Deferred |
| I1 — Canonical alias-creation→webhook does not deliver for ordinary self-hosted users by default | Integration | Low-Medium | Medium | Both gates documented precisely with `file:line`; set `EVENT_WEBHOOK` + partner user to enable | ✅ Documented |
| I2 — Activation email disabled by default (`NOT_SEND_EMAIL=true`) | Integration | Low | High | Document shows default behavior + non-default MTA path; real deploy needs MTA/Postfix | ✅ Documented |

---

## 7. Visual Project Status

**Project hours (integrity-critical): Completed = 28, Remaining = 3, Total = 31.** These values match Section 1.2 and the Section 2.2 sum exactly.

```mermaid
%%{init: {"theme":"base","themeVariables":{"pie1":"#5B39F3","pie2":"#FFFFFF","pieStrokeColor":"#B23AF2","pieStrokeWidth":"2px","pieOuterStrokeWidth":"2px","pieTitleTextSize":"18px","pieSectionTextSize":"15px","pieLegendTextSize":"14px"}}}%%
pie showData title Project Hours (Completed vs Remaining)
    "Completed Work" : 28
    "Remaining Work" : 3
```

**Remaining work by priority** (sums to the 3 remaining hours):

```mermaid
%%{init: {"theme":"base","themeVariables":{"pie1":"#5B39F3","pie2":"#A8FDD9","pieStrokeColor":"#B23AF2","pieStrokeWidth":"2px","pieOuterStrokeWidth":"2px","pieTitleTextSize":"16px","pieSectionTextSize":"14px","pieLegendTextSize":"13px"}}}%%
pie showData title Remaining Hours by Priority
    "High" : 2
    "Medium" : 1
```

**Remaining work by category (Section 2.2):**

| Category | Hours | Priority |
|----------|-------|----------|
| Technical-accuracy review & runtime re-verification | 2 | High |
| Editorial review & stakeholder sign-off / merge acceptance | 1 | Medium |
| **Total** | **3** | — |

> Color key — **Completed = Dark Blue `#5B39F3`**, **Remaining = White `#FFFFFF`** (priority accent uses Mint `#A8FDD9`). Remaining Work (3 h) is identical across Sections 1.2, 2.2, and 7.

---

## 8. Summary & Recommendations

**Achievements.** The project delivered exactly what the AAP scoped: a single, runtime-grounded Q&A documentation file that answers Q1 (readiness), Q2 (new-user flow), and Q3 (behind-the-scenes services) with verbatim evidence and precise `file:line` citations. All five process entry points were run and observed; the full register → verify → login → dashboard journey was exercised through the real HTTP endpoints; and the background/internal-service behavior (job cadence, email forwarding, event pipeline) was captured live. The repository was left byte-for-byte unchanged apart from the deliverable.

**Remaining gaps.** None in the autonomous AAP scope. The remaining **3 hours** are human path-to-production activities appropriate for a documentation artifact: a technical-accuracy review with runtime re-verification, and editorial/stakeholder sign-off.

**Critical path to production.** (1) Human technical-accuracy review → (2) coverage-pass confirmation → (3) editorial review → (4) stakeholder sign-off and merge. No engineering rework is required.

**Success metrics.** 27 cited files / 88 citations all resolving at HEAD; 174/174 autonomous validation checks passing; 0 source-code changes; clean working tree; verified cleanup to the seeded DB baseline.

**Production readiness assessment.** The deliverable is **production-ready pending human review**. The project is **90.3% complete** (28 / 31 hours); the outstanding 10% is entirely human review/acceptance, consistent with the principle that a documentation artifact is finalized by human sign-off rather than further autonomous work.

| Metric | Value |
|--------|-------|
| Completion | 90.3% (28/31 h) |
| AAP requirements delivered | 28 / 28 (100%) |
| Autonomous validation checks | 174 / 174 passing |
| Source-code changes | 0 |
| Remaining (human) | 3 h |

---

## 9. Development Guide

This guide covers both (a) locating and validating the deliverable and (b) reproducing the runtime evidence that grounds it. Commands were tested during this assessment where the environment allowed.

### 9.1 System Prerequisites

- **Canonical runtime image:** `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (default configuration).
- **Languages/runtimes:** Python `^3.10` (observed 3.10.18); Node v10 for **asset build only** (assets are prebuilt in the image).
- **Services:** PostgreSQL 13+ (observed 15.13); Redis (observed 7.0.15).
- **Ports:** `7777` (web/HTTP), `20381` (local SMTP; maps from 25 in production).
- **Default config:** `/app/.env` is a byte-for-byte copy of `example.env`. Decisive flags: `NOT_SEND_EMAIL=true`, `DISABLE_ONBOARDING=true`; `EVENT_WEBHOOK` unset.

### 9.2 Environment Setup

```bash
# Confirm the running config is the shipped default
diff -q .env example.env        # (no output) => identical to example.env

# Confirm runtime versions
python --version                # Python 3.10.x
psql "postgresql://myuser:mypassword@localhost:5432/simplelogin" -tAc "show server_version;"
redis-cli ping                  # PONG
```

### 9.3 Provisioning / Migration / Seed

```bash
# Canonical bring-up recipe (CONTRIBUTING.md:L106); seeds john@wick.com / password
alembic upgrade head && flask dummy-data

# Verify migration head and seeded users
psql "postgresql://myuser:mypassword@localhost:5432/simplelogin" -tAc "SELECT version_num FROM alembic_version;"
psql "postgresql://myuser:mypassword@localhost:5432/simplelogin" -tAc "SELECT id,email,activated FROM users ORDER BY id LIMIT 3;"
```

### 9.4 Application Startup (the five process entry points)

```bash
# 1. Web app — canonical production entry point (Dockerfile:L47), bound to 0.0.0.0
/app/venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15

# 2. Inbound SMTP handler (listens on 20381)
/app/venv/bin/python email_handler.py

# 3. Background job runner (10-second polling loop)
/app/venv/bin/python job_runner.py

# 4. Event listener (the "listener" subcommand is REQUIRED)
/app/venv/bin/python event_listener.py listener

# 5. Cron — scheduled by yacron reading crontab.yml; yacron runs: python cron.py -j <job>
/app/venv/bin/python cron.py -j stats
```

> **Note:** `python server.py` (Flask dev server) binds to `127.0.0.1` only and is not reachable from outside the container — use the Gunicorn form above (binds `0.0.0.0`).

### 9.5 Verification Steps

```bash
# Web readiness — expect HTTP/1.1 200 OK and body "success"
curl -sS -i http://localhost:7777/health

# UI landing — expect 200 (login), 302 (root redirect), 200 (register)
curl -sS -o /dev/null -w "%{http_code}\n" http://localhost:7777/auth/login
curl -sS -o /dev/null -w "%{http_code}\n" http://localhost:7777/
curl -sS -o /dev/null -w "%{http_code}\n" http://localhost:7777/auth/register
```

Expected readiness log signals: `>>> init logging <<<`; `Listen for port 20381`; `Using PostgresEventSource` / `Starting with HttpEventSink`; job runner idle poll every ~10 s.

### 9.6 Example Usage — view & validate the deliverable

```bash
# Locate & size the deliverable
ls -la blitzy/documentation/app_2cd6ee777f8c.md

# Structural validation — fences must be EVEN, placeholders must be 0
grep -c '^```' blitzy/documentation/app_2cd6ee777f8c.md          # 62 (even)
grep -cE 'TODO|FIXME|TBD|PLACEHOLDER' blitzy/documentation/app_2cd6ee777f8c.md  # 0

# Repository hygiene — clean tree; only the deliverable added vs base
git status --porcelain                                            # (empty)
git diff 2cd6ee77 HEAD --name-status                              # A blitzy/documentation/app_2cd6ee777f8c.md
```

Programmatic citation-resolution check (tested; result: 27 files / 88 citations / 0 missing / 0 out-of-range):

```bash
python3 - <<'PY'
import re, os
text = open("blitzy/documentation/app_2cd6ee777f8c.md", encoding="utf-8").read()
pat = re.compile(r'`([A-Za-z0-9_./-]+\.(?:py|yml|env|toml|md|html)):L(\d+)')
refs = {}
for m in pat.finditer(text):
    refs.setdefault(m.group(1), set()).add(int(m.group(2)))
missing = [p for p in refs if not os.path.isfile(p)]
oob = sum(1 for p,ls in refs.items() if os.path.isfile(p)
          for ln in ls if ln > sum(1 for _ in open(p, errors="replace")))
print(f"files={len(refs)} citations={sum(len(v) for v in refs.values())} missing={len(missing)} out_of_range={oob}")
PY
```

### 9.7 Troubleshooting

- **Web app unreachable from host:** `python server.py` binds `127.0.0.1` only — start with `gunicorn wsgi:app -b 0.0.0.0:7777` instead.
- **Activation link not in logs:** expected under `NOT_SEND_EMAIL=true` (only subject/from/to are logged). To see the real link, run a temporary MTA with `NOT_SEND_EMAIL` disabled (labeled non-default).
- **Event listener stays idle during normal use:** expected — `EVENT_WEBHOOK` is unset by default, so the alias-creation pipeline short-circuits before emitting a `SyncEvent`; even with a webhook, ordinary self-hosted users are blocked by the partner-user gate.
- **Browser console `ReferenceError: plausible is not defined` on the register waiting page:** benign, non-blocking; the analytics loader early-returns on non-prod hosts and never defines the global stub. Out of scope for the read-only task.
- **`429` on `/auth/login` or `/auth/activate`:** expected `10/minute` rate limit (deducted on failed attempts; per-worker in-memory counters under `-w 2`). `/auth/register` has no limiter.

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---------|---------|
| Provision DB + seed | `alembic upgrade head && flask dummy-data` |
| Start web app | `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` |
| Start SMTP handler | `python email_handler.py` |
| Start job runner | `python job_runner.py` |
| Start event listener | `python event_listener.py listener` |
| Run a cron job | `python cron.py -j stats` |
| Health probe | `curl -sS -i http://localhost:7777/health` |
| Repo hygiene | `git status --porcelain` ; `git diff 2cd6ee77 HEAD --name-status` |

### B. Port Reference

| Port | Service | Notes |
|------|---------|-------|
| 7777 | Web app (HTTP) | Gunicorn `-b 0.0.0.0:7777`; `EXPOSE 7777` in Dockerfile |
| 20381 | Local SMTP handler | aiosmtpd; maps from 25 in production |
| 5432 | PostgreSQL | Application DB + `LISTEN/NOTIFY` |
| 6379 | Redis | Sessions / rate limiting |

### C. Key File Locations

| Path | Role |
|------|------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The deliverable** |
| `server.py`, `wsgi.py` | Flask app factory / WSGI object; `/health`; access log |
| `email_handler.py` | Inbound SMTP handler + forwarding lifecycle |
| `job_runner.py` | 10-second `Job` queue drain loop |
| `event_listener.py`, `events/` | Event listener + `PostgresEventSource` / `HttpEventSink` / `Runner` |
| `cron.py`, `crontab.yml` | Scheduled-maintenance dispatcher (yacron) |
| `app/auth/views/{register,activate,login,login_utils}.py` | Auth flow |
| `app/dashboard/views/index.py` | Dashboard landing |
| `app/log.py` | Logging bootstrap + uniform format |
| `example.env`, `app/config.py` | Default configuration |

### D. Technology Versions

| Component | Version (observed / manifest) |
|-----------|-------------------------------|
| Python | 3.10.18 (`pyproject.toml` `^3.10`) |
| Gunicorn | 20.0.4 |
| PostgreSQL | 15.13 (requirement 13+) |
| Redis | 7.0.15 |
| Flask | `^1.1.2` |
| SQLAlchemy | 1.3.24 |
| aiosmtpd | `^1.2` |
| yacron | `^0.11.1` |
| Node (asset build only) | v10 (image); not used at runtime |

### E. Environment Variable Reference

| Variable | Default | Effect on observations |
|----------|---------|------------------------|
| `NOT_SEND_EMAIL` | `true` | Transactional email logged as metadata only; activation link not in logs |
| `DISABLE_ONBOARDING` | `true` | Onboarding `Job` rows not enqueued on self-hosted registration |
| `EVENT_WEBHOOK` | unset | Alias-creation event pipeline wired but dormant by default |
| `URL` | `http://localhost:7777` | Base URL used to build the activation link |
| `DB_URI` | `postgresql://myuser:mypassword@localhost:5432/simplelogin` | Application database |
| `POSTFIX_PORT` | (self-hosting) | Outbound MTA port (e.g., `1025` for a local catcher) |

### F. Developer Tools Guide

| Tool | Use |
|------|-----|
| `git` | Verify hygiene: `git status --porcelain`, `git diff 2cd6ee77 HEAD --name-status`, `git log --author="agent@blitzy.com" --oneline` |
| `psql` | Verify migration head, seeded users, and post-cleanup baseline counts |
| `redis-cli` | `ping` → `PONG` readiness check |
| `curl` | Health probe and UI landing status codes |
| `python3` (regex script) | Programmatic citation-resolution check (Section 9.6) |
| `grep` / `wc` | Markdown structural validation (fences, headings, placeholders) |

### G. Glossary

| Term | Meaning |
|------|---------|
| **AAP** | Agent Action Plan — the primary directive defining project scope |
| **Alias** | A SimpleLogin email address that forwards to a user's real mailbox |
| **ActivationCode** | The token issued at registration and consumed at `/auth/activate` for identity verification |
| **SyncEvent** | A DB row representing an event, announced via PostgreSQL `NOTIFY` |
| **HttpEventSink** | The webhook delivery target for consumed events |
| **VERP / reverse-alias** | The rewritten `From` address enabling replies to route back through SimpleLogin |
| **Non-canonical** | A value obtained via a bypassing interface (e.g., direct DB read) rather than the real user-facing path — always labeled as such |
| **Non-default** | A value obtained after changing a shipped default flag — always labeled as such |
| **⚠ (coverage pass)** | An item honestly marked as gated/unavailable under default config, rather than overclaimed as complete |

---

*Blitzy Project Guide generated for branch `blitzy-04138eae-fe6d-47c5-986f-bf39151cbea3`. Completion: **90.3%** (28 completed / 3 remaining / 31 total hours). Colors — Completed `#5B39F3`, Remaining `#FFFFFF`.*