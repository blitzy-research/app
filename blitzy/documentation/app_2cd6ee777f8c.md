# SimpleLogin Runtime Investigation — Evidence-Grounded Technical Q&A

**Branch:** `app_2cd6ee777f8c` &nbsp;•&nbsp; **Commit:** `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`
**Runtime:** Docker image `andrewparkscaleai/coding-agent:simple-login__app__2cd6ee777f8c2d3531559588bcfb18627ffb5d2c` (from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`) — Python 3.10.18, Postgres 15, Redis 7.
**Methodology (RUN-FIRST):** every answer below was produced by **building and running** the SimpleLogin app (gunicorn on `:7777` against Postgres + Redis) and driving it with temporary observation scripts. The quoted output blocks are the **actual captured runtime output** — HTTP status lines and bodies, raw Redis bytes, decoded cookies, database rows, measured timings, flashed messages, log lines, and tracebacks. Code references use `file:line` against the repository at the commit above. Temporary scripts were removed afterward; the repository is unchanged apart from this document.

## How the system was run

The app was served exactly as the `Dockerfile` CMD specifies (`Dockerfile:47`), inside the provided container:

```bash
# inside the running container (services: Postgres 15 @ :5432, Redis 7 @ :6379)
source /app/venv/bin/activate
export PYTHONPATH=/app CONFIG=/app/tests/test.env \
       DB_URI=postgresql://test:test@localhost:5432/test   # test.env pins :15432; real port is :5432
# DISABLE_RATE_LIMIT intentionally left UNSET so the login limiter is active (Q8)
gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
```

Boot / readiness (verbatim):

```
[2026-07-01 04:42:24 +0000] [3505] [INFO] Starting gunicorn 20.0.4
[2026-07-01 04:42:24 +0000] [3505] [INFO] Listening at: http://0.0.0.0:7777 (3505)
[2026-07-01 04:42:24 +0000] [3506] [INFO] Booting worker with pid: 3506
[2026-07-01 04:42:24 +0000] [3507] [INFO] Booting worker with pid: 3507
load config file /app/tests/test.env
>>> URL: http://localhost
```

```
$ curl -s -i http://localhost:7777/health
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Content-Type: text/html; charset=utf-8
Content-Length: 7
Set-Cookie: slapp=beed8265-726a-4f98-9200-77bd1b1eb494.o4WHuaFNjAWm1QF5HhyJ3lJIdEU; Expires=Wed, 08-Jul-2026 04:42:42 GMT; HttpOnly; Path=/; SameSite=Lax

success
```

`/health` returns `"success", 200` (`server.py:213-215`) and already stamps the `slapp` session cookie in the `<uuid>.<signature>` form (relevant to Q3/Q4).

**Test identity provisioned** (committed to the `test` DB so the live gunicorn sees it): user `blitzy_qna_thwf4xlu1j@mailbox.test` (id `395`, `activated=t`), password `password`, and API key id `45`. The database schema was already migrated (alembic head `32f25cbf12f6`, `api_key` table present). The provisioning modeled the existing helpers `tests/utils.py:create_new_user` and `ApiKey.create(user.id, ...)` (`tests/api/utils.py:11`) but was executed as a throwaway script, not committed.

> **Reporting stance (per the governing rule):** this is an *investigation*. Where a behavior looks like a defect (Q2 `None.sudo_mode_at`, Q4 session-ID non-rotation), it is **reported with rationale, not fixed**.

---

## Q1 — Keyless, session-authenticated API call

**Question decomposed:** (1) the actual HTTP **status code**; (2) the response **body structure**.

**What was run** — with a valid `slapp` session cookie and **no** `Authentication` header, call an authenticated API endpoint (`GET /api/user_info`):

```python
s = requests.Session()
r = s.get("http://localhost:7777/auth/login")
csrf = re.search(r'name="csrf_token"[^>]*value="([^"]+)"', r.text).group(1)
s.post("http://localhost:7777/auth/login",
       data={"email": EMAIL, "password": "password", "csrf_token": csrf})
# session cookie now set; NO Authentication header on the next call:
r1 = s.get("http://localhost:7777/api/user_info")
```

**Verbatim output:**

```
=== cookies sent to API: {'slapp': 'd0a76835-3358-46d8-803f-0968ceaaca24.qg6kkxf7LDw17-7KyQJ8R9NDrOE'}
Q1: GET /api/user_info WITH session cookie, NO Authentication header
STATUS_LINE: HTTP 200 OK
Content-Type: application/json
RAW_BODY: {"can_create_reverse_alias":true,"connected_proton_address":null,"email":"blitzy_qna_thwf4xlu1j@mailbox.test","in_trial":true,"is_premium":true,"max_alias_free_plan":3,"name":"Blitzy QnA User","profile_picture_url":null}
BODY_KEYS_SORTED: ['can_create_reverse_alias', 'connected_proton_address', 'email', 'in_trial', 'is_premium', 'max_alias_free_plan', 'name', 'profile_picture_url']
```

**Answer:**

- **Status code: `200 OK`.**
- **Body structure:** a JSON **object** with exactly **8 keys**: `can_create_reverse_alias`, `connected_proton_address`, `email`, `in_trial`, `is_premium`, `max_alias_free_plan`, `name`, `profile_picture_url`. In this run: `is_premium=true`, `in_trial=true`, `max_alias_free_plan=3`, `email="blitzy_qna_thwf4xlu1j@mailbox.test"`, `name="Blitzy QnA User"`, the two nullable fields `null`.

**File:line references & rationale:**

- The API key is read from the header literally named **`Authentication`** — `api_code = request.headers.get("Authentication")` (`app/api/base.py:17`), **not** `Authorization`.
- With no key found but an authenticated web session, the fallback binds the session user and proceeds: `if not api_key:` → `if current_user.is_authenticated:` → `g.user = current_user` (`app/api/base.py:20-25`). On this path `g.api_key` becomes `None` (`app/api/base.py:42`). (In this branch the cookie-gate that would otherwise require the `X-Sl-Allowcookies` header is commented out — `app/api/base.py:22-24` — so a bare session cookie already triggers the fallback.)
- The endpoint is `@api_bp.route("/user_info")` + `@require_api_auth` (`app/api/views/user_info.py:50-51`) returning `jsonify(user_to_dict(user))` (`app/api/views/user_info.py:67`). The 8 body keys come from `user_to_dict` (`app/api/views/user_info.py:28-47`: `name` L30, `is_premium` L31, `email` L32, `in_trial` L33, `max_alias_free_plan` L34, `connected_proton_address` L35, `can_create_reverse_alias` L36, `profile_picture_url` L43/L45).

**Corroborating evidence — the header name really is `Authentication`:** a *valid* key sent under the wrong header `Authorization` (and no cookie) is treated as keyless → `401`, while the same key under `Authentication` → `200`:

```
Q1 supplement: valid key under WRONG header 'Authorization', NO cookie
HTTP/1.1 401 UNAUTHORIZED
--- same valid key under CORRECT header 'Authentication', NO cookie ---
HTTP/1.1 200 OK
```

And with neither cookie nor key the request is rejected with `401` by the else-branch `return jsonify(error="Wrong api key"), 401` (`app/api/base.py:27`):

```
Q1-contrast: GET /api/user_info with NO cookie and NO key
STATUS_LINE: HTTP 401 UNAUTHORIZED
RAW_BODY: {"error":"Wrong api key"}
```

---

## Q2 — Privileged (sudo) operation with only a browser session

**Question decomposed:** (1) the exact (non-standard) **HTTP status code**; (2) the exact **error-message text**.

**What was run** — with only the browser session (no API key), call a `@require_api_sudo` endpoint, `DELETE /api/user` (`app/api/views/user.py:12-13`):

```python
# same authenticated `requests.Session` as Q1 (session cookie only, no Authentication header)
r2 = s.delete("http://localhost:7777/api/user")
```

**Verbatim output (client side):**

```
Q2: DELETE /api/user WITH session cookie only, NO Authentication header
STATUS_LINE: HTTP 500 INTERNAL SERVER ERROR
Content-Type: application/json
RAW_BODY (first 800 chars): '{"error":"Internal error"}\n'
```

**Verbatim output (server side — gunicorn log, the decisive evidence):**

```
2026-07-01 04:46:06,890 - SL - ERROR - 3507 - "/app/server.py:390" - error_handler() -  - 'NoneType' object has no attribute 'sudo_mode_at'
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
2026-07-01 04:46:06,891 - SL - DEBUG - 3507 - "/app/server.py:284" - after_request() -  - 127.0.0.1 DELETE /api/user ImmutableMultiDict([]) 500, takes 0.0035140514373779297
```

**Answer (resolved empirically):**

- **Status code: `500 INTERNAL SERVER ERROR`.**
- **Error-message text (response body): `{"error":"Internal error"}`** (with a trailing newline), produced by the global error handler at `server.py:390`.
- The `440 "Need sudo"` literal is **NOT** what is returned on the session-only path — it is **unreachable** here.

**File:line references & rationale:**

- `require_api_sudo` runs `authorize_request()` (succeeds via the Q1 session fallback, setting `g.api_key = None` at `app/api/base.py:42`) and then evaluates `if not check_sudo_mode_is_active(g.api_key):` (`app/api/base.py:69`).
- `check_sudo_mode_is_active(api_key)` dereferences `api_key.sudo_mode_at` (`app/api/base.py:47`). With `api_key is None`, this is `None.sudo_mode_at` → `AttributeError`, which the framework turns into **HTTP 500** before the `return jsonify(error="Need sudo"), 440` line (`app/api/base.py:70`) is ever reached.
- This matches the user's clue that the status "isn't a standard authorization error": **440** is itself a **non-standard** code (Microsoft IIS "Login Time-out"), but in practice the request never even gets that far — it crashes with **500**. The session-only user also cannot *enter* sudo mode to avoid this, because `PATCH /api/sudo` sets `g.api_key.sudo_mode_at = arrow.now()` (`app/api/views/sudo.py:24`) and would hit the same `None` dereference. Reported, not fixed.

---

## Q3 — Session storage internals

**Question decomposed:** (1) the **raw storage bytes**; (2) the **serialization format**; (3) the **keys** present in the deserialized authenticated-session data; (4) how the **session key itself** is structured.

**What was run** — authenticate, take the `slapp` cookie, unsign it to recover the session id, read the raw Redis value at `session:<uuid>`, then `pickle.loads` it:

```python
signer = itsdangerous.Signer("secret", salt="session", key_derivation="hmac")  # app's cookie signer
cookie = s.cookies.get("slapp")
uuid_part, sig_part = cookie.rsplit(".", 1)          # cookie form: <uuid>.<signature>
sid = signer.unsign(cookie).decode()                  # verified session id
raw = redis.Redis(host="localhost", port=6379).get("session:%s" % sid)
data = pickle.loads(raw)
```

**Verbatim output:**

```
RAW_COOKIE: 491f757f-b0c3-4b46-ad73-61a4b9eead50.7PMJdOgWVV7uraa6xVEx_V6xu4k
COOKIE_UUID_PART: 491f757f-b0c3-4b46-ad73-61a4b9eead50
COOKIE_SIGNATURE_PART: 7PMJdOgWVV7uraa6xVEx_V6xu4k
UNSIGNED_SESSION_ID: 491f757f-b0c3-4b46-ad73-61a4b9eead50
UUID_MATCHES_UNSIGNED: True
REDIS_KEY: session:491f757f-b0c3-4b46-ad73-61a4b9eead50
RAW_BYTES_len: 300
RAW_BYTES_repr: b'\x80\x04\x95!\x01\x00\x00\x00\x00\x00\x00}\x94(\x8c\n_permanent\x94\x88\x8c\x06_fresh\x94\x88\x8c\ncsrf_token\x94\x8c(70b5c4454cb8698afe4e1558caa571c6832e90ac\x94\x8c\x08_user_id\x94\x8c$0d852c8f-d9a8-443a-aef4-583610deef12\x94\x8c\x03_id\x94\x8c\x80f4a4143536f3e6712e19e6ce901a12f1ff7e8c98d04dd3e41c746e85093b48a1bfaa93650d1759a0cb7f13cba57b7f96e40ed981f0c49af1cb94f9905ee1dd03\x94\x8c\tsudo_time\x94J\xc8\x9bDju.'
RAW_BYTES_first16_hex: 80049521010000000000007d94288c0a
DESERIALIZED_TYPE: dict
DESERIALIZED_KEYS: ['_permanent', '_fresh', 'csrf_token', '_user_id', '_id', 'sudo_time']
DESERIALIZED_DICT: {'_permanent': True, '_fresh': True, 'csrf_token': '70b5c4454cb8698afe4e1558caa571c6832e90ac', '_user_id': '0d852c8f-d9a8-443a-aef4-583610deef12', '_id': 'f4a4143536f3e6712e19e6ce901a12f1ff7e8c98d04dd3e41c746e85093b48a1bfaa93650d1759a0cb7f13cba57b7f96e40ed981f0c49af1cb94f9905ee1dd03', 'sudo_time': 1782881224}
REDIS_TTL_seconds: 604800
```

**Answer:**

- **Raw bytes (300 bytes):** begin `b'\x80\x04\x95!\x01\x00\x00\x00\x00\x00\x00}\x94(...'` (first 16 bytes hex `80049521010000000000007d94288c0a`).
- **Serialization format: Python `pickle`** — the `\x80\x04` preamble is the pickle **protocol 4** opcode (`PROTO 4`).
- **Keys in the deserialized authenticated session (a `dict`):** `_permanent`, `_fresh`, `csrf_token`, `_user_id`, `_id`, `sudo_time` — enumerated from the *actual* unpickled object. (Flask-Login contributes `_user_id`, `_fresh`, `_id`; Flask/CSRF contributes `_permanent`, `csrf_token`; SimpleLogin's sudo feature contributes `sudo_time`.)
- **Session-key structure:** the Redis key is `session:<uuid>` = `session:491f757f-b0c3-4b46-ad73-61a4b9eead50`. The **cookie** is `<uuid>.<signature>` = `491f757f-b0c3-4b46-ad73-61a4b9eead50.7PMJdOgWVV7uraa6xVEx_V6xu4k`; splitting on the last `.` yields the UUID session id and its HMAC signature, and unsigning reproduces the UUID exactly (`UUID_MATCHES_UNSIGNED: True`). The observed **TTL is `604800` seconds (7 days)** because `_user_id` is present.

**File:line references & rationale:**

- Pickle serializer: `try: import cPickle as pickle / except ImportError: import pickle` (`app/session.py:9-12`); the store writes `pickle.dumps(dict(session))` (`app/session.py:91`).
- Redis key builder: `SESSION_PREFIX = "session"` (`app/session.py:18`) and `_get_key` returns `f"{SESSION_PREFIX}:{session_Id}"` (`app/session.py:43-45`).
- Cookie signer: `itsdangerous.Signer(app.secret_key, salt="session", key_derivation="hmac")` (`app/session.py:37-41`); the cookie is `signer.sign(want_bytes(session.session_id))` (`app/session.py:102-104`). Cookie name is `SESSION_COOKIE_NAME = "slapp"` (`app/config.py:199`); the secret is `app.secret_key = FLASK_SECRET` (`server.py:151`).
- TTL: `ttl = int(app.permanent_session_lifetime.total_seconds())`, dropping to `300` only when `"_user_id" not in session` (`app/session.py:92-96`); the 7-day lifetime is set at `server.py:207` (`timedelta(days=7)` → 604800s), matching the observed TTL because `_user_id` is present.

---

## Q4 — Session-identifier rotation on login

**Question decomposed:** (1) the session ID **before** login; (2) the session ID **after** login; (3) a **verdict** on whether it rotates.

**What was run** — establish an *unauthenticated* session (GET the login page), record its decoded session id, then log in **reusing the same cookie jar**, and record the decoded session id again. Redis was inspected before/after to show the *same* key gaining `_user_id`:

```python
r = s.get(".../auth/login")                 # sets an unauthenticated slapp cookie
sid_before = signer.unsign(s.cookies["slapp"]).decode()
# ... login with the SAME session s ...
s.post(".../auth/login", data={...})
sid_after  = signer.unsign(s.cookies["slapp"]).decode()
```

**Verbatim output:**

```
COOKIE_BEFORE: 31898bba-8721-4f38-926f-0532dad77a60.6dKtQRRIR7C0zPeCn07AmHLX68s
SID_BEFORE: 31898bba-8721-4f38-926f-0532dad77a60
REDIS_BEFORE_keys: ['_permanent', '_fresh', 'csrf_token']
HAS_user_id_BEFORE: False
COOKIE_AFTER: 31898bba-8721-4f38-926f-0532dad77a60.6dKtQRRIR7C0zPeCn07AmHLX68s
SID_AFTER: 31898bba-8721-4f38-926f-0532dad77a60
REDIS_AFTER_keys: ['_permanent', '_fresh', 'csrf_token', '_user_id', '_id', 'sudo_time']
HAS_user_id_AFTER: True
SESSION_ID_PRESERVED: True
SAME_REDIS_KEY_now_authenticated: True
```

**Answer:**

- **Before login:** session ID `31898bba-8721-4f38-926f-0532dad77a60` (Redis keys `['_permanent', '_fresh', 'csrf_token']`, no `_user_id`).
- **After login:** session ID `31898bba-8721-4f38-926f-0532dad77a60` — **identical**. The very same Redis key simply gained `_user_id`, `_id`, `sudo_time` (`HAS_user_id_AFTER: True`).
- **Verdict: the session identifier is NOT rotated on login — it is preserved** (`SESSION_ID_PRESERVED: True`). The same identifier that existed *before* authentication becomes the *authenticated* session id, which is the classic **session-fixation** condition.

**File:line references & rationale:**

- `open_session` reuses an existing valid session id and only mints a new UUID when none is present/valid: it returns `ServerSession(data, session_id=session_id)` for an existing id (`app/session.py:68-80`, reuse at `:73-77`). A fresh UUID is assigned only on purge/logout (`purge_session`, `app/session.py:61-66`, new UUID at `:64`).
- `login_user()` does not rotate the id, and `login_manager.session_protection = "strong"` (`app/extensions.py:8`) only detects environment changes (IP/user-agent hash) — it deletes a session on mismatch but does **not** regenerate the id on a normal login. Hence the identifier is preserved. Reported as a session-fixation implication, not fixed.

---

## Q5 — Email forwarding header handling (real test email)

**Question decomposed:** for each of three incoming headers — a custom `X-` header, a `Received` header, and a `Reply-To` header — does it **survive** to the forwarded message or is it **stripped**?

**What was run** — a message carrying all three probe headers was injected through the real inbound handler `email_handler.MailHandler()._handle(envelope, msg)` with `mail_sender` in store-instead-of-send mode (modeled on `tests/handler/test_preserved_headers.py`), then the forwarded message's headers were dumped:

```python
msg["X-Test-Custom-Blitzy"] = "custom-header-probe-9c3f"
msg[headers.RECEIVED]       = "from blitzy-probe.example (HELO probe) by mx.sl.local ... BLITZYPROBE123 ..."
msg[headers.REPLY_TO]       = "blitzy-replyto-probe@external-sender.test"   # external, != alias
envelope.rcpt_tos = [alias.email]
result = email_handler.MailHandler()._handle(envelope, msg)
fwd = mail_sender.get_stored_emails()[0].msg
```

**Verbatim output — handler result, forwarded header set, and per-header verdict:**

```
=== _handle result: 250 Message accepted for delivery (status.E200 = 250 Message accepted for delivery )
=== stored emails count: 1

=== FORWARDED message: ALL headers (name -> value) ===
  Authentication-Results       : mx.google.com; dkim=pass header.i=@matera.eu ...
  Content-Type                 : multipart/alternative; boundary="----sinikael-?=_1-16563448907660.10629093370416887"
  In-Reply-To                  : <imported@frontapp.com_81c5208b4cff8b0633f167fda4e6e8e8f63b7a9b>
  Subject                      : Something
  Message-ID                   : <af07e94a66ece6564ae30a2aaac7a34c@frontapp.com>
  Content-Transfer-Encoding    : 7bit
  X-SimpleLogin-Type           : Forward
  X-SimpleLogin-EmailLog-ID    : 204
  X-SimpleLogin-Envelope-From  : env.fvepgolnhmoedjodqhya@fvepgolnhmoedjodqhya.com
  X-SimpleLogin-Original-From  : fvepgolnhmoedjodqhya@fvepgolnhmoedjodqhya.com
  X-SimpleLogin-Envelope-To    : shoved_caches546@sl.local
  Date                         : Wed, 01 Jul 2026 04:48:52 -0000
  References                   : <imported@frontapp.com_t:...> ...
  From                         : "fvepgolnhmoedjodqhya at fvepgolnhmoedjodqhya.com" <fvepgolnhmoedjodqhya_at_fvepgolnhmoedjodqhya_com_qxquyslct@sl.local>
  Reply-To                     : "blitzy-replyto-probe at external-sender.test" <blitzy-replyto-probe_at_external-sender_test_gdupzdx@sl.local>
  Cc                           : "wcwujqomekmscmaaesle at ..." <..._zjzsdba@sl.local>
  To                           : shoved_caches546@sl.local
  DKIM-Signature               : v=1; a=rsa-sha256; c=relaxed/simple; d=sl.local; ...

=== VERDICT per target header in FORWARDED message ===
  X-Test-Custom-Blitzy : STRIPPED (absent)
  Received             : STRIPPED (absent)
  Reply-To             : PRESENT value=['"blitzy-replyto-probe at external-sender.test" <blitzy-replyto-probe_at_external-sender_test_gdupzdx@sl.local>']  original_probe_survived=False
```

The forwarding log confirms the exact mechanism for `Reply-To` (note **`old:None`** — the original value was *already deleted* before the rewrite was added):

```
.../email_handler.py:589 handle_forward() - Create or get contact for reply_to_header:blitzy-replyto-probe@external-sender.test
.../email_handler.py:873 forward_email_to_mailbox() - Reply-To header, new:"blitzy-replyto-probe at external-sender.test" <blitzy-replyto-probe_at_external-sender_test_gdupzdx@sl.local>, old:None
```

**Answer (per header):**

- **Custom `X-` header (`X-Test-Custom-Blitzy`): STRIPPED** — absent from the forwarded message.
- **`Received`: STRIPPED** — absent from the forwarded message.
- **`Reply-To`: the original value does NOT survive; it is replaced.** The original `blitzy-replyto-probe@external-sender.test` is gone; in its place is a **rewritten reverse-alias** `Reply-To`: `"blitzy-replyto-probe at external-sender.test" <blitzy-replyto-probe_at_external-sender_test_gdupzdx@sl.local>`.

**File:line references & rationale:**

- Forwarding keeps only an allow-list, then deletes everything else. `headers_to_keep = [FROM, TO, CC, SUBJECT, DATE, MESSAGE_ID, REFERENCES, IN_REPLY_TO, SL_QUEUE_ID, LIST_UNSUBSCRIBE, LIST_UNSUBSCRIBE_POST] + MIME_HEADERS` (`email_handler.py:793-807`), plus `AUTHENTICATION_RESULTS` when `user.include_header_email_header` (`email_handler.py:808-809`), followed by `delete_all_headers_except(msg, headers_to_keep)` (`email_handler.py:810`). `delete_all_headers_except` lowercases the allow-list and deletes any header not in it (`app/email_utils.py:536-542`).
- Neither an arbitrary `X-` header nor `RECEIVED = "Received"` (`app/email/headers.py:14`) is in the allow-list, so both are stripped. (Note SimpleLogin's *own* `X-SimpleLogin-*` headers, e.g. `SL_DIRECTION = "X-SimpleLogin-Type"` at `app/email/headers.py:34`, appear only because they are **added after** the deletion — they are not survivors of the incoming message.) `AUTHENTICATION_RESULTS` survived here because `user.include_header_email_header` is true (default), placing it on the allow-list at `email_handler.py:808-809`.
- `REPLY_TO = "Reply-To"` (`app/email/headers.py:13`) is also absent from the allow-list, so the *original* value is deleted at `email_handler.py:810`. Earlier, because the incoming `Reply-To` is present and differs from the alias, a reply-to contact is created (`email_handler.py:586-594`); consequently a **rewritten** reverse-alias `Reply-To` is re-added *after* the deletion via `add_or_replace_header(msg, "Reply-To", reply_to_contact.new_addr())` (`email_handler.py:869-873`). The log line's `old:None` proves the original was already gone at that point.

---


## Q6 — Alias-creation (signed suffix) token expiry boundary

**Question decomposed:** (1) **confirm the expiry window**; (2) **demonstrate success immediately**; (3) **demonstrate failure after the window**.

**What was run** — the app's own signer was used to mint a signed suffix, then unsigned at several ages using **pre-aged tokens** (timestamps back-dated), rather than really waiting. Both the raw `itsdangerous` call and the app-level `check_suffix_signature` were exercised at each age:

```python
from app import config, alias_suffix
signer = alias_suffix.signer                       # itsdangerous.TimestampSigner(config.CUSTOM_ALIAS_SECRET)
tok = signer.sign(".blitzytest@sl.local")          # freshly signed
# pre-age by monkeypatching the timestamp source, then:
signer.unsign(tok_aged, max_age=600)               # raw itsdangerous
alias_suffix.check_suffix_signature(tok_aged)      # app-level wrapper
```

**Verbatim output:**

```
CUSTOM_ALIAS_SECRET == 'secretcustom_alias' (== FLASK_SECRET + 'custom_alias')
signer type: TimestampSigner
SignatureExpired subclasses BadSignature: True

[age~0s] raw signer.unsign(max_age=600) -> '.blitzytest@sl.local'
[age~0s] check_suffix_signature(...)     -> '.blitzytest@sl.local'
[age599s] raw signer.unsign(max_age=600) -> '.blitzytest@sl.local'
[age599s] check_suffix_signature(...)     -> '.blitzytest@sl.local'
[age600s] raw signer.unsign(max_age=600) -> SUCCESS '.blitzytest@sl.local' (age==max_age is NOT expired)
[age600s] check_suffix_signature(...)     -> '.blitzytest@sl.local'
[age601s] raw signer.unsign(max_age=600) -> RAW EXCEPTION TRACEBACK:
[age601s] check_suffix_signature(...)     -> None
```

The raw exception at age 601s:

```
Traceback (most recent call last):
  File "/tmp/blitzy_obs/q6.py", line 51, in <module>
    signer.unsign(tok601, max_age=600)
  File "/app/venv/lib/python3.10/site-packages/itsdangerous/timed.py", line 91, in unsign
    raise SignatureExpired(
itsdangerous.exc.SignatureExpired: Signature age 601 > 600 seconds
```

**Answer:**

- **Expiry window: `600` seconds (10 minutes).** Confirmed empirically: valid at ages `~0s`, `599s`, and exactly `600s` (age == max_age is **not** expired); first failure at `601s` with `Signature age 601 > 600 seconds`.
- **Immediate success:** raw `signer.unsign(..., max_age=600)` and `check_suffix_signature(...)` both return the decoded suffix `.blitzytest@sl.local`.
- **Post-window failure:** the **raw** `signer.unsign` **raises** `itsdangerous.exc.SignatureExpired: Signature age 601 > 600 seconds`, but the **app-level** `check_suffix_signature(...)` **returns `None`** — because `SignatureExpired` subclasses `BadSignature`, and the wrapper catches `BadSignature`.

**File:line references & rationale:**

- Signer: `signer = itsdangerous.TimestampSigner(config.CUSTOM_ALIAS_SECRET)` (`app/alias_suffix.py:11`); secret `CUSTOM_ALIAS_SECRET = FLASK_SECRET + "custom_alias"` (`app/config.py:201`) — observed value `'secretcustom_alias'` because dev `FLASK_SECRET="secret"`.
- Validator: `check_suffix_signature` calls `signer.unsign(signed_suffix, max_age=600).decode()` (`app/alias_suffix.py:37-40`) and `except itsdangerous.BadSignature: return None` (`app/alias_suffix.py:41-42`). Because `SignatureExpired` is a subclass of `BadSignature`, expiry is swallowed and surfaces as `None`.
- Consumers then log `LOG.w("Alias creation time expired for %s", ...)` and reject the request on a `None` result: `app/dashboard/views/custom_alias.py:90`, `app/api/views/new_custom_alias.py:70`, `app/oauth/views/authorize.py:184`. (itsdangerous version in the runtime image: `1.1.0`.)

---

## Q7 — API key usage statistics

**Question decomposed:** (1) **which `api_key` fields update** after several calls; (2) their **actual observed values**.

**What was run** — a **fresh** API key was created, its row was read, then **5** authenticated `GET /api/user_info` calls were made over HTTP to the live gunicorn instance (so the commits persist), and the row was re-read:

```bash
# fresh key id=46, code=fvxrevtx…<redacted ephemeral test-DB key>  (the code value is not one of the fields Q7 asks about)
psql -c "SELECT id, times, last_used, sudo_mode_at FROM api_key WHERE id=46"
for i in 1..5: curl -H "Authentication: <code>" http://localhost:7777/api/user_info
psql -c "SELECT id, times, last_used, sudo_mode_at FROM api_key WHERE id=46"
```

**Verbatim output:**

```
=== DB row BEFORE any call (id=46) ===
 id | times | last_used | sudo_mode_at
----+-------+-----------+--------------
 46 |     0 |           |

=== 5 authenticated GET /api/user_info calls ===
call #1 -> HTTP 200  at 04:50:45.773442599
call #2 -> HTTP 200  at 04:50:46.200560944
call #3 -> HTTP 200  at 04:50:46.627256044
call #4 -> HTTP 200  at 04:50:47.054232123
call #5 -> HTTP 200  at 04:50:47.481062685

=== DB row AFTER 5 calls (id=46) ===
 id | times |        last_used         | sudo_mode_at
----+-------+--------------------------+--------------
 46 |     5 | 2026-07-01 04:50:47.4645 |
```

**Answer:**

- **Fields updated:** `times` and `last_used`.
- **Observed values after 5 calls:** `times` went `0 → 5` (i.e. `times == N`, N=5); `last_used` went `NULL → 2026-07-01 04:50:47.4645` (the timestamp of the latest, 5th call).
- **`sudo_mode_at` is UNCHANGED** — it remained `NULL` (empty) throughout, because no sudo activation occurred.

**File:line references & rationale:**

- Each authenticated call with a key present executes the `else` branch: `g.api_key.last_used = arrow.now()`, `g.api_key.times += 1`, then `Session.commit()` (`app/api/base.py:30-32`). Five calls therefore set `times = 5` and stamp `last_used` with the last call's time.
- Model: `class ApiKey` (`app/models.py:2350`), `__tablename__ = "api_key"` (`app/models.py:2353`), unique `code` (`app/models.py:2356`), `last_used` ArrowType default `None` (`app/models.py:2358`), `times` Integer default `0`, not-null (`app/models.py:2359`), `sudo_mode_at` ArrowType default `None` (`app/models.py:2360`). Nothing in the authenticated read path touches `sudo_mode_at`, so it stays `NULL`.
- Note: this update happens **only** on the key-present branch. The keyless session-fallback path of Q1 sets `g.api_key = None` (`app/api/base.py:42`) and does **not** touch these counters — consistent with the fresh key starting at `times = 0` until the keyed calls were made.

---

## Q8 — Failed-login behavior

**Question decomposed:** (1) the specific **log/event output**; (2) the exact **HTTP response details**.

**What was run** — wrong credentials were POSTed to `/auth/login` against the live instance (with a valid CSRF token harvested from the login page), capturing the status line, whether it redirected, the flashed message rendered into the HTML body, and the server logs produced. A second, isolated burst then exceeded the limiter:

```python
# harvest csrf_token from GET /auth/login, then:
r = s.post(".../auth/login", data={"csrf_token": tok,
                                    "email": USER_EMAIL,
                                    "password": "definitely-the-wrong-password"})
# inspect r.status_code, r.is_redirect, and 'Email or password incorrect' in r.text
```

**Verbatim output — single failed login:**

```
POST_status_line: HTTP 200 OK
is_redirect: False
flash_present 'Email or password incorrect': True
flashed_fragment: '    \n\n            <script>toastr.error("Email or password incorrect");</script>\n'
has_logout_marker (would indicate success): False
```

**Log delta during the failed POST** (only the generic per-request DEBUG lines appear — **no** `LOG.*` line from the login event):

```
... - SL - DEBUG - ... - after_request() - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes ...
... - SL - DEBUG - ... - after_request() - 127.0.0.1 POST /auth/login ImmutableMultiDict([...]) 200, takes ...
```

**Verbatim output — limiter burst (isolated, `10/minute`):**

```
attempt #1  -> 200|text/html; charset=utf-8
attempt #2  -> 200|text/html; charset=utf-8
...
attempt #10 -> 200|text/html; charset=utf-8
attempt #11 -> 429|text/html; charset=utf-8
attempt #12 -> 429|text/html; charset=utf-8
```

The `429` page is SimpleLogin's custom error page — heading `429`, message `Whoa, slow down there, pardner!`.

**Answer:**

- **HTTP response details:** **HTTP `200 OK`** (a re-render, **not** a redirect and **not** a `401`). The response body contains the flashed error, emitted as `<script>toastr.error("Email or password incorrect");</script>` — i.e. the exact text **`Email or password incorrect`**. No logout marker is present (login did not succeed).
- **Log/event output:** the failed-login branch emits **no `LOG.*` line**. The only log output is the generic `after_request` DEBUG line reporting `POST /auth/login ... 200`. The login-failure event (`LoginEvent(...failed).send()`) records **only** a NewRelic custom event, which produces no visible log line here.
- **Limiter:** after 10 failing attempts within a minute, the **11th** returns **HTTP `429`** with SimpleLogin's custom "Whoa, slow down there, pardner!" page (10 allowed, 11th blocked).

**File:line references & rationale:**

- The wrong-credentials branch: `g.deduct_limit = True` (`app/auth/views/login.py:47`), `form.password.data = None` (`app/auth/views/login.py:48`), `flash("Email or password incorrect", "error")` (`app/auth/views/login.py:49`), `LoginEvent(LoginEvent.ActionType.failed).send()` (`app/auth/views/login.py:50`). It then falls through to `return render_template("auth/login.html", form=form, ...)` (`app/auth/views/login.py:74`) → **HTTP 200**, never a 401.
- `LoginEvent.send()` calls only `newrelic.agent.record_custom_event("LoginEvent", {"action": ..., "source": "web"})` (`app/events/auth_event.py:22-25`) — there is **no** `LOG.*` statement, which is why no login-specific log line appears; the visible line is the generic request logger in `after_request` (`server.py:284`).
- Rate limit: the route is decorated `@limiter.limit("10/minute", deduct_when=lambda r: hasattr(g, "deduct_limit") and g.deduct_limit)` (`app/auth/views/login.py:21-24`); only failed attempts (which set `g.deduct_limit = True`) count toward the limit, so the 11th failure in a minute yields `429`.

---


## Coverage Pass

Each question is re-read below and every distinct sub-part is confirmed answered, with a pointer to where the evidence lives in this document.

- **Q1 — keyless, session-authenticated API call**
  - ✅ *Actual HTTP status code* — `HTTP 200 OK` (see Q1: "Verbatim output", status line).
  - ✅ *Response-body structure* — JSON object with 8 keys (`can_create_reverse_alias`, `connected_proton_address`, `email`, `in_trial`, `is_premium`, `max_alias_free_plan`, `name`, `profile_picture_url`); full body quoted in Q1. Also: the `Authentication` vs `Authorization` header distinction (`app/api/base.py:17`).

- **Q2 — sudo-protected operation with only a browser session**
  - ✅ *Exact HTTP status code* — `HTTP 500 INTERNAL SERVER ERROR` (empirically resolved; the `440 "Need sudo"` literal at `app/api/base.py:70` is unreachable on this path). See Q2.
  - ✅ *Exact error-message text* — response body `{"error":"Internal error"}` with the server-side `AttributeError: 'NoneType' object has no attribute 'sudo_mode_at'` traceback quoted in Q2.

- **Q3 — session storage internals**
  - ✅ *Raw storage bytes* — 300 bytes beginning `b'\x80\x04\x95...'` (Q3 verbatim `RAW_BYTES_repr`).
  - ✅ *Serialization format* — Python `pickle`, protocol 4 (`\x80\x04`). See Q3.
  - ✅ *Keys in deserialized authenticated session* — `_permanent`, `_fresh`, `csrf_token`, `_user_id`, `_id`, `sudo_time` (Q3 `DESERIALIZED_KEYS`).
  - ✅ *Session-key structure* — Redis key `session:<uuid>`; cookie `<uuid>.<signature>` (Q3, with unsign confirming the UUID).

- **Q4 — session-identifier rotation on login**
  - ✅ *Before-login session ID* — `31898bba-8721-4f38-926f-0532dad77a60` (Q4 `SID_BEFORE`).
  - ✅ *After-login session ID* — `31898bba-8721-4f38-926f-0532dad77a60` (Q4 `SID_AFTER`).
  - ✅ *Verdict* — NOT rotated / preserved (`SESSION_ID_PRESERVED: True`), with the session-fixation implication (report only). See Q4.

- **Q5 — email forwarding header handling**
  - ✅ *Custom `X-` header* (`X-Test-Custom-Blitzy`) — **STRIPPED** (Q5 verdict).
  - ✅ *`Received` header* — **STRIPPED** (Q5 verdict).
  - ✅ *`Reply-To` header* — original **replaced**; a rewritten reverse-alias `Reply-To` appears in its place (`original_probe_survived=False`), with the `old:None` log proving the original was deleted before re-add. See Q5.

- **Q6 — alias-creation token expiry boundary**
  - ✅ *Confirm the window* — `600` seconds (valid at 0/599/600s, expired at 601s). See Q6.
  - ✅ *Immediate success* — raw `unsign` and `check_suffix_signature` return `.blitzytest@sl.local`. See Q6.
  - ✅ *Post-window failure* — raw `unsign` raises `SignatureExpired: Signature age 601 > 600 seconds`; `check_suffix_signature(...)` returns `None`. See Q6.

- **Q7 — API key usage statistics**
  - ✅ *Which fields update* — `times` and `last_used` (not `sudo_mode_at`). See Q7.
  - ✅ *Observed values* — `times` `0 → 5` (== N); `last_used` `NULL → 2026-07-01 04:50:47.4645`; `sudo_mode_at` remained `NULL`. See Q7 DB rows.

- **Q8 — failed-login behavior**
  - ✅ *Log/event output* — no `LOG.*` line from the failed-login event; only the generic `after_request` DEBUG line (`POST /auth/login ... 200`); `LoginEvent.send()` records only a NewRelic custom event (`app/events/auth_event.py:22-25`). See Q8.
  - ✅ *HTTP response details* — `HTTP 200 OK` re-render (not 401, not a redirect) containing the exact flashed text `Email or password incorrect`; and the `10/minute` limiter returns `HTTP 429` on the 11th failing attempt. See Q8.

**Result:** all eight questions and every enumerated sub-part are addressed with verbatim runtime evidence, exact literals, and `file:line` references.

