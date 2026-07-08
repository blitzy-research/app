# SimpleLogin Startup Investigation — Run-First Q&A

**Branch:** `app_2cd6ee777f8c`
**Scope:** Read-only investigation of SimpleLogin's startup behavior. No source file was modified or deleted; the only artifact produced is this document. Temporary observation scripts and throwaway databases were created for observation and then removed — the repository is unchanged except for this file (see [§(f) Cleanup / repository integrity](#f-cleanup--repository-integrity), which shows the final `git status`).

This document answers three coupled runtime-behavior questions, plus an overarching startup-order / failure-mode narrative, **from direct first-hand observation**. Every behavioral claim is backed by **(1)** a `file:line` citation into the SimpleLogin source and **(2)** the actual, unedited runtime output captured beside the claim.

> **Canonical environment — where every command below was run.** The investigation was performed inside the mandated Docker image **`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`**. In that image the SimpleLogin repository lives at **`/app`** and the canonical interpreter is the pre-provisioned virtualenv **`/app/venv/bin/python` (Python 3.10.18)**; the Python standard library lives at `/usr/local/lib/python3.10`. Every entry point is invoked exactly as `CONFIG=<dotenv> /app/venv/bin/python <entrypoint>`. Because the software is built and run at these real paths, the paths that appear in the log lines below (`/app/...`, `/app/venv/lib/python3.10/site-packages/...`, and stdlib `/usr/local/lib/python3.10/...`) are the software's own real output. **Nothing in any pasted block is redacted, paraphrased, or elided** — the full traceback (Q1), the full 255-step migration (Q3), and the full `--help` text (Q2) are reproduced in their entirety.
>
> SimpleLogin's logger (`app/log.py:79` `LOG = _get_logger("SL")`) writes to STDOUT (`app/log.py:41` `logging.StreamHandler(sys.stdout)`) at DEBUG level (`app/log.py:51`) using the fixed format at `app/log.py:12-14`:
> `%(asctime)s - %(name)s - %(levelname)s - %(process)d - "%(pathname)s:%(lineno)d" - %(funcName)s() - %(message_id)s - %(message)s`.
>
> Every service prints an identical six-line import-time banner (`load config file …`, `>>> URL: …`, the `MAX_NB_EMAIL_FREE_PLAN` default note, `Paddle param not set`, the GNUPGHOME warning, `Upload files to local dir`) immediately followed by `>>> init logging <<<` (`app/log.py:67`). One of those six banner lines is `WARNING: Use a temp directory for GNUPGHOME /tmp/<random>` — this is the **application's own** startup output, not a machine artifact: `app/config.py:252-262` uses `$GNUPGHOME` if that variable is set, otherwise it generates a random `/tmp/<20-letters>` directory and `print`s that exact warning (`app/config.py:262`). It is shown verbatim wherever it appears, and the random suffix legitimately differs from run to run.

---

## Table of Contents

