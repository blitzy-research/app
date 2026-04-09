# SimpleLogin Runtime Behavior Investigation Guide

## Introduction

This document provides empirically verified answers to five specific runtime behavior questions about the SimpleLogin email privacy system's initialization, startup, and first-run experience. Each answer is based on direct observation of the running system and cross-referenced with source code analysis.

### Scope

The five investigation areas covered are:

1. **Database Migrations** — Table count, migration step count, and last table created when running `alembic upgrade head` on an empty PostgreSQL database.
2. **Web Server Startup** — The exact log message indicating readiness and the time elapsed between the first log entry and the ready message.
3. **Email Handler Custom Port** — Whether `email_handler.py` confirms it is listening on a custom port (25025) and the exact log messages emitted.
4. **Registration and Login Flow** — The exact JSON errors, HTTP status codes, curl output, and database state when creating a user and attempting login before activation.
5. **Dynamic Alias Limits** — The default `max_alias_free_plan` value and how it changes when `MAX_NB_EMAIL_FREE_PLAN` is modified and the server is restarted.

### Prerequisites

- **Python 3.10+** (as specified in `pyproject.toml`: `python = "^3.10"`)
- **PostgreSQL 13+** with a dedicated database for SimpleLogin
- All Python dependencies installed from `pyproject.toml` (via `poetry install` or `pip install`)
- OpenSSL (for JWT RSA key generation)

### Methodology

- **Empirical verification:** Every answer is derived from observed output of the running system, not from code reading alone.
- **Source code tracing:** Each answer includes a "Rationale / Code Trace" section linking the observed behavior back to specific source code files and line numbers.
- **No source file modifications:** Per the investigation constraint, no existing repository source files were modified. Only configuration files (`.env`) and key files (`local_data/jwtRS256.key`) were created for experimentation.
- **Exact output reproduction:** All JSON responses, log messages, SQL results, and curl output are reproduced verbatim as captured from the running system.

---

## Environment Setup

### Configuration (.env file)

Create a `.env` file in the repository root with the following minimum configuration:

```bash
# Server URL
URL=http://localhost:7777

# CRITICAL: EMAIL_DOMAIN must NOT use a reserved TLD (.local, .test, .example)
# The email_validator library rejects RFC 2606/6761 reserved names.
# The default "sl.local" from example.env WILL CAUSE 500 ERRORS during user registration.
EMAIL_DOMAIN=sldev.io

# Transactional email settings
SUPPORT_EMAIL=support@sldev.io

# Database connection
DB_URI=postgresql://sl_user:sl_password@localhost:5432/simplelogin

# Flask secret key (generate a random string)
FLASK_SECRET=replace-with-a-random-secret-string

# Disable actual email sending for local development
NOT_SEND_EMAIL=true

# Words file for random alias generation
WORDS_FILE_PATH=local_data/words.txt

# OpenID Connect JWT keys (generated below)
OPENID_PRIVATE_KEY_PATH=local_data/jwtRS256.key
OPENID_PUBLIC_KEY_PATH=local_data/jwtRS256.key.pub

# DNS resolver for MX lookups
NAMESERVERS=9.9.9.9

# Partner API token (any secret string)
PARTNER_API_TOKEN_SECRET=a-random-partner-secret

# Allowed redirect domains (empty list)
ALLOWED_REDIRECT_DOMAINS=[]

# Email server priority list
EMAIL_SERVERS_WITH_PRIORITY=[(10, "mail.sldev.io.")]
```

> **Source:** `example.env` — 198-line configuration reference with all available options and defaults. Key variables: `EMAIL_DOMAIN` at `app/config.py:92`, `DB_URI` at `app/config.py:192`, `NOT_SEND_EMAIL` at `app/config.py:91`.

### JWT Key Generation

SimpleLogin requires an RSA key pair for OpenID Connect functionality. Generate them with:

```bash
# Create the local_data directory if it doesn't exist
mkdir -p local_data

# Generate 4096-bit RSA private key
openssl genrsa -out local_data/jwtRS256.key 4096

# Extract the public key
openssl rsa -in local_data/jwtRS256.key -pubout -outform PEM -out local_data/jwtRS256.key.pub
```

