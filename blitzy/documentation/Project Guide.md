# Blitzy Project Guide
## SimpleLogin Inbound Bounce Handling — Enumeration-Oracle Security Investigation

---

# 1. Executive Summary

## 1.1 Project Overview

This project is an **evidence-based, read-only security investigation** of SimpleLogin's inbound bounce-email handling. The objective was to determine — by actually running the canonical inbound SMTP path and capturing real responses — whether the system exposes an information-leakage / recipient-enumeration **oracle**, and to deliver the findings as a single comprehensive answer document. The audience is SimpleLogin maintainers and security engineers. The technical scope spans the inbound bounce-routing pipeline (`email_handler.py`, `app/email_utils.py`, `app/email/status.py`, `app/config.py`, `app/errors.py`, `app/models.py`, `app/handler/spamd_result.py`). The sole deliverable is `blitzy/documentation/app_2cd6ee777f8c.md`; the source repository is left byte-for-byte unchanged. The investigation confirms a live oracle on the older unsigned VERP address format.

## 1.2 Completion Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieTitleTextSize':'18px','pieSectionTextSize':'14px','pieSectionTextColor':'#B23AF2','pieLegendTextColor':'#111111'}}}%%
pie showData title Completion Status — 89.8% Complete
    "Completed Work (AI + Manual)" : 53
    "Remaining Work" : 6
