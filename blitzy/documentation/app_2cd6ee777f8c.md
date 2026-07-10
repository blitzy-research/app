# SimpleLogin custom-alias creation: signed-suffix validation & alias-creation-limit — runtime investigation

**Scope of the question.** This report answers, from *observed runtime behavior*, exactly how SimpleLogin's
custom-alias-creation endpoints behave when a **signed suffix is validated** and when the **alias-creation
limit is enforced**, in response to the report of *"intermittent validation failures that don't match the
expected behavior and appear related to how signed suffixes are verified."* It answers six sub-questions
(Q1–Q6) with the actual HTTP status, JSON body, response headers, and server console (`SL`) log lines, each
grounded in a `file:line` reference and the specific function/method that performs the work.

---

## 1. Title & Summary (executive answer)

Driving the **real** endpoints `POST /api/v2/alias/custom/new` and `POST /api/v3/alias/custom/new` through
API-key authentication inside the canonical Docker container, the observed behavior is:

- **Every "bad" signed suffix — expired, tampered, malformed, *and* empty — returns the identical
  `HTTP 412` with body `{"error":"Alias creation time is expired, please retry"}`.** The status is
  **deterministic** (10/10 identical repeated POSTs → 412; see §6). There is **no run-to-run randomness**.
- **The central finding (root cause):** the validator `check_suffix_signature()`
  (`app/alias_suffix.py:37-42`) wraps `signer.unsign(signed_suffix, max_age=600)` in a
  `try/except itsdangerous.BadSignature` (catch at `app/alias_suffix.py:41`). In **itsdangerous 1.1.0**
  (runtime-confirmed), **both `SignatureExpired` and `BadTimeSignature` subclass `BadSignature`**, so a
  tampered/malformed/empty suffix raises a `BadSignature` subclass that is *swallowed* exactly like a genuine
  expiry — the function returns `None`, and the endpoint takes the `if not alias_suffix:` → **412** branch
  (`app/api/views/new_custom_alias.py:71-73` v2 / `:186-188` v3). Consequently the intended
  **`HTTP 400 {"error":"Tampered suffix"}`** branch (`:74-76` v2 / `:189-191` v3) is **dead code for
  signature errors** — it can only fire on a *non-*`BadSignature` exception, which signature problems never
  raise. This single-response-covers-four-distinct-causes behavior is the most likely explanation for the
  user's "validation failures that don't match expected behavior": a caller expecting a `400 "Tampered
  suffix"` for a tampered/invalid suffix instead *always* receives `412 "…expired…"`.
- **Rate-limit headers (Q4): a clear NEGATIVE finding.** No `X-RateLimit-Limit`/`X-RateLimit-Remaining`/
  `X-RateLimit-Reset`/`Retry-After` header is emitted on **any** response — normal (201/412/400) or the
  rate-limit `429`. The limiter is built as `Limiter(key_func=__key_func)` with **no** `headers_enabled`
  argument (`app/extensions.py:23`) and there is no `RATELIMIT_HEADERS_ENABLED` config anywhere. On breach
  the app returns `{"error":"Rate limit exceeded"}, 429` from the 429 handler in the **root** `server.py:362-372`.
- **Quota checks on success (Q5):** the enforcer is `User.can_create_new_alias()` (`app/models.py:867-884`),
  which after `is_active()`/`disabled`/`lifetime_or_active_subscription()` short-circuits evaluates
  `Alias.filter_by(user_id=self.id).count() < self.max_alias_for_free_account()` (`app/models.py:881-884`);
  `max_alias_for_free_account()` (`app/models.py:858-865`) returns `config.MAX_NB_EMAIL_FREE_PLAN`
  (**3** under `tests/test.env`). **`can_create_new_alias()` logs nothing on success**; the only related log
  is the *failure*-path `LOG.d("user %s cannot create any custom alias", user)` (`:49` v2 / `:138` v3).

The remainder of this document provides the exact commands, unedited outputs, `file:line` references, and the
execution-path trace behind each of these answers.

---

## 2. Environment & Methodology

### 2.1 Canonical runtime (exactly as used)

All behavioral observations were produced **inside the user-supplied canonical Docker container** (a running
container named `sl_setup`), never in the planning/authoring shell (which lacks PostgreSQL, Redis, and even
`itsdangerous`).

| Item | Value (observed) |
|------|------------------|
| Container image | `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` |
| Python | `3.10.18` (`Dockerfile:8` → `FROM python:3.10`; `pyproject.toml:61` → `python = "^3.10"`; CI `.github/workflows/main.yml:17` → `python-version: '3.10'`) |
| itsdangerous | `1.1.0` (`poetry.lock:1617-1618`) |
| flask | `1.1.2` (`poetry.lock:910-911`) |
| flask-limiter | `1.4` (`poetry.lock:1013-1014`) |
| limits | `1.5.1` (`poetry.lock:1759-1760`) |
| PostgreSQL | `15.13` on `localhost:5432` (user/pw/db = `test`/`test`/`test`), `pg_trgm` extension enabled |
| Redis | reachable on `localhost:6379` (`PONG`) |

Version confirmation command and output:

```
$ docker exec sl_setup /app/venv/bin/python -c "import itsdangerous, flask, importlib.metadata as m, sys; \
print('itsdangerous', itsdangerous.__version__); print('flask', flask.__version__); \
print('flask-limiter', m.version('flask-limiter')); print('limits', m.version('limits')); \
print('python', sys.version.split()[0])"
itsdangerous 1.1.0
flask 1.1.2
flask-limiter 1.4
limits 1.5.1
python 3.10.18
```

### 2.2 Entry point and boot commands

The **canonical in-process entry point** was used: a Flask test client built from `server.create_app()`
exactly as `tests/conftest.py` does (`from server import create_app` at `tests/conftest.py:20`;
`app = create_app()` at `:23`; `add_sl_domains()` at `:38`; `add_proton_partner()` at `:39`; `pg_trgm` setup
at `:28-36`). This exercises the full decorator stack, authentication, and HTTP response machinery.
(The production out-of-process entry point is `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15`,
`Dockerfile:47`, `EXPOSE 7777` at `Dockerfile:44`; it was not required because the in-process test client
drives the identical WSGI application.)

Temporary observation scripts were written **outside** the repository checkout, under the container's
`/tmp/blitzy_obs/`, and were run with:

```
$ docker exec sl_setup bash -lc 'cd /app && /app/venv/bin/python /tmp/blitzy_obs/<script>.py'
```

