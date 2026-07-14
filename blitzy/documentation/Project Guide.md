# Blitzy Project Guide — SimpleLogin Alias Reply-Handling Runtime Root-Cause Analysis

> **Document type:** Read-only bug-investigation / Q&A analysis (Documentation deliverable)
> **Source branch:** `app_2cd6ee777f8c` · **HEAD:** `54be6937` · **Base:** `2cd6ee777f8c`
> **Sole deliverable:** `blitzy/documentation/app_2cd6ee777f8c.md` (2,139 lines)
> **Brand color key:** <span style="color:#5B39F3">■ Completed / AI Work = Dark Blue `#5B39F3`</span> · ⬜ Remaining = White `#FFFFFF`

---

## 1. Executive Summary

### 1.1 Project Overview

SimpleLogin is a Python email-alias privacy service whose backend relays replies to alias-forwarded mail back to external contacts while keeping users' real mailboxes hidden. This project is a **read-only, runtime-first investigation** answering a user's question: are some alias replies mis-routed to the wrong user, and where does an incorrect routing decision originate? Rather than modify code, the work stands up the real pipeline, drives synthetic inbound replies through the canonical SMTP entry point, captures live runtime evidence, and delivers a single evidence-backed markdown analysis. The target audience is the reporting engineer and SimpleLogin maintainers; the business impact is a confidentiality-relevant diagnosis of the reverse-alias reply path with an exact, citation-anchored root cause.

### 1.2 Completion Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeWidth':'2px','pieTitleTextSize':'16px','pieSectionTextSize':'14px'}}}%%
pie showData title Completion — 91.7% Complete
    "Completed (AI)" : 66
    "Remaining" : 6
```

| Metric | Value |
|--------|-------|
| **Total Hours** | **72** |
| **Completed Hours (AI + Manual)** | **66** (66 AI + 0 Manual) |
| **Remaining Hours** | **6** |
| **Percent Complete** | **91.7%** (66 ÷ 72) |

> The 91.7% is computed strictly from AAP-scoped hours: `Completed ÷ (Completed + Remaining) = 66 ÷ 72`. All 21 AAP requirements are complete; the sub-100% figure reflects genuine human path-to-production (review/acceptance) on a large technical analysis, capped at 99% per Blitzy assessment policy.

### 1.3 Key Accomplishments

- ✅ **All five sub-questions (Q1–Q5) answered explicitly and by name**, each with observed runtime evidence and `file:line` citations.
- ✅ **Canonical entry point exercised** — every scenario driven through the real `MailHandler.handle_DATA` SMTP callback (zero mocks / monkeypatch / bypass).
- ✅ **Runtime-first methodology honored** — the pipeline was built and run first (79 real handler log lines captured), then the analysis was written from what was observed.
- ✅ **Root cause pinpointed** — `Contact.get_by(reply_email=…).first()` (`email_handler.py:L986`) over a non-unique `reply_email` column (`app/models.py:L1899`) resolved by `.first()` with no `ORDER BY` (`app/models.py:L84`), reproduced canonically in `S3`/`S3b`.
- ✅ **Dual user reference captured** — persistence records `EmailLog.user_id=contact.user_id` (`L1046`) while delivery gates authorize against `alias.user` (`L1004`); divergence reproduced in `S2`.
- ✅ **Two concrete reply-body defects reproduced** — cross-user recipient disclosure (`RUNTIME-01`) and silent non-delivery on domain letter-case (`RUNTIME-02`).
- ✅ **11 scenarios / 104 assertions pass, 0 fail**; existing project suite `tests/test_email_handler.py` **23/23 passed**.
- ✅ **Read-only mandate verified** — `git diff 2cd6ee77..HEAD` touches exactly one file (the deliverable); zero source changes; working tree git-clean.
- ✅ **Cleanup complete** — all temporary observation scaffolding removed; the full harness is preserved verbatim inside the deliverable's raw-output section for reproducibility.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| _None that block the deliverable._ The deliverable is complete, well-formed, fully cited, and read-only compliant. | N/A | N/A | N/A |
| (Informational — **out of AAP scope**) Diagnosed product fragility: non-unique `reply_email` + `.first()` can mis-route/mis-attribute replies | Potential wrong-user routing & privacy exposure in production | Human maintainer (triage) | Pending human decision |
| (Informational — **out of AAP scope**) `RUNTIME-01` cross-user recipient disclosure & `RUNTIME-02` silent drop in the To-header rewrite (`email_handler.py:L364`) | Confidentiality / availability of replies | Human maintainer (triage) | Pending human decision |

> These are **diagnosed findings the analysis was commissioned to surface**, not defects in the deliverable. Fixing them is explicitly out of scope per AAP §0.3.2 (diagnose-and-answer only).

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-----------------|----------------|-------------------|-------------------|-------|
| Canonical GHCR image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` | Container pull / runtime | Runtime reproduction requires Python 3.10 + PostgreSQL 13 + Redis 6; the default assessment container is Python 3.13 with no PG/Redis/Poetry/internet | Mitigated — investigation ran in the canonical image; harness embedded verbatim in the deliverable for re-run | Blitzy / reviewer |

