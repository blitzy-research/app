# SimpleLogin — Custom‑Alias Creation: Signed‑Suffix Validation & Limit Enforcement

**An investigative, runtime‑grounded answer document.** Every behavioral claim below was produced by *running the real code* inside the prescribed container and capturing the actual, unedited output. Commands and `file:line` references accompany each observation. Where a statement could not be observed at runtime it is explicitly labelled **`inferred`**.

- **Codebase:** SimpleLogin (Poetry‑managed Flask email‑alias service)
- **Runtime:** container image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (alias `andrewparkscaleai/coding-agent:simple-login__app__2cd6ee777f8c…`), repo baked at `/app`, commit `2cd6ee777f8c`, Python **3.10.18**
- **Pinned libraries observed:** `itsdangerous 1.1.0`, `flask 1.1.2`, `flask-limiter 1.4`, `werkzeug 1.0.1`, `redis 4.6.0`, `gunicorn 20.0.4`
- **Endpoints under investigation:** `POST /api/v2/alias/custom/new` and `POST /api/v3/alias/custom/new` [`app/api/views/new_custom_alias.py:28-235`]

---

## 1. TL;DR — Direct Answer to All Five Questions

> **Lead finding (root cause of the "intermittent validation failures that don't match expected behavior"):**
> An **invalid (tampered)** signed suffix and a genuinely **expired** signed suffix return the **exact same** response — **HTTP `412 PRECONDITION FAILED`** with body `{"error":"Alias creation time is expired, please retry"}` and the **same** console log line `LOG.w("Alias creation time expired for %s", user)`. The endpoint's *other* branch — `except Exception → HTTP 400 {"error":"Tampered suffix"}` — is **effectively unreachable** for ordinary string input. The cause is that `check_suffix_signature` catches `itsdangerous.BadSignature` [`app/alias_suffix.py:37-42`], and with the pinned **itsdangerous 1.1.0** both `SignatureExpired` **and** `BadTimeSignature` are subclasses of `BadSignature`, so a tampered, expired, or garbage suffix **all** get swallowed identically and return `None`. This was demonstrated end‑to‑end over real HTTP (see §3 and §9).

| # | Question | Direct answer (observed) |
|---|----------|--------------------------|
| **Q1** | Invalid / expired suffix → exact status codes & error messages? | **Both → `HTTP 412`**, body `{"error":"Alias creation time is expired, please retry"}`. (A garbage string also → 412.) Never 400 "Tampered suffix" for ordinary input. |
| **Q2** | Console log entries for those failures? | A single **WARNING**: `… - SL - WARNING - … - "/app/app/api/views/new_custom_alias.py:72" - new_custom_alias_v2() - … - Alias creation time expired for <User …>` — emitted by `LOG.w(...)` [`new_custom_alias.py:72`]. Identical for tampered, expired, and garbage. |
| **Q3** | Rate‑limiting headers — present, and values? | **No `X-RateLimit-*` and no `Retry-After` headers appear on any response** (201, 400, 409, or 429). Stable across ≥2 runs. Only `Content-Type`, `Content-Length`, `Access-Control-Allow-Origin`, `Set-Cookie` are present. The `429` body is `{"error":"Rate limit exceeded"}`, produced by SimpleLogin's own error handler [`server.py:362-370`], not by Flask‑Limiter's default. |
| **Q4** | Success‑path quota checks & logged values? | The success path runs `user.can_create_new_alias()` [`app/models.py:867-884`] (which internally may call `max_alias_for_free_account()` [`app/models.py:858-865`]). **No quota value is logged on the 201 path** — `can_create_new_alias()` emits no log line itself; the only quota‑related log statement anywhere is the **failure‑path** `LOG.d("user %s cannot create any custom alias", user)` [`new_custom_alias.py:49`]. |
| **Q5** | Execution‑path trace & rejection conditions? | Decorator chain `@limiter.limit(ALIAS_LIMIT)` → `@require_api_auth` → `@parallel_limiter.lock(name="alias_creation")`, then in‑handler order: **quota → empty‑body → signature → prefix/suffix → duplicate → dots → create**. Signed suffixes are validated by `check_suffix_signature`/`verify_prefix_suffix` [`app/alias_suffix.py:37-91`]; creation limits are enforced by three independent components — `can_create_new_alias()` (account quota), `@limiter.limit(ALIAS_LIMIT)` (HTTP rate limit → 429), and `@parallel_limiter.lock` (Redis concurrency lock → 429). See §5 diagram. |

---

## 2. Environment & How to Reproduce

All observations were captured inside the prescribed container (`docker exec sl_setup …`), against a **real** PostgreSQL 15 (`localhost:5432`, `test/test/test`, with the `pg_trgm` extension) and a **real** Redis (`localhost:6379`). Dependencies live in the venv at `/app/venv`, so every probe was run with `/app/venv/bin/python`.

**Canonical library/interpreter versions actually observed:**

```
$ /app/venv/bin/python -c "import itsdangerous, flask, flask_limiter, werkzeug; \
  print('python 3.10.18'); print('itsdangerous', itsdangerous.__version__); \
  print('flask', flask.__version__); print('flask_limiter', flask_limiter.__version__); \
  print('werkzeug', werkzeug.__version__)"
itsdangerous 1.1.0
flask 1.1.2
flask_limiter 1.4
werkzeug 1.0.1
```

Two runners were used, both driving the **real** Flask app through its **real** entry point:

1. **pytest `flask_client` fixture** [`tests/conftest.py:59-77`] — the fastest canonical harness. It drives the real endpoint against real Postgres and rolls the transaction back. It sets `config.DISABLE_RATE_LIMIT = True` [`tests/conftest.py:65,73`], so it is used for status/body/log observations of every condition **except** the rate‑limit headers. Canonical invocation (note the runtime DB port is **5432**, overriding the `15432` written in `tests/test.env:17`):

   ```
   cd /app && env -i PATH=/usr/local/bin:/usr/bin:/bin HOME=/root \
     DB_URI='postgresql://test:test@localhost:5432/test' CONFIG=tests/test.env \
     GNUPGHOME=/tmp/sl_gnupg PYTHONPATH=/app \
     /app/venv/bin/python -m pytest <probe>.py -o addopts='' -p no:cacheprovider -s -q
   ```

