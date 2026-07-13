# SimpleLogin Runtime Behavior Investigation — Evidence-Backed Answers (Q1–Q8)

This document empirically verifies **eight runtime behaviors** of the SimpleLogin
application's authentication, session-management, and email-forwarding subsystems.
Every answer below is grounded in **what actually happened when the system was run** —
not in code reading alone. For each question you will find:

- **(a) Command(s)** — the exact shell / HTTP / SQL command executed (fenced).
- **(b) Raw output** — the complete, unedited output that command produced (fenced).
- **(c) Answer** — the concrete, resolved conclusion.
- **(d) `file:line` reference(s)** — the governing source location, naming the responsible function/method.
- **OBSERVED vs INFERRED** — every value is labelled; observed values are preferred and inferred explanations are called out explicitly.

**Methodology (run-first):** the canonical stack was built and started *before* any
answer was written, and each answer was derived from captured runtime output. The real
canonical entry points were exercised — the running Flask web app on
`http://localhost:7777` and the standalone SMTP `email_handler.py` on port `20381`.
No SimpleLogin source file was modified; the investigation is strictly read-only and
the only artifact produced is this document.

---

## Environment / Run Setup

All commands were executed **inside the canonical Docker container** built from the
image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (the pod host runs
Python 3.13 and cannot host SimpleLogin's pinned Python-3.10 stack, so the container is
the canonical environment). The embedded repository lives at `/app` and is checked out at
git commit `2cd6ee77` (matching the branch suffix `app_2cd6ee777f8c`). The Python
interpreter is `/app/venv/bin/python` = **Python 3.10.18**, with the exact `poetry.lock`
pins present (Flask 1.1.2, Flask-Login 0.5.0, Werkzeug 1.0.1, itsdangerous 1.1.0,
redis 4.6.0, SQLAlchemy 1.3.24, arrow 0.16.0, bcrypt 3.2.0, gunicorn 20.0.4, aiosmtpd 1.4.2).

### Infrastructure (bundled inside the image)

- **PostgreSQL 15** — cluster `15/main` online on port `5432`; role `myuser` / `mypassword`,
  database `simplelogin` (owner `myuser`). *(PostgreSQL 13+ is the stated requirement;
  the image ships 15.)*
- **Redis 7.0.15** — online on port `6379`.

Verification (OBSERVED):

```bash
pg_lsclusters
redis-cli ping
```

```text
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log

PONG
```

### `.env` configuration

`.env` (a copy of `example.env`, **git-ignored**, so editing it is not a source
modification) drives the app. The values relevant to this investigation:

| Key | Value | Source |
|-----|-------|--------|
| `URL` | `http://localhost:7777` | `example.env:L6` |
| `NOT_SEND_EMAIL` | `true` (default; **commented out only for the Q5 email test** — see Q5) | `example.env:L19` |
| `EMAIL_DOMAIN` | `sl.local` | `example.env:L22` |
| `DB_URI` | `postgresql://myuser:mypassword@localhost:5432/simplelogin` | `example.env:L75` |
| `FLASK_SECRET` | `secret` | `example.env:L77` |
| `MEM_STORE_URI` | `redis://localhost` | appended to `.env` |

**Postgres port reconciliation (chosen approach):** `CONTRIBUTING.md:L100` maps Postgres
to host port `15432`, whereas `example.env:L75` uses `5432`. In this canonical image the
bundled PostgreSQL listens on **`5432`** and `.env`'s `DB_URI` already uses `5432`, so no
reconciliation edit was needed — approach (i), run Postgres on `5432`, is satisfied out of
the box.

**Redis session store (decisive for Q3 & Q4):** `MEM_STORE_URI` defaults to `None`
(`app/config.py:L568` — `MEM_STORE_URI = os.environ.get("MEM_STORE_URI", None)`), and
`server.py:L163-L165` installs `initialize_redis_services(app, MEM_STORE_URI)` **only**
`if MEM_STORE_URI:`. That call is what swaps Flask's default client-side cookie session for
the server-side `RedisSessionStore`. `.env` sets `MEM_STORE_URI=redis://localhost`, so the
real `RedisSessionStore` path is active (confirmed below by the presence of `session:*`
keys in Redis). This is a canonical-configuration prerequisite (Redis is a stated stack
component), **not** a bypass.

### Canonical entry points — how they were started

The web app is served with **gunicorn** (the canonical production WSGI launcher,
`gunicorn wsgi:app -b 0.0.0.0:7777`); `wsgi.py` is simply `from server import create_app;
app = create_app()`. Running under gunicorn (debug **off**) yields production-accurate HTTP
responses — important for Q2, where Flask's dev-mode interactive debugger would otherwise
replace the real HTTP 500 with an HTML debugger page. The database was migrated and the dev
account seeded via the documented `alembic upgrade head && flask dummy-data` step
(`CONTRIBUTING.md:L106`).

```bash
# database migration + dev-data seeding (CONTRIBUTING.md:L106)
alembic upgrade head && flask dummy-data
# web app (canonical WSGI launcher), single worker, logs captured to /tmp/gunicorn.log
FLASK_APP=server.py /app/venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 30 > /tmp/gunicorn.log 2>&1 &
# standalone SMTP email handler on port 20381 (CONTRIBUTING.md:L184-221), logs to /tmp/eh.log
FLASK_APP=server.py /app/venv/bin/python email_handler.py > /tmp/eh.log 2>&1 &
```

Both listeners were confirmed up (OBSERVED):

```bash
nc -z -w2 localhost 7777 && echo "7777 OPEN"
nc -z -w2 localhost 20381 && echo "20381 OPEN"
```

```text
7777 OPEN
20381 OPEN
```

The SimpleLogin application logger runs at **DEBUG** level
(`app/log.py:L51` — `logger.setLevel(logging.DEBUG)`), and `after_request()`
(`server.py:L284`) logs every HTTP transaction (method, path, **status code**, timing) to
`/tmp/gunicorn.log`. This is the source of the log evidence used for Q2 and Q8.

