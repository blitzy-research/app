# Blitzy Project Guide — SimpleLogin Bounce-Handling Security Investigation

**Source branch:** `app_2cd6ee777f8c` · **Working branch:** `blitzy-39c4b256-751b-400d-8231-ba7992fdce8e` · **HEAD:** `c33d7534`
**Deliverable:** `blitzy/documentation/app_2cd6ee777f8c.md` (798 lines, 76 KB) · **Repository write footprint:** 1 file added, 0 source files modified.

> Brand color legend — Completed/AI work: **Dark Blue `#5B39F3`** · Remaining/Not-completed: **White `#FFFFFF`** · Headings/Accents: **Violet-Black `#B23AF2`** · Highlight/Soft accent: **Mint `#A8FDD9`**.

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a code-grounded security investigation of SimpleLogin's inbound bounce-email subsystem. The objective: determine empirically whether the older, unsigned bounce-address format (`bounce+{id}+@domain`) is an externally reachable enumeration oracle that leaks internal `EmailLog` state through observable SMTP responses. The deliverable is a single comprehensive Markdown report answering six objectives (O1–O6) with byte-for-byte runtime evidence captured live in the prescribed Docker container, exact wire-response tables, code locators, and rationale. The target audience is SimpleLogin's security engineers and maintainers. Scope is strictly read-only — zero source changes; one documentation file added. Business impact: surfaces an unauthenticated information-disclosure risk and frames it against BATV industry practice for bounce-address validation.

### 1.2 Completion Status

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#FFFFFF','pieStroke':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#A8FDD9','pieLegendTextColor':'#B23AF2'}}}%%
pie showData title Completion Status (91.4% Complete)
    "Completed Work (AI)" : 37
    "Remaining Work" : 3.5
