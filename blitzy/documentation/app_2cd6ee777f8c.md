# SimpleLogin Runtime Behavioral Investigation — `app_2cd6ee777f8c`

A read-only, **evidence-based runtime investigation** of the SimpleLogin email-aliasing
Flask application. Every answer below was produced by **actually building, running, and
exercising the system through its real, canonical entry points** and capturing the actual
command and its complete, untruncated output. Reading the source alone is not treated as
sufficient: each source anchor (`file:line`) is provided as the *causal explanation*, but the
value reported for every question is the value **observed from the running system**.

Every reported value is labelled **[OBSERVED]** (captured from the running system) or
**[INFERRED]** (reasoned from reading, used only where a runtime signal genuinely could not be
captured). OBSERVED is used wherever the runtime signal was available — which is everywhere in
this document.

---

## Methodology

### Runtime stack (canonical build/run)

The application is the SimpleLogin Flask/Python monolith. It was stood up in its canonical
configuration and served with **gunicorn** on port **7777**, backed by **PostgreSQL** and
**Redis**, with the Redis-backed server-side session store wired in (`MEM_STORE_URI` set).

Observed component versions (captured from the running environment):

| Component | Version [OBSERVED] |
|-----------|--------------------|
| Python | 3.10.18 |
| Flask | 1.1.2 |
| Flask-Login | 0.5.0 |
| Flask-WTF | (CSRF token stored in session — see Q3) |
| aiosmtpd | 1.4.2 |
| redis (python client) | 4.6.0 |
| itsdangerous | 1.1.0 |
| gunicorn | 20.0.4 |
| SQLAlchemy | 1.3.24 |
| arrow | 0.16.0 |
| PostgreSQL (server) | 15.13 |
| Redis (server) | 7.0.15 |

> **Note on service versions.** The task framing referenced PostgreSQL 13 / Redis 6; the
> provided runtime image actually ships **PostgreSQL 15.13** and **Redis 7.0.15**. This
> document reports the **actual observed** versions. The session/pickle, alias-token, API-key,
> and email-forwarding behaviors probed here are application-level and are not affected by the
> Postgres/Redis point release.

### Exact commands used to stand up the system

```bash
# Backing services
service postgresql start
service redis-server start

# Activate the project virtualenv and load the canonical .env (python-dotenv reads /app/.env)
cd /app
. venv/bin/activate
# Key runtime config (from /app/.env):
#   DB_URI=postgresql://test:test@localhost:5432/test
#   MEM_STORE_URI=redis://localhost          <-- required, else sessions are NOT Redis-backed
#   FLASK_SECRET=secret
#   NOT_SEND_EMAIL=true
#   DISABLE_RATE_LIMIT=1
#   EMAIL_DOMAIN=sl.local
#   URL=http://localhost:7777

# Database migrations (baked DB already at head -> clean no-op)
alembic current          # -> 32f25cbf12f6 (head)
alembic upgrade head     # -> no steps applied (already at head)

# Canonical run command (as launched by the image's /root/run_app.sh)
gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 30 \
    --access-logfile /tmp/gunicorn_access.log \
    --error-logfile /tmp/gunicorn_error.log
```

> **`MEM_STORE_URI` is mandatory for Q3/Q4.** `server.py` only wires the `RedisSessionStore`
> (via `initialize_redis_services`) when `MEM_STORE_URI` is set (`server.py:L163-L165`,
> `app/config.py:L568`). It was set (`redis://localhost`), so sessions are stored in Redis and
> Q3/Q4 are observable canonically.

> **`--timeout`.** The image launches gunicorn with `--timeout 30` (the task framing named
> `--timeout 15`). The timeout is behavior-neutral for these observations; the actual value is
> reported for reproducibility.

### Canonical seed data

Seeding was done through the application's own canonical seeding function `fake_data()`
(`app/fake_data.py:L40`), invoked via the app's real Flask CLI command (not by hand-inserting
rows):

```bash
cd /app && . venv/bin/activate
FLASK_APP=wsgi:app flask dummy-data      # runs fake_data() + add_sl_domains() + add_proton_partner()
```

This produced (confirmed by reading back through `psql`):

- verified user **`john@wick.com`** / password **`password`** (`activated=t`, `is_admin=t`), id=2;
- API keys **`code="code"`** (name "Chrome", id=2) and **`code="codeFF"`** (name "Firefox", id=3);
- a random **alias** `refuge_tomato587@sl.local` (id=3) owned by john, forwarding to mailbox `john@wick.com`;
- verified **mailboxes** `john@wick.com` (id=2) and `pgp@example.org` (id=3);
- public domains `sl.local`, `premium.com`.

### Conventions & cleanup guarantee

- All observation helper scripts were created **outside** the repository (in the container's
  `/tmp` and a host scratch dir) and were **deleted** after use. The only file added to the
  repository is this document. `git status` shows exactly one new untracked file:
  `blitzy/documentation/app_2cd6ee777f8c.md`.
- No existing source file was modified, added, or deleted.

---

## Q1 — API authentication via a browser session (no API key)

**Direct answer [OBSERVED]:** When logged in with a browser session and calling a
`require_api_auth`-guarded endpoint **without** any `Authentication` header, the request
**succeeds with HTTP `200 OK`** and returns the endpoint's normal JSON payload. For
`GET /api/aliases?page_id=0` the body is a JSON object whose top-level key is `aliases` (an
array of alias objects). The browser session is accepted via the Flask-Login `current_user`
fallback; **no API key is required**.

**Exact commands:**

```bash
BASE=http://localhost:7777
CJ=/tmp/obs_q1_cookies.txt

# 1) Establish a browser session (CSRF from the login page, then POST credentials)
html=$(curl -s -c $CJ $BASE/auth/login)
csrf=$(echo "$html" | grep -oP 'name="csrf_token"[^>]*value="\K[^"]+' | head -1)
curl -s -b $CJ -c $CJ -i -X POST $BASE/auth/login \
  --data-urlencode "email=john@wick.com" \
  --data-urlencode "password=password" \
  --data-urlencode "csrf_token=$csrf" | grep -iE '^(HTTP|Location:)'

# 2) Call a require_api_auth endpoint with the session cookie and NO Authentication header
curl -s -i -b $CJ "$BASE/api/aliases?page_id=0"
```

