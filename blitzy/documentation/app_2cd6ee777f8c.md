# SimpleLogin Empirical Investigation — Q1–Q8 (branch `app_2cd6ee777f8c`)

This document answers eight behavioral questions about the SimpleLogin email‑aliasing
application. Per the `SWE-AtlasQnA-Repo` rule set, **every answer was produced by running the
relevant code path against a live runtime and pasting the actual observed output** — not by
reading the code alone. Each behavioral claim is placed next to the specific verbatim output
line that demonstrates it, together with the exact `file:line` reference in the source tree and
the rationale. Where the observed behavior contradicts an intuitive expectation, the observed
result is reported exactly as seen.

---

## Environment & Runtime

The investigation ran inside the provided container image
(`andrewparkscaleai/coding-agent:simple-login__app__2cd6ee777f8c…` from
`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`), using the repository's
Python 3.10 virtualenv (`.venv`) and the exact `poetry.lock`‑pinned dependencies. Versions were
confirmed at runtime via `importlib.metadata`:

```
flask==1.1.2
werkzeug==1.0.1
flask-login==0.5.0
flask-limiter==1.4
itsdangerous==1.1.0
redis==4.6.0
aiosmtpd==1.4.2
arrow==0.16.0
newrelic==8.8.0
gunicorn==20.0.4
python 3.10.20
```

| Component | Version | Source of truth |
|---|---|---|
| Python | 3.10.20 | `pyproject.toml:61` (`python = "^3.10"`) |
| flask | 1.1.2 | `poetry.lock` |
| werkzeug | 1.0.1 | `poetry.lock` |
| flask-login | 0.5.0 | `poetry.lock` (`session_protection = "strong"`, `app/extensions.py:8`) |
| flask-limiter | 1.4 | `poetry.lock` (login rate limit `10/minute`) |
| itsdangerous | 1.1.0 | `poetry.lock` (session cookie signer + alias‑suffix `TimestampSigner`) |
| redis | 4.6.0 | `poetry.lock` (session/rate‑limit backend client) |
| aiosmtpd | 1.4.2 | `poetry.lock` (inbound SMTP handler) |
| arrow | 0.16.0 | `poetry.lock` (`last_used`, sudo window timestamps) |
| newrelic | 8.8.0 | `poetry.lock` (Q8 custom event) |
| PostgreSQL server | 13 | Docker `postgres:13`, host port `15432` |
| Redis server | 6 | Docker `redis:6`, host port `6379` |

### Services and how they were started

- **PostgreSQL 13** (`sl-postgres`) on `127.0.0.1:15432` and **Redis 6** (`sl-redis`) on
  `127.0.0.1:6379`, both already running as Docker containers matching the CI stack.
- Configuration was taken from `tests/test.env`, which sets
  `DB_URI=postgresql://test:test@localhost:15432/test` (`tests/test.env:17`),
  `FLASK_SECRET=secret` (`tests/test.env:20`), `NOT_SEND_EMAIL=true` (`tests/test.env:7`),
  `URL=http://localhost` (`tests/test.env:2`), and — critically —
  `MEM_STORE_URI=redis://localhost` (`tests/test.env:78`).
- Because `MEM_STORE_URI` is set, `server.py:163-165` calls `initialize_redis_services(...)`, which
  installs the server‑side `RedisSessionStore` (`app/redis_services.py:12`). This was **confirmed at
  runtime**: `type(app.session_interface).__name__` printed `RedisSessionStore` (module
  `app.session`). Server‑side Redis sessions are therefore the active session backend (required for
  Q3/Q4).
- The **Flask web application** was started with gunicorn against `wsgi:app`
  (`wsgi.py` → `from server import create_app; app = create_app()`):

  ```
  CONFIG=tests/test.env PYTHONPATH=$PWD .venv/bin/gunicorn wsgi:app -b 127.0.0.1:7777 \
      --workers 1 --timeout 120 --log-level info
  ```

  Observed startup (verbatim):

  ```
  [2026-07-01 22:03:54 +0000] [40980] [INFO] Starting gunicorn 20.0.4
  [2026-07-01 22:03:54 +0000] [40980] [INFO] Listening at: http://127.0.0.1:7777 (40980)
  [2026-07-01 22:03:54 +0000] [40980] [INFO] Using worker: sync
  [2026-07-01 22:03:54 +0000] [40982] [INFO] Booting worker with pid: 40982
  load config file /tmp/.../tests/test.env
  >>> URL: http://localhost
  Upload files to local dir
  >>> init logging <<<
  ```

  The `>>> init logging <<<` line is printed by `app/log.py:67`. A health probe returned
  `GET /auth/login -> HTTP 200`.
- The **inbound SMTP pipeline** (`email_handler.py`) was exercised in‑process for Q5 through its
  real top‑level entry point `email_handler.handle(envelope, msg)` (`email_handler.py:1945`), with
  outgoing mail captured by `app.mail_sender.mail_sender` instead of being delivered
  (`NOT_SEND_EMAIL=true`).

### Seed data

Following the `tests/conftest.py` / `tests/utils.py` pattern (`create_app()` from `server.py`,
then `add_sl_domains()` + `add_proton_partner()` from `init_app.py`), one activated **User**, one
**Alias**, and one **ApiKey** were created and **committed** so the separate gunicorn process
observes them:

```
user_id      = 793
email        = q_investigation@mailbox.test
password     = investig8-pw          (a made-up test password)
alias_id     = 1287
alias_email  = simplelogin-newsletter.narrow598@sl.local
api_key_id   = 91
api_key_code = ltxgzsgc…             (60-char random ephemeral test key; truncated here)
```

### Methodology notes

