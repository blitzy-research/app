# Blitzy Project Guide — SimpleLogin Reply-Pipeline Mis-Routing Investigation

## 1. Executive Summary

### 1.1 Project Overview

This engagement is a **code-as-truth diagnostic investigation** of SimpleLogin (a self-hosted email-aliasing service built on Python/Flask with an `aiosmtpd` inbound SMTP pipeline). Governed by rule **SWE-AtlasQnA-Repo**, the objective was to produce a single, comprehensive Markdown deliverable — `blitzy/documentation/app_2cd6ee777f8c.md` — that authoritatively explains how the inbound reply pipeline derives a reply-email, resolves it to a `Contact`, and selects the forwarding user/alias, and **why a reply can silently reach the wrong user** (race conditions, uniqueness assumptions, timing). The audience is SimpleLogin maintainers and security reviewers. No product code is shipped; the source tree is left byte-for-byte unchanged.

### 1.2 Completion Status

The project is **90.9% complete** (AAP-scoped). All mandated analysis, build-and-run observation, and the durable deliverable are complete and independently verified; the only remaining work is the path-to-production human review/sign-off appropriate for a diagnostic document.

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieTitleTextSize':'16px','pieSectionTextColor':'#B23AF2'}}}%%
pie showData title Completion Status — 90.9% Complete (40h of 44h)
    "Completed Work" : 40
    "Remaining Work" : 4