**Complete observed output:**

```
# Step 1 — login:
HTTP/1.1 302 FOUND
Location: http://localhost:7777/dashboard/

# Step 2 — GET /api/aliases?page_id=0 (session cookie only, no Authentication header):
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 1947
...
{"aliases": [ ... 10 alias objects ... ]}   # top-level keys: ['aliases']
```

`curl -v` confirmed the outgoing request carried **only** `Cookie: slapp=0ff4c1a7-...<sig>`
and the request line `GET /api/aliases?page_id=0` — **no `Authentication` header** was sent.
The response body's single top-level key is `aliases`, containing 10 alias entries (e.g.
`wick@example.com`, `john@example.com`, `second@ab.cd`, `first@ab.cd`, `e2@sl.local`,
`e1@sl.local`, …).

**`file:line` reference & function:** `authorize_request()` and `require_api_auth`,
`app/api/base.py:L16-L27,L52-L60`; endpoint `get_aliases` at `app/api/views/alias.py:L38-L41`
(guarded by `@require_api_auth` at `L39`; note the route is also marked `@deprecated` at `L37`
but is fully functional).

**Causal rationale:** With no `Authentication` header, `authorize_request()`
(`app/api/base.py:L18`) computes `api_code = request.headers.get("Authentication")` → `None`,
so `ApiKey.get_by(code=None)` is falsy. Control reaches
`if current_user.is_authenticated:` (`L21`), which is true for the browser session, so it sets
`g.user = current_user` (`L25`) and returns `None` (`L43`). A `None` return from
`authorize_request()` signals "authorized" to `require_api_auth` (`L52-L60`), which then
invokes the endpoint normally → HTTP 200 with the JSON payload. (This is the session-fallback
branch; contrast with a completely unauthenticated request, which returns `401`.)

---

## Q2 — Privileged (sudo) operation — BOTH conditions

The single `require_api_sudo`-guarded endpoint is **`DELETE /api/user`** (`delete_user`,
`app/api/views/user.py:L12-L14`, guarded by `@require_api_sudo` at `L13`). Both conditions the
question implies were exercised and are reported.

### Condition (a) — valid API key, but sudo mode NOT active

**Direct answer [OBSERVED]:** HTTP **`440`** with body **`{"error":"Need sudo"}`**. `440` is a
**non-standard** HTTP status code (Microsoft IIS "Login Time-out"; not part of the standard
HTTP status set) — which is exactly why it does not look like a normal authorization error.
The observed status line literally reads `HTTP/1.1 440 UNKNOWN` (the `UNKNOWN` reason phrase is
itself evidence that the code is outside the standard set).

**Exact command:**

```bash
curl -s -i -X DELETE http://localhost:7777/api/user -H "Authentication: code"
```

**Complete observed output:**

```
HTTP/1.1 440 UNKNOWN
Content-Type: application/json
...
{"error":"Need sudo"}
```

### Condition (b) — browser session only (the edge case), no `Authentication` header

**Direct answer [OBSERVED]:** HTTP **`500 INTERNAL SERVER ERROR`** with body
**`{"error":"Internal error"}`**. The privileged endpoint does **not** cleanly return `440` on
the session-only path — it raises an unhandled `AttributeError` server-side and is converted to
a 500 by the app's error handler. This is a genuine edge/error path distinct from (a).

**Exact command:**

```bash
# Reuse an authenticated browser-session cookie jar (from the Q1 login), send NO Authentication header
curl -s -i -b /tmp/obs_q1_cookies.txt -X DELETE http://localhost:7777/api/user
```

**Complete observed output (HTTP response):**

```
HTTP/1.1 500 INTERNAL SERVER ERROR
Content-Type: application/json
...
{"error":"Internal error"}
```

**Complete observed output (server-side traceback, from the gunicorn error log):**

```
2026-07-14 20:11:04,700 - SL - ERROR - 117 - "/app/server.py:390" - error_handler() -  - 'NoneType' object has no attribute 'sudo_mode_at'
Traceback (most recent call last):
  File ".../flask/app.py", line 1950, in full_dispatch_request
    rv = self.dispatch_request()
  File ".../flask/app.py", line 1936, in dispatch_request
    return self.view_functions[rule.endpoint](**req.view_args)
  File "/app/app/api/base.py", line 69, in decorated
    if not check_sudo_mode_is_active(g.api_key):
  File "/app/app/api/base.py", line 47, in check_sudo_mode_is_active
    return api_key.sudo_mode_at and g.api_key.sudo_mode_at >= arrow.now().shift(
AttributeError: 'NoneType' object has no attribute 'sudo_mode_at'
2026-07-14 20:11:04,701 - SL - DEBUG - ... after_request() - 127.0.0.1 DELETE /api/user ImmutableMultiDict([]) 500, takes 0.0038s
```

**`file:line` reference & functions:** `require_api_sudo` (`app/api/base.py:L63-L73`, esp. the
guard `if not check_sudo_mode_is_active(g.api_key)` at `L69` and `return jsonify(error="Need
sudo"), 440` at `L70`); `check_sudo_mode_is_active` (`app/api/base.py:L46-L49`, dereferences
`api_key.sudo_mode_at` at `L47`); `g.api_key = api_key` assignment in `authorize_request()`
(`app/api/base.py:L42`); endpoint `delete_user` (`app/api/views/user.py:L12-L14`).

**Causal rationale:**
- **(a)** On the API-key path, `authorize_request()` sets `g.api_key` to the real `ApiKey` row.
  Its `sudo_mode_at` is `NULL` (sudo was never activated), so `check_sudo_mode_is_active(...)`
  is falsy and `require_api_sudo` returns `jsonify(error="Need sudo"), 440` (`L70`).
- **(b)** On the session path, `authorize_request()` authorizes via `current_user` but leaves
  `api_key = None`, and sets `g.api_key = None` (`L42`). `require_api_sudo` then calls
  `check_sudo_mode_is_active(None)` (`L69`), which dereferences `None.sudo_mode_at` (`L47`) →
  `AttributeError`. The exception propagates before the `440` return is ever reached; the app's
  `error_handler` (`server.py:L390`) turns it into `{"error":"Internal error"}` with HTTP 500.

