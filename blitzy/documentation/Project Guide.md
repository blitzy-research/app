# Blitzy Project Guide — SimpleLogin Runtime-Behavior Q&A Investigation (`app_2cd6ee777f8c`)

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a single, evidence-grounded Markdown document — `blitzy/documentation/app_2cd6ee777f8c.md` — that answers three runtime-behavior questions about the open-source **SimpleLogin** email-alias service by *executing the real code paths* inside the canonical Docker container, not by reading alone. **Q1** traces mailbox verification-code enforcement under repeated wrong attempts; **Q2** traces the background-`Job` lifecycle and its error behavior; **Q3** traces the VERP bounce-address format and how bounce handling differs by direction (forward vs reply). The task is strictly **read-only**: no source, configuration, test, migration, or dependency file is modified — the only artifact added is the answer document. Target consumers are engineers and reviewers who need exact, observed behavior with `file:line` references and honest observed-vs-inferred labeling.

### 1.2 Completion Status

```mermaid
%%{init: {"theme":"base","themeVariables":{"pie1":"#5B39F3","pie2":"#FFFFFF","pieStrokeColor":"#B23AF2","pieStrokeWidth":"2px","pieOuterStrokeColor":"#B23AF2","pieOuterStrokeWidth":"2px","pieTitleTextSize":"16px","pieSectionTextSize":"13px","pieLegendTextSize":"13px"}}}%%
pie showData title Completion Status — 90.0% Complete
    "Completed Work (AI)" : 63
    "Remaining Work" : 7
```

| Metric | Hours |
|--------|-------|
| **Total Hours** | **70** |
| Completed Hours (AI + Manual) | 63 (63 AI + 0 Manual) |
| Remaining Hours | 7 |
| **Percent Complete** | **90.0%** |

> Completion is computed with the PA1 AAP-scoped, hours-based method: `63 / (63 + 7) = 90.0%`. It measures how much of the Agent-Action-Plan-scoped work plus path-to-production was completed autonomously. Per policy the value is capped below 100% before human review; the residual 7 hours are genuine human review/verification/merge work described in Sections 1.6, 2.2, and the risk assessment.

### 1.3 Key Accomplishments

- [x] **Single-file, additive deliverable produced** at the mandated path/name `blitzy/documentation/app_2cd6ee777f8c.md` (8,775 lines, ~655 KB, valid UTF-8) — the only file changed across the entire branch.
- [x] **Q1 answered from runtime observation** — reproduced the `MailboxActivation.tries` progression `0 → 1 → 2 → 3`, terminal code-invalidation (`clear_activation_codes_for_mailbox` + `CannotVerifyError`), the 15-minute expiry guard, and the key nuance that `/dashboard/mailbox_verify` returns **HTTP 302, never 429** (no rate-limiter on the route).
- [x] **Q2 answered with a substantive finding** — reproduced the normal lifecycle `ready(0) → taken(1) → done(2)` and proved the error path: `job_runner.py` has **no `try/except`** around `process_job`, so a failing job crashes the runner (exit code 1), leaving the job stuck at `taken(1)` with `attempts` incremented, **never `error(3)`**.
- [x] **Q3 answered with byte-level precision** — captured the exact VERP format `{sl}.{base32(payload)}.{base32(hmac-sig)}@{domain}` (sha3-224, 8-byte signature, no `=` padding), the HMAC-not-expiry validity semantics, round-trip decode + tamper→`None`, and the direction-dependent dispatch (**forward → E211** keyed on `mailbox.email`; **reply → E212** keyed on `contact.website_email` + user `Notification`).
- [x] **Investigation ran through canonical entry points** inside the required GHCR container (Python 3.10.18 venv, Postgres 15.13 / 77 tables, Redis), with every scenario executed **twice** for stability.
- [x] **Rigorous evidence discipline** — 148 `[OBSERVED]` labels, a 4-label provenance scheme (`OBSERVED` / `SOURCE-VERIFIED` / `INFERRED` / `NON-CANONICAL`) exceeding the AAP's binary requirement, 16 Evidence blocks, 3 `file:line` reference tables, and a 36-row coverage-pass matrix.
- [x] **Read-only mandate fully preserved** — net-zero database (all-77-table fingerprint identical before/after), all temporary scripts removed, working tree clean.
- [x] **Five-commit QA cycle** resolving ~48 review/acceptance findings; Final Validator reported all production-readiness gates PASS with zero discrepancies.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| One Q2 sub-behavior — the end-to-end multi-restart *retry-to-exhaustion* cycle — is `[INFERRED]`, not observed end-to-end | Low. Its boundaries (no-retry-in-process, 30-min re-selection gate, 5-attempt ceiling) are all OBSERVED; only the full loop was not run to exhaustion. Honestly labeled in the document. | Human reviewer / QA | ~2h |
| Strong internal-behavior claims across three subsystems await subject-matter confirmation | Medium. Documentation-only impact (a subtle misread would misinform a reader). Two-run stability + zero-discrepancy validation already reduce the risk. | SME reviewer | ~4h |

