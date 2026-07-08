# Custom Alias Creation — Signed-Suffix Validation & Creation-Limit Enforcement: A Run-Verified Investigation

> **Status:** Read-only investigation. This document is the *only* file written to the repository. No source file was modified, added, or deleted — verified by the final `git status --porcelain` and `git diff --name-status` in §F.7. The identified defect (tampered/garbage/empty suffixes reported as *expired*) is **documented, not fixed**, per the task constraint.

> **Evidence & redaction policy.** Every fenced `text` block below is verbatim, unedited runtime output captured from the real endpoints — full status line, complete response-header set, full JSON/HTML body, and the matching `"SL"` console log lines. The **only** substitution is that the value of the session cookie is shown as `slapp=<redacted>` (the cookie *structure* — `Domain`, `Expires`, `HttpOnly`, `Path`, `SameSite` — is preserved). A session cookie is a live credential (CWE-532); redacting only its value keeps the evidence complete while leaking no secret. No other bytes were altered, and no evidence-bearing content was elided with `…`.

---

## 0. The Question

The reported problem, verbatim intent:

> *"I am debugging intermittent validation failures during custom alias creation that don't match the expected behavior, which seems related to how signed suffixes are verified."*

Four explicit objectives, each answered below **from observed runtime output** (not code-reading alone):

- **O1 — Invalid / expired signed suffixes:** the exact HTTP status codes, error messages, and validation log entries produced. → §A
- **O2 — Rate-limiting headers:** whether they appear in responses, and their values. → §B
- **O3 — Successful creation:** what quota-related checks occur and what values get logged when the system verifies whether the user can create more aliases. → §C
- **O4 — Execution-path trace:** which component validates signed suffixes, which enforces creation limits, and what conditions trigger rejection. → §D
- **Plus — Intermittency:** *why* the failures are intermittent, attributed to concrete code-level causes. → §E

Constraint (verbatim): *"Don't modify the repository code — temporary test scripts or API calls are fine but keep the codebase unchanged."*

---

## TL;DR — Headline Findings

1. **The "failures that don't match expected behavior" are a mislabel bug.** `check_suffix_signature` (`app/alias_suffix.py:37-42`) wraps `signer.unsign(signed_suffix, max_age=600)` in `try / except itsdangerous.BadSignature: return None`. Because the `itsdangerous 1.1.0` exception hierarchy is `SignatureExpired ⊂ BadTimeSignature ⊂ BadSignature ⊂ BadData` (verified at runtime, §A.2), **every** failure mode — expired, tampered, garbage, empty — is caught and collapsed to `None`. The endpoint's `if not alias_suffix:` branch (`app/api/views/new_custom_alias.py:71`) then fires for *all* of them, returning **`412 {"error":"Alias creation time is expired, please retry"}`** and logging `LOG.w("Alias creation time expired for %s")` at `new_custom_alias.py:72` (v2) / `:187` (v3). The adjacent `except Exception: … return jsonify(error="Tampered suffix"), 400` branch (`new_custom_alias.py:74-76`) is **effectively dead for ordinary bad tokens**: a tampered suffix is reported as *expired*. This is confirmed end-to-end against the live **v2 and v3** endpoints (§A.4).

2. **No rate-limit headers are emitted on any response.** Neither `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, nor `Retry-After` appear on a `201`, `4xx`, or any of the three distinct `429`s. Root cause: the limiter is built as `Limiter(key_func=__key_func)` with no `headers_enabled` (`app/extensions.py:23`), and Flask-Limiter 1.4's default `RATELIMIT_HEADERS_ENABLED` is `False`. A programmatic header scan confirms the absence explicitly (§B).

3. **Four independent creation-limit layers guard the endpoint**, in this order: Flask-Limiter `@limiter.limit(ALIAS_LIMIT)` → the `parallel_limiter` concurrency lock → the `can_create_new_alias()` count quota → the `Alias.create()` per-user token bucket. Two of them (concurrency, token bucket) are silently disabled when Redis is unreachable. (§D, §E)

4. **The intermittency is code-level, not environmental**: the 600-second suffix-validity window, the concurrency `429` under double submission, the Flask-Limiter `5/minute` rolling burst window, and the Redis-availability gate. (§E)

---

## Runtime & Methodology (RUN-FIRST)

This investigation was performed **by building and running the real code paths** and capturing actual output; the answers below are written from that output. The real API entry points `POST /api/v2/alias/custom/new` and `POST /api/v3/alias/custom/new` were driven directly; valid signed suffixes were minted exactly the way the application mints them (`signer.sign(...)`, `app/alias_suffix.py:11`; the same `get_alias_suffixes()` used by `GET /api/v4|v5/alias/options`, `app/api/views/alias_options.py:69,140`), never via a debug/bypass hook.

**Intermittency methodology (identical input).** Because the report concerns *intermittent* failures, each condition was driven with the **same unchanged request** and re-submitted, and every condition was run in **two independent cross-process runs** (RUN1, RUN2, distinct users). For the suffix-content conditions the *exact same signed-suffix string* was POSTed twice (labeled `IDENTICAL signed_suffix (posted twice)` in the output); for the duplicate and quota conditions the *same payload* was replayed so the observable change is purely the backend state, not the input. The observed outcome distribution is reported as a distribution, never reduced to a single "representative" result.

All temporary observation scripts lived **outside** the repository (host `/tmp/probes/…`) and were streamed into the app container over stdin (`docker exec -i … python -`); none was ever written under the repo tree. They were removed on completion (§F.7).

### Canonical runtime

The stack was stood up exactly as `scripts/run-test.sh` prescribes, with Redis additionally reachable so all four limiter layers are in their active canonical state:

- **PostgreSQL 13** on host port `15432` (`scripts/run-test.sh:7`); database already migrated (`CONFIG=tests/test.env alembic upgrade head`, `scripts/run-test.sh:13`).
- **Redis** reachable at `redis://localhost` (`tests/test.env:78`), so `rate_limiter` and `parallel_limiter` are wired (`server.py:163-165` → `app/redis_services.py:9-11`).
- **Interpreter & dependencies** — the exact versions locked in `poetry.lock`, confirmed at runtime:

```text
=== INTERPRETER & DEPENDENCY VERSIONS ===
python           : 3.10.18
itsdangerous     : 1.1.0
flask            : 1.1.2
werkzeug         : 1.0.1
limits           : 1.5.1
flask_limiter    : 1.4
```

### Effective configuration (canonical `tests/test.env` + `app/config.py` defaults)

Confirmed at runtime and stable across runs:

```text
=== EFFECTIVE CONFIG (loaded from tests/test.env) ===
EMAIL_DOMAIN            : sl.local
MAX_NB_EMAIL_FREE_PLAN  : 3
MEM_STORE_URI           : redis://localhost
CUSTOM_ALIAS_SECRET     : 'secretcustom_alias'
ALIAS_LIMIT             : '100/day;50/hour;5/minute'
ALIAS_CREATE_RATE_LIMIT_FREE: [(10, 900), (50, 3600)]
ALIAS_CREATE_RATE_LIMIT_PAID: [(50, 900), (200, 3600)]
DISABLE_RATE_LIMIT      : False
DISABLE_ALIAS_SUFFIX    : False
```

Notes:
- `MAX_NB_EMAIL_FREE_PLAN=3` comes from `tests/test.env:13` (the code default is `5` at `app/config.py:121-124`).
- `CUSTOM_ALIAS_SECRET = FLASK_SECRET + "custom_alias"` = `"secret" + "custom_alias"` = `"secretcustom_alias"` (`app/config.py:201`, `FLASK_SECRET="secret"` at `tests/test.env:20`).
- `DISABLE_RATE_LIMIT` and `DISABLE_ALIAS_SUFFIX` are **unset** (`False`), so both the limiters and suffix signing are active.

The reproduction driver mirrored `tests/conftest.py`: `create_app()`, `TESTING=True`, `WTF_CSRF_ENABLED=False`, `SERVER_NAME="sl.test"`, `add_sl_domains()` + `add_proton_partner()`, and a cookie session established via the `login()` helper pattern (`tests/utils.py:46-59`). With a cookie session, `@require_api_auth` authorizes through its `current_user` fallback (`app/api/base.py:20-27`) and the Flask-Limiter key becomes `userid:{id}` (`app/extensions.py:14-19`). The exact commands and full unedited output blocks are in §F.

At bootstrap the driver confirmed the limiter storage is live and the flags are canonical:

```text
=== LIMITER STORAGE LIVENESS (after create_app wiring) ===
>>> lock_redis(parallel)=<limits.storage.RedisStorage object at 0x785771c6f910>
>>> lock_redis(bucket)  =<limits.storage.RedisStorage object at 0x785771c6f910>
>>> DISABLE_RATE_LIMIT=False DISABLE_ALIAS_SUFFIX=False MAX_NB_EMAIL_FREE_PLAN=3
```

---

## §A — O1: Invalid / Expired Signed-Suffix Behavior

**Answer (summary):** An **expired** suffix yields `412 {"error":"Alias creation time is expired, please retry"}` with `LOG.w("Alias creation time expired for %s")` at `app/api/views/new_custom_alias.py:72` (v2) / `:187` (v3). A **tampered**, **garbage**, or **empty** suffix yields the **same** `412` and the **same** log line — *not* the `400 {"error":"Tampered suffix"}` that the code's structure appears to intend. This uniform collapse is the reported "validation failures that don't match the expected behavior," and it was observed on **both** the v2 and v3 endpoints.

### A.1 The validating component and the 600-second window

Signed suffixes are minted and verified by a single module-level signer (`app/alias_suffix.py:11`):

```python
signer = itsdangerous.TimestampSigner(config.CUSTOM_ALIAS_SECRET)
```

The verifier is `check_suffix_signature` (`app/alias_suffix.py:37-42`) — the pivotal six lines:

```python
def check_suffix_signature(signed_suffix: str) -> Optional[str]:
    # hypothesis: user will click on the button in the 600 secs
    try:
        return signer.unsign(signed_suffix, max_age=600).decode()
    except itsdangerous.BadSignature:
        return None
```

Validity is bounded to `max_age=600` seconds (10 minutes). The endpoint consumes the result (`app/api/views/new_custom_alias.py:69-76`, v2):

```python
    try:
        alias_suffix = check_suffix_signature(signed_suffix)
        if not alias_suffix:
            LOG.w("Alias creation time expired for %s", user)
            return jsonify(error="Alias creation time is expired, please retry"), 412
    except Exception:
        LOG.w("Alias suffix is tampered, user %s", user)
        return jsonify(error="Tampered suffix"), 400
```

The v3 handler contains the byte-for-byte same construct, logging at `new_custom_alias.py:187` and returning `412` at `:188`, with the dead `400 "Tampered suffix"` branch at `:190-191`.

### A.2 The `itsdangerous` exception hierarchy (runtime-verified)

The reason the two branches collapse into one is the exception hierarchy. Verified at runtime against the actual installed `itsdangerous 1.1.0`:

```text
=== ITSDANGEROUS EXCEPTION HIERARCHY (issubclass) ===
SignatureExpired  subclass of BadTimeSignature: True
BadTimeSignature  subclass of BadSignature    : True
SignatureExpired  subclass of BadSignature    : True
BadSignature      subclass of BadData         : True
```

So `SignatureExpired ⊂ BadTimeSignature ⊂ BadSignature ⊂ BadData`. This is corroborated by the `itsdangerous` documentation (the `SignatureExpired`/`BadTimeSignature`/`BadSignature` inheritance chain). Because `check_suffix_signature` catches the **base** class `itsdangerous.BadSignature`, it swallows *both* the timestamp-expiry error (`SignatureExpired`) *and* the signature-mismatch/format errors (`BadTimeSignature`, `BadSignature`) — returning `None` in every failing case, and **never re-raising**. The endpoint's surrounding `except Exception:` therefore never sees an exception for ordinary bad tokens.

### A.3 The collapse, observed directly

Driving the real application signer, `check_suffix_signature` returns the decoded value for a valid token and `None` for *every* failure mode:

```text
=== RAW check_suffix_signature COLLAPSE (real app signer) ===
valid    -> '.probeword@sl.local'
expired  -> None
tampered -> None
garbage  -> None
empty    -> None
```

The distinct underlying exceptions that `unsign(..., max_age=600)` would have raised (all subclasses of `BadSignature`, hence all swallowed) are:

```text
=== RAW unsign(max_age=600) exception types ===
expired  -> SignatureExpired: Signature age 700 > 600 seconds
tampered -> BadTimeSignature: Signature b'2EY5owgTRKj671Ta5ZtYuTkzwtA' does not match
garbage  -> BadSignature: No b'.' found in value
empty    -> BadSignature: No b'.' found in value
```

