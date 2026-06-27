# Blitzy Project Guide

**Project:** SimpleLogin Inbound Alias Reply Mis-Routing — Code-Grounded Investigation
**Branch:** `blitzy-38f8a2c9-c381-44b3-ba1d-3ed9ef6c6431` · **HEAD:** `2f9fccf9` · **Base:** `2cd6ee77`
**Deliverable:** `blitzy/documentation/app_2cd6ee777f8c.md`

> **Legend (Blitzy brand colors):** <span style="color:#5B39F3">■</span> **Completed / AI Work** `#5B39F3` (Dark Blue) · □ **Remaining / Not Completed** `#FFFFFF` (White) · <span style="color:#B23AF2">Headings/Accents</span> `#B23AF2` · <span style="color:#A8FDD9">Highlight</span> `#A8FDD9`

---

## 1. Executive Summary

### 1.1 Project Overview

This project produced a single, code-grounded investigative analysis of SimpleLogin's inbound alias **reply-handling** pipeline. The target audience is SimpleLogin's backend engineers and security reviewers. The investigation answers six questions (O1–O6): which component receives an inbound reply, how the reverse-alias is "recognized," how it resolves to a contact/alias/user, which `user_id` the reply is attributed and forwarded to, the actual data flow from a **live execution trace**, and the most likely point where wrong-user routing originates. The business impact is risk reduction: it isolates a confidentiality-relevant routing defect to a single lookup and recommends (without implementing) a remediation. The technical scope is the reply path of a ~2,400-line SMTP handler and its ORM resolution chain.

### 1.2 Completion Status

The project is **92.3% complete** on an AAP-scoped, hours-based basis. All autonomous investigative work is delivered; the remaining effort is human review and triage (path-to-production).

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieOuterStrokeWidth':'2px','pieSectionTextColor':'#B23AF2','pieTitleTextSize':'16px','pieLegendTextSize':'13px'}}}%%
pie showData
    title AAP-Scoped Completion — 92.3% Complete
    "Completed Work (AI)" : 24
    "Remaining Work" : 2
