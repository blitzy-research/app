# Blitzy Project Guide — SimpleLogin Local Runtime Verification (Documentation Deliverable)

> **Deliverable under assessment:** `blitzy/documentation/app_2cd6ee777f8c.md` — a single, evidence-backed Markdown answer document (2,211 lines) that answers four first-time-operator runtime-verification questions about a locally running SimpleLogin deployment.
>
> **Task class:** Documentation / runtime-investigation (read-only). No source code was to be modified.
>
> **Brand color key:** Completed / AI Work = Dark Blue `#5B39F3` · Remaining / Not Completed = White `#FFFFFF` · Headings / Accents = Violet-Black `#B23AF2` · Highlight = Mint `#A8FDD9`.

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivered a comprehensive, evidence-backed operator runbook that answers a first-time operator's runtime-verification questions about a locally running **SimpleLogin** deployment — a Flask-based email-alias and privacy service. The single artifact, `blitzy/documentation/app_2cd6ee777f8c.md`, documents (Q1) the observable readiness signals in logs and UI, (Q2) the register → verify → login → dashboard new-user flow, (Q3) the behind-the-scenes background jobs and internal services that support email forwarding and identity verification, and (Q4) the read-only + cleanup constraints. The work was performed under a strict **Run-First** discipline: every behavioral claim is paired with the exact command, its raw observed output, and a `file:line` citation — grounded in actually building, running, and observing the system without changing a single line of source.

### 1.2 Completion Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeWidth':'2px','pieTitleTextSize':'16px','pieSectionTextColor':'#B23AF2','pieLegendTextColor':'#333333','pieOpacity':'1'}}}%%
pie showData title AAP Completion — 92.8% Complete
    "Completed Work (AI)" : 64
    "Remaining Work" : 5
```

<div align="center"><strong>92.8% Complete</strong></div>

| Metric | Value |
|--------|-------|
| **Total Hours** | **69** |
| **Completed Hours (AI + Manual)** | **64** (64 AI + 0 Manual) |
| **Remaining Hours** | **5** |
| **Percent Complete** | **92.8%** |

> **Calculation (PA1, AAP-scoped, hours-based):** Completion % = Completed Hours ÷ (Completed + Remaining) = 64 ÷ (64 + 5) = 64 ÷ 69 = **92.8%**. All 27 discrete AAP requirements are Completed; the remaining 5 hours are path-to-production activities (human acceptance review, optional canonical re-run, PR merge). Capped below 99% pending human review per honest-assessment policy.

### 1.3 Key Accomplishments

- ✅ **Sole AAP deliverable authored and committed** — `blitzy/documentation/app_2cd6ee777f8c.md` (2,211 lines, ~20,773 words, 162 code blocks, 236 embedded `file:line` citation references across 30 unique source files).
- ✅ **Read-only constraint honored perfectly** — `git diff --name-status 2cd6ee77..HEAD` shows exactly **one file added** (+2,211 / −0); zero source files changed; working tree clean.
- ✅ **Full local runtime provisioned and exercised** — Python 3.10 + PostgreSQL 15 + Redis, `alembic upgrade head` (head `32f25cbf12f6`, 77 tables), `flask dummy-data` seeding, and all four entry points booted (`server.py` :7777, `job_runner.py`, `email_handler.py` :20381, `cron.py`).
- ✅ **All four questions (Q1–Q4) answered by name** with live-observed evidence — readiness banners, the register→verify→login→dashboard flow (15 sub-flows), the Postgres `LISTEN/NOTIFY simplelogin_sync_events` event bus, the 10-second job poll loop, and the live SMTP forwarding engine.
- ✅ **Citation integrity validated** — the Final Validator audited 176/176 distinct citations as resolving to canonical source; an independent spot-check of 6/6 sampled citations resolved exactly.
- ✅ **Exact database baseline restoration proven** — the temporary probe user (id=7) and its default alias (id=18) were removed via canonical ORM cascade; all 77 tables restored to the seeded baseline (0 post-seed rows).
- ✅ **Honest, transparent evidence discipline** — non-canonical values (the image-patched RE2 binding) are explicitly labeled, and one genuine image-config defect (a broken bundled `yacron` schedule) is reported rather than hidden.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| _None — no release-blocking issues._ The deliverable is fully authored, runtime-validated, citation-audited, and committed with a clean working tree. | None | — | — |

> All items below the release bar are tracked as **non-blocking** risks in Section 6 (all Low severity) and as optional/next-step human tasks in Sections 1.6 and 2.2.

### 1.5 Access Issues

| System / Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-------------------|----------------|-------------------|-------------------|-------|
| Git repository (branch `blitzy-3f604de9…`) | Read/Write | None — deliverable committed; working tree clean | ✅ No issue | — |
| Canonical Docker image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` | Registry pull | Reproducing the live observations (re-running §Q4.4) requires pulling this image; registry access needed by whoever re-runs the observations | ⚠ Non-blocking (deliverable already validated) | Reviewer / DevOps |

