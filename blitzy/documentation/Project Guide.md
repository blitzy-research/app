# Blitzy Project Guide — SimpleLogin Alias-Creation Investigation (Q&A)

> **Deliverable:** `blitzy/documentation/app_2cd6ee777f8c.md` — a runtime-verified, `file:line`-cited answer explaining end-to-end how SimpleLogin creates a new email alias.
> **Task type:** Read-only investigation / Q&A (documentation). No existing source file modified.
> **Brand color legend:** 🟦 **Completed / AI Work** = Dark Blue `#5B39F3` · ⬜ **Remaining / Not Completed** = White `#FFFFFF` · Headings/Accents = Violet-Black `#B23AF2` · Highlight = Mint `#A8FDD9`.

---

## 1. Executive Summary

### 1.1 Project Overview

This project produces an authoritative, runtime-verified explanation of how the **SimpleLogin** Flask email-alias service behaves end-to-end when a user creates a new alias. It targets engineers and reviewers who need ground truth — the exact frontend request, backend response, database mutations, background work, and failure behavior — established by *running* the application rather than reading code. The technical scope spans the web dashboard flow (HTTP 302), the JSON API flow (HTTP 201), the shared `Alias.create` persistence core, the event pipeline, and every documented error path. The single deliverable is one Markdown answer document; **zero existing source files are changed**, satisfying a strict read-only mandate. Business impact: a reliable, citation-backed reference that de-risks future changes to the alias-creation subsystem.

### 1.2 Completion Status

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieTitleTextSize':'16px','pieSectionTextSize':'14px','pieLegendTextColor':'#333333'}}}%%
pie showData title Completion — 95.5% Complete (42h of 44h)
    "Completed Work (AI)" : 42
    "Remaining Work (Human)" : 2
```

<sub>🟦 Completed = `#5B39F3` · ⬜ Remaining = `#FFFFFF`</sub>

| Metric | Hours |
|---|---|
| **Total Hours** | **44.0** |
| Completed Hours (AI + Manual) | 42.0 |
| &nbsp;&nbsp;• AI (Blitzy autonomous) | 42.0 |
| &nbsp;&nbsp;• Manual (human, to date) | 0.0 |
| Remaining Hours | 2.0 |
| **Percent Complete** | **95.5%** |

**Calculation (PA1, AAP-scoped):** Completion % = Completed ÷ Total × 100 = **42 ÷ 44 × 100 = 95.45% → 95.5%**.

### 1.3 Key Accomplishments

- ✅ **Full application run locally (O1)** — canonical dev runtime stood up in the attached Docker container: `gunicorn wsgi:app` on port 7777, PostgreSQL 13, Redis 7, schema at head, seeded dummy data.
- ✅ **Authenticated as the seeded test user (O2)** — `john@wick.com` login flow captured (CSRF token length 91, `POST /auth/login` → 302 → authenticated dashboard 200).
- ✅ **Alias creation observed on both entry points (O3)** — the web "Random Alias" form and the JSON API, driven through real HTTP.
- ✅ **Exact frontend request captured (O4)** — `application/x-www-form-urlencoded` `POST /dashboard/` with `form-name=create-random-email` + `csrf_token` (+ optional `generator_scheme`).
- ✅ **Backend responses captured (O5)** — web **HTTP 302** redirect to `/dashboard/?highlight_alias_id=<id>`; API **HTTP 201** with a 17-key JSON body, each key mapped to its serializer source line.
- ✅ **Database delta proven (O6)** — the **3-table creation core** (`alias` INSERT, `daily_metric` INSERT/UPDATE, `alias_audit_log` INSERT) measured with read-only before/after probes and **stable across ≥2 identical runs**; API adds one `api_key` auth write only.
- ✅ **Logs & background work characterized (O7)** — single creation log line + event-dispatch short-circuit (`EVENT_WEBHOOK` unset); `job_runner.py`/`event_listener.py` run and stay idle; no `NOTIFY`/`sync_event`.
- ✅ **Failure behavior enumerated (O8)** — free-plan cap, trashed-alias reuse, invalid CSRF, invalid API mode, rate-limit 429, malformed-hostname 500 — each with response + log + before/after DB probe.
- ✅ **Read-only mandate honored** — single-file diff, clean working tree, all temporary scripts removed; **151 `file:line` citations**, coverage-pass table.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|---|---|---|---|
| _None — no unresolved issues block release or validation._ | The in-scope deliverable is complete, runtime-verified, and required zero validator edits. | — | — |

