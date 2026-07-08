# Blitzy Project Guide — Intermittent "Reply Routed to the Wrong User" Diagnostic Investigation

> **Repository:** SimpleLogin (`app`) · **Source branch:** `app_2cd6ee777f8c` @ `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c` · **Working branch:** `blitzy-21ffb38f-7c02-40ed-ae41-d6a514f6966c`
> **Task type:** Read-only, runtime-grounded root-cause investigation (rule set **SWE-AtlasQnA-Repo**) · **Brand palette:** Completed = Dark Blue `#5B39F3` · Remaining = White `#FFFFFF` · Headings/Accents = Violet-Black `#B23AF2` · Highlight = Mint `#A8FDD9`

---

## 1. Executive Summary

### 1.1 Project Overview

This project is a **read-only diagnostic investigation** of SimpleLogin's alias **reply-handling** pipeline. A user reported that some inbound replies — forwarded through an alias — are intermittently "routed to the wrong user," even though logs show the alias being recognized. The mandate was to bring up the canonical stack, drive a real inbound reply through the true aiosmtpd entry point, trace the alias→user resolution end-to-end, reproduce the intermittency by replaying identical input, and name the most-likely origin — all while leaving the repository byte-for-byte unchanged. The sole durable deliverable is one runtime-grounded Markdown answer document. Target audience: SimpleLogin backend and security engineers who must validate the finding and plan remediation.

### 1.2 Completion Status

The completion percentage is computed with the **PA1 AAP-scoped methodology**: only work defined by the Agent Action Plan (AAP) plus the path-to-production activities required to *deploy* the diagnostic deliverable (human review/verification/sign-off) are counted. Remediation of the diagnosed defect is **explicitly out of AAP scope** and is therefore excluded from the denominator (documented separately as a recommended follow-on).

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieTitleTextSize':'18px','pieLegendTextColor':'#B23AF2'}}}%%
pie showData title Completion — 86.0% (43 of 50 hrs)
    "Completed Work (AI)" : 43
    "Remaining Work" : 7