### Confirming login (canonical `/auth/login` flow)

**OBSERVED runtime detail:** the real login route is **`/auth/login`**, because the auth
blueprint sets `url_prefix="/auth"` (`app/auth/base.py:L3-L4`). Authentication uses a
Flask-WTF form with a CSRF token, so an authenticated `slapp` cookie is obtained by
`GET /auth/login` (to receive a session + CSRF token) followed by `POST /auth/login` with
the credentials and that CSRF token, using a persistent cookie jar.

```bash
# GET the login page (fresh cookie jar) and extract the hidden csrf_token
curl -s -c /tmp/jar.txt http://localhost:7777/auth/login \
  | grep -o 'name="csrf_token"[^>]*value="[^"]*"'
# POST the canonical dev credentials with that csrf_token, reusing the jar
curl -s -i -c /tmp/jar.txt -b /tmp/jar.txt \
  --data-urlencode "email=john@wick.com" \
  --data-urlencode "password=password" \
  --data-urlencode "csrf_token=<token>" \
  http://localhost:7777/auth/login
```

```text
HTTP/1.1 302 FOUND
Location: http://localhost:7777/dashboard/
Set-Cookie: slapp=6150c256-e356-4c64-9473-315bf626a4e6.Sg4cMf9yQd_w2mkwWlliABthn2A; Expires=Mon, 20-Jul-2026 16:50:07 GMT; HttpOnly; Path=/; SameSite=Lax
```

The **302 redirect to `/dashboard/`** confirms a successful login with the canonical dev
credentials `john@wick.com / password` (`CONTRIBUTING.md:L108-L109`). The resulting
authenticated cookie jar (`slapp` cookie) was reused for the API/session questions.

Observation-only tooling used throughout: `curl`, `redis-cli`, `psql`, `swaks`, and a
throwaway `aiosmtpd` mail sink (for Q5). These are not project dependencies; no packages
were added, updated, or removed.

---

## Q1 — API auth via browser session (no API key)

**Question:** What actual HTTP status code and response body structure come back when an
authenticated browser session calls an `/api/*` endpoint **without** the API-key header?

**(a) Commands**

```bash
# (1) authenticated slapp cookie only, NO Authentication header
curl -sS -i -b /tmp/auth_jar.txt http://localhost:7777/api/user_info
# (2) contrast: no cookie, no key
curl -sS -i http://localhost:7777/api/user_info
```

**(b) Raw output**

```text
# (1) session cookie, NO Authentication header
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Mon, 13 Jul 2026 16:51:37 GMT
Connection: close
Content-Type: application/json
Content-Length: 244
Access-Control-Allow-Origin: *
Set-Cookie: slapp=6150c256-e356-4c64-9473-315bf626a4e6.Sg4cMf9yQd_w2mkwWlliABthn2A; Expires=Mon, 20-Jul-2026 16:51:37 GMT; HttpOnly; Path=/; SameSite=Lax

{"can_create_reverse_alias":true,"connected_proton_address":null,"email":"john@wick.com","in_trial":false,"is_premium":true,"max_alias_free_plan":5,"name":"John Wick","profile_picture_url":"http://localhost:7777/static/upload/profile_pic.svg"}
```

```text
# (2) contrast: no cookie, no key
HTTP/1.1 401 UNAUTHORIZED
Server: gunicorn/20.0.4
Date: Mon, 13 Jul 2026 16:51:37 GMT
Connection: close
Content-Type: application/json
Content-Length: 26
Access-Control-Allow-Origin: *
Set-Cookie: slapp=b50f1700-07aa-4e9b-9389-af0502757f55.1vJUW0oKprUcsPbi6CG_wfvc-zc; Expires=Mon, 20-Jul-2026 16:51:37 GMT; HttpOnly; Path=/; SameSite=Lax

{"error":"Wrong api key"}
```

**(c) Answer (OBSERVED)**

1. **Status code:** `200 OK`.
2. **Body structure:** a flat JSON object with `Content-Type: application/json` and exactly
   these keys (observed, note they differ from a naïve guess):
   `can_create_reverse_alias` (bool), `connected_proton_address` (null), `email` (str),
   `in_trial` (bool), `is_premium` (bool), `max_alias_free_plan` (int), `name` (str),
   `profile_picture_url` (str).
   For `john@wick.com`: `{"can_create_reverse_alias":true,"connected_proton_address":null,`
   `"email":"john@wick.com","in_trial":false,"is_premium":true,"max_alias_free_plan":5,`
   `"name":"John Wick","profile_picture_url":"http://localhost:7777/static/upload/profile_pic.svg"}`.
3. **Contrast proves the fallback:** with **no** cookie and **no** key the identical
   endpoint returns `401 UNAUTHORIZED` with body `{"error":"Wrong api key"}`. Therefore the
   `200` above was produced by the **session fallback**, not by any API key.

**(d) `file:line` references**

- `authorize_request()` reads the API key from the **non-standard `Authentication`** header:
  `app/api/base.py:L17` — `api_code = request.headers.get("Authentication")`.
- Session fallback — when no key is present and the user is authenticated:
  `app/api/base.py:L20-L25` — `if not api_key: if current_user.is_authenticated: g.user = current_user`.
- The `401 "Wrong api key"` branch (no key, not authenticated):
  `app/api/base.py:L27` — `return jsonify(error="Wrong api key"), 401`.
- Endpoint `user_info()` guarded by `@require_api_auth`:
  `app/api/views/user_info.py:L50-L52`; body assembled by `user_to_dict()`
  at `app/api/views/user_info.py:L28`.

---

## Q2 — Privileged operation via browser session (non-standard error)

**Question:** What specific (non-standard) status code and exact error message text does the
system actually return when a privileged operation is attempted with **only** a browser
session? *(This was treated as "observe, do not assume".)*

