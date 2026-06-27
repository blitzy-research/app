# Blitzy Project Guide — SimpleLogin Normal Alias-Creation Flow Investigation

> **Repository:** SimpleLogin (`app`) · **Branch:** `blitzy-2842bc31-af72-4efc-beee-3eeae8cda64b` · **HEAD:** `6829b878` · **AAP base:** `2cd6ee777f8c`
> **Rule:** `SWE-AtlasQnA-Repo` (documentation-only) · **Deliverable:** `blitzy/documentation/app_2cd6ee777f8c.md`

---

## 1. Executive Summary

### 1.1 Project Overview

SimpleLogin is an open-source email-alias / privacy service implemented as a Python/Flask monolith. This engagement is a **documentation-only investigation** (governed by rule `SWE-AtlasQnA-Repo`) whose single deliverable is a code-grounded answer document — `blitzy/documentation/app_2cd6ee777f8c.md` — explaining exactly how SimpleLogin behaves during a normal alias-creation flow: the frontend request, the backend response, the database writes (and **how many tables are touched**), the background / follow-up work, and the failure modes. Every behavioral claim is cited to source (`file:line`) and accompanied by explicit reasoning. The audience is engineers needing an authoritative behavioral reference. **Zero source files were modified**; the investigation is strictly read-only.

### 1.2 Completion Status

```mermaid
%%{init: {"theme": "base", "themeVariables": {"pie1": "#5B39F3", "pie2": "#FFFFFF", "pieStrokeColor": "#B23AF2", "pieStrokeWidth": "2px", "pieOuterStrokeColor": "#B23AF2", "pieTitleTextColor": "#B23AF2", "pieSectionTextColor": "#B23AF2"}}}%%
pie showData title Completion — 92.5% Complete
    "Completed Work (AI)" : 37
    "Remaining Work" : 3
```

| Metric | Value |
|--------|-------|
| **Total Hours** | **40.0 h** |
| **Completed Hours (AI + Manual)** | **37.0 h** (37.0 AI + 0.0 Manual) |
| **Remaining Hours** | **3.0 h** |
| **Percent Complete** | **92.5 %** |

> Completion % follows the AAP-scoped (PA1) methodology: `Completed ÷ (Completed + Remaining) = 37 ÷ 40 = 92.5%`. All autonomous AAP-scoped work is delivered; the remaining 3.0 h is the path-to-production human review/acceptance gate.

### 1.3 Key Accomplishments

- ✅ **Single deliverable created** — a 386-line, code-grounded investigation document answering requirements R1–R6 with explicit reasoning for each.
- ✅ **★ Headline answer established** — a normal single-mailbox, non-partner dashboard alias creation writes **exactly three tables**: `alias`, `alias_audit_log`, `daily_metric`.
- ✅ **225 / 225 citations verified accurate** — every `file:line` locator confirmed byte-identical against source (independently re-corroborated during this assessment via ~19 distinct spot-checks of all headline claims).
- ✅ **All behavioral claims reproduced live** — dashboard `302`, API `201` (17-key JSON), exact 3-table write set, event-dispatcher no-op, and failure modes (`409` / `412` / `400`, dashboard duplicate → `200`).
- ✅ **Perfect scope compliance** — `git diff` vs AAP base = exactly one file added (`+386/-0`); **zero source files** added, modified, or deleted; working tree clean.
- ✅ **Rule compliance** — code-as-truth, six explicit "Why / Reasoning" subsections, correct filename and `blitzy/documentation/` placement, markdown pre-commit hook (trailing-whitespace) passes.
- ✅ **Test baseline healthy** — 30 alias-creation-specific tests pass 100%; full suite 638 passed / 1 environmental failure; coverage 71.69% (≥ 55% gate).
- ✅ **Read-only honored** — all temporary scripts removed, dev database restored to baseline, repository pristine.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| *None — no blocking issues* | The deliverable is complete, accurate, and committed; no issue blocks release or validation | — | — |
| Environmental test failure: `test_apple_process_payment` (non-blocking) | Cosmetic only — an external Apple App Store HTTPS call that times out in the offline sandbox; not a code defect, not a regression, unrelated to alias creation | Human reviewer (acknowledge) | n/a (passes with internet) |

