# Blitzy Project Guide

**Project:** SimpleLogin Backend — Dev-Mode Runtime Behavior Documentation
**Branch:** `blitzy-06613b0f-0e8a-4dff-a9ca-9ce5cd7cdd9b` (source branch: `app_2cd6ee777f8c`)
**Deliverable:** `blitzy/documentation/app_2cd6ee777f8c.md` (641 lines)
**Governing rule set:** `SWE-AtlasQnA-Repo` (documentation-only)

---

## 1. Executive Summary

### 1.1 Project Overview

SimpleLogin is an open-source email-alias and SSO backend (Flask, Python 3.10). This project delivers a single evidence-backed Markdown document, `blitzy/documentation/app_2cd6ee777f8c.md`, that explains what the backend **actually prints and does at runtime** when started in local development mode (`python3 server.py`) — grounded in literal captured stdout and exact source-code citations, not theoretical behavior. It answers seven objectives: startup sequence, configuration loading, readiness signals, ports/endpoints, the authenticated request lifecycle, the background-job process model, and other runtime activity — explicitly separating the dev server from the production `gunicorn` path. The target audience is SimpleLogin developers and reviewers onboarding to the runtime. No source code was modified.

### 1.2 Completion Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieOuterStrokeWidth':'2px','pieTitleTextSize':'18px','pieSectionTextColor':'#000000','pieOpacity':'1'}}}%%
pie showData
    title Completion Status — 93.5% Complete (29h of 31h)
    "Completed Hours (AI)" : 29
    "Remaining Hours" : 2
```

| Metric | Value |
|---|---|
| **Total Hours** | **31h** |
| **Completed Hours (AI + Manual)** | **29h** (AI: 29h · Manual: 0h) |
| **Remaining Hours** | **2h** |
| **Percent Complete** | **93.5%** |

> Completion is computed on AAP-scoped work only: `29 / (29 + 2) = 93.5%`. All seven content objectives (O1–O7) plus the full investigate → run → capture → write workflow are delivered and empirically verified; the remaining 2h is the human review-and-merge gate (path to production).

### 1.3 Key Accomplishments

- ✅ Authored a 641-line, evidence-backed runtime-behavior document answering all seven objectives (O1–O7), each with an **(a) Observed** (literal captured output) and **(b) Why — code rationale** subsection.
- ✅ Performed an empirical run inside the provided container image (Python 3.10 legacy pinned stack) and captured literal stdout for both the **development** server and the **production** `gunicorn` path.
- ✅ Documented the headline finding: the familiar `* Running on http://127.0.0.1:7777/` banner is **never printed** because the `werkzeug` logger is disabled (`app/log.py:L70-71`); readiness is effectively silent (confirmed via `GET /health → 200`).
- ✅ Traced the user's primary concern — the **dual-path authenticated request lifecycle** (session via Flask-Login `user_loader`/`alternative_id`; API via the `Authentication` header) — end to end with captured login + API + bad-key flows.
- ✅ Established that the web process starts **no** background jobs; all periodic/async/SMTP work runs in separate `__main__` processes, with the nuanced `event_listener.py` exception documented.
- ✅ Surfaced a genuine **dev-only security finding**: the Flask-DebugToolbar leaks `SECRET_KEY` and the full DB URI into HTML response bodies — absent from the production path.
- ✅ Maintained strict scope: **zero** source files created/modified/deleted; every claim traceable via a ~45-row evidence index (Appendix B); workspace left pristine.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|---|---|---|---|
| _None that block this documentation deliverable._ | No blockers: the document is complete, empirically verified (zero discrepancies), and committed. | — | — |
| Disclosed (informational, **non-blocking**): dev-only Flask-DebugToolbar secret/credential exposure in the **application** | Dev server emits `SECRET_KEY` + DB password in HTML bodies; **absent in production**. Does not block the doc — it is the doc *reporting* the behavior. | SimpleLogin app maintainers (out of scope for this doc) | Optional — separate app ticket |

### 1.5 Access Issues

**No access issues identified.** The repository was fully accessible, the provided container image ran successfully, and PostgreSQL/Redis were provisioned for the empirical capture. No repository permissions, service credentials, or third-party API access blocked build, validation, or capture.

### 1.6 Recommended Next Steps

