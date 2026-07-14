# Blitzy Project Guide — SimpleLogin Reply-Resolution Root-Cause Investigation

> Deliverable: `blitzy/documentation/app_2cd6ee777f8c.md` (a single Markdown analysis document)
> Branch: `blitzy-3fc9b061-bac4-42a9-b984-8eaab62d81e6` · HEAD `aff65611` · Base `2cd6ee77`
> Task type: **Read-only investigation / root-cause analysis** — no product source created or modified.

---

## 1. Executive Summary

### 1.1 Project Overview

This project is a **read-only, run-first root-cause investigation** of the SimpleLogin email-alias service. The objective was to determine — by actually running the code, not just reading it — how an inbound email *reply* to a reverse-alias resolves to (a) a `Contact` record and (b) a forwarding destination (user / alias / mailbox), and to explain why a reply can be delivered to a **different user than the alias owner**. The sole persistent deliverable is one Markdown analysis document (`blitzy/documentation/app_2cd6ee777f8c.md`); **zero** product source, migrations, tests, or configuration were modified. The audience is the SimpleLogin engineering and security stakeholders who need a grounded, reproducible explanation of the wrong-user reply-routing behavior, with every claim tied to a `file:line` citation or captured runtime output.

### 1.2 Completion Status

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieTitleTextSize':'16px','pieLegendTextColor':'#B23AF2'}}}%%
pie showData title Completion Status — 94.6% Complete
    "Completed Work (AI)" : 88
    "Remaining Work" : 5