- For the HTTP questions (Q1, Q2, Q8) a real `requests.Session` cookie session was used against
  the live gunicorn server rather than the pytest `CustomTestClient`. The test client injects the
  header `X-Sl-Allowcookies: allow` on every request (`tests/conftest.py:54,69`;
  `HEADER_ALLOW_API_COOKIES = "X-Sl-Allowcookies"`, `app/constants.py:1`). Because the cookie gate
  is commented out (see Q1), that header does not change behavior, but a real browser‑style cookie
  session avoids any masking.
- Rate limiting was **enabled** during the run: `DISABLE_RATE_LIMIT = "DISABLE_RATE_LIMIT" in
  os.environ` (`app/config.py:602`) and that variable was not set.
- Server log lines quoted below are the SimpleLogin `SL` logger's stdout, whose format is defined
  at `app/log.py:12-14`. All temporary observation scripts were deleted after capture; the only
  file added to the repository is this document.

---

## Q1 — API call WITHOUT `Authentication` header while logged into the web interface

**Question.** With only a browser session cookie (no API key header), what HTTP status code comes
back from a JSON API endpoint, and what is the response body structure?

**How it was run.** A `requests.Session` logged in via `POST /auth/login` (obtaining the `slapp`
cookie), then issued `GET /api/user_info` with **no `Authentication` header**:

```python
s = requests.Session()
# GET /auth/login to read csrf_token, then:
s.post("http://127.0.0.1:7777/auth/login",
       data={"email": EMAIL, "password": PASSWORD, "csrf_token": csrf}, allow_redirects=False)
r1 = s.get("http://127.0.0.1:7777/api/user_info")   # NO Authentication header
print(r1.status_code, r1.headers.get("Content-Type"), r1.text)
print(list(r1.json().keys()))
```

**Observed output (verbatim).**

```
=== LOGIN POST status: 302 Location: http://127.0.0.1:7777/dashboard/
=== cookies after login: ['slapp']

########## Q1: GET /api/user_info (cookie session, NO Authentication header) ##########
HTTP status_code: 200
Content-Type: application/json
Response body (text): {"can_create_reverse_alias":true,"connected_proton_address":null,"email":"q_investigation@mailbox.test","in_trial":true,"is_premium":true,"max_alias_free_plan":3,"name":"Q Investigation User","profile_picture_url":null}
Response JSON keys: ['can_create_reverse_alias', 'connected_proton_address', 'email', 'in_trial', 'is_premium', 'max_alias_free_plan', 'name', 'profile_picture_url']
```

**Answer.**

- **HTTP status code is `200`.** Evidence: `HTTP status_code: 200`.
- **The response body is a JSON object** (`Content-Type: application/json`) with exactly these
  **eight keys**: `can_create_reverse_alias`, `connected_proton_address`, `email`, `in_trial`,
  `is_premium`, `max_alias_free_plan`, `name`, `profile_picture_url`. Evidence: the
  `Response JSON keys: [...]` line and the verbatim body above.

**Citations.** `authorize_request()` at `app/api/base.py:16-43`: when `not api_key`
(`app/api/base.py:20`) and `current_user.is_authenticated` (`app/api/base.py:21`), the
`HEADER_ALLOW_API_COOKIES` guard is **commented out** (`app/api/base.py:22-24`), so
`g.user = current_user` runs unconditionally (`app/api/base.py:25`) and `g.api_key = api_key` is
set to `None` (`app/api/base.py:42`). The endpoint `GET /api/user_info`
(`app/api/views/user_info.py:50-52`) returns `jsonify(user_to_dict(user))`
(`app/api/views/user_info.py:67`); the eight keys are exactly those built by `user_to_dict`
(`app/api/views/user_info.py:28-47`).

**Rationale.** With no API key present, `authorize_request()` falls back to the Flask‑Login
session established by the browser cookie. Since the cookie gate is disabled, the authenticated
`current_user` is accepted and the endpoint returns its normal `200` JSON payload.

---

## Q2 — Privileged operation with ONLY a browser session (no API key)

**Question.** A privileged (sudo) operation performed with only a browser session returns a
non‑standard status code with an unusual error message. What exact status code and exact error
message text does the system return?

**How it was run.** Using the **same** authenticated `requests.Session` from Q1, a
`DELETE /api/user` was issued (the only `@require_api_sudo` endpoint), with no `Authentication`
header. The server's stdout log delta was captured around the request:

```python
r2 = s.delete("http://127.0.0.1:7777/api/user")
print(r2.status_code, r2.headers.get("Content-Type"), r2.text)
# + read the new bytes appended to the gunicorn stdout log
```

**Observed output (verbatim).**

```
########## Q2: DELETE /api/user (cookie session, NO Authentication header) ##########
HTTP status_code: 500
Content-Type: application/json
Response body (text): {"error":"Internal error"}
----- SERVER LOG delta during DELETE /api/user -----
2026-07-01 22:05:23,086 - SL - ERROR - 40982 - "/tmp/.../server.py:390" - error_handler() -  - 'NoneType' object has no attribute 'sudo_mode_at'
Traceback (most recent call last):
  File ".../flask/app.py", line 1950, in full_dispatch_request
    rv = self.dispatch_request()
  File ".../flask/app.py", line 1936, in dispatch_request
    return self.view_functions[rule.endpoint](**req.view_args)
  File ".../app/api/base.py", line 69, in decorated
    if not check_sudo_mode_is_active(g.api_key):
  File ".../app/api/base.py", line 47, in check_sudo_mode_is_active
    return api_key.sudo_mode_at and g.api_key.sudo_mode_at >= arrow.now().shift(
AttributeError: 'NoneType' object has no attribute 'sudo_mode_at'
2026-07-01 22:05:23,086 - SL - DEBUG - 40982 - ".../server.py:284" - after_request() -  - 127.0.0.1 DELETE /api/user ImmutableMultiDict([]) 500, takes 0.003726959228515625
```