```

| Metric | Value |
|--------|-------|
| **Total Hours** | **26.0 h** |
| **Completed Hours (AI + Manual)** | **24.0 h** (24.0 AI + 0.0 Manual) |
| **Remaining Hours** | **2.0 h** |
| **Percent Complete** | **92.3%**  →  24.0 / (24.0 + 2.0) × 100 |

### 1.3 Key Accomplishments

- ✅ Built and ran the SimpleLogin development stack (Python 3.10.18, PostgreSQL 13, Redis 6, Poetry) with migrations at head (`32f25cbf12f6`).
- ✅ Simulated an inbound reply via `email_handler.handle()` in `NOT_SEND_EMAIL=true` print mode and captured a **live execution trace** end-to-end (`250 Message accepted for delivery`).
- ✅ Answered all six objectives (O1–O6) with rationale and `file:line` citations across a 503-line / ~6,200-word document.
- ✅ Localized wrong-user routing to a **single lookup** — `Contact.get_by(reply_email=...)` at `email_handler.py:L986` — and proved four compounding structural conditions.
- ✅ **OBSERVED demonstration** of the defect: a duplicate `reply_email` across two users persisted with **no `IntegrityError`**, and the resolver collapsed two users onto one (`contact.id=178`/`179`, `user_id=479`/`480`).
- ✅ Discovered an additional **confidentiality leak**: on the rejection path, one user's mailbox address was disclosed to an unrelated user.
- ✅ Documented a remediation **recommendation only** (UNIQUE constraint + deterministic lookup) — deliberately not implemented, per the task rule.
- ✅ Left the source tree **git-clean**: exactly one file added, **zero** source files modified; all temporary artifacts removed.

### 1.4 Critical Unresolved Issues

> The **deliverable itself has no unresolved issues** — it is complete and validated. The item below is the **defect the investigation surfaced in the SimpleLogin system**, which is intentionally **not fixed** (out of AAP scope) and is recorded here for owner action.

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| Wrong-user reply routing at `Contact.get_by(reply_email=...)` (`email_handler.py:L986`); cross-user `reply_email` collision mis-attributes/mis-delivers a reply and can leak one user's address to another | High — confidentiality/privacy (cross-user data exposure) + mis-routing | SimpleLogin maintainers / Security | Pending triage (HT-2, 0.5 h) |

### 1.5 Access Issues

**No access issues identified.** The repository, the development stack (PostgreSQL 13, Redis 6), and the test harness were all reachable; the email handler imported and executed cleanly under `CONFIG=tests/test.env`. One environment nuance is noted for transparency (not an access blocker):

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-----------------|----------------|-------------------|-------------------|-------|
| Container `/app/local_data/dkim.key` | Read (runtime) | The container's runtime key is a drifted PKCS#8 key `dkimpy` cannot parse (pre-existing env artifact); the committed repo key is valid PKCS#1 | Resolved/worked around — delegated DKIM via `RSPAMD_SIGN_DKIM=1`; independent of the routing finding | Environment owner |

### 1.6 Recommended Next Steps

1. **[High]** SME technical review & sign-off of the investigation — validate the wrong-user thesis, the four structural conditions, and the OBSERVED traces against the cited source (HT-1, 1.5 h).
2. **[Medium]** Remediation triage — accept the analysis-only recommendation and open a tracked ticket for the proposed UNIQUE constraint + deterministic lookup (HT-2, 0.5 h).
3. **[Low / Future, out of scope]** Plan a separate remediation project: add the `UNIQUE` constraint + Alembic migration, de-duplicate existing rows, and canonicalize `reply_email` storage/lookup (FR-1…FR-3, ~18–36 h — **not** part of this project's hours).
4. **[Low / Future, out of scope]** Security data audit: scan production for existing cross-user `reply_email` collisions and assess prior exposure (FR-3).

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

All completed components trace to AAP-specified investigative work. **Total = 24.0 h** (matches Completed Hours in §1.2).

| Component | Hours | Description |
|-----------|-------|-------------|
| Development environment build & run (R1) | 3.0 | Stand up Python 3.10.18 / PostgreSQL 13 / Redis 6 / Poetry; apply Alembic migrations to head; achieve clean `email_handler` import under `CONFIG=tests/test.env`. |
| Static analysis of reply pipeline & resolution chain — O1–O3 (R3–R5) | 4.0 | Trace `handle()` → `is_reverse_alias` → `handle_reply()` → `Contact.get_by` → `alias` → `user`; document mailbox selection. |
| Root-cause analysis — O6 (R8) | 3.0 | Identify and evidence the four compounding structural conditions (unordered `.first()`, no DB uniqueness, racy app check, normalization/case divergence). |
| Live trace simulation & wrong-user reproduction — O5 (R2, R7) | 5.0 | Build the `Envelope`+`handle()` harness in print mode; reproduce the cross-user `reply_email` collision and shadowed-user rejection; discover the confidentiality leak; capture raw log lines. |
| Authoring the cited analysis — O4 + deliverable (R6, R9) | 6.0 | Write the 503-line / ~6,200-word document: executive summary, O1–O6 sections, §9 recommendation, Appendix A–E, sequence diagram. |
| Citation verification & QA hardening (R10) | 2.5 | Verify 262 citations (94/94 exact); enforce OBSERVED-vs-INFERRED discipline; address three review rounds (CR-3, QA, precision edits). |
| Cleanup & git hygiene (R11–R13) | 0.5 | Delete temporary scripts (host + container); roll back the DB transaction; verify git-clean and zero source modifications; restore container. |
| **Total Completed** | **24.0** | |

### 2.2 Remaining Work Detail

All remaining work is human path-to-production for a documentation deliverable. **Total = 2.0 h** (matches Remaining Hours in §1.2 and §7).

| Category | Hours | Priority |
|----------|-------|----------|
| SME technical review & sign-off of investigation findings (HT-1 / P1) | 1.5 | High |
| Remediation triage decision / open tracking ticket — decision only, not implementation (HT-2 / P2) | 0.5 | Medium |
| **Total Remaining** | **2.0** | |

> **Integrity:** §2.1 (24.0) + §2.2 (2.0) = **26.0 h** = Total Project Hours in §1.2. ✔

### 2.3 Out-of-Scope Future Remediation (NOT counted in the 26.0 h total)

The AAP explicitly forbids implementing a fix; the items below address the surfaced system defect and are sized **for stakeholder awareness only**. They are a separate future engagement and do **not** affect the 92.3% completion or the 26.0 h total.

| ID | Future Task | Priority | Rough Hours (excluded) |
|----|-------------|----------|------------------------|
| FR-1 | Add `UNIQUE` constraint on `Contact.reply_email` + Alembic migration; de-duplicate existing rows; wire `create_contact` `IntegrityError` recovery to the new constraint | High (future) | 8–16 |
| FR-2 | Canonicalize `reply_email` storage + lookup (case-folded), reconciling `normalize_reply_email` with `canonicalize_email`/`sanitize_email`; add regression tests | Medium (future) | 6–12 |
| FR-3 | Security data audit: detect existing cross-user `reply_email` collisions; assess prior mis-attribution/mis-delivery exposure | Medium (future) | 4–8 |
| | **Future total (separate project, excluded)** | | **~18–36** |

---

## 3. Test Results

All entries below originate **exclusively** from Blitzy's autonomous validation logs for this task (Final Validator, all five gates PASS). Coverage % is reported as **N/A** for a documentation deliverable; the relevant assurance metrics are pass rate, live-trace reproduction, and citation accuracy.

| Test Category | Framework | Total | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------|--------|--------|-----------|-------|
| Reply-handler module tests | pytest 7.x | 23 | 23 | 0 | N/A | `tests/test_email_handler.py` run under the canonical committed DKIM key (PKCS#1). |
| Full project regression baseline | pytest 7.x | 637 | 637 | 0 | N/A | Project suite baseline captured at setup; the 23 reply tests reconcile with this baseline. |
| Live-trace OBSERVED-claim reproduction | Custom trace harness (mirrors `tests/conftest.py` external-transaction pattern + `tests/test_email_handler.py:L165-188`) | 12 | 12 | 0 | N/A | Every OBSERVED claim in Appendix A independently reproduced (intake, recognition log, resolution, attribution, delivery, duplicate persists w/o `IntegrityError`, shadowed-user `E214`, confidentiality leak, print-branch send, etc.). |
| Citation accuracy verification | `file:line` exact-match audit | 94 | 94 | 0 | N/A | 100% of distinct cited `file:line` anchors verified against source across 16 files; 6 primary cited files byte-identical (md5) working-tree == source `2cd6ee77` == container `/app`. |

**Aggregate:** 766 checks executed (23 + 637 + 12 + 94), **766 passed / 0 failed**. No tests were authored or modified by this task (no test files added; CI unchanged) — the reply-handler and regression suites are the project's own tests executed during validation.

---

## 4. Runtime Validation & UI Verification

**UI verification: Not applicable.** This is a backend investigation (AAP §0.5.3); there is no user interface, no Figma frames, and no front-end surface in scope.

**Runtime validation** (observed on the running dev stack):

- ✅ **Operational** — `MailHandler.handle_DATA` → `handle()` receives the envelope/message (intake log at `email_handler.py:L1980`).
- ✅ **Operational** — Reverse-alias recognition fires the "Reply phase" log at `email_handler.py:L2196` (the user's "alias recognized" line).
- ✅ **Operational** — Decisive resolution `Contact.get_by(reply_email=...)` (`L986`) → `alias` (`L994`) → `user` (`L1004`); attribution `EmailLog.create(user_id=contact.user_id)` (`L1042-L1050`).
- ✅ **Operational** — Delivery path reaches the `NOT_SEND_EMAIL=true` print branch (`app/mail_sender.py:L130-L137`); `handle()` returns `250 Message accepted for delivery`.
- ✅ **Operational (defect demonstrated)** — Duplicate `reply_email` across users 479/480 persisted with **no `IntegrityError`**; resolver returned a single arbitrary row; shadowed user's reply recognized then rejected under the wrong user's alias with `250 SL E214`.
- ✅ **Operational (security finding demonstrated)** — Cross-user collision disclosed user B's mailbox address to user A via the unauthorized-use alert (`app/mail_sender.py:L131`).
- ⚠ **Partial (environment)** — DKIM signing under the container's drifted PKCS#8 key raises `KeyFormatError` *after* the routing decision; bypassed via `RSPAMD_SIGN_DKIM=1` to deterministically reach the print branch. Immaterial to routing (signing at `L1220-1221` occurs after the decisive lookup at `L986`).
- ▫ **Not applicable** — External API integrations: none in the reply path within scope.

---

## 5. Compliance & Quality Review

AAP deliverables and task rules cross-mapped to Blitzy quality/compliance benchmarks. Fixes applied during autonomous validation are noted; there are no outstanding compliance items for the deliverable.

| Benchmark / Rule | Status | Progress | Evidence / Notes |
|------------------|--------|----------|------------------|
| Documentation-only — no source modification | ✅ Pass | 100% | `git diff 2cd6ee77..HEAD` = 1 added file; 6 primary source files md5-verified UNCHANGED. |
| Code is the source of truth — every claim cited `file:line` | ✅ Pass | 100% | 262 citations; 94/94 verified exact (Gate 2). |
| Build & run the source (live trace, not static-only) | ✅ Pass | 100% | Gate 3/4; `handle()` executed end-to-end; OBSERVED claims reproduced. |
| Provide rationale / thinking | ✅ Pass | 100% | Per-objective rationale; §8 "why most likely"; §9 rationale. |
| OBSERVED vs INFERRED clearly labeled | ✅ Pass | 100% | Appendix A ledger maps each claim to evidence. |
| Remediation recommended, NOT implemented | ✅ Pass | 100% | §9 records the fix as "deliberately NOT implemented." |
| Correct deliverable name & location | ✅ Pass | 100% | `blitzy/documentation/app_2cd6ee777f8c.md` (matches source branch name). |
| All six objectives (O1–O6) answered | ✅ Pass | 100% | §3–§8 with OBSERVED evidence. |
| Cleanup / leave-no-trace; source git-clean | ✅ Pass | 100% | Appendix D; `git status --porcelain` empty; temp scripts deleted; DB txn rolled back. |
| Markdown conventions / lint (pre-commit) | ✅ Pass | 100% | Trailing-whitespace 0, EOF newline OK, no tabs, balanced fences (36, even). |
| DMARC/DKIM precision correctness (fixed in validation) | ✅ Pass | 100% | Precision edits in commit `2f9fccf9` corrected DMARC return-value narrative and DKIM key characterization; re-verified citations. |

---

## 6. Risk Assessment

Risks are split into **(A) risks to the deliverable** — all Low and Mitigated — and **(B) risks the investigation surfaces in the SimpleLogin system** — genuinely High and intentionally Open (fixing them is out of AAP scope).

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| T1 — Some causal claims are INFERRED (natural duplicate race) vs OBSERVED (manual seed reproduction) | Technical (deliverable) | Low | Low | OBSERVED-vs-INFERRED ledger (App A); 100% of OBSERVED claims reproduced | Mitigated |
| T2 — Line-number citations could drift if source changes | Technical (deliverable) | Low | Low | Pinned to commit `2cd6ee77`; 6 files md5-verified; citation index (App E) | Mitigated |
| **T3 — Wrong-user routing from unordered `.first()` + no DB uniqueness on `reply_email`** | **Technical (system)** | **High** | **Medium** | Documented recommendation (UNIQUE + deterministic lookup); NOT implemented per scope | **Open** (needs triage) |
| **S1 — Cross-user `reply_email` collision → mis-attribution + mis-delivery + address leak (confidentiality)** | **Security (system)** | **High** | **Low–Medium** | Documented + recommended fix; requires human security triage | **Open** (deliberately not fixed) |
| S2 — Document discloses an OSS security weakness | Security (deliverable) | Low | Low | Internal `blitzy/documentation`; analysis-only; no upstream publication | Managed |
| O1 — Drifted container DKIM key causes spurious reply-test failures for re-runners | Operational (env) | Low | Medium | Root-caused as pre-existing env artifact; committed key valid; `RSPAMD_SIGN_DKIM=1` workaround documented | Mitigated |
| O2 — Disposable test DB drift (+4 users; pg sequences advanced) | Operational (env) | Low | Low | Transparently disclosed; test DB disposable; source & deliverable git-clean | Accepted |
| I1 — Reproducing the live trace requires the full dev stack | Integration | Low | Medium | App B documents exact config/commands; `scripts/run-test.sh` provisions equivalent | Mitigated |
| I2 — `blitzy/documentation/` absent from upstream SimpleLogin | Integration | Low | Low | By design this is the destination repo; not for upstream merge | Accepted |

---

## 7. Visual Project Status

**Project hours — completed vs remaining** (Completed = Dark Blue `#5B39F3`, Remaining = White `#FFFFFF`):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieOuterStrokeWidth':'2px','pieSectionTextColor':'#B23AF2','pieTitleTextSize':'16px','pieLegendTextSize':'13px'}}}%%
pie showData
    title Project Hours Breakdown (Total 26 h)
    "Completed Work" : 24
    "Remaining Work" : 2
