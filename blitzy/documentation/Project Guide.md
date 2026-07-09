# Blitzy Project Guide — SimpleLogin Dev-Stack Startup & Failure Investigation

> Brand legend — **Completed / AI Work:** Dark Blue `#5B39F3` · **Remaining / Not Completed:** White `#FFFFFF` · **Headings / Accents:** Violet-Black `#B23AF2` · **Highlight:** Mint `#A8FDD9`

---

## 1. Executive Summary

### 1.1 Project Overview

This project is a **read-only runtime investigation** of the SimpleLogin Flask monolith (commit `2cd6ee777f8c`). A first-time developer needed the dev stack brought up from scratch and three specific startup/failure behaviors documented with real captured output: (1) the exception raised by `python server.py` against an empty (un-migrated) database; (2) the required Python services and their exact readiness/port-binding output once fully initialized; and (3) the SMTP rejection returned when `init_app.py` is skipped and an `@sl.local` email is injected. The sole deliverable is one evidence-backed markdown answer document. No source code is changed — behaviors are observed, not modified.

### 1.2 Completion Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieOuterStrokeWidth':'2px','pieSectionTextColor':'#B23AF2','pieTitleTextSize':'16px'}}}%%
pie showData title Completion Status — 91.7% Complete
    "Completed Work (AI)" : 33
    "Remaining Work" : 3