> There are **no compilation, build, or test failures** to resolve — the deliverable is a documentation artifact and the referenced pytest harness passes (exit code 0). The two items above are verification/closure tasks, not defects.

### 1.5 Access Issues

| System / Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-------------------|----------------|-------------------|-------------------|-------|
| Canonical GHCR image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` | Container registry pull | Exact-literal reproduction of observed values (e.g., VERP strings) requires this image + `tests/test.env`; the host checkout (Python 3.13, no deps) is non-canonical | Available — used successfully during the autonomous investigation (image id `sha256:ea242796bbce…`) | DevOps / reviewer |

No blocking access issues were identified. Repository write access, container runtime, Postgres, and Redis were all available and exercised during the autonomous investigation.

### 1.6 Recommended Next Steps

1. **[High]** Perform a subject-matter accuracy review of the three answers against the cited `file:line` locations — prioritize the Q2 "no `try/except` → never `error(3)`" finding and the Q3 VERP format + E211/E212 direction dispatch. (~4h)
2. **[Medium]** Close the single `[INFERRED]` item by running the Q2 end-to-end multi-restart retry-to-exhaustion cycle in the canonical container to convert the one PARTIAL coverage row to PASS (or record the observed distribution if it differs). (~2h)
3. **[Low]** Review and merge the additive documentation PR after confirming single-file scope (`git diff --name-status` = `A blitzy/documentation/app_2cd6ee777f8c.md`). (~1h)
4. **[Low · out of scope]** Optionally route the surfaced SimpleLogin findings — the Q2 runner-crash reliability behavior and the Q3 security-relevant behaviors (extra-recipient bounce handling, VERP-as-`MAIL FROM`, replay, DEBUG code exposure) — to the SimpleLogin maintainers as separate work. The AAP explicitly mandates document-only/read-only, so these were **not** fixed here.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|-------|-------------|
| Q1 — Mailbox verification enforcement (investigation + write-up) | 10 | Drove the canonical `GET /dashboard/mailbox_verify` route via a real `/auth/login` session; observed `tries` `0→1→2→3`, terminal invalidation, 15-min expiry guard, malformed/missing-input branches, and HTTP 302 (never 429). Six Evidence blocks (A–F). |
| Q2 — Background-`Job` lifecycle + error path (investigation + write-up) | 12 | Ran the real `job_runner.py` drain loop; observed `ready(0)→taken(1)→done(2)`; proved the no-`try/except` crash (exit 1, job stuck `taken`, never `error(3)`); mapped retry eligibility, cleanup, and edge branches (unknown-name, NULL-payload, concurrency, live poller); disclosed the DKIM PKCS#1 prerequisite. Five Evidence blocks (A–E). |
| Q3 — VERP format + direction-dependent bounce dispatch (investigation + write-up) | 16 | Captured the exact VERP byte format and decode semantics; drove `email_handler.handle()` for both directions (forward E211 / reply E212); demonstrated auto-disable and auto-reply edges plus five adversarial sub-scenarios (E.1–E.5). Five Evidence blocks (A–E). |
| Canonical environment setup + proof | 5 | Stood up the GHCR container (Python 3.10.18 venv, 182 pinned deps == `poetry.lock`), Postgres 15.13 (77 tables), Redis; captured three environment proofs and a boundary disclosure. |
| Evidence-capture discipline + two-run stability methodology | 5 | Enforced unedited output + exact command + `file:line` for every claim; two-run semantic-diff stability; the 4-label provenance scheme (OBSERVED/SOURCE-VERIFIED/INFERRED/NON-CANONICAL). |
| Coverage pass (row-level matrices + cross-cutting dimensions) | 3 | Built the consolidated coverage pass: Q1 = 9, Q2 = 12, Q3 = 15 named sub-parts + 9 cross-cutting dimensions, each with explicit status and provenance. |
| Read-only guarantee + cleanup + net-zero verification | 3 | All-77-table net-zero checker (fingerprint identical before/after), exact-filename refused-email deletion, temp-script removal, single-file git-scope confirmation. |
| Document assembly + QA iteration (5 commits, ~48 findings) | 9 | Assembled the deliverable at the mandated path; iterated through five commits resolving ~48 review/acceptance findings; ensured balanced code fences and valid UTF-8. |
| **Total** | **63** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|-------|----------|
| SME technical-accuracy review of the Q1/Q2/Q3 answers against cited `file:line` (path-to-production) | 4 | High |
| Close the single `[INFERRED]` item — run the Q2 end-to-end multi-restart retry-to-exhaustion cycle (AAP sub-part) | 2 | Medium |
| PR review + merge of the additive documentation (path-to-production) | 1 | Low |
| **Total** | **7** | |

> Out-of-scope follow-ups (routing the Q2 reliability finding and Q3 security-relevant behaviors to SimpleLogin maintainers) are **0 hours against this project** — the AAP mandates read-only/document-only and forbids fixing observed behavior. They are noted in Sections 1.6 and 6 for stakeholder awareness only.

---

## 3. Test Results

All entries originate from Blitzy's autonomous validation logs for this project. Because the deliverable is a documentation artifact, correctness was established by **validation-by-running** — driving the real entry points and comparing observed output to the documented claims — supplemented by the project's own pytest harness. Every runtime scenario was executed **twice** for stability with zero discrepancies.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|------------|-------|
| Unit test harness (Q2 job runner + cleanup) | pytest 8.4.1 (Python 3.10.18) | 2 | 2 | 0 | N/A (coverage gate intentionally disabled via `-o addopts=""`) | `test_get_jobs_to_run` + `test_cleanup_old_jobs`, run against a disposable clone DB; exit code 0. Tagged `[NON-CANONICAL]` supporting in the deliverable (canonical proof is the live daemon). |
| Runtime behavioral verification — Q1 | Observation scripts via real `GET /dashboard/mailbox_verify` | 6 | 6 | 0 | N/A | Evidence A–F (enforcement fn, canonical HTTP, malformed inputs, branch matrix, method/CSRF/headers, limiter-ON no-429); each run twice. |
| Runtime behavioral verification — Q2 | Observation scripts via real `job_runner.py` daemon | 4 | 4 | 0 | N/A | Evidence A/B/C/E (normal lifecycle, error-path crash, retry eligibility + cleanup, edge branches); each run twice. |
| Runtime behavioral verification — Q3 | Observation scripts via real `email_handler.handle()` / `generate_verp_email` | 5 | 5 | 0 | N/A | Evidence A–E incl. adversarial E.1–E.5 (VERP format/decode, direction dispatch, auto-reply, auto-disable, adversarial); each run twice. |
| **Totals** | — | **17** | **17** | **0** | — | 15 runtime scenarios (each ×2 = 30 runs) + 2 unit tests; zero discrepancies across all runs. |

> **Integrity note.** No bespoke unit tests are authored for a `.md` deliverable; that is correct for a read-only Q&A task. The "tests" above are the autonomous validation runs recorded in the deliverable's Evidence blocks plus the referenced canonical pytest harness — all sourced from Blitzy's own execution logs.

---

## 4. Runtime Validation & UI Verification

**Runtime validation** — all three canonical subsystems were driven through their real entry points inside the GHCR container and behaved as documented:

- ✅ **Q1 — Dashboard/HTTP mailbox-verify flow** (`GET /dashboard/mailbox_verify` → `mailbox_utils.verify_mailbox_code`): Operational. Reproduced `tries 0→1→2→3`, terminal invalidation, 15-minute expiry guard, and **HTTP 302 on every submission (never 429)** with the rate-limit framework enabled.
- ✅ **Q2 — Background-`Job` daemon** (`job_runner.py` drain loop): Operational for the normal lifecycle `ready(0)→taken(1)→done(2)`, `attempts 0→1`.
- ⚠ **Q2 — Error path (documented SimpleLogin finding, not a deliverable defect):** a failing `process_job` propagates uncaught, **the runner process exits (code 1)**, and the job is left at `taken(1)` with `attempts` incremented — **never `error(3)`**. Reproduced deterministically; documented as a finding per the read-only mandate (not fixed).
- ✅ **Q3 — SMTP bounce pipeline** (`email_handler.handle()` + `generate_verp_email` / `get_verp_info_from_email`): Operational. VERP address generated/round-tripped byte-identically; tampered address → `None`; direction-dependent dispatch returned **E211** (forward) and **E212** (reply) with the corresponding `Bounce` / `RefusedEmail` / `EmailLog` / `Notification` state changes.

**UI verification:** ✅ Not applicable — there is **no user-facing UI in scope**. The task is a backend runtime investigation; the only HTTP surface exercised is the `/dashboard/mailbox_verify` route, verified at the protocol level (status codes/headers), not visually.

**Environment health:** ✅ Postgres 15.13 online (77-table schema), ✅ Redis responding (`PONG`), ✅ all three subsystem modules import cleanly under `/app/venv/bin/python`.

---

## 5. Compliance & Quality Review

The deliverable is cross-mapped to the governing rule set ("SWE-AtlasQnA-Repo", AAP §0.7) and Blitzy quality benchmarks. Fixes applied during autonomous validation are noted; there are no outstanding compliance items.

| # | AAP Rule / Quality Benchmark | Status | Progress | Evidence / Notes |
|---|------------------------------|--------|----------|------------------|
| 1 | Single deliverable at fixed location/name (`blitzy/documentation/app_2cd6ee777f8c.md`) | ✅ Pass | 100% | `git diff --name-status` = `A blitzy/documentation/app_2cd6ee777f8c.md`. |
| 2 | Investigate by running the code first, then write | ✅ Pass | 100% | 15 runtime scenarios + pytest harness executed through real entry points; 148 `[OBSERVED]` labels. |
| 3 | Use the canonical entry point (no bypass/mock/stub) | ✅ Pass | 100% | Q1 HTTP route via `/auth/login`; Q2 real daemon; Q3 real `handle()`. Non-canonical supporting uses all disclosed with canonical proof alongside. |
| 4 | Reproduce exact behavior; confirm stability (≥ 2 runs) | ✅ Pass | 100% | Every Evidence block run twice with a semantic two-run diff classifying incidental lines. |
| 5 | Exercise every condition (primary + secondary/error/edge; before/intermediate/after) | ✅ Pass | 100% | Lockout, expiry, malformed inputs (Q1); unknown-name, NULL-payload, concurrency (Q2); auto-reply, auto-disable, adversarial E.1–E.5 (Q3). |
| 6 | Show actual, unedited output + the exact command for every claim | ✅ Pass | 100% | 320 balanced code-fence lines carrying verbatim command output (trailing whitespace/tabs preserved inside fences per the unedited-output mandate). |
| 7 | Be exact and grounded (`file:line` refs; name the function; label inferred) | ✅ Pass | 100% | 3 per-question `file:line` tables; citation path-prefixes normalized in commit `2a2a78e3` (finding M1). |
| 8 | Answer every part and every named item + coverage pass | ✅ Pass | 100% (1 sub-part PARTIAL) | 36-row coverage matrix; one honestly-labeled PARTIAL/`[INFERRED]` row (Q2 retry-to-exhaustion). |
| 9 | Read-only scope; temp scripts removed; repo byte-clean | ✅ Pass | 100% | Net-zero DB (all-77-table fingerprint identical), temp scripts deleted, working tree clean, only the deliverable added. |
| 10 | Well-formed, reviewable Markdown | ✅ Pass | 100% | Valid UTF-8; 320 fence lines (balanced); prose free of stray trailing whitespace. |

**Fixes applied during autonomous QA (5 commits):** initial draft → rewrite resolving 17 review findings → 4 findings → 27 acceptance findings → citation path-prefix (M1) + bare-basename (I1) normalization. Net: ~48 findings resolved, zero remaining.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| T1 — Q2 end-to-end retry-to-exhaustion cycle is `[INFERRED]`, not run end-to-end | Technical | Low | Low | Run the multi-restart cycle in the canonical container (task R2 / HT-2); boundaries already OBSERVED | Open (mitigated) |
| T2 — Strong internal-behavior claims across 3 subsystems could contain a subtle misread | Technical | Medium | Low | SME accuracy review (task R1 / HT-1); two-run stability + zero-discrepancy validation already reduce it | Open (pending review) |
| T3 — Observed literal values (VERP strings, `EMAIL_DOMAIN='sl.local'`) are tied to `tests/test.env` | Technical / Config | Low | Medium | Config-drift disclosure (§E1 of the deliverable) documents defaults vs custom values | Mitigated |
| S1 — Document surfaces pre-existing SimpleLogin security-relevant behaviors (extra-recipient, VERP-as-`MAIL FROM`, replay, DEBUG code exposure) | Security | Informational | N/A | No vulnerability introduced (additive doc, no secrets reproduced); optionally route findings to maintainers | Informational / out-of-scope-to-fix |
| O1 — Exact reproduction depends on the GHCR image + `tests/test.env`; temp scripts were deleted per mandate | Operational | Low | Low | Image ref + exact commands embedded verbatim in the deliverable preserve reproducibility | Mitigated |
| O2 — Documented Q2 behavior (runner crashes; job never reaches `error(3)`; retry only after restart) is a reliability characteristic of **SimpleLogin itself** | Operational | Medium (for SimpleLogin) | N/A | AAP scopes this as document-only; route to maintainers as separate work | Documented, intentionally not fixed |
| I1 — Purely additive doc; no imports/config/interface/schema/migration changes | Integration | Negligible | Low | Standard PR review + merge (task R3 / HT-3) | Mitigated |

**Overall:** No High or Critical risks. The highest is T2 (Medium, technical accuracy), fully addressed by the High-priority SME review. Every risk maps to a remaining task (R1/R2/R3) or is transparently disclosed / explicitly out of scope.

---

## 7. Visual Project Status

**Project hours breakdown** (Completed = Dark Blue `#5B39F3`, Remaining = White `#FFFFFF`):

