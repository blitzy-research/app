# Blitzy Project Guide

## Section 1 — Executive Summary

### 1.1 Project Overview

This project is a **read-only, run-first codebase investigation** of SimpleLogin (an open-source email-aliasing application). The sole deliverable is one evidence-based Markdown document, `blitzy/documentation/app_2cd6ee777f8c.md`, that explains — from observed runtime behavior — how an inbound email *reply* resolves to a specific `Contact` and its forwarding destination (owning user, alias, delivery mailbox), and **why a reply can be delivered to the wrong user**. The audience is SimpleLogin maintainers and reviewers. Business impact: it root-causes a latent cross-user mail-routing defect (a non-unique `reply_email` column combined with an unordered `LIMIT 1` lookup) with reproducible evidence, enabling an informed remediation decision. No application code was changed.

### 1.2 Completion Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#111111','pieLegendTextColor':'#111111'}}}%%
pie showData title Completion — 90.9% Complete
    "Completed Work (AI)" : 30
    "Remaining Work" : 3
```

| Metric | Value |
|--------|-------|
| **Total Hours** | 33.0 h |
| **Completed Hours (AI + Manual)** | 30.0 h (30.0 AI + 0.0 Manual) |
| **Remaining Hours** | 3.0 h |
| **Percent Complete** | **90.9%** |

Completion is computed with the AAP-scoped, hours-based PA1 methodology: `Completed / (Completed + Remaining) × 100 = 30 / 33 = 90.9%`. The figure is capped below 100% because human review and acceptance remain outstanding.

### 1.3 Key Accomplishments

- ✅ Established the canonical runtime (Docker container `sl-work`, Python 3.10.18, SQLAlchemy 1.3.24, psycopg2 2.9.3, aiosmtpd 1.4.2, PostgreSQL 15 on port 15432) and booted the app via `server.create_app()` with `CONFIG=tests/test.env`.
- ✅ Reproduced all seven investigation requirements (R1–R7) at runtime through the **real** entry point `email_handler.handle()` → `handle_reply()`, not a bypass.
- ✅ Captured the root cause with primary evidence: emitted SQL is an unordered `LIMIT 1` (no `ORDER BY`), confirmed by both the SQLAlchemy echo and `EXPLAIN`.
- ✅ Reproduced the reported run-to-run inconsistency ("sometimes wrong user") by re-running the same unchanged input and observing the resolved user flip across a heap reorder and across three independent OS processes.
- ✅ Exercised all eight guarded exits of the reply handler (E501, E502×2, E503, E504, E214 canonical; E201, E506 forced and clearly labeled non-canonical).
- ✅ Delivered a 789-line, fully grounded (`file:line`) answer document with verbatim, unedited command output for every claim.
- ✅ Left the repository unchanged apart from the single new document (net diff = one added file); all temporary observation scripts removed.
- ✅ Independent final validation across 13 phases reproduced every claim with zero discrepancies and passed all five production-readiness gates.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| None blocking the deliverable | The investigation document is complete, validated, and repository-clean. No issue blocks release of the document. | — | — |
| (Informational, out of AAP scope) Underlying wrong-user routing defect in SimpleLogin remains unfixed | Product-level correctness/security risk in SimpleLogin itself; explicitly **not** in scope for this documentation task (AAP §0.5.2) | SimpleLogin maintainers | Downstream ticket |

### 1.5 Access Issues

No access issues identified. The repository, the canonical Docker image (`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`), PostgreSQL (port 15432, role `test`/`test`), and Redis were all reachable during the autonomous investigation and validation. The deliverable requires no third-party credentials or external service access.

### 1.6 Recommended Next Steps

1. **[High]** Perform a human technical review of `blitzy/documentation/app_2cd6ee777f8c.md`, focusing on the root-cause reasoning and the R1–R7 evidence.
2. **[High]** Independently spot-check a sample of `file:line` citations against source base `2cd6ee77`; optionally re-run the reproduction in the canonical Docker environment.
3. **[Medium]** Obtain stakeholder acceptance and merge/publish the single-file pull request.
4. **[Low]** (Informational, out of AAP scope) Open a downstream remediation ticket for the wrong-user routing defect and route the cross-user exposure finding to security triage.

---

## Section 2 — Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|-------|-------------|
| Canonical runtime establishment | 3.0 | Boot the app in the canonical Docker env (PostgreSQL:15432, Redis, Poetry deps) via `server.create_app()` with `CONFIG=tests/test.env` (`EMAIL_DOMAIN=sl.local`). |
| Source-code investigation & citation grounding | 4.0 | Read and anchor the 7 reference files (`email_handler.py`, `app/models.py`, `app/email_validation.py`, `app/email_utils.py`, `app/contact_utils.py`, `app/email/status.py`, the reply_email migration) for exact `file:line` grounding. |
| Observation-harness authoring | 6.0 | Write temporary run-first scripts: seed `User→Mailbox→Alias→Contact` and drive the real `handle()`→`handle_reply()`; an 8-exit matrix harness; a cross-process resolution harness. |
| Runtime observation & evidence capture | 5.0 | Execute the harnesses and capture verbatim output for R1–R7, including emitted SQL + `EXPLAIN`, the cross-process user flip, the TOCTOU dual-insert, and before/during/after state transitions. |
| Answer-document authoring | 6.0 | Compose the 789-line document: TL;DR, environment, R1–R7, guarded-exit matrix, state-transition report, root-cause synthesis, coverage pass, and repository verification. |
| QA refinement across 4 commits | 2.0 | Address code-review findings, fix the R1 `normalize_reply_email` case-handling nuance, and make the HEAD/status proof reproducible. |
| Web-search corroboration | 1.0 | Validate the two DB mechanisms (PostgreSQL unordered-`LIMIT` non-determinism; check-then-insert TOCTOU races) against authoritative references as secondary support. |
| Independent final validation | 3.0 | 13-phase runtime reproduction of R1–R7 + 8 guarded exits + state transitions; 100% citation grounding; 5 production-readiness gates; cleanup verification. |
| **Total Completed** | **30.0** | Sum of all completed components. |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|-------|----------|
| Human technical review of the investigation document (reasoning + root-cause soundness) | 1.5 | High |
| Independent citation/claim spot-check & optional reproduction re-run in the canonical env | 1.0 | High |
| Stakeholder acceptance & PR merge/publish | 0.5 | Medium |
| **Total Remaining** | **3.0** | — |

### 2.3 Hours Summary

| Bucket | Hours |
|--------|-------|
| Completed (Section 2.1) | 30.0 |
| Remaining (Section 2.2) | 3.0 |
| **Total Project Hours** | **33.0** |

Integrity: Section 2.1 (30.0) + Section 2.2 (3.0) = 33.0 = Total Hours in Section 1.2. Remaining hours (3.0) are identical in Sections 1.2, 2.2, and 7.

> **Scope note:** Remediation of the underlying SimpleLogin defect (e.g., a `UNIQUE` constraint on `contact.reply_email`, a deterministic `ORDER BY`/row-locking in the lookup, or contact de-duplication) is **out of scope** per AAP §0.5.2 and is therefore **excluded** from the hour totals above. It appears only as an informational downstream recommendation in Sections 1.6 and 8.

---

## Section 3 — Test Results

For this read-only documentation task, no unit-test suite was added to the repository (adding code is out of scope). The "tests" below are **Blitzy's autonomous run-first reproduction observations** — each drives the real `email_handler.handle()` → `handle_reply()` entry point and asserts the documented behavior against captured runtime output. All entries originate from Blitzy's autonomous validation logs for this project. "Coverage %" denotes AAP **condition/requirement** coverage, not source line coverage (no application code was added or modified).

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|------------|-------|
| Requirement reproduction (R1–R7) | Run-first harness: `server.create_app()` + real `handle()`/`handle_reply()`, SQLAlchemy 1.3.24, aiosmtpd 1.4.2, PostgreSQL 15 | 7 | 7 | 0 | 100% | Each requirement reproduced with verbatim output; zero discrepancies |
| Guarded-exit matrix | Same harness (production `handle_reply`) | 8 | 8 | 0 | 100% | 6 canonical (E501, E502×2, E503, E504, E214) + 2 forced & labeled NON-CANONICAL (E201, E506) |
| Multi-event stability (R4) | Same harness | 13 | 13 | 0 | 100% | 3 full `handle()` events (distinct sequential `EmailLog`) + 10 unanimous `get_by` lookups |
| Divergence / distribution (R5a) | Same harness + 3 fresh OS processes + `psql` `EXPLAIN` | 13 | 13 | 0 | 100% | 10 in-process + 3 cross-process lookups all resolve to User B after heap reorder |
| State-transition capture | Same harness + heap-order mutation | 3 | 3 | 0 | 100% | before (User A) → after-insert (User A) → after-reorder (User B); input unchanged |
| Race / uniqueness (R6) | Same harness | 2 | 2 | 0 | 100% | TOCTOU dual-insert both succeed (proves `reply_email` non-unique); genuine `uq_contact` collision does raise |
| Citation grounding spot-checks | Source read vs. document | 7 | 7 | 0 | 100% | `get_by`, `uq_contact`, `reply_email` column, migration `unique=False`, status constants, `normalize_reply_email` |
| **Total** | — | **53** | **53** | **0** | **100%** | Zero discrepancies across all autonomous reproductions |

---

## Section 4 — Runtime Validation & UI Verification

**Runtime health (reproduction harness in the canonical environment):**

- ✅ **Operational** — Application boots via `server.create_app()` under `CONFIG=tests/test.env`.
- ✅ **Operational** — PostgreSQL 15 reachable on port 15432 (role `test`/`test`); alembic at head; 77 tables.
- ✅ **Operational** — Redis responds `PONG`.
- ✅ **Operational** — Real entry point `email_handler.handle()` → `handle_reply()` executes end-to-end across all conditions.
- ✅ **Operational** — Happy-path reply returns `250 Message accepted for delivery`; all canonical guarded exits return their exact `app/email/status.py` constants.
- ⚠ **Partial** — `E201` (SPF) and `E506` (spam) reproduce **only when forced**; `ENFORCE_SPF` and `ENABLE_SPAM_ASSASSIN` default to off in the canonical config. Both are clearly labeled NON-CANONICAL in the document.

**UI verification:**

- **Not applicable** — the reply-resolution path under investigation is a headless SMTP flow with no user-interface surface (AAP §0.9). No screens, components, or Figma assets are in scope.

**Read-only / repository integrity:**

- ✅ **Operational** — Net diff versus source base `2cd6ee77` is exactly one added file (`A blitzy/documentation/app_2cd6ee777f8c.md`); working tree clean; no temporary scripts left in the repository.

---

## Section 5 — Compliance & Quality Review

The task is governed by the **SWE-AtlasQnA-Repo** rule set (AAP §0.7). Each mandate is cross-mapped to Blitzy's autonomous validation evidence below.

| Benchmark / Rule | Requirement | Status | Evidence / Notes |
|------------------|-------------|--------|------------------|
| Deliverable | One Markdown file at `blitzy/documentation/<branch>.md` | ✅ Pass | `app_2cd6ee777f8c.md` created (789 lines) |
| Run-first | Build & run before writing; answer from observation | ✅ Pass | All claims backed by executed harness output |
| Reproduce-inconsistency | Re-run same input; report distribution | ✅ Pass | User flip observed in-process and across 3 OS processes |
| Real-path | Exercise real entry point, not a bypass | ✅ Pass | Drives `handle()`→`handle_reply()` via crafted aiosmtpd `Envelope` |
| Default/canonical config | Run in default configuration; state commands | ✅ Pass | `CONFIG=tests/test.env`; forced branches labeled non-canonical |
| Exhaustive conditions | Enumerate & exercise every variant/exit | ✅ Pass | 8 guarded exits + happy path all reproduced |
| State-transition | Report before/during/after | ✅ Pass | 595 → 595 → 596 with unchanged input |
| Verbatim output + command | Complete unedited output per claim | ✅ Pass | Full logs incl. SL log lines and `psql` output preserved |
| Grounding | `file:line` per factual claim; label inferred | ✅ Pass | Independently spot-checked; 100% accurate |
| Coverage-pass | Confirm every named sub-question answered | ✅ Pass | Coverage-pass table maps every R + named item |
| Read-only scope | No source modified; temp scripts removed | ✅ Pass | Net diff = 1 added file; tree clean |
| Zero placeholders | No TODO/FIXME/stubs | ✅ Pass | Placeholder scan returns 0 |

**Fixes applied during autonomous validation:** R1 `normalize_reply_email` case-handling nuance corrected (commit `f26b4f24`); code-review findings addressed (commit `cd57ef00`); HEAD/status proof made reproducible (commit `38c3c41c`). The independent final validator required **no** further corrections — the document reproduced with zero discrepancies.

**Outstanding quality items:** none within scope. Six trailing-whitespace lines exist inside fenced verbatim tool-output blocks and were intentionally preserved to honor the verbatim-output mandate (they do not affect Markdown rendering).

---

## Section 6 — Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Concrete DB IDs are from a single captured run; re-runs show different IDs (PostgreSQL sequence advance) | Technical | Low | High | Document explicitly frames numbers as one captured run; the qualitative behavior is invariant | Mitigated |
| Documented wrong-user routing is a real latent SimpleLogin defect left unfixed (by design) | Technical | High | Medium | Out of scope per AAP §0.5.2; recommend downstream remediation ticket | Open (downstream) |
| Citation `file:line` drift if SimpleLogin source evolves past base `2cd6ee77` | Technical | Low | Low | Document pins the immutable source-base commit hash | Mitigated |
| Document surfaces a cross-user information-exposure mechanism | Security | High | Medium | Route finding to security triage as a downstream action | Open (downstream) |
| Secrets/credentials leakage in the document | Security | Low | Low | Only throwaway test-env values (`test`/`test`, `sl.local`) appear; verified clean | Mitigated |
| Reproduction depends on the canonical Docker image; host default shell (Python 3.13/no-PG) is non-canonical | Operational | Low | Medium | Exact image + commands stated; non-canonical shell explicitly labeled | Mitigated |
| Heap-order flip is non-deterministic; a naive re-run may not flip | Operational | Low | Medium | Document provides deterministic triggers (delete+reinsert; `EXPLAIN` with `enable_seqscan` off) + cross-process confirmation | Mitigated |
| Integration/runtime coupling of the deliverable | Integration | None | Low | Standalone Markdown artifact; no API keys, external services, or CI coupling | N/A |

---

## Section 7 — Visual Project Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#111111','pieLegendTextColor':'#111111'}}}%%
pie showData title Project Hours Breakdown
    "Completed Work" : 30
    "Remaining Work" : 3
```

