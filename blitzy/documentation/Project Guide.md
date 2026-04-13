# Blitzy Project Guide

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a deep investigative analysis of SimpleLogin's inbound reply-email resolution logic — specifically how the system derives the reply email address from an inbound SMTP envelope, resolves it to an associated `Contact` record, and selects the forwarding destination (alias and user). The investigation traces both the forward-phase contact creation path (where `reply_email` values are generated) and the reply-phase resolution path (where those values are looked up), analyzing race conditions, uniqueness assumption violations, and timing-related inconsistencies. The sole deliverable is a 778-line markdown analysis document placed at `blitzy/documentation/app_2cd6ee777f8c.md`, with zero modifications to existing source files.

### 1.2 Completion Status

```mermaid
pie title Project Completion Status
    "Completed (32h)" : 32
    "Remaining (6h)" : 6
```

| Metric | Value |
|--------|-------|
| **Total Project Hours** | 38 |
| **Completed Hours (AI)** | 32 |
| **Remaining Hours** | 6 |
| **Completion Percentage** | 84.2% |

**Calculation**: 32 completed hours / (32 + 6) total hours = 84.2% complete

### 1.3 Key Accomplishments

- ✅ Traced the complete reply-email resolution data flow from SMTP envelope reception through `handle_reply()` contact lookup to outbound delivery (12 documented steps)
- ✅ Documented concrete runtime values at each resolution step with happy-path and failure scenario tables
- ✅ Identified and documented the TOCTOU race condition in `generate_reply_email()` → `available_sl_email()` → `Contact.create()` chain
- ✅ Confirmed `reply_email` column lacks database-level unique constraint (migration evidence: `unique=False` in `2021_071310_78403c7b8089_.py`)
- ✅ Analyzed `.first()` non-determinism in `ModelMixin.get_by()` for duplicate `reply_email` scenarios
- ✅ Documented the mailbox authorization check as an imperfect safety net against cross-user misrouting
- ✅ Evaluated single-connection scoped-session architecture in `app/db.py` for concurrency implications
- ✅ Produced comprehensive 778-line analysis document with 30+ verified code references
- ✅ Verified zero source file modifications — read-only constraint fully honored
- ✅ All code citations spot-checked against actual source files for accuracy

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| Findings require security team peer review | TOCTOU analysis conclusions need validation by domain experts before acting on recommendations | Security Team | 2 hours |
| Stakeholder review pending | Engineering leads should confirm investigation scope and conclusions are sufficient | Engineering Lead | 2 hours |

### 1.5 Access Issues

No access issues identified. The project is a read-only investigation producing a documentation artifact. All source code was accessible and all code citations were verified against the repository contents.

### 1.6 Recommended Next Steps

1. **[High]** Security team should peer-review the TOCTOU race condition analysis (Section 6 of the document) and validate conclusions before considering any remediation
2. **[High]** Engineering leads should review the full analysis document and confirm findings address all investigation objectives
3. **[Medium]** If remediation is desired, evaluate adding a `UNIQUE` constraint on `Contact.reply_email` with appropriate migration strategy
4. **[Medium]** Consider adding distributed locking for reply-email generation (similar to existing `parallel_limiter` pattern for alias creation)
5. **[Low]** After any future code changes to the analyzed files, verify that document line number references remain accurate

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|-------|-------------|
| Deep source code analysis | 8 | Systematic analysis of 8,000+ lines across `email_handler.py`, `app/contact_utils.py`, `app/email_utils.py`, `app/email_validation.py`, `app/models.py`, `app/db.py`, 3 migration files, and 4 test files |
| Reply-phase resolution documentation (Section 3) | 4 | Traced and documented all 12 steps of `handle_reply()` from SMTP entry through outbound delivery with exact line numbers |
| Forward-phase generation documentation (Section 2) | 3 | Documented 6 subsections covering `get_or_create_contact()`, `create_contact()`, `generate_reply_email()`, `available_sl_email()`, parallel creation path, and `Contact.create()` override |
| Race condition and TOCTOU analysis (Section 6) | 4 | Analyzed TOCTOU window with step-by-step timeline, session architecture, app context lifecycle, collision probability assessment, and missing distributed locking |
| Cross-user mismatch scenario analysis (Section 7) | 3 | Answered three key investigation questions with detailed code evidence and edge case analysis |
| Runtime values documentation (Section 4) | 2 | Created concrete runtime values table for happy path and documented 7 failure scenarios with trigger conditions |
| Database schema and migration analysis (Section 5) | 2 | Analyzed Contact model definition, 3 migration files, and synthesized implication summary |
| Behavioral conclusions synthesis (Section 8) | 1.5 | Synthesized 6 evidence-based conclusions from all investigation axes |
| Code citation verification | 2 | Spot-checked all 30+ file/function/line references against actual source code |
| Document structure and formatting (Section 9, ToC) | 1.5 | Created code references appendix with 30+ entries and migration reference table; structured ToC |
| Validation and environment verification | 1 | Verified compilation, ran test suite (634 passed), confirmed zero source modifications via git |
| **Total** | **32** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|-------|----------|
| Security team peer review of TOCTOU findings | 2 | High |
| Stakeholder review and feedback incorporation | 2 | Medium |
| Code reference freshness verification | 1 | Low |
| Editorial corrections from review feedback | 1 | Low |
| **Total** | **6** | |