```

**Remaining work by category** (from §2.2 — sums to 2.0 h):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#A8FDD9','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieOuterStrokeWidth':'2px','pieSectionTextColor':'#B23AF2','pieTitleTextSize':'15px','pieLegendTextSize':'13px'}}}%%
pie showData
    title Remaining Work by Category (2 h)
    "SME review & sign-off [High]" : 1.5
    "Remediation triage [Medium]" : 0.5
```

> **Integrity:** "Remaining Work" = **2 h** in the pie chart = §1.2 Remaining Hours = sum of §2.2 "Hours" column. ✔ "Completed Work" = **24 h** = §1.2 Completed Hours = sum of §2.1 "Hours" column. ✔

---

## 8. Summary & Recommendations

**Achievements.** The project is **92.3% complete** (24.0 h of 26.0 h). It delivers a single, rigorous, code-grounded investigation that answers all six objectives from a **live execution trace**, not static reading alone. The central result is sharp: the user a reply is attributed to and delivered for is decided by **one** lookup — `Contact.get_by(reply_email=...)` at `email_handler.py:L986` — and everything downstream is a deterministic consequence of which `Contact` it returns. The document explains the user's paradox precisely: "alias recognized" comes from a **separate, earlier** recognition lookup, so logs can show recognition while the later decisive lookup selects a different user. Four compounding structural conditions are evidenced, and the wrong-user mechanism was **reproduced live**, including a confidentiality leak.

