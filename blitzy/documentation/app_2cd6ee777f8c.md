# SimpleLogin Behavioral Investigation — app_2cd6ee777f8c

This document presents the results of a live behavioral investigation of the SimpleLogin application. Every conclusion below is grounded in runtime evidence (verbatim HTTP request/response captures, Redis dumps, PostgreSQL reads, and Flask server log lines) — not static code analysis alone.

## Methodology

| Component | Value |
|-----------|-------|
| Flask dev server | `http://localhost:7777` (Werkzeug 1.0.1 / Python 3.10.18) |
| Session cookie name | `slapp` (per `app/config.py` `SESSION_COOKIE_NAME = "slapp"`) |
| Database | PostgreSQL 15 at `postgresql://test:test@localhost:15432/test` |
| Session / cache store | Redis 7.0 at `redis://localhost:6379` (`MEM_STORE_URI`) |
| Configuration file | `tests/test.env` (via `CONFIG=tests/test.env`) |
| `FLASK_SECRET` | `secret` |
| `CUSTOM_ALIAS_SECRET` | `"secretcustom_alias"` (FLASK_SECRET + "custom_alias", per `app/config.py`) |
| Test user | `id=408`, `email=test@test.com`, password `password123`, `alternative_id=c706596b-f97b-4e2d-8c82-3fa402f1b8c8`, `activated=True` |
| Test API key | `id=46`, `user_id=408`, `code=hbijkhwsqyakgofduorvidjfhmmpffaqyxaagvycvvwruelntbhxfqtyszuj` |

**Investigation protocol:** The Flask application was started with the test configuration, a test user and API key were created directly in the database (via the ORM), then each of the eight investigation questions was answered by exercising the live system with `curl`, interrogating Redis with the Python `redis` library, querying PostgreSQL through the ORM, and capturing server log output. **No existing source files in the repository were modified.** All temporary scripts and helper artifacts produced during the investigation were deleted after the evidence was captured.

---

## Section 1 — API Authentication with Browser Session

### Method

An authenticated browser session was established by logging in via `POST /auth/login` (CSRF token extracted from the login page). Then `GET /api/user_info` was issued carrying **only** the `slapp` session cookie — **no** `Authentication` header was sent.

### Evidence

**Login handshake (cookie jar carried through):**

```text
# 1) GET /auth/login to obtain cookie + CSRF
> GET /auth/login HTTP/1.1
< HTTP/1.0 200 OK
< Set-Cookie: slapp=d9884851-3f0d-4b71-9e3d-02971caee288.TcXBaK06iYKbkyeqxGLMFdOUa0s; ...

# 2) POST /auth/login with correct credentials
HTTP 302  (redirect to /dashboard/)
```

**API request using only the session cookie (no `Authentication` header):**

```bash
curl -v -b cookies.txt http://localhost:7777/api/user_info
```

```http
> GET /api/user_info HTTP/1.1
> Host: localhost:7777
> User-Agent: curl/7.88.1
> Accept: */*
> Cookie: slapp=d9884851-3f0d-4b71-9e3d-02971caee288.TcXBaK06iYKbkyeqxGLMFdOUa0s
>
< HTTP/1.0 200 OK
< Content-Type: application/json
< Content-Length: 234
< Access-Control-Allow-Origin: *
< Set-Cookie: slapp=d9884851-3f0d-4b71-9e3d-02971caee288.TcXBaK06iYKbkyeqxGLMFdOUa0s; Expires=Fri, 24-Apr-2026 02:20:42 GMT; HttpOnly; Path=/; SameSite=Lax
< Server: Werkzeug/1.0.1 Python/3.10.18
< Date: Fri, 17 Apr 2026 02:20:42 GMT
```

**Response body:**

```json
{
  "can_create_reverse_alias": true,
  "connected_proton_address": null,
  "email": "test@test.com",
  "in_trial": true,
  "is_premium": true,
  "max_alias_free_plan": 3,
  "name": "Test User",
  "profile_picture_url": null
}
```

### Observed Result

**HTTP 200 OK** with the complete JSON user-info payload. The request carried only `Cookie: slapp=<uuid>.<signature>` and no `Authentication` header, yet the API accepted the request and returned the full user record.

### Conclusion

This behavior comes from `app/api/base.py` `authorize_request()` (lines 16–43). When `request.headers.get("Authentication")` returns `None`, `ApiKey.get_by(code=None)` yields `None`, so the `if not api_key:` branch runs. It checks `current_user.is_authenticated`, which is `True` because the `slapp` cookie was accepted by `RedisSessionStore.open_session()` (`app/session.py` lines 68–80) and the Flask-Login `user_loader` (`server.py` line 220) successfully loaded the user from `_user_id`. `authorize_request()` then sets `g.user = current_user` and proceeds (with `g.api_key = None` set at line 42). The browser session is therefore sufficient to authenticate any API endpoint that uses `@require_api_auth`.

```python
# app/api/base.py lines 16–43 (excerpt)
def authorize_request() -> Optional[Tuple[str, int]]:
    api_code = request.headers.get("Authentication")
    api_key = ApiKey.get_by(code=api_code)

    if not api_key:
        if current_user.is_authenticated:
            g.user = current_user
        else:
            return jsonify(error="Wrong api key"), 401
    else:
        # Update api key stats
        api_key.last_used = arrow.now()
        api_key.times += 1
        Session.commit()
        g.user = api_key.user
    ...
    g.api_key = api_key
    return None
```

---

## Section 2 — Privileged Operation Access via Session Cookie