```

| Metric | Hours |
|--------|-------|
| **Total Hours** | 44.0 |
| **Completed Hours (AI + Manual)** | 40.0 (AI: 40.0 + Manual: 0.0) |
| **Remaining Hours** | 4.0 |
| **Percent Complete** | **90.9%** |

> Calculation (PA1, AAP-scoped): `Completion % = Completed / (Completed + Remaining) = 40 / (40 + 4) = 40 / 44 = 90.9%`.

### 1.3 Key Accomplishments

- ✅ Provisioned a faithful runtime (Python 3.10, PostgreSQL 13, Redis 6) and applied Alembic migrations to **build and run** the system for live observation.
- ✅ Traced and documented the full **reply-resolution path**: `handle()` → `is_reverse_alias()` → `handle_reply()` → `Contact.get_by(reply_email=…)` → `contact.alias.user` (deliverable §2–§4).
- ✅ Traced and documented the **generation path** and its TOCTOU window: `create_contact()` → `generate_reply_email()` → lock-free `available_sl_email()` (deliverable §5).
- ✅ Established **database-constraint ground truth**: `reply_email` is indexed but **not unique** at any layer — model, both migrations, and live PostgreSQL introspection (deliverable §6).
- ✅ Reproduced **runtime evidence A–D** on an ephemeral database: duplicate `reply_email` persists with no `IntegrityError`; `get_by().first()` emits no `ORDER BY` and silently returns one arbitrary row; deleting the resolved row flips the destination user; concurrent generation passes the lock-free check and double-inserts (deliverable §8).
- ✅ Synthesized a **6-link, code-confirmed root-cause chain** plus a Mermaid diagram and candidate remediations (analysis only) (deliverable §7, §9).
- ✅ Authored the **533-line deliverable** with a 25-row verified locator index (Appendix B); independently verified ~60 cited locators against source — all accurate.
- ✅ Guaranteed **scope isolation**: `git diff` vs base = exactly one added file; working tree clean; all instrumentation lived in `/tmp` and was deleted.

### 1.4 Critical Unresolved Issues

There are **no unresolved issues in the work product**. The deliverable compiles/validates cleanly, all autonomous tests pass, and the source repository is provably unchanged. The single item awaiting a human decision is the disposition of the **product defect the analysis surfaces** (whose remediation is explicitly out-of-scope for this engagement).

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| Product defect surfaced by analysis: silent reply mis-routing to a different user when two Contacts share one `reply_email` | Potential disclosure of reply content to the wrong user; **diagnosis complete, fix is out-of-scope** by AAP | Human SME / Product owner | Pending disposition (see §1.6) |

### 1.5 Access Issues

**No access issues identified.** All work was performed within the provided repository and an ephemeral, self-provisioned PostgreSQL 13 + Redis 6 stack. No external credentials, third-party APIs, or restricted resources were required.

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-----------------|----------------|-------------------|-------------------|-------|
| Source repository | Read/Write (branch) | None — full access | ✅ Resolved | — |
| Ephemeral PostgreSQL 13 / Redis 6 | Local provisioning | None — self-provisioned & torn down | ✅ Resolved | — |

### 1.6 Recommended Next Steps

1. **[High]** Conduct an SME technical review of `blitzy/documentation/app_2cd6ee777f8c.md` — validate the 6-link root-cause chain and spot-check a sample of the Appendix B locators against the source branch (≈2.0h).
2. **[Medium]** Make a formal disposition decision — accept the diagnosis, confirm the no-source-modification guarantee, and decide whether/when to schedule the (out-of-scope) remediation as separate future work (≈1.0h).
3. **[Low]** *(Optional)* File a remediation tracking ticket capturing the documented candidate fixes (DB `UNIQUE` on `reply_email` + race-safe get-or-create; `one_or_none()`; inbound concurrency lock) for a future, separately-scoped change (≈1.0h).

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

All completed work traces directly to AAP requirements (R1–R6), the mandated deliverable, build-and-run provisioning, isolation, and validation.

| Component | Hours | Description |
|-----------|-------|-------------|
| Environment provisioning / build-and-run | 5.0 | Python 3.10 venv + Poetry-locked deps; ephemeral PostgreSQL 13 + Redis 6; Alembic `upgrade head`; SimpleLogin env vars (AAP build-and-run prerequisite) |
| R1 — Reply-email derivation (§2) | 3.0 | Trace `handle()` routing, `is_reverse_alias()`, `handle_reply()`, `reply_email = rcpt_to`, normalization |
| R2 — Contact resolution (§3) | 2.5 | Document `Contact.get_by()` = `filter_by().first()` (no `ORDER BY`, no raise on multiplicity) |
| R3 — Destination selection (§4) | 2.0 | Document `alias = contact.alias` → `user = alias.user`; `EmailLog` creation |
| R4 — Reply-email generation / TOCTOU (§5) | 4.0 | Document `create_contact()`, `generate_reply_email()` check-then-act, lock-free `available_sl_email()`, `IntegrityError` recovery scope, entropy nuance |
| R5 — Constraint ground truth (§6) | 3.0 | Confirm `uq_contact` only; `reply_email` index-not-unique; both migrations; live PG introspection |
| R6 — Empirical observation harness (§8) | 6.0 | Build `/tmp` instrumentation; reproduce Evidence A–D across repeated/concurrent reply events |
| Synthesis — root-cause chain & rationale (§7, §9) | 3.0 | 6-link code-confirmed chain + Mermaid; conditions; candidate remediations (analysis only) |
| Deliverable authoring | 4.0 | Write/structure the 533-line Markdown; Appendix A methodology; Appendix B locator index |
| Background research | 1.5 | Reverse-alias semantics; race-safe get-or-create & uniqueness patterns in SQLAlchemy |
| Final validation pass | 6.0 | Verify ~60 locators; run 639-test suite; reproduce runtime evidence; 1 precision fix; prove source unchanged |
| **Total Completed** | **40.0** | |

### 2.2 Remaining Work Detail

Remaining work is exclusively the **path-to-production human gate** for a diagnostic document. Remediation implementation is **excluded** (explicitly out-of-scope per AAP §0.3.2).

| Category | Hours | Priority |
|----------|-------|----------|
| SME technical review of deliverable (accuracy & completeness; spot-check locators; validate root-cause chain) | 2.0 | High |
| Disposition decision (accept diagnosis; confirm source unchanged; decide remediation as future work) | 1.0 | Medium |
| *(Optional)* File remediation tracking ticket for documented candidate fixes | 1.0 | Low |
| **Total Remaining** | **4.0** | |

### 2.3 Total Project Hours

| Bucket | Hours |
|--------|-------|
| Completed (Section 2.1) | 40.0 |
| Remaining (Section 2.2) | 4.0 |
| **Total Project Hours** | **44.0** |

> Integrity: `2.1 (40.0) + 2.2 (4.0) = 44.0` = Total in Section 1.2 ✓.

---

## 3. Test Results

All tests below originate from **Blitzy's autonomous validation logs** for this project. The suite was executed with `CONFIG=tests/test.env GITHUB_ACTIONS_TEST=true poetry run pytest` against the provisioned PostgreSQL 13 + Redis 6 stack. Because this is a documentation-only engagement, no new tests were authored; the **full existing SimpleLogin suite** was run as a regression gate to prove the source remains correct and unchanged.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|------------|-------|
| Full regression suite (unit + integration) | pytest | 639 | 639 | 0 | N/A (not measured) | Baseline-matching; 51 benign warnings; reply/forward handler, contact-utils, email-utils, models tests all green |
| Byte-compile / import sanity | `py_compile` | 7 modules | 7 | 0 | — | `email_handler.py`, `app/models.py`, `app/email_utils.py`, `app/contact_utils.py`, `app/email_validation.py`, `app/utils.py`, `app/parallel_limiter.py` all OK |
| Runtime evidence reproduction (R6) | Custom `/tmp` harness | 4 evidence sets (A–D) | 4 | 0 | — | Reproduced on ephemeral DB: dup `reply_email` persists; `.first()` no `ORDER BY`; delete flips destination user; TOCTOU double-insert |

**Aggregate:** 639/639 automated tests passed (100% pass rate), 0 failed. Test types: unit + integration (pytest), static byte-compile, and behavioral runtime reproduction.

---

## 4. Runtime Validation & UI Verification

**UI Verification: Not Applicable.** This engagement ships **no UI and no product code** — the sole artifact is a Markdown analysis document. There is no front-end surface to verify.

**Runtime & integration validation (from autonomous logs):**

- ✅ **Operational** — Dependency layer: all Poetry-locked dependencies install and all 7 key modules import cleanly under `CONFIG=tests/test.env`.
- ✅ **Operational** — Datastores: PostgreSQL 13.23 reachable (Alembic head `32f25cbf12f6`); Redis 6 reachable (`PONG`).
- ✅ **Operational** — Migrations: ephemeral throwaway DB migrated via `alembic upgrade head`; schema introspection matches the deliverable's §6.3 (PK `forward_email_pkey`, unique `uq_contact`, **non-unique** `ix_contact_reply_email`, `reply_email varchar(128)`).
- ✅ **Operational** — Evidence A: two Contacts (different users/aliases) persisted with an identical `reply_email`, **no `IntegrityError`**.
- ✅ **Operational** — Evidence B/B2/B3: compiled `get_by` SQL has **no `ORDER BY`**; `.first()` returns silently while `.one()` raises `MultipleResultsFound`; deleting the resolved row flips the destination user (A→B).
- ✅ **Operational** — Evidence C: TOCTOU — both concurrent actors pass the lock-free `available_sl_email()` check and both INSERT the same `reply_email`.
- ✅ **Operational** — Evidence D: helper outputs match (`normalize_reply_email`, `is_reverse_alias`, entropy branches).
- ✅ **Operational** — Scope: `git diff` vs base = exactly `A blitzy/documentation/app_2cd6ee777f8c.md`; working tree clean; instrumentation removed.

---

## 5. Compliance & Quality Review

### 5.1 AAP Requirement Compliance Matrix

| AAP Requirement | Deliverable Section | Status | Progress |
|-----------------|--------------------|--------|----------|
| **R1** — Reply-email derivation & reverse-alias routing | §2 | ✅ Pass | 100% |
| **R2** — Contact resolution (`get_by().first()` single-vs-multiple) | §3 | ✅ Pass | 100% |
| **R3** — Destination selection (`contact.alias` → `alias.user`) | §4 | ✅ Pass | 100% |
| **R4** — Reply-email generation & TOCTOU window | §5 | ✅ Pass | 100% |
| **R5** — Constraint ground truth (no `reply_email` uniqueness) | §6 | ✅ Pass | 100% |
| **R6** — Empirical observation across multiple/concurrent events | §8 | ✅ Pass | 100% |
| Root-cause synthesis & rationale | §7, §9 | ✅ Pass | 100% |
| Deliverable created at mandated path | `blitzy/documentation/app_2cd6ee777f8c.md` | ✅ Pass | 100% |

### 5.2 Rule SWE-AtlasQnA-Repo Compliance Matrix

| Directive | Status | Evidence |
|-----------|--------|----------|
| Create `<source_branch_name>.md` answering the question comprehensively | ✅ Pass | `blitzy/documentation/app_2cd6ee777f8c.md` (533 lines) |
| Build and run the source to analyze behavior | ✅ Pass | Ephemeral PG13 + Redis6; migrations applied; 639-test suite run; Evidence A–D reproduced |
| Code-as-truth; no assumptions | ✅ Pass | ~60 locators verified against source; schema verified vs model **and** migration **and** live DB |
| Provide thinking / rationale | ✅ Pass | §7 root-cause chain + §9 rationale/conditions |
| Do not modify existing source files | ✅ Pass | `git diff` vs base = 1 added file; 0 source files changed |
| Add no other code to the source repo | ✅ Pass | All instrumentation in `/tmp`, deleted after use |
| Place document in `blitzy/documentation/` | ✅ Pass | Correct destination path |

### 5.3 Fixes Applied During Autonomous Validation

- **MINOR-1 (commit `080e6313`):** corrected a code-fence header line range from `L1042-1051` to `L1042-1050` for the `EmailLog.create(...)` statement, aligning it with the document's four other references. Net change `+1/-1` to the deliverable only. This was the single inaccuracy found across ~60 cited locators.

### 5.4 Outstanding Quality Items

None for the work product. The only outstanding item is human sign-off (Section 2.2).

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| **R-1** Silent reply mis-routing to a *different* user when two Contacts share one `reply_email` (potential disclosure) — the product defect the analysis documents | Security | High (product-level) | Low–Medium | Candidate remediations documented as **analysis only** (DB `UNIQUE` on `reply_email`; `one_or_none()`; inbound `parallel_limiter` lock; race-safe get-or-create) | Documented; remediation **out-of-scope** per AAP, deferred to human disposition |
| **R-2** Cited line-number drift if the source branch advances past `app_2cd6ee777f8c` | Technical | Low | Low | Branch pinned in doc; Appendix B 25-row locator index; ~60 locators re-verified | Mitigated |
| **R-3** Run-specific runtime values in Evidence A–D (random alias names/ids/local-part lengths) not byte-reproducible | Technical | Low | Medium (by design) | Doc labels values verbatim-from-its-own-run; the **behaviors** reproduce deterministically; Appendix A methodology | Accepted / Mitigated |
| **R-4** Schema divergence: live `reply_email` is `varchar(128)` vs model `String(512)`; no widening migration exists | Technical | Low | N/A (observed fact) | Explained in §6.2/§6.3/Appendix A; does **not** affect the uniqueness conclusion | Resolved / Explained in-doc |
| **R-5** Runtime evidence not independently re-runnable without PG13/Redis6/Py3.10 + migrations | Operational | Low | Medium | Section 9 development guide + Appendix A give exact reproduction steps | Mitigated |
| **R-6** Accidental source-tree modification would breach SWE-AtlasQnA-Repo | Operational / Scope | Medium | Very Low | `git diff` vs base = 1 file; working tree clean; `/tmp` instrumentation deleted | Verified clean |
| **R-7** Documented defect affects the inbound mail pipeline under high-volume concurrency | Integration | Medium (product) | Low | Conditions documented in §9.2; remediation out-of-scope | Documented; deferred |

> Note: This engagement ships no product code, so deployment/integration risks for the **deliverable** are negligible. The High-severity item (R-1) is the **product defect the analysis surfaces** — its remediation is explicitly out-of-scope; no High-severity risk is attributable to the deliverable itself.

---

## 7. Visual Project Status

### 7.1 Project Hours Breakdown

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieTitleTextSize':'16px','pieSectionTextColor':'#B23AF2'}}}%%
pie showData title Project Hours — 40h Completed / 4h Remaining (90.9% Complete)
    "Completed Work" : 40
    "Remaining Work" : 4
```