```

| Metric | Value |
|--------|-------|
| **Total Hours** | **59** |
| **Completed Hours (AI + Manual)** | **53** |
| **Remaining Hours** | **6** |
| **Percent Complete** | **89.8%** |

> Completion is computed with the PA1 AAP-scoped, hours-based methodology: `Completed / (Completed + Remaining) = 53 / 59 = 89.8%`. Every hour traces to a specific Agent Action Plan (AAP) requirement or a path-to-acceptance activity. The remaining 6 hours are human path-to-acceptance gates (review, remediation decision) — there is no application to deploy for a read-only documentation deliverable.

## 1.3 Key Accomplishments

- ✅ **Sole deliverable authored & validated** — `blitzy/documentation/app_2cd6ee777f8c.md` (4,519 lines / 61,702 words) answering all 7 investigation objectives with runtime-captured evidence.
- ✅ **Live enumeration oracle confirmed at runtime** — unsigned `bounce+{id}+@sl.local` path yields `550 SL E512 No such email log` (invalid id) vs `250 SL E213 Unknown email ignored` (valid, non-bounce), captured through the canonical `handle_DATA` entry point.
- ✅ **Oracle richness proven** — three-way discrimination including `550 SL E510 so such user` for an existent-but-inactive user; existence check provably **precedes** the `is_bounce` gate, so even a non-bounce probe enumerates.
- ✅ **Security boundary pinpointed** — the unsigned path reaches `EmailLog.get(id)` with **no** cryptographic gate, while the newer signed (BATV-style HMAC) format returns `None` on bad/expired signatures and diverts to `550 SL E515`.
- ✅ **Full condition matrix exercised** — 48 distinct scenario rows across {address format × id validity × `is_bounce` × SPF verdict × response layer}, each reproduced ≥2× with stable results (spot-checks 5×).
- ✅ **Canonical fidelity cross-checked** — the project's own SPF tests (`test_prevent_5xx_from_spf`, `test_preserve_5xx_with_valid_spf`, `test_preserve_5xx_with_no_header`) pass and match the investigation's independent observations; 6/6 relevant tests green on a fresh run.
- ✅ **Read-only constraint honored absolutely** — `git diff base..HEAD` shows exactly one added file; all 14 referenced source/test files are unchanged; temporary observation scripts lived outside the repo and were removed.
- ✅ **Security framing grounded in established practice** — mapped to CWE-204 (Observable Response Discrepancy), BATV, and VERP via background research.

## 1.4 Critical Unresolved Issues

The **deliverable itself has zero unresolved issues** — the Final Validator independently re-ran the canonical paths and found no discrepancies, so no edits were required. The single item requiring human action is **external to the deliverable**: the live product vulnerability the report discloses.

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| Disclosed enumeration oracle remains **unremediated** in the product (unsigned bounce path, CWE-204) | External attacker can enumerate valid `email_log_id`s and distinguish inactive users; bounded to internal integer ids | Security / backend maintainer | Pending triage (HT-2, 2h decision) |
| Human sign-off of the investigation report not yet performed | Findings not yet formally accepted | Security engineer / SME | Pending review (HT-1, 3h) |

> Note: designing or implementing a remediation is **explicitly out of the AAP scope** (Section 0.3.2). The rows above capture the human **decision/acceptance** gates only, not implementation effort.

## 1.5 Access Issues

**No access issues identified.**

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-----------------|----------------|-------------------|-------------------|-------|
| Git repository (branch `blitzy-b29137f8-…`) | Read/write | None — all git operations succeed; working tree clean | ✅ Resolved (no issue) | — |
| Canonical runtime (Py 3.10 container, PostgreSQL 13/15, Redis) | Runtime | None — provisioned and used successfully during investigation | ✅ Resolved (no issue) | — |
| Third-party services / credentials | — | Not required for a read-only documentation deliverable | ✅ N/A | — |

## 1.6 Recommended Next Steps

1. **[High]** Have a security engineer / SimpleLogin maintainer review and sign off on `blitzy/documentation/app_2cd6ee777f8c.md`, validating the oracle finding and spot-checking citations (HT-1, 3h).
2. **[Medium]** Convene a triage to decide the remediation approach for the disclosed oracle — e.g., uniform response on the unsigned path, deprecation of the legacy unsigned format, or an HMAC gate (HT-2, 2h; decision only).
3. **[Low]** Optionally reproduce the findings independently on the canonical Python 3.10 / PostgreSQL 13 container to build confidence before actioning (HT-3, 1h).
4. **[Low]** File a tracked security ticket referencing the report's Phase D (oracle) and Phase F (boundary) so remediation implementation can be scheduled as separate, in-scope future work.

---

# 2. Project Hours Breakdown

## 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|-------|-------------|
| [P2P] Canonical runtime provisioning | 6 | Stood up the canonical environment: Python 3.10 venv, Poetry-installed dependencies, PostgreSQL 13 (+15) on port 15432, Redis; replicated schema via `pg_dump \| psql` to work around the pre-existing repo-wide broken Alembic-from-scratch migration; wired the `tests/conftest.py` bootstrap. |
| [AAP methodology] Observation harness, fixtures & scripts | 10 | Built 11+ self-contained observation scripts; minted real `User`/`Alias`/`Contact`/`EmailLog` fixtures; constructed both address forms (unsigned + signed) plus corrupted/expired signed variants; wrote drivers for all three response layers (`handle` / `_handle` / `handle_DATA`). |
| [AAP methodology] Condition-matrix execution & evidence capture | 8 | Executed the 48-row scenario matrix across three layers, each cell ≥2× (spot-checks 5×); verified determinism via SHA-256 comparison over 3–4 runs; toggled `is_bounce` inputs; forced inactive-user (E510), future/old signed, and SPF verdicts; re-ran the full matrix on AAP-mandated PostgreSQL 13 (byte-identical). |
| [AAP Objectives 1–7] Investigation report authoring | 16 | Wrote the 61,702-word report: Direct Answer, Phases A–K, all 7 objectives with cited evidence, the 48-row matrix, the final coverage pass, the methodology attestation, and embedded verbatim transcripts + full script sources. |
| [AAP] Web-search security framing | 2 | Researched and integrated CWE-204 (Observable Response Discrepancy), BATV (draft-levine-smtp-batv-01), and VERP to frame findings against established practice (Phase H). |
| [AAP] Cleanup & repository-cleanliness verification | 1 | Removed all temporary observation scripts (container + host, both outside the repo); verified `git status`/`git diff` cleanliness and reference-file integrity. |
| [Quality] QA-finding resolution & Final Validation | 10 | Resolved QA findings across 4 doc-only commits (incl. a 19-finding and a 7-finding round); Final Validator independently re-ran the full canonical matrix, verified every claimed SMTP response and file:line citation, and cross-checked the 6/6 canonical tests. |
| **Total Completed** | **53** | |

## 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|-------|----------|
| Security/SME technical review & sign-off of the investigation report (HT-1) | 3 | High |
| Security triage & remediation **decision** on the disclosed oracle (HT-2; implementation out of AAP scope) | 2 | Medium |
| Optional independent reproduction on the canonical Py 3.10 / PG 13 container (HT-3) | 1 | Low |
| **Total Remaining** | **6** | |

## 2.3 Hours Reconciliation & Confidence

| Check | Result |
|-------|--------|
| Section 2.1 total (Completed) | 53 h |
| Section 2.2 total (Remaining) | 6 h |
| 2.1 + 2.2 = Section 1.2 Total | 53 + 6 = **59 h** ✅ |
| Section 1.2 Remaining = Section 2.2 total = Section 7 "Remaining Work" | 6 = 6 = 6 ✅ |
| Completion % = 53 / 59 | **89.8%** ✅ |

**Confidence:** *High* for completed work (git-verified deliverable, validator-reproduced evidence, byte-accurate citations spot-checked in this session). *High* for remaining-work sizing (well-defined human review/decision gates with no hidden engineering, since remediation is out of scope).

---

# 3. Test Results

This is a **read-only investigation**: per the AAP, **no new tests were authored or committed**. The results below originate entirely from Blitzy's autonomous validation logs — (a) the project's **own** canonical tests re-run to confirm harness fidelity, and (b) the autonomous **runtime observation matrix** that drove the canonical handler chain.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|------------|-------|
| Canonical SPF-rewrite tests | pytest | 3 | 3 | 0 | n/a (targeted) | `test_prevent_5xx_from_spf`, `test_preserve_5xx_with_valid_spf`, `test_preserve_5xx_with_no_header` — exercise the `_handle()` SPF `5XX→E216` path; match the investigation's independent observations. |
| Investigation-relevant suite (fresh run) | pytest | 6 | 6 | 0 | n/a (targeted) | Final Validator's fresh run: SPF + VERP round-trip tests all green, confirming the observation harness is the canonical one. |
| Autonomous runtime observation matrix | Custom canonical harness (aiosmtpd `handle`/`_handle`/`handle_DATA`) | 48 scenario cells | 48 | 0 | Full condition matrix | Every cell reproduced ≥2× (spot-checks 5×); all `STABLE = yes`; 11 distinct SMTP status strings captured verbatim and independently reproduced by the validator. |
| Cross-engine re-validation (PostgreSQL 13) | Custom canonical harness | Full matrix (re-run) | All | 0 | — | Entire matrix re-run twice on AAP-mandated PostgreSQL 13.23; every wire response byte-identical to the PostgreSQL 15 baseline. |

**Integrity note:** All rows above derive from Blitzy's autonomous test/validation execution logs for this project. No third-party or fabricated results are included. The 11 runtime-observed SMTP codes are: `E200`, `E205`, `E206`, `E211`, `E212`, `E213`, `E214`, `E216`, `E510`, `E512`, `E515` (four further codes — `E404`, `E502`, `E504`, `E524` — are source-quoted only and labeled as such in the report).

---

# 4. Runtime Validation & UI Verification

**Runtime validation** was the core of this project — the canonical inbound handler chain was driven end-to-end and produced real SMTP responses (not "not tested").

- ✅ **Operational** — Canonical entry chain `MailHandler.handle_DATA` → `_handle` → `handle` executed end-to-end across the full matrix.
- ✅ **Operational** — Unsigned-path oracle: invalid id → `550 SL E512 No such email log`; valid non-bounce id → `250 SL E213 Unknown email ignored` (captured via `handle_DATA`).
- ✅ **Operational** — Oracle richness: existent-but-inactive user → `550 SL E510 so such user`; active-user real bounce → `250 SL E211 Bounce Forward phase handled` / `250 SL E212 Bounce Reply phase handled`.
- ✅ **Operational** — Signed (BATV-style) path: corrupted/future-dated signatures → `get_verp_info_from_email()` returns `None` → diverts to `550 SL E515 Email not exist` (never reaches the oracle).
- ✅ **Operational** — SPF edge: fail/soft-fail rewrites returned `5XX` → `250 SL E216 Handled spf policy`; the valid-id branch raises before the rewrite, so the SL sub-code oracle survives at the string level.
- ✅ **Operational** — Committed state transition on a successful bounce (`bounced`/`refused_email_id`/`bounced_mailbox_id`, +1 Bounce / +1 RefusedEmail) captured with before/after values across two runs.
- ✅ **Operational** — Determinism: exact-input repetitions 5/5 identical; SHA-256 comparison across runs confirms stability.
- ⚠ **Partial (by design / out of scope)** — Observations were driven through the `handle_DATA` message callback within the app context rather than over a live TCP SMTP socket; this **is** the canonical callback an external sender's `DATA` reaches, and results were cross-checked against the project's own tests.

**UI Verification:** ❌ **Not applicable.** This is a backend SMTP investigation delivered as documentation; the project introduces **no user interface**. No screens, components, or client flows exist to verify.

---

# 5. Compliance & Quality Review

Cross-mapping of AAP deliverables and rules to their validation status. Fixes applied during autonomous validation are noted; there are no outstanding items on the deliverable.

| AAP Deliverable / Rule | Benchmark | Status | Progress | Evidence / Notes |
|------------------------|-----------|--------|----------|------------------|
| Objective 1 — two address formats | Answered by name + runtime evidence | ✅ Pass | 100% | Phase B; `parse_id_from_bounce('bounce+12345+@sl.local')=12345` (user example verbatim); signed round-trips. |
| Objective 2 — routing (4 branches) | Answered by name + runtime evidence | ✅ Pass | 100% | Phase C; transactional/forward/reply/iCloud; matrix rows 14–19, 47–48. |
| Objective 3 — external-attacker feasibility | Answered + canonical capture | ✅ Pass | 100% | Phase D.1; envelope-only probe distinguishes valid/invalid id. |
| Objective 4 — security boundary | Answered + boundary pinpointed | ✅ Pass | 100% | Phase F; unsigned reaches `EmailLog.get` w/o HMAC; signed returns `None` on 6 conditions. |
| Objective 5 — exact SMTP responses | Verbatim strings captured | ✅ Pass | 100% | Phase I.1; 11 codes runtime-observed, 4 source-quoted & labeled. |
| Objective 6 — `is_bounce` spoofability | Criteria + attacker-controllability shown | ✅ Pass | 100% | Phase E; `mail_from=="<>"` AND `multipart/report`, both attacker-controllable. |
| Objective 7 — richness + ordering | Multi-way discrimination + ordering | ✅ Pass | 100% | Phase D.2–D.4; E510 inactive-user; existence check precedes `is_bounce`. |
| Run-first, canonical-path-only | Values from real entry chain; non-canonical labeled | ✅ Pass | 100% | Phase A.2 + I.5 attestation; direct-call values explicitly labeled. |
| Exercise every condition (incl. edges) | Full matrix + edge cases | ✅ Pass | 100% | 48-row matrix, all `STABLE=yes`. |
| Complete, unedited evidence + `file:line` | Verbatim output + citations | ✅ Pass | 100% | 144 file:line citations; 87 transcript blocks; Phase K verbatim. |
| Determinism (≥2× runs, stability) | Repeat + report distribution | ✅ Pass | 100% | STABLE across runs; K.0 SHA-256; K.9 5× repetition. |
| Web-search framing (CWE-204/BATV/VERP) | Grounded against established practice | ✅ Pass | 100% | Phase H with MITRE CWE-204 + Nessus 10249 references. |
| Final coverage pass | Every question + named item confirmed | ✅ Pass | 100% | Phase I.4 all `[x]` + named-item paragraph. |
| **Read-only constraint (absolute)** | Only the answer doc changes | ✅ Pass | 100% | `git diff base..HEAD` = one added file; 14 reference files unchanged. |
| Deliverable location/name rule | `blitzy/documentation/<branch>.md` | ✅ Pass | 100% | `blitzy/documentation/app_2cd6ee777f8c.md` present. |
| Secret handling (DOC-SEC-1) | No secrets emitted | ✅ Pass (fixed in validation) | 100% | Secrets masked before emission during QA. |
| Env discrepancy disclosure | Report actual, don't normalize | ✅ Pass | 100% | Phase A.3; PG13 mandated vs PG15 in-image + aiosmtpd 1.4.6 vs 1.4.2 disclosed; PG13 re-run byte-identical. |
| Remediation | Out of scope | ✅ Pass (correctly excluded) | 100% | No fix proposed/implemented; noted only that signed format closes the gap. |

**Quality summary:** All AAP deliverables and rules **pass**. QA fixes applied during autonomous validation (secret masking, citation corrections, cleanliness scoping) are resolved. No outstanding compliance items remain on the deliverable.

---

# 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| **Live enumeration oracle in the product** — unsigned bounce path reaches `EmailLog.get(id)` with no HMAC gate (`E512` vs `E213`; `E510` for inactive user); existence check precedes `is_bounce` gate. CWE-204. | Security | **High** | High | Newer signed VERP (BATV-style HMAC) closes the gap on that format; report recommends human triage; leak bounded to internal integer `email_log_id`s (not mailbox names). Remediation is out of AAP scope. | **Open** — human remediation decision required (HT-2) |
| Environment reproducibility gap — canonical reproduction needs the Py 3.10 container; current host is Py 3.13.7 and repo-wide Alembic-from-scratch is broken. | Technical | Low | Medium | Report documents the exact docker invocation + `pg_dump \| psql` schema-replication workaround (Phase K.1a); findings are DB-version-independent. | Mitigated / Documented |
| Version drift (aiosmtpd 1.4.6 mandated vs 1.4.2 actual; PG 13 mandated vs 15.13 in-image). | Technical | Low | Low | Full matrix re-run on PG 13.23 = byte-identical; both are off the status-string path (routing lives in `email_handler.py` + `status.py`). | Mitigated |
| Secret exposure in embedded transcripts. | Security | Low | Low | Validator resolved DOC-SEC-1; secrets masked before emission; no `VERP_EMAIL_SECRET`/keys leaked. | Mitigated |
| Pre-existing repo-wide Alembic migration defect blocks clean-slate DB bring-up. | Operational | Low | Medium | Disclosed with workaround (Phase K.1a); out of task scope; does not affect the finding. | Documented (not owned by this task) |
| Point-in-time snapshot — findings anchored to base commit `2cd6ee777f8c`; future code edits could drift responses. | Operational | Low | Low | Every claim anchored to a specific `file:line` + commit; reproducible against the pinned commit. | Accepted |
| Harness drives `handle_DATA` in-process rather than over a live TCP socket. | Integration | Low | Low | `handle_DATA` is the canonical aiosmtpd callback; cross-checked vs the project's own 3 SPF tests (identical) + 6/6 tests pass. | Mitigated / Labeled |
| Signer-clock shim could be mistaken for a verifier mock. | Integration | Low | Low | Fully disclosed (Phases A.2/I.5/K.4); scoped only to minting FUTURE/OLD input addresses; verifier + handler ran unpatched; released before submission. | Disclosed / Mitigated |

**Assessment:** The only High-severity item is the **subject** of the investigation (the live product oracle), correctly disclosed with remediation deferred to humans per scope — a finding to action, not a defect in the deliverable. All deliverable-quality risks are Low and Mitigated/Documented. **No risk blocks acceptance of the report.**

---

# 7. Visual Project Status

### 7.1 Project Hours Breakdown

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieTitleTextSize':'18px','pieSectionTextSize':'14px','pieSectionTextColor':'#B23AF2','pieLegendTextColor':'#111111'}}}%%
pie showData title Project Hours — Completed vs Remaining
    "Completed Work" : 53
    "Remaining Work" : 6
```

