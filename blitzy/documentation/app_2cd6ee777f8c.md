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

> **Every block presented as evidence is captured verbatim from a real run** of the stack — the
> literal, unedited stdout/stderr, traceback, SMTP transcript, and `LOG.*` lines as actually
> emitted, complete with the timestamps, PIDs, and per-email message-ids of that run. These are
> the `console`/`text` fenced blocks throughout Q1–Q3 and the appendices. A small number of
> blocks are **illustrative rather than captured**, and are labeled as such: the documented
> startup-order outline (§0.1), the source-derived log-format string (Appendix A), and the
> Mermaid decision-path diagram (Q3). Reading the source tells us *what to expect and why*; the
> captured artifacts are the *evidence*. Each answer ends with a code-grounded rationale citing
> the responsible source `file:line`.

---

## 0. Runtime environment & methodology

All three experiments were reproduced inside the project's documented Docker runtime
(`andrewparkscaleai/coding-agent:simple-login__app__2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`,
pulled from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` and built on
`python:3.10` per `Dockerfile`). The observed runtime is:

| Component | Observed value |
|-----------|----------------|
| Python | `3.10.18` (project venv at `/app/venv`) |
| PostgreSQL | `15.13` on `localhost:5432`, database `simplelogin`, role `myuser` |
| Flask / Werkzeug | `1.1.2` / `1.0.1` |
| SQLAlchemy / psycopg2-binary | `1.3.24` / `2.9.3` |
| aiosmtpd | `1.4.2` |
| alembic | `1.4.3` |
| gunicorn | `20.0.4` |

These versions are not floating: they are the exact releases pinned in `poetry.lock` and
installed by Poetry into the project venv (`pyproject.toml` declares `python = "^3.10"`), so the
observed behavior is deterministic across rebuilds of the image.

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
bring-up order is (illustrative outline, not captured runtime output):

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
$ export PGURI=postgresql://myuser:mypassword@localhost:5432/simplelogin
$ echo 'drop schema public cascade; create schema public;' | psql "$PGURI"
DROP SCHEMA
CREATE SCHEMA

$ psql "$PGURI" -c "\dt"
Did not find any relations.
```

The `\dt` output `Did not find any relations.` confirms the database has **0 tables** — it
exists but is un-migrated.

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
WARNING: Use a temp directory for GNUPGHOME /tmp/zmmqmeixtjtzvalmegkr
Upload files to local dir
>>> init logging <<<
2026-06-26 22:26:27,156 - SL - DEBUG - 14835 - "/workspace/app/utils.py:17" - <module>() -  - load words file: /workspace/local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/doknteczerehiiglefhv
Upload files to local dir
>>> init logging <<<
2026-06-26 22:26:30,141 - SL - DEBUG - 14852 - "/workspace/app/utils.py:17" - <module>() -  - load words file: /workspace/local_data/test_words.txt
```

> Two notes on this banner. **(1)** The import preamble appears **twice** because
> `app.run(debug=True, …)` enables the Werkzeug **reloader**: the parent process (PID `14835`)
> prints the `* Serving Flask app … / * Debug mode: on` banner, then re-execs a reloaded worker
> (PID `14852`) that re-imports the app and is the process that actually serves requests — which
> is why the request log below shows PID `14852`. **(2)** The familiar Werkzeug
> `* Running on http://127.0.0.1:7777/` access-banner line is **absent** because SimpleLogin
> disables the `werkzeug` logger at import (`app/log.py:70-71`, `log.disabled = True`); the
> `* Serving Flask app … / * Debug mode: on` lines are printed by Flask's `run()` itself, so they
> still appear. Readiness is confirmed by `GET /health` (below).

Then drive the login flow in a browser (reproduced here with `requests`, preserving the
session cookie + CSRF token):

```console
# 2) GET /  -> anonymous user is redirected to /auth/login (no DB query). Full `curl -i` output:
$ curl -i http://localhost:7777/
HTTP/1.0 302 FOUND
Content-Type: text/html; charset=utf-8
Content-Length: 229
Location: http://localhost:7777/auth/login
Vary: Cookie
Set-Cookie: slapp=eyJfZnJlc2giOmZhbHNlLCJfcGVybWFuZW50Ijp0cnVlfQ.aj78mA.T7W8tCYgvfp2anVJnaUyvvMcLPo; Expires=Fri, 03-Jul-2026 22:26:32 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Fri, 26 Jun 2026 22:26:32 GMT

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<title>Redirecting...</title>
<h1>Redirecting...</h1>
<p>You should be redirected automatically to target URL: <a href="/auth/login">/auth/login</a>.  If not click the link.

# 3) GET /auth/login  -> the login page RENDERS CLEANLY (HTTP 200) against the empty DB
$ curl -s -o /dev/null -w "%{http_code}\n" http://localhost:7777/auth/login
200

# 4) Drive the full login flow with `requests` (CSRF-aware); the POST submits the form
GET / -> 302; Location: http://localhost:7777/auth/login
GET /auth/login -> 200; csrf_token present: True
POST /auth/login -> 500
```

The `POST` returns **HTTP 500**, and the response body is SimpleLogin's **custom `error/500.html`
page** — *not* the Werkzeug interactive debugger. These are the first lines of the captured
response body (verbatim — the project's themed HTML shell):

```html
<!DOCTYPE html>
<html lang="en"
      dir="ltr"
      data-theme="">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport"
          content="width=device-width, user-scalable=no, initial-scale=1.0, maximum-scale=1.0, minimum-scale=1.0" />
    <meta http-equiv="X-UA-Compatible" content="ie=edge" />
    <meta http-equiv="Content-Language" content="en" />
    <meta name="msapplication-TileColor" content="#2d89ef" />
    <meta name="theme-color" content="#4188c9" />
```

Further down, the same captured body carries the verbatim `<title>` block (rendering as
`| SimpleLogin`) and a `Server error` heading — the markers of the branded `error/500.html`
template, which a Werkzeug debugger page would not have:

```html
    <title>
      
      | SimpleLogin
    </title>
```

### Where the error surfaces — render (`GET`) vs. submit (`POST`)

The server's own request log settles this unambiguously: every `GET` (`/` and `/auth/login`)
completes with `302`/`200` (the page renders with no `users` query), and the error only fires on
the `POST`. The following is the verbatim `after_request` log block (`server.py:284`), captured
from the `python server.py` run — `GET /` and `GET /auth/login` appear twice each because both a
`curl` probe and the `requests` driver issued them before the `POST`:

```text
2026-06-26 22:26:32,115 - SL - DEBUG - 14852 - "/workspace/server.py:284" - after_request() -  - 127.0.0.1 GET / ImmutableMultiDict([]) 302, takes 0.0007169246673583984
2026-06-26 22:26:32,269 - SL - DEBUG - 14852 - "/workspace/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.1458439826965332
2026-06-26 22:26:32,373 - SL - DEBUG - 14852 - "/workspace/server.py:284" - after_request() -  - 127.0.0.1 GET / ImmutableMultiDict([]) 302, takes 0.0007653236389160156
2026-06-26 22:26:32,410 - SL - DEBUG - 14852 - "/workspace/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.03385472297668457
2026-06-26 22:26:32,421 - SL - ERROR - 14852 - "/workspace/server.py:390" - error_handler() -  - (psycopg2.errors.UndefinedTable) relation "users" does not exist
2026-06-26 22:26:32,426 - SL - DEBUG - 14852 - "/workspace/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 500, takes 0.013346195220947266
```

> The full multi-line `ProgrammingError` traceback (shown in the next section) is emitted by
> `error_handler()` **between** the `ERROR` line (timestamped `22:26:32,421`) and the final
> `after_request` line for the `POST` (timestamped `22:26:32,426`, status `500`); it is broken out
> below only for readability.

**Conclusion (observed):** the missing-table error is triggered by the login **`POST`
(submission)**, not by opening/rendering the page. The `GET` render path performs no query
against `users`.

### The full, verbatim Python traceback

Because `server.py`'s global handler `@app.errorhandler(Exception)` (`server.py:388`) catches
the exception, calls `LOG.e(e)` (`server.py:390`), and renders the 500 page, the **complete
traceback is written to the server's stderr** (`LOG.e` is an alias of `logging.Logger.exception`,
`app/log.py:77`). This is the exact block emitted (captured from `python server.py`, unedited):

```text
2026-06-26 22:26:32,421 - SL - ERROR - 14852 - "/workspace/server.py:390" - error_handler() -  - (psycopg2.errors.UndefinedTable) relation "users" does not exist
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
`psycopg2.errors.UndefinedTable: relation "users" does not exist`**. The offending statement is
the `SELECT … FROM users WHERE users.email = %(email_1)s LIMIT %(param_1)s` shown in full in the
`[SQL: …]` section of the traceback above, bound with parameters
`{'email_1': 'john@wick.com', 'param_1': 1}`.

### Rationale — why this happens

- **Why the server starts at all against an empty DB.** `app/db.py` builds the engine and opens
  a connection **at import time** —
  `engine = create_engine(config.DB_URI, connect_args={"application_name": config.DB_CONN_NAME})`
  (`app/db.py:9-11`) and `connection = engine.connect()` (`app/db.py:12`), then
  `Session = scoped_session(sessionmaker(bind=connection))` (`app/db.py:14`). Connecting requires
  the *database* to exist, but it does **not** require any
  *tables*. That is why `python server.py` boots cleanly and `/health` works even with zero
  tables — the schema is only touched on the first ORM query.
- **Why `GET` is fine but `POST` fails.** `GET /` → `index()` reads
  `current_user.is_authenticated` for an anonymous user (no query) and redirects to
  `auth.login` (`server.py:251-255`). `GET /auth/login` renders `auth/login.html` and does
  **not** query `users` (`app/auth/views/login.py:25-82`). Only on submit does
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
The initialization seeds the SL domain table; running `init_app.py` emits (verbatim tail of
`python init_app.py`, captured this run as PID 15729):

```text
2026-06-26 22:58:59,728 - SL - DEBUG - 15729 - "/workspace/app/utils.py:17" - <module>() -  - load words file: /workspace/local_data/test_words.txt
2026-06-26 22:59:01,812 - SL - DEBUG - 15729 - "/workspace/init_app.py:36" - load_pgp_public_keys() -  - Finish load_pgp_public_keys
2026-06-26 22:59:01,814 - SL - INFO - 15729 - "/workspace/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain
```

Confirm the seeded domain (`DB_URI` is the `example.env` value
`postgresql://myuser:mypassword@localhost:5432/simplelogin`, `example.env:75`):

```console
$ psql "$DB_URI" -c "SELECT id, domain, use_as_reverse_alias, premium_only FROM public_domain ORDER BY id;"
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

Startup banner (verbatim — full captured `python server.py` stdout/stderr, parent PID 15737):

```text
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/xjrofjlismyvvbdpgizh
Upload files to local dir
>>> init logging <<<
2026-06-26 22:59:03,126 - SL - DEBUG - 15737 - "/workspace/app/utils.py:17" - <module>() -  - load words file: /workspace/local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/wjsemelozszobqvvyghc
Upload files to local dir
>>> init logging <<<
2026-06-26 22:59:06,188 - SL - DEBUG - 15753 - "/workspace/app/utils.py:17" - <module>() -  - load words file: /workspace/local_data/test_words.txt
```

The preamble appears **twice** because `debug=True` enables the Werkzeug auto-reloader, which
forks a child worker (PID 15753 here) that re-imports the app. Note that the usual Werkzeug
`* Running on http://127.0.0.1:7777` line is **not** printed: SimpleLogin disables the
`werkzeug` logger at import (`app/log.py:70-71`), so readiness is signalled by the
`* Serving Flask app` / `* Debug mode: on` lines plus a successful `GET /health`.

Port-binding proof (the `/health` body is the strongest proof the webapp is accepting
connections):

```console
$ curl -s http://localhost:7777/health ; echo " [HTTP $(curl -s -o /dev/null -w '%{http_code}' http://localhost:7777/health)]"
success [HTTP 200]
```

The `/health` route returns `"success", 200` (`server.py:213-215`); the bind to
`127.0.0.1:7777` comes from `app.run(debug=True, port=7777)` (`server.py:588`). The production
form is `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2`. (The socket-level `LISTEN` proof for both
ports is in *Port-binding proof (`/proc/net/tcp`)* below.)

### 2b. Email handler — `python email_handler.py` (binds `0.0.0.0:20381`)

Readiness lines (verbatim — full captured `python email_handler.py` output, PID 15767):

```text
2026-06-26 22:59:10,530 - SL - DEBUG - 15767 - "/workspace/app/utils.py:17" - <module>() -  - load words file: /workspace/local_data/test_words.txt
2026-06-26 22:59:11,864 - SL - INFO - 15767 - "/workspace/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-06-26 22:59:11,866 - SL - DEBUG - 15767 - "/workspace/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

Port-binding proof (the SMTP `220` greeting is the strongest proof the controller is accepting
connections):

```console
$ /app/venv/bin/python -c "import smtplib; c=smtplib.SMTP(); print(c.connect('localhost',20381)); print(c.ehlo('probe.local')[0]); c.quit()"
(220, b'45fc3e9dd28a Python SMTP 1.4.2')
250
```

In `__main__`, argparse defaults the port to `20381` and logs `Listen for port 20381`
(`email_handler.py:2399,2403`); `main(port)` then creates
`Controller(MailHandler(), hostname="0.0.0.0", port=port)` (`email_handler.py:2383`), calls
`controller.start()` (`email_handler.py:2385`), logs `Start mail controller 0.0.0.0 20381`
(`email_handler.py:2386`), and idles in `while True: time.sleep(2)` (`email_handler.py:2392-2393`).
The `220 45fc3e9dd28a Python SMTP 1.4.2` greeting (where `45fc3e9dd28a` is the container
hostname) is emitted by the underlying `aiosmtpd` 1.4.2 `Controller`.

### 2c. Job runner — `python job_runner.py` (no port)

The job runner emits the standard initialization preamble and then goes **quiet** — it is
"ready" once it enters its polling loop and will only log again when it picks up a job:

```text
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/yohsdevrbzbuhxzadwdy
Upload files to local dir
>>> init logging <<<
2026-06-26 22:59:14,935 - SL - DEBUG - 15785 - "/workspace/app/utils.py:17" - <module>() -  - load words file: /workspace/local_data/test_words.txt
```

It binds no port; its `__main__` (`job_runner.py:329`) enters `while True:` (`job_runner.py:330`),
opens an app context, drains `get_jobs_to_run()`, and `time.sleep(10)` (`job_runner.py:347`).
(With no queued jobs it prints nothing further — that silence *is* the steady state.)

### Port-binding proof (`/proc/net/tcp`)

`ss`, `lsof`, and `netstat` are **not** installed in this image, so the listening sockets are
read directly from `/proc/net/tcp` (state `0A` = `LISTEN`) and decoded. With the webapp and the
email handler both up:

```console
$ /app/venv/bin/python - <<'PY'
import struct, socket
for line in open('/proc/net/tcp').read().splitlines()[1:]:
    f = line.split(); local, st = f[1], f[3]
    if st != '0A':  # 0A = LISTEN
        continue
    ip_hex, port_hex = local.split(':'); port = int(port_hex, 16)
    if port in (7777, 20381):
        ip = socket.inet_ntoa(struct.pack('<I', int(ip_hex, 16)))
        print(f'LISTEN {ip}:{port}')
PY
LISTEN 127.0.0.1:7777
LISTEN 0.0.0.0:20381
```

Raw corroboration from `/proc/net/tcp` (`1E61` = 7777, `4F9D` = 20381; column 4 `0A` = LISTEN;
local address `0100007F` = `127.0.0.1`, `00000000` = `0.0.0.0`):

```text
# /proc/net/tcp (state 0A=LISTEN); ports 1E61=7777, 4F9D=20381
   3: 00000000:4F9D 00000000:0000 0A 00000000:00000000 00:00000000 00000000     0        0 532325546 1 0000000000000000 100 0 0 10 0
   4: 0100007F:1E61 00000000:0000 0A 00000000:00000000 00:00000000 00000000     0        0 532284854 1 0000000000000000 100 0 0 10 0
```

This confirms the webapp bound `127.0.0.1:7777` and the email handler bound `0.0.0.0:20381`.

### All three processes running

```console
$ ps -eo pid,cmd | grep -E "[s]erver.py|[e]mail_handler.py|[j]ob_runner.py"
  15737 /app/venv/bin/python server.py
  15753 /app/venv/bin/python /workspace/server.py
  15767 /app/venv/bin/python email_handler.py
  15785 /app/venv/bin/python job_runner.py
```

(`15737` is the webapp parent and `15753` its Werkzeug auto-reloader child; `15767` is the email
handler and `15785` the job runner — four processes for the three services, because debug mode
doubles the webapp.)

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
$ psql "$DB_URI" -c "SELECT count(*) FROM public_domain;"
 count 
-------
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
reply: retcode (250); Msg: b'45fc3e9dd28a\nSIZE 33554432\n8BITMIME\nSMTPUTF8\nHELP'
send: 'mail FROM:<somebody@example.com>\r\n'
reply: b'250 OK\r\n'
reply: retcode (250); Msg: b'OK'
send: 'rcpt TO:<anything@sl.local>\r\n'
reply: b'250 OK\r\n'
reply: retcode (250); Msg: b'OK'
send: 'data\r\n'
reply: b'354 End data with <CR><LF>.<CR><LF>\r\n'
reply: retcode (354); Msg: b'End data with <CR><LF>.<CR><LF>'
data: (354, b'End data with <CR><LF>.<CR><LF>')
send: b'From: somebody@example.com\r\nTo: anything@sl.local\r\nSubject: O3 rejection test\r\nMessage-ID: <o3-test@example.com>\r\n\r\nbody\r\n.\r\n'
reply: b'550 SL E515 Email not exist\r\n'
reply: retcode (550); Msg: b'SL E515 Email not exist'
data: (550, b'SL E515 Email not exist')
send: 'quit\r\n'
reply: b'221 Bye\r\n'
reply: retcode (221); Msg: b'Bye'
```

### The SMTP status returned to the sender

The injection script also prints the two decisive replies it received (verbatim stdout):

```text
RCPT-REPLY: 250 b'OK'
DATA-FINAL-REPLY: 550 b'SL E515 Email not exist'
```

The recipient is accepted at `RCPT` time, but after the message body is transmitted the handler
evaluates it and returns the final reply **`550 SL E515 Email not exist`** to the sender.

### The verbatim rejection log lines

The complete, contiguous handler output for the injected message (PID 15965; the per-email
message-id `6e8fb0b3-1cf0-4d86-b9e3-6e8da181cd75` is set by `set_message_id()` and threads every
line). Nothing is elided:

```text
2026-06-26 23:05:31,432 - SL - DEBUG - 15965 - "/workspace/app/log.py:24" - set_message_id() -  - set message_id 6e8fb0b3-1cf0-4d86-b9e3-6e8da181cd75
2026-06-26 23:05:31,432 - SL - DEBUG - 15965 - "/workspace/email_handler.py:2342" - _handle() - 6e8fb0b3-1cf0-4d86-b9e3-6e8da181cd75 - ====>=====>====>====>====>====>====>====>
2026-06-26 23:05:31,432 - SL - INFO - 15965 - "/workspace/email_handler.py:2343" - _handle() - 6e8fb0b3-1cf0-4d86-b9e3-6e8da181cd75 - New message, mail from somebody@example.com, rctp tos ['anything@sl.local'] 
2026-06-26 23:05:31,433 - SL - INFO - 15965 - "/workspace/email_handler.py:1956" - handle() - 6e8fb0b3-1cf0-4d86-b9e3-6e8da181cd75 - Set CONTENT_TRANSFER_ENCODING
2026-06-26 23:05:31,433 - SL - DEBUG - 15965 - "/workspace/email_handler.py:1963" - handle() - 6e8fb0b3-1cf0-4d86-b9e3-6e8da181cd75 - Cannot parse Postfix queue ID from None None
2026-06-26 23:05:31,501 - SL - DEBUG - 15965 - "/workspace/email_handler.py:1980" - handle() - 6e8fb0b3-1cf0-4d86-b9e3-6e8da181cd75 - ==>> Handle mail_from:somebody@example.com, rcpt_tos:['anything@sl.local'], header_from:somebody@example.com, header_to:anything@sl.local, cc:None, reply-to:None, message_id:<o3-test@example.com>, client_ip:None, headers:[('From', 'somebody@example.com'), ('To', 'anything@sl.local'), ('Subject', 'O3 rejection test'), ('Message-ID', '<o3-test@example.com>'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-06-26 23:05:31,506 - SL - DEBUG - 15965 - "/workspace/email_handler.py:2202" - handle() - 6e8fb0b3-1cf0-4d86-b9e3-6e8da181cd75 - Forward phase somebody@example.com(somebody@example.com) -> anything@sl.local
2026-06-26 23:05:31,516 - SL - DEBUG - 15965 - "/workspace/email_handler.py:545" - handle_forward() - 6e8fb0b3-1cf0-4d86-b9e3-6e8da181cd75 - alias anything@sl.local not exist. Try to see if it can be created on the fly
2026-06-26 23:05:31,572 - SL - INFO - 15965 - "/workspace/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - 6e8fb0b3-1cf0-4d86-b9e3-6e8da181cd75 - Cannot auto-create custom domain alias for anything@sl.local because there's no custom domain for sl.local
2026-06-26 23:05:31,572 - SL - INFO - 15965 - "/workspace/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - 6e8fb0b3-1cf0-4d86-b9e3-6e8da181cd75 - Cannot auto-create anything@sl.local since it has no directory separator
2026-06-26 23:05:31,572 - SL - DEBUG - 15965 - "/workspace/email_handler.py:551" - handle_forward() - 6e8fb0b3-1cf0-4d86-b9e3-6e8da181cd75 - alias anything@sl.local cannot be created on-the-fly, return 550
2026-06-26 23:05:31,573 - SL - INFO - 15965 - "/workspace/email_handler.py:2367" - _handle() - 6e8fb0b3-1cf0-4d86-b9e3-6e8da181cd75 - Finish mail_from somebody@example.com, rcpt_tos ['anything@sl.local'], takes 0.14161252975463867 seconds with return code '550 SL E515 Email not exist'<<===
```

The five lines that constitute the rejection are: **New message** (`email_handler.py:2343`,
`LOG.i`), **Forward phase** (`email_handler.py:2202`, `LOG.d`), **alias … not exist. Try to see
if it can be created on the fly** (`email_handler.py:545`, `LOG.d`), **alias … cannot be created
on-the-fly, return 550** (`email_handler.py:551`, `LOG.d`), and **Finish … with return code
'550 SL E515 Email not exist'<<===** (`email_handler.py:2367`, `LOG.i`).

> The `rctp tos` spelling in the `New message` line is verbatim from the source
> (`email_handler.py:2344`). The two `app/alias_utils.py` lines (104, 165) are the handler
> explaining *why* the on-the-fly creation failed: there is no custom domain for `sl.local`, and
> the local-part has no directory separator.

### SPF-override caveat — why the message is injected *without* SpamAssassin headers

After `handle()` returns a `5xx`, `MailHandler._handle` will **silently replace it with `E216`
("250 SL E216 Handled spf policy")** *iff* the message carries SpamAssassin/milter headers from
which `SpamdResult.extract_from_headers(msg)` (`email_handler.py:2356`) parses an SPF result of
`fail`/`soft_fail` (`email_handler.py:2356-2365`). A locally injected message with **no**
`X-Spamd-Result` header yields `spamd_result = None`, so the override does not fire and the
genuine `550` is preserved — which is exactly what we observe above.

This override is not hypothetical — the project's own test suite encodes it:
`tests/test_email_handler.py::test_prevent_5xx_from_spf` asserts that a `5xx` is rewritten to
`250 SL E216 Handled spf policy` **only** when the message carries an SPF-fail `X-Spamd-Result`
header, while `::test_preserve_5xx_with_valid_spf` asserts the `5xx` is preserved otherwise. The
rewrite is performed by `_handle` (`email_handler.py:2362,2365`). Because the local injection
above carries no such header, the override does not fire and the genuine
**`550 SL E515 Email not exist`** is what the sender receives — which is the answer to Q3. (No
extra SPF-fail experiment is reproduced here; the negative case above, plus these two tests, is
sufficient.)

### The rejection decision path

The following diagram is an **illustrative** summary of the path (not captured output); each node
cites the responsible source `file:line`:

```mermaid
flowchart TD
    A["Inbound email to anything@sl.local"] --> B["MailHandler.handle_DATA -> _handle -> handle()"]
    B --> C{"Reverse-alias / reply?"}
    C -->|No| D["Forward branch -> handle_forward()  (email_handler.py:2202,2208)"]
    D --> E{"Alias.get_by(email) exists?  (email_handler.py:543)"}
    E -->|No| F["LOG.d: alias not exist, try auto-create  (email_handler.py:545)"]
    F --> G["try_auto_create()  (email_handler.py:549, app/alias_utils.py:202)"]
    G --> H{"catch-all CustomDomain OR directory?"}
    H -->|"no public_domain entry, no CustomDomain, no directory"| I["returns None  (app/alias_utils.py:220-224)"]
    I --> J["LOG.d: cannot be created on-the-fly, return 550  (email_handler.py:551)"]
    J --> K["return status.E515 = '550 SL E515 Email not exist'  (email_handler.py:555, app/email/status.py:51)"]
    K --> L{"SpamdResult present AND SPF fail/softfail?  (email_handler.py:2356-2361)"}
    L -->|"No (no X-Spamd-Result header)"| M["Final SMTP reply: 550 SL E515 Email not exist"]
    L -->|"Yes"| N["Replaced with E216 = '250 SL E216 Handled spf policy'  (email_handler.py:2365)"]
```

### Rationale — why `550 SL E515`

1. A recipient that is not a reverse-alias is routed into the **Forward** branch, which logs
   `Forward phase somebody@example.com(somebody@example.com) -> anything@sl.local`
   (`email_handler.py:2202`) and calls `handle_forward()` (`email_handler.py:2208`).
2. In `handle_forward` (`email_handler.py:536`), `alias = Alias.get_by(email=alias_address)`
   (`email_handler.py:543`) returns `None` (no such alias), so it logs
   `alias anything@sl.local not exist. Try to see if it can be created on the fly`
   (`email_handler.py:545-548`) and calls `try_auto_create(alias_address)`
   (`email_handler.py:549`).
3. `try_auto_create` (`app/alias_utils.py:202`) calls `try_auto_create_via_domain`
   (`app/alias_utils.py:220`) then `try_auto_create_directory` (`app/alias_utils.py:222`); with
   both returning `None`, `try_auto_create` returns `None` (`app/alias_utils.py:220-224`). The
   first logs `Cannot auto-create custom domain alias for anything@sl.local because there's no
   custom domain for sl.local` (`app/alias_utils.py:104`) — there is no catch-all/verified
   `CustomDomain` for `sl.local`; the second logs `Cannot auto-create anything@sl.local since it
   has no directory separator` (`app/alias_utils.py:165`). (`is_valid_alias_address_domain` —
   which checks `SLDomain.get_by(domain)` then a verified `CustomDomain`,
   `app/email_utils.py:557-563` — is **supporting domain-validity context**: it confirms
   `sl.local` is unconfigured once `public_domain` is empty, but it is **not** called on this
   forward-rejection path.)
4. With `alias` still `None`, `handle_forward` logs
   `alias anything@sl.local cannot be created on-the-fly, return 550` (`email_handler.py:551`) and returns
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
2026-06-26 23:05:31,572 - SL - DEBUG - 15965 - "/workspace/email_handler.py:551" - handle_forward() - 6e8fb0b3-1cf0-4d86-b9e3-6e8da181cd75 - alias anything@sl.local cannot be created on-the-fly, return 550
```

- `SL` is the logger name (`LOG = _get_logger("SL")`, `app/log.py:79`) — it is *not* the module
  name, so all SimpleLogin lines read `SL` regardless of which file emitted them.
- `15965` is the OS process id; `"/workspace/email_handler.py:551"` is the emitting
  `pathname:lineno`; `handle_forward()` is the function; `6e8fb0b3-1cf0-4d86-b9e3-6e8da181cd75`
  is the per-email message-id set by `set_message_id()` for request/email tracing.
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

