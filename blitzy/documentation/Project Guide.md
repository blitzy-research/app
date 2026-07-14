# Blitzy Project Guide — SimpleLogin `RedisSessionStore` Deserialization Investigation

> **Deliverable class:** Read-only investigative Q&A / security-documentation artifact
> **Source branch:** `app_2cd6ee777f8c` · **Working branch:** `blitzy-69b57f9c-a60d-44e2-934e-aba188d82f39`
> **Base commit:** `2cd6ee77` · **HEAD:** `529df049`
> **Brand color legend:** 🟪 Completed / AI Work = Dark Blue `#5B39F3` · ⬜ Remaining / Not Completed = White `#FFFFFF` · Headings/Accents = Violet-Black `#B23AF2` · Highlight = Mint `#A8FDD9`

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a single, runtime-grounded investigative document answering how SimpleLogin's custom Flask `SessionInterface` — `RedisSessionStore` in `app/session.py` — deserializes server-side session data with Python `pickle`, and precisely what happens when the Redis-stored payload is corrupted, malformed, or maliciously tampered with. The target audience is SimpleLogin's security engineers and maintainers. The technical scope spans the session subsystem, its Redis activation/wiring, the login/logout entry points, session configuration, and the test harness. The business impact is a definitive, evidence-backed security finding (insecure deserialization, CWE-502) with a documented threat model — captured as authoritative internal knowledge without changing any production source.

### 1.2 Completion Status

The project is **90.9% complete**. All Agent Action Plan (AAP) deliverables are finished and validated; the remaining 9.1% consists solely of human path-to-production gates (subject-matter review, PR merge, and publication) that agents cannot self-approve.

```mermaid
%%{init: {"theme": "base", "themeVariables": {"pie1": "#5B39F3", "pie2": "#FFFFFF", "pieStrokeColor": "#B23AF2", "pieOuterStrokeColor": "#B23AF2", "pieStrokeWidth": "2px", "pieTitleTextColor": "#B23AF2", "pieSectionTextColor": "#111111", "pieLegendTextColor": "#111111"}}}%%
pie showData title Completion Status — 90.9% Complete
    "Completed Work (AI) — 50h" : 50
    "Remaining Work — 5h" : 5
```

| Metric | Hours |
|--------|-------|
| **Total Hours** | **55** |
| Completed Hours (AI) | 50 |
| Completed Hours (Manual) | 0 |
| **Completed Hours (AI + Manual)** | **50** |
| **Remaining Hours** | **5** |
| **Percent Complete** | **90.9%** |

> Completion is computed by the AAP-scoped hours method (PA1): `Completed ÷ (Completed + Remaining) × 100 = 50 ÷ 55 × 100 = 90.9%`.

### 1.3 Key Accomplishments

- ✅ Built the canonical runtime (Python 3.10, Poetry, PostgreSQL @ 15432, Redis) with `MEM_STORE_URI` set so `RedisSessionStore` is the active session interface.
- ✅ Documented the baseline session lifecycle (login → logout) from observed runtime output: signed `slapp` cookie, pickled payload at Redis key `session:<sid>`, `setex` TTLs (604800 authenticated / 300 anonymous).
- ✅ Empirically established the corrupted-payload behavior: a **silent reset** — `pickle.loads` raises, `except Exception: pass` swallows it, and a fresh `ServerSession(uuid4)` is returned; no error surfaced, no failure.
- ✅ Proved the observability outcome: the module imports **no logger**, so the reset is indistinguishable in logs from a never-logged-in user (fd-level A/B log capture).
- ✅ Demonstrated the harmless-reset ↔ genuine-RCE boundary with two weaponized-pickle proofs (MAL-1 `os.system` RCE returning `0`; MAL-2 auth-preserving) and an 8-row return-object matrix.
- ✅ Resolved the threat model with a signature-independence proof: the HMAC signs only the session-id pointer, not the unsigned pickle payload.
- ✅ Delivered `blitzy/documentation/app_2cd6ee777f8c.md` (1,193 lines, 117 `file:line` citations, 8 verbatim evidence blocks) while keeping the repository pristine (read-only scope).

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| None blocking release of the deliverable | The single artifact is complete, committed, and byte-accurate; all sub-questions answered | — | — |
| (Advisory, out of scope) Documented CWE-502 finding is not remediated | Security risk to the SimpleLogin app, not to this deliverable; remediation is a separate project per AAP §0.4.2 | SimpleLogin maintainers | Separate follow-up |