> No access issue blocks acceptance of the deliverable itself — it is already authored, validated, and committed. The single dependency is registry access for anyone who wishes to **re-reproduce** the observations from scratch.

### 1.6 Recommended Next Steps

1. **[High]** Perform human technical review and acceptance of `blitzy/documentation/app_2cd6ee777f8c.md` — confirm Q1–Q4 are answered accurately and completely, and spot-check a sample of the 236 citation references against source. _(≈3.5h)_
2. **[Medium]** Approve and merge the pull request; confirm exactly one file was added; archive the document as the operator runbook. _(≈0.5h)_
3. **[Low]** _(Optional polish)_ Re-run `email_handler.py` in a pristine canonical container (shipping `pyre2`, not the image-patched `google-re2` shim) to capture identical startup banners and drop the non-canonical RE2 caveat in §0.7 / §Q1.5. _(≈1h)_

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

Every completed component traces to a specific AAP requirement (R1–R27) or a documented section (§0–§Q4) of the deliverable.

| Component | Hours | Description |
|-----------|-------|-------------|
| Runtime provisioning & entry-point bring-up | 7 | R1–R2 — Provision Python 3.10 + PostgreSQL 15 + Redis, resolve pinned deps, out-of-repo `CONFIG`, `alembic upgrade head` (head `32f25cbf12f6`, 77 tables), `flask dummy-data` seeding, and boot all four entry points. |
| §0 Environment, run-recipe & logging documentation | 6 | §0.1–0.8 — Environment/versions, entry points, canonical run recipe, config knobs + precedence, `dummy-data` bootstrap, the "SL" logging backbone, non-canonical disclosures, and trusted-local-only security posture. |
| Q1 Readiness-signals investigation & write-up | 5 | OBJ-1 / R3–R7 — Flask banner + `>>> init logging <<<`, `GET /` → 302 → `/auth/login`, `/git` `/live` `/exception` health endpoints, `job_runner` readiness, `email_handler` controller banners. |
| Q2 New-user-flow investigation & write-up | 15 | OBJ-2 / R8–R13 — register → verify → login → dashboard plus 15 sub-flows (error/boundary, MFA, CSRF, rate-limit 429, logout, next-redirect safety, method matrix 405) with `activated` `f`→`t` DB transitions. |
| Q3 Behind-the-scenes services investigation & write-up | 12 | OBJ-3 / R14–R18 — mail-path convergence, `job_runner` 10s cadence (×2 runs), Postgres `LISTEN/NOTIFY` round-trip, LIVE SMTP forwarding (250 forwarded / 550 SL E515 / 250-not-forwarded), `cron`/`yacron`. |
| Q4 Constraints, cleanup & baseline-restoration proofs | 5 | OBJ-4 / R19–R22 — read-only proof, ORM cascade cleanup + exact 77-table restoration, targeted-PID teardown, reproduce-from-scratch recipe. |
| Evidence discipline: 236 citations + coverage pass | 6 | Rules 1–4 / R23–R25 — 236 embedded `file:line` citation references, canonical/non-canonical labeling, and the Rule-4 coverage pass. |
| QA review cycles & final runtime re-validation | 8 | R26–R27 — full rewrite + 40+ findings resolved across 5 review cycles; final-validator re-verification, 176/176 citation audit, typo fix, and commit. |
| **Total Completed** | **64** | |

> **Validation:** The Hours column sums to **64**, matching the Completed Hours in Section 1.2. ✓

### 2.2 Remaining Work Detail

Each remaining category is a path-to-production activity for a documentation deliverable (human acceptance, optional polish, merge) — not outstanding AAP implementation work.

| Category | Hours | Priority |
|----------|-------|----------|
| Human technical review & acceptance of the answer document | 3.5 | High |
| Optional pristine-canonical `email_handler` re-run (drop non-canonical RE2 label) | 1 | Low |
| PR merge / publish & archive | 0.5 | Medium |
| **Total Remaining** | **5** | |

> **Validation:** The Hours column sums to **5**, matching the Remaining Hours in Section 1.2 and the "Remaining Work" value in the Section 7 pie chart. ✓

### 2.3 Hours Reconciliation

| Quantity | Hours | Source |
|----------|-------|--------|
| Completed (Section 2.1) | 64 | Sum of 8 completed components |
| Remaining (Section 2.2) | 5 | Sum of 3 remaining categories |
| **Total Project Hours** | **69** | 64 + 5 |
| **Percent Complete** | **92.8%** | 64 ÷ 69 |

