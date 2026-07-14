# Blitzy Project Guide

**Project:** SimpleLogin Development Stack Bring-Up — Runtime-Verified Answers (QnA Documentation)
**Branch:** `blitzy-7065bab1-a6a2-494e-8554-4f1f5dff7530` · **HEAD:** `239ce63c` · **Base:** `2cd6ee777f8c`
**Deliverable:** `blitzy/documentation/app_2cd6ee777f8c.md` (2,448 lines)
**Task type:** Documentation (runtime-observation QnA) · **Scope:** Strict read-only, single-file additive, net-zero

> **Legend / Blitzy brand colors used throughout:** Completed / AI Work = **Dark Blue `#5B39F3`** · Remaining / Not Completed = **White `#FFFFFF`** · Headings & accents = **Violet-Black `#B23AF2`** · Highlights = **Mint `#A8FDD9`**.

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a single, evidence-backed markdown document that answers three questions about standing up the SimpleLogin development stack from scratch, written entirely from first-hand runtime observation of the application's real behavior. It serves engineers onboarding to SimpleLogin who find the startup order and failure modes under-documented. The document precisely captures (Q1) the exact Python exception raised against an empty, unmigrated database; (Q2) every Python service required to run the system, with real startup logs and port bindings; and (Q3) the SMTP status code and rejection log when initialization is skipped. The task is investigation-and-documentation under a strict read-only constraint — no source code is changed; the only net repository change is the new document.

### 1.2 Completion Status

The completion percentage is computed using the AAP-scoped, hours-based methodology: `Completed Hours ÷ Total Hours × 100`. All autonomous work in scope of the Agent Action Plan is finished and validated; the remaining hours are the human review-and-merge gate that cannot be autonomously completed.

```mermaid
%%{init: {"theme":"base","themeVariables":{"pie1":"#5B39F3","pie2":"#FFFFFF","pieStrokeColor":"#B23AF2","pieOuterStrokeColor":"#B23AF2","pieStrokeWidth":"2px","pieTitleTextSize":"16px","pieSectionTextSize":"13px","pieLegendTextSize":"13px"}}}%%
pie showData title Completion — 91.67% (33.0 of 36.0 hrs)
    "Completed Work (AI)" : 33
    "Remaining Work (Human)" : 3
```

| Metric | Value |
|---|---|
| **Total Hours** | **36.0 h** |
| **Completed Hours (AI + Manual)** | **33.0 h** (AI: 33.0 h · Manual: 0.0 h) |
| **Remaining Hours** | **3.0 h** |
| **Percent Complete** | **91.67%** |

> Formula: `33.0 ÷ 36.0 × 100 = 91.67%`. These exact figures (36.0 / 33.0 / 3.0 / 91.67%) are used consistently in Sections 2, 7, and 8.

### 1.3 Key Accomplishments

- ✅ **Deliverable authored and committed** — `blitzy/documentation/app_2cd6ee777f8c.md`, 2,448 lines (195 KB), sole author `Blitzy Agent <agent@blitzy.com>`.
- ✅ **Q1 fully answered** — empty-database boot succeeds, first ORM query fails with `sqlalchemy.exc.ProgrammingError` wrapping `psycopg2.errors.UndefinedTable: relation "users" does not exist`; the distinct pre-database `KeyError: 'URL'` (missing `.env`) is demonstrated and kept separate.
- ✅ **Q2 fully answered** — all **five** required Python services enumerated and started with real readiness logs and port-binding proof (web `0.0.0.0:7777`, SMTP `0.0.0.0:20381`, plus `job_runner`, `cron`, `event_listener`), including an integrated all-four-simultaneous topology.
- ✅ **Q3 fully answered** — `550 SL E515 Email not exist` captured with full rejection log chain; the `250 SL E207` alternate branch and the causal nuance (forward path checks `CustomDomain`, not `SLDomain`) both exercised.
- ✅ **Runtime-first methodology honored** — every claim grounded in captured output plus its producing command; **324 file:line citations** across 36 files; strict Observed-vs-Inferred labeling; **43-row coverage matrix** over every named item and implied condition.
- ✅ **Byte-for-byte reproducibility** — all three scenarios independently reproduced and stable across two runs in the canonical container.
- ✅ **Net-zero read-only compliance** — branch diff vs. base is exactly one file added (+2,448 / −0); working tree clean; all scratch DBs, temp `.env`, and helper scripts removed.
- ✅ **Autonomous QA closed** — 6-commit author→review→fix lifecycle; 114 citations verified (113/114), the one defect fixed and committed (`239ce63c`) while preserving 36 verbatim CRLFs.

