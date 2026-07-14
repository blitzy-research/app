# Blitzy Project Guide

## SimpleLogin Custom-Alias Signed-Suffix Validation & Limit-Enforcement — Runtime-Grounded Investigation

> **Deliverable type:** Documentation / Q&A code-investigation (isolated, read-only)
> **Sole artifact:** `blitzy/documentation/app_2cd6ee777f8c.md` (6,216 lines)
> **Brand color legend:** <span style="color:#5B39F3">■ Completed / AI Work — Dark Blue `#5B39F3`</span> · <span>□ Remaining / Not Completed — White `#FFFFFF`</span>

---

## 1. Executive Summary

### 1.1 Project Overview

This engagement is a runtime-grounded code investigation of **SimpleLogin**, an open-source Flask email-alias service. A user debugging *intermittent custom-alias validation failures* asked five evidence-seeking questions about how signed suffixes are verified and how alias-creation limits are enforced. The deliverable is a single investigative document — not a code change — that traces `POST /api/v2|v3/alias/custom/new` end-to-end and reports, with actual observed output, the exact HTTP status codes, error messages, server-console logs, rate-limiting headers, and quota checks for invalid, expired, and successful submissions. The source repository is treated as strictly read-only. Target audience: the requesting engineer and SimpleLogin maintainers who must decide whether to remediate the discovered validation behavior.

### 1.2 Completion Status

**Completion methodology (PA1):** The percentage measures autonomous work delivered against the Agent Action Plan (AAP) scope plus path-to-production, expressed in engineering hours. All five explicit requirements and all implicit requirements are complete; the only remaining work is inherently-human path-to-production (SME review + sign-off/merge).

**Formula:** Completed Hours ÷ Total Hours × 100 = 34 ÷ 37 × 100 = **91.9%**

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieOuterStrokeWidth':'2px','pieTitleTextSize':'16px','pieSectionTextColor':'#B23AF2'}}}%%
pie showData title Completion — 91.9% (34h of 37h)
    "Completed Work" : 34
    "Remaining Work" : 3
