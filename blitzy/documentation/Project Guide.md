# Blitzy Project Guide

**Project:** SimpleLogin — Custom-Alias Signed-Suffix Validation & Alias-Creation-Limit Runtime Investigation
**Deliverable:** `blitzy/documentation/app_2cd6ee777f8c.md`
**Branch:** `blitzy-9b4ce125-7165-4a3a-b9c5-9be93b399ff9` (base `app_2cd6ee777f8c`) · **HEAD:** `79a06da7`
**Task type:** Runtime Q&A Investigation (Documentation) · **Rule set:** SWE-AtlasQnA-Repo

> **Legend / Brand Colors** — <span style="color:#5B39F3">**Completed / AI Work = Dark Blue `#5B39F3`**</span> · Remaining / Not Completed = White `#FFFFFF` · Headings/Accents = Violet-Black `#B23AF2` · Highlight = Mint `#A8FDD9`

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a single runtime-observed investigation report explaining precisely how SimpleLogin's custom-alias-creation API behaves when a signed suffix is validated and when the free-plan alias-creation limit is enforced. The target audience is the SimpleLogin engineering team investigating intermittent validation failures on the custom-alias endpoints. The deliverable answers six discrete sub-questions (Q1–Q6) with directly observed HTTP responses, server-console logs, and precise `file:line` references, and it identifies the root cause of the reported symptom. The technical scope is a cross-cutting, read-only investigation of the alias-creation subsystem (API layer, signed-suffix validator, quota model, rate limiter, and logging), executed by running the real application — the source repository is left byte-for-byte unchanged.

### 1.2 Completion Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeWidth':'2px','pieTitleTextSize':'16px','pieSectionTextSize':'14px','pieLegendTextSize':'14px'}}}%%
pie showData
    title Completion — 92.3% Complete
    "Completed Work (h)" : 48
    "Remaining Work (h)" : 4