### 1.4 Critical Unresolved Issues

There are **no critical unresolved issues**. The single defect found during validation was resolved and committed. The table records the one advisory item carried forward.

| Issue | Impact | Owner | ETA |
|---|---|---|---|
| Verbatim CRLF fragility (advisory): 36 intentional CRLF endings in captured SMTP transcripts can be silently normalized by naive text tooling | Low — cosmetic drift of protocol-true captures if the file is edited without byte-precise tooling | Human reviewer / any future editor | On next edit (guidance provided) |
| Citation freshness (advisory): 324 `file:line` citations are pinned to source HEAD `2cd6ee77` | Low — line numbers would drift only if the document is re-based onto a newer SimpleLogin source | Human reviewer | At review time |

### 1.5 Access Issues

| System / Resource | Type of Access | Issue Description | Resolution Status | Owner |
|---|---|---|---|---|
| Git repository | Read/Write to branch | None — working tree clean, deliverable committed, branch writable | ✅ No issue | Blitzy Agent |
| Canonical runtime image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` | Container registry pull | Advisory only — a reviewer wishing to *optionally* re-run scenarios needs registry access; **not blocking** because all exact captures are already embedded in the document | ⚠ Advisory (non-blocking) | Human reviewer |
| Service credentials / third-party APIs | N/A | Not required — this is a documentation deliverable with zero external service wiring | ✅ Not applicable | — |

### 1.6 Recommended Next Steps

1. **[High]** Perform the technical accuracy review — read the Q1/Q2/Q3 answers and spot-check a sample of the 324 `file:line` citations against source HEAD `2cd6ee77`.
2. **[High]** Perform the editorial review — verify readability, heading/format consistency, and correct application of Observed-vs-Inferred labels.
3. **[Medium]** Approve and merge the branch (single-file, net-zero add).
4. **[Medium]** Link the document into the team's onboarding / docs index and notify the dev-onboarding audience.
5. **[Low]** *(Optional)* Independently re-run one scenario in the canonical container to personally confirm — redundant with the validator's byte-for-byte reproduction.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

Every completed component traces to a specific Agent Action Plan requirement. All hours below were delivered autonomously (AI).

| Component | Hours | Description |
|---|---:|---|
| Environment provisioning & `.env` prerequisite | 3.5 | Python 3.10 venv, PostgreSQL 15, Redis 6379, Poetry deps + `pyre2==0.3.6` build; temporary `.env` from `example.env` satisfying mandatory `URL`/`EMAIL_DOMAIN`/`SUPPORT_EMAIL` (`app/config.py:79,92,93`). Doc Section 0.1/0.2. |
| Q1 — empty-DB startup error observation | 4.0 | Empty unmigrated DB → `python server.py` → open `/login` → capture full `ProgrammingError`/`UndefinedTable` traceback + exact relation `users`; observe connect-succeeds/query-fails boundary; demonstrate distinct pre-DB `KeyError`. |
| Q2 — required-services enumeration + startup/port-bind | 6.0 | Enumerate & start all 5 services; capture readiness logs + port-binding evidence via `/proc/net/tcp`; distinguish listeners from workers; integrated all-four-simultaneous topology with PID→port mapping. |
| Q3 — skipped-init email rejection | 6.0 | Migrated-but-uninitialized DB (`public_domain` empty) → inject real inbound mail to `@sl.local` → capture `550 SL E515` + 11-line rejection chain; `E207` alternate branch; null reverse-path `MAIL FROM:<>`; causal cross-check after `init_app.py`. |
| Deliverable authoring | 6.0 | Compose the 2,448-line document: three answers, security notice, execution chronology, 324 `file:line` citations, Observed/Inferred discipline, exact producing commands. |
| Coverage pass | 1.5 | Build the 43-row coverage matrix mapping every named item/implied condition (Q1 a–i, Q2 a–q, Q3 a–r, cleanup) to evidence. |
| Cleanup to net-zero | 1.0 | Drop 4 scratch DBs, remove temp `.env` and helper scripts, prove net-zero with runtime checks; repo left git-clean with only the new document. |
| Autonomous QA, citation verification & defect fixes | 5.0 | 6-commit author→review→fix lifecycle: 18 code-review findings, QA findings F1/F2/F3, 114-citation verification (38 files), byte-precise doubled-path fix `239ce63c` preserving 36 CRLFs. |
| **Total Completed** | **33.0** | **Matches Completed Hours in Section 1.2** |

### 2.2 Remaining Work Detail

All remaining work is the human path-to-production gate; none is autonomously completable.

| Category | Hours | Priority |
|---|---:|---|
| Human technical + editorial review of the deliverable (read 2,448 lines; spot-check citations; verify Observed/Inferred labels & formatting) | 2.0 | High |
| Merge / PR sign-off + docs-index linking + stakeholder notification | 1.0 | Medium |
| **Total Remaining** | **3.0** | **Matches Remaining Hours in Section 1.2 & Section 7 pie** |

### 2.3 Hours Summary

| Bucket | Hours | Share |
|---|---:|---:|
| Completed (AI) | 33.0 | 91.67% |
| Remaining (Human) | 3.0 | 8.33% |
| **Total** | **36.0** | **100%** |

> **Integrity check:** Section 2.1 (33.0) + Section 2.2 (3.0) = 36.0 = Total Hours in Section 1.2. ✔

---

## 3. Test Results

**Nature of testing.** This is a documentation deliverable; there are **no in-scope automated unit tests** to add or run. The acceptance-equivalent validation is **independent, first-hand runtime reproduction of every documented claim**, performed by Blitzy's autonomous validation inside the canonical container. Every entry below originates from Blitzy's autonomous validation logs for this project (Integrity Rule 3).

| Test Category | Framework | Total Checks | Passed | Failed | Coverage % | Notes |
|---|---|---:|---:|---:|---:|---|
| Q1 runtime reproduction | Runtime observation (real entry point `python server.py`) | 9 | 9 | 0 | 100% | Coverage rows Q1 a–i; incl. connect/query boundary, HTTP 500 (CL 5749), exact relation `users`, KeyError distinction; stable across 2 runs |
| Q2 runtime reproduction | Runtime observation (dev + canonical `gunicorn`, aiosmtpd, workers) | 17 | 17 | 0 | 100% | Coverage rows Q2 a–q; 5 services, port binds, integrated all-4 topology, PID→port mapping; listener stable across 2 runs |
| Q3 runtime reproduction | Runtime observation (real aiosmtpd listener `:20381`) | 18 | 18 | 0 | 100% | Coverage rows Q3 a–r; E515 primary + E207 branch + null reverse-path; causal post-init cross-check; net-zero rows; stable across 2 runs |
| Citation verification | Source cross-reference vs. HEAD `2cd6ee77` | 114 | 114 | 0 | 100% | 113/114 accurate on first pass; the single doubled-path defect fixed & committed (`239ce63c`) → 114/114 |
| Net-zero / cleanliness | Runtime checks (`git status`, port & DB inventory) | 1 | 1 | 0 | 100% | Repo diff = 1 file added; scratch DBs dropped; temp `.env`/scripts removed; ports 7777/20381 free |
| **Total** | — | **159** | **159** | **0** | **100%** | All checks sourced from Blitzy autonomous validation logs |

> **Note:** "Checks" are discrete verified assertions drawn from the document's 43-row coverage matrix plus the 114 citation verifications and the net-zero sweep — not a synthetic unit-test suite.

---

## 4. Runtime Validation & UI Verification

All services were exercised through their real entry points (no mocks/bypasses) in the canonical container.

**Service runtime health**
- ✅ **Web (dev)** `python server.py` — Operational; binds `127.0.0.1:7777`.
- ✅ **Web (canonical)** `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` — Operational; `Starting gunicorn 20.0.4`, `Listening at: http://0.0.0.0:7777`, 2 workers booted; `GET /health` → `200`.
- ✅ **SMTP handler** `python email_handler.py` — Operational; binds `0.0.0.0:20381`; live banner `220 … Python SMTP 1.4.2`.
- ✅ **job_runner.py** — Operational; took a real bounded job (`ready → done`); net-zero (probe row deleted).
- ✅ **cron.py -j delete_old_monitoring** — Operational; `Start running cronjob` → exit code `0`.
- ✅ **event_listener.py listener** — Operational; `Using PostgresEventSource` → `Starting to listen to events`.
- ✅ **Integrated topology** — all four long-lived services co-running (`4/4` alive); 7777 and 20381 ownership mapped to PIDs; clean teardown releases both ports.

