# Blitzy Project Guide — SimpleLogin Runtime-Behavior Onboarding Q&A

> **Scope note.** This is a **documentation-only, strictly read-only** engagement governed by the
> `SWE-AtlasQnA-Repo` rule. The single in-scope deliverable is one markdown file —
> `blitzy/documentation/app_2cd6ee777f8c.md` — an evidence-backed onboarding Q&A that answers four
> runtime-behavior questions about the SimpleLogin Flask application. **No source file was modified.**
> Completion is measured against the Agent Action Plan (AAP) scope plus path-to-production.

---

## 1. Executive Summary

### 1.1 Project Overview

The objective was to produce a single, evidence-backed onboarding document explaining how the SimpleLogin
Flask application **actually behaves at runtime** — with every claim captured from real execution rather
than inferred from reading source. Targeting engineers onboarding into the codebase, it resolves four
questions: (Q1) the TCP bind port and startup logs across both the Werkzeug dev server and the gunicorn
production server; (Q2) the health-check status code and body; (Q3) the alias-creation API JSON response
and what is persisted to the database (table and column values), across both the random and custom
endpoints; and (Q4) the exact error and break location when PostgreSQL is unavailable at startup. The
deliverable is `blitzy/documentation/app_2cd6ee777f8c.md`.

### 1.2 Completion Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieTitleTextSize':'18px','pieLegendTextColor':'#B23AF2'}}}%%
pie showData title Completion Status (Completed vs Remaining Hours)
    "Completed Work AI 27h" : 27
    "Remaining Work 3h" : 3
```

| Metric | Value |
|--------|-------|
| **Total Hours** | **30** |
| **Completed Hours (AI + Manual)** | **27** (27 AI autonomous + 0 Manual) |
| **Remaining Hours** | **3** |
| **Percent Complete** | **90%** |

> **Calculation (PA1, AAP-scoped):** Completion % = Completed ÷ (Completed + Remaining) × 100 =
> 27 ÷ (27 + 3) × 100 = **90%**. All 24 AAP-specified requirements are complete and validated; the
> residual 3 hours are inherent path-to-production activities (human peer review, spot-check, merge)
> that cannot be autonomously completed.
>
> **Color key:** Completed = Dark Blue `#5B39F3`; Remaining = White `#FFFFFF`.

### 1.3 Key Accomplishments

- ✅ **Single in-scope deliverable authored and committed** — `blitzy/documentation/app_2cd6ee777f8c.md`, 638 lines, resolving all four questions.
- ✅ **All four questions answered with verbatim observed output** — port/logs, health, alias API + DB persistence, and the PostgreSQL-down error, each with a direct answer, exact `file:line` citations, the exact command run, and unedited output.
- ✅ **Both required variants of every multi-mode question exercised** — dev server **and** gunicorn (Q1); random **and** custom alias endpoints (Q3); happy path **and** error path.
- ✅ **Every value observed from the running app** (canonical `CONFIG=tests/test.env`, Python 3.10, PostgreSQL on 15432, Redis, Alembic 77-table schema), confirmed **stable across ≥2 runs**.
- ✅ **Read-only mandate honored perfectly** — `git diff` vs base = exactly one added file, 638 insertions, 0 deletions; `git status --porcelain` empty; all temporary scripts cleaned up.
- ✅ **Autonomous validation passed all 4 production-readiness gates**, with 5 evidence-grounded fidelity fixes applied to the deliverable; independent re-verification confirmed **14/14 spot-checked citations exact**.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| _None._ No blocking issues remain. The deliverable is complete, validated, and committed; all remaining work is standard human acceptance. | — | — | — |

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-----------------|----------------|-------------------|-------------------|-------|
| _None_ | — | **No access issues identified.** The investigation ran in the canonical warmed container with full access to Python 3.10, Poetry-locked dependencies, PostgreSQL (15432), and Redis; the branch is committed and the repository is accessible. | N/A | — |

> Two **reproducibility caveats** (not access issues) are disclosed in the deliverable: (1) a from-scratch Poetry build may need `cbor2==5.4.6` in place of the pinned `cbor2==5.2.0` sdist; (2) the investigation used PostgreSQL 15.13 vs. CI's `postgres:13`. Neither affects any of the four answers.

### 1.6 Recommended Next Steps