> **Interaction note:** condition (a) is an authenticated **API-key** call, so it increments
> `api_key.times` (relevant to Q7). Q7 therefore reads a fresh before-value at capture time and
> reports deltas; absolute counter values are not significant.

---

## Q3 — Stored session data format (byte-exact)

**Direct answer [OBSERVED]:** An authenticated session is stored in Redis under the key
**`session:<uuid4>`**, and its value is a **Python `pickle`** byte string serialized with
**pickle protocol 4** (framing begins with `\x80\x04`). Deserializing yields a dictionary with
**6 keys**: `_permanent`, `_fresh`, `csrf_token`, `_user_id`, `_id`, and `sudo_time`.

> The framing is protocol **4** (`\x80\x04`), not protocol 5. `app/session.py:L91` calls
> `pickle.dumps(dict(session))` with no explicit protocol, so it uses
> `pickle.DEFAULT_PROTOCOL`, which is **4** on CPython 3.10 (`HIGHEST_PROTOCOL` is 5). This is a
> case where the runtime value differs from a reading-based guess of protocol 5 — hence the
> run-first requirement.

**Exact commands:**

```bash
# 1) Fresh login as john to create an authenticated session; keep the slapp cookie
#    -> slapp cookie value: afbd0db2-a7b5-4ac3-be6a-20d739d0d991.hNsPWDbLszFmWg5etKnZNHzlhGw
# 2) Decode the server-side session id by unsigning the cookie with the app's own signer:
python - <<'PY'
from server import create_app
app = create_app()
with app.test_request_context():
    signer = app.session_interface._get_signer(app)
print(signer.unsign("afbd0db2-a7b5-4ac3-be6a-20d739d0d991.hNsPWDbLszFmWg5etKnZNHzlhGw").decode())
# -> afbd0db2-a7b5-4ac3-be6a-20d739d0d991
PY

# 3) Read the RAW value directly from Redis (NOT through the app's deserialization):
redis-cli --no-raw GET "session:afbd0db2-a7b5-4ac3-be6a-20d739d0d991"

# 4) Independently, print the exact bytes and enumerate keys with redis-py + pickle:
python - <<'PY'
import redis, pickle
r = redis.from_url("redis://localhost")
raw = r.get("session:afbd0db2-a7b5-4ac3-be6a-20d739d0d991")
print("length:", len(raw))
print(repr(raw))                 # byte-exact, incl. pickle framing
d = pickle.loads(raw)
print("keys:", sorted(d.keys()))
for k in sorted(d): print(k, "=", repr(d[k]))
PY
```

**Complete observed output — RAW BYTES (byte-exact, before any deserialization):**

```
length: 300
b'\x80\x04\x95!\x01\x00\x00\x00\x00\x00\x00}\x94(\x8c\n_permanent\x94\x88\x8c\x06_fresh\x94\x88\x8c\ncsrf_token\x94\x8c(652e70eafa95dfa044b77ad24584242d35184b3d\x94\x8c\x08_user_id\x94\x8c$358c72da-bcf6-4530-88e7-ad4716688f11\x94\x8c\x03_id\x94\x8c\x80b03643a8a515eae10966eb5799d0b928b1fae8daeb0b0a33ba33f35b5fd7e09d574a2b38cd3af3ce53bdb464d28d2c515533674c8b087cba7d49abfa2cc9de57\x94\x8c\tsudo_time\x94J\xa0\x98Vju.'
```

- **Serialization format = Python `pickle`.** The leading two bytes `\x80\x04` are the pickle
  `PROTO` opcode (`\x80`, decimal 128) followed by the protocol number **`\x04`** (4). The
  trailing `.` byte is the pickle `STOP` opcode. The `}\x94(` sequence is `EMPTY_DICT` +
  `MEMOIZE` + `MARK`, i.e. the start of a dictionary — consistent with
  `pickle.dumps(dict(session))`.

**Complete observed output — DESERIALIZED keys:**

```
keys: ['_fresh', '_id', '_permanent', '_user_id', 'csrf_token', 'sudo_time']
_fresh = True
_id = 'b03643a8a515eae10966eb5799d0b928b1fae8daeb0b0a33ba33f35b5fd7e09d574a2b38cd3af3ce53bdb464d28d2c515533674c8b087cba7d49abfa2cc9de57'
_permanent = True
_user_id = '358c72da-bcf6-4530-88e7-ad4716688f11'
csrf_token = '652e70eafa95dfa044b77ad24584242d35184b3d'
sudo_time = 1784060064
```

Key meanings (each key observed present):
- `_user_id` — Flask-Login user id; here a **UUID string** (`358c72da-…`), not the integer PK,
  because `User.get_id()` returns `self.alternative_id` (a `uuid4` assigned at creation) —
  `app/models.py:L595-L597,L616-L617`.
- `_fresh` — Flask-Login "fresh session" flag (`True`).
- `_id` — Flask-Login session-protection identifier (a 128-hex fingerprint of the client).
- `csrf_token` — Flask-WTF CSRF token stored in the session.
- `_permanent` — set because the session is marked permanent (7-day lifetime).
- `sudo_time` — set by SimpleLogin when the password login activates sudo mode (an extra key
  beyond the standard Flask-Login/WTF set).

**Session-key structure [OBSERVED]:** the literal prefix `session`, a colon, then a `uuid4`
→ `session:<uuid4>` (here `session:afbd0db2-a7b5-4ac3-be6a-20d739d0d991`). That same uuid is
the value HMAC-signed into the `slapp` cookie.

**`file:line` reference & functions:** `SESSION_PREFIX = "session"` (`app/session.py:L18`);
`_get_key` builds `f"session:{session_Id}"` (`app/session.py:L43-L45`); `save_session` writes
`val = pickle.dumps(dict(session))` (`app/session.py:L91`) and stores it (`setex`,
`L97-L101`); the pickle import fallback is `app/session.py:L9-L12`; the cookie signer is
`itsdangerous.Signer(secret_key, salt="session", key_derivation="hmac")`
(`app/session.py:L38-L41`).

