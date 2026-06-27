# Blitzy Project Guide

**Project:** SimpleLogin Email-Forwarding Flow — Forensic Q&A Investigation
**Source Branch:** `app_2cd6ee777f8c`
**Working Branch:** `blitzy-8675c1b5-23ed-4f3e-901d-a14ff7ff7637`
**HEAD Commit:** `68f08f29`
**Investigated Commit:** `2cd6ee77` ("chore: emit some missing contact audit logs (#2269)")
**Task Type:** Read-only forensic investigation / documentation (ruleset `SWE-AtlasQnA-Repo`)

---

## 1. Executive Summary

### 1.1 Project Overview

This project is a **forensic, runtime-grounded investigation** of SimpleLogin's inbound email-forwarding flow, motivated by a production issue concerning inconsistent forwarding behavior. The objective was to produce a single Markdown answer document capturing **genuine runtime values** — not values inferred by reading code — for four specific questions: (Q1) success vs. alias-not-found log behavior, (Q2) how the SimpleLogin Message-ID differs from the sender's original, (Q3) the transformed `From` header and reverse-alias format, and (Q4) the database records created by one forward. The target users are SimpleLogin engineers triaging the production issue. The technical scope is deliberately narrow and **read-only**: exactly one new file is produced and zero source files are modified.

### 1.2 Completion Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeWidth':'2px','pieSectionTextColor':'#B23AF2','pieTitleTextSize':'18px','pieLegendTextSize':'14px'}}}%%
pie showData
    title Completion Status — 92.0% Complete
    "Completed Work (hrs)" : 23
    "Remaining Work (hrs)" : 2