### Method

Using the same cookie jar from Section 1 (authenticated as `test@test.com`), `DELETE /api/user` was issued. This endpoint is decorated with `@require_api_sudo` (see `app/api/views/user.py` lines 12–14). The Flask server log was cleared immediately before the request so the resulting log lines could be captured in isolation.

### Evidence

**Request:**

```bash
curl -v -b cookies.txt -X DELETE http://localhost:7777/api/user
```

```http
> DELETE /api/user HTTP/1.1
> Host: localhost:7777
> User-Agent: curl/7.88.1
> Accept: */*
> Cookie: slapp=d9884851-3f0d-4b71-9e3d-02971caee288.TcXBaK06iYKbkyeqxGLMFdOUa0s
>
< HTTP/1.0 500 INTERNAL SERVER ERROR
< Content-Type: application/json
< Content-Length: 32
< Access-Control-Allow-Origin: *
< Set-Cookie: slapp=d9884851-3f0d-4b71-9e3d-02971caee288.TcXBaK06iYKbkyeqxGLMFdOUa0s; Expires=Fri, 24-Apr-2026 02:20:51 GMT; HttpOnly; Path=/; SameSite=Lax
< Server: Werkzeug/1.0.1 Python/3.10.18
< Date: Fri, 17 Apr 2026 02:20:51 GMT
```

**Response body:**

```json
{
  "error": "Internal error"
}
```

**Flask server log (verbatim):**

```text
2026-04-17 02:20:51,006 - SL - ERROR - 23740 - "/workspace/server.py:390" - error_handler() -  - 'NoneType' object has no attribute 'sudo_mode_at'
Traceback (most recent call last):
  File "/app/venv/lib/python3.10/site-packages/flask/app.py", line 1950, in full_dispatch_request
    rv = self.dispatch_request()
  File "/app/venv/lib/python3.10/site-packages/flask_debugtoolbar/__init__.py", line 125, in dispatch_request
    return view_func(**req.view_args)
  File "/usr/local/lib/python3.10/cProfile.py", line 110, in runcall
    return func(*args, **kw)
  File "/workspace/app/api/base.py", line 69, in decorated
    if not check_sudo_mode_is_active(g.api_key):
  File "/workspace/app/api/base.py", line 47, in check_sudo_mode_is_active
    return api_key.sudo_mode_at and g.api_key.sudo_mode_at >= arrow.now().shift(
AttributeError: 'NoneType' object has no attribute 'sudo_mode_at'
2026-04-17 02:20:51,006 - SL - DEBUG - 23740 - "/workspace/server.py:284" - after_request() -  - 127.0.0.1 DELETE /api/user ImmutableMultiDict([]) 500, takes 0.006419181823730469
```

### Observed Result

- **HTTP status:** `500 INTERNAL SERVER ERROR`
- **Response body:** `{"error": "Internal error"}`
- **Server-side exception:** `AttributeError: 'NoneType' object has no attribute 'sudo_mode_at'` at `app/api/base.py` line 47
- The response is **not** a controlled sudo-required 440 response — the handler crashes *before* it can return "Need sudo".

### Conclusion

When the request is authenticated purely by the browser session (no API key), `authorize_request()` sets `g.api_key = None` (`app/api/base.py` line 42, executed when `api_key` is `None`). The `require_api_sudo` decorator then calls `check_sudo_mode_is_active(g.api_key)` (line 69), which dereferences `api_key.sudo_mode_at` on a `None` value (line 47), raising `AttributeError`. The generic `@app.errorhandler(Exception)` in `server.py` lines 388–394 catches the exception, logs it via `LOG.e(e)`, checks `request.path.startswith("/api/")`, and returns `jsonify(error="Internal error"), 500`. The intended `jsonify(error="Need sudo"), 440` on line 70 of `app/api/base.py` is never reached because the `check_sudo_mode_is_active` call itself throws before returning. Therefore, any sudo-protected endpoint crashes with HTTP 500 when accessed via a pure browser-session authentication path.

```python
# app/api/base.py lines 46–49
def check_sudo_mode_is_active(api_key: ApiKey) -> bool:
    return api_key.sudo_mode_at and g.api_key.sudo_mode_at >= arrow.now().shift(
        minutes=-SUDO_MODE_MINUTES_VALID
    )

# app/api/base.py lines 63–73
def require_api_sudo(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        error_return = authorize_request()
        if error_return:
            return error_return
        if not check_sudo_mode_is_active(g.api_key):
            return jsonify(error="Need sudo"), 440
        return f(*args, **kwargs)
    return decorated
```

```python
# server.py lines 388–394
@app.errorhandler(Exception)
def error_handler(e):
    LOG.e(e)
    if request.path.startswith("/api/"):
        return jsonify(error="Internal error"), 500
    else:
        return render_template("error/500.html"), 500
```

---

## Section 3 — Session Storage Inspection

### Method

Two session states were inspected against Redis:

1. A **pre-login** session, created by a bare `GET /auth/login` (no credentials submitted).
2. A **post-login** session, created by completing the login flow for `test@test.com`.

For each, the raw Redis bytes, TTL, pickle protocol header, and the deserialized dictionary were recorded.

### Evidence

**(a) Pre-login session — raw Redis value and deserialized form**

Cookie observed after `GET /auth/login`:

```text
#HttpOnly_localhost  FALSE  /  FALSE  1776997266  slapp  46f58eb4-3d47-443a-b1ac-27f288799888.x-wwbBNkfMQdVaaqlCrFNGVeZbw
```

