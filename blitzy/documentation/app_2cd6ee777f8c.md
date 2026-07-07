# SimpleLogin Runtime-Behavior Investigation — Evidence-Backed Answers

This document answers **eight runtime-behavior questions (Q1–Q8)** about the **SimpleLogin** Flask
email-alias application. **Every answer was produced by actually building, running, and observing
the live system** — never from reading source code alone.

Each question is presented in the same fixed six-part structure, leading with the direct answer:

1. **Question (restated)** and the **direct answer**;
2. the **exact command(s)** executed, including the **complete, self-contained probe script**;
3. the **complete, unedited output** those commands produced (verbatim, in fenced blocks);
4. the governing **`file:line` + function** citation (full repository-relative paths);
5. **before / during / after** state where the behavior changes state (Q4, Q7);
6. the **cause → effect** rationale grounded in the cited code.

Findings that are unflattering — Q2's HTTP 500 `AttributeError`, Q4's absence of session-id
rotation — are reported exactly as observed and are **not** remediated: this is a strictly
read-only investigation, so the only committed artifact is this document.

> **Local-fixture & sensitive-value note (all displayed secrets are throwaway).**
> Every credential, secret, API key, session cookie, CSRF token, signature, and signing secret
> shown below is a **local, ephemeral fixture value** originating from the repository's canonical
> test configuration (`example.env`) and the `flask dummy-data` seed command. Specifically:
> the login `john@wick.com` / `password`, the API keys `code` and `codeFF`, `FLASK_SECRET=secret`
> (hence `CUSTOM_ALIAS_SECRET='secretcustom_alias'`), the database password `mypassword`, and every
> `slapp` session cookie / CSRF token / reverse-alias captured at runtime are **not production
> secrets**. They exist only inside this disposable investigation environment and are safe to
> display as evidence. Session cookies, session UUIDs, HMAC signatures, `itsdangerous` timestamps,
> and randomly-generated reverse-alias local-parts are **regenerated on every run**, so their exact
> bytes differ run-to-run; the *structure* and the *behavior* are the reproducible facts.

---

## Canonical runtime & invocation

**Environment (canonical, per repo).** Docker image
`andrewparkscaleai/coding-agent:simple-login__app__2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`.
Runtime versions match CI: **Python 3.10** (observed `Python 3.10.20`), **PostgreSQL 13**, **Redis 6**
(`.github/workflows/main.yml` service containers `postgres:13` and `redis:6`; `Dockerfile` base
`python:3.10`). All dependencies are the exact `poetry.lock` versions installed into the in-repo
virtualenv `.venv` (Flask 1.1.2, flask-login 0.5.0, werkzeug 1.0.1, itsdangerous 1.1.0, redis 4.6.0,
sqlalchemy 1.3.24, aiosmtpd 1.4.2, gunicorn 20.0.4).

**Configuration (default/canonical).** `.env` is derived from `example.env` (`cp example.env .env`)
with two required additions from the setup instructions: `MEM_STORE_URI=redis://localhost:6379`
(server-side Redis sessions are only enabled when this is set) and `GNUPGHOME=/tmp/gnupg`.
`example.env` supplies the rest, notably `URL=http://localhost:7777`, `EMAIL_DOMAIN=sl.local`,
`NOT_SEND_EMAIL=true`, `DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin`, and
`FLASK_SECRET=secret`.

**Exact build / migrate / seed / run commands (canonical).**

```bash
# backends (Docker, canonical versions)
docker run -d --name sl-postgres -e POSTGRES_USER=myuser -e POSTGRES_PASSWORD=mypassword \
    -e POSTGRES_DB=simplelogin -p 5432:5432 -p 15432:5432 postgres:13
docker run -d --name sl-redis -p 6379:6379 redis:6

# configuration
cp example.env .env
printf 'MEM_STORE_URI=redis://localhost:6379\nGNUPGHOME=/tmp/gnupg\n' >> .env

# schema (app DB) -> head 32f25cbf12f6, 77 tables
.venv/bin/alembic upgrade head

# seed data through real entry points
FLASK_APP=wsgi.py .venv/bin/flask dummy-data

# run the web application (canonical gunicorn command)
.venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15

# run the inbound SMTP handler (separate process, listens on :20381)
.venv/bin/python email_handler.py
```

**Seed fixtures (created by `flask dummy-data`, exercised below).** User `john@wick.com` /
`password` (activated); API keys `code` (Chrome) and `codeFF` (Firefox); alias
`begins_cashew220@sl.local` → mailbox `john@wick.com` (no PGP, used for the Q5 forward capture);
alias `example@example.com` → PGP mailbox. **API authentication uses the non-standard request header
`Authentication:` (not `Authorization:`)** — this is the SimpleLogin API contract
(`docs/api.md:73`); every API probe below sets that exact header.

**Foundation health observed at runtime:**

```text
$ docker exec sl-redis redis-cli ping
PONG
$ docker exec sl-postgres pg_isready
/var/run/postgresql:5432 - accepting connections
$ .venv/bin/alembic current
32f25cbf12f6 (head)
$ curl -s -o /dev/null -w "gunicorn :7777 -> HTTP %{http_code}\n" http://localhost:7777/
gunicorn :7777 -> HTTP 302
```


---

## Q1 — Calling an API endpoint without the `Authentication` header while logged in

**Question (restated).** When an API endpoint is invoked **without** the API-key header by a caller
who holds an authenticated browser session, what exact HTTP status code and response-body structure
come back? (And, for contrast, what happens with neither header nor session?)

**Direct answer.**
- **Authenticated browser session, no `Authentication` header → `HTTP 200 OK`** with the normal
  `user_info` JSON body. SimpleLogin falls back to the Flask-Login session (`current_user`).
- **No cookie and no header → `HTTP 401 UNAUTHORIZED`** with body `{"error":"Wrong api key"}`
  (`Content-Length: 26`).

**Command(s).** Self-contained probe (establishes a real login session with `requests`, then issues
the probe request over a **raw socket** so the complete wire response is captured verbatim):

```bash
.venv/bin/python /tmp/qprobes/probe_q1.py
```

```python
#!/usr/bin/env python3
"""Q1 probe: API call WITHOUT the `Authentication` header.
Condition (a): authenticated browser session (slapp cookie), NO Authentication header.
Condition (b): NO cookie AND NO Authentication header.
Uses a RAW socket for the probe request so the COMPLETE, unedited wire response
(status line + ALL headers + full body) is captured verbatim.
Run:  .venv/bin/python /tmp/qprobes/probe_q1.py
"""
import re
import socket
import requests

HOST, PORT = "localhost", 7777
BASE = f"http://{HOST}:{PORT}"


def raw_http(method, path, extra_headers=None):
    """Send one HTTP/1.1 request over a raw socket; return (request_text, raw_response_bytes)."""
    lines = [f"{method} {path} HTTP/1.1", f"Host: {HOST}:{PORT}", "Connection: close"]
    for k, v in (extra_headers or {}).items():
        lines.append(f"{k}: {v}")
    req = "\r\n".join(lines) + "\r\n\r\n"
    s = socket.create_connection((HOST, PORT), timeout=10)
    s.sendall(req.encode())
    buf = b""
    while True:
        chunk = s.recv(4096)
        if not chunk:
            break
        buf += chunk
    s.close()
    return req, buf


# --- establish an authenticated browser session through the REAL login flow ---
sess = requests.Session()
login_page = sess.get(f"{BASE}/auth/login")
csrf = re.search(r'name="csrf_token"[^>]*value="([^"]+)"', login_page.text).group(1)
lr = sess.post(
    f"{BASE}/auth/login",
    data={"csrf_token": csrf, "email": "john@wick.com", "password": "password"},
    allow_redirects=False,
)
slapp = sess.cookies.get("slapp")

print("=" * 70)
print("Q1(a): authenticated browser session, NO Authentication header")
print("=" * 70)
print(f"[login POST] status={lr.status_code} location={lr.headers.get('Location')}")
print(f"[login POST] slapp cookie now: {slapp}")
print()
req_a, resp_a = raw_http("GET", "/api/user_info", {"Cookie": f"slapp={slapp}"})
print("---- RAW REQUEST ----")
print(req_a, end="")
print("---- RAW RESPONSE (complete, unedited) ----")
print(resp_a.decode("latin-1"))

print()
print("=" * 70)
print("Q1(b): NO cookie AND NO Authentication header")
print("=" * 70)
req_b, resp_b = raw_http("GET", "/api/user_info")
print("---- RAW REQUEST ----")
print(req_b, end="")
print("---- RAW RESPONSE (complete, unedited) ----")
print(resp_b.decode("latin-1"))
```

**Complete, unedited output.**

```text
======================================================================
Q1(a): authenticated browser session, NO Authentication header
======================================================================
[login POST] status=302 location=http://localhost:7777/dashboard/
[login POST] slapp cookie now: 3f28f927-8d86-4a22-9d55-95601b76e320.CSxXJmAxDdLp6vxXbbz-qRAWjxQ

---- RAW REQUEST ----
GET /api/user_info HTTP/1.1
Host: localhost:7777
Connection: close
Cookie: slapp=3f28f927-8d86-4a22-9d55-95601b76e320.CSxXJmAxDdLp6vxXbbz-qRAWjxQ

---- RAW RESPONSE (complete, unedited) ----
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Mon, 06 Jul 2026 23:12:36 GMT
Connection: close
Content-Type: application/json
Content-Length: 244
Access-Control-Allow-Origin: *
Set-Cookie: slapp=3f28f927-8d86-4a22-9d55-95601b76e320.CSxXJmAxDdLp6vxXbbz-qRAWjxQ; Expires=Mon, 13-Jul-2026 23:12:36 GMT; HttpOnly; Path=/; SameSite=Lax

{"can_create_reverse_alias":true,"connected_proton_address":null,"email":"john@wick.com","in_trial":false,"is_premium":true,"max_alias_free_plan":5,"name":"John Wick","profile_picture_url":"http://localhost:7777/static/upload/profile_pic.svg"}


======================================================================
Q1(b): NO cookie AND NO Authentication header
======================================================================
---- RAW REQUEST ----
GET /api/user_info HTTP/1.1
Host: localhost:7777
Connection: close

---- RAW RESPONSE (complete, unedited) ----
HTTP/1.1 401 UNAUTHORIZED
Server: gunicorn/20.0.4
Date: Mon, 06 Jul 2026 23:12:36 GMT
Connection: close
Content-Type: application/json
Content-Length: 26
Access-Control-Allow-Origin: *
Set-Cookie: slapp=9f03aa05-4974-45f5-8da1-ffc14797c634.baHj8RLGtlkwtAbuZT2DIkgAlIQ; Expires=Mon, 13-Jul-2026 23:12:36 GMT; HttpOnly; Path=/; SameSite=Lax

{"error":"Wrong api key"}
```

**Citation.** `app/api/base.py:16-27`, function `authorize_request()`: the API key is read from
the non-standard header at `app/api/base.py:17` (`api_code = request.headers.get("Authentication")`);
when it is absent/unknown the code falls back to the session at `app/api/base.py:20-25` (the nested
branch `if not api_key:` → `if current_user.is_authenticated:` → `g.user = current_user`); and only
when there is also no authenticated session
does it return `app/api/base.py:26-27` (`return jsonify(error="Wrong api key"), 401`). The endpoint
itself is `app/api/views/user_info.py:67` (`return jsonify(user_to_dict(user))`), whose payload is
built by `user_to_dict()` at `app/api/views/user_info.py:28-47`. The non-standard `Authentication`
header is the documented contract at `docs/api.md:73`.

**Cause → effect.** `authorize_request()` runs before every API view. With no `Authentication`
header, `ApiKey.get_by(code=None)` yields `None`, so the `if not api_key` branch executes and, if
`current_user` is authenticated (the browser session), it authorizes the request as that user — the
endpoint returns `200` with the ordinary JSON. Remove the session too and the same branch reaches
the `else` and returns the `401 {"error":"Wrong api key"}`. The status/body therefore depend purely
on whether a valid session survives the header's absence.

---

## Q2 — A privileged, sudo-gated operation with only a browser session

**Question (restated).** When a privileged, sudo-gated operation is attempted with only a browser
session, what **specific (non-standard) HTTP status code** and **exact error-message text** are
returned?

**Direct answer.** The only sudo-gated API endpoint is **`DELETE /api/user`**. Two distinct
conditions must be distinguished, and both are exercised below:
- **(a) Browser-session-only (the literal "only a browser session" case) → `HTTP 500 INTERNAL
  SERVER ERROR`, body `{"error":"Internal error"}` (`Content-Length: 27`).** This is *not* a clean
  authorization error: the sudo check dereferences `None`, raising
  `AttributeError: 'NoneType' object has no attribute 'sudo_mode_at'`, which the global error handler
  converts to a generic 500. **This is an observed defect; per the read-only mandate it is reported,
  not fixed.**
- **(b) Valid API key without recent sudo → `HTTP 440 UNKNOWN`, body `{"error":"Need sudo"}`
  (`Content-Length: 22`).** `440` is a **non-standard** status code (introduced by Microsoft IIS as
  "Login Timeout"), repurposed here with the unusual `"Need sudo"` message.

**Command(s).** Self-contained probe (raw socket for both variants) plus extraction of the complete
server-side traceback the 500 path emitted to the gunicorn log:

```bash
.venv/bin/python /tmp/qprobes/probe_q2.py
# the complete server-side traceback for variant (a), verbatim from the gunicorn log:
sed -n '31,$p' /tmp/qprobes/gunicorn.log
```

