# SimpleLogin Operational Verification & Behavioral Walkthrough

## Introduction

This document provides a comprehensive, evidence-based reference for understanding SimpleLogin's runtime behavior. It answers three interconnected questions by tracing actual code paths, log output, and system indicators observed directly in the repository source code:

1. **How to confirm the platform is actually working** — identifying the specific log messages, UI responses, HTTP endpoints, and system indicators that prove each component (Flask web app, PostgreSQL, Redis, Postfix, background workers) is initialized and healthy.
2. **A walkthrough of the typical new-user product experience** — tracing the complete journey from registration form submission through email activation, login, and the dashboard landing page, documenting the HTTP lifecycle, database mutations, and visible UI behavior at each step.
3. **What happens behind the scenes during that flow** — surfacing the background service indicators that prove job runners, event dispatchers, email handlers, cron schedulers, and monitoring exporters are active, polling, and communicating correctly during and after a new-user registration.

All claims below reference specific file paths and line numbers from the SimpleLogin repository. No source code modifications are recommended or included.

---

## Section 1: Application Readiness Verification

This section identifies every observable signal that confirms SimpleLogin's subsystems are initialized and ready to handle requests. Each indicator is grounded in a specific code location.

### 1.1 Console Output During Startup

The very first visible output during application startup comes from Python module-level code that executes at import time, before any Flask request processing begins.

#### Logger Initialization Banner

- **`app/log.py:67`** prints `">>> init logging <<<"` to stdout. This is the first thing visible on the console because `app/log.py` is imported at the module level by `server.py` (via `from app.log import LOG` at `server.py:83`). When this message appears, it confirms the logging subsystem is bootstrapped.

  **Why it matters:** If this line does not appear, the Python module import chain has failed — nothing else will work.

#### URL Configuration Confirmation

- **`app/config.py:80`** prints `">>> URL: <value>"` where `<value>` is the URL loaded from the `URL` environment variable. In local development with `example.env`, this reads:
  ```
  >>> URL: http://localhost:7777
  ```
  This is printed because `app/config.py:79` reads `URL = os.environ["URL"]` and `app/config.py:80` immediately calls `print(">>> URL:", URL)`.

  **Why it matters:** This confirms the most critical configuration variable — the server's public URL — is loaded correctly from the environment. If this URL is wrong, activation links, OAuth callbacks, and all absolute URLs in emails will be broken.

#### Config File Loading

- **`app/config.py:66-68`**: When the `CONFIG` environment variable is set, the line `print("load config file", config_file)` confirms that a dotenv file has been located and is being loaded via `load_dotenv()`. In test environments, this prints the path to `tests/test.env`.

  **Why it matters:** This confirms the application is reading environment variables from the expected file, rather than falling back to system-level environment variables or defaults.

#### Free Plan Email Limit Fallback

- **`app/config.py:120-124`**: When `MAX_NB_EMAIL_FREE_PLAN` is not set in the environment (which is typical in local development, since `example.env` does not define it), a `try/except` block catches the `KeyError` and prints:
  ```
  MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
  ```
  Then `MAX_NB_EMAIL_FREE_PLAN` is set to `5`.

  **Why it matters:** This confirms the application gracefully handles missing optional configuration with sensible defaults, and alerts the operator when defaults are being used.

#### Paddle Payment Param Fallback

- **`app/config.py:216-220`**: When Paddle payment environment variables are not set, a `try/except` block prints `"Paddle param not set"` and sets vendor/product IDs to `-1`. This is expected for self-hosted instances that do not use Paddle for payment processing.

### 1.2 Log Format and Logger Configuration

- **`app/log.py:12-14`** defines the log format string:
  ```
  %(asctime)s - %(name)s - %(levelname)s - %(process)d - "%(pathname)s:%(lineno)d" - %(funcName)s() - %(message_id)s - %(message)s
  ```
  This format includes the timestamp, logger name (`SL`), severity level, process ID, source file path with line number (as a clickable link in IDEs like PyCharm), function name, an email lifecycle `message_id` (for tracing a single email through the handler), and the log message itself.

- **`app/log.py:79`**: The application-wide logger is created as `LOG = _get_logger("SL")`, named `"SL"` (SimpleLogin). All application code uses `LOG.d()` (debug), `LOG.i()` (info), `LOG.w()` (warning), and `LOG.e()` (exception) — these are shortcut aliases added at `app/log.py:74-77`.

- **`app/log.py:70-71`**: The default Werkzeug HTTP request logger is explicitly disabled (`log.disabled = True`) to prevent duplicate request logging, since SimpleLogin implements its own request lifecycle logging (see Section 1.9).

- **`app/log.py:61-62`**: When `COLOR_LOG` is set in the environment (as suggested in `example.env:16`), `coloredlogs.install(level="DEBUG", logger=logger, fmt=_log_format)` is called to enable color-coded log output for local development convenience.

  **Why it matters:** Understanding the log format is essential for parsing and monitoring log output. The `message_id` field is particularly important — it allows tracing a single inbound email's entire lifecycle through the email handler, from receipt to delivery.

### 1.3 Flask App Initialization Sequence

The `create_app()` function at **`server.py:139`** is the Flask application factory. It orchestrates the entire application bootstrap in a deterministic sequence:

| Step | Code Location | Action | Readiness Signal |
|------|---------------|--------|------------------|
| 1 | `server.py:140` | `app = Flask(__name__)` | Flask instance created |
| 2 | `server.py:142` | `ProxyFix(app.wsgi_app, x_for=1, x_host=1)` | WSGI middleware wrapped for reverse proxy |
| 3 | `server.py:144` | `app.url_map.strict_slashes = False` | Trailing slash tolerance enabled |
| 4 | `server.py:151` | `app.secret_key = FLASK_SECRET` | Session encryption key set |
| 5 | `server.py:159` | `SESSION_COOKIE_NAME = "slapp"` set via config | Cookie name configured to avoid conflicts |
| 6 | `server.py:163-165` | If `MEM_STORE_URI` is set: configure rate limiter storage and call `initialize_redis_services()` | Redis backend activated (conditional) |
| 7 | `server.py:167` | `limiter.init_app(app)` | Flask-Limiter ready |
| 8 | `server.py:169` | `setup_error_page(app)` | Custom 400/401/403/404/405/500 error handlers registered |
| 9 | `server.py:171` | `init_extensions(app)` | Flask-Login `login_manager.init_app(app)` called |
| 10 | `server.py:172` | `register_blueprints(app)` | All 11 blueprints registered |
| 11 | `server.py:173` | `set_index_page(app)` | Root `/` route registered (redirects to dashboard or login) |
| 12 | `server.py:174` | `jinja2_filter(app)` | Custom Jinja2 template filters added |
| 13 | `server.py:179` | `init_admin(app)` | Flask-Admin panel initialized |
| 14 | `server.py:180` | `setup_paddle_callback(app)` | Payment webhook routes registered |
| 15 | `server.py:200` | `CORS(app, resources={r"/api/*": {"origins": "*"}})` | CORS enabled on `/api/*` routes |
| 16 | `server.py:204-207` | `@app.before_request make_session_permanent()` | Session lifetime set to 7 days |
| 17 | `server.py:209-211` | `@app.teardown_appcontext cleanup()` | `Session.remove()` called after each request |
| 18 | `server.py:213-215` | Health check endpoint registered | `/health` returns `"success", 200` |

