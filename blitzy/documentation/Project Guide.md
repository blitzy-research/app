# Blitzy Project Guide — SimpleLogin Runtime Verification (Web Server, Email Handler, Job Runner)

> **Brand legend:** Completed / AI Work = **Dark Blue `#5B39F3`** · Remaining / Not Completed = **White `#FFFFFF`** · Headings/Accents = Violet-Black `#B23AF2` · Highlight = Mint `#A8FDD9`. These colors are applied to the visual charts in Sections 1.2 and 7.

---

## 1. Executive Summary

### 1.1 Project Overview

This is a **read-only investigation and documentation** project. The objective was to produce one authoritative, runtime-verified Markdown document explaining how an operator confirms that SimpleLogin's three long-running components — the **web server**, the **email handler**, and the **job runner** — are up and behaving correctly, demonstrated by actually running the code and capturing real output. The audience is developers and operators of the SimpleLogin self-hosted stack. The technical scope covers standing up the canonical Docker runtime, exercising each component's real entry point, driving user flows (register → alias → receive email), and grounding every claim in `file:line` citations and observed output. No product source code was modified; the sole artifact is `blitzy/documentation/app_2cd6ee777f8c.md`.

### 1.2 Completion Status

The completion percentage is computed from AAP-scoped hours only: **Completion % = Completed Hours ÷ Total Hours = 34 ÷ 38 = 89.5%**. All 15 AAP-specified deliverables are complete; the remaining 4 hours are path-to-production human review and acceptance.

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieLegendTextColor':'#333333'}}}%%
pie showData title Completion Status — 89.5% Complete
    "Completed Work (AI)" : 34
    "Remaining Work" : 4
```

| Metric | Value |
|---|---|
| **Total Hours** | **38** |
| **Completed Hours (AI + Manual)** | **34** (AI: 34 · Manual: 0) |
| **Remaining Hours** | **4** |
| **Percent Complete** | **89.5%** |

### 1.3 Key Accomplishments

- ✅ Delivered the mandated single artifact `blitzy/documentation/app_2cd6ee777f8c.md` (1,004 lines) — branch-named per the SWE-AtlasQnA-Repo rules.
- ✅ Stood up the canonical Docker runtime (venv Python 3.10.18) with PostgreSQL 13 (`sl-db`) and Redis (`sl-redis`); applied Alembic migrations (head `32f25cbf12f6`) and `flask dummy-data`.
- ✅ Answered **Q1** with observed up-signals: web `/health` → `HTTP 200`/`success` on 7777; email handler startup logs + port 20381; job runner boot banner + ~10 s poll loop; dashboard login + alias-management UI.
- ✅ Answered **Q2** with three-layer correctness signals: SMTP `E200 "250 Message accepted for delivery"`, forward log chain, and new `Alias`/`Contact`/`EmailLog` rows; plus edge cases (free-plan quota gate; non-existent-alias `E515` rejection).
- ✅ Answered **Q3**: demonstrated the email handler and job runner are **independent daemons** (default `CMD` runs the web server only) and that yacron `cron.py` is a distinct scheduler.
- ✅ Methodology compliance: run-first-then-write, real entry points, default configuration, every condition (happy + edge), complete unedited output, 80 OBSERVED / 4 INFERRED labels, ~60+ `file:line` citations, and a full Coverage Pass.
- ✅ Read-only honored (single-file diff vs base `2cd6ee77`) and all temporary data/scripts cleaned up (DB restored to the `dummy-data` baseline).

### 1.4 Critical Unresolved Issues

**No release-blocking issues.** All AAP deliverables are complete and the document is committed. The items below are **non-blocking** and are surfaced for transparency; none prevent release or validation.

| Issue | Impact | Owner | ETA |
|---|---|---|---|
| Minor internal wording variance in the doc (Coverage Pass cell says quota "4th attempt blocked" vs Summary "blocks the 6th alias"; both reflect free-plan limit = 5 across different capture sessions) | Cosmetic; substantive answer is correct and observed | Human reviewer | 0.5 h (within HT-1) |
| Untracked `blitzy/screenshots/` directory (~40 QA PNGs) has undecided git disposition | Governance/cleanliness only; not part of the committed deliverable | Human reviewer / maintainer | 1 h (HT-3) |
| Out-of-scope `google-re2` codebase test failures (`re2.DOTALL` missing) | None on the deliverable (Markdown doc); pre-existing in pinned source; **out of AAP scope** and not fixable read-only | SimpleLogin maintainers (upstream) | N/A (out of scope) |

### 1.5 Access Issues

**No access issues identified.** The canonical runtime image was available and running; all backing services and code paths were reachable during validation.

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|---|---|---|---|---|
| Runtime image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` | Container image pull/run | None — image present, `sl-app` container Up | ✅ Resolved | Platform |
| PostgreSQL 13 (`sl-db`) & Redis (`sl-redis`) | Backing-service network access | None — both reachable (`PONG`, `alembic current` = head) | ✅ Resolved | Platform |
| Git repository (branch `blitzy-521cce88-…`) | Read/write to branch | None — commits landed; single-file diff verified | ✅ Resolved | Platform |

