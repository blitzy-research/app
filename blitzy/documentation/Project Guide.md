# Blitzy Project Guide — SimpleLogin Runtime-Behavior Analysis

> Branch `app_2cd6ee777f8c` @ commit `2cd6ee77` · Documentation deliverable · SWE-AtlasQnA-Repo

---

## 1. Executive Summary

### 1.1 Project Overview

This project authors a single, comprehensive technical-analysis document that answers three runtime-behavior questions about the open-source **SimpleLogin** email-alias platform, grounded strictly in source code ("code-as-truth") with precise `file:line` citations and explicit reasoning. The target audience is engineers and reviewers who need authoritative, citation-backed answers about (Q1) mailbox-verification retry/lockout, (Q2) the background `Job` lifecycle and error/retry behavior, and (Q3) the VERP bounce-address format and forward-vs-reply handling. The technical scope spans three subsystems — web/dashboard, the job runner, and inbound email/bounce processing — investigated read-only. The business impact is reduced onboarding time and de-risked maintenance through accurate, verifiable system documentation. The SimpleLogin source tree is treated as strictly read-only; the only artifact produced is the answer document.

### 1.2 Completion Status

The project is **91.3% complete** based on AAP-scoped engineering hours. All deliverable work is finished and validated; the remaining 2 hours are human-only path-to-production gates (subject-matter-expert sign-off and PR merge).

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieTitleTextSize':'17px','pieLegendTextColor':'#B23AF2'}}}%%
pie showData title Completion Status — 91.3% Complete
    "Completed (AI)" : 21
    "Remaining" : 2