2. **Live Gunicorn server** [`Dockerfile:47`] — used for genuine over‑the‑wire HTTP evidence and to mint suffixes through the canonical producer `GET /api/v4/alias/options`:

   ```
   cd /app && set -a && . /tmp/sl.env && set +a && \
     /app/venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 30
   ```

**Canonical minting of `signed_suffix` (never hand‑forged):** valid suffixes were produced either by the shared signer `app.alias_suffix.signer.sign(...).decode()` (keyed by `CUSTOM_ALIAS_SECRET = FLASK_SECRET + "custom_alias"` = `"secretcustom_alias"` in the test env [`app/config.py:201`, `tests/test.env:20`]) or by the real API producer `GET /api/v4/alias/options` (`get_alias_suffixes` [`app/api/views/alias_options.py`]). The **expired** case was produced by temporarily backdating the *real* signer's timestamp inside the probe only (never editing product code), so `signer.unsign(signed_suffix, max_age=600)` raises `SignatureExpired` for real.

> **Context (not a result):** the planning/analysis sandbox was Python 3.12 with no Poetry/Postgres/Redis and could not import the 2020‑era pinned Flask stack; **all** results in this document were therefore captured inside the prescribed container as described above.

**Scope note (read‑only):** the SimpleLogin source repository was treated as strictly read‑only. All probe scripts were temporary and were removed after evidence capture; the only file added is this document (`blitzy/documentation/app_2cd6ee777f8c.md`).

---

## 3. Q1 — Invalid / Expired Signed Suffixes → Exact Status Codes & Error Messages

**Answer:** A **tampered (invalid)** suffix and a genuinely **expired** suffix both return **`HTTP 412 PRECONDITION FAILED`** with the identical JSON body `{"error":"Alias creation time is expired, please retry"}`. A **garbage** string behaves identically. The 400 `"Tampered suffix"` branch does not fire for ordinary string input (see §9 for the root cause).

The exact source of these two outcomes is the signed‑suffix `try/except` block in the handler [`app/api/views/new_custom_alias.py:69-76`]:

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

### 3.1 Over‑the‑wire HTTP evidence (live Gunicorn, canonical producer)

The `signed_suffix` was minted by the canonical producer `GET /api/v4/alias/options` (`get_alias_suffixes`), then driven through the real endpoint with `curl -i`. Complete, unedited output:

**Mint (canonical):**
```
$ curl -s -H 'Authentication: <api-key>' http://localhost:7777/api/v4/alias/options
{"can_create":true,"prefix_suggestion":"","suffixes":[[".word364@d1.test",".word364@d1.test.alUhNQ.xg6WSUtZ9tdqLxKRv0HK3LH4Tww"], … ,[".test398@sl.local",".test398@sl.local.alUhUg._9UQ-s3ai3Q8D3f6MR-50Js3bAU"]]}
# canonical VALID signed_suffix = ".test398@sl.local.alUhUg._9UQ-s3ai3Q8D3f6MR-50Js3bAU"
```

**VALID → 201:**
```
$ curl -s -i -X POST -H 'Authentication: <api-key>' -H 'Content-Type: application/json' \
    -d '{"alias_prefix":"livevalid","signed_suffix":"<VALID>"}' \
    http://localhost:7777/api/v2/alias/custom/new
HTTP/1.1 201 CREATED
Server: gunicorn/20.0.4
Content-Type: application/json
Content-Length: 446
Access-Control-Allow-Origin: *
Set-Cookie: slapp=…; Expires=Mon, 20-Jul-2026 17:33:07 GMT; HttpOnly; Path=/; SameSite=Lax

{"alias":"livevalid.test398@sl.local","creation_date":"2026-07-13 17:33:07+00:00","creation_timestamp":1783963987,"disable_pgp":false,"email":"livevalid.test398@sl.local","enabled":true,"id":2820,"latest_activity":null,"mailbox":{"email":"livedemo_wfsbqt@mailbox.test","id":2010},"mailboxes":[{"email":"livedemo_wfsbqt@mailbox.test","id":2010}],"name":null,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null,"pinned":false,"support_pgp":false}
```

**TAMPERED (last character of the valid token flipped) → 412:**
```
$ curl -s -i -X POST -H 'Authentication: <api-key>' -H 'Content-Type: application/json' \
    -d '{"alias_prefix":"livetamper","signed_suffix":"<TAMPERED>"}' \
    http://localhost:7777/api/v2/alias/custom/new
HTTP/1.1 412 PRECONDITION FAILED
Server: gunicorn/20.0.4
Content-Type: application/json
Content-Length: 57
Access-Control-Allow-Origin: *
Set-Cookie: slapp=…; HttpOnly; Path=/; SameSite=Lax

{"error":"Alias creation time is expired, please retry"}
```

**EXPIRED (real signer, timestamp backdated 601 s so `unsign(max_age=600)` raises `SignatureExpired`) → 412 (identical):**
```
$ curl -s -i -X POST -H 'Authentication: <api-key>' -H 'Content-Type: application/json' \
    -d '{"alias_prefix":"liveexp","signed_suffix":"<EXPIRED>"}' \
    http://localhost:7777/api/v2/alias/custom/new
HTTP/1.1 412 PRECONDITION FAILED
Server: gunicorn/20.0.4
Content-Type: application/json
Content-Length: 57
Access-Control-Allow-Origin: *
Set-Cookie: slapp=…; HttpOnly; Path=/; SameSite=Lax

{"error":"Alias creation time is expired, please retry"}
```

**GARBAGE (`"not-a-real-token"`) → 412 (identical):**
```
HTTP/1.1 412 PRECONDITION FAILED
Content-Length: 57
…
{"error":"Alias creation time is expired, please retry"}
```

