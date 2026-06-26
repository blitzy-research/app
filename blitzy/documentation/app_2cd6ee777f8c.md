# SimpleLogin — First-Time Local-Development Startup Behavior (Q&A)

This document walks through bringing up the **whole SimpleLogin stack from scratch** on a
fresh local-development machine and records *what actually happens along the way*. It answers
three specific questions:

1. **Q1 —** With PostgreSQL running and an **empty** database (the database exists, but the
   migrations have **not** been run, so there are no tables), what is the **full Python
   exception/traceback** when the server is started with `python server.py` and the login page
   is opened in a browser?
2. **Q2 —** After the migrations **and** the initialization script (`init_app.py`) have run,
   **which Python services must be running** for the system to work, and what is the **verbatim
   stdout/stderr** that confirms each one is up, bound to its port, and ready to accept
   connections?
3. **Q3 —** If the migrations are run but `init_app.py` is **not** run (so no email domains are
   configured), and an email addressed to any `@sl.local` address is received by the email
   handler, what **SMTP status code** is returned to the sender, and what **log lines** explain
   the rejection?

> **Every code block below is captured from a real run** of the stack — the literal
> stdout/stderr, traceback, SMTP transcript, and log lines as actually emitted. Reading the
> source tells us *what to expect and why*; the captured artifacts are the *evidence*. Each
> answer ends with a code-grounded rationale citing the responsible source `file:line`.

---

## 0. Runtime environment & methodology

All three experiments were reproduced inside the project's documented Docker runtime
(`andrewparkscaleai/coding-agent:simple-login__app__2cd6ee777f8c...`, built on `python:3.10`
per `Dockerfile`). The observed runtime is:

| Component | Observed value |
|-----------|----------------|
| Python | `3.10.18` (project venv at `/app/venv`) |
| PostgreSQL | `15.13` on `localhost:5432`, database `simplelogin`, role `myuser` |
| Flask / Werkzeug | `1.1.2` / `1.0.1` |
| SQLAlchemy / psycopg2-binary | `1.3.24` / `2.9.3` |
| aiosmtpd | `1.4.2` |
| alembic | `1.4.3` |
| gunicorn | `20.0.4` |

The repository is checked out at `/workspace` (the repository root — this is the path that
appears in every captured log line below).

**Local configuration.** Per `CONTRIBUTING.md` ("Run the code locally"), the only setup needed
is a `.env` file. It is created by copying the shipped template — `cp example.env .env` — which
already provides every variable `app/config.py` reads **at import time**:

- `URL=http://localhost:7777` (`example.env:6`) → consumed by `URL = os.environ["URL"]` (`app/config.py:79`)
- `EMAIL_DOMAIN=sl.local` (`example.env:22`) → consumed by `EMAIL_DOMAIN = os.environ["EMAIL_DOMAIN"].lower()` (`app/config.py:92`)
- `DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin` (`example.env:75`) → consumed by `DB_URI = os.environ["DB_URI"]` (`app/config.py:192`)
- `NOT_SEND_EMAIL=true` (`example.env:19`), `FLASK_SECRET=secret` (`example.env:77`)

> This `.env` is a **temporary, non-committed** observation aid (it is already in `.gitignore`);
> it is not part of this deliverable.

A consequence of `EMAIL_DOMAIN=sl.local` is that `app/config.py:160` computes
`ALIAS_DOMAINS = OTHER_ALIAS_DOMAINS + [EMAIL_DOMAIN] == ["sl.local"]`. So `sl.local` is exactly
the domain that **`init_app.py` would register** — which is why *skipping* `init_app.py` (Q3)
leaves the domain table empty.

### 0.1 The documented startup order

SimpleLogin's backend "consists of 2 main components: the webapp and the email handler"
(`CONTRIBUTING.md`, "General Architecture"), supported by background workers. The documented
bring-up order is:

```
PostgreSQL up
   →  alembic upgrade head        # create the schema (CONTRIBUTING.md:106, alembic.ini:5)
   →  python init_app.py          # seed SL domains + load PGP keys (README.md:441-448)
   →  start services              # webapp / email_handler / job_runner
```

- The documented **local** path (`CONTRIBUTING.md:88-109`): `cp example.env .env` →
  `alembic upgrade head && flask dummy-data && python3 server.py` → open
  `http://localhost:7777` and log in with `john@wick.com / password`.
- The reference **Docker** deployment (`README.md:441-495`) runs four steps in order:
  `init_app.py` (container `sl-init`), then the webapp (`sl-app`, `-p 127.0.0.1:7777:7777`),
  then `python email_handler.py` (`sl-email`, `-p 127.0.0.1:20381:20381`), then
  `python job_runner.py` (`sl-job-runner`).

