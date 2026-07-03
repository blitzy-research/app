# SimpleLogin — Runtime-Behavior Q&A (developer-environment bring-up)

This document answers three runtime-behavior questions about the SimpleLogin stack. Every answer was produced **by actually running the code first** and then quoting the captured output verbatim. Each behavioral claim is paired with exactly one adjacent evidence block (the command that was run plus its captured output). Any statement not directly observed is prefixed with `(inferred)`. Exact identifiers, strings, status codes, and file paths are cited with their `file:line` reference.

The three questions are:

- **Q1 — Empty-database failure mode.** With PostgreSQL running and an empty database (no tables, migrations not run), what exact error is hit when the server is started with `python server.py` and the login page is then opened in a browser?
- **Q2 — Required runtime services.** After migrations *and* the initialization script `init_app.py` have been run, which Python services must be running for the system to work?
- **Q3 — Skipped-initialization rejection.** If migrations are run but `init_app.py` is skipped — so the SLDomain table should be empty (no email domains configured) — what SMTP status code does the email handler return, and what does it log, when a message addressed to any `@sl.local` address arrives?

---

## 1. Introduction — canonical environment, exact commands, and configuration actually used

### 1.1 Runtime versions and setup state actually observed

All commands were run inside the provided Docker image using the project's in-project virtual environment `./.venv` (Python 3.10 via pyenv), which is the canonical interpreter a normal user runs this project with in this image. The dependencies were already installed into `./.venv`, and the `.env` was already created from `example.env`, by the environment provisioning; the canonical commands a normal user runs to reach this same state (`poetry sync` and `cp example.env .env`) are shown with observed evidence in §1.1.1 below.

Command:

```
$ ./.venv/bin/python --version
$ docker exec sl-db psql --version
$ ./.venv/bin/python - <<'PY'
import importlib.metadata as m
for p in ["SQLAlchemy","aiosmtpd","psycopg2-binary","Flask","python-dotenv","Flask-Migrate","alembic"]:
    print(f"{p}=={m.version(p)}")
PY
```

Observed output:

```
Python 3.10.20
psql (PostgreSQL) 13.23 (Debian 13.23-1.pgdg13+1)
SQLAlchemy==1.3.24
aiosmtpd==1.4.2
psycopg2-binary==2.9.3
Flask==1.1.2
python-dotenv==0.14.0
Flask-Migrate==2.5.3
alembic==1.4.3
```

These match the manifest pins: Python `^3.10` [pyproject.toml:61] (base image `FROM python:3.10` [Dockerfile:8]), `SQLAlchemy = "1.3.24"` [pyproject.toml:116], `flask = "^1.1.2"` [pyproject.toml:62], `psycopg2-binary = "^2.9.3"` [pyproject.toml:71], `python-dotenv = "^0.14.0"` [pyproject.toml:68], `Flask-Migrate = "^2.5.3"` [pyproject.toml:77]. Note the installed **aiosmtpd is `1.4.2`** whereas the manifest pins `aiosmtpd = "^1.2"` [pyproject.toml:87]; the observed value `1.4.2` is reported (it also appears in the SMTP banner in Q2).

#### 1.1.1 Canonical setup commands and the observed setup state

The canonical developer setup, per the project's own docs, is two steps: install dependencies with **`poetry sync`** [CONTRIBUTING.md:31] (the project targets "Python 3.10 and poetry to manage dependencies" [CONTRIBUTING.md:23]) into an in-project virtualenv activated with `source .venv/bin/activate` [CONTRIBUTING.md:251], then create the local settings file with **`cp example.env .env`** [CONTRIBUTING.md:88]. In this image both steps were already performed by the environment provisioning (dependencies pre-installed into `./.venv`; `.env` pre-created from `example.env`), so the observed setup state was *verified* rather than re-run, as follows.

Claim: the dependency toolchain is Poetry 1.8.5 and the pre-provisioned `./.venv` already satisfies every runtime import.

```
$ /root/.local/bin/poetry --version
Poetry (version 1.8.5)
$ test -d .venv && echo ".venv EXISTS (in-project virtualenv, gitignored)"
.venv EXISTS (in-project virtualenv, gitignored)
$ ./.venv/bin/python -c "import flask, sqlalchemy, aiosmtpd, psycopg2, dotenv, alembic; print('all key imports OK from ./.venv')"
all key imports OK from ./.venv
```

Claim: `.env` is `example.env` with exactly one intended deviation — the database port — proving it was created from `example.env` (`cp example.env .env`).

```
$ ls -l .env
-rw-r--r-- 1 root root 5790 Jul  2 22:47 .env
$ diff example.env .env
75c75
< DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin
---
> DB_URI=postgresql://myuser:mypassword@localhost:15432/simplelogin
```

The `diff` shows `.env` is byte-for-byte identical to `example.env` except line 75, where `DB_URI`'s port is `15432` instead of `example.env`'s `5432` [example.env:75] — because in this image the dockerized PostgreSQL 13 publishes its container port `5432` on host port `15432`. Every other line (including `EMAIL_DOMAIN=sl.local` [example.env:22]) is the shipped canonical value, so this is the default/canonical configuration a normal user runs.

### 1.2 Exact invocation commands

- Interpreter: `./.venv/bin/python` (Python 3.10.20). In the shell below, `./.venv/bin` is placed first on `PATH` so that the literal command **`python server.py`** (as the user phrased it) resolves to this interpreter.
- Migrations: `alembic upgrade head` (alembic `script_location = migrations` [alembic.ini:5]; the URL is set from the app config — `from app.config import DB_URI` [migrations/env.py:27] then `config.set_main_option('sqlalchemy.url', DB_URI)` [migrations/env.py:35], so it uses the `DB_URI` from `.env`).
- Initialization/seed: `python init_app.py`.
- Web app: `python server.py` (dev server on port 7777).
- Email handler: `python email_handler.py` (SMTP on port 20381).

### 1.3 Configuration actually used (canonical `.env` from `example.env`)

Config is loaded from `.env` via python-dotenv — `from dotenv import load_dotenv` [app/config.py:9], read from a `CONFIG`-pointed file if set [app/config.py:65] else `load_dotenv()` [app/config.py:71]. The app **hard-requires** these env vars at import (it refuses to start without them):

- `URL = os.environ["URL"]` [app/config.py:79]
- `EMAIL_DOMAIN = os.environ["EMAIL_DOMAIN"].lower()` [app/config.py:92]
- `SUPPORT_EMAIL = os.environ["SUPPORT_EMAIL"]` [app/config.py:93]
- `DB_URI = os.environ["DB_URI"]` [app/config.py:192]
- `FLASK_SECRET = os.environ["FLASK_SECRET"]` [app/config.py:196], which additionally does `if not FLASK_SECRET: raise RuntimeError("FLASK_SECRET is empty. Please define it.")` [app/config.py:197-198].

Command and observed `.env` values (derived from `example.env`):

```
$ grep -E '^(URL|EMAIL_DOMAIN|SUPPORT_EMAIL|DB_URI|FLASK_SECRET)=' .env
URL=http://localhost:7777
EMAIL_DOMAIN=sl.local
SUPPORT_EMAIL=support@sl.local
DB_URI=postgresql://myuser:mypassword@localhost:15432/simplelogin
FLASK_SECRET=secret
```

The canonical `example.env` ships `EMAIL_DOMAIN=sl.local` [example.env:22] (also `URL` [example.env:6], `SUPPORT_EMAIL` [example.env:40], `DB_URI` [example.env:75], `FLASK_SECRET` [example.env:77]). As shown by the `diff` in §1.1.1, the only deviation from `example.env` is the `DB_URI` **port `15432`** instead of `example.env`'s `5432` — because in this image the dockerized PostgreSQL 13 publishes its `5432` on host port `15432` (the shipped helper `scripts/reset_local_db.sh` sets the same value: `export DB_URI=postgresql://myuser:mypassword@localhost:15432/simplelogin` [scripts/reset_local_db.sh:3]). `EMAIL_DOMAIN=sl.local` is the `@sl.local` domain referenced in Q3, and it is folded into `ALIAS_DOMAINS`: when `ALIAS_DOMAINS` is not in the env, `ALIAS_DOMAINS = OTHER_ALIAS_DOMAINS + [EMAIL_DOMAIN]` [app/config.py:160]. Verified at runtime (the first five lines are the `app.config` import banner emitted by `print()` on every invocation — the `GNUPGHOME` temp path is randomized per run):

```
$ ./.venv/bin/python -c "from app import config; print('ALIAS_DOMAINS =', config.ALIAS_DOMAINS); print('EMAIL_DOMAIN =', config.EMAIL_DOMAIN)"
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/hrlufvotqpgewaefztpf
Upload files to local dir
ALIAS_DOMAINS = ['sl.local']
EMAIL_DOMAIN = sl.local
```

### 1.4 Logging is the evidence channel

Every service logs to **stdout** through one logger named `"SL"` — `LOG = _get_logger("SL")` [app/log.py:79] — using a format that embeds `"%(pathname)s:%(lineno)d"` [app/log.py:14]; werkzeug/Flask request logs are disabled (`log = logging.getLogger("werkzeug")` / `log.disabled = True` [app/log.py:70-71]); importing the logging module prints `>>> init logging <<<` [app/log.py:67]. Crucially, `logging.Logger.e = logging.Logger.exception` [app/log.py:77], so a `LOG.e(e)` call emits a **full traceback** to stdout. Therefore stdout (merged with stderr) was captured to a log file for every process started here.