**Why it matters:** If `create_app()` completes without exception, every component above is wired. The health check endpoint (step 18) provides a simple HTTP-level confirmation that the entire initialization chain succeeded.

### 1.4 Blueprint Registration

**`server.py:233-246`** registers all application blueprints, which define the URL routing structure:

| Blueprint | Source Module | URL Prefix | Purpose |
|-----------|---------------|------------|---------|
| `auth_bp` | `app/auth/base.py:3` | `/auth` | Registration, login, activation, MFA, logout |
| `monitor_bp` | `app/monitor/base.py` | `/` | Internal monitoring endpoints |
| `dashboard_bp` | `app/dashboard/base.py:3` | `/dashboard` | Alias management, settings, account |
| `developer_bp` | `app/developer/base.py` | `/developer` | OAuth app management for developers |
| `phone_bp` | `app/phone/base.py` | `/phone` | Phone number alias feature |
| `oauth_bp` | `app/oauth/base.py` | `/oauth` | OAuth2 provider (first registration) |
| `oauth_bp` | `app/oauth/base.py` | `/oauth2` | OAuth2 provider (second registration, for compatibility) |
| `onboarding_bp` | `app/onboarding/base.py` | `/onboarding` | New user onboarding flow |
| `discover_bp` | `app/discover/base.py` | `/discover` | Service discovery / recommendations |
| `internal_bp` | `app/internal/base.py` | `/internal` | Internal API endpoints |
| `api_bp` | `app/api/base.py` | `/api` | Public REST API |

**Why it matters:** Blueprint registration confirms that all URL routes are active. If any blueprint import fails (e.g., due to a missing dependency), `create_app()` will raise an `ImportError` and the application will not start.

### 1.5 Health Check Endpoint

- **`server.py:213-215`** defines:
  ```python
  @app.route("/health", methods=["GET"])
  def healthcheck():
      return "success", 200
  ```

- **Verification command:**
  ```bash
  curl http://localhost:7777/health
  ```
  Expected response: `success` with HTTP status code `200`.

**Why it matters:** This is the definitive, single-request test for application readiness. Load balancers, container orchestrators (Docker health checks, Kubernetes liveness probes), and monitoring systems can all poll this endpoint. A `200` response proves the Flask app is initialized, the WSGI server is accepting connections, and the application factory completed without error. Note that this endpoint does NOT verify database or Redis connectivity — it only confirms the web server is up.

### 1.6 Database Connectivity

Database connectivity is established at module import time, before any request handling begins:

- **`app/db.py:9-11`**: `engine = create_engine(config.DB_URI, connect_args={"application_name": config.DB_CONN_NAME})` — creates the SQLAlchemy engine using the `DB_URI` from configuration and tags connections with `application_name` (defaulting to `"webapp"` per `app/config.py:193`).
- **`app/db.py:12`**: `connection = engine.connect()` — a physical database connection is established immediately at import time. If PostgreSQL is unreachable, this line raises an `OperationalError` and the application fails to start.
- **`app/db.py:14`**: `Session = scoped_session(sessionmaker(bind=connection))` — creates a thread-local scoped session factory used throughout the application.

The default database URI from `example.env:75` is:
```
postgresql://myuser:mypassword@localhost:5432/simplelogin
```

**Why it matters:** Because `app/db.py` is imported at module level (via `from app.db import Session` in `server.py:76`), a database connection failure prevents the application from starting entirely. If the Flask app is serving requests, the database is connected.

### 1.7 Redis Readiness (Conditional)

Redis is used for session storage, rate limiting, and concurrent request locking, but it is **optional** — the application functions without it when `MEM_STORE_URI` is not set.

- **`server.py:163-165`**: If `MEM_STORE_URI` is set, `server.py` configures the Flask-Limiter storage backend and calls `initialize_redis_services(app, MEM_STORE_URI)` from `app/redis_services.py`.
- **`app/redis_services.py:9-25`**: `initialize_redis_services()` handles three Redis URL schemes:
  - `redis://` or `rediss://` — standard Redis connection, sets up `RedisSessionStore`, concurrent lock, and rate limit storage
  - `redis+sentinel://` — Redis Sentinel for HA deployments, with separate master/slave storage connections
  - Any other scheme — raises `RuntimeError`

**Why it matters:** When `MEM_STORE_URI` is not set (as in `example.env`), Flask uses its default cookie-based session storage and in-memory rate limiting. This is acceptable for local development but not for production. The absence of Redis errors in logs confirms either Redis is connected successfully or is not configured.

### 1.8 Flask-Login and Rate Limiting

- **`app/extensions.py:7-8`**: `login_manager = LoginManager()` with `login_manager.session_protection = "strong"` — Flask-Login is configured with "strong" session protection, which regenerates the session identifier on each request and invalidates sessions if the client IP or user agent changes.

- **`app/extensions.py:22-23`**: `limiter = Limiter(key_func=__key_func)` — Flask-Limiter is configured with a custom key function that rate-limits based on user identity when logged in, or IP address when anonymous.

- **`app/extensions.py:14-19`**: The `__key_func()` function returns `f"userid:{current_user.id}"` for authenticated users and `f"ip:{ip_addr}"` for anonymous visitors. This ensures authenticated users are rate-limited per account (not per IP), which is important for users behind shared NATs.