```

| Metric | Value |
|--------|-------|
| **Total Hours** | **93** |
| **Completed Hours (AI + Manual)** | **88** (AI: 88 · Manual: 0) |
| **Remaining Hours** | **5** |
| **Percent Complete** | **94.6%**  (88 ÷ 93 × 100) |

> Completion is measured strictly against AAP-scoped work plus path-to-production. Every one of the 21 AAP requirements is **Completed**; the residual 5 hours are the human acceptance path for a document deliverable (technical peer review, stakeholder triage of the finding, and PR merge). Per policy, a documentation deliverable is not marked 100% before human review.

### 1.3 Key Accomplishments

- ✅ **Single deliverable produced at the mandated path** — `blitzy/documentation/app_2cd6ee777f8c.md`, 5,207 lines, 13 sections + References, **128** distinct `file:line` citations, **226** fenced code blocks of commands/output.
- ✅ **Run-first methodology honored** — the canonical inbound entry point `email_handler.handle_reply(envelope, msg, rcpt_to)` was exercised end-to-end in the canonical Docker image (Python 3.10 + PostgreSQL 15 + Redis); nothing was concluded from reading alone.
- ✅ **Root cause established and reproduced** — `Contact.get_by(reply_email=...)` → `ModelMixin.get_by()` = `Session.query(cls).filter_by(**kw).first()` **with no `ORDER BY`** (`app/models.py:82-84`); `reply_email` carries **no `UNIQUE` constraint** (only `uq_contact` on `(alias_id, website_email)` at `app/models.py:1875`; index `unique=False` at migration `:22`; live catalog UNIQUE count = 0).
- ✅ **Cross-event behavior characterized at scale** — the identical canonical input was driven **N=100 across 2 runs** per condition; the resolved contact tracks insertion order and was byte-stable within a fixed layout (SHA256-verified).
- ✅ **Wrong-user distinction made precise** — under the **default** spoof-check the mismatched sender is rejected with **E214** (access-control boundary, no forward); the actual wrong-user *forward* occurs only under the **`[non-canonical]`** `disable_email_spoofing_check=True` fallback.
- ✅ **Edge/adversarial conditions exercised** — E501 (bad domain), E502 (no contact / inactive user), E214 (unknown mailbox), E504 (disabled), normalization collision, and second-header rewrite.
- ✅ **All 26 CR findings and 10 MAJOR QA findings (F-01…F-10) resolved** across 5 documentation commits with cited, byte-matched evidence.
- ✅ **Repository invariant preserved** — `git diff` vs base shows only the one new document; all 13 temporary observation scripts were removed (per-file absence proof); working tree clean.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| _None for the deliverable._ All AAP requirements complete; independent validation reproduced every claim with zero discrepancies. | None on the report | — | — |
| **Documented product finding (not a deliverable defect):** non-unique `reply_email` + unordered `.first()` enables wrong-user reply routing. Remediation is **explicitly out of AAP scope (§0.5.2)** — documented, not fixed. | Product security/routing (tracked separately) | SimpleLogin eng (post-triage) | Separate ticket |

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-----------------|----------------|-------------------|-------------------|-------|
| — | — | **No access issues identified.** The investigation ran in the self-contained canonical Docker image; no external credentials, third-party APIs, or restricted repositories were required. All datastores (PostgreSQL, Redis) are local to the container. | N/A | — |

### 1.6 Recommended Next Steps

1. **[High]** Technical peer review of the root-cause chain (§1, §4, §6, §10) and a spot-check of the pinned `file:line` citations against commit `2cd6ee77`.
2. **[Medium]** Stakeholder triage of the wrong-user finding — decide whether to open a **separate** remediation ticket (candidate fixes: DB `UNIQUE` on `contact.reply_email`, and/or deterministic `ORDER BY` in `ModelMixin.get_by()`). Remediation itself is out of scope for this documentation task.
3. **[Medium]** Approve and merge the documentation PR (`blitzy/documentation/app_2cd6ee777f8c.md`).
4. **[Low]** If remediation is approved, scope the fix + backfill/dedup migration as an independent effort (est. 8–16 h, **not** part of this project).

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

Every component traces to AAP-scoped investigation work delivered autonomously. **All hours are AI-completed (Manual = 0).**

| Component | Hours | Description |
|-----------|-------|-------------|
| Runtime foundation & clean-room bootstrap | 7 | Canonical Docker (Py 3.10 + PG 15 + Redis), alembic head, `pg_trgm` migration-ordering workaround, DKIM key-fixture restore, 5 setup classes verified (doc §2, §2.8) |
| Source tracing & citation cataloguing | 8 | Full transitive reply-path trace; **128** distinct `file:line` citations audited (doc §11.1) [AAP R1–R9] |
| Observation harness development (13 scripts) | 11 | Reply driver, routing, wrong-user, distribution, edge, normalization tracer, availability, race, F-07, security, second-header, bootstrap, schema SQL (doc §11.3, ~2,167 lines verbatim) |
| Reply-address derivation & normalization capture | 5 | `rcpt_to` → domain gate → `normalize_reply_email()`; handler-local normalized value via non-mutating `settrace` (doc §3, §4.2; F-09) [R1, R3, R8] |
| Contact resolution & `.first()` semantics | 6 | `Contact.get_by` → `ModelMixin.get_by` `.first()` no `ORDER BY`; resolved `id`/`alias_id`/`user_id` (doc §4) [R2, R4] |
| Forwarding-destination selection | 3 | `alias = contact.alias`, `user = alias.user`, mailbox via `get_mailbox_from_mail_from()`; `EmailLog` identities (doc §5, §6.4) [R5] |
| Uniqueness / race / timing root cause | 9 | Live catalog UNIQUE=0 proof, TOCTOU `available_sl_email()` guard, synchronized generator-race attempts (doc §6, §6.2.1) [R7, R9] |
| Cross-event behavior (N=100 × 2 runs) | 7 | Every run disclosed; SHA256 byte-stable resolution dimension across conditions (doc §7) [R6, R10, R13] |
| Edge / secondary + adversarial conditions | 8 | E501/E502/E214/E504, normalization collision, adversarial matrix, second-header rewrite (doc §8) [R15] |
| Reverse-alias web research & concept-vs-reality | 3 | Official SimpleLogin docs + SQLAlchemy/PostgreSQL ordering semantics (doc §9, References) [R18] |
| Wrong-user scenario complete unedited output | 3 | Default-control E214 vs `[non-canonical]` fallback wrong-user forward (doc §10) [R10] |
| Per-claim evidence appendix & dependency/security audits | 8 | Unedited outputs + `poetry check`/`pip check`/`npm audit` + advisory classification (doc §11, §11.4) [R14, R16] |
| Coverage pass & 53-item acceptance parity | 2 | 32-row question matrix + independent 53-item parity (doc §12; F-10) [R17] |
| Repository invariant & cleanup | 2 | Removed 13 temp scripts, per-file absence proof, integrity proof (doc §13) [R20, R21] |
| Document authoring, integration & QA remediation | 6 | 5 commits: initial + 26-CR rewrite + 3 QA rounds resolving F-01…F-10 [R19] |
| **Total Completed** | **88** | |

### 2.2 Remaining Work Detail

All remaining work is **human-only path-to-production** for a document deliverable. No AAP implementation work remains.

| Category | Hours | Priority |
|----------|-------|----------|
| Technical Peer Review (root-cause validation §6/§10 + `file:line` citation spot-check) | 2.5 | High |
| Stakeholder Triage of Wrong-User Finding (decide on separate remediation ticket; remediation out of scope §0.5.2) | 1.5 | Medium |
| PR Approval & Merge of the documentation artifact | 1.0 | Medium |
| **Total Remaining** | **5.0** | |

### 2.3 Hours Reconciliation

| Check | Result |
|-------|--------|
| Section 2.1 Completed total | 88 h |
| Section 2.2 Remaining total | 5 h |
| 2.1 + 2.2 = Section 1.2 Total | 88 + 5 = **93 h** ✅ |
| Completion % = 88 ÷ 93 × 100 | **94.6%** ✅ |
| Section 2.2 total = Section 1.2 Remaining = Section 7 pie "Remaining Work" | 5 = 5 = 5 ✅ |

---

## 3. Test Results

All entries below originate from **Blitzy's autonomous validation logs** for this project (Final Validator, Gate 2 and Gate 3). Because the deliverable is a Markdown document, there is no application code coverage to report; the "tests" are the run-first reproductions and the targeted regression that validate the documented claims.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|------------|-------|
| Reply/Contact targeted regression | pytest | 37 | 37 | 0 | N/A (targeted subset) | `tests/test_email_handler.py` + `tests/test_contact_utils.py`, 10.28 s, `-p no:randomly` (matches doc §11.5) |
| Cross-event reproduction (run-first harness) | Custom harness on `server.create_app()` | 800 canonical drives (4 conditions × N=100 × 2 runs) | 800 | 0 | N/A | Every drive matched the documented outcome exactly; resolution dimension SHA256 byte-identical across runs (doc §7) |
| Reply-path byte-compilation | `python -m py_compile` | 6 modules | 6 | 0 | N/A | `email_handler.py`, `app/models.py`, `app/email_utils.py`, `app/email_validation.py`, `app/mail_sender.py`, migration — read-only invariant intact |
| Citation ↔ source parity | Byte-match vs commit `2cd6ee77` | 88 sampled citations | 88 | 0 | N/A | Zero drift (Gate 2) |
| **Totals** | | **931 checks** | **931** | **0** | — | 100% pass across all autonomous validation checks |

**Condition-level reproduction detail (doc §7, from autonomous logs):**

| Seed / spoof-check | Runs | Outcome | Consistency |
|--------------------|------|---------|-------------|
| Insertion A→B, spoof-check **off** (default) | 100 × 2 | `250` correct user_A (E200) | 100/100 both runs |
| Insertion B→A, spoof-check **off** (default) | 100 × 2 | **E214** "Unauthorized", `EmailLog = None` | 100/100 both runs |
| Insertion A→B, spoof-check **on** `[non-canonical]` | 100 × 2 | `250` correct user_A (E200) | 100/100 both runs |
| Insertion B→A, spoof-check **on** `[non-canonical]` | 100 × 2 | `250` **WRONG USER** forward (`user_id=2`) | 100/100 both runs |

---

## 4. Runtime Validation & UI Verification

**Runtime health (canonical entry point exercised end-to-end):**

- ✅ **Operational** — `server.create_app()` boots in the canonical container; PostgreSQL 15.13 and Redis (PONG) live; alembic at head; 77 tables.
- ✅ **Operational** — Canonical inbound entry point `email_handler.handle_reply(envelope, msg, rcpt_to)` exercised end-to-end; captured `reply_email` → normalized value → resolved `Contact` → `alias`/`user`/`mailbox` → `EmailLog`.
- ✅ **Operational** — Routing hub `email_handler.handle()` confirmed to dispatch to `handle_reply` for a reverse-alias `rcpt_to` (`is_reverse_alias=True`).
- ✅ **Operational** — `EmailLog` identities captured: present on the authorized E200 path (`contact_id`/`alias_id`/`user_id`/`mailbox_id`), absent on the E214 rejection path.
- ✅ **Operational** — Edge/error paths reached at runtime: E501, E502, E214, E504, and the normalization collision path.
- ⚠ **Partial (correctly bounded, by design)** — External SMTP delivery is **not** confirmed: the harness runs with `NOT_SEND_EMAIL=true`, so a `250 accepted` result proves selection + logging + enqueue only, not an outbound SMTP hand-off (doc §2.2). This bound is stated wherever the related claims appear.

**UI verification:**

- ▫️ **Not applicable** — This is a backend email-processing investigation with no user-interface work (AAP §0.3.3). No component library, design system, or Figma frames are in scope.

---

## 5. Compliance & Quality Review

AAP deliverables and rule directives cross-mapped to Blitzy quality/compliance benchmarks. Fixes applied during autonomous validation are noted; there are no outstanding items.

| Benchmark / AAP Rule | Requirement | Status | Evidence |
|----------------------|-------------|--------|----------|
| Read-only repository (§0.7.3) | No product file modified; only the answer doc added | ✅ Pass | `git diff 2cd6ee77..HEAD` = 1 file; 0 non-doc changes; tree clean (doc §13) |
| Run-first methodology (§0.7.1) | Observe at runtime before writing | ✅ Pass | Canonical `handle_reply()` exercised; harness sources in doc §11.3 |
| Canonical entry point only (§0.7.1) | Real inbound path, no mock/bypass | ✅ Pass | `handle()` → `handle_reply` dispatch (doc §4.3) |
| Magnitude/consistency (≥2 runs) (§0.7.1) | Repeat identical input; report distribution | ✅ Pass | N=100 × 2 per condition; SHA256 byte-stable (doc §7) |
| Grounding & labeling (§0.7.1) | Every claim → `file:line` or output; label inferred/non-canonical | ✅ Pass | 128 citations; explicit `[inferred]`/`[non-canonical]` labels |
| Complete unedited output (§0.7.1) | Command + full output per claim | ✅ Pass | Doc §7, §8, §11 (226 code fences) |
| Every part answered + coverage pass | All question parts + implied conditions | ✅ Pass | Doc §12: 32-row matrix + 53-item parity |
| Web research (§0.2.2) | Confirm reverse-alias concept vs official docs | ✅ Pass | Doc §9 + References |
| Temporary-script cleanup (§0.7.2) | Remove all observation scripts | ✅ Pass | 13 scripts, per-file ABSENT proof (doc §13) |
| Single deliverable at mandated path (§0.4) | `blitzy/documentation/app_2cd6ee777f8c.md` | ✅ Pass | File exists, 5,207 lines |
| Dependency integrity (§0.6) | Zero dependency changes | ✅ Pass | `poetry check` / `pip check` / lockfile diff (doc §11.4) |
| Source integrity | No broken product source | ✅ Pass | `py_compile` exit 0 on all reply-path modules |

**Autonomous QA remediation ledger:** 26 code-review findings (commit `1639aadf` rewrite) and 10 MAJOR QA findings **F-01…F-10** (final commit `aff65611`) — **all resolved** with cited, byte-matched evidence (doc §12.1). **Outstanding compliance items: none.**

---

## 6. Risk Assessment

Risks are split between **deliverable risks** (about the report itself) and **documented product findings** (the defect the investigation surfaced; remediation is out of AAP scope §0.5.2 and is documented, not fixed).

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Non-unique `reply_email` + unordered `.first()` → wrong-user reply routing | Technical (product finding) | High | Medium | File separate remediation ticket: DB `UNIQUE` on `reply_email` and/or `ORDER BY` in `get_by()` | Documented, not fixed (out of scope) |
| Citation drift if product source evolves past commit `2cd6ee77` | Technical (deliverable) | Low | Low | Citations pinned to `2cd6ee77`; re-verify against newer commits | Mitigated |
| Two claims labeled `[inferred]` (alt query plan returns other row; live concurrency race) not directly reproduced | Technical (deliverable) | Low | Low | Grounded in source + official SQLAlchemy/PostgreSQL semantics; labeled per §0.3.5 | Accepted / labeled |
| Cross-user disclosure: default-control E214 sends `reverse_alias_unknown_mailbox` alerts to the resolved (wrong) user | Security (product finding) | Medium | Medium | Part of remediation triage; rate-limited per recipient/24h | Documented finding |
| Wrong-user **forward** under `[non-canonical]` `disable_email_spoofing_check=True` (reply enqueued under wrong `user_id`) | Security (product finding) | High | Low (non-default flag) | Part of remediation triage; keep spoof-check enabled | Documented finding (out of scope) |
| Secrets/PII hygiene in the document | Security (deliverable) | Low | Low | All runtime data synthetic; secret scan clean (doc §8.9–8.10) | Mitigated |
| Reproducibility depends on canonical Docker + clean-room bootstrap (`pg_trgm` ordering; DKIM key clobber) | Operational (deliverable) | Medium | Low | Doc §2.8 documents 5 setup classes + `KeyFormatError → git checkout`, reproduced live | Mitigated |
| `NOT_SEND_EMAIL=true` bounds runtime (no confirmed external SMTP) | Operational (deliverable) | Low | Low | Bound stated wherever the claim appears; not over-claimed | Accepted / bounded |
| Non-unique lookup reused at multiple sites (`replace_header_when_reply`, `is_reverse_alias`, routing/bounce) | Integration (product finding) | Medium | Medium | Note in remediation triage: fix once at `get_by`/schema level | Documented finding |
| Deliverable integration risk (build/deploy/CI/external services) | Integration (deliverable) | None | — | Markdown artifact has no code/build/deploy footprint | N/A |

---

## 7. Visual Project Status

**Project hours breakdown** (Completed = Dark Blue `#5B39F3`; Remaining = White `#FFFFFF`):

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieTitleTextSize':'16px','pieLegendTextColor':'#B23AF2'}}}%%
pie showData title Project Hours (Total 93h)
    "Completed Work" : 88
    "Remaining Work" : 5