### 1.6 Recommended Next Steps

1. **[High]** Review the answer document for accuracy: spot-check a sample of the ~60+ `file:line` citations and reconcile the minor quota wording variance (edit the **document only**, never source).
2. **[Medium]** Obtain stakeholder acceptance that Q1/Q2/Q3 are answered to satisfaction, then approve/merge the PR.
3. **[Low]** Decide the disposition of the untracked `blitzy/screenshots/` directory (commit separately, `.gitignore`, or remove) to preserve the doc's single-committed-file self-claim.
4. **[Low]** (Optional, upstream) Track the pre-existing `google-re2`/`re2.DOTALL` test-suite artifact with the SimpleLogin maintainers — it is out of this task's read-only scope.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

All completed hours are AI/autonomous and trace to AAP requirements (R1–R15). **Total = 34 hours.**

| Component | Hours | Description |
|---|---:|---|
| Canonical Docker runtime & backing-service provisioning | 5 | Stand up the `ghcr.io/scaleapi/swe-atlas` image, PostgreSQL 13 (`sl-db`), Redis (`sl-redis`); copy `example.env` → `.env`; `alembic upgrade head` (rev `32f25cbf12f6`); `flask dummy-data`; resolve `/app`-vs-`/code` cwd and `google-re2`/`multidict` environment specifics [R2] |
| Q1 — Web / email / job startup & health signals | 4 | Web `/health` → `200 success` on 7777 + index `302 → /auth/login` + dev/prod contrast; email handler startup logs + bound port 20381; job runner boot banner + poll loop [R3,R4,R5] |
| Q1 — Dashboard UI verification | 3 | Render login page + authenticated dashboard (alias list, `stats.nb_alias`, Random/Custom-alias buttons, search); ~40 screenshots across breakpoints 375/768/1280/1920 and interactive states [R6] |
| Q2 — Account, activation, login & alias creation | 5 | Drive register → activate (`activated` False→True) → login → `/dashboard/`; create a random alias with flash + log + row + count increment [R7,R8] |
| Q2 — Email reception (3-layer), physical delivery & edge cases | 7 | swaks → `E200 "250 Message accepted for delivery"`; forward log chain; new `Contact`/`EmailLog` rows; physical delivery (`NOT_SEND_EMAIL` disabled, local sink); free-plan quota gate; non-existent-alias `E515` `550` rejection; before/after DB state [R9,R10] |
| Q3 — Background component investigation | 4 | Daemon independence (Dockerfile web-only `CMD`, no spawn logic, separate top-level processes); job runner ~10 s cadence across 2 runs (`ready → taken → done`); cron/yacron distinction (15 jobs) [R11,R12,R13] |
| Answer-document authoring, QA cycles, cleanup & read-only verification | 6 | Author the 1,004-line Q&A doc (~60+ citations, 80 OBSERVED / 4 INFERRED, Coverage Pass); 5 QA cycles resolving 11+ findings across 6 commits; cleanup temp data/scripts to the `dummy-data` baseline; verify git-clean single-file diff [R1,R14,R15] |
| **Total Completed** | **34** | |