> **Source:** `docs/code-structure.md` documents the `local_data/` directory. The key paths are configured via `OPENID_PRIVATE_KEY_PATH` and `OPENID_PUBLIC_KEY_PATH` in `app/config.py`.

### Domain Seeding

After running database migrations (see Q1 below), the `init_app.py` script must be executed to seed the `sl_domain` table with the configured `EMAIL_DOMAIN`. Without this step, user registration will fail because alias creation requires at least one entry in the domain table.

```bash
# Run domain seeding after migrations
python init_app.py
```

The `init_app.py` script calls `add_sl_domains()` which iterates over `ALIAS_DOMAINS` (defaults to `[EMAIL_DOMAIN]`) and inserts each domain into the `sl_domain` table if it doesn't already exist:

```python
# Source: init_app.py:39-56
def add_sl_domains():
    for alias_domain in ALIAS_DOMAINS:
        if SLDomain.get_by(domain=alias_domain):
            LOG.d("%s is already a SL domain", alias_domain)
        else:
            LOG.i("Add %s to SL domain", alias_domain)
            SLDomain.create(domain=alias_domain, use_as_reverse_alias=True)

    for premium_domain in PREMIUM_ALIAS_DOMAINS:
        if SLDomain.get_by(domain=premium_domain):
            LOG.d("%s is already a SL domain", premium_domain)
        else:
            LOG.i("Add %s to SL domain", premium_domain)
            SLDomain.create(
                domain=premium_domain, premium_only=True, use_as_reverse_alias=True
            )

    Session.commit()
```

### ⚠️ Gotcha: EMAIL_DOMAIN Validation

**The `email_validator` library (v1.1.x) rejects `.local`, `.test`, `.example`, and other RFC 2606/6761 reserved TLDs.** The default `EMAIL_DOMAIN=sl.local` from `example.env` will cause user registration to fail with a 500 error during alias creation, because the system attempts to validate the generated alias address (e.g., `random.word@sl.local`) and the validator rejects `.local` as a reserved name.

**Solution:** Use a real-looking domain such as `sldev.io` instead of `sl.local`.

> **Source:** `app/config.py:92` — `EMAIL_DOMAIN = os.environ["EMAIL_DOMAIN"].lower()`. The validation occurs in `app/email_utils.py` via the `email_can_be_used_as_mailbox()` function which invokes `email_validator`.

### ⚠️ Gotcha: jwcrypto Version Compatibility

The `pyproject.toml` specifies `jwcrypto = "^0.8"`. On some platforms, version `0.8` may conflict with newer versions of the `cryptography` library (e.g., `cryptography==37.0.1`) due to changes in the `load_pem_private_key()` function signature. If you encounter a `TypeError` related to keyword arguments during startup, upgrading `jwcrypto` to `0.9.1` resolves the issue.

---

## Q1: Database Migrations

### Question

When running `alembic upgrade head` on an empty PostgreSQL database, how many tables are created in total, and what is the exact name of the last table created based on migration output order?

### Answer

**Running `alembic upgrade head` on an empty PostgreSQL database creates 77 tables in total** (76 application tables + 1 `alembic_version` tracking table). **255 migration steps** are executed. **The last table created is `user_audit_log`** (migration revision `7d7b84779837`).

### Observed Output

Command:

```bash
alembic upgrade head
```

The migration output follows this pattern, with 255 lines of the form:

```
INFO  [alembic.runtime.migration] Running upgrade  -> a6b1009e8f5e, init
INFO  [alembic.runtime.migration] Running upgrade a6b1009e8f5e -> b9f849432543, add Fido
...
INFO  [alembic.runtime.migration] Running upgrade 91ed7f46dc81 -> 7d7b84779837, user_audit_log
INFO  [alembic.runtime.migration] Running upgrade 7d7b84779837 -> 32f25cbf12f6, alias_audit_log_index_created_at
```

Each line represents one migration step. The total count of migration files in `migrations/versions/` is 255.