---

## 3. Test Results

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|------------|-------|
| Unit + Integration | pytest | 639 | 634 | 5 | 55%+ (threshold) | Full repository test suite executed by Blitzy's autonomous validation |

**Details on failures**:
- All 5 failures are in `tests/test_mail_sender.py` — **out of scope** for this investigation
- Root cause: IPv6 `::1` loopback unavailability in the test environment (`OSError: [Errno 99] Cannot assign requested address`)
- These are environment-specific networking issues, not code bugs
- No in-scope test failures detected

**Note**: This project is a read-only investigation producing a documentation artifact. No new test files were created as the deliverable is a markdown analysis document. The existing test suite was executed to verify repository integrity remained intact with zero source modifications.

---

## 4. Runtime Validation & UI Verification

### Runtime Health

- ✅ **Repository integrity**: Zero source files modified — confirmed via `git diff origin/app_2cd6ee777f8c...blitzy-3e0bc84d-cc97-4cb4-a2be-f0ae22a1b5fe` showing only `A blitzy/documentation/app_2cd6ee777f8c.md`
- ✅ **Deliverable file exists**: `blitzy/documentation/app_2cd6ee777f8c.md` — 778 lines, committed at `d8ef4a00`
- ✅ **Git working tree clean**: Only untracked file is `dump.rdb` (Redis artifact, not project-related)
- ✅ **Dependencies intact**: All Python packages installed and importable (SQLAlchemy 1.3.24, Flask 1.1.2, aiosmtpd, psycopg2-binary)
- ✅ **Code citation accuracy**: All 30+ file/function/line references in the analysis document verified against actual source files

### Code Citation Spot-Checks

- ✅ `ModelMixin.get_by()` at `app/models.py` L82-84 — confirmed `.filter_by(**kw).first()` pattern
- ✅ `Contact.reply_email` at `app/models.py` L1899 — confirmed `sa.Column(sa.String(512), nullable=False, index=True)` without `unique=True`
- ✅ `available_sl_email()` at `app/models.py` L1425-1432 — confirmed SELECT-only check
- ✅ `handle_reply()` at `email_handler.py` L966-990 — confirmed `Contact.get_by(reply_email=reply_email)` lookup
- ✅ `generate_reply_email()` at `app/email_utils.py` L1103-1153 — confirmed loop with `available_sl_email()` check
- ✅ `is_reverse_alias()` at `app/email_utils.py` L1156-1163 — confirmed `Contact.get_by(reply_email=address)` call
- ✅ `app/db.py` engine/connection/Session at L9-14 — confirmed single connection architecture
- ✅ Migration `2021_071310_78403c7b8089_.py` L22 — confirmed `unique=False` on index creation

### UI Verification

Not applicable — this project produces a documentation artifact only, with no UI components.

---

## 5. Compliance & Quality Review