The expired token here was produced without waiting 600 s by patching the time source `itsdangerous` reads while signing (`mock.patch("itsdangerous.timed.time.time", lambda: real()-700)`), then unsigning now — a faithful expiry, since the endpoint's own `unsign(..., max_age=600)` raises `SignatureExpired: Signature age 700 > 600 seconds`.

### A.4 End-to-end evidence against the live endpoints (v2 **and** v3, identical input ×2)

Each block below is the complete, unedited output for **one unchanged request submitted twice**. The v2 and v3 handlers are exercised separately for every invalid case so the collapse is shown observed — not inferred — on both.

**Condition 2 — Expired suffix (age 700 s > 600 s) → `412` on both endpoints.** The *identical* signed-suffix string is POSTed twice; both attempts return the same `412` and the same `:72`/`:187` log line.

*v2:*
```text
===== [RUN1] CONDITION 2 — expired suffix (age 700s>600s) (v2); SAME request x2 =====
user=user_37dsf2xdwu@mailbox.test id=943
IDENTICAL signed_suffix (posted twice): .expword@sl.local.ak3pNg.65ms-3EYGFx0rQqKEhNBP5YhNEY
----- v2 expired attempt 1 (identical input) -----
2026-07-08 06:19:30,248 - SL - WARNING - 13902 - "/code/app/api/views/new_custom_alias.py:72" - new_custom_alias_v2() -  - Alias creation time expired for <User 943 Test User user_37dsf2xdwu@mailbox.test>
2026-07-08 06:19:30,248 - SL - DEBUG - 13902 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 412, takes 0.012840986251831055
STATUS-LINE: 412 PRECONDITION FAILED | status_code: 412
RESPONSE-HEADERS:
    Content-Type: application/json
    Content-Length: 57
    Access-Control-Allow-Origin: *
    Set-Cookie: slapp=<redacted>; Domain=.sl.test; Expires=Wed, 15-Jul-2026 06:19:30 GMT; HttpOnly; Path=/; SameSite=Lax
JSON-BODY: {"error":"Alias creation time is expired, please retry"}

----- v2 expired attempt 2 (identical input) -----
2026-07-08 06:19:30,260 - SL - WARNING - 13902 - "/code/app/api/views/new_custom_alias.py:72" - new_custom_alias_v2() -  - Alias creation time expired for <User 943 Test User user_37dsf2xdwu@mailbox.test>
2026-07-08 06:19:30,261 - SL - DEBUG - 13902 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 412, takes 0.010485649108886719
STATUS-LINE: 412 PRECONDITION FAILED | status_code: 412
RESPONSE-HEADERS:
    Content-Type: application/json
    Content-Length: 57
    Access-Control-Allow-Origin: *
    Set-Cookie: slapp=<redacted>; Domain=.sl.test; Expires=Wed, 15-Jul-2026 06:19:30 GMT; HttpOnly; Path=/; SameSite=Lax
JSON-BODY: {"error":"Alias creation time is expired, please retry"}
```

*v3:*
```text
===== [RUN1] CONDITION 2 — expired suffix (age 700s>600s) (v3); SAME request x2 =====
user=user_mkijyohtc7@mailbox.test id=944
IDENTICAL signed_suffix (posted twice): .expword@sl.local.ak3pNg.65ms-3EYGFx0rQqKEhNBP5YhNEY
----- v3 expired attempt 1 (identical input) -----
2026-07-08 06:19:30,844 - SL - WARNING - 13902 - "/code/app/api/views/new_custom_alias.py:187" - new_custom_alias_v3() -  - Alias creation time expired for <User 944 Test User user_mkijyohtc7@mailbox.test>
2026-07-08 06:19:30,845 - SL - DEBUG - 13902 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v3/alias/custom/new ImmutableMultiDict([]) 412, takes 0.011498212814331055
STATUS-LINE: 412 PRECONDITION FAILED | status_code: 412
RESPONSE-HEADERS:
    Content-Type: application/json
    Content-Length: 57
    Access-Control-Allow-Origin: *
    Set-Cookie: slapp=<redacted>; Domain=.sl.test; Expires=Wed, 15-Jul-2026 06:19:30 GMT; HttpOnly; Path=/; SameSite=Lax
JSON-BODY: {"error":"Alias creation time is expired, please retry"}

----- v3 expired attempt 2 (identical input) -----
2026-07-08 06:19:30,857 - SL - WARNING - 13902 - "/code/app/api/views/new_custom_alias.py:187" - new_custom_alias_v3() -  - Alias creation time expired for <User 944 Test User user_mkijyohtc7@mailbox.test>
2026-07-08 06:19:30,857 - SL - DEBUG - 13902 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v3/alias/custom/new ImmutableMultiDict([]) 412, takes 0.01055598258972168
STATUS-LINE: 412 PRECONDITION FAILED | status_code: 412
RESPONSE-HEADERS:
    Content-Type: application/json
    Content-Length: 57
    Access-Control-Allow-Origin: *
    Set-Cookie: slapp=<redacted>; Domain=.sl.test; Expires=Wed, 15-Jul-2026 06:19:30 GMT; HttpOnly; Path=/; SameSite=Lax
JSON-BODY: {"error":"Alias creation time is expired, please retry"}
```

**Condition 3 — Tampered suffix (mutate the last character) → `412` (the headline anomaly), both endpoints.** `raw check_suffix_signature(tampered) = None` is printed inline, confirming the collapse is what drives the `412`; the status is **never** `400 "Tampered suffix"`.

*v2:*
```text
===== [RUN1] CONDITION 3 — tampered suffix (last char mutated) (v2); SAME request x2 =====
user=user_hq35tkbfl5@mailbox.test id=945
valid   : .tampword@sl.local.ak3r8w.6bcXBX2OF1R7sO9MrIjU9BZIrmM
tampered: .tampword@sl.local.ak3r8w.6bcXBX2OF1R7sO9MrIjU9BZIrmA
raw check_suffix_signature(tampered) = None
----- v2 tampered attempt 1 (identical input) -----
2026-07-08 06:19:31,482 - SL - WARNING - 13902 - "/code/app/api/views/new_custom_alias.py:72" - new_custom_alias_v2() -  - Alias creation time expired for <User 945 Test User user_hq35tkbfl5@mailbox.test>
2026-07-08 06:19:31,483 - SL - DEBUG - 13902 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 412, takes 0.017936229705810547
STATUS-LINE: 412 PRECONDITION FAILED | status_code: 412
RESPONSE-HEADERS:
    Content-Type: application/json
    Content-Length: 57
    Access-Control-Allow-Origin: *
    Set-Cookie: slapp=<redacted>; Domain=.sl.test; Expires=Wed, 15-Jul-2026 06:19:31 GMT; HttpOnly; Path=/; SameSite=Lax
JSON-BODY: {"error":"Alias creation time is expired, please retry"}

----- v2 tampered attempt 2 (identical input) -----
2026-07-08 06:19:31,503 - SL - WARNING - 13902 - "/code/app/api/views/new_custom_alias.py:72" - new_custom_alias_v2() -  - Alias creation time expired for <User 945 Test User user_hq35tkbfl5@mailbox.test>
2026-07-08 06:19:31,504 - SL - DEBUG - 13902 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 412, takes 0.017499685287475586
STATUS-LINE: 412 PRECONDITION FAILED | status_code: 412
RESPONSE-HEADERS:
    Content-Type: application/json
    Content-Length: 57
    Access-Control-Allow-Origin: *
    Set-Cookie: slapp=<redacted>; Domain=.sl.test; Expires=Wed, 15-Jul-2026 06:19:31 GMT; HttpOnly; Path=/; SameSite=Lax
JSON-BODY: {"error":"Alias creation time is expired, please retry"}
```

*v3:*
```text
===== [RUN1] CONDITION 3 — tampered suffix (last char mutated) (v3); SAME request x2 =====
user=user_8wd3ojr2zf@mailbox.test id=946
valid   : .tampword@sl.local.ak3r9A.9CCH1MjMR7r6_9rERV1ZKz9GIec
tampered: .tampword@sl.local.ak3r9A.9CCH1MjMR7r6_9rERV1ZKz9GIeA
raw check_suffix_signature(tampered) = None
----- v3 tampered attempt 1 (identical input) -----
2026-07-08 06:19:32,150 - SL - WARNING - 13902 - "/code/app/api/views/new_custom_alias.py:187" - new_custom_alias_v3() -  - Alias creation time expired for <User 946 Test User user_8wd3ojr2zf@mailbox.test>
2026-07-08 06:19:32,151 - SL - DEBUG - 13902 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v3/alias/custom/new ImmutableMultiDict([]) 412, takes 0.018873929977416992
STATUS-LINE: 412 PRECONDITION FAILED | status_code: 412
RESPONSE-HEADERS:
    Content-Type: application/json
    Content-Length: 57
    Access-Control-Allow-Origin: *
    Set-Cookie: slapp=<redacted>; Domain=.sl.test; Expires=Wed, 15-Jul-2026 06:19:32 GMT; HttpOnly; Path=/; SameSite=Lax
JSON-BODY: {"error":"Alias creation time is expired, please retry"}

----- v3 tampered attempt 2 (identical input) -----
2026-07-08 06:19:32,172 - SL - WARNING - 13902 - "/code/app/api/views/new_custom_alias.py:187" - new_custom_alias_v3() -  - Alias creation time expired for <User 946 Test User user_8wd3ojr2zf@mailbox.test>
2026-07-08 06:19:32,173 - SL - DEBUG - 13902 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v3/alias/custom/new ImmutableMultiDict([]) 412, takes 0.019235610961914062
STATUS-LINE: 412 PRECONDITION FAILED | status_code: 412
RESPONSE-HEADERS:
    Content-Type: application/json
    Content-Length: 57
    Access-Control-Allow-Origin: *
    Set-Cookie: slapp=<redacted>; Domain=.sl.test; Expires=Wed, 15-Jul-2026 06:19:32 GMT; HttpOnly; Path=/; SameSite=Lax
JSON-BODY: {"error":"Alias creation time is expired, please retry"}
```

**Condition 4 — Garbage (`"garbage"`) and empty (`""`) suffix → `412`, both endpoints, identical input ×2.**

*garbage, v2:*
```text
===== [RUN1] CONDITION 4 — garbage suffix (v2); SAME request x2 =====
user=user_hbwiqs48yr@mailbox.test id=947 signed_suffix='garbage'
----- v2 garbage attempt 1 (identical input) -----
2026-07-08 06:19:32,822 - SL - WARNING - 13902 - "/code/app/api/views/new_custom_alias.py:72" - new_custom_alias_v2() -  - Alias creation time expired for <User 947 Test User user_hbwiqs48yr@mailbox.test>
2026-07-08 06:19:32,823 - SL - DEBUG - 13902 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 412, takes 0.013423681259155273
STATUS-LINE: 412 PRECONDITION FAILED | status_code: 412
RESPONSE-HEADERS:
    Content-Type: application/json
    Content-Length: 57
    Access-Control-Allow-Origin: *
    Set-Cookie: slapp=<redacted>; Domain=.sl.test; Expires=Wed, 15-Jul-2026 06:19:32 GMT; HttpOnly; Path=/; SameSite=Lax
JSON-BODY: {"error":"Alias creation time is expired, please retry"}

----- v2 garbage attempt 2 (identical input) -----
2026-07-08 06:19:32,836 - SL - WARNING - 13902 - "/code/app/api/views/new_custom_alias.py:72" - new_custom_alias_v2() -  - Alias creation time expired for <User 947 Test User user_hbwiqs48yr@mailbox.test>
2026-07-08 06:19:32,837 - SL - DEBUG - 13902 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 412, takes 0.0117034912109375
STATUS-LINE: 412 PRECONDITION FAILED | status_code: 412
RESPONSE-HEADERS:
    Content-Type: application/json
    Content-Length: 57
    Access-Control-Allow-Origin: *
    Set-Cookie: slapp=<redacted>; Domain=.sl.test; Expires=Wed, 15-Jul-2026 06:19:32 GMT; HttpOnly; Path=/; SameSite=Lax
JSON-BODY: {"error":"Alias creation time is expired, please retry"}
```

