# SimpleLogin Local Development Environment — Startup Behavior Exploration Guide

## Introduction

This document answers three hands-on, exploratory questions about the **SimpleLogin local development environment startup process**. Each answer is derived entirely from the source code — every claim cites specific files and line numbers as evidence. No assumptions or typical-pattern references are used.

### Approach

Each question is answered by tracing the exact execution path through the source code, identifying which functions are called, what database queries are issued, and what output is produced. The document follows a progressive-disclosure structure: first the observable behavior is described, then the underlying code trace is provided as rationale.

### Prerequisite Environment

The following environment is assumed, based on `example.env` and `CONTRIBUTING.md`:

- **Python 3.10+** with dependencies installed via `poetry install`
- **PostgreSQL** running on `localhost:5432` with database `simplelogin` created
- **`.env` file** copied from `example.env` with these key values:

| Variable | Value | Source |
|----------|-------|--------|
| `URL` | `http://localhost:7777` | `example.env:6` |
| `NOT_SEND_EMAIL` | `true` | `example.env:19` |
| `EMAIL_DOMAIN` | `sl.local` | `example.env:22` |
| `SUPPORT_EMAIL` | `support@sl.local` | `example.env:40` |
| `DB_URI` | `postgresql://myuser:mypassword@localhost:5432/simplelogin` | `example.env:75` |
| `FLASK_SECRET` | `secret` | `example.env:77` |

The standard local development startup command is documented at `CONTRIBUTING.md:103–109`:

```bash
alembic upgrade head && flask dummy-data && python3 server.py
```

This guide explores what happens when parts of that sequence are skipped or executed out of order.

---

## Question 1: Database-Not-Migrated Error on Login Page

### Setup Context

**Scenario:** PostgreSQL is running and the database `simplelogin` exists, but it is completely empty — no tables, no migrations executed. The user runs `python server.py` and navigates to the login page in a browser.

### Server Startup Behavior

Running `python server.py` invokes the entry point at `server.py:598–599`:

```python
if __name__ == "__main__":
    local_main()
```

Source: `server.py:598–599`

The `local_main()` function at `server.py:572–588` performs these steps:

1. Sets `config.COLOR_LOG = True` (line 573)
2. Calls `create_app()` (line 574) to build the full Flask application with all blueprints, extensions, error handlers, admin panel, CORS, sessions, etc. (defined at `server.py:139–217`)
3. Enables the Flask debug toolbar (lines 577–582)
4. Starts the development server: `app.run(debug=True, port=7777)` (line 588)

Source: `server.py:572–588`

**Database connection at import time:** When `server.py` imports from `app.db` (line 76: `from app.db import Session`), the module `app/db.py` is loaded. At import time, it:

1. Creates the SQLAlchemy engine: `engine = create_engine(config.DB_URI, ...)` (line 9–10)
2. Immediately opens a connection: `connection = engine.connect()` (line 12)
3. Creates a scoped session: `Session = scoped_session(sessionmaker(bind=connection))` (line 14)

Source: `app/db.py:9–14`

**This connection succeeds against an empty database** because establishing a PostgreSQL connection only requires the database to exist — it does not require any tables.

**First visible output:** The `app/log.py` module executes `print(">>> init logging <<<")` at module load time (line 67), which is the first line printed to stdout.

Source: `app/log.py:67`

### Login Page GET Request — Success

Navigating to `http://localhost:7777/` hits the `index()` route at `server.py:250–255`:

```python
@app.route("/", methods=["GET", "POST"])
def index():
    if current_user.is_authenticated:
        return redirect(url_for("dashboard.index"))
    else:
        return redirect(url_for("auth.login"))
```

Source: `server.py:250–255`

For an unauthenticated visitor, `current_user` is Flask-Login's `AnonymousUserMixin` (where `is_authenticated` returns `False`), so the browser is redirected to `/login`.

**Important:** The `@login_manager.user_loader` callback at `server.py:220–230` calls `User.get_by(alternative_id=...)`, but this callback is **only invoked when a valid session cookie is present**. For a first-time visitor with no session cookie, no database query occurs during this check.

Source: `server.py:220–230`

The `/login` route handler at `app/auth/views/login.py:21–82` processes the GET request:

1. `current_user.is_authenticated` (line 28) — returns `False` for unauthenticated visitors
2. `form = LoginForm(request.form)` (line 36) — creates the form object
3. `form.validate_on_submit()` (line 40) — returns `False` for GET requests (no POST data submitted)
4. Falls through to `render_template("auth/login.html", ...)` (lines 74–82)

Source: `app/auth/views/login.py:25–82`