1. **[High]** Peer-review the deliverable's runtime claims and `file:line` citations for onboarding accuracy and clarity (2.0h).
2. **[Medium]** Independently reproduce a spot-check — boot the dev server and confirm the `127.0.0.1:7777` bind and `GET /health → 200 "success"` (0.5h).
3. **[Medium]** Merge the PR and link the document from the onboarding index (README/wiki) so new engineers can discover it (0.5h).
4. **[Low]** Schedule periodic re-validation of the document on major framework/schema upgrades to guard against runtime-behavior drift.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

All completed work was delivered autonomously (AI). Each component traces to specific AAP requirements.

| Component | Hours | Description |
|-----------|-------|-------------|
| Environment provisioning (AAP E1–E3) | 5.0 | Canonical Python 3.10 + Poetry-locked deps; PostgreSQL on port 15432 + Redis; Alembic schema applied (`alembic upgrade head` → 77 tables, head `32f25cbf12f6`). |
| Q1 — Bind port & startup logs (AAP Q1a–f) | 3.5 | Booted both entry points (dev `python server.py` → `127.0.0.1:7777`; gunicorn `-b 0.0.0.0:7777`); captured kernel socket (`/proc/net/tcp`) + startup logs; analyzed absent Werkzeug banner (`app/log.py:L70-71`); web-researched Flask default bind host. |
| Q2 — Health check (AAP Q2a–c) | 2.0 | `GET /health` → 200, body `success`, `text/html`; verified `after_request` timing exclusion by contrast with `/live`; documented sibling monitor endpoints `/live`, `/git`, `/exception`. |
| Q3 — Alias API + DB persistence (AAP Q3a–f) | 5.0 | Seeded user/API key/default mailbox via real factories; `POST` to random **and** custom endpoints → `201` JSON; DB row read-back from table `alias`; JSON↔column mapping; historical `gen_email`→`alias` rename investigation via pg catalog; stability across ≥2 runs. |
| Q4 — PostgreSQL unavailable (AAP Q4a–d) | 2.5 | Reproduced two-layer `psycopg2`/SQLAlchemy error; pinpointed eager break at `app/db.py:L12`; traced full import chain; explained IPv6/IPv4 ordering (getaddrinfo RFC 3484). |
| Deliverable authoring (AAP D1–D4) | 5.0 | Authored the 638-line markdown: per-question structure (direct answer, citations, exact command, verbatim output, cause→effect), environment caveats, and coverage checklist. |
| Autonomous validation + fidelity fixes | 3.0 | Ran the real app across ≥2 runs per question; applied 5 evidence-grounded fidelity fixes to the `.md`; integrity checks (fences, whitespace, line endings). |
| Cleanup + repository-unchanged verification (AAP M1–M3) | 1.0 | Removed all temporary observation scripts/logs; verified `git status --porcelain` empty and diff = single file. |
| **Total Completed** | **27.0** | **Matches Completed Hours in §1.2.** |

### 2.2 Remaining Work Detail

All remaining work is **path-to-production** and inherently requires a human.

| Category | Hours | Priority |
|----------|-------|----------|
| Technical peer review of runtime claims & citations | 2.0 | High |
| Independent reproducibility spot-check (dev bind + `/health`) | 0.5 | Medium |
| Merge PR & publish/link to onboarding index | 0.5 | Medium |
| **Total Remaining** | **3.0** | **Matches Remaining Hours in §1.2 and §7.** |

### 2.3 Hours Reconciliation

| Line | Hours |
|------|-------|
| §2.1 Completed total | 27.0 |
| §2.2 Remaining total | 3.0 |
| **Sum (= §1.2 Total)** | **30.0** |
| Completion % (27 ÷ 30) | **90%** |

---

## 3. Test Results

> **Integrity note.** This is a documentation-only deliverable with **no unit tests**; no source code was
> changed, so the existing repository test suite is unaffected. The meaningful validation for this task is
> **runtime-behavior verification** — booting the real application and confirming every documented value
> against observed output. All entries below originate from **Blitzy's autonomous validation logs** for
> this project, each confirmed **stable across ≥2 runs**.

