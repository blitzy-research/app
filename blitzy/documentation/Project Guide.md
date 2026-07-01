# Blitzy Project Guide — SimpleLogin Alias‑Creation Investigation

> **Document type:** Run‑verified technical investigation & documentation deliverable
> **Branch:** `blitzy-58574f64-14a3-42b9-a002-7e260cc83c19` · **HEAD:** `201f909a` · **Base:** `2cd6ee77`
> **Deliverable:** `blitzy/documentation/app_2cd6ee777f8c.md` (1,105 lines · 80,653 bytes)

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a single, run‑verified technical investigation documenting exactly what happens when a user creates a new alias in **SimpleLogin**, an open‑source Python/Flask email‑aliasing application. The deliverable answers six requirements — live run (R1), frontend request (R2), backend response (R3), database changes (R4), side‑effects (R5), and error paths (R6) — across both creation surfaces (server‑rendered dashboard and REST API) and both variants (random and custom). Every claim is grounded in verbatim runtime evidence with exact `file:line` citations. The task is strictly **read‑only**: no source, schema, configuration, or test was modified. The audience is engineers onboarding to SimpleLogin's alias internals and reviewers validating system behavior.

### 1.2 Completion Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieOuterStrokeWidth':'2px','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieLegendTextColor':'#B23AF2'}}}%%
pie showData title Completion 93.6% — 44h of 47h (AAP-scoped)
    "Completed Work (h)" : 44
    "Remaining Work (h)" : 3
```

| Metric | Value |
|--------|-------|
| **Total Hours** | **47** |
| **Completed Hours (AI + Manual)** | **44** (44 AI · 0 Manual) |
| **Remaining Hours** | **3** |
| **Percent Complete** | **93.6%** |

> **Basis (PA1).** Completion measures AAP‑scoped + path‑to‑production work only. All 11 AAP‑specified deliverables are complete; the 3 remaining hours are exclusively human path‑to‑production (technical review + merge). Formula: `44 / (44 + 3) × 100 = 93.6%`. Legend: **Completed = Dark Blue `#5B39F3`**, **Remaining = White `#FFFFFF`**.

### 1.3 Key Accomplishments

- ✅ **Single mandated deliverable produced and committed** — `blitzy/documentation/app_2cd6ee777f8c.md` (1,105 lines), filename derived from source branch `app_2cd6ee777f8c`.
- ✅ **All six requirements answered (R1–R6)** with an explicit closing coverage pass confirming `R1 ✓ R2 ✓ R3 ✓ R4 ✓ R5 ✓ R6 ✓`.
- ✅ **Run‑first methodology honored** — the app was booted live (`alembic upgrade head && flask dummy-data && flask run`), authenticated as `john@wick.com`, and aliases were created on all four entry points.
- ✅ **Dual‑surface, dual‑variant coverage** — server‑rendered dashboard (302 + flash) and REST API (201 JSON), random and custom.
- ✅ **Database mutations proven by row‑count diffing** — 3 tables written on the direct/dashboard path (`alias`, `daily_metric`, `alias_audit_log`), a 4th (`api_key`) on the API surface, with `alias_mailbox`/`sync_event` correctly shown as conditional.
- ✅ **All 8 error paths (E1–E8) deliberately triggered and captured** — duplicate + `IntegrityError→rollback`, trashed, invalid prefix, tampered/expired suffix, quota, CSRF, rate‑limit, concurrency lock.
- ✅ **179/179 `file:line` citations verified accurate** (independently spot‑checked); 140 code fences balanced; 1 Mermaid diagram closed.
- ✅ **Read‑only scope perfectly preserved** — `git diff` shows exactly one added file; zero changes to `app/**`, `templates/**`, `migrations/**`, `tests/**`, or config.
- ✅ **Temporary observation harness removed** — no lasting repository or dependency footprint.

### 1.4 Critical Unresolved Issues

No critical, release‑blocking issues were identified. The deliverable passed all five autonomous production‑readiness gates with zero discrepancies. The only gating action is standard human review prior to merge.

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| _None — no in‑scope blockers_ | Deliverable is complete, verified, and committed | — | — |
| Pending human technical review (standard gate, not a defect) | Required before merge to target branch | Reviewing engineer | On review (≈2h) |

### 1.5 Access Issues