```

| Metric | Hours |
|--------|-------|
| **Total Hours** | **36** |
| Completed Hours (AI + Manual) | 33 (AI: 33 · Manual: 0) |
| Remaining Hours | 3 |
| **Percent Complete** | **91.7%** |

> Completion is calculated with the AAP-scoped hours methodology: `Completed ÷ (Completed + Remaining) = 33 ÷ 36 = 91.7%`. All autonomous AAP-scoped work is delivered and validated; the remaining 3 hours are human path-to-production (technical review + PR merge).

### 1.3 Key Accomplishments

- ✅ **Dev stack brought up from scratch** — PostgreSQL 13.23, Redis 7.4.9, Python 3.10.18 + Poetry-locked deps, transient `.env` from `example.env`.
- ✅ **Q1 answered with the complete, unedited exception** — `sqlalchemy.exc.ProgrammingError` wrapping `psycopg2.errors.UndefinedTable` (`relation "users" does not exist`), reproduced byte-for-byte (HTTP 500, 5749-byte custom `500.html`).
- ✅ **Q2 answered with a two-tier service map** — essential (web app :7777, email handler :20381, PostgreSQL :5432, Redis :6379) vs auxiliary (`job_runner.py`, `cron.py`, `event_listener.py`), each with its captured readiness banner and port-binding proof.
- ✅ **Q3 answered with the authentic SMTP reply** — `550 SL E515 Email not exist` captured directly at the sender via `swaks`, plus the rejection log line at `email_handler.py:551`.
- ✅ **Three honest findings surfaced** — MEM_STORE_URI-unset Redis behavior (§3.2.4), SPF/E216 caveat (§4.6), and the causal correction that an empty `SLDomain` is *contextual, not causal* to the 550 (§4.7).
- ✅ **1,015-line deliverable** with 90 captured-output code blocks and 191 `file:line` citations; every named item in all three questions covered (§5.3).
- ✅ **Read-only mandate perfectly preserved** — `git diff` = exactly one file added, zero source files modified.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| _None._ Blitzy's autonomous validation reproduced all three scenarios byte-for-byte, verified all 191 citations (zero discrepancies), and confirmed the read-only mandate. No blocking issues remain. | — | — | — |

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-----------------|----------------|-------------------|-------------------|-------|
| _No access issues identified._ The investigation ran in the self-contained Docker topology (`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`); PostgreSQL, Redis, the app venv, and `swaks` were all locally available. No external credentials or third-party API access were required. | — | — | — | — |

### 1.6 Recommended Next Steps

1. **[High]** Perform the human technical review of `blitzy/documentation/app_2cd6ee777f8c.md` — confirm the three answers, spot-check a sample of the 191 citations against source, and review §5.3/§5.4 coverage.
2. **[Medium]** Approve and merge the PR (3 commits by `agent@blitzy.com`); confirm the working tree stays clean and no source files are touched.
3. **[Low, optional]** Independently reproduce one scenario (e.g., the Q3 `swaks` injection → `550 SL E515`) in a fresh environment using the §1.5/§4.3 recipes.
4. **[Low, optional]** If the document will be maintained long-term, guard the intentional code-fence trailing-whitespace evidence against the `pre-commit` `trailing-whitespace` hook and add a stale-citation note keyed to commit `2cd6ee777f8c`.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

Each component traces to a specific AAP requirement (categories A–F). **Total = 33 hours.**

| Component | Hours | Description |
|-----------|-------|-------------|
| Dev-stack environment bring-up & configuration | 4 | [AAP §0.5.1] Provision PostgreSQL 13, Redis :6379, Python 3.10 + Poetry-locked deps; materialize transient `.env` from `example.env`; verify version matrix (SQLAlchemy 1.3.24, Flask 1.1.2, aiosmtpd 1.4.2, Werkzeug 1.0.1). |
| Q1 investigation — empty-schema exception | 4 | [AAP Req 1] BEFORE-state proof (empty schema), `python server.py`, GET/POST login, capture full 49-column traceback, 500 render-path analysis, reasoning. |
| Q2 investigation — required-services enumeration | 6 | [AAP Req 2] `alembic upgrade head` (255 scripts) + `init_app.py`; start 4 essential + 3 auxiliary services; capture readiness banners + port-binding/no-port proofs; MEM_STORE_URI finding. |
| Q3 investigation — unconfigured-domain rejection | 5 | [AAP Req 3] Reach migrated-but-uninitialized state (`public_domain`=0), inject `@sl.local` via `swaks`, capture `550 SL E515`, trace decision chain, 3 honest findings, AFTER-state. |
| Answer-document authoring & structuring | 9 | [AAP §0.4.2] Write the 1,015-line deliverable: 90 captured-output code blocks, 191 `file:line` citations, per-answer reasoning, §5 coverage pass + observed-vs-inferred. |
| Code-review response & byte-accurate corrections | 3 | [AAP §0.7 rules] Commit `95be77c7` resolved 8 review findings (full traceback, auxiliary services, real UUID/elapsed, real psql AFTER-state); commit `298bab13` corrected the binding-proof justification. |
| External-tool web-search validation, QA/coverage passes & cleanup | 2 | [AAP §0.2.2 / §0.7] Validate `swaks` + `aiosmtpd` semantics against authoritative docs; repeatability discipline (3× per scenario); read-only cleanup (remove temp scripts/`.env`). |
| **Total Completed** | **33** | |

### 2.2 Remaining Work Detail

Each category traces to a path-to-production need (category G). **Total = 3 hours.**

| Category | Hours | Priority |
|----------|-------|----------|
| Human technical review & sign-off of the answer document (read 1,015 lines; spot-check sample of 191 citations vs source; verify all 3 answers + named-item coverage; confirm read-only diff) | 2.5 | High |
| PR approval & merge to target branch (confirm working tree clean, no source files touched) | 0.5 | Medium |
| **Total Remaining** | **3** | |

> Optional, non-blocking follow-ups (independent scenario re-run; long-term doc-maintenance guards) are **0 hours** and excluded from the remaining total so cross-section integrity holds.

### 2.3 Hours Calculation & Basis of Estimate

- **Total Project Hours:** `33 (completed) + 3 (remaining) = 36`.
- **Completion %:** `33 ÷ 36 = 91.666… ≈ 91.7%`.
- **Cross-section check:** Section 2.1 total (33) + Section 2.2 total (3) = 36 = Section 1.2 Total Hours. Remaining (3) is identical across Sections 1.2, 2.2, and 7.
- **Confidence:** High. The scope is a single, well-defined markdown deliverable; all autonomous work is complete/committed/validated, so the only estimation uncertainty is the human review duration (2.5h ± 0.5h), which does not change the headline percentage materially.

---

## 3. Test Results

For a read-only documentation deliverable there is **no traditional unit-test suite in scope** — the repository's `pytest`/`sl-test-db` suite is explicitly out of scope per the AAP. Per the validator's framing, "tests" map to two Blitzy autonomous-validation activities: **(a) runtime reproduction** of each documented scenario and **(b) code-citation verification** against source. Every row below originates from Blitzy's autonomous validation logs for this project.

| Test Category | Framework / Tooling | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|---------------------|-------------|--------|--------|-----------|-------|
| Runtime reproduction — Q1 (empty schema) | psql + `python server.py` (Werkzeug 1.0.1) + curl | 3 | 3 | 0 | 100% | GET `/auth/login`=200; POST=500, Content-Length 5749 byte-identical; full `ProgrammingError`/`UndefinedTable` traceback byte-for-byte; 3× stable. |
| Runtime reproduction — Q2 essential services | curl `/health`, `pg_isready`, `redis-cli`, aiosmtpd | 4 | 4 | 0 | 100% | Web app `/health`=success on :7777; email handler `Listen for port 20381` + `Start mail controller 0.0.0.0 20381`; PostgreSQL :5432; Redis PONG :6379. |
| Runtime reproduction — Q2 auxiliary services | `python` entry points | 3 | 3 | 0 | 100% | `job_runner.py`, `cron.py -j sanity_check` (exit 0), `event_listener.py`; confirmed bind NO TCP port (socket count unchanged). |
| Runtime reproduction — Q3 (unconfigured domain) | `swaks` + aiosmtpd | 3 | 3 | 0 | 100% | `swaks --to e1@sl.local` → `550 SL E515 Email not exist` byte-accurate (exit 26); rejection log at `email_handler.py:551`; 3× stable. |
| Code-citation verification | manual + `grep` vs source @ HEAD | 191 | 191 | 0 | 100% | Every `file:line` cited in the doc verified against source; **zero discrepancies**. |
| **Total** | | **204** | **204** | **0** | **100%** | All checks sourced from Blitzy autonomous validation logs. |

> **Pass rate: 100% (204/204).** "Coverage %" here denotes scenario/named-item and citation-grounding coverage, not statement coverage of a code test suite (none is in scope).

---

## 4. Runtime Validation & UI Verification

Legend: ✅ Operational / reproduced-as-documented · ⚠ Partial · ❌ Failing

**Q1 — Empty-schema failure (a failure the doc must reproduce):**
- ✅ `GET /auth/login` renders HTTP **200** with no DB access (static render).
- ✅ `POST /auth/login` triggers the first ORM query (`User.get_by` → `ModelMixin.get_by` at `app/models.py:84`) → **HTTP 500**, exact `sqlalchemy.exc.ProgrammingError` / `psycopg2.errors.UndefinedTable` (`relation "users" does not exist`).
- ✅ Render path confirmed: app's **custom `templates/error/500.html`** (not the Werkzeug interactive debugger).

**Q2 — Required services (fully-initialized state):**
- ✅ **Web app** (`server.py`) — `/health` returns `success` on :7777.
- ✅ **Email handler** (`email_handler.py`) — banner `Listen for port 20381` + `Start mail controller 0.0.0.0 20381`; port :20381 accepts connections (proved via `/dev/tcp`, since `ss`/`iproute2` is absent in the container).
- ✅ **PostgreSQL** :5432 — `pg_isready` OK.
- ✅ **Redis** :6379 — `PONG`.
- ✅ **Auxiliary** — `job_runner.py`, `cron.py`, `event_listener.py` start and run; explicitly bind **no** TCP port.
- ✅ Honest finding: with `MEM_STORE_URI` unset, the web app opens **no** Redis connection in dev (`connected_clients:1`) — documented in §3.2.4.

**Q3 — Unconfigured-domain rejection (migrated-but-uninitialized):**
- ✅ BEFORE state: `SELECT count(*) FROM public_domain` = **0**; `alias` = 0.
- ✅ `swaks` injection to `e1@sl.local` → SMTP **`550 SL E515 Email not exist`** returned to the sender (byte-accurate).
- ✅ Rejection log: `alias e1@sl.local cannot be created on-the-fly, return 550` (`email_handler.py:551`).
- ✅ AFTER state: `alias` = 0 (rejection creates no alias); re-injection after populating `SLDomain` still returns 550 (§4.7).

**UI Verification:** No UI change was requested or made. The login page is exercised **only as a trigger** for the Q1 exception. UI observed: `GET /auth/login` = 200 (Jinja render); error path = custom `500.html`. No visual regression scope applies.

---

## 5. Compliance & Quality Review

The AAP's governing **SWE-AtlasQnA-Repo ruleset (§0.7)** is the compliance benchmark. Each rule is cross-mapped to the delivered document with pass/fail status.

| # | AAP Rule (§0.7) | Status | Evidence / Progress |
|---|-----------------|--------|---------------------|
| R1 | Deliverable naming & location — `<source_branch>.md` in `blitzy/documentation/` | ✅ Pass | `blitzy/documentation/app_2cd6ee777f8c.md` created (branch `app_2cd6ee777f8c`). |
| R2 | Investigate by RUNNING first, then write | ✅ Pass | Every answer backed by captured runtime output; 90 code blocks of real transcripts. |
| R3 | Observe magnitude/frequency/timing at real scale | ✅ Pass | 255 migrations counted; real per-message UUID + elapsed-time float captured (§4). |
| R4 | Reproduce reported inconsistency faithfully | ✅ Pass (N/A) | No run-to-run inconsistency reported; scenarios shown 3× byte-stable instead. |
| R5 | Exercise exact code path via REAL entry point | ✅ Pass | `python server.py`, `python email_handler.py`, `swaks` — no synthetic stand-ins. |
| R6 | Use default, canonical configuration | ✅ Pass | `.env` from `example.env`; documented commands (`alembic upgrade head`, `init_app.py`). |
| R7 | Exercise every condition the question implies | ✅ Pass | GET vs POST (Q1); essential vs auxiliary + each port (Q2); SPF/E216 + populated-SLDomain (Q3). |
| R8 | Report before/intermediate/after state | ✅ Pass | BEFORE (empty schema / `public_domain`=0) and AFTER (`alias`=0) states captured. |
| R9 | Pick the implementation that manifests the behavior | ✅ Pass | Decision chain traced to `handle_forward` → `status.E515` (§4.5). |
| R10 | Include actual, complete, unedited output per condition | ✅ Pass | Full 49-column traceback (no ellipsis); complete `swaks` transcript. |
| R11 | Honor explicit & directional instructions | ✅ Pass | Dev-vs-prod SMTP port stated (20381 vs 25); honest findings reported, not forced. |
| R12 | Show observed output for every claim; label inferred vs observed | ✅ Pass | §5.4: exactly **1** item labeled `(inferred)` (Q1 `load_user` alternate trigger). |
| R13 | Answer every part and every named item | ✅ Pass | §5.3 named-item coverage pass maps every named item in Q1/Q2/Q3. |
| R14 | Be exact & grounded (`file:line`, function names) | ✅ Pass | 191 `file:line` citations; zero discrepancies on re-verification. |
| R15 | Provide reasoning/rationale per answer | ✅ Pass | §2.7, §3.4, §4.5/§4.7 provide explicit rationale. |
| R16 | Read-only scope; remove temp scripts | ✅ Pass | `git diff` = 1 file added, 0 source modified; no temp scripts tracked; `.env` gitignored. |

**Fixes applied during autonomous validation:** None required — the document was accurate and complete as authored; the validator made no edits (consistent with the read-only mandate). **Outstanding compliance items:** None.

---

## 6. Risk Assessment

All risks are **Low severity** — a read-only documentation deliverable ships no code and changes no system behavior. Risks are inherent to a point-in-time investigation and are mitigated or accepted with grounded evidence.

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Citation/version staleness — 191 `file:line` citations pinned to HEAD `2cd6ee777f8c`; future commits shift line numbers | Technical | Low | Medium | Doc pins commit SHA + exact version matrix (§1.1) | Accepted |
| Byte-fidelity vs environment — exact outputs (5749-byte 500 page, tracebacks, banners) depend on pinned stack | Technical | Low | Low | Full version matrix + Docker image pinned in doc | Mitigated |
| Transient `.env` holds dev credentials (`myuser`/`mypassword`, `FLASK_SECRET`) | Security | Low | Low | `.env` + `.env.*` gitignored; confirmed NOT tracked; never committed | Mitigated |
| No production security surface — no code shipped, no auth/authz change, no dependency added | Security | Low | Low | Read-only investigation; scope forbids source change | N/A |
| `pre-commit` `trailing-whitespace` hook could strip 20 intentional code-fence evidence lines | Operational | Low | Low | No active pre-commit hook (`.git/hooks` = git-lfs only); whitespace is inside captured-output fences | Mitigated |
| No automated markdown-lint/doc-CI gate for future edits | Operational | Low | Low | Current doc manually validated; single-file scope | Accepted |
| Q3 reproduction requires `swaks` (observation tool, not a project dep) | Integration | Low | Low | Doc §5.2 documents usage + authoritative refs; confirmed absent from `pyproject`/`poetry.lock` | Mitigated |
| Q2 readiness proofs assume default dev ports (7777/20381/5432/6379) | Integration | Low | Low | Doc states exact ports + dev-vs-prod distinction (20381 vs 25) | Mitigated |

---

## 7. Visual Project Status

**Project hours (Completed = Dark Blue `#5B39F3`, Remaining = White `#FFFFFF`):**

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieOuterStrokeWidth':'2px','pieSectionTextColor':'#B23AF2','pieTitleTextSize':'16px'}}}%%
pie showData title Project Hours Breakdown (Total 36h)
    "Completed Work" : 33
    "Remaining Work" : 3