| Compliance Area | Requirement | Status | Evidence |
|----------------|-------------|--------|----------|
| Read-only constraint | No existing source files modified | ✅ Pass | `git diff` shows only `A blitzy/documentation/app_2cd6ee777f8c.md` |
| SWE-AtlasQnA-Repo rule | Document named `app_2cd6ee777f8c.md` in `blitzy/documentation/` | ✅ Pass | File exists at correct path, committed |
| Evidence-based conclusions | All findings cite specific file paths, line numbers, and function names | ✅ Pass | 30+ code references verified; Section 9 appendix catalogs all citations |
| Code as source of truth | No assumptions; all conclusions grounded in actual source code | ✅ Pass | Document traces exact code paths with extracted code snippets |
| Thinking and rationale | Document provides reasoning behind every answer | ✅ Pass | Each section includes "Rationale", "Reasoning", or "Evidence chain" subsections |
| Three key questions answered | Same reply_email → different contacts, transient failure, wrong user | ✅ Pass | Section 7 addresses all three with detailed evidence |
| TOCTOU race fully documented | Step-by-step timeline of the race condition | ✅ Pass | Section 6.1 provides 5-step TOCTOU timeline |
| Migration evidence cited | ORM definition AND migration history cross-referenced | ✅ Pass | Section 5 cites 3 migrations with revision IDs and line numbers |
| Document completeness | All 9 sections present per agent prompt specification | ✅ Pass | Sections 1-9 all present with Table of Contents |
| Test suite integrity | Existing tests pass without degradation | ✅ Pass | 634/639 passed; 5 failures are pre-existing environment-specific issues |

### Autonomous Validation Fixes Applied

No fixes were required. The deliverable is a documentation artifact that passed all validation checks on first commit:
- File content verified for completeness and accuracy
- Code citations spot-checked against source files
- Git state confirmed clean

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| TOCTOU findings may be disputed without security peer review | Technical | Medium | Medium | Schedule dedicated security team review session | Open |
| Line number references may become stale after future source changes | Technical | Low | Medium | Add a note in the document about reference freshness; re-verify after major changes | Documented |
| Findings could be misinterpreted as requiring immediate remediation | Operational | Medium | Low | Document clearly states the practical collision probability is astronomically low | Mitigated |
| Analysis scope may not fully satisfy all stakeholder questions | Operational | Low | Low | Stakeholder review step included in remaining work | Open |
| IPv6 test environment limitation prevents full test suite validation | Technical | Low | High | 5 failures in `test_mail_sender.py` are environment-specific; document as known limitation | Documented |

---

## 7. Visual Project Status

```mermaid
pie title Project Hours Breakdown
    "Completed Work" : 32
    "Remaining Work" : 6
```

**Remaining Work Breakdown by Priority:**

| Priority | Hours | Tasks |
|----------|-------|-------|
| High | 2 | Security team peer review of TOCTOU findings |
| Medium | 2 | Stakeholder review and feedback incorporation |
| Low | 2 | Code reference freshness verification + editorial corrections |
| **Total** | **6** | |

---

## 8. Summary & Recommendations

### Achievement Summary

The Blitzy autonomous agents successfully delivered a comprehensive 778-line investigative analysis document examining SimpleLogin's inbound reply-email resolution logic. The project is **84.2% complete** (32 hours completed out of 38 total hours). All AAP-scoped deliverables have been implemented:

- The complete reply-email resolution data flow was traced from SMTP envelope reception through `handle_reply()` contact lookup to outbound delivery, documenting 12 discrete steps with exact code locations.
- Runtime values were documented at each resolution step, including a concrete happy-path example and 7 failure scenarios.
- Three key cross-contact/cross-user mismatch questions were answered with evidence-based conclusions.
- The TOCTOU race condition in the `generate_reply_email()` → `available_sl_email()` → `Contact.create()` chain was fully analyzed, including a step-by-step timeline, collision probability assessment, and session architecture implications.
- The analysis document was delivered as `blitzy/documentation/app_2cd6ee777f8c.md` with zero source file modifications, honoring the read-only constraint.

### Remaining Gaps

The 6 remaining hours consist entirely of path-to-production peer review and editorial work:
- Security team peer review to validate TOCTOU race condition conclusions (2h)
- Stakeholder review to confirm investigation scope and findings (2h)
- Code reference freshness verification and editorial corrections (2h)

### Critical Path to Production