```python
#!/usr/bin/env python3
"""Q2 probe: sudo-gated DELETE /api/user under two conditions.
Variant (a): browser-session-only cookie, NO Authentication header
             -> current_user fallback sets g.api_key=None; require_api_sudo ->
                check_sudo_mode_is_active(None) dereferences None.sudo_mode_at ->
                AttributeError -> server error_handler -> HTTP 500 {"error":"Internal error"}.
Variant (b): valid API key (Authentication: code) WITHOUT recent sudo
             -> require_api_sudo returns HTTP 440 {"error":"Need sudo"} (non-standard IIS code).
Raw socket captures the COMPLETE, unedited wire response for each.
Run:  .venv/bin/python /tmp/qprobes/probe_q2.py
"""
import re
import socket
import requests

HOST, PORT = "localhost", 7777
BASE = f"http://{HOST}:{PORT}"


def raw_http(method, path, extra_headers=None):
    lines = [f"{method} {path} HTTP/1.1", f"Host: {HOST}:{PORT}", "Connection: close"]
    for k, v in (extra_headers or {}).items():
        lines.append(f"{k}: {v}")
    req = "\r\n".join(lines) + "\r\n\r\n"
    s = socket.create_connection((HOST, PORT), timeout=10)
    s.sendall(req.encode())
    buf = b""
    while True:
        chunk = s.recv(4096)
        if not chunk:
            break
        buf += chunk
    s.close()
    return req, buf


# --- authenticated browser session via the real login flow ---
sess = requests.Session()
login_page = sess.get(f"{BASE}/auth/login")
csrf = re.search(r'name="csrf_token"[^>]*value="([^"]+)"', login_page.text).group(1)
sess.post(
    f"{BASE}/auth/login",
    data={"csrf_token": csrf, "email": "john@wick.com", "password": "password"},
    allow_redirects=False,
)
slapp = sess.cookies.get("slapp")

print("=" * 70)
print("Q2(a): DELETE /api/user  browser-session-only (slapp cookie, NO Authentication header)")
print("=" * 70)
print(f"[session] slapp cookie: {slapp}")
print()
req_a, resp_a = raw_http("DELETE", "/api/user", {"Cookie": f"slapp={slapp}"})
print("---- RAW REQUEST ----")
print(req_a, end="")
print("---- RAW RESPONSE (complete, unedited) ----")
print(resp_a.decode("latin-1"))

print()
print("=" * 70)
print("Q2(b): DELETE /api/user  valid API key WITHOUT recent sudo (Authentication: code)")
print("=" * 70)
req_b, resp_b = raw_http("DELETE", "/api/user", {"Authentication": "code"})
print("---- RAW REQUEST ----")
print(req_b, end="")
print("---- RAW RESPONSE (complete, unedited) ----")
print(resp_b.decode("latin-1"))
```

**Complete, unedited output (both variants, raw wire responses).**

```text
======================================================================
Q2(a): DELETE /api/user  browser-session-only (slapp cookie, NO Authentication header)
======================================================================
[session] slapp cookie: 8eb27881-2c47-41eb-b6c4-ae11687cf824.nCMGKkLpUVDLbl1anawUQUCN5cE

---- RAW REQUEST ----
DELETE /api/user HTTP/1.1
Host: localhost:7777
Connection: close
Cookie: slapp=8eb27881-2c47-41eb-b6c4-ae11687cf824.nCMGKkLpUVDLbl1anawUQUCN5cE

---- RAW RESPONSE (complete, unedited) ----
HTTP/1.1 500 INTERNAL SERVER ERROR
Server: gunicorn/20.0.4
Date: Mon, 06 Jul 2026 23:13:00 GMT
Connection: close
Content-Type: application/json
Content-Length: 27
Access-Control-Allow-Origin: *
Set-Cookie: slapp=8eb27881-2c47-41eb-b6c4-ae11687cf824.nCMGKkLpUVDLbl1anawUQUCN5cE; Expires=Mon, 13-Jul-2026 23:13:00 GMT; HttpOnly; Path=/; SameSite=Lax

{"error":"Internal error"}


======================================================================
Q2(b): DELETE /api/user  valid API key WITHOUT recent sudo (Authentication: code)
======================================================================
---- RAW REQUEST ----
DELETE /api/user HTTP/1.1
Host: localhost:7777
Connection: close
Authentication: code

---- RAW RESPONSE (complete, unedited) ----
HTTP/1.1 440 UNKNOWN
Server: gunicorn/20.0.4
Date: Mon, 06 Jul 2026 23:13:00 GMT
Connection: close
Content-Type: application/json
Content-Length: 22
Access-Control-Allow-Origin: *
Set-Cookie: slapp=32256ddd-f04f-42ae-9ce9-c8151c51ced3.UtjXB_AhIYKgRpiefF2jwa9cl0Y; Expires=Mon, 13-Jul-2026 23:13:00 GMT; HttpOnly; Path=/; SameSite=Lax

{"error":"Need sudo"}
```

**Complete, unedited output (server-side traceback for variant (a), verbatim — full absolute
paths exactly as the `%(pathname)s` log format emits them).**

```text
2026-07-06 23:13:00,328 - SL - DEBUG - 64023 - "/tmp/blitzy/app/blitzy-f78a0548-c369-4cf4-92f5-e5a16d484468_00bd29/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.0016326904296875
2026-07-06 23:13:00,570 - SL - DEBUG - 64023 - "/tmp/blitzy/app/blitzy-f78a0548-c369-4cf4-92f5-e5a16d484468_00bd29/app/auth/views/login_utils.py:35" - after_login() -  - log user <User 1 John Wick john@wick.com> in
2026-07-06 23:13:00,570 - SL - DEBUG - 64023 - "/tmp/blitzy/app/blitzy-f78a0548-c369-4cf4-92f5-e5a16d484468_00bd29/app/auth/views/login_utils.py:44" - after_login() -  - redirect user to dashboard
2026-07-06 23:13:00,570 - SL - DEBUG - 64023 - "/tmp/blitzy/app/blitzy-f78a0548-c369-4cf4-92f5-e5a16d484468_00bd29/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 302, takes 0.23946857452392578
2026-07-06 23:13:00,577 - SL - ERROR - 64023 - "/tmp/blitzy/app/blitzy-f78a0548-c369-4cf4-92f5-e5a16d484468_00bd29/server.py:390" - error_handler() -  - 'NoneType' object has no attribute 'sudo_mode_at'
Traceback (most recent call last):
  File "/tmp/blitzy/app/blitzy-f78a0548-c369-4cf4-92f5-e5a16d484468_00bd29/.venv/lib/python3.10/site-packages/flask/app.py", line 1950, in full_dispatch_request
    rv = self.dispatch_request()
  File "/tmp/blitzy/app/blitzy-f78a0548-c369-4cf4-92f5-e5a16d484468_00bd29/.venv/lib/python3.10/site-packages/flask/app.py", line 1936, in dispatch_request
    return self.view_functions[rule.endpoint](**req.view_args)
  File "/tmp/blitzy/app/blitzy-f78a0548-c369-4cf4-92f5-e5a16d484468_00bd29/app/api/base.py", line 69, in decorated
    if not check_sudo_mode_is_active(g.api_key):
  File "/tmp/blitzy/app/blitzy-f78a0548-c369-4cf4-92f5-e5a16d484468_00bd29/app/api/base.py", line 47, in check_sudo_mode_is_active
    return api_key.sudo_mode_at and g.api_key.sudo_mode_at >= arrow.now().shift(
AttributeError: 'NoneType' object has no attribute 'sudo_mode_at'
2026-07-06 23:13:00,578 - SL - DEBUG - 64023 - "/tmp/blitzy/app/blitzy-f78a0548-c369-4cf4-92f5-e5a16d484468_00bd29/server.py:284" - after_request() -  - 127.0.0.1 DELETE /api/user ImmutableMultiDict([]) 500, takes 0.003824949264526367
2026-07-06 23:13:00,590 - SL - DEBUG - 64024 - "/tmp/blitzy/app/blitzy-f78a0548-c369-4cf4-92f5-e5a16d484468_00bd29/server.py:284" - after_request() -  - 127.0.0.1 DELETE /api/user ImmutableMultiDict([]) 440, takes 0.010100841522216797
```

**Citation.** Endpoint `app/api/views/user.py:12-14`, function `delete_user()`, decorated
`@require_api_sudo`. The guard is `app/api/base.py:63-73`, function `require_api_sudo` (the sudo
check at `app/api/base.py:69`, and `return jsonify(error="Need sudo"), 440` at `app/api/base.py:70`).
The dereference that fails on `None` is `app/api/base.py:46-49`, function
`check_sudo_mode_is_active` (`api_key.sudo_mode_at` at `app/api/base.py:47`). The 500 translation is
`server.py:388-394`, function `error_handler` (`LOG.e(e)` at `server.py:390`; the API branch
`return jsonify(error="Internal error"), 500` at `server.py:391-392`).

**Cause → effect.**
- **(a)** With no `Authentication` header, `authorize_request()` takes the session fallback and sets
  `g.api_key = None`. `require_api_sudo` then calls `check_sudo_mode_is_active(g.api_key)` →
  `None.sudo_mode_at`, raising `AttributeError`. The traceback confirms the chain
  `app/api/base.py:69 → app/api/base.py:47`. `error_handler` catches it, logs it, and (because the
  path starts with `/api/`) returns the generic `500 {"error":"Internal error"}`. So "only a browser
  session" produces a **server error**, not an authorization error.
- **(b)** With a real API key that has no recent sudo timestamp, `check_sudo_mode_is_active` returns a
  falsy value cleanly, so `require_api_sudo` returns the intended `440 {"error":"Need sudo"}`.

---

## Q3 — Session data format in the storage backend

**Question (restated).** What are the **raw bytes** persisted in the session store, what is the
**serialization format**, which **keys** are present in a deserialized authenticated session, and how
is the storage **key** itself structured?

**Direct answer.** Sessions are stored **server-side in Redis** by a custom `RedisSessionStore`.
- **Storage key structure:** `session:<uuid4>` (e.g. `session:b3f84b92-079d-402f-b188-b21e4f9a09fa`).
- **Serialization format:** Python **`pickle`**, **protocol 4** — the raw value begins with the
  bytes `\x80\x04` (`\x80` = `PROTO` opcode, `\x04` = protocol number), confirmed by
  `pickletools.dis`.
- **Deserialized authenticated-session keys:** `_permanent`, `_fresh`, `csrf_token`, `_user_id`,
  `_id`, `sudo_time` (six keys). A non-authenticated session holds only `_permanent`, `_fresh`,
  `csrf_token` (no `_user_id`).
- **Cookie structure:** the browser cookie `slapp` = `<uuid4>.<hmac-signature>`; only the signed
  identifier lives in the cookie — the pickled data stays in Redis.
- **TTL:** `604800` s (7 days) once authenticated; `300` s for an anonymous session.

**Command(s).** Two self-contained probes read Redis **directly** (via `redis-py`) to capture exactly
what `RedisSessionStore` wrote — the authenticated session, then the non-authenticated contrast:

```bash
.venv/bin/python /tmp/qprobes/probe_q3.py
.venv/bin/python /tmp/qprobes/probe_q3_nonauth.py
```

```python
#!/usr/bin/env python3
"""Q3 probe: session data format in the storage backend (Redis).
Captures: the storage KEY structure, the RAW persisted bytes (complete repr),
the serialization format (pickle + protocol byte proof), the deserialized KEYS
of an authenticated session, the cookie structure, and the TTL.
Reads Redis DIRECTLY (redis-py) to observe exactly what RedisSessionStore wrote.
Run:  .venv/bin/python /tmp/qprobes/probe_q3.py
"""
import re
import pickle
import pickletools
import requests
import redis

HOST, PORT = "localhost", 7777
BASE = f"http://{HOST}:{PORT}"

# --- authenticate through the real login flow ---
sess = requests.Session()
login_page = sess.get(f"{BASE}/auth/login")
csrf = re.search(r'name="csrf_token"[^>]*value="([^"]+)"', login_page.text).group(1)
sess.post(
    f"{BASE}/auth/login",
    data={"csrf_token": csrf, "email": "john@wick.com", "password": "password"},
    allow_redirects=False,
)
slapp = sess.cookies.get("slapp")
print("=" * 70)
print("Q3: authenticated session storage in Redis")
print("=" * 70)
print(f"[cookie] slapp = {slapp!r}")
session_id = slapp.split(".", 1)[0]
signature = slapp.split(".", 1)[1]
print(f"[cookie] structure = <uuid4>.<hmac-signature>")
print(f"[cookie]   uuid4 part      = {session_id}")
print(f"[cookie]   signature part  = {signature}")

# --- read Redis DIRECTLY ---
r = redis.from_url("redis://localhost:6379")
storage_key = f"session:{session_id}"
print()
print(f"[redis] storage KEY structure = 'session:<uuid4>'")
print(f"[redis] this session's KEY    = {storage_key}")
print(f"[redis] KEYS session:* (matching this session):")
for k in r.keys(f"session:{session_id}*"):
    print(f"           {k!r}  ttl={r.ttl(k)}")

raw = r.get(storage_key)
print()
print("[redis] RAW persisted VALUE (complete, unedited repr):")
print(repr(raw))
print()
print(f"[redis] raw length = {len(raw)} bytes")
print(f"[redis] first two bytes = {raw[:2]!r}  (0x80 = PROTO opcode; second byte = protocol number)")
print(f"[redis] pickle protocol number = {raw[1]}")

print()
print("[pickle] pickletools.dis of first opcodes (proof of pickle framing):")
import io
buf = io.StringIO()
pickletools.dis(raw, annotate=1, out=buf)
# print only the first several opcode lines (proof of PROTO/FRAME); full stream is long
for line in buf.getvalue().splitlines()[:6]:
    print("   " + line)

obj = pickle.loads(raw)
print()
print(f"[pickle] pickle.loads(raw) type = {type(obj).__name__}")
print(f"[pickle] deserialized KEYS = {sorted(obj.keys())}")
print("[pickle] full deserialized dict (values shown verbatim):")
for k in sorted(obj.keys()):
    print(f"           {k!r}: {obj[k]!r}")
```

```python
#!/usr/bin/env python3
"""Q3 supplement: contrast an UNAUTHENTICATED session with the authenticated one.
Shows the ttl=300 branch (app/session.py:95-96) and the reduced key-set (no _user_id).
Run:  .venv/bin/python /tmp/qprobes/probe_q3_nonauth.py
"""
import pickle
import requests
import redis

BASE = "http://localhost:7777"
sess = requests.Session()
sess.get(f"{BASE}/auth/login")  # anonymous request -> anonymous session w/ CSRF token
slapp = sess.cookies.get("slapp")
sid = slapp.split(".", 1)[0]
r = redis.from_url("redis://localhost:6379")
key = f"session:{sid}"
raw = r.get(key)
print(f"[non-auth] cookie slapp        = {slapp}")
print(f"[non-auth] storage key         = {key}")
print(f"[non-auth] TTL (seconds)       = {r.ttl(key)}   (app/session.py:95-96: 300 when '_user_id' not in session)")
print(f"[non-auth] raw bytes           = {raw!r}")
obj = pickle.loads(raw)
print(f"[non-auth] deserialized KEYS   = {sorted(obj.keys())}   ('_user_id' absent => not logged in)")
```

**Complete, unedited output (authenticated session).**