The three experiments below **deliberately deviate** from this order where a question requires
it: Q1 **skips** the migrations (empty DB), and Q3 **skips** `init_app.py` (migrated but
uninitialized DB). Q2 follows the full order.

### 0.2 TL;DR — the three answers

| # | Question | Answer (captured live) |
|---|----------|------------------------|
| **Q1** | Empty-DB error on opening login | `sqlalchemy.exc.ProgrammingError: (psycopg2.errors.UndefinedTable) relation "users" does not exist` — raised on the **login POST** (not on page render), surfaced as **HTTP 500** + the custom `error/500.html` page, with the full traceback written to the server's stderr by `LOG.e(e)` (`server.py:390`). |
| **Q2** | Required services + readiness | **webapp** (`server.py`, binds `127.0.0.1:7777`), **email handler** (`email_handler.py`, binds `0.0.0.0:20381`), **job runner** (`job_runner.py`, no port). `event_listener.py` and `cron.py` are auxiliary. |
| **Q3** | Email rejection, `init_app.py` skipped | **`550 SL E515 Email not exist`** returned to the sender by `handle_forward` (`email_handler.py:555`, `app/email/status.py:51`). |

---

## Q1 — The empty-database error

**Database state:** PostgreSQL is running and the `simplelogin` database exists, but it is
**empty / un-migrated** — `alembic upgrade head` was deliberately **not** run, so there are no
tables.

```console
$ echo 'drop schema public cascade; create schema public;' \
    | psql postgresql://myuser:mypassword@localhost:5432/simplelogin
DROP SCHEMA
CREATE SCHEMA

$ psql ... -c "\dt"
Did not find any relations.          # 0 tables — empty DB
```

### Reproduction steps

```console
# 1) Start the webapp exactly as documented (python server.py -> local_main() -> app.run(debug=True, port=7777))
$ python server.py
```

The server **starts successfully even though the DB has no tables** — startup banner (verbatim):

```text
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/dfwvbznqgitmjxuvsxpy
Upload files to local dir
>>> init logging <<<
2026-06-26 21:26:21,161 - SL - DEBUG - 13789 - "/workspace/app/utils.py:17" - <module>() -  - load words file: /workspace/local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
```

> Note: the familiar Werkzeug `* Running on http://127.0.0.1:7777/` access-banner line is
> **absent** because SimpleLogin disables the `werkzeug` logger at import (`app/log.py:70-71`,
> `log.disabled = True`). The `* Serving Flask app ... / * Debug mode: on` banner is printed by
> Flask's `run()` itself, so it still appears. Readiness is confirmed by `GET /health` (below).

Then drive the login flow in a browser (reproduced here with `requests`, preserving the
session cookie + CSRF token):

```console
# 2) GET /  -> anonymous user is redirected to the login page (no DB query)
$ curl -i http://localhost:7777/
HTTP/1.0 302 FOUND
Location: http://localhost:7777/auth/login
...

# 3) GET /auth/login  -> the login page RENDERS CLEANLY (HTTP 200) against the empty DB
$ curl -s -o /dev/null -w "%{http_code}\n" http://localhost:7777/auth/login
200

# 4) POST /auth/login with the documented credentials -> HTTP 500
GET  /auth/login -> 200; csrf_token present: True
POST /auth/login -> 500
```

The `POST` returns **HTTP 500**, and the response body is SimpleLogin's **custom `error/500.html`
page** (note the `| SimpleLogin` title) — *not* the Werkzeug interactive debugger:

```html
<!DOCTYPE html>
<html lang="en" dir="ltr" data-theme="">
  <head>
    ...
    <title>
       | SimpleLogin
    </title>
    ...
```

### Where the error surfaces — render (`GET`) vs. submit (`POST`)

The server's own request log settles this unambiguously: every `GET /auth/login` returns `200`
(the page renders with no `users` query), and the error only fires on the `POST`:

```text
... - SL - DEBUG - 13823 - "/workspace/server.py:284" - after_request() -  - 127.0.0.1 GET / ImmutableMultiDict([]) 302, takes 0.00086...
... - SL - DEBUG - 13823 - "/workspace/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.12902...
... - SL - DEBUG - 13823 - "/workspace/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.03497...
... - SL - DEBUG - 13823 - "/workspace/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.12499...
... - SL - ERROR - 13823 - "/workspace/server.py:390" - error_handler() -  - (psycopg2.errors.UndefinedTable) relation "users" does not exist
... - SL - DEBUG - 13823 - "/workspace/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 500, takes 0.01595...
```

