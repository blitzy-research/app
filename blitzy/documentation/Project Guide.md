# Blitzy Project Guide — SimpleLogin Reply-Resolution Root-Cause Investigation

> Branch: `blitzy-c2a41c33-7e7e-4397-9fc6-7fc76ebf8165` · Baseline `2cd6ee77` → HEAD `8aff3020`
> Deliverable: `blitzy/documentation/app_2cd6ee777f8c.md` (845 lines) · SHA256 `3f1ef003019e03cedf8c6c98b0225a473294df0ba5e07e64dc32195290d106ac`
> Task class: **Read-only investigative documentation / Q&A** (governing rules: *SWE-AtlasQnA-Repo*)

---

## 1. Executive Summary

### 1.1 Project Overview

This project is a **read-only, evidence-based root-cause investigation** of how SimpleLogin (a Python/Flask email-aliasing service) resolves an inbound email **reply** to a Contact and forwarding destination — and whether that resolution can misroute a reply to the **wrong user** through race conditions, uniqueness assumptions, or timing. The sole deliverable is one Markdown document (`blitzy/documentation/app_2cd6ee777f8c.md`, 845 lines) that answers nine explicit requirements (R1–R9) written from **observed runtime behavior**, driving the real `aiosmtpd` entry point `email_handler.handle()` → `handle_reply()`. Target audience is SimpleLogin maintainers and security reviewers. Business impact: it surfaces a **latent misrouting vulnerability** and its mitigating controls. The repository is left **byte-for-byte unchanged** except the single documentation file.

### 1.2 Completion Status

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieOuterStrokeWidth':'2px','pieSectionTextColor':'#111111','pieLegendTextColor':'#111111','pieTitleTextSize':'16px'}}}%%
pie showData title AAP-Scoped Completion — 92.0% (46h of 50h)
    "Completed Work (AI)" : 46
    "Remaining Work" : 4
```

| Metric | Value |
|---|---|
| **Total Hours** | **50** |
| **Completed Hours (AI + Manual)** | **46** (46 AI + 0 Manual) |
| **Remaining Hours** | **4** |
| **Percent Complete** | **92.0%** |

Completion is computed on **AAP-scoped work only** (PA1): `Completed / (Completed + Remaining) = 46 / (46 + 4) = 92.0%`. Colors: **Completed = Dark Blue `#5B39F3`**, **Remaining = White `#FFFFFF`**.

### 1.3 Key Accomplishments

- ✅ **All nine requirements (R1–R9) answered by name**, each backed by the exact command, complete unedited output, and `file:line` citations.
- ✅ **Real entry point exercised** — every experiment drives `email_handler.handle()` → `handle_reply()`; no synthetic stand-ins (any direct-call illustration is explicitly labeled non-canonical).
- ✅ **Eight runtime experiments (E1–E8)** covering the happy path, multi-event stability, the duplicate-`reply_email` selection flip, and the full reply-handler status-code set (`E200`/`E214`/`E501`/`E502`/`E503`/`E504`).
- ✅ **Root cause established with runtime proof**: the reply lookup `Contact.get_by(reply_email=…)` is an **unordered `LIMIT 1`** over a **non-unique** `reply_email` index; under a duplicate it selects a database-arbitrary Contact and can flip over time (Contact 327/user 904 → Contact 328/user 905).
- ✅ **Mitigating control characterized**: the anti-spoofing gate `get_mailbox_from_mail_from()` usually converts a mis-resolution into an `E214` bounce, except under `disable_email_spoofing_check=True` or a shared mailbox (then a **silent wrong-user delivery** — `EmailLog user_id=908` while replier is user 907).
- ✅ **Canonical stack provisioned & validated** — Python 3.10.18 / SQLAlchemy 1.3.24 / PostgreSQL 15, schema at Alembic head `32f25cbf12f6`.
- ✅ **Read-only guarantee intact** — `git diff 2cd6ee77 HEAD` shows only the one added file; zero source/test/migration/config changes; 14 temporary observation scripts removed.
- ✅ **CI-clean & validated** — reply-path suite `tests/test_email_handler.py` **23 passed / 0 failed**; pre-commit hooks pass; 100% of citations verified.

