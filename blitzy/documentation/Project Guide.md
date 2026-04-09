# Blitzy Project Guide

---

## 1. Executive Summary

### 1.1 Project Overview

This project creates a comprehensive investigative runtime-behavior analysis document for the SimpleLogin open-source email aliasing platform. The deliverable (`blitzy/documentation/app_2cd6ee777f8c.md`) traces three internal workflows — mailbox verification code enforcement, background task lifecycle, and email forwarding bounce (VERP) handling — through direct source code analysis. The document answers specific behavioral questions about enforcement limits, retry semantics, and direction-dependent bounce processing, serving as a technical reference for developers working on or integrating with these subsystems. No source code modifications were made; the output is a standalone 740-line Markdown document with embedded Mermaid diagrams and 26 source file citations.

### 1.2 Completion Status

| Metric | Value |
|--------|-------|
| **Total Project Hours** | 34 |
| **Completed Hours (AI)** | 30 |
| **Remaining Hours** | 4 |
| **Completion Percentage** | 88.2% |

**Calculation:** 30 completed hours / (30 + 4) total hours = 88.2% complete.

```mermaid
pie title Project Completion Status
    "Completed (30h)" : 30
    "Remaining (4h)" : 4
```

### 1.3 Key Accomplishments

- ✅ Created `blitzy/documentation/app_2cd6ee777f8c.md` — 740 lines, 46KB comprehensive runtime behavior analysis
- ✅ Workflow 1 fully documented: Mailbox verification code enforcement with `MAX_ACTIVATION_TRIES=3`, 15-minute expiry, destructive enforcement, state machine diagram, and test suite cross-validation
- ✅ Workflow 2 fully documented: Background task lifecycle with `Job.create()` → `get_jobs_to_run()` → `process_job()` → `done` pipeline, implicit stale-lock retry pattern, and `JOB_MAX_ATTEMPTS=5` terminal failure documentation
- ✅ Workflow 3 fully documented: VERP address format specification (`sl.{b32_payload}.{b32_signature}@{domain}`), HMAC-SHA3-224 signing, legacy dual-format compatibility, forward vs. reply divergent handling, and 4-tier auto-disable threshold rules
- ✅ 3 Mermaid diagrams embedded (2 stateDiagram-v2, 1 flowchart)
- ✅ 26 source file citations with specific line numbers verified against actual codebase
- ✅ 8 rationale blocks explaining key design decisions (destructive enforcement, stale-lock pattern, asymmetric bounce handling)
- ✅ Zero source files modified — read-only constraint honored
- ✅ Clean working tree with no temporary artifacts

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| Source line number references may become stale if upstream code changes | Medium — citations could point to wrong lines | Human Developer | Before merge if upstream has changed |
| Document not linked from main repository documentation index | Low — document is discoverable in `blitzy/documentation/` but not referenced from `README.md` or `docs/` | Human Developer | 1h post-merge |

### 1.5 Access Issues

No access issues identified. The project is a documentation-only deliverable that does not require database access, API keys, service credentials, or third-party integrations. All source files are read-only inputs from the existing repository.

### 1.6 Recommended Next Steps