```mermaid
%%{init: {"theme":"base","themeVariables":{"pie1":"#5B39F3","pie2":"#FFFFFF","pieStrokeColor":"#B23AF2","pieStrokeWidth":"2px","pieOuterStrokeColor":"#B23AF2","pieOuterStrokeWidth":"2px","pieTitleTextSize":"16px","pieSectionTextSize":"13px","pieLegendTextSize":"13px"}}}%%
pie showData title Project Hours Breakdown (Total 70h)
    "Completed Work" : 63
    "Remaining Work" : 7
```

**Remaining hours by category** (sums to 7h, matching Sections 1.2 and 2.2):

```mermaid
%%{init: {"theme":"base","themeVariables":{"pie1":"#5B39F3","pie2":"#B23AF2","pie3":"#A8FDD9","pieStrokeColor":"#333333","pieStrokeWidth":"1px","pieTitleTextSize":"15px","pieSectionTextSize":"13px","pieLegendTextSize":"13px"}}}%%
pie showData title Remaining Work by Category (7h)
    "SME accuracy review [High]" : 4
    "Close [INFERRED] Q2 item [Medium]" : 2
    "PR review + merge [Low]" : 1
```

> **Integrity check.** "Remaining Work" = **7h** in the pie above equals the Remaining Hours in Section 1.2 and the sum of the Section 2.2 Hours column. "Completed Work" = **63h** equals Completed Hours in Section 1.2 and the sum of the Section 2.1 Hours column. `63 + 7 = 70` = Total Hours.