### 1.4 Critical Unresolved Issues

There are **no issues blocking the validity of the deliverable** — it is validated, CI-clean, and byte-for-byte read-only. The items below are **pending human actions**, not defects in the deliverable.

| Issue | Impact | Owner | ETA |
|---|---|---|---|
| Human peer review & sign-off not yet performed | Deliverable validated but not yet formally **accepted** (non-blocking to its correctness) | Reviewer / Maintainer | 2.5h |
| Latent `reply_email` misrouting finding awaits triage | If unactioned, the latent misrouting risk persists in production SimpleLogin (remediation is **out of scope** for this investigation) | SimpleLogin maintainers | 1.0h (triage only) |

### 1.5 Access Issues

**No access issues identified.** The repository was fully accessible, the canonical prebuilt Docker image was available to the autonomous validation environment (all experiments executed successfully), and no third-party credentials or API access were required (`NOT_SEND_EMAIL=true`; outbound mail captured in-memory).

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|---|---|---|---|---|
| Repository working tree | Read/Write (git) | None — accessible; read-only guarantee honored | ✅ No issue | Blitzy Agent |
| Canonical Docker image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` | Container runtime | None — image present locally (2.14 GB); required only to *re-run* experiments (not an access blocker) | ✅ No issue (env note) | Reviewer |
| Third-party email/SMTP services | External API | None required — sending disabled, mail stored in-memory | ✅ N/A | — |

### 1.6 Recommended Next Steps

1. **[High]** Perform the technical peer review & sign-off of `blitzy/documentation/app_2cd6ee777f8c.md` — read the document, spot-check citations, confirm the R1–R9 answers and methodology (**2.5h**).
2. **[Medium]** Triage the latent `reply_email` uniqueness/misrouting finding and decide whether to open a follow-up remediation ticket (**1.0h**).
3. **[Medium]** Approve the pull request and merge the single-file deliverable to the target branch (**0.5h**).
4. **[Low]** *(Out of scope, future project)* If triage warrants, scope a remediation: add DB-level uniqueness on `reply_email`, replace the unordered `first()` with an ordered/constrained query, and/or lock contact creation.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

All components below were delivered **autonomously (AI)** and map to AAP-scoped investigation work.

| Component | Hours | Description |
|---|---:|---|
| Environment provisioning | 4 | Canonical Python 3.10 venv, 177 pinned dependencies, PostgreSQL 15, `alembic upgrade head` to `32f25cbf12f6`, config from `example.env`/`test.env` |
| Reply-path code comprehension | 6 | Traced ~8,130 lines across `email_handler.py`, `app/models.py`, `app/email_utils.py`, `app/email_validation.py`, `app/contact_utils.py`, `app/email/status.py`, `app/utils.py` + 2 migrations |
| Non-invasive instrumentation harness | 4 | Runtime spy monkeypatch around `Contact.get_by` (records/prints, delegates to original), repo test-util seeding, `mail_sender` capture — **no source edits** |
| Runtime experiments E1–E8 | 12 | Happy path, stability (12×2 runs), duplicate + heap-reorder flip, `E501`/`E502`/normalization asymmetry, `E214` anti-spoofing (3 configs), `IntegrityError` rollback/refetch race, `E503`, `E504` |
| Output analysis & value capture | 4 | Extracted exact runtime values, emitted SQL (no `ORDER BY`), distributions, `EmailLog` ids, before/during/after state |
| Document authoring (845 lines) | 10 | TL;DR, environment/provenance, methodology, R1–R9 sections with embedded evidence, cause→effect narrative, coverage pass |
| Validation, citation verification, review fixes, CI | 4 | 100% citation verification, review-findings remediation (commit `18928d55` +225/−13), CI trailing-whitespace compliance (commit `8aff3020`) |
| Cleanup & read-only verification | 2 | Removed 14 temporary observation scripts; `git status`/diff confirmation of byte-for-byte read-only guarantee |
| **Total Completed** | **46** | — |

### 2.2 Remaining Work Detail

All remaining work is **human path-to-production** for a documentation deliverable (no autonomous work remains within AAP scope).

| Category | Hours | Priority |
|---|---:|---|
| Human technical peer review & sign-off of the investigation | 2.5 | High |
| Follow-up remediation-ticket triage decision (latent `reply_email` finding; remediation itself out of scope) | 1.0 | Medium |
| PR approval & merge of the single-file deliverable to target branch | 0.5 | Medium |
| **Total Remaining** | **4.0** | — |

### 2.3 Hours Reconciliation

- Section 2.1 (Completed) = **46h** = Completed Hours in §1.2.
- Section 2.2 (Remaining) = **4h** = Remaining Hours in §1.2 = §7 "Remaining Work".
- §2.1 + §2.2 = 46 + 4 = **50h** = Total Project Hours in §1.2.
- Completion = 46 / 50 = **92.0%**.

---

## 3. Test Results

All tests below originate from **Blitzy's autonomous validation logs** for this project (executed on the canonical stack inside the prebuilt container). Because the deliverable is read-only documentation, the "tests" are (a) the existing reply-path suite the validator ran to prove the code path is healthy, and (b) the eight investigative runtime reproductions that produced the document's evidence.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---|---|---:|---:|---:|---:|---|
| Reply-path unit/integration suite | pytest | 23 | 23 | 0 | n/a (targeted suite) | `tests/test_email_handler.py`, canonical flags `-o addopts="" -p no:cacheprovider`, exit 0 |
| Investigative runtime reproductions (E1–E8) | Custom harness via real `email_handler.handle()` | 8 | 8 | 0 | n/a | All matched documented values; only DB row ids differ run-to-run (disclosed) |
| Pre-commit / CI lint (deliverable) | pre-commit | 1 | 1 | 0 | n/a | `trailing-whitespace` **Passed**; check-yaml/djLint/ruff **Skipped** (non-applicable to `.md`) |
| **Total** | — | **32** | **32** | **0** | — | 100% pass rate across all autonomous checks |

**Notes on coverage:** the suite is a *targeted* reply-path suite, not a full-project coverage run; no line-coverage percentage was produced by the autonomous logs, so none is fabricated here. The deliverable changes **zero** source lines, so the wider test suite is unaffected by construction.

---

## 4. Runtime Validation & UI Verification

Runtime health was validated by driving the **real** `aiosmtpd`-style entry point end-to-end. Status legend: ✅ Operational · ⚠ Partial / reproduced-vulnerability · ❌ Failing.

**Reply-path runtime (backend):**
- ✅ **E1 — Happy path**: `handle()` → `handle_reply()` returns `E200`; `EmailLog(is_reply=True)`; relayed to `contact.website_email`.
- ✅ **E2 — Multi-event stability**: SAME input × 12, across 2 process runs → distributions `{325:12}` then `{326:12}`; deterministic when `reply_email` is unique.
- ⚠ **E3 — Duplicate `reply_email`**: unordered `get_by().first()` selects a database-arbitrary Contact; SAME input flipped **Contact 327/user 904 → Contact 328/user 905** after an unrelated `UPDATE` reordered the heap (the finding, reproduced).
- ✅ **E4 — Non-resolution**: `E502` (no contact), normalization asymmetry (`#`→`_` renders a stored Contact unreachable), `E501` (wrong domain) — all reproduced.
- ⚠ **E5 — Anti-spoofing outcome**: default → `E214` **bounce** (protective, ✅); `disable_email_spoofing_check=True` → **silent wrong-user delivery** (`EmailLog user_id=908`, replier user 907, ⚠); shared mailbox → delivered to arbitrarily-selected contact (⚠).
- ✅ **E6 — Contact-creation race**: `uq_contact` + `IntegrityError` rollback/refetch converge to one Contact; loser's `reply_email` discarded.
- ✅ **E7 — `E503`**: resolved alias on an unverified/removed domain → rejected, not forwarded (`outbound = 0`).
- ✅ **E8 — `E504`**: disabled account → rejected (`is_active()` vs `can_send_or_receive()` distinction), not forwarded.

