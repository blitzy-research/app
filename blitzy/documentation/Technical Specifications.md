# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Feature Objective

Based on the prompt, the Blitzy platform understands that the new feature requirement is to **conduct a comprehensive first-run initialization investigation and runtime behavior analysis** of the SimpleLogin email privacy and alias system. The user is not requesting code changes but rather a thorough empirical investigation that answers specific behavioral questions about the system when bootstrapped from scratch.

The feature requirements, restated with enhanced clarity:

- **Database Migration Analysis:** Execute Alembic migrations against an empty PostgreSQL database and determine (a) the total number of tables created and (b) the exact name of the last table created based on migration output order.
- **Web Server Startup Characterization:** Start the Flask/Gunicorn web server and identify the exact log message confirming readiness, along with the elapsed time in milliseconds from the first log entry to that ready message.
- **Email Handler Custom Port Verification:** Launch the `email_handler.py` SMTP service with port `25025` and capture the exact startup log message confirming the listener is bound to that port.
- **Registration and Pre-Activation Login Test:** Create a user account with email `testuser@example.com` and password `testpass123`, then immediately attempt login via the API before activating the account. Capture the exact JSON error response, the HTTP status code, and the full curl command output. Also query the database directly for the user record to report the actual boolean values of the `activated` and `notification` columns.
- **Dynamic Alias Limit Configuration Test:** Verify how the `MAX_NB_EMAIL_FREE_PLAN` setting affects the `/api/user_info` endpoint's `max_alias_free_plan` response. Create a user, call the endpoint, then change the configuration value to `10`, restart the server, create a second user, and compare responses for both users before and after the change.

Implicit requirements detected:
- A PostgreSQL 13 database must be provisioned and accessible
- The `.env` or `CONFIG` environment file must be properly configured with required variables (`DB_URI`, `FLASK_SECRET`, `URL`, `EMAIL_DOMAIN`, etc.)
- OpenID key files (`jwtRS256.key`, `jwtRS256.key.pub`) and word list files must be present in `local_data/`
- Redis may be needed for rate limiting and sessions (via `MEM_STORE_URI`)
- All investigation outputs must be documented in a markdown file named `app_2cd6ee777f8c.md` placed in `blitzy/documentation/`

### 0.1.2 Special Instructions and Constraints

- **CRITICAL: No source file modifications.** The user explicitly states: "Don't modify any of the source files, but feel free to create test config files or .env files to experiment with different settings."
- **SWE-AtlasQnA-Repo Rule:** Create a new markdown document named `app_2cd6ee777f8c.md` in the `blitzy/documentation` directory that comprehensively answers all questions posed.
- **Empirical Answers Required:** The user wants actual runtime observations, not theoretical code analysis: "I'm trying to understand the actual runtime behavior here, not just what the code suggests should happen."
- **Custom Config Files Allowed:** Test `.env` files and configuration files may be created for experimentation without modifying repository source.
- **Architectural Conventions:** The system uses Flask-Migrate (Alembic) for migrations, Flask blueprints for routing, `aiosmtpd` for SMTP, and environment variable-driven configuration via `python-dotenv`.

### 0.1.3 Technical Interpretation

These feature requirements translate to the following technical implementation strategy:

- To **determine migration table count**, we will set up a PostgreSQL database, configure `DB_URI`, and run `alembic upgrade head` while capturing the output to count `CREATE TABLE` operations and identify the last table created.
- To **capture web server startup logs**, we will launch the Flask development server via `server.py`'s `local_main()` or via `gunicorn wsgi:app` and capture stdout/stderr to identify the readiness message and compute timing deltas.
- To **verify email handler port binding**, we will invoke `python email_handler.py -p 25025` and capture the `LOG.i("Listen for port %s", args.port)` and `LOG.d("Start mail controller %s %s", controller.hostname, controller.port)` messages from the output.
- To **test the registration-then-login flow**, we will issue `POST /api/auth/register` followed by `POST /api/auth/login` via curl and observe that the login returns HTTP 422 with `{"error": "Account not activated"}` (as defined in `app/api/views/auth.py` line 77). We will then query PostgreSQL directly for the `activated` and `notification` column values.
- To **test dynamic alias limits**, we will create users under different `MAX_NB_EMAIL_FREE_PLAN` values and call `GET /api/user_info` to observe the `max_alias_free_plan` field, which is computed dynamically via `User.max_alias_for_free_account()` reading `config.MAX_NB_EMAIL_FREE_PLAN` at request time.


## 0.2 Repository Scope Discovery

### 0.2.1 Comprehensive File Analysis