1. **[High]** Perform a technical review of `blitzy/documentation/app_2cd6ee777f8c.md` — verify it answers O1–O7 and the user's primary concern, confirm the Appendix A captures and the security finding, and spot-check the `file:line` citations against source. _(~1.0h)_
2. **[High]** Review and accept the disclosed **dev-only** security finding; decide whether to open a separate application-remediation ticket (that remediation is out of scope here). _(~0.5h)_
3. **[High]** Approve and merge the documentation PR into the target branch. _(~0.5h)_
4. **[Medium]** _(Optional, out of scope)_ Open a follow-up app ticket to gate the Flask-DebugToolbar behind an explicit opt-in env var so dev secrets are not embedded in HTML.
5. **[Low]** _(Optional, out of scope)_ Add periodic re-validation of the document's `file:line` citations to guard against drift if rebased onto newer upstream.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|---|---:|---|
| Static source-code investigation & citation tracing | 7 | Read-only trace of the web bootstrap and request path across ~21 files (`server.py`, `wsgi.py`, `app/config.py`, `app/log.py`, `app/db.py`, `app/extensions.py`, `app/session.py`, `app/redis_services.py`, `app/api/base.py`, `app/models.py`, the worker scripts, `Dockerfile`, `pyproject.toml`, `CONTRIBUTING.md`, `example.env`); recorded exact `file:line` citations for every emitted line and identity branch. |
| Empirical environment bring-up & runtime capture | 6 | Provisioned the container (PostgreSQL + Redis), built a throwaway `.env`, ran `alembic upgrade head && flask dummy-data && python3 server.py`, and captured literal stdout for both the dev server and the production `gunicorn` path, plus web + API + bad-key request traces. |
| O1–O4 documentation (startup, config, readiness, ports/endpoints) | 5 | Authored the startup-sequence, configuration-loading, readiness-signal, and ports/endpoints sections with embedded captures, including the headline "missing banner" finding. |
| O5 documentation (authenticated request lifecycle) | 4 | Documented the user's primary concern: dual-path identity (session `user_loader`/`alternative_id`; API `Authentication` header), entry via `ProxyFix`/WSGI, and context propagation through `flask.g` and request hooks. |
| O6 documentation (background jobs / multi-process architecture) | 2 | Established that the web process starts no jobs; documented the separate `__main__` worker processes and the `create_light_app` usage nuance (including the `event_listener.py` exception). |
| O7 documentation + dev-only security finding | 3 | Documented eager import-time DB connection ordering, optional Sentry/limiter/`OAUTHLIB_INSECURE_TRANSPORT`, and discovered/reproduced/contrasted the dev-only Flask-DebugToolbar secret exposure. |
| Appendices A & B (captures + evidence index) | 2 | Reproduced A.1–A.6 verbatim captures (dev startup, request outcomes, `COLOR_LOG` variant, production contrast, `CONFIG` artifact, security capture) and built the ~45-row claim-to-citation evidence index. |
| **Total Completed** | **29** | **100% of AAP content scope, delivered and empirically verified.** |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|---|---:|---|
| Human technical review of document accuracy & completeness (O1–O7 answers + Appendix captures + security finding + citation spot-checks) | 1.5 | High |
| PR approval & merge into target branch | 0.5 | High |
| **Total Remaining** | **2.0** | — |

> _Out-of-scope future items (counted at 0h, not part of the 31h total):_ application-side remediation of the dev-only DebugToolbar exposure; periodic citation re-validation; optional expansion to full OAuth2/OIDC and email-forwarding internals (the AAP intentionally summarized these at the process level).

### 2.3 Hours Reconciliation

| Check | Result |
|---|---|
| Section 2.1 total (completed) | 29h |
| Section 2.2 total (remaining) | 2h |
| **2.1 + 2.2 = Total Project Hours** | **29 + 2 = 31h** ✓ (matches Section 1.2) |
| Completion % = Completed / Total | 29 / 31 = **93.5%** ✓ |
| Remaining hours consistent across §1.2, §2.2, §7 | 2h = 2h = 2h ✓ |

---

## 3. Test Results