**API / protocol verification**
- ✅ `GET /` → `302` → `/auth/login`; `GET /auth/login` → `200`; `POST /auth/login` (empty DB) → `500` (generic `error/500.html`, 5749 bytes).
- ✅ SMTP inbound `nonexistent@sl.local` → `550 SL E515 Email not exist`.
- ✅ SMTP inbound with ignore-bounce sender → `250 SL E207 No bounce report`.

**UI verification**
- ⚠ **Partial (by design):** the login page is *exercised* only to trigger the backend query for Q1; no UI is designed, altered, or asserted for visual fidelity. The relevant fact is behavioral: `/` redirects to `auth.login` and the login POST surfaces the exception. No other UI is in scope.

---

## 5. Compliance & Quality Review

Cross-mapping of AAP deliverables and the "SWE-AtlasQnA-Repo" rules to their realized status.

| Deliverable / Rule | Benchmark | Status | Evidence |
|---|---|---|---|
| Single deliverable `blitzy/documentation/app_2cd6ee777f8c.md` | Exactly one new file, correct `<branch>.md` name/location | ✅ Pass | Branch diff = 1 file added (+2,448/−0) |
| Strict read-only | No existing source modified/deleted | ✅ Pass | `git diff --name-status` = single `A` line; tree clean |
| Net-zero temp artifacts | All temp `.env`/scripts/scratch DBs removed | ✅ Pass | Cleanup section proves removal; 4 scratch DBs dropped |
| Run-first-then-write | Answers from observation, not reading | ✅ Pass | 97 Observed labels; captures with producing commands |
| Observed vs. Inferred labeling | Every inferred claim labeled | ✅ Pass | 97 Observed / 12 Inferred throughout |
| Exact & grounded (`file:line`) | Every claim cited | ✅ Pass | 324 citations across 36 files |
| Answer every named item | Each named entity addressed by name | ✅ Pass | 43-row coverage matrix |
| Exercise every condition (incl. edge/error) | Primary + alternate/edge/boundary | ✅ Pass | E515/E207 branches; connect/query boundary; null reverse-path |
| Include actual, unedited output | No paraphrase/elision | ✅ Pass | 86 verbatim code fences; 36 preserved CRLFs |
| Canonical path only | Real entry points, no bypass | ✅ Pass | `server.py`, `gunicorn`, aiosmtpd `:20381` |
| Stability across ≥2 runs | Confirm variable values stable | ✅ Pass | Q1/Q2/Q3 each reproduced twice |