The investigation touches all major subsystems of the SimpleLogin monorepo. The following is an exhaustive catalog of every file and folder relevant to the questions posed.

#### Entrypoint and Server Files

| File | Purpose | Relevance |
|------|---------|-----------|
| `server.py` | Central Flask bootstrap: assembles the web app, wires extensions, routes, admin views, error handling, CLI commands, rate limiting, sessions, and dev startup | Web server startup behavior, readiness logging, `create_app()` and `local_main()` functions |
| `wsgi.py` | WSGI app object for production Gunicorn launch (`from server import create_app; app = create_app()`) | Production server entrypoint |
| `email_handler.py` | SMTP inbound processor using `aiosmtpd.controller.Controller`; accepts `--port` argument; `main(port)` starts the controller | Email handler custom port startup, log messages at lines 2381–2404 |
| `job_runner.py` | Continuously drains queued background jobs in 10-second polling loop | Background job system behavior |
| `cron.py` | Defines all scheduled maintenance jobs (stats, cleanup, HIBP, trial notifications, etc.) | Cron task system understanding |
| `crontab.yml` | YAML-based cron schedule definitions for 15 recurring maintenance jobs | Cron scheduling configuration |
| `init_app.py` | Seeds SL domains and loads PGP keys on initialization | First-run initialization |
| `Dockerfile` | Multi-stage Docker build; exposes port 7777; launches Gunicorn with 2 workers | Deployment architecture |

#### Configuration Files

| File | Purpose | Relevance |
|------|---------|-----------|
| `example.env` | Documents all available environment variables with defaults and comments | Reference for all configurable values including `MAX_NB_EMAIL_FREE_PLAN`, `DB_URI`, `FLASK_SECRET` |
| `app/config.py` | Loads environment-driven settings; derives `MAX_NB_EMAIL_FREE_PLAN` (default 5), `MAX_NB_EMAIL_OLD_FREE_PLAN` (default 15), database URI, logging, and all feature flags | Dynamic alias limit behavior, port configuration, registration settings |
| `alembic.ini` | Alembic migration configuration pointing to `migrations/` script location | Migration execution configuration |
| `tests/test.env` | Test-specific environment configuration with `MAX_NB_EMAIL_FREE_PLAN=3` | Reference for test configuration patterns |
| `pyproject.toml` | Poetry dependency management; declares Python `^3.10`, all 50+ dependencies with version constraints | Environment setup, dependency versions |

#### Application Core Modules

| File | Purpose | Relevance |
|------|---------|-----------|
| `app/models.py` | Defines all 76 SQLAlchemy ORM models (3,844 lines); `User` class with `activated`, `notification` columns; `max_alias_for_free_account()` method | Database schema, user column values, alias limit computation |
| `app/db.py` | Creates SQLAlchemy engine, scoped session, and database connection | Database connectivity |
| `app/log.py` | Logging configuration with custom format, `EmailHandlerFilter`, `LOG` singleton | Log message format for all components |
| `app/extensions.py` | Flask-Login and Flask-Limiter initialization | Rate limiting for API endpoints |
| `app/pw_models.py` | `PasswordOracle` mixin providing bcrypt password hashing | Password storage for registration test |

#### API Views (Registration, Login, User Info)

| File | Purpose | Relevance |
|------|---------|-----------|
| `app/api/views/auth.py` | `/api/auth/register` (POST), `/api/auth/login` (POST), `/api/auth/activate` (POST) endpoints | Registration flow, pre-activation login error response |
| `app/api/views/user_info.py` | `/api/user_info` (GET) endpoint returning `max_alias_free_plan` from `user.max_alias_for_free_account()` | Dynamic alias limit verification via API |
| `app/api/base.py` | API blueprint definition; `authorize_request()` reads `Authentication` header, validates API key, checks `user.is_active()` and `user.disabled` | API authentication mechanism |
| `app/api/serializer.py` | Data shaping for alias-related API responses | API response structure |

#### Migration Infrastructure

| File/Folder | Purpose | Relevance |
|-------------|---------|-----------|
| `migrations/env.py` | Alembic environment bootstrap; imports `Base.metadata`, sets `DB_URI` for migrations | Migration runtime configuration |
| `migrations/versions/` | Contains 255 Alembic revision files spanning 2019–2024 | Table creation count, last-created table identification |
| `scripts/new-migration.sh` | Helper for generating migrations using temporary PostgreSQL 13 Docker | Migration workflow reference |
| `scripts/reset_local_db.sh` | Drops and recreates the schema, runs `alembic upgrade head`, seeds fake data | Local database reset workflow |