### 1.5 The three distinct database states

The three questions require three *different* database states, so the database was torn down and re-provisioned between them (mirroring `scripts/reset_local_db.sh`, which does `drop schema public cascade; create schema public;` then `alembic upgrade head`):

- **Q1 — empty / unmigrated:** database exists but has **no tables** (migrations deliberately not run).
- **Q2 — migrated + initialized:** `alembic upgrade head` **and** `python init_app.py` both run.
- **Q3 — migrated but NOT initialized:** `alembic upgrade head` run, but `init_app.py` (and `flask dummy-data`) **skipped**, leaving the SLDomain table empty.

Each scenario was **re-run at least twice** and reproduced the identical categorical result (error type / service readiness / SMTP code + log line), as noted per section (rule: confirm stability across ≥2 runs).

---

## 2. Q1 — Empty-database failure mode

### 2.1 Provision the empty database (no migrations) and verify it has no tables

The empty database was created exactly the way the project itself creates a fresh database — `dropdb` then `createdb` (the same pattern the docs use for the test DB, `dropdb test && createdb test` [CONTRIBUTING.md:73]) — and migrations were deliberately **not** run.

Command and output — drop and recreate the database (both commands print nothing and exit `0`):

```
$ docker exec sl-db dropdb -U myuser simplelogin; echo "dropdb exit=$?"
dropdb exit=0
$ docker exec sl-db createdb -U myuser -O myuser simplelogin; echo "createdb exit=$?"
createdb exit=0
```

Claim: the freshly created database has **zero** tables in the `public` schema.

```
$ docker exec sl-db psql -U myuser -d simplelogin -c "SELECT count(*) FROM information_schema.tables WHERE table_schema='public';"
 count 
-------
     0
(1 row)
```

Claim: `psql`'s `\dt` confirms there are no relations at all.

```
$ docker exec sl-db psql -U myuser -d simplelogin -c "\dt"
Did not find any relations.
```

### 2.2 `python server.py` starts successfully against the empty database and binds :7777

Claim: starting the server does **not** fail against an empty-but-existing database — it starts and becomes ready to serve.

Command and captured startup stdout. The output shows **two import passes with two PIDs** because werkzeug's reloader (enabled by `app.run(debug=True, port=7777)` [server.py:588]) first imports the app in the reloader parent (PID `77939`), then spawns the worker child (PID `77953`) that re-imports and serves; the randomized `GNUPGHOME` temp directory therefore differs between the two passes:

```
$ export PATH="$PWD/.venv/bin:$PATH"; export PYTHONUNBUFFERED=1
$ python server.py            # started in its own session; stdout+stderr captured to a log
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/evggydyrrwmtalvqxilv
Upload files to local dir
>>> init logging <<<
2026-07-03 00:38:42,937 - SL - DEBUG - 77939 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/app/utils.py:17" - <module>() -  - load words file: /tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/yvttnrifliakebzyevfu
Upload files to local dir
>>> init logging <<<
2026-07-03 00:38:44,918 - SL - DEBUG - 77953 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/app/utils.py:17" - <module>() -  - load words file: /tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/local_data/test_words.txt
```

Independent readiness / bind proof — the `/health` route returns `return "success", 200` [server.py:213-215]:

```
$ curl -s -i http://localhost:7777/health
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 7
Vary: Cookie
Set-Cookie: slapp=eyJfcGVybWFuZW50Ijp0cnVlfQ.akcElg.oCh4HCnI1KtMBWryNDCDf6wRvDs; Expires=Fri, 10-Jul-2026 00:38:46 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.20
Date: Fri, 03 Jul 2026 00:38:46 GMT

success
```

**Cause → effect:** the process reaches the query-serving state because `app/db.py` opens the database connection at **import** time — `engine = create_engine(config.DB_URI, connect_args={"application_name": config.DB_CONN_NAME})` [app/db.py:9-11] then `connection = engine.connect()` [app/db.py:12] — and connecting to an *existing but empty* database succeeds (no table is touched yet). The dev server is then launched from the `if __name__ == "__main__": local_main()` entrypoint [server.py:598-599] via `app.run(debug=True, port=7777)` [server.py:588]. The two PIDs above (`77939` parent, `77953` worker child) are werkzeug's reloader, which `debug=True` enables — observed directly in the duplicated startup banner.

### 2.3 Opening the login page: a plain GET renders with no error

Claim: opening the login page with a GET does **not** hit the error — it renders HTTP 200 without querying the database.

Command and output — `GET /` (the `index()` view [server.py:251]) redirects an anonymous visitor to the login page:

```
$ curl -s -i http://localhost:7777/
HTTP/1.0 302 FOUND
Content-Type: text/html; charset=utf-8
Content-Length: 229
Location: http://localhost:7777/auth/login
Vary: Cookie
Set-Cookie: slapp=eyJfZnJlc2giOmZhbHNlLCJfcGVybWFuZW50Ijp0cnVlfQ.akcElg.xrGsFLXRFx0cvEHXSLHYGvK9eWs; Expires=Fri, 10-Jul-2026 00:38:46 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.20
Date: Fri, 03 Jul 2026 00:38:46 GMT

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<title>Redirecting...</title>
<h1>Redirecting...</h1>
<p>You should be redirected automatically to target URL: <a href="/auth/login">/auth/login</a>.  If not click the link.
```

Command and output — `GET /auth/login` returns **HTTP 200** (response headers via `curl -D -`; the HTML body of `Content-Length: 379295` bytes is written to a file):

```
$ curl -s -c cookies -D - -o getlogin.body http://localhost:7777/auth/login
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 379295
Vary: Cookie
Set-Cookie: slapp=eyJfZnJlc2giOmZhbHNlLCJfcGVybWFuZW50Ijp0cnVlLCJjc3JmX3Rva2VuIjoiY2ExYWVmNDA4OWYxMTM0ZWZjNGNkZGQ5ZGRmNjZjMjdlMGQzNTE1YSJ9.akcElg.6TJHmV2oB21IKrD9UHPuKxJDMMA; Expires=Fri, 10-Jul-2026 00:38:46 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.20
Date: Fri, 03 Jul 2026 00:38:46 GMT
```

Claim: that HTTP 200 body is the actual rendered login page (a scoped `grep` of the saved body for the page title):

```
$ grep -oE '<title>|Login|\| SimpleLogin' getlogin.body | head -5
<title>
Login
| SimpleLogin
Login
Login
```

**Cause → effect:** the `auth.login` view only queries the `User` table inside the POST branch `if form.validate_on_submit():` [app/auth/views/login.py:40] where `user = User.get_by(email=email) or User.get_by(email=canonical_email)` [app/auth/views/login.py:43]; a plain GET simply renders `auth/login.html` [app/auth/views/login.py:74] and issues no query. So merely *displaying* the login page does not touch a missing table.

### 2.4 Submitting the login form (real POST) triggers the exact error → HTTP 500

To force the first real database query, the login form was submitted through the real HTTP path: the `csrf_token` hidden field was read from the `GET /auth/login` body captured in §2.3, then submitted together with `email` and `password` via `POST /auth/login`, reusing the same session cookie jar.

Command and the HTTP response headers the client receives (`curl -D -`, body written to a file):

```
$ CSRF=$(grep -oE 'name="csrf_token"[^>]*value="[^"]+"' getlogin.body | grep -oE 'value="[^"]+"' | head -1 | sed -E 's/value="([^"]+)"/\1/')
$ curl -s -b cookies -c cookies \
    --data-urlencode "email=test@example.com" \
    --data-urlencode "password=password123" \
    --data-urlencode "csrf_token=${CSRF}" \
    -D - -o post.body http://localhost:7777/auth/login
HTTP/1.0 500 INTERNAL SERVER ERROR
Content-Type: text/html; charset=utf-8
Content-Length: 5749
Vary: Cookie
Set-Cookie: slapp=eyJfZnJlc2giOmZhbHNlLCJfcGVybWFuZW50Ijp0cnVlLCJjc3JmX3Rva2VuIjoiY2ExYWVmNDA4OWYxMTM0ZWZjNGNkZGQ5ZGRmNjZjMjdlMGQzNTE1YSJ9.akcEpQ.GWE49bcxARHA__PJgECvWcSt5zs; Expires=Fri, 10-Jul-2026 00:39:01 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.20
Date: Fri, 03 Jul 2026 00:39:01 GMT
```

Claim: the `Content-Length: 5749` body is the rendered `error/500.html` template (not the werkzeug interactive debugger). The template's signature strings are present in the body:

```
$ grep -oE "Server error|Looks like we are having some server issues|We are notified and will look at this issue asap" post.body | sort | uniq -c
      1 Looks like we are having some server issues
      1 Server error
      1 We are notified and will look at this issue asap
```

Those strings come from `templates/error/500.html` (its `{% block error_name %}Server error{% endblock %}` heading and the "We are notified and will look at this issue asap!" body line).

### 2.5 The exact error: full Python traceback (verbatim), first failing relation = `users`

Claim: the exact error is a `sqlalchemy.exc.ProgrammingError` wrapping `psycopg2.errors.UndefinedTable`, with the message `relation "users" does not exist`; the **first failing relation is `users`**.