```

| Metric | Hours |
|--------|------:|
| **Total Hours** | **50** |
| Completed Hours — AI (autonomous) | 43 |
| Completed Hours — Manual (human) | 0 |
| **Completed Hours (AI + Manual)** | **43** |
| **Remaining Hours** | **7** |
| **Percent Complete** | **86.0 %** |

> **Calculation:** `Completion % = Completed / (Completed + Remaining) = 43 / (43 + 7) = 43 / 50 = 86.0 %`.
> All 15 AAP-scoped autonomous requirements are **Completed**; the 14 % remaining is purely the human path-to-production (independent review, reference verification, security sign-off, environment-independent re-run) required before a confidentiality-sensitive finding can be trusted and acted upon.

### 1.3 Key Accomplishments

- ✅ **Canonical runtime stood up and verified** — Python 3.10.18, PostgreSQL 13.23, Redis 6.2.22; schema at Alembic head `32f25cbf12f6`; every dependency pin matches `poetry.lock` exactly.
- ✅ **Real entry point exercised two ways** — a direct call to the production `email_handler.handle()` **and** a full aiosmtpd `Controller(MailHandler())` socket drive; both reached `handle_reply()`, wrote an `EmailLog`, and returned `250 Message accepted for delivery`.
- ✅ **Intermittency reproduced live** — the identical inbound reply routed to **user 829** in one DB/plan state and to **user 830 (the wrong user)** in another, stable across ≥2 runs and fully reversible; both a physical-order flip (MVCC) and a query-plan flip (seq-scan vs index-scan) were demonstrated.
- ✅ **Root cause identified with cause→effect reasoning** — `Contact.get_by(reply_email=…).first()` (`email_handler.py:986`), a `filter_by(...).first()` with **no `ORDER BY`** (`app/models.py:83-84`), over a **non-unique** `reply_email` column (`app/models.py:1899`; migration `:22`). Classified as a **confidentiality / tenant-isolation defect**.
- ✅ **Every condition exercised** — 10 decision branches (E200/E201/E214/E501/E502/E503/E504/E506) plus both sides of the unknown-mailbox fallback and the normalization-collapse behavior, each with complete captured output; non-default branches honestly labeled **NON-CANONICAL**.
- ✅ **All 8 explicit asks answered by name** in a 1,398-line / 11,391-word document with 36 documented command invocations and a consolidated `file:line` appendix.
- ✅ **Read-only mandate honored** — `git diff` vs source shows a single added file; 15 key source/test/config/migration files verified byte-identical; the diagnosed defect was deliberately **not** remediated.

### 1.4 Critical Unresolved Issues

There are **no unresolved issues in the deliverable itself** — it is complete, accurate, runtime-grounded, and committed. The items below concern the **subject** of the diagnosis (a live defect intentionally left unremediated per the read-only mandate) and the human validation still required.

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| Diagnosed confidentiality defect remains unremediated (by design — fix is out of AAP scope) | A reply can be relayed to another user's contact and logged under another user's `user_id` | Backend / Security | Follow-on task |
| Independent human re-reproduction of the finding not yet performed | Finding not yet peer-confirmed on a reviewer's own fixtures | Backend reviewer | 3 h |
| Production data not yet audited for existing duplicate `reply_email` rows | Blast radius in production is unknown | SRE / Backend | Follow-on task |

### 1.5 Access Issues

**No access issues identified.** The canonical Docker stack (`sl-app`, `sl-test-db`, `sl-redis`), the repository, the local PostgreSQL/Redis services, and the pinned virtualenv were all fully accessible and functional throughout the investigation. No external credentials, third-party APIs, or repository permissions blocked any step.

| System / Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-------------------|----------------|-------------------|-------------------|-------|
| SimpleLogin repository (git) | Read/Write | None | ✅ No issue | — |
| Canonical Docker stack (`sl-app`/`sl-test-db`/`sl-redis`) | Container exec | None | ✅ No issue | — |
| PostgreSQL 13 (`localhost:15432`) / Redis 6 | Service | None | ✅ No issue | — |

### 1.6 Recommended Next Steps

1. **[High]** Independently review and re-reproduce the finding — replay the identical input, apply the physical-order flip and `SET enable_seqscan = off`, and confirm the correct⇄wrong-user flip (see §9).
2. **[High]** Verify every `file:line` reference against source commit `2cd6ee77` to confirm no line drift, then formally accept the diagnosis.
3. **[High]** *(Follow-on, out of AAP scope)* Audit production for duplicate `reply_email` rows: `SELECT reply_email, count(*) FROM contact GROUP BY reply_email HAVING count(*) > 1;`.
4. **[Medium]** *(Follow-on, out of AAP scope)* Plan remediation — a DB `UniqueConstraint` on `Contact.reply_email` plus a dedupe data-migration, and/or a deterministic `ORDER BY` tie-break, plus making `available_sl_email()` atomic.
5. **[Medium]** Route the confidentiality classification to code owners / security and confirm the reproduction on a second clean stack or CI.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

Every completed component traces to a specific AAP requirement. All 43 hours were delivered **autonomously (AI)**; manual hours = 0.

| Component | Hours | Description |
|-----------|------:|-------------|
| Canonical environment bring-up & pin verification | 5 | Python 3.10.18 stack, PostgreSQL 13, Redis 6, `alembic upgrade head` → `32f25cbf12f6`, DKIM key, dependency pins verified against `poetry.lock` (AAP env prerequisite). |
| Fixture graph design & seeding | 3 | Two independent identity graphs `(User, Mailbox, Alias, Contact)` sharing **one** `reply_email`, recreating the precondition the schema permits (AAP §0.5.1). |
| Real entry-point drive harness | 5 | Drive 1 — direct `email_handler.handle()`; Drive 2 — full aiosmtpd `Controller(MailHandler())` socket, matching `main()` at `:2383` (AAP asks a, b; real-entry-point rule). |
| Resolution-chain instrumentation & capture | 4 | BEFORE/DURING/AFTER capture of `contact.id`, `alias_id`, `user_id`, `mailbox.id`, `EmailLog`, and recipient (AAP asks c, d; observe-state rule). |
| Intermittency reproduction | 6 | N=30 identical-input replays, MVCC physical-order flip + flip-back, seq-scan vs index-scan plan analysis, ≥2-run stability (AAP ask e; reproduce-inconsistency rule). |
| Edge-branch & decision-code coverage | 5 | 10 branches (E200/E201/E214/E501/E502/E503/E504/E506), both unknown-mailbox sides, normalization-collapse parity with `tests/test_email_utils.py:594-596` (AAP ask f; every-condition rule). |
| Root-cause analysis & cause→effect reasoning | 4 | Single-sourced-identity argument, confidentiality classification, contributing factors (advisory guard + normalization) (AAP ask g). |
| Diagnostic document authoring | 9 | 1,398 lines / 11,391 words; all 8 asks answered by name; 36 command invocations with complete unedited output; appendix `file:line` index; coverage pass (AAP deliverable). |
| Read-only cleanup & verification | 2 | Removed all `/tmp` scripts, deleted seeded rows via FK cascade, confirmed the repository byte-for-byte unchanged via `git status` (AAP ask h; read-only rule). |
| **Total Completed** | **43** | **Matches Completed Hours in §1.2.** |

### 2.2 Remaining Work Detail

Each remaining item is a **path-to-production** activity required to deploy (validate, accept, and route) the diagnostic deliverable. These are inherently human activities.

| Category | Hours | Priority |
|----------|------:|----------|
| Independent human review & re-reproduction of the confidentiality finding (replay loop + order/plan flip; confirm 829⇄830 mis-routing) | 3.0 | High |
| Verify all `file:line` references against source commit `2cd6ee77` (confirm zero drift) | 1.5 | High |
| Security triage & formal sign-off of the confidentiality classification; route to code owners | 1.5 | Medium |
| Confirm reproduction is environment-independent (second clean stack / CI run) | 1.0 | Medium |
| **Total Remaining** | **7.0** | **Matches Remaining Hours in §1.2 and §7 pie.** |

> **Out-of-scope remediation follow-on (indicative only — NOT counted in the 50 h total or the completion %).** The AAP explicitly forbids implementing the fix (§0.3.2). These are listed for stakeholder planning: production duplicate-`reply_email` audit (~2 h); add DB `UniqueConstraint` + dedupe migration (~4–6 h); deterministic `ORDER BY` tie-break / atomic `available_sl_email()` (~3 h); regression test for 1:1 `reply_email`→`(alias, contact)` (~3 h); operational invariant/alert on resolved `user_id` (~2 h).

### 2.3 Hours Basis & Confidence

- **Basis:** hours reflect the diagnostic engineering effort (environment, harness, reproduction, analysis, authoring) sized from the 1,398-line evidence-dense deliverable, the 36 documented command invocations, and the reproduction complexity.
- **Confidence:** **High** for completed work (all deliverables verified present and accurate) and for the remaining human-review estimates (well-defined activities). The out-of-scope remediation ranges carry **Medium** confidence pending the production data audit.

---

## 3. Test Results

All tests below originate from **Blitzy's autonomous validation execution** on the canonical stack (`docker exec sl-app … pytest`, `CONFIG=tests/test.env GITHUB_ACTIONS_TEST=true`), re-run and confirmed during this assessment. The task is read-only (no test files added or modified); these are the repository's own reply-phase tests plus the runtime-reproduction and static checks executed by the validation systems.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|------------:|-------:|-------:|-----------:|-------|
| Reply-phase unit/integration (`pytest -k reply`) | pytest 6.x | 6 | 6 | 0 | n/a (targeted) | Incl. `test_replace_contacts_and_user_in_reply_phase` (AAP-referenced two-contact fixture), `test_send_email_from_non_canonical_address_on_reply`, `test_email_sent_to_noreply`, `test_dmarc_reply_quarantine[REJECT/QUARANTINE/SOFTFAIL]`. |
| Runtime reproduction — real entry point | Custom driver via aiosmtpd | 2 drives | 2 | 0 | n/a | Drive 1 direct `handle()`; Drive 2 full `Controller` socket. Both reached `handle_reply`, wrote `EmailLog`, returned E200. |
| Intermittency distribution (identical input) | Custom replay loop | 4 runs (N≈30 each) | 4 | 0 | n/a | Baseline→user 829; post-flip→user 830 (wrong), stable ×2; flip-back→829. Deterministic per state, flips across states. |
| Decision-branch coverage | Custom branch driver | 10 branches | 10 | 0 | n/a | E200/E201/E214/E501/E502/E503/E504/E506; E201/E506 labeled NON-CANONICAL (require non-default flags). |
| Static — byte-compile of referenced modules | `python -m py_compile` | 7 modules | 7 | 0 | n/a | `email_handler.py`, `app/models.py`, `app/email_utils.py`, `app/email_validation.py`, `app/utils.py`, `app/contact_utils.py`, `app/config.py`. |

**Independent reproduction (validation gate):** the Final Validator's own ephemeral reproduction (fresh users 916/917, contacts sharing one `reply_email`) matched the deliverable's characterization 100 %. This assessment additionally confirmed, via read-only query, that **11 pre-existing test-suite contacts already share `reply_email = 'rep@sl.local'`** — corroborating that the duplicate-`reply_email` precondition arises naturally, without any agent-seeded fixtures.

---

## 4. Runtime Validation & UI Verification

**UI verification:** Not applicable — this is a backend SMTP/email-pipeline investigation with no user interface in scope. No Figma designs or UI components were provided.

**Runtime validation (all executed on the canonical stack):**

- ✅ **Operational — Canonical stack:** `sl-app` (Python 3.10.18), `sl-test-db` (PostgreSQL 13.23, port 15432), `sl-redis` (6.2.22) all `Up`.
- ✅ **Operational — Schema:** Alembic at head `32f25cbf12f6`; `ix_contact_reply_email` confirmed **non-unique** (`CREATE INDEX … btree (reply_email)`); the only unique constraint on `contact` is `uq_contact(alias_id, website_email)` — none on `reply_email`.
- ✅ **Operational — Real entry point:** `email_handler.handle()` and the full `Controller(MailHandler())` socket both drive `handle_DATA → _handle → handle → handle_reply`, create an `EmailLog`, and deliver to `contact.website_email`; observed status `250 Message accepted for delivery` (E200).
- ✅ **Operational — Alias→user resolution:** single-sourced from one `Contact` row — `Contact.get_by(reply_email=…)` (`:986`) → `alias = contact.alias` (`:994`) → `user = alias.user` (`:1004`) → mailbox authorization (`:1019`) → `EmailLog(user_id=contact.user_id)` (`:1046`) → recipient (`:1121`).
- ✅ **Operational — Intermittency:** identical input observed routing to user 829 and (after a benign write / plan change) to user 830, reproducibly and reversibly.
- ⚠ **Partial (by design) — API integrations:** DKIM signing, SPF/DMARC, and spam-assassin paths run under the default configuration; the SPF-enforcement (`E201`) and reply-spam (`E506`) branches are **inert in default config** and were demonstrated only with non-default flags — explicitly labeled **NON-CANONICAL** in the deliverable.
- ❌ **Failing:** none.

---

## 5. Compliance & Quality Review

Cross-mapping the AAP deliverables and the governing **SWE-AtlasQnA-Repo** rules to their observed status. Fixes applied during autonomous validation are noted; there are no outstanding compliance items.

| Benchmark / Rule | Requirement | Status | Evidence / Notes |
|------------------|-------------|--------|------------------|
| Deliverable location | `blitzy/documentation/app_2cd6ee777f8c.md` created | ✅ Pass | 1,398 lines; single added file. |
| Investigate by running first | Build & run before writing; capture real output | ✅ Pass | 36 command invocations with complete unedited output. |
| Real entry point only | aiosmtpd `MailHandler` path; label non-canonical | ✅ Pass | Direct `handle()` + full `Controller` socket; E201/E506 labeled NON-CANONICAL. |
| Reproduce run-to-run inconsistency | Same input repeated; report distribution; stable ≥2 runs | ✅ Pass | N=30 replays; 829⇄830 across states; Run2 + Run2b stability. |
| Default canonical configuration | `EMAIL_DOMAIN=sl.local`; state exact commands | ✅ Pass | `CONFIG=tests/test.env`; commands documented. |
| Exercise every condition | Happy path + edge/fallback/error branches | ✅ Pass | 10-branch decision matrix + normalization collapse. |
| Observe state before/during/after | Report values at each stage | ✅ Pass | BEFORE/DURING/AFTER captures in §(c)/§(d). |
| Actual, complete, unedited output | No paraphrasing of output/code | ✅ Pass | Verbatim captures inside code fences. |
| Be exact & grounded (`file:line`) | Name the function; cite `file:line` | ✅ Pass | 100 % of spot-checked refs accurate; appendix index. |
| Answer every part | All 8 asks + named sub-questions | ✅ Pass | Explicit coverage pass at the end of the document. |
| Read-only scope | No existing file modified; temp scripts removed | ✅ Pass | 15 key files byte-identical; clean `git status`. |
| Do not implement the fix | Diagnosis only, no remediation | ✅ Pass | No `UniqueConstraint`/`ORDER BY`/atomic guard added. |
| Code quality of deliverable | No placeholders/TODO/stubs in prose | ✅ Pass | The single "stub" mention describes a legitimate non-canonical test technique. |

**Fixes applied during autonomous validation:** a second commit (`d3a0d518`) addressed code-review findings on the first draft (`cc0866ab`), refining reference precision and honest characterization of the non-determinism. No compliance gaps remain.

---

## 6. Risk Assessment

Risks are categorized per PA3. The deliverable itself carries no defect risk; the significant risks concern the **diagnosed subject defect** (intentionally unremediated) and reproduction caveats.

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Cross-account reply disclosure: a reply relayed to another user's contact and logged under another user's `user_id` | Security | Critical | Low–Medium (needs duplicate `reply_email`) | Prioritize remediation (unique constraint + dedupe); audit production data | Diagnosed, unremediated (fix out of AAP scope) |
| Diagnosed defect remains unfixed per read-only mandate | Technical | High | Medium | Remediation follow-on: `UniqueConstraint` + `ORDER BY` tie-break + atomic guard | Open (by design) |
| `available_sl_email()` uniqueness guard is advisory-only (`app/models.py:1425`) — no DB constraint | Security | Medium | Medium | Add DB `UniqueConstraint`; make the guard atomic | Documented |
| No production audit for existing duplicate `reply_email` rows | Operational | Medium | Unknown | Run `GROUP BY reply_email HAVING count(*) > 1` in production | Recommended |
| Symptom is silent — logs show the alias "recognized"; mis-route raises no error | Operational | Medium | Medium | Add invariant/alert that resolved `contact.user_id` matches the expected recipient | Recommended |
| Reproduction depends on a specific physical-order/plan state | Technical | Low | Medium | Deliverable provides exact SQL to force the state (`UPDATE` flip, `SET enable_seqscan = off`) | Mitigated (documented) |
| Production topology (pooling/replicas/PG version) may change which row `first()` returns | Integration | Low | Low | Behavior is order-unspecified regardless; noted in deliverable | Mitigated |
| E201/E506 branches inert in default config; demonstrated with non-default flags | Integration | Low (informational) | n/a | Labeled NON-CANONICAL in deliverable | Documented |
| `file:line` references drift as source evolves | Process | Low | High over time | Exact source commit recorded; symbols named alongside line numbers | Mitigated |

---

## 7. Visual Project Status

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieTitleTextSize':'18px','pieLegendTextColor':'#B23AF2'}}}%%
pie showData title Project Hours Breakdown (Total 50 hrs)
    "Completed Work" : 43
    "Remaining Work" : 7
```