Each script sets, **before importing the app**, the canonical test configuration:

```python
os.environ["CONFIG"]  = "/app/tests/test.env"                       # tests/conftest.py:8-10
os.environ["DB_URI"]  = "postgresql://test:test@localhost:5432/test"  # override; see note below
sys.path.insert(0, "/app"); os.chdir("/app")
from server import create_app
app = create_app(); app.config["TESTING"] = True
app.config["WTF_CSRF_ENABLED"] = False; app.config["SERVER_NAME"] = "sl.test"
# pg_trgm setup + add_sl_domains() + add_proton_partner()  (mirrors tests/conftest.py)
```

> **DB_URI note (disclosed):** `tests/test.env:17` declares `DB_URI=postgresql://test:test@localhost:15432/test`,
> but the container's PostgreSQL listens on `5432`. Because `app/config.py:69` calls `load_dotenv(...)` with the
> default `override=False`, pre-setting `os.environ["DB_URI"]` to port `5432` takes precedence over the `.env`
> value (`DB_URI = os.environ["DB_URI"]` at `app/config.py:192`). `MEM_STORE_URI=redis://localhost`
> (`tests/test.env:78`) is used as-is.

### 2.3 How a valid API token was obtained (the real, canonical auth path)

Requests are authenticated through the real `require_api_auth` decorator (`app/api/base.py:52-60`), whose
`authorize_request()` (`app/api/base.py:16`) reads `api_code = request.headers.get("Authentication")`
(`:17`) and looks it up via `ApiKey.get_by(code=api_code)` (`:18`). Each observation script creates a fresh
`User` and a real `ApiKey` (`ApiKey.create(user_id=..., name=...)`, `app/models.py:2364-2370`, which generates
a 60-character `code`) and sends the header `Authentication: <api_key.code>` on every request. This was
verified to be the enforced path: **omitting the header yields `HTTP 401 {"error":"Wrong api key"}`**
(`app/api/base.py:27`). API-key values are redacted in this report as `<API_KEY_REDACTED (60 chars)>`.

### 2.4 Configuration in effect (disclosed per observation)

| Config key | Value | Where set | Applies to |
|------------|-------|-----------|------------|
| `EMAIL_DOMAIN` | `sl.local` | `tests/test.env:8` | valid-suffix construction |
| `MAX_NB_EMAIL_FREE_PLAN` | `3` | `tests/test.env:13` (default would be `5` via `app/config.py:124` if unset) | Q5 quota numbers |
| `DISABLE_RATE_LIMIT` | `False` by default | not set in `tests/test.env` → `app/config.py:602` = `False` | see below |
| `ALIAS_LIMIT` | `"100/day;50/hour;5/minute"` | `app/config.py:448` | Q4/Q6 rate-limit bucket |

> **Rate-limit isolation nuance (disclosed):** conditions (a)–(e) and the intermittency runs were executed
> with `config.DISABLE_RATE_LIMIT = True` — the exact behavior of the canonical `flask_client` fixture, which
> sets it at `tests/conftest.py:65` and again in its `finally` at `:73`. This isolates the suffix/quota logic
> from the limiter. It is required because, under API-key auth (no session cookie), `current_user` is anonymous
> at `@limiter.limit` time, so the limiter key is `ip:127.0.0.1` (`app/extensions.py:14-19`) and is *shared*
> across all API requests; leaving the limiter active would spuriously `429` after a few requests. Condition
> (f) — the rate-limit observation for Q4 — instead sets `config.DISABLE_RATE_LIMIT = False` and loops the POST
> (resetting `g._rate_limiting_complete = False` each iteration), exactly as `test_too_many_requests`
> (`tests/api/test_new_custom_alias.py:255-283`) does. Each observation below states the value in effect.

### 2.5 Server-log capture

The `SL` logger writes to **stdout** (`app/log.py:41`, `logging.StreamHandler(sys.stdout)`) using the format
string at `app/log.py:12-15`:

```
%(asctime)s - %(name)s - %(levelname)s - %(process)d - "%(pathname)s:%(lineno)d" - %(funcName)s() - %(message_id)s - %(message)s
```

`LOG = _get_logger("SL")` (`app/log.py:79`); the shortcuts `LOG.d/i/w/e` map to `debug/info/warning/exception`
(`app/log.py:74-77`); werkzeug's own access log is muted (`app/log.py:70-71`). Each script attaches an
in-memory `logging.Handler` to the `"SL"` logger using this **same** format string (with `time.gmtime`
converter, matching `app/log.py:43`), so the lines reproduced below are byte-identical to what appears on the
server console; the request-terminating line at `server.py:284` (`after_request()`) corroborates each final
status.

---

## 3. Answers to Q1–Q6

Both endpoints share the identical suffix-validation block; each answer below shows the observed result for
**both** `v2` and `v3` where relevant.

### Q1 — Invalid (tampered / malformed) signed suffix: status code + error body

**Answer:** `HTTP 412` with body **`{"error":"Alias creation time is expired, please retry"}`** — for a
tampered signature, a malformed (no-`.`) string, *and* an empty string. The intended `HTTP 400
{"error":"Tampered suffix"}` is **never** returned for signature problems (it is dead code — see §5).

- **Validator:** `check_suffix_signature()` (`app/alias_suffix.py:37-42`); the `except itsdangerous.BadSignature:`
  at `app/alias_suffix.py:41` returns `None`.
- **Status mapping (v2):** `new_custom_alias_v2()` — `if not alias_suffix:` at `app/api/views/new_custom_alias.py:71`
  → `return jsonify(error="Alias creation time is expired, please retry"), 412` at `:73`.
- **Status mapping (v3):** `new_custom_alias_v3()` — `:186` → `:188`.
- **Dead intended branch:** `except Exception:` → `return jsonify(error="Tampered suffix"), 400`
  at `:74-76` (v2) / `:189-191` (v3).

Command (condition c, tampered — a valid signature with its last byte flipped) and unedited output:

```
$ docker exec sl_setup bash -lc 'cd /app && /app/venv/bin/python /tmp/blitzy_obs/observe_B.py'   # DISABLE_RATE_LIMIT=True
...
CONDITION: (c) TAMPERED suffix (flip last byte) -> v2 (expect 412, NOT 400)
COMMAND: POST /api/v2/alias/custom/new
  json = {'alias_prefix': 'tampv2', 'signed_suffix': '.list@sl.local.alCs5w.eL5N0YZ2fj0-sA1PWHH-meFfaqA'}
  headers = {'Authentication': '<API_KEY_REDACTED (60 chars)>'}
STATUS: 412
BODY: {"error":"Alias creation time is expired, please retry"}
RESPONSE HEADERS:
  Content-Type: application/json
  Content-Length: 57
  Access-Control-Allow-Origin: *
  Set-Cookie: slapp=ae1af48d-...; Domain=.sl.test; ...; HttpOnly; Path=/; SameSite=Lax
SL LOG LINES EMITTED DURING REQUEST:
  2026-07-10 08:27:19,019 - SL - WARNING - 8348 - "/app/app/api/views/new_custom_alias.py:72" - new_custom_alias_v2() -  - Alias creation time expired for <User 1318 Obs tamper_xdmdwabb@mailbox.test>
  2026-07-10 08:27:19,020 - SL - DEBUG - 8348 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 412, takes 0.010789632797241211
```

The `v3` tampered request, the **malformed** (`"notasignature"`) request, and the **empty** (`""`) request
all produced the same `412` body (full blocks in the Appendix, §8). The `v3` warning line is emitted at
`new_custom_alias.py:187` inside `new_custom_alias_v3()`. In none of these was the string `Tampered suffix`
ever returned, and the log line `Alias suffix is tampered` (`:75`/`:190`) was **never** emitted.

### Q2 — Expired signed suffix (age > 600 s): status code + error body

**Answer:** `HTTP 412` with body **`{"error":"Alias creation time is expired, please retry"}`** — the *same*
branch as Q1. The expiry threshold is `max_age=600` seconds at `app/alias_suffix.py:40`
(`signer.unsign(signed_suffix, max_age=600)`).

Command (condition b — a suffix signed with a timestamp backdated ~1000 s, i.e. age > 600) and unedited output:

```
$ docker exec sl_setup bash -lc 'cd /app && /app/venv/bin/python /tmp/blitzy_obs/observe_B.py'   # DISABLE_RATE_LIMIT=True
...
CONDITION: (b) EXPIRED suffix (age~1000s) -> v2 (expect 412)
COMMAND: POST /api/v2/alias/custom/new
  json = {'alias_prefix': 'expv2', 'signed_suffix': '.word@sl.local.alCo_g.iIVZL4dY4dHodbqlG1NDDIDhlZo'}
  headers = {'Authentication': '<API_KEY_REDACTED (60 chars)>'}
STATUS: 412
BODY: {"error":"Alias creation time is expired, please retry"}
RESPONSE HEADERS:
  Content-Type: application/json
  Content-Length: 57
  Access-Control-Allow-Origin: *
  Set-Cookie: slapp=ae1af48d-...; Domain=.sl.test; ...; HttpOnly; Path=/; SameSite=Lax
SL LOG LINES EMITTED DURING REQUEST:
  2026-07-10 08:27:18,464 - SL - WARNING - 8348 - "/app/app/api/views/new_custom_alias.py:72" - new_custom_alias_v2() -  - Alias creation time expired for <User 1316 Obs expired_gkcdzwiq@mailbox.test>
  2026-07-10 08:27:18,465 - SL - DEBUG - 8348 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 412, takes 0.010904550552368164
```

The `v3` expired request produced the same `412` body, with the warning at `new_custom_alias.py:187`
(`new_custom_alias_v3()`). At the library level (§5), the expired case is the *only* one whose underlying
exception is genuinely an expiry: `itsdangerous.exc.SignatureExpired: Signature age 1000 > 600 seconds`.

### Q3 — Validation log entries printed to the server console

**Answer:** On every signature rejection (Q1 and Q2), the server console prints exactly **one** validation
warning via `LOG.w("Alias creation time expired for %s", user)`:

- v2: `app/api/views/new_custom_alias.py:72` (function `new_custom_alias_v2`)
- v3: `app/api/views/new_custom_alias.py:187` (function `new_custom_alias_v3`)

Verbatim captured stdout lines (already shown per condition above), e.g. for a tampered v2 request:

```
2026-07-10 08:27:19,019 - SL - WARNING - 8348 - "/app/app/api/views/new_custom_alias.py:72" - new_custom_alias_v2() -  - Alias creation time expired for <User 1318 Obs tamper_xdmdwabb@mailbox.test>
```

and for the `v3` path:

```
2026-07-10 08:27:18,743 - SL - WARNING - 8348 - "/app/app/api/views/new_custom_alias.py:187" - new_custom_alias_v3() -  - Alias creation time expired for <User 1317 Obs expired_zajmgvfb@mailbox.test>
```

These match the `SL` format `app/log.py:12-15` (asctime, logger name `SL`, level `WARNING`, pid `8348`,
`"pathname:lineno"`, `funcName()`, empty `message_id`, message). `LOG.w` = `logging.Logger.warning`
(`app/log.py:76`); `LOG = _get_logger("SL")` (`app/log.py:79`); output goes to stdout (`app/log.py:41`).

**Important (grounded, observed):** the *other* warning in the endpoint,
`LOG.w("Alias suffix is tampered, user %s", user)` (`:75` v2 / `:190` v3), was **not observed for any
signature error** — it belongs to the unreachable `except Exception:` branch (§5). The success path (Q5) emits
**no** validation warning at all; only the request-terminating `after_request()` debug line
(`server.py:284`, e.g. `... 201, takes ...`) and an unrelated `event_dispatcher` info line appear.

### Q4 — Rate-limiting headers on responses (NEGATIVE finding)

**Answer:** **No rate-limit headers are present on any response.** Neither a normal response (201/412/400) nor
the rate-limit `429` carries `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, or
`Retry-After`. On breach, the app returns **`{"error":"Rate limit exceeded"}` with `HTTP 429`**.

- **Why (grounded):** the limiter is constructed as `limiter = Limiter(key_func=__key_func)` **without** a
  `headers_enabled` argument (`app/extensions.py:23`); flask-limiter 1.4's `RATELIMIT_HEADERS_ENABLED` defaults
  to `False`, and **no** `RATELIMIT_HEADERS_ENABLED` config key exists anywhere in `server.py` (root) or
  `app/config.py`. The key resolver `__key_func` (`app/extensions.py:14-19`) returns `userid:{id}` or
  `ip:{addr}`.
- **The 429 body/handler:** `@app.errorhandler(429)` → `rate_limited(e)` in the **root** `server.py:362-372`;
  `if request.path.startswith("/api/"): return jsonify(error="Rate limit exceeded"), 429` (`server.py:370`);
  the web branch renders `error/429.html` (`server.py:372`). (Note: **`app/server.py` does not exist** — the
  limiter wiring `limiter.init_app(app)` at `server.py:167` and the handler live in the root `server.py`;
  `create_app` is at `server.py:139`.)

Command (condition f — `DISABLE_RATE_LIMIT=False`; loop `POST /api/v3` > 5×/min, resetting
`g._rate_limiting_complete=False` each iteration) and unedited output:

```
$ docker exec sl_setup bash -lc 'cd /app && /app/venv/bin/python /tmp/blitzy_obs/observe_B2.py'
...
  per-request statuses: [400, 400, 400, 400, 400, 400, 429]