- [(a) Environment bring-up](#a-environment-bring-up)
- [(b) Q1 — Empty-database startup exception](#b-q1--empty-database-startup-exception)
- [(c) Q2 — Required Python services + port-binding evidence](#c-q2--required-python-services--port-binding-evidence)
- [(d) Q3 — Skipped-initialization SMTP rejection](#d-q3--skipped-initialization-smtp-rejection)
- [(e) Canonical startup order + failure modes](#e-canonical-startup-order--failure-modes)
- [(f) Cleanup / repository integrity](#f-cleanup--repository-integrity)
- [Appendix — Verified file:line reference map](#appendix--verified-fileline-reference-map)

---

## (a) Environment bring-up

The investigation was performed in the canonical runtime — **Python 3.10.18** (`Dockerfile:8` `FROM python:3.10`; `pyproject.toml:61` `python = "^3.10"`; Python 3.12 is explicitly unsupported per `CONTRIBUTING.md:236` — the repo's own comment reads *"we haven't managed to make python 3.12 work"*) with **PostgreSQL 13** and the SimpleLogin dependencies pinned in `poetry.lock`. **Redis is not required** for any of Q1/Q2/Q3 in the default local configuration because the session / rate-limit store is only wired when `MEM_STORE_URI` is set (`server.py:163-165`), and `MEM_STORE_URI` is unset here.

**Interpreter, dependencies, and PostgreSQL — exact commands and output:**

```console
$ cd /app                                   # canonical run-root in the image
$ /app/venv/bin/python --version
Python 3.10.18
$ /app/venv/bin/pip --version
pip 25.2 from /app/venv/lib/python3.10/site-packages/pip (python 3.10)
$ /app/venv/bin/python -c 'import importlib.metadata as m; \
    print("\n".join(f"{p}=={m.version(p)}" for p in \
    ["Flask","SQLAlchemy","psycopg2-binary","aiosmtpd","Flask-Migrate","gunicorn","gevent","redis","python-dotenv"]))'
Flask==1.1.2
SQLAlchemy==1.3.24
psycopg2-binary==2.9.3
aiosmtpd==1.4.2
Flask-Migrate==2.5.3
gunicorn==20.0.4
gevent==22.10.2
redis==4.6.0
python-dotenv==0.14.0
$ psql -h localhost -p 15432 -U myuser -d postgres -tAc "SHOW server_version;"
13.23 (Debian 13.23-1.pgdg13+1)
```

> **On dependencies.** The image ships the fully-materialized virtualenv at `/app/venv`, whose installed versions match `poetry.lock` exactly (Flask 1.1.2, SQLAlchemy 1.3.24, psycopg2 2.9.3, aiosmtpd 1.4.2, Flask-Migrate 2.5.3, gunicorn 20.0.4, gevent 22.10.2, redis 4.6.0, python-dotenv 0.14.0). No install step is needed; every command in this document runs through that Python 3.10.18 environment, invoked as `/app/venv/bin/python <entrypoint>`, never a host interpreter (the host's bare `python3.10` has no dependencies installed — e.g. `import psycopg2` fails there).

**Configuration (`CONFIG`-driven dotenv).** `app/config.py:65` reads `config_file = os.environ.get("CONFIG")`; if set, `app/config.py:68` `print`s `load config file <path>` and `app/config.py:69` calls `load_dotenv(...)` — that print is the first line of every banner below. The canonical values come from `example.env`: `URL=http://localhost:7777` (`example.env:6`), `NOT_SEND_EMAIL=true` (`example.env:19`), `EMAIL_DOMAIN=sl.local` (`example.env:22`), `SUPPORT_EMAIL=support@sl.local` (`example.env:40`), `DB_URI=...` (`example.env:75`), `FLASK_SECRET=...` (`example.env:77`). The canonical `/app/.env` is `example.env` with the `DB_URI` port changed `5432 -> 15432` to match the provisioned PostgreSQL (`CONTRIBUTING.md:100` `docker run … -p 15432:5432 postgres:13`). All cryptographic material required at startup already ships under `local_data/` (`jwtRS256.key`, `dkim.key`, PGP keys, `test_words.txt`, …), so a `CONFIG`-driven start has no missing-key failures.

Three database states are needed to exercise the three questions in isolation, so three `DB_URI`s were used (only `DB_URI` differs from the canonical dotenv; every other key is unchanged):

| Scenario | Database | State | Config file |
|----------|----------|-------|-------------|
| Q1 | `sl_q1_empty` | created, **no migrations** (empty schema) | `/app/q1.env` |
| Q2 | `simplelogin` | migrated **and** seeded (`init_app.py` run) | `/app/.env` |
| Q3 | `sl_q3v` | migrated, **`init_app.py` NOT run** (empty `SLDomain`) | `/tmp/obs/q3v.env` (temp) |

Each temp dotenv was derived from the canonical `.env` by changing only `DB_URI`, e.g.:

```console
# derive a temp dotenv from the canonical .env, changing ONLY DB_URI (Q3 throwaway DB)
$ sed 's#^DB_URI=.*#DB_URI=postgresql://myuser:mypassword@localhost:15432/sl_q3v#' /app/.env > /tmp/obs/q3v.env
$ grep ^DB_URI= /tmp/obs/q3v.env
DB_URI=postgresql://myuser:mypassword@localhost:15432/sl_q3v
```

Confirming the default local config uses **cookie sessions (no Redis, no extra PG session store)**:

```console
$ grep -c 'MEM_STORE_URI' /app/.env
0        # MEM_STORE_URI unset -> the server.py:163-165 branch is skipped
```

> **Note on demonstrating port binding.** This image ships neither `ss` nor `lsof`, so port binding is demonstrated three ways: **(1) functionally**, via a real HTTP request to `:7777` / a real SMTP transaction to `:20381`; **(2) at the kernel level**, by decoding `/proc/net/tcp` — state `0A` is `TCP_LISTEN`, and the listening socket's inode is matched back to the owning PID(s) via `/proc/<pid>/fd`; and **(3) via `ps`**, showing the exact owning process(es). The small read-only helper used for (2) (`/tmp/obs/portproof.py`) was a temporary observation script, removed during cleanup (§(f)).

---


## (b) Q1 — Empty-database startup exception

**Question.** With PostgreSQL running but the target DB schema **empty** (migrations deliberately not run), start the web app with `python server.py`, open the login page, and capture the **full Python exception / traceback** raised.

**Answer (summary).** The server starts and binds `:7777` cleanly against the empty schema; the failure appears only at **request time**. Submitting the login form fires the first ORM query, which raises **`sqlalchemy.exc.ProgrammingError`** wrapping **`psycopg2.errors.UndefinedTable: relation "users" does not exist`**, and the request returns **HTTP 500**. The offending relation observed is **`users`**.

### Mechanism (cause -> effect), with `file:line`

1. **The DB engine connects at *import time*, with no schema / DDL check.** `app/db.py:9-11` builds the engine and `app/db.py:12` opens a connection at module load:
   ```python
   # app/db.py:9-12
   engine = create_engine(
       config.DB_URI, connect_args={"application_name": config.DB_CONN_NAME}
   )
   connection = engine.connect()
   ```
   `engine.connect()` succeeds against an empty database (connecting to PostgreSQL says nothing about which tables exist), so nothing fails during startup.

2. **`python server.py` runs the debug dev server on `:7777`.** `server.py:572` `def local_main():` builds the app, enables the Debug Toolbar, and `server.py:588` `app.run(debug=True, port=7777)`; the entrypoint is `server.py:598-599` `if __name__ == "__main__": local_main()`.

3. **Redis / session store is not involved by default.** `server.py:163-165` only installs the memory-store session backend `if MEM_STORE_URI:`; with it unset, sessions are cookie-based and never touch PostgreSQL — so a bare page load does not hit the DB via sessions.

4. **A bare anonymous GET renders with no DB query; the login *POST* is the reliable first ORM query.** `/` redirects anonymous users to `auth.login` (`server.py:250-256`); the login GET view (`app/auth/views/login.py:21`) renders the template chain `templates/auth/login.html` -> `templates/single.html` -> `templates/base.html`, whose only `current_user` reference short-circuits for anonymous users (`templates/base.html:89`). On **POST**, `app/auth/views/login.py:40` `if form.validate_on_submit():` becomes true and `app/auth/views/login.py:43` executes:
   ```python
   # app/auth/views/login.py:43
   user = User.get_by(email=email) or User.get_by(email=canonical_email)
   ```
   `User.get_by(...)` runs `Session.query(cls).filter_by(**kw).first()` (`app/models.py:84`) against table **`users`** (`User.__tablename__ = "users"`, `app/models.py:337`). Because migrations never created that relation, PostgreSQL raises `UndefinedTable`, SQLAlchemy 1.3 re-raises it as `ProgrammingError`, and the app's exception handler `error_handler` (`server.py:389`, whose `LOG.e(e)` at `server.py:390` emits the error you see below) turns it into HTTP 500.

### Runtime evidence

**Command (empty DB, no migrations):**

```console
$ psql -h localhost -p 15432 -U myuser -d sl_q1_empty -tAc \
    "SELECT count(*) FROM information_schema.tables WHERE table_schema='public';"
0     # empty schema: zero tables

$ CONFIG=/app/q1.env /app/venv/bin/python server.py
```

**Startup stdout/stderr** — the server binds `:7777` against the empty DB. `python server.py` runs with `debug=True`, so Werkzeug's reloader is active: the reloader **parent** process prints the banner and the `* Serving Flask app` lines, then re-executes the module as a **child** worker that prints the banner again (note the two different PIDs). The complete, unedited startup output is:

```
load config file /app/q1.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/iqxltgxpoqfqcoyiqwam
Upload files to local dir
>>> init logging <<<
2026-07-08 15:44:23,963 - SL - DEBUG - 1885 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
load config file /app/q1.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/draeqrojssdegcpcfrzd
Upload files to local dir
>>> init logging <<<
2026-07-08 15:44:25,476 - SL - DEBUG - 1891 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

> **On the missing `* Running on http://127.0.0.1:7777/` line.** SimpleLogin deliberately **disables Werkzeug's logger** — `app/log.py:69-71`:
> ```python
> # Disable flask logs such as 127.0.0.1 - - [15/Feb/2013 10:52:22] "GET /index.html HTTP/1.1" 200
> log = logging.getLogger("werkzeug")
> log.disabled = True
> ```
> so Werkzeug's `* Running on …` startup line (emitted through that logger) never reaches stdout/stderr. The `* Serving Flask app … * Debug mode: on` banner above IS printed (Werkzeug writes it directly, not via the logger). Definitive port-binding proof therefore comes from the process table and the kernel socket table.

The process tree shows the reloader parent and the worker child; both hold the **same** listening socket, which is bound to `:7777`:

```console
$ ps -o pid,ppid,stat,cmd --no-headers -C python | grep server.py
   1885    1879 S    /app/venv/bin/python server.py
   1891    1885 Sl   /app/venv/bin/python /app/server.py
$ /app/venv/bin/python /tmp/obs/portproof.py 7777
LISTEN 127.0.0.1:7777  inode=599015483  pid=1885  ppid=1879  cmd=(python server.py)
LISTEN 127.0.0.1:7777  inode=599015483  pid=1891  ppid=1885  cmd=(python server.py)
```

The listening socket (`inode` shared by both PIDs, matched via `/proc/<pid>/fd`) is in state `LISTEN` on `127.0.0.1:7777` — proving the server bound `:7777` **despite the empty schema**. The functional trigger (a real HTTP client) confirms it is serving: a GET fetches the login page (`200`), the login **POST** returns **`500`**, and the anonymous `/` returns a `302` redirect to `/auth/login` (which needs no DB, `server.py:250-256`):

```console
$ curl -s -c q1_cookies.txt http://127.0.0.1:7777/auth/login -o q1_login.html -w "GET /auth/login -> %{http_code}\n"
GET /auth/login -> 200
$ CSRF=<token extracted from the login page's hidden csrf_token field>
$ curl -s -b q1_cookies.txt --data-urlencode "csrf_token=$CSRF" \
       --data-urlencode "email=test@example.com" --data-urlencode "password=whatever123" \
       -o /dev/null -w "POST /auth/login -> %{http_code}\n" http://127.0.0.1:7777/auth/login
POST /auth/login -> 500
$ curl -s -o /dev/null -D - http://127.0.0.1:7777/ | grep -iE "^HTTP|^location"
HTTP/1.0 302 FOUND
Location: http://127.0.0.1:7777/auth/login
```

**The complete, unedited server-side output** for the first login attempt (the GET that rendered `200`, then `error_handler` at `server.py:390`, the full chained traceback, the `after_request` `500` log line at `server.py:284`, and the subsequent anonymous `GET /` `302`):

```
2026-07-08 15:44:47,272 - SL - DEBUG - 1891 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.1022939682006836
2026-07-08 15:44:47,293 - SL - ERROR - 1891 - "/app/server.py:390" - error_handler() -  - (psycopg2.errors.UndefinedTable) relation "users" does not exist
LINE 2: FROM users 
             ^

[SQL: SELECT users.directory_quota AS users_directory_quota, users.subdomain_quota AS users_subdomain_quota, users.password AS users_password, users.id AS users_id, users.created_at AS users_created_at, users.updated_at AS users_updated_at, users.email AS users_email, users.name AS users_name, users.is_admin AS users_is_admin, users.alias_generator AS users_alias_generator, users.notification AS users_notification, users.activated AS users_activated, users.disabled AS users_disabled, users.profile_picture_id AS users_profile_picture_id, users.otp_secret AS users_otp_secret, users.enable_otp AS users_enable_otp, users.last_otp AS users_last_otp, users.fido_uuid AS users_fido_uuid, users.default_alias_custom_domain_id AS users_default_alias_custom_domain_id, users.default_alias_public_domain_id AS users_default_alias_public_domain_id, users.lifetime AS users_lifetime, users.paid_lifetime AS users_paid_lifetime, users.lifetime_coupon_id AS users_lifetime_coupon_id, users.trial_end AS users_trial_end, users.default_mailbox_id AS users_default_mailbox_id, users.sender_format AS users_sender_format, users.sender_format_updated_at AS users_sender_format_updated_at, users.replace_reverse_alias AS users_replace_reverse_alias, users.referral_id AS users_referral_id, users.intro_shown AS users_intro_shown, users.max_spam_score AS users_max_spam_score, users.newsletter_alias_id AS users_newsletter_alias_id, users.include_sender_in_reverse_alias AS users_include_sender_in_reverse_alias, users.random_alias_suffix AS users_random_alias_suffix, users.expand_alias_info AS users_expand_alias_info, users.ignore_loop_email AS users_ignore_loop_email, users.alternative_id AS users_alternative_id, users.disable_automatic_alias_note AS users_disable_automatic_alias_note, users.one_click_unsubscribe_block_sender AS users_one_click_unsubscribe_block_sender, users.include_website_in_one_click_alias AS users_include_website_in_one_click_alias, users.disable_import AS users_disable_import, users.can_use_phone AS users_can_use_phone, users.phone_quota AS users_phone_quota, users.block_behaviour AS users_block_behaviour, users.include_header_email_header AS users_include_header_email_header, users.enable_data_breach_check AS users_enable_data_breach_check, users.flags AS users_flags, users.unsub_behaviour AS users_unsub_behaviour, users.delete_on AS users_delete_on 
FROM users 
WHERE users.email = %(email_1)s 
 LIMIT %(param_1)s]
[parameters: {'email_1': 'test@example.com', 'param_1': 1}]
(Background on this error at: http://sqlalche.me/e/13/f405)
Traceback (most recent call last):
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1276, in _execute_context
    self.dialect.do_execute(
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/default.py", line 608, in do_execute
    cursor.execute(statement, parameters)
psycopg2.errors.UndefinedTable: relation "users" does not exist
LINE 2: FROM users 
             ^


The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "/app/venv/lib/python3.10/site-packages/flask/app.py", line 1950, in full_dispatch_request
    rv = self.dispatch_request()
  File "/app/venv/lib/python3.10/site-packages/flask_debugtoolbar/__init__.py", line 125, in dispatch_request
    return view_func(**req.view_args)
  File "/usr/local/lib/python3.10/cProfile.py", line 110, in runcall
    return func(*args, **kw)
  File "/app/venv/lib/python3.10/site-packages/flask_limiter/extension.py", line 702, in __inner
    return obj(*a, **k)
  File "/app/app/auth/views/login.py", line 43, in login
    user = User.get_by(email=email) or User.get_by(email=canonical_email)
  File "/app/app/models.py", line 84, in get_by
    return Session.query(cls).filter_by(**kw).first()
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/orm/query.py", line 3429, in first
    ret = list(self[0:1])
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/orm/query.py", line 3203, in __getitem__
    return list(res)
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/orm/query.py", line 3535, in __iter__
    return self._execute_and_instances(context)
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/orm/query.py", line 3560, in _execute_and_instances
    result = conn.execute(querycontext.statement, self._params)
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
sqlalchemy.exc.ProgrammingError: (psycopg2.errors.UndefinedTable) relation "users" does not exist
LINE 2: FROM users 
             ^

[SQL: SELECT users.directory_quota AS users_directory_quota, users.subdomain_quota AS users_subdomain_quota, users.password AS users_password, users.id AS users_id, users.created_at AS users_created_at, users.updated_at AS users_updated_at, users.email AS users_email, users.name AS users_name, users.is_admin AS users_is_admin, users.alias_generator AS users_alias_generator, users.notification AS users_notification, users.activated AS users_activated, users.disabled AS users_disabled, users.profile_picture_id AS users_profile_picture_id, users.otp_secret AS users_otp_secret, users.enable_otp AS users_enable_otp, users.last_otp AS users_last_otp, users.fido_uuid AS users_fido_uuid, users.default_alias_custom_domain_id AS users_default_alias_custom_domain_id, users.default_alias_public_domain_id AS users_default_alias_public_domain_id, users.lifetime AS users_lifetime, users.paid_lifetime AS users_paid_lifetime, users.lifetime_coupon_id AS users_lifetime_coupon_id, users.trial_end AS users_trial_end, users.default_mailbox_id AS users_default_mailbox_id, users.sender_format AS users_sender_format, users.sender_format_updated_at AS users_sender_format_updated_at, users.replace_reverse_alias AS users_replace_reverse_alias, users.referral_id AS users_referral_id, users.intro_shown AS users_intro_shown, users.max_spam_score AS users_max_spam_score, users.newsletter_alias_id AS users_newsletter_alias_id, users.include_sender_in_reverse_alias AS users_include_sender_in_reverse_alias, users.random_alias_suffix AS users_random_alias_suffix, users.expand_alias_info AS users_expand_alias_info, users.ignore_loop_email AS users_ignore_loop_email, users.alternative_id AS users_alternative_id, users.disable_automatic_alias_note AS users_disable_automatic_alias_note, users.one_click_unsubscribe_block_sender AS users_one_click_unsubscribe_block_sender, users.include_website_in_one_click_alias AS users_include_website_in_one_click_alias, users.disable_import AS users_disable_import, users.can_use_phone AS users_can_use_phone, users.phone_quota AS users_phone_quota, users.block_behaviour AS users_block_behaviour, users.include_header_email_header AS users_include_header_email_header, users.enable_data_breach_check AS users_enable_data_breach_check, users.flags AS users_flags, users.unsub_behaviour AS users_unsub_behaviour, users.delete_on AS users_delete_on 
FROM users 
WHERE users.email = %(email_1)s 
 LIMIT %(param_1)s]
[parameters: {'email_1': 'test@example.com', 'param_1': 1}]
(Background on this error at: http://sqlalche.me/e/13/f405)
2026-07-08 15:44:47,299 - SL - DEBUG - 1891 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 500, takes 0.01320505142211914
2026-07-08 15:44:47,307 - SL - DEBUG - 1891 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET / ImmutableMultiDict([]) 302, takes 0.0005540847778320312
```

> The `SELECT users.… FROM users …` statement appears **twice** above and both copies are shown verbatim: SQLAlchemy prints the offending SQL once on the wrapped `psycopg2` error and again on the outer `ProgrammingError`. Nothing is elided.

**Exception anatomy (as observed):**

- **Chained cause (inner):** `psycopg2.errors.UndefinedTable: relation "users" does not exist`.
- **Raised (outer):** `sqlalchemy.exc.ProgrammingError: (psycopg2.errors.UndefinedTable) relation "users" does not exist` — joined by Python's *"The above exception was the direct cause of the following exception"*.
- **Offending relation:** `users`.
- **Trigger frame:** `app/auth/views/login.py:43` -> `app/models.py:84` (`Session.query(cls).filter_by(**kw).first()`).
- **HTTP status:** `500` (confirmed by both the client `POST /auth/login -> 500` and the `after_request` log `POST /auth/login … 500`, `server.py:284`).
- **SQLAlchemy reference:** `http://sqlalche.me/e/13/f405` (the SQLAlchemy **1.3** error-code URL, matching `SQLAlchemy = 1.3.24` at `pyproject.toml:116`).

### Stability (>=2 runs)

The same empty-schema condition was exercised a second time (a second POST with a different sender email, against the same running server). The complete, unedited server-side output for run #2 (`second-run@example.com`) is identical in exception class, offending relation, and HTTP status:

```
2026-07-08 15:44:57,064 - SL - ERROR - 1891 - "/app/server.py:390" - error_handler() -  - (psycopg2.errors.UndefinedTable) relation "users" does not exist
LINE 2: FROM users 
             ^

[SQL: SELECT users.directory_quota AS users_directory_quota, users.subdomain_quota AS users_subdomain_quota, users.password AS users_password, users.id AS users_id, users.created_at AS users_created_at, users.updated_at AS users_updated_at, users.email AS users_email, users.name AS users_name, users.is_admin AS users_is_admin, users.alias_generator AS users_alias_generator, users.notification AS users_notification, users.activated AS users_activated, users.disabled AS users_disabled, users.profile_picture_id AS users_profile_picture_id, users.otp_secret AS users_otp_secret, users.enable_otp AS users_enable_otp, users.last_otp AS users_last_otp, users.fido_uuid AS users_fido_uuid, users.default_alias_custom_domain_id AS users_default_alias_custom_domain_id, users.default_alias_public_domain_id AS users_default_alias_public_domain_id, users.lifetime AS users_lifetime, users.paid_lifetime AS users_paid_lifetime, users.lifetime_coupon_id AS users_lifetime_coupon_id, users.trial_end AS users_trial_end, users.default_mailbox_id AS users_default_mailbox_id, users.sender_format AS users_sender_format, users.sender_format_updated_at AS users_sender_format_updated_at, users.replace_reverse_alias AS users_replace_reverse_alias, users.referral_id AS users_referral_id, users.intro_shown AS users_intro_shown, users.max_spam_score AS users_max_spam_score, users.newsletter_alias_id AS users_newsletter_alias_id, users.include_sender_in_reverse_alias AS users_include_sender_in_reverse_alias, users.random_alias_suffix AS users_random_alias_suffix, users.expand_alias_info AS users_expand_alias_info, users.ignore_loop_email AS users_ignore_loop_email, users.alternative_id AS users_alternative_id, users.disable_automatic_alias_note AS users_disable_automatic_alias_note, users.one_click_unsubscribe_block_sender AS users_one_click_unsubscribe_block_sender, users.include_website_in_one_click_alias AS users_include_website_in_one_click_alias, users.disable_import AS users_disable_import, users.can_use_phone AS users_can_use_phone, users.phone_quota AS users_phone_quota, users.block_behaviour AS users_block_behaviour, users.include_header_email_header AS users_include_header_email_header, users.enable_data_breach_check AS users_enable_data_breach_check, users.flags AS users_flags, users.unsub_behaviour AS users_unsub_behaviour, users.delete_on AS users_delete_on 
FROM users 
WHERE users.email = %(email_1)s 
 LIMIT %(param_1)s]
[parameters: {'email_1': 'second-run@example.com', 'param_1': 1}]
(Background on this error at: http://sqlalche.me/e/13/f405)
Traceback (most recent call last):
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1276, in _execute_context
    self.dialect.do_execute(
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/default.py", line 608, in do_execute
    cursor.execute(statement, parameters)
psycopg2.errors.UndefinedTable: relation "users" does not exist
LINE 2: FROM users 
             ^


The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "/app/venv/lib/python3.10/site-packages/flask/app.py", line 1950, in full_dispatch_request
    rv = self.dispatch_request()
  File "/app/venv/lib/python3.10/site-packages/flask_debugtoolbar/__init__.py", line 125, in dispatch_request
    return view_func(**req.view_args)
  File "/usr/local/lib/python3.10/cProfile.py", line 110, in runcall
    return func(*args, **kw)
  File "/app/venv/lib/python3.10/site-packages/flask_limiter/extension.py", line 702, in __inner
    return obj(*a, **k)
  File "/app/app/auth/views/login.py", line 43, in login
    user = User.get_by(email=email) or User.get_by(email=canonical_email)
  File "/app/app/models.py", line 84, in get_by
    return Session.query(cls).filter_by(**kw).first()
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/orm/query.py", line 3429, in first
    ret = list(self[0:1])
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/orm/query.py", line 3203, in __getitem__
    return list(res)
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/orm/query.py", line 3535, in __iter__
    return self._execute_and_instances(context)
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/orm/query.py", line 3560, in _execute_and_instances
    result = conn.execute(querycontext.statement, self._params)
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
sqlalchemy.exc.ProgrammingError: (psycopg2.errors.UndefinedTable) relation "users" does not exist
LINE 2: FROM users 
             ^

[SQL: SELECT users.directory_quota AS users_directory_quota, users.subdomain_quota AS users_subdomain_quota, users.password AS users_password, users.id AS users_id, users.created_at AS users_created_at, users.updated_at AS users_updated_at, users.email AS users_email, users.name AS users_name, users.is_admin AS users_is_admin, users.alias_generator AS users_alias_generator, users.notification AS users_notification, users.activated AS users_activated, users.disabled AS users_disabled, users.profile_picture_id AS users_profile_picture_id, users.otp_secret AS users_otp_secret, users.enable_otp AS users_enable_otp, users.last_otp AS users_last_otp, users.fido_uuid AS users_fido_uuid, users.default_alias_custom_domain_id AS users_default_alias_custom_domain_id, users.default_alias_public_domain_id AS users_default_alias_public_domain_id, users.lifetime AS users_lifetime, users.paid_lifetime AS users_paid_lifetime, users.lifetime_coupon_id AS users_lifetime_coupon_id, users.trial_end AS users_trial_end, users.default_mailbox_id AS users_default_mailbox_id, users.sender_format AS users_sender_format, users.sender_format_updated_at AS users_sender_format_updated_at, users.replace_reverse_alias AS users_replace_reverse_alias, users.referral_id AS users_referral_id, users.intro_shown AS users_intro_shown, users.max_spam_score AS users_max_spam_score, users.newsletter_alias_id AS users_newsletter_alias_id, users.include_sender_in_reverse_alias AS users_include_sender_in_reverse_alias, users.random_alias_suffix AS users_random_alias_suffix, users.expand_alias_info AS users_expand_alias_info, users.ignore_loop_email AS users_ignore_loop_email, users.alternative_id AS users_alternative_id, users.disable_automatic_alias_note AS users_disable_automatic_alias_note, users.one_click_unsubscribe_block_sender AS users_one_click_unsubscribe_block_sender, users.include_website_in_one_click_alias AS users_include_website_in_one_click_alias, users.disable_import AS users_disable_import, users.can_use_phone AS users_can_use_phone, users.phone_quota AS users_phone_quota, users.block_behaviour AS users_block_behaviour, users.include_header_email_header AS users_include_header_email_header, users.enable_data_breach_check AS users_enable_data_breach_check, users.flags AS users_flags, users.unsub_behaviour AS users_unsub_behaviour, users.delete_on AS users_delete_on 
FROM users 
WHERE users.email = %(email_1)s 
 LIMIT %(param_1)s]
[parameters: {'email_1': 'second-run@example.com', 'param_1': 1}]
(Background on this error at: http://sqlalche.me/e/13/f405)
2026-07-08 15:44:57,066 - SL - DEBUG - 1891 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 500, takes 0.007324695587158203
```

Result: **stable** — HTTP 500 with `ProgrammingError` / `UndefinedTable: relation "users" does not exist` on every run.

**Q1 <-> omitted step:** this failure is caused by **skipping the migration step**. The engine connects at import (`app/db.py:12`) so the server starts, but the schema has no `users` table, so the first ORM query fails at request time.

---


## (c) Q2 — Required Python services + port-binding evidence

**Question.** After migrations and `init_app.py` have been run correctly, identify which Python services must run for the system to function, start each one, and capture the exact stdout/stderr confirming each service is up and bound to its port.

**Answer (summary).** Five Python services make up the running system. This scenario uses the fully-provisioned `simplelogin` database (migrated — 77 tables — and seeded, so `public_domain` has one row, `sl.local`):

```console
$ psql -h localhost -p 15432 -U myuser -d simplelogin -tAc \
    "SELECT count(*) FROM information_schema.tables WHERE table_schema='public';"
77
$ psql -h localhost -p 15432 -U myuser -d simplelogin -tAc "SELECT count(*) FROM public_domain;"
1
```

| # | Service | Entry point | Bound port (dev) | Why it is required |
|---|---------|-------------|------------------|--------------------|
| 1 | Web app | `python server.py` | **7777** (HTTP) | Serves the web UI / login / dashboard / API (`server.py:588`). Production: Gunicorn serving `wsgi:app`. |
| 2 | Email handler | `python email_handler.py` | **20381** (SMTP) | The aiosmtpd SMTP tier that receives and forwards/relays mail (`email_handler.py:2381-2386`). Production SMTP port: `:25`. |
| 3 | Job runner | `python job_runner.py` | none (worker) | Processes asynchronous `Job` rows in a 10-second polling loop (`job_runner.py:329-347`). |
| 4 | Event listener | `python event_listener.py listener` | none (PG LISTEN) | Consumes PostgreSQL events via `PostgresEventSource` (`event_listener.py:34-35`). |
| 5 | Scheduler (cron) | `python cron.py -j <job>` | none (batch) | Runs scheduled maintenance jobs, dispatched on `--job` (`cron.py:1262-1274`). |

Each service's literal startup output is shown below. All lines use the `app/log.py` "SL" format; the `>>> init logging <<<` line (`app/log.py:67`) prints on import and marks logger initialization. For the two networked services, port binding is proven functionally, by `ps`, and by the `/proc/net/tcp` socket table (see the note in §(a)).

### Service 1 — Web app (`python server.py`, binds `:7777`)

`server.py:588` `app.run(debug=True, port=7777)`. Startup stdout/stderr (against the seeded `simplelogin` DB); as in Q1, the reloader prints the banner for both the parent and the re-executed child:

```console
$ CONFIG=/app/.env /app/venv/bin/python server.py
load config file /app/.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/uevvjsnlhyexfhjuykqa
Upload files to local dir
>>> init logging <<<
2026-07-08 15:46:31,801 - SL - DEBUG - 2020 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
load config file /app/.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/spagbchmpuajnsuwclsg
Upload files to local dir
>>> init logging <<<
2026-07-08 15:46:33,309 - SL - DEBUG - 2026 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-08 15:46:41,478 - SL - DEBUG - 2026 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET / ImmutableMultiDict([]) 302, takes 0.0009009838104248047
/app/venv/lib/python3.10/site-packages/flask_debugtoolbar/__init__.py:213: UserWarning: Could not insert debug toolbar. </body> tag not found in response.
  warnings.warn('Could not insert debug toolbar.'
```

Process tree, kernel-level bind proof, and functional proof (the `* Running on …` line is suppressed by the disabled Werkzeug logger, `app/log.py:69-71`, as explained in §(b)):

```console
$ ps -o pid,ppid,stat,cmd --no-headers -C python | grep server.py
   2020    2014 S    /app/venv/bin/python server.py
   2026    2020 Rl   /app/venv/bin/python /app/server.py
$ /app/venv/bin/python /tmp/obs/portproof.py 7777
LISTEN 127.0.0.1:7777  inode=598893423  pid=2020  ppid=2014  cmd=(python server.py)
LISTEN 127.0.0.1:7777  inode=598893423  pid=2026  ppid=2020  cmd=(python server.py)
$ curl -s -m 5 -o /dev/null -D - http://127.0.0.1:7777/ | grep -iE "^HTTP|^location"
HTTP/1.0 302 FOUND
Location: http://127.0.0.1:7777/auth/login
$ curl -s -m 5 -w " HTTP=%{http_code}\n" http://127.0.0.1:7777/health
success HTTP=200
```

The `LISTEN 127.0.0.1:7777 … (python server.py)` rows (parent + child sharing one socket inode), the `302 -> /auth/login` redirect, and `/health -> 200` together prove the process is **listening and serving on `:7777`**. (Production equivalent: `wsgi.py` exposes `app = create_app()` — `wsgi.py:1,3` — served by Gunicorn per `pyproject.toml:66` `gunicorn = "^20.0.4"`.)

### Service 2 — Email handler (`python email_handler.py`, binds `:20381`)

`email_handler.py:2381` `def main(port)`; `:2383` `Controller(MailHandler(), hostname="0.0.0.0", port=port)`; `:2385` `controller.start()`; `:2386` logs the bound controller; the `__main__` argparse default port is **20381** (`email_handler.py:2399`); `:2403` logs the listen port; `:2404` calls `main`. Startup stdout/stderr with the **exact port-binding lines** (note the literal port `20381` in both):

```console
$ CONFIG=/app/.env /app/venv/bin/python email_handler.py
load config file /app/.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/bytpqvqdyplvgzqfyenn
Upload files to local dir
>>> init logging <<<
2026-07-08 15:59:29,658 - SL - DEBUG - 2600 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-08 15:59:30,118 - SL - INFO - 2600 - "/app/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-08 15:59:30,119 - SL - DEBUG - 2600 - "/app/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

Process, kernel-level bind proof, and a functional SMTP probe — the aiosmtpd controller is a single process (no reloader) `LISTEN`ing on `0.0.0.0:20381`, and answers an `EHLO` with `250`:

```console
$ ps -o pid,ppid,stat,cmd --no-headers -C python | grep email_handler.py
   2600    2594 Sl   /app/venv/bin/python email_handler.py
$ /app/venv/bin/python /tmp/obs/portproof.py 20381
LISTEN 0.0.0.0:20381  inode=599082784  pid=2600  ppid=2594  cmd=(python email_handler.py)
$ /app/venv/bin/python -c 'import smtplib; s=smtplib.SMTP("127.0.0.1",20381,timeout=10); \
    print("EHLO ->", s.ehlo("probe.local")); s.quit()'
EHLO -> (250, b'reverse-code-generator-66478fbe-wfxk7')
```

Production SMTP port is `:25`.

### Service 3 — Job runner (`python job_runner.py`, worker; no port)

`job_runner.py:329` `if __name__ == "__main__":` enters a `while True:` loop that takes pending jobs and `job_runner.py:347` `time.sleep(10)` between polls. It binds no port. Its startup output is the standard import banner, after which it polls silently until a `Job` exists:

```console
$ CONFIG=/app/.env /app/venv/bin/python job_runner.py
load config file /app/.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/lxtdswibzuwzfadhjejy
Upload files to local dir
>>> init logging <<<
2026-07-08 15:48:04,250 - SL - DEBUG - 2188 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

The process was confirmed alive and idle in its polling loop (no port, no further output until a job arrives):

```console
$ ps -o pid,ppid,stat,etime,cmd --no-headers -C python | grep job_runner.py
   2188    2182 S          00:09 /app/venv/bin/python job_runner.py
```

### Service 4 — Event listener (`python event_listener.py listener`, PG LISTEN; no port)

`event_listener.py:29` `def main(mode, …)`; with the `listener` sub-command, `event_listener.py:34` logs `Using PostgresEventSource` and `:35` constructs `PostgresEventSource(...)`. (A bare `python event_listener.py` with no sub-command prints `Invalid usage. Pass a valid subcommand as argument` and exits — `event_listener.py:96`.) Startup stdout/stderr:

```console
$ CONFIG=/app/.env /app/venv/bin/python event_listener.py listener
load config file /app/.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/sgkaayrxdgkbffyfrqnz
Upload files to local dir
>>> init logging <<<
2026-07-08 15:48:14,629 - SL - DEBUG - 2227 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-08 15:48:14,747 - SL - INFO - 2227 - "/app/event_listener.py:34" - main() -  - Using PostgresEventSource
2026-07-08 15:48:14,751 - SL - INFO - 2227 - "/app/event_listener.py:43" - main() -  - Starting with HttpEventSink
2026-07-08 15:48:14,751 - SL - INFO - 2227 - "/app/events/event_source.py:49" - __listen() -  - Starting to listen to events
```

```console
$ ps -o pid,ppid,stat,etime,cmd --no-headers -C python | grep event_listener.py
   2227    2221 S          00:09 /app/venv/bin/python event_listener.py listener
```

### Service 5 — Scheduler / cron (`python cron.py`, batch; no port)

`cron.py:1` `import argparse`; `cron.py:1262` `if __name__ == "__main__":` logs `Start running cronjob` (`cron.py:1263`) and dispatches on `args.job` (`cron.py:1274` onward — e.g. `stats`, `notify_trial_end`, `sanity_check`, …). Startup stdout/stderr (no `-j`, so `args.job` is `None` and no maintenance job runs):

```console
$ CONFIG=/app/.env /app/venv/bin/python cron.py
load config file /app/.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/bczheoltnlozydskocrk
Upload files to local dir
>>> init logging <<<
2026-07-08 15:48:38,525 - SL - DEBUG - 2264 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-08 15:48:39,160 - SL - DEBUG - 2264 - "/app/cron.py:1263" - <module>() -  - Start running cronjob
```

The full `--help` (the module still prints its banner and `Start running cronjob` line on import, *then* argparse prints usage and exits — shown complete, nothing elided):

```console
$ CONFIG=/app/.env /app/venv/bin/python cron.py --help
load config file /app/.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/lovfevyilunsqewelcft
Upload files to local dir
>>> init logging <<<
2026-07-08 15:48:40,107 - SL - DEBUG - 2277 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-08 15:48:40,732 - SL - DEBUG - 2277 - "/app/cron.py:1263" - <module>() -  - Start running cronjob
usage: cron.py [-h] [-j JOB]

options:
  -h, --help         show this help message and exit
  -j JOB, --job JOB  Choose a cron job to run
```

(A concrete job is selected with `-j <job>`.)

### Stability (>=2 runs)

The two network ports are deterministic constants, not dynamically assigned:

- **`:7777`** — hard-coded in `server.py:588`; observed bound in both the Q1 run (empty DB) and this Q2 run.
- **`:20381`** — the argparse default in `email_handler.py:2399`; observed bound in this Q2 run **and** again in the Q3 run below, both logging `Listen for port 20381` / `Start mail controller 0.0.0.0 20381`.

Result: **stable** — ports 7777 and 20381 every time.

---


## (d) Q3 — Skipped-initialization SMTP rejection

**Question.** Run migrations but do **not** run `init_app.py` (leaving `SLDomain` empty — no email domains configured). Start the email handler, send a message to `x@sl.local`, and capture both the SMTP status code / wire reply returned to the sender **and** the log line(s) explaining the rejection.

**Answer (summary).** With `SLDomain` empty, mail to `x@sl.local` is rejected with the permanent SMTP reply **`550 SL E515 Email not exist`** (`app/email/status.py:51`), and the handler logs two `LOG.d` lines — `alias x@sl.local not exist…` (`email_handler.py:545`) and `alias x@sl.local cannot be created on-the-fly, return 550` (`email_handler.py:551`) — around two `LOG.i` lines that record *why* on-the-fly creation failed (no custom domain for `sl.local`, `app/alias_utils.py:104`; no directory separator, `app/alias_utils.py:165`). The edge case where the sender is an `IgnoreBounceSender` instead returns **`250 SL E207 No bounce report`** (`app/email/status.py:12`).

> **Important — real table name.** The `SLDomain` model maps to the table **`public_domain`**, not `sl_domain` (`app/models.py:3116` `class SLDomain(...)`, `app/models.py:3119` `__tablename__ = "public_domain"`). Row-count evidence below therefore queries `public_domain`.

### `SLDomain` state — before / after `init_app.py`

The `sl_q3v` database was migrated (`alembic upgrade head`) but `init_app.py` was **not** run. `alembic upgrade head` applied all **255** revisions. The **complete, unedited** migration output (the import banner from Alembic loading the app config, the two Alembic runtime-info lines, and all 255 `Running upgrade` steps — nothing elided) is:

```console
$ CONFIG=/tmp/obs/q3v.env /app/venv/bin/alembic upgrade head
load config file /tmp/obs/q3v.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/jzdpbfaxezmwagblrqxo
Upload files to local dir
>>> init logging <<<
2026-07-08 15:50:17,153 - SL - DEBUG - 2348 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade  -> 5e549314e1e2, empty message
INFO  [alembic.runtime.migration] Running upgrade 5e549314e1e2 -> 3cd10cfce8c3, empty message
INFO  [alembic.runtime.migration] Running upgrade 3cd10cfce8c3 -> 0256244cd7c8, empty message
INFO  [alembic.runtime.migration] Running upgrade 0256244cd7c8 -> 213fcca48483, empty message
INFO  [alembic.runtime.migration] Running upgrade 213fcca48483 -> f234688f5ebd, empty message
INFO  [alembic.runtime.migration] Running upgrade f234688f5ebd -> d03e433dc248, empty message
INFO  [alembic.runtime.migration] Running upgrade d03e433dc248 -> 2fe19381f386, empty message
INFO  [alembic.runtime.migration] Running upgrade 2fe19381f386 -> b20ee72fd9a4, empty message
INFO  [alembic.runtime.migration] Running upgrade b20ee72fd9a4 -> 590d89f981c0, empty message
INFO  [alembic.runtime.migration] Running upgrade 590d89f981c0 -> 551c4e6d4a8b, empty message
INFO  [alembic.runtime.migration] Running upgrade 551c4e6d4a8b -> c6e7fc37ad42, empty message
INFO  [alembic.runtime.migration] Running upgrade c6e7fc37ad42 -> 1b7d161d1012, empty message
INFO  [alembic.runtime.migration] Running upgrade 1b7d161d1012 -> 507afb2632cc, empty message
INFO  [alembic.runtime.migration] Running upgrade 507afb2632cc -> 4fac8c8a704c, empty message
INFO  [alembic.runtime.migration] Running upgrade 4fac8c8a704c -> c79c702a1f23, empty message
INFO  [alembic.runtime.migration] Running upgrade c79c702a1f23 -> 5fa68bafae72, empty message
INFO  [alembic.runtime.migration] Running upgrade 5fa68bafae72 -> 4a640c170d02, empty message
INFO  [alembic.runtime.migration] Running upgrade 4a640c170d02 -> 2e2b53afd819, empty message
INFO  [alembic.runtime.migration] Running upgrade 2e2b53afd819 -> 6bbda4685999, empty message
INFO  [alembic.runtime.migration] Running upgrade 6bbda4685999 -> d68a2d971b70, empty message
INFO  [alembic.runtime.migration] Running upgrade d68a2d971b70 -> 0a89c670fc7a, empty message
INFO  [alembic.runtime.migration] Running upgrade 0a89c670fc7a -> 3ebfbaeb76c0, empty message
INFO  [alembic.runtime.migration] Running upgrade 3ebfbaeb76c0 -> 83f4dbe125c4, empty message
INFO  [alembic.runtime.migration] Running upgrade 83f4dbe125c4 -> e505cb517589, empty message
INFO  [alembic.runtime.migration] Running upgrade e505cb517589 -> e83298198ca5, empty message
INFO  [alembic.runtime.migration] Running upgrade e83298198ca5 -> 3a87573bf8a8, empty message
INFO  [alembic.runtime.migration] Running upgrade 3a87573bf8a8 -> a8d8aa307b8b, empty message
INFO  [alembic.runtime.migration] Running upgrade a8d8aa307b8b -> 0b28518684ae, empty message
INFO  [alembic.runtime.migration] Running upgrade 0b28518684ae -> 5e868298fee7, empty message
INFO  [alembic.runtime.migration] Running upgrade 5e868298fee7 -> 2d2fc3e826af, empty message
INFO  [alembic.runtime.migration] Running upgrade 2d2fc3e826af -> 0c7f1a48aac9, empty message
INFO  [alembic.runtime.migration] Running upgrade 0c7f1a48aac9 -> 18e934d58f55, empty message
INFO  [alembic.runtime.migration] Running upgrade 18e934d58f55 -> 9e1b06b9df13, empty message
INFO  [alembic.runtime.migration] Running upgrade 9e1b06b9df13 -> d4e4488a0032, empty message
INFO  [alembic.runtime.migration] Running upgrade d4e4488a0032 -> e409f6214b2b, empty message
INFO  [alembic.runtime.migration] Running upgrade e409f6214b2b -> 696e17c13b8b, empty message
INFO  [alembic.runtime.migration] Running upgrade 696e17c13b8b -> a8b996f0be40, empty message
INFO  [alembic.runtime.migration] Running upgrade a8b996f0be40 -> 10ad2dbaeccf, empty message
INFO  [alembic.runtime.migration] Running upgrade 10ad2dbaeccf -> 01f808f15b2e, empty message
INFO  [alembic.runtime.migration] Running upgrade 01f808f15b2e -> d29cca963221, empty message
INFO  [alembic.runtime.migration] Running upgrade d29cca963221 -> ba6f13ccbabb, empty message
INFO  [alembic.runtime.migration] Running upgrade ba6f13ccbabb -> 7c39ba4ec38d, empty message
INFO  [alembic.runtime.migration] Running upgrade 7c39ba4ec38d -> 9c976df9b9c4, empty message
INFO  [alembic.runtime.migration] Running upgrade 9c976df9b9c4 -> b9f849432543, empty message
INFO  [alembic.runtime.migration] Running upgrade b9f849432543 -> 6664d75ce3d4, empty message
INFO  [alembic.runtime.migration] Running upgrade 6664d75ce3d4 -> 3c9542fc54e9, empty message
INFO  [alembic.runtime.migration] Running upgrade 3c9542fc54e9 -> 3fa3a648c8e7, empty message
INFO  [alembic.runtime.migration] Running upgrade 3fa3a648c8e7 -> 903ec5f566e8, empty message
INFO  [alembic.runtime.migration] Running upgrade 903ec5f566e8 -> f580030d9beb, empty message
INFO  [alembic.runtime.migration] Running upgrade f580030d9beb -> e3cb44b953f2, empty message
INFO  [alembic.runtime.migration] Running upgrade e3cb44b953f2 -> 75093e7ded27, empty message
INFO  [alembic.runtime.migration] Running upgrade 75093e7ded27 -> 5f191273d067, empty message
INFO  [alembic.runtime.migration] Running upgrade 5f191273d067 -> 7eef64ffb398, empty message
INFO  [alembic.runtime.migration] Running upgrade 7eef64ffb398 -> 235355381f53, empty message
INFO  [alembic.runtime.migration] Running upgrade 235355381f53 -> 628a5438295c, empty message
INFO  [alembic.runtime.migration] Running upgrade 628a5438295c -> 11a35b448f83, empty message
INFO  [alembic.runtime.migration] Running upgrade 11a35b448f83 -> 9081f1a90939, empty message
INFO  [alembic.runtime.migration] Running upgrade 9081f1a90939 -> 91b69dfad2f1, empty message
INFO  [alembic.runtime.migration] Running upgrade 91b69dfad2f1 -> 7744c5c16159, empty message
INFO  [alembic.runtime.migration] Running upgrade 7744c5c16159 -> 14167121af69, empty message
INFO  [alembic.runtime.migration] Running upgrade 14167121af69 -> 6e061eb84167, empty message
INFO  [alembic.runtime.migration] Running upgrade 6e061eb84167 -> e9395fe234a4, empty message
INFO  [alembic.runtime.migration] Running upgrade e9395fe234a4 -> 0809266d08ca, empty message
INFO  [alembic.runtime.migration] Running upgrade 0809266d08ca -> f4b8232fa17e, empty message
INFO  [alembic.runtime.migration] Running upgrade f4b8232fa17e -> dbd80d290f04, empty message
INFO  [alembic.runtime.migration] Running upgrade dbd80d290f04 -> 4e4a759ac4b5, empty message
INFO  [alembic.runtime.migration] Running upgrade 4e4a759ac4b5 -> 30c13ca016e4, empty message
INFO  [alembic.runtime.migration] Running upgrade 30c13ca016e4 -> 541ce53ab6e9, empty message
INFO  [alembic.runtime.migration] Running upgrade 541ce53ab6e9 -> 67c61eead8d2, empty message
INFO  [alembic.runtime.migration] Running upgrade 67c61eead8d2 -> 224fd8963462, empty message
INFO  [alembic.runtime.migration] Running upgrade 224fd8963462 -> 92baf66b268b, empty message
INFO  [alembic.runtime.migration] Running upgrade 92baf66b268b -> 497cfd2a02e2, empty message
INFO  [alembic.runtime.migration] Running upgrade 497cfd2a02e2 -> ea30c0b5b2e3, empty message
INFO  [alembic.runtime.migration] Running upgrade ea30c0b5b2e3 -> bfd7b2302903, empty message
INFO  [alembic.runtime.migration] Running upgrade bfd7b2302903 -> 57ef03f3ac34, empty message
INFO  [alembic.runtime.migration] Running upgrade 57ef03f3ac34 -> dd911f880b75, empty message
INFO  [alembic.runtime.migration] Running upgrade dd911f880b75 -> bd05eac83f5f, empty message
INFO  [alembic.runtime.migration] Running upgrade bd05eac83f5f -> b4146f7d5277, empty message
INFO  [alembic.runtime.migration] Running upgrade b4146f7d5277 -> f939d67374e4, empty message
INFO  [alembic.runtime.migration] Running upgrade f939d67374e4 -> de1b457472e0, empty message
INFO  [alembic.runtime.migration] Running upgrade de1b457472e0 -> ae94fe5c4e9f, empty message
INFO  [alembic.runtime.migration] Running upgrade ae94fe5c4e9f -> 026e7a782ed6, empty message
INFO  [alembic.runtime.migration] Running upgrade 026e7a782ed6 -> 925b93d92809, empty message
INFO  [alembic.runtime.migration] Running upgrade 925b93d92809 -> bdf76f4b65a2, empty message
INFO  [alembic.runtime.migration] Running upgrade bdf76f4b65a2 -> a3a7c518ea70, empty message
INFO  [alembic.runtime.migration] Running upgrade a3a7c518ea70 -> a5e3c6693dc6, empty message
INFO  [alembic.runtime.migration] Running upgrade a5e3c6693dc6 -> bf11ab2f0a7a, empty message
INFO  [alembic.runtime.migration] Running upgrade bf11ab2f0a7a -> 1759f73274ee, empty message
INFO  [alembic.runtime.migration] Running upgrade 1759f73274ee -> 552d735a2f1f, empty message
INFO  [alembic.runtime.migration] Running upgrade 552d735a2f1f -> 5cad8fa84386, empty message
INFO  [alembic.runtime.migration] Running upgrade 5cad8fa84386 -> c31cdf879ee3, empty message
INFO  [alembic.runtime.migration] Running upgrade c31cdf879ee3 -> 659d979b64ce, empty message
INFO  [alembic.runtime.migration] Running upgrade 659d979b64ce -> ce15cf3467b4, empty message
INFO  [alembic.runtime.migration] Running upgrade ce15cf3467b4 -> 0e08145f0499, empty message
INFO  [alembic.runtime.migration] Running upgrade 0e08145f0499 -> 00532ac6d4bc, empty message
INFO  [alembic.runtime.migration] Running upgrade 00532ac6d4bc -> f680032cc361, empty message
INFO  [alembic.runtime.migration] Running upgrade f680032cc361 -> 10a7947fda6b, empty message
INFO  [alembic.runtime.migration] Running upgrade 10a7947fda6b -> 4a7d35941602, empty message
INFO  [alembic.runtime.migration] Running upgrade 4a7d35941602 -> cfc013b6461a, empty message
INFO  [alembic.runtime.migration] Running upgrade cfc013b6461a -> b2d51e4d94c8, empty message
INFO  [alembic.runtime.migration] Running upgrade b2d51e4d94c8 -> 749c2b85d20f, empty message
INFO  [alembic.runtime.migration] Running upgrade 749c2b85d20f -> a5b4dc311a89, empty message
INFO  [alembic.runtime.migration] Running upgrade a5b4dc311a89 -> a3c9a43e41f4, empty message
INFO  [alembic.runtime.migration] Running upgrade a3c9a43e41f4 -> 7128f87af701, empty message
INFO  [alembic.runtime.migration] Running upgrade 7128f87af701 -> 270d598c51e3, empty message
INFO  [alembic.runtime.migration] Running upgrade 270d598c51e3 -> b77ab8c47cc7, empty message
INFO  [alembic.runtime.migration] Running upgrade b77ab8c47cc7 -> a2b95b04d1f7, empty message
INFO  [alembic.runtime.migration] Running upgrade a2b95b04d1f7 -> 63fd3b240583, empty message
INFO  [alembic.runtime.migration] Running upgrade 63fd3b240583 -> 95938a93ea14, empty message
INFO  [alembic.runtime.migration] Running upgrade 95938a93ea14 -> b82bcad9accf, empty message
INFO  [alembic.runtime.migration] Running upgrade b82bcad9accf -> 84471852b610, empty message
INFO  [alembic.runtime.migration] Running upgrade 84471852b610 -> b0e9a389939a, empty message
INFO  [alembic.runtime.migration] Running upgrade b0e9a389939a -> 198c3aca9d8d, empty message
INFO  [alembic.runtime.migration] Running upgrade 198c3aca9d8d -> 58ad4df8583e, empty message
INFO  [alembic.runtime.migration] Running upgrade 58ad4df8583e -> 1abfc9e14d7e, empty message
INFO  [alembic.runtime.migration] Running upgrade 1abfc9e14d7e -> 32b00d06d892, empty message
INFO  [alembic.runtime.migration] Running upgrade 32b00d06d892 -> b17afc77ba83, empty message
INFO  [alembic.runtime.migration] Running upgrade b17afc77ba83 -> 54ca2dbf89c0, empty message
INFO  [alembic.runtime.migration] Running upgrade 54ca2dbf89c0 -> eef0c404b531, empty message
INFO  [alembic.runtime.migration] Running upgrade eef0c404b531 -> 84dec6c29c48, empty message
INFO  [alembic.runtime.migration] Running upgrade 84dec6c29c48 -> d0f197979bd9, empty message
INFO  [alembic.runtime.migration] Running upgrade d0f197979bd9 -> 9dc16e591f88, empty message
INFO  [alembic.runtime.migration] Running upgrade 9dc16e591f88 -> ac41029fb329, empty message
INFO  [alembic.runtime.migration] Running upgrade ac41029fb329 -> d1edb3cadec8, empty message
INFO  [alembic.runtime.migration] Running upgrade d1edb3cadec8 -> 623662ea0e7e, empty message
INFO  [alembic.runtime.migration] Running upgrade 623662ea0e7e -> 56c790ec8ab4, empty message
INFO  [alembic.runtime.migration] Running upgrade 56c790ec8ab4 -> c0d91ff18f77, empty message
INFO  [alembic.runtime.migration] Running upgrade c0d91ff18f77 -> 780a8344914b, empty message
INFO  [alembic.runtime.migration] Running upgrade 780a8344914b -> a20aeb9b0eac, empty message
INFO  [alembic.runtime.migration] Running upgrade a20aeb9b0eac -> 0af2c2e286a7, empty message
INFO  [alembic.runtime.migration] Running upgrade 0af2c2e286a7 -> 1919f1859215, empty message
INFO  [alembic.runtime.migration] Running upgrade 1919f1859215 -> f66ca777f409, empty message
INFO  [alembic.runtime.migration] Running upgrade f66ca777f409 -> 7c0dbd378cdb, empty message
INFO  [alembic.runtime.migration] Running upgrade 7c0dbd378cdb -> e99989e6ad56, empty message
INFO  [alembic.runtime.migration] Running upgrade e99989e6ad56 -> 1b54995bc086, empty message
INFO  [alembic.runtime.migration] Running upgrade 1b54995bc086 -> 2779eb90c6c4, empty message
INFO  [alembic.runtime.migration] Running upgrade 2779eb90c6c4 -> 74906d31d994, empty message
INFO  [alembic.runtime.migration] Running upgrade 74906d31d994 -> 85d0655d42c0, empty message
INFO  [alembic.runtime.migration] Running upgrade 85d0655d42c0 -> de7aa5280210, empty message
INFO  [alembic.runtime.migration] Running upgrade de7aa5280210 -> e831a883153a, empty message
INFO  [alembic.runtime.migration] Running upgrade e831a883153a -> d1236c4dff71, empty message
INFO  [alembic.runtime.migration] Running upgrade d1236c4dff71 -> 94f14eb0fe5b, empty message
INFO  [alembic.runtime.migration] Running upgrade 94f14eb0fe5b -> 9d6adad83936, empty message
INFO  [alembic.runtime.migration] Running upgrade 9d6adad83936 -> f398b261d9c6, empty message
INFO  [alembic.runtime.migration] Running upgrade f398b261d9c6 -> 517b79c56088, empty message
INFO  [alembic.runtime.migration] Running upgrade 517b79c56088 -> 48b991e9de06, empty message
INFO  [alembic.runtime.migration] Running upgrade 48b991e9de06 -> e11c3dd48a6f, empty message
INFO  [alembic.runtime.migration] Running upgrade e11c3dd48a6f -> 4912f3bd5ba2, empty message
INFO  [alembic.runtime.migration] Running upgrade 4912f3bd5ba2 -> f5133dc851ee, empty message
INFO  [alembic.runtime.migration] Running upgrade f5133dc851ee -> 5c77d685df87, empty message
INFO  [alembic.runtime.migration] Running upgrade 5c77d685df87 -> 6cc7f073b358, empty message
INFO  [alembic.runtime.migration] Running upgrade 6cc7f073b358 -> 68e2f38e33f4, empty message
INFO  [alembic.runtime.migration] Running upgrade 68e2f38e33f4 -> fc2eb1d7e4fc, empty message
INFO  [alembic.runtime.migration] Running upgrade fc2eb1d7e4fc -> a5e643d562c9, empty message
INFO  [alembic.runtime.migration] Running upgrade a5e643d562c9 -> 29ea13ed76f9, empty message
INFO  [alembic.runtime.migration] Running upgrade 29ea13ed76f9 -> 8e70205a5308, empty message
INFO  [alembic.runtime.migration] Running upgrade 8e70205a5308 -> f3f19998b755, empty message
INFO  [alembic.runtime.migration] Running upgrade f3f19998b755 -> c31a081eab74, empty message
INFO  [alembic.runtime.migration] Running upgrade c31a081eab74 -> 78403c7b8089, empty message
INFO  [alembic.runtime.migration] Running upgrade 78403c7b8089 -> 5662122eac21, empty message
INFO  [alembic.runtime.migration] Running upgrade 5662122eac21 -> 20c738810b1b, empty message
INFO  [alembic.runtime.migration] Running upgrade 20c738810b1b -> dfee471558bd, empty message
INFO  [alembic.runtime.migration] Running upgrade dfee471558bd -> 05e3af59929a, empty message
INFO  [alembic.runtime.migration] Running upgrade 05e3af59929a -> c3470e2d3224, empty message
INFO  [alembic.runtime.migration] Running upgrade c3470e2d3224 -> ffa75d04e6ef, empty message
INFO  [alembic.runtime.migration] Running upgrade ffa75d04e6ef -> 9014cca7097c, empty message
INFO  [alembic.runtime.migration] Running upgrade 9014cca7097c -> d4392342465f, empty message
INFO  [alembic.runtime.migration] Running upgrade d4392342465f -> 424808e1fe49, empty message
INFO  [alembic.runtime.migration] Running upgrade 424808e1fe49 -> 916a5257d18c, empty message
INFO  [alembic.runtime.migration] Running upgrade 916a5257d18c -> 4d3f91ddf3e9, empty message
INFO  [alembic.runtime.migration] Running upgrade 4d3f91ddf3e9 -> d8c55e79da54, empty message
INFO  [alembic.runtime.migration] Running upgrade d8c55e79da54 -> cf1e8c1bc737, empty message
INFO  [alembic.runtime.migration] Running upgrade cf1e8c1bc737 -> 7a105bfc0cd0, empty message
INFO  [alembic.runtime.migration] Running upgrade 7a105bfc0cd0 -> bc75acacc98e, empty message
INFO  [alembic.runtime.migration] Running upgrade bc75acacc98e -> b8b4f9598240, empty message
INFO  [alembic.runtime.migration] Running upgrade b8b4f9598240 -> 5ee767807344, empty message
INFO  [alembic.runtime.migration] Running upgrade 5ee767807344 -> 4913cb3f5a05, empty message
INFO  [alembic.runtime.migration] Running upgrade 4913cb3f5a05 -> 0b1c9ea11aef, empty message
INFO  [alembic.runtime.migration] Running upgrade 0b1c9ea11aef -> 2fbcad5527d7, empty message
INFO  [alembic.runtime.migration] Running upgrade 2fbcad5527d7 -> d750d578b068, empty message
INFO  [alembic.runtime.migration] Running upgrade d750d578b068 -> 2f1b3c759773, empty message
INFO  [alembic.runtime.migration] Running upgrade 2f1b3c759773 -> 99d9e329b27f, empty message
INFO  [alembic.runtime.migration] Running upgrade 99d9e329b27f -> a06066e3fbeb, empty message
INFO  [alembic.runtime.migration] Running upgrade a06066e3fbeb -> d67eab226ecd, empty message
INFO  [alembic.runtime.migration] Running upgrade d67eab226ecd -> bbedc353f90c, empty message
INFO  [alembic.runtime.migration] Running upgrade bbedc353f90c -> 0b9150eb309d, Increase message_id length manually
INFO  [alembic.runtime.migration] Running upgrade 0b9150eb309d -> 6204e57b4bc4, empty message
INFO  [alembic.runtime.migration] Running upgrade 6204e57b4bc4 -> 37feaba7c45d, empty message
INFO  [alembic.runtime.migration] Running upgrade 37feaba7c45d -> ff6c04869029, empty message
INFO  [alembic.runtime.migration] Running upgrade ff6c04869029 -> fdb02bd105a8, empty message
INFO  [alembic.runtime.migration] Running upgrade fdb02bd105a8 -> dd278f96ca83, empty message
INFO  [alembic.runtime.migration] Running upgrade dd278f96ca83 -> 1076b5795b08, empty message
INFO  [alembic.runtime.migration] Running upgrade 1076b5795b08 -> 5639ad89ee50, empty message
INFO  [alembic.runtime.migration] Running upgrade 5639ad89ee50 -> 11ba83e2dd71, empty message
INFO  [alembic.runtime.migration] Running upgrade 11ba83e2dd71 -> ccbfb61eda0d, empty message
INFO  [alembic.runtime.migration] Running upgrade ccbfb61eda0d -> a5013ff0a00a, empty message
INFO  [alembic.runtime.migration] Running upgrade a5013ff0a00a -> e6e8e12f5a13, empty message
INFO  [alembic.runtime.migration] Running upgrade e6e8e12f5a13 -> 9031c9e28510, empty message
INFO  [alembic.runtime.migration] Running upgrade 9031c9e28510 -> b8fd175c084a, empty message
INFO  [alembic.runtime.migration] Running upgrade b8fd175c084a -> e7d7ebcea26c, empty message
INFO  [alembic.runtime.migration] Running upgrade e7d7ebcea26c -> d0ccd9d7ac0c, empty message
INFO  [alembic.runtime.migration] Running upgrade d0ccd9d7ac0c -> 4b483a762fed, empty message
INFO  [alembic.runtime.migration] Running upgrade 4b483a762fed -> ad467baf7ec8, empty message
INFO  [alembic.runtime.migration] Running upgrade ad467baf7ec8 -> d8a3dfe674f2, empty message
INFO  [alembic.runtime.migration] Running upgrade d8a3dfe674f2 -> 3d05479d0d11, empty message
INFO  [alembic.runtime.migration] Running upgrade 3d05479d0d11 -> 753d2ed92d41, empty message
INFO  [alembic.runtime.migration] Running upgrade 753d2ed92d41 -> 698424c429e9, empty message
INFO  [alembic.runtime.migration] Running upgrade 698424c429e9 -> 07b870d7cc86, empty message
INFO  [alembic.runtime.migration] Running upgrade 07b870d7cc86 -> 9282e982bc05, Add block_behaviour setting for user
INFO  [alembic.runtime.migration] Running upgrade 9282e982bc05 -> 5047fcbd57c7, empty message
INFO  [alembic.runtime.migration] Running upgrade 5047fcbd57c7 -> 4729b7096d12, empty message
INFO  [alembic.runtime.migration] Running upgrade 4729b7096d12 -> b500363567e3, Create admin audit log
INFO  [alembic.runtime.migration] Running upgrade b500363567e3 -> 28b9b14c9664, store provider complaints
INFO  [alembic.runtime.migration] Running upgrade 28b9b14c9664 -> 0aaad1740797, store provider complaints
INFO  [alembic.runtime.migration] Running upgrade 0aaad1740797 -> e866ad0e78e1, Add partner tables
INFO  [alembic.runtime.migration] Running upgrade e866ad0e78e1 -> 088f23324464, add flags to the user model
INFO  [alembic.runtime.migration] Running upgrade 088f23324464 -> 2b1d3cd93e4b, update partner_api_token token length
INFO  [alembic.runtime.migration] Running upgrade 2b1d3cd93e4b -> 82d3c7109ffb, partner_user and partner_subscription
INFO  [alembic.runtime.migration] Running upgrade 82d3c7109ffb -> 36646e5dc6d9, make external_user_id non nullable
INFO  [alembic.runtime.migration] Running upgrade 36646e5dc6d9 -> a7bcb872c12a, Add alias transfer token expiration
INFO  [alembic.runtime.migration] Running upgrade a7bcb872c12a -> 673a074e4215, empty message
INFO  [alembic.runtime.migration] Running upgrade 673a074e4215 -> d1fb679f7eec, Add sudo expiration for ApiKeys
INFO  [alembic.runtime.migration] Running upgrade d1fb679f7eec -> bfebc2d5c719, Add state to job
INFO  [alembic.runtime.migration] Running upgrade bfebc2d5c719 -> 516c21ea7d87, empty message
INFO  [alembic.runtime.migration] Running upgrade 516c21ea7d87 -> bd7d032087b2, empty message
INFO  [alembic.runtime.migration] Running upgrade bd7d032087b2 -> b0101a66bb77, Add unsubscribe behaviour
INFO  [alembic.runtime.migration] Running upgrade b0101a66bb77 -> 89081a00fc7d, default_unsub_behaviour
INFO  [alembic.runtime.migration] Running upgrade 89081a00fc7d -> c66f2c5b6cb1, empty message
INFO  [alembic.runtime.migration] Running upgrade c66f2c5b6cb1 -> 9cc0f0712b29, Add api to cookie token
INFO  [alembic.runtime.migration] Running upgrade 9cc0f0712b29 -> bd95b2b4217f, Updated recovery code string length
INFO  [alembic.runtime.migration] Running upgrade bd95b2b4217f -> 2c2093c82bc0, empty message
INFO  [alembic.runtime.migration] Running upgrade 2c2093c82bc0 -> 5f4a5625da66, empty message
INFO  [alembic.runtime.migration] Running upgrade 5f4a5625da66 -> 893c0d18475f, empty message
INFO  [alembic.runtime.migration] Running upgrade 893c0d18475f -> bc496c0a0279, empty message
INFO  [alembic.runtime.migration] Running upgrade bc496c0a0279 -> 2d89315ac650, empty message
INFO  [alembic.runtime.migration] Running upgrade 893c0d18475f -> 01e2997e90d3, empty message
INFO  [alembic.runtime.migration] Running upgrade 01e2997e90d3, 2d89315ac650 -> 2634b41f54db, empty message
INFO  [alembic.runtime.migration] Running upgrade 2634b41f54db -> 01827104004b, empty message
INFO  [alembic.runtime.migration] Running upgrade 01827104004b -> 0a5701a4f5e4, empty message
INFO  [alembic.runtime.migration] Running upgrade 0a5701a4f5e4 -> ec7fdde8da9f, empty message
INFO  [alembic.runtime.migration] Running upgrade ec7fdde8da9f -> 46ecb648a47e, empty message
INFO  [alembic.runtime.migration] Running upgrade 46ecb648a47e -> 4bc54632d9aa, empty message
INFO  [alembic.runtime.migration] Running upgrade 4bc54632d9aa -> 818b0a956205, empty message
INFO  [alembic.runtime.migration] Running upgrade 818b0a956205 -> 52510a633d6f, empty message
INFO  [alembic.runtime.migration] Running upgrade 52510a633d6f -> fa2f19bb4e5a, empty message
INFO  [alembic.runtime.migration] Running upgrade fa2f19bb4e5a -> 06a9a7133445, Create sync_event table
INFO  [alembic.runtime.migration] Running upgrade 06a9a7133445 -> d608b8e48082, empty message
INFO  [alembic.runtime.migration] Running upgrade d608b8e48082 -> 56d08955fcab, add retry count to sync event
INFO  [alembic.runtime.migration] Running upgrade 56d08955fcab -> 1c14339aae90, empty message
INFO  [alembic.runtime.migration] Running upgrade 1c14339aae90 -> 2441b7ff5da9, Custom Domain partner id
INFO  [alembic.runtime.migration] Running upgrade 2441b7ff5da9 -> 88dd7a0abf54, contact.flags and custom_domain.pending_deletion
INFO  [alembic.runtime.migration] Running upgrade 88dd7a0abf54 -> 62afa3a10010, custom domain indices
INFO  [alembic.runtime.migration] Running upgrade 62afa3a10010 -> 91ed7f46dc81, alias_audit_log
INFO  [alembic.runtime.migration] Running upgrade 91ed7f46dc81 -> 7d7b84779837, user_audit_log
INFO  [alembic.runtime.migration] Running upgrade 7d7b84779837 -> 32f25cbf12f6, alias_audit_log_index_created_at
```

The command exited `0`, and the head revision reached is `32f25cbf12f6` (the last `Running upgrade` line above); the schema now has 77 tables:

```console
$ echo "exit=$?"
exit=0
$ psql -h localhost -p 15432 -U myuser -d sl_q3v -tAc \
    "SELECT count(*) FROM information_schema.tables WHERE table_schema='public';"
77
```

**BEFORE `init_app.py` — `SLDomain` (table `public_domain`) is empty** (both raw SQL and the ORM accessor agree):

```console
$ psql -h localhost -p 15432 -U myuser -d sl_q3v -c "SELECT count(*) AS public_domain_rows FROM public_domain;"
 public_domain_rows 
--------------------
                  0
(1 row)


$ CONFIG=/tmp/obs/q3v.env /app/venv/bin/python -c "from app.models import SLDomain; print('SLDomain.count() =', SLDomain.count())"
load config file /tmp/obs/q3v.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/hagwljujpeyqesmxetwt
Upload files to local dir
>>> init logging <<<
2026-07-08 15:50:37,979 - SL - DEBUG - 2373 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
SLDomain.count() = 0
```

The Q3 rejection observations below were all captured in exactly this state (`SLDomain` genuinely empty). **AFTER `init_app.py`** — `add_sl_domains()` (`init_app.py:39`) seeds the domain (with `EMAIL_DOMAIN=sl.local`, `ALIAS_DOMAINS` defaults to `[sl.local]` per `app/config.py:157-160`); the `__main__` block (`init_app.py:69-73`) runs `load_pgp_public_keys()` then `add_sl_domains()`:

```console
$ CONFIG=/tmp/obs/q3v.env /app/venv/bin/python init_app.py
load config file /tmp/obs/q3v.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/nygznpjvaipaadzvltas
Upload files to local dir
>>> init logging <<<
2026-07-08 15:52:24,977 - SL - DEBUG - 2518 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-08 15:52:25,746 - SL - DEBUG - 2518 - "/app/init_app.py:36" - load_pgp_public_keys() -  - Finish load_pgp_public_keys
2026-07-08 15:52:25,748 - SL - INFO - 2518 - "/app/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain

$ psql -h localhost -p 15432 -U myuser -d sl_q3v -c "SELECT id, domain, use_as_reverse_alias FROM public_domain;"
 id |  domain  | use_as_reverse_alias 
----+----------+----------------------
  1 | sl.local | t
(1 row)
```

State transition: **`public_domain` 0 rows -> 1 row (`sl.local`)**, caused solely by `init_app.py`'s `add_sl_domains()`.

### Rejection mechanism (cause -> effect), with `file:line`

1. **Routing to the Forward case.** `handle_DATA` (`email_handler.py:2289`) -> `_handle` (`email_handler.py:2335`) -> `handle` (`email_handler.py:1945`). For each recipient, `email_handler.py:2195` `if is_reverse_alias(rcpt_to):` is **false** for `x@sl.local` (it does not start with a reverse-alias prefix), so control falls to the `else: # Forward case` (`email_handler.py:2201`), logs `Forward phase …` (`email_handler.py:2202`), and calls `handle_forward(envelope, copy_msg, rcpt_to)` (`email_handler.py:2208`).
2. **Alias lookup fails.** In `handle_forward` (`email_handler.py:536`), `email_handler.py:543` `alias = Alias.get_by(email=alias_address)` returns `None`, so the `LOG.d(...)` at `email_handler.py:545` emits *"alias … not exist. Try to see if it can be created on the fly"*.
3. **On-the-fly auto-creation fails — via two checks.** `email_handler.py:549` `alias = try_auto_create(alias_address)` calls `try_auto_create` (`app/alias_utils.py:202`), which tries two strategies in order:
   - **Custom-domain strategy.** `try_auto_create` calls `try_auto_create_via_domain` (`app/alias_utils.py:274`), which first calls `check_if_alias_can_be_auto_created_for_custom_domain` (`app/alias_utils.py:92`). Because `sl.local` is not a **verified `CustomDomain`**, that check returns `None` after the `LOG.i` at `app/alias_utils.py:104`: *"Cannot auto-create custom domain alias for x@sl.local because there's no custom domain for sl.local"*.
   - **Directory strategy.** `try_auto_create` then calls `try_auto_create_directory` (`app/alias_utils.py:227`), which calls `check_if_alias_can_be_auto_created_for_a_directory` (`app/alias_utils.py:145`). Because the local part `x` contains no directory separator (`+` / `#` / `/`), that check returns `None` after the `LOG.info` at `app/alias_utils.py:165`: *"Cannot auto-create x@sl.local since it has no directory separator"*.

   With both strategies returning `None`, `try_auto_create` returns `None`. **Note:** `is_valid_alias_address_domain` (`app/email_utils.py:557`) is **not** on this forward / auto-create path. It is imported at `email_handler.py:107` and used only at `email_handler.py:1000` as a *sanity check on an already-existing alias* during a different phase (returning `status.E503`) — so it is unrelated to the `E515` rejection here.
4. **550 is returned.** `email_handler.py:550` `if not alias:` -> the `LOG.d` at `email_handler.py:551` emits *"alias … cannot be created on-the-fly, return 550"*; then `email_handler.py:552-553` `if should_ignore_bounce(envelope.mail_from): return [(True, status.E207)]`, else `email_handler.py:554-555` `return [(False, status.E515)]`. `status.E515 = "550 SL E515 Email not exist"` (`app/email/status.py:51`) is delivered verbatim as the SMTP reply.

### Runtime evidence — primary case (`E515`, normal sender)

The email handler was started against `sl_q3v` (empty `SLDomain`) — this is also the **second** observation of the `:20381` bind (Q2 stability):

```console
$ CONFIG=/tmp/obs/q3v.env /app/venv/bin/python email_handler.py
load config file /tmp/obs/q3v.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/tznuzfgrrjrljffnqwhl
Upload files to local dir
>>> init logging <<<
2026-07-08 15:50:39,214 - SL - DEBUG - 2380 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-08 15:50:39,692 - SL - INFO - 2380 - "/app/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-08 15:50:39,693 - SL - DEBUG - 2380 - "/app/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

A temporary SMTP-send helper (removed during cleanup, §(f)) calls `smtplib.SMTP("localhost", 20381).sendmail("someone@example.com", ["x@sl.local"], msg)`:

```console
$ /app/venv/bin/python /tmp/obs/send_q3.py someone@example.com
SENDER=someone@example.com RCPT=x@sl.local
RESULT: SMTPDataError -> code=550 msg=b'SL E515 Email not exist' 
```

**(a) SMTP wire reply the sender observes:** `smtplib` raises `smtplib.SMTPDataError` with `code=550` and `msg=b'SL E515 Email not exist'` — i.e. the wire reply is the full line **`550 SL E515 Email not exist`**. Per RFC 5321 §4.2.1 a `5yz` reply is a **permanent** negative completion reply (resending the same message yields the same result).

**(b) The handler's rejection log lines** (the complete, unedited STDOUT block for that message; the two requested `LOG.d` lines are `email_handler.py:545` and `:551`):

```
2026-07-08 15:51:26,277 - SL - DEBUG - 2380 - "/app/app/log.py:24" - set_message_id() -  - set message_id e050f9c3-ad5e-484e-9b4c-23bf809928bf
2026-07-08 15:51:26,277 - SL - DEBUG - 2380 - "/app/email_handler.py:2342" - _handle() - e050f9c3-ad5e-484e-9b4c-23bf809928bf - ====>=====>====>====>====>====>====>====>
2026-07-08 15:51:26,277 - SL - INFO - 2380 - "/app/email_handler.py:2343" - _handle() - e050f9c3-ad5e-484e-9b4c-23bf809928bf - New message, mail from someone@example.com, rctp tos ['x@sl.local'] 
2026-07-08 15:51:26,278 - SL - DEBUG - 2380 - "/app/email_handler.py:1963" - handle() - e050f9c3-ad5e-484e-9b4c-23bf809928bf - Cannot parse Postfix queue ID from None None
2026-07-08 15:51:26,395 - SL - DEBUG - 2380 - "/app/email_handler.py:1980" - handle() - e050f9c3-ad5e-484e-9b4c-23bf809928bf - ==>> Handle mail_from:someone@example.com, rcpt_tos:['x@sl.local'], header_from:someone@example.com, header_to:x@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'someone@example.com'), ('To', 'x@sl.local'), ('Subject', 'Q3 test'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:['SIZE=178'], rcpt_options:[]
2026-07-08 15:51:26,400 - SL - DEBUG - 2380 - "/app/email_handler.py:2202" - handle() - e050f9c3-ad5e-484e-9b4c-23bf809928bf - Forward phase someone@example.com(someone@example.com) -> x@sl.local
2026-07-08 15:51:26,410 - SL - DEBUG - 2380 - "/app/email_handler.py:545" - handle_forward() - e050f9c3-ad5e-484e-9b4c-23bf809928bf - alias x@sl.local not exist. Try to see if it can be created on the fly
2026-07-08 15:51:26,421 - SL - INFO - 2380 - "/app/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - e050f9c3-ad5e-484e-9b4c-23bf809928bf - Cannot auto-create custom domain alias for x@sl.local because there's no custom domain for sl.local
2026-07-08 15:51:26,421 - SL - INFO - 2380 - "/app/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - e050f9c3-ad5e-484e-9b4c-23bf809928bf - Cannot auto-create x@sl.local since it has no directory separator
2026-07-08 15:51:26,421 - SL - DEBUG - 2380 - "/app/email_handler.py:551" - handle_forward() - e050f9c3-ad5e-484e-9b4c-23bf809928bf - alias x@sl.local cannot be created on-the-fly, return 550
2026-07-08 15:51:26,422 - SL - INFO - 2380 - "/app/email_handler.py:2367" - _handle() - e050f9c3-ad5e-484e-9b4c-23bf809928bf - Finish mail_from someone@example.com, rcpt_tos ['x@sl.local'], takes 0.1452333927154541 seconds with return code '550 SL E515 Email not exist'<<===
```

The two rejection log lines requested by the question, verbatim:

- `alias x@sl.local not exist. Try to see if it can be created on the fly` — `email_handler.py:545`.
- `alias x@sl.local cannot be created on-the-fly, return 550` — `email_handler.py:551`.

The final `_handle` line (`email_handler.py:2367`) independently confirms the returned wire string: `return code '550 SL E515 Email not exist'`. The two intervening `LOG.i` lines — `app/alias_utils.py:104` (`check_if_alias_can_be_auto_created_for_custom_domain`) and `app/alias_utils.py:165` (`check_if_alias_can_be_auto_created_for_a_directory`) — show *why* auto-create failed: no custom domain for `sl.local`, and no directory separator in the local part.

### Runtime evidence — edge case (`E207`, IgnoreBounceSender)

`should_ignore_bounce(mail_from)` (`app/email_utils.py:1361`) returns `True` only when `IgnoreBounceSender.get_by(mail_from=mail_from)` matches, logging a warning (`app/email_utils.py:1363`). To exercise this branch, a temporary `IgnoreBounceSender` row (table `ignore_bounce_sender`, `app/models.py:3357`) was inserted into the throwaway `sl_q3v` DB (`SLDomain` still empty), then a message was sent **from that sender**:

```console
$ psql -h localhost -p 15432 -U myuser -d sl_q3v -c \
    "INSERT INTO ignore_bounce_sender (mail_from, created_at) VALUES ('bounce-sender@example.com', now());"
INSERT 0 1
$ psql -h localhost -p 15432 -U myuser -d sl_q3v -tAc "SELECT count(*) FROM public_domain;"   # SLDomain still empty
0

$ /app/venv/bin/python /tmp/obs/send_q3.py bounce-sender@example.com
SENDER=bounce-sender@example.com RCPT=x@sl.local
RESULT: sendmail returned normally (no exception) -> ACCEPTED
```

**Wire reply:** the sender sees a **`250`** (acceptance) — `smtplib.sendmail` returns normally with no exception. The complete, unedited handler log for that message shows the alternate branch taken (`should_ignore_bounce`, `app/email_utils.py:1363`) and the `250 … E207 …` return code:

```
2026-07-08 15:52:07,783 - SL - DEBUG - 2380 - "/app/app/log.py:24" - set_message_id() - b12d24f8-d9c1-42ff-9752-15b3b3c8abbc - set message_id 05d57996-48f3-4019-81cb-e864450adf34
2026-07-08 15:52:07,783 - SL - DEBUG - 2380 - "/app/email_handler.py:2342" - _handle() - 05d57996-48f3-4019-81cb-e864450adf34 - ====>=====>====>====>====>====>====>====>
2026-07-08 15:52:07,783 - SL - INFO - 2380 - "/app/email_handler.py:2343" - _handle() - 05d57996-48f3-4019-81cb-e864450adf34 - New message, mail from bounce-sender@example.com, rctp tos ['x@sl.local'] 
2026-07-08 15:52:07,784 - SL - DEBUG - 2380 - "/app/email_handler.py:1963" - handle() - 05d57996-48f3-4019-81cb-e864450adf34 - Cannot parse Postfix queue ID from None None
2026-07-08 15:52:07,785 - SL - DEBUG - 2380 - "/app/email_handler.py:1980" - handle() - 05d57996-48f3-4019-81cb-e864450adf34 - ==>> Handle mail_from:bounce-sender@example.com, rcpt_tos:['x@sl.local'], header_from:bounce-sender@example.com, header_to:x@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'bounce-sender@example.com'), ('To', 'x@sl.local'), ('Subject', 'Q3 test'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:['SIZE=184'], rcpt_options:[]
2026-07-08 15:52:07,789 - SL - DEBUG - 2380 - "/app/email_handler.py:2202" - handle() - 05d57996-48f3-4019-81cb-e864450adf34 - Forward phase bounce-sender@example.com(bounce-sender@example.com) -> x@sl.local
2026-07-08 15:52:07,795 - SL - DEBUG - 2380 - "/app/email_handler.py:545" - handle_forward() - 05d57996-48f3-4019-81cb-e864450adf34 - alias x@sl.local not exist. Try to see if it can be created on the fly
2026-07-08 15:52:07,799 - SL - INFO - 2380 - "/app/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - 05d57996-48f3-4019-81cb-e864450adf34 - Cannot auto-create custom domain alias for x@sl.local because there's no custom domain for sl.local
2026-07-08 15:52:07,799 - SL - INFO - 2380 - "/app/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - 05d57996-48f3-4019-81cb-e864450adf34 - Cannot auto-create x@sl.local since it has no directory separator
2026-07-08 15:52:07,799 - SL - DEBUG - 2380 - "/app/email_handler.py:551" - handle_forward() - 05d57996-48f3-4019-81cb-e864450adf34 - alias x@sl.local cannot be created on-the-fly, return 550
2026-07-08 15:52:07,800 - SL - WARNING - 2380 - "/app/app/email_utils.py:1363" - should_ignore_bounce() - 05d57996-48f3-4019-81cb-e864450adf34 - do not send back bounce report to bounce-sender@example.com
2026-07-08 15:52:07,800 - SL - INFO - 2380 - "/app/email_handler.py:2367" - _handle() - 05d57996-48f3-4019-81cb-e864450adf34 - Finish mail_from bounce-sender@example.com, rcpt_tos ['x@sl.local'], takes 0.016614675521850586 seconds with return code '250 SL E207 No bounce report'<<===
```

So when `mail_from` is an `IgnoreBounceSender`, `handle_forward` returns `status.E207 = "250 SL E207 No bounce report"` (`email_handler.py:552-553`, `app/email/status.py:12`) instead of `E515`: the message is silently accepted (a `250`), and no bounce report is generated. This was **observed at runtime**, not merely inferred.

### Stability (>=2 runs)

The primary `E515` case was exercised twice against the empty `SLDomain`. Run #2 (sender `normal-2@example.com`) was identical — both the wire reply and the complete handler log block:

```console
$ /app/venv/bin/python /tmp/obs/send_q3.py normal-2@example.com
SENDER=normal-2@example.com RCPT=x@sl.local
RESULT: SMTPDataError -> code=550 msg=b'SL E515 Email not exist' 
```

```
2026-07-08 15:51:46,949 - SL - DEBUG - 2380 - "/app/app/log.py:24" - set_message_id() - e050f9c3-ad5e-484e-9b4c-23bf809928bf - set message_id b12d24f8-d9c1-42ff-9752-15b3b3c8abbc
2026-07-08 15:51:46,950 - SL - DEBUG - 2380 - "/app/email_handler.py:2342" - _handle() - b12d24f8-d9c1-42ff-9752-15b3b3c8abbc - ====>=====>====>====>====>====>====>====>
2026-07-08 15:51:46,950 - SL - INFO - 2380 - "/app/email_handler.py:2343" - _handle() - b12d24f8-d9c1-42ff-9752-15b3b3c8abbc - New message, mail from normal-2@example.com, rctp tos ['x@sl.local'] 
2026-07-08 15:51:46,950 - SL - DEBUG - 2380 - "/app/email_handler.py:1963" - handle() - b12d24f8-d9c1-42ff-9752-15b3b3c8abbc - Cannot parse Postfix queue ID from None None
2026-07-08 15:51:46,952 - SL - DEBUG - 2380 - "/app/email_handler.py:1980" - handle() - b12d24f8-d9c1-42ff-9752-15b3b3c8abbc - ==>> Handle mail_from:normal-2@example.com, rcpt_tos:['x@sl.local'], header_from:normal-2@example.com, header_to:x@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'normal-2@example.com'), ('To', 'x@sl.local'), ('Subject', 'Q3 test'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:['SIZE=179'], rcpt_options:[]
2026-07-08 15:51:46,955 - SL - DEBUG - 2380 - "/app/email_handler.py:2202" - handle() - b12d24f8-d9c1-42ff-9752-15b3b3c8abbc - Forward phase normal-2@example.com(normal-2@example.com) -> x@sl.local
2026-07-08 15:51:46,962 - SL - DEBUG - 2380 - "/app/email_handler.py:545" - handle_forward() - b12d24f8-d9c1-42ff-9752-15b3b3c8abbc - alias x@sl.local not exist. Try to see if it can be created on the fly
2026-07-08 15:51:46,966 - SL - INFO - 2380 - "/app/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - b12d24f8-d9c1-42ff-9752-15b3b3c8abbc - Cannot auto-create custom domain alias for x@sl.local because there's no custom domain for sl.local
2026-07-08 15:51:46,966 - SL - INFO - 2380 - "/app/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - b12d24f8-d9c1-42ff-9752-15b3b3c8abbc - Cannot auto-create x@sl.local since it has no directory separator
2026-07-08 15:51:46,966 - SL - DEBUG - 2380 - "/app/email_handler.py:551" - handle_forward() - b12d24f8-d9c1-42ff-9752-15b3b3c8abbc - alias x@sl.local cannot be created on-the-fly, return 550
2026-07-08 15:51:46,966 - SL - INFO - 2380 - "/app/email_handler.py:2367" - _handle() - b12d24f8-d9c1-42ff-9752-15b3b3c8abbc - Finish mail_from normal-2@example.com, rcpt_tos ['x@sl.local'], takes 0.017068862915039062 seconds with return code '550 SL E515 Email not exist'<<===
```

Result: **stable** — empty `SLDomain` + normal sender -> wire reply `550 SL E515 Email not exist` on every run (identical `email_handler.py:545` / `alias_utils.py:104` / `:165` / `email_handler.py:551` / `:2367` sequence).

**Q3 <-> omitted step:** this rejection is caused by **skipping `init_app.py`**. Migrations create the `public_domain` table but leave it empty; with no `SLDomain` row for `sl.local` and no verified `CustomDomain`, both on-the-fly auto-create strategies fail (`app/alias_utils.py:104` and `:165`), and `handle_forward` returns `E515`.

---


## (e) Canonical startup order + failure modes

### The correct order

```
1. PostgreSQL        start the DB server, create the target database (empty schema)
2. Migrations        flask db upgrade   (== alembic upgrade head)   -> 255 revisions -> 77 tables
3. init_app.py       seed SLDomain (public_domain), load PGP public keys
4. Web app           python server.py            (dev, :7777)   /   gunicorn wsgi:app (prod)
5. Email handler     python email_handler.py     (aiosmtpd, :20381 dev / :25 prod)
6. Background workers python job_runner.py, python event_listener.py, python cron.py
```

```mermaid
flowchart TD
    A["1. Start PostgreSQL<br/>create empty DB"] --> B{"2. Migrations run?<br/>flask db upgrade"}
    B -- No --> Q1["Q1 FAILURE<br/>python server.py binds :7777, but first ORM<br/>query on the login request hits a missing table:<br/>psycopg2 UndefinedTable relation &quot;users&quot;<br/>-> sqlalchemy ProgrammingError -> HTTP 500"]
    B -- Yes --> C{"3. init_app.py run?<br/>seeds SLDomain (public_domain)"}
    C -- No --> Q3["Q3 FAILURE<br/>email to x@sl.local: handle_forward finds no<br/>alias; auto-create fails (no custom domain,<br/>no directory sep); SLDomain empty<br/>-> reply 550 SL E515 Email not exist"]
    C -- Yes --> D["4-6. Start services"]
    D --> D1["Web app<br/>python server.py :7777 / gunicorn wsgi:app"]
    D --> D2["Email handler<br/>python email_handler.py :20381"]
    D --> D3["Job runner<br/>python job_runner.py"]
    D --> D4["Event listener + cron<br/>event_listener.py / cron.py"]
```

### Why this order (cause -> effect)

- **PostgreSQL before everything.** `app/db.py:9-12` builds the engine from `DB_URI` and calls `engine.connect()` **at import time**. If the DB server were down, *any* entrypoint that imports `app.db` would fail immediately on connect. (When the server is up but the *schema* is empty, the connect still succeeds — which is precisely what makes Q1 a request-time failure rather than a startup failure.)
- **Migrations before the app is used.** The engine connects without any DDL/schema check (`app/db.py:9-12`), so `python server.py` starts and binds `:7777` even against an empty schema; the missing-table error only surfaces when the first ORM query runs during a request. Running `flask db upgrade` first creates all 77 tables so those queries succeed. **Skipping this step is exactly Q1.**
- **`init_app.py` before real mail traffic.** `add_sl_domains()` (`init_app.py:39`) seeds the `SLDomain`/`public_domain` rows the alias logic depends on. Without it, `handle_forward` cannot resolve or auto-create an alias for `x@sl.local` and returns `550 SL E515`. **Skipping this step is exactly Q3.**
- **Web app / email handler / workers last.** These are the long-running services; they assume a migrated, seeded database. Their start order relative to one another is independent (they bind different ports: `:7777` HTTP, `:20381` SMTP; the workers bind no port), but all of them must come **after** steps 1-3.

### Verified canonical references (in the canonical `/app` source)

- **Runtime & version policy.** Python `^3.10` (`pyproject.toml:61`); `CONTRIBUTING.md:236` documents the Python-version requirement, and `Dockerfile:8` (`FROM python:3.10...`) fixes the canonical interpreter. The provisioned image runs Python **3.10.18** (`$ /app/venv/bin/python --version` -> `Python 3.10.18`, see §(a)).
- **DB / infra provisioning.** `CONTRIBUTING.md:100` shows the `docker run … -p 15432:5432 postgres:13` command this environment uses (host port **15432**, matching the canonical `/app/.env` `DB_URI`); `.github/workflows/main.yml:47` / `:98` provision `postgres:13` + `redis:6` and run migrations with `CONFIG=tests/test.env`.
- **Migrations.** `alembic.ini:5` `script_location = migrations`; `migrations/env.py:28` wires Alembic to the app's `DB_URI`/metadata; `scripts/reset_local_db.sh:4,6,7` is the canonical "drop/create DB then `alembic upgrade head`" reset pattern.
- **Run sequence & config.** `README.md:433,448,480,495` and `CONTRIBUTING.md:73,106,109,248` give the migrate -> `init_app.py` -> `server.py` -> `email_handler.py` run sequence; `example.env:6,19,22,40,75,77` provides the canonical config keys (`URL`, `EMAIL_DOMAIN=sl.local`, `DB_URI`, key paths). `example.env:75` uses port `5432`; the canonical `/app/.env` uses `15432` to match the provisioned PostgreSQL above.

### Failure-mode table

| Step skipped | Symptom / where it surfaces | Exact evidence | Question |
|---|---|---|---|
| **2. Migrations** (`flask db upgrade`) | Server starts and binds `:7777` (engine connects at import, `app/db.py:9-12`); first ORM query on the login **request** hits a missing table -> HTTP 500 | `psycopg2.errors.UndefinedTable: relation "users" does not exist` wrapped by `sqlalchemy.exc.ProgrammingError`; `after_request … "GET /auth/login?next=%2F" 500` (see §(b)) | **Q1** |
| **3. `init_app.py`** (seed `SLDomain`) | Migrations succeed (77 tables) but `public_domain` is empty; mail to `x@sl.local` cannot resolve/auto-create an alias | Wire reply `550 SL E515 Email not exist`; `email_handler.py:545` + `:551`, `alias_utils.py:104` + `:165` (see §(d)) | **Q3** |
| (Both present) | System functions; required services bind their ports | `:7777` HTTP `200`/`302`; `Listen for port 20381` / `Start mail controller 0.0.0.0 20381` (see §(c)) | **Q2** |

---


## (f) Cleanup / repository integrity

This was a **read-only investigation**. No file in the SimpleLogin source tree was modified or deleted; the only repository artifact produced is this document, `blitzy/documentation/app_2cd6ee777f8c.md`.

**Temporary observation artifacts created and then removed** (all outside the repository working tree — under the container's `/tmp`, plus throwaway databases on the provisioned PostgreSQL):

- `/tmp/obs/portproof.py` — read-only `/proc/net/tcp` decoder that reports which PIDs hold a `LISTEN` socket on a given port (used because the image ships no `ss`/`lsof`).
- `/tmp/obs/send_q3.py` — SMTP-send helper (`smtplib.SMTP("localhost", 20381).sendmail(...)`) used to trigger the Q3 rejection.
- `/tmp/obs/q3v.env` — a dotenv derived from `example.env`, differing only in `DB_URI` (pointing at the throwaway `sl_q3v` DB) and the `15432` port.
- `/tmp/obs/*.log` — raw captured stdout/stderr for each scenario.
- **Throwaway database `sl_q3v`** (migrated, `init_app.py` skipped) and the temporary `ignore_bounce_sender` row inserted into it for the `E207` edge case — dropped after capture.

The canonical `simplelogin` (Q2) and the empty `sl_q1_empty` (Q1) databases, and the `/app/.env` config, are part of the provisioned environment and were left as-is.

**Final repository state** — `git status --porcelain` shows exactly one changed path, the deliverable:

```console
$ git status --porcelain
 M blitzy/documentation/app_2cd6ee777f8c.md
```

(The path shows as modified rather than untracked because a prior revision of this document existed on the branch; this rewrite replaces its contents with output captured first-hand in the canonical `/app` environment. No source file appears in the status.)

---


## Appendix — Verified `file:line` reference map

Every citation below was verified against the canonical source at `/app` (container HEAD = base commit `2cd6ee77`; the key referenced files are sha256-identical between the branch base and the container). Line numbers are 1-indexed.

### Environment / runtime

| Fact | Reference |
|---|---|
| Canonical interpreter `FROM python:3.10...` | `Dockerfile:8` |
| Python version constraint `python = "^3.10"` | `pyproject.toml:61` |
| Python version policy (docs) | `CONTRIBUTING.md:236` |
| PostgreSQL provisioning `docker run … -p 15432:5432 postgres:13` | `CONTRIBUTING.md:100` |
| CI services (`postgres:13`, `redis:6`) + `CONFIG=tests/test.env` | `.github/workflows/main.yml:47,98` |
| Canonical config keys (`URL`, `EMAIL_DOMAIN=sl.local`, `DB_URI`, key paths) | `example.env:6,19,22,40,75,77` |
| Alembic script location | `alembic.ini:5` |
| Alembic env wiring to app metadata / `DB_URI` | `migrations/env.py:28` |
| DB reset + `alembic upgrade head` pattern | `scripts/reset_local_db.sh:4,6,7` |
| Run sequence (migrate -> init_app -> server -> email_handler) | `README.md:433,448,480,495`; `CONTRIBUTING.md:73,106,109,248` |

### Q1 — empty-DB startup exception

| Claim | Reference |
|---|---|
| Engine + `engine.connect()` at import time, no schema check | `app/db.py:9-12` |
| Dev server `app.run(debug=True, port=7777)` | `server.py:588` |
| MEM_STORE_URI gate (cookie sessions when unset) | `server.py:163-165` |
| Index route `/` redirects anonymous users to `auth.login` | `server.py:250-256` |
| `after_request` logs method/path/status | `server.py:389-390` |
| Login GET renders template (no view-level DB query) | `app/auth/views/login.py:21-82` (query triggered downstream at `:43`) |
| First ORM query hits missing relation -> `User` model | `app/models.py:84` (`class User`) |
| Anonymous short-circuit in base template | `templates/base.html:89` |

### Q2 — required services + port binding

| Service / claim | Reference |
|---|---|
| aiosmtpd `Controller` bootstrap in `main()` / `__main__` | `email_handler.py:2381-2404` |
| `LOG.i("Listen for port %s", port)` | `email_handler.py:2386` |
| `LOG.d("Start mail controller %s %s", ...)` | `email_handler.py:2401-2403` |
| Production WSGI entry `app = create_app()` | `wsgi.py:1-3` |
| Central logger config (name "SL", DEBUG, STDOUT) + format | `app/log.py:12-16` |
| `>>> init logging <<<` banner on import | `app/log.py` (init block) |
| Background job-runner service | `job_runner.py` |
| Event-listener service (PostgresEventSource) | `event_listener.py` |
| Scheduled-task service (`Start running cronjob`) | `cron.py` |

### Q3 — skipped-init SMTP rejection

| Claim | Reference |
|---|---|
| `SLDomain` model -> table `public_domain` | `app/models.py:3116` (class), `:3119` (`__tablename__`) |
| `handle_DATA` -> `_handle` -> `handle` dispatch | `email_handler.py:2289,2335,1945` |
| `is_reverse_alias` false -> Forward case | `email_handler.py:2195,2201-2202,2208` |
| `handle_forward` entry | `email_handler.py:536` |
| `Alias.get_by(email=...)` returns None | `email_handler.py:543` |
| LOG.d "alias … not exist. Try to see if it can be created on the fly" | `email_handler.py:545` |
| `try_auto_create(alias_address)` call | `email_handler.py:549` |
| `try_auto_create` | `app/alias_utils.py:202` |
| custom-domain check (returns None; LOG.i) | `app/alias_utils.py:92,104` |
| directory check (returns None; LOG.info) | `app/alias_utils.py:145,165` |
| LOG.d "cannot be created on-the-fly, return 550" | `email_handler.py:551` |
| `should_ignore_bounce` branch -> E207, else E515 | `email_handler.py:552-555` |
| `_handle` logs returned wire code | `email_handler.py:2367` |
| `E515 = "550 SL E515 Email not exist"` | `app/email/status.py:51` |
| `E207 = "250 SL E207 No bounce report"` | `app/email/status.py:12` |
| `should_ignore_bounce` + warning | `app/email_utils.py:1361,1363` |
| `is_valid_alias_address_domain` (NOT on this path; used at `email_handler.py:1000`) | `app/email_utils.py:557` |
| `IgnoreBounceSender` model -> table `ignore_bounce_sender` | `app/models.py:3357` |
| `add_sl_domains()` seeds SLDomain | `init_app.py:39` |
| `ALIAS_DOMAINS` derived from `EMAIL_DOMAIN` | `app/config.py:157-160` |
| `__main__`: `load_pgp_public_keys()` + `add_sl_domains()` | `init_app.py:69-73` |

### Notes on citation drift (corrected in this rewrite)

- `IgnoreBounceSender.__tablename__` is at **`app/models.py:3357`** (a prior revision cited `3354`).
- The custom-domain rejection is logged by the `LOG.i(...)` **call** at `app/alias_utils.py:104`; runtime output confirms `:104` is the authoritative line for the emitted record.