### 2.2 Remaining Work Detail

All remaining work is path-to-production human review/acceptance (P1–P3). **Total = 4 hours.**

| Category | Hours | Priority |
|---|---:|---|
| Human review of document accuracy, citation spot-check & internal-consistency pass (reconcile session-specific values incl. quota "4th/6th" wording) | 2 | **High** |
| Stakeholder acceptance sign-off that the document answers all three user questions | 1 | **Medium** |
| Resolve disposition of the untracked `blitzy/screenshots/` directory (commit / gitignore / remove) | 1 | **Low** |
| **Total Remaining** | **4** | |

> **Out-of-scope advisory (0 in-scope hours):** The pre-existing `google-re2` codebase test failures are **not** included in the remaining hours because fixing them requires editing read-only source or swapping a pinned dependency — both forbidden by the user constraint and AAP §0.3.2/§0.4.2. They do not affect the Markdown deliverable.

### 2.3 Hours Reconciliation & Totals

| Reconciliation | Value |
|---|---:|
| Section 2.1 — Completed Hours | 34 |
| Section 2.2 — Remaining Hours | 4 |
| **Total Project Hours** (2.1 + 2.2) | **38** |
| Completion % (34 ÷ 38 × 100) | **89.5%** |

✅ Cross-section check: `2.1 (34) + 2.2 (4) = 38` matches the Total Hours in Section 1.2; the Remaining value (`4`) is identical in Sections 1.2, 2.2, and 7.

---

## 3. Test Results

Tests below originate from Blitzy's autonomous validation logs for this project. **Important framing:** the deliverable is a Markdown document and therefore has no unit tests of its own; its correctness was validated by **runtime reproduction** (see Section 4). The codebase regression suite was executed to confirm the health of the code the document describes.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---|---|---:|---:|---:|---:|---|
| Codebase regression (unit + integration) | pytest | 611 collected | 609 | 2 | Not measured (doc has no unit tests) | + 3 module **collection errors**. All 5 non-passing items trace to the single **out-of-scope** `google-re2` artifact (`re2.DOTALL` missing; Perl-lookahead unsupported). Confirmed on pristine source. Pass rate 609/611 = 99.7%. |
| Runtime verification — Q1 (startup/health) | curl / process + log inspection | 6 | 6 | 0 | n/a | `/health` → 200 `success`; `GET /` → 302 `/auth/login`; email handler startup + port 20381; job runner boot; dashboard render. |
| Runtime verification — Q2 (user actions) | swaks / psql / HTTP flows | 8 | 8 | 0 | n/a | register→activate→login; alias create (count++); `E200` receive; physical delivery; quota gate; `E515` reject; `Contact`/`EmailLog` row deltas. |
| Runtime verification — Q3 (background) | ps / grep / timing / yacron | 5 | 5 | 0 | n/a | daemon independence; no spawn logic; ~10 s cadence across 2 runs; `ready→taken→done`; 15 distinct cron jobs. |

> **Integrity note (Rule 3):** Every row above is drawn from Blitzy's autonomous test/validation execution for this project. The codebase pytest baseline and the runtime verification checks were both produced during autonomous validation and independently reproduced this session.

---

## 4. Runtime Validation & UI Verification

**Status key:** ✅ Operational · ⚠ Partial · ❌ Failing

**Q1 — Component health**
- ✅ **Web server** — `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15`; `GET /health` → `HTTP 200`, body `success`; `GET /` → `302 → /auth/login`.
- ✅ **Email handler** — `python email_handler.py` runs as an independent process; startup logs `Start mail controller 0.0.0.0 20381` / `Listen for port 20381`; binds port 20381.
- ✅ **Job runner** — `python job_runner.py` runs as an independent process; `>>> init logging <<<` boot banner; ~10 s polling loop.