**API integration:** ✅ Outbound relay target captured via `mail_sender.store_emails_instead_of_sending()`; `SendRequest.envelope_to == contact.website_email` confirmed. No external network calls (sending disabled).

**UI verification:** ⚠ **Not applicable** — this is a backend email-routing investigation with **no UI component in scope**. The repository's `static/` front-end (Node 10) was intentionally not provisioned and is irrelevant to the reply path.

---

## 5. Compliance & Quality Review

AAP deliverables and governing-rule requirements cross-mapped to Blitzy quality/compliance benchmarks. Progress legend: ✅ Pass · ⚠ Partial · ❌ Fail.

| Benchmark / AAP Requirement | Status | Evidence / Fix Applied |
|---|:--:|---|
| R1 — Reply-address derivation answered | ✅ | §R1 + E1 (`reply_email=rcpt_to` L972, normalize L984, routing L2195) |
| R2 — Contact identification answered | ✅ | §R2 + E1 spy + E3 emitted SQL (`filter_by().first()`, no `ORDER BY`) |
| R3 — Runtime values observed | ✅ | §R3 result table (Contact/alias/user/mailbox/website_email/status/EmailLog) |
| R4 — Behavior across multiple reply events | ✅ | §R4 + E2 (SAME input, N=12, ≥2 runs) |
| R5 — Same reply email → different Contacts | ✅ | §R5 + E3 setup/observe/flip (327→328) |
| R6 — Temporary non-resolution | ✅ | §R6 + E4/E7/E8 (E501/E502/E503/E504 + normalization) |
| R7 — Correct-now-wrong-later | ✅ | §R7 + E5 (E214 bounce vs silent wrong-user delivery) |
| R8 — Root causes (race/uniqueness/timing) | ✅ | §R8 + E3 + E6 (4 interlocking facts) |
| R9 — Cause → effect narrative | ✅ | §R9 synthesis of E1–E6 |
| Investigate-by-running-first (runtime-observed) | ✅ | Every claim has command + complete unedited output |
| Real entry point (no synthetic stand-ins) | ✅ | All E1–E8 via `email_handler.handle()`; direct-calls labeled |
| Reproduce inconsistency with SAME input | ✅ | E2/E3 report distributions across ≥2 runs |
| Exact & grounded (`file:line` for every claim) | ✅ | 100% citations verified (validator + independent spot-check) |
| Complete, unedited output included | ✅ | 24 verbatim output blocks; only fixed startup banner filtered (disclosed) |
| Read-only repository (no source modified) | ✅ | `git diff 2cd6ee77 HEAD -- ':!blitzy'` empty; 14 temp scripts removed |
| Deliverable naming/location | ✅ | `blitzy/documentation/app_2cd6ee777f8c.md` (source branch name) |
| CI / pre-commit compliance | ✅ | `trailing-whitespace` pass (fixed in commit `8aff3020`) |
| Zero placeholders / TODO / FIXME | ✅ | Grep confirms none; 48 balanced code fences |

