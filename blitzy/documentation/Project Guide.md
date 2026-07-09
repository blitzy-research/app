# Blitzy Project Guide
## SimpleLogin Email-Forwarding Pipeline — Runtime Investigation (Q1–Q4)

> **Brand legend** — <span style="color:#5B39F3">**Dark Blue #5B39F3 = Completed / AI Work**</span> · **White #FFFFFF = Remaining / Not Completed** · Violet-Black #B23AF2 headings/accents · Mint #A8FDD9 highlight.

---

## 1. Executive Summary

### 1.1 Project Overview

This project is a **read-only runtime investigation** of the SimpleLogin email-forwarding pipeline, undertaken to resolve a production report of "inconsistent" alias forwarding. Its sole deliverable is a single evidence-backed answer document that captures the *actual generated values* observed when the real SMTP handler (`email_handler.handle`) processes an inbound email addressed to an alias. It answers four questions: the exact success-vs-failure log text, the SL `Message-ID` versus the original, the transformed `From` header (reverse-alias format), and the database records created by one forward (real IDs and timestamps). The target users are the SimpleLogin engineers triaging the incident. No application behavior was changed — the work is documentation only.

### 1.2 Completion Status

```mermaid
%%{init: {"theme": "base", "themeVariables": {"pie1": "#5B39F3", "pie2": "#FFFFFF", "pieStrokeColor": "#B23AF2", "pieOuterStrokeColor": "#B23AF2", "pieSectionTextColor": "#111111", "pieTitleTextColor": "#B23AF2", "pieLegendTextColor": "#111111"}}}%%
pie showData title Completion Status — 94.7% Complete
    "Completed Work (AI)" : 36
    "Remaining Work" : 2
```

| Metric | Value |
|--------|-------|
| **Total Hours** | **38.0 h** |
| **Completed Hours (AI + Manual)** | **36.0 h** (AI 36.0 + Manual 0.0) |
| **Remaining Hours** | **2.0 h** |
| **Percent Complete** | **94.7 %** (36.0 / 38.0 × 100) |

### 1.3 Key Accomplishments

- ✅ **Single deliverable created and committed** — `blitzy/documentation/app_2cd6ee777f8c.md` (689 lines), the only change vs. base (net +689 / −0).
- ✅ **All four questions answered from observed runtime output** — Q1 (success/failure log text), Q2 (SL vs. original `Message-ID`), Q3 (transformed `From` / reverse-alias), Q4 (DB records with real IDs & timestamps).
- ✅ **Real entry point exercised** — every value captured by calling `email_handler.handle(envelope, msg)`, the production SMTP path.
- ✅ **Both branches of Q1 exercised** — successful forward → `E200`; non-existent, non-auto-creatable alias → `E515`.
- ✅ **Q2 nuance nailed** — proved the forward *preserves* the original `Message-ID` and that the SL id is minted *only* on the reply path via `make_msgid`.
- ✅ **Reported "inconsistency" reproduced, not engineered away** — byte-identical input run in two independent processes; stable structure separated from genuinely-varying components (Category A/B/C).
- ✅ **Empirical accuracy beyond the AAP hypothesis** — discovered the `user_audit_log` DB row (a real forward side effect) that the AAP's preliminary 2-row expectation missed.
- ✅ **Read-only scope honored** — zero source/config/test/migration modifications; working tree clean; temp scripts and services torn down.
- ✅ **Autonomous validation passed** — `py_compile`/`compileall` clean; `tests/test_email_handler.py` 23 passed / 0 failed; deliverable independently reproduced against the canonical runtime.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| _None._ No unresolved issues block release or validation. The deliverable is complete, accurate, and committed; the only outstanding item is the human review/acceptance gate (see §1.6 / §2.2). | — | — | — |

### 1.5 Access Issues

