
# Blitzy Project Guide — Reply-Email Resolution Code Analysis

**Project Branch:** `blitzy-e93b4109-d71b-4598-a7ea-3cda45247412`
**Source/Base Commit:** `2cd6ee77` (`chore: emit some missing contact audit logs (#2269)`)
**Head Commit:** `aa23938c` (`docs: correct off-by-one line reference for 'give-up' raise Exception (1152 -> 1153) in cross-reference table`)
**Repository:** SimpleLogin (`app`) — open-source email alias solution
**Deliverable:** `blitzy/documentation/app_2cd6ee777f8c.md`

---

## 1. Executive Summary

### 1.1 Project Overview

This project is a **pure investigative documentation task** against the SimpleLogin email alias server. The AAP mandates production of a single, comprehensive, code-grounded markdown analysis answering whether SimpleLogin's inbound reply-email resolution pipeline — the mechanism by which a reverse-alias reply address is translated into a `Contact` record and thereby routed to the correct alias owner — can misroute a reply to the wrong user. The AAP strictly prohibits any source-code modifications; the sole artifact is `blitzy/documentation/app_2cd6ee777f8c.md`. The target readership is SimpleLogin's engineering and security team, and the business impact is providing authoritative evidence for downstream architectural decisions (e.g., whether to enforce `UNIQUE(reply_email)` at the database layer).

### 1.2 Completion Status

```mermaid
pie title Project Status — 90% Complete
    "Completed Work (36h)" : 36
    "Remaining Work (4h)" : 4
```