```

| Metric | Hours |
|--------|-------|
| **Total Hours** | 37 |
| **Completed Hours (AI + Manual)** | 34 |
| &nbsp;&nbsp;• AI (Blitzy autonomous) | 34 |
| &nbsp;&nbsp;• Manual (human, to date) | 0 |
| **Remaining Hours** | 3 |
| **Percent Complete** | **91.9%** |

### 1.3 Key Accomplishments

- ✅ Single deliverable `blitzy/documentation/app_2cd6ee777f8c.md` created (6,216 lines) answering all five questions with runtime-observed evidence.
- ✅ **Lead root-cause finding proven:** tampered + expired + garbage signed suffixes all collapse to **HTTP 412** because `check_suffix_signature` catches the superclass `itsdangerous.BadSignature` (`app/alias_suffix.py:37-42`); the `"Tampered suffix" 400` branch is effectively unreachable for string input.
- ✅ Real endpoint exercised end-to-end through two canonical runners: the pytest `flask_client` harness **and** a live `gunicorn wsgi:app` server on `:7777`.
- ✅ 24-row condition matrix + 28-row evidence matrix — every claim tied to a probe/command and its complete, unedited output.
- ✅ Rate-limiting answered with limiting **enabled** (no `X-RateLimit-*` headers; first `429` at request #6), and web-search-grounded to `flask-limiter 1.4` (the doc even corrects the AAP's "1.5" version assumption).
- ✅ Read-only constraint honored — source tree byte-for-byte unchanged; all temporary probes removed; PostgreSQL and Redis restored to baseline (net-zero).

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| Underlying SimpleLogin product behavior (412-collapse; `"Tampered suffix" 400` unreachable) is **reported, not fixed** (out of AAP scope) | The user's intermittent-validation-failure symptom persists until a maintainer acts | SimpleLogin maintainer / requesting engineer | Optional follow-up (~2–4h, see §8) |
| `v2` endpoint has **no** `alias_prefix` validation gate → malformed prefix surfaces as `HTTP 500` (v3 returns a clean 400) | Informational; disclosed for maintainer awareness (no data leak — generic body) | SimpleLogin maintainer | Optional follow-up (~1–2h) |
| No content-level blocker for the deliverable itself | Document is complete and internally consistent | — | — |

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-----------------|----------------|-------------------|-------------------|-------|
| SimpleLogin repository | Git write | Branch `blitzy-692fd7bf-...` accessible; single doc committed; working tree clean | ✅ Resolved | Blitzy Agent |
| Prescribed container image | Runtime (PostgreSQL+pg_trgm, Redis, Poetry, Python 3.10) | Investigation ran inside the prescribed image; the Project-Guide host lacks these services (expected) | ✅ Resolved (in-container) | Blitzy Agent |
| External services / third-party APIs | — | None required for this investigation | ✅ N/A | — |

No blocking access issues identified. Runtime observations were captured inside the prescribed container image; this Project-Guide host intentionally does not re-run them (test/runtime results below originate from Blitzy's autonomous validation logs).

### 1.6 Recommended Next Steps

1. **[High]** SME technical review of `blitzy/documentation/app_2cd6ee777f8c.md` — verify the five answers, spot-check `file:line` citations and captured-output transcripts, and confirm the lead 412-collapse finding (**2h**, in scope).
2. **[Medium]** Stakeholder sign-off and merge of the single-file documentation PR to the destination branch (**1h**, in scope).
3. **[Medium]** *(Optional, out of AAP scope — addresses the user's underlying problem)* File an upstream fix so `check_suffix_signature` distinguishes `SignatureExpired` (→412 expired) from `BadSignature` (→400 tampered) (~2–4h).
4. **[Low]** *(Optional, out of scope)* Add a `check_alias_prefix` gate to the `v2` endpoint so malformed prefixes return a clean `400` instead of a `500` (~1–2h).
5. **[Low]** *(Optional, out of scope)* Add regression tests for tampered/expired suffixes and fix the pre-existing `test_minimal_payload` fixture-isolation flake (SAVEPOINT/nested transaction) (~2–3h).

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

Every component traces to a specific AAP requirement (R1–R5), an implicit requirement (I1–I7), or a prerequisite. **Total = 34h.**

| Component | Hours | Description |
|-----------|-------|-------------|
| Runtime environment provisioning & app boot | 4 | Provision PostgreSQL + `pg_trgm`, Redis, Poetry dependencies (Python 3.10); boot via pytest `flask_client` harness and live `gunicorn wsgi:app` [prereqs / I1] |
| Q1 — Invalid/expired/garbage failure responses | 3 | Exercise real endpoints; capture exact HTTP status + body (412; and 400/409 boundaries); live `curl -i` [R1] |
| Q2 — Server-console log capture | 3 | Capture `"SL"` logger output per condition; per-emitter `file:line` table (`LOG.w`/`LOG.d`/`after_request`) [R2] |
| Q3 — Rate-limiting headers | 4 | Live gunicorn with limiting enabled; observe 429 boundary across two runs; web-search + `flask-limiter 1.4` header-default analysis [R3, I5] |
| Q4 — Success-path quota checks | 3 | Trace `can_create_new_alias()` + `max_alias_for_free_account()`; all six quota branches incl. trial-still-capped; confirm nothing logged on 201 [R4] |
| Q5 — Execution-path trace | 4 | Decorator chain, in-handler branch order (v2/v3), four limit enforcers, Mermaid flow diagrams [R5] |
| Root-cause analysis | 5 | `itsdangerous 1.1.0` MRO runtime proof; verified tamper construction; non-string 500s; v2/v3 split; 40× determinism [I3] |
| Boundary & edge conditions | 3 | Empty body, duplicate, in-trash, two-consecutive-dots, custom-domain, prefix, unicode [I2] |
| Document authoring & evidence matrices | 3 | 6,216-line document: executive summary, 24-row condition matrix, 28-row evidence matrix, appendices, methodology, secondary paths [I6 + writing] |
| Net-zero cleanup & read-only verification | 2 | Remove temporary probes; restore PostgreSQL/Redis to baseline; git-verify source unchanged [I7] |
| **Total Completed** | **34** | **Matches Completed Hours in §1.2** |

### 2.2 Remaining Work Detail

Each category is inherently-human path-to-production for a documentation deliverable. **Total = 3h.**

| Category | Hours | Priority |
|----------|-------|----------|
| SME technical review of the answer document (accuracy, citations, transcripts, lead finding) | 2 | High |
| Stakeholder sign-off & PR merge to destination branch | 1 | Medium |
| **Total Remaining** | **3** | **Matches Remaining Hours in §1.2 and §7** |

> **Out of scope (excluded from the completion denominator):** Fixing SimpleLogin's discovered behavior (412-collapse; v2 malformed-prefix 500; unreachable "Tampered suffix" 400) is explicitly *report-not-remediate* per AAP §0.5.2. Optional upstream-fix effort (~5–9h across OPT-A/B/C) is listed in §1.6 and §8 as recommendations only and is **not** counted in the 3h remaining.

### 2.3 Hours Summary

| Bucket | Hours | Share |
|--------|-------|-------|
| Completed (AI) | 34 | 91.9% |
| Remaining (Human) | 3 | 8.1% |
| **Total Project** | **37** | **100%** |

**Cross-section check:** §2.1 (34) + §2.2 (3) = 37 = §1.2 Total ✓ · Remaining 3h identical in §1.2, §2.2, §7 ✓.

---

## 3. Test Results

For a documentation/Q&A deliverable there is no first-party application code to unit-test; "testing" means **reproducing every documented runtime claim** through the real endpoint. All results below originate from **Blitzy's autonomous validation logs** (pytest `flask_client` harness + live `gunicorn` server inside the prescribed container). No results are fabricated on the Project-Guide host.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|------------|-------|
| Shipped endpoint module suite | pytest | 10 | 9 | 1 | n/a | `tests/api/test_new_custom_alias.py` run against **read-only** source. The single failure (`test_minimal_payload`) fails only at **module scope** and **passes in isolation** — a pre-existing fixture-isolation flake (no SAVEPOINT; app-level commits leak past `transaction.rollback()`), out of scope and not introduced by this deliverable |
| Runtime condition reproductions | Werkzeug test client (`server.create_app()`) | 24 | 24 | 0 | n/a | The §8 condition matrix — success (201), tampered/expired/garbage (412), empty (400), duplicate (409), two-dots (400), wrong prefix/suffix/domain (400), quota (400), non-string suffix (500), rate-limit/lock/bucket (429). Every row reproduced with captured output |
| Over-the-wire evidence | Live `gunicorn wsgi:app` + `curl -i` | 3 | 3 | 0 | n/a | Raw transport headers for the 201 / 412 / 429 responses; confirms **no** `X-RateLimit-*`/`Retry-After` headers appear |
| Rate-limit boundary stability | Werkzeug test client | 2 | 2 | 0 | n/a | Two isolated runs both `[201×5, 429, 429]`; first 429 at request #6; `RUN1 == RUN2` |
| Determinism check | Werkzeug test client | 40 | 40 | 0 | n/a | 40 identical tampered/expired requests → `{412: 40}`, zero variance (§9.7) |

**Aggregate:** 79 discrete autonomous checks executed; 78 passed; 1 pre-existing out-of-scope module-scope flake (passes in isolation). Frameworks: **pytest**, **Werkzeug test client**, **live gunicorn + curl**. No code coverage metric applies to a documentation deliverable.

---

## 4. Runtime Validation & UI Verification

**Runtime health** (captured inside the prescribed container):

- ✅ **Operational** — App boots via the pytest `flask_client` fixture (`server.create_app()`).
- ✅ **Operational** — App boots as a live `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` server (Dockerfile CMD).
- ✅ **Operational** — PostgreSQL 15 accepting connections on `:5432`; `pg_trgm` present; schema at Alembic head `32f25cbf12f6` (77 tables).
- ✅ **Operational** — Redis reachable (`redis-cli ping → PONG`); used by Flask-Limiter and the parallel-limiter lock.
- ✅ **Operational** — Pinned libraries confirmed at runtime: `itsdangerous 1.1.0`, `flask 1.1.2`, `flask-limiter 1.4`, `werkzeug 1.0.1`, `redis 4.6.0`, `sqlalchemy 1.3.24`, Python `3.10.18`.

**API integration outcomes** (`POST /api/v2|v3/alias/custom/new`, real API-key auth):

- ✅ **Operational** — Valid signed suffix under quota → `201` with serialized alias body.
- ✅ **Operational** — Tampered / expired / garbage suffix → `412` `{"error":"Alias creation time is expired, please retry"}` (Content-Length 57).
- ✅ **Operational** — Boundary responses: empty `400`, duplicate `409`, two-dots `400`, wrong prefix/suffix `400`, quota `400`, rate-limit/lock/bucket `429`.
- ✅ **Operational** — Canonical `signed_suffix` minting via `GET /api/v4|v5/alias/options` (v4 pairs vs v5 objects) and the shared `signer`.
- ⚠ **Partial (informational)** — `v2` malformed `alias_prefix` surfaces as `500 {"error":"Internal error"}` (no `check_alias_prefix` gate); v3 returns a clean `400`. Documented, not remediated.

**UI verification:** **N/A for this scope.** The deliverable is a Markdown document, and the investigated surfaces are JSON API endpoints with no frontend component. No browser/UI verification, Figma comparison, or Design-System compliance applies (AAP §0.9). The secondary web-UI (dashboard) path is *documented* (`flash()` vs JSON) in §10.1 of the deliverable but was not a UI test target.

---

## 5. Compliance & Quality Review

Cross-mapping of AAP deliverables and governing-rule mandates to observed quality benchmarks. All items validated against the deliverable and the read-only source.

| Benchmark / Requirement | Source | Status | Progress |
|--------------------------|--------|--------|----------|
| R1 — Invalid/expired failure status + messages | AAP §0.1.1 | ✅ Pass | 100% |
| R2 — Server-console log entries | AAP §0.1.1 | ✅ Pass | 100% |
| R3 — Rate-limiting headers presence/values | AAP §0.1.1 | ✅ Pass | 100% |
| R4 — Success-path quota checks + logged values | AAP §0.1.1 | ✅ Pass | 100% |
| R5 — Execution-path trace + rejection conditions | AAP §0.1.1 | ✅ Pass | 100% |
| Run-first, observe-then-write methodology | Rule set (SWE-AtlasQnA-Repo) | ✅ Pass | 100% |
| Canonical entry point (real endpoint, no mocks/hand-forged tokens) | Rules | ✅ Pass | 100% |
| Every condition exercised (not just happy path) | Rules | ✅ Pass | 100% (24-row matrix) |
| Actual output included for every claim (no paraphrase, no `// ...` elision) | Rules | ✅ Pass | 176 balanced code fences; 0 elisions |
| Web-search validation of Flask-Limiter 1.4 header default | AAP §0.2.2 | ✅ Pass | Corrected to v0.7 / Issue 22 |
| Read-only source (byte-for-byte unchanged) | AAP §0.5.2 / Rules | ✅ Pass | Git-verified |
| Temporary probes removed; DB/Redis net-zero | AAP §0.7 | ✅ Pass | Restored to baseline |
| Deliverable location/name (`blitzy/documentation/app_2cd6ee777f8c.md`) | AAP §0.8 | ✅ Pass | Correct path/name |
| Zero placeholders / TODO / FIXME | Blitzy quality | ✅ Pass | 0 markers |
| Grounding — every claim carries `file:line` or observed output | Rules | ✅ Pass | 28-row evidence matrix |