| System / Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-------------------|----------------|-------------------|-------------------|-------|
| Canonical container image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` | Container registry pull | Needed only for *independent re-verification*; the canonical Python 3.10 runtime is not available on a stock Python 3.13 host. Not required to accept the deliverable. | Not blocking — image tag documented; validator already reproduced independently | Reviewing engineer |

> No access issues prevent build validation, integration, or acceptance of this deliverable. The single entry above is a convenience note for optional re-verification only.

### 1.6 Recommended Next Steps

1. **[High]** Review and accept the answer document `blitzy/documentation/app_2cd6ee777f8c.md`; confirm Q1–Q4 (and the variability analysis) satisfy the incident question, then close the production "inconsistent forwarding" report.
2. **[Low]** (Optional) Independently re-run the observation harness in the canonical Python 3.10 image to reconfirm the run-varying values and the Category A/B/C classification.
3. **[Low]** (Optional) If the `file:line` citations will be referenced long-term, pin readers to commit `2cd6ee777f8c…` (already noted in the document) so line numbers remain valid as the source evolves.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

All completed hours are autonomous (AI); there were no manual/human hours in the delivered work.

| Component | Hours | Description |
|-----------|------:|-------------|
| Canonical runtime provisioning & schema verification | 3.0 | Start PostgreSQL (listen port aligned to `15432`) + Redis; verify schema at Alembic head `32f25cbf12f6` (77 tables); confirm `alembic upgrade head` is a no-op; verify dependency versions against `pyproject.toml`. |
| DKIM key-format environment fix | 2.0 | Diagnose `dkimpy` ASN1 error; re-encode the *same* key material PKCS#8→PKCS#1 to a temp path and point `DKIM_PRIVATE_KEY_PATH` to it; `local_data/dkim.key` left untouched; re-run tests to 23/23. |
| Observation harness authoring | 6.0 | Author temp `observe.py`: seed user + mailbox + fixed alias, enable `mail_sender.store_emails_instead_of_sending()`, build `Envelope` + `EmailMessage`, call the real `handle()`, snapshot all 77 tables before/after, and read records back through the ORM. |
| Scenario execution & evidence capture | 3.0 | Exercise forward ×2, non-existent-alias failure, reply (mint + reuse), no-`Message-ID` reply ×2, and a different-sender forward — across two independent processes (pids 407 / 431). |
| Q1 analysis & writeup | 4.0 | Success (`E200`) and failure (`E515`) unedited `SL` log blocks, per-line `file:line` tables, and cause→effect reasoning. |
| Q2 analysis & writeup | 4.0 | Forward-preserves-original vs. reply-mints nuance; `make_msgid` token decomposition; structural-difference table; mint/reuse/fresh-mint cases. |
| Q3 analysis & writeup | 3.0 | Exact transformed `From`; `AT` display-name anatomy; both `generate_reply_email` branches; cross-sender comparison. |
| Q4 analysis & writeup | 3.5 | All-77-table diff methodology; three-row finding incl. the `user_audit_log` discovery; `ModelMixin` id/timestamp shape; creation order. |
| Run-to-run variability analysis & writeup | 2.5 | Two-process comparison; Category A (varies) / B (stable) / C (input-driven) classification of every observed value. |
| Document assembly, citation verification & appendix | 2.0 | Purpose / Environment & Invocation / Cleanup sections; verification of all cited `file:line` references; appendix compilation. |
| Code-review response iteration | 3.0 | Substantive review-driven rewrite (commit `a92e5a41`, +458 / −356). |
| **Total Completed** | **36.0** | — |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|------:|----------|
| Human review & acceptance / sign-off of the answer document (verify Q1–Q4 + variability, spot-check evidence & citations, close the incident) | 1.5 | High |
| (Optional) Independent re-run in the canonical Python 3.10 image to reconfirm run-varying values | 0.5 | Low |
| **Total Remaining** | **2.0** | — |

### 2.3 Total Project Hours & Reconciliation

| Line | Hours |
|------|------:|
| Section 2.1 — Completed | 36.0 |
| Section 2.2 — Remaining | 2.0 |
| **Total Project Hours** | **38.0** |
| **Completion** | **94.7 %** |

> **Integrity check:** 36.0 (2.1) + 2.0 (2.2) = **38.0** = Total in §1.2 ✓ · Remaining 2.0 is identical in §1.2, §2.2, and §7 ✓.

---

## 3. Test Results

All tests below originate from Blitzy's autonomous validation logs for this project.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|------------:|-------:|-------:|-----------:|-------|
| Unit / Integration (email handler) | pytest | 23 | 23 | 0 | Not measured (targeted harness) | `tests/test_email_handler.py` — the canonical `handle()` invocation harness (21 test functions; `test_dmarc_reply_quarantine` is parametrized). 11 initial failures were **all** DKIM "Cannot create DKIM signature" (PKCS#8 vs PKCS#1); resolved via an **environment-only** key re-encode (no source change) → re-run **23/23 pass**. |
| Static compilation | `py_compile` / `compileall` | — | Pass | 0 | — | `python -m py_compile email_handler.py` OK; `python -m compileall app/` exit 0; all target modules import under `CONFIG=tests/test.env`, `PYTHONPATH=/app`. |
| Runtime reproduction (deliverable accuracy) | Custom observation harness | 1 (×2 runs) | Pass | 0 | — | `observe.py` executed byte-identical input in two processes; every Q1–Q4 claim confirmed. Differences limited to the run-varying components the document itself documents. See §4. |

> **Note:** No line-coverage percentage was reported by the autonomous logs for this targeted investigation; the pytest run validated the canonical `handle()` path rather than a coverage sweep. Values are reported honestly rather than estimated.

---

## 4. Runtime Validation & UI Verification

Status legend: ✅ Operational · ⚠ Partial · ❌ Failing

**Runtime health — email-forwarding pipeline (real `handle()` entry point):**

- ✅ **Successful forward** — valid alias routes to `handle_forward`; returns `E200` `"250 Message accepted for delivery"`; one message stored.
- ✅ **Failure path** — non-existent, non-auto-creatable alias declines both auto-create checks and returns `E515` `"550 SL E515 Email not exist"`; nothing stored.
- ✅ **Reply path** — replying to the reverse-alias mints the SL `Message-ID` via `make_msgid`; a repeat with the same reply `Message-ID` reuses the stored id; a no-`Message-ID` reply fresh-mints each time.
- ✅ **Outgoing-message capture** — `mail_sender.store_emails_instead_of_sending()` retained the fully transformed message for byte-fidelity inspection of `From` (Q3) and `Message-ID` (Q2).
- ✅ **Zero external side effects** — outbound suppressed two ways (`NOT_SEND_EMAIL=true` + the store hook); no mail transmitted, no third-party calls.
- ✅ **Database persistence verified** — all-77-table before/after diff proved exactly three rows created on a new-sender forward (`contact`, `user_audit_log`, `email_log`) with real IDs & `created_at`; **no** `message_id_matching` on the forward path.
- ✅ **Run-to-run stability** — two independent processes produced byte-identical stable values (status codes, `From`/reverse-alias, preserved forward `Message-ID`, reused reply SL id).
- ✅ **Compilation & imports** — `py_compile` / `compileall` clean under the canonical config.

**UI verification:**

- ⚠ **Not applicable** — this is a backend email-pipeline investigation with no frontend changes. No UI was in scope, so no UI verification was performed. (SimpleLogin's web UI exists in the repository but is untouched by this task.)

---

## 5. Compliance & Quality Review

Cross-map of AAP deliverables / rules → autonomous quality & compliance benchmarks.

| Requirement (AAP / rule set) | Benchmark | Status | Progress | Notes / Fixes Applied |
|------------------------------|-----------|--------|----------|-----------------------|
| Q1 — success vs. failure log text with `file:line` | Evidence-backed, unedited output | ✅ Pass | 100% | Full `SL` blocks + per-line citation tables; `E200`/`E515` verified at `status.py:2`/`:51`. |
| Q2 — SL vs. original `Message-ID` | Correct semantics + structural contrast | ✅ Pass | 100% | Forward-preserves nuance proven; `make_msgid` at `email_handler.py:1311-1314` verified. |
| Q3 — transformed `From` / reverse-alias | Exact string + format rationale | ✅ Pass | 100% | Both `generate_reply_email` branches documented; `new_addr` `:2008` verified. |
| Q4 — DB records (IDs + timestamps) | Empirical, all-tables diff | ✅ Pass | 100% | 3-row finding incl. `user_audit_log`; `ModelMixin` `:62-65` verified; no `message_id_matching` on forward. |
| Run-first, then write | Values from observed runtime output | ✅ Pass | 100% | All values captured from `run.log` / `run2.log`. |
| Exercise every condition | Success + failure + reply + new/existing contact | ✅ Pass | 100% | All branches exercised, plus different-sender case. |
| Reproduce the inconsistency | Repeat identical input, report distribution | ✅ Pass | 100% | Two processes; Category A/B/C table. |
| Canonical build & config | `EMAIL_DOMAIN=sl.local`, Python 3.10, exact commands | ✅ Pass | 100% | `tests/test.env` values + `Dockerfile python:3.10` confirmed; commands stated. |
| Read-only scope | No source/config/test/migration edits | ✅ Pass | 100% | `git diff --name-only` = only the deliverable; tree clean. |
| Deliverable name/location | `blitzy/documentation/<branch>.md` | ✅ Pass | 100% | `app_2cd6ee777f8c.md`; base SHA matches AAP exactly. |
| Cleanup | Temp scripts removed, services torn down | ✅ Pass | 100% | Documented in the deliverable; nothing untracked in repo. |
| Compilation & unit tests | Clean compile; canonical tests pass | ✅ Pass | 100% | `compileall` exit 0; `test_email_handler.py` 23/23 (DKIM fix env-only). |
| Human review / acceptance | Stakeholder sign-off | ⬜ Pending | 0% | The sole remaining item (§2.2, §1.6). |

**Quality note (fix applied during autonomous validation):** the DKIM signing prerequisite (`dkimpy` requires PKCS#1) was satisfied by re-encoding the *same* key material to a temp path and selecting it via `DKIM_PRIVATE_KEY_PATH` — an **environment-only** change that left `local_data/dkim.key` and the source tree unmodified.

---

## 6. Risk Assessment

All identified risks are **Low** severity. No High or Critical risks exist for this read-only, zero-source-change documentation task.

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Canonical Python 3.10 runtime not reproducible on a stock host (host has 3.13) — needed only for optional re-verification | Technical | Low | Medium | Exact image tag + commands documented; validator already reproduced independently | Mitigated |
| Inherent run-to-run nondeterminism (`random_string`, `make_msgid`) misread as a code defect | Technical | Low | Low | Category A/B/C analysis separates stable structure from varying components; recommends monitors assert shape, not exact strings | Resolved (explained) |
| `file:line` citations may drift if source is later edited | Technical | Low | Medium (over time) | All references pinned to commit `2cd6ee777f8c…` | Documented |
| DKIM temp key handling | Security | Low | Low | PKCS#1 re-encode of the *same* material written to `/tmp` and removed; `local_data/dkim.key` untouched; nothing secret committed (tree clean) | Resolved |
| Document quotes reverse-alias / IDs / timestamps | Security | Low | Low | Values are ephemeral test-seed data on `sl.local` from a disposable DB — not production PII | Resolved |
| Backing services + port `15432` alignment required to re-execute the harness | Operational | Low | Low | Exact service-start commands documented; Postgres data directory untouched | Documented |
| No external integrations / credentials involved | Integration | Low | Low | Outbound suppressed two ways (`NOT_SEND_EMAIL` + store hook); no API keys required | Resolved (by design) |

---

## 7. Visual Project Status

```mermaid
%%{init: {"theme": "base", "themeVariables": {"pie1": "#5B39F3", "pie2": "#FFFFFF", "pieStrokeColor": "#B23AF2", "pieOuterStrokeColor": "#B23AF2", "pieSectionTextColor": "#111111", "pieTitleTextColor": "#B23AF2", "pieLegendTextColor": "#111111"}}}%%
pie showData title Project Hours Breakdown (Total 38.0 h)
    "Completed Work" : 36
    "Remaining Work" : 2