```

| Metric | Hours |
|---|---|
| **Total Hours** | 23.0 |
| **Completed Hours (AI + Manual)** | 21.0 |
| &nbsp;&nbsp;• AI / Autonomous | 21.0 |
| &nbsp;&nbsp;• Manual (human, to date) | 0.0 |
| **Remaining Hours** | 2.0 |
| **Percent Complete** | **91.3%** |

> Formula: Completion % = Completed ÷ Total = 21.0 ÷ 23.0 = **91.3%**

### 1.3 Key Accomplishments

- ✅ Authored the single required deliverable at the exact path/name `blitzy/documentation/app_2cd6ee777f8c.md` (filename = source branch name).
- ✅ Answered **Q1 (mailbox verification lockout)** with the `tries` counter, `MAX_ACTIVATION_TRIES = 3`, the lockout-deletes-row mechanism, and the absence of an HTTP rate-limiter — all citation-backed.
- ✅ Answered **Q2 (background Job lifecycle)** with full creation → `taken` → `done` tracing, at-least-once semantics, the no-`try/except` fail-fast posture, and the 30-min / 5-attempt retry bounds.
- ✅ Answered **Q3 (VERP bounce address)** with the exact format `sl.<base32-payload>.<base32-hmac>@<domain>`, O(1) `EmailLog.id` identification, and a complete forward-vs-reply comparison table.
- ✅ Provided an explicit **Rationale / Thinking** block for every question (3 of 3).
- ✅ Documented **5 source quirks** (observed, not fixed) for transparency.
- ✅ **158 file:line citations across 15 files verified** with zero errors; **runtime-corroborated** in the sanctioned Docker image.
- ✅ **Zero source files modified** — strict read-only-source compliance confirmed by `git diff`.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|---|---|---|---|
| _None_ — no blocking issues identified | No compilation errors, no failing tests, no missing functionality; deliverable is complete, accurate, well-formed, and committed | — | — |

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|---|---|---|---|---|
| _None_ | — | No access issues identified. The repository was fully accessible; the optional runtime-corroboration Docker image (`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`) was reachable and used ephemerally by the validator | N/A | — |

**No access issues identified.**

### 1.6 Recommended Next Steps

1. **[Medium]** Have a subject-matter expert review the Q3 VERP forward-vs-reply analysis (the most complex section) for technical correctness against their domain understanding — ~1.0h.
2. **[Medium]** Have a subject-matter expert review the Q1 (mailbox lockout) and Q2 (Job lifecycle) answers — ~0.5h.
3. **[Low]** Review and merge the PR adding `blitzy/documentation/app_2cd6ee777f8c.md` to the target branch — ~0.5h.
4. **[Low]** _(Optional)_ Re-run the dependency-free static recheck commands (Section 9) to independently reconfirm citations — 0h.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

Every completed component traces to a specific AAP requirement (the deliverable document and its sections) or to mandated code-as-truth fidelity/runtime corroboration.

| Component | Hours | Description |
|---|---|---|
| Q1 — Mailbox verification lockout analysis & write-up | 3.5 | Traced `verify_mailbox_code()`, `MAX_ACTIVATION_TRIES`, `MailboxActivation` model, `/mailbox_verify` route; authored citation-backed answer + rationale |
| Q2 — Background Job lifecycle analysis & write-up | 3.5 | Traced runner loop, `get_jobs_to_run()`, `process_job()`, `JobState`, retry constants, cleanup; authored answer + rationale |
| Q3 — VERP bounce address & forward-vs-reply analysis & write-up | 6.0 | Traced `generate_verp_email()`/`get_verp_info_from_email()` (HMAC/base32), generation sites, bounce handlers across an 83 KB module; built comparison table + auto-reply special case + rationale |
| Supporting sections (Environment & Methodology, Documented Quirks, Reference Files) | 2.0 | Methodology note, 5 documented quirks, read-only reference-file inventory |
| VERP convention external research | 0.5 | Validated VERP terminology to frame the identifier-based variant |
| Runtime corroboration (ephemeral Docker) | 2.5 | Booted sanctioned GHCR image (Python 3.10, PostgreSQL + Redis); verified all documented constants; `generate_verp_email` round-trip + tamper-rejection |
| Citation verification + consistency fixes + iteration | 3.0 | Hand-verified 158 citations across 15 files; corrected Q3 VERP lifetime semantics; reconciled reference-file count |
| **Completed Total** | **21.0** | |

> Section 2.1 total (21.0h) equals Completed Hours in Section 1.2. ✔

### 2.2 Remaining Work Detail

Each remaining category is human-only path-to-production work; none represents remediation of a deliverable defect.

| Category | Hours | Priority |
|---|---|---|
| Human SME Review & Sign-off (Q1/Q2/Q3 technical correctness) | 1.5 | Medium |
| PR Review & Merge (to target branch) | 0.5 | Low |
| **Remaining Total** | **2.0** | |

> Section 2.2 total (2.0h) equals Remaining Hours in Section 1.2 and "Remaining Work" in Section 7. ✔
> Section 2.1 (21.0h) + Section 2.2 (2.0h) = **23.0h** = Total Project Hours in Section 1.2. ✔

### 2.3 Confidence Levels

| Item | Confidence | Notes |
|---|---|---|
| Completed hours (21.0h) | High | Work is committed, verified, and runtime-corroborated; estimate reflects forensic-analysis effort across three subsystems |
| Remaining hours (2.0h) | High | Well-defined human review + merge; scope is bounded and understood |
| Completion % (91.3%) | High | Hours-based, derived solely from AAP-scoped work + path-to-production |

---

## 3. Test Results

This is a **documentation-only task with a strictly read-only source tree**, so no software unit/integration tests were authored or executed against SimpleLogin (the source's own ~100 test files were neither run nor modified as part of this task). Instead, Blitzy's autonomous validation systems performed **documentation-validation checks** — citation verification, runtime constant corroboration, and structural well-formedness. **Every check below originates from Blitzy's autonomous validation logs for this project.**

| Test Category | Framework / Method | Total | Passed | Failed | Coverage % | Notes |
|---|---|---|---|---|---|---|
| Citation Verification | Manual `file:line` audit (grep/sed) | 158 | 158 | 0 | 100% of cited refs | All citations across 15 files parsed; zero out-of-bounds, zero missing files |
| Runtime Constant Corroboration | Python REPL in GHCR Docker (py3.10) | 9 | 9 | 0 | 100% of documented constants | `MAX_ACTIVATION_TRIES`=3; `JobState`=(0,1,2,3); `VerpType`=(0,1,2); `JOB_MAX_ATTEMPTS`=5; `JOB_TAKEN_RETRY_WAIT_MINS`=30; `VERP_PREFIX`="sl"; `VERP_MESSAGE_LIFETIME`=432000; `VERP_TIME_START`=1640995200; `VERP_HMAC_ALGO`="sha3-224" |
| VERP Round-trip / Tamper | Python (`generate_verp_email` / `get_verp_info_from_email`) | 3 | 3 | 0 | gen + decode + tamper | Exact `sl.<b32>.<b32>@domain` (lowercased); round-trip → `(bounce_forward, 12345)`; tampered address rejected |
| Markdown Well-Formedness | Structural lint (grep) | 4 | 4 | 0 | structure | 17 headings; balanced code-fence pairs; 0 trailing-whitespace lines; 0 tab characters |
| Internal Consistency | Manual review | 1 | 1 | 0 | reference-count | "ten files" wording reconciled with cited files (commit `a37bda6d`) |
| **Total** | | **175** | **175** | **0** | **100%** | Zero failures across all autonomous validation checks |

> **Integrity note:** All results above are sourced from Blitzy's autonomous validation logs (the 5 production-readiness gates). An independent re-verification during this assessment spot-checked ~12 citations across Q1/Q2/Q3 and confirmed every one matches source exactly.

---

## 4. Runtime Validation & UI Verification

There is **no user interface** in this deliverable (it is a markdown document), so UI verification is not applicable. Runtime validation consisted of corroborating documented behavior by executing the actual SimpleLogin code in the sanctioned GHCR Docker image.

- ✅ **Operational** — Image boot: GHCR image (commit `2cd6ee77`, Python 3.10 venv) started with in-image PostgreSQL + Redis.
- ✅ **Operational** — Constant corroboration: all 9 documented constants imported and matched their documented values exactly.
- ✅ **Operational** — `generate_verp_email()` produced the exact documented format `sl.<base32>.<base32>@domain` (lowercased).
- ✅ **Operational** — VERP round-trip: `get_verp_info_from_email()` decoded the generated address back to `(bounce_forward, 12345)`.
- ✅ **Operational** — Tamper rejection: a modified VERP address failed the HMAC check and was rejected (returns `None`), confirming tamper-evidence.
- ✅ **Operational** — Markdown render: document is well-formed (17 headings; balanced code fences; resolving internal anchors).
- ✅ **Operational** — Ephemerality: container removed after corroboration; no host mounts; **zero residual artifacts**; working tree clean.
- ⚠ **Partial** — N/A. No partial or degraded behaviors were observed.
- ❌ **Failing** — None.

---

## 5. Compliance & Quality Review

The applicable compliance benchmarks are the **SWE-AtlasQnA-Repo** rules plus the AAP constraints. Each is cross-mapped to its outcome below.

| Benchmark / AAP Requirement | Status | Progress | Evidence |
|---|---|---|---|
| Deliverable named `<source_branch>.md` (`app_2cd6ee777f8c.md`) | ✅ Pass | 100% | File present at exact name |
| Placed in `blitzy/documentation/` (destination repo) | ✅ Pass | 100% | `blitzy/documentation/app_2cd6ee777f8c.md` |
| Comprehensively answers Q1, Q2, Q3 | ✅ Pass | 100% | One section per question, all sub-parts covered |
| Code-as-truth with `file:line` citations | ✅ Pass | 100% | 158 citations verified; 0 errors |
| Explicit thinking / rationale provided | ✅ Pass | 100% | 3 "Rationale / Thinking" blocks (one per question) |
| No existing source file modified | ✅ Pass | 100% | `git diff` = 1 file added, 0 modified |
| No extra code added to source repo | ✅ Pass | 100% | Only the `.md` deliverable |
| Runtime inspection ephemeral (no residual artifacts) | ✅ Pass | 100% | Container removed; no host mounts; tree clean |
| Observed quirks documented, NOT fixed | ✅ Pass | 100% | 5 quirks labeled "observed, NOT fixed" |
| Line citations pinned to HEAD `2cd6ee77` | ✅ Pass | 100% | Stated in document header |
| Markdown well-formed | ✅ Pass | 100% | 17 headings; balanced fences; no tabs/trailing-ws |

**Fixes applied during autonomous validation:**
- Corrected Q3 VERP lifetime semantics (future-bound, not a staleness/expiry check) and added methodology citations + a code-fence language tag — commit `fd8ad04a`.
- Reconciled the "Reference Files" count with the actual set of cited files — commit `a37bda6d`.

**Outstanding compliance items:** None.

---

## 6. Risk Assessment

Overall risk profile is **Low** — the deliverable introduces no executable code, no source changes, no credentials, and no dependencies.

| Risk | Category | Severity | Probability | Mitigation | Status |
|---|---|---|---|---|---|
| Line citations could drift if SimpleLogin source advances past `2cd6ee77` | Technical | Low | Medium | Every citation is explicitly pinned to branch `app_2cd6ee777f8c` @ commit `2cd6ee77` in the document header | Mitigated |
| Documented quirks misread as action items | Technical | Low | Low | Quirks are clearly labeled "observed, NOT fixed" and noted as out-of-scope per AAP | Mitigated |
| Security findings in doc (e.g., unguarded `/mailbox_verify`) mistaken for new vulnerabilities introduced here | Security | Informational | Low | Document accurately frames these as existing SimpleLogin observations, not deliverable changes; zero code added | Mitigated |
| Optional Docker runtime image unavailable to a future reviewer | Integration | Low | Low | Dependency-free static recheck (grep/sed) fully suffices and is documented in Section 9 | Mitigated |
| Deliverable validated but not yet SME-signed-off | Operational / Process | Low | Low | All citations independently verified + runtime-corroborated, minimizing the chance of SME-found errors | Open (human gate) |

> **Note:** No operational deployment risks apply — there is no production runtime footprint. Runtime corroboration was fully ephemeral and left no artifacts.

---

## 7. Visual Project Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieTitleTextSize':'16px','pieLegendTextColor':'#B23AF2'}}}%%
pie showData title Project Hours Breakdown (Total 23h)
    "Completed Work" : 21
    "Remaining Work" : 2
```