```

| Metric | Value |
|---|---|
| Total Project Hours | 40.5 |
| Completed Hours (AI + Manual) | 37 (37 AI + 0 Manual) |
| Remaining Hours | 3.5 |
| Percent Complete | 91.4% |

> Completion is computed on AAP-scoped hours only: `37 / (37 + 3.5) = 37 / 40.5 = 91.4%`. The figure is intentionally capped below 100% because the remaining work is human acceptance/sign-off of a security deliverable, which cannot be self-performed by the autonomous agent.

### 1.3 Key Accomplishments

- ✅ Sole in-scope deliverable authored and committed: `blitzy/documentation/app_2cd6ee777f8c.md` (798 lines).
- ✅ All six investigation objectives (O1–O6) answered, each with rationale and code locators.
- ✅ Security verdict established empirically: the unsigned `bounce+{id}+@domain` format **is** an externally reachable, multi-state enumeration/information-disclosure oracle.
- ✅ Root cause localized to the OR-coupling in `handle()` at `email_handler.py:L2057-2061`, where unsigned-prefix acceptance sits on equal footing with the signed-VERP check (objective O3).
- ✅ Seven distinct SMTP wire responses captured **byte-for-byte LIVE** in the prescribed Docker container (Python 3.10 + PostgreSQL 15 + Redis).
- ✅ Every code locator (`path:Lxxx`) and every SMTP wire string verified against current source (programmatic check: 82 exact matches, 0 mismatches), including the verbatim source typo `E510 "so such user"`.
- ✅ Read-only mandate proven at the git level: `git diff 2cd6ee77..HEAD --name-status` returns exactly one **A**dded file; zero source files touched; working tree clean.
- ✅ Transient probe scripts created for evidence capture, then deleted — none committed.
- ✅ Industry framing (BATV / backscatter / DSN null-sender) researched and applied to characterize the unsigned path as a pre-BATV anti-pattern.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|---|---|---|---|
| Human security-engineer review of the investigation verdict not yet performed | Findings unvalidated by a human SME before stakeholder distribution | Security Engineer (SME) | 2h |
| Stakeholder acceptance / sign-off pending | Deliverable not formally accepted into the security record | Project Stakeholder | 1h |
| Documented oracle vulnerability remains open by design (remediation explicitly out-of-scope per AAP §0.3.2) | Real, unauthenticated enumeration oracle persists in the product until a separate remediation initiative is scheduled | Security/Eng leadership | Out-of-scope (future initiative) |

> Note: There are **no compilation, test, or accuracy defects** outstanding in the deliverable itself. The "unresolved issues" above are process gates (human review) plus the inherent finding the report documents — not deficiencies in the autonomous work.

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|---|---|---|---|---|
| Source repository (`app_2cd6ee777f8c`) | Read/Write (git) | None — full access; deliverable committed | Resolved | Blitzy Agent |
| Docker container `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` | Pull/run | None — container available; build + runtime succeeded | Resolved | Blitzy Agent |
| PostgreSQL 15 / Redis (in-container) | Service credentials | None — services available inside container | Resolved | Blitzy Agent |

> No access issues identified that block build, validation, or delivery. The only host-environment limitation (host lacks `psycopg2`) is expected: DB-backed reproduction is intended to run inside the provided container, not on an arbitrary host.

### 1.6 Recommended Next Steps

1. **[High]** Assign a security engineer to review the investigation: validate the verdict, confirm the OR-coupling localization at `email_handler.py:L2057-2063`, and spot-check the 7-row evidence table and code locators (≈2h).
2. **[Medium]** Route the deliverable to the project stakeholder for formal acceptance and sign-off into the security record (≈1h).
3. **[Low]** Render the Markdown in the target documentation viewer to confirm the 7-row table, the single Mermaid flowchart, and all 32 code fences display correctly (≈0.5h).
4. **[Medium]** Open a **separate** remediation-triage initiative for the documented oracle (enforce signatures / collapse OR-coupling / broaden SPF black-holing / add bounce rate-limiting). Explicitly out-of-scope for this documentation task per AAP §0.3.2.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

Each completed component traces to a specific Agent Action Plan (AAP) objective or a required path-to-production activity. Hours reflect engineering effort actually invested (static analysis, probe construction, live runtime reproduction, authoring, and validation).

| Component | Hours | Description |
|---|---|---|
| O1 — Two-format routing analysis | 6.0 | Traced `handle()` routing across transactional/forward/reply branches; documented the unsigned plaintext path (`parse_id_from_bounce`) vs the HMAC-signed VERP path (`generate_verp_email` / `get_verp_info_from_email`); fixed exact branch conditions and check ordering. |
| O2 — External-attacker oracle feasibility | 3.0 | Demonstrated empirically that an external sender's unsigned `bounce+N+@domain` reaches `EmailLog.get()` with an attacker-chosen integer and yields distinguishable responses. |
| O3 — Security-boundary localization | 3.0 | Localized the crux to the OR-coupling at `email_handler.py:L2057-2061` where unsigned acceptance is equal-footing with signed-VERP; the analytical core of the report. |
| O4 — Exact SMTP-response capture | 7.0 | Built transient aiosmtpd `Envelope()` probes, seeded `EmailLog` rows, and captured seven literal wire strings; tabulated each to its `app/email/status.py` E-code and originating return statement. |
| O5 — Bounce-detection criteria & spoofability | 3.0 | Showed `is_bounce()` checks only `mail_from == "<>"` and `Content-Type: multipart/report` (both attacker-controllable); contrasted pass vs fail routing. |
| O6 — Oracle depth + SPF gate analysis | 3.0 | Enumerated distinct codes (E512/E510/E211/E212/E213) proving the oracle leaks more than existence; analyzed the SPF black-holing gate as a partial mitigation. |
| R9 — Web research & industry framing | 2.0 | Researched BATV, backscatter, and DSN null-sender semantics to frame the signed format as BATV-style and the unsigned path as a pre-BATV anti-pattern. |
| ENV — Docker build & live runtime reproduction | 4.0 | Built/ran SimpleLogin in the prescribed container (Python 3.10 + Postgres 15 + Redis), applied the DKIM PKCS#1 fix, and executed the DB-backed probe end-to-end. |
| DOC — Deliverable authoring & code citations | 3.0 | Authored the 798-line report (sections a–g, appendices A–C), wired ~50 `path:Lxxx` citations, evidence tables, and the probe matrix. |
| VAL — Autonomous validation & re-verification | 3.0 | Re-verified every locator and wire string against source, re-ran both empirical tiers live, and confirmed read-only compliance at the git level. |
| **Total** | **37.0** | **Sum of completed AAP-scoped components.** |

### 2.2 Remaining Work Detail

All remaining work is human acceptance/verification of the security deliverable. Each item traces to a path-to-production gate that the autonomous agent cannot self-perform.

| Category | Hours | Priority |
|---|---|---|
| SME security review of investigation findings (validate verdict, confirm OR-coupling localization, spot-check 7-row table + locators) | 2.0 | High |
| Stakeholder review & acceptance / sign-off into the security record | 1.0 | Medium |
| Markdown rendering & formatting verification in target viewer (7-row table, Mermaid flowchart, 32 code fences) | 0.5 | Low |
| **Total** | **3.5** | **—** |

**Out-of-scope follow-up (NOT counted in the 3.5h):** FU-1 — Remediation triage of the documented oracle (enforce signatures on the legacy format, collapse the OR-coupling, broaden SPF black-holing, add bounce rate-limiting). This is a separate future initiative explicitly excluded by AAP §0.3.2 (estimated independently at 4–8 engineering hours of design plus implementation, to be scoped by the owning team). It is deliberately excluded from the project total so the completion percentage reflects only this documentation mission.

### 2.3 Hours Reconciliation

- Section 2.1 completed total: **37.0h**
- Section 2.2 remaining total: **3.5h**
- Cross-check: `37.0 + 3.5 = 40.5` = Total Project Hours in Section 1.2. ✅
- Completion: `37 / 40.5 = 0.91358… → 91.4%` (matches Section 1.2). ✅

---

## 3. Test Results

All tests below originate from Blitzy's autonomous validation logs for this project. The in-scope subset (bounce/VERP) is the relevant portion of the wider regression suite; the full-suite row is included for context only.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---|---|---|---|---|---|---|
| Bounce handler harness (in-scope) | pytest | 23 | 23 | 0 | Not measured | `tests/test_email_handler.py` — primary bounce-routing harness |
| Bounce/VERP format units (in-scope) | pytest | 14 | 14 | 0 | Not measured | `tests/test_email_utils.py` subset: `parse_id_from_bounce`, VERP round-trip, `should_ignore_bounce`, `get_mailbox_bounce_info`, disable-bounce, header/orig-message bounce |
| Live DB-backed probe (in-scope) | pytest | 1 | 1 | 0 | N/A | Appendix B probe; drives `handle()` / `_handle()`; reproduces all 7 SMTP wire strings byte-for-byte |
| SPF-gate anchor tests (in-scope) | pytest | 3 | 3 | 0 | N/A | `test_prevent_5xx_from_spf` (E216), `test_preserve_5xx_with_valid_spf` (E512), `test_preserve_5xx_with_no_header` (E512) |
| Full regression baseline (context only) | pytest | 639 | 621 | 18 | Not measured | All 18 failures are OUT-OF-SCOPE subsystems (PGP/GPG ×2, billing/payments ×2, events/webhooks/subscriptions ×11, dashboard/alias CRUD ×2, google-re2 regex infra ×1) |

**In-scope pass rate:** 41 / 41 = **100%** (23 + 14 + 1 + 3). Zero failures anywhere in the bounce subsystem.

**Coverage note:** Line/branch coverage was **not measured** in the autonomous logs; rather than fabricate a number, it is reported as "Not measured." Functionally, the probe matrix exercises every reachable terminal of the unsigned bounce path: E512 (no log), E510 (inactive user), E211 (forward phase), E212 (reply phase), E213 (valid id / non-bounce), E515 (tampered VERP), and E216 (SPF black-hole).

**Integrity note (Rule 3):** Every test row above is sourced from Blitzy's autonomous test-execution logs for this project. The 18 full-suite failures are pre-existing, unrelated to the deliverable, explicitly out-of-scope per AAP §0.1.2/§0.3.2, and physically impossible to remediate within a documentation-only mandate (a Markdown file cannot change test outcomes, and fixing them would require editing out-of-scope source — the #1 forbidden action). They are documented, not fixed.

---

## 4. Runtime Validation & UI Verification

Runtime behavior was validated LIVE inside the prescribed Docker container (Python 3.10.18 + PostgreSQL 15 + Redis 4.6.0). The seven SMTP/LMTP wire strings below were reproduced **byte-for-byte** against the deliverable's Appendix A.2 transcript.

**SMTP/LMTP wire-response reproduction (unsigned `bounce+N+@domain` probe matrix):**

- ✅ Operational — Row 1 · Non-existent `EmailLog` id → `550 SL E512 No such email log`
- ✅ Operational — Row 2 · Valid id, active user, forward-phase → `250 SL E211 Bounce Forward phase handled`
- ✅ Operational — Row 3 · Valid id, active user, reply-phase → `250 SL E212 Bounce Reply phase handled`
- ✅ Operational — Row 4 · Valid id, **inactive** user → `550 SL E510 so such user` *(verbatim source typo preserved)*
- ✅ Operational — Row 5 · Valid id, message not a bounce → `250 SL E213 Unknown email ignored` (`handle()` raises `VERPForward`, mapped in `handle_DATA`)
- ✅ Operational — Row 6 · Tampered signed-VERP address → `550 SL E515 Email not exist` (HMAC mismatch diverts away from `EmailLog.get()` — the decisive contrast vs the unsigned path)
- ✅ Operational — Row 7 · Any 5xx + `R_SPF_FAIL` → `250 SL E216 Handled spf policy` (SPF gate fires; engine logs at `email_handler.py:L2362`)

**Format-layer (Tier-1) reproduction:**

- ✅ Operational — `parse_id_from_bounce("bounce+12345+@domain")` → `12345`; also `1` and `99999999999999`; unsigned non-bounce input → `None`.
- ✅ Operational — VERP round-trip: `generate_verp_email` → `get_verp_info_from_email` → `(bounce_forward, 12345)`; tampered payload → `None`.
- ✅ Operational — `is_bounce()` truth table: `(<>, multipart/report)` → True; non-null sender → False; wrong content-type → False.

**Application context / services:**

- ✅ Operational — Flask app context via `create_light_app()`; PostgreSQL (77 tables, alembic head `32f25cbf12f6`) and Redis reachable; all key dependencies present.

**Observed run-to-run variance (expected, documented):**

- ⚠ Partial (by design) — The VERP payload `minutes` field is time-dependent and increments monotonically with wall-clock across runs (A.1 → A.2 → re-run), exactly as the deliverable's honesty notes predict. This proves the timestamp is computed live and is **not** an inconsistency.

**UI Verification:**

- ✅ N/A (no UI surface) — The subject under analysis is a backend SMTP/LMTP email handler, and the deliverable is a Markdown document. There is no web UI, screen, or visual component in scope; consequently no browser/Lighthouse/screenshot verification applies. The "interface" validated here is the SMTP wire protocol, covered exhaustively above.

---

## 5. Compliance & Quality Review

The matrix cross-maps every AAP deliverable and governing-rule requirement to its verification status. Progress indicators: ✅ Pass · ⚠ Partial/Pending · ❌ Fail.

| Deliverable / Benchmark | Status | Progress | Notes |
|---|---|---|---|
| O1 — Two-format routing analysis | ✅ Pass | 100% | Dedicated section; `handle()` branches + helpers documented with locators |
| O2 — External-attacker oracle feasibility | ✅ Pass | 100% | Empirically demonstrated via live probes |
| O3 — Security-boundary localization (OR-coupling) | ✅ Pass | 100% | Crux localized to `email_handler.py:L2057-2063` |
| O4 — Exact SMTP-response capture | ✅ Pass | 100% | 7 wire strings tied to `status.py` E-codes + return statements |
| O5 — Bounce-detection criteria & spoofability | ✅ Pass | 100% | `is_bounce()` two-check spoofability shown |
| O6 — Oracle depth + SPF gate | ✅ Pass | 100% | Multi-state leak + partial SPF mitigation analyzed |
| Rule — Single answer doc named `<source_branch>.md` | ✅ Pass | 100% | `app_2cd6ee777f8c.md` |
| Rule — Placed under `blitzy/documentation/` | ✅ Pass | 100% | Exact path confirmed |
| Rule — Build & run the code (empirical) | ✅ Pass | 100% | Live reproduction in prescribed container |
| Rule — Code-as-truth with locators | ✅ Pass | 100% | ~50 `path:Lxxx` citations; 82 wire-string matches verified |
| Rule — Provide rationale/thinking | ✅ Pass | 100% | Rationale accompanies each objective |
| Rule — Do NOT modify existing source files | ✅ Pass | 100% | `git diff 2cd6ee77..HEAD --name-status` = 1 Added file only |
| Rule — Do NOT add other code to source repo | ✅ Pass | 100% | Probe scripts transient, then deleted; none committed |
| Directive — Evidence over theory | ✅ Pass | 100% | Every behavioral claim backed by reproduced output |
| Directive — Clean up transient test scripts | ✅ Pass | 100% | Container/host probes removed; working tree clean |
| Quality — No placeholders/TODO/stubs in deliverable | ✅ Pass | 100% | Scanned; markdown structurally sound (32 balanced fences) |
| Process — Human SME review | ⚠ Pending | 0% | Path-to-production gate (Section 2.2, 2.0h) |
| Process — Stakeholder acceptance/sign-off | ⚠ Pending | 0% | Path-to-production gate (Section 2.2, 1.0h) |
| Process — Rendered-format verification | ⚠ Pending | 0% | Path-to-production gate (Section 2.2, 0.5h) |

**Fixes applied during autonomous validation:** code-locator drift corrected and the Appendix A.2 live transcript re-captured to fix probe↔transcript provenance (commit `c33d7534`); review findings from the prior pass addressed (commit `bfff1834`). **Outstanding items:** the three human process gates above (3.5h total) — no code/accuracy defects remain.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|---|---|---|---|---|---|
| T1 — Code-locator drift across ~50 `path:Lxxx` citations | Technical | Low | Medium | Final-Validator re-verified every locator against current source; pin review to HEAD `c33d7534` | Mitigated |
| T2 — VERP signature/timestamp run-to-run variance misread as inconsistency | Technical | Low | Low | Honesty notes in deliverable explain the time-dependent `minutes` field; reproduction confirms monotonic increment | Resolved |
| S1 — Documented vulnerability: external unauthenticated multi-state enumeration oracle on the unsigned bounce path | Security | High | High | **Open by design** — remediation explicitly out-of-scope per AAP §0.3.2; routed to HT-1 (SME) and a separate follow-up initiative (FU-1) | Open (by design) |
| S2 — Secret/credential exposure within the deliverable | Security | High (would-be) | Very Low | All secrets (e.g., `VERP_EMAIL_SECRET`) REDACTED in the report; verified during validation | Resolved |
| O1 — Reproducibility depends on the provided Docker container | Operational | Low | Low | Appendix B ships a ready-to-run probe; exact build/run commands documented in Section 9 | Mitigated |
| O2 — No CI binds deliverable locators to source over time | Operational | Low | Medium | Accepted; recommend a future lightweight locator-lint if the report is kept long-lived | Accepted |
| I1 — Source-tree integration risk | Integration | Negligible | Very Low | Zero source/schema/dependency changes; single added Markdown file | Resolved |
| I2 — Merge/rebase line-drift shifting cited line numbers | Integration | Low | Medium | Accepted; citations pinned to HEAD; re-verify if rebased onto a newer base | Accepted |

**Dominant residual risk:** **S1** — the documented oracle is a *real* product weakness. It is the primary actionable finding and the reason HT-1 (SME review) is High priority. The investigation's job was to surface and prove it, not to fix it; remediation is a separate, owner-scoped initiative.

---

## 7. Visual Project Status

**Project hours — completed vs remaining (Total 40.5h):**

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#FFFFFF','pieStroke':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#A8FDD9','pieLegendTextColor':'#B23AF2'}}}%%
pie showData title Project Hours Breakdown (Total 40.5h)
    "Completed Work" : 37
    "Remaining Work (Human)" : 3.5
```