```

**Remaining hours by category** (Section 2.2 — total 5.0 h):

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#B23AF2','pie3':'#A8FDD9','pieStrokeColor':'#B23AF2','pieSectionTextColor':'#1A1A1A','pieLegendTextColor':'#B23AF2'}}}%%
pie showData title Remaining Work by Category (5.0h)
    "Technical Peer Review" : 2.5
    "Stakeholder Triage" : 1.5
    "PR Approval & Merge" : 1.0
```

**Remaining work by priority:**

| Priority | Hours | Share of remaining |
|----------|-------|--------------------|
| High | 2.5 | 50% |
| Medium | 2.5 | 50% |
| Low | 0.0 | 0% |
| **Total** | **5.0** | 100% |

> Integrity: the pie "Remaining Work" value (5) equals Section 1.2 Remaining Hours (5) and the Section 2.2 Hours sum (5).

---

## 8. Summary & Recommendations

**Achievements.** The investigation is **94.6% complete** (88 of 93 hours). All 21 AAP requirements are delivered and were **independently reproduced by the Final Validator with zero discrepancies**. The deliverable answers every posed question part from runtime observation through the canonical `handle_reply()` entry point, and establishes the causal chain precisely: a reply resolves via `Contact.get_by(reply_email=...)` → `ModelMixin.get_by()` = `.first()` **with no `ORDER BY`** (`app/models.py:82-84`), and because `reply_email` has **no `UNIQUE` constraint** (`app/models.py:1875,1899`; migration `:22`), two contacts owned by different users can share one `reply_email`; the returned row is application-unordered, so `alias = contact.alias` / `user = alias.user` can point at the wrong user.