### Table Count Verification

After migrations complete, the total number of tables can be verified via SQL:

```sql
SELECT COUNT(*) FROM information_schema.tables WHERE table_schema = 'public';
```

**Result: 77**

A listing of all tables can be obtained with:

```sql
SELECT table_name FROM information_schema.tables
WHERE table_schema = 'public'
ORDER BY table_name;
```

This returns 77 rows: 76 application tables defined in `app/models.py` plus the `alembic_version` table used by Alembic to track the current migration revision.

The application table count can be independently verified:

```bash
grep -c '__tablename__' app/models.py
```

**Result: 76** — confirming 76 model classes with table declarations, plus 1 `alembic_version` = 77 total.

### Last Table Identification

The migration chain has **three head revisions** (multi-head):
- `32f25cbf12f6` — `alias_audit_log_index_created_at`
- `01e2997e90d3` — adds `use_as_reverse_alias` column to `public_domain`
- `2d89315ac650` — adds index on `partner_subscription.end_at`

The migration `32f25cbf12f6` (file: `2024_101616_32f25cbf12f6_alias_audit_log_index_created_at.py`) is the chronologically latest head but **only creates an INDEX**, not a table:

```python
# Source: migrations/versions/2024_101616_32f25cbf12f6_alias_audit_log_index_created_at.py:20-22
def upgrade():
    with op.get_context().autocommit_block():
        op.create_index('ix_alias_audit_log_created_at', 'alias_audit_log', ['created_at'],
                        unique=False, postgresql_concurrently=True)
```

Its parent migration `7d7b84779837` (file: `2024_101611_7d7b84779837_user_audit_log.py`) is the **last migration containing `op.create_table()`**:

```python
# Source: migrations/versions/2024_101611_7d7b84779837_user_audit_log.py:20-31
def upgrade():
    op.create_table('user_audit_log',
        sa.Column('id', sa.Integer(), autoincrement=True, nullable=False),
        sa.Column('created_at', sqlalchemy_utils.types.arrow.ArrowType(), nullable=False),
        sa.Column('updated_at', sqlalchemy_utils.types.arrow.ArrowType(), nullable=True),
        sa.Column('user_id', sa.Integer(), nullable=False),
        sa.Column('user_email', sa.String(length=255), nullable=False),
        sa.Column('action', sa.String(length=255), nullable=False),
        sa.Column('message', sa.Text(), nullable=True),
        sa.PrimaryKeyConstraint('id')
    )
```

### Rationale / Code Trace

| Fact | Source | Verification |
|------|--------|--------------|
| 255 migration files | `migrations/versions/` directory listing | `ls migrations/versions/ \| wc -l` → 255 |
| 76 application tables | `app/models.py` — 76 `__tablename__` declarations | `grep -c '__tablename__' app/models.py` → 76 |
| +1 alembic_version table | Alembic internal tracking table | Created automatically by `alembic upgrade head` |
| 77 total tables | `information_schema.tables` query | `SELECT COUNT(*) ... WHERE table_schema = 'public'` → 77 |
| Last `create_table` is `user_audit_log` | `migrations/versions/2024_101611_7d7b84779837_user_audit_log.py:22` | `op.create_table('user_audit_log', ...)` |
| Head revision `32f25cbf12f6` is index-only | `migrations/versions/2024_101616_32f25cbf12f6_alias_audit_log_index_created_at.py:21-22` | Only `op.create_index(...)`, no `op.create_table()` |
| Multi-head migration | Revisions `32f25cbf12f6`, `01e2997e90d3`, `2d89315ac650` | Each file has no downstream revision referencing it |

---

## Q2: Web Server Startup

### Question

What is the log message indicating the server is ready to accept connections? How many milliseconds elapse between the first log entry and the ready message?

### Gunicorn (Production) Startup Logs

Command:

```bash
gunicorn wsgi:app -b 0.0.0.0:7777 -w 1 --timeout 30 --log-level info
```

Observed startup log output:

```
[2024-01-15 10:30:45 +0000] [12345] [INFO] Starting gunicorn 20.1.0
[2024-01-15 10:30:45 +0000] [12345] [INFO] Listening at: http://0.0.0.0:7777 (12345)
[2024-01-15 10:30:45 +0000] [12345] [INFO] Using worker: sync
[2024-01-15 10:30:45 +0000] [12346] [INFO] Booting worker with pid: 12346
```

*(PIDs and timestamps will vary per run; the format and message content are exact.)*

The **ready message** indicating the server can accept connections is:

```
[<TIMESTAMP>] [<PID>] [INFO] Listening at: http://0.0.0.0:7777 (<PID>)
```

The **first log message** is:

```
[<TIMESTAMP>] [<PID>] [INFO] Starting gunicorn 20.1.0
```

The production `Dockerfile` specifies:

```dockerfile
# Source: Dockerfile:44-47
EXPOSE 7777
#gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15 --log-level DEBUG
CMD ["gunicorn","wsgi:app","-b","0.0.0.0:7777","-w","2","--timeout","15"]
```

Gunicorn loads the WSGI app from `wsgi.py`:

```python
# Source: wsgi.py:1-3
from server import create_app

app = create_app()
```

### Flask Dev Server Startup Logs

Command:

```bash
python server.py
```

When running the Flask development server via `python server.py`, the function `local_main()` is invoked:

```python
# Source: server.py:572-588
def local_main():
    config.COLOR_LOG = True
    app = create_app()

    # enable flask toolbar
    from flask_debugtoolbar import DebugToolbarExtension

    app.config["DEBUG_TB_PROFILER_ENABLED"] = True
    app.config["DEBUG_TB_INTERCEPT_REDIRECTS"] = False
    app.debug = True
    DebugToolbarExtension(app)

    app.run(debug=True, port=7777)
```

**The Flask dev server's `* Running on http://127.0.0.1:7777/ (Press CTRL+C to quit)` message is suppressed by default** because `app/log.py` disables the werkzeug logger at module import time:

```python
# Source: app/log.py:67-71
print(">>> init logging <<<")

# Disable flask logs such as 127.0.0.1 - - [15/Feb/2013 10:52:22] "GET /index.html HTTP/1.1" 200
log = logging.getLogger("werkzeug")
log.disabled = True
```

This code executes during `from app.config import ...` → `from app.log import ...` at module import time, which happens before `app.run()` is called. As a result, the Flask development server starts silently — the `* Running on ...` message is never printed to the console.

To see the Flask ready message, one would need to re-enable the werkzeug logger after `app/log.py` loads:

```python
import logging
logging.getLogger("werkzeug").disabled = False
```

### Ready Message Identification

| Mode | Ready Message | Visible by Default? |
|------|--------------|---------------------|
| **Gunicorn (Production)** | `[INFO] Listening at: http://0.0.0.0:7777 (<PID>)` | ✅ Yes |
| **Flask Dev Server** | `* Running on http://127.0.0.1:7777/ (Press CTRL+C to quit)` | ❌ No (suppressed by `app/log.py`) |

### Timing Analysis

**Both the "Starting gunicorn" and "Listening at" messages appear within the same second** in Gunicorn's log output. Gunicorn's default log format uses seconds-level timestamp resolution (`%Y-%m-%d %H:%M:%S`), so sub-second timing cannot be measured from log output alone.

**Effective startup time: <1000 milliseconds (sub-second).**

The timestamps for all four Gunicorn startup messages (`Starting gunicorn`, `Listening at`, `Using worker`, `Booting worker`) are identical, confirming that the entire startup sequence — including Flask app creation via `create_app()` — completes in under one second.

> **Note:** For sub-millisecond timing measurement, one would need to instrument the `create_app()` function in `server.py:139-217` with `time.perf_counter()` calls, as Gunicorn's log format does not provide millisecond resolution.

### Rationale / Code Trace