**(a) Commands**

```bash
# privileged endpoint (DELETE /api/user, @require_api_sudo) with ONLY the browser session
curl -sS -i -X DELETE -b /tmp/auth_jar.txt http://localhost:7777/api/user
# ...and the corresponding new lines from the server log
tail -n +<mark> /tmp/gunicorn.log
```

**(b) Raw output — HTTP response**

```text
HTTP/1.1 500 INTERNAL SERVER ERROR
Server: gunicorn/20.0.4
Date: Mon, 13 Jul 2026 16:51:53 GMT
Connection: close
Content-Type: application/json
Content-Length: 27
Access-Control-Allow-Origin: *
Set-Cookie: slapp=6150c256-e356-4c64-9473-315bf626a4e6.Sg4cMf9yQd_w2mkwWlliABthn2A; Expires=Mon, 20-Jul-2026 16:51:53 GMT; HttpOnly; Path=/; SameSite=Lax

{"error":"Internal error"}
```

**(b) Raw output — server log / traceback (`/tmp/gunicorn.log`)**

```text
2026-07-13 16:51:53,405 - SL - ERROR - 715 - "/app/server.py:390" - error_handler() -  - 'NoneType' object has no attribute 'sudo_mode_at'
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
2026-07-13 16:51:53,406 - SL - DEBUG - 715 - "/app/server.py:284" - after_request() -  - 127.0.0.1 DELETE /api/user ImmutableMultiDict([]) 500, takes 0.0043833255767822266
```

**(b) Raw output — contrast: key-authenticated caller without active sudo → 440**

```bash
# pre-seeded key "code" has sudo_mode_at = NULL
curl -sS -i -X DELETE -H "Authentication: code" http://localhost:7777/api/user
```

```text
HTTP/1.1 440 UNKNOWN
Server: gunicorn/20.0.4
Date: Mon, 13 Jul 2026 16:52:17 GMT
Connection: close
Content-Type: application/json
Content-Length: 22
Access-Control-Allow-Origin: *
Set-Cookie: slapp=7f798a68-9e6e-4c12-a01a-be2c0cdf711f.oa0WOGT-gjRkRH1S5iyIIx9YmRE; Expires=Mon, 20-Jul-2026 16:52:17 GMT; HttpOnly; Path=/; SameSite=Lax

{"error":"Need sudo"}
```

**(c) Answer (OBSERVED)**

- With **only a browser session**, the privileged operation returns
  **HTTP `500 INTERNAL SERVER ERROR`** with the exact body **`{"error":"Internal error"}`**.
  This is *not* a standard authorization error (not 401/403) and is *not* the `440 "Need sudo"`
  that one might expect.
- **Root cause (observed in the traceback, reported as-found — not patched, per read-only
  scope):** on the browser-session path `authorize_request()` sets `g.api_key = None`. The
  `@require_api_sudo` wrapper then calls `check_sudo_mode_is_active(g.api_key)`, which
  dereferences `api_key.sudo_mode_at` on a `None` object, raising
  `AttributeError: 'NoneType' object has no attribute 'sudo_mode_at'`. The global exception
  handler converts that into the `{"error":"Internal error"}` / `500` seen above.
- **The `440 "Need sudo"` path does exist**, but only for a **key-authenticated** caller
  whose key has no active sudo mode (contrast above, using pre-seeded key `code` whose
  `sudo_mode_at` is `NULL`): that path returns **`440 UNKNOWN`** with body `{"error":"Need sudo"}`.
  (`440` is a non-standard status code, so the reason phrase is literally `UNKNOWN`.)

**(d) `file:line` references**

- `g.api_key = None` on the browser-session path: `app/api/base.py:L42` — `g.api_key = api_key`
  (where `api_key` is `None`), in `authorize_request()`.
- The failing dereference: `app/api/base.py:L46-L49`, specifically `L47`
  `return api_key.sudo_mode_at and g.api_key.sudo_mode_at >= ...`, in `check_sudo_mode_is_active()`.
- The wrapper that invokes it: `app/api/base.py:L69` — `if not check_sudo_mode_is_active(g.api_key):`
  in `require_api_sudo`'s `decorated()`; the `440` return is `app/api/base.py:L69-L70`
  `return jsonify(error="Need sudo"), 440`.
- The `{"error":"Internal error"}` / `500`: `server.py:L388-L392` `error_handler()` —
  `if request.path.startswith("/api/"): return jsonify(error="Internal error"), 500`.
- Endpoint `delete_user()` (`DELETE /api/user`, `@require_api_sudo`):
  `app/api/views/user.py:L12-L14`.

---

## Q3 — Session data format in storage

**Question:** Capture the raw bytes from the storage backend; what format are they in; what
keys are present in the deserialized data for an authenticated session; and how is the
session storage key structured?

**(a) Commands**

```bash
# (1) derive the session UUID by unsigning the slapp cookie
python - <<'PY'
import itsdangerous
cookie = "6150c256-e356-4c64-9473-315bf626a4e6.Sg4cMf9yQd_w2mkwWlliABthn2A"
signer = itsdangerous.Signer("secret", salt="session", key_derivation="hmac")
print("unsigned UUID =", signer.unsign(cookie).decode())
PY

# (2) raw bytes + type + TTL from Redis
redis-cli TYPE "session:6150c256-e356-4c64-9473-315bf626a4e6"
redis-cli TTL  "session:6150c256-e356-4c64-9473-315bf626a4e6"
redis-cli --no-raw GET "session:6150c256-e356-4c64-9473-315bf626a4e6"

# (3) deserialize with pickle and enumerate keys
python - <<'PY'
import pickle, redis
r = redis.Redis.from_url("redis://localhost")
raw = r.get("session:6150c256-e356-4c64-9473-315bf626a4e6")
print("len", len(raw), "first3", raw[:3])
data = pickle.loads(raw)
print("type", type(data).__name__, "keys", list(data.keys()))
PY
```