> There are **no unresolved issues that block acceptance of this documentation deliverable.** The CWE-502 item is a *documented finding*, intentionally left unremediated because remediation is explicitly out of this task's scope.

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-----------------|----------------|-------------------|-------------------|-------|
| Repository (source branch) | Git read/write | None — branch present, working tree clean | Resolved | — |
| PostgreSQL @ localhost:15432 | Service | None — accepting connections | Resolved | — |
| Redis @ 127.0.0.1:6379 | Service | None — responds `PONG` | Resolved | — |
| Docker Hub canonical image tag | Image pull | The Docker Hub tag named in setup notes is not pullable; the equivalent **GHCR** image is accessible and was used | Resolved (GHCR fallback) | — |

> Aside from the Docker Hub tag (resolved via the accessible GHCR image), **no access issues** prevent build validation, integration, or reproduction.

### 1.6 Recommended Next Steps

1. **[High]** Perform SME technical review of `blitzy/documentation/app_2cd6ee777f8c.md` — verify citations resolve, confirm all five sub-questions are answered, and spot-reproduce at least one evidence block in the canonical environment.
2. **[Medium]** Review and merge the pull request (single-file, read-only diff) to the main branch.
3. **[Low]** Publish the artifact to the internal security knowledge base and notify the security team / maintainers; apply access control (the doc embeds a working exploit demonstration).
4. **[High — separate project]** Open a remediation ticket for the documented CWE-502 finding (replace `pickle` with a safe serializer such as JSON, or sign/authenticate the payload, or restrict `Unpickler.find_class`). *Out of scope for this task; tracked for awareness.*
5. **[Medium — separate/ops]** Harden Redis operationally (network isolation, `AUTH`, monitoring writes to `session:*`).

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|-------|-------------|
| Canonical runtime environment build & provenance | 6 | Python 3.10 venv, `poetry install` (177 locked pins), PostgreSQL @ 15432 + Redis, GHCR container + native cross-check, recorded exact build/run commands (doc §1.2–§1.3) |
| Baseline session lifecycle observation | 4 | Real `POST auth.login` / `GET auth.logout`; captured `slapp` cookie = `signer.sign(sid)`, Redis `session:<sid>`, pickle `\x80\x04` (protocol 4), TTLs 604800/300 (§2, blocks 4.0–4.2) |
| Corrupted / malformed payload observation | 3 | Silent reset via `except Exception: pass` → `ServerSession(uuid4)`; before/during/after Redis + cookie state (§3, blocks 4.3/4.3b) |
| Response & log observability capture | 4 | fd-level A/B log capture, no-logger `grep` proof, six-token checks all `False` (§4, block 4.5) |
| Edge / error path observation | 3 | `None` (loads never called), empty `b''` (`EOFError`), truncated (`UnpicklingError`), `BadSignature` (pre-Redis reject) each exercised (§5, blocks 4.4a–d) |
| Boundary & malicious-pickle RCE demonstration | 6 | 8-row return-object matrix, MAL-1 `os.system` RCE (marker written, returns `0` → SAME sid), MAL-2 auth-preserving (§6, block 4.6) |
| Threat-model signature-independence proof | 3 | HMAC signs only the sid pointer; payload stored raw/unsigned; victim's own cookie reaches `pickle.loads` with attacker bytes (§7, block 4.7) |
| Answer document authoring | 10 | 1,193-line doc, 117 `file:line` citations, verbatim evidence blocks, §8 coverage table, §9 observed-vs-inferred ledger |
| Security classification & external research | 2 | CWE-502 / OWASP A08:2021, Python `pickle` docs, GHSA-g8c6-8fjj-2r4m / CVE-2025-61765 analogue (web-searched) |
| Byte-sensitivity verification & reproducibility | 3 | Byte-exact signatures/pickle bytes; `_id` fingerprint `002d6488…0c0916` identical across container + native; ≥2-run stability (§1.6, §9) |
| Read-only discipline & cleanup | 1 | Ownership-safe Redis cleanup, temp-script removal, repo-pristine verification (Appendix B) |
| Iterative QA validation & refinement cycles | 5 | 5 commits: initial + 16 code-review fixes + DB-isolation/git-ref corrections + QA report 7 (4 MAJOR) + reproducible-command hardening |
| **Total Completed** | **50** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|-------|----------|
| SME technical review of the answer document | 3 | High |
| PR review & merge to main | 1 | Medium |
| Publish to security knowledge base / notify stakeholders | 1 | Low |
| **Total Remaining** | **5** | |