#### Supporting Data Files

| File | Purpose | Relevance |
|------|---------|-----------|
| `local_data/jwtRS256.key` | RSA private key for OpenID/JWT signing | Required for server startup |
| `local_data/jwtRS256.key.pub` | RSA public key for OpenID/JWT verification | Required for server startup |
| `local_data/test_words.txt` | Word list for generating random email alias suffixes | Required for alias creation during registration |
| `local_data/words.txt` | Full English word list for alias generation | Used when `WORDS_FILE_PATH` points here |
| `local_data/dkim.key` | DKIM private key for email signing | Optional for email handler |

#### Test Infrastructure

| File | Purpose | Relevance |
|------|---------|-----------|
| `tests/conftest.py` | Pytest bootstrap; creates Flask app, seeds test data, provides fixtures | Test configuration patterns |
| `tests/test.env` | Test environment configuration | Reference for test `MAX_NB_EMAIL_FREE_PLAN=3` |

### 0.2.2 Integration Point Discovery

- **API Registration Endpoint** (`app/api/views/auth.py` lines 87–141): Creates `User` record, generates `AccountActivation` code, sends activation email
- **API Login Endpoint** (`app/api/views/auth.py` lines 29–84): Checks `user.activated` flag; returns 422 with `{"error": "Account not activated"}` when `not user.activated`
- **API User Info Endpoint** (`app/api/views/user_info.py` lines 50–67): Returns `max_alias_free_plan` computed by `User.max_alias_for_free_account()`
- **User Model** (`app/models.py` lines 336–865): `activated` column (Boolean, default False), `notification` column (Boolean, default True, server_default "1"), `max_alias_for_free_account()` method reads `config.MAX_NB_EMAIL_FREE_PLAN`
- **Config Loading** (`app/config.py` lines 120–124): `MAX_NB_EMAIL_FREE_PLAN` loaded from environment; falls back to 5 if not set
- **Email Handler Main** (`email_handler.py` lines 2381–2404): `main(port)` creates `Controller` on `0.0.0.0:port`; logs `"Start mail controller %s %s"` and `"Listen for port %s"`
- **Database Schema** (`app/models.py`): 76 `__tablename__` declarations representing the full set of database tables

### 0.2.3 New File Requirements

- **CREATE:** `blitzy/documentation/app_2cd6ee777f8c.md` — Comprehensive Q&A markdown document answering all user questions with empirical runtime evidence, rationale, and code-based reasoning
- **CREATE (temporary):** `.env` test configuration file(s) for experimenting with different `MAX_NB_EMAIL_FREE_PLAN` settings without modifying source


## 0.3 Dependency Inventory

### 0.3.1 Key Packages

The following table catalogs the critical private and public packages relevant to this investigation, with exact versions sourced from `pyproject.toml` and `poetry.lock`:

| Registry | Package | Version | Purpose |
|----------|---------|---------|---------|
| PyPI | python | ^3.10 | Runtime (Dockerfile uses `python:3.10` base image) |
| PyPI | flask | ^1.1.2 | Web framework; `create_app()` in `server.py` |
| PyPI | gunicorn | ^20.0.4 | Production WSGI server; exposes port 7777 |
| PyPI | SQLAlchemy | 1.3.24 (pinned) | ORM for all 76 database models |
| PyPI | psycopg2-binary | ^2.9.3 | PostgreSQL driver |
| PyPI | Flask-Migrate | ^2.5.3 | Alembic migration wrapper for Flask |
| PyPI | aiosmtpd | ^1.2 | SMTP server for `email_handler.py` |
| PyPI | flask_login | ^0.5.0 | User session management |
| PyPI | flask-cors | ^3.0.9 | CORS support on `/api/*` endpoints |
| PyPI | Flask-Limiter | ^1.4 | Rate limiting (10/minute on auth endpoints) |
| PyPI | arrow | ^0.16.0 | Datetime handling via `ArrowType` columns |
| PyPI | sqlalchemy_utils | ^0.36.8 | `ArrowType` and utility types |
| PyPI | python-dotenv | ^0.14.0 | `.env` file loading for configuration |
| PyPI | bcrypt | ^3.2.0 | Password hashing via `PasswordOracle` |
| PyPI | coloredlogs | ^14.0 | Colored log output for local development |
| PyPI | sentry_sdk | ^2.16.0 | Error tracking (optional) |
| PyPI | newrelic | 8.8.0 (pinned) | APM monitoring integration |
| PyPI | redis | ^4.5.3 | Session store, rate limiter backend |
| PyPI | yacron | ^0.11.1 | YAML-based cron scheduler |
| PyPI | email_validator | ^1.1.1 | Email address validation |
| PyPI | dnspython | ^2.0.0 | DNS record lookups |
| PyPI | cryptography | 37.0.1 (pinned) | Cryptographic operations |
| PyPI | jwcrypto | ^0.8 | JOSE/JWT operations for OpenID |
| Infrastructure | PostgreSQL | 13 | Primary database (per `scripts/new-migration.sh` and `.github/workflows/main.yml`) |
| Infrastructure | Redis | v6 | Session/cache store (via `MEM_STORE_URI`) |