**(b) Raw output**

```text
# (1) unsign the cookie -> UUID
cookie          = 6150c256-e356-4c64-9473-315bf626a4e6.Sg4cMf9yQd_w2mkwWlliABthn2A
unsigned UUID   = 6150c256-e356-4c64-9473-315bf626a4e6
redis key       = session:6150c256-e356-4c64-9473-315bf626a4e6
```

```text
# (2) TYPE / TTL / raw GET
string
604752
"\x80\x04\x95!\x01\x00\x00\x00\x00\x00\x00}\x94(\x8c\n_permanent\x94\x88\x8c\x06_fresh\x94\x88\x8c\ncsrf_token\x94\x8c(a9bcb9e4c6a22efa8a6180c53410541d2a9c3bf6\x94\x8c\b_user_id\x94\x8c$0fb423fb-e47b-41b7-bfcd-c21453f33dca\x94\x8c\x03_id\x94\x8c\x80b03643a8a515eae10966eb5799d0b928b1fae8daeb0b0a33ba33f35b5fd7e09d574a2b38cd3af3ce53bdb464d28d2c515533674c8b087cba7d49abfa2cc9de57\x94\x8c\tsudo_time\x94J?\x17Uju."
```

```text
# (3) pickle.loads -> dict + keys + values
raw type in redis : bytes len 300 bytes
first 3 bytes     : b'\x80\x04\x95' (pickle PROTO opcode 0x80 0x04 = protocol 4)
pickle.loads type : dict
deserialized keys : ['_permanent', '_fresh', 'csrf_token', '_user_id', '_id', 'sudo_time']
--- key : value(type) ---
  '_permanent': True  <bool>
  '_fresh': True  <bool>
  'csrf_token': 'a9bcb9e4c6a22efa8a6180c53410541d2a9c3bf6'  <str>
  '_user_id': '0fb423fb-e47b-41b7-bfcd-c21453f33dca'  <str>
  '_id': 'b03643a8a515eae10966eb5799d0b928b1fae8daeb0b0a33ba33f35b5fd7e09d574a2...(truncated)  <str>
  'sudo_time': 1783961407  <int>
```

**(c) Answer (OBSERVED)**

1. **Raw bytes:** shown above — a 300-byte `string` value beginning `\x80\x04\x95…` and
   ending `…\tsudo_time\x94J?\x17Uju.`.
2. **Format:** **Python `pickle`** (protocol 4 — the leading `\x80\x04` is the pickle
   `PROTO 4` opcode). `pickle.loads` yields a `dict`.
3. **Deserialized key set (authenticated session):**
   `['_permanent', '_fresh', 'csrf_token', '_user_id', '_id', 'sudo_time']`
   — i.e. `_permanent=True`, `_fresh=True`, `csrf_token` (str),
   `_user_id` (the Flask-Login user id, a UUID string), `_id` (the Flask-Login
   session-protection identifier, a 128-hex string), and `sudo_time` (int epoch).
   *(Note: the real set includes `_permanent` and `sudo_time` in addition to the commonly
   expected `_user_id`/`_fresh`/`_id`/`csrf_token`.)*
4. **Key structure:** the Redis storage key is **`session:<uuid4>`**
   (here `session:6150c256-e356-4c64-9473-315bf626a4e6`). The **cookie** itself is
   `<uuid4>.<hmac-signature>` (an `itsdangerous.Signer` value). The observed **TTL is
   `604752` seconds (~7 days)** because the session contains `_user_id` (authenticated);
   unauthenticated sessions get 300 s.

*(The `csrf_token`, `_user_id`, and `_id` values above are ephemeral local-dev session
internals from a disposable container whose `FLASK_SECRET` is the public dev default
`"secret"`; they are not usable against any real deployment and are shown as evidence.)*

**(d) `file:line` references**

- Value serialization = Python pickle: `app/session.py:L91` —
  `val = pickle.dumps(dict(session))`, in `RedisSessionStore.save_session()`.
- Storage-key structure `session:<id>`: `app/session.py:L18`
  (`SESSION_PREFIX = "session"`) and `app/session.py:L43-L45`
  (`_get_key()` → `f"{SESSION_PREFIX}:{session_Id}"`).
- Cookie signer (`<uuid>.<hmac>`): `app/session.py:L38-L41` — `_get_signer()` →
  `itsdangerous.Signer(app.secret_key, salt="session", key_derivation="hmac")`;
  `app.secret_key = FLASK_SECRET` (`server.py:L151`), `FLASK_SECRET="secret"` (`example.env:L77`).
- TTL rule (7-day vs 300 s): `app/session.py:L92-L96` (`if "_user_id" not in session: ttl = 300`),
  with `permanent_session_lifetime = timedelta(days=7)` at `server.py:L207`.

---

## Q4 — Session identifier behavior during login (fixation test)

**Question:** If you capture the session ID from the cookie **before** login, then complete
login, does the identifier value stay the same or get replaced? Show the actual before/after
values.

**(a) Commands**

```bash
# BEFORE: GET /auth/login on a fresh jar -> receive slapp cookie
curl -sS -i -c /tmp/jar_q4.txt http://localhost:7777/auth/login   # capture Set-Cookie
# (extract csrf_token from the page for the POST)
# POST /auth/login with correct creds, SAME jar
curl -sS -i -c /tmp/jar_q4.txt -b /tmp/jar_q4.txt \
  --data-urlencode "email=john@wick.com" \
  --data-urlencode "password=password" \
  --data-urlencode "csrf_token=<token>" \
  http://localhost:7777/auth/login                                 # capture Set-Cookie
# unsign both cookies to reveal the embedded UUID identifier
```

**(b) Raw output**