**Conclusion (observed):** the missing-table error is triggered by the login **`POST`
(submission)**, not by opening/rendering the page. The `GET` render path performs no query
against `users`.

### The full, verbatim Python traceback

Because `server.py`'s global handler `@app.errorhandler(Exception)` (`server.py:388`) catches
the exception, calls `LOG.e(e)` (`server.py:390`), and renders the 500 page, the **complete
traceback is written to the server's stderr** (`LOG.e` is an alias of `logging.Logger.exception`,
`app/log.py:77`). This is the exact block emitted (captured from `python server.py`, unedited):

```text
2026-06-26 21:27:33,857 - SL - ERROR - 13823 - "/workspace/server.py:390" - error_handler() -  - (psycopg2.errors.UndefinedTable) relation "users" does not exist
LINE 2: FROM users 
             ^

[SQL: SELECT users.directory_quota AS users_directory_quota, users.subdomain_quota AS users_subdomain_quota, users.password AS users_password, users.id AS users_id, users.created_at AS users_created_at, users.updated_at AS users_updated_at, users.email AS users_email, users.name AS users_name, users.is_admin AS users_is_admin, users.alias_generator AS users_alias_generator, users.notification AS users_notification, users.activated AS users_activated, users.disabled AS users_disabled, users.profile_picture_id AS users_profile_picture_id, users.otp_secret AS users_otp_secret, users.enable_otp AS users_enable_otp, users.last_otp AS users_last_otp, users.fido_uuid AS users_fido_uuid, users.default_alias_custom_domain_id AS users_default_alias_custom_domain_id, users.default_alias_public_domain_id AS users_default_alias_public_domain_id, users.lifetime AS users_lifetime, users.paid_lifetime AS users_paid_lifetime, users.lifetime_coupon_id AS users_lifetime_coupon_id, users.trial_end AS users_trial_end, users.default_mailbox_id AS users_default_mailbox_id, users.sender_format AS users_sender_format, users.sender_format_updated_at AS users_sender_format_updated_at, users.replace_reverse_alias AS users_replace_reverse_alias, users.referral_id AS users_referral_id, users.intro_shown AS users_intro_shown, users.max_spam_score AS users_max_spam_score, users.newsletter_alias_id AS users_newsletter_alias_id, users.include_sender_in_reverse_alias AS users_include_sender_in_reverse_alias, users.random_alias_suffix AS users_random_alias_suffix, users.expand_alias_info AS users_expand_alias_info, users.ignore_loop_email AS users_ignore_loop_email, users.alternative_id AS users_alternative_id, users.disable_automatic_alias_note AS users_disable_automatic_alias_note, users.one_click_unsubscribe_block_sender AS users_one_click_unsubscribe_block_sender, users.include_website_in_one_click_alias AS users_include_website_in_one_click_alias, users.disable_import AS users_disable_import, users.can_use_phone AS users_can_use_phone, users.phone_quota AS users_phone_quota, users.block_behaviour AS users_block_behaviour, users.include_header_email_header AS users_include_header_email_header, users.enable_data_breach_check AS users_enable_data_breach_check, users.flags AS users_flags, users.unsub_behaviour AS users_unsub_behaviour, users.delete_on AS users_delete_on 
FROM users 
WHERE users.email = %(email_1)s 
 LIMIT %(param_1)s]
[parameters: {'email_1': 'john@wick.com', 'param_1': 1}]
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
  File "/workspace/app/auth/views/login.py", line 43, in login
    user = User.get_by(email=email) or User.get_by(email=canonical_email)
  File "/workspace/app/models.py", line 84, in get_by
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
[parameters: {'email_1': 'john@wick.com', 'param_1': 1}]
(Background on this error at: http://sqlalche.me/e/13/f405)
```

**The exception is a `sqlalchemy.exc.ProgrammingError` (SQLAlchemy 1.3.24) directly caused by
`psycopg2.errors.UndefinedTable: relation "users" does not exist`**, with the offending query
`SELECT ... FROM users WHERE users.email = %(email_1)s LIMIT %(param_1)s` and parameters
`{'email_1': 'john@wick.com', 'param_1': 1}`.

### Production form (gunicorn) — same error, cleaner stack

For completeness, the same `POST` against the **production** WSGI form
(`gunicorn wsgi:app -b 0.0.0.0:7777 -w 2`, where `wsgi.py` is `from server import create_app;
app = create_app()`) produces the **identical** `ProgrammingError` and the same HTTP 500 +
`error/500.html`. The only difference is the call stack lacks the dev-only
`flask_debugtoolbar`/`cProfile` frames:

```text
[2026-06-26 21:28:56 +0000] [13954] [INFO] Starting gunicorn 20.0.4
[2026-06-26 21:28:56 +0000] [13954] [INFO] Listening at: http://0.0.0.0:7777 (13954)
[2026-06-26 21:28:56 +0000] [13954] [INFO] Using worker: sync
[2026-06-26 21:28:56 +0000] [13964] [INFO] Booting worker with pid: 13964
...
... - SL - ERROR - 13964 - "/workspace/server.py:390" - error_handler() -  - (psycopg2.errors.UndefinedTable) relation "users" does not exist
...
Traceback (most recent call last):
  File "/app/venv/lib/python3.10/site-packages/flask/app.py", line 1950, in full_dispatch_request
    rv = self.dispatch_request()
  File "/app/venv/lib/python3.10/site-packages/flask/app.py", line 1936, in dispatch_request
    return self.view_functions[rule.endpoint](**req.view_args)
  File "/app/venv/lib/python3.10/site-packages/flask_limiter/extension.py", line 702, in __inner
    return obj(*a, **k)
  File "/workspace/app/auth/views/login.py", line 43, in login
    user = User.get_by(email=email) or User.get_by(email=canonical_email)
  File "/workspace/app/models.py", line 84, in get_by
    return Session.query(cls).filter_by(**kw).first()
  ...
sqlalchemy.exc.ProgrammingError: (psycopg2.errors.UndefinedTable) relation "users" does not exist
```

### Rationale — why this happens

- **Why the server starts at all against an empty DB.** `app/db.py` builds the engine and opens
  a connection **at import time** — `engine = create_engine(config.DB_URI, ...)` (`app/db.py:9-11`)
  and `connection = engine.connect()` (`app/db.py:12`), then `Session = scoped_session(...)`
  (`app/db.py:14`). Connecting requires the *database* to exist, but it does **not** require any
  *tables*. That is why `python server.py` boots cleanly and `/health` works even with zero
  tables — the schema is only touched on the first ORM query.
- **Why `GET` is fine but `POST` fails.** `GET /` → `index()` reads
  `current_user.is_authenticated` for an anonymous user (no query) and redirects to
  `auth.login` (`server.py:251-255`). `GET /auth/login` renders `auth/login.html` and does
  **not** query `users` (`app/auth/views/login.py:25-83`). Only on submit does
  `form.validate_on_submit()` become true (`app/auth/views/login.py:40`) and the view run the
  **first** `users` query: `user = User.get_by(email=email) or User.get_by(email=canonical_email)`
  (`app/auth/views/login.py:43`).
- **Why this query maps to the missing `users` relation.** `ModelMixin.get_by` executes
  `Session.query(cls).filter_by(**kw).first()` (`app/models.py:83-84`) against `User`, whose
  `__tablename__ = "users"` (`app/models.py:336-337`). With no `users` table, psycopg2 raises
  `UndefinedTable`, which SQLAlchemy wraps as `ProgrammingError`.
- **Why the browser sees a 500 page (not the interactive debugger).** Even though the server
  runs with `debug=True` (`server.py:581,588`), Flask's `handle_user_exception` finds the
  registered `@app.errorhandler(Exception)` handler (`server.py:388`) and invokes it **before**
  the exception can propagate to Werkzeug's interactive debugger. The handler logs the full
  traceback with `LOG.e(e)` (`server.py:390`) and, for a non-`/api/` path, returns
  `render_template("error/500.html"), 500` (`server.py:394`). The production (gunicorn) form has
  no debugger at all, so it follows the same handler — which is why both forms behave
  identically here.

**Source map for Q1:** `app/db.py:9-14` · `server.py:251-255` · `app/auth/views/login.py:40,43`
· `app/models.py:83-84,336-337` · `server.py:388-394` · `server.py:581,588` · `app/log.py:70-71,77`.

---


## Q2 — Required services & their readiness logs

**Database state:** fully initialized — `alembic upgrade head` **then** `python init_app.py`.
The initialization seeds the SL domain table; running `init_app.py` emits:

```text
... - SL - DEBUG - 14185 - "/workspace/init_app.py:36" - load_pgp_public_keys() -  - Finish load_pgp_public_keys
... - SL - INFO - 14185 - "/workspace/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain
```