**Remaining hours by category (Section 2.2):**

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#A8FDD9','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieTitleTextSize':'15px','pieLegendTextColor':'#B23AF2'}}}%%
pie showData title Remaining Work by Category (2h)
    "Human SME Review & Sign-off" : 1.5
    "PR Review & Merge" : 0.5
```

**Priority distribution of remaining work:**

| Priority | Hours | Share |
|---|---|---|
| High | 0.0 | 0% |
| Medium | 1.5 | 75% |
| Low | 0.5 | 25% |
| **Total** | **2.0** | **100%** |

> **Integrity:** "Remaining Work" = 2h here equals Section 1.2 Remaining Hours (2h) and the sum of the Section 2.2 Hours column (1.5 + 0.5 = 2h). "Completed Work" = 21h equals Section 1.2 Completed Hours. Brand colors: Completed = Dark Blue `#5B39F3`, Remaining = White `#FFFFFF`.

---

## 8. Summary & Recommendations

**Achievements.** The project delivers a complete, accurate, citation-backed runtime-behavior analysis of SimpleLogin answering all three posed questions. Every claim is traceable to source, **158 citations across 15 files were verified with zero errors**, and the behavior was **runtime-corroborated** in the sanctioned Docker image. The deliverable was produced with strict scope discipline: **exactly one file added, zero source files modified.**