- **Completed Work** (Dark Blue `#5B39F3`): **53 h**
- **Remaining Work** (White `#FFFFFF`): **6 h**
- **Total:** 59 h — **89.8% complete**

> Integrity: the "Remaining Work" value (6) equals Section 1.2 Remaining Hours (6) and the sum of the Section 2.2 "Hours" column (3 + 2 + 1 = 6).

### 7.2 Remaining Work by Priority (hours)

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#B23AF2','pie3':'#A8FDD9','pieStrokeColor':'#333333','pieStrokeWidth':'1px','pieSectionTextColor':'#111111','pieLegendTextColor':'#111111'}}}%%
pie showData title Remaining 6h by Priority
    "High — SME review & sign-off" : 3
    "Medium — remediation decision" : 2
    "Low — optional reproduction" : 1
```

| Priority | Hours |
|----------|-------|
| High | 3 |
| Medium | 2 |
| Low | 1 |
| **Total** | **6** |

---

# 8. Summary & Recommendations

**Achievements.** The project delivered a rigorous, evidence-based security investigation that answers all seven objectives with runtime-captured proof. It confirms that SimpleLogin's inbound bounce path **does** expose a recipient/identifier enumeration oracle on the **older, unsigned** VERP address format: an external sender controlling only the SMTP envelope can distinguish a valid `email_log_id` (`250 SL E213`) from an invalid one (`550 SL E512`), with three-way richness that also reveals inactive users (`550 SL E510`). The **newer, signed** BATV-style format closes the gap. Every claim is backed by verbatim SMTP output and `file:line` citations, and the whole matrix was independently reproduced by the Final Validator with no discrepancies.

**Remaining gaps.** With the engineering investigation complete and validated, the project is **89.8% complete** (53 of 59 hours). The remaining **6 hours** are entirely human path-to-acceptance activities: SME review/sign-off (3h), a remediation **decision** on the disclosed oracle (2h), and an optional independent reproduction (1h). No further engineering is required for the deliverable itself.

**Critical path to production.** For a read-only documentation deliverable, "production" means the finding is reviewed, accepted, and actioned. The critical path is: (1) SME sign-off → (2) remediation triage decision → (3) creation of a tracked security ticket for the (separately scoped) fix. There is no build, service, or pipeline to deploy.

**Success metrics.**

| Metric | Target | Result |
|--------|--------|--------|
| Objectives answered with runtime evidence | 7 / 7 | ✅ 7 / 7 |
| Read-only constraint (files changed outside the doc) | 0 | ✅ 0 |
| Runtime SMTP codes captured & reproduced | All observed | ✅ 11 / 11 |
| Canonical tests passing | Pass | ✅ 6 / 6 |
| Cross-section numeric integrity | Consistent | ✅ 53 + 6 = 59; 89.8% |

**Production-readiness assessment.** The deliverable is **production-ready** within the read-only investigation's definition of done: complete, accurate, evidence-based, correctly cited, and byte-for-byte clean. It should not be marked 100% until a human reviewer has signed off on the security finding — hence 89.8%. **Recommendation: accept the report, then triage the disclosed oracle for a separately scoped remediation.**

---

# 9. Development Guide

> This is a read-only investigation delivered as a single Markdown document. There is no application to install or start. "Development" here means **reading, verifying, and (optionally) reproducing** the findings. All commands below were executed successfully during guide preparation, unless marked *(canonical env)*.

## 9.1 System Prerequisites

- **Canonical runtime (for reproduction):** the swe-atlas image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (container `sl-app-0`), providing **Python 3.10.x**, **PostgreSQL 13** on port **15432**, **Redis 7.x**, and the Poetry-installed dependency set (`aiosmtpd 1.4.2` pinned by `poetry.lock`).
- **Git + Git LFS.**
- **For reading/verifying only:** any host with `git`, `grep`/`sed`, and a Markdown viewer. (This host is Python 3.13.7 — fine for reading/verifying, but runtime reproduction requires the Python 3.10 container.)

## 9.2 Environment Setup

```bash
# The canonical harness config (values verified):
#   EMAIL_DOMAIN=sl.local   NOT_SEND_EMAIL=true
#   DB_URI=postgresql://test:test@localhost:15432/test
export CONFIG=tests/test.env
```

The Flask app, SQLAlchemy session, `pg_trgm` extension, and Alembic head are bootstrapped by `tests/conftest.py` (`create_app` + `add_sl_domains` + `add_proton_partner`), exactly as the observation harness uses.

## 9.3 Dependency Installation

```bash
# Canonical: dependencies are pre-installed in the image venv at /app/venv (Poetry).
# If rebuilding from source (Python 3.10):
poetry install
# This task is dependency-neutral — no package was added, upgraded, or removed.
```

## 9.4 Read / Verify / Reproduce

```bash
# 1) Locate and read the deliverable
ls -la blitzy/documentation/app_2cd6ee777f8c.md
sed -n '7,26p' blitzy/documentation/app_2cd6ee777f8c.md        # Direct Answer