> **Cross-section integrity:** 64 (2.1) + 5 (2.2) = **69** = Total Project Hours in Section 1.2 (Rule 2 ✓). Remaining = **5** across Sections 1.2, 2.2, and 7 (Rule 1 ✓).

---

## 3. Test Results

All entries below originate exclusively from **Blitzy's autonomous validation logs** for this project (the Final Validator's runtime re-verification and static citation audit), corroborated by an independent citation spot-check performed during this assessment. Because this is a strictly read-only documentation task with **zero source files changed**, SimpleLogin's own `pytest` suite was intentionally **not executed** — it is out of AAP scope, there is no changed source whose unit behavior could be affected, and running it would create test-database artifacts that would violate the cleanliness constraint. The applicable "tests" for a documentation deliverable are the runtime re-verifications of every documented claim plus the citation and structural audits.

| Test Category | Framework / Method | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|--------------------|-------------|--------|--------|------------|-------|
| Static Citation Audit | Blitzy `file:line` resolver (validator) | 176 | 176 | 0 | 100% | All distinct citations resolve to canonical host source; independently spot-checked 6/6. Document embeds 236 total citation references across 30 files. |
| Q1 Readiness Runtime Re-Verification | Live runtime observation (`curl` + log capture) | 5 | 5 | 0 | 100% | `>>> init logging <<<` (×2 w/ reloader) + Flask banner; `GET /` 302→`/auth/login`→200; `/git` 200; `/live` 200; `/exception` 500; worker banners. |
| Q2 New-User Flow Runtime Re-Verification | Live HTTP + DB state capture | 15 | 15 | 0 | 100% | register→activate→login→dashboard + boundary/MFA/CSRF/rate-limit(429)/logout/next-redirect/method-matrix(405); `activated` `f`→`t`. |
| Q3 Behind-the-Scenes Runtime Re-Verification | Live process + Postgres `LISTEN/NOTIFY` | 6 | 6 | 0 | 100% | mail path under `NOT_SEND_EMAIL`; 10s poll ×2 (10.016s / 10.017s); `NOTIFY` round-trip; LIVE SMTP 250/550/250; `cron.py -j stats` exit 0 (+1 image `yacron` defect honestly reported). |
| Q4 Constraints & Cleanup Verification | `git` + ORM baseline diff | 5 | 5 | 0 | 100% | read-only proof; exact 77-table restoration (0 post-seed rows); targeted-PID teardown; reproduce recipe; 1-file `git` proof. |
| Markdown Structural Lint | fence/header/anchor/placeholder checker | 1 | 1 | 0 | 100% | 162 balanced code blocks; internal anchors resolve; no stray incomplete-work placeholders. |
| **Total** | | **208** | **208** | **0** | **100%** | Zero failures across all autonomous validation checks. |

> **Integrity note (Rule 3):** No test in this table is fabricated or borrowed from an external suite — every entry is a check that Blitzy's autonomous agents actually executed against the live runtime or the committed artifact.

---

## 4. Runtime Validation & UI Verification

**Legend:** ✅ Operational · ⚠ Partial / caveated · ❌ Failing

**Runtime health (live-observed):**
- ✅ **Web application** — `server.py` on `http://localhost:7777` via `app.run(debug=True, port=7777)` [server.py:L588].
- ✅ **Root redirect** — `GET /` → `302` → `/auth/login` → `200` (unauthenticated).
- ✅ **Monitor `/git`** — `200`, returns build SHA1 `"dev"` [app/monitor/views.py:L5-L7].
- ✅ **Monitor `/live`** — `200`, returns `"live"` [app/monitor/views.py:L10-L11].
- ✅ **Monitor `/exception`** — `500` by design (deliberate exception probe) [app/monitor/views.py:L15-L16].
- ✅ **Background job worker** — `job_runner.py` 10-second poll loop measured at 10.016s / 10.017s [job_runner.py:L347].
- ✅ **Event bus** — Postgres `LISTEN/NOTIFY simplelogin_sync_events` full producer→consumer round-trip [app/events/event_dispatcher.py:L14].
- ⚠ **SMTP forwarding engine** — `email_handler.py` booted LIVE binding aiosmtpd on port 20381 (`Listen for port 20381` [email_handler.py:L2403]); **caveat:** booted under a non-canonical image-patched RE2 binding (transparently labeled in §0.7 / §Q1.5). Live forwarding observed (250 forwarded / 550 SL E515 / 250-not-forwarded).
- ⚠ **Scheduled tasks** — `cron.py -j stats` entry runs (exit 0); **caveat:** the bundled `yacron` schedule references `/code/cron.py` while the repo lives at `/app`, so the scheduled child exits 2 — an honestly-reported image-config defect (out-of-scope to fix under read-only).

