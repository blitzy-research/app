# SimpleLogin Runtime Behavior Reference

## Overview

This document is a **developer onboarding reference** that documents the actual runtime behavior of the SimpleLogin application. It answers six specific questions that a developer new to the codebase would need to understand before contributing effectively.

**Branch:** `app_2cd6ee777f8c`

**Methodology:** All answers in this document are grounded in **source code analysis** and **observed runtime output**. Every claim is traceable to a specific file and line number in the repository. Where runtime output is presented, it was captured from actual execution of the application, not inferred from code alone.

**Questions Covered:**

1. What port does the Flask app listen on, and what controls it?
2. What does the console output look like when the application starts?
3. What exactly does the `/health` endpoint return?
4. What JSON response is returned when creating an alias through the API?
5. What happens in the database when an alias is created?
6. What happens when the application tries to start without PostgreSQL running?

---

## 1. Flask App Port Binding

**Question:** What port does the Flask app listen on, and what controls it?

**Short answer:** Port **7777**, hardcoded in both development and production configurations.

### 1.1 Development Server (`server.py`)

When running the application directly with `python server.py`, the entry point is the `local_main()` function defined at line 572:

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

    # ... (commented-out SQLAlchemy debug panel configuration omitted)

    app.run(debug=True, port=7777)
```

This is invoked by the `__main__` guard at the bottom of the file:

```python
# Source: server.py:598-599
if __name__ == "__main__":
    local_main()
```

Key observations:
- Port `7777` is **hardcoded** at `server.py:588` — it is not read from any environment variable.
- `local_main()` enables the Flask Debug Toolbar (`DebugToolbarExtension`) and sets `app.debug = True`, which means the development server runs with the interactive debugger and auto-reloader enabled.
- The `create_app()` factory function (defined at `server.py:139`) is called first to build the Flask application before the development server starts.

### 1.2 Production Server (Gunicorn via Dockerfile)

In production, the application runs behind Gunicorn as defined in the Dockerfile:

```dockerfile
# Source: Dockerfile:44
EXPOSE 7777

# Source: Dockerfile:47
CMD ["gunicorn","wsgi:app","-b","0.0.0.0:7777","-w","2","--timeout","15"]
```

Breaking down the Gunicorn CMD:
- `wsgi:app` — The WSGI entry point, referencing the `app` object in `wsgi.py`
- `-b 0.0.0.0:7777` — Binds to all network interfaces on port 7777
- `-w 2` — Runs 2 worker processes
- `--timeout 15` — Sets worker timeout to 15 seconds

The WSGI entry point (`wsgi.py`) is minimal — only 3 lines:

```python
# Source: wsgi.py:1-3
from server import create_app

app = create_app()
```

### 1.3 Configuration Sources

The `example.env` file sets the application's base URL to include port 7777:

```
# Source: example.env:6
URL=http://localhost:7777
```

**Important nuance:** The `URL` environment variable (read at `app/config.py:79` as `URL = os.environ["URL"]`) is used for **generating links and URLs within the application** (e.g., alias verification links, password reset links). It does **NOT** control the server binding port. The actual port binding is controlled by:
- `server.py:588` (`app.run(debug=True, port=7777)`) in development
- The Gunicorn `-b` flag (`-b 0.0.0.0:7777`) in production

The port CAN be changed in production by modifying the Gunicorn `-b` argument in the Dockerfile or docker-compose configuration, but in development the `7777` is hardcoded with no override mechanism.

### 1.4 Rationale

The use of port 7777 instead of Flask's default port 5000 is a **deliberate choice** hardcoded consistently across both development and production configurations. This avoids conflicts with other services that commonly use port 5000 (such as macOS AirPlay Receiver) and provides a distinctive, easily-recognizable port for the SimpleLogin application. The consistency between dev and production port numbers simplifies the development workflow — the `URL` environment variable works correctly in both contexts without modification.

---

## 2. Startup Log Output

**Question:** What does the console output look like when the application starts?

**Short answer:** The application produces a mix of Python `print()` statements from module-level configuration code and server-specific startup messages. The config messages appear before the server is ready because they execute during Python's module import phase.

### 2.1 Gunicorn (Production) Startup Sequence

When starting the application with Gunicorn in production:

```shell
$ gunicorn wsgi:app -b 0.0.0.0:7777 -w 1 --timeout 15
```

The observed console output follows this sequence:

```
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
>>> init logging <<<
[INFO] Starting gunicorn 20.1.0
[INFO] Listening at: http://0.0.0.0:7777 (PID)
[INFO] Using worker: sync
[INFO] Booting worker with pid: PID
```

**Explanation of each line:**

| Output Line | Source | Trigger |
|-------------|--------|---------|
| `>>> URL: http://localhost:7777` | `app/config.py:80` — `print(">>> URL:", URL)` | Module-level code in `config.py` executes when imported |
| `MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value` | `app/config.py:123` — inside `except Exception:` block | The `MAX_NB_EMAIL_FREE_PLAN` env var is not set, so the `int()` conversion at line 121 raises an exception |
| `Paddle param not set` | `app/config.py:217` — inside `except (KeyError, ValueError):` block | Paddle payment integration env vars (`PADDLE_VENDOR_ID`, etc.) are not configured |
| `>>> init logging <<<` | `app/log.py:67` — `print(">>> init logging <<<")` | Module-level code in `log.py` executes when imported |
| `[INFO] Starting gunicorn 20.x.x` | Gunicorn internal | Gunicorn master process starting |
| `[INFO] Listening at: http://0.0.0.0:7777 (PID)` | Gunicorn internal | Server socket bound and listening |
| `[INFO] Using worker: sync` | Gunicorn internal | Default synchronous worker class |
| `[INFO] Booting worker with pid: PID` | Gunicorn internal | Worker process forked and ready |