*empty, v2:*
```text
===== [RUN1] CONDITION 4 — empty suffix (v2); SAME request x2 =====
user=user_1nxgkixmfv@mailbox.test id=948 signed_suffix=''
----- v2 empty attempt 1 (identical input) -----
2026-07-08 06:19:33,491 - SL - WARNING - 13902 - "/code/app/api/views/new_custom_alias.py:72" - new_custom_alias_v2() -  - Alias creation time expired for <User 948 Test User user_1nxgkixmfv@mailbox.test>
2026-07-08 06:19:33,492 - SL - DEBUG - 13902 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 412, takes 0.013547658920288086
STATUS-LINE: 412 PRECONDITION FAILED | status_code: 412
RESPONSE-HEADERS:
    Content-Type: application/json
    Content-Length: 57
    Access-Control-Allow-Origin: *
    Set-Cookie: slapp=<redacted>; Domain=.sl.test; Expires=Wed, 15-Jul-2026 06:19:33 GMT; HttpOnly; Path=/; SameSite=Lax
JSON-BODY: {"error":"Alias creation time is expired, please retry"}

----- v2 empty attempt 2 (identical input) -----
2026-07-08 06:19:33,510 - SL - WARNING - 13902 - "/code/app/api/views/new_custom_alias.py:72" - new_custom_alias_v2() -  - Alias creation time expired for <User 948 Test User user_1nxgkixmfv@mailbox.test>
2026-07-08 06:19:33,511 - SL - DEBUG - 13902 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 412, takes 0.016704559326171875
STATUS-LINE: 412 PRECONDITION FAILED | status_code: 412
RESPONSE-HEADERS:
    Content-Type: application/json
    Content-Length: 57
    Access-Control-Allow-Origin: *
    Set-Cookie: slapp=<redacted>; Domain=.sl.test; Expires=Wed, 15-Jul-2026 06:19:33 GMT; HttpOnly; Path=/; SameSite=Lax
JSON-BODY: {"error":"Alias creation time is expired, please retry"}
```

*garbage, v3:*
```text
===== [RUN1] CONDITION 4 — garbage suffix (v3); SAME request x2 =====
user=user_325eldm0r1@mailbox.test id=949 signed_suffix='garbage'
----- v3 garbage attempt 1 (identical input) -----
2026-07-08 06:19:34,178 - SL - WARNING - 13902 - "/code/app/api/views/new_custom_alias.py:187" - new_custom_alias_v3() -  - Alias creation time expired for <User 949 Test User user_325eldm0r1@mailbox.test>
2026-07-08 06:19:34,179 - SL - DEBUG - 13902 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v3/alias/custom/new ImmutableMultiDict([]) 412, takes 0.017559528350830078
STATUS-LINE: 412 PRECONDITION FAILED | status_code: 412
RESPONSE-HEADERS:
    Content-Type: application/json
    Content-Length: 57
    Access-Control-Allow-Origin: *
    Set-Cookie: slapp=<redacted>; Domain=.sl.test; Expires=Wed, 15-Jul-2026 06:19:34 GMT; HttpOnly; Path=/; SameSite=Lax
JSON-BODY: {"error":"Alias creation time is expired, please retry"}

----- v3 garbage attempt 2 (identical input) -----
2026-07-08 06:19:34,200 - SL - WARNING - 13902 - "/code/app/api/views/new_custom_alias.py:187" - new_custom_alias_v3() -  - Alias creation time expired for <User 949 Test User user_325eldm0r1@mailbox.test>
2026-07-08 06:19:34,201 - SL - DEBUG - 13902 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v3/alias/custom/new ImmutableMultiDict([]) 412, takes 0.01827216148376465
STATUS-LINE: 412 PRECONDITION FAILED | status_code: 412
RESPONSE-HEADERS:
    Content-Type: application/json
    Content-Length: 57
    Access-Control-Allow-Origin: *
    Set-Cookie: slapp=<redacted>; Domain=.sl.test; Expires=Wed, 15-Jul-2026 06:19:34 GMT; HttpOnly; Path=/; SameSite=Lax
JSON-BODY: {"error":"Alias creation time is expired, please retry"}
```

*empty, v3:*
```text
===== [RUN1] CONDITION 4 — empty suffix (v3); SAME request x2 =====
user=user_ouod8oxy9u@mailbox.test id=950 signed_suffix=''
----- v3 empty attempt 1 (identical input) -----
2026-07-08 06:19:34,861 - SL - WARNING - 13902 - "/code/app/api/views/new_custom_alias.py:187" - new_custom_alias_v3() -  - Alias creation time expired for <User 950 Test User user_ouod8oxy9u@mailbox.test>
2026-07-08 06:19:34,862 - SL - DEBUG - 13902 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v3/alias/custom/new ImmutableMultiDict([]) 412, takes 0.020219087600708008
STATUS-LINE: 412 PRECONDITION FAILED | status_code: 412
RESPONSE-HEADERS:
    Content-Type: application/json
    Content-Length: 57
    Access-Control-Allow-Origin: *
    Set-Cookie: slapp=<redacted>; Domain=.sl.test; Expires=Wed, 15-Jul-2026 06:19:34 GMT; HttpOnly; Path=/; SameSite=Lax
JSON-BODY: {"error":"Alias creation time is expired, please retry"}

----- v3 empty attempt 2 (identical input) -----
2026-07-08 06:19:34,883 - SL - WARNING - 13902 - "/code/app/api/views/new_custom_alias.py:187" - new_custom_alias_v3() -  - Alias creation time expired for <User 950 Test User user_ouod8oxy9u@mailbox.test>
2026-07-08 06:19:34,884 - SL - DEBUG - 13902 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v3/alias/custom/new ImmutableMultiDict([]) 412, takes 0.0183870792388916
STATUS-LINE: 412 PRECONDITION FAILED | status_code: 412
RESPONSE-HEADERS:
    Content-Type: application/json
    Content-Length: 57
    Access-Control-Allow-Origin: *
    Set-Cookie: slapp=<redacted>; Domain=.sl.test; Expires=Wed, 15-Jul-2026 06:19:34 GMT; HttpOnly; Path=/; SameSite=Lax
JSON-BODY: {"error":"Alias creation time is expired, please retry"}
```

### A.5 Conclusion for O1 (the mislabel — documented, not fixed)

| Input | Underlying `unsign` exception | `check_suffix_signature` returns | Observed HTTP | Observed log line (v2 / v3) |
|-------|-------------------------------|----------------------------------|---------------|-------------------|
| Valid, fresh | (none) | decoded suffix | `201` | audit "New alias created" |
| **Expired** (>600 s) | `SignatureExpired` | `None` | `412` "…expired, please retry" | `LOG.w` `new_custom_alias.py:72` / `:187` |
| **Tampered** | `BadTimeSignature` | `None` | `412` "…expired, please retry" | `LOG.w` `new_custom_alias.py:72` / `:187` |
| **Garbage** | `BadSignature` | `None` | `412` "…expired, please retry" | `LOG.w` `new_custom_alias.py:72` / `:187` |
| **Empty** | `BadSignature` | `None` | `412` "…expired, please retry" | `LOG.w` `new_custom_alias.py:72` / `:187` |

**Rationale / root cause:** the `400 {"error":"Tampered suffix"}` path at `new_custom_alias.py:74-76` (v2) / `:190-191` (v3) can only be reached if `check_suffix_signature` *raises* an exception. It never does for a `BadSignature` subclass, because it catches that base class internally and returns `None` (`app/alias_suffix.py:41-42`). Therefore, for every ordinary bad token, control takes the `if not alias_suffix:` branch and returns `412 "expired"`. A developer who submits a tampered or malformed suffix and expects a "tampered"/"bad signature" style error instead sees "Alias creation time is expired, please retry" — behavior that does not match the apparent intent of the code. Per the task constraint, this is **documented, not remediated**. The web-UI sibling `app/dashboard/views/custom_alias.py` shares the same `check_suffix_signature` (`:90`) and `flash(...)`es the same "Alias creation time is expired, please retry" message (`:93`), so it exhibits the identical collapse in the dashboard form flow (noted as a secondary sibling; the question is API-focused).

---


## §B — O2: Rate-Limiting Headers

**Answer:** **No rate-limit headers are present on any response.** There is no `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, or `Retry-After` on a `201`, on any of the `4xx` rejections, or on any of the three distinct `429` responses. The only response headers observed are `Content-Type`, `Content-Length`, `Access-Control-Allow-Origin: *`, and `Set-Cookie` (plus `Server`/`Date`/`Connection` when observed over real HTTP via gunicorn).

### B.1 Root cause

The Flask-Limiter instance is constructed with only a key function and **no header configuration** (`app/extensions.py:23`):

```python
limiter = Limiter(key_func=__key_func)
```

Flask-Limiter 1.4's default for `RATELIMIT_HEADERS_ENABLED` is `False`, and the `headers_enabled` constructor argument (unused here) is what would turn on the `X-RateLimit-*` headers — corroborated against the Flask-Limiter documentation. Since neither is set anywhere in the codebase, no rate-limit headers are written. Furthermore, SimpleLogin installs a **custom 429 error handler** (`server.py:362-372`) that, for any `/api/` path, returns `jsonify(error="Rate limit exceeded"), 429` (`server.py:370`) — a JSON body, not the Flask-Limiter default HTML page, and still with no rate-limit headers.

> **Observed correction to a common expectation:** because of this custom handler, the Flask-Limiter `429` carries a **JSON** body (`{"error":"Rate limit exceeded"}`), *not* HTML. All three `429` sources (Flask-Limiter, `parallel_limiter`, and the `Alias.create` token bucket) funnel through the same handler and therefore produce the identical 32-byte JSON body.

### B.2 The complete header sets (evidence)

The full, verbatim header set for each response class is embedded in the section that answers it; none carries any `X-RateLimit-*` or `Retry-After` header:

- **`201` success** (Condition 1, v2) — headers `Content-Type`, `Content-Length`, `Access-Control-Allow-Origin`, `Set-Cookie` only: see §C.3.
- **Token-bucket `429`** (Condition 8) — `FULL RESPONSE-HEADERS` block, same four keys: see §F.5.
- **`parallel_limiter` concurrency `429`** (Condition 9): see §E.2.
- **Flask-Limiter burst `429`**, captured over a **real gunicorn HTTP connection** via `curl -D -` (Condition 10) — only standard HTTP headers (`Server`, `Date`, `Connection`) plus the same four: see §E.3.

A programmatic check over the full test-client header set on a `201` success confirms the absence explicitly:

```text
===== O2 — explicit rate-limit-header scan (test client, 201 success) =====
2026-07-08 06:25:17,712 - SL - DEBUG - 14076 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04726076126098633
response status: 201
ALL response header keys: ['Access-Control-Allow-Origin', 'Content-Length', 'Content-Type', 'Set-Cookie']
X-RateLimit*/Retry-After headers present: NONE
```

**Conclusion for O2:** rate-limit headers are **absent on every response class** (`201`, `400`, `409`, `412`, and all three `429`s). Their values therefore cannot be reported because they are never emitted — a direct consequence of `Limiter(key_func=__key_func)` lacking `headers_enabled` (`app/extensions.py:23`) and the framework default `RATELIMIT_HEADERS_ENABLED=False`.

---


## §C — O3: Successful-Creation Quota Checks (and What Each Logs)

**Answer:** On the success path, two quota-related checks run in order, and a third audit record is written on creation:

1. **Count quota — `User.can_create_new_alias()`** (`app/models.py:867-884`). On success it **logs nothing**; only on breach does the *endpoint* log `LOG.d("user %s cannot create any custom alias")` (`new_custom_alias.py:49`) and return `400`.
2. **Per-user token bucket — inside `Alias.create()`** (`app/models.py:1628-1641`), via `rate_limiter.check_bucket_limit(...)` (`app/rate_limiter.py:19-42`). On success it **logs nothing**; only on breach does it `LOG.i("Rate limit hit for {lock_name} (bucket id {bucket_id}) -> {value}/{max_hits}")` (`app/rate_limiter.py:33`) and raise `429`.
3. **Success audit log** — `emit_alias_audit_log(new_alias, AliasAuditLogAction.CreateAlias, "New alias created")` (`app/models.py:1688-1689`).

So on a *purely successful* creation, the quota checks are **silent** — the only quota-relevant value they expose is a *state change* (the alias count increments), which is exactly what makes the failure boundary hard to see until the very request that trips it. The success itself, however, **is** recorded — in the `AliasAuditLog` table (observed below).

### C.1 The count-quota gate

`can_create_new_alias()` (`app/models.py:867-884`) returns `True`/`False`; `max_alias_for_free_account()` (`app/models.py:858-865`) reads `config.MAX_NB_EMAIL_FREE_PLAN`. For a genuine free account the decision is `Alias.filter_by(user_id=<id>).count() < MAX_NB_EMAIL_FREE_PLAN`.

**Observed state before/after each creation (Condition 7, `MAX_NB_EMAIL_FREE_PLAN=3`, one unchanged signed suffix reused across attempts):** a fresh non-partner user is auto-created *with one* "newsletter" alias inside `User.create` (`app/models.py:633-640`), so the starting count is **1** (observed, not assumed). The user can create 2 more; the 3rd custom attempt is refused:

```text
===== [RUN1] CONDITION 7 — count quota (v2); create up to MAX then one more =====
user=user_28mhl8zpyi@mailbox.test id=954
start count (includes auto newsletter alias) = 1
CONSTANT signed_suffix across attempts: .h9eea5@sl.local.ak3r-Q.iYNbLWT3anRgO6n0w96bxAqezDw
----- quota attempt 1 (count before=1) prefix=q1h9eea5 -----
2026-07-08 06:19:37,980 - SL - DEBUG - 13902 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.06008648872375488
STATUS-LINE: 201 CREATED | status_code: 201
RESPONSE-HEADERS:
    Content-Type: application/json
    Content-Length: 442
    Access-Control-Allow-Origin: *
    Set-Cookie: slapp=<redacted>; Domain=.sl.test; Expires=Wed, 15-Jul-2026 06:19:37 GMT; HttpOnly; Path=/; SameSite=Lax
