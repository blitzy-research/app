# Blitzy Project Guide — SimpleLogin Alias Email-Forwarding Runtime Investigation

> **Brand color key:** Completed / AI Work = Dark Blue `#5B39F3` · Remaining / Not Completed = White `#FFFFFF` · Headings / Accents = Violet-Black `#B23AF2` · Highlight = Mint `#A8FDD9`

---

## 1. Executive Summary

### 1.1 Project Overview

This project is a **read-only runtime investigation** of SimpleLogin's alias email-forwarding path. The deliverable is a single evidence-backed Markdown document, `blitzy/documentation/app_2cd6ee777f8c.md`, that answers four diagnostic questions — success/failure log text, the SL Message-ID versus the original, the transformed `From` header (reverse-alias format), and the database records created by one forward — using **actual observed runtime values** captured by driving the real SMTP entry point `email_handler.handle()` on the canonical Python 3.10 / PostgreSQL 13 / Redis 6 stack. The target users are SimpleLogin engineers diagnosing a reported production inconsistency. No source code was modified; the investigation surfaces (but does not fix) the likely root cause.

### 1.2 Completion Status

```mermaid
%%{init: {"theme": "base", "themeVariables": {"pie1": "#5B39F3", "pie2": "#FFFFFF", "pieStrokeColor": "#B23AF2", "pieOuterStrokeColor": "#B23AF2", "pieStrokeWidth": "2px", "pieOuterStrokeWidth": "2px", "pieTitleTextSize": "16px", "pieSectionTextSize": "13px", "pieLegendTextSize": "13px"}}}%%
pie showData title Project Completion — 91.4% Complete (Hours)
    "Completed Work (AI)" : 32
    "Remaining Work" : 3
```

<p align="center"><strong>Center label:</strong> <code>91.4% Complete</code></p>

| Metric | Value |
|---|---|
| **Total Hours** | **35** |
| **Completed Hours (AI + Manual)** | **32** |
| &nbsp;&nbsp;• AI / Autonomous (Blitzy agents) | 32 |
| &nbsp;&nbsp;• Manual (Human) | 0 |
| **Remaining Hours** | **3** |
| **Percent Complete** | **91.4%** |

> Completion % follows the PA1 AAP-scoped methodology: `Completed ÷ (Completed + Remaining) = 32 ÷ 35 = 91.4%`. All 32 completed hours were delivered autonomously; 0 human hours have been spent to date.

### 1.3 Key Accomplishments

- ✅ Single deliverable created at the exact mandated path/name: `blitzy/documentation/app_2cd6ee777f8c.md` (562 lines, 63,475 bytes).
- ✅ **Q1** answered — both branches exercised separately: successful forward → `E200` `"250 Message accepted for delivery"`; non-existent alias → `E515` `"550 SL E515 Email not exist"`, with complete unedited `SL` DEBUG log output.
- ✅ **Q2** answered — the headline finding: a forward **preserves the original `Message-ID` byte-for-byte** (`sl_message_id` stays `NULL`); the SL Message-ID is minted **only in the reply phase**. Reply contrast captured (`<178345238069.27339.1560924962577896419.2@sl.local>` + `MessageIDMatching`).
- ✅ **Q3** answered — transformed `From` via `Contact.new_addr()` (AT format) plus the reverse-alias format; investigation observed the **sender-included** reverse-alias and correctly diagnosed the ORM `default=True`.
- ✅ **Q4** answered — one new-sender forward creates exactly **1 `Contact` + 1 `UserAuditLog` + 1 `EmailLog`** with actual IDs/timestamps; `sl_message_id` `NULL`; no `MessageIDMatching`.
- ✅ Every value captured through the **real entry point** `email_handler.handle()` on the **canonical stack**; each observation repeated (run1/run2).
- ✅ Autonomous validation: full suite **637 passed / 0 failed**; `tests/test_email_handler.py` **23 passed / 0 failed**; all 67+ `file:line` citations verified exact.
- ✅ One factual defect (DB-persistence/cleanup narrative) diagnosed, corrected, and re-verified against runtime.
- ✅ **Read-only constraint upheld** — net change from baseline is exactly one file; working tree clean; temporary scripts and the throwaway PostgreSQL 13 instance removed.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|---|---|---|---|
| _None blocking._ The deliverable is complete and independently validated. | No release-blocking issues. | — | — |
| (Advisory, out of scope) Reported production inconsistency root cause — forward preserves `Message-ID` while reply mints an SL Message-ID — is **documented but not remediated**. | Product/eng decision required; not a defect in the deliverable. Fixing was explicitly out of AAP scope. | Human reviewer / SimpleLogin eng | Follow-up ticket (see 1.6) |