**Remaining hours by category (Section 2.2):**

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#5B39F3','primaryTextColor':'#B23AF2','lineColor':'#B23AF2'}}}%%
graph LR
    A["Human review & re-reproduction — 3.0 h (High)"]
    B["Verify file:line refs — 1.5 h (High)"]
    C["Security triage & sign-off — 1.5 h (Medium)"]
    D["Environment-independent re-run — 1.0 h (Medium)"]
```

> **Integrity check:** "Remaining Work" = **7 h** here equals the Remaining Hours in §1.2 and the sum of the §2.2 Hours column (3.0 + 1.5 + 1.5 + 1.0 = 7.0). "Completed Work" = **43 h** equals §1.2 Completed and the §2.1 total. `43 + 7 = 50` = Total.

---

## 8. Summary & Recommendations

**Achievements.** The investigation is **86.0 % complete** (43 of 50 hours). All 15 AAP-scoped autonomous requirements are delivered: the canonical stack was stood up, the real aiosmtpd entry point was driven two independent ways, the alias→user resolution was traced end-to-end, and the intermittent "wrong user" symptom was **reproduced live** (user 829 ⇄ user 830 under identical input). The most-likely origin is named with cause→effect precision: `Contact.get_by(reply_email=…).first()` at `email_handler.py:986` — an unordered `filter_by(...).first()` (`app/models.py:83-84`) over a **non-unique** `reply_email` column (`app/models.py:1899`) — from which every downstream identity (alias, user, authorizing mailbox, logged `user_id`, delivery recipient) is single-sourced, so a wrong-row pick mis-routes the entire reply. Correctly classified as a **confidentiality / tenant-isolation defect**.

**Remaining gaps (14 %).** Purely human path-to-production: independent re-reproduction, `file:line` verification against the source commit, security sign-off of the confidentiality classification, and an environment-independent re-run — **7 hours** total. No autonomous work remains.

**Critical path to production.** (1) Human peer review + re-reproduction → (2) reference verification + formal acceptance → (3) *(out-of-scope follow-on)* production duplicate-`reply_email` audit and remediation (DB `UniqueConstraint` + dedupe migration, deterministic tie-break, atomic guard) → (4) regression test + operational alert.

**Success metrics.** Deliverable present and committed (✅); read-only mandate honored — 15 key files byte-identical, clean working tree (✅); all 8 asks answered by name (✅); finding independently reproduced by validation (✅); repository reply-phase tests 6/6 passing (✅).

**Production readiness assessment.** The **deliverable is production-ready** — accurate, complete, runtime-grounded, and merge-ready. The **underlying application defect it diagnoses is not production-safe** and warrants a prioritized, separately-scoped remediation. This guide provides the exact reproduction and remediation path for that follow-on.

| Dimension | Status |
|-----------|--------|
| AAP-scoped completion | 86.0 % (43 / 50 h) |
| Deliverable quality | Production-ready |
| Read-only mandate | Honored (git tree clean; only the answer document added) |
| Blocking issues | None |

---

## 9. Development Guide

This guide reproduces the environment and the finding. All commands were **tested during this assessment** on the canonical stack and are copy-pasteable.

### 9.1 System Prerequisites

- **Docker Engine 28.x** with `docker compose` (the canonical stack runs as three containers).
- ~2 GB free disk for the canonical image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`.
- Host ports **15432** (PostgreSQL) and **6379** (Redis) available.
- **Do not** use the host's system Python (3.12.x) for observed values — it is **non-canonical**. Use the container's `/app/venv/bin/python` (Python 3.10.18) only.