```

| Metric | Hours |
|---|---|
| **Total Hours** | **52** |
| Completed Hours (AI + Manual) | 48 (AI: 48 · Manual: 0) |
| Remaining Hours | 4 |
| **Percent Complete** | **92.3%** |

> Completion is computed on AAP-scoped work only (PA1): `48 / (48 + 4) = 48/52 = 92.3%`. All autonomous investigation work is complete and validated; the remaining 4 hours is inherently-human review/acceptance.

### 1.3 Key Accomplishments

- ✅ **All six sub-questions (Q1–Q6) answered** from directly observed runtime output, not code reading alone.
- ✅ **Real canonical path exercised** — endpoints `POST /api/v2/alias/custom/new` and `POST /api/v3/alias/custom/new` driven through the full decorator + auth stack, via both `server.create_app()` (in-process) and production `gunicorn wsgi:app` on port 7777.
- ✅ **Root cause identified** — the `itsdangerous.BadSignature` catch collapses expired/tampered/malformed/empty suffixes into a single HTTP 412, making the intended HTTP 400 "Tampered suffix" branch dead code.
- ✅ **Intermittency reproduced honestly** — the same unchanged input is deterministic (always 412); the base64 trailing-bit tamper-generator nuance is disclosed.
- ✅ **Negative finding confirmed for Q4** — no `X-RateLimit-*` / `Retry-After` headers are emitted on any response status.
- ✅ **Read-only mandate upheld** — `git status` clean; the diff versus the base branch is exactly one new file; all temporary observation scripts removed; DB/Redis residue reconciled to zero.
- ✅ **Grounding verified** — all 89 unique `file:line` references confirmed against the container source (0 out-of-bounds, 0 missing).

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|---|---|---|---|
| _None blocking deliverable acceptance_ | The report is complete, validated, and requires zero corrections. | — | — |
| (Informational) Documented dead-code defect: 412 swallows tampered/expired suffixes | Product behavior, not a deliverable defect. **Documented, not fixed (out of scope).** Requires a human triage decision. | Product/Eng | Post-review |

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|---|---|---|---|---|
| Git repository | Write (branch) | None — deliverable committed on the branch; working tree clean | ✅ Resolved | — |
| Canonical Docker container (`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`) | Pull/run | None — image available; PostgreSQL 15 + Redis provisioned by `/build.sh` | ✅ Resolved | — |
| PostgreSQL / Redis (container services) | Service | None — both online and exercised during validation | ✅ Resolved | — |

**No access issues identified** that would prevent build validation, integration, or acceptance of this deliverable. (The product's Apple receipt-verification external call is a product dependency, not a deliverable access issue.)

### 1.6 Recommended Next Steps

1. **[High]** Review and accept the investigation report (`blitzy/documentation/app_2cd6ee777f8c.md`) — confirm Q1–Q6 answers, the central finding, and the coverage pass meet stakeholder needs.
2. **[Low]** Optionally run an independent spot-check reproduction in the canonical container (e.g., POST a tampered suffix → expect HTTP 412) for confidence before wider distribution.
3. **[Low]** Make a triage decision on the documented dead-code finding — decide whether to open a **separate, out-of-scope** product ticket to fix the 412-vs-400 branch.
4. **[Low]** (Out of scope, optional) Backlog the surfaced product follow-ups: patch flagged dependency advisories and add a timeout to the `apple.py` external call that hangs the full test suite.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|---|---:|---|
| Canonical runtime provisioning & app boot | 5.0 | Docker container, PostgreSQL 15 (+pg_trgm), Redis, in-process `create_app()` client, live `gunicorn wsgi:app`, real API-auth token (AAP §0.2.3, report §2.1–2.5, §2.9–2.10) |
| Q1 — Invalid-suffix investigation | 3.0 | Tampered/malformed/empty suffixes across v2+v3; exact status + body capture (report §3.1) |
| Q2 — Expired-suffix investigation | 2.0 | Suffix age > 600 s across v2+v3; determinism confirmation (report §3.2) |
| Q3 — Validation-log capture | 2.5 | `SL` logger stdout capture per condition; bounded positive/negative counts (report §3.3) |
| Q4 — Rate-limit-header investigation | 3.5 | Full header scan across statuses, 429 breach body, security/cache headers, Flask-Limiter behavior research (report §3.4) |
| Q5 — Quota-check trace | 4.0 | `can_create_new_alias` / `max_alias_for_free_account` / token bucket / exact boundary (report §3.5) |
| Q6 — Execution-path trace | 3.5 | Decorator stack, 12-branch table, flowchart, parallel-lock key (report §3.6, §4) |
| Central finding — root cause | 4.0 | `itsdangerous` `BadSignature` collapse; non-canonical library stand-in; dashboard path (report §5) |
| Intermittency reproduction | 3.0 | Same unchanged input, multi-process repeats, determinism proof, base64 nuance (report §6) |
| `file:line` verification + coverage pass | 3.0 | 89 references verified; Q1–Q6 + named-element checklist (report §7) |
| Read-only guarantee + stateful cleanup | 2.0 | Byte-for-byte repo integrity, DB/Redis net-zero cleanup, sanitization (report §2.8, §8) |
| Report authoring | 6.0 | 2,063-line comprehensive markdown: exec summary, methodology, answers, trace, appendix |
| Test-suite & dependency-advisory observations | 1.5 | Focused-test state-leak, full-suite hang, pip/npm audit advisories (report §2.11–2.12) |
| Iterative QA hardening (6 commits) | 5.0 | Resolving code-review + QA findings across the branch history |
| **Total Completed** | **48.0** | Matches Completed Hours in §1.2 |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|---|---:|---|
| Report Review & Acceptance (stakeholder sign-off) | 2.0 | High |
| Independent Verification (spot-check reproduction in canonical container) | 1.5 | Low |
| Defect Triage Decision (documented dead-code finding; **fix is out of scope**) | 0.5 | Low |
| **Total Remaining** | **4.0** | Matches Remaining Hours in §1.2 and §7 |

### 2.3 Out-of-Scope Follow-Ups (Not Counted in Hours)

These arise from the report's findings but are **explicitly out of AAP scope** and are **excluded** from all hour totals and the completion percentage. They are listed for the product team's backlog only:

| Follow-up | Est. (if pursued) | Notes |
|---|---|---|
| Fix the 412-vs-400 dead-code branch (`alias_suffix.py::check_suffix_signature`) | 2–4h | Report documents, does not fix. |
| Patch/upgrade flagged dependency advisories (pip + npm audit) | Variable | Product concern; surfaced in §2.12 of the report. |
| Add timeout to Apple receipt-verification external call (`apple.py:319`) | 1–2h | Pre-existing, unrelated; unblocks full-suite CI. |

---

## 3. Test Results

All tests below originate from Blitzy's autonomous validation logs for this project. The deliverable is a Markdown report (nothing to compile); the project's own alias-creation unit tests were exercised as runtime observation instruments, and the endpoints were driven across every condition as functional runtime observations.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---|---|---:|---:|---:|---:|---|
| Unit — alias-creation module (`tests/api/test_new_custom_alias.py`) | pytest 6.x | 10 | 10 | 0 | N/A | Isolated word-pool run (env-only `WORDS_FILE_PATH`, /tmp); exit 0, ~6.5 s. Canonical result. |
| Runtime — endpoint conditions (Q1–Q6) | curl + Flask test client / gunicorn | 8 conditions | 8 | 0 | N/A | fresh/valid→201, expired→412, tampered→412, malformed→412, empty→412, quota-exceeded→400, duplicate→409, wrong-prefix→400; reproduced on v2 **and** v3 |
| Runtime — determinism / repeat | curl (multi-process) | 5 repeats | 5 | 0 | N/A | Same unchanged input → 412 every time (deterministic) |

**Documented environment artifact (not a defect):** running the alias module against the reused canonical DB with the canonical 3-word `WORDS_FILE_PATH` produces **2 of 10** `409` duplicate collisions (`test_v2`, `test_minimal_payload`) — a shared-DB state-leak + tiny-word-pool interaction, root-caused in report §2.11(a); it disappears with an isolated word pool (10/10 pass).

**Out-of-scope, pre-existing blocker:** the full 639-test suite hangs on `test_apple_process_payment` due to an un-timed external HTTP call at `apple.py:319` — unrelated to the alias-creation investigation and protected by the read-only mandate (report §2.11(b)).

---

## 4. Runtime Validation & UI Verification

This is a headless API investigation — there is no UI surface. Runtime health and API-integration outcomes were verified across two entry points.

**Application runtime**
- ✅ **Operational** — Python 3.10.18; app imports and boots cleanly via `server.create_app()` (in-process) and live `gunicorn wsgi:app -b 127.0.0.1:7777 -w 2 --timeout 15`.
- ✅ **Operational** — PostgreSQL 15.13 online (`pg_trgm 1.6`, `alembic_version` at head); Redis online (`PONG`).
- ✅ **Operational** — clean stop/start lifecycle observed; behavior stable post-restart (412/201 reproduced from fresh processes).

**API integration outcomes (real endpoints under API auth)**
- ✅ **Operational** — Valid signed suffix → **HTTP 201** alias JSON.
- ✅ **Operational** — Expired / tampered / malformed / empty suffix → **HTTP 412** `{"error":"Alias creation time is expired, please retry"}` (Content-Length 57).
- ✅ **Operational** — Free-plan quota exhausted → **HTTP 400** "You have reached the limitation of a free account…".
- ✅ **Operational** — Duplicate alias → **HTTP 409**; missing/invalid token → **HTTP 401** "Wrong api key".
- ✅ **Operational** — Rate-limit breach → **HTTP 429** `{"error":"Rate limit exceeded"}` (no rate-limit headers present).
- ⚠ **Partial (by design)** — The intended HTTP 400 "Tampered suffix" branch is **unreachable** for signature errors (central finding); documented, not fixed.

**UI verification:** ❌ Not applicable — no front-end/UI component in this task (AAP §0.9: no Figma/design assets).

---

## 5. Compliance & Quality Review

Cross-mapping of AAP deliverables and rule-set directives to observed compliance.

| Benchmark / Deliverable | Status | Progress | Evidence / Notes |
|---|:---:|:---:|---|
| Q1 — invalid-suffix status + body | ✅ Pass | 100% | Report §3.1; HTTP 412 body verified at source |
| Q2 — expired-suffix status + body | ✅ Pass | 100% | Report §3.2; deterministic across repeats |
| Q3 — validation log entries | ✅ Pass | 100% | `new_custom_alias.py:72`/`:187`; tamper-log count = 0 |
| Q4 — rate-limit headers | ✅ Pass | 100% | Negative finding confirmed; limiter built without `headers_enabled` |
| Q5 — quota checks on success | ✅ Pass | 100% | `models.py:867-884`/`:858-865`; no success logging |
| Q6 — execution-path trace | ✅ Pass | 100% | Validator + enforcer + decorator stack + branch table |
| Run-first methodology | ✅ Pass | 100% | Observations captured before conclusions (§2.7) |
| Real canonical path (not helper bypass) | ✅ Pass | 100% | Two entry points; API-auth token (§2.5, §2.9) |
| Reproduce actual intermittency | ✅ Pass | 100% | Same input repeated; distribution reported (§6) |
| Exact & grounded (`file:line` + function) | ✅ Pass | 100% | 89 references verified, 0 out-of-bounds |
| Configuration disclosure (`MAX_NB_EMAIL_FREE_PLAN`) | ✅ Pass | 100% | 3 (test) vs 5 (example) stated per observation (§2.6) |
| Coverage pass (all sub-questions + named elements) | ✅ Pass | 100% | Report §7.1–§7.2 |
| Read-only source repository | ✅ Pass | 100% | `git status` clean; diff = 1 file; scripts removed |
| Single deliverable at correct path/name | ✅ Pass | 100% | `blitzy/documentation/app_2cd6ee777f8c.md` |
| Human review & acceptance | ⬜ Pending | 0% | Path-to-production; inherently human (4h) |

**Fixes applied during autonomous validation:** the report was hardened across 6 commits resolving code-review and QA findings (e.g., DB net-residue claim correction, token-bucket wiring correction, 11 QA findings, 3 MINOR findings). The Final Validator required **zero further deliverable corrections** — every claim, all 89 `file:line` references, and all structural/behavioral assertions were confirmed accurate at runtime.

**Outstanding items:** human review/acceptance only (Section 2.2).

---

## 6. Risk Assessment

The deliverable itself (a validated Markdown report) carries near-zero delivery risk. The material risks are (a) deliverable acceptance and (b) product-level findings the report surfaces for the team to act on.

| Risk | Category | Severity | Probability | Mitigation | Status |
|---|---|:---:|:---:|---|---|
| Documented dead-code defect (412 swallows tampered/expired) left unaddressed | Technical | Medium | High | Report flags it as root cause; recommend follow-up ticket. **Out of scope to fix.** | Documented / Open (product) |
| Findings depend on pinned `itsdangerous 1.1.0` exception hierarchy | Technical | Low | Low | Versions pinned; version-dependence disclosed (§2.3, §5.1) | Mitigated |
| Quota numbers depend on `MAX_NB_EMAIL_FREE_PLAN` (3 vs 5) | Technical | Low | Low | Config disclosed per observation (§2.6) | Mitigated |
| Pinned-dependency advisories (pip + npm audit) unpatched | Security | Medium | Medium | Surfaced as supplementary observations (§2.12). Product concern, out of scope | Documented (informational) |
| Credential leakage in captured artifacts | Security | Low | Very Low | API keys redacted throughout; sanitization statement (§8.2) | Mitigated |
| Reproduction requires exact container + PG + Redis | Operational | Low | Medium | Exact image ID + build/invocation commands provided (§2.1, §9) | Mitigated |
| Full 639-test suite hangs (un-timed `apple.py` external call) | Operational | Low | Medium | Documented; recommend separate remediation (§2.11b). Out of scope | Documented (out of scope) |
| Product makes un-timed external HTTP call (Apple receipt) | Integration | Low | Low | Documented; product fragility, out of scope | Documented (product) |
| Deliverable integration surface | Integration | None | — | Standalone Markdown; no build/deploy/runtime deps | N/A |

---

## 7. Visual Project Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeWidth':'2px','pieTitleTextSize':'16px','pieSectionTextSize':'14px','pieLegendTextSize':'14px'}}}%%
pie showData
    title Project Hours Breakdown (Total 52h)
    "Completed Work" : 48
    "Remaining Work" : 4
```

