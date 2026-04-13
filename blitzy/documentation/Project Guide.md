# Blitzy Project Guide — Alias Reply-Handling Flow Investigation

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a comprehensive investigative analysis of the alias reply-handling flow in the SimpleLogin email aliasing system. The sole deliverable is a markdown document (`blitzy/documentation/app_2cd6ee777f8c.md`) that traces the end-to-end runtime path for inbound reply messages through the SMTP handler, maps every routing decision point with exact code references, and identifies the most likely component where incorrect user routing could originate. The investigation covers 9 critical source files (~8,700 lines of code), documents 13 pipeline stages, ranks 7 failure points by likelihood and impact, and analyzes 4 edge cases — all grounded in static code analysis with no modifications to the existing codebase.

### 1.2 Completion Status

```mermaid
pie title Project Completion
    "Completed (35h)" : 35
    "Remaining (6h)" : 6
```

| Metric | Value |
|---|---|
| **Total Project Hours** | **41** |
| **Completed Hours (AI)** | **35** |
| **Remaining Hours** | **6** |
| **Completion Percentage** | **85.4%** |

**Calculation:** 35 completed hours / (35 completed + 6 remaining) = 35 / 41 = **85.4% complete**

### 1.3 Key Accomplishments

- ✅ Created comprehensive 944-line investigation document with 104 verified code references
- ✅ Traced complete 13-stage reply pipeline from `MailHandler.handle_DATA()` through `sl_sendmail()` delivery
- ✅ Identified and ranked 7 distinct failure points where incorrect routing could originate
- ✅ Analyzed 4 edge cases: canonical email matching, multi-mailbox aliases, dual-path `is_reverse_alias()` detection, `NonReverseAliasInReplyPhase` exception handling
- ✅ Produced Mermaid data flow diagram mapping all routing decision points
- ✅ Primary finding documented: `Contact.get_by(reply_email=...)` at `email_handler.py:986` is the highest-likelihood point for incorrect routing
- ✅ All 191 Python files compile with zero errors
- ✅ 634/639 tests pass (5 out-of-scope failures in `test_mail_sender.py` due to container IPv6 limitations)
- ✅ Zero existing files modified — investigation is purely analytical
- ✅ Applied factual correction pass in second commit to ensure code reference accuracy

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|---|---|---|---|
| Investigation findings require SME validation against production behavior | Findings may not fully reflect production edge cases or undocumented patches | Engineering Team | 1–2 days after merge |
| Code reference line numbers may drift if `email_handler.py` is modified on main branch | References in document could become stale after upstream commits | Human Developer | Ongoing maintenance |

### 1.5 Access Issues

No access issues identified. The investigation is a read-only static code analysis that does not require external service credentials, API keys, or deployment infrastructure. All analysis was performed against the local repository checkout.

### 1.6 Recommended Next Steps

