# SimpleLogin — First-Time Local Bring-Up: Observed Runtime Behavior (Q1–Q3)

This document answers three concrete runtime-behavior questions about the **SimpleLogin** Flask
monolith, based on **actually running the stack** and capturing verbatim output. It is a read-only
investigation: no source file was modified or deleted, and the only permanent artifact is this
document.

Each behavioral claim below is paired with the specific observed output line that demonstrates it and
the command or code that produced it, followed by an exact `file:line` citation into the source tree.

## How this was observed

**Runtime and versions**

- **Python 3.10.18** (observed: `python --version` → `Python 3.10.18`). The project requires Python
  3.10 and explicitly notes that 3.12 does not work:
  <br>`CONTRIBUTING.md:236` → `# we haven't managed to make python 3.12 work`
  <br>Corroborated by `pyproject.toml:61` → `python = "^3.10"`, `Dockerfile:8` → `FROM python:3.10`,
  and `.github/workflows/main.yml:40` → `python-version: ["3.10"]`.
- **PostgreSQL 15.13** (observed: `show server_version;` → `15.13 (Debian 15.13-0+deb12u1)`). This is
  newer than the documented CI image `postgres:13` (`.github/workflows/main.yml:47` → `image:
  postgres:13`) but satisfies the app's `>= 13` requirement; no PostgreSQL-15-specific behavior affects
  Q1–Q3. PostgreSQL is a hard dependency — the app uses PostgreSQL-specific features, so SQLite is not a
  substitute.
- Observation environment: the provided Docker container with a pre-provisioned virtualenv at
  `/app/venv` (all `poetry.lock` versions) and environment variables sourced from the container's
  `/build.sh` (`DB_URI=postgresql://test:test@localhost:5432/test`, `EMAIL_DOMAIN=sl.local`
  matching `example.env:22`, `FLASK_SECRET=secret`). No `.env` file was created; the repo's `.env` is
  gitignored anyway (`.gitignore:4` → `.env`).
- Behavior-critical dependencies at the exact `poetry.lock` versions: Flask 1.1.2, Werkzeug 1.0.1,
  SQLAlchemy 1.3.24, psycopg2-binary 2.9.3, Flask-Login 0.5.0, Flask-Migrate 2.5.3, alembic 1.4.3,
  aiosmtpd 1.4.2, dnspython 2.0.0, email-validator 1.1.3.

**Three database states**

The three questions require three distinct database states, staged by resetting between runs with
`drop schema public cascade; create schema public;` (the reset pattern used by
`scripts/reset_local_db.sh:4`):

| State | How reached | Used for |
|-------|-------------|----------|
| A — empty / un-migrated | schema present, **no tables** | Q1 |
| B — migrated + initialized | `alembic upgrade head` **+** `python init_app.py` | Q2 |
| C — migrated only | `alembic upgrade head` **alone** (skip `init_app.py`) | Q3 |

**Honest deviations (stated for transparency; none affects the Q1/Q2/Q3 outcomes)**

- (a) The observation environment is the provided Docker container, whose PostgreSQL is **15.13** rather
  than the documented `postgres:13` (`.github/workflows/main.yml:47`); it satisfies the app's `>= 13`
  requirement and no PostgreSQL-15-specific behavior is exercised by the three answers.
- (b) Migrations were applied with **`alembic upgrade head`**. In this environment `flask db upgrade`
  (`README.md:433`) fails with `KeyError: 'migrate'` (the Flask-Migrate CLI is not registered on the
  app), so the underlying Alembic command — which applies the identical migration set — was used
  directly (this is also the command the contributor guide uses, `CONTRIBUTING.md:106`).
- (c) The database is the container's pre-provisioned `test` database
  (`DB_URI=postgresql://test:test@localhost:5432/test`), not a hand-created `simplelogin` DB; this only
  changes the DB name/credentials shown in observed commands, not any Q1/Q2/Q3 outcome.

**Cleanup**

All reproduction artifacts are ephemeral and outside the repository: the observation ran inside the
provided Docker container using its pre-provisioned `/app/venv`, and a temporary SMTP-injection script
(`/tmp/q3_inject.py`) was used and removed. The `server.py`, `email_handler.py`, and `job_runner.py`
processes started for the observations were stopped afterward. No file in the source tree was modified;
`git status` shows only the new `blitzy/documentation/app_2cd6ee777f8c.md`.

---

## Q1 — Empty-database login failure