```text
----- BEFORE: GET /auth/login (fresh jar, unauthenticated) -----
Set-Cookie: slapp=3bbe825a-4662-42fb-a18e-ea23440fd8eb.6glBIA_vgu1EqTuH5N7-oce_5Ok; Expires=Mon, 20-Jul-2026 16:53:25 GMT; HttpOnly; Path=/; SameSite=Lax

----- POST /auth/login (correct creds, same jar) -----
HTTP/1.1 302 FOUND
Location: http://localhost:7777/dashboard/
Set-Cookie: slapp=3bbe825a-4662-42fb-a18e-ea23440fd8eb.6glBIA_vgu1EqTuH5N7-oce_5Ok; Expires=Mon, 20-Jul-2026 16:53:25 GMT; HttpOnly; Path=/; SameSite=Lax

----- Unsign both to reveal embedded UUID identifiers -----
BEFORE cookie: 3bbe825a-4662-42fb-a18e-ea23440fd8eb.6glBIA_vgu1EqTuH5N7-oce_5Ok
BEFORE uuid  : 3bbe825a-4662-42fb-a18e-ea23440fd8eb
AFTER  cookie: 3bbe825a-4662-42fb-a18e-ea23440fd8eb.6glBIA_vgu1EqTuH5N7-oce_5Ok
AFTER  uuid  : 3bbe825a-4662-42fb-a18e-ea23440fd8eb
VERDICT      : SAME (unchanged, no rotation)
```

**(c) Answer (OBSERVED)**

1. **Before value (pre-login):** `slapp` = `3bbe825a-4662-42fb-a18e-ea23440fd8eb.6glBIA_vgu1EqTuH5N7-oce_5Ok`;
   embedded identifier UUID = **`3bbe825a-4662-42fb-a18e-ea23440fd8eb`**.
2. **After value (post-login, same request chain, on the `302` response):**
   `slapp` = `3bbe825a-4662-42fb-a18e-ea23440fd8eb.6glBIA_vgu1EqTuH5N7-oce_5Ok`;
   embedded identifier UUID = **`3bbe825a-4662-42fb-a18e-ea23440fd8eb`**.
3. **Verdict:** the identifier **stays the SAME** — it is **not replaced** on login. In fact
   the entire cookie is **byte-identical** before and after (the plain `Signer` HMAC over an
   unchanged UUID is deterministic, so the signature is identical too). SimpleLogin does
   **not** rotate the session identifier at login, so login does not itself defend against
   session fixation. *(This was independently reproduced: a separate earlier login produced
   UUID `6150c256-e356-4c64-9473-315bf626a4e6` unchanged across the same before/after — 2
   consistent runs.)*

**(d) `file:line` references**

- Identifier reuse: `app/session.py:L68-L80` — `open_session()` calls
  `extract_and_validate_session_id()` and only generates a fresh UUID
  (`ServerSession(session_id=str(uuid.uuid4()))`) when **no** valid signed identifier is
  present; otherwise it reuses the validated `session_id`.
- Fresh UUID only on purge/logout: `app/session.py:L61-L66` — `purge_session()`.
- Flask-Login session protection (does not rotate on login):
  `app/extensions.py:L8` — `login_manager.session_protection = "strong"`. **INFERRED
  (corroborated by research):** `"strong"` compares a client-derived identifier and can mark
  the session non-fresh / delete it on mismatch, but it does not regenerate the cookie's
  session identifier at `login_user()`. The **observed** before/after values above are the
  primary evidence; this note only explains *why* they match.

---

## Q5 — Email-forwarding header handling

**Question:** For an inbound email carrying a custom `X-` header, a `Received` header, and a
`Reply-To` header — which survive to the forwarded message and which are stripped? Send a
real test email and examine the result.

**Setup (documented canonical email test, `CONTRIBUTING.md:L192-L221`):** to *view* the
forwarded message (rather than just a one-line summary), the documented procedure is to
comment out `NOT_SEND_EMAIL` and point Postfix at a receiving MTA. In `.env` (git-ignored)
`NOT_SEND_EMAIL=true` was commented out and `POSTFIX_SERVER=localhost` / `POSTFIX_PORT=1025`
were set; a throwaway `aiosmtpd` sink on `127.0.0.1:1025` stood in for mailcatcher/MailHog
and captured the full forwarded message. The `email_handler.py` (the real canonical entry
point under test) was restarted to pick up this config; it continued to receive inbound mail
on **port 20381**. The alias `e1@sl.local` (dev-seeded, enabled, forwards to
`john@wick.com`) was used — the exact alias used in the canonical example at
`CONTRIBUTING.md:L218`. *(Under the default `NOT_SEND_EMAIL=true`, `MailSender.send()` only
logs a `subject/from/to` summary — `app/mail_sender.py:L130-L137` — and never emits the full
forwarded header block, which is why the documented sink configuration is required to answer
this question.)*

**(a) Command**

```bash
swaks --to e1@sl.local --from tester@example.com --server 127.0.0.1:20381 \
  --h-Subject "Q5 header forwarding test" \
  --header "X-Custom-Test: hello-x-header" \
  --header "Received: from evil.example.com by test-injected" \
  --header "Reply-To: replyto-original@example.com" \
  --body "Q5 header forwarding test body"
```

**(b) Raw output — the handler accepted and forwarded the message**

```text
 -> X-Custom-Test: hello-x-header
 -> Received: from evil.example.com by test-injected
 -> Reply-To: replyto-original@example.com
 ->
 -> Q5 header forwarding test body
 -> .
<-  250 Message accepted for delivery
 -> QUIT
<-  221 Bye

[sink] captured #1 -> /tmp/q5_forwarded_1.eml mail_from=sl.lmycyibtfqqdemzygi3tonc5.s6i4me7lr4vmm@sl.local rcpt_tos=['john@wick.com']
```

**(b) Raw output — the FORWARDED message as received by the MTA sink** (the three
`X-Sink-*` lines are added by the sink to record envelope info; everything below them is the
message SimpleLogin actually sent):