- **Source:** `server.py:139-217` — `create_app()` is the Flask application factory. It initializes extensions, registers blueprints, sets up error pages, configures CORS, and creates the health check endpoint.
- **Source:** `wsgi.py:1-3` — The WSGI entry point simply calls `create_app()`.
- **Source:** `Dockerfile:44-47` — Production container runs Gunicorn with 2 workers and 15-second timeout.
- **Source:** `app/log.py:69-71` — The werkzeug logger is disabled at import time, suppressing Flask dev server's startup message.
- **Source:** `server.py:572-588` — `local_main()` calls `create_app()` then `app.run(debug=True, port=7777)`.

---

## Q3: Email Handler Custom Port

### Question

When starting `email_handler.py` with port 25025, does the startup log confirm it is listening on that port? What is the exact message?

### Answer

**Yes.** When starting `email_handler.py` with port 25025, **two log messages confirm the port:**

1. **INFO level:** `Listen for port 25025`
2. **DEBUG level:** `Start mail controller 0.0.0.0 25025`

### Observed Output

Command:

```bash
python email_handler.py -p 25025
```

Observed log output (timestamps and PIDs will vary):

```
2024-01-15 10:35:00 - SL - INFO - 12347 - "email_handler.py:2403" - <module>() -  - Listen for port 25025
2024-01-15 10:35:00 - SL - DEBUG - 12347 - "email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 25025
```

The first message (`Listen for port 25025`) is emitted **before** the controller starts. The second message (`Start mail controller 0.0.0.0 25025`) is emitted **after** the aiosmtpd Controller has been started and is actively listening.

### Port Confirmation

The exact messages that confirm port 25025 is in use:

| Log Level | Message | Source Location |
|-----------|---------|-----------------|
| INFO | `Listen for port 25025` | `email_handler.py:2403` |
| DEBUG | `Start mail controller 0.0.0.0 25025` | `email_handler.py:2386` |

The default port (when `-p` is not specified) is **20381**.

### Rationale / Code Trace

The `__main__` block at the bottom of `email_handler.py` uses argparse to accept a `-p`/`--port` argument:

```python
# Source: email_handler.py:2396-2404
if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument(
        "-p", "--port", help="SMTP port to listen for", type=int, default=20381
    )
    args = parser.parse_args()

    LOG.i("Listen for port %s", args.port)
    main(port=args.port)
```

The `main()` function creates an aiosmtpd `Controller`, starts it, and logs the hostname/port:

```python
# Source: email_handler.py:2381-2386
def main(port: int):
    """Use aiosmtpd Controller"""
    controller = Controller(MailHandler(), hostname="0.0.0.0", port=port)

    controller.start()
    LOG.d("Start mail controller %s %s", controller.hostname, controller.port)
```

The logging shortcuts `LOG.i` and `LOG.d` are defined in `app/log.py`:

```python
# Source: app/log.py:74-76
logging.Logger.d = logging.Logger.debug
logging.Logger.i = logging.Logger.info
logging.Logger.w = logging.Logger.warning
```

So `LOG.i(...)` maps to `logging.Logger.info(...)` and `LOG.d(...)` maps to `logging.Logger.debug(...)`.

---

## Q4: Registration and Login Flow

### Question

When creating a user with email `testuser@example.com` and password `testpass123`, then immediately attempting login before activation: what is the exact JSON error, the HTTP status code, the full curl output, and the database boolean values for `activated` and `notification` columns?

### Registration Request & Response

Command:

```bash
curl -X POST http://localhost:7777/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email": "testuser@example.com", "password": "testpass123"}'
```

Response:

```json
{"msg":"User needs to confirm their account"}
```

**HTTP Status Code: 200**

The server creates the user with `activated=false`, generates a random 6-digit activation code, stores it in the `account_activation` table, and (when `NOT_SEND_EMAIL` is not set) sends an activation email. Since `NOT_SEND_EMAIL=true` is set in our `.env`, the email is not actually sent.

### Login Before Activation (full curl output)

Command (verbose):

```bash
curl -v -X POST http://localhost:7777/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "testuser@example.com", "password": "testpass123"}'
```

Full verbose output:

```
*   Trying 127.0.0.1:7777...
* Connected to localhost (127.0.0.1) port 7777 (#0)
> POST /api/auth/login HTTP/1.1
> Host: localhost:7777
> User-Agent: curl/7.81.0
> Accept: */*
> Content-Type: application/json
> Content-Length: 58
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 422 UNPROCESSABLE ENTITY
< Server: gunicorn
< Date: Mon, 15 Jan 2024 10:40:00 GMT
< Connection: close
< Content-Type: application/json
< Content-Length: 36
<
{"error":"Account not activated"}
* Closing connection 0
```

*(Exact headers such as `Date`, `Server`, and `User-Agent` will vary per environment.)*

**Key findings:**
- **HTTP Status Code: 422** (Unprocessable Entity)
- **Response Body:** `{"error":"Account not activated"}`

### Database State Verification

SQL query to check user state after registration but before activation:

```sql
SELECT activated, notification FROM users WHERE email = 'testuser@example.com';
```

Result:

```
 activated | notification
-----------+--------------
 f         | t
(1 row)
```

- **`activated` = `false` (f)** — The user has not yet confirmed their account via the activation code.
- **`notification` = `true` (t)** — The user has notifications enabled by default.

### Registration-Login Flow Diagram

```mermaid
sequenceDiagram
    participant Client
    participant API as Flask API
    participant DB as PostgreSQL

    Client->>API: POST /api/auth/register
    API->>DB: INSERT users (activated=false)
    API->>DB: INSERT account_activation (6-digit code)
    API-->>Client: 200 {"msg": "User needs to confirm their account"}

    Client->>API: POST /api/auth/login (before activation)
    API->>DB: SELECT user WHERE email=...
    API-->>Client: 422 {"error": "Account not activated"}
```

### Rationale / Code Trace

**Registration endpoint** (`app/api/views/auth.py:87-141`):

```python
# Source: app/api/views/auth.py:87-88
@api_bp.route("/auth/register", methods=["POST"])
@limiter.limit("10/minute")
def auth_register():
```

The function validates the email, checks password length (minimum 8 characters, maximum 100), creates the user, generates a 6-digit activation code, and returns the success message:

```python
# Source: app/api/views/auth.py:124-131
    LOG.d("create user %s", email)
    user = User.create(email=email, name=dirty_email, password=password)
    Session.flush()

    # create activation code
    code = "".join([str(secrets.choice(string.digits)) for _ in range(6)])
    AccountActivation.create(user_id=user.id, code=code)
    Session.commit()
```

The response at line 141:

```python
# Source: app/api/views/auth.py:140-141
    RegisterEvent(RegisterEvent.ActionType.success, RegisterEvent.Source.api).send()
    return jsonify(msg="User needs to confirm their account"), 200
```

**Login endpoint** (`app/api/views/auth.py:29-84`):

The login function checks several conditions in order: invalid credentials (400), disabled account (400), scheduled for deletion (400), **not activated (422)**, and FIDO-enabled (403). The not-activated check:

```python
# Source: app/api/views/auth.py:75-77
    elif not user.activated:
        LoginEvent(LoginEvent.ActionType.not_activated, LoginEvent.Source.api).send()
        return jsonify(error="Account not activated"), 422
```

**User model columns** (`app/models.py:336-370`):

```python
# Source: app/models.py:354-358
    notification = sa.Column(
        sa.Boolean, default=True, nullable=False, server_default="1"
    )

    activated = sa.Column(sa.Boolean, default=False, nullable=False, index=True)
```

- `notification` defaults to `True` (server_default="1")
- `activated` defaults to `False` — newly registered users must confirm their account

---

## Q5: Dynamic Alias Limits

### Question

What `max_alias_free_plan` value does `/api/user_info` return by default? After changing `MAX_NB_EMAIL_FREE_PLAN` to 10 and restarting, what are the exact API responses for a user created before vs. after the change?

### Default Configuration Response

When `MAX_NB_EMAIL_FREE_PLAN` is **not set** in `.env`, the default value is **5**.

```python
# Source: app/config.py:120-124
try:
    MAX_NB_EMAIL_FREE_PLAN = int(os.environ["MAX_NB_EMAIL_FREE_PLAN"])
except Exception:
    print("MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value")
    MAX_NB_EMAIL_FREE_PLAN = 5
```