**No access issues identified.** The canonical execution environment (the user‑supplied Docker image with PostgreSQL 13, Redis, and the Python 3.10 venv pre‑provisioned) was fully available; the application booted, seeded, authenticated, and served requests, and the database was directly inspectable throughout the investigation.

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-----------------|----------------|-------------------|-------------------|-------|
| SimpleLogin app (`:7777`) | Runtime / HTTP | None — served and authenticated successfully | ✅ No issue | — |
| PostgreSQL 13 (`:5432`) | Database read | None — `psql` count(*) snapshots succeeded | ✅ No issue | — |
| Redis (`:6379`) | Cache/limiter | None — `PONG`; limiters exercised | ✅ No issue | — |
| Git repository / branch | Read/write (single file) | None — deliverable committed on correct branch | ✅ No issue | — |

### 1.6 Recommended Next Steps

1. **[High]** Conduct a technical review of `blitzy/documentation/app_2cd6ee777f8c.md` — confirm R1–R6 are fully answered, spot‑check a sample of the 179 citations against source, and optionally re‑run a few of the 26 documented producer commands (≈2h).
2. **[High]** Confirm no live secrets are present — verify the session‑cookie and CSRF redactions are intact and that only published dev fixtures appear (part of the review above).
3. **[Medium]** Decide the disposition of the 22 untracked screenshots and 1 screen recording under `blitzy/` — commit as supporting evidence or add to `.gitignore` (≈1h, combined with merge).
4. **[Medium]** Approve and merge the pull request into the target branch.
5. **[Low]** If SimpleLogin source is later modified, re‑verify the `file:line` citations (currently pinned to commit `2cd6ee77`) and optionally link the document from a docs index.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

All rows below are AAP‑scoped deliverables completed autonomously by Blitzy agents. Column total = **44h** (matches Completed Hours in §1.2).

| Component | Hours | Description |
|-----------|-------|-------------|
| R1 — Environment & live run | 3 | Provisioned standard dev setup; ran boot sequence (`alembic upgrade head && flask dummy-data && flask run`); established authenticated session for `john@wick.com` including CSRF scrape. |
| Read‑only observation harness | 4 | Built out‑of‑repo SQLAlchemy `before_cursor_execute` statement‑capture listener + per‑table `SELECT count(*)` snapshots + log tailing + HTTP replay; removed on completion. |
| R2 — Frontend request capture | 3 | Captured method/URL/headers/body verbatim across 4 entry points: dashboard random + custom (form POST) and API random + custom v2/v3 (JSON POST). |
| R3 — Backend response capture | 3 | Documented dashboard `302 + flash` and API `201` JSON; field‑by‑field mapping to `serialize_alias_info_v2`; grounded response timings/`Content-Length`. |
| R4 — Database‑change investigation | 5 | Proved via before/after row‑count diffs: 3 tables direct (`alias`, `daily_metric`, `alias_audit_log`), 4th (`api_key`) on API surface; captured INSERT/UPDATE statements; conditional `alias_mailbox`/`sync_event`; live table shapes. |
| R5 — Side‑effects investigation | 3 | Enumerated `Alias.create` steps; captured verbatim sync‑event suppression log; confirmed unconditional audit‑log write; documented log format. |
| R6 — Error‑path investigation | 8 | Deliberately triggered and captured all 8 categories (E1–E8, 12 sub‑cases) with HTTP + log + DB‑state; stood up a throwaway `:7778` limiter‑enabled instance for rate‑limit/lock proofs. |
| Document authoring | 8 | Authored 1,105‑line Q&A: title/methodology preamble, TOC, control‑flow Mermaid diagram, reasoning per answer, closing coverage pass. |
| Citation grounding & verification | 3 | Produced and verified 179 unique `file:line` citations against source. |
| Read‑only cleanup & scope preservation | 1 | Removed all temporary harness artifacts; verified clean working tree and single‑file diff. |
| Code‑review / QA remediation | 3 | Addressed code‑review + QA findings across 3 remediation commits (E4 flash mapping, R3 timings, R4 `daily_metric` back‑ref). |
| **Total** | **44** | |

### 2.2 Remaining Work Detail

All rows below are human path‑to‑production activities. Column total = **3h** (matches Remaining Hours in §1.2 and §7 pie chart).