### 1.5 Access Issues

| System / Resource | Type of Access | Issue Description | Resolution Status | Owner |
|---|---|---|---|---|
| _n/a_ | _n/a_ | **No access issues identified.** The canonical runtime (Docker image with prebuilt `venv`, PostgreSQL 13, Redis 7) is fully available; the app builds, runs, and serves; git branch and history are accessible. | Resolved / N/A | — |

### 1.6 Recommended Next Steps

1. **[High]** Perform SME/technical review and acceptance of `blitzy/documentation/app_2cd6ee777f8c.md` — spot-check a sample of the 151 citations against the base commit and sanity-check the runtime claims. *(~1.5h)*
2. **[High]** Approve the pull request and merge the single-file addition into the target branch. *(~0.5h)*
3. **[Low, optional]** Independently re-run one read-only DB row-count probe to reconfirm the 3-table delta (already proven autonomously twice) as a confidence aid. *(0h — optional)*
4. **[Out-of-scope]** Track the pre-existing `google-re2`/`pyre2` regex-test mismatch and `test_apple` external-API timeout under a **separate authorized task** (both are unfixable within this read-only mandate). *(0h here)*

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

Each component traces to a specific AAP objective (O1–O8) or ruleset (§0.7 R9–R16).

| Component | Hours | Description |
|---|---:|---|
| Environment standup & canonical dev runtime **(O1)** | 4.0 | Docker-in-Docker; start PostgreSQL 13 + Redis 7; apply Alembic schema to head; `flask dummy-data`; launch `gunicorn wsgi:app` on :7777; verify health. |
| Test-user authentication capture **(O2)** | 1.5 | Drive `/auth/login` for `john@wick.com`; capture CSRF (len 91), cookies, 302→dashboard 200. |
| Happy-path web + API request/response capture + 17-key JSON enumeration **(O3/O4/O5)** | 5.0 | Capture verbatim FE request (132 bytes); web 302 `Location`; API 201; map all 17 JSON keys to `serialize_alias_info_v2` source lines. |
| Database delta instrumentation + 2-run stability proof **(O6)** | 4.5 | Read-only row-count probe over 8 candidate tables; before/after through the real HTTP path; prove 3-table core; `daily_metric` INSERT-vs-UPDATE sub-cases; API `api_key` write; confirm stability ×2. |
| Logs & background-worker observation **(O7)** | 3.0 | Per-request log slice; trace `EventDispatcher` three early returns / short-circuit; run `job_runner.py` + `event_listener.py` and confirm idle; prove no `NOTIFY`/`sync_event`. |
| Failure-path reproduction ×6 **(O8)** | 8.0 | Free-plan cap (runtime-only free user topped to cap), trashed-alias reuse (2 surfaces), invalid CSRF, invalid API mode, rate-limit 429 (2-worker bucket taxonomy), malformed-hostname 500 — each with response + log + DB probe. |
| Answer-document authoring **(R9–R16)** | 10.0 | 1,381 lines / 12,598 words / 10 sections / 151 `file:line` citations; raw output embedded verbatim; coverage-pass table + evidence index. |
| Review-fix cycles ×3 + independent final live re-verification | 6.0 | Code-review rework (+686/-266), citation-precision nits (CP4), acceptance findings (CP5); plus the Final Validator's independent live O1–O8 re-verification. |
| **Total Completed** | **42.0** | Matches Completed Hours in §1.2. |

### 2.2 Remaining Work Detail

Each category is standard **path-to-production** for a documentation deliverable (no code ships).

| Category | Hours | Priority |
|---|---:|---|
| Human SME/technical review & acceptance of the answer document (R18) | 1.5 | High |
| PR approval & merge/publish to target branch (R19) | 0.5 | High |
| **Total Remaining** | **2.0** | Matches Remaining Hours in §1.2 and Section 7 pie. |