# 2) Navigate the report phases
grep -nE '^## Phase [A-K]' blitzy/documentation/app_2cd6ee777f8c.md

# 3) Reproduce (canonical env) — the invocation form embedded in the report:
docker exec -e CONFIG=/work/tests/test.env -e PYTHONPATH=/work -e GITHUB_ACTIONS_TEST=true \
  -w /work sl-app-0 /app/venv/bin/python /tmp/blitzy_evidence/<script>.py
# 16 self-contained observation scripts are embedded verbatim in Phase K.
```

## 9.5 Verification Steps (all verified during guide prep)

```bash
# Read-only constraint — must list ONLY the one document:
git diff --name-status origin/app_2cd6ee777f8c..HEAD
#   A    blitzy/documentation/app_2cd6ee777f8c.md

# Clean working tree (empty output = clean):
git status --porcelain

# Spot-check cited SMTP status strings against source:
sed -n '21p;49p' app/email/status.py
#   E213 = "250 SL E213 Unknown email ignored"
#   E512 = "550 SL E512 No such email log"

# Spot-check the is_bounce criteria citation:
sed -n '1813,1818p' email_handler.py

# Confirm VERP helper locations:
grep -nE 'def (generate_verp_email|get_verp_info_from_email|parse_id_from_bounce)' app/email_utils.py