1. **[High]** Have a subject-matter expert who owns the reply pipeline review the investigation findings for accuracy and completeness — particularly the 7 ranked failure points and the primary conclusion about `Contact.get_by()` (3h)
2. **[High]** Validate the primary finding by searching production logs for instances where `Contact.get_by(reply_email=...)` returned an unexpected Contact record (included in the 3h expert review)
3. **[Medium]** Verify that all 104 code reference line numbers in the document match the latest state of the main branch before merging (1h)
4. **[Medium]** Cross-reference the edge case analysis (Section 3 of the document) with production runtime logs to confirm the identified failure modes have been observed in practice (2h)
5. **[Low]** Consider adding a database-level unique constraint on `Contact.reply_email` to mitigate Failure Point #1, as recommended in the document's conclusion

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|---|---|---|
| Repository & Code Analysis | 8 | Deep static analysis of 9 critical files (~8,700 LOC): `email_handler.py`, `app/email_utils.py`, `app/email_validation.py`, `app/models.py`, `app/utils.py`, `app/contact_utils.py`, `app/errors.py`, `app/handler/dmarc.py`, `app/mail_sender.py`; plus test infrastructure, config files, and EML fixtures |
| Reply Pipeline Trace (13 Stages) | 10 | Detailed per-stage walkthrough from SMTP reception → Flask context → central dispatch → reverse-alias detection → reply entry → Contact resolution → DMARC → mailbox authorization → SPF → EmailLog → spam check → header manipulation → DKIM/delivery, with exact code references per stage |
| Failure Point Identification & Ranking | 5 | Identified, analyzed, and ranked 7 distinct failure points with evidence-based rationale: Contact resolution query, reply email normalization, `is_reverse_alias()` dual-path, canonical email mismatch, spoofing check fallback, `replace_header_when_reply()` email drop, stale Contact records |
| Edge Case Analysis | 3 | 4 detailed edge cases: canonical vs non-canonical email matching in `canonicalize_email()`, multi-mailbox alias mailbox selection, `is_reverse_alias()` dual-path detection inconsistency, `NonReverseAliasInReplyPhase` exception behavior |
| Data Flow Diagram | 1 | Mermaid flowchart documenting complete reply-phase data flow with all routing decision points and failure paths |
| Documentation Writing & Formatting | 4 | Structured 944-line markdown document with table of contents, executive summary, 5 major sections, ranked failure table, code blocks, and cross-referenced code citations |
| Factual Correction & Review | 2 | Second commit (`fef52b07`) correcting factual errors in initial analysis; verification of line numbers and function signatures against actual source |
| Compilation Verification | 1 | Compiled all 191 Python files in `app/` directory plus root `email_handler.py` with zero errors |
| Test Suite Execution & Analysis | 1 | Executed full test suite (639 tests); analyzed 5 out-of-scope failures in `test_mail_sender.py`; confirmed 23/23 email handler tests pass; verified no in-scope regressions |
| **Total Completed** | **35** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|---|---|---|
| Human Expert Review of Investigation Findings | 3 | High |
| Code Reference Verification Against Latest Main Branch | 1 | Medium |
| Cross-reference Findings with Production Runtime Logs | 2 | Medium |
| **Total Remaining** | **6** | |

**Integrity Check:** 35 (completed) + 6 (remaining) = 41 (total project hours) ✓

---

## 3. Test Results

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---|---|---|---|---|---|---|
| Unit & Integration (Full Suite) | pytest 7.x | 639 | 634 | 5 | N/A | 5 failures all in out-of-scope `test_mail_sender.py` — IPv6 socket binding unavailable in container (`OSError: [Errno 99] Cannot assign requested address`) |
| Email Handler Tests | pytest 7.x | 23 | 23 | 0 | N/A | All reply-phase, forward-phase, and routing tests pass (100%) |
| Python Compilation Check | py_compile | 191 | 191 | 0 | 100% | All Python files in `app/` directory compile without errors |
| Root Module Compilation | py_compile | 1 | 1 | 0 | 100% | `email_handler.py` compiles without errors |

**Out-of-scope failures (5):**
1. `tests/test_mail_sender.py::test_mail_sender_save_unsent_to_disk[closed_dummy_server]`
2. `tests/test_mail_sender.py::test_mail_sender_save_unsent_to_disk[inner0]`
3. `tests/test_mail_sender.py::test_mail_sender_save_unsent_to_disk[inner1]`
4. `tests/test_mail_sender.py::test_send_unsent_email_from_fs`
5. `tests/test_mail_sender.py::test_failed_resend_does_not_delete_file`

All 5 failures are caused by IPv6 socket binding (`::1`) being unavailable in the CI container environment. These are pre-existing environment-specific issues unrelated to this documentation-only task. No modifications were made to `app/mail_sender.py` or `tests/test_mail_sender.py`.

---

## 4. Runtime Validation & UI Verification

### Runtime Health