**Remaining work by category (3.5h total):**

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#B23AF2','pie3':'#A8FDD9','pieStroke':'#FFFFFF','pieOuterStrokeColor':'#B23AF2','pieTitleTextColor':'#B23AF2','pieLegendTextColor':'#B23AF2'}}}%%
pie showData title Remaining Work by Category (3.5h)
    "SME Security Review" : 2
    "Stakeholder Sign-off & Acceptance" : 1
    "Render/Format Verification" : 0.5
```

**Remaining-by-category bar (hours · priority):**

| Category | Hours | Priority | Bar |
|---|---|---|---|
| SME Security Review | 2.0 | High | ████████ |
| Stakeholder Sign-off & Acceptance | 1.0 | Medium | ████ |
| Render/Format Verification | 0.5 | Low | ██ |

**Brand color legend:**

| Swatch | Hex | Meaning |
|---|---|---|
| Dark Blue | `#5B39F3` | Completed / AI work |
| White | `#FFFFFF` | Remaining / Not completed |
| Violet-Black | `#B23AF2` | Headings / Accents |
| Mint | `#A8FDD9` | Highlight / Soft accent |

> Integrity (Rule 1): the "Remaining" total in this section (3.5h) equals Section 1.2 Remaining Hours and the Section 2.2 "Hours" column sum. The "Completed" total (37h) equals Section 1.2 Completed Hours and the Section 2.1 sum.

