# Blitzy Project Guide
### SimpleLogin Runtime Investigation — Web Server, Email Handler, and Job Runner

> **Deliverable type:** Documentation (runtime-grounded Q&A investigation report)
> **Brand legend:** <span style="color:#5B39F3">■</span> **Completed / AI Work — Dark Blue `#5B39F3`** &nbsp;|&nbsp; <span style="color:#B23AF2">■</span> Remaining / Not Completed — White `#FFFFFF` (outlined) &nbsp;|&nbsp; Headings/Accents `#B23AF2` &nbsp;|&nbsp; Highlight `#A8FDD9`

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivers one evidence-grounded Q&A document that proves, from **live runtime observation** of a locally-running SimpleLogin instance, that the three named components — the **web server** (`:7777`), the **email handler** (`:20381`), and the **job runner** (10-second poll) — start, bind, log, and process work correctly, and that the core end-user flows (account creation, sign-in, alias creation, inbound-mail reception, dashboard management) behave as designed. SimpleLogin is an open-source email-aliasing service (Python/Flask monolith). The audience is operators and engineers verifying component health. The technical scope is a disciplined **run-observe-document** procedure; per the read-only constraint, **no source code is created or modified** — the sole artifact is the Markdown report.

### 1.2 Completion Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#B23AF2'}}}%%
pie showData title Project Completion — 90.9% Complete
    "Completed Work (AI)" : 40
    "Remaining Work" : 4