**UI verification (rendered-template + HTTP evidence):**
- ✅ **Registration UI** — `POST /auth/register` → `200` rendering `register_waiting_activation.html` [app/auth/views/register.py:L31].
- ✅ **Activation** — `GET /auth/activate?code=…` flips `users.activated` `f`→`t`, consumes the single-use code, and `302` → `/dashboard/` [app/auth/views/activate.py:L50-L53].
- ✅ **Login routing** — `after_login()` → `dashboard.index` [app/auth/views/login_utils.py:L40-L46]; `john@wick.com / password` → `302` → `/dashboard/`.
- ✅ **Dashboard UI** — `GET /dashboard/` → `200` rendering `dashboard/index.html` (Alias/Mailbox view) [app/dashboard/views/index.py:L216].

> **UI evidence method:** The final deliverable intentionally uses reproducible **text-based** evidence (HTTP status codes + rendered-template names + database state transitions) rather than screenshots; two early screenshots were removed during the full rewrite in favor of this more durable, citation-grounded evidence.

---

## 5. Compliance & Quality Review

This matrix cross-maps each AAP deliverable/rule to Blitzy's quality and compliance benchmarks. Fixes applied during autonomous validation are noted.

| Benchmark / AAP Rule | Requirement | Status | Progress | Notes / Fixes Applied |
|----------------------|-------------|--------|----------|-----------------------|
| **MainRule — Deliverable & scope** | One Markdown doc at `blitzy/documentation/app_2cd6ee777f8c.md`, read-only repo | ✅ Pass | 100% | Exactly one file added; 0 source files changed; clean tree. |
| **Rule 1 — Run-First investigation** | Build & run first; observe real output | ✅ Pass | 100% | All Q1–Q4 signals captured live; per-run-variable fields annotated. |
| **Rule 2 — Exhaustive coverage** | Enumerate every condition/variant/state | ✅ Pass | 100% | 15 Q2 sub-flows (error/MFA/CSRF/rate-limit/logout/method-matrix); before/during/after state captured. |
| **Rule 3 — Faithful instruction-following** | Exact entry points; observed output beside each claim; label inferred | ✅ Pass | 100% | Real entry points used; non-canonical RE2 labeled; found `yacron` defect reported honestly. |
| **Rule 4 — Complete, precise, grounded** | Answer each named item; file:line + function names | ✅ Pass | 100% | Coverage pass maps every named Q1–Q4 item to its answering section + signal. |
| **OBJ-1 — Readiness (Q1)** | Startup indicators in logs/UI | ✅ Pass | 100% | §Q1.1–Q1.5 fully answered. |
| **OBJ-2 — New-user flow (Q2)** | register→verify→login→dashboard | ✅ Pass | 100% | §Q2.1–Q2.15 fully answered. |
| **OBJ-3 — Behind-the-scenes (Q3)** | Background jobs & internal services | ✅ Pass | 100% | §Q3.1–Q3.6 fully answered. |
| **OBJ-4 — Constraints (Q4)** | Cleanup temp artifacts; no source changes | ✅ Pass | 100% | §Q4.1–Q4.5; exact baseline restoration proven. |
| **Zero-placeholder policy** | No stray TODO/FIXME/placeholder markers | ✅ Pass | 100% | Structural lint clean; only legitimate narrative/`mktemp` template tokens. |
| **Citation integrity** | Every claim carries a resolvable `file:line` | ✅ Pass | 100% | 176/176 audited; 6/6 independently spot-checked. Fix applied: §Q3.4 citation path corrected (`app/handler/status.py`→`app/email/status.py`) and a §Q3.5→§Q3.4 cross-reference typo. |
| **Markdown structure** | Balanced fences, resolvable anchors | ✅ Pass | 100% | 162 balanced code blocks; all internal `§Qx.y` refs resolve. |

**Autonomous fixes applied during validation (summary):** full rewrite from genuine evidence resolving 29 initial review findings; resolution of F1/F2/F3, F-Q2.2, I-1, I-2, and 7 MAJOR QA findings across multiple cycles; a §Q3.4 citation-path correction; and a final §Q3.5→§Q3.4 cross-reference typo fix (commit `58e09c43`).

---

## 6. Risk Assessment