**Fixes applied during autonomous validation:** two §12.2 evidence-matrix citation corrections were committed as `ae263df7` — decorator-order ranges corrected to span the three named decorators (`v2 L29-31` / `v3 L116-118`), and the Redis-absent no-op citation corrected to `parallel_limiter.py:51-52`. During earlier commits the investigation was remediated with runtime-grounded evidence and eight QA findings were resolved.

**Outstanding compliance items:** none for the deliverable. The two product-behavior findings are compliant-by-design as *reported-not-remediated* per AAP scope.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Documented behavior is version-specific (`itsdangerous 1.1.0`, `flask-limiter 1.4`, Python 3.10) | Technical | Low | Low | Versions pinned and printed at runtime (deliverable §2.1) | Mitigated |
| Underlying product bugs (412-collapse; unreachable "Tampered suffix" 400) remain unfixed by design | Technical | Medium | High | Doc leads with root cause + exact `file:line`; upstream-fix recommended (§1.6, §8) | Open (out of scope) |
| `v2` malformed-prefix `500` (no `check_alias_prefix` gate) | Technical | Low | Medium | Disclosed for maintainer awareness; generic body (no detail leaked) | Open (informational) |
| Pre-existing `test_minimal_payload` module-scope flake | Technical | Low | Low | Diagnosed in validator logs; not introduced by deliverable | Documented |
| Test-data hygiene (signed suffixes, keys, cookies) | Security | Low | Low | Synthetic/disposable suffixes; public test-only key; API keys withheld (0600 file); cookies redacted (§12.4) | Mitigated |
| Informational disclosure of latent endpoint behavior | Security | Low | Low | No product code changed → no new attack surface; generic error body | Open (informational) |
| Reproduction requires provisioning PostgreSQL+`pg_trgm`, Redis, Poetry, Python 3.10 | Operational | Low-Medium | Medium | Clean-start order (§2.2) + Development Guide (§9); prescribed container referenced | Mitigated |
| Temporary probes removed → re-run requires recreating them | Operational | Low | Low | All probe sources embedded verbatim in deliverable §11 | Mitigated |
| Limiter behavior depends on Redis presence (absent → lock + bucket become no-ops) | Integration | Low | Low | Redis-absent no-op paths documented (§7.7, §7.8) | Mitigated |
| Rate-limit-header conclusion specific to `flask-limiter 1.4` | Integration | Low | Low | Version pinned + web-search grounded (§5.1) | Mitigated |