---

## 8. Summary & Recommendations

**Achievements.** The project is **91.4% complete** on an AAP-scoped basis (37 of 40.5 hours). The single in-scope deliverable — `blitzy/documentation/app_2cd6ee777f8c.md` — is authored, committed, internally consistent, and empirically validated. All six objectives (O1–O6) are answered with rationale and code locators, and the central security verdict is proven with live, byte-for-byte SMTP evidence: the older unsigned `bounce+{id}+@domain` format is an externally reachable, multi-state enumeration/information-disclosure oracle, rooted in the OR-coupling at `email_handler.py:L2057-2061`.

**Remaining gaps (3.5h, all human).** What remains is not engineering but acceptance: SME security review (2.0h), stakeholder sign-off (1.0h), and rendered-format verification (0.5h). None can be self-performed by the autonomous agent, which is precisely why completion is capped at 91.4% rather than 100%.

**Critical path to production.** SME review → stakeholder acceptance → rendered-format check → (separately) schedule remediation triage (FU-1, out-of-scope here).

**Success metrics.** In-scope tests: 41/41 passing (100%). Read-only mandate: 1 added file, 0 source modifications. Wire-string fidelity: 7/7 reproduced byte-for-byte. Locator accuracy: 82 verified matches, 0 mismatches.

**Production readiness assessment.** The deliverable is **production-ready as a documentation artifact** and safe to merge (zero source/schema/dependency impact). It should not be considered *organizationally accepted* until the 3.5h of human review/sign-off is complete. The vulnerability it documents (risk S1) is real and open by design; remediation must be tracked as a distinct initiative.