Overall risk posture: **LOW** — a read-only documentation deliverable with zero source changed, fully runtime-validated, and a clean repository.

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Non-canonical RE2 binding for `email_handler` boot (image-patched `import re` vs canonical `re2`) | Technical | Low | Low | Transparently labeled in §0.7 + §Q1.5; startup banners are canonical text/`file:line`; optional pristine re-run available | Documented / Mitigated |
| Per-run-variable fields (PIDs, timestamps, GNUPGHOME, cookies, Content-Length) not byte-reproducible | Technical | Low | Medium | Annotated as per-run-variable; claims rest on invariant signals | Documented / Accepted |
| Citation line-number drift if source evolves | Technical | Low | Low | Document pinned to branch + base HEAD `2cd6ee77` | Accepted |
| Local debug runtime `app.run(debug=True)` — Flask Debug Toolbar renders SECRET_KEY + DB URI | Security | Low (informational) | Low | §0 documents trusted-local-only posture; never production; zero source change | Documented / N-A-to-prod |
| Temporary probe-user credentials created during observation | Security | Low | Low | Fully cleaned; exact baseline restoration proven (0 post-seed rows) | Resolved |
| Bundled `yacron` schedule broken in canonical image (`/code/cron.py` vs `/app` → exit 2) | Operational | Low (informational) | N/A (pre-existing image defect) | Honestly reported in §Q3.5; `cron.py` entry itself runs (exit 0); out-of-scope to fix under read-only | Reported / Out-of-scope |
| Reproducibility depends on canonical Docker image availability | Operational | Low | Low | §Q4.4 full executable bring-up + teardown; image tag pinned | Mitigated |
| No CI to auto-re-run documented observations; accuracy relies on point-in-time capture + human review | Integration | Low | Low | §Q4.4 reproduce recipe enables on-demand re-verification | Accepted |
| Doc-only deliverable; no product runtime/deploy integration required or performed | Integration | None | N/A | Zero source change; nothing to integrate | N/A |

---

## 7. Visual Project Status

### 7.1 Project Hours Breakdown

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeWidth':'2px','pieTitleTextSize':'16px','pieSectionTextColor':'#B23AF2','pieLegendTextColor':'#333333','pieOpacity':'1'}}}%%
pie showData title Project Hours — Completed vs Remaining
    "Completed Work" : 64
    "Remaining Work" : 5
```

> **Completed Work = 64h** (Dark Blue `#5B39F3`) · **Remaining Work = 5h** (White `#FFFFFF`). The "Remaining Work" value (5) equals the Remaining Hours in Section 1.2 and the sum of the Section 2.2 Hours column (Rule 1 ✓).

### 7.2 Remaining Hours by Category (Section 2.2)

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'xyChart': {'plotColorPalette': '#5B39F3'}}}}%%
xychart-beta
    title "Remaining Hours by Category (Total = 5h)"
    x-axis ["Human Review (High)", "Canonical Re-run (Low)", "PR Merge (Medium)"]
    y-axis "Hours" 0 --> 4
    bar [3.5, 1, 0.5]
```

### 7.3 AAP Requirement Status Distribution

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#A8FDD9','pie3':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieSectionTextColor':'#B23AF2','pieLegendTextColor':'#333333'}}}%%
pie showData title AAP Requirements (27 total)
    "Completed" : 27
    "Partially Completed" : 0
    "Not Started" : 0
```

---

## 8. Summary & Recommendations

### 8.1 Achievements

The autonomous agents delivered a complete, evidence-backed operator runbook for a locally running SimpleLogin instance and did so under a strict read-only constraint that was honored perfectly: exactly one file was added to the repository and no source was modified. All **27 discrete AAP requirements** (R1–R27) spanning provisioning, the four operator questions (Q1–Q4), the five methodology rules, and the deliverable-authoring constraints are **Completed**. Every behavioral claim in the 2,211-line document is grounded in a command, its raw observed output, and a `file:line` citation, and the Final Validator reproduced every documented signal live and audited 176/176 distinct citations.

### 8.2 Remaining Gaps & Critical Path to Production

For a documentation deliverable, "production" means human acceptance and publication rather than software deployment. The critical path is short (**5 hours** total): (1) **human technical review and acceptance** of the document (3.5h, High), (2) **PR merge/publish and archive** (0.5h, Medium), and (3) an **optional** pristine-canonical `email_handler` re-run to drop the (AAP-permitted) non-canonical RE2 label (1h, Low). There are no release-blocking issues and no outstanding AAP implementation work.

### 8.3 Production Readiness Assessment

| Dimension | Assessment |
|-----------|------------|
| AAP completion | **92.8%** (64 of 69 hours) — all 27 requirements Completed |
| Deliverable quality | High — evidence discipline enforced; 176/176 citations audited; structural lint clean |
| Constraint compliance | Full — read-only honored (1 file added), exact DB baseline restored |
| Risk posture | Low across all categories |
| Blocking issues | None |
| Confidence | High — well-defined scope, fully runtime-validated |

### 8.4 Success Metrics

- **Read-only compliance:** 1 file added / 0 source files changed ✅
- **Question coverage:** Q1–Q4 all answered by name with live evidence ✅
- **Citation integrity:** 176/176 audited + 6/6 independent spot-check ✅
- **Cleanup:** exact 77-table baseline restoration, 0 post-seed rows ✅
- **Validation gates:** all 5 production-readiness gates PASS ✅