```

| Metric | Hours |
|--------|------:|
| **Total Hours** | **44** |
| **Completed Hours (AI + Manual)** | **40** (AI: 40, Manual: 0) |
| **Remaining Hours** | **4** |
| **Percent Complete** | **90.9%** |

> Completion is computed with the AAP-scoped, hours-based methodology: `Completed ÷ (Completed + Remaining) = 40 ÷ 44 = 90.9%`. Every AAP deliverable is delivered and validated; the remaining 4h is the mandatory human review + merge gate (a documentation deliverable has no build/deploy/CI/code-integration work because zero code changed).

### 1.3 Key Accomplishments

- ✅ Authored `blitzy/documentation/app_2cd6ee777f8c.md` (836 lines) — a complete Q&A answer to all three question clusters.
- ✅ Brought up the full SimpleLogin runtime (PostgreSQL 15.13, Redis 7.0.15, MailHog 1.0.1) and started all three components.
- ✅ **Requirement 1 (liveness/health)** proven with verbatim evidence: gunicorn startup markers + `/health`→`success`/200 + `/live`→`live` + `/git`→`dev` + bound port `7777`; email handler startup markers + SMTP `220` banner + bound port `20381`; job runner 10-second cadence (measured deltas 10.018s / 10.019s).
- ✅ **Requirement 2 (user actions)** proven: account creation (verified mailbox), alias creation (flash + count 10→11), inbound email forwarded to MailHog (`X-SimpleLogin-Type: Forward`), plus the bad-credentials negative path.
- ✅ **Requirement 3 (background)** proven: email handler auto-accepts every message; job runner auto-polls and dispatches across idle / burst / natural-consequence situations (partner `proton-welcome-1` job driven end-to-end to `done`).
- ✅ 108 distinct `file:line` citations across 15 files — independently re-verified **100% accurate** (zero line-number drift), including the real source typo `rctp tos`.
- ✅ Coverage pass over every named item and a transparent "explicitly flagged — not verified" section (5 honest caveats).
- ✅ Read-only + cleanup obligations met: all temporary test data/scripts removed; repository restored to seeded baseline; `git` diff shows exactly one added file.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| _None — no unresolved issues block release or validation._ | — | — | — |

> The deliverable is complete and validated. The remaining work is the standard human review/merge gate (see §1.6 and §2.2), not an unresolved defect.

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-----------------|----------------|-------------------|-------------------|-------|
| _n/a_ | _n/a_ | **No access issues identified.** The repository is committed with a clean working tree; all supporting services (PostgreSQL, Redis, MailHog) ran locally in the provided Docker runtime; no external credentials or third-party API access were required for the deliverable. | N/A | — |

### 1.6 Recommended Next Steps

1. **[High]** Technical SME review & acceptance of `blitzy/documentation/app_2cd6ee777f8c.md` — confirm all three question clusters are answered to satisfaction and the flagged caveats are appropriate (~2h).
2. **[Medium]** Merge the single-file documentation PR to the target branch — no code integration, migration, or deploy required (~0.5h).
3. **[Low]** Optional independent QA — spot-verify a sample of citations, re-run 2–3 documented commands, and optionally exercise the honestly-flagged negative paths (~1.5h).

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|------:|-------------|
| Runtime foundation & environment bring-up | 4 | Provision PostgreSQL 15.13 + Redis 7.0.15, apply Alembic migrations, `flask dummy-data` seed (`john@wick.com`/`password`), configure local mail path (MailHog), capture the clean baseline (2 users / 0 jobs / 10 aliases). |
| Component liveness & health evidence — Requirement 1 | 8 | Web server startup markers + `/health`→`success`/200 + `/live`→`live` + `/git`→`dev` + bound-port proof (`/proc/net/tcp` `1E61`=`0A`); email handler startup markers + SMTP `220` banner + bound port `20381` (`4F9D`); job runner 10-second cadence (measured deltas); dashboard sign-in + alias-list signals. |
| End-to-end user-action investigation — Requirement 2 | 6 | Account creation (`User.create`, verified default mailbox); alias creation (dashboard POST + flash `Alias …@sl.local has been created` + count 10→11); inbound email (`smtplib` → handler `New message`/`Finish … 250 Message accepted` → MailHog forward); bad-credentials edge. |
| Background auto-processing investigation — Requirement 3 | 6 | Email handler auto-accept; job runner idle / burst / natural-consequence situations; partner `proton-welcome-1` end-to-end to `done` (DB state 0→1→2); onboarding enqueue analysis; `process_job` dispatch table + `Unknown job name` fallback. |
| Deliverable authoring | 8 | 836-line evidence-grounded Q&A document; one-claim-one-evidence pairing; 108 exact `file:line` citations; coverage pass over every named item; 5 honest "not-verified" caveats. |
| Review-finding resolution | 4 | Resolve 5 code-review findings (F1 citation corrections `.env`→`example.env:L150` + `app/config.py:L401`; F2 verbatim-evidence rework replacing truncated/parsed/placeholder blocks); 500 insertions / 122 deletions. |
| Independent validation & cleanup | 4 | Re-verify ~120 citations against live source; reproduce all Q1/Q2/Q3 runtime claims; confirm 5 caveats honest; remove all test data + temp scripts; restore seeded baseline; verify read-only repo state. |
| **Total** | **40** | **Sum of completed hours (matches §1.2 Completed Hours).** |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|------:|----------|
| Technical SME review & acceptance of the Q&A document | 2.0 | High |
| Independent citation/command spot-verification + optional flagged-path checks (optional QA) | 1.5 | Low |
| PR merge & branch close-out | 0.5 | Medium |
| **Total** | **4.0** | — |

> **Integrity:** §2.1 total (40h) + §2.2 total (4h) = **44h** (§1.2 Total). §2.2 total (4h) = §1.2 Remaining Hours = §7 pie "Remaining Work".

### 2.3 Basis of Estimate

Hours reflect a disciplined run-observe-document investigation of a five-entry-point Flask monolith: environment bring-up, forensic evidence capture across three components (verbatim logs, HTTP/SMTP responses, `/proc/net/tcp` port proofs, measured job cadence, MailHog delivery), an 836-line rigorously-cited report, a full review-and-fix cycle (500/122 line churn), and independent validation. Confidence is **High** — the AAP is well-defined (three explicit question clusters), the deliverable is complete and committed, and every runtime claim was reproduced against the live stack.

---

## 3. Test Results

> **Nature of "tests" for a documentation deliverable:** the report is a Markdown file and has no unit tests of its own. Its "tests" are the **autonomous validation checks** executed by Blitzy's validation systems — document well-formedness, `file:line` citation accuracy, and reproduction of every runtime claim against the live stack. **All rows below originate from Blitzy's autonomous validation logs** (the "Compilation" and "Test" gates) and were corroborated by independent spot-checks during this assessment.

| Test Category | Framework / Method | Total | Passed | Failed | Coverage % | Notes |
|---------------|--------------------|------:|-------:|-------:|-----------:|-------|
| Document well-formedness | `grep`/`sed` structural checks | 4 | 4 | 0 | 100% | 836 lines; 90 balanced code fences; 12 H1 / 6 H2 / 15 H3; UTF-8 intact. |
| Citation accuracy (`file:line`) | Source cross-reference vs live `/app` | 108 | 108 | 0 | 100% | 108 distinct citations across 15 files; **zero** line-number drift; verbatim literals incl. `rctp tos`, log format, `DISABLE_ONBOARDING` guard. |
| Runtime claim — Q1 (liveness/health) | Live observation (`curl`, raw-socket SMTP, `/proc/net/tcp`) | 11 | 11 | 0 | 100% | gunicorn markers; `/health`→`success`/200; `/live`→`live`; `/git`→`dev`; port 7777 bound; email startup; `220` banner; port 20381 bound; 10s cadence. |
| Runtime claim — Q2 (user actions) | Live observation (`User.create`, dashboard POST, `smtplib`, MailHog API, `psql`) | 9 | 9 | 0 | 100% | Account+verified mailbox; alias flash; count 10→11; `sendmail`→`{}`; `New message`; `Finish … 250`; MailHog `Forward`; bad-creds flash. |
| Runtime claim — Q3 (background) | Live observation (`ps`, `psql`, log deltas) | 8 | 8 | 0 | 100% | 3 procs alive ~10min; idle error-free; burst 10.018/10.019s; 3 jobs→`done`; partner `proton-welcome-1`→`done` (state 0→1→2). |
| Caveat honesty verification | Source + runtime confirmation | 5 | 5 | 0 | 100% | All 5 "not-verified-by-running" caveats confirmed accurate and by-design. |
| **Total (deliverable validation)** | — | **145** | **145** | **0** | **100%** | All checks green. |

> **Out-of-scope note (transparency):** the SimpleLogin repository's own `pytest` suite has **83 pre-existing failures** (external SaaS dependencies, HTTP 500s on email/mailbox endpoints, `re2` lookahead, `pgpy`). These are **unrelated to this documentation deliverable** and are **not counted** above. The AAP read-only constraint forbids modifying source/tests, and this doc-only change touches no code.

---

## 4. Runtime Validation & UI Verification

**Component liveness (all observed live during validation):**
- ✅ **Web server** (`gunicorn wsgi:app :7777`) — `Starting gunicorn 20.0.4`, `Listening at: http://0.0.0.0:7777`, 2 workers booted; `/health`→`success`/HTTP 200 (Content-Length 7); `/live`→`live`; `/git`→`dev`; port 7777 in `LISTEN` state.
- ✅ **Email handler** (`email_handler.py` aiosmtpd `:20381`) — `Listen for port 20381`, `Start mail controller 0.0.0.0 20381`; SMTP banner `220 … Python SMTP 1.4.2`; port 20381 in `LISTEN` state; processes inbound messages.
- ✅ **Job runner** (`job_runner.py`) — 10-second poll cadence (measured deltas 10.018s / 10.019s); jobs picked one-per-poll and driven to `done`.