**Remaining gaps.** None in the deliverable. The remaining **2.0 h** is human path-to-production: SME review/sign-off (1.5 h) and remediation triage (0.5 h). The remediation **implementation** is deliberately excluded — the task rule forbids code changes — and is sized separately (~18–36 h) for awareness only.

**Critical path to production.** (1) SME validates the findings against the cited source; (2) the team opens a tracked ticket for the recommended UNIQUE constraint + deterministic lookup; (3) a separate engineering effort implements and migrates the fix and audits production data for prior collisions.

**Production-readiness assessment.** The **deliverable is production-ready**: complete, validated across all five gates, lint-clean, and committed with a git-clean source tree (zero source files modified). The **subject system** carries an open, High-severity confidentiality risk that this investigation surfaces and recommends fixing.

| Metric | Result |
|--------|--------|
| AAP-scoped completion | 92.3% (24.0 / 26.0 h) |
| Objectives answered (O1–O6) | 6 / 6 |
| Validation gates passed | 5 / 5 |
| Tests passed / failed | 766 / 0 (Blitzy logs) |
| Source files modified | 0 |
| Files added | 1 (`app_2cd6ee777f8c.md`) |

---

## 9. Development Guide

This task is documentation-only. There are two workflows: **(A) review the deliverable** (works on any machine with git) and **(B) reproduce the live trace** (requires the full SimpleLogin dev stack). All review/verification commands below were tested on the host.

