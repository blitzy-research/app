# SimpleLogin local/dev startup investigation — branch `app_2cd6ee777f8c`

This document answers three questions about how the SimpleLogin application (a Flask + PostgreSQL email‑aliasing service with an [`aiosmtpd`](https://aiosmtpd.aio-libs.org/)-based inbound SMTP handler) behaves at local/development startup.

Every answer below was produced by **actually building and running the relevant code paths** and capturing the **real output verbatim**. Each captured block shows the exact command (or script) that produced it, the `file:line` source citations that ground it, and the reasoning behind the observed behavior. Where a value cannot be verified by reading or running the code, that is stated explicitly rather than asserted.

The three questions:

1. **Empty‑database startup error** — With PostgreSQL running but the database empty (schema `public` exists, migrations **not** run → no application tables), what **full Python exception** is raised when the server is started with `python server.py` and the login page is then opened?
2. **Required runtime services** — Once database migrations **and** `init_app.py` have run, **which Python services must be running**? Start each and show the exact stdout/stderr confirming it is ready and bound to its port.
3. **Skipped initialization → email rejection** — If migrations run but `init_app.py` is **not** run (so the `SLDomain`/`public_domain` table is empty), what **SMTP status code** does the email handler return to the sender for an inbound message addressed to any `@sl.local` address, and what does it **log** to explain the rejection?

---

## Reproduction environment

### Runtime and pinned versions actually observed

All investigation ran inside the provided Docker image (`simplelogin-dev:local`, derived from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`), with the repository mounted at `/code`. Observed runtime:

```bash
python --version
PGPASSWORD=mypassword psql -h localhost -p 5432 -U myuser -d simplelogin -tAc "show server_version;"
redis-server --version | head -1
python -c "import flask, sqlalchemy, psycopg2, aiosmtpd, gunicorn; \
  print('Flask', flask.__version__); print('SQLAlchemy', sqlalchemy.__version__); \
  print('psycopg2', psycopg2.__version__); print('aiosmtpd', aiosmtpd.__version__); \
  print('gunicorn', gunicorn.__version__)"
```

```text
Python 3.10.18
15.13 (Debian 15.13-0+deb12u1)
Redis server v=7.0.15 sha=00000000:0 malloc=jemalloc-5.3.0 bits=64 build=3f20e06e76a2b578
Flask 1.1.2
SQLAlchemy 1.3.24
psycopg2 2.9.3 (dt dec pq3 ext lo64)
aiosmtpd 1.4.2
gunicorn 20.0.4
```

These match the pins declared in the project manifest: `pyproject.toml:L2` `target-version = ['py310']`, `pyproject.toml:L61` `python = "^3.10"`, `pyproject.toml:L62` `flask = "^1.1.2"`, `pyproject.toml:L66` `gunicorn = "^20.0.4"`, `pyproject.toml:L71` `psycopg2-binary = "^2.9.3"`, `pyproject.toml:L77` `Flask-Migrate = "^2.5.3"`, `pyproject.toml:L87` `aiosmtpd = "^1.2"` (resolved to `1.4.2`), and `pyproject.toml:L116` `SQLAlchemy = "1.3.24"`. The container image base is `Dockerfile:L8` `FROM python:3.10` (an earlier stage builds frontend assets with `Dockerfile:L2` `FROM node:10.17.0-alpine as npm`).

The three behaviors below depend on:

- **SQLAlchemy `1.3.24`** — raises `sqlalchemy.exc.ProgrammingError` against a missing relation (Question 1).
- **psycopg2 `2.9.3`** — the underlying driver that raises `psycopg2.errors.UndefinedTable` (Question 1).
- **aiosmtpd `1.4.2`** — the SMTP server framework whose `handle_DATA` return value becomes the SMTP reply to the sender (Questions 2 and 3).

### Branch confirmation

The rule set mandates the deliverable be named after the **source** branch, `app_2cd6ee777f8c`. The on‑disk destination checkout reports:

```bash
git rev-parse --abbrev-ref HEAD
git rev-parse --short HEAD
```

```text
blitzy-65192d6a-2444-4a53-bf73-6398fde93dec
2cd6ee77
```

The working‑tree branch is the destination branch `blitzy-65192d6a-2444-4a53-bf73-6398fde93dec`; the HEAD commit short hash `2cd6ee77` corresponds to the source branch name `app_2cd6ee777f8c`. Per the naming convention, this document is written to `blitzy/documentation/app_2cd6ee777f8c.md`.

### Working `.env`

The application requires a `.env` file (or `CONFIG=<path>`). An ephemeral working `.env` was derived from `example.env` (this file is git‑ignored and is removed at the end — see the Coverage pass). The keys that matter here, quoted from `example.env`:

- `example.env:L6` `URL=http://localhost:7777`
- `example.env:L22` `EMAIL_DOMAIN=sl.local`
- `example.env:L75` `DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin`
- `example.env:L77` `FLASK_SECRET=secret`

### Database port reconciliation (5432 vs 15432)

There is a genuine port discrepancy in the repository:

- `example.env:L75` uses port **`5432`** (`postgresql://myuser:mypassword@localhost:5432/simplelogin`).
- `scripts/reset_local_db.sh:L3` uses port **`15432`** (`export DB_URI=postgresql://myuser:mypassword@localhost:15432/simplelogin`).

Both instances are actually listening in this environment. To decide which to use, the live sockets were inspected:

```bash
ss -ltnp | grep -E ":5432|:15432"
```

```text
LISTEN 0      244        127.0.0.1:15432      0.0.0.0:*
LISTEN 0      244        127.0.0.1:5432       0.0.0.0:*
```

**Decision: this investigation uses port `5432`** — the `simplelogin` database that the application reads from `.env`/`example.env` (`myuser`/`mypassword`/`simplelogin`). Port `15432`, by contrast, is — in this environment — the **test** PostgreSQL instance, and the two source files that reference it do not agree: `scripts/reset_local_db.sh:L3` points at `myuser:mypassword@localhost:15432/simplelogin` (quoted above), whereas the test config `tests/test.env:L17` uses `test:test@localhost:15432/test`. Probing port `15432` confirms it accepts `test`/`test`/`test` but rejects the reset script's `myuser`/`mypassword`/`simplelogin`:

```bash
PGPASSWORD=test psql -h localhost -p 15432 -U test -d test -tAc "select 1;"
PGPASSWORD=mypassword psql -h localhost -p 15432 -U myuser -d simplelogin -tAc "select 1;"
```

```text
1
psql: error: connection to server at "localhost" (::1), port 15432 failed: FATAL:  password authentication failed for user "myuser"
```

The `scripts/reset_local_db.sh:L3` URI is therefore stale relative to this environment — its `myuser`/`mypassword`/`simplelogin` credentials live only on port `5432` — which is a further reason this investigation used the application's `.env`/`example.env` port `5432`. This discrepancy is **described, not fixed** (the task is read‑only).

### Migration set

The schema is managed by Alembic. The Alembic environment imports the ORM metadata and the DB URI:

- `migrations/env.py:L26` `from app.models import Base`
- `migrations/env.py:L27` `from app.config import DB_URI`

The revision set size (used for grounding the empty‑vs‑migrated boundary):

```bash
ls migrations/versions/*.py | wc -l
```

```text
255
```

Running `alembic upgrade head` brings the schema to head revision `32f25cbf12f6` and creates 77 tables (see Question 2).

### The three database states

The questions require three distinct database states, produced with the same primitives that `scripts/reset_local_db.sh` uses (`scripts/reset_local_db.sh:L4` `drop schema public cascade; create schema public;`, `scripts/reset_local_db.sh:L6` `alembic upgrade head`; note the script prefixes `poetry run`, but this container installs dependencies globally with no Poetry, so `alembic`/`python` are invoked directly):

1. **State 1 (Question 1)** — schema `public` exists, migrations **not** run → no application tables:
   ```bash
   psql "$DB_URI" -c "drop schema public cascade; create schema public;"
   ```
2. **State 2 (Question 2)** — migrations **and** initialization run:
   ```bash
   alembic upgrade head
   python init_app.py
   ```
3. **State 3 (Question 3)** — migrations run **without** initialization (leave `public_domain`/`SLDomain` empty):
   ```bash
   psql "$DB_URI" -c "drop schema public cascade; create schema public;"
   alembic upgrade head
   # NOTE: init_app.py deliberately NOT run
   ```

---

## Answer to Question 1 — Empty-database login exception

**Short answer.** Opening the login page and submitting the form against an empty schema raises **`sqlalchemy.exc.ProgrammingError`**, wrapping **`psycopg2.errors.UndefinedTable`**, with the message **`(psycopg2.errors.UndefinedTable) relation "users" does not exist`**. The missing relation is **`users`**. The exception is triggered by the **POST** of the login form (not the GET render), and — despite `debug=True` — it is surfaced by the global `@app.errorhandler(Exception)` which renders the styled `error/500.html` page and returns **HTTP `500`** to the browser (the interactive Werkzeug debugger does **not** appear).

### Code‑citation chain

1. `server.py:L598-L599` `if __name__ == "__main__":` → `local_main()`.
2. `server.py:L572` `def local_main():`; `server.py:L581` `app.debug = True`; `server.py:L588` `app.run(debug=True, port=7777)` — the dev server on port `7777`.
3. `server.py:L250` `@app.route("/", methods=["GET", "POST"])`; `server.py:L251` `def index()`; `server.py:L255` `return redirect(url_for("auth.login"))` — the index redirects anonymous users to the login route.
4. `server.py:L220` `@login_manager.user_loader`; `server.py:L221` `def load_user(alternative_id)`; `server.py:L222` `user = User.get_by(alternative_id=alternative_id)` — the Flask‑Login user loader runs **only** when a session cookie is present, so an anonymous GET issues no query here.
5. `app/auth/views/login.py:L21` `@auth_bp.route("/login", methods=["GET", "POST"])`; `app/auth/views/login.py:L25` `def login()`; `app/auth/views/login.py:L40` `if form.validate_on_submit():` (False on GET; requires a valid CSRF token on POST); `app/auth/views/login.py:L43` `user = User.get_by(email=email) or User.get_by(email=canonical_email)` — the **first** database query.
6. `app/models.py:L336` `class User(Base, ModelMixin, UserMixin, PasswordOracle):`; `app/models.py:L337` `__tablename__ = "users"` — the relation that does not exist in an empty schema. The query is issued through `app/models.py:L84` `return Session.query(cls).filter_by(**kw).first()` (the `ModelMixin.get_by` helper).
7. `server.py:L388` `@app.errorhandler(Exception)`; `server.py:L389` `def error_handler(e)`; `server.py:L390` `LOG.e(e)`; `server.py:L391` `if request.path.startswith("/api/")`; `server.py:L394` `return render_template("error/500.html"), 500`. Because `app/log.py:L77` aliases `logging.Logger.e = logging.Logger.exception`, `LOG.e(e)` prints the full traceback to stdout. (`app/log.py:L67` also emits the `>>> init logging <<<` print at import.)

### Commands run

State 1 was established, the server started, and the login page exercised:

```bash
export DB_URI="postgresql://myuser:mypassword@localhost:5432/simplelogin"

# State 1: empty schema, migrations NOT run
psql "$DB_URI" -c "drop schema public cascade; create schema public;"
psql "$DB_URI" -c "\dt"          # -> "Did not find any relations."

# Start the dev server (app.run(debug=True, port=7777))
python server.py > /tmp/q1_server.log 2>&1 &

# GET the login form (anonymous -> no DB query), scrape the csrf_token
curl -s -o /dev/null -w "HTTP %{http_code} -> %header{location}\n" -c /tmp/cj.txt "http://localhost:7777/"
curl -s -o /tmp/login_get.html -w "HTTP %{http_code}, bytes=%{size_download}\n" -b /tmp/cj.txt -c /tmp/cj.txt "http://localhost:7777/auth/login"
CSRF=$(grep -oE 'name="csrf_token"[^>]*value="[^"]*"' /tmp/login_get.html | head -1 | sed -E 's/.*value="([^"]*)".*/\1/')

# POST credentials with the valid csrf_token -> triggers User.get_by(email=email)
curl -s -o /tmp/login_post.html \
  -w "HTTP %{http_code}, bytes=%{size_download}, content-type=%header{content-type}\n" \
  -b /tmp/cj.txt -c /tmp/cj.txt \
  --data-urlencode "email=test@example.com" \
  --data-urlencode "password=secretpass123" \
  --data-urlencode "csrf_token=${CSRF}" \
  "http://localhost:7777/auth/login"
```

### Verbatim output — server startup

```text
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/eqxbhrennyhbpukkmtge
Upload files to local dir
>>> init logging <<<
2026-07-01 05:05:05,071 - SL - DEBUG - 1200 - "/code/app/utils.py:17" - <module>() -  - load words file: /code/local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
```

`* Debug mode: on` confirms `app.run(debug=True, …)`. (The startup block prints twice because `debug=True` enables the Werkzeug reloader, which spawns a child process — parent PID 1200, child PID 1214.) Note there is **no** Werkzeug `Running on http://…` banner: SimpleLogin silences the `werkzeug` logger, so readiness is signalled by `* Serving Flask app "server"` plus the listening socket on `7777`.

### Verbatim output — GET vs POST (sub‑part d)

```text
===GET / (root) — redirect to /auth/login===
HTTP 302 -> Location: http://localhost:7777/auth/login

===GET /auth/login — HTTP 200, NO DB query on anonymous GET===
HTTP 200, bytes=335022

===POST /auth/login with credentials (triggers first DB query)===
HTTP 500, bytes=5749, content-type=text/html; charset=utf-8
```

The server's own request log confirms the GET succeeded with no error and only the POST failed:

```text
2026-07-01 05:05:07,845 - SL - DEBUG - 1214 - "/code/server.py:284" - after_request() -  - 127.0.0.1 GET / ImmutableMultiDict([]) 302, takes 0.0009739398956298828
2026-07-01 05:05:07,996 - SL - DEBUG - 1214 - "/code/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.12711119651794434
2026-07-01 05:05:08,024 - SL - DEBUG - 1214 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 500, takes 0.013110160827636719
```

This is direct evidence for the GET‑vs‑POST distinction: the anonymous **GET** of `/auth/login` returns **HTTP 200** (335,022 bytes) with no exception because it renders the form without touching the database; the **POST** returns **HTTP 500** because it is the first request to execute `User.get_by(email=…)`.

### Verbatim output — which surface shows the error (sub‑part e)

The POST response body was inspected for debugger markers:

```text
Werkzeug-debugger-traceback-marker count: 0
werkzeug-string count: 0
```

The body is the branded SimpleLogin 500 page (its `<title>` is `| SimpleLogin` and it loads `dashboard.css`), not the Werkzeug interactive debugger. Even though `debug=True`, the global `@app.errorhandler(Exception)` intercepts the exception first (Flask's `handle_user_exception` finds a registered handler for `Exception` and therefore does not re‑raise into the debugger), logs it via `LOG.e(e)`, and returns `render_template("error/500.html"), 500`.

### Verbatim output — the full exception (sub‑parts a, b, c)

Captured from the server log; the record is emitted at `"/code/server.py:390" - error_handler()` at level `ERROR` (i.e. `LOG.e(e)` → `logging.Logger.exception`, which appends the full traceback):

```text
2026-07-01 05:05:08,018 - SL - ERROR - 1214 - "/code/server.py:390" - error_handler() -  - (psycopg2.errors.UndefinedTable) relation "users" does not exist
LINE 2: FROM users
             ^

[SQL: SELECT users.directory_quota AS users_directory_quota, users.subdomain_quota AS users_subdomain_quota, users.password AS users_password, users.id AS users_id, users.created_at AS users_created_at, users.updated_at AS users_updated_at, users.email AS users_email, users.name AS users_name, users.is_admin AS users_is_admin, users.alias_generator AS users_alias_generator, users.notification AS users_notification, users.activated AS users_activated, users.disabled AS users_disabled, users.profile_picture_id AS users_profile_picture_id, users.otp_secret AS users_otp_secret, users.enable_otp AS users_enable_otp, users.last_otp AS users_last_otp, users.fido_uuid AS users_fido_uuid, users.default_alias_custom_domain_id AS users_default_alias_custom_domain_id, users.default_alias_public_domain_id AS users_default_alias_public_domain_id, users.lifetime AS users_lifetime, users.paid_lifetime AS users_paid_lifetime, users.lifetime_coupon_id AS users_lifetime_coupon_id, users.trial_end AS users_trial_end, users.default_mailbox_id AS users_default_mailbox_id, users.sender_format AS users_sender_format, users.sender_format_updated_at AS users_sender_format_updated_at, users.replace_reverse_alias AS users_replace_reverse_alias, users.referral_id AS users_referral_id, users.intro_shown AS users_intro_shown, users.max_spam_score AS users_max_spam_score, users.newsletter_alias_id AS users_newsletter_alias_id, users.include_sender_in_reverse_alias AS users_include_sender_in_reverse_alias, users.random_alias_suffix AS users_random_alias_suffix, users.expand_alias_info AS users_expand_alias_info, users.ignore_loop_email AS users_ignore_loop_email, users.alternative_id AS users_alternative_id, users.disable_automatic_alias_note AS users_disable_automatic_alias_note, users.one_click_unsubscribe_block_sender AS users_one_click_unsubscribe_block_sender, users.include_website_in_one_click_alias AS users_include_website_in_one_click_alias, users.disable_import AS users_disable_import, users.can_use_phone AS users_can_use_phone, users.phone_quota AS users_phone_quota, users.block_behaviour AS users_block_behaviour, users.include_header_email_header AS users_include_header_email_header, users.enable_data_breach_check AS users_enable_data_breach_check, users.flags AS users_flags, users.unsub_behaviour AS users_unsub_behaviour, users.delete_on AS users_delete_on
FROM users
WHERE users.email = %(email_1)s
 LIMIT %(param_1)s]
[parameters: {'email_1': 'test@example.com', 'param_1': 1}]
(Background on this error at: http://sqlalche.me/e/13/f405)
Traceback (most recent call last):
  File "/usr/local/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1276, in _execute_context
    self.dialect.do_execute(
  File "/usr/local/lib/python3.10/site-packages/sqlalchemy/engine/default.py", line 608, in do_execute
    cursor.execute(statement, parameters)
psycopg2.errors.UndefinedTable: relation "users" does not exist
LINE 2: FROM users
             ^


The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "/usr/local/lib/python3.10/site-packages/flask/app.py", line 1950, in full_dispatch_request
    rv = self.dispatch_request()
  File "/usr/local/lib/python3.10/site-packages/flask_debugtoolbar/__init__.py", line 125, in dispatch_request
    return view_func(**req.view_args)
  File "/usr/local/lib/python3.10/cProfile.py", line 110, in runcall
    return func(*args, **kw)
  File "/usr/local/lib/python3.10/site-packages/flask_limiter/extension.py", line 702, in __inner
    return obj(*a, **k)
  File "/code/app/auth/views/login.py", line 43, in login
    user = User.get_by(email=email) or User.get_by(email=canonical_email)
  File "/code/app/models.py", line 84, in get_by
    return Session.query(cls).filter_by(**kw).first()
  File "/usr/local/lib/python3.10/site-packages/sqlalchemy/orm/query.py", line 3429, in first
    ret = list(self[0:1])
  File "/usr/local/lib/python3.10/site-packages/sqlalchemy/orm/query.py", line 3203, in __getitem__
    return list(res)
  File "/usr/local/lib/python3.10/site-packages/sqlalchemy/orm/query.py", line 3535, in __iter__
    return self._execute_and_instances(context)
  File "/usr/local/lib/python3.10/site-packages/sqlalchemy/orm/query.py", line 3560, in _execute_and_instances
    result = conn.execute(querycontext.statement, self._params)
  File "/usr/local/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1011, in execute
    return meth(self, multiparams, params)
  File "/usr/local/lib/python3.10/site-packages/sqlalchemy/sql/elements.py", line 298, in _execute_on_connection
    return connection._execute_clauseelement(self, multiparams, params)
  File "/usr/local/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1124, in _execute_clauseelement
    ret = self._execute_context(
  File "/usr/local/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1316, in _execute_context
    self._handle_dbapi_exception(
  File "/usr/local/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1510, in _handle_dbapi_exception
    util.raise_(
  File "/usr/local/lib/python3.10/site-packages/sqlalchemy/util/compat.py", line 182, in raise_
    raise exception
  File "/usr/local/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1276, in _execute_context
    self.dialect.do_execute(
  File "/usr/local/lib/python3.10/site-packages/sqlalchemy/engine/default.py", line 608, in do_execute
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
```

(The block above is the **complete, unabridged** `ERROR` record exactly as written to the server log by `LOG.e(e)` — no output is elided or truncated. It includes the full `users.*` column list (all 49 columns, in both the psycopg2 detail and the wrapped SQLAlchemy detail), and the complete chained psycopg2 → SQLAlchemy traceback. The parameters line `[parameters: {'email_1': 'test@example.com', 'param_1': 1}]` shows the submitted email `test@example.com`.)

### Rationale and explicit sub‑part coverage

- **(a) Full exception class path:** `sqlalchemy.exc.ProgrammingError`, chained from (`__cause__`) `psycopg2.errors.UndefinedTable`. The traceback's `The above exception was the direct cause of the following exception:` line shows psycopg2's `UndefinedTable` being wrapped by SQLAlchemy's `ProgrammingError`.
- **(b) Exact message:** `(psycopg2.errors.UndefinedTable) relation "users" does not exist` (followed by `LINE 2: FROM users`).
- **(c) Missing relation:** `users` — from `app/models.py:L337` `__tablename__ = "users"`. The failing SQL is `SELECT … FROM users WHERE users.email = %(email_1)s LIMIT %(param_1)s`.
- **(d) Trigger — POST, not GET:** The anonymous GET render performs no database query (`GET /auth/login … 200`); the first query `User.get_by(email=…)` (`app/auth/views/login.py:L43`) runs only after `form.validate_on_submit()` passes on POST (`app/auth/views/login.py:L40`), which requires a valid CSRF token. The traceback bottoms out at `app/auth/views/login.py`, line 43, confirming the POST path.
- **(e) Surface — `error/500.html`, not the Werkzeug debugger:** The response body contains zero debugger markers and is the styled SimpleLogin 500 page. The global `@app.errorhandler(Exception)` (`server.py:L388-L394`) intercepts and renders `templates/error/500.html`. The traceback also reveals the dev middleware chain installed by `local_main()` (Flask‑DebugToolbar at `flask_debugtoolbar/__init__.py:125`, the cProfile profiler, and Flask‑Limiter at `flask_limiter/extension.py:702` from the `@limiter.limit` decorator on the login view).
- **(f) HTTP status returned to the browser:** `500` (`HTTP 500, bytes=5749, content-type=text/html; charset=utf-8`, corroborated by the `POST /auth/login … 500` request‑log line).

---


## Answer to Question 2 — Required runtime services

**Short answer.** After migrations and `init_app.py` have run, the three Python services that must be running for the system to work are:

1. **The web app** — `python server.py` — Werkzeug dev server bound to **`127.0.0.1:7777`**.
2. **The email handler** — `python email_handler.py` — `aiosmtpd` controller bound to **`0.0.0.0:20381`**.
3. **The job runner** — `python job_runner.py` — a background polling loop that binds **no port** and prints **no readiness banner**.

**PostgreSQL is required**; **Redis is optional** (used only when `MEM_STORE_URI` is set). `wsgi.py` is the production (Gunicorn) entry point, not a separate dev service.

This service set matches the startup order documented in the README: `README.md:L433` `flask db upgrade` → `README.md:L448` `python init_app.py` → the web app on `README.md:L461` (`-p 127.0.0.1:7777:7777`) → `README.md:L480` `python email_handler.py` (`README.md:L477` `-p 127.0.0.1:20381:20381`) → `README.md:L495` `python job_runner.py`.

### State 2 setup (migrations + initialization)

```bash
export DB_URI="postgresql://myuser:mypassword@localhost:5432/simplelogin"
alembic upgrade head
python init_app.py
```

```text
# alembic upgrade head (tail)
INFO  [alembic.runtime.migration] Running upgrade 91ed7f46dc81 -> 7d7b84779837, user_audit_log
INFO  [alembic.runtime.migration] Running upgrade 7d7b84779837 -> 32f25cbf12f6, alias_audit_log_index_created_at

# alembic current
32f25cbf12f6 (head)

# table count in public schema
77

# python init_app.py (tail) — seeds SLDomain and loads PGP keys
2026-07-01 04:28:04,740 - SL - DEBUG - 353 - "/code/init_app.py:36" - load_pgp_public_keys() -  - Finish load_pgp_public_keys
2026-07-01 04:28:04,742 - SL - INFO - 353 - "/code/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain

# public_domain rows after init_app.py
1
sl.local
```

`init_app.py` (`init_app.py:L73` `add_sl_domains()`, defined at `init_app.py:L39`) seeds one `SLDomain` row: `sl.local`. This is the state assumed by Question 2 (and the row that Question 3 deliberately leaves absent).

### Service 1 — Web app (`python server.py`)

```bash
python server.py > /tmp/q2_web.log 2>&1 &
ss -ltnp | grep 7777
```

Verbatim readiness output:

```text
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/pvejhybsrwwaafyznubr
Upload files to local dir
>>> init logging <<<
2026-07-01 04:28:16,772 - SL - DEBUG - 374 - "/code/app/utils.py:17" - <module>() -  - load words file: /code/local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
```

Port‑binding proof:

```text
LISTEN 0      128        127.0.0.1:7777       0.0.0.0:*    users:(("python",pid=394,fd=6),("python",pid=394,fd=5),("python",pid=374,fd=5))
```

The web app binds `127.0.0.1:7777` (localhost only). The readiness signal is `* Serving Flask app "server"` / `* Debug mode: on` plus the LISTEN socket; there is no Werkzeug `Running on …` banner because the `werkzeug` logger is silenced. The dev entry point is `server.py:L588` `app.run(debug=True, port=7777)` inside `local_main()`.

### Service 2 — Email handler (`python email_handler.py`)

```bash
python email_handler.py > /tmp/q2_email.log 2>&1 &
ss -ltnp | grep 20381
```

Verbatim readiness output (the two readiness log lines):

```text
2026-07-01 04:28:17,587 - SL - INFO - 375 - "/code/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-01 04:28:17,589 - SL - DEBUG - 375 - "/code/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

Port‑binding proof:

```text
LISTEN 0      100          0.0.0.0:20381      0.0.0.0:*    users:(("python",pid=375,fd=8))
```

These readiness lines are emitted by `email_handler.py:L2403` `LOG.i("Listen for port %s", args.port)` and `email_handler.py:L2386` `LOG.d("Start mail controller %s %s", controller.hostname, controller.port)`. The default port `20381` is set at `email_handler.py:L2399` (argparse `default=20381`), and the controller is built at `email_handler.py:L2383` `controller = Controller(MailHandler(), hostname="0.0.0.0", port=port)` and started at `email_handler.py:L2385` `controller.start()` inside `email_handler.py:L2381` `def main(port: int):`. The handler binds `0.0.0.0:20381` (all interfaces) and runs as a single process (no reloader).

### Service 3 — Job runner (`python job_runner.py`)

```bash
python job_runner.py > /tmp/q2_jobrunner.log 2>&1 &
ss -ltnp        # (no new LISTEN socket appears for the job runner)
```

Verbatim startup output — **only** the import‑time prints; there is no readiness banner and no `Take job` line (which would appear only when a job exists):

```text
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/lefzsmbrqcodxtzrqrbh
Upload files to local dir
>>> init logging <<<
2026-07-01 04:28:16,693 - SL - DEBUG - 376 - "/code/app/utils.py:17" - <module>() -  - load words file: /code/local_data/test_words.txt
```

**The job runner binds no port and exposes no "ready to accept connections" banner.** Its readiness is simply entering the polling loop `job_runner.py:L330` `while True:`, wrapped in `job_runner.py:L332` `with create_light_app().app_context():`, iterating `job_runner.py:L333` `for job in get_jobs_to_run():` (defined at `job_runner.py:L307`) — logging `job_runner.py:L334` `LOG.d("Take job %s", job)` only when a job exists — and then `job_runner.py:L347` `time.sleep(10)`. The process (PID 376) is alive but does not appear in `ss -ltnp`.

### Combined socket snapshot

A single `ss -ltnp` while all three services were running:

```text
State  Recv-Q Send-Q Local Address:Port  Peer Address:Port Process
LISTEN 0      244        127.0.0.1:15432      0.0.0.0:*
LISTEN 0      244        127.0.0.1:5432       0.0.0.0:*
LISTEN 0      128        127.0.0.1:7777       0.0.0.0:*    users:(("python",pid=394,fd=6),("python",pid=394,fd=5),("python",pid=374,fd=5))
LISTEN 0      511          0.0.0.0:6379       0.0.0.0:*    users:(("redis-server",pid=79,fd=6))
LISTEN 0      100          0.0.0.0:20381      0.0.0.0:*    users:(("python",pid=375,fd=8))
LISTEN 0      244            [::1]:5432          [::]:*
LISTEN 0      244            [::1]:15432         [::]:*
LISTEN 0      511             [::]:6379          [::]:*    users:(("redis-server",pid=79,fd=7))
```

The web app (`pid=394`/`374`) owns `127.0.0.1:7777`, the email handler (`pid=375`) owns `0.0.0.0:20381`, and the job runner (`pid=376`) owns **no** socket. PostgreSQL (`5432`, `15432`) and Redis (`6379`) are infrastructure, not application services.

### Supporting facts (sub‑part e)

- **PostgreSQL required.** All three questions require a reachable PostgreSQL; it is listening on `5432` here.
- **Redis optional.** Redis is used only when `MEM_STORE_URI` is set. In this local run it is unset:
  ```text
  (MEM_STORE_URI not set in .env => Redis NOT used by app)
  ```
  The gate is `server.py:L163` `if MEM_STORE_URI:` → `server.py:L164` sets the Flask‑Limiter storage URL → `server.py:L165` `initialize_redis_services(app, MEM_STORE_URI)`. With `MEM_STORE_URI` unset, none of this runs and Redis is unused.
- **`wsgi.py` is the production variant.** `wsgi.py:L1` `from server import create_app` and `wsgi.py:L3` `app = create_app()`. This is the object Gunicorn serves in production; it is not a separate dev service and is not started by `python server.py` (which uses `local_main()` + `app.run(...)`).

### Explicit sub‑part coverage

- **(a) Enumerate each required service:** web app (`python server.py`), email handler (`python email_handler.py`), job runner (`python job_runner.py`).
- **(b) Verbatim readiness output:** captured for each service above.
- **(c) Port binding verified:** `127.0.0.1:7777` (web) and `0.0.0.0:20381` (email handler) proven via `ss -ltnp`.
- **(d) Job runner binds no port / no banner:** stated explicitly and proven by the job runner's absence from `ss -ltnp`.
- **(e) PostgreSQL required / Redis optional / `wsgi.py` production:** documented above.

---


## Answer to Question 3 — Skipped init and `@sl.local` rejection

**Short answer.** The email handler returns **`550 SL E515 Email not exist`** to the sender, and it logs (verbatim) `alias anything@sl.local cannot be created on-the-fly, return 550`. The rejection is **not** caused directly by the empty `SLDomain`/`public_domain` table; it is caused because **no matching `Alias` row exists and on‑the‑fly auto‑creation fails** (both the custom‑domain and the directory sub‑checks fail).

### State 3 setup (migrated, `init_app.py` NOT run)

```bash
export DB_URI="postgresql://myuser:mypassword@localhost:5432/simplelogin"
psql "$DB_URI" -c "drop schema public cascade; create schema public;"
alembic upgrade head          # head 32f25cbf12f6, 77 tables
# init_app.py deliberately NOT run
psql "$DB_URI" -tAc "select count(*) from public_domain;"
```

```text
0
```

The `public_domain` table (the `SLDomain` model, `app/models.py:L3116` `class SLDomain(Base, ModelMixin):`, `app/models.py:L3119` `__tablename__ = "public_domain"`) is empty because `init_app.py:L73` `add_sl_domains()` was not run.

### The email handler was started, then a plain message injected

The handler was started exactly as in Question 2 (`Listen for port 20381`, `Start mail controller 0.0.0.0 20381`, LISTEN on `0.0.0.0:20381`). A plain inbound message was injected with Python's standard‑library `smtplib` (no new dependency). The message uses a **non‑bounce** `MAIL FROM` (`sender@example.com`) and carries **no** `X-Spamd-Result` header — both deliberate, to obtain a clean `E515` (see the caveats below):

```python
import smtplib

HOST, PORT = "127.0.0.1", 20381
MAIL_FROM = "sender@example.com"          # non-bounce sender (avoids E207)
RCPT_TO   = "anything@sl.local"           # unknown @sl.local recipient
# PLAIN message, NO X-Spamd-Result header (avoids the E216 SPF override)
MSG = (
    "From: sender@example.com\r\n"
    "To: anything@sl.local\r\n"
    "Subject: Q3 plain injection test\r\n"
    "Message-ID: <q3-test@example.com>\r\n"
    "\r\n"
    "This is a plain test body with no spam header.\r\n"
)

client = smtplib.SMTP(HOST, PORT, timeout=30)
client.set_debuglevel(1)                  # echo the raw SMTP dialog
client.ehlo("test.client")
client.docmd("MAIL", "FROM:<" + MAIL_FROM + ">")
client.docmd("RCPT", "TO:<" + RCPT_TO + ">")
client.docmd("DATA")
client.send(MSG.encode() + b".\r\n")
code, msg = client.getreply()             # the end-of-DATA reply == the handler's return value
print("SMTP_CODE=" + str(code))
print("SMTP_MESSAGE=" + msg.decode(errors="replace"))
client.quit()
```

### Verbatim output — the SMTP reply the sender receives (sub‑part a)

```text
send: 'MAIL FROM:<sender@example.com>\r\n'
reply: b'250 OK\r\n'
send: 'RCPT TO:<anything@sl.local>\r\n'
reply: b'250 OK\r\n'
send: 'DATA\r\n'
reply: b'354 End data with <CR><LF>.<CR><LF>\r\n'
send: b'From: sender@example.com\r\nTo: anything@sl.local\r\nSubject: Q3 plain injection test\r\nMessage-ID: <q3-test@example.com>\r\n\r\nThis is a plain test body with no spam header.\r\n.\r\n'
reply: b'550 SL E515 Email not exist\r\n'
=== FINAL END-OF-DATA REPLY (what the SENDER receives) ===
SMTP_CODE=550
SMTP_MESSAGE=SL E515 Email not exist
COMBINED=550 SL E515 Email not exist
```

The recipient is **accepted at `RCPT TO` with `250 OK`**; the rejection arrives only at the **end of `DATA`** as `550 SL E515 Email not exist`. This is the exact literal defined at `app/email/status.py:L51`:

```text
E515 = "550 SL E515 Email not exist"
```

### Verbatim output — what the handler logs (sub‑part b) and the causal chain (sub‑part c)

The handler's log for this transaction, in order:

```text
2026-07-01 05:16:47,244 - SL - DEBUG - 1404 - "/code/app/log.py:24" - set_message_id() -  - set message_id 5e00dbec-14af-475c-b0f2-2c9150d0277b
2026-07-01 05:16:47,244 - SL - DEBUG - 1404 - "/code/email_handler.py:2342" - _handle() - 5e00dbec-14af-475c-b0f2-2c9150d0277b - ====>=====>====>====>====>====>====>====>
2026-07-01 05:16:47,244 - SL - INFO - 1404 - "/code/email_handler.py:2343" - _handle() - 5e00dbec-14af-475c-b0f2-2c9150d0277b - New message, mail from sender@example.com, rctp tos ['anything@sl.local']
2026-07-01 05:16:47,245 - SL - INFO - 1404 - "/code/email_handler.py:1956" - handle() - 5e00dbec-14af-475c-b0f2-2c9150d0277b - Set CONTENT_TRANSFER_ENCODING
2026-07-01 05:16:47,246 - SL - DEBUG - 1404 - "/code/email_handler.py:1963" - handle() - 5e00dbec-14af-475c-b0f2-2c9150d0277b - Cannot parse Postfix queue ID from None None
2026-07-01 05:16:47,387 - SL - DEBUG - 1404 - "/code/email_handler.py:1980" - handle() - 5e00dbec-14af-475c-b0f2-2c9150d0277b - ==>> Handle mail_from:sender@example.com, rcpt_tos:['anything@sl.local'], header_from:sender@example.com, header_to:anything@sl.local, cc:None, reply-to:None, message_id:<q3-test@example.com>, client_ip:None, headers:[('From', 'sender@example.com'), ('To', 'anything@sl.local'), ('Subject', 'Q3 plain injection test'), ('Message-ID', '<q3-test@example.com>'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-01 05:16:47,392 - SL - DEBUG - 1404 - "/code/email_handler.py:2202" - handle() - 5e00dbec-14af-475c-b0f2-2c9150d0277b - Forward phase sender@example.com(sender@example.com) -> anything@sl.local
2026-07-01 05:16:47,402 - SL - DEBUG - 1404 - "/code/email_handler.py:545" - handle_forward() - 5e00dbec-14af-475c-b0f2-2c9150d0277b - alias anything@sl.local not exist. Try to see if it can be created on the fly
2026-07-01 05:16:47,416 - SL - INFO - 1404 - "/code/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - 5e00dbec-14af-475c-b0f2-2c9150d0277b - Cannot auto-create custom domain alias for anything@sl.local because there's no custom domain for sl.local
2026-07-01 05:16:47,416 - SL - INFO - 1404 - "/code/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - 5e00dbec-14af-475c-b0f2-2c9150d0277b - Cannot auto-create anything@sl.local since it has no directory separator
2026-07-01 05:16:47,416 - SL - DEBUG - 1404 - "/code/email_handler.py:551" - handle_forward() - 5e00dbec-14af-475c-b0f2-2c9150d0277b - alias anything@sl.local cannot be created on-the-fly, return 550
2026-07-01 05:16:47,417 - SL - INFO - 1404 - "/code/email_handler.py:2367" - _handle() - 5e00dbec-14af-475c-b0f2-2c9150d0277b - Finish mail_from sender@example.com, rcpt_tos ['anything@sl.local'], takes 0.17296862602233887 seconds with return code '550 SL E515 Email not exist'<<===
```

These lines make the **true causal chain** explicit, and it is **not** "empty `SLDomain` → reject". The chain is:

1. The message enters the forward phase: `email_handler.py:L2202` (`handle()`), then `email_handler.py:L536` `def handle_forward(...)`.
2. `email_handler.py:L543` `alias = Alias.get_by(email=alias_address)` returns `None` → `email_handler.py:L545` logs `alias anything@sl.local not exist. Try to see if it can be created on the fly`.
3. `email_handler.py:L549` `alias = try_auto_create(alias_address)` (`app/alias_utils.py:L202`) returns `None` because **both** sub‑checks fail:
   - **Custom‑domain check** (`app/alias_utils.py:L92` `check_if_alias_can_be_auto_created_for_custom_domain`, via `try_auto_create_via_domain` at `app/alias_utils.py:L274`) logs at `app/alias_utils.py:L104`: `Cannot auto-create custom domain alias for anything@sl.local because there's no custom domain for sl.local`.
   - **Directory check** (`app/alias_utils.py:L145` `check_if_alias_can_be_auto_created_for_a_directory`, via `try_auto_create_directory` at `app/alias_utils.py:L227`) logs at `app/alias_utils.py:L165`: `Cannot auto-create anything@sl.local since it has no directory separator`. A plain address such as `anything@sl.local` contains none of the directory separators (`/`, `+`, `#`), even though `sl.local` **is** in `config.ALIAS_DOMAINS` (`app/config.py:L160` `ALIAS_DOMAINS = OTHER_ALIAS_DOMAINS + [EMAIL_DOMAIN]`, with `EMAIL_DOMAIN=sl.local` from `app/config.py:L92`; the directory‑domain membership is checked by `app/email_utils.py:L548` `for domain in config.ALIAS_DOMAINS:` inside `app/email_utils.py:L545` `def can_create_directory_for_address(...)`).
4. With auto‑creation failed, `email_handler.py:L551` logs `alias anything@sl.local cannot be created on-the-fly, return 550` and `email_handler.py:L555` `return [(False, status.E515)]`.
5. `_handle` records the outcome at `email_handler.py:L2367`: `Finish … with return code '550 SL E515 Email not exist'<<===`.

The role of the empty `SLDomain`/`public_domain` table is indirect: it governs which domains the **web UI** offers for alias creation. With it empty, users cannot create `@sl.local` aliases through the UI, so no `Alias` rows exist for `@sl.local`, so inbound mail to any such address finds no alias and cannot auto‑create one — hence `E515`.

### The `aiosmtpd` contract (sub‑part d)

That the `550` string is delivered to the sending client follows from the `aiosmtpd` handler‑hook contract. Per the `aiosmtpd` documentation, a handler that implements a hook "assumes responsibility for the status messages returned to the client"; the hook's return value **is** the SMTP response, and if the hook returns `None` or raises, the framework logs the error and returns a `451` instead. The `Controller` "creates a TCP‑based server, listening on an `ip_address:port` pair." In SimpleLogin, `email_handler.py:L2289` `async def handle_DATA(self, server, session, envelope)` returns the value produced by `email_handler.py:L2335` `def _handle(...)`, which returns the value from the module‑level `email_handler.py:L1945` `def handle(envelope, msg) -> str:` (`email_handler.py:L2233` `return res[0][1]`) — i.e. the `status.E515` string bubbles all the way out and becomes the end‑of‑DATA reply. This is exactly what the raw wire capture above shows (`reply: b'550 SL E515 Email not exist\r\n'`).

### Caveats — E216 / E207 / E404 (sub‑part e)

- **E216 (SPF override) did not fire — verified.** `_handle` contains a branch (`email_handler.py:L2357` `if return_status[0] == "5":`) that replaces any 5xx with `status.E216 = "250 SL E216 Handled spf policy"` (`app/email/status.py:L24`), but only when a `SpamdResult` with a failing SPF check is present (`email_handler.py:L2362-L2365`). The injected plain message has no `X-Spamd-Result` header, so `SpamdResult.extract_from_headers` returns `None` (`app/handler/spamd_result.py:L83` `spam_result_header = msg.get_all(headers.SPAMD_RESULT)`, `app/handler/spamd_result.py:L84-L85` `if not spam_result_header: return None`; the header name is `app/email/headers.py:L16` `SPAMD_RESULT = "X-Spamd-Result"`). The override branch therefore did not run, confirmed by the log:
  ```text
  Replacing-5XX-to-216 log count: 0
  ```
  Had the message been routed through a full Postfix→SpamAssassin chain with an SPF‑failing return‑path, the reply could instead have been `250 SL E216 Handled spf policy`.
- **E207 (ignored bounce) avoided by design.** `handle_forward` returns `status.E207 = "250 SL E207 No bounce report"` (`app/email/status.py:L12`) at `email_handler.py:L552-L553` **instead** of `E515` when `should_ignore_bounce(envelope.mail_from)` is true (`app/email_utils.py:L1361`). That requires an `IgnoreBounceSender` row (none exist here), and a non‑bounce sender `sender@example.com` was used, so this branch did not fire.
- **E404 (unexpected error) not triggered.** `email_handler.py:L2332` returns `status.E404 = "421 SL E404 Unexpected error - Retry later"` (`app/email/status.py:L32`) only if `handle_DATA` hits an unhandled exception. None occurred; the observed reply was the clean `550 SL E515 Email not exist`.
- Note also that `E502`/`E508`/`E515` share the identical text `550 SL E… Email not exist` but different codes (`app/email/status.py:L39`, `L45`, `L51`); the forward path returns specifically **`E515`**, as the `Finish … with return code '550 SL E515 Email not exist'` log confirms.

### Explicit sub‑part coverage

- **(a) SMTP status returned to the sender:** `550 SL E515 Email not exist` (raw wire `reply: b'550 SL E515 Email not exist\r\n'`; `SMTP_CODE=550`).
- **(b) What it logs:** `alias anything@sl.local cannot be created on-the-fly, return 550` (`email_handler.py:L551`), preceded by `alias anything@sl.local not exist. Try to see if it can be created on the fly` (`email_handler.py:L545`) and followed by `Finish … with return code '550 SL E515 Email not exist'<<===` (`email_handler.py:L2367`).
- **(c) True causal chain:** no `Alias` row + `try_auto_create()` fails on both the custom‑domain and directory sub‑checks (see the two `alias_utils.py` log lines). Not "empty `SLDomain` → reject" directly.
- **(d) SMTP‑framework contract:** the `aiosmtpd` `handle_DATA` return value is the reply to the sender; verified by the raw wire capture.
- **(e) Caveats and observed reply:** the E216 override did not fire (no `X-Spamd-Result`), E207/E404 were avoided; the actually‑observed reply is `550 SL E515 Email not exist`.

---


## Coverage pass

Re‑reading each question and confirming every sub‑part is answered from observed output:

**Question 1 — empty‑database login exception**

- [x] (a) Full exception class path — `sqlalchemy.exc.ProgrammingError` wrapping `psycopg2.errors.UndefinedTable`.
- [x] (b) Exact error message — `(psycopg2.errors.UndefinedTable) relation "users" does not exist`.
- [x] (c) Missing relation — `users` (`app/models.py:L337`).
- [x] (d) GET vs POST trigger — GET `/auth/login` returns HTTP 200 with no DB query; the POST triggers `User.get_by(email=…)` (`app/auth/views/login.py:L43`). Both captured.
- [x] (e) Surface — the global `@app.errorhandler(Exception)` renders `templates/error/500.html` (zero Werkzeug‑debugger markers in the body), not the interactive debugger.
- [x] (f) HTTP status returned to the browser — `500`.

**Question 2 — required runtime services**

- [x] (a) Services enumerated — web app (`server.py`), email handler (`email_handler.py`), job runner (`job_runner.py`).
- [x] (b) Verbatim readiness output — captured for all three.
- [x] (c) Port binding verified — `127.0.0.1:7777` (web) and `0.0.0.0:20381` (email handler) via `ss -ltnp`.
- [x] (d) Job runner binds no port / no banner — stated explicitly and proven by its absence from `ss -ltnp`.
- [x] (e) PostgreSQL required / Redis optional (`MEM_STORE_URI`) / `wsgi.py` production variant — documented.

**Question 3 — skipped init → `@sl.local` rejection**

- [x] (a) SMTP status returned to the sender — `550 SL E515 Email not exist`.
- [x] (b) Rejection log line — `alias anything@sl.local cannot be created on-the-fly, return 550` (`email_handler.py:L551`), with surrounding lines.
- [x] (c) True causal chain — no `Alias` row + `try_auto_create()` fails on both sub‑checks (custom‑domain and directory), not "empty `SLDomain` → reject".
- [x] (d) `aiosmtpd` contract — the `handle_DATA` return value is the reply to the sender; confirmed by the raw wire capture.
- [x] (e) E216/E207/E404 caveats — E216 override did not fire (no `X-Spamd-Result`, `Replacing‑5XX` count 0); E207/E404 avoided; observed reply is `E515`.

### Repository left unchanged except the new document

All ephemeral artifacts (the working `.env`, the temporary observation logs, and the `smtplib` injection script) were removed after capture. Because the `blitzy/` directory is itself new/untracked, the default `git status --porcelain` collapses it to a single directory entry; expanding untracked files with `-uall` shows the one and only added path is this document:

```bash
git status --porcelain          # default collapses the new untracked dir
git status --porcelain -uall     # expand every untracked file
```

```text
?? blitzy/

?? blitzy/documentation/app_2cd6ee777f8c.md
```

No tracked repository file was modified or deleted (`git status --porcelain` lists no `M`/`D`/`A` entries); the only change is the addition of this document.

