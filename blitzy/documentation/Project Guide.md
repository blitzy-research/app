# Blitzy Project Guide
## SimpleLogin — First-Time Initialization Runtime Behavior Q&A (Q1–Q5)

> **Document type:** Documentation-only, read-only investigation deliverable
> **Governing rule set:** SWE-AtlasQnA-Repo
> **Branch:** `blitzy-c98c3202-123e-448c-9193-0c9eef90a885` · **Base:** `app_2cd6ee777f8c` · **HEAD:** `ee0b37dd`
>
> **Color legend (Blitzy brand):** <span style="color:#5B39F3">■</span> **Completed / AI Work — Dark Blue `#5B39F3`**  ·  <span style="color:#B23AF2">■</span> Headings/Accents — Violet-Black `#B23AF2`  ·  □ **Remaining / Not Completed — White `#FFFFFF`**  ·  <span style="color:#A8FDD9">■</span> Highlight — Mint `#A8FDD9`

---

## 1. Executive Summary

### 1.1 Project Overview

This project produces one comprehensive Markdown document that empirically answers five runtime-behavior questions about **SimpleLogin** — an open-source email-privacy/aliasing Flask application — during first-time initialization. The target audience is an operator standing the system up from scratch (web server, email handler, database migrations) who needs *observed* answers, not code-reading inferences. Technical scope spans database migrations (Alembic), the Gunicorn web server, the aiosmtpd email handler, and the REST API for registration/login and user info. Business impact: authoritative, reproducible operator documentation that de-risks first deployment and surfaces pre-existing application defects. The work is strictly read-only on source; the sole deliverable is `blitzy/documentation/app_2cd6ee777f8c.md`.

### 1.2 Completion Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieOuterStrokeWidth':'2px','pieTitleTextSize':'16px','pieSectionTextColor':'#111111','pieLegendTextColor':'#111111'}}}%%
pie showData title Project Completion — 85.9% Complete (55 of 64 hrs)
    "Completed Work (Dark Blue #5B39F3)" : 55
    "Remaining Work (White #FFFFFF)" : 9
```

| Metric | Hours |
|---|---|
| **Total Hours** | **64** |
| **Completed Hours (AI + Manual)** | **55** (AI: 55 · Manual: 0) |
| **Remaining Hours** | **9** |
| **Percent Complete** | **85.9%** |

> Completion is computed with the PA1 AAP-scoped methodology: `Completed ÷ (Completed + Remaining) = 55 ÷ 64 = 85.9%`. The AAP-specified deliverable is fully delivered and runtime-validated; the remaining 9 hours are human path-to-production gates (review, independent reproduction, anomaly triage, sign-off).

### 1.3 Key Accomplishments

- ✅ **Single validated deliverable authored** at the mandated path `blitzy/documentation/app_2cd6ee777f8c.md` (978 lines, 69 KB, 8,603 words).
- ✅ **Q1 answered & runtime-verified:** 77 tables total (76 app + `alembic_version`); last-created table `user_audit_log`; 255 migration steps; stable on a second fresh database.
- ✅ **Q2 answered & runtime-verified:** readiness line `Listening at: http://0.0.0.0:7777 (<master-pid>)`; ≈0.2 ms socket-bind / ≈623 ms request-serving latency, stable across 4 runs; Werkzeug-disabled caveat; two failure modes.
- ✅ **Q3 answered & runtime-verified:** INFO `Listen for port 25025` + DEBUG `Start mail controller 0.0.0.0 25025`; occupied-port failure; default-port (20381) contrast.
- ✅ **Q4 answered & runtime-verified:** register → 200, login-before-activation → 422 `{"error":"Account not activated"}`; DB `activated=f`/`notification=t`; all 7 sibling login-error branches.
- ✅ **Q5 answered & runtime-verified:** limit `3` before, `10` after config change + restart for **both** a pre-existing and a new user (config-at-import semantics); unset fallback `5`.
- ✅ **Read-only guarantee upheld:** no source file modified; `git diff --name-status` against base shows a single added `.md`; working tree clean.
- ✅ **Grounding & methodology:** every value carries exact command + unedited output + `file:line` reference; 8 groundings independently spot-checked accurate; framework behavior corroborated via web research; 26-row coverage checklist; "no Inferred values".
- ✅ **QA-hardened** across 5 refinement cycles (code review + QA Reports 7 & 8, 13+ findings) and passed final independent runtime re-validation with **zero fixes required**.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|---|---|---|---|
| *(None blocking the deliverable)* | The AAP deliverable is complete, accurate, well-formed, and committed; final validation required zero fixes. | — | — |
| Disclosed pre-existing **application** findings (not deliverable defects) — see §6 | Do not block this documentation task; warrant a human triage decision (remediation is out of read-only scope) | App maintainers | Post-review |

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|---|---|---|---|---|
| PyPI / OSV advisory DB | Network (outbound) | `pip-audit` is not installed in the pinned venv and requires network access unavailable in the offline container; a live CVE scan could not be run through the canonical environment | Worked around — advisories grounded via observed installed versions cross-referenced to public NVD/OSV/GHSA records (§A.7 of deliverable) | Reviewer (optional live re-scan) |