> **Out-of-scope follow-ups (NOT counted in the 5h above):** CWE-502 remediation and Redis operational hardening are separate projects per AAP §0.4.2 and are intentionally excluded from this project's hours.

### 2.3 Hours Reconciliation & Methodology

| Reconciliation Check | Value | Status |
|----------------------|-------|--------|
| Section 2.1 completed rows sum | 50h | ✅ equals Section 1.2 Completed |
| Section 2.2 remaining rows sum | 5h | ✅ equals Section 1.2 Remaining |
| Section 2.1 + Section 2.2 | 55h | ✅ equals Section 1.2 Total |
| Completion formula | 50 ÷ 55 × 100 = 90.9% | ✅ equals Section 1.2 percent |
| Section 7 pie ("Completed":50, "Remaining":5) | 50 / 5 | ✅ matches Section 1.2 |

---

## 3. Test Results

All tests and observation runs below originate from Blitzy's autonomous validation logs for this project. The canonical `pytest` suite was **independently re-run by the Project Manager** during this assessment and reproduced the exact result.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|------------|-------|
| Auth login/logout (canonical entry-point template) | pytest | 3 | 3 | 0 | Targeted (n/a) | `tests/auth/test_login.py` — the template the doc cites for the real entry points; PM re-run = `3 passed, 18 warnings, EXIT 0` |
| Runtime observation harnesses | Custom Python (real entry points) | 5 | 5 | 0 | Sub-question (see below) | Evidence blocks 4.0–4.7; each ran EXIT 0 capturing verbatim output |
| PM independent verification probe | Custom Python | 1 | 1 | 0 | n/a | Confirmed `RedisSessionStore` active + silent-reset finding byte-for-byte |
| **Total** | | **9** | **9** | **0** | — | 100% pass / EXIT 0 |

**Sub-question coverage (the meaningful "coverage" for an investigative deliverable):** 5 of 5 named sub-questions + the umbrella "how is it deserialized?" question are answered and mapped to evidence in the document's §8 coverage table — **6/6 = 100% ✅**.

> **Warnings note:** the 18 `pytest` warnings are pre-existing third-party `DeprecationWarning`s (`flask_limiter`/`distutils`, `flask_admin`/`pkg_resources`, `gnupg`/`setDaemon`). They are not errors and are out of scope for this read-only task.

---

## 4. Runtime Validation & UI Verification

This is a backend session-subsystem investigation; there is **no UI component**. Runtime validation focuses on the application build, session-interface activation, and real entry-point exercise.