1. **[High]** Conduct peer review of technical accuracy — verify behavioral claims against source code, especially the three Mermaid diagrams and the auto-disable threshold table
2. **[High]** Verify all 26 line number citations against the current state of the source branch to confirm no upstream drift
3. **[Medium]** Review editorial quality — check clarity of rationale sections and consistency of terminology
4. **[Low]** Consider adding a cross-reference link from `docs/code-structure.md` (currently a stub) or `README.md` to the new document for discoverability

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|-------|-------------|
| Source code analysis — Workflow 1 (Mailbox Verification) | 4 | Deep reading of `app/mailbox_utils.py`, `app/models.py` (MailboxActivation), `app/dashboard/views/mailbox.py`, and `tests/test_mailbox_utils.py` to trace verification enforcement logic |
| Source code analysis — Workflow 2 (Job Lifecycle) | 4 | Deep reading of `job_runner.py`, `app/models.py` (Job, JobState), `app/config.py` (job constants), and `cron.py` to trace background task state machine |
| Source code analysis — Workflow 3 (VERP/Bounce Handling) | 6 | Deep reading of `email_handler.py` (bounce handlers, VERP routing), `app/email_utils.py` (VERP generation/parsing, should_disable), `app/models.py` (VerpType, EmailLog, Bounce, RefusedEmail), and `app/config.py` (bounce constants) |
| Document authoring — Introduction & structure | 0.5 | Scope statement, methodology, document skeleton with hierarchical headings |
| Document authoring — Workflow 1 sections | 3 | ~160 lines covering activation code generation, verify_mailbox_code() trace, enforcement limits, terminal states, and test validation |
| Document authoring — Workflow 2 sections | 4 | ~220 lines covering job creation, eligibility query, processing loop, error recovery, observable state table, and dispatcher |
| Document authoring — Workflow 3 sections | 5 | ~320 lines covering VERP format, parsing, legacy formats, bounce routing, forward/reply handling, comparison table, and threshold rules |
| Mermaid diagrams | 1.5 | 2 stateDiagram-v2 (verification state machine, job lifecycle) and 1 flowchart (bounce handling decision tree) |
| Summary tables & data compilation | 1 | Enforcement limits table, JobState table, job names table, VerpType table, forward-vs-reply comparison table, auto-disable threshold table, observable state table |
| Test case cross-referencing | 0.5 | Validated behavioral claims against `tests/test_mailbox_utils.py` (4 test cases) and `tests/test_email_utils.py` |
| Code review fixes | 0.5 | Addressed 3 code review findings (5 insertions, 5 deletions in second commit) |
| Source references section | 0.5 | Compiled complete inventory of 10 analyzed source files with line ranges |
| **Total Completed** | **30** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|-------|----------|
| Peer review of technical accuracy | 2 | High |
| Line number citation verification against current source | 1 | High |
| Editorial and formatting review | 1 | Medium |
| **Total Remaining** | **4** | |

---

## 3. Test Results

This is a documentation-only project — no application code was created or modified, so no unit, integration, or UI tests were executed as part of the Blitzy autonomous validation pipeline.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|------------|-------|
| Documentation Validation | Manual (Blitzy Agent) | 4 | 4 | 0 | 100% | Four production-readiness gates verified by Final Validator |

**Production-Readiness Gates Verified:**
1. **GATE 1 — Documentation completeness:** All 3 workflows answered, all user questions addressed ✅
2. **GATE 2 — File at correct path:** `blitzy/documentation/app_2cd6ee777f8c.md` confirmed ✅
3. **GATE 3 — Zero source files modified:** `git diff --name-only` confirmed read-only constraint ✅
4. **GATE 4 — Source citations verified:** All 26 line number references validated against actual codebase ✅

---

## 4. Runtime Validation & UI Verification

This project is a documentation-only deliverable. No application runtime, API endpoints, or UI components were created or modified. Runtime validation is not applicable.

**Document Structural Validation:**
- ✅ Markdown syntax valid — all headers, tables, code blocks, and lists render correctly
- ✅ 3 Mermaid diagram blocks properly opened and closed with fenced syntax
- ✅ No trailing whitespace (pre-commit hook compliant)
- ✅ File size: 740 lines / 46,066 bytes
- ✅ All 128 table rows have consistent column alignment
- ⚠️ Mermaid rendering depends on platform support (GitHub natively renders; other viewers may vary)

---

## 5. Compliance & Quality Review