The **complete, unabridged** server stdout emitted for that `POST /auth/login` is reproduced below exactly as captured — the full `LOG.e(e)` block (which begins with the `error_handler()` line), the entire 50-column `SELECT` list (printed twice, as SQLAlchemy prints it), **every** stack frame with its full absolute path, both exception sections (the `psycopg2` cause and the `sqlalchemy` effect), and the trailing `after_request` line — nothing is omitted, abbreviated, or shortened:

```
2026-07-03 00:39:01,407 - SL - ERROR - 77953 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/server.py:390" - error_handler() -  - (psycopg2.errors.UndefinedTable) relation "users" does not exist
LINE 2: FROM users 
             ^

[SQL: SELECT users.directory_quota AS users_directory_quota, users.subdomain_quota AS users_subdomain_quota, users.password AS users_password, users.id AS users_id, users.created_at AS users_created_at, users.updated_at AS users_updated_at, users.email AS users_email, users.name AS users_name, users.is_admin AS users_is_admin, users.alias_generator AS users_alias_generator, users.notification AS users_notification, users.activated AS users_activated, users.disabled AS users_disabled, users.profile_picture_id AS users_profile_picture_id, users.otp_secret AS users_otp_secret, users.enable_otp AS users_enable_otp, users.last_otp AS users_last_otp, users.fido_uuid AS users_fido_uuid, users.default_alias_custom_domain_id AS users_default_alias_custom_domain_id, users.default_alias_public_domain_id AS users_default_alias_public_domain_id, users.lifetime AS users_lifetime, users.paid_lifetime AS users_paid_lifetime, users.lifetime_coupon_id AS users_lifetime_coupon_id, users.trial_end AS users_trial_end, users.default_mailbox_id AS users_default_mailbox_id, users.sender_format AS users_sender_format, users.sender_format_updated_at AS users_sender_format_updated_at, users.replace_reverse_alias AS users_replace_reverse_alias, users.referral_id AS users_referral_id, users.intro_shown AS users_intro_shown, users.max_spam_score AS users_max_spam_score, users.newsletter_alias_id AS users_newsletter_alias_id, users.include_sender_in_reverse_alias AS users_include_sender_in_reverse_alias, users.random_alias_suffix AS users_random_alias_suffix, users.expand_alias_info AS users_expand_alias_info, users.ignore_loop_email AS users_ignore_loop_email, users.alternative_id AS users_alternative_id, users.disable_automatic_alias_note AS users_disable_automatic_alias_note, users.one_click_unsubscribe_block_sender AS users_one_click_unsubscribe_block_sender, users.include_website_in_one_click_alias AS users_include_website_in_one_click_alias, users.disable_import AS users_disable_import, users.can_use_phone AS users_can_use_phone, users.phone_quota AS users_phone_quota, users.block_behaviour AS users_block_behaviour, users.include_header_email_header AS users_include_header_email_header, users.enable_data_breach_check AS users_enable_data_breach_check, users.flags AS users_flags, users.unsub_behaviour AS users_unsub_behaviour, users.delete_on AS users_delete_on 
FROM users 
WHERE users.email = %(email_1)s 
 LIMIT %(param_1)s]
[parameters: {'email_1': 'test@example.com', 'param_1': 1}]
(Background on this error at: http://sqlalche.me/e/13/f405)
Traceback (most recent call last):
  File "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/.venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1276, in _execute_context
    self.dialect.do_execute(
  File "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/.venv/lib/python3.10/site-packages/sqlalchemy/engine/default.py", line 608, in do_execute
    cursor.execute(statement, parameters)
psycopg2.errors.UndefinedTable: relation "users" does not exist
LINE 2: FROM users 
             ^


The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/.venv/lib/python3.10/site-packages/flask/app.py", line 1950, in full_dispatch_request
    rv = self.dispatch_request()
  File "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/.venv/lib/python3.10/site-packages/flask_debugtoolbar/__init__.py", line 125, in dispatch_request
    return view_func(**req.view_args)
  File "/root/.pyenv/versions/3.10.20/lib/python3.10/cProfile.py", line 110, in runcall
    return func(*args, **kw)
  File "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/.venv/lib/python3.10/site-packages/flask_limiter/extension.py", line 702, in __inner
    return obj(*a, **k)
  File "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/app/auth/views/login.py", line 43, in login
    user = User.get_by(email=email) or User.get_by(email=canonical_email)
  File "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/app/models.py", line 84, in get_by
    return Session.query(cls).filter_by(**kw).first()
  File "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/.venv/lib/python3.10/site-packages/sqlalchemy/orm/query.py", line 3429, in first
    ret = list(self[0:1])
  File "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/.venv/lib/python3.10/site-packages/sqlalchemy/orm/query.py", line 3203, in __getitem__
    return list(res)
  File "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/.venv/lib/python3.10/site-packages/sqlalchemy/orm/query.py", line 3535, in __iter__
    return self._execute_and_instances(context)
  File "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/.venv/lib/python3.10/site-packages/sqlalchemy/orm/query.py", line 3560, in _execute_and_instances
    result = conn.execute(querycontext.statement, self._params)
  File "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/.venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1011, in execute
    return meth(self, multiparams, params)
  File "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/.venv/lib/python3.10/site-packages/sqlalchemy/sql/elements.py", line 298, in _execute_on_connection
    return connection._execute_clauseelement(self, multiparams, params)
  File "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/.venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1124, in _execute_clauseelement
    ret = self._execute_context(
  File "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/.venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1316, in _execute_context
    self._handle_dbapi_exception(
  File "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/.venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1510, in _handle_dbapi_exception
    util.raise_(
  File "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/.venv/lib/python3.10/site-packages/sqlalchemy/util/compat.py", line 182, in raise_
    raise exception
  File "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/.venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1276, in _execute_context
    self.dialect.do_execute(
  File "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/.venv/lib/python3.10/site-packages/sqlalchemy/engine/default.py", line 608, in do_execute
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
2026-07-03 00:39:01,413 - SL - DEBUG - 77953 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 500, takes 0.015386581420898438
```

### 2.6 How the traceback surfaces (which path produced HTTP 500)

Claim: the traceback surfaces via the **registered global exception handler**, not werkzeug's interactive debugger.

Evidence (from §2.5): the very first line of the captured error is emitted by `error_handler()` at `server.py:390` — i.e. the `@app.errorhandler(Exception)` handler `def error_handler(e)` [server.py:389] whose body is `LOG.e(e)` [server.py:390] followed by `return render_template("error/500.html"), 500` [server.py:394] for non-`/api/` paths (the `/api/` branch would instead `return jsonify(error="Internal error"), 500` [server.py:391-392]). The `LOG.e(e)` call (recall `LOG.e = logging.Logger.exception` [app/log.py:77]) is what printed the full traceback to stdout; the client then received the rendered `error/500.html` (matching §2.4). The final `after_request` line shown at the end of §2.5 records `127.0.0.1 POST /auth/login ImmutableMultiDict([]) 500` [server.py:284], confirming the request completed with status 500.

**Cause → effect summary for Q1:** the connection opened at import [app/db.py:12] succeeds on the empty database, so `python server.py` starts and even serves the login-page GET; the failure occurs only on the **first request that queries a not-yet-migrated table** — the login-form POST, which runs `User.get_by(email=email)` [app/auth/views/login.py:43] → `Session.query(cls).filter_by(**kw).first()` [app/models.py:84] against the missing `users` table, raising `sqlalchemy.exc.ProgrammingError: (psycopg2.errors.UndefinedTable) relation "users" does not exist`.

### 2.7 Reproducibility

A second identical `POST /auth/login` (a different email) against the still-empty database reproduced the same result:

```
$ curl -s -b cookies -c cookies \
    --data-urlencode "email=second@example.com" \
    --data-urlencode "password=password123" \
    --data-urlencode "csrf_token=${CSRF}" \
    -o /dev/null -w "RUN2 RESPONSE_STATUS=%{http_code}\n" http://localhost:7777/auth/login
RUN2 RESPONSE_STATUS=500
```

The corresponding server-stdout lines for run #2 (the identical `error_handler()` error and the `500` `after_request` line):

```
2026-07-03 00:39:01,436 - SL - ERROR - 77953 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/server.py:390" - error_handler() -  - (psycopg2.errors.UndefinedTable) relation "users" does not exist
2026-07-03 00:39:01,437 - SL - DEBUG - 77953 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 500, takes 0.0065653324127197266
```

Both runs produced HTTP 500 and the identical `psycopg2.errors.UndefinedTable: relation "users" does not exist` / `sqlalchemy.exc.ProgrammingError`. The result reproduced identically.

---

## 3. Q2 — Required runtime services

### 3.1 Provision the migrated + initialized database

The canonical bring-up per the README is to run migrations, then the initialization script `init_app.py`: `flask db upgrade` [README.md:433] then `python init_app.py` [README.md:448]. Here `alembic upgrade head` (what `flask db upgrade` calls under the hood) was used directly.

Command and output — migrations. The full output is **264 lines** (the 5-line `app.config` import banner + `>>> init logging <<<` + the load-words DEBUG line + 2 `alembic.runtime.migration` context lines + **255** `Running upgrade` lines, one per migration version). Because it is long, it is shown below via explicit scoping commands, each with its **complete** output — no line is abbreviated:

```
$ export PATH="$PWD/.venv/bin:$PATH"; export PYTHONUNBUFFERED=1
$ alembic upgrade head > q2_alembic.txt 2>&1; echo "alembic exit=$?"
alembic exit=0
$ wc -l q2_alembic.txt
264 q2_alembic.txt
$ grep -c 'Running upgrade' q2_alembic.txt
255
$ head -9 q2_alembic.txt
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/zpnvoijavqovbilmsfzv
Upload files to local dir
>>> init logging <<<
2026-07-03 01:06:12,331 - SL - DEBUG - 82776 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/app/utils.py:17" - <module>() -  - load words file: /tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/local_data/test_words.txt
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
$ sed -n '10p' q2_alembic.txt      # the FIRST migration, applied from an empty database (blank base revision)
INFO  [alembic.runtime.migration] Running upgrade  -> 5e549314e1e2, empty message
$ tail -3 q2_alembic.txt           # the LAST three migrations, ending at head 32f25cbf12f6
INFO  [alembic.runtime.migration] Running upgrade 62afa3a10010 -> 91ed7f46dc81, alias_audit_log
INFO  [alembic.runtime.migration] Running upgrade 91ed7f46dc81 -> 7d7b84779837, user_audit_log
INFO  [alembic.runtime.migration] Running upgrade 7d7b84779837 -> 32f25cbf12f6, alias_audit_log_index_created_at
```

Claim: after migrations the `public` schema has **77** tables and the recorded Alembic revision is the head `32f25cbf12f6`.

```
$ docker exec sl-db psql -U myuser -d simplelogin -tAc "SELECT count(*) FROM information_schema.tables WHERE table_schema='public';"
77
$ docker exec sl-db psql -U myuser -d simplelogin -tAc "SELECT version_num FROM alembic_version;"
32f25cbf12f6
```

Command and complete output — `python init_app.py` (its `__main__` wraps `create_light_app().app_context()` [init_app.py:71] then calls `load_pgp_public_keys()` [init_app.py:72] and `add_sl_domains()` [init_app.py:73]):

```
$ python init_app.py
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/jlvayletvunpysmqcopu
Upload files to local dir
>>> init logging <<<
2026-07-03 01:06:40,204 - SL - DEBUG - 83498 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/app/utils.py:17" - <module>() -  - load words file: /tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/local_data/test_words.txt
2026-07-03 01:06:41,311 - SL - DEBUG - 83498 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/init_app.py:36" - load_pgp_public_keys() -  - Finish load_pgp_public_keys
2026-07-03 01:06:41,313 - SL - INFO - 83498 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain
```

Claim: `add_sl_domains()` seeded the `public_domain` table with the single domain `sl.local`.

```
$ docker exec sl-db psql -U myuser -d simplelogin -c "SELECT id, domain FROM public_domain;"
 id |  domain  
----+----------
  1 | sl.local
(1 row)
```

The database is now migrated (77 tables, alembic head `32f25cbf12f6`) and initialized (`add_sl_domains()` [init_app.py:39] seeded `public_domain` with `sl.local`, observed in the `Add sl.local to SL domain` log line above at `init_app.py:44`).

### 3.2 The five process entry points

`CONTRIBUTING.md` names the process entry points: `wsgi.py and server.py: the webapp` [CONTRIBUTING.md:145], `email_handler.py: the email handler` [CONTRIBUTING.md:146], and `cron.py: the cronjob` [CONTRIBUTING.md:147]; the repository additionally ships `job_runner.py` and `event_listener.py`. Each was started via its real process and its startup output plus an independent bind/liveness check captured. The canonical run order per the README is `flask db upgrade` [README.md:433] then `python init_app.py` [README.md:448] then `python email_handler.py` [README.md:480] then `python job_runner.py` [README.md:495].

Note on the independent bind/liveness proofs used below: this container has neither `ss` nor `lsof`, so port state is proven with Python's `socket.connect_ex((host, port))` — it returns `0` when a port is **listening** and `111` (ECONNREFUSED) when nothing is listening. "Binds no port" is proven per-process by enumerating the process's socket file descriptors under `/proc/<pid>/fd` and cross-referencing them against the `LISTEN`-state (`st == 0A`) sockets in `/proc/net/tcp` and `/proc/net/tcp6`, and process liveness is proven with `kill -0 <pid>` after a bounded wait.

#### 3.2.1 Webapp — `python server.py` (port 7777) — REQUIRED

Command and complete startup stdout. As in Q1, the output shows **two import passes with two PIDs** because werkzeug's reloader (enabled by `app.run(debug=True, port=7777)` [server.py:588]) imports the app in the reloader parent (PID `83926`) then spawns the worker child (PID `83940`); the last two lines are a benign `flask_debugtoolbar` `UserWarning` emitted while rendering the 7-byte `/health` response (which has no `</body>` tag for the toolbar to attach to):

```
$ python server.py            # started in its own session; stdout+stderr captured to a log
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/jesqroeljylcnhchnjhh
Upload files to local dir
>>> init logging <<<
2026-07-03 01:07:00,690 - SL - DEBUG - 83926 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/app/utils.py:17" - <module>() -  - load words file: /tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/pwqilpzxowxbjlasgbos
Upload files to local dir
>>> init logging <<<
2026-07-03 01:07:02,680 - SL - DEBUG - 83940 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/app/utils.py:17" - <module>() -  - load words file: /tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/local_data/test_words.txt
/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/.venv/lib/python3.10/site-packages/flask_debugtoolbar/__init__.py:213: UserWarning: Could not insert debug toolbar. </body> tag not found in response.
  warnings.warn('Could not insert debug toolbar.'
```

Independent readiness proof — the health route `return "success", 200` [server.py:213-215] returns HTTP 200:

```
$ curl -s -i http://localhost:7777/health
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 7
Vary: Cookie
Set-Cookie: slapp=eyJfcGVybWFuZW50Ijp0cnVlfQ.akcLQQ.sS_EgP0po_j1Mxu3___22wZ731A; Expires=Fri, 10-Jul-2026 01:07:13 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.20
Date: Fri, 03 Jul 2026 01:07:13 GMT

success
```

Independent bind proof — a raw TCP `connect_ex` to :7777 succeeds (`0`) while the server runs:

```
$ python3 -c "import socket; s=socket.socket(); print('connect_ex(127.0.0.1:7777) =', s.connect_ex(('127.0.0.1',7777)), '(0 == open/listening)'); s.close()"
connect_ex(127.0.0.1:7777) = 0 (0 == open/listening)
```

**Cause → effect:** the process binds **:7777** and answers `/health` with `success`/200, proving it is up and ready to accept connections. In production this same app is served by gunicorn instead of the dev server — `CMD ["gunicorn","wsgi:app","-b","0.0.0.0:7777","-w","2","--timeout","15"]` [Dockerfile:47] — where `wsgi.py` does `from server import create_app` then `app = create_app()`; either way it binds :7777. (The gunicorn production path is labelled `(inferred)` — it is read from the Dockerfile, not run here; the dev-server path above was run and observed.)

#### 3.2.2 Email handler — `python email_handler.py` (port 20381) — REQUIRED

Command and complete startup stdout (the last two lines are the readiness log lines):

```
$ python email_handler.py
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/hyhvbmboictjfjpscuhv
Upload files to local dir
>>> init logging <<<
2026-07-03 01:07:48,805 - SL - DEBUG - 84612 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/app/utils.py:17" - <module>() -  - load words file: /tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/local_data/test_words.txt
2026-07-03 01:07:49,557 - SL - INFO - 84612 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-03 01:07:49,559 - SL - DEBUG - 84612 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

These come from `LOG.i("Listen for port %s", args.port)` [email_handler.py:2403] and `LOG.d("Start mail controller %s %s", controller.hostname, controller.port)` [email_handler.py:2386]; the default port is argparse `default=20381` [email_handler.py:2399] and the controller binds `hostname="0.0.0.0"` [email_handler.py:2383]. Independent bind proof — a real TCP connection to :20381 returns the SMTP greeting, and `QUIT` is accepted:

```
$ python3 -c "import socket; s=socket.socket(); s.settimeout(5); s.connect(('127.0.0.1',20381)); print('SMTP BANNER:', repr(s.recv(1024).decode().strip())); s.sendall(b'QUIT\r\n'); print('QUIT REPLY:', repr(s.recv(1024).decode().strip())); s.close()"
SMTP BANNER: '220 reverse-code-generator-df363dc0-wcbzb Python SMTP 1.4.2'
QUIT REPLY: '221 Bye'
```

The `220` greeting proves it is bound and accepting connections; `Python SMTP 1.4.2` is the aiosmtpd version (matching §1.1), and `reverse-code-generator-df363dc0-wcbzb` is this container's hostname.

#### 3.2.3 Job runner — `python job_runner.py` (no port) — AUXILIARY

Command and complete startup stdout (only the import banner; it then polls the `Job` table):

```
$ python job_runner.py
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/yxqfwpnxaaevstyfszou
Upload files to local dir
>>> init logging <<<
2026-07-03 01:13:45,461 - SL - DEBUG - 87866 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/app/utils.py:17" - <module>() -  - load words file: /tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/local_data/test_words.txt
```

It prints nothing more on an empty `Job` table because its `__main__` is a `while True:` loop [job_runner.py:330] that, inside `create_light_app().app_context()`, calls `get_jobs_to_run()` [job_runner.py:307] and `process_job(job)` [job_runner.py:342] then `time.sleep(10)` [job_runner.py:347]; it logs per-job only when a job exists.

Liveness proof — after a bounded 8-second wait the process is still running (it did not exit):

```
$ sleep 8; kill -0 87866 && echo "ALIVE: pid 87866 still running after 8s (did not exit)" || echo DEAD
ALIVE: pid 87866 still running after 8s (did not exit)
```

No-port proof — the process holds no listening TCP socket (its one socket fd is the PostgreSQL connection, none in `LISTEN` state):

```
$ python3 - 87866 <<'PY'
import os, sys, glob, re
pid = sys.argv[1]
inodes = set()
for fd in glob.glob(f"/proc/{pid}/fd/*"):
    try:
        m = re.match(r"socket:\[(\d+)\]", os.readlink(fd))
        if m: inodes.add(m.group(1))
    except OSError: pass