1. Security team validates the TOCTOU analysis and confirms the collision probability assessment
2. Engineering leads confirm all investigation questions have been answered satisfactorily
3. Any review feedback is incorporated into the document
4. Document is approved for distribution to relevant stakeholders

### Production Readiness Assessment

The deliverable (analysis document) is **complete and ready for review**. Since this is a documentation artifact rather than a code change, "production readiness" means the document accurately reflects the source code behavior and provides actionable insights. All code citations have been verified, all conclusions are evidence-based, and the document structure follows the SWE-AtlasQnA-Repo specification. The remaining 6 hours of work are exclusively peer review and minor editorial corrections.

---

## 9. Development Guide

### System Prerequisites

| Software | Version | Purpose |
|----------|---------|---------|
| Python | ^3.10 (tested with 3.10.20) | Runtime environment |
| Poetry | Latest | Dependency management |
| PostgreSQL | 12+ | Database (for running tests) |
| Redis | 6+ | Caching layer (for running tests) |
| Git | 2.30+ | Version control |

### Environment Setup

```bash
# 1. Clone the repository and switch to the feature branch
git clone <repository-url>
cd <repository-root>
git checkout blitzy-3e0bc84d-cc97-4cb4-a2be-f0ae22a1b5fe

# 2. Install Poetry if not present
curl -sSL https://install.python-poetry.org | python3 -

# 3. Install Python dependencies via Poetry
poetry install --no-interaction --no-ansi

# 4. Activate the Poetry virtual environment
poetry shell
# Or use: source $(poetry env info --path)/bin/activate
```

### Viewing the Analysis Document

```bash
# The deliverable is a markdown file — view directly
cat blitzy/documentation/app_2cd6ee777f8c.md

# Or view with a markdown renderer
# (e.g., on GitHub, the file renders automatically in the PR)

# Verify the file exists and has expected content
wc -l blitzy/documentation/app_2cd6ee777f8c.md
# Expected output: 778 blitzy/documentation/app_2cd6ee777f8c.md
```

### Verifying Repository Integrity

```bash
# Confirm no source files were modified
git diff origin/app_2cd6ee777f8c --name-status
# Expected output: A  blitzy/documentation/app_2cd6ee777f8c.md

# Confirm only 1 file was added
git diff origin/app_2cd6ee777f8c --stat
# Expected: blitzy/documentation/app_2cd6ee777f8c.md | 778 +++...
#           1 file changed, 778 insertions(+)

# Confirm git working tree is clean
git status
# Expected: "nothing to commit" (dump.rdb may appear as untracked)
```

### Running the Test Suite (Optional)

To verify the existing test suite passes without degradation:

```bash
# Ensure PostgreSQL is running on port 15432 with test/test credentials
# Ensure Redis is running on default port

# Activate Poetry environment
source $(poetry env info --path)/bin/activate

# Run the test suite
python -m pytest tests/ -x -q --tb=short --timeout=300

# Expected: 634 passed, 5 failed
# The 5 failures are in tests/test_mail_sender.py due to IPv6 loopback
# unavailability — these are environment-specific, not code issues
```

### Verifying Key Dependencies

```bash
# Activate Poetry environment first, then verify key packages
python -c "import sqlalchemy; print('SQLAlchemy', sqlalchemy.__version__)"
# Expected: SQLAlchemy 1.3.24

python -c "import flask; print('Flask', flask.__version__)"
# Expected: Flask 1.1.2

python -c "import aiosmtpd; print('aiosmtpd OK')"
# Expected: aiosmtpd OK
```

### Troubleshooting

| Issue | Resolution |
|-------|-----------|
| `ModuleNotFoundError: No module named 'newrelic'` | Ensure you're using the Poetry virtual environment, not the system Python. Run `poetry shell` first. |
| `KeyError: 'URL'` when importing `email_handler` | The `email_handler` module requires environment variables from `tests/test.env` or `.env`. This is expected — the analysis document does not require running the email handler. |
| PostgreSQL connection refused on port 15432 | PostgreSQL must be running on port 15432 with user `test`, password `test`, database `test` (as specified in `tests/test.env`). Only needed for running tests. |
| IPv6 test failures in `test_mail_sender.py` | These 5 failures are due to IPv6 `::1` not being available on the loopback interface. They are pre-existing environment-specific issues unrelated to this project. |