**Overall risk posture:** Low. Because no product code was written or changed, the deliverable introduces **no new runtime, security, or integration risk** to SimpleLogin. The single Medium item is the *unfixed underlying product behavior*, which is out of scope by directive and surfaced as the top human action item.

---

## 7. Visual Project Status

**Project Hours Breakdown** — Completed = Dark Blue `#5B39F3`, Remaining = White `#FFFFFF`.

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieOuterStrokeWidth':'2px','pieSectionTextColor':'#B23AF2','pieTitleTextSize':'16px'}}}%%
pie showData title Project Hours Breakdown (Total 37h)
    "Completed Work" : 34
    "Remaining Work" : 3
```

**Remaining Hours by Category** (from §2.2 — sums to 3h):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#A8FDD9','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieOuterStrokeWidth':'2px','pieSectionTextColor':'#B23AF2','pieTitleTextSize':'16px'}}}%%
pie showData title Remaining Work by Category (3h)
    "SME Technical Review (High)" : 2
    "Sign-off & Merge (Medium)" : 1
```

**AAP Requirement Completion** (12 of 12 scoped requirements complete; only human path-to-production remains):

| Requirement group | Items | Complete | Status |
|-------------------|-------|----------|--------|
| Explicit (R1–R5) | 5 | 5 | ✅ 100% |
| Implicit (I1–I7) | 7 | 7 | ✅ 100% |
| Path-to-production (P1–P2) | 2 | 0 | ⏳ Human |