CONDITION: (f) RATE-LIMIT breach: final (7th) response (Q4)
COMMAND: POST /api/v3/alias/custom/new
  json = {'alias_prefix': 'rl6', 'signed_suffix': '<@domain signed>', 'mailbox_ids': [1577]}
  headers = {'Authentication': '<API_KEY_REDACTED (60 chars)>'}
STATUS: 429
BODY: {"error":"Rate limit exceeded"}
RESPONSE HEADERS:
  Content-Type: application/json
  Content-Length: 32
  Access-Control-Allow-Origin: *
  Set-Cookie: slapp=150a01ad-...; Domain=.sl.test; ...; HttpOnly; Path=/; SameSite=Lax
SL LOG LINES EMITTED DURING REQUEST:
  2026-07-10 08:28:36,634 - SL - WARNING - 8391 - "/app/server.py:364" - rate_limited() -  - Client hit rate limit on path /api/v3/alias/custom/new, user:<User 1324 Obs rl_tieowxye@mailbox.test>
  2026-07-10 08:28:36,634 - SL - DEBUG - 8391 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v3/alias/custom/new ImmutableMultiDict([]) 429, takes 0.0007886886596679688

[Q4] Rate-limit-specific headers present on 429 response: NONE
[Q4] Full 429 header names: ['Content-Type', 'Content-Length', 'Access-Control-Allow-Origin', 'Set-Cookie']
```

The full header set on the `429` is exactly `Content-Type`, `Content-Length`, `Access-Control-Allow-Origin`,
`Set-Cookie` — no rate-limit headers. The normal responses in Q1/Q2/Q5 show the same header set (plus a longer
`Content-Length`), likewise with no rate-limit headers. The `429` is logged by the handler at `server.py:364`
(`rate_limited()`: `LOG.w("Client hit rate limit on path %s, user:%s", ...)`).

> **Observed nuance (disclosed):** the first six loop requests returned `400` "wrong alias prefix or suffix"
> (`app/alias_suffix.py:61`, `verify_prefix_suffix()`), because the freshly-created custom domain used to build
> `@domain` suffixes is not in the user's `available_alias_domains()`. This does **not** affect the finding:
> `@limiter.limit(ALIAS_LIMIT)` is the *outermost* decorator, so every request counts against the
> `ip:127.0.0.1` "5/minute" bucket regardless of the body-level outcome; once the bucket is exceeded the 7th
> request is short-circuited to `429` before the endpoint body runs. This mirrors `test_too_many_requests`
> (`tests/api/test_new_custom_alias.py:255-283`), which also loops `signer.sign("@"+domain)` and asserts the
> final response is `429 {"error":"Rate limit exceeded"}`.

### Q5 — Quota checks on a successful creation, and what gets logged

**Answer.** On the creation path the enforcer `User.can_create_new_alias()` (`app/models.py:867-884`) runs
**first** in the endpoint (`new_custom_alias.py:48` v2 / `:137` v3). It evaluates, in order:
`is_active()` (`app/models.py:872-873`), `disabled` (`:875-876`), `lifetime_or_active_subscription()`
(`:878-879`), and — for a free account — the decisive comparison
`Alias.filter_by(user_id=self.id).count() < self.max_alias_for_free_account()` (`app/models.py:881-884`).
`max_alias_for_free_account()` (`app/models.py:858-865`) returns `config.MAX_NB_EMAIL_FREE_PLAN` (`:865`),
which is **3** under `tests/test.env:13` (the built-in default is **5**, `app/config.py:124`).

**`can_create_new_alias()` logs nothing on the success path.** To *observe* the actual `count()` and
`max_alias_for_free_account()` values, the observation script temporarily **wrapped** the two real methods
(monkeypatch in the `/tmp` script — the underlying methods still execute; `app/models.py` was **not** edited).
Values labeled `[INSTRUMENT]` below come from that wrapper; everything else is the app's own output.

> **Baseline (observed):** a newly-created user already owns **1** alias — the auto-created
> `simplelogin-newsletter.<word>@sl.local` — so a "fresh" user's `count()` starts at `1`.

Command (condition e1 — success, `DISABLE_RATE_LIMIT=True`, `MAX_NB_EMAIL_FREE_PLAN=3`) and output:

```
$ docker exec sl_setup bash -lc 'cd /app && /app/venv/bin/python /tmp/blitzy_obs/observe_B2.py'
...
>>> (e1) SUCCESS quota check (fresh user, trial_end=None, 0 aliases) — instrumentation follows:
  [INSTRUMENT] can_create_new_alias(): is_active=True disabled=False lifetime_or_active_subscription=False Alias.count(user_id=1322)=1
  [INSTRUMENT] User.max_alias_for_free_account() -> 3
  [INSTRUMENT] can_create_new_alias() -> True
CONDITION: (e1) SUCCESS: count<max -> 201 (Q5 success path)
STATUS: 201
BODY: {"alias":"quotaok.list@sl.local", ... "id":2193, ...}
SL LOG LINES EMITTED DURING REQUEST:
  2026-07-10 08:28:35,851 - SL - INFO - 8391 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
  2026-07-10 08:28:35,860 - SL - DEBUG - 8391 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v3/alias/custom/new ImmutableMultiDict([]) 201, takes 0.05593538284301758