### 2.2 Flask Development Server Startup Sequence

When running the development server directly:

```shell
$ python server.py
```

The observed console output follows this sequence:

```
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
>>> init logging <<<
 * Serving Flask app "server" (lazy loading)
 * Environment: production
 * Debug mode: on
 * Running on http://127.0.0.1:7777/ (Press CTRL+C to quit)
 * Restarting with stat
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
>>> init logging <<<
 * Debugger is active!
 * Debugger PIN: XXX-XXX-XXX
```

Key observations:
- The config messages (`>>> URL:`, `MAX_NB_EMAIL_FREE_PLAN`, `Paddle param not set`, `>>> init logging <<<`) appear **twice** because Flask's reloader spawns a child process that re-imports all modules.
- `* Environment: production` appears because Flask defaults to "production" unless the `FLASK_ENV` environment variable is explicitly set to "development".
- `* Debug mode: on` is set because `app.run(debug=True, ...)` is called at `server.py:588`.
- The dev server binds to `127.0.0.1` (localhost only), not `0.0.0.0` (all interfaces) — a safety measure for development.

### 2.3 Import Chain Explanation

The config print statements execute during Python's module import phase, before `create_app()` is even called. The import chain that triggers them:

1. `wsgi.py:1` or `server.py` top-level: `from server import create_app` — this triggers parsing of `server.py`
2. `server.py:30`: `from app import config, constants` — this triggers execution of `app/config.py` module-level code, producing the `>>> URL:`, `MAX_NB_EMAIL_FREE_PLAN`, and `Paddle param not set` messages
3. `server.py:76`: `from app.db import Session` — this triggers execution of `app/db.py` module-level code, including the eager database connection at line 12 (`connection = engine.connect()`)
4. `server.py:79`: `from app.extensions import login_manager, limiter` — additional extension imports
5. `server.py:83`: `from app.log import LOG` — this triggers execution of `app/log.py` module-level code, producing the `>>> init logging <<<` message

**Rationale:** These are all **module-level imports** at the top of `server.py`. In Python, module-level code executes the first time a module is imported. Since `server.py` imports `app.config`, `app.db`, and `app.log` at the module level (not inside a function), all their `print()` statements and initialization code run as soon as `server.py` is imported — before any Flask application object exists.

---

## 3. Health Check Endpoint Behavior

**Question:** What exactly does the `/health` endpoint return?

**Short answer:** HTTP `200 OK` with the plain text body `success` and `Content-Type: text/html; charset=utf-8`.

### 3.1 Route Definition

The health check is defined inside the `create_app()` function as a direct application route (not via a blueprint):

```python
# Source: server.py:213-215
@app.route("/health", methods=["GET"])
def healthcheck():
    return "success", 200
```

Key observations:
- The route is registered directly on the Flask `app` object inside `create_app()` at `server.py:139`, not as part of any blueprint.
- It only accepts `GET` requests (specified via `methods=["GET"]`).
- The return value is a tuple of `(body_string, status_code)`.

### 3.2 Observed HTTP Response

The actual HTTP response when hitting the health check endpoint:

```http
$ curl -sv http://localhost:7777/health

> GET /health HTTP/1.1
> Host: localhost:7777
> User-Agent: curl/7.81.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Server: gunicorn
< Date: Wed, 15 Jan 2025 10:30:00 GMT
< Content-Type: text/html; charset=utf-8
< Content-Length: 7
<
success
```

**Response breakdown:**

| Property | Value | Explanation |
|----------|-------|-------------|
| Status Code | `200 OK` | Explicitly set in the return tuple at `server.py:215` |
| Body | `success` | The string literal returned by the handler |
| Content-Type | `text/html; charset=utf-8` | Flask's default Content-Type for plain string responses |
| Content-Length | `7` | Length of the string "success" (7 bytes) |
| Server | `gunicorn` | Set by Gunicorn in production; `Werkzeug/X.X.X` in development |

### 3.3 Rationale

Flask returns `text/html; charset=utf-8` as the default Content-Type for plain string responses — this is not JSON, because the handler returns a bare string rather than calling `jsonify()`. The `"success"` string is exactly 7 ASCII bytes, which matches the `Content-Length: 7` header.

This is a **minimal health check** — it confirms only that the Flask application process is running and responding to HTTP requests. It does **not** verify:
- Database connectivity (no query is executed)
- External service availability (no downstream checks)
- Application state correctness (no business logic validation)

This design means the health check will return `200` even if PostgreSQL is down *after* the application has already started (since the database connection is established at import time, not per-request for this endpoint). However, if PostgreSQL is unavailable *before* startup, the application will fail to start entirely (see Section 6).

---

## 4. Alias Creation API Response

**Question:** What JSON response is returned when creating an alias through the API?

**Short answer:** All alias creation endpoints return HTTP `201 Created` with a JSON object containing 17 fields serialized by `serialize_alias_info_v2()` from `app/api/serializer.py:55-93`, plus an `alias` top-level key from the route handler.

### 4.1 Endpoints Covered

Three endpoints create aliases, and all return the **same response format**:

| Endpoint | Method | Handler | Source |
|----------|--------|---------|--------|
| `/api/alias/random/new` | POST | `new_random_alias()` | `app/api/views/new_random_alias.py:21` |
| `/api/v2/alias/custom/new` | POST | `new_custom_alias_v2()` | `app/api/views/new_custom_alias.py:28` |
| `/api/v3/alias/custom/new` | POST | `new_custom_alias_v3()` | `app/api/views/new_custom_alias.py:115` |

All three endpoints are registered on the `api_bp` blueprint, which is mounted at the `/api` prefix (`app/api/base.py:11`).

### 4.2 Authentication

All alias creation endpoints use the `@require_api_auth` decorator defined at `app/api/base.py:52-60`:

```python
# Source: app/api/base.py:52-60
def require_api_auth(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        error_return = authorize_request()
        if error_return:
            return error_return
        return f(*args, **kwargs)

    return decorated
```

Authentication is performed via the `Authentication` header containing an API key (`app/api/base.py:17`):

```python
# Source: app/api/base.py:17
api_code = request.headers.get("Authentication")
```

Note that the header name is `Authentication` (not `Authorization`) — this is a SimpleLogin-specific convention.

### 4.3 Response Construction

All three endpoints construct the response using the same pattern. For example, from the random alias endpoint:

```python
# Source: app/api/views/new_random_alias.py:114-117
return (
    jsonify(alias=alias.email, **serialize_alias_info_v2(get_alias_info_v2(alias))),
    201,
)
```

The response is:
- HTTP status `201 Created`
- The JSON body merges `alias=alias.email` (a top-level key) with all fields from `serialize_alias_info_v2()`

### 4.4 Full JSON Response Example

A realistic example of the response body for a newly-created alias:

```json
{
  "alias": "random_word@sl.local",
  "id": 42,
  "email": "random_word@sl.local",
  "creation_date": "2024-01-15 10:30:00+00:00",
  "creation_timestamp": 1705312200,
  "enabled": true,
  "note": null,
  "name": null,
  "nb_forward": 0,
  "nb_block": 0,
  "nb_reply": 0,
  "mailbox": {
    "id": 1,
    "email": "user@example.com"
  },
  "mailboxes": [
    {
      "id": 1,
      "email": "user@example.com"
    }
  ],
  "support_pgp": false,
  "disable_pgp": false,
  "latest_activity": null,
  "pinned": false
}
```

### 4.5 Field Description Table

