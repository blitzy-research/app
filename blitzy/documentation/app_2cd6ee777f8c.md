# SimpleLogin First-Run Initialization & Runtime Behavior Investigation

> **Investigation Date:** April 13, 2026
> **Branch:** `app_2cd6ee777f8c` (commit `2cd6ee77`)
> **Methodology:** All findings are derived from empirical runtime testing against a freshly provisioned environment — not from static code analysis alone.

---

## Table of Contents

1. [Database Migration Analysis](#1-database-migration-analysis)
2. [Web Server Startup Characterization](#2-web-server-startup-characterization)
3. [Email Handler Custom Port Verification](#3-email-handler-custom-port-verification)
4. [Registration and Pre-Activation Login Test](#4-registration-and-pre-activation-login-test)
5. [Dynamic Alias Limit Configuration Test](#5-dynamic-alias-limit-configuration-test)
6. [Key Findings Summary](#6-key-findings-summary)

---

## 1. Database Migration Analysis

### Question
How many tables are created when running all Alembic migrations against a fresh PostgreSQL database? What is the last table created by migration order?

### Environment Setup

A fresh PostgreSQL 16 database was provisioned, dropped, and recreated to ensure a completely empty state:

```sql
DROP DATABASE IF EXISTS simplelogin;
CREATE DATABASE simplelogin;
```

The `.env` file was configured with:

```text
DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin
```

### Migration Execution

All migrations were executed against the empty database using:

```bash
alembic upgrade head
```

**Observed Output (first 10 lines):**

```text
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade  -> 5e549314e1e2, empty message
INFO  [alembic.runtime.migration] Running upgrade 5e549314e1e2 -> 3cd10cfce8c3, empty message
INFO  [alembic.runtime.migration] Running upgrade 3cd10cfce8c3 -> 0256244cd7c8, empty message
INFO  [alembic.runtime.migration] Running upgrade 0256244cd7c8 -> 213fcca48483, empty message
INFO  [alembic.runtime.migration] Running upgrade 213fcca48483 -> f234688f5ebd, empty message
INFO  [alembic.runtime.migration] Running upgrade f234688f5ebd -> d03e433dc248, empty message
INFO  [alembic.runtime.migration] Running upgrade d03e433dc248 -> 2fe19381f386, empty message
INFO  [alembic.runtime.migration] Running upgrade 2fe19381f386 -> b20ee72fd9a4, empty message
```

**Observed Output (last 10 lines):**

```text
INFO  [alembic.runtime.migration] Running upgrade 88dd7a0abf54 -> 62afa3a10010, custom domain indices
INFO  [alembic.runtime.migration] Running upgrade 62afa3a10010 -> 91ed7f46dc81, alias_audit_log
INFO  [alembic.runtime.migration] Running upgrade 91ed7f46dc81 -> 7d7b84779837, user_audit_log
INFO  [alembic.runtime.migration] Running upgrade 7d7b84779837 -> 32f25cbf12f6, alias_audit_log_index_created_at
```

The `alembic_version` table records the final head revision:

```sql
SELECT version_num FROM alembic_version;
```

```text
 version_num
--------------
 32f25cbf12f6
```

### Table Count Verification

After all migrations completed, a direct query against the PostgreSQL information schema was executed:

```sql
SELECT count(*) AS total_tables
FROM information_schema.tables
WHERE table_schema = 'public';
```

```text
 total_tables
--------------
           77
```

This count includes the **`alembic_version`** metadata table. Excluding it:

```sql
SELECT count(*) AS model_tables
FROM information_schema.tables
WHERE table_schema = 'public'
  AND table_name != 'alembic_version';
```

```text
 model_tables
--------------
           76
```

Cross-referencing with `app/models.py`, there are exactly **76 `__tablename__` declarations**, confirming the final table count.

### Answer: Total Tables Created

**76 application tables** (plus 1 `alembic_version` table = 77 total) exist in the final database state after running all 255 Alembic migrations.

### Migration File Statistics

| Metric | Count |
|--------|-------|
| Migration revision files in `migrations/versions/` | **255** |
| Total `op.create_table()` calls in `upgrade()` functions | **80** |
| Unique table names created via `op.create_table()` | **79** |
| Tables dropped during migration history (in `upgrade()`) | 4 (`metric`, `partner`, `client_scope`, `scope`) |
| Tables renamed during migration history | 3 (`gen_email` → `alias`, `forward_email` → `contact`, `forward_email_log` → `email_log`) |

### Table Lifecycle Highlights

Several tables were created, dropped, or renamed across the migration history:

- **`gen_email`**: Created in the initial migration `5e549314e1e2` (2019-06-23), later renamed to **`alias`** in revision `e9395fe234a4` (2020-03-17)
- **`forward_email`**: Created in `5fa68bafae72`, later renamed to **`contact`** in revision `7744c5c16159` (2020-03-17)
- **`forward_email_log`**: Created in `6bbda4685999`, later renamed to **`email_log`** in revision `6e061eb84167` (2020-03-17)
- **`partner`**: Created in `b20ee72fd9a4` (2019-07-01), dropped in `2e2b53afd819` (2019-11-15), recreated in `e866ad0e78e1` (2022-05-05)
- **`metric`**: Created in `2779eb90c6c4` (2021-01-25), dropped in `20c738810b1b` (2021-07-28) — replaced by `metric2`
- **`client_scope`** and **`scope`**: Created in the initial migration, dropped in `551c4e6d4a8b`

### Answer: Last Table Created by Migration Order

**`user_audit_log`** — created in revision `7d7b84779837` (file `2024_101611_7d7b84779837_user_audit_log.py`, Create Date: 2024-10-16 11:52:49).

This is the chronologically last `op.create_table()` call in the entire migration history. The subsequent migration `32f25cbf12f6` (alias_audit_log_index_created_at) only creates an index, not a table.

**Migration file content (lines 22–31):**

```python
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

### Rationale

- The migration revision chain is linear with one merge point (`2634b41f54db` merging `01e2997e90d3` and `2d89315ac650`), and the single head is `32f25cbf12f6`.
- Table count was verified both empirically (SQL count against the running database) and statically (grep of `__tablename__` in `app/models.py`).
- The "last table created" was determined by sorting all `op.create_table()` calls in `upgrade()` functions by their migration file's `Create Date` field. The last 5 chronologically are: `sync_event` (2024-05-17), `mailbox_activation` (2024-07-30), `alias_audit_log` (2024-10-11), and **`user_audit_log` (2024-10-16)**.

---

## 2. Web Server Startup Characterization

### Question
What is the exact log message confirming Flask dev server readiness, and what is the elapsed time from the first log entry to the ready message?

### Test Setup

The Flask development server was started via `python server.py`, which calls `local_main()` (defined at `server.py` lines 572–588). The function:

1. Sets `config.COLOR_LOG = True` (line 573)
2. Creates the Flask app via `create_app()` (line 574)
3. Configures Flask Debug Toolbar (lines 577–582)
4. Starts the Werkzeug dev server with `app.run(debug=True, port=7777)` (line 588)

### Observed Startup Output

The following is the **actual captured output** from `python server.py` with precise wall-clock timestamps:

```text
[0.674s] >>> URL: http://localhost:7777
[1.638s] Paddle param not set
[1.638s] WARNING: Use a temp directory for GNUPGHOME /tmp/wfjqbfctywqakqgxmltf
[1.638s] Upload files to local dir
[1.638s] >>> init logging <<<
[1.638s] 2026-04-13 21:54:59,130 - SL - DEBUG - 36892 - "app/utils.py:17" - <module>() -  - load words file: local_data/test_words.txt
[1.638s]  * Serving Flask app "server" (lazy loading)
[2.309s]  * Environment: production
[2.309s]    WARNING: This is a development server. Do not use it in a production deployment.
[2.309s]    Use a production WSGI server instead.
[2.309s]  * Debug mode: on
```

### Critical Finding: The Werkzeug `* Running on` Message Is Suppressed

**The standard Werkzeug readiness message `* Running on http://127.0.0.1:7777/ (Press CTRL+C to quit)` does NOT appear in the output**, despite the server being fully operational (verified via `curl http://localhost:7777/health` returning `success`).

**Root Cause:** `app/log.py` lines 69–71 explicitly disable the Werkzeug logger:

```python
# Disable flask logs such as 127.0.0.1 - - [15/Feb/2013 10:52:22] "GET /index.html HTTP/1.1" 200
log = logging.getLogger("werkzeug")
log.disabled = True
```

Werkzeug's `_log()` function (in `werkzeug/serving.py`) sends all messages — including the `* Running on` startup message — through `logging.getLogger("werkzeug").info(...)`. When `log.disabled = True`, Python's logging framework suppresses all messages from this logger.

The messages that DO appear (`* Serving Flask app`, `* Environment: production`, `* Debug mode: on`) come from Flask's `show_server_banner()` function in `flask/cli.py`, which uses `click.echo()` — a direct stdout write that bypasses the logging system entirely.

### Answer: Dev Server Readiness Message

The **last visible startup message** confirming the dev server is configured and launching is:

```text
 * Debug mode: on
```

The standard Werkzeug `* Running on http://127.0.0.1:7777/ (Press CTRL+C to quit)` message is **suppressed** by `app/log.py` line 71 (`log.disabled = True`), even though the server is fully operational and listening.

### Gunicorn (Production) Startup Message

For comparison, when starting via Gunicorn (`gunicorn wsgi:app -b 0.0.0.0:7777 --workers 2`), the readiness message IS visible because Gunicorn uses its own logging system, not the disabled Werkzeug logger:

```text
[2026-04-13 21:52:54 +0000] [33016] [INFO] Starting gunicorn 20.0.4
[2026-04-13 21:52:54 +0000] [33016] [INFO] Listening at: http://0.0.0.0:7777 (33016)
[2026-04-13 21:52:54 +0000] [33016] [INFO] Using worker: sync
[2026-04-13 21:52:54 +0000] [33017] [INFO] Booting worker with pid: 33017
[2026-04-13 21:52:54 +0000] [33018] [INFO] Booting worker with pid: 33018
```

The Gunicorn readiness confirmation is:

```text
[INFO] Listening at: http://0.0.0.0:7777 (PID)
```

### Answer: Elapsed Time

| Event | Wall-Clock Time |
|-------|----------------|
| First output (`>>> URL: http://localhost:7777`) | 0.674s |
| `>>> init logging <<<` | 1.638s |
| `* Serving Flask app "server"` | 1.638s |
| `* Debug mode: on` (last visible message) | 2.309s |
| Server operational (health check passes) | ~3.5s |

The elapsed time from the first log entry (`>>> URL`) to the last visible startup message (`* Debug mode: on`) is approximately **1,635 milliseconds** (2.309s – 0.674s).

Note: In a Gunicorn production setup, the time from first log to `Listening at:` is nearly instantaneous (under 100ms) because Gunicorn logs its own readiness before the application modules finish loading in the worker processes.

### Log Format

The SL application log format is defined in `app/log.py` line 12–14:

```python
_log_format = (
    "%(asctime)s - %(name)s - %(levelname)s - %(process)d - "
    '"%(pathname)s:%(lineno)d" - %(funcName)s() - %(message_id)s - %(message)s'
)
```

Timestamps use UTC via `time.gmtime` (set at `app/log.py` line 43: `console_handler.formatter.converter = time.gmtime`).

### Rationale

- The `local_main()` function at `server.py` line 572 is the dev-mode entrypoint. It calls `create_app()` which wires up all Flask extensions, blueprints (auth, API, dashboard, developer, OAuth, phone, etc.), admin panel, CORS, rate limiting, and session management (lines 139–217).
- The Werkzeug logger is disabled at module import time when `app/log.py` is first loaded. This happens during `from app.log import LOG` in `server.py` line 83.
- Werkzeug 1.0.1 (the version installed) routes ALL log messages through `logging.getLogger("werkzeug")`. The `_log('info', ' * Running on...')` call in `werkzeug/serving.py` is suppressed because `logger.disabled = True` causes Python's `Logger.isEnabledFor()` to return `False`.

---

## 3. Email Handler Custom Port Verification

### Question
What is the exact startup log message when `email_handler.py` is launched with port 25025?

### Test Execution

The email handler was started with:

```bash
python email_handler.py -p 25025
```

### Observed Output

The following is the **complete captured output** from the email handler startup:

```text
>>> URL: http://localhost:7777
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/gpqntocufrvrkebuiwtm
Upload files to local dir
>>> init logging <<<
2026-04-13 21:53:17,402 - SL - DEBUG - 33682 - "email_handler.py:17" - <module>() -  - load words file: local_data/test_words.txt
2026-04-13 21:53:18,059 - SL - INFO - 33682 - "email_handler.py:2403" - <module>() -  - Listen for port 25025
2026-04-13 21:53:18,060 - SL - DEBUG - 33682 - "email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 25025
```

### Answer: Exact Startup Log Messages

There are **two** log messages confirming the email handler startup with port 25025:

**Message 1 (INFO level) — Port listen announcement:**

```text
2026-04-13 21:53:18,059 - SL - INFO - 33682 - "email_handler.py:2403" - <module>() -  - Listen for port 25025
```

**Message 2 (DEBUG level) — Controller start confirmation:**

```text
2026-04-13 21:53:18,060 - SL - DEBUG - 33682 - "email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 25025
```

### Code Path Analysis

The startup flow in `email_handler.py`:

1. **`__main__` block (lines 2396–2404):**
   ```python
   parser = argparse.ArgumentParser()
   parser.add_argument(
       "-p", "--port", help="SMTP port to listen for", type=int, default=20381
   )
   args = parser.parse_args()
   LOG.i("Listen for port %s", args.port)    # Line 2403 → INFO level
   main(port=args.port)                       # Line 2404
   ```

2. **`main(port)` function (lines 2381–2393):**
   ```python
   def main(port: int):
       controller = Controller(MailHandler(), hostname="0.0.0.0", port=port)
       controller.start()
       LOG.d("Start mail controller %s %s", controller.hostname, controller.port)  # Line 2386 → DEBUG level
       ...
       while True:
           time.sleep(2)
   ```

### Rationale

- The logger name is `"SL"` (from `app/log.py` line 79: `LOG = _get_logger("SL")`).
- `LOG.i` is a shortcut for `LOG.info` (defined at `app/log.py` line 75: `logging.Logger.i = logging.Logger.info`).
- `LOG.d` is a shortcut for `LOG.debug` (defined at `app/log.py` line 74: `logging.Logger.d = logging.Logger.debug`).
- The `message_id` field in the log format is empty (`""`) during startup because no email is being processed (the `EmailHandlerFilter` returns an empty string when `_MESSAGE_ID` is not set).
- The `aiosmtpd.controller.Controller` creates a separate thread running the SMTP server on `0.0.0.0:25025`. The `controller.start()` call is non-blocking.
- The default port when `-p` is not specified is `20381` (line 2399).

---

## 4. Registration and Pre-Activation Login Test

### Question
What happens when you register `testuser@example.com` with password `testpass123`, then immediately attempt to log in before activating the account? What is the exact JSON error response, HTTP status code, and full curl command output? What are the actual `activated` and `notification` column values in the database?

### Test Environment

The web server was running via Gunicorn (`gunicorn wsgi:app -b 0.0.0.0:7777 --workers 2`) against a freshly migrated PostgreSQL database with no existing users.

### Step 1: Registration

**Curl command:**

```bash
curl -v -X POST http://localhost:7777/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email": "testuser@example.com", "password": "testpass123"}'
```

**Full curl output:**

```http
> POST /api/auth/register HTTP/1.1
> Host: localhost:7777
> User-Agent: curl/8.5.0
> Accept: */*
> Content-Type: application/json
> Content-Length: 60
>
< HTTP/1.1 200 OK
< Server: gunicorn/20.0.4
< Date: Mon, 13 Apr 2026 21:53:07 GMT
< Connection: close
< Content-Type: application/json
< Content-Length: 46
< Access-Control-Allow-Origin: *
<
{"msg":"User needs to confirm their account"}
```

**HTTP Status Code:** `200 OK`

**JSON Response Body:**

```json
{"msg": "User needs to confirm their account"}
```

### Step 2: Login (Pre-Activation)

**Curl command:**

```bash
curl -v -X POST http://localhost:7777/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "testuser@example.com", "password": "testpass123"}'
```

**Full curl output:**

```http
> POST /api/auth/login HTTP/1.1
> Host: localhost:7777
> User-Agent: curl/8.5.0
> Accept: */*
> Content-Type: application/json
> Content-Length: 60
>
< HTTP/1.1 422 UNPROCESSABLE ENTITY
< Server: gunicorn/20.0.4
< Date: Mon, 13 Apr 2026 21:53:07 GMT
< Connection: close
< Content-Type: application/json
< Content-Length: 34
< Access-Control-Allow-Origin: *
<
{"error":"Account not activated"}
```

**HTTP Status Code:** `422 UNPROCESSABLE ENTITY`

**JSON Error Response:**

```json
{"error": "Account not activated"}
```

### Step 3: Database Column Verification

**SQL query:**

```sql
SELECT activated, notification
FROM users
WHERE email = 'testuser@example.com';
```

**Result:**

```text
 activated | notification
-----------+--------------
 f         | t
(1 row)
```

### Answer Summary

| Item | Value |
|------|-------|
| Registration HTTP status | **200 OK** |
| Registration response | `{"msg":"User needs to confirm their account"}` |
| Login HTTP status | **422 UNPROCESSABLE ENTITY** |
| Login error response | `{"error":"Account not activated"}` |
| `activated` column value | **`False`** (PostgreSQL: `f`) |
| `notification` column value | **`True`** (PostgreSQL: `t`) |

### Code Path Analysis

#### Registration Flow (`app/api/views/auth.py` lines 87–141)

1. **Line 103–104:** Email is extracted and canonicalized: `email = canonicalize_email(dirty_email)`
2. **Line 107–109:** `DISABLE_REGISTRATION` check — not set, so registration proceeds
3. **Line 110:** Email validation via `email_can_be_used_as_mailbox(email)` and `personal_email_already_used(email)` — `testuser@example.com` passes both
4. **Line 116–122:** Password length check (≥8 chars, ≤100 chars) — `testpass123` is 11 characters, passes
5. **Line 125:** `User.create(email=email, name=dirty_email, password=password)` — creates the User record

Inside `User.create()` (`app/models.py` lines 602–668):
- Line 604: Calls `super().create(email=email, name=name[:100])` which sets `activated=False` (the column default from line 358)
- Line 606–607: Hashes and stores the password via `user.set_password(password)` (bcrypt, from `app/pw_models.py`)
- Line 611: Creates a verified `Mailbox` for the user
- Lines 634–640: Creates first alias with prefix `"simplelogin-newsletter"`
- Line 646–648: `DISABLE_ONBOARDING` is set in our `.env`, so onboarding emails are skipped

Back in the registration endpoint:
- Lines 129–131: Generates a 6-digit activation code and creates an `AccountActivation` record
- Lines 133–138: Sends activation email — suppressed because `NOT_SEND_EMAIL=true` in `.env`
- Line 141: Returns `{"msg": "User needs to confirm their account"}` with HTTP 200

#### Login Rejection Flow (`app/api/views/auth.py` lines 29–84)

1. **Lines 48–50:** Parses JSON request body
2. **Lines 55–60:** Extracts and sanitizes email, computes canonical form
3. **Line 62:** Looks up user: `User.get_by(email=email)` — finds the user
4. **Line 64:** Verifies password via `user.check_password(password)` — bcrypt comparison succeeds
5. **Line 67–69:** Checks `user.disabled` — is `False`, passes
6. **Line 70–74:** Checks `user.delete_on` — is `None`, passes
7. **Line 75–77:** **Checks `not user.activated`** — `activated` is `False`, so this condition is `True`
8. **Line 77:** Returns `jsonify(error="Account not activated"), 422`

#### Column Default Analysis

- **`activated`** (`app/models.py` line 358): `sa.Column(sa.Boolean, default=False, nullable=False, index=True)` — Python ORM default is `False`. No code in the registration path sets `activated=True`. The user must explicitly activate via the `/api/auth/activate` endpoint.
- **`notification`** (`app/models.py` lines 354–356): `sa.Column(sa.Boolean, default=True, nullable=False, server_default="1")` — Python ORM default is `True`, PostgreSQL server default is `"1"` (True). The only case where `notification` is set to `False` during user creation is when `from_partner=True` (line 623), which is not the case for API registration.

---

## 5. Dynamic Alias Limit Configuration Test

### Question
How does changing `MAX_NB_EMAIL_FREE_PLAN` from 5 to 10 affect the `/api/user_info` endpoint's `max_alias_free_plan` response? Does it affect existing users, new users, or both?

### Configuration Loading Mechanism

`MAX_NB_EMAIL_FREE_PLAN` is loaded in `app/config.py` lines 120–124:

```python
try:
    MAX_NB_EMAIL_FREE_PLAN = int(os.environ["MAX_NB_EMAIL_FREE_PLAN"])
except Exception:
    print("MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value")
    MAX_NB_EMAIL_FREE_PLAN = 5
```

This is a **module-level variable** that is set once when `app/config.py` is first imported. The value is read from `os.environ`, which includes variables loaded from `.env` via `python-dotenv` (`app/config.py` lines 65–71). Changing the `.env` file requires a **server restart** for the new value to take effect, because the module is only imported once per process.

### Test Protocol

#### Step 1: Establish Baseline (MAX_NB_EMAIL_FREE_PLAN=5)

With `.env` containing `MAX_NB_EMAIL_FREE_PLAN=5`, the server was started and User A (`testuser@example.com`) was registered, activated, and logged in to obtain an API key.

**API call:**

```bash
curl -s http://localhost:7777/api/user_info \
  -H "Authentication: <API_KEY_A>"
```

**Response:**

```json
{
    "can_create_reverse_alias": true,
    "connected_proton_address": null,
    "email": "testuser@example.com",
    "in_trial": true,
    "is_premium": true,
    "max_alias_free_plan": 5,
    "name": "testuser@example.com",
    "profile_picture_url": null
}
```

**`max_alias_free_plan` = 5** ✅

#### Step 2: Change Configuration and Restart

1. Modified `.env`: changed `MAX_NB_EMAIL_FREE_PLAN=5` to `MAX_NB_EMAIL_FREE_PLAN=10`
2. Killed the Gunicorn process (`pkill -f gunicorn`)
3. Restarted: `gunicorn wsgi:app -b 0.0.0.0:7777 --workers 2`
4. Verified health: `curl -s http://localhost:7777/health` → `success`

#### Step 3: Create User B and Compare

User B (`testuser2@example.com`) was registered, activated, and logged in with the new configuration.

**User B's `/api/user_info` response:**

```json
{
    "can_create_reverse_alias": true,
    "connected_proton_address": null,
    "email": "testuser2@example.com",
    "in_trial": true,
    "is_premium": true,
    "max_alias_free_plan": 10,
    "name": "testuser2@example.com",
    "profile_picture_url": null
}
```

**`max_alias_free_plan` = 10** ✅ (as expected with new config)

#### Step 4: Re-check User A (Existing User)

**User A's `/api/user_info` response after config change:**

```json
{
    "can_create_reverse_alias": true,
    "connected_proton_address": null,
    "email": "testuser@example.com",
    "in_trial": true,
    "is_premium": true,
    "max_alias_free_plan": 10,
    "name": "testuser@example.com",
    "profile_picture_url": null
}
```

**`max_alias_free_plan` = 10** — **Also changed!**

### Answer: The Configuration Change Affects ALL Users

The change from `MAX_NB_EMAIL_FREE_PLAN=5` to `MAX_NB_EMAIL_FREE_PLAN=10` (with server restart) affects **ALL users**, not just newly created ones.

| Scenario | User A (existing) | User B (new) |
|----------|-------------------|--------------|
| Before change (config=5) | `max_alias_free_plan: 5` | N/A |
| After change (config=10) | `max_alias_free_plan: 10` | `max_alias_free_plan: 10` |

### Why This Happens

The `max_alias_free_plan` value is **NOT stored in the database per-user**. Instead, it is computed dynamically at request time.

**Call chain:**

1. `GET /api/user_info` → `app/api/views/user_info.py` line 67: `return jsonify(user_to_dict(user))`
2. `user_to_dict(user)` → line 34: `"max_alias_free_plan": user.max_alias_for_free_account()`
3. `User.max_alias_for_free_account()` → `app/models.py` lines 858–865:

```python
def max_alias_for_free_account(self) -> int:
    if (
        self.FLAG_FREE_OLD_ALIAS_LIMIT
        == self.flags & self.FLAG_FREE_OLD_ALIAS_LIMIT
    ):
        return config.MAX_NB_EMAIL_OLD_FREE_PLAN
    else:
        return config.MAX_NB_EMAIL_FREE_PLAN
```

This method reads `config.MAX_NB_EMAIL_FREE_PLAN` directly from the module-level variable — it does not read from the database or from any per-user setting. Since the config is reloaded when the server restarts, ALL users get the new value.

### Exception: The `FLAG_FREE_OLD_ALIAS_LIMIT` Flag

Users with the `FLAG_FREE_OLD_ALIAS_LIMIT` flag set (bit 2, value `1 << 2 = 4`, defined at `app/models.py` line 341) will always receive `config.MAX_NB_EMAIL_OLD_FREE_PLAN` (default: 15) regardless of `MAX_NB_EMAIL_FREE_PLAN`. This flag is designed for legacy users who were grandfathered into a higher free-tier alias limit.

### Key Configuration Behavior Summary

| Property | Value |
|----------|-------|
| Configuration variable | `MAX_NB_EMAIL_FREE_PLAN` |
| Default value (if not set) | `5` |
| Where defined | `app/config.py` lines 120–124 |
| Loading mechanism | Module-level `int(os.environ["MAX_NB_EMAIL_FREE_PLAN"])` at import time |
| Change requires | **Server restart** (not hot-reloadable; module-level variable) |
| Per-user storage | **No** — computed dynamically from global config at request time |
| Impact of change | **All users** — both existing and new |
| Override mechanism | `FLAG_FREE_OLD_ALIAS_LIMIT` flag → uses `MAX_NB_EMAIL_OLD_FREE_PLAN` (default 15) instead |

---

## 6. Key Findings Summary

### Finding 1: Database Migrations
Running all 255 Alembic migrations against a fresh PostgreSQL database results in **76 application tables** (77 including `alembic_version`). The last table created by migration execution order is **`user_audit_log`** from revision `7d7b84779837` (October 2024). Several tables were created, renamed, or dropped across the migration history, but the final state matches the 76 `__tablename__` declarations in `app/models.py`.

### Finding 2: Flask Dev Server Readiness
The standard Werkzeug `* Running on http://127.0.0.1:7777/` message is **suppressed** because `app/log.py` disables the werkzeug logger (`log.disabled = True`). The last visible startup message is `* Debug mode: on`. In production via Gunicorn, the readiness message `[INFO] Listening at: http://0.0.0.0:7777` IS visible. The elapsed time from first output to last visible startup message is approximately **1,635 ms**.

### Finding 3: Email Handler Port Binding
Running `python email_handler.py -p 25025` produces two log messages:
1. **INFO:** `Listen for port 25025` (from `__main__` at line 2403)
2. **DEBUG:** `Start mail controller 0.0.0.0 25025` (from `main()` at line 2386)

Both use the `SL` logger with the standard log format. The `message_id` field is empty during startup.

### Finding 4: Pre-Activation Login Rejection
Registering `testuser@example.com` returns HTTP 200 with `{"msg":"User needs to confirm their account"}`. Immediately logging in returns HTTP **422 UNPROCESSABLE ENTITY** with `{"error":"Account not activated"}`. The database shows `activated=False` and `notification=True` for the newly created user.

### Finding 5: Dynamic Alias Limits
Changing `MAX_NB_EMAIL_FREE_PLAN` from 5 to 10 (with server restart) affects **ALL users** — not just newly created ones. This is because `User.max_alias_for_free_account()` reads the global config variable at request time rather than a per-user stored value. The only exception is users with the `FLAG_FREE_OLD_ALIAS_LIMIT` flag, who always get `MAX_NB_EMAIL_OLD_FREE_PLAN` (default 15).