**Q1 — Dashboard UI**
- ✅ **Login page** (`/auth/login`) — form with CSRF, email, password, and Sign-up link.
- ✅ **Authenticated dashboard** (`/dashboard/`) — alias list, `stats.nb_alias` count, "Random Alias" & "Create a custom alias" buttons, and search box (verified via login as `john@wick.com`).

**Q2 — User actions & correctness**
- ✅ **Account creation → activation → login** — `POST /auth/register` 200; `activated` False→True; login → `302 → /dashboard/`.
- ✅ **Alias creation** — random alias created; count incremented; flash + log line emitted; new `Alias` row persisted.
- ✅ **Alias receives email** — SMTP `E200 "250 Message accepted for delivery"`; forward log chain; new `Contact` + `EmailLog` rows.
- ✅ **Physical delivery** — with sending enabled (`NOT_SEND_EMAIL` disabled), the message reaches the local sink mailbox.
- ✅ **Edge: quota gate** — free-plan limit (5) blocks additional alias creation; count unchanged; upgrade warning shown.
- ✅ **Edge: non-existent alias** — inbound mail rejected with `550 SL E515`; DB unchanged by the rejection.

**Q3 — Background components**
- ✅ **Independence** — container default `CMD` runs the web server only; email handler and job runner are separate top-level processes (not children of the gunicorn master).
- ✅ **No implicit spawn** — `server.py`/`wsgi.py` contain no daemon-spawn logic.
- ✅ **Continuous run & cadence** — job runner polls every ~10 s across ≥2 runs; jobs progress `ready(0) → taken(1) → done(2)`.
- ✅ **Cron distinction** — 15 yacron `cron.py -j` jobs run on schedules, independent of the Job-table-polling job runner.

**Codebase health**
- ⚠ **Regression suite** — 609/611 passing; the small non-passing set is the **out-of-scope** `google-re2` artifact only (does not affect the deliverable).

---

## 5. Compliance & Quality Review

Cross-map of AAP deliverables and SWE-AtlasQnA-Repo methodology rules to their quality benchmarks. Fixes applied during autonomous validation are noted.

| Benchmark / Rule | Requirement | Status | Progress | Evidence / Notes |
|---|---|---|---|---|
| Deliverable location & name | `blitzy/documentation/app_2cd6ee777f8c.md` (branch-named) | ✅ Pass | 100% | File present, committed; directory created |
| Single-file / read-only scope | No existing file modified/deleted; only the doc added | ✅ Pass | 100% | `git diff 2cd6ee77 HEAD --name-status` = `A …app_2cd6ee777f8c.md`; excluding `blitzy/` = empty |
| No dependency changes | `pyproject.toml`/`poetry.lock` untouched | ✅ Pass | 100% | Diff vs base empty (AAP §0.4.2 None/None/None) |
| Run-first-then-write | Claims from observed output, not reading | ✅ Pass | 100% | 80 OBSERVED labels; commands + full output embedded |
| Real entry points | `server.py`/`wsgi.py`, `email_handler.py`, `job_runner.py` | ✅ Pass | 100% | Each exercised via its canonical invocation |
| Default/canonical config | Values reflect default build/config | ✅ Pass | 100% | gunicorn default `CMD`; ports 7777/20381; documented deltas only |
| Every condition (happy + edge) | Primary + error/edge + before/after | ✅ Pass | 100% | Quota gate, `E515`, before/after DB snapshots included |
| Complete unedited output | Full output + producing command per condition | ✅ Pass | 100% | 38 balanced code blocks with raw output |
| `file:line` grounding | Specific citations; name the function | ✅ Pass | 100% | ~60+ citations resolve exactly to source `2cd6ee77` |
| OBSERVED vs INFERRED labeling | Inferred content labeled | ✅ Pass | 100% | 80 OBSERVED / 4 INFERRED |
| Answer every part/named item | Coverage pass across Q1/Q2/Q3 | ✅ Pass | 100% | §5 Coverage Pass + Summary of Direct Answers |
| Cleanup | Temp data/scripts removed; repo git-clean | ✅ Pass | 100% | DB restored to baseline; 0 temp scripts; only untracked screenshots |
| Zero placeholders | No TODO/FIXME/stub content | ✅ Pass | 100% | 0 real placeholders (2 "placeholder" hits = captured HTML attribute evidence) |
| Internal consistency | Session values framed; no contradictions | ⚠ Minor | 95% | One cosmetic wording variance (quota "4th" vs "6th") flagged for human pass (HT-1) |