> No access issues prevented completing the AAP deliverable. All five questions were fully exercised and validated inside the mandated image.

### 1.6 Recommended Next Steps

1. **[High]** SME technical review of the deliverable — verify Q1–Q5 answers, `file:line` groundings, coverage checklist, and Observed/Inferred labels.
2. **[High]** Independent runtime reproduction spot-check inside the mandated image — re-verify the headline values (77/`user_audit_log`, readiness line + timing magnitude, 422 + DB columns, before-3/after-10).
3. **[Medium]** Disclosed-anomaly triage & ticketing decision — decide whether to file the disclosed application findings (Issues 2/7/8/11/12 + DKIM prerequisite) as separate work items.
4. **[Medium]** Stakeholder sign-off & merge — approve the doc-only PR and merge to `main`.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

All completed work is AI (autonomous) and traces to a specific AAP requirement or prerequisite.

| Component | Hours | Description |
|---|---:|---|
| Environment stand-up & troubleshooting | 7 | Launch mandated image; `bash /build.sh` → `poetry install` (177 locked deps); re2 binding workaround (pyre2 source-build fails → prebuilt wheel + google-re2 + one-line `sed`); PostgreSQL 15.13 + Redis 7.0.15; ephemeral `.env` from `example.env`; DKIM PKCS#8→PKCS#1 prerequisite |
| Q1 — Migration table-count investigation | 4 | `alembic upgrade head` on two fresh empty DBs; `information_schema.tables` query → 77 tables / `user_audit_log`; 255-file created-vs-altered analysis; stability confirmation |
| Q2 — Web-server readiness + timing | 6 | Canonical Gunicorn launch (`Dockerfile:47`); identify `Listening at:` line; millisecond timing across 4 runs (socket-bind ≈0.2 ms vs request-serving ≈623 ms); Werkzeug-disabled caveat; failure modes A (missing `DB_URI`→exit 3) & B (Redis down→500) |
| Q3 — Email handler custom port | 3 | `python email_handler.py --port 25025`; capture INFO/DEBUG confirmation lines; live SMTP transaction (220/250/221); occupied-port errno 98/exit 1; default-port 20381 contrast |
| Q4 — Register/login-before-activation | 5 | `curl` register (200) + login-before-activation (422); exact JSON body; direct `psql` query for `activated`/`notification`; 7 sibling login-error branches; malformed-payload 500 disclosure |
| Q5 — Dynamic alias limit + restart | 6 | `/api/user_info` before (3) / after (10); config change + full server restart (new master PID proof); both users reflect new limit; real register→activate→login API-key flow; unset-fallback (5) |
| Framework research (web search) | 2 | Confirm Gunicorn arbiter `Listening at:` convention and aiosmtpd threaded `Controller` readiness/`ready_timeout` behavior to interpret observed logs authoritatively |
| Anomaly discovery & `file:line` grounding | 4 | Surface, reproduce, and ground 5 pre-existing application anomalies (Issues 2/7/8/11/12) + DKIM prerequisite; disclose (not fix) per read-only scope |
| Answer-document authoring | 9 | Compose the 978-line grounded Markdown: direct-answer-first, exact commands, complete unedited output, `file:line` rationale, sibling/edge coverage, Observed/Inferred labels |
| Coverage pass & Observed/Inferred discipline | 2 | 26-row coverage checklist mapping every named item (email, password, ports, endpoint, columns, limit values); "no Inferred values" verification |
| Cleanup & read-only verification | 1 | Drop throwaway DBs, flush Redis, remove temp scripts/workspaces, destroy container; confirm delivered repo byte-for-byte unchanged (`git status`) |
| QA refinement across 5 cycles | 6 | Code-review fixes + QA Report 7 (13 findings) + QA Report 8 accuracy findings; env-preamble correction to mandated image; grounding corrections |
| **Total Completed** | **55** | |