**Question.** With PostgreSQL running and an empty database (schema present, **no tables** — Alembic
migrations **not** applied), start `python server.py`, open the login page, and capture the full
verbatim Python exception/traceback.

### Precondition

- **Empty DB confirmed.** The `public` schema was reset with `drop schema public cascade; create schema public;` (the reset pattern of `scripts/reset_local_db.sh`), so the *database* exists but has **no tables**. Verified verbatim immediately before launching the server:

```
$ PGPASSWORD=test psql -h localhost -U test -d test -c "\dt"
Did not find any relations.
$ PGPASSWORD=test psql -h localhost -U test -d test -tAc "select count(*) from information_schema.tables where table_schema='public';"
0
```

- **App import and `create_app()` SUCCEED on the empty DB.** This is because `app/db.py` opens the
  connection at **import time**, and the *database* itself exists (only the tables are missing):

```
# app/db.py:9-14
engine = create_engine(
    config.DB_URI, connect_args={"application_name": config.DB_CONN_NAME}
)
connection = engine.connect()

Session = scoped_session(sessionmaker(bind=connection))
```

  The import-time connect is at `app/db.py:12` (`connection = engine.connect()`); the
  `connect_args={"application_name": config.DB_CONN_NAME}` is at `app/db.py:10`.
- **Server launched via `python server.py`.** The module's `__main__` guard runs `local_main()`
  (`server.py:598` `if __name__ == "__main__":`). `local_main()` (`server.py:572`) enables the
  Flask-DebugToolbar **profiler** (`server.py:579` `app.config["DEBUG_TB_PROFILER_ENABLED"] = True`;
  `server.py:582` `DebugToolbarExtension(app)`), sets `app.debug = True` (`server.py:581`), and calls
  `app.run(debug=True, port=7777)` (`server.py:588`). The profiler is why the traceback below contains
  `flask_debugtoolbar` and `cProfile` frames.

### Observed HTTP behavior (one claim -> one evidence)

Requests were issued with `curl` sharing one cookie jar, so the Flask-WTF CSRF token rendered into the
GET form validates on the POST (otherwise `validate_on_submit()` returns `False` and line 43 never runs).

- **`GET /` -> `HTTP/1.0 302 FOUND`**, redirecting anonymous visitors to `/auth/login`. Verbatim
  response headers from `curl -s -D - -o /dev/null -c cookies.txt http://127.0.0.1:7777/`:

```
HTTP/1.0 302 FOUND
Content-Type: text/html; charset=utf-8
Content-Length: 229
Location: http://127.0.0.1:7777/auth/login
Set-Cookie: slapp=58abf08f-453f-4ce5-8d0f-ff2dd36d310f.ZbmmD1e8ip_D3_8sBfd-4rI8sFk; Expires=Wed, 08-Jul-2026 23:25:54 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Wed, 01 Jul 2026 23:25:54 GMT
```

  Source: `server.py:251` `def index():` -> `server.py:255` `return redirect(url_for("auth.login"))`.
  The `Server:` header confirms Werkzeug 1.0.1 on Python 3.10.18.

- **`GET /auth/login` -> HTTP 200, no DB query, no error.** **KEY FINDING: opening the
  login page does NOT raise on an empty DB** — the anonymous form renders fine. Command
  `curl -s -b cookies.txt -c cookies.txt -o login.html -w 'HTTP %{http_code}, %{size_download} bytes\n' http://127.0.0.1:7777/auth/login`:

```
HTTP 200, 342155 bytes
```

  The invariant, reproducible signal here is the **`HTTP 200`** (the GET renders without touching the
  database — any query against the empty DB would raise `UndefinedTable`, as the POST below does). The
  `size_download` value is **not** deterministic and must not be read as a fixed page size: because
  `local_main()` enables the Flask-DebugToolbar profiler under `debug=True` (`server.py:579`
  `app.config["DEBUG_TB_PROFILER_ENABLED"] = True`, `server.py:582` `DebugToolbarExtension(app)`), the
  returned HTML embeds the toolbar's profiler panel whose per-request `CPU:`/`View:` timing values
  differ on every request, and the first (cold) request after startup is substantially larger than warm
  ones. Across repeated fetches the same page measured ~214,300–215,200 bytes on warm requests and
  ~342,000 bytes on the first cold request after startup; only the `HTTP 200` (and the absence of any
  `UndefinedTable` error) is stable. (By contrast, the POST 500 body below is a stable 5744 bytes,
  because the debug toolbar is not injected into the error response.)

  Reason: the `auth_bp` blueprint has no `before_request` hook (`app/auth/base.py:1-5`), and the
  template `templates/auth/login.html` carries no anonymous DB-backed data.