**Conclusion: The login page renders successfully on GET because no database query is executed.** The template is rendered with form variables and configuration flags only.

### Login Page POST Request — The Exception

Submitting the login form sends a POST to `/login`. The handler now follows a different path:

1. `form.validate_on_submit()` (line 40) returns `True` (form data includes email and password)
2. `email = sanitize_email(form.email.data)` (line 41)
3. `canonical_email = canonicalize_email(email)` (line 42)
4. **Critical line 43:** `user = User.get_by(email=email) or User.get_by(email=canonical_email)`

Source: `app/auth/views/login.py:40–43`

This calls `ModelMixin.get_by()` defined at `app/models.py:82–84`:

```python
@classmethod
def get_by(cls, **kw):
    return Session.query(cls).filter_by(**kw).first()
```

Source: `app/models.py:82–84`

This issues a SQL `SELECT` query against the `users` table (because `User.__tablename__ = "users"` at `app/models.py:337`). Since the database has **no tables** (no migrations were executed), this query fails with a PostgreSQL error.

### Full Exception

The exception raised is:

```text
sqlalchemy.exc.ProgrammingError: (psycopg2.errors.UndefinedTable) relation "users" does not exist
LINE 1: SELECT users.id AS users_id, ... FROM users WHERE users.email = ...
```

The call chain producing this exception is:

```text
app/auth/views/login.py:43    user = User.get_by(email=email)
  → app/models.py:84            return Session.query(cls).filter_by(**kw).first()
    → SQLAlchemy                   SELECT ... FROM users WHERE users.email = ?
      → psycopg2                     PostgreSQL: relation "users" does not exist
        → psycopg2.errors.UndefinedTable
      → sqlalchemy.exc.ProgrammingError
```

**What the user sees in the browser:** The generic exception handler at `server.py:388–394` catches all unhandled exceptions:

```python
@app.errorhandler(Exception)
def error_handler(e):
    LOG.e(e)
    if request.path.startswith("/api/"):
        return jsonify(error="Internal error"), 500
    else:
        return render_template("error/500.html"), 500
```

Source: `server.py:388–394`

Since `/login` is not an API path, the browser displays the `error/500.html` template with a **500 Internal Server Error** response. The full exception traceback is logged to the console via `LOG.e(e)` (which is an alias for `logging.Logger.exception`, defined at `app/log.py:77`).

### Rationale

The GET request succeeds because the `/login` route handler only renders a Jinja2 template — no database query is issued. The POST request fails because `User.get_by(email=...)` at `app/auth/views/login.py:43` triggers `Session.query(User).filter_by(email=...).first()` at `app/models.py:84`, which issues a `SELECT` query against the `users` table (`app/models.py:337`). With no tables in the database, PostgreSQL raises `UndefinedTable`, which SQLAlchemy wraps as `ProgrammingError`.

### Sequence Diagram: Login Page Request Lifecycle

```mermaid
sequenceDiagram
    participant B as Browser
    participant F as Flask (server.py)
    participant L as login() (login.py)
    participant M as ModelMixin (models.py)
    participant S as Session (db.py)
    participant P as PostgreSQL

    B->>F: GET /
    F->>B: 302 Redirect → /login

    B->>F: GET /login
    F->>L: login()
    L->>L: form.validate_on_submit() → False (GET)
    L->>F: render_template("auth/login.html")
    F->>B: 200 OK (login page HTML)

    B->>F: POST /login (email + password)
    F->>L: login()
    L->>L: form.validate_on_submit() → True
    L->>M: User.get_by(email=...)
    M->>S: Session.query(User).filter_by(email=...).first()
    S->>P: SELECT ... FROM users WHERE email = ...
    P-->>S: ERROR: relation "users" does not exist
    S-->>M: ProgrammingError (UndefinedTable)
    M-->>L: Exception propagates
    L-->>F: Unhandled exception
    F->>F: errorhandler(Exception) → LOG.e(e)
    F->>B: 500 (error/500.html)
```

---

## Question 2: Required Services and Startup Verification

### Service Inventory

The SimpleLogin local development stack consists of **three Python services**. The repository's `CONTRIBUTING.md:143–147` identifies the main entry points:

> - `wsgi.py` and `server.py`: the webapp
> - `email_handler.py`: the email handler
> - `cron.py`: the cronjob

Source: `CONTRIBUTING.md:143–147`

For **local development**, the three services to run are `server.py`, `email_handler.py`, and `job_runner.py`. The job runner is separately documented at `CONTRIBUTING.md:225–229`:

> Some features require a job handler (such as GDPR data export). To test such feature you need to run the job_runner