> **Optional / out-of-scope (0h — excluded from totals):** independent DB-probe re-run (confidence aid, already proven); pre-existing `google-re2`/`pyre2` + `test_apple` issues (unfixable within read-only scope). These carry **0 hours** and do not affect the arithmetic.

### 2.3 Hours Reconciliation

| Check | Result |
|---|---|
| Section 2.1 total (Completed) | 42.0 |
| Section 2.2 total (Remaining) | 2.0 |
| **2.1 + 2.2 = Total (§1.2)** | **42.0 + 2.0 = 44.0 ✅** |
| Remaining consistent across §1.2 / §2.2 / §7 | 2.0 = 2.0 = 2.0 ✅ |
| Completion % | 42 / 44 = 95.5% ✅ |

---

## 3. Test Results

All results below originate from **Blitzy's autonomous validation logs** for this project. Because the task is a read-only Q&A investigation, the authoritative validation is the direct **runtime observation** of the alias-creation flow through the real HTTP entry points; the relevant **unit suites** were additionally executed for the exact functionality the document describes.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---|---|---:|---:|---:|---:|---|
| Unit — Alias creation (random) | pytest | 6 | 6 | 0 | n/a (scope-focused) | `tests/api/test_new_random_alias.py` — the random-alias endpoint the document documents. |
| Unit — Alias creation (custom) | pytest | 10 | 10 | 0 | n/a (scope-focused) | `tests/api/test_new_custom_alias.py` — sibling custom-alias endpoint. |
| Runtime — Web flow (302) | curl vs live gunicorn | 1 | 1 | 0 | — | `POST /dashboard/` → HTTP 302 `Location: /dashboard/?highlight_alias_id=<id>`. |
| Runtime — API flow (201) | curl vs live gunicorn | 1 | 1 | 0 | — | `POST /api/alias/random/new` → HTTP 201 + 17-key JSON. |
| Runtime — DB delta stability | SELECT-only probe ×2 | 2 | 2 | 0 | — | 3-table core stable across two identical web creations. |
| Runtime — Failure paths | curl vs live gunicorn | 6 | 6 | 0 | — | free-cap, trashed reuse, CSRF, mode, rate-limit 429, malformed-hostname 500 — all reproduced. |
| **In-scope total** | — | **26** | **26** | **0** | — | **16 unit + 10 runtime observations; 0 failures.** |

> **Out-of-scope (reported, not part of this deliverable):** a `google-re2` vs `poetry.lock` `pyre2` mismatch causes ~3 collection errors + 1 regex-test failure in email/regex test files, and `tests/test_apple.py::test_apple_process_payment` times out against an external API with no network. Neither touches the alias-creation flow; both are **unfixable within the read-only scope** (would require a forbidden dependency change or edits to read-only files).

---

## 4. Runtime Validation & UI Verification

**Runtime health (live application):**
- ✅ **Operational** — App factory + WSGI: `gunicorn wsgi:app` serving on `0.0.0.0:7777` (Python 3.10.18; Flask 1.1.2; SQLAlchemy 1.3.24).
- ✅ **Operational** — Datastore: PostgreSQL 13 (schema at head); Cache: Redis 7 available.
- ✅ **Operational** — Authentication: `GET /auth/login` (CSRF len 91) → `POST` 302 → authenticated `/dashboard/` 200 for `john@wick.com`.

**API integration outcomes:**
- ✅ **Operational** — Web creation: `POST /dashboard/` → **HTTP 302**, `Location: http://localhost:7777/dashboard/?highlight_alias_id=<id>&query=&sort=&filter=`.
- ✅ **Operational** — API creation: `POST /api/alias/random/new` → **HTTP 201** + JSON (16 keys from `serialize_alias_info_v2` + top-level `alias`; `latest_activity=null`).
- ✅ **Operational** — DB effects: `alias` INSERT, `daily_metric` INSERT/UPDATE, `alias_audit_log` INSERT — **3 tables, stable ×2**; API adds `api_key` auth counters only.
- ✅ **Operational** — Background/events: event dispatch short-circuits (`EVENT_WEBHOOK` unset); no `sync_event`/`NOTIFY`; workers idle.