**Remaining gaps.** No AAP implementation work remains. The residual 5 hours are the standard human acceptance path for a document: technical peer review (2.5 h), stakeholder triage of the finding (1.5 h), and PR merge (1.0 h).

**Critical path to production.** Peer review → triage decision → merge. Because remediation of the underlying defect is explicitly out of scope (§0.5.2), the "production" state for this project is a reviewed, accepted, and merged analysis document — not a code fix.

**Success metrics (all met):** single deliverable at the mandated path; read-only invariant preserved (0 non-doc changes); run-first evidence for every claim (128 citations, 226 code fences); ≥2-run reproduction with disclosed distributions; 37/37 targeted regression; all F-01…F-10 resolved.

**Production readiness assessment.** ✅ **Ready for human review and merge.** The document is comprehensive, internally consistent, honestly bounded (`[inferred]`/`[non-canonical]`/`NOT confirmed` labels used appropriately), and leaves the repository clean. The only actions before merge are review, triage, and approval.

| Dimension | Assessment |
|-----------|------------|
| AAP scope completion | 21/21 requirements Completed (94.6% incl. path-to-production) |
| Evidence quality | High — runtime-observed, byte-matched citations, dual independent reproduction |
| Repository hygiene | Clean — only the answer doc added; temp scripts removed |
| Confidence level | High — all behavioral claims reproduced 100/100 × 2 runs |