**Remaining hours by category (from Section 2.2):**

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#B23AF2','pie2':'#A8FDD9','pie3':'#5B39F3','pieStrokeColor':'#333333','pieOuterStrokeColor':'#333333','pieTitleTextSize':'15px','pieSectionTextSize':'13px','pieLegendTextSize':'13px'}}}%%
pie showData
    title Remaining Work — 4h
    "Report Review & Acceptance [High]" : 2.0
    "Independent Verification [Low]" : 1.5
    "Defect Triage Decision [Low]" : 0.5
```

> **Integrity check:** "Remaining Work" = **4** here, in the §1.2 metrics table, and as the sum of the §2.2 "Hours" column. "Completed Work" = **48** = §2.1 total. 48 + 4 = 52 = Total Hours.

---

## 8. Summary & Recommendations

**Achievements.** The project delivers a complete, runtime-grounded answer to all six sub-questions plus the root cause of the reported symptom. The investigation was performed against the real, canonical application (both the in-process factory and the production `gunicorn wsgi:app` transport), with every behavioral claim backed by directly observed, unedited output and a verified `file:line` reference. The central discovery — that `check_suffix_signature`'s `except itsdangerous.BadSignature` clause collapses expired, tampered, malformed, and empty suffixes into a single HTTP 412, rendering the intended HTTP 400 "Tampered suffix" branch dead code — cleanly explains why a caller expecting a 400 for a corrupted token always sees a 412 "…is expired…". Per scope, this is documented, not remediated.

**Remaining gaps.** All autonomous, AAP-scoped work is complete and independently validated (zero corrections required). The only remaining work is human: review and acceptance of the report (2h), an optional independent spot-check reproduction (1.5h), and a triage decision on the documented dead-code finding (0.5h) — **4 hours total**.

**Critical path to production.** For a documentation deliverable, "production" means stakeholder acceptance. The path is short and unblocked: review → (optional) spot-check → sign-off. No build, deployment, integration, or configuration work is required to ship the report.

**Production readiness assessment.** The deliverable is **ready for review** at **92.3% complete** (`48 / 52` AAP-scoped hours). The source repository is byte-for-byte unchanged, the read-only mandate is fully honored, and the coverage pass confirms every sub-question and named element is addressed.

| Success Metric | Target | Observed |
|---|---|---|
| Sub-questions answered | 6 / 6 | 6 / 6 ✅ |
| Claims grounded in observed output + `file:line` | 100% | 100% (89 refs verified) ✅ |
| Source repository unchanged | Byte-for-byte | ✅ (diff = 1 file) |
| Deliverable corrections required at validation | 0 | 0 ✅ |
| Alias-creation unit tests (isolated pool) | 10 pass | 10 pass ✅ |

---

## 9. Development Guide

This guide reproduces the **runtime observations** behind the report. The deliverable (Markdown) itself needs no build/run. All commands are copy-pasteable and grounded in verified source and captured output. **Respect the read-only mandate:** use process-local environment variables and `/tmp` files only — never modify repository files.

### 9.1 System Prerequisites

- **Docker Engine** (to run the canonical container).
- The container itself provides: **Python 3.10**, **PostgreSQL 15** (with `pg_trgm`), **Redis**, and Poetry-installed dependencies. No host language toolchain is required to reproduce.

### 9.2 Environment Setup (canonical container)

```bash
# Create a long-lived container (ENTRYPOINT is /bin/bash, so override with sleep)
docker run -d --name sl_setup \
  --entrypoint sleep \
  ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0 infinity