```text
======================================================================
Q3: authenticated session storage in Redis
======================================================================
[cookie] slapp = 'b3f84b92-079d-402f-b188-b21e4f9a09fa.pkIYe_BWEsEUNkUZIVQ6f9XDS8w'
[cookie] structure = <uuid4>.<hmac-signature>
[cookie]   uuid4 part      = b3f84b92-079d-402f-b188-b21e4f9a09fa
[cookie]   signature part  = pkIYe_BWEsEUNkUZIVQ6f9XDS8w

[redis] storage KEY structure = 'session:<uuid4>'
[redis] this session's KEY    = session:b3f84b92-079d-402f-b188-b21e4f9a09fa
[redis] KEYS session:* (matching this session):
           b'session:b3f84b92-079d-402f-b188-b21e4f9a09fa'  ttl=604800

[redis] RAW persisted VALUE (complete, unedited repr):
b'\x80\x04\x95!\x01\x00\x00\x00\x00\x00\x00}\x94(\x8c\n_permanent\x94\x88\x8c\x06_fresh\x94\x88\x8c\ncsrf_token\x94\x8c(0e46ba6caceefedfdbf1d12508bc0d955872f683\x94\x8c\x08_user_id\x94\x8c$8a0e2333-a5e9-4447-b149-7584af84be26\x94\x8c\x03_id\x94\x8c\x80f4a4143536f3e6712e19e6ce901a12f1ff7e8c98d04dd3e41c746e85093b48a1bfaa93650d1759a0cb7f13cba57b7f96e40ed981f0c49af1cb94f9905ee1dd03\x94\x8c\tsudo_time\x94J\xa66Lju.'

[redis] raw length = 300 bytes
[redis] first two bytes = b'\x80\x04'  (0x80 = PROTO opcode; second byte = protocol number)
[redis] pickle protocol number = 4

[pickle] pickletools.dis of first opcodes (proof of pickle framing):
       0: \x80 PROTO      4 Protocol version indicator.
       2: \x95 FRAME      289 Indicate the beginning of a new frame.
      11: }    EMPTY_DICT     Push an empty dict.
      12: \x94 MEMOIZE    (as 0) Store the stack top into the memo.  The stack is not popped.
      13: (    MARK              Push markobject onto the stack.
      14: \x8c     SHORT_BINUNICODE '_permanent' Push a Python Unicode string object.

[pickle] pickle.loads(raw) type = dict
[pickle] deserialized KEYS = ['_fresh', '_id', '_permanent', '_user_id', 'csrf_token', 'sudo_time']
[pickle] full deserialized dict (values shown verbatim):
           '_fresh': True
           '_id': 'f4a4143536f3e6712e19e6ce901a12f1ff7e8c98d04dd3e41c746e85093b48a1bfaa93650d1759a0cb7f13cba57b7f96e40ed981f0c49af1cb94f9905ee1dd03'
           '_permanent': True
           '_user_id': '8a0e2333-a5e9-4447-b149-7584af84be26'
           'csrf_token': '0e46ba6caceefedfdbf1d12508bc0d955872f683'
           'sudo_time': 1783379622
```

**Complete, unedited output (non-authenticated session contrast — `ttl=300`, no `_user_id`).**

```text
[non-auth] cookie slapp        = 22cfc35b-79d1-4e9a-8cf5-ec6abadb2876.-_s0zlsD-MX76pVHQqIZUoCAIzY
[non-auth] storage key         = session:22cfc35b-79d1-4e9a-8cf5-ec6abadb2876
[non-auth] TTL (seconds)       = 300   (app/session.py:95-96: 300 when '_user_id' not in session)
[non-auth] raw bytes           = b'\x80\x04\x95U\x00\x00\x00\x00\x00\x00\x00}\x94(\x8c\n_permanent\x94\x88\x8c\x06_fresh\x94\x89\x8c\ncsrf_token\x94\x8c(f0dbd2da6fbadc7e255d8bf94a82809f1b3d8a0a\x94u.'
[non-auth] deserialized KEYS   = ['_fresh', '_permanent', 'csrf_token']   ('_user_id' absent => not logged in)
```

**Citation.** `app/session.py`, class `RedisSessionStore`: `SESSION_PREFIX = "session"` at
`app/session.py:18`; the signer `itsdangerous.Signer(app.secret_key, salt="session",
key_derivation="hmac")` at `app/session.py:38-41`; the Redis key builder
`f"{SESSION_PREFIX}:{session_id}"` at `app/session.py:44-45`; `open_session` at
`app/session.py:68-80`; and `save_session`, which writes `pickle.dumps(dict(session))` at
`app/session.py:91`, chooses `ttl = 300 if "_user_id" not in session` else 7 days at
`app/session.py:95-96`, and signs + sets the cookie at `app/session.py:102-114`. The store is bound
to Flask at `app/redis_services.py:12`
(`app.session_interface = RedisSessionStore(storage.storage, storage.storage, app)`); the cookie
name is `app/config.py:199` (`SESSION_COOKIE_NAME = "slapp"`); the signing secret and cookie policy
(`app.secret_key`, `SESSION_COOKIE_SAMESITE = "Lax"`, 7-day `permanent_session_lifetime`) are applied
in `server.py`.

**Cause → effect.** On save, `RedisSessionStore.save_session` pickles the whole session dict and
stores it under `session:<uuid4>`, keeping only the HMAC-signed UUID in the `slapp` cookie — so the
raw bytes in Redis are a pickle stream (`\x80\x04…`) and the cookie is `<uuid4>.<signature>`. The
`_user_id`/`_fresh`/`_id` keys are written by Flask-Login on authentication and `csrf_token` by
Flask-WTF; because `_user_id` is present only once logged in, it is exactly the field that flips the
TTL from `300` to `604800`.

---

## Q4 — Session identifier behavior across authentication (session-fixation)

**Question (restated).** Capture the session identifier from the cookie **before** login, drive the
full login flow, capture it **after** login, and report whether the identifier is **preserved** or
**replaced**.

**Direct answer.** The identifier is **PRESERVED** — not rotated. The embedded `uuid4` *and* the
entire signed `slapp` cookie are **byte-identical** before and after a successful login. Because the
pre-authentication session id survives into the authenticated session, SimpleLogin is **exposed to
session fixation**. The result was **stable across 2 independent runs**. (Observed defect; reported,
not remediated.)

**Command(s).** Self-contained probe; two runs, each capturing the `slapp` cookie before the login
POST and after it, over the same cookie jar:

```bash
.venv/bin/python /tmp/qprobes/probe_q4.py
```

```python
#!/usr/bin/env python3
"""Q4 probe: session identifier behavior across authentication (session-fixation test).
For each run: capture the slapp cookie (and its embedded uuid4) BEFORE login on an
unauthenticated request, drive the REAL login POST, then capture the slapp cookie
AFTER login. Compare the uuid4 identifiers to determine reuse vs. rotation.
Repeated across 2 independent runs to confirm stability.
Run:  .venv/bin/python /tmp/qprobes/probe_q4.py
"""
import re
import requests

BASE = "http://localhost:7777"


def one_run(run_no):
    print("=" * 70)
    print(f"RUN {run_no}")
    print("=" * 70)
    sess = requests.Session()

    # --- BEFORE: unauthenticated GET establishes an anonymous session + CSRF token ---
    r1 = sess.get(f"{BASE}/auth/login")
    set_cookie_before = r1.headers.get("Set-Cookie")
    slapp_before = sess.cookies.get("slapp")
    uuid_before = slapp_before.split(".", 1)[0]
    print("[BEFORE login]")
    print(f"   Set-Cookie header : {set_cookie_before}")
    print(f"   slapp cookie      : {slapp_before}")
    print(f"   embedded uuid4    : {uuid_before}")

    csrf = re.search(r'name="csrf_token"[^>]*value="([^"]+)"', r1.text).group(1)

    # --- login POST reusing the SAME session/cookie jar ---
    r2 = sess.post(
        f"{BASE}/auth/login",
        data={"csrf_token": csrf, "email": "john@wick.com", "password": "password"},
        allow_redirects=False,
    )
    set_cookie_after = r2.headers.get("Set-Cookie")
    slapp_after = sess.cookies.get("slapp")
    uuid_after = slapp_after.split(".", 1)[0]
    print(f"[login POST] status={r2.status_code} location={r2.headers.get('Location')}")
    print("[AFTER login]")
    print(f"   Set-Cookie header : {set_cookie_after}")
    print(f"   slapp cookie      : {slapp_after}")
    print(f"   embedded uuid4    : {uuid_after}")

    print()
    print(f"   uuid4 BEFORE == uuid4 AFTER ? {uuid_before == uuid_after}")
    print(f"   full slapp BEFORE == full slapp AFTER ? {slapp_before == slapp_after}")
    print(f"   => identifier {'PRESERVED (no rotation)' if uuid_before == uuid_after else 'REPLACED (rotated)'}")
    print()
    return uuid_before, uuid_after


results = [one_run(1), one_run(2)]
print("=" * 70)
print("STABILITY ACROSS RUNS")
print("=" * 70)
for i, (b, a) in enumerate(results, 1):
    print(f"   run {i}: before={b}  after={a}  preserved={b == a}")
```

**Complete, unedited output (before/after, 2 runs).**

```text
======================================================================
RUN 1
======================================================================
[BEFORE login]
   Set-Cookie header : slapp=c51e84de-24eb-41ca-aae3-c1efcc5094c2.dphe7oRS6e9EidLG8iXqG7vNko0; Expires=Mon, 13-Jul-2026 23:14:28 GMT; HttpOnly; Path=/; SameSite=Lax
   slapp cookie      : c51e84de-24eb-41ca-aae3-c1efcc5094c2.dphe7oRS6e9EidLG8iXqG7vNko0
   embedded uuid4    : c51e84de-24eb-41ca-aae3-c1efcc5094c2
[login POST] status=302 location=http://localhost:7777/dashboard/
[AFTER login]
   Set-Cookie header : slapp=c51e84de-24eb-41ca-aae3-c1efcc5094c2.dphe7oRS6e9EidLG8iXqG7vNko0; Expires=Mon, 13-Jul-2026 23:14:29 GMT; HttpOnly; Path=/; SameSite=Lax
   slapp cookie      : c51e84de-24eb-41ca-aae3-c1efcc5094c2.dphe7oRS6e9EidLG8iXqG7vNko0
   embedded uuid4    : c51e84de-24eb-41ca-aae3-c1efcc5094c2

   uuid4 BEFORE == uuid4 AFTER ? True
   full slapp BEFORE == full slapp AFTER ? True
   => identifier PRESERVED (no rotation)

======================================================================
RUN 2
======================================================================
[BEFORE login]
   Set-Cookie header : slapp=17f1bf0f-5a3d-4743-af26-5efc91dd3772.ED25ecbEmJoZehnZES9lnlMcdBA; Expires=Mon, 13-Jul-2026 23:14:29 GMT; HttpOnly; Path=/; SameSite=Lax
   slapp cookie      : 17f1bf0f-5a3d-4743-af26-5efc91dd3772.ED25ecbEmJoZehnZES9lnlMcdBA
   embedded uuid4    : 17f1bf0f-5a3d-4743-af26-5efc91dd3772
[login POST] status=302 location=http://localhost:7777/dashboard/
[AFTER login]
   Set-Cookie header : slapp=17f1bf0f-5a3d-4743-af26-5efc91dd3772.ED25ecbEmJoZehnZES9lnlMcdBA; Expires=Mon, 13-Jul-2026 23:14:29 GMT; HttpOnly; Path=/; SameSite=Lax
   slapp cookie      : 17f1bf0f-5a3d-4743-af26-5efc91dd3772.ED25ecbEmJoZehnZES9lnlMcdBA
   embedded uuid4    : 17f1bf0f-5a3d-4743-af26-5efc91dd3772

   uuid4 BEFORE == uuid4 AFTER ? True
   full slapp BEFORE == full slapp AFTER ? True
   => identifier PRESERVED (no rotation)

======================================================================
STABILITY ACROSS RUNS
======================================================================
   run 1: before=c51e84de-24eb-41ca-aae3-c1efcc5094c2  after=c51e84de-24eb-41ca-aae3-c1efcc5094c2  preserved=True
   run 2: before=17f1bf0f-5a3d-4743-af26-5efc91dd3772  after=17f1bf0f-5a3d-4743-af26-5efc91dd3772  preserved=True
```

**Before / after (identifier comparison).**

| Run | `uuid4` before login | `uuid4` after login | Preserved? |
|-----|----------------------|---------------------|------------|
| 1 | `c51e84de-24eb-41ca-aae3-c1efcc5094c2` | `c51e84de-24eb-41ca-aae3-c1efcc5094c2` | **yes** (full cookie byte-identical) |
| 2 | `17f1bf0f-5a3d-4743-af26-5efc91dd3772` | `17f1bf0f-5a3d-4743-af26-5efc91dd3772` | **yes** (full cookie byte-identical) |

**Citation.** `app/session.py:68-80`, function `open_session`: when the incoming cookie carries a
validly-signed identifier, that same `session_id` is reused to build the `ServerSession` (no new
UUID is minted). The login flow calls Flask-Login's `login_user()`, which writes `_user_id` into the
existing session dict but does **not** rotate the identifier. Flask-Login runs with
`session_protection = "strong"` at `app/extensions.py:8`, which regenerates the *remember-me* token
and `_fresh`/`_id` markers but does **not** change the server-side session id or the `slapp` UUID.

**Cause → effect.** Since `open_session` reuses the already-signed UUID and the login path never
calls a session-rotation primitive, the pre-login identifier persists verbatim into the
authenticated session. An attacker who fixes a victim's pre-auth `slapp` value would therefore retain
a valid identifier after the victim authenticates — the definition of session fixation.


---

## Q5 — Which forwarded email headers survive, and which are stripped

**Question (restated).** Send a real test email carrying (a) a **custom `X-` header**, (b) a
**`Received` header**, and (c) a **`Reply-To` header**, then examine the forwarded message and report
the fate of **each of the three named headers**.

**Direct answer (each named header).**
- **`X-Custom-Test` → STRIPPED.** It is absent from the forwarded message (not on the header
  whitelist).
- **`Received` → STRIPPED.** It is absent from the forwarded message (not on the whitelist).
- **`Reply-To` → REPLACED.** The sender's original `Reply-To` is removed by the whitelist, and
  because the inbound message carried a `Reply-To`, SimpleLogin re-adds a **new** `Reply-To` pointing
  at a **reverse-alias** (`…@sl.local`) so replies route back through SimpleLogin. The `From` header
  is likewise rewritten to a reverse-alias; `To`, `Subject`, and the MIME headers are preserved.