```

**Remaining hours by priority (from §2.2):**

| Priority | Hours | Share of Remaining |
|----------|------:|-------------------:|
| High (review & acceptance) | 1.5 | 75% |
| Low (optional re-run) | 0.5 | 25% |
| **Total Remaining** | **2.0** | 100% |

> **Integrity check:** pie "Remaining Work" = **2.0** = §1.2 Remaining = §2.2 total ✓ · pie "Completed Work" = **36.0** = §1.2 Completed = §2.1 total ✓ · Colors: Completed **#5B39F3**, Remaining **#FFFFFF** ✓.

---

## 8. Summary & Recommendations

**Achievements.** The project is **94.7% complete** (36.0 of 38.0 hours). The single AAP deliverable — `blitzy/documentation/app_2cd6ee777f8c.md` — is authored, committed, and independently validated. It answers all four questions from *observed* runtime output with unedited logs, exact header/record values, `file:line` citations, and cause→effect reasoning, and it rigorously reproduces the reported "inconsistency" by running byte-identical input across two processes. Every cited reference spot-checked during this assessment matched the on-disk source exactly.

**Remaining gaps.** Only the human **review/acceptance** gate remains (2.0 h) — read the document, confirm it resolves the incident question, and sign off. There are no compilation, configuration, integration, deployment, or optimization tasks outstanding, because no application code was changed and the deliverable is a self-contained answer document.

**Critical path to production.** For a documentation deliverable, "production" is acceptance and use by the triage team. The critical path is a single step: **§1.6 step 1** (review & accept). The optional re-run (§1.6 step 2) is confirmatory only.

**Success metrics.** (1) All four questions answered with observed evidence — ✅. (2) Read-only scope preserved (tree clean, only the deliverable added) — ✅. (3) Autonomous validation green (compile clean; 23/23 tests; deliverable reproduced) — ✅. (4) The "inconsistency" explained and de-risked for monitoring — ✅.

**Production readiness assessment.** **Ready for human review.** The deliverable meets every AAP acceptance criterion and, notably, its empirical Q4 answer is *more accurate* than the AAP's own preliminary hypothesis (it discovered the `user_audit_log` side effect). Risk is uniformly Low. Recommendation: **accept and close the incident** after the §1.6 review step.

---

## 9. Development Guide

This guide explains how to reproduce the investigation and verify the deliverable. Commands are grouped into **host-runnable (tested)** and **canonical-per-image (documented)**.

### 9.1 System Prerequisites

- **Git + Git LFS** (repository uses LFS).
- **Docker** — recommended path; run the canonical image which bundles everything. *(Verified present on the assessment host: Docker 28.5.2.)*
- **OpenSSL** — for the DKIM key re-encode step. *(Verified present: OpenSSL 3.5.3.)*
- **Canonical runtime (inside the image):** Python **3.10** (image reports `3.10.18`), PostgreSQL, Redis.
- **Canonical image:** `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`.

> ⚠ The canonical runtime is **not** reproducible on a stock host: this host runs Python **3.13.7** (3.10 unavailable in apt), and `poetry`, `alembic`, `psql`, `redis-cli`, and the baked `venv/` are absent. Use the container image for any live re-execution.

### 9.2 Environment Setup

Canonical configuration is selected via `CONFIG=tests/test.env`:

```bash
# tests/test.env (canonical values — do not modify)
NOT_SEND_EMAIL=true
EMAIL_DOMAIN=sl.local
DB_URI=postgresql://test:test@localhost:15432/test
```

The image ships the venv (`/app/venv`), PostgreSQL (align listen port to `15432`), Redis, and the schema at Alembic head `32f25cbf12f6` (77 tables).

### 9.3 Dependency Installation

```bash
# Inside the canonical image: ZERO installs needed — the venv is complete.
# For a native reproduction (requires Python 3.10):
poetry install
```

### 9.4 Runtime & Schema (canonical-per-image)

```bash
# 1) Start backing services (system services in the canonical image)
service postgresql start && service redis-server start