# Enter it
docker exec -it sl_setup bash
```

The image's `/build.sh` has already: created the venv, run `poetry install --no-root`, started PostgreSQL + Redis, generated keys, dropped+recreated the schema, run `alembic upgrade head`, and written `local_data/test_words.txt`.

### 9.3 Dependency & Service Verification

```bash
/app/venv/bin/python --version          # Python 3.10.18
/app/venv/bin/gunicorn --version        # gunicorn (version 20.0.4)
/app/venv/bin/pip freeze | grep -iE '^(Flask|Flask-Limiter|limits|itsdangerous|SQLAlchemy|psycopg2-binary|redis)=='
#   Flask==1.1.2  Flask-Limiter==1.4  limits==1.5.1  itsdangerous==1.1.0
#   SQLAlchemy==1.3.24  psycopg2-binary==2.9.3  redis==4.6.0

PGPASSWORD=test psql -h localhost -U test -d test -tAc "select version();"   # PostgreSQL 15.13
PGPASSWORD=test psql -h localhost -U test -d test -tAc "select extversion from pg_extension where extname='pg_trgm';"  # 1.6
redis-cli ping                          # PONG
```

### 9.4 Application Startup (two canonical entry points)

**(a) In-process (primary):** build a Flask test client from `server.create_app()`.

```bash
CONFIG=/app/tests/test.env \
DB_URI=postgresql://test:test@localhost:5432/test \
PYTHONPATH=/app \
/app/venv/bin/python   # then: from server import create_app; app = create_app(); client = app.test_client()
```

**(b) Live gunicorn (production form, confirmation):**

```bash
cd /app && \
CONFIG=tests/test.env \
DB_URI=postgresql://test:test@localhost:5432/test \
DISABLE_RATE_LIMIT=1 EVENT_WEBHOOK_DISABLE=1 PYTHONPATH=/app \
/app/venv/bin/gunicorn wsgi:app -b 127.0.0.1:7777 -w 2 --timeout 15
```

### 9.5 Verification & Example Usage

```bash
# Exercise the real endpoint (redact your API key). Repeat for /api/v3/... .
curl -s -D - -o /tmp/body -X POST http://127.0.0.1:7777/api/v2/alias/custom/new \
  -H "Authentication: <API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"alias_prefix":"livev2ok","signed_suffix":"<VALID_SIGNED_SUFFIX>"}'