The forwarding transformation (whitelist + `From` rewrite + `Reply-To` re-add) runs **entirely
upstream** of mail transport, so it is identical whether the message is finally printed
(`NOT_SEND_EMAIL=true`) or actually sent. This is demonstrated below in **two complementary ways**.

### Q5(a) — Canonical default runtime (`NOT_SEND_EMAIL=true`)

Under the canonical configuration, the inbound handler on `:20381` performs the full transformation
and then hands the message to `MailSender.send()`, which — because `NOT_SEND_EMAIL=true` — logs a
**summary only** (subject/from/to) and returns without serializing the transformed message. This is a
**documented limitation of the default configuration**: the whitelist *decisions* are observable in
the handler's `DEBUG` log (decisively for `Reply-To`), but the full transformed header block is
**not emitted** on this path. The relevant code is `app/mail_sender.py:130-137` (the
`if config.NOT_SEND_EMAIL:` summary-log branch).

**Command(s).** Inject one crafted message (all three named headers) into the canonical handler and
read the handler's `DEBUG` log for that message:

```bash
.venv/bin/python /tmp/qprobes/probe_q5_craft.py 20381
# then read the handler DEBUG log lines emitted for this message:
sed -n '9,$p' /tmp/qprobes/email_handler.log
```

```python
#!/usr/bin/env python3
"""Q5 craft+send: build ONE test message carrying the three named headers
(a custom X- header, a Received header, and a Reply-To header) and inject it
into the running inbound SMTP handler for alias begins_cashew220@sl.local.
Usage:  .venv/bin/python probe_q5_craft.py <handler_port>
"""
import smtplib
import sys
from email.message import EmailMessage

HANDLER_PORT = int(sys.argv[1])
ENVELOPE_FROM = "alice@external.example"
ALIAS = "begins_cashew220@sl.local"

msg = EmailMessage()
msg["From"] = "Alice Sender <alice@external.example>"
msg["To"] = ALIAS
msg["Subject"] = "Q5 header-forwarding probe"
msg["X-Custom-Test"] = "custom-x-header-value-12345"
msg["Received"] = "from probe.example (probe.example [203.0.113.7]) by mx.local; Mon, 06 Jul 2026 00:00:00 +0000"
msg["Reply-To"] = "Alice Replyto <replyto@external.example>"
msg.set_content("Q5 body: verifying which headers survive forwarding.\n")

print("=" * 70)
print(f"CRAFTED INPUT MESSAGE (envelope MAIL FROM={ENVELOPE_FROM}, RCPT TO={ALIAS}, handler :{HANDLER_PORT})")
print("=" * 70)
print(msg.as_string())
print("=" * 70)
print("Named headers present in the INPUT message:")
print(f"   X-Custom-Test : {msg['X-Custom-Test']!r}")
print(f"   Received      : {msg['Received']!r}")
print(f"   Reply-To      : {msg['Reply-To']!r}")
print("=" * 70)

with smtplib.SMTP("127.0.0.1", HANDLER_PORT, timeout=20) as s:
    s.sendmail(ENVELOPE_FROM, [ALIAS], msg.as_bytes())
print(f"[sent] message injected into handler on port {HANDLER_PORT}")
```

**Complete, unedited output — the crafted INPUT message (all three named headers present).**

```text
======================================================================
CRAFTED INPUT MESSAGE (envelope MAIL FROM=alice@external.example, RCPT TO=begins_cashew220@sl.local, handler :20381)
======================================================================
From: Alice Sender <alice@external.example>
To: begins_cashew220@sl.local
Subject: Q5 header-forwarding probe
X-Custom-Test: custom-x-header-value-12345
Received: from probe.example (probe.example [203.0.113.7]) by mx.local; Mon,
 06 Jul 2026 00:00:00 +0000
Reply-To: Alice Replyto <replyto@external.example>
Content-Type: text/plain; charset="utf-8"
Content-Transfer-Encoding: 7bit
MIME-Version: 1.0

Q5 body: verifying which headers survive forwarding.

======================================================================
Named headers present in the INPUT message:
   X-Custom-Test : 'custom-x-header-value-12345'
   Received      : 'from probe.example (probe.example [203.0.113.7]) by mx.local; Mon, 06 Jul 2026 00:00:00 +0000'
   Reply-To      : 'Alice Replyto <replyto@external.example>'
======================================================================
[sent] message injected into handler on port 20381
```

**Complete, unedited output — the canonical handler `DEBUG` log for this message.** Note the
decisive line at `email_handler.py:873`: `Reply-To header, new:… old:None` — the `old:None` proves
the sender's original `Reply-To` was **already stripped** by the whitelist *before* the reverse-alias
`Reply-To` is re-added. The final `app/mail_sender.py:131` line is the summary-only log (the
documented limitation of the default path).

```text
2026-07-06 23:17:22,364 - SL - DEBUG - 64021 - "/tmp/blitzy/app/blitzy-f78a0548-c369-4cf4-92f5-e5a16d484468_00bd29/app/log.py:24" - set_message_id() -  - set message_id 4488c4f9-1605-4168-b46c-b9b18e4b4c34
2026-07-06 23:17:22,365 - SL - DEBUG - 64021 - "/tmp/blitzy/app/blitzy-f78a0548-c369-4cf4-92f5-e5a16d484468_00bd29/email_handler.py:2342" - _handle() - 4488c4f9-1605-4168-b46c-b9b18e4b4c34 - ====>=====>====>====>====>====>====>====>
2026-07-06 23:17:22,365 - SL - INFO - 64021 - "/tmp/blitzy/app/blitzy-f78a0548-c369-4cf4-92f5-e5a16d484468_00bd29/email_handler.py:2343" - _handle() - 4488c4f9-1605-4168-b46c-b9b18e4b4c34 - New message, mail from alice@external.example, rctp tos ['begins_cashew220@sl.local'] 
2026-07-06 23:17:22,366 - SL - DEBUG - 64021 - "/tmp/blitzy/app/blitzy-f78a0548-c369-4cf4-92f5-e5a16d484468_00bd29/email_handler.py:1963" - handle() - 4488c4f9-1605-4168-b46c-b9b18e4b4c34 - Cannot parse Postfix queue ID from ['from probe.example (probe.example [203.0.113.7]) by mx.local; Mon,\n 06 Jul 2026 00:00:00 +0000'] from probe.example (probe.example [203.0.113.7]) by mx.local; Mon,
 06 Jul 2026 00:00:00 +0000
2026-07-06 23:17:22,499 - SL - DEBUG - 64021 - "/tmp/blitzy/app/blitzy-f78a0548-c369-4cf4-92f5-e5a16d484468_00bd29/email_handler.py:1980" - handle() - 4488c4f9-1605-4168-b46c-b9b18e4b4c34 - ==>> Handle mail_from:alice@external.example, rcpt_tos:['begins_cashew220@sl.local'], header_from:Alice Sender <alice@external.example>, header_to:begins_cashew220@sl.local, cc:None, reply-to:Alice Replyto <replyto@external.example>, message_id:None, client_ip:None, headers:[('From', 'Alice Sender <alice@external.example>'), ('To', 'begins_cashew220@sl.local'), ('Subject', 'Q5 header-forwarding probe'), ('X-Custom-Test', 'custom-x-header-value-12345'), ('Received', 'from probe.example (probe.example [203.0.113.7]) by mx.local; Mon,\n 06 Jul 2026 00:00:00 +0000'), ('Reply-To', 'Alice Replyto <replyto@external.example>'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:['SIZE=455'], rcpt_options:[]
2026-07-06 23:17:22,503 - SL - DEBUG - 64021 - "/tmp/blitzy/app/blitzy-f78a0548-c369-4cf4-92f5-e5a16d484468_00bd29/email_handler.py:2202" - handle() - 4488c4f9-1605-4168-b46c-b9b18e4b4c34 - Forward phase alice@external.example(Alice Sender <alice@external.example>) -> begins_cashew220@sl.local
2026-07-06 23:17:22,519 - SL - DEBUG - 64021 - "/tmp/blitzy/app/blitzy-f78a0548-c369-4cf4-92f5-e5a16d484468_00bd29/email_handler.py:580" - handle_forward() - 4488c4f9-1605-4168-b46c-b9b18e4b4c34 - Create or get contact for from_header:Alice Sender <alice@external.example>
2026-07-06 23:17:22,543 - SL - DEBUG - 64021 - "/tmp/blitzy/app/blitzy-f78a0548-c369-4cf4-92f5-e5a16d484468_00bd29/app/contact_utils.py:110" - create_contact() - 4488c4f9-1605-4168-b46c-b9b18e4b4c34 - Created contact <Contact 6 alice@external.example 2> for alias <Alias 2 begins_cashew220@sl.local> with email alice@external.example invalid_email=False
2026-07-06 23:17:22,543 - SL - DEBUG - 64021 - "/tmp/blitzy/app/blitzy-f78a0548-c369-4cf4-92f5-e5a16d484468_00bd29/email_handler.py:589" - handle_forward() - 4488c4f9-1605-4168-b46c-b9b18e4b4c34 - Create or get contact for reply_to_header:Alice Replyto <replyto@external.example>
2026-07-06 23:17:22,564 - SL - DEBUG - 64021 - "/tmp/blitzy/app/blitzy-f78a0548-c369-4cf4-92f5-e5a16d484468_00bd29/app/contact_utils.py:110" - create_contact() - 4488c4f9-1605-4168-b46c-b9b18e4b4c34 - Created contact <Contact 7 replyto@external.example 2> for alias <Alias 2 begins_cashew220@sl.local> with email replyto@external.example invalid_email=False
2026-07-06 23:17:22,565 - SL - INFO - 64021 - "/tmp/blitzy/app/blitzy-f78a0548-c369-4cf4-92f5-e5a16d484468_00bd29/app/handler/dmarc.py:33" - apply_dmarc_policy_for_forward_phase() - 4488c4f9-1605-4168-b46c-b9b18e4b4c34 - DMARC check disabled
2026-07-06 23:17:22,572 - SL - DEBUG - 64021 - "/tmp/blitzy/app/blitzy-f78a0548-c369-4cf4-92f5-e5a16d484468_00bd29/email_handler.py:688" - forward_email_to_mailbox() - 4488c4f9-1605-4168-b46c-b9b18e4b4c34 - Forward <Contact 6 alice@external.example 2> -> <Alias 2 begins_cashew220@sl.local> -> <Mailbox 1 john@wick.com>
2026-07-06 23:17:22,576 - SL - DEBUG - 64021 - "/tmp/blitzy/app/blitzy-f78a0548-c369-4cf4-92f5-e5a16d484468_00bd29/email_handler.py:740" - forward_email_to_mailbox() - 4488c4f9-1605-4168-b46c-b9b18e4b4c34 - Create <EmailLog 4> for <Contact 6 alice@external.example 2>, <User 1 John Wick john@wick.com>, <Mailbox 1 john@wick.com>
2026-07-06 23:17:22,581 - SL - WARNING - 64021 - "/tmp/blitzy/app/blitzy-f78a0548-c369-4cf4-92f5-e5a16d484468_00bd29/email_handler.py:857" - forward_email_to_mailbox() - 4488c4f9-1605-4168-b46c-b9b18e4b4c34 - missing date header, create one
2026-07-06 23:17:22,581 - SL - DEBUG - 64021 - "/tmp/blitzy/app/blitzy-f78a0548-c369-4cf4-92f5-e5a16d484468_00bd29/email_handler.py:867" - forward_email_to_mailbox() - 4488c4f9-1605-4168-b46c-b9b18e4b4c34 - From header, new:"Alice Sender - alice at external.example" <alice_at_external_example_qqmjkiljq@sl.local>, old:Alice Sender <alice@external.example>
2026-07-06 23:17:22,582 - SL - DEBUG - 64021 - "/tmp/blitzy/app/blitzy-f78a0548-c369-4cf4-92f5-e5a16d484468_00bd29/email_handler.py:873" - forward_email_to_mailbox() - 4488c4f9-1605-4168-b46c-b9b18e4b4c34 - Reply-To header, new:"Alice Replyto - replyto at external.example" <replyto_at_external_example_mujeekpent@sl.local>, old:None
2026-07-06 23:17:22,582 - SL - DEBUG - 64021 - "/tmp/blitzy/app/blitzy-f78a0548-c369-4cf4-92f5-e5a16d484468_00bd29/email_handler.py:316" - replace_header_when_forward() - 4488c4f9-1605-4168-b46c-b9b18e4b4c34 - Delete Cc header, old value None
2026-07-06 23:17:22,582 - SL - DEBUG - 64021 - "/tmp/blitzy/app/blitzy-f78a0548-c369-4cf4-92f5-e5a16d484468_00bd29/email_handler.py:313" - replace_header_when_forward() - 4488c4f9-1605-4168-b46c-b9b18e4b4c34 - Replace To header, old: begins_cashew220@sl.local, new: begins_cashew220@sl.local
2026-07-06 23:17:22,582 - SL - INFO - 64021 - "/tmp/blitzy/app/blitzy-f78a0548-c369-4cf4-92f5-e5a16d484468_00bd29/app/handler/unsubscribe_generator.py:36" - _generate_header_with_original_behaviour() - 4488c4f9-1605-4168-b46c-b9b18e4b4c34 - Email has no unsubscribe header
2026-07-06 23:17:22,582 - SL - DEBUG - 64021 - "/tmp/blitzy/app/blitzy-f78a0548-c369-4cf4-92f5-e5a16d484468_00bd29/email_handler.py:893" - forward_email_to_mailbox() - 4488c4f9-1605-4168-b46c-b9b18e4b4c34 - Forward mail from alice@external.example to john@wick.com, mail_options:['SIZE=455'], rcpt_options:[] 
2026-07-06 23:17:22,583 - SL - DEBUG - 64021 - "/tmp/blitzy/app/blitzy-f78a0548-c369-4cf4-92f5-e5a16d484468_00bd29/app/mail_sender.py:131" - send() - 4488c4f9-1605-4168-b46c-b9b18e4b4c34 - send email with subject 'Q5 header-forwarding probe', from '"Alice Sender - alice at external.example" <alice_at_external_example_qqmjkiljq@sl.local>' to 'begins_cashew220@sl.local'
2026-07-06 23:17:22,583 - SL - INFO - 64021 - "/tmp/blitzy/app/blitzy-f78a0548-c369-4cf4-92f5-e5a16d484468_00bd29/email_handler.py:2367" - _handle() - 4488c4f9-1605-4168-b46c-b9b18e4b4c34 - Finish mail_from alice@external.example, rcpt_tos ['begins_cashew220@sl.local'], takes 0.21873164176940918 seconds with return code '250 Message accepted for delivery'<<===
```

