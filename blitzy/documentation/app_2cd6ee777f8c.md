# SimpleLogin Runtime Verification and Behavioral Analysis

## Introduction

This document provides **code-grounded answers** to runtime-verification and behavioral-analysis questions about the [SimpleLogin](https://github.com/simple-login/app) open-source email alias application. Every claim is derived from direct source-code analysis with specific **file paths, line numbers, and function references** — no assumptions or "typical behavior" statements are made.

SimpleLogin is composed of **three independent, long-running processes** that must be started separately:

| Process | Entry Point | Default Port | Purpose |
|---------|------------|--------------|---------|
| **Flask Web Server** | `server.py` → `create_app()` | 7777 | Dashboard UI, authentication, REST API, admin panel |
| **aiosmtpd Email Handler** | `email_handler.py` → `main()` | 20381 | Receives and processes inbound SMTP email for aliases |
| **Polling Job Runner** | `job_runner.py` → `__main__` | N/A | Executes background tasks (onboarding, account deletion, data export) |

An optional fourth component — the **cron scheduler** (`cron.py` driven by `crontab.yml` via yacron) — handles recurring maintenance tasks such as stats collection, HIBP checks, and log cleanup.

> **Notation Convention:** References follow the format `file.py:L###` to indicate the file and line number. Function names are formatted as `function_name()`.

---

## Section A — Startup Verification

This section documents the observable evidence that confirms each of the three core processes is alive, healthy, and capable of serving requests.

---

### A1. Flask Web Server Startup

#### A1.1 Application Factory: `create_app()`

The Flask application is constructed by `create_app()` at **`server.py:L139`**. This function performs the following initialization sequence:

1. **Creates the Flask instance** and applies `ProxyFix` middleware for deployment behind Nginx (`server.py:L140-142`).
2. **Configures SQLAlchemy** with the `DB_URI` connection string (`server.py:L146-147`).
3. **Sets the secret key** from the `FLASK_SECRET` environment variable (`server.py:L151`).
4. **Initializes the rate limiter** via `limiter.init_app(app)` (`server.py:L167`).
5. **Sets up error pages**, extensions, blueprints, Jinja2 filters, and admin interface (`server.py:L169-183`).
6. **Registers all blueprints** through `register_blueprints(app)` (`server.py:L172`, defined at `server.py:L233-246`):
   - `auth_bp` — Authentication routes (login, register, activate, MFA)
   - `dashboard_bp` — Dashboard and alias management UI
   - `api_bp` — REST API endpoints
   - `oauth_bp` — OAuth provider endpoints (registered twice for `/oauth` and `/oauth2`)
   - `monitor_bp` — Monitoring endpoints
   - `developer_bp`, `phone_bp`, `onboarding_bp`, `discover_bp`, `internal_bp`
7. **Enables CORS** on `/api/*` endpoints (`server.py:L200`).
8. **Configures session permanence** — cookies valid for 7 days (`server.py:L204-207`).
9. **Registers the `/health` endpoint** (`server.py:L213-215`).
10. **Registers the teardown handler** to clean up database sessions (`server.py:L209-211`).

If the `FLASK_PROFILER_PATH` environment variable is set, Flask-Profiler is also enabled (`server.py:L185-197`), which provides a profiling dashboard (excluding `/static`, `/git`, `/exception`, and `/health` paths).

**Rationale:** The `create_app()` factory pattern is the single initialization path for both development and production. Every blueprint, extension, and middleware is wired here — if this function completes without error, the Flask app is fully configured.

#### A1.2 The `/health` Endpoint

The health check is defined at **`server.py:L213-215`**:

```python
@app.route("/health", methods=["GET"])
def healthcheck():
    return "success", 200
```

**Observable behavior:**
- An HTTP GET to `http://localhost:7777/health` returns the plain-text body `"success"` with HTTP status code `200`.
- This endpoint is intentionally excluded from request logging (see A1.4 below), so it does not generate log noise during health polling.

**Verification command:**
```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost:7777/health
# Expected output: 200
```

**Rationale:** The health endpoint performs no database queries, authentication checks, or complex logic. A `200` response confirms that the Flask process is running, the WSGI layer is functional, and the routing system is operational. It does not, however, confirm database connectivity — for that, a successful dashboard login is required.

#### A1.3 Local Development Entry Point: `local_main()`

For local development, the entry point is `local_main()` at **`server.py:L565-588`**, invoked when the script is run directly (`server.py:L598-599`):

```python
if __name__ == "__main__":
    local_main()
```

`local_main()` performs:
1. Calls `create_app()` to build the Flask application.
2. Enables the Flask Debug Toolbar with profiler (`server.py:L577-582`).
3. Sets `app.debug = True` (`server.py:L581`).
4. Starts the development server: `app.run(debug=True, port=7777)` (`server.py:L588`).

**Observable log output:** When started, Flask's built-in Werkzeug server prints:
```
 * Serving Flask app "server" (lazy loading)
 * Running on http://127.0.0.1:7777/ (Press CTRL+C to quit)
```

**Note:** Werkzeug's default request logging is **disabled** at `app/log.py:L70-71`:
```python
log = logging.getLogger("werkzeug")
log.disabled = True
```
This means the standard `127.0.0.1 - - [date] "GET /path HTTP/1.1" 200` lines are suppressed. Instead, SimpleLogin uses its own `after_request` logging (see A1.4).

#### A1.4 Production Entry Point: Gunicorn via `wsgi.py`

For production deployment, the WSGI module at **`wsgi.py:L1-3`** provides the app object:

```python
from server import create_app
app = create_app()
```

The Dockerfile CMD at **`Dockerfile:L47`** starts Gunicorn:

```dockerfile
CMD ["gunicorn","wsgi:app","-b","0.0.0.0:7777","-w","2","--timeout","15"]
```

**Observable log output:** Gunicorn produces startup messages like:
```
[INFO] Starting gunicorn 20.0.4
[INFO] Listening at: http://0.0.0.0:7777
[INFO] Using worker: sync
[INFO] Booting worker with pid: <PID>
```

#### A1.5 Request Logging: `before_request` and `after_request` Hooks

**Before-request hook** (`server.py:L257-270`):
- Records `g.start_time = time.time()` for requests **not** starting with `/static`, `/admin/static`, or `/_debug_toolbar`.
- Also captures the referral code from the `slref` query parameter into the session.

**After-request hook** (`server.py:L272-296`):
- Logs every non-static HTTP request using `LOG.d()` with the format:
  ```
  <remote_addr> <method> <path> <args> <status_code>, takes <elapsed_seconds>
  ```
- **Excluded paths** (not logged): `/static`, `/admin/static`, `/_debug_toolbar`, `/git`, `/favicon.ico`, `/health`.
- Also records a `HttpResponseStatus` custom event to New Relic (`server.py:L293-295`).

**Example log output:**
```
2024-01-15 10:30:45 - SL - DEBUG - 12345 - "server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard/ ImmutableMultiDict([]) 200, takes 0.0523
```

**Rationale:** The request-level logging provides direct evidence that the web server is processing HTTP requests. Every authenticated page load, API call, and form submission will produce a log line — the presence of these lines confirms the server is alive and routing requests correctly.

---

### A2. Email Handler Startup

#### A2.1 The `main()` Function

The email handler's startup function is `main(port)` at **`email_handler.py:L2381-2393`**:

```python
def main(port: int):
    """Use aiosmtpd Controller"""
    controller = Controller(MailHandler(), hostname="0.0.0.0", port=port)
    controller.start()
    LOG.d("Start mail controller %s %s", controller.hostname, controller.port)

    if LOAD_PGP_EMAIL_HANDLER:
        LOG.w("LOAD PGP keys")
        load_pgp_public_keys()

    while True:
        time.sleep(2)
```

This function:
1. Creates an aiosmtpd `Controller` wrapping the `MailHandler()` class, bound to `0.0.0.0` on the configured port (`email_handler.py:L2383`).
2. Calls `controller.start()` to begin accepting SMTP connections in a background thread (`email_handler.py:L2385`).
3. Logs `"Start mail controller %s %s"` showing the hostname and port (`email_handler.py:L2386`).
4. If `LOAD_PGP_EMAIL_HANDLER` is configured, loads PGP public keys into the keyring (`email_handler.py:L2388-2390`).
5. Enters an infinite `while True: time.sleep(2)` loop to keep the process alive (`email_handler.py:L2392-2393`).

#### A2.2 The `__main__` Block

The entry point at **`email_handler.py:L2396-2404`**:

```python
if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument(
        "-p", "--port", help="SMTP port to listen for", type=int, default=20381
    )
    args = parser.parse_args()
    LOG.i("Listen for port %s", args.port)
    main(port=args.port)
```

- The default SMTP port is **20381** (`email_handler.py:L2399`).
- Before calling `main()`, logs `"Listen for port %s"` at INFO level (`email_handler.py:L2403`).

#### A2.3 Observable Startup Log Sequence

When `python email_handler.py` is executed, the following log sequence confirms readiness:

```
<timestamp> - SL - INFO - <pid> - "email_handler.py:2403" - <module>() -  - Listen for port 20381
<timestamp> - SL - DEBUG - <pid> - "email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

If PGP loading is enabled:
```
<timestamp> - SL - WARNING - <pid> - "email_handler.py:2389" - main() -  - LOAD PGP keys
```

#### A2.4 Docker Deployment

In Docker deployment, the email handler runs as a separate container (**`README.md:L470-481`**):

```bash
docker run -d \
    --name sl-email \
    -v $(pwd)/sl:/sl \
    -v $(pwd)/sl/upload:/code/static/upload \
    -v $(pwd)/simplelogin.env:/code/.env \
    -v $(pwd)/dkim.key:/dkim.key \
    -v $(pwd)/dkim.pub.key:/dkim.pub.key \
    -p 127.0.0.1:20381:20381 \
    --restart always \
    --network="sl-network" \
    simplelogin/app:3.4.0 python email_handler.py
```

Key observations:
- Port `20381` is mapped to the host (`-p 127.0.0.1:20381:20381`).
- Container has `--restart always` to ensure automatic recovery.
- Uses the same Docker image (`simplelogin/app:3.4.0`) as the web app, but with a different command.

#### A2.5 Verification Methods

1. **Check logs** for the startup sequence: `"Listen for port 20381"` followed by `"Start mail controller 0.0.0.0 20381"`.
2. **Test SMTP connectivity** using `swaks` (as documented in `CONTRIBUTING.md:L218`):
   ```bash
   swaks --to e1@sl.local --from hey@google.com --server 127.0.0.1:20381
   ```
3. **For Docker deployments**, check container logs:
   ```bash
   docker logs sl-email
   ```

**Rationale:** The two-line startup log sequence (`"Listen for port"` → `"Start mail controller"`) provides definitive evidence that the aiosmtpd Controller has initialized and is accepting SMTP connections. The `swaks` test additionally confirms end-to-end SMTP protocol handling.

---

### A3. Job Runner Startup

#### A3.1 The Main Polling Loop

The job runner's entry point is the `__main__` block at **`job_runner.py:L329-347`**:

```python
if __name__ == "__main__":
    while True:
        # wrap in an app context to benefit from app setup like database cleanup, sentry integration, etc
        with create_light_app().app_context():
            for job in get_jobs_to_run():
                LOG.d("Take job %s", job)

                # mark the job as taken, whether it will be executed successfully or not
                job.taken = True
                job.taken_at = arrow.now()
                job.state = JobState.taken.value
                job.attempts += 1
                Session.commit()
                process_job(job)

                job.state = JobState.done.value
                Session.commit()

            time.sleep(10)
```

The loop:
1. Creates a Flask application context via `create_light_app().app_context()` (`job_runner.py:L332`) for database access.
2. Calls `get_jobs_to_run()` to query the `Job` table (`job_runner.py:L333`).
3. For each job found: logs `"Take job %s"` (`job_runner.py:L334`), transitions the job to `taken` state, processes it, then transitions to `done` state.
4. Sleeps for **10 seconds** between iterations (`job_runner.py:L347`).

#### A3.2 Job Query Logic: `get_jobs_to_run()`

Defined at **`job_runner.py:L307-326`**, this function queries the `Job` table for records matching:

- **Ready jobs:** `state == JobState.ready.value` (i.e., `state == 0` per `app/models.py:L254`)
- **OR stale taken jobs:** `state == JobState.taken.value` AND `taken_at` older than 30 minutes AND `attempts < JOB_MAX_ATTEMPTS` (default 5)
- **AND** `run_at IS NULL` OR `run_at <= now + 10 minutes`

The 30-minute stale-job retry window (`config.JOB_TAKEN_RETRY_WAIT_MINS`) and maximum 5 attempts (`config.JOB_MAX_ATTEMPTS`) provide fault tolerance — if a job runner crashes mid-processing, another iteration will pick up the stale job.

#### A3.3 Observable Startup Behavior

**Important:** The job runner does **not** emit an explicit startup log message. Unlike the web server and email handler, there is no `"Job runner started"` line. Its presence is confirmed by:

1. The appearance of `"Take job <Job>"` log lines whenever jobs are available in the queue.
2. The **absence of error logs** during the continuous 10-second polling cycle.
3. The transition of `Job` records in the database from `state=0` (ready) → `state=1` (taken) → `state=2` (done).

**Example log output when a job is processed:**
```
<timestamp> - SL - DEBUG - <pid> - "job_runner.py:334" - <module>() -  - Take job <Job 42 onboarding-1 {'user_id': 5}>
```

#### A3.4 Docker Deployment

In Docker, the job runner runs as a separate container (**`README.md:L486-496`**):

```bash
docker run -d \
    --name sl-job-runner \
    -v $(pwd)/sl:/sl \
    -v $(pwd)/sl/upload:/code/static/upload \
    -v $(pwd)/simplelogin.env:/code/.env \
    -v $(pwd)/dkim.key:/dkim.key \
    -v $(pwd)/dkim.pub.key:/dkim.pub.key \
    --restart always \
    --network="sl-network" \
    simplelogin/app:3.4.0 python job_runner.py
```

Key observations:
- **No port mapping** is needed — the job runner only polls the database, it does not listen on any port.
- `--restart always` ensures automatic recovery.

**Rationale:** The job runner is a "quiet" process by design. It only logs when it finds work to do. In a freshly started system with no pending jobs, it will produce no output at all — this is normal and expected behavior. The 10-second polling interval means there will be a maximum 10-second delay between a job being queued and being picked up.

---

### A4. Logging Infrastructure

#### A4.1 Log Format

The log format is defined at **`app/log.py:L12-14`**:

```python
_log_format = (
    "%(asctime)s - %(name)s - %(levelname)s - %(process)d - "
    '"%(pathname)s:%(lineno)d" - %(funcName)s() - %(message_id)s - %(message)s'
)
```

This produces log lines with:
- **Timestamp** (`asctime`)
- **Logger name** (`name` — always `"SL"`, set at `app/log.py:L79`)
- **Log level** (`levelname` — DEBUG, INFO, WARNING, EXCEPTION)
- **Process ID** (`process`)
- **Source file path and line number** (`pathname:lineno` — clickable in PyCharm)
- **Function name** (`funcName`)
- **Message ID** (`message_id` — used for email lifecycle correlation)
- **Log message** (`message`)

#### A4.2 Log Level Shortcuts

At **`app/log.py:L73-77`**, convenience shortcuts are added to the `Logger` class:

| Shortcut | Maps To | Usage |
|----------|---------|-------|
| `LOG.d(...)` | `logging.Logger.debug` | Debug-level messages (request details, state changes) |
| `LOG.i(...)` | `logging.Logger.info` | Info-level messages (email arrival, significant events) |
| `LOG.w(...)` | `logging.Logger.warning` | Warning-level messages (PGP loading, bounce handling) |
| `LOG.e(...)` | `logging.Logger.exception` | Exception-level messages (unhandled errors with stack trace) |

#### A4.3 Email Lifecycle Correlation

The `EmailHandlerFilter` class at **`app/log.py:L28-37`** automatically injects the current `_MESSAGE_ID` into every log record:

```python
class EmailHandlerFilter(logging.Filter):
    """automatically add message-id to keep track of an email processing"""
    def filter(self, record):
        message_id = self.get_message_id()
        record.message_id = message_id if message_id else ""
        return True
```

The `set_message_id()` function at **`app/log.py:L22-25`** updates the global `_MESSAGE_ID`:

```python
def set_message_id(message_id):
    global _MESSAGE_ID
    LOG.d("set message_id %s", message_id)
    _MESSAGE_ID = message_id
```

This is called in `_handle()` (`email_handler.py:L2340`) with a `uuid.uuid4()` value, allowing all log lines for a single email to be correlated.

#### A4.4 Werkzeug Log Suppression

At **`app/log.py:L70-71`**:
```python
log = logging.getLogger("werkzeug")
log.disabled = True
```

This disables Flask's default request logging (e.g., `127.0.0.1 - - [date] "GET / HTTP/1.1" 200`). SimpleLogin replaces it with the `after_request` hook described in section A1.5.

#### A4.5 Colored Logging

When the `COLOR_LOG` configuration flag is set (from the `COLOR_LOG` environment variable), the `coloredlogs` library is enabled at **`app/log.py:L61-62`**:

```python
if COLOR_LOG:
    coloredlogs.install(level="DEBUG", logger=logger, fmt=_log_format)
```

This applies ANSI color codes to console output, making it easier to distinguish log levels during local development.

---

## Section B — User-Facing Workflow Verification

This section documents what the system produces (in logs, database records, and the dashboard UI) when a user signs in, manages aliases, and interacts with the alias management surface.

---

### B1. User Registration Flow

#### B1.1 Route Definition

The registration endpoint is defined at **`app/auth/views/register.py:L31`**:

```python
@auth_bp.route("/register", methods=["GET", "POST"])
def register():
```

The `auth_bp` blueprint (defined in `app/auth/base.py`) is mounted with no URL prefix, making the route accessible at `/auth/register`.

#### B1.2 Registration Guards

Before processing the form:

1. **Already authenticated check** (`register.py:L33-36`): If the user is already logged in, flashes `"You are already logged in"` and redirects to the dashboard.
2. **Registration disabled check** (`register.py:L38-40`): If `config.DISABLE_REGISTRATION` is `True`, flashes `"Registration is closed"` and redirects to the login page.

#### B1.3 Form Validation

The `RegisterForm` at **`register.py:L23-28`** requires:
- **Email**: non-empty (DataRequired validator)
- **Password**: non-empty, minimum 8 characters, maximum 100 characters

If `HCAPTCHA_SECRET` is configured (`register.py:L47-71`), hCaptcha validation is performed via a POST to `https://hcaptcha.com/siteverify`. A failed captcha logs a warning: `"User put wrong captcha %s %s"` (`register.py:L59-62`).

#### B1.4 Email Validation

At **`register.py:L73-83`**:
1. The email is canonicalized via `canonicalize_email()`.
2. `email_can_be_used_as_mailbox(email)` checks whether the email domain is valid and not blocked. If invalid, flashes `"You cannot use this email address as your personal inbox."`.
3. `personal_email_already_used(email)` checks for existing accounts. If duplicate, flashes `"Email {email} already used"`.

#### B1.5 User Creation (Success Path)

On successful validation (**`register.py:L85-104`**):

1. **Logs** `"create user %s"` at DEBUG level (`register.py:L85`).
2. **Creates the User record**: `User.create(email=email, name=form.email.data, password=form.password.data, referral=get_referral())` (`register.py:L86-91`).
3. **Commits** the transaction (`register.py:L92`).
4. **Sends the activation email** via `send_activation_email(user, next_url)` (`register.py:L95`).
5. **Emits** a `RegisterEvent(success)` analytics event (`register.py:L96`).
6. **Increments** `DailyMetric.nb_new_web_non_proton_user` (`register.py:L97`).
7. **Renders** the `auth/register_waiting_activation.html` template (`register.py:L104`).

**Database records created:**
- One `User` row with `activated=False` (default).
- One `ActivationCode` row (created by `send_activation_email()`).

#### B1.6 Activation Email

The `send_activation_email()` function at **`register.py:L117-129`**:
1. Deletes any prior `ActivationCode` records for the user (`register.py:L119`).
2. Creates a new `ActivationCode` with a random 30-character code (`register.py:L120`).
3. Constructs the activation link: `{URL}/auth/activate?code={activation.code}` (`register.py:L124`).
4. Calls `email_utils.send_activation_email(user, activation_link)` (`register.py:L129`).

**Local development note:** When `NOT_SEND_EMAIL=true` is set in the `.env` file (see `app/config.py:L91`: `NOT_SEND_EMAIL = "NOT_SEND_EMAIL" in os.environ`), the email is **not actually sent via SMTP**. However, the `ActivationCode` record is still created in the database and can be retrieved manually for testing.

---

### B2. Account Activation Flow

The activation endpoint is defined at **`app/auth/views/activate.py:L13`**:

```python
@auth_bp.route("/activate", methods=["GET", "POST"])
def activate():
```

#### B2.1 Activation Process

1. **If already authenticated** (`activate.py:L18-22`): Returns a 400 error with `"You are already logged in"`.
2. **Retrieves the activation code** from the `code` query parameter (`activate.py:L24-26`).
3. **Invalid code** (`activate.py:L28-36`): Returns 400 with `"Activation code cannot be found"` and triggers the rate limiter.
4. **Expired code** (`activate.py:L38-46`): Returns 400 with `"Activation code was expired"` and offers a resend link.
5. **Success** (`activate.py:L48-67`):
   - Sets `user.activated = True` (`activate.py:L49`).
   - Calls `login_user(user)` to establish the session (`activate.py:L50`).
   - Deletes the used `ActivationCode` (`activate.py:L53`).
   - Commits the transaction (`activate.py:L54`).
   - Flashes `"Your account has been activated"` (`activate.py:L56`).
   - Sends a welcome email via `email_utils.send_welcome_email(user)` (`activate.py:L58`).
   - Redirects to the `next` URL if present (logs `"redirect user to %s"` at `activate.py:L63`), otherwise redirects to `dashboard.index` (logs `"redirect user to dashboard"` at `activate.py:L66`).

**Database changes:**
- `User.activated` updated from `False` to `True`.
- `ActivationCode` row deleted.

---

### B3. User Sign-In Flow

#### B3.1 Route Definition

The login endpoint is defined at **`app/auth/views/login.py:L21-24`**:

```python
@auth_bp.route("/login", methods=["GET", "POST"])
@limiter.limit(
    "10/minute", deduct_when=lambda r: hasattr(g, "deduct_limit") and g.deduct_limit
)
def login():
```

The rate limiter allows 10 failed attempts per minute, only deducted when `g.deduct_limit` is set to `True` (which happens on wrong credentials).

#### B3.2 Already Authenticated

If the user is already logged in (`login.py:L28-34`):
- With a `next_url`: Logs `"user is already authenticated, redirect to %s"` (`login.py:L30`) and redirects.
- Without a `next_url`: Logs `"user is already authenticated, redirect to dashboard"` (`login.py:L33`) and redirects to `dashboard.index`.

#### B3.3 Credential Validation

The `LoginForm` at **`login.py:L16-18`** captures email and password. On form submission (`login.py:L40-72`):

1. **Email sanitization**: `sanitize_email()` and `canonicalize_email()` clean the input (`login.py:L41-42`).
2. **User lookup**: `User.get_by(email=email)` or `User.get_by(email=canonical_email)` (`login.py:L43`).
3. **Password check**: `user.check_password(form.password.data)` (`login.py:L45`).

#### B3.4 Failure Cases

| Condition | Flash Message | Event | Line |
|-----------|--------------|-------|------|
| Wrong email or password | `"Email or password incorrect"` | `LoginEvent(failed)` | `login.py:L45-50` |
| Account disabled | `"Your account is disabled..."` | `LoginEvent(disabled_login)` | `login.py:L51-56` |
| Scheduled for deletion | `"Your account is scheduled to be deleted on {date}"` | `LoginEvent(scheduled_to_be_deleted)` | `login.py:L57-62` |
| Not activated | `"Please check your inbox for the activation email..."` | `LoginEvent(not_activated)` | `login.py:L63-69` |

For wrong credentials, `g.deduct_limit = True` is set (`login.py:L47`) to trigger the rate limiter.

#### B3.5 Success Path

On successful credential validation (`login.py:L70-72`):
1. Emits `LoginEvent(success)`.
2. Calls `after_login(user, next_url)`.

#### B3.6 Post-Login Flow: `after_login()`

Defined at **`app/auth/views/login_utils.py:L12-45`**:

1. **FIDO enabled** (`login_utils.py:L20-27`): Stores `user.id` in `session[MFA_USER_ID]` and redirects to `auth.fido`.
2. **OTP enabled** (`login_utils.py:L28-33`): Stores `user.id` in `session[MFA_USER_ID]` and redirects to `auth.mfa`.
3. **No MFA** (`login_utils.py:L35-45`):
   - Logs `"log user %s in"` at DEBUG level (`login_utils.py:L35`).
   - Calls `login_user(user)` to establish the Flask-Login session (`login_utils.py:L36`).
   - Sets `session["sudo_time"] = int(time())` for elevated-privilege tracking (`login_utils.py:L37`).
   - Redirects to `next_url` (if present) or `dashboard.index`.

**Rationale:** The MFA check in `after_login()` is the branching point that determines whether the user goes directly to the dashboard or must complete a second authentication factor. This explains why a user with FIDO or OTP enabled will see an additional challenge screen before reaching the dashboard.

---

### B4. Dashboard and Alias Management

#### B4.1 Dashboard Landing Page

The dashboard index route is defined at **`app/dashboard/views/index.py:L55-67`**:

```python
@dashboard_bp.route("/", methods=["GET", "POST"])
@login_required
```

The `dashboard_bp` blueprint (from `app/dashboard/base.py`) is mounted at `/dashboard`, making this accessible at `/dashboard/`.

#### B4.2 Dashboard Statistics

The `get_stats(user)` function at **`index.py:L32-52`** computes the `Stats` dataclass (`index.py:L24-29`):

| Stat | Query | Line |
|------|-------|------|
| `nb_alias` | `Alias.filter_by(user_id=user.id).count()` | `index.py:L33` |
| `nb_forward` | `EmailLog` where `is_reply=False, blocked=False, bounced=False` | `index.py:L34-38` |
| `nb_reply` | `EmailLog` where `is_reply=True, blocked=False, bounced=False` | `index.py:L39-43` |
| `nb_block` | `EmailLog` where `is_reply=False, blocked=True, bounced=False` | `index.py:L44-48` |

These statistics are displayed on the dashboard UI, providing the user with a summary of their alias usage.

#### B4.3 Random Alias Creation

When the dashboard form is submitted with `form-name == "create-random-email"` (**`index.py:L97-121`**):

1. Checks `current_user.can_create_new_alias()` (`index.py:L98`) — if false, flashes `"You need to upgrade your plan to create new alias."`.
2. Determines the alias generation scheme (word-based or UUID) from the form or user preference (`index.py:L99-103`).
3. Creates the alias: `Alias.create_new_random(user=current_user, scheme=scheme)` (`index.py:L104`).
4. Sets `alias.mailbox_id = current_user.default_mailbox_id` (`index.py:L106`).
5. Commits the transaction (`index.py:L108`).
6. Logs `"create new random alias %s for user %s"` at DEBUG level (`index.py:L110`).
7. Flashes `"Alias {alias.email} has been created"` (`index.py:L111`).
8. Redirects to the dashboard with `highlight_alias_id` query parameter (`index.py:L113-121`).

**Database records created:**
- One `Alias` row with the generated email address, linked to the user and their default mailbox.

#### B4.4 Alias Deletion

When the form is submitted with `form-name == "delete-alias"` (**`index.py:L144-150`**):

1. Logs `"User {current_user} requested deletion of alias {alias}"` at INFO level (`index.py:L145`).
2. Calls `alias_utils.delete_alias(alias, current_user, AliasDeleteReason.ManualAction, commit=True)` (`index.py:L147-148`).
3. Flashes `"Alias {email} has been deleted"` (`index.py:L150`).

#### B4.5 Alias Disabling

When the form is submitted with `form-name == "disable-alias"` (**`index.py:L151-156`**):

1. Calls `alias_utils.change_alias_status(alias, enabled=False, message="Set enabled=False from dashboard")` (`index.py:L152-153`).
2. Commits the transaction (`index.py:L155`).
3. Flashes `"Alias {alias.email} has been disabled"` (`index.py:L156`).

#### B4.6 Custom Alias Creation

The custom alias route at **`app/dashboard/views/custom_alias.py:L30-34`**:

```python
@dashboard_bp.route("/custom_alias", methods=["GET", "POST"])
@limiter.limit(ALIAS_LIMIT, methods=["POST"])
@login_required
@parallel_limiter.lock(name="alias_creation")
def custom_alias():
```

Key steps:
1. Checks `current_user.can_create_new_alias()` (`custom_alias.py:L36`) — if false, logs `"%s can't create new alias"` (`custom_alias.py:L37`) and flashes an upgrade warning.
2. Validates the alias prefix with `check_alias_prefix()` (`custom_alias.py:L64`) — allows lowercase letters, numbers, dashes, dots, and underscores up to 40 characters.
3. Validates the signed suffix with `check_suffix_signature()` (`custom_alias.py:L55-60`).
4. Creates `Alias` and `AliasMailbox` records on success.

---

### B5. Seed Data for Local Development

The `fake_data()` function at **`app/fake_data.py:L40`** seeds the development database with a test user:

```python
user = User.create(
    email="john@wick.com",
    name="John Wick",
    password="password",
    activated=True,
    is_admin=True,
    otp_secret="base32secret3232",
    intro_shown=True,
    fido_uuid=None,
)
```
(**`fake_data.py:L44-54`**)

**Key properties of the seeded user:**
- Email: `john@wick.com`
- Password: `password`
- Pre-activated (`activated=True`)
- Has admin privileges (`is_admin=True`)
- OTP secret is set but OTP is not enabled by default
- Trial end is set to `None` (no trial restrictions) at `fake_data.py:L55`

This function is invoked via the `flask dummy-data` CLI command, as documented in **`CONTRIBUTING.md:L106`**:

```bash
alembic upgrade head && flask dummy-data && python3 server.py
```

After running these commands, the application is accessible at `http://localhost:7777` and the user can log in with `john@wick.com / password` (**`CONTRIBUTING.md:L109`**).

---

## Section C — End-to-End Email Flow Verification

This section traces the complete lifecycle of an inbound email arriving at an alias, documenting every log line, database record, and SMTP status code produced.

---

### C1. SMTP Entry Point: `MailHandler.handle_DATA()`

The `MailHandler` class at **`email_handler.py:L2288`** implements the aiosmtpd `handle_DATA` interface:

```python
class MailHandler:
    async def handle_DATA(self, server, session, envelope: Envelope):
```
(**`email_handler.py:L2289`**)

This async method is called by aiosmtpd for every inbound SMTP message. It:

1. **Parses the raw email** using `email.message_from_bytes(envelope.original_content)` (`email_handler.py:L2290`).
2. **Delegates to `_handle()`** for actual processing (`email_handler.py:L2292`).
3. **Catches specific exceptions:**

| Exception | Handler | Return Status | Line |
|-----------|---------|---------------|------|
| `CannotCreateContactForReverseAlias` | Logs warning about reverse-alias used in forward phase | `status.E524` ("550 SL E524 Wrong use of reverse-alias") | `email_handler.py:L2297-2307` |
| `VERPReply`, `VERPForward`, `VERPTransactional` | Logs warning about email handling failure | `status.E213` ("250 SL E213 Unknown email ignored") | `email_handler.py:L2308-2318` |
| Generic `Exception` | Logs error with full envelope details, saves message for debugging | `status.E404` ("421 SL E404 Unexpected error - Retry later") | `email_handler.py:L2319-2332` |

**Rationale:** The exception handling hierarchy ensures that known error conditions return specific status codes, while truly unexpected errors return a 4xx code (retry-able) rather than a 5xx code (permanent failure). This prevents email loss during transient errors.

---

### C2. Message Lifecycle Tracking: `_handle()`

The `_handle()` method at **`email_handler.py:L2334-2378`** wraps the core processing with lifecycle tracking:

1. **Records start time**: `start = time.time()` (`email_handler.py:L2336`).
2. **Generates unique message ID**: `message_id = str(uuid.uuid4())` (`email_handler.py:L2339`).
3. **Sets message ID for log correlation**: `set_message_id(message_id)` (`email_handler.py:L2340`), which updates the global `_MESSAGE_ID` in `app/log.py` so all subsequent log lines include this ID.
4. **Logs separator**: `"====>=====>====>====>====>====>====>====>"` (`email_handler.py:L2342`).
5. **Logs email arrival** at INFO level: `"New message, mail from %s, rctp tos %s"` (`email_handler.py:L2343-2346`).
6. **Creates application context**: `create_light_app().app_context()` (`email_handler.py:L2352`) to provide database access within the SMTP handler.
7. **Calls `handle(envelope, msg)`** — the central dispatch function (`email_handler.py:L2353`).
8. **SPF-aware status adjustment**: If the return status starts with `"5"` and the SPF check result is `fail` or `soft_fail`, the status is replaced with `E216` ("250 SL E216 Handled spf policy") to prevent bounce storms (`email_handler.py:L2357-2365`). Logs: `"Replacing 5XX to 216 status because the return-path failed the spf check"` (`email_handler.py:L2362-2364`).
9. **Logs completion** at INFO level: `"Finish mail_from %s, rcpt_tos %s, takes %s seconds with return code '%s'<<==="` (`email_handler.py:L2367-2373`).

**Example complete log lifecycle for one email:**
```
SL - DEBUG - "email_handler.py:2342" - ====>=====>====>====>====>====>====>====>
SL - INFO  - "email_handler.py:2343" - <uuid> - New message, mail from sender@example.com, rctp tos ['alias@sl.local']
...processing logs...
SL - INFO  - "email_handler.py:2367" - <uuid> - Finish mail_from sender@example.com, rcpt_tos ['alias@sl.local'], takes 0.234 seconds with return code '250 Message accepted for delivery'<<===
```

---

### C3. Central Dispatch: `handle()`

The `handle()` function at **`email_handler.py:L1945`** is the central routing logic:

```python
def handle(envelope: Envelope, msg: Message) -> str:
    """Return SMTP status"""
```

#### C3.1 Pre-Processing

1. **Sanitizes addresses**: `mail_from` and `rcpt_tos` are cleaned via `sanitize_email()` (`email_handler.py:L1948-1952`).
2. **Sets default Content-Transfer-Encoding**: If missing, defaults to `"7bit"` (`email_handler.py:L1954-1957`).
3. **Extracts Postfix queue ID**: Parses the `Received` header for message tracking (`email_handler.py:L1959-1967`). If found, calls `set_message_id(postfix_queue_id)` to replace the UUID with the Postfix ID for correlation.
4. **Ignore check**: `should_ignore(mail_from, rcpt_tos)` at `email_handler.py:L1969` — if true, logs `"Ignore email mail_from=%s rcpt_to=%s"` and returns `E204` ("250 SL E204 ignore").
5. **Header sanitization**: Cleans FROM, TO, CC, REPLY_TO, and MESSAGE_ID headers (`email_handler.py:L1973-1978`).

#### C3.2 Comprehensive Header Logging

At **`email_handler.py:L1980-1994`**, `handle()` logs all email metadata at DEBUG level:

```
==>> Handle mail_from:<from>, rcpt_tos:<tos>, header_from:<from_header>,
header_to:<to_header>, cc:<cc>, reply-to:<reply_to>, message_id:<msg_id>,
client_ip:<ip>, headers:<all_headers>, mail_options:<opts>, rcpt_options:<opts>
```

This log line is critical for debugging — it captures the complete state of the email at the dispatch point.

#### C3.3 Dispatch Routing

After pre-processing, `handle()` determines the email type and routes to the appropriate handler:

| Email Type | Detection Method | Handler | Typical Status |
|------------|-----------------|---------|----------------|
| **Forward** (to alias) | `rcpt_to` matches an alias address | `handle_forward()` | `E200` on success |
| **Reply** (to reverse-alias) | `rcpt_to` matches a `Contact.reply_email` | `handle_reply()` | `E200` on success |
| **Bounce (forward phase)** | VERP bounce detection | Bounce handling logic | `E211` |
| **Bounce (reply phase)** | VERP bounce detection | Bounce handling logic | `E212` |
| **Unsubscribe** | `/unsubscribe/` in address | Unsubscribe handling | `E202` |
| **Hotmail complaint** | Special headers detected | Complaint handler | `E208` |
| **Yahoo complaint** | Special headers detected | Complaint handler | `E210` |

---

### C4. Forward Email Flow: `handle_forward()`

The `handle_forward()` function at **`email_handler.py:L536-676`** processes emails sent to an alias address for delivery to the user's mailbox.

#### C4.1 Alias Lookup

1. **Direct lookup**: `Alias.get_by(email=alias_address)` (`email_handler.py:L543`).
2. **If not found** (`email_handler.py:L544-555`):
   - Logs `"alias %s not exist. Try to see if it can be created on the fly"` (`email_handler.py:L545-548`).
   - Attempts `try_auto_create(alias_address)` (`email_handler.py:L549`).
   - If still not found: Logs `"alias %s cannot be created on-the-fly, return 550"` (`email_handler.py:L551`).
   - Returns `E515` ("550 SL E515 Email not exist") (`email_handler.py:L555`), or `E207` if the sender should be ignored (`email_handler.py:L553`).

#### C4.2 User Status Checks

1. **Soft-deleted user** (`email_handler.py:L559-561`): Logs `"User {user} has been soft deleted"` and returns `E502` ("550 SL E502 Email not exist").
2. **Account disabled** (`email_handler.py:L563-568`): Logs `"User {user} cannot receive emails"` and returns `E504` ("550 SL E504 Account disabled").

#### C4.3 Cycle Detection

At **`email_handler.py:L570-577`**: If the email's `mail_from` matches one of the alias's authorized addresses (i.e., the user's own mailbox), it's a cycle email:
- Logs `"cycle email sent from %s to %s"` (`email_handler.py:L575`).
- Calls `handle_email_sent_to_ourself()` to send a notification.
- Returns `E209` ("250 SL E209 Email Loop").

#### C4.4 Contact Creation

At **`email_handler.py:L579-581`**:
1. Extracts the `From` header: `from_header = get_header_unicode(msg[headers.FROM])`.
2. Logs `"Create or get contact for from_header:%s"` (`email_handler.py:L580`).
3. Calls `get_or_create_contact(from_header, envelope.mail_from, alias)` (`email_handler.py:L581`).

**Database records created/accessed:**
- A `Contact` row is either found (by `alias_id` + `website_email`) or created with:
  - `user_id`: The alias owner's ID
  - `alias_id`: The target alias's ID
  - `website_email`: The sender's email address
  - `reply_email`: A generated reverse-alias address for replying

#### C4.5 Blocked Email Handling

If the alias is disabled or the contact has `block_forward=True` (**`email_handler.py:L596-612`**):

1. Logs `"%s is disabled, do not forward"` (`email_handler.py:L597`).
2. Creates an `EmailLog` with `blocked=True` (`email_handler.py:L598-604`).
3. Returns `E200` by default, or `E502` if `user.block_behaviour == BlockBehaviourEnum.return_5xx` (`email_handler.py:L608-610`).

**Rationale:** Returning `E200` (250 accept) for blocked emails is intentional — it prevents the sending MTA from retrying or generating bounce messages, which would reveal the existence of the alias.

#### C4.6 DMARC Policy

At **`email_handler.py:L614-619`**: `apply_dmarc_policy_for_forward_phase(alias, contact, envelope, msg)` evaluates the sender's DMARC policy. If the policy requires rejection, a `dmarc_delivery_status` is returned and `handle_forward()` returns early.

#### C4.7 Mailbox Delivery

For each mailbox associated with the alias (**`email_handler.py:L632-676`**):

| Condition | Action | Status |
|-----------|--------|--------|
| Mailbox not verified | Skip, log `"%s unverified, do not forward"` | `E517` |
| Mailbox email is also an alias (loop) | Mark mailbox unverified, send alert email | `E525` |
| Valid mailbox | Call `forward_email_to_mailbox()` | Per-mailbox result |

#### C4.8 `forward_email_to_mailbox()`

Defined at **`email_handler.py:L679-740`**:

1. Logs `"Forward %s -> %s -> %s"` showing contact → alias → mailbox (`email_handler.py:L688`).
2. **Disabled mailbox check** (`email_handler.py:L690-695`): Returns `E518` if disabled.
3. **Same-domain conflict** (`email_handler.py:L698-730`): If alias and mailbox share the same domain, sends an alert and returns `E405` ("421 SL E405 Mailbox domain problem - Retry later").
4. **Creates EmailLog** (`email_handler.py:L732-739`):
   ```python
   email_log = EmailLog.create(
       contact_id=contact.id,
       user_id=contact.user_id,
       mailbox_id=mailbox.id,
       alias_id=contact.alias_id,
       message_id=str(msg[headers.MESSAGE_ID]),
       commit=True,
   )
   ```
5. Logs `"Create %s for %s, %s, %s"` showing the EmailLog, contact, user, and mailbox (`email_handler.py:L740`).

**Database records created:**
- One `EmailLog` row per mailbox delivery with:
  - `contact_id`: The sender contact's ID
  - `user_id`: The alias owner's ID
  - `mailbox_id`: The target mailbox's ID
  - `alias_id`: The alias's ID
  - `message_id`: The email's Message-ID header
  - `is_reply=False` (default), `blocked=False` (default), `bounced=False` (default)

---

### C5. SMTP Status Codes Reference

All SMTP status codes are defined in **`app/email/status.py`**. These codes are returned by the email handler to the sending MTA and indicate the outcome of email processing.

#### 2xx Success Codes (250 — Message Accepted)

| Code | Value | Meaning |
|------|-------|---------|
| `E200` | `"250 Message accepted for delivery"` | Successful forward or reply |
| `E201` | `"250 SL E201"` | Generic success |
| `E202` | `"250 Unsubscribe request accepted"` | Unsubscribe processed |
| `E203` | `"250 SL E203 email can't be sent from a reverse-alias"` | Reverse-alias misuse (accepted silently) |
| `E204` | `"250 SL E204 ignore"` | Email intentionally ignored |
| `E205` | `"250 SL E205 bounce handled"` | Bounce notification processed |
| `E206` | `"250 SL E206 Out of office"` | Out-of-office auto-reply handled |
| `E207` | `"250 SL E207 No bounce report"` | Sender is an IgnoreBounceSender |
| `E208` | `"250 SL E208 Hotmail complaint handled"` | Hotmail/Outlook complaint processed |
| `E209` | `"250 SL E209 Email Loop"` | Cycle detected (mailbox → own alias) |
| `E210` | `"250 SL E210 Yahoo complaint handled"` | Yahoo complaint processed |
| `E211` | `"250 SL E211 Bounce Forward phase handled"` | VERP bounce in forward phase |
| `E212` | `"250 SL E212 Bounce Reply phase handled"` | VERP bounce in reply phase |
| `E213` | `"250 SL E213 Unknown email ignored"` | VERP-related unknown email ignored |
| `E214` | `"250 SL E214 Unauthorized for using reverse alias"` | Unauthorized reverse-alias usage |
| `E215` | `"250 SL E215 Handled dmarc policy"` | DMARC policy enforced |
| `E216` | `"250 SL E216 Handled spf policy"` | SPF policy enforced (replaced 5xx) |

#### 4xx Retry Codes (421 — Temporary Failure)

| Code | Value | Meaning |
|------|-------|---------|
| `E402` | `"421 SL E402 Encryption failed - Retry later"` | PGP/GPG encryption failure |
| `E404` | `"421 SL E404 Unexpected error - Retry later"` | Unhandled exception (retry-able) |
| `E405` | `"421 SL E405 Mailbox domain problem - Retry later"` | Alias and mailbox share same domain |
| `E407` | `"421 SL E407 Retry later"` | Generic temporary failure |

#### 5xx Permanent Error Codes (550 — Permanent Failure)

| Code | Value | Meaning |
|------|-------|---------|
| `E501` | `"550 SL E501"` | Generic permanent error |
| `E502` | `"550 SL E502 Email not exist"` | Soft-deleted user or blocked alias (5xx mode) |
| `E503` | `"550 SL E503"` | Generic permanent error |
| `E504` | `"550 SL E504 Account disabled"` | User cannot send or receive |
| `E505` | `"550 SL E505"` | Generic permanent error |
| `E506` | `"550 SL E506 Email detected as spam"` | SpamAssassin flagged as spam |
| `E507` | `"550 SL E507 Wrongly formatted subject"` | Invalid subject format |
| `E508` | `"550 SL E508 Email not exist"` | Address not found |
| `E509` | `"550 SL E509 unauthorized"` | Unauthorized sender |
| `E510` | `"550 SL E510 so such user"` | User not found |
| `E511` | `"550 SL E511 unsubscribe error"` | Unsubscribe processing error |
| `E512` | `"550 SL E512 No such email log"` | Referenced email log not found |
| `E514` | `"550 SL E514 Email sent to noreply address"` | Noreply address rejection |
| `E515` | `"550 SL E515 Email not exist"` | Alias does not exist and cannot be auto-created |
| `E516` | `"550 SL E516 invalid mailbox"` | No valid mailboxes for alias |
| `E517` | `"550 SL E517 unverified mailbox"` | Mailbox not verified |
| `E518` | `"550 SL E518 Disabled mailbox"` | Mailbox is disabled |
| `E519` | `"550 SL E519 Email detected as spam"` | Spam detected (alternate code) |
| `E521` | `"550 SL E521 Cannot reach mailbox"` | Mailbox unreachable |
| `E522` | `"550 SL E522 The user you are trying to contact is receiving mail at a rate that prevents additional messages from being delivered."` | Rate-limited recipient |
| `E523` | `"550 SL E523 Unknown error"` | Unknown error |
| `E524` | `"550 SL E524 Wrong use of reverse-alias"` | Reverse-alias used in forward phase |
| `E525` | `"550 SL E525 Alias loop"` | Mailbox-is-alias loop detected |

---

### C6. Database Records Created During Email Processing

#### C6.1 `Contact` Model

Defined at **`app/models.py:L1863`**, the `Contact` table stores external senders who have emailed an alias:

| Column | Type | Purpose |
|--------|------|---------|
| `user_id` | FK → `User.id` | The alias owner |
| `alias_id` | FK → `Alias.id` | The alias that received the email |
| `website_email` | String(512) | The sender's actual email address |
| `website_from` | String(1024) | The full `From` header (e.g., `"Alice <alice@example.com>"`) |
| `reply_email` | String(512) | The generated reverse-alias address for replies |
| `name` | String(512) | The sender's display name |
| `is_cc` | Boolean | Whether the contact was created via CC |
| `mail_from` | Text | The SMTP envelope `MAIL FROM` address |
| `invalid_email` | Boolean | Whether the contact has an empty/invalid email |

**Uniqueness constraint:** `(alias_id, website_email)` — one contact per sender per alias (`app/models.py:L1875`).

#### C6.2 `EmailLog` Model

Defined at **`app/models.py:L2060`**, the `EmailLog` table is the audit trail for every email processed:

| Column | Type | Purpose |
|--------|------|---------|
| `user_id` | FK → `User.id` | The alias owner |
| `contact_id` | FK → `Contact.id` | The sender contact |
| `alias_id` | FK → `Alias.id` | The alias involved |
| `mailbox_id` | FK → `Mailbox.id` | The target mailbox (for forwards) |
| `message_id` | String | The email's `Message-ID` header |
| `is_reply` | Boolean | `True` if this is a reply (user → contact) |
| `blocked` | Boolean | `True` if the email was blocked (alias disabled or contact blocked) |
| `bounced` | Boolean | `True` if the email bounced |
| `auto_replied` | Boolean | `True` if this was an auto-reply (vacation) |
| `is_spam` | Boolean | `True` if SpamAssassin flagged it |
| `spam_score` | Float | SpamAssassin score |
| `spam_status` | Text | SpamAssassin status text |

**Index:** `ix_email_log_created_at` on `created_at` for efficient time-based queries.

#### C6.3 `Notification` Model

Defined at **`app/models.py:L3065`**, in-app notifications displayed in the dashboard:

| Column | Type | Purpose |
|--------|------|---------|
| `user_id` | FK → `User.id` | The notification recipient |
| `message` | Text | HTML notification content |
| `title` | String(512) | Notification title |
| `read` | Boolean | Whether the user has read it |

Notifications are generated for events like cycle email detection, mailbox validation failures, and account-level alerts.

---

## Section D — Background Component Behavior

This section answers whether the email handler and job runner start automatically with the web server, and what runtime behavior confirms they are functioning correctly.

---

### D1. Process Independence

**The email handler and job runner are NOT started automatically by the web server.** Each of the three core processes is an independent entry point that must be launched separately.

#### D1.1 Source Code Evidence

Each process has its own `__main__` or `__name__ == "__main__"` block:

| Process | Entry Point | Main Function |
|---------|------------|---------------|
| Web server | `server.py:L598-599` → `local_main()` | `create_app()` → `app.run()` |
| Email handler | `email_handler.py:L2396-2404` | `main(port)` → `Controller.start()` |
| Job runner | `job_runner.py:L329-347` | `while True:` polling loop |

There is **no code** in `create_app()`, `local_main()`, or anywhere in `server.py` that imports, spawns, or starts either the email handler or job runner.

#### D1.2 Documentation Evidence

**`CONTRIBUTING.md:L143-147`** explicitly documents three separate entry points:

> *"The repo consists of the three following entry points:*
> - *wsgi.py and server.py: the webapp.*
> - *email_handler.py: the email handler.*
> - *cron.py: the cronjob."*

Local development instructions in **`CONTRIBUTING.md`** specify separate commands:
- Web server (`CONTRIBUTING.md:L106`): `alembic upgrade head && flask dummy-data && python3 server.py`
- Email handler (`CONTRIBUTING.md:L212`): `python email_handler.py`
- Job runner (`CONTRIBUTING.md:L228`): `python job_runner.py`

#### D1.3 Docker Deployment Evidence

The Docker deployment at **`README.md:L454-496`** uses three separate containers from the same image:

| Container | Command | Port | Line |
|-----------|---------|------|------|
| `sl-app` | Default (Gunicorn via Dockerfile CMD) | `127.0.0.1:7777:7777` | `README.md:L454-464` |
| `sl-email` | `python email_handler.py` | `127.0.0.1:20381:20381` | `README.md:L470-481` |
| `sl-job-runner` | `python job_runner.py` | None needed | `README.md:L486-496` |

All three containers use `--restart always` for automatic recovery.

Two additional initialization containers are run once before starting the services:
- `sl-migration` (`README.md:L425-434`): Runs `flask db upgrade` to apply database migrations.
- `sl-init` (`README.md:L441-449`): Runs `python init_app.py` to seed SL domains and configuration.

**Rationale:** The multi-container architecture follows the Unix philosophy of "do one thing well." Each process has a focused responsibility, can be scaled independently, and can be restarted without affecting the others. The email handler and job runner share the same codebase and configuration but operate as completely independent processes.

---

### D2. Job Runner Operational Behavior

#### D2.1 Job Types

The `process_job()` function at **`job_runner.py:L188-304`** handles the following job types:

| Job Name | Config Constant | Description | Lines |
|----------|----------------|-------------|-------|
| `onboarding-1` | `JOB_ONBOARDING_1` | Send "send-from-alias" onboarding email | `L189-197` |
| `onboarding-2` | `JOB_ONBOARDING_2` | Send "mailbox" onboarding email | `L198-206` |
| `onboarding-4` | `JOB_ONBOARDING_4` | Send "PGP" onboarding email (skipped for Proton mailboxes) | `L207-220` |
| `batch-import` | `JOB_BATCH_IMPORT` | Process CSV batch alias import | `L222-225` |
| `delete-account` | `JOB_DELETE_ACCOUNT` | Delete user account and send confirmation email | `L226-244` |
| `delete-mailbox` | `JOB_DELETE_MAILBOX` | Delete a mailbox | `L245-246` |
| `delete-domain` | `JOB_DELETE_DOMAIN` | Delete custom domain with audit logging | `L248-284` |
| `send-user-report` | `JOB_SEND_USER_REPORT` | GDPR data export | `L285-288` |
| `send-proton-welcome-1` | `JOB_SEND_PROTON_WELCOME_1` | Proton welcome email | `L289-294` |
| `send-alias-creation-events` | `JOB_SEND_ALIAS_CREATION_EVENTS` | Dispatch alias creation events | `L295-302` |
| Unknown | N/A | Logs error `"Unknown job name %s"` | `L304` |

#### D2.2 Job State Machine

The `JobState` enum at **`app/models.py:L253-257`**:

```python
class JobState(EnumE):
    ready = 0
    taken = 1
    done = 2
    error = 3
```

**State transitions in the main loop** (`job_runner.py:L337-345`):

```
ready (0) ──[Take]──► taken (1) ──[Process]──► done (2)
              │                        │
              │                        └──[Error]──► error (3)
              │
              └──[Stale? Retry]──► taken (1)  (if attempts < 5)
```

When a job is picked up:
1. `job.taken = True` (`job_runner.py:L337`)
2. `job.taken_at = arrow.now()` (`job_runner.py:L338`)
3. `job.state = JobState.taken.value` (`job_runner.py:L339`)
4. `job.attempts += 1` (`job_runner.py:L340`)
5. Transaction committed (`job_runner.py:L341`)
6. `process_job(job)` executes (`job_runner.py:L342`)
7. `job.state = JobState.done.value` (`job_runner.py:L344`)
8. Transaction committed (`job_runner.py:L345`)

#### D2.3 Retry Mechanism

The `get_jobs_to_run()` function at **`job_runner.py:L307-326`** includes a retry mechanism for stale jobs:

A job is considered stale and eligible for retry when:
- `state == JobState.taken.value` (was picked up but never completed)
- `taken_at` is older than 30 minutes (`config.JOB_TAKEN_RETRY_WAIT_MINS`)
- `attempts < config.JOB_MAX_ATTEMPTS` (default: 5)

This handles scenarios where a job runner process crashes mid-processing — after 30 minutes, the next polling cycle will retry the job automatically.

#### D2.4 Log Patterns

| Event | Log Level | Message Pattern | Source |
|-------|-----------|----------------|--------|
| Job picked up | DEBUG | `"Take job <Job ID name payload>"` | `job_runner.py:L334` |
| Onboarding email | DEBUG | `"send onboarding send-from-alias email to user %s"` | `job_runner.py:L196` |
| Account deletion | WARNING | `"Delete user %s"` | `job_runner.py:L235` |
| Domain deletion | DEBUG | `"Domain %s deleted"` | `job_runner.py:L272` |
| Unknown job | EXCEPTION | `"Unknown job name %s"` | `job_runner.py:L304` |
| No user found | INFO | `"No user found for %s"` | `job_runner.py:L231` |

---

### D3. Email Handler Operational Behavior

#### D3.1 Runtime Characteristics

The email handler operates as a **long-lived async SMTP server**:

- The aiosmtpd `Controller` runs the SMTP protocol in a background thread (`email_handler.py:L2383-2385`).
- The main thread enters an infinite `time.sleep(2)` loop (`email_handler.py:L2392-2393`) to keep the process alive.
- Each inbound email triggers `MailHandler.handle_DATA()` asynchronously.
- Database access is obtained per-email via `create_light_app().app_context()` (`email_handler.py:L2352`).

#### D3.2 Key Log Patterns

**Startup sequence:**
```
SL - INFO  - Listen for port 20381
SL - DEBUG - Start mail controller 0.0.0.0 20381
```

**Per-email processing (success):**
```
SL - DEBUG - ====>=====>====>====>====>====>====>====>
SL - INFO  - <uuid> - New message, mail from sender@example.com, rctp tos ['alias@sl.local']
SL - DEBUG - <uuid> - ==>> Handle mail_from:sender@example.com, rcpt_tos:['alias@sl.local'], ...
SL - DEBUG - <uuid> - Create or get contact for from_header:sender@example.com
SL - DEBUG - <uuid> - Forward sender@example.com -> alias@sl.local -> user@mailbox.com
SL - DEBUG - <uuid> - Create <EmailLog ID> for <Contact>, <User>, <Mailbox>
SL - INFO  - <uuid> - Finish mail_from sender@example.com, rcpt_tos ['alias@sl.local'], takes 0.234 seconds with return code '250 Message accepted for delivery'<<===
```

**Per-email processing (alias not found):**
```
SL - DEBUG - ====>=====>====>====>====>====>====>====>
SL - INFO  - <uuid> - New message, mail from sender@example.com, rctp tos ['unknown@sl.local']
SL - DEBUG - <uuid> - alias unknown@sl.local not exist. Try to see if it can be created on the fly
SL - DEBUG - <uuid> - alias unknown@sl.local cannot be created on-the-fly, return 550
SL - INFO  - <uuid> - Finish mail_from sender@example.com, rcpt_tos ['unknown@sl.local'], takes 0.012 seconds with return code '550 SL E515 Email not exist'<<===
```

**Error conditions:**
```
SL - EXCEPTION - email handling fail with error:<error_details> mail_from:<from>, rcpt_tos:<tos>, ...
```

#### D3.3 The `create_light_app()` Pattern

Both the email handler (`email_handler.py:L2352`) and job runner (`job_runner.py:L332`) use `create_light_app()` (defined at `server.py:L127-136`) to obtain a minimal Flask application context:

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

This lightweight app factory provides database connectivity without the overhead of the full web application (blueprints, middleware, session management, etc.). The `shutdown_session` teardown ensures proper database connection cleanup after each use.

---

### D4. Cron Job Behavior

The cron scheduler is driven by `crontab.yml` via yacron. The `cron.py` entry point handles the following scheduled tasks (from **`crontab.yml`**):

| Job | Schedule | Command |
|-----|----------|---------|
| Growth stats | Daily at 00:00 | `python /code/cron.py -j stats` |
| Delete old monitoring | Daily at 01:15 | `python /code/cron.py -j delete_old_monitoring` |
| Custom domain check | Daily at 02:15 | `python /code/cron.py -j check_custom_domain` |
| HIBP breach check | Daily at 03:15 | `python /code/cron.py -j check_hibp` |
| Notify HIBP breaches | Daily at 04:15 | `python /code/cron.py -j notify_hibp` |
| Delete logs | Daily at 05:15 | `python /code/cron.py -j delete_logs` |
| Delete old data | Daily at 05:30 | `python /code/cron.py -j delete_old_data` |
| Poll Apple subscriptions | Daily at 06:15 | `python /code/cron.py -j poll_apple_subscription` |
| Notify trial end | Daily at 08:15 | `python /code/cron.py -j notify_trial_end` |
| Notify manual subscription end | Daily (scheduled) | `python /code/cron.py -j notify_manual_subscription_end` |

**Note:** As documented in `CONTRIBUTING.md:L143-147`, `cron.py` is the third entry point alongside `wsgi.py`/`server.py` and `email_handler.py`. The job runner (`job_runner.py`) is a fourth implicit entry point that handles on-demand jobs rather than scheduled tasks.

---

## Section E — Supplementary Information

### E1. Configuration Reference for Local Development

Key environment variables from **`app/config.py`** and **`example.env`**:

| Variable | Default | Purpose | Source |
|----------|---------|---------|--------|
| `NOT_SEND_EMAIL` | `true` (in example.env) | When set, suppresses actual SMTP delivery — essential for local dev | `app/config.py:L91` |
| `URL` | `http://localhost:7777` | Base URL for the application | `example.env:L6` |
| `EMAIL_DOMAIN` | `sl.local` | Domain used to create alias email addresses | `example.env:L22` |
| `SUPPORT_EMAIL` | `support@sl.local` | Transactional email sender address | `example.env:L40` |
| `DB_URI` | (must be configured) | PostgreSQL connection string | `app/config.py` |
| `COLOR_LOG` | unset | Enables colored console log output via `coloredlogs` | `example.env:L16` |
| `POSTFIX_SERVER` | `localhost` (for testing) | Outbound SMTP server | `CONTRIBUTING.md:L205` |
| `POSTFIX_PORT` | `1025` (for testing) | Outbound SMTP port | `CONTRIBUTING.md:L206` |
| `DISABLE_REGISTRATION` | unset | When set, blocks new user registrations | `app/config.py` |
| `FLASK_SECRET` | (must be configured) | Flask session encryption key | `app/config.py` |

**Critical for local development:** The `NOT_SEND_EMAIL` flag at `app/config.py:L91` is a simple environment presence check:
```python
NOT_SEND_EMAIL = "NOT_SEND_EMAIL" in os.environ
```
When this variable exists in the environment (any value), all outbound email delivery is suppressed. The email content is logged but not sent via SMTP.

---

### E2. Testing and Diagnostic Tools

#### E2.1 Testing Email with `swaks`

As documented in **`CONTRIBUTING.md:L186-221`**:

**Prerequisites:**
1. Run a local MTA (mailcatcher or MailHog) to capture forwarded emails.
2. Configure `.env`:
   ```
   # Comment out NOT_SEND_EMAIL
   # NOT_SEND_EMAIL=true

   POSTFIX_SERVER=localhost
   POSTFIX_PORT=1025
   ```
3. Start the email handler: `python email_handler.py`

**Test command:**
```bash
swaks --to e1@sl.local --from hey@google.com --server 127.0.0.1:20381
```

**Expected results:**
- The email handler logs show the processing pipeline (see section D3.2).
- The forwarded email appears in the local MTA's web UI (typically at `http://localhost:1080/`).
- The `EmailLog` and `Contact` tables gain new records.

#### E2.2 Troubleshooting Procedures

From **`docs/troubleshooting.md`**:

**Problem A: Welcome email not received after registration**
1. Test Postfix delivery: `swaks --to your-mailbox@mail.com`
2. If that works, test container-to-Postfix connectivity:
   ```bash
   docker exec -it sl-app bash
   apt update && apt install telnet -y
   telnet 10.0.0.1 25
   ```
3. If `10.0.0.1` doesn't work, try `172.17.0.1` (Docker default host IP) and set `POSTFIX_SERVER=172.17.0.1` in the config.

**Problem B: Alias forwarding not working**
1. Verify Postfix recognizes the alias domain:
   ```bash
   postmap -q mydomain.com pgsql:/etc/postfix/pgsql-relay-domains.cf
   # Should return: mydomain.com
   ```
2. Verify transport mapping:
   ```bash
   postmap -q mydomain.com pgsql:/etc/postfix/pgsql-transport-maps.cf
   # Should return: smtp:127.0.0.1:20381
   ```
3. Check `sl-email` container logs: `docker logs sl-email`

---

### E3. Monitoring

The `monitoring.py` module (at **`monitoring.py:L1`**) exports operational metrics:

- **Postfix queue sizes** (`monitoring.py:L39-48`): `log_postfix_metrics()` checks `/var/spool/postfix/incoming`, `/var/spool/postfix/active`, and `/var/spool/postfix/deferred` queue directories. Logs: `"postfix queue sizes %s %s %s"`.
- **Custom New Relic metrics**: `Custom/postfix_incoming_queue`, `Custom/postfix_active_queue`, `Custom/postfix_deferred_queue` (`monitoring.py:L46-48`).
- **Alert threshold**: If `incoming_queue + active_queue > 50` for more than 10 consecutive checks (`_max_nb_fails = 10`), an alert is triggered (`monitoring.py:L16-23`).

---

## Appendix: Quick-Start Verification Checklist

For a newly launched SimpleLogin instance, use this checklist to confirm all components are operational:

### 1. Web Server
- [ ] `curl http://localhost:7777/health` returns `"success"` with HTTP 200
- [ ] Request logs appear showing `<ip> GET /dashboard/ ... 200, takes <elapsed>`
- [ ] Dashboard loads at `http://localhost:7777/dashboard/`

### 2. Email Handler
- [ ] Logs show `"Listen for port 20381"` followed by `"Start mail controller 0.0.0.0 20381"`
- [ ] SMTP connection succeeds: `swaks --to test@sl.local --from test@example.com --server 127.0.0.1:20381`
- [ ] `docker logs sl-email` (for Docker deployments) shows no errors

### 3. Job Runner
- [ ] Process is running (check with `ps aux | grep job_runner` or `docker ps`)
- [ ] When a job is queued (e.g., after registration), `"Take job ..."` appears in logs
- [ ] Jobs in the database transition from `state=0` (ready) to `state=2` (done)

### 4. User Workflow
- [ ] Registration at `/auth/register` creates a `User` record and `ActivationCode`
- [ ] Login at `/auth/login` with valid credentials redirects to `/dashboard/`
- [ ] Dashboard shows `Stats` (alias count, forward count, reply count, block count)
- [ ] Creating a random alias flashes `"Alias ... has been created"`

### 5. Email Flow
- [ ] Sending to a valid alias produces `"New message..."` → `"Finish ... with return code '250 Message accepted for delivery'"` in email handler logs
- [ ] `Contact` and `EmailLog` records appear in the database
- [ ] Sending to a non-existent alias returns `"550 SL E515 Email not exist"`

---

*This document was generated through source code analysis of the SimpleLogin application repository. All claims are grounded in specific file paths, line numbers, and function references as cited throughout.*