# Run the canonical SPF tests (canonical env):
python -m pytest tests/test_email_handler.py -k "prevent_5xx or preserve_5xx" \
  -v --no-header -p no:cacheprovider
#   3 passed, 20 deselected
```

## 9.6 Example Usage — reproduce the core oracle

```text
# Invalid id probe (envelope recipient):
bounce+99999999999999+@sl.local   ->   550 SL E512 No such email log

# Valid id, non-bounce probe (substitute a real email_log_id):
bounce+<valid_email_log_id>+@sl.local   ->   250 SL E213 Unknown email ignored
```

The divergence between these two responses **is** the enumeration oracle. The full driver is embedded in the report at **Phase K.3 (Script B)**.

## 9.7 Troubleshooting

- **"`python` is 3.13, not 3.10"** — use the canonical container/venv for runtime reproduction; the host Python is only for reading/verifying.
- **"`alembic upgrade head` fails on a fresh DB"** — this is a **pre-existing, repo-wide** defect (the `note_pg_trgm_index` migration runs before the `gen_email`→`alias` rename under a single-transaction `env.py`), **not** introduced by this task. It fails identically on PostgreSQL 13 and 15. Workaround: replicate the schema with `pg_dump | psql` (see report Phase K.1a).
- **"Which DB port?"** — `15432` (per `tests/test.env`).
- **"No emails are sent"** — expected: `NOT_SEND_EMAIL=true` in the harness by design.

---

# 10. Appendices

## Appendix A — Command Reference

| Purpose | Command |
|---------|---------|
| Locate deliverable | `ls -la blitzy/documentation/app_2cd6ee777f8c.md` |
| Read Direct Answer | `sed -n '7,26p' blitzy/documentation/app_2cd6ee777f8c.md` |
| List report phases | `grep -nE '^## Phase [A-K]' blitzy/documentation/app_2cd6ee777f8c.md` |
| Read-only proof | `git diff --name-status origin/app_2cd6ee777f8c..HEAD` |
| Clean-tree check | `git status --porcelain` |
| Branch commits over base | `git log --oneline origin/app_2cd6ee777f8c..HEAD` |
| Citation spot-check (status) | `sed -n '21p;49p' app/email/status.py` |
| Citation spot-check (is_bounce) | `sed -n '1813,1818p' email_handler.py` |
| VERP helper locations | `grep -nE 'def (generate_verp_email\|get_verp_info_from_email\|parse_id_from_bounce)' app/email_utils.py` |
| Canonical SPF tests | `python -m pytest tests/test_email_handler.py -k "prevent_5xx or preserve_5xx" -v --no-header -p no:cacheprovider` |

## Appendix B — Port Reference

| Service | Port | Notes |
|---------|------|-------|
| PostgreSQL (test DB) | 15432 | `DB_URI=postgresql://test:test@localhost:15432/test` |
| Redis | 6379 (default) | Provisioned in canonical container |
| SMTP inbound (aiosmtpd) | n/a (in-process) | Observed via the `handle_DATA` callback, not a bound socket |