| Category | Hours | Priority |
|----------|-------|----------|
| Documentation technical review (read 1,105 lines; verify R1–R6; spot‑check citations; confirm redactions) | 2 | High |
| PR merge & evidence‑artifact housekeeping (decide commit/gitignore for untracked screenshots & recording; merge) | 1 | Medium |
| **Total** | **3** | |

> _Optional / off critical path (0h): periodic citation re‑verification if the underlying SimpleLogin source is later modified; linking the document from a repository docs index._

---

## 3. Test Results

The deliverable is a Markdown document and has **no unit tests of its own**. The "tests" below are **Blitzy's autonomous runtime re‑verification checks** — every documented claim was reproduced live and every citation resolved — as recorded in the autonomous validation logs. All values originate from those logs (integrity rule satisfied).

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|------------|-------|
| Happy‑path flow reproduction (R1–R3) | HTTP replay (`curl`) + Flask runtime | 5 | 5 | 0 | 100% | dashboard random/custom + API random/custom v2/v3 |
| Database‑change verification (R4) | `psql` `SELECT count(*)` + SQLAlchemy event capture | 4 | 4 | 0 | 100% | 3‑table direct, 4th `api_key` on API, conditional `alias_mailbox`, `sync_event`=0 |
| Side‑effect verification (R5) | Server‑log capture | 3 | 3 | 0 | 100% | sync‑event suppression log, unconditional audit write, create log line |
| Error‑path reproduction (R6) | HTTP replay + log + DB diff | 12 | 12 | 0 | 100% | 8 categories / 12 sub‑cases (E1–E8), all DB diff = 0 |
| `file:line` citation resolution | Source anchor verification | 179 | 179 | 0 | 100% | zero drift vs current source |
| Markdown structural checks | `grep` / anchor slug check | 3 | 3 | 0 | 100% | 140 code fences balanced, 1 Mermaid closed, TOC anchors resolve |
| Source byte‑parity (md5) | `md5sum` (running app vs deliverable repo) | 13 | 13 | 0 | 100% | citations valid for the exact running app |
| **Total** | **Blitzy autonomous validation harness** | **219** | **219** | **0** | **100%** | zero discrepancies across all validation phases |

> **Disclosure (out of scope).** The repository's own `pytest` suite is explicitly out of scope for this read‑only task (modifying `tests/`/source is forbidden) and carries a **pre‑existing baseline of 556 passed / 83 failed** that is unrelated to this deliverable and was left untouched. It is **excluded** from all completion and hours figures.

---

## 4. Runtime Validation & UI Verification

**Runtime health** — the application was booted from the standard development setup and exercised live:

- ✅ **App boot** — `alembic upgrade head` (exit 0), `flask dummy-data` (seed confirmed via duplicate‑key on `john@wick.com`), `flask run --host=0.0.0.0 --port=7777` serving.
- ✅ **Reachability** — `GET /` → `302` (redirect to `/auth/login`); `GET /dashboard/` → `302` while unauthenticated.
- ✅ **Authentication** — login as `john@wick.com / password` established session + CSRF token.
- ✅ **PostgreSQL 13** — reachable; row‑count snapshots and table‑shape queries succeeded.
- ✅ **Redis** — `PONG`; token‑bucket and distributed‑lock limiters exercised.

**API integration outcomes:**

- ✅ `POST /api/alias/random/new` → `201 CREATED` with the documented JSON body (17 keys via `serialize_alias_info_v2`).
- ✅ `POST /api/v3/alias/custom/new` → `201`; `POST /api/v2/alias/custom/new` → `201`.
- ✅ Error contracts confirmed: `400` (invalid prefix / quota), `409` (duplicate), `412` (expired/tampered suffix), `429` (rate limit / concurrency lock).

**UI verification** (server‑rendered dashboard):

- ✅ **Dashboard random create** — form POST `/dashboard/` (`form-name=create-random-email`) → `302 FOUND` + success flash.
- ✅ **Dashboard custom create** — prefix/suffix/mailbox form → `302` + success flash.
- ✅ **Visual evidence captured** — 22 screenshots (login, dashboard, random/custom forms, success flashes, responsive breakpoints 375/768/1280/1920) + 1 screen recording of the random‑create flow (currently untracked; see §1.6 / §2.2).
- ⚠ **Note** — captured runtime values (alias ids, timings) are illustrative of shape, not run‑invariant; this is disclosed in the document.