**Answer (reported exactly as observed).**

- **The status code is `500`** — not `401`, not `403`, and not the `440` "Need sudo" code.
  Evidence: `HTTP status_code: 500`.
- **The exact error message text in the body is `{"error":"Internal error"}`.** Evidence:
  `Response body (text): {"error":"Internal error"}`.
- **The underlying cause is an `AttributeError`.** Evidence: the logged line
  `'NoneType' object has no attribute 'sudo_mode_at'` and the traceback ending in
  `AttributeError: 'NoneType' object has no attribute 'sudo_mode_at'`.

**Citations.** The sole `@require_api_sudo` endpoint is `DELETE /api/user`
(`app/api/views/user.py:12-14`). `require_api_sudo` calls `check_sudo_mode_is_active(g.api_key)`
(`app/api/base.py:69`); because Q1's fallback set `g.api_key = None` (`app/api/base.py:42`),
`check_sudo_mode_is_active` dereferences `api_key.sudo_mode_at` on `None`
(`app/api/base.py:46-49`, specifically `app/api/base.py:47`) and raises `AttributeError`. The
`440` branch (`app/api/base.py:70`) is never reached. The global handler
`@app.errorhandler(Exception)` (`server.py:388-394`) logs the exception via `LOG.e(e)`
(`server.py:390`; `LOG.e` = `logging.Logger.exception`, `app/log.py:77`) and, because the path
starts with `/api/` (`server.py:391`), returns `jsonify(error="Internal error"), 500`
(`server.py:392`). There is no global `CSRFProtect`, so the `DELETE` genuinely reaches the view.

**Rationale.** A browser session carries no API key, so `g.api_key` is `None`. The sudo check
dereferences that `None`, raising `AttributeError`, which the catch‑all handler converts into a
JSON `500`. The "unusual" status/message pair (`500` / `Internal error`) is thus an unhandled
programming error surfacing, not an authorization decision.

---

## Q3 — Session data storage format (inspected directly in the backend)

**Question.** Examine stored session data directly in the storage backend: capture the raw bytes,
show what format they are in, list the keys present in the deserialized data for an authenticated
session, and show how the session key itself is structured.

**How it was run.** After logging in, the `slapp` cookie was unsigned to the underlying session id
using the same signer the app uses (`itsdangerous.Signer("secret", salt="session",
key_derivation="hmac")`, mirroring `app/session.py:37-41`). The Redis value at `session:<id>` was
then read raw, its TTL fetched, and the bytes deserialized with `pickle`:

```python
import redis, itsdangerous, pickle
signer = itsdangerous.Signer("secret", salt="session", key_derivation="hmac")
sid = signer.unsign(slapp_cookie).decode()
r = redis.Redis(host="localhost", port=6379)
raw = r.get(f"session:{sid}"); ttl = r.ttl(f"session:{sid}")
print(repr(raw[:200])); print(hex(raw[0])); print(ttl)
data = pickle.loads(raw); print(data); print(sorted(data.keys()))
```

**Observed output (verbatim).**

```
########## Q3: session storage format (authenticated session) ##########
session key layout: session:0a82aa68-c621-43cd-8934-7952635c3654   (SESSION_PREFIX:'session' + ':' + uuid)
TTL(seconds): 604800
raw type: bytes len: 300
raw first byte (hex): 0x80 (pickle protocol opcode \x80 => pickle stream)
raw bytes (repr, first 200): b'\x80\x04\x95!\x01\x00\x00\x00\x00\x00\x00}\x94(\x8c\n_permanent\x94\x88\x8c\x06_fresh\x94\x88\x8c\ncsrf_token\x94\x8c(a38f4f24a3187cec2dc2396a7c0e615beee99706\x94\x8c\x08_user_id\x94\x8c$a05c7925-b487-4e09-b1ad-9398e22877fe\x94\x8c\x03_id\x94\x8c\x80f4a4143536f3e6712e19e6ce901a12f1ff7e8c98d04dd3e4'
pickle.loads -> type: dict
deserialized dict: {'_permanent': True, '_fresh': True, 'csrf_token': 'a38f4f24a3187cec2dc2396a7c0e615beee99706', '_user_id': 'a05c7925-b487-4e09-b1ad-9398e22877fe', '_id': 'f4a4143536f3e6712e19e6ce901a12f1ff7e8c98d04dd3e41c746e85093b48a1bfaa93650d1759a0cb7f13cba57b7f96e40ed981f0c49af1cb94f9905ee1dd03', 'sudo_time': 1782943684}
deserialized dict keys: ['_fresh', '_id', '_permanent', '_user_id', 'csrf_token', 'sudo_time']
```

**Answer.**

- **Raw bytes.** The stored value is a Python `bytes` object of length `300`; its first byte is
  `0x80`. Evidence: `raw type: bytes len: 300` and `raw first byte (hex): 0x80`. The full leading
  bytes are shown in the `raw bytes (repr, first 200)` line, beginning `b'\x80\x04\x95…'`.
- **Format is Python `pickle`.** The `\x80` opcode is the pickle `PROTO` marker and `\x80\x04`
  indicates **pickle protocol 4**. Evidence: the `\x80\x04` prefix in the repr, and
  `pickle.loads -> type: dict` succeeding.