### 1.5 Access Issues

| System / Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-------------------|----------------|-------------------|-------------------|-------|
| Sandbox outbound internet | Network (HTTPS egress) | No outbound internet in the validation sandbox; affects only the unrelated external `test_apple_process_payment` test — has **no impact** on the documentation deliverable | Accepted (non-blocking) | Human reviewer |
| Source repository | Git write | None — deliverable committed by `agent@blitzy.com`; tree clean | Resolved | — |

> No access issues block the deliverable, its validation, or its review. The single network constraint affects only one out-of-scope external test.

### 1.6 Recommended Next Steps

1. **[High]** Perform a technical review and accuracy validation of `blitzy/documentation/app_2cd6ee777f8c.md` — read all 386 lines, validate the R1–R6 answers and reasoning, and spot-check a sample of the 225 citations against source at HEAD `2cd6ee777f8c`. *(~1.5 h)*
2. **[Medium]** Apply any review feedback / minor revisions (contingency; expected near-zero since validation found no inaccuracies). *(~1.0 h)*
3. **[Low]** Confirm the document fully answers the original behavioral question, then approve/merge the PR and close out. *(~0.5 h)*

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|-------|-------------|
| R1 — Run & observe baseline | 5.0 | Provisioned the standard dev stack (canonical Docker image, PostgreSQL, Redis), seeded `john@wick.com`, documented the verbatim setup, and resolved the local-dev schema (`db.create_all()` vs Alembic) nuance |
| R2 — Frontend request (4 flows) | 3.0 | Traced all four entry points (dashboard custom/random + API custom v2/v3 + API random): endpoints, methods, content-types, payloads, decorators, and signed-suffix options |
| R3 — Backend response | 2.5 | Documented dashboard `302` + flash (PRG) and API `201` JSON; traced `serialize_alias_info_v2` to the full 17-key response shape incl. `latest_activity: null` |
| R4 — DB write accounting + table count | 5.5 | Walked `Alias.create()` and the view's unit-of-work; produced the ★3-table headline and the complete 8-row always+conditional write table; **runtime-instrumented** via SQLAlchemy statement listener |
| R5 — Background work & events | 3.5 | Analysed the triple-gated event dispatcher (no-op in dev), synchronous vs. asynchronous side effects, and the three repository-root worker processes |
| R6 — Failure-mode investigation | 5.0 | Mapped the full validation chain, `IntegrityError` rollback semantics, and three independent `429` mechanisms; discovered the dead `400` API branch; built the 18-row consolidated matrix |
| Live runtime behavioral verification | 4.0 | Reproduced every key claim in the live container (302 / 201 / 3-tables / event no-op / 409 / 412 / 400) with zero DB pollution and full cleanup |
| Citation-accuracy verification | 3.0 | Verified all 225 `file:line` citations across 26 source files (md5 source-parity) |
| Document authoring & structure | 3.0 | Authored methodology, TOC, overview, Mermaid sequence diagram, write tables, failure matrix, and summary; ensured anchors and markdown hooks pass |
| QA correction cycles | 2.5 | Four iterative refinement commits (code-accuracy, citation findings F1–F4, dashboard-duplicate → `200`, API tampered-suffix → `412`) |
| **Total Completed** | **37.0** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|-------|----------|
| Human technical review & accuracy validation of the deliverable | 1.5 | High |
| Incorporate review feedback / minor revisions (contingency) | 1.0 | Medium |
| Stakeholder acceptance & sign-off (confirm question answered, merge PR) | 0.5 | Low |
| **Total Remaining** | **3.0** | |

### 2.3 Reconciliation