# 2) Apply schema — a no-op here because the image is already at head:
CONFIG=tests/test.env poetry run alembic upgrade head   # reports head 32f25cbf12f6
```

### 9.5 Reproducing the Observation (canonical-per-image)

```bash
cd /app
export PYTHONPATH=/app
export CONFIG=tests/test.env            # mirrors tests/conftest.py:8-10
export GITHUB_ACTIONS_TEST=true

# DKIM: re-encode the SAME key material PKCS#8 -> PKCS#1 to a TEMP path
# (do NOT modify local_data/dkim.key)
mkdir -p /tmp/sl_investigate
openssl rsa -in /app/local_data/dkim.key -traditional -out /tmp/sl_investigate/dkim.key
export DKIM_PRIVATE_KEY_PATH=/tmp/sl_investigate/dkim.key

# Author a temp observe.py OUTSIDE the repo that:
#   - seeds a user + mailbox + fixed alias
#   - calls mail_sender.store_emails_instead_of_sending()
#   - builds an Envelope + EmailMessage and calls email_handler.handle(envelope, msg)
#   - snapshots all tables before/after and reads records back via the ORM
# Run it once PER PROCESS (twice) to demonstrate run-to-run stability:
/app/venv/bin/python /tmp/sl_investigate/observe.py > /tmp/sl_investigate/run.log  2>&1
/app/venv/bin/python /tmp/sl_investigate/observe.py > /tmp/sl_investigate/run2.log 2>&1
```

### 9.6 Verification

**Host-runnable (tested during this assessment — all pass):**

```bash
# Repository integrity
git rev-parse --abbrev-ref HEAD          # -> blitzy-c9405258-f45e-47e2-ba3d-e4946b0c4ea2
git rev-parse --short HEAD               # -> a92e5a41
[ -z "$(git status --porcelain)" ] && echo CLEAN        # -> CLEAN
git diff --name-only 2cd6ee77 HEAD       # -> blitzy/documentation/app_2cd6ee777f8c.md (only)
git diff --numstat  2cd6ee77 HEAD        # -> 689   0   <deliverable>