---

## 8. Summary & Recommendations

**Achievements.** The autonomous work is essentially complete: a single, additive, evidence-grounded document answers all three runtime-behavior questions from real execution inside the canonical container. The investigation is unusually rigorous — 148 observed data points, a four-label provenance scheme, two-run stability for every scenario, a 36-row coverage matrix, and a verified net-zero, read-only footprint (only `blitzy/documentation/app_2cd6ee777f8c.md` was added). The Final Validator reported all production-readiness gates PASS with zero discrepancies.

**Remaining gaps.** Seven hours of human work remain, all verification/merge rather than defect-fixing: a subject-matter accuracy review of the three answers (4h), closing the single honestly-labeled `[INFERRED]` item — the Q2 end-to-end retry-to-exhaustion cycle (2h), and PR review + merge (1h).

**Critical path to production.** For a documentation deliverable, "production" means *reviewed and merged*. The path is therefore: (1) SME accuracy review → (2) optional `[INFERRED]`-item closure → (3) merge. There is no build, deployment, infrastructure, or CI/CD work because the change is purely additive and read-only.

**Success metrics.** Single-file scope preserved (✅); every named sub-part of Q1/Q2/Q3 answered (✅, with one PARTIAL row transparently labeled); referenced pytest harness passes (✅ exit 0); repository byte-clean apart from the deliverable (✅).