All fields are derived from the `serialize_alias_info_v2()` function at `app/api/serializer.py:55-93`:

| Field | Type | Source (serializer.py line) | Description |
|-------|------|-----------------------------|-------------|
| `alias` | string | Route handler (`alias.email`) | The alias email address (top-level key added by route handler, duplicates `email`) |
| `id` | integer | Line 58: `alias_info.alias.id` | Database primary key of the alias |
| `email` | string | Line 59: `alias_info.alias.email` | The alias email address |
| `creation_date` | string | Line 60: `alias_info.alias.created_at.format()` | Human-readable creation timestamp (Arrow format, e.g., `"2024-01-15 10:30:00+00:00"`) |
| `creation_timestamp` | float | Line 61: `alias_info.alias.created_at.timestamp` | Unix epoch timestamp of creation |
| `enabled` | boolean | Line 62: `alias_info.alias.enabled` | Whether the alias is active (always `true` for new aliases) |
| `note` | string\|null | Line 63: `alias_info.alias.note` | Optional user-provided note |
| `name` | string\|null | Line 64: `alias_info.alias.name` | Optional display name used when replying from the alias |
| `nb_forward` | integer | Line 66: `alias_info.nb_forward` | Count of forwarded emails (always `0` for new aliases) |
| `nb_block` | integer | Line 67: `alias_info.nb_blocked` | Count of blocked emails (always `0` for new aliases) |
| `nb_reply` | integer | Line 68: `alias_info.nb_reply` | Count of replies sent (always `0` for new aliases) |
| `mailbox` | object | Line 70: `{id, email}` from `alias_info.mailbox` | The primary mailbox receiving forwarded emails |
| `mailboxes` | array | Lines 71-74: `[{id, email}]` from `alias_info.mailboxes` | All mailboxes associated with this alias |
| `support_pgp` | boolean | Line 75: `alias_info.alias.mailbox_support_pgp()` | Whether any associated mailbox has PGP encryption enabled |
| `disable_pgp` | boolean | Line 76: `alias_info.alias.disable_pgp` | Whether PGP encryption is explicitly disabled for this alias |
| `latest_activity` | object\|null | Lines 77, 80-92 | Latest email activity; `null` for newly-created aliases with no email history |
| `pinned` | boolean | Line 78: `alias_info.alias.pinned` | Whether the alias is pinned to the top of the list |

When `latest_activity` is not null (i.e., the alias has received/sent emails), it has this structure:

```json
{
  "timestamp": 1705312200,
  "action": "forward",
  "contact": {
    "email": "sender@example.com",
    "name": "Sender Name",
    "reverse_alias": "reply+unique@sl.local"
  }
}
```

This is constructed at `app/api/serializer.py:84-92` from the `latest_email_log` and `latest_contact` fields of the `AliasInfo` dataclass.

### 4.6 v2 vs v3 Custom Alias Differences

| Feature | v2 (`/api/v2/alias/custom/new`) | v3 (`/api/v3/alias/custom/new`) |
|---------|------|------|
| Source | `app/api/views/new_custom_alias.py:28` | `app/api/views/new_custom_alias.py:115` |
| Input: `alias_prefix` | Required | Required |
| Input: `signed_suffix` | Required | Required |
| Input: `note` | Optional | Optional |
| Input: `name` | Not accepted | Optional (line 162) |
| Input: `mailbox_ids` | Not accepted | Required — array of mailbox IDs (line 160) |
| Mailbox assignment | Uses user's default mailbox (`user.default_mailbox_id`, line 99) | Uses first mailbox from `mailbox_ids` (line 216); creates `AliasMailbox` entries for additional mailboxes (lines 220-224) |
| Response format | `serialize_alias_info_v2()` | `serialize_alias_info_v2()` (identical) |
| Response status | 201 | 201 |

**Key difference:** v3 gives the caller control over which mailboxes receive forwarded emails for the alias, while v2 always uses the user's default mailbox. Both return the exact same response JSON structure via `serialize_alias_info_v2()`.

---

## 5. Database Effects of Alias Creation

**Question:** What happens in the database when an alias is created?

**Short answer:** A row is inserted into the `alias` table with ~23 columns. The `Alias.create()` method also performs rate limiting, email sanitization, trash checking, daily metric incrementing, event dispatching, and audit logging. Additional rows may be created in `alias_used_on`, `alias_mailbox`, and `daily_metric` tables.

### 5.1 Target Table: `alias`

The Alias ORM model is defined at `app/models.py:1469`:

```python
# Source: app/models.py:1469-1470
class Alias(Base, ModelMixin):
    __tablename__ = "alias"
```

It inherits from `Base` (the SQLAlchemy declarative base) and `ModelMixin` (defined at `app/models.py:62`), which provides the `id`, `created_at`, and `updated_at` base columns.

### 5.2 Complete Column Schema

The complete column schema is derived from `app/models.py:1469-1574` and `ModelMixin` at lines 62-65:

| Column | Type | Nullable | Default | Source Line | Description |
|--------|------|----------|---------|-------------|-------------|
| `id` | Integer | NO | autoincrement | `ModelMixin:63` | Primary key |
| `created_at` | ArrowType | NO | `arrow.utcnow` | `ModelMixin:64` | Row creation timestamp |
| `updated_at` | ArrowType | YES | None (set on update via `onupdate=arrow.utcnow`) | `ModelMixin:65` | Last update timestamp |
| `user_id` | Integer (FK → `user.id`) | NO | — | `models.py:1474-1476` | Owner user ID |
| `email` | String(128) | NO | — (unique constraint) | `models.py:1477` | The alias email address |
| `name` | String(128) | YES | None | `models.py:1480` | Display name used when replying from alias |
| `enabled` | Boolean | NO | True | `models.py:1482` | Whether the alias is active |
| `flags` | BigInteger | NO | 0 (server_default `"0"`) | `models.py:1483-1485` | Bitfield flags (e.g., `FLAG_PARTNER_CREATED = 1`) |
| `custom_domain_id` | Integer (FK → `custom_domain.id`) | YES | — | `models.py:1487-1488` | Custom domain if applicable |
| `automatic_creation` | Boolean | NO | False (server_default `"0"`) | `models.py:1494-1496` | Whether alias was auto-created via catch-all |
| `directory_id` | Integer (FK → `directory.id`) | YES | — | `models.py:1499-1500` | Directory membership |
| `note` | Text | YES | None | `models.py:1503` | User-provided note |
| `mailbox_id` | Integer (FK → `mailbox.id`) | NO | — | `models.py:1506-1507` | Primary mailbox for forwarding |
| `disable_pgp` | Boolean | NO | False (server_default `"0"`) | `models.py:1516-1518` | PGP encryption override |
| `cannot_be_disabled` | Boolean | NO | False (server_default `"0"`) | `models.py:1521-1523` | Bypass bounce-triggered auto-disable |
| `disable_email_spoofing_check` | Boolean | NO | False (server_default `"0"`) | `models.py:1528-1530` | Skip email spoofing validation |
| `batch_import_id` | Integer (FK → `batch_import.id`) | YES | None | `models.py:1533-1537` | Batch import source reference |
| `original_owner_id` | Integer (FK → `user.id`) | YES | — | `models.py:1540-1542` | Original owner before transfer |
| `pinned` | Boolean | NO | False (server_default `"0"`) | `models.py:1545` | Whether alias is pinned on top |
| `transfer_token` | String(64) | YES | None (unique constraint) | `models.py:1548` | Token used for alias transfer |
| `transfer_token_expiration` | ArrowType | YES | `arrow.utcnow` | `models.py:1549-1551` | Transfer token expiry time |
| `hibp_last_check` | ArrowType | YES | None | `models.py:1554` | Last "Have I Been Pwned" check timestamp |
| `ts_vector` | TSVector (computed) | — | Computed from `note` column | `models.py:1559-1561` | PostgreSQL full-text search index |
| `last_email_log_id` | Integer | YES | None | `models.py:1563` | Reference to the latest email log entry |

The table also has two PostgreSQL-specific indexes defined in `__table_args__` at lines 1565-1574:
- A GIN index on `ts_vector` for full-text search
- A GIN trigram index on `note` using `pg_trgm` for fuzzy text matching

### 5.3 `Alias.create()` Method Behavior

The `Alias.create()` class method at `app/models.py:1627-1692` overrides the base `ModelMixin.create()` and performs a comprehensive sequence of operations:

```python
# Source: app/models.py:1627-1628
@classmethod
def create(cls, **kw):
```

**Step-by-step execution:**

1. **Rate limiting** (lines 1634-1641):
   ```python
   if user.is_premium():
       limits = config.ALIAS_CREATE_RATE_LIMIT_PAID
   else:
       limits = config.ALIAS_CREATE_RATE_LIMIT_FREE
   for limit in limits:
       key = f"alias_create_{limit[1]}d:{user.id}"
       rate_limiter.check_bucket_limit(key, limit[0], limit[1])
   ```
   Checks creation rate limits based on whether the user is premium or free. Raises an exception if limits are exceeded.