**User-facing / UI verification:**
- ✅ **Sign-in** — `john@wick.com`/`password` → HTTP 302 → `/dashboard/`.
- ✅ **Alias management (dashboard)** — seeded alias list `e0/e1/e2@sl.local`; `nb_alias` count; random-alias creation flash `Alias …@sl.local has been created`; count incremented 10→11.
- ✅ **Inbound forward → MailHog** — message to alias forwarded to `john@wick.com` with `X-SimpleLogin-Type: Forward`, VERP return-path, and reverse-alias `From`.

**Supporting services:**
- ✅ **PostgreSQL 15.13** — reachable; user/alias/job state queried directly.
- ✅ **Redis 7.0.15** — answered `PONG`.
- ✅ **MailHog 1.0.1** — captured the forwarded message (API `:8025`).

**Honestly-flagged partial observations (by design; code-cited, not runtime-triggered):**
- ⚠ **SMTP negative paths** (cannot-receive `E504`/`E207`, reverse-alias `E524`, VERP `E213`) — not triggered (the successful test returned `250 Message accepted for delivery`); cited from source.
- ⚠ **`proton-welcome-1` outbound email** — the branch executed and the job reached `done`, but `welcome_proton()` early-returns for a user with no Proton communication address, so no separate MailHog delivery was claimed for it.

