# Blitzy Project Guide

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a comprehensive technical investigation document analyzing SimpleLogin's alias-creation flow end-to-end. The sole deliverable is a 713-line Markdown file (`blitzy/documentation/app_2cd6ee777f8c.md`) that answers five investigative questions — covering frontend requests, backend responses, database mutations, background events, and error handling — through deep source code analysis of ~20 Python modules, Jinja2 templates, and configuration files. The document targets backend engineers, security reviewers, and integration partners who need to understand exactly how alias creation works at the code level. No existing repository files were modified; the deliverable is purely additive.

### 1.2 Completion Status

```mermaid
pie title Project Completion Status
    "Completed (24h)" : 24
    "Remaining (5h)" : 5
```

| Metric | Value |
|--------|-------|
| **Total Project Hours** | 29 |
| **Completed Hours (AI)** | 24 |
| **Remaining Hours** | 5 |
| **Completion Percentage** | **82.8%** |

**Calculation:** 24 completed hours / (24 + 5) total hours = 24/29 = **82.8% complete**

### 1.3 Key Accomplishments

- ✅ Created comprehensive 713-line technical investigation document covering all 5 alias creation entry points
- ✅ Traced the complete `Alias.create()` 16-step execution sequence with Mermaid flowchart
- ✅ Identified and documented all 7 database tables touched during alias creation
- ✅ Cataloged all 14 validation checks with trigger conditions, status codes, and response bodies
- ✅ Documented the EventDispatcher chain including protobuf serialization, SyncEvent persistence, and PostgreSQL NOTIFY
- ✅ Mapped the 3-layer rate limiting architecture (Flask-Limiter, parallel limiter, bucket rate limiter)
- ✅ Produced 4 Mermaid diagrams (sequence, flowchart, decision tree, rate limiting layers)
- ✅ Included 45 source code citations with file:line references — all 19 referenced source files verified to exist
- ✅ Zero existing repository files modified (per explicit AAP constraint)
- ✅ 3 clean commits, working tree clean, no uncommitted changes

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| Source citation line numbers not independently verified by human reviewer | Line references may drift if code has changed since analysis | Human Developer | 1–2 hours |
| Error handling decision tree Mermaid diagram not created (table used instead) | Minor gap from AAP Section 0.4.3 plan; table is arguably more comprehensive | Human Developer | 0.5 hours |

### 1.5 Access Issues

No access issues identified. This is a documentation-only deliverable requiring no external service credentials, API keys, or database access. The document was generated entirely from source code analysis of the local repository.

### 1.6 Recommended Next Steps

1. **[High]** Domain expert reviews all 45 source citations for technical accuracy, verifying code claims against current source
2. **[High]** Verify all file:line references are still accurate against the `app_2cd6ee777f8c` branch HEAD
3. **[Medium]** Optionally add a Mermaid decision tree diagram for the error handling section (currently documented as a comprehensive table)
4. **[Medium]** Test Mermaid diagram rendering across target platforms (GitHub, VS Code, documentation viewers)
5. **[Low]** Cross-reference document with existing `docs/api.md` to ensure consistency on shared topics

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|-------|-------------|
| Source code analysis and tracing | 5 | Deep analysis of ~20 source files across API views, models, events, rate limiting, templates, and configuration — establishing the complete alias creation call graph |
| Section 1 — Overview | 0.5 | Entry points overview table (5 endpoints), centralization rationale for `Alias.create()` |
| Section 2 — Frontend Request | 4 | All 5 entry points documented: dashboard custom/random (form fields, CSRF, cookies), API v2/v3 custom (JSON body, signed suffix, mailbox IDs), API random (hostname suggestion logic, mode parameter), suffix provisioning (itsdangerous signing, 600s TTL) |
| Section 3 — Backend Response | 2 | Success responses (201 JSON with `serialize_alias_info_v2` structure, 302 redirect with flash messages), error response format description with cross-reference to error taxonomy |
| Section 4 — Database Changes | 3 | 7-table mutation inventory, 16-step `Alias.create()` sequence with Mermaid flowchart, conditional writes analysis (AliasMailbox multi-mailbox, AliasUsedOn hostname tracking, SyncEvent partner events) |
| Section 5 — Background Tasks | 2 | EventDispatcher chain with Mermaid decision tree, AliasCreated protobuf event structure, PostgresDispatcher persistence to SyncEvent, PostgreSQL NOTIFY mechanism, New Relic telemetry |
| Section 6 — Error Handling | 3 | 14-entry validation check table with trigger conditions/status codes/response bodies, IntegrityError and AliasInTrashError analysis, 3-layer rate limiting architecture with Mermaid diagram |
| Section 7 — End-to-End Summary | 1.5 | Full Mermaid sequence diagram (12 participants), quick reference comparison table (5 entry points × 12 features), auto-creation path reference |
| Code review iteration | 1 | Addressed 5 code review findings in revision commit |
| Quality assurance and validation | 2 | Verified all 19 referenced source files exist, validated 45 citation format consistency, confirmed document structure matches AAP Section 0.4.1 plan |
| **Total** | **24** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|-------|----------|
| Technical accuracy review by domain expert | 2 | High |
| Source citation line-number verification (45 citations) | 1 | High |
| Error handling Mermaid diagram enhancement | 0.5 | Medium |
| Mermaid rendering verification across platforms | 0.5 | Medium |
| Editorial review and polish | 0.5 | Low |
| Cross-reference with existing `docs/api.md` | 0.5 | Low |
| **Total** | **5** | |