| Test Category | Framework / Method | Total Checks | Passed | Failed | Coverage | Notes |
|---------------|--------------------|--------------|--------|--------|----------|-------|
| Q1 — Bind port & startup logs | `/proc/net/tcp` + `ss`; dev & gunicorn boot; stdout capture | 5 | 5 | 0 | 100% of named sub-parts | Dev `127.0.0.1:7777`, gunicorn `0.0.0.0:7777`; gunicorn INFO block; SL bootstrap block; Werkzeug banner confirmed absent. |
| Q2 — Health check & monitor siblings | Werkzeug test client / `curl` | 5 | 5 | 0 | 100% | `/health`→200 `success` `text/html`; `after_request` exclusion (contrast `/live`); `/live`→`live`, `/git`→`dev`, `/exception`→500. |
| Q3 — Alias API + DB persistence | Werkzeug test client + `psql` catalog read-back | 6 | 6 | 0 | 100% (both endpoints) | Random & custom `POST`→`201` (17-key JSON); persisted to table `alias`; column read-back; `gen_email`→`alias` rename catalog; stability ≥2 runs. |
| Q4 — PostgreSQL-down error | `python -c "import server"` with PG stopped; stderr capture | 4 | 4 | 0 | 100% | Two-layer `psycopg2`/`sqlalchemy.exc.OperationalError`; eager break `app/db.py:L12`; import chain; restart-succeeds complementary proof. |
| Citation grounding (independent re-verification) | `sed`/`grep` against live source | 14 | 14 | 0 | 100% of spot-checked citations | All Q1–Q4 core `file:line` references exact against source. |
| Deliverable integrity | Shell (`wc`, `grep`, `od`, `file`) | 5 | 5 | 0 | 100% | 638 lines; 28 fences (14 balanced pairs); 0 trailing-whitespace; LF-only; final newline present. |
| Dependency version check | `pip check` + version assertions | — | Pass | 0 | — | `pip check`: no broken requirements; Flask 1.1.2 / Werkzeug 1.0.1 / gunicorn 20.0.4 / SQLAlchemy 1.3.24 / psycopg2 2.9.3 / arrow 0.16.0 match the doc. |
| **Totals** | — | **39** | **39** | **0** | **100%** | All runtime-behavior verifications passed. |

---

## 4. Runtime Validation & UI Verification

**Runtime health (observed):**

- ✅ **Dev server boot** — `python server.py` binds `127.0.0.1:7777` (loopback); SL bootstrap block emitted.
- ✅ **Production server boot** — `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` binds `0.0.0.0:7777`; master + 2 workers; gunicorn INFO block emitted.
- ✅ **Health endpoint** — `GET /health` → `200`, body `success`, `Content-Type: text/html; charset=utf-8`.
- ✅ **Monitor endpoints** — `/live` → `live`; `/git` → `dev`; `/exception` → `500` (intentional).
- ✅ **Alias API (random)** — `POST /api/alias/random/new` → `201` with 17-key JSON; row persisted to `alias`.
- ✅ **Alias API (custom)** — `POST /api/v3/alias/custom/new` → identical `201` shape, same `alias` table.
- ✅ **Database persistence** — persisted row read back and mapped field-by-field to the JSON response.
- ✅ **Error path (PG down)** — eager import-time failure reproduced at `app/db.py:L12`; two-layer error captured verbatim.

**API integration:** ✅ Operational — authentication via the `Authentication` header (`app/api/base.py:L16-18`) exercised end-to-end for both alias endpoints.

**UI verification:** ⚠ **Not applicable** — the AAP explicitly involves no user-facing UI work (AAP §0.3.3). No screens, components, or visual flows are in scope; the deliverable is a markdown document. No UI regressions are possible.

---

## 5. Compliance & Quality Review

Cross-mapping of the governing `SWE-AtlasQnA-Repo` rule directives and AAP deliverables to observed outcomes.