**Production-readiness assessment.** **90.0% complete.** The deliverable is accurate, fully cited, well-formed, and committed; it is ready for human review. The only substantive judgment call remaining is subject-matter confirmation of the strongest claims (notably the Q2 error-path finding), which is why completion is held at 90.0% rather than higher. Recommended disposition: **approve for review, then merge** after the High-priority accuracy check.

| Metric | Value |
|--------|-------|
| AAP-scoped completion | 90.0% (63h / 70h) |
| Files changed vs base | 1 (added) |
| Review findings resolved | ~48 across 5 commits |
| Open defects | 0 |
| Remaining human hours | 7 |

---

## 9. Development Guide

This guide explains how to view the deliverable, reproduce the runtime investigation, and verify the read-only guarantee. Commands are split into **host-side** (runnable in any checkout; tested during this assessment) and **canonical-container** (require the GHCR image; the source of all observed values).

### 9.1 System Prerequisites

- **Docker** (Engine 28.x verified) — to run the canonical container.
- **Git** (2.51 verified) + Git LFS — to inspect the branch and confirm scope.
- **Canonical image:** `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (image id `sha256:ea242796bbce…`), which bundles **Python 3.10.18**, the project virtual-env at `/app/venv`, **PostgreSQL 15.13**, and **Redis**.
- A Markdown viewer for the deliverable (optional).
- **Note:** the local host checkout (system Python 3.13, no dependencies installed) is **non-canonical** — `python3 -c "import flask"` fails there. Use `/app/venv/bin/python` inside the container for any reproduction.

### 9.2 Environment Setup (canonical container)

```bash
# 1. Confirm the running canonical container and its image
docker inspect sl_canon --format 'state={{.State.Status}} image={{.Config.Image}}'