```console
$ psql ... -c "SELECT id, domain, use_as_reverse_alias, premium_only FROM public_domain ORDER BY id;"
 id |  domain  | use_as_reverse_alias | premium_only
----+----------+----------------------+--------------
  1 | sl.local | t                    | f
(1 row)
```

### Service inventory

The reference Docker deployment (`README.md:441-495`) runs **three** long-lived Python
processes. These are the services that must be running for the system to work:

| Service | Command | Binds | Readiness signal |
|---------|---------|-------|------------------|
| **Webapp** | `python server.py` | `127.0.0.1:7777` | `* Serving Flask app` / `* Debug mode: on`; `GET /health` → `success` |
| **Email handler** | `python email_handler.py` | `0.0.0.0:20381` | `Listen for port 20381` + `Start mail controller 0.0.0.0 20381`; SMTP `220` greeting |
| **Job runner** | `python job_runner.py` | *(no port)* | enters the `while True` poll loop (`time.sleep(10)`) |

Two further scripts are **auxiliary** (not long-lived listeners; not required for the core
flows): `event_listener.py` (its `__main__` requires a subcommand, so a bare invocation does not
start a listener) and `cron.py` (scheduled tasks invoked via `yacron`/`crontab.yml`).
`wsgi.py` is not a separate service — it is the gunicorn import target
(`from server import create_app; app = create_app()`) used to run the webapp in production.

### 2a. Webapp — `python server.py` (binds `127.0.0.1:7777`)

Startup banner (verbatim):

```text
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/cbaysamragrrdmgxtrqq
Upload files to local dir
>>> init logging <<<
2026-06-26 21:33:33,710 - SL - DEBUG - 14205 - "/workspace/app/utils.py:17" - <module>() -  - load words file: /workspace/local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
```

Port-binding proof:

```console
$ curl -s http://localhost:7777/health ; echo " [HTTP $(curl -s -o /dev/null -w %{http_code} http://localhost:7777/health)]"
success [HTTP 200]

# listening socket
LISTEN  127.0.0.1:7777
```

The `/health` route returns `"success", 200` (`server.py:213-215`); the bind to
`127.0.0.1:7777` comes from `app.run(debug=True, port=7777)` (`server.py:588`). The production
form is `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2`.

### 2b. Email handler — `python email_handler.py` (binds `0.0.0.0:20381`)

Readiness lines (verbatim):

```text
... - SL - INFO - 14206 - "/workspace/email_handler.py:2403" - <module>() -  - Listen for port 20381
... - SL - DEBUG - 14206 - "/workspace/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

Port-binding proof (the SMTP `220` greeting is the strongest proof the controller is accepting
connections):

```console
$ python -c "import smtplib; c=smtplib.SMTP(); print(c.connect('localhost',20381)); print(c.ehlo('probe.local')[0]); c.quit()"
(220, b'45fc3e9dd28a Python SMTP 1.4.2')
250

# listening socket
LISTEN  0.0.0.0:20381
```

In `__main__`, argparse defaults the port to `20381` and logs `Listen for port 20381`
(`email_handler.py:2399,2403`); `main(port)` then creates
`Controller(MailHandler(), hostname="0.0.0.0", port=port)` (`email_handler.py:2383`), calls
`controller.start()` (`email_handler.py:2385`), logs `Start mail controller 0.0.0.0 20381`
(`email_handler.py:2386`), and idles in `while True: time.sleep(2)` (`email_handler.py:2392-2393`).
The `220 ... Python SMTP 1.4.2` greeting is emitted by the underlying `aiosmtpd` 1.4.2
`Controller`.

### 2c. Job runner — `python job_runner.py` (no port)

The job runner emits the standard initialization preamble and then goes **quiet** — it is
"ready" once it enters its polling loop and will only log again when it picks up a job:

```text
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/vktffpalwqvrhklyxxgh
Upload files to local dir
>>> init logging <<<
2026-06-26 21:33:45,686 - SL - DEBUG - 14311 - "/workspace/app/utils.py:17" - <module>() -  - load words file: /workspace/local_data/test_words.txt
```

It binds no port; its `__main__` (`job_runner.py:329`) enters `while True:` (`job_runner.py:330`),
opens an app context, drains `get_jobs_to_run()`, and `time.sleep(10)` (`job_runner.py:347`).
(With no queued jobs it prints nothing further — that silence *is* the steady state.)

### All three processes running

```console
$ ps -eo pid,cmd | grep -E "[s]erver.py|[e]mail_handler.py|[j]ob_runner.py"
  14205 /app/venv/bin/python server.py
  14206 /app/venv/bin/python email_handler.py
  14274 /app/venv/bin/python /workspace/server.py     # the dev-server reloader child
  14311 /app/venv/bin/python job_runner.py
