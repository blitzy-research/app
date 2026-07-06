# SimpleLogin Startup Investigation — Run-First Q&A

**Branch:** `app_2cd6ee777f8c`
**Scope:** Read-only investigation of SimpleLogin's startup behavior. No source file was modified or deleted; the only artifact produced is this document. Temporary observation scripts and throwaway databases were used and then removed (see §6 *Cleanup*).

This document answers three coupled runtime-behavior questions, plus an overarching startup-order/failure-mode narrative, **from direct first-hand observation**. Every behavioral claim is backed by **(1)** a `file:line` citation into the SimpleLogin source and **(2)** the actual, unedited runtime output captured beside the claim.

> **Reading the captured logs.** SimpleLogin's logger (`app/log.py:79` `LOG = _get_logger("SL")`) writes to STDOUT (`app/log.py:41` `logging.StreamHandler(sys.stdout)`) at DEBUG level (`app/log.py:51`) using the fixed format at `app/log.py:12-14`:
> `"%(asctime)s - %(name)s - %(levelname)s - %(process)d - \"%(pathname)s:%(lineno)d\" - %(funcName)s() - %(message_id)s - %(message)s"`.
> In every pasted log line below, the absolute working-directory prefix has been replaced with `<repo>` and the interpreter's stdlib path with `<python3.10>` (these are machine-specific and carry no behavioral meaning); the SMTP greeting hostname is shown as `<host>`. Everything else is **verbatim**.

---

## Table of Contents