The three failure cases are byte‑for‑byte identical (`Content-Length: 57` in every case) — the direct demonstration that **invalid and expired are indistinguishable at the API**.

### 3.2 Status‑code / error‑message table (invalid, expired, successful)

| Input to `signed_suffix` | HTTP status | JSON body | Emitting branch |
|--------------------------|-------------|-----------|-----------------|
| Valid (minted via signer / options API) | `201 CREATED` | full serialized alias (see §6) | `new_custom_alias.py:109-112` |
| **Tampered** (valid token, last char flipped) | `412 PRECONDITION FAILED` | `{"error":"Alias creation time is expired, please retry"}` | `new_custom_alias.py:71-73` |
| **Expired** (`> max_age=600 s`) | `412 PRECONDITION FAILED` | `{"error":"Alias creation time is expired, please retry"}` | `new_custom_alias.py:71-73` |
| Garbage string | `412 PRECONDITION FAILED` | `{"error":"Alias creation time is expired, please retry"}` | `new_custom_alias.py:71-73` |
| (theoretical) non‑`BadSignature` exception | `400 BAD REQUEST` | `{"error":"Tampered suffix"}` | `new_custom_alias.py:74-76` — **effectively unreachable**, see §9 |

The same results were reproduced through the pytest `flask_client` harness (see §8 for the full per‑condition transcript, including v2 and v3).

---

## 4. Q2 — Server‑Console Validation LOG Entries for Those Failures

**Answer:** For a tampered, expired, or garbage suffix the endpoint emits a **single `WARNING`** line from `LOG.w("Alias creation time expired for %s", user)` [`app/api/views/new_custom_alias.py:72`]. There is **no** additional validation log; in particular the `"Alias suffix is tampered, user %s"` line [`new_custom_alias.py:75`] is never produced for ordinary input (its branch is unreachable — §9).

The `"SL"` logger writes to `sys.stdout` via `StreamHandler(sys.stdout)` [`app/log.py:41`], at level DEBUG [`app/log.py:51`], `propagate=False` [`app/log.py:59`], using the format [`app/log.py:12-15`]:

```
"%(asctime)s - %(name)s - %(levelname)s - %(process)d - \"%(pathname)s:%(lineno)d\" - %(funcName)s() - %(message_id)s - %(message)s"
```

`LOG.w` is the `logging.Logger.warning` shortcut [`app/log.py:76`]; `message_id` defaults to the empty string [`app/log.py:19,33`].

**Real server stdout captured from the live Gunicorn run** (three consecutive failures — tampered, expired, garbage — each producing the *same* WARNING line, and each followed by the request‑completion DEBUG line):

```
2026-07-13 17:33:07,060 - SL - WARNING - 3762 - "/app/app/api/views/new_custom_alias.py:72" - new_custom_alias_v2() -  - Alias creation time expired for <User 1700 Live Demo livedemo_wfsbqt@mailbox.test>
2026-07-13 17:33:07,061 - SL - DEBUG - 3762 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 412, takes 0.004633426666259766
2026-07-13 17:33:07,083 - SL - WARNING - 3763 - "/app/app/api/views/new_custom_alias.py:72" - new_custom_alias_v2() -  - Alias creation time expired for <User 1700 Live Demo livedemo_wfsbqt@mailbox.test>
2026-07-13 17:33:07,084 - SL - DEBUG - 3763 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 412, takes 0.01418447494506836
2026-07-13 17:33:07,098 - SL - WARNING - 3762 - "/app/app/api/views/new_custom_alias.py:72" - new_custom_alias_v2() -  - Alias creation time expired for <User 1700 Live Demo livedemo_wfsbqt@mailbox.test>
2026-07-13 17:33:07,098 - SL - DEBUG - 3762 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 412, takes 0.005208730697631836
```

The `pathname:lineno` field (`"/app/app/api/views/new_custom_alias.py:72"`) and `funcName` (`new_custom_alias_v2()`) tie each captured line to the exact emitting statement. The same WARNING line was also captured through the pytest harness for the v2 handler, e.g.:

```
… - SL - WARNING - 3544 - "/app/app/api/views/new_custom_alias.py:72" - new_custom_alias_v2() -  - Alias creation time expired for <User 1683 Test User user_xt5aawaikt@mailbox.test>
```

**Levels reference** [`app/log.py:74-77`]: `LOG.d`=DEBUG, `LOG.i`=INFO, `LOG.w`=WARNING, `LOG.e`=ERROR‑with‑traceback (`logging.Logger.exception`). The failure line here is a **WARNING** (`LOG.w`).


---

## 5. Q3 — Rate‑Limiting Headers: Do Any Appear, and What Are Their Values?