No repository, credential, or third-party API access issues were identified. The task requires no external service credentials (email egress is disabled via `NOT_SEND_EMAIL=true`).

### 1.6 Recommended Next Steps

1. **[Medium]** SME review & acceptance of the 2,139-line analysis — validate the Q1–Q5 answers and the bottom-line verdict against maintainer knowledge of the reply pipeline (~2.5h).
2. **[Medium]** Spot-check a sample of the 212 `file:line` citations at HEAD `2cd6ee77` and confirm the observed/inferred labeling (~1.5h).
3. **[Low]** Triage decision: determine whether the three diagnosed latent fragilities warrant remediation tickets — the fix itself is out of this task's scope (~1h).
4. **[Low]** (Optional) Independently reproduce the embedded harness in the canonical GHCR image to confirm the documented runtime results (~1h).

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|-------|-------------|
| Environment provisioning & schema stand-up | 4 | Canonical GHCR image; PostgreSQL remapped to `:15432` and started; Redis started; venv; `CONFIG=tests/test.env alembic upgrade head` → head `32f25cbf12f6` (77 tables) |
| Runtime observation harness engineering | 14 | Author the in-process harness: 11 scenarios (`S1, S2, S3, S3b, S8a–f, S9, S10, S11, RUNTIME-01/02/INFO-01`), 104 assertions, fixture builders, SL-logger capture, outbound-egress probe — all driven through the canonical `handle_DATA` |
| Q1–Q4 runtime tracing & signal capture | 8 | Drive scenarios ≥2× on identical input; capture handler log lines, persisted `EmailLog` rows, and captured outbound `SendRequest`; confirm routing-tuple stability |
| Q5 root-cause differential analysis | 8 | Reproduce four hypotheses: dual user reference (`S2`), non-unique `reply_email` + `.first()` (`S3`/`S3b`), spoofing-check fallback (`S9`), and un-scoped To-header rewrite (`RUNTIME-01/02`) |
| Edge / error condition coverage | 7 | Guard paths `E501–E504` + control-char normalization (`S8a–f`), multi-mailbox alias + `notify_mailbox` (`S10`), `NOREPLIES` + bounce `<>` (`S11`) |
| Analysis document authoring | 12 | 2,139-line evidence-backed deliverable: verdict, Q1–Q5 sections, 12-step data-flow narrative + Mermaid diagram, edge-condition table, 22-claim observed/inferred ledger, security/privacy note, coverage recap |
| Citation verification & accuracy | 5 | 212 `file:line` citation occurrences validated at HEAD `2cd6ee77`; three corrections applied (`NonReverseAliasInReplyPhase` → L372, `normalize_reply_email` → L25–38, SpamAssassin import-chain note) |
| QA & code-review remediation | 7 | Five-commit remediation history: initial doc → code-review-rejection remediation → QA F1/F2 → assertion-backed evidence → citation fixes |
| Web-search conceptual validation | 1 | Validate the reverse-alias reply model against official documentation (background framing only) |
| **Total Completed** | **66** | **= Section 1.2 Completed Hours** |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|-------|----------|
| SME review & acceptance of the analysis document (read 2,139 lines; validate Q1–Q5 + verdict; spot-check citations) | 4 | Medium |
| Independent reproduction of the runtime trace + fix-triage decision (optional harness re-run in canonical image; decide whether to file remediation tickets — fix is out of scope) | 2 | Low |
| **Total Remaining** | **6** | **= Section 1.2 Remaining Hours = Section 7 "Remaining Work"** |