```

**Remaining hours by category (from Section 2.2 — sums to 3h):**

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#B23AF2','pie2':'#A8FDD9','pieStrokeColor':'#5B39F3','pieStrokeWidth':'2px','pieSectionTextColor':'#333333','pieTitleTextSize':'14px'}}}%%
pie showData title Remaining Work by Category (3h)
    "Human review & sign-off [High]" : 2.5
    "PR approval & merge [Medium]" : 0.5
```

> **Integrity check:** Pie "Remaining Work" (3) = Section 1.2 Remaining Hours (3) = Section 2.2 total (3). Pie "Completed Work" (33) = Section 2.1 total (33). Completed + Remaining = 36 = Total.

---

## 8. Summary & Recommendations

**Achievements.** The project is **91.7% complete** (33 of 36 hours). Blitzy autonomously brought up the SimpleLogin dev stack from scratch and produced a single, comprehensive 1,015-line answer document that resolves all three user questions with real, byte-accurate captured evidence: the Q1 empty-schema `ProgrammingError`/`UndefinedTable`, the Q2 two-tier required-services map with readiness banners and port-binding proofs, and the Q3 `550 SL E515` SMTP rejection with its explanatory log line. The document further surfaced three honest, non-obvious findings (MEM_STORE_URI Redis behavior, the SPF/E216 caveat, and the "empty-`SLDomain`-is-contextual-not-causal" correction) and passed a full code-review cycle that replaced every abbreviated block with complete unedited output.

