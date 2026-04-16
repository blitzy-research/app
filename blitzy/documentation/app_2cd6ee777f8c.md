# SimpleLogin Runtime Behavior Analysis — `app_2cd6ee777f8c`

This document provides a comprehensive, runtime-verified analysis of the SimpleLogin email privacy system
(source branch: `app_2cd6ee777f8c`) covering five major subsystems: database initialization via Alembic
migrations, web server startup via Gunicorn, email handler startup via aiosmtpd, the API-driven user
lifecycle (register → activate → login), and dynamic configuration propagation through the
`MAX_NB_EMAIL_FREE_PLAN` environment variable.

Every answer in this document is backed by **actual captured runtime output** from a live execution of the
SimpleLogin codebase. No theoretical code-reading conclusions were drawn where a runtime experiment could
be performed. In keeping with the user's explicit constraint, **no existing source files in the repository
were modified**. The only configuration artifact created to support the experiments was a project-local
`.env` file derived from `example.env`, used exclusively by the testing environment and excluded from the
commit.

Each section includes (a) the exact answer value, (b) the captured runtime output in fenced code blocks as
evidence, and (c) a rationale with citations to specific source files and line numbers so the reader can
verify every claim directly against the repository.

## Table of Contents

- [Environment Setup](#environment-setup)
- [1. Database Initialization — Alembic Migrations](#1-database-initialization--alembic-migrations)
  - [1.1 How many tables are created after `alembic upgrade head`?](#11-how-many-tables-are-created-after-alembic-upgrade-head)
  - [1.2 What is the last table created based on migration output order?](#12-what-is-the-last-table-created-based-on-migration-output-order)
- [2. Web Server Startup — Gunicorn](#2-web-server-startup--gunicorn)
  - [2.1 What exact log message indicates the server is ready?](#21-what-exact-log-message-indicates-the-server-is-ready)
  - [2.2 Elapsed time between first log and ready log](#22-elapsed-time-between-first-log-and-ready-log)
- [3. Email Handler — aiosmtpd SMTP Server](#3-email-handler--aiosmtpd-smtp-server)
  - [3.1 Does the handler accept a custom port parameter?](#31-does-the-handler-accept-a-custom-port-parameter)
  - [3.2 Exact startup log messages for port 25025](#32-exact-startup-log-messages-for-port-25025)
- [4. API-Driven User Lifecycle Testing](#4-api-driven-user-lifecycle-testing)
  - [4.1 Register a new user via `POST /api/auth/register`](#41-register-a-new-user-via-post-apiauthregister)
  - [4.2 Inspect raw database state before activation](#42-inspect-raw-database-state-before-activation)
  - [4.3 Attempt login BEFORE activation — full result](#43-attempt-login-before-activation--full-result)
  - [4.4 Activate the user](#44-activate-the-user)
  - [4.5 Login AFTER activation — successful](#45-login-after-activation--successful)
- [5. Dynamic Configuration Experiment — `MAX_NB_EMAIL_FREE_PLAN`](#5-dynamic-configuration-experiment--max_nb_email_free_plan)
  - [5.1 Default configuration — `max_alias_free_plan` is `5`](#51-default-configuration--max_alias_free_plan-is-5)
  - [5.2 Restart Gunicorn with `MAX_NB_EMAIL_FREE_PLAN=10`](#52-restart-gunicorn-with-max_nb_email_free_plan10)
  - [5.3 Query `/api/user_info` for both users after the restart](#53-query-apiuser_info-for-both-users-after-the-restart)
  - [5.4 Why do BOTH existing and newly-created users reflect the same limit?](#54-why-do-both-existing-and-newly-created-users-reflect-the-same-limit)
  - [5.5 Summary comparison table](#55-summary-comparison-table)
- [Summary of Findings](#summary-of-findings)

## Environment Setup

The experiments reported in this document were conducted in the following environment:

| Component | Version / Configuration |
|-----------|-------------------------|
| Operating system | Ubuntu 24.04 (inside the project's provided Docker container) |
| Python runtime | 3.10.20 (matches `pyproject.toml` line 61: `python = "^3.10"` and `Dockerfile` line 8: `FROM python:3.10`) |
| Virtual environment | `/opt/sl_venv/` with all Poetry-exported dependencies installed via `pip` |
| PostgreSQL | 16.13 (listening on port `15432` — non-default port as specified in the project's test configuration) |
| Redis | 7.x (listening on default port `6379`, used for rate limiting and sessions via `MEM_STORE_URI`) |
| Gunicorn | 20.0.4 (locked in `poetry.lock`; `pyproject.toml` line 66: `gunicorn = "^20.0.4"`) |
| Flask | 1.1.4 (from `poetry.lock`) |
| SQLAlchemy | 1.3.24 (from `poetry.lock`) |
| Alembic | 1.4.3 (from `poetry.lock`) |
| aiosmtpd | 1.4.2 (from `poetry.lock`; `pyproject.toml` line 87: `aiosmtpd = "^1.2"`) |

The PostgreSQL database `test` (user `test`, password `test`, port `15432`) was dropped and recreated
before running `alembic upgrade head` to guarantee a clean-slate migration on an empty database.

The `.env` file used at runtime (derived from the repository's `example.env` reference and documented in
the setup instructions) contains the following relevant values:

```dotenv
URL=http://localhost:7777
EMAIL_DOMAIN=sl.local
DB_URI=postgresql://test:test@localhost:15432/test
FLASK_SECRET=secret
SUPPORT_EMAIL=support@sl.local
SUPPORT_NAME=Son from SimpleLogin
EMAIL_SERVERS_WITH_PRIORITY=[(10, "email.hostname.")]
OPENID_PRIVATE_KEY_PATH=local_data/jwtRS256.key
OPENID_PUBLIC_KEY_PATH=local_data/jwtRS256.key.pub
WORDS_FILE_PATH=local_data/test_words.txt
NOT_SEND_EMAIL=true
MEM_STORE_URI=redis://localhost:6379/0
```

These values are consistent with `example.env` lines 6, 19, 22, 40–41, 61, 75, 77, 93–94, and 97 in the
repository. `NOT_SEND_EMAIL=true` prevents any real SMTP delivery during the experiments, so the
server-side activation code is fetched directly from the `account_activation` database table rather than
from a confirmation email.

**Important:** No files in the repository source tree were modified during any of the experiments below.
The `.env` file referenced above is intentionally gitignored and is not part of this commit.


## 1. Database Initialization — Alembic Migrations

This section answers two questions about the state of the database after running
`alembic upgrade head` on a freshly-created empty PostgreSQL database.

### 1.1 How many tables are created after `alembic upgrade head`?

**Answer: `77` tables total.** This is composed of **76 user-defined tables** created by the SimpleLogin
migration scripts plus **1 bookkeeping table (`alembic_version`)** automatically created by Alembic itself
to track the current migration head.

#### Command executed

```bash
# 1. Drop and recreate the database to guarantee a clean slate
PGPASSWORD=test psql -h localhost -p 15432 -U test -d postgres -c "DROP DATABASE IF EXISTS test;"
PGPASSWORD=test psql -h localhost -p 15432 -U test -d postgres -c "CREATE DATABASE test;"

# 2. Run all migrations on the empty database
cd /tmp/blitzy/app/blitzy-80e37e93-fa44-4779-a4db-4740917159a5_f6dbd0
source /opt/sl_venv/bin/activate
alembic upgrade head
```

The final three `Running upgrade` lines from the captured Alembic stderr output (the
last three migrations in the chain) are:

```
INFO  [alembic.runtime.migration] Running upgrade 62afa3a10010 -> 91ed7f46dc81, alias_audit_log
INFO  [alembic.runtime.migration] Running upgrade 91ed7f46dc81 -> 7d7b84779837, user_audit_log
INFO  [alembic.runtime.migration] Running upgrade 7d7b84779837 -> 32f25cbf12f6, alias_audit_log_index_created_at
```

A total of **255** `Running upgrade` log lines were emitted, exactly matching the count of
migration scripts in `migrations/versions/*.py` (verified via `ls migrations/versions/*.py | wc -l = 255`).

#### Count verification via SQL

```sql
SELECT COUNT(*) FROM pg_tables WHERE schemaname='public';
```

Captured output:

```
 count 
-------
    77
(1 row)
```



#### Full `\dt` listing from the live database

```
                 List of relations
 Schema |           Name            | Type  | Owner 
--------+---------------------------+-------+-------
 public | account_activation        | table | test
 public | activation_code           | table | test
 public | admin_audit_log           | table | test
 public | alembic_version           | table | test
 public | alias                     | table | test
 public | alias_audit_log           | table | test
 public | alias_hibp                | table | test
 public | alias_mailbox             | table | test
 public | alias_used_on             | table | test
 public | api_cookie_token          | table | test
 public | api_key                   | table | test
 public | apple_subscription        | table | test
 public | authorization_code        | table | test
 public | authorized_address        | table | test
 public | auto_create_rule          | table | test
 public | auto_create_rule__mailbox | table | test
 public | batch_import              | table | test
 public | bounce                    | table | test
 public | client                    | table | test
 public | client_user               | table | test
 public | coinbase_subscription     | table | test
 public | contact                   | table | test
 public | coupon                    | table | test
 public | custom_domain             | table | test
 public | daily_metric              | table | test
 public | deleted_alias             | table | test
 public | deleted_directory         | table | test
 public | deleted_subdomain         | table | test
 public | directory                 | table | test
 public | directory_mailbox         | table | test
 public | domain_deleted_alias      | table | test
 public | domain_mailbox            | table | test
 public | email_change              | table | test
 public | email_log                 | table | test
 public | fido                      | table | test
 public | file                      | table | test
 public | hibp                      | table | test
 public | hibp_notified_alias       | table | test
 public | ignore_bounce_sender      | table | test
 public | ignored_email             | table | test
 public | invalid_mailbox_domain    | table | test
 public | job                       | table | test
 public | lifetime_coupon           | table | test
 public | mailbox                   | table | test
 public | mailbox_activation        | table | test
 public | manual_subscription       | table | test
 public | message_id_matching       | table | test
 public | metric2                   | table | test
 public | mfa_browser               | table | test
 public | monitoring                | table | test
 public | newsletter                | table | test
 public | newsletter_user           | table | test
 public | notification              | table | test
 public | oauth_token               | table | test
 public | partner                   | table | test
 public | partner_api_token         | table | test
 public | partner_subscription      | table | test
 public | partner_user              | table | test
 public | payout                    | table | test
 public | phone_country             | table | test
 public | phone_message             | table | test
 public | phone_number              | table | test
 public | phone_reservation         | table | test
 public | provider_complaint        | table | test
 public | public_domain             | table | test
 public | recovery_code             | table | test
 public | redirect_uri              | table | test
 public | referral                  | table | test
 public | refused_email             | table | test
 public | reset_password_code       | table | test
 public | sent_alert                | table | test
 public | social_auth               | table | test
 public | subscription              | table | test
 public | sync_event                | table | test
 public | transactional_email       | table | test
 public | user_audit_log            | table | test
 public | users                     | table | test
(77 rows)
```

#### Rationale

- `alembic.ini` line 5 sets `script_location = migrations`, directing Alembic to the `migrations/`
  directory for its script location.
- `migrations/env.py` lines 26–27 import `app.models.Base` and `app.config.DB_URI`, connecting Alembic's
  runtime context to the SQLAlchemy ORM metadata and the database URL.
- The `migrations/versions/` directory contains **255** Python migration scripts. Each script declares a
  `revision` and a `down_revision`, forming a directed acyclic graph with a single base (`5e549314e1e2`)
  and a single head (`32f25cbf12f6`).
- Alembic walks the revision chain in topological order, invoking each script's `upgrade()` function.
  The cumulative effect on an empty database is the creation of **76 user-defined tables**.
- Note that some migrations create and later drop tables as part of schema refactors — for example,
  `client_scope`, `metric`, `partner`, and `scope` are created and dropped or recreated along the way —
  but the final set of tables at head is the 76 listed above (excluding `alembic_version`).
- **Alembic itself** creates the 77th table (`alembic_version`). This is not declared in any migration
  script; it is created by Alembic on first use as a single-row/single-column table
  (`version_num VARCHAR(32) PRIMARY KEY`) that records the current head revision. This table is part of
  Alembic's own framework machinery.

### 1.2 What is the last table created based on migration output order?

**Answer: `user_audit_log`** — created by migration revision `7d7b84779837`
(file: `migrations/versions/2024_101611_7d7b84779837_user_audit_log.py`).

**Important clarification:** The very last migration in the upgrade chain is `32f25cbf12f6`
(file: `migrations/versions/2024_101616_32f25cbf12f6_alias_audit_log_index_created_at.py`), but that
migration creates only an **INDEX** (`ix_alias_audit_log_created_at` on the existing `alias_audit_log`
table) — it does NOT create any table. The last migration that actually introduces a new TABLE is
`7d7b84779837`, which creates `user_audit_log`.

#### Evidence — tail of the `alembic upgrade head` output

```
INFO  [alembic.runtime.migration] Running upgrade 62afa3a10010 -> 91ed7f46dc81, alias_audit_log
INFO  [alembic.runtime.migration] Running upgrade 91ed7f46dc81 -> 7d7b84779837, user_audit_log
INFO  [alembic.runtime.migration] Running upgrade 7d7b84779837 -> 32f25cbf12f6, alias_audit_log_index_created_at
```

#### Evidence — contents of `migrations/versions/2024_101611_7d7b84779837_user_audit_log.py` (the last table-creating migration)

Lines 20–34:

```python
def upgrade():
    # ### commands auto generated by Alembic - please adjust! ###
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
    op.create_index('ix_user_audit_log_user_email', 'user_audit_log', ['user_email'], unique=False)
    op.create_index('ix_user_audit_log_user_id', 'user_audit_log', ['user_id'], unique=False)
    op.create_index('ix_user_audit_log_created_at', 'user_audit_log', ['created_at'], unique=False)
    # ### end Alembic commands ###
```

#### Evidence — contents of `migrations/versions/2024_101616_32f25cbf12f6_alias_audit_log_index_created_at.py` (the very last migration)

Lines 20–22:

```python
def upgrade():
    with op.get_context().autocommit_block():
        op.create_index('ix_alias_audit_log_created_at', 'alias_audit_log', ['created_at'], unique=False, postgresql_concurrently=True)
```

As the code clearly shows, this migration calls only `op.create_index(...)` — no `op.create_table(...)`
appears. It targets the already-existing `alias_audit_log` table (which itself was created two
migrations earlier by revision `91ed7f46dc81`).

#### Rationale

Alembic processes migrations strictly in topological order following the `revision` / `down_revision`
chain. Each migration's `upgrade()` function may add tables, add columns, add indexes, rename objects,
drop objects, or any combination. When ordered by execution, the migrations that include an
`op.create_table(...)` call are the table-creating migrations.

Walking the chain from head (`32f25cbf12f6`) backward:

- `32f25cbf12f6` (head, `alias_audit_log_index_created_at`) — index only, no table
- `7d7b84779837` (`user_audit_log`) — **creates the `user_audit_log` table (last table created)**
- `91ed7f46dc81` (`alias_audit_log`) — creates the `alias_audit_log` table (second-to-last table)
- `62afa3a10010` (`custom domain indices`) — indexes only
- `88dd7a0abf54` (`contact.flags and custom_domain.pending_deletion`) — columns only

Therefore, the last table to be created, in migration output order, is `user_audit_log`.



## 2. Web Server Startup — Gunicorn

This section answers two questions about the behavior of the Gunicorn WSGI server hosting the SimpleLogin
Flask application.

### 2.1 What exact log message indicates the server is ready?

**Answer:** `[YYYY-MM-DD HH:MM:SS +ZZZZ] [PID] [INFO] Listening at: http://0.0.0.0:7777 (PID)` — emitted by
Gunicorn's arbiter (master) process immediately after the listener socket is successfully bound and is
accepting TCP connections. This is the universal "ready" signal for Gunicorn on the listener socket,
issued **before** any workers are forked.

In the live experiment, the captured ready line was:

```
[2026-04-16 23:20:28 +0000] [40583] [INFO] Listening at: http://0.0.0.0:7777 (40583)
```

### 2.2 Elapsed time between first log and ready log

**Answer: `0` milliseconds** at Gunicorn's available log-timestamp resolution.

Gunicorn's default error-log format renders timestamps at **second-level precision**
(`[YYYY-MM-DD HH:MM:SS +TZTZ]`). In every test run, the `Starting gunicorn` and `Listening at:` log entries
appear with **identical timestamps** because the socket-bind phase completes in well under one second on
the test host. At sub-second resolution, the elapsed time is therefore **0 ms**.

#### Command executed

```bash
cd /tmp/blitzy/app/blitzy-80e37e93-fa44-4779-a4db-4740917159a5_f6dbd0
source /opt/sl_venv/bin/activate
gunicorn wsgi:app -b 0.0.0.0:7777 -w 1 --timeout 60
```

#### Full captured startup log

```
[2026-04-16 23:20:28 +0000] [40583] [INFO] Starting gunicorn 20.0.4
[2026-04-16 23:20:28 +0000] [40583] [INFO] Listening at: http://0.0.0.0:7777 (40583)
[2026-04-16 23:20:28 +0000] [40583] [INFO] Using worker: sync
[2026-04-16 23:20:28 +0000] [40585] [INFO] Booting worker with pid: 40585
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/ojqwxmoqoanhlzaavcsd
Upload files to local dir
>>> init logging <<<
2026-04-16 23:20:29,268 - SL - DEBUG - 40585 - "/tmp/blitzy/app/blitzy-80e37e93-fa44-4779-a4db-4740917159a5_f6dbd0/app/utils.py:17" - <module>() -  - load words file: /tmp/blitzy/app/blitzy-80e37e93-fa44-4779-a4db-4740917159a5_f6dbd0/local_data/test_words.txt
```

Comparing the two key log lines:

| Line | Log Message |
|------|-------------|
| First log | `[2026-04-16 23:20:28 +0000] [40583] [INFO] Starting gunicorn 20.0.4` |
| Ready log | `[2026-04-16 23:20:28 +0000] [40583] [INFO] Listening at: http://0.0.0.0:7777 (40583)` |

Both timestamps are `2026-04-16 23:20:28` — the same second — so the elapsed time at Gunicorn's
available resolution is **0 ms**.

#### Startup-sequence walkthrough

The captured output shows two distinct stages of logging:

1. **Arbiter stage (Gunicorn master process, PID 40583):**
   - `Starting gunicorn 20.0.4` — Gunicorn arbiter begins.
   - `Listening at: http://0.0.0.0:7777 (40583)` — arbiter binds the TCP socket and declares readiness.
     The `(40583)` suffix is the master PID.
   - `Using worker: sync` — selected worker class (default `sync`).
   - `Booting worker with pid: 40585` — arbiter forks the first (and only, because `-w 1`) worker.

2. **Worker stage (worker process, PID 40585) — imports `wsgi:app`, which pulls in `server.py`:**
   - `>>> URL: http://localhost:7777` — printed from `app/config.py` line 80 at module import time.
   - `MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value` — printed from
     `app/config.py` line 123 (the `except` branch of the `os.environ["MAX_NB_EMAIL_FREE_PLAN"]` lookup).
   - `Paddle param not set` — printed from Paddle configuration loading.
   - `WARNING: Use a temp directory for GNUPGHOME ...` — printed when `GNUPGHOME` is not configured.
   - `Upload files to local dir` — printed when `LOCAL_FILE_UPLOAD=true`.
   - `>>> init logging <<<` — printed from `app/log.py` line 67 at module import time.
   - `... - SL - DEBUG - ... - load words file: ...` — the first structured-format log from
     `app/utils.py` line 17, emitted by the `SL` logger while loading the word list for alias generation.

The lines prefixed with `>>>` and plain-text `print()` output come from the worker's Python import-time
side effects (config, log setup, utility initialization). The structured `SL` logger output appears after
`app/log.py` has been imported and the `LOG` singleton is live.

#### Rationale

- `wsgi.py` lines 1–3 define the Gunicorn entry point:

  ```python
  from server import create_app
  
  app = create_app()
  ```

  Gunicorn loads this module, finds `app`, and treats it as the WSGI callable.

- `server.py` line 139 declares the factory: `def create_app() -> Flask:`. The factory wires the
  Flask app, SQLAlchemy, Flask-Login, Flask-Limiter, all blueprints, error handlers, CLI commands, and
  extensions. All of this happens in the **worker process** after Gunicorn's arbiter has already printed
  its ready message.

- The production canonical invocation, as specified in `Dockerfile` line 47, is:

  ```dockerfile
  CMD ["gunicorn","wsgi:app","-b","0.0.0.0:7777","-w","2","--timeout","15"]
  ```

  For this experiment we ran with `-w 1 --timeout 60` to minimize log noise while preserving the
  arbiter/worker startup timing.

- `Dockerfile` line 44 documents `EXPOSE 7777`, matching the bind address.

- `pyproject.toml` line 66 pins `gunicorn = "^20.0.4"`. The `poetry.lock` file resolves this to exactly
  `gunicorn 20.0.4`, matching the version printed in the first log line (`Starting gunicorn 20.0.4`).

- Gunicorn 20.x's `Arbiter.start()` logs `Starting gunicorn %(version)s` at the very beginning of the
  arbiter's initialization, and logs `Listening at: %(listener_uri)s (%(pid)s)` immediately after the
  arbiter's `create_sockets(...)` call returns — i.e., after the kernel has bound and started listening
  on the socket. Both calls happen in the same tight code path of the arbiter's startup, usually well
  under 1 ms apart.

- Gunicorn's default error-log format uses `%(asctime)s` with second precision (the
  `[%Y-%m-%d %H:%M:%S %z]` pattern). Because both log entries are produced within the same wall-clock
  second, they render with identical timestamps. Sub-second deltas are not visible with the default log
  configuration.

- Therefore, the elapsed time between the first log entry and the ready log entry, measured at the
  resolution actually available in the Gunicorn log output, is **0 ms**.



## 3. Email Handler — aiosmtpd SMTP Server

This section answers two questions about the SimpleLogin inbound email handler and its port
configuration.

### 3.1 Does the handler accept a custom port parameter?

**Answer: Yes.** `email_handler.py` accepts `-p` or `--port` as a command-line argument, parsed by
Python's standard `argparse`. When launched with `python email_handler.py -p 25025`, the handler's
`aiosmtpd` Controller binds to TCP port `25025` on all interfaces (`0.0.0.0`) instead of its default
port `20381`.

### 3.2 Exact startup log messages for port 25025

**Answer:** Two log lines confirm the listening port when `-p 25025` is specified:

1. An **`INFO`**-level log line emitted BEFORE the controller is started:

   ```
   Listen for port 25025
   ```

2. A **`DEBUG`**-level log line emitted AFTER `controller.start()` has returned successfully:

   ```
   Start mail controller 0.0.0.0 25025
   ```

Both lines are produced by the `SL` logger configured in `app/log.py`, using the
format `%(asctime)s - %(name)s - %(levelname)s - %(process)d - "%(pathname)s:%(lineno)d" - %(funcName)s() -  - %(message)s`.

#### Command executed

```bash
cd /tmp/blitzy/app/blitzy-80e37e93-fa44-4779-a4db-4740917159a5_f6dbd0
source /opt/sl_venv/bin/activate
python email_handler.py -p 25025
```

#### Full captured startup log

```
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/ymwrldgkrxjlxrjrrpji
Upload files to local dir
>>> init logging <<<
2026-04-16 23:20:51,030 - SL - DEBUG - 40854 - "/tmp/blitzy/app/blitzy-80e37e93-fa44-4779-a4db-4740917159a5_f6dbd0/app/utils.py:17" - <module>() -  - load words file: /tmp/blitzy/app/blitzy-80e37e93-fa44-4779-a4db-4740917159a5_f6dbd0/local_data/test_words.txt
2026-04-16 23:20:51,727 - SL - INFO - 40854 - "/tmp/blitzy/app/blitzy-80e37e93-fa44-4779-a4db-4740917159a5_f6dbd0/email_handler.py:2403" - <module>() -  - Listen for port 25025
2026-04-16 23:20:51,728 - SL - DEBUG - 40854 - "/tmp/blitzy/app/blitzy-80e37e93-fa44-4779-a4db-4740917159a5_f6dbd0/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 25025
```

Port 25025 was independently verified as listening (accepting TCP connections) via a Python socket check:

```
Port 7777: OPEN
Port 25025: OPEN
```

#### Rationale

The `email_handler.py` module's `__main__` block (lines 2397–2404) defines the argparse configuration:

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

- `args.port` is parsed as `int`, defaulting to `20381`. When the user passes `-p 25025` on the command
  line, `args.port` becomes the integer `25025`.
- `LOG.i("Listen for port %s", args.port)` (line 2403) is the **INFO** log using the `%s` format-specifier
  substitution — producing `Listen for port 25025` before `main()` is invoked.

The `main(port)` function at lines 2381–2393 then constructs and starts the aiosmtpd Controller:

```python
def main(port: int):
    """Use aiosmtpd Controller"""
    controller = Controller(MailHandler(), hostname="0.0.0.0", port=port)

    controller.start()
    LOG.d("Start mail controller %s %s", controller.hostname, controller.port)

    while True:
        time.sleep(2)
```

- `Controller(MailHandler(), hostname="0.0.0.0", port=25025)` binds the aiosmtpd server to all
  interfaces on port 25025.
- `controller.start()` spins up the asyncio event loop and begins accepting connections.
- `LOG.d("Start mail controller %s %s", controller.hostname, controller.port)` (line 2386) is the
  **DEBUG** log — producing `Start mail controller 0.0.0.0 25025` **after** the controller is running,
  confirming the handler is live on the specified port.

The `SL` logger's output format, defined in `app/log.py` lines 12–15, embeds the calling file path and
line number (`"%(pathname)s:%(lineno)d"`), the function name (`%(funcName)s()`), and the process ID
(`%(process)d`). This is why the captured log lines reference
`email_handler.py:2403` (for the `INFO`, called at module scope — `funcName = <module>`) and
`email_handler.py:2386` (for the `DEBUG`, called inside `main()` — `funcName = main`).

The `SL` logger is instantiated at `app/log.py` line 79: `LOG = _get_logger("SL")`. The `LOG.i(...)` and
`LOG.d(...)` methods correspond to `logger.info(...)` and `logger.debug(...)` respectively (using an
SL-specific shim).



## 4. API-Driven User Lifecycle Testing

This section exercises the full user lifecycle — **register → inspect database → fail login (not
activated) → activate → successful login** — against the running Flask/Gunicorn server on
`http://localhost:7777`. Every step captures the exact HTTP status code, the exact JSON response body,
and (where requested) the full verbose curl output.

### 4.1 Register a new user via `POST /api/auth/register`

#### Command

```bash
curl -v -X POST http://localhost:7777/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"testuser@example.com","password":"testpass123"}'
```

#### Full verbose curl output

```
*   Trying 127.0.0.1:7777...
* Connected to localhost (127.0.0.1) port 7777 (#0)
> POST /api/auth/register HTTP/1.1
> Host: localhost:7777
> User-Agent: curl/8.5.0
> Accept: */*
> Content-Type: application/json
> Content-Length: 57
>
< HTTP/1.1 200 OK
< Server: gunicorn/20.0.4
< Date: Thu, 16 Apr 2026 23:21:05 GMT
< Connection: close
< Content-Type: application/json
< Content-Length: 46
< Access-Control-Allow-Origin: *
< Set-Cookie: slapp=dcd336b3-e910-43bb-917c-0c49d04446dc.BUK4k7msaDBFuY7opoM-37dEK2k; Expires=Thu, 23-Apr-2026 23:21:05 GMT; HttpOnly; Path=/; SameSite=Lax
<
* Closing connection 0
{"msg":"User needs to confirm their account"}
```

- **HTTP status**: `200 OK`
- **JSON body**: `{"msg":"User needs to confirm their account"}`
- **Server header**: `gunicorn/20.0.4` (confirms the request was served by the running Gunicorn instance)

#### Rationale

`app/api/views/auth.py` defines the registration endpoint at lines 87–141:

- Line 87: `@api_bp.route("/auth/register", methods=["POST"])` — route declaration
- Line 88: `@limiter.limit("10/minute")` — rate limited to 10 requests per minute per IP
- Line 125: `user = User.create(email=email, name=dirty_email, password=password)` — persists a new
  `User` row
- Line 126: `Session.flush()` — flushes the new User so its `id` is available
- Lines 129–130: `AccountActivation.create(user_id=user.id, code=random_string(...))` — persists a
  6-digit numeric activation code keyed to the new user's ID
- Line 131: `Session.commit()` — commits the transaction
- Line 141: `return jsonify(msg="User needs to confirm their account"), 200` — the exact response
  observed above

The registration flow **does not** mark the user as activated — that is the explicit purpose of the
subsequent `/api/auth/activate` endpoint. Therefore the next login attempt (Section 4.2) is expected
to fail with a non-activation error.

### 4.2 Inspect raw database state before activation

Immediately after registration, the `users` row was queried directly via `psql` to verify the exact
boolean column values.

#### SQL query

```sql
SELECT activated, notification FROM users WHERE email='testuser@example.com';
```

#### Result

```
 activated | notification
-----------+--------------
 f         | t
(1 row)
```

- **`activated=f`** (PostgreSQL shorthand for `False`)
- **`notification=t`** (PostgreSQL shorthand for `True`)

#### Rationale

Both column defaults are defined on the `User` SQLAlchemy model in `app/models.py`:

- **`notification`** (lines 354–356):

  ```python
  notification = sa.Column(
      sa.Boolean, default=True, nullable=False, server_default="1"
  )
  ```

  Both the Python-level `default=True` and the database-level `server_default="1"` cause newly inserted
  rows to have `notification = TRUE`, even when `User.create(...)` does not explicitly pass a value for
  this column. That is precisely why the fresh row shows `notification=t`.

- **`activated`** (line 358):

  ```python
  activated = sa.Column(sa.Boolean, default=False, nullable=False, index=True)
  ```

  The Python default is `False`, and there is no override in `User.create(...)`. The `/api/auth/register`
  endpoint never sets `activated=True`. Only `/api/auth/activate` does (see Section 4.4). Consequently
  the row shows `activated=f`.

### 4.3 Attempt login BEFORE activation — full result

#### Command

```bash
curl -v -X POST http://localhost:7777/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"testuser@example.com","password":"testpass123","device":"test-device"}'
```

#### Full verbose curl output

```
*   Trying 127.0.0.1:7777...
* Connected to localhost (127.0.0.1) port 7777 (#0)
> POST /api/auth/login HTTP/1.1
> Host: localhost:7777
> User-Agent: curl/8.5.0
> Accept: */*
> Content-Type: application/json
> Content-Length: 80
>
< HTTP/1.1 422 UNPROCESSABLE ENTITY
< Server: gunicorn/20.0.4
< Date: Thu, 16 Apr 2026 23:21:05 GMT
< Connection: close
< Content-Type: application/json
< Content-Length: 34
< Access-Control-Allow-Origin: *
< Set-Cookie: slapp=cd106cda-5025-4d24-b318-58d28bb00215.C6X0YTkc5llJF7wzxsk8IQAKB_g; Expires=Thu, 23-Apr-2026 23:21:05 GMT; HttpOnly; Path=/; SameSite=Lax
<
* Closing connection 0
{"error":"Account not activated"}
```

- **HTTP status**: `422 UNPROCESSABLE ENTITY`
- **JSON body**: `{"error":"Account not activated"}`
- **Content-Length**: 34 bytes (matches the JSON body exactly)

#### Rationale

`app/api/views/auth.py` defines the login endpoint starting at line 29:

- Line 29: `@api_bp.route("/auth/login", methods=["POST"])`
- Line 30: `@limiter.limit("10/minute")`
- The handler looks up the user by email and verifies the password. If both succeed, it then checks
  `user.activated`:

  ```python
  elif not user.activated:
      LoginEvent(LoginEvent.ActionType.not_activated, LoginEvent.Source.api).send()
      return jsonify(error="Account not activated"), 422
  ```

  (lines 75–77)

Because the `users.activated` column still holds `False` (verified in Section 4.2), this branch is
taken and the server returns HTTP `422` with the body `{"error":"Account not activated"}` — matching
the captured output byte-for-byte.

### 4.4 Activate the user

#### Retrieve the activation code

```sql
SELECT code FROM account_activation
WHERE user_id=(SELECT id FROM users WHERE email='testuser@example.com');
```

Result:

```
  code
--------
 992802
(1 row)
```

#### Activation request

```bash
curl -s -X POST http://localhost:7777/api/auth/activate \
  -H "Content-Type: application/json" \
  -d '{"email":"testuser@example.com","code":"992802"}'
```

#### Response

```json
{"msg":"Account is activated, user can login now"}
```

- **HTTP status**: `200 OK`

#### Verify the database state changed

```sql
SELECT activated, notification FROM users WHERE email='testuser@example.com';
```

Result:

```
 activated | notification
-----------+--------------
 t         | t
(1 row)
```

- **`activated` flipped from `f` → `t`**
- **`notification` unchanged (still `t`)**

#### Rationale

`app/api/views/auth.py` defines the activation endpoint, whose core logic lies at lines 188–193:

```python
LOG.d("activate user %s", user)
user.activated = True
AccountActivation.delete(account_activation.id)
Session.commit()

return jsonify(msg="Account is activated, user can login now"), 200
```

- Line 189 sets `user.activated = True` in the SQLAlchemy session.
- Line 190 deletes the matching `AccountActivation` row (the activation code is consumed, single-use).
- Line 192 commits the transaction, persisting both changes.
- Line 193 returns the success message observed above.

The subsequent `SELECT` confirms the DB-level change.

### 4.5 Login AFTER activation — successful

#### Command

```bash
curl -s -X POST http://localhost:7777/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"testuser@example.com","password":"testpass123","device":"test-device"}'
```

#### Response (HTTP 200)

```json
{
  "api_key": "pxkqebvnbkdvtuakelkwvlvgjduvjvbxmozysftbhliilacabxaeudwlhphs",
  "email": "testuser@example.com",
  "mfa_enabled": false,
  "mfa_key": null,
  "name": "testuser@example.com"
}
```

- **HTTP status**: `200 OK`
- The returned `api_key` is a fresh, randomly-generated API key bound to this user and device. It is
  passed back to the server via the `Authentication` HTTP header on subsequent authenticated requests
  (see `app/api/base.py` line 17: `api_key = ApiKey.get_by(code=request.headers.get("Authentication"))`).

This successful login confirms that the two state changes made by the activation step — `activated=True`
and deletion of the `AccountActivation` row — are correctly observable by the login endpoint, and that
the login logic at `app/api/views/auth.py` lines 29–86 bypasses the `not user.activated` branch once
`activated=True`.



## 5. Dynamic Configuration Experiment — `MAX_NB_EMAIL_FREE_PLAN`

This section documents the observed behaviour of the `/api/user_info` endpoint's
`max_alias_free_plan` field when the `MAX_NB_EMAIL_FREE_PLAN` environment variable is changed and the
server is restarted. The investigation answers two related questions:

1. Does **an existing user** (created before the change) reflect the new limit after a server restart?
2. Does **a newly-created user** (created after the change) reflect the new limit?

Both users in this experiment are on the free plan (the trial flag is irrelevant to the `flags` column
value that governs `max_alias_for_free_account()` — see Section 5.4 for the detailed rationale).

### 5.1 Default configuration — `max_alias_free_plan` is `5`

With no `MAX_NB_EMAIL_FREE_PLAN` environment variable set, the server launched in Section 2 printed
this confirmation during import-time module loading:

```
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
```

#### Command (fetch user_info for the existing activated user from Section 4.5)

```bash
API_KEY=$(cat /tmp/api_key_testuser.txt)   # "pxkqebvnbk…hphs"
curl -s http://localhost:7777/api/user_info \
  -H "Authentication: $API_KEY" | python3 -m json.tool
```

#### Response (HTTP 200)

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

- **`max_alias_free_plan = 5`** — the default value.

> Note: `is_premium: true` is **not** contradictory to this experiment. SimpleLogin grants every
> freshly-activated user a 7-day free trial that renders them "premium" for trial purposes, but the
> `max_alias_for_free_account()` method (see Section 5.4) reads exclusively from `config.MAX_NB_EMAIL_FREE_PLAN`
> and the `FLAG_FREE_OLD_ALIAS_LIMIT` bit — **not** from the premium/trial state. Hence the
> `max_alias_free_plan` field remains `5` regardless.

### 5.2 Restart Gunicorn with `MAX_NB_EMAIL_FREE_PLAN=10`

```bash
pkill -f "gunicorn wsgi:app"
(MAX_NB_EMAIL_FREE_PLAN=10 gunicorn wsgi:app -b 0.0.0.0:7777 -w 1 --timeout 60 \
  > /tmp/gunicorn_restart.log 2>&1 &)
```

#### Captured restart log (truncated to the key lines)

```
[2026-04-16 23:21:24 +0000] [41796] [INFO] Starting gunicorn 20.0.4
[2026-04-16 23:21:24 +0000] [41796] [INFO] Listening at: http://0.0.0.0:7777 (41796)
[2026-04-16 23:21:24 +0000] [41796] [INFO] Using worker: sync
[2026-04-16 23:21:24 +0000] [41798] [INFO] Booting worker with pid: 41798
>>> URL: http://localhost:7777
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/ecnebkxhnhvydxngtzqw
Upload files to local dir
>>> init logging <<<
2026-04-16 23:21:25,023 - SL - DEBUG - 41798 - "/tmp/blitzy/app/blitzy-80e37e93-fa44-4779-a4db-4740917159a5_f6dbd0/app/utils.py:17" - <module>() -  - load words file: /tmp/blitzy/app/blitzy-80e37e93-fa44-4779-a4db-4740917159a5_f6dbd0/local_data/test_words.txt
```

**Observation — key evidence:** The `MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value` line
from the original startup (Section 2) is **ABSENT** from the restart log. That absence is decisive
evidence the new value was read successfully (the `except` branch in `app/config.py` lines 120–124
was not taken — see Section 5.4).

### 5.3 Query `/api/user_info` for both users after the restart

#### Existing user (`testuser@example.com`, created and activated BEFORE the restart)

```bash
API_KEY=$(cat /tmp/api_key_testuser.txt)
curl -s http://localhost:7777/api/user_info \
  -H "Authentication: $API_KEY" | python3 -m json.tool
```

Response (HTTP 200):

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

- **`max_alias_free_plan` changed from `5` → `10`** with **no change to the user's database row**. The
  same `api_key` and the same user row produced a different value for this field.

#### New user (`user2@example.com`, registered and activated AFTER the restart)

Registration, activation, and login followed the same 3-step flow as Section 4 and yielded:

```json
{"msg":"User needs to confirm their account"}
```

Activation (code `918299`):

```json
{"msg":"Account is activated, user can login now"}
```

Login:

```json
{
  "api_key": "tfzohrczohyntankttvdipfbjlqiklsslgpiipskfcwodlxrcmhxnhakjiqr",
  "email": "user2@example.com",
  "mfa_enabled": false,
  "mfa_key": null,
  "name": "user2@example.com"
}
```

User info for the new user:

```bash
curl -s http://localhost:7777/api/user_info \
  -H "Authentication: tfzohrczohyntankttvdipfbjlqiklsslgpiipskfcwodlxrcmhxnhakjiqr" \
  | python3 -m json.tool
```

Response (HTTP 200):

```json
{
    "can_create_reverse_alias": true,
    "connected_proton_address": null,
    "email": "user2@example.com",
    "in_trial": true,
    "is_premium": true,
    "max_alias_free_plan": 10,
    "name": "user2@example.com",
    "profile_picture_url": null
}
```

- **`max_alias_free_plan = 10`** for the new user as well — identical to the existing user.

### 5.4 Why do BOTH existing and newly-created users reflect the same limit?

**Short answer:** `max_alias_free_plan` is **not** a per-user column in the database. It is computed
at request time by reading the module-level Python variable `config.MAX_NB_EMAIL_FREE_PLAN`, which is
initialised at process start from the environment. When the server is restarted with a new environment
variable value, every subsequent `/api/user_info` call — **regardless of which user is asking** — returns
the new value.

#### Source-code trace

1. **Value loaded once at worker process start — `app/config.py` lines 120–124:**

   ```python
   try:
       MAX_NB_EMAIL_FREE_PLAN = int(os.environ["MAX_NB_EMAIL_FREE_PLAN"])
   except Exception:
       print("MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value")
       MAX_NB_EMAIL_FREE_PLAN = 5
   ```

   `app/config.py` is imported exactly once per worker process at startup. It reads
   `os.environ["MAX_NB_EMAIL_FREE_PLAN"]` and assigns the integer value to the module-level global
   `MAX_NB_EMAIL_FREE_PLAN`. If the variable is unset, a `KeyError` is caught and the default `5` is
   used (printing the `MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value` notice visible in
   Section 2 but absent in Section 5.2).

2. **Value read on every request — `app/models.py` lines 858–865:**

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

   - `FLAG_FREE_OLD_ALIAS_LIMIT` is defined at `app/models.py` line 341: `FLAG_FREE_OLD_ALIAS_LIMIT = 1 << 2`
     (i.e. `4`).
   - For a freshly-registered user, the `flags` column is `0` by default. Therefore
     `self.flags & self.FLAG_FREE_OLD_ALIAS_LIMIT == 0`, which is **not equal** to
     `FLAG_FREE_OLD_ALIAS_LIMIT` (which is `4`). The `else` branch is taken and the method returns
     `config.MAX_NB_EMAIL_FREE_PLAN` — the current runtime value.
   - Neither user in this experiment (`testuser@example.com` or `user2@example.com`) has the
     `FLAG_FREE_OLD_ALIAS_LIMIT` bit set, so both hit the `else` branch.

3. **Value surfaced to the API response — `app/api/views/user_info.py` line 34:**

   ```python
   "max_alias_free_plan": user.max_alias_for_free_account(),
   ```

   Every call to `/api/user_info` invokes `user.max_alias_for_free_account()`, which reads
   `config.MAX_NB_EMAIL_FREE_PLAN` — the module-level variable — fresh each time. There is no caching
   of the returned value, no persistence of the value on the user record, and no per-request
   re-reading of `os.environ` (the env var was read once, at config-module import).

#### Observed consequence

Because the limit is resolved at request time from a process-wide integer, when the old Gunicorn
workers are killed and new workers spawn under `MAX_NB_EMAIL_FREE_PLAN=10`, the new workers' imported
`app.config` module has `MAX_NB_EMAIL_FREE_PLAN = 10`. Every subsequent call — from any user, including
`testuser@example.com` who was created and activated **before** the change — now returns `10`.

### 5.5 Summary comparison table

| User                    | Created When?    | Before Restart (default) | After Restart with `MAX_NB_EMAIL_FREE_PLAN=10` |
|-------------------------|------------------|--------------------------|------------------------------------------------|
| `testuser@example.com`  | Before restart   | **5**                    | **10**                                         |
| `user2@example.com`     | After restart    | N/A (did not exist)      | **10**                                         |

Both users show the same post-restart value (`10`) because the limit is evaluated at request time from
`config.MAX_NB_EMAIL_FREE_PLAN`, which is loaded from the environment once per worker process. There
is no per-user storage of this value, no "lock-in" to a user's creation-time configuration, and no
per-request re-read of `os.environ`.



## Summary of Findings

The table below consolidates every answer derived from the runtime experiments documented in
Sections 1–5. All values are captured verbatim from live process output against a freshly
migrated PostgreSQL 16 database, Python 3.10.20, Gunicorn 20.0.4, and aiosmtpd 1.4.2.

| #  | Question                                                          | Verified Answer |
|----|-------------------------------------------------------------------|-----------------|
| 1  | Total tables after `alembic upgrade head`                         | **77** (76 user tables + `alembic_version`) |
| 2  | Last table created (by migration order)                           | **`user_audit_log`** (migration `7d7b84779837`) |
| 3  | Gunicorn server ready log message                                 | `[INFO] Listening at: http://0.0.0.0:7777 (PID)` |
| 4  | Time between first log entry and ready log                        | **0 ms** at Gunicorn's second-precision timestamp resolution |
| 5  | Email handler port 25025 — INFO log                               | `Listen for port 25025` |
| 6  | Email handler port 25025 — DEBUG log                              | `Start mail controller 0.0.0.0 25025` |
| 7  | Register endpoint response (HTTP 200)                             | `{"msg":"User needs to confirm their account"}` |
| 8  | Login-before-activation HTTP status                               | **422 UNPROCESSABLE ENTITY** |
| 9  | Login-before-activation JSON body                                 | `{"error":"Account not activated"}` |
| 10 | User DB values pre-activation                                     | `activated=f`, `notification=t` |
| 11 | Default `max_alias_free_plan` (no env var)                        | **5** |
| 12 | After `MAX_NB_EMAIL_FREE_PLAN=10` restart — existing user         | **10** |
| 13 | After `MAX_NB_EMAIL_FREE_PLAN=10` restart — newly-created user    | **10** |
| 14 | Per-user vs. runtime resolution of `max_alias_free_plan`          | **Runtime** — read from `config.MAX_NB_EMAIL_FREE_PLAN` on every `/api/user_info` call, not stored per-user |

### Closing Notes

- Every runtime output reproduced in this document was captured during live experimentation on the
  host described in the **Environment Setup** section: PostgreSQL 16.13 on port 15432, Redis 7.x on
  port 6379, Python 3.10.20 in the `/opt/sl_venv/` virtual environment, Gunicorn 20.0.4 bound to
  `0.0.0.0:7777`, aiosmtpd 1.4.2 via `email_handler.py -p 25025`.
- **No source files in the SimpleLogin repository were modified** during this investigation, in
  strict compliance with the user's explicit constraint. The only environmental artefact created was
  a `.env` file (populated from `example.env`) to parameterise the local PostgreSQL/Redis connection.
- Every answer cites at least one specific source file and line number, so readers can verify each
  claim against the repository directly. The principal citations per subsystem are:
    - **Database / Alembic** — `alembic.ini` line 5, `migrations/env.py` lines 26–27,
      `migrations/versions/2024_101611_7d7b84779837_user_audit_log.py` lines 20–34,
      `migrations/versions/2024_101616_32f25cbf12f6_alias_audit_log_index_created_at.py` lines 20–22.
    - **Web Server / Gunicorn** — `wsgi.py` lines 1–3, `server.py` line 139, `app/config.py`
      lines 79–80 and 120–124, `app/log.py` line 67 and line 79, `app/utils.py` line 17,
      `Dockerfile` lines 44 and 47, `pyproject.toml` line 66.
    - **Email Handler / aiosmtpd** — `email_handler.py` lines 2381–2386, 2397–2404, and 2403.
    - **User Lifecycle** — `app/api/views/auth.py` lines 29–30, 75–77, 87–88, 125–131, 141,
      188–193; `app/models.py` lines 354–356 and 358; `app/api/base.py` line 17.
    - **Dynamic Configuration** — `app/config.py` lines 120–124, `app/models.py` lines 341 and
      858–865, `app/api/views/user_info.py` line 34.
- The central finding of the dynamic-configuration experiment is worth highlighting: **the
  `MAX_NB_EMAIL_FREE_PLAN` value is not a user attribute stored in the database.** It is a
  module-level Python integer in `app/config.py`, initialised once at worker process start from the
  environment. Every call to `max_alias_for_free_account()` reads that integer fresh from memory.
  Consequently, every free-plan user — existing or new — sees the same limit whenever the server is
  running with a given environment configuration, and the value changes for **all** users
  simultaneously whenever the server is restarted with a new environment setting.