### 2.2 Remaining Work Detail

All remaining work is human path-to-production activity. Each traces to a standard acceptance gate for a technical deliverable. (Remediation of disclosed application anomalies is **out of scope** for this read-only task and is deliberately excluded from these hours.)

| Category | Hours | Priority |
|---|---:|---|
| SME technical review of the deliverable | 3 | High |
| Independent runtime reproduction spot-check (mandated image) | 3 | High |
| Disclosed-anomaly triage & ticketing decision | 2 | Medium |
| Stakeholder sign-off & merge doc-only branch to `main` | 1 | Medium |
| **Total Remaining** | **9** | |

> **Cross-section integrity:** Section 2.1 (55) + Section 2.2 (9) = **64** = Total Project Hours (Section 1.2). Section 2.2 total (9) = Section 1.2 Remaining (9) = Section 7 pie "Remaining Work" (9).

### 2.3 Hours Calculation Summary

- **Completed:** 55 h — all 13 AAP-scoped requirements delivered & runtime-validated (Environment 7; Q1 4; Q2 6; Q3 3; Q4 5; Q5 6; Research 2; Anomalies 4; Authoring 9; Coverage 2; Cleanup 1; QA refinement 6).
- **Remaining:** 9 h — human acceptance gates (Review 3; Reproduction 3; Triage decision 2; Sign-off/merge 1).
- **Total:** 64 h.  **Completion = 55 ÷ 64 = 85.9%.**

---

## 3. Test Results

For this documentation task, the "tests" are **runtime verifications** of each question, all executed by Blitzy's autonomous validation systems by re-driving each canonical code path inside the mandated image and diffing against the document's claimed values. All originate from Blitzy's autonomous validation logs.

| Test Category | Framework / Tooling | Total Checks | Passed | Failed | Coverage % | Notes |
|---|---|---:|---:|---:|---:|---|
| Q1 — Migration verification | Alembic + `psql` (`information_schema`) | 4 | 4 | 0 | 100% | 77 tables; last = `user_audit_log`; 255 steps; identical on 2nd fresh DB |
| Q2 — Web-server readiness/timing | Gunicorn 20.0.4 + timestamp stamper + HTTP probe | 5 | 5 | 0 | 100% | `Listening at:` line; ≈0.2 ms / ≈623 ms across 4 runs; 2 failure modes |
| Q3 — Email-handler port | aiosmtpd 1.4.2 + live SMTP dialogue | 5 | 5 | 0 | 100% | INFO+DEBUG on 25025; occupied-port errno 98/exit 1; default 20381 |
| Q4 — Register/login/DB state | `curl` + `psql` | 6 | 6 | 0 | 100% | 200 → 422; `activated=f`/`notification=t`; 7 sibling branches; malformed-500 disclosure |
| Q5 — Dynamic alias limit | `curl` + config change + full restart | 5 | 5 | 0 | 100% | before 3 / after 10 for both users; unset → 5 |
| **Runtime verification subtotal** | — | **25** | **25** | **0** | **100%** | All Q1–Q5 values re-verified exact |
| *(Reference) SimpleLogin unit suite* | pytest | 639 | 633 | 6 | — | From setup log; 6 environment-specific exclusions, outside this documentation task's scope; no in-scope change required |