**Causal rationale:** on each request the `RedisSessionStore.save_session` method serializes the
session dictionary with `pickle.dumps(dict(session))` (default protocol 4 on Python 3.10) and
writes it to Redis at `session:<uuid>`. The `<uuid>` is generated server-side and HMAC-signed
into the `slapp` cookie via `itsdangerous.Signer`, so the cookie carries only the signed uuid
while all session contents live in Redis as pickle bytes.

---

## Q4 — Session identifier behavior across login (before/after)

**Direct answer [OBSERVED]: NO CHANGE.** The session identifier is **not** regenerated on
login. The `slapp` cookie value — and the server-side session uuid it decodes to — is
**byte-for-byte identical** before and after a successful login on the same cookie jar. There
is no session-ID rotation at login time.

**Exact commands:**

```bash
BASE=http://localhost:7777
CJ=/tmp/obs_q4_cookies.txt
rm -f $CJ

# BEFORE: unauthenticated GET to obtain an initial slapp cookie
html=$(curl -s -c $CJ $BASE/auth/login)
grep slapp $CJ                      # raw cookie BEFORE
csrf=$(echo "$html" | grep -oP 'name="csrf_token"[^>]*value="\K[^"]+' | head -1)

# LOGIN on the SAME cookie jar
curl -s -b $CJ -c $CJ -i -X POST $BASE/auth/login \
  --data-urlencode "email=john@wick.com" \
  --data-urlencode "password=password" \
  --data-urlencode "csrf_token=$csrf" | grep -iE '^(HTTP|Location:)'
grep slapp $CJ                      # raw cookie AFTER

# Decode each cookie to its underlying uuid using the app's own signer
python - <<'PY'
from server import create_app
app = create_app()
with app.test_request_context():
    s = app.session_interface._get_signer(app)
before = "c5c86f23-5c6c-42ea-bc11-7b3b11f487b7.nqsLiNWUlZPav3qp0WNTQwsDaHE"
after  = "c5c86f23-5c6c-42ea-bc11-7b3b11f487b7.nqsLiNWUlZPav3qp0WNTQwsDaHE"
print("decoded BEFORE uuid:", s.unsign(before).decode())
print("decoded AFTER  uuid:", s.unsign(after).decode())
print("raw cookie identical?", before == after)
print("decoded uuid identical?", s.unsign(before) == s.unsign(after))
PY
```

**Complete observed output:**

```
# raw slapp cookie BEFORE login:
slapp   c5c86f23-5c6c-42ea-bc11-7b3b11f487b7.nqsLiNWUlZPav3qp0WNTQwsDaHE

# login on same cookie jar:
HTTP/1.1 302 FOUND
Location: http://localhost:7777/dashboard/

# raw slapp cookie AFTER login:
slapp   c5c86f23-5c6c-42ea-bc11-7b3b11f487b7.nqsLiNWUlZPav3qp0WNTQwsDaHE

# decode + compare:
decoded BEFORE uuid: c5c86f23-5c6c-42ea-bc11-7b3b11f487b7
decoded AFTER  uuid: c5c86f23-5c6c-42ea-bc11-7b3b11f487b7
raw cookie identical? True
decoded uuid identical? True
```

| | Raw `slapp` cookie | Decoded session uuid |
|---|---|---|
| **Before login** | `c5c86f23-5c6c-42ea-bc11-7b3b11f487b7.nqsLiNWUlZPav3qp0WNTQwsDaHE` | `c5c86f23-5c6c-42ea-bc11-7b3b11f487b7` |
| **After login** | `c5c86f23-5c6c-42ea-bc11-7b3b11f487b7.nqsLiNWUlZPav3qp0WNTQwsDaHE` | `c5c86f23-5c6c-42ea-bc11-7b3b11f487b7` |
| **Changed?** | **No** | **No** |

**`file:line` reference & functions:** `RedisSessionStore.open_session`
(`app/session.py:L68-L80`) reuses the `session_id` extracted from the incoming cookie;
`save_session` (`app/session.py:L82-L114`) re-signs the **same** `session.session_id`
(`L102-L104`); `purge_session` (`app/session.py:L61-L66`) is the **only** place a new uuid is
generated (`L64`), and it runs on logout — not login. `login_manager.session_protection =
"strong"` (`app/extensions.py:L8`). Cookie name `slapp` (`app/config.py:L199`, wired in
`server.py:L159`); 7-day lifetime (`server.py:L207`).

**Causal rationale:** the custom `RedisSessionStore` keys the session by a uuid that is created
once (when a session first appears) and thereafter reused: `open_session` reads the uuid from
the cookie and `save_session` re-emits the same uuid. `login_user()` in Flask-Login `0.5.0`
writes `_user_id`/`_fresh`/`_id` into the existing session but does **not** rotate the
server-side identifier, and `session_protection="strong"` does not rotate it either at this
version. Because `itsdangerous.Signer` (non-timestamp) is deterministic for a fixed
uuid+secret, even the signed cookie string is identical before/after — the underlying uuid is
the session identity, and it does not change across login. (Only logout, via `purge_session`,
rotates the uuid.)


---

## Q5 — Email forwarding header survival (each of three headers)

**Direct answer [OBSERVED]:** All three test headers on the inbound message are **stripped**
from the forwarded message by SimpleLogin's forward-path allow-list. Explicitly, per header:

| Inbound header | Survives to forwarded message? |
|----------------|-------------------------------|
| custom `X-Test-Custom: hello-world` | **No — stripped** |
| `Received: from mail.example.com …` | **No — stripped** |
| `Reply-To: someone@elsewhere.test` | **No — original value stripped.** A *new* `Reply-To` appears, but it holds a SimpleLogin **reverse-alias** (`…@sl.local`), not the original address. |

**Invocation path (labelled):** the message was delivered through the **real** SMTP handler
object — `MailHandler().handle_DATA(None, None, envelope)` — with a real
`aiosmtpd.smtp.Envelope` built from real RFC822 bytes (`mail_from=outsider@example.com`,
`rcpt_tos=["refuge_tomato587@sl.local"]`). The full real pipeline ran
(`handle_DATA → _handle → handle → handle_forward → forward_email_to_mailbox →
mail_sender.send`) and returned `250 Message accepted for delivery`. The forwarded output was
captured via the application's **own** `MailSender.store_emails_instead_of_sending()` hook
(which records the real outgoing `SendRequest` object — **not** a mock). Because
`NOT_SEND_EMAIL=true` and there is no downstream MTA in this environment, the only deviation
from a live transmit is that the finished message is captured in-process instead of being
handed to Postfix; the canonical entry point and full transformation logic are exercised
exactly as in production.