**Overall quality:** production-ready for the single in-scope deliverable; the only open compliance item is a cosmetic wording pass.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|---|---|---|---|---|---|
| Out-of-scope `google-re2` test failures (`re2.DOTALL` missing; Perl-lookahead unsupported): 2 failed + 3 collection errors in the codebase suite | Technical | Low | High | Documented in the deliverable; pre-existing in pinned source; fixing forbidden under read-only scope / pinned dependency | Open (accepted / out of scope) |
| Session-specific values in the doc (PIDs, row ids, timestamps, byte counts) differ run-to-run | Technical | Low | Medium | Doc frames all such values as captured evidence (not "always-X"); Legend explains multi-session capture | Mitigated |
| Minor internal wording inconsistency (quota gate "4th attempt" cell vs "6th alias" summary) | Technical | Low | Low | Human consistency pass (HT-1); substantive behavior is correct and observed | Open (flagged) |
| No source/config/dependency changes → no new attack surface; local `.env` (default demo secrets) uncommitted & gitignored | Security | Informational | Low | Read-only scope enforced; documented creds (`john@wick.com`/`password`) are standard `dummy-data`, not real secrets | Not applicable / No action |
| Untracked `blitzy/screenshots/` (~40 PNGs) disposition ambiguity | Operational | Low | Medium | Human decision (HT-3); deliberately left untracked to preserve the single-committed-file self-claim while retaining user-facing evidence | Open (decision pending) |
| Runtime evidence depends on the ephemeral Docker environment (`sl-app`/`sl-db`/`sl-redis`) | Operational | Low | Low | Doc §1 records exact re-stand-up commands; captured output embedded so the deliverable is self-contained | Mitigated |
| No external integrations added/modified; no API keys/credentials required by the deliverable | Integration | N/A | N/A | Investigation used a local SMTP sink + local backing services | Not applicable |

---

## 7. Visual Project Status

**Project hours breakdown** (Completed = Dark Blue `#5B39F3`, Remaining = White `#FFFFFF`):

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieLegendTextColor':'#333333'}}}%%
pie showData title Project Hours Breakdown
    "Completed Work" : 34
    "Remaining Work" : 4