---

## 5. Compliance & Quality Review

| AAP / Rule Benchmark | Requirement | Status | Progress |
|----------------------|-------------|:------:|:--------:|
| Read-only source | No existing source/config/dependency file modified | ✅ Pass | 100% |
| Deliverable location & name | `blitzy/documentation/app_2cd6ee777f8c.md` (branch-named) | ✅ Pass | 100% |
| Run-first methodology | Built & ran the stack; wrote from observation | ✅ Pass | 100% |
| One-claim-one-evidence | Each claim paired with its producing command + verbatim output | ✅ Pass | 100% |
| Exact `file:line` grounding | 108 distinct citations, 100% accurate | ✅ Pass | 100% |
| Observe at representative scale | >1 poll cycle observed (measured 10s cadence) | ✅ Pass | 100% |
| Exhaustive coverage pass | Every named item + every "e.g./such as" example addressed | ✅ Pass | 100% |
| Honest flagging of unverifiable | 5 caveats explicitly listed and justified | ✅ Pass | 100% |
| Cleanup / temp-data removal | Test data, scripts, and env overrides removed; baseline restored | ✅ Pass | 100% |
| No dependency changes | Zero manifest/lockfile/import edits | ✅ Pass | 100% |
| Verbatim evidence (no paraphrase) | Truncated/parsed/placeholder blocks replaced with exact output | ✅ Pass | 100% |
| Secret-free evidence | Set-Cookie values withheld; only public seed creds shown | ✅ Pass | 100% |

**Fixes applied during autonomous validation:** 5 review findings resolved in commit `e9040a8d` (F1 — corrected an untracked-file citation to the tracked template `example.env:L150` plus consumer `app/config.py:L401`; F2 — replaced all truncated/parsed/placeholder evidence blocks with exact producing commands and verbatim output; plus three further findings), 500 insertions / 122 deletions. During final validation, 43 stray untracked prior-checkpoint screenshots were removed (outside the single-file deliverable scope) and all runtime test data/scripts were cleaned.