### Q5(b) — Complete forwarded wire message (real `_send_to_smtp` delivery path)

To capture the **complete transformed header block** verbatim, the message is observed at the SMTP
delivery boundary. In production `NOT_SEND_EMAIL` is unset and `MailSender.send()` takes the real
`_send_to_smtp()` path (`app/mail_sender.py:144-177`), delivering to `POSTFIX_SERVER:POSTFIX_PORT`.
Here a local SMTP **sink** stands in for Postfix and records the exact bytes the handler forwards.
Because the header transformation happens **before** `send()` is ever called, these bytes are
byte-for-byte what canonical forwarding produces — this exercises the **real** delivery code path and
is *more* production-canonical than the summary-only default. The only non-default settings are the
disabling of `NOT_SEND_EMAIL` and pointing Postfix at the local sink; both are **explicitly labeled**
here.

**Command(s).** Start the sink and a second handler (on `:20382`, `NOT_SEND_EMAIL` disabled via a
transient config file that points Postfix at the sink), then inject the same crafted message:

```bash
# transient env file (outside the repo): copy .env, drop NOT_SEND_EMAIL, point Postfix at the sink
grep -v '^NOT_SEND_EMAIL=' .env > /tmp/qprobes/q5.env
printf 'POSTFIX_SERVER=127.0.0.1\nPOSTFIX_PORT=20999\n' >> /tmp/qprobes/q5.env

# start the SMTP sink (captures the full forwarded message bytes)
env -u NOT_SEND_EMAIL .venv/bin/python /tmp/qprobes/probe_q5_sink.py 20999 /tmp/qprobes/q5_wire_capture.eml &

# start a 2nd inbound handler on :20382 with NOT_SEND_EMAIL disabled, Postfix -> sink
env -u NOT_SEND_EMAIL CONFIG=/tmp/qprobes/q5.env .venv/bin/python email_handler.py --port 20382 &

# inject the same crafted message into the 2nd handler
.venv/bin/python /tmp/qprobes/probe_q5_craft.py 20382
```

```python
#!/usr/bin/env python3
"""Q5 SMTP sink: a minimal aiosmtpd server that stands in for Postfix at the
SMTP delivery boundary. It captures the COMPLETE raw bytes of the message the
inbound handler forwards (i.e. the fully-transformed forwarded message) and
writes them verbatim to <outfile>, then keeps running.
Usage:  .venv/bin/python probe_q5_sink.py <listen_port> <outfile>
"""
import asyncio
import sys
from aiosmtpd.controller import Controller


class CaptureHandler:
    def __init__(self, outfile):
        self.outfile = outfile

    async def handle_DATA(self, server, session, envelope):
        with open(self.outfile, "wb") as f:
            f.write(envelope.content)
        print(f"[sink] CAPTURED {len(envelope.content)} bytes  "
              f"mail_from={envelope.mail_from} rcpt_tos={envelope.rcpt_tos}", flush=True)
        return "250 Message accepted for delivery"


PORT = int(sys.argv[1])
OUTFILE = sys.argv[2]
controller = Controller(CaptureHandler(OUTFILE), hostname="127.0.0.1", port=PORT)
controller.start()
print(f"[sink] listening on 127.0.0.1:{PORT}, writing captures to {OUTFILE}", flush=True)
try:
    asyncio.get_event_loop().run_forever()
except KeyboardInterrupt:
    controller.stop()
```

**Complete, unedited output — the complete transformed forwarded message captured at the SMTP
boundary (verbatim wire bytes).**

```text
Subject: Q5 header-forwarding probe
Content-Type: text/plain; charset="utf-8"
Content-Transfer-Encoding: 7bit
MIME-Version: 1.0
X-SimpleLogin-Type: Forward
X-SimpleLogin-EmailLog-ID: 5
X-SimpleLogin-Envelope-From: alice@external.example
X-SimpleLogin-Original-From: Alice Sender <alice@external.example>
X-SimpleLogin-Envelope-To: begins_cashew220@sl.local
Date: Mon, 06 Jul 2026 23:18:13 -0000
From: "Alice Sender - alice at external.example"
 <alice_at_external_example_qqmjkiljq@sl.local>
Reply-To: "Alice Replyto - replyto at external.example"
 <replyto_at_external_example_mujeekpent@sl.local>
To: begins_cashew220@sl.local

Q5 body: verifying which headers survive forwarding.
```

In the captured wire message: **`X-Custom-Test` is absent** (stripped), **`Received` is absent**
(stripped), and **`Reply-To` is the reverse-alias** `…_mujeekpent@sl.local` (replaced) — while
`From` is a reverse-alias, and `To`/`Subject`/`Content-Type`/`Content-Transfer-Encoding`/`MIME-Version`
survive. (SimpleLogin also injects its own `X-SimpleLogin-*` control headers *downstream* of the
whitelist; arbitrary inbound `X-` headers like `X-Custom-Test` are still stripped. The reverse-alias
local-parts are randomly generated per contact, so they differ run-to-run; their `@sl.local`
structure is deterministic.)

**Citation.** The whitelist `headers_to_keep` is assembled at `email_handler.py:793-806` (plus
`headers.MIME_HEADERS` at `email_handler.py:806`) and applied by `delete_all_headers_except(msg,
headers_to_keep)` at `email_handler.py:810`; the generic stripper is
`app/email_utils.py:536-542`, function `delete_all_headers_except` (it lowercases the whitelist and
deletes every header not in it). Neither `Received` (`app/email/headers.py:14`) nor `Reply-To`
(`app/email/headers.py:13`) nor any custom `X-` header is on that whitelist. The `From` rewrite is
`email_handler.py:864-867` and the reverse-alias `Reply-To` re-add is `email_handler.py:869-873`
(guarded by `if reply_to_contact:`, created at `email_handler.py:586-594` only when the inbound
message carried a `Reply-To`). The summary-only default path is `app/mail_sender.py:130-137`; the
real send path is `app/mail_sender.py:144-177`, function `_send_to_smtp`.

**Cause → effect.** `delete_all_headers_except` keeps only whitelisted headers, so the custom
`X-Custom-Test` and the `Received` header are dropped, and the sender's `Reply-To` is dropped as
well. Because the inbound message *did* contain a `Reply-To`, a `reply_to_contact` was created, so
after the whitelist the handler re-adds a fresh `Reply-To` bearing that contact's reverse-alias — the
observed `old:None` at `email_handler.py:873` confirms the original was already gone at re-add time.
Net effect: `X-Custom-Test` stripped, `Received` stripped, `Reply-To` replaced.

---

## Q6 — Alias-creation token expiration window

**Question (restated).** Experimentally determine the expiry window of the signed alias-creation
suffix — test a token immediately, then after waiting, until the boundary where verification fails.

**Direct answer.** The window is **600 seconds**, inclusive of 600: a signed suffix verifies while
its age is **≤ 600 s** and fails once its age is **≥ 601 s**. This was **stable across 2 backdated
runs** and confirmed by a **real wall-clock** run (VALID at 598.1 s of real elapsed time, EXPIRED at
602.0 s). The window comes from the hard-coded `max_age=600` in `check_suffix_signature`.

**Command(s).** Two self-contained probes exercise the **real** signer
(`app.alias_suffix.signer`) and the **real** verifier (`app.alias_suffix.check_suffix_signature`).
The first sweeps ages across the boundary over two runs (age is simulated by driving the signer's own
clock at sign time; the *verification* is the fully-real entry point). The second is an unaccelerated
real-wall-clock confirmation.

```bash
PYTHONPATH=$PWD .venv/bin/python /tmp/qprobes/probe_q6_sweep.py
PYTHONPATH=$PWD .venv/bin/python -u /tmp/qprobes/probe_q6_realtime.py
```

```python
#!/usr/bin/env python3
"""Q6 boundary sweep: locate the alias-suffix expiry window using the REAL signer
(app.alias_suffix.signer) and the REAL verifier (app.alias_suffix.check_suffix_signature).
Token age is simulated by driving the signer's OWN clock (get_timestamp) at sign time;
verification runs through the real check_suffix_signature (max_age=600 hardcoded there).
Repeated across 2 runs to confirm the boundary is stable.
Run (from repo root, .env supplies CUSTOM_ALIAS_SECRET):
     .venv/bin/python /tmp/qprobes/probe_q6_sweep.py
"""
import itsdangerous
from app import config
from app.alias_suffix import signer, check_suffix_signature

print(f"itsdangerous version = {itsdangerous.__version__}")
print(f"config.CUSTOM_ALIAS_SECRET = {config.CUSTOM_ALIAS_SECRET!r}")
print(f"check_suffix_signature uses signer.unsign(..., max_age=600)  [app/alias_suffix.py:40]")
print()

SUFFIX = ".q6probe@sl.local"
AGES = [0, 300, 599, 600, 601, 700]

orig_get_timestamp = signer.get_timestamp


def sign_with_age(age_seconds):
    """Produce a REAL signature whose embedded timestamp is `age_seconds` in the past,
    by temporarily setting the signer's clock back; the signature itself is genuine."""
    base = orig_get_timestamp()
    signer.get_timestamp = lambda: base - age_seconds
    try:
        token = signer.sign(SUFFIX).decode()
    finally:
        signer.get_timestamp = orig_get_timestamp
    return token


for run in (1, 2):
    print("=" * 70)
    print(f"RUN {run}: age -> check_suffix_signature() result (valid returns suffix, expired returns None)")
    print("=" * 70)
    for age in AGES:
        token = sign_with_age(age)
        result = check_suffix_signature(token)
        verdict = "VALID  " if result is not None else "EXPIRED"
        print(f"   age={age:4d}s  -> {verdict}  return={result!r}")
    print()
```

```python
#!/usr/bin/env python3
"""Q6 real wall-clock confirmation (NO clock manipulation): sign a suffix with the
REAL signer now, then verify with the REAL check_suffix_signature immediately,
just before 600s, and just after 600s of ACTUAL elapsed time. Heartbeats every 60s.
Run:  .venv/bin/python -u /tmp/qprobes/probe_q6_realtime.py
"""
import time
from datetime import datetime, timezone
from app.alias_suffix import signer, check_suffix_signature

SUFFIX = ".q6realtime@sl.local"


def now():
    return datetime.now(timezone.utc).strftime("%Y-%m-%d %H:%M:%S UTC")


t0 = time.time()
token = signer.sign(SUFFIX).decode()
print(f"[{now()}] SIGNED token at t0 (real wall clock); token={token}", flush=True)


def wait_to(target_elapsed):
    while True:
        elapsed = time.time() - t0
        if elapsed >= target_elapsed:
            return elapsed
        time.sleep(min(60, target_elapsed - elapsed))
        print(f"[{now()}] ...heartbeat elapsed={time.time()-t0:6.1f}s", flush=True)


def check(label, target_elapsed):
    elapsed = wait_to(target_elapsed)
    result = check_suffix_signature(token)
    verdict = "VALID  " if result is not None else "EXPIRED"
    print(f"[{now()}] elapsed={elapsed:7.1f}s  {label:12s} -> {verdict}  return={result!r}", flush=True)


check("immediate", 0)
check("just-under", 598)
check("just-over", 602)
print(f"[{now()}] DONE", flush=True)
```

**Complete, unedited output — backdated boundary sweep (2 runs).**

```text
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
Upload files to local dir
>>> init logging <<<
2026-07-06 23:20:41,190 - SL - DEBUG - 71168 - "/tmp/blitzy/app/blitzy-f78a0548-c369-4cf4-92f5-e5a16d484468_00bd29/app/utils.py:17" - <module>() -  - load words file: /tmp/blitzy/app/blitzy-f78a0548-c369-4cf4-92f5-e5a16d484468_00bd29/local_data/test_words.txt
itsdangerous version = 1.1.0
config.CUSTOM_ALIAS_SECRET = 'secretcustom_alias'
check_suffix_signature uses signer.unsign(..., max_age=600)  [app/alias_suffix.py:40]

======================================================================
RUN 1: age -> check_suffix_signature() result (valid returns suffix, expired returns None)
======================================================================
   age=   0s  -> VALID    return='.q6probe@sl.local'
   age= 300s  -> VALID    return='.q6probe@sl.local'
   age= 599s  -> VALID    return='.q6probe@sl.local'
   age= 600s  -> VALID    return='.q6probe@sl.local'
   age= 601s  -> EXPIRED  return=None
   age= 700s  -> EXPIRED  return=None

======================================================================
RUN 2: age -> check_suffix_signature() result (valid returns suffix, expired returns None)
======================================================================
   age=   0s  -> VALID    return='.q6probe@sl.local'
   age= 300s  -> VALID    return='.q6probe@sl.local'
   age= 599s  -> VALID    return='.q6probe@sl.local'
   age= 600s  -> VALID    return='.q6probe@sl.local'
   age= 601s  -> EXPIRED  return=None
   age= 700s  -> EXPIRED  return=None
```

**Complete, unedited output — real wall-clock confirmation (~602 s, heartbeats every 60 s).**

```text
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
Upload files to local dir
>>> init logging <<<
2026-07-06 23:30:11,159 - SL - DEBUG - 73362 - "/tmp/blitzy/app/blitzy-f78a0548-c369-4cf4-92f5-e5a16d484468_00bd29/app/utils.py:17" - <module>() -  - load words file: /tmp/blitzy/app/blitzy-f78a0548-c369-4cf4-92f5-e5a16d484468_00bd29/local_data/test_words.txt
[2026-07-06 23:30:11 UTC] SIGNED token at t0 (real wall clock); token=.q6realtime@sl.local.akw6gw.i2ZTqb8P4eUBW5QURmYVHieyy2I
[2026-07-06 23:30:11 UTC] elapsed=    0.0s  immediate    -> VALID    return='.q6realtime@sl.local'
[2026-07-06 23:31:11 UTC] ...heartbeat elapsed=  60.0s
[2026-07-06 23:32:11 UTC] ...heartbeat elapsed= 120.1s
[2026-07-06 23:33:11 UTC] ...heartbeat elapsed= 180.1s
[2026-07-06 23:34:11 UTC] ...heartbeat elapsed= 240.2s
[2026-07-06 23:35:11 UTC] ...heartbeat elapsed= 300.2s
[2026-07-06 23:36:11 UTC] ...heartbeat elapsed= 360.3s
[2026-07-06 23:37:11 UTC] ...heartbeat elapsed= 420.3s
[2026-07-06 23:38:11 UTC] ...heartbeat elapsed= 480.4s
[2026-07-06 23:39:11 UTC] ...heartbeat elapsed= 540.4s
[2026-07-06 23:40:09 UTC] ...heartbeat elapsed= 598.1s
[2026-07-06 23:40:09 UTC] elapsed=  598.1s  just-under   -> VALID    return='.q6realtime@sl.local'
[2026-07-06 23:40:13 UTC] ...heartbeat elapsed= 602.0s
[2026-07-06 23:40:13 UTC] elapsed=  602.0s  just-over    -> EXPIRED  return=None
[2026-07-06 23:40:13 UTC] DONE
```