- **`POST /auth/login`** (form fields `csrf_token` + `email=john@wick.com` + `password=password`) **->
  HTTP 500.** **This is the request that triggers the exception** — `validate_on_submit()` passes, so
  `User.get_by(email=email)` runs against the missing `users` table:

```
HTTP 500
```

- **Server-side request log for all three requests.** The `werkzeug` request logger is disabled
  (`app/log.py:70-71` `log = logging.getLogger("werkzeug")` / `log.disabled = True`), so these lines
  are emitted by the app's own `after_request` (`server.py:284`):

```
2026-07-01 23:25:54,089 - SL - DEBUG - 2720 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET / ImmutableMultiDict([]) 302, takes 0.0007202625274658203
2026-07-01 23:25:54,203 - SL - DEBUG - 2720 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.10338449478149414
2026-07-01 23:25:54,234 - SL - DEBUG - 2720 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 500, takes 0.015875577926635742
```

### The exact exception — full verbatim traceback (captured from the `python server.py` console)

The exception is emitted by the app's generic error handler `error_handler()` (`server.py:389`), which
calls `LOG.e(e)` (`server.py:390`). `LOG.e` is mapped to `logging.Logger.exception` (`app/log.py:77`
`logging.Logger.e = logging.Logger.exception`), so the **complete chained traceback** is printed to the
console. The following is the entire block exactly as observed — no frames omitted, nothing elided,
SQL and parameters unabridged:

```
2026-07-01 23:25:54,227 - SL - ERROR - 2720 - "/app/server.py:390" - error_handler() -  - (psycopg2.errors.UndefinedTable) relation "users" does not exist
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
[parameters: {'email_1': 'john@wick.com', 'param_1': 1}]
(Background on this error at: http://sqlalche.me/e/13/f405)
```

### Answer

- **Exception class = `sqlalchemy.exc.ProgrammingError`** wrapping **`psycopg2.errors.UndefinedTable`**,
  message **`relation "users" does not exist`**.
- **Triggered on the POST** login submission (not the GET). The failing ORM call is at
  `app/auth/views/login.py:43`
  (`user = User.get_by(email=email) or User.get_by(email=canonical_email)`), which delegates to
  `app/models.py:84` (`return Session.query(cls).filter_by(**kw).first()`). That call sits inside
  `if form.validate_on_submit():` (`app/auth/views/login.py:40`), i.e. the form-submit / POST path.
- The `http://sqlalche.me/e/13/f405` URL in the message confirms **SQLAlchemy 1.3** (installed 1.3.24).
- **Correction to a natural assumption — the generic error handler is NOT bypassed under `debug=True`.**
  As observed, `@app.errorhandler(Exception)` (`server.py:388`) -> `error_handler()` (`server.py:389`)
  **runs**: it logs the full traceback via `LOG.e(e)` (`server.py:390`) and then returns
  `render_template("error/500.html"), 500` (`server.py:394`). The POST response body was the branded
  SimpleLogin 500 page (5744 bytes), **not** the interactive Werkzeug debugger. The full Python
  traceback reaches the console precisely because `LOG.e` = `logging.Logger.exception`
  (`app/log.py:77`), while the `werkzeug` logger is disabled (`app/log.py:70-71`).
- **Precise nuance:** an empty database does **not** break app import or the login-page GET; it breaks
  the **first ORM query against the missing `users` table**, which happens when credentials are
  submitted.

---

## Q2 — Required Python services and startup / port-binding evidence

**Question.** After `flask db upgrade` and `python init_app.py` (system correctly set up), determine
which Python services must be running, start each, and capture the exact stdout/stderr startup output
confirming each is running and ready — including proof that services actually bind their ports.

### Setup (State B — migrated + initialized)

The `public` schema was reset, then migrated with `alembic upgrade head` (the container's supported
migration command; `flask db upgrade` errors with `KeyError: 'migrate'` in this environment — see the
methodology preamble). The migration applied all **255 revisions** (`migrations/versions/` contains 255
files, and a fresh run emits 255 `Running upgrade` lines) and created **77 public tables**. The
head-revision line and the resulting table count:

```
$ alembic upgrade head 2>&1 | tail -1
INFO  [alembic.runtime.migration] Running upgrade 7d7b84779837 -> 32f25cbf12f6, alias_audit_log_index_created_at
$ PGPASSWORD=test psql -h localhost -U test -d test -tAc "select count(*) from information_schema.tables where table_schema='public' and table_type='BASE TABLE';"
77
```