> **Integrity note (Rule 3):** every entry above is sourced from Blitzy's autonomous validation/setup logs for this project. The 25 runtime-verification checks are the task's actual tests (all pass); the unit-suite row is included only as referenced environment context.

---

## 4. Runtime Validation & UI Verification

Runtime health of each canonical code path exercised during the investigation (inside the disposable mandated-image container):

- ✅ **Operational** — Database migrations: `alembic upgrade head` completes (exit 0, 255 steps) → 77 tables materialized.
- ✅ **Operational** — Web server: Gunicorn binds and logs `Listening at: http://0.0.0.0:7777`; HTTP probe confirms request-serving readiness (~623 ms after first log).
- ✅ **Operational** — Email handler: aiosmtpd `Controller` listens on custom port 25025; live SMTP `220`/`250`/`221` dialogue confirmed.
- ✅ **Operational** — REST API: `register` → 200, `activate` → success, `login` → 422 before activation / api_key after, `user_info` → JSON with `max_alias_free_plan`.
- ✅ **Operational** — Backing services: PostgreSQL 15.13 and Redis 7.0.15 healthy throughout.
- ⚠ **Partial** — DKIM signing on registration requires a one-time PKCS#8→PKCS#1 key conversion prerequisite (§A.6); without it, `register` returns 500 (disclosed, read-only-safe workaround provided).
- ⚠ **Partial** — Malformed-input handling: wrong-type/missing/null email payloads produce unhandled HTTP 500s (disclosed Issue 7).
- ▫ **Not Applicable** — UI verification: the five questions target API/SMTP/CLI/DB paths only; the dashboard UI is explicitly out of scope for this task, so no UI verification (screenshots/Lighthouse) applies.

---

## 5. Compliance & Quality Review

Cross-mapping the AAP deliverable and governing SWE-AtlasQnA-Repo rules to Blitzy quality benchmarks:

| Benchmark / Rule | Status | Progress | Notes |
|---|---|---|---|
| Read-only guarantee — no source file modified | ✅ Pass | 100% | `git diff --name-status 2cd6ee77..HEAD` = single `A blitzy/documentation/app_2cd6ee777f8c.md`; working tree clean |
| Deliverable at mandated branch-named path | ✅ Pass | 100% | `blitzy/documentation/app_2cd6ee777f8c.md` present & committed at HEAD `ee0b37dd` |
| Observe-don't-infer (real, unedited output) | ✅ Pass | 100% | Every value carries exact command + complete output; document states "no Inferred values" |
| Canonical entry points only | ✅ Pass | 100% | Gunicorn (`Dockerfile:47`), `email_handler.py` CLI, real REST endpoints, `alembic upgrade head` |
| `file:line` grounding of factual claims | ✅ Pass | 100% | 8 groundings independently spot-checked against source — all accurate |
| Coverage of every named item | ✅ Pass | 100% | 26-row coverage checklist (email, password, ports 25025/20381, `/api/user_info`, `max_alias_free_plan`, `activated`/`notification`, limits 3/5/10) |
| Magnitude/timing stability (≥2 runs) | ✅ Pass | 100% | Q2 timing across 4 runs; Q1 counts on a 2nd fresh DB |
| Every-condition coverage (not just happy path) | ✅ Pass | 100% | 7 sibling login branches; 3 malformed-payload 500s; both Q2 failure modes; Q3 occupied-port; Q5 unset-fallback |
| Web-search framework corroboration | ✅ Pass | 100% | Gunicorn arbiter + aiosmtpd Controller behavior cited in References |
| Cleanup obligation | ✅ Pass | 100% | Throwaway DBs/scripts/workspaces/container removed; delivered repo verified unchanged |
| Markdown well-formedness | ✅ Pass | 100% | Balanced code fences; internal anchor links resolve via GitHub slug rules |

**Fixes applied during autonomous validation:** 5 refinement cycles — initial draft → code-review runtime-evidence/grounding fixes → env-preamble correction to the mandated image + DKIM prerequisite → QA Report 7 (13 findings) → QA Report 8 accuracy findings. Final independent runtime re-validation required **zero additional fixes**.