```text
X-Sink-Mail-From: sl.lmycyibtfqqdemzygi3tonc5.s6i4me7lr4vmm@sl.local
X-Sink-Rcpt-Tos: john@wick.com
X-Sink-Boundary: ===== forwarded message as received by MTA sink =====
Date: Mon, 13 Jul 2026 16:54:44 +0000
Subject: Q5 header forwarding test
Message-Id: <20260713165444.002836@8de7fd3e9430>
Content-Transfer-Encoding: 7bit
X-SimpleLogin-Type: Forward
X-SimpleLogin-EmailLog-ID: 3
X-SimpleLogin-Envelope-From: tester@example.com
X-SimpleLogin-Original-From: tester@example.com
X-SimpleLogin-Envelope-To: e1@sl.local
From: "tester at example.com" <tester_at_example_com_xuloyio@sl.local>
Reply-To: "replyto-original at example.com"
 <replyto-original_at_example_com_brmvynaawj@sl.local>
To: e1@sl.local

Q5 header forwarding test body
```

**(c) Answer (OBSERVED) — fate of each of the three headers**

| Inbound header | Inbound value | Fate in forwarded message |
|----------------|---------------|---------------------------|
| `X-Custom-Test` | `hello-x-header` | **STRIPPED** — absent from the forwarded message (not in the whitelist). |
| `Received` | `from evil.example.com by test-injected` | **STRIPPED** — absent; the forwarded message contains **zero** `Received:` headers. |
| `Reply-To` | `replyto-original@example.com` | **Original value does NOT survive** — stripped by the whitelist, then **re-added rewritten** to the reverse-alias contact: `Reply-To: "replyto-original at example.com" <replyto-original_at_example_com_brmvynaawj@sl.local>`. |

So **none of the three original header values reach the recipient**: `X-Custom-Test` and
`Received` are removed entirely, and `Reply-To` is replaced with a reverse-alias address (the
original `replyto-original@example.com` does not appear). For reference, `From` is likewise
rewritten to a reverse-alias (`tester_at_example_com_xuloyio@sl.local`), and SimpleLogin
adds its own `X-SimpleLogin-*` control headers. Only whitelisted headers such as `Date`,
`Subject`, `Message-Id`, and MIME headers pass through.

**(d) `file:line` references**

- Forward path builds the keep-list and strips everything else:
  `email_handler.py:L793-L807` (`headers_to_keep = [FROM, TO, CC, SUBJECT, DATE, MESSAGE_ID,
  REFERENCES, IN_REPLY_TO, SL_QUEUE_ID, LIST_UNSUBSCRIBE, LIST_UNSUBSCRIBE_POST] + MIME_HEADERS`)
  then `email_handler.py:L810` (`delete_all_headers_except(msg, headers_to_keep)`), inside
  `forward_email_to_mailbox()` (defined at `email_handler.py:L679`).
- The stripping itself (case-insensitive keep-list; deletes all others):
  `app/email_utils.py:L536-L542` — `delete_all_headers_except()`.
- `From` rewrite to reverse-alias: `email_handler.py:L864-L866`
  (`add_or_replace_header(msg, "From", new_from_header)`).
- `Reply-To` re-added to the reverse-alias contact (only because the inbound message *had* a
  `Reply-To`, which caused a `reply_to_contact` to be created at
  `email_handler.py:L586-L594`): `email_handler.py:L869-L873`
  (`add_or_replace_header(msg, "Reply-To", reply_to_contact.new_addr())`).
- Header-name constants: `app/email/headers.py:L13` (`REPLY_TO = "Reply-To"`),
  `app/email/headers.py:L14` (`RECEIVED = "Received"`).

---

## Q6 — Alias-creation token expiry window (timing / magnitude)

**Question:** Experimentally determine the token expiry window — test a token immediately,
wait, and re-test until it stops working; find the boundary.

**Method (canonical API path, key in the `Authentication` header):** fetch a fresh
`signed_suffix` from `GET /api/v5/alias/options`, then `POST /api/v2/alias/custom/new`. Two
**independent parallel runs** were executed; each brackets the hypothesized 600 s boundary
with three attempts — **immediate** (age ≈ 0 s), **just under** (age ≈ 594.6 s), and **just
over** (age ≈ 606.7 s). Token "age" is measured from the moment `GET …/alias/options` signed
the suffix. The scale/durations are stated explicitly per attempt below.

**(a) Commands (issued programmatically per attempt)**

```bash
# fetch a fresh signed_suffix
curl -s -H "Authentication: <q6-key>" http://localhost:7777/api/v5/alias/options
# create the alias with that suffix (immediately, or after aging it)
curl -s -H "Authentication: <q6-key>" -H "Content-Type: application/json" \
  -X POST http://localhost:7777/api/v2/alias/custom/new \
  --data '{"alias_prefix":"q6rNxxx","signed_suffix":"<suffix>"}'
```

**(b) Raw output**