---

## 5. Compliance & Quality Review

Cross‑map of AAP mandates (rule "SWE‑AtlasQnA‑Repo") and quality benchmarks to observed status.

| AAP Mandate / Benchmark | Requirement | Status | Progress | Evidence |
|--------------------------|-------------|--------|----------|----------|
| Single branch‑named deliverable | One file `blitzy/documentation/app_2cd6ee777f8c.md` | ✅ Pass | 100% | `git diff --name-status` = single `A` line |
| Run‑first, then write | Build/run before authoring | ✅ Pass | 100% | boot output + live captures throughout |
| Quote observed output verbatim | Logs, HTTP, counts with producing command | ✅ Pass | 100% | 140 fenced console/HTTP blocks; 26 producer commands |
| Answer every part (R1–R6) | Decompose + coverage pass | ✅ Pass | 100% | coverage table: `R1✓ R2✓ R3✓ R4✓ R5✓ R6✓` |
| Be exact & grounded | `file:line` for every factual claim | ✅ Pass | 100% | 179/179 citations verified accurate |
| Provide reasoning | Rationale per answer | ✅ Pass | 100% | "Answer & reasoning" narrative per section |
| Read‑only scope | No source/config/schema/test edits | ✅ Pass | 100% | zero non‑deliverable changes in diff |
| Harness cleanup | Temporary scripts removed | ✅ Pass | 100% | validator Gate 5; clean tree |
| Markdown well‑formedness | Balanced fences, valid diagram/anchors | ✅ Pass | 100% | 140 fences balanced; 1 Mermaid closed |

**Fixes applied during autonomous validation** (already merged into the deliverable):