```

### Rationale — why these are the required services

- **Webapp** serves the entire HTTP/UI surface and the API; it is the process that binds
  `127.0.0.1:7777` (`server.py:588`) and answers `/health` (`server.py:213-215`).
- **Email handler** is the SMTP entry point for all inbound mail. Without it, no alias email is
  delivered, forwarded, or replied to. It binds `0.0.0.0:20381` via the `aiosmtpd` `Controller`
  (`email_handler.py:2383-2386`) and is the process exercised in Q3.
- **Job runner** executes asynchronous/background work (account deletion, batch imports, etc.)
  by polling the job queue (`job_runner.py:330,347`). It is the third process in the reference
  deployment (`README.md:486-495`).

**Source map for Q2:** `server.py:213-215,588` · `email_handler.py:2383,2385,2386,2399,2403` ·
`job_runner.py:329,330,347` · `wsgi.py` · `README.md:441-495` · `init_app.py:36,44`.

---


## Q3 — Email rejection when `init_app.py` is skipped

**Database state:** `alembic upgrade head` was run (schema present) **but `python init_app.py`
was NOT run.** Because `add_sl_domains()` (`init_app.py:39`, called from `__main__` at
`init_app.py:73`) never executes, the `SLDomain` table — physical table **`public_domain`**
(`app/models.py:3116,3119`) — stays **empty**:

```console
$ psql ... -c "SELECT count(*) AS public_domain_rows FROM public_domain;"
 public_domain_rows
--------------------
                  0
(1 row)
```

So **no email domains are configured**, even though `EMAIL_DOMAIN=sl.local` (and therefore
`ALIAS_DOMAINS == ["sl.local"]`, `app/config.py:160`) — the domain `sl.local` simply was never
inserted.

### Reproduction — inject an email to `anything@sl.local`

With the email handler running on `0.0.0.0:20381` (started exactly as in Q2), a throwaway
`smtplib` script injects a minimal message to `anything@sl.local`. **Crucially, the message
carries no SpamAssassin/milter (`X-Spam*` / `X-Spamd-Result`) headers** (see the SPF caveat
below). The full SMTP wire transcript (`smtplib` debug output, verbatim):

```text
send: 'ehlo tester.local\r\n'
reply: b'250-45fc3e9dd28a\r\n'
reply: b'250-SIZE 33554432\r\n'
reply: b'250-8BITMIME\r\n'
reply: b'250-SMTPUTF8\r\n'
reply: b'250 HELP\r\n'
send: 'mail FROM:<somebody@example.com>\r\n'
reply: b'250 OK\r\n'
send: 'rcpt TO:<anything@sl.local>\r\n'
reply: b'250 OK\r\n'
send: 'data\r\n'
reply: b'354 End data with <CR><LF>.<CR><LF>\r\n'
data: (354, b'End data with <CR><LF>.<CR><LF>')
send: b'From: somebody@example.com\r\nTo: anything@sl.local\r\nSubject: O3 rejection test\r\nMessage-ID: <o3-test@example.com>\r\n\r\nThis message should be rejected because public_domain is empty.\r\n.\r\n'
reply: b'550 SL E515 Email not exist\r\n'
reply: retcode (550); Msg: b'SL E515 Email not exist'
data: (550, b'SL E515 Email not exist')
send: 'quit\r\n'
reply: b'221 Bye\r\n'
```

### The SMTP status returned to the sender

```text
RCPT TO:<anything@sl.local>  ->  250 OK
DATA (end-of-message)        ->  550 SL E515 Email not exist
```

The recipient is accepted at `RCPT` time, but after the message body is transmitted the handler
evaluates it and returns the final reply **`550 SL E515 Email not exist`** to the sender.

### The verbatim rejection log lines

The email handler emitted the following (captured from its stdout; the per-email message-id is
`0a129f12-4e28-4ce0-9518-931a6103fdfd`):

```text
... - SL - INFO  - 14049 - "/workspace/email_handler.py:2343" - _handle() - 0a129f12-... - New message, mail from somebody@example.com, rctp tos ['anything@sl.local'] 
... - SL - DEBUG - 14049 - "/workspace/email_handler.py:2202" - handle() - 0a129f12-... - Forward phase somebody@example.com(somebody@example.com) -> anything@sl.local
... - SL - DEBUG - 14049 - "/workspace/email_handler.py:545"  - handle_forward() - 0a129f12-... - alias anything@sl.local not exist. Try to see if it can be created on the fly
... - SL - INFO  - 14049 - "/workspace/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - 0a129f12-... - Cannot auto-create custom domain alias for anything@sl.local because there's no custom domain for sl.local
... - SL - INFO  - 14049 - "/workspace/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - 0a129f12-... - Cannot auto-create anything@sl.local since it has no directory separator
... - SL - DEBUG - 14049 - "/workspace/email_handler.py:551"  - handle_forward() - 0a129f12-... - alias anything@sl.local cannot be created on-the-fly, return 550
... - SL - INFO  - 14049 - "/workspace/email_handler.py:2367" - _handle() - 0a129f12-... - Finish mail_from somebody@example.com, rcpt_tos ['anything@sl.local'], takes 0.14508891105651855 seconds with return code '550 SL E515 Email not exist'<<===
```

> The `rctp tos` spelling in the first line is verbatim from the source (`email_handler.py:2344`).
> The two `app/alias_utils.py` lines (104, 165) are the handler explaining *why* the on-the-fly
> creation failed: there is no custom domain for `sl.local`, and the local-part has no directory
> separator.

### SPF-override caveat — why the message is injected *without* SpamAssassin headers

After `handle()` returns a `5xx`, `MailHandler._handle` will **silently replace it with `E216`
("250 SL E216 Handled spf policy")** *iff* the message carries SpamAssassin/milter headers from
which `SpamdResult.extract_from_headers(msg)` (`email_handler.py:2356`) parses an SPF result of
`fail`/`soft_fail` (`email_handler.py:2356-2365`). A locally injected message with **no**
`X-Spamd-Result` header yields `spamd_result = None`, so the override does not fire and the
genuine `550` is preserved — which is exactly what we observe above.

To prove the override is real (and that omitting the headers is what surfaces the true `550`),
the **same** `anything@sl.local` injection was repeated **with** an SPF-fail header
(`X-Spamd-Result: ... R_SPF_FAIL ...`). The final reply flipped to `250`:

```text
# injection WITH X-Spamd-Result reporting R_SPF_FAIL
FINAL DATA reply -> 250 SL E216 Handled spf policy