Source: `CONTRIBUTING.md:225–229`

All three services share the `create_light_app()` function from `server.py:127–136` for obtaining a Flask application context:

```python
def create_light_app() -> Flask:
    app = Flask(__name__)
    app.config["SQLALCHEMY_DATABASE_URI"] = DB_URI
    app.config["SQLALCHEMY_TRACK_MODIFICATIONS"] = False

    @app.teardown_appcontext
    def shutdown_session(response_or_exc):
        Session.remove()

    return app
```

Source: `server.py:127–136`

### Service 1: Flask Web Server (`server.py`)

| Property | Value |
|----------|-------|
| **Command** | `python server.py` |
| **Entry point** | `server.py:598–599` → `local_main()` at `server.py:572–588` |
| **Port** | `7777` |
| **Bind address** | `127.0.0.1` (Flask's default when no `host` parameter is passed) |

**Startup sequence:**

1. `local_main()` sets `config.COLOR_LOG = True` (line 573)
2. Calls `create_app()` (line 574) which builds the full Flask app with all blueprints, error handlers, admin, CORS, sessions, rate limiting, and more (lines 139–217)
3. Enables Flask debug toolbar (lines 577–582)
4. Calls `app.run(debug=True, port=7777)` (line 588) — only `debug` and `port` are passed; Flask defaults `host` to `127.0.0.1`

Source: `server.py:572–588`

**Expected stdout/stderr output:**

```text
>>> init logging <<<
 * Serving Flask app "app" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
 * Running on http://127.0.0.1:7777/ (Press CTRL+C to quit)
 * Restarting with stat
>>> init logging <<<
 * Debugger is active!
 * Debugger PIN: xxx-xxx-xxx
```

**Note:** The `>>> init logging <<<` message (from `app/log.py:67`) appears twice because Flask's debug mode reloader restarts the process. The werkzeug request logger is disabled at `app/log.py:70–71` (`log = logging.getLogger("werkzeug"); log.disabled = True`), so individual HTTP request logs will not appear in the console. However, the Flask startup banner is printed directly by `app.run()` before the logger is disabled.

Source: `app/log.py:67, 70–71`

### Service 2: Email Handler (`email_handler.py`)

| Property | Value |
|----------|-------|
| **Command** | `python email_handler.py` |
| **Entry point** | `email_handler.py:2396–2404` |
| **Port** | `20381` |
| **Bind address** | `0.0.0.0` |

**Startup sequence:**

1. The `if __name__ == "__main__":` block at `email_handler.py:2396` uses `argparse` with default port `20381` (line 2399)
2. Logs `LOG.i("Listen for port %s", args.port)` (line 2403)
3. Calls `main(port=args.port)` (line 2404)

Source: `email_handler.py:2396–2404`

The `main()` function at `email_handler.py:2381–2393`:

1. Creates an aiosmtpd `Controller`: `Controller(MailHandler(), hostname="0.0.0.0", port=port)` (line 2383)
2. Starts the SMTP server: `controller.start()` (line 2385) — this launches the SMTP server in a separate thread
3. Logs `LOG.d("Start mail controller %s %s", controller.hostname, controller.port)` (line 2386)
4. If `LOAD_PGP_EMAIL_HANDLER` is set in the environment, loads PGP keys (lines 2388–2390) — **not set by default**
5. Enters an infinite keep-alive loop: `while True: time.sleep(2)` (lines 2392–2393)

Source: `email_handler.py:2381–2393`

**Expected stdout output:**

```text
>>> init logging <<<
2025-01-15 10:00:00,000 - SL - INFO - 12345 - "email_handler.py:2403" - <module>() -  - Listen for port 20381
2025-01-15 10:00:00,050 - SL - DEBUG - 12345 - "email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

The log format follows the pattern defined at `app/log.py:12–14`:

```python
_log_format = (
    "%(asctime)s - %(name)s - %(levelname)s - %(process)d - "
    '"%(pathname)s:%(lineno)d" - %(funcName)s() - %(message_id)s - %(message)s'
)
```

Source: `app/log.py:12–14`

The `message_id` field is empty in the startup logs because `set_message_id()` (at `app/log.py:22–25`) is only called during email processing. The `EmailHandlerFilter` at `app/log.py:28–37` sets `record.message_id` to an empty string when no message is being processed.

### Service 3: Job Runner (`job_runner.py`)

| Property | Value |
|----------|-------|
| **Command** | `python job_runner.py` |
| **Entry point** | `job_runner.py:329–347` |
| **Port** | None (no network listener) |

**Startup sequence:**

The `if __name__ == "__main__":` block at `job_runner.py:329` enters an infinite polling loop:

```python
if __name__ == "__main__":
    while True:
        with create_light_app().app_context():
            for job in get_jobs_to_run():
                LOG.d("Take job %s", job)
                # ... process job ...
            time.sleep(10)
```

Source: `job_runner.py:329–347`

Each iteration:

1. Creates a Flask app context via `create_light_app()` (line 332)
2. Queries for pending jobs via `get_jobs_to_run()` (line 333, defined at lines 307–326)
3. Processes each job found
4. Sleeps for 10 seconds: `time.sleep(10)` (line 347)

Source: `job_runner.py:329–347`

**Expected stdout output:**

```text
>>> init logging <<<
```

Then **silence** — unless there are pending jobs in the database. The service produces no output during idle polling because `get_jobs_to_run()` simply returns an empty list, and no logging occurs for empty iterations. When a job is found, it logs `LOG.d("Take job %s", job)` (line 334).

**Confirming the service is running:** The process does not exit — it continuously sleeps 10 seconds between poll cycles. Its liveness is confirmed by observing the process remains active.

### Port Binding Verification

To verify each service is bound to its expected port:

```bash
# Web server — should show python process on port 7777
lsof -i :7777

# Email handler — should show python process on port 20381
lsof -i :20381

# Job runner — no port; verify process is running
ps aux | grep job_runner.py
```

### Rationale

The SimpleLogin local development stack requires exactly three services because the architecture separates concerns across distinct processes:

1. **Why three services?** The web server (`server.py`) handles HTTP requests, the email handler (`email_handler.py`) handles inbound SMTP traffic, and the job runner (`job_runner.py`) processes asynchronous background tasks (e.g., GDPR data export). Each has a dedicated entry point under `if __name__ == "__main__"`, and `CONTRIBUTING.md:143–147` and `CONTRIBUTING.md:225–229` document them as the required local development processes.

2. **`create_app()` vs `create_light_app()` architectural split:** The web server uses `create_app()` (`server.py:139–217`), which builds the full Flask application with all blueprints, error handlers, admin interface, CORS, session management, rate limiting, and the debug toolbar. The email handler and job runner use `create_light_app()` (`server.py:127–136`), which creates a minimal Flask app with only the database session configured — no HTTP routing or web middleware. This split exists because the email handler and job runner only need a Flask application context for SQLAlchemy database access; they do not serve HTTP traffic.

3. **Port binding evidence:** The web server binds to port `7777` via `app.run(debug=True, port=7777)` at `server.py:588`. The email handler binds to port `20381` via `Controller(MailHandler(), hostname="0.0.0.0", port=port)` at `email_handler.py:2383`, with the default port set by `argparse` at `email_handler.py:2399`. The job runner has no port binding — it is a pure polling loop with `time.sleep(10)` at `job_runner.py:347`.

4. **Log output evidence:** All three services share the logging infrastructure from `app/log.py`. The `>>> init logging <<<` message at `app/log.py:67` is printed at import time. The structured log format at `app/log.py:12–14` produces the `<timestamp> - SL - <LEVEL> - <pid> - "<file>:<line>" - <func>() - <message_id> - <message>` pattern seen in the email handler and job runner output. The web server's werkzeug request logger is explicitly disabled at `app/log.py:70–71`.

### Component Diagram: Service Topology

```mermaid
graph TB
    subgraph "SimpleLogin Local Development Stack"
        subgraph "Service 1: Web Server"
            S["server.py<br/>Flask dev server<br/>127.0.0.1:7777"]
        end

        subgraph "Service 2: Email Handler"
            E["email_handler.py<br/>aiosmtpd Controller<br/>0.0.0.0:20381"]
        end

        subgraph "Service 3: Job Runner"
            J["job_runner.py<br/>DB polling loop<br/>No port (sleeps 10s)"]
        end

        subgraph "Shared Infrastructure"
            DB[(PostgreSQL<br/>localhost:5432)]
            APP["create_light_app()<br/>server.py:127–136"]
            LOG_MOD["app/log.py<br/>LOG = _get_logger('SL')"]
        end
    end

    S -->|"create_app() uses<br/>app/db.py Session"| DB
    E -->|"_handle() wraps in<br/>create_light_app().app_context()<br/>email_handler.py:2352"| APP
    J -->|"each iteration wraps in<br/>create_light_app().app_context()<br/>job_runner.py:332"| APP
    APP -->|"configures<br/>SQLALCHEMY_DATABASE_URI"| DB
    S --> LOG_MOD
    E --> LOG_MOD
    J --> LOG_MOD
```

---

## Question 3: Missing `init_app.py` — Email Rejection Behavior

### Setup Context

**Scenario:** Database migrations have been applied (`alembic upgrade head`), so all tables exist. However, `init_app.py` has **NOT** been executed, leaving the `public_domain` table (which backs the `SLDomain` model) empty. The email handler is started, and an incoming email is simulated to `test@sl.local`.

### What `init_app.py` Seeds

The `add_sl_domains()` function at `init_app.py:39–56` populates the `public_domain` table:

```python
def add_sl_domains():
    for alias_domain in ALIAS_DOMAINS:
        if SLDomain.get_by(domain=alias_domain):
            LOG.d("%s is already a SL domain", alias_domain)
        else:
            LOG.i("Add %s to SL domain", alias_domain)
            SLDomain.create(domain=alias_domain, use_as_reverse_alias=True)
    # ... also iterates PREMIUM_ALIAS_DOMAINS (empty by default) ...
    Session.commit()
```

Source: `init_app.py:39–56`

The `ALIAS_DOMAINS` configuration at `app/config.py:157–161`:

```python
if "ALIAS_DOMAINS" in os.environ:
    ALIAS_DOMAINS = sl_getenv("ALIAS_DOMAINS")
else:
    ALIAS_DOMAINS = OTHER_ALIAS_DOMAINS + [EMAIL_DOMAIN]
```

Source: `app/config.py:157–161`

With the default `.env`, `EMAIL_DOMAIN=sl.local` (from `example.env:22`), `OTHER_ALIAS_DOMAINS` is empty, so `ALIAS_DOMAINS = ["sl.local"]`. The `PREMIUM_ALIAS_DOMAINS` list is also empty by default.

Source: `app/config.py:92, 153–154`

The `SLDomain` model is defined at `app/models.py:3116–3119`:

```python
class SLDomain(Base, ModelMixin):
    """SimpleLogin domains"""
    __tablename__ = "public_domain"
```

Source: `app/models.py:3116–3119`

**When `init_app.py` is NOT run, the `public_domain` table is empty — no `SLDomain` records exist for `sl.local`.** This means the system has no database-level awareness of which domains it manages, even though the config variable `ALIAS_DOMAINS` still contains `["sl.local"]`.

### Simulating the Incoming Email

With the email handler running, simulate sending an email:

```bash
swaks --to test@sl.local --from sender@example.com --server 127.0.0.1:20381
```

Source: This test pattern is documented at `CONTRIBUTING.md:209–219`

### Email Handler Flow Trace

#### Step 1: `MailHandler.handle_DATA()`

The aiosmtpd controller calls `MailHandler.handle_DATA()` at `email_handler.py:2289` when a complete message is received. It parses the raw message and calls `self._handle(envelope, msg)` (line 2292).

Source: `email_handler.py:2288–2292`

#### Step 2: `_handle()` — Message Lifecycle Setup

The `_handle()` method at `email_handler.py:2335` performs setup:

1. Generates a unique `message_id` for lifecycle tracking (lines 2339–2340)
2. Logs `"====>=====>====>====>====>====>====>====>"`  (line 2342) — the visual delimiter for a new message
3. Logs `"New message, mail from %s, rctp tos %s"` (lines 2343–2346)
4. Wraps in `create_light_app().app_context()` (line 2352) and calls `handle(envelope, msg)` (line 2353)

Source: `email_handler.py:2335–2353`

#### Step 3: `handle()` — Main Dispatch

The `handle()` function at `email_handler.py:1945` sanitizes addresses and processes headers. For each recipient in `rcpt_tos`, it determines whether the address is a reverse alias or a forward target.

Since `test@sl.local` is **not** a reverse alias, the code enters the **Forward phase** at lines 2201–2211:

```python
else:  # Forward case
    LOG.d("Forward phase %s(%s) -> %s", mail_from, copy_msg[headers.FROM], rcpt_to)
    for is_delivered, smtp_status in handle_forward(envelope, copy_msg, rcpt_to):
        res.append((is_delivered, smtp_status))
```

Source: `email_handler.py:2201–2211`

#### Step 4: `handle_forward()` — Alias Lookup and Auto-Creation

The `handle_forward()` function at `email_handler.py:536`:

1. Sets `alias_address = rcpt_to` (line 541) — this is `test@sl.local`
2. Queries `alias = Alias.get_by(email=alias_address)` (line 543) — returns `None` because no aliases have been created
3. Enters the `if not alias:` block (line 544)
4. Logs: `"alias %s not exist. Try to see if it can be created on the fly"` (lines 545–547)
5. Calls `alias = try_auto_create(alias_address)` (line 549)

Source: `email_handler.py:536–549`

#### Step 5: `try_auto_create()` — Auto-Creation Attempt

The `try_auto_create()` function at `app/alias_utils.py:202–224`:

1. **VERP checks** (lines 204–212): `test@sl.local` does not start with `BOUNCE_PREFIX_FOR_REPLY_PHASE` or `BOUNCE_PREFIX`, so these checks pass
2. **Email validation** (lines 214–218): `test@sl.local` is a valid email format
3. **Custom domain auto-creation** (line 220): Calls `try_auto_create_via_domain(address)`

Source: `app/alias_utils.py:202–220`

#### Step 5a: `try_auto_create_via_domain()` — Custom Domain Check

At `app/alias_utils.py:274`, this calls `check_if_alias_can_be_auto_created_for_custom_domain(address)` (line 276), which at `app/alias_utils.py:92–142`:

1. Extracts the domain part: `sl.local`
2. Queries `CustomDomain.get_by(domain="sl.local")` (line 101)
3. Returns `None` — there are no custom domains in the database

Source: `app/alias_utils.py:100–107`

Since there is no custom domain, `try_auto_create_via_domain` returns `None` (line 278).

Source: `app/alias_utils.py:274–278`

#### Step 5b: `try_auto_create_directory()` — Directory Alias Check

At `app/alias_utils.py:222`, `try_auto_create` falls back to `try_auto_create_directory(address)`.

This calls `check_if_alias_can_be_auto_created_for_a_directory(address)` at `app/alias_utils.py:231`, which at lines 145–199:

1. Calls `can_create_directory_for_address("test@sl.local")` (line 153)
   - This function (at `app/email_utils.py:545–554`) checks if the email ends with `@` + any domain in `config.ALIAS_DOMAINS`
   - Since `config.ALIAS_DOMAINS = ["sl.local"]` and `"test@sl.local".endswith("@sl.local")` is `True`, it returns `True`
2. Checks for a directory separator (`/`, `+`, or `#`) in the address (lines 157–166)
   - `test@sl.local` contains **none** of these separators
   - Logs: `"Cannot auto-create test@sl.local since it has no directory separator"` (line 165)
   - Returns `None` (line 166)

Source: `app/alias_utils.py:145–166`, `app/email_utils.py:545–554`

Therefore `try_auto_create_directory` returns `None` (line 234–235), and `try_auto_create()` returns `None` (line 224).

Source: `app/alias_utils.py:222–224, 227–235`

#### Step 6: Back in `handle_forward()` — The Rejection

Back at `email_handler.py:550`:

```python
if not alias:
    LOG.d("alias %s cannot be created on-the-fly, return 550", alias_address)
    if should_ignore_bounce(envelope.mail_from):
        return [(True, status.E207)]
    else:
        return [(False, status.E515)]
```

Source: `email_handler.py:550–555`

- `alias` is still `None` (line 550)
- Logs: `"alias test@sl.local cannot be created on-the-fly, return 550"` (line 551)
- `should_ignore_bounce("sender@example.com")` returns `False` for a normal sender address (line 552)
- Returns `[(False, status.E515)]` (line 555)

#### Step 7: Status Resolution

The `handle()` function at `email_handler.py:2227–2233` processes the result list. Since there are no successful deliveries, it returns the first failure status:

```python
# Failed delivery for all, return the first failure
return res[0][1]
```

Source: `email_handler.py:2232–2233`

This returns the string `"550 SL E515 Email not exist"`.

Back in `_handle()` at lines 2357–2365, the SPF check evaluates whether to replace the 5XX status. For a test email sent via `swaks` without SPF headers, the `SpamdResult` is either absent or lacks SPF data, so the status remains unchanged.

Source: `email_handler.py:2357–2365`

### SMTP Status Code Returned

The exact SMTP status string returned to the sender is:

```text
550 SL E515 Email not exist
```

This is defined at `app/email/status.py:51`:

```python
E515 = "550 SL E515 Email not exist"
```

Source: `app/email/status.py:51`

This is a **permanent rejection** (5xx class), meaning the sender's MTA will **not** retry delivery.

### Log Output Explaining the Rejection

The complete log output sequence for this email lifecycle:

```text
<timestamp> - SL - DEBUG - <pid> - "email_handler.py:2342" - _handle() - <message_id> - ====>=====>====>====>====>====>====>====>
<timestamp> - SL - INFO - <pid> - "email_handler.py:2343" - _handle() - <message_id> - New message, mail from sender@example.com, rctp tos ['test@sl.local']
<timestamp> - SL - DEBUG - <pid> - "email_handler.py:2202" - handle() - <message_id> - Forward phase sender@example.com(...) -> test@sl.local
<timestamp> - SL - DEBUG - <pid> - "email_handler.py:545" - handle_forward() - <message_id> - alias test@sl.local not exist. Try to see if it can be created on the fly
<timestamp> - SL - DEBUG - <pid> - "email_handler.py:551" - handle_forward() - <message_id> - alias test@sl.local cannot be created on-the-fly, return 550
<timestamp> - SL - INFO - <pid> - "email_handler.py:2367" - _handle() - <message_id> - Finish mail_from sender@example.com, rcpt_tos ['test@sl.local'], takes X.XX seconds with return code '550 SL E515 Email not exist'<<===
```

Each log line follows the format defined at `app/log.py:12–14`:

```text
%(asctime)s - %(name)s - %(levelname)s - %(process)d - "%(pathname)s:%(lineno)d" - %(funcName)s() - %(message_id)s - %(message)s
```

Source: `app/log.py:12–14`

The `message_id` field is populated by `set_message_id()` at `email_handler.py:2340` at the start of each email lifecycle, and is injected into every log record by the `EmailHandlerFilter` class at `app/log.py:28–37`.

### Rationale and Impact of Missing `init_app.py`

For the specific test case of `test@sl.local`, the immediate cause of the E515 rejection is straightforward: there is no existing `Alias` record with email `test@sl.local`, and auto-creation fails because:

1. `sl.local` is not a `CustomDomain` — so catch-all/rule-based auto-creation via `try_auto_create_via_domain()` is not possible (Source: `app/alias_utils.py:100–107`)
2. `test@sl.local` has no directory separator (`/`, `+`, or `#`) — so directory-based auto-creation via `try_auto_create_directory()` is not possible (Source: `app/alias_utils.py:153–166`)

**The broader impact of not running `init_app.py`** is that the `public_domain` table remains empty. While the config variable `ALIAS_DOMAINS` still contains `["sl.local"]` (loaded from the `.env` file at runtime), the database has no corresponding `SLDomain` records. This affects:

- **`is_valid_alias_address_domain()`** at `app/email_utils.py:557–566` returns `False` for `sl.local` because it checks `SLDomain.get_by(domain=domain)` against the database (line 560), not the config variable. This function is used in various validation paths throughout the codebase.
- **Web UI alias creation**: Users cannot create aliases on `sl.local` through the web interface because the domain does not appear as an available option from the `SLDomain` table.
- **Alias validation**: Any code path that calls `SLDomain.get_by(domain="sl.local")` will get `None`, potentially causing unexpected behavior in other parts of the system.

Source: `app/email_utils.py:557–566`, `init_app.py:39–56`, `app/models.py:3116–3119`

### Flowchart: Email Handler Forward-Phase Decision Tree

```mermaid
flowchart TD
    A["handle_DATA()<br/>email_handler.py:2289"] --> B["_handle()<br/>email_handler.py:2335"]
    B --> C["handle()<br/>email_handler.py:1945"]
    C --> D{"is_reverse_alias(rcpt_to)?<br/>email_handler.py:2195"}
    D -->|No| E["Forward phase<br/>email_handler.py:2201"]
    D -->|Yes| R["handle_reply()"]
    E --> F["handle_forward()<br/>email_handler.py:536"]
    F --> G{"Alias.get_by(email='test@sl.local')<br/>email_handler.py:543"}
    G -->|"None (no alias exists)"| H["try_auto_create('test@sl.local')<br/>app/alias_utils.py:202"]
    G -->|"Found"| FWD["Forward to mailbox"]
    H --> I{"try_auto_create_via_domain()<br/>app/alias_utils.py:220"}
    I --> J["CustomDomain.get_by(domain='sl.local')<br/>app/alias_utils.py:101"]
    J -->|"None (not a custom domain)"| K["return None"]
    K --> L{"try_auto_create_directory()<br/>app/alias_utils.py:222"}
    L --> M["can_create_directory_for_address()<br/>app/email_utils.py:545"]
    M -->|"True (sl.local in config.ALIAS_DOMAINS)"| N{"Has directory separator?<br/>/ or + or #<br/>app/alias_utils.py:157–166"}
    N -->|"No separator in 'test@sl.local'"| O["return None"]
    O --> P["try_auto_create returns None<br/>app/alias_utils.py:224"]
    P --> Q["handle_forward returns<br/>(False, E515)<br/>email_handler.py:555"]
    Q --> S["SMTP Response:<br/>550 SL E515 Email not exist"]

    style S fill:#f96,stroke:#333,stroke-width:2px
    style Q fill:#fcc,stroke:#333
```

---

## Source References

All source files referenced in this document, with their key line ranges:

| File | Key Lines | Content |
|------|-----------|---------|
| `server.py` | 127–136 | `create_light_app()` — minimal Flask app factory |
| | 139–217 | `create_app()` — full Flask app with blueprints, extensions, error handlers |
| | 220–230 | `@login_manager.user_loader` — user session loader callback |
| | 249–255 | `set_index_page()` — root route redirect to login |
| | 388–394 | `@app.errorhandler(Exception)` — generic 500 error handler |
| | 572–588 | `local_main()` — development server startup |
| | 598–599 | `if __name__ == "__main__"` — entry point |
| `email_handler.py` | 536–555 | `handle_forward()` — alias lookup, auto-create, E515 return |
| | 1945–2233 | `handle()` — main email dispatch, forward/reply routing |
| | 2195–2211 | Forward phase — `is_reverse_alias` check and `handle_forward` call |
| | 2227–2233 | Return status resolution — first success or first failure |
| | 2288–2332 | `MailHandler` class — aiosmtpd integration |
| | 2335–2378 | `_handle()` — message lifecycle, app context, SPF check |
| | 2381–2393 | `main()` — aiosmtpd Controller setup, keep-alive loop |
| | 2396–2404 | `if __name__ == "__main__"` — argparse, entry point |
| `init_app.py` | 39–56 | `add_sl_domains()` — seeds SLDomain records from ALIAS_DOMAINS |
| | 69–73 | `if __name__ == "__main__"` — entry point |
| `job_runner.py` | 307–326 | `get_jobs_to_run()` — job query with state/timing filters |
| | 329–347 | `if __name__ == "__main__"` — polling loop entry point |
| `app/db.py` | 9–14 | Engine creation, connection, scoped Session |
| `app/config.py` | 92 | `EMAIL_DOMAIN` — loaded from environment |
| | 153–161 | `OTHER_ALIAS_DOMAINS`, `ALIAS_DOMAINS` — domain lists |
| | 192 | `DB_URI` — database connection string |
| | 196 | `FLASK_SECRET` — Flask secret key |
| | 339 | `LOAD_PGP_EMAIL_HANDLER` — PGP loading flag |
| `app/models.py` | 52 | `Base = declarative_base()` |
| | 62–84 | `ModelMixin` — including `get_by()` at lines 82–84 |
| | 336–337 | `User` class — `__tablename__ = "users"` |
| | 3116–3119 | `SLDomain` class — `__tablename__ = "public_domain"` |
| `app/auth/views/login.py` | 16–18 | `LoginForm` class |
| | 21–82 | `login()` route handler — GET/POST logic |
| `app/email/status.py` | 51 | `E515 = "550 SL E515 Email not exist"` |
| `app/alias_utils.py` | 92–142 | `check_if_alias_can_be_auto_created_for_custom_domain()` |
| | 145–199 | `check_if_alias_can_be_auto_created_for_a_directory()` |
| | 202–224 | `try_auto_create()` — main auto-creation dispatcher |
| | 227–271 | `try_auto_create_directory()` |
| | 274–334 | `try_auto_create_via_domain()` |
| `app/email_utils.py` | 545–554 | `can_create_directory_for_address()` — checks config ALIAS_DOMAINS |
| | 557–566 | `is_valid_alias_address_domain()` — checks DB SLDomain table |
| `app/log.py` | 12–14 | Log format string |
| | 28–37 | `EmailHandlerFilter` — message_id injection |
| | 67 | `print(">>> init logging <<<")` — import-time output |
| | 70–71 | Werkzeug logger disabled |
| | 74–77 | Logger shortcut methods: `.d`, `.i`, `.w`, `.e` |
| | 79 | `LOG = _get_logger("SL")` |
| `app/extensions.py` | 7–8 | `login_manager` — Flask-Login LoginManager |
| | 23 | `limiter` — Flask-Limiter instance |
| `example.env` | 6 | `URL=http://localhost:7777` |
| | 19 | `NOT_SEND_EMAIL=true` |
| | 22 | `EMAIL_DOMAIN=sl.local` |
| | 40 | `SUPPORT_EMAIL=support@sl.local` |
| | 75 | `DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin` |
| | 77 | `FLASK_SECRET=secret` |
| `CONTRIBUTING.md` | 103–109 | Local run instructions |
| | 143–147 | Entry points list |
| | 209–219 | Email handler test with swaks |
| | 225–229 | Job runner documentation |