- ✅ **Python Compilation**: All 192 Python files (191 in `app/` + `email_handler.py` root) compile without errors
- ✅ **SMTP Handler Module**: `email_handler.py` (2,404 lines) loads and parses correctly
- ✅ **Core Pipeline Modules**: `app/email_utils.py`, `app/email_validation.py`, `app/models.py`, `app/utils.py`, `app/contact_utils.py` all compile successfully
- ✅ **Test Infrastructure**: pytest discovery finds 639 tests across 35+ test files; all test fixtures and conftest.py modules load correctly
- ✅ **Documentation Deliverable**: `blitzy/documentation/app_2cd6ee777f8c.md` exists (944 lines, 50,967 bytes), with valid Mermaid syntax and properly formatted markdown

### UI Verification

Not applicable — this is a documentation-only investigation task with no UI components.

### API Integration Verification

Not applicable — no API endpoints were created or modified. The investigation is a static code analysis with no runtime SMTP interaction.

### Git Repository Integrity

- ✅ **Branch**: `blitzy-ba251627-6167-4efe-9077-4a4833bce80a` — 2 commits ahead of `origin/app_2cd6ee777f8c`
- ✅ **Files Changed**: 1 file added (`blitzy/documentation/app_2cd6ee777f8c.md`), 0 existing files modified
- ✅ **Working Directory**: Clean (only `dump.rdb` untracked — Redis dump correctly excluded)
- ✅ **No Unintended Modifications**: `git diff --name-status origin/app_2cd6ee777f8c...HEAD` shows only the single documentation file addition

---

## 5. Compliance & Quality Review

| AAP Requirement | Status | Evidence |
|---|---|---|
| End-to-end trace of inbound reply flow | ✅ Pass | 13 stages documented (Stages 1–13) covering SMTP reception through outbound delivery |
| Map exact data flow from reception to delivery | ✅ Pass | Mermaid data flow diagram in Section 4 of deliverable |
| Identify specific component for incorrect routing | ✅ Pass | 7 ranked failure points in Section 2 of deliverable; primary finding: `Contact.get_by()` at `email_handler.py:986` |
| Explain complete reply-phase mechanism | ✅ Pass | Reverse-alias generation, Contact resolution, mailbox authorization, header rewriting, VERP delivery all covered |
| Analyze `normalize_reply_email()` normalization | ✅ Pass | Failure Point #2 with detailed lossy transformation analysis |
| Examine `is_reverse_alias()` dual-path logic | ✅ Pass | Failure Point #3 with Path A/B inconsistency analysis; Edge Case Section 3.3 |
| Canonical vs non-canonical email matching | ✅ Pass | Edge Case Section 3.1 with `canonicalize_email()` analysis |
| Multi-mailbox aliases analysis | ✅ Pass | Edge Case Section 3.2 with `get_mailbox_from_mail_from()` analysis |
| Disabled spoofing checks analysis | ✅ Pass | Failure Point #5 with silent fallback analysis |
| Stale Contact records analysis | ✅ Pass | Failure Point #7 with `create_contact()` IntegrityError handling analysis |
| Evidence-based with exact code references | ✅ Pass | 104 code references with file paths and line numbers verified against repository |
| Create `app_2cd6ee777f8c.md` in `blitzy/documentation` | ✅ Pass | File exists: 944 lines, 50,967 bytes, 2 commits |
| No modifications to existing files | ✅ Pass | `git diff --name-status` shows only 1 file added, 0 modified |
| No additional code files added | ✅ Pass | Only markdown documentation file added |
| Cleanup temporary artifacts | ✅ Pass | No temp scripts, logs, or instrumentation in repository |
| Provide thinking/rationale behind answers | ✅ Pass | Every conclusion in the document includes "Rationale" and "Evidence of risk" subsections |

### Autonomous Validation Fixes Applied