> **Cross-section integrity:** Section 2.1 (66) + Section 2.2 (6) = **72** Total Hours (Section 1.2). Remaining = **6** in Sections 1.2, 2.2, and 7. There is **no remaining autonomous AAP engineering work** — all 21 AAP requirements are complete; the 6h is entirely human path-to-production.

---

## 3. Test Results

All results below originate from **Blitzy's autonomous validation logs** for this project (executed in the canonical GHCR image; not re-run in the assessment container, which lacks Python 3.10 / PostgreSQL / Redis).

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|------------|-------|
| Reply-flow runtime scenarios | pytest (canonical `handle_DATA`) | 11 | 11 | 0 | N/A (path tracing) | `S1, S2, S3, S3b, S8a–f, S9, S10, S11, RUNTIME-01, RUNTIME-02, RUNTIME-INFO-01`; each ≥2× or once per toggled branch |
| Embedded harness assertions | pytest `assert` ledger | 104 | 104 | 0 | N/A | Every behavioral claim anchored to an `ASSERT PASS` line; 0 failures |
| Existing project suite (reply handler) | pytest 7.3.1 | 23 | 23 | 0 | N/A (regression) | `tests/test_email_handler.py`; 18 warnings; identical to baseline (proves read-only) |
| **Aggregate** | — | **138** | **138** | **0** | — | 100% pass rate across autonomous validation |

**Coverage note (honest):** This is a runtime **path-tracing** investigation, not a coverage-driven test effort, so no formal line-coverage percentage was produced. Instead, **branch/path coverage of the reply handler** was the explicit goal and was achieved: the happy path, all four guard branches (`E501/E502/E503/E504`), normalization, spoofing-check ON/OFF, multi-mailbox selection, `NOREPLIES`/bounce short-circuits, and the two To-header-rewrite defect paths were each exercised through the canonical entry point.

---

## 4. Runtime Validation & UI Verification

**No UI component** — this is a backend email-pipeline investigation delivered as documentation. Runtime validation focused on the SMTP reply path.

Runtime health (from autonomous validation logs):

- ✅ **Operational** — Canonical SMTP entry point `MailHandler.handle_DATA` exercised end-to-end in-process.
- ✅ **Operational** — Schema built via canonical CI recipe (`alembic upgrade head` → `32f25cbf12f6`, 77 tables).
- ✅ **Operational** — Happy path `S1`: `250 Message accepted for delivery`, `EmailLog.user_id=47` (equal to `contact.user_id` and `alias.user_id`), relayed to the external contact, `From` rewritten to the alias, real mailbox absent from the entire 747-byte message.
- ✅ **Operational** — Outbound intent captured via `mail_sender.get_stored_emails()` with `NOT_SEND_EMAIL=true`; **no real email left the environment**.
- ✅ **Operational** — All guard paths returned exact SMTP statuses (`E501/E502/E503/E504`) with no `EmailLog` created.
- ✅ **Operational** — Regression suite `tests/test_email_handler.py`: 23/23 passed.
- ⚠ **Partial (by design — diagnosed, not fixed)** — `S2` dual-user divergence reproduced (`EmailLog.user_id=49` while gates used `alias.user_id=48`); `S3`/`S3b` non-unique `reply_email` collision reproduced (selected owner delivered, shadowed owner rejected `E214`).
- ⚠ **Partial (by design — diagnosed, not fixed)** — `RUNTIME-01` cross-user recipient disclosure and `RUNTIME-02` silent drop reproduced; these are the diagnosed product defects the analysis was commissioned to surface.