### 7.2 Remaining Work by Priority (hours from Section 2.2)

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#A8FDD9','pie3':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieSectionTextColor':'#B23AF2'}}}%%
pie showData title Remaining Work by Priority (4h total)
    "High — SME review" : 2
    "Medium — Disposition decision" : 1
    "Low — Optional ticket" : 1
```

> Integrity: Section 7 "Remaining Work" (4) = Section 1.2 Remaining (4.0) = Section 2.2 total (4.0) ✓. "Completed Work" (40) = Section 1.2 Completed (40.0) = Section 2.1 total (40.0) ✓.

---

## 8. Summary & Recommendations

**Achievements.** The engagement delivered the single mandated artifact — a 533-line, code-grounded analysis that answers all six requirements (R1–R6) with cited locators and live runtime evidence. The system was built and run; PostgreSQL 13 + Redis 6 were provisioned; the full 639-test suite passed; and Evidence A–D were reproduced on an ephemeral database. The source repository is provably unchanged (`git diff` vs base = exactly one added file).

**Completion.** The project is **90.9% complete** (40.0h of 44.0h, AAP-scoped). The remaining 4.0h is exclusively the path-to-production human review/sign-off appropriate for a diagnostic deliverable.

**Critical path to production.** (1) SME technical review of the deliverable; (2) formal disposition decision on the surfaced defect; (3) optional remediation tracking ticket. None of these block the *deliverable itself*, which is complete and verified.

**Remaining gaps.** No gaps exist in the work product. The one substantive open question is a **product decision** — whether and when to remediate the documented mis-routing defect — which the AAP deliberately placed out-of-scope for this diagnostic engagement.

**Production-readiness assessment.** The deliverable is **production-ready** (accurate, internally consistent, comprehensive, evidence-backed, source-preserving). It is suitable for stakeholder and security review as-is.

| Success Metric | Target | Actual | Status |
|----------------|--------|--------|--------|
| AAP requirements answered (R1–R6) | 6/6 | 6/6 | ✅ |
| Source files modified | 0 | 0 | ✅ |
| Autonomous tests passing | 100% | 639/639 (100%) | ✅ |
| Cited locators verified accurate | ~60 | ~60 (1 minor fix) | ✅ |
| Deliverable at mandated path | Yes | Yes | ✅ |

---

## 9. Development Guide

This guide explains how to build, run, validate, and reproduce the runtime evidence for the SimpleLogin reply-pipeline investigation, and how to view/validate the deliverable. All commands were tested; representative outputs are shown.

### 9.1 System Prerequisites

- **Python 3.10** (runtime of record — `Dockerfile:L8` `FROM python:3.10`; `pyproject.toml:L61` `python = "^3.10"`).
- **PostgreSQL 13** (`.github/workflows/main.yml:L47` `image: postgres:13`).
- **Redis 6** (`.github/workflows/main.yml:L91-92`).
- **Poetry** (dependency management against `poetry.lock`).
- **Git** and (optionally) **Docker 28.x** for the containerized path.

> Note: the canonical reproduction path is the **Docker stack** used by the autonomous validator (`sl-app` + `sl-postgres` PG13 + `sl-redis`). A native Poetry path mirroring the Dockerfile/CI is also given.

### 9.2 Environment Setup

```bash
# From the repository root
cd /tmp/blitzy/app/blitzy-aa9e51ec-4ed4-4ff2-8455-049ee33b076d_1b7bbd

