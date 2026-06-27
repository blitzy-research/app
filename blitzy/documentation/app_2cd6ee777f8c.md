# SimpleLogin Server‑Side Sessions: Runtime Behavior of the `pickle` Deserialization Path and the Harmless‑Reset ↔ RCE Boundary

> **Investigation document** — answers five questions about how the SimpleLogin Flask application handles **server‑side session data at runtime**, with a focus on the `pickle` deserialization path in `app/session.py` and the precise boundary between a harmless session reset and a genuine remote‑code‑execution (RCE) risk. Every conclusion is anchored to an exact code locator (`file:Lxx`) and a short rationale, and the behavioral claims are backed by **runtime observation of the actual running application** (not theory). This is a documentation‑only analysis; no application code is changed, and any mitigation is discussed analytically only.

## Abstract

SimpleLogin optionally stores Flask session **data** server‑side in Redis, pickled, while the browser holds only an itsdangerous‑signed session **UUID** in the `slapp` cookie. When that store is active, the request path deserializes the Redis bytes with `pickle.loads` inside a bare `try/except Exception: pass` (`app/session.py:L76`, `L78‑L79`). Empirically, malformed bytes therefore cause the session to be **silently reset** to a fresh anonymous session — **no HTTP 500, no propagated exception, and no log line** — while a *well‑formed but malicious* pickle whose payload defines `__reduce__` **executes arbitrary code during `pickle.loads` without raising**. The cookie signature (effectively **HMAC‑SHA1** for the pinned `itsdangerous` 1.1.0) only authenticates *which* Redis key is selected; it does **not** authenticate the integrity of the bytes stored at that key. **The central thesis of this document is therefore that the meaningful trust boundary for the deserialization risk is _Redis write access_, not the cookie signature.** A party who can write Redis but cannot forge the signed ID can still poison **their own** session key — for which they legitimately hold a valid cookie — and trigger `loads` on their own request. This mirrors the python‑socketio Redis+pickle precedent (GHSA‑g8c6‑8fjj‑2r4m).

---

## 1. Overview — Session architecture and the `MEM_STORE_URI` activation gate

### 1.1 The activation gate (scope of everything that follows)

**The entire `pickle`/Redis session path is active _only_ when `MEM_STORE_URI` is set to a `redis://`, `rediss://`, or `redis+sentinel://` URL.** The application factory wires the custom session store only inside the `if MEM_STORE_URI:` block:

```python
# server.py:L163-L165
if MEM_STORE_URI:
    app.config[flask_limiter.extension.C.STORAGE_URL] = MEM_STORE_URI
    initialize_redis_services(app, MEM_STORE_URI)
```

`initialize_redis_services` installs `RedisSessionStore` as `app.session_interface` for `redis://`/`rediss://` URLs (via `limits.storage.RedisStorage`) and for `redis+sentinel://` (via `RedisSentinelStorage`); any other value raises `RuntimeError`:

```python
# app/redis_services.py:L9-L25
def initialize_redis_services(app: flask.Flask, redis_url: str):
    if redis_url.startswith("redis://") or redis_url.startswith("rediss://"):
        storage = limits.storage.RedisStorage(redis_url)
        app.session_interface = RedisSessionStore(storage.storage, storage.storage, app)
        ...
    elif redis_url.startswith("redis+sentinel://"):
        ...
    else:
        raise RuntimeError(... )
```

`MEM_STORE_URI` itself is read from the environment and defaults to `None` (`app/config.py:L568`), so when it is unset the `if` block is skipped entirely.

**Rationale / consequence:** When `MEM_STORE_URI` is **unset**, `app.session_interface` is never replaced, so Flask's **default signed‑cookie session** is used, the session data lives (signed) in the cookie, and **no `pickle`/Redis deserialization happens at all** — the five questions below are moot for that configuration. *All conclusions in this document are scoped to the active path* (a `MEM_STORE_URI` redis URL is configured). The test harness activates exactly this path: `tests/test.env:L78` sets `MEM_STORE_URI=redis://localhost`. The runtime check `isinstance(app.session_interface, RedisSessionStore)` returned **`True`** under that configuration (observed‑from‑running‑app).

### 1.2 Cookie vs. server‑side storage

The `slapp` cookie carries **only** the itsdangerous‑signed session UUID; the session **data** lives server‑side in Redis under the key `session:<uuid>`, pickled — it never transits the client.

- Cookie name: `SESSION_COOKIE_NAME = "slapp"` (`app/config.py:L199`), applied to the app config at `server.py:L159`.
- Redis key format: `_get_key` returns `f"{SESSION_PREFIX}:{session_Id}"`, i.e. `session:<uuid>`, where `SESSION_PREFIX = "session"` (`app/session.py:L18`, `L43-L45`).
- Secret key for signing: `app.secret_key = FLASK_SECRET` (`server.py:L151`; `FLASK_SECRET` is required and non‑empty per `app/config.py:L196-L198`).
- Cookie hardening present (orthogonal to deserialization): `SESSION_COOKIE_SAMESITE = "Lax"` (`server.py:L162`) and `SESSION_COOKIE_SECURE = True` when the site URL is https (`server.py:L161`).

**Rationale:** Because only the signed UUID is in the cookie, an end user can read their session id but cannot read or alter the server‑side session contents through the cookie — the contents are addressed indirectly via the Redis key. This separation is exactly what makes the *integrity of the Redis bytes* (Q5) the operative question.

### 1.3 The two classes that implement the store

- `ServerSession(CallbackDict, SessionMixin)` (`app/session.py:L21-L28`): a dict‑backed session whose `on_update` callback sets `self.modified = True` (`L23-L24`); it also carries a `session_id`.
- `RedisSessionStore(SessionInterface)` (`app/session.py:L31-L114`): the custom Flask session interface providing `open_session` (read path), `save_session` (write path), `extract_and_validate_session_id`, `purge_session`, and the signer/key helpers.

### 1.4 The decision path (read‑side)

The diagram below summarizes the read path (`open_session`) that all later sections analyze. It is the spine of the investigation: signature → key selection → `pickle.loads` → benign restore vs. harmless reset vs. code execution.