- ✅ **Application build** — `create_app()` succeeds under `CONFIG=tests/test.env` (Python 3.10.20 venv).
- ✅ **Session interface active** — `app.session_interface` is `RedisSessionStore` because `MEM_STORE_URI=redis://localhost` is set (`server.py:163-165`). Independently confirmed by PM probe.
- ✅ **Real entry points exercised** — `POST auth.login` (302 → dashboard) and `GET auth.logout` (redirect + cookie deletion) via the `flask_client` template.
- ✅ **Baseline lifecycle** — pickled payload written to `session:<sid>` via `setex`; signed `slapp` cookie == `signer.sign(sid)`; logout deletes the key + 3 cookies and re-sets a fresh anonymous cookie.
- ✅ **Corrupted-payload path** — overwriting the Redis payload drives the next real request into `pickle.loads` → raises → silent reset (new `uuid4` sid, no error). Reproduced by PM.
- ✅ **Edge paths** — missing/`None`, empty `b''`, truncated, and `BadSignature` all behave exactly as documented.
- ✅ **Backing services healthy** — Redis `PONG`; PostgreSQL @ 15432 accepting connections.
- ⚠ **API logout path** — documented by **code reference only** (not runtime-exercised); honestly labeled *Inferred* in §7.5/§9 (calls the identical `logout_session()`).
- ❌ *(none)* — no failing runtime checks.

**API integration outcome:** the session read/write path against Redis (`get`/`setex` of `session:<sid>`) is operational and observed end-to-end; no external third-party API integration is in scope.

---

## 5. Compliance & Quality Review

Cross-mapping the AAP directives (rule set **"SWE-AtlasQnA-Repo"**) and deliverable requirements to Blitzy's quality benchmarks.

| Benchmark / AAP Directive | Requirement | Status | Evidence / Fixes Applied |
|---------------------------|-------------|--------|--------------------------|
| Deliverable exists & correctly named | `blitzy/documentation/app_2cd6ee777f8c.md` | ✅ Pass | 1,193 lines; named for source branch |
| Investigate by RUNNING first | Runtime observation before writing | ✅ Pass | 8 verbatim evidence blocks (4.0–4.7) |
| Canonical entry points | `POST auth.login` / `GET auth.logout` | ✅ Pass | §1.4, §2; `test_login.py` template |
| Canonical build/configuration stated | Exact build & invocation commands | ✅ Pass | §1.2–§1.3 (hardened for literal reproducibility, commit `529df049`) |
| Actual, complete, unedited output | No paraphrase/elision of evidence | ✅ Pass | Blocks embedded verbatim |
| Byte-sensitive verification | Signatures & pickle bytes vs emitted | ✅ Pass | `signer.sign(sid)` matches cookie; `\x80\x04`; `_id` fingerprint identical |
| Exercise every condition | Primary + edge/error paths | ✅ Pass | §3, §5, §6 (matrix + 2 malicious pickles) |
| `file:line` grounding | Every factual claim cited | ✅ Pass | 117 citations; all verified exact by PM |
| Observed vs. inferred labeling | Honest classification | ✅ Pass | §9 ledger; API logout marked *Inferred* |
| Answer every part | All sub-questions covered | ✅ Pass | §8 coverage table (6/6 ✅) |
| Read-only scope | No source modified; temp scripts removed | ✅ Pass | `git diff` = only doc added; Appendix B |
| No dependency changes | Locked pins unchanged | ✅ Pass | 7 session-critical deps at exact `poetry.lock` versions |
| Zero placeholders/TODO | Production-ready document | ✅ Pass | Scan returned none |

**Fixes applied during autonomous validation (from git history):** 16 code-review findings addressed; Appendix A DB-isolation claim and stale git-provenance references corrected; QA report 7 (4 MAJOR) resolved; §1.3 build/run commands made literally reproducible.