**Fixes applied during autonomous validation:** 18 code-review findings resolved; QA findings F1/F2/F3 resolved with runtime-verified evidence; 2 off-by-one citation fixes (Q1) + 1 (Q2 `local_main`); doubled-path (`app/app/`) citation defect fixed (`239ce63c`) with byte-precise editing that preserved 36 verbatim CRLFs.

**Outstanding compliance items:** none autonomous. Human editorial/technical sign-off remains (Section 2.2).

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|---|---|---|---|---|---|
| R-T1 Citation drift if re-based onto newer source | Technical | Low | Medium | Citations pinned to documented HEAD `2cd6ee77`; validator verified 113/114 then fixed 1 | Mitigated |
| R-T2 Verbatim CRLF corruption by naive text tooling | Technical | Low | Medium | Use byte-precise (`rb`/`wb`) edits; documented for reviewers | Open (advisory) |
| R-T3 Reproducibility value-drift under different versions | Technical | Low | Low | Exact versions + canonical image stated; 2-run stability confirmed | Mitigated |
| R-S1 Documented `0.0.0.0` binds (7777/20381) | Security | Low | Low | Explicit Security & isolation notice: reproduce only in isolated, disposable container with unpublished ports | Mitigated |
| R-S2 Frozen old dependency versions may carry CVEs | Security | Low | N/A | Pre-existing project state; remediation explicitly out of scope (read-only) | Accepted (out of scope) |
| R-S3 New attack surface from this change | Security | None | N/A | Zero code/deps added; markdown only; net-zero verified | N/A |
| R-O1 Three gunicorn zombie PIDs (defunct) | Operational | Very Low | N/A | Hold no ports/resources; clear on container restart; cannot reap without killing PID 1 (forbidden) | Accepted (informational) |
| R-O2 Canonical-container dependence for re-run | Operational | Low | Low–Med | Exact captures embedded so re-run is confirmatory, not required | Mitigated (advisory) |
| R-I1 External canonical image availability | Integration | Low | Low | Image ref + all commands documented; observed values recorded | Mitigated |
| R-I2 No CI/CD or service-integration points | Integration | None | N/A | Doc-only; no pipelines, keys, or wiring to break | N/A |