### 0.3.2 Dependency Updates

No dependency updates are required for this investigation. All packages are already declared in `pyproject.toml` and locked in `poetry.lock`. The task is purely investigative and produces only a documentation artifact.

### 0.3.3 Import Dependencies Relevant to Investigation

The following import chains are critical to understanding the runtime behavior being investigated:

- **Registration flow:** `app/api/views/auth.py` → `app/models.py` (User.create) → `app/db.py` (Session) → PostgreSQL
- **Login rejection:** `app/api/views/auth.py` → `User.activated` check → returns `jsonify(error="Account not activated"), 422`
- **User info endpoint:** `app/api/views/user_info.py` → `user_to_dict(user)` → `user.max_alias_for_free_account()` → `app/config.py` `MAX_NB_EMAIL_FREE_PLAN`
- **Email handler startup:** `email_handler.py` → `argparse` (port arg) → `aiosmtpd.controller.Controller` → `LOG.i` / `LOG.d` logging
- **Database migrations:** `alembic upgrade head` → `migrations/env.py` → `app.models.Base.metadata` → PostgreSQL DDL


## 0.4 Integration Analysis

### 0.4.1 Existing Code Touchpoints

The investigation touches the following integration points across the system. No modifications are made, but each must be exercised or inspected.

#### Direct Runtime Execution Points

- **`alembic upgrade head`** (via `migrations/env.py`): Runs all 255 migration revision files against PostgreSQL. The `env.py` module imports `Base.metadata` from `app.models` and injects `DB_URI` from `app.config`. Executes in online mode using `engine_from_config()` with `pool.NullPool`.
- **`server.py` → `create_app()`** (lines 139–217): Assembles the Flask application, registers all blueprints (`auth_bp`, `api_bp`, `dashboard_bp`, `developer_bp`, etc.), initializes Flask-Login, Flask-Limiter, Flask-Admin, Paddle, Coinbase, CORS, session handling, and the `/health` endpoint.
- **`server.py` → `local_main()`** (lines 572–588): Enables `COLOR_LOG`, creates app, configures debug toolbar, starts Flask dev server on port 7777 with `app.run(debug=True, port=7777)`.
- **`email_handler.py` → `main(port)`** (lines 2381–2393): Creates `Controller(MailHandler(), hostname="0.0.0.0", port=port)`, calls `controller.start()`, and logs `"Start mail controller %s %s"` with hostname and port.
- **`email_handler.py` → `__main__`** (lines 2396–2404): Parses `-p`/`--port` argument (default 20381), logs `"Listen for port %s"` with the specified port, then calls `main(port=args.port)`.

#### API Endpoints Exercised

- **`POST /api/auth/register`** (`app/api/views/auth.py` lines 87–141):
  - Validates email via `email_can_be_used_as_mailbox()` and `personal_email_already_used()`
  - Checks `DISABLE_REGISTRATION` flag
  - Creates `User` record with `User.create(email=email, name=dirty_email, password=password)`
  - Generates 6-digit activation code, creates `AccountActivation` record
  - Returns `{"msg": "User needs to confirm their account"}` with HTTP 200

- **`POST /api/auth/login`** (`app/api/views/auth.py` lines 29–84):
  - Looks up user by email (both original and canonical)
  - Checks password with `user.check_password(password)`
  - When `not user.activated`: returns `{"error": "Account not activated"}` with **HTTP 422**
  - Rate limited to 10/minute via `@limiter.limit("10/minute")`

- **`GET /api/user_info`** (`app/api/views/user_info.py` lines 50–67):
  - Requires API authentication via `@require_api_auth` decorator
  - Returns `user_to_dict(user)` which includes `"max_alias_free_plan": user.max_alias_for_free_account()`
  - `max_alias_for_free_account()` checks `FLAG_FREE_OLD_ALIAS_LIMIT` flag on the user; if not set, returns `config.MAX_NB_EMAIL_FREE_PLAN`

#### Database/Schema Interaction Points