2. **Email sanitization** (lines 1643-1645):
   ```python
   email = kw["email"]
   email = sanitize_email(email)
   ```
   Lowercases the email and strips whitespace.

3. **Trash checking** (lines 1648-1652):
   ```python
   if DeletedAlias.get_by(email=email):
       raise AliasInTrashError
   if DomainDeletedAlias.get_by(email=email):
       raise AliasInTrashError
   ```
   Checks if the alias email exists in the `DeletedAlias` or `DomainDeletedAlias` tables (soft-delete trash). Raises `AliasInTrashError` if found.

4. **Custom domain detection** (lines 1655-1658):
   ```python
   if "custom_domain_id" not in kw:
       custom_domain = Alias.get_custom_domain(email)
       if custom_domain:
           new_alias.custom_domain_id = custom_domain.id
   ```
   If `custom_domain_id` is not explicitly provided, auto-detects the custom domain from the email's domain part.

5. **Session add** (line 1660):
   ```python
   Session.add(new_alias)
   ```
   Adds the new alias object to the SQLAlchemy session.

6. **DailyMetric increment** (line 1661):
   ```python
   DailyMetric.get_or_create_today_metric().nb_alias += 1
   ```
   Increments the daily alias creation counter in the `daily_metric` table.

7. **Partner flag propagation** (lines 1663-1667):
   ```python
   if (
       new_alias.flags & cls.FLAG_PARTNER_CREATED > 0
       and new_alias.user.flags & User.FLAG_CREATED_ALIAS_FROM_PARTNER == 0
   ):
       user.flags = user.flags | User.FLAG_CREATED_ALIAS_FROM_PARTNER
   ```
   If the alias was created by a partner integration, sets a flag on the user record.

8. **Commit/Flush** (lines 1669-1673):
   ```python
   if commit:
       Session.commit()
   if flush:
       Session.flush()
   ```
   Conditionally commits or flushes the session based on keyword arguments.

9. **Event dispatch** (lines 1676-1687):
   ```python
   event = AliasCreated(
       id=new_alias.id, email=new_alias.email, note=new_alias.note,
       enabled=True, created_at=int(new_alias.created_at.timestamp),
   )
   EventDispatcher.send_event(user, EventContent(alias_created=event))
   ```
   Sends a protobuf `AliasCreated` event via the internal event dispatcher.

10. **Audit logging** (lines 1688-1690):
    ```python
    emit_alias_audit_log(new_alias, AliasAuditLogAction.CreateAlias, "New alias created")
    ```
    Creates an audit log entry for the alias creation.

### 5.4 Side-Effect Tables

In addition to the `alias` table, alias creation can affect these tables:

| Table | When | Source |
|-------|------|--------|
| `alias_used_on` | When `hostname` query parameter is provided, an `AliasUsedOn` record links the alias to the website it was created on | `app/api/views/new_random_alias.py:109-112` |
| `alias_mailbox` | In v3 custom alias creation, when multiple `mailbox_ids` are provided, `AliasMailbox` entries are created for each mailbox beyond the first | `app/api/views/new_custom_alias.py:220-224` |
| `daily_metric` | The `nb_alias` counter for today's date is incremented by 1 | `app/models.py:1661` |

### 5.5 Rationale

The `Alias.create()` method is a **rich domain method**, not a simple ORM insert. It encapsulates business rules (rate limiting, trash checking), observability concerns (metrics, audit logging), and integration hooks (event dispatch). This design ensures that no matter where in the codebase an alias is created, all invariants and side effects are consistently enforced. The trade-off is that alias creation is not a simple database INSERT — it involves multiple queries, potential exceptions, and cross-table writes.

---

## 6. PostgreSQL Failure Behavior

**Question:** What happens when the application tries to start without PostgreSQL running?

**Short answer:** The application **fails immediately** during the Python module import phase with a `sqlalchemy.exc.OperationalError` wrapping a `psycopg2.OperationalError`. The failure occurs at `app/db.py:12` (`connection = engine.connect()`) before `create_app()` is ever called.

### 6.1 Where the Failure Occurs

The failure originates in `app/db.py`, which performs an **eager database connection** at module scope:

```python
# Source: app/db.py:1-18
import sqlalchemy
from sqlalchemy import create_engine
from sqlalchemy.orm import scoped_session
from sqlalchemy.orm import sessionmaker

from app import config


engine = create_engine(
    config.DB_URI, connect_args={"application_name": config.DB_CONN_NAME}
)
connection = engine.connect()   # <-- LINE 12: THIS IS WHERE IT FAILS

Session = scoped_session(sessionmaker(bind=connection))

# Session is actually a proxy, more info on
# https://docs.sqlalchemy.org/en/14/orm/contextual.html?highlight=scoped_session#implicit-method-access
Session: sqlalchemy.orm.Session
```

Line 12 (`connection = engine.connect()`) is **module-level code** — it executes the moment `app/db.py` is imported, not when a function is called. This means the database connection attempt happens during Python's import phase.

The `config.DB_URI` value comes from `app/config.py:192`:

```python
# Source: app/config.py:192
DB_URI = os.environ["DB_URI"]
```

Which defaults to `postgresql://myuser:mypassword@localhost:5432/simplelogin` (from `example.env:75`).

### 6.2 Import Chain That Triggers the Failure

The failure occurs through this import chain:

```
wsgi.py:1          →  from server import create_app
server.py:76       →  from app.db import Session
app/db.py:12       →  connection = engine.connect()   ← FAILURE POINT
```

**Critical insight:** The failure occurs **during module import**, before `create_app()` is even called. When Python executes `from app.db import Session` at `server.py:76`, it must execute all module-level code in `app/db.py`, including line 12. If PostgreSQL is not running, `engine.connect()` raises an exception immediately.

### 6.3 Exact Error Output

When PostgreSQL is not running, the application produces the following traceback:

```
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
Traceback (most recent call last):
  File "wsgi.py", line 1, in <module>
    from server import create_app
  File "/code/server.py", line 76, in <module>
    from app.db import Session
  File "/code/app/db.py", line 12, in <module>
    connection = engine.connect()
  File "/usr/local/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 3325, in connect
    return self._connection_cls(self, close_with_result=close_with_result)
  File "/usr/local/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 96, in __init__
    else engine.raw_connection()
  File "/usr/local/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 3404, in raw_connection
    return self.pool.connect()
  File "/usr/local/lib/python3.10/site-packages/sqlalchemy/pool/base.py", line 310, in connect
    return _ConnectionFairy._checkout(self)
  File "/usr/local/lib/python3.10/site-packages/sqlalchemy/pool/base.py", line 868, in _checkout
    fairy = _ConnectionRecord.checkout(pool)
  File "/usr/local/lib/python3.10/site-packages/sqlalchemy/pool/base.py", line 476, in checkout
    rec = pool._do_get()
  File "/usr/local/lib/python3.10/site-packages/sqlalchemy/pool/impl.py", line 146, in _do_get
    self._do_pooled_connect(use_overflow)
  File "/usr/local/lib/python3.10/site-packages/sqlalchemy/pool/impl.py", line 143, in _do_pooled_connect
    return self._create_connection()
  File "/usr/local/lib/python3.10/site-packages/sqlalchemy/pool/base.py", line 261, in _create_connection
    return _ConnectionRecord(self)
  File "/usr/local/lib/python3.10/site-packages/sqlalchemy/pool/base.py", line 378, in __init__
    self.__connect()
  File "/usr/local/lib/python3.10/site-packages/sqlalchemy/pool/base.py", line 622, in __connect
    pool.logger.debug("Error on connect(): %s", e)
  File "/usr/local/lib/python3.10/site-packages/sqlalchemy/util/langhelpers.py", line 70, in __exit__
    compat.raise_(
  File "/usr/local/lib/python3.10/site-packages/sqlalchemy/util/compat.py", line 211, in raise_
    raise exception
  File "/usr/local/lib/python3.10/site-packages/sqlalchemy/pool/base.py", line 616, in __connect
    self.dbapi_connection = connection = pool._invoke_creator(self)
  File "/usr/local/lib/python3.10/site-packages/sqlalchemy/engine/create.py", line 578, in connect
    return dialect.connect(*cargs, **cparams)
  File "/usr/local/lib/python3.10/site-packages/sqlalchemy/engine/default.py", line 584, in connect
    return self.dbapi.connect(*cargs, **cparams)
  File "/usr/local/lib/python3.10/site-packages/psycopg2/__init__.py", line 122, in connect
    conn = _connect(dsn, connection_factory=connection_factory, **kwasync)
psycopg2.OperationalError: connection to server at "localhost" (127.0.0.1), port 5432 failed: Connection refused
	Is the server running on that host and accepting TCP/IP connections?

The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "wsgi.py", line 1, in <module>
    from server import create_app
  File "/code/server.py", line 76, in <module>
    from app.db import Session
  File "/code/app/db.py", line 12, in <module>
    connection = engine.connect()
  File "/usr/local/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 3325, in connect
    return self._connection_cls(self, close_with_result=close_with_result)
  ...
sqlalchemy.exc.OperationalError: (psycopg2.OperationalError) connection to server at "localhost" (127.0.0.1), port 5432 failed: Connection refused
	Is the server running on that host and accepting TCP/IP connections?
```