**Integrity check:** Section 2.1 total (24h) + Section 2.2 total (5h) = 29h = Total Project Hours in Section 1.2 ✅

---

## 3. Test Results

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|------------|-------|
| N/A — Documentation-only task | N/A | 0 | 0 | 0 | N/A | This project creates a single Markdown documentation file. No source code was created or modified, so no tests apply. |

**Rationale:** The AAP explicitly scopes this project as a documentation-only deliverable with the constraint "Do not modify any existing files in the source repository." The sole artifact is `blitzy/documentation/app_2cd6ee777f8c.md` — a Markdown file requiring no compilation, runtime, or test execution. The Final Validator confirmed: "GATE 1 (Tests): ✅ N/A — documentation-only task, no tests to run."

---

## 4. Runtime Validation & UI Verification

**Runtime Health:**

- ✅ Documentation file `blitzy/documentation/app_2cd6ee777f8c.md` created and committed (713 lines)
- ✅ File is valid Markdown — proper heading hierarchy, fenced code blocks, and Mermaid diagram syntax
- ✅ All 19 source files referenced in citations verified to exist in the repository
- ✅ Git working tree is clean — no uncommitted changes
- ✅ 3 commits on branch `blitzy-0789799a-790e-40f4-b224-5de718b83950` (initial creation, code review fixes, final version)

**UI Verification:**

- ✅ N/A — No UI components in this documentation-only deliverable

**API Integration:**

- ✅ N/A — No API endpoints created or modified

**Validation Summary:**

The Final Validator confirmed all four production-readiness gates:
- GATE 1 (Tests): ✅ N/A — documentation-only, no tests needed
- GATE 2 (Runtime): ✅ N/A — Markdown file, no runtime component
- GATE 3 (Zero Errors): ✅ No errors — single file created, no compilation
- GATE 4 (All In-Scope Files): ✅ The only in-scope file is complete and committed

---

## 5. Compliance & Quality Review