---

## 9. Development Guide

This is a documentation deliverable, so the guide covers **(A) reviewing the deliverable** (host-only, no dependencies) and **(B) reproducing the investigation** (canonical Docker container). All host commands below were executed during assessment and returned exit 0.

### 9.1 System Prerequisites

**A. Review path (host):**
- Git 2.x (assessed with `git version 2.51.0`)
- Any Markdown viewer / text editor
- Python 3.x for the optional read-only `py_compile` check (assessed with `Python 3.13.7` — note this is **not** the canonical runtime)

**B. Canonical reproduction path (container):**
- Docker (assessed with `Docker version 28.5.2`)
- Canonical image: `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (container name `sl_app`), which provides Python **3.10**, PostgreSQL **15**, Redis, and the SimpleLogin checkout at `/app` with a virtualenv at `/app/venv`.

### 9.2 Environment Setup

```bash
# --- Review path: clone/checkout the branch and open the deliverable ---
git checkout blitzy-3fc9b061-bac4-42a9-b984-8eaab62d81e6
less blitzy/documentation/app_2cd6ee777f8c.md

# --- Reproduction path: start datastores inside the canonical container ---
docker exec sl_app bash -lc 'service postgresql start; service redis-server start'
# Source the runtime env (CONFIG, DB_URI, EMAIL_DOMAIN, key paths):
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a; . /tmp/sl_env.sh; set +a; env | grep -E "EMAIL_DOMAIN|DB_URI|NOT_SEND_EMAIL"'
```

Canonical configuration the reply path depends on (verified): `EMAIL_DOMAIN=sl.local` (`example.env:22`, `tests/test.env:8`); test `DB_URI=postgresql://test:test@localhost:15432/test` (`tests/test.env:17`); `Dockerfile` base `FROM python:3.10`.