**Overall risk posture:** Low. No blocking risks. The only genuinely open item is the R-T2 advisory on byte-precise editing.

---

## 7. Visual Project Status

**Overall completion (hours)** — Completed = Dark Blue `#5B39F3`, Remaining = White `#FFFFFF`.

```mermaid
%%{init: {"theme":"base","themeVariables":{"pie1":"#5B39F3","pie2":"#FFFFFF","pieStrokeColor":"#B23AF2","pieOuterStrokeColor":"#B23AF2","pieStrokeWidth":"2px","pieSectionTextSize":"13px","pieLegendTextSize":"13px"}}}%%
pie showData title Project Hours Breakdown
    "Completed Work" : 33
    "Remaining Work" : 3
```

**Remaining work by priority (hours)**

```mermaid
%%{init: {"theme":"base","themeVariables":{"pie1":"#B23AF2","pie2":"#A8FDD9","pieStrokeColor":"#5B39F3","pieOuterStrokeColor":"#5B39F3","pieStrokeWidth":"2px","pieSectionTextSize":"13px","pieLegendTextSize":"13px"}}}%%
pie showData title Remaining Hours by Priority
    "High — Review" : 2
    "Medium — Merge/Publish" : 1
```

**Remaining hours by category (Section 2.2)**

| Category | Hours | Bar |
|---|---:|---|
| Human technical + editorial review | 2.0 | ██████████████████████████████ |
| Merge / PR sign-off + docs-index link | 1.0 | ███████████████ |
| **Total** | **3.0** | |

> **Integrity check:** Section 7 "Remaining Work" (3) = Section 1.2 Remaining Hours (3.0) = Section 2.2 total (3.0). ✔ Priority pie (2 + 1) = 3. ✔

---

## 8. Summary & Recommendations

**Achievements.** The project is **91.67% complete** (33.0 of 36.0 hours). The sole AAP deliverable — a 2,448-line, runtime-verified QnA document — is authored, validated, and committed. All three questions are answered from first-hand observation through real entry points, with 324 `file:line` citations, strict Observed/Inferred discipline, a 43-row coverage matrix, and byte-for-byte reproducibility across two runs. The repository is a clean, net-zero, single-file addition, honoring the strict read-only constraint.

**Remaining gaps.** The remaining **3.0 hours** are entirely the human path-to-production gate: a technical + editorial review of the document (2.0h) and merge/publish (1.0h). There is no product code, no failing test, and no blocking issue. Because the maximum realistic autonomous completion before human review is capped below 100%, 91.67% reflects a deliverable that is functionally done but awaiting human sign-off.

**Critical path to production.** (1) Technical accuracy review → (2) editorial review → (3) merge → (4) index/notify. Estimated wall-clock: well within a single review session.

**Success metrics (all met autonomously).** Three questions answered ✔ · every named item covered ✔ · every claim grounded in capture or `file:line` ✔ · 2-run stability ✔ · net-zero repository ✔ · zero unresolved defects ✔.