| Benchmark / Rule | Requirement | Status | Evidence |
|------------------|-------------|--------|----------|
| Deliverable rule | Single markdown named `<source_branch>.md` in `blitzy/documentation/` | ✅ Pass | `blitzy/documentation/app_2cd6ee777f8c.md` present and committed. |
| Run-first, evidence-driven | Build & run before writing; author from observation | ✅ Pass | App booted via both entry points; all values captured from execution. |
| Observed, not inferred | Every value from real execution; inferences labeled | ✅ Pass | Verbatim output blocks per question; inferred statements labeled. |
| Answer every named item | Each mechanism/file/condition addressed by name | ✅ Pass | Coverage checklist enumerates all named sub-parts (11 items). |
| Exact & grounded | Actual value + `file:line`; name the function | ✅ Pass | 14/14 spot-checked citations exact; handlers/factories named. |
| Complete, unedited output | Full command output, no elided logic | ✅ Pass | `after_request` handler and werkzeug-disable block quoted in full. |
| Cover all candidate implementations | Both alias endpoints; both server entry points | ✅ Pass | Random + custom; dev + gunicorn all exercised. |
| Exercise happy + error paths | Not just the happy path | ✅ Pass | Health/alias happy paths + PG-down error path. |
| Stability | Confirm stable across ≥2 runs | ✅ Pass | Stability notes; structural invariants vs volatile fields identified. |
| Web-search requirement | Confirm Flask dev-server default bind host | ✅ Pass | Flask 1.1.2 source + Quickstart cited for `127.0.0.1` default. |
| Read-only scope | No source file modified/added except the doc | ✅ Pass | `git diff` = 1 file added; `git status --porcelain` empty. |
| Temporary-script cleanup | Remove all temp scripts afterward | ✅ Pass | Validator removed 21 + 57 + 2 temp artifacts; repo clean. |
| Final coverage pass | Re-read question; confirm each item answered | ✅ Pass | Coverage checklist present (all `[x]`). |
| Autonomous validation fixes | Discrepancies resolved during validation | ✅ Pass | 5 evidence-grounded fidelity fixes applied to the `.md` (commit `8ff11c4e`). |

**Document quality:** 638 lines, 14 balanced code-fence pairs, LF-only, no trailing whitespace, final newline present, UTF-8. **Outstanding items:** none.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| T1 — Reader treats run-specific values (PIDs, timestamps, random local-parts, row id) as invariant | Technical | Low | Low | Doc explicitly labels volatile fields and includes per-question stability notes | Mitigated |
| T2 — `file:line` citations drift if upstream source changes | Technical | Low | Medium | Citations pinned to base commit `2cd6ee77`; each cites the stable function/handler name alongside the line number | Mitigated |
| S1 — Reproduction of test credentials (`test:test`) / config in the doc | Security | Low | Low | Pre-existing **public** test fixtures (`tests/test.env`); API keys shown as `<api_key.code>` placeholders; no production secrets | Mitigated (no new exposure) |
| O1 — Documentation staleness as SimpleLogin evolves | Operational | Medium | Medium | Pinned to commit + canonical config; recommend re-validation on major upgrades | Open (not release-blocking) |
| I1 — From-scratch reproduction hits `cbor2==5.2.0` sdist failure or PG-version mismatch | Integration | Low | Low | Both caveats disclosed in "Environment caveats"; confirmed neither affects the four answers; warmed image ships the exact pin | Mitigated (disclosed) |

**Overall risk posture: LOW.** No High-severity and no release-blocking risks. Zero source code changed means no regression, compilation, or test surface.

---

## 7. Visual Project Status

**Overall completion (hours):**

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieTitleTextSize':'18px','pieLegendTextColor':'#B23AF2'}}}%%
pie showData title Project Hours Completed vs Remaining
    "Completed Work" : 27
    "Remaining Work" : 3
```

> **Integrity:** "Remaining Work" = **3** here equals the Remaining Hours in §1.2 and the sum of the §2.2 Hours column. "Completed Work" = **27** equals §2.1 total. Colors: Completed = `#5B39F3`, Remaining = `#FFFFFF`.

**Completed hours by category (breakdown of the 27h):**

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pieOuterStrokeColor':'#B23AF2','pieSectionTextColor':'#111111','pieTitleTextSize':'16px','pie1':'#5B39F3','pie2':'#7B5CF5','pie3':'#9B7DF7','pie4':'#B23AF2','pie5':'#A8FDD9','pie6':'#6D28D9','pie7':'#8B5CF6','pie8':'#C4B5FD'}}}%%
pie showData title Completed Hours by Category
    "Env provisioning" : 5
    "Q1 port and logs" : 3.5
    "Q2 health" : 2
    "Q3 alias API and DB" : 5
    "Q4 PG-down" : 2.5
    "Deliverable authoring" : 5
    "Validation and fixes" : 3
    "Cleanup and verify" : 1