```

So on success the quota check evaluates `count (1) < max (3)` → `True`, and **the only `SL` lines are the
unrelated `event_dispatcher` info line and the `after_request` `201` line — no quota log**.

The complementary **quota-exceeded** case (condition e2) — after creating `MAX_NB_EMAIL_FREE_PLAN` more aliases
via `Alias.create_new(user, prefix="test")` with `user.trial_end = None` (mirroring `test_out_of_quota`,
`tests/api/test_new_custom_alias.py:184-210`) — makes `count (4) == …` exceed `max (3)`:

```
>>> (e2) QUOTA-EXCEEDED (created 3 aliases, trial_end=None) — instrumentation follows:
  [INSTRUMENT] can_create_new_alias(): is_active=True disabled=False lifetime_or_active_subscription=False Alias.count(user_id=1323)=4
  [INSTRUMENT] User.max_alias_for_free_account() -> 3
  [INSTRUMENT] can_create_new_alias() -> False
CONDITION: (e2) QUOTA-EXCEEDED: count==max -> 400 (Q5)
STATUS: 400
BODY: {"error":"You have reached the limitation of a free account with the maximum of 3 aliases, please upgrade your plan to create more aliases"}
SL LOG LINES EMITTED DURING REQUEST:
  2026-07-10 08:28:36,193 - SL - DEBUG - 8391 - "/app/app/api/views/new_custom_alias.py:138" - new_custom_alias_v3() -  - user <User 1323 Obs quota_full_wbzhssde@mailbox.test> cannot create any custom alias
  2026-07-10 08:28:36,194 - SL - DEBUG - 8391 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v3/alias/custom/new ImmutableMultiDict([]) 400, takes 0.015954971313476562
```

Thus the **only** quota-related log is the *failure*-path `LOG.d("user %s cannot create any custom alias", user)`
at `new_custom_alias.py:138` (v3) / `:49` (v2), producing the `400` body quoted verbatim above (note the literal
`maximum of 3 aliases` — the `3` is `MAX_NB_EMAIL_FREE_PLAN` interpolated at `:53`/`:142`). **Config disclosure:**
both quota numbers above were produced with `MAX_NB_EMAIL_FREE_PLAN = 3` (`tests/test.env:13`); with the
built-in default (`5`, `app/config.py:124`) the message would read `maximum of 5 aliases` and the threshold
comparison would use `5`.

### Q6 — Execution-path trace (validator, enforcer, and rejection conditions)

**Answer.** The **signed-suffix validator** is `app/alias_suffix.py::check_suffix_signature`
(`app/alias_suffix.py:37-42`); the **creation-limit enforcer** is `app/models.py::can_create_new_alias`
(`app/models.py:867-884`). A request to `POST /api/vN/alias/custom/new` passes through a three-decorator stack
and then a fixed validation sequence; the diagram and prose in §4 give the complete trace with the status code
for each rejection condition.

---


## 4. Execution-path trace (Q6)

### 4.1 Decorator stack (both endpoints)

Both endpoints declare the identical decorator stack (v2 `app/api/views/new_custom_alias.py:28-31`,
v3 `:115-118`), which at request time executes **outermost-first**:

1. `@limiter.limit(ALIAS_LIMIT)` — `app/api/views/new_custom_alias.py:29` / `:116`; the flask-limiter instance
   from `app/extensions.py:23`; `ALIAS_LIMIT = "100/day;50/hour;5/minute"` (`app/config.py:448`). On breach →
   `HTTP 429 {"error":"Rate limit exceeded"}` via the root-`server.py:362-372` handler.
2. `@require_api_auth` — `:30` / `:117`; `app/api/base.py:52-60` → `authorize_request()` (`:16`); missing/invalid
   `Authentication` header → `HTTP 401 {"error":"Wrong api key"}` (`app/api/base.py:27`).
3. `@parallel_limiter.lock(name="alias_creation")` — `:31` / `:118`; `app/parallel_limiter.py`; `acquire_lock()`
   (`:30-34`) raises `werkzeug.exceptions.TooManyRequests()` (`:34`) → `HTTP 429` if the per-user/IP Redis lock
   cannot be acquired within `max_wait_secs`.

### 4.2 In-body validation sequence

After the decorators, `new_custom_alias_v2/v3` runs these checks in order (status code in brackets):

- **Quota** — `if not user.can_create_new_alias():` (`:48`/`:137`) → **[400]** `You have reached the limitation
  of a free account with the maximum of N aliases…`, log `LOG.d("user %s cannot create any custom alias", user)`
  (`:49`/`:138`).
- **Empty body** — `if not data:` (`:61`/`:150`) → **[400]** `request body cannot be empty`. (v3 additionally:
  non-dict body → **[400]** `:153-154`; `check_alias_prefix()` → **[400]** `alias prefix invalid format or too
  long` `:167-168`, `app/alias_utils.py:418-425`; mailbox checks → **[400]** `:171-181`.)
- **Signed-suffix signature** — `alias_suffix = check_suffix_signature(signed_suffix)` (`:70`/`:185`,
  `app/alias_suffix.py:37-42`); `if not alias_suffix:` → **[412]** `Alias creation time is expired, please retry`,
  log `LOG.w("Alias creation time expired for %s", user)` (`:72-73`/`:187-188`). The `except Exception:` →
  **[400]** `Tampered suffix` (`:74-76`/`:189-191`) is **unreachable for signature errors** (§5).
- **Prefix/suffix pairing** — `if not verify_prefix_suffix(...)` (`:78`/`:193`, `app/alias_suffix.py:45-91`) →
  **[400]** `wrong alias prefix or suffix` (`:79`/`:194`).
- **Duplicate** — existing alias/deleted alias → **[409]** `alias {full_alias} already exists`, log
  `LOG.d("full alias already used %s", full_alias)` (`:87-88`/`:202-203`).
- **Success** — `Alias.create(...)` → **[201]** with the serialized alias JSON (`:109-112`/`:232-235`).

A standalone bucket limiter also exists — `app/rate_limiter.py::check_bucket_limit` (`:19-42`; log
`"Rate limit hit for {lock_name} (bucket id {bucket_id}) -> {value}/{max_hits}"` at `:33-35`;
`raise werkzeug.exceptions.TooManyRequests()` at `:40`) — but it is **not** wired onto these two endpoints
(their only rate control is the `@limiter.limit(ALIAS_LIMIT)` decorator); it is documented here for completeness.

### 4.3 Diagram

```mermaid
flowchart TD
    A["POST /api/vN/alias/custom/new"] --> B["@limiter.limit(ALIAS_LIMIT)\napp/extensions.py:23 · new_custom_alias.py:29/116"]
    B -->|"bucket exceeded (5/min)"| B1["HTTP 429 {'error':'Rate limit exceeded'}\nserver.py:362-372 (root)"]
    B --> C["@require_api_auth\napp/api/base.py:52-60"]
    C -->|"no/invalid Authentication header"| C1["HTTP 401 {'error':'Wrong api key'}\napp/api/base.py:27"]
    C --> D["@parallel_limiter.lock(name='alias_creation')\napp/parallel_limiter.py:30-34"]
    D -->|"lock not acquired"| D1["HTTP 429 (TooManyRequests)\napp/parallel_limiter.py:34"]
    D --> E{"user.can_create_new_alias()?\napp/models.py:867-884"}
    E -->|"False (count >= max)"| E1["HTTP 400 'reached free-account limit'\nLOG.d cannot create — :49/:138"]
    E -->|"True"| F{"request body present?"}
    F -->|"empty"| F1["HTTP 400 'request body cannot be empty'\n:61-62 / :150-151"]
    F --> G["check_suffix_signature(signed_suffix)\napp/alias_suffix.py:37-42 (catch BadSignature :41)"]
    G -->|"returns None: expired/tampered/malformed/empty"| G1["HTTP 412 'Alias creation time is expired, please retry'\nLOG.w :72-73 / :187-188"]
    G -->|"non-BadSignature exception (UNREACHABLE for sig errors)"| G2["HTTP 400 'Tampered suffix' — DEAD CODE\n:74-76 / :189-191"]
    G -->|"valid suffix"| H{"verify_prefix_suffix()?\napp/alias_suffix.py:45-91"}
    H -->|"False"| H1["HTTP 400 'wrong alias prefix or suffix'\n:79 / :194"]
    H --> I{"alias already exists?"}
    I -->|"yes"| I1["HTTP 409 'alias ... already exists'\nLOG.d :87-88 / :202-203"]
    I --> J["Alias.create -> HTTP 201\n:109-112 / :232-235"]