| Fix | Commit | Description |
|---|---|---|
| Factual error correction | `fef52b07` | Corrected factual errors in the investigation document identified during the validation review pass |

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|---|---|---|---|---|---|
| Line number references become stale after upstream code changes | Technical | Medium | High | Pin references to commit hash `2cd6ee77`; note in document header that line numbers are verified against this specific commit | Mitigated |
| Investigation findings may not fully reflect production edge cases | Technical | Medium | Medium | Recommend SME review and cross-referencing with production logs as immediate next steps | Open — requires human action |
| 5 out-of-scope test failures in `test_mail_sender.py` | Technical | Low | Certain (environment-specific) | IPv6 (`::1`) socket binding unavailable in container; pre-existing issue unrelated to this task | Accepted — out of scope |
| `Contact.reply_email` column lacks unique constraint (finding from investigation) | Security | High | Low | Document recommends adding database-level unique constraint; requires schema migration by human developer | Open — recommendation only |
| `normalize_reply_email()` lossy transformation could cause routing collisions | Technical | High | Low | Documented as Failure Point #2; requires code change to normalize at generation time to eliminate asymmetry | Open — recommendation only |
| Document may contain incorrect conclusions if code analysis missed a path | Operational | Medium | Low | Two-pass review (initial analysis + factual correction commit); recommended human SME validation | Mitigated — pending human review |
| `dump.rdb` untracked file in working directory | Operational | Low | Certain | Redis dump file from test environment; correctly excluded from git tracking via `.gitignore` patterns | Accepted |

---

## 7. Visual Project Status

```mermaid
pie title Project Hours Breakdown
    "Completed Work" : 35
    "Remaining Work" : 6
```

**Completed Work (35h):** Repository analysis (8h), reply pipeline trace (10h), failure point identification (5h), edge case analysis (3h), data flow diagram (1h), documentation writing (4h), factual correction (2h), compilation verification (1h), test execution (1h)

**Remaining Work (6h):** Human expert review (3h), code reference verification (1h), production log cross-reference (2h)

---

## 8. Summary & Recommendations

### Achievement Summary

The Blitzy autonomous agents successfully delivered a comprehensive 944-line investigation document analyzing the alias reply-handling flow in the SimpleLogin email aliasing system. The project is **85.4% complete** (35 hours completed out of 41 total hours). All AAP-scoped deliverables have been implemented: the end-to-end reply pipeline trace (13 stages), failure point identification (7 ranked points), edge case analysis (4 cases), data flow diagram, and the documentation file itself — all with 104 verified code references. The investigation concluded that `Contact.get_by(reply_email=...)` at `email_handler.py:986` is the most likely single point where incorrect user routing could originate, with the `normalize_reply_email()` normalization step being a compounding risk factor.

### Remaining Gaps

The 6 remaining hours consist entirely of human validation tasks that require domain expertise and production system access:
1. **Subject-matter expert review** (3h) to validate the accuracy of routing analysis and failure point rankings against actual production behavior
2. **Code reference verification** (1h) to ensure line numbers still match if the main branch has received upstream commits since analysis
3. **Production log cross-reference** (2h) to confirm that the identified failure modes have been observed in real-world operation

### Production Readiness Assessment

The documentation deliverable is **production-ready for merge**. No source code was modified, all 191 Python files compile without errors, 634/639 tests pass (5 out-of-scope failures), and the document itself is complete and self-contained. The remaining 6 hours of human validation work can be performed post-merge as part of the team's normal review process.

### Key Recommendations

1. **Prioritize SME review** of the primary finding (Contact resolution as highest-risk failure point) before implementing any code changes based on the investigation
2. **Consider adding a unique constraint** on `Contact.reply_email` at the database level, as recommended in the document's conclusion, to eliminate the race condition risk identified in Failure Point #1
3. **Normalize at generation time** — apply `normalize_reply_email()` during Contact creation (in `create_contact()`) to ensure the stored value matches what will be looked up during reply processing, eliminating the asymmetry identified in Failure Point #2

---

## 9. Development Guide

### System Prerequisites