- **Section 2.1 total (Completed) = 37.0 h** → equals Section 1.2 "Completed Hours".
- **Section 2.2 total (Remaining) = 3.0 h** → equals Section 1.2 "Remaining Hours" and Section 7 pie "Remaining Work".
- **2.1 + 2.2 = 37.0 + 3.0 = 40.0 h** → equals Section 1.2 "Total Hours".
- **Completion = 37.0 ÷ 40.0 = 92.5 %** → identical in Sections 1.2, 7, and 8.

---

## 3. Test Results

All tests below originate from **Blitzy's autonomous validation logs** for this project (Final Validator, run at the documented baseline).

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|------------|-------|
| Alias-creation unit/integration | pytest | 30 | 30 | 0 | (within 71.69) | The 3 alias-creation suites: `api/test_new_custom_alias.py` (10), `api/test_new_random_alias.py` (6), `dashboard/test_custom_alias.py` (14) — **100% pass** |
| Full project suite | pytest | 639 | 638 | 1 | 71.69% | Single failure = `test_apple_process_payment` (external Apple App Store HTTPS timeout in offline sandbox) — not a code defect, not a regression, unrelated to the deliverable |
| Citation accuracy | Custom md5 source-parity check | 225 | 225 | 0 | n/a | Every `file:line` locator in the deliverable verified byte-identical against source at commit `2cd6ee777f8c` |
| Runtime behavioral reproduction | Live container (Flask test client + SQLAlchemy statement listener) | 7 key claims | 7 | 0 | n/a | 302 / 201 (17 keys) / exact 3-table write set / event no-op / `409` / `412` / `400` (+ dashboard duplicate → `200`) all reproduced live |

> **Integrity note:** Coverage gate is ≥ 55%; the measured 71.69% passes. The 30 alias-creation tests — the tests most relevant to the deliverable's subject — pass at 100%.

---

## 4. Runtime Validation & UI Verification

**Runtime health (live container, Python 3.10.18 + PostgreSQL + Redis):**

- ✅ **App boots** — `create_app()` builds the Flask app and **all alias-creation routes register**.
- ✅ **Flow A (dashboard custom)** — real login + `POST /dashboard/custom_alias` → **`302`** redirect to `/dashboard/?highlight_alias_id=<id>` + flash "Alias `<email>` has been created".
- ✅ **Flow C (API custom)** — `POST /api/v3/alias/custom/new` (Authentication header) → **`201`** JSON with 17 keys (`alias` == email; `latest_activity: null`).
- ✅ **Flow D (API random)** — `POST /api/alias/random/new` → **`201`** JSON.
- ✅ **Database write accounting** — single-mailbox creation touches **exactly 3 tables**; two-mailbox touches 4 (adds `alias_mailbox`); confirmed for SL-domain and custom-domain.
- ✅ **Event dispatcher** — no-op in dev (`PartnerUser=None`, `EVENT_WEBHOOK=None`): **zero `sync_event` rows, no `NOTIFY`**.
- ✅ **Failure modes** — API duplicate → `409`; tampered/expired/malformed suffix → `412`; empty body → `400`; dashboard duplicate → inline `200` re-render (no duplicate row).

**UI verification:**

- ⚠ **No UI deliverable** — this is a documentation-only engagement; the SimpleLogin dashboard alias form is an **observed surface**, not a delivered component. The user-facing behavior (the `302` + flash and the highlighted alias) was verified at the HTTP level via the live flow above rather than via browser screenshots.

---

## 5. Compliance & Quality Review