**Answer:** **No rate‑limiting headers appear at all.** With rate limiting explicitly **enabled**, neither `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, nor `Retry-After` is present on **any** response — not on a `201`, and not on the `429`. This was **stable across ≥2 runs**. The only response headers are `Content-Type`, `Content-Length`, `Access-Control-Allow-Origin`, and `Set-Cookie`.

### 5.1 Why the observation required enabling rate limiting

The pytest fixture disables rate limiting (`config.DISABLE_RATE_LIMIT = True` [`tests/conftest.py:65,73`]), which short‑circuits the limiter via the request filter `disable_rate_limit()` [`app/extensions.py:26-28`]. To observe the limiter's real behavior, the probe set `config.DISABLE_RATE_LIMIT = False`, used a lifetime user (to bypass the account quota so requests reach the limiter), and exceeded `ALIAS_LIMIT = "100/day;50/hour;5/minute"` [`app/config.py:448`] on the SL domain, applying the documented flask‑limiter test‑client workaround `g._rate_limiting_complete = False` between requests (GH issue #147). The limiter key is `userid:{current_user.id}` [`app/extensions.py:14-19`].

### 5.2 Observed output — RUN 1 (per‑request status + header presence)

```
################ RUN 1 (ALIAS_LIMIT='100/day;50/hour;5/minute', lifetime user) ################
  req 0 -> status=201  rate-limit-headers-present=NONE
  req 1 -> status=201  rate-limit-headers-present=NONE
  req 2 -> status=201  rate-limit-headers-present=NONE
  req 3 -> status=201  rate-limit-headers-present=NONE
  req 4 -> status=201  rate-limit-headers-present=NONE
  req 5 -> status=201  rate-limit-headers-present=NONE
  req 6 -> status=429  rate-limit-headers-present=NONE

  --- FULL headers on first 201 (RUN 1) ---
      Content-Type: application/json
      Content-Length: 432
      Access-Control-Allow-Origin: *
      Set-Cookie: slapp=102b23c0-…; Domain=.sl.test; Expires=Mon, 20-Jul-2026 17:29:53 GMT; HttpOnly; Path=/; SameSite=Lax
  >>> RUN 1 201: X-RateLimit-*/Retry-After present? -> NONE

  --- 429 body: {"error": "Rate limit exceeded"}
  --- FULL headers on 429 (RUN 1) ---
      Content-Type: application/json
      Content-Length: 32
      Access-Control-Allow-Origin: *
      Set-Cookie: slapp=102b23c0-…; Domain=.sl.test; Expires=Mon, 20-Jul-2026 17:29:54 GMT; HttpOnly; Path=/; SameSite=Lax
  >>> RUN 1 429: X-RateLimit-*/Retry-After present? -> NONE (no rate-limit headers emitted)
```

### 5.3 Observed output — RUN 2 (stability confirmation)

```
################ RUN 2 (ALIAS_LIMIT='100/day;50/hour;5/minute', lifetime user) ################
  req 0..5 -> status=409  rate-limit-headers-present=NONE
  req 6    -> status=429  rate-limit-headers-present=NONE

  --- 429 body: {"error": "Rate limit exceeded"}
  --- FULL headers on 429 (RUN 2) ---
      Content-Type: application/json
      Content-Length: 32
      Access-Control-Allow-Origin: *
      Set-Cookie: slapp=635bc28e-…; Domain=.sl.test; Expires=Mon, 20-Jul-2026 17:29:54 GMT; HttpOnly; Path=/; SameSite=Lax
  >>> RUN 2 429: X-RateLimit-*/Retry-After present? -> NONE (no rate-limit headers emitted)
```

*(RUN 2's earlier requests returned `409` because RUN 1's committed aliases persisted in the container's test database; this is irrelevant to the header question — the limiter counts every hit regardless of the handler outcome, the `429` still fired, and header presence was again `NONE`.)* A first run against a custom domain likewise showed `NONE` on the `429`, confirming the result does not depend on the prior request statuses.

### 5.4 Why there are no headers, and where the 429 body comes from

- **Header gating.** Flask‑Limiter only writes `X-RateLimit-*` headers when `RATELIMIT_HEADERS_ENABLED` is true, whose **default is `False`**. SimpleLogin initializes the limiter with `limiter.init_app(app)` [`server.py:167`] and passes **no** `headers_enabled` argument (and never sets `RATELIMIT_HEADERS_ENABLED` anywhere), so headers stay off. When they *are* enabled, the emitted set is `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, `Retry-After` — none of which we observed.
- **Version nuance.** The pinned **Flask‑Limiter 1.4** predates the 1.5 changelog entry *"Bug fix: Correct default setting for enabling rate limit headers"*, which is exactly why the presence/values had to be **observed at runtime** rather than assumed — and the observation is unambiguous: **no headers**.
- **The 429 body.** The body `{"error":"Rate limit exceeded"}` is **not** Flask‑Limiter's default text; it is produced by SimpleLogin's own error handler [`server.py:362-370`], which also logs a WARNING:

  ```
  2026-07-13 17:28:42,595 - SL - WARNING - 3635 - "/app/server.py:364" - rate_limited() -  - Client hit rate limit on path /api/v3/alias/custom/new, user:<User 1694 Test User user_gnlq7nutob@mailbox.test>
  ```
  ```python
  @app.errorhandler(429)
  def rate_limited(e):
      LOG.w("Client hit rate limit on path %s, user:%s", request.path, get_current_user())
      if request.path.startswith("/api/"):
          return jsonify(error="Rate limit exceeded"), 429
  ```
  This custom handler replaces Flask‑Limiter's default response object, which is a second reason no limiter headers survive onto the wire.

---

## 6. Q4 — Successful Creation: Which Quota Checks Run, and What Is Logged

**Answer:** The success path invokes exactly one quota gate — `user.can_create_new_alias()` [`app/models.py:867-884`] — which, for a free account, compares the alias count to `max_alias_for_free_account()` [`app/models.py:858-865`]. **No quota value is logged on the `201` success path.** `can_create_new_alias()` contains **no logging statement**; the only quota‑related log line in the whole path is the **failure‑path** `LOG.d("user %s cannot create any custom alias", user)` [`new_custom_alias.py:49`].

### 6.1 The quota logic (read‑only reference)

```python
# app/models.py:867-884
def can_create_new_alias(self) -> bool:
    if not self.is_active():                 # L872-873 -> False
        return False
    if self.disabled:                        # L875-876 -> False
        return False
    if self.lifetime_or_active_subscription():   # L878-879 -> True
        return True
    return (                                 # L881-884
        Alias.filter_by(user_id=self.id).count() < self.max_alias_for_free_account()
    )
```

`max_alias_for_free_account()` [`app/models.py:858-865`] returns `MAX_NB_EMAIL_OLD_FREE_PLAN` for old‑flagged accounts, otherwise `MAX_NB_EMAIL_FREE_PLAN` [`app/config.py:121-124`] (which the test env sets to **3** [`tests/test.env:13`]).

### 6.2 Observed success path — before/after quota state, and the absence of a quota log

