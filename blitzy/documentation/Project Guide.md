# Blitzy Project Guide — SimpleLogin Custom-Alias Diagnostic Documentation

> **Project type:** Read-only diagnostic documentation (Q&A) · **Repository:** SimpleLogin · **Branch:** `app_2cd6ee777f8c`
> **Brand legend:** <span style="color:#5B39F3">■ Completed / AI Work (Dark Blue #5B39F3)</span> · <span>□ Remaining / Not Completed (White #FFFFFF)</span> · Headings/Accents Violet-Black #B23AF2 · Highlight Mint #A8FDD9

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a single, evidence-based Markdown document, `blitzy/documentation/app_2cd6ee777f8c.md`, that authoritatively explains the runtime behavior of **SimpleLogin's custom-alias-creation flow** — specifically how *signed suffixes* are verified and how *alias-creation limits* are enforced. It is a **read-only diagnostic**: it answers six precise questions (failure responses, validation logs, rate-limit headers, success-path quota checks, quota logging, and the validating component) to help the team diagnose intermittent validation failures that "don't match expected behavior." Every claim is grounded in source code (`file:line`) and confirmed by live runtime reproduction. No SimpleLogin source was created, modified, or deleted; the only artifact produced is the documentation file.

### 1.2 Completion Status

```mermaid
%%{init: {"theme":"base","themeVariables":{"pie1":"#5B39F3","pie2":"#FFFFFF","pieStrokeColor":"#B23AF2","pieStrokeWidth":"2px","pieOuterStrokeWidth":"2px","pieTitleTextColor":"#B23AF2","pieSectionTextColor":"#111111","pieLegendTextColor":"#111111"}}}%%
pie showData title Project Completion — 92.2% complete (hours)
    "Completed Work (AI) 29.5h" : 29.5
    "Remaining Work (Human) 2.5h" : 2.5
```

| Metric | Value |
|--------|-------|
| **Total Hours** | **32.0 h** |
| **Completed Hours (AI + Manual)** | **29.5 h** (AI 29.5 h + Manual 0.0 h) |
| **Remaining Hours** | **2.5 h** |
| **Percent Complete** | **92.2 %** (29.5 / 32.0) |

> Completion is computed strictly from AAP-scoped work plus path-to-production. The single deliverable is fully authored, validated against code-as-truth, live-reproduced, and committed; the remaining 2.5 h is human review/sign-off (path-to-production for a documentation artifact).

### 1.3 Key Accomplishments

- ✅ **All six questions answered** with code citations *and* rationale (Q1 failure responses, Q2 validation logs, Q3 rate-limit headers, Q4 success-path quota checks, Q5 quota logging, Q6 validating component & rejection conditions).
- ✅ **Headline root cause identified & live-confirmed:** a *tampered/malformed* signed suffix returns the **same** `HTTP 412 {"error":"Alias creation time is expired, please retry"}` as an *expired* one — the single most likely explanation for "failures that don't match expected behavior."
- ✅ **Three independent limiting mechanisms disentangled:** Flask-Limiter HTTP route limit, the bucket quota inside `Alias.create()`, and the per-user/IP concurrency lock — each with distinct triggers.
- ✅ **Rate-limit headers proven absent** (three independent evidence lines + live HTTP confirmation).
- ✅ **Live runtime reproduction** of every answer via Flask test client + gunicorn + curl with PostgreSQL + Redis.
- ✅ **Read-only mandate honored:** SimpleLogin source is byte-for-byte unchanged (1 file added, +850/-0).
- ✅ **Validation correction (D1)** applied: `LOG.e` behavior corrected to the literal `NoneType: None` artifact (not a real traceback).
- ✅ **Document quality:** 850 lines, 43 headings, 14 balanced code-fence pairs, 0 trailing-whitespace lines, 31/31 intra-doc anchors resolve.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| _None blocking._ Deliverable is complete, validated, and committed. | No blocker to delivery; only human sign-off remains. | — | — |
| SME review & sign-off not yet performed (path-to-production, not a defect) | Document cannot be formally "accepted" until a human reviews it | Reviewing engineer / SME | 1.5 h |

> There are **no unresolved technical defects** in the deliverable. The reported SimpleLogin behaviors (F1–F4) are intentionally documented as findings, **not** fixed — remediation is explicitly out of scope under the read-only mandate.

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-----------------|----------------|-------------------|-------------------|-------|
| _None_ | — | No access issues identified. Repository, source, and validation logs were fully accessible; live reproduction completed with PostgreSQL + Redis. | N/A | — |

**No access issues identified.**

### 1.6 Recommended Next Steps

1. **[High]** Review and sign off on `blitzy/documentation/app_2cd6ee777f8c.md`; spot-check the load-bearing citations (§2.2 headline finding, §3.2 `LOG.e` nuance, §5.3 success-path logging). *(~1.5 h)*
2. **[Medium]** Independently re-confirm the headline 412-vs-400 conflation in your own environment before making any remediation decision (fetch a `signed_suffix`, POST a corrupted one, observe 412). *(~1.0 h)*
3. **[Low]** Decide (in a **separate** work order) whether to remediate findings F1–F4 (412/400 conflation, stale "15 aliases" docstring, disabled rate-limit headers, misleading `(hits,days)` comment). *Out of scope for this read-only task.*
4. **[Low]** Add the document to the team knowledge base / runbook so on-call engineers can use the §6.5 diagnostic checklist.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

All completed work was performed autonomously by Blitzy agents (Manual = 0 h). Each component traces to an AAP requirement.

| Component | Hours | Description |
|-----------|-------|-------------|
| C1 — Execution-path tracing & validating-component identification (§1) | 3.5 | Traced suffix issuance (`get_alias_suffixes`→`signer.sign`, 600 s clock start) and both creation endpoints; identified `app/alias_suffix.py` as the validating component (Q6 foundation). |
| C2 — Q1/Q6 failure responses (§2) | 4.0 | Built full v2 (7 conditions) and v3 (12 conditions) rejection tables with exact status codes, literal error bodies, and log calls; isolated the 412-vs-400 conflation; documented the web/dashboard contrast. |
| C3 — Q2 server-console validation logs (§3) | 2.5 | Cataloged every `LOG.*` call on the path with level + line; documented the `SL` logger stdout format and the `LOG.e`→`NoneType: None` nuance. |
| C4 — Q3 rate-limit headers + three-mechanism analysis (§4) | 3.0 | Proved headers absent via three independent evidence lines; disentangled Flask-Limiter / bucket quota / concurrency lock; web research on Flask-Limiter 1.4 header semantics. |
| C5 — Q4/Q5 success-path quota checks & logging (§5) | 2.5 | Traced `can_create_new_alias()` and the bucket quota in `Alias.create()`; established that nothing is logged on a successful under-quota creation. |
| C6 — Intermittency root-cause synthesis + diagnostic checklist (§6) | 2.5 | Synthesized four independent intermittency triggers and a symptom→mechanism→evidence diagnostic checklist. |
| C7 — Live runtime reproduction (Appendix A) | 5.0 | Built/ran SimpleLogin (Flask test client + gunicorn + curl) with PostgreSQL + Redis; reproduced all six answers and exercised all three limiting mechanisms distinctly. |
| C8 — Document authoring, citations (Appendix B), pinned deps (Appendix C), structure & lint | 3.5 | Authored the 850-line evidence-cited document; assembled the citations list; verified fences/whitespace/anchors. |
| C9 — Validation & corrections (D1 + 3 follow-ups) | 3.0 | Code-as-truth verification; D1 `LOG.e` correction (4 occurrences) plus concurrency-lock scope and citation-range corrections across 4 commits. |
| **Total Completed** | **29.5** | **Matches Completed Hours in §1.2.** |

### 2.2 Remaining Work Detail

Path-to-production for a documentation deliverable = human review/sign-off. **Remediation of findings is out of scope** (read-only mandate) and is therefore *not* included below.

| Category | Hours | Priority |
|----------|-------|----------|
| Documentation review & sign-off (SME) | 1.5 | High |
| Independent live re-confirmation of headline finding | 1.0 | Medium |
| **Total Remaining** | **2.5** | **Matches Remaining Hours in §1.2 and §7.** |

> **Excluded from hours (separate work order):** remediation of findings F1–F4. These were reported, not fixed, per the read-only mandate, and are not part of this project's scope or hour totals.

### 2.3 Hours Reconciliation

| Check | Result |
|-------|--------|
| §2.1 Completed total | 29.5 h |
| §2.2 Remaining total | 2.5 h |
| §2.1 + §2.2 | **32.0 h** = Total Hours in §1.2 ✓ |
| Completion % | 29.5 / 32.0 = **92.2 %** ✓ |

---

## 3. Test Results

All tests below originate from **Blitzy's autonomous validation logs** for this project. Because the task is read-only, **zero production code and zero tests were added to the repository** — these are the existing SimpleLogin tests executed as reproduction/reference harnesses during validation.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|-----------|-------|
| API reproduction — `tests/api/test_new_custom_alias.py` (in-scope) | pytest | 10 | 10 | 0 | N/A* | v2/v3 endpoints, API-key auth, limit toggling |
| API reproduction — `tests/api/test_alias_options.py` (in-scope) | pytest | 3 | 3 | 0 | N/A* | Signed-suffix issuance (start of 600 s window) |
| Dashboard reference — `tests/dashboard/test_custom_alias.py` | pytest | 14 | 14 | 0 | N/A* | Web path (flash/redirect); passes on a pristine DB |
| **Total** | **pytest** | **27** | **27** | **0** | **—** | **100 % pass rate** |

\* **Coverage is N/A by design:** this is a read-only documentation task that added no production code, so code-coverage of new code is not a meaningful metric. The in-scope API reproduction suite (13 tests) ran deterministically (`-p no:randomly`). The earlier intermittency seen on the dashboard suite was traced to shared-DB data pollution (tests using `commit=True` with only 3 words in `tests/data/test_words.txt`, producing legitimate 409 "already exists" responses) — **not** a code or documentation defect; on a pristine DB it is 14/14.

---

## 4. Runtime Validation & UI Verification

Runtime validation was performed by building SimpleLogin like `tests/conftest.py` and running it both via an in-process Flask test client and a real **gunicorn 20.0.4** server with **curl** (`CONFIG=tests/test.env`, Redis active, `DISABLE_RATE_LIMIT=False`).

**API runtime health**

- ✅ **Operational** — App boots and serves the alias endpoints; `GET /api/v{4,5}/alias/options` issues signed suffixes; `POST /api/v{2,3}/alias/custom/new` creates aliases.
- ✅ **Operational** — **Q1/Q6:** `VALID → 201` (serialized alias payload); `TAMPERED`, `GARBAGE`, and `EXPIRED` (700 s > 600 s) suffixes **all → 412** `{"error":"Alias creation time is expired, please retry"}` (the headline finding, reproduced live); wrong-domain valid signature → `400 {"error":"wrong alias prefix or suffix"}`; quota gate → `400` with `MAX_NB_EMAIL_FREE_PLAN` interpolated (=3 in `test.env`).
- ✅ **Operational** — **Q2:** `SL` logger writes to stdout; captured WARNING `Alias creation time expired for <User>` (412 path) and the ERROR line at `app/alias_suffix.py:61`, immediately followed by the literal `NoneType: None` (the D1 artifact).
- ✅ **Operational** — **Q3:** No `X-RateLimit-*` / `Retry-After` headers on either `201` or `429` over real HTTP; the 429 body is `{"error":"Rate limit exceeded"}`.
- ✅ **Operational** — **Q4/Q5:** `can_create_new_alias()` + `check_bucket_limit()` execute; on a successful under-quota creation the quota machinery logs **nothing** (confirmed live).

**UI verification**

- ✅ **Operational** — The web (dashboard) path (`app/dashboard/views/custom_alias.py`) was exercised via the 14 reference tests; it shares the same verifier and concurrency lock but responds with `flash()` messages + `redirect()` (HTTP 302) rather than JSON status codes. No new UI was introduced by this task.
- ⚠ **Partial (by design)** — No standalone front-end build/visual QA applies; this is a diagnostic-documentation deliverable, not a UI feature.

---

## 5. Compliance & Quality Review

AAP deliverables cross-mapped to quality/compliance benchmarks. Fixes applied during autonomous validation are noted.

| Benchmark / AAP Requirement | Status | Progress | Evidence / Notes |
|-----------------------------|--------|----------|------------------|
| Read-only mandate — source byte-for-byte unchanged | ✅ Pass | 100% | `git diff 2cd6ee77..HEAD` = 1 file, +850/-0; zero source files touched |
| Filename binding — `app_2cd6ee777f8c.md` (= branch name) | ✅ Pass | 100% | File present at exact name |
| Placement — `blitzy/documentation/` | ✅ Pass | 100% | Correct destination directory |
| Q1 failure responses answered | ✅ Pass | 100% | §2.1–§2.4 (verifier + v2/v3 rejection tables) |
| Q2 validation logs answered | ✅ Pass | 100% | §3.1–§3.3 (format + entry catalog) |
| Q3 rate-limit headers answered | ✅ Pass | 100% | §4.1 (absent, 3 proofs + live) |
| Q4 success-path quota checks answered | ✅ Pass | 100% | §5.1, §5.2, §5.4 |
| Q5 quota logging answered | ✅ Pass | 100% | §5.3 (nothing logged on success) |
| Q6 validating component & conditions answered | ✅ Pass | 100% | §1.3 + §2.3/§2.4 |
| Code-as-truth evidence (file:line citations) | ✅ Pass | 100% | Inline + Appendix B; spot-checks accurate; 31/31 anchors resolve |
| Rationale provided per answer | ✅ Pass | 100% | Each section includes reasoning, not bare values |
| Intermittency root-cause analysis | ✅ Pass | 100% | §6.1–§6.5 + diagnostic checklist |
| Live reproduction (all 3 mechanisms distinctly) | ✅ Pass | 100% | Appendix A; mechanisms ①②③ exercised |
| Markdown quality (fences/whitespace/headings/anchors) | ✅ Pass | 100% | 14 balanced fence pairs, 0 trailing WS, 43 headings, 31/31 anchors |
| D1 correction (`LOG.e` → `NoneType: None`) | ✅ Pass (fixed in validation) | 100% | Corrected across 4 occurrences (commit `f3600945`) |
| Findings reported, not fixed (F1–F4) | ✅ Pass | 100% | Documented as findings; remediation out of scope |
| SME review & sign-off | ⏳ Outstanding | 0% | Human path-to-production (1.5 h) |

**Fixes applied during autonomous validation:** D1 (`LOG.e` behavior corrected to the literal `NoneType: None` artifact across 4 occurrences); concurrency-lock scope corrected to current-user-or-IP; `app/parallel_limiter.py` Appendix B off-by-one citation corrected; citation ranges refined. **Outstanding:** human sign-off only.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| RK1 — Headline 412-vs-400 finding depends on the `itsdangerous` 1.1.0 exception hierarchy (`SignatureExpired`/`BadTimeSignature` ⊂ `BadSignature`) | Technical | Low | Low | Versions pinned (§0/Appendix C); finding tied to exact hierarchy; re-confirm via R2 | Mitigated |
| RK2 — Line-number citations may drift with future source refactors | Technical | Low | Medium | Citations include file paths + function/symbol names, not only lines; pinned to branch | Mitigated |
| RK3 — Production-code regression / compile / test breakage | Technical | None (info) | N/A | Zero production-code change; source byte-for-byte unchanged | Closed (by design) |
| RK4 — New attack surface / secret exposure from the deliverable | Security | Low | Low | Read-only; references config var **names** only (e.g. `CUSTOM_ALIAS_SECRET`), never values; no new endpoints/deps | Mitigated |
| RK5 — 412/400 conflation masks tamper-vs-expiry signals (latent in SimpleLogin) | Security | Low (info) | N/A | Surfaced as Finding F1 for the team; remediation out of scope | Open finding — reported, not fixed |
| RK6 — Redis absence silently no-ops bucket quota + concurrency lock → environment-dependent intermittency | Operational | Medium | Medium | Documented in §6.4 + diagnostic checklist §6.5; reported as finding | Open finding — reported, not fixed |
| RK7 — Reader misapplies the §6.5 checklist (misattributes a 429) | Operational | Low | Low | Three mechanisms explicitly separated with distinguishing evidence per symptom | Mitigated |
| RK8 — Reproduction without Redis fails to surface mechanisms ②③ (429) | Integration | Low | Medium | Prerequisites (PostgreSQL 13 + Redis 6) documented in Appendix A; §6.4 notes no-op behavior | Mitigated |

**Overall risk posture: LOW.** As a read-only diagnostic, the deliverable introduces no deployment, regression, or security risk to SimpleLogin. The only Medium-severity item (RK6) is a *SimpleLogin* finding reported for the team's awareness, not a defect of the deliverable. No High or Critical risks.

---

## 7. Visual Project Status

```mermaid
%%{init: {"theme":"base","themeVariables":{"pie1":"#5B39F3","pie2":"#FFFFFF","pieStrokeColor":"#B23AF2","pieStrokeWidth":"2px","pieOuterStrokeWidth":"2px","pieTitleTextColor":"#B23AF2","pieSectionTextColor":"#111111","pieLegendTextColor":"#111111"}}}%%
pie showData title Project Hours — Completed vs Remaining
    "Completed Work" : 29.5
    "Remaining Work" : 2.5
```

**Remaining hours by category (from §2.2):**

| Category | Hours | Priority |
|----------|-------|----------|
| Documentation review & sign-off (SME) | 1.5 | High |
| Independent live re-confirmation of headline finding | 1.0 | Medium |
| **Total** | **2.5** | — |

> **Integrity:** "Remaining Work" (2.5 h) equals Remaining Hours in §1.2 and the sum of the §2.2 Hours column. "Completed Work" (29.5 h) equals Completed Hours in §1.2. Colors: Completed = Dark Blue `#5B39F3`, Remaining = White `#FFFFFF`.

---

## 8. Summary & Recommendations

**Achievements.** The project is **92.2 % complete** (29.5 h of 32.0 h). The sole AAP deliverable — a comprehensive, evidence-cited diagnostic of SimpleLogin's custom-alias-creation flow — is fully authored, validated against code-as-truth, reproduced live, and committed, with the SimpleLogin source left byte-for-byte unchanged. All six questions are answered with rationale, the three limiting mechanisms are disentangled, and the headline root cause (the 412-vs-400 expired/tampered conflation) is identified and live-confirmed.

**Remaining gaps.** The outstanding 2.5 h is human path-to-production: SME review/sign-off (1.5 h) and an optional independent re-confirmation of the headline finding (1.0 h). No technical defects remain in the deliverable.

**Critical path to production.** Review → sign-off → (optional) independent re-confirmation → publish to the team runbook. There is no build/deploy pipeline for a documentation artifact.

**Success metrics.** All six questions answered ✓; 100 % of reproduction tests passing (27/27) ✓; source unchanged ✓; document lint-clean with 31/31 anchors resolving ✓; headline finding reproduced live ✓.

**Production-readiness assessment.** **Ready for human review.** As a read-only diagnostic, the deliverable is production-ready in content and form; formal acceptance awaits SME sign-off. Per Blitzy policy, completion is capped below 100 % until that human review occurs.

| Metric | Value |
|--------|-------|
| Completion | 92.2 % |
| Completed / Total hours | 29.5 / 32.0 |
| Remaining hours | 2.5 |
| Reproduction tests passing | 27 / 27 (100 %) |
| Source files modified | 0 |
| Open blocking issues | 0 |

---

## 9. Development Guide

This guide covers (A) accessing/verifying the deliverable in this repository, and (B) building/running SimpleLogin to reproduce the six answers. Commands in part A are tested in this environment. Commands in part B are the project's own canonical commands (from `scripts/run-test.sh` and `.github/workflows/main.yml`) targeting SimpleLogin's authoritative **Python 3.10** runtime.

### 9.1 System Prerequisites

- **Python 3.10** (SimpleLogin pins `python = "^3.10"`; Dockerfile `FROM python:3.10`).
- **Poetry** (dependency management; `poetry install`).
- **Docker** (to run PostgreSQL 13 and Redis 6 locally).
- **Node 10.17.0** (only for building static assets; not needed to reproduce the API behavior).
- App listens on **port 7777**.

### 9.2 Access & Verify the Deliverable (tested here)

```bash
# From the repository root:
cd /tmp/blitzy/app/blitzy-bcd3781c-568c-4126-88cb-3db9dfe77959_d55f84

# 1) Confirm the deliverable exists
ls -la blitzy/documentation/app_2cd6ee777f8c.md

# 2) Read it
sed -n '1,80p' blitzy/documentation/app_2cd6ee777f8c.md   # or open in your editor

# 3) Prove the SimpleLogin source is unchanged (expect ONLY the .md, status "A")
git diff --name-status 2cd6ee77..HEAD

# 4) Confirm a clean working tree
git status --porcelain    # (no output = clean)

# 5) Markdown integrity checks
grep -c -E '^[`]{3}' blitzy/documentation/app_2cd6ee777f8c.md   # 28 (even = balanced)
grep -c -E ' +$'     blitzy/documentation/app_2cd6ee777f8c.md   # 0 trailing-whitespace lines
grep -c -E '^#{1,6} ' blitzy/documentation/app_2cd6ee777f8c.md  # 43 headings
```

### 9.3 Environment Setup (SimpleLogin — to reproduce the answers)

```bash
# Start backing services (PostgreSQL 13 on 15432, Redis 6 on 6379)
docker run -d --name sl-test-db \
  -e POSTGRES_PASSWORD=test -e POSTGRES_USER=test -e POSTGRES_DB=test \
  -p 15432:5432 postgres:13