**Production-readiness assessment.** **Ready for human review and merge.** Confidence: **High** — the scope is small and fully evidenced by validation logs and git state. Recommended action: proceed with the Section 1.6 next steps.

| Metric | Value |
|---|---|
| Completion | 91.67% |
| Completed / Total hours | 33.0 / 36.0 |
| Remaining hours | 3.0 |
| Unresolved defects | 0 |
| Blocking risks | 0 |
| Confidence | High |

---

## 9. Development Guide

This guide has two parts: **(A)** reviewing the deliverable (works on any host with `git`) and **(B)** reproducing the three documented scenarios (requires the canonical container). Part A commands were executed and verified on the working host; Part B commands are the exact, byte-for-byte-verified commands from the deliverable and the Final Validator's canonical-container runs.

### 9.1 System Prerequisites

- **Review (Part A):** `git` (validated: 2.51.0) and any markdown viewer or `python3`.
- **Reproduction (Part B):** Debian 12 base; **Python 3.10** venv (validated 3.10.18); **PostgreSQL 15** on `:5432`; **Redis** on `:6379`; pinned deps **Flask 1.1.2**, **SQLAlchemy 1.3.24**, **aiosmtpd 1.4.2**, **gunicorn 20.0.4**, **psycopg2-binary 2.9.3**, **python-dotenv 0.14.0**, and a working `re2` (**pyre2 0.3.6**). Canonical image: `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`.

### 9.2 Part A — Access & Review the Deliverable (verified on this host)

```bash
# View the committed deliverable (verified: exit 0, 195,288 bytes)
git show HEAD:blitzy/documentation/app_2cd6ee777f8c.md | less

# Confirm provenance (verified: sole author Blitzy Agent <agent@blitzy.com>)
git log --format='%an <%ae>' -- blitzy/documentation/app_2cd6ee777f8c.md | sort -u

# Confirm net-zero change vs base (verified: single 'A' line)
git diff --name-status 2cd6ee777f8c HEAD
git status --porcelain   # expect empty (clean tree)

# Quick structure sanity (verified: 86 fences = balanced; 56 headings)
grep -c '^```' blitzy/documentation/app_2cd6ee777f8c.md   # even number => balanced
```

### 9.3 Part B — Reproduce the Three Scenarios (canonical container)

```bash
# 1) Start infrastructure
service postgresql start
service redis-server start
redis-cli -h localhost -p 6379 ping           # expect: PONG

# 2) Build the re2 dependency if needed (stack will not import without it)
pip install "cython<3" "pybind11>=2.10"
pip install pyre2==0.3.6 --no-build-isolation

# 3) Satisfy the .env prerequisite FIRST (mandatory env vars at import time)
cp example.env .env       # sets URL / EMAIL_DOMAIN / SUPPORT_EMAIL
# Without .env, `import app.config` raises KeyError: 'URL' (app/config.py:79)
# -- a DISTINCT, earlier failure from the Q1 empty-DB error. Do not conflate.
```

**Q1 — empty database → login page → exception**
```bash
PGPASSWORD=mypassword createdb -U myuser -h localhost sl_q1_obs   # DO NOT migrate
DB_URI=postgresql://myuser:mypassword@localhost:5432/sl_q1_obs python server.py &
# Open http://localhost:7777/login and submit the form (or POST /auth/login).
# Browser: HTTP 500 (generic error/500.html, 5749 bytes).
# Server stdout: sqlalchemy.exc.ProgrammingError (psycopg2.errors.UndefinedTable)
#                relation "users" does not exist   (frames: app/auth/views/login.py:43 -> app/models.py:84)
```

**Q2 — required services (migrated + initialized DB)**
```bash
PGPASSWORD=mypassword createdb -U myuser -h localhost sl_full
DB_URI=postgresql://myuser:mypassword@localhost:5432/sl_full alembic upgrade head   # 255 steps -> head 32f25cbf12f6, 77 tables
DB_URI=postgresql://myuser:mypassword@localhost:5432/sl_full python init_app.py     # seed domains

# Web (canonical) — binds 0.0.0.0:7777
gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15 &
curl -s http://localhost:7777/health            # expect: 200