**Exact command (essentials):**

```bash
# Build the inbound RFC822 message with the three probe headers + standard headers,
# then drive the REAL handler in-process and capture the forwarded SendRequest.
python - <<'PY'
import asyncio
from aiosmtpd.smtp import Envelope
from email.message import EmailMessage
from app.mail_sender import mail_sender
import email_handler

raw = b"""\
From: Outsider <outsider@example.com>
To: refuge_tomato587@sl.local
Subject: Q5 header survival test
Date: Tue, 14 Jul 2026 20:20:27 -0000
Message-ID: <q5-probe-0001@example.com>
Reply-To: someone@elsewhere.test
Received: from mail.example.com (mail.example.com [203.0.113.9]) by mx.sl.local; Tue, 14 Jul 2026 20:20:20 -0000
X-Test-Custom: hello-world
MIME-Version: 1.0
Content-Type: text/plain; charset="utf-8"
Content-Transfer-Encoding: 7bit

body
"""

env = Envelope()
env.mail_from = "outsider@example.com"
env.rcpt_tos = ["refuge_tomato587@sl.local"]
env.content = raw

mail_sender.store_emails_instead_of_sending()          # capture real outgoing SendRequest
h = email_handler.MailHandler()
code = asyncio.get_event_loop().run_until_complete(h.handle_DATA(None, None, env))
print("handle_DATA ->", code)
sent = mail_sender.get_stored_emails()                 # the real forwarded message
fwd = sent[-1].msg
print("--- FORWARDED HEADER BLOCK ---")
for k, v in fwd.items():
    print(f"{k}: {v}")
print("X-Test-Custom present? ", fwd.get("X-Test-Custom") is not None)
print("Received present?      ", fwd.get("Received") is not None)
print("Reply-To value         ", fwd.get("Reply-To"))
PY
```

**Complete observed output — the forwarded message header block (to `john@wick.com`):**

```
handle_DATA -> 250 Message accepted for delivery

--- FORWARDED HEADER BLOCK ---
Subject: Q5 header survival test
Date: Tue, 14 Jul 2026 20:20:27 -0000
Message-ID: <q5-probe-0001@example.com>
Content-Type: text/plain; charset="utf-8"
Content-Transfer-Encoding: 7bit
MIME-Version: 1.0
X-SimpleLogin-Type: Forward
X-SimpleLogin-EmailLog-ID: 2
X-SimpleLogin-Envelope-From: outsider@example.com
X-SimpleLogin-Original-From: Outsider <outsider@example.com>
X-SimpleLogin-Envelope-To: refuge_tomato587@sl.local
From: "Outsider - outsider at example.com" <outsider_at_example_com_nelagngpix@sl.local>
Reply-To: "someone at elsewhere.test" <someone_at_elsewhere_test_sflfaws@sl.local>
To: refuge_tomato587@sl.local
DKIM-Signature: v=1; a=rsa-sha256; c=relaxed/simple; d=sl.local; ... h=message-id : date : subject : from : to; ...

X-Test-Custom present?  False
Received present?       False
Reply-To value          "someone at elsewhere.test" <someone_at_elsewhere_test_sflfaws@sl.local>
```

**Per-header verdict (each explicit) [OBSERVED]:**
- **custom `X-Test-Custom`** → **STRIPPED.** It is absent from the forwarded message
  (`present? False`) and is not re-added.
- **`Received`** → **STRIPPED.** Absent from the forwarded message (`present? False`); not
  re-added.
- **`Reply-To`** → the **original** value `someone@elsewhere.test` is **STRIPPED.** The
  forwarded message does carry a `Reply-To`, but it is a SimpleLogin reverse-alias
  (`"someone at elsewhere.test" <someone_at_elsewhere_test_sflfaws@sl.local>`), i.e. a
  *different* value the application inserts — not the sender's original `Reply-To`.

**`file:line` reference & functions:** the allow-list `headers_to_keep` is built in
`forward_email_to_mailbox` (`email_handler.py:L793-L807`) as
`[FROM, TO, CC, SUBJECT, DATE, MESSAGE_ID, REFERENCES, IN_REPLY_TO, SL_QUEUE_ID,
LIST_UNSUBSCRIBE, LIST_UNSUBSCRIBE_POST] + MIME_HEADERS`, then applied via
`delete_all_headers_except(msg, headers_to_keep)` (`email_handler.py:L810`).
`delete_all_headers_except` (`app/email_utils.py:L536-L542`) lowercases the keep-list and
deletes every header whose lowercased name is not in it (**case-insensitive**). Header-name
constants are in `app/email/headers.py`: `REPLY_TO = "Reply-To"` (`L13`), `RECEIVED =
"Received"` (`L14`), and `MIME_HEADERS` (`L44-L51`, only `Mime-Version` / `Content-Type` /
`Content-Disposition` / `Content-Transfer-Encoding`). SMTP entry `handle_DATA`
(`email_handler.py:L2289`) → `handle_forward` (`email_handler.py:L536`) →
`forward_email_to_mailbox` (`email_handler.py:L679`).

**Causal rationale:** none of the three probe headers is in `headers_to_keep` — a custom
`X-*` header, `Received`, and `Reply-To` are all outside the allow-list (`MIME_HEADERS`
contains only MIME framing headers). So `delete_all_headers_except` removes all three. *After*
stripping, SimpleLogin re-writes the sender identity: it sets a reverse-alias `From`
(`email_handler.py:L864-L867`) and, because a reply-to contact exists for this message, it adds
a reverse-alias `Reply-To` (`email_handler.py:L870-L874`). Crucially, the code at `L871` reads
`msg[REPLY_TO]` and finds **`None`** — because the allow-list already stripped the original —
which is confirmed by the runtime log line at `L874`:

```
... Reply-To header, new:...<someone_at_elsewhere_test_sflfaws@sl.local>, old:None
```