**Remaining hours by category (Section 2.2):**

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'primaryColor':'#5B39F3','primaryTextColor':'#FFFFFF','primaryBorderColor':'#B23AF2','lineColor':'#B23AF2'}}}%%
xychart-beta
    title "Remaining Hours by Category"
    x-axis ["Technical review", "Citation spot-check", "Acceptance & merge"]
    y-axis "Hours" 0 --> 2
    bar [1.5, 1.0, 0.5]
```

Integrity: the pie chart's "Completed Work" (30) and "Remaining Work" (3) match the Section 1.2 metrics table and the Section 2.1/2.2 totals exactly. The bar chart's values sum to 3.0, equal to Remaining Hours.

---

## Section 8 — Summary & Recommendations

**Achievements.** The project delivers a complete, validated, evidence-based investigation document that answers all seven posed questions (R1–R7) from observed runtime behavior. It establishes — with primary runtime evidence (emitted SQL and `EXPLAIN`) — that `Contact.get_by(reply_email=...).first()` is an unordered `LIMIT 1` query, and that because the `reply_email` column is indexed but **not** unique, duplicate rows can exist and the row PostgreSQL returns first is not stable once physical heap order changes. The reported "sometimes wrong user" behavior was reproduced honestly (in-process and across three OS processes) rather than sanitized, and all eight guarded exits plus before/during/after state transitions were captured verbatim.

**Remaining gaps & critical path to production.** The project is **90.9% complete** (30 of 33 hours). The only remaining work is human-side path-to-production for a documentation deliverable: a technical review (1.5 h), an independent citation/claim spot-check with optional reproduction (1.0 h), and stakeholder acceptance & merge (0.5 h) — 3.0 hours total. There is no outstanding engineering work within the AAP scope.

**Production readiness.** The deliverable is production-ready pending human sign-off: it is fully grounded, free of placeholders, well-formed, and the repository is clean with a net diff of exactly one added file. Success metrics — 100% of AAP requirements completed, zero validation discrepancies, all five gates passed — are met.

**Downstream recommendations (informational, out of AAP scope).** Maintainers should open a remediation ticket for the underlying defect (candidate fixes: a `UNIQUE` constraint on `contact.reply_email`, a deterministic `ORDER BY`/row-locking in the lookup, or contact de-duplication) and route the cross-user exposure finding to security triage. These are explicitly **not** part of this documentation task and carry no hours here.

---

## Section 9 — Development Guide

### 9.1 System Prerequisites

- **Git** (to clone/inspect the branch and verify read-only state).
- **Docker** with the canonical image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (container name `sl-work`) — required to *reproduce* the investigation.
- **Python 3.10** (canonical: 3.10.18), **PostgreSQL 15** (port 15432, role `test`/`test`), **Redis** — all supplied by the canonical image.
- A Markdown viewer (any editor, or a browser with a Markdown/Mermaid renderer) to *read* the deliverable.

> The host's default shell may be Python 3.13 with no local PostgreSQL — that environment is **non-canonical** and will not reproduce the behavior. Always reproduce via `docker exec sl-work ...`.

### 9.2 Read & Review the Deliverable

```bash
# From the repository root
ls -la blitzy/documentation/app_2cd6ee777f8c.md
wc -l  blitzy/documentation/app_2cd6ee777f8c.md      # expect: 789 lines