```mermaid
flowchart TD
    A["Incoming request with slapp cookie"] --> B{"Signature valid?<br/>extract_and_validate_session_id<br/>app/session.py:L47-L59"}
    B -- "No / no cookie" --> F["Fresh ServerSession (new uuid)<br/>app/session.py:L70-L71 and L80"]
    B -- "Yes" --> C["Read Redis key session:UUID<br/>app/session.py:L73-L74"]
    C --> D{"Key present?"}
    D -- "No" --> F
    D -- "Yes" --> E["pickle.loads(val)<br/>app/session.py:L76 — DESERIALIZATION SINK"]
    E -- "Valid benign pickle" --> G["Restore session dict (normal)<br/>app/session.py:L77"]
    E -- "Garbage / truncated / empty" --> H["except Exception: pass<br/>app/session.py:L78-L79"]
    H --> F
    E -- "Valid MALICIOUS pickle (__reduce__)" --> I["Arbitrary code executes at loads<br/>DESERIALIZATION RISK"]
    F --> J["New slapp Set-Cookie<br/>no HTTP 500, no log line"]
```

> **Single‑sink fact:** `pickle` is imported and used in exactly one file in the whole repository — `app/session.py` (confirmed via `grep -rln pickle app/ server.py` → only `app/session.py`). There is no other code path that deserializes session bytes, so this analysis is complete with respect to the five questions.

---

## 2. Q1 — What normal sessions look like as users log in and out

**Answer in one line:** A logged‑in session is a small Python dict (e.g. `_user_id`, `_fresh`, `sudo_time`, a CSRF token, and — with `session_protection="strong"` — a client‑identity `_id`), pickled into Redis under `session:<uuid>` with a 7‑day TTL for authenticated sessions (300 s for anonymous), addressed by an HMAC‑SHA1‑signed `slapp` cookie that holds only the UUID; logout destroys the authenticated session (deletes its Redis key and rotates the UUID) and the web route emits delete-cookie headers for `slapp`/`mfa`/`dark-mode`, but `save_session` then runs unconditionally on the same response and re-issues a *new* signed `slapp` cookie for a fresh **anonymous** session — so the browser is left holding a rotated, anonymous session id rather than no cookie at all.

### 2.1 Login — how the dict gets populated

The primary web login path is `after_login` (`app/auth/views/login_utils.py:L12`). On a non‑2FA (or post‑2FA) success it calls Flask‑Login's `login_user` and stamps a sudo timestamp:

```python
# app/auth/views/login_utils.py:L36-L37
login_user(user)
session["sudo_time"] = int(time())
```

- `login_user(user)` (`L36`) is what populates the Flask‑Login keys in the session: `_user_id` (the value `open_session` later keys the TTL branch on) and `_fresh`. Because `login_manager.session_protection = "strong"` (`app/extensions.py:L8`), Flask‑Login additionally stores a client‑identity hash under `_id` to pin the session to a browser fingerprint. **Rationale:** these keys are written by Flask‑Login *into the Flask `session`*, and the `session` is exactly what `save_session` later pickles (§2.3).
- `session["sudo_time"]` (`L37`) records when the user authenticated, used to gate "sudo"-protected actions.
- For 2FA‑pending flows, `after_login` instead stores a marker `session[MFA_USER_ID] = user.id` (FIDO path `L23`, OTP path `L29`) where `MFA_USER_ID = "mfa_user_id"` (`app/config.py:L295`) **without** calling `login_user`. So a 2FA‑pending session is *anonymous* (no `_user_id`) until the second factor completes.

A Flask‑WTF `csrf_token` is also commonly present in the dict (the in‑code comment at `app/session.py:L93-L94` explicitly notes the CSRF token is the reason anonymous sessions are persisted at all).

### 2.2 Other session writers — the full set of values that can be pickled

The dict that gets pickled is whatever the request handlers have written into `session`. The complete set of writers across the codebase (line numbers **re‑verified against current source**) is:

| Key(s) written | Where (verified locator) | Notes |
|---|---|---|
| `_user_id`, `_fresh`, `_id` | via `login_user(...)` — `login_utils.py:L36`, `recovery.py:L55`, `change_email.py:L38`, `fido.py:L64`/`L112`, `api_to_cookie.py:L24`, `mfa.py:L57`/`L75`, `activate.py:L50`, `api/views/auth_mfa.py:L73`, `api/views/auth.py:L363` | All call sites that authenticate → `_user_id` present → 7‑day TTL |
| `sudo_time` | `login_utils.py:L37`, `dashboard/views/enter_sudo.py:L34`, `fido.py:L111`; reset to `0` in `internal/exit_sudo.py:L8` | "sudo" timestamp |
| `mfa_user_id` (= `MFA_USER_ID`) | written `login_utils.py:L23`/`L29`; deleted `mfa.py:L71`, `recovery.py:L53` | 2FA‑pending marker (anonymous session) |
| `fido_uuid`, `fido_challenge` | `dashboard/views/fido_setup.py:L119-L120`; `fido_challenge` also `fido.py:L136` | WebAuthn/FIDO registration & login state |
| `oauth_state` / `oauth_redirect_next` (via `SESSION_STATE_KEY` / `SESSION_NEXT_KEY`) | `auth/views/oidc.py:L44-L45` (constants defined `L25-L26`) | OIDC login state |
| `oauth_state`, `google_next_url` | `auth/views/google.py:L43`, `L37` | Google OAuth state |
| `oauth_state`, `facebook_next_url` | `auth/views/facebook.py:L47`, `L38` | Facebook OAuth state |
| `oauth_state` | `auth/views/github.py:L34` | GitHub OAuth state |
| `oauth_next`, `oauth_scheme`, `oauth_mode`, `oauth_state`, `oauth_action` (via `SESSION_STATE_KEY`/`SESSION_ACTION_KEY`) | `auth/views/proton.py:L85`,`L91`,`L97`/`L99`,`L105`,`L106` (constants `L36-L37`) | Proton OAuth state |
| `slref` (referral) | **read** at `login_utils.py:L62`; the referral value otherwise arrives via the `slref` *cookie* (`_REFERRAL_COOKIE`, `L49`) | Read‑side only in this module |

