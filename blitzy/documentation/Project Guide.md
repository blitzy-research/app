# Blitzy Project Guide — SimpleLogin Server-Side Session Deserialization Investigation

> **Artifact under review:** `blitzy/documentation/app_2cd6ee777f8c.md` (444 lines) — a documentation-only security investigation of how the SimpleLogin Flask application deserializes server-side session data via Python `pickle` in Redis.
>
> **Color legend (Blitzy brand):** Completed / AI Work = **Dark Blue `#5B39F3`** · Remaining / Not Completed = **White `#FFFFFF`** · Headings / Accents = **Violet-Black `#B23AF2`** · Highlight = **Mint `#A8FDD9`**.

---

## 1. Executive Summary

### 1.1 Project Overview

This project is a **documentation-only security investigation** of the SimpleLogin Flask application (an email-alias/privacy service). The objective was to empirically determine and document how the app's custom server-side session store deserializes Redis-stored session bytes with `pickle`, and to locate the precise boundary between a harmless session reset and a genuine remote-code-execution (RCE) risk. The deliverable is a single self-contained markdown answer document resolving five user questions, each grounded in exact code citations and validated against the live running application. The target audience is the SimpleLogin engineering and security teams. No application code was changed; repository immutability was a hard constraint.

### 1.2 Completion Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieLegendTextColor':'#000000','pieStrokeWidth':'2px'}}}%%
pie showData title Completion — 90.0% Complete (30h total)
    "Completed Work (AI)" : 27
    "Remaining Work" : 3