| AAP Requirement | Status | Evidence |
|-----------------|--------|----------|
| Create `blitzy/documentation/app_2cd6ee777f8c.md` | ✅ Pass | File exists at correct path, 740 lines, committed |
| Document Workflow 1 — Mailbox Verification Enforcement | ✅ Pass | Lines 15-173: activation code generation, verify_mailbox_code() full trace, enforcement limits, terminal states, state machine diagram, test validation |
| Document Workflow 2 — Background Task Lifecycle | ✅ Pass | Lines 176-398: job creation/scheduling, eligibility query, processing loop, error recovery, observable state, state machine diagram, dispatcher |
| Document Workflow 3 — VERP Bounce Handling | ✅ Pass | Lines 401-721: VERP format spec, parsing, legacy formats, bounce detection/routing, forward/reply handling, comparison table, threshold rules, flowchart |
| Source citations with line numbers | ✅ Pass | 26 citations verified against codebase |
| 3 Mermaid diagrams (1 per workflow) | ✅ Pass | stateDiagram-v2 (verification), stateDiagram-v2 (job lifecycle), flowchart TD (bounce handling) |
| Summary tables for structured data | ✅ Pass | 128 table rows across 10+ tables |
| Rationale/thinking provided for key observations | ✅ Pass | 8 rationale blocks (destructive enforcement, stale-lock pattern, asymmetric bounce handling, check ordering, etc.) |
| No existing source files modified | ✅ Pass | `git diff --name-status` shows only `A blitzy/documentation/app_2cd6ee777f8c.md` |
| No temporary artifacts left | ✅ Pass | `git status` shows clean working tree |
| File named `<source_branch_name>.md` | ✅ Pass | `app_2cd6ee777f8c.md` matches branch name `app_2cd6ee777f8c` |
| Document `should_disable()` multi-tier thresholds | ✅ Pass | Lines 622-664: 4 tiers documented with scope, window, condition, and reason string |
| Document `JobState.error` never-set observation | ✅ Pass | Lines 205-207 and 326-328: documented with rationale |
| Document legacy VERP dual-format handling | ✅ Pass | Lines 481-495: legacy prefix patterns and dual-format routing |
| Document destructive `MailboxActivation` enforcement | ✅ Pass | Lines 113-131: `DELETE` behavior, rationale, recovery path |

**Fixes Applied During Validation:**
- 3 code review findings addressed in second commit (`1e7b0792`): 5 line insertions, 5 line deletions — minor corrections to content accuracy

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Source code line number references become stale after upstream changes | Technical | Medium | Medium | Re-verify all 26 citations before publishing if source branch has been updated | Open — requires human verification |
| Mermaid diagrams may not render in all Markdown viewers | Technical | Low | Low | GitHub natively supports Mermaid; for other platforms, consider exporting diagrams as PNG fallbacks | Acknowledged |
| Document not discoverable from main documentation index | Operational | Low | High | Add cross-reference link from `docs/code-structure.md` or `README.md` | Open — human task |
| Behavioral observations may not reflect future code changes | Technical | Low | Medium | Document is a point-in-time analysis; add a "Last verified" datestamp header | Open — human task |
| No automated validation pipeline for documentation accuracy | Operational | Low | Medium | Consider adding a CI step that checks referenced file paths exist and line ranges are valid | Open — future enhancement |

---

## 7. Visual Project Status

```mermaid
pie title Project Hours Breakdown
    "Completed Work" : 30
    "Remaining Work" : 4
```

**Remaining Work by Priority:**

| Priority | Category | Hours |
|----------|----------|-------|
| High | Peer review of technical accuracy | 2 |
| High | Line number citation verification | 1 |
| Medium | Editorial and formatting review | 1 |
| **Total** | | **4** |

---

## 8. Summary & Recommendations

### Achievements

The Blitzy autonomous agents successfully delivered a comprehensive 740-line runtime behavior analysis document that fully addresses all three workflows specified in the Agent Action Plan. The project is 88.2% complete (30 hours completed out of 34 total hours). Every AAP requirement — including all workflow documentation, Mermaid diagrams, summary tables, source citations, and constraints — has been fulfilled. The Final Validator confirmed all four production-readiness gates passed.

### Remaining Gaps

The remaining 4 hours of work are standard human review tasks required before production deployment:

1. **Peer review** (2h): A human developer familiar with the SimpleLogin codebase should verify that all behavioral claims accurately reflect the source code, paying particular attention to the `should_disable()` threshold logic and the job retry semantics.
2. **Citation freshness** (1h): All 26 line number references should be verified against the current state of the source branch, as any upstream commits could shift line numbers.
3. **Editorial polish** (1h): Final pass for clarity, grammar, and terminology consistency.

### Production Readiness Assessment

The document is **ready for human review and merge** pending the three remaining tasks above. No blocking issues were identified. The document meets all AAP quality requirements: comprehensive source citations, embedded diagrams, structured tables, and clear rationale for all key behavioral observations.

### Recommendations

1. Merge the document after completing the peer review — no code changes are required.
2. Consider establishing a periodic review cadence (e.g., quarterly) to keep line number citations current as the codebase evolves.
3. Link the document from `docs/code-structure.md` (currently a stub) to improve discoverability for future contributors.