**Integrity note:** "Remaining Work" = **3h**, identical to §1.2 metrics and the §2.2 category sum.

---

## 8. Summary & Recommendations

**Achievements.** The engagement delivered a comprehensive, runtime-grounded answer document (6,216 lines) that resolves all five of the user's questions with observed evidence, a 24-row condition matrix, and a 28-row evidence matrix. It leads with the most probable explanation of the user's "intermittent validation failures": on the custom-alias endpoints a **tampered**, **expired**, and **garbage** signed suffix are indistinguishable — all return **HTTP 412** — because `check_suffix_signature` (`app/alias_suffix.py:37-42`) catches the superclass `itsdangerous.BadSignature`, and `SignatureExpired`/`BadTimeSignature` subclass it in `itsdangerous 1.1.0`. This was proven at runtime, including a 40× determinism check, and the source tree was left byte-for-byte unchanged.

**Remaining gaps.** None within AAP scope. The remaining **3h** is inherently-human path-to-production: an SME technical review of the document (**2h**) and stakeholder sign-off/merge (**1h**).

**Critical path to production.** (1) SME reviews the document and confirms the five answers and the lead finding → (2) stakeholder signs off and merges the single-file PR. That completes the deliverable's lifecycle.

**Beyond scope (recommended).** To actually resolve the user's underlying symptom, a maintainer should apply the upstream fix that distinguishes `SignatureExpired` from `BadSignature` (~2–4h), optionally add a `v2` prefix-validation gate (~1–2h) and regression tests (~2–3h). These are **not** part of the 37h project total (report-not-remediate directive).

**Success metrics.**

| Metric | Result |
|--------|--------|
| AAP requirements delivered | 12 / 12 scoped (R1–R5, I1–I7) |
| Questions answered with observed evidence | 5 / 5 |
| Autonomous validation checks passed | 78 / 79 (1 pre-existing out-of-scope flake) |
| Source files modified | 0 (read-only honored) |
| Placeholders / TODO / elisions | 0 |
| **AAP-scoped completion** | **91.9%** |

**Production-readiness assessment.** The deliverable is **production-ready**: complete, internally consistent, runtime-grounded, and read-only-compliant, pending the standard human review-and-merge gate. At **91.9% complete**, the residual is entirely human acceptance work.

---

## 9. Development Guide

This guide reproduces the investigation and verifies the deliverable. Steps are grounded in the Dockerfile, `tests/conftest.py`, `tests/test.env`, and the deliverable's live-captured §2. Host-runnable verification commands were tested; container-dependent steps are labeled **[container]**.

### 9.1 System Prerequisites