### 9.2 Environment Setup

The canonical stack is three long-running containers with the host repository bind-mounted at `/app` inside `sl-app`:

```bash
# Verify the three canonical containers are running
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}'
# Expected: sl-app (…simple-login_app_1.0), sl-redis (redis:6, 6379), sl-test-db (postgres:13, 15432->5432)

# Confirm the canonical interpreter and configuration
docker exec sl-app bash -lc '/app/venv/bin/python --version'          # -> Python 3.10.18
grep -E 'EMAIL_DOMAIN|DB_URI' tests/test.env                          # EMAIL_DOMAIN=sl.local ; DB_URI=…@localhost:15432/test
```

> `CONFIG=tests/test.env` is required because `app/config.py` loads that env file at import time and selects the **default** configuration (`EMAIL_DOMAIN=sl.local`).

### 9.3 Dependency Verification

Dependencies are pre-installed in the container's isolated virtualenv and pinned to `poetry.lock`. Verify (do **not** reinstall):

```bash
docker exec sl-app bash -lc '/app/venv/bin/pip freeze | grep -iE "^(aiosmtpd|SQLAlchemy|psycopg2-binary|flanker|dkimpy|dnspython|email-validator|arrow|redis|Flask|alembic)=="'
# Expected pins: aiosmtpd==1.4.2  SQLAlchemy==1.3.24  psycopg2-binary==2.9.3  flanker==0.9.11
#                dkimpy==1.0.5  dnspython==2.0.0  email-validator==1.1.3  arrow==0.16.0
#                redis==4.6.0  Flask==1.1.2  alembic==1.4.3
```