---

## 9. Development Guide

### System Prerequisites

This is a documentation-only project. The deliverable is a standalone Markdown file that requires no build step, compilation, or runtime environment. To review and verify the document, you need:

| Prerequisite | Version | Purpose |
|--------------|---------|---------|
| Git | 2.x+ | Clone repository and inspect the deliverable |
| Markdown viewer | Any | Render the document (VS Code, GitHub web UI, or any Markdown renderer with Mermaid support) |
| Python | 3.10+ | Only needed if verifying source code references interactively |
| Text editor | Any | Review and edit the Markdown file |

### Environment Setup

```bash
# 1. Clone the repository and switch to the feature branch
git clone <repository_url>
cd <repository_directory>
git checkout blitzy-be22c4b0-019d-42e9-a9ed-f74dac9a3a2e

# 2. Verify the deliverable exists
ls -la blitzy/documentation/app_2cd6ee777f8c.md
# Expected output: -rw-r--r-- ... 46066 ... blitzy/documentation/app_2cd6ee777f8c.md

# 3. Check that no source files were modified
git diff --name-status origin/app_2cd6ee777f8c...HEAD
# Expected output: A    blitzy/documentation/app_2cd6ee777f8c.md
```

### Viewing the Document

```bash
# Option 1: View in terminal
cat blitzy/documentation/app_2cd6ee777f8c.md

# Option 2: Count key metrics
wc -l blitzy/documentation/app_2cd6ee777f8c.md
# Expected: 740 lines

grep -c "Source:" blitzy/documentation/app_2cd6ee777f8c.md
# Expected: 26 citations

grep -c "mermaid" blitzy/documentation/app_2cd6ee777f8c.md
# Expected: 3 (opening tags for 3 diagram blocks)
```

For best rendering with Mermaid diagrams, open the file in GitHub's web UI or VS Code with a Mermaid preview extension.

### Verifying Source Citations

To verify that line number references in the document match the actual source code:

```bash
# Example: Verify MAX_ACTIVATION_TRIES at app/mailbox_utils.py:43
sed -n '43p' app/mailbox_utils.py
# Should show: MAX_ACTIVATION_TRIES = 3

# Example: Verify JobState enum at app/models.py:253-257
sed -n '253,257p' app/models.py
# Should show the JobState enum with ready=0, taken=1, done=2, error=3

# Example: Verify VERP_PREFIX at app/config.py:500
sed -n '500p' app/config.py
# Should show VERP_PREFIX configuration

# Example: Verify get_jobs_to_run at job_runner.py:307-326
sed -n '307,326p' job_runner.py
# Should show the eligibility query function
```

### Troubleshooting

| Issue | Resolution |
|-------|------------|
| Mermaid diagrams not rendering | Use GitHub web UI or install a Mermaid-compatible Markdown preview extension (e.g., `bierner.markdown-mermaid` for VS Code) |
| Line numbers don't match source | The source code may have been updated since the document was written. Re-verify against the specific commit that the document was authored against (`1fb56506`) |
| File not found at expected path | Ensure you are on the correct branch: `git checkout blitzy-be22c4b0-019d-42e9-a9ed-f74dac9a3a2e` |

---

## 10. Appendices

### A. Command Reference

| Command | Purpose |
|---------|---------|
| `git diff --name-status origin/app_2cd6ee777f8c...HEAD` | Verify only the documentation file was changed |
| `wc -l blitzy/documentation/app_2cd6ee777f8c.md` | Count document lines (expected: 740) |
| `grep -c "Source:" blitzy/documentation/app_2cd6ee777f8c.md` | Count source citations (expected: 26) |
| `grep -c "mermaid" blitzy/documentation/app_2cd6ee777f8c.md` | Count Mermaid diagram blocks (expected: 3) |
| `git log --oneline HEAD --not origin/app_2cd6ee777f8c` | View commits on the feature branch |
| `git status` | Verify clean working tree |

### C. Key File Locations