**UI verification:**
- ✅ **Operational** — The "Random Alias" control in `templates/dashboard/index.html` renders and submits the documented form; post-creation redirect highlights the new alias via `highlight_alias_id`.
- ⚠ **Partial (by design / out of scope)** — No new UI was built or changed; only the existing dashboard control was observed. No visual regression suite applies to this documentation task.

**Failure-surface verification (all reproduced live):**
- ✅ Free-plan cap → API **400** / web **302** + upgrade flash (zero DB write).
- ✅ Invalid CSRF → **302** back to `request.url` + "Invalid request" (zero DB write).
- ✅ Invalid API `mode` → **400** (`… must be either word or uuid`).
- ✅ Trashed-alias reuse → `AliasInTrashError` (random endpoint catches → 201 fallback; custom endpoint → **409**).
- ✅ Rate limiting → **429** `{"error":"Rate limit exceeded"}` (observed 10×201 then 10×429 under 2 workers × in-memory bucket).
- ✅ Malformed `hostname` (API) → **500** `EmailSyntaxError` edge (documented and evidenced, zero write).

---

## 5. Compliance & Quality Review

Cross-mapping AAP deliverables and rules to quality/compliance benchmarks. Fixes applied during autonomous validation are noted.

| Benchmark / AAP Requirement | Status | Progress | Notes |
|---|:--:|:--:|---|
| **O1** Run full app locally (canonical config) | ✅ Pass | 100% | gunicorn :7777 + Postgres 13 + Redis 7, seeded. |
| **O2** Authenticate seeded test user | ✅ Pass | 100% | `john@wick.com` login flow captured. |
| **O3** Create alias while observing (web + API) | ✅ Pass | 100% | Both entry points exercised. |
| **O4** Capture exact frontend request | ✅ Pass | 100% | Verbatim `x-www-form-urlencoded` POST captured. |
| **O5** Observe backend response | ✅ Pass | 100% | 302 (web) + 201 (API, 17-key JSON) captured. |
| **O6** Identify DB changes (count + entities) | ✅ Pass | 100% | 3-table core proven, stable ×2; absences confirmed. |
| **O7** Inspect logs / background work | ✅ Pass | 100% | Short-circuit + idle workers observed. |
| **O8** Characterize failure behavior | ✅ Pass | 100% | 6 error paths reproduced with response/log/DB surfaces. |
| **R9** Exactly one doc, correct name/location | ✅ Pass | 100% | `blitzy/documentation/app_2cd6ee777f8c.md`. |
| **R10** Every claim grounded (`file:line`/output) | ✅ Pass | 100% | 151 citations; raw output before summaries. |
| **R11** Run-first methodology | ✅ Pass | 100% | Written from observation, not reading. |
| **R12** Exercise real HTTP entry points | ✅ Pass | 100% | curl vs gunicorn; no internal-helper bypass. |
| **R13** Default canonical configuration | ✅ Pass | 100% | `EVENT_WEBHOOK`/`MEM_STORE_URI` unset; `NOT_SEND_EMAIL=true`. |
| **R14** Exercise happy + every error/edge path | ✅ Pass | 100% | §3–§7 of the answer document. |
| **R15** Confirm magnitude stability (≥2 runs) | ✅ Pass | 100% | 3 tables on both runs (§5.1/§5.2). |
| **R16** Coverage pass (every named item) | ✅ Pass | 100% | §9 mapping table. |
| **R17** Read-only: no source modified; scripts cleaned | ✅ Pass | 100% | Single-file diff; clean `/tmp`; clean tree. |
| **Markdown hygiene** | ✅ Pass | 100% | 10 sections; balanced code fences (124); 0 trailing whitespace. |
| **`[inferred]` labeling of read-only statements** | ✅ Pass | 100% | Inferred statements explicitly labeled. |
| **Secret hygiene** | ✅ Pass | 100% | Session-cookie payloads redacted; CSRF/local test key are runtime-local fixtures, labeled. |
| **R18** Human review/acceptance | ⏳ Pending | 0% | Path-to-production; scheduled in §2.2. |
| **R19** PR merge/publish | ⏳ Pending | 0% | Path-to-production; scheduled in §2.2. |