- **User Table** (`users`): Created by migrations; columns `activated` (Boolean, default False), `notification` (Boolean, default True, server_default "1"), `email` (String(256), unique), `name`, password hash fields
- **Account Activation Table** (`account_activation`): Created during registration; stores `user_id`, `code` (6-digit), `tries` (default 3)
- **Mailbox Table** (`mailbox`): Auto-created for each new user in `User.create()` with `Mailbox.create(user_id=user.id, email=user.email, verified=True)`
- **Alias Table** (`alias`): First alias auto-created for non-partner users with prefix `"simplelogin-newsletter"`
- **All 76 Tables**: Created via sequential Alembic migrations from `migrations/versions/`

### 0.4.2 Configuration Dependencies

| Configuration Variable | Source | Default | Impact on Investigation |
|----------------------|--------|---------|------------------------|
| `DB_URI` | Environment / `.env` | `postgresql://myuser:mypassword@localhost:5432/simplelogin` | Database connection for migrations and API |
| `FLASK_SECRET` | Environment / `.env` | `secret` | Required for Flask app startup |
| `URL` | Environment / `.env` | `http://localhost:7777` | Server URL used in OpenID config |
| `EMAIL_DOMAIN` | Environment / `.env` | `sl.local` | Used for alias creation during user registration |
| `MAX_NB_EMAIL_FREE_PLAN` | Environment / `.env` | `5` (hardcoded fallback) | The alias limit returned by `/api/user_info` |
| `NOT_SEND_EMAIL` | Environment / `.env` | `true` | Prevents actual email sending during registration |
| `WORDS_FILE_PATH` | Environment / `.env` | `local_data/words.txt` | Word list for alias suffix generation |
| `OPENID_PRIVATE_KEY_PATH` | Environment / `.env` | `local_data/jwtRS256.key` | Required for OpenID metadata setup |
| `OPENID_PUBLIC_KEY_PATH` | Environment / `.env` | `local_data/jwtRS256.key.pub` | Required for JWKS endpoint |
| `EMAIL_SERVERS_WITH_PRIORITY` | Environment / `.env` | N/A (required) | MX server configuration |
| `SUPPORT_EMAIL` | Environment / `.env` | `support@sl.local` | Required environment variable |
| `DISABLE_REGISTRATION` | Environment / `.env` | Not set (registration open) | Must NOT be set to allow registration |
| `DISABLE_ONBOARDING` | Environment / `.env` | `true` (for self-hosted) | Disables onboarding emails |
| `LOCAL_FILE_UPLOAD` | Environment / `.env` | `true` | Uses local filesystem instead of S3 |
| `PARTNER_API_TOKEN_SECRET` | Environment / `.env` | `changeme` | Required for startup |
| `ALLOWED_REDIRECT_DOMAINS` | Environment / `.env` | `[]` | Redirect domain whitelist |
| `NAMESERVERS` | Environment / `.env` | `1.1.1.1` | DNS resolver configuration |


## 0.5 Technical Implementation

### 0.5.1 File-by-File Execution Plan