```

**Remaining hours by task (breakdown of the 3h):**

| Task | Hours | Priority |
|------|-------|----------|
| Technical peer review | 2.0 | High |
| Reproducibility spot-check | 0.5 | Medium |
| Merge & publish | 0.5 | Medium |
| **Total** | **3.0** | — |

---

## 8. Summary & Recommendations

**Achievements.** The project is **90% complete** (27 of 30 hours). The single AAP deliverable —
`blitzy/documentation/app_2cd6ee777f8c.md` — is authored, validated, and committed. All 24 AAP-specified
requirements are complete: the four onboarding questions are answered with verbatim observed output,
exact `file:line` citations, and the exact commands used; both alias endpoints and both server entry
points are exercised; and the happy and error paths are both demonstrated. Autonomous validation passed
all four production-readiness gates, and independent re-verification confirmed 14/14 spot-checked
citations exact.

**Remaining gaps.** The residual **3 hours (10%)** are entirely path-to-production activities that
inherently require a human: technical peer review of the runtime claims (2.0h), an independent
reproducibility spot-check (0.5h), and merge/publish (0.5h). There are **no blocking defects**, no
failing tests, and no missing functionality.

**Critical path to production.** Peer review → reproducibility spot-check → merge and link from the
onboarding index. This can realistically be completed within a single short review cycle.

**Production readiness assessment.** **READY** for human review and merge. The read-only mandate is
perfectly honored (`git status --porcelain` empty; diff = one added file), the document integrity is
clean, and the risk posture is Low with no release-blocking items. The one Open risk (O1, documentation
staleness) is inherent to any runtime-behavior document and is addressed with a re-validation
recommendation rather than a code change.

| Success Metric | Target | Actual |
|----------------|--------|--------|
| AAP questions answered with observed evidence | 4 / 4 | ✅ 4 / 4 |
| Multi-mode variants covered (endpoints + entry points) | All | ✅ Random+custom, dev+gunicorn |
| Repository files modified (source) | 0 | ✅ 0 |
| Spot-checked citations exact | 100% | ✅ 14 / 14 |
| Production-readiness gates passed | 4 / 4 | ✅ 4 / 4 |

---

## 9. Development Guide

This guide documents how to reproduce the runtime investigation and how to verify the deliverable.
Commands are grouped as **(A) canonical reproduction** (validated in the warmed container) and
**(B) read-only verification** (tested in the assessment environment; all pass).

### 9.1 System Prerequisites

- **Python 3.10** — canonical runtime (`Dockerfile:L8` `FROM python:3.10`; CI matrix `python-version: ["3.10"]` at `.github/workflows/main.yml:L40`; `pyproject.toml` `python = "^3.10"`).
- **PostgreSQL** — canonical CI image `postgres:13` (`.github/workflows/main.yml:L47`); the investigation used 15.13 on **port 15432** to match `tests/test.env`.
- **Redis** — backs Flask-Limiter rate limiting and session storage (`MEM_STORE_URI=redis://localhost`).
- **Poetry 1.8.5** — dependency resolution from `poetry.lock`.
- OS: Linux; ~1 GB free disk for dependencies and the schema.

### 9.2 Environment Setup

```bash
# Select the canonical dotenv configuration (resolved at app/config.py:L65-69)
export CONFIG=tests/test.env

# tests/test.env provides (already in the repo — do not edit):
#   DB_URI=postgresql://test:test@localhost:15432/test   (tests/test.env:L17)
#   MEM_STORE_URI=redis://localhost                       (tests/test.env:L78)
#   EMAIL_DOMAIN=sl.local                                 (tests/test.env:L8)
#   URL=http://localhost                                  (tests/test.env:L2)
```

Ensure PostgreSQL is listening on `localhost:15432` and Redis on `localhost:6379`.

### 9.3 Dependency Installation

```bash
# Install the exact Poetry-locked dependency set
poetry install

# Verify the environment is consistent
pip check    # expect: "No broken requirements found."
```

### 9.4 Schema Application

```bash
# Apply the full schema via Alembic/Flask-Migrate (produces 77 tables; head 32f25cbf12f6)
CONFIG=tests/test.env alembic upgrade head
```

### 9.5 Application Startup