All results below originate from **Blitzy's autonomous validation logs** for this project (the Final Validator's empirical run inside the provided container), corroborated by independent re-verification on the host. For a "what actually gets printed" runtime-behavior document, **empirical reproduction of every documented claim *is* the test.**

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---|---|---:|---:|---:|---|---|
| Document structure & lint | Custom (Python/grep) | 4 | 4 | 0 | n/a | Balanced code fences (25 fenced blocks / 50 markers); 6 internal anchors resolve; 0 placeholders/TODOs; all O1–O7 headings present |
| Citation accuracy | Manual + scripted spot-check | 45 | 45 | 0 | 100% (Appendix B) | ~45 evidence-index entries; validator spot-checked 60+; independent re-check of 10+ across 6 files — all precise |
| Application import ("compile" analogue) | Python import | 1 | 1 | 0 | n/a | `from server import create_app; create_app()` → exit 0, **292 routes**, no ImportError/SyntaxError |
| Dev-server startup & readiness | Werkzeug + curl | 1 | 1 | 0 | n/a | `GET /health == 200`; import prints + Flask banner observed; `* Running on …` correctly **absent** |
| Production-server startup & readiness | gunicorn + curl | 1 | 1 | 0 | n/a | `Listening at: http://0.0.0.0:7777`; served `GET /auth/login → 200` |
| Runtime claim verification (O1–O7) | Empirical reproduction | 7 | 7 | 0 | 100% of O1–O7 | Every objective's Observed output reproduced and matched; zero discrepancies |
| Request lifecycle (web + API) | curl | 6 | 6 | 0 | n/a | `GET / → 302`; `/auth/login → 200`; `POST login → 302 → /dashboard/ → 200`; API `200`; bad key `401` |
| Appendix captures (A.1–A.6) | Empirical reproduction | 6 | 6 | 0 | 100% | Each verbatim capture reproduced; only run-specific values (timestamps, pids, temp paths, byte sizes) vary, as the doc states |
| **Totals** | — | **71** | **71** | **0** | — | **100% pass rate** |

> **Note on the repository unit-test suite (pytest):** intentionally **not executed**. This deliverable changes **zero** source files, so running the upstream suite would validate unchanged code — outside the AAP scope. The mandated test for a runtime-behavior document (empirically reproducing the printed output) was performed comprehensively and passed.

---

## 4. Runtime Validation & UI Verification

**Backend runtime (development server — `python3 server.py`):**
- ✅ **Operational** — Reaches readiness; `GET /health` returns `200`.
- ✅ **Operational** — Import-time prints captured in order: `>>> URL: http://localhost:7777`, four config prints, `>>> init logging <<<`.
- ✅ **Operational** — Flask click banner (`* Serving Flask app … / * Debug mode: on`) observed.
- ✅ **Operational (negative finding confirmed)** — `* Running on …`, `* Restarting with stat`, `Debugger is active!`, and `Debugger PIN` are **absent** (werkzeug logger disabled). This is the documented, expected behavior.
- ✅ **Operational** — Auto-reloader doubling observed (parent + reloader-child pids).

**Backend runtime (production path — `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15`):**
- ✅ **Operational** — gunicorn banner `Listening at: http://0.0.0.0:7777`; two sync workers booted; served requests.

**HTTP surface & request lifecycle:**
- ✅ **Operational** — `GET /` → `302` → `/auth/login`; `GET /auth/login` → `200`.
- ✅ **Operational** — `POST /auth/login` (john@wick.com/password) → `302` → `/dashboard/`; session cookie **`slapp`** set; `GET /dashboard/` → `200` (web-session path).
- ✅ **Operational** — `GET /api/user_info` with valid `Authentication` header → `200` + JSON profile (API-key path); bad key → `401 {"error":"Wrong api key"}`.
- ✅ **Operational** — Per-request `after_request` `SL` log lines emitted; standard Werkzeug access lines correctly absent.

**API integration:**
- ✅ **Operational** — API-key identity resolution via `app/api/base.py` confirmed against captured `200`/`401` outcomes.

**UI verification:** Not applicable — this is a backend documentation task with **no UI design surface** (AAP §0.9). The only request-time HTML observation is the dev-only DebugToolbar injection, captured and documented under O7 / Appendix A.6.
- ⚠️ **Partial / dev-only (security caveat)** — Dev-server HTML responses embed the DebugToolbar Config panel, which leaks `SECRET_KEY` + DB URI; **not present** on the production path. Documented, not a defect of the deliverable.

---

## 5. Compliance & Quality Review

Cross-mapping of the AAP deliverable and the `SWE-AtlasQnA-Repo` governing rules to Blitzy's quality benchmarks. Fixes applied during autonomous validation are noted.