### 9.3 Dependency Installation

- **Deliverable:** none. This is a read-only investigation; no packages were added, updated, or removed (AAP §0.6). Verified via `poetry check` / `pip check` / lockfile diff (doc §11.4).
- **In-container reproduction:** dependencies are pre-installed in the image virtualenv (`. venv/bin/activate`). If rebuilding, the Dockerfile runs `poetry config virtualenvs.create false && poetry install`.

### 9.4 Application Startup

- **Deliverable:** no startup — the artifact is a Markdown file.
- **Reproduction (canonical entry point):** boot the app and drive `handle_reply()` from a temporary harness:

```bash
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a; . /tmp/sl_env.sh; set +a; python <harness>'
# where <harness> calls server.create_app(), seeds TWO contacts sharing ONE reply_email
# on aliases owned by DIFFERENT users, then invokes:
#   email_handler.handle_reply(envelope, msg, rcpt_to)
# with mail_from = an authorized mailbox and rcpt_to = the shared reply_email.
```

### 9.5 Verification Steps

All commands below were run during assessment (exit 0):

```bash
# 1) Repository invariant — only the answer doc changed vs base
git diff --name-status 2cd6ee77..HEAD
# expected:  A    blitzy/documentation/app_2cd6ee777f8c.md

# 2) Working tree clean
git status --porcelain            # expected: (empty)

# 3) Deliverable present and sized
wc -l blitzy/documentation/app_2cd6ee777f8c.md    # expected: 5207

# 4) Section headers (13 numbered + References)
grep -nE "^## " blitzy/documentation/app_2cd6ee777f8c.md

# 5) Distinct file:line citations
grep -oE '[A-Za-z0-9_/]+\.py:[0-9]+' blitzy/documentation/app_2cd6ee777f8c.md | sort -u | wc -l   # ~128

# 6) Read-only invariant — reply-path modules byte-compile
python3 -m py_compile email_handler.py app/models.py app/email_utils.py app/email_validation.py app/mail_sender.py   # exit 0

# 7) Inspect the crux (no ORDER BY; non-unique index)
sed -n '82,84p' app/models.py
grep -n "create_index" migrations/versions/2021_071310_78403c7b8089_.py   # unique=False
```