docker run -d --name sl-redis -p 6379:6379 redis:6
sleep 3
```

### 9.4 Dependency Installation

```bash
# Install Python dependencies with Poetry (Python 3.10)
poetry install --no-interaction --no-ansi --no-root
```

### 9.5 Database Migration & Startup

```bash
# Apply migrations against the test config
CONFIG=tests/test.env poetry run alembic upgrade head

# Run the app (serves on http://localhost:7777)
CONFIG=tests/test.env poetry run python server.py
# (Alternatively, a WSGI server: CONFIG=tests/test.env poetry run gunicorn wsgi:app -b 0.0.0.0:7777)
```

### 9.6 Verification — Run the In-Scope Reproduction Tests

```bash
# In-scope API reproduction suite (expect 13 passed, deterministically)
CONFIG=tests/test.env poetry run pytest \
  tests/api/test_new_custom_alias.py tests/api/test_alias_options.py \
  -p no:randomly -o addopts="" --timeout=120

# Full CI suite (optional)
poetry run pytest -c pytest.ci.ini
```

### 9.7 Example Usage — Reproduce the Six Answers

Authentication uses the **`Authentication`** HTTP header carrying an API-key code (`app/api/base.py:L17`).

```bash
# (1) Fetch a fresh signed_suffix
curl -s -H "Authentication: $API_KEY" http://localhost:7777/api/v4/alias/options | python -m json.tool
#   -> use a value from the "suffixes" field (the signed_suffix)