---

## 5. Compliance & Quality Review

Cross-mapping of AAP deliverables and mandated rules to their validation status.

| AAP Deliverable / Rule Benchmark | Status | Progress | Notes / Evidence |
|----------------------------------|--------|----------|------------------|
| Q1 — Entry point identified | ✅ Pass | 100% | `handle_DATA` L2289 → `_handle` L2335 → `handle` L1945 (citations verified) |
| Q2 — Alias→user resolution chain | ✅ Pass | 100% | `is_reverse_alias` L1156 → `handle_reply` L966 → `Contact.get_by` L986 → `contact.alias` L994 → `alias.user` L1004 |
| Q3 — Concrete integer `user_id` | ✅ Pass | 100% | Observed `user_id=47`; dual reference documented (`L1046` vs `L1004`) |
| Q4 — End-to-end data flow | ✅ Pass | 100% | 12-step gate-by-gate narrative + Mermaid diagram, each node tied to `S1`/`S2` evidence |
| Q5 — Most-likely root cause | ✅ Pass | 100% | 4 hypotheses reproduced; single origin named (`L986` + `L1899` + `L84`) |
| Runtime-first methodology | ✅ Pass | 100% | Code built & run first; 79 real handler log lines captured before writing |
| Canonical entry point only (no bypass) | ✅ Pass | 100% | All scenarios via real `handle_DATA`; zero mocks/monkeypatch |
| Reproduce reported inconsistency directly | ✅ Pass | 100% | `S3b` runs the same unchanged fixture 5× and reports the (stable, data-dependent) distribution |
| Exercise every condition (primary + edge/error) | ✅ Pass | 100% | Happy path + `S8a–f` + `S9` + `S10` + `S11` + `RUNTIME-01/02` |
| Evidence quality (actual output + `file:line` + observed/inferred) | ✅ Pass | 100% | 212 citations, observed/inferred legend, 22-claim ledger, raw transcript |
| Answer every part & every named item | ✅ Pass | 100% | Coverage-recap final pass covers Q1–Q5 + all edge conditions |
| Deliverable at mandated path | ✅ Pass | 100% | `blitzy/documentation/app_2cd6ee777f8c.md` |
| Read-only scope (no source modified) | ✅ Pass | 100% | `git diff 2cd6ee77..HEAD` = 1 file (deliverable); zero source changes |
| Cleanup / git-clean | ✅ Pass | 100% | No leftover harness scripts; working tree clean |
| Markdown well-formedness | ✅ Pass | 100% | 40 balanced code fences, 1 Mermaid diagram, 46 table rows |