| Summary Metric | Value |
|---|---|
| AAP-scoped completion | 91.4% |
| Total / Completed / Remaining hours | 40.5 / 37 / 3.5 |
| In-scope test pass rate | 100% (41/41) |
| Source files modified | 0 |
| Dominant residual risk | S1 (documented oracle, open by design) |

---

## 9. Development Guide

This guide covers two modes: **(A) Static verification** of the deliverable on any host (no database required), and **(B) Full empirical reproduction** of the SMTP wire responses, which requires the prescribed Docker container.

### 9.1 System Prerequisites

- **Mode A (static):** `git`, `python3` (3.8+), and any Markdown viewer. No database, no Postgres, no Redis.
- **Mode B (empirical):** The provided Docker container `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`, which ships Python **3.10.18**, PostgreSQL **15**, and Redis. The application targets Python 3.10 (`Dockerfile: FROM python:3.10`; `pyproject.toml: python = "^3.10"`).
- **Note on arbitrary hosts:** A generic host (e.g., Python 3.13 without `psycopg2`) can run Mode A and the Tier-1 format functions, but **cannot** run the DB-backed probe (`ModuleNotFoundError: No module named 'psycopg2'`). Use the container for Mode B.

### 9.2 Environment Setup

```bash
# Locate the repository and confirm the working branch / HEAD
cd /tmp/blitzy/app/blitzy-39c4b256-751b-400d-8231-ba7992fdce8e_f70b77
git rev-parse --abbrev-ref HEAD     # blitzy-39c4b256-751b-400d-8231-ba7992fdce8e
git rev-parse --short HEAD          # c33d7534
```