### 9.4 Schema / Migration

```bash
# Confirm the schema is at head (apply only if a fresh DB is used)
docker exec sl-test-db psql -U test -d test -tc "SELECT version_num FROM alembic_version;"   # -> 32f25cbf12f6
# To (re)apply from scratch:
# docker exec sl-app bash -lc 'cd /app && CONFIG=tests/test.env /app/venv/bin/alembic upgrade head'
```

### 9.5 Verification Steps

```bash
# 1) Byte-compile the referenced modules (must print OK)
docker exec sl-app bash -lc 'cd /app && /app/venv/bin/python -m py_compile email_handler.py app/models.py app/email_validation.py && echo OK'

# 2) Run the repository's reply-phase tests (expected: 6 passed, 17 deselected)
docker exec sl-app bash -lc 'cd /app && CONFIG=tests/test.env GITHUB_ACTIONS_TEST=true /app/venv/bin/python -m pytest tests/test_email_handler.py -k reply -p no:cacheprovider -q'

# 3) Run the AAP-referenced two-contact reply test specifically
docker exec sl-app bash -lc 'cd /app && CONFIG=tests/test.env GITHUB_ACTIONS_TEST=true /app/venv/bin/python -m pytest tests/test_email_handler.py -k test_replace_contacts_and_user_in_reply_phase -p no:cacheprovider -q'
```