**Remaining gaps (critical path to production).** The only remaining work is **human path-to-production**: a technical review/sign-off of the document (2.5h) and PR approval/merge (0.5h). There is no outstanding autonomous work, no failing check, and no unresolved issue.

**Success metrics.** 204/204 validation checks passed (100%): all three scenarios reproduced 3× byte-stable, and all 191 citations verified with zero discrepancies. The read-only mandate is perfectly preserved (`git diff` = exactly one file added).

**Production-readiness assessment.** The deliverable is **ready for human acceptance.** Because it is documentation (not shippable code), "production" means merging the reviewed answer document. Recommended path: (1) technical review, (2) approve & merge, (3) optionally reproduce one scenario independently for confidence. Confidence is **High** — the scope is narrow, fully delivered, and independently re-verified at runtime.

| Metric | Value |
|--------|-------|
| AAP-scoped completion | 91.7% (33/36h) |
| Scenarios reproduced | 3/3 (byte-stable, 3× each) |
| Validation checks passed | 204/204 (100%) |
| Citations verified | 191/191 (0 discrepancies) |
| Source files modified | 0 (read-only preserved) |
| Blocking issues | 0 |

---

## 9. Development Guide

This guide brings up the SimpleLogin dev stack and reproduces the three investigations. Commands were verified against the repository at HEAD `2cd6ee777f8c`.