For Mode B, the canonical in-container environment is sourced from the prepared env file; `NOT_SEND_EMAIL=true` (in `example.env`) prints outbound mail rather than sending it, enabling safe local observation.

```bash
# Inside the container, after /build.sh and the DKIM PKCS#1 fix:
cd /app
set -a; . /app/sl_env.sh; set +a
unset PYTEST_ADDOPTS
export CONFIG=/app/tests/test.env
```

### 9.3 Dependency Installation

Dependencies are managed by **Poetry** with a pinned `poetry.lock` and are pre-installed in the container's virtualenv at `/app/venv`. No installation is required inside the provided container. (On a bespoke host you would run `poetry install`, but Mode B is the supported empirical path.)

### 9.4 Build / Run & Verification — Mode A (Static, any host)

```bash
# 1) Confirm the deliverable exists and its size
wc -l blitzy/documentation/app_2cd6ee777f8c.md       # -> 798
du -h  blitzy/documentation/app_2cd6ee777f8c.md       # -> 76K

# 2) Prove the read-only mandate (exactly one ADDED file, zero source edits)
git diff 2cd6ee77..HEAD --name-status                 # -> A  blitzy/documentation/app_2cd6ee777f8c.md
git diff 2cd6ee77..HEAD --stat                        # -> 1 file changed, 798 insertions(+)

# 3) Confirm Markdown code fences are balanced (must be even)
python3 -c "print('fences:', open('blitzy/documentation/app_2cd6ee777f8c.md').read().count('\`\`\`'))"

# 4) Reproduce the Tier-1 format parse on any host (no DB needed)
python3 -c "import sys; sys.path.insert(0,'.'); \
addr='bounce+12345+@domain'; \
print('id =', int(addr[addr.find('+'):addr.rfind('+')]))"   # -> id = 12345
```