```text
======================= Q6 RUN 1 =======================
[16:58:34 epoch=1783961914] ===== Q6 RUN 1 (key=rxepsrgg...) window_hypothesis=600s =====
[16:58:34 epoch=1783961914] RUN1 IMMEDIATE age=0.0s -> HTTP=201 BODY={"alias":"q6r1imm1783961914@old.com",...,"id":13,...}
[16:58:34 epoch=1783961914] RUN1 fetched UNDER+OVER suffixes at t0=1783961914; bracketing 600s boundary
[17:08:29 epoch=1783962509] RUN1 UNDER(pre-600) age=594.6s -> HTTP=201 BODY={"alias":"q6r1under1783962509@old.com",...,"id":15,...}
[17:08:41 epoch=1783962521] RUN1 OVER(post-600) age=606.7s -> HTTP=412 BODY={"error":"Alias creation time is expired, please retry"}
[17:08:41 epoch=1783962521] RUN1 DONE

======================= Q6 RUN 2 =======================
[16:58:37 epoch=1783961917] ===== Q6 RUN 2 (key=rxepsrgg...) window_hypothesis=600s =====
[16:58:37 epoch=1783961917] RUN2 IMMEDIATE age=0.0s -> HTTP=201 BODY={"alias":"q6r2imm1783961917@old.com",...,"id":14,...}
[16:58:38 epoch=1783961918] RUN2 fetched UNDER+OVER suffixes at t0=1783961918; bracketing 600s boundary
[17:08:32 epoch=1783962512] RUN2 UNDER(pre-600) age=594.6s -> HTTP=201 BODY={"alias":"q6r2under1783962512@old.com",...,"id":16,...}
[17:08:44 epoch=1783962524] RUN2 OVER(post-600) age=606.7s -> HTTP=412 BODY={"error":"Alias creation time is expired, please retry"}
[17:08:44 epoch=1783962524] RUN2 DONE
```

**(c) Answer (OBSERVED)**

1. **Works immediately:** yes — at age `0.0 s` the create returns **HTTP `201`** (both runs).
2. **Stops working after waiting:** yes — at age `606.7 s` (just past 600 s) the create
   returns **HTTP `412`** with body **`{"error":"Alias creation time is expired, please retry"}`**
   (both runs).
3. **Boundary bracketed (~600 s):** at age `594.6 s` the token **still works** (`201`); at
   age `606.7 s` it is **expired** (`412`). The boundary therefore lies within
   `[594.6 s, 606.7 s]`, i.e. the window is **600 seconds (10 minutes)**.
4. **Durations & stability:** stated ages per attempt were `0.0 s`, `594.6 s`, `606.7 s`; the
   result was **identical across both independent runs** (`201 / 201 / 412`), confirming
   stability.

**(d) `file:line` references**

- Expiry check: `app/alias_suffix.py:L37-L40`, specifically `L40`
  `return signer.unsign(signed_suffix, max_age=600).decode()`, in `check_suffix_signature()`
  (returns `None` on `SignatureExpired`, which produces the 412).
- The signer: `app/alias_suffix.py:L11` — `signer = itsdangerous.TimestampSigner(config.CUSTOM_ALIAS_SECRET)`;
  `CUSTOM_ALIAS_SECRET = FLASK_SECRET + "custom_alias"` at `app/config.py:L201`.
- The 412 response: `app/api/views/new_custom_alias.py:L70-L73` — `alias_suffix =
  check_suffix_signature(signed_suffix); if not alias_suffix: return jsonify(error="Alias
  creation time is expired, please retry"), 412`, in `new_custom_alias_v2()`. (This expiry
  check runs *before* the `409 already-exists` check, so unique prefixes are not required to
  observe the 412.) A fresh suffix returns `201` at `new_custom_alias.py:L109-L112`.

---

## Q7 — API-key usage statistics (magnitude)

**Question:** After several API calls with the same key, which DB fields are updated and what
actual values do you observe?

**Method:** a real, system-generated API key (`name = q7-key`, created canonically via
`POST /api/api_key`) was used. **N = 5** key-authenticated calls per run; the `api_key` row
was read (`psql`) immediately before and after each run; the whole cycle was repeated **twice**.

**(a) Commands**

```bash
# BEFORE / AFTER read
psql ... -c "SELECT id,name,times,last_used FROM api_key WHERE name='q7-key';"
# N=5 key-authenticated calls with the SAME key
for i in $(seq 1 5); do
  curl -s -o /dev/null -w "%{http_code} " -H "Authentication: <q7-key>" http://localhost:7777/api/user_info
done
```

**(b) Raw output**

```text
################## Q7 RUN 1 (N=5 key-authenticated calls, same key q7-key) ##################
----- BEFORE (psql) -----
-[ RECORD 1 ]-----
id        | 4
name      | q7-key
times     | 0
last_used |

----- 5 calls: GET /api/user_info with Authentication: <q7-key> -----
HTTP codes: 200 200 200 200 200
----- AFTER (psql) -----
-[ RECORD 1 ]-------------------------
id        | 4
name      | q7-key
times     | 5
last_used | 2026-07-13 16:59:48.854608
```

```text
################## Q7 RUN 2 (N=5 key-authenticated calls, same key q7-key) ##################
----- BEFORE (psql) -----
-[ RECORD 1 ]-------------------------
id        | 4
name      | q7-key
times     | 5
last_used | 2026-07-13 16:59:48.854608

----- 5 calls -----
HTTP codes: 200 200 200 200 200
----- AFTER (psql) -----
-[ RECORD 1 ]-------------------------
id        | 4
name      | q7-key
times     | 10
last_used | 2026-07-13 16:59:50.132383
```

**(c) Answer (OBSERVED)**

1. **Fields updated:** exactly two — **`times`** and **`last_used`**. (`id`, `name`, `code`,
   `user_id` are unchanged.)
2. **Actual values:**
   - **Run 1:** `times` `0 → 5`; `last_used` `NULL → 2026-07-13 16:59:48.854608`.
   - **Run 2:** `times` `5 → 10`; `last_used` `2026-07-13 16:59:48.854608 → 2026-07-13 16:59:50.132383`.
3. **N:** `5` calls per run (all returned HTTP `200`).
4. **Stability (≥ 2 runs):** confirmed — `times` increases by **exactly N (= 5)** each run,
   and `last_used` advances to a fresh recent timestamp each run.

**(d) `file:line` references**

- Per-request update on key auth: `app/api/base.py:L30-L32` — `api_key.last_used =
  arrow.now(); api_key.times += 1; Session.commit()`, in `authorize_request()`.
- Column definitions on the `ApiKey` model: `app/models.py:L2358`
  (`last_used = sa.Column(ArrowType, default=None)`) and `app/models.py:L2359`
  (`times = sa.Column(sa.Integer, default=0, nullable=False)`); the class is at
  `app/models.py:L2350`.