### 9.1 System Prerequisites

- **Review path:** `git`; any Markdown viewer; Python 3 (for git tooling).
- **Reproduction path:** Python **3.10.x** (host carries 3.13 — use the container for 3.10), **PostgreSQL 13**, **Redis 6**, **Poetry**, **Docker 20+**.

### 9.2 Environment Setup

**(A) Review the deliverable:**
```bash
git checkout blitzy-38f8a2c9-c381-44b3-ba1d-3ed9ef6c6431
ls -la blitzy/documentation/app_2cd6ee777f8c.md   # 503 lines, ~53 KB
```

**(B) Provision the reproduction stack** (matches `scripts/run-test.sh`):
```bash
docker run -d --name sl-test-db \
  -e POSTGRES_PASSWORD=test -e POSTGRES_USER=test -e POSTGRES_DB=test \
  -p 15432:5432 postgres:13
sleep 3
CONFIG=tests/test.env poetry run alembic upgrade head   # -> head 32f25cbf12f6
```

### 9.3 Dependency Installation (reproduction)
```bash
poetry install   # python ^3.10, SQLAlchemy 1.3.24, aiosmtpd ^1.2 (pyproject.toml:L61/L116/L87)
```

### 9.4 Run / Simulate an Inbound Reply (reproduction)
```bash
# Project's own reply-handler suite (validated: 23 passed):
CONFIG=tests/test.env poetry run pytest tests/test_email_handler.py -v

# Manual trace mirrors tests/test_email_handler.py:L165-188 —
# construct an Envelope with rcpt_tos=[contact.reply_email], then call
# email_handler.handle(envelope, msg). NOT_SEND_EMAIL=true prints instead of sending.
# Delegate DKIM to rspamd to reach the print branch on this container:
CONFIG=tests/test.env RSPAMD_SIGN_DKIM=1 poetry run python <your_temporary_trace_script>.py
```