```

| Metric | Hours |
|--------|-------|
| **Total Hours** | **30** |
| Completed Hours (AI + Manual) | 27 (27 AI + 0 Manual) |
| Remaining Hours | 3 |
| **Percent Complete** | **90.0%** |

> Completion is computed strictly from AAP-scoped work + path-to-production using the hours-based formula: **27 / (27 + 3) = 90.0%**. Remediation of the documented `pickle` risk is **out of scope** per the AAP and is therefore **excluded** from remaining hours.

### 1.3 Key Accomplishments

- ✅ Single answer document `blitzy/documentation/app_2cd6ee777f8c.md` authored (444 lines / 6,163 words), answering all five user questions with code citations, rationale, and empirical evidence.
- ✅ All five questions **empirically validated** against the **live** `RedisSessionStore` (real `open_session`/`save_session`/`_get_signer` executed; not a re-implementation).
- ✅ `pickle` confirmed as the **single deserialization sink** in the entire repository (only `app/session.py`).
- ✅ Central thesis established and proven: the meaningful trust boundary is **Redis write access**, not the cookie signature.
- ✅ The crafted-`__reduce__` RCE primitive demonstrated executing during `pickle.loads` (`app/session.py:L76`); garbage bytes shown to reset harmlessly — boundary = pickle-stream **validity**.
- ✅ The HMAC-SHA1 cookie signature reproduced **byte-for-byte**, proving the empirical claims came from the real runtime.
- ✅ **Repository immutability preserved**: the only change vs. base is the one new markdown file (444 insertions, 0 deletions); working tree clean.
- ✅ Citations verified 100% accurate; markdown structural sanity passes (balanced fences, valid Mermaid, consistent tables).

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| Document not yet human-reviewed/signed-off | Analytical security claims should be SME-validated before informing decisions | Security/Backend SME | 0.5 day |
| (Informational) Documented `pickle`→RCE finding remains unmitigated | Real app risk; **remediation explicitly out of scope** for this task — requires a separate decision/project | SimpleLogin Security | Backlog (separate scope) |

> There are **no blocking defects** in the deliverable itself. The first row is the in-scope path-to-production gap (review/sign-off). The second row is a finding the document **surfaces**, not a defect introduced by this task; its remediation is out of scope.

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-----------------|----------------|-------------------|-------------------|-------|
| SimpleLogin git repository | Read/Write | None — branch cloned, deliverable committed | ✅ Resolved | Blitzy Agent |
| Empirical runtime (PostgreSQL + Redis + Python 3.10) | Provisioned container | Live validation ran in the warmed container `sl-qna-env-0`; the **default review sandbox** has Python 3.13 and lacks `redis-cli`/`poetry`, so re-running probes requires the provisioned environment | ⚠ Documented (not blocking) | Reviewer |

> No credential, repository-permission, or third-party-API access issues prevent build validation, integration, or merge of this documentation deliverable.

### 1.6 Recommended Next Steps

1. **[High]** Conduct an SME/security technical review of the analytical findings and the central trust-boundary thesis; spot-check the empirical evidence and key code citations. (2h)
2. **[Medium]** Verify repository immutability/placement compliance and approve the PR for merge. (1h)
3. **[Low — separate scope]** Convene a decision on whether to pursue the documented (out-of-scope) mitigations (safe serializer, payload HMAC, logging the silent `except`, deleting stale keys). These are **not** part of this project's hours.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|-------|-------------|
| Runtime environment provisioning & build/run | 5 | Python 3.10 + Poetry-pinned deps + PostgreSQL (port 15432) + Redis; `CONFIG=tests/test.env` to activate `RedisSessionStore` (`MEM_STORE_URI` gate); `alembic upgrade head` [AAP R10] |
| Source-code tracing & citation grounding | 4 | Read `app/session.py` end-to-end + ~20 session-writer view files + `config.py`/`server.py`/`extensions.py`/`redis_services.py` to anchor every claim [AAP R1–R9] |
| Empirical probing of all five behaviors | 6 | Normal mint/read; corruption ×4 payloads; crafted `__reduce__` RCE (marker-file proof); signature reproduction + tamper/read-no-key spy [AAP R11–R12] |
| Web-search corroboration | 1 | `pickle` RCE model; `itsdangerous` `Signer` semantics; GHSA-g8c6-8fjj-2r4m ≡ CVE-2025-61765 precedent [AAP R13] |
| Authoring the answer document | 7 | 444 lines / 6,163 words: Q1–Q5 + overview + Mermaid diagram + environment/reproduction + references + central thesis [AAP R1–R9, R16] |
| Autonomous validation & QA | 4 | Citation audit (100% accurate), immutability verification, byte-for-byte signature reproduction, markdown sanity, 3 correction commits [AAP R14–R15, R17] |
| **Total Completed** | **27** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|-------|----------|
| Security/SME technical review of analytical findings & central thesis | 2 | High |
| Immutability/placement compliance check + final sign-off & merge approval | 1 | Medium |
| **Total Remaining** | **3** | |

> **Out of scope (NOT counted in the 30h total):** remediation of the documented `pickle` risk — replace `pickle` with JSON/MessagePack, add an HMAC over the Redis payload, add logging to the silent `except` branch, or delete the offending key on failure. Per the AAP these are analytical-only and would constitute a separate project.

---

## 3. Test Results

All entries below originate from **Blitzy's autonomous validation logs** for this project and were corroborated by inline re-verification against the source. Because this is a documentation-only task with **no compilable/testable application module as a deliverable**, the validation gates were reinterpreted: *citation accuracy* serves as the "compilation" analogue and *empirical behavioral validation* serves as the "test" analogue. No project unit-test files were added (adding tests is out of scope); the application's own behavior was validated empirically against the live store.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|-----------|-------|
| Empirical behavioral validation (Q1–Q5) | Live `RedisSessionStore` + Flask `test_client` + Redis probes | 5 | 5 | 0 | 100% | Each user question validated against the real `open_session`/`save_session`/`_get_signer` |
| Corruption sub-probes (Q2/Q3) | `pickle.loads` on mutated Redis keys | 4 | 4 | 0 | 100% | Garbage ASCII, random binary, truncated, empty → `UnpicklingError`×3 + `EOFError`; all silent reset, HTTP 200 (not 500), 0 log lines |
| Crafted RCE probe (Q4) | `__reduce__` → `os.system` marker file | 1 | 1 | 0 | 100% | Code executed during `pickle.loads` (`L76`) without tripping the `except` |
| Signature/tamper validation (Q5) | Real `_get_signer` (HMAC-SHA1) | 4 | 4 | 0 | 100% | Exact sig reproduced byte-for-byte; value-flip / sig-flip / wrong-secret → `BadSignature`; tampered cookie → Redis key never read |
| Citation accuracy ("compilation" analogue) | Manual cross-reference (`grep`/`sed` vs. source) | All cited locators | All | 0 | 100% | Single-sink confirmed (only `app/session.py`); all sampled locators (L76, L78-79, L91, L37-41, etc.) verified exact |
| Markdown structural sanity | `grep`/`wc` lint | 5 checks | 5 | 0 | n/a | 47 headings; 20 code fences (balanced/even); 1 valid Mermaid block; 43 consistent table rows; 0 trailing-whitespace; trailing newline present |

> **Integrity note:** the figures above are sourced from Blitzy's autonomous validation of this branch; the document-review track (Section 9) was additionally re-run inline during this assessment and all checks passed.

---

## 4. Runtime Validation & UI Verification

**Runtime health (live container `sl-qna-env-0`; `app/session.py` md5 `0af288a039deb1620a154fcc63e77ed8` byte-identical host↔container):**

- ✅ **Operational** — `RedisSessionStore` active under `MEM_STORE_URI=redis://localhost`; `isinstance(app.session_interface, RedisSessionStore)` is `True`.
- ✅ **Operational** — Login flow via `test_client`: authenticated session written to `session:<uuid>` with TTL **604800s (7 days)**; dict `[_fresh, _id, _permanent, _user_id, sudo_time]`.
- ✅ **Operational** — Logout: HTTP **302**, old key deleted (TTL `-2`), new anonymous key (TTL `300`), two `slapp` `Set-Cookie` directives (delete + rotated).
- ✅ **Operational** — Corruption replay: HTTP **200 (not 500)**, fresh `slapp` `Set-Cookie`, **0** corruption log lines → confirms silent reset.
- ✅ **Operational** — Crafted `__reduce__` pickle: shell command executed during `pickle.loads`; `open_session` returned a restored session without raising.
- ✅ **Operational** — Signature: HMAC-SHA1 confirmed (`openssl_sha1` + `key_derivation=hmac` + salt `session`); tampered cookie → no Redis read.
- ✅ **Operational** — Backing services: PostgreSQL **15.13** and Redis **7.0.15** reachable; Python **3.10.18**.