Redis inspection:

```python
import redis, pickle
r = redis.Redis.from_url('redis://localhost:6379', decode_responses=False)
key = 'session:46f58eb4-3d47-443a-b1ac-27f288799888'
raw = r.get(key)
```

```text
PRE-LOGIN SESSION
KEY: session:46f58eb4-3d47-443a-b1ac-27f288799888
TTL: 295
RAW_LEN: 96
RAW_HEX: 80049555000000000000007d94288c0a5f7065726d616e656e7494888c065f667265736894898c0a637372665f746f6b656e948c286364633866313034633037626437336338333037373630323063333263643533616536333666383494752e
RAW_REPR: b'\x80\x04\x95U\x00\x00\x00\x00\x00\x00\x00}\x94(\x8c\n_permanent\x94\x88\x8c\x06_fresh\x94\x89\x8c\ncsrf_token\x94\x8c(cdc8f104c07bd73c830776020c32cd53ae636f84\x94u.'
DESERIALIZED: {'_permanent': True, '_fresh': False, 'csrf_token': 'cdc8f104c07bd73c830776020c32cd53ae636f84'}
KEYS_SORTED: ['_fresh', '_permanent', 'csrf_token']
```

**(b) Post-login session — raw Redis value and deserialized form**

Cookie observed after completing login:

```text
#HttpOnly_localhost  FALSE  /  FALSE  1776997238  slapp  d9884851-3f0d-4b71-9e3d-02971caee288.TcXBaK06iYKbkyeqxGLMFdOUa0s
```

Redis inspection:

```text
POST-LOGIN SESSION
KEY: session:d9884851-3f0d-4b71-9e3d-02971caee288
TTL: 604788          # ≈ 7.0000 days
RAW_LEN: 300
RAW_HEX_FIRST_4: 80049521
RAW_HEX_FIRST_20: 80049521010000000000007d94288c0a5f706572
RAW_REPR_FIRST_60: b'\x80\x04\x95!\x01\x00\x00\x00\x00\x00\x00}\x94(\x8c\n_permanent\x94\x88\x8c\x06_fresh\x94\x88\x8c\ncsrf_token\x94\x8c(cc41f56'
DESERIALIZED:
{'_fresh': True,
 '_id': 'b03643a8a515eae10966eb5799d0b928b1fae8daeb0b0a33ba33f35b5fd7e09d574a2b38cd3af3ce53bdb464d28d2c515533674c8b087cba7d49abfa2cc9de57',
 '_permanent': True,
 '_user_id': 'c706596b-f97b-4e2d-8c82-3fa402f1b8c8',
 'csrf_token': 'cc41f5699e4990d72eeb8440efdb50190531f837',
 'sudo_time': 1776392438}
KEYS_SORTED: ['_fresh', '_id', '_permanent', '_user_id', 'csrf_token', 'sudo_time']
```

### Observed Result

| Property | Pre-login | Post-login |
|----------|-----------|------------|
| Redis key | `session:46f58eb4-3d47-443a-b1ac-27f288799888` | `session:d9884851-3f0d-4b71-9e3d-02971caee288` |
| TTL (seconds) | `295` (≈ 5 min) | `604788` (≈ 7 days) |
| Byte length | 96 | 300 |
| First 4 bytes (hex) | `80 04 95 55` | `80 04 95 21` |
| Dict keys | `_fresh`, `_permanent`, `csrf_token` | `_fresh`, `_id`, `_permanent`, `_user_id`, `csrf_token`, `sudo_time` |