# SMTP handler — binds 0.0.0.0:20381 (banner: 220 ... Python SMTP 1.4.2)
python email_handler.py &

# Background workers (no client-facing port)
python job_runner.py &
python cron.py -j delete_old_monitoring          # runs once, exit 0
python event_listener.py listener &              # "Starting to listen to events"
```

**Q3 — migrations run but init_app.py skipped → @sl.local rejection**
```bash
PGPASSWORD=mypassword createdb -U myuser -h localhost sl_probe_obs
DB_URI=postgresql://myuser:mypassword@localhost:5432/sl_probe_obs alembic upgrade head   # migrate ONLY
# Do NOT run init_app.py -> public_domain stays empty
DB_URI=postgresql://myuser:mypassword@localhost:5432/sl_probe_obs python email_handler.py &
# Inject inbound mail sender@example.com -> nonexistent@sl.local via raw-socket SMTP to :20381
# Result: 550 SL E515 Email not exist  (+ 11-line rejection chain from handle_forward)
# Alternate branch: if sender is in ignore_bounce_sender -> 250 SL E207 No bounce report
```

### 9.4 Verification

- **Web:** `curl -s http://localhost:7777/health` → `200`.
- **SMTP:** connecting to `:20381` yields the `220 … Python SMTP 1.4.2` banner.
- **Ports/PIDs:** inspect `/proc/net/tcp` and `/proc/<pid>/fd` to confirm 7777 (gunicorn master + 2 workers) and 20381 (email_handler) ownership.
- **Workers:** `job_runner` transitions a job `ready → done`; `cron` exits `0`; `event_listener` logs `Starting to listen to events`.

### 9.5 Troubleshooting

- `KeyError: 'URL'` at `app/config.py:79` → `.env` missing; run `cp example.env .env` (this is the pre-DB failure, not Q1).
- `ImportError`/`ModuleNotFoundError` for `re2` → build `pyre2==0.3.6` as in step 2.
- `connection refused` on 5432/6379 → start PostgreSQL/Redis.
- `/auth/login` POST returns 500 with `relation "users" does not exist` → **expected** for Q1 (empty, unmigrated DB); run `alembic upgrade head` to migrate for Q2/Q3.
- **Editing this document:** use byte-precise (`open('rb')`/`open('wb')`) edits to preserve the 36 verbatim CRLFs in SMTP transcripts; avoid tools that normalize CRLF→LF.

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---|---|
| View deliverable | `git show HEAD:blitzy/documentation/app_2cd6ee777f8c.md` |
| Net-zero check | `git diff --name-status 2cd6ee777f8c HEAD` |
| Start infra | `service postgresql start && service redis-server start` |
| Redis check | `redis-cli -h localhost -p 6379 ping` |
| Create DB | `PGPASSWORD=mypassword createdb -U myuser -h localhost <db>` |
| Migrate | `DB_URI=postgresql://myuser:mypassword@localhost:5432/<db> alembic upgrade head` |
| Seed domains | `python init_app.py` |
| Web (dev) | `python server.py` |
| Web (canonical) | `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` |
| SMTP handler | `python email_handler.py` |
| Job worker | `python job_runner.py` |
| Cron one-shot | `python cron.py -j <job>` |
| Event listener | `python event_listener.py listener` |

### B. Port Reference

| Port | Service | Binding | Type |
|---|---|---|---|
| 7777 | Web app (`server.py` / `gunicorn wsgi:app`) | `127.0.0.1` (dev) / `0.0.0.0` (canonical) | Client-facing (HTTP) |
| 20381 | SMTP handler (`email_handler.py`, aiosmtpd) | `0.0.0.0` | Client-facing (SMTP) |
| 5432 | PostgreSQL | localhost | Datastore |
| 6379 | Redis | localhost | Cache / sessions / limiter |
| — | `job_runner`, `cron`, `event_listener` | none | Background workers (no client port) |

### C. Key File Locations