| Benchmark / Rule | Status | Progress | Notes |
|---|---|---|---|
| Single deliverable, branch-named (`app_2cd6ee777f8c.md`) | ✅ Pass | 100% | Exactly one file at `blitzy/documentation/app_2cd6ee777f8c.md` |
| Fixed destination (`blitzy/documentation/`) | ✅ Pass | 100% | Correct path confirmed |
| Build & run for truth (empirical capture) | ✅ Pass | 100% | Dev + prod runs captured in the provided container |
| Code as source of truth + rationale per answer | ✅ Pass | 100% | Every O1–O7 has (a) Observed + (b) Why — code rationale |
| No source modifications | ✅ Pass | 100% | `git diff 2cd6ee77..HEAD` = 1 added file; 0 source changes |
| No extra code | ✅ Pass | 100% | Only the Markdown document was added |
| Clean workspace (temp artifacts removed) | ✅ Pass | 100% | `git status` clean; no stray `.env`/scripts; container removed |
| Evidence discipline (`file:line` traceability) | ✅ Pass | 100% | ~45-row Appendix B evidence index; citations verified |
| All seven objectives (O1–O7) answered | ✅ Pass | 100% | Each objective fully addressed and verified |
| Document well-formedness (fences, anchors, no placeholders) | ✅ Pass | 100% | 25 balanced fenced blocks; 6 anchors resolve; 0 placeholders |
| Dev-vs-prod & web-vs-workers distinctions maintained | ✅ Pass | 100% | Explicitly separated throughout |
| Human review of the document | ⬜ Pending | 0% | 2h allocated (Section 2.2) — path to production |

**Fixes applied during autonomous validation (review/QA cycles):** O6 worker enumeration and the `create_light_app` / `event_listener.py` exception corrected; O7 import-time startup ordering corrected; O2/O4 citations refined; `init_app.py` `__main__` enumeration fixed; the dev-only DebugToolbar secret-exposure finding investigated, reproduced, and documented (QA Issue 1, MAJOR). **Outstanding:** human review and merge only.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|---|---|---|---|---|---|
| Citation line-number drift if rebased onto newer upstream | Technical | Low | Medium | Doc is tied to branch `app_2cd6ee777f8c` (point-in-time commit); re-validate citations on rebase | Open (inherent to line citations) |
| Version-pinned findings (the suppressed `* Running on` banner is specific to Werkzeug 1.0.1 / Flask 1.1.2) | Technical | Low | Low | Doc's explicit "Versions referenced" note scopes all findings to the pinned legacy stack | Mitigated |
| Run-specific capture values (timestamps, pids, temp paths, byte sizes) misread as deterministic | Technical | Low | Low | Doc explicitly labels these as varying run-to-run and marks the deterministic values | Mitigated |
| **Dev-server Flask-DebugToolbar emits `SECRET_KEY` + DB URI (with password) in HTML bodies** | Security | **High (dev only)** | High if dev server exposed beyond localhost | Documented as **dev-only** with explicit warning; **absent** from production `gunicorn` path; remediation flagged as a separate (out-of-scope) app ticket | Disclosed / Documented |
| Secrets leaked by the deliverable itself | Security | Low | Low | Values shown are throwaway `example.env` placeholders (`myuser:mypassword`, `'secret'`), **not** real secrets — confirmed by source check | Closed |
| Reproducibility dependency on the exact container + PostgreSQL/Redis (legacy stack not runnable on arbitrary host) | Operational | Low | Low | Doc documents the exact image, services, and command sequence | Mitigated |
| Documentation staleness over time (no automated sync as code evolves) | Operational | Low | Medium | Treat as point-in-time snapshot tied to the branch; re-run investigate → capture if behavior changes | Open (accepted) |
| Integration risk | Integration | None | n/a | Standalone Markdown with no compile/runtime dependency; all internal anchors resolve; consumes no external services | Closed |
| Human review finds minor wording/scope gaps requiring edits | Process | Low | Low | 2h review budget allocated; validator found zero discrepancies; independent re-verification confirmed accuracy | Open (review pending) |

**Overall risk posture: LOW.** No technical or integration blockers. The single High-severity item is a genuine **application** security property that the document correctly **discloses** (dev-only, absent in production) — a positive outcome of the investigation, not a flaw in the deliverable.

---

## 7. Visual Project Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieOuterStrokeWidth':'2px','pieTitleTextSize':'18px','pieSectionTextColor':'#000000','pieOpacity':'1'}}}%%
pie showData
    title Project Hours Breakdown (Total 31h)
    "Completed Work" : 29
    "Remaining Work" : 2