# (2) Q1/Q6 — SUCCESS path (201)
curl -s -o /dev/null -w "%{http_code}\n" -X POST \
  -H "Authentication: $API_KEY" -H "Content-Type: application/json" \
  -d '{"alias_prefix":"myalias","signed_suffix":"<FRESH_SIGNED_SUFFIX>"}' \
  http://localhost:7777/api/v2/alias/custom/new            # -> 201

# (3) Q1/Q6 — HEADLINE FINDING: a corrupted suffix returns 412 (not 400)
curl -s -X POST \
  -H "Authentication: $API_KEY" -H "Content-Type: application/json" \
  -d '{"alias_prefix":"myalias","signed_suffix":"TOTALLY-CORRUPTED"}' \
  http://localhost:7777/api/v2/alias/custom/new
#   -> 412 {"error":"Alias creation time is expired, please retry"}

# (4) Q3 — inspect headers (no X-RateLimit-* / Retry-After)
curl -sI -X POST -H "Authentication: $API_KEY" -H "Content-Type: application/json" \
  -d '{}' http://localhost:7777/api/v2/alias/custom/new
```

To forge a known suffix inside a throwaway Python shell (as the tests do):

```python
from app.alias_suffix import signer
signed = signer.sign(".word@domain").decode()    # fresh, valid signed_suffix
```

### 9.8 Troubleshooting

- **`error: externally-managed-environment` (pip):** use Poetry (the project's tool) or a virtualenv; do not `pip install` into the system Python.
- **`412` for a seemingly fresh suffix:** the suffix may be > 600 s old **or** corrupted — both return 412 (the conflation). Re-fetch a fresh `signed_suffix`.
- **`429` not reproducing:** ensure **Redis is running** and `DISABLE_RATE_LIMIT` is **unset**; without Redis the bucket quota and concurrency lock silently no-op (§6.4 of the deliverable).
- **Tests flaky on a shared DB:** run on a pristine DB with `-p no:randomly`; a 409 "already exists" is *correct* documented behavior caused by data pollution, not a defect.
- **Cleanup:** `docker rm -f sl-test-db sl-redis`.

---

## 10. Appendices

### Appendix A — Command Reference

| Purpose | Command |
|---------|---------|
| Confirm deliverable | `ls -la blitzy/documentation/app_2cd6ee777f8c.md` |
| Prove source unchanged | `git diff --name-status 2cd6ee77..HEAD` |
| Clean working tree | `git status --porcelain` |
| Start PostgreSQL | `docker run -d --name sl-test-db -e POSTGRES_PASSWORD=test -e POSTGRES_USER=test -e POSTGRES_DB=test -p 15432:5432 postgres:13` |
| Start Redis | `docker run -d --name sl-redis -p 6379:6379 redis:6` |
| Install deps | `poetry install --no-interaction --no-ansi --no-root` |
| Migrate DB | `CONFIG=tests/test.env poetry run alembic upgrade head` |
| Run app | `CONFIG=tests/test.env poetry run python server.py` |
| In-scope tests | `CONFIG=tests/test.env poetry run pytest tests/api/test_new_custom_alias.py tests/api/test_alias_options.py -p no:randomly` |
| Cleanup | `docker rm -f sl-test-db sl-redis` |

### Appendix B — Port Reference

| Service | Port | Notes |
|---------|------|-------|
| SimpleLogin web/API | 7777 | `EXPOSE 7777`; `server.py` |
| PostgreSQL (test) | 15432 → 5432 | `DB_URI=postgresql://test:test@localhost:15432/test` |
| Redis | 6379 | Backs session store, bucket quota, concurrency lock |