listen = set()
for path in ("/proc/net/tcp", "/proc/net/tcp6"):
    with open(path) as f:
        next(f)
        for line in f:
            p = line.split()
            if p[3] == "0A": listen.add(p[9])   # 0A == TCP_LISTEN
own = inodes & listen
print(f"job_runner pid {pid}: socket-fd count={len(inodes)}, in LISTEN state={len(own)}")
print("LISTENING sockets owned:", sorted(own) if own else "NONE - process binds no TCP port")
PY
job_runner pid 87866: socket-fd count=1, in LISTEN state=0
LISTENING sockets owned: NONE - process binds no TCP port
```

#### 3.2.4 Event listener — `python event_listener.py` (no port; requires a subcommand) — AUXILIARY

It requires a subcommand. With none, it prints a usage error and exits non-zero:

```
$ python event_listener.py; echo "exit=$?"
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/vizooqxvdppwboorwyks
Upload files to local dir
>>> init logging <<<
2026-07-03 01:08:57,429 - SL - DEBUG - 85678 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/app/utils.py:17" - <module>() -  - load words file: /tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/local_data/test_words.txt
Invalid usage. Pass a valid subcommand as argument
exit=1
```

With the `listener` subcommand it starts the PostgreSQL LISTEN/NOTIFY consumer (source `postgresql://myuser:mypassword@localhost:15432/simplelogin`, since `EVENT_LISTENER_DB_URI = os.environ.get("EVENT_LISTENER_DB_URI", DB_URI)` [app/config.py:637]). Complete startup stdout:

```
$ python event_listener.py listener
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/ziolydaviamjtryqvcft
Upload files to local dir
>>> init logging <<<
2026-07-03 01:14:10,551 - SL - DEBUG - 88138 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/app/utils.py:17" - <module>() -  - load words file: /tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/local_data/test_words.txt
2026-07-03 01:14:10,689 - SL - INFO - 88138 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/event_listener.py:34" - main() -  - Using PostgresEventSource
2026-07-03 01:14:10,693 - SL - INFO - 88138 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/event_listener.py:43" - main() -  - Starting with HttpEventSink
2026-07-03 01:14:10,693 - SL - INFO - 88138 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/events/event_source.py:49" - __listen() -  - Starting to listen to events
```

Those come from `LOG.i("Using PostgresEventSource")` [event_listener.py:34], `LOG.i("Starting with HttpEventSink")` [event_listener.py:43], and `LOG.info("Starting to listen to events")` [events/event_source.py:49]. Its modes are `DEAD_LETTER = "dead_letter"` [event_listener.py:16] and `LISTENER = "listener"` [event_listener.py:17].

No-port proof — like the job runner, it binds no TCP port (its socket fds are PostgreSQL connections, none in `LISTEN` state):

```
$ python3 - 88138 <<'PY'
import os, sys, glob, re
pid = sys.argv[1]
inodes = set()
for fd in glob.glob(f"/proc/{pid}/fd/*"):
    try:
        m = re.match(r"socket:\[(\d+)\]", os.readlink(fd))
        if m: inodes.add(m.group(1))
    except OSError: pass
listen = set()
for path in ("/proc/net/tcp", "/proc/net/tcp6"):
    with open(path) as f:
        next(f)
        for line in f:
            p = line.split()
            if p[3] == "0A": listen.add(p[9])
own = inodes & listen
print(f"event_listener pid {pid}: socket-fd count={len(inodes)}, in LISTEN state={len(own)}")
print("LISTENING sockets owned:", sorted(own) if own else "NONE - process binds no TCP port")
PY
event_listener pid 88138: socket-fd count=2, in LISTEN state=0
LISTENING sockets owned: NONE - process binds no TCP port
```

#### 3.2.5 Cron — `cron.py` (no port; scheduler-driven, not a daemon) — AUXILIARY

`cron.py` is not a persistent daemon: it is invoked per-job by a scheduler, takes a `-j`/`--job` argument [cron.py:1266-1267], runs that one job, and exits. To observe its real startup output, it was invoked exactly as `crontab.yml` invokes it for the growth-stats job — `command: python /code/cron.py -j stats` on `schedule: "0 0 * * *"` [crontab.yml:2-5]. Complete output:

```
$ python cron.py -j stats; echo "cron.py -j stats exit=$?"
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/kytiayabdntxdkuegenl
Upload files to local dir
>>> init logging <<<
2026-07-03 01:09:30,373 - SL - DEBUG - 86208 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/app/utils.py:17" - <module>() -  - load words file: /tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/local_data/test_words.txt
2026-07-03 01:09:31,307 - SL - DEBUG - 86208 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/cron.py:1263" - <module>() -  - Start running cronjob
2026-07-03 01:09:31,308 - SL - DEBUG - 86208 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/cron.py:1275" - <module>() -  - Compute growth and daily monitoring stats
2026-07-03 01:09:31,308 - SL - WARNING - 86208 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/cron.py:540" - stats() -  - ADMIN_EMAIL not set, nothing to do
cron.py -j stats exit=0
```

**Cause → effect:** `-j stats` runs the `if args.job == "stats":` branch [cron.py:1274], logging `Start running cronjob` [cron.py:1263] then `Compute growth and daily monitoring stats` [cron.py:1275]; the job then logs `ADMIN_EMAIL not set, nothing to do` and returns early — from `LOG.w("ADMIN_EMAIL not set, nothing to do")` [cron.py:540], because `ADMIN_EMAIL` is unset in the canonical `.env`. The process exits `0` immediately after; it never binds a port and never loops. `crontab.yml` schedules **15** such one-shot jobs: `stats`, `delete_old_monitoring`, `check_custom_domain`, `check_hibp`, `notify_hibp`, `delete_logs`, `delete_old_data`, `poll_apple_subscription`, `notify_trial_end`, `notify_manual_subscription_end`, `notify_premium_end`, `delete_scheduled_users`, `send_undelivered_mails`, `clear_alias_audit_log`, and `clear_user_audit_log` [crontab.yml:1-96].

### 3.3 Which services must run "for the system to work"

**Answer:** the two services that must be running are the **webapp (`python server.py`, or `gunicorn wsgi:app` in production) on :7777** and the **email handler (`python email_handler.py`) on :20381**.

This is grounded both in the observed binds above and in the project's own statement that the backend has exactly two main components. The verbatim lines from `CONTRIBUTING.md`:

```
SimpleLogin backend consists of 2 main components:

- the `webapp` used by several clients: the web app, the browser extensions (Chrome & Firefox for now), OAuth clients (apps that integrate "Sign in with SimpleLogin" button) and mobile apps.

- the `email handler`: implements the email forwarding (i.e. alias receiving email) and email sending (i.e. alias sending email).
```

**Cause → effect, per service:**

- **Webapp (`server.py`/`wsgi.py`) — REQUIRED.** It is the HTTP tier used by the clients named above (web app, browser extensions, OAuth clients and mobile apps) [CONTRIBUTING.md:16]; without it there is no UI/API. Proven ready by `GET /health` returning `success`/200 and `connect_ex(:7777)=0` (§3.2.1).
- **Email handler (`email_handler.py`) — REQUIRED.** It implements alias email forwarding and sending [CONTRIBUTING.md:18] — SimpleLogin's core function. Proven ready by the `220` SMTP banner on :20381 (§3.2.2).
- **Job runner (`job_runner.py`) — AUXILIARY.** A background poller of the `Job` table (async work such as onboarding emails, deletions); the system serves requests and forwards mail without it. Stays alive but binds no port (§3.2.3).
- **Event listener (`event_listener.py`) — AUXILIARY.** A PostgreSQL event consumer (event fan-out); needs an explicit `listener`/`dead_letter` subcommand and binds no port (§3.2.4).
- **Cron (`cron.py`) — AUXILIARY.** Scheduled maintenance invoked by a scheduler per `crontab.yml`, one process per job, not a persistent daemon (§3.2.5).

### 3.4 Reproducibility

The two required services were restarted and re-verified. Webapp run #2:

```
$ python server.py    # run #2
$ curl -s -w " HTTP %{http_code}\n" http://localhost:7777/health
success HTTP 200
```

Email handler run #2 — the same SMTP banner and the same readiness log lines:

```
$ python email_handler.py    # run #2
$ python3 -c "import socket; s=socket.socket(); s.settimeout(5); s.connect(('127.0.0.1',20381)); print('SMTP BANNER:', repr(s.recv(1024).decode().strip())); s.close()"
SMTP BANNER: '220 reverse-code-generator-df363dc0-wcbzb Python SMTP 1.4.2'
$ grep -E 'Listen for port|Start mail controller' q2_email2.log
2026-07-03 01:10:31,989 - SL - INFO - 86776 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-03 01:10:31,991 - SL - DEBUG - 86776 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

Both reproduced identically (webapp `success`/200; email handler the same `220 reverse-code-generator-df363dc0-wcbzb Python SMTP 1.4.2` banner and the same `Listen for port 20381` / `Start mail controller 0.0.0.0 20381` lines).

---

## 4. Q3 — Skipped-initialization rejection

### 4.1 Provision "migrated but not initialized" and verify the SLDomain table is empty

Run migrations but **skip `init_app.py`** (and skip `flask dummy-data`, which would also seed the table via `add_sl_domains()`). The SLDomain model is physically the `public_domain` table — `class SLDomain(Base, ModelMixin)` [app/models.py:3116] with `__tablename__ = "public_domain"` [app/models.py:3119].

The exact commands run (drop + recreate the database, migrate only; `init_app.py` and `flask dummy-data` are **not** run):

```
$ docker exec sl-db dropdb -U myuser simplelogin
dropdb exit=0
$ docker exec sl-db createdb -U myuser -O myuser simplelogin
createdb exit=0
$ alembic upgrade head        # migrations only; init_app.py and flask dummy-data NOT run
alembic exit=0
```

Verify **the SLDomain table should be empty (no email domains configured)** — count is `0` and the table has zero rows:

```
$ docker exec sl-db psql -U myuser -d simplelogin -c "SELECT count(*) FROM public_domain;"
 count 
-------
     0
(1 row)

$ docker exec sl-db psql -U myuser -d simplelogin -c "SELECT * FROM public_domain;"
 id | created_at | updated_at | domain | premium_only | can_use_subdomain | hidden | order | partner_id | use_as_reverse_alias 
----+------------+------------+--------+--------------+-------------------+--------+-------+------------+----------------------
(0 rows)
```

The `public_domain` table exists (migrations created its schema) but has **0 rows** — confirming the precondition. This directly contrasts with Q2, where `python init_app.py` seeded it with `sl.local` (§3.1).

### 4.2 Start the email handler and inject a real message to `xyz@sl.local`

Start the real live listener. Its **complete** startup stdout (verbatim), showing it binds port 20381 and starts the aiosmtpd controller:

```
$ python email_handler.py
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/qrbpwamvjqpbbqozetmf
Upload files to local dir
>>> init logging <<<
2026-07-03 01:19:18,015 - SL - DEBUG - 91017 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/app/utils.py:17" - <module>() -  - load words file: /tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/local_data/test_words.txt
2026-07-03 01:19:18,764 - SL - INFO - 91017 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-03 01:19:18,765 - SL - DEBUG - 91017 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

The last two lines are the readiness markers: `Listen for port 20381` (`LOG.i("Listen for port %s", args.port)` [email_handler.py:2403], default `20381` [email_handler.py:2399]) and `Start mail controller 0.0.0.0 20381` (`LOG.d("Start mail controller %s %s", controller.hostname, controller.port)` [email_handler.py:2386], from `Controller(MailHandler(), hostname="0.0.0.0", port=port)` [email_handler.py:2383]).

Inject a real message through the **live aiosmtpd listener** on `localhost:20381` using `smtplib` (this exercises the real `MailHandler.handle_DATA` path — no handler function is called directly). The complete client script `/tmp/blitzy_repro/inject_q3.py`:

```python
import smtplib, sys
from email.message import EmailMessage
sender = sys.argv[1] if len(sys.argv) > 1 else "external-sender@example.com"
msg = EmailMessage()
msg["From"] = sender
msg["To"] = "xyz@sl.local"
msg["Subject"] = "Q3 skipped-init rejection test"
msg.set_content("This is a test message injected via the live aiosmtpd listener.")
s = smtplib.SMTP("localhost", 20381, timeout=15)
s.set_debuglevel(1)
try:
    s.sendmail(sender, ["xyz@sl.local"], msg.as_string())
    print("UNEXPECTED: sendmail returned without exception")
except smtplib.SMTPDataError as e:
    print(f"SMTP_CODE: {e.smtp_code}")
    print(f"SMTP_ERROR: {e.smtp_error.decode() if isinstance(e.smtp_error, bytes) else e.smtp_error}")
except smtplib.SMTPRecipientsRefused as e:
    print(f"RECIPIENTS_REFUSED: {e.recipients}")
finally:
    try: s.quit()
    except Exception: pass
```

### 4.3 The SMTP status code returned to the sender: `550 SL E515 Email not exist`

Claim: the email handler returns SMTP **`550 SL E515 Email not exist`** to the sender at end-of-DATA.

The **complete** raw SMTP wire captured by `smtplib` `set_debuglevel(1)` for run #1 (sender `external-sender@example.com`), verbatim — note RCPT TO first returns `250 OK`, and the rejection is delivered only at end-of-DATA:

```
$ python3 /tmp/blitzy_repro/inject_q3.py external-sender@example.com
send: 'ehlo [10.236.7.59]\r\n'
reply: b'250-reverse-code-generator-df363dc0-wcbzb\r\n'
reply: b'250-SIZE 33554432\r\n'
reply: b'250-8BITMIME\r\n'
reply: b'250-SMTPUTF8\r\n'
reply: b'250 HELP\r\n'
reply: retcode (250); Msg: b'reverse-code-generator-df363dc0-wcbzb\nSIZE 33554432\n8BITMIME\nSMTPUTF8\nHELP'
send: 'mail FROM:<external-sender@example.com> size=256\r\n'
reply: b'250 OK\r\n'
reply: retcode (250); Msg: b'OK'
send: 'rcpt TO:<xyz@sl.local>\r\n'
reply: b'250 OK\r\n'
reply: retcode (250); Msg: b'OK'
send: 'data\r\n'
reply: b'354 End data with <CR><LF>.<CR><LF>\r\n'
reply: retcode (354); Msg: b'End data with <CR><LF>.<CR><LF>'
data: (354, b'End data with <CR><LF>.<CR><LF>')
send: b'From: external-sender@example.com\r\nTo: xyz@sl.local\r\nSubject: Q3 skipped-init rejection test\r\nContent-Type: text/plain; charset="utf-8"\r\nContent-Transfer-Encoding: 7bit\r\nMIME-Version: 1.0\r\n\r\nThis is a test message injected via the live aiosmtpd listener.\r\n.\r\n'
reply: b'550 SL E515 Email not exist\r\n'
reply: retcode (550); Msg: b'SL E515 Email not exist'
data: (550, b'SL E515 Email not exist')
send: 'rset\r\n'
reply: b'250 OK\r\n'
reply: retcode (250); Msg: b'OK'
SMTP_CODE: 550
SMTP_ERROR: SL E515 Email not exist
send: 'quit\r\n'
reply: b'221 Bye\r\n'
reply: retcode (221); Msg: b'Bye'
```

The decisive reply is `550 SL E515 Email not exist` (the `reply:` line at end-of-DATA in the wire above), parsed by the client as `SMTP_CODE: 550` / `SMTP_ERROR: SL E515 Email not exist`. This literal is the constant `E515 = "550 SL E515 Email not exist"` [app/email/status.py:51].

### 4.4 The rejection log lines (verbatim)

Claim: the handler logs the alias-does-not-exist attempt, both auto-create failures, and the cannot-create-on-the-fly rejection.

The **complete** handler stdout for the run #1 injected message (message-id `5f0ade8c-4d13-4733-874c-db61fefcbf61`), verbatim — every line for this message, with full paths and full message-id:

```
2026-07-03 01:19:32,718 - SL - DEBUG - 91017 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/app/log.py:24" - set_message_id() -  - set message_id 5f0ade8c-4d13-4733-874c-db61fefcbf61
2026-07-03 01:19:32,718 - SL - DEBUG - 91017 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/email_handler.py:2342" - _handle() - 5f0ade8c-4d13-4733-874c-db61fefcbf61 - ====>=====>====>====>====>====>====>====>
2026-07-03 01:19:32,718 - SL - INFO - 91017 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/email_handler.py:2343" - _handle() - 5f0ade8c-4d13-4733-874c-db61fefcbf61 - New message, mail from external-sender@example.com, rctp tos ['xyz@sl.local'] 
2026-07-03 01:19:32,720 - SL - DEBUG - 91017 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/email_handler.py:1963" - handle() - 5f0ade8c-4d13-4733-874c-db61fefcbf61 - Cannot parse Postfix queue ID from None None
2026-07-03 01:19:32,869 - SL - DEBUG - 91017 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/email_handler.py:1980" - handle() - 5f0ade8c-4d13-4733-874c-db61fefcbf61 - ==>> Handle mail_from:external-sender@example.com, rcpt_tos:['xyz@sl.local'], header_from:external-sender@example.com, header_to:xyz@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'external-sender@example.com'), ('To', 'xyz@sl.local'), ('Subject', 'Q3 skipped-init rejection test'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:['SIZE=256'], rcpt_options:[]
2026-07-03 01:19:32,874 - SL - DEBUG - 91017 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/email_handler.py:2202" - handle() - 5f0ade8c-4d13-4733-874c-db61fefcbf61 - Forward phase external-sender@example.com(external-sender@example.com) -> xyz@sl.local
2026-07-03 01:19:32,886 - SL - DEBUG - 91017 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/email_handler.py:545" - handle_forward() - 5f0ade8c-4d13-4733-874c-db61fefcbf61 - alias xyz@sl.local not exist. Try to see if it can be created on the fly
2026-07-03 01:19:32,897 - SL - INFO - 91017 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - 5f0ade8c-4d13-4733-874c-db61fefcbf61 - Cannot auto-create custom domain alias for xyz@sl.local because there's no custom domain for sl.local
2026-07-03 01:19:32,897 - SL - INFO - 91017 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - 5f0ade8c-4d13-4733-874c-db61fefcbf61 - Cannot auto-create xyz@sl.local since it has no directory separator
2026-07-03 01:19:32,897 - SL - DEBUG - 91017 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/email_handler.py:551" - handle_forward() - 5f0ade8c-4d13-4733-874c-db61fefcbf61 - alias xyz@sl.local cannot be created on-the-fly, return 550
2026-07-03 01:19:32,898 - SL - INFO - 91017 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/email_handler.py:2367" - _handle() - 5f0ade8c-4d13-4733-874c-db61fefcbf61 - Finish mail_from external-sender@example.com, rcpt_tos ['xyz@sl.local'], takes 0.17997241020202637 seconds with return code '550 SL E515 Email not exist'<<===
```

