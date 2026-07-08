# SimpleLogin Runtime Behavior Investigation — Empirical Q&A

This document answers **eight runtime‑behavior questions (Q1–Q8)** about SimpleLogin's
authentication and email‑forwarding subsystems. **Every answer was produced by actually
RUNNING the relevant code path and capturing its real, complete, unedited output** — not by
reading the source alone. Each section provides: (a) the direct answer, (b) the exact
command(s) run, (c) the complete unedited output, (d) the concrete observed value(s),
(e) `file:line` grounding, (f) cause→effect reasoning, and (g) explicit **OBSERVED** vs
**INFERRED** labeling.

## Runtime & Methodology

- **Runtime:** Python 3.10.18 inside the designated Docker container
  (`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`), SimpleLogin source at
  `/app`, commit `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`.
- **Pinned dependency versions (from `poetry.lock`; confirmed at runtime via `python -c "import …; print(__version__)"`):**
  `flask 1.1.2`, `flask-login 0.5.0`, `itsdangerous 1.1.0`, `werkzeug 1.0.1`, `redis 4.6.0`,
  `SQLAlchemy 1.3.24`, `psycopg2-binary 2.9.3`, `aiosmtpd 1.4.2`, `flanker 0.9.11`,
  `flask-limiter 1.4`, `flask-wtf 0.14.3`, `gunicorn 20.0.4`, `arrow 0.16.0`.
  **All observations below reflect THESE exact versions**, not newer releases.
- **Datastores:** PostgreSQL 15 (`postgresql://test:test@localhost:5432/test`, alembic head
  `32f25cbf12f6`) and Redis (`redis://localhost`).
- **Canonical web/API entry point:** `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15`
  (the `Dockerfile` CMD@47; `wsgi.py` = `from server import create_app` / `app = create_app()`).
  For the failed‑login rate‑limit edge (Q8) a second gunicorn was run on `:7778` with rate
  limiting **on**.
- **Email entry point (Q5):** the `email_handler.py` forward path (`aiosmtpd`).
- **Token entry point (Q6):** the real `app/alias_suffix.py` signer.
- **Seeded fixtures:** verified user `john@wick.com` (id=1, `activated=True`, password `password`),
  seeded by `flask dummy-data`; dedicated API keys created for Q2 (id=134) and Q7 (id=135).