| Path | Role |
|---|---|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The deliverable** (only net repository change) |
| `server.py` / `wsgi.py` | Web app entry (`local_main()` port 7777 / gunicorn `create_app`) |
| `app/db.py` | Import-time `engine.connect()` (`:12`) — Q1 boundary |
| `app/config.py` | Mandatory env vars (`:79,92,93`); `ALIAS_DOMAINS` |
| `app/models.py` | `SLDomain.__tablename__ = "public_domain"` (`:3116-3119`); `User` (`:337`) |
| `app/auth/views/login.py` | `/login` view (`:21-25`) — triggers Q1 query |
| `init_app.py` | `add_sl_domains()` seeding (`:39-56,69-73`) — skipped in Q3 |
| `email_handler.py` | aiosmtpd listener `:20381`; `handle_forward` E515 path |
| `app/alias_utils.py` | `try_auto_create*` returns `None` on empty `public_domain` |
| `app/email/status.py` | `E515` (`:51`), `E207` (`:12`) |
| `app/email_utils.py` | `is_reverse_alias`, `should_ignore_bounce` (`:1361-1366`) |
| `migrations/` + `alembic.ini` | 255 migration versions; `script_location = migrations` |
| `example.env` / `Dockerfile` | `.env` template; `python:3.10`, `EXPOSE 7777`, gunicorn CMD |

### D. Technology Versions

| Component | Version | Source |
|---|---|---|
| Python | 3.10 (runtime 3.10.18) | `pyproject.toml` / Dockerfile |
| Flask | 1.1.2 | `pyproject.toml` |
| SQLAlchemy | 1.3.24 | `pyproject.toml` |
| psycopg2-binary | 2.9.3 | `pyproject.toml` |
| aiosmtpd | 1.4.2 (declared `^1.2`) | runtime capture |
| gunicorn | 20.0.4 | `pyproject.toml` |
| python-dotenv | 0.14.0 | `pyproject.toml` |
| pyre2 | 0.3.6 | runtime capture |
| PostgreSQL | 15 | container |
| Redis | 6379 (running) | container |

### E. Environment Variable Reference

| Variable | Purpose | Notes |
|---|---|---|
| `URL` | Base app URL | Mandatory at import (`app/config.py:79`); missing → `KeyError` |
| `EMAIL_DOMAIN` | Primary alias domain (`sl.local`) | Mandatory (`app/config.py:92`); feeds `ALIAS_DOMAINS` |
| `SUPPORT_EMAIL` | Support contact | Mandatory (`app/config.py:93`) |
| `DB_URI` | PostgreSQL connection string | Overrides `.env` when exported inline; drives which DB is queried |
| `CONFIG` | Optional config path | Unset → `load_dotenv()` branch (`app/config.py:71`) |

### F. Developer Tools Guide

- **Git:** view, diff, and provenance commands in Appendix A / Section 9.2.
- **PostgreSQL client (`psql` / `createdb` / `dropdb`):** DB provisioning and inspection (canonical container).
- **`redis-cli`:** liveness check (`ping` → `PONG`).
- **`/proc/net/tcp` + `/proc/<pid>/fd`:** authoritative port-ownership evidence for the two listeners.
- **Raw-socket SMTP client:** the document embeds the complete injection client source used for Q3 (real inbound path, not a bypass).
- **Byte-precise editor (`open('rb')`/`open('wb')`):** required for any edit that must preserve the 36 verbatim CRLFs.

### G. Glossary

| Term | Meaning |
|---|---|
| **AAP** | Agent Action Plan — the authoritative task specification |
| **QnA document** | The runtime-verified answer document that is this project's deliverable |
| **Net-zero** | Repository left unchanged except the single new document; all temp artifacts removed |
| **`public_domain`** | Physical table backing the `SLDomain` model; seeded by `init_app.py` |
| **E515 / E207** | SMTP status constants: `550 SL E515 Email not exist` / `250 SL E207 No bounce report` |
| **Observed vs. Inferred** | Direct runtime capture vs. reading-derived claim (labeled per SWE-AtlasQnA rules) |
| **Canonical entry point** | The real invocation a normal user runs (no mocks/bypasses) |

---

*Prepared by the Blitzy autonomous assessment agent. Completion (91.67%), hours (33.0 completed / 3.0 remaining / 36.0 total), and all cross-section figures are internally consistent and validated against Sections 1.2, 2.1, 2.2, and 7.*