**Outstanding compliance items:** none for the deliverable. (SME sign-off is a human gate captured in Section 2.2.)

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| R1 Environment reproducibility drift (per-run uuids/signatures/timestamps vary) | Technical | Low | Medium | §9 separates invariant vs per-run values; `_id` fingerprint byte-identical across environments | Mitigated |
| R2 Citation drift if source evolves (117 refs pinned to `2cd6ee77`) | Technical | Low | Low | Doc pins the exact commit; read-only task changes no source | Mitigated |
| R3 **Documented CWE-502 insecure `pickle` deserialization** — valid weaponized pickle executes inside `pickle.loads` (`app/session.py:76`) before the `except` at `:78` → RCE | Security | High | Low–Medium | Contingent on Redis write access; NOT reachable by a normal web client with only a signed cookie. Remediation (JSON / sign payload / restrict `find_class`) is a **separate, out-of-scope** follow-up | Open (documented by design) |
| R4 Exploit-primer sensitivity (doc embeds a working weaponized pickle) | Security | Low | Low | Treat as access-controlled internal security artifact at publication | Note (handle at publish) |
| R5 Deployment/regression risk from this task | Operational | None | — | Repo pristine; no source changed | N/A |
| R6 Operational exposure of the finding (Redis reachability) | Operational | Medium | Low | Network-isolate Redis, enable `AUTH`, monitor writes to `session:*` | Recommendation (out of scope to implement) |
| R7 Reproduction dependency (PG@15432 + Redis + Py3.10 + locked pins) | Integration | Low | Low | §1.3 exact commands + self-contained canonical GHCR container | Mitigated |
| R8 API-logout coverage gap (code-reference only) | Integration | Low | Low | Honestly labeled *Inferred* in §7.5/§9; calls identical `logout_session()` | Mitigated |

> **No critical risks to the deliverable.** The single High-severity item (R3) is the *documented finding itself* — correctly reported and correctly scoped as a follow-up remediation, not a defect of this documentation project.

---

## 7. Visual Project Status

**Overall hours breakdown** (🟪 Completed `#5B39F3` · ⬜ Remaining `#FFFFFF`):

```mermaid
%%{init: {"theme": "base", "themeVariables": {"pie1": "#5B39F3", "pie2": "#FFFFFF", "pieStrokeColor": "#B23AF2", "pieOuterStrokeColor": "#B23AF2", "pieStrokeWidth": "2px", "pieTitleTextColor": "#B23AF2", "pieSectionTextColor": "#111111", "pieLegendTextColor": "#111111"}}}%%
pie showData title Project Hours Breakdown (Total 55h)
    "Completed Work" : 50
    "Remaining Work" : 5
```

**Remaining work by priority** (sums to the 5h in Sections 1.2 / 2.2):

```mermaid
%%{init: {"theme": "base", "themeVariables": {"pie1": "#5B39F3", "pie2": "#8A6FF6", "pie3": "#C9BEFB", "pieStrokeColor": "#B23AF2", "pieOuterStrokeColor": "#B23AF2", "pieStrokeWidth": "2px", "pieTitleTextColor": "#B23AF2"}}}%%
pie showData title Remaining Work by Priority (5h)
    "High — SME Review" : 3
    "Medium — PR Merge" : 1
    "Low — Publish" : 1
```

**Remaining hours by category (bar-style table):**

| Category | Hours | Bar |
|----------|-------|-----|
| SME technical review | 3 | ███████████████ |
| PR review & merge | 1 | █████ |
| Publish & notify | 1 | █████ |
| **Total** | **5** | |

> **Integrity:** the pie "Remaining Work" value (5) equals Section 1.2 Remaining Hours (5) and the Section 2.2 Hours sum (5).

---

## 8. Summary & Recommendations

**Achievements.** The project delivers a complete, runtime-grounded answer to how SimpleLogin's `RedisSessionStore` deserializes session data and how it behaves under corrupted, malformed, and malicious payloads. Every one of the five named sub-questions — plus the umbrella "how is it deserialized?" — is answered from directly observed runtime behavior, backed by 8 verbatim evidence blocks and 117 exact `file:line` citations. The central finding (a corrupted payload triggers a **silent reset** via `except Exception: pass`) and the security boundary (a *valid weaponized pickle* achieves RCE inside `pickle.loads` before the `except` can intervene) are demonstrated, not merely inferred. The Project Manager independently reproduced the canonical `test_login.py` suite (3/3, EXIT 0) and the silent-reset behavior.