# Read it (any pager/editor)
less blitzy/documentation/app_2cd6ee777f8c.md
```

Expected: an 85,196-byte, 789-line document with a TL;DR, environment section, R1–R7 answers, a guarded-exit matrix, a state-transition report, a root-cause synthesis, a coverage pass, and a repository-verification section.

### 9.3 Verify the Read-Only State

```bash
# Net change versus the SimpleLogin source base must be exactly one added file
git diff --name-status 2cd6ee777f8c...HEAD
# expect: A    blitzy/documentation/app_2cd6ee777f8c.md

# Working tree must be clean
git status --porcelain            # expect: no output

# No temporary observation scripts must remain in the repo tree
find . -type f \( -name 'obs_*.py' -o -name 'tmp_*.py' -o -name '*_observation*.py' \) -not -path './.git/*'
# expect: no output
```

### 9.4 Document Well-Formedness Checks

```bash
# Code fences must be balanced (even count)
grep -c '^```' blitzy/documentation/app_2cd6ee777f8c.md        # expect: 68 (even)

# No placeholders/TODOs
grep -cE 'TODO|FIXME|XXX|PLACEHOLDER|coming soon|TBD' blitzy/documentation/app_2cd6ee777f8c.md   # expect: 0

# Heading hierarchy overview
grep -nE '^#{1,4} ' blitzy/documentation/app_2cd6ee777f8c.md   # 1 H1 + 15 H2 + 6 H3 + 2 H4
```

### 9.5 Reproduce the Investigation (canonical environment)

```bash
# 1) Boot the app and drive the REAL reply entry point.
#    <observation_script> seeds User -> Mailbox -> Alias -> Contact, then calls
#    email_handler.handle(Envelope, Message) (which dispatches to handle_reply()).
docker exec sl-work bash -lc \
  'cd /app && PYTHONPATH=/app CONFIG=/app/tests/test.env python <observation_script>'