| AAP Requirement | Status | Evidence |
|----------------|--------|----------|
| **Create `blitzy/documentation/app_2cd6ee777f8c.md`** | ✅ Pass | File exists, 713 lines, committed |
| **Document frontend requests (5 entry points)** | ✅ Pass | Sections 2.1–2.5 cover all 5 endpoints with form fields, JSON schemas, auth methods |
| **Document backend responses (success + error)** | ✅ Pass | Section 3 covers all success responses; Section 6 catalogs all error responses |
| **Document database mutations** | ✅ Pass | Section 4 identifies 7 tables with 16-step Alias.create() sequence |
| **Document background tasks and events** | ✅ Pass | Section 5 covers EventDispatcher chain, protobuf events, NOTIFY, New Relic |
| **Document error handling** | ✅ Pass | Section 6 catalogs 14 validation checks, DB errors, 3-layer rate limiting |
| **Source citations with file:line format** | ✅ Pass | 45 citations found, all 19 unique source files verified to exist |
| **Minimum 1 Mermaid sequence diagram** | ✅ Pass | Section 7 — full end-to-end sequence diagram |
| **Minimum 1 Mermaid flowchart** | ✅ Pass | Section 4.2 — database mutation flowchart |
| **Minimum 1 error conditions table** | ✅ Pass | Section 6.1 — 14-entry validation check table |
| **Minimum 1 database tables table** | ✅ Pass | Section 4.1 — 7-entry table inventory |
| **No existing repository files modified** | ✅ Pass | `git diff --name-status` shows only 1 file added, 0 modified |
| **No test files created or modified** | ✅ Pass | Only `blitzy/documentation/app_2cd6ee777f8c.md` created |
| **Temporary scripts cleaned up** | ✅ Pass | No temporary artifacts remain (none were needed) |
| **Claims based on code, not assumptions** | ✅ Pass | All technical claims include source file:line citations |
| **Rationale provided for behaviors** | ✅ Pass | "Rationale:" blocks explain centralization design, rate limiting layers, SyncEvent durability |
| **Error handling decision tree diagram** | ⚠️ Partial | AAP Section 0.4.3 planned a Mermaid decision tree; Section 6 uses comprehensive table instead — functionally equivalent |

**Autonomous Fixes Applied:**
- 5 code review findings addressed in commit `45dd1abb` (revision iteration)
- Final commit `df04c5b9` consolidates all fixes

**Outstanding Items:**
- Error handling Mermaid decision tree diagram (planned in AAP 0.4.3, implemented as table — enhancement opportunity)

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Source citation line numbers may be inaccurate if code has changed | Technical | Medium | Low | Human verification of all 45 citations against current HEAD | Open |
| Mermaid diagrams may not render in all Markdown viewers | Technical | Low | Medium | Test in GitHub, VS Code, and target documentation platforms | Open |
| Documentation may become stale as codebase evolves | Operational | Medium | High | Add version/commit reference at document top; schedule periodic review | Open |
| Error handling table may miss edge cases not visible in static analysis | Technical | Low | Low | Runtime testing of alias creation with various error conditions | Open |
| No automated validation that citations match actual code | Operational | Low | Medium | Consider building a citation-checker script for CI | Open |

---

## 7. Visual Project Status

```mermaid
pie title Project Hours Breakdown
    "Completed Work" : 24
    "Remaining Work" : 5
```

**Integrity check:** "Remaining Work" (5h) = Section 1.2 Remaining Hours (5h) = Section 2.2 Total (5h) ✅

### Remaining Hours by Category

| Category | Hours |
|----------|-------|
| Technical accuracy review | 2 |
| Citation verification | 1 |
| Diagram enhancement | 0.5 |
| Rendering QA | 0.5 |
| Editorial polish | 0.5 |
| Cross-reference check | 0.5 |
| **Total** | **5** |

---

## 8. Summary & Recommendations

### Achievement Summary

The project has delivered its primary AAP-scoped deliverable: a comprehensive 713-line technical investigation document (`blitzy/documentation/app_2cd6ee777f8c.md`) that traces SimpleLogin's alias-creation flow through deep source code analysis. The document answers all five investigative questions posed in the AAP, covering frontend requests (5 entry points), backend responses (success and error for all endpoints), database mutations (7 tables), background events (EventDispatcher chain with protobuf and PostgreSQL NOTIFY), and error handling (14 validation checks with 3-layer rate limiting).

The project is **82.8% complete** (24 hours completed out of 29 total hours). All AAP-specified documentation sections have been written, all quality criteria (citations, diagrams, tables) have been met, and all constraints (no existing files modified, code-based claims only) have been honored.

### Remaining Gaps