```

Expected outcomes:

| Condition | Status | Body |
|---|---|---|
| Valid suffix | 201 | alias JSON |
| Expired / tampered / malformed / empty suffix | 412 | `{"error":"Alias creation time is expired, please retry"}` |
| Quota exhausted (`MAX_NB_EMAIL_FREE_PLAN=3`) | 400 | "You have reached the limitation of a free account…" |
| Duplicate alias | 409 | `alias … already exists` |
| Missing/invalid token | 401 | `{"error":"Wrong api key"}` |
| Rate-limit breach | 429 | `{"error":"Rate limit exceeded"}` (no rate-limit headers) |

**Run the alias-creation unit tests (isolated word pool → 10 pass):**

```bash
DB_URI=postgresql://test:test@localhost:5432/test \
WORDS_FILE_PATH=/tmp/unique_words.txt \
/app/venv/bin/python -m pytest tests/api/test_new_custom_alias.py \
  --timeout=60 --timeout-method=signal -p no:randomly
# => 10 passed
```

### 9.6 Troubleshooting

- **`DB_URI` port mismatch:** `tests/test.env` pins `:15432`, but the container Postgres listens on `:5432`. Override `DB_URI` to `:5432` at invocation (as above) — do **not** edit `test.env`.
- **`409` duplicate collisions in focused tests:** caused by the reused DB + the canonical 3-word `WORDS_FILE_PATH`. Point `WORDS_FILE_PATH` at an isolated large word list under `/tmp` (env-only) to get 10/10 passing.
- **`pytest tests/` hangs:** the full suite stalls on `test_apple_process_payment` (un-timed external call at `apple.py:319`). Scope pytest to the alias module, or always pass `--timeout=… --timeout-method=signal`.
- **Read-only integrity check:** after any run, `git status --porcelain` should show only the deliverable (or nothing). Remove all `/tmp` observation scripts when finished.

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---|---|
| Create container | `docker run -d --name sl_setup --entrypoint sleep ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0 infinity` |
| Python version | `/app/venv/bin/python --version` |
| Boot (live) | `/app/venv/bin/gunicorn wsgi:app -b 127.0.0.1:7777 -w 2 --timeout 15` |
| Alias unit tests | `/app/venv/bin/python -m pytest tests/api/test_new_custom_alias.py --timeout=60 --timeout-method=signal -p no:randomly` |
| Read-only check | `git status --porcelain` |
| Diff vs base | `git diff --stat origin/app_2cd6ee777f8c...HEAD` |

### B. Port Reference

| Service | Port | Notes |
|---|---|---|
| App (gunicorn) | 7777 | `-b 127.0.0.1:7777`; Dockerfile `EXPOSE 7777` |
| PostgreSQL (container) | 5432 | override `DB_URI` to this port |
| PostgreSQL (`test.env` default) | 15432 | not used inside the container |
| Redis | 6379 | `MEM_STORE_URI=redis://localhost` |