Captured through the pytest harness. A freshly‑created free user already owns **one** default alias (a signup "newsletter" alias) and has an active trial:

```
FRESH USER id= 1680
alias count = 1
  alias: simplelogin-newsletter.word970@sl.local | creation via signup default
max_alias_for_free_account() = 3
can_create_new_alias() = True
trial_end = 2026-07-20T18:25:39+00:00 | lifetime = False
```

The `201` creation itself:

```
[Q4 quota BEFORE] count=1 can_create_new_alias()=True trial_end=2026-07-20T18:26:36+00:00

===== COND 1 SUCCESS v2 (expect 201) =====
HTTP status: 201
JSON body  : {"alias": "p7xj1ohp3.word@sl.local", "creation_date": "2026-07-13 17:26:37+00:00",
  "creation_timestamp": 1783963597, "disable_pgp": false, "email": "p7xj1ohp3.word@sl.local",
  "enabled": true, "id": 2788, "latest_activity": null,
  "mailbox": {"email": "user_dzpmhb9vo2@mailbox.test", "id": 1991},
  "mailboxes": [{"email": "user_dzpmhb9vo2@mailbox.test", "id": 1991}],
  "name": null, "nb_block": 0, "nb_forward": 0, "nb_reply": 0, "note": null,
  "pinned": false, "support_pgp": false}
SL validation log line(s):
  <none emitted by the validation branch>
[Q4 quota AFTER ] count=2 can_create_new_alias()=True
[Q4 success body keys] = ['alias', 'creation_date', 'creation_timestamp', 'disable_pgp', 'email',
  'enabled', 'id', 'latest_activity', 'mailbox', 'mailboxes', 'name', 'nb_block', 'nb_forward',
  'nb_reply', 'note', 'pinned', 'support_pgp']
```

The line `SL validation log line(s): <none emitted by the validation branch>` is the observed, direct confirmation that **the 201 path logs no quota value**. The `201` body is built by `serialize_alias_info_v2(get_alias_info_v2(alias))` [`app/api/serializer.py:55,252`], jsonified together with `alias=full_alias` [`new_custom_alias.py:109-112`]. The v3 handler produced an identically shaped body.

### 6.3 Contrast — the only quota log is on the failure path

When the quota **is** exhausted, the log line does appear (this is the sole quota‑related log statement in the path):

```
[Q4/quota] fresh user, trial_end=None, max_alias_for_free_account()=3
[Q4/quota] count BEFORE filling=1 can_create=True
[Q4/quota] count AFTER filling=3 can_create=False (created 2 fillers)

===== COND 7 QUOTA EXCEEDED (expect 400) =====
HTTP status: 400
JSON body  : {"error": "You have reached the limitation of a free account with the maximum of 3 aliases, please upgrade your plan to create more aliases"}
SL validation log line(s):
  2026-07-13 17:26:41,953 - SL - DEBUG - 3544 - "/app/app/api/views/new_custom_alias.py:49" - new_custom_alias_v2() -  - user <User 1689 Test User user_q2a6llxbbo@mailbox.test> cannot create any custom alias
```

Note the **before/during/after** quota state: count `1 → 3` after filling, at which point `can_create_new_alias()` flips to `False` and the `400` fires with the `LOG.d` at `new_custom_alias.py:49` (DEBUG level).


---

## 7. Q5 — Full Execution‑Path Trace & Rejection Conditions

**Components that validate the signed suffix:** `check_suffix_signature` [`app/alias_suffix.py:37-42`] (accepts/rejects the timestamped signature) and `verify_prefix_suffix` [`app/alias_suffix.py:45-91`] (validates the prefix/suffix format and that the domain belongs to the user).

**Components that enforce creation limits (three, independent):**
1. `user.can_create_new_alias()` [`app/models.py:867-884`] — **account‑level quota** → `HTTP 400`.
2. `@limiter.limit(ALIAS_LIMIT)` [`app/extensions.py:23`; `app/config.py:448`] — **Flask‑Limiter HTTP rate limit** (`userid:{id}` key) → `HTTP 429`.
3. `@parallel_limiter.lock(name="alias_creation")` [`app/parallel_limiter.py`] — **Redis concurrency lock** → `HTTP 429` (werkzeug `TooManyRequests`); a **no‑op** when Redis is absent [`app/parallel_limiter.py:51-52`].

**Decorator order (top→bottom)** [`new_custom_alias.py:28-31` / `115-118`]:
`@api_bp.route(..., methods=["POST"])` → `@limiter.limit(ALIAS_LIMIT)` → `@require_api_auth` → `@parallel_limiter.lock(name="alias_creation")`.

**In‑handler branch order** [`new_custom_alias.py:48-112`]: **quota** (L48) → hostname (L58) → **empty body** (L61) → **signature** (L69‑76) → **verify prefix/suffix** (L78) → **duplicate** (L82‑88) → **two dots** (L90‑94) → **create → 201** (L96‑112). The order matters: because the quota check precedes the signature check, a **quota‑exhausted user receives the `400` quota message regardless of suffix validity** (confirmed in §8, COND 7).

```mermaid
flowchart TD
    A["POST /api/v2 or v3 /alias/custom/new"] --> B{"@limiter.limit(ALIAS_LIMIT)"}
    B -- exceeded --> B1["HTTP 429 Rate limit exceeded"]
    B -- ok --> C{"@require_api_auth"}
    C -- "no / invalid key (no session)" --> C1["HTTP 401 Wrong api key"]
    C -- "disabled (api-key auth)" --> C2["HTTP 403 Disabled account"]
    C -- ok --> D{"@parallel_limiter.lock(alias_creation)"}
    D -- "contended (Redis present)" --> D1["HTTP 429 TooManyRequests"]
    D -- "acquired / no-op (no Redis)" --> E{"user.can_create_new_alias()"}
    E -- False --> E1["HTTP 400 quota msg + LOG.d 'cannot create any custom alias'"]
    E -- True --> F{"request body present?"}
    F -- empty --> F1["HTTP 400 request body cannot be empty"]
    F -- ok --> G{"check_suffix_signature(signed_suffix)"}
    G -- "None: BadSignature (tampered OR expired OR garbage)" --> G1["HTTP 412 'Alias creation time is expired' + LOG.w 'expired'"]
    G -- "non-BadSignature exception" --> G2["HTTP 400 'Tampered suffix' + LOG.w 'tampered' (effectively unreachable)"]
    G -- valid --> H{"verify_prefix_suffix()"}
    H -- False --> H1["HTTP 400 wrong alias prefix or suffix"]
    H -- True --> I{"duplicate or '..'?"}
    I -- duplicate --> I1["HTTP 409 alias already exists + LOG.d"]
    I -- "'..' present" --> I2["HTTP 400 2 consecutive dot signs"]
    I -- no --> J["Alias.create + commit -> HTTP 201 serialize_alias_info_v2"]
```