**Remaining gaps.** None in the AAP scope. The outstanding 5 hours are human path-to-production gates: SME review, PR merge, and publication.

**Critical path to production.** SME technical review → PR merge → publish to the security knowledge base. This is a documentation artifact, so "production" means an accepted, published, authoritative internal reference.

**Production readiness.** The deliverable is **production-ready pending human review**. It is complete, committed, byte-accurate, and the repository is pristine (read-only scope fully honored — only the one document was added).

**Success metrics.**

| Metric | Target | Actual | Status |
|--------|--------|--------|--------|
| AAP sub-questions answered | 5/5 (+umbrella) | 6/6 | ✅ |
| `file:line` citations verified | 100% | 117/117 exact | ✅ |
| Canonical tests passing | 3/3 | 3/3 (EXIT 0) | ✅ |
| Source files modified | 0 | 0 | ✅ |
| Placeholders/TODO in deliverable | 0 | 0 | ✅ |
| AAP-scoped completion | ~100% of AAP work | 100% of AAP items; 90.9% incl. human gates | ✅ |

**Recommendation.** Accept the deliverable after SME review and merge. Separately, open a remediation ticket for the documented CWE-502 finding and harden Redis operationally — both explicitly outside this task's read-only scope.

The project is **90.9% complete**; the remaining 9.1% is human review, merge, and publication.

---

## 9. Development Guide

> Every command below was executed successfully during this assessment. Run from the repository root:
> `/tmp/blitzy/app/blitzy-69b57f9c-a60d-44e2-934e-aba188d82f39_8eee3e`

### 9.1 System Prerequisites

- **Python 3.10** (canonical; `pyproject.toml` pins `python = "^3.10"`; project venv is 3.10.20)
- **Poetry** (dependencies honor `poetry.lock`, 177 pins — unchanged; read-only task)
- **PostgreSQL** reachable at `localhost:15432` (per `tests/test.env`)
- **Redis** reachable at `127.0.0.1:6379` (`redis://localhost`)
- **Git + Git LFS**; Linux OS
- Canonical container (optional, authoritative): `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` — its `/app` is source commit `2cd6ee77`

### 9.2 Environment Setup

```bash
# Select the canonical env file (sets FLASK_SECRET, DB_URI, and MEM_STORE_URI)
export CONFIG="$(pwd)/tests/test.env"

# Confirm the activation variable is present — this is what enables RedisSessionStore
grep -E '^MEM_STORE_URI|^DB_URI|^FLASK_SECRET' tests/test.env
# DB_URI=postgresql://test:test@localhost:15432/test
# FLASK_SECRET=secret
# MEM_STORE_URI=redis://localhost

# Verify backing services are up
redis-cli -h 127.0.0.1 -p 6379 ping        # -> PONG
pg_isready -h localhost -p 15432           # -> accepting connections
```

### 9.3 Dependency Installation & Verification

```bash
# (Idempotent) install locked dependencies — read-only task makes no changes
poetry install

# Verify the session-critical dependencies at their exact locked versions
venv/bin/python -c "import flask,flask_login,itsdangerous,werkzeug,redis,limits; \
print(flask.__version__, flask_login.__version__, itsdangerous.__version__, \
werkzeug.__version__, redis.__version__, limits.__version__)"
# -> 1.1.2 0.5.0 1.1.0 1.0.1 4.6.0 1.5.1
```

### 9.4 Startup & Canonical Verification

```bash
# Idempotent DB check (alembic lives in the venv, not on bare PATH)
CONFIG="$(pwd)/tests/test.env" venv/bin/alembic upgrade head

# Run the canonical login/logout template the document cites (TESTED: 3 passed, EXIT 0)
CONFIG="$(pwd)/tests/test.env" GITHUB_ACTIONS_TEST=true PYTHONPATH="$(pwd)" \
  venv/bin/python -m pytest tests/auth/test_login.py -v
```

### 9.5 Verifying the Deliverable & Read-Only Scope