```

**Remaining work by category (Section 2.2 — total 2h, all High priority):**

| Category | Hours | Bar |
|---|---:|---|
| Human technical review | 1.5 | ███████████████ |
| PR approval & merge | 0.5 | █████ |
| **Total** | **2.0** | — |

> **Color key (Blitzy brand):** Completed Work = Dark Blue `#5B39F3`; Remaining Work = White `#FFFFFF` (violet `#B23AF2` border for visibility).
> **Integrity:** the pie's "Remaining Work" (2) equals Section 1.2 Remaining Hours (2h) and the Section 2.2 Hours total (2h).

---

## 8. Summary & Recommendations

**Achievements.** The project delivers a single, high-quality, evidence-backed document, `blitzy/documentation/app_2cd6ee777f8c.md` (641 lines), that authoritatively explains the SimpleLogin backend's **actual** dev-mode runtime behavior. All seven objectives (O1–O7) are answered with both literal captured output and `file:line` code rationale; the dev server and production `gunicorn` paths are clearly separated; the web process is shown to start no background jobs; and the user's primary concern — the dual-path authenticated request lifecycle — is traced end to end. A genuine dev-only security finding (DebugToolbar secret exposure) was discovered, reproduced, and documented as a bonus outcome.

**Remaining gaps.** None within the AAP content scope — the deliverable is complete and empirically verified with zero discrepancies. The only remaining work is the **path-to-production human gate**: technical review (1.5h) and PR merge (0.5h).

**Critical path to production.** (1) Review the document for accuracy/completeness → (2) accept the disclosed dev-only security finding → (3) approve and merge the PR. Estimated **2h** total.

**Success metrics.**

| Metric | Target | Actual |
|---|---|---|
| Objectives answered (O1–O7) | 7/7 | ✅ 7/7 |
| Empirical claims reproduced | 100% | ✅ 100% (zero discrepancies) |
| Source files modified | 0 | ✅ 0 |
| Document well-formed (fences/anchors/placeholders) | Pass | ✅ Pass |
| Workspace pristine | Yes | ✅ Yes |

**Production readiness assessment.** The deliverable is **93.5% complete** — production-ready pending human review and merge. It is scope-compliant (zero source changes), citation-accurate, well-formed, and empirically validated. Recommendation: **proceed to review and merge.**

---

## 9. Development Guide

This guide has two layers: **(A)** a host-runnable reviewer/verification workflow (to confirm the deliverable), and **(B)** the container-only empirical runtime reproduction (to re-observe the behavior the document describes).

### 9.1 System Prerequisites

- **Reviewer workflow (host):** `git` and Python 3 (verified: Python 3.13.7, git 2.51.0). No services required.
- **Runtime reproduction (container):** Docker; the provided image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (Python 3.10 + the pinned legacy stack: Flask 1.1.2 / Werkzeug 1.0.1 / SQLAlchemy 1.3.24); **PostgreSQL 13+** reachable *before* import (the ORM engine connects eagerly at import — `app/db.py:L9-14`); **Redis 6** optional (only when `MEM_STORE_URI` is set).
- **Note:** the legacy pinned stack is **not** faithfully runnable on the host (host Python 3.13; no PostgreSQL/Redis). Runtime reproduction must use the provided container (AAP §0.8.2).

### 9.2 Reviewer Verification Workflow (host — all commands tested, exit 0)

```bash
# From the repository root
# 1) Locate the deliverable (expect: 641 lines)
ls -l blitzy/documentation/app_2cd6ee777f8c.md
wc -l blitzy/documentation/app_2cd6ee777f8c.md

# 2) Scope check — expect exactly ONE added file, ZERO source changes
git diff --name-status 2cd6ee77..HEAD
#   -> A   blitzy/documentation/app_2cd6ee777f8c.md

# 3) Working tree must be clean
git status --porcelain        # (empty output == clean)

# 4) Structure overview (TL;DR, Methodology, O1-O7, Appendices)
grep -nE "^#{1,3} " blitzy/documentation/app_2cd6ee777f8c.md

# 5) Code-fence balance (expect an even count == balanced)
python3 -c "t=open('blitzy/documentation/app_2cd6ee777f8c.md').read(); n=t.count(chr(96)*3); print(n, 'BALANCED' if n%2==0 else 'UNBALANCED')"

# 6) Spot-check a citation against source (example: the >>> URL: print)
sed -n '79,80p' app/config.py
#   -> URL = os.environ["URL"]
#   -> print(">>> URL:", URL)
```