**Color legend:** Completed = Dark Blue (#5B39F3) · Remaining = White (#FFFFFF)

| Metric | Hours |
|---|---|
| **Total Project Hours** | **40** |
| Completed Hours (AI + Manual) | 36 |
| Remaining Hours | 4 |
| **Completion Percentage** | **90%** |

Formula: 36 / (36 + 4) × 100 = **90%**

### 1.3 Key Accomplishments

- ✅ Authoritative 1,451-line / 105,146-byte analysis document created at `blitzy/documentation/app_2cd6ee777f8c.md` (sole AAP deliverable).
- ✅ All five AAP investigation dimensions fully addressed: reply-email derivation, contact resolution logic, cross-event behavioral consistency, race conditions & uniqueness analysis, and runtime value documentation.
- ✅ End-to-end code path traced from `aiosmtpd.handle_DATA` (line 2289) through `handle()` → `is_reverse_alias()` → `handle_reply()` → `Contact.get_by(reply_email=…)` → alias → user → mailbox → delivery.
- ✅ 229+ file:line citations, all verified against source (one off-by-one typo found and corrected in commit `aa23938c`).
- ✅ 106-row evidence table cross-referencing every code reference.
- ✅ 41-row runtime-values table covering every decision point in the reply pipeline.
- ✅ 2 mermaid sequence diagrams visualizing the SMTP-to-delivery flow and the check-then-insert TOCTOU race window.
- ✅ Live PostgreSQL schema verification confirmed `ix_contact_reply_email` is non-unique and `uq_contact(alias_id, website_email)` is the only `UNIQUE` constraint on the `contact` table.
- ✅ 3 iterative revision cycles (commits `faf404a7`, `f736041a`, `aa23938c`) addressing 10+ QA findings without touching any source file.
- ✅ Strict AAP compliance — `git diff --stat 2cd6ee77..HEAD` shows exactly one file added (the deliverable) and **zero source files modified**.
- ✅ Full test suite regression-free: 631 / 631 runnable tests pass; 37 / 37 targeted tests in `test_contact_utils.py` + `test_email_handler.py` pass.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|---|---|---|---|
| None — all AAP requirements satisfied; only human acceptance workflow remains | N/A | N/A | N/A |

### 1.5 Access Issues

No access issues identified. All infrastructure required to validate the deliverable (PostgreSQL on port 15432, Redis on port 6379, Python 3.10.20 virtualenv, SQLAlchemy 1.3.24, pytest 7.x) was pre-provisioned by the setup agent and operated without credential, permission, or network issues throughout the session.

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|---|---|---|---|---|
| — | — | No access issues identified | — | — |

### 1.6 Recommended Next Steps

1. **[Medium]** Technical review of the analysis document by a SimpleLogin engineer familiar with the email pipeline — verify analytical correctness of the three identified pathways (concurrent-insert collision, `normalize_reply_email` collision, `.first()` indeterminacy).
2. **[Medium]** Stakeholder acceptance of the document as the canonical answer to the charter question and merge of this PR into the target branch.
3. **[Low]** Optional environment housekeeping — decide whether the untracked `dump.rdb` Redis artifact (created by the Redis service during test execution, not by the agent) should be added to `.gitignore` in a separate, non-AAP PR.
4. **[Low]** Separately scope any architectural follow-up suggested by the document's findings (e.g., the question of adding a `UNIQUE` constraint on `contact.reply_email`) — this is explicitly out of AAP scope and must not be conflated with the acceptance of the deliverable itself.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|---|---|---|
| **[AAP D1+C1-C5] Codebase investigation & analysis** | 10 | Systematic reading of `email_handler.py` (2,404 lines), `app/models.py` (focus on `Contact`, `Alias`, `User`, `Mailbox`, `SLDomain`, `ModelMixin`), `app/contact_utils.py`, `app/email_utils.py` (`generate_reply_email`, `is_reverse_alias`), `app/email_validation.py` (`normalize_reply_email`), `app/db.py` (session & isolation), and the two `reply_email`-relevant migrations. Cross-referencing against existing tests in `tests/test_email_handler.py` and `tests/test_contact_utils.py`. |
| **[AAP D1+C1] Section 2 drafting — Reply-Email Resolution Flow** | 3 | Line-by-line trace of `handle_DATA` → `handle()` → `is_reverse_alias()` → `handle_reply()` → `Contact.get_by(reply_email=…)`, with mermaid sequence diagram. |
| **[AAP D1+C2] Section 3 drafting — Normalization Before Lookup** | 1.5 | Analysis of `normalize_reply_email()` at `app/email_validation.py:25–38`, including the `convert_to_id()` delegation and lossy-mapping collision channel. |
| **[AAP D1+C4] Section 4 drafting — Double Lookup (TOCTOU in Routing)** | 2 | Documentation of the double `Contact.get_by(reply_email=…)` invocation (in `is_reverse_alias` then in `handle_reply`), with race-window analysis. |
| **[AAP D1+K2] Section 5 drafting — `ModelMixin.get_by()` Semantics** | 1 | Analysis of `.first()` indeterminacy at `app/models.py:82–84`, including SQLAlchemy 1.3.24 query emission (LIMIT 1 with no ORDER BY). |
| **[AAP D1+K3+C4] Section 6 drafting — Schema & Missing UNIQUE Constraint** | 2 | Deep analysis of `Contact.reply_email` column definition (`app/models.py:1899`), migration `2021_071310_78403c7b8089_.py:22` (`unique=False`), and the only existing `UniqueConstraint` `uq_contact(alias_id, website_email)`. Live PostgreSQL `\d contact` verification. |
| **[AAP D1+C1+K4] Section 7 drafting — Forward-Phase Reply-Email Generation** | 3 | Analysis of `generate_reply_email()` at `app/email_utils.py:1103–1153`, including the 1000-iteration loop, random-string keyspace (≥ 26²⁰), `available_sl_email()` pre-check, and `include_sender_in_reverse_alias` branch. |
| **[AAP D1+C4] Sections 8–10 drafting — Check-Then-Act & Concurrency** | 4 | Analysis of `available_sl_email()` (`app/models.py:1425–1432`), non-atomic `Contact.create(commit=True)` (`app/contact_utils.py:92–103`), `aiosmtpd` threading model, Flask app-context-per-email pattern, and PostgreSQL `READ COMMITTED` isolation (`app/db.py:1–18`). |
| **[AAP D1] Sections 11–14 drafting — Routing Chain & Cross-Event Analysis** | 5 | Documentation of `contact → alias → user → mailbox` derivation chain, `disable_email_spoofing_check` fallback branch, alias transfer considerations, and E501–E504/E214 transient failure modes. |
| **[AAP D1+C5] Section 15 drafting — Runtime Values Observed (41-row table)** | 2 | Tabulation of every runtime value along the reply pipeline, derived by code inspection (since the AAP prohibits live instrumentation). |
| **[AAP D1] Sections 16–18 drafting — Risk Assessment, Evidence Table & Conclusions** | 2 | Risk matrix (architectural vs. practical), 106-row evidence table, and direct answers to each of the five investigation dimensions + charter question. |
| **[AAP R5] Live database schema verification & test execution** | 1 | Direct `psql` query against `\d contact` to confirm `ix_contact_reply_email` is non-unique btree and `uq_contact(alias_id, website_email)` is the only `UNIQUE` constraint. Full-suite pytest run (631 tests passing). |
| **[AAP R6] Citation verification** | 2 | Programmatic verification of all 229+ file:line citations against actual source code. One off-by-one (`1152 → 1153`) identified. |
| **[AAP C1-C5] QA revision cycle (3 commits)** | 2.5 | Commit `faf404a7` — 10 code-review findings (7 MAJOR + 3 MINOR): corrected domain-validation claim, documented `disable_email_spoofing_check` fallback, fixed 6-parameter signature for `handle_unknown_mailbox`, and others. Commit `f736041a` — replaced two oversized code blocks with prose paraphrases. Commit `aa23938c` — off-by-one line-number fix. |
| **Completed Total** | **36** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|---|---|---|
| [Path-to-production] Technical review of the analysis document by SimpleLogin engineer — verify analytical correctness, cross-check citations on a sampling basis, evaluate the three identified pathways | 2 | Medium |
| [Path-to-production] Stakeholder acceptance & PR merge workflow — sign-off from product/security lead and merge into target branch | 1 | Medium |
| [Path-to-production] Environment artifact cleanup — decide disposition of untracked `dump.rdb` Redis artifact (optional `.gitignore` update in a separate, non-AAP PR) | 0.5 | Low |
| [Path-to-production] Optional follow-up ticket scoping — formal separate ticket to evaluate the architectural question raised by the document (adding `UNIQUE(reply_email)`) — explicitly out of AAP scope | 0.5 | Low |
| **Remaining Total** | **4** | |

### 2.3 Scope Alignment Note

Every hour above traces to a specific AAP requirement (D1 = primary deliverable; C1–C5 = five investigation dimensions from §0.1.1; K1–K5 = key code findings from §0.5.3; R1–R7 = user-specified rules from §0.7.1) or a standard path-to-production activity for a documentation deliverable. No hours are allocated to work outside the AAP scope.

**Verification:**
- Section 2.1 sum: 10 + 3 + 1.5 + 2 + 1 + 2 + 3 + 4 + 5 + 2 + 2 + 1 + 2 + 2.5 = **36 hours** ✓
- Section 2.2 sum: 2 + 1 + 0.5 + 0.5 = **4 hours** ✓
- Section 2.1 + Section 2.2 = 36 + 4 = **40 hours** = Total Project Hours in Section 1.2 ✓

---

## 3. Test Results

All test results below originate from Blitzy's autonomous validation of this branch against the provisioned PostgreSQL (port 15432) and Redis (port 6379) instances. The environment was established by the setup agent; tests were executed by the final validator and re-confirmed during project-guide generation. No new tests were authored (the AAP is a documentation-only task that prohibits source modifications).

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---|---|---|---|---|---|---|
| **Targeted reply-pipeline tests** (`tests/test_contact_utils.py` + `tests/test_email_handler.py`) | pytest 7.x | 37 | 37 | 0 | Runtime-pass verified | Primary behavioral evidence for the reply-email resolution pipeline documented in the deliverable. Includes contact-creation IntegrityError paths, reply-phase DMARC/contact-replacement/canonical-address cases. |
| **Full unit + integration suite** (`tests/` excluding `test_mail_sender.py`) | pytest 7.x | 631 | 631 | 0 | Runtime-pass verified | Regression-free — zero new failures introduced by the documentation task (no source files modified). Execution time 161.94 s. |
| **Mail-sender tests** (`tests/test_mail_sender.py`) | pytest 7.x | 8 | 3 | 5 | N/A | 5 failures are **pre-existing, environment-specific** (Kubernetes raises `OSError: errno 99` where the assertion expects `ConnectionRefusedError`). Documented by the setup agent as unrelated to this task; fixing would require modifying `app/mail_sender.py`, which the AAP explicitly prohibits. |
| **Aggregate runnable totals** | pytest 7.x | **639** | **634** | **5 (pre-existing env-only)** | 99.2% runnable pass rate | All non-environmental tests pass. |

### 3.1 Test Execution Infrastructure

- **Runner:** `python -m pytest` (pytest 7.x) from the project-root virtualenv (`venv/`).
- **Config:** `pytest.ci.ini` enables `--cov` with `coverage.ini`.
- **Database fixture:** `tests/test.env` points to `postgresql://test:test@localhost:15432/test`; migrations applied to head `32f25cbf12f6`.
- **Redis fixture:** default on `localhost:6379`, confirmed `PONG`.

### 3.2 Interpretation

The primary behavioral claim in the deliverable — that the current code pipeline is safe under normal operating conditions — is consistent with the 100% pass rate on the targeted test files (`tests/test_contact_utils.py` and `tests/test_email_handler.py`). The architectural gaps called out in the deliverable (missing `UNIQUE`, TOCTOU window, `.first()` indeterminacy) are **by design not exercised** by the existing test suite — they are structural findings, not reproducible bugs, which is itself part of the deliverable's analytical contribution.

---

## 4. Runtime Validation & UI Verification

This project has no UI or HTTP-service surface of its own — it is a single markdown document committed to the repository. Runtime validation therefore focuses on the infrastructure required to validate the document's claims against a live codebase.

### 4.1 Infrastructure Health

- ✅ **PostgreSQL 15 on port 15432** — `pg_isready` returns "accepting connections"; `\d contact` query executed successfully; `alembic_version` is at head `32f25cbf12f6`.
- ✅ **Redis on port 6379** — `redis-cli ping` returns `PONG`.
- ✅ **Python virtualenv (`venv/`)** — Python 3.10.20; SQLAlchemy 1.3.24 (matching the version pinned in `pyproject.toml` and referenced throughout the deliverable).
- ✅ **SQLAlchemy ORM self-check** — all 631 runnable non-mail-sender tests importing models/handlers pass, implying that every code module referenced in the deliverable imports and compiles cleanly.

### 4.2 Deliverable Artifact Verification

- ✅ **Path correct** — `blitzy/documentation/app_2cd6ee777f8c.md` exists (owner root:root, 105,146 bytes, last modified Apr 17).
- ✅ **Filename matches branch** — `app_2cd6ee777f8c.md` corresponds to source branch `app_2cd6ee777f8c` per AAP §0.7.1.
- ✅ **Content volume** — 1,451 lines; 19 top-level sections; 93 subsections; 168 markdown table rows; 2 mermaid diagrams; 22 code blocks; 229+ file:line citations.
- ✅ **Schema claim verified against live DB** — `ix_contact_reply_email` is a btree index (non-unique); only `UNIQUE` constraint is `uq_contact(alias_id, website_email)`. Matches the document's Section 6.

### 4.3 AAP Compliance Verification

- ✅ **Zero source files modified** — `git diff --stat 2cd6ee77..HEAD` shows exactly one file added (`blitzy/documentation/app_2cd6ee777f8c.md`, +1,451 lines).
- ✅ **Branch on correct commit** — HEAD = `aa23938c` on `blitzy-e93b4109-d71b-4598-a7ea-3cda45247412`.
- ✅ **Working tree clean of tracked modifications** — only an untracked `dump.rdb` (Redis runtime artifact, not authored by the agent).

### 4.4 UI Screenshots

Not applicable — this project has no UI component. The deliverable is a markdown file consumed via `git show` / IDE preview / GitHub's rendered markdown.

---

## 5. Compliance & Quality Review

The AAP imposes several compliance requirements that are primarily structural (the deliverable's location, name, and evidence grounding) and procedural (repository must remain unmodified). All are validated below.

| Compliance Area | Requirement (AAP §) | Status | Evidence |
|---|---|---|---|
| Deliverable path | §0.1.1, §0.2.2, §0.7.1 — place at `blitzy/documentation/app_2cd6ee777f8c.md` | ✅ Pass | File exists at exact path (verified via `ls -la blitzy/documentation/`). |
| Deliverable name | §0.7.1 — must match source branch `app_2cd6ee777f8c` | ✅ Pass | Filename matches branch name verbatim. |
| No source modifications | §0.1.2, §0.7.1 (`SWE-AtlasQnA-Repo` rule) | ✅ Pass | `git diff --stat 2cd6ee77..HEAD` shows only the one added file; zero deletions or modifications of existing files. |
| No other new code | §0.1.2, §0.7.1 | ✅ Pass | Only one file added; no new scripts, modules, or tests created. |
| Evidence-based conclusions | §0.1.2, §0.7.2 | ✅ Pass | 229+ file:line citations; all verified; 106-row evidence table cross-referencing every source location. |
| Five investigation dimensions | §0.1.1 | ✅ Pass | Dimension 1 (derivation) → Section 7; Dimension 2 (resolution) → Section 2; Dimension 3 (cross-event) → Section 12; Dimension 4 (race/uniqueness) → Sections 6, 8, 9, 10; Dimension 5 (runtime values) → Section 15. |
| Runtime behavior grounding | §0.7.1 — build and run source code to analyze repository | ✅ Pass | Full pytest run (631 tests); live PostgreSQL schema inspection; SQLAlchemy ORM import validation. |
| Rationale for each conclusion | §0.7.1 | ✅ Pass | Every claim in Sections 1, 12, 13, 16, 18 supported by code references. |
| Distinguish theoretical vs. practical | §0.7.2 | ✅ Pass | Section 1.3, Section 16 (Risk Assessment), Section 18.4 explicitly separate architectural gaps from practical likelihood (birthday bound, throughput ceilings, etc.). |
| Cleanup of temporary tooling | §0.1.2 | ✅ Pass | No temporary scripts were created; no cleanup required. Working tree contains only the intentional deliverable + an untracked Redis runtime artifact. |
| Temporary scripts removed | §0.1.2 | ✅ Pass | None were created. |
| Citation accuracy | §0.7.1 — do not make assumptions, base on code | ✅ Pass | All 229+ citations verified; one off-by-one (1152 → 1153) found and fixed in commit `aa23938c`. |

### 5.1 Quality Review — Document Craftsmanship

- **Structure:** 19 numbered sections plus an investigation charter; sections are topically atomic (1 = exec summary, 2–4 = reply pipeline, 5 = ORM semantics, 6 = schema, 7–10 = forward + concurrency, 11 = routing chain, 12–14 = cross-event, 15 = runtime, 16 = risk, 17 = evidence, 18 = conclusions).
- **Traceability:** Every architectural claim in Sections 1, 12, 13, 14, 16, 18 is cross-referenced to Sections 2–11 where the underlying code is traced in detail.
- **Defensive hedging:** The deliverable explicitly distinguishes "under normal operating conditions (no)" from "architecturally (yes, through three narrow pathways)" — avoiding over-claiming either correctness or vulnerability.
- **Non-prescriptive:** Per AAP §0.6.2, the deliverable describes findings rather than prescribing fixes; it raises the `UNIQUE(reply_email)` question as an architectural observation, not a refactoring recommendation.

---

## 6. Risk Assessment

All risks below are categorized per PA3 and are assessed relative to the **next-step production-readiness path** (human review → merge), not against the AAP deliverable itself (which is complete).

| Risk | Category | Severity | Probability | Mitigation | Status |
|---|---|---|---|---|---|
| Reviewer disagreement on analytical conclusions in the deliverable | Technical | Low | Low | The deliverable provides 229+ file:line citations and a 106-row evidence table; any disagreement is resolvable by cross-checking the cited code. | Accepted |
| Stakeholder requests additional investigation dimensions beyond the five AAP-specified ones | Technical | Low | Low | If this occurs, scope as a separate AAP; the current deliverable fully satisfies the current charter. | Mitigated |
| Untracked `dump.rdb` Redis artifact is misinterpreted as repository pollution | Operational | Low | Low | It is in the working tree but explicitly untracked; the AAP requires the repository to remain unmodified, so adding it to `.gitignore` (itself a source modification) was correctly deferred. Handle separately post-merge. | Accepted |
| Pre-existing `tests/test_mail_sender.py` environment failures (`OSError: errno 99`) are attributed to this PR | Operational | Low | Low | Failures are documented by the setup agent as environment-specific (Kubernetes-level); `git log` shows no modification of `app/mail_sender.py` in this PR. | Mitigated |
| Regression introduced into non-targeted test paths | Technical | Low | Very Low | Zero source modifications means zero regression surface; 631 / 631 runnable non-mail-sender tests pass. | Mitigated |
| The document's architectural findings (missing `UNIQUE` constraint) get conflated with an implicit recommendation to fix them inside this PR | Operational | Low | Low | Section 18.6 and Section 16 explicitly frame the findings as defense-in-depth observations, not prescribed fixes; AAP §0.6.2 explicitly excludes refactoring from scope. | Mitigated |
| Concurrency-model inferences rely on assumptions about production deployment (thread count, load, SMTP throughput) | Technical | Low | Medium | Section 10 and Section 16 frame the race window as "architecturally possible" and compute explicit birthday bounds for the ≥ 26²⁰ keyspace; any production instance can verify its throughput against the documented thresholds. | Accepted |
| Document refers to SQLAlchemy 1.3.24-specific session semantics; future upgrade could change behavior | Technical | Low | Low | The deliverable explicitly cites SQLAlchemy 1.3.24 (pinned in `pyproject.toml`) as the observed version; any upgrade would require a companion review of Sections 5, 9, 10. | Accepted |
| Hypothetical alias-transfer edge case (`Alias.transfer_token`) mentioned in Section 18.6 as a more plausible misrouting cause than reply-email collision — could be misunderstood as in-scope finding | Integration | Low | Low | Section 18.6 explicitly frames the alias-transfer scenario as "more plausible explanation" for hypothetical production incidents, not as a current defect. | Mitigated |
| Security risk: the deliverable publicly (within repo) documents architectural gaps that could attract malicious exploitation | Security | Low | Very Low | The AAP mandates this investigation and placement. The keyspace bounds (≥ 26²⁰, birthday ≥ 10¹⁴) make exploitation astronomically impractical. The document is internal-repo, not public blog. | Accepted |

### 6.1 Overall Risk Profile

**Overall risk: LOW.** This is a pure documentation deliverable with zero source modifications, zero regression surface, and a well-scoped, evidence-grounded artifact. The only material path-to-production risks are human-review latency and potential stakeholder feedback requiring minor revisions, both of which are normal for any documentation deliverable.

---

## 7. Visual Project Status

### 7.1 Overall Project Hours

```mermaid
pie title Project Hours Breakdown
    "Completed Work" : 36
    "Remaining Work" : 4
```

**Color mapping:** Completed Work = Dark Blue (#5B39F3) · Remaining Work = White (#FFFFFF)

### 7.2 Remaining Work by Priority

```mermaid
pie title Remaining Hours by Priority
    "Medium Priority" : 3
    "Low Priority" : 1
```

### 7.3 Remaining Work by Category (Section 2.2 breakdown)

```mermaid
pie title Remaining Hours by Category
    "Technical Review" : 2
    "Stakeholder Acceptance & Merge" : 1
    "Environment Artifact Cleanup" : 0.5
    "Follow-up Scoping (out of AAP)" : 0.5
```

**Integrity check:** The "Remaining Work" slice (4 h) in Section 7.1 matches the Remaining Hours metric in Section 1.2 (4 h) and the sum of Section 2.2's Hours column (2 + 1 + 0.5 + 0.5 = 4 h). ✓

---

## 8. Summary & Recommendations

### 8.1 Achievements

This project delivered the single AAP-specified artifact — `blitzy/documentation/app_2cd6ee777f8c.md` — as a 1,451-line, 105,146-byte, code-grounded analysis that:

- Answers the charter question (can SimpleLogin's reply-email resolution misroute replies to the wrong user?) directly: **under normal operating conditions, no; architecturally, yes, through three narrow pathways.**
- Traces every line of the reply pipeline from `aiosmtpd.handle_DATA` (`email_handler.py:2289`) through the single deterministic contact lookup at `email_handler.py:986` to delivery via `sl_sendmail` at `email_handler.py:1220–1231`.
- Grounds every claim in ≥ 229 file:line citations, all verified against the source (one off-by-one fix applied).
- Documents the fundamental architectural finding — the absence of a `UNIQUE` constraint on `contact.reply_email` — and the application-level uniqueness guarantee (`available_sl_email` pre-check against a ≥ 26²⁰ keyspace) that currently compensates for it.
- Distinguishes architectural vulnerabilities from practical risks with explicit birthday-bound arithmetic (≥ 10¹⁴ candidates before ~50% collision in the 20-char subspace).
- Addresses every one of the AAP's five investigation dimensions (§0.1.1) and every one of its seven user-specified rules (§0.7.1).

### 8.2 Completion Metrics

| Metric | Value |
|---|---|
| Total Project Hours | 40 |
| Completed Hours | 36 |
| Remaining Hours | 4 |
| Completion Percentage | **90%** |
| AAP requirements completed | 100% (all D1, C1–C5, K1–K5, R1–R7 satisfied) |
| Source files modified | 0 (AAP-required) |
| Test suite regression | 0 |
| Citations verified | 229+ (100% of cited file:line references) |

### 8.3 Remaining Gaps & Critical Path to Production

The project is **90% complete**. The 4 remaining hours are entirely human-acceptance activities inherent to any documentation deliverable:

1. A SimpleLogin engineer reviews the document for analytical correctness (2 h).
2. Stakeholders sign off and the PR is merged (1 h).
3. Optional environment housekeeping around the untracked `dump.rdb` (0.5 h).
4. Optional follow-up ticket scoping for any architectural decisions inspired by the document — explicitly out of AAP scope (0.5 h).

There are **no outstanding AAP items**. The deliverable is content-complete and internally consistent.

### 8.4 Success Metrics

- **Deliverable exists at the exact specified path and name:** ✓
- **Covers all five AAP investigation dimensions:** ✓
- **Zero source modifications:** ✓
- **Evidence-grounded (every claim cited):** ✓
- **Live schema claim verification:** ✓
- **Full-suite test regression-free:** ✓ (631 / 631 runnable non-env-specific tests pass)
- **Distinguishes theoretical vs. practical:** ✓
- **Non-prescriptive per AAP §0.6.2:** ✓

### 8.5 Production Readiness Assessment

**Status: PRODUCTION-READY** for the deliverable itself. Documentation PRs have a narrower production-readiness definition than code PRs — the gates are (a) content completeness, (b) evidentiary grounding, (c) AAP rule compliance, and (d) zero regression in ambient code. All four gates pass:

- ✅ **Gate 1 (Content completeness):** 1,451 lines covering all five AAP dimensions.
- ✅ **Gate 2 (Evidentiary grounding):** 229+ file:line citations, 106-row evidence table, 41-row runtime-values table, live DB verification.
- ✅ **Gate 3 (AAP rule compliance):** Zero source modifications; correct path and filename; no temporary scripts left behind.
- ✅ **Gate 4 (Zero regression):** 631 / 631 runnable non-env-specific tests pass; `aa23938c`'s git diff contains exactly one file.

The project is ready for human technical review and merge.

---

## 9. Development Guide

This guide describes how to build, run, and inspect the SimpleLogin application in a way that matches the environment used to validate the deliverable — including running the test suite against the live PostgreSQL and Redis instances and reproducing the `contact` table schema verification documented in Section 6 of the deliverable.

### 9.1 System Prerequisites

| Requirement | Minimum Version | Notes |
|---|---|---|
| Python | 3.10 | Per `pyproject.toml` `python = "^3.10"`. Session used Python 3.10.20. |
| PostgreSQL | 12+ | Any PostgreSQL version supporting `READ COMMITTED` isolation; session used Postgres on port 15432. |
| Redis | 5+ | For session/cache; session used default port 6379. |
| Poetry | 1.x | Preferred tool for dependency installation from `pyproject.toml` / `poetry.lock`. Alternative: `pip install` from requirements extracted via `poetry export`. |
| OS | Linux (container) | Session ran on Debian-based container; macOS works with equivalent PG/Redis tooling. |
| Disk | ~500 MB | `venv/` alone is ≈ 370 MB; repository total ≈ 416 MB. |

### 9.2 Environment Setup

The repository ships an already-provisioned `venv/` directory at the project root. If starting from scratch, either use the provided venv or recreate it.

**Option A — Use the provided virtualenv (fastest, matches validated state):**

```bash
cd /tmp/blitzy/app/blitzy-e93b4109-d71b-4598-a7ea-3cda45247412_0ea6de
source venv/bin/activate
python --version   # Should report Python 3.10.20
pip show SQLAlchemy | head -2   # Should report Version: 1.3.24
```

**Option B — Recreate the virtualenv with Poetry:**

```bash
cd /tmp/blitzy/app/blitzy-e93b4109-d71b-4598-a7ea-3cda45247412_0ea6de
python3.10 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install poetry
poetry install --no-interaction
```

### 9.3 Environment Variables

Tests use `tests/test.env` and the app uses `.env` (or env vars directly). The validated test configuration:

```bash
# tests/test.env (already present in the repo)
DB_URI=postgresql://test:test@localhost:15432/test
EMAIL_DOMAIN=sl.local
OTHER_ALIAS_DOMAINS=["d1.test", "d2.test", "sl.local"]
SUPPORT_EMAIL=support@sl.local
NOT_SEND_EMAIL=true
FLASK_SECRET=secret
```

A full `.env` template is available at `example.env`; required minimum variables for running the app locally are `URL`, `EMAIL_DOMAIN`, `SUPPORT_EMAIL`, `DB_URI`, and `FLASK_SECRET`.

### 9.4 Infrastructure Services

**PostgreSQL** — start and confirm:

```bash
pg_isready -h localhost -p 15432
# Expected: localhost:15432 - accepting connections
```

**Redis** — start and confirm:

```bash
redis-cli -h 127.0.0.1 -p 6379 ping
# Expected: PONG
```

### 9.5 Database Setup

Apply migrations to head:

```bash
cd /tmp/blitzy/app/blitzy-e93b4109-d71b-4598-a7ea-3cda45247412_0ea6de
source venv/bin/activate
# The test DB already has migrations applied; to manually verify:
PGPASSWORD=test psql -h localhost -p 15432 -U test -d test \
  -c "SELECT version_num FROM alembic_version;"
# Expected: 32f25cbf12f6
```

### 9.6 Reproducing the Deliverable's Schema Verification

Section 6 of the deliverable depends on a live PostgreSQL claim about the `contact` table's indexes. To reproduce:

```bash
PGPASSWORD=test psql -h localhost -p 15432 -U test -d test -c "\d contact"
```

Expected output (abridged):

```text
Indexes:
    "forward_email_pkey" PRIMARY KEY, btree (id)
    "ix_contact_alias_id" btree (alias_id)
    "ix_contact_reply_email" btree (reply_email)       <-- non-unique
    "uq_contact" UNIQUE CONSTRAINT, btree (alias_id, website_email)
```

Observe that `ix_contact_reply_email` has no "UNIQUE CONSTRAINT" qualifier — this is the core schema observation the deliverable rests on.

### 9.7 Running the Test Suite

**Targeted reply-pipeline tests (primary behavioral evidence for the deliverable):**

```bash
cd /tmp/blitzy/app/blitzy-e93b4109-d71b-4598-a7ea-3cda45247412_0ea6de
source venv/bin/activate
python -m pytest tests/test_contact_utils.py tests/test_email_handler.py --tb=short -q
# Expected: 37 passed
```

**Full regression suite (excludes pre-existing env-specific mail_sender failures):**

```bash
python -m pytest tests/ --ignore=tests/test_mail_sender.py --tb=short -q
# Expected: 631 passed
```

**Full suite with coverage (slow, uses the CI config):**

```bash
python -m pytest tests/ --ignore=tests/test_mail_sender.py -c pytest.ci.ini
# Produces htmlcov/ coverage report
```

### 9.8 Viewing the Deliverable

The deliverable is a single markdown file. View it directly:

```bash
less blitzy/documentation/app_2cd6ee777f8c.md
```

Or render to HTML via any markdown renderer (GitHub, pandoc, VS Code preview).

### 9.9 Reviewing the Branch Diff

```bash
# What files changed on this branch?
git diff --stat 2cd6ee77..HEAD
# Expected: exactly one file changed: blitzy/documentation/app_2cd6ee777f8c.md (+1451)

# Commit history for this branch
git log --oneline 2cd6ee77..HEAD
# Expected: 4 commits, all with "docs:" prefix

# Verify no source files were modified
git diff --name-status 2cd6ee77..HEAD | grep -v "^A" | grep -v "^$"
# Expected: no output (only 'A' adds, zero 'M' modifications)
```

### 9.10 Common Issues & Troubleshooting

| Symptom | Cause | Resolution |
|---|---|---|
| `psql: FATAL: password authentication failed for user "postgres"` | Using wrong DB credentials — session's test DB uses `test:test`, not `postgres`. | Use `PGPASSWORD=test psql -h localhost -p 15432 -U test -d test ...`. |
| `tests/test_mail_sender.py` reports 5 failures with `OSError: [Errno 99]` | Pre-existing, environment-specific — Kubernetes raises a different errno than the assertion expects. | Not caused by this project; documented by setup agent. Excluded from the regression-gated run using `--ignore=tests/test_mail_sender.py`. Fixing requires modifying `app/mail_sender.py`, which the AAP prohibits. |
| `dump.rdb` appears in the working tree | Redis runtime persistence artifact written by the test-fixture Redis instance. | Leave untracked; it was not created by the agent and is already excluded from commits. Optional follow-up: add to `.gitignore` in a separate non-AAP PR. |
| Migration not at head | Fresh DB or stale volume. | Run `flask db upgrade` (or equivalent Alembic command) to bring `alembic_version` to `32f25cbf12f6`. |
| `pip install` fails on a Poetry dependency | Poetry lock file mismatch. | Prefer Option A above (use the pre-provisioned `venv/`). |
| Test run says "DeprecationWarning: setDaemon() is deprecated" | `gnupg` upstream deprecation. | Benign; does not affect pass/fail. |

### 9.11 Reference Locations Used in the Deliverable

If reviewing the deliverable and cross-checking citations, the primary files are:

```bash
# Reply-phase entry and routing
less +966 email_handler.py          # handle_reply
less +2195 email_handler.py          # is_reverse_alias dispatch
less +1156 app/email_utils.py        # is_reverse_alias definition

# Forward-phase reply-email generation
less +1103 app/email_utils.py        # generate_reply_email
less +42 app/contact_utils.py        # create_contact
less +180 email_handler.py           # get_or_create_contact

# Data model
less +1863 app/models.py             # Contact class start
less +1899 app/models.py             # reply_email column — index=True, no unique
less +1425 app/models.py             # available_sl_email
less +82 app/models.py               # ModelMixin.get_by

# Schema
less migrations/versions/2021_071310_78403c7b8089_.py
less migrations/versions/5fa68bafae72_.py

# DB session / concurrency
less app/db.py

# Normalization
less +25 app/email_validation.py     # normalize_reply_email
less +41 app/utils.py                # random_string
```

---

## 10. Appendices

### Appendix A — Command Reference

| Command | Purpose |
|---|---|
| `source venv/bin/activate` | Activate the pre-provisioned Python 3.10 virtualenv |
| `pg_isready -h localhost -p 15432` | Check PostgreSQL availability |
| `redis-cli -h 127.0.0.1 -p 6379 ping` | Check Redis availability |
| `PGPASSWORD=test psql -h localhost -p 15432 -U test -d test -c "\d contact"` | Inspect the `contact` table schema (reproduces Section 6 of the deliverable) |
| `PGPASSWORD=test psql -h localhost -p 15432 -U test -d test -c "SELECT version_num FROM alembic_version;"` | Confirm migration head (`32f25cbf12f6`) |
| `python -m pytest tests/test_contact_utils.py tests/test_email_handler.py --tb=short -q` | Run targeted reply-pipeline tests (37 tests) |
| `python -m pytest tests/ --ignore=tests/test_mail_sender.py --tb=short -q` | Run full regression suite (631 tests) |
| `git diff --stat 2cd6ee77..HEAD` | Confirm only the deliverable file was added |
| `git log --oneline 2cd6ee77..HEAD` | List the 4 documentation commits |
| `wc -l blitzy/documentation/app_2cd6ee777f8c.md` | Confirm deliverable line count (1,451) |
| `grep -c "^## " blitzy/documentation/app_2cd6ee777f8c.md` | Count top-level sections (19) |

### Appendix B — Port Reference

| Service | Port | Notes |
|---|---|---|
| PostgreSQL | 15432 | Test DB; credentials `test:test`, database `test` per `tests/test.env` |
| Redis | 6379 | Default; no authentication |
| Flask dev server (not used in this task) | 7777 | Documented in `example.env` as `URL=http://localhost:7777` |
| SMTP inbound (aiosmtpd, not run in this task) | configurable | Referenced in `email_handler.py:2289` handler |

### Appendix C — Key File Locations

| File | Path | Purpose |
|---|---|---|
| **Deliverable** | `blitzy/documentation/app_2cd6ee777f8c.md` | The sole AAP artifact (1,451 lines / 105,146 bytes) |
| Core inbound handler | `email_handler.py` (2,404 lines) | SMTP receive → forward/reply dispatch |
| ORM models | `app/models.py` | `Contact`, `Alias`, `User`, `Mailbox`, `SLDomain`, `ModelMixin` |
| Contact creation utility | `app/contact_utils.py` | `create_contact()` with `IntegrityError` handling |
| Email-string utilities | `app/email_utils.py` | `generate_reply_email()`, `is_reverse_alias()` |
| Email normalization | `app/email_validation.py` | `normalize_reply_email()` |
| Random-string utilities | `app/utils.py` | `random_string()`, `convert_to_id()` |
| DB session | `app/db.py` | Scoped session, default `READ COMMITTED` |
| `reply_email` index migration | `migrations/versions/2021_071310_78403c7b8089_.py` | Creates `ix_contact_reply_email` with `unique=False` |
| Original `forward_email` table migration | `migrations/versions/5fa68bafae72_.py` | Historical schema context for `reply_email` column |
| Test config | `pytest.ci.ini`, `coverage.ini`, `tests/test.env` | CI coverage + DB pointer |
| Dependency manifest | `pyproject.toml`, `poetry.lock` | SQLAlchemy 1.3.24, psycopg2-binary, aiosmtpd, etc. |

### Appendix D — Technology Versions

| Technology | Version | Source |
|---|---|---|
| Python | 3.10.20 | Session venv; `pyproject.toml` constraint is `^3.10` |
| SQLAlchemy | 1.3.24 (pinned) | `pyproject.toml` line `SQLAlchemy = "1.3.24"` |
| psycopg2-binary | ≥ 2.9.3 (caret) | `pyproject.toml` |
| Flask | ≥ 1.1.2 (caret) | `pyproject.toml` |
| Flask-Migrate | ≥ 2.5.3 (caret) | `pyproject.toml` — wraps Alembic |
| aiosmtpd | ≥ 1.2 (caret) | `pyproject.toml` |
| email_validator | ≥ 1.1.1 (caret) | `pyproject.toml` |
| flanker | ≥ 0.9.11 (caret) | `pyproject.toml` — email-address parsing |
| arrow | ≥ 0.16.0 (caret) | `pyproject.toml` — datetime |
| sqlalchemy-utils | ≥ 0.36.8 (caret) | `pyproject.toml` — `ArrowType` column |
| python-gnupg | ≥ 0.4.6 (caret) | `pyproject.toml` — PGP encryption |
| PGPy | 0.5.4 (pinned) | `pyproject.toml` — PGP fallback |
| dkimpy | ≥ 1.0.5 (caret) | `pyproject.toml` — DKIM signing |
| newrelic | 8.8.0 (pinned) | `pyproject.toml` — APM |
| pytest | ≥ 7.0.0 (caret) | `pyproject.toml` dev-dependency |
| pytest-cov | ≥ 3.0.0 (caret) | `pyproject.toml` dev-dependency |
| PostgreSQL | 12+ | Runtime; test instance on port 15432 |
| Redis | 5+ | Runtime; test instance on port 6379 |
| Alembic migration head | `32f25cbf12f6` | Verified via `SELECT version_num FROM alembic_version` |

### Appendix E — Environment Variable Reference

| Variable | Purpose | Test Value |
|---|---|---|
| `DB_URI` | PostgreSQL connection URI | `postgresql://test:test@localhost:15432/test` |
| `EMAIL_DOMAIN` | Primary domain for alias generation — used in reply-domain check at `email_handler.py:977` | `sl.local` |
| `OTHER_ALIAS_DOMAINS` | Additional domains usable as aliases | `["d1.test", "d2.test", "sl.local"]` |
| `SUPPORT_EMAIL` | Return address for transactional emails | `support@sl.local` |
| `NOT_SEND_EMAIL` | If `true`, only log emails rather than send them | `true` |
| `FLASK_SECRET` | Flask session-signing secret | `secret` (tests only) |
| `URL` | Base URL for the app | `http://localhost` (tests) / `http://localhost:7777` (local dev) |
| `LOCAL_FILE_UPLOAD` | Use local filesystem for uploads instead of S3 | `1` (tests) |
| `ENFORCE_SPF` | If set, enforce SPF during reply phase (see `email_handler.py:1036–1040`) | unset (tests) |
| `DKIM_PRIVATE_KEY_PATH` | DKIM signing key location | `local_data/dkim.key` |

Refer to `example.env` for the full list (~50 variables).

### Appendix F — Developer Tools Guide

| Tool | Usage |
|---|---|
| **Poetry** | `poetry install`, `poetry run pytest` — preferred for dependency management |
| **pytest** | Test runner; use `-q` for quiet output, `--tb=short` for abbreviated tracebacks, `--ignore=<file>` to skip known env-failing tests |
| **psql** | PostgreSQL CLI — `PGPASSWORD=test psql -h localhost -p 15432 -U test -d test` |
| **redis-cli** | Redis CLI — `redis-cli -h 127.0.0.1 -p 6379` |
| **flask db** | Alembic wrapper — `flask db upgrade` / `flask db current` / `flask db history` |
| **ruff** | Linter per `pyproject.toml` `[tool.ruff]` — excludes `.venv`, `migrations`, `app/events/generated` |
| **black** | Formatter per `pyproject.toml` `[tool.black]` — target Python 3.10 |
| **pylint** | Static analyzer — configured via `.pylintrc` |
| **djlint** | Jinja template linter — configured in `pyproject.toml` |
| **git** | `git diff --stat 2cd6ee77..HEAD` to verify only one file changed; `git log --oneline 2cd6ee77..HEAD` to list commits |

### Appendix G — Glossary

| Term | Definition |
|---|---|
| **Reverse alias** | The random reply-email address generated for a `(alias, website_email)` pair during the forward phase, stored in `contact.reply_email`. When an external sender replies to this address, SimpleLogin resolves it back to the original alias and user. |
| **Forward phase** | The inbound-email pipeline in which an external sender emails an alias (e.g., `alice.s2k@simplelogin.co`); SimpleLogin looks up the alias, creates or finds a `Contact` for the sender, generates the reverse alias, and forwards the email to the alias owner's real mailbox. |
| **Reply phase** | The inbound-email pipeline in which the alias owner replies *from* their real mailbox *to* the reverse alias; SimpleLogin looks up the `Contact` by `reply_email`, re-derives the alias and user, authorizes the sender's mailbox, rewrites headers, and delivers the reply to the external sender. |
| **TOCTOU** | Time-of-check-to-time-of-use. The race window between a check (e.g., `available_sl_email(...)`) and a subsequent act (e.g., `Contact.create(...)`), during which another thread or session can invalidate the assumption from the check. |
| **READ COMMITTED** | PostgreSQL's default transaction isolation level. Committed writes by other transactions are visible to subsequent reads in the current transaction, but the current transaction's own writes are not blocked by a concurrent transaction's same-row writes unless one holds a row lock. |
| **UNIQUE constraint** | Database-level enforcement that no two rows in a table share the same value for a given column (or combination of columns). `contact.reply_email` has a B-tree *index* but no `UNIQUE` constraint — a key finding of the deliverable. |
| **`.first()` indeterminacy** | Behavior of `SQLAlchemy.Query.first()` when multiple rows match and no `ORDER BY` is specified — the returned row is planner-dependent and not guaranteed stable over time. Relevant only in the pathological case where duplicate `reply_email` rows exist. |
| **aiosmtpd** | Asyncio-based SMTP server implementation. Runs SimpleLogin's inbound SMTP handler via `MailHandler` on a dedicated thread; each `handle_DATA` call creates a fresh Flask app context and SQLAlchemy scoped session. |
| **Flask app context** | Flask's per-request (or per-operation) container for application-scoped state, including the SQLAlchemy session. Each inbound email gets its own context via `create_light_app().app_context()` at `email_handler.py:2352`. |
| **Scoped session (SQLAlchemy)** | `scoped_session(sessionmaker(bind=connection))` in `app/db.py:1–18` — one logical session per thread or app context, tied to a connection from the SQLAlchemy pool. |
| **Birthday bound** | The keyspace size at which random sampling produces a ~50% probability of collision, approximately `√(2N)` where `N` is the total keyspace. For the ≥ 26²⁰ reply-email keyspace, this is ≥ 10¹⁴ — far beyond realistic email throughput. |
| **E501 / E502 / E503 / E504 / E214** | SimpleLogin SMTP response codes defined in `app/email/status.py`: E501 = unknown reply-domain; E502 = contact not found / user inactive; E503 = alias domain invalid; E504 = user disabled; E214 = unauthorized mailbox (soft-accepted 250 to suppress backscatter). |
| **SL domain** | A domain SimpleLogin operates (`SLDomain` model); only some have `use_as_reverse_alias=True` in the generator, though the reply-phase lookup uses a plain domain filter without that flag. |

---

## Cross-Section Integrity — Pre-Submission Checklist

| Rule | Status | Evidence |
|---|---|---|
| Rule 1 — Remaining hours identical in Sections 1.2, 2.2, and 7 | ✓ Pass | Section 1.2 metrics: 4 h · Section 2.2 sum: 2 + 1 + 0.5 + 0.5 = 4 h · Section 7.1 pie "Remaining Work": 4 |
| Rule 2 — Section 2.1 + Section 2.2 = Total Project Hours | ✓ Pass | 36 + 4 = 40 h (matches Section 1.2 Total) |
| Rule 3 — All tests from Blitzy's autonomous validation logs | ✓ Pass | All 631/631 runnable + 37/37 targeted + 3/8 mail_sender counts come from the final validator's pytest runs in this session (reconfirmed during guide generation) |
| Rule 4 — Access issues validated | ✓ Pass | Section 1.5 explicitly states "No access issues identified"; PostgreSQL, Redis, venv all confirmed operational |
| Rule 5 — Colors (Completed = Dark Blue #5B39F3, Remaining = White #FFFFFF) | ✓ Pass | Color legend stated in Sections 1.2 and 7.1 |
| Completion % calculation | ✓ Pass | Section 1.2 states exact formula 36 / (36 + 4) × 100 = 90% |
| Section 2.1 completed rows sum | ✓ Pass | 10 + 3 + 1.5 + 2 + 1 + 2 + 3 + 4 + 5 + 2 + 2 + 1 + 2 + 2.5 = 36 (matches Section 1.2 Completed Hours) |
| Section 2.2 remaining rows sum | ✓ Pass | 2 + 1 + 0.5 + 0.5 = 4 (matches Section 1.2 Remaining Hours) |
| Section 7 pie chart hours | ✓ Pass | "Completed Work" = 36 · "Remaining Work" = 4 (matches Section 1.2 exactly) |
| Section 8 narrative references correct % | ✓ Pass | Section 8.2 metrics table states 90% exactly; Section 8.3 uses "90% complete" verbatim |
| No conflicting statements across sections | ✓ Pass | Verified — all % and hour mentions aligned at 90% / 40 h / 36 h / 4 h |

**All integrity checks pass.** The project guide is ready for submission.