```bash
# The single deliverable (expect 1193 lines)
wc -l blitzy/documentation/app_2cd6ee777f8c.md

# Read-only scope: ONLY the doc was added since the base commit
git diff 2cd6ee77 --name-status         # -> A  blitzy/documentation/app_2cd6ee777f8c.md

# Working tree is clean
test -z "$(git status --porcelain)" && echo CLEAN || echo DIRTY   # -> CLEAN
```

### 9.6 Reading & Reproducing the Investigation

```bash
# One-paragraph answer
sed -n '1,14p' blitzy/documentation/app_2cd6ee777f8c.md

# Full section map
grep -nE '^#{1,3} ' blitzy/documentation/app_2cd6ee777f8c.md
```

To reproduce the central finding, run the Appendix A harness verbatim inside the canonical container (doc §1.3), or use the minimal read-only pattern: build the app via `create_app()` with `CONFIG=tests/test.env`, confirm `app.session_interface` is `RedisSessionStore`, write a valid pickle to `session:<sid>` (session restores), then overwrite it with garbage bytes (→ silent reset: new `uuid4` sid, empty session, no error). Always delete only the Redis keys you created.

### 9.7 Troubleshooting

- **`app.session_interface` is not `RedisSessionStore`** → `MEM_STORE_URI` is unset; ensure `CONFIG` points at `tests/test.env`. Without it, Flask falls back to the default `SecureCookieSessionInterface` (`server.py:163-165`).
- **PostgreSQL "connection refused"** → the cluster is on `5432`, not `15432`. `tests/test.env` pins `15432`; in the GHCR container move the cluster port `5432 → 15432` *before* starting it (doc §1.3).
- **`alembic: command not found`** → it is not on the bare PATH; use `venv/bin/alembic`.
- **18 `DeprecationWarning`s from pytest** → pre-existing third-party warnings (`flask_limiter`, `flask_admin`, `gnupg`); not errors — ignore.
- **GHCR container exits `126` / "cannot execute binary file" with `sleep infinity`** → the image `ENTRYPOINT` is `["/bin/bash"]`; override the entrypoint to `sleep` so the container stays alive (doc §1.3).

---

## 10. Appendices

### Appendix A — Command Reference

| Command | Purpose |
|---------|---------|
| `export CONFIG="$(pwd)/tests/test.env"` | Select canonical config (enables `RedisSessionStore`) |
| `redis-cli -h 127.0.0.1 -p 6379 ping` | Health-check Redis (→ `PONG`) |
| `pg_isready -h localhost -p 15432` | Health-check PostgreSQL |
| `poetry install` | Install locked dependencies (idempotent) |
| `CONFIG=... venv/bin/alembic upgrade head` | Idempotent DB head check |
| `CONFIG=... GITHUB_ACTIONS_TEST=true PYTHONPATH=$(pwd) venv/bin/python -m pytest tests/auth/test_login.py -v` | Run canonical login/logout tests |
| `git diff 2cd6ee77 --name-status` | Confirm read-only scope (only doc added) |
| `wc -l blitzy/documentation/app_2cd6ee777f8c.md` | Confirm deliverable size (1193) |
| `grep -nE '^#{1,3} ' blitzy/documentation/app_2cd6ee777f8c.md` | Print document section map |

### Appendix B — Port Reference

| Service | Port | Source |
|---------|------|--------|
| PostgreSQL | 15432 | `tests/test.env` `DB_URI` |
| Redis | 6379 | `redis://localhost` (`MEM_STORE_URI`) |
| App URL (config) | http://localhost | `tests/test.env` `URL` |

### Appendix C — Key File Locations