```

```bash
# 2) Confirm the unordered LIMIT 1 that causes the wrong-user behavior.
#    In-process: register a SQLAlchemy before_cursor_execute hook to print the emitted SQL,
#    then run the ORM lookup and an EXPLAIN.
docker exec sl-work bash -lc \
  'cd /app && PGPASSWORD=test psql -h localhost -p 15432 -U test -d test \
   -c "EXPLAIN SELECT id FROM contact WHERE reply_email='"'"'<R>'"'"' LIMIT 1;"'
```

```bash
# 3) Reproduce the flip: run a fresh-boot lookup 3 times, and force index vs seq scan.
docker exec sl-work bash -lc \
  'cd /app && PGPASSWORD=test psql -h localhost -p 15432 -U test -d test \
   -c "SET enable_seqscan=off; EXPLAIN SELECT id FROM contact WHERE reply_email='"'"'<R>'"'"' LIMIT 1;"'
```

Replace `<R>` with the reverse-alias address produced by `generate_reply_email()` during seeding (for example, `sender_at_external_test_<suffix>@sl.local`).

### 9.6 Verification Steps

- The happy-path reply returns `250 Message accepted for delivery`.
- The emitted contact lookup SQL contains `LIMIT` and **no** `ORDER BY`.
- After inserting a second `Contact` with the same `reply_email` for a different user and reordering the heap (delete+reinsert), the identical lookup resolves to the other user — reproduced across three fresh OS processes.
- Each guarded exit returns its exact `app/email/status.py` constant.

### 9.7 Troubleshooting

- **"Behavior won't reproduce on my machine"** — you are likely in the non-canonical host shell (Python 3.13, no PostgreSQL). Use `docker exec sl-work ...`.
- **"The user didn't flip on re-run"** — the flip depends on physical heap order; force it via delete+reinsert or `SET enable_seqscan=off` before the `EXPLAIN`/lookup.
- **"IDs don't match the document"** — expected. PostgreSQL sequences advance between runs; the document frames its IDs as a single captured run. The qualitative behavior is invariant.
- **"`psql` connection refused"** — confirm PostgreSQL is listening on port 15432 inside the container and that `CONFIG=/app/tests/test.env` is set.

---

## Section 10 — Appendices

### Appendix A — Command Reference

| Purpose | Command |
|---------|---------|
| Size the deliverable | `wc -l blitzy/documentation/app_2cd6ee777f8c.md` |
| Read-only net diff | `git diff --name-status 2cd6ee777f8c...HEAD` |
| Clean-tree check | `git status --porcelain` |
| Temp-script scan | `find . -type f -name 'obs_*.py' -not -path './.git/*'` |
| Fence balance | `grep -c '^```' blitzy/documentation/app_2cd6ee777f8c.md` |
| Placeholder scan | `grep -cE 'TODO\|FIXME\|XXX\|PLACEHOLDER' blitzy/documentation/app_2cd6ee777f8c.md` |
| Boot & drive real path | `docker exec sl-work bash -lc 'cd /app && PYTHONPATH=/app CONFIG=/app/tests/test.env python <script>'` |
| EXPLAIN the lookup | `PGPASSWORD=test psql -h localhost -p 15432 -U test -d test -c "EXPLAIN SELECT id FROM contact WHERE reply_email='<R>' LIMIT 1;"` |

### Appendix B — Port Reference

| Service | Port | Notes |
|---------|------|-------|
| PostgreSQL | 15432 | Canonical test DB (`tests/test.env` `DB_URI`), role `test`/`test`, database `test` |
| Redis | 6379 | Responds `PONG` in the canonical container |

### Appendix C — Key File Locations

| Path | Role |
|------|------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The sole deliverable** (created) |
| `email_handler.py` | `handle()` dispatch, `handle_reply()`, `get_mailbox_from_mail_from()` |
| `app/models.py` | `ModelMixin.get_by()` (`:83-84`), `Contact` table, `uq_contact` (`:1875`), `reply_email` column (`:1899`), `available_sl_email()` (`:1425`) |
| `app/email_validation.py` | `normalize_reply_email()` (`:25`), `_ALLOWED_CHARS` (`:9`) |
| `app/email_utils.py` | `generate_reply_email()` (`:1103`), `is_reverse_alias()` (`:1156`) |
| `app/contact_utils.py` | `create_contact()` (`:42`) + `IntegrityError` fallback (`:113-118`) |
| `app/email/status.py` | SMTP status constants (E201/E214/E501–E506) |
| `migrations/versions/2021_071310_78403c7b8089_.py` | `reply_email` index created with `unique=False` (`:22`) |
| `tests/test.env` | Canonical config (`EMAIL_DOMAIN=sl.local`, `DB_URI` on 15432, `NOT_SEND_EMAIL=true`) |

### Appendix D — Technology Versions

| Component | Version |
|-----------|---------|
| Python | 3.10.18 |
| SQLAlchemy | 1.3.24 |
| psycopg2(-binary) | 2.9.3 |
| aiosmtpd | 1.4.2 |
| PostgreSQL | 15 |
| Flask | ^1.1.2 |
| Flask-Migrate (Alembic) | ^2.5.3 |
| pytest | ^7.0.0 |

### Appendix E — Environment Variable Reference

| Variable | Value (canonical) | Purpose |
|----------|-------------------|---------|
| `CONFIG` | `/app/tests/test.env` | Selects the canonical test configuration |
| `EMAIL_DOMAIN` | `sl.local` | Alias/reverse-alias domain used during seeding |
| `DB_URI` | `postgresql://test:test@localhost:15432/test` | Canonical PostgreSQL connection |
| `NOT_SEND_EMAIL` | `true` | Prevents real SMTP egress during observation |
| `DMARC_CHECK_ENABLED` | `true` | Canonical inbound-check setting |
| `PYTHONPATH` | `/app` | Ensures app modules import inside the container |