The key tables resolve (non-null `to_regclass`):

```
$ PGPASSWORD=test psql -h localhost -U test -d test -c "select to_regclass('public.users') as users, to_regclass('public.public_domain') as public_domain, to_regclass('public.job') as job;"
 users | public_domain | job
-------+---------------+-----
 users | public_domain | job
(1 row)
```

Before `init_app.py`, `public_domain` is empty; `python init_app.py` (`README.md:448`) then seeds it:

```
$ PGPASSWORD=test psql -h localhost -U test -d test -tAc "select count(*) from public_domain;"
0
$ python init_app.py
2026-07-01 23:35:20,034 - SL - INFO - 2844 - "/app/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain
$ PGPASSWORD=test psql -h localhost -U test -d test -c "select id, domain, use_as_reverse_alias from public_domain order by id;"
 id |  domain  | use_as_reverse_alias
----+----------+----------------------
  1 | sl.local | t
(1 row)
```

Source: `init_app.py:44` `LOG.i("Add %s to SL domain", alias_domain)`; `add_sl_domains()` is defined at
`init_app.py:39` and called at `init_app.py:73`, creating rows via `init_app.py:45`
`SLDomain.create(domain=alias_domain, use_as_reverse_alias=True)`.

### Answer: the system requires THREE long-lived Python services

These are the README self-hosting core trio (`README.md:433-495`, which documents exactly these three
`docker run ...` services and no others). For each, the readiness claim is paired with observed proof —
readiness is **demonstrated**, not asserted.

#### [1/3] Webapp — `python server.py`

Dev server, `debug=True`, port **7777**. The production equivalent is
`gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` (`Dockerfile:47`), where `wsgi.py` is simply
`from server import create_app` / `app = create_app()` (`wsgi.py:1-3`).

- **Startup log (verbatim).** Because `debug=True` runs the Werkzeug auto-reloader, `python server.py`
  runs **two** processes — the reloader supervisor (PID 3052) and the reloaded worker child (PID 3068,
  which carries `WERKZEUG_RUN_MAIN=true`); each imports `app/db.py`. Note the Werkzeug
  `Running on http://...` banner is **absent** because the `werkzeug` logger is disabled
  (`app/log.py:70-71`); readiness is therefore shown via `/health` + the port probe below:

```
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-01 23:39:35,168 - SL - DEBUG - 3052 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-01 23:39:36,703 - SL - DEBUG - 3068 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

- **Readiness proof (functional):** `curl http://127.0.0.1:7777/health` -> body `success`, HTTP 200:

```
$ curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:7777/health
200
$ curl -s http://127.0.0.1:7777/health
success
```

  Source: `server.py:213` `@app.route("/health", methods=["GET"])` -> `server.py:215`
  `return "success", 200`.

- **Port-binding proof (kernel + socket):** port 7777 is in `LISTEN` state on `127.0.0.1`
  (`0100007F:1E61`), and a socket probe returns `0` (open):

```
connect_ex 127.0.0.1:7777 -> 0
local_address=0100007F:1E61  st=0A(LISTEN)  ->  port 7777
```

#### [2/3] Email handler — `python email_handler.py`

An `aiosmtpd`-based SMTP server on port **20381**.

- **Startup log (verbatim — real PID 2872, real path):**

```
2026-07-01 23:35:44,351 - SL - INFO - 2872 - "/app/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-01 23:35:44,352 - SL - DEBUG - 2872 - "/app/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

  Sources: `email_handler.py:2403` `LOG.i("Listen for port %s", args.port)`; `email_handler.py:2383`
  `controller = Controller(MailHandler(), hostname="0.0.0.0", port=port)`; `email_handler.py:2386`
  `LOG.d("Start mail controller %s %s", controller.hostname, controller.port)`; default port 20381
  (argparse `default=20381`).

- **Port-binding proof + SMTP-speaking proof:** port 20381 is in `LISTEN` state on `0.0.0.0`
  (`00000000:4F9D`), a socket probe returns `0`, and the server emits its SMTP banner and answers
  `EHLO` (output of a raw-socket probe against 127.0.0.1:20381):

```
connect_ex 127.0.0.1:20381 -> 0
local_address=00000000:4F9D  st=0A(LISTEN)  ->  port 20381
```

```
BANNER: 220 4d33f21244c2 Python SMTP 1.4.2
EHLO-REPLY:
250-4d33f21244c2
250-SIZE 33554432
250-8BITMIME
250-SMTPUTF8
250 HELP
QUIT: 221 Bye
```

  The banner host `4d33f21244c2` is the container hostname; `Python SMTP 1.4.2` is the `aiosmtpd`
  version (matching `aiosmtpd 1.4.2`).

#### [3/3] Job runner — `python job_runner.py`

A background worker. **It has no port and prints no startup banner by design** — it enters a poll loop:
`while True: ... time.sleep(10)` (`job_runner.py:330` and `job_runner.py:347`).

- **Readiness proof (functional).** With the `job` table initially empty, a `ready` job was inserted
  (`state=0`) at `23:38:15`; within one 10 s poll cycle the runner picked it up (log at `23:38:24`),
  and the row advanced to `taken=t, state=2, attempts=1`. Producing commands and exact output:

```
$ PGPASSWORD=test psql -h localhost -U test -d test -c "select id,name,taken,state,attempts from job order by id;"
 id | name | taken | state | attempts
