# SimpleLogin Behavioral Investigation — Answers (branch `app_2cd6ee777f8c`)

This document answers eight precise behavioral questions about the **SimpleLogin**
(Python/Flask email‑aliasing) codebase. Each answer is grounded in **evidence captured
from the live, running system** and corroborated against the **source code as the
ground truth**, with inline `path:line` citations. The investigation is strictly
**read‑only**: no source file was modified, and every helper/probe script created to
observe behavior was deleted after use — this Markdown document is the only artifact
that persists.

- **Repository commit:** `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c` (branch `app_2cd6ee777f8c`).
- **Method in one line:** stand up the app in the prescribed container, seed it with
  `flask dummy-data`, drive a disposable probe per question, capture the **actual**
  runtime artifact (status code, raw bytes, before/after values, DB row, log line),
  then reconcile it with the implementing code.

---

## Environment & Method

### Runtime
All evidence was gathered inside the prescribed container image
`andrewparkscaleai/coding-agent:simple-login__app__2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`
(from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`). It bundles
**Python 3.10**, the Poetry‑managed dependencies, **Postgres**, and **Redis**. The app
targets Python 3.10 with pinned legacy dependencies (Flask 1.1.2, itsdangerous 1.1.0,
redis 4.6.0, SQLAlchemy 1.3.24, Werkzeug 1.0.1, flask‑login 0.5.0); the project notes
that Python 3.12 does not work (`CONTRIBUTING.md:L236`), so the container's provisioned
environment was used as‑is.

Inside the container the application binds to `127.0.0.1:7777`, so every HTTP probe was
issued from within the container (`docker exec … curl http://127.0.0.1:7777/…`).

### Configuration & startup
The runtime `.env` was created from `example.env` and left with its defaults. The
values relevant to this investigation:

| Key | Value | Source | Why it matters |
|-----|-------|--------|----------------|
| `URL` | `http://localhost:7777` | `example.env:L6` | Web base URL |
| `NOT_SEND_EMAIL` | `true` | `example.env:L19` | Outbound mail is not dispatched (critical for Q5) |
| `EMAIL_DOMAIN` | `sl.local` | `example.env:L22` | Alias domain |
| `DB_URI` | `postgresql://myuser:mypassword@localhost:5432/simplelogin` | `example.env:L75` | Dev database |
| `FLASK_SECRET` | `secret` | `example.env:L77` | Base secret; session cookie + derived secrets |
| `MEM_STORE_URI` | `redis://localhost` | (runtime `.env`) | Enables the Redis server‑side session store |

Startup sequence (per `CONTRIBUTING.md:L106` and the registered CLI command at
`server.py:L490`):

```bash
cp example.env .env            # NOT_SEND_EMAIL=true is kept
alembic upgrade head           # apply migrations (77 tables)
flask dummy-data               # seed fixtures (server.py:L490)
python3 server.py              # web app on 127.0.0.1:7777
python3 email_handler.py       # inbound SMTP (aiosmtpd) on 0.0.0.0:20381 — for Q5
```

Both services were confirmed running: the web app answered an authenticated API call
with HTTP 200, and the SMTP handler announced `220 … Python SMTP 1.4.2` on port 20381.

### Seeded fixtures (from `app/fake_data.py`)
- Account **`john@wick.com` / `password`**, `activated=True`, `is_admin=True`
  (`app/fake_data.py:L45,L47,L48,L49`).
- API keys: **`Chrome`** with code **`code`** (`app/fake_data.py:L121-L122`) and
  **`Firefox`** with code **`codeFF`** (`app/fake_data.py:L124-L125`). These are sent in
  the `Authentication` request header.
- Forwarding alias **`e1@sl.local`** → mailbox `john@wick.com` (used for Q5).

### Cleanup
All probe scripts and capture files (`curl` cookie jars, a `redis-cli`/pickle reader, an
in‑process email‑forward driver, two token‑timing probers, and `psql` queries) were
created under a throwaway directory and **deleted after evidence capture**. The seeded
dev database and Redis were mutated only transiently by the probes; no repository file
was touched.

---

## Q1 — API call WITHOUT an API key while logged in

### Question (verbatim)
> "When logged into the web interface and making API calls WITHOUT an API key header,
> requests seem to work. Confirm the ACTUAL HTTP status code returned and the response
> body structure."

### Method / commands
Log in through the browser flow as `john@wick.com` to obtain the `slapp` session cookie,
then call a `require_api_auth` endpoint (`GET /api/user_info`) sending **only the cookie**
— no `Authentication` header. The no‑auth case is captured for contrast.

```bash
# 1) GET /auth/login -> capture CSRF token + cookie jar
# 2) POST /auth/login (email, password, csrf_token) -> 302 to /dashboard/  (jar_after.txt)
# 3) session-only API call (cookie only, NO Authentication header):
curl -i -b jar_after.txt http://127.0.0.1:7777/api/user_info
# contrast: no cookie, no header
curl -i http://127.0.0.1:7777/api/user_info
```

### Captured runtime evidence
Session‑only call (cookie, no API key) — **HTTP 200**:

```text
HTTP/1.0 200 OK
{
  "can_create_reverse_alias": true,
  "connected_proton_address": null,
  "email": "john@wick.com",
  "in_trial": false,
  "is_premium": true,
  "max_alias_free_plan": 5,
  "name": "John Wick",
  "profile_picture_url": "http://localhost:7777/static/upload/profile_pic.svg"
}
```

No cookie and no header — **HTTP 401**:

```text
HTTP/1.0 401 UNAUTHORIZED
{
  "error": "Wrong api key"
}
```

### Code citation(s)
- `app/api/base.py:L16-L43` — `authorize_request()`. `L17` reads the `Authentication`
  header; `L18` looks up `ApiKey.get_by(code=…)`; `L20-L25` falls back to the Flask‑Login
  `current_user` when no key is present (`g.user = current_user`); `L26-L27` returns
  `jsonify(error="Wrong api key"), 401` when there is **neither** a key **nor** an
  authenticated session.
- `app/api/views/user_info.py:L50-L52` — `GET /api/user_info` guarded by
  `@require_api_auth`; `L28-L47` builds the response dict; `L65-L67` returns
  `jsonify(user_to_dict(user))`.

### Definitive answer
A session‑authenticated API call **succeeds with HTTP 200** and returns the endpoint's
normal JSON body — the API key is **not required** when a valid browser session exists.
The observed body for `GET /api/user_info` contained eight keys: `name`, `is_premium`,
`email`, `in_trial`, `max_alias_free_plan`, `connected_proton_address`,
`can_create_reverse_alias`, and `profile_picture_url`. With neither a key nor a session,
the response is **HTTP 401** with body `{"error": "Wrong api key"}`.

### Rationale
`authorize_request()` treats a logged‑in `current_user` as authorization: when the
`Authentication` header is absent (`api_key` is falsy), it checks
`current_user.is_authenticated` and, if true, sets `g.user = current_user` and returns
`None` (no error). The `require_api_auth` decorator therefore lets the view run normally,
so the browser session alone is sufficient. Only when there is no key *and* no
authenticated session does the function short‑circuit with `401 {"error":"Wrong api key"}`.

---

## Q2 — Privileged operation with only a browser session → non‑standard status code

### Question (verbatim)
> "Accessing privileged operations with just a browser session returns a NON-STANDARD
> authorization status code with an UNUSUAL error message. Determine the specific status
> code and EXACT error message text actually returned."

### Method / commands
The privileged (sudo‑gated) endpoint is `DELETE /api/user` (`app/api/views/user.py:L12-L16`,
"Delete the user. Requires sudo mode."). Two paths were exercised, plus the sudo‑entry
endpoint for context. Before probing, `sudo_mode_at` was confirmed `NULL` for both API
keys and `users.delete_on` was `NULL`, so the request short‑circuits **before** any
deletion occurs.

```bash
# (a) Clean path: valid API key whose sudo mode is inactive
curl -i -X DELETE -H "Authentication: code" http://127.0.0.1:7777/api/user
# (b) Pure browser-session path: only the slapp cookie, no API key
curl -i -X DELETE -b jar_after.txt http://127.0.0.1:7777/api/user
# (c) Context: entering sudo requires a password re-check
curl -i -X PATCH -H "Authentication: code" -H "Content-Type: application/json" \
     -d '{"password":"wrongpass"}' http://127.0.0.1:7777/api/sudo
```

### Captured runtime evidence
(a) Clean path — valid key, sudo inactive — **HTTP 440**:

```text
HTTP/1.0 440 UNKNOWN
{
  "error": "Need sudo"
}
```

(b) Pure browser‑session path — **HTTP 500** (not a clean 440):

```text
HTTP/1.0 500 INTERNAL SERVER ERROR
{
  "error": "Internal error"
}
```

Server log for the 500 (root cause):

```text
SL - ERROR - "/app/server.py:390" - error_handler() - 'NoneType' object has no attribute 'sudo_mode_at'
Traceback (most recent call last):
    if not check_sudo_mode_is_active(g.api_key):
  File "/app/app/api/base.py", line 47, in check_sudo_mode_is_active
    return api_key.sudo_mode_at and g.api_key.sudo_mode_at >= arrow.now().shift(
AttributeError: 'NoneType' object has no attribute 'sudo_mode_at'
```

(c) `PATCH /api/sudo` with a wrong/missing password — **HTTP 403** `{"error":"Invalid password"}`.

### Code citation(s)
- `app/api/base.py:L63-L72` — `require_api_sudo`; `L69-L70` returns
  `jsonify(error="Need sudo"), 440` when sudo is not active.
- `app/api/base.py:L46-L49` — `check_sudo_mode_is_active(api_key)` evaluates
  `api_key.sudo_mode_at and g.api_key.sudo_mode_at >= …`; `L13` defines
  `SUDO_MODE_MINUTES_VALID = 5`.
- `app/api/base.py:L42` — `g.api_key = api_key`; on the pure‑session path `api_key` is
  `None`, so `check_sudo_mode_is_active(None)` dereferences `None.sudo_mode_at` and raises
  `AttributeError` (surfacing as HTTP 500).
- `app/api/views/user.py:L12-L16` — the sudo‑gated `DELETE /api/user`.
- `app/api/views/sudo.py:L8-L27` — `PATCH /api/sudo` re‑checks the password; `L19-L22`
  returns `403 {"error":"Invalid password"}`; `L24` sets `g.api_key.sudo_mode_at = arrow.now()`.

### Definitive answer
The non‑standard status code is **HTTP 440** and the exact body is
**`{"error": "Need sudo"}`** (observed status line: `440 UNKNOWN`, because Werkzeug has no
registered reason phrase for 440). **HTTP 440 is not an IETF standard code** — it
originates as a Microsoft IIS "Login Time‑out" code signalling that the session has
expired and re‑authentication is required; SimpleLogin repurposes it to mean "sudo mode
required," where a conventional API would use 401 or 403.

There is an important nuance, confirmed at runtime: the clean 440 is produced by a
request that carries a **valid API key whose sudo mode is inactive**. A **pure
browser‑session** request to the same endpoint instead returns **HTTP 500**, because
`g.api_key` is `None` and `check_sudo_mode_is_active(None)` raises an `AttributeError` on
`None.sudo_mode_at` (see the captured traceback pointing at `app/api/base.py` line 47).

### Rationale
`require_api_sudo` proceeds only when `check_sudo_mode_is_active` is truthy; otherwise it
short‑circuits with `440 {"error":"Need sudo"}`. Sudo mode is entered via
`PATCH /api/sudo` (a password re‑check) and is valid for `SUDO_MODE_MINUTES_VALID = 5`
minutes. The pure‑session 500 is a direct consequence of the dual‑auth design: a
session‑only caller is authorized (`g.user` is set) but has no API key, so `g.api_key`
is `None`, and the sudo check is written to dereference an `ApiKey` attribute rather than
to tolerate `None`.

---

## Q3 — Session storage format (examine raw bytes in the storage backend)

### Question (verbatim)
> "Examine stored session data directly in the storage backend. Capture raw bytes,
> identify their format, list the keys present in the deserialized data for an
> authenticated session, and describe how the session key itself is structured."

### Method / commands
After logging in (Q1), read the authenticated session straight out of Redis, decode it
with Python `pickle`, and unsign the `slapp` cookie to recover the raw identifier.

```bash
redis-cli KEYS "session:*"
redis-cli --no-raw GET "session:8bb4e244-e110-47d9-a486-e0ea2510386c"
redis-cli TTL "session:8bb4e244-e110-47d9-a486-e0ea2510386c"
# then, in a small disposable Python reader:
#   raw = redis.from_url("redis://localhost").get("session:<uuid>")
#   pickle.loads(raw); list(d.keys())
#   itsdangerous.Signer("secret", salt="session", key_derivation="hmac").unsign(<slapp cookie>)
```

### Captured runtime evidence
Redis key shape and raw value (note the leading bytes):

```text
key:  session:8bb4e244-e110-47d9-a486-e0ea2510386c
raw:  "\x80\x04\x95!\x01\x00\x00\x00\x00\x00\x00}\x94(\x8c\n_permanent\x94\x88
       \x8c\x06_fresh\x94\x88\x8c\ncsrf_token\x94\x8c(145902ba8815b5ae19b5a61235e2…
       \x8c\b_user_id\x94\x8c$ffde7011-f2da-43dc-a793-07aa8a267055\x94 …
       \x8c\x03_id\x94\x8c\x80b03643a8a515eae10966eb5799d0b928b1fae8da… \x8c\tsudo_time\x94J…u."
len:  300 bytes
```

Programmatic decode (disposable Python reader):

```text
first 8 bytes (repr): b'\x80\x04\x95!\x01\x00\x00\x00'
pickle PROTO opcode byte0 = 0x80, protocol version = 4
starts with PHP "a:N:{" ? : False
deserialized type: dict
KEYS: ['_fresh', '_id', '_permanent', '_user_id', 'csrf_token', 'sudo_time']
  '_permanent': True
  '_fresh':     True
  'csrf_token': '145902ba8815b5ae19b5a61235e233718d739113'
  '_user_id':   'ffde7011-f2da-43dc-a793-07aa8a267055'
  '_id':        'b03643a8a515eae10966eb5799d0b928b1fae8da…'   (128-hex Flask-Login fingerprint)
  'sudo_time':  1782510015

slapp cookie unsign -> '8bb4e244-e110-47d9-a486-e0ea2510386c'  (== redis key suffix: True)
TTL of authenticated session: 604767 seconds  (~7 days)
```

### Code citation(s)
- `app/session.py:L43-L45` — `_get_key` → `f"{SESSION_PREFIX}:{session_Id}"`, with
  `SESSION_PREFIX = "session"` (`app/session.py:L18`) and the identifier minted as
  `str(uuid.uuid4())` (`app/session.py:L71,L80`).
- `app/session.py:L91` — value is written with `pickle.dumps(dict(session))`; read back
  with `pickle.loads(val)` (`app/session.py:L76`); the import is `cPickle`/`pickle`
  (`app/session.py:L9-L12`).
- `app/session.py:L38-L41` — the `slapp` cookie carries the identifier signed with
  `itsdangerous.Signer(app.secret_key, salt="session", key_derivation="hmac")`, signed at
  `L102-L104`. Cookie name `slapp` is `app/config.py:L199`.
- `server.py:L207` — `permanent_session_lifetime = timedelta(days=7)` (the observed
  ~604,800 s TTL); `app/session.py:L92-L96` caps sessions **without** `_user_id` at 300 s.
- Wiring: `app/redis_services.py:L12` sets `app.session_interface = RedisSessionStore(...)`
  via `initialize_redis_services`, called from `server.py:L165` (guarded by `MEM_STORE_URI`
  at `server.py:L163`).
- `app/paddle_utils.py:L51` — the only use of `phpserialize` in the codebase (Paddle
  webhook signatures), confirming sessions are **not** PHP‑serialized.

### Definitive answer
Sessions are stored **server‑side in Redis**, under a key of the form
**`session:<uuid4>`** (observed: `session:8bb4e244-e110-47d9-a486-e0ea2510386c`). The
stored value is a **Python `pickle`** of the session dict — the raw bytes begin with the
pickle protocol‑4 framing `\x80\x04\x95` (decimal opcode `0x80` = `PROTO`, version `4`),
**not** PHP‑serialized text such as `a:N:{…}`. For the authenticated session the exact
deserialized key set observed was **`['_fresh', '_id', '_permanent', '_user_id',
'csrf_token', 'sudo_time']`** — i.e. the Flask‑Login keys `_user_id`, `_fresh`, `_id`,
the Flask‑WTF `csrf_token`, plus `_permanent` (set because the app marks sessions
permanent) and SimpleLogin's `sudo_time`. The `slapp` cookie itself holds **only** the
itsdangerous‑signed identifier (`<uuid4>.<hmac-signature>`); unsigning it recovers exactly
the `<uuid4>` used as the Redis key suffix. The authenticated session's TTL was 604,767 s,
consistent with the 7‑day lifetime.

### Rationale
`RedisSessionStore` is a custom `SessionInterface`: `save_session` pickles `dict(session)`
and stores it under `session:<uuid4>`, while the response cookie carries just the
HMAC‑signed identifier. On the next request `open_session` unsigns the cookie, looks up
the Redis entry, and `pickle.loads` it back into a `ServerSession`. This is a classic
server‑side session pattern — the data lives in Redis, the cookie is only a signed
pointer — which is why the bytes are Python pickle and why the cookie alone reveals
nothing about the session contents.

### Assumption corrected
A common assumption is that the session bytes are **PHP‑serialized**. The raw bytes prove
otherwise: they are **Python pickle** (leading `\x80\x04\x95`). The `phpserialize`
dependency exists solely for Paddle webhook signature verification
(`app/paddle_utils.py:L51`) and is never used for sessions.

---

## Q4 — Session identifier rotation across login (before/after)

### Question (verbatim)
> "Capture the session ID from the cookie BEFORE login, go through the login flow, and
> determine if the identifier value stays the same or is replaced with a new one. Show
> actual before/after values."

### Method / commands
Capture the `slapp` cookie from the unauthenticated `GET /auth/login` (the **before**
value), complete the login `POST`, then capture the `slapp` cookie again (the **after**
value). Unsign both with the app's session signer and compare the recovered UUIDs.

```bash
curl -i -c jar_before.txt http://127.0.0.1:7777/auth/login        # capture BEFORE slapp
# POST email+password+csrf_token using jar_before.txt -> jar_after.txt   # capture AFTER slapp
# unsign both with itsdangerous.Signer("secret", salt="session", key_derivation="hmac")
```

### Captured runtime evidence
```text
RAW cookie BEFORE login: 8bb4e244-e110-47d9-a486-e0ea2510386c.oS0skyfxjLhZvrvCaUBiP31JhxI
RAW cookie AFTER  login: 8bb4e244-e110-47d9-a486-e0ea2510386c.oS0skyfxjLhZvrvCaUBiP31JhxI
Cookies byte-identical?                    : True
Decoded id BEFORE                          : 8bb4e244-e110-47d9-a486-e0ea2510386c
Decoded id AFTER                           : 8bb4e244-e110-47d9-a486-e0ea2510386c
Identifiers MATCH (reused, NOT rotated)?   : True
```

The login itself succeeded (the `POST` returned `302` to `/dashboard/`, and an
authenticated `GET /dashboard/` returned `200`), so the unchanged identifier is observed
across a genuine privilege transition.

### Code citation(s)
- `app/session.py:L68-L80` — `open_session` reuses a validly‑signed identifier; it mints
  a fresh `uuid4` only when none is present/valid.
- `app/session.py:L102-L104` — `save_session` re‑signs the **same** `session.session_id`.
- `app/session.py:L47-L59` — `extract_and_validate_session_id` returns `None` only on
  `BadSignature` (a forged/unsigned identifier gets a fresh session, but a legitimately
  signed one is preserved).
- `app/session.py:L61-L66` and `L117-L121` — rotation happens **only on logout**:
  `purge_session` deletes the Redis entry and assigns a new `uuid4`, invoked by
  `logout_session`.
- `app/extensions.py:L8` — Flask‑Login `session_protection = "strong"` is a separate
  client‑fingerprint defense, **not** identifier rotation.

### Definitive answer
The session identifier is **REUSED, not rotated, across login** — the before and after
values are byte‑identical (`8bb4e244-e110-47d9-a486-e0ea2510386c` in both cases, including
the same HMAC signature). The cookie does not change at all when the anonymous session is
elevated to an authenticated one.

### Rationale
`open_session` keeps any validly‑signed identifier it receives, and `save_session`
re‑signs that same `session.session_id`, so the UUID survives the login transition; a new
identifier is minted only when the incoming cookie is missing or fails signature
validation, or on logout via `purge_session`. The established best practice is to
regenerate the session identifier on login to defend against session fixation, but neither
Flask core nor Flask‑Login does so automatically, and this **custom** `RedisSessionStore`
does not implement rotation on login either. (Flask‑Login's `session_protection="strong"`
guards against client‑fingerprint mismatches — a different mechanism that does not rotate
the identifier.)

---


## Q5 — Forwarded‑email header handling (custom X‑header, Received, Reply‑To)

### Question (verbatim)
> "For an incoming email with a custom X-header, a Received header, and a Reply-To header
> — determine which of these three survive to the forwarded message and which get
> stripped. Send an actual test email and examine the result."

### Method / commands
With `NOT_SEND_EMAIL=true`, the live SMTP path does not emit the full forwarded message
(it only logs a one‑line summary at `app/mail_sender.py:L130-L135`). To capture the
**complete** forwarded headers while still exercising the **real** forward code, the
running app's `mail_sender` was switched into its built‑in store mode
(`store_emails_instead_of_sending`, `app/mail_sender.py:L102`) — the captured
`SendRequest` is appended **before** the `NOT_SEND_EMAIL` early‑return
(`app/mail_sender.py:L128-L135`). A crafted inbound message carrying all three headers was
then pushed through the actual `email_handler.handle_forward(...)` to the seeded alias
`e1@sl.local`, and the resulting forwarded message's headers were read back from the store.

```python
# disposable in-process driver (deleted afterward)
from app.mail_sender import mail_sender
import email, email_handler
msg = email.message_from_string(
    "From: Outside Sender <outsider@gmail.com>\n"
    "To: e1@sl.local\nSubject: Q5 Header Survival Probe\n"
    "Reply-To: secret-replyto@external-example.com\n"
    "Received: from probe.example.org (... [203.0.113.55]) by mx.sl.local ...\n"
    "X-Custom-Probe-Header: CUSTOM_VALUE_SHOULD_BE_STRIPPED\n"
    "Message-ID: <q5probe-unique@external-example.com>\n"
    "Content-Type: text/plain; charset=utf-8\n\nbody\n")
class Env: mail_from="outsider@gmail.com"; rcpt_tos=["e1@sl.local"]; mail_options=[]; rcpt_options=[]
mail_sender.purge_stored_emails(); mail_sender.store_emails_instead_of_sending(True)
email_handler.handle_forward(Env(), msg, "e1@sl.local")
for sr in mail_sender.get_stored_emails(): print(dict(sr.msg.items()))
```

### Captured runtime evidence
`handle_forward(...)` returned `[(True, '250 Message accepted for delivery')]`. The
**forwarded** message (envelope_to = `john@wick.com`) had exactly these headers:

```text
Subject: Q5 Header Survival Probe
Message-ID: <q5probe-unique@external-example.com>
Content-Type: text/plain; charset=utf-8
X-SimpleLogin-Type: Forward
X-SimpleLogin-EmailLog-ID: 3
X-SimpleLogin-Envelope-From: outsider@gmail.com
X-SimpleLogin-Original-From: Outside Sender <outsider@gmail.com>
X-SimpleLogin-Envelope-To: e1@sl.local
Date: Fri, 26 Jun 2026 21:46:04 -0000
From: "Outside Sender - outsider at gmail.com" <outsider_at_gmail_com_vdjdqgz@sl.local>
Reply-To: "secret-replyto at external-example.com" <secret-replyto_at_external-example_com_gckjqozq@sl.local>
To: e1@sl.local

--- survival check on the forwarded message ---
X-Custom-Probe-Header present? : None        (STRIPPED)
Received present?              : None        (STRIPPED)
Reply-To present?              : <secret-replyto_at_external-example_com_gckjqozq@sl.local>   (REWRITTEN, not the original)
From (rewritten?)              : <outsider_at_gmail_com_vdjdqgz@sl.local>                     (REWRITTEN reverse-alias)
envelope_from / to             : sl.<reverse-path>@sl.local / john@wick.com
```

The inbound original carried `X-Custom-Probe-Header: CUSTOM_VALUE_SHOULD_BE_STRIPPED`,
`Received: from probe.example.org …`, and `Reply-To: secret-replyto@external-example.com`
— none of those original values appear in the forwarded message.

### Code citation(s)
- `email_handler.py:L793-L807` — the forward phase builds a header **whitelist**
  `headers_to_keep = [FROM, TO, CC, SUBJECT, DATE, MESSAGE_ID, REFERENCES, IN_REPLY_TO,
  SL_QUEUE_ID, LIST_UNSUBSCRIBE, LIST_UNSUBSCRIBE_POST] + headers.MIME_HEADERS`;
  `L808-L809` conditionally appends `AUTHENTICATION_RESULTS`; `L810` calls
  `delete_all_headers_except(msg, headers_to_keep)`.
- `app/email_utils.py:L536-L542` — `delete_all_headers_except` lowercases the keep‑list and
  deletes every header not on it.
- `app/email/headers.py:L13` (`REPLY_TO = "Reply-To"`), `L14` (`RECEIVED = "Received"`),
  and `L44-L51` (`MIME_HEADERS` = Mime‑Version/Content‑Type/Content‑Disposition/
  Content‑Transfer‑Encoding) — confirming that neither a custom `X-` header, nor
  `Received`, nor `Reply-To` is on the keep‑list.
- `email_handler.py:L864-L867` — `From` is rewritten to the reverse‑alias
  `contact.new_addr()`; `L869-L873` — `Reply-To` is re‑added (as a reverse‑alias) **only
  when** `reply_to_contact` exists.

### Definitive answer
Of the three headers, **none survive verbatim**:
- the **custom `X-` header is STRIPPED** (absent from the forwarded message);
- the **`Received` header is STRIPPED**;
- the **`Reply-To` header is NOT preserved** — its original value is removed by the
  whitelist, and because the inbound message had a Reply‑To (creating a reply‑to contact),
  a **new** `Reply-To` pointing to a SimpleLogin reverse‑alias is added in its place.

(For completeness, `From` is likewise rewritten to a reverse‑alias, and SimpleLogin adds
its own `X-SimpleLogin-*` and a `Date` header.)

### Rationale
Forwarding applies a strict header **whitelist** via `delete_all_headers_except`:
everything outside the keep‑list is deleted, which removes the custom `X-` header and the
`Received` header outright. Sender‑identifying headers (`From`, `Reply-To`) are then
deliberately **rewritten** to reverse‑alias addresses so the recipient cannot see the
real sender address and can reply through SimpleLogin's relay — which is why the original
`Reply-To` value does not survive even though a `Reply-To` header is present in the output.

---

## Q6 — Alias‑creation token expiration window

### Question (verbatim)
> "Experimentally verify the expiration time window by testing a token immediately, then
> waiting and re-testing until the boundary where it stops working is found."

### Method / commands
Two complementary experiments used the app's own signer
(`itsdangerous.TimestampSigner(config.CUSTOM_ALIAS_SECRET)`) and the real verifier
(`app/alias_suffix.py:check_suffix_signature`, which calls
`signer.unsign(signed_suffix, max_age=600)`):

1. **Real‑time, wall‑clock:** sign a suffix, then call `check_suffix_signature` at real
   elapsed times bracketing 600 s (595, 598, 599, 600, 601, 602, 603, 605 s).
2. **Instant boundary sweep:** craft tokens whose embedded timestamp is back‑dated by a
   precise age (0…610 s) and verify each with the same `check_suffix_signature`.

```python
from app.alias_suffix import signer, check_suffix_signature  # max_age=600 is hard-coded
token = signer.sign(".test@sl.local").decode()
check_suffix_signature(token)   # immediately valid; re-test across the ~600s boundary
```

### Captured runtime evidence
Real‑time wall‑clock prober (`CUSTOM_ALIAS_SECRET == FLASK_SECRET + "custom_alias"`
confirmed `True`; itsdangerous 1.1.0):

```text
[t=0.0s]   check_suffix_signature -> '.realtimetest@sl.local'   (valid)
[t=595.1s] -> '.realtimetest@sl.local'   (valid)
[t=598.1s] -> '.realtimetest@sl.local'   (valid)
[t=599.1s] -> '.realtimetest@sl.local'   (valid)
[t=600.1s] -> '.realtimetest@sl.local'   (valid)
[t=601.1s] -> None                       (EXPIRED)
[t=602.1s] -> None                       (EXPIRED)
[t=603.1s] -> None                       (EXPIRED)
[t=605.1s] -> None                       (EXPIRED)
```

Instant back‑dated boundary sweep (same signer + verifier):

```text
age(s):   0  300  595  598  599  600 | 601  602  605  610
valid? :  T   T    T    T    T    T  |  F    F    F    F
```

Both experiments agree exactly: a signed suffix is valid through ~600 s and is rejected
from ~601 s onward.

### Code citation(s)
- `app/alias_suffix.py:L11` — `signer = itsdangerous.TimestampSigner(config.CUSTOM_ALIAS_SECRET)`.
- `app/alias_suffix.py:L40` — `signer.unsign(signed_suffix, max_age=600).decode()`;
  `L41-L42` — `except itsdangerous.BadSignature: return None`.
- `app/config.py:L201` — `CUSTOM_ALIAS_SECRET = FLASK_SECRET + "custom_alias"`.

### Definitive answer
The expiration window is **600 seconds (10 minutes)**. Tokens validate immediately and
remain valid up to and including ~600 s; at ~601 s and beyond, `check_suffix_signature`
returns **`None`**.

### Rationale
`check_suffix_signature` passes `max_age=600` to `TimestampSigner.unsign`. itsdangerous
embeds a signing timestamp in the token and, on `unsign`, raises `SignatureExpired` (a
subclass of `BadSignature`) once the token's age exceeds `max_age`; the verifier catches
`BadSignature` and returns `None`. Hence any token older than the 600‑second window is
rejected — which both the real‑time and the back‑dated experiments demonstrate at the
601 s boundary.

---


## Q7 — API key usage statistics across N calls

### Question (verbatim)
> "When making several API calls with the same key, determine which DB fields get updated
> and the ACTUAL observed values after the calls."

### Method / commands
Read the baseline `api_key` rows, make exactly **N = 5** keyed calls with the `Chrome`
key (`code`), then re‑read the rows.

```bash
psql … -c "SELECT code,name,times,last_used FROM api_key ORDER BY id;"   # baseline
for i in $(seq 1 5); do
  curl -s -o /dev/null -H "Authentication: code" http://127.0.0.1:7777/api/user_info
done
psql … -c "SELECT code,name,times,last_used FROM api_key ORDER BY id;"   # after
```

### Captured runtime evidence
```text
BASELINE:
  code   | Chrome  | times=9  | last_used=2026-06-26 21:40:59.476331
  codeFF | Firefox | times=0  | last_used=(null)

5 keyed calls to /api/user_info  ->  all HTTP 200

AFTER:
  code   | Chrome  | times=14 | last_used=2026-06-26 21:46:41.101605
  codeFF | Firefox | times=0  | last_used=(null)

wall-clock at the last call: 2026-06-26 21:46:41 UTC
```

`times` advanced by exactly **5** (9 → 14); `last_used` advanced to **21:46:41.101605**,
matching the wall‑clock time of the final call. The unused `Firefox` key was unchanged
(`times=0`, `last_used` still NULL).

### Code citation(s)
- `app/api/base.py:L30-L32` — on each keyed call: `api_key.last_used = arrow.now()`,
  `api_key.times += 1`, `Session.commit()`.
- `app/models.py:L2353` — table `api_key`; `L2356-L2360` — columns `code`, `name`,
  `last_used` (ArrowType), `times` (Integer, default 0), `sudo_mode_at` (ArrowType).

### Definitive answer
Exactly **two** DB fields are updated per keyed API call: **`times`** (incremented by 1)
and **`last_used`** (set to the current time). After N = 5 calls, the observed values were
`times = 14` (baseline 9 + 5) and `last_used = 2026-06-26 21:46:41.101605` (≈ the final
call's timestamp). Keys not used in the calls are untouched.

### Rationale
The keyed branch of `authorize_request()` records usage statistics — `last_used = arrow.now()`
and `times += 1` — and commits on **every** authenticated API request that presents a
valid `Authentication` key. Because the commit happens inline on each request, the
counter increases by exactly the number of keyed calls and `last_used` tracks the most
recent one. (Session‑authenticated calls take the no‑key branch and therefore do not touch
these columns.)

---

## Q8 — Failed‑login behavior (logs + HTTP response)

### Question (verbatim)
> "When a login attempt fails with wrong credentials, capture the specific log messages
> that appear and the HTTP response details returned."

### Method / commands
`GET /auth/login` to obtain a fresh CSRF token, then `POST /auth/login` with a **wrong**
password for `john@wick.com`; capture the HTTP status and body, and the server log lines
emitted for that POST.

```bash
curl -i -c jar.txt http://127.0.0.1:7777/auth/login                 # fresh csrf
curl -i -b jar.txt -d "email=john@wick.com" \
     -d "password=TOTALLY_WRONG_PASSWORD" --data-urlencode "csrf_token=$CSRF" \
     http://127.0.0.1:7777/auth/login
# inspect only the new server-log lines for this POST
```

### Captured runtime evidence
```text
HTTP/1.0 200 OK
... body contains the flashed message: "Email or password incorrect"
... (no Location header -> no redirect; the login page is re-rendered)
```

Server log lines emitted for the failed login (the only lines produced):

```text
SL - DEBUG - "/app/server.py:284" - after_request() - 127.0.0.1 GET  /auth/login ImmutableMultiDict([]) 200, takes 0.022…
SL - DEBUG - "/app/server.py:284" - after_request() - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 200, takes 0.279…
```

There was **no ERROR/WARNING log line** for the failed authentication itself — only the
DEBUG access‑log entry showing HTTP **200** for `POST /auth/login`.

### Code citation(s)
- `app/auth/views/login.py:L45` — credential check (`if not user or not user.check_password(...)`);
  `L47` — `g.deduct_limit = True`; `L49` — `flash("Email or password incorrect", "error")`;
  `L50` — `LoginEvent(LoginEvent.ActionType.failed).send()`; `L74` —
  `render_template("auth/login.html", …)` (served as HTTP 200).
- `app/events/auth_event.py:L22-L25` — `LoginEvent.send()` calls
  `newrelic.agent.record_custom_event("LoginEvent", {…})` — a telemetry custom event, not a
  Python log line.

### Definitive answer
A wrong‑credentials login **re‑renders the login page with HTTP 200** (no redirect),
flashes the message **"Email or password incorrect"** into the response body, and emits a
`LoginEvent.failed` **New Relic custom event**. There is **no dedicated Python log line**
for the failure — the only server output is the access‑log entry recording `POST
/auth/login … 200`.

### Rationale
The failure branch flashes the error and records telemetry (`LoginEvent(...).send()`),
but it does **not** redirect or set an error status; control falls through to the final
`render_template("auth/login.html", …)`, which Flask serves with HTTP 200. Because
`LoginEvent.send()` records a New Relic custom event rather than writing to the logger, no
error/warning line appears — anyone looking for an error log for the failed login will not
find one; the observable signals are the flashed message and the access‑log 200.

---

## Assumptions corrected (consolidated)

The investigation explicitly verified four points where a prior assumption could differ
from the running system:

1. **Session serialization is Python `pickle`, not PHP serialization.** The raw Redis
   bytes begin with the pickle protocol‑4 opcode `\x80\x04\x95`, never PHP's `a:N:{…}`
   (`app/session.py:L91,L76`). `phpserialize` is used only for Paddle webhooks
   (`app/paddle_utils.py:L51`).
2. **The privileged‑operation status is HTTP `440` with body `{"error":"Need sudo"}`** —
   a non‑standard code (Microsoft IIS "Login Time‑out"), not the conventional 401/403
   (`app/api/base.py:L69-L70`). A pure browser‑session call to that endpoint instead
   yields HTTP 500 due to `None.sudo_mode_at` (`app/api/base.py:L42,L46-L49`).
3. **The session identifier is reused across login, not rotated.** The `slapp` cookie is
   byte‑identical before and after login (`app/session.py:L68-L80,L102-L104`); rotation
   occurs only on logout (`app/session.py:L61-L66,L117-L121`).
4. **The alias‑suffix token window is exactly 600 seconds.** Verified at the 601 s
   boundary by both a real‑time and a back‑dated experiment (`app/alias_suffix.py:L40`).

---

## Summary table

| # | Question (topic) | Observed answer |
|---|------------------|-----------------|
| Q1 | API auth without a key | Session‑only call → **HTTP 200** + JSON; no auth → **401** `{"error":"Wrong api key"}` |
| Q2 | Privileged op status code | **HTTP 440** `{"error":"Need sudo"}` (key, sudo inactive); pure session → **HTTP 500** (`None.sudo_mode_at`) |
| Q3 | Session storage format | Redis `session:<uuid4>`; **Python pickle** (proto 4); keys `[_fresh,_id,_permanent,_user_id,csrf_token,sudo_time]`; `slapp` = signed id only |
| Q4 | Session id rotation | **Reused, not rotated** — `slapp` byte‑identical before/after login |
| Q5 | Forwarded‑email headers | Custom `X-` **stripped**, `Received` **stripped**, `Reply-To` **not preserved** (rewritten to reverse‑alias) |
| Q6 | Alias token expiry | **600 seconds** (valid ≤600 s, rejected ≥601 s) |
| Q7 | API key usage stats | **`times` +1/call** and **`last_used` = now** per keyed call (9→14 over 5 calls) |
| Q8 | Failed login | **HTTP 200** re‑render + flashed "Email or password incorrect" + New Relic `LoginEvent.failed` (no error log line) |