**Why it matters:** Flask-Login's "strong" session protection provides session hijacking mitigation. The rate limiter prevents brute-force attacks on login and registration endpoints. Both must be initialized for the application to handle authentication securely.

### 1.9 WSGI Entry Point

- **`wsgi.py:1-3`**:
  ```python
  from server import create_app
  app = create_app()
  ```
  This is the Gunicorn target module. Gunicorn imports `wsgi:app`, which triggers `create_app()`.

- **`Dockerfile:47`**: The production CMD is:
  ```
  CMD ["gunicorn","wsgi:app","-b","0.0.0.0:7777","-w","2","--timeout","15"]
  ```
  This starts Gunicorn with 2 worker processes, listening on all interfaces at port 7777, with a 15-second request timeout.

**Why it matters:** Gunicorn worker startup messages in the console (e.g., `[INFO] Booting worker with pid: ...`) confirm that the WSGI server has forked its workers and is ready to accept HTTP connections. The `--timeout 15` setting means any request taking longer than 15 seconds will be killed — important for understanding timeout behavior during email operations.

### 1.10 Request Lifecycle Logging

Every HTTP request (except static assets) is logged with timing information:

- **`server.py:257-265`**: The `@app.before_request` handler captures `g.start_time = time.time()` for all non-static requests (excluding paths starting with `/static`, `/admin/static`, or `/_debug_toolbar`). It also extracts referral codes from `?slref=code` query parameters into the session.

- **`server.py:272-296`**: The `@app.after_request` handler logs every request (excluding `/static`, `/admin/static`, `/_debug_toolbar`, `/git`, `/favicon.ico`, and `/health`) with the format:
  ```
  {remote_addr} {method} {path} {args} {status_code}, takes {elapsed_seconds}
  ```
  using `LOG.d()`. It also records a New Relic custom event `"HttpResponseStatus"` with the response status code.

**Why it matters:** This request-level logging is the primary tool for observing application behavior in real time. When you see log lines like `127.0.0.1 POST /auth/register ImmutableMultiDict([]) 302, takes 0.15`, you know the registration endpoint was hit and responded with a redirect in 150ms.

### 1.11 Email Handler Readiness

The email handler is a separate process that receives inbound SMTP email:

- **`email_handler.py:2381-2386`**: The `main(port)` function creates an aiosmtpd `Controller(MailHandler(), hostname="0.0.0.0", port=port)`, calls `controller.start()`, then logs:
  ```
  Start mail controller 0.0.0.0 20381
  ```

- **`email_handler.py:2396-2404`**: When run as `__main__`, it defaults to port `20381` and logs `"Listen for port {port}"` at startup.

- **`email_handler.py:2388-2390`**: If `LOAD_PGP_EMAIL_HANDLER` is set, it calls `load_pgp_public_keys()` and logs `"LOAD PGP keys"` at WARNING level.

- The handler enters an infinite loop with `time.sleep(2)` to keep the controller alive.

**Why it matters:** The `"Start mail controller"` log message confirms the SMTP server is actively listening for inbound email. Without this process running, inbound email forwarding and reply handling will not function. The default port 20381 is the internal SMTP port that Postfix forwards to.

### 1.12 Job Runner Readiness

The job runner is a separate process that polls the database for background jobs:

- **`job_runner.py:329-347`**: The `if __name__ == "__main__"` block enters an infinite loop:
  1. Creates a Flask app context via `create_light_app().app_context()`
  2. Calls `get_jobs_to_run()` to find eligible jobs
  3. For each job: marks it as `taken`, increments `attempts`, calls `process_job(job)`, marks it as `done`
  4. Sleeps for 10 seconds between iterations

- **`job_runner.py:307-326`**: `get_jobs_to_run()` queries for `Job` records matching:
  - `Job.state == JobState.ready` (new jobs), OR
  - `Job.state == JobState.taken AND taken_at < now - 30 minutes AND attempts < JOB_MAX_ATTEMPTS` (stale/retried jobs)
  - AND `run_at IS NULL OR run_at <= now + 10 minutes` (eligible to run now or within the next polling window)

**Why it matters:** The 10-second sleep interval acts as the job runner's heartbeat. If you see no output from the job runner for 10+ seconds, it means the database poll returned zero eligible jobs — which is the normal idle state. Active job processing will produce log lines for each job dispatched.

### 1.13 Monitoring Service Readiness

- **`monitoring.py:157-171`**: The `if __name__ == "__main__"` block initializes a `MetricExporter` with a New Relic license key, then enters an infinite 60-second loop calling:
  - `log_postfix_metrics()` (`monitoring.py:39-55`) — counts files in `/var/spool/postfix/{incoming,active,deferred}` and counts smtp/smtpd/bounce/cleanup processes
  - `log_nb_db_connection()` (`monitoring.py:88-94`) — queries `pg_stat_activity` for total connection count
  - `log_pending_to_process_events()` (`monitoring.py:112-119`) — counts `sync_event` rows where `taken_time IS NULL`
  - `log_events_pending_dead_letter()` (`monitoring.py:122-139`) — counts stale events older than 10 minutes
  - `log_failed_events()` (`monitoring.py:142-154`) — counts events with `retry_count >= 10`
  - `log_nb_db_connection_by_app_name()` (`monitoring.py:98-108`) — groups DB connections by `application_name`

**Why it matters:** The monitoring service provides the operational metrics that power alerting and dashboards. Log lines like `"postfix queue sizes X Y Z"` and `"number of db connections N"` confirm the monitoring loop is running and that Postfix and PostgreSQL are accessible from the monitoring process.

---

## Section 2: End-to-End New-User Flow Walkthrough

This section traces the complete product experience for a first-time user, from opening the registration page to landing on the alias management dashboard. Each step documents the HTTP request/response cycle, database mutations, and visible UI behavior.

### Step 1: Registration Page (`GET /auth/register`)

**Route definition:** `app/auth/views/register.py:31` registers the route on `auth_bp` at `/register` accepting both GET and POST methods.

**What happens on GET:**
1. If the user is already authenticated (`current_user.is_authenticated` at `register.py:33`), they are redirected to `dashboard.index` with a flash message `"You are already logged in"`.
2. If `config.DISABLE_REGISTRATION` is set (`register.py:38`), registration is blocked with flash `"Registration is closed"` and the user is redirected to the login page.
3. Otherwise, the template `auth/register.html` is rendered with a `RegisterForm` (`register.py:106-114`).