----+------+-------+-------+----------
(0 rows)
$ python -c "from app.db import Session; from app.models import Job; j=Job.create(name='test-job', payload={}, commit=True); print(j.id, j.state, j.taken, j.attempts)"
1 0 False 0
```

  Job-runner log (verbatim — real PID 2875):

```
2026-07-01 23:38:24,635 - SL - DEBUG - 2875 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 1 test-job {}>
2026-07-01 23:38:24,639 - SL - ERROR - 2875 - "/app/job_runner.py:304" - process_job() -  - Unknown job name test-job
NoneType: None
```

```
$ PGPASSWORD=test psql -h localhost -U test -d test -c "select id,name,taken,state,attempts from job order by id;"
 id |   name   | taken | state | attempts
----+----------+-------+-------+----------
  1 | test-job | t     |     2 |        1
(1 row)
```

  Sources: `job_runner.py:334` `LOG.d("Take job %s", job)`; `job_runner.py:304`
  `LOG.e("Unknown job name %s", job.name)`; `JobState` values `ready=0`, `taken=1`, `done=2`
  (`app/models.py:254-256`). The unknown job name is logged and the loop continues without raising, so
  the row reaches `state=2` (done).

  The bare **`NoneType: None`** line that immediately follows the ERROR is emitted deterministically (it
  appears after every `Unknown job name` line, confirmed across repeated jobs) and is **not** an error
  in itself: `LOG.e` is aliased to `logging.Logger.exception` (`app/log.py:77`
  `logging.Logger.e = logging.Logger.exception`), which always logs with `exc_info=True`. At
  `job_runner.py:304` that call sits in a plain `else` branch — **not** inside an `except` block — so
  `sys.exc_info()` is `(None, None, None)`, and the logging formatter renders that empty exception info
  as the literal text `NoneType: None`. (This is the same `LOG.e` = `Logger.exception` mechanism that,
  in Q1, prints the *full* chained traceback — there it runs inside the active `@app.errorhandler`
  exception context, so `exc_info` carries the real `ProgrammingError`.)

### Cross-service database proof

Each service connects to PostgreSQL with `application_name = webapp` (`DB_CONN_NAME` defaults to
`"webapp"`, `app/config.py:193`; `app/db.py:10` passes `connect_args={"application_name":
config.DB_CONN_NAME}` and opens the connection at `app/db.py:12`). Starting the services one at a time
and counting `webapp` backends attributes the connections exactly:

```
$ Q="select count(*) from pg_stat_activity where datname='test' and application_name='webapp';"
baseline webapp backends (no services): 0
after server.py: webapp backends = 2
after email_handler.py: webapp backends = 3
after job_runner.py: webapp backends = 4
```

`python server.py` opens **two** `webapp` backends (not one) because the Werkzeug reloader runs two
processes — verified from the process tree (the supervisor `python server.py` PID 3052, and its worker
child `/app/venv/bin/python /app/server.py` PID 3068 whose environ contains `WERKZEUG_RUN_MAIN=true`).
`email_handler.py` and `job_runner.py` open one each, for **4** total. The four backends and their
staggered `backend_start` times:

```
 pid  | application_name |         backend_start         |        state        
------+------------------+-------------------------------+---------------------
 3056 | webapp           | 2026-07-01 23:39:35.153934+00 | idle
 3071 | webapp           | 2026-07-01 23:39:36.688384+00 | idle
 3090 | webapp           | 2026-07-01 23:39:40.257461+00 | idle
 3109 | webapp           | 2026-07-01 23:39:43.688412+00 | idle in transaction