### 9.5 Verification Steps (all tested on host)
```bash
git status --porcelain                       # (empty) => clean working tree
git diff --name-status 2cd6ee77..HEAD        # A  blitzy/documentation/app_2cd6ee777f8c.md
# Confirm zero source modifications (expect identical md5 vs base):
for f in email_handler.py app/models.py; do \
  diff <(git show 2cd6ee77:"$f") "$f" >/dev/null && echo "UNCHANGED $f"; done
grep -nE '^#{1,3} ' blitzy/documentation/app_2cd6ee777f8c.md   # section map (Exec Summary + §1–§10)
grep -n '### E. Citation index' blitzy/documentation/app_2cd6ee777f8c.md   # -> line 452
```

### 9.6 How to Read the Document (example usage)
- Follow **O1 → O6** in §3–§8. Start with the Executive Summary for the one-line finding.
- Consult **Appendix A** (Observed-vs-Inferred ledger) to see exactly what was executed vs inferred.
- Consult **Appendix C** for verbatim raw log lines from the live trace.
- See **§9 of the document** for the recommendation (analysis-only).

### 9.7 Troubleshooting
- **`dkim.KeyFormatError` on reply tests:** caused by the container's drifted PKCS#8 `/app/local_data/dkim.key` (pre-existing env artifact). Use the committed PKCS#1 repo key, or set `RSPAMD_SIGN_DKIM=1` to delegate DKIM and reach the `NOT_SEND_EMAIL` print branch. Independent of the routing finding.
- **Host Python is 3.13, not 3.10:** run the stack inside the `python:3.10` container (`Dockerfile`).
- **`psql`/`redis-cli`/`poetry` not on host PATH:** use `scripts/run-test.sh` (provisions `postgres:13` in Docker) and run Poetry inside the container.
- **Test DB drift / pg sequence gaps:** expected — the disposable `sl-test-db` is not the deliverable; recreate it via `scripts/run-test.sh` for a clean baseline.

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---------|---------|
| Locate deliverable | `ls -la blitzy/documentation/app_2cd6ee777f8c.md` |
| Verify clean tree | `git status --porcelain` |
| Diff vs base | `git diff --name-status 2cd6ee77..HEAD` |
| Authorship | `git log --author="agent@blitzy.com" 2cd6ee77..HEAD --oneline` |
| Provision DB | `docker run -d --name sl-test-db -e POSTGRES_PASSWORD=test -e POSTGRES_USER=test -e POSTGRES_DB=test -p 15432:5432 postgres:13` |
| Migrate | `CONFIG=tests/test.env poetry run alembic upgrade head` |
| Reply tests | `CONFIG=tests/test.env poetry run pytest tests/test_email_handler.py -v` |
| Section map | `grep -nE '^#{1,3} ' blitzy/documentation/app_2cd6ee777f8c.md` |

### B. Port Reference

| Service | Port | Source |
|---------|------|--------|
| PostgreSQL (test DB) | 15432 → 5432 | `tests/test.env:L17`, `scripts/run-test.sh` |
| Redis (mem store) | 6379 (default) | `tests/test.env:L78` (`MEM_STORE_URI=redis://localhost`) |
| Web (gunicorn, context only) | 7777 | project default (not exercised by this task) |