## Appendix C — Key File Locations

| File | Role |
|------|------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The sole deliverable** — investigation report |
| `email_handler.py` | Canonical entry chain, routing, `is_bounce`, `handle_bounce` (REFERENCE) |
| `app/email_utils.py` | `parse_id_from_bounce`, `generate_verp_email`, `get_verp_info_from_email` (REFERENCE) |
| `app/email/status.py` | SMTP status-string constants (REFERENCE) |
| `app/config.py` | `BOUNCE_PREFIX`/`BOUNCE_SUFFIX`, `VERP_*` (REFERENCE) |
| `app/errors.py` | `VERPForward`/`VERPReply`/`VERPTransactional` (REFERENCE) |
| `app/models.py` | `VerpType`, `EmailLog`, `Alias`, `User`, `Contact` (REFERENCE) |
| `app/handler/spamd_result.py` | SPF verdict parser feeding the `E216` rewrite (REFERENCE) |
| `tests/conftest.py`, `tests/test.env`, `tests/utils.py` | Canonical harness reused by observation scripts (REFERENCE) |

## Appendix D — Technology Versions

| Component | Mandated (AAP) | Actual (observed) | Impact on findings |
|-----------|----------------|-------------------|--------------------|
| Python | 3.10 | 3.10.18 (canonical container) | None |
| aiosmtpd | 1.4.6 | 1.4.2 (`poetry.lock`-pinned) | None — off the status-string path |
| PostgreSQL | 13 | 15.13 in-image; **13.23 also provisioned** | None — matrix byte-identical on both |
| Redis | — | 7.0.15 | None |
| Alembic head | — | `32f25cbf12f6` | Schema replicated via `pg_dump\|psql` (fresh migration broken repo-wide) |