**Environment preamble.** Unless otherwise noted, every command below is executed inside the
container as `docker exec sl bash -c '<cmd>'` from the host, after this canonical setup
(sourced from the image's `/build.sh`):

```bash
cd /app
. venv/bin/activate
eval "$(grep '^export ' /build.sh)"
# exports include: DB_URI=postgresql://test:test@localhost:5432/test
#                  FLASK_SECRET=secret   MEM_STORE_URI=redis://localhost
#                  PYTHONPATH=/app   FLASK_APP=server.py   DISABLE_RATE_LIMIT=1
```

`FLASK_SECRET=secret` is a **fake local development value**, not a real credential.
`MEM_STORE_URI=redis://localhost` is required so the custom `RedisSessionStore` is installed
(`server.py:165` `initialize_redis_services`).

**Startup evidence (canonical gunicorn):**

```
$ docker exec -d sl bash -c 'cd /app && . venv/bin/activate && eval "$(grep "^export " /build.sh)" \
    && exec gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15 > /tmp/gunicorn_7777.log 2>&1'
$ grep -E 'Starting gunicorn|Listening at|Using worker|Booting worker' /tmp/gunicorn_7777.log
[2026-07-08 05:10:27 +0000] [2587] [INFO] Starting gunicorn 20.0.4
[2026-07-08 05:10:28 +0000] [2587] [INFO] Listening at: http://0.0.0.0:7777 (2587)
[2026-07-08 05:10:28 +0000] [2587] [INFO] Using worker: sync
[2026-07-08 05:10:28 +0000] [2594] [INFO] Booting worker with pid: 2594
[2026-07-08 05:10:28 +0000] [2595] [INFO] Booting worker with pid: 2595
$ curl -s -o /dev/null -w "HTTP %{http_code} -> redirect: %{redirect_url}\n" http://localhost:7777/
HTTP 302 -> redirect: http://localhost:7777/auth/login
```

---

## Q1 — Session‑authenticated API access WITHOUT an API key

**(a) Direct answer.** With an authenticated browser session (the signed `slapp` cookie) but
**no `Authentication` header**, `GET /api/user_info` returns **HTTP 200 OK** and the full user
JSON — the request "seems to work" because of a **session fallback**. Without either a cookie
or a key, the same endpoint returns **HTTP 401** with body `{"error":"Wrong api key"}`.

**(b) Exact commands.** Log `john@wick.com` in through the web UI (CSRF‑protected), then call
the API with only the cookie; also call it with nothing.

```bash
BASE=http://localhost:7777 ; JAR=/tmp/q1_cookies.txt ; rm -f "$JAR"
# 1) GET login page -> establish CSRF session + extract csrf_token
CSRF=$(curl -s -c "$JAR" "$BASE/auth/login" | grep -oP 'name="csrf_token"[^>]*value="\K[^"]+' | head -1)
# 2) POST credentials -> authenticated slapp cookie stored in $JAR
curl -s -i -b "$JAR" -c "$JAR" \
  --data-urlencode "email=john@wick.com" --data-urlencode "password=password" \
  --data-urlencode "csrf_token=$CSRF" "$BASE/auth/login" | head -20
# 3) Q1 PRIMARY: cookie present, NO Authentication header
curl -s -i -b "$JAR" "$BASE/api/user_info"
# 4) Q1 ERROR PATH: no cookie, no Authentication header
curl -s -i "$BASE/api/user_info"
```

**(c) Complete unedited output.**

```
===== STEP 2: POST /auth/login with john@wick.com / password (+csrf) =====
HTTP/1.1 302 FOUND
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 04:21:37 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 229
Location: http://localhost:7777/dashboard/
Set-Cookie: slapp=e2274242-78a0-4e87-8621-5c88f68c10f2.eQFzxwhw0XcLLAILGX62g9-S3rs; Expires=Wed, 15-Jul-2026 04:21:37 GMT; HttpOnly; Path=/; SameSite=Lax

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<title>Redirecting...</title>
<h1>Redirecting...</h1>
<p>You should be redirected automatically to target URL: <a href="/dashboard/">/dashboard/</a>.  If not click the link.

===== STEP 3 (Q1 PRIMARY): GET /api/user_info WITH cookie, NO Authentication header =====
$ curl -s -i -b $JAR http://localhost:7777/api/user_info
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 04:21:37 GMT
Connection: close
Content-Type: application/json
Content-Length: 239
Access-Control-Allow-Origin: *
Set-Cookie: slapp=e2274242-78a0-4e87-8621-5c88f68c10f2.eQFzxwhw0XcLLAILGX62g9-S3rs; Expires=Wed, 15-Jul-2026 04:21:37 GMT; HttpOnly; Path=/; SameSite=Lax

{"can_create_reverse_alias":true,"connected_proton_address":null,"email":"john@wick.com","in_trial":false,"is_premium":true,"max_alias_free_plan":3,"name":"John Wick","profile_picture_url":"http://localhost/static/upload/profile_pic.svg"}

===== STEP 4 (Q1 ERROR PATH): GET /api/user_info with NO cookie, NO Authentication header =====
$ curl -s -i http://localhost:7777/api/user_info
HTTP/1.1 401 UNAUTHORIZED
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 04:21:37 GMT
Connection: close
Content-Type: application/json
Content-Length: 26
Access-Control-Allow-Origin: *
Set-Cookie: slapp=85f92c0a-26a9-4020-82d9-425376d695d6.TfpiOWkVmS58r23rrRwFFoTza1g; Expires=Wed, 15-Jul-2026 04:19:28 GMT; HttpOnly; Path=/; SameSite=Lax

{"error":"Wrong api key"}
```

**(d) Concrete observed values.**
- **Primary (session fallback):** status line `HTTP/1.1 200 OK`; body
  `{"can_create_reverse_alias":true,"connected_proton_address":null,"email":"john@wick.com","in_trial":false,"is_premium":true,"max_alias_free_plan":3,"name":"John Wick","profile_picture_url":"http://localhost/static/upload/profile_pic.svg"}`.
  The eight body keys are `can_create_reverse_alias`, `connected_proton_address`, `email`,
  `in_trial`, `is_premium`, `max_alias_free_plan`, `name`, `profile_picture_url`.
- **Error path:** status line `HTTP/1.1 401 UNAUTHORIZED`; body `{"error":"Wrong api key"}`.

**(e) `file:line` grounding.** `authorize_request()` reads the `Authentication` header
(`app/api/base.py:17`) and looks up the key (`:18`); when no key is found it checks
`current_user.is_authenticated` and, if true, sets `g.user = current_user`
(`app/api/base.py:20-25`) — the session fallback. The `HEADER_ALLOW_API_COOKIES` gate that
would otherwise require the `X-Sl-Allowcookies` header is **commented out**
(`app/api/base.py:22-24`), so the fallback depends **only** on `current_user.is_authenticated`.
When neither key nor session exists it returns `jsonify(error="Wrong api key"), 401`
(`app/api/base.py:27`). The endpoint is `GET /api/user_info`
(`app/api/views/user_info.py:50-52`, `@require_api_auth`); the body is built by `user_to_dict`
(`app/api/views/user_info.py:28-47`).

**(f) Cause→effect.** The `slapp` cookie authenticates the Flask‑Login session, so
`current_user.is_authenticated` is `True`; `authorize_request()` therefore takes the fallback
branch, sets `g.user`, returns `None` (no error), and `user_info()` renders the JSON → **200**.
Remove the cookie and `current_user.is_authenticated` is `False`, so the `else` branch returns
**401 `{"error":"Wrong api key"}`**.

**(g) OBSERVED vs INFERRED.** Both the 200 body and the 401 body are **OBSERVED** at runtime
via `curl -i` against the canonical gunicorn server.

---

## Q2 — Privileged ("sudo") operation with only a browser session

**(a) Direct answer.** The single `@require_api_sudo` endpoint is `DELETE /api/user`. Two
distinct outcomes were **OBSERVED**: with **only a browser session** (cookie, no
`Authentication` header) it returns **HTTP 500** `{"error":"Internal error"}` (an
`AttributeError` because the sudo check dereferences a `None` API key); with a **real API key
that has no active sudo** it returns the non‑standard **HTTP 440** `{"error":"Need sudo"}`.

**(b) Exact commands.**

```bash
# (a) browser session ONLY: first establish john's authenticated cookie jar (web login),
#     then DELETE /api/user with ONLY that cookie (NO Authentication header).
BASE=http://localhost:7777 ; JAR=/tmp/q1_cookies.txt ; rm -f "$JAR"
CSRF=$(curl -s -c "$JAR" "$BASE/auth/login" | grep -oP 'name="csrf_token"[^>]*value="\K[^"]+' | head -1)
curl -s -o /dev/null -b "$JAR" -c "$JAR" \
  --data-urlencode "email=john@wick.com" --data-urlencode "password=password" \
  --data-urlencode "csrf_token=$CSRF" "$BASE/auth/login"
# record the log length BEFORE the request, then capture EXACTLY the new lines it appends
mark=$(wc -l < /tmp/gunicorn_7777.log)
curl -s -i -b "$JAR" -X DELETE "$BASE/api/user"          # session fallback -> g.api_key=None -> 500
tail -n +$((mark+1)) /tmp/gunicorn_7777.log              # the server-side traceback for case (a)
# (b) real API key (id=134) with sudo_mode_at = NULL, via Authentication header
KEY134=$(psql "$DB_URI" -t -A -c "select code from api_key where id=134;")   # 60-char throwaway code
curl -s -i -H "Authentication: $KEY134" -X DELETE "$BASE/api/user"           # -> 440 {"error":"Need sudo"}
# NOTE: the id=134 code (created via ApiKey.create for john@wick.com) is a throwaway credential
# scoped to the disposable investigation container and its fake FLASK_SECRET=secret; it is shown
# REDACTED as <Q2_API_KEY_REDACTED> in the output below.
```

**(c) Complete unedited output.**

```
========== Q2 (a): browser session ONLY (john cookie, NO Authentication header) → DELETE /api/user ==========
$ mark=$(wc -l < /tmp/gunicorn_7777.log)   # record log length BEFORE the request
mark=41
$ curl -s -i -b /tmp/q1_cookies.txt -X DELETE http://localhost:7777/api/user
HTTP/1.1 500 INTERNAL SERVER ERROR
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 05:16:03 GMT
Connection: close
Content-Type: application/json
Content-Length: 27
Access-Control-Allow-Origin: *
Set-Cookie: slapp=040cc8b9-3c48-4ae9-9263-7bafee8ec84a.fTwS48F-c6v9UcurEmdwzY3WjMw; Expires=Wed, 15-Jul-2026 05:16:03 GMT; HttpOnly; Path=/; SameSite=Lax

{"error":"Internal error"}


========== gunicorn log NEW lines after Q2(a) — the traceback ==========
$ tail -n +$((mark+1)) /tmp/gunicorn_7777.log
2026-07-08 05:16:03,738 - SL - ERROR - 2595 - "/app/server.py:390" - error_handler() -  - 'NoneType' object has no attribute 'sudo_mode_at'
Traceback (most recent call last):
  File "/app/venv/lib/python3.10/site-packages/flask/app.py", line 1950, in full_dispatch_request
    rv = self.dispatch_request()
  File "/app/venv/lib/python3.10/site-packages/flask/app.py", line 1936, in dispatch_request
    return self.view_functions[rule.endpoint](**req.view_args)
  File "/app/app/api/base.py", line 69, in decorated
    if not check_sudo_mode_is_active(g.api_key):
  File "/app/app/api/base.py", line 47, in check_sudo_mode_is_active
    return api_key.sudo_mode_at and g.api_key.sudo_mode_at >= arrow.now().shift(
AttributeError: 'NoneType' object has no attribute 'sudo_mode_at'
2026-07-08 05:16:03,738 - SL - DEBUG - 2595 - "/app/server.py:284" - after_request() -  - 127.0.0.1 DELETE /api/user ImmutableMultiDict([]) 500, takes 0.0034606456756591797

========== Q2 (b): real API key WITHOUT active sudo (Authentication header, sudo_mode_at=NULL) → DELETE /api/user ==========
$ curl -s -i -H 'Authentication: <Q2_API_KEY_REDACTED>' -X DELETE http://localhost:7777/api/user
HTTP/1.1 440 UNKNOWN
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 05:16:03 GMT
Connection: close
Content-Type: application/json
Content-Length: 22
Access-Control-Allow-Origin: *
Set-Cookie: slapp=f3dfe085-9b65-4811-b8dc-27372f1d8dcf.e-EYBjWbKVwtHIToJnumaJIMNB8; Expires=Wed, 15-Jul-2026 05:16:03 GMT; HttpOnly; Path=/; SameSite=Lax

{"error":"Need sudo"}
```

**(d) Concrete observed values.**
- **(a) browser session only:** `HTTP/1.1 500 INTERNAL SERVER ERROR`; body
  `{"error":"Internal error"}`; server log:
  `AttributeError: 'NoneType' object has no attribute 'sudo_mode_at'`.
- **(b) real API key, no active sudo:** `HTTP/1.1 440 UNKNOWN`; body `{"error":"Need sudo"}`.
  Note the reason phrase is literally `UNKNOWN` — Werkzeug has no standard phrase for the
  non‑standard 440 code (440 is a Microsoft IIS "Login Time‑out", not an IETF status).

**(e) `file:line` grounding.** `require_api_sudo` calls `authorize_request()`
(`app/api/base.py:66`) then `if not check_sudo_mode_is_active(g.api_key): return
jsonify(error="Need sudo"), 440` (`app/api/base.py:69-70`). `check_sudo_mode_is_active`
returns `api_key.sudo_mode_at and g.api_key.sudo_mode_at >= arrow.now().shift(minutes=-SUDO_MODE_MINUTES_VALID)`
(`app/api/base.py:46-49`). Under the browser‑session fallback, `authorize_request()` never
assigns a key, leaving `g.api_key = None` (`app/api/base.py:42`). The sole sudo endpoint is
`DELETE /api/user` (`app/api/views/user.py:12-14`). The 500 body/logging comes from the app's
error handler (`server.py:390`).

**(f) Cause→effect.** Case (a): the session fallback sets `g.user` but leaves `g.api_key =
None`; `check_sudo_mode_is_active(None)` evaluates `None.sudo_mode_at`, raising
`AttributeError`, which the global error handler turns into **500 `{"error":"Internal error"}`**.
Case (b): a real key is present, so `authorize_request()` succeeds and `g.api_key` is the key;
its `sudo_mode_at` is `NULL`, so `check_sudo_mode_is_active` returns a falsy value and the
decorator returns **440 `{"error":"Need sudo"}`**.

**(g) OBSERVED vs INFERRED.** During scoping, the 500 outcome for the pure browser session was
**INFERRED** from reading (a `None` dereference). It is now **OBSERVED**: the 500 status, the
`{"error":"Internal error"}` body, and the exact `AttributeError` traceback at
`app/api/base.py:47` (called from `:69`) were all captured at runtime. The 440 outcome is also
**OBSERVED**.

---

## Q3 — Session storage representation

**(a) Direct answer.** For an authenticated session the value stored in Redis is a **Python
`pickle`** byte string of `dict(session)`, held under a key of the form **`session:<uuid4>`**.
The deserialized session contains the keys `_permanent`, `_fresh`, `csrf_token`, `_user_id`,
`_id`, `sudo_time`. The `slapp` cookie is `<session_id:uuid4>.<base64 HMAC‑SHA1 signature>`;
the part before the final `.` is the `session_id`, and the Redis key is that id prefixed with
`session:`.

**(b) Exact commands.** Log in (as in Q1), then unsign the cookie, read the raw bytes directly
from Redis, and deserialize with `pickle`.

```python
import re, pickle, itsdangerous, redis
from app import config

jar = open("/tmp/q3_cookies.txt").read()
cookie_val = re.search(r"slapp\s+(\S+)", jar).group(1)

# same signer as RedisSessionStore._get_signer (app/session.py:39-41)
signer = itsdangerous.Signer(config.FLASK_SECRET, salt="session", key_derivation="hmac")
session_id = signer.unsign(cookie_val).decode()
key = "session:%s" % session_id          # app/session.py:18,44-45

r = redis.from_url(config.MEM_STORE_URI)
raw = r.get(key)
print(repr(raw))                          # RAW BYTES
data = pickle.loads(raw)                   # app/session.py:76,91  -> format = pickle
print(sorted(data.keys()))
for k, v in data.items():
    print("   %-14s = %r" % (k, v))
print("TTL:", r.ttl(key))
```

**(c) Complete unedited output.**

```
slapp cookie value (raw, from browser jar):
   b5e4833a-e788-4b13-b2ee-5cfa578d45a7.pYQdoTxcHFs93-uP6YfNwbkTpx0
  structure = <session_id:uuid4> + "." + <base64 hmac-sha1 signature>

unsigned session_id (uuid4): b5e4833a-e788-4b13-b2ee-5cfa578d45a7
Redis key: session:b5e4833a-e788-4b13-b2ee-5cfa578d45a7

=== RAW BYTES stored in Redis (repr) ===
b'\x80\x04\x95!\x01\x00\x00\x00\x00\x00\x00}\x94(\x8c\n_permanent\x94\x88\x8c\x06_fresh\x94\x88\x8c\ncsrf_token\x94\x8c(3bd5ab114476e286bd8f4466a918f5b775580df6\x94\x8c\x08_user_id\x94\x8c$b8c76184-7578-4c9f-b992-7df89f495d60\x94\x8c\x03_id\x94\x8c\x80b03643a8a515eae10966eb5799d0b928b1fae8daeb0b0a33ba33f35b5fd7e09d574a2b38cd3af3ce53bdb464d28d2c515533674c8b087cba7d49abfa2cc9de57\x94\x8c\tsudo_time\x94J\xb0\xd0Mju.'

=== RAW BYTES length === 300 bytes
first byte = 0x80 (pickle PROTO opcode 0x80 => pickle protocol marker)

=== pickle.loads(raw) -> deserialized dict ===
type: <class 'dict'>
enumerated keys: ['_fresh', '_id', '_permanent', '_user_id', 'csrf_token', 'sudo_time']
full dict:
   _permanent     = True
   _fresh         = True
   csrf_token     = '3bd5ab114476e286bd8f4466a918f5b775580df6'
   _user_id       = 'b8c76184-7578-4c9f-b992-7df89f495d60'
   _id            = 'b03643a8a515eae10966eb5799d0b928b1fae8daeb0b0a33ba33f35b5fd7e09d574a2b38cd3af3ce53bdb464d28d2c515533674c8b087cba7d49abfa2cc9de57'
   sudo_time      = 1783484592

=== Redis TTL on the key (seconds) === 604800
(7 days = 604800s for authenticated session; app/session.py:92-96, server.py:207)
```

**(d) Concrete observed values.**
- **Raw bytes (300 bytes):**
  `b'\x80\x04\x95!\x01\x00\x00\x00\x00\x00\x00}\x94(\x8c\n_permanent\x94\x88\x8c\x06_fresh\x94\x88\x8c\ncsrf_token\x94\x8c(3bd5ab114476e286bd8f4466a918f5b775580df6\x94\x8c\x08_user_id\x94\x8c$b8c76184-7578-4c9f-b992-7df89f495d60\x94\x8c\x03_id\x94\x8c\x80b03643a8a515eae10966eb5799d0b928b1fae8daeb0b0a33ba33f35b5fd7e09d574a2b38cd3af3ce53bdb464d28d2c515533674c8b087cba7d49abfa2cc9de57\x94\x8c\tsudo_time\x94J\xb0\xd0Mju.'`
- **Serialization format:** `pickle` (leading `\x80\x04` = pickle protocol 4 `PROTO` opcode).
- **Deserialized keys:** `_permanent=True`, `_fresh=True`,
  `csrf_token='3bd5ab114476e286bd8f4466a918f5b775580df6'`,
  `_user_id='b8c76184-7578-4c9f-b992-7df89f495d60'` (a UUID — Flask‑Login stores
  `User.get_id()`), `_id=<128‑hex Flask‑Login session identifier>`, `sudo_time=1783484592`.
- **Redis key structure:** `session:b5e4833a-e788-4b13-b2ee-5cfa578d45a7` (`session:<uuid4>`);
  cookie `b5e4833a-e788-4b13-b2ee-5cfa578d45a7.pYQdoTxcHFs93-uP6YfNwbkTpx0`; TTL `604800`s.

**(e) `file:line` grounding.** `SESSION_PREFIX = "session"` (`app/session.py:18`); the key is
`f"{SESSION_PREFIX}:{session_Id}"` (`app/session.py:44-45`). `save_session` computes
`val = pickle.dumps(dict(session))` (`app/session.py:91`) and stores it with `setex`
(`app/session.py:97-101`); `open_session` reads it back with `pickle.loads` (`app/session.py:76`).
The cookie signer is `itsdangerous.Signer(app.secret_key, salt="session",
key_derivation="hmac")` (`app/session.py:39-41`), signed at `app/session.py:102-104`. The TTL
is `permanent_session_lifetime` for authenticated sessions, or `300`s when `_user_id` is absent
(`app/session.py:92-96`); the 7‑day lifetime is set at `server.py:207`. The store is installed
at `app/redis_services.py:12` (and `:17` for sentinel).

**(f) Cause→effect.** Flask calls `save_session` at the end of the request; because the custom
`RedisSessionStore` serializes with `pickle.dumps(dict(session))`, the on‑disk (in‑Redis)
representation is a pickle stream (hence the leading `\x80\x04`). The session id is a
`uuid4` minted server‑side, stored in the cookie only in signed form, and used verbatim (with
the `session:` prefix) as the Redis key. Because `_user_id` is present (the user is logged in),
the TTL is the 7‑day `permanent_session_lifetime` = `604800`s rather than the 300s used for
anonymous sessions.

**(g) OBSERVED vs INFERRED.** Everything here is **OBSERVED**: the raw bytes and TTL were read
directly from Redis, and the key list came from `pickle.loads` of those exact bytes.

---

## Q4 — Session identifier behavior across login (session‑fixation check)

**(a) Direct answer.** The session identifier is **UNCHANGED across login — it is NOT rotated**.
The `slapp` cookie captured before login and after a successful login is **byte‑for‑byte
identical**, and both unsign to the same `uuid4` session id. (Per project scope this is
**reported, not fixed**.)

**(b) Exact commands.** Capture the `Set-Cookie` from the pre‑login `GET /auth/login`, then from
the post‑login `POST`, and unsign both.

```bash
BASE=http://localhost:7777
# BEFORE: GET login page, save pre-login cookie + headers
curl -s -D /tmp/q4_pre_headers.txt -c /tmp/q4_pre.txt "$BASE/auth/login" >/tmp/page.html
CSRF=$(grep -oP 'name="csrf_token"[^>]*value="\K[^"]+' /tmp/page.html | head -1)
# AFTER: POST credentials, save post-login cookie + headers
curl -s -D /tmp/q4_post_headers.txt -o /dev/null -b /tmp/q4_pre.txt -c /tmp/q4_post.txt \
  --data-urlencode "email=john@wick.com" --data-urlencode "password=password" \
  --data-urlencode "csrf_token=$CSRF" "$BASE/auth/login"
```
```python
import re, itsdangerous
from app import config
signer = itsdangerous.Signer(config.FLASK_SECRET, salt="session", key_derivation="hmac")
sid = lambda p: signer.unsign(re.search(r"slapp\s+(\S+)", open(p).read()).group(1)).decode()
print("BEFORE:", sid("/tmp/q4_pre.txt"))
print("AFTER :", sid("/tmp/q4_post.txt"))
```

**(c) Complete unedited output.**

```
===== STEP 1: GET /auth/login  (BEFORE login) — capture pre-login Set-Cookie =====
Set-Cookie: slapp=1b76a2a2-998f-42ac-9f48-756e72e16a0c.IRQF_oMCC6N8AmAL7AmCabzwv88; Expires=Wed, 15-Jul-2026 04:23:40 GMT; HttpOnly; Path=/; SameSite=Lax
csrf_token (from pre-login page): ImZlZWYxMDJlZmI0YTFmNjcyN2QyZGFlZTllZDMyNGQ5MDBmODJhMjEi.ak3QzA.BREediCKf32D1jCgrtZL7xquSSw

===== STEP 2: POST /auth/login john@wick.com (AFTER login) — capture post-login Set-Cookie =====
HTTP/1.1 302 FOUND
Location: http://localhost:7777/dashboard/
Set-Cookie: slapp=1b76a2a2-998f-42ac-9f48-756e72e16a0c.IRQF_oMCC6N8AmAL7AmCabzwv88; Expires=Wed, 15-Jul-2026 04:23:40 GMT; HttpOnly; Path=/; SameSite=Lax

BEFORE login:
  raw slapp cookie : 1b76a2a2-998f-42ac-9f48-756e72e16a0c.IRQF_oMCC6N8AmAL7AmCabzwv88
  unsigned session_id: 1b76a2a2-998f-42ac-9f48-756e72e16a0c
AFTER login:
  raw slapp cookie : 1b76a2a2-998f-42ac-9f48-756e72e16a0c.IRQF_oMCC6N8AmAL7AmCabzwv88
  unsigned session_id: 1b76a2a2-998f-42ac-9f48-756e72e16a0c

session_id SAME across login?  -> True
CONCLUSION: UNCHANGED / NOT ROTATED (session-fixation consideration)
```

**(d) Concrete observed values.**
- **Before login:** cookie `1b76a2a2-998f-42ac-9f48-756e72e16a0c.IRQF_oMCC6N8AmAL7AmCabzwv88`;
  session id `1b76a2a2-998f-42ac-9f48-756e72e16a0c`.
- **After login:** cookie `1b76a2a2-998f-42ac-9f48-756e72e16a0c.IRQF_oMCC6N8AmAL7AmCabzwv88`;
  session id `1b76a2a2-998f-42ac-9f48-756e72e16a0c`.
- **Same?** `True` → **UNCHANGED / NOT ROTATED** (the entire signed cookie, including the HMAC,
  is identical).

**(e) `file:line` grounding.** The login path is `after_login()` → `login_user(user)`
(`app/auth/views/login_utils.py:36`) followed by `session["sudo_time"] = int(time())`
(`app/auth/views/login_utils.py:37`). Flask‑Login is configured with
`login_manager.session_protection = "strong"` (`app/extensions.py:8`). The custom
`RedisSessionStore` has **no** `regenerate`/rotation method; `open_session` reuses the `uuid4`
extracted from the incoming (pre‑login CSRF) cookie (`app/session.py:68-80`, mint at `:71`/`:80`
only when none is present). `login_user` merely adds `_user_id` to the **same** session dict.

**(f) Cause→effect.** The pre‑login `GET` already establishes a session (needed to hold the
CSRF token), minting `session_id = 1b76a2a2…`. On login, `login_user` adds `_user_id` to that
existing session dict without changing `session.session_id`; since there is no rotation hook,
`save_session` re‑signs the **same** id, producing the **identical** cookie. Hence the id is
stable across the authentication boundary — a session‑fixation consideration.

**(g) OBSERVED vs INFERRED.** The pre‑scoping hypothesis (from reading) was "unchanged"; this is
now **OBSERVED** — the before/after cookies and unsigned ids match exactly. Per AAP §0.5.2 this
behavior is **reported, not fixed**.

---


## Q5 — Email‑forwarding header handling

**(a) Direct answer.** Of the three headers on the incoming message: the custom `X-Custom-Test`
header is **STRIPPED**, the `Received` header is **STRIPPED**, and the original `Reply-To` is
**STRIPPED and then REPLACED** with a new reverse‑alias value (a *different* address, not the
original). Additionally the `From` header is **rewritten** to a reverse‑alias. Only headers on
the `headers_to_keep` allow‑list survive.

**(b) Exact commands.** A raw `.eml` carrying the three headers is parsed and driven through the
real `email_handler.handle_forward(...)` path against a freshly seeded alias; the outbound
`sl_sendmail` call is intercepted at runtime (no source modification) to capture the exact
forwarded message. The complete self‑contained observation script `/tmp/q5_probe.py` is run
inside the container and its **entire** stdout+stderr is captured with **no** filtering:

```bash
docker exec sl bash -c 'cd /app && . venv/bin/activate \
  && eval "$(grep "^export " /build.sh)" \
  && python /tmp/q5_probe.py' 2>&1
```

The observation script, run verbatim:

```python
#!/usr/bin/env python3
"""Q5 - empirical header-handling probe for the email-forwarding path.

Drives a crafted incoming message (custom X-* header + Received + Reply-To)
through the REAL email_handler.handle_forward(...) path against a freshly
seeded alias, intercepts the outbound sl_sendmail call at runtime (no source
modification), and dumps the forwarded message's complete header set so we can
observe which of the three sibling headers survive and which are stripped.

Usage:  python /tmp/q5_probe.py
"""
import email
from aiosmtpd.smtp import Envelope

import email_handler
from app.models import User, Alias, DeletedAlias

PROBE_EMAIL = "q5-forward-probe@sl.local"

# --- ensure the probe alias is free (idempotent re-run guard) -------------
_existing = Alias.get_by(email=PROBE_EMAIL)
if _existing is not None:
    from app.alias_utils import delete_alias
    delete_alias(_existing, _existing.user, commit=True)
_trash = DeletedAlias.get_by(email=PROBE_EMAIL)
if _trash is not None:
    DeletedAlias.delete(_trash.id, commit=True)

# --- intercept the outbound forwarded message (runtime monkeypatch) -------
captured = {}
def _capture(from_addr, to_addr, msg, mail_options=(), rcpt_options=(),
             is_forward=False, **kw):
    captured["from_addr"], captured["to_addr"], captured["msg"] = from_addr, to_addr, msg
email_handler.sl_sendmail = _capture

u = User.get_by(email="john@wick.com")
alias = Alias.create(user_id=u.id, email=PROBE_EMAIL, mailbox_id=1, commit=True)
print("Seeded alias: %s -> mailbox_id= %s enabled= %s" % (alias.email, alias.mailbox_id, alias.enabled))

raw_eml = (
 "From: External Sender <sender@external.com>\n"
 "To: q5-forward-probe@sl.local\n"
 "Subject: Q5 header-handling probe\n"
 "Date: Wed, 08 Jul 2026 04:00:00 +0000\n"
 "Message-ID: <q5-probe-0001@external.com>\n"
 "Received: from mail.external.com (mail.external.com [203.0.113.9]) by mx.sl.local"
 " with ESMTP id ABC123; Wed, 08 Jul 2026 04:00:01 +0000\n"
 "Reply-To: Reply Sender <replyto-sender@external.com>\n"
 "X-Custom-Test: hello\n"
 "Content-Type: text/plain; charset=us-ascii\n"
 "Content-Transfer-Encoding: 7bit\nMIME-Version: 1.0\n\nbody\n")
msg = email.message_from_string(raw_eml)

print("")
print("===== ORIGINAL (incoming) message headers \u2014 BEFORE forwarding =====")
for k, v in msg.items():
    print("  %-28s: %s" % (k, v))

print("")
print("===== Driving email_handler.handle_forward(envelope, msg, rcpt_to='q5-forward-probe@sl.local') =====")
env = Envelope()
env.mail_from = "sender@external.com"
env.rcpt_tos = ["q5-forward-probe@sl.local"]
env.mail_options = []
env.rcpt_options = []
res = email_handler.handle_forward(env, msg, "q5-forward-probe@sl.local")
print("handle_forward returned: %r" % (res,))

print("")
print("===== FORWARDED message \u2014 COMPLETE header set (verbatim, AFTER forwarding) =====")
print("sl_sendmail from_addr: %s" % captured["from_addr"])
print("sl_sendmail to_addr  : %s" % captured["to_addr"])
print("---- forwarded headers ----")
fwd = captured["msg"]
for k, v in fwd.items():
    print("  %-28s: %s" % (k, v))

print("")
print("===== PER-HEADER VERDICT (Q5 three siblings) =====")
def verdict(name):
    vals = fwd.get_all(name)
    if vals is None:
        return "<STRIPPED / absent>"
    return repr(vals)
print("  %-16s AFTER-forward -> %s" % ("X-Custom-Test", verdict("X-Custom-Test")))
print("  %-16s AFTER-forward -> %s" % ("Received", verdict("Received")))
print("  %-16s AFTER-forward -> %s" % ("Reply-To", verdict("Reply-To")))
print("  %-16s AFTER-forward -> %s" % ("From", verdict("From")))
```

**(c) Complete unedited output.** The full, unfiltered stdout+stderr of the command above
(including the SimpleLogin import/init‑logging lines `>>> URL`, `Upload files to local dir`,
`>>> init logging <<<`, and `load words file`) is reproduced below in its entirety.

```
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-08 05:32:36,517 - SL - DEBUG - 3106 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-08 05:32:37,251 - SL - INFO - 3106 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
Seeded alias: q5-forward-probe@sl.local -> mailbox_id= 1 enabled= True

===== ORIGINAL (incoming) message headers — BEFORE forwarding =====
  From                        : External Sender <sender@external.com>
  To                          : q5-forward-probe@sl.local
  Subject                     : Q5 header-handling probe
  Date                        : Wed, 08 Jul 2026 04:00:00 +0000
  Message-ID                  : <q5-probe-0001@external.com>
  Received                    : from mail.external.com (mail.external.com [203.0.113.9]) by mx.sl.local with ESMTP id ABC123; Wed, 08 Jul 2026 04:00:01 +0000
  Reply-To                    : Reply Sender <replyto-sender@external.com>
  X-Custom-Test               : hello
  Content-Type                : text/plain; charset=us-ascii
  Content-Transfer-Encoding   : 7bit
  MIME-Version                : 1.0

===== Driving email_handler.handle_forward(envelope, msg, rcpt_to='q5-forward-probe@sl.local') =====
2026-07-08 05:32:37,263 - SL - DEBUG - 3106 - "/app/email_handler.py:580" - handle_forward() -  - Create or get contact for from_header:External Sender <sender@external.com>
2026-07-08 05:32:37,287 - SL - DEBUG - 3106 - "/app/app/contact_utils.py:110" - create_contact() -  - Created contact <Contact 299 sender@external.com 1574> for alias <Alias 1574 q5-forward-probe@sl.local> with email sender@external.com invalid_email=False
2026-07-08 05:32:37,287 - SL - DEBUG - 3106 - "/app/email_handler.py:589" - handle_forward() -  - Create or get contact for reply_to_header:Reply Sender <replyto-sender@external.com>
2026-07-08 05:32:37,308 - SL - DEBUG - 3106 - "/app/app/contact_utils.py:110" - create_contact() -  - Created contact <Contact 300 replyto-sender@external.com 1574> for alias <Alias 1574 q5-forward-probe@sl.local> with email replyto-sender@external.com invalid_email=False
2026-07-08 05:32:37,309 - SL - INFO - 3106 - "/app/app/handler/dmarc.py:33" - apply_dmarc_policy_for_forward_phase() -  - DMARC check disabled
2026-07-08 05:32:37,316 - SL - DEBUG - 3106 - "/app/email_handler.py:688" - forward_email_to_mailbox() -  - Forward <Contact 299 sender@external.com 1574> -> <Alias 1574 q5-forward-probe@sl.local> -> <Mailbox 1 john@wick.com>
2026-07-08 05:32:37,323 - SL - DEBUG - 3106 - "/app/email_handler.py:740" - forward_email_to_mailbox() -  - Create <EmailLog 622> for <Contact 299 sender@external.com 1574>, <User 1 John Wick john@wick.com>, <Mailbox 1 john@wick.com>
2026-07-08 05:32:37,329 - SL - DEBUG - 3106 - "/app/email_handler.py:867" - forward_email_to_mailbox() -  - From header, new:"External Sender - sender at external.com" <sender_at_external_com_mbciholwr@sl.local>, old:External Sender <sender@external.com>
2026-07-08 05:32:37,330 - SL - DEBUG - 3106 - "/app/email_handler.py:873" - forward_email_to_mailbox() -  - Reply-To header, new:"Reply Sender - replyto-sender at external.com" <replyto-sender_at_external_com_bopmfldug@sl.local>, old:None
2026-07-08 05:32:37,330 - SL - DEBUG - 3106 - "/app/email_handler.py:316" - replace_header_when_forward() -  - Delete Cc header, old value None
2026-07-08 05:32:37,330 - SL - DEBUG - 3106 - "/app/email_handler.py:313" - replace_header_when_forward() -  - Replace To header, old: q5-forward-probe@sl.local, new: q5-forward-probe@sl.local
2026-07-08 05:32:37,330 - SL - INFO - 3106 - "/app/app/handler/unsubscribe_generator.py:36" - _generate_header_with_original_behaviour() -  - Email has no unsubscribe header
2026-07-08 05:32:37,333 - SL - DEBUG - 3106 - "/app/email_handler.py:893" - forward_email_to_mailbox() -  - Forward mail from sender@external.com to john@wick.com, mail_options:[], rcpt_options:[]
handle_forward returned: [(True, '250 Message accepted for delivery')]

===== FORWARDED message — COMPLETE header set (verbatim, AFTER forwarding) =====
sl_sendmail from_addr: sl.lmycyibwgizcyibsgm3tiobzgjoq.stjwcxvgwxhze@sl.local
sl_sendmail to_addr  : john@wick.com
---- forwarded headers ----
  Subject                     : Q5 header-handling probe
  Date                        : Wed, 08 Jul 2026 04:00:00 +0000
  Message-ID                  : <q5-probe-0001@external.com>
  Content-Type                : text/plain; charset=us-ascii
  Content-Transfer-Encoding   : 7bit
  MIME-Version                : 1.0
  X-SimpleLogin-Type          : Forward
  X-SimpleLogin-EmailLog-ID   : 622
  X-SimpleLogin-Envelope-From : sender@external.com
  X-SimpleLogin-Original-From : External Sender <sender@external.com>
  X-SimpleLogin-Envelope-To   : q5-forward-probe@sl.local
  From                        : "External Sender - sender at external.com" <sender_at_external_com_mbciholwr@sl.local>
  Reply-To                    : "Reply Sender - replyto-sender at external.com" <replyto-sender_at_external_com_bopmfldug@sl.local>
  To                          : q5-forward-probe@sl.local
  DKIM-Signature              : v=1; a=rsa-sha256; c=relaxed/simple; d=sl.local;  i=@sl.local; q=dns/txt; s=dkim; t=1783488757; h=message-id : date :  subject : from : to; bh=Ck5SoRNWUpSR4X0COv7R5ub2pUTtl6xz4dTFz++ji4M=;  b=Hc6cE9Hhtv22rrDoQsoTgx70ITkHnnYCD6iUKgRGknrhBYzz5JD0sXKWk/kLxGfjDKnej  8/YuHwPKR+UQUqfbzfBCj0ALku/DTz69qCg3jxL0wtXZK7TWHNnb9PJQFz5aKSnX5Km2onN  ypt7LGhfryJon9woYVF4n0/P+5o1hE4=

===== PER-HEADER VERDICT (Q5 three siblings) =====
  X-Custom-Test    AFTER-forward -> <STRIPPED / absent>
  Received         AFTER-forward -> <STRIPPED / absent>
  Reply-To         AFTER-forward -> ['"Reply Sender - replyto-sender at external.com" <replyto-sender_at_external_com_bopmfldug@sl.local>']
  From             AFTER-forward -> ['"External Sender - sender at external.com" <sender_at_external_com_mbciholwr@sl.local>']
```

**(d) Concrete observed values (all three sibling items).**
- **Custom `X-Custom-Test: hello`** — present before → **STRIPPED** (absent) after.
- **`Received: from mail.external.com …`** — present before → **STRIPPED** (absent) after.
- **`Reply-To: Reply Sender <replyto-sender@external.com>`** — present before → original
  **STRIPPED**, then **REPLACED** with
  `"Reply Sender - replyto-sender at external.com" <replyto-sender_at_external_com_bopmfldug@sl.local>`
  (a reverse‑alias, a *different* value).
- (Related) **`From`** rewritten from `External Sender <sender@external.com>` to
  `"External Sender - sender at external.com" <sender_at_external_com_mbciholwr@sl.local>`.
- Headers that **survived** (on the allow‑list): `Subject`, `Date`, `Message-ID`,
  `Content-Type`, `Content-Transfer-Encoding`, `MIME-Version`, `To`. SimpleLogin then adds its
  own `X-SimpleLogin-*` and `DKIM-Signature` headers.

**(e) `file:line` grounding.** The forward path builds `headers_to_keep`
(`email_handler.py:793-807`) = `From, To, Cc, Subject, Date, Message-ID, References,
In-Reply-To, X-SL-Queue-Id, List-Unsubscribe, List-Unsubscribe-Post` + `MIME_HEADERS`
(`Mime-Version, Content-Type, Content-Disposition, Content-Transfer-Encoding` — defined at
`app/email/headers.py:44-49`), optionally appends `Authentication-Results` if
`user.include_header_email_header` (`email_handler.py:808-809`), then calls
`delete_all_headers_except(msg, headers_to_keep)` (`email_handler.py:810`). That function
lower‑cases the allow‑list and deletes any header not on it (`app/email_utils.py:536-542`).
`Reply-To` / `Received` name constants are at `app/email/headers.py:13-14`. After stripping,
`From` is rewritten to `contact.new_addr()` (`email_handler.py:864-866`) and, when a
`reply_to_contact` exists, a **new** `Reply-To = reply_to_contact.new_addr()` is added
(`email_handler.py:869-872`). The `reply_to_contact` itself is created from the original
`Reply-To` earlier in `handle_forward` (`email_handler.py:587-594`).

**(f) Cause→effect.** `delete_all_headers_except` keeps only allow‑listed headers, so
`X-Custom-Test` and `Received` (neither on the list) are removed. The original `Reply-To` is
also removed by the same allow‑list step — note the log line at `email_handler.py:873` reads
`old:None`, proving the original `Reply-To` was already stripped at `:810` **before** the
rewrite at `:872`. Because a `reply_to_contact` had been created from the original `Reply-To`
(captured at `:587-594` before stripping), a brand‑new reverse‑alias `Reply-To` is then added.
Thus the recipient sees a *different* `Reply-To`, not the sender's original.

**(g) OBSERVED vs INFERRED.** All verdicts are **OBSERVED** from the intercepted forwarded
message. `handle_forward` returned `[(True, '250 Message accepted for delivery')]` and
`DMARC check disabled` (`app/handler/dmarc.py:33`), so no DMARC quarantine interfered with the
canonical forward.

---


## Q6 — Alias‑creation token expiry window

**(a) Direct answer.** The signed alias‑suffix token is valid for **600 seconds (10 minutes)**.
`check_suffix_signature` returns the suffix immediately after signing and continues to return it
right up to age 600s, then returns `None` once the token is older than 600s. Two independent
runs bracket the boundary consistently: last valid at `t=600.001s`, first expired at
`t=601.001s`/`t=601.002s`.

**(b) Exact commands.** Sign a suffix with the **real** `app/alias_suffix.py` signer, then probe
`check_suffix_signature` at increasing ages that cross the 600s boundary. The complete
self‑contained script `/tmp/q6_timing.py` (below) takes a run label and selects a probe schedule
via `PROBE_SCHEDULES[run]` (Run A probes `0,300,590,598,599,600,601,602,605,610`s over ~610s of
wall‑clock; Run B probes `0,595,599,601,603`s over ~603s). Two runs are launched detached, in
parallel, so their overlapping schedules bracket the boundary for stability.

```python
#!/usr/bin/env python3
"""Q6 — empirical expiry-window probe for the alias-suffix signed token.

Signs a suffix with the REAL app/alias_suffix.py signer, then calls the REAL
check_suffix_signature() at a schedule of increasing ages that straddle the
max_age=600 boundary (app/alias_suffix.py:40). Prints, for each probe, the
monotonic elapsed time and whether the token is still VALID or has EXPIRED.

Usage:  python /tmp/q6_timing.py <RUN_LABEL>
        RUN_LABEL selects a probe schedule: "A" (dense, ~610s) or "B" (~603s).
"""
import sys
import time

from app import config                                   # canonical config
from app.alias_suffix import signer, check_suffix_signature  # real module

# Probe schedules (seconds since signing). Two independent schedules give an
# overlapping, stability-confirming bracket around the 600s boundary.
PROBE_SCHEDULES = {
    "A": [0, 300, 590, 598, 599, 600, 601, 602, 605, 610],
    "B": [0, 595, 599, 601, 603],
}

run = sys.argv[1] if len(sys.argv) > 1 else "A"
probes = PROBE_SCHEDULES[run]
tag = "[RUN %s]" % run

suffix = ".investigation-q6@example.com"
signed = signer.sign(suffix).decode()                    # app/alias_suffix.py:114-style

print("%s CUSTOM_ALIAS_SECRET=%r (=FLASK_SECRET+\"custom_alias\", app/config.py:201)"
      % (tag, config.CUSTOM_ALIAS_SECRET))
print("%s signer=itsdangerous.TimestampSigner (app/alias_suffix.py:11); "
      "check_suffix_signature max_age=600 (app/alias_suffix.py:40)" % tag)
print("%s signed_suffix=%r" % (tag, signed))
sys.stdout.flush()

t0 = time.monotonic()
for p in probes:
    now = time.monotonic() - t0
    if p > now:
        time.sleep(p - now)
    elapsed = time.monotonic() - t0
    res = check_suffix_signature(signed)                 # app/alias_suffix.py:37-42, max_age=600
    if res is None:
        verdict = "EXPIRED (returns None)"
        shown = "None"
        print("%s t=%8.3fs  check_suffix_signature -> %-31s %s"
              % (tag, elapsed, shown, verdict))
    else:
        verdict = "VALID (returns suffix)"
        print("%s t=%8.3fs  check_suffix_signature -> %-31r %s"
              % (tag, elapsed, res, verdict))
    sys.stdout.flush()
print("%s DONE" % tag)
sys.stdout.flush()
```

Launch (both detached, in parallel), then collect each complete log after they finish:

```bash
docker exec -d sl bash -c 'cd /app && . venv/bin/activate \
  && eval "$(grep "^export " /build.sh)" \
  && python /tmp/q6_timing.py A > /tmp/q6_runA.log 2>&1'
docker exec -d sl bash -c 'cd /app && . venv/bin/activate \
  && eval "$(grep "^export " /build.sh)" \
  && python /tmp/q6_timing.py B > /tmp/q6_runB.log 2>&1'
# after both finish (~610s / ~603s):
docker exec sl cat /tmp/q6_runA.log
docker exec sl cat /tmp/q6_runB.log
```

**(c) Complete unedited output.** The full, unfiltered logs of both runs (including the
SimpleLogin import/init‑logging lines `>>> URL`, `Upload files to local dir`,
`>>> init logging <<<`, and `load words file`) are reproduced below in their entirety.

Run A — `docker exec sl cat /tmp/q6_runA.log`:

```
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-08 05:09:05,208 - SL - DEBUG - 2539 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
[RUN A] CUSTOM_ALIAS_SECRET='secretcustom_alias' (=FLASK_SECRET+"custom_alias", app/config.py:201)
[RUN A] signer=itsdangerous.TimestampSigner (app/alias_suffix.py:11); check_suffix_signature max_age=600 (app/alias_suffix.py:40)
[RUN A] signed_suffix='.investigation-q6@example.com.ak3bcQ.fhOAjYecEx8GImx0RDpJ9Zt3pZM'
[RUN A] t=   0.000s  check_suffix_signature -> '.investigation-q6@example.com' VALID (returns suffix)
[RUN A] t= 300.079s  check_suffix_signature -> '.investigation-q6@example.com' VALID (returns suffix)
[RUN A] t= 590.100s  check_suffix_signature -> '.investigation-q6@example.com' VALID (returns suffix)
[RUN A] t= 598.004s  check_suffix_signature -> '.investigation-q6@example.com' VALID (returns suffix)
[RUN A] t= 599.001s  check_suffix_signature -> '.investigation-q6@example.com' VALID (returns suffix)
[RUN A] t= 600.001s  check_suffix_signature -> '.investigation-q6@example.com' VALID (returns suffix)
[RUN A] t= 601.001s  check_suffix_signature -> None                            EXPIRED (returns None)
[RUN A] t= 602.001s  check_suffix_signature -> None                            EXPIRED (returns None)
[RUN A] t= 605.003s  check_suffix_signature -> None                            EXPIRED (returns None)
[RUN A] t= 610.005s  check_suffix_signature -> None                            EXPIRED (returns None)
[RUN A] DONE
```

Run B — `docker exec sl cat /tmp/q6_runB.log`:

```
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-08 05:09:05,316 - SL - DEBUG - 2547 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
[RUN B] CUSTOM_ALIAS_SECRET='secretcustom_alias' (=FLASK_SECRET+"custom_alias", app/config.py:201)
[RUN B] signer=itsdangerous.TimestampSigner (app/alias_suffix.py:11); check_suffix_signature max_age=600 (app/alias_suffix.py:40)
[RUN B] signed_suffix='.investigation-q6@example.com.ak3bcQ.fhOAjYecEx8GImx0RDpJ9Zt3pZM'
[RUN B] t=   0.000s  check_suffix_signature -> '.investigation-q6@example.com' VALID (returns suffix)
[RUN B] t= 595.100s  check_suffix_signature -> '.investigation-q6@example.com' VALID (returns suffix)
[RUN B] t= 599.004s  check_suffix_signature -> '.investigation-q6@example.com' VALID (returns suffix)
[RUN B] t= 601.002s  check_suffix_signature -> None                            EXPIRED (returns None)
[RUN B] t= 603.002s  check_suffix_signature -> None                            EXPIRED (returns None)
[RUN B] DONE
```

**(d) Concrete observed values.**
- **Immediate:** valid (`Run A` `t=0.000s`, `Run B` `t=0.000s` → returns the suffix).
- **Post‑expiry:** `None` (`Run A` `t=601.001s`+, `Run B` `t=601.002s`+).
- **Boundary (run scale/duration):** the two runs used different wall‑clock spans that both cross
  the 600s boundary — **Run A spanned ~610s** (last probe `t=610.005s`) and **Run B spanned ~603s**
  (last probe `t=603.002s`). **Last VALID at `t=600.001s` (Run A); first EXPIRED at `t=601.001s`
  (Run A) and `t=601.002s` (Run B).** The window is therefore **≈600 seconds**.
- **Stability (≥2 runs):** both runs agree — valid at ≤600s (Run A confirms `t=600.001s` still
  valid), expired at ≥601s.

**(e) `file:line` grounding.** `signer = itsdangerous.TimestampSigner(config.CUSTOM_ALIAS_SECRET)`
(`app/alias_suffix.py:11`); `CUSTOM_ALIAS_SECRET = FLASK_SECRET + "custom_alias"`
(`app/config.py:201`) — observed value `'secretcustom_alias'`. `check_suffix_signature` calls
`signer.unsign(signed_suffix, max_age=600).decode()` and catches `itsdangerous.BadSignature`,
returning `None` (`app/alias_suffix.py:37-42`; comment "user will click on the button in the
600 secs" at `:38`). Tokens are produced with `signer.sign(suffix).decode()`
(e.g. `app/alias_suffix.py:114`).

**(f) Cause→effect.** `itsdangerous 1.1.0`'s `TimestampSigner.unsign(value, max_age=600)`
computes `age = now - timestamp` (1‑second resolution) and raises `SignatureExpired` (a subclass
of `BadSignature`) when `age > 600`. Because the check uses strict `>`, a token whose age is
exactly 600s is still accepted (hence `t=600.001s` still valid — its integer age is 600), while
a token older than 600s (`t=601.001s`, integer age 601) is rejected and
`check_suffix_signature` returns `None`. The expiry is thus a property of the library's timed
signer with `max_age=600`, not any SimpleLogin‑specific timer.

**(g) OBSERVED vs INFERRED.** The 600s boundary is **OBSERVED** — established by real elapsed‑time
probing of the real signer across two runs, not inferred from reading the constant.

---

## Q7 — API key usage statistics

**(a) Direct answer.** On each **API‑key‑authenticated** request, two `api_key` columns update:
`times` is incremented by 1 and `last_used` is set to the current timestamp. After 5 calls with
the dedicated key (id=135), `times` went from `0` → `5` and `last_used` went from `NULL` →
`2026-07-08 04:24:29.551406`. A **session‑fallback** call (Q1) does **not** touch these — `times`
stayed `5`.

**(b) Exact commands.** Read the columns with `psql` before and after issuing 5 key‑authenticated
`GET /api/user_info` calls, then after one session‑fallback call.

```bash
DB="postgresql://test:test@localhost:5432/test"
SQL="SELECT id, name, times, last_used, sudo_mode_at FROM api_key WHERE id = 135;"
Q7KEY="<Q7_API_KEY_REDACTED>"   # 60-char code of ApiKey id=135 (redacted; throwaway container credential)

psql "$DB" -c "$SQL"                                              # BEFORE
for i in 1 2 3 4 5; do
  curl -s -o /dev/null -w "  call #$i -> HTTP %{http_code}\n" \
       -H "Authentication: $Q7KEY" http://localhost:7777/api/user_info
done
psql "$DB" -c "$SQL"                                              # AFTER 5 key calls
curl -s -o /dev/null -w "  session-fallback -> HTTP %{http_code}\n" \
     -b /tmp/q1_cookies.txt http://localhost:7777/api/user_info   # session fallback (no key)
psql "$DB" -c "$SQL"                                              # AFTER session fallback
```

**(c) Complete unedited output.**

```
===== Q7 BEFORE any key-authenticated call =====
$ psql "postgresql://test:test@localhost:5432/test" -c "SELECT id, name, times, last_used, sudo_mode_at FROM api_key WHERE id = 135;"
 id  |       name       | times | last_used | sudo_mode_at
-----+------------------+-------+-----------+--------------
 135 | q7-investigation |     0 |           |
(1 row)


===== Make N=5 key-authenticated calls: GET /api/user_info -H "Authentication: <Q7KEY>" =====
  call #1 -> HTTP 200
  call #2 -> HTTP 200
  call #3 -> HTTP 200
  call #4 -> HTTP 200
  call #5 -> HTTP 200

===== Q7 AFTER 5 key-authenticated calls =====
$ psql "postgresql://test:test@localhost:5432/test" -c "SELECT id, name, times, last_used, sudo_mode_at FROM api_key WHERE id = 135;"
 id  |       name       | times |         last_used          | sudo_mode_at
-----+------------------+-------+----------------------------+--------------
 135 | q7-investigation |     5 | 2026-07-08 04:24:29.551406 |
(1 row)


===== 1 SESSION-FALLBACK call (john cookie, NO Authentication) — must NOT touch key times =====
  session-fallback call -> HTTP 200

===== Q7 key stats AFTER the session-fallback call (expect times UNCHANGED = 5) =====
$ psql "postgresql://test:test@localhost:5432/test" -c "SELECT id, name, times, last_used, sudo_mode_at FROM api_key WHERE id = 135;"
 id  |       name       | times |         last_used          | sudo_mode_at
-----+------------------+-------+----------------------------+--------------
 135 | q7-investigation |     5 | 2026-07-08 04:24:29.551406 |
(1 row)
```

**(d) Concrete observed values.**
- **`times`:** before `0` → after 5 key calls `5` (delta `+5`, i.e. `+1` per key‑authenticated
  request) → after a session‑fallback call still `5` (unchanged).
- **`last_used`:** before `NULL` → after `2026-07-08 04:24:29.551406` (timestamp of the latest,
  5th, call).
- **`sudo_mode_at`:** `NULL` throughout (not touched by normal API auth).

**(e) `file:line` grounding.** When a real key is presented, `authorize_request()` executes the
`else` branch: `api_key.last_used = arrow.now()` (`app/api/base.py:30`), `api_key.times += 1`
(`app/api/base.py:31`), `Session.commit()` (`app/api/base.py:32`). These run **only** for real
keys, not under the session fallback (`app/api/base.py:20-25`). The columns are declared on the
`ApiKey` model (class at `app/models.py:2350`): `last_used = sa.Column(ArrowType, default=None)`
(`app/models.py:2358`), `times = sa.Column(sa.Integer, default=0, nullable=False)`
(`app/models.py:2359`), `sudo_mode_at = sa.Column(ArrowType, default=None)`
(`app/models.py:2360`).

**(f) Cause→effect.** Each of the 5 key‑authenticated requests enters the `else` branch, bumping
`times` and stamping `last_used`, then commits — so after 5 requests `times = 5` and `last_used`
equals the last call's `arrow.now()`. The session‑fallback request takes the `if not api_key`
branch instead, which never references any `api_key`, so no counters change — confirming the
statistics are **API‑key‑only**.

**(g) OBSERVED vs INFERRED.** All values are **OBSERVED** by reading the PostgreSQL `api_key` row
directly with `psql` before and after the calls.

---


## Q8 — Failed login (wrong credentials)

**(a) Direct answer.** A login with wrong credentials returns **HTTP 200** (a re‑render of the
login page, **not** 401), with the flash message **"Email or password incorrect"** embedded in
the HTML body. The only log output is the framework access line
(`… POST /auth/login … 200 …`); there is **no dedicated application log line** for the failed
branch — the failure is recorded only as a New‑Relic custom event. After 10 failures within a
minute, further attempts return **HTTP 429**.

**(b) Exact commands.** Part 1 (200 + flash + logs) on the canonical gunicorn (`:7777`, rate
limit off). Part 2 (429 edge) on a second gunicorn (`:7778`) started with `DISABLE_RATE_LIMIT`
**unset** so rate limiting is **on**.

```bash
# PART 1 — canonical gunicorn :7777 (rate limit OFF)
BASE=http://localhost:7777 ; JAR=/tmp/q8_cookies.txt ; rm -f "$JAR"
mark=$(wc -l < /tmp/gunicorn_7777.log)   # record log length BEFORE the login-page GET + failed POST
CSRF=$(curl -s -c "$JAR" "$BASE/auth/login" | grep -oP 'name="csrf_token"[^>]*value="\K[^"]+' | head -1)
curl -s -b "$JAR" -c "$JAR" \
  --data-urlencode "email=john@wick.com" --data-urlencode "password=wrongpassword" \
  --data-urlencode "csrf_token=$CSRF" "$BASE/auth/login" -o /tmp/q8_response.html -D /tmp/q8_response_headers.txt
cat /tmp/q8_response_headers.txt                  # status line + response headers
grep -n "Email or password incorrect" /tmp/q8_response.html
tail -n +$((mark+1)) /tmp/gunicorn_7777.log      # the emitted server logs during the failed login

# PART 2 — 429 edge: second gunicorn with rate limiting ON, then run the repeated-login loop
docker exec -d sl bash -c 'cd /app && . venv/bin/activate && eval "$(grep "^export " /build.sh)" \
   && unset DISABLE_RATE_LIMIT && exec gunicorn wsgi:app -b 0.0.0.0:7778 -w 2 --timeout 15 > /tmp/gunicorn_7778.log 2>&1'
bash /tmp/q8_429_loop.sh                          # the exact loop is shown in full below
```

The complete Part-2 loop script (`/tmp/q8_429_loop.sh`), run verbatim above:

```bash
#!/usr/bin/env bash
# Q8 429 edge: drive repeated FAILED logins (wrong password) against the
# rate-limited gunicorn on :7778 (started with DISABLE_RATE_LIMIT unset), record
# the HTTP status per attempt, and capture the FIRST 429 response IN FULL
# (status line + headers + body head). The failed-login branch sets g.deduct_limit,
# so each failure deducts from the "10/minute" bucket (app/auth/views/login.py:22-24).
set -u
BASE=http://localhost:7778
JAR=/tmp/q8_429_jar.txt ; rm -f "$JAR"
RESP_DIR=/tmp/q8_429_resp ; rm -rf "$RESP_DIR" ; mkdir -p "$RESP_DIR"

CSRF=$(curl -s -c "$JAR" "$BASE/auth/login" \
        | grep -oP 'name="csrf_token"[^>]*value="\K[^"]+' | head -1)
echo "csrf_token: $CSRF"
echo 'Rate limit rule: @limiter.limit("10/minute", deduct_when=...g.deduct_limit) [app/auth/views/login.py:22-24]'
echo ""
echo "===== Repeated failed logins (wrong password) — observe status per attempt ====="
first429=""
for i in $(seq 1 15); do
  curl -s -i -b "$JAR" -c "$JAR" \
    --data-urlencode "email=john@wick.com" \
    --data-urlencode "password=wrongpassword" \
    --data-urlencode "csrf_token=$CSRF" \
    "$BASE/auth/login" > "$RESP_DIR/resp_$i.txt"
  code=$(awk 'NR==1{print $2}' "$RESP_DIR/resp_$i.txt")
  printf "  attempt #%s -> HTTP %s\n" "$i" "$code"
  if [ "$code" = "429" ] && [ -z "$first429" ]; then first429="$i"; fi
done
echo ""
echo "===== Full response of the FIRST 429 (status line + headers + body head) ====="
FILE="$RESP_DIR/resp_$first429.txt"
# 1) status line + headers (everything up to and including the blank CRLF line)
sed -n '1,/^\r\{0,1\}$/p' "$FILE"
# 2) body head (first 20 lines after the header/body separator)
echo "---- body head (first 20 lines of the 429 HTML) ----"
awk 'b{print} /^\r?$/{b=1}' "$FILE" | head -20
```

**(c) Complete unedited output.**

```
=== Q8 PART 1: failed login (wrong creds) on canonical gunicorn :7777 (rate limit OFF) ===
$ mark=$(wc -l < /tmp/gunicorn_7777.log)   # record log length BEFORE the login-page GET + failed POST
mark=58
--- response status line + headers (failed POST) ---
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 05:24:12 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 7275
Set-Cookie: slapp=98a16904-c9d3-4de7-823d-1728f407995a.wIkEU24moEXpUQTCzDlKuJzqM4A; Expires=Wed, 15-Jul-2026 05:24:12 GMT; HttpOnly; Path=/; SameSite=Lax

--- flash message location in the rendered HTML (grep -n) ---
93:            <script>toastr.error("Email or password incorrect");</script>

=== gunicorn :7777 NEW log lines during the failed login (tail -n +$((mark+1))) ===
2026-07-08 05:24:12,398 - SL - DEBUG - 2595 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.0012831687927246094
2026-07-08 05:24:12,651 - SL - DEBUG - 2595 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 200, takes 0.24167585372924805
```

```
csrf_token: IjZiMWU3ZTZjY2QxYmZhZjVmODcyZTkxMmI4Yjg5ZTJlMzdmY2Y4ZjQi.ak3eMA.UZng9Jolyc_PTv3VMTIvIUrt0nM
Rate limit rule: @limiter.limit("10/minute", deduct_when=...g.deduct_limit) [app/auth/views/login.py:22-24]

===== Repeated failed logins (wrong password) — observe status per attempt =====
  attempt #1 -> HTTP 200
  attempt #2 -> HTTP 200
  attempt #3 -> HTTP 200
  attempt #4 -> HTTP 200
  attempt #5 -> HTTP 200
  attempt #6 -> HTTP 200
  attempt #7 -> HTTP 200
  attempt #8 -> HTTP 200
  attempt #9 -> HTTP 200
  attempt #10 -> HTTP 200
  attempt #11 -> HTTP 429
  attempt #12 -> HTTP 429
  attempt #13 -> HTTP 429
  attempt #14 -> HTTP 429
  attempt #15 -> HTTP 429

===== Full response of the FIRST 429 (status line + headers + body head) =====
HTTP/1.1 429 TOO MANY REQUESTS
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 05:20:51 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 5617
Set-Cookie: slapp=86cbcabb-fcca-403e-b573-47c0859a33f0.LJ5Gok-68iiaS2tOy0TqYWPetPM; Expires=Wed, 15-Jul-2026 05:20:51 GMT; HttpOnly; Path=/; SameSite=Lax

---- body head (first 20 lines of the 429 HTML) ----

<!DOCTYPE html>
<html lang="en"
      dir="ltr"
      data-theme="">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport"
          content="width=device-width, user-scalable=no, initial-scale=1.0, maximum-scale=1.0, minimum-scale=1.0" />
    <meta http-equiv="X-UA-Compatible" content="ie=edge" />
    <meta http-equiv="Content-Language" content="en" />
    <meta name="msapplication-TileColor" content="#2d89ef" />
    <meta name="theme-color" content="#4188c9" />
    <meta name="apple-mobile-web-app-status-bar-style"
          content="black-translucent" />
    <meta name="apple-mobile-web-app-capable" content="yes" />
    <meta name="mobile-web-app-capable" content="yes" />
    <meta name="HandheldFriendly" content="True" />
    <meta name="MobileOptimized" content="320" />
    <meta name="referrer" content="no-referrer" />
```

**(d) Concrete observed values.**
- **HTTP response:** `HTTP/1.1 200 OK` (not 401); body length 7275; flash rendered as
  `<script>toastr.error("Email or password incorrect");</script>` (line 93 of the HTML).
- **Emitted log output:** only the `server.py:284` access lines
  (`… GET /auth/login … 200 …` and `… POST /auth/login ImmutableMultiDict([]) 200, takes 0.241…`).
  **No dedicated application log line** exists for the failed‑credential branch.
- **429 edge:** attempts `#1–#10` → `HTTP 200`; attempt `#11` onward → `HTTP 429 TOO MANY REQUESTS`.

**(e) `file:line` grounding.** The bad‑credential branch is
`if not user or not user.check_password(form.password.data):` (`app/auth/views/login.py:45`),
setting `g.deduct_limit = True` (`:47`), `form.password.data = None` (`:48`),
`flash("Email or password incorrect", "error")` (`:49`), and
`LoginEvent(LoginEvent.ActionType.failed).send()` (`:50`); control then falls through to
`render_template("auth/login.html", …)` (`:74`), which is why the status is **200**.
`LoginEvent.send()` is `newrelic.agent.record_custom_event("LoginEvent", {...})` **only**
(`app/events/auth_event.py:22-25`) — no app log line. The rate limit is
`@limiter.limit("10/minute", deduct_when=lambda r: hasattr(g, "deduct_limit") and g.deduct_limit)`
(`app/auth/views/login.py:22-24`); rate limiting is gated by
`config.DISABLE_RATE_LIMIT = "DISABLE_RATE_LIMIT" in os.environ` (`app/config.py:602`) via the
`disable_rate_limit` request filter (`app/extensions.py:26-28`). The access log line is emitted
by `after_request` (`server.py:284`).

**(f) Cause→effect.** A wrong password matches the `if not user or not user.check_password(...)`
branch, which flashes the error and sets `g.deduct_limit`, but does **not** return an error
status — execution continues to `render_template(...)`, so Flask returns **200** with the
re‑rendered login page (the flash appears as a `toastr.error(...)` script). No `LOG.*` call
exists in that branch (only a New‑Relic custom event), so no dedicated app log line is produced
— just the generic access line. The `deduct_when` lambda deducts from the `10/minute` bucket
**only** on failed attempts; after 10 failures the bucket is exhausted and the limiter
short‑circuits subsequent requests with **429** before the view runs.

**(g) OBSERVED vs INFERRED.** All values are **OBSERVED**: the 200 status + flash HTML, the exact
log lines (and the absence of a failed‑branch app log line), and the 429 threshold. Part 2 used a
dedicated gunicorn on `:7778` started with `DISABLE_RATE_LIMIT` unset (confirmed
`config.DISABLE_RATE_LIMIT = False`), stated here as the non‑default toggle used to exercise the
rate‑limit edge (the canonical `example.env` likewise omits `DISABLE_RATE_LIMIT`, so rate
limiting is on there too).

---

## Coverage Pass

Re‑reading each question and confirming every named sub‑item is answered with real evidence:

- **Q1** — HTTP status code ✔ (`200` fallback / `401` error) **AND** JSON body ✔
  (`{"can_create_reverse_alias":…,"name":"John Wick",…}` / `{"error":"Wrong api key"}`). Both
  the 200 session‑fallback path and the 401 no‑auth error path exercised.
- **Q2** — status code ✔ **AND** error message ✔ for **both** conditions: browser‑session‑only →
  `500 {"error":"Internal error"}` (with the `AttributeError: 'NoneType' … 'sudo_mode_at'`
  traceback), and real‑API‑key‑without‑sudo → `440 {"error":"Need sudo"}`.
- **Q3** — raw bytes ✔ (300‑byte `b'\x80\x04…'`) **AND** format = pickle ✔ (`\x80\x04` proto 4)
  **AND** deserialized keys ✔ (`_permanent,_fresh,csrf_token,_user_id,_id,sudo_time`) **AND**
  `session:<uuid4>` key structure ✔ (`session:b5e4833a-…`, cookie `<uuid4>.<hmac>`).
- **Q4** — before id ✔ (`1b76a2a2-…`) **AND** after id ✔ (`1b76a2a2-…`, identical) **AND**
  same‑or‑rotated conclusion ✔ (**UNCHANGED / NOT ROTATED**).
- **Q5** — custom `X-*` ✔ (**stripped**) **AND** `Received` ✔ (**stripped**) **AND** `Reply-To` ✔
  (**stripped then replaced** with a reverse‑alias), each with observed before/after and the
  verbatim forwarded header set (plus the `From` rewrite).
- **Q6** — immediate‑valid ✔ **AND** post‑expiry‑`None` ✔ **AND** ~600s boundary ✔ (last valid
  `t=600.001s`, first expired `t=601.001s`) **AND** ≥2‑run stability ✔ (Run A + Run B agree).
- **Q7** — `last_used` ✔ (`NULL` → `2026-07-08 04:24:29.551406`) **AND** `times` ✔ (`0` → `5`),
  **both** with actual before **and** after values (plus session‑fallback non‑increment), via
  raw `psql` output.
- **Q8** — emitted log output ✔ (only the `server.py:284` access lines; **no** failed‑branch app
  log line; New‑Relic‑only event) **AND** HTTP response ✔ (`200` + flash "Email or password
  incorrect") **AND** the `429` rate‑limit edge ✔.

---

## Read‑only Guarantee

This was a read‑only investigation. No existing SimpleLogin source file was modified; all
temporary observation scripts and crafted `.eml` fixtures were created inside the container's
`/tmp` (outside any tracked tree) and removed afterward. The only committed artifact is this
document. Verified on the host repository **after committing**: the working tree is clean
(`git status --porcelain` prints nothing), and diffing against the pre‑investigation baseline
commit `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c` shows exactly one added file and no source
changes:

```
$ git status --porcelain

$ git diff 2cd6ee777f8c2d3531559588bcfb18627ffb5d2c --name-status
A	blitzy/documentation/app_2cd6ee777f8c.md
```

The empty `git status --porcelain` confirms every change is committed with nothing left
untracked or modified, and the baseline `--name-status` diff confirms the sole change
introduced across the entire investigation is this one new document (`A` = added) — no tracked
source file appears as added, modified, or deleted.