**Remaining gaps.** The project is **91.3% complete (21h of 23h)**. The only remaining work is human-only path-to-production: subject-matter-expert technical sign-off (1.5h) and PR review & merge (0.5h). There are **no blocking issues, no compilation errors, no failing tests, and no missing functionality**.

**Critical path to production.** SME review of Q3 (most complex) → SME review of Q1/Q2 → PR merge. Estimated total elapsed effort: **2 hours**.

**Success metrics.**

| Metric | Result |
|---|---|
| AAP requirements completed | 12 of 14 (the other 2 are human-only path-to-production) |
| Citations verified | 158 / 158 (100%) |
| Source files modified | 0 (as required) |
| Autonomous validation checks passed | 175 / 175 (100%) |
| Documentation well-formedness | Pass |

**Production-readiness assessment.** The deliverable is **ready for human review and merge**. Confidence is High: the analysis is internally consistent, well-formed, runtime-corroborated, and independently re-verified. At approximately **91% complete**, the autonomous work is finished; only human sign-off remains. Consistent with honest-assessment practice, completion is reported below 100% because final SME sign-off is a genuine human gate that has not yet occurred.

---

## 9. Development Guide

This is a documentation deliverable, so the "development" workflow is **viewing and re-verifying** the analysis. Every command below was tested in the repository and is copy-pasteable. All commands assume you start at the repository root.