## Appendix E — Environment Variable Reference

| Variable | Value | Purpose |
|----------|-------|---------|
| `CONFIG` | `tests/test.env` | Selects the canonical test configuration |
| `EMAIL_DOMAIN` | `sl.local` | Must match `bounce+{id}+@sl.local` and signed addresses |
| `NOT_SEND_EMAIL` | `true` | Prevents outbound mail during observation |
| `DB_URI` | `postgresql://test:test@localhost:15432/test` | Real DB backing valid/invalid id distinction |
| `PYTHONPATH` | `/work` | Repo root inside the canonical container |
| `GITHUB_ACTIONS_TEST` | `true` | Harness flag mirrored from CI |

## Appendix F — Developer Tools Guide

| Tool | Use in this project |
|------|---------------------|
| `git diff --name-status` / `git status --porcelain` | Prove the read-only constraint and clean tree |
| `grep` / `sed` | Navigate the report and spot-check `file:line` citations against source |
| `pytest` | Re-run the project's own canonical SPF/VERP tests to confirm harness fidelity |
| `docker exec` | Drive observation scripts inside the canonical `sl-app-0` container |
| `pg_dump` / `psql` | Replicate the DB schema (workaround for the pre-existing broken migration) |
| `sha256sum` | Determinism comparison across repeated observation runs (report Phase K.0) |