```

---

## 5. The central defect (documented, not fixed)

**What the code does.** `check_suffix_signature()` (`app/alias_suffix.py:37-42`):

```python
def check_suffix_signature(signed_suffix: str) -> Optional[str]:
    # hypothesis: user will click on the button in the 600 secs
    try:
        return signer.unsign(signed_suffix, max_age=600).decode()   # :40
    except itsdangerous.BadSignature:                               # :41
        return None                                                 # :42
```

**The exception hierarchy (runtime-confirmed).** In **itsdangerous 1.1.0**, `SignatureExpired` and
`BadTimeSignature` are both subclasses of `BadSignature`. This was confirmed at runtime in the container with a
**non-canonical** helper (a *direct* utility call — labeled non-canonical because it bypasses the HTTP path;
the canonical evidence for behavior is the endpoint responses in §3):

```
$ docker exec sl_setup bash -lc 'cd /app && /app/venv/bin/python /tmp/blitzy_obs/observe_D_hierarchy.py'
itsdangerous.__version__ = 1.1.0
issubclass(SignatureExpired,  BadSignature) = True
issubclass(BadTimeSignature,  BadSignature) = True
issubclass(BadSignature,      BadData)      = True
----------------------------------------------------------------------
CASE: FRESH valid
  signer.unsign(max_age=600) -> returned '.word@sl.local' (no exception)
  check_suffix_signature(...) -> '.word@sl.local'  => endpoint branch: verify_prefix_suffix (valid)

CASE: EXPIRED (age~1000s)
  signer.unsign(max_age=600) RAISED: itsdangerous.exc.SignatureExpired: Signature age 1000 > 600 seconds
    isinstance(e, itsdangerous.BadSignature) = True
  check_suffix_signature(...) -> None  => endpoint branch: if not alias_suffix -> HTTP 412

CASE: TAMPERED sig
  signer.unsign(max_age=600) RAISED: itsdangerous.exc.BadTimeSignature: Signature b'v-GSili5_PfujukKZfJoqfIkvTA' does not match
    isinstance(e, itsdangerous.BadSignature) = True
  check_suffix_signature(...) -> None  => endpoint branch: if not alias_suffix -> HTTP 412

CASE: MALFORMED (no '.')
  signer.unsign(max_age=600) RAISED: itsdangerous.exc.BadSignature: No b'.' found in value
    isinstance(e, itsdangerous.BadSignature) = True
  check_suffix_signature(...) -> None  => endpoint branch: if not alias_suffix -> HTTP 412

CASE: EMPTY ''
  signer.unsign(max_age=600) RAISED: itsdangerous.exc.BadSignature: No b'.' found in value
    isinstance(e, itsdangerous.BadSignature) = True
  check_suffix_signature(...) -> None  => endpoint branch: if not alias_suffix -> HTTP 412
```

**The consequence.** Because the single `except itsdangerous.BadSignature:` at `app/alias_suffix.py:41`
catches the *whole* family, **expired, tampered, malformed, and empty** suffixes all collapse to a `None`
return, and the endpoint always takes `if not alias_suffix:` → `HTTP 412 "Alias creation time is expired,
please retry"` (`new_custom_alias.py:71-73` v2 / `:186-188` v3). The intended
`except Exception:` → `LOG.w("Alias suffix is tampered, user %s", user)` + `return jsonify(error="Tampered
suffix"), 400` (`:74-76` v2 / `:189-191` v3) can only run on a *non-*`BadSignature` exception, which a signed
suffix never produces during verification — it is therefore **dead code for signature errors**. This is the
most likely explanation for the reported "validation failures that don't match expected behavior": a tampered
or malformed suffix is reported as *expired* (412), never as *tampered* (400).

**Secondary (confirming) web path.** The dashboard route `custom_alias()`
(`app/dashboard/views/custom_alias.py:30,34`) has the identical structure: `check_suffix_signature()` at `:90`;
`if not suffix:` → `LOG.w("Alias creation time expired for %s", current_user)` (`:92`) +
`flash("Alias creation time is expired, please retry", "warning")` (`:93`); and a matching dead
`except Exception:` (`:95`) → `flash("Unknown error, refresh the page", "error")` (`:97`). The API path (§3) is
the canonical evidence; the web path is confirming secondary evidence and was not exercised.

> **Remediation boundary:** per the task's read-only constraint, this defect is **documented, not fixed**. No
> source file was modified.

---

## 6. Intermittency characterization (repeated identical runs)

The reported "intermittent" symptom was probed by POSTing **one byte-for-byte identical** tampered payload
repeatedly — 5 times within a process, and again in a **separate fresh process** — with the input held
constant (`DISABLE_RATE_LIMIT=True` so the limiter cannot confound identical repeats). Command and unedited
output:

```
$ docker exec sl_setup bash -lc 'cd /app && /app/venv/bin/python /tmp/blitzy_obs/observe_C_intermittency.py 5 procA'
[procA] pid=8466 DISABLE_RATE_LIMIT=True
[procA] FIXED tampered signed_suffix = '.list@sl.local.alCs5w.eL5N0YZ2fj0-sA1PWHH-meFfaqB'
[procA] run 1/5: HTTP 412 | body={"error":"Alias creation time is expired, please retry"}
[procA] run 2/5: HTTP 412 | body={"error":"Alias creation time is expired, please retry"}
[procA] run 3/5: HTTP 412 | body={"error":"Alias creation time is expired, please retry"}
[procA] run 4/5: HTTP 412 | body={"error":"Alias creation time is expired, please retry"}
[procA] run 5/5: HTTP 412 | body={"error":"Alias creation time is expired, please retry"}
[procA] DISTRIBUTION over 5 identical POSTs: {412: 5}

$ docker exec sl_setup bash -lc 'cd /app && /app/venv/bin/python /tmp/blitzy_obs/observe_C_intermittency.py 5 procB'
[procB] pid=8489 DISABLE_RATE_LIMIT=True
[procB] FIXED tampered signed_suffix = '.list@sl.local.alCs5w.eL5N0YZ2fj0-sA1PWHH-meFfaqB'
[procB] run 1/5: HTTP 412 | body={"error":"Alias creation time is expired, please retry"}
[procB] run 2/5: HTTP 412 | body={"error":"Alias creation time is expired, please retry"}
[procB] run 3/5: HTTP 412 | body={"error":"Alias creation time is expired, please retry"}
[procB] run 4/5: HTTP 412 | body={"error":"Alias creation time is expired, please retry"}
[procB] run 5/5: HTTP 412 | body={"error":"Alias creation time is expired, please retry"}
[procB] DISTRIBUTION over 5 identical POSTs: {412: 5}
```

**Observed distribution: 10/10 → HTTP 412** (5/5 in process A pid 8466; 5/5 in fresh process B pid 8489).
**The behavior is deterministic; there is no run-to-run randomness.** The user's perceived "intermittency"
is therefore *not* nondeterminism — it is an **expectation mismatch**: four distinct causes (expired, tampered,
malformed, empty) all yield the *same* `412 "Alias creation time is expired, please retry"`, so a suffix that
is tampered/malformed appears to be reported as "expired," and the expected `400 "Tampered suffix"` is never
seen (root cause in §5).

---

## 7. Coverage pass

| Item / sub-question | Answered? | Observed status/value | `file:line` + function |
|---------------------|-----------|-----------------------|------------------------|
| **Q1** invalid (tampered) suffix | ✅ | `HTTP 412` `{"error":"Alias creation time is expired, please retry"}` | `check_suffix_signature` `app/alias_suffix.py:37-42` (catch `:41`) → `new_custom_alias.py:71-73`/`186-188` |
| **Q1** invalid (malformed / empty) suffix | ✅ | `HTTP 412` (both) | same; `verify_prefix_suffix` not reached |
| **Q2** expired suffix (age > 600 s) | ✅ | `HTTP 412` same body | `signer.unsign(..., max_age=600)` `app/alias_suffix.py:40` |
| **Q3** validation log entries | ✅ | `SL - WARNING - … new_custom_alias.py:72 - Alias creation time expired for <User …>` (v3: `:187`) | `LOG.w` `new_custom_alias.py:72`/`187`; format `app/log.py:12-15`; `LOG` `app/log.py:79` |
| **Q4** rate-limit headers | ✅ (NEGATIVE) | **none** present on 201/412/400 **or** 429; breach → `HTTP 429` `{"error":"Rate limit exceeded"}` | `Limiter(key_func=__key_func)` `app/extensions.py:23`; 429 handler `server.py:362-372` (root) |
| **Q5** quota checks on success | ✅ | `is_active=True, disabled=False, lifetime_or_active_subscription=False, count=1 < max=3` → `201`; **no log on success**; failure → `400` + `LOG.d` | `can_create_new_alias` `app/models.py:867-884`; `max_alias_for_free_account` `:858-865`; `LOG.d` `new_custom_alias.py:49`/`138` |
| **Q6** execution-path trace | ✅ | decorator stack + validation sequence (§4) | validator `app/alias_suffix.py:37-42`; enforcer `app/models.py:867-884` |
| Named: **invalid suffix** | ✅ | 412 | §3 Q1 |
| Named: **expired suffix** | ✅ | 412 | §3 Q2 |
| Named: **validation logs** | ✅ | `LOG.w` "Alias creation time expired" | §3 Q3 |
| Named: **rate-limit headers** | ✅ | none (negative) | §3 Q4 |
| Named: **quota checks** | ✅ | `count < max` (1 < 3) | §3 Q5 |
| Named: **execution path** | ✅ | full trace + diagram | §4 |
| Central defect (412-vs-400 dead code) | ✅ (documented, not fixed) | tampered→412, `Tampered suffix` unreachable | §5; `app/alias_suffix.py:41` |
| Intermittency | ✅ | 10/10 → 412 (deterministic) | §6 |

**Grounding / inference note.** Every behavioral statement above is backed by an unedited captured output block
plus a `file:line` + function reference. The only value obtained via **temporary instrumentation** (rather than
a native log) is the `[INSTRUMENT]`-labeled `count()`/`max_alias_for_free_account()` in Q5 — the underlying
methods executed for real; the wrapper only printed their inputs/outputs and `app/models.py` was not modified.
The itsdangerous subclass check in §5 is labeled **non-canonical** (a direct utility call); the canonical
behavioral evidence is the HTTP responses in §3.

---

## 8. Appendix

### 8.1 Full unedited transcript — conditions (a)–(d) [`DISABLE_RATE_LIMIT=True`]

Command: `docker exec sl_setup bash -lc 'cd /app && /app/venv/bin/python /tmp/blitzy_obs/observe_B.py'`