### 9.1 System Prerequisites

- **Git** ≥ 2.30 (tested with 2.51.0) — to inspect the change set.
- **A POSIX shell** with `grep`, `sed`, `cat`, `wc` — for static rechecks (no project dependencies needed).
- **(Optional) Docker** ≥ 20 (tested with 28.5.2) — only if you wish to reproduce the runtime corroboration.
- **(Optional, for runtime recheck only)** the source targets **Python 3.10** (per `pyproject.toml`, `Dockerfile`, and the CI workflow); the sanctioned Docker image already carries a 3.10 venv plus PostgreSQL + Redis.

### 9.2 How to View the Deliverable

```bash
# From the repository root
ls -la blitzy/documentation/app_2cd6ee777f8c.md
cat blitzy/documentation/app_2cd6ee777f8c.md
```

Expected: a ~36 KB / 282-line markdown file with one section per question (Q1, Q2, Q3), an Environment & Methodology note, a Documented Quirks section, and a Reference Files section.

### 9.3 Confirm Scope (read-only-source compliance)

```bash
# Should print exactly ONE added file and nothing else
git diff --name-status origin/app_2cd6ee777f8c...HEAD
```

Expected output:

```text
A	blitzy/documentation/app_2cd6ee777f8c.md
```

### 9.4 Static Citation Recheck (no dependencies)

```bash
# Q1 — mailbox verification lockout
grep -n "MAX_ACTIVATION_TRIES = 3" app/mailbox_utils.py          # -> 43:MAX_ACTIVATION_TRIES = 3
sed -n '195,211p' app/mailbox_utils.py                            # lockout -> expiry -> wrong-code increment

# Q2 — background Job lifecycle
sed -n '337,347p' job_runner.py                                   # taken/attempts++ /commit -> process_job -> done
grep -n "JOB_MAX_ATTEMPTS\|JOB_TAKEN_RETRY_WAIT_MINS" app/config.py  # -> 564:=5, 565:=30
sed -n '253,257p' app/models.py                                   # JobState ready/taken/done/error = 0/1/2/3

# Q3 — VERP bounce address
grep -n 'VERP_PREFIX\|VERP_MESSAGE_LIFETIME' app/config.py        # -> 499:5*86400, 500:"sl"
sed -n '1438,1464p' app/email_utils.py                            # generate_verp_email()
sed -n '247,250p' app/models.py                                   # VerpType bounce_forward/bounce_reply/transactional
sed -n '19,20p;47p;49p' app/email/status.py                       # E211/E212/E510(typo)/E512
```

### 9.5 Markdown Well-Formedness Check

```bash
DOC="blitzy/documentation/app_2cd6ee777f8c.md"
grep -cE '^#{1,4} ' "$DOC"      # headings  -> 17
grep -c '```' "$DOC"            # code fences (even = balanced) -> 6
grep -cE ' +$' "$DOC"           # trailing-whitespace lines -> 0
grep -cP '\t' "$DOC"            # tab characters -> 0
# Citation count (single-quote the pattern so backticks are literal):
grep -oE '`[A-Za-z_/.]+\.(py|toml|yml):[0-9]+(-[0-9]+)?`' "$DOC" | wc -l   # -> 156
```

### 9.6 (Optional) Ephemeral Docker Runtime Recheck

> Optional — the static recheck above is sufficient. This reproduces the runtime corroboration and **leaves no artifacts** (the container is removed at the end).

```bash
docker run -d --name sl_corrob --entrypoint /bin/bash \
  ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0 -lc 'sleep infinity'