> **Peripheral‑locator caveat acted upon:** the OAuth/OIDC state is written at `oidc.py:L44-L45` *through* the named constants `SESSION_STATE_KEY`/`SESSION_NEXT_KEY` (defined at `L25-L26` with the string values `"oauth_state"`/`"oauth_redirect_next"`), **not** as a literal `session["oauth_state"]` at `L25-26`. `activate.py`, `change_email.py`, `api_to_cookie.py`, `api/views/auth.py`, and `api/views/auth_mfa.py` contain **no** `session[...]` writes — they populate the session purely via `login_user` (`activate.py:L50`, `change_email.py:L38`, `api_to_cookie.py:L24`, `api/views/auth.py:L363`, `api/views/auth_mfa.py:L73`). `app/admin_model.py` has **no active session writer or login path at all**: its `login_as` action — including the `login_user(user)` call at `L317` — is entirely **commented out** (`admin_model.py:L305-L319`), so it never mutates the session at runtime. `reset_password.py:L71-L73` does **not** call `login_user` *directly* (in‑code comment at `L71`: "do not use `login_user(user)` here"); instead it **delegates to `after_login`** (`login_utils.py:L12-L45`), so a password reset **can** still mint or mutate a session: for **non‑MFA** users `after_login` calls `login_user(user)` and writes `session["sudo_time"]` (`login_utils.py:L36-L37`), minting an **authenticated** session, while for **MFA/FIDO** users it writes `session[MFA_USER_ID]` (`login_utils.py:L23`/`L29`), creating a **2FA‑pending anonymous** session.

### 2.3 Serialization and TTL — how the dict reaches Redis

`save_session` pickles the *entire* session dict and stores it with an expiry, then signs the UUID and emits the cookie:

```python
# app/session.py:L91-L101 (excerpt)
val = pickle.dumps(dict(session))                      # L91
ttl = int(app.permanent_session_lifetime.total_seconds())  # L92
# We need to keep the non-authenticated ones because the csrf token is stored in the session.
if "_user_id" not in session:                          # L95
    ttl = 300                                          # L96
self._redis_w.setex(name=self._get_key(session.session_id), value=val, time=ttl)  # L97-L101
```

- **Serializer:** `pickle.dumps(dict(session))` (`L91`). The `import` resolves to the standard‑library `pickle` on CPython 3 (the `cPickle` alias only exists on Python 2; `app/session.py:L9-L12`).
- **TTL selection (the discriminating behavior):** the TTL is `app.permanent_session_lifetime` *unless* `"_user_id"` is absent, in which case it is pinned to **300 s** (`L95-L96`). The `make_session_permanent` before‑request hook sets `session.permanent = True` and `app.permanent_session_lifetime = timedelta(days=7)` on every request (`server.py:L205-L207`), so **authenticated** sessions get a **7‑day** TTL while **anonymous / 2FA‑pending** sessions get **300 s**. **Rationale:** the presence of `_user_id` is *exactly* the predicate at `L95`, which is why the `login_user` call sites in §2.2 determine the long‑lived branch.
- **Cookie emission:** the UUID is signed and written as the `slapp` cookie (`L102-L114`), with the hardening attributes from §1.2.

**Empirical confirmation (observed from the running app; `MEM_STORE_URI=redis://localhost`):**

- `pickle.loads(pickle.dumps(d)) == d` is **True** for representative session dicts (exact round‑trip).
- Real `session:<uuid>` payload sizes read back with the Redis client — **representative, since the exact size depends on the dict contents**:
  - minimal authenticated `{_user_id, _fresh, sudo_time}` → **58 bytes**, TTL observed as the `permanent_session_lifetime` (non‑300) branch.
  - fuller authenticated, adding a 128‑hex `_id` and a 40‑hex `csrf_token` → **251 bytes**.
  - anonymous `{csrf_token}` → **71 bytes**, TTL observed as **exactly 300 s** (`L95-L96`).
