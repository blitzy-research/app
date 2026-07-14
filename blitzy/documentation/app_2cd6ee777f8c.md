# SimpleLogin Startup Behavior — Q&A / Runtime-Observation Report

> **Scope note.** This is a **read-only behavioral investigation** of the SimpleLogin
> codebase (branch `app_2cd6ee777f8c`, commit `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`).
> Its sole purpose is to *observe and document* what the development stack actually does at
> three startup checkpoints — it does **not** change, fix, or delete any source. Every
> behavioral claim below is accompanied by the **exact command** that produced it and the
> **complete, unedited output** captured at runtime. All full-application executions were
> performed inside the **canonical Docker image**
> `andrewparkscaleai/coding-agent:simple-login__app__2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`
> (from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`) — **Python 3.10.18**,
> Poetry-pinned deps (Flask 1.1.2, **SQLAlchemy 1.3.24**, psycopg2-binary 2.9.3,
> Werkzeug 1.0.1, **aiosmtpd 1.4.2**), PostgreSQL 15. Values that are **not** a real-path
> capture are explicitly labeled `DB-layer stand-in` or `INFERRED (from code)`.

## Direct answers at a glance

| # | Question | Direct answer |
|---|----------|---------------|
| **REQ-1** | Empty DB → `python server.py` → open login page → full exception? | **`sqlalchemy.exc.ProgrammingError`** wrapping **`psycopg2.errors.UndefinedTable: relation "users" does not exist`**, raised on the login **POST** (first SQL query), rendered as HTTP **500**. |
| **REQ-2** | After migrations + `init_app.py`, which Python services + readiness? | **3 services**: web app `python server.py` (**:7777**), email handler `python email_handler.py` (**:20381**), job runner `python job_runner.py` (**no port**, readiness = "process alive & polling"). `cron.py`/`event_listener.py` are **not** required. |
| **REQ-3** | Migrations run, `init_app.py` skipped → email to `x@sl.local`? | **Rejected** with SMTP reply **`550 SL E515 Email not exist`** [app/email/status.py:51] — driven by alias-nonexistence + auto-create failure (not directly by the empty `SLDomain`). |

---

## §1 Environment & startup-order preamble

### 1.1 Canonical runtime

All scenario executions ran inside the canonical image (working dir `/app`). A from-scratch
`poetry install` of the pinned lock on a generic host fails on old build-from-source pins
(`cffi==1.14.4`, `cbor2==5.2.0`), so a hand-rebuilt environment would be **non-canonical**;
the image ships **Python 3.10** [Dockerfile:8] with the Poetry deps pre-installed in a venv at
`/app/venv`. Observed interpreter/dependency pins:

```text
$ /app/venv/bin/python -c "import sys,sqlalchemy,psycopg2,flask,aiosmtpd,werkzeug; \
    print('py',sys.version.split()[0]); print('sqlalchemy',sqlalchemy.__version__); \
    print('psycopg2',psycopg2.__version__); print('flask',flask.__version__); \
    print('aiosmtpd',aiosmtpd.__version__); print('werkzeug',werkzeug.__version__)"
py 3.10.18
sqlalchemy 1.3.24
psycopg2 2.9.3 (dt dec pq3 ext lo64)
flask 1.1.2
aiosmtpd 1.4.2
werkzeug 1.0.1
```

> **Note on `python`:** in the image, a bare `python` resolves to `/usr/local/bin/python`
> (3.10.18) which does **not** carry the app deps; the canonical `python server.py` etc. run
> with the venv active. Below, the documented command is `python <script>` (venv activated);
> it was executed as `/app/venv/bin/python <script>`, which is equivalent.

### 1.2 Configuration — the five unguarded boot variables

Runtime configuration is environment-driven through python-dotenv. The loader reads the file
named by `CONFIG` (else `./.env`):

```python
# app/config.py:65-71
config_file = os.environ.get("CONFIG")
if config_file:
    config_file = get_abs_path(config_file)
    print("load config file", config_file)
    load_dotenv(get_abs_path(config_file))
else:
    load_dotenv()
```

Five variables are read with **bare `os.environ[...]` subscripts**, so a missing value raises
`KeyError` at import time. All five are present in `example.env`, which our observation config
`/tmp/obs.env` is a byte-for-byte copy of:

| Variable | Definition | `example.env` value |
|----------|------------|---------------------|
| `URL` | `os.environ["URL"]` [app/config.py:79] | `URL=http://localhost:7777` [example.env:6] |
| `EMAIL_DOMAIN` | `os.environ["EMAIL_DOMAIN"].lower()` [app/config.py:92] | `EMAIL_DOMAIN=sl.local` [example.env:22] |
| `SUPPORT_EMAIL` | `os.environ["SUPPORT_EMAIL"]` [app/config.py:93] | `SUPPORT_EMAIL=support@sl.local` [example.env:40] |
| `DB_URI` | `os.environ["DB_URI"]` [app/config.py:192] | `DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin` [example.env:75] |
| `FLASK_SECRET` | `os.environ["FLASK_SECRET"]` [app/config.py:196] (+`raise RuntimeError` if empty [app/config.py:197-198]) | `FLASK_SECRET=secret` [example.env:77] |

`ALIAS_DOMAINS = OTHER_ALIAS_DOMAINS + [EMAIL_DOMAIN]` [app/config.py:160] therefore defaults to
`["sl.local"]`, making `sl.local` the natural domain to exercise in REQ-3.

The observation config was created **outside** the repository tree and never committed:

```bash
cp /app/example.env /tmp/obs.env      # canonical values incl. DB_URI -> .../simplelogin
export CONFIG=/tmp/obs.env
```

### 1.3 Redis is optional

`RedisSessionStore` is initialized **only** when `MEM_STORE_URI` is set [server.py:163-165];
`example.env` does not set it (confirmed: `grep -i MEM_STORE_URI example.env` → no match), so the
dev stack uses default Flask cookie sessions and an in-memory limiter. Consequently a login-page
**GET** touches neither Redis nor the database.

### 1.4 Ordered bring-up and per-scenario DB reset

The canonical order mirrors `README.md`:

```text
Postgres up  ->  create empty `simplelogin` DB  ->  (REQ-1 stops here)
             ->  migrations (alembic upgrade head)  ->  python init_app.py  ->  start services
```

The three checkpoints are **mutually-exclusive** database states, so the canonical
`simplelogin` database (the exact name in `example.env`'s `DB_URI`) was dropped/recreated between
scenarios:

- **REQ-1** = empty (no migrations)
- **REQ-2** = migrated **+** initialized
- **REQ-3** = migrated **but not** initialized

Reset command used between scenarios (Postgres role `myuser`, superuser):

```bash
psql -U myuser -d postgres -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity \
     WHERE datname='simplelogin' AND pid <> pg_backend_pid()"
psql -U myuser -d postgres -c "DROP DATABASE IF EXISTS simplelogin"
psql -U myuser -d postgres -c "CREATE DATABASE simplelogin OWNER myuser"
```

### 1.5 Migrations — `flask db upgrade` vs `alembic upgrade head` (observed)

The `README` documents `flask db upgrade`, but in this codebase **Flask-Migrate is not
registered** in `create_app`, so that command **fails**; the working, canonical migration path
is `alembic upgrade head` (exactly what CI uses:
`CONFIG=tests/test.env poetry run alembic upgrade head` [.github/workflows/main.yml:98]).
The schema is built from **255** version files [alembic.ini:5 `script_location = migrations`].
Both behaviors are shown with evidence in §3.1.

### 1.6 Stdout log format & the disabled werkzeug logger

Every SimpleLogin log line uses this format [app/log.py:12-16], written to **stdout** via
`StreamHandler(sys.stdout)` [app/log.py:41]:

```text
%(asctime)s - %(name)s - %(levelname)s - %(process)d - "%(pathname)s:%(lineno)d" - %(funcName)s() - %(message_id)s - %(message)s
```

**Observed nuance:** `app/log.py:70-71` sets `logging.getLogger("werkzeug").disabled = True`.
At capture time this **suppresses** the Werkzeug lines that are emitted through that logger — the
`* Running on http://127.0.0.1:7777` banner, `* Debugger is active!`, `* Debugger PIN:` and
`* Restarting with stat` — while **Flask's own** banner (`* Serving Flask app`,
`* Environment: production`, `* Debug mode: on`, printed via `click`) and SimpleLogin's own
`after_request` access log (SL logger, [server.py:284-292]) remain visible. This is confirmed by
live capture in §2 (a `grep` for those banners returns **0** matches).

### 1.7 Port 25 vs 20381 (dev vs prod) — flagged

The higher-level system spec describes the SMTP handler at **port 25** (the production Postfix
relay target). In local development, `python email_handler.py` binds the argparse **default 20381**
[email_handler.py:2399]. **The port actually bound and observed throughout this report is 20381.**

---

## §2 REQ-1 — Empty-database startup failure

### 2.1 Direct answer

With an **empty** `simplelogin` database (created, **migrations not run**), `python server.py`
**starts normally** and the login page **GET renders HTTP 200 with no query**. The failure
occurs on the login **POST (submit)**, when the first SQL query runs against the missing table.
The exception raised is:

> **`sqlalchemy.exc.ProgrammingError`** wrapping **`psycopg2.errors.UndefinedTable: relation "users" does not exist`**

The app's global error handler catches it and returns HTTP **500** (`error/500.html`); the full
traceback is written to stdout by `LOG.e(e)`.

### 2.2 Commands run

```bash
# Substrate: EMPTY simplelogin DB (no migrations)
psql -U myuser -d postgres -c "DROP DATABASE IF EXISTS simplelogin"
psql -U myuser -d postgres -c "CREATE DATABASE simplelogin OWNER myuser"
#   verify: 0 public tables, users table absent
psql -U myuser -d simplelogin -tAc \
  "SELECT count(*) FROM information_schema.tables WHERE table_schema='public'"   # -> 0
psql -U myuser -d simplelogin -tAc "SELECT to_regclass('public.users') IS NULL"  # -> t

# Start the web app through its real entry point (local_main -> app.run(debug=True, port=7777))
export CONFIG=/tmp/obs.env
python server.py

# (other shell) GET renders the form with NO DB query:
curl -i http://127.0.0.1:7777/auth/login
# then drive the REAL login POST reusing the session cookie + scraped csrf_token:
python /tmp/req1_login_post.py
```

The login page is `/auth/login` — route `@auth_bp.route("/login", methods=["GET", "POST"])`
[app/auth/views/login.py:21] under blueprint `url_prefix="/auth"` [app/auth/base.py:3-5]
(visiting `/` redirects anonymous users to `auth.login` [server.py:249-255]).

### 2.3 Observed output

**(a) `python server.py` startup banner** — the server binds `:7777` against the **empty** DB
without error (import-time `connection = engine.connect()` [app/db.py:12] succeeds; `create_app`
issues no query). Note the banner appears **twice** because `app.run(debug=True)` enables the
Werkzeug reloader (parent PID 4059 + child PID 4066), and note the **absence** of any
`* Running on` / `* Debugger PIN` line (§1.6):

```text
load config file /tmp/obs.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/oeklecxawjhbvsodurbt
Upload files to local dir
>>> init logging <<<
2026-07-14 20:02:37,906 - SL - DEBUG - 4059 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
load config file /tmp/obs.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/stewgzbyejjyqdziwjqe
Upload files to local dir
>>> init logging <<<
2026-07-14 20:02:39,467 - SL - DEBUG - 4066 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

Confirmation that the disabled werkzeug logger suppresses those banners:

```text
$ grep -c -E "Running on|Debugger PIN|Debugger is active|Restarting with" /tmp/req1_server.log
0
```

**(b) GET `/auth/login` → HTTP 200, no DB query.** The `curl -i` response line + headers (session
cookie value redacted):

```text
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 347629
Vary: Cookie
Set-Cookie: slapp=<REDACTED_SESSION_COOKIE>; Expires=Tue, 21-Jul-2026 20:14:54 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Tue, 14 Jul 2026 20:14:54 GMT
```

The corresponding SimpleLogin `after_request` access log confirms **200** with no error/query:

```text
2026-07-14 20:03:04,604 - SL - DEBUG - 4066 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.10499763488769531
```

> **Why GET does not query the DB.** The only `current_user` reference in the render chain
> (`login.html` → `single.html` → `base.html`) is
> `{% if NOW.timestamp < 1701475201 and current_user.is_authenticated and ... %}`
> [templates/base.html:89]. The epoch `1701475201` = 2023-12-01; at the current date the first
> clause is `False`, so Jinja short-circuits and `current_user.is_authenticated` is never
> evaluated (and an anonymous `is_authenticated` would not hit the DB anyway).

**(c) The real login POST — CSRF methodology.** The login form is a `flask_wtf` `FlaskForm`
rendering `<form method="post">` + `{{ form.csrf_token }}` [templates/auth/login.html:16-17], and
`WTF_CSRF_ENABLED` defaults **True**. A bare `curl -X POST` fails CSRF, never reaches the query,
and produces **no** error. The faithful path (used here) is a `requests.Session()` that GETs the
form to obtain the session cookie + `csrf_token`, then reposts them with credentials. Client-side
result:

```text
GET /auth/login -> 200
csrf_token scraped: True
POST /auth/login -> 500
POST Content-Type: text/html; charset=utf-8
contains-500-page-marker: True
```

**(d) The complete server-side traceback** (unedited), logged by the global
`@app.errorhandler(Exception)` → `LOG.e(e)` (where `LOG.e = logging.Logger.exception`
[app/log.py:77]) at **server.py:390**, followed by the `after_request` line showing the **500**:

```text
2026-07-14 20:04:08,538 - SL - ERROR - 4066 - "/app/server.py:390" - error_handler() -  - (psycopg2.errors.UndefinedTable) relation "users" does not exist
LINE 2: FROM users 
             ^

[SQL: SELECT users.directory_quota AS users_directory_quota, users.subdomain_quota AS users_subdomain_quota, users.password AS users_password, users.id AS users_id, users.created_at AS users_created_at, users.updated_at AS users_updated_at, users.email AS users_email, users.name AS users_name, users.is_admin AS users_is_admin, users.alias_generator AS users_alias_generator, users.notification AS users_notification, users.activated AS users_activated, users.disabled AS users_disabled, users.profile_picture_id AS users_profile_picture_id, users.otp_secret AS users_otp_secret, users.enable_otp AS users_enable_otp, users.last_otp AS users_last_otp, users.fido_uuid AS users_fido_uuid, users.default_alias_custom_domain_id AS users_default_alias_custom_domain_id, users.default_alias_public_domain_id AS users_default_alias_public_domain_id, users.lifetime AS users_lifetime, users.paid_lifetime AS users_paid_lifetime, users.lifetime_coupon_id AS users_lifetime_coupon_id, users.trial_end AS users_trial_end, users.default_mailbox_id AS users_default_mailbox_id, users.sender_format AS users_sender_format, users.sender_format_updated_at AS users_sender_format_updated_at, users.replace_reverse_alias AS users_replace_reverse_alias, users.referral_id AS users_referral_id, users.intro_shown AS users_intro_shown, users.max_spam_score AS users_max_spam_score, users.newsletter_alias_id AS users_newsletter_alias_id, users.include_sender_in_reverse_alias AS users_include_sender_in_reverse_alias, users.random_alias_suffix AS users_random_alias_suffix, users.expand_alias_info AS users_expand_alias_info, users.ignore_loop_email AS users_ignore_loop_email, users.alternative_id AS users_alternative_id, users.disable_automatic_alias_note AS users_disable_automatic_alias_note, users.one_click_unsubscribe_block_sender AS users_one_click_unsubscribe_block_sender, users.include_website_in_one_click_alias AS users_include_website_in_one_click_alias, users.disable_import AS users_disable_import, users.can_use_phone AS users_can_use_phone, users.phone_quota AS users_phone_quota, users.block_behaviour AS users_block_behaviour, users.include_header_email_header AS users_include_header_email_header, users.enable_data_breach_check AS users_enable_data_breach_check, users.flags AS users_flags, users.unsub_behaviour AS users_unsub_behaviour, users.delete_on AS users_delete_on 
FROM users 
WHERE users.email = %(email_1)s 
 LIMIT %(param_1)s]
[parameters: {'email_1': 'a@b.c', 'param_1': 1}]
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

[SQL: SELECT users.directory_quota AS users_directory_quota, ... (full User column list, identical to the block above) ... users.delete_on AS users_delete_on 
FROM users 
WHERE users.email = %(email_1)s 
 LIMIT %(param_1)s]
[parameters: {'email_1': 'a@b.c', 'param_1': 1}]
(Background on this error at: http://sqlalche.me/e/13/f405)
2026-07-14 20:04:08,544 - SL - DEBUG - 4066 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 500, takes 0.013438224792480469
```

**(e) DB-layer stand-in** — `DB-layer stand-in — not the full HTTP login path` (corroboration only).
This isolates the "connect succeeds, first query fails" distinction by mirroring
`app/db.py:9-12` directly (`create_engine(DB_URI)` + `engine.connect()`, then one `SELECT`):

```text
$ python /tmp/req1_dblayer_standin.py     # against the same empty `simplelogin`
engine.connect() -> SUCCEEDED on empty DB (no error at connect)
--- first query against missing table raised: ---
Traceback (most recent call last):
  ...
psycopg2.errors.UndefinedTable: relation "users" does not exist
LINE 1: SELECT id FROM users WHERE email = 'a@b.c'
                       ^
The above exception was the direct cause of the following exception:
  ...
sqlalchemy.exc.ProgrammingError: (psycopg2.errors.UndefinedTable) relation "users" does not exist
LINE 1: SELECT id FROM users WHERE email = 'a@b.c'
                       ^
[SQL: SELECT id FROM users WHERE email = 'a@b.c']
(Background on this error at: http://sqlalche.me/e/13/f405)
```

### 2.4 Cause → effect (with `file:line`)

1. **Import-time connect succeeds — the failure is at the first *query*, not at connect.**
   `app/db.py` builds the engine from `config.DB_URI` [app/db.py:9-11] and eagerly opens a
   connection at import: `connection = engine.connect()` [app/db.py:12]. Against an empty
   database this **succeeds** (observed: the server bound `:7777` and `/health` returned 200).
2. **GET renders 200 with no query** — short-circuited `NOW.timestamp < 1701475201` guard
   [templates/base.html:89]; `before_request` [server.py:257-270] issues no query.
3. **POST fires the query.** Inside `if form.validate_on_submit():` [app/auth/views/login.py:40]
   (which first passes CSRF), the line
   `user = User.get_by(email=email) or User.get_by(email=canonical_email)`
   [app/auth/views/login.py:43] calls `User.get_by` → `Session.query(cls).filter_by(**kw).first()`
   [app/models.py:84], issuing `SELECT ... FROM users ...` against table `users`
   (`class User ... __tablename__ = "users"` [app/models.py:336-337]). The missing relation makes
   psycopg2 raise `UndefinedTable`, which SQLAlchemy 1.3.24 wraps as
   `sqlalchemy.exc.ProgrammingError`.
4. **In-app handling → HTTP 500.** The global `@app.errorhandler(Exception)` [server.py:388-394]
   runs `LOG.e(e)` (the full traceback above) and, for non-`/api/` paths,
   `return render_template("error/500.html"), 500`. Because this handler **intercepts** the
   exception, Werkzeug's interactive debugger page does **not** engage — the complete traceback
   is what `LOG.e` (logging.exception) writes to stdout, which is exactly what is captured in
   §2.3(d). *(This is an observed correction to the a-priori expectation that `debug=True` would
   also surface a Werkzeug interactive traceback page: the app-level handler takes precedence.)*


---

## §3 REQ-2 — Required Python services + readiness evidence

### 3.1 Direct answer

After **migrations** and `python init_app.py` both succeed, the system needs **three** Python
services for local development:

1. **Web application** — `python server.py`, port **7777**.
2. **Email handler** — `python email_handler.py`, port **20381**.
3. **Job runner** — `python job_runner.py`, **no port / no bind banner** (readiness =
   "process alive and polling every 10s").

The auxiliary services `cron.py` (yacron) and `event_listener.py` (Proton `LISTEN/NOTIFY`) are
**not** part of the required set and were not started.

### 3.2 Prerequisite commands (migrations + init)

**Migrations — observed reality of both commands.** The README-documented `flask db upgrade`
**fails** because Flask-Migrate is not registered in `create_app`:

```text
$ CONFIG=/tmp/obs.env FLASK_APP=server.py python -m flask db upgrade
  ...
  File "/app/venv/lib/python3.10/site-packages/flask_migrate/cli.py", line 134, in upgrade
    _upgrade(directory, revision, sql, tag, x_arg)
  File "/app/venv/lib/python3.10/site-packages/flask_migrate/__init__.py", line 96, in wrapped
    f(*args, **kwargs)
  File "/app/venv/lib/python3.10/site-packages/flask_migrate/__init__.py", line 269, in upgrade
    config = current_app.extensions['migrate'].migrate.get_config(directory,
KeyError: 'migrate'
```

The **canonical** migration command (as used by CI:
`CONFIG=tests/test.env poetry run alembic upgrade head` [.github/workflows/main.yml:98]) succeeds
and applies all **255** version files [alembic.ini:5]:

```text
$ CONFIG=/tmp/obs.env alembic upgrade head        # exit=0
load config file /tmp/obs.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/dgysazihvddqpvptuvor
...
INFO  [alembic.runtime.migration] Running upgrade 88dd7a0abf54 -> 62afa3a10010, custom domain indices
INFO  [alembic.runtime.migration] Running upgrade 62afa3a10010 -> 91ed7f46dc81, alias_audit_log
INFO  [alembic.runtime.migration] Running upgrade 91ed7f46dc81 -> 7d7b84779837, user_audit_log
INFO  [alembic.runtime.migration] Running upgrade 7d7b84779837 -> 32f25cbf12f6, alias_audit_log_index_created_at

$ grep -c "Running upgrade" /tmp/req2_alembic.log        # -> 255
$ psql -U myuser -d simplelogin -tAc \
    "SELECT count(*) FROM information_schema.tables WHERE table_schema='public'"   # -> 77
$ psql -U myuser -d simplelogin -tAc "SELECT to_regclass('public.users') IS NOT NULL"  # -> t
$ psql -U myuser -d simplelogin -tAc "SELECT count(*) FROM public_domain"           # -> 0 (pre-init)
```

> **Note:** `python -m alembic` is **not** runnable (`No module named alembic.__main__`); the
> `alembic` **console script** must be used, matching CI's `poetry run alembic`.

**Initialization** — `python init_app.py` runs `load_pgp_public_keys()` and `add_sl_domains()`
[init_app.py:69-73]; the latter seeds `SLDomain` from `ALIAS_DOMAINS`:

```text
$ CONFIG=/tmp/obs.env python init_app.py        # exit=0
...
2026-07-14 20:07:51,949 - SL - DEBUG - 4395 - "/app/init_app.py:36" - load_pgp_public_keys() -  - Finish load_pgp_public_keys
2026-07-14 20:07:51,951 - SL - INFO - 4395 - "/app/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain

$ psql -U myuser -d simplelogin -tAc "SELECT count(*) FROM public_domain"   # -> 1
$ psql -U myuser -d simplelogin -tAc "SELECT domain FROM public_domain"     # -> sl.local
```

### 3.3 Service 1 — Web application (`python server.py`, :7777)

`local_main()` → `app.run(debug=True, port=7777)` [server.py:572-588,598-599]. Readiness banner
(again doubled by the debug reloader, parent PID 4446 + child 4469; the werkzeug
`* Running on`/`Debugger PIN` lines are suppressed per §1.6):

```text
load config file /tmp/obs.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/siyrrprbcuqwjtapcmfi
Upload files to local dir
>>> init logging <<<
2026-07-14 20:08:08,078 - SL - DEBUG - 4446 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
...
2026-07-14 20:08:09,646 - SL - DEBUG - 4469 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

Port-binding + readiness proof:

```text
$ curl -sS -o /dev/null -w 'GET /health -> %{http_code}\n' http://127.0.0.1:7777/health
GET /health -> 200
$ fuser 7777/tcp                      # PIDs owning the listening socket
7777/tcp:   4446  4469
$ awk '$4=="0A"{print $2}' /proc/net/tcp | grep -i ':1E61$'
0100007F:1E61                         # 0100007F = 127.0.0.1, 0x1E61 = 7777  (LISTEN)
```

**Production alternative:** `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` [Dockerfile:47],
where `wsgi.py` is simply `app = create_app()`; its readiness banner is
`Listening at: http://0.0.0.0:7777`.

### 3.4 Service 2 — Email handler (`python email_handler.py`, :20381)

The `__main__` block logs `Listen for port 20381` [email_handler.py:2403] **before** `main()`,
which builds `Controller(MailHandler(), hostname="0.0.0.0", port=port)` [email_handler.py:2383],
calls `controller.start()` [email_handler.py:2385], logs `Start mail controller 0.0.0.0 20381`
[email_handler.py:2386], and blocks on `while True: time.sleep(2)` [email_handler.py:2392-2393].
Observed readiness lines, **in runtime order**:

```text
2026-07-14 20:08:08,723 - SL - INFO - 4456 - "/app/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-14 20:08:08,724 - SL - DEBUG - 4456 - "/app/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

Port-binding + live-SMTP proof:

```text
$ fuser 20381/tcp
20381/tcp:   4456
$ awk '$4=="0A"{print $2}' /proc/net/tcp | grep -i ':4F9D$'
00000000:4F9D                         # 00000000 = 0.0.0.0, 0x4F9D = 20381  (LISTEN)
$ python -c "import smtplib; s=smtplib.SMTP('127.0.0.1',20381,timeout=8); \
    print('NOOP', s.docmd('NOOP')); print('EHLO', s.ehlo('obs')[0]); s.quit()"
NOOP (250, b'OK')
EHLO 250
```

The `0.0.0.0` bind address matches `hostname="0.0.0.0"` in the `Controller` constructor.

### 3.5 Service 3 — Job runner (`python job_runner.py`, no port)

**Honest readiness = "process alive and polling," not a bind banner.** The `__main__` block enters
`while True:` inside `create_light_app().app_context()`, iterates `get_jobs_to_run()`, logs
`Take job %s` **only when a job is claimed**, then `time.sleep(10)` [job_runner.py:329-347]. It
binds **no port** and prints **no explicit ready banner**. Its **complete** log after start (no
reloader doubling), captured across a **>10s** window (i.e. spanning a full poll cycle), shows the
init lines then silence — with an empty `Job` table there is no `Take job` line:

```text
load config file /tmp/obs.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/blogaertuduyephzhlko
Upload files to local dir
>>> init logging <<<
2026-07-14 20:09:05,533 - SL - DEBUG - 4598 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

Liveness + no-port proof:

```text
$ pgrep -af job_runner.py
4598 /app/venv/bin/python job_runner.py            # alive after the poll window
$ fuser 7777/tcp 20381/tcp                          # only server(4446/4469) + email(4456); NOT 4598
7777/tcp:   4446  4469
20381/tcp:   4456
```

### 3.6 Cause → effect (with `file:line`)

- **Web app** is the HTTP tier; `local_main()` → `app.run(debug=True, port=7777)`
  [server.py:572-588] binds `127.0.0.1:7777`; `/health` proves it serves requests.
- **Email handler** is the SMTP ingress; the aiosmtpd `Controller` [email_handler.py:2383] binds
  `0.0.0.0:20381` and the two log lines [email_handler.py:2403,2386] are the readiness signal.
- **Job runner** is a background worker with no socket; readiness can only honestly be reported as
  "process alive and polling every 10s" [job_runner.py:329-347] — reporting a bind banner would be
  false.
- **Migrations** build the 77-table schema from 255 version files [alembic.ini:5]; **`init_app.py`**
  seeds `SLDomain` (`Add sl.local to SL domain` [init_app.py:44]) — the step deliberately skipped
  in REQ-3.


---

## §4 REQ-3 — Skipped-initialization email rejection

### 4.1 Direct answer

With **migrations run** but `python init_app.py` **skipped** (so `SLDomain`, physical table
`public_domain` [app/models.py:3116-3119], is empty), an inbound email to any `x@sl.local` address
is **rejected**. The email handler returns the SMTP reply:

> **`550 SL E515 Email not exist`** [app/email/status.py:51]

The sending client receives it as `smtplib.SMTPDataError(550, b'SL E515 Email not exist')` at the
end of the `DATA` phase.

### 4.2 Commands run

```bash
# Migrations ONLY — do NOT run init_app.py
psql -U myuser -d postgres -c "DROP DATABASE IF EXISTS simplelogin"
psql -U myuser -d postgres -c "CREATE DATABASE simplelogin OWNER myuser"
CONFIG=/tmp/obs.env alembic upgrade head           # 255 steps -> 77 tables
#   verify migrated-but-NOT-initialized:
psql -U myuser -d simplelogin -tAc "SELECT count(*) FROM public_domain"   # -> 0  (SLDomain empty)
psql -U myuser -d simplelogin -tAc "SELECT count(*) FROM alias"           # -> 0  (no aliases)

# Start the SMTP ingress on :20381
CONFIG=/tmp/obs.env python email_handler.py

# (other shell) send to x@sl.local from a NON-bounce sender through the real aiosmtpd entry point
python /tmp/req3_send.py
#   req3_send.py: smtplib.SMTP("127.0.0.1", 20381); s.set_debuglevel(1)
#                 s.sendmail("someone@example.com", ["x@sl.local"], EmailMessage(...).as_bytes())
```

### 4.3 Observed output

**(a) The exact SMTP reply** — full transcript from `smtplib` with `set_debuglevel(1)`; note
`MAIL FROM`/`RCPT TO` are accepted with `250 OK` and the **`550`** is returned after `DATA`:

```text
send: 'ehlo [172.17.0.2]\r\n'
reply: b'250-46e83d7aea89\r\n'
reply: b'250-SIZE 33554432\r\n'
reply: b'250-8BITMIME\r\n'
reply: b'250-SMTPUTF8\r\n'
reply: b'250 HELP\r\n'
reply: retcode (250); Msg: b'46e83d7aea89\nSIZE 33554432\n8BITMIME\nSMTPUTF8\nHELP'
send: 'mail FROM:<someone@example.com> size=151\r\n'
reply: b'250 OK\r\n'
reply: retcode (250); Msg: b'OK'
send: 'rcpt TO:<x@sl.local>\r\n'
reply: b'250 OK\r\n'
reply: retcode (250); Msg: b'OK'
send: 'data\r\n'
reply: b'354 End data with <CR><LF>.<CR><LF>\r\n'
reply: retcode (354); Msg: b'End data with <CR><LF>.<CR><LF>'
data: (354, b'End data with <CR><LF>.<CR><LF>')
send: b'From: someone@example.com\nTo: x@sl.local\nSubject: test\nContent-Type: text/plain; charset="utf-8"\nContent-Transfer-Encoding: 7bit\nMIME-Version: 1.0\n\nhi\n\r\n.\r\n'
reply: b'550 SL E515 Email not exist\r\n'
reply: retcode (550); Msg: b'SL E515 Email not exist'
data: (550, b'SL E515 Email not exist')
send: 'rset\r\n'
reply: b'250 OK\r\n'
reply: retcode (250); Msg: b'OK'
send: 'quit\r\n'
reply: b'221 Bye\r\n'
reply: retcode (221); Msg: b'Bye'
RESULT: SMTPDataError -> code= 550 msg= b'SL E515 Email not exist'
```

**(b) The rejection log lines** from the handler (SL log format; all lines share one
`message_id` = `460e9acb-d9dc-42f6-8635-53114d1df7b6`). The **two lines that explain the
rejection** are marked; the two intermediate `alias_utils` lines show *why* on-the-fly creation
failed:

```text
2026-07-14 20:11:53,825 - SL - INFO - 4766 - "/app/email_handler.py:2343" - _handle() - 460e9acb-d9dc-42f6-8635-53114d1df7b6 - New message, mail from someone@example.com, rctp tos ['x@sl.local'] 
2026-07-14 20:11:53,968 - SL - DEBUG - 4766 - "/app/email_handler.py:1980" - handle() - 460e9acb-d9dc-42f6-8635-53114d1df7b6 - ==>> Handle mail_from:someone@example.com, rcpt_tos:['x@sl.local'], header_from:someone@example.com, header_to:x@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'someone@example.com'), ('To', 'x@sl.local'), ('Subject', 'test'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:['SIZE=151'], rcpt_options:[]
2026-07-14 20:11:53,973 - SL - DEBUG - 4766 - "/app/email_handler.py:2202" - handle() - 460e9acb-d9dc-42f6-8635-53114d1df7b6 - Forward phase someone@example.com(someone@example.com) -> x@sl.local
2026-07-14 20:11:53,983 - SL - DEBUG - 4766 - "/app/email_handler.py:545" - handle_forward() - 460e9acb-d9dc-42f6-8635-53114d1df7b6 - alias x@sl.local not exist. Try to see if it can be created on the fly     <-- REJECTION LINE 1
2026-07-14 20:11:53,993 - SL - INFO - 4766 - "/app/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - 460e9acb-d9dc-42f6-8635-53114d1df7b6 - Cannot auto-create custom domain alias for x@sl.local because there's no custom domain for sl.local
2026-07-14 20:11:53,993 - SL - INFO - 4766 - "/app/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - 460e9acb-d9dc-42f6-8635-53114d1df7b6 - Cannot auto-create x@sl.local since it has no directory separator
2026-07-14 20:11:53,993 - SL - DEBUG - 4766 - "/app/email_handler.py:551" - handle_forward() - 460e9acb-d9dc-42f6-8635-53114d1df7b6 - alias x@sl.local cannot be created on-the-fly, return 550   <-- REJECTION LINE 2
2026-07-14 20:11:53,994 - SL - INFO - 4766 - "/app/email_handler.py:2367" - _handle() - 460e9acb-d9dc-42f6-8635-53114d1df7b6 - Finish mail_from someone@example.com, rcpt_tos ['x@sl.local'], takes 0.16962909698486328 seconds with return code '550 SL E515 Email not exist'<<===
```

### 4.4 Cause → effect (with `file:line`)

A bare `x@sl.local` forward enters `handle_forward(envelope, msg, rcpt_to)` [email_handler.py:536]:

```python
alias = Alias.get_by(email=alias_address)                     # :543  -> None (no aliases)
if not alias:
    LOG.d("alias %s not exist. Try to see if it can be created on the fly", ...)  # :545-548
    alias = try_auto_create(alias_address)                    # :549  -> None
    if not alias:
        LOG.d("alias %s cannot be created on-the-fly, return 550", ...)           # :551
        if should_ignore_bounce(envelope.mail_from):
            return [(True, status.E207)]                       # :553 (only for ignore-bounce senders)
        else:
            return [(False, status.E515)]                      # :555 -> 550 SL E515 Email not exist
```

- `try_auto_create(address)` [app/alias_utils.py:202-224] first tries
  `try_auto_create_via_domain` [app/alias_utils.py:274-278] — returns `None` (observed:
  *"Cannot auto-create custom domain alias for x@sl.local because there's no custom domain for
  sl.local"* [app/alias_utils.py:104]) — then `try_auto_create_directory`
  [app/alias_utils.py:227-235] — returns `None` (observed: *"Cannot auto-create x@sl.local since
  it has no directory separator"* [app/alias_utils.py:165]). Both fail in a
  migrated-but-uninitialized DB → `try_auto_create` returns `None` → **E515**.
- The final reply string is assembled by `handle()` — for this single-recipient forward,
  `handle_forward` returns `[(False, status.E515)]`, `handle()` returns that failure
  [email_handler.py:2233], and `_handle` [email_handler.py:2367] logs the closing
  `return code '550 SL E515 Email not exist'`. aiosmtpd sends it as the reply to `DATA`.

**Sender edge (why the sender matters):** `E515` is returned **only when**
`should_ignore_bounce(envelope.mail_from)` is `False` [email_handler.py:552-555]; an ignore-bounce
sender would instead yield `E207` (`250 SL E207 No bounce report`). The message was therefore sent
from a normal sender (`someone@example.com`) to observe the `550`.

**SPF-downgrade edge (did not fire):** `_handle` downgrades a `5xx` to `E216` **only if**
`spamd_result.spf` is `fail`/`soft_fail` [email_handler.py:2357-2365]; for this plain test email
`SpamdResult.extract_from_headers(msg)` is `None`, so no downgrade occurred and the `550` stood.

### 4.5 Cause **honesty**: the `SLDomain` nuance (`@sl.local`)

Leading with the direct answer (**`550 SL E515 Email not exist`**), the honest cause is:

- On the **forward** path the rejection is driven by **alias-nonexistence + auto-create failure**
  (the `:543 → :549 → :555` chain above), which is **independent of the `SLDomain`
  (`public_domain`) table contents**. This is confirmed by the observed logs — the failure reasons
  are "no custom domain" and "no directory separator", **not** a domain-validity check.
- The `SLDomain` check lives in `is_valid_alias_address_domain` [app/email_utils.py:557-563]
  (`SLDomain.get_by(domain=domain)` [app/email_utils.py:560]) and governs the **reply** path, not
  this forward path.
- Skipping `python init_app.py` **is** the scenario premise and **does** leave `SLDomain` empty —
  because `add_sl_domains()` [init_app.py:39-56], invoked only from `init_app.py`'s `__main__`
  [init_app.py:69-73], never runs — but the **operative cause of this specific 550 is the missing
  alias**, not the empty `SLDomain`. (`sl.local` is the default `EMAIL_DOMAIN` [app/config.py:92]
  from `example.env`.)


---

## §5 Cleanup & reproducibility note

All observation was performed inside the canonical container `sl-setup` (image
`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`). Every temporary artifact used
for observation lived **outside** the repository tree (under `/tmp` inside the container) and was
removed on completion, so the source repository is left byte-for-byte unchanged except for this
one document.

Temporary artifacts used (all outside the repo, all removed):

| Artifact (in container `/tmp`) | Purpose |
|--------------------------------|---------|
| `/tmp/obs.env` | Copy of `example.env` (canonical boot vars); pointed `CONFIG` at it |
| `/tmp/req1_login_post.py` | REQ-1 real HTTP login flow (GET → scrape `csrf_token` → POST) |
| `/tmp/req1_dblayer_standin.py` | REQ-1 labeled DB-layer stand-in |
| `/tmp/req3_send.py` | REQ-3 `smtplib` sender to `x@sl.local` |
| `/tmp/req1_server.log`, `/tmp/req2_*.log`, `/tmp/req3_*.log` | Captured stdout/stderr |
| `simplelogin` database contents | Reset (drop/recreate) between scenarios; ephemeral |

No source file was modified or deleted; no dependency, schema, or migration was authored; the
observed behaviors were **documented, not remediated**. On completion:

```text
$ git status --porcelain
?? blitzy/documentation/app_2cd6ee777f8c.md
```

— i.e., the only repository change is this answer document.

---

## §6 Coverage checklist

Every named item is addressed by name with an exact value + `file:line`, and each requirement
leads with its direct answer:

| Item | Where addressed | Exact value / evidence |
|------|-----------------|------------------------|
| **`server.py`** | §2, §3.3 | web entry `local_main()` → `app.run(debug=True, port=7777)` [server.py:572-588,598-599]; global error handler `@app.errorhandler(Exception)` [server.py:388-394] |
| **`init_app.py`** | §3.2, §3.6, §4.5 | seeds `SLDomain` via `add_sl_domains()` [init_app.py:39-56,69-73] → `Add sl.local to SL domain` [init_app.py:44]; run in REQ-2, **skipped** in REQ-3 |
| **`email_handler`** (`email_handler.py`) | §3.4, §4 | SMTP `Controller` on **20381** [email_handler.py:2383,2403,2386]; `handle_forward` rejection path [email_handler.py:536,543,549,551,555] |
| **`SLDomain`** | §4.5 | physical table `public_domain` [app/models.py:3116-3119]; forward-vs-reply cause nuance (reply-path check at [app/email_utils.py:557-563]) |
| **`@sl.local`** | §2, §4 | concrete `x@sl.local` recipient exercised; `sl.local` = default `EMAIL_DOMAIN` [app/config.py:92] / [example.env:22] |
| **SMTP status code** | §4.1, §4.3 | exact **`550 SL E515 Email not exist`** [app/email/status.py:51] |
| **migrations** | §1.5, §3.2 | `flask db upgrade` **fails** (`KeyError: 'migrate'`); canonical `alembic upgrade head` builds schema from **255** version files [alembic.ini:5] → 77 tables |
| **login page** | §2 | `/auth/login` [app/auth/views/login.py:21]; GET-200 vs POST-500 distinction; `User.get_by` at [app/auth/views/login.py:43] |
| Leads with direct answer | §2.1, §3.1, §4.1 | ✓ each section opens with the direct answer |
| Observed vs. labeled | throughout | real-path captures unlabeled; the REQ-1 DB-layer stand-in explicitly labeled |
| Exact command with every output | §2.2, §3.2–3.5, §4.2 | ✓ |
| **Port 25 vs 20381** flag | §1.7, §3.4 | prod spec = port 25; dev argparse default = **20381** [email_handler.py:2399]; observed bind = **20381** |

### Additional observed nuances (all confirmed at runtime)

1. **Import-time connect succeeds on an empty DB** — the REQ-1 failure is at the first *query*,
   not at connect [app/db.py:12]. *(Observed: server bound `:7777`, `/health`=200 on empty DB.)*
2. **GET vs POST (REQ-1)** — GET `/auth/login` = 200 with no query (guard [templates/base.html:89]);
   POST triggers the error. Both states captured.
3. **CSRF on the login POST** — a valid `csrf_token` + session cookie are required to reach
   `User.get_by`; a bare POST is rejected by CSRF and does not error. Method documented in §2.3(c).
4. **werkzeug logger disabled** [app/log.py:70-71] — the `* Running on`/`Debugger PIN` banners are
   suppressed (grep count 0); Flask's own banner and the SL `after_request` log remain visible.
5. **Job runner honesty** — "process alive and polling every 10s," not a bind banner
   [job_runner.py:329-347].
6. **REQ-3 sender edge** — sent from a non-bounce sender to observe `E515`; an ignore-bounce
   sender would yield `E207` [email_handler.py:552-555].
7. **REQ-3 cause honesty** — the `550` is caused by alias-nonexistence + auto-create failure
   (forward path), not directly by the empty `SLDomain` (which governs the reply path).
8. **SPF downgrade did not fire** — `5xx`→`E216` requires an SPF fail/soft_fail
   [email_handler.py:2357-2365]; `spamd_result` was `None` for the plain test email.