**Outstanding in-scope items:** none.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|:--------:|:-----------:|------------|:------:|
| Citation line-number drift if source evolves after this branch | Technical | Low | Medium | 108 citations pinned to branch `app_2cd6ee777f8c` (base `2cd6ee77`); doc states it reflects that snapshot; `SL` log lines self-embed `pathname:lineno`. | Mitigated |
| Point-in-time runtime values (PIDs, timestamps, VERP tokens, elapsed seconds) not byte-identical on re-run | Technical | Low | Low | Each claim shows its producing command; the behavioral **pattern** is the claim, not the exact value. | Accepted |
| Accidental secret leakage in captured HTTP/DB output | Security | Medium | Low | `Set-Cookie` values withheld (header names only); cookie-jar files used; only public seed credentials shown; F2 finding enforced secret-free extraction. | Mitigated |
| Runtime service versions differ from canonical docs (PG 15.13 / Redis 7.0.15 vs PG 13 / Redis v6) | Operational | Low | Low | Explicitly flagged as caveat #5; noted not to affect any cited behavior. | Mitigated |
| Standalone Markdown not auto-published to a docs portal | Operational | Low | Low | Placed at the AAP-mandated path `blitzy/documentation/`; manual publish if the org requires a portal. | Open |
| Pre-existing 83 `pytest` failures visible in CI (unrelated to deliverable) | Integration | Low | High | Documented as pre-existing & out-of-scope; read-only constraint forbids fixing; the doc-only change touches no code and is CI-neutral. | Open (out of scope) |
| PR merge conflict | Integration | Low | Low | Only adds a new `blitzy/documentation/` directory; no source file touched. | Mitigated |

> **Overall risk posture: Low.** The read-only, zero-code-change nature of the deliverable eliminates build/compile/integration risk. No unresolved technical or security blockers.

---

## 7. Visual Project Status

**Project hours breakdown** (Completed = Dark Blue `#5B39F3`; Remaining = White `#FFFFFF`):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#B23AF2'}}}%%
pie showData title Project Hours — Completed vs Remaining
    "Completed Work" : 40
    "Remaining Work" : 4
```

**Remaining work by category** (hours from §2.2; sums to 4h):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#B23AF2','pie2':'#5B39F3','pie3':'#A8FDD9','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#B23AF2'}}}%%
pie showData title Remaining 4h by Category
    "SME Review & Acceptance [High]" : 2.0
    "Spot-verification / optional QA [Low]" : 1.5
    "PR Merge & Close-out [Medium]" : 0.5
```

> **Integrity:** the "Remaining Work" pie value (**4**) equals §1.2 Remaining Hours and the §2.2 "Hours" column sum. The "Completed Work" value (**40**) equals §1.2 Completed Hours and the §2.1 total.

---

## 8. Summary & Recommendations

**Achievements.** The project delivers a complete, evidence-grounded runtime investigation report that answers all three question clusters with verbatim runtime evidence and exact `file:line` grounding. All three named components (web server, email handler, job runner) were shown live and responding; all core user flows (account creation, alias creation, inbound-mail reception, dashboard management) were exercised end-to-end; and background auto-processing was demonstrated across multiple situations. The 108 citations were independently verified 100% accurate, and the read-only + cleanup obligations were fully met.

**Remaining gaps & critical path to production.** The project is **90.9% complete** (40h of 44h). Because this is a documentation deliverable with **zero code changes**, there is no build, deployment, CI, or integration work — the critical path is simply: (1) SME acceptance review → (2) PR merge, with optional independent QA in between. Total remaining effort is **4h**.

**Success metrics.** All 145 autonomous validation checks passed (100%); zero citation drift; every runtime claim reproduced; five caveats honestly flagged and confirmed; repository read-only-clean with exactly one added file.

**Production readiness.** The single in-scope deliverable is **production-ready pending human sign-off**. No unresolved issues, no access issues, and a uniformly Low risk posture. The only gate is the standard documentation review/merge cycle.

| Metric | Value |
|--------|-------|
| Completion | 90.9% (40h / 44h) |
| Autonomous validation checks | 145 passed / 0 failed |
| `file:line` citation accuracy | 100% (108/108) |
| Source files modified | 0 (read-only) |
| Files added | 1 (`blitzy/documentation/app_2cd6ee777f8c.md`, 836 lines) |
| Unresolved blockers | None |

---

## 9. Development Guide

This deliverable is a Markdown document. This guide covers **(A)** how to review/verify the deliverable itself and **(B)** how to reproduce the runtime the document describes so its commands can be re-run.