**Outstanding compliance items:** none for the deliverable. Disclosed pre-existing **application** findings (§6) are grounded and reported per the read-only mandate, not remediated.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|---|---|---|---|---|---|
| Instance-specific values (PIDs, timestamps, cookie signatures, API keys, exact ms) differ run-to-run; a reviewer diffing verbatim may misread as inaccuracy | Technical | Low | Medium | Values explicitly labeled "Observed"; variance explained; secrets redacted | Mitigated |
| Q2 millisecond timing is environment-sensitive (hardware/load dependent) | Technical | Low | Medium | Reported as magnitude + range across 4 runs; stated as magnitude-sensitive | Mitigated |
| Environment drift — mandated image PG 15 / Redis 7 differ from CI pins (PG 13 / Redis 6) | Technical | Low | Low | Exact observed versions pinned in §A.5; shown not to affect reported values | Mitigated |
| Pinned dependency CVEs — gunicorn 20.0.4 (CVE-2024-1135/6827), aiosmtpd 1.4.2 (CVE-2024-27305/34083) | Security | High | Medium | Disclosed (§A.7 / Issue 11) with upgrade path (gunicorn 22/23; aiosmtpd 1.4.5/6); remediation out of read-only scope | Open — disclosed |
| Authorization inconsistency — past-due scheduled-deletion account regains API-key access + PII (200) while login blocks it (400) | Security | High | Low | Grounded at `app/models.py:766-769` / `app/api/base.py`; disclosed for human triage | Open — disclosed |
| Credential logging — API key supplied as a query param logged verbatim (rejected 401 but logged) | Security | Medium | Medium | Grounded at `server.py:289`; disclosed | Open — disclosed |
| Missing response security headers + unhandled 500s on malformed input | Security | Medium | Medium | Grounded (Issues 12 & 7); disclosed | Open — disclosed |
| DKIM PKCS#8 key blocks registration (500) until converted to PKCS#1 | Operational | Medium | High (if reproducing) | §A.6 read-only-safe conversion on a copy + `DKIM_PRIVATE_KEY_PATH` override + restart | Mitigated |
| Reproduction requires the exact mandated Docker image | Operational | Low | Medium | §A.1 names exact image + full commit SHA | Mitigated |
| Backing-service dependency — PostgreSQL + Redis must be up (Redis down → every request 500) | Integration | Low | Low | §A.4 health-check step | Mitigated |
| re2 build fragility — locked `pyre2 0.3.6` source build fails | Integration | Low | Medium | §A.2 documents the observed wheel + google-re2 + `sed` mechanism | Mitigated |

> Security findings are properties of the **application/environment** surfaced by the investigation — not defects in the deliverable. They do not block the documentation deliverable but are the primary driver for the recommended anomaly-triage task (§1.6, item 3).

---

## 7. Visual Project Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieOuterStrokeWidth':'2px','pieSectionTextColor':'#111111','pieLegendTextColor':'#111111'}}}%%
pie showData title Project Hours Breakdown (Total 64 hrs · 85.9% Complete)
    "Completed Work" : 55
    "Remaining Work" : 9
```

**Remaining hours by category (Section 2.2):**

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'primaryColor':'#5B39F3','primaryTextColor':'#111111','lineColor':'#B23AF2'}}}%%
graph LR
    subgraph "Remaining Work — 9 hrs"
    A["SME technical review — 3h (High)"]
    B["Runtime reproduction spot-check — 3h (High)"]
    C["Anomaly triage decision — 2h (Medium)"]
    D["Sign-off & merge — 1h (Medium)"]
    end
```

> **Integrity (Rule 1):** pie "Remaining Work" = **9** = Section 1.2 Remaining = Section 2.2 total. Pie "Completed Work" = **55** = Section 1.2 Completed. Colors: Completed = Dark Blue `#5B39F3`; Remaining = White `#FFFFFF`.

---

## 8. Summary & Recommendations