- **Prescribed container image** (canonical runtime): `andrewparkscaleai/coding-agent:simple-login__app__2cd6ee777f8c2d3531559588bcfb18627ffb5d2c` (from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`).
- **Python 3.10** (container ships 3.10.18).
- **PostgreSQL 15** with the `pg_trgm` extension.
- **Redis** (Flask-Limiter store + parallel-limiter lock).
- **Poetry** (`virtualenvs.create false`).
- **Pinned libraries:** `itsdangerous 1.1.0`, `flask 1.1.2`, `flask-limiter 1.4`, `werkzeug 1.0.1`, `redis 4.6.0`, `sqlalchemy 1.3.24`.

### 9.2 Environment Setup **[container]**

```bash
# 1. Start the two backing services (idempotent), in this order.
service postgresql start      # PostgreSQL 15 on localhost:5432
service redis-server start    # Redis on localhost:6379

# 2. Confirm readiness BEFORE launching the app (do not race the boot).
pg_isready -h localhost -p 5432 -U test    # -> localhost:5432 - accepting connections
redis-cli ping                             # -> PONG

# 3. Confirm the pg_trgm extension and migrated schema.
PGPASSWORD=test psql -h localhost -U test -d test -tAc \
  "SELECT extname FROM pg_extension WHERE extname='pg_trgm'"   # -> pg_trgm
PGPASSWORD=test psql -h localhost -U test -d test -tAc \
  "SELECT version_num FROM alembic_version"                    # -> 32f25cbf12f6
```

The shipped image is already migrated to Alembic head `32f25cbf12f6` (77 tables). If provisioning from scratch, the idempotent `/build.sh` performs the migration and extension creation (re-running against an already-migrated DB is a no-op).

**Configuration environment variables** (see §10-E for the full reference): `CONFIG=tests/test.env`, `DB_URI=postgresql://test:test@localhost:5432/test` (runtime port **5432** overrides the `15432` written in `tests/test.env:17`), `GNUPGHOME=/tmp/sl_gnupg` (must hold the shipped PGP keys, or `create_app()` fails at import), `PYTHONPATH=/app`.

### 9.3 Dependency Installation **[container]**

```bash
# As the Dockerfile does (line 32); or use the prebuilt /app/venv.
cd /app && poetry install --no-interaction --no-ansi --no-root

# Verify the pinned versions match the shipping application.
/app/venv/bin/python - <<'PY'
import sys, itsdangerous, flask, flask_limiter, werkzeug, redis, sqlalchemy
for n, v in [("python", sys.version.split()[0]), ("itsdangerous", itsdangerous.__version__),
             ("flask", flask.__version__), ("flask_limiter", flask_limiter.__version__),
             ("werkzeug", werkzeug.__version__), ("redis", redis.__version__),
             ("sqlalchemy", sqlalchemy.__version__)]:
    print(f"{n:<14} {v}")
PY
```

### 9.4 Application Startup — Two Canonical Runners **[container]**

```bash
# Runner A — fast probe harness (rolls back; commits nothing; DISABLE_RATE_LIMIT=True).
# Use for status/body/log observations of every condition EXCEPT rate-limit headers.
cd /app && env -i PATH=/usr/local/bin:/usr/bin:/bin HOME=/root \
  CONFIG=tests/test.env DB_URI='postgresql://test:test@localhost:5432/test' \
  GNUPGHOME=/tmp/sl_gnupg PYTHONPATH=/app \
  /app/venv/bin/python /tmp/sl_probes/probe_functional.py
# Probes were removed for net-zero — recreate them verbatim from deliverable §11.

# Runner B — live gunicorn server (genuine over-the-wire headers; rate limiting ENABLED
# by leaving DISABLE_RATE_LIMIT unset — config.py:602 reads the flag by presence).
gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
```

### 9.5 Verification Steps

**Host-runnable (tested — read-only / deliverable integrity):**

```bash
cd <repo-root>

# Only the single documentation file was added; source is unchanged.
git diff 2cd6ee77..HEAD --name-status
#   -> A  blitzy/documentation/app_2cd6ee777f8c.md

git diff 2cd6ee77..HEAD -- . ':(exclude)blitzy/documentation/*' | wc -l
#   -> 0   (source byte-for-byte baseline)

wc -l blitzy/documentation/app_2cd6ee777f8c.md      # -> 6216
grep -c '^```' blitzy/documentation/app_2cd6ee777f8c.md   # -> 176 (even = balanced)
```

**Container-runnable (endpoint behavior):**

- Mint a valid `signed_suffix` (via `GET /api/v4/alias/options` or `app.alias_suffix.signer.sign(...).decode()`), then `POST /api/v2/alias/custom/new`:
  - valid + under quota → `201` serialized alias;
  - tampered / expired / garbage → `412 {"error":"Alias creation time is expired, please retry"}`;
  - repeat 6× within a minute → `429 {"error":"Rate limit exceeded"}` (first 429 at request #6).
- `curl -i` against the live gunicorn server → confirm **no** `X-RateLimit-*` / `Retry-After` headers on any response.

### 9.6 Example Usage

```bash
# Against the live gunicorn server (Runner B), with a real API key:
curl -i -X POST http://localhost:7777/api/v2/alias/custom/new \
  -H "Authentication: <API_KEY>" -H "Content-Type: application/json" \
  -d '{"alias_prefix":"demo","signed_suffix":"<VALID_SIGNED_SUFFIX>"}'
#   -> HTTP/1.1 201 CREATED   (no X-RateLimit-* headers present)

# Tampered suffix -> 412 (identical to expired and garbage):
curl -s -o /dev/null -w '%{http_code}\n' -X POST \
  http://localhost:7777/api/v2/alias/custom/new \
  -H "Authentication: <API_KEY>" -H "Content-Type: application/json" \
  -d '{"alias_prefix":"demo","signed_suffix":"<TAMPERED>"}'
#   -> 412
```

### 9.7 Troubleshooting

- **`create_app()` fails at import** → `GNUPGHOME` must point to a clean directory holding the shipped PGP keys.
- **Boot race / connection refused** → confirm `pg_isready` and `redis-cli ping → PONG` *before* launching the app.
- **Port confusion (`15432` vs `5432`)** → use `5432` at runtime; it overrides `tests/test.env:17`.
- **"No rate-limit headers appear"** → expected: `flask-limiter 1.4` defaults `headers_enabled=False` and SimpleLogin never enables it (`server.py:167`). The 429 body is `{"error":"Rate limit exceeded"}`.
- **429-on-contention never fires** → the parallel-limiter lock and per-user create bucket are **no-ops when Redis is absent** (`parallel_limiter.py:51-52`); ensure Redis is running.
- **`test_minimal_payload` fails at module scope but passes in isolation** → pre-existing fixture-isolation flake (no SAVEPOINT in `tests/conftest.py`); out of scope, not a defect in the endpoint.

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---------|---------|
| Verify only the doc was added | `git diff 2cd6ee77..HEAD --name-status` |
| Verify source unchanged | `git diff 2cd6ee77..HEAD -- . ':(exclude)blitzy/documentation/*' \| wc -l` (→ 0) |
| Document line count | `wc -l blitzy/documentation/app_2cd6ee777f8c.md` (→ 6216) |
| Balanced code fences | `grep -c '^\`\`\`' blitzy/documentation/app_2cd6ee777f8c.md` (→ 176) |
| Start services **[container]** | `service postgresql start` · `service redis-server start` |
| Readiness **[container]** | `pg_isready -h localhost -p 5432 -U test` · `redis-cli ping` |
| Install deps **[container]** | `poetry install --no-interaction --no-ansi --no-root` |
| Live server **[container]** | `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` |
| Run module tests **[container]** | `pytest tests/api/test_new_custom_alias.py -v` |

### B. Port Reference

| Port | Service | Notes |
|------|---------|-------|
| 5432 | PostgreSQL | Runtime DB port (overrides `15432` in `tests/test.env:17`) |
| 6379 | Redis | Flask-Limiter store + parallel-limiter lock |
| 7777 | Gunicorn (`wsgi:app`) | Dockerfile CMD; `-w 2 --timeout 15` |

### C. Key File Locations

| Path | Role |
|------|------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The sole deliverable** (answer document) |
| `app/api/views/new_custom_alias.py` | Endpoint handlers `v2` [L28-112], `v3` [L115-235]; signed-suffix branch [L69-76] |
| `app/alias_suffix.py` | `signer` [L11]; `check_suffix_signature` [L37-42]; `verify_prefix_suffix` [L45-91] |
| `app/models.py` | `can_create_new_alias()` [L867-884]; `max_alias_for_free_account()` [L858-865] |
| `app/config.py` | `ALIAS_LIMIT` [L448]; `CUSTOM_ALIAS_SECRET` [L201]; `MAX_NB_EMAIL_FREE_PLAN` |
| `app/extensions.py` | Flask-Limiter instance + key function [L14-23] |
| `app/parallel_limiter.py` | Concurrency lock; Redis-absent no-op [L51-52] |
| `app/log.py` | `"SL"` logger → stdout [L79]; format [L12-15] |
| `server.py` | `create_app()`; `limiter.init_app(app)` [L167] |
| `tests/conftest.py` | `flask_client` fixture [L59-77] |
| `tests/test.env` | Runtime config (`EMAIL_DOMAIN=sl.local`, `MAX_NB_EMAIL_FREE_PLAN=3`, `FLASK_SECRET=secret`) |
| `Dockerfile` | `FROM python:3.10`; CMD `gunicorn wsgi:app ...` |

### D. Technology Versions

| Package | Version | Role |
|---------|---------|------|
| Python | 3.10.18 | Runtime |
| flask | 1.1.2 | Web framework / test client |
| flask-limiter | 1.4 | `@limiter.limit(ALIAS_LIMIT)`; header default `False` |
| itsdangerous | 1.1.0 | `TimestampSigner`; `BadSignature` superclass of `SignatureExpired` (root cause) |
| werkzeug | 1.0.1 | `TooManyRequests` (429) from parallel-limiter |
| redis | 4.6.0 | Limiter store + lock |
| sqlalchemy | 1.3.24 | ORM (`can_create_new_alias()` count query) |
| gunicorn | 20.0.4 | WSGI server (`wsgi:app` on `:7777`) |
| psycopg2-binary | 2.9.3 | PostgreSQL driver (`pg_trgm`) |

### E. Environment Variable Reference

| Variable | Value (investigation) | Purpose |
|----------|-----------------------|---------|
| `CONFIG` | `tests/test.env` | Selects the test configuration |
| `DB_URI` | `postgresql://test:test@localhost:5432/test` | Runtime DB (port 5432 overrides `15432` in `test.env:17`) |
| `GNUPGHOME` | `/tmp/sl_gnupg` | Clean dir with shipped PGP keys (or `create_app()` fails) |
| `PYTHONPATH` | `/app` | Importable package root |
| `DISABLE_RATE_LIMIT` | *(unset for Runner B)* | Presence-based (`config.py:602`); **unset** to enable rate limiting |
| `EMAIL_DOMAIN` | `sl.local` | Alias domain |
| `MAX_NB_EMAIL_FREE_PLAN` | `3` | Free-account alias cap (quota tests) |
| `FLASK_SECRET` | `secret` | Test-only; base of `CUSTOM_ALIAS_SECRET` (`+"custom_alias"`) |
| `MEM_STORE_URI` | `redis://localhost` | Flask-Limiter / lock store |

### F. Developer Tools Guide

No browser-based developer tooling applies (the deliverable is Markdown; investigated surfaces are JSON APIs). Tooling used during the investigation:

| Tool | Use |
|------|-----|
| `pytest` | Shipped module suite + `flask_client` harness for isolated reproductions |
| Werkzeug test client | Drives the real endpoint in-process (rolls back all writes) |
| `gunicorn` + `curl -i` | Over-the-wire raw-header capture (rate-limit-header answer) |
| `psql` / `redis-cli` | Data-store readiness, quota-count inspection, net-zero verification |
| `git` | Read-only verification of source integrity |

### G. Glossary

| Term | Definition |
|------|------------|
| **signed_suffix** | A `TimestampSigner`-signed alias suffix minted by `get_alias_suffixes` and surfaced via `GET /api/v4\|v5/alias/options`; validated by `check_suffix_signature`. |
| **check_suffix_signature** | Validator at `app/alias_suffix.py:37-42`; returns `None` on any `itsdangerous.BadSignature` (tampered/expired/garbage) → HTTP 412. |
| **BadSignature / SignatureExpired** | In `itsdangerous 1.1.0`, `SignatureExpired` and `BadTimeSignature` subclass `BadSignature`; the single `except` collapses all three to 412 (root cause). |
| **ALIAS_LIMIT** | `"100/day;50/hour;5/minute"` (`config.py:448`); the HTTP rate limit; first 429 at request #6. |
| **can_create_new_alias()** | Account-level quota gate (`models.py:867-884`); keys on `lifetime_or_active_subscription()` — trial users are still capped. |
| **parallel_limiter.lock** | Redis concurrency lock (`name="alias_creation"`); raises `TooManyRequests` (429); **no-op** when Redis is absent. |
| **Net-zero** | Post-investigation state: temporary probes removed and PostgreSQL/Redis restored to their pre-investigation baseline. |
| **412-collapse** | The lead finding: tampered, expired, and garbage suffixes all return the same HTTP 412 expiry response. |

---

*Cross-section integrity confirmed prior to submission: §2.1 (34h) + §2.2 (3h) = 37h = §1.2 Total; Remaining 3h identical across §1.2, §2.2, and §7; completion 34/37 = 91.9% consistent across §1.2, §7, and §8; all Section 3 results originate from Blitzy's autonomous validation logs; brand colors applied (Completed `#5B39F3`, Remaining `#FFFFFF`).*