| File | Purpose |
|------|---------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **Deliverable** — Runtime behavior analysis document (740 lines) |
| `app/mailbox_utils.py` | Source — Mailbox verification logic (Workflow 1) |
| `app/models.py` | Source — Data models: MailboxActivation, Job, JobState, VerpType, EmailLog, Bounce, RefusedEmail |
| `app/config.py` | Source — Configuration constants: JOB_MAX_ATTEMPTS, VERP_PREFIX, BOUNCE_PREFIX, etc. |
| `job_runner.py` | Source — Background job polling loop and processing (Workflow 2) |
| `email_handler.py` | Source — Inbound email handler, VERP routing, bounce processing (Workflow 3) |
| `app/email_utils.py` | Source — VERP generation/parsing, should_disable() bounce thresholds |
| `app/errors.py` | Source — Exception hierarchy (VERPTransactional, VERPForward, VERPReply) |
| `app/dashboard/views/mailbox.py` | Source — Web dashboard verification route |
| `tests/test_mailbox_utils.py` | Source — Verification behavior test suite |
| `tests/test_email_utils.py` | Source — VERP and bounce threshold test suite |

### D. Technology Versions

| Technology | Version | Role |
|------------|---------|------|
| Python | ^3.10 | Runtime language for SimpleLogin application |
| Flask | ^1.1.2 | Web framework hosting dashboard routes |
| SQLAlchemy | 1.3.24 | ORM layer for all data models |
| PostgreSQL | 13+ | Primary database |
| Arrow | ^0.16.0 | Timestamp arithmetic in verification and scheduling |
| aiosmtpd | ^1.2 | SMTP server for inbound email handling |
| Poetry | Latest | Python dependency management |
| Node.js | 10.17.0 | Frontend asset compilation |
| Mermaid | GitHub-native | Diagram rendering in Markdown |
| Git | 2.x+ | Version control |

### E. Environment Variable Reference

The following environment variables are referenced in the documented workflows (from `example.env` and `app/config.py`):

| Variable | Default/Example | Relevance |
|----------|----------------|-----------|
| `VERP_PREFIX` | `"sl"` | Prefix for VERP bounce addresses |
| `VERP_EMAIL_SECRET` | *(must be set, ≥32 chars)* | HMAC signing key for VERP addresses |
| `VERP_MESSAGE_LIFETIME` | `432000` (5 days) | Maximum VERP timestamp age |
| `JOB_MAX_ATTEMPTS` | `5` | Maximum retry attempts for background jobs |
| `JOB_TAKEN_RETRY_WAIT_MINS` | `30` | Minutes before a stale job becomes re-eligible |
| `ALIAS_AUTOMATIC_DISABLE` | `True` | Feature flag for auto-disabling aliases on bounces |
| `BOUNCE_PREFIX` | `"bounce+"` | Legacy forward-phase bounce address prefix |
| `BOUNCE_SUFFIX` | `"+@{EMAIL_DOMAIN}"` | Legacy forward-phase bounce address suffix |
| `MAILBOX_VERIFICATION_OVERRIDE_CODE` | *(unset)* | Testing override for verification codes |

### G. Glossary

| Term | Definition |
|------|------------|
| **VERP** | Variable Envelope Return Path — a technique where each outgoing email uses a unique envelope sender address, enabling bounce attribution to the specific original email |
| **Activation Code** | A 6-digit numeric or URL-safe token generated for mailbox verification, stored in the `MailboxActivation` model |
| **Stale-Lock Pattern** | The job runner's implicit failure detection mechanism: a job left in `taken` state with an aged `taken_at` timestamp is presumed failed and becomes re-eligible |
| **Forward Phase** | The email direction from an external sender through an alias to the user's mailbox |
| **Reply Phase** | The email direction from the user's mailbox through an alias to an external contact |
| **DSN** | Delivery Status Notification — the bounce report format (RFC 3464) with Content-Type `multipart/report` |
| **HMAC-SHA3-224** | The cryptographic signing algorithm used for VERP addresses, truncated to 8 bytes for compactness |
| **should_disable()** | A multi-tier bounce threshold function that evaluates whether an alias should be automatically disabled based on forward-phase bounce volume |
| **JobState** | Enum with values: ready (0), taken (1), done (2), error (3). Note: error state is defined but never used by the job runner |
| **MailboxActivation** | Database model storing verification codes with a tries counter and created_at timestamp for expiry enforcement |