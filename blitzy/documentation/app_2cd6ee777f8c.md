# SimpleLogin — First-Time Local Bring-Up: Observed Runtime Behavior (Q1–Q3)

This document answers three concrete runtime-behavior questions about the **SimpleLogin** Flask
monolith, based on **actually running the stack** and capturing verbatim output. It is a read-only
investigation: no source file was modified or deleted, and the only permanent artifact is this
document.

Each behavioral claim below is paired with the specific observed output line that demonstrates it and
the command or code that produced it, followed by an exact `file:line` citation into the source tree.

## How this was observed

**Runtime and versions**

- **Python 3.10.20.** The project requires Python 3.10 and explicitly notes that 3.12 does not work:
  <br>`CONTRIBUTING.md:236` → `# we haven't managed to make python 3.12 work`
  <br>Corroborated by `pyproject.toml:61` → `python = "^3.10"`, `Dockerfile:8` → `FROM python:3.10`,
  and `.github/workflows/main.yml:40` → `python-version: ["3.10"]`.
- **PostgreSQL 13.23** (CI parity: `.github/workflows/main.yml:47` → `image: postgres:13`). PostgreSQL
  is a hard dependency — the app uses PostgreSQL-specific features, so SQLite is not a substitute.
- Isolated virtualenv plus a local `.env` copied from `example.env`. The `.env` is gitignored
  (`.gitignore:4` → `.env`), so it never appears in `git status`.
- Behavior-critical dependencies at the exact `poetry.lock` versions: Flask 1.1.2, Werkzeug 1.0.1,
  SQLAlchemy 1.3.24, psycopg2-binary 2.9.3, Flask-Login 0.5.0, Flask-Migrate 2.5.3, alembic 1.4.3,
  aiosmtpd 1.4.2, dnspython 2.0.0, email-validator 1.1.3.

**Three database states**

The three questions require three distinct database states, staged by resetting between runs with
`drop schema public cascade; create schema public;` (the reset pattern used by
`scripts/reset_local_db.sh`):

| State | How reached | Used for |
|-------|-------------|----------|
| A — empty / un-migrated | schema present, **no tables** | Q1 |
| B — migrated + initialized | `alembic upgrade head` **+** `python init_app.py` | Q2 |
| C — migrated only | `alembic upgrade head` **alone** (skip `init_app.py`) | Q3 |

**Honest deviations (stated for transparency; none affects the Q1/Q2/Q3 outcomes)**

- (a) `pyre2` (the C++ RE2 binding) could not build in the observation environment, so a venv-only
  `re2.py` shim proxying the standard-library `re` module was used (`import re2 as re`; API-compatible).
  No regex code path is exercised by any of the three answers.
- (b) PostgreSQL listened on port **5433** in the observation environment, so the local `.env` `DB_URI`
  used 5433 instead of `example.env`'s 5432 (`example.env:75` shows `...localhost:5432/simplelogin`).
- (c) `aiospamc==0.10.0` had to be installed for `email_handler.py` to import (`app/email/spam.py`
  imports it).
- (d) A couple of runtime-irrelevant version pins resolved to the nearest available wheel.

**Cleanup**

All reproduction artifacts are ephemeral and outside the repository (a `/tmp` virtualenv, a temporary
SMTP-injection script, and the gitignored `.env`); the running `email_handler.py` and PostgreSQL were
stopped afterward. `git status` shows only the new `blitzy/documentation/app_2cd6ee777f8c.md`.

---

## Q1 — Empty-database login failure

**Question.** With PostgreSQL running and an empty database (schema present, **no tables** — Alembic
migrations **not** applied), start `python server.py`, open the login page, and capture the full
verbatim Python exception/traceback.

### Precondition

- **Empty DB confirmed.** `psql \dt` reported "Did not find any relations" (0 tables).
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
- **Server launched via `local_main()`** (`server.py:572-588`), which sets `app.debug = True` and calls
  `app.run(debug=True, port=7777)` at `server.py:588`.