# Seed local configuration from the shipped example
cp example.env .env        # defines DB_URI and EMAIL_DOMAIN=sl.local
# Key declarations (verified):
#   example.env:L22  EMAIL_DOMAIN=sl.local
#   example.env:L75  DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin
# For the test suite, tests/test.env is used instead:
#   tests/test.env:L8   EMAIL_DOMAIN=sl.local
#   tests/test.env:L17  DB_URI=postgresql://test:test@localhost:15432/test
```

Provision datastores (containerized path):

```bash
# PostgreSQL 13 (test DSN: postgresql://test:test@localhost:15432/test)
docker run -d --name sl-postgres -e POSTGRES_USER=test -e POSTGRES_PASSWORD=test \
  -e POSTGRES_DB=test -p 15432:5432 postgres:13

# Redis 6
docker run -d --name sl-redis -p 6379:6379 redis:6
```

### 9.3 Dependency Installation

```bash
# Install Python 3.10 deps from the lockfile (no manifest changes)
poetry install
```

Expected: Poetry resolves the locked stack (SQLAlchemy 1.3.24, psycopg2-binary, aiosmtpd, flanker, redis, flask, flask-migrate, etc.).

Static sanity (works even outside the venv — pure syntax check):

```bash
python3 -m py_compile email_handler.py app/models.py app/email_utils.py \
  app/contact_utils.py app/email_validation.py app/utils.py app/parallel_limiter.py