- The TTL for an authenticated session, observed **through the real `before_request` path** (so `make_session_permanent` ran), was **604800 s (= 7 days)**. *(When `save_session` is invoked in isolation — bypassing `before_request` — Flask's default `permanent_session_lifetime` of 31 days/2 678 400 s applies instead; this confirms the TTL is driven entirely by `permanent_session_lifetime` as set at `server.py:L207`.)*
- The emitted `slapp` cookie has the shape `"<uuid>.<base64url‑sig>"`, e.g. `7d5e9e49-677f-4cb3-9d12-5f178e94503e.tfPbbgkl6PpRoN4DRVgQCcZNsJo` — UUID, a literal `.`, then a 27‑character signature (see §6 for the signature analysis).

### 2.4 Logout — how the session is destroyed

Both web and API logout funnel through `logout_session()`:

```python
# app/session.py:L117-L121
def logout_session():
    logout_user()
    purge_fn = getattr(current_app.session_interface, "purge_session", None)
    if callable(purge_fn):
        purge_fn(session)
```

`purge_session` deletes the Redis key and rotates the session id:

```python
# app/session.py:L61-L66
def purge_session(self, session: ServerSession):
    try:
        self._redis_w.delete(self._get_key(session.session_id))   # L63 — delete session:<uuid>
        session.session_id = str(uuid.uuid4())                    # L64 — rotate the id
    except AttributeError:
        pass
```

- **Web logout** (`app/auth/views/logout.py:L10`) calls `logout_session()` and additionally deletes the `slapp`, `mfa`, and `dark-mode` cookies (`L13-L15`).
- **API logout** (`app/api/views/user_info.py:L140`) mirrors this via `logout_session()`.

**Rationale:** `logout_user()` (Flask‑Login) clears the identity keys from the dict; `purge_session()` then **removes the server‑side bytes** and rotates the UUID so the old key is unrecoverable. The web route then emits **delete-cookie** headers for `slapp`/`mfa`/`dark-mode` (`app/auth/views/logout.py:L13-L15`). **However, the next request is _not_ left without a session id.** `save_session` runs unconditionally during response finalization (`app/session.py:L82-L114`) — it does *not* check `session.modified` or whether the dict is empty — and the logout request still leaves a **non-empty, modified** session: the route flashes `"You are logged out"` (adding `_flashes`), `make_session_permanent` has set `_permanent` (`server.py:L205-L207`), and `logout_user()` removes only the Flask-Login identity keys, leaving the residual `csrf_token` and `sudo_time`. Consequently `save_session` pickles that now-anonymous dict into a **new** `session:<new-uuid>` key with the 300 s anonymous TTL (`app/session.py:L91`, `L95-L96`) and appends a **later** `slapp=<new-uuid>.<sig>` `Set-Cookie` (`app/session.py:L102-L114`) **after** the route's delete-cookie header. Two `slapp` `Set-Cookie` directives therefore coexist on the logout response; a client applies them in order, so the **later `set` wins** and the browser keeps a *valid, rotated, **anonymous*** `slapp` cookie. The next request thus carries the new anonymous id and (within the 300 s window) takes the `B -- Yes` branch against the freshly written anonymous key — it does **not** arrive session-less. The security conclusion is unchanged — the **authenticated** session is destroyed and cannot be resumed — but the precise final state is a *rotated anonymous session*, not a session-less client (empirically confirmed below).

**Empirical confirmation (observed from the running app; `MEM_STORE_URI=redis://localhost`, `RedisSessionStore` active).** A test-client login created `session:<old>` containing `_user_id` (TTL `604800` s). `GET /auth/logout` then returned **HTTP 302** whose `Set-Cookie` headers were, *in order*: (1) `slapp=; Expires=Thu, 01-Jan-1970 …; Max-Age=0` plus the matching `mfa=` / `dark-mode=` deletions (the route's `delete_cookie`), followed by (4) `slapp=<new-uuid>.<sig>; Domain=.sl.test; Expires=…; HttpOnly; Path=/; SameSite=Lax` (from `save_session`). After the response, `EXISTS session:<old>` → `0` (TTL `-2`, deleted) while `EXISTS session:<new>` → `1` with TTL `300`, and the new key's unpickled dict held `{_flashes, _permanent, csrf_token, sudo_time}` — **no `_user_id`** (i.e. anonymous). The client cookie jar's final `slapp` value `unsign`ed (via `RedisSessionStore._get_signer(app)`) to the rotated `<new-uuid>` (≠ the old UUID). With CSRF disabled (as in the pytest harness) the residual dict is `{_flashes, _permanent, sudo_time}`; the `csrf_token` simply reflects whether Flask-WTF wrote one. **Net: the old authenticated session is destroyed, but the next request carries a new _anonymous_ `slapp` session id, not no session id.**


---

## 3. Q2 — What happens if the session data in Redis is corrupted or malformed?

**Answer: (b) the session is silently reset.** Not (a) fail, not (c) surface an error.

### 3.1 The read path and the bare exception guard

```python
# app/session.py:L68-L80
def open_session(self, app: flask.Flask, request: flask.Request):
    session_id = self.extract_and_validate_session_id(app, request)   # L69
    if not session_id:
        return ServerSession(session_id=str(uuid.uuid4()))            # L70-L71 (no/invalid cookie)

    val = self._redis_r.get(self._get_key(session_id))                # L73
    if val is not None:                                               # L74
        try:                                                          # L75
            data = pickle.loads(val)                                  # L76 — THE DESERIALIZATION SINK
            return ServerSession(data, session_id=session_id)         # L77
        except Exception:                                             # L78 — catches EVERYTHING
            pass                                                      # L79 — silent: no log, no re-raise
    return ServerSession(session_id=str(uuid.uuid4()))                # L80 — fresh anonymous session
```

**Walk‑through and rationale:**
1. The signed `slapp` cookie is validated to a UUID (`L69`); a missing/invalid cookie short‑circuits to a fresh session (`L70-L71`).
2. The Redis bytes for `session:<uuid>` are fetched (`L73`). If the key is absent (`val is None`), control falls through to `L80` → a fresh session.
3. If bytes exist, `pickle.loads(val)` runs at **`L76`** inside a `try` whose `except Exception: pass` (`L78-L79`) catches **every** exception type.
4. **Malformed bytes raise inside `pickle.loads`; the bare `except` swallows the exception; control falls through to `L80`**, returning `ServerSession(session_id=str(uuid.uuid4()))` — a **fresh, empty, anonymous** session with a newly minted UUID.

Because `except Exception` is the broadest practical catch (it covers `pickle.UnpicklingError`, `EOFError`, `ValueError`, `AttributeError`, `ImportError`, etc.), there is no class of *malformed* input that escapes it. The result is option **(b)**: the corrupted session is **silently reset**, not surfaced as an error and not a hard failure.

### 3.2 Empirical confirmation (observed from the running app)

Using the real `open_session` with a *valid signed cookie* whose backing Redis key was then overwritten with corrupt bytes:

| Corrupt payload | `pickle.loads` raises | `open_session` result |
|---|---|---|
| garbage ASCII (`b"this is not a pickle at all"`) | `UnpicklingError` | fresh session: id rotated, `_user_id` absent, dict `{}` |
| random binary (`os.urandom(64)`) | `UnpicklingError` | fresh session (silent reset) |
| truncated valid pickle (first 10 bytes) | `UnpicklingError` | fresh session (silent reset) |
| empty (`b""`) | `EOFError` | fresh session (silent reset) |

In **every** case `open_session` returned a fresh `ServerSession` and **no exception propagated to the caller**. **The stale corrupted key remained in Redis** after the reset (it is not deleted on the read path); it simply expires when its TTL lapses. **Rationale:** the only write that happens on the corrupted request is the subsequent `save_session`, which stores the *new* (empty) session under the *new* UUID — the old key is orphaned, not purged.

---

## 4. Q3 — What shows up in HTTP responses and in logs when that happens?

**Answer: nothing alarming. There is no HTTP 500, no propagated exception, and no log line from this path. The only observable artifact is a new `slapp` `Set-Cookie` on the next response.**

### 4.1 Why — the rationale from the code

The `except` body is a literal `pass` (`app/session.py:L79`): it contains **no `LOG.…` call, no `raise`, no metric, no side effect of any kind**. Consequently:
- **No exception escapes `open_session`**, so Flask's error handling is never engaged and **no HTTP 500** is produced by this path.
- **No log line is emitted** for the corruption event, because nothing in the `try/except` logs (contrast with the rest of the codebase, which logs liberally via `app/log.py:LOG`).
- A fresh session id is created, so on the way out `save_session` writes a new `session:<uuid>` key and emits a **new `slapp` `Set-Cookie`** (`app/session.py:L102-L114`). That cookie rotation is the **sole** externally observable signal.

### 4.2 Empirical confirmation (observed from the running app)

Driving a real request through the Flask test client: a first request to `/auth/login` returned **HTTP 200** and issued a `slapp` `Set-Cookie`. The backing Redis key was then overwritten with corrupt bytes and the request replayed with the same cookie. The replayed (corrupted‑session) request:
- returned **HTTP 200 — not 500**;
- issued a **fresh `slapp` `Set-Cookie`** (new signed UUID); and
- produced **zero** log lines mentioning session pickling/corruption/error (a logging handler attached for the duration of the request captured **0** such lines).

**Implication:** corruption of server‑side session bytes is, by design of this `except` branch, **invisible** to operators via both HTTP status and logs. (A mitigation — e.g. logging in the `except` — is discussed analytically in §5.3; it is **not** implemented here, per scope.)


---

## 5. Q4 — Where is the boundary between a harmless reset and a real deserialization risk?

**Answer (empirical): the boundary is the _validity of the pickle stream_, not merely whether the bytes were altered.** Garbage / truncated / random bytes raise inside `pickle.loads` and are swallowed → harmless reset (Q2/Q3). A **well‑formed pickle whose payload defines `__reduce__`** does **not** raise — it **executes arbitrary code at `pickle.loads` (`app/session.py:L76`)** and then returns a "restored" session. The same line of code (`L76`) is both the harmless‑reset trigger and the RCE sink; which outcome occurs depends solely on whether the bytes form a *valid, malicious* pickle versus *invalid garbage*.

### 5.1 Why `__reduce__` is the primitive

`pickle` is a *programmable* serialization format: during unpickling, an object's `__reduce__` method tells the unpickler which callable to invoke and with which arguments to reconstruct the object. An attacker‑authored class whose `__reduce__` returns `(os.system, (cmd,))` therefore causes `os.system(cmd)` to run **as part of deserialization** — before any application code inspects the result. Crucially, such a stream is a *valid* pickle, so `pickle.loads` **completes successfully and returns** rather than raising — which means it never reaches the `except Exception: pass` branch at all.

### 5.2 Empirical confirmation (observed from the running app)

A crafted pickle was produced from a class whose `__reduce__` returned `(os.system, (cmd,))`, where `cmd` wrote a marker file. The well‑formed payload was stored at a **valid, signed** `session:<uuid>` key, and the **real `open_session`** was invoked over it:

- crafted pickle size ≈ **105 bytes** *(representative — the exact size depends on the embedded command string; a shorter command yields ≈ 93 bytes)*;
- the marker file **did not exist before** `pickle.loads` and **existed afterward** → the shell command **executed during `pickle.loads`** (`app/session.py:L76`);
- `open_session` **did not raise** and returned a "restored" `ServerSession` → confirming the malicious case **does not trip the `except` branch**.

This is the crux of Q4: **the bare `except` only protects against _invalid_ bytes; it provides no protection whatsoever against _valid, malicious_ bytes.** A defender watching for resets/errors (Q3) sees nothing in the RCE case either — the dangerous path is *quieter* than the harmless one.

### 5.3 Mitigations — analytical discussion only (NOT implemented here, per scope)

The following are described for completeness; **none is implemented in this repository as part of this task**, and the AAP explicitly places remediation out of scope:
- **Replace `pickle` with a safe serializer** (JSON or MessagePack) for session data. This removes the code‑execution primitive entirely because those formats deserialize to inert data structures, not arbitrary callables. (Trade‑off: only JSON‑serializable values may be stored.)
- **Authenticate the Redis payload** with an HMAC/MAC keyed by `FLASK_SECRET` (sign the *bytes*, verify before `loads`). This would make tampered bytes fail an integrity check *before* reaching `pickle.loads`, closing the Q5 byte‑integrity gap.
- **Log (and/or alert on) the `except` branch** (`app/session.py:L78-L79`) so resets/corruption become observable (directly addresses Q3's invisibility).
- **Defensively also delete the offending key** on a deserialization failure, instead of leaving a stale key to expire by TTL.

Each is a *design change to `app/session.py` or its dependencies* and is therefore deliberately **not** applied here.

---

## 6. Q5 — If an attacker can tamper with session bytes but cannot forge the signed session ID, does that change the risk?

**Answer: it constrains _how_ an attacker delivers a malicious pickle, but it does not remove the risk — because the signature authenticates _key selection_, not _payload integrity_. The operative trust boundary is _Redis write access_.**

### 6.1 What the signature actually gates

`open_session` first resolves the cookie to a UUID via `extract_and_validate_session_id`:

```python
# app/session.py:L47-L59
@classmethod
def extract_and_validate_session_id(cls, app, request) -> Optional[str]:
    unverified_session_Id = request.cookies.get(app.session_cookie_name)  # L51
    if not unverified_session_Id:
        return None                                                       # L52-L53
    signer = cls._get_signer(app)
    try:
        sid_as_bytes = signer.unsign(unverified_session_Id)               # L56
        return sid_as_bytes.decode()                                      # L57
    except itsdangerous.BadSignature:
        return None                                                       # L58-L59
```

The signer is:

```python
# app/session.py:L37-L41
itsdangerous.Signer(app.secret_key, salt="session", key_derivation="hmac")
```

For the pinned **itsdangerous 1.1.0**, the default `digest_method` is **SHA‑1**, and `key_derivation="hmac"` derives the signing key as `HMAC‑SHA1(secret_key, salt)`; the signature is then `HMAC‑SHA1(key, value)`. So the `slapp` signature is effectively **HMAC‑SHA1**.

**The decisive point:** if the signature does not verify, `unsign` raises `BadSignature` and `extract_and_validate_session_id` returns `None` (`L58-L59`); `open_session` then short‑circuits at `L70-L71` and **never reads any Redis key**. Therefore **a forged or tampered session ID never causes the corresponding Redis key to be read** — the signature governs **which key is selected**, full stop. It says *nothing* about the integrity of the bytes stored at that key: those bytes are never re‑signed or verified against the cookie. **Their integrity is governed entirely by who can write to Redis.**

### 6.2 Empirical confirmation (observed from the running app, via the real `_get_signer`)

- `digest_method = openssl_sha1`, `key_derivation = "hmac"`, `salt = "session"` → **HMAC‑SHA1** confirmed.
- `sign("11111111-2222-3333-4444-555555555555")` → `"11111111-2222-3333-4444-555555555555.w_aQPf9xb7cP_w-oqM2Y2Yk6dq0"` — exactly the `slapp` `<uuid>.<sig>` format, with a **27‑character** base64url signature (a 20‑byte SHA‑1 digest, unpadded).
- `unsign(valid)` recovers the UUID.
- A **1‑bit flip of the value** → `BadSignature`; a **1‑bit flip of the signature** → `BadSignature`; **signing with a wrong secret** → `BadSignature`. Forging/re‑signing is impossible without `FLASK_SECRET`.
- Feeding a **tampered cookie** to the real `open_session` returned a **fresh session** (new UUID, empty dict) and **never read** the targeted Redis key — empirically confirming the "signature gates key selection" conclusion.

### 6.3 Why the risk is therefore governed by Redis write access

Combine §5 (a valid malicious pickle executes at `L76`) with §6.1 (the signature only selects the key):

- An attacker who can **write to Redis** can place a crafted `__reduce__` pickle at a `session:<uuid>` key. To get it *loaded*, a request must arrive bearing a **validly signed** cookie for that same `<uuid>`.
- Even **without** the ability to forge the signature, an attacker who holds a *legitimate* `slapp` cookie (i.e. their **own** session) can overwrite **their own** `session:<uuid>` key with a malicious pickle and then make a normal request — `open_session` validates *their own* signature, reads *their own* key, and runs `pickle.loads` on the attacker‑controlled bytes (`L76`). The signature requirement is satisfied trivially because it is *their* session.
- Conversely, an attacker who can only manipulate the **cookie** (client‑side) but cannot write Redis gains nothing: tampering the cookie just fails `unsign` (`L58-L59`) → harmless fresh session.

**Therefore the meaningful trust boundary is _Redis write access_, not the cookie signature.** The signature usefully prevents *cookie‑only* / client‑side byte injection (you cannot point a victim at an arbitrary forged key, nor smuggle a payload through the cookie), but it does **not** protect the server‑side bytes once an attacker can write them. This matches the **python‑socketio Redis+pickle precedent, advisory GHSA‑g8c6‑8fjj‑2r4m**, where a server using a Redis backend with `pickle` is exploitable by an attacker who *first* obtains access to the Redis queue and sends a crafted pickle that executes on deserialization — i.e. the boundary there, too, is Redis (message‑queue) write access.

### 6.4 Orthogonal note on `session_protection = "strong"`

`login_manager.session_protection = "strong"` (`app/extensions.py:L8`) causes Flask‑Login to bind a session to a client‑identity hash (stored as `_id`) and to invalidate the *login state* if the fingerprint changes. **This is orthogonal to byte‑level deserialization:** it operates on the *already‑deserialized* dict, **after** `pickle.loads` has run at `L76`. It cannot prevent code execution that occurs *during* `loads`, and it does not authenticate the Redis bytes. It therefore does not change any conclusion above.


---

## 7. Environment and Reproduction

This section records the exact runtime used so that every empirical claim above can be reproduced independently. **All findings labelled "observed from the running app" in §2–§6 were produced inside a live SimpleLogin instance** whose `app/session.py` is **byte‑for‑byte identical** to the version in this branch (verified by `md5sum`: `0af288a039deb1620a154fcc63e77ed8` in both the working tree and the running container `/app/app/session.py`). No claim relies on a re‑implementation; the real `RedisSessionStore.open_session`, `save_session`, and `_get_signer` were invoked directly.

### 7.1 Runtime versions

| Component | Version | Source of truth |
|-----------|---------|-----------------|
| Python | **3.10.18** | `pyproject.toml` `python = "^3.10"`; `Dockerfile` `FROM python:3.10` (L8); CI matrix `["3.10"]` (`.github/workflows/main.yml:L40`) |
| Flask | **1.1.2** | `poetry.lock` (provides `SessionInterface`/`SessionMixin`, `app.session_cookie_name`) |
| Flask‑Login | **0.5.0** | `poetry.lock` (`login_user`/`logout_user`; `session_protection="strong"`) |
| itsdangerous | **1.1.0** | `poetry.lock` (`Signer`; default digest SHA‑1; with `key_derivation="hmac"` ⇒ HMAC‑SHA1) |
| Werkzeug | **1.0.1** | `poetry.lock` (`CallbackDict` base for `ServerSession`) |
| redis (py) | **4.6.0** | `poetry.lock` (Redis client backing the store) |
| limits | **1.5.1** | `poetry.lock` (`limits.storage.RedisStorage` supplies the connection in `initialize_redis_services`) |
| Flask‑Limiter | **1.4** | `poetry.lock` (shares the Redis instance via `MEM_STORE_URI`) |
| PostgreSQL | **13** (harness) / 15 (container) | `scripts/run-test.sh` provisions `postgres:13` on host port `15432` |
| Redis | **7.0.15** | observed in the runtime container `sl-qna-env-0` (`redis-server --version` → `v=7.0.15`; `redis-cli INFO server` → `redis_version:7.0.15`) — the server‑side session backend; `MEM_STORE_URI`/`tests/test.env` do not pin a Redis version |

### 7.2 Activating the server‑side store

The `pickle`/Redis path exists **only** when `MEM_STORE_URI` is a redis URL (§1.1). The investigation loaded `CONFIG=tests/test.env`, which sets:

- `MEM_STORE_URI=redis://localhost` (`tests/test.env:L78`) — installs `RedisSessionStore` as `app.session_interface` (`server.py:L163-L165` → `app/redis_services.py:L9-L25`).
- `DB_URI=postgresql://test:test@localhost:15432/test` (`tests/test.env:L17`) — PostgreSQL on port `15432` (the container adapts this to its local `5432`).
- `FLASK_SECRET=secret` (`tests/test.env:L20`) — the signing key for the `slapp` cookie. (This is a **published test value**, not a production credential.)

The harness mirrors `scripts/run-test.sh`: a Dockerised `postgres:13` (`-p 15432:5432`), `CONFIG=tests/test.env poetry run alembic upgrade head`, then the application is constructed via the `wsgi.py` entry point (`app = create_app()`). At runtime, `initialize_redis_services` was confirmed to have set `app.session_interface` to a `RedisSessionStore` and `app.session_cookie_name == "slapp"`.

### 7.3 Method

1. **Normal session (Q1):** mint a real session by exercising the login flow / a `test_client()` request, then read the resulting key with `redis-cli GET session:<uuid>` to observe the pickled dictionary and its size. The authenticated TTL of **604800 s (7 days)** was confirmed after a real request triggered the `make_session_permanent` before‑request hook (`server.py:L205-L207`); the anonymous TTL of **300 s** follows the `"_user_id" not in session` branch (`app/session.py:L95-L96`).
2. **Corruption (Q2/Q3):** overwrite that key with (a) garbage ASCII, (b) random binary, (c) a truncated valid pickle, and (d) an empty value, then replay a request and observe `open_session`. Every case produced a fresh session, no propagated exception, an HTTP **200** (not 500), no log line, and a new `slapp` `Set-Cookie`; the corrupted key persisted until TTL.
3. **Crafted payload (Q4):** overwrite the key with a well‑formed pickle whose payload defines `__reduce__` → `(os.system, (cmd,))`, where `cmd` writes a marker file. The marker appeared **after** `open_session` ran and `pickle.loads` **did not raise**, demonstrating code execution at `app/session.py:L76`. The marker was removed by the probe.
4. **Signature (Q5):** drive the real `_get_signer(app)` to confirm HMAC‑SHA1, the `<uuid>.<sig>` cookie shape, recovery of a valid UUID, and `BadSignature` on value tamper / signature tamper / wrong secret; then feed a tampered cookie to `open_session` and confirm no Redis key is read.

### 7.4 Repository hygiene

All scratch/observation scripts lived **outside** the repository (under `/tmp`, and copied into the container's `/tmp`) and were deleted afterward; any marker files created by the crafted‑pickle probe were removed. The working tree was verified clean (`git status --porcelain` empty) both before and after observation, with `app/session.py` unchanged. The **only** persistent change introduced by this task is this document, `blitzy/documentation/app_2cd6ee777f8c.md`.

### 7.5 Provenance of each empirical number

| Claim | Provenance |
|-------|-----------|
| Pickle round‑trip equality; representative sizes (~58 / ~251 / ~71 bytes); 300 s vs. 604800 s TTL selection | **Observed from the running app** (real `save_session`/`open_session`, byte‑identical `app/session.py`) |
| Corruption → `UnpicklingError`/`EOFError`, all caught → silent reset; HTTP 200; no log line; stale key persists | **Observed from the running app** |
| Crafted `__reduce__` pickle (~105 bytes here; size depends on the embedded command) executes during `loads` without raising | **Observed from the running app** |
| HMAC‑SHA1 signer; `<uuid>.<27‑char‑sig>` cookie; `BadSignature` on tamper/wrong‑secret; tampered cookie reads no key | **Observed from the running app** (via the real `_get_signer`) |

> Byte sizes are **representative**, not fixed constants — they depend on the session dictionary's contents (number of keys, length of `_id`/`csrf_token`) and, for the crafted case, on the embedded command string. They are provided as concrete illustrations, not invariants.

---

## 8. References and Citations

**Code is the source of truth.** The locators below ground every conclusion in this document. The external sources that follow are **corroborating only** — they confirm the general risk model and library semantics but do not, by themselves, establish any behavioural claim about this codebase.

### 8.1 Primary code locators (grouped by file)

- **`app/session.py`** — the single `pickle` sink in the repository (confirmed via `grep -rln pickle app/ server.py`):
  - `import cPickle as pickle` / fallback `import pickle` — `L9-L12`
  - `SESSION_PREFIX = "session"` — `L18`
  - `class ServerSession(CallbackDict, SessionMixin)` — `L21-L28`
  - `class RedisSessionStore(SessionInterface)` — `L31-L114`
  - `_get_signer` (`Signer(..., salt="session", key_derivation="hmac")`) — `L37-L41`
  - `_get_key` (Redis key `session:<uuid>`) — `L43-L45`
  - `extract_and_validate_session_id` (cookie read; `unsign`; `BadSignature → None`) — `L47-L59`
  - `purge_session` (delete key; rotate UUID) — `L61-L66`
  - `open_session` (extract id; `get`; **`pickle.loads(val)` L76**; **`except Exception: pass` L78-L79**; fresh session L80) — `L68-L80`
  - `save_session` (**`pickle.dumps(dict(session))` L91**; TTL = `permanent_session_lifetime`; **anonymous TTL 300 s L95-L96**; `setex`; sign UUID; `set_cookie`) — `L82-L114`
  - `logout_session()` (`logout_user()` then `purge_session()`) — `L117-L121`
- **`app/redis_services.py`** — `initialize_redis_services` installs `RedisSessionStore` for `redis://`/`rediss://`/`redis+sentinel://`, else `RuntimeError` — `L9-L25`
- **`server.py`** — `app.secret_key = FLASK_SECRET` (`L151`); `SESSION_COOKIE_NAME` (`L159`); `SESSION_COOKIE_SECURE` (`L161`); `SESSION_COOKIE_SAMESITE = "Lax"` (`L162`); `if MEM_STORE_URI: initialize_redis_services(...)` (`L163-L165`); `make_session_permanent` → `permanent_session_lifetime = timedelta(days=7)` (`L205-L207`)
- **`app/config.py`** — `FLASK_SECRET = os.environ["FLASK_SECRET"]` (`L196`); `SESSION_COOKIE_NAME = "slapp"` (`L199`); `MFA_USER_ID = "mfa_user_id"` (`L295`); `MEM_STORE_URI = os.environ.get("MEM_STORE_URI", None)` (`L568`); dotenv `CONFIG` loader (`L65-L71`)
- **`app/extensions.py`** — `login_manager.session_protection = "strong"` (`L8`)
- **`app/auth/views/login_utils.py`** — `after_login` (`L12`); `session[MFA_USER_ID]` on 2FA‑pending flows (`L23`, `L29`); `login_user(user)` (`L36`); `session["sudo_time"] = int(time())` (`L37`); referral `slref` read (`L62`)
- **`app/auth/views/logout.py`** — `logout_session()` (`L10`); deletes `slapp`/`mfa`/`dark-mode` cookies (`L13-L15`)
- **`app/api/views/user_info.py`** — API logout via `logout_session()` (`L140`)
- **`app/internal/exit_sudo.py`** — `session["sudo_time"] = 0` (`L8`)
- **Other session writers (line numbers re‑verified against current source):** `app/auth/views/oidc.py` (constants `SESSION_STATE_KEY="oauth_state"`/`SESSION_NEXT_KEY="oauth_redirect_next"` at `L25-L26`; **writes** `session[SESSION_STATE_KEY]`/`session[SESSION_NEXT_KEY]` at `L44-L45`); `google.py` (`L37`, `L43`); `facebook.py` (`L38`, `L47`); `github.py` (`L34`); `proton.py` (`L85`/`L91`/`L97`/`L99`/`L105`/`L106`); `mfa.py` (`del session[MFA_USER_ID]` `L71`; `login_user` `L57`/`L75`); `fido.py` (`session["fido_challenge"]` `L136`; `session["sudo_time"]` `L111`; `del session[MFA_USER_ID]` `L109`; `login_user` `L64`/`L112`); `recovery.py` (`del session[MFA_USER_ID]` `L53`; `login_user` `L55`); `dashboard/views/enter_sudo.py` (`session["sudo_time"]` `L34`); `dashboard/views/fido_setup.py` (`session["fido_uuid"]`/`session["fido_challenge"]` `L119-L120`)
- **`login_user` call sites populating `_user_id` (⇒ 7‑day TTL):** `login_utils.py:L36`, `recovery.py:L55`, `change_email.py:L38`, `fido.py:L64`/`L112`, `api_to_cookie.py:L24`, `mfa.py:L57`/`L75`, `activate.py:L50`, `api/views/auth_mfa.py:L73`, `api/views/auth.py:L363`. `reset_password.py:L71-L73` does **not** call `login_user` *directly* but **delegates to `after_login`** (`login_utils.py:L12-L45`), which authenticates non‑MFA users (`login_user` at `L36`) or creates `MFA_USER_ID` 2FA‑pending state for MFA users (`L23`/`L29`). `admin_model.py` has **no active session writer** — its `login_as`/`login_user(user)` code is commented out (`L305-L319`).
- **Build / run / version context:** `tests/test.env` (`DB_URI` `L17`; `FLASK_SECRET` `L20`; `MEM_STORE_URI` `L78`); `scripts/run-test.sh` (`postgres:13` on `15432`; `alembic upgrade head`; pytest); `wsgi.py` (`app = create_app()`); `Dockerfile` (`FROM python:3.10`); `pyproject.toml` (`python = "^3.10"`); `.github/workflows/main.yml` (CI matrix `["3.10"]`); `poetry.lock` (pinned versions in §7.1)

### 8.2 Corroborating external sources

*(Corroborating only; the code above is the source of truth. Paraphrased, not quoted at length.)*

- **Python `pickle` deserialization → RCE via `__reduce__`.** General security guidance establishes that `pickle.loads` on attacker‑controlled bytes can execute arbitrary code, with an object's `__reduce__` method as the exploitation primitive, and that `pickle` is unsafe for untrusted input (safer alternatives include JSON/MessagePack). Sources: Semgrep, *"Insecure Deserialization in Python"* (semgrep.dev); dhound, *"Pickle Code Execution"* (knowledge.dhound.io) — the latter specifically warns against storing session data with `pickle` and recommends signed cookies or database‑backed sessions. These corroborate §4–§5.
- **Directly analogous precedent — python‑socketio.** Advisory **GHSA‑g8c6‑8fjj‑2r4m** (≡ **CVE‑2025‑61765**, CWE‑502; python‑socketio < 5.14.0) describes a setup closely analogous to SimpleLogin's: servers using a **Redis** message‑queue backend encode inter‑server messages with `pickle` and deserialize them via `pickle.loads()` on receipt; an attacker who **first gains access to the Redis queue** can send a crafted pickle that executes arbitrary code on deserialization via `__reduce__`. The advisory's own remediation removed `pickle` in favour of JSON. This corroborates the central thesis of §6: the meaningful trust boundary is **Redis (message‑queue) write access**, not the web‑facing cookie. (Source: GitHub Security Advisory `miguelgrinberg/python-socketio`.)
- **`itsdangerous` `Signer` semantics.** The official ItsDangerous documentation (itsdangerous.palletsprojects.com) describes that a `Signer` signs a value and `unsign()`s it to verify it has not changed, raising `BadSignature` if the value (or signature) was altered, and that a recipient can read but not modify a signed value without the key. For the pinned **itsdangerous 1.1.0**, the default digest is **SHA‑1**; combined with `key_derivation="hmac"` (`app/session.py:L37-L41`), the `slapp` signature is effectively **HMAC‑SHA1**. This corroborates §6.1–§6.2.

### 8.3 Summary of answers

| # | Question | Answer | Anchor |
|---|----------|--------|--------|
| Q1 | Normal session lifecycle | Authenticated dict (`_user_id`, `_fresh`, `_id`, `sudo_time`, `csrf_token`, …) pickled into `session:<uuid>` with a **7‑day** TTL; anonymous/2FA‑pending sessions get **300 s**; logout destroys the authenticated session (deletes + rotates the key, sends delete-cookie headers), but `save_session` then re-issues a new `slapp` cookie + anonymous key, so the client keeps a rotated **anonymous** session | `app/session.py:L82-L114`, `L117-L121` |
| Q2 | Corrupted/malformed data | **(b) silently reset the session** — every deserialization exception is swallowed and a fresh anonymous session is returned | `app/session.py:L74-L80` |
| Q3 | HTTP/log observability | **No HTTP 500, no propagated exception, no log line**; only a new `slapp` `Set-Cookie` on the next response | `app/session.py:L78-L79` |
| Q4 | Harmless‑reset vs. RCE boundary | The boundary is the **validity of the pickle stream**: garbage ⇒ exception ⇒ reset; a valid `__reduce__` pickle ⇒ **arbitrary code executes at `pickle.loads`** (no exception) | `app/session.py:L76` |
| Q5 | Byte tampering vs. forging the signed ID | The signature (HMAC‑SHA1) gates **which key is selected**, not byte integrity; the real exposure is **Redis write access** — an attacker holding a valid cookie can poison their **own** key. `session_protection="strong"` is orthogonal | `app/session.py:L47-L59`, `L76`; `app/extensions.py:L8` |

**Central thesis (restated):** for this server‑side `pickle`/Redis session store, the meaningful trust boundary for the deserialization risk is **Redis write access, not the cookie signature**.