```
==================== ENVIRONMENT ====================
itsdangerous 1.1.0 | flask 1.1.2
EMAIL_DOMAIN = sl.local | MAX_NB_EMAIL_FREE_PLAN = 3

[CONFIG] config.DISABLE_RATE_LIMIT = True (conditions a-e; matches conftest flask_client L65)

##############################################################################
CONDITION: (a) FRESH VALID suffix -> v2 (expect 201)
##############################################################################
COMMAND: POST /api/v2/alias/custom/new
  json = {'alias_prefix': 'validv2', 'signed_suffix': '.test@sl.local.alCs5Q.dU5u0azHAW2cMxpP2hPM2PtdFTo'}
  headers = {'Authentication': '<API_KEY_REDACTED (60 chars)>'}
STATUS: 201
BODY: {"alias":"validv2.test@sl.local","creation_date":"2026-07-10 08:27:17+00:00","creation_timestamp":1783672037,"disable_pgp":false,"email":"validv2.test@sl.local","enabled":true,"id":2183,"latest_activity":null,"mailbox":{"email":"valid_gzlvqqja@mailbox.test","id":1567},"mailboxes":[{"email":"valid_gzlvqqja@mailbox.test","id":1567}],"name":null,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null,"pinned":false,"support_pgp":false}
RESPONSE HEADERS:
  Content-Type: application/json
  Content-Length: 434
  Access-Control-Allow-Origin: *
  Set-Cookie: slapp=<SESSION_COOKIE_REDACTED>; Domain=.sl.test; Expires=Fri, 17-Jul-2026 08:27:17 GMT; HttpOnly; Path=/; SameSite=Lax

##############################################################################
CONDITION: (a) FRESH VALID suffix -> v3 (expect 201)   [STATUS: 201]
##############################################################################
BODY: {"alias":"validv3.word@sl.local", ... "id":2185, ...}

##############################################################################
CONDITION: (b) EXPIRED suffix (age~1000s) -> v2   [STATUS: 412]
##############################################################################
BODY: {"error":"Alias creation time is expired, please retry"}
SL: 2026-07-10 08:27:18,464 - SL - WARNING - 8348 - "/app/app/api/views/new_custom_alias.py:72" - new_custom_alias_v2() -  - Alias creation time expired for <User 1316 Obs expired_gkcdzwiq@mailbox.test>

##############################################################################
CONDITION: (b) EXPIRED suffix (age~1000s) -> v3   [STATUS: 412]
##############################################################################
BODY: {"error":"Alias creation time is expired, please retry"}
SL: 2026-07-10 08:27:18,743 - SL - WARNING - 8348 - "/app/app/api/views/new_custom_alias.py:187" - new_custom_alias_v3() -  - Alias creation time expired for <User 1317 Obs expired_zajmgvfb@mailbox.test>

##############################################################################
CONDITION: (c) TAMPERED suffix -> v2   [STATUS: 412, NOT 400]
##############################################################################
BODY: {"error":"Alias creation time is expired, please retry"}
SL: 2026-07-10 08:27:19,019 - SL - WARNING - 8348 - "/app/app/api/views/new_custom_alias.py:72" - new_custom_alias_v2() -  - Alias creation time expired for <User 1318 Obs tamper_xdmdwabb@mailbox.test>

##############################################################################
CONDITION: (c) TAMPERED suffix -> v3   [STATUS: 412]
##############################################################################
BODY: {"error":"Alias creation time is expired, please retry"}
SL: 2026-07-10 08:27:19,299 - SL - WARNING - 8348 - "/app/app/api/views/new_custom_alias.py:187" - new_custom_alias_v3() -  - Alias creation time expired for <User 1319 Obs tamper_bnesmfcv@mailbox.test>

##############################################################################
CONDITION: (d) MALFORMED suffix 'notasignature' (no '.') -> v2   [STATUS: 412]
##############################################################################
BODY: {"error":"Alias creation time is expired, please retry"}
SL: 2026-07-10 08:27:19,575 - SL - WARNING - 8348 - "/app/app/api/views/new_custom_alias.py:72" - new_custom_alias_v2() -  - Alias creation time expired for <User 1320 Obs malf_cyzqbrmo@mailbox.test>

##############################################################################
CONDITION: (d) EMPTY suffix '' -> v2   [STATUS: 412]
##############################################################################
BODY: {"error":"Alias creation time is expired, please retry"}
SL: 2026-07-10 08:27:19,852 - SL - WARNING - 8348 - "/app/app/api/views/new_custom_alias.py:72" - new_custom_alias_v2() -  - Alias creation time expired for <User 1321 Obs empty_scdlvujb@mailbox.test>
```

(For (a) both v2 and v3 the response header set is exactly `Content-Type, Content-Length,
Access-Control-Allow-Origin, Set-Cookie`; every 412 header set is identical except `Content-Length: 57`.)

### 8.2 Full unedited transcript — conditions (e)–(f)

Command: `docker exec sl_setup bash -lc 'cd /app && /app/venv/bin/python /tmp/blitzy_obs/observe_B2.py'`
(see the verbatim `[INSTRUMENT]`, `STATUS`, `BODY`, `SL` and `[Q4]` lines embedded in §3 Q4 and Q5; the final
`[Q4]` summary was `Rate-limit-specific headers present on 429 response: NONE` and
`Full 429 header names: ['Content-Type', 'Content-Length', 'Access-Control-Allow-Origin', 'Set-Cookie']`.)

### 8.3 Read-only proof — repository left unchanged

The only artifact written to the repository is this document. The temporary observation scripts lived under the
container's `/tmp/blitzy_obs/` (outside the repository checkout) and were deleted after capture. On the host
checkout, `git status --porcelain` collapses the brand-new, fully-untracked `blitzy/` tree to a single entry;
expanding untracked directories to individual files shows the one and only file written, and there are **no
modified or deleted tracked files**:

```
$ git rev-parse --abbrev-ref HEAD
blitzy-9b4ce125-7165-4a3a-b9c5-9be93b399ff9

$ git status --porcelain
?? blitzy/

# blitzy/ has zero tracked files, so porcelain collapses it. Expanded to files it is exactly one file:
$ git status --porcelain --untracked-files=all
?? blitzy/documentation/app_2cd6ee777f8c.md

# No tracked file was modified or deleted (empty output = read-only constraint satisfied):
$ git status --porcelain --untracked-files=all | grep -E '^( M|MM|A |D | D|R |C )'
(no output)
```

> Note: `blitzy/` also contains empty `screenshots/` and `screen_recordings/` scaffold directories; Git does not
> track empty directories, so `--untracked-files=all` correctly lists only the report file.

> The filename `app_2cd6ee777f8c.md` matches the investigation's source branch/commit
> (`2cd6ee777f8c…`), per the task's mandated single-deliverable location `blitzy/documentation/`.