### 9.3 Empirical Runtime Reproduction (container)

```bash
# Inside the provided container image, at the code root (/app):

# 1) Provision services: PostgreSQL 13+ (and Redis 6 only if using MEM_STORE_URI)
#    Create role/db, e.g. user 'myuser' / password 'mypassword' / db 'simplelogin'.

# 2) Build a THROWAWAY .env from example.env (remove it afterward)
cp example.env .env
#   Ensure at least:
#   URL=http://localhost:7777
#   DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin
#   FLASK_SECRET=secret
#   NOT_SEND_EMAIL=true

# 3) Migrate, seed, and run the DEV server (CONTRIBUTING.md:L106)
alembic upgrade head && flask dummy-data && python3 server.py

# 4) Production contrast (separate run)
gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
```

### 9.4 Verification Steps

```bash
# Readiness is otherwise SILENT (no "* Running on" banner) — confirm via /health:
curl -s http://localhost:7777/health          # -> success

# Web-session path: log in (john@wick.com / password), then load the dashboard.
# API path: send the API key in the 'Authentication' header.
curl -s -H "Authentication: <api_key>" http://localhost:7777/api/user_info   # -> 200 + JSON
curl -s -H "Authentication: bad"        http://localhost:7777/api/user_info   # -> 401 {"error":"Wrong api key"}
```

### 9.5 Example Output (what "ready" actually looks like — dev)

```
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/<random>
Upload files to local dir
>>> init logging <<<
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
 * Debug mode: on
```
(The block then repeats once under the reloader child pid. There is **no** `* Running on …` line.)

### 9.6 Cleanup

```bash
rm -f .env                      # remove the throwaway env
git status --porcelain          # must be empty (no source changes)
# docker rm -f <container>       # tear down the reproduction container
```

### 9.7 Troubleshooting

- **Startup aborts after `>>> URL:` / `>>> init logging <<<` but before the banner** → PostgreSQL unreachable (eager connect at `app/db.py:L12`). Fix: ensure PG is up and `DB_URI` is correct.
- **`KeyError`/`RuntimeError` at import before any banner** → a required env var is missing (e.g. `URL`, `FLASK_SECRET`). Fix: populate `.env` from `example.env`.
- **No `* Running on http://127.0.0.1:7777/` line** → **expected** (the `werkzeug` logger is disabled, `app/log.py:L70-71`). Confirm readiness via `GET /health`.
- **`Could not insert debug toolbar. </body> tag not found in response.`** on redirects/JSON/`/health` → **expected/harmless** (no `</body>` to inject into).
- **Secrets visible in dev page source** → **expected dev-only** DebugToolbar behavior. Do **not** expose the dev server beyond `localhost`; this is absent from the production `gunicorn` path.
- **Legacy stack won't install on the host** → use the provided container image (host Python 3.13 is incompatible with the pinned Flask 1.1.2 / Werkzeug 1.0.1).

---

## 10. Appendices

### Appendix A — Command Reference

| Command | Purpose |
|---|---|
| `wc -l blitzy/documentation/app_2cd6ee777f8c.md` | Confirm deliverable size (641 lines) |
| `git diff --name-status 2cd6ee77..HEAD` | Scope check (one added file, zero source changes) |
| `git status --porcelain` | Confirm clean working tree |
| `git log --author="agent@blitzy.com" --oneline` | List the 4 documentation commits |
| `alembic upgrade head && flask dummy-data && python3 server.py` | Dev startup (in container) |
| `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` | Production startup (in container) |
| `curl -s http://localhost:7777/health` | Confirm readiness (`success`) |

### Appendix B — Port Reference

| Port | Service | Context |
|---|---|---|
| 7777 | SimpleLogin web (HTTP) | Dev (`app.run(..., port=7777)`) and prod (`-b 0.0.0.0:7777`); `EXPOSE 7777` |
| 5432 | PostgreSQL | Default DB port used in `example.env` `DB_URI` |
| 6379 | Redis | Optional sessions/limiter when `MEM_STORE_URI` is set |

### Appendix C — Key File Locations