> **Bottom line:** The project is **92.8% complete**. The sole deliverable is authored, validated, and committed; the remaining ~5 hours are human review, merge, and optional polish.

---

## 9. Development Guide

This guide explains how to build, run, verify, and troubleshoot the SimpleLogin local runtime that the deliverable documents. It is grounded in the canonical run recipe (`CONTRIBUTING.md`) and the deliverable's §0 environment section.

### 9.1 System Prerequisites

- **Canonical runtime (recommended):** the pinned Docker image
  `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`
  (tag `simple-login__app__2cd6ee777f8c…`). It ships a pre-built virtualenv at `/app/venv` (**Python 3.10.18**), PostgreSQL, Redis, and all pinned dependencies.
- **Services:** PostgreSQL 15, Redis. Ports: **7777** (web), **20381** (SMTP / `email_handler`).
- **Interpreter note:** run inside the canonical image. A host interpreter (e.g., Python 3.13) is **not** the documented environment and may fail to import optional native extensions (see Troubleshooting).

### 9.2 Environment Setup

Keep configuration **outside** the repository checkout to preserve the read-only constraint:

```bash
# Copy the template to an out-of-repo location and point CONFIG at it
cp /app/example.env /root/sl.env
export CONFIG=/root/sl.env
```

- `app/config.py` `get_abs_path()` passes an absolute `CONFIG` path through unchanged [app/config.py:L14-L20]; it is consumed at import time by the `load_dotenv` block [app/config.py:L65-L71].
- `load_dotenv()` uses the default `override=False`, so shell environment variables win over file values.
- Canonical local-dev knobs (present in `example.env`): `URL=http://localhost:7777` (L6), `NOT_SEND_EMAIL=true` (L19), `EMAIL_DOMAIN=sl.local` (L22), `DISABLE_ONBOARDING=true` (L150). `NOT_SEND_EMAIL=true` makes the mailer **log** email content instead of sending it — this is how the activation code is observed.

### 9.3 Dependency Installation

```bash
# Inside the canonical image, dependencies are pre-built — activate the venv:
source /app/venv/bin/activate
python3 --version   # -> Python 3.10.18
```

- The source-canonical install command is `poetry sync` (`CONTRIBUTING.md`), but the canonical image **does not ship a `poetry` binary** and already contains the resolved environment — so **skip `poetry sync`** and activate `/app/venv`.
- Representative pinned versions observed: Flask 1.1.2, Werkzeug 1.0.1, Flask-Login 0.5.0, Flask-WTF 0.14.3, WTForms 2.3.3, SQLAlchemy 1.3.24, psycopg2-binary 2.9.3, Flask-Migrate 2.5.3, bcrypt 3.2.0, pyotp 2.4.0, webauthn 0.4.7, gunicorn 20.0.4, coloredlogs 14.0, alembic 1.4.3, yacron 0.11.2.
- Two observed drifts from the technical spec (reported honestly; neither affects auth/readiness behavior): `aiosmtpd==1.4.2` (spec 1.2), `redis==4.6.0` (spec 4.5.3).

### 9.4 Application Startup

```bash
# Start backing services (idempotent)
pg_ctlcluster 15 main start || true
redis-server --daemonize yes || true

# Initialize schema, seed demo data, and launch the web app
export CONFIG=/root/sl.env
alembic upgrade head            # migration head: 32f25cbf12f6
flask dummy-data                # seeds john@wick.com (id 1) + winston@continental.com (id 2)
python3 server.py               # Flask dev server on :7777  [server.py:L588]
```

Start the background workers in separate shells (each with `CONFIG` exported):

```bash
export CONFIG=/root/sl.env
python3 job_runner.py           # 10-second job poll loop  [job_runner.py:L347]
python3 email_handler.py        # aiosmtpd SMTP controller on :20381  [email_handler.py:L2403]
# Optional: python3 cron.py -j stats   ;  python3 event_listener.py
```

- `flask dummy-data` invokes `dummy_data()` [server.py:L490-L497], which calls `fake_data()`, `add_sl_domains()`, and `add_proton_partner()`.
- `wsgi.py` is the **production** entry point (`from server import create_app`); `server.py`'s `app.run(...)` is the **local dev** server.

### 9.5 Verification Steps

```bash
# Readiness — expect a 302 to /auth/login, then 200
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:7777/          # 302
curl -s -L -o /dev/null -w "%{http_code}\n" http://localhost:7777/        # 200 (login)

# Health endpoints
curl -s http://localhost:7777/git      # -> dev
curl -s http://localhost:7777/live     # -> live
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:7777/exception  # 500 (by design)
```