| Software | Version | Purpose |
|---|---|---|
| Python | 3.10+ | Runtime (targeted via `target-version = ['py310']` in Black config) |
| Poetry | 2.x | Dependency management |
| PostgreSQL | 12+ | Database backend |
| Redis | 6+ | Session store and rate limiter |
| Git | 2.x | Version control |

### Environment Setup

```bash
# 1. Clone the repository and checkout the branch
git clone <repository-url>
cd app
git checkout blitzy-ba251627-6167-4efe-9077-4a4833bce80a

# 2. Install Python dependencies via Poetry
poetry install --no-interaction --no-ansi

# 3. Ensure PostgreSQL is running and create test database
# (Adjust credentials as needed for your environment)
createdb -U test test

# 4. Set required environment variables
export DB_URI="postgresql://test:test@localhost:5432/test"
export CONFIG=tests/test.env
export GITHUB_ACTIONS_TEST=true
```

### Dependency Installation

```bash
# Install all dependencies (production + dev)
poetry install --no-interaction --no-ansi

# Verify installation
poetry run python -c "import flask; import sqlalchemy; print('Dependencies OK')"
```

### Compilation Verification

```bash
# Compile the main SMTP handler
poetry run python -m py_compile email_handler.py

# Compile all application Python files
poetry run python -c "
import py_compile, os
count = errors = 0
for r, d, fs in os.walk('app'):
    for f in fs:
        if f.endswith('.py'):
            count += 1
            try:
                py_compile.compile(os.path.join(r, f), doraise=True)
            except py_compile.PyCompileError as e:
                errors += 1
                print(f'ERROR: {e}')
print(f'Compiled {count} files, {errors} errors')
"
# Expected output: Compiled 191 files, 0 errors
```

### Running Tests

```bash
# Run the full test suite
env DB_URI="postgresql://test:test@localhost:5432/test" \
    CONFIG=tests/test.env \
    GITHUB_ACTIONS_TEST=true \
    poetry run pytest --tb=short --no-header -q
# Expected: 634 passed, 5 failed (out-of-scope mail_sender IPv6 issue)

# Run only email handler tests (most relevant to this investigation)
env DB_URI="postgresql://test:test@localhost:5432/test" \
    CONFIG=tests/test.env \
    GITHUB_ACTIONS_TEST=true \
    poetry run pytest tests/test_email_handler.py --tb=short --no-header -q
# Expected: 23 passed in ~6 seconds
```

### Viewing the Investigation Document

```bash
# The investigation document is located at:
cat blitzy/documentation/app_2cd6ee777f8c.md

# Or view specific sections:
# Table of contents
sed -n '17,26p' blitzy/documentation/app_2cd6ee777f8c.md

# Primary finding (Failure Point #1)
sed -n '619,660p' blitzy/documentation/app_2cd6ee777f8c.md

# Ranked failure points summary table
sed -n '778,800p' blitzy/documentation/app_2cd6ee777f8c.md
```

### Troubleshooting

| Issue | Resolution |
|---|---|
| `OSError: [Errno 99] Cannot assign requested address` in `test_mail_sender.py` | IPv6 (`::1`) not available in container — these 5 tests are out-of-scope and pre-existing |
| Poetry deprecation warnings about `poetry.dev-dependencies` | Cosmetic warning only; does not affect functionality |
| `DeprecationWarning: setDaemon()` from gnupg.py | Third-party library warning; does not affect test results |
| PostgreSQL connection refused | Ensure PostgreSQL is running on `localhost:5432` with user `test`, password `test`, database `test` |

---

## 10. Appendices

### A. Command Reference

| Command | Purpose |
|---|---|
| `poetry run python -m py_compile email_handler.py` | Compile-check the main SMTP handler |
| `poetry run pytest tests/test_email_handler.py --tb=short -q` | Run email handler test suite |
| `poetry run pytest --tb=short --no-header -q` | Run the full test suite |
| `git diff --stat origin/app_2cd6ee777f8c...HEAD` | View files changed on this branch |
| `git log --oneline HEAD --not origin/app_2cd6ee777f8c` | View commits on this branch |