**Fixes applied during autonomous validation (deliverable-only, prose):** three genuine citation inaccuracies corrected — `NonReverseAliasInReplyPhase` L370→L372, `normalize_reply_email` L25–39→L25–38, and a SpamAssassin import-chain note — committed as `54be6937`. **Outstanding compliance items:** none.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Runtime harness not reproducible in the default assessment env (needs canonical GHCR Python 3.10 + PG 13 + Redis 6) | Technical | Low | Medium | Full harness embedded verbatim in deliverable §(h); documented run commands + image id `ea242796bbce`; static citations independently verified | Mitigated / Documented |
| Diagnosed non-unique `reply_email` + `.first()` fragility remains unfixed (real prod mis-routing possible) | Technical | Medium | Low | Correctly diagnosed & documented (Q5b); fix explicitly out of scope §0.3.2; needs human triage | Open (by design) |
| `.first()` selection order observed-but-not-guaranteed could mislead a reader into assuming determinism | Technical | Low | Low | Deliverable explicitly labels the selection observed-not-guaranteed (`models.py:L84`, no `ORDER BY`; ledger claim #7) | Mitigated |
| Cross-user recipient disclosure (`RUNTIME-01`) + silent drop (`RUNTIME-02`) latent in product | Security | Medium | Low | Reproduced & documented §(f)/§(j) with `file:line` (`L364`, `L1183–L1198`); flagged privacy-relevant; fix out of scope | Open (by design) |
| Container-only provisioning fixes (`dkim.key` PKCS#1, `re2`→`re` shim) not in repo source | Operational | Low | Low | Documented §(k) as container-only; host git-tracked source already correct; use provided GHCR image | Mitigated / Documented |
| Nuanced non-binary verdict may not satisfy a stakeholder expecting a yes/no | Operational | Low | Low | Bottom-line-up-front verdict §(a) with explicit misunderstanding-vs-real-bug decomposition | Mitigated |
| Reproduction depends on canonical image state; image drift could alter absolute Postgres IDs | Integration | Low | Low | Deliverable predicts absolute IDs differ run-to-run; only structural invariants asserted, not absolute IDs | Mitigated |

---

## 7. Visual Project Status

**Project hours breakdown** (`Remaining Work` = 6 matches Section 1.2 and Section 2.2 exactly):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieTitleTextSize':'16px','pieSectionTextSize':'14px'}}}%%
pie showData title Project Hours Breakdown (Total 72h)
    "Completed Work" : 66
    "Remaining Work" : 6
```

**Remaining hours by priority** (all 6 remaining hours; no High/blocking work):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#A8FDD9','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px'}}}%%
pie showData title Remaining Hours by Priority
    "Medium (review + citations)" : 4
    "Low (reproduce + triage)" : 2
```

**Completed hours by category** (sums to 66):

| Category | Hours |
|----------|-------|
| Harness engineering | 14 |
| Document authoring | 12 |
| Q1–Q4 tracing | 8 |
| Q5 differential | 8 |
| Edge/error coverage | 7 |
| QA remediation | 7 |
| Citation verification | 5 |
| Environment stand-up | 4 |
| Web-search validation | 1 |
| **Total** | **66** |

---

## 8. Summary & Recommendations

**Achievements.** The project delivers a rigorous, runtime-first root-cause analysis of SimpleLogin's alias reply-handling flow. It answers all five sub-questions with observed evidence, drives every scenario through the real `MailHandler.handle_DATA` entry point (no mocks or bypass), and pinpoints a single most-likely origin of incorrect routing: the contact-resolution step `Contact.get_by(reply_email=…).first()` (`email_handler.py:L986`) over a non-unique `reply_email` column (`app/models.py:L1899`) resolved by `.first()` with no `ORDER BY` (`app/models.py:L84`). It additionally reproduces a dual-user-reference fragility and two concrete reply-body defects (`RUNTIME-01` cross-user disclosure, `RUNTIME-02` silent drop). The read-only mandate was honored to the byte and all temporary scaffolding was cleaned up.

**Verdict conveyed by the analysis.** In the default, correctly-owned configuration the reply flow routes correctly and deterministically, so an ordinary "wrong user" report is most likely a *misunderstanding* of the outward-relay reverse-alias model. However, the pipeline contains *real, reproducible latent fragilities* that genuinely mis-route the decision under specific-but-plausible data conditions — so the behavior *can* be a real bug. Both interpretations are partly correct; which applies depends on the data.

**Remaining gaps & critical path to production.** The project is **91.7% complete** (66 of 72 hours). There is no remaining autonomous engineering work; the 6 remaining hours are entirely human path-to-production: SME review/acceptance of the analysis, a citation spot-check, an optional independent reproduction, and a triage decision on whether to open remediation tickets. **Fixing the diagnosed defects is explicitly out of this task's scope** and is therefore not counted as remaining AAP work.

**Success metrics.** All five sub-questions answered ✅ · 11/11 scenarios + 104/104 assertions pass ✅ · project regression 23/23 ✅ · zero source files modified ✅ · working tree git-clean ✅ · 212 citations verified ✅.

**Production-readiness assessment.** The deliverable is **production-ready as a documentation artifact**: complete, accurate, well-formed, fully cited, and compliant with every AAP rule. It is ready for human acceptance and hand-off to maintainers for a fix-or-not decision.

---

## 9. Development Guide

This project's "application" is an analysis document plus a reproducible runtime harness. The steps below cover reviewing the deliverable, verifying the read-only mandate, and (optionally) reproducing the runtime trace.

### 9.1 System Prerequisites

- **For reviewing the deliverable (any machine):** Git and any Markdown/Mermaid viewer.
- **For reproducing the runtime trace (recommended: the canonical image):**
  - Python **3.10** (`pyproject.toml:L61` → `python = "^3.10"`)
  - PostgreSQL **13** (`.github/workflows/main.yml` → `image: postgres:13`)
  - Redis **6** (`.github/workflows/main.yml` → `redis-version: 6`)
  - **Poetry** for dependency management
  - Canonical platform: GHCR image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (ships the project at `2cd6ee77` with a prebuilt venv at `/app/venv`, PostgreSQL, and Redis)

> ⚠ **The default assessment container (Python 3.13, no PostgreSQL/Redis/Poetry/internet) cannot reproduce the runtime harness.** Use the canonical GHCR image, where the pinned dependencies (`SQLAlchemy 1.3.24`, `aiosmtpd 1.4.2`, etc.) resolve correctly on Python 3.10.

### 9.2 Retrieve the Deliverable & Verify Read-Only Scope

```bash
# From the repository root on branch blitzy-464d5006-57bc-4d45-97ca-b1e6ba56fd80
wc -l blitzy/documentation/app_2cd6ee777f8c.md          # -> 2139

# Prove ZERO source files changed since the base commit
git diff --name-only 2cd6ee77..HEAD -- email_handler.py 'app/**' 'tests/**' \
  pyproject.toml poetry.lock 'migrations/**' '.github/**' Dockerfile README.md 'docs/**'
# (expected: empty output)

# Prove the working tree is clean and only the deliverable was added
git status --porcelain                                   # (expected: empty)
git diff --name-status 2cd6ee77..HEAD                    # -> A  blitzy/documentation/app_2cd6ee777f8c.md
```

### 9.3 Review the Analysis

```bash
# List the verdict + Q1-Q5 answer anchors
grep -nE '^## \((a|b|c|d|e|f)\)' blitzy/documentation/app_2cd6ee777f8c.md

# Verify markdown well-formedness (fence count must be EVEN)
grep -c '^```' blitzy/documentation/app_2cd6ee777f8c.md  # -> 40 (balanced)
grep -c '^```mermaid' blitzy/documentation/app_2cd6ee777f8c.md  # -> 1

# Spot-check the root-cause citation (the resolution pivot)
sed -n '986p' email_handler.py    # -> contact = Contact.get_by(reply_email=reply_email)
sed -n '1046p' email_handler.py   # -> user_id=contact.user_id,
sed -n '1004p' email_handler.py   # -> user = alias.user
sed -n '84p'  app/models.py       # -> return Session.query(cls).filter_by(**kw).first()
sed -n '1899p' app/models.py      # -> reply_email = sa.Column(sa.String(512), nullable=False, index=True)
```

### 9.4 (Optional) Reproduce the Runtime Trace — Canonical Image

```bash
# 1) Inside the canonical GHCR image container, remap Postgres to the test port & start services
#    (Postgres cluster version may be 13/15 depending on the image; start it and Redis)
pg_ctlcluster 15 main start          # or the version shipped by the image
redis-server --daemonize yes

# 2) Build the schema via the canonical CI recipe (uses the image venv)
CONFIG=tests/test.env /app/venv/bin/alembic upgrade head   # -> head 32f25cbf12f6, 77 tables

# 3) Recreate the observation harness from the deliverable's raw-output section (§ h),
#    save it as tests/blitzy_reply_trace.py, then drive the REAL SMTP callback:
CONFIG=tests/test.env GITHUB_ACTIONS_TEST=true \
  /app/venv/bin/python -m pytest tests/blitzy_reply_trace.py -s
#   -> 11 passed, 104 ASSERT PASS, 0 FAIL
```

### 9.5 Regression (verify source is unchanged & green)

```bash
CONFIG=tests/test.env GITHUB_ACTIONS_TEST=true \
  /app/venv/bin/python -m pytest tests/test_email_handler.py --timeout=60
#   -> 23 passed, 18 warnings
```

### 9.6 Verification Checklist

- [ ] `git diff --name-status 2cd6ee77..HEAD` lists only `blitzy/documentation/app_2cd6ee777f8c.md`
- [ ] `git status --porcelain` is empty (clean tree)
- [ ] Deliverable is 2,139 lines with 40 balanced code fences and 1 Mermaid diagram
- [ ] Q1–Q5 anchors present; root-cause citations resolve to the quoted lines
- [ ] (Optional) Harness reproduces 11 passed / 104 assertions; regression 23/23

### 9.7 Troubleshooting

- **`ModuleNotFoundError` / dependency resolution fails on Python 3.13:** the pinned deps require Python 3.10 — use the canonical GHCR image (its `/app/venv` is prebuilt).
- **`could not connect to server` (Postgres) / Redis connection refused:** start `postgresql` (on port `15432` per `DB_URI` in `tests/test.env`) and `redis-server` before migrations/tests.
- **`dkim.key` load error / `re2` import error:** these are container-only provisioning quirks (image `dkim.key` PKCS#1; a `re2`→`re` shim). They apply to the image scratch copy only — **never** to repository source — and are documented in the deliverable §(k).
- **Absolute row IDs differ from the document:** expected — Postgres sequence IDs vary run-to-run. Assert **structural invariants** (e.g., `EmailLog.user_id == contact.user_id`), not absolute integers, exactly as the deliverable does.

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---------|---------|
| Size the deliverable | `wc -l blitzy/documentation/app_2cd6ee777f8c.md` |
| Read-only proof | `git diff --name-only 2cd6ee77..HEAD -- email_handler.py 'app/**' 'tests/**'` |
| Clean-tree proof | `git status --porcelain` |
| Changed-files list | `git diff --name-status 2cd6ee77..HEAD` |
| Commit history | `git log --oneline 2cd6ee77..HEAD` |
| Build schema | `CONFIG=tests/test.env poetry run alembic upgrade head` |
| Reproduce harness | `CONFIG=tests/test.env GITHUB_ACTIONS_TEST=true poetry run pytest tests/blitzy_reply_trace.py -s` |
| Regression suite | `CONFIG=tests/test.env GITHUB_ACTIONS_TEST=true poetry run pytest tests/test_email_handler.py` |

### B. Port Reference

| Service | Port | Source |
|---------|------|--------|
| PostgreSQL (test) | `15432` | `DB_URI=postgresql://test:test@localhost:15432/test` (`tests/test.env`) |
| Redis (mem store) | `6379` (default) | `MEM_STORE_URI=redis://localhost` (`tests/test.env`) |
| SimpleLogin SMTP listener (production ingress) | `20381` | `README.md:L367` (Postfix → `127.0.0.1:20381`) |

### C. Key File Locations

| File | Role |
|------|------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The deliverable** (analysis) |
| `email_handler.py` | Inbound SMTP processor — `handle_DATA` L2289, `_handle` L2335, `handle` L1945, `handle_reply` L966 |
| `app/email_utils.py` | `is_reverse_alias()` L1156 (reply-vs-forward classifier) |
| `app/models.py` | ORM: `get_by`=`.first()` L84, `Contact` L1863 (`reply_email` L1899, `uq_contact` L1875), `EmailLog` L2060 |
| `app/mail_sender.py` | `store_emails_instead_of_sending()` L102 / `get_stored_emails()` L108 (capture seam) |
| `app/alias_utils.py` | `transfer_alias()` L458–540 (keeps `contact.user_id` & `alias.user_id` in sync) |
| `tests/test_email_handler.py` | Canonical in-process harness pattern; regression suite |
| `tests/test.env` | Test configuration (`NOT_SEND_EMAIL`, `EMAIL_DOMAIN`, `DB_URI`) |
| `.github/workflows/main.yml` | Canonical build/run recipe (Postgres 13, Redis 6, alembic head) |

### D. Technology Versions

| Component | Version | Source |
|-----------|---------|--------|
| Python | 3.10 (`^3.10`) | `pyproject.toml:L61` |
| PostgreSQL | 13 | `.github/workflows/main.yml` |
| Redis | 6 | `.github/workflows/main.yml` |
| SQLAlchemy | 1.3.24 | `poetry.lock` |
| aiosmtpd | 1.4.2 | `poetry.lock` |
| Flask | 1.1.2 | `poetry.lock` |
| alembic | 1.4.3 | `poetry.lock` (migration head `32f25cbf12f6`) |
| pytest | 7.3.1 | `poetry.lock` |
| Dependency manager | Poetry | `pyproject.toml` |

### E. Environment Variable Reference

| Variable | Value (test) | Purpose |
|----------|--------------|---------|
| `CONFIG` | `tests/test.env` | Points the app/migrations at test configuration |
| `NOT_SEND_EMAIL` | `true` | Captures outbound mail instead of sending (no real egress) |
| `EMAIL_DOMAIN` | `sl.local` | Alias/reverse-alias domain |
| `OTHER_ALIAS_DOMAINS` | `["d1.test","d2.test","sl.local"]` | Additional managed alias domains |
| `DB_URI` | `postgresql://test:test@localhost:15432/test` | Test database |
| `MEM_STORE_URI` | `redis://localhost` | Redis (rate limiting / locks) |
| `DMARC_CHECK_ENABLED` | `true` | DMARC gate (non-blocking in reply phase without spamd headers) |
| `GITHUB_ACTIONS_TEST` | `true` | Test-mode flag used by the suite |

### F. Developer Tools Guide

- **Git diff/log** — verify read-only scope and authorship (`git log --author="agent@blitzy.com" 2cd6ee77..HEAD --oneline` → 5 commits).
- **`sed -n '<line>p' <file>`** — resolve any `file:line` citation from the deliverable against source.
- **`grep -c '^\`\`\`'`** — confirm Markdown code-fence balance (even = balanced).
- **pytest `-s`** — run the harness/regression with stdout visible for the runtime transcript.
- **`mail_sender.store_emails_test_decorator`** — the production capture seam used to observe outbound `SendRequest`s without sending mail.

### G. Glossary

| Term | Meaning |
|------|---------|
| **Reverse alias** | The per-`(alias, contact)` `reply_email` address a user replies to; SimpleLogin relays outward to the contact while hiding the real mailbox |
| **`Contact`** | ORM row linking an alias to an external `website_email` and its `reply_email`; the resolution pivot for replies |
| **`EmailLog`** | Persisted record of a handled message; `is_reply=True` for replies; stores `user_id=contact.user_id` |
| **Dual user reference** | The reply handler's coexisting `alias.user` (authorization) and `contact.user_id` (persistence) references |
| **`handle_DATA`** | The aiosmtpd SMTP `DATA` callback — the canonical inbound entry point |
| **Observed vs. inferred** | Deliverable labeling: *observed* = from captured runtime output or a quoted source line; *inferred* = a deduction from observed facts |
| **E214 / E501–E504 / E206** | SimpleLogin SMTP status codes returned by `handle_reply` guard paths |
| **AAP** | Agent Action Plan — the governing project specification |

---

*Prepared by the Blitzy autonomous assessment agent. Completion (91.7%) is measured strictly against AAP-scoped hours (66 completed / 72 total). All test results originate from Blitzy's autonomous validation logs. Completed = Dark Blue `#5B39F3`; Remaining = White `#FFFFFF`.*