**Fixes applied during autonomous validation:** three review-driven revision cycles were absorbed autonomously before final validation — a major code-review rework (commit `2a2a893e`, +686/-266), three citation-precision nits (commit `c8e7caa1`, CP4), and acceptance findings (commit `892111a0`, CP5, +140/-25). The Final Validator then re-verified all runtime claims and required **zero further edits**.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|---|---|:--:|:--:|---|---|
| Citation line-drift — 151 `file:line` refs pinned to base commit; future source edits could shift line numbers. | Technical | Low | Medium | Citations anchored to base commit `2cd6ee77`; reviewer verifies against that commit. | Documented / Accepted |
| Config-dependent counts — the "3 tables / no background work" result holds only in default config (`EVENT_WEBHOOK` unset, single mailbox, non-partner user). | Technical | Low | Low | Default-config boundary + conditional-write caveats explicitly stated (§8.2 / §7.6 of the answer). | Mitigated |
| Secret exposure — captured output could leak sensitive values. | Security | Low | Low | Session-cookie payloads redacted; CSRF token + local test key `freecode` are runtime-local fixtures (not production secrets), labeled as such. | Mitigated |
| Operational surface — none (static document; no deployed service). | Operational | Low | Low | No runtime service ships; nothing to monitor/health-check. | N/A (no exposure) |
| Pre-existing out-of-scope test issues — `google-re2` vs `pyre2` (~3 collection errors + 1 failure); `test_apple` external-API timeout. | Integration | Low | N/A | Unrelated to alias creation; unfixable in read-only scope (would violate §0.3.2/§0.4.2/§0.7.4); route to a separate authorized task. | Documented / Out-of-scope |
| Reviewer interpretation/nuance disagreement on characterizations. | Documentation | Low | Low | Exhaustive coverage-pass table (§9) maps every named item; premium-user nuance flagged prominently. | Pending human review |

**Overall risk posture: LOW.** No code changes ship, so there is no runtime/security/operational attack surface introduced. The dominant risks are documentation-integrity risks (citation precision, configuration boundaries), all mitigated in-document.

---

## 7. Visual Project Status

**Project hours (AAP-scoped):**

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieTitleTextSize':'16px','pieSectionTextSize':'14px','pieLegendTextColor':'#333333'}}}%%
pie showData title Project Hours Breakdown (Total 44h)
    "Completed Work" : 42
    "Remaining Work" : 2
```

<sub>🟦 Completed Work = `#5B39F3` (42h) · ⬜ Remaining Work = `#FFFFFF` (2h). "Remaining Work" (2h) equals §1.2 Remaining Hours and the §2.2 total.</sub>

**Remaining hours by priority (from §2.2):**

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#B23AF2','pieStrokeColor':'#FFFFFF','pieOuterStrokeColor':'#B23AF2','pieStrokeWidth':'1px'}}}%%
pie showData title Remaining Work by Task (2h, both High priority)
    "SME review & acceptance" : 1.5
    "PR approval & merge" : 0.5