Every file listed below will be either created or inspected (no source modifications per user's explicit instruction).

#### Group 1 — Documentation Output (CREATE)

- **CREATE:** `blitzy/documentation/app_2cd6ee777f8c.md` — The primary deliverable. A comprehensive markdown document answering all user questions with:
  - Exact migration table count and last table name
  - Web server readiness log message and startup timing
  - Email handler port 25025 startup confirmation message
  - Registration/login flow results with full curl output, JSON error, HTTP status
  - Database query results for `activated` and `notification` columns
  - Dynamic alias limit experiment results (before/after configuration change)
  - Thinking and rationale behind every answer

#### Group 2 — Environment Configuration (CREATE, temporary)

- **CREATE:** `.env` file at the repository root — Test environment configuration for running the system, containing:
  - `DB_URI=postgresql://...` pointing to a local PostgreSQL 13 instance
  - `FLASK_SECRET=secret`
  - `URL=http://localhost:7777`
  - `EMAIL_DOMAIN=sl.local`
  - `MAX_NB_EMAIL_FREE_PLAN=5` (initial value, then changed to `10`)
  - `NOT_SEND_EMAIL=true`
  - All other required environment variables from `example.env`

#### Group 3 — Files Inspected (READ-ONLY, no modifications)

- **INSPECT:** `server.py` — Web server startup flow, log messages
- **INSPECT:** `email_handler.py` — SMTP handler startup, port argument parsing, log messages
- **INSPECT:** `app/api/views/auth.py` — Registration and login endpoint behavior
- **INSPECT:** `app/api/views/user_info.py` — User info endpoint, `max_alias_free_plan` field
- **INSPECT:** `app/models.py` — User model (`activated`, `notification` columns), `max_alias_for_free_account()`
- **INSPECT:** `app/config.py` — `MAX_NB_EMAIL_FREE_PLAN` loading logic
- **INSPECT:** `migrations/env.py` — Alembic environment configuration
- **INSPECT:** `migrations/versions/*` — All 255 migration files
- **INSPECT:** `alembic.ini` — Migration settings
- **INSPECT:** `app/db.py` — Database engine and session setup
- **INSPECT:** `app/log.py` — Log format definition
- **INSPECT:** `job_runner.py` — Background job runner behavior
- **INSPECT:** `cron.py` — Cron task definitions
- **INSPECT:** `crontab.yml` — Cron schedule configuration

### 0.5.2 Implementation Approach

The investigation follows a sequential empirical approach:

**Step 1 — Environment Setup:**
Establish the runtime by provisioning PostgreSQL 13, configuring environment variables via a custom `.env` file, and ensuring all required data files are in place (`local_data/jwtRS256.key`, `local_data/test_words.txt`, etc.).

**Step 2 — Migration Analysis:**
Execute `alembic upgrade head` against an empty database, capture stdout/stderr to count all `CREATE TABLE` statements, and identify the final table from the migration output sequence. Cross-reference with the 76 `__tablename__` declarations in `app/models.py`.

**Step 3 — Web Server Startup:**
Launch the Flask development server (via `python server.py` or `gunicorn wsgi:app -b 0.0.0.0:7777`) and capture timestamps from the first log entry through the readiness confirmation. Analyze the log format defined in `app/log.py`:
```
%(asctime)s - %(name)s - %(levelname)s - ...
```

**Step 4 — Email Handler Port Test:**
Run `python email_handler.py -p 25025` and capture the two key log messages:
- `LOG.i("Listen for port %s", args.port)` → prints before `main()` is called
- `LOG.d("Start mail controller %s %s", controller.hostname, controller.port)` → prints after the controller starts

**Step 5 — Registration and Login Test:**
Issue curl commands against the running web server:
- `POST /api/auth/register` with `{"email": "testuser@example.com", "password": "testpass123"}`
- `POST /api/auth/login` with `{"email": "testuser@example.com", "password": "testpass123"}`
- Capture the JSON response body and HTTP status code
- Query PostgreSQL: `SELECT activated, notification FROM users WHERE email = 'testuser@example.com'`

**Step 6 — Dynamic Alias Limit Test:**
- Create User A with `MAX_NB_EMAIL_FREE_PLAN=5`, call `/api/user_info`
- Modify `.env` to set `MAX_NB_EMAIL_FREE_PLAN=10`, restart server
- Create User B, call `/api/user_info` for both users
- Document whether the configuration change is reflected dynamically for both users or only newly created ones

### 0.5.3 Key Code Paths and Expected Outputs

#### Registration → Login Flow (Code Path)

```mermaid
sequenceDiagram
    participant Client
    participant FlaskAPI as Flask API
    participant UserModel as User Model
    participant DB as PostgreSQL

    Client->>FlaskAPI: POST /api/auth/register
    FlaskAPI->>UserModel: User.create(email, password)
    UserModel->>DB: INSERT INTO users (activated=False, notification=True)
    UserModel->>DB: INSERT INTO mailbox (verified=True)
    UserModel->>DB: INSERT INTO alias (simplelogin-newsletter prefix)
    FlaskAPI->>DB: INSERT INTO account_activation (6-digit code)
    FlaskAPI-->>Client: 200 {"msg": "User needs to confirm their account"}

    Client->>FlaskAPI: POST /api/auth/login
    FlaskAPI->>DB: SELECT FROM users WHERE email = ?
    FlaskAPI->>FlaskAPI: Check user.activated == False
    FlaskAPI-->>Client: 422 {"error": "Account not activated"}
```

#### Alias Limit Computation (Code Path)

```mermaid
flowchart LR
    A[GET /api/user_info] --> B[user_to_dict]
    B --> C[user.max_alias_for_free_account]
    C --> D{FLAG_FREE_OLD_ALIAS_LIMIT set?}
    D -->|Yes| E[config.MAX_NB_EMAIL_OLD_FREE_PLAN = 15]
    D -->|No| F[config.MAX_NB_EMAIL_FREE_PLAN = 5 or env value]
    E --> G[Return value in JSON]
    F --> G
```


## 0.6 Scope Boundaries

### 0.6.1 Exhaustively In Scope

All items below are part of this investigation and must be addressed in the output document.

#### Documentation Output

- `blitzy/documentation/app_2cd6ee777f8c.md` — Primary deliverable document

#### Configuration Files (temporary creation allowed)

- `.env` — Custom test environment configuration
- `example.env` — Reference for all available settings

#### Migration Infrastructure

- `alembic.ini` — Migration configuration
- `migrations/env.py` — Alembic environment bootstrap
- `migrations/versions/**/*.py` — All 255 migration revision files

#### Server and Entrypoints

- `server.py` — Flask web server startup (`create_app()`, `local_main()`)
- `wsgi.py` — WSGI production entrypoint
- `email_handler.py` — SMTP handler with port argument parsing
- `job_runner.py` — Background job runner
- `cron.py` — Scheduled maintenance jobs
- `crontab.yml` — Cron schedule definitions

#### Application Core

- `app/config.py` — Environment variable loading, `MAX_NB_EMAIL_FREE_PLAN`
- `app/models.py` — All 76 ORM models, User class, alias limit logic
- `app/db.py` — Database engine and session
- `app/log.py` — Logging format and configuration

#### API Endpoints

- `app/api/views/auth.py` — Registration, login, activation endpoints
- `app/api/views/user_info.py` — User info endpoint (`max_alias_free_plan`)
- `app/api/base.py` — API authentication mechanism

#### Supporting Files

- `local_data/jwtRS256.key` — OpenID private key (required for startup)
- `local_data/jwtRS256.key.pub` — OpenID public key (required for startup)
- `local_data/test_words.txt` — Word list for alias generation
- `local_data/dkim.key` — DKIM key (optional)
- `Dockerfile` — Deployment reference
- `scripts/new-migration.sh` — Migration workflow reference
- `scripts/reset_local_db.sh` — Database reset reference

#### Initialization Scripts

- `init_app.py` — Domain seeding and PGP key loading

### 0.6.2 Explicitly Out of Scope

- **Source code modifications:** No existing `.py`, `.html`, `.js`, `.css`, or other source files will be modified
- **Frontend/UI changes:** No template, static asset, or dashboard changes
- **Payment/subscription integration:** Paddle, Coinbase, Apple subscription modules
- **Social login providers:** GitHub, Google, Facebook, Proton OAuth flows
- **PGP encryption features:** Key management, email encryption/signing
- **Custom domain DNS verification:** Domain ownership, MX, SPF, DKIM verification
- **Performance optimization:** No profiling, caching, or query optimization work
- **CI/CD pipeline:** `.github/workflows/` is not relevant
- **Monitoring/observability:** New Relic, Upcloud monitoring, `monitoring.py`
- **Phone aliasing features:** Experimental phone number/reservation module
- **Email forwarding/reply logic:** The core email processing pipeline in `email_handler.py` (only startup behavior is in scope)
- **Refactoring of any existing code:** Explicitly prohibited by user instructions
- **Test suite execution:** The pytest suite in `tests/` is not executed (though `tests/test.env` is referenced for configuration patterns)


## 0.7 Rules for Feature Addition

### 0.7.1 User-Specified Rules

The following rules are explicitly stated by the user and must be strictly observed:

- **No source file modifications:** "Don't modify any of the source files" — no existing `.py`, `.html`, `.js`, `.css`, or configuration files in the repository may be altered in any way
- **Temporary config files permitted:** "Feel free to create test config files or .env files to experiment with different settings" — new `.env` files and custom configuration files may be created for experimentation purposes
- **Runtime behavior focus:** "I'm trying to understand the actual runtime behavior here, not just what the code suggests should happen" — all answers must be derived from actually executing the system and observing its real output, not from static code analysis alone
- **Document output requirement (implementation rule):** Create a markdown document named `app_2cd6ee777f8c.md` (matching the source branch name) in the `blitzy/documentation` directory
- **Rationale required:** Provide thinking and rationale behind every answer
- **Code-as-truth:** Do not make assumptions; base all answers on the code and runtime output as the authoritative source
- **No additional code in source repository:** Only the requested markdown document may be added to the repository

### 0.7.2 Verification Conventions

- Every user question must be answered with exact log output, exact JSON responses, or exact database query results obtained from live execution
- Curl commands and their full output must be captured and reproduced verbatim in the document
- Database query results must reflect actual `SELECT` output, not inferred values
- When the user asks for "exact" messages, those messages must be copied character-for-character from observed runtime output
- Timing measurements (e.g., milliseconds between log entries) must be derived from actual timestamp parsing, not estimation

### 0.7.3 Environment Integrity Rules

- The PostgreSQL database must start empty (fresh initialization) to match the user's "set everything up from scratch" requirement
- Each experiment (registration, login, alias limits) should be performed in sequence on the same running instance to reflect a realistic initialization flow
- When changing `MAX_NB_EMAIL_FREE_PLAN` to test dynamic behavior, the server must be fully restarted (not hot-reloaded) to confirm that `app/config.py` re-reads the environment variable at import time
- The `.env` file is the mechanism for changing configuration; it must not override variables unrelated to the experiment being conducted

### 0.7.4 Documentation Standards

- The output markdown document must be self-contained and comprehensible without access to the codebase
- Each question from the user must have a clearly labeled answer section
- Raw output (logs, curl responses, SQL results) must be presented in fenced code blocks
- Explanatory text must link the observed behavior back to specific source file locations (file path and line number) as rationale


## 0.8 References

### 0.8.1 Repository Files and Folders Searched

The following files and folders were inspected to derive all conclusions in this Agent Action Plan.

#### Root-Level Entrypoints and Configuration

| File Path | Purpose |
|-----------|---------|
| `server.py` | Flask web server bootstrap, `create_app()`, dev server on port 7777 |
| `wsgi.py` | WSGI production entrypoint importing `create_app` |
| `email_handler.py` | SMTP handler with aiosmtpd, argparse port selection, startup logging |
| `job_runner.py` | Background job runner polling loop for async tasks |
| `cron.py` | Scheduled cron task execution |
| `init_app.py` | Domain seeding and PGP key loading |
| `pyproject.toml` | Poetry dependency manifest, Python ^3.10, all package versions |
| `example.env` | Reference environment configuration with all available variables |
| `alembic.ini` | Alembic migration configuration |
| `Dockerfile` | Multi-stage Docker build, Gunicorn production command |

#### Core Application Modules

| File Path | Purpose |
|-----------|---------|
| `app/config.py` | Environment variable loading, `MAX_NB_EMAIL_FREE_PLAN` (default 5), all settings |
| `app/models.py` | 76 ORM models, User class (activated, notification columns), alias limit logic |
| `app/db.py` | SQLAlchemy engine creation, scoped session management |
| `app/log.py` | Log format string, "SL" logger configuration |

#### API Endpoints

| File Path | Purpose |
|-----------|---------|
| `app/api/views/auth.py` | Registration (`POST /api/auth/register`), login (`POST /api/auth/login`), activation |
| `app/api/views/user_info.py` | `GET /api/user_info` returning `max_alias_free_plan` |
| `app/api/base.py` | API blueprint at `/api`, authentication via `Authentication` header |

#### Migration Infrastructure

| File Path | Purpose |
|-----------|---------|
| `migrations/env.py` | Alembic environment bootstrap, imports `Base.metadata` from `app.models` |
| `migrations/versions/` | Directory containing 255 migration revision files |

#### Supporting Data Files

| File Path | Purpose |
|-----------|---------|
| `local_data/jwtRS256.key` | OpenID private key required for server startup |
| `local_data/jwtRS256.key.pub` | OpenID public key required for server startup |
| `local_data/test_words.txt` | Word list for alias generation |
| `local_data/dkim.key` | DKIM signing key |

#### Test Configuration

| File Path | Purpose |
|-----------|---------|
| `tests/test.env` | Test environment config (`MAX_NB_EMAIL_FREE_PLAN=3`, test DB URI) |

#### Folders Explored

| Folder Path | Purpose |
|-------------|---------|
| `/` (root) | Full repository structure discovery |
| `app/` | Core application package |
| `app/api/` | API blueprint and view modules |
| `app/api/views/` | API endpoint implementations |
| `migrations/` | Alembic migration infrastructure |
| `migrations/versions/` | Individual migration revision files |
| `local_data/` | Keys, certificates, and seed data |
| `scripts/` | Utility and automation scripts |
| `tests/` | Test suite and test configuration |

### 0.8.2 Existing Tech Spec Sections Reviewed

| Section | Key Information Derived |
|---------|------------------------|
| 1.1 Executive Summary | System overview, 76 tables, 255 migrations, architectural summary |
| 6.2 Database Design | Full schema organization across 8 domains, table relationships |

### 0.8.3 Attachments

No attachments were provided for this project. No Figma designs, wireframes, or external documents were included.

### 0.8.4 External URLs

No external URLs or Figma screen references were provided by the user.

### 0.8.5 Source Branch

- **Branch name:** `app_2cd6ee777f8c`
- **Head commit:** `2cd6ee77`
- **Output document path:** `blitzy/documentation/app_2cd6ee777f8c.md`


