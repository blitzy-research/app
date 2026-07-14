# How SimpleLogin's `RedisSessionStore` Deserializes Session Data — and What Happens When the Redis Payload Is Corrupted, Malformed, or Maliciously Tampered With

> **Deliverable:** runtime-grounded investigative answer for the source branch `app_2cd6ee777f8c`.
> **Subject:** the custom Flask `SessionInterface` named `RedisSessionStore(SessionInterface)` in `app/session.py` — the *only* file in the repository that calls `pickle.loads`/`pickle.dumps`.
> **Method:** every behavioral claim below sits next to the **actual, unedited output** of a temporary observation harness that exercised the **real** login/logout/request entry points **inside the supplied canonical Docker container** (`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`, whose `/app` tree is checked out at exactly the source commit `2cd6ee777f8c…`). Every factual claim carries a `file:line` citation and names the exact function/method that performs the work. A native host install reproduced the same behavior as a confirming cross-check (§1.2).

---

## TL;DR (one-paragraph answer)

SimpleLogin's server-side session subsystem is a custom Flask `SessionInterface` — `RedisSessionStore` in `app/session.py` — that stores each session's data as a **standard-library `pickle`** blob in Redis under the key `session:<sid>` and identifies it with an `itsdangerous` HMAC-**SHA1**-signed session id carried in the `slapp` cookie. **Deserialization is a single call, `pickle.loads(val)`, in `open_session` at `app/session.py:76`** — and, critically, **both that call *and* the `ServerSession(data, session_id=session_id)` that consumes its result sit inside the *same* `try` block (`app/session.py:75-77`).** When the Redis payload is **non-malicious garbage** (invalid opcodes, truncated bytes, or an empty `b''`), `pickle.loads` raises, a bare `except Exception: pass` at `app/session.py:78-79` swallows it, and `open_session` falls through to `return ServerSession(session_id=str(uuid.uuid4()))` at `app/session.py:80` — a **silent reset** to a brand-new anonymous session id. The request does **not** fail and **no** error is surfaced (a normal `302` redirect to `/auth/login` for a `@login_required` page, or a `200` login page for `GET auth.login`), and **no** session/pickle-specific line is written to the logs (the module imports no logger, `app/session.py:1-16`), so the reset is **indistinguishable in the logs from a user who was never logged in**. **The boundary between a harmless reset and a genuine risk is precise, and it is governed by *what `pickle.loads` does with the attacker's bytes* — not by a different code path.** If the bytes form a *valid, weaponized* pickle whose `__reduce__` returns a callable, that callable **executes *inside* `pickle.loads` at `app/session.py:76` — *before* the `except` at `:78` can intervene** — so arbitrary code runs regardless of what happens next. What happens *next* is decided by the value `loads` returns: a weaponized `os.system(...)` pickle returns its **exit code `0`** (a *falsey* int), so `loads` does **not** raise; control reaches `ServerSession(0, session_id=session_id)` at `app/session.py:77` **with the SAME sid**, yielding an empty anonymous session that is **NOT** a `uuid4` reset (observed: probe `200`, sid unchanged, TTL `300`). A weaponized pickle that instead returns a **valid session dict** even **preserves authentication** (observed: probe `302`, sid unchanged, TTL `604800`). Thus the malicious case does **not**, in general, share the benign case's `uuid4`-reset epilogue — the code has already executed, and the resulting session is whatever the *returned object* dictates. Finally, the HMAC signature protects **only the session-id pointer** (which Redis key `open_session` reads; `app/session.py:37-41`), not the integrity of the unsigned pickle payload written by `pickle.dumps(dict(session))` at `app/session.py:91`. Therefore an attacker who can **write to Redis but cannot forge the signed cookie** still reaches `pickle.loads` with attacker-controlled bytes — the victim's own valid, unforged cookie merely points `open_session` at the poisoned key — so being unable to forge the id does **not** meaningfully reduce the risk. This is **CWE-502 (Deserialization of Untrusted Data) / OWASP A08:2021 (Software and Data Integrity Failures)**, closely analogous to the python-socketio advisory **GHSA-g8c6-8fjj-2r4m**; the RCE is contingent on Redis write access and is not reachable by a normal web client who can only present a signed cookie.

---

## 1. Investigation method & canonical environment

### 1.1 Read-only discipline

This is a **documentation / knowledge-extraction** task, not a fix. No existing source file was modified; the only artifact added to the repository is this Markdown document. All observation code was placed **outside** the repository (copied into the container under `/tmp/obs_harness.py`, and run from `/tmp` on the native cross-check host); it was removed afterward, and the authoritative container was **destroyed** at completion (Appendix B). The repository working tree is clean except for this file (the actual `git status` / `git diff` output is shown in Appendix B). The deserialization risk described here is **explained, not remediated** — remediation is explicitly out of scope.

### 1.2 Canonical runtime and evidence provenance