Reading the decisive lines as cause → effect:

- `New message, mail from external-sender@example.com, rctp tos ['xyz@sl.local']` — the `LOG.i` info call [email_handler.py:2343] records the inbound message entering `_handle()`.
- `Forward phase external-sender@example.com(external-sender@example.com) -> xyz@sl.local` [email_handler.py:2202] — the message is routed to the plain forward path `handle_forward()`.
- `alias xyz@sl.local not exist. Try to see if it can be created on the fly` — `LOG.d("alias %s not exist. Try to see if it can be created on the fly", alias_address)` [email_handler.py:545] fires because `Alias.get_by(email=alias_address)` [email_handler.py:543] returned nothing.
- `Cannot auto-create custom domain alias for xyz@sl.local because there's no custom domain for sl.local` [app/alias_utils.py:104] — the custom-domain auto-create strategy fails (no `CustomDomain` for `sl.local`).
- `Cannot auto-create xyz@sl.local since it has no directory separator` [app/alias_utils.py:165] — the directory auto-create strategy fails (the plain address has no `/`, `+`, or `#`).
- `alias xyz@sl.local cannot be created on-the-fly, return 550` — `LOG.d("alias %s cannot be created on-the-fly, return 550", alias_address)` [email_handler.py:551] fires because `try_auto_create()` returned `None`.
- `Finish mail_from external-sender@example.com, rcpt_tos ['xyz@sl.local'], takes 0.17997241020202637 seconds with return code '550 SL E515 Email not exist'<<===` [email_handler.py:2367] confirms the returned reply is exactly `'550 SL E515 Email not exist'`.

### 4.5 Both status-code variants of the rejection (E515 vs E207)

The reject branch itself chooses between two status codes [email_handler.py:552-555]:

```python
if should_ignore_bounce(envelope.mail_from):
    return [(True, status.E207)]
else:
    return [(False, status.E515)]
```

- **Primary (observed): `status.E515` = `"550 SL E515 Email not exist"`** [app/email/status.py:51] → SMTP `550`. This is what a normal external sender receives — exactly what was observed in §4.3/§4.4.
- **Edge variant: `status.E207` = `"250 SL E207 No bounce report"`** [app/email/status.py:12] → SMTP `250`. Returned only when `should_ignore_bounce(envelope.mail_from)` is true. That function `def should_ignore_bounce(mail_from: str) -> bool:` [app/email_utils.py:1361] returns True only if `IgnoreBounceSender.get_by(mail_from=mail_from)` matches. Our senders (`external-sender@example.com`, `someoneelse@test.org`) are **not** ignore-bounce senders, so the branch takes the `else` and returns `E515` — which is why `E515` (SMTP `550`) was observed rather than `E207` (SMTP `250`).

### 4.6 Cause → effect: the true role of the empty SLDomain / `public_domain` table

Claim: for a **plain** `xyz@sl.local` recipient, the `550` arises because **no alias exists and none can be auto-created** — and this is **independent of the `public_domain` (SLDomain) table being empty**.

Traced through `handle_forward()` [email_handler.py:536]: `alias = Alias.get_by(email=alias_address)` [email_handler.py:543] returns nothing → logs the "not exist" line [email_handler.py:545] → `alias = try_auto_create(alias_address)` [email_handler.py:549; app/alias_utils.py:202] returns `None` → logs "cannot be created on-the-fly, return 550" [email_handler.py:551] → returns `E515`. `try_auto_create` returns `None` because **both** auto-create strategies fail, exactly as the run #1 logs in §4.4 show:

- **Via custom domain** — `check_if_alias_can_be_auto_created_for_custom_domain()` [app/alias_utils.py:92] fails at `custom_domain = CustomDomain.get_by(domain=alias_domain)` → no match for `sl.local`, so it logs "Cannot auto-create custom domain alias for xyz@sl.local because there's no custom domain for sl.local" [app/alias_utils.py:104] and returns `None`. This consults the `custom_domain` table, **not** `public_domain`.
- **Via directory** — `check_if_alias_can_be_auto_created_for_a_directory()` [app/alias_utils.py:145] fails: the plain address has no directory separator, so it logs "Cannot auto-create xyz@sl.local since it has no directory separator" [app/alias_utils.py:165] and returns `None`.

Crucially, the directory *gate* `can_create_directory_for_address()` [app/email_utils.py:545] iterates **`config.ALIAS_DOMAINS`** (`for domain in config.ALIAS_DOMAINS:` [app/email_utils.py:548]) — an **env-derived** value, **not** the `SLDomain`/`public_domain` table. Since `config.ALIAS_DOMAINS == ['sl.local']` (from `EMAIL_DOMAIN`, §1.3), that gate actually *passes* for `xyz@sl.local` — which is precisely why the "does not belong to a valid directory domain" log [app/email_utils.py:551] is **absent** from §4.4, and the directory strategy instead fails only at the "no directory separator" check [app/alias_utils.py:165]. So the empty `public_domain` table is **not** what produced this `550`.

What the `public_domain` (SLDomain) table *does* gate are **other** code paths — not the plain-forward auto-create path:

- `is_valid_alias_address_domain()` [app/email_utils.py:557] returns True when `SLDomain.get_by(domain=domain)` matches [app/email_utils.py:560].
- The reply / reverse-alias domain lookup uses `sl_domain: SLDomain = SLDomain.get_by(domain=reply_domain)` [email_handler.py:978].

`init_app.py`'s `add_sl_domains()` [init_app.py:39] — which calls `SLDomain.create(domain=alias_domain, use_as_reverse_alias=True)` [init_app.py:45] and logs `Add %s to SL domain` [init_app.py:44] — is what *would* populate `public_domain` from `ALIAS_DOMAINS`; skipping it is why the table is empty. But for this **plain-address forward**, the `550 SL E515 Email not exist` is caused by the missing, non-auto-creatable **alias** — not by the empty `public_domain` table per se.

### 4.7 Reproducibility

The injection was performed three times against the same running handler (senders `external-sender@example.com`, `someoneelse@test.org`, then `external-sender@example.com` again). Every run returned the identical SMTP code and rejection log line. The **complete** handler stdout for run #2 (message-id `fad2106a-6f99-402f-b3a1-e65909cc634e`, sender `someoneelse@test.org`), verbatim:

```
2026-07-03 01:19:48,679 - SL - DEBUG - 91017 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/app/log.py:24" - set_message_id() - 5f0ade8c-4d13-4733-874c-db61fefcbf61 - set message_id fad2106a-6f99-402f-b3a1-e65909cc634e
2026-07-03 01:19:48,679 - SL - DEBUG - 91017 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/email_handler.py:2342" - _handle() - fad2106a-6f99-402f-b3a1-e65909cc634e - ====>=====>====>====>====>====>====>====>
2026-07-03 01:19:48,679 - SL - INFO - 91017 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/email_handler.py:2343" - _handle() - fad2106a-6f99-402f-b3a1-e65909cc634e - New message, mail from someoneelse@test.org, rctp tos ['xyz@sl.local'] 
2026-07-03 01:19:48,680 - SL - DEBUG - 91017 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/email_handler.py:1963" - handle() - fad2106a-6f99-402f-b3a1-e65909cc634e - Cannot parse Postfix queue ID from None None
2026-07-03 01:19:48,682 - SL - DEBUG - 91017 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/email_handler.py:1980" - handle() - fad2106a-6f99-402f-b3a1-e65909cc634e - ==>> Handle mail_from:someoneelse@test.org, rcpt_tos:['xyz@sl.local'], header_from:someoneelse@test.org, header_to:xyz@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'someoneelse@test.org'), ('To', 'xyz@sl.local'), ('Subject', 'Q3 skipped-init rejection test'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:['SIZE=249'], rcpt_options:[]
2026-07-03 01:19:48,687 - SL - DEBUG - 91017 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/email_handler.py:2202" - handle() - fad2106a-6f99-402f-b3a1-e65909cc634e - Forward phase someoneelse@test.org(someoneelse@test.org) -> xyz@sl.local
2026-07-03 01:19:48,694 - SL - DEBUG - 91017 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/email_handler.py:545" - handle_forward() - fad2106a-6f99-402f-b3a1-e65909cc634e - alias xyz@sl.local not exist. Try to see if it can be created on the fly
2026-07-03 01:19:48,700 - SL - INFO - 91017 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - fad2106a-6f99-402f-b3a1-e65909cc634e - Cannot auto-create custom domain alias for xyz@sl.local because there's no custom domain for sl.local
2026-07-03 01:19:48,701 - SL - INFO - 91017 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - fad2106a-6f99-402f-b3a1-e65909cc634e - Cannot auto-create xyz@sl.local since it has no directory separator
2026-07-03 01:19:48,701 - SL - DEBUG - 91017 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/email_handler.py:551" - handle_forward() - fad2106a-6f99-402f-b3a1-e65909cc634e - alias xyz@sl.local cannot be created on-the-fly, return 550
2026-07-03 01:19:48,701 - SL - INFO - 91017 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/email_handler.py:2367" - _handle() - fad2106a-6f99-402f-b3a1-e65909cc634e - Finish mail_from someoneelse@test.org, rcpt_tos ['xyz@sl.local'], takes 0.0223236083984375 seconds with return code '550 SL E515 Email not exist'<<===
```

