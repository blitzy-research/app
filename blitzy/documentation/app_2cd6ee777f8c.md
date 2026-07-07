# SimpleLogin Startup Investigation — Run-First Q&A

**Branch:** `app_2cd6ee777f8c`
**Scope:** Read-only investigation of SimpleLogin's startup behavior. No source file was modified or deleted; the only artifact produced is this document. Temporary observation scripts and throwaway databases were created for observation and then removed — the repository is unchanged except for this file (see [§(f) Cleanup / repository integrity](#f-cleanup--repository-integrity), which shows the final `git status`).

This document answers three coupled runtime-behavior questions, plus an overarching startup-order / failure-mode narrative, **from direct first-hand observation**. Every behavioral claim is backed by **(1)** a `file:line` citation into the SimpleLogin source and **(2)** the actual, unedited runtime output captured beside the claim.

> **Reading the captured logs — everything below is verbatim.** The investigation was run in the project's canonical layout: the repository is checked out at **`/code`** (the image's working directory — `Dockerfile:3` and `Dockerfile:18` both `WORKDIR /code`) and the interpreter is the canonical **Python 3.10** whose standard library lives at **`/usr/local/lib/python3.10`** (`Dockerfile:8` `FROM python:3.10`). Because the software was built and run at these canonical paths, the paths that appear in the log lines below (`/code/...`, `/usr/local/lib/python3.10/...`, and the in-project virtualenv `/code/.venv/...`) are the software's own real output. **Nothing in any pasted block is redacted, paraphrased, or elided** — the full traceback (Q1), the full 255-step migration (Q3), and the full `--help` text (Q2) are reproduced in their entirety.
>
> SimpleLogin's logger (`app/log.py:79` `LOG = _get_logger("SL")`) writes to STDOUT (`app/log.py:41` `logging.StreamHandler(sys.stdout)`) at DEBUG level (`app/log.py:51`) using the fixed format at `app/log.py:12-14`:
> `%(asctime)s - %(name)s - %(levelname)s - %(process)d - "%(pathname)s:%(lineno)d" - %(funcName)s() - %(message_id)s - %(message)s`.
>
> Every service prints an identical six-line import-time banner (`load config file …`, `>>> URL: …`, the `MAX_NB_EMAIL_FREE_PLAN` default note, `Paddle param not set`, the GNUPGHOME warning, `Upload files to local dir`) immediately followed by `>>> init logging <<<` (`app/log.py:67`). One of those six banner lines is `WARNING: Use a temp directory for GNUPGHOME /tmp/<random>` — this is the **application's own** startup output, not a machine artifact: `app/config.py:252-262` uses `$GNUPGHOME` if that variable is set, otherwise it generates a random `/tmp/<20-letters>` directory and `print`s that exact warning (`app/config.py:262`). It is shown verbatim wherever it appears.

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