- **Serialization format:** Python `pickle`, **protocol 4**. The hex stream begins with `0x80 0x04`, where `0x80` is the pickle `PROTO` opcode and `0x04` is the protocol version (per the Python pickle spec). The next byte `0x95` is the `FRAME` opcode, followed by an 8-byte little-endian frame length (`0x55 00 00 00 00 00 00 00` = 85 bytes for the pre-login dict; `0x21 01 00 00 00 00 00 00` = 289 bytes for the post-login dict).
- **Redis key structure:** `session:<uuid4>` — a UUID4 generated in `app/session.py` line 71 (`str(uuid.uuid4())`).
- **Cookie structure:** `<uuid>.<hmac-signature>` — the UUID in plaintext, a dot separator, and a URL-safe base64-encoded HMAC-SHA1 signature.
- **Authenticated session keys:** `_permanent`, `_fresh`, `csrf_token`, `_user_id` (the user's `alternative_id` UUID), `_id` (the strong-session identifier), and `sudo_time` (unix epoch of the login moment).

### Conclusion

These findings map directly onto `app/session.py`:

- **Redis key prefix** is constant `SESSION_PREFIX = "session"` (line 18). `_get_key()` returns `f"{SESSION_PREFIX}:{session_Id}"` (lines 43–45), producing the observed `session:<uuid>` format.
- **Pickle serialization** happens in `save_session()` at line 91: `val = pickle.dumps(dict(session))`. Python's default protocol for `pickle.dumps` on Python 3.8+ is protocol 4, which matches the observed `80 04` header bytes. Data is decoded in `open_session()` at line 76 via `pickle.loads(val)`.
- **TTL logic** (`save_session()` lines 92–96) explicitly sets `ttl = 300` when `"_user_id" not in session` (unauthenticated) — matching the observed `295s` TTL — and otherwise falls back to `int(app.permanent_session_lifetime.total_seconds())` (7 days — explicitly set by SimpleLogin at `server.py:207` via `app.permanent_session_lifetime = timedelta(days=7)`, overriding Flask's own 31-day default) — matching the observed `604788s` TTL.
- **Cookie signing** (lines 102–104) uses `itsdangerous.Signer(app.secret_key, salt="session", key_derivation="hmac")` to sign the session UUID. The resulting cookie value is `<uuid>.<signature>`.
- **Session keys:**
  - `_permanent` and `_fresh` are Flask-Login bookkeeping flags.
  - `csrf_token` is set by Flask-WTF on first use.
  - `_user_id` is Flask-Login's identifier, storing the user's `alternative_id` (per `User.get_id()` in `app/models.py`); the observed value `c706596b-f97b-4e2d-8c82-3fa402f1b8c8` matches the test user's `alternative_id`.
  - `_id` is Flask-Login's "strong session protection" token (SHA-512 of `remote_addr|user_agent` salted by the Flask secret). It is only present once `login_manager.session_protection = "strong"` takes effect, which is configured in `app/extensions.py`.
  - `sudo_time` is set by `after_login()` in `app/auth/views/login_utils.py` line 37: `session["sudo_time"] = int(time())`.

---

## Section 4 — Session ID Behavior During Login

### Method

A fresh cookie jar was created, a `GET /auth/login` was issued (no credentials) to capture the pre-login `slapp` cookie, then a `POST /auth/login` with valid credentials was performed through the same cookie jar (cookies carried forward). The `slapp` cookie before and after the login was compared byte-for-byte.

### Evidence

```text
=== PRE-LOGIN COOKIES (cookies_exp4.txt) ===
#HttpOnly_localhost  FALSE  /  FALSE  1776997280  slapp  6763308a-b37a-4c35-9e72-2e4c0523c3c0.--q3zQhSMiIrnUYHrcJJ5_fp5MA

=== PRE_COOKIE_FULL ===
6763308a-b37a-4c35-9e72-2e4c0523c3c0.--q3zQhSMiIrnUYHrcJJ5_fp5MA

=== PRE_UUID ===
6763308a-b37a-4c35-9e72-2e4c0523c3c0

=== POST LOGIN ===
HTTP 302

=== POST-LOGIN COOKIES (cookies_exp4.txt) ===
#HttpOnly_localhost  FALSE  /  FALSE  1776997281  slapp  6763308a-b37a-4c35-9e72-2e4c0523c3c0.--q3zQhSMiIrnUYHrcJJ5_fp5MA

=== POST_COOKIE_FULL ===
6763308a-b37a-4c35-9e72-2e4c0523c3c0.--q3zQhSMiIrnUYHrcJJ5_fp5MA

=== POST_UUID ===
6763308a-b37a-4c35-9e72-2e4c0523c3c0

=== COMPARISON ===
COOKIE VALUE UNCHANGED (UUID + SIGNATURE)
```

### Observed Result

| | Before `POST /auth/login` | After `POST /auth/login` |
|---|---|---|
| Cookie value | `6763308a-b37a-4c35-9e72-2e4c0523c3c0.--q3zQhSMiIrnUYHrcJJ5_fp5MA` | `6763308a-b37a-4c35-9e72-2e4c0523c3c0.--q3zQhSMiIrnUYHrcJJ5_fp5MA` |
| UUID portion | `6763308a-b37a-4c35-9e72-2e4c0523c3c0` | `6763308a-b37a-4c35-9e72-2e4c0523c3c0` |
| HMAC signature | `--q3zQhSMiIrnUYHrcJJ5_fp5MA` | `--q3zQhSMiIrnUYHrcJJ5_fp5MA` |

**The session ID (UUID) is UNCHANGED across the login transition.** The entire cookie value — UUID and signature — is byte-identical. The pre-login session record in Redis (keyed `session:6763308a-...`) is mutated in place with the post-login dictionary keys (`_user_id`, `_id`, `sudo_time`); no new Redis key is created.

### Conclusion

This is a **session-fixation-relevant** observation. Flask-Login's `login_user()`, called in `app/auth/views/login_utils.py` line 36 by `after_login()`, writes `_user_id` into the existing session dictionary. It does **not** allocate a new `session.session_id`. The `RedisSessionStore.purge_session()` method in `app/session.py` lines 61–66 is the only code path that regenerates the session UUID (via `session.session_id = str(uuid.uuid4())`), and it is invoked only from `logout_session()` at lines 117–121 — never during login. On the next `save_session()` call after login, the same UUID is re-signed (producing the same signature because the signing key and UUID are both unchanged), and `setex()` rewrites the Redis value under the same `session:<uuid>` key with a promoted 7-day TTL.

```python
# app/auth/views/login_utils.py lines 35–37
LOG.d("log user %s in", user)
login_user(user)
session["sudo_time"] = int(time())

# app/session.py lines 61–66 (called only on logout)
def purge_session(self, session: ServerSession):
    try:
        self._redis_w.delete(self._get_key(session.session_id))
        session.session_id = str(uuid.uuid4())
    except AttributeError:
        pass
```

---

## Section 5 — Email Header Forwarding Behavior

### Method

A `Message` object was constructed in-process with `X-Custom-Header`, `Received`, and `Reply-To` present alongside standard headers. The actual production-path utility `app.email_utils.delete_all_headers_except` was invoked with the **identical** `headers_to_keep` list used by `email_handler.py` `forward_email_to_mailbox()` (lines 793–807). The message headers were inspected before and after the call.

### Evidence

```python
from email.message import Message
from app.email_utils import delete_all_headers_except
from app.email import headers

headers_to_keep = [
    headers.FROM,
    headers.TO,
    headers.CC,
    headers.SUBJECT,
    headers.DATE,
    headers.MESSAGE_ID,
    headers.REFERENCES,
    headers.IN_REPLY_TO,
    headers.SL_QUEUE_ID,
    headers.LIST_UNSUBSCRIBE,
    headers.LIST_UNSUBSCRIBE_POST,
] + headers.MIME_HEADERS
```

```text
HEADERS_TO_KEEP = ['From', 'To', 'Cc', 'Subject', 'Date', 'Message-ID', 'References',
                   'In-Reply-To', 'X-SL-Queue-Id', 'List-Unsubscribe', 'List-Unsubscribe-Post',
                   'mime-version', 'content-type', 'content-disposition', 'content-transfer-encoding']

BEFORE delete_all_headers_except:
  From: sender@external.com
  To: alias@sl.local
  Subject: Header test
  Date: Mon, 1 Jan 2024 00:00:00 +0000
  Message-ID: <abc123@external.com>
  Content-Type: text/plain; charset=utf-8
  X-Custom-Header: custom-value-to-test
  Received: from example.com by mx.sl.local; Mon, 1 Jan 2024 00:00:00 +0000
  Reply-To: someone-reply-to@elsewhere.com

AFTER delete_all_headers_except:
  From: sender@external.com
  To: alias@sl.local
  Subject: Header test
  Date: Mon, 1 Jan 2024 00:00:00 +0000
  Message-ID: <abc123@external.com>
  Content-Type: text/plain; charset=utf-8

SURVIVAL CHECK:
X-Custom-Header survived: False
Received survived: False
Reply-To survived: False
From survived: True
Subject survived: True
Content-Type survived: True
```

### Observed Result

| Header | Present Before | Present After | Verdict |
|--------|----------------|---------------|---------|
| `From` | yes | yes | kept (on allow-list) |
| `To` | yes | yes | kept (on allow-list) |
| `Subject` | yes | yes | kept (on allow-list) |
| `Date` | yes | yes | kept (on allow-list) |
| `Message-ID` | yes | yes | kept (on allow-list) |
| `Content-Type` | yes | yes | kept (MIME header allow-listed) |
| `X-Custom-Header` | yes | **no** | **STRIPPED** |
| `Received` | yes | **no** | **STRIPPED** |
| `Reply-To` | yes | **no** | **STRIPPED** |

All three target headers — `X-Custom-Header`, `Received`, and `Reply-To` — are stripped by the forwarding pipeline. None of them appears in the `headers_to_keep` allow-list.

### Conclusion

In `email_handler.py` lines 793–810, `forward_email_to_mailbox()` builds an explicit allow-list and calls `delete_all_headers_except(msg, headers_to_keep)`. The allow-list contains the message-hygiene headers (`From`, `To`, `Cc`, `Subject`, `Date`, `Message-ID`, `References`, `In-Reply-To`, `List-Unsubscribe`, `List-Unsubscribe-Post`, `X-SL-Queue-Id`) plus `MIME_HEADERS` (`Mime-Version`, `Content-Type`, `Content-Disposition`, `Content-Transfer-Encoding`). The constants `headers.REPLY_TO = "Reply-To"` and `headers.RECEIVED = "Received"` (`app/email/headers.py` lines 13–14) are defined but deliberately NOT in the keep-list. Any header whose lowercase name is not in this allow-list is deleted by `delete_all_headers_except()` (`app/email_utils.py` lines 536–542), which iterates the message's `_headers` list in reverse and calls `del msg._headers[i]` for every unmatched entry:

```python
# app/email_utils.py lines 536–542
def delete_all_headers_except(msg: Message, headers: [str]):
    headers = [h.lower() for h in headers]
    for i in reversed(range(len(msg._headers))):
        header_name = msg._headers[i][0].lower()
        if header_name not in headers:
            del msg._headers[i]
```

```python
# email_handler.py lines 793–810 (the forward pipeline)
headers_to_keep = [
    headers.FROM, headers.TO, headers.CC, headers.SUBJECT, headers.DATE,
    headers.MESSAGE_ID, headers.REFERENCES, headers.IN_REPLY_TO,
    headers.SL_QUEUE_ID, headers.LIST_UNSUBSCRIBE, headers.LIST_UNSUBSCRIBE_POST,
] + headers.MIME_HEADERS
if user.include_header_email_header:
    headers_to_keep.append(headers.AUTHENTICATION_RESULTS)
delete_all_headers_except(msg, headers_to_keep)
```

**Note on `Reply-To`:** A full `handle_forward()` call (`email_handler.py` line 536) may *reconstruct* a `Reply-To` header downstream using the reverse-alias address when `reply_to_contact` exists. This reconstructed header is a newly-crafted value pointing to the reverse-alias; the **original sender's** `Reply-To` is still stripped first by `delete_all_headers_except()`. In the forwarded message, only the SimpleLogin-generated reverse-alias Reply-To (if any) is visible — never the inbound one.

---

## Section 6 — Alias Creation Token Expiration

### Method

The production-path signer `app.alias_suffix.signer` (an `itsdangerous.TimestampSigner` initialised with `config.CUSTOM_ALIAS_SECRET`) was exercised directly. A fresh token was signed, then `signer.unsign()` was called with various `max_age` values at different elapsed times to find the boundary. `check_suffix_signature()` (the function actually used by the web app) was also exercised to confirm the hard-coded `max_age=600`.

### Evidence

```text
CUSTOM_ALIAS_SECRET = 'secretcustom_alias'
Signer type: TimestampSigner

SIGNED TOKEN: .test-suffix@sl.local.aeGZOw.JayFjy80p5Go3tsYLHhp-zOjMss

--- Test 1: max_age=600 immediately ---
SUCCESS — result: .test-suffix@sl.local

--- Test 2: max_age=0 immediately (expect FAIL) ---
SignatureExpired: Signature age 1 > 0 seconds

--- Test 3: max_age=1 after 3 seconds (expect FAIL) ---
SignatureExpired: Signature age 4 > 1 seconds

--- Test 4: max_age=600 after ~4 seconds (expect SUCCESS) ---
SUCCESS — result: .test-suffix@sl.local

--- Test 5: check_suffix_signature() ---
check_suffix_signature result: .test-suffix@sl.local

--- Test 6: Tampered signature ---
BadTimeSignature: Signature b'JayFjy80p5Go3tsYLHhp-zXXXXX' does not match

--- Test 7: check_suffix_signature on tampered token ---
check_suffix_signature (tampered) result: None
```

**Direct boundary tests.** To find the exact boundary where a token "stops working" (per the investigation requirement), we constructed tokens with back-dated timestamps by temporarily monkey-patching `signer.get_timestamp` to return `now - N` for specific values of `N`, then verified the behaviour of both `signer.unsign(token, max_age=600)` and the production `check_suffix_signature(token)`:

```text
--- Boundary Test 8: simulate 601-second-old token ---
OLD (601s backdated) TOKEN: .test-suffix@sl.local.aeGvMA.PzaUo3YgM7n88lFhePqCEECNlsU
SignatureExpired: Signature age 601 > 600 seconds

--- Boundary Test 9: simulate 599-second-old token ---
YOUNG (599s backdated) TOKEN: .test-suffix@sl.local.aeGvMg.1lB0uwSWsIWbotJcNyQf8m0uBFQ
SUCCESS - result: .test-suffix@sl.local

--- Boundary Test 10: check_suffix_signature on 601-second-old token ---
check_suffix_signature (601s old) result: None  (should be None)

--- Boundary Test 11: check_suffix_signature on 599-second-old token ---
check_suffix_signature (599s old) result: '.test-suffix@sl.local'
```

### Observed Result

| Test | `max_age` | Elapsed time | Outcome |
|------|-----------|--------------|---------|
| 1 | `600` | ~0 s | ✅ VALID — returns `.test-suffix@sl.local` |
| 2 | `0` | 1 s after signing | ❌ `itsdangerous.SignatureExpired: "Signature age 1 > 0 seconds"` |
| 3 | `1` | 4 s after signing | ❌ `itsdangerous.SignatureExpired: "Signature age 4 > 1 seconds"` |
| 4 | `600` | ~4 s after signing | ✅ VALID — returns `.test-suffix@sl.local` |
| 5 | (hard-coded 600) | ~4 s | ✅ `check_suffix_signature()` returns `.test-suffix@sl.local` |
| 6 | `600` (tampered) | ~4 s | ❌ `itsdangerous.BadTimeSignature: "Signature b'…XXXXX' does not match"` (a subclass of `BadSignature`) |
| 7 | (hard-coded 600, tampered) | ~4 s | `check_suffix_signature()` returns `None` |
| **8** | `600` | **601 s** | ❌ `itsdangerous.SignatureExpired: "Signature age 601 > 600 seconds"` |
| **9** | `600` | **599 s** | ✅ VALID — returns `.test-suffix@sl.local` |
| **10** | (hard-coded 600) | 601 s | `check_suffix_signature()` returns `None` |
| **11** | (hard-coded 600) | 599 s | `check_suffix_signature()` returns `.test-suffix@sl.local` |

**The expiration window is exactly 600 seconds (10 minutes).** The boundary tests at `max_age=0` and `max_age=1` (Tests 2–3) produce `SignatureExpired` with messages of the form `"Signature age <elapsed> > <max_age> seconds"`, confirming that `itsdangerous` compares elapsed time against the supplied `max_age` and raises as soon as `elapsed > max_age`. The direct boundary tests 8–11 pin the cutoff: a token that is **599 seconds old is accepted**, a token that is **601 seconds old is rejected**, and the production function `check_suffix_signature` exhibits the identical cut-off (returning the decoded suffix in one case, `None` in the other). The boundary is therefore exactly `elapsed ≤ 600` ⇒ accept, `elapsed > 600` ⇒ reject.

### Conclusion

The 600-second window is hard-coded in `app/alias_suffix.py`:

```python
# app/alias_suffix.py line 11
signer = itsdangerous.TimestampSigner(config.CUSTOM_ALIAS_SECRET)

# app/alias_suffix.py lines 37–42
def check_suffix_signature(signed_suffix: str) -> Optional[str]:
    # hypothesis: user will click on the button in the 600 secs
    try:
        return signer.unsign(signed_suffix, max_age=600).decode()
    except itsdangerous.BadSignature:
        return None
```

- `TimestampSigner` embeds a timestamp in the signed token (the middle segment `aeGZOw` in our observed token `.test-suffix@sl.local.aeGZOw.JayFjy80p5Go3tsYLHhp-zOjMss` is the base62-encoded timestamp).
- `signer.unsign(signed_suffix, max_age=600)` enforces that the embedded timestamp is less than 600 seconds old; if not, it raises `itsdangerous.SignatureExpired` (a subclass of `BadSignature`).
- `check_suffix_signature()` catches all `BadSignature` subclasses and returns `None`, which the alias-creation view interprets as "invalid / expired suffix".
- `CUSTOM_ALIAS_SECRET` in our test environment resolves to `"secretcustom_alias"` (FLASK_SECRET `"secret"` + literal string `"custom_alias"`).

Thus a user clicking the "Create alias" confirmation button must do so within **10 minutes (600 seconds)** of the page being rendered, or the suffix token is silently rejected.

---

## Section 7 — API Key Usage Statistics Tracking

### Method

The `api_key` table row for the test API key was queried before and after performing three successive API calls carrying the key in the `Authentication` header. To obtain a clean baseline, the row was first reset to `times=0, last_used=None, sudo_mode_at=None` and committed, then the ORM `Session.expire_all()` was called so that subsequent reads hit the database (and reflect any writes made by the Flask worker's own SQLAlchemy session).

### Evidence

**BEFORE — three API calls (after reset to a clean baseline):**

```text
BEFORE: id=<key_id> user_id=<user_id> code=<api_key_code> times=0 last_used=None sudo_mode_at=None
```

**Three API calls with the key:**

```bash
for i in 1 2 3; do
  curl -s -H 'Authentication: <api_key_code>' \
       http://localhost:7777/api/user_info -w '\nHTTP %{http_code}\n'
done
```

```text
CALL 1: HTTP 200
CALL 2: HTTP 200
CALL 3: HTTP 200
```

All three calls returned the same JSON user-info body as in Section 1:

```json
{
  "can_create_reverse_alias": true,
  "connected_proton_address": null,
  "email": "test@test.com",
  "in_trial": true,
  "is_premium": true,
  "max_alias_free_plan": 3,
  "name": "Test User",
  "profile_picture_url": null
}
```

**AFTER — three API calls:**

```text
AFTER: times=3 last_used=<Arrow [2026-04-17T04:02:41.861677+00:00]> sudo_mode_at=None
```

### Observed Result

| Column | Before | After | Delta |
|--------|--------|-------|-------|
| `times` | **0** | **3** | **+3** (exactly one increment per call) |
| `last_used` | `None` | `<Arrow [2026-04-17T04:02:41.861677+00:00]>` | Overwritten each call — final value ≈ the timestamp of the third call |
| `sudo_mode_at` | `None` | `None` | **unchanged** — regular API calls do NOT modify `sudo_mode_at` |

Two columns are updated: `times` (incremented by 1 per successful key-authenticated request) and `last_used` (replaced with the current `arrow.now()` on each call). `sudo_mode_at` is NOT touched by regular API usage.

### Conclusion

The update occurs in `authorize_request()` at `app/api/base.py` lines 28–32, on the branch that runs when `ApiKey.get_by(code=api_code)` returns a hit:

```python
# app/api/base.py lines 28–32
else:
    # Update api key stats
    api_key.last_used = arrow.now()
    api_key.times += 1
    Session.commit()
    g.user = api_key.user
```

Every request through `@require_api_auth` (and `@require_api_sudo`) invokes `authorize_request()` first. When a valid API key is present in the `Authentication` header, the matching `ApiKey` row is mutated in place: `last_used` is set to the current wall-clock time (`arrow.now()`), `times` is incremented by 1, and the change is flushed with `Session.commit()`. The `sudo_mode_at` column (`app/models.py` line 2360, `ArrowType`, `default=None`) is only set elsewhere — by the sudo-mode endpoint in the auth flow — not by regular API usage, which explains why it remained `None` across all three calls.

```python
# app/models.py lines 2350–2360 (ApiKey schema)
class ApiKey(Base, ModelMixin):
    """used in browser extension to identify user"""
    __tablename__ = "api_key"
    user_id = sa.Column(sa.ForeignKey(User.id, ondelete="cascade"), nullable=False)
    code = sa.Column(sa.String(128), unique=True, nullable=False)
    name = sa.Column(sa.String(128), nullable=True)
    last_used = sa.Column(ArrowType, default=None)
    times = sa.Column(sa.Integer, default=0, nullable=False)
    sudo_mode_at = sa.Column(ArrowType, default=None)
```

---

## Section 8 — Failed Login Response and Logging

### Method

A fresh cookie jar was created, `GET /auth/login` fetched a valid CSRF token, then `POST /auth/login` was issued with the correct email `test@test.com` but password `WRONG_PASSWORD`. The HTTP response line, the Flask server console output, and the rendered HTML body were all captured.

### Evidence

**HTTP request/response (verbatim):**

```http
> POST /auth/login HTTP/1.1
> Host: localhost:7777
> User-Agent: curl/7.88.1
> Accept: */*
> Cookie: slapp=5ee37732-71c2-4c3a-b8ef-43e1e46fb768.fCgYMMfN-SGAmg-_3siX_ArVchU
> Content-Length: 146
> Content-Type: application/x-www-form-urlencoded
>
< HTTP/1.0 200 OK
< Content-Type: text/html; charset=utf-8
< Content-Length: 359004
< Set-Cookie: slapp=5ee37732-71c2-4c3a-b8ef-43e1e46fb768.fCgYMMfN-SGAmg-_3siX_ArVchU; Expires=Fri, 24-Apr-2026 02:22:13 GMT; HttpOnly; Path=/; SameSite=Lax
< Server: Werkzeug/1.0.1 Python/3.10.18
< Date: Fri, 17 Apr 2026 02:22:13 GMT
```

**Toastr flash message extracted from the rendered HTML:**

```text
toastr.error("Email or password incorrect")
```

**Flask server log lines for the GET + failing POST pair:**

```text
2026-04-17 02:22:12,956 - SL - DEBUG - 23740 - "/workspace/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.03744339942932129
2026-04-17 02:22:13,278 - SL - DEBUG - 23740 - "/workspace/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 200, takes 0.31087756156921387
```

### Observed Result

- **HTTP status code:** `200 OK` — a failed credential check does *not* return 401/403/4xx. The login page is simply re-rendered in response.
- **Flash error rendering:** the server sends the HTML for `auth/login.html` which includes `toastr.error("Email or password incorrect")` — produced by Jinja rendering of the flashed message.
- **Session cookie:** unchanged (`5ee37732-...`) — the failed login does not rotate or invalidate the session.
- **Server log:** a single standard access-log DEBUG line of the form `127.0.0.1 POST /auth/login ImmutableMultiDict([]) 200, takes <seconds>`. There is no explicit `"login failed"` or `"authentication failure"` WARNING/ERROR log line emitted for this event by the web app. The POST took ~311 ms (dominated by bcrypt password verification).
- **LoginEvent:** `LoginEvent(LoginEvent.ActionType.failed).send()` is invoked (see `app/auth/views/login.py` line 50), which in turn calls `newrelic.agent.record_custom_event("LoginEvent", {"action": "failed", "source": "web"})`. Without a New Relic agent configured, this call is a no-op and produces no console log output — matching what we observed.

### Conclusion

The code path is `app/auth/views/login.py` lines 45–50, which runs after `User.get_by()` resolves the user but `user.check_password(form.password.data)` returns `False`:

```python
# app/auth/views/login.py lines 40–50
if form.validate_on_submit():
    email = sanitize_email(form.email.data)
    canonical_email = canonicalize_email(email)
    user = User.get_by(email=email) or User.get_by(email=canonical_email)

    if not user or not user.check_password(form.password.data):
        # Trigger rate limiter
        g.deduct_limit = True
        form.password.data = None
        flash("Email or password incorrect", "error")
        LoginEvent(LoginEvent.ActionType.failed).send()
```

Three things happen:

1. `g.deduct_limit = True` — the `@limiter.limit("10/minute", deduct_when=...)` decorator (line 22–24) deducts a rate-limit token only for failed attempts, matching the intended "10 failed logins per minute" quota.
2. `flash("Email or password incorrect", "error")` — the flash category `"error"` is picked up by the base template, which renders it as a `toastr.error("…")` JavaScript call inside the HTML body.
3. `LoginEvent(LoginEvent.ActionType.failed).send()` — dispatches a New Relic custom event via `newrelic.agent.record_custom_event("LoginEvent", {"action": "failed", "source": "web"})` (`app/events/auth_event.py` lines 22–25). In a local dev environment without New Relic, this is silent.

The view function then falls through to `render_template("auth/login.html", ...)` at lines 74–82, which returns HTTP 200 by default. No redirect, no 4xx — the login form is simply re-rendered with the error flash. The Flask access-log DEBUG line `"POST /auth/login ImmutableMultiDict([]) 200"` is the only log emitted by the SimpleLogin application for this event.

---

## Consolidated Findings

| # | Investigation question | Observed answer |
|---|------------------------|-----------------|
| 1 | API call with session cookie and no `Authentication` header | **HTTP 200** + full user-info JSON (`{"email":"test@test.com", ...}`) |
| 2 | Sudo endpoint (`DELETE /api/user`) via session cookie only | **HTTP 500** `{"error":"Internal error"}` — `AttributeError: 'NoneType' object has no attribute 'sudo_mode_at'` at `app/api/base.py:47` |
| 3 | Session serialization | Python `pickle` **protocol 4** (`0x80 0x04` header) |
| 3 | Authenticated session dict keys | `_fresh`, `_id`, `_permanent`, `_user_id`, `csrf_token`, `sudo_time` |
| 3 | Redis session key structure | `session:<uuid4>` (e.g. `session:d9884851-3f0d-4b71-9e3d-02971caee288`) |
| 3 | Cookie structure | `<uuid>.<hmac-signature>` signed via `itsdangerous.Signer(secret, salt="session", key_derivation="hmac")` |
| 3 | TTL | 300 s unauthenticated, 604 800 s (7 d) authenticated |
| 4 | Session UUID across login | **UNCHANGED** — UUID and signature are byte-identical pre/post login |
| 5 | `X-Custom-Header` through forwarding | **STRIPPED** |
| 5 | `Received` through forwarding | **STRIPPED** |
| 5 | `Reply-To` through forwarding | **STRIPPED** (may be re-added downstream as a reverse-alias address, but the original `Reply-To` is removed) |
| 6 | Alias suffix token expiration window | **600 seconds (10 minutes)** — hard-coded `max_age=600` in `check_suffix_signature()` (`app/alias_suffix.py:40`) |
| 7 | API-key fields updated per authenticated call | `times` += 1, `last_used` := `arrow.now()` (`sudo_mode_at` unchanged) |
| 7 | Observed deltas for 3 calls (clean baseline) | `times: 0 → 3`, `last_used: None → 2026-04-17T04:02:41.861677+00:00` |
| 8 | Failed login HTTP status | **HTTP 200** (login page is re-rendered) |
| 8 | Failed login flash message | `toastr.error("Email or password incorrect")` |
| 8 | Failed login server log | One standard DEBUG access-log line: `POST /auth/login ImmutableMultiDict([]) 200, takes 0.310...` (no explicit "login failed" log line; a silent `LoginEvent(failed)` New-Relic custom event is fired) |