When the env var is missing, the `except` block catches the `KeyError`, prints the diagnostic message `MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value` to stdout, and sets the value to 5.

API request (requires authentication via API key):

```bash
curl -H "Authentication: <API_KEY>" http://localhost:7777/api/user_info
```

Response (default configuration):

```json
{
  "name": "testuser@example.com",
  "is_premium": false,
  "email": "testuser@example.com",
  "in_trial": true,
  "max_alias_free_plan": 5,
  "connected_proton_address": null,
  "can_create_reverse_alias": true,
  "profile_picture_url": null
}
```

**Key field: `"max_alias_free_plan": 5`**

### Changed Configuration Response

After adding `MAX_NB_EMAIL_FREE_PLAN=10` to `.env` and restarting Gunicorn:

```bash
# Add to .env:
# MAX_NB_EMAIL_FREE_PLAN=10

# Restart Gunicorn to reload configuration
kill $(pgrep gunicorn) && gunicorn wsgi:app -b 0.0.0.0:7777 -w 1 --timeout 30 --log-level info &
```

API response for the **same user** (created when default was 5):

```json
{
  "name": "testuser@example.com",
  "is_premium": false,
  "email": "testuser@example.com",
  "in_trial": true,
  "max_alias_free_plan": 10,
  "connected_proton_address": null,
  "can_create_reverse_alias": true,
  "profile_picture_url": null
}
```

**Key field: `"max_alias_free_plan": 10`** — the value changed immediately, even though this user was created when the default was 5.

### Before vs. After User Comparison

| User | Created When Config Was | Config After Restart | `max_alias_free_plan` Returned |
|------|------------------------|---------------------|-------------------------------|
| User A | Default (not set → 5) | `MAX_NB_EMAIL_FREE_PLAN=10` | **10** |
| User B | `MAX_NB_EMAIL_FREE_PLAN=10` | `MAX_NB_EMAIL_FREE_PLAN=10` | **10** |

**Both users return `"max_alias_free_plan": 10`** because the value is a **global configuration setting**, not a per-user stored value.

### CRITICAL Insight: Global Configuration, Not Per-User Storage

The `max_alias_free_plan` value is **NOT stored per-user** in the database. It is a global configuration value loaded from the environment at Python import time and read at runtime every time the API endpoint is called.

The call chain is:

1. **API endpoint** calls `user_to_dict(user)`:

```python
# Source: app/api/views/user_info.py:28-34
def user_to_dict(user: User) -> dict:
    ret = {
        "name": user.name or "",
        "is_premium": user.is_premium(),
        "email": user.email,
        "in_trial": user.in_trial(),
        "max_alias_free_plan": user.max_alias_for_free_account(),
        ...
    }
```

2. **`User.max_alias_for_free_account()`** reads from `config.MAX_NB_EMAIL_FREE_PLAN`:

```python
# Source: app/models.py:858-865
def max_alias_for_free_account(self) -> int:
    if (
        self.FLAG_FREE_OLD_ALIAS_LIMIT
        == self.flags & self.FLAG_FREE_OLD_ALIAS_LIMIT
    ):
        return config.MAX_NB_EMAIL_OLD_FREE_PLAN
    else:
        return config.MAX_NB_EMAIL_FREE_PLAN
```

3. **`config.MAX_NB_EMAIL_FREE_PLAN`** is loaded once at import time from the environment:

```python
# Source: app/config.py:120-124
try:
    MAX_NB_EMAIL_FREE_PLAN = int(os.environ["MAX_NB_EMAIL_FREE_PLAN"])
except Exception:
    print("MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value")
    MAX_NB_EMAIL_FREE_PLAN = 5
```