**API integration:** The Flask session API surface was exercised end-to-end via `test_client` (login `200`, logout `302`); PostgreSQL and Redis connectivity verified.

**UI verification:** ⚠ **Not applicable** — this is a backend/CLI documentation investigation with **no web-UI deliverable**. No browser-based UI exists to verify for this task; no Figma designs were provided.

---

## 5. Compliance & Quality Review

This matrix cross-maps the AAP deliverables and governing rules ("SWE-AtlasQnA-Repo") to their validated status.

| Benchmark / AAP Requirement | Status | Progress | Notes |
|-----------------------------|--------|----------|-------|
| Single answer document created | ✅ Pass | 100% | `blitzy/documentation/app_2cd6ee777f8c.md` |
| Named after source branch (`app_2cd6ee777f8c.md`) | ✅ Pass | 100% | Exact filename match |
| Placed in `blitzy/documentation/` | ✅ Pass | 100% | Exact path match |
| All five questions answered with citations + rationale | ✅ Pass | 100% | Q1–Q5 each cite `file:Lxx` + rationale |
| Empirical (not theoretical) methodology | ✅ Pass | 100% | All claims backed by live-runtime observation |
| Code is the source of truth | ✅ Pass | 100% | Citations verified exact; web sources corroborating only |
| Repository immutability (no source modified) | ✅ Pass | 100% | `git diff` = only the new doc; tree clean |
| Documentation-only — no remediation implemented | ✅ Pass | 100% | Mitigations discussed analytically in §5.3, not coded |
| Temporary scratch scripts cleaned up | ✅ Pass | 100% | `/tmp/sl_probes` removed; tree clean before & after |
| Active-path scoping (`MEM_STORE_URI`) stated | ✅ Pass | 100% | Doc §1.1 scopes all conclusions to the active path |
| Reproducibility recorded (versions + steps) | ✅ Pass | 100% | Doc §7 records versions, activation, method, provenance |
| Human SME review / sign-off | ⏳ Pending | 0% | The 3h remaining path-to-production work |