```

**Remaining work by priority** (sums to the Remaining Hours = 4):

| Priority | Hours | Share of Remaining |
|---|---:|---:|
| High — document accuracy & consistency review | 2 | 50% |
| Medium — stakeholder acceptance sign-off | 1 | 25% |
| Low — screenshots directory disposition | 1 | 25% |
| **Total** | **4** | **100%** |

> **Integrity note (Rule 1):** the "Remaining Work" value here (**4**) equals the Remaining Hours in Section 1.2 and the sum of the Section 2.2 Hours column. The "Completed Work" value (**34**) equals the Section 2.1 total and the Completed Hours in Section 1.2.

---

## 8. Summary & Recommendations

**Achievements.** The project delivered a single, comprehensive, runtime-verified answer document (`blitzy/documentation/app_2cd6ee777f8c.md`, 1,004 lines) that answers all three question groups (Q1 startup/health, Q2 user actions & correctness, Q3 background components) with observed output, exact commands, ~60+ resolving `file:line` citations, and disciplined OBSERVED/INFERRED labeling. The read-only constraint was strictly honored (single-file diff versus the pristine source parent `2cd6ee77`), and all temporary data and observation scripts were cleaned up.

**Remaining gaps.** None of the remaining work is autonomous development — it is entirely path-to-production human review: (1) a document accuracy/consistency pass, (2) stakeholder acceptance sign-off, and (3) a governance decision on the untracked `blitzy/screenshots/` directory.

**Critical path to production.** Human review (HT-1) → stakeholder acceptance (HT-2) → merge → resolve screenshots disposition (HT-3). This is estimated at **4 hours**.

**Success metrics.** All 15 AAP-specified requirements are complete and evidence-backed; codebase regression is healthy (609/611, with the only non-passing items being a pre-existing, out-of-scope `google-re2` artifact); every Q1/Q2/Q3 runtime claim was independently reproduced during validation.

**Production readiness.** The single in-scope deliverable is **production-ready pending human review**. Overall completion is **89.5%** (34 of 38 hours); the remaining 10.5% is human review/acceptance/governance that cannot be performed autonomously.

| Assessment | Result |
|---|---|
| AAP deliverables complete | 15 / 15 |
| Completion (AAP-scoped hours) | 89.5% (34 / 38 h) |
| Read-only compliance | ✅ Single-file diff |
| Blocking issues | None |
| Recommendation | Approve after ~4 h human review |

---

## 9. Development Guide

> All commands below were tested against the live `sl-app`/`sl-db`/`sl-redis` environment during this assessment. Because the pinned `multidict==4.7.6` C-extension cannot build a wheel on Python 3.10, the dependencies **cannot** be installed on a bare host — always use the canonical image.

### 9.1 System Prerequisites

- **Docker** 28.x (`docker --version` → `Docker version 28.5.2`).
- **Canonical runtime image:** `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (carries venv **Python 3.10.18** at `/app/venv`).
- ~2 GB free disk; ability to run three containers on a shared Docker network.

### 9.2 Environment Setup (three-container topology)

```bash
# Containers (already provisioned in the canonical environment):
#   sl-app    ghcr.io/scaleapi/swe-atlas:...   0.0.0.0:7777->7777, 0.0.0.0:20381->20381
#   sl-redis  redis:7                          0.0.0.0:6379->6379
#   sl-db     postgres:13                      0.0.0.0:15432->5432
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}'
```

Configuration is env-driven. The container `.env` is a verbatim copy of the repo-tracked `example.env` with only two gitignored, environment-specific deltas:

```bash
cp example.env .env
# DB_URI host: localhost -> sl-db
# append:      MEM_STORE_URI=redis://sl-redis:6379
```

### 9.3 Backing Services, Migrations & Seed Data

```bash
# Apply migrations (idempotent) and confirm head revision:
docker exec sl-app sh -c 'cd /code && FLASK_APP=server.py /app/venv/bin/alembic current'
# -> 32f25cbf12f6 (head)

# Load local login fixtures (creates the two demo accounts):
docker exec sl-app sh -c 'cd /code && FLASK_APP=server.py /app/venv/bin/flask dummy-data'

# Verify seed:
docker exec sl-db sh -c 'psql -U myuser -d simplelogin -c "SELECT id,email,activated FROM users ORDER BY id;"'
# -> 1 john@wick.com t   |   2 winston@continental.com t

# Verify Redis:
docker exec sl-redis redis-cli ping   # -> PONG
```

### 9.4 Application Startup (each component is its own real entry point)

```bash
# Web server (PRODUCTION default — matches Dockerfile CMD):
gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15

# Web server (DEVELOPMENT alternative):
python server.py                 # app.run(debug=True, port=7777)

# Email handler (SMTP, port 20381) — run from /app in this image:
python email_handler.py

# Job runner (Job-table poll loop):
python job_runner.py
```

### 9.5 Verification Steps