**Citation.** `app/alias_suffix.py:11` defines
`signer = itsdangerous.TimestampSigner(config.CUSTOM_ALIAS_SECRET)`; `app/alias_suffix.py:37-42`,
function `check_suffix_signature`, calls `signer.unsign(signed_suffix, max_age=600).decode()` at
`app/alias_suffix.py:40` and returns `None` on `itsdangerous.BadSignature` (of which
`SignatureExpired` is a subclass). The secret is `app/config.py:201`
(`CUSTOM_ALIAS_SECRET = FLASK_SECRET + "custom_alias"`, observed `'secretcustom_alias'`). On expiry
the caller flashes the message at `app/dashboard/views/custom_alias.py:90-93`
(`flash("Alias creation time is expired, please retry", "warning")`).

**Cause → effect & stability.** `TimestampSigner` embeds a Unix timestamp in the token; `unsign`
computes `age = now − embedded` and raises `SignatureExpired` when `age > max_age`. With
`max_age=600`, `age == 600` passes (`600 > 600` is false) and `age == 601` fails — exactly the
observed boundary. The backdated sweep reproduced the boundary identically in both runs, and the real
wall-clock run independently confirmed it (valid at 598.1 s, expired at 602.0 s), so the 600-second
window is stable and genuinely time-based rather than an artifact of clock manipulation.

---

## Q7 — API-key usage statistics

**Question (restated).** Make several API calls with the same key and report which database columns
are updated and their observed values.

**Direct answer.** On the **API-key** path, exactly two `api_key` columns update per call:
- **`times`** — incremented by **+1** on every API-key call (observed `0 → 1 → 2 → 3 → 4 → 5` over
  N = 5 calls);
- **`last_used`** — set to the current timestamp on every call (a fresh value each call; final
  observed value `2026-07-06 23:21:38.807932`).