(4 rows)
```

### Answer

Required services = **webapp (`server.py` / `gunicorn wsgi:app`, port 7777)**,
**`email_handler.py` (port 20381)**, and **`job_runner.py` (no port)** — the three `docker run ...`
services documented in the README self-hosting guide (`README.md:433-495`).

`cron.py` and `event_listener.py` are additional entry points that exist in the repository but are
**not** among those three README services and are **not** needed for the login/email flows in Q1/Q3:
`cron.py` is a CLI task runner (`cron.py:1262` `if __name__ == "__main__":`, `cron.py:1264`
`argparse.ArgumentParser()`) invoked on a schedule by cron/`yacron` (e.g. `crontab.yml`
`command: python /code/cron.py -j stats`), and `event_listener.py` is a PostgreSQL event-source daemon
(`event_listener.py:29` `def main(...)`, `event_listener.py:94` `if __name__ == "__main__":`, using
`PostgresEventSource(EVENT_LISTENER_DB_URI)` at `event_listener.py:35`). (No repository
docker-compose/supervisor/Procfile enumerates a fixed process set, so no specific "N-process" superset
is asserted here beyond the README's three core services.)

---


## Q3 — Skipped-init SMTP rejection (exact status code + logs)

**Question.** With migrations run but `init_app.py` **not** run (so the `SLDomain` table — physically
named `public_domain` — is empty and no email domains are configured), start the email handler and
simulate an inbound email to any `@sl.local` address. Determine (a) which SMTP status code is returned
to the sender, and (b) what it logs to explain the rejection.

### Setup (State C — migrated only)

- DB reset, then **only** `alembic upgrade head` — `init_app.py` and `flask dummy-data` were
  **deliberately skipped.** Both would seed `public_domain`: `flask dummy-data` (`server.py:490`
  `@app.cli.command("dummy-data")` / `server.py:491` `def dummy_data():`) and `add_sl_domains()`
  (`init_app.py:39-56`). So the Q3 state is reached with `flask db upgrade`/`alembic upgrade head`
  **alone**.
- **Empty precondition verified immediately before injection** (real `psql` command + output):

```
$ PGPASSWORD=test psql -h localhost -U test -d test -c "select count(*) as public_domain from public_domain; select count(*) as custom_domain from custom_domain; select count(*) as alias from alias;"
 public_domain 
---------------
             0
(1 row)

 custom_domain 
---------------
             0
(1 row)

 alias 
-------
     0
(1 row)
```

  So the `SLDomain` table — physically named `public_domain` (`app/models.py:3116`
  `class SLDomain(Base, ModelMixin):`, `app/models.py:3119` `__tablename__ = "public_domain"`) — is
  **empty**.

### Action

Started `python email_handler.py` on this migrated-only DB. Its verbatim startup log (real PID 3292)
confirms it is listening on port 20381:

```
2026-07-01 23:56:17,526 - SL - INFO - 3292 - "/app/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-01 23:56:17,528 - SL - DEBUG - 3292 - "/app/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

Then a message to an `@sl.local` address was injected via a temporary Python raw-socket SMTP script
(`python /tmp/q3_inject.py`, removed afterward) mirroring the documented client command:

```bash
swaks --to e1@sl.local --from hey@google.com --server 127.0.0.1:20381
```

Source: `CONTRIBUTING.md:218` (the fenced block opens at `CONTRIBUTING.md:217`). The sender
`hey@google.com` is an **ordinary** address (not a bounce address), so the E515 path applies (see the
bounce nuance below).

### (a) SMTP status code returned to the sender (verbatim transcript)

Verbatim stdout of the injection script (the multi-line `EHLO` reply is joined with ` | ` by the
script for single-line display; its raw multi-line `250-` form appears in Q2):

```
BANNER            -> 220 4d33f21244c2 Python SMTP 1.4.2
EHLO              -> 250-4d33f21244c2 | 250-SIZE 33554432 | 250-8BITMIME | 250-SMTPUTF8 | 250 HELP
MAIL FROM <hey@google.com> -> 250 OK
RCPT TO <e1@sl.local>   -> 250 OK
DATA              -> 354 End data with <CR><LF>.<CR><LF>
DATA (end-of-data)-> 550 SL E515 Email not exist
QUIT              -> 221 Bye
```