### Appendix F — Developer Tools Guide

- **Read-first, then reproduce:** review the document, then reproduce in `sl-work` via `docker exec`.
- **SQLAlchemy echo:** attach a `before_cursor_execute` event to print the exact emitted SQL for the contact lookup.
- **`psql` + `EXPLAIN`:** confirm the plan is an unordered `Limit → (Seq|Index) Scan`; toggle `SET enable_seqscan=off` to force the index path.
- **Cross-process check:** run the fresh-boot lookup script three times to confirm the resolution is heap-order-driven, not a session-cache artifact.

### Appendix G — Glossary

| Term | Meaning |
|------|---------|
| **Reverse alias / reply email** | The `@sl.local` address (`Contact.reply_email`) that receives a reply and maps back to a `Contact`. |
| **`handle_reply()`** | The reply-resolution orchestrator invoked by `handle()` when `is_reverse_alias(rcpt_to)` is true. |
| **Unordered `LIMIT 1`** | A `SELECT ... LIMIT 1` with no `ORDER BY`; the returned row is not guaranteed stable across executions. |
| **TOCTOU** | Time-of-check-to-time-of-use race; here, a check-then-insert window that permits duplicate `reply_email` rows. |
| **`uq_contact`** | The only uniqueness constraint on `contact`: `(alias_id, website_email)` — it does **not** cover `reply_email`. |
| **Heap order** | PostgreSQL's physical row storage order; can shift after updates/deletes/vacuum, changing which row an unordered `LIMIT 1` returns. |
| **Non-canonical** | A value/branch produced outside the default configuration (e.g., forced `E201`/`E506`), explicitly labeled as such. |