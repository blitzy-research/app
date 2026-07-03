# SimpleLogin — Runtime-Behavior Q&A (developer-environment bring-up)

This document answers three runtime-behavior questions about the SimpleLogin stack. Every answer was produced **by actually running the code first** and then quoting the captured output verbatim. Each behavioral claim is paired with exactly one adjacent evidence block (the command that was run plus its captured output). Any statement not directly observed is prefixed with `(inferred)`. Exact identifiers, strings, status codes, and file paths are cited with their `file:line` reference.

The three questions are:

- **Q1 — Empty-database failure mode.** With PostgreSQL running and an empty database (no tables, migrations not run), what exact error is hit when the server is started with `python server.py` and the login page is then opened in a browser?
- **Q2 — Required runtime services.** After migrations *and* the initialization script `init_app.py` have been run, which Python services must be running for the system to work?
- **Q3 — Skipped-initialization rejection.** If migrations are run but `init_app.py` is skipped — so the SLDomain table should be empty (no email domains configured) — what SMTP status code does the email handler return, and what does it log, when a message addressed to any `@sl.local` address arrives?

---

## 1. Introduction — canonical environment, exact commands, and configuration actually used

### 1.1 Runtime versions actually observed

All commands were run inside the provided Docker image using the project's in-project virtual environment `./.venv` (Python 3.10 via pyenv), which is the canonical interpreter a normal user runs this project with in this image.

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

These match the manifest pins: Python `^3.10` [pyproject.toml:61] (base image `FROM python:3.10` [Dockerfile:8]), `SQLAlchemy = "1.3.24"` [pyproject.toml:116], `Flask = "^1.1.2"` [pyproject.toml:62], `psycopg2-binary = "^2.9.3"` [pyproject.toml:71], `python-dotenv = "^0.14.0"` [pyproject.toml:68], `Flask-Migrate = "^2.5.3"` [pyproject.toml:77]. Note the installed **aiosmtpd is `1.4.2`** whereas the manifest pins `aiosmtpd = "^1.2"` [pyproject.toml:87]; the observed value `1.4.2` is reported (it also appears in the SMTP banner in Q2).

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

The canonical `example.env` ships `EMAIL_DOMAIN=sl.local` [example.env:22] (also `URL` [example.env:6], `SUPPORT_EMAIL` [example.env:40], `DB_URI` [example.env:75], `FLASK_SECRET` [example.env:77]). The only deviation is the `DB_URI` **port `15432`** instead of `example.env`'s `5432` — because in this image the dockerized PostgreSQL 13 publishes its `5432` on host port `15432` (the shipped helper `scripts/reset_local_db.sh` uses the same `...localhost:15432/simplelogin`). `EMAIL_DOMAIN=sl.local` is the `@sl.local` domain referenced in Q3, and it is folded into `ALIAS_DOMAINS`: when `ALIAS_DOMAINS` is not in the env, `ALIAS_DOMAINS = OTHER_ALIAS_DOMAINS + [EMAIL_DOMAIN]` [app/config.py:160]. Verified at runtime:

```
$ ./.venv/bin/python -c "from app import config; print('ALIAS_DOMAINS =', config.ALIAS_DOMAINS); print('EMAIL_DOMAIN =', config.EMAIL_DOMAIN)"
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

Command and output (drop + recreate the `public` schema, then verify emptiness):

```
$ docker exec sl-db psql -U myuser -d simplelogin -c 'drop schema public cascade; create schema public;'
NOTICE:  drop cascades to 82 other objects
DETAIL:  drop cascades to table alembic_version
...
CREATE SCHEMA

$ docker exec sl-db psql -U myuser -d simplelogin -c "SELECT count(*) FROM information_schema.tables WHERE table_schema='public';"
 count
-------
     0
(1 row)