- The exact reply is **`550 SL E515 Email not exist`**, returned at **end-of-DATA**.
- **RCPT is accepted (`250 OK`)** because `MailHandler` (`email_handler.py:2288` `class MailHandler:`)
  defines only `handle_DATA` (`email_handler.py:2289` `async def handle_DATA(...)`) and **no**
  `handle_RCPT`, so `aiosmtpd` accepts the recipient by default and the rejection happens after the
  message body is received.

### (b) What it logs to explain the rejection (verbatim ordered trail)

The complete ordered trail emitted to the `email_handler.py` console for this message (verbatim — real
timestamps, real PID 3292, real message-id `376cd35f-d838-43a3-a429-2ca840c2d1c2`, nothing elided):

```
2026-07-01 23:56:19,320 - SL - DEBUG - 3292 - "/app/app/log.py:24" - set_message_id() -  - set message_id 376cd35f-d838-43a3-a429-2ca840c2d1c2
2026-07-01 23:56:19,320 - SL - DEBUG - 3292 - "/app/email_handler.py:2342" - _handle() - 376cd35f-d838-43a3-a429-2ca840c2d1c2 - ====>=====>====>====>====>====>====>====>
2026-07-01 23:56:19,320 - SL - INFO - 3292 - "/app/email_handler.py:2343" - _handle() - 376cd35f-d838-43a3-a429-2ca840c2d1c2 - New message, mail from hey@google.com, rctp tos ['e1@sl.local'] 
2026-07-01 23:56:19,321 - SL - INFO - 3292 - "/app/email_handler.py:1956" - handle() - 376cd35f-d838-43a3-a429-2ca840c2d1c2 - Set CONTENT_TRANSFER_ENCODING
2026-07-01 23:56:19,321 - SL - DEBUG - 3292 - "/app/email_handler.py:1963" - handle() - 376cd35f-d838-43a3-a429-2ca840c2d1c2 - Cannot parse Postfix queue ID from None None
2026-07-01 23:56:19,438 - SL - DEBUG - 3292 - "/app/email_handler.py:1980" - handle() - 376cd35f-d838-43a3-a429-2ca840c2d1c2 - ==>> Handle mail_from:hey@google.com, rcpt_tos:['e1@sl.local'], header_from:hey@google.com, header_to:e1@sl.local, cc:None, reply-to:None, message_id:<q3-test@tester.local>, client_ip:None, headers:[('Date', 'Wed, 01 Jul 2026 23:45:00 +0000'), ('To', 'e1@sl.local'), ('From', 'hey@google.com'), ('Subject', 'q3 test'), ('Message-ID', '<q3-test@tester.local>'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-01 23:56:19,443 - SL - DEBUG - 3292 - "/app/email_handler.py:2202" - handle() - 376cd35f-d838-43a3-a429-2ca840c2d1c2 - Forward phase hey@google.com(hey@google.com) -> e1@sl.local
2026-07-01 23:56:19,453 - SL - DEBUG - 3292 - "/app/email_handler.py:545" - handle_forward() - 376cd35f-d838-43a3-a429-2ca840c2d1c2 - alias e1@sl.local not exist. Try to see if it can be created on the fly
2026-07-01 23:56:19,462 - SL - INFO - 3292 - "/app/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - 376cd35f-d838-43a3-a429-2ca840c2d1c2 - Cannot auto-create custom domain alias for e1@sl.local because there's no custom domain for sl.local
2026-07-01 23:56:19,462 - SL - INFO - 3292 - "/app/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - 376cd35f-d838-43a3-a429-2ca840c2d1c2 - Cannot auto-create e1@sl.local since it has no directory separator
2026-07-01 23:56:19,462 - SL - DEBUG - 3292 - "/app/email_handler.py:551" - handle_forward() - 376cd35f-d838-43a3-a429-2ca840c2d1c2 - alias e1@sl.local cannot be created on-the-fly, return 550
2026-07-01 23:56:19,463 - SL - INFO - 3292 - "/app/email_handler.py:2367" - _handle() - 376cd35f-d838-43a3-a429-2ca840c2d1c2 - Finish mail_from hey@google.com, rcpt_tos ['e1@sl.local'], takes 0.1435842514038086 seconds with return code '550 SL E515 Email not exist'<<===
```

The rejection is explained by three consecutive lines: the alias does not exist
(`email_handler.py:545`), neither auto-create path succeeds (`app/alias_utils.py:104` custom-domain and
`app/alias_utils.py:165` directory), and the handler therefore returns 550 (`email_handler.py:551`). The
final `email_handler.py:2367` line records the end-to-end result and the **exact** timing
`takes 0.1435842514038086 seconds` with `return code '550 SL E515 Email not exist'`.