- **Keys present for an authenticated session:** `_fresh`, `_id`, `_permanent`, `_user_id`,
  `csrf_token`, `sudo_time`. Evidence: `deserialized dict keys: ['_fresh', '_id', '_permanent',
  '_user_id', 'csrf_token', 'sudo_time']`. Here `_user_id`, `_fresh`, `_id` are Flask‑Login fields,
  `csrf_token` is Flask‑WTF's token, `sudo_time` is set by `after_login`
  (`app/auth/views/login_utils.py:37`), and `_permanent` is the permanent‑session flag.
- **Session key structure.** The Redis key is `session:<uuid>` —
  `session:0a82aa68-c621-43cd-8934-7952635c3654`. Evidence: the `session key layout:` line.
- **TTL.** `604800` seconds (7 days) for this authenticated session. Evidence:
  `TTL(seconds): 604800`.

**Citations.** Key layout: `_get_key` returns `f"{SESSION_PREFIX}:{session_Id}"`
(`app/session.py:44-45`) with `SESSION_PREFIX = "session"` (`app/session.py:18`). Serialization:
`save_session` writes `val = pickle.dumps(dict(session))` (`app/session.py:91`) and stores it with
`setex(...)` (`app/session.py:97-101`); `open_session` reads it back with `pickle.loads(val)`
(`app/session.py:76`). TTL: `ttl = int(app.permanent_session_lifetime.total_seconds())`
(`app/session.py:92`) where `app.permanent_session_lifetime = timedelta(days=7)` (`server.py:207`)
gives `604800`; anonymous sessions instead use `ttl = 300` (`app/session.py:95-96`). The cookie is
named `slapp` (`app/config.py:199`, applied at `server.py:159`) and holds the signed id
(`app/session.py:102-114`).

**Rationale.** SimpleLogin uses a custom server‑side session store: only a signed opaque id lives
in the `slapp` cookie, while the actual session dictionary is pickled and stored in Redis under
`session:<uuid>` with a TTL. The `0x80` leading byte and successful `pickle.loads` are the direct
evidence of the pickle format.

---

## Q4 — Session identifier behavior during authentication (session fixation)

**Question.** Capture the session id from the cookie BEFORE login, go through the login flow, and
determine whether the identifier stays the same or is replaced with a new one. Show the actual
before and after values.

**How it was run.** A fresh `requests.Session` made an anonymous `GET /auth/login` (capturing and
decoding the `slapp` cookie → session id BEFORE), then `POST /auth/login` with correct credentials
on the same session (decoding the `slapp` cookie → session id AFTER). The Redis dictionary was
dumped both times.

**Observed output (verbatim).**

```
########## Q4: session identifier BEFORE vs AFTER login ##########
BEFORE login:
  slapp cookie (raw): 0a82aa68-c621-43cd-8934-7952635c3654.1Jj6mgkpliogv__U2fQJC6nlJF8
  decoded session id (BEFORE): 0a82aa68-c621-43cd-8934-7952635c3654
  redis dict (BEFORE): {'_permanent': True, '_fresh': False, 'csrf_token': 'a38f4f24a3187cec2dc2396a7c0e615beee99706'}
  redis keys (BEFORE): ['_fresh', '_permanent', 'csrf_token']

AFTER login (POST /auth/login -> 302 ):
  Set-Cookie slapp (on 302): 0a82aa68-c621-43cd-8934-7952635c3654.1Jj6mgkpliogv__U2fQJC6nlJF8
  decoded session id (AFTER, from Set-Cookie): 0a82aa68-c621-43cd-8934-7952635c3654
  decoded session id (AFTER, from jar): 0a82aa68-c621-43cd-8934-7952635c3654
  redis keys (AFTER): ['_fresh', '_id', '_permanent', '_user_id', 'csrf_token', 'sudo_time']
  '_user_id' in AFTER dict: True value: a05c7925-b487-4e09-b1ad-9398e22877fe

>>> Q4 CONCLUSION:
  session id BEFORE: 0a82aa68-c621-43cd-8934-7952635c3654
  session id AFTER : 0a82aa68-c621-43cd-8934-7952635c3654
  SAME identifier across login?: True
```

**Answer (reported exactly as observed).**

- **The identifier is UNCHANGED across login** — it is *not* replaced with a new one.
  - BEFORE value: `0a82aa68-c621-43cd-8934-7952635c3654`. Evidence:
    `decoded session id (BEFORE): 0a82aa68-c621-43cd-8934-7952635c3654`.
  - AFTER value: `0a82aa68-c621-43cd-8934-7952635c3654`. Evidence:
    `decoded session id (AFTER, from Set-Cookie): 0a82aa68-c621-43cd-8934-7952635c3654`.
  - Direct comparison: `SAME identifier across login?: True`.
- **Only the session *data* changes**: it gains `_user_id` (plus `_id` and `sudo_time`). Evidence:
  keys go from `['_fresh', '_permanent', 'csrf_token']` (BEFORE) to `['_fresh', '_id', '_permanent',
  '_user_id', 'csrf_token', 'sudo_time']` (AFTER), and `'_user_id' in AFTER dict: True value:
  a05c7925-b487-4e09-b1ad-9398e22877fe`.

**Citations.** `after_login` performs `login_user(user)` and sets `session["sudo_time"]`
(`app/auth/views/login_utils.py:36-37`) but does **no** id rotation
(`app/auth/views/login_utils.py:12-45`). The only rotation is `purge_session`, which assigns a new
uuid (`app/session.py:61-66`, specifically `app/session.py:64`) and is invoked from
`logout_session()` (`app/session.py:117-121`) — i.e. on logout, not login.
`login_manager.session_protection = "strong"` (`app/extensions.py:8`) writes the Flask‑Login `_id`
into the session dict but does not change the server‑side uuid, consistent with the observed
result. The id was decoded with the same signer defined at `app/session.py:37-41`.