### C. Key File Locations

| Path | Role |
|---|---|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The deliverable** (2,063 lines) |
| `app/api/views/new_custom_alias.py` | Endpoints v2 (`:28-112`) / v3 (`:115-235`); 412/400 branches |
| `app/alias_suffix.py` | `check_suffix_signature` (`:37-42`) — validator (central finding) |
| `app/models.py` | `can_create_new_alias` (`:867-884`), `max_alias_for_free_account` (`:858-865`) |
| `app/extensions.py` | `limiter = Limiter(key_func=__key_func)` (`:23`, no `headers_enabled`) |
| `app/rate_limiter.py` / `app/parallel_limiter.py` | Token bucket / concurrency lock |
| `server.py` (container `/app/server.py`) | `limiter.init_app` (`:167`), 429 handler (`:362-372`) |
| `tests/api/test_new_custom_alias.py` | 10 alias-creation unit tests |

### D. Technology Versions

| Component | Version |
|---|---|
| Python | 3.10.18 |
| Flask | 1.1.2 |
| Flask-Limiter | 1.4 |
| limits | 1.5.1 |
| itsdangerous | 1.1.0 |
| SQLAlchemy | 1.3.24 |
| psycopg2-binary | 2.9.3 |
| redis (client) | 4.6.0 |
| gunicorn | 20.0.4 |
| PostgreSQL | 15.13 (pg_trgm 1.6) |