# Deliverable integrity
wc -l blitzy/documentation/app_2cd6ee777f8c.md          # -> 689
n=$(grep -c '^```' blitzy/documentation/app_2cd6ee777f8c.md); \
  [ $((n%2)) -eq 0 ] && echo "fences BALANCED ($n)"     # -> BALANCED (52)
grep -cE '^## Q[1-4] ' blitzy/documentation/app_2cd6ee777f8c.md   # -> 4

# Static citation spot-checks (confirm the doc's file:line refs)
sed -n '2p;51p' app/email/status.py     # -> E200 / E515 exact strings
sed -n '12,15p' app/log.py              # -> SL log format string
sed -n '62,65p' app/models.py           # -> ModelMixin id/created_at/updated_at
```

**Canonical-per-image (requires the container image):**

```bash
CONFIG=tests/test.env poetry run pytest tests/test_email_handler.py   # -> 23 passed
```

### 9.7 Troubleshooting

- **`Cannot create DKIM signature` / `ASN1FormatError (got 30 expecting 02)`** — the key is PKCS#8 but `dkimpy` needs PKCS#1. Re-encode with `openssl rsa -traditional` to a **temp** path and set `DKIM_PRIVATE_KEY_PATH`; never edit `local_data/dkim.key`.
- **`alembic` / psycopg2 "connection refused"** — PostgreSQL must listen on **`15432`** to match the `tests/test.env` `DB_URI` (align `postgresql.conf`; leave the data directory untouched).
- **`ModuleNotFoundError` importing app modules** — export `PYTHONPATH=/app` and set `CONFIG=tests/test.env` *before* importing any app module (mirrors `tests/conftest.py:8-10`).
- **Python version mismatch** — the application targets **3.10**; a host Python 3.13 will not run the app. Use the canonical image.

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---------|---------|
| Current branch / head | `git rev-parse --abbrev-ref HEAD` / `git rev-parse --short HEAD` |
| Clean-tree check | `[ -z "$(git status --porcelain)" ] && echo CLEAN` |
| Changed files vs. base | `git diff --name-only 2cd6ee77 HEAD` |
| Line delta vs. base | `git diff --numstat 2cd6ee77 HEAD` |
| Deliverable size | `wc -l blitzy/documentation/app_2cd6ee777f8c.md` |
| Fence-balance check | `grep -c '^\`\`\`' blitzy/documentation/app_2cd6ee777f8c.md` |
| Start services (image) | `service postgresql start && service redis-server start` |
| Apply schema (image) | `CONFIG=tests/test.env poetry run alembic upgrade head` |
| Run tests (image) | `CONFIG=tests/test.env poetry run pytest tests/test_email_handler.py` |
| DKIM re-encode | `openssl rsa -in /app/local_data/dkim.key -traditional -out /tmp/sl_investigate/dkim.key` |

### B. Port Reference

| Service | Port | Notes |
|---------|-----:|-------|
| PostgreSQL | 15432 | Canonical `tests/test.env` `DB_URI`; image listen-port aligned to this value |
| Redis | 6379 | `redis://localhost` (`MEM_STORE_URI`) |