# corresponding handler log
... - SL - INFO - 14049 - "/workspace/email_handler.py:2362" - _handle() - 480b017c-... - Replacing 5XX to 216 status because the return-path failed the spf check
... - SL - INFO - 14049 - "/workspace/email_handler.py:2367" - _handle() - 480b017c-... - Finish mail_from somebody@example.com, rcpt_tos ['anything@sl.local'], takes 0.0173... seconds with return code '250 SL E216 Handled spf policy'<<===
```

This matches the behavior exercised by the test suite
(`tests/test_email_handler.py::test_prevent_5xx_from_spf` and `::test_preserve_5xx_with_valid_spf`).
**Therefore the answer to Q3 — the genuine domain-not-configured rejection — is observed by
injecting without SpamAssassin headers, giving `550 SL E515 Email not exist`.**

### The rejection decision path

```mermaid
flowchart TD
    A["Inbound email to anything@sl.local"] --> B["MailHandler.handle_DATA -> _handle -> handle()"]
    B --> C{"Reverse-alias / reply?"}
    C -->|No| D["Forward branch -> handle_forward()  (email_handler.py:2202,2208)"]
    D --> E{"Alias.get_by(email) exists?  (email_handler.py:543)"}
    E -->|No| F["LOG.d: alias not exist, try auto-create  (email_handler.py:545)"]
    F --> G["try_auto_create()  (email_handler.py:549, alias_utils.py:202)"]
    G --> H{"catch-all CustomDomain OR directory?"}
    H -->|"no public_domain entry, no CustomDomain, no directory"| I["returns None  (alias_utils.py:224)"]
    I --> J["LOG.d: cannot be created on-the-fly, return 550  (email_handler.py:551)"]
    J --> K["return status.E515 = '550 SL E515 Email not exist'  (email_handler.py:555, status.py:51)"]
    K --> L{"SpamdResult present AND SPF fail/softfail?  (email_handler.py:2356-2361)"}
    L -->|"No (no X-Spamd-Result header)"| M["Final SMTP reply: 550 SL E515 Email not exist"]
    L -->|"Yes"| N["Replaced with E216 = '250 SL E216 Handled spf policy'  (email_handler.py:2365)"]