# Expected: no output, exit code 0 (all modules byte-compile cleanly)
```

### 9.4 Apply Migrations

```bash
# alembic.ini:L5  script_location = migrations
CONFIG=tests/test.env poetry run flask db upgrade
# or, equivalently:
CONFIG=tests/test.env poetry run alembic upgrade head
# Expected head revision: 32f25cbf12f6
```

### 9.5 Run the Test Suite (Regression Gate)

```bash
CONFIG=tests/test.env GITHUB_ACTIONS_TEST=true poetry run pytest
# Expected: 639 passed, 0 failed (≈51 benign warnings)
```

### 9.6 Reproduce the Runtime Evidence (R6)

The investigation used **temporary** scripts placed only in `/tmp` (never in the source tree) against a throwaway database. Outline of the methodology (full detail in deliverable Appendix A):

```bash
# 1) Create a throwaway DB and migrate it to head
#    (e.g., a database named 'replcheck'), then point CONFIG/DB_URI at it.
# 2) Seed two Contacts (different users/aliases) sharing an identical reply_email
#    -> observe NO IntegrityError (Evidence A).
# 3) Call Contact.get_by(reply_email=...) repeatedly
#    -> compiled SQL has NO ORDER BY; .first() returns one arbitrary row (Evidence B);
#       .one() raises MultipleResultsFound while .first() is silent (Evidence B2).
# 4) Delete the resolved row and re-query
#    -> destination user flips A -> B (Evidence B3).
# 5) Drive concurrent create_contact()/generate_reply_email()
#    -> both pass lock-free available_sl_email() and both INSERT the same reply_email (Evidence C).
# 6) Tear everything down: drop the throwaway DB, delete /tmp scripts.
```

### 9.7 View & Validate the Deliverable

```bash
DOC="blitzy/documentation/app_2cd6ee777f8c.md"