### 9.6 Example Usage — Observe the Precondition & Read the Finding

```bash
# Confirm the mis-routing PRECONDITION exists live (READ-ONLY): duplicate reply_email is permitted
docker exec sl-test-db psql -U test -d test -c \
  "SELECT reply_email, count(*) FROM contact GROUP BY reply_email HAVING count(*) > 1;"
# Observed live: rep@sl.local | 11  (pre-existing test-suite contacts share one reply_email)

# Inspect the exact resolution site and query helper cited by the diagnosis
sed -n '984,987p'   email_handler.py     # contact = Contact.get_by(reply_email=reply_email)
sed -n '82,85p'     app/models.py        # get_by = Session.query(cls).filter_by(**kw).first()
sed -n '1897,1900p' app/models.py        # reply_email = sa.Column(..., index=True)  (NON-UNIQUE)

# Read the diagnostic deliverable
sed -n '1,60p' blitzy/documentation/app_2cd6ee777f8c.md
```

*(Advanced) To reproduce the wrong-user flip, seed two `Contact` rows sharing one `reply_email`, drive `email_handler.handle()` with `envelope.rcpt_tos=[shared_reply_email]`, then flip physical order with a benign `UPDATE …` or `SET enable_seqscan = off` and re-run — see §(e) of the deliverable for the exact scripts and captured output. Any rows you seed must be removed afterward.*