### Observed HTTP behavior (one claim → one evidence)

- **`GET /` → `HTTP/1.0 302 FOUND`**, `Location: http://127.0.0.1:7777/auth/login`. The index route
  redirects anonymous visitors to `auth.login`:

```
GET / -> HTTP/1.0 302 FOUND
Location: http://127.0.0.1:7777/auth/login
```

  Source: `server.py:251` `def index():` → `server.py:255` `return redirect(url_for("auth.login"))`.

- **`GET /auth/login` → HTTP 200** (~340871 bytes), **no DB query, no error.** **KEY FINDING: opening
  the login page does NOT raise on an empty DB** — the anonymous form renders fine.

```
GET /auth/login -> HTTP 200 (~340871 bytes)  # login form renders; no exception
```

  Reason: the `auth_bp` blueprint has no `before_request` hook (`app/auth/base.py:1-5`), and the
  template `templates/auth/login.html` carries no anonymous DB-backed data.

- **`POST /auth/login`** (form fields `csrf_token` + `email=john@wick.com` + `password=password`) **→
  HTTP 500.** **This is the request that triggers the exception.**

```
POST /auth/login  (email=john@wick.com, password=password) -> HTTP 500
```

### The exact exception (verbatim, captured from the server console under `debug=True`)

```
psycopg2.errors.UndefinedTable: relation "users" does not exist
LINE 2: FROM users
             ^

The above exception was the direct cause of the following exception:

sqlalchemy.exc.ProgrammingError: (psycopg2.errors.UndefinedTable) relation "users" does not exist
[SQL: SELECT users.directory_quota ... FROM users WHERE users.email = %(email_1)s LIMIT %(param_1)s]
[parameters: {'email_1': 'john@wick.com', 'param_1': 1}]
(Background on this error at: http://sqlalche.me/e/13/f405)
```

**Application call site (verbatim frames):**

```
File ".../app/auth/views/login.py", line 43, in login
    user = User.get_by(email=email) or User.get_by(email=canonical_email)
File ".../app/models.py", line 84, in get_by
    return Session.query(cls).filter_by(**kw).first()
```

### Answer

- **Exception class = `sqlalchemy.exc.ProgrammingError`** wrapping **`psycopg2.errors.UndefinedTable`**,
  message **`relation "users" does not exist`**.
- **Triggered on the POST** login submission (not the GET). The failing ORM call is at
  `app/auth/views/login.py:43`
  (`user = User.get_by(email=email) or User.get_by(email=canonical_email)`), which delegates to
  `app/models.py:84` (`return Session.query(cls).filter_by(**kw).first()`). The `login.py:43` call sits
  inside `if form.validate_on_submit():` (`app/auth/views/login.py:40`), i.e. the form-submit / POST
  path.
- The `http://sqlalche.me/e/13/f405` URL in the message confirms **SQLAlchemy 1.3**.
- Because `python server.py` runs with `debug=True` (`server.py:588`), the request returns HTTP 500 and
  the raw traceback prints to the server console/stderr; the generic `@app.errorhandler(Exception)`
  (`server.py:388` → `server.py:394` `return render_template("error/500.html"), 500`) is bypassed under
  the Flask debugger.
- **Precise nuance:** an empty database does **not** break app import or the login-page GET; it breaks
  the **first ORM query against the missing `users` table**, which happens when credentials are
  submitted.

---

## Q2 — Required Python services and startup / port-binding evidence

**Question.** After `flask db upgrade` and `python init_app.py` (system correctly set up), determine
which Python services must be running, start each, and capture the exact stdout/stderr startup output
confirming each is running and ready — including proof that services actually bind their ports.

### Setup (State B — migrated + initialized)

- DB reset, then `alembic upgrade head` (equivalent to `flask db upgrade`, `README.md:433`) — created
  **77 public tables**; `to_regclass('public.users')` and `to_regclass('public.public_domain')` were
  both non-null.