**Fixes applied during autonomous validation:** (1) review-findings addressed in commit `18928d55` (+225/−13, e.g., added E503/E504 guard-branch coverage and normalization-pipeline clarification); (2) trailing-whitespace stripped from 2 verbatim table-header lines for CI in commit `8aff3020` (evidence preserved byte-for-byte). **Outstanding compliance items:** none.

---

## 6. Risk Assessment

Two layers are distinguished: **[DELIVERY]** = risk to this documentation deliverable/project; **[FINDING]** = a subject-matter vulnerability the investigation *surfaced* in SimpleLogin (remediation is out of scope for this task).

| Risk | Category | Severity | Probability | Mitigation | Status |
|---|---|:--:|:--:|---|---|
| [FINDING] Silent wrong-user email delivery when a mis-resolved alias has `disable_email_spoofing_check=True` or shares a mailbox | Security | High (if triggered) | Low | Anti-spoofing `E214` gate catches the common case; documented with runtime proof; recommend remediation ticket | Open (finding) |
| [FINDING] Unordered `get_by().first()` (no `ORDER BY`) → nondeterministic, time-varying Contact selection under duplicate `reply_email` | Technical | Medium | Low | Documented (E3 flip); recommend ordered/constrained query | Open (finding) |
| [FINDING] `reply_email` has no DB-level uniqueness (soft TOCTOU generation only) | Technical | Medium | Low | Documented (E3/E6 contrast with `uq_contact`); recommend unique constraint | Open (finding) |
| [FINDING] `normalize_reply_email` asymmetry can make a stored Contact permanently unreachable (`E502`) | Technical | Low | Low | Documented (E4b); recommend generation/lookup symmetry review | Open (finding) |
| [DELIVERY] Findings not actioned after delivery | Operational | Medium | Medium | §1.6/§8 recommend triage; owner = maintainers | Open (human decision) |
| [DELIVERY] Documentation staleness — pinned to baseline `2cd6ee77`; `file:line` drift if reply path changes | Operational | Low | Medium | Doc records exact HEAD/baseline + citations; re-run after relevant changes | Accepted |
| [DELIVERY] Canonical-stack reproduction dependency (Docker image; host Python 3.13/SQLAlchemy 2.0 incompatible) | Integration | Low | Medium | Complete unedited output embedded; exact env provenance + commands + image id documented | Mitigated |
| [DELIVERY] Citation/framing dispute at review | Technical | Low | Low | 100% citations verified; complete unedited output embedded | Mitigated |
| [DELIVERY] Secrets/credential exposure in deliverable | Security | Low | Low | No secrets; `NOT_SEND_EMAIL=true`; mail captured in-memory | Mitigated / N/A |
| [DELIVERY] External-integration failure | Integration | Low | Low | No external API/webhook/network integration involved | N/A |