### 9.1 System Prerequisites

- **Python 3.10** (`pyproject.toml`: `python = "^3.10"`; container used 3.10.18)
- **Poetry** (or the prebuilt venv shipped in the Docker image)
- **PostgreSQL 13** (used 13.23) — database `simplelogin`, user `myuser`/`mypassword`
- **Redis** on **:6379** (used 7.4.9)
- **swaks** (SMTP transaction tester) — needed only to reproduce Q3; it is an observation tool, **not** a project dependency
- *(Optional)* Docker image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`

### 9.2 Environment Setup

```bash
# From the repository root — create the transient .env (gitignored)
cp example.env .env
```

Required keys (read via `os.environ[...]`; missing keys raise `KeyError` at import — `app/config.py:79,92,192`):

```bash
URL=http://localhost:7777                                            # example.env:6
EMAIL_DOMAIN=sl.local                                                # example.env:22
DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin     # example.env:75
NOT_SEND_EMAIL=true                                                  # example.env:19
FLASK_SECRET=secret                                                  # example.env:77
# MEM_STORE_URI intentionally unset (see doc §3.2.4)
```

### 9.3 Dependency Installation

```bash
# Canonical install (per Dockerfile); or simply activate the prebuilt venv
poetry install --no-interaction --no-ansi --no-root
```

### 9.4 Database-State Recipes (choose one per scenario)

```bash
# Q1 — Empty schema (do NOT run alembic). Valid because create_all appears nowhere:
echo 'drop schema public cascade; create schema public;' | psql "$DB_URI"
grep -rn "create_all" server.py app/ | wc -l          # expect: 0

