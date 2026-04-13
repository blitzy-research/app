# SimpleLogin Runtime Behavior Investigation

This document provides a comprehensive, evidence-based investigation of eight specific runtime behaviors in the SimpleLogin email alias platform. Each finding is derived from deep source code analysis, tracing exact execution paths through the codebase with file references and line numbers. Where runtime execution is not feasible due to infrastructure dependencies (PostgreSQL, Redis, SMTP), evidence is provided through detailed code-path tracing that demonstrates the exact execution flow, return values, and side effects the system would produce.

All conclusions are based on the code as the source of truth. No assumptions are made.

---

## Table of Contents

1. [API Session-Fallback Behavior](#1-api-session-fallback-behavior)
2. [Sudo-Protected Endpoint with Session-Only Auth](#2-sudo-protected-endpoint-with-session-only-auth)
3. [Raw Session Data Structure in Redis](#3-raw-session-data-structure-in-redis)
4. [Session Identifier Rotation During Login](#4-session-identifier-rotation-during-login)
5. [Email Forwarding Header Preservation](#5-email-forwarding-header-preservation)
6. [Alias Creation Token Expiration Window](#6-alias-creation-token-expiration-window)
7. [API Key Usage Statistics Tracking](#7-api-key-usage-statistics-tracking)
8. [Login Failure Response and Logging](#8-login-failure-response-and-logging)

---

## 1. API Session-Fallback Behavior

### Question

What is the exact HTTP status code and JSON response body structure returned when an authenticated browser session (Flask-Login) is used to access API endpoints without an `Authentication` header?

### Code-Path Analysis

The entire API authentication pipeline is implemented in `app/api/base.py` (lines 16–43) in the `authorize_request()` function. Here is the exact execution trace when a browser-session-authenticated user makes an API request without an `Authentication` header:

**Step 1** — `app/api/base.py` (line 17):
```python
api_code = request.headers.get("Authentication")
```
Since no `Authentication` header is present, `api_code` is `None`.

**Step 2** — `app/api/base.py` (line 18):
```python
api_key = ApiKey.get_by(code=api_code)
```
`ApiKey.get_by(code=None)` queries the database for an API key with `code=None`. No API key has a NULL code (the column is `nullable=False` per `app/models.py` line 2356), so this returns `None`.

**Step 3** — `app/api/base.py` (line 20):
```python
if not api_key:
```
Evaluates to `True` because `api_key` is `None`. Execution enters the fallback branch.

**Step 4** — `app/api/base.py` (line 21):
```python
if current_user.is_authenticated:
```
For a user with an active browser session, Flask-Login populates `current_user` from the session cookie. `current_user.is_authenticated` evaluates to `True`. Flask-Login is configured with `session_protection = "strong"` (`app/extensions.py` line 8), which validates the session against the user agent and IP address.

**Step 5** — `app/api/base.py` (line 25):
```python
g.user = current_user
```
The request-global user is set to the session-authenticated user.

**Step 6** — `app/api/base.py` (lines 36–40): The function checks if the user is disabled or inactive. For a normal active user, both checks pass without returning an error.

**Step 7** — `app/api/base.py` (line 42):
```python
g.api_key = api_key
```
**This is the critical line.** `api_key` is still `None` from Step 2. So `g.api_key` is set to `None`. This has significant downstream consequences (see [Question 2](#2-sudo-protected-endpoint-with-session-only-auth)).

**Step 8** — `app/api/base.py` (line 43):
```python
return None
```
Returns `None`, signaling successful authorization.

**Step 9** — The `require_api_auth` decorator (`app/api/base.py` lines 52–60) checks:
```python
error_return = authorize_request()
if error_return:
    return error_return
return f(*args, **kwargs)
```
Since `authorize_request()` returned `None`, the condition `if error_return:` is `False`, and the actual endpoint function executes normally.

### Expected Behavior

Authentication **succeeds**. The HTTP response is:

```
HTTP/1.1 200 OK
Content-Type: application/json
```

The response body is the standard JSON payload of whichever API endpoint was called. For example, calling `GET /api/user_info` would return the user info JSON. The session fallback path produces the same successful authentication result as the API key path.

### Key Finding

The session fallback path sets `g.api_key = None` (line 42), whereas the API key path sets `g.api_key` to the actual `ApiKey` object. This difference is invisible for normal API endpoints but causes crashes on sudo-protected endpoints (see [Question 2](#2-sudo-protected-endpoint-with-session-only-auth)) and means API key usage statistics are NOT updated for session-based requests (see [Question 7](#7-api-key-usage-statistics-tracking)).

### Supporting Evidence

- `app/extensions.py` (line 8): `login_manager.session_protection = "strong"` — Flask-Login validates session integrity
- `server.py` (lines 158–162): Session cookie named `slapp`, configured with `SESSION_COOKIE_SAMESITE = "Lax"` and conditionally `SESSION_COOKIE_SECURE = True`
- `server.py` (lines 204–207): Sessions are set permanent with 7-day lifetime via `before_request` handler

---

## 2. Sudo-Protected Endpoint with Session-Only Auth

### Question

What is the specific HTTP status code and exact error message text returned when a browser-session-authenticated user (without an API key) attempts to invoke a `require_api_sudo`-protected endpoint?

### Code-Path Analysis

There are **two distinct paths** through the sudo check, and the session-only path triggers a crash. Both are documented below.

#### Path A — Session-Only User Hitting Sudo Endpoint (CRASH PATH)

**Step 1** — `app/api/base.py` (line 66):
```python
error_return = authorize_request()
```
This succeeds via the session fallback path documented in [Question 1](#1-api-session-fallback-behavior). After this call: `g.api_key = None` and `g.user = current_user`.

**Step 2** — `app/api/base.py` (line 69):
```python
if not check_sudo_mode_is_active(g.api_key):
```
Calls `check_sudo_mode_is_active(None)` because `g.api_key` is `None`.

**Step 3** — Inside `check_sudo_mode_is_active()` at `app/api/base.py` (lines 46–49):
```python
def check_sudo_mode_is_active(api_key: ApiKey) -> bool:
    return api_key.sudo_mode_at and g.api_key.sudo_mode_at >= arrow.now().shift(
        minutes=-SUDO_MODE_MINUTES_VALID
    )
```
`api_key` is `None`. Accessing `None.sudo_mode_at` raises:
```
AttributeError: 'NoneType' object has no attribute 'sudo_mode_at'
```

**Step 4** — This `AttributeError` is an unhandled exception that propagates up through Flask's request handling.

**Step 5** — Flask's catch-all exception handler in `server.py` (lines 388–394) catches it:
```python
@app.errorhandler(Exception)
def error_handler(e):
    LOG.e(e)
    if request.path.startswith("/api/"):
        return jsonify(error="Internal error"), 500
    else:
        return render_template("error/500.html"), 500
```
Since this is an `/api/` path, the JSON branch executes.

**Expected response for session-only user:**

```
HTTP/1.1 500 Internal Server Error
Content-Type: application/json

{"error": "Internal error"}
```

**Rationale**: This is a bug. The `check_sudo_mode_is_active()` function does not handle the case where `api_key` is `None`, which occurs whenever a browser-session-authenticated user (without an API key) reaches a sudo-protected endpoint. The function's type hint declares `api_key: ApiKey` but the caller passes `g.api_key` which is `None` in the session fallback path.

#### Path B — API-Key User Without Active Sudo (STANDARD FAILURE)

When a user authenticates with a valid API key but has not activated sudo mode:

**Step 1** — `authorize_request()` succeeds with `g.api_key` set to the actual `ApiKey` object.

**Step 2** — `check_sudo_mode_is_active(g.api_key)` at `app/api/base.py` (lines 46–49):
- If `api_key.sudo_mode_at` is `None` (sudo never activated): Python's short-circuit evaluation of `None and ...` returns `None` (falsy).
- If `api_key.sudo_mode_at` is set but older than 5 minutes (`SUDO_MODE_MINUTES_VALID = 5` at line 13): the comparison `sudo_mode_at >= arrow.now().shift(minutes=-5)` returns `False`.

**Step 3** — `app/api/base.py` (line 70):
```python
return jsonify(error="Need sudo"), 440
```

**Expected response for API-key user without sudo:**

```
HTTP/1.1 440
Content-Type: application/json

{"error": "Need sudo"}
```

**Key observations about this response:**
- **HTTP 440 is not a standard HTTP status code.** The IANA HTTP status code registry does not define 440. Some Microsoft IIS implementations use 440 for "Login Time-out" but this is proprietary.
- **"Need sudo" is not a standard authorization error message.** Standard HTTP 401/403 messages like "Unauthorized" or "Forbidden" are not used here.

#### Additional Finding — Sudo Activation Endpoint Also Crashes

The sudo activation endpoint in `app/api/views/sudo.py` (line 24):
```python
g.api_key.sudo_mode_at = arrow.now()
```
This endpoint uses `@require_api_auth` (not `@require_api_sudo`), so it bypasses the sudo check. However, at line 24 it directly accesses `g.api_key.sudo_mode_at`, which would also crash with `AttributeError` for session-only users because `g.api_key` is `None`.

### Supporting Evidence

- `app/api/base.py` (line 13): `SUDO_MODE_MINUTES_VALID = 5` — sudo mode is valid for 5 minutes
- `app/api/views/sudo.py` (lines 8–27): The `PATCH /api/sudo` endpoint requires password confirmation to activate sudo mode
- `server.py` (lines 388–394): The catch-all exception handler logs the error and returns HTTP 500 with `{"error": "Internal error"}` for API paths

---

## 3. Raw Session Data Structure in Redis

### Question

What is the raw data from the Redis session storage backend — the serialization format, Redis key structure, HMAC-signed cookie value structure, and all keys present in the deserialized session dictionary for an authenticated user?

### Code-Path Analysis

The session storage is implemented in `app/session.py` via the `RedisSessionStore` class (lines 31–114).

#### Redis Key Structure

`app/session.py` (lines 18, 43–45):
```python
SESSION_PREFIX = "session"

@classmethod
def _get_key(cls, session_Id: str) -> str:
    return f"{SESSION_PREFIX}:{session_Id}"
```

The Redis key format is:
```
session:{uuid4}
```

Example:
```
session:a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

The `session_Id` is a UUID4 string generated by `str(uuid.uuid4())` at `app/session.py` (line 71) when a new session is created.

#### Serialization Format

`app/session.py` (lines 9–12, 91):
```python
try:
    import cPickle as pickle
except ImportError:
    import pickle

# ... in save_session():
val = pickle.dumps(dict(session))
```

The raw bytes stored in Redis are **Python pickle-serialized bytes** of the session dictionary. The `dict(session)` call converts the `ServerSession` (a `CallbackDict` subclass) to a plain Python dict before serializing.

Deserialization occurs in `open_session()` at `app/session.py` (line 76):
```python
data = pickle.loads(val)
```

#### HMAC-Signed Cookie Value Structure

`app/session.py` (lines 37–41): The signer is initialized as:
```python
@classmethod
def _get_signer(cls, app) -> itsdangerous.Signer:
    return itsdangerous.Signer(
        app.secret_key, salt="session", key_derivation="hmac"
    )
```

`app/session.py` (lines 102–104): The cookie value is generated by:
```python
signed_session_id = self._get_signer(app).sign(
    itsdangerous.want_bytes(session.session_id)
)
```

The `itsdangerous.Signer.sign()` method produces a value in the format:
```
{session_uuid}.{hmac_signature}
```

The dot (`.`) is the default separator between the payload and the HMAC signature. The HMAC is computed using the Flask `SECRET_KEY` with salt `"session"` and HMAC key derivation.

The cookie name is **`slapp`**, configured at `app/config.py` (line 199):
```python
SESSION_COOKIE_NAME = "slapp"
```

And applied in `server.py` (line 159):
```python
app.config["SESSION_COOKIE_NAME"] = SESSION_COOKIE_NAME
```

The cookie is set with these attributes (from `server.py` lines 160–162 and the `save_session()` method):
- `HttpOnly`: True (Flask default)
- `SameSite`: Lax
- `Secure`: True if the application URL starts with `https`

#### Session Dictionary Keys for an Authenticated User

The following keys are present in the deserialized session dictionary after a successful login:

| Key | Source | Type | Description |
|-----|--------|------|-------------|
| `_user_id` | `flask_login.login_user()` at `app/auth/views/login_utils.py` (line 36) | `str` | Return value of `User.get_id()` — returns `self.alternative_id` if set, otherwise `str(self.id)` (`app/models.py` lines 595–599) |
| `_fresh` | `flask_login.login_user()` | `bool` | `True` for a fresh login (the default parameter in `login_user()`) |
| `_id` | Flask-Login session protection | `str` | A hex string derived from the user agent and IP address. Used by `session_protection = "strong"` (`app/extensions.py` line 8) to detect session hijacking |
| `csrf_token` | Flask-WTF CSRF protection | `str` | A random hex token string generated by Flask-WTF for CSRF protection |
| `sudo_time` | `app/auth/views/login_utils.py` (line 37): `session["sudo_time"] = int(time())` | `int` | Unix timestamp (integer) marking when the web login occurred |

**Conditionally present keys:**

| Key | Source | Condition | Description |
|-----|--------|-----------|-------------|
| `slref` | `server.py` (line 270): `session["slref"] = ref_code` | Present only if user arrived via a referral URL with `?slref=code` parameter | Referral code string |
| `mfa_user_id` | `app/auth/views/login_utils.py` (lines 23, 29): `session[MFA_USER_ID] = user.id` | Present only during MFA flow (user has FIDO or OTP enabled). Value of `MFA_USER_ID` constant is `"mfa_user_id"` from `app/config.py` (line 295) | The `user.id` (integer) of the user awaiting MFA completion |

**Expected deserialized session dictionary for a standard authenticated user (no MFA, no referral):**

```python
{
    '_user_id': 'ab12cd34-ef56-7890-abcd-1234567890ef',  # User.alternative_id or str(User.id)
    '_fresh': True,
    '_id': '6f8b2e1a3c4d5f...',                          # hex string from user agent + IP
    'csrf_token': 'a1b2c3d4e5f6...',                     # random hex token
    'sudo_time': 1700000000,                               # int(time.time()) at login
}
```

#### TTL (Time-to-Live)

`app/session.py` (lines 92–96):
```python
ttl = int(app.permanent_session_lifetime.total_seconds())
# Only 5 minutes for non-authenticated sessions.
# We need to keep the non-authenticated ones because the csrf token is stored in the session.
if "_user_id" not in session:
    ttl = 300
```

`server.py` (lines 206–207):
```python
session.permanent = True
app.permanent_session_lifetime = timedelta(days=7)
```

- **Authenticated sessions**: TTL = 7 days = **604,800 seconds**
- **Unauthenticated sessions**: TTL = **300 seconds** (5 minutes) — kept briefly for CSRF token storage

### Supporting Evidence

- `app/session.py` (line 97–100): `self._redis_w.setex(name=self._get_key(session.session_id), value=val, time=ttl)` — stores the pickled session data with the computed TTL
- `app/session.py` (lines 105–114): The response cookie is set with `expires`, `httponly`, `domain`, `path`, `secure`, and `samesite` attributes

---

## 4. Session Identifier Rotation During Login

### Question

Does the session ID value in the browser cookie change during the login flow?

### Code-Path Analysis

The complete login flow is traced below, following the session ID from pre-login through post-login.

#### Pre-Login State

When a user first visits the login page, Flask creates an unauthenticated session for CSRF token storage. This triggers `open_session()`:

`app/session.py` (lines 68–80):
```python
def open_session(self, app: flask.Flask, request: flask.Request):
    session_id = self.extract_and_validate_session_id(app, request)
    if not session_id:
        return ServerSession(session_id=str(uuid.uuid4()))
    val = self._redis_r.get(self._get_key(session_id))
    if val is not None:
        try:
            data = pickle.loads(val)
            return ServerSession(data, session_id=session_id)
        except Exception:
            pass
    return ServerSession(session_id=str(uuid.uuid4()))
```

On the first visit, no cookie exists, so `session_id` is `None` (line 70), and a new `ServerSession` is created with a fresh `uuid4()` session ID (line 71). This session ID is signed and stored in the `slapp` cookie.

#### Login Request

When the user submits the login form, the browser sends the existing `slapp` cookie.

**Step 1** — `app/session.py` `open_session()` (line 69):
```python
session_id = self.extract_and_validate_session_id(app, request)
```
The `extract_and_validate_session_id()` method (`app/session.py` lines 48–59) reads the `slapp` cookie, verifies its HMAC signature using the signer, and extracts the session UUID. Since the cookie is valid, `session_id` is the **same UUID** that was created during the first visit.

**Step 2** — `app/session.py` (lines 73–77):
```python
val = self._redis_r.get(self._get_key(session_id))
if val is not None:
    try:
        data = pickle.loads(val)
        return ServerSession(data, session_id=session_id)
```
The existing session data (containing `csrf_token`) is loaded from Redis. A `ServerSession` is created with the **same `session_id`** — it is reused, not regenerated.

**Step 3** — `app/auth/views/login.py` (line 40+): The login form validates, user and password check pass.

**Step 4** — `app/auth/views/login_utils.py` (lines 35–37):
```python
LOG.d("log user %s in", user)
login_user(user)
session["sudo_time"] = int(time())
```
Flask-Login's `login_user(user)` adds `_user_id`, `_fresh`, and `_id` to the **existing** session object. It does **NOT** call `purge_session()` or generate a new session ID. The same `ServerSession` instance with the same `session_id` attribute is modified in place.

**Step 5** — `app/session.py` `save_session()` (lines 97–104):
```python
self._redis_w.setex(
    name=self._get_key(session.session_id),  # SAME session_id
    value=val,
    time=ttl,
)
signed_session_id = self._get_signer(app).sign(
    itsdangerous.want_bytes(session.session_id)  # SAME session_id
)
```
The session is saved to Redis using the **same session_id** from `open_session()`. The **same session_id** is signed and written to the response cookie.

### Expected Behavior

**The session ID does NOT change during login.** The same UUID that was created for the unauthenticated session persists unchanged after `login_user()` is called.

Example timeline:
```
1. Visit /auth/login  → Cookie set: slapp=<UUID_A>.<sig_A>
2. Submit login form  → Cookie sent: slapp=<UUID_A>.<sig_A>
3. Login succeeds     → Cookie set: slapp=<UUID_A>.<sig_A>  (SAME UUID_A)
```

### Contrast with Logout

Only `logout_session()` rotates the session ID. `app/session.py` (lines 117–121):
```python
def logout_session():
    logout_user()
    purge_fn = getattr(current_app.session_interface, "purge_session", None)
    if callable(purge_fn):
        purge_fn(session)
```

And `purge_session()` at `app/session.py` (lines 61–66):
```python
def purge_session(self, session: ServerSession):
    try:
        self._redis_w.delete(self._get_key(session.session_id))
        session.session_id = str(uuid.uuid4())
    except AttributeError:
        pass
```

This deletes the old Redis key (line 63) and assigns a **new** `uuid4()` session ID (line 64). This rotation only happens during logout — never during login.

### Key Finding

**Session fixation concern**: The session ID stays identical before and after login. An attacker who obtains a pre-authentication session ID (e.g., through a network sniff of the unauthenticated session cookie) could potentially use it to access the post-authentication session, since the cookie value does not change when the session is promoted from unauthenticated to authenticated.

This relates to [Question 3](#3-raw-session-data-structure-in-redis): the same Redis key `session:{uuid}` that initially held only a `csrf_token` is overwritten with the full authenticated session data including `_user_id`, without changing the key name.

---

## 5. Email Forwarding Header Preservation

### Question

Which of three specific headers — a custom `X-*` header, the `Received` header, and the `Reply-To` header — survive the forwarding pipeline in `email_handler.py`?

### Code-Path Analysis

The email forwarding pipeline in `forward_email_to_mailbox()` uses a whitelist approach to header preservation. Only headers explicitly listed in `headers_to_keep` survive; all others are stripped.

#### The Whitelist

`email_handler.py` (lines 793–807):
```python
headers_to_keep = [
    headers.FROM,                   # "From"
    headers.TO,                     # "To"
    headers.CC,                     # "Cc"
    headers.SUBJECT,                # "Subject"
    headers.DATE,                   # "Date"
    headers.MESSAGE_ID,             # "Message-ID"
    headers.REFERENCES,             # "References"
    headers.IN_REPLY_TO,            # "In-Reply-To"
    headers.SL_QUEUE_ID,            # "X-SL-Queue-Id"
    headers.LIST_UNSUBSCRIBE,       # "List-Unsubscribe"
    headers.LIST_UNSUBSCRIBE_POST,  # "List-Unsubscribe-Post"
] + headers.MIME_HEADERS
```

Where `headers.MIME_HEADERS` is defined at `app/email/headers.py` (lines 44–51):
```python
MIME_HEADERS = [
    MIME_VERSION,                    # "Mime-Version"
    CONTENT_TYPE,                    # "Content-Type"
    CONTENT_DISPOSITION,             # "Content-Disposition"
    CONTENT_TRANSFER_ENCODING,       # "Content-Transfer-Encoding"
]
MIME_HEADERS = [h.lower() for h in MIME_HEADERS]
```

The complete whitelist (15 headers) is: `From`, `To`, `Cc`, `Subject`, `Date`, `Message-ID`, `References`, `In-Reply-To`, `X-SL-Queue-Id`, `List-Unsubscribe`, `List-Unsubscribe-Post`, `mime-version`, `content-type`, `content-disposition`, `content-transfer-encoding`.

**Optional addition** — `email_handler.py` (lines 808–809):
```python
if user.include_header_email_header:
    headers_to_keep.append(headers.AUTHENTICATION_RESULTS)
```
If the user's `include_header_email_header` setting is `True` (default is `True` per `app/models.py` lines 534–536), `Authentication-Results` is also preserved. This does not affect the three queried headers.

#### The Stripping Function

`app/email_utils.py` (lines 536–542):
```python
def delete_all_headers_except(msg: Message, headers: [str]):
    headers = [h.lower() for h in headers]
    for i in reversed(range(len(msg._headers))):
        header_name = msg._headers[i][0].lower()
        if header_name not in headers:
            del msg._headers[i]
```

This performs **case-insensitive** comparison. It iterates through `msg._headers` in reverse order and deletes any header whose name (lowercased) is not in the whitelist (also lowercased).

Applied at `email_handler.py` (line 810):
```python
delete_all_headers_except(msg, headers_to_keep)
```

#### Analysis of Each Queried Header

**1. Custom `X-*` header (e.g., `X-Custom-Header`):**

The only `X-` header in the whitelist is `X-SL-Queue-Id` (`headers.SL_QUEUE_ID` defined at `app/email/headers.py` line 24). Other X-headers such as `X-Rspamd-Queue-Id` (line 15), `X-Spamd-Result` (line 16), and `X-Spam-Status` (line 19) are defined as constants in `app/email/headers.py` but are **NOT included** in the `headers_to_keep` whitelist.

**Result: STRIPPED.** Any custom `X-*` header (except `X-SL-Queue-Id`) is removed by `delete_all_headers_except()`.

**2. `Received` header:**

Defined at `app/email/headers.py` (line 14):
```python
RECEIVED = "Received"
```

This constant exists but is **NOT included** in the `headers_to_keep` whitelist at `email_handler.py` (lines 793–807).

**Result: STRIPPED.** The `Received` header is removed by `delete_all_headers_except()`.

**3. `Reply-To` header:**

Defined at `app/email/headers.py` (line 13):
```python
REPLY_TO = "Reply-To"
```

`Reply-To` is **NOT in the base whitelist**. However, it has special handling that occurs in two phases:

**Phase 1 — Before header stripping** (`email_handler.py` lines 586–594):
```python
reply_to_contact = None
if msg[headers.REPLY_TO]:
    reply_to = get_header_unicode(msg[headers.REPLY_TO])
    LOG.d("Create or get contact for reply_to_header:%s", reply_to)
    # ignore when reply-to = alias
    if reply_to == alias.email:
        LOG.i("Reply-to same as alias %s", alias)
    else:
        reply_to_contact = get_or_create_reply_to_contact(reply_to, alias, msg)
```

Before header stripping, the handler reads the original `Reply-To` value. If `Reply-To` differs from the alias email, a `reply_to_contact` is created. If `Reply-To` equals the alias email (line 591), no contact is created.

**Phase 2 — Header stripping** (line 810): `delete_all_headers_except(msg, headers_to_keep)` removes the original `Reply-To` header because it is not in the whitelist.

**Phase 3 — After header stripping** (`email_handler.py` lines 869–873):
```python
if reply_to_contact:
    reply_to_header = msg[headers.REPLY_TO]
    new_reply_to_header = reply_to_contact.new_addr()
    add_or_replace_header(msg, "Reply-To", new_reply_to_header)
    LOG.d("Reply-To header, new:%s, old:%s", new_reply_to_header, reply_to_header)
```

If a `reply_to_contact` was created in Phase 1, a **new** `Reply-To` header is added with the reverse-alias address (e.g., `reply_to_contact.new_addr()`). This is NOT the original value — it is rewritten to a SimpleLogin reverse-alias address.

**Result: The original `Reply-To` value is NEVER preserved as-is.** It is either:
- **Stripped entirely** (if `reply_to == alias.email`, no contact is created, and the header is removed)
- **Rewritten to a reverse-alias address** (if `reply_to != alias.email`, the original value is replaced with the reverse-alias address)

### Summary Table

| Header | In Whitelist? | Special Handling? | Survives Forwarding? |
|--------|:---:|:---:|:---:|
| Custom `X-*` header | No | No | **No — stripped** |
| `Received` | No | No | **No — stripped** |
| `Reply-To` | No | Yes — rewritten to reverse-alias | **No — original value lost** (may be replaced with reverse-alias) |

---

## 6. Alias Creation Token Expiration Window

### Question

What is the exact time window during which a signed alias suffix token remains valid?

### Code-Path Analysis

The alias suffix signing and verification is implemented in `app/alias_suffix.py`.

#### Signer Initialization

`app/alias_suffix.py` (line 11):
```python
signer = itsdangerous.TimestampSigner(config.CUSTOM_ALIAS_SECRET)
```

This is a module-level `itsdangerous.TimestampSigner`, which embeds the current timestamp into every signature. The secret key is:

`app/config.py` (line 201):
```python
CUSTOM_ALIAS_SECRET = FLASK_SECRET + "custom_alias"
```

The secret is derived by concatenating the master `FLASK_SECRET` (from the `FLASK_SECRET` environment variable, `app/config.py` line 196) with the string `"custom_alias"`.

#### Signature Verification

`app/alias_suffix.py` (lines 37–42):
```python
def check_suffix_signature(signed_suffix: str) -> Optional[str]:
    # hypothesis: user will click on the button in the 600 secs
    try:
        return signer.unsign(signed_suffix, max_age=600).decode()
    except itsdangerous.BadSignature:
        return None
```

The `max_age=600` parameter (line 40) tells `itsdangerous.TimestampSigner.unsign()` to reject any signature where:

```
current_time - sign_time > 600 seconds
```

#### Signing (for context)

Alias suffixes are signed when presented to the user. For example, in `get_alias_suffixes()` at `app/alias_suffix.py` (line 114):
```python
signed_suffix=signer.sign(suffix).decode(),
```

The `TimestampSigner.sign()` method appends a base64-encoded timestamp and HMAC signature to the suffix value.

### Expected Behavior

The alias creation token expires after exactly **600 seconds (10 minutes)**.

```
Token valid:    sign_time ≤ current_time ≤ sign_time + 600s
Token expired:  current_time > sign_time + 600s
```

When the token is expired, `signer.unsign()` raises `itsdangerous.SignatureExpired` (a subclass of `itsdangerous.BadSignature`), which is caught by the `except` clause, and `check_suffix_signature()` returns `None`. This causes the alias creation to be rejected by the caller.

### Rationale

The comment at line 38 — `# hypothesis: user will click on the button in the 600 secs` — explains the design decision: 10 minutes is considered sufficient time for a user to select and create an alias after the suffix options are presented.

### Supporting Evidence

- `app/alias_suffix.py` (lines 114, 128, 155, 185): All calls to `signer.sign(suffix)` in `get_alias_suffixes()` produce tokens that are subject to the same 600-second expiration
- The `itsdangerous.TimestampSigner` class stores the timestamp as a base64-encoded integer seconds value appended to the payload

---

## 7. API Key Usage Statistics Tracking

### Question

What exact database fields are updated when API calls are made with a key?

### Code-Path Analysis

The API key usage statistics are updated inside `authorize_request()` at `app/api/base.py` (lines 28–32):

```python
else:
    # Update api key stats
    api_key.last_used = arrow.now()
    api_key.times += 1
    Session.commit()
```

This `else` branch executes only when `api_key` is **not None** — i.e., when a valid API key was provided in the `Authentication` header.

**Line 30**: `api_key.last_used = arrow.now()` — Updates the `last_used` column to the current UTC timestamp as an Arrow datetime object.

**Line 31**: `api_key.times += 1` — Increments the `times` column by 1. This is a Python-level increment on the SQLAlchemy-managed attribute.

**Line 32**: `Session.commit()` — Immediately persists both changes to the PostgreSQL database. The commit happens within the authentication step, before the endpoint function executes.

### ApiKey Model Schema

`app/models.py` (lines 2350–2375):

```python
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

### Field-by-Field Documentation

| Field | Type | Default | Updated Per API Call? | Update Value | Description |
|-------|------|---------|:---:|---|---|
| `code` | `String(128)` | `random_string(60)` (line 2366) | No | — | Unique API key identifier sent in `Authentication` header |
| `name` | `String(128)` | `None` | No | — | Optional device/client name |
| `user_id` | `FK → User.id` | Required | No | — | Owning user's ID |
| `last_used` | `ArrowType` | `None` | **Yes** | `arrow.now()` | Timestamp of the most recent API call |
| `times` | `Integer` | `0` | **Yes** | Previous value + 1 | Cumulative count of all API calls made with this key |
| `sudo_mode_at` | `ArrowType` | `None` | No (updated only by `PATCH /api/sudo`) | — | Timestamp when sudo mode was last activated |

### Expected State After N API Calls

After N successful API calls using the same key:
- `last_used` = timestamp of the Nth (most recent) call
- `times` = N

### Cross-Reference with Session Fallback

These statistics are **ONLY** updated when a valid API key is provided in the `Authentication` header. As documented in [Question 1](#1-api-session-fallback-behavior), session-fallback requests follow the `if not api_key:` branch (line 20) which skips the `else` block (lines 28–32) entirely. Session-based API calls do **NOT** update any usage statistics.

### Supporting Evidence

- `app/models.py` (lines 2364–2370): `ApiKey.create()` generates a random 60-character code via `random_string(60)`, with a UUID4 fallback if the code collides
- `app/api/base.py` (line 34): `g.user = api_key.user` — the user is set from the API key's relationship, not from the session

---

## 8. Login Failure Response and Logging

### Question

What are the exact HTTP response details and log messages produced when a login attempt fails with wrong credentials?

### Code-Path Analysis

There are **two distinct login paths** — the web HTML form login and the API JSON login — each with different response formats.

#### Web Login Path

`app/auth/views/login.py` (lines 45–50):

```python
if not user or not user.check_password(form.password.data):
    # Trigger rate limiter
    g.deduct_limit = True
    form.password.data = None
    flash("Email or password incorrect", "error")
    LoginEvent(LoginEvent.ActionType.failed).send()
```

Step-by-step execution:

**Line 45**: `if not user or not user.check_password(form.password.data):` — This condition is `True` when either the email doesn't match any user OR the password is wrong. The `check_password()` method (from `app/pw_models.py`) uses `bcrypt.checkpw()` for constant-time password comparison.

**Line 47**: `g.deduct_limit = True` — Sets a flag on Flask's `g` context object. The rate limiter decorator at `app/auth/views/login.py` (lines 22–24):
```python
@limiter.limit(
    "10/minute", deduct_when=lambda r: hasattr(g, "deduct_limit") and g.deduct_limit
)
```
This means the rate limit counter is only decremented on **failed** login attempts, not successful ones. After 10 failures per minute (per user ID if logged in, per IP if not — see `app/extensions.py` lines 14–19), subsequent requests receive HTTP 429.

**Line 48**: `form.password.data = None` — Clears the password from the form object so it is not rendered back in the HTML response.

**Line 49**: `flash("Email or password incorrect", "error")` — Queues a flash message with category `"error"`. This is displayed in the re-rendered login template.

**Line 50**: `LoginEvent(LoginEvent.ActionType.failed).send()` — Emits a New Relic custom event. From `app/events/auth_event.py` (lines 22–25):
```python
def send(self):
    newrelic.agent.record_custom_event(
        "LoginEvent", {"action": self.action.name, "source": self.source.name}
    )
```
Since no explicit `source` is passed, the default `Source.web` is used (`app/events/auth_event.py` line 18).

The New Relic event payload:
```json
{"action": "failed", "source": "web"}
```

**Lines 74–82**: Execution falls through to:
```python
return render_template(
    "auth/login.html",
    form=form,
    next_url=next_url,
    show_resend_activation=show_resend_activation,
    connect_with_proton=CONNECT_WITH_PROTON,
    connect_with_oidc=OIDC_CLIENT_ID is not None,
    connect_with_oidc_icon=CONNECT_WITH_OIDC_ICON,
)
```

**Expected web response:**

```
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8

<!-- Rendered login.html template with flash message "Email or password incorrect" -->
```

Note: The HTTP status code is **200** (not 401 or 403). Flask's `render_template()` returns a 200 by default. The error is communicated through the flash message displayed in the HTML, not through the HTTP status code.

#### API Login Path

`app/api/views/auth.py` (lines 64–66):

```python
if not user or not user.check_password(password):
    LoginEvent(LoginEvent.ActionType.failed, LoginEvent.Source.api).send()
    return jsonify(error="Email or password incorrect"), 400
```

**Line 64**: Same credential check as the web path.

**Line 65**: Emits a New Relic event with explicit `Source.api`:
```json
{"action": "failed", "source": "api"}
```

**Line 66**: Returns a JSON error response with HTTP 400.

**Expected API response:**

```
HTTP/1.1 400 Bad Request
Content-Type: application/json

{"error": "Email or password incorrect"}
```

#### After-Request Logging (Both Paths)

`server.py` (lines 272–296) — The `after_request` handler logs every request (excluding static files):

```python
LOG.d(
    "%s %s %s %s %s, takes %s",
    request.remote_addr,
    request.method,
    request.path,
    request.args,
    res.status_code,
    time.time() - start_time,
)
```

**Expected log output for web login failure:**
```
127.0.0.1 POST /auth/login ImmutableMultiDict([]) 200, takes 0.05
```

**Expected log output for API login failure:**
```
127.0.0.1 POST /api/auth/login ImmutableMultiDict([]) 400, takes 0.03
```

Additionally, `server.py` (lines 293–295):
```python
newrelic.agent.record_custom_event(
    "HttpResponseStatus", {"code": res.status_code}
)
```

This records a second New Relic event:
- Web path: `{"code": 200}`
- API path: `{"code": 400}`

### Summary of New Relic Events Emitted on Login Failure

| Event Type | Payload (Web) | Payload (API) |
|---|---|---|
| `LoginEvent` | `{"action": "failed", "source": "web"}` | `{"action": "failed", "source": "api"}` |
| `HttpResponseStatus` | `{"code": 200}` | `{"code": 400}` |

### Summary of All Side Effects

| Side Effect | Web Path | API Path |
|---|---|---|
| HTTP Status Code | `200` | `400` |
| Response Body | Rendered HTML with flash message | `{"error": "Email or password incorrect"}` |
| Flash Message | `"Email or password incorrect"` (category: `"error"`) | N/A (JSON API) |
| Rate Limit Deduction | Yes (`g.deduct_limit = True`) | Yes (via `@limiter.limit("10/minute")` on `auth_login` at `app/api/views/auth.py` line 30) |
| Password Cleared | Yes (`form.password.data = None`) | N/A (no form object) |
| LoginEvent to New Relic | `{"action": "failed", "source": "web"}` | `{"action": "failed", "source": "api"}` |
| Request Log | `{ip} POST /auth/login ... 200, takes {duration}` | `{ip} POST /api/auth/login ... 400, takes {duration}` |
| HttpResponseStatus to New Relic | `{"code": 200}` | `{"code": 400}` |

### Supporting Evidence

- `app/events/auth_event.py` (lines 6–25): Complete `LoginEvent` class with `ActionType` enum (success=0, failed=1, disabled_login=2, not_activated=3, scheduled_to_be_deleted=4) and `Source` enum (web=0, api=1)
- `app/api/views/auth.py` (lines 29–30): Rate limiter on API login: `@limiter.limit("10/minute")` — note that unlike the web path, the API path deducts on EVERY request (no `deduct_when` condition), meaning both successful and failed API logins count against the rate limit
- `app/auth/views/login.py` (lines 22–24): Web rate limiter only deducts on failure via `deduct_when=lambda r: hasattr(g, "deduct_limit") and g.deduct_limit`