---

## 7. Visual Project Status

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieOuterStrokeWidth':'2px','pieSectionTextColor':'#111111','pieLegendTextColor':'#111111','pieTitleTextSize':'16px'}}}%%
pie showData title Project Hours Breakdown (Total 50h)
    "Completed Work" : 46
    "Remaining Work" : 4
```

**Remaining hours by category (Section 2.2):**

```mermaid
%%{init: {'theme':'base','themeVariables':{'xyChart':{'plotColorPalette':'#5B39F3'}}}}%%
xychart-beta
    title "Remaining Work by Category (hours)"
    x-axis ["Peer review (H)", "Finding triage (M)", "PR merge (M)"]
    y-axis "Hours" 0 --> 3
    bar [2.5, 1.0, 0.5]
```

- **Completed Work = 46h** (Dark Blue `#5B39F3`) · **Remaining Work = 4h** (White `#FFFFFF`).
- The "Remaining Work" value (**4h**) equals §1.2 Remaining Hours and the sum of the §2.2 "Hours" column — integrity holds.

---

## 8. Summary & Recommendations

**Achievements.** The investigation is **92.0% complete** on an AAP-scoped basis (46 of 50 hours). Every one of the nine requirements (R1–R9) is answered by name, from **observed runtime behavior** driven through the real `email_handler.handle()` → `handle_reply()` entry point, with each claim backed by the exact command, complete unedited output, and `file:line` citations. The deliverable establishes — with reproduced runtime evidence — that reply routing rests on a **latent uniqueness assumption**: `Contact.get_by(reply_email=…)` is an unordered `LIMIT 1` over a **non-unique** `reply_email` index, so a duplicate `reply_email` lets the SAME input resolve to **different Contacts/users over time** (Contact 327/user 904 → 328/user 905), and — when the anti-spoofing gate is disabled or a mailbox is shared — produce a **silent wrong-user delivery**.