- [(a) Environment bring-up](#a-environment-bring-up)
- [(b) Q1 — Empty-database startup exception](#b-q1--empty-database-startup-exception)
- [(c) Q2 — Required Python services + port-binding evidence](#c-q2--required-python-services--port-binding-evidence)
- [(d) Q3 — Skipped-initialization SMTP rejection](#d-q3--skipped-initialization-smtp-rejection)
- [(e) Canonical startup order + failure modes](#e-canonical-startup-order--failure-modes)
- [Appendix — Verified file:line reference map](#appendix--verified-fileline-reference-map)

---

## (a) Environment bring-up

The investigation was performed in the canonical runtime — **Python 3.10** (`Dockerfile:8` `FROM python:3.10`; `pyproject.toml:61` `python = "^3.10"`; Python 3.12 is explicitly unsupported per `CONTRIBUTING.md:236` *"we haven't managed to make python 3.12 work"*) with **PostgreSQL 13** and the SimpleLogin dependencies from `poetry.lock`. **Redis is not required** for any of Q1/Q2/Q3 in the default local configuration because the session/rate-limit store is only wired when `MEM_STORE_URI` is set (`server.py:163-165`), and `MEM_STORE_URI` is unset here.

**Interpreter / tooling (exact commands and output):**

```console
$ export PATH="/root/.local/bin:$PATH"
$ ./.venv/bin/python --version
Python 3.10.20
$ poetry --version
Poetry (version 1.8.5)
$ psql -h localhost -p 15432 -U myuser -d postgres -tAc "SHOW server_version;"
13.23 (Debian 13.23-1.pgdg13+1)
```

> The dependencies are installed in the in-project virtualenv `./.venv` (Python 3.10.20). The canonical install command is `poetry install` (`CONTRIBUTING.md:248`); in this environment the equivalent already-materialized `.venv` was used, and every command below is run through `./.venv/bin/python` (i.e., the 3.10 interpreter), never the host interpreter.

**Configuration (`CONFIG`-driven dotenv).** `app/config.py:65-69` selects the dotenv from the `CONFIG` environment variable — if `CONFIG` is set it loads that file (`load_dotenv(get_abs_path(config_file))`), otherwise it loads `.env`. `app/config.py:14-19` (`get_abs_path`) returns absolute paths unchanged, so a dotenv anywhere on disk works. The canonical values come from `example.env`: `URL=http://localhost:7777` (`example.env:6`), `NOT_SEND_EMAIL=true` (`example.env:19`), `EMAIL_DOMAIN=sl.local` (`example.env:22`), `SUPPORT_EMAIL=support@sl.local` (`example.env:40`), `DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin` (`example.env:75`), `FLASK_SECRET=secret` (`example.env:77`). All cryptographic material required at startup already ships under `local_data/` (`jwtRS256.key`, `dkim.key`, PGP keys, `test_words.txt`, …), so a `CONFIG`-driven start has no missing-key failures.

Three database states are needed to exercise the three questions in isolation, so three `DB_URI`s were used (only `DB_URI` differs from the canonical dotenv; every other key is unchanged):

| Scenario | Database | State | Config file |
|----------|----------|-------|-------------|
| Q1 | `sl_q1_empty` | created, **no migrations** (empty schema) | `q1.env` (temp, outside repo) |
| Q2 | `simplelogin` | migrated **and** seeded (`init_app.py` run) | repo `.env` |
| Q3 | `sl_q3` | migrated, **`init_app.py` NOT run** (empty `SLDomain`) | `q3.env` (temp, outside repo) |

```console
# create the empty (Q1) and migrated-but-unseeded (Q3) databases in the running PostgreSQL
$ psql -h localhost -p 15432 -U myuser -d postgres -c "CREATE DATABASE sl_q1_empty OWNER myuser;"
CREATE DATABASE
$ psql -h localhost -p 15432 -U myuser -d postgres -c "CREATE DATABASE sl_q3 OWNER myuser;"
CREATE DATABASE

# derive the two temporary dotenvs from the canonical .env, changing ONLY DB_URI
$ sed 's#^DB_URI=.*#DB_URI=postgresql://myuser:mypassword@localhost:15432/sl_q1_empty#' .env > /tmp/sl_investigation/q1.env
$ sed 's#^DB_URI=.*#DB_URI=postgresql://myuser:mypassword@localhost:15432/sl_q3#'       .env > /tmp/sl_investigation/q3.env
```

Confirming the default local config uses **cookie sessions (no Redis, no extra PG session store)**:

```console
$ grep -c 'MEM_STORE_URI' .env
0        # MEM_STORE_URI unset -> server.py:163-165 branch is skipped
```

> **Note on running the services.** Each service below is launched via `./.venv/bin/python <entrypoint>` with `CONFIG=<dotenv>` and `PYTHONUNBUFFERED=1` (so STDOUT is line-flushed and captured in real time; this does not change behavior). The web app additionally sets `WERKZEUG_RUN_MAIN=true`, which makes Flask's debug dev-server run **single-process** (the reloaded worker) rather than forking a reloader parent+child; the bound socket and request handling are identical.
>
> **Note on `ss`/`lsof`.** In this container `ss -ltn` did **not** list the dev-server / SMTP sockets (a tooling limitation of the environment), so port binding is demonstrated **functionally** — via an actual HTTP request to `:7777` and an actual SMTP connection to `:20381` — which is stronger evidence than a socket table row.

---

## (b) Q1 — Empty-database startup exception

**Question.** With PostgreSQL running but the target DB schema **empty** (migrations deliberately not run), start the web app with `python server.py`, open the login page, and capture the **full Python exception/traceback** raised.

**Answer (summary).** The server starts and binds `:7777` cleanly against the empty schema; the failure appears only at **request time**. Submitting the login form fires the first ORM query, which raises **`sqlalchemy.exc.ProgrammingError`** wrapping **`psycopg2.errors.UndefinedTable: relation "users" does not exist`**, and the request returns **HTTP 500**. The offending relation observed is **`users`**.

### Mechanism (cause → effect), with `file:line`

1. **The DB engine connects at *import time*, with no schema/DDL check.** `app/db.py:9-11` builds the engine and `app/db.py:12` opens a connection at module load:
   ```python
   # app/db.py:9-12
   engine = create_engine(
       config.DB_URI, connect_args={"application_name": config.DB_CONN_NAME}
   )
   connection = engine.connect()
   ```
   `engine.connect()` succeeds against an empty database (connecting to PostgreSQL says nothing about which tables exist), so nothing fails during startup.

2. **`python server.py` runs the debug dev server on `:7777`.** `server.py:572` `def local_main():` builds the app, enables the Debug Toolbar, sets `app.debug = True`, and `server.py:588` `app.run(debug=True, port=7777)`; the entrypoint is `server.py:598-599` `if __name__ == "__main__": local_main()`.

3. **Redis/session store is not involved by default.** `server.py:163-165` only installs the memory-store session backend `if MEM_STORE_URI:`; with it unset, sessions are cookie-based and never touch PostgreSQL — so a bare page load does not hit the DB via sessions.

4. **A bare anonymous GET renders with no DB query; the login *POST* is the reliable first ORM query.** `/` redirects anonymous users to `auth.login` (`server.py:250-256`); the login GET view (`app/auth/views/login.py:21`) renders the template chain `templates/auth/login.html` → `templates/single.html` → `templates/base.html`, whose only `current_user` reference short-circuits for anonymous users (`templates/base.html:89`). On **POST**, `app/auth/views/login.py:40` `if form.validate_on_submit():` becomes true and `app/auth/views/login.py:43` executes:
   ```python
   # app/auth/views/login.py:43
   user = User.get_by(email=email) or User.get_by(email=canonical_email)
   ```
   `User.get_by(...)` runs `Session.query(cls).filter_by(**kw).first()` (`app/models.py:84`) against table **`users`** (`User.__tablename__ = "users"`, `app/models.py:337`). Because migrations never created that relation, PostgreSQL raises `UndefinedTable`, SQLAlchemy 1.3 re-raises it as `ProgrammingError`, and the app's error handler (`server.py:390`) turns it into HTTP 500.

### Runtime evidence

**Command (empty DB, no migrations):**

```console
$ psql -h localhost -p 15432 -U myuser -d sl_q1_empty -tAc \
    "SELECT count(*) FROM information_schema.tables WHERE table_schema='public';"
0     # empty schema: zero tables

$ CONFIG=/tmp/sl_investigation/q1.env WERKZEUG_RUN_MAIN=true PYTHONUNBUFFERED=1 \
    ./.venv/bin/python server.py
```

**The server binds `:7777` despite the empty DB** — proven by live requests (the `/` redirect and `/health` need no DB):

```console
$ curl -s -m 5 -o /dev/null -D - http://127.0.0.1:7777/ | grep -iE '^HTTP|^location'
HTTP/1.0 302 FOUND
Location: http://127.0.0.1:7777/auth/login

$ curl -s -m 5 -w " HTTP=%{http_code}\n" http://127.0.0.1:7777/health
success HTTP=200
```

The dev-server startup banner (from the canonical `python server.py` reloader parent) confirms debug mode is on:

```
 * Serving Flask app "server" (lazy loading)
 * Environment: production
 * Debug mode: on
```

**Triggering the first ORM query** — GET the login page to obtain a CSRF token, then POST the form:

```console
$ curl -s -c cookies.txt http://127.0.0.1:7777/auth/login -o login.html -w "%{http_code}\n"
200
$ CSRF=$(grep -oE 'name="csrf_token"[^>]*value="[^"]+"' login.html | sed -E 's/.*value="([^"]+)".*/\1/')
$ curl -s -b cookies.txt \
    --data-urlencode "csrf_token=${CSRF}" \
    --data-urlencode "email=test@example.com" \
    --data-urlencode "password=whatever123" \
    -o /dev/null -w "HTTP_STATUS=%{http_code}\n" \
    http://127.0.0.1:7777/auth/login
HTTP_STATUS=500
```

**The complete, unedited server-side traceback** printed to STDOUT/STDERR for that request (HTTP 500). The app's error handler first logs the exception (`server.py:390`), then the full chained traceback is emitted:

```
2026-07-06 23:00:12,071 - SL - ERROR - 61160 - "<repo>/server.py:390" - error_handler() -  - (psycopg2.errors.UndefinedTable) relation "users" does not exist
LINE 2: FROM users 
             ^

[SQL: SELECT users.directory_quota AS users_directory_quota, users.subdomain_quota AS users_subdomain_quota, users.password AS users_password, users.id AS users_id, users.created_at AS users_created_at, users.updated_at AS users_updated_at, users.email AS users_email, users.name AS users_name, users.is_admin AS users_is_admin, users.alias_generator AS users_alias_generator, users.notification AS users_notification, users.activated AS users_activated, users.disabled AS users_disabled, users.profile_picture_id AS users_profile_picture_id, users.otp_secret AS users_otp_secret, users.enable_otp AS users_enable_otp, users.last_otp AS users_last_otp, users.fido_uuid AS users_fido_uuid, users.default_alias_custom_domain_id AS users_default_alias_custom_domain_id, users.default_alias_public_domain_id AS users_default_alias_public_domain_id, users.lifetime AS users_lifetime, users.paid_lifetime AS users_paid_lifetime, users.lifetime_coupon_id AS users_lifetime_coupon_id, users.trial_end AS users_trial_end, users.default_mailbox_id AS users_default_mailbox_id, users.sender_format AS users_sender_format, users.sender_format_updated_at AS users_sender_format_updated_at, users.replace_reverse_alias AS users_replace_reverse_alias, users.referral_id AS users_referral_id, users.intro_shown AS users_intro_shown, users.max_spam_score AS users_max_spam_score, users.newsletter_alias_id AS users_newsletter_alias_id, users.include_sender_in_reverse_alias AS users_include_sender_in_reverse_alias, users.random_alias_suffix AS users_random_alias_suffix, users.expand_alias_info AS users_expand_alias_info, users.ignore_loop_email AS users_ignore_loop_email, users.alternative_id AS users_alternative_id, users.disable_automatic_alias_note AS users_disable_automatic_alias_note, users.one_click_unsubscribe_block_sender AS users_one_click_unsubscribe_block_sender, users.include_website_in_one_click_alias AS users_include_website_in_one_click_alias, users.disable_import AS users_disable_import, users.can_use_phone AS users_can_use_phone, users.phone_quota AS users_phone_quota, users.block_behaviour AS users_block_behaviour, users.include_header_email_header AS users_include_header_email_header, users.enable_data_breach_check AS users_enable_data_breach_check, users.flags AS users_flags, users.unsub_behaviour AS users_unsub_behaviour, users.delete_on AS users_delete_on 
FROM users 
WHERE users.email = %(email_1)s 
 LIMIT %(param_1)s]
[parameters: {'email_1': 'test@example.com', 'param_1': 1}]
(Background on this error at: http://sqlalche.me/e/13/f405)
Traceback (most recent call last):
  File "<repo>/.venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1276, in _execute_context
    self.dialect.do_execute(
  File "<repo>/.venv/lib/python3.10/site-packages/sqlalchemy/engine/default.py", line 608, in do_execute
    cursor.execute(statement, parameters)
psycopg2.errors.UndefinedTable: relation "users" does not exist
LINE 2: FROM users 
             ^


The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "<repo>/.venv/lib/python3.10/site-packages/flask/app.py", line 1950, in full_dispatch_request
    rv = self.dispatch_request()
  File "<repo>/.venv/lib/python3.10/site-packages/flask_debugtoolbar/__init__.py", line 125, in dispatch_request
    return view_func(**req.view_args)
  File "<python3.10>/cProfile.py", line 110, in runcall
    return func(*args, **kw)
  File "<repo>/.venv/lib/python3.10/site-packages/flask_limiter/extension.py", line 702, in __inner
    return obj(*a, **k)
  File "<repo>/app/auth/views/login.py", line 43, in login
    user = User.get_by(email=email) or User.get_by(email=canonical_email)
  File "<repo>/app/models.py", line 84, in get_by
    return Session.query(cls).filter_by(**kw).first()
  File "<repo>/.venv/lib/python3.10/site-packages/sqlalchemy/orm/query.py", line 3429, in first
    ret = list(self[0:1])
  File "<repo>/.venv/lib/python3.10/site-packages/sqlalchemy/orm/query.py", line 3203, in __getitem__
    return list(res)
  File "<repo>/.venv/lib/python3.10/site-packages/sqlalchemy/orm/query.py", line 3535, in __iter__
    return self._execute_and_instances(context)
  File "<repo>/.venv/lib/python3.10/site-packages/sqlalchemy/orm/query.py", line 3560, in _execute_and_instances
    result = conn.execute(querycontext.statement, self._params)
  File "<repo>/.venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1011, in execute
    return meth(self, multiparams, params)
  File "<repo>/.venv/lib/python3.10/site-packages/sqlalchemy/sql/elements.py", line 298, in _execute_on_connection
    return connection._execute_clauseelement(self, multiparams, params)
  File "<repo>/.venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1124, in _execute_clauseelement
    ret = self._execute_context(
  File "<repo>/.venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1316, in _execute_context
    self._handle_dbapi_exception(
  File "<repo>/.venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1510, in _handle_dbapi_exception
    util.raise_(
  File "<repo>/.venv/lib/python3.10/site-packages/sqlalchemy/util/compat.py", line 182, in raise_
    raise exception
  File "<repo>/.venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1276, in _execute_context
    self.dialect.do_execute(
  File "<repo>/.venv/lib/python3.10/site-packages/sqlalchemy/engine/default.py", line 608, in do_execute
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
2026-07-06 23:00:12,077 - SL - DEBUG - 61160 - "<repo>/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 500, takes 0.01436305046081543
```

> The `SELECT users.… FROM users …` statement appears **twice** above and both copies are shown verbatim: SQLAlchemy prints the offending SQL once on the wrapped `psycopg2` error and again on the outer `ProgrammingError`. Nothing is elided.

**Exception anatomy (as observed):**

- **Chained cause (inner):** `psycopg2.errors.UndefinedTable: relation "users" does not exist`.
- **Raised (outer):** `sqlalchemy.exc.ProgrammingError: (psycopg2.errors.UndefinedTable) relation "users" does not exist` — joined by Python's *"The above exception was the direct cause of the following exception"*.
- **Offending relation:** `users`.
- **Trigger frame:** `app/auth/views/login.py:43` → `app/models.py:84` (`Session.query(cls).filter_by(**kw).first()`).
- **HTTP status:** `500` (confirmed by both the `HTTP_STATUS=500` client response and the `after_request` log `POST /auth/login … 500`, `server.py:284`).
- **SQLAlchemy reference:** `http://sqlalche.me/e/13/f405` (the SQLAlchemy **1.3** error-code URL, matching `SQLAlchemy = 1.3.24` at `pyproject.toml:116`).

### Stability (≥2 runs)

The same empty-schema condition was exercised twice. Run #2 (a different sender email) produced the identical exception class and relation:

```
2026-07-06 23:00:54,879 - SL - ERROR - 61160 - "<repo>/server.py:390" - error_handler() -  - (psycopg2.errors.UndefinedTable) relation "users" does not exist
psycopg2.errors.UndefinedTable: relation "users" does not exist
sqlalchemy.exc.ProgrammingError: (psycopg2.errors.UndefinedTable) relation "users" does not exist
2026-07-06 23:00:54,880 - SL - DEBUG - 61160 - "<repo>/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 500, takes 0.005578041076660156
```

Result: **stable** — HTTP 500 with `ProgrammingError`/`UndefinedTable: relation "users" does not exist` on every run.

**Q1 ↔ omitted step:** this failure is caused by **skipping the migration step**. The engine connects at import (`app/db.py:12`) so the server starts, but the schema has no `users` table, so the first ORM query fails at request time.

---


## (c) Q2 — Required Python services + port-binding evidence

**Question.** After migrations and `init_app.py` have been run correctly, identify which Python services must run for the system to function, start each one, and capture the exact stdout/stderr confirming each service is up and bound to its port.

**Answer (summary).** Five Python services make up the running system. This scenario uses the fully-provisioned `simplelogin` database (migrated — 77 tables — and seeded, `public_domain` has one row `sl.local`):

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

Each service's literal startup output is shown below. All lines use the `app/log.py` "SL" format; the `>>> init logging <<<` line (`app/log.py:67`) prints on import and marks logger initialization.

### Service 1 — Web app (`python server.py`, binds `:7777`)

`server.py:588` `app.run(debug=True, port=7777)`. Command and functional bind proof (against the seeded `simplelogin` DB):

```console
$ CONFIG=.env WERKZEUG_RUN_MAIN=true PYTHONUNBUFFERED=1 ./.venv/bin/python server.py
load config file <repo>/.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/fgcpwsgdpozybsusaqzg
Upload files to local dir
>>> init logging <<<
2026-07-06 23:01:50,084 - SL - DEBUG - 62275 - "<repo>/app/utils.py:17" - <module>() -  - load words file: <repo>/local_data/test_words.txt

$ curl -s -m 5 -o /dev/null -D - http://127.0.0.1:7777/ | grep -iE '^HTTP|^location'
HTTP/1.0 302 FOUND
Location: http://127.0.0.1:7777/auth/login
$ curl -s -m 5 -w " HTTP=%{http_code}\n" http://127.0.0.1:7777/health
success HTTP=200
```

The `302 → /auth/login` and `/health → 200` responses prove the process is **listening and serving on `:7777`**. (Production equivalent: `wsgi.py` exposes `app = create_app()` — `wsgi.py:1,3` — served by Gunicorn per `pyproject.toml:66` `gunicorn = "^20.0.4"`.)

### Service 2 — Email handler (`python email_handler.py`, binds `:20381`)

`email_handler.py:2381` `def main(port)`; `:2383` `controller = Controller(MailHandler(), hostname="0.0.0.0", port=port)`; `:2385` `controller.start()`; `:2386` logs the bound controller; the `__main__` argparse default port is **20381** (`email_handler.py:2399`); `:2403` logs the listen port; `:2404` calls `main`. Command and **exact port-binding lines** (note the literal port `20381` in both):

```console
$ CONFIG=.env PYTHONUNBUFFERED=1 ./.venv/bin/python email_handler.py
load config file <repo>/.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/nowsbqzvvliilxtttpkc
Upload files to local dir
>>> init logging <<<
2026-07-06 23:01:26,679 - SL - DEBUG - 62002 - "<repo>/app/utils.py:17" - <module>() -  - load words file: <repo>/local_data/test_words.txt
2026-07-06 23:01:27,357 - SL - INFO - 62002 - "<repo>/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-06 23:01:27,359 - SL - DEBUG - 62002 - "<repo>/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

Independent corroboration — an actual SMTP client connecting to `:20381` receives the aiosmtpd greeting:

```console
$ python3 -c "import socket; s=socket.create_connection(('127.0.0.1',20381),timeout=5); print(s.recv(200).decode().strip())"
220 <host> Python SMTP 1.4.2
```

(`Python SMTP 1.4.2` is the aiosmtpd version — satisfies `pyproject.toml:87` `aiosmtpd = "^1.2"`.) Production SMTP port is `:25`.

### Service 3 — Job runner (`python job_runner.py`, worker; no port)

`job_runner.py:329` `if __name__ == "__main__":` enters a `while True:` loop that takes pending jobs and `job_runner.py:347` `time.sleep(10)` between polls. It binds no port. Its startup output is the standard import banner, after which it polls silently until a `Job` exists:

```console
$ CONFIG=.env PYTHONUNBUFFERED=1 ./.venv/bin/python job_runner.py
load config file <repo>/.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/rmwviujlalcyqcpcwhks
Upload files to local dir
>>> init logging <<<
2026-07-06 23:01:49,994 - SL - DEBUG - 62278 - "<repo>/app/utils.py:17" - <module>() -  - load words file: <repo>/local_data/test_words.txt
```

The process was confirmed alive and idle in its polling loop (no port, no further output until a job arrives):

```console
$ ps -p 62278 -o pid,stat,etime,cmd --no-headers
  62278 S          00:50 ./.venv/bin/python job_runner.py
```

### Service 4 — Event listener (`python event_listener.py listener`, PG LISTEN; no port)

`event_listener.py:29` `def main(mode, …)`; when the `listener` sub-command is used, `event_listener.py:34` logs `Using PostgresEventSource` and `:35` constructs `PostgresEventSource(EVENT_LISTENER_DB_URI)`. (A bare `python event_listener.py` with no sub-command prints `Invalid usage. Pass a valid subcommand as argument` and exits 1 — `event_listener.py:95-97`.) Command and exact startup lines:

```console
$ CONFIG=.env PYTHONUNBUFFERED=1 ./.venv/bin/python event_listener.py listener
load config file <repo>/.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/jleexnxqlfekjmourhfr
Upload files to local dir
>>> init logging <<<
2026-07-06 23:01:49,979 - SL - DEBUG - 62281 - "<repo>/app/utils.py:17" - <module>() -  - load words file: <repo>/local_data/test_words.txt
2026-07-06 23:01:50,092 - SL - INFO - 62281 - "<repo>/event_listener.py:34" - main() -  - Using PostgresEventSource
2026-07-06 23:01:50,095 - SL - INFO - 62281 - "<repo>/event_listener.py:43" - main() -  - Starting with HttpEventSink
2026-07-06 23:01:50,096 - SL - INFO - 62281 - "<repo>/events/event_source.py:49" - __listen() -  - Starting to listen to events
```

### Service 5 — Scheduler / cron (`python cron.py`, batch; no port)

`cron.py:1` `import argparse`; `cron.py:1262` `if __name__ == "__main__":` logs `Start running cronjob` (`cron.py:1263`) and dispatches on `args.job` (`cron.py:1274` onward — e.g. `stats`, `notify_trial_end`, `sanity_check`, …). Command, startup line, and usage:

```console
$ CONFIG=.env PYTHONUNBUFFERED=1 ./.venv/bin/python cron.py
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/fhmuntfbhrpiasgntpmh
Upload files to local dir
>>> init logging <<<
2026-07-06 23:02:37,163 - SL - DEBUG - 62828 - "<repo>/app/utils.py:17" - <module>() -  - load words file: <repo>/local_data/test_words.txt
2026-07-06 23:02:38,006 - SL - DEBUG - 62828 - "<repo>/cron.py:1263" - <module>() -  - Start running cronjob

$ CONFIG=.env PYTHONUNBUFFERED=1 ./.venv/bin/python cron.py --help
...
usage: cron.py [-h] [-j JOB]

options:
  -h, --help         show this help message and exit
  -j JOB, --job JOB  Choose a cron job to run
```

(With no `-j`, `args.job` is `None`, so no maintenance job runs; a concrete job is selected with `-j <job>`.)

### Stability (≥2 runs)

The two network ports are deterministic constants, not dynamically assigned:

- **`:7777`** — hard-coded in `server.py:588`; observed bound in both the Q1 run (empty DB) and this Q2 run.
- **`:20381`** — the argparse default in `email_handler.py:2399`; observed bound in this Q2 run **and** again in the Q3 run below, both logging `Listen for port 20381` / `Start mail controller 0.0.0.0 20381`.

Result: **stable** — ports 7777 and 20381 every time.

---


## (d) Q3 — Skipped-initialization SMTP rejection

**Question.** Run migrations but do **not** run `init_app.py` (leaving `SLDomain` empty — no email domains configured). Start the email handler, send a message to `x@sl.local`, and capture both the SMTP status code / wire reply returned to the sender **and** the log line(s) explaining the rejection.

**Answer (summary).** With `SLDomain` empty, mail to `x@sl.local` is rejected with the permanent SMTP reply **`550 SL E515 Email not exist`** (`app/email/status.py:51`), and the handler logs two `LOG.d` lines: `alias x@sl.local not exist…` and `alias x@sl.local cannot be created on-the-fly, return 550`. The edge case where the sender is an `IgnoreBounceSender` instead returns **`250 SL E207 No bounce report`** (`app/email/status.py:12`).

> **Important — real table name.** The `SLDomain` model maps to the table **`public_domain`**, not `sl_domain` (`app/models.py:3116` `class SLDomain(...)` → `__tablename__ = "public_domain"`). Row-count evidence below therefore queries `public_domain`.

### `SLDomain` state — before/after `init_app.py`

The `sl_q3` database was migrated (`alembic upgrade head`) but `init_app.py` was **not** run. `alembic upgrade head` applied all 255 revisions:

```console
$ CONFIG=/tmp/sl_investigation/q3.env ./.venv/bin/alembic upgrade head
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade  -> 5e549314e1e2, empty message
... (255 "Running upgrade" steps) ...
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

$ CONFIG=/tmp/sl_investigation/q3.env ./.venv/bin/python -c \
    "from app.models import SLDomain; print('SLDomain.count() =', SLDomain.count()); print('rows =', [d.domain for d in SLDomain.all()])"
SLDomain.count() = 0
rows = []
```

**AFTER `init_app.py`** — `add_sl_domains()` (`init_app.py:39`) seeds the domain (with `EMAIL_DOMAIN=sl.local`, `ALIAS_DOMAINS` defaults to `[sl.local]` per `app/config.py:157-160`); the `__main__` block (`init_app.py:69-73`) runs `load_pgp_public_keys()` then `add_sl_domains()`:

```console
$ CONFIG=/tmp/sl_investigation/q3.env ./.venv/bin/python init_app.py
2026-07-06 23:05:23,226 - SL - DEBUG - 64743 - "<repo>/init_app.py:36" - load_pgp_public_keys() -  - Finish load_pgp_public_keys
2026-07-06 23:05:23,228 - SL - INFO - 64743 - "<repo>/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain

$ psql -h localhost -p 15432 -U myuser -d sl_q3 -c "SELECT id, domain, use_as_reverse_alias FROM public_domain;"
 id |  domain  | use_as_reverse_alias 
----+----------+----------------------
  1 | sl.local | t
(1 row)
```

State transition: **`public_domain` 0 rows → 1 row (`sl.local`)**, caused solely by `init_app.py`'s `add_sl_domains()`. The Q3 rejection observations below were all captured **before** this seeding, i.e. with `SLDomain` genuinely empty.

### Rejection mechanism (cause → effect), with `file:line`

1. **Routing to the Forward case.** `handle_DATA` (`email_handler.py:2289`) → `_handle` (`email_handler.py:2335`) → `handle` (`email_handler.py:1945`). For each recipient, `email_handler.py:2195` `if is_reverse_alias(rcpt_to):` is **false** for `x@sl.local` (it does not start with a reverse-alias prefix), so control falls to the `else: # Forward case` (`email_handler.py:2201`) which calls `handle_forward(envelope, copy_msg, rcpt_to)` (`email_handler.py:2208`).
2. **Alias lookup fails.** In `handle_forward` (`email_handler.py:536`), `email_handler.py:543` `alias = Alias.get_by(email=alias_address)` returns `None`, so `email_handler.py:544-548` logs *"alias … not exist. Try to see if it can be created on the fly"*.
3. **Auto-create fails because no domain is valid.** `email_handler.py:549` `alias = try_auto_create(alias_address)` (`app/alias_utils.py:202`) tries `try_auto_create_via_domain` (`app/alias_utils.py:220`) then `try_auto_create_directory` (`app/alias_utils.py:222`). The domain gate `is_valid_alias_address_domain()` (`app/email_utils.py:557`) returns `True` only if `SLDomain.get_by(domain=…)` (`app/email_utils.py:560`) or `CustomDomain.get_by(domain=…, verified=True)` (`app/email_utils.py:563`) matches — otherwise `return False` (`app/email_utils.py:566`). With `SLDomain` empty and no verified custom domain for `sl.local`, both fail, so `try_auto_create` returns `None`.
4. **550 is returned.** `email_handler.py:550` `if not alias:` → `email_handler.py:551` logs *"alias … cannot be created on-the-fly, return 550"*; then `email_handler.py:552-553` `if should_ignore_bounce(envelope.mail_from): return [(True, status.E207)]`, else `email_handler.py:554-555` `return [(False, status.E515)]`. `status.E515 = "550 SL E515 Email not exist"` (`app/email/status.py:51`) is delivered verbatim as the SMTP reply.

### Runtime evidence — primary case (`E515`, normal sender)

The email handler was started against `sl_q3` (empty `SLDomain`) — this is also the **second** observation of the `:20381` bind (Q2 stability):

```console
$ CONFIG=/tmp/sl_investigation/q3.env PYTHONUNBUFFERED=1 ./.venv/bin/python email_handler.py
2026-07-06 23:03:42,542 - SL - INFO - 63658 - "<repo>/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-06 23:03:42,544 - SL - DEBUG - 63658 - "<repo>/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

The temporary SMTP-send helper (removed during cleanup) simply calls `smtplib.SMTP("localhost", 20381).sendmail("someone@example.com", ["x@sl.local"], msg)`:

```console
$ ./.venv/bin/python send_q3.py someone@example.com
SENDER=someone@example.com RCPT=x@sl.local
RESULT: SMTPDataError -> code=550 msg=b'SL E515 Email not exist'
```

**(a) SMTP wire reply the sender observes:** `smtplib` raises `smtplib.SMTPDataError` with `code=550` and `msg=b'SL E515 Email not exist'` — i.e. the wire reply is the full line **`550 SL E515 Email not exist`**. Per RFC 5321 §4.2.1 a `5yz` reply is a **permanent** negative completion reply (resending the same message yields the same result).

**(b) The handler's rejection log lines** (verbatim STDOUT for that message — the two `LOG.d` lines are highlighted, shown in full end-to-end context):

```
2026-07-06 23:04:05,288 - SL - INFO - 63658 - "<repo>/email_handler.py:2343" - _handle() - cd681037-8523-4ae5-8849-67f769ea0490 - New message, mail from someone@example.com, rctp tos ['x@sl.local'] 
2026-07-06 23:04:05,290 - SL - DEBUG - 63658 - "<repo>/email_handler.py:1963" - handle() - cd681037-8523-4ae5-8849-67f769ea0490 - Cannot parse Postfix queue ID from None None
2026-07-06 23:04:05,416 - SL - DEBUG - 63658 - "<repo>/email_handler.py:1980" - handle() - cd681037-8523-4ae5-8849-67f769ea0490 - ==>> Handle mail_from:someone@example.com, rcpt_tos:['x@sl.local'], header_from:someone@example.com, header_to:x@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'someone@example.com'), ('To', 'x@sl.local'), ('Subject', 'Q3 test'), ('Content-Transfer-Encoding', '7bit')], mail_options:['SIZE=91'], rcpt_options:[]
2026-07-06 23:04:05,421 - SL - DEBUG - 63658 - "<repo>/email_handler.py:2202" - handle() - cd681037-8523-4ae5-8849-67f769ea0490 - Forward phase someone@example.com(someone@example.com) -> x@sl.local
2026-07-06 23:04:05,432 - SL - DEBUG - 63658 - "<repo>/email_handler.py:545" - handle_forward() - cd681037-8523-4ae5-8849-67f769ea0490 - alias x@sl.local not exist. Try to see if it can be created on the fly
2026-07-06 23:04:05,442 - SL - INFO - 63658 - "<repo>/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - cd681037-8523-4ae5-8849-67f769ea0490 - Cannot auto-create custom domain alias for x@sl.local because there's no custom domain for sl.local
2026-07-06 23:04:05,442 - SL - INFO - 63658 - "<repo>/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - cd681037-8523-4ae5-8849-67f769ea0490 - Cannot auto-create x@sl.local since it has no directory separator
2026-07-06 23:04:05,442 - SL - DEBUG - 63658 - "<repo>/email_handler.py:551" - handle_forward() - cd681037-8523-4ae5-8849-67f769ea0490 - alias x@sl.local cannot be created on-the-fly, return 550
2026-07-06 23:04:05,443 - SL - INFO - 63658 - "<repo>/email_handler.py:2367" - _handle() - cd681037-8523-4ae5-8849-67f769ea0490 - Finish mail_from someone@example.com, rcpt_tos ['x@sl.local'], takes 0.15468144416809082 seconds with return code '550 SL E515 Email not exist'<<===
```

The two rejection log lines requested by the question, verbatim:

- `alias x@sl.local not exist. Try to see if it can be created on the fly` — `email_handler.py:545` (the `LOG.d(...)` call spanning `email_handler.py:544-548`).
- `alias x@sl.local cannot be created on-the-fly, return 550` — `email_handler.py:551`.

The final `_handle` line (`email_handler.py:2367`) independently confirms the returned wire string: `return code '550 SL E515 Email not exist'`. The two intervening `app/alias_utils.py:104` / `:165` lines show *why* auto-create failed (no custom domain for `sl.local`; no directory separator in the local part).

### Runtime evidence — edge case (`E207`, IgnoreBounceSender)

`should_ignore_bounce(mail_from)` (`app/email_utils.py:1361`) returns `True` only when `IgnoreBounceSender.get_by(mail_from=mail_from)` matches (`app/email_utils.py:1362`), logging a warning (`app/email_utils.py:1363`). To exercise this branch, a temporary `IgnoreBounceSender` row (table `ignore_bounce_sender`, `app/models.py:3354`) was inserted into the throwaway `sl_q3` DB (`SLDomain` still empty), then a message was sent **from that sender**:

```console
$ CONFIG=/tmp/sl_investigation/q3.env ./.venv/bin/python -c \
    "from app.models import IgnoreBounceSender; from app.db import Session; \
     r=IgnoreBounceSender.create(mail_from='bounce-sender@example.com'); Session.commit(); \
     print('inserted id=%s mail_from=%s; count=%s' % (r.id, r.mail_from, IgnoreBounceSender.count()))"
inserted id=1 mail_from=bounce-sender@example.com; count=1
# SLDomain still empty:
$ psql -h localhost -p 15432 -U myuser -d sl_q3 -tAc "SELECT count(*) FROM public_domain;"
0

$ ./.venv/bin/python send_q3.py bounce-sender@example.com
SENDER=bounce-sender@example.com RCPT=x@sl.local
RESULT: sendmail returned normally (no exception) -> ACCEPTED
```

**Wire reply:** the sender sees a **`250`** (acceptance) — `smtplib.sendmail` returns normally with no exception. The handler log shows the alternate branch taken and the `250 … E207 …` return code:

```
2026-07-06 23:05:01,167 - SL - DEBUG - 63658 - "<repo>/email_handler.py:545" - handle_forward() - 85cbff24-ccbd-4139-9f44-d0770410d935 - alias x@sl.local not exist. Try to see if it can be created on the fly
2026-07-06 23:05:01,171 - SL - DEBUG - 63658 - "<repo>/email_handler.py:551" - handle_forward() - 85cbff24-ccbd-4139-9f44-d0770410d935 - alias x@sl.local cannot be created on-the-fly, return 550
2026-07-06 23:05:01,172 - SL - WARNING - 63658 - "<repo>/app/email_utils.py:1363" - should_ignore_bounce() - 85cbff24-ccbd-4139-9f44-d0770410d935 - do not send back bounce report to bounce-sender@example.com
2026-07-06 23:05:01,172 - SL - INFO - 63658 - "<repo>/email_handler.py:2367" - _handle() - 85cbff24-ccbd-4139-9f44-d0770410d935 - Finish mail_from bounce-sender@example.com, rcpt_tos ['x@sl.local'], takes 0.017747163772583008 seconds with return code '250 SL E207 No bounce report'
```

So when `mail_from` is an `IgnoreBounceSender`, `handle_forward` returns `status.E207 = "250 SL E207 No bounce report"` (`email_handler.py:552-553`, `app/email/status.py:12`) instead of `E515`: the message is silently accepted (a `250`), and no bounce report is generated. This was **observed at runtime**, not merely inferred.

### Stability (≥2 runs)

The primary `E515` case was exercised twice against the empty `SLDomain`. Run #2 (sender `normal-2@example.com`) was identical:

```
2026-07-06 23:04:45,525 - SL - DEBUG - 63658 - "<repo>/email_handler.py:545" - handle_forward() - 99889f7a-4f07-4d17-8f06-23ebfe0040c2 - alias x@sl.local not exist. Try to see if it can be created on the fly
2026-07-06 23:04:45,548 - SL - DEBUG - 63658 - "<repo>/email_handler.py:551" - handle_forward() - 99889f7a-4f07-4d17-8f06-23ebfe0040c2 - alias x@sl.local cannot be created on-the-fly, return 550
2026-07-06 23:04:45,549 - SL - INFO - 63658 - "<repo>/email_handler.py:2367" - _handle() - 99889f7a-4f07-4d17-8f06-23ebfe0040c2 - Finish mail_from normal-2@example.com, rcpt_tos ['x@sl.local'], takes 0.03571367263793945 seconds with return code '550 SL E515 Email not exist'
```

Result: **stable** — empty `SLDomain` + normal sender → `550 SL E515 Email not exist` on every run.

**Q3 ↔ omitted step:** this rejection is caused by **skipping `init_app.py`**. Migrations create the `public_domain` table but leave it empty; with no `SLDomain` row for `sl.local`, `is_valid_alias_address_domain()` is false, auto-create fails, and the handler returns `E515`.

---


## (e) Canonical startup order + failure modes

**Correct startup order:**

```
database  →  migrations           →  init_app.py            →  web app                →  email handler            →  background workers
(PostgreSQL,  (flask db upgrade /     (seeds SLDomain via       (python server.py :7777   (python email_handler.py    (job_runner.py,
 create DB)    alembic upgrade head)   add_sl_domains())         / gunicorn wsgi:app)      :20381 / :25 prod)          event_listener.py, cron.py)
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

**Why this order.** PostgreSQL must exist first because the engine connects at import (`app/db.py:12`). Migrations must run next to create the 77 tables (Alembic via Flask-Migrate; `alembic.ini:5` `script_location = migrations`; `migrations/env.py:28` `target_metadata = Base.metadata`, `migrations/env.py:62,87` `run_migrations_online()`; **255** revision files under `migrations/versions/`). `init_app.py` then seeds `SLDomain` (`init_app.py:39,69-73`) so that inbound mail to `@sl.local` is recognized. Only then are the web app, email handler, and background workers meaningful.

**Verified canonical references for this sequence:**

- `README.md`: `flask db upgrade` (`README.md:433`) → `python init_app.py` (`README.md:448`) → `python email_handler.py` (`README.md:480`) → `python job_runner.py` (`README.md:495`).
- `CONTRIBUTING.md`: Python 3.10 + poetry (`CONTRIBUTING.md:23`); PostgreSQL via `docker run … postgres:13 … -p 15432:5432` (`CONTRIBUTING.md:100`); dev one-liner `alembic upgrade head && flask dummy-data && python3 server.py` (`CONTRIBUTING.md:106`); then open `http://localhost:7777` (`CONTRIBUTING.md:109`).
- `scripts/reset_local_db.sh`: `drop schema public cascade; create schema public;` piped to `psql` (`scripts/reset_local_db.sh:4`) → `poetry run alembic upgrade head` (`:6`) → `poetry run flask dummy-data` (`:7`).
- `.github/workflows/main.yml`: provisions `postgres:13` (`.github/workflows/main.yml:47`) and runs `CONFIG=tests/test.env poetry run alembic upgrade head` (`:98`).

**Failure modes — cause and effect of skipping a step (backed by the captured output above):**

| Skipped step | Symptom (observed) | Mechanism |
|--------------|--------------------|-----------|
| **Migrations** (→ Q1) | `python server.py` binds `:7777`, but the first login POST returns **HTTP 500** with `sqlalchemy.exc.ProgrammingError` / `psycopg2.errors.UndefinedTable: relation "users" does not exist` | Engine connects at import with no DDL check (`app/db.py:12`), so startup succeeds; the request-time ORM query (`app/auth/views/login.py:43` → `app/models.py:84`) hits a table the migrations never created. |
| **`init_app.py`** (→ Q3) | Migrations applied, but mail to `x@sl.local` is rejected with **`550 SL E515 Email not exist`** | `public_domain`/`SLDomain` is empty, so `is_valid_alias_address_domain()` is false (`app/email_utils.py:557-566`); `handle_forward()` auto-create fails and returns `status.E515` (`email_handler.py:551-555`, `app/email/status.py:51`). |

Both failure modes were reproduced first-hand: **Q1 ↔ omitted migration step**, **Q3 ↔ omitted `init_app.py` step**.

---

## Appendix — Verified `file:line` reference map

All line numbers below were confirmed against the source at investigation time.

**Q1 (empty-DB exception):**
- `app/db.py:9-11` engine creation; `app/db.py:12` `connection = engine.connect()` (import-time, no DDL check).
- `server.py:163-165` `if MEM_STORE_URI:` (Redis/session gate, unset by default); `server.py:250-256` `/` → `auth.login`; `server.py:572` `local_main()`; `server.py:588` `app.run(debug=True, port=7777)`; `server.py:598-599` entrypoint; `server.py:284` `after_request` log; `server.py:390` `error_handler`.
- `app/auth/views/login.py:21` route; `:40` `validate_on_submit()`; `:43` `User.get_by(email=...)`.
- `app/models.py:84` `get_by` → `Session.query(cls).filter_by(**kw).first()`; `app/models.py:337` `User.__tablename__ = "users"`.
- Render chain: `templates/auth/login.html` → `templates/single.html` → `templates/base.html:89` (anonymous short-circuit).
- Observed: `sqlalchemy.exc.ProgrammingError` wrapping `psycopg2.errors.UndefinedTable: relation "users" does not exist` → HTTP 500; SQLAlchemy 1.3 URL `sqlalche.me/e/13/f405`.

**Q2 (required services):**
- Web app: `server.py:588` (`:7777`); `wsgi.py:1,3` `create_app()` (Gunicorn `wsgi:app`).
- Email handler: `email_handler.py:2381` `main`, `:2383` `Controller(hostname="0.0.0.0", port)`, `:2385` `start()`, `:2386` `Start mail controller`, `:2399` default `20381`, `:2403` `Listen for port`, `:2404` `main`.
- Job runner: `job_runner.py:329` `__main__`, `:347` `time.sleep(10)`.
- Event listener: `event_listener.py:29` `main`, `:34` `Using PostgresEventSource`, `:35` `PostgresEventSource(EVENT_LISTENER_DB_URI)`, `:95-97` no-subcommand usage/exit.
- Cron: `cron.py:1` argparse, `:1262` `__main__`, `:1263` `Start running cronjob`, `:1274+` `args.job` dispatch.
- Log: `app/log.py:12-14` format, `:41` `StreamHandler(sys.stdout)`, `:51` DEBUG, `:67` `>>> init logging <<<`, `:74-77` `d/i/w/e`, `:79` `LOG = _get_logger("SL")`.

**Q3 (skipped-init SMTP rejection):**
- Dispatch: `email_handler.py:2289` `handle_DATA`, `:2335` `_handle`, `:1945` `handle`, `:2195` `is_reverse_alias`, `:2201` `else # Forward case`, `:2208` `handle_forward(...)`.
- `handle_forward`: `email_handler.py:536` def, `:543` `Alias.get_by(email=...)`, `:544-548` "not exist" log (`:545`), `:549` `try_auto_create`, `:551` "cannot be created on-the-fly, return 550" log, `:552-553` `should_ignore_bounce` → `E207`, `:554-555` `E515`; `:2367` final `_handle` "return code" line.
- `app/alias_utils.py:202` `try_auto_create`, `:220` `try_auto_create_via_domain`, `:222` `try_auto_create_directory`, `:104`/`:165` auto-create-failure logs.
- `app/email_utils.py:557` `is_valid_alias_address_domain`, `:560` `SLDomain.get_by`, `:563` `CustomDomain.get_by(verified=True)`, `:566` `return False`; `:1361` `should_ignore_bounce`, `:1363` "do not send back bounce report" log.
- `app/email/status.py:51` `E515 = "550 SL E515 Email not exist"`; `app/email/status.py:12` `E207 = "250 SL E207 No bounce report"`.
- `app/models.py:3116` `SLDomain` → `__tablename__ = "public_domain"`; `app/models.py:3354` `IgnoreBounceSender` → `__tablename__ = "ignore_bounce_sender"`.
- `init_app.py:39` `add_sl_domains`, `:44` `Add … to SL domain` log, `:69-73` `__main__`; `app/config.py:157-160` `ALIAS_DOMAINS` default.

**Startup order / env / runtime:**
- `README.md:433/448/480/495`; `CONTRIBUTING.md:23/100/106/109/236/248`; `scripts/reset_local_db.sh:4/6/7`; `.github/workflows/main.yml:47/98`.
- `alembic.ini:5`; `migrations/env.py:28/62/87`; 255 files under `migrations/versions/`.
- `example.env:6/19/22/40/75/77`; `app/config.py:65-69` (`CONFIG` dotenv), `:14-19` (`get_abs_path`).
- `pyproject.toml:61` (`python = "^3.10"`), `:66` gunicorn, `:71` psycopg2-binary, `:77` Flask-Migrate, `:87` aiosmtpd, `:116` `SQLAlchemy = 1.3.24`; `Dockerfile:8` `FROM python:3.10`.