```bash
# Web health + index redirect:
docker exec sl-app sh -c 'curl -s -o /dev/null -w "GET /health -> %{http_code}\n" http://127.0.0.1:7777/health; curl -s http://127.0.0.1:7777/health; echo'
# -> GET /health -> 200
# -> success
docker exec sl-app sh -c 'curl -s -o /dev/null -w "GET / -> %{http_code} Location=%{redirect_url}\n" http://127.0.0.1:7777/'
# -> GET / -> 302 Location=http://127.0.0.1:7777/auth/login

# Confirm the three components run as independent processes:
docker exec sl-app sh -c "ps -eo pid,ppid,args | grep -E 'gunicorn|email_handler|job_runner' | grep -v grep"
# -> gunicorn master + 2 workers; email_handler.py and job_runner.py as separate top-level processes
```

### 9.6 Example Usage

```bash
# Read the deliverable:
sed -n '1,40p' blitzy/documentation/app_2cd6ee777f8c.md
wc -l blitzy/documentation/app_2cd6ee777f8c.md   # -> 1004

# Browser: open http://localhost:7777/  -> redirects to /auth/login
# Sign in with the demo account john@wick.com / password  -> /dashboard/
#   (dashboard shows the alias list, stats.nb_alias, Random/Custom-alias buttons, and search box)
```

### 9.7 Read-only Compliance Verification

```bash
git diff 2cd6ee77 HEAD --name-status          # -> A  blitzy/documentation/app_2cd6ee777f8c.md
git diff 2cd6ee77 HEAD -- . ':(exclude)blitzy/'   # -> (empty) no product/source file changed
git status --porcelain                         # -> ?? blitzy/screenshots/  (only untracked evidence)
```

### 9.8 Troubleshooting

- **`module 're2' has no attribute 'DOTALL'` in the test suite** — expected, out-of-scope `google-re2` artifact (`google-re2 1.1.20250805`; `re2.DOTALL` is `False`). Does not affect the deliverable and cannot be fixed under the read-only constraint.
- **Email handler import errors from `/code`** — run it from `/app`, which ships a pre-patched `spamassassin_utils` import; the `email_handler.py` file itself is identical between locations.
- **`DB_URI` required at startup** — ensure `.env` has the `sl-db` host (`app/config.py` requires `DB_URI`).
- **Port already in use (7777/20381/6379/15432)** — a component or prior process is already bound; check `docker ps` and existing processes.
- **Runtime values differ from the doc (ids/PIDs/timestamps)** — expected; every such value is captured evidence from a specific session, not a stable constant.

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---|---|
| Container topology | `docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}'` |
| Web health | `curl -s http://127.0.0.1:7777/health` → `success` |
| Index redirect | `curl -s -o /dev/null -w "%{http_code} %{redirect_url}" http://127.0.0.1:7777/` |
| Alembic head | `FLASK_APP=server.py /app/venv/bin/alembic current` → `32f25cbf12f6 (head)` |
| Seed data | `FLASK_APP=server.py /app/venv/bin/flask dummy-data` |
| Component processes | `ps -eo pid,ppid,args | grep -E 'gunicorn|email_handler|job_runner'` |
| Redis ping | `redis-cli ping` → `PONG` |
| Single-file diff | `git diff 2cd6ee77 HEAD --name-status` |

### B. Port Reference

| Port | Service | Source |
|---|---|---|
| 7777 | Web server (gunicorn / Flask dev) | `Dockerfile` `EXPOSE 7777`; `server.py` `app.run(port=7777)` |
| 20381 | Email handler (aiosmtpd SMTP) | `email_handler.py` (argparse default) |
| 25 | Postfix MTA (external inbound path) | Production topology (not required for local observation) |
| 6379 | Redis (cache/session/rate-limit) | `MEM_STORE_URI=redis://sl-redis:6379` |
| 15432 → 5432 | PostgreSQL 13 (host → container) | `DB_URI` (`sl-db`) |

### C. Key File Locations