### 1.5 Access Issues

**No access issues identified.** The repository, the canonical Docker image (`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`), PostgreSQL 13, and Redis 6 were all accessible during autonomous execution; no third-party credentials or external API access were required (the investigation used mail-sender store mode with `NOT_SEND_EMAIL=true`, contacting no real SMTP/DNS service).

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|---|---|---|---|---|
| Source repository (branch `blitzy-8a7a4847-…`) | Read/Write (git) | None | ✅ No issue | — |
| Canonical Docker image / Python 3.10 / PostgreSQL 13 / Redis 6 | Runtime | None | ✅ No issue | — |
| External SMTP / DNS / third-party APIs | N/A | Not used (store mode, `NOT_SEND_EMAIL=true`) | ✅ Not required | — |

### 1.6 Recommended Next Steps

1. **[High]** Technical review of `blitzy/documentation/app_2cd6ee777f8c.md`: confirm the four answers are coherent and spot-check a sample of the 67 `file:line` citations against source (≈1.5h).
2. **[Medium]** Approve and merge the PR, then **file a follow-up remediation ticket** capturing the Q2 forward-vs-reply Message-ID root cause for the backlog (the fix itself is out of scope for this read-only task) (≈1.0h).
3. **[Low]** _Optional_ — independently reproduce the headline Q2 finding on the canonical stack before acting on it; the Final Validator already reproduced all four answers (≈0.5h).

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|---|---:|---|
| Canonical environment standup & schema provisioning | 4 | Stand up the canonical stack (Docker image, Python 3.10, PostgreSQL 13 on port 15432, Redis 6); `poetry install`; `CONFIG=tests/test.env alembic upgrade head` → 77 tables (incl. resolving the bare-schema `pg_trgm` transactional-DDL nuance). |
| Runtime observation harness construction | 5 | Build throwaway pytest harness: `aiosmtpd` `Envelope`/`Message`, seed via `create_new_user()` + `Alias.create_new_random()`, capture outbound via `mail_sender.store_emails_instead_of_sending()`/`get_stored_emails()`, exact `SL`-format DEBUG log capture, DB snapshotting under the `flask_client` transaction. |
| Q1 capture — success (E200) + non-existent alias (E515) | 3 | Drive `handle()` for both branches separately, x2 runs; capture full `SL` DEBUG lines and byte-exact status strings. |
| Q2 capture — forward preservation + reply mint | 3 | Confirm forward preserves original `Message-ID` (`sl_message_id` NULL); exercise reply so `replace_original_message_id()` mints the SL Message-ID; capture `MessageIDMatching`; byte-exact, x2 runs. |
| Q3 capture — transformed `From` + reverse-alias | 2 | Capture transformed `From` and bare reverse-alias; observe the sender-included format and diagnose the `User.include_sender_in_reverse_alias` ORM `default=True`. |
| Q4 capture — before/after DB snapshots | 2 | Snapshot `EmailLog`/`Contact`/`UserAuditLog`/`MessageIDMatching` before/after one forward; report actual IDs and `arrow.utcnow` timestamps; new-sender vs known-sender cases. |
| Document authoring | 5 | Author the 562-line evidence-backed Markdown: all required sections, run1/run2 complete output, record→code mapping tables, verbatim question headings. |
| Citation grounding & exact verification | 2 | Ground every claim with `file:line` + responsible function; verify 67–73 locators exact against pristine source. |
| DB-persistence defect diagnosis & correction | 3 | Diagnose the incorrect rollback/persistence narrative; independent autocommit-connection re-verification on PG13 + PG15 (20 probes); apply 4 surgical `str_replace` edits; re-validate. |
| Codebase validation & document QA | 2 | Run full canonical suite (637 passed / 0 failed, 168.67s); confirm Markdown well-formedness (60 balanced fences); re-run all four answers. |
| Cleanup, read-only verification & commit | 1 | Remove temporary scripts; tear down throwaway PostgreSQL 13 cluster; reset environment; verify 1-file diff / clean tree; commit. |
| **Total Completed** | **32** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|---|---:|---|
| Documentation Review & Verification (read the 562-line doc; spot-check citations & the Q2 headline finding; optional confirmatory reproduction on the canonical stack) | 2.0 | Medium |
| Sign-off, Merge & Follow-up Ticket (approve/merge PR; file backlog ticket for the out-of-scope Q2 root-cause remediation) | 1.0 | Medium |
| **Total Remaining** | **3.0** | |