$ docker exec sl-db psql -U myuser -d simplelogin -c "\dt"
Did not find any relations.
```

The database now has **0 tables** / "Did not find any relations."

### 2.2 `python server.py` starts successfully against the empty database and binds :7777

Claim: starting the server does **not** fail against an empty-but-existing database.

Command and captured startup stdout:

```
$ export PATH="$PWD/.venv/bin:$PATH"; export PYTHONUNBUFFERED=1
$ python server.py            # started in the background, stdout+stderr captured to a log
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/exclujtpbpmevyeaulyw
Upload files to local dir
>>> init logging <<<
2026-07-02 23:36:22,148 - SL - DEBUG - 43623 - ".../app/utils.py:17" - <module>() -  - load words file: .../local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
```

Independent readiness / bind proof (the `/health` route returns `return "success", 200` [server.py:214-215]):

```
$ curl -s -w "\nHTTP %{http_code}\n" http://localhost:7777/health
success
HTTP 200
```

**Cause → effect:** the process reaches the query-serving state because `app/db.py` opens the database connection at **import** time — `engine = create_engine(...)` [app/db.py:9] then `connection = engine.connect()` [app/db.py:12] — and connecting to an *existing but empty* database succeeds (no table is touched yet). The dev server is launched by `local_main()` [server.py:572] via `app.run(debug=True, port=7777)` [server.py:588] from the `if __name__ == "__main__": local_main()` entrypoint [server.py:598-599]. `(inferred)` The two process IDs in the logs (a reloader parent and a worker child) are werkzeug's reloader, which `debug=True` enables.

### 2.3 Opening the login page: a plain GET renders with no error

Claim: opening the login page with a GET does **not** hit the error — it renders HTTP 200 without querying the database.

Command and output — `GET /` (the `index()` view [server.py:251]) redirects an anonymous visitor to the login page:

```
$ curl -s -i http://localhost:7777/
HTTP/1.0 302 FOUND
Content-Type: text/html; charset=utf-8
Content-Length: 229
Location: http://localhost:7777/auth/login
...
```

Command and output — `GET /auth/login` renders the login page with **HTTP 200** and no error:

```
$ curl -s -i http://localhost:7777/auth/login | head -8
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 381563
...
    <title>
      Login
      | SimpleLogin
```

**Cause → effect:** the `auth.login` view only queries the `User` table inside the POST branch `if form.validate_on_submit():` [app/auth/views/login.py:40] where `user = User.get_by(email=email) or User.get_by(email=canonical_email)` [app/auth/views/login.py:43]; a plain GET simply renders `auth/login.html` [app/auth/views/login.py:74] and the anonymous `current_user.is_authenticated` check [app/auth/views/login.py:28] issues no query. So merely *displaying* the login page does not touch a missing table.

### 2.4 Submitting the login form (real POST) triggers the exact error → HTTP 500

To force the first real database query, the login form was submitted through the real HTTP path (a browser "logging in"): a `GET /auth/login` first to obtain the session cookie and the `csrf_token` hidden field, then a `POST /auth/login` with `email`, `password`, and `csrf_token`.

Command and the HTTP response the client receives:

```
$ curl -s -i -b cookies -c cookies \
    --data-urlencode "email=test@example.com" \
    --data-urlencode "password=password123" \
    --data-urlencode "csrf_token=${CSRF}" \
    http://localhost:7777/auth/login
HTTP/1.0 500 INTERNAL SERVER ERROR
Content-Type: text/html; charset=utf-8
Content-Length: 5749
...
```

The response body is the rendered `error/500.html` template (not the werkzeug interactive debugger). Signature strings present in the 5749-byte body:

```
$ grep -oE "Server error|Looks like we are having some server issues|We are notified and will look at this issue asap" response.html | sort | uniq -c
      1 Looks like we are having some server issues
      1 Server error
      1 We are notified and will look at this issue asap
```

Those strings come from `templates/error/500.html` (`{% block error_name %}Server error{% endblock %}` and "Looks like we are having some server issues... We are notified and will look at this issue asap!").

### 2.5 The exact error: full Python traceback (verbatim), first failing relation = `users`

Claim: the exact error is a `sqlalchemy.exc.ProgrammingError` wrapping `psycopg2.errors.UndefinedTable`, with the message `relation "users" does not exist`; the **first failing relation is `users`**.

Captured server stdout (verbatim; the repeated ~50-column SELECT list and the intermediate SQLAlchemy engine frames are elided at the clearly-marked `…`, everything else is exactly as emitted):

```
2026-07-02 23:37:08,109 - SL - ERROR - 43635 - "/tmp/blitzy/app/blitzy-33f70bcd-a718-4307-83f4-2e539e81e639_3e4fde/server.py:390" - error_handler() -  - (psycopg2.errors.UndefinedTable) relation "users" does not exist
LINE 2: FROM users
             ^

[SQL: SELECT users.directory_quota AS users_directory_quota, users.subdomain_quota AS users_subdomain_quota, users.password AS users_password, users.id AS users_id, … (full ~50-column SELECT list elided) … users.delete_on AS users_delete_on
FROM users
WHERE users.email = %(email_1)s
 LIMIT %(param_1)s]