**Remaining gaps (4h, all human path-to-production).** (1) technical peer review & sign-off (2.5h); (2) triage of the latent `reply_email` finding into a follow-up remediation ticket (1.0h; remediation itself out of scope); (3) PR approval & merge (0.5h). There are **no autonomous gaps** within the AAP scope and **no compilation/test failures** to fix — the task is read-only documentation, validated and CI-clean.

**Critical path to production.** Peer review → finding triage → merge. The dominant item is human review of the 845-line analysis; the canonical Docker image is available if the reviewer chooses to re-run any experiment.

**Success metrics.** 9/9 requirements answered; 32/32 autonomous checks passing (23 reply-path tests + 8 reproductions + 1 lint); 100% citation accuracy; read-only guarantee byte-for-byte intact (only 1 file added).

**Production-readiness assessment.** The deliverable is **production-ready pending human sign-off**. It is complete, internally consistent, evidence-backed, and leaves the repository unchanged except the single documentation file. Recommended disposition: **accept and merge**, then triage the surfaced finding separately.

| Metric | Value |
|---|---|
| AAP-scoped completion | 92.0% (46h / 50h) |
| Requirements answered | 9 / 9 (R1–R9) |
| Autonomous checks passing | 32 / 32 |
| Source files modified | 0 (read-only guarantee intact) |
| Remaining (human) | 4h |

---

## 9. Development Guide

Because the deliverable is **read-only documentation**, this guide covers (A) viewing the deliverable and verifying the read-only guarantee, (B) spot-checking citations, and (C) reproducing the runtime observations in the canonical environment. All host-runnable commands below were **tested** during assessment.

### 9.1 System Prerequisites

- **Git** (repository access).
- For **verification/review only**: any shell (no special runtime needed).
- For **reproducing experiments** (optional): **Docker** (tested with 28.5.2) and the prebuilt canonical image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (Python **3.10.18**, SQLAlchemy **1.3.24**, PostgreSQL **15**). The host interpreter (**Python 3.13 / SQLAlchemy 2.0**) is **incompatible** with the pinned API — do **not** run the reply path on the host.

### 9.2 View the Deliverable & Verify the Read-Only Guarantee

```bash
# From the repository root:
cd /tmp/blitzy/app/blitzy-c2a41c33-7e7e-4397-9fc6-7fc76ebf8165_9eedb3

# 1) Locate & size the single deliverable
ls -l blitzy/documentation/app_2cd6ee777f8c.md          # ~86 KB, 845 lines

# 2) Read-only proof — only the doc was added since baseline
git diff --name-status 2cd6ee77 HEAD                    # -> A blitzy/documentation/app_2cd6ee777f8c.md

# 3) Confirm ZERO source/test/migration/config changes
git diff --name-only 2cd6ee77 HEAD -- ':!blitzy'        # -> (empty)

# 4) Working tree is clean
git status --porcelain                                  # -> (empty)
```

Expected: step 2 prints exactly one added path; steps 3–4 print nothing.

### 9.3 Spot-Check Key Citations (read-only)

```bash
# Reply-address derivation, normalization, and the single Contact lookup
sed -n '972p;984p;986p' email_handler.py
#   reply_email = rcpt_to
#   reply_email = normalize_reply_email(reply_email)
#   contact = Contact.get_by(reply_email=reply_email)

# get_by is an unordered .first()
sed -n '83,84p' app/models.py
#   def get_by(cls, **kw):
#       return Session.query(cls).filter_by(**kw).first()

# reply_email index is NON-unique
sed -n '1899p' app/models.py
#   reply_email = sa.Column(sa.String(512), nullable=False, index=True)

# DB migration proves unique=False
grep -n "ix_contact_reply_email" migrations/versions/2021_071310_78403c7b8089_.py
#   op.create_index(op.f('ix_contact_reply_email'), 'contact', ['reply_email'], unique=False)
```