# 2. Verify the project venv and pinned dependencies (NOT bare python3)
docker exec sl_canon bash -lc '/app/venv/bin/python --version; \
  /app/venv/bin/python -c "import flask, sqlalchemy, aiosmtpd, flanker; \
  print(flask.__version__, sqlalchemy.__version__, aiosmtpd.__version__)"'

# 3. Verify data services and the constants the answers depend on
docker exec sl_canon bash -lc '
  service postgresql status | head -1; redis-cli ping;
  export CONFIG=tests/test.env PYTHONPATH=/app DB_URI="postgresql://test:test@localhost:5432/test";
  /app/venv/bin/python -c "from app import config; from app.mailbox_utils import MAX_ACTIVATION_TRIES; \
  print(\"MAX_ACTIVATION_TRIES\", MAX_ACTIVATION_TRIES, \"| JOB_MAX_ATTEMPTS\", config.JOB_MAX_ATTEMPTS, \
  \"| VERP_PREFIX\", repr(config.VERP_PREFIX))"'
```

Expected: Python 3.10.18; `flask 1.1.2 / sqlalchemy 1.3.24 / aiosmtpd 1.4.2`; Postgres online; Redis `PONG`; `MAX_ACTIVATION_TRIES 3 | JOB_MAX_ATTEMPTS 5 | VERP_PREFIX 'sl'`.

### 9.3 Dependency Installation

**None required.** This is a read-only task; the container's `/app/venv` already holds the 182 pinned dependencies (`== poetry.lock`). Do **not** `pip install` or `poetry install` anything — the environment is fixed and the source tree must remain unchanged.

### 9.4 Viewing the Deliverable (host-side — tested)

```bash
cd <repo-root>
wc -l blitzy/documentation/app_2cd6ee777f8c.md          # -> 8775
iconv -f utf-8 -t utf-8 blitzy/documentation/app_2cd6ee777f8c.md >/dev/null && echo "VALID UTF-8"
grep -c '```' blitzy/documentation/app_2cd6ee777f8c.md   # -> 320 (even => balanced)
# Jump to a specific answer:
grep -nE '^## Q[123] ' blitzy/documentation/app_2cd6ee777f8c.md
```

### 9.5 Reproducing the Investigation (canonical container)

Run everything via the venv interpreter with the canonical env exported:

```bash
docker exec sl_canon bash -lc '
  cd /app
  export CONFIG=tests/test.env PYTHONPATH=/app DB_URI="postgresql://test:test@localhost:5432/test"
  # --- Q3 (self-contained, NET-ZERO: encode/decode only) ---
  /app/venv/bin/python - <<PY