```bash
# (A) Development server — binds 127.0.0.1:7777 (loopback only)
CONFIG=tests/test.env python server.py

# (B) Production server — binds 0.0.0.0:7777 (all interfaces)
CONFIG=tests/test.env gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
```

### 9.6 Verification Steps

```bash
# Health check — expect: HTTP/1.1 200 OK, Content-Type text/html; charset=utf-8, body: success
curl -sS -i http://127.0.0.1:7777/health

# Confirm the listening socket / bind host
ss -ltnp | grep 7777          # or: grep 1E61 /proc/net/tcp   (1E61 = port 7777 in hex)

# Sibling monitor endpoints
curl -sS http://127.0.0.1:7777/live   # -> live
curl -sS http://127.0.0.1:7777/git    # -> dev (local, non-CI build)
```

### 9.7 Example Usage — Alias Creation (Q3)

```bash
# Random alias — returns 201 with a 17-key JSON body; persists a row to table "alias"
curl -sS -X POST \
  -H "Authentication: <api_key.code>" \
  http://127.0.0.1:7777/api/alias/random/new

# Custom alias (sibling) — identical 201 shape; same "alias" table
curl -sS -X POST \
  -H "Authentication: <api_key.code>" \
  -H "Content-Type: application/json" \
  -d '{"alias_prefix":"my-custom-prefix","signed_suffix":"<signed>","mailbox_ids":[1]}' \
  http://127.0.0.1:7777/api/v3/alias/custom/new
```

### 9.8 Deliverable Verification (tested — all pass)

~~~bash
# Repository must be unchanged except the single deliverable
git status --porcelain                       # -> empty (clean)
git diff 2cd6ee77 --name-status              # -> A  blitzy/documentation/app_2cd6ee777f8c.md
git ls-files blitzy/                         # -> only the deliverable

# Document integrity
wc -l blitzy/documentation/app_2cd6ee777f8c.md          # -> 638
grep -c '```' blitzy/documentation/app_2cd6ee777f8c.md  # -> 28 (14 balanced pairs)
~~~

### 9.9 Troubleshooting

- **`psycopg2.OperationalError` / `sqlalchemy.exc.OperationalError` at import time.** PostgreSQL is not reachable. This is the **eager, import-time** connection at `app/db.py:L12` — the exact behavior documented for Q4, not a bug. Start PostgreSQL on `localhost:15432` and re-run; the import then succeeds.
- **From-scratch build fails on `cbor2==5.2.0`.** Its `setuptools_scm`-derived version can resolve to `0.0.0` and the sdist is rejected. Substitute the nearest wheeled release `cbor2==5.4.6`. `cbor2` is used only for FIDO/WebAuthn paths and does not affect the four answers.
- **Werkzeug `* Running on ...` banner and access logs are missing.** Expected — the `werkzeug` logger is disabled at `app/log.py:L70-71`. The socket is still bound (confirm via `ss`/`/proc/net/tcp`).
- **Port 7777 already in use.** A stale server is running; stop it before starting a new one.

---

## 10. Appendices

### Appendix A — Command Reference

| Purpose | Command |
|---------|---------|
| Select config | `export CONFIG=tests/test.env` |
| Install deps | `poetry install` |
| Dependency sanity | `pip check` |
| Apply schema | `CONFIG=tests/test.env alembic upgrade head` |
| Dev server (127.0.0.1:7777) | `CONFIG=tests/test.env python server.py` |
| Production server (0.0.0.0:7777) | `CONFIG=tests/test.env gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` |
| Health probe | `curl -sS -i http://127.0.0.1:7777/health` |
| Reproduce PG-down error | `CONFIG=tests/test.env python -c "import server"` (with PostgreSQL stopped) |
| Repo unchanged check | `git status --porcelain` |
| Diff scope | `git diff 2cd6ee77 --name-status` |

### Appendix B — Port Reference

| Port | Service | Notes |
|------|---------|-------|
| 7777 | SimpleLogin web (Flask/gunicorn) | Dev binds `127.0.0.1`; production binds `0.0.0.0` (`Dockerfile:L44` `EXPOSE 7777`) |
| 15432 | PostgreSQL | Per `tests/test.env` `DB_URI` |
| 6379 | Redis | Default `MEM_STORE_URI=redis://localhost` |

### Appendix C — Key File Locations