### 9.4 Reproduce the Runtime Observations (optional, canonical container)

The document embeds every command and its complete unedited output; re-running is optional. The canonical invocation pattern (as recorded in the deliverable) is:

```bash
# Start the prebuilt canonical container (image present locally, ~2.14 GB)
docker run -d --name sl-app ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0
# (or `docker exec` into an already-running instance)

# Confirm the canonical stack + DB
docker exec sl-app bash -c 'source /root/sl_env.sh && cd /app && \
  /app/venv/bin/python --version && \
  /app/venv/bin/python -c "import sqlalchemy; print(sqlalchemy.__version__)" && \
  pg_isready -h localhost -p 5432 && \
  /app/venv/bin/alembic current'
# Expect: Python 3.10.18 · 1.3.24 · accepting connections · 32f25cbf12f6 (head)

# Drive a reply through the REAL entry point using a temporary script placed in
# the container's /tmp (NOT the repo tree), then remove it afterward:
docker exec sl-app bash -c 'source /root/sl_env.sh && cd /app && \
  /app/venv/bin/python /tmp/sl_investigation/e1_happy_path.py'
```

### 9.5 Troubleshooting

- **`ImportError` / SQLAlchemy API errors on host** → you are on Python 3.13 / SQLAlchemy 2.0; switch to the canonical container (SQLAlchemy 1.3.24). Do not `pip install` a different SQLAlchemy into the system interpreter.
- **`alembic upgrade head` prints no steps** → expected; the DB is already at head `32f25cbf12f6` (idempotent).
- **Startup banner noise** → the config loader prints a fixed banner; the deliverable filters only that banner via `grep -v -E "load config file|>>> URL:|…"` and never elides experiment output.
- **Different DB row ids on re-run** → expected and disclosed; ids accumulate across runs, but each output block is internally consistent.

---

## 10. Appendices

### Appendix A — Command Reference

| Purpose | Command |
|---|---|
| Locate deliverable | `ls -l blitzy/documentation/app_2cd6ee777f8c.md` |
| Read-only proof (added file) | `git diff --name-status 2cd6ee77 HEAD` |
| Prove no source changes | `git diff --name-only 2cd6ee77 HEAD -- ':!blitzy'` |
| Clean-tree check | `git status --porcelain` |
| Commit history | `git log --oneline 2cd6ee77..HEAD` |
| Citation spot-check | `sed -n '972p;984p;986p' email_handler.py` |
| Non-unique index proof | `sed -n '1899p' app/models.py` |
| Migration proof | `grep -n ix_contact_reply_email migrations/versions/2021_071310_78403c7b8089_.py` |
| Canonical version check | `docker exec sl-app bash -c 'source /root/sl_env.sh && /app/venv/bin/python --version'` |
| Alembic state | `docker exec sl-app bash -c 'source /root/sl_env.sh && cd /app && /app/venv/bin/alembic current'` |

### Appendix B — Port Reference

| Service | Port | Notes |
|---|---|---|
| PostgreSQL (canonical container) | 5432 | `DB_URI=postgresql://test:test@localhost:5432/test` (canonical run env) |
| Redis (canonical container) | 6379 | Started by the image; not exercised by the reply-lookup experiments |
| SMTP inbound (`aiosmtpd`) | n/a (in-process) | Experiments call `email_handler.handle()` directly; no socket bound |

### Appendix C — Key File Locations