### 2.3 Total Project Hours (Reconciliation)

| Line | Hours |
|---|---:|
| Section 2.1 — Completed | 32 |
| Section 2.2 — Remaining | 3 |
| **Total Project Hours (= Section 1.2)** | **35** |
| **Percent Complete** = 32 ÷ 35 | **91.4%** |

> ✅ **Cross-section check:** 2.1 (32) + 2.2 (3) = 35 = Section 1.2 Total. Remaining (3) is identical in Section 1.2, Section 2.2, and the Section 7 pie chart.

---

## 3. Test Results

All results below originate from **Blitzy's autonomous validation logs** for this project. The committed deliverable is a Markdown document and therefore has **no unit tests of its own**; the tests validate the **codebase health and the referenced code paths** that the document describes, on the canonical stack.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---|---|---:|---:|---:|---|---|
| Full canonical suite | pytest | 637 | 637 | 0 | See note | Ran in 168.67s with 51 warnings (background-thread & SA warnings only). Excludes network-only `tests/api/test_apple.py`. |
| Referenced-path suite (`tests/test_email_handler.py`) | pytest | 23 | 23 | 0 | See note | Directly exercises the `handle()` / forward / reply paths the document documents. |
| Runtime answer reproduction (success / non-existent / reply) | pytest harness (throwaway) | 3 paths × 2 runs | 6 | 0 | N/A | Real `handle()` driven; E200 / E515 / minted SL Message-ID reproduced; byte-identical DKIM body hash across runs. |
| DB-state rollback probes (PG13 + PG15) | pytest harness (throwaway) | 20 | 20 | 0 | N/A | 20/20 forward probes returned E200 and left 0 rows after teardown rollback. |
| **Deliverable (`app_2cd6ee777f8c.md`)** | Markdown structural checks | — | — | — | N/A | Not a test target: 562 lines, 60 balanced code fences, 4 verbatim question headings present, 67 `file:line` locators verified exact. |

> **Coverage note:** the repository's coverage gate is `fail_under = 55` (`coverage.ini`). The clean 637/0 run was executed with coverage disabled (`-o addopts=""`) to isolate pass/fail; a discrete coverage percentage was therefore not separately captured in the validation logs and is not fabricated here. Code coverage is **not applicable** to the Markdown deliverable itself.

---

## 4. Runtime Validation & UI Verification

**Runtime health (real entry point `email_handler.handle()` on the canonical stack):**

- ✅ **Success forward** — valid alias + verified mailbox → `E200` `"250 Message accepted for delivery"`; full `forward_email_to_mailbox()` DEBUG sequence emitted.
- ✅ **Non-existent alias** — → `E515` `"550 SL E515 Email not exist"`; `not exist` / `cannot be created on-the-fly, return 550` DEBUG lines emitted.
- ✅ **Reply** — routes to `handle_reply()`; SL Message-ID minted via `make_msgid()` and `MessageIDMatching` row created.
- ✅ **Outbound message capture** — via `mail_sender` store mode: forward preserves original `Message-ID`; `From` transformed to reverse-alias; byte-exact serialization confirmed.
- ✅ **Database record creation** — new-sender forward → 1 `Contact` + 1 `UserAuditLog` + 1 `EmailLog`; `sl_message_id` `NULL`; 0 `MessageIDMatching`.
- ✅ **Canonical stack health** — Python 3.10.18, PostgreSQL 13.23, SQLAlchemy 1.3.24, Flask 1.1.2, aiosmtpd 1.4.2, arrow 0.16.0; 77 tables; alembic head `32f25cbf12f6`.
- ✅ **Read-only integrity** — `git diff 2cd6ee77 HEAD --name-status` = single `A` entry; working tree clean.