**Fixes applied during autonomous validation:** the document was refined across 3 commits — the initial investigation, a correction to the session-writer inventory and Redis runtime version, and a correction to the Q1 logout final-state — improving factual precision. **Outstanding:** human review/sign-off only.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| `pickle` deserialization → RCE: a well-formed `__reduce__` pickle at a validly-signed `session:<uuid>` key executes arbitrary code at `pickle.loads` (`app/session.py:L76`) | Security | High | Low–Medium (requires Redis write access **and** `MEM_STORE_URI` active) | Documented analytically (safe serializer / payload HMAC, doc §5.3); **remediation out of scope** | Open — Documented (deferred to humans) |
| Silent corruption/tamper invisibility: malformed bytes reset the session with no log line and no HTTP 500 (`except Exception: pass`, `L78-79`) | Security / Operational | Medium | Medium | Documented analytically (log/alert in `except`; delete stale key); **out of scope to implement** | Open — Documented |
| Citation line-number drift if `app/session.py` is later refactored | Technical | Low | Low | Point-in-time snapshot pinned to branch/commit; records `session.py` md5 | Mitigated |
| Empirical probes not reproducible in the default sandbox (needs Py 3.10 + PostgreSQL + Redis) | Technical / Integration | Low | Medium | Doc §7 records exact versions + activation + bootstrap command | Mitigated (documented) |
| Document pending human review/sign-off before informing security decisions | Operational | Medium | Medium (until reviewed) | Schedule the 3h SME/security review + sign-off | Open — Pending review |

> The first two rows are **findings the deliverable surfaces** about the SimpleLogin application — they are the subject of the investigation, not defects introduced by this task. The documentation task itself adds zero code and introduces zero new runtime risk.

---

## 7. Visual Project Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieLegendTextColor':'#000000','pieStrokeWidth':'2px'}}}%%
pie showData title Project Hours Breakdown (30h total)
    "Completed Work" : 27
    "Remaining Work" : 3
```

**Remaining work by category (Section 2.2 — total 3h):**

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#B23AF2','pieStrokeColor':'#FFFFFF','pieOuterStrokeColor':'#B23AF2','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#FFFFFF','pieLegendTextColor':'#000000','pieStrokeWidth':'1px'}}}%%
pie showData title Remaining Work by Category (3h)
    "SME/Security Review [High]" : 2
    "Compliance Check + Sign-off [Medium]" : 1
```

> **Integrity check:** Pie "Remaining Work" (3) = Section 1.2 Remaining (3h) = Section 2.2 total (3h). Pie "Completed Work" (27) = Section 1.2 Completed (27h) = Section 2.1 total (27h). Colors: Completed = Dark Blue `#5B39F3`, Remaining = White `#FFFFFF`.

---

## 8. Summary & Recommendations

**Achievements.** The project delivered a rigorous, empirically-grounded answer to all five user questions in a single, immutability-compliant document. It identifies `pickle` as the sole deserialization sink, characterizes the normal session lifecycle (7-day authenticated / 300s anonymous TTLs, `slapp` HMAC-SHA1 cookie), proves that malformed bytes are **silently reset** (answer to Q2 = **(b)**) with no HTTP 500 and no log line, demonstrates that a valid `__reduce__` pickle **executes code** at `pickle.loads`, and establishes the central thesis that the **trust boundary is Redis write access, not the cookie signature**.

**Remaining gaps & critical path.** The project is **90.0% complete** (27 of 30 hours). The only remaining work is **path-to-production human review**: a 2h SME/security review of the findings and a 1h immutability/placement compliance check plus sign-off and merge. There are no defects to fix in the deliverable.