### Appendix C — Key File Locations

| Path | Role |
|------|------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The deliverable** (850 lines) |
| `app/alias_suffix.py` | Signed-suffix component: `check_suffix_signature` (max_age=600), `verify_prefix_suffix` |
| `app/api/views/new_custom_alias.py` | v2/v3 creation endpoints; all rejection branches |
| `app/api/views/alias_options.py` | Signed-suffix issuance (`/api/v4,v5/alias/options`) |
| `app/extensions.py` | Flask-Limiter construction (no `headers_enabled`) |
| `app/rate_limiter.py` | Bucket quota `check_bucket_limit()` → 429 |
| `app/parallel_limiter.py` | Per-user/IP concurrency lock → 429 |
| `app/models.py` | `can_create_new_alias()`, `Alias.create()` |
| `app/config.py` | `ALIAS_LIMIT`, `MAX_NB_EMAIL_FREE_PLAN`, `DISABLE_RATE_LIMIT` |
| `server.py` | Global 429 handler; `limiter.init_app()` |
| `app/log.py` | `SL` logger; `LOG.d/i/w/e` shortcuts |

### Appendix D — Technology Versions

| Package | Version | Relevance |
|---------|---------|-----------|
| Python | 3.10 | SimpleLogin runtime (Dockerfile/pyproject) |
| flask | 1.1.2 | `jsonify` responses; the `api` blueprint |
| flask-limiter | 1.4 | HTTP route limiting; **headers off by default** |
| itsdangerous | 1.1.0 | `TimestampSigner`; `SignatureExpired`/`BadTimeSignature` ⊂ `BadSignature` |
| werkzeug | 1.0.1 | `TooManyRequests` (429) |
| redis | 4.6.0 | Session store, bucket quota, concurrency lock |
| limits | 1.5.1 | `RedisStorage` abstraction |
| sqlalchemy | 1.3.24 | ORM persisting `Alias` rows |
| newrelic | 8.8.0 | `BucketRateLimit` event on bucket breach |
| gunicorn | 20.0.4 | WSGI server used during live validation |