| File | Role |
|------|------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The deliverable** (onboarding Q&A) |
| `server.py` | Dev entry (`local_main` → `app.run`, L572-588); `/health` (L213-215); `after_request` (L272-296) |
| `wsgi.py` | Gunicorn WSGI target (`wsgi:app`) |
| `Dockerfile` | Prod `CMD` gunicorn (L47); `EXPOSE 7777` (L44); `FROM python:3.10` (L8) |
| `app/db.py` | Eager `engine.connect()` at import (L12) — Q4 break point |
| `app/log.py` | Werkzeug logger disabled (L70-71); log format (L12-14); init line (L67) |
| `app/config.py` | Startup prints (L68, L80, L262, L328); config keys |
| `app/api/views/new_random_alias.py` | Random alias endpoint; response assembly (L114-117) |
| `app/api/views/new_custom_alias.py` | Custom alias endpoints (v2 L28, v3 L115) |
| `app/api/serializer.py` | `serialize_alias_info_v2` (L55-93) |
| `app/models.py` | `Alias` model / table `alias` (L1470); `User.create` (L602-613); `ApiKey.create` (L2365) |
| `tests/test.env` | Canonical run configuration |

### Appendix D — Technology Versions

| Component | Version | Source |
|-----------|---------|--------|
| Python | 3.10 (observed 3.10.18) | `Dockerfile:L8`, `pyproject.toml`, `.github/workflows/main.yml:L40` |
| Flask | 1.1.2 | `poetry.lock` |
| Werkzeug | 1.0.1 | `poetry.lock` |
| gunicorn | 20.0.4 | `poetry.lock` |
| Flask-SQLAlchemy | 2.5.1 | `poetry.lock` |
| SQLAlchemy | 1.3.24 | `poetry.lock` |
| psycopg2-binary | 2.9.3 | `poetry.lock` |
| arrow | 0.16.0 | `poetry.lock` |
| PostgreSQL | 13 (CI) / 15.13 (investigation) | `.github/workflows/main.yml:L47` |
| Poetry | 1.8.5 | Investigation toolchain |

### Appendix E — Environment Variable Reference

| Variable | Value (canonical) | Purpose |
|----------|-------------------|---------|
| `CONFIG` | `tests/test.env` | Selects the dotenv config file (`app/config.py:L65-69`) |
| `DB_URI` | `postgresql://test:test@localhost:15432/test` | PostgreSQL connection (`tests/test.env:L17`) |
| `MEM_STORE_URI` | `redis://localhost` | Redis for Flask-Limiter/session (`tests/test.env:L78`) |
| `EMAIL_DOMAIN` | `sl.local` | Alias email domain (`tests/test.env:L8`) |
| `URL` | `http://localhost` | Base URL (`tests/test.env:L2`) |

### Appendix F — Developer Tools Guide

| Tool | Use |
|------|-----|
| `ss -ltnp` / `/proc/net/tcp` | Confirm the listening socket and bind host (port 7777 = hex `1E61`) |
| `curl -sS -i` | Probe HTTP endpoints and capture status, headers, and body |
| `psql` | Read persisted rows and inspect the schema catalog (`gen_email_id_seq`, `gen_email_pkey`) |
| `alembic upgrade head` | Apply the 77-table schema |
| `git status --porcelain` / `git diff` | Verify the read-only mandate (repo unchanged except the deliverable) |
| `pip check` | Confirm dependency consistency |

### Appendix G — Glossary

| Term | Definition |
|------|------------|
| AAP | Agent Action Plan — the primary directive defining project scope and requirements |
| Deliverable | The single in-scope file `blitzy/documentation/app_2cd6ee777f8c.md` |
| Eager connection | The module-level `engine.connect()` at `app/db.py:L12` executed at import time, making PostgreSQL a hard startup dependency |
| Random alias endpoint | `POST /api/alias/random/new` — server-generated local-part |
| Custom alias endpoint | `POST /api/v3/alias/custom/new` — client prefix + signed suffix |
| Volatile field | A run-specific value (PID, timestamp, random local-part, row id) that varies between runs but is structurally stable |
| `gen_email` → `alias` | Historical table rename; the sequence and PK constraint retain `gen_email_*` names |
| Path-to-production | Human acceptance activities (review, spot-check, merge) required to deploy the deliverable |