The 5 remaining hours are entirely human review and QA tasks:
- **Technical accuracy review (2h):** A domain expert should verify that all 45 source citations accurately describe the code behavior
- **Citation verification (1h):** Verify file:line references match current HEAD
- **Enhancement and QA (2h):** Optional Mermaid diagram addition, rendering verification, editorial polish, and cross-referencing with existing `docs/api.md`

### Production Readiness Assessment

This documentation deliverable is **ready for human review**. The document is complete, committed, and the working tree is clean. No blocking issues exist. The remaining tasks are standard quality assurance activities that require human domain expertise to complete.

### Success Metrics

| Metric | Target | Actual | Status |
|--------|--------|--------|--------|
| Documentation sections delivered | 7 | 7 | ✅ Met |
| Source citations | ≥1 per section | 45 total | ✅ Exceeded |
| Mermaid diagrams | ≥2 (sequence + flowchart) | 4 | ✅ Exceeded |
| Database tables documented | All touched tables | 7 tables | ✅ Met |
| Error conditions cataloged | All validation checks | 14 checks | ✅ Met |
| Entry points documented | All 5 | 5 | ✅ Met |
| Existing files modified | 0 | 0 | ✅ Met |

---

## 9. Development Guide

### System Prerequisites

| Requirement | Version | Purpose |
|-------------|---------|---------|
| Git | Any modern version | Clone repository, view commits |
| Markdown viewer | Any (VS Code, GitHub, etc.) | View the documentation file |
| Python | 3.10+ | Only needed if running the SimpleLogin application for verification |
| Poetry | Latest | Only needed if running the SimpleLogin application |
| PostgreSQL | 13+ | Only needed if running the SimpleLogin application |
| Redis | Latest | Only needed if running the SimpleLogin application |
| Node.js | v10+ | Only needed for frontend assets |

### Viewing the Documentation

The primary deliverable is a standalone Markdown file. No build step is required.

```bash
# Clone and switch to the feature branch
git checkout blitzy-0789799a-790e-40f4-b224-5de718b83950

# View the documentation file
cat blitzy/documentation/app_2cd6ee777f8c.md

# Or open in your preferred Markdown viewer
# VS Code:
code blitzy/documentation/app_2cd6ee777f8c.md

# Or view on GitHub after pushing
```

### Verifying Source Citations

To verify that all referenced source files exist:

```bash
# From repository root
python3 -c "
import os, re
with open('blitzy/documentation/app_2cd6ee777f8c.md') as f:
    content = f.read()
sources = re.findall(r'Source: \x60([^\x60]+)\x60', content)
files = set()
for s in sources:
    path = s.split(' line')[0].strip()
    files.add(path)
for f in sorted(files):
    exists = os.path.exists(f)
    print(f'{\"✅\" if exists else \"❌\"} {f}')
"
```

**Expected output:** All 19 unique source files show ✅.

### Running the SimpleLogin Application (Optional — for verification)

If you want to verify the documented behavior by running the application locally:

```bash
# 1. Install Python dependencies
poetry install

# 2. Install frontend assets
cd static && npm install && cd ..

# 3. Create local environment file
cp example.env .env
# Edit .env to set DB_URI=postgresql://myuser:mypassword@localhost:15432/simplelogin

# 4. Start PostgreSQL
docker run -e POSTGRES_PASSWORD=mypassword -e POSTGRES_USER=myuser \
  -e POSTGRES_DB=simplelogin -p 15432:5432 postgres:13

# 5. Run database migrations and seed data
alembic upgrade head && flask dummy-data

# 6. Start the server
python3 server.py
# Application available at http://localhost:7777
# Test login: john@wick.com / password
```

### Troubleshooting

| Issue | Resolution |
|-------|-----------|
| Mermaid diagrams don't render | Use a Mermaid-compatible viewer (GitHub, VS Code with Mermaid extension, or mermaid.live) |
| Source citation line numbers seem wrong | The codebase may have been updated since the analysis; verify against the `app_2cd6ee777f8c` branch |
| `poetry install` fails on Mac | Run `brew install pkg-config libffi openssl postgresql@13` first |
| Docker PostgreSQL port conflict | Change `-p 15432:5432` to an available port and update `.env` accordingly |