docker exec sl_corrob bash -lc 'pg_ctlcluster 15 main start; redis-server --daemonize yes'
docker exec sl_corrob bash -lc 'cd /app && /app/venv/bin/python -c \
  "from app import config, mailbox_utils; from app.models import JobState, VerpType; \
   print(mailbox_utils.MAX_ACTIVATION_TRIES, config.JOB_MAX_ATTEMPTS, config.VERP_PREFIX)"'
# Expected: 3 5 sl
docker rm -f sl_corrob
```

### 9.7 Verification Steps

1. The deliverable exists and renders as well-formed markdown (§9.2, §9.5).
2. The change set is exactly one added file (§9.3).
3. Each Q1/Q2/Q3 constant and code block matches source (§9.4).
4. _(Optional)_ Runtime constants match (§9.6) → prints `3 5 sl`.

### 9.8 Troubleshooting

| Symptom | Likely Cause | Resolution |
|---|---|---|
| `git diff ... origin/app_2cd6ee777f8c` errors | Base ref not fetched locally | `git fetch origin app_2cd6ee777f8c`, then retry |
| Static recheck line numbers differ | Source moved past `2cd6ee77` | Check out commit `2cd6ee77`; citations are pinned to that revision |
| Citation-count command prints `0` | Backticks interpreted by the shell | Single-quote the `grep -oE '...'` pattern (as shown in §9.5) |
| Docker recheck fails to start DB | Cluster name/version differs in image | `docker exec sl_corrob bash -lc 'pg_lsclusters'` and start the listed cluster |
| `docker: command not found` | Docker not installed | Skip §9.6 — the static recheck (§9.4) fully verifies all claims without dependencies |

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---|---|
| View deliverable | `cat blitzy/documentation/app_2cd6ee777f8c.md` |
| Confirm scope | `git diff --name-status origin/app_2cd6ee777f8c...HEAD` |
| Confirm authorship | `git log --author="agent@blitzy.com" --oneline` |
| Q1 recheck | `grep -n "MAX_ACTIVATION_TRIES = 3" app/mailbox_utils.py` |
| Q2 recheck | `sed -n '337,347p' job_runner.py` |
| Q3 recheck | `sed -n '1438,1464p' app/email_utils.py` |
| Well-formedness | `grep -cE '^#{1,4} ' blitzy/documentation/app_2cd6ee777f8c.md` |

### B. Port Reference

| Service | Port | Applicability |
|---|---|---|
| PostgreSQL | 5432 | Only inside the optional runtime-recheck Docker image (§9.6) |
| Redis | 6379 | Only inside the optional runtime-recheck Docker image (§9.6) |

> The deliverable itself exposes no ports; ports apply solely to the optional Docker corroboration.

### C. Key File Locations

| File | Role |
|---|---|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The deliverable** (only file added) |
| `app/mailbox_utils.py` | Q1: `MAX_ACTIVATION_TRIES`, `verify_mailbox_code()` |
| `app/models.py` | Q1 `MailboxActivation`; Q2 `Job`/`JobState`; Q3 `EmailLog`/`Bounce`/`RefusedEmail`/`VerpType` |
| `app/dashboard/views/mailbox.py` | Q1: `/mailbox_verify` route (no rate-limiter) |
| `job_runner.py` | Q2: runner loop, `get_jobs_to_run()`, `process_job()` |
| `app/config.py` | Q2 `JOB_MAX_ATTEMPTS`/`JOB_TAKEN_RETRY_WAIT_MINS`; Q3 `VERP_PREFIX`/`VERP_MESSAGE_LIFETIME` |
| `tasks/cleanup_old_jobs.py` | Q2: terminal-state cleanup; `JobState.error` reference |
| `app/email_utils.py` | Q3: `generate_verp_email()`/`get_verp_info_from_email()`, `should_disable()` |
| `email_handler.py` | Q3: VERP generation sites, inbound routing, bounce handlers |
| `app/email/status.py` | Q3: E200/E211/E212/E510/E512 SMTP status strings |
| `app/s3.py` | Q3: refused-email artifact storage (boto3 + local fallback) |

### D. Technology Versions

| Technology | Version | Source |
|---|---|---|
| Python (source target) | ^3.10 | `pyproject.toml`, `Dockerfile`, CI workflow |
| SQLAlchemy | 1.3.24 | `poetry.lock` |
| Flask | ^1.1.2 | `pyproject.toml` |
| Flask-Limiter | ^1.4 | `pyproject.toml` (relevant to Q1's missing limiter) |
| aiosmtpd | ^1.2 | `pyproject.toml` (inbound SMTP for Q3) |
| redis | ^4.5.3 | `pyproject.toml` |
| VERP HMAC algorithm | sha3-224 | `app/email_utils.py:69` |

### E. Environment Variable Reference

> These variables govern the analyzed behaviors; none are set or changed by this task.

| Variable | Relevance |
|---|---|
| `VERP_PREFIX` | Q3: address prefix (default `"sl"`) — `app/config.py:500` |
| `VERP_EMAIL_SECRET` | Q3: HMAC signing key for VERP tokens |
| `EMAIL_DOMAIN` | Q3: default domain when `sender_domain` is absent |
| `MAILBOX_VERIFICATION_OVERRIDE_CODE` | Q1: optional fixed activation code override — `app/mailbox_utils.py:228-229` |
| `ALIAS_AUTOMATIC_DISABLE` | Q3: gates forward-phase alias auto-disable in `should_disable()` |
| `LOCAL_FILE_UPLOAD` | Q3: switches S3 vs local-disk artifact storage — `app/s3.py` |
| `CONFIG` (e.g. `tests/test.env`) | Runtime recheck only: selects the config file inside the Docker image |

### F. Developer Tools Guide

| Tool | Use in this project |
|---|---|
| `git` | Inspect the change set and confirm zero source modifications |
| `grep` / `sed` | Dependency-free static recheck of every cited `file:line` |
| `cat` / `wc` | View the deliverable and confirm size/structure |
| Docker | _(Optional)_ Reproduce ephemeral runtime corroboration via the sanctioned GHCR image |
| A Markdown viewer | Render the deliverable for review (GitHub renders it natively) |

### G. Glossary

| Term | Definition |
|---|---|
| **VERP** | Variable Envelope Return Path — encodes per-message info in the envelope sender so bounces map to the original message. SimpleLogin uses a signed, opaque identifier (`EmailLog.id`) variant |
| **HMAC** | Hash-based Message Authentication Code; here SHA3-224 truncated to 8 bytes, making the VERP token tamper-evident |
| **MailboxActivation** | Row holding a mailbox's current verification `code` and `tries` counter (Q1) |
| **`tries`** | Counter of failed verification attempts; lockout at `MAX_ACTIVATION_TRIES = 3` (Q1) |
| **Job / JobState** | A background task row and its state enum: `ready`=0, `taken`=1, `done`=2, `error`=3 (Q2) |
| **At-least-once** | Execution guarantee where `attempts` is incremented & committed before dispatch, bounding retries (Q2) |
| **EmailLog** | Per-message log row; its `id` is embedded in bounce VERP addresses for O(1) identification (Q3) |
| **Bounce / RefusedEmail** | Records persisted on a delivery failure; the original message + report are stored in S3 (Q3) |
| **Forward vs Reply phase** | Forward = inbound mail to alias undeliverable to mailbox; Reply = user's reply from alias undeliverable to contact (Q3) |
| **Code-as-truth** | Methodology requiring every claim to trace to a specific source `file:line` |

---

*Generated by the Blitzy Platform · Completion measured against AAP-scoped work + path-to-production · Brand colors: Completed `#5B39F3`, Remaining `#FFFFFF`.*