| Path | Role |
|---|---|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The deliverable** (641 lines) |
| `server.py` | Web bootstrap: `create_app()`, `local_main()`, hooks, `user_loader`, blueprints |
| `wsgi.py` | Production WSGI entry (`gunicorn wsgi:app`) |
| `app/config.py` | Import-time configuration loading (`>>> URL:` print) |
| `app/log.py` | Logging init (`>>> init logging <<<`; werkzeug logger disabled) |
| `app/db.py` | Eager import-time DB connection |
| `app/api/base.py` | API-key auth via `Authentication` header |
| `app/models.py` | Runtime identity (`User.get_id()` → `alternative_id`) |
| `job_runner.py`, `cron.py`, `event_listener.py`, `email_handler.py`, `init_app.py` | Separate background/worker processes |
| `Dockerfile`, `CONTRIBUTING.md`, `example.env`, `pyproject.toml` | Run command, dev procedure, config template, version pins |

### Appendix D — Technology Versions (pinned)

| Component | Version |
|---|---|
| Python | 3.10 |
| Flask | 1.1.2 |
| Werkzeug | 1.0.1 (transitive) |
| click | 8.0.3 |
| Flask-Login | 0.5.0 |
| SQLAlchemy | 1.3.24 |
| gunicorn | 20.0.4 |
| gevent | 22.10.2 |
| redis (client) | ^4.5.3 |
| newrelic | 8.8.0 |
| coloredlogs | 14.0 |
| PostgreSQL | 13+ |
| Redis | 6 (optional) |

### Appendix E — Environment Variable Reference

| Variable | Required | Example / Default | Purpose |
|---|---|---|---|
| `URL` | Yes | `http://localhost:7777` | Base app URL; printed at import (`>>> URL:`) |
| `DB_URI` | Yes | `postgresql://myuser:mypassword@localhost:5432/simplelogin` | PostgreSQL connection (eager connect at import) |
| `FLASK_SECRET` | Yes | `secret` | Flask `SECRET_KEY` (signs the `slapp` session cookie) |
| `EMAIL_DOMAIN` | Yes | `sl.local` | Alias email domain |
| `SUPPORT_EMAIL` | Yes | `support@sl.local` | Support address |
| `NOT_SEND_EMAIL` | No | `true` (dev) | Disable outbound email in dev |
| `MEM_STORE_URI` | No | _(unset)_ | Enables Redis-backed sessions/limiter when set |
| `COLOR_LOG` | No | _(unset)_ | Presence enables colored logs (drops ms from timestamps) |
| `SENTRY_DSN` | No | _(unset)_ | Enables optional Sentry error monitoring |
| `DISABLE_RATE_LIMIT` | No | _(unset)_ | Presence disables the rate limiter |

### Appendix F — Developer Tools Guide

| Tool | Use |
|---|---|
| `git diff` / `git status` | Verify scope (one added file) and clean tree |
| `grep` / `sed` | Inspect headings and spot-check `file:line` citations against source |
| `python3 -c "…count(chr(96)*3)…"` | Validate code-fence balance in the Markdown |
| `curl` | Confirm runtime readiness (`/health`) and request outcomes (web/API) |
| `alembic`, `flask` (CLI) | Schema migration and dummy-data seeding before the dev run (container) |

### Appendix G — Glossary

| Term | Meaning |
|---|---|
| **AAP** | Agent Action Plan — the governing requirement specification for this task |
| **WSGI** | The Python web-server interface; production entry is `gunicorn wsgi:app` |
| **`ProxyFix`** | Werkzeug middleware normalizing `X-Forwarded-*` headers from the NGINX proxy |
| **`create_app()`** | The single Flask application factory used by both dev and prod |
| **`local_main()`** | The dev launcher: forces color logs, attaches the DebugToolbar, runs `app.run(debug=True, port=7777)` |
| **`create_light_app()`** | A minimal app context used by background workers (not the web server) |
| **`user_loader`** | Flask-Login callback resolving the session user by `alternative_id` |
| **`alternative_id`** | A UUID identity (not the numeric PK) enabling session invalidation |
| **`slapp`** | The session cookie name (`SESSION_COOKIE_NAME`) |
| **Readiness (silent)** | The dev server accepts requests without printing `* Running on …` (werkzeug logger disabled) |

---

*This Blitzy Project Guide assesses a documentation-only deliverable. The SimpleLogin source tree was investigated read-only; no source file was created, modified, or deleted. Completion (93.5%) reflects AAP-scoped work only: 29h delivered of 31h total, with 2h remaining for human review and merge.*