- Then `python init_app.py` (`README.md:448`) logged the domain seed:

```
Add sl.local to SL domain
```

  Source: `init_app.py:44` `LOG.i("Add %s to SL domain", alias_domain)`; `add_sl_domains()` is defined
  at `init_app.py:39` and called at `init_app.py:73`, creating rows via `init_app.py:45`
  `SLDomain.create(domain=alias_domain, use_as_reverse_alias=True)`. Afterward `public_domain` held one
  row:

```
 id | domain   | use_as_reverse_alias
----+----------+----------------------
  1 | sl.local | t
```

### Answer: the system requires THREE long-lived Python services

These are the README self-hosting core trio (`README.md:433-495`). For each, the readiness claim is
paired with observed proof — readiness is **demonstrated**, not asserted.

#### [1/3] Webapp — `python server.py`

Dev server, `debug=True`, port **7777**. The production equivalent is
`gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` (`Dockerfile:47`), where `wsgi.py` is simply
`from server import create_app` / `app = create_app()`.

- **Readiness proof (functional):** `curl http://127.0.0.1:7777/health` → body `success`, HTTP 200.

```
$ curl -s -o /dev/body -w '%{http_code}\n' http://127.0.0.1:7777/health
200
$ curl -s http://127.0.0.1:7777/health
success
```

  Source: `server.py:213` `@app.route("/health", methods=["GET"])` → `server.py:215`
  `return "success", 200`.

- **Port proof:** a socket probe of port 7777 returns `0` (open / listening).

```
socket.connect_ex(("127.0.0.1", 7777)) -> 0
```

- **Honest note:** with a `WERKZEUG_RUN_MAIN=true` workaround the Werkzeug `Running on http://...`
  banner was suppressed, so readiness is proven via `/health` + the socket check rather than that
  banner.

#### [2/3] Email handler — `python email_handler.py`

An `aiosmtpd`-based SMTP server on port **20381**.

- **Startup log (verbatim; the `<pid>` and leading timestamp vary by run):**