### 9.5 Build / Run & Verification — Mode B (Empirical reproduction, container)

```bash
# After /build.sh + DKIM PKCS#1 fix and the env setup in 9.2:
cd /app
set -a; . /app/sl_env.sh; set +a
unset PYTEST_ADDOPTS
export CONFIG=/app/tests/test.env

# Run the DB-backed probe (extracted verbatim from the deliverable's Appendix B)
/app/venv/bin/python -m pytest -s -p no:cacheprovider -o addopts="" \
  --timeout=60 --timeout-method=signal \
  tests/blitzy_live_probe.py::test_probe                      # -> 1 passed

# Run the in-scope bounce harness + format units
/app/venv/bin/python -m pytest -q tests/test_email_handler.py # -> 23 passed
/app/venv/bin/python -m pytest -q tests/test_email_utils.py -k "bounce or verp"  # -> 14 passed
```

Expected wire strings (each reproduced byte-for-byte): `550 SL E512 No such email log`, `250 SL E211 Bounce Forward phase handled`, `250 SL E212 Bounce Reply phase handled`, `550 SL E510 so such user`, `250 SL E213 Unknown email ignored`, `550 SL E515 Email not exist`, `250 SL E216 Handled spf policy`.

### 9.6 Example Usage (reading the deliverable)

```bash
# View the security verdict and the 7-row evidence table
sed -n '1,60p'      blitzy/documentation/app_2cd6ee777f8c.md   # intro + verdict
grep -n "SL E5\|SL E2" blitzy/documentation/app_2cd6ee777f8c.md  # locate wire-string rows
grep -n "L2057\|L2061\|L2063" blitzy/documentation/app_2cd6ee777f8c.md  # OR-coupling locators
```

### 9.7 Troubleshooting (common error cases and resolutions)

- **`DetachedInstanceError`** when seeding rows → route seeded `EmailLog` rows through `handle()` (which manages the session), not through `_handle()` directly.
- **`AttributeError: 'str' object has no attribute 'policy'`** → build the DSN message from raw RFC822 **bytes** via `email.message_from_bytes(...)`, not from a `str`.
- **DB connects to the wrong port / unexpected schema** → `DB_URI` from the sourced env (port 5432) overrides `test.env` (port 15432) because config loads dotenv with `override=False`. Unset or align `DB_URI` if you intend to use the test DB.
- **DKIM signing errors at startup** → ensure the DKIM private key is in **PKCS#1** format (the container fix converts it before the probe runs).
- **`ModuleNotFoundError: No module named 'psycopg2'`** on a generic host → expected; run Mode B inside the provided container for any DB-backed step.

---

## 10. Appendices

### Appendix A — Command Reference

| Purpose | Command |
|---|---|
| Confirm branch | `git rev-parse --abbrev-ref HEAD` |
| Confirm HEAD | `git rev-parse --short HEAD` |
| Deliverable line count | `wc -l blitzy/documentation/app_2cd6ee777f8c.md` |
| Read-only proof (name-status) | `git diff 2cd6ee77..HEAD --name-status` |
| Read-only proof (stat) | `git diff 2cd6ee77..HEAD --stat` |
| Agent commit log | `git log 2cd6ee77..HEAD --pretty=format:'%h %an %s'` |
| Tier-1 parse demo | `python3 -c "a='bounce+12345+@domain'; print(int(a[a.find('+'):a.rfind('+')]))"` |
| In-container env | `cd /app; set -a; . /app/sl_env.sh; set +a; unset PYTEST_ADDOPTS; export CONFIG=/app/tests/test.env` |
| DB-backed probe | `/app/venv/bin/python -m pytest -s -p no:cacheprovider -o addopts="" --timeout=60 --timeout-method=signal tests/blitzy_live_probe.py::test_probe` |
| Bounce harness | `/app/venv/bin/python -m pytest -q tests/test_email_handler.py` |

### Appendix B — Port Reference