| AAP Deliverable / Quality Benchmark | Status | Progress | Notes |
|-------------------------------------|--------|----------|-------|
| R1 — Run & observe (standard dev setup) | ✅ Pass | 100% | Verbatim setup documented; live-run reproduced |
| R2 — Frontend request (4 flows) | ✅ Pass | 100% | All endpoints/methods/payloads/decorators cited |
| R3 — Backend response (302 / 201) | ✅ Pass | 100% | Full 17-key API shape; PRG dashboard redirect |
| R4 — DB changes + table count | ✅ Pass | 100% | ★3-table headline + 8-row write table; runtime-proven |
| R5 — Background work / events | ✅ Pass | 100% | Triple-gated dispatcher no-op; workers explained |
| R6 — Failure modes | ✅ Pass | 100% | 4 categories + 18-row matrix; response/log/DB state |
| Code-grounded, no assumptions | ✅ Pass | 100% | 225/225 citations verified accurate |
| Reasoning / rationale provided | ✅ Pass | 100% | Six "Why / Reasoning" subsections |
| Zero source-file modifications | ✅ Pass | 100% | `git diff` = 1 file added, 0 source changes |
| Read-only investigation + cleanup | ✅ Pass | 100% | Temp scripts removed; dev DB restored; tree clean |
| Filename = branch + correct placement | ✅ Pass | 100% | `blitzy/documentation/app_2cd6ee777f8c.md` |
| Markdown pre-commit hook (trailing-whitespace) | ✅ Pass | 100% | 0 violations; balanced fences; ends with newline |
| Human review & acceptance | ⬜ Pending | 0% | Path-to-production gate (3.0 h) — see Section 2.2 |

**Fixes applied during autonomous validation (4 QA rounds):** (1) general code-accuracy corrections; (2) citation findings F1–F4; (3) dashboard duplicate pre-insert path corrected to HTTP `200`; (4) API tampered-suffix failure mode corrected to HTTP `412`. **Outstanding compliance items:** none beyond the human review/acceptance gate.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Citation / documentation drift — `file:line` locators may shift if the doc is rebased onto newer source (active codebase, 4,911 commits) | Technical | Low | Medium | HEAD commit pinned in the document header; re-verify citations if rebased | Mitigated |
| Environmental test failure (`test_apple_process_payment`) — external Apple HTTPS timeout offline | Technical | Low | Low | Documented expected baseline failure; not a regression; passes with internet | Accepted |
| Security exposure from the deliverable | Security | None | n/a | Read-only document adds no code, attack surface, or secrets; dev creds shown are the project's own published `CONTRIBUTING.md` / `example.env` defaults | N/A |
| Snapshot maintainability — point-in-time investigation has no automated freshness check | Operational | Low | Medium | HEAD hash pinned; treat as a versioned snapshot tied to a commit | Accepted |
| Live-run reproducibility — requires the canonical Docker image + Python 3.10 (3.12+ unsupported) + PostgreSQL + Redis | Integration | Low | Low | Canonical image provided; "code-as-truth" fallback per governing rule | Mitigated |

**Informational code observations (recorded in the deliverable; out-of-scope source changes the AAP forbids — zero project hours, optional maintainer follow-ups):**

- The API `400 "Tampered suffix"` branch is **unreachable** (`check_suffix_signature()` returns `None` rather than raising → control always reaches the `412` path).
- A **dev-server shared-session concurrency note** (truly-concurrent same-email dashboard submits can surface an `InvalidRequestError` / `500` on the threaded dev server; production Gunicorn is unaffected; no partial row is ever persisted).

---

## 7. Visual Project Status

**Project hours — completed vs. remaining** (Completed = Dark Blue `#5B39F3`, Remaining = White `#FFFFFF`):

```mermaid
%%{init: {"theme": "base", "themeVariables": {"pie1": "#5B39F3", "pie2": "#FFFFFF", "pieStrokeColor": "#B23AF2", "pieStrokeWidth": "2px", "pieOuterStrokeColor": "#B23AF2", "pieTitleTextColor": "#B23AF2", "pieSectionTextColor": "#B23AF2"}}}%%
pie showData title Project Hours Breakdown (Total 40h)
    "Completed Work" : 37
    "Remaining Work" : 3
```

**Remaining hours by category** (sums to the 3.0 h Remaining total):

```mermaid
%%{init: {"theme": "base", "themeVariables": {"pie1": "#5B39F3", "pie2": "#B23AF2", "pie3": "#A8FDD9", "pieTitleTextColor": "#B23AF2", "pieSectionTextColor": "#000000"}}}%%
pie showData title Remaining Work by Category (3h)
    "Technical review (High)" : 1.5
    "Revision contingency (Medium)" : 1.0
    "Acceptance & sign-off (Low)" : 0.5
```