### C. Key File Locations

| File | Role |
|------|------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The sole deliverable** (503 lines) |
| `email_handler.py` | Inbound SMTP router `handle()` (`L1945`), `handle_reply()` (`L966`), decisive lookup (`L986`) |
| `app/models.py` | `ModelMixin.get_by` (`L83-85`), `Contact.reply_email` (`L1899`), `uq_contact` (`L1874-1876`), `EmailLog` |
| `app/email_utils.py` | `is_reverse_alias` (`L1156/L1158`), `generate_reply_email` (`L1103`) |
| `app/email_validation.py` | `normalize_reply_email` (`L25-38`) |
| `app/contact_utils.py` | `create_contact` (`L42`), `IntegrityError` recovery (`L113-118`) |
| `tests/test_email_handler.py` | Reply-simulation harness (`L34-52`, `L165-188`) |
| `tests/test.env`, `scripts/run-test.sh` | Print-mode config; DB provision + migrate + test |

### D. Technology Versions

| Component | Version | Source |
|-----------|---------|--------|
| Python | 3.10.18 (declared `^3.10`) | `pyproject.toml:L61`, `Dockerfile:L8` |
| SQLAlchemy | 1.3.24 | `pyproject.toml:L116` |
| aiosmtpd | ^1.2 | `pyproject.toml:L87` |
| PostgreSQL | 13 | `scripts/run-test.sh`, CI `main.yml:L47` |
| Redis | 6 | CI `main.yml:L92-94` |
| pytest | ^7.0.0 | `pyproject.toml:L122` |
| Alembic migration head | `32f25cbf12f6` | runtime (`alembic current`) |

### E. Environment Variable Reference

| Variable | Value | Purpose |
|----------|-------|---------|
| `CONFIG` | `tests/test.env` | Selects test configuration |
| `NOT_SEND_EMAIL` | `true` | Print instead of sending mail (safe simulation) — `tests/test.env:L7` |
| `EMAIL_DOMAIN` | `sl.local` | Alias/reverse-alias domain — `tests/test.env:L8` |
| `DB_URI` | `postgresql://test:test@localhost:15432/test` | Test DB connection — `tests/test.env:L17` |
| `MEM_STORE_URI` | `redis://localhost` | Redis connection — `tests/test.env:L78` |
| `RSPAMD_SIGN_DKIM` | `1` (trace only) | Delegate DKIM signing to rspamd to reach the print branch (`app/config.py:L482`) |

### F. Developer Tools Guide

- **Live-trace harness:** mirror `tests/test_email_handler.py:L165-188` — build an `email.message.Message`, an `aiosmtpd` `Envelope` with `rcpt_tos=[contact.reply_email]`, then call `email_handler.handle(envelope, msg)`. Wrap seeded rows in a `connection.begin()` transaction and `rollback()` (per `tests/conftest.py:L61,L75-77`) to leave the DB clean.
- **No browser/UI tooling** was required (backend investigation; no UI in scope).
- **Citation auditing:** extract anchors with `grep -oE '[A-Za-z_/]+\.(py|toml|env|yml|sh):L?[0-9]+'` and confirm each against source.

### G. Glossary

| Term | Definition |
|------|------------|
| **Reverse-alias / `reply_email`** | The per-contact address (`Contact.reply_email`) a user replies *to*; SimpleLogin relays from the alias to the contact's real address. |
| **Recognition lookup** | `is_reverse_alias()` check that emits the "Reply phase" log — fires *before* the decisive lookup. |
| **Decisive lookup** | `Contact.get_by(reply_email=...)` at `email_handler.py:L986` — selects the contact/alias/user; the single routing pivot. |
| **Shadowed user** | When two contacts share a `reply_email`, the user whose contact `.first()` does **not** return — their replies resolve to the other user. |
| **OBSERVED vs INFERRED** | OBSERVED = reproduced in the live trace; INFERRED = concluded from code reading. Tracked in Appendix A of the deliverable. |
| **TOCTOU** | Time-of-check-to-time-of-use race; here, the unbacked application-layer uniqueness check on `reply_email`. |