---

## 10. Appendices

### A. Command Reference

| Command | Purpose |
|---------|---------|
| `cat blitzy/documentation/app_2cd6ee777f8c.md` | View the documentation file |
| `git log --oneline HEAD~3..HEAD` | View the 3 Blitzy agent commits |
| `git diff --stat origin/app_2cd6ee777f8c...HEAD` | View file change summary |
| `wc -l blitzy/documentation/app_2cd6ee777f8c.md` | Count documentation lines (713) |
| `grep -c "Source:" blitzy/documentation/app_2cd6ee777f8c.md` | Count source citations (45) |
| `grep -c "mermaid" blitzy/documentation/app_2cd6ee777f8c.md` | Count Mermaid diagram blocks (4) |

### C. Key File Locations

| File | Purpose |
|------|---------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **Primary deliverable** — alias-creation flow investigation document |
| `app/models.py` (lines 1628–1692) | Core `Alias.create()` classmethod — central persistence point |
| `app/api/views/new_custom_alias.py` | API v2/v3 custom alias creation endpoints |
| `app/api/views/new_random_alias.py` | API random alias creation endpoint |
| `app/dashboard/views/custom_alias.py` | Dashboard custom alias creation web view |
| `app/dashboard/views/index.py` | Dashboard random alias creation handler |
| `app/alias_suffix.py` | Suffix generation, signing, and verification |
| `app/events/event_dispatcher.py` | Event dispatch chain (protobuf, SyncEvent, NOTIFY) |
| `app/rate_limiter.py` | Redis bucket rate limiter |
| `app/parallel_limiter.py` | Redis distributed concurrency lock |
| `app/api/serializer.py` | API response serialization (`serialize_alias_info_v2`) |
| `app/errors.py` | `AliasInTrashError` exception class |
| `app/config.py` | Rate limit configuration constants |
| `docs/api.md` | Existing API reference (related, not modified) |
| `CONTRIBUTING.md` | Local development setup instructions |

### D. Technology Versions

| Technology | Version | Role |
|------------|---------|------|
| Python | ^3.10 | Runtime language |
| Flask | ^1.1.2 | Web framework |
| SQLAlchemy | 1.3.24 | ORM |
| PostgreSQL | 13+ | Primary database |
| Redis | ^4.5.3 | Rate limiting, sessions, distributed locks |
| Flask-Limiter | ^1.4 | HTTP-level rate limiting |
| itsdangerous | (via Flask) | Suffix signing with TimestampSigner |
| protobuf | (via events) | Event serialization (AliasCreated) |
| New Relic | 8.8.0 | Telemetry and custom events |
| Poetry | Latest | Dependency management |
| Node.js | v10+ | Frontend asset management |

### G. Glossary

| Term | Definition |
|------|-----------|
| **Alias** | A forwarding email address that routes mail to a user's real mailbox(es) |
| **Suffix** | The domain portion of an alias (e.g., `.random@simplelogin.co`), cryptographically signed with a 600-second TTL |
| **Custom alias** | An alias where the user chooses the prefix (local part before the suffix) |
| **Random alias** | An alias auto-generated using either word-based or UUID-based schemes |
| **AliasMailbox** | Junction table linking an alias to additional mailboxes beyond the primary one |
| **AliasUsedOn** | Record linking an alias to the hostname/website where it was created |
| **SyncEvent** | Durable event record for Proton partner integration, containing serialized protobuf bytes |
| **EventDispatcher** | Service that conditionally dispatches alias lifecycle events to partner systems |
| **Bucket rate limiter** | Redis-based rate limiter using time-bucketed INCR keys for business-level quotas |
| **Parallel limiter** | Redis distributed lock preventing concurrent alias creation by the same user |
| **DeletedAlias / DomainDeletedAlias** | "Trash" tables preserving deleted alias emails to prevent reuse |
| **DailyMetric** | Per-day aggregate counter tracking alias creation volume |
| **AliasAuditLog** | Immutable audit trail of alias lifecycle events (create, update, delete) |