JSON-BODY: {"alias":"q1h9eea5.h9eea5@sl.local","creation_date":"2026-07-08 06:19:37+00:00","creation_timestamp":1783491577,"disable_pgp":false,"email":"q1h9eea5.h9eea5@sl.local","enabled":true,"id":1722,"latest_activity":null,"mailbox":{"email":"user_28mhl8zpyi@mailbox.test","id":1113},"mailboxes":[{"email":"user_28mhl8zpyi@mailbox.test","id":1113}],"name":null,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null,"pinned":false,"support_pgp":false}

count after = 2
----- quota attempt 2 (count before=2) prefix=q2h9eea5 -----
2026-07-08 06:19:38,071 - SL - DEBUG - 13902 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.08147215843200684
STATUS-LINE: 201 CREATED | status_code: 201
RESPONSE-HEADERS:
    Content-Type: application/json
    Content-Length: 442
    Access-Control-Allow-Origin: *
    Set-Cookie: slapp=<redacted>; Domain=.sl.test; Expires=Wed, 15-Jul-2026 06:19:38 GMT; HttpOnly; Path=/; SameSite=Lax
JSON-BODY: {"alias":"q2h9eea5.h9eea5@sl.local","creation_date":"2026-07-08 06:19:38+00:00","creation_timestamp":1783491578,"disable_pgp":false,"email":"q2h9eea5.h9eea5@sl.local","enabled":true,"id":1723,"latest_activity":null,"mailbox":{"email":"user_28mhl8zpyi@mailbox.test","id":1113},"mailboxes":[{"email":"user_28mhl8zpyi@mailbox.test","id":1113}],"name":null,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null,"pinned":false,"support_pgp":false}

count after = 3
----- quota attempt 3 (count before=3) prefix=q3h9eea5 -----
2026-07-08 06:19:38,107 - SL - DEBUG - 13902 - "/code/app/api/views/new_custom_alias.py:49" - new_custom_alias_v2() -  - user <User 954 Test User user_28mhl8zpyi@mailbox.test> cannot create any custom alias
2026-07-08 06:19:38,108 - SL - DEBUG - 13902 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 400, takes 0.02290630340576172
STATUS-LINE: 400 BAD REQUEST | status_code: 400
RESPONSE-HEADERS:
    Content-Type: application/json
    Content-Length: 141
    Access-Control-Allow-Origin: *
    Set-Cookie: slapp=<redacted>; Domain=.sl.test; Expires=Wed, 15-Jul-2026 06:19:38 GMT; HttpOnly; Path=/; SameSite=Lax
JSON-BODY: {"error":"You have reached the limitation of a free account with the maximum of 3 aliases, please upgrade your plan to create more aliases"}

count after = 3
```

Observe: the two *successful* creations logged **nothing** quota-related (only the `after_request` access line, and the `count` went `1→2→3`); the `LOG.d "… cannot create any custom alias"` appears **only** on the refused 3rd attempt, at `new_custom_alias.py:49`. This is `can_create_new_alias()` returning `False` at `Alias…count()(3) < 3` → `False`.

### C.2 The per-user token bucket inside `Alias.create()`

For each `(hits, seconds)` in the applicable limit list, `Alias.create()` (`app/models.py:1641`) calls:

```python
rate_limiter.check_bucket_limit(f"alias_create_{seconds}d:{user.id}", hits, seconds)
```

The limit list depends on plan (`app/models.py:1635,1637`): **free** = `[(10, 900), (50, 3600)]`, **paid** = `[(50, 900), (200, 3600)]` (`app/config.py:554-558`). On success `check_bucket_limit` is silent; on breach it logs `LOG.i` (`app/rate_limiter.py:33`) and raises `429`.

> **Observed nuance that matters for "successful creation" accounting (a correction to a naive reading):** a fresh user is on a **7-day trial**, and `is_premium()` returns `True` during a trial. Consequently `Alias.create()` selects the **PAID** bucket `(50, 900)` for a brand-new user — even though `can_create_new_alias()` still uses the **free** count cap of `3` because that gate relies on `lifetime_or_active_subscription()`, which is `False` during a trial. **Net effect under the canonical `MAX_NB_EMAIL_FREE_PLAN=3`:** a free/trial user is refused by the **count quota** (`400`) at attempt 3 — *regardless of trial state* — so the token bucket is **never** the binding constraint for such a user. Trial expiry only flips which bucket `Alias.create()` *selects* (PAID `(50, 900)` → FREE `(10, 900)`); it does **not** lift the count-quota-at-3 barrier, because `can_create_new_alias()` returns `count < 3` whenever `lifetime_or_active_subscription()` is `False`, which holds for **both** a fresh trial and an expired trial (`app/models.py:867-884`). This was confirmed at runtime: under cap=3, both a fresh-trial user (PAID bucket selected) and an expired-trial user (FREE bucket selected) return `[201, 201, 400, …]` — the token bucket is reached in neither case. To exercise the token bucket in isolation, **Condition 8 (§F.5) deliberately raised `MAX_NB_EMAIL_FREE_PLAN` to `1000`** (disclosed there and in §F.1) so the count quota no longer masks it; both plan buckets were then observed to breach — the **PAID** bucket at `-> 51/50` and the **FREE** bucket at `-> 11/10`.

The bucket key stored in Redis is derived from `lock_name = alias_create_900d:{user.id}` and `bucket_id = int_time - (int_time % seconds)` (`app/rate_limiter.py:22-27`). On the success path the value simply increments; the logged breach values observed were `-> 51/50` (paid) and `-> 11/10` (free) — see §F.5, Condition 8. When Redis is unreachable, this check **no-ops** immediately (`app/rate_limiter.py:28`), so no bucket enforcement occurs (§E.4).

### C.3 The success audit log (observed `AliasAuditLog` record)

On a successful `201`, `Alias.create()` emits an audit action `"New alias created"` (`app/models.py:1688-1689`). This is not merely inferred from the source: after the `201` the driver queried the `AliasAuditLog` table for the created alias and printed the row. The `201` body is the full alias-info object; the `AUDIT-RECORD` line below is the persisted audit row for the same alias (`action='create'`, `message='New alias created'`):

```text
===== [RUN1] CONDITION 1 — valid fresh suffix (v2) =====
user=user_vq6xygeyyk@mailbox.test id=941 default_mailbox_id=1100 start_count=1
signed_suffix=.h95tuq@sl.local.ak3r8A.6yTffOpVtLLzwAe557C5pLKLwJE
2026-07-08 06:19:29,020 - SL - DEBUG - 13902 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.048174142837524414
STATUS-LINE: 201 CREATED | status_code: 201
RESPONSE-HEADERS:
    Content-Type: application/json
    Content-Length: 438
    Access-Control-Allow-Origin: *
    Set-Cookie: slapp=<redacted>; Domain=.sl.test; Expires=Wed, 15-Jul-2026 06:19:29 GMT; HttpOnly; Path=/; SameSite=Lax
JSON-BODY: {"alias":"prefix.h95tuq@sl.local","creation_date":"2026-07-08 06:19:29+00:00","creation_timestamp":1783491569,"disable_pgp":false,"email":"prefix.h95tuq@sl.local","enabled":true,"id":1705,"latest_activity":null,"mailbox":{"email":"user_vq6xygeyyk@mailbox.test","id":1100},"mailboxes":[{"email":"user_vq6xygeyyk@mailbox.test","id":1100}],"name":null,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null,"pinned":false,"support_pgp":false}

alias_count_after=2
--- AliasAuditLog query for created alias (proves success audit) ---
AUDIT-RECORD: id=1783 user_id=941 alias_id=1705 alias_email='prefix.h95tuq@sl.local' action='create' message='New alias created' created_at=2026-07-08T06:19:29.013134+00:00
```

`AliasAuditLogAction.CreateAlias` serializes to the string `"create"` (`app/alias_audit_log_utils.py`), which is exactly the `action='create'` observed in the queried row.

**Conclusion for O3:** the two quota checks that decide "can this user create more aliases" are `can_create_new_alias()` (count quota → `400` + `LOG.d` at `new_custom_alias.py:49` on breach, silent on success) and the `Alias.create()` token bucket (→ `429` + `LOG.i` at `rate_limiter.py:33` on breach, silent on success). On a successful creation neither logs a quota value; the only observable quota signal is the incrementing alias count (observed `1→2→3` in §C.1). The success itself is recorded via the `"New alias created"` audit action, observed as a real `AliasAuditLog` row above.

---


## §D — O4: Execution-Path Trace & Rejection Conditions

**Answer:** The signed suffix is validated by `check_suffix_signature` / `verify_prefix_suffix` (`app/alias_suffix.py:37-91`). Creation limits are enforced by **four independent layers**, applied as the ordered decorator/validation chain below. Each rejection maps to a specific status code, message, and (where present) log line.

### D.1 The ordered chain (v2, `app/api/views/new_custom_alias.py:28-112`)

The decorators are applied outermost-first at request time:

```python
@api_bp.route("/v2/alias/custom/new", methods=["POST"])   # L28
@limiter.limit(ALIAS_LIMIT)                                # L29  -> Flask-Limiter (layer 1)
@require_api_auth                                          # L30  -> auth (401/403)
@parallel_limiter.lock(name="alias_creation")             # L31  -> concurrency lock (layer 2)
def new_custom_alias_v2():
```

1. **Flask-Limiter `@limiter.limit(ALIAS_LIMIT)`** (`app/extensions.py:23`, `app/config.py:448`, `ALIAS_LIMIT="100/day;50/hour;5/minute"`) → `429` when any window is exceeded. This is the **outermost** layer, so it counts *every* request to the route.
2. **`@require_api_auth`** (`app/api/base.py:16-60`): `401 "Wrong api key"` (`base.py:20-27`), `403 "Disabled account"` (`base.py:36-37`). With a cookie session, authorization falls through to `g.user = current_user`.
3. **`@parallel_limiter.lock(name="alias_creation")`** (`app/parallel_limiter.py:19-73`): acquires a per-user Redis lock `cl:{current_user.id}:alias_creation` (`parallel_limiter.py:56`, TTL `max_wait_secs=5` at `:23`); if it is already held, `acquire_lock` raises `TooManyRequests` → `429` (`parallel_limiter.py:30-34`). No-ops when Redis is absent (`parallel_limiter.py:51-52`).
4. **`can_create_new_alias()` count quota** (`app/models.py:867-884`) → `400` quota message + `LOG.d` (`new_custom_alias.py:49`).
5. **`check_suffix_signature`** (`app/alias_suffix.py:37-42`) → `412` for `None` (expired/tampered/garbage/empty); the `400 "Tampered suffix"` branch is effectively dead (§A).
6. **`verify_prefix_suffix`** (`app/alias_suffix.py:45-91`) → `400 "wrong alias prefix or suffix"` (`new_custom_alias.py:78-79`); logs `LOG.e "wrong alias suffix …"` at `app/alias_suffix.py:61`.
7. **Duplicate check** (`new_custom_alias.py:82-88`) → `409 "alias … already exists"` + `LOG.d "full alias already used …"` (`:87`).
8. **`".." check`** (`new_custom_alias.py:90-94`) → `400`.
9. **`Alias.create()` token bucket** (`app/models.py:1641` → `app/rate_limiter.py:19-42`) → `429` + `LOG.i "Rate limit hit …"` (`rate_limiter.py:33`).
10. **Success** → `201` + audit `"New alias created"` (`new_custom_alias.py:109-112`, `models.py:1688-1689`).

### D.2 v3 differences (`app/api/views/new_custom_alias.py:115-235`) — observed

v3 carries the same decorator chain (`:116-118`) and the same suffix/quota/duplicate logic, but adds validation **before** the suffix check:
- request body must be a dict → `400` (`:153-154`);
- `check_alias_prefix` → `400 "alias prefix invalid format or too long"` (`:167-168`);
- `mailbox_ids` must be a list → `400 "mailbox_ids must be an array of id"` (`:171-172`); each must be owned & verified → `400 "Errors with Mailbox"` (`:174-178`); at least one required → `400 "At least one mailbox must be selected"` (`:180-181`).

The suffix branch (`:184-191`) is identical in effect to v2. This was **observed, not inferred**: the v3 endpoint was driven with `mailbox_ids=[default_mailbox_id]` for a valid suffix (`201`, Condition 1 v3 — `prefix.buldyq@sl.local`, id 1707, with its own `AliasAuditLog` row, §F.3), and for the expired, tampered, garbage, and empty suffixes (all `412` with `LOG.w` at `new_custom_alias.py:187`, §A.4), and for a duplicate payload (`201` then `409` with `LOG.d` at `new_custom_alias.py:202`, §F.4). The §A mislabel therefore applies equally to v3, confirmed against the live v3 handler.

### D.3 Dashboard sibling (secondary)

`app/dashboard/views/custom_alias.py` reuses `get_alias_suffixes` / `check_suffix_signature` / `verify_prefix_suffix` under the same `@limiter.limit(ALIAS_LIMIT, methods=["POST"])` (`:31`) and `@parallel_limiter.lock(name="alias_creation")` (`:33`). It is a **web-form** flow that `flash(...)`es "Alias creation time is expired, please retry" (`:90,93`) rather than returning JSON/`412`, so it shares the same collapse/mislabel. Noted as a sibling only — the question is API-focused.

### D.4 Rejection-condition table (all observed)

| Rejection | Component (file:line) | Status | Body / message | Log line |
|-----------|------------------------|--------|----------------|----------|
| Over Flask-Limiter window | `extensions.py:23`, `config.py:448` | `429` | `{"error":"Rate limit exceeded"}` (JSON via `server.py:362-372`) | `LOG.w server.py:364` |
| Missing/invalid API key | `api/base.py:20-27` | `401` | `Wrong api key` | — |
| Disabled account | `api/base.py:36-37` | `403` | `Disabled account` | — |
| Concurrency lock held | `parallel_limiter.py:30-34,56` | `429` | `{"error":"Rate limit exceeded"}` | `LOG.w server.py:364` |
| Count quota exceeded | `models.py:867-884`, `new_custom_alias.py:49` | `400` | `You have reached the limitation of a free account with the maximum of 3 aliases…` | `LOG.d new_custom_alias.py:49` |
| Suffix `None` (expired/tampered/garbage/empty) | `alias_suffix.py:37-42`, `new_custom_alias.py:71-73` | `412` | `Alias creation time is expired, please retry` | `LOG.w new_custom_alias.py:72` (v2) / `:187` (v3) |
| Suffix raises (dead for bad tokens) | `new_custom_alias.py:74-76` | `400` | `Tampered suffix` | `LOG.w` (effectively unreachable) |
| Prefix/suffix mismatch | `alias_suffix.py:45-91`, `new_custom_alias.py:78-79` | `400` | `wrong alias prefix or suffix` | `LOG.e alias_suffix.py:61` |
| Duplicate alias | `new_custom_alias.py:82-88` | `409` | `alias {full_alias} already exists` | `LOG.d new_custom_alias.py:87` (v2) / `:202` (v3) |
| `".."` in alias | `new_custom_alias.py:90-94` | `400` | `2 consecutive dot signs aren't allowed…` | — |
| Token bucket exceeded | `models.py:1641`, `rate_limiter.py:30-40` | `429` | `{"error":"Rate limit exceeded"}` | `LOG.i rate_limiter.py:33` |
| Success | `new_custom_alias.py:109-112` | `201` | full alias info object | audit `"New alias created"` (`models.py:1688-1689`) |