[parameters: {'email_1': 'test@example.com', 'param_1': 1}]
(Background on this error at: http://sqlalche.me/e/13/f405)
Traceback (most recent call last):
  File ".../.venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1276, in _execute_context
    self.dialect.do_execute(
  File ".../.venv/lib/python3.10/site-packages/sqlalchemy/engine/default.py", line 608, in do_execute
    cursor.execute(statement, parameters)
psycopg2.errors.UndefinedTable: relation "users" does not exist
LINE 2: FROM users
             ^


The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File ".../.venv/lib/python3.10/site-packages/flask/app.py", line 1950, in full_dispatch_request
    rv = self.dispatch_request()
  File ".../.venv/lib/python3.10/site-packages/flask_debugtoolbar/__init__.py", line 125, in dispatch_request
    return view_func(**req.view_args)
  File "/root/.pyenv/versions/3.10.20/lib/python3.10/cProfile.py", line 110, in runcall
    return func(*args, **kw)
  File ".../.venv/lib/python3.10/site-packages/flask_limiter/extension.py", line 702, in __inner
    return obj(*a, **k)
  File ".../app/auth/views/login.py", line 43, in login
    user = User.get_by(email=email) or User.get_by(email=canonical_email)
  File ".../app/models.py", line 84, in get_by
    return Session.query(cls).filter_by(**kw).first()
  … (intermediate SQLAlchemy ORM/engine frames elided) …
  File ".../.venv/lib/python3.10/site-packages/sqlalchemy/engine/default.py", line 608, in do_execute
    cursor.execute(statement, parameters)
sqlalchemy.exc.ProgrammingError: (psycopg2.errors.UndefinedTable) relation "users" does not exist
LINE 2: FROM users
             ^

[SQL: SELECT users.directory_quota AS users_directory_quota, … (full ~50-column SELECT list elided) … FROM users WHERE users.email = %(email_1)s  LIMIT %(param_1)s]
[parameters: {'email_1': 'test@example.com', 'param_1': 1}]
(Background on this error at: http://sqlalche.me/e/13/f405)
2026-07-02 23:37:08,115 - SL - DEBUG - 43635 - ".../server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 500, takes 0.01574540138244629
```

### 2.6 How the traceback surfaces (which path produced HTTP 500)

Claim: the traceback surfaces via the **registered global exception handler**, not werkzeug's interactive debugger.

Evidence (from §2.5): the very first line of the captured error is emitted by `error_handler()` at `"...server.py:390"` — i.e. the `@app.errorhandler(Exception)` handler `def error_handler(e)` [server.py:389] whose body is `LOG.e(e)` [server.py:390] followed by `return render_template("error/500.html"), 500` [server.py:394] for non-`/api/` paths (the `/api/` branch would instead `return jsonify(error="Internal error"), 500` [server.py:391-392]). The `LOG.e(e)` call (recall `LOG.e = logging.Logger.exception` [app/log.py:77]) is what printed the full traceback to stdout; the client then received the rendered `error/500.html` (matching §2.4). The `after_request` log line `... POST /auth/login ImmutableMultiDict([]) 500 ...` confirms the request completed with status 500.

**Cause → effect summary for Q1:** the connection opened at import [app/db.py:12] succeeds on the empty database, so `python server.py` starts and even serves the login page GET; the failure only occurs on the **first request that queries a not-yet-migrated table** — the login form POST, which runs `User.get_by(...)` [app/auth/views/login.py:43] → `Session.query(cls).filter_by(**kw).first()` [app/models.py:84] against the missing `users` table, raising `sqlalchemy.exc.ProgrammingError: (psycopg2.errors.UndefinedTable) relation "users" does not exist`.

### 2.7 Reproducibility

A second identical `POST /auth/login` (a different email) against the still-empty database reproduced the same result:

```
$ curl -s ... --data-urlencode "email=second@example.com" ... http://localhost:7777/auth/login -o /dev/null -w "RUN2 RESPONSE_STATUS=%{http_code}\n"
RUN2 RESPONSE_STATUS=500

# new server-stdout error line for run #2:
2026-07-02 23:38:28,886 - SL - ERROR - 43635 - ".../server.py:390" - error_handler() -  - (psycopg2.errors.UndefinedTable) relation "users" does not exist
2026-07-02 23:38:28,887 - SL - DEBUG - 43635 - ".../server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 500, takes 0.0055637359619140625
```

Both runs produced HTTP 500 and the identical `psycopg2.errors.UndefinedTable: relation "users" does not exist` / `sqlalchemy.exc.ProgrammingError`. The result reproduced identically.

---

## 3. Q2 — Required runtime services

### 3.1 Provision the migrated + initialized database

First run migrations, then the initialization script `init_app.py`.

Command and output — migrations:

```
$ alembic upgrade head
...
INFO  [alembic.runtime.migration] Running upgrade 7d7b84779837 -> 32f25cbf12f6, alias_audit_log_index_created_at

$ docker exec sl-db psql -U myuser -d simplelogin -tAc "SELECT count(*) FROM information_schema.tables WHERE table_schema='public';"
77
$ docker exec sl-db psql -U myuser -d simplelogin -tAc "SELECT version_num FROM alembic_version;"
32f25cbf12f6
```

Command and output — `python init_app.py` (its `__main__` wraps `create_light_app().app_context()` [init_app.py:71] then calls `load_pgp_public_keys()` [init_app.py:72] and `add_sl_domains()` [init_app.py:73]):

```
$ python init_app.py
>>> URL: http://localhost:7777
...
>>> init logging <<<
2026-07-02 23:39:49,818 - SL - DEBUG - 46248 - ".../init_app.py:36" - load_pgp_public_keys() -  - Finish load_pgp_public_keys
2026-07-02 23:39:49,820 - SL - INFO - 46248 - ".../init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain

$ docker exec sl-db psql -U myuser -d simplelogin -c "SELECT id, domain FROM public_domain;"
 id |  domain
----+----------
  1 | sl.local
(1 row)
```

The database is now migrated (77 tables, alembic head `32f25cbf12f6`) and initialized (`add_sl_domains()` [init_app.py:39] seeded `public_domain` with `sl.local`).

### 3.2 The five process entry points

`CONTRIBUTING.md` names the process entry points: `wsgi.py and server.py: the webapp` [CONTRIBUTING.md:145], `email_handler.py: the email handler` [CONTRIBUTING.md:146], and `cron.py: the cronjob` [CONTRIBUTING.md:147]; the repository additionally ships `job_runner.py` and `event_listener.py`. Each was started via its real process and its startup output + an independent bind/liveness check captured. The canonical run order per the README is `flask db upgrade` [README.md:433] → `python init_app.py` [README.md:448] → `python email_handler.py` [README.md:480] → `python job_runner.py` [README.md:495].

#### 3.2.1 Webapp — `python server.py` (port 7777) — REQUIRED

Startup stdout:

```
$ python server.py
>>> URL: http://localhost:7777
...
>>> init logging <<<
2026-07-02 23:40:05,964 - SL - DEBUG - 46545 - ".../app/utils.py:17" - <module>() -  - load words file: .../local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
```

Independent readiness proof — the health route `return "success", 200` [server.py:214-215]:

```
$ curl -s -w " HTTP %{http_code}\n" http://localhost:7777/health
success HTTP 200
```

`(inferred)` In production this same app is served by gunicorn — `CMD ["gunicorn","wsgi:app","-b","0.0.0.0:7777","-w","2","--timeout","15"]` [Dockerfile:47] — where `wsgi.py` is `from server import create_app` then `app = create_app()`. Either way it binds **:7777**.

#### 3.2.2 Email handler — `python email_handler.py` (port 20381) — REQUIRED

Startup stdout (the readiness log lines):

```
$ python email_handler.py
...
2026-07-02 23:40:23,488 - SL - INFO - 46831 - ".../email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-02 23:40:23,489 - SL - DEBUG - 46831 - ".../email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

These come from `LOG.i("Listen for port %s", args.port)` [email_handler.py:2403] and `LOG.d("Start mail controller %s %s", controller.hostname, controller.port)` [email_handler.py:2386]; the default port is argparse `default=20381` [email_handler.py:2399] and the controller binds `hostname="0.0.0.0"` [email_handler.py:2383]. Independent bind proof — a real TCP connection to :20381 returns the SMTP greeting, then `QUIT` is accepted:

```
$ python - <<'PY'
import socket
s=socket.socket(); s.settimeout(5); s.connect(("127.0.0.1",20381))
print("SMTP BANNER:", repr(s.recv(1024).decode().strip()))
s.sendall(b"QUIT\r\n"); print("QUIT REPLY:", repr(s.recv(1024).decode().strip())); s.close()
PY
SMTP BANNER: '220 reverse-code-generator-df363dc0-wcbzb Python SMTP 1.4.2'
QUIT REPLY: '221 Bye'
```

The `220` greeting proves it is bound and accepting connections; `Python SMTP 1.4.2` is the aiosmtpd version (matching §1.1).

#### 3.2.3 Job runner — `python job_runner.py` (no port) — AUXILIARY

Startup stdout (only the import banner; then it polls the `Job` table):

```
$ python job_runner.py
>>> URL: http://localhost:7777
...
>>> init logging <<<
2026-07-02 23:40:39,248 - SL - DEBUG - 47103 - ".../app/utils.py:17" - <module>() -  - load words file: .../local_data/test_words.txt
# (process stays alive; no further output on an empty Job table)
```

`(inferred)` It prints nothing more here because it logs `LOG.d("Take job %s", job)` only when a job exists; its `__main__` is a `while True:` loop that, inside `create_light_app().app_context()`, calls `get_jobs_to_run()` [job_runner.py:307] and `process_job(job)` [job_runner.py:188] then `time.sleep(10)`. It binds **no port** — it is a background job processor.

#### 3.2.4 Event listener — `python event_listener.py` (no port; requires a subcommand) — AUXILIARY

It requires a subcommand. With none, it prints a usage error and exits non-zero:

```
$ python event_listener.py; echo "exit=$?"
...
>>> init logging <<<
2026-07-02 23:41:07,186 - SL - DEBUG - 47370 - ".../app/utils.py:17" - <module>() -  - load words file: .../local_data/test_words.txt
Invalid usage. Pass a valid subcommand as argument
exit=1
```

With the `listener` subcommand it starts the PostgreSQL LISTEN/NOTIFY consumer (source `postgresql://myuser:mypassword@localhost:15432/simplelogin`, since `EVENT_LISTENER_DB_URI = os.environ.get("EVENT_LISTENER_DB_URI", DB_URI)` [app/config.py:637]):

```
$ python event_listener.py listener
...
2026-07-02 23:41:08,379 - SL - INFO - 47377 - ".../event_listener.py:34" - main() -  - Using PostgresEventSource
2026-07-02 23:41:08,383 - SL - INFO - 47377 - ".../event_listener.py:43" - main() -  - Starting with HttpEventSink
2026-07-02 23:41:08,383 - SL - INFO - 47377 - ".../events/event_source.py:49" - __listen() -  - Starting to listen to events
```

It binds **no TCP port** (it consumes PostgreSQL events); its modes are `DEAD_LETTER = "dead_letter"` and `LISTENER = "listener"` [event_listener.py:16-17].

#### 3.2.5 Cron — `cron.py` (no port; scheduler-driven, not a daemon) — AUXILIARY

`cron.py` is not a persistent daemon: it is invoked per-job by a scheduler. `crontab.yml` runs it as, e.g., `command: python /code/cron.py -j stats` with `schedule: "0 0 * * *"`; it takes a `-j`/`--job` argument [cron.py:1265-1267] and exits after doing that one job. `crontab.yml` schedules 15 such jobs (e.g. `stats`, `delete_old_monitoring`, `check_custom_domain`, `check_hibp`, `notify_hibp`, `delete_logs`, `delete_old_data`, `poll_apple_subscription`, `notify_trial_end`, `notify_manual_subscription_end`, `notify_premium_end`, `delete_scheduled_users`, `send_undelivered_mails`, `clear_alias_audit_log`, `clear_user_audit_log`).

### 3.3 Which services must run "for the system to work"

**Answer:** the two services that must be running are the **webapp (`python server.py`, or `gunicorn wsgi:app` in production) on :7777** and the **email handler (`python email_handler.py`) on :20381**.

This is grounded both in the observed binds above and in the project's own statement: `SimpleLogin backend consists of 2 main components:` [CONTRIBUTING.md:14] — `- the webapp ...` [CONTRIBUTING.md:16] and `- the email handler ...` [CONTRIBUTING.md:18].

**Cause → effect, per service:**

- **Webapp (`server.py`/`wsgi.py`) — REQUIRED.** It is the HTTP tier used by the web app, browser extensions, OAuth clients and mobile apps [CONTRIBUTING.md:16]; without it there is no UI/API. Proven ready by `GET /health → success/200` (§3.2.1).
- **Email handler (`email_handler.py`) — REQUIRED.** It implements alias email forwarding and sending [CONTRIBUTING.md:18] — SimpleLogin's core function. Proven ready by the `220` SMTP banner on :20381 (§3.2.2).
- **Job runner (`job_runner.py`) — AUXILIARY.** A background poller of the `Job` table (async work such as onboarding emails, deletions); the system serves requests and forwards mail without it. No port (§3.2.3).
- **Event listener (`event_listener.py`) — AUXILIARY.** A PostgreSQL event consumer (event fan-out); needs an explicit `listener`/`dead_letter` subcommand and binds no port (§3.2.4).
- **Cron (`cron.py`) — AUXILIARY.** Scheduled maintenance invoked by a scheduler per `crontab.yml`, not a persistent process (§3.2.5).

### 3.4 Reproducibility

The two required services were restarted and re-verified:

```
$ python server.py    # run #2
$ curl -s -w " HTTP %{http_code}" http://localhost:7777/health
success HTTP 200

$ python email_handler.py    # run #2
SMTP BANNER: '220 reverse-code-generator-df363dc0-wcbzb Python SMTP 1.4.2'
2026-07-02 23:41:53,337 - SL - INFO - 47672 - ".../email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-02 23:41:53,339 - SL - DEBUG - 47672 - ".../email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

Both reproduced identically (webapp `success`/200; email handler same `220 ... Python SMTP 1.4.2` banner and the same `Listen for port 20381` / `Start mail controller 0.0.0.0 20381` lines).


---

## 4. Q3 — Skipped-initialization rejection

### 4.1 Provision "migrated but not initialized" and verify the SLDomain table is empty

Run migrations but **skip `init_app.py`** (and skip `flask dummy-data`, which would also seed the table via `add_sl_domains()`). The SLDomain model is physically the `public_domain` table — `class SLDomain(Base, ModelMixin)` [app/models.py:3116] with `__tablename__ = "public_domain"` [app/models.py:3119].

Command and output — verify **the SLDomain table should be empty (no email domains configured)**:

```
$ docker exec sl-db psql -U myuser -d simplelogin -c 'drop schema public cascade; create schema public;' >/dev/null
$ alembic upgrade head   # migrations only; init_app.py and flask dummy-data NOT run
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

The `public_domain` table exists (migrations created its schema) but has **0 rows** — confirming the precondition. This directly contrasts with Q2, where `python init_app.py` seeded it with `sl.local` (§3.1). Independent confirmation that migrations alone never seed it: immediately after `alembic upgrade head` in Q2 (before running `init_app.py`), `SELECT count(*) FROM public_domain;` was also `0`.

### 4.2 Start the email handler and inject a real message to `xyz@sl.local`

Start the real live listener:

```
$ python email_handler.py
...
2026-07-02 23:43:17,162 - SL - INFO - 49155 - ".../email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-02 23:43:17,163 - SL - DEBUG - 49155 - ".../email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

Inject a real message through the **live aiosmtpd listener** on `localhost:20381` using `smtplib` (this exercises the real `handle_DATA` path — no handler function is called directly). The client script:

```
import smtplib
from email.message import EmailMessage
msg = EmailMessage()
msg["From"] = "external-sender@example.com"
msg["To"] = "xyz@sl.local"
msg["Subject"] = "Q3 skipped-init rejection test"
msg.set_content("This is a test message injected via the live aiosmtpd listener.")
s = smtplib.SMTP("localhost", 20381, timeout=15)
s.set_debuglevel(1)
s.sendmail("external-sender@example.com", ["xyz@sl.local"], msg.as_string())
```

### 4.3 The SMTP status code returned to the sender: `550 SL E515 Email not exist`

Claim: the email handler returns SMTP **`550 SL E515 Email not exist`** to the sender.

Captured raw SMTP wire (from `smtplib` debug output) — note the rejection is delivered at **end-of-DATA** (RCPT TO first returns `250 OK`):

```
send: 'mail FROM:<external-sender@example.com> size=256\r\n'
reply: b'250 OK\r\n'
send: 'rcpt TO:<xyz@sl.local>\r\n'
reply: b'250 OK\r\n'
send: 'data\r\n'
reply: b'354 End data with <CR><LF>.<CR><LF>\r\n'
send: b'From: external-sender@example.com\r\nTo: xyz@sl.local\r\n...\r\n.\r\n'
reply: b'550 SL E515 Email not exist\r\n'
reply: retcode (550); Msg: b'SL E515 Email not exist'
```

Parsed by the client:

```
SMTP_CODE: 550
SMTP_ERROR: SL E515 Email not exist
```

The literal `550 SL E515 Email not exist` is the constant `E515 = "550 SL E515 Email not exist"` [app/email/status.py:51].

### 4.4 The rejection log lines (verbatim)

Claim: the handler logs the alias-does-not-exist attempt and then the cannot-create-on-the-fly rejection.

Captured handler stdout for the injected message (message-id `84aa14e4-...`), verbatim:

```
2026-07-02 23:43:43,593 - SL - INFO - 49155 - ".../email_handler.py:2343" - _handle() - 84aa14e4-... - New message, mail from external-sender@example.com, rctp tos ['xyz@sl.local']
2026-07-02 23:43:43,736 - SL - DEBUG - 49155 - ".../email_handler.py:2202" - handle() - 84aa14e4-... - Forward phase external-sender@example.com(external-sender@example.com) -> xyz@sl.local
2026-07-02 23:43:43,749 - SL - DEBUG - 49155 - ".../email_handler.py:545" - handle_forward() - 84aa14e4-... - alias xyz@sl.local not exist. Try to see if it can be created on the fly
2026-07-02 23:43:43,760 - SL - INFO - 49155 - ".../app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - 84aa14e4-... - Cannot auto-create custom domain alias for xyz@sl.local because there's no custom domain for sl.local
2026-07-02 23:43:43,761 - SL - INFO - 49155 - ".../app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - 84aa14e4-... - Cannot auto-create xyz@sl.local since it has no directory separator
2026-07-02 23:43:43,761 - SL - DEBUG - 49155 - ".../email_handler.py:551" - handle_forward() - 84aa14e4-... - alias xyz@sl.local cannot be created on-the-fly, return 550
2026-07-02 23:43:43,762 - SL - INFO - 49155 - ".../email_handler.py:2367" - _handle() - 84aa14e4-... - Finish mail_from external-sender@example.com, rcpt_tos ['xyz@sl.local'], takes 0.16880083084106445 seconds with return code '550 SL E515 Email not exist'<<===
```

The two decisive rejection lines are `alias xyz@sl.local not exist. Try to see if it can be created on the fly` (the log at [email_handler.py:546]) and `alias xyz@sl.local cannot be created on-the-fly, return 550` (`LOG.d("alias %s cannot be created on-the-fly, return 550", alias_address)` [email_handler.py:551]). The final `_handle()` line confirms the returned reply is `'550 SL E515 Email not exist'`.

### 4.5 Both status-code variants of the rejection (E515 vs E207)

The reject branch itself chooses between two status codes [email_handler.py:552-555]:

```python
if should_ignore_bounce(envelope.mail_from):
    return [(True, status.E207)]
else:
    return [(False, status.E515)]
```

- **Primary (observed): `status.E515` = `"550 SL E515 Email not exist"`** [app/email/status.py:51] → SMTP `550`. This is what a normal external sender receives — and is exactly what was observed in §4.3.
- **Edge variant: `status.E207` = `"250 SL E207 No bounce report"`** [app/email/status.py:12] → SMTP `250`. This is returned only when `should_ignore_bounce(envelope.mail_from)` is true. That function is `def should_ignore_bounce(mail_from: str)` [app/email_utils.py:1361] and returns True only if `IgnoreBounceSender.get_by(mail_from=mail_from)` matches. A plain external sender (`external-sender@example.com`) is **not** an ignore-bounce sender, so the branch takes the `else` and returns `E515`. This is why `550 ... E515` was observed rather than `250 ... E207`.

### 4.6 Cause → effect: the true role of the empty SLDomain / `public_domain` table

Claim: for a **plain** `xyz@sl.local` recipient, the `550` arises because **no alias exists and none can be auto-created** — and this is **independent of the `public_domain` (SLDomain) table being empty**.

Traced through `handle_forward()`: `alias = Alias.get_by(email=alias_address)` [email_handler.py:543] returns nothing → it logs the "not exist" line [email_handler.py:546] → `alias = try_auto_create(alias_address)` [email_handler.py:549; app/alias_utils.py:202] returns `None` → it logs "cannot be created on-the-fly, return 550" [email_handler.py:551] → returns `E515`. `try_auto_create` returns `None` because **both** auto-create strategies fail, exactly as the logs in §4.4 show:

- **Via custom domain** — `try_auto_create_via_domain()` [app/alias_utils.py:274] fails: there is no `CustomDomain` for `sl.local` (log at [app/alias_utils.py:104]: "... because there's no custom domain for sl.local"). This consults the `custom_domain` table, not `public_domain`.
- **Via directory** — `try_auto_create_directory()` [app/alias_utils.py:227] fails: the plain address has no directory separator (log at [app/alias_utils.py:165]: "... since it has no directory separator").

Crucially, the directory *gate* `can_create_directory_for_address()` [app/email_utils.py:545] iterates **`config.ALIAS_DOMAINS`** (`for domain in config.ALIAS_DOMAINS:` [app/email_utils.py:548]) — an **env-derived** value, **not** the `SLDomain` table. Since `config.ALIAS_DOMAINS == ['sl.local']` (from `EMAIL_DOMAIN`, §1.3), that gate actually *passes* for `xyz@sl.local`; the directory strategy still fails only because there is no separator in `xyz`. So the empty `public_domain` table is **not** what produced this `550`.

What the `public_domain` (SLDomain) table *does* gate are **other** code paths — not the plain-forward auto-create path:

- `is_valid_alias_address_domain()` [app/email_utils.py:557] returns True when `SLDomain.get_by(domain=domain)` matches [app/email_utils.py:560].
- The reply / reverse-alias domain lookup uses `sl_domain: SLDomain = SLDomain.get_by(domain=reply_domain)` [email_handler.py:978].

`init_app.py`'s `add_sl_domains()` [init_app.py:39] is what *would* populate `public_domain` from `ALIAS_DOMAINS`; skipping it is why the table is empty. But for this **plain-address forward**, the `550 SL E515 Email not exist` is caused by the missing, non-auto-creatable **alias** — not by the empty `public_domain` table per se.

### 4.7 Reproducibility

The injection was performed three times (senders `external-sender@example.com` twice and `someoneelse@test.org` once). Every run returned the same SMTP code and log line:

```
# run #2 (external-sender@example.com) and run #3 (someoneelse@test.org):
SMTP_CODE: 550
SMTP_ERROR: SL E515 Email not exist
reply: b'550 SL E515 Email not exist\r\n'

2026-07-02 23:45:04,133 - SL - DEBUG - 49155 - ".../email_handler.py:551" - handle_forward() - e1089d7f-... - alias xyz@sl.local cannot be created on-the-fly, return 550
2026-07-02 23:45:04,134 - SL - INFO - 49155 - ".../email_handler.py:2367" - _handle() - e1089d7f-... - Finish mail_from external-sender@example.com, rcpt_tos ['xyz@sl.local'], takes 0.01825237274169922 seconds with return code '550 SL E515 Email not exist'<<===
2026-07-02 23:45:04,214 - SL - DEBUG - 49155 - ".../email_handler.py:551" - handle_forward() - 3074f0c4-... - alias xyz@sl.local cannot be created on-the-fly, return 550
2026-07-02 23:45:04,214 - SL - INFO - 49155 - ".../email_handler.py:2367" - _handle() - 3074f0c4-... - Finish mail_from someoneelse@test.org, rcpt_tos ['xyz@sl.local'], takes 0.017209529876708984 seconds with return code '550 SL E515 Email not exist'<<===
```

All three reproduced `550 SL E515 Email not exist` with the identical `email_handler.py:551` rejection log line.

---

## 5. Read-only scope confirmation

The investigation modified no existing repository file. All transient artifacts (observation scripts, the working `.env`, and the three PostgreSQL states) live outside the tracked source tree or are gitignored (`.env` and `.venv` are in `.gitignore`), and the temporary scripts were removed. After the investigation, the working tree contained only the new deliverable:

```
$ git status --porcelain
?? blitzy/
```

(`blitzy/` is the newly added directory holding only this document, `blitzy/documentation/app_2cd6ee777f8c.md`; no existing file shows as modified.)

---

## 6. Coverage-pass checklist

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

### Summary answers

- **Q1:** `python server.py` starts against an empty database (import-time `engine.connect()` [app/db.py:12] succeeds) and even serves `GET /auth/login`; submitting the login form (POST) runs `User.get_by(...)` [app/auth/views/login.py:43] and raises **`sqlalchemy.exc.ProgrammingError` wrapping `psycopg2.errors.UndefinedTable`: `relation "users" does not exist`** (first failing relation `users`), which the global `@app.errorhandler(Exception)` [server.py:389-394] logs via `LOG.e(e)` and renders as `error/500.html` with **HTTP 500**.
- **Q2:** two services must run for the system to work — the **webapp** (`python server.py` / `gunicorn wsgi:app`, :7777) and the **email handler** (`python email_handler.py`, :20381); `job_runner.py`, `event_listener.py`, and `cron.py` are auxiliary.
- **Q3:** with `public_domain` empty, a message to `xyz@sl.local` is rejected with **`550 SL E515 Email not exist`** and the log line **`alias xyz@sl.local cannot be created on-the-fly, return 550`** — caused by "no alias exists and none can be auto-created", independent of the empty `SLDomain`/`public_domain` table.