**Authentication gate detail** [`app/api/base.py:16-43`]: `authorize_request()` reads the `Authentication` header (L17), looks up `ApiKey.get_by(code=…)` (L18). With no key and no authenticated session → `401 "Wrong api key"` (L27); when a key **is** present, `g.user = api_key.user` (L34) and then `if g.user.disabled → 403 "Disabled account"` (L36‑37), `if not g.user.is_active() → 401 "Account does not exist"` (L39‑40). This runs **before** any business logic via the `require_api_auth` decorator (L52).

---

## 8. All Conditions Exercised — Commands, Status, Body, and Logs

Every distinct branch was exercised through the **real** entry point (pytest `flask_client` WSGI harness unless noted "live"). Each condition below shows the observed HTTP status, the complete JSON body, and the validation‑branch `"SL"` log line(s). The probe minted valid suffixes with `signer.sign(f".word@{EMAIL_DOMAIN}").decode()` (or the options API for the live cases) and authenticated the user with `tests/utils.login()` (session) or a real `ApiKey` (header).

| # | Condition | Status | JSON body (verbatim) | `"SL"` validation log line |
|---|-----------|--------|----------------------|----------------------------|
| 1 | **SUCCESS** (v2) | `201` | `{"alias":"p7xj1ohp3.word@sl.local", … full serialized alias …}` | *none* (no quota log on success) |
| 1b | **SUCCESS** (v3) | `201` | `{"alias":"ph7sauto2.word@sl.local", … }` | *none* |
| 1c | **SUCCESS** via API‑key header (live) | `201` | `{"alias":"livevalid.test398@sl.local", … }` | *none* |
| 2 | **Tampered** suffix | `412` | `{"error":"Alias creation time is expired, please retry"}` | `LOG.w new_custom_alias.py:72 "Alias creation time expired for <User …>"` |
| 3 | **Expired** suffix (`>600 s`) | `412` | `{"error":"Alias creation time is expired, please retry"}` | `LOG.w new_custom_alias.py:72 "Alias creation time expired for <User …>"` |
| 3b | **Garbage** suffix | `412` | `{"error":"Alias creation time is expired, please retry"}` | `LOG.w new_custom_alias.py:72 "Alias creation time expired for <User …>"` |
| 4 | **Empty body** (`{}`) | `400` | `{"error":"request body cannot be empty"}` | *none* |
| 5 | **Duplicate** alias | `409` | `{"error":"alias p2334jkqm.word@sl.local already exists"}` | `LOG.d new_custom_alias.py:87 "full alias already used p2334jkqm.word@sl.local"` |
| 6 | **Two consecutive dots** (`alias_prefix="prefix."`) | `400` | `{"error":"2 consecutive dot signs aren't allowed in an email address"}` | *none* |
| 7 | **Quota exceeded** | `400` | `{"error":"You have reached the limitation of a free account with the maximum of 3 aliases, please upgrade your plan to create more aliases"}` | `LOG.d new_custom_alias.py:49 "user <User …> cannot create any custom alias"` |
| 8 | **Wrong API key**, no session | `401` | `{"error":"Wrong api key"}` | *none* |
| 8b | **No auth** at all | `401` | `{"error":"Wrong api key"}` | *none* |
| 9 | **Disabled** account (API‑key auth) | `403` | `{"error":"Disabled account"}` | *none* |
| 9b | **Disabled** account (session auth) | `401` | `{"error":"Wrong api key"}` | *none* (see note) |
| 10 | **Rate limit exceeded** | `429` | `{"error":"Rate limit exceeded"}` | `LOG.w server.py:364 "Client hit rate limit on path … user:<User …>"` |
| 11 | **Concurrency‑lock contention** (Redis present) | `429` | `{"error":"Rate limit exceeded"}` | *(429 handled by `server.py:362-370`)* |

### 8.1 Selected verbatim transcripts

**Duplicate (409):**
```
[dup first create] status=201 alias=p2334jkqm.word@sl.local
===== COND 5 DUPLICATE alias (expect 409) =====
HTTP status: 409
JSON body  : {"error": "alias p2334jkqm.word@sl.local already exists"}
SL validation log line(s):
  2026-07-13 17:26:40,717 - SL - DEBUG - 3544 - "/app/app/api/views/new_custom_alias.py:87" - new_custom_alias_v2() -  - full alias already used p2334jkqm.word@sl.local
```

**Two consecutive dots (400):**
```
===== COND 6 TWO CONSECUTIVE DOTS (expect 400) =====
HTTP status: 400
JSON body  : {"error": "2 consecutive dot signs aren't allowed in an email address"}
SL validation log line(s):
  <none emitted by the validation branch>
```

**Auth gates (401 / 403):**
```
===== COND 8 WRONG/MISSING API KEY, no session (expect 401) =====
HTTP status: 401
JSON body  : {"error": "Wrong api key"}

===== COND 9 DISABLED via API-key auth (expect 403 'Disabled account') =====
HTTP status: 403
JSON body  : {"error": "Disabled account"}

===== COND 9b DISABLED via SESSION auth (observed 401 'Wrong api key') =====
HTTP status: 401
JSON body  : {"error": "Wrong api key"}
```