**Production-readiness assessment.** As a documentation artifact, the deliverable is **ready for review and merge**: it is accurate (citations 100% verified), complete (all five questions answered with rationale + empirical evidence), and compliant (no source modified, correct name/location, scratch tooling cleaned up). Final sign-off is a human gate by design.

**Success metrics.**

| Metric | Target | Actual |
|--------|--------|--------|
| User questions answered | 5/5 | 5/5 ✅ |
| Citation accuracy | 100% | 100% ✅ |
| Repository immutability | 0 source files changed | 0 changed ✅ |
| Empirical validation | All claims runtime-verified | All verified ✅ |
| Completion (AAP-scoped) | — | **90.0%** |

**Recommendation.** Proceed to SME review and merge. Separately (and outside this project's scope), schedule a decision on the documented `pickle` mitigations.

---

## 9. Development Guide

This guide covers two tracks: **(A) Document Review** — the in-scope path-to-production activity (fully runnable with just `git` + a text viewer), and **(B) Empirical Reproduction** — rebuilding the runtime to re-run the probes (requires Python 3.10 + PostgreSQL + Redis).

### 9.1 System Prerequisites

- **For document review (Track A):** `git`, any text/markdown viewer. No build required.
- **For empirical reproduction (Track B):** Python **3.10** (project pin — note the default sandbox ships Python 3.13, a mismatch), **Poetry**, **PostgreSQL** (test harness uses port 15432), **Redis**, **Docker** (to run `postgres:13`).

### 9.2 Environment Setup

```bash
# Clone / enter the repository (branch under review)
cd /tmp/blitzy/app/blitzy-02437353-3a52-4330-a3bc-42279dbf5e21_daf2a9

# (Track B only) the investigation config activates the Redis session store:
#   tests/test.env sets:
#     MEM_STORE_URI=redis://localhost      # L78 — activates RedisSessionStore
#     DB_URI=postgresql://test:test@localhost:15432/test   # L17
#     FLASK_SECRET=secret                  # L20 — PUBLISHED TEST VALUE, not a production secret
export CONFIG=tests/test.env
```

### 9.3 Dependency Installation (Track B)

```bash
# Install the pinned dependencies (Flask 1.1.2, Flask-Login 0.5.0, itsdangerous 1.1.0,
# Werkzeug 1.0.1, redis 4.6.0, limits 1.5.1, Flask-Limiter 1.4)
poetry install
```

### 9.4 Application Startup (Track B)

```bash
# 1) Start PostgreSQL 13 (mirrors scripts/run-test.sh)
docker run -d --name sl-test-db \
  -e POSTGRES_PASSWORD=test -e POSTGRES_USER=test -e POSTGRES_DB=test \
  -p 15432:5432 postgres:13
sleep 3   # allow the DB to accept connections

# 2) Ensure a local Redis is running (so MEM_STORE_URI=redis://localhost resolves)
#    e.g. `redis-server &` or `docker run -d --name sl-redis -p 6379:6379 redis:7`

# 3) Apply migrations
CONFIG=tests/test.env poetry run alembic upgrade head

# 4) Construct the app (wsgi entrypoint: app = create_app())
#    For a one-off shell:  CONFIG=tests/test.env poetry run python -c "from wsgi import app; print(type(app.session_interface).__name__)"
#    Expected output: RedisSessionStore
```

### 9.5 Verification

**Track A — Document review (run inline; all verified passing during this assessment):**

```bash
# 1) Deliverable exists
test -f blitzy/documentation/app_2cd6ee777f8c.md && wc -l blitzy/documentation/app_2cd6ee777f8c.md   # -> 444

# 2) Immutability — only the doc was added vs base
git diff --name-status origin/app_2cd6ee777f8c...HEAD     # -> A  blitzy/documentation/app_2cd6ee777f8c.md

# 3) Working tree clean
git status --porcelain                                    # -> (empty)

# 4) Authorship
git log --author="agent@blitzy.com" origin/app_2cd6ee777f8c..HEAD --oneline   # -> 3 commits

# 5) pickle single-sink claim
grep -rln "pickle" app/ server.py                          # -> app/session.py

# 6) Spot-check key citations
sed -n '76p;78,79p;91p' app/session.py
#   L76 -> data = pickle.loads(val)
#   L78 -> except Exception:
#   L79 -> pass
#   L91 -> val = pickle.dumps(dict(session))

# 7) Markdown structural sanity
DOC=blitzy/documentation/app_2cd6ee777f8c.md
grep -c '^```' "$DOC"        # even number (balanced fences)
grep -c '^```mermaid' "$DOC" # >= 1
```

**Track B — Empirical reproduction (requires the provisioned runtime):**

1. Mint a real session through a `test_client()` login, then `redis-cli GET session:<uuid>` to read the pickled dict and observe its size/TTL.
2. Overwrite the key with (a) garbage ASCII, (b) random binary, (c) a truncated pickle, (d) empty bytes; replay a request → expect a fresh session, HTTP 200, no log line, a new `slapp` cookie.
3. Overwrite the key with a well-formed `__reduce__` pickle whose payload writes a marker file; invoke `open_session` → the marker appears (code executed at `L76`), no exception raised.
4. Drive the real `_get_signer(app)` → confirm HMAC-SHA1, valid-UUID recovery, and `BadSignature` on value/signature/secret tamper; feed a tampered cookie to `open_session` → no Redis key is read.

> The warmed environment can be brought up with: `CLONE_INDEX=0 bash /tmp/blitzy/sl_qna_bootstrap.sh up` (container `sl-qna-env-0`).

### 9.6 Example Usage

```bash
# Re-read the exact read-path the document analyzes
sed -n '68,80p' app/session.py

# Confirm the MEM_STORE_URI activation gate
grep -n "MEM_STORE_URI" server.py app/config.py
#   server.py: if MEM_STORE_URI: initialize_redis_services(app, MEM_STORE_URI)
#   app/config.py:568: MEM_STORE_URI = os.environ.get("MEM_STORE_URI", None)
```

### 9.7 Troubleshooting

- **No `pickle`/Redis behavior at all:** `MEM_STORE_URI` is unset → Flask's default **signed-cookie** session is used and the entire deserialization path is inactive (the five questions are moot). Set `CONFIG=tests/test.env`.
- **Python version mismatch:** the default review sandbox runs Python 3.13; the project targets **3.10**. Use the provisioned container (or a `pyenv`/`venv` 3.10) for Track B.
- **`redis-cli` / `poetry` not found:** absent in the default sandbox — install them or use the warmed container `sl-qna-env-0`.
- **`alembic upgrade head` fails to connect:** PostgreSQL on port 15432 is not ready yet — increase the `sleep` after `docker run`.

---

## 10. Appendices

### Appendix A — Command Reference

| Purpose | Command |
|---------|---------|
| Confirm deliverable exists | `wc -l blitzy/documentation/app_2cd6ee777f8c.md` |
| Immutability check | `git diff --name-status origin/app_2cd6ee777f8c...HEAD` |
| Working-tree clean check | `git status --porcelain` |
| Commit authorship | `git log --author="agent@blitzy.com" origin/app_2cd6ee777f8c..HEAD --oneline` |
| `pickle` single-sink check | `grep -rln "pickle" app/ server.py` |
| Citation spot-check | `sed -n '76p;78,79p;91p' app/session.py` |
| Migrate test DB | `CONFIG=tests/test.env poetry run alembic upgrade head` |
| Confirm store active | `CONFIG=tests/test.env poetry run python -c "from wsgi import app; print(type(app.session_interface).__name__)"` |

### Appendix B — Port Reference

| Service | Port | Notes |
|---------|------|-------|
| PostgreSQL (test harness) | 15432 | Mapped to container 5432 by `scripts/run-test.sh` |
| Redis | 6379 | Backs `MEM_STORE_URI=redis://localhost` |

### Appendix C — Key File Locations

| File | Role |
|------|------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The deliverable** (444 lines) |
| `app/session.py` | The single `pickle` sink: `open_session` (`pickle.loads` L76; `except` L78-79), `save_session` (`pickle.dumps` L91; TTL L95-96), `_get_signer` L37-41 |
| `app/redis_services.py` | Installs `RedisSessionStore` (L9-25) |
| `server.py` | App factory; `MEM_STORE_URI` activation gate (L163-165) |
| `app/config.py` | `SESSION_COOKIE_NAME="slapp"` (L199); `MEM_STORE_URI` (L568) |
| `app/extensions.py` | `session_protection = "strong"` (L8) |
| `tests/test.env` | Activates the store: `MEM_STORE_URI` (L78), `DB_URI` (L17), `FLASK_SECRET` (L20) |
| `scripts/run-test.sh` | Repro harness (postgres:13, alembic, pytest) |

### Appendix D — Technology Versions

| Component | Version | Source of truth |
|-----------|---------|-----------------|
| Python | 3.10.18 | `pyproject.toml` `^3.10`; `Dockerfile FROM python:3.10`; CI matrix `["3.10"]` |
| Flask | 1.1.2 | `poetry.lock` |
| Flask-Login | 0.5.0 | `poetry.lock` |
| itsdangerous | 1.1.0 | `poetry.lock` (default SHA-1 + `key_derivation="hmac"` ⇒ HMAC-SHA1) |
| Werkzeug | 1.0.1 | `poetry.lock` |
| redis (py) | 4.6.0 | `poetry.lock` |
| limits | 1.5.1 | `poetry.lock` |
| Flask-Limiter | 1.4 | `poetry.lock` |
| PostgreSQL | 13 (harness) / 15.13 (container) | `scripts/run-test.sh` / observed |
| Redis | 7.0.15 | observed in container `sl-qna-env-0` |

### Appendix E — Environment Variable Reference

| Variable | Value (test harness) | Purpose |
|----------|----------------------|---------|
| `CONFIG` | `tests/test.env` | dotenv file the app loads |
| `MEM_STORE_URI` | `redis://localhost` | **Activates `RedisSessionStore`**; unset ⇒ Flask default cookie session |
| `DB_URI` | `postgresql://test:test@localhost:15432/test` | PostgreSQL connection |
| `FLASK_SECRET` | `secret` | Signs the `slapp` cookie (**published test value**, not production) |

### Appendix F — Developer Tools Guide

- **`git`** — immutability and authorship verification (Appendix A).
- **`grep` / `sed` / `wc`** — citation re-verification and markdown sanity.
- **`poetry`** — dependency management for Track B reproduction.
- **`docker`** — provisioning `postgres:13` for the test harness.
- **`redis-cli`** — reading/mutating `session:<uuid>` keys during empirical probes.
- **`alembic`** — applying database migrations before app startup.

### Appendix G — Glossary

| Term | Definition |
|------|------------|
| `pickle` | Python's native object serialization; **unsafe on untrusted input** because `__reduce__` can execute arbitrary code on load |
| `__reduce__` | The pickle protocol hook that names a callable + args to reconstruct an object — the RCE primitive |
| `RedisSessionStore` | SimpleLogin's custom Flask `SessionInterface` storing pickled session dicts in Redis |
| `slapp` | The session cookie name; carries only the itsdangerous-signed session UUID |
| `MEM_STORE_URI` | Env var that, when a redis URL, activates the server-side `pickle`/Redis session path |
| HMAC-SHA1 | The effective signature algorithm for the `slapp` cookie (itsdangerous 1.1.0 default SHA-1 + `key_derivation="hmac"`) |
| Trust boundary | The line separating trusted from untrusted input; here proven to be **Redis write access** |
| GHSA-g8c6-8fjj-2r4m | python-socketio advisory (≡ CVE-2025-61765, CWE-502) — the analogous Redis+pickle precedent |