### D.5 Execution-path flowchart

The chain below was confirmed against the observed outputs. Two annotations reflect observed corrections to the originally-expected behavior: the `412` collapse (§A) and the JSON (not HTML) `429`.

```mermaid
flowchart TD
    A[POST /api/v2 or v3 /alias/custom/new] --> B{Flask-Limiter ALIAS_LIMIT?}
    B -- over limit --> B1[429 JSON Rate limit exceeded - no rate-limit headers]
    B -- ok --> C{require_api_auth}
    C -- missing/invalid key --> C1[401 Wrong api key]
    C -- disabled --> C2[403 Disabled account]
    C -- ok --> D{parallel_limiter lock held?}
    D -- held --> D1[429 JSON Rate limit exceeded concurrency]
    D -- acquired --> E{can_create_new_alias?}
    E -- false --> E1[400 quota message + LOG.d cannot create]
    E -- true --> F[check_suffix_signature max_age 600]
    F -- returns None: expired/tampered/garbage/empty --> F1[412 Alias creation time is expired + LOG.w expired]
    F -- non-BadSignature error --> F2[400 Tampered suffix + LOG.w tampered - EFFECTIVELY DEAD]
    F -- valid --> G{verify_prefix_suffix}
    G -- false --> G1[400 wrong alias prefix or suffix + LOG.e]
    G -- true --> H{duplicate alias?}
    H -- yes --> H1[409 alias already exists + LOG.d]
    H -- no --> I[Alias.create -> check_bucket_limit]
    I -- bucket exceeded --> I1[429 JSON Rate limit exceeded + LOG.i Rate limit hit]
    I -- ok --> J[201 Created + audit New alias created]
%% Redis-gated layers D and I are skipped when lock_redis is None
%% All /api/ 429s are JSON via server.py:362-372 (not HTML)
```

### D.6 Component-level answer to "which validates suffixes / which enforces limits"

- **Validates signed suffixes:** `check_suffix_signature` (signature + 600 s expiry, `app/alias_suffix.py:37-42`) then `verify_prefix_suffix` (domain ownership + leading-dot rule, `app/alias_suffix.py:45-91`).
- **Enforces creation limits (four):** (1) Flask-Limiter `@limiter.limit(ALIAS_LIMIT)` (`extensions.py:23`); (2) `parallel_limiter` concurrency lock (`parallel_limiter.py`); (3) `can_create_new_alias()` count quota (`models.py:867-884`); (4) `Alias.create()` token bucket (`models.py:1641` + `rate_limiter.py`).

---


## §E — Why the Failures Are Intermittent

The intermittency is **code-level and reproducible**, not a vague environmental flake. Four concrete causes were exercised. First, the deterministic cases establish that identical *content* faults are stable across runs; then the genuinely time/concurrency/state-dependent causes explain run-to-run variation on otherwise-identical input.

### E.0 Deterministic content faults (baseline distribution across 2 identical-input runs)

For the suffix-content and quota conditions, identical input yields identical output across two independent cross-process runs (RUN1 and RUN2, different users). The aggregate status distribution was **byte-identical** between the two runs:

```text
RUN1 status distribution      RUN2 status distribution
  201 : 6                       201 : 6
  400 : 3                       400 : 3
  409 : 2                       409 : 2
  412 : 16                      412 : 16
  429 : 2                       429 : 2
```

Per-condition (each cell is the outcome of the *same* request submitted twice):

| Condition | RUN1 | RUN2 |
|-----------|------|------|
| 1 valid (v2, v3) | `201`, `201` | `201`, `201` |
| 2 expired (v2 ×2, v3 ×2) | `412` ×4 | `412` ×4 |
| 3 tampered (v2 ×2, v3 ×2) | `412` ×4 | `412` ×4 |
| 4 garbage / empty (v2 ×2 each, v3 ×2 each) | `412` ×8 | `412` ×8 |
| 5 prefix/suffix mismatch (v2 ×2) | `400`, `400` | `400`, `400` |
| 6 duplicate (v2, v3) | `201`,`409` / `201`,`409` | `201`,`409` / `201`,`409` |
| 7 quota (start count 1) | `201,201,400` @attempt 3 | `201,201,400` @attempt 3 |
| 9 concurrency (lock held, ×2) | `429`, `429` | `429`, `429` |

These are *not* the source of the reported intermittency — they are stable. The intermittency comes from the four causes below, where the *same* request can succeed or fail depending on timing, concurrency, or backend state.

### E.1 The 600-second suffix-validity window (`app/alias_suffix.py:37-42`)

A signed suffix is valid for exactly `max_age=600` seconds. A suffix minted by `GET /api/v?/alias/options` (`app/api/views/alias_options.py:69,140`) and then POSTed **within 10 minutes** succeeds; the **identical** signed-suffix string POSTed after 10 minutes fails with `412 "…expired, please retry"`. The very same token thus flips from success to failure purely as a function of elapsed time — a classic "worked a moment ago, fails now" intermittency, and (because of §A) tampered/garbage tokens are reported under this same "expired" message, compounding the confusion. Runtime proof of the boundary: `unsign(..., max_age=600)` on a 700-second-old token → `SignatureExpired: Signature age 700 > 600 seconds` (§A.3).

### E.2 The `parallel_limiter` concurrency `429` (`app/parallel_limiter.py:19-73`)

Two overlapping alias-creation requests from the same user collide on the Redis lock `cl:{user.id}:alias_creation`; the second raises `TooManyRequests` → `429`. Under normal single-request use nothing happens, but a double-click or a rapid client retry produces an intermittent `429`. Observed (Condition 9; method: pre-set the exact Redis lock key the code uses to simulate an in-flight holder — clearly labeled as a **simulated** concurrent holder — then issue the same real request twice so the *real* `acquire_lock` at `parallel_limiter.py:30-34` raises):

```text
===== [RUN1] CONDITION 9 — parallel_limiter concurrency (v2); simulated in-flight holder =====
user=user_9qy14vj2se@mailbox.test id=955
pre-setting lock key (SIMULATED concurrent in-flight holder): cl:955:alias_creation
----- v2 concurrency attempt 1 (identical input) -----
redis get(lock) = b'held-by-inflight'
2026-07-08 06:19:38,770 - SL - WARNING - 13902 - "/code/server.py:364" - rate_limited() -  - Client hit rate limit on path /api/v2/alias/custom/new, user:<User 955 Test User user_9qy14vj2se@mailbox.test>
2026-07-08 06:19:38,770 - SL - DEBUG - 13902 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 429, takes 0.006455659866333008
STATUS-LINE: 429 TOO MANY REQUESTS | status_code: 429
RESPONSE-HEADERS:
    Content-Type: application/json
    Content-Length: 32
    Access-Control-Allow-Origin: *
    Set-Cookie: slapp=<redacted>; Domain=.sl.test; Expires=Wed, 15-Jul-2026 06:19:38 GMT; HttpOnly; Path=/; SameSite=Lax
JSON-BODY: {"error":"Rate limit exceeded"}

----- v2 concurrency attempt 2 (identical input) -----
redis get(lock) = b'held-by-inflight'
2026-07-08 06:19:38,779 - SL - WARNING - 13902 - "/code/server.py:364" - rate_limited() -  - Client hit rate limit on path /api/v2/alias/custom/new, user:<User 955 Test User user_9qy14vj2se@mailbox.test>
2026-07-08 06:19:38,779 - SL - DEBUG - 13902 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 429, takes 0.00626373291015625
STATUS-LINE: 429 TOO MANY REQUESTS | status_code: 429
RESPONSE-HEADERS:
    Content-Type: application/json
    Content-Length: 32
    Access-Control-Allow-Origin: *
    Set-Cookie: slapp=<redacted>; Domain=.sl.test; Expires=Wed, 15-Jul-2026 06:19:38 GMT; HttpOnly; Path=/; SameSite=Lax
JSON-BODY: {"error":"Rate limit exceeded"}
```

The lock auto-expires after `max_wait_secs=5` (`parallel_limiter.py:23`), so the window is sub-second-to-5-seconds wide — exactly the profile of an intermittent failure.

### E.3 The Flask-Limiter `5/minute` rolling burst window (`app/config.py:448`)

`ALIAS_LIMIT="100/day;50/hour;5/minute"`. The `5/minute` component is a rolling 60-second window (the `limits 1.5.1` fixed-window strategy over Redis). The 6th identical request within a minute is rejected with `429`. Observed over a **real gunicorn HTTP server** via `curl -D -` (Condition 10) — the transition from `201`/`409` to `429` is unambiguous, and the `429` carries no rate-limit headers (only `Server`/`Date`/`Connection` plus the four standard keys):