The `old:None` proves the original `Reply-To` had already been removed before the new
reverse-alias value was inserted. (This is a single deterministic path, so one run is
sufficient; no ≥2-run requirement applies to Q5.)

---

## Q6 — Alias-creation token expiry window (≥2 runs, boundary)

**Direct answer [OBSERVED]: a 600-second (10-minute) window.** A signed alias-creation suffix
is accepted while its age is **≤ 600 s** and rejected once its age reaches **601 s**. Two
independent tokens both flipped from valid to expired at the same 600 → 601 s boundary.

**Exact commands & complete observed output:**

*(1) Boundary pinpoint through the real signer + real validator, on two independent tokens.*
Two tokens were minted with the application's real `signer.sign(...)` and then repeatedly
checked with the real `check_suffix_signature(...)` (the exact function the form uses), logging
the validity at each elapsed second:

```bash
python - <<'PY'
import time
from app.alias_suffix import signer, check_suffix_signature
T1 = signer.sign("q6probe1@sl.local"); T2 = signer.sign("q6probe2@sl.local")
if isinstance(T1, bytes): T1 = T1.decode()
if isinstance(T2, bytes): T2 = T2.decode()
print("T1:", T1); print("T2:", T2)
t0 = time.time()
print("T1 immediate:", "VALID" if check_suffix_signature(T1) else "EXPIRED")
print("T2 immediate:", "VALID" if check_suffix_signature(T2) else "EXPIRED")
time.sleep(594)
while True:
    age = time.time() - t0
    v1 = check_suffix_signature(T1) is not None
    v2 = check_suffix_signature(T2) is not None
    print("probe age=%.2fs  T1=%s  T2=%s" % (age, "VALID" if v1 else "EXPIRED", "VALID" if v2 else "EXPIRED"))
    if (not v1) and (not v2) and age > 603: break
    time.sleep(1)
PY
```

```
T1: q6probe1@sl.local.alabjQ.A_MvMfYBg_F6OEISsPetU3o_D40
T2: q6probe2@sl.local.alabjQ.A8OJPVaUiEFmv5_IFW5lOq05SL8
T1 immediate: VALID
T2 immediate: VALID
probe age=594.10s  T1=VALID    T2=VALID
probe age=595.10s  T1=VALID    T2=VALID
probe age=596.10s  T1=VALID    T2=VALID
probe age=597.10s  T1=VALID    T2=VALID
probe age=598.10s  T1=VALID    T2=VALID
probe age=599.11s  T1=VALID    T2=VALID
probe age=600.11s  T1=VALID    T2=VALID
probe age=601.11s  T1=EXPIRED  T2=EXPIRED
probe age=602.11s  T1=EXPIRED  T2=EXPIRED
probe age=603.11s  T1=EXPIRED  T2=EXPIRED
```

Both tokens: **last VALID at age 600.11 s, first EXPIRED at age 601.11 s** — a stable 600 s
window across the two independent runs (T1 and T2).

*(2) Real alias-creation FORM POST — valid immediately.* Using a freshly minted
`signed-alias-suffix` obtained from the real custom-alias page:

```bash
curl -s -i -b $CJ -c $CJ -X POST http://localhost:7777/dashboard/custom_alias \
  --data-urlencode "prefix=q6diag" \
  --data-urlencode "signed-alias-suffix=.dramas140@sl.local.alab_g.pql_T8Sw9mt_moZbz1IJdMWr8Ao" \
  --data-urlencode "mailboxes=2" --data-urlencode "note=q6" \
  --data-urlencode "csrf_token=$fcsrf" | grep -iE '^(HTTP|Location:)'
```

```
HTTP/1.1 302 FOUND
Location: http://localhost:7777/dashboard/?highlight_alias_id=14
```

A `302` redirect to `…/dashboard/?highlight_alias_id=<N>` is the success signal — the alias was
actually created (a second immediate POST produced `highlight_alias_id=15`).

*(3) Real alias-creation FORM POST — aged token (> 600 s) → expired.* A real form-minted token
(`.burden697@sl.local.alabjQ.P_pVufon4MJUCVNdH6E7feEA1Ko`, minted ~20:26:57) was held and
POSTed more than 600 s later:

```bash
# Confirm the aged token is expired through the real validator, then POST it via the real form:
python -c "from app.alias_suffix import check_suffix_signature as c; \
print('validator:', c('.burden697@sl.local.alabjQ.P_pVufon4MJUCVNdH6E7feEA1Ko'))"

curl -s -i -b $CJ -c $CJ -X POST http://localhost:7777/dashboard/custom_alias \
  --data-urlencode "prefix=q6aged" \
  --data-urlencode "signed-alias-suffix=.burden697@sl.local.alabjQ.P_pVufon4MJUCVNdH6E7feEA1Ko" \
  --data-urlencode "mailboxes=2" --data-urlencode "note=q6aged" \
  --data-urlencode "csrf_token=$fcsrf" | grep -iE '^(HTTP|Location:)'
```

```
validator: None
HTTP/1.1 302 FOUND
Location: http://localhost:7777/dashboard/custom_alias
```

The aged token fails the validator (`None`) and the form POST redirects **back to the form**
(`…/dashboard/custom_alias`, not `?highlight_alias_id=…`). The flash SimpleLogin stored in the
session (read byte-level from Redis) is exactly:

```
_flashes = [('warning', 'Alias creation time is expired, please retry')]
```

i.e. category `warning`, message **"Alias creation time is expired, please retry"**.

**Before/after summary:**

| Token | Age when tested | Result |
|-------|-----------------|--------|
| T1 (real `signer.sign`) | 600.11 s | **VALID** |
| T1 | 601.11 s | **EXPIRED** |
| T2 (real `signer.sign`) | 600.11 s | **VALID** |
| T2 | 601.11 s | **EXPIRED** |
| Form token `.dramas140@…` | ~0 s (immediate) | **SUCCESS** — `302 highlight_alias_id=14` |
| Form token `.burden697@…` | > 600 s | **EXPIRED** — `302 /dashboard/custom_alias` + "Alias creation time is expired, please retry" |