| Path | Role |
|---|---|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The deliverable** (845 lines) |
| `email_handler.py` | Inbound SMTP hub; `handle()` L1945, `handle_reply()` L966, lookup L986 |
| `app/models.py` | `ModelMixin.get_by` L83–84; Contact schema L1863; `reply_email` index L1899; `uq_contact` L1874–1876 |
| `app/email_utils.py` | `generate_reply_email` L1103; `is_reverse_alias` L1156 |
| `app/email_validation.py` | `normalize_reply_email` L25–38 |
| `app/contact_utils.py` | `create_contact` L42; `IntegrityError` rollback/refetch L113–119 |
| `app/email/status.py` | Status codes `E200`/`E214`/`E501`–`E504` |
| `migrations/versions/2021_071310_78403c7b8089_.py` | `ix_contact_reply_email … unique=False` (L22) |
| `migrations/versions/2020_031711_0809266d08ca_.py` | `uq_contact(alias_id, website_email)` (L45) |
| `tests/test_email_handler.py` | Reply-path test harness (23 tests) |

### Appendix D — Technology Versions

| Component | Version | Source |
|---|---|---|
| Python (canonical) | 3.10.18 | `pyproject.toml`:L61 `^3.10`; `Dockerfile`:L8 `FROM python:3.10` |
| SQLAlchemy | 1.3.24 | `pyproject.toml`:L116 |
| Flask | 1.1.2 | `poetry.lock` |
| Flask-SQLAlchemy | 2.5.1 | `poetry.lock` |
| aiosmtpd | 1.4.2 | `poetry.lock` |
| alembic | 1.4.3 | `poetry.lock` (schema head `32f25cbf12f6`) |
| psycopg2-binary | 2.9.3 | `poetry.lock` |
| arrow | 0.16.0 | `poetry.lock` |
| PostgreSQL | 15 | canonical container |
| Host stack (incompatible) | Python 3.13.7 / SQLAlchemy 2.0.51 | assessment host — do not use |

### Appendix E — Environment Variable Reference

| Variable | Canonical Value | Purpose |
|---|---|---|
| `CONFIG` | `/app/tests/test.env` | App configuration file |
| `DB_URI` | `postgresql://test:test@localhost:5432/test` | Database connection (canonical run) |
| `EMAIL_DOMAIN` | `sl.local` | Reverse-alias domain used in experiments |
| `NOT_SEND_EMAIL` | `true` | Disables real sending; mail captured in-memory |
| `DISABLE_RATE_LIMIT` | `1` | Removes rate limiting for deterministic runs |

*(Repository default template `example.env` uses `DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin`, L75.)*

### Appendix F — Developer Tools Guide

- **git** — read-only verification (`diff`, `status`, `log`) as in §9.2.
- **sed / grep** — citation spot-checks against source (§9.3).
- **docker** (28.5.2) — run the canonical image to reproduce experiments (§9.4).
- **pytest** — reply-path suite: `python -m pytest tests/test_email_handler.py -o addopts="" -p no:cacheprovider` (run inside the canonical container).
- **pre-commit** — CI lint parity for the deliverable: `pre-commit run --files blitzy/documentation/app_2cd6ee777f8c.md`.

### Appendix G — Glossary

| Term | Meaning |
|---|---|
| **Reverse-alias / `reply_email`** | The per-`(alias, contact)` address SimpleLogin generates; replying to it routes through `handle_reply`. |
| **`get_by().first()`** | `Session.query(cls).filter_by(**kw).first()` — an **unordered `LIMIT 1`**; returns a database-arbitrary row under duplicates. |
| **`uq_contact`** | The only unique constraint on `contact`: `(alias_id, website_email)`. `reply_email` is **not** covered. |
| **TOCTOU** | Time-of-check-to-time-of-use; the soft, non-transactional uniqueness check in `generate_reply_email`/`available_sl_email`. |
| **Anti-spoofing gate** | `get_mailbox_from_mail_from()` — rejects (`E214`) a reply whose `mail_from` isn't authorized for the resolved alias, unless `disable_email_spoofing_check`. |
| **E200/E214/E501–E504** | Reply-handler SMTP status codes: accepted / unauthorized-reverse-alias / bad-domain / no-contact / unknown-alias-domain / account-disabled. |
| **Latent defect** | A weakness that does not fire in normal operation but manifests once a precondition (here, a duplicate `reply_email`) is introduced. |