```text
=================== REAL HTTP BURST (6 identical POSTs, fresh window) ===================
----- curl request 1 -----
HTTP/1.1 201 CREATED
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 06:24:23 GMT
Connection: close
Content-Type: application/json
Content-Length: 436
Access-Control-Allow-Origin: *
Set-Cookie: slapp=<redacted>; Expires=Wed, 15-Jul-2026 06:24:23 GMT; HttpOnly; Path=/; SameSite=Lax

BODY: {"alias":"burst.4qnd9u@sl.local","creation_date":"2026-07-08 06:24:23+00:00","creation_timestamp":1783491863,"disable_pgp":false,"email":"burst.4qnd9u@sl.local","enabled":true,"id":1987,"latest_activity":null,"mailbox":{"email":"user_bixdu43he7@mailbox.test","id":1138},"mailboxes":[{"email":"user_bixdu43he7@mailbox.test","id":1138}],"name":null,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null,"pinned":false,"support_pgp":false}

----- curl request 2 -----
HTTP/1.1 409 CONFLICT
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 06:24:23 GMT
Connection: close
Content-Type: application/json
Content-Length: 55
Access-Control-Allow-Origin: *
Set-Cookie: slapp=<redacted>; Expires=Wed, 15-Jul-2026 06:24:23 GMT; HttpOnly; Path=/; SameSite=Lax

BODY: {"error":"alias burst.4qnd9u@sl.local already exists"}

----- curl request 3 -----
HTTP/1.1 409 CONFLICT
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 06:24:23 GMT
Connection: close
Content-Type: application/json
Content-Length: 55
Access-Control-Allow-Origin: *
Set-Cookie: slapp=<redacted>; Expires=Wed, 15-Jul-2026 06:24:23 GMT; HttpOnly; Path=/; SameSite=Lax

BODY: {"error":"alias burst.4qnd9u@sl.local already exists"}

----- curl request 4 -----
HTTP/1.1 409 CONFLICT
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 06:24:23 GMT
Connection: close
Content-Type: application/json
Content-Length: 55
Access-Control-Allow-Origin: *
Set-Cookie: slapp=<redacted>; Expires=Wed, 15-Jul-2026 06:24:23 GMT; HttpOnly; Path=/; SameSite=Lax

BODY: {"error":"alias burst.4qnd9u@sl.local already exists"}

----- curl request 5 -----
HTTP/1.1 409 CONFLICT
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 06:24:23 GMT
Connection: close
Content-Type: application/json
Content-Length: 55
Access-Control-Allow-Origin: *
Set-Cookie: slapp=<redacted>; Expires=Wed, 15-Jul-2026 06:24:23 GMT; HttpOnly; Path=/; SameSite=Lax

BODY: {"error":"alias burst.4qnd9u@sl.local already exists"}

----- curl request 6 -----
HTTP/1.1 429 TOO MANY REQUESTS
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 06:24:23 GMT
Connection: close
Content-Type: application/json
Content-Length: 32
Access-Control-Allow-Origin: *
Set-Cookie: slapp=<redacted>; Expires=Wed, 15-Jul-2026 06:24:23 GMT; HttpOnly; Path=/; SameSite=Lax

BODY: {"error":"Rate limit exceeded"}

=================== gunicorn SL rate-limit log lines ===================
2026-07-08 06:24:23,534 - SL - WARNING - 14040 - "/code/server.py:364" - rate_limited() -  - Client hit rate limit on path /api/v2/alias/custom/new, user:<flask_login.mixins.AnonymousUserMixin object at 0x77ff02f7f8e0>
=================== gunicorn 'Server' header confirm ===================
(server header shown in curl -D - output above)
```

Here the first identical POST created the alias (`201`); the next four identical POSTs were duplicates (`409`) yet **still consumed limiter hits** because `@limiter.limit` is the *outermost* decorator and counts before the handler body runs; the 6th crossed the `5/minute` window and returned `429`. Two facets of this layer therefore produce intermittency on *identical* input:

- **It counts pre-auth / pre-outcome** (code-grounded: `@limiter.limit` is the outermost decorator at `new_custom_alias.py:29`, so every request to the route — including ones that later `409`/`401` — increments the window). A client that retries a duplicate or an unauthenticated request still burns its `5/minute` budget, so the position of the `429` shifts with recent traffic.
- **It accumulates across the rolling 60-second window and is shared across worker processes** (code-grounded: the limiter storage is the shared `RedisStorage` wired at `server.py:163-165`). Requests from different gunicorn workers in the same 60-second window count against the same key, so whether the *n*-th identical request is allowed depends on how many hits already landed in the live window.

> **Observed methodology note (harness artifact, not a SimpleLogin bug):** under Flask's *test client*, reusing a single `app.app_context()` across many client calls causes Flask-Limiter's per-request guard flag to persist, suppressing the limiter for later calls; this does not occur in production (gunicorn), where each request gets a fresh application context. The Flask-Limiter evidence above was therefore captured **authoritatively via a real gunicorn server**, not the test client, precisely to avoid that artifact.

### E.4 The Redis-availability gate (`app/rate_limiter.py:28`, `app/parallel_limiter.py:51`, `server.py:163-165`)

Two of the four limiters — the concurrency lock and the token bucket — are wired **only** when `MEM_STORE_URI` is set and Redis is reachable (`server.py:163-165` → `app/redis_services.py:9-11`). When `lock_redis is None`, both silently no-op: `check_bucket_limit` returns immediately (`rate_limiter.py:28`) and the parallel lock does nothing (`parallel_limiter.py:51-52`). So the *same* workload that returns `429` with Redis up returns `201` with Redis down. Demonstrated directly (runtime toggle of `lock_redis = None`, ephemeral, no repo change):

```text
===== E.4 — SIMULATE NO REDIS: set both lock_redis = None (runtime toggle, ephemeral) =====
rate_limiter.lock_redis = None  parallel_limiter.lock_redis = None

--- (1) rate_limiter.check_bucket_limit('nr_demo', max_hits=10, 900) called 12x (would breach at value 11 WITH redis) ---
   -> no-op confirmed: never raised across 12 calls

--- (2) endpoint: expired-trial FREE user, MAX_NB_EMAIL_FREE_PLAN=1000, NO redis, 13 creations (WITH redis breaches at value 11) ---
2026-07-08 06:25:18,338 - SL - DEBUG - 14076 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04549264907836914
2026-07-08 06:25:18,383 - SL - DEBUG - 14076 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.0429074764251709
2026-07-08 06:25:18,429 - SL - DEBUG - 14076 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04409480094909668
2026-07-08 06:25:18,471 - SL - DEBUG - 14076 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04046750068664551
2026-07-08 06:25:18,514 - SL - DEBUG - 14076 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04123115539550781
2026-07-08 06:25:18,557 - SL - DEBUG - 14076 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.041596412658691406
2026-07-08 06:25:18,599 - SL - DEBUG - 14076 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.039877891540527344
2026-07-08 06:25:18,645 - SL - DEBUG - 14076 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04420137405395508
2026-07-08 06:25:18,688 - SL - DEBUG - 14076 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04110431671142578
2026-07-08 06:25:18,731 - SL - DEBUG - 14076 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04167628288269043
2026-07-08 06:25:18,777 - SL - DEBUG - 14076 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04377913475036621
2026-07-08 06:25:18,821 - SL - DEBUG - 14076 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.042031288146972656
2026-07-08 06:25:18,864 - SL - DEBUG - 14076 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04164004325866699
   is_premium=False
   statuses (13 creations): [201, 201, 201, 201, 201, 201, 201, 201, 201, 201, 201, 201, 201]
   -> NO bucket 429 (all 201): bucket limiter DISABLED without redis
   (contrast: Condition 8b WITH redis breaches at value 11/10 -> 429)
```

If a deployment intermittently loses Redis connectivity (or runs a mix of configured/unconfigured workers), the concurrency and bucket `429`s appear and disappear accordingly — a fourth, infrastructure-shaped source of the reported intermittency, but one that is fully explained by the code-level gate above.

### E.5 Summary of intermittency causes

| Cause | Mechanism | Citation | Manifestation on identical input |
|-------|-----------|----------|----------------------------------|
| 600 s suffix window | token valid only 10 min | `alias_suffix.py:37-42` | same token: `201` now, `412` later |
| Concurrency lock | second overlapping request rejected | `parallel_limiter.py:19-73` | double-click → intermittent `429` |
| Flask-Limiter `5/minute` | rolling 60 s window, counts pre-outcome, shared across processes | `config.py:448`, `extensions.py:23` | 6th (or fewer) request → `429`; window state varies |
| Redis-availability gate | bucket + concurrency no-op without Redis | `rate_limiter.py:28`, `parallel_limiter.py:51`, `server.py:163-165` | `429` with Redis, `201` without |

---


## §F — Build & Invocation Commands, Full Output Blocks

### F.1 Exact commands

Infrastructure (canonical, per `scripts/run-test.sh`), inside the provided container image (SimpleLogin, Python 3.10.18 venv at `/app/venv`), with Postgres 13 on `:15432` and Redis on `:6379` reachable via `--network host`:

```bash
# Postgres 13 (scripts/run-test.sh:7) and Redis
docker run -d --name sl-test-db -e POSTGRES_PASSWORD=test -e POSTGRES_USER=test -e POSTGRES_DB=test -p 15432:5432 postgres:13
docker run -d --name sl-redis -p 6379:6379 redis:6

# DB migration (scripts/run-test.sh:13)
docker exec sl-app bash -lc 'source /app/venv/bin/activate; cd /code; CONFIG=tests/test.env alembic upgrade head'
```

Reproduction drivers were kept **outside** the repository (host `/tmp/probes/`) and streamed into the container over stdin, so nothing was ever written under the repo tree:

```bash
# Environment/deps/config + itsdangerous hierarchy + check_suffix_signature collapse
docker exec -i sl-app bash -lc \
  'source /app/venv/bin/activate; cd /code; CONFIG=tests/test.env python -' \
  < /tmp/probes/probe_env.py > /tmp/probes/evidence_env.txt 2>&1

# Conditions 1-7 & 9 (v2 AND v3), identical-input repetition, via the Flask test client
#   (cookie session; suffixes minted via signer.sign / get_alias_suffixes)
docker exec -i sl-app bash -lc \
  'source /app/venv/bin/activate; cd /code; CONFIG=tests/test.env PYTHONUNBUFFERED=1 RUN_TAG=RUN1 python -' \
  < /tmp/probes/probe_main.py > /tmp/probes/evidence_run1.txt 2>&1
docker exec -i sl-app bash -lc \
  'source /app/venv/bin/activate; cd /code; CONFIG=tests/test.env PYTHONUNBUFFERED=1 RUN_TAG=RUN2 python -' \
  < /tmp/probes/probe_main.py > /tmp/probes/evidence_run2.txt 2>&1

# Condition 8 (Alias.create token bucket) — PAID (fresh trial) and FREE (expired trial)
#   NOTE: probe_cond8.py raises MAX_NB_EMAIL_FREE_PLAN to 1000 IN-PROCESS (ephemeral runtime
#   toggle; no file changed) so the count quota — which under the canonical cap=3 refuses a
#   free/trial user at attempt 3 (400) regardless of trial state — does not mask the token
#   bucket. All other configuration is canonical (CONFIG=tests/test.env). See §F.5 / §C.2.
docker exec -i sl-app bash -lc \
  'source /app/venv/bin/activate; cd /code; CONFIG=tests/test.env PYTHONUNBUFFERED=1 RUN_TAG=RUN1 python -' \
  < /tmp/probes/probe_cond8.py > /tmp/probes/evidence_cond8_run1.txt 2>&1

# Condition 10 (Flask-Limiter burst) — authoritative REAL server via gunicorn + curl -D -
docker exec -i sl-app bash -lc 'bash -s' < /tmp/probes/real_server_cond10.sh > /tmp/probes/evidence_cond10.txt 2>&1
#   internally: CONFIG=tests/test.env gunicorn wsgi:app -b 127.0.0.1:7788 -w 1 --timeout 60
#               then 6x: curl -s -D - -X POST http://127.0.0.1:7788/api/v2/alias/custom/new \
#                          -H "Authentication: <ApiKey.code>" -H "Content-Type: application/json" \
#                          -d '{"alias_prefix":"burst","signed_suffix":"<signed_suffix>"}'

# O2 explicit header scan + E.4 no-Redis gate demonstration
docker exec -i sl-app bash -lc \
  'source /app/venv/bin/activate; cd /code; CONFIG=tests/test.env python -' \
  < /tmp/probes/probe_extra.py > /tmp/probes/evidence_extra.txt 2>&1
```

All `/tmp/probes/*.py` and `/tmp/probes/*.txt` artifacts are outside the repository and were deleted on completion (§F.7).

### F.2 Dependency & config confirmation

The full verbatim output is in the Runtime section above (versions, effective config, `issubclass` hierarchy, `check_suffix_signature` collapse, raw `unsign` exception types, and limiter storage liveness). It confirms `itsdangerous 1.1.0`, `flask 1.1.2`, `werkzeug 1.0.1`, `limits 1.5.1`, `flask_limiter 1.4`, `python 3.10.18`, and the canonical config values.