> **Note on COND 9 vs 9b (an exact, observed nuance):** the `403 "Disabled account"` gate [`app/api/base.py:36-37`] is reached only when authenticating **via an API key** (`g.user = api_key.user` is set directly). Under **session** authentication a disabled user is rejected earlier with `401 "Wrong api key"`, because Flask‑Login's `load_user()` returns `None` for a disabled user (`if user.disabled: return None` [`server.py:225-226`]), making `current_user` anonymous so `authorize_request()` falls into the `401` branch [`app/api/base.py:20-27`]. Both were observed.

**Concurrency lock (429 on contention, no‑op without Redis):**
```
[lock] parallel_limiter.lock_redis = <limits.storage.RedisStorage object at 0x7d1f502abdc0>
[lock] type = RedisStorage | active (not None)? -> True
[lock] pre-acquired cl:1698:alias_creation -> True
[lock] CONTENTION POST -> status=429 body={"error": "Rate limit exceeded"}
[lock] NO-OP (lock_redis=None) POST -> status=201 (held key ignored)
```
Here the probe pre‑acquired the exact key the decorator would take — `cl:{current_user.id}:alias_creation` [`app/parallel_limiter.py:56`] — so the endpoint's `acquire_lock()` (Redis `set(..., nx=True)`) failed and raised `werkzeug.exceptions.TooManyRequests()` [`app/parallel_limiter.py:30-34`] → `429`. Setting `lock_redis = None` exercised the no‑op path [`app/parallel_limiter.py:51-52`], and the same held key was ignored → `201`. This lock is **independent of `DISABLE_RATE_LIMIT`**.


---

## 9. Root‑Cause Deep‑Dive — Why Tampered *and* Expired Both Return 412

`check_suffix_signature` [`app/alias_suffix.py:37-42`] is the sole gate on the signature:

```python
signer = itsdangerous.TimestampSigner(config.CUSTOM_ALIAS_SECRET)   # L11

def check_suffix_signature(signed_suffix: str) -> Optional[str]:     # L37-42
    try:
        return signer.unsign(signed_suffix, max_age=600).decode()
    except itsdangerous.BadSignature:
        return None
```

With the pinned **itsdangerous 1.1.0**, `SignatureExpired` and `BadTimeSignature` are both **subclasses of `BadSignature`**, so the single `except itsdangerous.BadSignature` clause catches *all three* failure modes and returns `None`. Verified at runtime:

```
$ /app/venv/bin/python probe_a.py
itsdangerous.__version__ = 1.1.0
issubclass(SignatureExpired,  BadSignature) = True
issubclass(BadTimeSignature,  BadSignature) = True
issubclass(BadSignature,      BadData)      = True
SignatureExpired MRO = ['SignatureExpired', 'BadTimeSignature', 'BadSignature', 'BadData', 'Exception', 'BaseException', 'object']
BadTimeSignature MRO = ['BadTimeSignature', 'BadSignature', 'BadData', 'Exception', 'BaseException', 'object']
```

And the function's return values, plus the concrete exception each input raises when *not* caught:

```
CUSTOM_ALIAS_SECRET = 'secretcustom_alias'
VALID signed_suffix    = '@example.com.alUeiw.-vAg_O1vpeCwKSOAOY9svudWHwc'
  check_suffix_signature(VALID)    -> '@example.com'
TAMPERED signed_suffix = '@example.com.alUeiw.-vAg_O1vpeCwKSOAOY9svudWHwA'
  check_suffix_signature(TAMPERED) -> None
GARBAGE signed_suffix  = 'not-a-real-token'
  check_suffix_signature(GARBAGE)  -> None

--- raw signer.unsign (uncaught) ---
  VALID(max_age=600)     -> unsign OK: '@example.com'
  TAMPERED               -> raised itsdangerous.exc.BadTimeSignature: Signature b'-vAg_…WwA' does not match
  GARBAGE                -> raised itsdangerous.exc.BadSignature: No b'.' found in value
  raw unsign(max_age=1)  -> raised itsdangerous.exc.SignatureExpired: Signature age 2 > 1 seconds
```

**Consequences observed at the endpoint:**
- Every one of tampered / expired / garbage makes `check_suffix_signature` return `None`, so `if not alias_suffix:` [`new_custom_alias.py:71`] fires → `LOG.w(...)` [`:72`] → `412 "Alias creation time is expired, please retry"` [`:73`]. This is exactly what §3/§4 show for all three inputs.
- The `except Exception:` → `LOG.w("Alias suffix is tampered, user %s", user)` [`:75`] → `400 "Tampered suffix"` [`:76`] branch is **effectively unreachable for ordinary string input**: `signed_suffix` is coerced to a stripped `str` before validation (`data.get("signed_suffix", "").strip()` [`new_custom_alias.py:65`], and `… or ""` then `.strip()` in v3 [`:157-158`]), and calling `signer.unsign()` on a `str` raises a `BadSignature` subclass — which is caught by the **inner** `except` first. We were unable to trigger the `400 "Tampered suffix"` branch through the real entry point with any string input; the only way to reach it would be a non‑`BadSignature` exception from `unsign`, which ordinary request payloads do not produce.

**This is the direct explanation of the reported "intermittent validation failures that don't match the expected behavior":** a client that submits a stale (expired) suffix and a client that submits a corrupted (tampered) suffix get the *same* `412 "…expired…"` message, so the "invalid token" case is indistinguishable from the "expired token" case, and the `"Tampered suffix"` message a developer might grep for never appears.

---

## 10. Secondary Paths & Caveats

### 10.1 Related paths that share the same helpers (documented, not the primary target)