from app.email_utils import generate_verp_email, get_verp_info_from_email
from app.models import VerpType
addr_f = generate_verp_email(VerpType.bounce_forward, 987654, "contact-domain.com")
addr_r = generate_verp_email(VerpType.bounce_reply, 987654, None)
print("forward:", addr_f)
print("reply:  ", addr_r)
print("decode(forward):", get_verp_info_from_email(addr_f))
print("decode(reply):  ", get_verp_info_from_email(addr_r))
PY'
```

Expected (values are `tests/test.env`-dependent): a `sl.<base32>.<base32>@<domain>` address for each direction; `decode(...)` returns `(<VerpType.bounce_forward: 0>, 987654)` and `(<VerpType.bounce_reply: 1>, 987654)`.

- **Q1** — exercise `GET /dashboard/mailbox_verify` through a real `/auth/login` session (Flask test client), submit an incorrect code four times, and observe `MailboxActivation.tries` progress `0→1→2→3`, then code invalidation. Every response is HTTP **302**, never 429. (See the deliverable's *Q1 Evidence B*.)
- **Q2** — create a `Job`, run the real `job_runner.py` drain loop; for the error path, register a handler that raises and observe the process exit (code 1) with the job left at `state=taken(1)`, `attempts=1`. The success path requires the git-tracked **PKCS#1** `local_data/dkim.key`. (See *Q2 Evidence A/B*.)
- **Q3 (full)** — drive `email_handler.handle()` with a `multipart/report` DSN to the VERP address for each direction and read back the `Bounce` / `RefusedEmail` / `EmailLog` / `Notification` rows and the E211/E212 status. (See *Q3 Evidence B*.)

### 9.6 Verifying the Read-Only Guarantee (host-side — tested)

```bash
git status --porcelain                                   # -> empty (clean tree)
git diff --name-status 2cd6ee777f8c..HEAD                # -> A blitzy/documentation/app_2cd6ee777f8c.md
git diff --name-only 2cd6ee777f8c..HEAD | grep -vc '^blitzy/documentation/'   # -> 0
```

### 9.7 Troubleshooting

- **`ModuleNotFoundError: No module named 'flask'`** — you used bare `python3`. Use `/app/venv/bin/python` inside the container.
- **Q2 success path leaves the job `taken(1)`** — the DKIM key is not PKCS#1. Use the git-tracked `local_data/dkim.key`; a regenerated **PKCS#8** key fails during `add_dkim_signature` before the local-send gate.
- **VERP strings differ from the document** — literal values depend on `tests/test.env` (`VERP_PREFIX='sl'`, `EMAIL_DOMAIN='sl.local'`, `VERP_EMAIL_SECRET`). The *format* is stable; the encoded bytes vary with config and the current minute.
- **`pytest` reports a coverage failure** — run the Q2 harness with `-o addopts=""` to drop the repo's `--cov` gate, against a disposable clone DB (`CREATE DATABASE test_obs TEMPLATE test`), since `test_cleanup_old_jobs` wipes the `job` table.
- **Two runs differ on a few lines** — expected; timestamps/PIDs/temp paths are process-incidental. The deliverable's two-run diffs classify these explicitly.

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---------|---------|
| Confirm read-only scope | `git diff --name-status 2cd6ee777f8c..HEAD` |
| Confirm clean tree | `git status --porcelain` |
| Deliverable size | `wc -l blitzy/documentation/app_2cd6ee777f8c.md` |
| UTF-8 validity | `iconv -f utf-8 -t utf-8 <file> >/dev/null && echo VALID` |
| Fence balance | `grep -c '```' <file>` |
| Inspect container | `docker inspect sl_canon --format '{{.State.Status}} {{.Config.Image}}'` |
| Venv interpreter | `docker exec sl_canon /app/venv/bin/python --version` |
| Q2 pytest harness | `/app/venv/bin/python -m pytest tests/jobs/test_job_runner.py tests/tasks/test_cleanup_old_jobs.py -o addopts="" -v` |

### B. Port Reference

| Service | Port | Notes |
|---------|------|-------|
| PostgreSQL | 5432 | `postgresql://test:test@localhost:5432/test` (canonical container) |
| Redis | 6379 | Backing store for rate-limiters; answers `PONG` |
| Flask (dev, if launched) | 7777 | SimpleLogin default web port; not required for this investigation |