### 9.7 Read-Only Confirmation

```bash
git status --short                                                   # clean tree expected
git status --porcelain -- . ':!blitzy/documentation/' | wc -l        # -> 0 (no non-doc changes)
git diff 2cd6ee77..HEAD --stat                                       # -> only blitzy/documentation/app_2cd6ee777f8c.md
```

> **Read-only invariant (precise):** the **git-tracked repository** (source/test/config/migration) is byte-for-byte unchanged; the only addition is the answer document. The local **test database is external state** (not tracked by git) and naturally accumulates ephemeral pytest fixtures from any test run — this does not affect the read-only mandate.

### 9.8 Troubleshooting

- **A container is not `Up`:** `docker start sl-test-db sl-redis sl-app`, then re-check `docker ps`.
- **`psql: connection refused`:** ensure host port `15432` maps to `sl-test-db:5432` (`docker port sl-test-db`).
- **Values look wrong / different Python:** you are likely using the host's system Python 3.12; always prefix with the container path `/app/venv/bin/python`.
- **`config` import error running a script:** you omitted `CONFIG=tests/test.env` — `app/config.py` requires it at import time.
- **Reference line doesn't match:** confirm you are on source commit `2cd6ee77`; line numbers are pinned to that commit and drift if the source changes.
- **Reproduction shows only one user:** the two contacts likely share the same physical order/plan — apply a benign `UPDATE` to flip MVCC order or `SET enable_seqscan = off` to change the plan.

---

## 10. Appendices

### Appendix A — Command Reference

| Purpose | Command |
|---------|---------|
| Stack status | `docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}'` |
| Canonical Python | `docker exec sl-app bash -lc '/app/venv/bin/python --version'` |
| Byte-compile | `docker exec sl-app bash -lc 'cd /app && /app/venv/bin/python -m py_compile email_handler.py app/models.py'` |
| Reply-phase tests | `docker exec sl-app bash -lc 'cd /app && CONFIG=tests/test.env GITHUB_ACTIONS_TEST=true /app/venv/bin/python -m pytest tests/test_email_handler.py -k reply -q'` |
| Alembic head | `docker exec sl-test-db psql -U test -d test -tc "SELECT version_num FROM alembic_version;"` |
| Duplicate-`reply_email` audit | `docker exec sl-test-db psql -U test -d test -c "SELECT reply_email, count(*) FROM contact GROUP BY reply_email HAVING count(*) > 1;"` |
| Read-only check | `git status --porcelain -- . ':!blitzy/documentation/' \| wc -l` |