**Provenance statement (read this first).** Every evidence block in §2–§7 is the **actual, unedited output of a single observation-harness run performed *inside the supplied canonical Docker container***. The environment setup instruction names the canonical build/run image two ways: a Docker Hub tag `andrewparkscaleai/coding-agent:simple-login__app__2cd6ee777f8c2d3531559588bcfb18627ffb5d2c` **and** the registry image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`. The Docker Hub tag is **not pullable** from this environment (access denied — captured verbatim below), so the **accessible** canonical image — the GHCR one named in the same instruction — was used. Its `/app` working tree is checked out at **exactly** the source commit under investigation, `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`, so the code observed is byte-for-byte the code being documented.

- **Authoritative runtime (container):** Python **3.10.18**, Redis **7.0.15**, PostgreSQL **15.13** — all *inside* the GHCR container. This satisfies the canonical constraint declared in `Dockerfile:8` (`FROM python:3.10`), `pyproject.toml:61` (`python = "^3.10"`), and CI `.github/workflows/main.yml:17,40` (`python-version '3.10'`).
- **Image identity (captured):** GHCR `Id = sha256:ea242796bbce36ca99ba9f783a4e7ac9d2ed3738e737f9a6d96bb22bbf1d9b58`; `RepoDigest = ghcr.io/scaleapi/swe-atlas@sha256:b82cb15631e92ade58b8cf10493550f03a54dc5a8f25d3f31ef41afc186ee2c1`; `Created = 2026-02-05T19:08:59Z`; `Entrypoint = /bin/bash`; `WorkingDir = /app`.
- **Source-branch commit (the code being investigated):** the container's `/app` HEAD is the fixed commit `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c` (`chore: emit some missing contact audit logs (#2269)`). This is the same commit — an ancestor of the destination-branch HEAD that carries this document.
- **Destination-branch commit (the commit that carries this document):** the current tip of branch `blitzy-69b57f9c-a60d-44e2-934e-aba188d82f39`. It is deliberately **not** pinned to a fixed hash here: a committed document cannot embed its own commit hash, and each remediation of this file produces a new commit, so the destination commit is identified by branch position rather than by a value that would immediately go stale. Relative to the source commit, the **only** repository change is the addition of this one document (`git diff --name-status 2cd6ee777f8c… HEAD` → `A blitzy/documentation/app_2cd6ee777f8c.md`; full output in Appendix B).
- **Interpreter / dependency pins** (captured from the container venv; exact pins from `poetry.lock`): flask **1.1.2** (`poetry.lock:910-911`), itsdangerous **1.1.0** (`poetry.lock:1617-1618`), redis **4.6.0** (`poetry.lock:2674-2675`), werkzeug **1.0.1** (`poetry.lock:3423-3424`), flask-login **0.5.0** (`poetry.lock:1029-1030`), limits **1.5.1** (`poetry.lock:1759-1760`), flask-limiter **1.4** (`poetry.lock:1013-1014`). No dependency was added, updated, or removed.
- **Backing services (container, localhost):** PostgreSQL **15.13** on port **15432** (role `test`/`test`, database `test`, 77 tables migrated to the Alembic head) and Redis **7.0.15** (`redis://localhost`).
- **Configuration:** `/app/tests/test.env`, which sets `FLASK_SECRET=secret` (`tests/test.env:20`), `DB_URI=postgresql://test:test@localhost:15432/test` (`tests/test.env:17`), and, critically, `MEM_STORE_URI=redis://localhost` (`tests/test.env:78`). Because `MEM_STORE_URI` is set, `RedisSessionStore` is installed — see the activation guard `if MEM_STORE_URI:` at `server.py:163-165`, which calls `initialize_redis_services(app, MEM_STORE_URI)` at `server.py:165` (`MEM_STORE_URI` defined at `app/config.py:568`).
- **Confirming cross-check (native host install).** The identical harness was also run in the platform-provisioned native host install of this repository (Python **3.10.20**, Redis **8.0.2**, PostgreSQL **17.10**). It reproduced the **same** structure and the **same** F1 return-object behavior (§6) as the container; it is reported here only as a **cross-check**, never as the authoritative source. The server-version differences (Redis 7 vs 8, PG 15 vs 17) do not affect the `get`/`setex`/`delete` of an opaque byte string or the pickle behavior; the pinned **client** `redis==4.6.0` governs the interaction in both environments, and every byte-sensitive result below is the exact bytes the **container** emitted.

The exact provenance commands and their **actual, unedited output** (captured to `/tmp/evidence/provenance.txt`) are:

```text
# [P1] canonical image named in setup instructions — Docker Hub tag is NOT pullable
$ docker pull andrewparkscaleai/coding-agent:simple-login__app__2cd6ee777f8c2d3531559588bcfb18627ffb5d2c
Error response from daemon: pull access denied for andrewparkscaleai/coding-agent, repository does not exist or may require 'docker login': denied: requested access to the resource is denied
EXIT=1

# [P2] the accessible canonical image (GHCR, named in the same instruction) — identity
$ docker image inspect ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0 --format '{{.Id}}'
sha256:ea242796bbce36ca99ba9f783a4e7ac9d2ed3738e737f9a6d96bb22bbf1d9b58
$ docker image inspect ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0 --format '{{index .RepoDigests 0}}'
ghcr.io/scaleapi/swe-atlas@sha256:b82cb15631e92ade58b8cf10493550f03a54dc5a8f25d3f31ef41afc186ee2c1
Created=2026-02-05T19:08:59.563478718Z ; Entrypoint=/bin/bash ; WorkingDir=/app

# [P3] the container's /app is the SOURCE COMMIT under investigation
$ docker exec <CID> git -C /app rev-parse HEAD
2cd6ee777f8c2d3531559588bcfb18627ffb5d2c
$ docker exec <CID> git -C /app log -1 --oneline
2cd6ee777f8c2d3531559588bcfb18627ffb5d2c chore: emit some missing contact audit logs (#2269)

# [P4] container runtime versions
Python 3.10.18
redis-server: Redis server v=7.0.15 sha=00000000:0 malloc=jemalloc-5.3.0 bits=64 build=3f20e06e76a2b578
psql: psql (PostgreSQL) 15.13 (Debian 15.13-0+deb12u1)
pg cluster: 15  main  15432 online postgres /var/lib/postgresql/15/main

# [P5] container dependency pins (exact, from venv)
flask=1.1.2 itsdangerous=1.1.0 werkzeug=1.0.1 redis=4.6.0 flask_login=0.5.0 limits=1.5.1 flask_limiter=1.4

# [P6] container services health
pg_public_tables=77
redis_ping=PONG
```

### 1.3 Exact build / run / invocation commands

These are the **exact, literal** commands used. `<CID>` is the container id of the running GHCR container (`d1e59585f7c0…`).

```bash
# --- start the canonical GHCR container (no host bind-mount; state is container-internal) ---
docker run -d --name slinv_probe \
  ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0 sleep infinity
CID=$(docker inspect -f '{{.Id}}' slinv_probe)

# --- bring up the backing services INSIDE the container ---
docker exec "$CID" bash -lc 'pg_ctlcluster 15 main start'          # PostgreSQL 15 on localhost:15432
docker exec "$CID" bash -lc 'redis-server --daemonize yes'         # Redis on 127.0.0.1:6379
docker exec "$CID" bash -lc 'cd /app && CONFIG=tests/test.env alembic upgrade head'  # 77 tables (once)

# --- copy the observation harness into the container (OUTSIDE the /app repo tree) ---
docker cp /tmp/obs_harness.py "$CID":/tmp/obs_harness.py

# --- run the harness against the REAL entry points, RedisSessionStore active ---
docker exec "$CID" bash -lc \
  'cd /app && PYTHONPATH=/app CONFIG=tests/test.env GITHUB_ACTIONS_TEST=true \
   /app/venv/bin/python /tmp/obs_harness.py'      # -> the verbatim output transcribed in §2-§7

# --- confirming cross-check on the native host install (identical harness) ---
export PATH="$HOME/.local/bin:$PATH"
cd /tmp/blitzy/app/blitzy-69b57f9c-a60d-44e2-934e-aba188d82f39_8eee3e
source venv/bin/activate
export CONFIG="$(pwd)/tests/test.env" GITHUB_ACTIONS_TEST=true PYTHONPATH="$(pwd)"
OBS_SKIP_DASHBOARD=1 python /tmp/obs_harness.py   # native cross-check (§6 behavior identical)

# --- cleanup: destroy the disposable container (removes ALL DB rows + Redis keys); see Appendix B ---
docker rm -f slinv_probe
rm -f /tmp/obs_harness.py /tmp/blitzy_obs_rce_mal1.txt /tmp/blitzy_obs_rce_mal2.txt
```

**App-boot verification.** With `CONFIG=tests/test.env`, `create_app()` yields `app.session_interface = app.session.RedisSessionStore`, `SESSION_COOKIE_NAME = slapp`, `app.secret_key = 'secret'`, and `MEM_STORE_URI = 'redis://localhost'` — i.e. the pickle deserialization path is **active** through the real entry points. (This exact activation state is shown as observed output in §2.1 below.)

### 1.4 Canonical entry points exercised

All behavior was exercised through the **real** entry points — never a bypass or synthetic stand-in:

- **login** — `client.post(url_for("auth.login"), data={"email": ..., "password": "password"})` → the `login()` view at `app/auth/views/login.py:25`, which on success calls `after_login()` at `app/auth/views/login_utils.py:12` (`login_user()` at `:36`, `session["sudo_time"] = int(time())` at `:37`).
- **auth-state probe (non-committing)** — `client.get(url_for("auth.login"))` → the `login()` view again; because it **redirects an already-authenticated user** (`app/auth/views/login.py:28-34`), the *status alone* reveals session state without mutating the database: **`302`** (Location `/dashboard/`) means the restored session is **authenticated**, **`200`** (the login form renders) means it is **anonymous**. This `GET` probe is used throughout §3–§7 precisely because it commits nothing.
- **authenticated request (login-required redirect chain)** — `client.get("/dashboard/")` → routed through `open_session` on the way in; used in §2.2 (authenticated `200`) and §3 (the corrupted-payload `302 → /auth/login?next=…` chain).
- **logout** — `client.get(url_for("auth.logout"))` → the `logout()` view at `app/auth/views/logout.py:9`, which calls `logout_session()` at `:10` and deletes the `slapp`/`mfa`/`dark-mode` cookies at `:13-15`.
- **API logout** — `GET /api/logout` (`app/api/views/user_info.py:131`) calls the **identical** `logout_session()` at `:140`; this path is labeled **verified-by-code-reference (same mechanism as web logout)**, not runtime-exercised (see §7.5).

The harness mirrors the canonical `tests/conftest.py` harness exactly: `os.environ["CONFIG"]=tests/test.env`, `app = create_app()`, `app.config["TESTING"]=True`, `WTF_CSRF_ENABLED=False`, `SERVER_NAME="sl.test"`, a `create_new_user()` (password `"password"`, `tests/utils.py:17`), all wrapped in `connection.begin()` and rolled back at teardown. Redis handles are read from the same `redis://localhost` the app uses. This is the exact template used by `tests/auth/test_login.py` and `tests/utils.py::login()` (`:46-59`).

### 1.5 A note on logging capture technique

SimpleLogin's `SL` logger writes to the process's **original stdout file descriptor**, established when logging is initialized at import time. Ordinary Python-level stdout redirection (reassigning `sys.stdout`) does **not** intercept it. To observe the logs honestly, §4 uses **OS file-descriptor-level capture**: the `capture_fd` helper `os.dup2()`-redirects **fd 1 (stdout) *and* fd 2 (stderr)** — the descriptors the `SL` logger writes to — to a temp file around a single request, then restores them (the helper is shown in Appendix A). This guarantees the captured log lines are exactly what the server emitted.

### 1.6 Byte-sensitivity and reproducibility

Byte-sensitive results were **captured, not assumed**: the pickle protocol was read from the emitted bytes (`pickle.DEFAULT_PROTOCOL = 4` on this Python 3.10.18 runtime; every emitted payload begins with `\x80\x04`, the PROTO opcode for protocol 4), and the `itsdangerous` cookie signature was verified by reproducing it with `signer.sign(...)`. All evidence below is the actual, unedited output from the container runtime. Per-run values (session ids, signatures, timestamps, user emails, `sudo_time`) differ from run to run, but the structure and behavior are **stable** — verified by running the harness end-to-end multiple times in the container **and** once on the native cross-check host, and confirming every structural invariant matched. In particular the flask-login "strong" session fingerprint `_id` (`002d6488…0c0916`) was **byte-identical** across the container run and the native cross-check (it is a deterministic function of the fixed request environment), while session ids and signatures varied as expected.

---

## 2. Baseline session lifecycle (login → logout)

This section documents what a "normal" session looks like at runtime: how it is created on login, what is written to Redis, what the signed cookie carries, and how logout tears it down. All output is from the harness run **inside the container** (§1.2/§1.3).

### 2.1 Environment / activation (observed — evidence block 4.0)

The harness dumped the following at start-up, confirming the pickle path is active and capturing the serializer/signer configuration:

```text
===== BLOCK 4.0 ENVIRONMENT =====
python: 3.10.18 (main, Jun 10 2025, 23:52:59) [GCC 12.2.0]
flask=1.1.2 itsdangerous=1.1.0 werkzeug=1.0.1 redis=4.6.0 flask_login=0.5.0 limits=1.5.1
MEM_STORE_URI='redis://localhost'
session_interface=RedisSessionStore
SESSION_COOKIE_NAME='slapp'
app.secret_key='secret'
permanent_session_lifetime(default,outside request)=2678400 s
redis client type = redis.client.Redis
pickle.DEFAULT_PROTOCOL = 4 ; HIGHEST_PROTOCOL = 5
signer = itsdangerous.signer.Signer
signer.digest_method = <built-in function openssl_sha1>
signer.sep = b'.'
test user id=4 email=user_g5nw4e6xkk@mailbox.test
```

**Interpretation (with citations):**

- `app.session_interface` is `app.session.RedisSessionStore`, installed by `initialize_redis_services` (`app/redis_services.py:9`) — the **sole wiring function** that assigns `app.session_interface` anywhere in the repository. That function has two branches and **both** assign the store: the normal `redis://` branch constructs a `RedisStorage` (`app/redis_services.py:11`) and assigns `app.session_interface = RedisSessionStore(...)` at `:12`, while the Redis-Sentinel branch constructs a `RedisSentinelStorage` (`:16`) and assigns the store at `:17-19`. The canonical `MEM_STORE_URI=redis://localhost` selects the normal branch (`:12`). `initialize_redis_services` runs only because `MEM_STORE_URI` is set — the activation guard `if MEM_STORE_URI:` at `server.py:163-165` calls it (`MEM_STORE_URI` defined at `app/config.py:568`).
- `SESSION_COOKIE_NAME = slapp` comes from `SESSION_COOKIE_NAME = "slapp"` at `app/config.py:199`.
- `app.secret_key = 'secret'` comes from `app.secret_key = FLASK_SECRET` at `server.py:151`, sourced from `FLASK_SECRET = os.environ["FLASK_SECRET"]` (`app/config.py:196-198`, with a mandatory non-empty guard) and supplied by `tests/test.env:20`.
- The serializer is stdlib `pickle` at **protocol 4** — on Python 3 the `try: import cPickle as pickle / except ImportError: import pickle` block at `app/session.py:9-12` falls back to the standard-library `pickle`. `pickle.DEFAULT_PROTOCOL = 4` was **captured** here, not assumed.
- The signer is `itsdangerous.signer.Signer` with `digest_method = openssl_sha1` (HMAC-**SHA1**) and separator `b'.'` — constructed by `_get_signer` as `itsdangerous.Signer(app.secret_key, salt="session", key_derivation="hmac")` at `app/session.py:37-41`.

**Important TTL nuance (observed + explained).** The line `permanent_session_lifetime = 2678400 s` (31 days) is read **at rest** (module import, outside any request), where it equals Flask's default. During any real request, the `@app.before_request def make_session_permanent()` at `server.py:204-207` sets `app.permanent_session_lifetime = timedelta(days=7)`, and `save_session` reads *that* per-request value at `app/session.py:92`. Hence the authenticated Redis TTL observed at login (§2.2) is **604800 s (7 days)**, while an anonymous session's TTL is **300 s** — the `if "_user_id" not in session: ttl = 300` branch at `app/session.py:95-96`. Both TTLs are **observed**, not merely inferred from the code: 604800 s on the authenticated key in §2.2, and 300 s on the anonymous key that is re-saved during logout in §2.3. The at-rest value (31 days) and the per-request effects (7 days / 300 s) are all reported from captured `setex` output.

### 2.2 Login — `POST auth.login` (observed — evidence block 4.1)

```text
===== BLOCK 4.1 BASELINE LOGIN (POST auth.login) =====
POST /auth/login -> status=302 location=http://sl.test/dashboard/
session id (sid) = 5aa9aae1-82b4-4628-81bb-3960d3ed2e93
redis key = session:5aa9aae1-82b4-4628-81bb-3960d3ed2e93
redis TTL seconds = 604800  (authenticated: make_session_permanent sets 7d)
pickled payload len = 244 bytes
pickled payload hex = 800495e9000000000000007d94288c0a5f7065726d616e656e7494888c065f667265736894888c085f757365725f6964948c2464386433636134392d336535392d343838622d396166352d633539613235366432326635948c035f6964948c803030326436343838373235663132373935363435626461626131336136336139623939323162643061613136646537393063366439393461323138353766343634333233393565646132666561623166343064303434366134633639343063376431663535623438643730666535626136393162316161386335306330393136948c097375646f5f74696d65944a698c556a752e
pickle.loads(payload) = {'_permanent': True, '_fresh': True, '_user_id': 'd8d3ca49-3e59-488b-9af5-c59a256d22f5', '_id': '002d6488725f12795645bdaba13a63a9b9921bd0aa16de790c6d994a21857f46432395eda2feab1f40d0446a4c6940c7d1f55b48d70fe5ba691b1aa8c50c0916', 'sudo_time': 1783991401}
--- pickletools.dis(payload) ---
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
   49: \x8c     SHORT_BINUNICODE 'd8d3ca49-3e59-488b-9af5-c59a256d22f5'
   87: \x94     MEMOIZE    (as 4)
   88: \x8c     SHORT_BINUNICODE '_id'
   93: \x94     MEMOIZE    (as 5)
   94: \x8c     SHORT_BINUNICODE '002d6488725f12795645bdaba13a63a9b9921bd0aa16de790c6d994a21857f46432395eda2feab1f40d0446a4c6940c7d1f55b48d70fe5ba691b1aa8c50c0916'
  224: \x94     MEMOIZE    (as 6)
  225: \x8c     SHORT_BINUNICODE 'sudo_time'
  236: \x94     MEMOIZE    (as 7)
  237: J        BININT     1783991401
  242: u        SETITEMS   (MARK at 13)
  243: .    STOP
highest protocol among opcodes = 4
Set-Cookie slapp (from response) = '5aa9aae1-82b4-4628-81bb-3960d3ed2e93.ctGWJZ9YqBIeikgzXmHKH5XKyq8'
signer.sign(sid)               = '5aa9aae1-82b4-4628-81bb-3960d3ed2e93.ctGWJZ9YqBIeikgzXmHKH5XKyq8'
cookie == signer.sign(sid) ? True
signer.unsign(cookie).decode() = '5aa9aae1-82b4-4628-81bb-3960d3ed2e93'
GET /dashboard/ (authenticated) -> status=200
```

**Interpretation (with citations):**

- **The cookie is `<sid>.<signature>`.** The signature is an `itsdangerous` HMAC-SHA1 MAC over the **session id string only**, keyed by `app.secret_key` with `salt="session"` (`app/session.py:37-41`). Two independent checks prove the signature covers the sid exclusively: reconstructing `signer.sign(sid)` reproduces the emitted cookie **byte-for-byte** (`cookie == signer.sign(sid) ? True`), and `signer.unsign(cookie)` recovers exactly the sid. This signing happens in `save_session` — `signed_session_id = self._get_signer(app).sign(itsdangerous.want_bytes(session.session_id))` and `response.set_cookie(...)` at `app/session.py:102-114`.
- **The Redis value is a raw pickle blob.** The key is `session:<sid>` built by `_get_key` (`f"{SESSION_PREFIX}:{session_Id}"`, `SESSION_PREFIX = "session"` at `app/session.py:18`, `app/session.py:43-45`). The value is `pickle.dumps(dict(session))` written by `save_session` at `app/session.py:91` via `self._redis_w.setex(...)` (`app/session.py:97-101`). The captured bytes begin with `\x80\x04` — the PROTO opcode for **protocol 4** — confirming the serializer and protocol from the emitted bytes (verified again by `pickletools.dis`, `highest protocol among opcodes = 4`).
- **The payload dict** is `{_permanent, _fresh, _user_id, _id, sudo_time}`. There is **no `csrf_token`** because the canonical harness sets `WTF_CSRF_ENABLED=False`. `_user_id` is the flask-login user id (`user.get_id()`); `_id` is the flask-login **"strong" session-protection** fingerprint, enabled by `login_manager.session_protection = "strong"` at `app/extensions.py:8`; `sudo_time` is written by `after_login()` at `app/auth/views/login_utils.py:37`.
- **The TTL is 604800 s (7 days)** because `"_user_id"` is present in the session, so the `ttl = 300` branch is *not* taken — `ttl = int(app.permanent_session_lifetime.total_seconds())` with the per-request 7-day lifetime (`app/session.py:92`, `:95-96`; the 7-day value set by `make_session_permanent`, `server.py:204-207`).

### 2.3 Logout — `GET auth.logout` (observed — evidence block 4.2)

```text
===== BLOCK 4.2 LOGOUT TEARDOWN (GET auth.logout) =====
before logout: redis key exists=True ttl=604800
GET /auth/logout -> status=302 location=http://sl.test/auth/login
Set-Cookie headers on logout response:
   slapp=; Expires=Thu, 01-Jan-1970 00:00:00 GMT; Max-Age=0; Path=/
   mfa=; Expires=Thu, 01-Jan-1970 00:00:00 GMT; Max-Age=0; Path=/
   dark-mode=; Expires=Thu, 01-Jan-1970 00:00:00 GMT; Max-Age=0; Path=/
   slapp=87adf8a1-2472-45bf-85d7-75cc03987979.BPnY34NTr0pfQTa6exMzprAOdlw; Domain=.sl.test; Expires=Tue, 21-Jul-2026 01:10:02 GMT; HttpOnly; Path=/; SameSite=Lax
after logout: OLD authenticated redis key session:5aa9aae1-82b4-4628-81bb-3960d3ed2e93 exists=False
post-logout NEW anonymous key = session:87adf8a1-2472-45bf-85d7-75cc03987979
post-logout NEW payload hex = 80049553000000000000007d94288c0a5f7065726d616e656e7494888c097375646f5f74696d65944a698c556a8c085f666c6173686573945d948c0773756363657373948c12596f7520617265206c6f67676564206f757494869461752e
post-logout NEW payload pickle.loads = {'_permanent': True, 'sudo_time': 1783991401, '_flashes': [('success', 'You are logged out')]}
post-logout NEW payload keys (sorted) = ['_flashes', '_permanent', 'sudo_time']
post-logout NEW payload TTL = 300 s (anonymous: '_user_id' absent -> 300)
```

**Interpretation (with citations):**

- `logout()` (`app/auth/views/logout.py:9-16`) calls `logout_session()` (`app/session.py:117-121`), which calls `logout_user()` then `purge_session()` (`app/session.py:61-66`). `purge_session` **DELETEs** the authenticated Redis key — `self._redis_w.delete(self._get_key(session.session_id))` at `app/session.py:63` — and assigns a fresh `uuid.uuid4()` at `:64`. This is confirmed by `after logout: OLD authenticated redis key session:5aa9aae1… exists=False`.
- The view then deletes the `slapp`, `mfa`, and `dark-mode` cookies (`app/auth/views/logout.py:13-15`) → **Set-Cookie headers 1–3** (empty value, `Max-Age=0`, epoch expiry).
- **Observed nuance beyond the naïve "just delete it" model (this is *not* a brand-new empty session).** `purge_session` does not create a new session object — it **mutates the current one in place**: it DELETEs the old Redis key (`app/session.py:63`) and reassigns `session.session_id = str(uuid.uuid4())` (`app/session.py:64`). `logout_user()` (flask-login 0.5.0) pops the authentication keys `_user_id`, `_fresh`, and `_id`, leaving the non-authentication keys in place. Because `make_session_permanent` set `session.permanent = True` earlier in the request (`server.py:206`), the `_permanent` key remains, and the pre-existing `sudo_time` (written at login by `after_login()`, `app/auth/views/login_utils.py:37`) is **not** cleared by logout. Then, before the response is built, `flash("You are logged out", "success")` at `app/auth/views/logout.py:11` **adds a `_flashes` entry** to that same session. Finally, `save_session` runs on the way out and persists this now-anonymous — **but non-empty** — dict under the new sid → **Set-Cookie header 4** (a new signed `slapp` pointing at the new key).
- **The post-logout payload was decoded to prove its contents** (it is *not* empty): `pickle.loads` of the new key yields `{'_permanent': True, 'sudo_time': 1783991401, '_flashes': [('success', 'You are logged out')]}`. Crucially, the authentication keys `_user_id`/`_fresh`/`_id` are **gone**, so flask-login treats the client as anonymous, but `sudo_time` and the logout flash **survive**. The new key is stored with **TTL 300 s** — the anonymous branch `if "_user_id" not in session: ttl = 300` at `app/session.py:95-96`, in contrast to the authenticated 604800 s at login.
- Being the *last* `Set-Cookie`, header 4 wins in the client jar. The net security effect is nonetheless correct: the **authenticated** payload is destroyed (old key exists=False), the surviving payload carries **no credential**, and the user is anonymous on the next request.

---


## 3. Corrupted / malformed payload behavior (the core question)

**Question (b):** on the *next request* after the Redis payload becomes invalid pickle, does the request **fail**, **silently reset** the session, or **surface an error** to the user?

**Answer: silent reset.** The request does not fail and no error is surfaced; the session is silently replaced with a fresh anonymous one. Through a `@login_required` page this manifests as a normal `302` redirect to the login page; through `GET auth.login` it manifests as a normal `200` login form.

### 3.1 Observed — authenticate, corrupt the Redis payload, issue the next real request (evidence blocks 4.3 + 4.3b)

The harness authenticates, overwrites the Redis key with malformed bytes, then re-issues a real request — first via the non-committing `GET auth.login` auth-state probe (block 4.3), then via a `@login_required` `GET /dashboard/` to show the full redirect chain (block 4.3b):

```text
===== BLOCK 4.3 CORRUPTED MALFORMED PICKLE =====
sid=35d4cb49-56c1-4202-bab8-6ccfe67395fc
before: exists=True ttl=604800
during (redis session:35d4cb49-56c1-4202-bab8-6ccfe67395fc value): len=16 hex=6e6f742d612d7069636b6c652d00ff99
auth_probe GET /auth/login -> status=200 (200=anon,302=auth)
response Set-Cookie sid = ce84e218-5330-4087-8a9e-84b378fb4224  (same as before? False)
after: session:35d4cb49-56c1-4202-bab8-6ccfe67395fc exists=True ttl=-1
interpretation: malformed bytes -> pickle.loads raises -> except -> NEW sid reset
pickle.loads(malformed) raised: UnpicklingError: invalid load key, 'n'.

===== BLOCK 4.3b CORRUPTED PAYLOAD THROUGH GET /dashboard/ (login-required redirect chain) =====
BEFORE corruption: authenticated sid=88e203fb-4ead-4d96-874e-4f933ef6fa21 ; GET /dashboard/ -> status=200 (200=authenticated)
AFTER (next real GET /dashboard/): status=302 ; Location=http://sl.test/auth/login?next=%2Fdashboard%2F%3F
sid CHANGED == SILENT RESET: 88e203fb-4ead-4d96-874e-4f933ef6fa21 -> 7e91fa92-b607-45cd-94a5-b7d31e225fd8
```

The malformed 16 bytes written to the Redis key were `b"not-a-pickle-\x00\xff\x99"` (hex `6e6f742d612d7069636b6c652d00ff99`). The before/during/after state was recorded explicitly:

- **Before:** a genuinely authenticated session (`GET /dashboard/ -> 200`), sid `88e203fb-…`.
- **During:** the stored value is 16 malformed bytes; calling `pickle.loads` on them raises `UnpicklingError: invalid load key, 'n'.` — proving the bytes are undecodable (`'n'` is the first byte, `0x6e`, of `"not-a-pickle-…"`).
- **After:** the next real request produces a silent reset — the `GET /dashboard/` returns `302` to `/auth/login?next=%2Fdashboard%2F%3F`, and the session id **changed** (`88e203fb-… → 7e91fa92-…`). (The `GET auth.login` probe form of the same condition returns `200`, likewise with a fresh sid `ce84e218-…`.) In **both** forms the corrupted key itself is left in Redis with **TTL `-1`** (no expiry) — an orphan the reset does not clean up, because `open_session` simply abandons it and points the new session at a brand-new key.

### 3.2 Line-by-line trace of `open_session` (`app/session.py:68-80`)

The crux is the `open_session` method. On the corrupted-payload request it executes as follows:

1. `session_id = self.extract_and_validate_session_id(app, request)` (`app/session.py:69`) — the `slapp` cookie is still validly signed (only the *Redis payload* was corrupted, not the cookie), so this returns the real sid.
2. `if not session_id:` (`:70`) is **false**, so the early fresh-session return at `:71` is skipped.
3. `val = self._redis_r.get(self._get_key(session_id))` (`:73`) — reads the corrupted bytes.
4. `if val is not None:` (`:74`) is **true** (the corrupted value exists).
5. `data = pickle.loads(val)` (`:76`) — **raises** `UnpicklingError` (observed above).
6. `except Exception:` (`:78`) catches it; `pass` (`:79`) swallows it silently — no re-raise, no logging.
7. Control falls through to `return ServerSession(session_id=str(uuid.uuid4()))` (`:80`) — a brand-new empty session with a new `uuid4` id.

Because the restored session is empty (no `_user_id`), flask-login treats the user as anonymous. The `302` observed through `GET /dashboard/` is then produced by a **two-step chain**, not directly by `@login_required`: the dashboard route (`/dashboard/`, `@dashboard_bp.route("/")` at `app/dashboard/views/index.py:55`) carries `@login_required` (`:56`); when it fails, flask-login's `unauthorized()` handler runs, but because **no `login_view` is configured** on the `LoginManager` (`app/extensions.py:7-8` sets only `session_protection = "strong"`), flask-login cannot redirect to a login page itself and instead calls `abort(401)`. That `401` is caught by the app-wide `@app.errorhandler(401)` at `server.py:347-353`, which — for a non-`/api/` path — issues `flash(...)` + `redirect(url_for("auth.login", next=request.full_path))`, yielding the `302` to `/auth/login`. The observed `Location: http://sl.test/auth/login?next=%2Fdashboard%2F%3F` confirms this exact path: the `next=%2Fdashboard%2F%3F` query is `request.full_path` of `/dashboard/` (URL-encoded, including the trailing `?`), which only the `errorhandler(401)` branch sets. Separately, the bare, non-logging `except Exception: pass` at `app/session.py:78-79` is precisely why the deserialization failure itself is **silent**: the error never propagates to the request handler and never becomes a `500` — the `302` is an ordinary unauthenticated-access redirect, indistinguishable from a user who was never logged in.

---

## 4. Response & log observability

**Question (c):** what actually appears in the HTTP **response** and in the server **logs** when the reset occurs?

**Answer:** the HTTP response is a normal redirect/`200` (no error page, no `500`). The logs contain **exactly one** line for the request — the same generic per-request line that `after_request()` (`server.py:272-296`) prints for **every non-excluded** request (a short exclusion list at `server.py:276-281` skips only `/static`, `/admin/static`, `/_debug_toolbar`, `/git`, `/favicon.ico`, and `/health`) — with the only difference being the HTTP status code. There is **no** session/pickle/unpickle/reset/error-specific log line.

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

### 4.2 OS fd-level capture: one NORMAL vs one CORRUPTED→RESET request (evidence block 4.5)

Using file-descriptor-level capture (see §1.5), the harness recorded the log output of a single normal authenticated request and a single corrupted→reset request. Here the auth-state probe is `GET auth.login`, so a **normal (authenticated)** request logs the "already authenticated, redirect to dashboard" line and returns `302`, while a **reset (anonymous)** request simply renders the login form and returns `200`:

```text
===== BLOCK 4.5 OBSERVABILITY (fd-level capture: NORMAL vs RESET request) =====
captured during NORMAL request = b'2026-07-14 01:10:03,818 - SL - DEBUG - 349 - "/app/app/auth/views/login.py:33" - login() -  - user is already authenticated, redirect to dashboard\n2026-07-14 01:10:03,819 - SL - DEBUG - 349 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 302, takes 0.002336740493774414\n'
captured during RESET  request = b'2026-07-14 01:10:04,062 - SL - DEBUG - 349 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.0009436607360839844\n'
  RESET capture contains 'session'    ? False
  RESET capture contains 'pickle'     ? False
  RESET capture contains 'Unpickling' ? False
  RESET capture contains 'reset'      ? False
  RESET capture contains 'Exception'  ? False
  RESET capture contains 'Traceback'  ? False
=> the ONLY output during a reset is server.py:284's generic per-request access log (present for EVERY request); open_session's except branch (L78-79 'pass') emits nothing session/pickle/reset-specific.
```

**Interpretation (with citations):**

- The reset emits **exactly one** log line, and it is the **same** generic per-request line that `after_request()` (`server.py:284`, inside the `server.py:272-296` handler) prints for **every non-excluded** request (the exclusion list at `server.py:276-281`). The only difference from a normal request is the HTTP status: `200` (reset → anonymous → login form) versus `302` (authenticated → redirect to dashboard) for this `GET auth.login` probe. (Through a `@login_required` page the same reset instead yields `302 → /auth/login`, per §3.2 — but still just one generic `after_request()` line.)
- There is **no** session-, pickle-, unpickle-, reset-, or `Traceback`-specific log line — programmatically confirmed by all six token checks returning `False`. This follows directly from the swallow-without-logging at `app/session.py:78-79` and the absence of a logger in the module (`app/session.py:1-16`).
- **Consequence:** in the logs, a corrupted-payload reset is **indistinguishable from a user who simply was not logged in**. Both produce the identical `after_request()` line differing only by status. The HTTP response itself is a normal redirect/login page — there is no error surfaced to the user or to log-based monitoring.

---

## 5. Edge / error paths

Beyond the corrupted-payload case, four related conditions were exercised through the real path (evidence block 4.4). Each distinguishes which conditions ever reach the deserializer — and, crucially, distinguishes a **deleted/missing** key (`None`, guard short-circuits) from a **stored-empty** `b''` value (not `None`, so `pickle.loads` *is* called and raises `EOFError`). The probe is the non-committing `GET auth.login` (`200`=anonymous reset, `302`=authenticated):

```text
===== BLOCK 4.4a MISSING REDIS VALUE =====
sid=08c2ecc4-d697-4649-a8b8-518f1c8beda5
before: exists=True ttl=604800
during (redis session:08c2ecc4-d697-4649-a8b8-518f1c8beda5 value): (deleted key -> val is None)
auth_probe GET /auth/login -> status=200 (200=anon,302=auth)
response Set-Cookie sid = 301d7c7e-8df1-43db-a265-7d34644367d4  (same as before? False)
after: session:08c2ecc4-d697-4649-a8b8-518f1c8beda5 exists=False ttl=-2
interpretation: val is None -> pickle.loads NOT called -> NEW sid reset

===== BLOCK 4.4b EMPTY BYTES =====
sid=5c4ba70f-4768-4647-b49c-701bace8868c
before: exists=True ttl=604800
during (redis session:5c4ba70f-4768-4647-b49c-701bace8868c value): len=0 hex=
auth_probe GET /auth/login -> status=200 (200=anon,302=auth)
response Set-Cookie sid = c53544fe-4b38-47a1-b4cf-4a92f595a97a  (same as before? False)
after: session:5c4ba70f-4768-4647-b49c-701bace8868c exists=True ttl=-1
interpretation: val==b'' is not None -> loads(b'') raises EOFError -> NEW sid reset
pickle.loads(b'') raised: EOFError: Ran out of input

===== BLOCK 4.4c TRUNCATED PICKLE =====
sid=50a8f117-13b7-41ef-9930-238dad31de9e
before: exists=True ttl=604800
during (redis session:50a8f117-13b7-41ef-9930-238dad31de9e value): len=122 hex=800495e9000000000000007d94288c0a5f7065726d616e656e7494888c065f667265736894888c085f757365725f6964948c2464386433636134392d336535392d343838622d396166352d633539613235366432326635948c035f6964948c803030326436343838373235663132373935363435626461626131
auth_probe GET /auth/login -> status=200 (200=anon,302=auth)
response Set-Cookie sid = 31d1a1d4-ab36-4f26-8ce4-ae8f951dcdde  (same as before? False)
after: session:50a8f117-13b7-41ef-9930-238dad31de9e exists=True ttl=-1
interpretation: truncated valid pickle -> loads raises -> NEW sid reset
pickle.loads(truncated) raised: UnpicklingError: pickle data was truncated

===== BLOCK 4.4d BAD SIGNATURE COOKIE =====
good cookie    = 'f767de0e-3402-4a7e-919a-39839872a5ff.aaxy5EPjeTfJPx_D3kdMI94l6_E'
tampered cookie= 'X767de0e-3402-4a7e-919a-39839872a5ff.aaxy5EPjeTfJPx_D3kdMI94l6_E'  (first char of signed sid flipped)
signer.unsign(tampered) raised: BadSignature: Signature b'aaxy5EPjeTfJPx_D3kdMI94l6_E' does not match
auth_probe with tampered cookie -> status=200 (200=anon) newsid=0a88ad90-5a7c-4266-8e17-3d0179310ac9 (differs from f767de0e-3402-4a7e-919a-39839872a5ff? True)
interpretation: BadSignature -> extract_and_validate returns None -> fresh uuid4; the sid f767de0e-3402-4a7e-919a-39839872a5ff Redis key is never read
```

**Interpretation (with citations):**

- **(4.4a) Deleted / missing Redis value → `None`.** After deleting the key, `redis.get(key)` returns `None` — a *missing* key, which is **distinct** from a key that holds empty bytes (4.4b). Because the value is `None`, the `if val is not None:` guard at `app/session.py:74` short-circuits and `pickle.loads` is **never called**. `open_session` falls straight through to the fresh-session return at `app/session.py:80`. This is the benign "session expired / evicted" case: a reset with **no deserialization at all** (`after: exists=False ttl=-2`, Redis's "no such key" TTL sentinel).
- **(4.4b) Stored-empty `b''` value → `EOFError`.** Writing an **empty byte string** to the key is **not** equivalent to deleting it: `redis.get(key)` returns `b''`, and `b'' is not None` is `True`, so the `if val is not None:` guard at `app/session.py:74` **passes** and `pickle.loads(b'')` **is** called. It raises `EOFError: Ran out of input` (an empty stream carries no opcodes), caught by the same `except Exception: pass` at `app/session.py:78-79` → silent reset. The observable contrast with 4.4a is exact: a **deleted** key reaches the deserializer **not at all**, whereas a **stored-empty** key reaches it and fails with `EOFError` — a difference invisible in the HTTP response (both are a fresh anonymous session) but real at the `pickle.loads` call site.
- **(4.4c) Truncated pickle.** A valid 244-byte payload truncated to its first 122 bytes is still recognizable as a protocol-4 pickle header but is incomplete. `pickle.loads` raises `UnpicklingError: pickle data was truncated`, caught by the same bare `except Exception: pass` at `app/session.py:78-79` → silent reset.
- **(4.4d) `BadSignature` cookie.** Flipping the **first character of the signed sid** yields a cookie whose HMAC no longer matches. `extract_and_validate_session_id` (`app/session.py:47-59`) reads the cookie (`request.cookies.get(app.session_cookie_name)`, `:51`), calls `signer.unsign(...)` (`:56`) which raises `itsdangerous.BadSignature` (`Signature b'aaxy5EPjeTfJPx_D3kdMI94l6_E' does not match`), and the `except itsdangerous.BadSignature: return None` at `:58-59` converts that to `None`. With `session_id is None`, `open_session` returns a fresh session at `:70-71` **without ever reading Redis**, so the original key `session:f767de0e-…` is never touched. This is the **key contrast**: tampering the *signed id* is detected and rejected **before** any deserialization occurs.
- **Scope of what the bare `except` catches (precision).** The reset is triggered by the errors caught at `app/session.py:78`, which is `except Exception:` — **not** `except BaseException:`. Every deserialization failure observed here is an ordinary `Exception` subclass: `UnpicklingError` (invalid opcode in §3, truncation in 4.4c) and `EOFError` (empty input in 4.4b) both derive from `Exception`, so all are swallowed. A hypothetical failure *outside* the `Exception` hierarchy (e.g., `KeyboardInterrupt` or `SystemExit`, which derive from `BaseException`) would **not** be swallowed and would propagate. In practice, pickle-decoding failures on malformed input are all `Exception` subclasses, so every malformed / truncated / empty payload is funneled into the same silent reset — but the catch is scoped to `Exception`, not universal.

Taken together, (4.4a)/(4.4d) show the two ways `pickle.loads` is **avoided** (missing/deleted Redis value → `None`; rejected signature), while (4.4b)/(4.4c) and §3 show the ways it is **reached and *raises* harmlessly** — an empty (`b''`) stream, a truncation, or an invalid opcode, each raising an `Exception` subclass that the bare `except` swallows into an identical silent reset. What §6 adds is the fourth possibility the edge paths do not cover: bytes that reach `pickle.loads` and do **not** raise.

---


## 6. The boundary: harmless reset vs. genuine deserialization risk

**Question (d):** where exactly is the boundary between a benign session reset (non-malicious malformed bytes) and a genuine deserialization risk (a well-formed *malicious* pickle that executes code during `pickle.loads`)?

**Answer:** the boundary is **not** a different code path — it is the *same* `pickle.loads(val)` at `app/session.py:76`. But the session outcome after that call is **not** a single fixed "silent reset." It is governed by **what `pickle.loads` does with the bytes**, and there are three distinct outcomes, because **both `pickle.loads(val)` *and* the `ServerSession(data, session_id=session_id)` that consumes its result live inside the *same* `try` block at `app/session.py:75-77`:**

```python
        if val is not None:          # app/session.py:74
            try:                     # app/session.py:75
                data = pickle.loads(val)                    # :76  <-- deserialize
                return ServerSession(data, session_id=session_id)  # :77  <-- consume, SAME sid
            except Exception:        # :78
                pass                 # :79
        return ServerSession(session_id=str(uuid.uuid4()))  # :80  <-- fresh uuid4 reset
```

1. **`loads` raises** (non-malicious garbage — invalid opcode, truncated, empty `b''`): the `except` at `:78` catches it and control reaches `:80` → a **fresh `uuid4` reset** (a NEW sid). This is §3/§5.
2. **`loads` returns**, then `ServerSession(data, session_id)` at `:77` **succeeds**: the session is rebuilt under the **SAME sid** (no `uuid4` reset). Whether it is authenticated depends only on whether `data` contained `_user_id`.
3. **`loads` returns**, but `ServerSession(data, session_id)` at `:77` **raises** while ingesting `data`: that exception is *also* inside the `try`, so the `except` at `:78` catches it and control reaches `:80` → a **fresh `uuid4` reset** (a NEW sid).

A weaponized pickle whose `__reduce__` returns a callable **executes that callable *inside* `pickle.loads` at `:76` — *before* the `except` at `:78` can run** — so arbitrary code executes regardless of which of outcomes 2/3 follows. The malicious case therefore does **not** in general share the benign case's `uuid4`-reset epilogue: it lands in outcome 2 or 3 depending on the *return value* the exploit leaves on the stack.

> **Safety note.** The demonstrations below are deliberately benign: each malicious pickle's *only* effect is to write a marker file (`/tmp/blitzy_obs_rce_mal1.txt` / `_mal2.txt`). No destructive action is taken, and the markers are removed afterward. The RCE is **contingent on Redis write access** (the attacker must be able to overwrite `session:<sid>`); it is not reachable by a normal web client.

### 6.1 Why the epilogue depends on the *return value* — the `ServerSession`/`CallbackDict` mechanics

`ServerSession.__init__(self, initial=None, session_id=None)` (`app/session.py:22-28`) forwards `initial` to its base via `super(ServerSession, self).__init__(initial, on_update)` (`app/session.py:26`); the base is werkzeug's `CallbackDict`, whose `__init__` runs `dict.__init__(self, initial or ())`. That one expression, evaluated at `:77` **inside the `try`**, is what makes the epilogue depend on the *type and truthiness* of whatever `pickle.loads` returned:

- **Falsey `initial`** (`None`, `0`, `""`, `[]`, `{}`): `initial or ()` collapses to `()`, so `dict.__init__(self, ())` builds an **empty** dict — construction **succeeds**, the SAME sid is retained, and the session is anonymous (`_user_id` absent).
- **Truthy, dict-convertible `initial`** (a real mapping, or a sequence of pairs like `[('x', 1)]`): `dict.__init__` populates the dict — construction **succeeds**, SAME sid; authenticated iff `_user_id` is present.
- **Truthy, *non*-convertible `initial`** (`1`, `"abc"`): `dict.__init__(self, 1)` / `dict.__init__(self, "abc")` **raises** (`TypeError` / `ValueError`) — and because this raise happens at `:77` inside the `try`, the `except` at `:78` catches it → **fresh `uuid4` reset** at `:80`.

This is why (below) `pickle.loads` returning `0` yields a *same-sid* anonymous session, while returning `1` yields a *new-sid* reset — even though both are "just an int."

### 6.2 The return-object matrix (non-malicious pickles) — evidence block 4.6

The harness overwrote a freshly-authenticated session's Redis key with pickles that decode to eight different objects, then issued the real `GET auth.login` probe (`200`=anonymous, `302`=authenticated) and recorded the probe status, whether the sid stayed the **same** (no `uuid4` reset) or became **NEW**, and the TTL of the *original* key afterward:

```text
===== BLOCK 4.6 RETURN-OBJECT MATRIX (non-malicious pickles) =====
Each row: pickle.loads returns <obj> -> ServerSession(<obj>, SAME sid) inside try (L77).
Legend: probe 200=anonymous, 302=authenticated; sid 'same' means NO reset.
loads->None             probe_status=200  sid=same  after_ttl(session:4e3363d9-f307-4c2d-b7bc-0be5f1e353cd)=300
loads->[]               probe_status=200  sid=same  after_ttl(session:a70133fc-42a4-40f3-9f82-93c9978d2ea1)=300
loads->0                probe_status=200  sid=same  after_ttl(session:1f9cd5df-8d44-4d5a-907b-ba662b5d562d)=300
loads->'' (empty str)   probe_status=200  sid=same  after_ttl(session:27ea8661-268f-48cf-8f1a-3eef8d3a94d3)=300
loads->1                probe_status=200  sid=NEW   after_ttl(session:80260c11-efb8-4eae-b346-1429f6089872)=-1
loads->'abc'            probe_status=200  sid=NEW   after_ttl(session:f57e85a6-b8cd-47af-a219-357230d1dc62)=-1
loads->[('x',1)]        probe_status=200  sid=same  after_ttl(session:27d241ba-2be0-4a81-8255-2262848dd337)=300
loads->VALID_AUTH_DICT  probe_status=302  sid=same  after_ttl(session:3e8c9606-718c-48cf-a601-be36f48b9c1a)=604800
--- matrix summary (label | probe | sid | ttl-of-orig-key) ---
   None             | 200 | same | 300
   []               | 200 | same | 300
   0                | 200 | same | 300
   '' (empty str)   | 200 | same | 300
   1                | 200 | NEW  | -1
   'abc'            | 200 | NEW  | -1
   [('x',1)]        | 200 | same | 300
   VALID_AUTH_DICT  | 302 | same | 604800
```

**Interpretation (with citations):** this matches §6.1 exactly. The falsey returns (`None`, `[]`, `0`, `''`) and the dict-convertible returns (`[('x',1)]`, `VALID_AUTH_DICT`) all **succeed** at `:77` and keep the **SAME sid** — no `uuid4` reset. Only the truthy non-convertible returns (`1`, `'abc'`) **raise inside the `try`** and therefore hit the `:80` reset (**NEW sid**). Authentication tracks `_user_id`: only `VALID_AUTH_DICT` (which contains `_user_id`) yields `302`/TTL `604800`; every other same-sid case is anonymous with the `if "_user_id" not in session: ttl = 300` branch (`app/session.py:95-96`) — hence `after_ttl=300` on those keys, whereas the reset cases leave the original key untouched at `-1`. **This is the crucial correction to the naïve model: a non-raising payload does *not* trigger the `uuid4` reset at `:80`; it is consumed at `:77` under the same sid.**

### 6.3 Malicious pickle #1 — `os.system` RCE that returns `0` (SAME sid, NOT a reset) — evidence block 4.6 MAL-1

The payload is `pickle.dumps(_RCE1())` where `_RCE1.__reduce__` returns `(os.system, (CMD,))` with `CMD` writing a benign marker. `os.system` returns the shell **exit code `0`**, so `pickle.loads` **returns `0`** — a *falsey int* — and does **not** raise:

```text
===== BLOCK 4.6 MAL-1 MALICIOUS os.system MARKER (RCE, returns 0) =====
malicious pickle len=91 hex=80049550000000000000008c05706f736978948c0673797374656d9493948c356563686f205243455f4d414c315f4558454355544544203e202f746d702f626c69747a795f6f62735f7263655f6d616c312e74787494859452942e
--- pickletools.dis(malicious MAL-1) ---
    0: \x80 PROTO      4
    2: \x95 FRAME      80
   11: \x8c SHORT_BINUNICODE 'posix'
   18: \x94 MEMOIZE    (as 0)
   19: \x8c SHORT_BINUNICODE 'system'
   27: \x94 MEMOIZE    (as 1)
   28: \x93 STACK_GLOBAL
   29: \x94 MEMOIZE    (as 2)
   30: \x8c SHORT_BINUNICODE 'echo RCE_MAL1_EXECUTED > /tmp/blitzy_obs_rce_mal1.txt'
   85: \x94 MEMOIZE    (as 3)
   86: \x85 TUPLE1
   87: \x94 MEMOIZE    (as 4)
   88: R    REDUCE
   89: \x94 MEMOIZE    (as 5)
   90: .    STOP
highest protocol among opcodes = 4
marker BEFORE loads exists? False
pickle.loads(MAL-1) returned: 0 (this is os.system's exit code)
marker AFTER  loads exists? True
marker contents = 'RCE_MAL1_EXECUTED'
through open_session: probe_status=200 sid=same ttl(session:0808cfc1-9466-4c2d-b211-101349749490)=300
marker file after real request exists? True
CONCLUSION MAL-1: RCE executed during loads; loads returned 0 (falsey) -> ServerSession(0, SAME sid) -> empty/anonymous; sid is SAME (NOT a new-uuid4 reset)
```

**Interpretation (with citations):**

- The payload begins with the **same `\x80\x04` PROTO-4 header** as a legitimate session payload (compare §2.2). The `STACK_GLOBAL` opcode resolves `posix.system` (i.e. `os.system`), and the `REDUCE` opcode **invokes it during unpickling** — the marker did not exist before `loads`, and exists (`'RCE_MAL1_EXECUTED'`) after, proving arbitrary code executed *inside* `pickle.loads` at `app/session.py:76`.
- **Crucially, `pickle.loads` returned `0`** — the `os.system` exit code — so it **did not raise**. Control therefore reached `ServerSession(0, session_id=session_id)` at `app/session.py:77`; `0` is falsey, so `dict.__init__(self, 0 or ())` builds an **empty** dict under the **SAME sid** (§6.1). The observed `probe_status=200 sid=same ttl=300` confirms it: the next real request sees an **anonymous, same-sid** session — **not** the `uuid4` reset of `:80`. **This refutes the intuitive assumption that a malicious pickle merely triggers the same session reset as malformed bytes do:** for `os.system`, there *is* a valid (falsey) return value, so `pickle.loads` **returns instead of raising**, `ServerSession(0, …)` succeeds at `:77`, and the sid is preserved — the code ran, but no `uuid4` reset occurred.
- The RCE is complete either way — the session-handling epilogue (same-sid-anonymous here) is *irrelevant to the attacker*, because the payload's side effect already happened at `:76`.

### 6.4 Malicious pickle #2 — RCE that *also* returns a valid auth dict (authentication PRESERVED) — evidence block 4.6 MAL-2

To make the point sharper, a second exploit's `__reduce__` returns `eval(expr)` where `expr` executes the marker command **and then evaluates to a captured valid authenticated session dict**. Now `pickle.loads` returns a real dict containing `_user_id`, so `ServerSession(dict, session_id)` at `:77` rebuilds an **authenticated** session under the SAME sid:

```text
===== BLOCK 4.6 MAL-2 MALICIOUS RCE + AUTH-PRESERVING DICT =====
pickle.loads(MAL-2) returned dict with _user_id? True
through open_session: probe_status=302 (302=auth preserved) sid=same ttl=604800
marker2 exists? True
CONCLUSION MAL-2: RCE executed AND loads returned a valid auth dict -> ServerSession(dict, SAME sid) -> authenticated retained
```

**Interpretation:** the marker was written (RCE executed inside `loads`), **and** the probe returned `302` with the SAME sid and TTL `604800` — i.e. the session is **still authenticated** after the malicious deserialization. This is outcome 2 of §6 taken to its extreme: not only is there no `uuid4` reset, the attacker's payload seamlessly reconstitutes a valid logged-in session while its side effect fires. It is the definitive counter-example to any "malicious pickle → silent reset" framing.

### 6.5 The boundary, stated precisely

- **Identical entry point, identical `pickle.loads(val)` at `app/session.py:76`.** The *sole* discriminator between a harmless reset and code execution is whether the attacker-controlled bytes constitute a **valid, weaponized** pickle program whose `__reduce__` returns a callable.
- **Non-malicious corruption** (§3/§5) makes `pickle.loads` **raise** → `except` at `:78` → **`uuid4` reset** at `:80` (NEW sid), harmless.
- **A weaponized pickle** executes its callable **inside `pickle.loads` at `:76`** (RCE), and *then* — because `loads` returned rather than raised — the `ServerSession(...)` at `:77` runs under the **SAME sid**, yielding an anonymous session (MAL-1, return `0`) or even a **fully authenticated** one (MAL-2, return a valid dict). The `uuid4` reset epilogue at `:80` is reached **only** if the returned object makes `ServerSession(...)` itself raise (§6.1, the `1`/`'abc'` matrix rows) — it is **not** the general outcome of the malicious case.

*(Context, not a recommendation: this is the textbook CWE-502 hazard of unpickling untrusted data — see §7.)*

---


## 7. Threat model: signed session-id *pointer* vs. unsigned pickle *payload*

**Sub-question (e):** *Does an attacker who can tamper with the stored session bytes but cannot forge the signed session id meaningfully change the risk?*

**Short answer: No — being unable to forge the signed id does not meaningfully reduce the risk.** The HMAC signature protects only the session-id *pointer* (which Redis key `open_session` reads), not the integrity of the unsigned pickle *payload* that pointer resolves to. An attacker with Redis **write** access therefore reaches `pickle.loads` with attacker-controlled bytes using the victim's *own* valid, unforged cookie.

### 7.1 The two artifacts and their integrity properties (observed + cited)

| Artifact | Where produced | Signed / integrity-protected? | Citation |
|----------|----------------|-------------------------------|----------|
| Session-id string (the `slapp` cookie's `<sid>` half) | `save_session` signs it with `self._get_signer(app).sign(...)` | **YES** — `itsdangerous` HMAC-SHA1, `salt="session"`, keyed by `app.secret_key` | `app/session.py:37-41`, `app/session.py:102-114` |
| Pickle payload (the Redis value under `session:<sid>`) | `save_session` writes `pickle.dumps(dict(session))` | **NO** — stored raw and unsigned via `setex` | `app/session.py:91`, `app/session.py:97-101` |

The signer covers the **sid only** (proven byte-for-byte in §2.2: reconstructing `signer.sign(sid)` reproduces the emitted cookie exactly). Nothing in `open_session` verifies, signs, or authenticates the Redis value before handing it to `pickle.loads` at `app/session.py:76` — the `if val is not None:` guard at `app/session.py:74` is the *only* check, and it tests presence, not integrity.

### 7.2 Signature-independence proof (evidence block 4.7 — verbatim)

The harness logs a victim in, captures the victim's genuine cookie, confirms it unsigns to the bare sid, then overwrites the victim's Redis payload with attacker bytes **without ever touching the cookie**, and issues the victim's next request with that same unchanged cookie:

```text
===== BLOCK 4.7 THREAT MODEL / SIGNATURE INDEPENDENCE =====
victim signed cookie = '1e9bf6b2-28e5-479c-9305-310a66700543.1IbYrczsQ4EUW2vPJRiAT58YENg'
unsign(cookie) = '1e9bf6b2-28e5-479c-9305-310a66700543'  <- cookie encodes ONLY the sid pointer (no payload)
attacker overwrites session:1e9bf6b2-28e5-479c-9305-310a66700543 payload WITHOUT touching the cookie ...
request with UNCHANGED cookie still reached pickle.loads with attacker bytes; probe_status=200 -> reset (NEW sid)
=> HMAC protects the sid pointer only; payload integrity is never verified before pickle.loads. Forging the cookie is unnecessary when Redis is writable; an attacker who CANNOT forge the cookie can still reach loads by writing the existing sid's key (see MAL-1/MAL-2).
```

**Interpretation (with citations):**

- **The signature is a function of the sid string alone.** The cookie unsigns to exactly the bare sid, and `salt="session"` confirms the signer domain (`app/session.py:37-41`). The signature is computed and verified over the session id, never over the Redis payload.
- **Tampering the payload does not invalidate the cookie.** The attacker overwrote only the Redis value; the cookie was never touched, and the victim's next request with that **unchanged** cookie was still accepted by `extract_and_validate_session_id` (`app/session.py:47-59`) and **still reached `pickle.loads`** with the attacker's bytes. This is the crux: the integrity check that *does* exist (the signature) is structurally blind to the payload it points at. (Here the attacker bytes were non-weaponized, so the outcome was a harmless reset — but the *reachability* of `pickle.loads` via the victim's own cookie is the point.)
- **The victim's own unforged cookie drives the exploit path.** Because reachability is established, the RCE demonstrations in §6.3/§6.4 are the same path with *weaponized* bytes: those blocks logged the victim in, overwrote the victim's own `session:<sid>` key, and issued the victim's own request via the victim's own signed cookie — and the marker was written (`app/session.py:73` reads the poisoned bytes, `:76` executes them). The attacker never forged, guessed, or stole the cookie.

**Conclusion for (e):** the signed id is *only a pointer*; the dangerous operation is performed on the *unsigned bytes it points to*. An attacker who can write to Redis but cannot forge the `slapp` cookie is therefore **not meaningfully constrained** — the victim (or any authenticated user whose key the attacker overwrote) supplies the valid pointer for free on their very next request. The inability to forge the id blocks *session hijacking by cookie forgery*, but it does **not** block *deserialization RCE via payload poisoning*, because those are two different attack surfaces protected by two different (and here, only one) integrity mechanisms.

### 7.3 Contingency and scope of the threat (observed boundary conditions)

The exploit is **contingent on Redis write access** and is *not* reachable by a normal web client:

- A normal client can only present a signed cookie. Tampering the signature is rejected at `extract_and_validate_session_id` → `None` (evidence block 4.4d: `signer.unsign(tampered)` raises `BadSignature`, the original key is never read). So a browser-only attacker cannot even reach the deserializer with chosen bytes.
- The RCE path opens only when the attacker can **write the Redis value** under an existing (or attacker-created) `session:<sid>` key — a precondition this investigation exercised **directly and at runtime** by overwriting the key from the observation harness (§6.3, §6.4, §7.2). *How* an attacker might obtain that Redis write access in a production deployment was **not observed** here; the following are **inferred, illustrative threat scenarios only** (background context, not runtime-observed): an exposed or compromised Redis instance, an SSRF-to-Redis primitive, a shared multi-tenant cache, or a network-adjacent attacker reaching an unauthenticated Redis port.

This contingency is stated honestly as a *precondition*, not a mitigation: given Redis write access, the signature provides no defense for the payload.

### 7.4 Classification and real-world precedent

This is a textbook instance of **CWE-502 (Deserialization of Untrusted Data)** ([MITRE CWE-502](https://cwe.mitre.org/data/definitions/502.html)), mapping to **OWASP Top 10 2021 category A08:2021 — Software and Data Integrity Failures** ([OWASP A08:2021](https://owasp.org/Top10/A08_2021-Software_and_Data_Integrity_Failures/)). The root property is documented in the official Python standard-library [`pickle` module documentation](https://docs.python.org/3/library/pickle.html), which explicitly warns that the module "is not secure" and that one should only unpickle data one trusts, because maliciously constructed pickle data can execute arbitrary code during unpickling — which is precisely why unpickling untrusted input constitutes CWE-502.

A close, recent real-world analogue is the **python-socketio** advisory [**GHSA-g8c6-8fjj-2r4m**](https://github.com/miguelgrinberg/python-socketio/security/advisories/GHSA-g8c6-8fjj-2r4m) ([**CVE-2025-61765**](https://nvd.nist.gov/vuln/detail/CVE-2025-61765), published October 2025). As described in that upstream advisory, python-socketio multi-server deployments that use a message-queue backend such as Redis encode their inter-server messages with `pickle` and deserialize received messages with `pickle.loads()` on the assumption that the queue is trusted. An attacker who has *already obtained access to that message queue* can therefore place a crafted pickle payload whose `__reduce__` method executes arbitrary code when a receiving server deserializes it. The advisory is explicit that the exposure is gated on backing-store access — single-server deployments without a message queue, and multi-server deployments whose queue is secured, are unaffected — and the maintainer's fix removed the pickle-based inter-server encoding in favor of a safer JSON scheme.

The parallel to SimpleLogin's `RedisSessionStore` is exact: the dangerous `pickle.loads` (`app/session.py:76`) runs against bytes fetched from a Redis backing store, and the exploit is reachable **only** when an attacker can write to that store — differing from the socketio case only in that the poisoned bytes arrive via a server-side *session* value rather than an inter-server *message*. *(This paragraph is background context for classification only; per the read-only scope it is explicitly not a remediation proposal.)*

### 7.5 API logout — verified by code reference (NOT runtime-exercised)

The API logout route is covered here by **code reference**, since it was not exercised at runtime. The route `GET /api/logout` (`app/api/views/user_info.py:131`) invokes the **same** `logout_session()` (`app/api/views/user_info.py:140`) demonstrated for web logout in §2.3, then deletes the session cookie (`app/api/views/user_info.py:142`); the route is guarded by `@require_api_auth`. Because it calls the identical teardown function (`logout_session()` → `logout_user()` + `purge_session()`, `app/session.py:117-121`), its session-destruction mechanism is the same as the runtime-exercised web logout. This item is labeled **verified-by-code-reference (same mechanism as web logout)**, *not* runtime-observed, and appears as such in the observed-vs-inferred ledger (§9).

---

## 8. Coverage pass — every sub-question mapped to its answer and evidence

The original question decomposes into five named parts. Each is answered explicitly, next to actual runtime output, as follows:

| # | Sub-question | Answer (one line) | Section | Evidence block | Status |
|---|--------------|-------------------|---------|----------------|--------|
| — | **How is session data deserialized?** | A single `pickle.loads(val)` in `open_session` at `app/session.py:76`; serializer is stdlib `pickle` (protocol 4, `\x80\x04`) after the Py3 `cPickle` `ImportError` fallback at `app/session.py:9-12`. Both `loads` and the `ServerSession(...)` that consumes it are inside one `try` (`:75-77`). | §2, §3, §6 | 4.0, 4.1, 4.3 | ✅ answered |
| (a) | **Baseline login/logout lifecycle** | Login writes `pickle.dumps(dict(session))` to `session:<sid>` via `setex` (7-day TTL) and sets a signed `slapp` cookie; logout deletes the key + 3 cookies and re-sets a fresh anonymous cookie (payload keeps `_permanent`/`sudo_time`/`_flashes`, drops `_user_id`). | §2 | 4.0, 4.1, 4.2 | ✅ answered |
| (b) | **Corrupted/malformed payload behavior on the next request (fail? silent reset? surfaced error?)** | **Silent reset.** `pickle.loads` raises, `except Exception: pass` (`app/session.py:78-79`) swallows it, a fresh `ServerSession(uuid4)` is returned (`:80`). Response is a normal `302` to `/auth/login` (or `200` login form) — **no failure, no surfaced error, no 500**. | §3 | 4.3, 4.3b | ✅ answered |
| (c) | **What appears in the HTTP response and server logs** | Response: normal redirect/login page (no error page). Logs: exactly **one** generic `after_request()` line (`server.py:284`), differing from a normal request only by status; **no** pickle/session/reset-specific log, because `app/session.py` imports no logger (`app/session.py:1-16`). The reset is indistinguishable from "user was never logged in." | §4 | 4.5 | ✅ answered |
| (d) | **Boundary between a harmless reset and a genuine deserialization risk** | Same `pickle.loads` at `:76`; outcome set by **what `loads` returns**, since `loads` **and** `ServerSession(...)` share one `try` (`:75-77`). Invalid/benign bytes **raise** → `uuid4` reset (`:80`). A **weaponized** pickle executes `__reduce__` *inside* `loads` **before** the `except` at `:78`; because it *returns* (e.g. `os.system`→`0`), `ServerSession(0, SAME sid)` runs at `:77` — **same-sid anonymous, NOT a reset** (MAL-1); a returned valid dict even **keeps auth** (MAL-2). Sole discriminator: is the byte-string a valid weaponized pickle? | §6 | 4.6 (matrix, MAL-1, MAL-2) | ✅ answered |
| (e) | **Threat model: attacker can tamper stored bytes but cannot forge the signed id** | Risk is **not** meaningfully reduced. HMAC signs only the sid *pointer* (`app/session.py:37-41`); the payload is stored raw/unsigned (`app/session.py:91`). Overwriting the payload leaves the cookie valid; the victim's *own* unforged cookie points `open_session` at poisoned bytes → `pickle.loads` reached → RCE (§6.3/§6.4 via the victim's own cookie). CWE-502 / OWASP A08:2021; analogous to GHSA-g8c6-8fjj-2r4m. | §7 | 4.7 (+ 4.6 MAL-1/2) | ✅ answered |

Supplementary edge/error conditions the question implies (all exercised): deleted/missing Redis value (`None`) — `pickle.loads` never reached (`app/session.py:74` guard), §5 / block 4.4a; stored-empty `b''` value — not `None`, so `pickle.loads` **is** called and raises `EOFError: Ran out of input`, caught, §5 / block 4.4b; truncated pickle — `UnpicklingError: pickle data was truncated`, caught, §5 / block 4.4c; `BadSignature` cookie — rejected pre-Redis (`app/session.py:47-59`), original key never read, §5 / block 4.4d.

---

## 9. Observed-vs-inferred ledger & reproducibility

Per the SWE-AtlasQnA-Repo methodology, every claim is classified below as **observed** (produced at runtime and shown verbatim in §2–§7), **captured** (a byte-sensitive value verified against the exact emitted bytes), or **inferred / code-reference** (not directly runtime-exercised).

| Claim | Classification | Basis |
|-------|----------------|-------|
| Deserializer is stdlib `pickle.loads` at `app/session.py:76` | **Observed** | Raised `UnpicklingError`/`EOFError` on corrupted/truncated/empty paths (blocks 4.3, 4.4b, 4.4c) |
| Pickle protocol = 4 (`\x80\x04` PROTO opcode) | **Captured** | `pickle.DEFAULT_PROTOCOL = 4` printed (4.0); every emitted payload begins `800495…` (hex); confirmed by `pickletools.dis` (4.1, 4.6) |
| `slapp` cookie signature covers the sid only | **Captured** | `signer.sign(sid)` reproduces the emitted cookie byte-for-byte (4.1); the cookie unsigns to the bare sid, unchanged by payload tamper (4.7) |
| Corrupted payload → silent reset (new sid, no error) | **Observed** | sid change + `302`/`200` + swallowed `UnpicklingError` captured (4.3, 4.3b) |
| Reset emits no session/pickle-specific log line | **Observed** | `grep` proof of no logger import + fd-level A/B log capture, all six token checks `False` (4.5) |
| Deleted / missing value (`None`) → `pickle.loads` never called | **Observed** | `deleted key -> val is None`, fresh reset, `after: exists=False ttl=-2` (4.4a) |
| Stored-empty `b''` (not `None`) → `pickle.loads` called → `EOFError` → reset | **Observed** | `len=0`, `EOFError: Ran out of input` swallowed, NEW sid (4.4b) |
| `BadSignature` cookie → rejected before Redis/pickle | **Observed** | `BadSignature: … does not match`, NEW sid, original key never read (4.4d) |
| Valid weaponized pickle → RCE inside `pickle.loads` | **Observed** | marker file created in isolation *and* through the real `open_session` path via the victim's own cookie (4.6 MAL-1/MAL-2) |
| **Weaponized `os.system` pickle returns `0` → `ServerSession(0, SAME sid)`, NOT a `uuid4` reset** | **Observed** | `pickle.loads(MAL-1) returned: 0`; `probe_status=200 sid=same ttl=300` (4.6 MAL-1) |
| **Weaponized pickle returning a valid dict → authentication preserved (SAME sid)** | **Observed** | `probe_status=302 sid=same ttl=604800`, marker written (4.6 MAL-2) |
| **Return-object model: falsey/dict-convertible returns keep SAME sid; `1`/`'abc'` raise in `ServerSession` → NEW sid** | **Observed** | matrix: `None/[]/0/''/[('x',1)]/dict` → same; `1`,`'abc'` → NEW (4.6 matrix) |
| Payload tampering does not change cookie validity | **Observed** | victim's unchanged cookie still reached `pickle.loads` with attacker bytes (4.7) |
| Authenticated TTL = 604800 s (7 days) | **Observed** | `setex` TTL captured on the authenticated key at login (4.1); `ttl = int(app.permanent_session_lifetime.total_seconds())` at `app/session.py:92` |
| Anonymous TTL = 300 s | **Observed** | `setex` TTL captured on the anonymous key re-saved during logout (4.2) and on same-sid matrix keys (4.6); `if "_user_id" not in session: ttl = 300` at `app/session.py:95-96` |
| `permanent_session_lifetime = 2678400 s (31 days)` at module rest | **Observed (at-rest)** | Printed at import (4.0); overridden per-request to 7 days by `make_session_permanent` (`server.py:204-207`) |
| Legitimate payload keys `{_permanent,_fresh,_user_id,_id,sudo_time}` | **Observed** | `pickle.loads(payload)` dict + `pickletools.dis` (4.1); no `csrf_token` because harness sets `WTF_CSRF_ENABLED=False` |
| API logout (`GET /api/logout`) destroys the session identically | **Inferred / code-reference** | Not runtime-exercised; calls the same `logout_session()` (`app/api/views/user_info.py:131,140,142`) — §7.5 |
| CWE-502 / OWASP A08:2021 classification & GHSA-g8c6-8fjj-2r4m analogy | **Background (external sources)** | Python `pickle` docs, OWASP, and the python-socketio advisory (CVE-2025-61765) — §7.4 |

**Reproducibility.** The evidence blocks embedded verbatim in §2–§7 are the **actual, unedited output of a single observation-harness run *inside the supplied canonical GHCR container*** (§1.2/§1.3), whose `/app` is the source commit `2cd6ee777f8c…`. The identical harness was also run on the native host install (Python 3.10.20, Redis 8.0.2, PG 17.10) as a confirming cross-check, and the return-object behavior of §6 — including MAL-1 returning `0` under the SAME sid and MAL-2 preserving authentication — was **identical**. Per-run values differ (session ids are fresh `uuid4`s, signatures/`sudo_time`/user-emails vary), but the *structure* and *behavior* are invariant: cookie shape `<sid>.<sig>`, protocol-4 payload starting `\x80\x04`, silent `uuid4` reset on a *raising* payload, same-sid consumption on a *returning* payload, no reset-specific log, and RCE on a valid weaponized pickle. The flask-login "strong" session fingerprint `_id` (`002d6488…0c0916`) is deterministic for a fixed request environment and was reproduced **byte-identically** across the container run and the native cross-check.

---


## Appendix A — The observation harness (complete, verbatim, exactly as run)

The script below is reproduced **complete and verbatim — exactly as executed inside the canonical GHCR container** — with no abbreviation, no comment-elision, and no pseudocode. It is the single temporary exerciser that produced evidence blocks **4.0–4.8** transcribed in §2–§7; it lived **outside** the repository (copied into the container at `/tmp/obs_harness.py`) and was removed when the container was destroyed (Appendix B). A reader can copy it, stand up the canonical environment exactly as in §1.3, and run it to regenerate structurally-identical evidence (per-run session ids, signatures, and timestamps differ — see §9).

The harness mirrors the canonical `tests/conftest.py` harness: `CONFIG=tests/test.env`, `create_app()`, `TESTING=True`, `WTF_CSRF_ENABLED=False`, `SERVER_NAME="sl.test"`, an authenticated user built with **`create_new_user()` imported from `tests/utils.py`** (password `"password"`), all wrapped in `connection.begin()` and rolled back at teardown. Redis handles use the same `redis://localhost` the app uses; the `login()`/probe pattern is the `POST auth.login` / `GET auth.login` template from `tests/auth/test_login.py`.

**Ownership-safe Redis cleanup (block 4.8 — corrected).** The harness tracks **every** session id it causes Redis to write — the login sids **and** every rotation — in the `CREATED_SIDS` set (via the `remember()` helper, which unsigns each `Set-Cookie: slapp` header, and via `fresh_login()` / `auth_probe()`). Its cleanup deletes **only** those tracked keys, one at a time with `r.delete(key(sid))` — it **never** runs a blanket `r.keys('session:*')` + `delete`. To prove this is ownership-safe, block 4.8 plants a **foreign sentinel** `session:qa-foreign-sentinel` before cleanup and confirms it **survives** (observed 4.8: `30` keys before; `28` tracked keys removed; `foreign sentinel … still exists AFTER cleanup? True`; a blanket delete `would have deleted 30 keys INCLUDING the foreign sentinel`). This matters because the same Redis db can hold other tenants' / parallel agents' session keys, which a blanket wipe would destroy.

The verbatim, unedited output of block 4.8 (from the authoritative container run) proves the ownership-safe cleanup leaves the foreign key untouched:

```text
===== BLOCK 4.8 OWNERSHIP-SAFE CLEANUP (F2) + FOREIGN SENTINEL SURVIVAL =====
total session:* keys before cleanup = 30
this run OWNS 30 sids (tracked, incl. rotations); foreign sentinel present? True
ownership-safe deletion removed 28 keys (ONLY this run's tracked sids; NO blanket keys('session:*'))
foreign sentinel session:qa-foreign-sentinel still exists AFTER cleanup? True
remaining session:* keys after ownership-safe cleanup = ['session:80b97b45-6537-474b-a373-0879ba139534', 'session:qa-foreign-sentinel']
CONTRAST: a blanket keys('session:*')+delete would have deleted 30 keys INCLUDING the foreign sentinel.
test sentinel removed by owner; our tracked keys remaining = 0
```

The `foreign sentinel … still exists AFTER cleanup? True` line is the empirical proof of the F2 correction: only the run's own tracked sids are deleted, and the `CONTRAST` line quantifies exactly what a blanket `keys('session:*')` wipe would have destroyed (all `30`, including the co-tenant sentinel).


**Database-state note (honest — and why the disposable container makes it moot).** The `connection.begin()` / `transaction.rollback()` teardown is *not* a fully-isolating SAVEPOINT restart (`begin_nested()`), so any application-level `Session.commit()` that fires mid-request escapes the outer rollback. The auth-state probe used throughout §3–§7 is `GET auth.login`, which is **non-committing**; but the baseline block 4.1 issues an authenticated `GET /dashboard/`, whose first load flips `intro_shown` to `True` and calls `Session.commit()` at `app/dashboard/views/index.py:177` — that commit persists the created `users` row despite the teardown. On the native cross-check host this residue is real, which is exactly why the harness is run there with `OBS_SKIP_DASHBOARD=1` (see §1.3) to avoid polluting the shared database; inside the **authoritative container** it is irrelevant because the entire container — PostgreSQL, Redis, script, and markers — is **destroyed** at completion (Appendix B), wiping every row and key. The git repository / working tree is left unchanged in both environments — the read-only guarantee that actually matters here, verified in Appendix B.

```python
# -*- coding: utf-8 -*-
# Ownership-safe runtime observation harness for SimpleLogin RedisSessionStore.
# READ-ONLY w.r.t. the repository. Exercises REAL entry points (POST auth.login,
# GET auth.login as a non-committing auth-state probe, GET auth.logout).
# Tracks EVERY sid it causes to be written (login + rotations) and deletes ONLY
# those (ownership, F2); rolls back its DB transaction. Prints unedited output.
import os, sys, io, uuid, binascii, pickle, pickletools, contextlib, traceback

from flask import testing, url_for
import redis as redislib

from server import create_app
from app import config as app_config
from app import constants
from app.db import Session, connection
from app.session import RedisSessionStore, ServerSession, SESSION_PREFIX
from tests.utils import create_new_user
import itsdangerous

def line(t=""): print(t, flush=True)
def hr(title): line("\n===== " + title + " =====")

app = create_app()
app.config["TESTING"] = True
app.config["WTF_CSRF_ENABLED"] = False
app.config["SERVER_NAME"] = "sl.test"
app_config.DISABLE_RATE_LIMIT = True

class CustomTestClient(testing.FlaskClient):
    def open(self, *args, **kwargs):
        if args and isinstance(args[0], str):
            headers = kwargs.pop("headers", {})
            headers.update({constants.HEADER_ALLOW_API_COOKIES: "allow"})
            kwargs["headers"] = headers
        return super().open(*args, **kwargs)
app.test_client_class = CustomTestClient

MEM = os.environ.get("MEM_STORE_URI") or "redis://localhost"
r = redislib.Redis.from_url(MEM)

CREATED_SIDS = set()
SIGNER = RedisSessionStore._get_signer(app)
COOKIE_NAME = app.config["SESSION_COOKIE_NAME"]

def key(sid): return f"{SESSION_PREFIX}:{sid}"
def new_client():
    c = app.test_client()
    c.environ_base[constants.HEADER_ALLOW_API_COOKIES] = "allow"
    return c
def signed(sid): return SIGNER.sign(itsdangerous.want_bytes(sid)).decode()

def setcookie_headers(resp):
    try: return list(resp.headers.getlist("Set-Cookie"))
    except Exception: return [v for (k,v) in resp.headers if k == "Set-Cookie"]

def slapp_from_headers(resp):
    for v in setcookie_headers(resp):
        if v.startswith(COOKIE_NAME + "=") and not v.startswith(COOKIE_NAME + "=;"):
            return v.split(";",1)[0][len(COOKIE_NAME)+1:]
    return None

def remember(resp):
    """Track every sid the run causes to be written (login + rotations)."""
    raw = slapp_from_headers(resp)
    if raw:
        try:
            sid = SIGNER.unsign(raw).decode(); CREATED_SIDS.add(sid); return sid
        except Exception:
            return None
    return None

def fresh_login(user):
    before = set(r.keys(f"{SESSION_PREFIX}:*"))
    c = new_client()
    resp = c.post(url_for("auth.login"),
                  data={"email": user.email, "password": "password"})
    after = set(r.keys(f"{SESSION_PREFIX}:*"))
    sid = None
    for k in (after - before):
        try:
            d = pickle.loads(r.get(k))
            if isinstance(d, dict) and "_user_id" in d:
                sid = k.decode().split(":",1)[1]; break
        except Exception:
            pass
    if sid is None and (after - before):
        sid = sorted(after - before)[0].decode().split(":",1)[1]
    if sid: CREATED_SIDS.add(sid)
    remember(resp)
    return c, resp, sid

def auth_probe(c):
    resp = c.get(url_for("auth.login"))
    raw = slapp_from_headers(resp)
    newsid = None
    if raw:
        try: newsid = SIGNER.unsign(raw).decode()
        except Exception: newsid = None
    if newsid: CREATED_SIDS.add(newsid)
    return resp.status_code, newsid

def snapshot(sid):
    k = key(sid); val = r.get(k); ttl = r.ttl(k)
    return {"exists": val is not None, "ttl": ttl,
            "bytes_len": (len(val) if val is not None else None), "raw": val}

def capture_fd(fn):
    """Run fn() capturing fd-level stdout+stderr (bytes)."""
    capfile = "/tmp/blitzy_obs_fdcap.txt"
    sys.stdout.flush(); sys.stderr.flush()
    so, se = os.dup(1), os.dup(2)
    fd = os.open(capfile, os.O_WRONLY | os.O_CREAT | os.O_TRUNC, 0o644)
    os.dup2(fd, 1); os.dup2(fd, 2)
    try:
        fn()
    finally:
        sys.stdout.flush(); sys.stderr.flush()
        os.dup2(so, 1); os.dup2(se, 2)
        os.close(fd); os.close(so); os.close(se)
    with open(capfile, "rb") as fh: data = fh.read()
    os.remove(capfile); return data

transaction = connection.begin()
BASE_AUTH_BYTES = None
BASE_AUTH_DICT = None

with app.app_context():
    try:
        # ---------------- BLOCK 4.0 : environment / provenance ----------------
        hr("BLOCK 4.0 ENVIRONMENT")
        import flask as _fl, itsdangerous as _it, werkzeug as _wz, redis as _rd
        import flask_login as _flg, limits as _lim
        line("python: " + sys.version.replace("\n"," "))
        line("flask=%s itsdangerous=%s werkzeug=%s redis=%s flask_login=%s limits=%s"
             % (_fl.__version__, _it.__version__, _wz.__version__, _rd.__version__,
                _flg.__version__, getattr(_lim,'__version__','?')))
        line("MEM_STORE_URI=%r" % MEM)
        line("session_interface=%s" % type(app.session_interface).__name__)
        line("SESSION_COOKIE_NAME=%r" % COOKIE_NAME)
        line("app.secret_key=%r" % app.secret_key)
        line("permanent_session_lifetime(default,outside request)=%d s"
             % int(app.permanent_session_lifetime.total_seconds()))
        line("redis client type = %s.%s" % (type(r).__module__, type(r).__name__))
        line("pickle.DEFAULT_PROTOCOL = %d ; HIGHEST_PROTOCOL = %d"
             % (pickle.DEFAULT_PROTOCOL, pickle.HIGHEST_PROTOCOL))
        line("signer = %s.%s" % (type(SIGNER).__module__, type(SIGNER).__name__))
        line("signer.digest_method = %r" % SIGNER.digest_method)
        line("signer.sep = %r" % SIGNER.sep)

        user = create_new_user()
        Session.flush()
        line("test user id=%s email=%s" % (user.id, user.email))

        # ---------------- BLOCK 4.1 : baseline login ----------------
        hr("BLOCK 4.1 BASELINE LOGIN (POST auth.login)")
        before = set(r.keys(f"{SESSION_PREFIX}:*"))
        c = new_client()
        resp = c.post(url_for("auth.login"),
                      data={"email": user.email, "password": "password"})
        after = set(r.keys(f"{SESSION_PREFIX}:*"))
        newk = sorted(after - before)
        line("POST /auth/login -> status=%s location=%s"
             % (resp.status_code, resp.headers.get("Location")))
        sid = None
        for k in newk:
            d = pickle.loads(r.get(k))
            if isinstance(d, dict) and "_user_id" in d:
                sid = k.decode().split(":",1)[1]
        assert sid, "no authenticated session key created"
        CREATED_SIDS.add(sid); remember(resp)
        raw = r.get(key(sid)); ttl = r.ttl(key(sid))
        BASE_AUTH_BYTES = raw
        BASE_AUTH_DICT = pickle.loads(raw)
        line("session id (sid) = %s" % sid)
        line("redis key = %s" % key(sid))
        line("redis TTL seconds = %s  (authenticated: make_session_permanent sets 7d)" % ttl)
        line("pickled payload len = %d bytes" % len(raw))
        line("pickled payload hex = %s" % binascii.hexlify(raw).decode())
        line("pickle.loads(payload) = %r" % (BASE_AUTH_DICT,))
        line("--- pickletools.dis(payload) ---")
        _b = io.StringIO(); pickletools.dis(raw, _b); line(_b.getvalue().rstrip())
        hdr_cookie = slapp_from_headers(resp)
        computed = signed(sid)
        line("Set-Cookie slapp (from response) = %r" % hdr_cookie)
        line("signer.sign(sid)               = %r" % computed)
        line("cookie == signer.sign(sid) ? %s" % (hdr_cookie == computed))
        line("signer.unsign(cookie).decode() = %r"
             % SIGNER.unsign(hdr_cookie or computed).decode())
        if os.environ.get("OBS_SKIP_DASHBOARD") != "1":
            rc = c.get(url_for("dashboard.index"))
            line("GET /dashboard/ (authenticated) -> status=%s" % rc.status_code)
        else:
            line("GET /dashboard/ SKIPPED (OBS_SKIP_DASHBOARD=1: native cross-check avoids intro_shown persist)")

        # ---------------- BLOCK 4.2 : logout teardown ----------------
        hr("BLOCK 4.2 LOGOUT TEARDOWN (GET auth.logout)")
        pre = snapshot(sid)
        rl = c.get(url_for("auth.logout"))
        remember(rl)
        post_exists = r.get(key(sid)) is not None
        line("before logout: redis key exists=%s ttl=%s" % (pre["exists"], pre["ttl"]))
        line("GET /auth/logout -> status=%s location=%s"
             % (rl.status_code, rl.headers.get("Location")))
        line("Set-Cookie headers on logout response:")
        for v in setcookie_headers(rl): line("   " + v)
        line("after logout: OLD authenticated redis key %s exists=%s" % (key(sid), post_exists))
        new_raw_cookie = slapp_from_headers(rl)
        if new_raw_cookie:
            new_sid = SIGNER.unsign(new_raw_cookie).decode()
            nk = key(new_sid); nval = r.get(nk)
            line("post-logout NEW anonymous key = %s" % nk)
            if nval is not None:
                line("post-logout NEW payload hex = %s" % binascii.hexlify(nval).decode())
                ndict = pickle.loads(nval)
                line("post-logout NEW payload pickle.loads = %r" % (ndict,))
                line("post-logout NEW payload keys (sorted) = %r" % sorted(ndict.keys()))
                line("post-logout NEW payload TTL = %s s (anonymous: '_user_id' absent -> 300)" % r.ttl(nk))

        def condition(title, make_bytes, expect_note=""):
            hr(title)
            cc, _rp, s = fresh_login(user)
            b_before = snapshot(s)
            payload = make_bytes()
            if payload is None:
                r.delete(key(s)); during = "(deleted key -> val is None)"
            else:
                r.set(key(s), payload); during = "len=%d hex=%s" % (
                    len(payload), binascii.hexlify(payload).decode())
            line("sid=%s" % s)
            line("before: exists=%s ttl=%s" % (b_before["exists"], b_before["ttl"]))
            line("during (redis session:%s value): %s" % (s, during))
            st, newsid = auth_probe(cc)
            same = (newsid == s)
            after_same = snapshot(s)
            line("auth_probe GET /auth/login -> status=%s (200=anon,302=auth)" % st)
            line("response Set-Cookie sid = %s  (same as before? %s)" % (newsid, same))
            line("after: session:%s exists=%s ttl=%s" % (s, after_same["exists"], after_same["ttl"]))
            if expect_note: line("interpretation: " + expect_note)
            return st, same, newsid

        # ---------------- BLOCK 4.3 : corrupted / malformed ----------------
        condition("BLOCK 4.3 CORRUPTED MALFORMED PICKLE",
                  lambda: b"not-a-pickle-\x00\xff\x99",
                  "malformed bytes -> pickle.loads raises -> except -> NEW sid reset")
        try:
            pickle.loads(b"not-a-pickle-\x00\xff\x99")
        except Exception as e:
            line("pickle.loads(malformed) raised: %s: %s" % (type(e).__name__, e))

        # ---------------- BLOCK 4.3b : corrupted -> GET /dashboard/ (302 redirect chain) ----------------
        hr("BLOCK 4.3b CORRUPTED PAYLOAD THROUGH GET /dashboard/ (login-required redirect chain)")
        cc, _rp, s_dash = fresh_login(user)
        pre_dash = cc.get(url_for("dashboard.index"))
        line("BEFORE corruption: authenticated sid=%s ; GET /dashboard/ -> status=%s (200=authenticated)"
             % (s_dash, pre_dash.status_code))
        r.set(key(s_dash), b"not-a-pickle-\x00\xff\x99")
        rd = cc.get(url_for("dashboard.index"))
        newsid_dash = None
        _rawc = slapp_from_headers(rd)
        if _rawc:
            try: newsid_dash = SIGNER.unsign(_rawc).decode(); CREATED_SIDS.add(newsid_dash)
            except Exception: pass
        line("AFTER (next real GET /dashboard/): status=%s ; Location=%s"
             % (rd.status_code, rd.headers.get("Location")))
        line("sid %s: %s -> %s"
             % (("CHANGED == SILENT RESET" if (newsid_dash and newsid_dash != s_dash) else "unchanged"),
                s_dash, newsid_dash))

        # ---------------- BLOCK 4.4 : edge / error paths ----------------
        condition("BLOCK 4.4a MISSING REDIS VALUE",
                  lambda: None,
                  "val is None -> pickle.loads NOT called -> NEW sid reset")
        condition("BLOCK 4.4b EMPTY BYTES",
                  lambda: b"",
                  "val==b'' is not None -> loads(b'') raises EOFError -> NEW sid reset")
        try:
            pickle.loads(b"")
        except Exception as e:
            line("pickle.loads(b'') raised: %s: %s" % (type(e).__name__, e))
        condition("BLOCK 4.4c TRUNCATED PICKLE",
                  lambda: BASE_AUTH_BYTES[: len(BASE_AUTH_BYTES)//2],
                  "truncated valid pickle -> loads raises -> NEW sid reset")
        try:
            pickle.loads(BASE_AUTH_BYTES[: len(BASE_AUTH_BYTES)//2])
        except Exception as e:
            line("pickle.loads(truncated) raised: %s: %s" % (type(e).__name__, e))

        # BadSignature edge (tamper the SIGNED sid payload so HMAC genuinely fails)
        hr("BLOCK 4.4d BAD SIGNATURE COOKIE")
        cc, _rp, s = fresh_login(user)
        good = signed(s)
        _dot = good.rfind(".")
        _sidp, _sigp = good[:_dot], good[_dot:]
        tampered = (("X" if _sidp[0] != "X" else "Y") + _sidp[1:]) + _sigp
        line("good cookie    = %r" % good)
        line("tampered cookie= %r  (first char of signed sid flipped)" % tampered)
        try:
            SIGNER.unsign(tampered)
            line("signer.unsign(tampered) -> DID NOT RAISE (unexpected)")
        except Exception as e:
            line("signer.unsign(tampered) raised: %s: %s" % (type(e).__name__, e))
        cc2 = new_client()
        try:
            cc2.set_cookie("sl.test", COOKIE_NAME, tampered)
        except TypeError:
            cc2.set_cookie(COOKIE_NAME, tampered, domain="sl.test")
        st, newsid = auth_probe(cc2)
        line("auth_probe with tampered cookie -> status=%s (200=anon) newsid=%s (differs from %s? %s)"
             % (st, newsid, s, newsid != s))
        line("interpretation: BadSignature -> extract_and_validate returns None -> "
             "fresh uuid4; the sid %s Redis key is never read" % s)

        # ---------------- BLOCK 4.5 : observability (no reset-specific log) ----------------
        hr("BLOCK 4.5 OBSERVABILITY (fd-level capture: NORMAL vs RESET request)")
        # (i) a NORMAL valid-session request
        cc, _rp, s_norm = fresh_login(user)
        norm = capture_fd(lambda: cc.get(url_for("auth.login")))
        # (ii) a RESET request (corrupted payload) on a fresh login
        cc, _rp, s_bad = fresh_login(user)
        r.set(key(s_bad), b"corrupt-\x00")
        bad = capture_fd(lambda: cc.get(url_for("auth.login")))
        line("captured during NORMAL request = %r" % norm)
        line("captured during RESET  request = %r" % bad)
        for tok in (b"session", b"pickle", b"Unpickling", b"reset", b"Exception", b"Traceback"):
            line("  RESET capture contains %-11r ? %s" % (tok.decode(), tok in bad))
        line("=> the ONLY output during a reset is server.py:284's generic per-request "
             "access log (present for EVERY request); open_session's except branch "
             "(L78-79 'pass') emits nothing session/pickle/reset-specific.")

        # ---------------- BLOCK 4.6 : RETURN-OBJECT MATRIX + malicious RCE (F1) ----------------
        hr("BLOCK 4.6 RETURN-OBJECT MATRIX (non-malicious pickles)")
        line("Each row: pickle.loads returns <obj> -> ServerSession(<obj>, SAME sid) inside try (L77).")
        line("Legend: probe 200=anonymous, 302=authenticated; sid 'same' means NO reset.")
        matrix = [
            ("None",           pickle.dumps(None)),
            ("[]",             pickle.dumps([])),
            ("0",              pickle.dumps(0)),
            ("'' (empty str)", pickle.dumps("")),
            ("1",              pickle.dumps(1)),
            ("'abc'",          pickle.dumps("abc")),
            ("[('x',1)]",      pickle.dumps([("x", 1)])),
            ("VALID_AUTH_DICT",BASE_AUTH_BYTES),
        ]
        results = []
        for label, payload in matrix:
            cc, _rp, s = fresh_login(user)
            r.set(key(s), payload)
            st, newsid = auth_probe(cc)
            same = (newsid == s)
            aft = snapshot(s)
            results.append((label, st, "same" if same else "NEW", aft["ttl"]))
            line("loads->%-16s probe_status=%s  sid=%-4s  after_ttl(session:%s)=%s"
                 % (label, st, ("same" if same else "NEW"), s, aft["ttl"]))
        line("--- matrix summary (label | probe | sid | ttl-of-orig-key) ---")
        for row in results: line("   %-16s | %s | %-4s | %s" % row)

        hr("BLOCK 4.6 MAL-1 MALICIOUS os.system MARKER (RCE, returns 0)")
        marker1 = "/tmp/blitzy_obs_rce_mal1.txt"
        if os.path.exists(marker1): os.remove(marker1)
        class _RCE1:
            def __reduce__(self):
                return (os.system, ("echo RCE_MAL1_EXECUTED > " + marker1,))
        mal1 = pickle.dumps(_RCE1())
        line("malicious pickle len=%d hex=%s" % (len(mal1), binascii.hexlify(mal1).decode()))
        line("--- pickletools.dis(malicious MAL-1) ---")
        _b = io.StringIO(); pickletools.dis(mal1, _b); line(_b.getvalue().rstrip())
        line("marker BEFORE loads exists? %s" % os.path.exists(marker1))
        _probe = pickle.loads(mal1)
        line("pickle.loads(MAL-1) returned: %r (this is os.system's exit code)" % (_probe,))
        line("marker AFTER  loads exists? %s" % os.path.exists(marker1))
        if os.path.exists(marker1):
            line("marker contents = %r" % open(marker1).read().strip())
        cc, _rp, s = fresh_login(user)
        if os.path.exists(marker1): os.remove(marker1)
        r.set(key(s), mal1)
        st, newsid = auth_probe(cc)
        same = (newsid == s)
        aft = snapshot(s)
        line("through open_session: probe_status=%s sid=%s ttl(session:%s)=%s"
             % (st, ("same" if same else "NEW"), s, aft["ttl"]))
        line("marker file after real request exists? %s" % os.path.exists(marker1))
        line("CONCLUSION MAL-1: RCE executed during loads; loads returned 0 (falsey) -> "
             "ServerSession(0, SAME sid) -> empty/anonymous; sid is %s (NOT a new-uuid4 reset)"
             % ("SAME" if same else "NEW"))

        hr("BLOCK 4.6 MAL-2 MALICIOUS RCE + AUTH-PRESERVING DICT")
        marker2 = "/tmp/blitzy_obs_rce_mal2.txt"
        if os.path.exists(marker2): os.remove(marker2)
        expr = "(__import__('os').system(%r), %r)[1]" % (
            "echo RCE_MAL2_EXECUTED > " + marker2, BASE_AUTH_DICT)
        class _RCE2:
            def __reduce__(self):
                return (eval, (expr,))
        try:
            mal2 = pickle.dumps(_RCE2())
            probe2 = pickle.loads(mal2)
            line("pickle.loads(MAL-2) returned dict with _user_id? %s"
                 % (isinstance(probe2, dict) and "_user_id" in probe2))
            cc, _rp, s = fresh_login(user)
            if os.path.exists(marker2): os.remove(marker2)
            r.set(key(s), mal2)
            st, newsid = auth_probe(cc)
            same = (newsid == s)
            aft = snapshot(s)
            line("through open_session: probe_status=%s (302=auth preserved) sid=%s ttl=%s"
                 % (st, ("same" if same else "NEW"), aft["ttl"]))
            line("marker2 exists? %s" % os.path.exists(marker2))
            line("CONCLUSION MAL-2: RCE executed AND loads returned a valid auth dict -> "
                 "ServerSession(dict, SAME sid) -> authenticated retained")
        except Exception as e:
            line("MAL-2 could not be constructed/observed: %s: %s" % (type(e).__name__, e))

        # ---------------- BLOCK 4.7 : threat model / signature independence ----------------
        hr("BLOCK 4.7 THREAT MODEL / SIGNATURE INDEPENDENCE")
        cc, _rp, s = fresh_login(user)
        ck = signed(s)
        line("victim signed cookie = %r" % ck)
        line("unsign(cookie) = %r  <- cookie encodes ONLY the sid pointer (no payload)" % SIGNER.unsign(ck).decode())
        line("attacker overwrites session:%s payload WITHOUT touching the cookie ..." % s)
        r.set(key(s), b"attacker-controlled-\x00bytes")
        st, newsid = auth_probe(cc)
        line("request with UNCHANGED cookie still reached pickle.loads with attacker bytes; "
             "probe_status=%s -> reset (%s)" % (st, "NEW sid" if newsid != s else "same"))
        line("=> HMAC protects the sid pointer only; payload integrity is never verified "
             "before pickle.loads. Forging the cookie is unnecessary when Redis is writable; "
             "an attacker who CANNOT forge the cookie can still reach loads by writing the "
             "existing sid's key (see MAL-1/MAL-2).")

        # ---------------- BLOCK 4.8 : ownership-safe cleanup + foreign sentinel (F2) ----------------
        hr("BLOCK 4.8 OWNERSHIP-SAFE CLEANUP (F2) + FOREIGN SENTINEL SURVIVAL")
        SENT = f"{SESSION_PREFIX}:qa-foreign-sentinel"
        r.set(SENT, b"foreign-do-not-delete")
        all_before = sorted(k.decode() for k in r.keys(f"{SESSION_PREFIX}:*"))
        line("total session:* keys before cleanup = %d" % len(all_before))
        line("this run OWNS %d sids (tracked, incl. rotations); foreign sentinel present? %s"
             % (len(CREATED_SIDS), r.exists(SENT) == 1))
        deleted = 0
        for sid_ in CREATED_SIDS:
            deleted += r.delete(key(sid_))
        line("ownership-safe deletion removed %d keys (ONLY this run's tracked sids; "
             "NO blanket keys('session:*'))" % deleted)
        line("foreign sentinel %s still exists AFTER cleanup? %s" % (SENT, r.exists(SENT) == 1))
        remaining_sess = [k.decode() for k in r.keys(f"{SESSION_PREFIX}:*")]
        line("remaining session:* keys after ownership-safe cleanup = %r" % remaining_sess)
        line("CONTRAST: a blanket keys('session:*')+delete would have deleted %d keys "
             "INCLUDING the foreign sentinel." % len(all_before))
        r.delete(SENT)
        line("test sentinel removed by owner; our tracked keys remaining = %d"
             % sum(r.exists(key(x)) for x in CREATED_SIDS))
        line("\nALL BLOCKS COMPLETE.")
    finally:
        try: transaction.rollback()
        except Exception: pass
        try: Session.rollback(); Session.close()
        except Exception: pass
```

> **Safety note.** In blocks 4.6/4.7 each malicious pickle's *only* side effect is writing a benign marker file (`/tmp/blitzy_obs_rce_mal1.txt` / `_mal2.txt`); no destructive command is executed. The markers are removed after the run, and the demonstrated RCE is contingent on Redis **write** access — it is not reachable by a normal web client.

---

## Appendix B — Cleanup, disposability, and read-only-scope confirmation

Per the read-only scope, the repository working tree was left unchanged except for this one document, and every transient artifact this investigation created was removed — concretely: the disposable container's PostgreSQL rows and Redis keys, the `/tmp/obs_harness.py` copy, and the benign RCE marker files (each enumerated and demonstrated below). Unlike a native run (whose PostgreSQL/Redis are shared and long-lived), the **authoritative** observations here were produced inside a **disposable, no-bind-mount container**, so the definitive cleanup is simply to **destroy the container** — which discards every DB row and Redis key it ever held, leaving nothing to clean up piecemeal.

### B.1 The authoritative container has no host bind-mount (state is container-internal)

```text
$ docker inspect slinv_probe --format '{{json .Mounts}}'
[]
```

No bind-mount means nothing the harness wrote (PostgreSQL rows, Redis keys, `/tmp/obs_harness.py`, marker files) touches the host or the repository; it all lives inside the container's own writable layer.

### B.2 Destroying the container removes all DB rows and Redis keys (evidence)

A throwaway container started from the same image was given a sentinel Redis key, then destroyed; a fresh container from the same image does not have it — proving container-internal state does not survive `docker rm -f`:

```text
sentinel in throwaway BEFORE destroy: 1
$ docker inspect slinv_dispose --format '{{json .Mounts}}'   # also no mount
[]
$ docker rm -f slinv_dispose
destroyed.
$ docker ps -a --filter name=slinv_dispose   # empty
sentinel in fresh container AFTER destroy: 0
=> Container-internal DB/Redis state does NOT survive docker rm -f; destroying the
   no-mount container removes ALL rows and keys. This is the authoritative cleanup for F3.
```

Accordingly, the authoritative-run cleanup is `docker rm -f slinv_probe` (§1.3), which discards the container's PostgreSQL (including any `users` row persisted by the `intro_shown` commit noted in Appendix A) **and** its entire Redis keyspace (including any rotated `session:*` keys and any orphaned corrupted key left at TTL `-1`, e.g. the malformed/truncated/empty keys in §3/§5). No test rows or session keys are left behind on any host. The `/tmp/obs_harness.py` copy and the benign marker files are removed as well.

### B.3 In-run ownership-safe Redis cleanup (defense in depth)

Independently of container disposal, the harness itself performs ownership-safe Redis cleanup (Appendix A, block 4.8): it deletes only the `session:*` keys it created (tracked in `CREATED_SIDS`) and never issues a blanket `keys('session:*')` wipe, so a co-tenant's session keys — proven with the surviving `session:qa-foreign-sentinel` — are untouched. On the native cross-check host this ownership discipline is what keeps other agents' keys intact; in the container it is belt-and-suspenders ahead of the container's destruction.

### B.4 Read-only-scope confirmation

The repository working tree differs **only** in this one document. The following is the **actual, unedited** `git` output (not an "expected" comment):

```text
$ git status --porcelain
 M blitzy/documentation/app_2cd6ee777f8c.md

$ git diff --name-status 2cd6ee777f8c2d3531559588bcfb18627ffb5d2c HEAD
A	blitzy/documentation/app_2cd6ee777f8c.md
```

Read these two commands with their different reference points in mind: `git status --porcelain` compares the working tree to the **current destination-branch HEAD** (the tip of `blitzy-69b57f9c-a60d-44e2-934e-aba188d82f39`, which already carries a prior copy of this document), so it reports this document as modified (` M`) while a remediation is uncommitted, and a clean tree once committed. `git diff --name-status` against the **source-branch commit under investigation** (the absolute hash `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`) shows the sole change is the **addition** (`A`) of this document. No source, test, configuration, manifest, CI, Docker, or dependency file appears in either output.

**Read-only discipline confirmed:** no existing source file was modified, created, or deleted; no dependency was added, updated, or removed; the only artifact added to the repository is `blitzy/documentation/app_2cd6ee777f8c.md`. The source-branch commit under investigation (`2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`) is unchanged by this work; the disposable container that produced the evidence was destroyed at completion.