### E. Environment Variable Reference

| Variable | Value (canonical) | Purpose |
|---|---|---|
| `CONFIG` | `tests/test.env` (or `/app/tests/test.env`) | Canonical test configuration |
| `DB_URI` | `postgresql://test:test@localhost:5432/test` | Override to container Postgres port |
| `MEM_STORE_URI` | `redis://localhost` | Rate-limit + parallel-lock storage |
| `MAX_NB_EMAIL_FREE_PLAN` | `3` (test) / `5` (example default) | Free-plan quota boundary |
| `WORDS_FILE_PATH` | isolated `/tmp` file (for clean test runs) | Alias word pool |
| `DISABLE_RATE_LIMIT` | `1` (for live-boot observation) | Disables the limiter |
| `EVENT_WEBHOOK_DISABLE` | `1` | Suppresses outbound webhooks during observation |
| `FLASK_SECRET` | `secret` | Base for `CUSTOM_ALIAS_SECRET = FLASK_SECRET + "custom_alias"` |

### F. Developer Tools Guide

- **git** — verify read-only integrity: `git status --porcelain`; branch diff: `git diff --stat origin/app_2cd6ee777f8c...HEAD` (expect exactly one file, `+2063/-0`).
- **psql / redis-cli** — service readiness and state inspection (see §9.3).
- **pytest** — scope to `tests/api/test_new_custom_alias.py`; always bound with `--timeout --timeout-method=signal` to avoid the unrelated full-suite hang.
- **curl** — exercise endpoints with `-D -` to dump response headers (used to confirm the Q4 negative finding).

### G. Glossary

| Term | Definition |
|---|---|
| **AAP** | Agent Action Plan — the authoritative project directive. |
| **Signed suffix** | A `TimestampSigner`-signed alias-domain suffix (`max_age=600s`) validated by `check_suffix_signature`. |
| **Central finding** | The `itsdangerous.BadSignature` catch collapsing all bad suffixes into one HTTP 412, making the HTTP 400 "Tampered suffix" branch dead code. |
| **Token bucket** | Per-user `rate_limiter.py::check_bucket_limit` invoked inside `Alias.create`. |
| **Parallel lock** | `parallel_limiter.lock` concurrency guard keyed `cl:<ip>:alias_creation`. |
| **Read-only mandate** | The rule that the source repository must be left byte-for-byte unchanged. |
| **Q1–Q6** | The six investigation sub-questions the report answers. |

---

*Blitzy Project Guide — generated from Blitzy autonomous validation logs. Completed work (AI) shown in Dark Blue `#5B39F3`; remaining work in White `#FFFFFF`.*