```

**Completion at a glance:**

| Dimension | Completed | Remaining | Total | % |
|---|---:|---:|---:|---:|
| AAP-scoped hours | 42.0 | 2.0 | 44.0 | 95.5% |
| AAP autonomous requirements (R1–R17) | 17 | 0 | 17 | 100% |
| Path-to-production items (R18–R19) | 0 | 2 | 2 | 0% |

---

## 8. Summary & Recommendations

**Achievements.** The project delivers a complete, exhaustive, runtime-verified answer to the multi-part question of how SimpleLogin creates a new email alias. Every objective (O1–O8) and every §0.7 rule (R9–R17) is satisfied and evidenced: the frontend contract (`x-www-form-urlencoded` `POST /dashboard/`), the two backend responses (web 302, API 201 with a 17-key JSON body), the **3-table creation core** proven stable across repeated runs, the event-dispatch short-circuit and idle background workers, and six distinct failure surfaces — each with raw output and `file:line` citations. The read-only mandate is honored exactly: a single-file diff, a clean working tree, and all temporary observation scripts removed.

**Remaining gaps.** Nothing in the AAP autonomous scope remains. The only outstanding work is **path-to-production for a documentation artifact**: human SME review/acceptance and PR merge (2.0h total). There is no deployment pipeline, service, or configuration to ship — the document *is* the product.

**Critical path to production.** (1) SME reviews and accepts the document (spot-check citations against base commit, sanity-check runtime claims); (2) approve and merge the PR. Estimated **2.0h**, both High priority.

**Success metrics.**

| Metric | Target | Actual |
|---|---|---|
| AAP objectives answered (O1–O8) | 8 / 8 | 8 / 8 ✅ |
| §0.7 rules satisfied (R9–R17) | 9 / 9 | 9 / 9 ✅ |
| Source files modified | 0 | 0 ✅ |
| `file:line` citations | high | 151 ✅ |
| Alias-creation unit tests passing | 16 / 16 | 16 / 16 ✅ |
| Validator edits required | 0 | 0 ✅ |

**Production-readiness assessment.** The in-scope deliverable is **production-ready at 95.5% overall completion** (42h of 44h). The residual 4.5% is exclusively human review/acceptance and merge — activities that, by policy, cannot be self-certified autonomously (max autonomous completion is capped below 100%). Recommendation: **proceed to human review and merge**; no rework is anticipated.

---

## 9. Development Guide

> All commands below were tested against the live environment during assessment. The application runtime steps follow the canonical procedure in `CONTRIBUTING.md`.

### 9.1 System Prerequisites

- **Docker Engine** (the canonical runtime ships as a container image).
- **Python 3.10** (canonical; image provides 3.10.18 with `poetry.lock` pins prebuilt in `venv/`).
- **Node.js 10.17.0** (frontend asset build; prebuilt in the image).
- **PostgreSQL 13** (primary datastore).
- **Redis 7** (rate-limit storage; optional in dev — limiter falls back to in-memory).

### 9.2 Environment Setup (canonical dev runtime)

```bash
# 1) Install Python dependencies (prebuilt in the image's venv)
poetry sync                     # CONTRIBUTING.md:L31

# 2) Build frontend assets (prebuilt in the image)
cd static && npm install        # CONTRIBUTING.md:L82
cd ..

# 3) Create local config from the template, then set DB_URI
cp example.env .env             # CONTRIBUTING.md:L88
# edit .env: DB_URI=postgresql://myuser:mypassword@localhost:15432/simplelogin   # L94

# 4) Start PostgreSQL 13
docker run -e POSTGRES_PASSWORD=mypassword -e POSTGRES_USER=myuser \
  -e POSTGRES_DB=simplelogin -p 15432:5432 postgres:13            # CONTRIBUTING.md:L100
```

### 9.3 Application Startup

```bash
# Apply schema, seed dev data, and serve the webapp on port 7777
alembic upgrade head && flask dummy-data && python3 server.py     # CONTRIBUTING.md:L106  (server bind: server.py:L588)

# Optional background workers (observe follow-up work)
python job_runner.py
python event_listener.py        # only needed when observing webhook follow-up
```

Login with the seeded credentials: **`john@wick.com` / `password`**.

### 9.4 Verification Steps (tested — expected output shown)

```bash
# Repository integrity — expect empty output (clean tree)
git status --porcelain | wc -l
# -> 0

# Exactly one file added vs base — expect a single 'A' line
git diff --name-status 2cd6ee77 HEAD
# -> A	blitzy/documentation/app_2cd6ee777f8c.md

# Deliverable present and has 10 top-level sections
test -f blitzy/documentation/app_2cd6ee777f8c.md && echo FILE PRESENT
grep -cE '^## [0-9]+\.' blitzy/documentation/app_2cd6ee777f8c.md
# -> FILE PRESENT
# -> 10

# Citation density — expect 151 file:line references
grep -oE '[A-Za-z0-9_/.-]+:L[0-9]+' blitzy/documentation/app_2cd6ee777f8c.md | wc -l
# -> 151

