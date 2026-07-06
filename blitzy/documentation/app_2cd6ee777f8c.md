# SimpleLogin Runtime-Behavior Investigation — Evidence-Backed Answers

This document answers **eight runtime-behavior questions** about the **SimpleLogin** Flask
email-alias application. **Every answer was derived by actually building, running, and observing the
live system** — not by reading source code alone. For each question you will find: the restated
question, the exact command(s) executed, the **complete, unedited** output, the governing
`file:line` + function citation, before/after state where applicable, and a cause→effect rationale.

> **Method note (honesty of evidence).** All probes drive the *real* entry points — the running
> HTTP API on `:7777`, the running inbound SMTP handler on `:20381`, and the real `itsdangerous`
> signer used by the alias-creation flow. Where a value could not be observed under the strict
> default configuration (Q5's full forwarded body under `NOT_SEND_EMAIL=true`, Q6's 10‑minute
> boundary, Q8's access log), the deviation is **explicitly labeled** and the *real* code path is
> still exercised. Findings that are unflattering (Q2's HTTP 500 `AttributeError`, Q4's lack of
> session-id rotation) are reported exactly as observed and are **not** remediated (this is a
> read-only investigation).

---

## Canonical runtime & invocation

**Environment (canonical, per repo):** Docker image
`andrewparkscaleai/coding-agent:simple-login__app__2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`.
Runtime versions match CI: **Python 3.10** (`Dockerfile:8` → `FROM python:3.10`; observed
`Python 3.10.20`), **PostgreSQL 13** (`.github/workflows/main.yml:47` → `image: postgres:13`;
observed `PostgreSQL 13.23`), **Redis 6** (`docker run ... redis:6`). The canonical run command is
`Dockerfile:47` → `CMD ["gunicorn","wsgi:app","-b","0.0.0.0:7777","-w","2","--timeout","15"]`, and
`wsgi:app` is `server.create_app()` (`wsgi.py`).

**Backends (canonical versions):**

```bash
docker run -d --name sl-postgres -e POSTGRES_USER=myuser -e POSTGRES_PASSWORD=mypassword \
    -e POSTGRES_DB=simplelogin -p 5432:5432 postgres:13
docker run -d --name sl-redis -p 6379:6379 redis:6
```

**Configuration** — `.env` derived from `example.env` (`cp example.env .env`) plus the two keys the
setup requires (`MEM_STORE_URI` enables the Redis session store per `server.py:163-165`;
`GNUPGHOME` for PGP). The app auto-loads `.env` via `load_dotenv()` at `app/config.py:71`. Relevant
values (all defaults from `example.env`):

```
URL=http://localhost:7777          # example.env:6
NOT_SEND_EMAIL=true                # example.env:19  (Q5 hook: outbound mail is not actually sent)
EMAIL_DOMAIN=sl.local              # example.env:22
DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin
FLASK_SECRET=secret                # -> app.secret_key (server.py:151); CUSTOM_ALIAS_SECRET="secretcustom_alias" (app/config.py:201)
MEM_STORE_URI=redis://localhost:6379
GNUPGHOME=/tmp/gnupg
```

**Schema initialization** (final revision captured):

```bash
$ .venv/bin/alembic current
32f25cbf12f6 (head)
$ .venv/bin/alembic upgrade head        # idempotent; 77 tables
```

**Web app (canonical) and inbound SMTP handler (separate process):**

```bash
nohup .venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15 &   # gunicorn 20.0.4, 2 sync workers
nohup .venv/bin/python email_handler.py &                              # aiosmtpd controller, listens :20381
```

The handler logs `Listen for port 20381` (`email_handler.py:2403`) and
`Start mail controller 0.0.0.0 20381` (`email_handler.py:2386`; controller at `email_handler.py:2383`).

**Seed fixtures** — created through the real model-creation path via the Flask CLI
`FLASK_APP=wsgi.py .venv/bin/flask dummy-data` (`fake_data()`; a real CLI entry point, labeled as
such — not hand-edited DB rows):

| Fixture | Value |
|---|---|
| User (activated) | `john@wick.com` / password `password` (user id 1) |
| API key (Chrome) | `code="code"` |
| API key (Firefox) | `code="codeFF"` (pristine, `times=0`) |
| Alias → PGP mailbox | `example@example.com` → `pgp@example.org` |
| Alias → plain mailbox | `begins_cashew220@sl.local` → `john@wick.com` (no PGP) |
| Verified mailbox | `john@wick.com` (user 1) |

**Auth header:** all API probes use the **non-standard `Authentication`** header (NOT
`Authorization`), per `docs/api.md:73` — *"the client includes the `api code` in `Authentication`
header in almost all requests."* The browser session cookie is **`slapp`** (`app/config.py:199`).

---

## Q1 — API call WITHOUT the `Authentication` header while holding a browser session

**Restated question.** What exact HTTP status code and response-body structure are returned when an
API endpoint is invoked **without** the API-key header by a caller holding an authenticated browser
session? And separately, with **neither** cookie nor header?

**Direct answer.**
- **(a) Authenticated browser session, no `Authentication` header → `HTTP 200`** plus the normal
  `user_info` JSON, served via the Flask-Login `current_user` fallback.
- **(b) No cookie and no header → `HTTP 401`** with body `{"error":"Wrong api key"}`.

**Command(s).**

```python
# /tmp/probe_q1.py (excerpt) — real HTTP client against the running app on :7777
s = requests.Session()
csrf = re.search(r'name="csrf_token"[^>]*value="([^"]+)"', s.get(f"{BASE}/auth/login").text).group(1)
s.post(f"{BASE}/auth/login",
       data={"csrf_token": csrf, "email": "john@wick.com", "password": "password"},
       allow_redirects=False)
r = s.get(f"{BASE}/api/user_info")          # (a) cookie present, NO Authentication header
r2 = requests.get(f"{BASE}/api/user_info")  # (b) no cookie, no Authentication header
```

**Complete, unedited output.**

```
======================================================================
Q1(a): authenticated browser session, NO Authentication header
======================================================================
[login POST] status=302 location=http://localhost:7777/dashboard/
[login POST] slapp cookie now: 8bdc1aa8-be5c-4d91-87e4-b29a95f71155.rYWmbCMTMaUAg0KKP7tNe7FNUQs

$ GET /api/user_info  (Cookie: slapp=<session>; NO Authentication header)
HTTP status: 200
Response headers (subset):
  Content-Type: application/json
  Content-Length: 244
Response body:
{"can_create_reverse_alias":true,"connected_proton_address":null,"email":"john@wick.com","in_trial":false,"is_premium":true,"max_alias_free_plan":5,"name":"John Wick","profile_picture_url":"http://localhost:7777/static/upload/profile_pic.svg"}


======================================================================
Q1(b): NO cookie AND NO Authentication header
======================================================================

$ GET /api/user_info  (no cookie, no Authentication header)
HTTP status: 401
Response headers (subset):
  Content-Type: application/json
  Content-Length: 26
Response body:
{"error":"Wrong api key"}
```

**Governing code.** `authorize_request()` — `app/api/base.py:16`.
- `app/api/base.py:17` `api_code = request.headers.get("Authentication")` → `None` when the header is absent.
- `app/api/base.py:18` `api_key = ApiKey.get_by(code=api_code)` → `None`.
- `app/api/base.py:20-25` `if not api_key: if current_user.is_authenticated: g.user = current_user` → the session fallback (case a).
- `app/api/base.py:27` `else: return jsonify(error="Wrong api key"), 401` → the no-auth case (case b).
- Endpoint: `user_info()` at `app/api/views/user_info.py:50-52` (`@api_bp.route("/user_info")` + `@require_api_auth`), returning `jsonify(user_to_dict(user))` (`user_info.py:67`); the JSON keys are built by `user_to_dict()` (`user_info.py:28-47`). Contract: `docs/api.md:73`.

**Cause → effect.** SimpleLogin reads the API key from the non-standard `Authentication` header. When
it is absent, `authorize_request()` does **not** immediately reject; it checks the Flask-Login
session and, if `current_user.is_authenticated`, sets `g.user = current_user` and returns `None`
(no error) — so the request proceeds and the endpoint returns `200` + the user JSON. With no session
either, control reaches the `else` at `base.py:27`, returning `401` with `{"error":"Wrong api key"}`
(exactly 26 bytes).

---

## Q2 — sudo-gated `DELETE /api/user` (both variants)

**Restated question.** What **specific (non-standard)** HTTP status code and **exact** error-message
text are returned when the privileged, sudo-gated operation is attempted (i) with **only a browser
session**, and (ii) with a **valid API key lacking recent sudo**?

**Direct answer.**
- **(a) Browser-session-only (no `Authentication` header) → `HTTP 500`, body `{"error":"Internal error"}`.**
  This is **not** a clean authorization error: `g.api_key` is `None`, and the sudo guard dereferences
  `None.sudo_mode_at`, raising `AttributeError`, which the global exception handler turns into a 500.
  *(This is an edge/bug finding — reported, not fixed.)*
- **(b) Valid API key without recent sudo → `HTTP 440`, body `{"error":"Need sudo"}`.** HTTP 440 is
  **non-standard** (introduced by Microsoft IIS as *"Login Timeout"*), repurposed here with the
  custom `"Need sudo"` message — which is why this is "not a standard authorization error."

`DELETE /api/user` (`delete_user()` at `app/api/views/user.py:12-14`) is the **sole** `@require_api_sudo`
endpoint; the guard is `require_api_sudo` at `app/api/base.py:63-73`.

**Command(s).**

```python
# /tmp/probe_q2.py (excerpt)
# (a) browser session only
s = requests.Session(); csrf = get_csrf(s)
s.post(f"{BASE}/auth/login", data={"csrf_token":csrf,"email":"john@wick.com","password":"password"}, allow_redirects=False)
r = s.delete(f"{BASE}/api/user")                                   # NO Authentication header
# (b) valid API key, sudo_mode_at is NULL
r2 = requests.delete(f"{BASE}/api/user", headers={"Authentication":"code"})
```

**Complete, unedited output.**

```
======================================================================
Q2(a): DELETE /api/user with ONLY a browser session (no Authentication header)
======================================================================
[authenticated] slapp cookie: 42b2317b-143b-47bd-a3b7-037c73ef4adc.2R-Ss9EYqbdVkmW1u3OpR7QjhKc

$ DELETE /api/user  (Cookie: slapp=<session>; NO Authentication header)
HTTP status: 500
Content-Type: application/json
Response body (first 800 chars):
{"error":"Internal error"}


======================================================================
Q2(b): DELETE /api/user with valid API key lacking recent sudo
======================================================================

$ DELETE /api/user  (Authentication: code ; api_key.sudo_mode_at is NULL)
HTTP status: 440
Content-Type: application/json
Content-Length: 22
Response body:
{"error":"Need sudo"}
```

**Server-side traceback for variant (a)** (complete, unedited, from the gunicorn stderr log):

```
2026-07-06 22:24:29,800 - SL - ERROR - 33646 - "server.py:390" - error_handler() -  - 'NoneType' object has no attribute 'sudo_mode_at'
Traceback (most recent call last):
  File ".../.venv/lib/python3.10/site-packages/flask/app.py", line 1950, in full_dispatch_request
    rv = self.dispatch_request()
  File ".../.venv/lib/python3.10/site-packages/flask/app.py", line 1936, in dispatch_request
    return self.view_functions[rule.endpoint](**req.view_args)
  File ".../app/api/base.py", line 69, in decorated
    if not check_sudo_mode_is_active(g.api_key):
  File ".../app/api/base.py", line 47, in check_sudo_mode_is_active
    return api_key.sudo_mode_at and g.api_key.sudo_mode_at >= arrow.now().shift(
AttributeError: 'NoneType' object has no attribute 'sudo_mode_at'
```

**Governing code.**
- Variant (a): with no key, `authorize_request()` sets `g.api_key = None` (`app/api/base.py:42`).
  `require_api_sudo` calls `check_sudo_mode_is_active(g.api_key)` (`app/api/base.py:69`), which at
  `app/api/base.py:47` evaluates `api_key.sudo_mode_at` on `None` → `AttributeError`. The global
  `@app.errorhandler(Exception)` `error_handler()` (`server.py:388-394`) logs the error
  (`server.py:390`) and, because the path starts with `/api/`, returns
  `jsonify(error="Internal error"), 500` (`server.py:392`).
- Variant (b): `check_sudo_mode_is_active` (`app/api/base.py:46-49`) returns falsy because
  `api_key.sudo_mode_at` is `NULL`, so `require_api_sudo` returns
  `jsonify(error="Need sudo"), 440` (`app/api/base.py:70`).

**Cause → effect.** The "just a browser session" case never populates `g.api_key`, so the sudo check
crashes on a `None` dereference and surfaces as a generic `500 {"error":"Internal error"}` rather
than a purpose-built authorization error. The API-key-without-sudo case is handled deliberately,
returning the unusual, non-standard **`440`** with the custom body **`{"error":"Need sudo"}`**.

---

## Q3 — Session data format in storage

**Restated question.** Capture the **raw bytes** persisted in the session storage backend, identify
the **serialization format**, enumerate the **keys** present in a deserialized authenticated session,
and describe how the **storage key** itself is structured.

**Direct answer.**
- **Storage backend:** Redis. **Storage key:** `session:<uuid4>` (prefix `session` + `:` + a UUID4).
- **Serialization format:** Python **`pickle`**, protocol **4** (raw bytes begin `\x80\x04` = the
  pickle `PROTO` opcode followed by version `4`; confirmed `pickle.DEFAULT_PROTOCOL == 4` on Python 3.10.20).
- **Deserialized keys (observed, 6):** `_permanent`, `_fresh`, `csrf_token`, `_user_id`, `_id`,
  `sudo_time`. *(This is more than the four one might expect; reported exactly as observed.)*
- **Cookie structure:** `slapp = <uuid4>.<hmac-signature>` — an opaque, HMAC-signed session id
  (the payload itself lives server-side in Redis, not in the cookie).

**Command(s).**

```python
# /tmp/probe_q3.py (excerpt) — authenticate via real login, locate & read the Redis value
signer = itsdangerous.Signer("secret", salt="session", key_derivation="hmac")   # == app/session.py:38-41
session_id = signer.unsign(cookie_val).decode()                                 # extract uuid4 from slapp cookie
r = redis.Redis(host="localhost", port=6379, db=0)
raw = r.get(f"session:{session_id}")            # RAW bytes exactly as stored (key == app/session.py:45)
data = pickle.loads(raw)                         # deserialize (== app/session.py:76)
```

**Complete, unedited output.**

```
STEP 1 — raw slapp cookie value (as emitted by the server):
'cf1eb618-5ba6-4a6a-b106-0c25b00f5a1d.tWGL85BqoEaIfiP4gzBAaqxle9w'
  structure: <uuid4>.<hmac-signature>  ->  split on last '.':
  uuid4 part      = cf1eb618-5ba6-4a6a-b106-0c25b00f5a1d
  signature part  = tWGL85BqoEaIfiP4gzBAaqxle9w

STEP 2 — unsigned session_id (uuid4) = cf1eb618-5ba6-4a6a-b106-0c25b00f5a1d

STEP 3 — Redis storage key = 'session:cf1eb618-5ba6-4a6a-b106-0c25b00f5a1d'

STEP 4 — RAW bytes persisted in Redis (repr, complete, unedited):
b'\x80\x04\x95!\x01\x00\x00\x00\x00\x00\x00}\x94(\x8c\n_permanent\x94\x88\x8c\x06_fresh\x94\x88\x8c\ncsrf_token\x94\x8c(8fbe64382e066269c58f991acb8768a7aa643165\x94\x8c\x08_user_id\x94\x8c$8a0e2333-a5e9-4447-b149-7584af84be26\x94\x8c\x03_id\x94\x8c\x80f4a4143536f3e6712e19e6ce901a12f1ff7e8c98d04dd3e41c746e85093b48a1bfaa93650d1759a0cb7f13cba57b7f96e40ed981f0c49af1cb94f9905ee1dd03\x94\x8c\tsudo_time\x94JN+Lju.'

  first byte = 128 (0x80)  -> pickle opcode marker
  total length = 300 bytes

STEP 5 — pickle.loads(raw) -> type=dict
  deserialized dict KEYS = ['_permanent', '_fresh', 'csrf_token', '_user_id', '_id', 'sudo_time']
  full deserialized dict:
    '_permanent': True
    '_fresh': True
    'csrf_token': '8fbe64382e066269c58f991acb8768a7aa643165'
    '_user_id': '8a0e2333-a5e9-4447-b149-7584af84be26'
    '_id': 'f4a4143536f3e6712e19e6ce901a12f1ff7e8c98d04dd3e41c746e85093b48a1bfaa93650d1759a0cb7f13cba57b7f96e40ed981f0c49af1cb94f9905ee1dd03'
    'sudo_time': 1783376718

STEP 6 — Redis TTL(seconds) on this authenticated key = 604800
```

Corroborating check of the pickle protocol:

```
$ .venv/bin/python -c "import pickle,sys; print(sys.version.split()[0]); print(pickle.DEFAULT_PROTOCOL); print(pickle.dumps({})[:2])"
3.10.20
4
b'\x80\x04'
```

**Governing code.** Custom `RedisSessionStore` (`app/session.py:31`), bound to `app.session_interface`
at `app/redis_services.py:12`.
- Key: `SESSION_PREFIX = "session"` (`app/session.py:18`), `_get_key()` returns
  `f"{SESSION_PREFIX}:{session_Id}"` (`app/session.py:44-45`) → `session:<uuid4>`.
- Serialization: `pickle.dumps(dict(session))` on write (`app/session.py:91`) and `pickle.loads(val)`
  on read (`app/session.py:76`). (`import cPickle as pickle` falls back to `import pickle` on Py3,
  `app/session.py:9-12`.)
- Cookie: signed by `itsdangerous.Signer(app.secret_key, salt="session", key_derivation="hmac")`
  (`app/session.py:38-41`); the signed id is written to the cookie in `save_session` (`app/session.py:102-107`).
  `app.secret_key = FLASK_SECRET` (`server.py:151`), cookie name `slapp` (`app/config.py:199`),
  `SESSION_COOKIE_SAMESITE = "Lax"` (`server.py:162`).
- TTL: `app.permanent_session_lifetime` = 7 days (`server.py:207`) → `604800`s for authenticated
  sessions; non-authenticated sessions get `ttl = 300` because `"_user_id" not in session`
  (`app/session.py:95-96`).

**Cause → effect.** SimpleLogin uses **server-side** sessions: the cookie carries only an opaque,
HMAC-signed UUID4 (`<uuid4>.<hmac-signature>`), while the actual session dictionary is
**pickle-serialized** and stored in Redis under `session:<uuid4>`. The observed authenticated
session contains Flask-Login bookkeeping (`_user_id`, `_id`, `_fresh`), the permanence flag
(`_permanent`), the Flask-WTF `csrf_token`, and SimpleLogin's own `sudo_time`. The 7-day TTL matches
`permanent_session_lifetime`, confirming this is an authenticated (not the 300 s non-auth) session.

---

## Q4 — Session identifier BEFORE vs AFTER login (session-fixation)

**Restated question.** Capture the session identifier from the cookie **before** login, drive the
full login flow, capture it **after** login, and report whether the identifier is **preserved or
replaced**.

**Direct answer.** The session identifier is **UNCHANGED** — SimpleLogin does **not** rotate the
session id on login. The embedded UUID4 (and in fact the entire signed cookie value) is byte-for-byte
identical before and after authentication. This is a **session-fixation** exposure (reported as an
observed finding; not remediated here).

**Command(s).**

```python
# /tmp/probe_q4.py (excerpt) — same requests.Session across the login
signer = itsdangerous.Signer("secret", salt="session", key_derivation="hmac")
r0 = s.get(f"{BASE}/auth/login")                          # unauthenticated; yields cookie + CSRF
before_cookie = s.cookies.get("slapp"); before_id = signer.unsign(before_cookie).decode()
s.post(f"{BASE}/auth/login", data={"csrf_token": csrf, "email":"john@wick.com","password":"password"}, allow_redirects=False)
after_cookie = s.cookies.get("slapp");  after_id  = signer.unsign(after_cookie).decode()
```

**Complete, unedited output (two runs).**

```
BEFORE LOGIN (unauthenticated GET /auth/login):
  slapp cookie   = 93349069-b3e1-4541-a7e4-d73bd317831a.1bm_wK5dPGDSNo9Yb8IDWx117g4
  session_id(uuid4) = 93349069-b3e1-4541-a7e4-d73bd317831a

[login POST] status=302 location=http://localhost:7777/dashboard/

AFTER LOGIN:
  slapp cookie   = 93349069-b3e1-4541-a7e4-d73bd317831a.1bm_wK5dPGDSNo9Yb8IDWx117g4
  session_id(uuid4) = 93349069-b3e1-4541-a7e4-d73bd317831a

COMPARISON:
  BEFORE session_id = 93349069-b3e1-4541-a7e4-d73bd317831a
  AFTER  session_id = 93349069-b3e1-4541-a7e4-d73bd317831a
  IDENTIFIER PRESERVED (unchanged)? True
  full cookie identical (incl signature)? True

=== RUN 2 (stability of the no-rotation finding) ===
COMPARISON:
  BEFORE session_id = 23d9e46b-be42-4908-8081-3e42d41fb6be
  AFTER  session_id = 23d9e46b-be42-4908-8081-3e42d41fb6be
  IDENTIFIER PRESERVED (unchanged)? True
  full cookie identical (incl signature)? True
```

**Governing code.** `RedisSessionStore.open_session()` (`app/session.py:68-80`): it calls
`extract_and_validate_session_id()` (`app/session.py:47-59`) and, when a validly-signed id is
present with data in Redis, returns `ServerSession(data, session_id=session_id)` (`app/session.py:77`)
— i.e. it **reuses** the incoming id. `login_user()` merely adds `_user_id` to that same dict; nothing
in the login path allocates a new UUID4 or calls `purge_session()`. Flask-Login is configured with
`session_protection = "strong"` (`app/extensions.py:8`), which regenerates the `_id` fingerprint but
**not** the server-side session id / cookie UUID4.

**Before → after.** BEFORE = the UUID4 minted for the anonymous `GET /auth/login`; AFTER (post
successful `302`→`/dashboard/`) = the **same** UUID4. Because the HMAC signature over an unchanged id
is deterministic, even the full signed cookie is identical.

**Cause → effect.** Since `open_session` reuses any validly-signed session id and the login flow adds
authenticated state *in place* (never rotating the id), an id known before authentication remains
valid after it — the classic precondition for session fixation. Reported as observed; no change made.


---

## Q5 — Email forwarding header handling (three named headers)

**Restated question.** Send a real test email carrying (a) a custom `X-` header, (b) a `Received`
header, and (c) a `Reply-To` header, then examine the forwarded message and report which of the three
**survive** and which are **stripped**.

**Direct answer (each header, by name).**
- **`X-Custom-Test` (custom `X-` header): STRIPPED** — absent from the forwarded message (not in the whitelist).
- **`Received`: STRIPPED** — absent from the forwarded message (not in the whitelist).
- **`Reply-To`: REPLACED** — the *original* value (`someone-external@example.com`) is removed by the
  whitelist, but a **new reverse-alias `Reply-To`** is re-added. A `Reply-To` therefore appears in the
  output, but pointing at a `…@sl.local` reverse-alias, **not** the sender's original address.

**Probe method.** The crafted RFC822 message is submitted through the handler's **real** inbound SMTP
entry point (`smtplib` → `localhost:<smtp port>`, envelope `MAIL FROM=<outsider@external-domain.test>`,
`RCPT TO=<alias>`). The header transformation happens inside `forward_email_to_mailbox()` **before**
delivery, independent of `NOT_SEND_EMAIL`. Two complementary observations are provided:

**(5a) Under the canonical `NOT_SEND_EMAIL=true` config** (handler on `:20381`, alias
`example@example.com`). Note: with `NOT_SEND_EMAIL=true`, `MailSender.send()` (`app/mail_sender.py:130-137`)
only **logs a summary** and returns — it does not print the full transformed message. The handler's
own DEBUG log nonetheless proves the From-rewrite and the Reply-To replacement:

```
- email_handler.py:867 - forward_email_to_mailbox() - From header, new:"Outside Sender - outsider at external-domain.test" <outsider_at_external-domain_test_jrfjufcxw@sl.local>, old:Outside Sender <outsider@external-domain.test>
- email_handler.py:873 - forward_email_to_mailbox() - Reply-To header, new:"someone-external at example.com" <someone-external_at_example_com_mcvelvsgab@sl.local>, old:None
- app/mail_sender.py:131 - send() - send email with subject 'Q5 header-forwarding probe', from '"Outside Sender - outsider at external-domain.test" <outsider_at_external-domain_test_jrfjufcxw@sl.local>' to 'example@example.com'
```

The `old:None` on the `Reply-To` line is significant: by the time the reverse-alias `Reply-To` is
added (`email_handler.py:870` reads `msg[headers.REPLY_TO]`), the original `Reply-To` has **already
been stripped** by the whitelist, so there is no old value to report.

**(5b) Full forwarded wire message** — to capture the *complete* transformed header block, a local
SMTP sink was placed at the mailbox's downstream MTA position and a second handler was run with the
real send path enabled. **Labeled deviation from default:** this second handler used
`CONFIG=/tmp/q5.env` (a copy of `.env` with `NOT_SEND_EMAIL` **removed** and
`POSTFIX_SERVER=127.0.0.1`, `POSTFIX_PORT=1025`), listening on SMTP port `20382`; the alias used was
`begins_cashew220@sl.local` → the **non-PGP** mailbox `john@wick.com` (so the body is not encrypted).
The forward code path (`forward_email_to_mailbox` → `delete_all_headers_except` → Reply-To re-add) is
identical; only the final delivery target differs. Commands:

```bash
grep -v '^NOT_SEND_EMAIL' .env > /tmp/q5.env
printf 'POSTFIX_SERVER=127.0.0.1\nPOSTFIX_PORT=1025\nPOSTFIX_TIMEOUT=10\n' >> /tmp/q5.env
# sink on 127.0.0.1:1025 prints the complete received message
nohup .venv/bin/python /tmp/sink.py &
CONFIG=/tmp/q5.env nohup .venv/bin/python email_handler.py -p 20382 &
.venv/bin/python /tmp/craft_email2.py 20382          # RCPT begins_cashew220@sl.local
```

Crafted input message (the three target headers in bold context):

```
From: Outside Sender <outsider@external-domain.test>
To: begins_cashew220@sl.local
Subject: Q5 header-forwarding probe
X-Custom-Test: hello-from-outside
Received: from evil.example.net (evil.example.net [203.0.113.7]) by
 mx.external-domain.test with ESMTP id ABC123; Mon, 06 Jul 2026 00:00:00 +0000
Reply-To: someone-external@example.com
Content-Type: text/plain; charset="utf-8"
Content-Transfer-Encoding: 7bit
MIME-Version: 1.0

This is the Q5 body. Testing which headers survive forwarding.
```

Complete, unedited forwarded message as received by the sink (the real wire output):

```
========== SINK RECEIVED FORWARDED MESSAGE ==========
envelope.mail_from = sl.lmycyibtfqqdemzxgmydgmk5.ct2aqcfu7hunq@sl.local
envelope.rcpt_tos  = ['john@wick.com']
---------- RAW MESSAGE (content) ----------
Subject: Q5 header-forwarding probe
Content-Type: text/plain; charset="utf-8"
Content-Transfer-Encoding: 7bit
MIME-Version: 1.0
X-SimpleLogin-Type: Forward
X-SimpleLogin-EmailLog-ID: 3
X-SimpleLogin-Envelope-From: outsider@external-domain.test
X-SimpleLogin-Original-From: Outside Sender <outsider@external-domain.test>
X-SimpleLogin-Envelope-To: begins_cashew220@sl.local
Date: Mon, 06 Jul 2026 22:31:33 -0000
From: "Outside Sender - outsider at external-domain.test"
 <outsider_at_external-domain_test_siqoqwidsw@sl.local>
Reply-To: "someone-external at example.com"
 <someone-external_at_example_com_xtrziixnv@sl.local>
To: begins_cashew220@sl.local

This is the Q5 body. Testing which headers survive forwarding.



========== END FORWARDED MESSAGE ==========
```

Comparing the crafted input to this output: **`X-Custom-Test` is gone**, **`Received` is gone**, and
the **`Reply-To` is a `…@sl.local` reverse-alias**, not the original `someone-external@example.com`.
(The `X-SimpleLogin-*` headers are *added* by SimpleLogin — user 1 has `include_header_email_header`
enabled — and are not the input headers under test.)

**Governing code.** `forward_email_to_mailbox()` (`email_handler.py:679`). The whitelist
`headers_to_keep` is assembled at `email_handler.py:793-807` = `FROM, TO, CC, SUBJECT, DATE,
MESSAGE_ID, REFERENCES, IN_REPLY_TO, SL_QUEUE_ID, LIST_UNSUBSCRIBE, LIST_UNSUBSCRIBE_POST` +
`MIME_HEADERS` — **neither `X-Custom-Test` nor `Received` is present**. It is applied via
`delete_all_headers_except(msg, headers_to_keep)` (`email_handler.py:810`); the helper lowercases the
whitelist and deletes every header not in it (`app/email_utils.py:536-542`). An incoming `Reply-To`
(≠ alias) causes a `reply_to_contact` to be created (`email_handler.py:586-594`); after the strip, the
reverse-alias `Reply-To` is re-added at `email_handler.py:869-873`
(`add_or_replace_header(msg, "Reply-To", reply_to_contact.new_addr())`). Header-name constants:
`REPLY_TO = "Reply-To"` (`app/email/headers.py:13`), `RECEIVED = "Received"` (`app/email/headers.py:14`),
`SL_QUEUE_ID = "X-SL-Queue-Id"` (`app/email/headers.py:24`).

**Cause → effect.** Forwarding uses a **whitelist-strip-then-selectively-re-add** model. Any header
not on the small whitelist — including arbitrary `X-*` headers and the tracing `Received` header — is
deleted. The original `Reply-To` is likewise deleted, but because the sender set one, SimpleLogin
substitutes a **reverse-alias** `Reply-To` so that a mailbox reply routes back through SimpleLogin to
the real correspondent while preserving the alias's privacy.

---

## Q6 — Alias-creation suffix expiry window

**Restated question.** Experimentally determine the expiry window of the signed alias-creation suffix
by testing a token immediately, then after waiting, until the boundary where verification fails.

**Direct answer.** The window is **600 seconds**. Through the real `check_suffix_signature()`, a
signed suffix verifies successfully for ages **0–600 s inclusive** and fails (returns `None`) at
**601 s and beyond**. This was **stable across 3 runs**.

**Probe method (real signer only).** The probe imports the *actual* module used by the dashboard
alias flow (`app.alias_suffix`) and uses its module-level
`signer = itsdangerous.TimestampSigner(config.CUSTOM_ALIAS_SECRET)` (`app/alias_suffix.py:11`) and its
`check_suffix_signature()` (`app/alias_suffix.py:37-42`, which calls
`signer.unsign(signed_suffix, max_age=600)`). Tokens are minted with the real `signer.sign(...)`
(exactly as `get_alias_suffixes` does at `app/alias_suffix.py:114/128/155/185`).

- **Part A/C (600 s boundary):** because a real 10‑minute wait is impractical, the boundary is probed
  with an **accelerated clock** — `itsdangerous.timed.time` is shifted so that the *real*
  `signer.unsign(..., max_age=600)` reads a later "now" when computing the signature age. **The signer
  and `unsign` are fully executed** (HMAC verification + timestamp decode + `age > max_age` check);
  only the wall-clock reading is advanced. *(Labeled: accelerated technique.)*
- **Part B (real wall clock):** independently confirms the *same* `TimestampSigner` really expires in
  real time, using a **reduced** `max_age=3` window and an actual 4‑second `sleep` (labeled reduced
  window — production hardcodes `max_age=600`).

**Complete, unedited output.**

```
itsdangerous version                : 1.1.0
config.CUSTOM_ALIAS_SECRET          : 'secretcustom_alias'
als.signer is TimestampSigner       : True
als.signer.secret_key               : b'secretcustom_alias'

========================================================================
PART A/C — 600s boundary via REAL check_suffix_signature (clock accelerated)
========================================================================
[RUN 1] signed suffix (real signer.sign): .q6probe_word@sl.local.akwtag.xxsSRyayOwtiHrH1uFl_5K_Rneo
    age=   0s -> check_suffix_signature = '.q6probe_word@sl.local'     VALID (returns suffix)
    age= 300s -> check_suffix_signature = '.q6probe_word@sl.local'     VALID (returns suffix)
    age= 599s -> check_suffix_signature = '.q6probe_word@sl.local'     VALID (returns suffix)
    age= 600s -> check_suffix_signature = '.q6probe_word@sl.local'     VALID (returns suffix)
    age= 601s -> check_suffix_signature = None                         EXPIRED (returns None)
    age= 700s -> check_suffix_signature = None                         EXPIRED (returns None)
    => max VALID age observed = 600s ; first EXPIRED age = 601s

[RUN 2] signed suffix (real signer.sign): .q6probe_word@sl.local.akwtag.xxsSRyayOwtiHrH1uFl_5K_Rneo
    age=   0s -> check_suffix_signature = '.q6probe_word@sl.local'     VALID (returns suffix)
    age= 300s -> check_suffix_signature = '.q6probe_word@sl.local'     VALID (returns suffix)
    age= 599s -> check_suffix_signature = '.q6probe_word@sl.local'     VALID (returns suffix)
    age= 600s -> check_suffix_signature = '.q6probe_word@sl.local'     VALID (returns suffix)
    age= 601s -> check_suffix_signature = None                         EXPIRED (returns None)
    age= 700s -> check_suffix_signature = None                         EXPIRED (returns None)
    => max VALID age observed = 600s ; first EXPIRED age = 601s

[RUN 3] signed suffix (real signer.sign): .q6probe_word@sl.local.akwtag.xxsSRyayOwtiHrH1uFl_5K_Rneo
    age=   0s -> check_suffix_signature = '.q6probe_word@sl.local'     VALID (returns suffix)
    age= 300s -> check_suffix_signature = '.q6probe_word@sl.local'     VALID (returns suffix)
    age= 599s -> check_suffix_signature = '.q6probe_word@sl.local'     VALID (returns suffix)
    age= 600s -> check_suffix_signature = '.q6probe_word@sl.local'     VALID (returns suffix)
    age= 601s -> check_suffix_signature = None                         EXPIRED (returns None)
    age= 700s -> check_suffix_signature = None                         EXPIRED (returns None)
    => max VALID age observed = 600s ; first EXPIRED age = 601s

STABILITY across 3 runs: max-valid ages = 600, 600, 600  (all equal? True)

========================================================================
PART B — REAL WALL-CLOCK expiry of the SAME TimestampSigner (small window, real sleep)
========================================================================
  t=+0.0s  unsign(max_age=3) -> '.q6realwait@sl.local'  (VALID)
  sleeping 4 real seconds ...
  t=+4.0s unsign(max_age=3) raised SignatureExpired: Signature age 4 > 3 seconds
```

**Governing code.** `signer = itsdangerous.TimestampSigner(config.CUSTOM_ALIAS_SECRET)`
(`app/alias_suffix.py:11`); `check_suffix_signature()` returns
`signer.unsign(signed_suffix, max_age=600).decode()` and catches `itsdangerous.BadSignature` →
returns `None` (`app/alias_suffix.py:37-42`). `CUSTOM_ALIAS_SECRET = FLASK_SECRET + "custom_alias"`
(`app/config.py:201`), observed as `b'secretcustom_alias'`. The downstream caller flashes on
expiry: `custom_alias.py:90` `suffix = check_suffix_signature(...)`, `custom_alias.py:91` `if not
suffix:`, `custom_alias.py:93` `flash("Alias creation time is expired, please retry", "warning")`.

**Cause → effect.** The `max_age=600` argument to `TimestampSigner.unsign` defines a **600‑second**
validity window (itsdangerous rejects only when `age > max_age`, so age `600` is still valid and
`601` is the first rejection). Past the window, `SignatureExpired` (a `BadSignature` subclass) is
raised and swallowed to `None`, which drives the dashboard's "Alias creation time is expired, please
retry" flash. Part B shows the identical mechanism in real wall-clock time.


---

## Q7 — API-key usage statistics (before / during / after)

**Restated question.** Make several API calls with the same key and report which database columns are
updated and their observed values.

**Direct answer.** Exactly two columns of the `api_key` row are updated on each API-key-authenticated
call: **`times`** (incremented by 1 per call) and **`last_used`** (stamped to the call time). Over
**N=5** calls, `times` advanced from `0` to `5` and `last_used` moved to the last call's timestamp.
`code` and `sudo_mode_at` are **unchanged**. **Contrast:** calls made via the browser-session fallback
(Q1) do **not** touch these columns at all.

**Command(s) and complete, unedited output (before → during → after).**

```
===== Q7 BEFORE (api_key codeFF) =====
  code  | times | last_used | sudo_mode_at
--------+-------+-----------+--------------
 codeFF |     0 |           |
(1 row)

===== Make N=5 API-key-authenticated calls: GET /api/user_info  (Authentication: codeFF) =====
  call #1 -> HTTP 200
  call #2 -> HTTP 200
  call #3 -> HTTP 200
  --- DURING (after 3 calls) ---
codeFF|3|2026-07-06 22:34:57.815269
  call #4 -> HTTP 200
  call #5 -> HTTP 200

===== Q7 AFTER (api_key codeFF) =====
  code  | times |         last_used          | sudo_mode_at
--------+-------+----------------------------+--------------
 codeFF |     5 | 2026-07-06 22:34:57.947459 |
(1 row)
```

The SELECT command used before/during/after:

```bash
docker exec sl-postgres psql -U myuser -d simplelogin -c \
  "select code,times,last_used,sudo_mode_at from api_key where code='codeFF';"
# API calls:
for i in 1 2 3 4 5; do curl -sS -H "Authentication: codeFF" http://localhost:7777/api/user_info; done
```

**Contrast — session-fallback path does NOT bump the counters (complete, unedited):**

```
CONTRAST — session-fallback path (Q1) must NOT touch api_key columns
  codeFF BEFORE session-only calls : codeFF|5|2026-07-06 22:34:57.947459
  making 5 SESSION-ONLY calls to GET /api/user_info (cookie, NO Authentication header):
    call #1 -> HTTP 200
    call #2 -> HTTP 200
    call #3 -> HTTP 200
    call #4 -> HTTP 200
    call #5 -> HTTP 200
  codeFF AFTER  session-only calls : codeFF|5|2026-07-06 22:34:57.947459
  => times/last_used UNCHANGED by the session-fallback path (updates happen only on the API-key branch)
```

**Governing code.** In `authorize_request()`, the API-key branch runs
`api_key.last_used = arrow.now()`; `api_key.times += 1`; `Session.commit()`
(`app/api/base.py:30-32`) — reached only in the `else` (valid-key) branch, never in the
`current_user` fallback branch (`app/api/base.py:20-27`). The `ApiKey` model
(`app/models.py:2350`) declares `code` (`app/models.py:2356`), `last_used`
(`app/models.py:2358`, `ArrowType default=None`), `times` (`app/models.py:2359`,
`Integer default=0 nullable=False`), and `sudo_mode_at` (`app/models.py:2360`).

**Before → during → after.** `times`: `0 → 3 → 5` (advanced by exactly N=5); `last_used`:
`NULL → 2026-07-06 22:34:57.815269 → 2026-07-06 22:34:57.947459`; `code`/`sudo_mode_at`: unchanged.

**Cause → effect.** Every request authenticated by an API key increments `times` and stamps
`last_used` before dispatching the view, giving per-key usage telemetry. The session-fallback branch
returns before that code, so browser-session traffic leaves the `api_key` row untouched — hence the
sharp before/after contrast.

---

## Q8 — Login with wrong credentials (response + log)

**Restated question.** Report the exact log message(s) and HTTP response details produced when a
login is attempted with incorrect credentials.

**Direct answer.** A wrong-credentials login returns **`HTTP 200 OK`** and **re-renders the login
form** (no redirect) with a flashed error **`"Email or password incorrect"`** — it is deliberately
**not** a `401`. The failure branch emits **no dedicated application log line**; it flashes and
records a **NewRelic custom event** (`LoginEvent`, `action=failed`) that is not written to a local log
by default. The only local log for the request is SimpleLogin's generic `after_request` DEBUG line,
which records the request as status **`200`**.

**Command(s).**

```python
# /tmp/probe_q8.py (excerpt) — two failure variants
for email, pw in [("john@wick.com","WRONG-password-123"), ("nobody@nowhere.test","whatever")]:
    s = requests.Session(); csrf = get_csrf(s)
    r = s.post(f"{BASE}/auth/login", data={"csrf_token":csrf,"email":email,"password":pw}, allow_redirects=False)
```

**Complete, unedited output (HTTP response, both variants).**

```
======================================================================
Q8 variant: wrong password (existing user)
======================================================================
$ POST /auth/login  (email=john@wick.com, password=<wrong>)
HTTP status line: 200 OK
Location header : None  (None => no redirect => form re-render)
Flash message present in body: Email or password incorrect
  body snippet: <script>toastr.error("Email or password incorrect");</script>

======================================================================
Q8 variant: nonexistent user
======================================================================
$ POST /auth/login  (email=nobody@nowhere.test, password=<wrong>)
HTTP status line: 200 OK
Location header : None  (None => no redirect => form re-render)
Flash message present in body: Email or password incorrect
  body snippet: <script>toastr.error("Email or password incorrect");</script>
```

**Complete, unedited output (canonical local log for the two failed POSTs).** SimpleLogin's
`after_request` handler logs every non-static request at DEBUG level (`server.py:284`):

```
2026-07-06 22:35:40,282 - SL - DEBUG - 33646 - "server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 200, takes 0.23995661735534668
2026-07-06 22:35:40,294 - SL - DEBUG - 33646 - "server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 200, takes 0.0049304962158203125
```

Both lines record status **`200`** (not `401`). The wrong-credentials branch itself
(`app/auth/views/login.py:45-50`) writes **no** log line of its own.

**Supplementary observation (labeled non-default flag).** The canonical `gunicorn` command
(`Dockerfile:47`) sets **no** `--access-logfile`, so a classic access log is **off by default**. Run
with `--access-logfile -` on a separate port, the access line for the failed POST is:

```
127.0.0.1 - - [06/Jul/2026:22:36:27 +0000] "POST /auth/login HTTP/1.1" 200 7017 "-" "python-requests/2.31.0"
```

**Governing code.** `login()` failure branch: `app/auth/views/login.py:45`
`if not user or not user.check_password(form.password.data):`, then `app/auth/views/login.py:49`
`flash("Email or password incorrect", "error")` and `app/auth/views/login.py:50`
`LoginEvent(LoginEvent.ActionType.failed).send()`. With no redirect, control falls to
`render_template("auth/login.html", ...)` at `app/auth/views/login.py:74-82` → `200`. `LoginEvent.send()`
calls `newrelic.agent.record_custom_event("LoginEvent", {...})` (`app/events/auth_event.py:22-25`).
The generic request log is `LOG.d(...)` in `after_request()` (`server.py:284-292`). Route prefix
`/auth` from `app/auth/base.py:3-4`.

**Cause → effect.** SimpleLogin treats a bad login as a **form-validation outcome** — it re-renders
the form with a flashed error and HTTP `200`, rather than emitting an HTTP `401`. Failure telemetry
is a NewRelic custom event, not an application error line, so the only local evidence is the generic
`after_request` DEBUG line (and, if explicitly enabled, the gunicorn access line) — both showing
status `200`.

---

## Repository state (read-only guarantee)

This investigation modified **no** source file. The single net change to the repository is the
creation of this document, `blitzy/documentation/app_2cd6ee777f8c.md`. All probe scripts, the local
SMTP sink, the throwaway `/tmp/q5.env`, and all captured logs lived outside the repository tree
(under `/tmp/…`) and were removed after use; the extra processes started for Q5 (SMTP sink on `1025`,
second handler on `20382`) and Q8 (supplementary gunicorn on `7778`) were stopped, leaving only the
canonical web app (`:7777`) and inbound handler (`:20381`).