> **Integrity check:** the pie "Remaining Work" = 3 h equals Section 1.2 Remaining Hours and the Section 2.2 "Hours" total; "Completed Work" = 37 h equals Section 2.1 total.

---

## 8. Summary & Recommendations

**Achievements.** The engagement delivered exactly what the AAP scoped: a single, comprehensive, **code-grounded** investigation document (`blitzy/documentation/app_2cd6ee777f8c.md`, 386 lines) that authoritatively answers how SimpleLogin behaves during a normal alias-creation flow. The ★headline result — **a normal single-mailbox dashboard creation writes exactly three tables (`alias`, `alias_audit_log`, `daily_metric`)** — is both code-derived and runtime-proven. All 225 citations are verified accurate, and every key behavioral claim was reproduced live.

**Remaining gaps.** None in the autonomous scope. The only remaining work is the **path-to-production human review/acceptance gate** (3.0 h): a technical review, a small revision contingency, and stakeholder sign-off.

**Critical path to production.** Review the document → spot-check citations → apply any feedback → approve and merge. There are no engineering blockers, no source changes required, and no infrastructure to provision for the deliverable itself.

**Production readiness.** The project is **92.5 % complete** on an AAP-scoped basis. The deliverable is production-ready: it is accurate, comprehensive, rule-compliant, and committed against a pristine working tree, with all five autonomous production-readiness gates passing. The single environmental test failure is unrelated and non-blocking.

| Success Metric | Target | Actual |
|----------------|--------|--------|
| AAP requirements answered (R1–R6) | 6 / 6 | 6 / 6 ✅ |
| Citation accuracy | 100% | 225 / 225 ✅ |
| Source files modified | 0 | 0 ✅ |
| Alias-creation tests passing | 100% | 30 / 30 ✅ |
| Reasoning per requirement | Required | 6 / 6 ✅ |
| AAP-scoped completion | — | 92.5% |

---

## 9. Development Guide

This project is documentation-only. The guide below covers **(A)** reproducing the live investigation by running SimpleLogin, and **(B)** accessing and verifying the deliverable document. Commands marked *(verified)* were executed during this assessment.

### 9.1 System Prerequisites

- **Python 3.10** + **Poetry** — the project pins `python = "^3.10"` and explicitly notes Python 3.12+ does **not** work (`CONTRIBUTING.md`).
- **PostgreSQL 13+** (primary datastore).
- **Redis** *(optional)* — backs the two custom rate-limiters and `LISTEN`/`NOTIFY`; the app degrades gracefully without it.
- **Node v10** *(frontend assets only)*.
- **Canonical environment:** Docker image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (ships Python 3.10 + Poetry-locked deps + PostgreSQL).

### 9.2 Environment Setup

```bash
# 1. Create a local settings file from the template
cp example.env .env

# 2. Edit DB_URI in .env to point at your Postgres port.
#    (CONTRIBUTING.md sample uses :35432; example.env default uses :5432;
#     match it to the port your container actually exposes — see step 3.)
#    DB_URI=postgresql://myuser:mypassword@localhost:15432/simplelogin

# 3. Start PostgreSQL (exposes host port 15432)
docker run -e POSTGRES_PASSWORD=mypassword -e POSTGRES_USER=myuser \
  -e POSTGRES_DB=simplelogin -p 15432:5432 postgres:13
```

### 9.3 Dependency Installation

```bash
# Install all pinned dependencies (flask 1.1.2, SQLAlchemy 1.3.24, redis, psycopg2-binary, arrow, ...)
poetry install
```

### 9.4 Application Startup

```bash
# Bring up the schema, seed dev data, and start the web server (CONTRIBUTING.md, verbatim)
alembic upgrade head && flask dummy-data && python3 server.py
# Then open http://localhost:7777 and log in with:  john@wick.com / password
```