> Note: `rctp tos` is the source's own spelling in the log format string (`email_handler.py:2343`
> `"New message, mail from %s, rctp tos %s "`) and is reproduced verbatim, not a transcription error.

### Source verification (exact literals)

- `app/email/status.py:51` → `E515 = "550 SL E515 Email not exist"` — matches the reply exactly.
- `email_handler.py:555` → `return [(False, status.E515)]` — the **only** `status.E515` return in the
  file.
- **Bounce-sender nuance:** `email_handler.py:552-553` →
  `if should_ignore_bounce(envelope.mail_from): return [(True, status.E207)]`, where
  `app/email/status.py:12` → `E207 = "250 SL E207 No bounce report"`. So a bounce-type sender would get
  **`250 SL E207 No bounce report`**, not the 550 — which is exactly why an ordinary sender
  (`hey@google.com`) was used to observe the E515 rejection.

### Root-cause precision

The rejection occurs because the alias **does not exist and cannot be auto-created** —
`try_auto_create()` (`app/alias_utils.py:202`, with sub-paths `try_auto_create_directory` at
`app/alias_utils.py:227` and `try_auto_create_via_domain` at `app/alias_utils.py:274`) returns `None`,
since both the custom-domain path (no `CustomDomain` for `sl.local`; log from
`check_if_alias_can_be_auto_created_for_custom_domain` at `app/alias_utils.py:104-105`) and the
directory path (no separator; log at `app/alias_utils.py:165`) fail.

It is **not** because `public_domain`/`SLDomain` is queried directly on this forward path — the
`SLDomain.get_by()` check lives in `is_valid_alias_address_domain()` on the **reply** path
(`app/email_utils.py:557` `def is_valid_alias_address_domain(...)`, `app/email_utils.py:560`
`if SLDomain.get_by(domain=domain):`). The empty `public_domain` is the **upstream reason** that no
verified/catch-all domain exists to auto-create the alias.

---


## Also report (observed, not fixed — read-only)

These items were observed during the investigation. Per the read-only rule they are **reported, not
corrected**.

- **`CONTRIBUTING.md` DB-port inconsistency.** The `DB_URI` example uses port **35432**
  (`CONTRIBUTING.md:94` → `DB_URI=postgresql://myuser:mypassword@localhost:35432/simplelogin`), but the
  PostgreSQL `docker run` command maps **15432** (`CONTRIBUTING.md:100` →
  `docker run ... -p 15432:5432 postgres:13`). The two ports disagree, so following the guide verbatim
  would point the app at a port where Postgres is not published. This is an observed documentation
  defect only; it was **not** corrected.
- **Additional context (informational).** The canonical local-dev startup order is
  `alembic upgrade head && flask dummy-data && python3 server.py` (`CONTRIBUTING.md:106`), with login
  `john@wick.com / password` (`CONTRIBUTING.md:109`). The note that the local dev DB is conventionally
  created via `db.create_all()` (rather than migrations) is at `CONTRIBUTING.md:129`.

---

## Final coverage checklist

Every concrete item named in the three questions is addressed by name above:

- [x] **`python server.py`** — launched for Q1 (empty DB → HTTP 500 on POST) and Q2 (webapp readiness
  via `/health` → `success`, 200).
- [x] **`init_app.py`** — run in Q2 (seeds `public_domain` with `sl.local`), deliberately skipped in Q3.
- [x] **`SLDomain` table (physically `public_domain`)** — empty precondition proven in Q3
  (`select count(*) from public_domain;` returns `0`, alongside `custom_domain` and `alias` both `0`).
- [x] **`@sl.local` address** — `e1@sl.local` used in the Q3 injection.
- [x] **"the login page"** — `GET /auth/login` returns HTTP 200 (no error); the **POST** triggers the
  exception (Q1).
- [x] **"email handler"** — `email_handler.py` is a required service (Q2) and hosts the rejection path
  (Q3).
- [x] **"SMTP status code"** — `550 SL E515 Email not exist` (Q3a).
- [x] **"what it logs to explain the rejection"** — the ordered log trail (Q3b), from `New message`
  through `Finish mail_from ... return code '550 SL E515 Email not exist'<<===`.
- [x] **Q2 required-services list** — webapp / `email_handler.py` / `job_runner.py`, each with
  per-service readiness and port-binding evidence (plus the `cron.py` / `event_listener.py`
  clarification).
- [x] **Observed doc defect** — `CONTRIBUTING.md` port 35432 vs 15432 noted, not fixed.