---

## 10. Appendices

### A. Command Reference

| Command | Purpose |
|---------|---------|
| `poetry install --no-interaction` | Install all Python dependencies |
| `poetry shell` | Activate the virtual environment |
| `python -m pytest tests/ -x -q --tb=short` | Run the test suite |
| `git diff origin/app_2cd6ee777f8c --name-status` | Verify only the analysis document was added |
| `wc -l blitzy/documentation/app_2cd6ee777f8c.md` | Verify document line count (expected: 778) |

### B. Port Reference

| Port | Service | Usage Context |
|------|---------|--------------|
| 15432 | PostgreSQL (test) | Test database — required only for running `pytest` |
| 6379 | Redis | Cache layer — required for test suite and email handler |
| 7777 | Gunicorn (web app) | Production web server — not used in this investigation |

### C. Key File Locations

| File | Purpose |
|------|---------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **Deliverable** — Comprehensive analysis document (778 lines) |
| `email_handler.py` | Central SMTP inbound processor; contains `handle_reply()`, `handle_forward()`, `handle()` |
| `app/contact_utils.py` | Contact creation/reconciliation workflow |
| `app/email_utils.py` | Reply email generation, reverse-alias identification |
| `app/email_validation.py` | Reply email normalization |
| `app/models.py` | ORM model definitions; `Contact` model with `reply_email` column |
| `app/db.py` | Database engine and session management |
| `tests/test.env` | Test environment configuration |
| `example.env` | Environment variable documentation |
| `pyproject.toml` | Dependency manifest (Poetry) |
| `migrations/versions/2021_071310_78403c7b8089_.py` | Migration creating `ix_contact_reply_email` index with `unique=False` |

### D. Technology Versions

| Technology | Version | Source |
|------------|---------|--------|
| Python | ^3.10 (tested 3.10.20) | `pyproject.toml` |
| SQLAlchemy | 1.3.24 (pinned) | `pyproject.toml` |
| Flask | ^1.1.2 (tested 1.1.2) | `pyproject.toml` |
| aiosmtpd | ^1.2 | `pyproject.toml` |
| psycopg2-binary | ^2.9.3 | `pyproject.toml` |
| PostgreSQL | 12+ | `tests/test.env` |
| Poetry | Latest | Build tool |

### E. Environment Variable Reference

| Variable | Example Value | Purpose |
|----------|---------------|---------|
| `URL` | `http://localhost` | Server URL (used by email handler and web app) |
| `EMAIL_DOMAIN` | `sl.local` | Primary email domain for alias and reverse-alias generation |
| `DB_URI` | `postgresql://test:test@localhost:15432/test` | PostgreSQL connection string |
| `FLASK_SECRET` | `secret` | Flask session secret key |
| `NOT_SEND_EMAIL` | `true` | Prevents actual email sending (for development/testing) |
| `DKIM_PRIVATE_KEY_PATH` | `local_data/dkim.key` | Path to DKIM signing key |

### G. Glossary

| Term | Definition |
|------|-----------|
| **Reply email / Reverse alias** | A system-generated email address (e.g., `sender_abc123@simplelogin.co`) stored on a `Contact` record, used to route replies from an alias owner back to the original sender |
| **Forward phase** | The processing path when an external email arrives for an alias; creates `Contact` records with generated `reply_email` values |
| **Reply phase** | The processing path when an alias owner replies to a reverse alias; resolves the `Contact` and delivers to the original sender |
| **TOCTOU** | Time-of-Check-to-Time-of-Use — a class of race condition where a check and subsequent action are not atomic |
| **`uq_contact`** | The unique constraint on `(alias_id, website_email)` in the `contact` table — the only uniqueness enforcement on contacts |
| **`ix_contact_reply_email`** | The non-unique B-tree index on `contact.reply_email` — used for query performance only, not data integrity |
| **`ModelMixin.get_by()`** | A generic ORM method that performs `Session.query(cls).filter_by(**kw).first()` — returns the first matching row or `None` |
| **Scoped session** | SQLAlchemy's `scoped_session` provides thread-local session proxies bound to a shared database connection |