**Achievements.** The project delivers a rigorously grounded, runtime-verified operator Q&A document for SimpleLogin first-time initialization. All five questions are answered from observed output captured inside the mandated image, with exact commands, complete unedited output, `file:line` grounding, sibling/edge coverage, and explicit Observed/Inferred labeling. The deliverable survived 5 QA refinement cycles and a final independent runtime re-validation with **zero fixes required**. The read-only guarantee is fully intact — the source repository is byte-for-byte unchanged.

**Remaining gaps.** None are AAP-source deliverables. The outstanding 9 hours are standard human acceptance gates for a technical document: SME review, an independent reproduction spot-check, a triage decision on the disclosed application findings, and sign-off/merge.

**Critical path to production.** (1) SME review → (2) independent reproduction spot-check → (3) anomaly triage decision → (4) sign-off & merge. None are blocking or high-risk; the deliverable is release-candidate quality.

**Production-readiness assessment.** The project is **85.9% complete** (55 of 64 hours). The AAP-specified scope is 100% delivered and validated; the residual is human verification and acceptance. Recommended verdict: **approve pending SME review and a reproduction spot-check.**

| Success metric | Target | Observed |
|---|---|---|
| AAP questions answered from observed runtime output | 5 / 5 | 5 / 5 ✅ |
| Runtime-verification checks passing | 100% | 25 / 25 (100%) ✅ |
| Source files modified (read-only) | 0 | 0 ✅ |
| Inferred (unobserved) values in deliverable | 0 | 0 ✅ |
| `file:line` groundings accurate (spot-check) | 100% | 8 / 8 ✅ |

---

## 9. Development Guide

> All application-runtime commands below execute **inside a disposable container** launched from the mandated image. The delivered repository checkout must **not** be executed against (read-only guarantee). Doc-level commands (git/markdown checks) are safe to run in the delivered checkout and were tested there.

### 9.1 System Prerequisites

- **Docker** (to launch the mandated image) — recommended path.
- **Mandated image:** `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (ships Python 3.10.18, PostgreSQL 15.13, Redis 7.0.15, `/app/venv` with 177 poetry-locked dependencies).
- **Host-only alternative** (if not using the image): Python 3.10, Poetry, PostgreSQL, Redis, OpenSSL, `curl`, `psql`.

### 9.2 Environment Setup

```bash
# Launch a long-lived disposable container (sleeping entrypoint, no published ports)
docker run -d --init --name sl_qafix \
  --entrypoint /bin/sleep \
  ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0 infinity

# Enter the container (all subsequent commands run inside it)
docker exec -it sl_qafix bash
```

### 9.3 Dependency Installation & Build

```bash
# Canonical bring-up: creates /app/venv, poetry install from poetry.lock,
# handles the re2 binding, starts PostgreSQL + Redis, creates DB `test`,
# generates local keys, runs `alembic upgrade head` (255 steps),
# and exports the runtime env (incl. MAX_NB_EMAIL_FREE_PLAN=3).
bash /build.sh ; echo "BUILD_EXIT=$?"     # expect BUILD_EXIT=0
```

> If a plain `poetry install` is used instead of `/build.sh`, expect the locked `pyre2 (0.3.6)` to fail its source build; the fallback is a prebuilt wheel + `google-re2` + a one-line `sed` on `app/spamassassin_utils.py:8` (used only as a drop-in `re` replacement).

### 9.4 One-Time DKIM Prerequisite (read-only-safe)

```bash
# OpenSSL 3.0.16 emits a PKCS#8 key that dkimpy cannot parse -> register would 500.
# Convert a COPY (never the tracked key) to PKCS#1 and point the app at it.
WORKDIR=$(mktemp -d)
cp /app/local_data/dkim.key "$WORKDIR/dkim.key"
openssl rsa -in "$WORKDIR/dkim.key" -traditional -out "$WORKDIR/dkim.key"
head -1 "$WORKDIR/dkim.key"                 # -----BEGIN RSA PRIVATE KEY-----  (PKCS#1)
export DKIM_PRIVATE_KEY_PATH="$WORKDIR/dkim.key"
# Restart the web server afterward so app/config.py re-reads the key at import.
```

### 9.5 Application Startup

```bash
# Web server (canonical production command, Dockerfile:47)
gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
# -> logs: [ts +0000] [<pid>] [INFO] Listening at: http://0.0.0.0:7777 (<pid>)