### 9.5 Verification Steps

```bash
# (verified) Confirm the deliverable exists and its size
wc -l blitzy/documentation/app_2cd6ee777f8c.md          # -> 386

# (verified) Confirm scope: exactly ONE file added vs the AAP base, zero source changes
git diff --name-status 2cd6ee777f8c HEAD                # -> A  blitzy/documentation/app_2cd6ee777f8c.md

# (verified) Confirm a pristine working tree
git status --porcelain                                  # -> (empty)

# (verified) Confirm citation density and markdown hygiene
grep -oE '([a-zA-Z0-9_./]+\.(py|md|env|toml)):[0-9]+' \
  blitzy/documentation/app_2cd6ee777f8c.md | wc -l      # -> 225
grep -cE ' +$' blitzy/documentation/app_2cd6ee777f8c.md # -> 0 (trailing-whitespace hook)
```

### 9.6 Example Usage (observe the flow)

```bash
# Web (Flow A): create a single-mailbox custom alias from the dashboard form
#   -> expect HTTP 302 redirect + flash, and EXACTLY 3 tables written
#      (alias, alias_audit_log, daily_metric)  — see deliverable §R4

# API (Flow C): create a custom alias programmatically
curl -s -X POST 'http://localhost:7777/api/v3/alias/custom/new' \
  -H 'Authentication: <API_KEY>' -H 'Content-Type: application/json' \
  -d '{"alias_prefix":"demo","signed_suffix":"<signed>","mailbox_ids":[<id>]}'
#   -> expect HTTP 201 + JSON alias object (17 keys; latest_activity: null)
```

### 9.7 Troubleshooting

- **App won't start on Python 3.12/3.13** — the project supports only Python 3.10; use the canonical Docker image. Where a live run is not reproducible, the document's verified source citations are the authoritative substitute (per the governing rule).
- **DB connection refused** — ensure `DB_URI`'s host port matches the port your Postgres container exposes (the `CONTRIBUTING.md` docker command maps host **15432**; the sample `DB_URI` line shows 35432 and `example.env` shows 5432 — a known minor doc inconsistency; align them).
- **No Redis** — expected; the two custom limiters no-op and Flask-Limiter falls back to in-memory storage, so alias creation still proceeds.
- **`test_apple_process_payment` fails** — expected offline (external Apple HTTPS call); it is not a regression and is unrelated to alias creation.

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---------|---------|
| Copy env template | `cp example.env .env` |
| Start PostgreSQL | `docker run -e POSTGRES_PASSWORD=mypassword -e POSTGRES_USER=myuser -e POSTGRES_DB=simplelogin -p 15432:5432 postgres:13` |
| Install dependencies | `poetry install` |
| Migrate + seed + run | `alembic upgrade head && flask dummy-data && python3 server.py` |
| Run full test suite | `pytest -c pytest.ci.ini` |
| Run alias tests only | `pytest tests/api/test_new_custom_alias.py tests/api/test_new_random_alias.py tests/dashboard/test_custom_alias.py` |
| Verify deliverable diff | `git diff --name-status 2cd6ee777f8c HEAD` |
| Check tree is clean | `git status --porcelain` |

### B. Port Reference

| Service | Port | Notes |
|---------|------|-------|
| Web dashboard / API (`server.py`) | 7777 | `http://localhost:7777` |
| PostgreSQL (host-mapped) | 15432 → 5432 | Per the `CONTRIBUTING.md` docker command |
| Redis | 6379 | Optional (rate-limit buckets, parallel lock, `LISTEN`/`NOTIFY`) |

### C. Key File Locations