**Form definition** at `app/auth/views/register.py:23-28`:
```python
class RegisterForm(FlaskForm):
    email = StringField("Email", validators=[validators.DataRequired()])
    password = StringField(
        "Password",
        validators=[validators.DataRequired(), validators.Length(min=8, max=100)],
    )
```

The form requires:
- **Email**: Non-empty string (DataRequired)
- **Password**: Non-empty string, minimum 8 characters, maximum 100 characters

The template also conditionally displays social login options (Proton, OIDC) based on whether `CONNECT_WITH_PROTON` and `OIDC_CLIENT_ID` are configured.

### Step 2: Registration Submission (`POST /auth/register`)

When the form is submitted, the following sequence executes:

1. **Form validation** (`register.py:45`): `form.validate_on_submit()` checks CSRF token validity and field validators (email non-empty, password 8-100 chars).

2. **hCaptcha verification** (`register.py:47-71`): Only when `HCAPTCHA_SECRET` is configured. The server-side verification posts to `https://hcaptcha.com/siteverify`. On failure, emits `RegisterEvent(catpcha_failed)` and re-renders the form.

3. **Email canonicalization** (`register.py:73`): `canonicalize_email(form.email.data)` normalizes the email (lowercasing, removing dots from Gmail addresses, etc.).

4. **Email domain check** (`register.py:74`): `email_can_be_used_as_mailbox(email)` verifies the email domain is not one of SimpleLogin's own alias domains (which would create a circular reference).

5. **Duplicate check** (`register.py:79-81`): `personal_email_already_used(email)` checks whether an existing user is already registered with this email address, also checking the sanitized form.

6. **User creation** (`register.py:86-91`):
   ```python
   user = User.create(
       email=email,
       name=form.email.data,
       password=form.password.data,
       referral=get_referral(),
   )
   ```

7. **Database commit** (`register.py:92`): `Session.commit()` persists the new User and all related records created by `User.create()` (see Step 3).

8. **Activation email dispatch** (`register.py:95`): `send_activation_email(user, next_url)` generates an activation code and sends the activation email.

9. **Telemetry** (`register.py:96`): `RegisterEvent(RegisterEvent.ActionType.success).send()` records a New Relic custom event.

10. **Daily metric increment** (`register.py:97`): `DailyMetric.get_or_create_today_metric().nb_new_web_non_proton_user += 1` increments the daily registration counter.

11. **Response** (`register.py:104`): The user sees `render_template("auth/register_waiting_activation.html")` — a page instructing them to check their email for the activation link.

### Step 3: What `User.create()` Does Internally

The `User.create()` classmethod at **`app/models.py:601-668`** performs a rich set of operations beyond simple row insertion:

| Line(s) | Action | Database Effect |
|----------|--------|-----------------|
| 603 | `email = sanitize_email(email)` | Sanitizes input email |
| 604 | `super(User, cls).create(email=email, name=name[:100], **kwargs)` | Inserts `users` row |
| 606-607 | `user.set_password(password)` (if password provided) | Bcrypt-hashes password via `PasswordOracle` mixin (`app/pw_models.py:11-14`) |
| 609 | `Session.flush()` | Assigns `user.id` from database sequence |
| 611 | `Mailbox.create(user_id=user.id, email=user.email, verified=True)` | Creates default mailbox (pre-verified) |
| 613 | `user.default_mailbox_id = mb.id` | Links user to default mailbox |
| 616-617 | `user.alternative_id = str(uuid.uuid4())` | Generates UUID for Flask-Login session identification |
| 634-640 | `Alias.create_new(user, prefix="simplelogin-newsletter", mailbox_id=mb.id, ...)` | Creates the user's first email alias |
| 643 | `user.newsletter_alias_id = alias.id` | Links newsletter alias to user |
| 646-648 | If `config.DISABLE_ONBOARDING` is set, return early | Skips onboarding job scheduling |
| 651-655 | `Job.create(name=JOB_ONBOARDING_1, run_at=arrow.now().shift(days=1))` | Schedules onboarding email #1 for +1 day |
| 656-660 | `Job.create(name=JOB_ONBOARDING_2, run_at=arrow.now().shift(days=2))` | Schedules onboarding email #2 for +2 days |
| 661-665 | `Job.create(name=JOB_ONBOARDING_4, run_at=arrow.now().shift(days=3))` | Schedules onboarding email #3 for +3 days |
| 666 | `Session.flush()` | Persists all pending changes |

**Key observations:**
- The first alias is always `simplelogin-newsletter@{FIRST_ALIAS_DOMAIN}` (where `FIRST_ALIAS_DOMAIN` defaults to `EMAIL_DOMAIN` per `app/config.py:168`).
- The alias note reads: `"This is your first alias. It's used to receive SimpleLogin communications like new features announcements, newsletters."`
- In `example.env:150`, `DISABLE_ONBOARDING=true` is set, meaning onboarding jobs are **not** scheduled in the default self-hosted configuration. In the SaaS version (where this is not set), three onboarding tip emails are scheduled at 1-day, 2-day, and 3-day intervals.
- Password hashing uses bcrypt via the `PasswordOracle` mixin at `app/pw_models.py:8-21`, which normalizes the password to NFKC Unicode form before hashing.

### Step 4: Activation Email Dispatch

The `send_activation_email()` function at **`app/auth/views/register.py:117-129`** handles activation code generation:

1. **`register.py:119`**: Deletes all previous `ActivationCode` records for this user to ensure only one valid code exists:
   ```python
   Session.query(ActivationCode).filter(ActivationCode.user_id == user.id).delete()
   ```

2. **`register.py:120`**: Creates a new activation code with a 30-character random string:
   ```python
   activation = ActivationCode.create(user_id=user.id, code=random_string(30))
   ```

3. **`register.py:124`**: Constructs the activation link:
   ```
   {URL}/auth/activate?code={activation.code}
   ```
   If a `next_url` was provided (e.g., the user was redirected from an OAuth flow), it is appended to the activation link.

4. **`register.py:129`**: Calls `email_utils.send_activation_email(user, activation_link)`.

**`app/email_utils.py:125-141`** sends the activation email with:
- **Subject:** `"Just one more step to join SimpleLogin"`
- **Text template:** `transactional/activation.txt`
- **HTML template:** `transactional/activation.html`