## Appendix G — Glossary

| Term | Definition |
|------|-----------|
| **VERP** | Variable Envelope Return Path — encodes per-message routing info (here an `email_log_id`) into the bounce address. |
| **BATV** | Bounce Address Tag Validation — signs the return path with a cryptographic tag + timestamp so forged bounces fail verification. The newer signed format is BATV-style. |
| **Enumeration oracle** | A system whose differing responses let an attacker infer internal state (here, whether an `email_log_id` exists / is active). |
| **CWE-204** | Observable Response Discrepancy — MITRE weakness matching the observed `250`-vs-`550` divergence. |
| **Canonical path** | The real inbound chain `handle_DATA` → `_handle` → `handle`; values obtained any other way are labeled *non-canonical*. |
| **`E512` / `E213` / `E510` / `E515`** | SimpleLogin SMTP sub-codes: no-such-email-log / unknown-email-ignored / no-such-user / email-not-exist. |
| **Signer-clock shim** | The single disclosed `patch("app.email_utils.time")` used only to mint future/old signed **input** addresses; the verifier and handler ran unpatched. |

---

*This Blitzy Project Guide reflects the AAP-scoped completion analysis for branch `blitzy-b29137f8-040e-4e4e-a3c9-fe1f749a529b`. Completion: **89.8%** (53 of 59 hours). Brand colors: Completed `#5B39F3`, Remaining `#FFFFFF`.*