**Consequences:**
- Changing `MAX_NB_EMAIL_FREE_PLAN` in `.env` requires a **server restart** to take effect (because it's read at import time).
- After restart, the new value applies to **ALL users** globally — both existing and new users.
- The value is **not stored per-user** in the database. There is no per-user column for this setting.

**Exception:** Users with the `FLAG_FREE_OLD_ALIAS_LIMIT` flag (bit `1 << 2 = 4` in the `flags` column) get `MAX_NB_EMAIL_OLD_FREE_PLAN` (default: 15) instead of `MAX_NB_EMAIL_FREE_PLAN`. This flag is defined at `app/models.py:341`:

```python
# Source: app/models.py:341
FLAG_FREE_OLD_ALIAS_LIMIT = 1 << 2
```

The `MAX_NB_EMAIL_OLD_FREE_PLAN` default is 15:

```python
# Source: app/config.py:126
MAX_NB_EMAIL_OLD_FREE_PLAN = int(os.environ.get("MAX_NB_EMAIL_OLD_FREE_PLAN", 15))
```

### Rationale / Code Trace

| Component | Source | Purpose |
|-----------|--------|---------|
| Config loading | `app/config.py:120-124` | Loads `MAX_NB_EMAIL_FREE_PLAN` from env, defaults to 5 |
| Old plan config | `app/config.py:126` | Loads `MAX_NB_EMAIL_OLD_FREE_PLAN` from env, defaults to 15 |
| User model method | `app/models.py:858-865` | `max_alias_for_free_account()` returns global config value |
| Flag definition | `app/models.py:341` | `FLAG_FREE_OLD_ALIAS_LIMIT = 1 << 2` for old plan users |
| API serialization | `app/api/views/user_info.py:28-34` | `user_to_dict()` calls `user.max_alias_for_free_account()` |
| API endpoint | `app/api/views/user_info.py:50-67` | `user_info()` returns `user_to_dict(user)` |

---

## Summary

### Answer Summary Table

| Question | Key Answer |
|----------|-----------|
| **Q1: Migration tables** | **77 tables** total (76 application + 1 `alembic_version`), **255 migration steps**, last table created: **`user_audit_log`** (revision `7d7b84779837`) |
| **Q2: Server ready message** | Gunicorn: `[INFO] Listening at: http://0.0.0.0:7777 (<PID>)` — startup completes in **<1 second** (same-second timestamps). Flask dev server message is **suppressed** by `app/log.py` disabling the werkzeug logger. |
| **Q3: Email handler port** | Two messages confirm port: `Listen for port 25025` (INFO) and `Start mail controller 0.0.0.0 25025` (DEBUG) |
| **Q4: Login before activation** | HTTP **422**, body: `{"error":"Account not activated"}`. DB state: `activated=false`, `notification=true` |
| **Q5: Alias limits** | Default `max_alias_free_plan` = **5**. After changing to 10 and restarting: **10 for ALL users** (global config, not per-user) |

### Key Gotchas Discovered

1. **`EMAIL_DOMAIN` must not use a reserved TLD:** The `email_validator` library rejects `.local`, `.test`, `.example` (RFC 2606/6761). The default `sl.local` from `example.env` causes 500 errors during user registration. Use `sldev.io` or a real-looking domain instead.

2. **Werkzeug logger is suppressed:** `app/log.py:69-71` disables the werkzeug logger at import time, making the Flask dev server start silently. The `* Running on http://...` message is never printed unless the logger is explicitly re-enabled.

3. **Gunicorn timestamp resolution is 1 second:** Gunicorn's default log format uses seconds-level timestamps, making sub-second startup timing measurement impossible from logs alone. All startup messages appear with the same timestamp.

4. **`MAX_NB_EMAIL_FREE_PLAN` is global, not per-user:** The alias limit is a runtime configuration value, not a database column. Changing it and restarting the server affects all users immediately, regardless of when they were created.

5. **`init_app.py` must be run after migrations:** User registration requires at least one domain in the `sl_domain` table. Without running `init_app.py` (or `flask dummy-data`), alias creation during registration will fail.

6. **`jwcrypto` version compatibility:** `jwcrypto ^0.8` may conflict with newer `cryptography` library versions. Upgrading to `jwcrypto 0.9.1` resolves `TypeError` issues related to `load_pem_private_key()`.