# Email handler on the custom port (separate shell)
python email_handler.py --port 25025
# -> INFO  "Listen for port 25025"
# -> DEBUG "Start mail controller 0.0.0.0 25025"
```

### 9.6 Verification / Reproduction of Q1–Q5

```bash
# Q1 — total tables + last-created table on a fresh empty DB
su postgres -c "createdb -O test q1a"
echo 'drop schema public cascade; create schema public;' \
  | PGPASSWORD=test psql "postgresql://test:test@localhost:5432/q1a"
DB_URI="postgresql://test:test@localhost:5432/q1a" alembic upgrade head
PGPASSWORD=test psql "postgresql://test:test@localhost:5432/q1a" -tAc \
  "select count(*) from information_schema.tables where table_schema='public';"   # -> 77

# Q2 — readiness line (grep the startup log)
gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15 2>&1 | grep "Listening at:"

# Q3 — port confirmation
python email_handler.py --port 25025 2>&1 | grep -E "Listen for port 25025|Start mail controller 0.0.0.0 25025"

# Q4 — register, then login BEFORE activation
curl -si -X POST http://localhost:7777/api/auth/register \
  -H 'Content-Type: application/json' \
  -d '{"email":"testuser@example.com","password":"testpass123"}'     # -> 200
curl -si -X POST http://localhost:7777/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"testuser@example.com","password":"testpass123"}'     # -> 422 {"error":"Account not activated"}
PGPASSWORD=test psql "$DB_URI" -tAc \
  "select activated, notification from users where email='testuser@example.com';"   # -> f | t