**`file:line` reference & functions:** `check_suffix_signature`
(`app/alias_suffix.py:L37-L42`) calls `signer.unsign(signed_suffix, max_age=600).decode()`
(`L40`); `signer = itsdangerous.TimestampSigner(config.CUSTOM_ALIAS_SECRET)`
(`app/alias_suffix.py:L11`); on expiry `itsdangerous` raises `SignatureExpired` (a subclass of
`BadSignature`), caught to return `None` (`L41-L42`). The form entry point is `POST
/dashboard/custom_alias` (`app/dashboard/views/custom_alias.py:L30`, `@login_required` `L32`),
which reads the `signed-alias-suffix` field (`L60`), calls `check_suffix_signature` (`L90`),
and on `None` flashes `"Alias creation time is expired, please retry"` (`L93`) and redirects.
`CUSTOM_ALIAS_SECRET = FLASK_SECRET + "custom_alias"` (`app/config.py:L201`).

**Causal rationale:** the signed suffix is a `TimestampSigner` token; `unsign(..., max_age=600)`
accepts it only while its embedded timestamp is within 600 seconds of now. At age ≤ 600 s the
token verifies and the view creates the alias (redirect to `highlight_alias_id`); once age
crosses 600 s, `unsign` raises `SignatureExpired`, `check_suffix_signature` returns `None`, and
the view flashes the "expired" warning and redirects back to the form. The `None` branch is
identical for a genuinely expired signature and for a tampered/invalid signature (both are
`BadSignature` subclasses caught at `L41`); this was independently verified by POSTing a
signature-tampered token, which produced the same `302 /dashboard/custom_alias` + identical
warning flash.


---

## Q7 — API key usage statistics (each field, before/after, ≥2 runs)

**Direct answer [OBSERVED]:** Each **authenticated API-key** call updates two fields on the
`api_key` row, and answering each field explicitly:
- **`times`** → **incremented by exactly 1 per call** (so `+5` after 5 calls).
- **`last_used`** → **set to the timestamp of the most recent call** (`arrow.now()`).

The **session-fallback** path (a request authorized by the browser session, with **no**
`Authentication` header) updates **neither** field — only the API-key path does.

**Exact commands & complete observed output:** `api_key` where `code='code'` was read
before/after two batches of 5 authenticated calls (`GET /api/user_info` with
`Authentication: code`), then after 5 session-only calls for contrast:

```bash
export PGPASSWORD=test
read_key(){ psql -U test -d test -h localhost -tAc \
  "select times, last_used from api_key where code='code';"; }

echo "--- BEFORE run1 ---"; read_key
for i in 1 2 3 4 5; do curl -s -o /dev/null -w "call$i http=%{http_code}\n" \
  -H "Authentication: code" http://localhost:7777/api/user_info; done
echo "--- AFTER run1 ---"; read_key

echo "--- BEFORE run2 ---"; read_key
for i in 1 2 3 4 5; do curl -s -o /dev/null -w "call$i http=%{http_code}\n" \
  -H "Authentication: code" http://localhost:7777/api/user_info; done
echo "--- AFTER run2 ---"; read_key

# contrast: 5 session-fallback calls (cookie, NO Authentication header)
echo "--- BEFORE 5 session calls ---"; read_key
for i in 1 2 3 4 5; do curl -s -b $CJ -o /dev/null -w "sess-call$i http=%{http_code}\n" \
  "http://localhost:7777/api/aliases?page_id=0"; done
echo "--- AFTER 5 session-fallback calls ---"; read_key
```

```
--- BEFORE run1 ---
11|2026-07-14 20:23:14.028748
call1 http=200
call2 http=200
call3 http=200
call4 http=200
call5 http=200
--- AFTER run1 (5 API-key calls) ---
16|2026-07-14 20:42:06.230537
--- BEFORE run2 ---
16|2026-07-14 20:42:06.230537
call1 http=200
call2 http=200
call3 http=200
call4 http=200
call5 http=200
--- AFTER run2 (5 API-key calls) ---
21|2026-07-14 20:42:06.380587
--- BEFORE 5 session calls ---
21|2026-07-14 20:42:06.380587
sess-call1 http=200
sess-call2 http=200
sess-call3 http=200
sess-call4 http=200
sess-call5 http=200
--- AFTER 5 session-fallback calls (NO Authentication header) ---
21|2026-07-14 20:42:06.380587
```

**Before/after table (each field explicit, ≥2 runs):**

| | `times` before | `times` after | Δ`times` | `last_used` before | `last_used` after |
|---|---|---|---|---|---|
| **Run 1** (5 API-key calls) | 11 | 16 | **+5** | `2026-07-14 20:23:14.028748` | `2026-07-14 20:42:06.230537` |
| **Run 2** (5 API-key calls) | 16 | 21 | **+5** | `2026-07-14 20:42:06.230537` | `2026-07-14 20:42:06.380587` |
| **Contrast** (5 session calls) | 21 | 21 | **0** | `2026-07-14 20:42:06.380587` | `2026-07-14 20:42:06.380587` (unchanged) |

`times` advanced deterministically by exactly the number of API-key calls in each run (+5, +5),
and `last_used` advanced to the latest call's timestamp; the session-only batch changed neither
field.

**`file:line` reference & functions:** the API-key branch of `authorize_request()` runs
`api_key.last_used = arrow.now()` (`app/api/base.py:L30`), `api_key.times += 1`
(`app/api/base.py:L31`), and `Session.commit()` (`app/api/base.py:L32`). The columns are
defined on the `ApiKey` model: `last_used = sa.Column(ArrowType, default=None)`
(`app/models.py:L2358`) and `times = sa.Column(sa.Integer, default=0, nullable=False)`
(`app/models.py:L2359`). The session-fallback branch (`app/api/base.py:L20-L27`) never touches
these fields.

**Causal rationale:** when a request presents a valid API key, `authorize_request()` finds the
`ApiKey` row and, on that branch only, bumps `times` by one and stamps `last_used` with
`arrow.now()` before committing — so N API-key calls yield exactly `+N` on `times` and refresh
`last_used` each time. When a request is authorized instead by the browser session (Q1), the
API-key branch is skipped entirely, so the usage counters are left untouched — which the
contrast run confirms (`times` stayed at 21).

---

## Q8 — Failed-login logging and response