# Q3 — Migrated but uninitialized (skip init_app.py):
echo 'drop schema public cascade; create schema public;' | psql "$DB_URI"
alembic upgrade head                                   # 255 scripts
psql "$DB_URI" -c "SELECT count(*) FROM public_domain;"  # expect: 0

# Q2 — Fully initialized:
echo 'drop schema public cascade; create schema public;' | psql "$DB_URI"
alembic upgrade head
python init_app.py                                     # logs: Add sl.local to SL domain
```

### 9.5 Application Startup & Verification

```bash
# Essential services
python server.py            # Flask web app on :7777 (dev)
python email_handler.py     # aiosmtpd SMTP handler on dev port :20381

# Auxiliary background workers (bind no TCP port)
python job_runner.py
python cron.py -j sanity_check
python event_listener.py

# Verify each is up
curl -s http://localhost:7777/health           # expect: success
pg_isready -h localhost -p 5432                 # expect: accepting connections
redis-cli -p 6379 ping                          # expect: PONG
# email handler stdout should show:
#   Listen for port 20381
#   Start mail controller 0.0.0.0 20381
```

### 9.6 Example Usage — Reproduce the Three Answers

```bash
# Q1: with the EMPTY schema and server.py running, submit a login POST
curl -s -o /dev/null -w "%{http_code}\n" \
  -X POST http://localhost:7777/auth/login \
  -d 'email=a@b.c&password=x'                   # expect: 500 (ProgrammingError/UndefinedTable in server log)

# Q3: with the MIGRATED-BUT-UNINITIALIZED schema and email_handler.py running, inject an email
swaks --to e1@sl.local --from hey@google.com --server 127.0.0.1:20381
#   expect reply: 550 SL E515 Email not exist   (CONTRIBUTING.md:218)
#   expect log:   alias e1@sl.local cannot be created on-the-fly, return 550  (email_handler.py:551)
```

### 9.7 Read-Only Verification (for reviewers)

```bash
# Confirm exactly one file added and zero source files modified
git diff --name-status 2cd6ee77..HEAD           # expect: A  blitzy/documentation/app_2cd6ee777f8c.md
test -f blitzy/documentation/app_2cd6ee777f8c.md && wc -l blitzy/documentation/app_2cd6ee777f8c.md
git status --porcelain                          # expect: (empty — clean working tree)
```

### 9.8 Troubleshooting

- **`KeyError: 'URL'` / `'EMAIL_DOMAIN'` / `'DB_URI'` at import** → the `.env` is missing or a required key is absent. Re-create it from `example.env` (§9.2).
- **`relation "users" does not exist`** on any page hitting the DB → migrations not applied. Run `alembic upgrade head`. (This is the *expected* Q1 behavior against an empty schema.)
- **Werkzeug `* Running on ...` line is absent** → expected; `app/log.py:70-71` disables the werkzeug logger.
- **Every `@sl.local` email returns `550`** → expected in the Q3 state (`public_domain` empty and no matching alias). Note §4.7: even after `init_app.py` populates `SLDomain`, a bare `@sl.local` recipient with no alias still returns 550 — the empty `SLDomain` is *contextual*, not the direct cause.
- **`swaks: command not found`** → install `swaks` (it is an observation tool, not part of the project deps).

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---------|---------|
| Create transient env | `cp example.env .env` |
| Install deps | `poetry install --no-interaction --no-ansi --no-root` |
| Reset schema (empty) | `echo 'drop schema public cascade; create schema public;' \| psql "$DB_URI"` |
| Run migrations | `alembic upgrade head` |
| Initialize domains | `python init_app.py` |
| Start web app | `python server.py` |
| Start email handler | `python email_handler.py` |
| Inject test email | `swaks --to e1@sl.local --from hey@google.com --server 127.0.0.1:20381` |
| Health check | `curl -s http://localhost:7777/health` |
| Read-only diff check | `git diff --name-status 2cd6ee77..HEAD` |