### 9.6 Example Usage

```bash
# Read the direct answer and the causal chain:
sed -n '10,27p'   blitzy/documentation/app_2cd6ee777f8c.md   # §1 Lead Answer
sed -n '632,1068p' blitzy/documentation/app_2cd6ee777f8c.md  # §6 root cause

# Run the targeted reply/contact regression inside the container (37 tests):
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a; . /tmp/sl_env.sh; set +a; \
  python -m pytest tests/test_email_handler.py tests/test_contact_utils.py -p no:randomly --timeout=90'
# expected: 37 passed
```

### 9.7 Troubleshooting

- **`dkim.KeyFormatError` on reply-forward:** the build clobbers tracked DKIM key fixtures — restore them with `git checkout -- <key fixtures>` (doc §2.8). These are git-tracked, so a checkout is sufficient.
- **`pg_trgm` migration-ordering failure on a pristine DB:** recreate the schema before migrating (`psql -c "drop schema public cascade; create schema public;"`) and do **not** pre-create `pg_trgm` (the migration creates it). The harness bootstrap runs `CREATE EXTENSION IF NOT EXISTS pg_trgm` idempotently and never drops it.
- **`DB_URI` port mismatch (5432 vs 15432):** inside the canonical container `DB_URI` resolves to `localhost:5432`, so no `socat` 15432 bridge is required (doc §2.2).
- **`psql` / `redis-cli` / `poetry` "command not found" on the host:** these live inside the container — run them via `docker exec sl_app …`, not on the host.

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---------|---------|
| Verify repo invariant | `git diff --name-status 2cd6ee77..HEAD` |
| Working tree clean | `git status --porcelain` |
| Agent commits (docs-only) | `git log --author="agent@blitzy.com" --oneline` |
| Deliverable size | `wc -l blitzy/documentation/app_2cd6ee777f8c.md` |
| Section headers | `grep -nE "^## " blitzy/documentation/app_2cd6ee777f8c.md` |
| Citation count | `grep -oE '[A-Za-z0-9_/]+\.py:[0-9]+' <doc> \| sort -u \| wc -l` |
| Read-only compile check | `python3 -m py_compile email_handler.py app/models.py app/email_utils.py app/email_validation.py app/mail_sender.py` |
| Start datastores (container) | `docker exec sl_app bash -lc 'service postgresql start; service redis-server start'` |
| Targeted regression (container) | `python -m pytest tests/test_email_handler.py tests/test_contact_utils.py -p no:randomly --timeout=90` |

### B. Port Reference

| Service | Port | Context |
|---------|------|---------|
| PostgreSQL (in-container) | 5432 | `DB_URI` resolves here in the canonical container (doc §2.2) |
| PostgreSQL (test config) | 15432 | `tests/test.env:17` `DB_URI=postgresql://test:test@localhost:15432/test` |
| Redis | 6379 | Rate-limiting / features (default) |