### F.3 Condition 1 — valid fresh suffix, v3 (complete block, with `AliasAuditLog` row)

The v2 valid block (with its audit row) is in §C.3. The v3 valid block below was produced by the real `POST /api/v3/alias/custom/new` handler with `mailbox_ids=[default_mailbox_id]`, and its `AliasAuditLog` row (`action='create'`, `message='New alias created'`) is queried and printed the same way:

```text
===== [RUN1] CONDITION 1 — valid fresh suffix (v3) =====
user=user_51kf7t251n@mailbox.test id=942 default_mailbox_id=1101 start_count=1
signed_suffix=.buldyq@sl.local.ak3r8Q.Xo6qbPThfYSUR1uuKSqtgajQVM4
2026-07-08 06:19:29,652 - SL - DEBUG - 13902 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v3/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04633736610412598
STATUS-LINE: 201 CREATED | status_code: 201
RESPONSE-HEADERS:
    Content-Type: application/json
    Content-Length: 438
    Access-Control-Allow-Origin: *
    Set-Cookie: slapp=<redacted>; Domain=.sl.test; Expires=Wed, 15-Jul-2026 06:19:29 GMT; HttpOnly; Path=/; SameSite=Lax
JSON-BODY: {"alias":"prefix.buldyq@sl.local","creation_date":"2026-07-08 06:19:29+00:00","creation_timestamp":1783491569,"disable_pgp":false,"email":"prefix.buldyq@sl.local","enabled":true,"id":1707,"latest_activity":null,"mailbox":{"email":"user_51kf7t251n@mailbox.test","id":1101},"mailboxes":[{"email":"user_51kf7t251n@mailbox.test","id":1101}],"name":null,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null,"pinned":false,"support_pgp":false}

alias_count_after=2
--- AliasAuditLog query for created alias (proves success audit) ---
AUDIT-RECORD: id=1785 user_id=942 alias_id=1707 alias_email='prefix.buldyq@sl.local' action='create' message='New alias created' created_at=2026-07-08T06:19:29.645589+00:00
```

### F.4 Conditions 5 & 6 — complete blocks

The complete, unedited blocks for the remaining conditions are reproduced verbatim in the sections that answer them:
- **Condition 2 (expired), 3 (tampered), 4 (garbage/empty)** — v2 and v3: §A.4.
- **Condition 7 (quota, count before/after):** §C.1.
- **Condition 9 (concurrency → `429`):** §E.2.
- **Condition 10 (Flask-Limiter burst, real gunicorn):** §E.3.

**Condition 5 — mismatch (validly-signed suffix for a domain the user cannot use), v2, identical input ×2:**

```text
===== [RUN1] CONDITION 5 — prefix/suffix mismatch (v2); validly-signed unavailable domain =====
user=user_3a8p8e7xbp@mailbox.test id=951
signed bad-domain suffix: @0eomyn.test.ak3r9w.IHLPneTNhQ3cnW8kcC5aQnvKbf0  raw_unsigned='@0eomyn.test'
----- v2 mismatch attempt 1 (identical input) -----
2026-07-08 06:19:35,564 - SL - ERROR - 13902 - "/code/app/alias_suffix.py:61" - verify_prefix_suffix() -  - wrong alias suffix @0eomyn.test, user <User 951 Test User user_3a8p8e7xbp@mailbox.test>
NoneType: None
2026-07-08 06:19:35,565 - SL - DEBUG - 13902 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 400, takes 0.03281831741333008
STATUS-LINE: 400 BAD REQUEST | status_code: 400
RESPONSE-HEADERS:
    Content-Type: application/json
    Content-Length: 41
    Access-Control-Allow-Origin: *
    Set-Cookie: slapp=<redacted>; Domain=.sl.test; Expires=Wed, 15-Jul-2026 06:19:35 GMT; HttpOnly; Path=/; SameSite=Lax
JSON-BODY: {"error":"wrong alias prefix or suffix"}

----- v2 mismatch attempt 2 (identical input) -----
2026-07-08 06:19:35,599 - SL - ERROR - 13902 - "/code/app/alias_suffix.py:61" - verify_prefix_suffix() -  - wrong alias suffix @0eomyn.test, user <User 951 Test User user_3a8p8e7xbp@mailbox.test>
NoneType: None
2026-07-08 06:19:35,600 - SL - DEBUG - 13902 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 400, takes 0.03200721740722656
STATUS-LINE: 400 BAD REQUEST | status_code: 400
RESPONSE-HEADERS:
    Content-Type: application/json
    Content-Length: 41
    Access-Control-Allow-Origin: *
    Set-Cookie: slapp=<redacted>; Domain=.sl.test; Expires=Wed, 15-Jul-2026 06:19:35 GMT; HttpOnly; Path=/; SameSite=Lax
JSON-BODY: {"error":"wrong alias prefix or suffix"}
```

**Condition 6 — duplicate full alias (same prefix+suffix twice), v2 then v3:** the first POST creates (`201`); the identical second POST returns `409` with `LOG.d "full alias already used …"` at `new_custom_alias.py:87` (v2) / `:202` (v3):

*v2:*
```text
===== [RUN1] CONDITION 6 — duplicate (v2); IDENTICAL valid payload posted twice (state change) =====
user=user_ajrwui40vb@mailbox.test id=952
IDENTICAL payload (posted twice): alias_prefix='dupe58f6' signed_suffix=.qhpv9z@sl.local.ak3r-A.CMJV5wWC54wJg_FOFdquFWuidsk
----- v2 duplicate attempt 1 (identical input) -----
2026-07-08 06:19:36,430 - SL - DEBUG - 13902 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.07901716232299805
STATUS-LINE: 201 CREATED | status_code: 201
RESPONSE-HEADERS:
    Content-Type: application/json
    Content-Length: 442
    Access-Control-Allow-Origin: *
    Set-Cookie: slapp=<redacted>; Domain=.sl.test; Expires=Wed, 15-Jul-2026 06:19:36 GMT; HttpOnly; Path=/; SameSite=Lax
JSON-BODY: {"alias":"dupe58f6.qhpv9z@sl.local","creation_date":"2026-07-08 06:19:36+00:00","creation_timestamp":1783491576,"disable_pgp":false,"email":"dupe58f6.qhpv9z@sl.local","enabled":true,"id":1718,"latest_activity":null,"mailbox":{"email":"user_ajrwui40vb@mailbox.test","id":1111},"mailboxes":[{"email":"user_ajrwui40vb@mailbox.test","id":1111}],"name":null,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null,"pinned":false,"support_pgp":false}

----- v2 duplicate attempt 2 (identical input) -----
2026-07-08 06:19:36,474 - SL - DEBUG - 13902 - "/code/app/api/views/new_custom_alias.py:87" - new_custom_alias_v2() -  - full alias already used dupe58f6.qhpv9z@sl.local
2026-07-08 06:19:36,475 - SL - DEBUG - 13902 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 409, takes 0.04148411750793457
STATUS-LINE: 409 CONFLICT | status_code: 409
RESPONSE-HEADERS:
    Content-Type: application/json
    Content-Length: 58
    Access-Control-Allow-Origin: *
    Set-Cookie: slapp=<redacted>; Domain=.sl.test; Expires=Wed, 15-Jul-2026 06:19:36 GMT; HttpOnly; Path=/; SameSite=Lax
JSON-BODY: {"error":"alias dupe58f6.qhpv9z@sl.local already exists"}
```

*v3:*
```text
===== [RUN1] CONDITION 6 — duplicate (v3); IDENTICAL valid payload posted twice (state change) =====
user=user_z8mirjrg59@mailbox.test id=953
IDENTICAL payload (posted twice): alias_prefix='dupeb02q' signed_suffix=.sb6tls@sl.local.ak3r-Q.HJI4FzvvMFOruVsqdx_q3YmUEtI
----- v3 duplicate attempt 1 (identical input) -----
2026-07-08 06:19:37,213 - SL - DEBUG - 13902 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v3/alias/custom/new ImmutableMultiDict([]) 201, takes 0.0750436782836914
STATUS-LINE: 201 CREATED | status_code: 201
RESPONSE-HEADERS:
    Content-Type: application/json
    Content-Length: 442
    Access-Control-Allow-Origin: *
    Set-Cookie: slapp=<redacted>; Domain=.sl.test; Expires=Wed, 15-Jul-2026 06:19:37 GMT; HttpOnly; Path=/; SameSite=Lax
JSON-BODY: {"alias":"dupeb02q.sb6tls@sl.local","creation_date":"2026-07-08 06:19:37+00:00","creation_timestamp":1783491577,"disable_pgp":false,"email":"dupeb02q.sb6tls@sl.local","enabled":true,"id":1720,"latest_activity":null,"mailbox":{"email":"user_z8mirjrg59@mailbox.test","id":1112},"mailboxes":[{"email":"user_z8mirjrg59@mailbox.test","id":1112}],"name":null,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null,"pinned":false,"support_pgp":false}

----- v3 duplicate attempt 2 (identical input) -----
2026-07-08 06:19:37,253 - SL - DEBUG - 13902 - "/code/app/api/views/new_custom_alias.py:202" - new_custom_alias_v3() -  - full alias already used dupeb02q.sb6tls@sl.local
2026-07-08 06:19:37,254 - SL - DEBUG - 13902 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v3/alias/custom/new ImmutableMultiDict([]) 409, takes 0.037441253662109375
STATUS-LINE: 409 CONFLICT | status_code: 409
RESPONSE-HEADERS:
    Content-Type: application/json
    Content-Length: 58
    Access-Control-Allow-Origin: *
    Set-Cookie: slapp=<redacted>; Domain=.sl.test; Expires=Wed, 15-Jul-2026 06:19:37 GMT; HttpOnly; Path=/; SameSite=Lax
JSON-BODY: {"error":"alias dupeb02q.sb6tls@sl.local already exists"}
```

### F.5 Condition 8 — token bucket (PAID and FREE), complete blocks

Each block is the **complete, unedited** capture: every per-POST `after_request` line up to the breach, the `LOG.i "Rate limit hit …"` at `rate_limiter.py:33`, the `server.py:364` handler line, and the full `429` header set + 32-byte JSON body. The auto-created newsletter alias consumes one bucket hit at signup (start count = 1), so the PAID breach value `51/50` lands on custom-alias POST #50 and the FREE breach `11/10` on POST #10.

> **Config disclosure (non-canonical override, isolated to Condition 8):** to reach the token bucket at all, `probe_cond8.py` raised `MAX_NB_EMAIL_FREE_PLAN` from its canonical `3` (the Runtime & Methodology *Effective configuration* block; `tests/test.env:13`) to `1000`. Under the canonical cap of `3`, the **count quota** refuses a free/trial user at attempt 3 (`400`, §C.1/§C.2) — *regardless of trial state* — so the token bucket would never be reached; raising the cap removes that mask and leaves the per-user token bucket as the sole binding limit (the Flask-Limiter `5/minute` burst is inert under the test client, which is precisely why Condition 10 is driven over a real gunicorn server instead — §E.3/§F.6). This mirrors the disclosure discipline already applied in §E.4. Every other setting is canonical (`CONFIG=tests/test.env`), and the breach *values* below (`51/50`, `11/10`), the `LOG.i` line at `rate_limiter.py:33`, the 32-byte `429` body, and the rate-limit-header absence are independent of the count cap and were reproduced identically across both runs.

**8a — fresh-trial user (`is_premium=True` → PAID bucket `(50,900)`):**