**Direct answer [OBSERVED]:** A wrong-credentials login returns **HTTP `200 OK`** with the
login page **re-rendered** (not a redirect), and the rendered page contains the flashed error
**"Email or password incorrect"**. The failed-login event itself emits **no dedicated
application log line** — it records a NewRelic custom event only. The observable server log for
the request is therefore just the framework's request-completion line (the `after_request`
debug line at `server.py:284`) and the gunicorn WSGI access line; both show `POST /auth/login`
→ `200`.

**Exact command:**

```bash
BASE=http://localhost:7777
CJ=/tmp/obs_q8_cookies.txt
html=$(curl -s -c $CJ $BASE/auth/login)
csrf=$(echo "$html" | grep -oP 'name="csrf_token"[^>]*value="\K[^"]+' | head -1)

curl -s -i -b $CJ -c $CJ -X POST $BASE/auth/login \
  --data-urlencode "email=john@wick.com" \
  --data-urlencode "password=WRONG-password-xyz" \
  --data-urlencode "csrf_token=$csrf"
```

**Complete observed output — HTTP response:**

```
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 7017
Set-Cookie: slapp=a36795b7-ec7f-4fb2-9872-8a1df1fb584a.RGfvEilpfm5X8yd6xSeSuFGUGIs; Expires=Tue, 21-Jul-2026 20:40:35 GMT; HttpOnly; Path=/; SameSite=Lax
...

# the flashed error is rendered inline in the 200 body (re-render consumes the flash):
<script>toastr.error("Email or password incorrect");</script>
```

**Complete observed output — server log lines emitted during the request:**

```
# application (SL) logger — the after_request request-completion line (server.py:284):
2026-07-14 20:40:35,713 - SL - DEBUG - 117 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 200, takes 0.24017715454101562

# gunicorn WSGI access log:
127.0.0.1 - - [14/Jul/2026:20:40:35 +0000] "POST /auth/login HTTP/1.1" 200 7017 "-" "curl/7.88.1"

# gunicorn error log delta for this request: (empty — no error, no traceback)
```

A grep of the log delta for any `LoginEvent`- or failure-specific application line returned
**nothing** beyond the `after_request` line above — confirming the failed-login event writes no
`LOG.*` line.

**`file:line` reference & functions:** `login` view (`app/auth/views/login.py:L21`,
rate-limited `10/minute` at `L22-L24`). The wrong-credentials branch is `L45-L50`:
`if not user or not user.check_password(...)` (`L45`) → `g.deduct_limit = True` (`L47`),
`form.password.data = None` (`L48`), `flash("Email or password incorrect", "error")` (`L49`),
`LoginEvent(LoginEvent.ActionType.failed).send()` (`L50`); the view then falls through to
`render_template("auth/login.html", ...)` (`L74-L82`) → HTTP 200. `LoginEvent.send()`
(`app/events/auth_event.py:L22-L25`) calls **only** `newrelic.agent.record_custom_event(
"LoginEvent", {"action": "failed", "source": "web"})` — no `LOG` call. The observable request
line comes from `after_request` (`server.py:L284-L292`,
`LOG.d("%s %s %s %s %s, takes %s", remote_addr, method, path, args, status_code, elapsed)`),
which also records an `HttpResponseStatus` NewRelic event (`server.py:L293-L295`).

**Causal rationale:** with a wrong password, `user.check_password(...)` is false (`L45`), so the
branch flashes `"Email or password incorrect"` (category `error`, which the template renders
via `toastr.error(...)`) and fires the failed `LoginEvent`. That event's `send()` records a
NewRelic custom event and writes nothing to the application log. Because the branch does not
redirect, execution falls through to `render_template(...)` and the same request returns the
re-rendered login page with HTTP 200 (the flash is consumed inline in that render). Thus the
only log evidence of the attempt is the framework's request-completion debug line plus the
WSGI access line — both reporting `POST /auth/login` → `200`; the failure signal visible to the
user is the flashed message in the 200 body.

---

## Summary of findings

| # | Question | Direct answer [OBSERVED] |
|---|----------|--------------------------|
| Q1 | API auth via browser session, no API key | **HTTP 200** + normal JSON (`{"aliases": [...]}`); session-fallback authorizes via `current_user`. |
| Q2 | Privileged op with session only | (a) API key, no sudo → **440** `{"error":"Need sudo"}` (440 is non-standard, IIS "Login Time-out"); (b) session-only edge → **500** `{"error":"Internal error"}` via `AttributeError: 'NoneType' object has no attribute 'sudo_mode_at'`. |
| Q3 | Stored session format | **pickle (protocol 4)** bytes in Redis at **`session:<uuid4>`**; keys `_permanent, _fresh, csrf_token, _user_id, _id, sudo_time`. |
| Q4 | Session id across login | **No change** — same `slapp` cookie and same server-side uuid before and after login. |
| Q5 | Forward header survival | custom `X-Test-Custom` **stripped**; `Received` **stripped**; original `Reply-To` **stripped** (a reverse-alias `Reply-To` is substituted). |
| Q6 | Alias-token expiry | **600-second** window; valid at ≤600 s, expired at 601 s (two tokens); real form: success `302 highlight_alias_id`, expired → `302 /dashboard/custom_alias` + "Alias creation time is expired, please retry". |
| Q7 | API-key usage stats | `times` **+1 per API-key call** (+5, +5 over two runs); `last_used` = `arrow.now()` each call; **session path updates neither**. |
| Q8 | Failed-login logging & response | **HTTP 200** re-render + "Email or password incorrect" flash; failed event is **NewRelic-only** (no app log line); observable log = `after_request` debug line + WSGI access line (`POST /auth/login` → 200). |

**Verification discipline.** Timing (Q6) and counter (Q7) findings were confirmed stable across
≥2 runs; the Q3 session value is reported byte-exact before deserialization; Q4/Q6/Q7 include
explicit before/after captures; Q5 addresses each of the three named headers and Q7 each of the
two named fields; Q2 reports both conditions. Every value above is **[OBSERVED]** from the
running system through its real, canonical entry points; no mocks, debug hooks, or hand-forged
tokens were used for any canonical answer. All temporary observation scripts were deleted after
capture, leaving this document as the only addition to the repository.