### C. Key File Locations

| Path | Role |
|------|------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The deliverable** (answer document, 689 lines) |
| `email_handler.py` | Real SMTP entry point (`handle` @1945) and forward/reply/failure orchestration |
| `app/models.py` | ORM models (`ModelMixin` @62-65, `Contact.new_addr` @2008, `EmailLog` @2060, `MessageIDMatching` @3365, `UserAuditLog` @3829) |
| `app/email_utils.py` | `generate_reply_email` @1103 (reverse-alias format) |
| `app/mail_sender.py` | `store_emails_instead_of_sending` @102 (outgoing-message capture) |
| `app/log.py` | `SL` logger + format string (@12-15, @79) |
| `app/email/status.py` | `E200` @2, `E515` @51 |
| `app/contact_utils.py` | `create_contact` (@104 emits the `user_audit_log`) |
| `tests/test_email_handler.py` | Canonical `handle()` harness (@85) |
| `tests/test.env` | Canonical config |
| `migrations/` | Alembic schema (head `32f25cbf12f6`) |

### D. Technology Versions

| Component | Version | Source |
|-----------|---------|--------|
| Python | 3.10.18 (constraint `^3.10`) | `Dockerfile` `FROM python:3.10`; `pyproject.toml:61` |
| SQLAlchemy | 1.3.24 | `pyproject.toml` |
| arrow | 0.16.0 | `pyproject.toml` (`created_at`/`updated_at` `ArrowType`) |
| aiosmtpd | 1.4.2 | `Envelope` / `MailHandler` |
| redis | 4.6.0 | rate-limit / cache client |
| Flask | 1.1.2 | app-context framework |
| dkimpy | 1.0.5 | DKIM signing (requires PKCS#1) |
| alembic | 1.4.3 | schema migrations |
| psycopg2-binary | 2.9.3 | PostgreSQL driver |
| PostgreSQL | 15 (CI: 13) | backing database |
| Redis | 7 | backing cache / rate-limit |

### E. Environment Variable Reference

| Variable | Value / Purpose |
|----------|-----------------|
| `CONFIG` | `tests/test.env` — canonical config; set before importing app modules |
| `EMAIL_DOMAIN` | `sl.local` — the SL/alias domain (governs the reverse-alias domain) |
| `NOT_SEND_EMAIL` | `true` — suppress outbound mail |
| `DB_URI` | `postgresql://test:test@localhost:15432/test` |
| `PYTHONPATH` | `/app` — repository root on the import path |
| `DKIM_PRIVATE_KEY_PATH` | temp PKCS#1 key path (environment-only DKIM fix) |
| `GITHUB_ACTIONS_TEST` | `true` — matches the CI invocation topology |

### F. Developer Tools Guide

- **Outgoing-message capture** — `mail_sender.store_emails_instead_of_sending()` (`app/mail_sender.py:102`) makes the real `send()` retain the fully transformed `EmailMessage` in `get_stored_emails()` (`:108`) instead of transmitting it. This is how the `From` (Q3) and `Message-ID` (Q2) values are read with byte fidelity and zero external effects.
- **Real entry point** — build an `aiosmtpd` `Envelope` + `EmailMessage` and call `email_handler.handle(envelope, msg)` (the pattern used by `tests/test_email_handler.py:85`) to exercise the production path without a live SMTP socket.
- **All-tables DB diff** — snapshot row counts of all public tables before and after a single `handle()` call to prove exactly which tables gain rows (used to establish the Q4 three-row answer empirically).

### G. Glossary

| Term | Meaning |
|------|---------|
| **Reverse-alias** | The `reply_email` address at `sl.local` used as the outgoing `From` so the sender's real address is masked; replying to it relays back to the original sender. |
| **SL Message-ID** | A SimpleLogin-minted `Message-ID` produced by `make_msgid(str(email_log.id), <alias domain>)` — **only** on the reply path; embeds the `EmailLog.id`. |
| **Forward vs. Reply** | Forward = inbound mail to an alias delivered to the mailbox (original `Message-ID` preserved). Reply = mailbox → reverse-alias relayed back to the sender (SL `Message-ID` minted). |
| **`E200` / `E515`** | SMTP status strings: `250 Message accepted for delivery` (success) / `550 SL E515 Email not exist` (non-existent alias). |
| **`ModelMixin`** | Base providing autoincrement `id`, `created_at` (`ArrowType`, default `arrow.utcnow`, UTC), and `updated_at` to all models. |
| **PKCS#8 / PKCS#1** | RSA key encodings; `dkimpy 1.0.5` requires PKCS#1 (`BEGIN RSA PRIVATE KEY`), addressed via an environment-only re-encode. |
| **Category A / B / C** | Variability classification: A = varies despite identical input (ids, timestamps, fresh mints); B = stable structure; C = changes only when the input changes. |

---

*Prepared by the Blitzy autonomous assessment agent. Completion (94.7%) reflects AAP-scoped work plus the path-to-production human-review gate. Brand colors: Completed **#5B39F3**, Remaining **#FFFFFF**.*