**`app/models.py:1202-1215`** defines the `ActivationCode` model:
- `code`: 128-character string column, unique
- `expired`: Defaults to `arrow.now().shift(hours=1)` (via `_expiration_1h()` at `app/models.py:1186-1187`) — 1-hour TTL
- `is_expired()`: Returns `True` if `self.expired < arrow.now()`

### Step 5: Local Development Email Behavior

In local development, emails are **not** actually sent. This behavior is controlled by:

- **`example.env:19`**: `NOT_SEND_EMAIL=true`
- **`app/config.py:91`**: `NOT_SEND_EMAIL = "NOT_SEND_EMAIL" in os.environ` — this is a boolean flag derived from the mere presence of the environment variable.

- **`app/mail_sender.py:126-137`**: The `MailSender.send()` method checks `config.NOT_SEND_EMAIL` at line 130. When True, it logs the email details at DEBUG level:
  ```
  send email with subject '{subject}', from '{from}' to '{to}'
  ```
  and returns `True` without establishing any SMTP connection.

**Practical implication for local development:** Since the activation email is logged but not sent, the activation code must be retrieved directly from the database to complete the activation flow:
```sql
SELECT code FROM activation_code WHERE user_id = <user_id>;
```
Then navigate to:
```
http://localhost:7777/auth/activate?code=<the_code>
```

### Step 6: Account Activation (`GET /auth/activate?code=...`)

The activation endpoint at **`app/auth/views/activate.py:13-69`** processes the activation code:

1. **`activate.py:18-22`**: If the user is already authenticated, returns a 400 error with `"You are already logged in"`.

2. **`activate.py:24`**: Extracts the code from the query string: `code = request.args.get("code")`.

3. **`activate.py:26`**: Looks up the code: `ActivationCode.get_by(code=code)`.

4. **`activate.py:28-36`**: If the code is not found, triggers the rate limiter (`g.deduct_limit = True`) and returns a 400 error with `"Activation code cannot be found"`.

5. **`activate.py:38-46`**: If the code is expired (older than 1 hour), returns a 400 error with `"Activation code was expired"` and shows a resend activation option (`show_resend_activation=True`).

6. **`activate.py:48-49`**: Retrieves the user and sets `user.activated = True`.

7. **`activate.py:50`**: Logs the user in immediately via `login_user(user)` — Flask-Login creates the session.

8. **`activate.py:53`**: Deletes the consumed activation code: `ActivationCode.delete(activation_code.id)`.

9. **`activate.py:54`**: Commits all changes: `Session.commit()`.

10. **`activate.py:56`**: Flashes the success message: `flash("Your account has been activated", "success")`.

11. **`activate.py:58`**: Sends the welcome email: `email_utils.send_welcome_email(user)`.
    - **`app/email_utils.py:97-112`**: Welcome email subject is `"Welcome to SimpleLogin"`, using templates `com/welcome.txt` and `com/welcome.html`.

12. **`activate.py:61-67`**: Redirects the user:
    - If a `next` query parameter was provided, redirects to that URL (after sanitization).
    - Otherwise, redirects to `url_for("dashboard.index")` — the dashboard landing page.

**Net database state after activation:**
- `users.activated` = `True`
- `activation_code` row deleted
- Flask session cookie set

### Step 7: Login (Subsequent Visits)

After initial activation, returning users log in via the login page.

**Route definition:** `app/auth/views/login.py:21-82` registers `/login` on `auth_bp`.

**Login flow (`POST /auth/login`):**

1. **`login.py:36`**: `LoginForm(request.form)` creates the form with email and password fields.

2. **`login.py:40`**: `form.validate_on_submit()` checks CSRF and field validation.

3. **`login.py:41-42`**: Email is sanitized and canonicalized:
   ```python
   email = sanitize_email(form.email.data)
   canonical_email = canonicalize_email(email)
   ```

4. **`login.py:43`**: User lookup with fallback:
   ```python
   user = User.get_by(email=email) or User.get_by(email=canonical_email)
   ```

5. **`login.py:45-50`**: **Credential check** — if user is not found OR `user.check_password(form.password.data)` fails (bcrypt verification via `app/pw_models.py:16-21`):
   - Rate limiter triggered (`g.deduct_limit = True`)
   - Flash: `"Email or password incorrect"`
   - Telemetry: `LoginEvent(LoginEvent.ActionType.failed).send()`

6. **`login.py:51-56`**: **Disabled account** — if `user.disabled` is True:
   - Flash: `"Your account is disabled. Please contact SimpleLogin team to re-enable your account."`
   - Telemetry: `LoginEvent(LoginEvent.ActionType.disabled_login).send()`

7. **`login.py:57-62`**: **Scheduled deletion** — if `user.delete_on is not None`:
   - Flash with deletion date
   - Telemetry: `LoginEvent(LoginEvent.ActionType.scheduled_to_be_deleted).send()`

8. **`login.py:63-69`**: **Not activated** — if `user.activated` is False:
   - Shows resend activation option
   - Flash: `"Please check your inbox for the activation email. You can also have this email re-sent"`
   - Telemetry: `LoginEvent(LoginEvent.ActionType.not_activated).send()`

9. **`login.py:70-72`**: **Success** — all checks passed:
   - `LoginEvent(LoginEvent.ActionType.success).send()`
   - `return after_login(user, next_url)`

### Step 8: The `after_login()` MFA Decision Tree

The `after_login()` function at **`app/auth/views/login_utils.py:12-45`** determines the post-login redirect based on the user's multi-factor authentication configuration:

```
after_login(user, next_url, login_from_proton=False)
│
├── Not from Proton login (line 19)?
│   ├── user.fido_enabled()? (line 20)
│   │   └── YES: session[MFA_USER_ID] = user.id → redirect to /auth/fido (lines 23-27)
│   │
│   ├── user.enable_otp? (line 28)
│   │   └── YES: session[MFA_USER_ID] = user.id → redirect to /auth/mfa (lines 29-33)
│   │
│   └── Neither FIDO nor OTP enabled (falls through)
│
└── No MFA required:
    ├── login_user(user) — Flask-Login session created (line 36)
    ├── session["sudo_time"] = int(time()) — marks elevated privilege timestamp (line 37)
    └── Redirect to next_url if provided, otherwise dashboard.index (lines 40-45)
```