- In the `server.py` log you should see `>>> init logging <<<` (twice, due to the Flask reloader) and the Werkzeug startup banner.
- `job_runner.py` logs its poll loop (~10s cadence); `email_handler.py` logs `Start mail controller 0.0.0.0 20381` and `Listen for port 20381`.
- **Login smoke test:** open `http://localhost:7777`, sign in with `john@wick.com / password` (`CONTRIBUTING.md`) → `302` → `/dashboard/`.

### 9.6 Example Usage — New-User Flow (curl)

```bash
# 1) Fetch the register page and extract the CSRF token
curl -s -c /tmp/jar.txt http://localhost:7777/auth/register  # scrape csrf_token from HTML

# 2) Register (expect HTTP 200 rendering register_waiting_activation.html)
curl -s -b /tmp/jar.txt -c /tmp/jar.txt \
  -d "email=probe@example.com&password=Password123&csrf_token=<TOKEN>" \
  -o /dev/null -w "%{http_code}\n" http://localhost:7777/auth/register     # 200

# 3) With NOT_SEND_EMAIL=true, read the 30-char ActivationCode from the server log
#    (the activation email is logged, not sent)

# 4) Activate (expect activated f->t and a 302 to /dashboard/)
curl -s -b /tmp/jar.txt -o /dev/null -w "%{http_code}\n" \
  "http://localhost:7777/auth/activate?code=<CODE>"                        # 302

# 5) Load the dashboard (expect 200)
curl -s -b /tmp/jar.txt -o /dev/null -w "%{http_code}\n" \
  http://localhost:7777/dashboard/                                          # 200
```

### 9.7 Troubleshooting

- **`email_handler.py` fails to import on a non-canonical host** — a `google-re2` shim lacking stdlib `re` flags trips `re.compile(..., re.DOTALL)` at `app/spamassassin_utils.py`. **Fix:** use the canonical Docker image (ships `pyre2`), or note the image-patched `import re` as non-canonical (as the deliverable does).
- **`yacron` scheduled job exits 2** — the bundled schedule invokes `/code/cron.py` while the repo lives at `/app`. **Fix (local testing):** run `python3 cron.py -j <job>` directly; the entry point itself runs (exit 0). This is a shipped image-config defect, not a SimpleLogin code defect.
- **Debug Toolbar leaks secrets** — the local debug server renders `SECRET_KEY` and the DB URI via the Flask Debug Toolbar [server.py:L576-L582]. Treat the runtime as **trusted-local-only**; never bind it to a public interface.
- **`CONFIG` not picked up** — ensure `CONFIG` is an **absolute** path and exported in the same shell before launching each process; remember shell env overrides file values (`override=False`).
- **Cleanup (mandatory for read-only hygiene)** — stop services by their captured PIDs; delete any probe users via the canonical ORM cascade (dynamic `id > 2`); remove `/root/sl.env` and `/tmp` scratch; confirm `git status` is clean.

---

## 10. Appendices

### Appendix A — Command Reference

| Purpose | Command |
|---------|---------|
| Point config out-of-repo | `cp /app/example.env /root/sl.env && export CONFIG=/root/sl.env` |
| Activate pre-built venv | `source /app/venv/bin/activate` |
| Start PostgreSQL | `pg_ctlcluster 15 main start` |
| Start Redis | `redis-server --daemonize yes` |
| Apply migrations | `alembic upgrade head` |
| Seed demo data | `flask dummy-data` |
| Run web app | `python3 server.py` |
| Run job worker | `python3 job_runner.py` |
| Run SMTP handler | `python3 email_handler.py` |
| Run a cron job | `python3 cron.py -j stats` |
| Verify read-only | `git diff --name-status 2cd6ee77..HEAD` |
| Confirm clean tree | `git status --porcelain` |

### Appendix B — Port Reference

| Port | Service | Source |
|------|---------|--------|
| 7777 | Flask web application (dev server) | `app.run(debug=True, port=7777)` [server.py:L588] |
| 20381 | `email_handler` aiosmtpd SMTP controller | `Listen for port 20381` [email_handler.py:L2403] |
| 5432 | PostgreSQL (default) | backing store + `LISTEN/NOTIFY` |
| 6379 | Redis (default) | sessions / rate limiting |

### Appendix C — Key File Locations