```
... - SL - INFO - <pid> - ".../email_handler.py:2403" - <module>() -  - Listen for port 20381
... - SL - DEBUG - <pid> - ".../email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

  Sources: `email_handler.py:2403` `LOG.i("Listen for port %s", args.port)`; `email_handler.py:2383`
  `controller = Controller(MailHandler(), hostname="0.0.0.0", port=port)`; `email_handler.py:2386`
  `LOG.d("Start mail controller %s %s", controller.hostname, controller.port)`; default port 20381
  (argparse `default=20381`).

- **Port proof + SMTP-speaking proof:** a socket probe of 20381 returns `0` (open); the server emits an
  SMTP banner and answers `EHLO`.

```
socket.connect_ex(("127.0.0.1", 20381)) -> 0
BANNER -> 220 ... Python SMTP 1.4.2
EHLO   -> 250 (multiline)
```

- **Honest note:** the first start crashed with `ModuleNotFoundError: No module named 'aiospamc'` until
  `aiospamc` was installed — an environment deviation only (see preamble), not a defect in the
  required-services answer.

#### [3/3] Job runner — `python job_runner.py`

A background worker. **It has no port and prints no startup banner by design** — it enters a poll loop:
`while True: ... time.sleep(10)` (`job_runner.py:330` and `job_runner.py:347`).

- **Readiness proof #1:** the process stays alive after entering the poll loop.
- **Readiness proof #2 (functional):** a ready row was inserted into the `job` table; within one 10 s
  poll the runner logged (verbatim):

```
Take job <Job 2 test-job {}>
Unknown job name test-job
```

  and the row advanced to `taken=t, state=2, attempts=1`. This proves the loop is live and querying the
  `job` table. Sources: `job_runner.py:334` `LOG.d("Take job %s", job)`; `job_runner.py:304`
  `LOG.e("Unknown job name %s", job.name)`.

### Cross-service database proof

Each of the three service processes opens one PostgreSQL backend named `webapp`:

```
select application_name, count(*) from pg_stat_activity where datname='simplelogin' group by 1
--> webapp = 3
```

`DB_CONN_NAME` defaults to `"webapp"` (`app/config.py:193`
`DB_CONN_NAME = os.environ.get("DB_CONN_NAME", "webapp")`), and `app/db.py` opens an import-time
connection with `connect_args={"application_name": config.DB_CONN_NAME}` (`app/db.py:10`;
`connection = engine.connect()` at `app/db.py:12`). So each service process holds one `webapp` backend;
the three `backend_start` times matched the staggered service launches.

### Answer

Required services = **webapp (`server.py` / `gunicorn wsgi:app`, port 7777)**,
**`email_handler.py` (port 20381)**, and **`job_runner.py` (no port)**.

`cron.py` and `event_listener.py` are the 4th and 5th processes in the full 5-process Docker superset,
but they are **not** in the README's core run list (`README.md:433-495`) and are not needed for the
login/email flows exercised in Q1/Q3.

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
- **Empty precondition verified immediately before injection:**

```
select count(*) from public_domain;   -> 0
select count(*) from custom_domain;    -> 0
select count(*) from alias;            -> 0
```

  So the `SLDomain` table — physically named `public_domain` (`app/models.py:3116`
  `class SLDomain(Base, ModelMixin):`, `app/models.py:3119` `__tablename__ = "public_domain"`) — is
  **empty**.

### Action

Started `python email_handler.py` on this migrated-only DB (`Listen for port 20381` confirmed as in
Q2), then injected a message to an `@sl.local` address via a temporary Python `smtplib` script mirroring
the documented command:

```bash
swaks --to e1@sl.local --from hey@google.com --server 127.0.0.1:20381
```

Source: `CONTRIBUTING.md:218` (the fenced block opens at `CONTRIBUTING.md:217`). The sender
`hey@google.com` is an **ordinary** address (not a bounce address), so the E515 path applies (see the
bounce nuance below).

### (a) SMTP status code returned to the sender (verbatim transcript)

```
BANNER            -> 220 reverse-file-mapper-... Python SMTP 1.4.2
EHLO              -> 250
MAIL FROM <hey@google.com> -> 250 OK
RCPT TO <e1@sl.local>   -> 250 OK
DATA (end-of-data)-> 550 SL E515 Email not exist
```

- The exact reply is **`550 SL E515 Email not exist`**, returned at **end-of-DATA**.
- **RCPT is accepted (`250 OK`)** because `MailHandler` (`email_handler.py:2288` `class MailHandler:`)
  defines only `handle_DATA` (`email_handler.py:2289` `async def handle_DATA(...)`) and **no**
  `handle_RCPT`, so `aiosmtpd` accepts the recipient by default and the rejection happens after the
  message body is received.

### (b) What it logs to explain the rejection (verbatim ordered trail)

Each `file:line` anchor below is the opening line of the LOG statement that produced the message:

```
email_handler.py:2343 _handle() : New message, mail from hey@google.com, rctp tos ['e1@sl.local']
email_handler.py:2202 handle()  : Forward phase hey@google.com(hey@google.com) -> e1@sl.local
email_handler.py:545  handle_forward() : alias e1@sl.local not exist. Try to see if it can be created on the fly
app/alias_utils.py:104 check_if_alias_can_be_auto_created_for_custom_domain() : Cannot auto-create custom domain alias for e1@sl.local because there's no custom domain for sl.local
app/alias_utils.py:165 check_if_alias_can_be_auto_created_for_a_directory() : Cannot auto-create e1@sl.local since it has no directory separator
email_handler.py:551  handle_forward() : alias e1@sl.local cannot be created on-the-fly, return 550
email_handler.py:2367 _handle() : Finish mail_from hey@google.com, rcpt_tos ['e1@sl.local'], takes 0.18... seconds with return code '550 SL E515 Email not exist'<<===
```

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
  (`select count(*) from public_domain; -> 0`).
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