### Appendix E — Environment Variable Reference

| Variable | Value (test) | Purpose |
|----------|--------------|---------|
| `CONFIG` | `tests/test.env` | Selects the runtime config file |
| `DB_URI` | `postgresql://test:test@localhost:15432/test` | PostgreSQL connection |
| `MAX_NB_EMAIL_FREE_PLAN` | `3` | Free-plan alias cap (interpolated into the quota-gate 400 message) |
| `DISABLE_RATE_LIMIT` | _unset_ | When set, disables the Flask-Limiter HTTP limit |
| `CUSTOM_ALIAS_SECRET` | (secret) | Key for the suffix `TimestampSigner` (referenced by name only) |
| `API_KEY` | (your key) | Value for the `Authentication` request header |

### Appendix F — Developer Tools Guide

- **git** — `git diff --name-status 2cd6ee77..HEAD` confirms only the documentation file changed; `git log --author=agent@blitzy.com --oneline` lists the 4 documentation commits.
- **pytest** — run the in-scope reproduction suite with `-p no:randomly` for determinism; use `--timeout=120` to guard hangs.
- **curl** — use `-sI` to inspect response headers (verifying the absence of rate-limit headers) and `-w "%{http_code}"` to print status codes.
- **Poetry** — `poetry install`, `poetry run <cmd>`; the project sets `virtualenvs.create false` in Docker but a local venv works equally.
- **Docker** — single-container PostgreSQL/Redis for local reproduction; remember to `docker rm -f` afterward.

### Appendix G — Glossary

| Term | Meaning |
|------|---------|
| Signed suffix | A domain suffix signed by `itsdangerous.TimestampSigner`; valid for 600 s (`max_age`) |
| 600 s window | Time-to-live of a signed suffix; starts at issuance in the alias-options endpoint |
| Bucket quota | `check_bucket_limit()` inside `Alias.create()`; wall-clock-aligned Redis `INCR` windows (Free `10,900:50,3600`) |
| Concurrency lock | `@parallel_limiter.lock("alias_creation")`; Redis `SET NX`, 5 s TTL; per current-user-or-IP |
| Flask-Limiter limit | `@limiter.limit(ALIAS_LIMIT)`; default `100/day;50/hour;5/minute` |
| 412 conflation | Expired **and** tampered suffixes both return `412 "...expired, please retry"` (Finding F1 / headline) |
| `LOG.e` artifact | `LOG.e == logging.Logger.exception`; outside an `except` block it appends the literal `NoneType: None` (Finding F5 / D1) |
| Finding (F1–F6) | A behavior reported in the deliverable but **not** fixed (read-only mandate) |