```text
===== [RUN1] CONDITION 8a — FRESH TRIAL user => is_premium True => PAID bucket (50,900) =====
user id=975 bucket=PAID limits=[(50, 900), (200, 3600)] is_premium=True lifetime_or_active_sub=False
NOTE: the auto-created newsletter alias already consumed 1 token-bucket hit at signup (start alias count=1),
      so the breach value (51/50) is reached on custom-alias POST #50.
2026-07-08 06:23:18,012 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04886627197265625
  POST #1: STATUS 201 (ok)  body(id)=1867
2026-07-08 06:23:18,058 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04342913627624512
2026-07-08 06:23:18,102 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04227280616760254
2026-07-08 06:23:18,146 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04248642921447754
2026-07-08 06:23:18,190 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.042542219161987305
2026-07-08 06:23:18,236 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04438304901123047
2026-07-08 06:23:18,280 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04265308380126953
2026-07-08 06:23:18,327 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04491710662841797
2026-07-08 06:23:18,373 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04394888877868652
2026-07-08 06:23:18,418 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04352903366088867
2026-07-08 06:23:18,466 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.0461115837097168
2026-07-08 06:23:18,513 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.0455174446105957
2026-07-08 06:23:18,561 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.046858787536621094
2026-07-08 06:23:18,608 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.044812917709350586
2026-07-08 06:23:18,654 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04433298110961914
2026-07-08 06:23:18,702 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04650616645812988
2026-07-08 06:23:18,749 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04499506950378418
2026-07-08 06:23:18,794 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04387259483337402
2026-07-08 06:23:18,840 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.043813228607177734
2026-07-08 06:23:18,888 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04653167724609375
2026-07-08 06:23:18,943 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.053182125091552734
2026-07-08 06:23:18,991 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.045487165451049805
2026-07-08 06:23:19,038 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04608964920043945
2026-07-08 06:23:19,085 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04532051086425781
2026-07-08 06:23:19,132 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04506540298461914
2026-07-08 06:23:19,181 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.046556711196899414
2026-07-08 06:23:19,226 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04364657402038574
2026-07-08 06:23:19,275 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.047432899475097656
2026-07-08 06:23:19,321 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04464101791381836
2026-07-08 06:23:19,368 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.045046091079711914
2026-07-08 06:23:19,416 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04558086395263672
2026-07-08 06:23:19,462 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.044756174087524414
2026-07-08 06:23:19,509 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.044846534729003906
2026-07-08 06:23:19,555 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04473257064819336
2026-07-08 06:23:19,605 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04797649383544922
2026-07-08 06:23:19,654 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04760003089904785
2026-07-08 06:23:19,701 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04474997520446777
2026-07-08 06:23:19,744 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04174447059631348
2026-07-08 06:23:19,787 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04127049446105957
2026-07-08 06:23:19,831 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.041674137115478516
2026-07-08 06:23:19,878 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.045662641525268555
2026-07-08 06:23:19,925 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04555010795593262
2026-07-08 06:23:19,970 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.042792320251464844
2026-07-08 06:23:20,015 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04307126998901367
2026-07-08 06:23:20,060 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.043254852294921875
2026-07-08 06:23:20,106 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.044356584548950195
2026-07-08 06:23:20,164 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.05558013916015625
2026-07-08 06:23:20,208 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.042971134185791016
2026-07-08 06:23:20,255 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.044979095458984375
  POST #49: STATUS 201 (ok)  body(id)=1915
2026-07-08 06:23:20,289 - SL - INFO - 13978 - "/code/app/rate_limiter.py:33" - check_bucket_limit() -  - Rate limit hit for alias_create_900d:975 (bucket id 1783491300) -> 51/50
2026-07-08 06:23:20,289 - SL - WARNING - 13978 - "/code/server.py:364" - rate_limited() -  - Client hit rate limit on path /api/v2/alias/custom/new, user:<User 975 T user_b6qtcj0ygh@mailbox.test>
2026-07-08 06:23:20,289 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 429, takes 0.03240036964416504
  POST #50: STATUS 429  <-- BREACH
  FULL RESPONSE-HEADERS:
      Content-Type: application/json
      Content-Length: 32
      Access-Control-Allow-Origin: *
      Set-Cookie: slapp=<redacted>; Domain=.sl.test; Expires=Wed, 15-Jul-2026 06:23:20 GMT; HttpOnly; Path=/; SameSite=Lax
  FULL BODY [32 bytes, CT='application/json']: {"error":"Rate limit exceeded"}
```

**8b — expired-trial (genuine free) user (`is_premium=False` → FREE bucket `(10,900)`):**

```text
===== [RUN1] CONDITION 8b — EXPIRED-TRIAL (genuine free) user => FREE bucket (10,900) =====
user id=976 bucket=FREE limits=[(10, 900), (50, 3600)] is_premium=False lifetime_or_active_sub=False
NOTE: the auto-created newsletter alias already consumed 1 token-bucket hit at signup (start alias count=1),
      so the breach value (11/10) is reached on custom-alias POST #10.
2026-07-08 06:23:20,976 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.044568538665771484
  POST #1: STATUS 201 (ok)  body(id)=1917
2026-07-08 06:23:21,022 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04384446144104004
2026-07-08 06:23:21,068 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.044199466705322266
2026-07-08 06:23:21,116 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04567742347717285
2026-07-08 06:23:21,160 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04237675666809082
2026-07-08 06:23:21,205 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04366183280944824
2026-07-08 06:23:21,247 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04084038734436035
2026-07-08 06:23:21,291 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.042171478271484375
2026-07-08 06:23:21,335 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 201, takes 0.04189491271972656
  POST #9: STATUS 201 (ok)  body(id)=1925
2026-07-08 06:23:21,369 - SL - INFO - 13978 - "/code/app/rate_limiter.py:33" - check_bucket_limit() -  - Rate limit hit for alias_create_900d:976 (bucket id 1783491300) -> 11/10
2026-07-08 06:23:21,369 - SL - WARNING - 13978 - "/code/server.py:364" - rate_limited() -  - Client hit rate limit on path /api/v2/alias/custom/new, user:<User 976 T user_3wpq8wq0me@mailbox.test>
2026-07-08 06:23:21,369 - SL - DEBUG - 13978 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 429, takes 0.03307938575744629
  POST #10: STATUS 429  <-- BREACH
  FULL RESPONSE-HEADERS:
      Content-Type: application/json
      Content-Length: 32
      Access-Control-Allow-Origin: *
      Set-Cookie: slapp=<redacted>; Domain=.sl.test; Expires=Wed, 15-Jul-2026 06:23:21 GMT; HttpOnly; Path=/; SameSite=Lax
  FULL BODY [32 bytes, CT='application/json']: {"error":"Rate limit exceeded"}
```

RUN2 reproduced both breach points identically — PAID `-> 51/50` at POST #50 and FREE `-> 11/10` at POST #10 (same `LOG.i` at `rate_limiter.py:33`, same 32-byte JSON `429`).

### F.6 Condition 10 — Flask-Limiter burst over a REAL gunicorn server

The complete burst block (six identical real HTTP POSTs, full `curl -D -` headers + bodies, and the gunicorn `SL` rate-limit log line) is embedded in §E.3. It shows `201, 409, 409, 409, 409, 429`, with the `429` carrying **no** rate-limit headers.

### F.7 Repository unchanged — final `git status`

The working tree is clean and the only path that differs from the pre-task baseline commit `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c` is this document. Captured on the assigned branch after authoring and committing:

```text
$ git rev-parse --abbrev-ref HEAD
blitzy-45a67586-f857-4a8e-ba25-5dedc7607a35

$ git status --porcelain
(no output — clean working tree)

$ git diff --name-status 2cd6ee777f8c2d3531559588bcfb18627ffb5d2c..HEAD
A	blitzy/documentation/app_2cd6ee777f8c.md
```

The single added path is `blitzy/documentation/app_2cd6ee777f8c.md` (this file), committed on the branch by `agent@blitzy.com`. No source, test, configuration, or dependency file was added, modified, or deleted, and no probe/evidence artifact exists anywhere under the repository tree (all lived under host `/tmp/probes/` and were removed). This satisfies the read-only constraint: *"keep the codebase unchanged."*

---

## §G — Coverage Pass

Every named/required item from the question, confirmed against observed evidence:

- [x] **O1 — invalid/expired signed suffixes: exact status codes, error messages, validation log entries.** Expired → `412 {"error":"Alias creation time is expired, please retry"}` + `LOG.w` at `new_custom_alias.py:72` (v2) / `:187` (v3); garbage/empty → same `412`; captured verbatim for **both v2 and v3** in §A.4. Status/message/log all shown.
- [x] **O1 — tampered-vs-expired mislabel (headline anomaly).** Reproduced on **both** endpoints: a tampered suffix returns `412` (not the `400 "Tampered suffix"` the code appears to intend), because `check_suffix_signature` (`alias_suffix.py:37-42`) catches the base `itsdangerous.BadSignature` and returns `None` for every failure mode; `raw check_suffix_signature(tampered)=None` observed directly (§A.2–A.3). The `except Exception → 400` branch (`new_custom_alias.py:74-76` / `:190-191`) is **effectively dead** for ordinary bad tokens. **Documented, not fixed** (per the read-only + no-remediation rules).
- [x] **O2 — rate-limiting headers: presence and values.** Shown **absent** via the complete header set on a `201` success and on **all three** distinct `429`s (Flask-Limiter burst, `parallel_limiter` concurrency, `Alias.create` token bucket), plus an explicit programmatic scan returning `NONE`. No `X-RateLimit-Limit/Remaining/Reset`, no `Retry-After` on any response. Root cause: `Limiter(key_func=__key_func)` with no `headers_enabled` (`extensions.py:23`) + Flask-Limiter 1.4 default `RATELIMIT_HEADERS_ENABLED=False` (§B).
- [x] **O3 — successful creation: quota checks and what each logs.** Two checks enumerated in order: (1) `User.can_create_new_alias()` count quota (`models.py:867-884`) — logs **nothing** on success, only `LOG.d "... cannot create any custom alias"` + `400` on breach (`new_custom_alias.py:49`); (2) `Alias.create()` per-user token bucket via `rate_limiter.check_bucket_limit` (`models.py:1641`, `rate_limiter.py:19-42`) — logs **nothing** on success, only `LOG.i "Rate limit hit ..."` + `429` on breach (`rate_limiter.py:33`). Success emits audit `"New alias created"` (`models.py:1688-1689`), **observed as a real `AliasAuditLog` row** (`action='create'`) after a `201` (§C.3, §F.3). Observed `Alias.filter_by(user_id).count()` before/after each creation (`1→2→3`) captured in §C.1, noting the auto-created newsletter alias at signup.
- [x] **O4 — execution-path trace.** Full ordered decorator/validation chain documented for both `POST /api/v2/alias/custom/new` and `POST /api/v3/...` (v3 differences **observed**, §D.2), the four creation-limit enforcers named, the rejection-condition table (status/message/log/`file:line`), the Mermaid flowchart, and the dashboard sibling note (§D).
- [x] **Intermittency — reproduced on identical input with observed distribution + code-level causes.** Deterministic baseline (RUN1 vs RUN2, byte-identical distribution `6×201, 3×400, 2×409, 16×412, 2×429`) plus the four causes each with citation and evidence: 600 s suffix window, `parallel_limiter` `429`, Flask-Limiter `5/minute` rolling window (authoritative gunicorn capture), and the Redis-availability gate that silently disables the bucket + concurrency limiters (§E).
- [x] **Canonical config + exact commands + ≥2 runs + full unedited outputs.** Effective config stated (`MAX_NB_EMAIL_FREE_PLAN=3`, `MEM_STORE_URI=redis://localhost`, `DISABLE_RATE_LIMIT`/`DISABLE_ALIAS_SUFFIX` unset); exact build/invocation commands in §F.1; every condition run twice with the distribution reported; complete unedited output blocks throughout §A–§F, with only session-cookie values redacted. The single deliberate deviation from canonical config — Condition 8's in-process `MAX_NB_EMAIL_FREE_PLAN=1000` raise, required so the count quota does not mask the token bucket in the full 49-POST run — is disclosed at its point of use (§F.5, §F.1) and explained in §C.2; every other condition ran under the canonical cap of `3`.
- [x] **Repo unchanged; temporary scripts removed.** Final `git status --porcelain` is empty and `git diff --name-status <baseline>..HEAD` shows only the new `blitzy/documentation/app_2cd6ee777f8c.md`; all observation scripts and evidence files lived under host `/tmp/probes/` and were deleted (§F.7).

### G.1 What "doesn't match expected behavior" actually is — one-paragraph synthesis

The reported *"intermittent validation failures that don't match the expected behavior"* has two distinct, run-verified explanations that compound each other. **First (the mislabel):** because `check_suffix_signature` swallows the whole `itsdangerous.BadSignature` subtree, a suffix that is tampered, truncated, garbage, or empty is reported with the *expired* message and a `412` — so a developer inspecting a genuinely malformed request sees "creation time is expired" and reasonably concludes the token timed out, when in fact it never validated. **Second (the true intermittency):** an *identical, correctly-signed* request legitimately flips outcome between attempts because of time- and concurrency-sensitive gates that are invisible in the request body — the 600-second suffix window (`412` once the token ages out), the `parallel_limiter` concurrency lock (`429` on overlapping/double-submitted requests), the Flask-Limiter `5/minute` rolling window that counts **before** the handler runs and accumulates across processes through shared Redis (`429`), and the Redis-availability gate that silently switches the bucket and concurrency limiters on or off depending on whether `lock_redis` is wired. Together these produce the "works sometimes, fails sometimes, with a confusing error" experience the user described — all without any code defect in the *limiters themselves*, and with the single genuine bug being the suffix-error mislabel, which this document diagnoses but (per scope) does not remediate.