### B. Port Reference

| Port | Service | Notes |
|---|---|---|
| 5432 | PostgreSQL | Database backend for test suite |
| 6379 | Redis | Session store (used by test fixtures) |
| 7777 | Gunicorn/Flask | Web application (not used in this investigation) |

### C. Key File Locations

| File | Purpose |
|---|---|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **Investigation deliverable** — the sole output of this project |
| `email_handler.py` | Primary analysis target — SMTP inbound handler (2,404 lines) |
| `app/email_utils.py` | Reverse-alias utilities: `is_reverse_alias()`, `generate_reply_email()` (1,505 lines) |
| `app/email_validation.py` | `normalize_reply_email()` normalization logic (38 lines) |
| `app/models.py` | ORM models: Contact, Alias, Mailbox, EmailLog (3,843 lines) |
| `app/utils.py` | `sanitize_email()`, `canonicalize_email()` address utilities (158 lines) |
| `app/contact_utils.py` | `create_contact()` with reply_email assignment (120 lines) |
| `app/errors.py` | Exception classes: `NonReverseAliasInReplyPhase`, `VERPReply` (130 lines) |
| `tests/test_email_handler.py` | Email handler test suite — 23 tests (413 lines) |

### D. Technology Versions

| Technology | Version | Source |
|---|---|---|
| Python | ^3.10 (runtime 3.10 in Dockerfile) | `pyproject.toml`, `Dockerfile` |
| Flask | ^1.1.2 | `pyproject.toml` |
| SQLAlchemy | 1.3.24 (pinned) | `pyproject.toml` |
| aiosmtpd | ^1.2 | `pyproject.toml` |
| pytest | ^7.0.0 | `pyproject.toml` |
| PostgreSQL | 12+ (test environment) | `tests/conftest.py` |
| Poetry | 2.3.4 (build tool) | Runtime detected |
| dkimpy | ^1.0.5 | `pyproject.toml` |
| email_validator | ^1.1.1 | `pyproject.toml` |

### E. Environment Variable Reference

| Variable | Example Value | Purpose |
|---|---|---|
| `DB_URI` | `postgresql://test:test@localhost:5432/test` | PostgreSQL connection string |
| `CONFIG` | `tests/test.env` | Path to environment configuration file |
| `GITHUB_ACTIONS_TEST` | `true` | Enables CI-specific test behavior |
| `EMAIL_DOMAIN` | `sl.local` | Domain used for alias creation (from `example.env`) |
| `NOT_SEND_EMAIL` | `true` | Print email content instead of sending (development mode) |
| `ENFORCE_SPF` | `true` | Enable SPF enforcement via Postfix headers |

### G. Glossary

| Term | Definition |
|---|---|
| **Reverse-alias** | A generated email address (e.g., `abc123@sl.local`) that maps back to a Contact record, enabling reply routing through the alias system |
| **Contact** | A database record representing an external email correspondent linked to an Alias; contains `reply_email` (reverse-alias) and `website_email` (real address) |
| **Alias** | A proxy email address owned by a User; forwards inbound mail to the user's Mailbox and routes replies back through reverse-aliases |
| **Mailbox** | A user's real email address registered with SimpleLogin for receiving forwarded emails |
| **VERP** | Variable Envelope Return Path — a technique for tracking bounces by encoding metadata in the return address |
| **SPF** | Sender Policy Framework — DNS-based email authentication mechanism |
| **DKIM** | DomainKeys Identified Mail — cryptographic email signing mechanism |
| **DMARC** | Domain-based Message Authentication, Reporting and Conformance — email authentication policy framework |
| **Canonical email** | The normalized form of an email address with dots removed and plus-suffixes truncated (for Gmail/ProtonMail domains) |