The investigation was performed in the canonical runtime — **Python 3.10** (`Dockerfile:8` `FROM python:3.10`; `pyproject.toml:61` `python = "^3.10"`; Python 3.12 is explicitly unsupported per `CONTRIBUTING.md:236` — the repo's own comment reads *"we haven't managed to make python 3.12 work"*) with **PostgreSQL 13** and the SimpleLogin dependencies pinned in `poetry.lock`. **Redis is not required** for any of Q1/Q2/Q3 in the default local configuration because the session / rate-limit store is only wired when `MEM_STORE_URI` is set (`server.py:163-165`), and `MEM_STORE_URI` is unset here.

**Interpreter, dependency install, and PostgreSQL — exact commands and output:**

```console
$ cd /code                          # canonical run-root (Dockerfile:3,18 WORKDIR /code)
$ python3.10 --version
Python 3.10.20
$ poetry --version
Poetry (version 1.8.5)
$ poetry check
All set!
$ poetry install --sync --no-root --dry-run | tail -1
Package operations: 0 installs, 1 update, 0 removals, 174 skipped
$ psql -h localhost -p 15432 -U myuser -d postgres -tAc "SHOW server_version;"
13.23 (Debian 13.23-1.pgdg13+1)
```

> **On the dependency install.** `poetry install` (the canonical command, `CONTRIBUTING.md:248`) materializes the locked dependency graph into the in-project virtualenv `/code/.venv`. Running it in `--dry-run --sync` mode against the already-materialized environment reports **`0 installs, 0 removals, 174 skipped`** — i.e. every locked package is already present and satisfied — plus a single **`1 update`** that is a no-op re-link of the C-extension package `pyre2` (same version `0.3.6` → `0.3.6`). In other words, the environment exactly matches `poetry.lock`. Every command in this document runs through that Python 3.10 environment, invoked as `/code/.venv/bin/python <entrypoint>` (equivalently `poetry run python <entrypoint>`), never a host interpreter.

**Configuration (`CONFIG`-driven dotenv).** `app/config.py:65` reads `config_file = os.environ.get("CONFIG")`; if set, `app/config.py:68` `print`s `load config file <path>` and `app/config.py:69` calls `load_dotenv(...)` — that print is the first line of every banner below. `app/config.py:14-20` (`get_abs_path`) returns absolute paths unchanged, so a dotenv anywhere on disk works. The canonical values come from `example.env`: `URL=http://localhost:7777` (`example.env:6`), `NOT_SEND_EMAIL=true` (`example.env:19`), `EMAIL_DOMAIN=sl.local` (`example.env:22`), `SUPPORT_EMAIL=support@sl.local` (`example.env:40`), `DB_URI=...` (`example.env:75`), `FLASK_SECRET=...` (`example.env:77`). All cryptographic material required at startup already ships under `local_data/` (`jwtRS256.key`, `dkim.key`, PGP keys, `test_words.txt`, …), so a `CONFIG`-driven start has no missing-key failures. The variable is exported once and inherited by every service:

```console
$ export CONFIG=.env                # canonical dotenv (derived from example.env)
```

Three database states are needed to exercise the three questions in isolation, so three `DB_URI`s were used (only `DB_URI` differs from the canonical dotenv; every other key is unchanged):

| Scenario | Database | State | Config file |
|----------|----------|-------|-------------|
| Q1 | `sl_q1_empty` | created, **no migrations** (empty schema) | `/code/q1.env` (temp) |
| Q2 | `simplelogin` | migrated **and** seeded (`init_app.py` run) | `/code/.env` |
| Q3 | `sl_q3` | migrated, **`init_app.py` NOT run** (empty `SLDomain`) | `/code/q3.env` (temp) |

```console
# create the empty (Q1) and migrated-but-unseeded (Q3) databases in the running PostgreSQL
$ psql -h localhost -p 15432 -U myuser -d postgres -c "CREATE DATABASE sl_q1_empty OWNER myuser;"
CREATE DATABASE
$ psql -h localhost -p 15432 -U myuser -d postgres -c "CREATE DATABASE sl_q3 OWNER myuser;"
CREATE DATABASE

# derive the two temporary dotenvs from the canonical .env, changing ONLY DB_URI
$ sed 's#^DB_URI=.*#DB_URI=postgresql://myuser:mypassword@localhost:15432/sl_q1_empty#' .env > /code/q1.env
$ sed 's#^DB_URI=.*#DB_URI=postgresql://myuser:mypassword@localhost:15432/sl_q3#'       .env > /code/q3.env
```

Confirming the default local config uses **cookie sessions (no Redis, no extra PG session store)**:

```console
$ grep -c 'MEM_STORE_URI' .env
0        # MEM_STORE_URI unset -> the server.py:163-165 branch is skipped
```

> **Note on demonstrating port binding.** This image ships neither `ss` nor `lsof`, so port binding is demonstrated three ways: **(1) functionally**, via a real HTTP request to `:7777` / a real SMTP transaction to `:20381`; **(2) at the kernel level**, by decoding `/proc/net/tcp` — state `0A` is `TCP_LISTEN`, and the listening socket's inode is matched back to the owning PID(s) via `/proc/<pid>/fd`; and **(3) via `ps`**, showing the exact owning process(es). The small read-only helper used for (2) (`portproof_all.py`) was a temporary observation script, removed during cleanup (§(f)).

---

## (b) Q1 — Empty-database startup exception

**Question.** With PostgreSQL running but the target DB schema **empty** (migrations deliberately not run), start the web app with `python server.py`, open the login page, and capture the **full Python exception / traceback** raised.

**Answer (summary).** The server starts and binds `:7777` cleanly against the empty schema; the failure appears only at **request time**. Submitting the login form fires the first ORM query, which raises **`sqlalchemy.exc.ProgrammingError`** wrapping **`psycopg2.errors.UndefinedTable: relation "users" does not exist`**, and the request returns **HTTP 500**. The offending relation observed is **`users`**.

### Mechanism (cause → effect), with `file:line`

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

4. **A bare anonymous GET renders with no DB query; the login *POST* is the reliable first ORM query.** `/` redirects anonymous users to `auth.login` (`server.py:250-256`); the login GET view (`app/auth/views/login.py:21`) renders the template chain `templates/auth/login.html` → `templates/single.html` → `templates/base.html`, whose only `current_user` reference short-circuits for anonymous users (`templates/base.html:89`). On **POST**, `app/auth/views/login.py:40` `if form.validate_on_submit():` becomes true and `app/auth/views/login.py:43` executes:
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

$ CONFIG=/code/q1.env /code/.venv/bin/python server.py
```

**Startup stdout/stderr** — the server binds `:7777` against the empty DB. `python server.py` runs with `debug=True`, so Werkzeug's reloader is active: the reloader **parent** process prints the banner and the `* Serving Flask app` lines, then re-executes the module as a **child** worker that prints the banner again (note the two different PIDs). The complete, unedited startup output is:

```
load config file /code/q1.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/gzywsmgbdphchglwhxtj
Upload files to local dir
>>> init logging <<<
2026-07-07 00:46:37,120 - SL - DEBUG - 104702 - "/code/app/utils.py:17" - <module>() -  - load words file: /code/local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
load config file /code/q1.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/ftqxmceycnmfexnzufws
Upload files to local dir
>>> init logging <<<
2026-07-07 00:46:38,904 - SL - DEBUG - 104718 - "/code/app/utils.py:17" - <module>() -  - load words file: /code/local_data/test_words.txt
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
 104702  104700 Ss   /code/.venv/bin/python server.py
 104718  104702 R    /code/.venv/bin/python /code/server.py
$ python3.10 portproof_all.py 7777
LISTEN 127.0.0.1:7777  inode=526223071  pid=104702  ppid=104700  cmd=(python server.py)
LISTEN 127.0.0.1:7777  inode=526223071  pid=104718  ppid=104702  cmd=(python server.py)
```

The listening socket (`inode` shared by both PIDs, matched via `/proc/<pid>/fd`) is in state `LISTEN` on `127.0.0.1:7777` — proving the server bound `:7777` **despite the empty schema**. The functional trigger (a real HTTP client) confirms it is serving: a GET fetches the login page (`200`), the login **POST** returns **`500`**, and the anonymous `/` returns a `302` redirect to `/auth/login` (which needs no DB, `server.py:250-256`):

```console
$ curl -s -c q1b_cookies.txt http://127.0.0.1:7777/auth/login -o q1b_login.html -w "GET /auth/login -> %{http_code}\n"
GET /auth/login -> 200
$ CSRF=<token extracted from login page>
$ curl -s -b q1b_cookies.txt --data-urlencode "csrf_token=$CSRF" --data-urlencode "email=test@example.com" --data-urlencode "password=whatever123" -o /dev/null -w "POST /auth/login -> %{http_code}\n" http://127.0.0.1:7777/auth/login
POST /auth/login -> 500
$ curl -s -o /dev/null -D - http://127.0.0.1:7777/ | grep -iE "^HTTP|^location"
HTTP/1.0 302 FOUND
Location: http://127.0.0.1:7777/auth/login
```

**The complete, unedited server-side output** for the first login attempt (the GET that rendered `200`, then `error_handler` at `server.py:390`, the full chained traceback, the `after_request` `500` log line at `server.py:284`, and the subsequent anonymous `GET /` `302`):

```
2026-07-07 00:46:40,107 - SL - DEBUG - 104718 - "/code/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.10113525390625
2026-07-07 00:46:40,130 - SL - ERROR - 104718 - "/code/server.py:390" - error_handler() -  - (psycopg2.errors.UndefinedTable) relation "users" does not exist
LINE 2: FROM users 
             ^

[SQL: SELECT users.directory_quota AS users_directory_quota, users.subdomain_quota AS users_subdomain_quota, users.password AS users_password, users.id AS users_id, users.created_at AS users_created_at, users.updated_at AS users_updated_at, users.email AS users_email, users.name AS users_name, users.is_admin AS users_is_admin, users.alias_generator AS users_alias_generator, users.notification AS users_notification, users.activated AS users_activated, users.disabled AS users_disabled, users.profile_picture_id AS users_profile_picture_id, users.otp_secret AS users_otp_secret, users.enable_otp AS users_enable_otp, users.last_otp AS users_last_otp, users.fido_uuid AS users_fido_uuid, users.default_alias_custom_domain_id AS users_default_alias_custom_domain_id, users.default_alias_public_domain_id AS users_default_alias_public_domain_id, users.lifetime AS users_lifetime, users.paid_lifetime AS users_paid_lifetime, users.lifetime_coupon_id AS users_lifetime_coupon_id, users.trial_end AS users_trial_end, users.default_mailbox_id AS users_default_mailbox_id, users.sender_format AS users_sender_format, users.sender_format_updated_at AS users_sender_format_updated_at, users.replace_reverse_alias AS users_replace_reverse_alias, users.referral_id AS users_referral_id, users.intro_shown AS users_intro_shown, users.max_spam_score AS users_max_spam_score, users.newsletter_alias_id AS users_newsletter_alias_id, users.include_sender_in_reverse_alias AS users_include_sender_in_reverse_alias, users.random_alias_suffix AS users_random_alias_suffix, users.expand_alias_info AS users_expand_alias_info, users.ignore_loop_email AS users_ignore_loop_email, users.alternative_id AS users_alternative_id, users.disable_automatic_alias_note AS users_disable_automatic_alias_note, users.one_click_unsubscribe_block_sender AS users_one_click_unsubscribe_block_sender, users.include_website_in_one_click_alias AS users_include_website_in_one_click_alias, users.disable_import AS users_disable_import, users.can_use_phone AS users_can_use_phone, users.phone_quota AS users_phone_quota, users.block_behaviour AS users_block_behaviour, users.include_header_email_header AS users_include_header_email_header, users.enable_data_breach_check AS users_enable_data_breach_check, users.flags AS users_flags, users.unsub_behaviour AS users_unsub_behaviour, users.delete_on AS users_delete_on 
FROM users 
WHERE users.email = %(email_1)s 
 LIMIT %(param_1)s]
[parameters: {'email_1': 'test@example.com', 'param_1': 1}]
(Background on this error at: http://sqlalche.me/e/13/f405)
Traceback (most recent call last):
  File "/code/.venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1276, in _execute_context
    self.dialect.do_execute(
  File "/code/.venv/lib/python3.10/site-packages/sqlalchemy/engine/default.py", line 608, in do_execute
    cursor.execute(statement, parameters)
psycopg2.errors.UndefinedTable: relation "users" does not exist
LINE 2: FROM users 
             ^


The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "/code/.venv/lib/python3.10/site-packages/flask/app.py", line 1950, in full_dispatch_request
    rv = self.dispatch_request()
  File "/code/.venv/lib/python3.10/site-packages/flask_debugtoolbar/__init__.py", line 125, in dispatch_request
    return view_func(**req.view_args)
  File "/usr/local/lib/python3.10/cProfile.py", line 110, in runcall
    return func(*args, **kw)
  File "/code/.venv/lib/python3.10/site-packages/flask_limiter/extension.py", line 702, in __inner
    return obj(*a, **k)
  File "/code/app/auth/views/login.py", line 43, in login
    user = User.get_by(email=email) or User.get_by(email=canonical_email)
  File "/code/app/models.py", line 84, in get_by
    return Session.query(cls).filter_by(**kw).first()
  File "/code/.venv/lib/python3.10/site-packages/sqlalchemy/orm/query.py", line 3429, in first
    ret = list(self[0:1])
  File "/code/.venv/lib/python3.10/site-packages/sqlalchemy/orm/query.py", line 3203, in __getitem__
    return list(res)
  File "/code/.venv/lib/python3.10/site-packages/sqlalchemy/orm/query.py", line 3535, in __iter__
    return self._execute_and_instances(context)
  File "/code/.venv/lib/python3.10/site-packages/sqlalchemy/orm/query.py", line 3560, in _execute_and_instances
    result = conn.execute(querycontext.statement, self._params)
  File "/code/.venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1011, in execute
    return meth(self, multiparams, params)
  File "/code/.venv/lib/python3.10/site-packages/sqlalchemy/sql/elements.py", line 298, in _execute_on_connection
    return connection._execute_clauseelement(self, multiparams, params)
  File "/code/.venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1124, in _execute_clauseelement
    ret = self._execute_context(
  File "/code/.venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1316, in _execute_context
    self._handle_dbapi_exception(
  File "/code/.venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1510, in _handle_dbapi_exception
    util.raise_(
  File "/code/.venv/lib/python3.10/site-packages/sqlalchemy/util/compat.py", line 182, in raise_
    raise exception
  File "/code/.venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1276, in _execute_context
    self.dialect.do_execute(
  File "/code/.venv/lib/python3.10/site-packages/sqlalchemy/engine/default.py", line 608, in do_execute
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
2026-07-07 00:46:40,137 - SL - DEBUG - 104718 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 500, takes 0.015317440032958984
2026-07-07 00:46:40,146 - SL - DEBUG - 104718 - "/code/server.py:284" - after_request() -  - 127.0.0.1 GET / ImmutableMultiDict([]) 302, takes 0.0007216930389404297
```

> The `SELECT users.… FROM users …` statement appears **twice** above and both copies are shown verbatim: SQLAlchemy prints the offending SQL once on the wrapped `psycopg2` error and again on the outer `ProgrammingError`. Nothing is elided.

**Exception anatomy (as observed):**

- **Chained cause (inner):** `psycopg2.errors.UndefinedTable: relation "users" does not exist`.
- **Raised (outer):** `sqlalchemy.exc.ProgrammingError: (psycopg2.errors.UndefinedTable) relation "users" does not exist` — joined by Python's *"The above exception was the direct cause of the following exception"*.
- **Offending relation:** `users`.
- **Trigger frame:** `app/auth/views/login.py:43` → `app/models.py:84` (`Session.query(cls).filter_by(**kw).first()`).
- **HTTP status:** `500` (confirmed by both the client `POST /auth/login -> 500` and the `after_request` log `POST /auth/login … 500`, `server.py:284`).
- **SQLAlchemy reference:** `http://sqlalche.me/e/13/f405` (the SQLAlchemy **1.3** error-code URL, matching `SQLAlchemy = 1.3.24` at `pyproject.toml:116`).

### Stability (≥2 runs)

The same empty-schema condition was exercised a second time (a second POST with a different sender email, against the same running server):

```console
$ curl -s -b q1b_cookies.txt --data-urlencode "csrf_token=$CSRF" --data-urlencode "email=second-run@example.com" --data-urlencode "password=whatever123" -o /dev/null -w "POST /auth/login (run2) -> %{http_code}\n" http://127.0.0.1:7777/auth/login
POST /auth/login (run2) -> 500
```

The complete, unedited server-side output for run #2 (`second-run@example.com`) is identical in exception class, offending relation, and HTTP status:

```
2026-07-07 00:46:40,159 - SL - ERROR - 104718 - "/code/server.py:390" - error_handler() -  - (psycopg2.errors.UndefinedTable) relation "users" does not exist
LINE 2: FROM users 
             ^

[SQL: SELECT users.directory_quota AS users_directory_quota, users.subdomain_quota AS users_subdomain_quota, users.password AS users_password, users.id AS users_id, users.created_at AS users_created_at, users.updated_at AS users_updated_at, users.email AS users_email, users.name AS users_name, users.is_admin AS users_is_admin, users.alias_generator AS users_alias_generator, users.notification AS users_notification, users.activated AS users_activated, users.disabled AS users_disabled, users.profile_picture_id AS users_profile_picture_id, users.otp_secret AS users_otp_secret, users.enable_otp AS users_enable_otp, users.last_otp AS users_last_otp, users.fido_uuid AS users_fido_uuid, users.default_alias_custom_domain_id AS users_default_alias_custom_domain_id, users.default_alias_public_domain_id AS users_default_alias_public_domain_id, users.lifetime AS users_lifetime, users.paid_lifetime AS users_paid_lifetime, users.lifetime_coupon_id AS users_lifetime_coupon_id, users.trial_end AS users_trial_end, users.default_mailbox_id AS users_default_mailbox_id, users.sender_format AS users_sender_format, users.sender_format_updated_at AS users_sender_format_updated_at, users.replace_reverse_alias AS users_replace_reverse_alias, users.referral_id AS users_referral_id, users.intro_shown AS users_intro_shown, users.max_spam_score AS users_max_spam_score, users.newsletter_alias_id AS users_newsletter_alias_id, users.include_sender_in_reverse_alias AS users_include_sender_in_reverse_alias, users.random_alias_suffix AS users_random_alias_suffix, users.expand_alias_info AS users_expand_alias_info, users.ignore_loop_email AS users_ignore_loop_email, users.alternative_id AS users_alternative_id, users.disable_automatic_alias_note AS users_disable_automatic_alias_note, users.one_click_unsubscribe_block_sender AS users_one_click_unsubscribe_block_sender, users.include_website_in_one_click_alias AS users_include_website_in_one_click_alias, users.disable_import AS users_disable_import, users.can_use_phone AS users_can_use_phone, users.phone_quota AS users_phone_quota, users.block_behaviour AS users_block_behaviour, users.include_header_email_header AS users_include_header_email_header, users.enable_data_breach_check AS users_enable_data_breach_check, users.flags AS users_flags, users.unsub_behaviour AS users_unsub_behaviour, users.delete_on AS users_delete_on 
FROM users 
WHERE users.email = %(email_1)s 
 LIMIT %(param_1)s]
[parameters: {'email_1': 'second-run@example.com', 'param_1': 1}]
(Background on this error at: http://sqlalche.me/e/13/f405)
Traceback (most recent call last):
  File "/code/.venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1276, in _execute_context
    self.dialect.do_execute(
  File "/code/.venv/lib/python3.10/site-packages/sqlalchemy/engine/default.py", line 608, in do_execute
    cursor.execute(statement, parameters)
psycopg2.errors.UndefinedTable: relation "users" does not exist
LINE 2: FROM users 
             ^


The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "/code/.venv/lib/python3.10/site-packages/flask/app.py", line 1950, in full_dispatch_request
    rv = self.dispatch_request()
  File "/code/.venv/lib/python3.10/site-packages/flask_debugtoolbar/__init__.py", line 125, in dispatch_request
    return view_func(**req.view_args)
  File "/usr/local/lib/python3.10/cProfile.py", line 110, in runcall
    return func(*args, **kw)
  File "/code/.venv/lib/python3.10/site-packages/flask_limiter/extension.py", line 702, in __inner
    return obj(*a, **k)
  File "/code/app/auth/views/login.py", line 43, in login
    user = User.get_by(email=email) or User.get_by(email=canonical_email)
  File "/code/app/models.py", line 84, in get_by
    return Session.query(cls).filter_by(**kw).first()
  File "/code/.venv/lib/python3.10/site-packages/sqlalchemy/orm/query.py", line 3429, in first
    ret = list(self[0:1])
  File "/code/.venv/lib/python3.10/site-packages/sqlalchemy/orm/query.py", line 3203, in __getitem__
    return list(res)
  File "/code/.venv/lib/python3.10/site-packages/sqlalchemy/orm/query.py", line 3535, in __iter__
    return self._execute_and_instances(context)
  File "/code/.venv/lib/python3.10/site-packages/sqlalchemy/orm/query.py", line 3560, in _execute_and_instances
    result = conn.execute(querycontext.statement, self._params)
  File "/code/.venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1011, in execute
    return meth(self, multiparams, params)
  File "/code/.venv/lib/python3.10/site-packages/sqlalchemy/sql/elements.py", line 298, in _execute_on_connection
    return connection._execute_clauseelement(self, multiparams, params)
  File "/code/.venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1124, in _execute_clauseelement
    ret = self._execute_context(
  File "/code/.venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1316, in _execute_context
    self._handle_dbapi_exception(
  File "/code/.venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1510, in _handle_dbapi_exception
    util.raise_(
  File "/code/.venv/lib/python3.10/site-packages/sqlalchemy/util/compat.py", line 182, in raise_
    raise exception
  File "/code/.venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1276, in _execute_context
    self.dialect.do_execute(
  File "/code/.venv/lib/python3.10/site-packages/sqlalchemy/engine/default.py", line 608, in do_execute
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
2026-07-07 00:46:40,160 - SL - DEBUG - 104718 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 500, takes 0.006679534912109375
```

Result: **stable** — HTTP 500 with `ProgrammingError` / `UndefinedTable: relation "users" does not exist` on every run.

**Q1 ↔ omitted step:** this failure is caused by **skipping the migration step**. The engine connects at import (`app/db.py:12`) so the server starts, but the schema has no `users` table, so the first ORM query fails at request time.

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
$ CONFIG=.env /code/.venv/bin/python server.py
load config file /code/.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/haujnvkeaacrtyhnhvts
Upload files to local dir
>>> init logging <<<
2026-07-07 00:48:15,052 - SL - DEBUG - 105155 - "/code/app/utils.py:17" - <module>() -  - load words file: /code/local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
load config file /code/.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/npofjeobbmqltuuunmuk
Upload files to local dir
>>> init logging <<<
2026-07-07 00:48:16,834 - SL - DEBUG - 105172 - "/code/app/utils.py:17" - <module>() -  - load words file: /code/local_data/test_words.txt
```

Process tree, kernel-level bind proof, and functional proof (the `* Running on …` line is suppressed by the disabled Werkzeug logger, `app/log.py:69-71`, as explained in §(b)):

```console
$ ps -o pid,ppid,stat,cmd --no-headers -C python | grep server.py
 105155       1 Ss   /code/.venv/bin/python server.py
 105172  105155 R    /code/.venv/bin/python /code/server.py
$ python3.10 portproof_all.py 7777
LISTEN 127.0.0.1:7777  inode=526368340  pid=105155  ppid=1  cmd=(python server.py)
LISTEN 127.0.0.1:7777  inode=526368340  pid=105172  ppid=105155  cmd=(python server.py)
$ curl -s -m 5 -o /dev/null -D - http://127.0.0.1:7777/ | grep -iE "^HTTP|^location"
HTTP/1.0 302 FOUND
Location: http://127.0.0.1:7777/auth/login
$ curl -s -m 5 -w " HTTP=%{http_code}\n" http://127.0.0.1:7777/health
success HTTP=200
```

The `LISTEN 127.0.0.1:7777 … (python server.py)` rows (parent + child sharing one socket inode), the `302 → /auth/login` redirect, and `/health → 200` together prove the process is **listening and serving on `:7777`**. (Production equivalent: `wsgi.py` exposes `app = create_app()` — `wsgi.py:1,3` — served by Gunicorn per `pyproject.toml:66` `gunicorn = "^20.0.4"`.)

### Service 2 — Email handler (`python email_handler.py`, binds `:20381`)

`email_handler.py:2381` `def main(port)`; `:2383` `Controller(MailHandler(), hostname="0.0.0.0", port=port)`; `:2385` `controller.start()`; `:2386` logs the bound controller; the `__main__` argparse default port is **20381** (`email_handler.py:2399`); `:2403` logs the listen port; `:2404` calls `main`. Startup stdout/stderr with the **exact port-binding lines** (note the literal port `20381` in both):

```console
$ CONFIG=.env /code/.venv/bin/python email_handler.py
load config file /code/.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/jcitxbwtmkjzgvmggipj
Upload files to local dir
>>> init logging <<<
2026-07-07 00:48:19,738 - SL - DEBUG - 105193 - "/code/app/utils.py:17" - <module>() -  - load words file: /code/local_data/test_words.txt
2026-07-07 00:48:20,443 - SL - INFO - 105193 - "/code/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-07 00:48:20,445 - SL - DEBUG - 105193 - "/code/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

Process and kernel-level bind proof — the aiosmtpd controller is a single process (no reloader) `LISTEN`ing on `0.0.0.0:20381`:

```console
$ ps -o pid,ppid,stat,cmd --no-headers -C python | grep email_handler.py
 105193       1 Ssl  /code/.venv/bin/python email_handler.py
$ python3.10 portproof_all.py 20381
LISTEN 0.0.0.0:20381  inode=526426661  pid=105193  ppid=1  cmd=(python email_handler.py)
```

Production SMTP port is `:25`.

### Service 3 — Job runner (`python job_runner.py`, worker; no port)

`job_runner.py:329` `if __name__ == "__main__":` enters a `while True:` loop that takes pending jobs and `job_runner.py:347` `time.sleep(10)` between polls. It binds no port. Its startup output is the standard import banner, after which it polls silently until a `Job` exists:

```console
$ CONFIG=.env /code/.venv/bin/python job_runner.py
load config file /code/.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/sqjjnnxnrhcwhdbsqsul
Upload files to local dir
>>> init logging <<<
2026-07-07 00:48:23,145 - SL - DEBUG - 105214 - "/code/app/utils.py:17" - <module>() -  - load words file: /code/local_data/test_words.txt
```

The process was confirmed alive and idle in its polling loop (no port, no further output until a job arrives):

```console
$ ps -o pid,ppid,stat,etime,cmd --no-headers -C python | grep job_runner.py
 105214       1 Ss         00:04 /code/.venv/bin/python job_runner.py
```

### Service 4 — Event listener (`python event_listener.py listener`, PG LISTEN; no port)

`event_listener.py:29` `def main(mode, …)`; with the `listener` sub-command, `event_listener.py:34` logs `Using PostgresEventSource` and `:35` constructs `PostgresEventSource(...)`. (A bare `python event_listener.py` with no sub-command prints `Invalid usage. Pass a valid subcommand as argument` and exits — `event_listener.py:96`.) Startup stdout/stderr:

```console
$ CONFIG=.env /code/.venv/bin/python event_listener.py listener
load config file /code/.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/fonuccyvqggmahntqyci
Upload files to local dir
>>> init logging <<<
2026-07-07 00:48:28,135 - SL - DEBUG - 105226 - "/code/app/utils.py:17" - <module>() -  - load words file: /code/local_data/test_words.txt
2026-07-07 00:48:28,252 - SL - INFO - 105226 - "/code/event_listener.py:34" - main() -  - Using PostgresEventSource
2026-07-07 00:48:28,256 - SL - INFO - 105226 - "/code/event_listener.py:43" - main() -  - Starting with HttpEventSink
2026-07-07 00:48:28,256 - SL - INFO - 105226 - "/code/events/event_source.py:49" - __listen() -  - Starting to listen to events
```

```console
$ ps -o pid,ppid,stat,etime,cmd --no-headers -C python | grep event_listener.py
 105226       1 Ss         00:04 /code/.venv/bin/python event_listener.py listener
```

### Service 5 — Scheduler / cron (`python cron.py`, batch; no port)

`cron.py:1` `import argparse`; `cron.py:1262` `if __name__ == "__main__":` logs `Start running cronjob` (`cron.py:1263`) and dispatches on `args.job` (`cron.py:1274` onward — e.g. `stats`, `notify_trial_end`, `sanity_check`, …). Startup stdout/stderr (no `-j`, so `args.job` is `None` and no maintenance job runs):

```console
$ CONFIG=.env /code/.venv/bin/python cron.py
load config file /code/.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/uhxrgtnfmmpbjjpkhdxh
Upload files to local dir
>>> init logging <<<
2026-07-07 00:48:33,159 - SL - DEBUG - 105233 - "/code/app/utils.py:17" - <module>() -  - load words file: /code/local_data/test_words.txt
2026-07-07 00:48:34,047 - SL - DEBUG - 105233 - "/code/cron.py:1263" - <module>() -  - Start running cronjob
```

The full `--help` (the module still prints its banner and `Start running cronjob` line on import, *then* argparse prints usage and exits — shown complete, nothing elided):

```console
$ CONFIG=.env /code/.venv/bin/python cron.py --help
load config file /code/.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/bfykncghburlvlhbuvwo
Upload files to local dir
>>> init logging <<<
2026-07-07 00:48:34,957 - SL - DEBUG - 105239 - "/code/app/utils.py:17" - <module>() -  - load words file: /code/local_data/test_words.txt
2026-07-07 00:48:35,812 - SL - DEBUG - 105239 - "/code/cron.py:1263" - <module>() -  - Start running cronjob
usage: cron.py [-h] [-j JOB]

options:
  -h, --help         show this help message and exit
  -j JOB, --job JOB  Choose a cron job to run
```

(A concrete job is selected with `-j <job>`.)

### Stability (≥2 runs)

The two network ports are deterministic constants, not dynamically assigned:

- **`:7777`** — hard-coded in `server.py:588`; observed bound in both the Q1 run (empty DB) and this Q2 run.
- **`:20381`** — the argparse default in `email_handler.py:2399`; observed bound in this Q2 run **and** again in the Q3 run below, both logging `Listen for port 20381` / `Start mail controller 0.0.0.0 20381`.

Result: **stable** — ports 7777 and 20381 every time.

---


## (d) Q3 — Skipped-initialization SMTP rejection

**Question.** Run migrations but do **not** run `init_app.py` (leaving `SLDomain` empty — no email domains configured). Start the email handler, send a message to `x@sl.local`, and capture both the SMTP status code / wire reply returned to the sender **and** the log line(s) explaining the rejection.

**Answer (summary).** With `SLDomain` empty, mail to `x@sl.local` is rejected with the permanent SMTP reply **`550 SL E515 Email not exist`** (`app/email/status.py:51`), and the handler logs two `LOG.d` lines — `alias x@sl.local not exist…` (`email_handler.py:545`) and `alias x@sl.local cannot be created on-the-fly, return 550` (`email_handler.py:551`) — around two `LOG.i` lines that record *why* on-the-fly creation failed (no custom domain for `sl.local`, `app/alias_utils.py:104`; no directory separator, `app/alias_utils.py:165`). The edge case where the sender is an `IgnoreBounceSender` instead returns **`250 SL E207 No bounce report`** (`app/email/status.py:12`).

> **Important — real table name.** The `SLDomain` model maps to the table **`public_domain`**, not `sl_domain` (`app/models.py:3116` `class SLDomain(...)`, `__tablename__ = "public_domain"`). Row-count evidence below therefore queries `public_domain`.

### `SLDomain` state — before / after `init_app.py`

The `sl_q3` database was migrated (`alembic upgrade head`) but `init_app.py` was **not** run. `alembic upgrade head` applied all **255** revisions. The **complete, unedited** migration output (the six-line import banner from Alembic loading the app config, the two Alembic runtime-info lines, and all 255 `Running upgrade` steps — nothing elided) is:

```console
$ CONFIG=/code/q3.env /code/.venv/bin/alembic upgrade head
load config file /code/q3.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/qhcvwmesysqtaleyvrmq
Upload files to local dir
>>> init logging <<<
2026-07-07 00:58:37,644 - SL - DEBUG - 108276 - "/code/app/utils.py:17" - <module>() -  - load words file: /code/local_data/test_words.txt
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
$ echo "exit=$?"
exit=0
```

**BEFORE `init_app.py` — `SLDomain` (table `public_domain`) is empty** (both raw SQL and the ORM accessor agree):

```console
$ psql -h localhost -p 15432 -U myuser -d sl_q3 -c "SELECT count(*) AS public_domain_rows FROM public_domain;"
 public_domain_rows 
--------------------
                  0
(1 row)


$ CONFIG=/code/q3.env python -c "from app.models import SLDomain; print('SLDomain.count() =', SLDomain.count())"
load config file /code/q3.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/tvwauguelloxpuahcdns
Upload files to local dir
>>> init logging <<<
2026-07-07 00:58:39,475 - SL - DEBUG - 108284 - "/code/app/utils.py:17" - <module>() -  - load words file: /code/local_data/test_words.txt
SLDomain.count() = 0
```

The Q3 rejection observations below were all captured in exactly this state (`SLDomain` genuinely empty). **AFTER `init_app.py`** — `add_sl_domains()` (`init_app.py:39`) seeds the domain (with `EMAIL_DOMAIN=sl.local`, `ALIAS_DOMAINS` defaults to `[sl.local]` per `app/config.py:157-160`); the `__main__` block (`init_app.py:69-73`) runs `load_pgp_public_keys()` then `add_sl_domains()`:

```console
$ CONFIG=/code/q3.env /code/.venv/bin/python init_app.py
load config file /code/q3.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/kcqhkaibjlryyerorvyl
Upload files to local dir
>>> init logging <<<
2026-07-07 00:58:47,373 - SL - DEBUG - 108314 - "/code/app/utils.py:17" - <module>() -  - load words file: /code/local_data/test_words.txt
2026-07-07 00:58:48,338 - SL - DEBUG - 108314 - "/code/init_app.py:36" - load_pgp_public_keys() -  - Finish load_pgp_public_keys
2026-07-07 00:58:48,340 - SL - INFO - 108314 - "/code/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain

$ psql -h localhost -p 15432 -U myuser -d sl_q3 -c "SELECT id, domain, use_as_reverse_alias FROM public_domain;"
 id |  domain  | use_as_reverse_alias 
----+----------+----------------------
  1 | sl.local | t
(1 row)

```

State transition: **`public_domain` 0 rows → 1 row (`sl.local`)**, caused solely by `init_app.py`'s `add_sl_domains()`.

### Rejection mechanism (cause → effect), with `file:line`

1. **Routing to the Forward case.** `handle_DATA` (`email_handler.py:2289`) → `_handle` (`email_handler.py:2335`) → `handle` (`email_handler.py:1945`). For each recipient, `email_handler.py:2195` `if is_reverse_alias(rcpt_to):` is **false** for `x@sl.local` (it does not start with a reverse-alias prefix), so control falls to the `else: # Forward case` (`email_handler.py:2201`), logs `Forward phase …` (`email_handler.py:2202`), and calls `handle_forward(envelope, copy_msg, rcpt_to)` (`email_handler.py:2208`).
2. **Alias lookup fails.** In `handle_forward` (`email_handler.py:536`), `email_handler.py:543` `alias = Alias.get_by(email=alias_address)` returns `None`, so the `LOG.d(...)` at `email_handler.py:545` emits *"alias … not exist. Try to see if it can be created on the fly"*.
3. **On-the-fly auto-creation fails — via two checks, neither of which is `is_valid_alias_address_domain`.** `email_handler.py:549` `alias = try_auto_create(alias_address)` calls `try_auto_create` (`app/alias_utils.py:202`), which tries two strategies in order:
   - **Custom-domain strategy.** `try_auto_create` calls `try_auto_create_via_domain` (`app/alias_utils.py:274`), which first calls `check_if_alias_can_be_auto_created_for_custom_domain` (`app/alias_utils.py:92`). Because `sl.local` is not a **verified `CustomDomain`**, that check returns `None` after the `LOG.i` at `app/alias_utils.py:104`: *"Cannot auto-create custom domain alias for x@sl.local because there's no custom domain for sl.local"*.
   - **Directory strategy.** `try_auto_create` then calls `try_auto_create_directory` (`app/alias_utils.py:227`), which calls `check_if_alias_can_be_auto_created_for_a_directory` (`app/alias_utils.py:145`). Because the local part `x` contains no directory separator (`+` / `#` / `/`), that check returns `None` after the `LOG.info` at `app/alias_utils.py:165`: *"Cannot auto-create x@sl.local since it has no directory separator"*.

   With both strategies returning `None`, `try_auto_create` returns `None`. **Note:** `is_valid_alias_address_domain` (`app/email_utils.py:557`) is **not** on this forward / auto-create path. It is imported at `email_handler.py:107` and used only at `email_handler.py:1000` as a *sanity check on an already-existing alias* during a different phase (*"verify alias domain is managed by SimpleLogin"*, returning `status.E503`) — so it is unrelated to the `E515` rejection here.
4. **550 is returned.** `email_handler.py:550` `if not alias:` → the `LOG.d` at `email_handler.py:551` emits *"alias … cannot be created on-the-fly, return 550"*; then `email_handler.py:552-553` `if should_ignore_bounce(envelope.mail_from): return [(True, status.E207)]`, else `email_handler.py:554-555` `return [(False, status.E515)]`. `status.E515 = "550 SL E515 Email not exist"` (`app/email/status.py:51`) is delivered verbatim as the SMTP reply.

### Runtime evidence — primary case (`E515`, normal sender)

The email handler was started against `sl_q3` (empty `SLDomain`) — this is also the **second** observation of the `:20381` bind (Q2 stability):

```console
$ CONFIG=/code/q3.env /code/.venv/bin/python email_handler.py
load config file /code/q3.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/vadxkzlvbvygiwsaxzrz
Upload files to local dir
>>> init logging <<<
2026-07-07 00:58:40,553 - SL - DEBUG - 108286 - "/code/app/utils.py:17" - <module>() -  - load words file: /code/local_data/test_words.txt
2026-07-07 00:58:41,236 - SL - INFO - 108286 - "/code/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-07 00:58:41,237 - SL - DEBUG - 108286 - "/code/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

A temporary SMTP-send helper (removed during cleanup, §(f)) calls `smtplib.SMTP("localhost", 20381).sendmail("someone@example.com", ["x@sl.local"], msg)`:

```console
$ /code/.venv/bin/python send_q3.py someone@example.com
SENDER=someone@example.com RCPT=x@sl.local
RESULT: SMTPDataError -> code=550 msg=b'SL E515 Email not exist'
```

**(a) SMTP wire reply the sender observes:** `smtplib` raises `smtplib.SMTPDataError` with `code=550` and `msg=b'SL E515 Email not exist'` — i.e. the wire reply is the full line **`550 SL E515 Email not exist`**. Per RFC 5321 §4.2.1 a `5yz` reply is a **permanent** negative completion reply (resending the same message yields the same result).

**(b) The handler's rejection log lines** (the complete, unedited STDOUT block for that message; the two requested `LOG.d` lines are `email_handler.py:545` and `:551`):

```
2026-07-07 00:58:42,366 - SL - DEBUG - 108286 - "/code/app/log.py:24" - set_message_id() -  - set message_id e0dd0278-f7c0-44f0-95dd-8a20b14da140
2026-07-07 00:58:42,366 - SL - DEBUG - 108286 - "/code/email_handler.py:2342" - _handle() - e0dd0278-f7c0-44f0-95dd-8a20b14da140 - ====>=====>====>====>====>====>====>====>
2026-07-07 00:58:42,366 - SL - INFO - 108286 - "/code/email_handler.py:2343" - _handle() - e0dd0278-f7c0-44f0-95dd-8a20b14da140 - New message, mail from someone@example.com, rctp tos ['x@sl.local'] 
2026-07-07 00:58:42,367 - SL - DEBUG - 108286 - "/code/email_handler.py:1963" - handle() - e0dd0278-f7c0-44f0-95dd-8a20b14da140 - Cannot parse Postfix queue ID from None None
2026-07-07 00:58:42,518 - SL - DEBUG - 108286 - "/code/email_handler.py:1980" - handle() - e0dd0278-f7c0-44f0-95dd-8a20b14da140 - ==>> Handle mail_from:someone@example.com, rcpt_tos:['x@sl.local'], header_from:someone@example.com, header_to:x@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'someone@example.com'), ('To', 'x@sl.local'), ('Subject', 'Q3 test'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:['SIZE=179'], rcpt_options:[]
2026-07-07 00:58:42,523 - SL - DEBUG - 108286 - "/code/email_handler.py:2202" - handle() - e0dd0278-f7c0-44f0-95dd-8a20b14da140 - Forward phase someone@example.com(someone@example.com) -> x@sl.local
2026-07-07 00:58:42,533 - SL - DEBUG - 108286 - "/code/email_handler.py:545" - handle_forward() - e0dd0278-f7c0-44f0-95dd-8a20b14da140 - alias x@sl.local not exist. Try to see if it can be created on the fly
2026-07-07 00:58:42,542 - SL - INFO - 108286 - "/code/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - e0dd0278-f7c0-44f0-95dd-8a20b14da140 - Cannot auto-create custom domain alias for x@sl.local because there's no custom domain for sl.local
2026-07-07 00:58:42,542 - SL - INFO - 108286 - "/code/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - e0dd0278-f7c0-44f0-95dd-8a20b14da140 - Cannot auto-create x@sl.local since it has no directory separator
2026-07-07 00:58:42,542 - SL - DEBUG - 108286 - "/code/email_handler.py:551" - handle_forward() - e0dd0278-f7c0-44f0-95dd-8a20b14da140 - alias x@sl.local cannot be created on-the-fly, return 550
2026-07-07 00:58:42,543 - SL - INFO - 108286 - "/code/email_handler.py:2367" - _handle() - e0dd0278-f7c0-44f0-95dd-8a20b14da140 - Finish mail_from someone@example.com, rcpt_tos ['x@sl.local'], takes 0.1776566505432129 seconds with return code '550 SL E515 Email not exist'<<===
```

The two rejection log lines requested by the question, verbatim:

- `alias x@sl.local not exist. Try to see if it can be created on the fly` — `email_handler.py:545`.
- `alias x@sl.local cannot be created on-the-fly, return 550` — `email_handler.py:551`.

The final `_handle` line (`email_handler.py:2367`) independently confirms the returned wire string: `return code '550 SL E515 Email not exist'`. The two intervening `LOG.i` lines — `app/alias_utils.py:104` (`check_if_alias_can_be_auto_created_for_custom_domain`) and `app/alias_utils.py:165` (`check_if_alias_can_be_auto_created_for_a_directory`) — show *why* auto-create failed: no custom domain for `sl.local`, and no directory separator in the local part.

### Runtime evidence — edge case (`E207`, IgnoreBounceSender)

`should_ignore_bounce(mail_from)` (`app/email_utils.py:1361`) returns `True` only when `IgnoreBounceSender.get_by(mail_from=mail_from)` matches, logging a warning (`app/email_utils.py:1363`). To exercise this branch, a temporary `IgnoreBounceSender` row (table `ignore_bounce_sender`, `app/models.py:3354`) was inserted into the throwaway `sl_q3` DB (`SLDomain` still empty), then a message was sent **from that sender**:

```console
$ psql -h localhost -p 15432 -U myuser -d sl_q3 -c "INSERT INTO ignore_bounce_sender (mail_from, created_at) VALUES ('bounce-sender@example.com', now());"
INSERT 0 1
$ psql -h localhost -p 15432 -U myuser -d sl_q3 -tAc "SELECT count(*) FROM public_domain;"   # SLDomain still empty
0

$ /code/.venv/bin/python send_q3.py bounce-sender@example.com
SENDER=bounce-sender@example.com RCPT=x@sl.local
RESULT: sendmail returned normally (no exception) -> ACCEPTED
```

**Wire reply:** the sender sees a **`250`** (acceptance) — `smtplib.sendmail` returns normally with no exception. The complete, unedited handler log for that message shows the alternate branch taken (`should_ignore_bounce`, `app/email_utils.py:1363`) and the `250 … E207 …` return code:

```
2026-07-07 00:58:44,741 - SL - DEBUG - 108286 - "/code/app/log.py:24" - set_message_id() - e47ff94b-59bb-4f96-ab1a-60023129fc36 - set message_id 5b22a7dc-8628-4dfc-857f-0feadc7ab4c9
2026-07-07 00:58:44,741 - SL - DEBUG - 108286 - "/code/email_handler.py:2342" - _handle() - 5b22a7dc-8628-4dfc-857f-0feadc7ab4c9 - ====>=====>====>====>====>====>====>====>
2026-07-07 00:58:44,741 - SL - INFO - 108286 - "/code/email_handler.py:2343" - _handle() - 5b22a7dc-8628-4dfc-857f-0feadc7ab4c9 - New message, mail from bounce-sender@example.com, rctp tos ['x@sl.local'] 
2026-07-07 00:58:44,742 - SL - DEBUG - 108286 - "/code/email_handler.py:1963" - handle() - 5b22a7dc-8628-4dfc-857f-0feadc7ab4c9 - Cannot parse Postfix queue ID from None None
2026-07-07 00:58:44,743 - SL - DEBUG - 108286 - "/code/email_handler.py:1980" - handle() - 5b22a7dc-8628-4dfc-857f-0feadc7ab4c9 - ==>> Handle mail_from:bounce-sender@example.com, rcpt_tos:['x@sl.local'], header_from:bounce-sender@example.com, header_to:x@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'bounce-sender@example.com'), ('To', 'x@sl.local'), ('Subject', 'Q3 test'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:['SIZE=185'], rcpt_options:[]
2026-07-07 00:58:44,747 - SL - DEBUG - 108286 - "/code/email_handler.py:2202" - handle() - 5b22a7dc-8628-4dfc-857f-0feadc7ab4c9 - Forward phase bounce-sender@example.com(bounce-sender@example.com) -> x@sl.local
2026-07-07 00:58:44,753 - SL - DEBUG - 108286 - "/code/email_handler.py:545" - handle_forward() - 5b22a7dc-8628-4dfc-857f-0feadc7ab4c9 - alias x@sl.local not exist. Try to see if it can be created on the fly
2026-07-07 00:58:44,757 - SL - INFO - 108286 - "/code/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - 5b22a7dc-8628-4dfc-857f-0feadc7ab4c9 - Cannot auto-create custom domain alias for x@sl.local because there's no custom domain for sl.local
2026-07-07 00:58:44,757 - SL - INFO - 108286 - "/code/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - 5b22a7dc-8628-4dfc-857f-0feadc7ab4c9 - Cannot auto-create x@sl.local since it has no directory separator
2026-07-07 00:58:44,757 - SL - DEBUG - 108286 - "/code/email_handler.py:551" - handle_forward() - 5b22a7dc-8628-4dfc-857f-0feadc7ab4c9 - alias x@sl.local cannot be created on-the-fly, return 550
2026-07-07 00:58:44,757 - SL - WARNING - 108286 - "/code/app/email_utils.py:1363" - should_ignore_bounce() - 5b22a7dc-8628-4dfc-857f-0feadc7ab4c9 - do not send back bounce report to bounce-sender@example.com
2026-07-07 00:58:44,758 - SL - INFO - 108286 - "/code/email_handler.py:2367" - _handle() - 5b22a7dc-8628-4dfc-857f-0feadc7ab4c9 - Finish mail_from bounce-sender@example.com, rcpt_tos ['x@sl.local'], takes 0.01692056655883789 seconds with return code '250 SL E207 No bounce report'<<===
```

So when `mail_from` is an `IgnoreBounceSender`, `handle_forward` returns `status.E207 = "250 SL E207 No bounce report"` (`email_handler.py:552-553`, `app/email/status.py:12`) instead of `E515`: the message is silently accepted (a `250`), and no bounce report is generated. This was **observed at runtime**, not merely inferred.

### Stability (≥2 runs)

The primary `E515` case was exercised twice against the empty `SLDomain`. Run #2 (sender `normal-2@example.com`) was identical — both the wire reply and the complete handler log block:

```console
$ /code/.venv/bin/python send_q3.py normal-2@example.com
SENDER=normal-2@example.com RCPT=x@sl.local
RESULT: SMTPDataError -> code=550 msg=b'SL E515 Email not exist'
```

```
2026-07-07 00:58:43,601 - SL - DEBUG - 108286 - "/code/app/log.py:24" - set_message_id() - e0dd0278-f7c0-44f0-95dd-8a20b14da140 - set message_id e47ff94b-59bb-4f96-ab1a-60023129fc36
2026-07-07 00:58:43,601 - SL - DEBUG - 108286 - "/code/email_handler.py:2342" - _handle() - e47ff94b-59bb-4f96-ab1a-60023129fc36 - ====>=====>====>====>====>====>====>====>
2026-07-07 00:58:43,601 - SL - INFO - 108286 - "/code/email_handler.py:2343" - _handle() - e47ff94b-59bb-4f96-ab1a-60023129fc36 - New message, mail from normal-2@example.com, rctp tos ['x@sl.local'] 
2026-07-07 00:58:43,602 - SL - DEBUG - 108286 - "/code/email_handler.py:1963" - handle() - e47ff94b-59bb-4f96-ab1a-60023129fc36 - Cannot parse Postfix queue ID from None None
2026-07-07 00:58:43,604 - SL - DEBUG - 108286 - "/code/email_handler.py:1980" - handle() - e47ff94b-59bb-4f96-ab1a-60023129fc36 - ==>> Handle mail_from:normal-2@example.com, rcpt_tos:['x@sl.local'], header_from:normal-2@example.com, header_to:x@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'normal-2@example.com'), ('To', 'x@sl.local'), ('Subject', 'Q3 test'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:['SIZE=180'], rcpt_options:[]
2026-07-07 00:58:43,608 - SL - DEBUG - 108286 - "/code/email_handler.py:2202" - handle() - e47ff94b-59bb-4f96-ab1a-60023129fc36 - Forward phase normal-2@example.com(normal-2@example.com) -> x@sl.local
2026-07-07 00:58:43,613 - SL - DEBUG - 108286 - "/code/email_handler.py:545" - handle_forward() - e47ff94b-59bb-4f96-ab1a-60023129fc36 - alias x@sl.local not exist. Try to see if it can be created on the fly
2026-07-07 00:58:43,617 - SL - INFO - 108286 - "/code/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - e47ff94b-59bb-4f96-ab1a-60023129fc36 - Cannot auto-create custom domain alias for x@sl.local because there's no custom domain for sl.local
2026-07-07 00:58:43,617 - SL - INFO - 108286 - "/code/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - e47ff94b-59bb-4f96-ab1a-60023129fc36 - Cannot auto-create x@sl.local since it has no directory separator
2026-07-07 00:58:43,617 - SL - DEBUG - 108286 - "/code/email_handler.py:551" - handle_forward() - e47ff94b-59bb-4f96-ab1a-60023129fc36 - alias x@sl.local cannot be created on-the-fly, return 550
2026-07-07 00:58:43,618 - SL - INFO - 108286 - "/code/email_handler.py:2367" - _handle() - e47ff94b-59bb-4f96-ab1a-60023129fc36 - Finish mail_from normal-2@example.com, rcpt_tos ['x@sl.local'], takes 0.016767501831054688 seconds with return code '550 SL E515 Email not exist'<<===
```

Result: **stable** — empty `SLDomain` + normal sender → wire reply `550 SL E515 Email not exist` on every run (identical `email_handler.py:545` / `alias_utils.py:104` / `:165` / `email_handler.py:551` / `:2367` sequence).

**Q3 ↔ omitted step:** this rejection is caused by **skipping `init_app.py`**. Migrations create the `public_domain` table but leave it empty; with no `SLDomain` row for `sl.local` and no verified `CustomDomain`, both on-the-fly auto-create strategies fail (`app/alias_utils.py:104` and `:165`), and `handle_forward` returns `E515`.

---


## (e) Canonical startup order + failure modes

**Correct startup order:**

```
database  ->  migrations           ->  init_app.py            ->  web app                ->  email handler            ->  background workers
(PostgreSQL,   (flask db upgrade /      (seeds SLDomain via        (python server.py :7777   (python email_handler.py    (job_runner.py,
 create DB)     alembic upgrade head)    add_sl_domains())          / gunicorn wsgi:app)      :20381 / :25 prod)          event_listener.py, cron.py)
```

```mermaid
flowchart TD
    A[Start PostgreSQL<br/>create empty DB] --> B{Migrations run?<br/>flask db upgrade / alembic upgrade head}
    B -- No --> Q1["Q1: python server.py binds :7777<br/>login POST -> first ORM query<br/>ProgrammingError / UndefinedTable relation &quot;users&quot; -> HTTP 500"]
    B -- Yes --> C{init_app.py run?<br/>seeds SLDomain / public_domain}
    C -- No --> Q3["Q3: email to x@sl.local<br/>handle_forward: alias not found<br/>auto-create fails: SLDomain empty<br/>return 550 SL E515 Email not exist"]
    C -- Yes --> D[Q2: start required services]
    D --> D1[Web app<br/>python server.py :7777 / gunicorn wsgi:app]
    D --> D2[Email handler<br/>python email_handler.py :20381]
    D --> D3[Job runner<br/>python job_runner.py]
    D --> D4[Event listener + cron<br/>event_listener.py / cron.py]
```

**Why this order.** PostgreSQL must exist first because the engine connects at import (`app/db.py:12`). Migrations must run next to create the 77 tables (Alembic via Flask-Migrate; `alembic.ini:5` `script_location = migrations`; `migrations/env.py:28` `target_metadata = Base.metadata`; **255** revision files under `migrations/versions/`). `init_app.py` then seeds `SLDomain` (`init_app.py:39,69-73`) so that inbound mail to `@sl.local` is recognized. Only then are the web app, email handler, and background workers meaningful.

**Verified canonical references for this sequence:**

- `README.md`: `flask db upgrade` (`README.md:433`) → `python init_app.py` (`README.md:448`) → `python email_handler.py` (`README.md:480`) → `python job_runner.py` (`README.md:495`).
- `CONTRIBUTING.md`: Python 3.10 + poetry (`CONTRIBUTING.md:23`); PostgreSQL via `docker run … postgres:13` (`CONTRIBUTING.md:100`); dev one-liner `alembic upgrade head && flask dummy-data && python3 server.py` (`CONTRIBUTING.md:106`); then open `http://localhost:7777` (`CONTRIBUTING.md:109`).
- `scripts/reset_local_db.sh`: `drop schema public cascade; create schema public;` piped to `psql` (`scripts/reset_local_db.sh:4`) → `poetry run alembic upgrade head` (`:6`) → `poetry run flask dummy-data` (`:7`).
- `.github/workflows/main.yml`: provisions `postgres:13` (`.github/workflows/main.yml:47`) and runs `CONFIG=tests/test.env poetry run alembic upgrade head` (`:98`).

**Failure modes — cause and effect of skipping a step (backed by the captured output above):**

| Skipped step | Symptom (observed) | Mechanism |
|--------------|--------------------|-----------|
| **Migrations** (→ Q1) | `python server.py` binds `:7777`, but the first login POST returns **HTTP 500** with `sqlalchemy.exc.ProgrammingError` / `psycopg2.errors.UndefinedTable: relation "users" does not exist` | Engine connects at import with no DDL check (`app/db.py:12`), so startup succeeds; the request-time ORM query (`app/auth/views/login.py:43` → `app/models.py:84`) hits a table the migrations never created. |
| **`init_app.py`** (→ Q3) | Migrations applied, but mail to `x@sl.local` is rejected with **`550 SL E515 Email not exist`** | `public_domain` / `SLDomain` is empty and no verified `CustomDomain` matches, so on-the-fly auto-create fails in both `check_if_alias_can_be_auto_created_for_custom_domain` (`app/alias_utils.py:104`) and `check_if_alias_can_be_auto_created_for_a_directory` (`app/alias_utils.py:165`); `handle_forward()` then returns `status.E515` (`email_handler.py:551-555`, `app/email/status.py:51`). |

Both failure modes were reproduced first-hand: **Q1 ↔ omitted migration step**, **Q3 ↔ omitted `init_app.py` step**.

---

## (f) Cleanup / repository integrity

This was a **read-only** investigation. The artifacts created to gather the evidence above were all transient and were removed after capture:

- **Throwaway databases:** `sl_q1_empty` and `sl_q3` (dropped from the running PostgreSQL).
- **Temporary dotenvs:** `/code/q1.env`, `/code/q3.env` (derived from `.env` by changing only `DB_URI`).
- **Temporary observation helpers:** `send_q3.py` (the SMTP-send client for Q3) and `portproof_all.py` (the read-only `/proc/net/tcp` decoder). Neither is committed.

No existing repository file was modified or deleted, and no code was added other than this document. The final working-tree state confirms it — the only change is the deliverable itself:

```console
$ git status --porcelain
 M blitzy/documentation/app_2cd6ee777f8c.md
```

---

## Appendix — Verified `file:line` reference map

All line numbers below were confirmed against the source at investigation time.

**Q1 (empty-DB exception):**
- `app/db.py:9-11` engine creation; `app/db.py:12` `connection = engine.connect()` (import-time, no DDL check).
- `server.py:163-165` `if MEM_STORE_URI:` (Redis / session gate, unset by default); `server.py:250-256` `/` → `auth.login`; `server.py:572` `local_main()`; `server.py:588` `app.run(debug=True, port=7777)`; `server.py:598-599` entrypoint; `server.py:284` `after_request` log; `server.py:389` `def error_handler` + `:390` `LOG.e(e)`.
- `app/auth/views/login.py:21` route; `:40` `validate_on_submit()`; `:43` `User.get_by(email=...)`.
- `app/models.py:84` `get_by` → `Session.query(cls).filter_by(**kw).first()`; `app/models.py:337` `User.__tablename__ = "users"`.
- Render chain: `templates/auth/login.html` → `templates/single.html` → `templates/base.html:89` (anonymous short-circuit).
- Observed: `sqlalchemy.exc.ProgrammingError` wrapping `psycopg2.errors.UndefinedTable: relation "users" does not exist` → HTTP 500; SQLAlchemy 1.3 URL `sqlalche.me/e/13/f405`.

**Q2 (required services):**
- Web app: `server.py:588` (`:7777`); `wsgi.py:1,3` `create_app()` (Gunicorn `wsgi:app`). `* Running on …` line suppressed by `app/log.py:69-71` (`werkzeug` logger disabled).
- Email handler: `email_handler.py:2381` `main`, `:2383` `Controller(hostname="0.0.0.0", port)`, `:2385` `start()`, `:2386` `Start mail controller`, `:2399` default `20381`, `:2403` `Listen for port`, `:2404` `main`.
- Job runner: `job_runner.py:329` `__main__`, `:347` `time.sleep(10)`.
- Event listener: `event_listener.py:29` `main`, `:34` `Using PostgresEventSource`, `:35` `PostgresEventSource(...)`, `:43` `Starting with HttpEventSink`, `events/event_source.py:49` `Starting to listen to events`, `:96` no-subcommand usage/exit.
- Cron: `cron.py:1` argparse, `:1262` `__main__`, `:1263` `Start running cronjob`, `:1274+` `args.job` dispatch.
- Log: `app/log.py:12-14` format, `:41` `StreamHandler(sys.stdout)`, `:51` DEBUG, `:67` `>>> init logging <<<`, `:69-71` `werkzeug` logger disabled, `:79` `LOG = _get_logger("SL")`.

**Q3 (skipped-init SMTP rejection):**
- Dispatch: `email_handler.py:2289` `handle_DATA`, `:2335` `_handle`, `:1945` `handle`, `:2195` `is_reverse_alias`, `:2201` `else # Forward case`, `:2202` `Forward phase` log, `:2208` `handle_forward(...)`.
- `handle_forward`: `email_handler.py:536` def, `:543` `Alias.get_by(email=...)`, `:545` "not exist" `LOG.d`, `:549` `try_auto_create`, `:551` "cannot be created on-the-fly, return 550" `LOG.d`, `:552-553` `should_ignore_bounce` → `E207`, `:554-555` `E515`; `:2367` final `_handle` "return code" line.
- Auto-create (the real gate): `app/alias_utils.py:202` `try_auto_create`; `:274` `try_auto_create_via_domain` → `check_if_alias_can_be_auto_created_for_custom_domain` (`:92`), failure `LOG.i` at `:104`; `:227` `try_auto_create_directory` → `check_if_alias_can_be_auto_created_for_a_directory` (`:145`), failure `LOG.info` at `:165`.
- `app/email_utils.py:557` `is_valid_alias_address_domain` — a *separate* sanity check used at `email_handler.py:1000` on an existing alias (returns `status.E503`), **not** on the forward auto-create path. `app/email_utils.py:1361` `should_ignore_bounce`, `:1363` "do not send back bounce report" log.
- `app/email/status.py:51` `E515 = "550 SL E515 Email not exist"`; `app/email/status.py:12` `E207 = "250 SL E207 No bounce report"`.
- `app/models.py:3116` `SLDomain` → `__tablename__ = "public_domain"`; `app/models.py:3354` `IgnoreBounceSender` → `__tablename__ = "ignore_bounce_sender"`.
- `init_app.py:39` `add_sl_domains`, `:69-73` `__main__`; `app/config.py:157-160` `ALIAS_DOMAINS` default.

**Startup order / env / runtime:**
- `README.md:433/448/480/495`; `CONTRIBUTING.md:23/100/106/109/236/248`; `scripts/reset_local_db.sh:4/6/7`; `.github/workflows/main.yml:47/98`.
- `alembic.ini:5`; `migrations/env.py:28`; 255 files under `migrations/versions/`.
- `example.env:6/19/22/40/75/77`; `app/config.py:65` (`CONFIG` env), `:68` (`load config file` print), `:69` (`load_dotenv`), `:14-20` (`get_abs_path`), `:252-262` (GNUPGHOME temp-dir warning).
- `pyproject.toml:61` (`python = "^3.10"`), `:66` gunicorn, `:116` `SQLAlchemy = 1.3.24`; `Dockerfile:3,18` `WORKDIR /code`, `Dockerfile:8` `FROM python:3.10`.