ls -l "$DOC"                              # 36770 bytes
wc -l "$DOC"                              # 533 lines
grep -c '^```' "$DOC"                     # 54 (even => balanced code fences)
grep -c '^```mermaid' "$DOC"              # 1 mermaid diagram
grep -nE '^#{1,2} ' "$DOC"               # lists all section headings (§1–§9, Appendix A/B)
```

Verify scope integrity (source unchanged):

```bash
git status --porcelain                    # (empty) => clean working tree
git diff --name-status 2cd6ee77..HEAD     # A  blitzy/documentation/app_2cd6ee777f8c.md
git diff --numstat   2cd6ee77..HEAD       # 533  0  blitzy/documentation/app_2cd6ee777f8c.md
git log --pretty=format:'%h %ae %s' 2cd6ee77..HEAD   # 3 commits, all agent@blitzy.com
```

### 9.8 Troubleshooting

- **`error: externally-managed-environment` when using pip** — this host's system Python is PEP-668 managed. Prefer a venv (`python -m venv .venv && source .venv/bin/activate`) or pass `--break-system-packages` for throwaway installs. Project deps should go through **Poetry**, not pip.
- **`poetry: command not found`** — Poetry is not on every host; use the Docker path (§9.2) or install Poetry into the environment first.
- **Migrations fail with connection refused** — confirm the Postgres container is up and the port matches the DSN (`15432` for `tests/test.env`).
- **Host Python is 3.13, not 3.10** — pure `py_compile` checks still pass on 3.13, but the test suite and runtime evidence must run on **Python 3.10** (use the Docker image `python:3.10` or a 3.10 venv) to match the runtime of record.
- **Wrong Alembic head** — re-run `alembic upgrade head`; the expected head is `32f25cbf12f6`.

---

## 10. Appendices

### Appendix A — Command Reference

| Purpose | Command |
|---------|---------|
| Install locked deps | `poetry install` |
| Byte-compile key modules | `python3 -m py_compile email_handler.py app/models.py app/email_utils.py app/contact_utils.py app/email_validation.py app/utils.py app/parallel_limiter.py` |
| Apply migrations | `CONFIG=tests/test.env poetry run flask db upgrade` |
| Run full test suite | `CONFIG=tests/test.env GITHUB_ACTIONS_TEST=true poetry run pytest` |
| Validate deliverable size | `wc -l blitzy/documentation/app_2cd6ee777f8c.md` |
| Check code-fence balance | `grep -c '^```' blitzy/documentation/app_2cd6ee777f8c.md` |
| Verify scope integrity | `git diff --name-status 2cd6ee77..HEAD` |

### Appendix B — Port Reference

| Service | Port | Source |
|---------|------|--------|
| PostgreSQL (test DSN) | 15432 → 5432 | `tests/test.env:L17` |
| PostgreSQL (example DSN) | 5432 | `example.env:L75` |
| Redis | 6379 | `.github/workflows/main.yml` (Redis v6) |

### Appendix C — Key File Locations

| File | Lines | Role |
|------|-------|------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | 533 | **The deliverable** |
| `email_handler.py` | 2404 | Inbound SMTP routing; `handle()`, `handle_reply()`, `handle_forward()` |
| `app/models.py` | 3843 | `Contact` model, `ModelMixin.get_by`, `available_sl_email` |
| `app/email_utils.py` | 1505 | `generate_reply_email`, `is_reverse_alias` |
| `app/contact_utils.py` | 120 | `create_contact` + `IntegrityError` recovery |
| `app/email_validation.py` | 38 | `normalize_reply_email`, `_ALLOWED_CHARS` |
| `app/utils.py` | 158 | `random_string` entropy |
| `app/parallel_limiter.py` | 73 | Redis limiter (dashboard/API only; **not** inbound) |
| `migrations/versions/2021_071310_78403c7b8089_.py` | 29 | `reply_email` index `unique=False` |
| `migrations/versions/5fa68bafae72_.py` | 39 | Original `contact.reply_email` column |

### Appendix D — Technology Versions

| Component | Version | Source |
|-----------|---------|--------|
| Python | 3.10 | `Dockerfile:L8`, `pyproject.toml:L61` |
| PostgreSQL | 13 (observed 13.23) | `.github/workflows/main.yml:L47` |
| Redis | 6 | `.github/workflows/main.yml:L91-92` |
| SQLAlchemy | 1.3.24 | `pyproject.toml` / `poetry.lock` |
| Flask | ^1.1.2 | `pyproject.toml` |
| aiosmtpd | ^1.2 | `pyproject.toml` |
| Alembic head | `32f25cbf12f6` | live migration head |

### Appendix E — Environment Variable Reference

| Variable | Example / Value | Source |
|----------|-----------------|--------|
| `EMAIL_DOMAIN` | `sl.local` | `example.env:L22`, `tests/test.env:L8` |
| `DB_URI` (app) | `postgresql://myuser:mypassword@localhost:5432/simplelogin` | `example.env:L75` |
| `DB_URI` (tests) | `postgresql://test:test@localhost:15432/test` | `tests/test.env:L17` |
| `FLASK_SECRET` | `secret` (test only) | `tests/test.env:L20` |
| `CONFIG` | `tests/test.env` | pytest/migration invocation |
| `GITHUB_ACTIONS_TEST` | `true` | test-suite invocation |