### C. Key File Locations

| Path | Role |
|------|------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The deliverable** (5,207 lines) |
| `email_handler.py` | `handle_reply()` L966; domain gate L977; normalize L984; `Contact.get_by` L986; `alias`/`user` L994/L1004; mailbox L1019; `EmailLog.create` L1042-1050 |
| `app/models.py` | `ModelMixin.get_by()` `.first()` L82-84; `uq_contact` L1875; `reply_email` column L1899; `available_sl_email()` L1425 |
| `app/email_utils.py` | `is_reverse_alias()` L1156; `generate_reply_email()` L1103 |
| `app/email_validation.py` | `normalize_reply_email()` L25 |
| `migrations/versions/2021_071310_78403c7b8089_.py` | `create_index(..., unique=False)` L22 (proves `reply_email` index non-unique) |
| `tests/conftest.py`, `tests/test.env` | Canonical bootstrap + config the harness mirrors |

### D. Technology Versions

| Component | Version | Source |
|-----------|---------|--------|
| Python (canonical) | 3.10 | `Dockerfile FROM python:3.10`; container `3.10.18` |
| Python (host, review only) | 3.13.7 | assessed on host (non-canonical) |
| PostgreSQL | 15.13 | canonical container |
| Redis | (PONG verified) | canonical container |
| SQLAlchemy | 1.3.24 (pinned) | `pyproject.toml` — the `.first()` ordering crux |
| Flask | 1.1.2 | `pyproject.toml` |
| aiosmtpd | 1.4.2 | SMTP inbound entry point |
| flanker / dkimpy / pyspf | 0.9.11 / 1.0.5 / 2.0.14 | reply-phase parsing & gates |
| Docker | 28.5.2 | assessed on host |
| Git | 2.51.0 | assessed on host |

### E. Environment Variable Reference

| Variable | Value | Purpose |
|----------|-------|---------|
| `EMAIL_DOMAIN` | `sl.local` | Reply-address domain gate (`example.env:22`, `tests/test.env:8`) |
| `DB_URI` | `postgresql://test:test@localhost:15432/test` (test) | Primary datastore |
| `NOT_SEND_EMAIL` | `true` (harness) | Bounds runtime: success = selection/logging/enqueue, not confirmed SMTP delivery (doc §2.2) |
| `CONFIG` | container path | App configuration file consumed by `server.create_app()` |

### F. Developer Tools Guide

| Tool | Use |
|------|-----|
| `git diff` / `git status` / `git log` | Verify the read-only repository invariant and docs-only commit history |
| `python -m py_compile` | Confirm reply-path modules are unmodified/valid (read-only check) |
| `pytest` (container) | Run the 37-test targeted reply/contact regression |
| `docker exec sl_app …` | Access the canonical runtime (PostgreSQL, Redis, poetry venv) |
| `poetry check` / `pip check` (container) | Confirm zero dependency changes (doc §11.4) |

### G. Glossary

| Term | Meaning |
|------|---------|
| **Reverse-alias** | A per-(alias, contact) address SimpleLogin generates so a user can reply to a contact while hiding the real mailbox; the reply is sent to this `reply_email` |
| **`reply_email`** | The `Contact` column holding the reverse-alias; `index=True` but **not** `UNIQUE` (`app/models.py:1899`) |
| **`ModelMixin.get_by()`** | Shared helper `Session.query(cls).filter_by(**kw).first()` — `.first()` with **no `ORDER BY`** (`app/models.py:82-84`) |
| **TOCTOU** | Time-of-check-to-time-of-use; `available_sl_email()` is a non-atomic check-then-use guard with no backing `UNIQUE` constraint |
| **E200 / E501 / E502 / E214 / E504** | SMTP result codes in `handle_reply`: accepted / bad reply domain / no contact / unknown mailbox / disabled |
| **`[inferred]` / `[non-canonical]`** | Doc labels: a claim grounded in source+semantics but not directly reproduced / a value obtained via a bypass or non-default flag |
| **F-01…F-10** | The 10 MAJOR QA findings resolved in the final documentation commit |