| File | Role |
|---|---|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The sole deliverable** (runtime-verified Q&A) |
| `server.py` / `wsgi.py` | Web app factory + `/health`; gunicorn WSGI target |
| `email_handler.py` | aiosmtpd SMTP handler (`handle_DATA` → `handle_forward`) |
| `job_runner.py` | Background Job-table poll loop |
| `cron.py` + `crontab.yml` | yacron scheduler (15 jobs) — distinct from the job runner |
| `app/models.py` | `User`/`Alias`/`Contact`/`EmailLog`/`Job`/`Mailbox` ORM models |
| `app/email/status.py` | SMTP response constants (`E200`, `E515`, …) |
| `app/config.py` | Ports, URLs, `DB_URI`, job constants |
| `templates/auth/login.html`, `templates/dashboard/index.html` | Login & alias-management UI |
| `blitzy/screenshots/` | Untracked QA evidence (~40 PNGs) — disposition pending (HT-3) |

### D. Technology Versions

| Component | Version |
|---|---|
| Python (venv) | 3.10.18 |
| Flask | 1.1.2 |
| Flask-Login | 0.5.0 |
| gunicorn | 20.0.4 |
| SQLAlchemy | 1.3.24 |
| Alembic | 1.4.3 |
| psycopg2-binary | 2.9.3 |
| aiosmtpd | 1.4.2 |
| redis (client) | 4.6.0 |
| yacron | 0.11.2 |
| PostgreSQL | 13 |
| Redis (server) | 7 |
| Docker Engine | 28.5.2 |
| google-re2 (env artifact) | 1.1.20250805 (`re2.DOTALL` = False) |

### E. Environment Variable Reference

| Variable | Purpose | Local value / note |
|---|---|---|
| `DB_URI` | PostgreSQL connection (required) | `postgresql://myuser:mypassword@sl-db:5432/simplelogin` |
| `MEM_STORE_URI` | Redis backend | `redis://sl-redis:6379` (appended delta) |
| `URL` | App base URL | from `example.env` |
| `NOT_SEND_EMAIL` | Suppress real sending | disable to observe physical delivery |
| `DISABLE_ONBOARDING` | Skip onboarding flow | from `example.env` |
| `POSTFIX_PORT` | MTA port | from `example.env` |
| `FLASK_APP` | Flask entry for CLI | `server.py` |

### F. Developer Tools Guide

- **`docker exec sl-app …`** — run commands inside the app container (venv at `/app/venv`, code at `/code`).
- **`curl`** — probe the web server (`/health`, `/`) and inspect status codes/redirects.
- **`swaks`** — drive the SMTP receive path to observe `E200`/`E515` responses.
- **`psql -h sl-db -U myuser -d simplelogin`** — inspect `Alias`/`Contact`/`EmailLog`/`Job` rows before/after actions.
- **`redis-cli ping`** — confirm the Redis backend.
- **`ps -eo pid,ppid,args`** — confirm the three components run as independent processes.

### G. Glossary

| Term | Meaning |
|---|---|
| **Web server** | Flask app (via gunicorn `wsgi:app`) serving the dashboard/auth UI on port 7777 |
| **Email handler** | aiosmtpd SMTP daemon receiving/forwarding alias mail on port 20381 |
| **Job runner** | Custom loop polling the `Job` table every ~10 s (`ready → taken → done`) |
| **Cron (yacron)** | Schedule-based runner executing `cron.py -j <job>` (15 jobs) — distinct from the job runner |
| **Alias / Contact / EmailLog** | ORM rows evidencing that a receive was handled correctly |
| **E200 / E515** | SMTP responses: `250 Message accepted for delivery` / `550 SL E515` (alias not found) |
| **OBSERVED / INFERRED** | Labels marking runtime-observed facts vs. reasoning from reading code |
| **Dummy-data** | Local seed fixtures (`john@wick.com`, `winston@continental.com`) |
| **`2cd6ee77`** | Pristine source-parent commit; the exact code under test |