# Q5 — dynamic limit before vs after (see §A of the deliverable for the full activate+api-key flow)
curl -s -H "Authentication: $API_KEY" http://localhost:7777/api/user_info   # "max_alias_free_plan": 3
export MAX_NB_EMAIL_FREE_PLAN=10 ; # restart gunicorn, then re-query -> 10 for both users
```

### 9.7 Doc-Level Verification (safe in the delivered checkout)

```bash
git status --porcelain                                   # empty  -> read-only guarantee intact
git diff --name-only 2cd6ee77..HEAD                      # single file: blitzy/documentation/app_2cd6ee777f8c.md
wc -l blitzy/documentation/app_2cd6ee777f8c.md           # 978
```

### 9.8 Cleanup

```bash
pkill -f 'gunicorn wsgi:app' ; pkill -f 'email_handler.py'
for db in q1a q1b qa_main ; do su postgres -c "dropdb --if-exists $db" ; done
redis-cli flushall
rm -rf /tmp/sl_*.*    # mktemp workspaces
exit                  # leave container
docker rm -f sl_qafix # destroy container (removes checkout, DBs, Redis, keys, temp)
```

### 9.9 Troubleshooting

- **`register` returns 500 `{"error":"Internal error"}`** → DKIM key is PKCS#8; apply the §9.4 PKCS#1 conversion and restart.
- **Every request returns 500** → Redis is down; start Redis and retry.
- **`email_handler.py` exits 1 with errno 98** → port already in use; choose a free port or stop the occupant.
- **`pyre2` build fails during `poetry install`** → expected; use `/build.sh`, which installs the prebuilt wheel + `google-re2`.
- **Gunicorn exits with status 3 immediately** → missing/invalid `DB_URI` (worker boot error); set a valid `DB_URI`.

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---|---|
| Launch container | `docker run -d --init --name sl_qafix --entrypoint /bin/sleep <image> infinity` |
| Canonical build | `bash /build.sh` |
| Run migrations | `alembic upgrade head` |
| Start web server | `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` |
| Start email handler | `python email_handler.py --port 25025` |
| Count tables | `psql "$DB_URI" -tAc "select count(*) from information_schema.tables where table_schema='public';"` |
| Verify read-only | `git status --porcelain` (empty) |

### B. Port Reference

| Port | Service | Notes |
|---|---|---|
| 7777 | Gunicorn web server | Canonical bind (`Dockerfile:47`) |
| 25025 | Email handler (Q3 custom) | `--port 25025` |
| 20381 | Email handler default | Contrast value when `--port` omitted |
| 5432 | PostgreSQL | Backing DB (mandated image) |
| 6379 | Redis | Backing cache/rate-limit store |

### C. Key File Locations

| Path | Role |
|---|---|
| `blitzy/documentation/app_2cd6ee777f8c.md` | The deliverable (only persisted change) |
| `wsgi.py` / `server.py` | Gunicorn entry point / `create_app()` (Q2) |
| `email_handler.py` | aiosmtpd controller + CLI (`:2386`, `:2403`, default `:2399`) (Q3) |
| `app/api/views/auth.py` | Register/login endpoints; 422 branch `:75-77` (Q4) |
| `app/api/views/user_info.py` | `/user_info`; `max_alias_free_plan` `:34` (Q5) |
| `app/config.py` | `MAX_NB_EMAIL_FREE_PLAN` read at import `:121-124` (Q5) |
| `app/models.py` | `activated`/`notification` defaults `:354-358`; `max_alias_for_free_account()` `:858-865` |
| `app/log.py` | Werkzeug logger disabled `:70-71` (Q2 caveat) |
| `alembic.ini` / `migrations/versions/*.py` | Migration config + 255 migration files (Q1) |
| `scripts/reset_local_db.sh` | Canonical empty-DB pattern (Q1) |
| `Dockerfile` | Canonical start command `:47` (Q2) |

### D. Technology Versions (observed in mandated image)

| Component | Version |
|---|---|
| Python | 3.10.18 |
| OpenSSL | 3.0.16 |
| gunicorn | 20.0.4 |
| alembic | 1.4.3 |
| Flask | 1.1.2 |
| SQLAlchemy | 1.3.24 |
| aiosmtpd | 1.4.2 |
| redis (client) | 4.6.0 |
| bcrypt | 3.2.0 |
| PostgreSQL (service) | 15.13 |
| Redis (service) | 7.0.15 |

### E. Environment Variable Reference

| Variable | Role | Value used |
|---|---|---|
| `DB_URI` | PostgreSQL connection string | `postgresql://test:test@localhost:5432/<db>` |
| `NOT_SEND_EMAIL` | Suppress real MTA so registration completes | `true` |
| `MAX_NB_EMAIL_FREE_PLAN` | Free-plan alias limit (read once at import) | `3` (build) → `10` (Q5 change); unset → `5` |
| `DKIM_PRIVATE_KEY_PATH` | Override to a PKCS#1 key copy | `$WORKDIR/dkim.key` |
| `DISABLE_REGISTRATION` | Must remain unset so registration stays open | unset |

### F. Developer Tools Guide

| Task | Tool / Command |
|---|---|
| Inspect commits on branch | `git log --oneline 2cd6ee77..HEAD` |
| Confirm authorship | `git log --author="agent@blitzy.com" --oneline` |
| Per-file diff vs base | `git diff 2cd6ee77..HEAD -- <path>` |
| Verify no source modified | `git diff --name-status 2cd6ee77..HEAD` (single `A` entry) |
| Query DB directly | `psql "$DB_URI" -tAc "<SQL>"` |
| Exercise the API | `curl -si -X POST http://localhost:7777/api/...` |

### G. Glossary

| Term | Meaning |
|---|---|
| AAP | Agent Action Plan — the governing project directive |
| Alembic DAG | The `down_revision` chain that determines migration execution order (not filename order) |
| Config-at-import | A value read once when a module is imported; requires a process restart to change (e.g., `MAX_NB_EMAIL_FREE_PLAN`) |
| Observed vs Inferred | "Observed" = captured from live runtime output; "Inferred" = deduced from code without runtime capture (this deliverable has none) |
| Read-only guarantee | The mandate that no existing source file is modified; the only persisted change is the deliverable |
| Socket-bind vs request-serving readiness | Gunicorn's `Listening at:` (socket bound) precedes true app readiness (workers finished importing `wsgi:app`) |
| PKCS#8 / PKCS#1 | RSA private-key encodings; dkimpy requires PKCS#1 (`BEGIN RSA PRIVATE KEY`) |