**Rationale.** Since login neither purges nor rotates the server‑side session id, the same
`slapp`/Redis identifier that existed for the anonymous visitor is retained after authentication;
only the dictionary contents change (an anonymous session simply becomes an authenticated one).
This means the framework does not perform session‑id regeneration on login.


---

## Q5 — Email forwarding header handling

**Question.** Given an incoming email with (a) a custom `X-` header, (b) a `Received` header, and
(c) a `Reply-To` header — which of these three make it through to the forwarded message and which
get stripped? Send an actual test email and examine the result.

**How it was run.** A crafted message was pushed through the real inbound entry point
`email_handler.handle(envelope, msg)` (`email_handler.py:1945`), with outgoing mail captured by the
`mail_sender` test capture facility (`store_emails_instead_of_sending` / `get_stored_emails`,
`app/mail_sender.py:102-109`). The incoming message carried a custom `X-Test-Custom: hello123`
header, a `Received:` header, and a `Reply-To:` header pointing to an address different from the
alias:

```python
from email.message import EmailMessage
from aiosmtpd.smtp import Envelope
msg = EmailMessage()
msg["From"] = "sender@external-example.test"
msg["To"] = alias.email
msg["Subject"] = "Q5 header fate test"
msg["X-Test-Custom"] = "hello123"
msg["Received"] = "from mx.external-example.test (...) by mailin; Wed, 01 Jul 2026 22:00:00 +0000"
msg["Reply-To"] = "original-replyto@some-other-domain.test"
envelope = Envelope(); envelope.mail_from = "sender@external-example.test"; envelope.rcpt_tos = [alias.email]
mail_sender.store_emails_instead_of_sending(True)
result = email_handler.handle(envelope, msg)
fwd = mail_sender.get_stored_emails()[0].msg
for k, v in fwd.items(): print(f"{k}: {v}")
```

**Observed output (verbatim) — the forwarded message's headers.**

```
########## email_handler.handle result (SMTP status) ##########
  result: 250 Message accepted for delivery  (status.E200 == 250 Message accepted for delivery )
  stored (forwarded) emails count: 1
  forward envelope_to (mailbox): q5_ucadpntk@mailbox.test

########## Q5: FORWARDED message headers (what actually goes out) ##########
  Subject: Q5 header fate test
  Content-Type: text/plain; charset="utf-8"
  Content-Transfer-Encoding: 7bit
  MIME-Version: 1.0
  X-SimpleLogin-Type: Forward
  X-SimpleLogin-EmailLog-ID: 407
  X-SimpleLogin-Envelope-From: sender@external-example.test
  X-SimpleLogin-Original-From: sender@external-example.test
  X-SimpleLogin-Envelope-To: icicle_phases012@sl.local
  Date: Wed, 01 Jul 2026 22:09:43 -0000
  From: "sender at external-example.test" <sender_at_external-example_test_qhajjllq@sl.local>
  Reply-To: "original-replyto at some-other-domain.test" <original-replyto_at_some-other-domain_test_bfwxwyu@sl.local>
  To: icicle_phases012@sl.local
  DKIM-Signature: v=1; a=rsa-sha256; c=relaxed/simple; d=sl.local; ...
```

```
########## Q5: FATE of the THREE named headers ##########
  (a) custom 'X-Test-Custom' (orig 'hello123'): present_in_forward=False value=[]  -> STRIPPED
  (b) 'Received' header:                        present_in_forward=False value=[]  -> STRIPPED
  (c) 'Reply-To' header: original='original-replyto@some-other-domain.test'  forwarded_value=['"original-replyto at some-other-domain.test" <original-replyto_at_some-other-domain_test_bfwxwyu@sl.local>']
       original Reply-To preserved? False  -> original STRIPPED, replaced by reverse-alias
```

Corroborating log line emitted during the rewrite (verbatim):

```
2026-07-01 22:09:43,869 - SL - DEBUG - 43984 - ".../email_handler.py:873" - forward_email_to_mailbox() -  - Reply-To header, new:"original-replyto at some-other-domain.test" <original-replyto_at_some-other-domain_test_bfwxwyu@sl.local>, old:None
```

**Answer — each of the three named headers by name.**

- **(a) Custom `X-Test-Custom` header → STRIPPED.** It does not appear in the forwarded message.
  Evidence: `(a) custom 'X-Test-Custom' (orig 'hello123'): present_in_forward=False value=[]  -> STRIPPED`
  (and it is absent from the forwarded header dump).
- **(b) `Received` header → STRIPPED.** It does not appear in the forwarded message. Evidence:
  `(b) 'Received' header: present_in_forward=False value=[]  -> STRIPPED`.
- **(c) `Reply-To` header → the ORIGINAL is STRIPPED, then a NEW `Reply-To` pointing to a
  reverse‑alias is added.** The forwarded `Reply-To` value is
  `"original-replyto at some-other-domain.test" <original-replyto_at_some-other-domain_test_bfwxwyu@sl.local>`,
  which is **not** the original `original-replyto@some-other-domain.test`. Evidence:
  `original Reply-To preserved? False  -> original STRIPPED, replaced by reverse-alias`, plus the
  `Reply-To header, new:… , old:None` log line — `old:None` shows the original header had already
  been removed by the allowlist step before the new one was added.