```

| Metric | Value |
|---|---|
| **Total Hours** | **25.0 h** |
| **Completed Hours (AI + Manual)** | **23.0 h** (AI: 23.0 h · Manual: 0.0 h) |
| **Remaining Hours** | **2.0 h** |
| **Percent Complete** | **92.0 %** |

> **Calculation (PA1, AAP-scoped):** `Completion % = Completed ÷ (Completed + Remaining) × 100 = 23.0 ÷ 25.0 × 100 = 92.0 %`. The denominator includes only AAP-scoped work and the standard human acceptance step for the deliverable. The motivating production issue is explicitly out of scope (investigate-not-fix) and is excluded.

### 1.3 Key Accomplishments

- ✅ **Sole deliverable authored and committed** — `blitzy/documentation/app_2cd6ee777f8c.md` (551 lines, ~30 KB), correctly named after the source branch and placed in `blitzy/documentation/`.
- ✅ **Q1 answered with genuine runtime evidence** — success path returns `250 Message accepted for delivery` (`status.E200`) with 9 ordered LOG lines; non-existent alias returns `550 SL E515 Email not exist` (`status.E515`) with 4 LOG lines.
- ✅ **Q2 answered with the pivotal nuance** — a forward **preserves** the original `Message-ID`; a new SL Message-ID is **minted only on the reply path** via `make_msgid(...)` and persisted in a `MessageIDMatching` row.
- ✅ **Q3 answered** — the transformed `From` header and the **prefix-less random reverse-alias** at `@sl.local` (`sender_format=0` AT, `include_sender_in_reverse_alias=True`).
- ✅ **Q4 answered** — a forward from a new sender creates **exactly 3 rows** (`Contact`, `UserAuditLog`, `EmailLog`) with captured IDs and `created_at` timestamps; a known contact creates only 1.
- ✅ **Captured from a faithful pinned stack** — Python 3.10.18, genuine compiled `pyre2 0.3.6`, `SQLAlchemy 1.3.24`; no dependency substitutions.
- ✅ **All four validation gates passed** — dependencies, compilation, unit tests (23/23 for `test_email_handler.py`), and runtime.
- ✅ **User constraints honored** — zero source-file modifications (`git diff` = 1 file added, 0 modified); all spun-up infrastructure cleaned up.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|---|---|---|---|
| _None blocking._ The deliverable is complete, validated, and committed. | No release/validation blockers for this documentation deliverable. | — | — |
| Underlying SimpleLogin inconsistent-forwarding production issue remains **unremediated by design** (out of scope: investigate-not-fix). | Behavior persists in production until engineers act on the findings; this is the intended handoff, not a defect in the deliverable. | SimpleLogin Eng (downstream) | To be scheduled by owning team |

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|---|---|---|---|---|
| Public internet (sandbox) | Outbound network | Sandbox has no internet; one full-suite test (Apple IAP) requires an external endpoint and fails for that reason only. Unrelated to email forwarding and out of scope. | Accepted (environmental, non-blocking) | Platform / Infra |

> No repository-permission, credential, or third-party-API access issues affect this read-only documentation deliverable. Aside from the noted sandbox network limitation, **no access issues identified**.

### 1.6 Recommended Next Steps

1. **[High]** Have a SimpleLogin-knowledgeable engineer review and accept the forensic Q&A (`app_2cd6ee777f8c.md`), confirming the captured values and rationale answer the original production question. *(≈1.5 h)*
2. **[Low]** Optionally reproduce the findings using the document's "Reproduction & cleanup" section to confirm the invariant structure/behavior (non-deterministic values will differ). *(≈0.5 h)*
3. **[Medium — downstream, out of scope]** Using the documented evidence, open a follow-up engineering ticket to remediate the inconsistent-forwarding production issue (e.g., reconcile reverse-alias/`sender_format` behavior across user configurations).
4. **[Low]** Merge the PR so the answer document is available to the team as the authoritative reference for this flow.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|---|---|---|
| Runtime environment standup | 3.0 | Faithful pinned stack: Python 3.10.18, genuine compiled `pyre2 0.3.6` (RE2 binding), PostgreSQL + Redis containers, `alembic upgrade head` → 77 tables. [R6] |
| Forensic source-code analysis & call-chain tracing | 5.0 | Read `email_handler.py` (2404 L) + `app/*` modules to map the forward/reply/failure call chain and locate every LOG line, status string, and citation. [R2–R5, R7] |
| Ephemeral out-of-repo probe + scenario driving | 4.0 | Authored an outside-repo probe importing `email_handler`; drove `handle()` under `mail_sender` store-mode across 3 scenarios, run twice for determinism. [R6, R8] |
| Runtime evidence capture & per-question analysis | 3.0 | Captured/organized Q1 logs + return codes, Q2 Message-ID comparison + matching row, Q3 `From` + reverse-alias decomposition, Q4 DB rows with IDs/timestamps. [R2–R5] |
| Answer document authoring | 5.0 | Wrote the 551-line forensic Q&A: methodology, Q1–Q4 (Captured + Rationale), citation index, reproduction & cleanup. [R1, R7, R10] |
| Final validation & re-grounding | 3.0 | Faithful re-run; 4 validation gates; re-captured all values into one coherent snapshot; replaced obsolete dependency-substitution caveat with an accurate Provenance note; well-formedness checks; commit; teardown. [R6, R8, R9] |
| **Total Completed** | **23.0** | **All autonomous (AI). Manual: 0.0 h.** |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|---|---|---|
| Human SME accuracy review & acceptance of the forensic findings vs. the original production question | 1.5 | High |
| Optional independent reproduction via the provided harness/script | 0.5 | Low |
| **Total Remaining** | **2.0** | — |

### 2.3 Hours Reconciliation

| Check | Result |
|---|---|
| Section 2.1 total (Completed) | 23.0 h |
| Section 2.2 total (Remaining) | 2.0 h |
| 2.1 + 2.2 = Total (Section 1.2) | 23.0 + 2.0 = **25.0 h** ✅ |
| Remaining matches Section 1.2 / Section 7 | 2.0 h ✅ |
| Completion % = 23.0 ÷ 25.0 × 100 | **92.0 %** ✅ |

---

## 3. Test Results

All tests below originate from **Blitzy's autonomous validation logs** for this project (the faithful-environment run).

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---|---|---|---|---|---|---|
| Email-handler suite (Unit/Integration) | pytest | 23 | 23 | 0 | N/A* | `tests/test_email_handler.py` — directly exercises forward, reply, and alias-not-found paths via `email_handler.handle()`. |
| Full regression suite | pytest | 639 | 638 | 1 | N/A* | Single failure is an external-network **Apple IAP** test that fails only because the sandbox has no internet; out of scope, pre-existing, environmental, unrelated to email forwarding, unaffected by this change (0 source files modified). |
| Compilation gate | `py_compile` / import | All referenced modules | Pass | 0 | — | `email_handler`, `app.models`, `app.contact_utils`, `app.email_utils`, `app.mail_sender`, `app.email.status`, `tests/*` all import + compile cleanly. |
| Runtime gate | Ephemeral probe (store-mode) | 3 scenarios × 2 runs | Pass | 0 | — | Forward (valid alias, new sender), forward (non-existent alias), reply (reverse-alias). Structure/behavior invariant; only non-deterministic values varied across runs. |

> *Coverage percentage was not separately reported in the validation logs for this read-only investigation; the email-forward path was exercised end-to-end. No fabricated coverage figure is presented.

---

## 4. Runtime Validation & UI Verification

This is a **backend SMTP-handler investigation; there is no UI component in scope**, so UI verification is not applicable. Runtime validation of the email-forward flow:

- ✅ **Operational — Inbound handler (`email_handler.handle()`):** Drove end-to-end across three scenarios; emitted the expected ordered LOG lines and SMTP return codes.
- ✅ **Operational — Success forward path:** Returned `status.E200` (`250 Message accepted for delivery`); 9 ordered LOG lines captured; `Session.commit()` persisted records.
- ✅ **Operational — Alias-not-found failure path:** Returned `status.E515` (`550 SL E515 Email not exist`); 4 LOG lines captured; no records persisted.
- ✅ **Operational — Reply path (Q2 contrast):** Minted a new SL Message-ID via `make_msgid(...)`; persisted a `MessageIDMatching` row linking original ↔ SL Message-ID.
- ✅ **Operational — Outbound transformation capture:** `mail_sender` store-mode (`get_stored_emails()`) captured the rewritten `From` header and reverse-alias without real SMTP delivery (`NOT_SEND_EMAIL=true`).
- ✅ **Operational — Database writes:** New-sender forward produced exactly 3 rows (`Contact`, `UserAuditLog`, `EmailLog`) with strictly increasing `created_at`; known-contact forward produced 1 (`EmailLog`).
- ⚠ **Partial — External delivery:** Not exercised by design (store-mode, no real SMTP); not required to answer Q1–Q4.

---

## 5. Compliance & Quality Review

Cross-map of AAP deliverables and `SWE-AtlasQnA-Repo` ruleset to outcomes.

| Requirement (AAP / Rule) | Benchmark | Status | Evidence |
|---|---|---|---|
| R1 — Create `app_2cd6ee777f8c.md` in `blitzy/documentation/` | Correct name + location | ✅ Pass (100%) | File exists at exact path; committed at HEAD `68f08f29`. |
| R2 — Q1: success + failure logs & return codes | Genuine runtime values | ✅ Pass (100%) | §1.1 (E200 + 9 lines), §1.2 (E515 + 4 lines); source-confirmed `status.py` L2/L51. |
| R3 — Q2: forward-preserves vs reply-mints + matching | Genuine runtime values | ✅ Pass (100%) | §2.1/§2.2; minted SL Message-ID + `MessageIDMatching` row captured. |
| R4 — Q3: transformed `From` + reverse-alias format | Genuine runtime values | ✅ Pass (100%) | §3.1; prefix-less reverse-alias; `sender_format=0`, include-sender=True. |
| R5 — Q4: DB rows with real IDs + `created_at` | Genuine runtime values | ✅ Pass (100%) | §4.1; 3 rows (Contact/UserAuditLog/EmailLog) with IDs + timestamps. |
| R6 — Build & run to capture runtime values (code-as-truth) | Faithful execution | ✅ Pass (100%) | Faithful pinned stack; `handle()` driven 3×2; Provenance note documents no substitutions. |
| R7 — Rationale per answer + inline citations | Reasoning + traceability | ✅ Pass (100%) | Rationale §1.3/§2.3/§3.2/§4.2; comprehensive Citation index table. |
| R8 — No source-file modifications | Byte-for-byte unchanged source | ✅ Pass (100%) | `git diff 2cd6ee77..HEAD` = 1 file added / 0 modified; `git status` clean; probe lived outside repo tree. |
| R9 — Clean up spun-up containers/DB | Infrastructure teardown | ✅ Pass (100%) | `sl-validate` container removed; host probe artifacts removed; pre-existing `sl-app` untouched. |
| R10 — Methodology section | How built/run/captured | ✅ Pass (100%) | §(0) Introduction & Methodology + Reproduction & cleanup. |

**Fixes applied during autonomous validation:** Re-grounded all captured values in a single coherent faithful snapshot (0 prior-run values remain); replaced the prior run's dependency-substitution caveat (a `re` shim for `pyre2`; `cbor2 5.6.4`) with an accurate Provenance note reflecting the genuine pinned stack; corrected the Q3 reverse-alias composition diagram and the Q4 authoring note; verified Markdown well-formedness (balanced fences, valid UTF-8, LF endings, no trailing whitespace) and pre-commit hooks.

**Outstanding compliance items:** None. All AAP requirements and ruleset directives are satisfied.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|---|---|---|---|---|---|
| T1 — Non-deterministic captured values (random reverse-alias suffix, `make_msgid` components, UUID Message-IDs, `created_at`) differ on each run; a reader expecting exact reproduction will see different strings. | Technical | Low | Medium | Provenance note + Citation index explicitly flag which values are non-deterministic; code-level structure/behavior is invariant and is what citations point to. | Mitigated |
| T2 — Snapshot is tied to commit `2cd6ee77` + the pinned stack; future code drift could diverge from the captured values. | Technical | Low | Low | Document pins the investigated commit and exact dependency stack. | Mitigated |
| T3 — Faithful reproduction requires Python 3.10 + genuine `pyre2 0.3.6`; a non-faithful stack (e.g., host Python 3.13) could alter regex behavior. | Technical | Low | Medium | Development Guide and the doc specify the exact pinned stack / Docker image and the `pyre2` build dependencies. | Mitigated |
| S1 — Sensitive data exposure. | Security | Negligible | Low | Read-only doc ships no code, endpoints, or credentials; sample data is synthetic; test DB credentials (`test/test/test`) are non-production. | Accepted |
| O1 — Orphaned containers/DB if teardown were incomplete. | Operational | Low | Low | Logs confirm `sl-validate` removed, host artifacts removed, pre-existing `sl-app` untouched, `git status` clean. | Resolved |
| O2 — Underlying inconsistent-forwarding production issue remains unremediated (by design). | Operational | Medium | N/A (existing condition) | Out of scope (investigate-not-fix); findings handed off to engineers via this document for a follow-up ticket. | Open — downstream handoff |
| I1 — Findings depend on the test configuration (`EMAIL_DOMAIN=sl.local`, `sender_format=0`, include-sender=True); production users with other settings yield different `From`/reverse-alias shapes. | Integration | Low | Medium | §3.2 explains the dependence on `sender_format` and the include-sender flag and reports the captured config. | Documented |
| I2 — No external service integration exercised (store-mode, no real SMTP); one full-suite failure is an external-network Apple IAP test. | Integration | Negligible | N/A | Not required for Q1–Q4; the IAP failure is environmental, out of scope, unrelated to email forwarding. | Accepted |

---

## 7. Visual Project Status

**Project Hours Breakdown** (Completed = Dark Blue `#5B39F3`; Remaining = White `#FFFFFF`):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeWidth':'2px','pieSectionTextColor':'#B23AF2','pieTitleTextSize':'16px','pieLegendTextSize':'13px'}}}%%
pie showData
    title Project Hours — Completed vs Remaining
    "Completed Work" : 23
    "Remaining Work" : 2
```

**Remaining Work by Category** (hours, from Section 2.2 — sums to 2.0 h):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#A8FDD9','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieSectionTextColor':'#B23AF2','pieTitleTextSize':'16px','pieLegendTextSize':'13px'}}}%%
pie showData
    title Remaining Work by Category (2.0 h total)
    "SME accuracy review (High)" : 1.5
    "Optional reproduction (Low)" : 0.5
```

> **Integrity:** Pie chart "Remaining Work" (2.0 h) = Section 1.2 Remaining (2.0 h) = Section 2.2 sum (2.0 h). "Completed Work" (23.0 h) = Section 1.2 Completed (23.0 h) = Section 2.1 sum (23.0 h).

---

## 8. Summary & Recommendations

**Achievements.** The project is **92.0% complete**. All ten AAP requirements are satisfied: the sole deliverable, `blitzy/documentation/app_2cd6ee777f8c.md`, comprehensively answers Q1–Q4 with **genuine runtime values**, rationale, and inline citations, captured from a faithful pinned stack (Python 3.10.18, genuine `pyre2 0.3.6`, `SQLAlchemy 1.3.24`). All four validation gates passed (dependencies, compilation, 23/23 email-handler tests, runtime). The two binding user constraints — **no source modifications** and **clean up spun-up infrastructure** — were both honored (`git diff` shows a single file added; containers and host artifacts removed).

**Remaining gaps (2.0 h).** The only outstanding work is human verification: a SME accuracy review/acceptance of the forensic findings (1.5 h, High) and an optional independent reproduction (0.5 h, Low). There are no compilation errors, no failing in-scope tests, and no configuration or deployment tasks, because the deliverable is a documentation artifact rather than running software.

**Critical path to production.** SME review → acceptance → merge. The document is then the authoritative reference for the flow.

**Production readiness assessment.** The deliverable is **ready for human review**. It is well-formed, internally consistent (timestamps from a single ~`2026-06-27T00:57:51Z` snapshot align across Q2 and Q4), and free of the earlier provenance caveat. The completion percentage is intentionally held below 100% to reserve the mandatory human acceptance step.

**Downstream (out of scope).** The motivating inconsistent-forwarding production issue is **investigated and documented, not remediated**. Engineers should open a follow-up ticket using this document's evidence — particularly the Q3 finding that reverse-alias / `From` shape depends on `sender_format` and `include_sender_in_reverse_alias`, which is a likely source of perceived inconsistency.

| Success Metric | Target | Actual | Status |
|---|---|---|---|
| Single artifact, correct name + location | Yes | `blitzy/documentation/app_2cd6ee777f8c.md` | ✅ |
| Source files modified | 0 | 0 | ✅ |
| Questions answered with runtime values | 4 / 4 | 4 / 4 | ✅ |
| In-scope validation gates passed | 4 / 4 | 4 / 4 | ✅ |
| Infrastructure cleaned up | Yes | Yes | ✅ |
| AAP-scoped completion | ~100% (minus human review) | 92.0% | ✅ |

---

## 9. Development Guide

This guide reproduces the **read-only** investigation. It mirrors the existing `scripts/run-test.sh` lifecycle and leaves the source tree byte-for-byte unchanged.

### 9.1 System Prerequisites

- **Docker** 28.x with the Compose plugin (verified: Docker 28.5.2, Compose v5.1.4).
- **Python 3.10** — the pinned runtime. *Faithful capture requires 3.10; do not use the host interpreter if it differs (e.g., 3.13).* Easiest path is the repo `Dockerfile` (`FROM python:3.10`).
- **Poetry** — dependency management (installed inside the Docker image per the `Dockerfile`).
- **Build tools for `pyre2`** — `libre2-dev`, `cmake`, `ninja-build`, `gcc`, `python3-dev` (the genuine RE2 binding compiles against these).

### 9.2 Environment Setup

```bash
# From the repository root on the working branch.
# 1) (Re)create an ephemeral PostgreSQL 13 test DB — same image/port/credentials as scripts/run-test.sh
docker rm -f sl-test-db 2>/dev/null || true
docker run -d --name sl-test-db \
  -e POSTGRES_PASSWORD=test -e POSTGRES_USER=test -e POSTGRES_DB=test \
  -p 15432:5432 postgres:13

# 2) Give the container a moment to accept connections
sleep 3
```

Runtime configuration comes from `tests/test.env` (read, never edited):

```bash
# Key values (verified):
#   NOT_SEND_EMAIL=true          # store-mode; no real SMTP delivery
#   EMAIL_DOMAIN=sl.local
#   DB_URI=postgresql://test:test@localhost:15432/test
#   MEM_STORE_URI=redis://localhost
#   DMARC_CHECK_ENABLED=true
```

### 9.3 Dependency Installation

```bash
# Install the EXACT pinned stack from poetry.lock — NO substitutions.
# (Genuine pyre2 0.3.6, SQLAlchemy 1.3.24, aiosmtpd 1.4.2, flanker 0.9.11,
#  arrow 0.16.0, Flask 1.1.2, cbor2 5.2.0, etc.)
poetry install
```

### 9.4 Apply the Schema

```bash
# Materialize the schema at Alembic head (-> 77 tables) using the test config.
CONFIG=tests/test.env poetry run alembic upgrade head
```

### 9.5 Drive a Forward (read-only capture)

**Option A — existing harness with verbose logging (no new files):**

```bash
CONFIG=tests/test.env poetry run pytest -s \
  -o log_cli=true -o log_cli_level=DEBUG \
  tests/test_email_handler.py -k forward
```

**Option B — ephemeral probe OUTSIDE the repository tree** that imports `email_handler`, wraps the run in `@mail_sender.store_emails_test_decorator`, and calls `email_handler.handle(envelope, msg)` for three scenarios (valid forward from a new sender, forward to a non-existent alias, reply to a reverse-alias). Capture:

- emitted `LOG.d` / `LOG.i` lines → **Q1**
- inbound vs. outbound `Message-ID` + any `message_id_matching` row → **Q2**
- the stored outbound message's `From` header via `mail_sender.get_stored_emails()` → **Q3**
- the `Contact` / `UserAuditLog` / `EmailLog` rows → **Q4**

### 9.6 Verification

- **Q1:** Success run ends with `250 Message accepted for delivery` (`status.E200`); a non-existent alias ends with `550 SL E515 Email not exist` (`status.E515`).
- **Q2:** A forward leaves `EmailLog.message_id` equal to the original `Message-ID` (no `message_id_matching` row); a reply produces a new `make_msgid(...)` SL Message-ID and a `MessageIDMatching` row.
- **Q3:** The stored outbound `From` contains a prefix-less random reverse-alias at `@sl.local`.
- **Q4:** A new-sender forward inserts exactly 3 rows; a known-contact forward inserts 1 (`EmailLog`).

### 9.7 Teardown (mandatory cleanup)

```bash
docker rm -f sl-test-db
git status --porcelain   # expect EMPTY — source tree byte-for-byte unchanged
```

### 9.8 Troubleshooting

- **Port 15432 already in use** → `docker rm -f sl-test-db` then re-create.
- **`pyre2` build failure** → install `libre2-dev cmake ninja-build gcc python3-dev` before `poetry install`.
- **Values differ from the document** → expected; the random suffix, `make_msgid` components, UUID Message-IDs, and `created_at` are non-deterministic. The code-level structure is invariant.
- **Host Python ≠ 3.10** → use the repo Docker image (`FROM python:3.10`) for faithful capture.
- **One full-suite test fails (Apple IAP)** → environmental (no sandbox internet), out of scope, unrelated to email forwarding.

---

## 10. Appendices

### Appendix A — Command Reference

| Command | Purpose |
|---|---|
| `docker run -d --name sl-test-db -e POSTGRES_PASSWORD=test -e POSTGRES_USER=test -e POSTGRES_DB=test -p 15432:5432 postgres:13` | Stand up the ephemeral test DB |
| `CONFIG=tests/test.env poetry run alembic upgrade head` | Apply schema to Alembic head (77 tables) |
| `CONFIG=tests/test.env poetry run pytest -s -o log_cli=true -o log_cli_level=DEBUG tests/test_email_handler.py -k forward` | Drive a forward with DEBUG log streaming |
| `poetry run pytest -c pytest.ci.ini` | Run the CI test suite |
| `docker rm -f sl-test-db` | Tear down the test DB |
| `git status --porcelain` | Confirm source tree unchanged |
| `git diff 2cd6ee77..HEAD --stat` | Confirm exactly one file added |

### Appendix B — Port Reference

| Port | Service | Notes |
|---|---|---|
| 15432 | PostgreSQL 13 (`sl-test-db`) | Maps to container `5432`; credentials `test/test/test` |
| 6379 | Redis (`MEM_STORE_URI`) | `redis://localhost` per `tests/test.env` |
| 7777 | gunicorn (web, production) | Not exercised by this investigation |

### Appendix C — Key File Locations

| Path | Role |
|---|---|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The deliverable** (the only file created) |
| `email_handler.py` | Inbound SMTP processor — Q1 logs, Q3 transform, Q4 `EmailLog` |
| `app/email/status.py` | SMTP status strings (`E200` L2, `E515` L51) |
| `app/contact_utils.py` | Q4 `Contact` + `UserAuditLog` creation |
| `app/email_utils.py` | Q3 reverse-alias generation |
| `app/models.py` | Q3 `Contact.new_addr()`; `EmailLog`/`Contact`/`MessageIDMatching`/`UserAuditLog` models |
| `app/mail_sender.py` | Store-mode capture (`get_stored_emails()`) |
| `tests/test_email_handler.py` | Harness pattern driving `handle()` |
| `tests/test.env` | Runtime config (read-only) |
| `scripts/run-test.sh` | DB lifecycle + teardown pattern |

### Appendix D — Technology Versions

| Component | Version | Source |
|---|---|---|
| Python | 3.10 (run 3.10.18) | `Dockerfile` `FROM python:3.10` |
| PostgreSQL | 13 (test DB) | `scripts/run-test.sh` |
| SQLAlchemy | 1.3.24 (exact pin) | `pyproject.toml` / `poetry.lock` |
| pyre2 | 0.3.6 (genuine RE2 binding) | `poetry.lock` |
| aiosmtpd | 1.4.2 | `poetry.lock` |
| flanker | 0.9.11 | `poetry.lock` |
| arrow | 0.16.0 | `poetry.lock` |
| Flask | 1.1.2 | `poetry.lock` |
| cbor2 | 5.2.0 | `poetry.lock` |
| Alembic head | `32f25cbf12f6` (77 tables) | migrations |

### Appendix E — Environment Variable Reference

| Variable | Value (test) | Purpose |
|---|---|---|
| `CONFIG` | `tests/test.env` | Points the app/Alembic at the test configuration |
| `NOT_SEND_EMAIL` | `true` | Enables store-mode; suppresses real SMTP delivery |
| `EMAIL_DOMAIN` | `sl.local` | Alias domain used for the captured reverse-alias |
| `DB_URI` | `postgresql://test:test@localhost:15432/test` | Test database connection |
| `MEM_STORE_URI` | `redis://localhost` | Redis backend / rate limiting |
| `DMARC_CHECK_ENABLED` | `true` | DMARC handling toggle (out of scope here) |

### Appendix F — Developer Tools Guide

| Tool | Use |
|---|---|
| `docker` / `docker compose` | Manage the ephemeral PostgreSQL test DB |
| `poetry` | Install the pinned dependency stack; run `alembic` / `pytest` |
| `alembic` | Apply the schema (`upgrade head`) |
| `pytest` (`-s`, `log_cli`) | Drive the handler and stream DEBUG logs |
| `mail_sender` store-mode | Capture the transformed outbound message in-process |
| `git diff` / `git status` | Verify zero source modifications |

### Appendix G — Glossary

| Term | Definition |
|---|---|
| **Reverse-alias** | An on-the-fly address created per sender so replies route back through the alias without exposing the real mailbox. Modern default: a prefix-less random local part at the alias domain. |
| **SL Message-ID** | A SimpleLogin-minted `Message-ID` (`make_msgid(...)`) created on the **reply** path; recorded in `message_id_matching`. A forward preserves the sender's original `Message-ID`. |
| **`status.E200` / `status.E515`** | SMTP replies `250 Message accepted for delivery` (success) and `550 SL E515 Email not exist` (alias not found). |
| **Store-mode** | `mail_sender` test mode (`NOT_SEND_EMAIL=true`) that retains outbound messages in-process for inspection instead of sending. |
| **`sender_format`** | Per-user setting controlling the rewritten `From` header shape; `0` = AT format (e.g., `... at ...`). |
| **Trace UUID** | A per-email log-correlation UUID minted by `MailHandler._handle()`; **not** the email `Message-ID` header. |
| **Code-as-truth** | The ruleset requirement that every reported value be a genuine runtime capture, not a static inference. |