### Appendix F — Developer Tools Guide

- **Poetry** — dependency resolution & venv management against `poetry.lock` (no manifest changes in this engagement).
- **Alembic / Flask-Migrate** — schema migrations; `alembic.ini:L5` `script_location = migrations`; head `32f25cbf12f6`.
- **pytest** — test runner; invoke non-interactively with `CONFIG=tests/test.env GITHUB_ACTIONS_TEST=true`.
- **Docker 28.x** — containerized PG13/Redis6 provisioning (canonical reproduction path).
- **git** — scope-integrity verification (`git diff --name-status 2cd6ee77..HEAD`).

### Appendix G — Glossary

| Term | Definition |
|------|------------|
| **Reverse-alias / reply-email** | A per-(alias, contact) generated address; replying to it relays the message back to the original sender while hiding the user's real mailbox. |
| **TOCTOU** | Time-Of-Check to Time-Of-Use — the window between a lock-free availability check and the later INSERT, during which two actors can both "win". |
| **`get_by().first()`** | `Session.query(cls).filter_by(**kw).first()` — returns one row with **no `ORDER BY`**; does not raise on multiple matches (unlike `.one()`). |
| **`uq_contact`** | The **only** unique constraint on `Contact`: `UniqueConstraint("alias_id", "website_email")`. `reply_email` is indexed but **not** unique. |
| **AAP** | Agent Action Plan — the authoritative scope document for this engagement. |
| **Path-to-production** | Standard activities (here: human SME review/sign-off) required to move a verified deliverable toward release. |