### 9.1 System Prerequisites

- **Track A (review the doc):** `git` and any Markdown viewer (VS Code preview, `grip`, or GitHub). _Verified here with git 2.51.0._
- **Track B (reproduce the runtime):** Docker; Python `^3.10` (report observed 3.10.18); PostgreSQL 13 (report observed 15.13); Redis v6 (report observed 7.0.15); MailHog 1.0.1; Poetry for dependency management.

### 9.2 Track A — Review & Verify the Deliverable _(all commands tested in this environment)_

```bash
# View the report and confirm size
wc -l blitzy/documentation/app_2cd6ee777f8c.md          # -> 836

# Confirm the change set is EXACTLY one added file (read-only compliance)
git diff --name-status 2cd6ee77..HEAD                    # -> A blitzy/documentation/app_2cd6ee777f8c.md

# Markdown well-formedness: code fences must be an even count (balanced)
grep -cE '^```' blitzy/documentation/app_2cd6ee777f8c.md # -> 90 (even => balanced)

# Header inventory
grep -cE '^# '  blitzy/documentation/app_2cd6ee777f8c.md # H1 -> 12
grep -cE '^## ' blitzy/documentation/app_2cd6ee777f8c.md # H2 -> 6
grep -cE '^### ' blitzy/documentation/app_2cd6ee777f8c.md # H3 -> 15

# Spot-verify a citation against source (expect /health -> success,200)
sed -n '213,215p' server.py
```

### 9.3 Track B — Reproduce the Runtime the Report Describes _(canonical commands from `CONTRIBUTING.md` / `Dockerfile` / the report)_

```bash
# 1) Infrastructure
docker run -e POSTGRES_PASSWORD=mypassword -e POSTGRES_USER=myuser \
  -e POSTGRES_DB=simplelogin -p 15432:5432 postgres:13        # CONTRIBUTING.md:L100
# start Redis; start MailHog (SMTP :1025, HTTP API :8025, UI :1080)

# 2) Environment: copy example.env -> .env; ensure local mail path:
#    comment NOT_SEND_EMAIL, set POSTFIX_SERVER=localhost, POSTFIX_PORT=1025

# 3) Migrations + seed + web server (canonical bring-up)
alembic upgrade head && flask dummy-data && python3 server.py  # CONTRIBUTING.md:L106
# seeded login: john@wick.com / password                       # CONTRIBUTING.md:L109

# 4) Start the three components (each in its own process)
gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15            # Dockerfile:L47 (prod)
python3 email_handler.py                                       # default --port 20381
python3 job_runner.py                                          # 10-second poll loop
```

### 9.4 Verification Steps

```bash
# Web server liveness
curl -i http://localhost:7777/health     # -> HTTP 200, body: success   (server.py:L213-215)
curl    http://localhost:7777/live       # -> live                       (app/monitor/views.py:L10-12)
curl    http://localhost:7777/git        # -> dev

# Email handler: read startup markers in its stdout, then probe the SMTP banner
#   expect: "Listen for port 20381" and "Start mail controller 0.0.0.0 20381"
python3 - <<'PY'
import socket
s=socket.create_connection(("localhost",20381),timeout=5)
print(s.recv(1024).decode().strip())     # -> 220 <host> Python SMTP <ver>
s.sendall(b"QUIT\r\n"); s.close()
PY

# Job runner: watch its stdout for the 10-second cadence -> "Take job ..." lines

# Port-binding when lsof/ss are unavailable (read /proc/net/tcp; 0A = LISTEN)
#   7777 = 0x1E61, 20381 = 0x4F9D
grep -iE '1E61|4F9D' /proc/net/tcp
```

### 9.5 Example Usage

```bash
# Create a random alias from the dashboard (authenticated) -> flash:
#   "Alias <name>@sl.local has been created"                 (index.py:L111)