**Citations.** The allowlist `headers_to_keep` (`email_handler.py:793-807`) contains `FROM`, `TO`,
`CC`, `SUBJECT`, `DATE`, `MESSAGE_ID`, `REFERENCES`, `IN_REPLY_TO`, `SL_QUEUE_ID`,
`LIST_UNSUBSCRIBE`, `LIST_UNSUBSCRIBE_POST` plus `headers.MIME_HEADERS` (`email_handler.py:807`);
everything else is removed by `delete_all_headers_except(msg, headers_to_keep)`
(`email_handler.py:810`; definition `app/email_utils.py:536-542`). `Reply-To` and `Received` are
defined at `app/email/headers.py:13` and `app/email/headers.py:14` respectively and are **not** in
that allowlist; `MIME_HEADERS` (`app/email/headers.py:44-51`) is only `mime-version`,
`content-type`, `content-disposition`, `content-transfer-encoding`, so the custom `X-Test-Custom`
qualifies for neither list. The replacement `Reply-To` comes from the reply‑to contact created at
`email_handler.py:586-594` and re‑added via `add_or_replace_header(msg, "Reply-To", …)` at
`email_handler.py:872`, logged at `email_handler.py:873`. The forward completed with
`status.E200` = `250 Message accepted for delivery`.

**Rationale.** Forwarding rebuilds the outgoing message from a strict allowlist: any header not on
the list — including arbitrary `X-` headers and `Received` — is dropped. `Reply-To` is a special
case: the original is dropped with everything else, but because the incoming message had a
`Reply-To` differing from the alias, SimpleLogin creates a reverse‑alias contact and inserts a new
`Reply-To` so that replies route back through SimpleLogin rather than to the real address.


---

## Q6 — Alias creation (suffix) token expiration window

**Question.** Experimentally determine the expiry window: test a token immediately, then advance
time and re‑test until the boundary where it stops working.

**How it was run.** The app's own module‑level signer and wrapper were imported
(`from app.alias_suffix import signer, check_suffix_signature`). A suffix was signed at a fixed
base timestamp, unsigned immediately, then re‑tested at ages 599, 600, 601, and 700 seconds by
monkeypatching `itsdangerous.TimestampSigner.get_timestamp` (so the boundary is captured exactly
without waiting 600 s of wall‑clock):

```python
import itsdangerous
from app.alias_suffix import signer, check_suffix_signature
BASE = signer.get_timestamp()
itsdangerous.TimestampSigner.get_timestamp = lambda self: BASE
signed = signer.sign("test.suffix").decode()
for age in [599, 600, 601, 700]:
    itsdangerous.TimestampSigner.get_timestamp = lambda self, a=age: BASE + a
    try:    print(age, "VALID", signer.unsign(signed, max_age=600).decode())
    except itsdangerous.SignatureExpired as e: print(age, "SignatureExpired", e)
```

**Observed output (verbatim).**

```
########## Q6: alias suffix token expiry window ##########
itsdangerous version: 1.1.0
app signer class: TimestampSigner
CUSTOM_ALIAS_SECRET == FLASK_SECRET + 'custom_alias' -> 'secretcustom_alias'
SIGNED: test.suffix.akWQWA.H0_91wBbsxongROxEfdt7zRMlkQ
IMMEDIATE unsign(max_age=600): test.suffix
age=599s: VALID -> test.suffix
age=600s: VALID -> test.suffix
age=601s: SignatureExpired -> Signature age 601 > 600 seconds
age=700s: SignatureExpired -> Signature age 700 > 600 seconds
SignatureExpired subclass of BadSignature: True
wrapper check_suffix_signature at age=601: None
wrapper check_suffix_signature at age=599: test.suffix
```

**Answer.**

- **The expiry window is `600` seconds.** A freshly signed token unsigns immediately
  (`IMMEDIATE unsign(max_age=600): test.suffix`) and remains valid at age 599 and at exactly 600
  (`age=599s: VALID -> test.suffix`, `age=600s: VALID -> test.suffix`).
- **The boundary is at age > 600 s.** At age 601 the raw `unsign` raises
  `itsdangerous.SignatureExpired` with the message **`Signature age 601 > 600 seconds`**, and at
  700 s **`Signature age 700 > 600 seconds`**. Evidence:
  `age=601s: SignatureExpired -> Signature age 601 > 600 seconds` and
  `age=700s: SignatureExpired -> Signature age 700 > 600 seconds`.
- **The wrapper `check_suffix_signature` returns `None` at expiry** and returns the suffix when
  valid. Evidence: `wrapper check_suffix_signature at age=601: None` and
  `wrapper check_suffix_signature at age=599: test.suffix`. This is because `SignatureExpired`
  subclasses `BadSignature` (`SignatureExpired subclass of BadSignature: True`), which the wrapper
  catches.

**Citations.** `signer = itsdangerous.TimestampSigner(config.CUSTOM_ALIAS_SECRET)`
(`app/alias_suffix.py:11`) with `CUSTOM_ALIAS_SECRET = FLASK_SECRET + "custom_alias"`
(`app/config.py:201`) — confirmed at runtime as `'secretcustom_alias'`. `check_suffix_signature`
calls `signer.unsign(signed_suffix, max_age=600).decode()` (`app/alias_suffix.py:40`) and catches
`itsdangerous.BadSignature` to return `None` (`app/alias_suffix.py:41-42`).

**Rationale.** The suffix token is a `TimestampSigner` signature validated with `max_age=600`, so
it is accepted while its age is at most 600 seconds and rejected once the age exceeds 600. The raw
library call surfaces the expiry as `SignatureExpired` ("Signature age {age} > 600 seconds"), while
SimpleLogin's wrapper deliberately collapses any bad/expired signature to `None`.

---

## Q7 — API key usage statistics

**Question.** When making several API calls with the same key, what DB fields get updated and what
actual values are observed after the calls?

**How it was run.** The seeded `ApiKey` row (`id=91`) was read directly from PostgreSQL with raw
SQL **before** any call, then **N = 5** authenticated calls were made to `GET /api/user_info` with
the header `Authentication: <api_key.code>`, then the row was re‑read **after**:

```python
SELECT id, code, times, last_used, sudo_mode_at, created_at, updated_at FROM api_key WHERE id=91;
# ... 5x requests.get(".../api/user_info", headers={"Authentication": CODE}) ...
```

**Observed output (verbatim).**

```
----- BEFORE any API call (raw SQL SELECT) -----
  id = 91
  times = 0
  last_used = None
  sudo_mode_at = None
  created_at = datetime.datetime(2026, 7, 1, 22, 3, 25, 217502)
  updated_at = None

  per-call HTTP statuses: [200, 200, 200, 200, 200]

----- AFTER 5 API calls (raw SQL SELECT) -----
  id = 91
  times = 5
  last_used = datetime.datetime(2026, 7, 1, 22, 6, 2, 532290)
  sudo_mode_at = None
  created_at = datetime.datetime(2026, 7, 1, 22, 3, 25, 217502)
  updated_at = datetime.datetime(2026, 7, 1, 22, 6, 2, 532522)

----- FIELD-BY-FIELD TRANSITIONS -----
  times: 0  ->  5   [CHANGED]
  last_used: None  ->  datetime.datetime(2026, 7, 1, 22, 6, 2, 532290)   [CHANGED]
  sudo_mode_at: None  ->  None   [unchanged]
  created_at: datetime.datetime(2026, 7, 1, 22, 3, 25, 217502)  ->  datetime.datetime(2026, 7, 1, 22, 3, 25, 217502)   [unchanged]
  updated_at: None  ->  datetime.datetime(2026, 7, 1, 22, 6, 2, 532522)   [CHANGED]
  id: 91  ->  91   [unchanged]
```

**Answer — each field named.**

- **`times`: `0` → `5`.** It increments exactly once per call; with `N = 5` calls the value is
  `times = 5`. Evidence: `times: 0  ->  5   [CHANGED]` (and `per-call HTTP statuses: [200, 200, 200,
  200, 200]`).
- **`last_used`: `None` → `2026-07-01 22:06:02.532290`.** It moves from `None` to the timestamp of
  the latest call. Evidence: `last_used: None  ->  datetime.datetime(2026, 7, 1, 22, 6, 2, 532290)
  [CHANGED]`.