**API integration outcomes:**

- ✅ No external SMTP/DNS/third-party services contacted (store mode + `NOT_SEND_EMAIL=true`) — deterministic, side-effect-free capture.

**UI verification:**

- ⚠ **Not applicable.** This task has **no UI component in scope** — it is a backend email-handler investigation whose sole artifact is a Markdown document. No screens, flows, or visual assets were produced or required.

---

## 5. Compliance & Quality Review

Cross-mapping of AAP deliverables and governing-rule requirements ("SWE-AtlasQnA-Repo") to Blitzy quality/compliance benchmarks.

| Benchmark / Requirement | AAP Reference | Status | Progress | Evidence |
|---|---|---|---|---|
| Deliverable at exact path `blitzy/documentation/<branch>.md` | 0.3.1 / 0.7 | ✅ Pass | 100% | File present; `git` shows single `A` add; name matches branch `app_2cd6ee777f8c`. |
| Q1 — success vs. failure logs + status (both branches) | 0.1.1 R1 | ✅ Pass | 100% | Doc L128–256; E200 & E515 byte-exact; full `SL` DEBUG lines; run1/run2. |
| Q2 — SL Message-ID forward-vs-original (+ reply contrast) | 0.1.1 R2 | ✅ Pass | 100% | Doc L257–424; forward preserves Message-ID / `sl_message_id` NULL; reply mint + `MessageIDMatching`. |
| Q3 — transformed `From` + reverse-alias format | 0.1.1 R3 | ✅ Pass | 100% | Doc L425–479; AT format; sender-included reverse-alias diagnosed. |
| Q4 — DB records with actual IDs + timestamps | 0.1.1 R4 | ✅ Pass | 100% | Doc L480–536; before/after; 3-row result; actual IDs/timestamps. |
| Run-first methodology (observe, don't infer) | 0.7 | ✅ Pass | 100% | Complete unedited output + producing command for every claim. |
| Real entry point only (`handle()`) | 0.5.4 / 0.7 | ✅ Pass | 100% | Doc states `handle_forward`/`forward_email_to_mailbox`/`handle_reply` not called directly. |
| Canonical configuration + exact commands stated | 0.8.1 | ✅ Pass | 100% | Python 3.10.18 / PG13.23 / Redis 6; exact commands in "Environment & method". |
| Exercise every condition (both Q1 branches + reply) | 0.7 | ✅ Pass | 100% | All three paths captured, x2 runs. |
| Before/during/after for state changes | 0.7 | ✅ Pass | 100% | Q2 inbound-vs-outbound Message-ID; Q4 DB before/after; DB-state probe. |
| Byte-exact verification of Message-IDs / `From` | 0.5.4 / 0.7 | ✅ Pass | 100% | Captured via `message_to_bytes()`; body-hash identical across runs. |
| Grounding — `file:line` + function for every claim | 0.7 | ✅ Pass | 100% | 67 distinct locators; sample independently re-verified exact. |
| Read-only — no existing file modified | 0.3.2 / 0.7 | ✅ Pass | 100% | Net change = 1 file (562 insertions, 0 deletions); clean tree. |
| Cleanup — temp scripts + spun-up DB/containers | 0.8.1 | ✅ Pass | 100% | Harness deleted; throwaway PG13 cluster dropped; env reset. |
| Diagnostic-only — no fix applied | 0.3.2 / 0.8.2 | ✅ Pass | 100% | No source edit; root cause documented, not remediated. |
| **Fix applied during validation** — DB-persistence narrative correction | (QA) | ✅ Resolved | 100% | Rollback/persistence claim corrected to match runtime; 4 `str_replace` edits; re-verified on PG13 + PG15. |
| **Outstanding** — human review & sign-off | Path-to-production | ⬜ Pending | 0% | See Sections 1.6 / 2.2 (3h remaining). |

**Overall compliance:** 16 of 17 benchmarks satisfied autonomously; the one remaining item is human review/sign-off, which is inherently a manual gate.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|---|---|---|---|---|---|
| Volatile captured values (random alias local-parts, `make_msgid` timeval/pid/randint, autoincrement IDs, timestamps, DKIM `t=`/`b=`) do not reproduce byte-identical on re-run | Technical | Low | High (by design) | Document explicitly labels volatile-vs-stable fields; stable structure and identical body-hash (`bh=`) demonstrated across run1/run2 | ✅ Mitigated |
| Reported production inconsistency root cause (forward preserves `Message-ID`; SL Message-ID minted only on reply) is **documented but unremediated** | Technical | Medium | Medium | Exact code locus identified (`email_handler.py:L1296`, `L1311-L1313`); recommend a follow-up remediation ticket | ⬜ Open (by design — fix out of scope) |
| No source/dependency/secret changes → new attack surface | Security | Low | Low | Read-only investigation; sample addresses/IPs are synthetic test fixtures (not PII) | ✅ Mitigated (no exposure) |
| Reproduction requires the canonical stack + a **bare** `public` schema before `alembic upgrade head` (`pg_trgm` transactional-DDL nuance) | Operational | Low | Medium | Exact commands and the bare-schema requirement documented; non-canonical results labeled | ✅ Mitigated |
| Rate-limiter bucket confound can raise `429` during alias seeding on repeated low-`user_id` runs (does not honor `DISABLE_RATE_LIMIT`) | Operational | Low | Medium | Flush Redis `bl:*` buckets before each run; 20/20 probes returned E200 after flushing | ✅ Documented |
| No external integrations changed; store mode avoids real SMTP/DNS | Integration | Low | Low | `NOT_SEND_EMAIL=true` + `store_emails_instead_of_sending()` | ✅ N/A |
| Merge path — repository CI must remain green | Integration | Low | Low | Full pytest suite green (637/0); read-only 1-file diff | ✅ Mitigated |

**Risk posture:** Low overall. The deliverable is a validated, read-only documentation artifact with no code, dependency, or secret changes. The single Medium risk is the by-design **unremediated** production root cause the document surfaces — a product/engineering follow-up decision, not a defect in the deliverable.

---

## 7. Visual Project Status

```mermaid
%%{init: {"theme": "base", "themeVariables": {"pie1": "#5B39F3", "pie2": "#FFFFFF", "pieStrokeColor": "#B23AF2", "pieOuterStrokeColor": "#B23AF2", "pieStrokeWidth": "2px", "pieOuterStrokeWidth": "2px", "pieTitleTextSize": "15px", "pieSectionTextSize": "13px", "pieLegendTextSize": "13px"}}}%%
pie showData title Project Hours Breakdown (91.4% Complete)
    "Completed Work" : 32
    "Remaining Work" : 3
```

**Remaining hours by category (Section 2.2):**

```mermaid
%%{init: {"theme": "base", "themeVariables": {"xyChart": {"plotColorPalette": "#5B39F3"}}}}%%
xychart-beta
    title "Remaining Work by Category (Hours)"
    x-axis ["Doc Review & Verification", "Sign-off / Merge / Ticket"]
    y-axis "Hours" 0 --> 3
    bar [2.0, 1.0]
```

> ✅ **Integrity:** the "Remaining Work" value (3) equals Section 1.2 Remaining Hours and the sum of the Section 2.2 "Hours" column (2.0 + 1.0 = 3.0). "Completed Work" (32) equals Section 1.2 Completed Hours and the Section 2.1 total.

---

## 8. Summary & Recommendations

**Achievements.** The project is **91.4% complete** (32 of 35 hours). Every AAP-scoped requirement was delivered autonomously and independently validated: the single mandated document exists at the exact path, all four questions are answered with complete, unedited runtime output captured through the real `email_handler.handle()` entry point on the canonical Python 3.10 / PostgreSQL 13 / Redis 6 stack, each observation is repeated (run1/run2), and every factual claim is grounded in a verified `file:line` locator. The full canonical test suite passes 637/0, and the read-only constraint is provably upheld (net change = one file).

**Remaining gaps.** The only outstanding work is **human**: technical review and sign-off of the document (~2h), plus a merge and a follow-up remediation ticket for the surfaced root cause (~1h). There is **no blocking autonomous work** — the codebase compiles and tests pass, and no source file required (or received) modification.

**Critical path to production.** Review → approve/merge → (separately) triage the Q2 forward-vs-reply Message-ID root cause. Remediating that behavior was **explicitly out of scope** for this read-only diagnostic task and should be tracked as its own change.

**Success metrics.**

| Metric | Target | Actual |
|---|---|---|
| AAP-scoped completion | ≤ 99% pre-review | 91.4% |
| Questions answered with runtime evidence | 4 / 4 | 4 / 4 |
| Test suite pass rate | 100% | 637 / 637 (100%) |
| Citations verified exact | 100% | 67 / 67 |
| Source files modified | 0 | 0 |

**Production readiness.** The deliverable is **ready for human review**. As a validated, read-only diagnostic document with zero code impact, its risk to production is negligible; the actionable output is the identified root cause, which should be handed to the SimpleLogin team as a follow-up.

---

## 9. Development Guide

This guide explains how to build the canonical environment, reproduce the runtime observations, and verify the deliverable. Runtime reproduction is performed **inside the canonical Docker image**; the read-only verification commands run anywhere with `git`.

### 9.1 System Prerequisites

- **Docker** (canonical image: `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`), **or** a manual stack of:
  - **Python 3.10** (`Dockerfile` `FROM python:3.10`; CI `python-version: '3.10'`)
  - **PostgreSQL 13** (CI `image: postgres:13`, published on host port **15432**)
  - **Redis 6** (CI `redis-version: 6`)
- **Poetry** (dependency manager; the image sets `poetry config virtualenvs.create false`)
- OS packages: `libre2-dev`, `libpq-dev`

### 9.2 Environment Setup

```bash
# Option A — use the canonical prebuilt image (recommended)
docker pull ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0
docker run -it --rm ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0 bash

# Option B — manual OS packages (Debian/Ubuntu)
sudo apt update && sudo apt install -y libre2-dev libpq-dev
```

Ensure PostgreSQL 13 is reachable at `postgresql://test:test@localhost:15432/test` and Redis at `redis://localhost` (these match `tests/test.env`).

### 9.3 Dependency Installation

```bash
cd /app                     # repository root inside the image
poetry install --no-interaction
# (Dockerfile variant: poetry install --no-interaction --no-ansi --no-root)
```

### 9.4 Schema Provisioning

> **Important:** start from a **bare** `public` schema. Migration `424808e1fe49` runs `CREATE EXTENSION pg_trgm` inside Alembic's transactional DDL; pre-creating the extension rolls back the just-created tables.

```bash
# Optional clean slate:
PGPASSWORD=test psql -h localhost -p 15432 -U test -d test \
  -c "drop schema public cascade; create schema public;"

# Provision all 77 tables (Alembic, NOT create_all):
CONFIG=tests/test.env poetry run alembic upgrade head
```

### 9.5 Reproducing the Runtime Observations

The investigation drives the **real** router `email_handler.handle(envelope, msg)` under the `flask_client` fixture (transaction + rollback), captures the outbound message via mail-sender store mode, and captures `SL` DEBUG logs. Run a throwaway harness under pytest with DEBUG surfaced:

```bash
source /app/venv/bin/activate                 # or rely on virtualenvs.create false
export DB_URI=postgresql://test:test@localhost:15432/test
redis-cli flushall                            # clear rate-limiter bl:* buckets (avoids 429 on repeated seeds)

unset PYTEST_ADDOPTS
CONFIG=tests/test.env python -m pytest -o addopts="" -p no:rerunfailures \
  --timeout=180 -q -s -o log_cli=true -o log_cli_level=DEBUG \
  tests/<throwaway_investigation>.py
# Delete tests/<throwaway_investigation>.py afterward — it must never be committed.
```

### 9.6 Verification Steps

```bash
# 1) Read-only proof: the only net change from baseline is the one document
git diff --stat 2cd6ee77 HEAD
#   => blitzy/documentation/app_2cd6ee777f8c.md | 562 +++...   1 file changed, 562 insertions(+)

git diff --name-status 2cd6ee77 HEAD
#   => A  blitzy/documentation/app_2cd6ee777f8c.md

git status --porcelain          # (empty output = clean working tree)

# 2) Deliverable structural checks
wc -l blitzy/documentation/app_2cd6ee777f8c.md          # => 562
awk '/^```/{c++} END{print "fences="c" balanced="(c%2==0?"YES":"NO")}' \
  blitzy/documentation/app_2cd6ee777f8c.md              # => fences=60 balanced=YES
grep -c '^## ' blitzy/documentation/app_2cd6ee777f8c.md  # top-level question/section headings

# 3) Codebase health (inside the canonical image)
CONFIG=tests/test.env python -m pytest -o addopts="" -p no:rerunfailures \
  --ignore=tests/api/test_apple.py --timeout=90 --timeout-method=signal -q
#   => 637 passed
```

### 9.7 Example Usage

```bash
# View the delivered investigation document
less blitzy/documentation/app_2cd6ee777f8c.md

# Jump to a specific answer (verbatim question headings)
grep -n '^## ' blitzy/documentation/app_2cd6ee777f8c.md
```

### 9.8 Troubleshooting

- **`alembic upgrade head` fails on `CREATE INDEX ... ON alias` / pg_trgm rollback** → the `public` schema was not bare. Drop & recreate it (see 9.4), then re-run.
- **`werkzeug.exceptions.TooManyRequests` (429) during alias seeding** → the per-`user_id` rate-limiter bucket is saturated (it ignores `DISABLE_RATE_LIMIT`). Run `redis-cli flushall` before seeding.
- **Q1 `LOG.d` DEBUG lines missing from output** → DEBUG capture is off. Add `-s -o log_cli=true -o log_cli_level=DEBUG` (the `SL` logger is set to DEBUG in `app/log.py`).
- **Values differ from the document** → you are likely on a non-canonical stack (e.g., PostgreSQL 15 or SQLAlchemy 2.x). Reproduce on Python 3.10 / PostgreSQL 13. Note that volatile fields (random local-parts, `make_msgid` components, IDs, timestamps) vary **by design** — compare structure, not exact bytes.
- **Rows appear to persist after a run** → they do not; the `flask_client` fixture's teardown `transaction.rollback()` discards all committed rows (verified via an independent autocommit connection). Autoincrement sequences still advance, so IDs grow across runs.

---

## 10. Appendices

### Appendix A — Command Reference

| Purpose | Command |
|---|---|
| Install OS packages | `sudo apt install -y libre2-dev libpq-dev` |
| Install Python deps | `poetry install --no-interaction` |
| Provision schema | `CONFIG=tests/test.env poetry run alembic upgrade head` |
| Run full test suite (clean) | `CONFIG=tests/test.env python -m pytest -o addopts="" -p no:rerunfailures --ignore=tests/api/test_apple.py --timeout=90 --timeout-method=signal -q` |
| Referenced-path tests | `CONFIG=tests/test.env poetry run pytest tests/test_email_handler.py -q` |
| Read-only diff proof | `git diff --name-status 2cd6ee77 HEAD` |
| Fence-balance check | `awk '/^```/{c++} END{print c, (c%2==0)}' blitzy/documentation/app_2cd6ee777f8c.md` |
| Flush rate-limiter buckets | `redis-cli flushall` |

### Appendix B — Port Reference

| Service | Port | Notes |
|---|---|---|
| PostgreSQL 13 | `15432` (host) → `5432` (container) | `DB_URI=postgresql://test:test@localhost:15432/test` (`tests/test.env:L17`) |
| Redis 6 | `6379` | `MEM_STORE_URI=redis://localhost` (`tests/test.env:L78`) |
| App (gunicorn, not used here) | `7777` | `Dockerfile` `EXPOSE 7777` (irrelevant to this investigation) |

### Appendix C — Key File Locations

| Path | Role |
|---|---|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The deliverable** (sole net change) |
| `email_handler.py` | SMTP router; `handle()` L1945, `handle_forward()` L536, `forward_email_to_mailbox()` L679, `handle_reply()` L966, `replace_original_message_id()` L1296 |
| `app/models.py` | `Contact.new_addr()` L2008, `EmailLog` L2060, `MessageIDMatching` L3365, `ModelMixin` L62 |
| `app/email_utils.py` | `generate_reply_email()` L1103, `sl_formataddr()` L1501 |
| `app/contact_utils.py` | `create_contact()` L42 (+ `UserAuditLog` emission) |
| `app/email/status.py` | `E200` L2, `E515` L51 |
| `app/log.py` | `SL` logger + DEBUG format |
| `app/mail_sender.py` | `store_emails_instead_of_sending()` / `get_stored_emails()` |
| `tests/conftest.py` | `flask_client` transaction/rollback fixture |
| `tests/test.env` | Canonical test configuration |

### Appendix D — Technology Versions (canonical runtime observed)

| Component | Version |
|---|---|
| Python | 3.10.18 |
| PostgreSQL | 13.23 |
| Redis | 6 |
| SQLAlchemy | 1.3.24 |
| Flask | 1.1.2 |
| aiosmtpd | 1.4.2 |
| arrow | 0.16.0 |
| Alembic head | `32f25cbf12f6` (77 tables) |

### Appendix E — Environment Variable Reference

| Variable | Value | Source |
|---|---|---|
| `CONFIG` | `tests/test.env` | Selects the canonical test config |
| `DB_URI` | `postgresql://test:test@localhost:15432/test` | `tests/test.env:L17` |
| `EMAIL_DOMAIN` | `sl.local` | `tests/test.env:L8` (reverse-alias domain) |
| `NOT_SEND_EMAIL` | `true` | `tests/test.env:L7` (no real SMTP) |
| `OTHER_ALIAS_DOMAINS` | `["d1.test", "d2.test", "sl.local"]` | `tests/test.env:L9` |
| `ALIAS_AUTOMATIC_DISABLE` | `true` | `tests/test.env:L62` |
| `MEM_STORE_URI` | `redis://localhost` | `tests/test.env:L78` |

### Appendix F — Developer Tools Guide

- **git** — read-only verification (`git diff`, `git status`, `git log --author="agent@blitzy.com"`).
- **poetry** — dependency install & command runner (`poetry run …`).
- **alembic** — schema provisioning (`alembic upgrade head`); never `create_all`.
- **pytest** — codebase validation and the throwaway observation harness; use `-s -o log_cli=true -o log_cli_level=DEBUG` to surface `SL` DEBUG lines.
- **psql / redis-cli** — DB inspection and rate-limiter bucket flushing (present in the canonical image).

### Appendix G — Glossary

| Term | Meaning |
|---|---|
| **Alias** | A SimpleLogin address that forwards inbound mail to a user's real mailbox. |
| **Reverse-alias** | The masked `reply_email` (`{sanitized-sender}_{random}@sl.local` by default) that lets a user reply while hiding the real mailbox. |
| **SL Message-ID** | A SimpleLogin-minted `Message-ID` of shape `<{timeval}.{pid}.{randint}.{email_log.id}@sl.local>`, created **only in the reply phase**. |
| **Forward phase** | Inbound external mail → alias → mailbox; preserves the original `Message-ID`. |
| **Reply phase** | Outbound reply via the reverse-alias; mints an SL Message-ID and a `MessageIDMatching` row. |
| **`E200` / `E515`** | SMTP result strings: `"250 Message accepted for delivery"` / `"550 SL E515 Email not exist"`. |
| **`flask_client` fixture** | Test fixture wrapping each run in `connection.begin()` and rolling back at teardown, leaving the DB unchanged. |
| **AAP** | Agent Action Plan — the governing specification for this task. |

---

*End of Blitzy Project Guide. All figures are internally consistent: Completed = 32h, Remaining = 3h, Total = 35h, Completion = 91.4%. Colors applied per brand: Completed `#5B39F3`, Remaining `#FFFFFF`.*