### Appendix B — Port Reference

| Service | Container | Host Port | Container Port |
|---------|-----------|----------:|---------------:|
| PostgreSQL 13 | `sl-test-db` | 15432 | 5432 |
| Redis 6 | `sl-redis` | 6379 | 6379 |
| aiosmtpd (ad-hoc reproduction) | `sl-app` | ephemeral (e.g. 20399) | — |

### Appendix C — Key File Locations

| Item | Path |
|------|------|
| **Deliverable** | `blitzy/documentation/app_2cd6ee777f8c.md` |
| SMTP entry point, router, reply handler | `email_handler.py` (`handle_DATA:2289`, `_handle:2335`, `handle:1945`, `handle_reply:966`, resolution `:986`, `EmailLog:1042-1050`, recipient `:1121`) |
| Generic `get_by` + models | `app/models.py` (`get_by:83-84`, `reply_email:1899`, `available_sl_email:1425`, `uq_contact:1874-1876`) |
| Reverse-alias generation / recognition | `app/email_utils.py` (`generate_reply_email:1103`, `is_reverse_alias:1156`) |
| Reply-address normalization | `app/email_validation.py` (`normalize_reply_email:25`) |
| Non-unique index origin | `migrations/versions/2021_071310_78403c7b8089_.py:22` |
| Decision codes | `app/email/status.py` |
| Reference reply test / fixtures | `tests/test_email_handler.py:274`, `tests/conftest.py`, `tests/test.env` |

### Appendix D — Technology Versions

| Component | Version |
|-----------|---------|
| Python (canonical) | 3.10.18 |
| PostgreSQL | 13.23 |
| Redis | 6.2.22 |
| SQLAlchemy | 1.3.24 |
| aiosmtpd | 1.4.2 |
| Flask | 1.1.2 |
| psycopg2-binary | 2.9.3 |
| flanker / dkimpy / dnspython | 0.9.11 / 1.0.5 / 2.0.0 |
| email-validator / arrow / redis (client) | 1.1.3 / 0.16.0 / 4.6.0 |
| alembic | 1.4.3 |
| Alembic schema head | `32f25cbf12f6` |

### Appendix E — Environment Variable Reference

| Variable | Value / Purpose |
|----------|-----------------|
| `CONFIG` | `tests/test.env` — required; `app/config.py` loads it at import time |
| `EMAIL_DOMAIN` | `sl.local` (default canonical configuration) |
| `DB_URI` | `postgresql://test:test@localhost:15432/test` |
| `GITHUB_ACTIONS_TEST` | `true` — CI test-mode flag used for the pytest runs |

### Appendix F — Developer Tools Guide

- **pytest** — repository reply-phase tests; use `-k reply -p no:cacheprovider -q` (non-watch, deterministic collection).
- **psql** — read-only inspection of the precondition and schema; use `-tc` for scalar queries.
- **python -m py_compile** — static byte-compile check of referenced modules (no execution side effects).
- **git diff / status** — authoritative read-only verification against source commit `2cd6ee77`.

### Appendix G — Glossary

| Term | Meaning |
|------|---------|
| Reverse-alias (`reply_email`) | Per-`(alias, contact)` address a user replies to; designed to map **1:1** so mailboxes/correspondents stay private. |
| `handle_reply()` | The reply-phase handler in `email_handler.py:966` that resolves the contact and relays the reply. |
| `get_by(...).first()` | Generic `ModelMixin` helper (`app/models.py:83-84`): `filter_by(**kw).first()` — **no `ORDER BY`**, so the DB may return either matching row. |
| Order-unspecified | Deterministic within a fixed physical order + query plan, but may flip across writes (MVCC), inserts, `VACUUM`, or plan changes — the mechanism behind the intermittency. |
| Single-sourced identity | Alias, user, mailbox, logged `user_id`, and recipient are all derived from the one `Contact` row at `:986`; a wrong row mis-routes the entire reply. |
| NON-CANONICAL | A value obtained only via non-default flags/stubs (e.g., `ENFORCE_SPF`, `ENABLE_SPAM_ASSASSIN`); labeled as such per the canonical-configuration rule. |