# Send an email to the alias and observe the forward
python3 - <<'PY'
import smtplib
from email.mime.text import MIMEText
m=MIMEText("inbound test"); m["Subject"]="test"
m["From"]="tester@example.com"; m["To"]="<alias>@sl.local"
with smtplib.SMTP("localhost",20381,timeout=15) as s:
    print(s.sendmail("tester@example.com",["<alias>@sl.local"],m.as_string()))  # -> {}
PY
# Handler logs: "New message, mail from ... rctp tos [...]" then
#               "Finish ... 250 Message accepted for delivery"  (email_handler.py:L2343-2368)

# Confirm the forward landed in MailHog (X-SimpleLogin-Type: Forward)
curl -s http://localhost:8025/api/v2/messages
```

### 9.6 Troubleshooting

- **Port not bound?** With `lsof`/`ss`/`netstat` absent, read `/proc/net/tcp` — `1E61`=7777, `4F9D`=20381, state `0A`=`LISTEN`.
- **No forward in the sink?** Ensure `NOT_SEND_EMAIL` is commented out and `POSTFIX_PORT=1025` points at MailHog.
- **Account creation enqueues no onboarding jobs?** `DISABLE_ONBOARDING` present in the environment triggers an early return (`app/models.py:L646-648`); onboarding jobs are also scheduled 1–3 days out (outside the runner's `now + 10 min` window).
- **`pytest` shows failures?** The repository has **83 pre-existing failures** unrelated to this deliverable (external SaaS deps, `re2`, `pgpy`); the read-only constraint forbids modifying source/tests.
- **Cleanup after reproduction:** delete temporary users/aliases/jobs (cascade), clear MailHog (`DELETE /api/v1/messages`), remove temp scripts, and revert any `.env` overrides so the repository stays read-only-clean.

---

## 10. Appendices

### A. Command Reference

| Command | Purpose |
|---------|---------|
| `git diff --name-status 2cd6ee77..HEAD` | Confirm the change set is exactly one added file |
| `wc -l blitzy/documentation/app_2cd6ee777f8c.md` | Verify report length (836) |
| `grep -cE '^\x60\x60\x60' <doc>` | Check code-fence balance (90, even) |
| `alembic upgrade head` | Apply database migrations |
| `flask dummy-data` | Seed `john@wick.com`/`password` + sample aliases |
| `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` | Start the web server (production entry) |
| `python3 email_handler.py` | Start the email handler (aiosmtpd, port 20381) |
| `python3 job_runner.py` | Start the job runner (10-second poll) |
| `curl -i http://localhost:7777/health` | Web liveness → `success`/200 |
| `grep -iE '1E61|4F9D' /proc/net/tcp` | Prove ports 7777/20381 are bound |

### B. Port Reference

| Port | Service | Notes |
|-----:|---------|-------|
| 7777 | Web server (gunicorn/Flask) | `/health`, `/live`, `/git`, dashboard |
| 20381 | Email handler (aiosmtpd) | Local SMTP; `--port` default |
| 5432 | PostgreSQL | In-container (`DB_URI`); host-mapped 15432 per `CONTRIBUTING.md` |
| 6379 | Redis | Sessions, rate limiting, locks |
| 1025 | MailHog SMTP | `POSTFIX_PORT` forward target |
| 8025 | MailHog HTTP API | Message inspection (UI mapped to host `:1080`) |

### C. Key File Locations

