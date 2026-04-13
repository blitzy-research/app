# SimpleLogin First-Run Initialization & Runtime Behavior Analysis

## Table of Contents

1. [Database Migration Analysis](#1-database-migration-analysis)
2. [Web Server Startup Characterization](#2-web-server-startup-characterization)
3. [Email Handler Custom Port Verification](#3-email-handler-custom-port-verification)
4. [Registration and Pre-Activation Login Test](#4-registration-and-pre-activation-login-test)
5. [Dynamic Alias Limit Configuration Test](#5-dynamic-alias-limit-configuration-test)

---

## 1. Database Migration Analysis

### Question
Execute Alembic migrations against an empty PostgreSQL database. How many tables are created, and what is the exact name of the last table created based on migration output order?

### Environment
- PostgreSQL 16 (running locally)
- Fresh empty database `simplelogin` with user `myuser`
- Alembic head revision: `32f25cbf12f6`

### Procedure
Executed `alembic upgrade head` against a completely empty PostgreSQL database with no prior schema.

### Migration Output (key lines)

```
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade  -> 5e549314e1e2, empty message
INFO  [alembic.runtime.migration] Running upgrade 5e549314e1e2 -> 3cd10cfce8c3, empty message
... (253 intermediate migration steps) ...
INFO  [alembic.runtime.migration] Running upgrade 91ed7f46dc81 -> 7d7b84779837, user_audit_log
INFO  [alembic.runtime.migration] Running upgrade 7d7b84779837 -> 32f25cbf12f6, alias_audit_log_index_created_at
```

### Results

**Total number of migration revisions executed:** 255

**Total number of tables created:** 77 (76 application tables + 1 `alembic_version` table)

This was verified by querying the database directly:

```sql
SELECT count(*) FROM information_schema.tables
WHERE table_schema='public' AND table_type='BASE TABLE';
```
```
 count
-------
    77
(1 row)
```

**The last table created (based on migration execution order):** `user_audit_log`

The migration chain ends with three revisions:
1. `91ed7f46dc81` — Creates the `alias_audit_log` table (2024-10-11)
2. `7d7b84779837` — Creates the `user_audit_log` table (2024-10-16) ← **Last table created**
3. `32f25cbf12f6` — Only creates an index on `alias_audit_log.created_at` (no new table)

The `user_audit_log` table is created by revision `7d7b84779837` with this schema:

```python
op.create_table('user_audit_log',
    sa.Column('id', sa.Integer(), autoincrement=True, nullable=False),
    sa.Column('created_at', ArrowType(), nullable=False),
    sa.Column('updated_at', ArrowType(), nullable=True),
    sa.Column('user_id', sa.Integer(), nullable=False),
    sa.Column('user_email', sa.String(length=255), nullable=False),
    sa.Column('action', sa.String(length=255), nullable=False),
    sa.Column('message', sa.Text(), nullable=True),
    sa.PrimaryKeyConstraint('id')
)
```

### Rationale
The 77 tables include `alembic_version` (automatically created by Alembic to track migration state) plus the 76 application tables declared with `__tablename__` in `app/models.py`. While the head revision (`32f25cbf12f6`) is the final migration to run, it only adds an index—not a table. The actual last `op.create_table()` call in the migration chain is in revision `7d7b84779837`, which creates `user_audit_log`.

### Complete Table List

```
account_activation, activation_code, admin_audit_log, alembic_version, alias,
alias_audit_log, alias_hibp, alias_mailbox, alias_used_on, api_cookie_token,
api_key, apple_subscription, authorization_code, authorized_address,
auto_create_rule, auto_create_rule__mailbox, batch_import, bounce, client,
client_user, coinbase_subscription, contact, coupon, custom_domain,
daily_metric, deleted_alias, deleted_directory, deleted_subdomain, directory,
directory_mailbox, domain_deleted_alias, domain_mailbox, email_change,
email_log, fido, file, hibp, hibp_notified_alias, ignore_bounce_sender,
ignored_email, invalid_mailbox_domain, job, lifetime_coupon, mailbox,
mailbox_activation, manual_subscription, message_id_matching, metric2,
mfa_browser, monitoring, newsletter, newsletter_user, notification,
oauth_token, partner, partner_api_token, partner_subscription, partner_user,
payout, phone_country, phone_message, phone_number, phone_reservation,
provider_complaint, public_domain, recovery_code, redirect_uri, referral,
refused_email, reset_password_code, sent_alert, social_auth, subscription,
sync_event, transactional_email, user_audit_log, users
```

---

## 2. Web Server Startup Characterization

### Question
Start the Flask/Gunicorn web server. What is the exact log message confirming readiness, and what is the elapsed time in milliseconds from the first log entry to that ready message?

### Procedure — Flask Development Server (`python server.py`)

Started the development server via `python server.py`, which invokes `local_main()` at `server.py:572`. This calls `create_app()`, configures the Flask debug toolbar, and runs `app.run(debug=True, port=7777)`.

### Flask Dev Server Startup Output

```
>>> URL: http://localhost:7777
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/hniszfecbjnovivdebnz
Upload files to local dir
>>> init logging <<<
2026-04-13 21:39:56,523 - SL - DEBUG - 20197 - "...app/utils.py:17" - <module>() -  - load words file: .../local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
>>> URL: http://localhost:7777
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/zrhccaxcaipkhzljsgif
Upload files to local dir
>>> init logging <<<
2026-04-13 21:39:58,186 - SL - DEBUG - 20204 - "...app/utils.py:17" - <module>() -  - load words file: .../local_data/test_words.txt
```

### Critical Finding: The Werkzeug `Running on` Message Is Suppressed

**The standard Werkzeug readiness message (`* Running on http://127.0.0.1:7777/ (Press CTRL+C to quit)`) is intentionally suppressed by the application.**

In `app/log.py` (lines 73–74), the codebase explicitly disables the Werkzeug logger:

```python
# Disable flask logs such as 127.0.0.1 - - [15/Feb/2013 10:52:22] "GET /index.html HTTP/1.1" 200
log = logging.getLogger("werkzeug")
log.disabled = True
```

Werkzeug's `run_simple()` function calls `_log("info", " * Running on %s://%s:%d/ ...")` internally, which routes through `logging.getLogger("werkzeug")`. Since that logger is disabled, the message is silently dropped.

As a result, the **effective readiness indicator** for the Flask development server is the **last `Debug mode: on` output** printed by Flask CLI via `click.echo()` (which bypasses the Python logging system and writes directly to stdout). After this line, the Werkzeug reloader spawns a child process that repeats the initialization, and then the server begins accepting connections.

### Readiness Confirmation Message

For the **Flask dev server** (`python server.py`):
```
 * Debug mode: on
```
This is the last printed message before the server starts listening. There is no explicit "ready" log line because the Werkzeug logger is disabled.

For **Gunicorn** (`gunicorn wsgi:app -b 0.0.0.0:7777`):
```
[2026-04-13 21:40:28 +0000] [20608] [INFO] Listening at: http://0.0.0.0:7777 (20608)
```
Gunicorn uses its own logging (not the disabled Werkzeug logger), so the readiness message is clearly visible.

### Procedure — Gunicorn Server

```
[2026-04-13 21:40:28 +0000] [20608] [INFO] Starting gunicorn 20.0.4
[2026-04-13 21:40:28 +0000] [20608] [INFO] Listening at: http://0.0.0.0:7777 (20608)
[2026-04-13 21:40:28 +0000] [20608] [INFO] Using worker: sync
[2026-04-13 21:40:28 +0000] [20609] [INFO] Booting worker with pid: 20609
[2026-04-13 21:40:28 +0000] [20610] [INFO] Booting worker with pid: 20610
>>> URL: http://localhost:7777
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME ...
Upload files to local dir
>>> init logging <<<
2026-04-13 21:40:29,033 - SL - DEBUG - 20609 - "...app/utils.py:17" - <module>() -  - load words file: .../local_data/test_words.txt
```

### Elapsed Time Analysis

**Flask dev server:**
- First log entry with timestamp: `2026-04-13 21:39:56,523` (the `load words file` DEBUG message from the parent process)
- The `* Serving Flask app` and `* Debug mode: on` messages are printed via `click.echo()` and have no timestamps
- Second log entry from the reloader child process: `2026-04-13 21:39:58,186`
- Elapsed time from first timestamped log to second (reloader child ready): **1,663 milliseconds**
- The server is accepting connections after the child process starts, which occurs after the second `load words file` entry

**Gunicorn:**
- First log entry: `[2026-04-13 21:40:28 +0000]` — `Starting gunicorn 20.0.4`
- Readiness message: `[2026-04-13 21:40:28 +0000]` — `Listening at: http://0.0.0.0:7777`
- Worker ready: `2026-04-13 21:40:29,033` — worker completes app loading
- Elapsed from start to Listening: **< 1 second** (same second)
- Elapsed from start to first worker completing app load: **~1,033 milliseconds**

### Rationale
The Flask development server suppresses the standard Werkzeug readiness message because `app/log.py` disables the `werkzeug` logger. This is a deliberate design choice to reduce log noise in development. The closest readiness indicator is `* Debug mode: on` for the dev server, or `Listening at: http://0.0.0.0:7777` for Gunicorn. The health endpoint (`GET /health`) returns `"success"` with HTTP 200 once the server is accepting connections, which was confirmed via `curl`.

---

## 3. Email Handler Custom Port Verification

### Question
Launch the `email_handler.py` SMTP service with port `25025` and capture the exact startup log message confirming the listener is bound to that port.

### Procedure
Executed `python email_handler.py -p 25025` and captured stdout.

### Exact Startup Log Output

```
>>> URL: http://localhost:7777
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/qhkyltuplbtjlyprpcvs
Upload files to local dir
>>> init logging <<<
2026-04-13 21:40:51,712 - SL - DEBUG - 21147 - "/tmp/blitzy/app/.../app/utils.py:17" - <module>() -  - load words file: .../local_data/test_words.txt
2026-04-13 21:40:52,379 - SL - INFO - 21147 - "/tmp/blitzy/app/.../email_handler.py:2403" - <module>() -  - Listen for port 25025
2026-04-13 21:40:52,381 - SL - DEBUG - 21147 - "/tmp/blitzy/app/.../email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 25025
```

### Key Log Messages

There are **two** relevant startup messages, in this order:

1. **`Listen for port 25025`** — Logged at INFO level from `email_handler.py:2403` in `<module>()` scope, immediately before calling `main(port=args.port)`. This is generated by:
   ```python
   LOG.i("Listen for port %s", args.port)
   ```

2. **`Start mail controller 0.0.0.0 25025`** — Logged at DEBUG level from `email_handler.py:2386` in `main()`, immediately after `controller.start()` returns. This is generated by:
   ```python
   controller = Controller(MailHandler(), hostname="0.0.0.0", port=port)
   controller.start()
   LOG.d("Start mail controller %s %s", controller.hostname, controller.port)
   ```

### Rationale
The `email_handler.py` `__main__` block (lines 2396–2404) first parses the `-p`/`--port` argument (defaulting to 20381), logs the intent with `LOG.i("Listen for port %s", args.port)`, then calls `main(port=args.port)`. Inside `main()` (lines 2381–2393), the `aiosmtpd.controller.Controller` is created on `hostname="0.0.0.0"` and the specified port, then `controller.start()` spawns the SMTP listener thread. After successful start, `LOG.d("Start mail controller %s %s", controller.hostname, controller.port)` confirms the binding. The handler then enters an infinite `while True: time.sleep(2)` loop.

---

## 4. Registration and Pre-Activation Login Test

### Question
Create a user account with email `testuser@example.com` and password `testpass123`, then immediately attempt login before activating the account. Capture the exact JSON error response, the HTTP status code, and the full curl command output. Also query the database directly for the actual boolean values of the `activated` and `notification` columns.

### Step 1: Registration

**Curl Command:**
```bash
curl -s -w "\nHTTP_STATUS_CODE: %{http_code}\n" -X POST http://localhost:7777/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email": "testuser@example.com", "password": "testpass123"}'
```

**Full Output:**
```json
{
  "msg": "User needs to confirm their account"
}

HTTP_STATUS_CODE: 200
```

The registration succeeds with **HTTP 200** and returns the message `"User needs to confirm their account"`. This is defined in `app/api/views/auth.py` line ~140.

### Step 2: Login Before Activation

**Curl Command:**
```bash
curl -s -w "\nHTTP_STATUS_CODE: %{http_code}\n" -X POST http://localhost:7777/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "testuser@example.com", "password": "testpass123", "device": "test"}'
```

**Full Output:**
```json
{
  "error": "Account not activated"
}

HTTP_STATUS_CODE: 422
```

The login attempt is rejected with **HTTP 422** and JSON body `{"error": "Account not activated"}`. This is the exact response from `app/api/views/auth.py` lines 75–77:

```python
elif not user.activated:
    return jsonify(error="Account not activated"), 422
```

### Step 3: Direct Database Query

**SQL Command:**
```sql
SELECT id, email, activated, notification FROM users WHERE email = 'testuser@example.com';
```

**Result:**
```
 id |        email         | activated | notification
----+----------------------+-----------+--------------
  1 | testuser@example.com | f         | t
(1 row)
```

### Actual Column Values

| Column | Value | Type | Explanation |
|--------|-------|------|-------------|
| `activated` | `f` (False) | Boolean | Default value. User has not confirmed their account via the activation code. Defined in `app/models.py` with `default=False`. |
| `notification` | `t` (True) | Boolean | Default value. Notifications are enabled for new users. Defined in `app/models.py` with `default=True, server_default="1"`. |

### Rationale

The registration flow in `app/api/views/auth.py` (lines 87–141) performs these steps:
1. Validates email can be used as a mailbox (`email_can_be_used_as_mailbox()`)
2. Checks the email isn't already in use (`personal_email_already_used()`)
3. Creates the `User` record via `User.create(email=email, name=dirty_email, password=password)` — this sets `activated=False` by default
4. Creates a `Mailbox` for the user (verified=True)
5. Creates the first alias (with `simplelogin-newsletter` prefix)
6. Generates a 6-digit activation code in the `AccountActivation` table
7. Returns `{"msg": "User needs to confirm their account"}`

The login flow in `app/api/views/auth.py` (lines 29–84) checks:
1. Looks up the user by email
2. Verifies the password via `user.check_password(password)`
3. Checks `user.activated` — since it's `False`, returns `422 {"error": "Account not activated"}`

The `notification` column defaults to `True` (server_default="1") as defined in the User model, meaning newly created users have notifications enabled by default.

---

## 5. Dynamic Alias Limit Configuration Test

### Question
How does the `MAX_NB_EMAIL_FREE_PLAN` setting affect the `/api/user_info` endpoint's `max_alias_free_plan` response? Does changing the configuration and restarting affect existing users, new users, or both?

### Procedure

**Phase 1:** With `MAX_NB_EMAIL_FREE_PLAN=5` in `.env`:
- Created User A (`testuser@example.com`) and activated the account
- Logged in to obtain an API key
- Called `GET /api/user_info`

**Phase 2:** Changed `.env` to `MAX_NB_EMAIL_FREE_PLAN=10`:
- Restarted the Flask server (full process restart, not hot-reload)
- Created User B (`testuser2@example.com`) and activated the account
- Called `GET /api/user_info` for both User A and User B

### Phase 1 Results — `MAX_NB_EMAIL_FREE_PLAN=5`

**User A's `/api/user_info` response:**
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

`max_alias_free_plan` = **5** ✓

### Phase 2 Results — `MAX_NB_EMAIL_FREE_PLAN=10` (after restart)

**User A's `/api/user_info` response (existing user):**
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

`max_alias_free_plan` = **10** ✓ (changed dynamically for existing user!)

**User B's `/api/user_info` response (new user):**
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

`max_alias_free_plan` = **10** ✓

### Key Finding

**The `MAX_NB_EMAIL_FREE_PLAN` setting is fully dynamic and applies to ALL users (both existing and new) after a server restart.** The value is NOT stored per-user in the database. It is read from `app/config.py` at import time and evaluated at request time.

### Comparison Table

| User | Before Change (=5) | After Change (=10) | Behavior |
|------|--------------------|--------------------|----------|
| User A (existing) | `max_alias_free_plan: 5` | `max_alias_free_plan: 10` | **Changed dynamically** |
| User B (new) | N/A (not yet created) | `max_alias_free_plan: 10` | Reflects current config |

### Rationale — Code Path Analysis

The `max_alias_free_plan` value in the API response is computed dynamically on each request through this code path:

1. **`app/api/views/user_info.py:34`** — The response includes:
   ```python
   "max_alias_free_plan": user.max_alias_for_free_account(),
   ```

2. **`app/models.py:858-865`** — The method reads the config at call time:
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

3. **`app/config.py:120-124`** — The config value is loaded from the environment at module import time:
   ```python
   try:
       MAX_NB_EMAIL_FREE_PLAN = int(os.environ["MAX_NB_EMAIL_FREE_PLAN"])
   except Exception:
       print("MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value")
       MAX_NB_EMAIL_FREE_PLAN = 5
   ```

The critical insight is that `config.MAX_NB_EMAIL_FREE_PLAN` is a **module-level variable** loaded once at import time. When the server restarts, the module is re-imported, the environment variable is re-read, and the new value becomes effective for ALL subsequent requests regardless of when the user was created. The value is never persisted per-user in the database—it is always read from the `config` module at request time.

The only exception is users who have the `FLAG_FREE_OLD_ALIAS_LIMIT` flag set on their `flags` column, in which case they receive `config.MAX_NB_EMAIL_OLD_FREE_PLAN` (default: 15) instead. Neither of our test users had this flag set.

---

## Environment Configuration Reference

The following `.env` file was used for all experiments:

```
URL=http://localhost:7777
NOT_SEND_EMAIL=true
EMAIL_DOMAIN=sl.local
SUPPORT_EMAIL=support@sl.local
SUPPORT_NAME=Son from SimpleLogin
EMAIL_SERVERS_WITH_PRIORITY=[(10, "email.hostname.")]
DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin
FLASK_SECRET=secret
OPENID_PRIVATE_KEY_PATH=local_data/jwtRS256.key
OPENID_PUBLIC_KEY_PATH=local_data/jwtRS256.key.pub
WORDS_FILE_PATH=local_data/test_words.txt
DISABLE_ONBOARDING=true
LOCAL_FILE_UPLOAD=true
NAMESERVERS=1.1.1.1
PARTNER_API_TOKEN_SECRET=changeme
ALLOWED_REDIRECT_DOMAINS=[]
MAX_NB_EMAIL_FREE_PLAN=5
MEM_STORE_URI=redis://localhost:6379/0
```