- **Web UI — `app/dashboard/views/custom_alias.py`.** Imports `check_suffix_signature`/`verify_prefix_suffix` [L9‑10], calls `can_create_new_alias()` [L36] and `check_suffix_signature` [L90]. It surfaces the **same** signature outcome via `flash("Alias creation time is expired, please retry", "warning")` [L93] (with `LOG.w` [L92]) instead of HTTP JSON, so it exhibits the **same 412‑style tampered≡expired behavior** but as a flashed warning. Its dots and duplicate messages differ in wording (`"Your alias can't contain 2 consecutive dots (..)"` [L104]; `"You already have this alias {full_alias}"` [L120]).
- **OAuth — `app/oauth/views/authorize.py`.** Imports `get_alias_suffixes, check_suffix_signature` [L7]; calls `can_create_new_alias()` [L169] which here **raises** `Exception(f"User {current_user} cannot create custom email")` [L170] rather than returning a quota JSON; `check_suffix_signature` [L184] (with `LOG.w` [L186] and `flash` [L187]); `verify_prefix_suffix` [L200].
- **Sibling — `app/api/views/new_random_alias.py`.** Shares the identical decorator chain (`@limiter.limit(ALIAS_LIMIT)` [L22], `@require_api_auth` [L23], `@parallel_limiter.lock(name="alias_creation")` [L24]) and calls `can_create_new_alias()` [L35], but creates a *random* (not custom) alias, so it does not touch `check_suffix_signature`.

These share the same underlying validators; the primary investigation above targets the API path (`/api/v2|v3/alias/custom/new`).

### 10.2 Caveats and `inferred` items

- **`400 "Tampered suffix"` unreachability is `inferred`** in the strict sense that we could not *produce* it through the real entry point despite varying the input (tampered, expired, garbage, empty‑string suffix). The reasoning — `str`‑coercion + `BadSignature`‑subclass capture — is grounded in `new_custom_alias.py:65,157-158` and `app/alias_suffix.py:37-42` and in the runtime MRO output above. Every *reachable* case was observed to return `412`.
- **Rate‑limit header default for Flask‑Limiter 1.4** was cross‑checked against documentation (`RATELIMIT_HEADERS_ENABLED` default `False`; the "correct default setting for enabling rate limit headers" fix lands in the 1.5 changelog, after 1.4). The concrete presence/values, however, were **observed at runtime** (§5) and are the authoritative result here: no headers.
- **Container clock** reads year 2026; all timestamps above are the real values emitted by the running server and are reported verbatim.
- **RUN 2 of the rate‑limit probe** returned `409`s on its early requests because RUN 1's committed aliases persisted in the container test DB; this does not affect the header conclusion (the `429` still fired with no headers).

---

## 11. Appendix — Temporary Probe Scripts (for transparency; not committed)

All probes were written under `/tmp/sl_probes` on the host and copied into the container (pytest probes into `/app/tests/` so the real `conftest.py` fixtures apply). **They were removed after evidence capture** — `git status` in both the deliverable repo and the container shows no probe files remain (see §12). The `"SL"` log lines were captured by attaching a handler to the `"SL"` logger reusing the app's own `_log_formatter` (the logger has `propagate=False` [`app/log.py:59`], so a directly‑attached handler is required).

**Probe A — MRO + `check_suffix_signature` (abridged):**
```python
import itsdangerous as i, time
from app.alias_suffix import check_suffix_signature, signer
print(i.__version__, issubclass(i.SignatureExpired, i.BadSignature), issubclass(i.BadTimeSignature, i.BadSignature))
valid = signer.sign("@example.com").decode()
print(check_suffix_signature(valid))                       # -> '@example.com'
print(check_suffix_signature(valid[:-1] + "A"))            # tampered -> None
print(check_suffix_signature("not-a-real-token"))          # garbage  -> None
```

**Endpoint probe — one isolated pytest function per condition (abridged):** each uses the real `flask_client` fixture, `tests/utils.login()` for session auth (or a real `ApiKey` for header auth), mints via `signer.sign(...)`, and dumps `r.status_code`, `r.json`, and the captured `"SL"` line. The **expired** token is minted by temporarily backdating the real signer's timestamp (probe‑only monkeypatch, product code untouched):
```python
def _mint_backdated(suffix, age_seconds):
    orig = signer.get_timestamp
    signer.get_timestamp = lambda: int(time.time()) - age_seconds
    try:
        return signer.sign(suffix).decode()
    finally:
        signer.get_timestamp = orig
```

**Rate‑limit probe (abridged):** `config.DISABLE_RATE_LIMIT = False`; lifetime user; loop 7 POSTs on the SL domain; `g._rate_limiting_complete = False` between requests; dump `r.headers` on the first `201` and on the `429`; repeated across two functions (RUN 1 / RUN 2).

**Concurrency‑lock probe (abridged):** pre‑`set(cl:{user.id}:alias_creation, …, nx=True)` in `parallel_limiter.lock_redis.storage`, POST → `429`; then `parallel_limiter.lock_redis = None`, POST → `201` (no‑op).

**Live demo (abridged):** start Gunicorn on `:7777`; `GET /api/v4/alias/options` to mint a canonical `signed_suffix`; `curl -i` POSTs for valid/tampered/expired/garbage/empty; read server stdout from the Gunicorn log.

---

## 12. Grounding & Scope Verification

- Every behavioral claim above sits next to its concrete observed output and/or a `file:line` reference and the command that produced it.
- All five questions are answered explicitly, addressing each named mechanism/function/condition/flag/status code by name.
- The lead finding (tampered **and** expired both → `412`) is demonstrated at runtime (§3, §9), including the itsdangerous 1.1.0 MRO output.
- The rate‑limit‑header observation was made with limiting **enabled** and confirmed **stable across two runs** (§5).
- The success path is stated precisely: **no quota value is logged on `201`**; the only quota log is the failure‑path `LOG.d` (§6).
- Non‑observable items are labelled **`inferred`** (§10.2).
- **Scope:** the SimpleLogin source repository was not modified; all temporary probe scripts were removed; the only file added is this document, `blitzy/documentation/app_2cd6ee777f8c.md`.