`code` and `sudo_mode_at` are **unchanged**. Critically, the **session-fallback** path updates
**neither** column: an authenticated call with no `Authentication` header (Q1's `200` case) left the
`codeFF` row **byte-identical**.

**Command(s).** Self-contained probe: reset the key to a clean baseline, read the row **before**, make
N = 5 API-key calls reading the row **after each** (during), read the row **after**, then contrast
with a session-fallback call:

```bash
.venv/bin/python /tmp/qprobes/probe_q7.py
```

```python
#!/usr/bin/env python3
"""Q7 probe: which api_key columns update on API-key-authenticated calls, before/during/after.
Governing code: app/api/base.py:30-32 sets ApiKey.last_used=arrow.now(); ApiKey.times+=1; Session.commit()
on the valid-API-key path. Columns defined in app/models.py:2356-2360 (code, last_used, times, sudo_mode_at).
Also contrasts the SESSION-FALLBACK path (cookie, no Authentication header), which must NOT touch these columns.
Run:  .venv/bin/python /tmp/qprobes/probe_q7.py
"""
import re
import psycopg2
import requests

DB_URI = "postgresql://myuser:mypassword@localhost:5432/simplelogin"
BASE = "http://localhost:7777"
KEY = "codeFF"
N = 5

conn = psycopg2.connect(DB_URI)
conn.autocommit = True
cur = conn.cursor()

COLS = "code, times, last_used, sudo_mode_at"


def select_row(tag):
    sql = f"SELECT {COLS} FROM api_key WHERE code = '{KEY}';"
    cur.execute(sql)
    row = cur.fetchone()
    print(f"[SQL {tag}] {sql}")
    print(f"[ROW {tag}] code={row[0]!r} times={row[1]} last_used={row[2]!r} sudo_mode_at={row[3]!r}")
    return row


print("=" * 70)
print("SETUP: reset codeFF to canonical baseline (times=0, last_used=NULL)")
print("=" * 70)
reset_sql = f"UPDATE api_key SET times = 0, last_used = NULL WHERE code = '{KEY}';"
print(f"[SQL setup] {reset_sql}")
cur.execute(reset_sql)
print("[ok] reset committed (autocommit)")
print()

print("=" * 70)
print(f"BEFORE: no API calls yet")
print("=" * 70)
select_row("before")
print()

print("=" * 70)
print(f"DURING: {N} API-key calls (GET /api/user_info, header 'Authentication: {KEY}'), row after each")
print("=" * 70)
for i in range(1, N + 1):
    print(f"[CMD call {i}] GET {BASE}/api/user_info   Authentication: {KEY}")
    r = requests.get(f"{BASE}/api/user_info", headers={"Authentication": KEY})
    print(f"[HTTP call {i}] status={r.status_code}")
    select_row(f"after call {i}")
    print()

print("=" * 70)
print("AFTER: final api_key row")
print("=" * 70)
after = select_row("after")
print()

print("=" * 70)
print("CONTRAST: SESSION-FALLBACK path (browser cookie, NO Authentication header)")
print("must NOT change codeFF.times / codeFF.last_used")
print("=" * 70)
before_contrast = select_row("before session-call")
sess = requests.Session()
lp = sess.get(f"{BASE}/auth/login")
csrf = re.search(r'name="csrf_token"[^>]*value="([^"]+)"', lp.text).group(1)
sess.post(f"{BASE}/auth/login",
          data={"csrf_token": csrf, "email": "john@wick.com", "password": "password"},
          allow_redirects=False)
print(f"[CMD session-call] GET {BASE}/api/user_info   (Cookie: slapp=...; NO Authentication header)")
rc = sess.get(f"{BASE}/api/user_info")
print(f"[HTTP session-call] status={rc.status_code}  (authenticated via current_user fallback)")
after_contrast = select_row("after session-call")
print()
print(f"[RESULT] session-fallback changed codeFF row? "
      f"{before_contrast != after_contrast}  (expected: False / unchanged)")
```

**Complete, unedited output (setup → before → during ×5 → after → session-fallback contrast).**

```text
======================================================================
SETUP: reset codeFF to canonical baseline (times=0, last_used=NULL)
======================================================================
[SQL setup] UPDATE api_key SET times = 0, last_used = NULL WHERE code = 'codeFF';
[ok] reset committed (autocommit)

======================================================================
BEFORE: no API calls yet
======================================================================
[SQL before] SELECT code, times, last_used, sudo_mode_at FROM api_key WHERE code = 'codeFF';
[ROW before] code='codeFF' times=0 last_used=None sudo_mode_at=None

======================================================================
DURING: 5 API-key calls (GET /api/user_info, header 'Authentication: codeFF'), row after each
======================================================================
[CMD call 1] GET http://localhost:7777/api/user_info   Authentication: codeFF
[HTTP call 1] status=200
[SQL after call 1] SELECT code, times, last_used, sudo_mode_at FROM api_key WHERE code = 'codeFF';
[ROW after call 1] code='codeFF' times=1 last_used=datetime.datetime(2026, 7, 6, 23, 21, 38, 751344) sudo_mode_at=None

[CMD call 2] GET http://localhost:7777/api/user_info   Authentication: codeFF
[HTTP call 2] status=200
[SQL after call 2] SELECT code, times, last_used, sudo_mode_at FROM api_key WHERE code = 'codeFF';
[ROW after call 2] code='codeFF' times=2 last_used=datetime.datetime(2026, 7, 6, 23, 21, 38, 777186) sudo_mode_at=None

[CMD call 3] GET http://localhost:7777/api/user_info   Authentication: codeFF
[HTTP call 3] status=200
[SQL after call 3] SELECT code, times, last_used, sudo_mode_at FROM api_key WHERE code = 'codeFF';
[ROW after call 3] code='codeFF' times=3 last_used=datetime.datetime(2026, 7, 6, 23, 21, 38, 786816) sudo_mode_at=None

[CMD call 4] GET http://localhost:7777/api/user_info   Authentication: codeFF
[HTTP call 4] status=200
[SQL after call 4] SELECT code, times, last_used, sudo_mode_at FROM api_key WHERE code = 'codeFF';
[ROW after call 4] code='codeFF' times=4 last_used=datetime.datetime(2026, 7, 6, 23, 21, 38, 796301) sudo_mode_at=None

[CMD call 5] GET http://localhost:7777/api/user_info   Authentication: codeFF
[HTTP call 5] status=200
[SQL after call 5] SELECT code, times, last_used, sudo_mode_at FROM api_key WHERE code = 'codeFF';
[ROW after call 5] code='codeFF' times=5 last_used=datetime.datetime(2026, 7, 6, 23, 21, 38, 807932) sudo_mode_at=None

======================================================================
AFTER: final api_key row
======================================================================
[SQL after] SELECT code, times, last_used, sudo_mode_at FROM api_key WHERE code = 'codeFF';
[ROW after] code='codeFF' times=5 last_used=datetime.datetime(2026, 7, 6, 23, 21, 38, 807932) sudo_mode_at=None

======================================================================
CONTRAST: SESSION-FALLBACK path (browser cookie, NO Authentication header)
must NOT change codeFF.times / codeFF.last_used
======================================================================
[SQL before session-call] SELECT code, times, last_used, sudo_mode_at FROM api_key WHERE code = 'codeFF';
[ROW before session-call] code='codeFF' times=5 last_used=datetime.datetime(2026, 7, 6, 23, 21, 38, 807932) sudo_mode_at=None
[CMD session-call] GET http://localhost:7777/api/user_info   (Cookie: slapp=...; NO Authentication header)
[HTTP session-call] status=200  (authenticated via current_user fallback)
[SQL after session-call] SELECT code, times, last_used, sudo_mode_at FROM api_key WHERE code = 'codeFF';
[ROW after session-call] code='codeFF' times=5 last_used=datetime.datetime(2026, 7, 6, 23, 21, 38, 807932) sudo_mode_at=None

[RESULT] session-fallback changed codeFF row? False  (expected: False / unchanged)
```

**Before / during / after (`api_key WHERE code='codeFF'`).**

| Stage | `times` | `last_used` | `code` | `sudo_mode_at` |
|-------|---------|-------------|--------|----------------|
| before (reset) | `0` | `None` | `codeFF` | `None` |
| after call 1 | `1` | `2026-07-06 23:21:38.751344` | `codeFF` | `None` |
| after call 2 | `2` | `2026-07-06 23:21:38.777186` | `codeFF` | `None` |
| after call 3 | `3` | `2026-07-06 23:21:38.786816` | `codeFF` | `None` |
| after call 4 | `4` | `2026-07-06 23:21:38.796301` | `codeFF` | `None` |
| after call 5 | `5` | `2026-07-06 23:21:38.807932` | `codeFF` | `None` |
| after session-fallback call | `5` | `2026-07-06 23:21:38.807932` | `codeFF` | `None` |

**Citation.** `app/api/base.py:30-32`, in `authorize_request()` on the valid-API-key path:
`api_key.last_used = arrow.now()` at `app/api/base.py:30`, `api_key.times = api_key.times + 1` at
`app/api/base.py:31`, and `Session.commit()` at `app/api/base.py:32`. The columns are defined on the
model at `app/models.py:2350-2360`, class `ApiKey`: `code` (`app/models.py:2356`), `last_used`
(`app/models.py:2358`), `times` (`app/models.py:2359`), `sudo_mode_at` (`app/models.py:2360`).

**Cause → effect.** Those three statements execute only inside the `if api_key:` branch — i.e. only
when a valid `Authentication` header resolves to an `ApiKey`. Each API-key call therefore stamps
`last_used` and bumps `times` by one and commits, which is exactly the monotonic `0→5` progression
and per-call timestamps observed. The session-fallback branch never touches `api_key`, so a
cookie-authenticated call leaves `times`/`last_used` unchanged — proving these statistics track
**API-key** usage specifically, not user activity in general.

---

## Q8 — Login failure with wrong credentials

**Question (restated).** Report the exact log message(s) and HTTP response details produced when a
login is attempted with incorrect credentials.

**Direct answer.** A wrong-credentials login returns **`HTTP 200 OK`** and **re-renders the login
form** (it is **not** a `401`, and **not** a redirect). The page carries the flashed error
**`Email or password incorrect`**, emitted in the body as
`<script>toastr.error("Email or password incorrect");</script>`. The message is **identical** for a
wrong password on an existing user and for a non-existent user (no user enumeration). There is **no
dedicated application `ERROR` log line** for the failure — the failure path records a NewRelic custom
event `LoginEvent` with `action="failed"` (telemetry, not a log line). The observable log entry is
the werkzeug/`after_request` access line, which records status **`200`** for the `POST /auth/login`.

**Command(s).** Self-contained probe (raw socket, both wrong-credential variants; full responses also
saved to disk), followed by extraction of the `after_request` `DEBUG` access lines:

```bash
.venv/bin/python /tmp/qprobes/probe_q8.py
# the after_request access-log lines (full absolute paths, verbatim from the gunicorn log):
sed -n '70,$p' /tmp/qprobes/gunicorn.log | grep -E '/auth/login|after_request'
```

```python
#!/usr/bin/env python3
"""Q8 probe: login with WRONG credentials. Captures the COMPLETE raw HTTP response
(status line + ALL headers + full body) via a raw socket, for two variants:
  (a) existing user (john@wick.com) with a WRONG password
  (b) a NON-EXISTENT user
Governing code: app/auth/views/login.py:45 (if not user or not user.check_password),
:49 flash("Email or password incorrect","error"), :50 LoginEvent(LoginEvent.ActionType.failed).send(),
:74-82 returns render_template of the auth/login.html template. Failure re-renders with HTTP 200 (no 401/redirect).
Run:  .venv/bin/python /tmp/qprobes/probe_q8.py
"""
import re
import socket
import urllib.parse
import requests

HOST, PORT = "localhost", 7777
BASE = f"http://{HOST}:{PORT}"


def raw_post_form(path, cookie, form):
    body = urllib.parse.urlencode(form)
    req = (
        f"POST {path} HTTP/1.1\r\n"
        f"Host: {HOST}:{PORT}\r\n"
        f"Connection: close\r\n"
        f"Cookie: {cookie}\r\n"
        f"Content-Type: application/x-www-form-urlencoded\r\n"
        f"Content-Length: {len(body)}\r\n"
        f"\r\n"
        f"{body}"
    )
    s = socket.create_connection((HOST, PORT), timeout=10)
    s.sendall(req.encode())
    buf = b""
    while True:
        chunk = s.recv(4096)
        if not chunk:
            break
        buf += chunk
    s.close()
    return req, buf


def run_variant(label, email, password, outfile):
    # fresh session -> obtain CSRF token + slapp cookie via the real login page
    sess = requests.Session()
    lp = sess.get(f"{BASE}/auth/login")
    csrf = re.search(r'name="csrf_token"[^>]*value="([^"]+)"', lp.text).group(1)
    cookie = f"slapp={sess.cookies.get('slapp')}"
    req, resp = raw_post_form(
        "/auth/login", cookie,
        {"csrf_token": csrf, "email": email, "password": password},
    )
    with open(outfile, "wb") as f:
        f.write(resp)
    head, _, body = resp.partition(b"\r\n\r\n")
    print("=" * 70)
    print(f"Q8{label}: email={email!r} password={password!r}")
    print("=" * 70)
    print("---- RAW REQUEST (form body included) ----")
    print(req)
    print("---- RAW RESPONSE: status line + ALL headers (verbatim) ----")
    print(head.decode("latin-1"))
    print()
    print(f"---- RESPONSE BODY: total {len(body)} bytes; full copy saved to {outfile} ----")
    # show every line that contains the flash / error banner, verbatim (the relevant result)
    text = body.decode("utf-8", "replace")
    flash_lines = [ln.strip() for ln in text.splitlines()
                   if "incorrect" in ln.lower() or "alert" in ln.lower()]
    print("---- body lines containing the flash/error banner (verbatim) ----")
    for ln in flash_lines:
        print(ln)
    print(f"---- 'Email or password incorrect' present in body? "
          f"{'Email or password incorrect' in text} ----")
    print()


run_variant("(a) wrong password, existing user", "john@wick.com", "wrongpassword",
            "/tmp/qprobes/q8a_full_response.http")
run_variant("(b) non-existent user", "nobody@nowhere.example", "whatever",
            "/tmp/qprobes/q8b_full_response.http")
```

**Complete, unedited output (both variants: raw request, status + all headers, flash banner).**

```text
======================================================================
Q8(a) wrong password, existing user: email='john@wick.com' password='wrongpassword'
======================================================================
---- RAW REQUEST (form body included) ----
POST /auth/login HTTP/1.1
Host: localhost:7777
Connection: close
Cookie: slapp=d764bc41-a29a-4da5-935f-40f2d80a3ace.9G74XgV8_fVfUvt27cG4jA_hUSI
Content-Type: application/x-www-form-urlencoded
Content-Length: 147

csrf_token=ImRhZWVkZjMzZWEzOTUwMzYzOGQzODRlYTFjY2I0ODIzYTU0MzZjNDki.akw4sw.qZaqkuk5l-Es1C_5sMyvSyGvUgs&email=john%40wick.com&password=wrongpassword
---- RAW RESPONSE: status line + ALL headers (verbatim) ----
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Mon, 06 Jul 2026 23:22:27 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 7017
Set-Cookie: slapp=d764bc41-a29a-4da5-935f-40f2d80a3ace.9G74XgV8_fVfUvt27cG4jA_hUSI; Expires=Mon, 13-Jul-2026 23:22:27 GMT; HttpOnly; Path=/; SameSite=Lax

---- RESPONSE BODY: total 7017 bytes; full copy saved to /tmp/qprobes/q8a_full_response.http ----
---- body lines containing the flash/error banner (verbatim) ----
<script>toastr.error("Email or password incorrect");</script>
---- 'Email or password incorrect' present in body? True ----

======================================================================
Q8(b) non-existent user: email='nobody@nowhere.example' password='whatever'
======================================================================
---- RAW REQUEST (form body included) ----
POST /auth/login HTTP/1.1
Host: localhost:7777
Connection: close
Cookie: slapp=8ace1cbf-c01d-45a6-8c5c-64ca27d6c914.jIeHzsKUPVTHYccWS315dD5MhXo
Content-Type: application/x-www-form-urlencoded
Content-Length: 151

csrf_token=ImRmNGYzMzc5Yzg0YTJjYzg3NTBlNzg3NTVkMTg1OTE3NzgxOGRjYjEi.akw4sw.yobeJFxof7sXnEsGZ1AGx_vuPk0&email=nobody%40nowhere.example&password=whatever
---- RAW RESPONSE: status line + ALL headers (verbatim) ----
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Mon, 06 Jul 2026 23:22:27 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 7026
Set-Cookie: slapp=8ace1cbf-c01d-45a6-8c5c-64ca27d6c914.jIeHzsKUPVTHYccWS315dD5MhXo; Expires=Mon, 13-Jul-2026 23:22:27 GMT; HttpOnly; Path=/; SameSite=Lax

---- RESPONSE BODY: total 7026 bytes; full copy saved to /tmp/qprobes/q8b_full_response.http ----
---- body lines containing the flash/error banner (verbatim) ----
<script>toastr.error("Email or password incorrect");</script>
---- 'Email or password incorrect' present in body? True ----
```

**Complete, unedited output — full raw HTTP response for variant (a) (wrong password, existing
user): status line + all headers + the entire response body, verbatim.**

```text
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Mon, 06 Jul 2026 23:22:27 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 7017
Set-Cookie: slapp=d764bc41-a29a-4da5-935f-40f2d80a3ace.9G74XgV8_fVfUvt27cG4jA_hUSI; Expires=Mon, 13-Jul-2026 23:22:27 GMT; HttpOnly; Path=/; SameSite=Lax


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
    <!-- Bing -->
    <meta name="msvalidate.01" content="2A313A69CBFD1A378C3B91734DC221A8" />
    <!-- Yandex -->
    <meta name="yandex-verification" content="c9e5d4d68bc983a1" />
    <meta name="description"
          content="Protect your email address with email ALIAS. Create a different email alias for each website. No more phishing, or spam." />
    <link rel="icon" href="/static/favicon.ico" type="image/x-icon" />
    <link rel="shortcut icon" type="image/x-icon" href="/static/favicon.ico" />
    <link rel="canonical" href="http://localhost:7777/auth/login" />
    <title>
      Login
      | SimpleLogin
    </title>
    <link rel="stylesheet"
          href="/static/node_modules/font-awesome/css/font-awesome.css" />
    <!-- Dashboard Core -->
    <link href="/static/assets/css/dashboard.css" rel="stylesheet" />
    <!-- Tabler JS -->
    <script src="/static/assets/js/vendors/jquery-3.2.1.min.js"></script>
    <script src="/static/assets/js/vendors/bootstrap.bundle.min.js"></script>
    <script src="/static/assets/js/vendors/jquery.sparkline.min.js"></script>
    <script src="/static/assets/js/vendors/selectize.min.js"></script>
    <script src="/static/assets/js/vendors/jquery.tablesorter.min.js"></script>
    <script src="/static/assets/js/vendors/jquery-jvectormap-2.0.3.min.js"></script>
    <script src="/static/assets/js/vendors/jquery-jvectormap-de-merc.js"></script>
    <script src="/static/assets/js/vendors/jquery-jvectormap-world-mill.js"></script>
    <script src="/static/assets/js/vendors/circle-progress.min.js"></script>
    <script src="/static/assets/js/core.js"></script>
    <!-- ClipboardJS -->
    <script src="/static/vendor/clipboard.min.js"></script>
    <!-- IntroJS -->
    <link rel="stylesheet"
          type="text/css"
          href="/static/node_modules/intro.js/minified/introjs.min.css" />
    <script src="/static/node_modules/intro.js/minified/intro.min.js"></script>
    <!-- Sentry -->
    <script src="/static/node_modules/%40sentry/browser/build/bundle.min.js"></script>
    <link rel="stylesheet" href="/static/vendor/bootstrap-social.min.css" />
    <!-- Toastr library -->
    <link rel="stylesheet"
          href="/static/node_modules/toastr/build/toastr.min.css" />
    <script src="/static/node_modules/toastr/build/toastr.min.js"></script>
    <script src="/static/node_modules/bootbox/dist/bootbox.min.js"></script>
    <!-- Multiple-select library -->
    <link rel="stylesheet"
          href="/static/node_modules/multiple-select/dist/multiple-select.min.css" />
    <script src="/static/node_modules/multiple-select/dist/multiple-select.min.js"></script>
    <!-- Parseley library -->
    <script src="/static/node_modules/parsleyjs/dist/parsley.min.js"></script>
    <script src="/static/node_modules/parsleyjs/dist/i18n/en.js"></script>
    <script src="/static/node_modules/htmx.org/dist/htmx.min.js"></script>
    
    <link rel="stylesheet"
          href="/static/darkmode.css?v=dev" />
    <link rel="stylesheet"
          type="text/css"
          href="/static/style.css?v=dev" />
    <script src="/static/js/theme.js"></script>
    <script>toastr.options.closeButton = true;</script>
    <!-- For additional head -->
    
  </head>
  <body>
    <div class="page">
      
      
      <div class="container">
        <!-- For flash messages -->
        
          <!-- Categories: success (green), info (blue), warning (yellow), danger (red) -->
          

            <script>toastr.error("Email or password incorrect");</script>
          
        
      </div>
      

  <div class="page-single">
    <div class="container">
      <div class="row">
        <div class="col mx-auto" style="max-width: 32rem">
          <div class="text-center mb-6">
            <a href="https://simplelogin.io">
              <img src="/static/logo.svg"
                   style="background-color: transparent;
                          height: 20px">
            </a>
          </div>
          

  
  <div class="card" style="border-radius: 2%">
    <div class="card-body p-6">
      <h1 class="card-title">Welcome back!</h1>
      <form method="post">
        <input id="csrf_token" name="csrf_token" type="hidden" value="ImRhZWVkZjMzZWEzOTUwMzYzOGQzODRlYTFjY2I0ODIzYTU0MzZjNDki.akw4sw.qZaqkuk5l-Es1C_5sMyvSyGvUgs">
        <div class="form-group">
          <label class="form-label">Email address</label>
          <input autofocus="true" class="form-control" id="email" name="email" required type="email" value="john@wick.com">
          
  

        </div>
        <div class="form-group">
          <label class="form-label">Password</label>
          <input class="form-control" id="password" name="password" required type="password" value="">
          
  

          <div class="text-muted">
            <a href="/auth/forgot_password" class="small">I forgot my password</a>
          </div>
        </div>
        <div class="form-footer">
          <button type="submit" class="btn btn-primary btn-block">Log in</button>
        </div>
      </form>
      
      
    </div>
  </div>
  <div class="text-center text-muted mt-2">
    Don't have an account yet?
    <a href="/auth/register">Sign up</a>
  </div>

        </div>
      </div>
    </div>
  </div>

    </div>
    <script>
  

  // default options for bootbox
  bootbox.setDefaults({
    closeButton: false,
    backdrop: true
  })

  var clipboard = new ClipboardJS('.clipboard');

  clipboard.on('success', function (e) {
    toastr.success("Copied to clipboard");
    e.clearSelection();
  });

  // Handle back or close button
  $('.back-or-close').on("click", function () {
    // the window is actually a popup, in this case just close it
    if (history.length == 1) {
      window.close();
    } else {
      history.back();
    }
  });

  document.body.addEventListener('htmx:responseError', function(evt) {
    toastr.error("Sorry for the inconvenience! Could you refresh the page & retry please?", "Unknown Error");
  });

    </script>
    <script src="/static/local-storage-polyfill.js"></script>
    <script src="/static/js/an.js?v=2"></script>
    <!-- For additional script -->
    
  </body>
</html>
```

**Complete, unedited output — full raw HTTP response for variant (b) (non-existent user):
status line + all headers + the entire response body, verbatim.**

```text
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Mon, 06 Jul 2026 23:22:27 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 7026
Set-Cookie: slapp=8ace1cbf-c01d-45a6-8c5c-64ca27d6c914.jIeHzsKUPVTHYccWS315dD5MhXo; Expires=Mon, 13-Jul-2026 23:22:27 GMT; HttpOnly; Path=/; SameSite=Lax


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
    <!-- Bing -->
    <meta name="msvalidate.01" content="2A313A69CBFD1A378C3B91734DC221A8" />
    <!-- Yandex -->
    <meta name="yandex-verification" content="c9e5d4d68bc983a1" />
    <meta name="description"
          content="Protect your email address with email ALIAS. Create a different email alias for each website. No more phishing, or spam." />
    <link rel="icon" href="/static/favicon.ico" type="image/x-icon" />
    <link rel="shortcut icon" type="image/x-icon" href="/static/favicon.ico" />
    <link rel="canonical" href="http://localhost:7777/auth/login" />
    <title>
      Login
      | SimpleLogin
    </title>
    <link rel="stylesheet"
          href="/static/node_modules/font-awesome/css/font-awesome.css" />
    <!-- Dashboard Core -->
    <link href="/static/assets/css/dashboard.css" rel="stylesheet" />
    <!-- Tabler JS -->
    <script src="/static/assets/js/vendors/jquery-3.2.1.min.js"></script>
    <script src="/static/assets/js/vendors/bootstrap.bundle.min.js"></script>
    <script src="/static/assets/js/vendors/jquery.sparkline.min.js"></script>
    <script src="/static/assets/js/vendors/selectize.min.js"></script>
    <script src="/static/assets/js/vendors/jquery.tablesorter.min.js"></script>
    <script src="/static/assets/js/vendors/jquery-jvectormap-2.0.3.min.js"></script>
    <script src="/static/assets/js/vendors/jquery-jvectormap-de-merc.js"></script>
    <script src="/static/assets/js/vendors/jquery-jvectormap-world-mill.js"></script>
    <script src="/static/assets/js/vendors/circle-progress.min.js"></script>
    <script src="/static/assets/js/core.js"></script>
    <!-- ClipboardJS -->
    <script src="/static/vendor/clipboard.min.js"></script>
    <!-- IntroJS -->
    <link rel="stylesheet"
          type="text/css"
          href="/static/node_modules/intro.js/minified/introjs.min.css" />
    <script src="/static/node_modules/intro.js/minified/intro.min.js"></script>
    <!-- Sentry -->
    <script src="/static/node_modules/%40sentry/browser/build/bundle.min.js"></script>
    <link rel="stylesheet" href="/static/vendor/bootstrap-social.min.css" />
    <!-- Toastr library -->
    <link rel="stylesheet"
          href="/static/node_modules/toastr/build/toastr.min.css" />
    <script src="/static/node_modules/toastr/build/toastr.min.js"></script>
    <script src="/static/node_modules/bootbox/dist/bootbox.min.js"></script>
    <!-- Multiple-select library -->
    <link rel="stylesheet"
          href="/static/node_modules/multiple-select/dist/multiple-select.min.css" />
    <script src="/static/node_modules/multiple-select/dist/multiple-select.min.js"></script>
    <!-- Parseley library -->
    <script src="/static/node_modules/parsleyjs/dist/parsley.min.js"></script>
    <script src="/static/node_modules/parsleyjs/dist/i18n/en.js"></script>
    <script src="/static/node_modules/htmx.org/dist/htmx.min.js"></script>
    
    <link rel="stylesheet"
          href="/static/darkmode.css?v=dev" />
    <link rel="stylesheet"
          type="text/css"
          href="/static/style.css?v=dev" />
    <script src="/static/js/theme.js"></script>
    <script>toastr.options.closeButton = true;</script>
    <!-- For additional head -->
    
  </head>
  <body>
    <div class="page">
      
      
      <div class="container">
        <!-- For flash messages -->
        
          <!-- Categories: success (green), info (blue), warning (yellow), danger (red) -->
          

            <script>toastr.error("Email or password incorrect");</script>
          
        
      </div>
      

  <div class="page-single">
    <div class="container">
      <div class="row">
        <div class="col mx-auto" style="max-width: 32rem">
          <div class="text-center mb-6">
            <a href="https://simplelogin.io">
              <img src="/static/logo.svg"
                   style="background-color: transparent;
                          height: 20px">
            </a>
          </div>
          

  
  <div class="card" style="border-radius: 2%">
    <div class="card-body p-6">
      <h1 class="card-title">Welcome back!</h1>
      <form method="post">
        <input id="csrf_token" name="csrf_token" type="hidden" value="ImRmNGYzMzc5Yzg0YTJjYzg3NTBlNzg3NTVkMTg1OTE3NzgxOGRjYjEi.akw4sw.yobeJFxof7sXnEsGZ1AGx_vuPk0">
        <div class="form-group">
          <label class="form-label">Email address</label>
          <input autofocus="true" class="form-control" id="email" name="email" required type="email" value="nobody@nowhere.example">
          
  

        </div>
        <div class="form-group">
          <label class="form-label">Password</label>
          <input class="form-control" id="password" name="password" required type="password" value="">
          
  

          <div class="text-muted">
            <a href="/auth/forgot_password" class="small">I forgot my password</a>
          </div>
        </div>
        <div class="form-footer">
          <button type="submit" class="btn btn-primary btn-block">Log in</button>
        </div>
      </form>
      
      
    </div>
  </div>
  <div class="text-center text-muted mt-2">
    Don't have an account yet?
    <a href="/auth/register">Sign up</a>
  </div>

        </div>
      </div>
    </div>
  </div>

    </div>
    <script>
  

  // default options for bootbox
  bootbox.setDefaults({
    closeButton: false,
    backdrop: true
  })

  var clipboard = new ClipboardJS('.clipboard');

  clipboard.on('success', function (e) {
    toastr.success("Copied to clipboard");
    e.clearSelection();
  });

  // Handle back or close button
  $('.back-or-close').on("click", function () {
    // the window is actually a popup, in this case just close it
    if (history.length == 1) {
      window.close();
    } else {
      history.back();
    }
  });

  document.body.addEventListener('htmx:responseError', function(evt) {
    toastr.error("Sorry for the inconvenience! Could you refresh the page & retry please?", "Unknown Error");
  });

    </script>
    <script src="/static/local-storage-polyfill.js"></script>
    <script src="/static/js/an.js?v=2"></script>
    <!-- For additional script -->
    
  </body>
</html>
```

**Complete, unedited output — the `after_request` access-log lines (full absolute paths).**

```text
2026-07-06 23:22:27,037 - SL - DEBUG - 64024 - "/tmp/blitzy/app/blitzy-f78a0548-c369-4cf4-92f5-e5a16d484468_00bd29/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.019193172454833984
2026-07-06 23:22:27,279 - SL - DEBUG - 64024 - "/tmp/blitzy/app/blitzy-f78a0548-c369-4cf4-92f5-e5a16d484468_00bd29/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 200, takes 0.23977041244506836
2026-07-06 23:22:27,284 - SL - DEBUG - 64024 - "/tmp/blitzy/app/blitzy-f78a0548-c369-4cf4-92f5-e5a16d484468_00bd29/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.001247406005859375
2026-07-06 23:22:27,290 - SL - DEBUG - 64024 - "/tmp/blitzy/app/blitzy-f78a0548-c369-4cf4-92f5-e5a16d484468_00bd29/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 200, takes 0.004595756530761719
```

**Citation.** `app/auth/views/login.py:45` is the failure condition
(`if not user or not user.check_password(form.password.data)`); `app/auth/views/login.py:49` is
`flash("Email or password incorrect", "error")`; `app/auth/views/login.py:50` is
`LoginEvent(LoginEvent.ActionType.failed).send()`; and `app/auth/views/login.py:74-82` returns the
re-rendered `auth/login.html` template via `render_template("auth/login.html", form=form,
next_url=next_url, show_resend_activation=show_resend_activation, connect_with_proton=CONNECT_WITH_PROTON,
connect_with_oidc=OIDC_CLIENT_ID is not None, connect_with_oidc_icon=CONNECT_WITH_OIDC_ICON)`.
`LoginEvent.send()` at `app/events/auth_event.py:22-25` calls
`newrelic.agent.record_custom_event("LoginEvent", {"action": self.action.name, "source":
self.source.name})` — a NewRelic custom event, not a log line. The access line is produced by
`after_request` at `server.py:284`.

**Cause → effect.** When the credentials do not match, the view does not redirect and does not raise
— it flashes the generic error and falls through to re-render the `auth/login.html` template, which
Flask returns with the default `200`. Hence the wrong-credentials response is a `200` re-render of the form (with the
`toastr.error("Email or password incorrect")` flash), identical for both variants because the same
branch handles "no such user" and "bad password". The only telemetry is the NewRelic `LoginEvent`
(`action="failed"`); the only log artifact is the `after_request` access line showing `POST
/auth/login … 200`.


---

## Coverage pass

Every distinct thing each question asks for — including every named header in Q5 and every variant
in Q1/Q2 — is enumerated below with its concrete observed value, the governing `file:line` +
function, a pointer to the section holding the verbatim observed evidence, the sibling variants
covered, and the causal reason.

| # | Required item / named variant | Concrete observed value | Governing `file:line` + function | Observed evidence | Sibling variants covered | Causal reason |
|---|-------------------------------|-------------------------|----------------------------------|-------------------|--------------------------|---------------|
| Q1-a | API call, session + no `Authentication` header | `HTTP 200 OK`, `Content-Length: 244`, `user_info` JSON | `app/api/base.py:17,20-25` `authorize_request` | Q1 output | Q1-b | Header absent -> `ApiKey.get_by(None)` is `None` -> session fallback to `current_user` |
| Q1-b | API call, no cookie + no header | `HTTP 401 UNAUTHORIZED`, `{"error":"Wrong api key"}` (26 B) | `app/api/base.py:26-27` `authorize_request` | Q1 output | Q1-a | No key and no session -> `else` branch returns 401 |
| Q2-a | Sudo-gated op, browser-session-only | `HTTP 500`, `{"error":"Internal error"}` (27 B); `AttributeError: 'NoneType' object has no attribute 'sudo_mode_at'` | `app/api/base.py:47` `check_sudo_mode_is_active`; `server.py:390-392` `error_handler` | Q2 output + traceback | Q2-b | `g.api_key=None` -> `None.sudo_mode_at` raises -> generic 500 |
| Q2-b | Sudo-gated op, API key without sudo | `HTTP 440 UNKNOWN`, `{"error":"Need sudo"}` (22 B) | `app/api/base.py:69-70` `require_api_sudo` | Q2 output | Q2-a | Clean falsy sudo check -> intended non-standard 440 |
| Q2-note | Non-standard status characterization | `440` = non-standard (IIS "Login Timeout"), message `"Need sudo"` | `app/api/base.py:70` `require_api_sudo` | Q2 direct answer | — | Repurposed non-standard code + unusual message |
| Q3-bytes | Raw persisted bytes | begins `\x80\x04` (pickle `PROTO 4`); 300-byte stream | `app/session.py:91` `save_session` | Q3 output | Q3-nonauth | `pickle.dumps(dict(session))` stored in Redis |
| Q3-format | Serialization format | Python `pickle`, protocol 4 (via `pickletools.dis`) | `app/session.py:9-12,91` | Q3 output | — | `RedisSessionStore` pickles the session dict |
| Q3-keys | Authenticated-session keys | `_permanent, _fresh, csrf_token, _user_id, _id, sudo_time` | `app/session.py:68-91`; Flask-Login/WTF writers | Q3 output | non-auth: `_permanent,_fresh,csrf_token` | `_user_id` written on login; `csrf_token` by Flask-WTF |
| Q3-key | Storage key structure | `session:<uuid4>` | `app/session.py:18,44-45` `_get_key` | Q3 output | — | `f"{SESSION_PREFIX}:{session_id}"` |
| Q3-cookie | Cookie structure | `slapp = <uuid4>.<hmac-signature>` | `app/session.py:38-41,102-114`; `app/config.py:199` | Q3 output | — | HMAC `itsdangerous.Signer(salt="session")` over the UUID |
| Q3-ttl | TTL | `604800` s auth / `300` s anon | `app/session.py:95-96` `save_session` | Q3 + Q3-nonauth output | — | TTL keyed on presence of `_user_id` |
| Q4 | Session id before vs after login | PRESERVED (uuid4 + full cookie byte-identical), 2 runs | `app/session.py:68-80` `open_session`; `app/extensions.py:8` | Q4 output + before/after table | run 1 & run 2 | Validly-signed id reused; `login_user()` does not rotate |
| Q5-x | Custom `X-Custom-Test` header | STRIPPED (absent from forward) | `email_handler.py:793-810`; `app/email_utils.py:536-542` | Q5(b) wire capture | Received, Reply-To | Not on `headers_to_keep` whitelist |
| Q5-received | `Received` header | STRIPPED (absent from forward) | `email_handler.py:810`; `app/email/headers.py:14` | Q5(b) wire capture | X-Custom-Test, Reply-To | Not on whitelist |
| Q5-replyto | `Reply-To` header | REPLACED (original stripped; reverse-alias `<reverse-alias>@sl.local` re-added) | `email_handler.py:869-873`; `app/email/headers.py:13` | Q5(a) log `old:None` + Q5(b) wire capture | X-Custom-Test, Received | Whitelist strips it; re-added because inbound had `Reply-To` |
| Q6 | Suffix expiry window | 600 s (age <=600 VALID, >=601 EXPIRED); stable 2 runs + real clock | `app/alias_suffix.py:37-42` `check_suffix_signature` (`max_age=600` @40) | Q6 sweep + realtime output | ages 0/300/599/600/601/700; real 0/598.1/602.0 | `unsign` raises `SignatureExpired` when `age>max_age` |
| Q7-times | `api_key.times` | `0 -> 5` (+1 per API-key call) | `app/api/base.py:31` `authorize_request`; `app/models.py:2359` | Q7 output + table | last_used; session-fallback contrast | `times += 1; commit()` only on API-key branch |
| Q7-lastused | `api_key.last_used` | stamped each call (final `2026-07-06 23:21:38.807932`) | `app/api/base.py:30` `authorize_request`; `app/models.py:2358` | Q7 output + table | times; session-fallback contrast | `last_used = arrow.now()` on API-key branch |
| Q7-unchanged | `code`, `sudo_mode_at`, session-fallback | unchanged; session call leaves row identical | `app/api/base.py:30-32`; `app/models.py:2356,2360` | Q7 output + table | API-key vs session paths | Update statements live only in `if api_key:` branch |
| Q8-status | Wrong-cred response | `HTTP 200 OK` re-render (not 401, not redirect), both variants | `app/auth/views/login.py:45,74-82` | Q8 output + full bodies | wrong-password & non-existent user | Failure flashes + `render_template` -> default 200 |
| Q8-flash | Flash message | `Email or password incorrect` (`toastr.error("Email or password incorrect")`) | `app/auth/views/login.py:49` | Q8 full bodies | both variants (identical) | Generic message -> no user enumeration |
| Q8-log | Log evidence | `after_request` access line for `POST /auth/login` shows status `200`; no ERROR line | `server.py:284` `after_request` | Q8 access-log output | — | Failure emits telemetry, not a log ERROR |
| Q8-telemetry | Telemetry | NewRelic custom event `LoginEvent` `action="failed"` | `app/auth/views/login.py:50`; `app/events/auth_event.py:22-25` | Q8 citation | — | `record_custom_event`, not a log line |

---

## Repository state & read-only guarantee

This investigation is strictly read-only. **No source file was modified.** The only committed
artifact is this document, `blitzy/documentation/app_2cd6ee777f8c.md`. All probe scripts, the
transient SMTP sink, the transient `q5.env` config, and all captured logs lived outside the
repository tree (`/tmp/qprobes`) and were **removed** after the observations were captured; the
transient second SMTP handler (`:20382`) and sink (`:20999`) were stopped, leaving only the canonical
web app (`:7777`) and inbound handler (`:20381`) running. Consequently
`git diff --name-status <baseline>` reports exactly one change:

```text
A	blitzy/documentation/app_2cd6ee777f8c.md
```

**Runtime defects observed (reported, not remediated).** Two behaviors observed above are
security-relevant and are intentionally left unfixed under the read-only mandate: **Q2(a)** — a
browser-session-only call to `DELETE /api/user` raises an `AttributeError` and returns a generic
`HTTP 500` instead of a clean authorization error (`app/api/base.py:47`); and **Q4** — the session
identifier is **not rotated** on login, exposing SimpleLogin to session fixation
(`app/session.py:68-80`). Both are documented with their verbatim runtime evidence above.