```

### Rationale — why `550 SL E515`

1. A recipient that is not a reverse-alias is routed into the **Forward** branch, which logs
   `Forward phase ...` (`email_handler.py:2202`) and calls `handle_forward()`
   (`email_handler.py:2208`).
2. In `handle_forward` (`email_handler.py:536`), `alias = Alias.get_by(email=alias_address)`
   (`email_handler.py:543`) returns `None` (no such alias), so it logs
   `alias ... not exist. Try to see if it can be created on the fly` (`email_handler.py:545-548`)
   and calls `try_auto_create(alias_address)` (`email_handler.py:549`).
3. `try_auto_create` (`app/alias_utils.py:202`) tries `try_auto_create_via_domain` then
   `try_auto_create_directory` and returns `None` (`app/alias_utils.py:220-224`): there is no
   catch-all/verified `CustomDomain` for `sl.local` and no directory. The underlying reason
   `sl.local` is unrecognized is that `is_valid_alias_address_domain` checks
   `SLDomain.get_by(domain)` then a verified `CustomDomain` (`app/email_utils.py:557-563`) —
   **both empty** because `public_domain` was never seeded.
4. With `alias` still `None`, `handle_forward` logs
   `alias ... cannot be created on-the-fly, return 550` (`email_handler.py:551`) and returns
   `[(False, status.E515)]` (`email_handler.py:555`), where
   `status.E515 = "550 SL E515 Email not exist"` (`app/email/status.py:51`).
5. Because the injected message has no SpamAssassin headers, the SPF override
   (`email_handler.py:2356-2365`) does not fire, so the handler's final reply to the sender is
   the genuine **`550 SL E515 Email not exist`** (`email_handler.py:2367`).

**Source map for Q3:** `init_app.py:39,73` · `app/models.py:3116,3119` ·
`email_handler.py:2202,2208,536,543,545-548,549,551,555,2356-2365,2367` ·
`app/alias_utils.py:202,220-224,104,165` · `app/email_utils.py:557-563` · `app/email/status.py:51`.

---


## Appendix A — Log-line format

Every `LOG.*` line above is produced through `app/log.py`, which installs a `coloredlogs`
formatter with this format string (`app/log.py:12-15`):

```text
%(asctime)s - %(name)s - %(levelname)s - %(process)d - "%(pathname)s:%(lineno)d" - %(funcName)s() - %(message_id)s - %(message)s
```

Reading a line such as:

```text
2026-06-26 21:31:16,618 - SL - DEBUG - 14049 - "/workspace/email_handler.py:551" - handle_forward() - 0a129f12-... - alias anything@sl.local cannot be created on-the-fly, return 550
```

- `SL` is the logger name (`LOG = _get_logger("SL")`, `app/log.py:79`) — it is *not* the module
  name, so all SimpleLogin lines read `SL` regardless of which file emitted them.
- `14049` is the OS process id; `"/workspace/email_handler.py:551"` is the emitting
  `pathname:lineno`; `handle_forward()` is the function; `0a129f12-...` is the per-email
  message-id set by `set_message_id()` for request/email tracing.
- The shortcuts map as `LOG.d`=DEBUG, `LOG.i`=INFO, `LOG.w`=WARNING, `LOG.e`=ERROR
  (`logging.Logger.exception`, which is why `LOG.e(e)` prints a full traceback)
  (`app/log.py:74-77`).
- `>>> init logging <<<` is a plain `print` emitted once when `app/log.py` is imported
  (`app/log.py:67`); it appears near the top of every service's startup output.

## Appendix B — Reproduction summary

| Q | DB state | Command(s) | Observed result |
|---|----------|-----------|-----------------|
| Q1 | empty / un-migrated (0 tables) | `python server.py`; `GET /`, `GET /auth/login`, then login `POST` | `GET`s OK; `POST` → HTTP 500 + `error/500.html`; stderr logs `sqlalchemy.exc.ProgrammingError` → `psycopg2.errors.UndefinedTable: relation "users" does not exist` |
| Q2 | migrated + `init_app.py` | `alembic upgrade head`; `python init_app.py`; `python server.py`; `python email_handler.py`; `python job_runner.py` | webapp `127.0.0.1:7777` (`/health`→`success`); email handler `0.0.0.0:20381` (SMTP `220`); job runner polling (no port) |
| Q3 | migrated, `init_app.py` skipped (`public_domain` empty) | `alembic upgrade head`; `python email_handler.py`; inject mail to `anything@sl.local` (no SpamAssassin headers) | final SMTP reply `550 SL E515 Email not exist` |

**Closing note.** This document is the only artifact produced; **no existing source file was
modified, created, or deleted**. The local `.env` (copied from `example.env`) and the small
throwaway `smtplib` injection scripts used to elicit the behavior are temporary observation aids
and are **not** committed to the repository.