| Service | Port | Notes |
|---|---|---|
| PostgreSQL (container) | 5432 | 77 tables; alembic head `32f25cbf12f6`; `DB_URI` here takes precedence (`override=False`) |
| PostgreSQL (test.env) | 15432 | Overridden by sourced `DB_URI:5432` when both are present |
| Redis | 6379 | Backs rate limiting in the inbound pipeline |
| Inbound SMTP/LMTP | configurable | `Controller(MailHandler(), hostname="0.0.0.0", port=port)`; probes call `handle()`/`_handle()` directly, so no listening socket is required |

### Appendix C — Key File Locations

| File | Role |
|---|---|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The deliverable** (798 lines) |
| `email_handler.py` | `handle()` routing, `handle_bounce()`, `is_bounce()`, `handle_DATA`/`_handle` + SPF gate |
| `app/email_utils.py` | `parse_id_from_bounce()`, `generate_verp_email()`, `get_verp_info_from_email()`, `should_ignore_bounce()` |
| `app/email/status.py` | Canonical E-code → SMTP wire-string mapping |
| `app/config.py` | `BOUNCE_PREFIX/SUFFIX`, `VERP_*` constants |
| `app/models.py` | `EmailLog`, `Bounce`, `User.is_active()` |
| `app/handler/spamd_result.py` | `SpamdResult` / SPF verdict consumed by the SPF gate |
| `tests/test_email_handler.py` | Reference harness pattern for probes |

### Appendix D — Technology Versions

| Component | Version |
|---|---|
| Python (container / target) | 3.10.18 |
| Python (generic host, static mode) | 3.13.7 |
| aiosmtpd | 1.4.2 |
| SQLAlchemy | 1.3.24 |
| Flask | 1.1.2 |
| redis (client) | 4.6.0 |
| cryptography | 37.0.1 |
| dkimpy | 1.0.5 |
| pyspf | 2.0.14 |
| psycopg2-binary | 2.9.3 |
| Flask-Limiter | 1.4 |
| PostgreSQL | 15 |

### Appendix E — Environment Variable Reference

| Variable | Purpose |
|---|---|
| `BOUNCE_PREFIX` | Default `bounce+`; matched by the unsigned forward-bounce branch |
| `BOUNCE_SUFFIX` | Default `+@{EMAIL_DOMAIN}`; matched by the unsigned forward-bounce branch |
| `VERP_PREFIX` | Prefix for the signed-VERP address format |
| `VERP_EMAIL_SECRET` | HMAC secret for VERP signing (**REDACTED** in the deliverable) |
| `VERP_MESSAGE_LIFETIME` | VERP validity window (5 days) |
| `NOT_SEND_EMAIL` | `true` prints outbound mail instead of sending — safe local observation |
| `DB_URI` | Postgres connection; sourced value (5432) overrides `test.env` (15432) |
| `CONFIG` | Points the app at the config/env file (e.g., `/app/tests/test.env`) |

### Appendix F — Developer Tools Guide

- **pytest** — primary test runner; use `-p no:cacheprovider -o addopts=""` to neutralize repo-level addopts during ad-hoc probe runs.
- **git diff `--name-status` / `--stat`** — the authoritative read-only-mandate check.
- **py_compile / import smoke test** — confirm referenced modules load under the canonical env.
- **Markdown viewer** — render the deliverable to verify the 7-row table, the single Mermaid flowchart, and all 32 code fences.

### Appendix G — Glossary

| Term | Definition |
|---|---|
| **BATV** | Bounce Address Tag Validation — signing the return-path with a token + timestamp so unsigned bounces can be rejected |
| **VERP** | Variable Envelope Return Path — per-message return address; here HMAC-signed in the newer format |
| **DSN** | Delivery Status Notification — a bounce message; uses null sender `MAIL FROM:<>` |
| **HMAC** | Hash-based Message Authentication Code — integrity/authenticity check used by VERP signing |
| **Oracle** | An interface whose distinguishable responses leak internal state to an observer |
| **Backscatter** | Misdirected bounce messages, often from spoofed senders |
| **SPF** | Sender Policy Framework — sender-authorization check; drives the 5xx black-holing gate |
| **EmailLog** | ORM record keyed by the integer an attacker supplies in `bounce+{id}+@domain` |
| **LMTP** | Local Mail Transfer Protocol — the wire protocol whose reply strings are the oracle's output |
| **OR-coupling** | The disjunction in `handle()` accepting an unsigned match on equal footing with the signed-VERP check |

---

*End of Blitzy Project Guide. Canonical metrics: Total 40.5h · Completed 37h · Remaining 3.5h · 91.4% complete (AAP-scoped). Repository footprint: 1 file added, 0 source files modified.*