| Path | Role |
|------|------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The deliverable** — evidence-backed answer document |
| `server.py` | Web entry point; `create_app`, dev server, `dummy-data` CLI |
| `wsgi.py` | Production WSGI entry point |
| `email_handler.py` | SMTP forwarding engine (aiosmtpd) |
| `job_runner.py` | Background job worker (10s poll) |
| `cron.py` / `crontab.yml` | Scheduled tasks (yacron) |
| `event_listener.py` / `events/` | Event consumer + producer/sink machinery |
| `app/config.py` | Env/`CONFIG` loading; absolute-path passthrough |
| `app/log.py` | The "SL" logger backbone |
| `app/monitor/views.py` | `/git`, `/live`, `/exception` health endpoints |
| `app/auth/views/` | `register.py`, `activate.py`, `login.py`, `login_utils.py` |
| `app/events/event_dispatcher.py` | `NOTIFY simplelogin_sync_events` publisher |
| `migrations/` | Alembic chain (head `32f25cbf12f6`) |
| `example.env` | Config template (URL, NOT_SEND_EMAIL, EMAIL_DOMAIN, DISABLE_ONBOARDING) |

### Appendix D — Technology Versions

| Component | Version | Notes |
|-----------|---------|-------|
| Python | 3.10.18 | Canonical image `/app/venv` |
| Flask | 1.1.2 | |
| Werkzeug | 1.0.1 | Observed in `Server:` header |
| Flask-Login | 0.5.0 | |
| Flask-WTF / WTForms | 0.14.3 / 2.3.3 | Forms + CSRF |
| SQLAlchemy | 1.3.24 | ORM |
| psycopg2-binary | 2.9.3 | Postgres driver + `LISTEN/NOTIFY` |
| Flask-Migrate / Alembic | 2.5.3 / 1.4.3 | Migration head `32f25cbf12f6` |
| bcrypt | 3.2.0 | Password hashing |
| pyotp / webauthn | 2.4.0 / 0.4.7 | MFA branches |
| aiosmtpd | 1.4.2 | ⚠ spec says 1.2 (honest drift) |
| redis | 4.6.0 | ⚠ spec says 4.5.3 (honest drift) |
| gunicorn | 20.0.4 | Production WSGI (unused locally) |
| coloredlogs | 14.0 | Log formatting |
| yacron | 0.11.2 | Cron scheduler |
| PostgreSQL / Redis | 15 / (image) | Backing services |

### Appendix E — Environment Variable Reference

| Variable | Value (local dev) | Source | Effect |
|----------|-------------------|--------|--------|
| `CONFIG` | `/root/sl.env` | shell | Absolute path to the out-of-repo config file |
| `URL` | `http://localhost:7777` | example.env:L6 | Base URL used to build links (e.g., activation) |
| `NOT_SEND_EMAIL` | `true` | example.env:L19 | Mailer logs email content instead of sending |
| `EMAIL_DOMAIN` | `sl.local` | example.env:L22 | Alias domain |
| `DISABLE_ONBOARDING` | `true` | example.env:L150 | Suppresses onboarding jobs |

### Appendix F — Developer Tools Guide

- **Reproduce all observations from scratch:** follow the deliverable's §Q4.4 — an executable bring-up (Postgres + Redis + config + `alembic upgrade head` + `flask dummy-data` + the four entry points) with bounded-wait readiness checks and an idempotent, targeted-PID teardown block.
- **Inspect the event bus:** issue a Postgres `LISTEN simplelogin_sync_events;` in one `psql` session and observe `NOTIFY` payloads emitted by `EventDispatcher.send_event` during activity.
- **Observe the activation code:** with `NOT_SEND_EMAIL=true`, tail the `server.py` log after `POST /auth/register` to read the 30-character single-use `ActivationCode`.
- **Verify cleanliness:** `git status --porcelain` (empty) and `git diff --name-status 2cd6ee77..HEAD` (exactly one `A` line) prove the read-only constraint.

### Appendix G — Glossary

| Term | Meaning |
|------|---------|
| **AAP** | Agent Action Plan — the authoritative statement of project scope and requirements |
| **Readiness signal** | An observable log line, HTTP response, or state transition proving the platform is ready |
| **`NOT_SEND_EMAIL`** | Local-dev flag that logs (rather than sends) email, making activation codes observable |
| **`LISTEN/NOTIFY`** | PostgreSQL pub/sub mechanism backing the `simplelogin_sync_events` internal event bus |
| **ActivationCode** | 30-character single-use token minted at registration and consumed on activation |
| **Canonical vs non-canonical** | "Canonical" = value observed in the default documented environment; "non-canonical" = value produced by a sandbox/image deviation, explicitly labeled |
| **Coverage pass** | Rule-4 final sweep mapping every named Q1–Q4 item to its answering section and observed signal |
| **Run-First** | Methodology requiring claims to come from actually building/running/observing, not from reading alone |

---

_This Project Guide was generated from Blitzy's autonomous validation logs and an independent assessment of the committed deliverable. All hour figures (64 completed / 5 remaining / 69 total) and the 92.8% completion figure are consistent across Sections 1.2, 2.1, 2.2, 2.3, 7, and 8._