### B. Port Reference

| Service | Port | Notes |
|---------|------|-------|
| Flask web app (dev) | 7777 | `server.py` → `app.run(debug=True, port=7777)` (`server.py:588`) |
| SMTP email handler (dev) | 20381 | `email_handler.py` argparse `default=20381` (`email_handler.py:2399`); **prod = 25** |
| PostgreSQL | 5432 | `DB_URI` (`example.env:75`); separate `sl-test-db` on 15432 for pytest only (not used) |
| Redis | 6379 | Sessions/rate-limit backing (`CONTRIBUTING.md:65`) |
| MailHog (optional) | 1080 | Dev mail viewer (`CONTRIBUTING.md:221`); not required for the rejection path |

### C. Key File Locations

| File | Role |
|------|------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The deliverable** (1,015 lines / 74,702 bytes) |
| `server.py` | Web entry — `local_main()`/`create_app()` (`:572-599`, `:139-217`) |
| `email_handler.py` | SMTP entry — `handle_forward` returns `status.E515` (`:551`) |
| `init_app.py` | `add_sl_domains()` seeds `SLDomain` (`:39-62`) |
| `app/models.py` | `class SLDomain` → table `public_domain` (`:3116-3119`) |
| `app/email/status.py` | `E515 = "550 SL E515 Email not exist"` (`:51`) |
| `app/config.py` | Required env keys `URL`/`EMAIL_DOMAIN`/`DB_URI` (`:79,92,192`) |
| `app/auth/views/login.py` | Login view; first ORM query on POST (`:43`) |
| `migrations/versions/*` | 255 Alembic scripts building the schema |

### D. Technology Versions

| Component | Version |
|-----------|---------|
| Python | 3.10.18 (`pyproject.toml` `^3.10`) |
| Flask | 1.1.2 |
| Werkzeug | 1.0.1 |
| SQLAlchemy | 1.3.24 |
| psycopg2(-binary) | 2.9.3 |
| aiosmtpd | 1.4.2 |
| redis (py) | 4.6.0 |
| PostgreSQL | 13.23 |
| Redis (server) | 7.4.9 |
| swaks | 20201014.0 |

### E. Environment Variable Reference

| Key | Value (dev) | Source | Effect |
|-----|-------------|--------|--------|
| `URL` | `http://localhost:7777` | `example.env:6` | Required at import (`app/config.py:79`) |
| `EMAIL_DOMAIN` | `sl.local` | `example.env:22` | Required at import (`:92`); default alias domain |
| `DB_URI` | `postgresql://myuser:mypassword@localhost:5432/simplelogin` | `example.env:75` | Required at import (`:192`) |
| `NOT_SEND_EMAIL` | `true` | `example.env:19` | Prints outbound mail instead of sending |
| `FLASK_SECRET` | `secret` | `example.env:77` | Flask session secret |
| `MEM_STORE_URI` | *(unset)* | — | When unset, web app opens no Redis connection in dev (§3.2.4) |

### F. Developer Tools Guide

- **`swaks`** — SMTP transaction tester; `--server host:port` selects the target, transcript prints the server reply (`<**` marks an unexpected 5xx). Used for Q3 injection. Authoritative: jetmore.org/john/code/swaks.
- **`aiosmtpd`** — async SMTP library powering `email_handler.py`; the handler's returned status string becomes the client reply, so `550 SL E515` is the authentic protocol reply, not a log artifact. Authoritative: aiosmtpd.aio-libs.org.
- **`psql`** — verify DB state (`SELECT count(*) FROM public_domain;`).
- **`git diff --name-status`** — confirm the read-only mandate (single file added).

### G. Glossary

| Term | Meaning |
|------|---------|
| AAP | Agent Action Plan — the governing project directive |
| `SLDomain` | SimpleLogin domain model; physical table `public_domain` |
| E515 | SMTP status constant `550 SL E515 Email not exist` |
| Migrated-but-uninitialized | Schema built via `alembic upgrade head` but `init_app.py` skipped → `public_domain` empty |
| Readiness banner | A process's stdout/stderr line proving it bound its port and is ready |
| Essential vs auxiliary | Tiering of Q2 services: essential (web/email/PG/Redis) vs auxiliary background workers |
| Read-only mandate | The AAP constraint that no source file may be created/modified/deleted except the answer doc |