# Code-fence balance — expect an even number
grep -cE '^```' blitzy/documentation/app_2cd6ee777f8c.md
# -> 124

# Alias-creation unit tests discovered — expect 16
grep -hcE '^def test_' tests/api/test_new_random_alias.py tests/api/test_new_custom_alias.py \
  | awk '{s+=$1} END{print s}'
# -> 16
```

### 9.5 Example Usage — reproduce the documented observation

```bash
# WEB flow -> HTTP 302 (requires an authenticated cookie jar + a fresh CSRF token)
curl -sS -o /dev/null -D - -b john.cookies -X POST http://localhost:7777/dashboard/ \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data 'form-name=create-random-email&csrf_token=<valid token>' | grep -iE '^HTTP|^Location'
# -> HTTP/1.1 302 FOUND
# -> Location: http://localhost:7777/dashboard/?highlight_alias_id=<id>&query=&sort=&filter=

# API flow -> HTTP 201 + JSON (requires an API key in the Authentication header)
curl -sS -D - -X POST http://localhost:7777/api/alias/random/new \
  -H 'Authentication: <api-key>' -H 'Content-Type: application/json' -d '{}'
# -> HTTP/1.1 201 CREATED  +  17-key JSON body (serialize_alias_info_v2 + top-level "alias")

# Read the answer document
less blitzy/documentation/app_2cd6ee777f8c.md
```

### 9.6 Troubleshooting

- **Read-only mandate:** never edit any existing source file — the *only* permitted write is the answer document. Verify with `git diff --name-status 2cd6ee77 HEAD` (must show a single `A` line).
- **Citations don't match source:** the 151 `file:line` references are pinned to base commit `2cd6ee77`; verify against that commit (`git show 2cd6ee77:<path>`), not a later tree.
- **DB delta differs from 3 tables:** the 3-table result holds only in the **default config** (`EVENT_WEBHOOK` unset, seeded single-mailbox non-partner user). Setting `EVENT_WEBHOOK` or using a partner user adds `sync_event`/`users` writes (documented as caveats).
- **`daily_metric` shows INSERT vs UPDATE:** the day's row is INSERTed on the first alias of a calendar day and UPDATEd (`nb_alias += 1`) thereafter — both are correct.
- **Free-plan cap while testing:** `john@wick.com` is premium (cap does not apply); reproduce the cap with a separate runtime-only free user (as the document does).
- **Pre-existing test errors (`google-re2`/`pyre2`, `test_apple`):** out-of-scope and expected; do **not** attempt to fix them here — that would violate the read-only mandate.

---

## 10. Appendices

### A. Command Reference

| Command | Purpose |
|---|---|
| `poetry sync` | Install Python dependencies (from `poetry.lock`). |
| `cd static && npm install` | Build/install frontend assets. |
| `cp example.env .env` | Create local runtime config from template. |
| `docker run … postgres:13` | Start PostgreSQL 13. |
| `alembic upgrade head` | Apply DB schema to head. |
| `flask dummy-data` | Seed dev data (creates `john@wick.com`). |
| `python3 server.py` | Serve the webapp on :7777 (dev). |
| `gunicorn wsgi:app` | Serve via WSGI (as run in the container). |
| `python job_runner.py` | Run the async job worker. |
| `python event_listener.py` | Run the Postgres LISTEN/NOTIFY consumer. |
| `git diff --name-status 2cd6ee77 HEAD` | Confirm single-file change. |
| `git status --porcelain` | Confirm clean working tree. |

### B. Port Reference

| Port | Service | Notes |
|---|---|---|
| 7777 | SimpleLogin webapp | Web + API entry points (`server.py:L588`). |
| 5432 / 15432 | PostgreSQL 13 | Container 5432 mapped to host 15432 (per `CONTRIBUTING.md:L100`). |
| 6379 | Redis 7 | Rate-limit storage; optional in dev. |

### C. Key File Locations

| Path | Role |
|---|---|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The deliverable** (answer document). |
| `server.py` / `wsgi.py` | App factory & WSGI entry. |
| `app/dashboard/views/index.py` | Web alias-creation handler (302). |
| `templates/dashboard/index.html` | "Random Alias" form (frontend request). |
| `app/api/views/new_random_alias.py` | API alias-creation endpoint (201). |
| `app/api/serializer.py` | `serialize_alias_info_v2` (JSON payload). |
| `app/models.py` | `Alias.create` core; `DailyMetric`; `AliasAuditLog`. |
| `app/events/event_dispatcher.py` | Event dispatch short-circuit. |
| `app/alias_audit_log_utils.py` | `emit_alias_audit_log` (audit INSERT). |
| `app/config.py` | `MAX_NB_EMAIL_FREE_PLAN`, `EVENT_WEBHOOK` defaults. |
| `app/fake_data.py` | Seeds `john@wick.com` / `password`. |
| `tests/api/test_new_random_alias.py` / `test_new_custom_alias.py` | Alias-creation unit tests (16). |

### D. Technology Versions

| Component | Version |
|---|---|
| Python | 3.10.18 (canonical 3.10) |
| Flask | 1.1.2 |
| SQLAlchemy | 1.3.24 |
| Alembic | 1.4.3 |
| gunicorn | 20.0.4 |
| PostgreSQL | 13 (13.23 observed) |
| Redis | 7 |
| Node.js | 10.17.0 |

### E. Environment Variable Reference

| Variable | Default (dev) | Effect on alias creation |
|---|---|---|
| `DB_URI` | `postgresql://…:15432/simplelogin` | Datastore connection. |
| `EVENT_WEBHOOK` | *unset* | Unset → event dispatch short-circuits; **no** `sync_event`/`NOTIFY`. |
| `MEM_STORE_URI` | *unset* | Unset → in-memory rate limiter (per-worker buckets). |
| `NOT_SEND_EMAIL` | `true` | No real email transmitted. |
| `MAX_NB_EMAIL_FREE_PLAN` | `5` | Free-plan alias cap (error condition exercised). |
| `EMAIL_DOMAIN` | `sl.local` | Domain for generated aliases. |