---

## Q8 — Failed login (wrong credentials)

**Question:** When a login attempt fails with wrong credentials, what specific log messages
appear and what HTTP response details come back?

**(a) Commands**

```bash
# GET the login page (fresh jar) to obtain a valid csrf_token
curl -s -c /tmp/jar_q8.txt http://localhost:7777/auth/login   # extract csrf_token
# POST wrong credentials with that csrf_token
curl -s -i -c /tmp/jar_q8.txt -b /tmp/jar_q8.txt \
  --data-urlencode "email=john@wick.com" \
  --data-urlencode "password=WRONG-PASSWORD-xyz" \
  --data-urlencode "csrf_token=<token>" \
  http://localhost:7777/auth/login
# capture the new server-log lines produced by the request
tail -n +<mark> /tmp/gunicorn.log
```

**(b) Raw output — HTTP response**

```text
--- HTTP status line + Content-Type/Length ---
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 7017

--- flashed error text found in body? (grep) ---
Email or password incorrect
--- HTML context around the flash (first match) ---
101:            <script>toastr.error("Email or password incorrect");</script>
```

**(b) Raw output — new server-log lines (`/tmp/gunicorn.log`)**

```text
2026-07-13 17:00:24,410 - SL - DEBUG - 714 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 200, takes 0.24028825759887695
```

**(c) Answer (OBSERVED)**

1. **HTTP response details:** **`200 OK`** — the login form is **re-rendered**, *not* an
   authorization error (not `401`/`403`). `Content-Type: text/html; charset=utf-8`,
   `Content-Length: 7017`. The flashed error message **`Email or password incorrect`** is
   present in the body, rendered (via `toastr`) at body line 101:
   `<script>toastr.error("Email or password incorrect");</script>`.
2. **Log messages:** the **only** application log line produced by the failed attempt is the
   request-completion line from `after_request()`:
   `... "/app/server.py:284" - after_request() - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 200, takes 0.24…`
   — i.e. it records the `POST /auth/login` returning **`200`**. There is **no dedicated
   "failed login" log line**: the failed `LoginEvent` records a New-Relic *custom event*, not
   a log entry (see below). This is the honest observed result.

**(d) `file:line` references**

- Wrong-credentials branch: `app/auth/views/login.py:L45` — `if not user or not
  user.check_password(form.password.data):`, inside `login()`.
- Rate-limit penalty flag: `app/auth/views/login.py:L47` — `g.deduct_limit = True`.
- Flash message: `app/auth/views/login.py:L49` — `flash("Email or password incorrect", "error")`.
- Failed event: `app/auth/views/login.py:L50` — `LoginEvent(LoginEvent.ActionType.failed).send()`.
  **OBSERVED nuance:** `LoginEvent.send()` at `app/events/auth_event.py:L22-L25` calls
  `newrelic.agent.record_custom_event("LoginEvent", {"action": "failed", "source": "web"})`
  — it does **not** write to the application logger, which is why no dedicated failed-login
  log message appears in the captured output.
- Re-render → HTTP 200: `app/auth/views/login.py:L74` — `return render_template("auth/login.html", …)`.
- The transaction log line is emitted by `after_request()` at `server.py:L284`.

---

## Coverage Confirmation

A final pass confirming every named sub-part of every question is answered above, each with
command + raw output + resolved answer + `file:line`, and observed/inferred labelling.

- **Q1** — ✅ exact **status code** (`200`); ✅ exact **body structure** (real JSON keys shown);
  ✅ contrast **no-cookie `401 {"error":"Wrong api key"}`** proving the session fallback.
- **Q2** — ✅ exact **status code** (`500`); ✅ exact **error message text**
  (`{"error":"Internal error"}`) + traceback (`AttributeError … 'sudo_mode_at'`);
  ✅ **440 `{"error":"Need sudo"}`** path demonstrated via a key-authenticated caller, with
  which path yields which code clearly labelled; defect reported, not patched.
- **Q3** — ✅ **raw bytes** shown; ✅ **format** = Python pickle (protocol 4; cite `session.py:L91`);
  ✅ actual **deserialized key set** (`_permanent, _fresh, csrf_token, _user_id, _id, sudo_time`);
  ✅ **key structure** `session:<uuid4>` (cite `session.py:L45`), cookie `<uuid>.<signature>`,
  and observed **TTL** `604752 s` (~7 days).
- **Q4** — ✅ actual **before** value (`3bbe825a-…`); ✅ actual **after** value (`3bbe825a-…`);
  ✅ **verdict**: SAME / unchanged (no rotation), with a second independent confirmation.
- **Q5** — ✅ each of the **three** headers individually: `X-Custom-Test` **stripped**,
  `Received` **stripped**, `Reply-To` original **does not survive** (rewritten to
  reverse-alias) — with the raw forwarded header block as proof.
- **Q6** — ✅ works **immediately** (`201`); ✅ **stops working** after waiting (`412` with
  exact body); ✅ **boundary** bracketed within `[594.6 s, 606.7 s]` ⇒ **600 s**;
  ✅ **durations stated** and **≥ 2-run stability**; raw `201` and `412` shown.
- **Q7** — ✅ **fields updated** (`times`, `last_used`); ✅ actual **before/after values**;
  ✅ **N = 5** stated; ✅ **≥ 2-run stability** (+5 each run; `last_used` advances); raw
  `psql` rows shown.
- **Q8** — ✅ specific **log messages** (only the `after_request` `200` transaction line;
  failed `LoginEvent` records a New-Relic custom event, not a log line); ✅ **HTTP response
  details**: `200` (re-render, not 401/403) with the flashed `Email or password incorrect`
  in the body.

**Read-only compliance:** no SimpleLogin source file was modified; all observation scripts
lived outside the repository and were removed; the sole repository change is the addition of
this document.