And the **complete** handler stdout for run #3 (message-id `134a39ba-4c62-4940-b62c-3810e3c3ebd3`, sender `external-sender@example.com`), verbatim:

```
2026-07-03 01:19:48,766 - SL - DEBUG - 91017 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/app/log.py:24" - set_message_id() - fad2106a-6f99-402f-b3a1-e65909cc634e - set message_id 134a39ba-4c62-4940-b62c-3810e3c3ebd3
2026-07-03 01:19:48,767 - SL - DEBUG - 91017 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/email_handler.py:2342" - _handle() - 134a39ba-4c62-4940-b62c-3810e3c3ebd3 - ====>=====>====>====>====>====>====>====>
2026-07-03 01:19:48,767 - SL - INFO - 91017 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/email_handler.py:2343" - _handle() - 134a39ba-4c62-4940-b62c-3810e3c3ebd3 - New message, mail from external-sender@example.com, rctp tos ['xyz@sl.local'] 
2026-07-03 01:19:48,767 - SL - DEBUG - 91017 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/email_handler.py:1963" - handle() - 134a39ba-4c62-4940-b62c-3810e3c3ebd3 - Cannot parse Postfix queue ID from None None
2026-07-03 01:19:48,769 - SL - DEBUG - 91017 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/email_handler.py:1980" - handle() - 134a39ba-4c62-4940-b62c-3810e3c3ebd3 - ==>> Handle mail_from:external-sender@example.com, rcpt_tos:['xyz@sl.local'], header_from:external-sender@example.com, header_to:xyz@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'external-sender@example.com'), ('To', 'xyz@sl.local'), ('Subject', 'Q3 skipped-init rejection test'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:['SIZE=256'], rcpt_options:[]
2026-07-03 01:19:48,773 - SL - DEBUG - 91017 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/email_handler.py:2202" - handle() - 134a39ba-4c62-4940-b62c-3810e3c3ebd3 - Forward phase external-sender@example.com(external-sender@example.com) -> xyz@sl.local
2026-07-03 01:19:48,779 - SL - DEBUG - 91017 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/email_handler.py:545" - handle_forward() - 134a39ba-4c62-4940-b62c-3810e3c3ebd3 - alias xyz@sl.local not exist. Try to see if it can be created on the fly
2026-07-03 01:19:48,784 - SL - INFO - 91017 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - 134a39ba-4c62-4940-b62c-3810e3c3ebd3 - Cannot auto-create custom domain alias for xyz@sl.local because there's no custom domain for sl.local
2026-07-03 01:19:48,784 - SL - INFO - 91017 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - 134a39ba-4c62-4940-b62c-3810e3c3ebd3 - Cannot auto-create xyz@sl.local since it has no directory separator
2026-07-03 01:19:48,784 - SL - DEBUG - 91017 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/email_handler.py:551" - handle_forward() - 134a39ba-4c62-4940-b62c-3810e3c3ebd3 - alias xyz@sl.local cannot be created on-the-fly, return 550
2026-07-03 01:19:48,785 - SL - INFO - 91017 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/email_handler.py:2367" - _handle() - 134a39ba-4c62-4940-b62c-3810e3c3ebd3 - Finish mail_from external-sender@example.com, rcpt_tos ['xyz@sl.local'], takes 0.018238067626953125 seconds with return code '550 SL E515 Email not exist'<<===
```

The client-parsed SMTP result was identical on both re-runs (extracted verbatim from the `smtplib` debug transcripts `q3_wire2.txt` and `q3_wire3.txt`):

```
# run #2 (someoneelse@test.org):
reply: b'550 SL E515 Email not exist\r\n'
SMTP_CODE: 550
SMTP_ERROR: SL E515 Email not exist
# run #3 (external-sender@example.com):
reply: b'550 SL E515 Email not exist\r\n'
SMTP_CODE: 550
SMTP_ERROR: SL E515 Email not exist
```

All three runs reproduced `550 SL E515 Email not exist` with the identical `email_handler.py:551` rejection log line and the identical `email_handler.py:2367` finish line carrying return code `'550 SL E515 Email not exist'`.

---

## 5. Read-only scope confirmation

The investigation created, modified, or deleted **no existing repository file**. The sole change introduced on this branch relative to the pristine upstream baseline is the single new deliverable, `blitzy/documentation/app_2cd6ee777f8c.md`. All transient artifacts — the temporary observation scripts, the working `.env`, the in-project virtualenv `.venv`, the capture directory `/tmp/blitzy_repro`, and the three transient PostgreSQL states — live outside the tracked source tree or are gitignored (`.env` and `.venv` are listed in `.gitignore`), and the temporary scripts were removed after the investigation.

The strongest proof is the whole-branch diff against the pristine baseline commit `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c` (the upstream tip this branch was cut from, i.e. the parent of the first commit that added the deliverable). The only file that differs is the deliverable, added (`A`); no existing source, configuration, documentation, or dependency file is Added, Modified (`M`), or Deleted (`D`):

```
$ git diff --name-status 2cd6ee777f8c2d3531559588bcfb18627ffb5d2c..HEAD
A	blitzy/documentation/app_2cd6ee777f8c.md
```

`git diff --stat` over the same range likewise reports exactly one file changed — the deliverable — with every line counted as an insertion, because the baseline contained no such file (and no `blitzy/` directory). The working-tree status captured at the time of writing (while finalizing this document, before its final commit) shows the deliverable as the only path with any change — no other file is modified, added, or untracked:

```
$ git status --porcelain --untracked-files=all
 M blitzy/documentation/app_2cd6ee777f8c.md
```

The single listed path `blitzy/documentation/app_2cd6ee777f8c.md` is this document itself; the leading ` M` is the working-tree modification column for the edits made while finalizing it. No existing source, configuration, documentation, or dependency file appears in either command's output, confirming the read-only scope was preserved.

---

## 6. Summary answers

- **Q1:** `python server.py` starts against an empty database (import-time `engine.connect()` [app/db.py:12] succeeds) and even serves `GET /auth/login`; submitting the login form (POST) runs `User.get_by(email=email)` [app/auth/views/login.py:43] and raises **`sqlalchemy.exc.ProgrammingError` wrapping `psycopg2.errors.UndefinedTable`: `relation "users" does not exist`** (first failing relation `users`), which the global `@app.errorhandler(Exception)` [server.py:389-394] logs via `LOG.e(e)` and renders as `error/500.html` with **HTTP 500**.
- **Q2:** two services must run for the system to work — the **webapp** (`python server.py` / `gunicorn wsgi:app`, :7777) and the **email handler** (`python email_handler.py`, :20381); `job_runner.py`, `event_listener.py`, and `cron.py` are auxiliary.
- **Q3:** with `public_domain` empty, a message to `xyz@sl.local` is rejected with **`550 SL E515 Email not exist`** and the log line **`alias xyz@sl.local cannot be created on-the-fly, return 550`** — caused by "no alias exists and none can be auto-created", independent of the empty `SLDomain`/`public_domain` table.

---

## 7. Coverage-pass checklist

- [x] `python server.py` — used verbatim as the Q1 start command (§2.2) and the Q2 webapp start command (§3.2.1).
- [x] `init_app.py` — run for Q2 (§3.1, "Add sl.local to SL domain") and skipped for Q3 (§4.1); both states contrasted.
- [x] `email_handler` / `email_handler.py` — started for Q2 (bind :20381, §3.2.2) and Q3 (live injection, §4.2–4.4).
- [x] `SLDomain` / `public_domain` — table identified [app/models.py:3116-3119]; emptiness verified for Q3 (§4.1).
- [x] `@sl.local` — used as the recipient domain (`xyz@sl.local`) in the Q3 injection (§4.2).
- [x] migrations — run (Q2 §3.1, Q3 §4.1) vs skipped (Q1 §2.1) states all covered.
- [x] Three DB states — empty (§2.1) / migrated+initialized (§3.1) / migrated-but-not-initialized (§4.1) — all provisioned.
- [x] SMTP status codes — **E515** (`550 SL E515 Email not exist`, §4.3) primary + **E207** (`250 SL E207 No bounce report`, §4.5) variant — both enumerated.
- [x] Exact rejection log line(s) — quoted verbatim (§4.4): `alias xyz@sl.local not exist. Try to see if it can be created on the fly` and `alias xyz@sl.local cannot be created on-the-fly, return 550`.
- [x] User's verbatim phrasings preserved in this document: "`python server.py`", "`init_app.py`", "`@sl.local`", and "the SLDomain table should be empty (no email domains configured)".