### F. Developer Tools Guide

| Tool | Use in this project |
|---|---|
| `curl` | Drive the real HTTP entry points (web 302 / API 201) and error paths. |
| `psql` / SELECT-only probe | Snapshot per-table row counts before/after creation (read-only). |
| `git` | Verify read-only integrity (single-file diff, clean tree) and citation base commit. |
| `docker` | Run the canonical runtime (app, Postgres 13, Redis 7). |
| `pytest` | Execute the alias-creation unit suites (16 tests). |

### G. Glossary

| Term | Meaning |
|---|---|
| **AAP** | Agent Action Plan — the authoritative requirements for this task. |
| **Creation core** | The shared `Alias.create` path both entry points converge on. |
| **3-table core** | The three tables written on every default creation: `alias`, `daily_metric`, `alias_audit_log`. |
| **Short-circuit** | Early return in `EventDispatcher.send_event` when `EVENT_WEBHOOK` is unset — no event persisted/emitted. |
| **Default config** | Seeded, non-partner, single-mailbox setup with `EVENT_WEBHOOK`/`MEM_STORE_URI` unset. |
| **Path-to-production** | For this deliverable: human review/acceptance + merge (no deployment applies). |
| **`[inferred]`** | Label for statements established by reading rather than running. |

---

### Cross-Section Integrity — Final Validation

| Rule | Check | Result |
|---|---|---|
| **Rule 1** (§1.2 ↔ §2.2 ↔ §7) | Remaining hours identical | 2.0 = 2.0 = 2.0 ✅ |
| **Rule 2** (§2.1 + §2.2 = Total) | 42.0 + 2.0 = 44.0 (§1.2 Total) | ✅ |
| **Rule 3** (§3) | All tests from Blitzy autonomous validation logs | ✅ |
| **Rule 4** (§1.5) | Access issues validated (none) | ✅ |
| **Rule 5** (Colors) | Completed = `#5B39F3`, Remaining = `#FFFFFF` | ✅ |
| Completion % consistency | 42/44 = 95.5% in §1.2, §7, §8 | ✅ |

*Percent complete: **95.5%** · Completed: **42h** · Remaining: **2h** · Total: **44h**.*