- **`updated_at`: `None` → `2026-07-01 22:06:02.532522`** (reported even though it is not in the
  question's obvious code path). Each call modifies the row, so the `ModelMixin` auto‑`onupdate`
  fires. Evidence: `updated_at: None  ->  datetime.datetime(2026, 7, 1, 22, 6, 2, 532522) [CHANGED]`.
- **`sudo_mode_at`: unchanged (`None`).** Evidence: `sudo_mode_at: None  ->  None   [unchanged]`.
- **`created_at` and `id` (and `code`): unchanged.** Evidence:
  `created_at: … [unchanged]`, `id: 91  ->  91   [unchanged]`.

**Citations.** When a valid API key is present, `authorize_request()` runs
`api_key.last_used = arrow.now()`, `api_key.times += 1`, `Session.commit()`
(`app/api/base.py:30-32`). The columns are defined on `ApiKey`
(`app/models.py:2350`): `last_used = sa.Column(ArrowType, default=None)` (`app/models.py:2358`),
`times = sa.Column(sa.Integer, default=0, nullable=False)` (`app/models.py:2359`), and
`sudo_mode_at = sa.Column(ArrowType, default=None)` (`app/models.py:2360`). `updated_at` changes
because `ApiKey` inherits `ModelMixin`, where `updated_at = sa.Column(ArrowType, default=None,
onupdate=arrow.utcnow)` (`app/models.py:65`); `created_at` (`app/models.py:64`) is not touched by
updates.

**Rationale.** Each authenticated API call explicitly stamps `last_used` and increments `times`,
then commits. Because the commit modifies the `api_key` row, SQLAlchemy's `onupdate` hook also
refreshes `updated_at` automatically. `sudo_mode_at` is only set by an explicit sudo‑mode action
(none was performed), so it stays `None`.


---

## Q8 — Failed login (wrong credentials)

**Question.** With wrong credentials, what specific log messages appear and what HTTP response
details come back?

**How it was run.** A `requests.Session` fetched `GET /auth/login` (for the CSRF token and session
cookie), then `POST /auth/login` with the seeded email but a deliberately wrong password. The full
response (status, headers, body) and the server stdout log delta were captured:

```python
r = s.post("http://127.0.0.1:7777/auth/login",
           data={"email": EMAIL, "password": "TOTALLY-WRONG-PASSWORD", "csrf_token": csrf},
           allow_redirects=False)
print(r.status_code); [print(k, v) for k, v in r.headers.items()]
print("Email or password incorrect" in r.text)
# + read the new bytes appended to the gunicorn stdout log
```

**Observed output (verbatim).**

```
########## Q8: POST /auth/login with WRONG password ##########
HTTP status_code: 200
Location header: None
----- FULL response headers -----
  Server: gunicorn/20.0.4
  Date: Wed, 01 Jul 2026 22:06:19 GMT
  Connection: close
  Content-Type: text/html; charset=utf-8
  Content-Length: 7290
  Set-Cookie: slapp=28bc2438-be46-441b-b8b1-2f97c2ab3008.vbifMMziVCEOSNapcym3R-0h6fQ; Expires=Wed, 08-Jul-2026 22:06:19 GMT; HttpOnly; Path=/; SameSite=Lax
----- body checks -----
  body length: 7290
  contains 'Email or password incorrect': True
  flash snippet: 'ger (red) -->                         <script>toastr.error("Email or password incorrect");</script> '
----- SERVER LOG delta during POST /auth/login (wrong password) -----
2026-07-01 22:06:19,366 - SL - DEBUG - 40982 - ".../server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 200, takes 0.24010515213012695
```

**Answer (reported exactly as observed).**

- **HTTP response details.**
  - **Status code is `200`** — the login template is re‑rendered, not an error status. Evidence:
    `HTTP status_code: 200` and `Location header: None`.
  - **`Content-Type: text/html; charset=utf-8`**, **`Content-Length: 7290`**, and a `Set-Cookie:
    slapp=…; HttpOnly; Path=/; SameSite=Lax` header. Evidence: the `FULL response headers` block.
  - **The body contains the flash message `Email or password incorrect`.** Evidence:
    `contains 'Email or password incorrect': True` and the flash snippet
    `<script>toastr.error("Email or password incorrect");</script>`.
- **Log messages.** Exactly **one** stdout log line was emitted for the request — the SimpleLogin
  `after_request` access line:
  `127.0.0.1 POST /auth/login ImmutableMultiDict([]) 200, takes 0.24010515213012695`. Evidence:
  the `SERVER LOG delta` block (a single line).
  - **This corrects the intuitive expectation of "no log line."** There is **no** Werkzeug access
    line (that logger is disabled, `app/log.py:70-71`) and **no** `LOG.*` line from the
    failed‑credentials branch itself (that branch only flashes and records a New Relic event); the
    one line that does appear comes from the global `after_request` handler
    (`server.py:272-296`, `LOG.d` at `server.py:284-292`), which logs every non‑static request.

**Citations.** The route `POST /auth/login` (`app/auth/views/login.py:21`) is rate‑limited
`@limiter.limit("10/minute", deduct_when=lambda r: hasattr(g, "deduct_limit") and g.deduct_limit)`
(`app/auth/views/login.py:22-24`) — the limit is only **deducted** on a failed attempt. The
failed‑credentials branch (`app/auth/views/login.py:45-50`) sets `g.deduct_limit = True`
(`app/auth/views/login.py:47`), clears the password (`app/auth/views/login.py:48`), calls
`flash("Email or password incorrect", "error")` (`app/auth/views/login.py:49`), and
`LoginEvent(LoginEvent.ActionType.failed).send()` (`app/auth/views/login.py:50`). `LoginEvent.send()`
records **only** a New Relic custom event (`app/events/auth_event.py:22-25`) — no stdout log. The
function then falls through to `render_template("auth/login.html", …)`
(`app/auth/views/login.py:74-82`), producing HTTP `200`. The Werkzeug access logger is disabled at
import (`app/log.py:70-71`); the observed line is the custom `after_request` logger
(`server.py:284`).

**Rationale.** A wrong password is treated as a normal (non‑exceptional) form outcome: SimpleLogin
flashes an error, records a New Relic event, deducts the rate‑limit token toward `10/minute`, and
re‑renders the login page with `200`. Because the failed branch has no `LOG.*` call and the
Werkzeug access logger is disabled, the only visible stdout log line is the generic
`after_request` access line for the request.

---

## Final coverage pass

Every distinct thing each question asks for, confirmed present above with verbatim evidence:

- **Q1** — HTTP status = `200` (evidenced); JSON body structure shown with all eight keys named
  (`can_create_reverse_alias`, `connected_proton_address`, `email`, `in_trial`, `is_premium`,
  `max_alias_free_plan`, `name`, `profile_picture_url`).
- **Q2** — status = `500` (explicitly **not** `401`/`403`/`440`); exact body
  `{"error":"Internal error"}`; the `LOG.e` `AttributeError: 'NoneType' object has no attribute
  'sudo_mode_at'` line and traceback — all with evidence.
- **Q3** — raw bytes shown (`bytes`, len `300`, leading `0x80` / `\x80\x04`); format stated as
  **pickle** (protocol 4); deserialized keys listed by name (`_fresh`, `_id`, `_permanent`,
  `_user_id`, `csrf_token`, `sudo_time`); key layout `session:<uuid>` shown; TTL `604800`.
- **Q4** — concrete BEFORE (`0a82aa68-c621-43cd-8934-7952635c3654`) and AFTER
  (`0a82aa68-c621-43cd-8934-7952635c3654`) identifier values shown; conclusion **unchanged**
  (`SAME identifier across login?: True`), reported as observed; data gains `_user_id`.
- **Q5** — all three named headers addressed by name: custom `X-Test-Custom` **stripped**,
  `Received` **stripped**, `Reply-To` **original stripped and replaced** by a reverse‑alias — each
  with evidence, plus the rewrite log line.
- **Q6** — exact window `600` seconds; success→expiry transition shown (valid at 599/600, expired
  at 601/700); exact message `Signature age {age} > 600 seconds`; wrapper returns `None` at expiry.
- **Q7** — each `ApiKey` field named: `times` (`0` → `5`, with N = 5 explicit), `last_used`
  (`None` → `2026-07-01 22:06:02.532290`), `sudo_mode_at` (**unchanged**, `None`), and `updated_at`
  (auto‑changed, `None` → `2026-07-01 22:06:02.532522`), with before/after DB values pasted.
- **Q8** — HTTP status = `200`; body contains `Email or password incorrect`; the actual observed
  log line reported (the single `after_request` access line) with the explicit note that the
  failed branch itself logs nothing and the Werkzeug access logger is disabled; rate limit
  `10/minute` noted.

Surprising results were reported exactly as observed: Q2's `500` via `AttributeError`, Q4's
preserved session identifier, Q7's auto‑changed `updated_at`, and Q8's single `after_request`
access line (rather than "no log line").