- Corrected E4 dashboard tampered‑suffix flash mapping (QA finding **F1**, commit `f5fa84b0`).
- Grounded R3 response timings and fixed the R4 `daily_metric` back‑reference (QA **CP4 #1/#2**, commit `201f909a`).
- Remediated code‑review findings on the investigation (commit `d823c5e4`).

**Outstanding compliance items:** none in scope. Human review (§2.2) is the final quality gate.

---

## 6. Risk Assessment

Overarching context: the strictly **read‑only** nature (zero source/config/schema/dependency changes) eliminates the usual software risk surface — no compilation, regression, or dependency‑vulnerability risk. Residual risks are documentation‑specific and low.

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| T1 — Citation drift if source later changes | Technical | Low | Low | Citations pinned to commit `2cd6ee77`; re‑verify on source change | Mitigated |
| T2 — Env‑specific runtime values (ids/timings/counts) not byte‑reproducible | Technical | Low | Medium | Values documented as illustrative; methodology + seed pinned; disclosed in doc | Mitigated |
| S1 — Secret exposure in verbatim HTTP captures | Security | Medium | Low | Live session cookie + 91‑char CSRF redacted; only published dev fixtures shown; human review confirms | Mitigated |
| O1 — Untracked evidence artifacts (22 screenshots + 1 recording) | Operational | Low | Low | Human decision to commit or `.gitignore`; committed deliverable is the `.md` only | Open (minor, §2.2) |
| I1 — Reproduction depends on canonical Docker env | Integration | Low | Medium | 26 producer commands + full boot/migrate/seed/serve sequence documented | Mitigated |
| N1 — Pre‑existing repo `pytest` baseline (556P/83F) | Operational (informational) | Low | N/A | Out of scope; read‑only rule forbids touching tests/source; left untouched | Accepted (out of scope) |

---

## 7. Visual Project Status

**Project hours breakdown** (Completed = Dark Blue `#5B39F3`, Remaining = White `#FFFFFF`):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieOuterStrokeWidth':'2px','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieLegendTextColor':'#B23AF2'}}}%%
pie showData title Project Hours Breakdown
    "Completed Work" : 44
    "Remaining Work" : 3
```

**Remaining hours by category** (from §2.2, total = 3h):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'xyChart': {'plotColorPalette':'#5B39F3'}}}}%%
xychart-beta
    title "Remaining Hours by Category"
    x-axis ["Doc Technical Review", "PR Merge & Housekeeping"]
    y-axis "Hours" 0 --> 3
    bar [2, 1]
```

| Priority | Remaining Hours | Share |
|----------|-----------------|-------|
| High | 2 | 66.7% |
| Medium | 1 | 33.3% |
| **Total** | **3** | **100%** |

> **Integrity check:** "Remaining Work" = **3h** here equals §1.2 Remaining Hours (3) and the §2.2 Hours total (3). "Completed Work" = **44h** equals §1.2 Completed Hours (44). `44 + 3 = 47` = Total.

---

## 8. Summary & Recommendations

**Achievements.** This task delivered a rigorous, run‑verified investigation of SimpleLogin's alias‑creation flow as a single Markdown document. All 11 AAP‑specified deliverables are complete: the application was run live, every request/response was captured verbatim, database mutations were proven by row‑count diffing, side‑effects and all 8 error paths were reproduced, and every factual claim carries an accurate `file:line` citation (179/179 verified). The strict read‑only mandate was honored perfectly — the branch adds exactly one file and changes nothing else.

**Remaining gaps.** None in the AAP scope. The project is **93.6% complete** (44 of 47 hours). The outstanding 3 hours are exclusively human path‑to‑production: a technical review of the document (2h) and PR merge plus evidence‑artifact housekeeping (1h).

**Critical path to production.**
1. Human technical review of the deliverable (accuracy, coverage, secret‑redaction). →
2. Decide disposition of untracked evidence artifacts. →
3. Merge the pull request.

**Success metrics.** All met: single branch‑named deliverable ✓; R1–R6 answered with coverage pass ✓; run‑first + verbatim evidence ✓; 179/179 citations accurate ✓; read‑only scope intact ✓; 219/219 autonomous re‑verification checks passed ✓.

**Production‑readiness assessment.** The deliverable is **production‑ready** pending standard human review. It passed all five autonomous production‑readiness gates with zero discrepancies. Confidence is **High** — the scope is well‑defined, the evidence is reproducible, and the risk surface is minimal by virtue of the read‑only constraint.

---

## 9. Development Guide

This deliverable is documentation, so the guide covers (A) consuming the document, (B) reproducing the run‑verified investigation, and (C) verification commands. All commands in Part C were executed and passed during this assessment.

### 9.1 System Prerequisites

- **To read the document:** `git` and any Markdown viewer with Mermaid support (GitHub renders it natively; VS Code with a Mermaid extension; or `grip` for local preview).
- **To reproduce the investigation (optional):** Python **3.10** + Poetry, Node **v10** (front‑end assets), **PostgreSQL 13+**, **Redis** (port 6379). The canonical environment is the user‑supplied Docker image with these pre‑provisioned (Python 3.10.18 venv at `/app/venv`).

### 9.2 Consuming the Deliverable (primary)

```bash
# From the repository root, locate and open the deliverable
git switch blitzy-58574f64-14a3-42b9-a002-7e260cc83c19
less blitzy/documentation/app_2cd6ee777f8c.md

# Optional local rendered preview (renders Mermaid + tables in a browser)
pip install grip
grip blitzy/documentation/app_2cd6ee777f8c.md    # then open http://localhost:6419
```

Navigate via the in‑document Table of Contents to **R1–R6** and the closing **Coverage pass**.

### 9.3 Environment Setup & Dependency Installation (reproduction)

```bash
# 1. Backend dependencies
poetry sync

# 2. Front-end assets
cd static && npm install && cd ..

# 3. Local configuration
cp example.env .env
# Edit DB_URI in .env to match your Postgres port if needed, e.g.:
#   DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin

# 4. Start services
docker run -e POSTGRES_PASSWORD=mypassword -e POSTGRES_USER=myuser \
  -e POSTGRES_DB=simplelogin -p 5432:5432 postgres:13   # PostgreSQL 13
redis-server --daemonize yes                             # Redis on :6379
```

### 9.4 Application Startup

```bash
# Migrate -> seed -> serve (per CONTRIBUTING.md:L106)
alembic upgrade head && flask dummy-data && python3 server.py
# or, for a host-reachable server with plain-text logs:
FLASK_APP=server.py flask run --host=0.0.0.0 --port=7777
```

### 9.5 Verification Steps

```bash
# App is up (expect HTTP 302 redirect to /auth/login)
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:7777/     # -> 302

# Log in at the browser and create an alias:
#   open http://localhost:7777  ->  john@wick.com / password  ->  "Random alias"
```

### 9.6 Example Usage — create an alias via the REST API

```bash
# Seed dev API key is "code" (published dev fixture)
curl -sS -X POST http://localhost:7777/api/alias/random/new \
  -H 'Authentication: code' \
  -H 'Content-Type: application/json' \
  -d '{"note":"demo alias"}'
# -> HTTP 201 with JSON: {"alias":"...@sl.local","id":...,"creation_date":...,"enabled":true, ...}

# Inspect the database write (read-only)
PGPASSWORD=mypassword psql -h localhost -U myuser -d simplelogin \
  -c "SELECT id, email, created_at FROM alias ORDER BY id DESC LIMIT 1;"
```

### 9.7 Verification / QA Commands (tested in this assessment — all passed)

````bash
# Deliverable exists and is the expected size
wc -l blitzy/documentation/app_2cd6ee777f8c.md          # -> 1105

# Read-only scope: expect a SINGLE added file
git diff 2cd6ee77..HEAD --name-status                   # -> A  blitzy/documentation/app_2cd6ee777f8c.md

# Markdown code fences are balanced (expect an even number)
grep -c '```' blitzy/documentation/app_2cd6ee777f8c.md  # -> 140

# Spot-check a citation (doc claims "def create" at models.py:L1628)
sed -n '1628p' app/models.py                            # -> "    def create(cls, **kw):"

# Count documented producer commands
grep -c 'curl ' blitzy/documentation/app_2cd6ee777f8c.md # -> 26

# Working tree has no tracked modifications
git status --porcelain --untracked-files=no | wc -l     # -> 0
````

### 9.8 Troubleshooting

- **`flask dummy-data` fails with a duplicate‑key error on `john@wick.com`** — expected if the DB is already seeded; it *proves* the seed row exists. Safe to ignore.
- **Limiters appear to "do nothing"** — the token‑bucket and distributed lock **no‑op when Redis is not configured** (`rate_limiter.py:L28-L29`, `parallel_limiter.py:L51-L52`); start Redis to exercise the 429 paths.
- **A cited line number doesn't match** — citations are pinned to commit `2cd6ee77`; check out that commit or re‑verify against current source.
- **Runtime ids/timings differ from the document** — expected; these depend on seeded DB state and are illustrative of shape, not invariants.
- **Mermaid diagram/pie not rendering** — view on GitHub or a Mermaid‑capable viewer; raw `less`/`cat` will show the fenced source.

---

## 10. Appendices

### Appendix A — Command Reference

| Command | Purpose |
|---------|---------|
| `alembic upgrade head` | Migrate schema to latest revision |
| `flask dummy-data` | Seed development database (creates `john@wick.com`) |
| `python3 server.py` / `flask run --host=0.0.0.0 --port=7777` | Start the Flask web process |
| `poetry sync` | Install backend dependencies |
| `cd static && npm install` | Build front‑end assets |
| `git diff 2cd6ee77..HEAD --name-status` | Verify read‑only scope (single added file) |
| `PGPASSWORD=mypassword psql -h localhost -U myuser -d simplelogin` | Inspect the database (read‑only) |
| `redis-server --daemonize yes` | Start Redis for limiter/lock paths |

### Appendix B — Port Reference

| Port | Service | Notes |
|------|---------|-------|
| 7777 | SimpleLogin web (Flask) | Dashboard + REST API; `URL=http://localhost:7777` (`example.env:L6`) |
| 5432 | PostgreSQL 13 | Primary DB; `DB_URI` target (`example.env:L75`) |
| 6379 | Redis | Limiter/lock storage; limiters no‑op when absent |
| 7778 | Throwaway limiter‑enabled Flask instance | Temporary (validation only); terminated on completion |
| 6419 | `grip` preview (optional) | Local Markdown rendering |

### Appendix C — Key File Locations

| Path | Role |
|------|------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The deliverable** (only file added) |
| `app/models.py` | `Alias.create` (L1628); `DailyMetric.get_or_create_today_metric` (L3280) |
| `app/dashboard/views/index.py` | Dashboard random route (L55); commit/log/redirect (L104–L119) |
| `app/dashboard/views/custom_alias.py` | Custom‑alias route (L30); create/flush/rollback (L139–L156) |
| `app/api/views/new_random_alias.py` | API random route (L21); 201 response (L114–L117) |
| `app/api/views/new_custom_alias.py` | API custom v2/v3 routes; 400/409/412 contracts |
| `app/events/event_dispatcher.py` | Sync‑event dispatch + guard clauses (L48, L57–L70) |
| `app/alias_audit_log_utils.py` | Unconditional audit‑log write (L18) |
| `app/api/serializer.py` | `serialize_alias_info_v2` — 201 JSON shape (L55) |
| `app/rate_limiter.py` / `app/parallel_limiter.py` | Token bucket / distributed lock (429 paths) |
| `templates/dashboard/index.html` | Alias‑creation forms + CSRF token |
| `CONTRIBUTING.md` / `example.env` | Setup guide (L106–L109) / default config |

### Appendix D — Technology Versions

| Component | Version | Source |
|-----------|---------|--------|
| Python (canonical runtime) | 3.10.18 (venv `/app/venv`) | Docker image |
| Flask | 1.1.2 | `poetry.lock` |
| SQLAlchemy | 1.3.24 (pinned) | `poetry.lock` |
| PostgreSQL | 13 | `CONTRIBUTING.md` |
| Redis (redis‑py) | 4.6.0 | runtime |
| Node.js (front‑end) | v10 | `CONTRIBUTING.md` |
| Git (assessment host) | 2.51.0 | measured |
| `.version` | `dev` | repository root |

### Appendix E — Environment Variable Reference

| Key | Default | `file:line` | Effect on the flow |
|-----|---------|-------------|--------------------|
| `URL` | `http://localhost:7777` | `example.env:L6` | Base URL the app serves and logs |
| `EMAIL_DOMAIN` | `sl.local` | `example.env:L22` | Random aliases end in `@sl.local` |
| `DB_URI` | `postgresql://myuser:mypassword@localhost:5432/simplelogin` | `example.env:L75` | Target database for all writes |
| `FLASK_SECRET` | `secret` | `example.env:L77` | Signs sessions and the alias suffix |
| `EVENT_WEBHOOK` | `None` | `app/config.py:L612` | When unset, sync‑event dispatch is suppressed |
| `EVENT_WEBHOOK_DISABLE` | `False` | `app/config.py:L616` | Hard‑disables event dispatch when set |
| `MAX_NB_EMAIL_FREE_PLAN` | `5` | `app/config.py:L121-L124` | Free‑plan alias quota (drives E5) |

### Appendix F — Developer Tools Guide

| Tool | Use in this project |
|------|---------------------|
| `psql` | Read‑only DB inspection; `SELECT count(*)` row‑count diffs (R4) |
| `curl` | HTTP replay of dashboard form POSTs and API JSON POSTs (R2/R3/R6) |
| SQLAlchemy `event.listen` (`before_cursor_execute`) | Out‑of‑repo statement capture to prove tables touched (R4) |
| Server log tailing | Verbatim log‑line capture for side‑effects and errors (R5/R6) |
| `git diff` / `md5sum` | Read‑only scope verification and source byte‑parity |
| `grip` (optional) | Local rendered Markdown/Mermaid preview |

### Appendix G — Glossary

| Term | Definition |
|------|------------|
| **AAP** | Agent Action Plan — the governing specification for this task |
| **Alias** | A SimpleLogin email alias forwarding to a user's real mailbox |
| **Random / Custom variant** | Auto‑generated alias vs. user‑specified prefix + signed suffix |
| **Dashboard surface** | Server‑rendered Jinja2 UI submitting form POSTs to `/dashboard/` |
| **REST API surface** | Token‑authenticated JSON endpoints under `/api/...` |
| **Sync event** | Protobuf `AliasCreated` event dispatched via PostgreSQL `LISTEN/NOTIFY`; suppressed for default non‑partner users |
| **Audit log** | `alias_audit_log` row written unconditionally on creation (`action="create"`) |
| **Producer command** | A `curl`/CLI command included in the doc that reproduces a captured observation |
| **Read‑only scope** | The rule that no existing repository file may be modified; only the deliverable is added |

---

*Generated by the Blitzy autonomous assessment agent. Completion basis: PA1 AAP‑scoped hours — 44 completed / 47 total = 93.6%. Cross‑section integrity validated (§1.2 ↔ §2.2 ↔ §7 remaining = 3h; §2.1 + §2.2 = 47h).*