| Path | Role |
|------|------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The deliverable** (836-line Q&A report) |
| `server.py` | Web app factory; `/health` (`success`/200) at L213–215; dev launcher L588 |
| `email_handler.py` | aiosmtpd controller (L2381–2386); `--port` default 20381; per-message logs L2343–2368 |
| `job_runner.py` | 10-second poll loop (L329–347); `Take job` L334; `process_job` L188–304 |
| `app/log.py` | Shared `SL` stdout logger; `>>> init logging <<<` marker L67 |
| `app/monitor/views.py` | `/git`, `/live`, `/exception` routes |
| `app/dashboard/views/index.py` | Alias list + random-alias flash (L104–111) |
| `app/fake_data.py` | Seed `john@wick.com` / `password` (L45, L47) |
| `example.env` | Runtime config template (`URL`, `NOT_SEND_EMAIL`, `EMAIL_DOMAIN`, `DB_URI`, `POSTFIX_PORT`, `DISABLE_ONBOARDING`) |
| `Dockerfile` | Production web command + `EXPOSE 7777` (L44–47) |
| `CONTRIBUTING.md` | Canonical local-run sequence (L100–112) |

### D. Technology Versions

| Component | Version | Source |
|-----------|---------|--------|
| Python | ^3.10 (observed 3.10.18) | `pyproject.toml` |
| Flask | ^1.1.2 | `pyproject.toml` |
| gunicorn | ^20.0.4 (observed 20.0.4) | `pyproject.toml` / `Dockerfile:L47` |
| aiosmtpd | ^1.2 (runtime SMTP 1.4.2) | `pyproject.toml` |
| SQLAlchemy | 1.3.24 | `pyproject.toml` |
| redis (client) | ^4.5.3 | `pyproject.toml` |
| PostgreSQL | 13 canonical / 15.13 observed | `CONTRIBUTING.md` / runtime |
| Redis (server) | v6 canonical / 7.0.15 observed | runtime |
| MailHog | 1.0.1 | runtime |

### E. Environment Variable Reference

| Variable | Example / Value | Purpose |
|----------|-----------------|---------|
| `URL` | `http://localhost:7777` | Base app URL (`example.env:L6`) |
| `NOT_SEND_EMAIL` | commented for local sink | When set, suppresses outbound mail (`example.env:L19`) |
| `EMAIL_DOMAIN` | `sl.local` | Alias domain (`example.env:L22`) |
| `DB_URI` | `postgresql://myuser:mypassword@localhost:5432/simplelogin` | Database DSN (`example.env:L75`) |
| `POSTFIX_SERVER` | `localhost` | Outbound MTA target |
| `POSTFIX_PORT` | `1025` | MailHog SMTP port (`example.env:L154`) |
| `DISABLE_ONBOARDING` | present (`true`) | Truthy by presence; early-returns before onboarding jobs (`example.env:L150` → `app/config.py:L401`) |

### F. Developer Tools Guide

- **Markdown review:** VS Code preview, `grip` (GitHub-flavored render), or view directly on GitHub. The document contains no Mermaid diagrams to render.
- **Citation verification:** use `sed -n 'START,ENDp' <file>` to open any cited line range and compare against the quoted literal in the report.
- **Runtime inspection (Track B):** `curl` (HTTP), Python `smtplib`/`socket` (SMTP), `psql` (DB state), MailHog API (`/api/v2/messages`), and `/proc/net/tcp` for port state when `lsof`/`ss` are unavailable.

### G. Glossary

| Term | Meaning |
|------|---------|
| **Alias** | A generated email address (e.g., `…@sl.local`) that forwards to a user's real mailbox. |
| **Email handler** | The aiosmtpd-based SMTP process (`email_handler.py`, port 20381) that receives and forwards inbound mail. |
| **Job runner** | The background worker (`job_runner.py`) that polls a `Job` queue every 10 seconds and dispatches work via `process_job`. |
| **VERP** | Variable Envelope Return Path — the encoded return-path SimpleLogin uses on forwarded mail. |
| **Reverse-alias** | A rewritten `From` address that lets replies route back through SimpleLogin. |
| **MailHog** | A local SMTP sink that captures forwarded mail for inspection (API `:8025`). |
| **`create_light_app()`** | The shared lightweight Flask app-context factory used by all three components (`server.py:L127`). |
| **`/proc/net/tcp`** | Kernel table of TCP sockets; state `0A` = `LISTEN`, used here to prove port binding. |