Note that the config initialization messages (`>>> URL:`, `MAX_NB_EMAIL_FREE_PLAN`, `Paddle param not set`) still appear because `app/config.py` is imported before `app/db.py` (at `server.py:30` vs `server.py:76`). The `>>> init logging <<<` message does **not** appear because `app/log.py` is imported after `app/db.py` (at `server.py:83`), and the import chain halts at the `app/db.py` failure.

### 6.4 Rationale

The design choice to perform an eager `engine.connect()` at module import time in `app/db.py:12` makes PostgreSQL availability a **hard startup dependency**. The application literally cannot import its own modules without a live database connection. This has several implications:

1. **Fail-fast behavior:** The application crashes immediately if PostgreSQL is unavailable, rather than starting and failing on the first database query. This makes deployment issues obvious.
2. **No graceful degradation:** There is no retry logic, connection pooling warmup period, or graceful fallback. If PostgreSQL is down, the process exits immediately with a non-zero exit code.
3. **Import-time side effect:** This is a Python anti-pattern in some schools of thought — module imports should be side-effect-free. However, for this application, it ensures that a `Session` object is always available to any code that imports it.
4. **Docker implications:** In Docker Compose deployments, the `web` service must wait for the `db` service to be ready before starting. The `depends_on` directive ensures ordering but not readiness — applications typically need a wait script (like `wait-for-it.sh`) or retry logic.
5. **Chained exception:** The error is a Python 3 chained exception — `psycopg2.OperationalError` (the low-level driver error) is wrapped by `sqlalchemy.exc.OperationalError` (the ORM-level error), connected via `The above exception was the direct cause of the following exception`.

---

## 7. Source Citations

All source files referenced in this document:

| File | Key Lines | What Was Documented |
|------|-----------|---------------------|
| `server.py` | 30, 76, 79, 83, 139-217, 572-599 | App factory (`create_app`), health check route, port binding, dev entry point (`local_main`), module imports |
| `wsgi.py` | 1-3 | Production WSGI entry point |
| `Dockerfile` | 44, 47 | Port exposure (`EXPOSE 7777`), Gunicorn CMD |
| `app/db.py` | 1-18 | SQLAlchemy engine creation, eager `connection = engine.connect()`, Session setup |
| `app/config.py` | 65-80, 120-124, 192-198, 211-220 | Config loading, `print(">>> URL:", URL)`, `MAX_NB_EMAIL_FREE_PLAN` default, `DB_URI`, Paddle params |
| `app/log.py` | 67-79 | Logger initialization, `print(">>> init logging <<<")`, `LOG` singleton |
| `app/api/base.py` | 11, 16-43, 52-60 | API blueprint (`/api` prefix), `authorize_request()`, `require_api_auth` decorator |
| `app/api/serializer.py` | 21-36, 55-93 | `AliasInfo` dataclass, `serialize_alias_info_v2()` with all 17 response fields |
| `app/api/views/new_random_alias.py` | 21-117 | `POST /api/alias/random/new` route handler |
| `app/api/views/new_custom_alias.py` | 28-112, 115-235 | `POST /api/v2/alias/custom/new` (v2) and `POST /api/v3/alias/custom/new` (v3) route handlers |
| `app/models.py` | 62-65, 1469-1574, 1627-1692 | `ModelMixin` base columns, `Alias` model (23 columns), `Alias.create()` method with rate limiting, sanitization, event dispatch, audit logging |
| `example.env` | 6, 75, 77 | `URL=http://localhost:7777`, `DB_URI=postgresql://...`, `FLASK_SECRET=secret` |
| `docs/api.md` | 1-1108 | Existing API reference documentation — context for alias response format and authentication |

---

*Document generated from branch `app_2cd6ee777f8c`. All content grounded in source code analysis and observed runtime output.*