### C. Key File Locations

| Path | Role |
|------|------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The sole deliverable** (Q&A answer document) |
| `app/mailbox_utils.py` | Q1 — `verify_mailbox_code`, `MAX_ACTIVATION_TRIES` |
| `app/dashboard/views/mailbox.py` | Q1 — `/mailbox_verify` route |
| `job_runner.py` | Q2 — drain loop, `get_jobs_to_run`, `process_job` |
| `tasks/cleanup_old_jobs.py` | Q2 — end-of-life job deletion |
| `app/email_utils.py` | Q3 — `generate_verp_email`, `get_verp_info_from_email` |
| `app/email_handler.py` | Q3 — `handle`, `handle_bounce`, forward/reply phases |
| `app/models.py` | `MailboxActivation`, `Job`/`JobState`, `VerpType`/`EmailLog` |
| `app/config.py` | Constants: `JOB_MAX_ATTEMPTS`, `VERP_PREFIX`, bounce prefixes |

### D. Technology Versions

| Component | Version | Source |
|-----------|---------|--------|
| Python (canonical) | 3.10.18 | `/app/venv` |
| Flask | 1.1.2 | `poetry.lock` |
| SQLAlchemy | 1.3.24 | `poetry.lock` |
| aiosmtpd | 1.4.2 | `poetry.lock` |
| flanker | 0.9.11 | `poetry.lock` |
| psycopg2-binary | 2.9.3 | `poetry.lock` |
| redis | 4.6.0 | `poetry.lock` |
| PostgreSQL | 15.13 | container |
| pytest | 8.4.1 | container venv |
| VERP HMAC algo | sha3-224 | `app/email_utils.py:69` |

### E. Environment Variable Reference

| Variable | Value (canonical) | Purpose |
|----------|-------------------|---------|
| `CONFIG` | `tests/test.env` | Selects the test configuration profile |
| `PYTHONPATH` | `/app` | Makes `app`, `job_runner`, `email_handler` importable |
| `DB_URI` | `postgresql://test:test@localhost:5432/test` | Postgres connection for observed state |
| `VERP_PREFIX` | `sl` | Prefix of generated VERP bounce addresses |
| `EMAIL_DOMAIN` | `sl.local` | Domain in observed addresses |
| `JOB_MAX_ATTEMPTS` | `5` | Retry ceiling for `Job` re-selection |
| `JOB_TAKEN_RETRY_WAIT_MINS` | `30` | Stale-`taken` re-selection window |
| `VERP_MESSAGE_LIFETIME` | `432000` (5 days) | Future-directed clock-sanity bound for VERP tokens |

### F. Developer Tools Guide

- **Docker** — run/inspect the canonical container; all observed values must come from here.
- **`docker exec … /app/venv/bin/python`** — the only correct interpreter; bare `python3` lacks dependencies.
- **`psql` / `redis-cli`** — inspect Postgres state and confirm Redis health.
- **`pytest`** — run the canonical Q2 harness (with `-o addopts=""` and a disposable clone DB).
- **`git`** — confirm the single-file, read-only scope before and after review.

### G. Glossary

| Term | Meaning |
|------|---------|
| **VERP** | Variable Envelope Return Path — a per-recipient return-path/bounce address that encodes the originating message so bounces are auto-identified. |
| **HMAC (sha3-224)** | Keyed hash used to sign the VERP payload (first 8 bytes), making the address tamper-resistant. |
| **DSN** | Delivery Status Notification — the `multipart/report` bounce message. |
| **`MailboxActivation`** | Row holding a mailbox's verification `code` and `tries` counter (Q1). |
| **`Job` / `JobState`** | Background-task row and its state enum: `ready(0)`, `taken(1)`, `done(2)`, `error(3)` (Q2). |
| **`EmailLog`** | Per-message record whose `is_reply` flag selects the bounce direction (Q3). |
| **Forward vs Reply phase** | Direction of the original message; determines the `Bounce` key and SMTP status (E211 vs E212). |
| **Net-zero** | The investigation left the database byte-identical (verified by an all-77-table fingerprint). |
| **Canonical entry point** | The real code path (HTTP route / daemon loop / SMTP handler), as opposed to a mock, stub, or direct-helper call. |
| **`[OBSERVED]` / `[SOURCE-VERIFIED]` / `[INFERRED]` / `[NON-CANONICAL]`** | The four provenance labels used to grade every claim in the deliverable. |