| Path | Role |
|------|------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The deliverable** (386 lines) |
| `app/models.py` | ORM; `Alias.create()` (`:1628`), the central write path |
| `app/dashboard/views/custom_alias.py` | Flow A — web custom alias (route `:30`, commit `:158`) |
| `app/dashboard/views/index.py` | Flow B — one-click random alias |
| `app/api/views/new_custom_alias.py` | Flow C — API custom alias (v2 + v3) |
| `app/api/views/new_random_alias.py` | Flow D — API random alias |
| `app/api/serializer.py` | `serialize_alias_info_v2` — 201 JSON shape |
| `app/events/event_dispatcher.py` | Triple-gated event dispatch |
| `app/alias_audit_log_utils.py` | Unconditional audit-log write |
| `app/rate_limiter.py` / `app/parallel_limiter.py` | `429` rate-limit / concurrency mechanisms |
| `CONTRIBUTING.md` / `docs/api.md` / `example.env` | Setup + API reference + default config |

### D. Technology Versions

| Component | Version | Source |
|-----------|---------|--------|
| Python | 3.10 (`^3.10`; 3.12+ unsupported) | `pyproject.toml`, `CONTRIBUTING.md` |
| Flask | 1.1.2 | `pyproject.toml` / `poetry.lock` |
| SQLAlchemy | 1.3.24 | `pyproject.toml` |
| psycopg2-binary | 2.9.x | `pyproject.toml` |
| redis (client) | 4.x | `pyproject.toml` |
| arrow | 0.16.x | `pyproject.toml` |
| PostgreSQL | 13+ | `CONTRIBUTING.md` |
| Node | v10 (frontend) | `CONTRIBUTING.md` |

### E. Environment Variable Reference

| Variable | Default | Relevance |
|----------|---------|-----------|
| `DB_URI` | `postgresql://myuser:mypassword@localhost:5432/simplelogin` (`example.env`) | Database connection; edit per local Postgres port |
| `EVENT_WEBHOOK` | `None` (`app/config.py:612`; absent from `example.env`) | Gates event dispatch — `None` in dev → event path is a no-op |
| `EVENT_WEBHOOK_DISABLE` | unset | Hard-disables event dispatch when set |
| `ALIAS_LIMIT` | `100/day;50/hour;5/minute` (`app/config.py:448`) | Flask-Limiter HTTP-window limit (`429`) |
| `ALIAS_CREATE_RATE_LIMIT_FREE` / `_PAID` | `10,900:50,3600` / `50,900:200,3600` | Per-user Redis token-bucket limits (`429`) |
| `MEM_STORE_URI` | `None` (`app/config.py:568`) | Redis URI for Flask-Limiter storage; in-memory fallback when unset |
| `DISABLE_RATE_LIMIT` | unset | Bypasses Flask-Limiter entirely |
| `FLASK_SECRET` | `secret` (`example.env:77`) | Session signing |

### F. Developer Tools Guide

| Tool | Use |
|------|-----|
| `pytest` (+ `pytest.ci.ini`, `coverage.ini`) | Test execution with coverage (gate ≥ 55%; baseline 71.69%) |
| `alembic` | Schema migrations (deploy/test path) |
| `flask dummy-data` | Seeds dev user `john@wick.com` and sample data via `fake_data()` |
| `poetry` | Dependency management against `poetry.lock` |
| `git diff --name-status <base> HEAD` | Confirm the single-file, zero-source-change footprint |
| `scripts/run-test.sh` | Spins up a `postgres:13` container for the test harness |

### G. Glossary

| Term | Meaning |
|------|---------|
| **AAP** | Agent Action Plan — the governing project specification |
| **Flow A / B / C / D** | The four alias-creation entry points (dashboard custom / dashboard random / API custom / API random) |
| **PRG** | Post/Redirect/Get — the dashboard's `302`-redirect-after-POST pattern |
| **Triple-gated dispatch** | Event dispatch suppressed unless webhook enabled **and** configured **and** user is a Proton partner |
| **`sync_event` / `NOTIFY`** | Event-sourcing row + PostgreSQL notification — written only when all event gates pass (not in dev) |
| **Unit of work** | `Session.add` → `flush` → single `commit` (with `rollback` on error) — the all-or-nothing transaction model |
| **Path-to-production** | Standard activities to move a verified deliverable to acceptance — here, human review/sign-off |