**Key details:**
- **FIDO/WebAuthn** takes priority over TOTP if both are enabled
- **`session[MFA_USER_ID]`** stores the user ID temporarily until MFA is completed — the user is NOT yet logged in via Flask-Login at this point
- **`session["sudo_time"]`** (`login_utils.py:37`) records the timestamp of login, used for sudo-mode protection on sensitive operations (similar to GitHub's sudo mode)
- The final redirect goes to `url_for("dashboard.index")` at `login_utils.py:45`

### Step 9: Dashboard Landing Page

After successful authentication (with or without MFA), the user lands on the dashboard.

**Route definition:** `app/dashboard/views/index.py:55-66` registers `/` on `dashboard_bp` (which has URL prefix `/dashboard`), resulting in the full path `/dashboard/`. The route is decorated with `@login_required`.

**Stats computation** at `app/dashboard/views/index.py:32-52`:

The `get_stats(user)` function computes four key metrics:

| Metric | Query | Purpose |
|--------|-------|---------|
| `nb_alias` | `Alias.filter_by(user_id=user.id).count()` | Total aliases owned by user |
| `nb_forward` | `EmailLog` where `is_reply=False, blocked=False, bounced=False` | Emails forwarded to real mailbox |
| `nb_reply` | `EmailLog` where `is_reply=True, blocked=False, bounced=False` | Replies sent from alias |
| `nb_block` | `EmailLog` where `is_reply=False, blocked=True, bounced=False` | Emails blocked by alias |

**Stats dataclass** at `app/dashboard/views/index.py:24-29`:
```python
@dataclass
class Stats:
    nb_alias: int
    nb_forward: int
    nb_reply: int
    nb_block: int
```

**For a brand-new user**, the dashboard will show:
- `nb_alias = 1` (the `simplelogin-newsletter` alias created during `User.create()`)
- `nb_forward = 0` (no emails have been forwarded yet)
- `nb_reply = 0` (no replies sent)
- `nb_block = 0` (no emails blocked)

The dashboard also supports alias creation (both custom and random), alias deletion, search, filtering, sorting, and pagination — all handled within the same `index()` view function (`index.py:67-170`).

---

## Section 3: Background Service Runtime Observability

This section documents the behind-the-scenes services that support the registration flow and ongoing platform operations. Each service runs as a separate process and communicates through shared database state, SMTP, or PostgreSQL NOTIFY channels.

### 3.1 Job Runner (`job_runner.py`)

The job runner is the primary background task processor, responsible for executing deferred work like onboarding emails, account deletions, and batch imports.

#### Main Loop

**`job_runner.py:329-347`**: The entry point runs an infinite loop:
```
while True:
    with create_light_app().app_context():
        for job in get_jobs_to_run():
            mark job as taken
            increment attempts
            process_job(job)
            mark job as done
    time.sleep(10)
```

Each iteration:
1. Creates a lightweight Flask app context (via `server.py:127-136` `create_light_app()`) for database access
2. Queries for eligible jobs
3. Processes each job within a try/except, marking state transitions
4. Sleeps 10 seconds before the next poll

#### Job Eligibility Query

**`job_runner.py:307-326`**: `get_jobs_to_run()` finds jobs matching:
- **New jobs:** `Job.state == JobState.ready`
- **Stale retries:** `Job.state == JobState.taken AND taken_at < now - 30min AND attempts < JOB_MAX_ATTEMPTS` (where `JOB_TAKEN_RETRY_WAIT_MINS` defaults to 30 and `JOB_MAX_ATTEMPTS` defaults to 5)
- **AND** the job is eligible to run now: `run_at IS NULL OR run_at <= now + 10min`

#### Onboarding Jobs

When `DISABLE_ONBOARDING` is not set, three onboarding jobs are created during registration (see Step 3 above):

| Job Name | Handler | Timing | Purpose |
|----------|---------|--------|---------|
| `onboarding-1` | `onboarding_send_from_alias(user)` at `job_runner.py:27-45` | +1 day | Tip email: "Send emails from your alias" |
| `onboarding-2` | `onboarding_mailbox(user)` at `job_runner.py:90-104` | +2 days | Tip email: "Multiple mailboxes" |
| `onboarding-4` | `onboarding_pgp(user)` at `job_runner.py:48-62` (with Proton mailbox check at `process_job()` lines 207-221) | +3 days | Tip email: "Secure your emails with PGP" |

Each handler retrieves the user's communication email via `user.get_communication_email()`, renders the email template, and sends it via `send_email()`.

**Observable indicators of job runner activity:**
- Log line `"Take job <job_repr>"` at `job_runner.py:334` when a job is picked up
- Job state transitions: `ready → taken → done` (viewable in the `job` database table)
- 10-second silence between polls = normal idle state

### 3.2 Event System

SimpleLogin uses two separate event systems: **New Relic custom events** for telemetry and a **protobuf-based event dispatcher** for synchronization with partner services (Proton).

#### Authentication Telemetry Events

**`app/events/auth_event.py:6-47`** defines two event classes:

**`LoginEvent`** (lines 6-25):
- Action types: `success`, `failed`, `disabled_login`, `not_activated`, `scheduled_to_be_deleted`
- Source types: `web`, `api`
- `send()` method calls `newrelic.agent.record_custom_event("LoginEvent", {"action": ..., "source": ...})`

**`RegisterEvent`** (lines 28-47):
- Action types: `success`, `failed`, `catpcha_failed`, `email_in_use`, `invalid_email`
- Source types: `web`, `api`
- `send()` method calls `newrelic.agent.record_custom_event("RegisterEvent", {"action": ..., "source": ...})`

**Why it matters:** These events provide real-time visibility into authentication success/failure rates. New Relic dashboards can alert on spikes in failed logins or registration anomalies.

#### Protobuf Event Dispatcher

**`app/events/event_dispatcher.py:14`**: `NOTIFICATION_CHANNEL = "simplelogin_sync_events"` — the PostgreSQL LISTEN/NOTIFY channel name.

**`app/events/event_dispatcher.py:23-26`** (`PostgresDispatcher.send()`):
1. Creates a `SyncEvent` row in the database with the serialized protobuf event bytes
2. Executes `NOTIFY simplelogin_sync_events, '{instance.id}'` to alert listeners

**`app/events/event_dispatcher.py:47-84`** (`EventDispatcher.send_event()`):
1. Checks `EVENT_WEBHOOK_DISABLE` — if set, events are suppressed entirely
2. Checks `EVENT_WEBHOOK` — if not configured and `skip_if_webhook_missing` is True, events are skipped
3. Retrieves the `PartnerUser` association for the user
4. Serializes a protobuf `Event` with `user_id`, `external_user_id`, `partner_id`, and `content`
5. Dispatches via the configured `Dispatcher` (defaults to `PostgresDispatcher`)
6. Records a New Relic `"EventStoredToDb"` custom event

**Why it matters:** This event system enables real-time synchronization between SimpleLogin and Proton. Events like `UserDeleted`, `AliasCreated`, etc. are dispatched through this pipeline.

### 3.3 Event Listener (`event_listener.py`)

The event listener consumes events produced by the dispatcher and forwards them to an external service.

**`event_listener.py:29-47`**: The `main(mode, dry_run, max_retries)` function configures:

| Parameter | Option | Implementation |
|-----------|--------|----------------|
| `mode=LISTENER` | `PostgresEventSource(EVENT_LISTENER_DB_URI)` | Listens on PostgreSQL LISTEN/NOTIFY channel `simplelogin_sync_events` |
| `mode=DEAD_LETTER` | `DeadLetterEventSource(max_retries)` | Processes failed events that exceeded retry limits |
| `dry_run=True` | `ConsoleEventSink()` | Prints events to console (for debugging) |
| `dry_run=False` | `HttpEventSink()` | Forwards events via HTTP to the configured webhook |

The `Runner(source=source, sink=sink).run()` orchestrates the event processing loop.

**Observable indicators:**
- Log lines `"Using PostgresEventSource"` or `"Using DeadLetterEventSource"` at startup
- Log lines `"Starting with ConsoleEventSink"` or `"Starting with HttpEventSink"`
- PostgreSQL `LISTEN` command visible in `pg_stat_activity` connection list

### 3.4 Email Handler (`email_handler.py`)

The email handler is the SMTP inbound processor that receives, classifies, and routes incoming email.

**Startup sequence** at `email_handler.py:2381-2404`:

1. **`email_handler.py:2383`**: Creates `Controller(MailHandler(), hostname="0.0.0.0", port=port)` using the aiosmtpd library
2. **`email_handler.py:2385`**: `controller.start()` — begins accepting SMTP connections in a background thread
3. **`email_handler.py:2386`**: Logs `"Start mail controller 0.0.0.0 20381"`
4. **`email_handler.py:2388-2390`**: Optionally loads PGP keys if `LOAD_PGP_EMAIL_HANDLER` is set
5. **`email_handler.py:2392-2393`**: Enters a keep-alive loop: `while True: time.sleep(2)`

**Per-email processing** at `email_handler.py:2370-2378`:
- Each inbound email receives a UUID `message_id` for lifecycle tracking (set via `app/log.py:22-25` `set_message_id()`)
- After processing, the handler logs: `mail_from`, `rcpt_tos`, elapsed time, and return status
- Records New Relic metrics: `Custom/email_handler_time` (processing duration) and `Custom/number_incoming_email` (counter)

**Observable indicators:**
- `"Start mail controller 0.0.0.0 20381"` at startup
- `"Listen for port 20381"` at startup
- Per-email log entries with `mail_from`, `rcpt_tos`, and timing
- New Relic custom metrics for email throughput and latency

### 3.5 Monitoring Service (`monitoring.py`)

The monitoring service exports operational metrics on a 60-second cycle.

**`monitoring.py:157-171`**: Main loop function calls:

| Function | Lines | What It Monitors | Metric Name |
|----------|-------|------------------|-------------|
| `log_postfix_metrics()` | 39-55 | Postfix queue sizes (incoming, active, deferred) and process counts (smtp, smtpd, bounce, cleanup) | `Custom/postfix_incoming_queue`, `Custom/postfix_active_queue`, `Custom/postfix_deferred_queue`, `Custom/process_{name}_count` |
| `log_nb_db_connection()` | 88-94 | Total PostgreSQL connections via `pg_stat_activity` | `Custom/nb_db_connections` |
| `log_nb_db_connection_by_app_name()` | 98-108 | PostgreSQL connections grouped by `application_name` (filters for `sl-*` prefixes) | `Custom/nb_db_app_connection/{app_name}` |
| `log_pending_to_process_events()` | 112-119 | Unprocessed `sync_event` rows (where `taken_time IS NULL`) | `Custom/sync_events_pending_to_process` |
| `log_events_pending_dead_letter()` | 122-139 | Stale events: taken but not completed within 10 minutes, or never taken but older than 10 minutes | `Custom/sync_events_pending_dead_letter` |
| `log_failed_events()` | 142-154 | Events with `retry_count >= 10` (permanently failed) | `Custom/sync_events_failed` |

**Observable indicators:**
- Log lines `"postfix queue sizes X Y Z"` every 60 seconds
- Log lines `"number of db connections N"` every 60 seconds
- Log lines `"number of events pending to process N"` every 60 seconds
- Silence for 60 seconds between cycles = normal behavior

### 3.6 Cron Jobs (`crontab.yml`)

SimpleLogin uses `yacron` to schedule recurring maintenance tasks. The schedule is defined in **`crontab.yml`** with 15 jobs:

| Job Name | Schedule | Concurrency | Description |
|----------|----------|-------------|-------------|
| `stats` | `0 0 * * *` (daily midnight) | Allow | Growth statistics computation |
| `delete_old_monitoring` | `15 1 * * *` (daily 01:15) | Allow | Remove old monitoring records |
| `check_custom_domain` | `15 2 * * *` (daily 02:15) | Allow | Verify custom domain DNS records |
| `check_hibp` | `15 3 * * *` (daily 03:15) | Forbid | Check Have I Been Pwned database |
| `notify_hibp` | `15 4 * * *` (daily 04:15) | Forbid | Send HIBP breach notifications |
| `delete_logs` | `15 5 * * *` (daily 05:15) | Allow | Remove old log entries |
| `delete_old_data` | `30 5 * * *` (daily 05:30) | Allow | Remove expired data |
| `poll_apple_subscription` | `15 6 * * *` (daily 06:15) | Allow | Check Apple subscription statuses |
| `notify_trial_end` | `15 8 * * *` (daily 08:15) | Allow | Send trial ending notifications |
| `notify_manual_subscription_end` | `15 9 * * *` (daily 09:15) | Allow | Send manual subscription ending notifications |
| `notify_premium_end` | `15 10 * * *` (daily 10:15) | Allow | Send premium ending notifications |
| `delete_scheduled_users` | `15 11 * * *` (daily 11:15) | Forbid | Process user deletion queue |
| `send_undelivered_mails` | `*/5 * * * *` (every 5 min) | Forbid | Retry failed email deliveries |
| `clear_alias_audit_log` | `0 * * * *` (hourly) | Forbid | Clean up old alias audit log entries |
| `clear_user_audit_log` | `0 * * * *` (hourly) | Forbid | Clean up old user audit log entries |

Each job is invoked as `python /code/cron.py -j <job_name>` with `shell: /bin/bash` and `captureStderr: true`.

**Concurrency policy:** Jobs marked `Forbid` will not start a new instance if a previous instance is still running. This prevents overlap for long-running or non-idempotent operations like HIBP checking and user deletion.

**Most relevant to the registration flow:** `send_undelivered_mails` (every 5 minutes) retries any emails that failed to deliver, including activation and welcome emails that may have failed on first attempt.

### 3.7 Email Sending in Local Development

Understanding the email behavior in local development is critical for testing the registration flow:

- **`example.env:19`**: `NOT_SEND_EMAIL=true` — this environment variable is set in the default local configuration.

- **`app/config.py:91`**: `NOT_SEND_EMAIL = "NOT_SEND_EMAIL" in os.environ` — a boolean derived from the environment variable's presence.

- **`app/mail_sender.py:130-137`**: When `config.NOT_SEND_EMAIL` is True, the `MailSender.send()` method:
  1. Logs the email at DEBUG level: `"send email with subject '{subject}', from '{from}' to '{to}'"`
  2. Returns `True` (indicating "success") without opening any SMTP connection

**This means in local development:**
- Activation emails are logged but NOT delivered to any mailbox
- Welcome emails are logged but NOT delivered
- Onboarding tip emails (if not disabled) are logged but NOT delivered
- All `send_email()` calls in `app/email_utils.py` pass through `MailSender.send()` and are subject to this short-circuit

**Workaround for testing:** To complete the activation flow locally, query the activation code directly from the database and construct the activation URL manually (see Step 5 above).

### 3.8 Domain Seeding (`init_app.py`)

The `init_app.py` script prepares the database with required seed data:

- **`init_app.py:39-56`** (`add_sl_domains()`): Iterates through `ALIAS_DOMAINS` and `PREMIUM_ALIAS_DOMAINS` configuration lists, creating `SLDomain` entries for each. Existing domains are skipped with a debug log. Premium domains are marked with `premium_only=True`.

- **`init_app.py:13-36`** (`load_pgp_public_keys()`): Loads PGP public keys for all mailboxes and contacts that have `pgp_public_key` set. Verifies fingerprint consistency and corrects mismatches.

- **`init_app.py:59-66`** (`add_proton_partner()`): Creates the Proton partner entry (name: `PROTON_PARTNER_NAME`, contact: `simplelogin@protonmail.com`) if it does not already exist.

- **`init_app.py:69-73`**: The script's `if __name__ == "__main__"` block runs within a `create_light_app().app_context()`, calling `load_pgp_public_keys()` and `add_sl_domains()`.

**Why it matters:** `init_app.py` must be run before the first user registration — otherwise, no alias domains will be available for creating the `simplelogin-newsletter` first alias, and `Alias.create_new()` will fail.

---

## Section 4: Ephemeral Test Data and Cleanup Notes

### Test Data Lifecycle

Any test users, aliases, or other data created during operational verification are **ephemeral** and should be cleaned up after testing:

- **User cleanup:** Deleting a `User` row from the `users` table cascades to all dependent records (mailboxes, aliases, activation codes, jobs, email logs, contacts, etc.) due to `ondelete="cascade"` foreign key constraints throughout `app/models.py`.

- **Database cleanup command:**
  ```sql
  DELETE FROM users WHERE email = 'test@example.com';
  ```
  This single statement removes the user and all associated data.

### Demo Data Seeding

- **`app/fake_data.py:40-50`** provides a `fake_data()` function that seeds the database with a demo user for development:
  - Email: `john@wick.com`
  - Name: `John Wick`
  - Password: `password`
  - `activated=True`, `is_admin=True`
  - Includes aliases, contacts, email logs, OAuth clients, and more

  This function is available as a Flask CLI command (registered via `register_custom_commands()` in `server.py`) and is intended purely for development convenience.

### Important Reminders

1. **This document does NOT create or remove any data** — it describes what the user should observe during manual testing.
2. **No source code modifications are recommended or required** — all behaviors described above are based on the existing codebase as-is.
3. **All activation codes expire after 1 hour** (`app/models.py:1186-1187`, `1212`) — if testing is paused, a new activation code must be generated.
4. **The `NOT_SEND_EMAIL=true` flag** in `example.env` means no actual emails leave the system during local testing — all email content is logged to the console at DEBUG level.
5. **The `DISABLE_ONBOARDING=true` flag** in `example.env` means onboarding jobs (onboarding-1, onboarding-2, onboarding-4) are NOT created during local `User.create()` — the job runner will have no onboarding work to process.

---

## Appendix: Quick Reference Commands

### Verify Application is Running
```bash
curl http://localhost:7777/health
# Expected: "success" with HTTP 200
```

### Check Startup Logs (look for these lines)
```
>>> init logging <<<
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
```

### Retrieve Activation Code (Local Dev)
```sql
SELECT code FROM activation_code WHERE user_id = (
    SELECT id FROM users WHERE email = 'your_test_email@example.com'
);
```

### Complete Activation (Local Dev)
```bash
curl -L "http://localhost:7777/auth/activate?code=<activation_code>"
```

### Check Job Queue
```sql
SELECT id, name, state, run_at, attempts FROM job ORDER BY created_at DESC LIMIT 10;
```

### Check User Registration
```sql
SELECT id, email, activated, created_at FROM users ORDER BY created_at DESC LIMIT 5;
```

### Check Alias Creation
```sql
SELECT id, email, user_id, created_at FROM alias WHERE user_id = <user_id>;
```

### Verify Email Handler Port
```bash
netstat -tlnp | grep 20381
# Or: ss -tlnp | grep 20381
```

### Check Postfix Queues
```bash
ls /var/spool/postfix/incoming/ | wc -l
ls /var/spool/postfix/active/ | wc -l
ls /var/spool/postfix/deferred/ | wc -l
```