| Path | Role |
|------|------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The deliverable** (1,193 lines) |
| `app/session.py` | `RedisSessionStore`, `ServerSession`, the only `pickle.loads` (L76) / `pickle.dumps` (L91), `open_session` silent-reset (L68–80) |
| `app/redis_services.py` | The only `app.session_interface = RedisSessionStore(...)` assignment (L12/L17) |
| `server.py` | `secret_key` (L151), cookie config (L159–162), `MEM_STORE_URI` guard (L163–165), 7-day lifetime (L205–207) |
| `app/config.py` | `FLASK_SECRET` (L196–198), `SESSION_COOKIE_NAME="slapp"` (L199), `MEM_STORE_URI` (L568) |
| `app/extensions.py` | `login_manager.session_protection = "strong"` (L8) |
| `app/auth/views/logout.py` | `logout_session()` + cookie deletion (L9–16) |
| `tests/test.env` | Supplies `FLASK_SECRET`, `DB_URI`, `MEM_STORE_URI` (enables the store) |
| `tests/auth/test_login.py` | Canonical login/logout template (3 tests) |

### Appendix D — Technology Versions

| Component | Version | Source |
|-----------|---------|--------|
| Python | 3.10 (venv 3.10.20) | `pyproject.toml` `^3.10` |
| flask | 1.1.2 | `poetry.lock` |
| flask-login | 0.5.0 | `poetry.lock` |
| itsdangerous | 1.1.0 | `poetry.lock` |
| werkzeug | 1.0.1 | `poetry.lock` |
| redis | 4.6.0 | `poetry.lock` |
| limits | 1.5.1 | `poetry.lock` |
| flask-limiter | 1.4 | `poetry.lock` |
| alembic | 1.4.3 | venv |
| pickle | stdlib (protocol 4) | Python 3.10 (`cPickle` `ImportError` fallback, `app/session.py:9-12`) |

### Appendix E — Environment Variable Reference

| Variable | Value (test config) | Purpose |
|----------|---------------------|---------|
| `CONFIG` | `$(pwd)/tests/test.env` | Selects the env file loaded at import |
| `MEM_STORE_URI` | `redis://localhost` | **Activates** `RedisSessionStore` (`server.py:163-165`) |
| `DB_URI` | `postgresql://test:test@localhost:15432/test` | PostgreSQL connection |
| `FLASK_SECRET` | `secret` | Seeds `app.secret_key` → the session-id HMAC signer |
| `GITHUB_ACTIONS_TEST` | `true` | Test-harness flag |
| `PYTHONPATH` | `$(pwd)` | Import resolution for the harness |

### Appendix F — Developer Tools Guide

| Tool | Use |
|------|-----|
| `pytest` | Run the canonical `tests/auth/test_login.py` login/logout template |
| `redis-cli` | Inspect `session:<sid>` keys, TTLs, and payload bytes |
| `pg_isready` / `psql` | Verify / query PostgreSQL @ 15432 |
| `venv/bin/alembic` | Idempotent DB head check (in-venv only) |
| `git diff 2cd6ee77 --name-status` | Confirm read-only scope |
| `pickletools.dis` | Disassemble emitted pickle payloads (byte-verification) |

### Appendix G — Glossary

| Term | Meaning |
|------|---------|
| `RedisSessionStore` | SimpleLogin's custom Flask `SessionInterface` storing pickled session data in Redis |
| `ServerSession` | `CallbackDict`/`SessionMixin` subclass returned by `open_session` |
| Silent reset | `pickle.loads` raises → `except Exception: pass` → fresh `ServerSession(uuid4)`; no error surfaced |
| `slapp` | The session cookie name (`SESSION_COOKIE_NAME`) carrying the signed session id |
| `session:<sid>` | The Redis key holding the pickled payload |
| CWE-502 | Deserialization of Untrusted Data (the documented finding) |
| OWASP A08:2021 | Software and Data Integrity Failures |
| GHSA-g8c6-8fjj-2r4m | python-socketio Redis-pickle advisory (CVE-2025-61765), the close real-world analogue |
| `__reduce__` | Method a malicious pickle uses to return a callable executed during `loads` |

---

*Generated by the Blitzy Platform. Completion measured against the Agent Action Plan (AAP-scoped, PA1 methodology): 50 of 55 hours complete = 90.9%.*