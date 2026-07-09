# SimpleLogin Dev-Stack Startup & Failure Investigation

**Repository:** SimpleLogin (`app`) — Flask monolith
**Git HEAD:** `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`
**Investigation type:** Read-only runtime observation. Every behavioral claim below is backed by **real, captured, unedited output** and the **exact command** that produced it. Every fenced evidence block is transcribed in full from captured bytes with **no elision** — where an ellipsis (`...`) appears it is either part of the actual emitted bytes (e.g. `swaks`'s `=== Trying 127.0.0.1:20381...` line, or the `500.html` "server issues..." text), part of a verbatim quotation of the user's question, or ordinary code shorthand in prose (e.g. `os.environ[...]`, `User.get_by(email=...)`); it is never a truncation of captured evidence. The single value not exercised at runtime is explicitly marked **`(inferred)`** (the Q1 alternate `load_user` trigger, §2.7); every other value was observed and is summarized in §5.4. No source file was created, modified, or deleted; the transient `.env` and all temporary scripts used for observation were removed afterward, leaving the repository byte-for-byte unchanged except for this single document.

This document answers three questions about how the stack behaves under three deliberately different setup conditions:

1. **Q1 — Empty database:** what exception is raised when the web app is started against an empty schema (migrations not run) and the login page is exercised.
2. **Q2 — Required services:** which Python services must run once migrations and `init_app.py` have been applied, with each service's captured readiness output and bound port.
3. **Q3 — Initialization skipped:** what the email handler returns and logs when an `@sl.local` message arrives while the `SLDomain` table (physical table `public_domain`) is empty.

---

## 1. Setup & Environment

### 1.1 Runtime, topology, and versions

The stack was brought up with the user-provided Docker image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`. The application container (`sl-app`) bind-mounts the repository at `/app` and runs the pre-built virtualenv at `/app/venv`. All containers run on the host network.

| Component | Container | Endpoint | Version (observed) |
|-----------|-----------|----------|--------------------|
| App / entry points | `sl-app` | — | Python **3.10.18** |
| PostgreSQL | `sl-postgres` | `127.0.0.1:5432` | **13.23** (Debian 13.23-1.pgdg13+1) |
| Redis | `sl-redis` | `127.0.0.1:6379` | server **7.4.9** |
| SMTP tester | `sl-app` | — | `swaks` **20201014.0** |

Key Python dependency versions, captured at runtime:

```
$ docker exec sl-app /app/venv/bin/python --version
Python 3.10.18

$ docker exec sl-app /app/venv/bin/python -c "import sqlalchemy, flask, psycopg2, redis; \
    import aiosmtpd; print('SQLAlchemy', sqlalchemy.__version__); print('Flask', flask.__version__); \
    print('psycopg2', psycopg2.__version__); print('redis', redis.__version__)"
SQLAlchemy 1.3.24
Flask 1.1.2
psycopg2 2.9.3 (dt dec pq3 ext lo64)
redis 4.6.0
# aiosmtpd (pip show): 1.4.2 ; Werkzeug 1.0.1 (from the web server's Server: header)
```

These match the constraints declared in `pyproject.toml`: `python = "^3.10"`, `SQLAlchemy = 1.3.24`, `flask = ^1.1.2`, `psycopg2-binary = ^2.9.3`, `aiosmtpd = ^1.2`, `redis = ^4.5.3`.

### 1.2 Configuration (transient `.env`)

The required environment keys are read via `os.environ[...]` in `app/config.py` and raise `KeyError` **at import time** if absent — so a `.env` is a hard prerequisite before any entry point can even be imported: `URL` (`app/config.py:79`), `EMAIL_DOMAIN` (`app/config.py:92`), `DB_URI` (`app/config.py:192`). A transient `.env` (a verbatim copy of `example.env`, which is gitignored via `.gitignore:4`) was used. The keys that matter for this investigation:

```
URL=http://localhost:7777          # example.env:6
NOT_SEND_EMAIL=true                # example.env:19
EMAIL_DOMAIN=sl.local              # example.env:22
DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin   # example.env:75
FLASK_SECRET=secret                # example.env:77
# MEM_STORE_URI is NOT set (see §3.2.4 for why this matters to Redis)
```

**Port choice (stated explicitly).** `example.env:75` uses port **5432** for `DB_URI`, whereas `scripts/reset_local_db.sh` uses **15432**. This investigation bound and used **5432** — the value in the canonical `.env` above. (`sl-postgres` listens on `127.0.0.1:5432`; a separate `sl-test-db` on `15432` exists for pytest only and was not used here.)

### 1.3 Canonical startup order and where each question deviates

The canonical order is: **PostgreSQL → `alembic upgrade head` (migrations) → `python init_app.py` → services.** Each question deliberately breaks one link:

- **Q1** skips migrations → empty schema.
- **Q3** runs migrations but skips `init_app.py` → `public_domain` empty.
- **Q2** performs the full order, then starts all services.

```mermaid
flowchart TD
    A[PostgreSQL running, empty simplelogin DB] --> B{alembic upgrade head run?}
    B -- No --> Q1[Q1: python server.py + open login -> ProgrammingError / UndefinedTable]
    B -- Yes: schema built --> C{python init_app.py run?}
    C -- No: public_domain empty --> Q3[Q3: email_handler.py + swaks @sl.local -> 550 SL E515 + rejection log]
    C -- Yes: SLDomain populated --> Q2[Q2: start all services -> capture readiness banners]
    Q2 --> D[Fully operational stack]
```

### 1.4 Migration command note (verified by reading + observed)

`README.md:433` documents `flask db upgrade`, but `Migrate()` is **not** registered on the app entry points: `shell.py` imports `flask_migrate` (`shell.py:1`) and calls `flask_migrate.upgrade()` (`shell.py:20`), but that call sits inside a dead `if False:` block (`shell.py:11`), and `shell.py`'s `__main__` (`shell.py:72`) merely calls `embed()` (`shell.py:73`). Therefore **`alembic upgrade head`** (`alembic.ini:5` → `script_location = migrations`) is the reliable path, consistent with the `CONTRIBUTING.md:106` quickstart. `migrations/env.py` imports `Base` (`migrations/env.py:26`) and `DB_URI` (`migrations/env.py:27`) and sets `target_metadata = Base.metadata` (`migrations/env.py:28`).

Observed: `alembic upgrade head` applied exactly **255** versioned migration scripts. The full run was captured to `/tmp/q3_alembic.log` (264 lines total = 9 preamble/context lines + 255 `Running upgrade` lines). Below are the complete env-preamble/context lines, the first three and last three `Running upgrade` lines, and the exact count proven by `grep -c` against the captured file. The intermediate 249 `Running upgrade` lines are omitted **for length only** (they are present in the captured log; the `grep -c` below counts the complete file):

```
$ docker exec -w /app sl-app bash -c '/app/venv/bin/alembic upgrade head > /tmp/q3_alembic.log 2>&1; echo "alembic exit=$?"'
alembic exit=0

# --- complete preamble / alembic context lines (the 9 non-"Running upgrade" lines) ---
$ docker exec sl-app grep -v "Running upgrade" /tmp/q3_alembic.log
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/gctupbbwuujomluptsdg
Upload files to local dir
>>> init logging <<<
2026-07-08 22:16:26,618 - SL - DEBUG - 3007 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.

# --- first three Running-upgrade lines (base of the chain; note the empty from-revision) ---
$ docker exec sl-app grep "Running upgrade" /tmp/q3_alembic.log | head -3
INFO  [alembic.runtime.migration] Running upgrade  -> 5e549314e1e2, empty message
INFO  [alembic.runtime.migration] Running upgrade 5e549314e1e2 -> 3cd10cfce8c3, empty message
INFO  [alembic.runtime.migration] Running upgrade 3cd10cfce8c3 -> 0256244cd7c8, empty message

# --- last three Running-upgrade lines (head of the chain) ---
$ docker exec sl-app grep "Running upgrade" /tmp/q3_alembic.log | tail -3
INFO  [alembic.runtime.migration] Running upgrade 62afa3a10010 -> 91ed7f46dc81, alias_audit_log
INFO  [alembic.runtime.migration] Running upgrade 91ed7f46dc81 -> 7d7b84779837, user_audit_log
INFO  [alembic.runtime.migration] Running upgrade 7d7b84779837 -> 32f25cbf12f6, alias_audit_log_index_created_at

# --- exact count against the complete captured file ---
$ docker exec sl-app grep -c "Running upgrade" /tmp/q3_alembic.log
255
```

### 1.5 DB-state recipes (used to reach each question's precondition)

- **Empty schema (Q1):** `echo 'drop schema public cascade; create schema public;' | psql "$DB_URI"`, and do **not** run alembic. This is valid because `create_all` appears **nowhere** in `server.py` or `app/` — the app never self-creates the schema:

  ```
  $ grep -rn "create_all" server.py app/ | wc -l
  0
  ```

- **Migrated-but-uninitialized (Q3):** reset schema, `alembic upgrade head`, but **skip** `python init_app.py` **and** `flask dummy-data` (dummy-data would also seed domains). Then `SELECT count(*) FROM public_domain;` returns `0`.
- **Fully-initialized (Q2):** reset schema, `alembic upgrade head`, then `python init_app.py`.

### 1.6 How processes log

All processes log single-line records to stdout via `app/log.py`, whose format string (`app/log.py:12-14`) embeds the emitting `"pathname:lineno" - funcName()`, making every line self-identifying:

```
"%(pathname)s:%(lineno)d" - %(funcName)s() - %(message_id)s - %(message)s
```

One consequence used below: `app/log.py:70-71` disables the werkzeug logger (`logging.getLogger("werkzeug").disabled = True`), which suppresses werkzeug's `* Running on ...` line (see §3.1).

---

## 2. Q1 — Empty-schema exception

> **Question (verbatim):** "With PostgreSQL running and an empty database created (no tables, migrations not run), what error do I actually hit when I start the server with `python server.py` and then try to open the login page in the browser? I'm looking for the full Python exception that gets raised."

### 2.1 Short answer

Opening the login page (a bare `GET /auth/login`) succeeds with **HTTP 200** and touches no database. The exception is raised only when the **first ORM query** executes — on submitting the login form (`POST /auth/login`). That query is `User.get_by(email=...)` at `app/auth/views/login.py:43`, executed by `ModelMixin.get_by` at `app/models.py:84` (`Session.query(cls).filter_by(**kw).first()`). Because the `users` relation does not exist in the empty schema, the exact exception is:

```
sqlalchemy.exc.ProgrammingError: (psycopg2.errors.UndefinedTable) relation "users" does not exist
```

i.e. a **`sqlalchemy.exc.ProgrammingError`** wrapping **`psycopg2.errors.UndefinedTable`** with the Postgres message **`relation "users" does not exist`**. In debug mode the request returns **HTTP 500** rendered as the app's **custom `templates/error/500.html`** page (not the Werkzeug interactive debugger — see §2.5).

### 2.2 BEFORE state — proof the schema is empty

Complete, unedited output of the schema reset (all 82 cascaded objects listed by Postgres in the `DETAIL`):

```
$ docker exec sl-postgres psql -U myuser -d simplelogin -c 'drop schema public cascade; create schema public;'
NOTICE:  drop cascades to 82 other objects
DETAIL:  drop cascades to table alembic_version
drop cascades to type plan_enum
drop cascades to table file
drop cascades to table users
drop cascades to table activation_code
drop cascades to table client
drop cascades to table alias
drop cascades to table authorization_code
drop cascades to table client_user
drop cascades to table oauth_token
drop cascades to table redirect_uri
drop cascades to table reset_password_code
drop cascades to table contact
drop cascades to type planenum2
drop cascades to table subscription
drop cascades to table email_log
drop cascades to table deleted_alias
drop cascades to table email_change
drop cascades to table api_key
drop cascades to table alias_used_on
drop cascades to table custom_domain
drop cascades to table lifetime_coupon
drop cascades to table directory
drop cascades to table job
drop cascades to table mailbox
drop cascades to table manual_subscription
drop cascades to table social_auth
drop cascades to table account_activation
drop cascades to table refused_email
drop cascades to table referral
drop cascades to type planenum_apple
drop cascades to table apple_subscription
drop cascades to table sent_alert
drop cascades to table alias_mailbox
drop cascades to table recovery_code
drop cascades to table domain_deleted_alias
drop cascades to table notification
drop cascades to table fido
drop cascades to table mfa_browser
drop cascades to table directory_mailbox
drop cascades to table public_domain
drop cascades to table domain_mailbox
drop cascades to table monitoring
drop cascades to table batch_import
drop cascades to table authorized_address
drop cascades to table coinbase_subscription
drop cascades to table bounce
drop cascades to table transactional_email
drop cascades to table metric2
drop cascades to table payout
drop cascades to table hibp
drop cascades to table alias_hibp
drop cascades to table ignored_email
drop cascades to table coupon
drop cascades to table hibp_notified_alias
drop cascades to table ignore_bounce_sender
drop cascades to extension pg_trgm
drop cascades to table auto_create_rule
drop cascades to table auto_create_rule__mailbox
drop cascades to table message_id_matching
drop cascades to table deleted_directory
drop cascades to table deleted_subdomain
drop cascades to table phone_country
drop cascades to table phone_number
drop cascades to table phone_message
drop cascades to table phone_reservation
drop cascades to table invalid_mailbox_domain
drop cascades to type block_behaviour_enum
drop cascades to table admin_audit_log
drop cascades to table provider_complaint
drop cascades to table partner
drop cascades to table partner_api_token
drop cascades to table partner_user
drop cascades to table partner_subscription
drop cascades to table newsletter
drop cascades to table newsletter_user
drop cascades to table api_cookie_token
drop cascades to table daily_metric
drop cascades to table sync_event
drop cascades to table mailbox_activation
drop cascades to table alias_audit_log
drop cascades to table user_audit_log
CREATE SCHEMA

$ docker exec sl-postgres psql -U myuser -d simplelogin -c '\dt'
Did not find any relations.

$ docker exec sl-postgres psql -U myuser -d simplelogin -c 'SELECT count(*) FROM users;'
ERROR:  relation "users" does not exist
LINE 1: SELECT count(*) FROM users;
                             ^

$ docker exec sl-postgres psql -U myuser -d simplelogin -tc \
    "SELECT count(*) FROM information_schema.tables WHERE table_schema='public';"
     0
```

(This drop is run only when the schema is non-empty; on an already-empty database the `drop schema public cascade` emits no `NOTICE`/`DETAIL` and just prints `CREATE SCHEMA`. The object count depends on the prior state — here 82 objects from a fully-migrated schema.)

### 2.3 Commands — start the server and exercise the login page

Start the web app through its real entry point (`server.py`'s `__main__` at `server.py:598` → `local_main()` at `server.py:572` → `app.debug = True` at `server.py:581` → `app.run(debug=True, port=7777)` at `server.py:588`; `create_app()` at `server.py:139` sets `SQLALCHEMY_DATABASE_URI = DB_URI` at `server.py:146`):

```
$ docker exec -w /app sl-app bash -c \
    'PYTHONUNBUFFERED=1 setsid /app/venv/bin/python server.py > /tmp/q1_server.log 2>&1 < /dev/null &'
```

Complete captured startup output (`/tmp/q1_server.log`, lines 1–19). In debug mode the Werkzeug reloader spawns two processes — the reloader parent (PID `2844`, which prints the Flask CLI banner) and the worker child (PID `2864`, which actually serves requests) — so the app-init preamble appears twice:

```
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/bpeutzaklskwivzmfulq
Upload files to local dir
>>> init logging <<<
2026-07-08 22:14:35,172 - SL - DEBUG - 2844 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/dhpcxmerhdzodskjmxhv
Upload files to local dir
>>> init logging <<<
2026-07-08 22:14:36,924 - SL - DEBUG - 2864 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

**Opening the login page (GET) — works, no DB hit.** A bare `GET /auth/login` renders statically; the route is `@auth_bp.route("/login", methods=["GET", "POST"])` (`app/auth/views/login.py:21`) → `def login()` (`:25`), and the first DB access is guarded by `if form.validate_on_submit():` (`:40`), which is false for GET:

```
$ docker exec sl-app curl -sS -c /tmp/q1_cookies.txt \
    -w '\n[HTTP_CODE=%{http_code}] [SIZE=%{size_download}]\n' http://127.0.0.1:7777/auth/login -o /tmp/q1_login_get.html
[HTTP_CODE=200] [SIZE=347619]

# server log confirms a clean 200 with no traceback (captured from /tmp/q1_server.log):
2026-07-08 22:14:52,762 - SL - DEBUG - 2864 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.10350561141967773
```

**Submitting the login form (POST) — triggers the first ORM query.** The POST carries the session cookie from the GET plus the CSRF token and credentials:

```
$ CSRF=$(grep -oP 'name="csrf_token"[^>]*value="\K[^"]+' /tmp/q1_login_get.html | head -1)
$ docker exec sl-app curl -sS -b /tmp/q1_cookies.txt -D /tmp/q1_post_headers.txt \
    --data-urlencode "csrf_token=${CSRF}" \
    --data-urlencode 'email=john@wick.com' \
    --data-urlencode 'password=password' \
    http://127.0.0.1:7777/auth/login -o /tmp/q1_post_body.html

# complete captured response headers (/tmp/q1_post_headers.txt):
HTTP/1.0 500 INTERNAL SERVER ERROR
Content-Type: text/html; charset=utf-8
Content-Length: 5749
Vary: Cookie
Set-Cookie: slapp=eyJfZnJlc2giOmZhbHNlLCJfcGVybWFuZW50Ijp0cnVlLCJjc3JmX3Rva2VuIjoiNWQ4NGU0MGIxMjYwNjZlOTkzNTZjNDI5YmUyODNlODM4MDA4OTZlZiJ9.ak7L3Q.6z2HQTP6KJB4tn9pB_ui6Pnqbho; Expires=Wed, 15-Jul-2026 22:14:53 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Wed, 08 Jul 2026 22:14:53 GMT
```

### 2.4 The full, unedited exception + traceback

Captured verbatim from the server's stderr (`/tmp/q1_server.log`; printed by `LOG.e(e)` inside the global error handler at `server.py:390`). This is the complete, unedited traceback — the full `SELECT` column list is shown in full (no ellipsis), exactly as emitted:

```
2026-07-08 22:14:53,167 - SL - ERROR - 2864 - "/app/server.py:390" - error_handler() -  - (psycopg2.errors.UndefinedTable) relation "users" does not exist
LINE 2: FROM users 
             ^

[SQL: SELECT users.directory_quota AS users_directory_quota, users.subdomain_quota AS users_subdomain_quota, users.password AS users_password, users.id AS users_id, users.created_at AS users_created_at, users.updated_at AS users_updated_at, users.email AS users_email, users.name AS users_name, users.is_admin AS users_is_admin, users.alias_generator AS users_alias_generator, users.notification AS users_notification, users.activated AS users_activated, users.disabled AS users_disabled, users.profile_picture_id AS users_profile_picture_id, users.otp_secret AS users_otp_secret, users.enable_otp AS users_enable_otp, users.last_otp AS users_last_otp, users.fido_uuid AS users_fido_uuid, users.default_alias_custom_domain_id AS users_default_alias_custom_domain_id, users.default_alias_public_domain_id AS users_default_alias_public_domain_id, users.lifetime AS users_lifetime, users.paid_lifetime AS users_paid_lifetime, users.lifetime_coupon_id AS users_lifetime_coupon_id, users.trial_end AS users_trial_end, users.default_mailbox_id AS users_default_mailbox_id, users.sender_format AS users_sender_format, users.sender_format_updated_at AS users_sender_format_updated_at, users.replace_reverse_alias AS users_replace_reverse_alias, users.referral_id AS users_referral_id, users.intro_shown AS users_intro_shown, users.max_spam_score AS users_max_spam_score, users.newsletter_alias_id AS users_newsletter_alias_id, users.include_sender_in_reverse_alias AS users_include_sender_in_reverse_alias, users.random_alias_suffix AS users_random_alias_suffix, users.expand_alias_info AS users_expand_alias_info, users.ignore_loop_email AS users_ignore_loop_email, users.alternative_id AS users_alternative_id, users.disable_automatic_alias_note AS users_disable_automatic_alias_note, users.one_click_unsubscribe_block_sender AS users_one_click_unsubscribe_block_sender, users.include_website_in_one_click_alias AS users_include_website_in_one_click_alias, users.disable_import AS users_disable_import, users.can_use_phone AS users_can_use_phone, users.phone_quota AS users_phone_quota, users.block_behaviour AS users_block_behaviour, users.include_header_email_header AS users_include_header_email_header, users.enable_data_breach_check AS users_enable_data_breach_check, users.flags AS users_flags, users.unsub_behaviour AS users_unsub_behaviour, users.delete_on AS users_delete_on 
FROM users 
WHERE users.email = %(email_1)s 
 LIMIT %(param_1)s]
[parameters: {'email_1': 'john@wick.com', 'param_1': 1}]
(Background on this error at: http://sqlalche.me/e/13/f405)
Traceback (most recent call last):
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1276, in _execute_context
    self.dialect.do_execute(
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/default.py", line 608, in do_execute
    cursor.execute(statement, parameters)
psycopg2.errors.UndefinedTable: relation "users" does not exist
LINE 2: FROM users 
             ^


The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "/app/venv/lib/python3.10/site-packages/flask/app.py", line 1950, in full_dispatch_request
    rv = self.dispatch_request()
  File "/app/venv/lib/python3.10/site-packages/flask_debugtoolbar/__init__.py", line 125, in dispatch_request
    return view_func(**req.view_args)
  File "/usr/local/lib/python3.10/cProfile.py", line 110, in runcall
    return func(*args, **kw)
  File "/app/venv/lib/python3.10/site-packages/flask_limiter/extension.py", line 702, in __inner
    return obj(*a, **k)
  File "/app/app/auth/views/login.py", line 43, in login
    user = User.get_by(email=email) or User.get_by(email=canonical_email)
  File "/app/app/models.py", line 84, in get_by
    return Session.query(cls).filter_by(**kw).first()
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/orm/query.py", line 3429, in first
    ret = list(self[0:1])
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/orm/query.py", line 3203, in __getitem__
    return list(res)
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/orm/query.py", line 3535, in __iter__
    return self._execute_and_instances(context)
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/orm/query.py", line 3560, in _execute_and_instances
    result = conn.execute(querycontext.statement, self._params)
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1011, in execute
    return meth(self, multiparams, params)
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/sql/elements.py", line 298, in _execute_on_connection
    return connection._execute_clauseelement(self, multiparams, params)
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1124, in _execute_clauseelement
    ret = self._execute_context(
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1316, in _execute_context
    self._handle_dbapi_exception(
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1510, in _handle_dbapi_exception
    util.raise_(
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/util/compat.py", line 182, in raise_
    raise exception
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1276, in _execute_context
    self.dialect.do_execute(
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/default.py", line 608, in do_execute
    cursor.execute(statement, parameters)
sqlalchemy.exc.ProgrammingError: (psycopg2.errors.UndefinedTable) relation "users" does not exist
LINE 2: FROM users 
             ^
[SQL: SELECT users.directory_quota AS users_directory_quota, users.subdomain_quota AS users_subdomain_quota, users.password AS users_password, users.id AS users_id, users.created_at AS users_created_at, users.updated_at AS users_updated_at, users.email AS users_email, users.name AS users_name, users.is_admin AS users_is_admin, users.alias_generator AS users_alias_generator, users.notification AS users_notification, users.activated AS users_activated, users.disabled AS users_disabled, users.profile_picture_id AS users_profile_picture_id, users.otp_secret AS users_otp_secret, users.enable_otp AS users_enable_otp, users.last_otp AS users_last_otp, users.fido_uuid AS users_fido_uuid, users.default_alias_custom_domain_id AS users_default_alias_custom_domain_id, users.default_alias_public_domain_id AS users_default_alias_public_domain_id, users.lifetime AS users_lifetime, users.paid_lifetime AS users_paid_lifetime, users.lifetime_coupon_id AS users_lifetime_coupon_id, users.trial_end AS users_trial_end, users.default_mailbox_id AS users_default_mailbox_id, users.sender_format AS users_sender_format, users.sender_format_updated_at AS users_sender_format_updated_at, users.replace_reverse_alias AS users_replace_reverse_alias, users.referral_id AS users_referral_id, users.intro_shown AS users_intro_shown, users.max_spam_score AS users_max_spam_score, users.newsletter_alias_id AS users_newsletter_alias_id, users.include_sender_in_reverse_alias AS users_include_sender_in_reverse_alias, users.random_alias_suffix AS users_random_alias_suffix, users.expand_alias_info AS users_expand_alias_info, users.ignore_loop_email AS users_ignore_loop_email, users.alternative_id AS users_alternative_id, users.disable_automatic_alias_note AS users_disable_automatic_alias_note, users.one_click_unsubscribe_block_sender AS users_one_click_unsubscribe_block_sender, users.include_website_in_one_click_alias AS users_include_website_in_one_click_alias, users.disable_import AS users_disable_import, users.can_use_phone AS users_can_use_phone, users.phone_quota AS users_phone_quota, users.block_behaviour AS users_block_behaviour, users.include_header_email_header AS users_include_header_email_header, users.enable_data_breach_check AS users_enable_data_breach_check, users.flags AS users_flags, users.unsub_behaviour AS users_unsub_behaviour, users.delete_on AS users_delete_on 
FROM users 
WHERE users.email = %(email_1)s 
 LIMIT %(param_1)s]
[parameters: {'email_1': 'john@wick.com', 'param_1': 1}]
(Background on this error at: http://sqlalche.me/e/13/f405)
```

The request-completion line logged immediately after (by `after_request()` at `server.py:284`) confirms the 500 response:

```
2026-07-08 22:14:53,173 - SL - DEBUG - 2864 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 500, takes 0.014399051666259766
```

(The `SELECT` above lists **all 49 `User` columns in full** — the earlier `...` abbreviation has been removed and the complete column list is transcribed exactly as emitted, in both the initial DBAPI error and the wrapping `sqlalchemy.exc.ProgrammingError`. The two `LINE 2: FROM users` carets are emitted verbatim by Postgres.)

### 2.5 Render-path observation (what the browser actually gets)

Even though the server runs with `app.run(debug=True)` / `app.debug = True`, the client does **not** receive the Werkzeug interactive debugger. The app registers a catch-all `@app.errorhandler(Exception)` at `server.py:388` → `error_handler(e)` (`:389`) which calls `LOG.e(e)` (`:390`, the source of the traceback above) and then, for non-`/api/` paths, `return render_template("error/500.html"), 500` (`:394`). Observed:

- Response status: **`HTTP/1.0 500 INTERNAL SERVER ERROR`**, `Content-Length: 5749`.
- Response body is the custom template `templates/error/500.html` (which `{% extends "error.html" %}`), showing the visible text **"Server error"** and **"Looks like we are having some server issues... We are notified and will look at this issue asap!"**.
- A scan of the body for Werkzeug interactive-debugger markers (`werkzeug`, `traceback`, `console`, `__debugger__`, `pin`) returned **0** matches.

So: the login page opens fine (200); the failure surfaces on form submit as a 500 whose body is the app's own error page, while the complete Python traceback is written to the server console/stderr.

### 2.6 Repeatability

The observation was repeated with three fresh `GET → POST` cycles against the running server (each obtaining its own session cookie + CSRF token from the GET), capturing the HTTP status and the terminal exception-class line each POST produced in `/tmp/q1_server.log`. Exact command and captured output:

```
$ docker exec sl-app bash -c '
for i in 1 2 3; do
  curl -sS -c /tmp/q1_rep_cookies_$i.txt http://127.0.0.1:7777/auth/login -o /tmp/q1_rep_get_$i.html
  CSRF=$(grep -oP "name=\"csrf_token\"[^>]*value=\"\K[^\"]+" /tmp/q1_rep_get_$i.html | head -1)
  before=$(wc -l < /tmp/q1_server.log)
  code=$(curl -sS -b /tmp/q1_rep_cookies_$i.txt -o /dev/null -w "%{http_code}" \
     --data-urlencode "csrf_token=$CSRF" --data-urlencode "email=john@wick.com" \
     --data-urlencode "password=password" http://127.0.0.1:7777/auth/login)
  excline=$(tail -n +$((before+1)) /tmp/q1_server.log | grep -m1 "^sqlalchemy.exc")
  echo "run $i: HTTP=$code | $excline"
done'
run 1: HTTP=500 | sqlalchemy.exc.ProgrammingError: (psycopg2.errors.UndefinedTable) relation "users" does not exist
run 2: HTTP=500 | sqlalchemy.exc.ProgrammingError: (psycopg2.errors.UndefinedTable) relation "users" does not exist
run 3: HTTP=500 | sqlalchemy.exc.ProgrammingError: (psycopg2.errors.UndefinedTable) relation "users" does not exist
```

All three cycles returned **HTTP 500** with the byte-identical `sqlalchemy.exc.ProgrammingError: (psycopg2.errors.UndefinedTable) relation "users" does not exist`, logged via `error_handler()` at `server.py:390`. The result is byte-stable across runs.

### 2.7 Reasoning

Schema creation is exclusively Alembic's responsibility; `create_all` is absent from `server.py` and all of `app/` (`grep -rn "create_all" server.py app/` → 0 matches, §1.5). With migrations skipped, the empty schema has no `users` relation (`class User` at `app/models.py:336`, `__tablename__ = "users"` at `app/models.py:337`). A bare `GET /auth/login` renders statically and never queries the DB, so "opening the login page" itself succeeds. The first ORM query — `User.get_by(email=...)` at `app/auth/views/login.py:43`, run by `ModelMixin.get_by` at `app/models.py:84` — fires only on form submission (`validate_on_submit()` at `:40`), strikes the non-existent table, and `psycopg2` raises `UndefinedTable`, which SQLAlchemy 1.3.24 re-raises as `sqlalchemy.exc.ProgrammingError`. (An authenticated session cookie is an alternate trigger: the `@login_manager.user_loader` `load_user` at `server.py:221` runs `User.get_by(alternative_id=...)` on any request — **(inferred)**, not exercised here because obtaining such a cookie itself requires the DB.)

---

## 3. Q2 — Required services and their readiness output

> **Question (verbatim):** "Once the database migrations and the initialization script (`init_app.py`) have been run and everything is set up correctly, I need to understand which Python services must be running for the system to work. Start each required service and show me the exact log output that confirms it's running and ready to accept connections. I want to see the actual startup messages from stdout/stderr... I need to verify the services are actually binding to their ports and working."

### 3.1 Prerequisite — reach the fully-initialized state

The "fully set up" state is reached by running migrations then the initialization script:

```
$ docker exec -w /app sl-app /app/venv/bin/alembic upgrade head    # 255 versioned scripts
$ docker exec -w /app sl-app /app/venv/bin/python init_app.py
```

`init_app.py`'s `add_sl_domains()` (`init_app.py:39`) seeds the `SLDomain`/`public_domain` table from `ALIAS_DOMAINS`, logging `LOG.i("Add %s to SL domain", ...)` at `init_app.py:44`; `__main__` (`init_app.py:69`) runs it inside `create_light_app().app_context()` (`init_app.py:71`). Captured first-run output (`init_app.py:44`, emitted via `LOG.i`):

```
2026-07-08 22:25:25,615 - SL - INFO - 3285 - "/app/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain
```

On a subsequent run it is idempotent (`init_app.py:42`, emitted via `LOG.d`):

```
2026-07-08 22:25:41,609 - SL - DEBUG - 3315 - "/app/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
```

State proof (before = 0 from Q3's migrated-but-uninitialized DB; after ≥ 1):

```
$ docker exec sl-postgres psql -U myuser -d simplelogin -c 'SELECT id, domain FROM public_domain;'
 id | domain
----+----------
  1 | sl.local
(1 row)
```

**A note on how "ready to accept connections" manifests here.** All processes log single-line records to stdout via `app/log.py` (format at `app/log.py:12-14`). SimpleLogin **intentionally disables the werkzeug logger** — `app/log.py:69-71`:

```
# Disable flask logs such as 127.0.0.1 - - [15/Feb/2013 10:52:22] "GET /index.html HTTP/1.1" 200
log = logging.getLogger("werkzeug")
log.disabled = True
```

so werkzeug's usual `* Running on http://127.0.0.1:7777` line **never appears**. The readiness signal for the web app is therefore the Flask CLI banner plus a successful `/health` probe (below), not a "Running on" line.

### 3.2 ESSENTIAL services (required for basic operation)

#### 3.2.1 Flask web app — `python server.py`, binds TCP **7777**

Start command and **complete** captured stdout. Debug mode spawns a reloader parent (PID 3380) which prints the Flask CLI banner and then forks the serving worker (PID 3400), so the app-init preamble prints twice; the trailing `flask_debugtoolbar` `UserWarning` is emitted the first time a non-HTML response (the `/health` probe below) is served:

```
$ docker exec -w /app sl-app bash -c \
    'PYTHONUNBUFFERED=1 setsid /app/venv/bin/python server.py > /tmp/q2_web.log 2>&1 < /dev/null &'
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/xwxkylmpbstgcmzpnwdp
Upload files to local dir
>>> init logging <<<
2026-07-08 22:30:10,673 - SL - DEBUG - 3380 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/klcwifjsaqbysobwhvhk
Upload files to local dir
>>> init logging <<<
2026-07-08 22:30:12,448 - SL - DEBUG - 3400 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
/app/venv/lib/python3.10/site-packages/flask_debugtoolbar/__init__.py:213: UserWarning: Could not insert debug toolbar. </body> tag not found in response.
  warnings.warn('Could not insert debug toolbar.'
```

Binding proof — because the werkzeug "Running on" line is suppressed (§3.1), the authoritative proof is a live `/health` request (route wired in `create_app`, returns `"success"`):

```
$ docker exec sl-app curl -sS -w ' [code=%{http_code}]\n' http://127.0.0.1:7777/health
success [code=200]
```

(`grep -c 'Running on' /tmp/q2_web.log` → `0`, confirming the suppression.)

**Production alternative:** dev uses `python server.py`; production uses Gunicorn — `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` (`Dockerfile:47`; `EXPOSE 7777` at `Dockerfile:44`). `wsgi.py` is simply `from server import create_app` (`wsgi.py:1`) + `app = create_app()` (`wsgi.py:3`) — the same `create_app()` factory the dev server uses, so the enumerated web service is identical under Gunicorn.

#### 3.2.2 Email handler — `python email_handler.py`, binds TCP **20381** (dev)

Start command and captured **readiness banner** (byte-accurate). This is the single handler instance (PID 3139) used for both this Q2 enumeration and the Q3 injection in §4.3, so the banner timestamps match §4:

```
$ docker exec -w /app sl-app bash -c \
    'PYTHONUNBUFFERED=1 setsid /app/venv/bin/python email_handler.py > /tmp/q3_email.log 2>&1 < /dev/null &'

2026-07-08 22:23:28,587 - SL - INFO - 3139 - "/app/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-08 22:23:28,589 - SL - DEBUG - 3139 - "/app/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

These two lines correspond to `LOG.i("Listen for port %s", args.port)` at `email_handler.py:2403` and `LOG.d("Start mail controller %s %s", controller.hostname, controller.port)` at `email_handler.py:2386`. Wiring: `def main(port)` at `email_handler.py:2381`; `controller = Controller(MailHandler(), hostname="0.0.0.0", port=port)` at `email_handler.py:2383`; argparse `default=20381` at `email_handler.py:2399`; `main(port=args.port)` at `email_handler.py:2404`.

Binding proof (`ss`/iproute2 is not installed in the container, so a TCP connect via `/dev/tcp` is the reliable proof):

```
$ docker exec sl-app bash -c 'timeout 3 bash -c "echo > /dev/tcp/127.0.0.1/20381" && echo "PORT 20381 ACCEPTS CONNECTIONS"'
PORT 20381 ACCEPTS CONNECTIONS
```

**Dev-vs-prod port:** dev binds **20381** (`email_handler.py:2399`); production binds the privileged port **25** (see §5).

#### 3.2.3 PostgreSQL — TCP **5432**

Readiness = accepting connections:

```
$ docker exec sl-postgres pg_isready -h 127.0.0.1 -p 5432 -U myuser -d simplelogin
127.0.0.1:5432 - accepting connections
```

#### 3.2.4 Redis — TCP **6379**

Readiness = `PONG`:

```
$ docker exec sl-redis redis-cli ping
PONG
```

**Honest finding — the web app does NOT connect to Redis in the canonical (`example.env`) configuration.** `MEM_STORE_URI` is read at `app/config.py:568` (`os.environ.get("MEM_STORE_URI", None)`) and is **unset** in the `.env` (derived from `example.env`). In `create_app`, both the Flask-Limiter storage URL and `initialize_redis_services()` are gated behind `if MEM_STORE_URI:` at `server.py:163` (`server.py:164-165`), so with it unset the limiter falls back to in-memory storage and sessions are cookie-based. Observed proof — with the web app running, Redis reports only our own `redis-cli` as a client:

```
$ docker exec -w /app sl-app bash -c 'grep -c "^MEM_STORE_URI=" .env; echo "MEM_STORE_URI=${MEM_STORE_URI:-<unset>}"'
0
MEM_STORE_URI=<unset>

$ docker exec sl-redis redis-cli info clients | grep connected_clients
connected_clients:1
```

Redis is therefore listed as an essential piece of the *documented* production topology (sessions / Flask-Limiter rate-limiting — `CONTRIBUTING.md:65`), but in the default local-dev config the web app does not open a Redis connection. This is reported honestly rather than asserted from the docs.

### 3.3 AUXILIARY services (background workers — bind NO TCP port)

For these, "readiness" is an app-initialization log line and entry into a work loop — none binds a listening TCP socket. **Proof method:** count listening sockets (TCP state `0A`) across both IPv4 and IPv6 via `/proc/net/tcp{,6}`, before and after starting each process. Captured baseline with the two essential Python servers (web app + email handler) running:

```
$ docker exec sl-app bash -c "cat /proc/net/tcp /proc/net/tcp6 | awk '\$4==\"0A\"' | wc -l"
7
$ docker exec sl-app bash -c "cat /proc/net/tcp /proc/net/tcp6 | awk '\$4==\"0A\" {print \$2}' | while read a; do p=\$(echo \$a|cut -d: -f2); printf '%d ' 0x\$p; done; echo"
15432 5432 6379 7777 20381 5432 6379
```

The 7 sockets decode to PostgreSQL `5432` and Redis `6379` (each listening on **both** IPv4 and IPv6 → 2 sockets apiece), the web app `7777`, the email handler `20381`, and the pytest-only test DB `15432` (IPv4-only). Starting each auxiliary process below left this count **unchanged at 7**, proving none binds a new port.

#### 3.3.1 `job_runner.py` — `python job_runner.py` (poller; no TCP port)

`__main__` at `job_runner.py:329` is a `while True:` loop (`:330`) that opens `create_light_app().app_context()` (`:332`), logs `LOG.d("Take job %s", job)` (`:334`) only when a job exists, and `time.sleep(10)` (`:347`) between passes. It is a poller, not a server. Exact start command and **complete** captured stdout (with an empty job queue the readiness output is the app-init block ending at the `load words file` line, after which it enters the poll loop):

```
$ docker exec -w /app sl-app bash -c \
    'PYTHONUNBUFFERED=1 setsid /app/venv/bin/python job_runner.py > /tmp/q2_jr.log 2>&1 < /dev/null & echo $!'
3892
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/qpzrlieyqfrfoblciymc
Upload files to local dir
>>> init logging <<<
2026-07-08 22:38:05,995 - SL - DEBUG - 3892 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

No-port proof — the listening-socket count is unchanged by this start:

```
$ docker exec sl-app bash -c "cat /proc/net/tcp /proc/net/tcp6 | awk '\$4==\"0A\"' | wc -l"   # before
7
$ docker exec sl-app bash -c "cat /proc/net/tcp /proc/net/tcp6 | awk '\$4==\"0A\"' | wc -l"   # after starting PID 3892
7
```

#### 3.3.2 `cron.py` — `python cron.py -j <job>` (one-shot; no TCP port)

`cron.py` is argparse-driven (`-j/--job`) and invoked per-job by yacron (`crontab.yml`); it runs a task and **exits** (it is not a daemon and binds no port). Exact invocation and **complete, unedited** output for the `sanity_check` job:

```
$ docker exec -w /app sl-app bash -c \
    'PYTHONUNBUFFERED=1 /app/venv/bin/python cron.py -j sanity_check' ; echo "exit=$?"
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/giqztffvosnqbvjrodnc
Upload files to local dir
>>> init logging <<<
2026-07-08 22:37:46,231 - SL - DEBUG - 3839 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-08 22:37:47,128 - SL - DEBUG - 3839 - "/app/cron.py:1263" - <module>() -  - Start running cronjob
2026-07-08 22:37:47,130 - SL - DEBUG - 3839 - "/app/cron.py:1296" - <module>() -  - Check data consistency
2026-07-08 22:37:47,130 - SL - DEBUG - 3839 - "/app/cron.py:725" - sanity_check() -  - sanitize user email
2026-07-08 22:37:47,260 - SL - DEBUG - 3839 - "/app/cron.py:730" - sanity_check() -  - sanitize alias address & name
2026-07-08 22:37:47,263 - SL - DEBUG - 3839 - "/app/cron.py:733" - sanity_check() -  - sanity contact address
2026-07-08 22:37:47,267 - SL - DEBUG - 3839 - "/app/cron.py:751" - sanity_check() -  - sanitize mailbox address
2026-07-08 22:37:47,268 - SL - DEBUG - 3839 - "/app/cron.py:756" - sanity_check() -  - normalize reverse alias
2026-07-08 22:37:47,269 - SL - DEBUG - 3839 - "/app/cron.py:765" - sanity_check() -  - clean domain name
2026-07-08 22:37:47,271 - SL - DEBUG - 3839 - "/app/cron.py:770" - sanity_check() -  - migrate domain trash if needed
2026-07-08 22:37:47,273 - SL - DEBUG - 3839 - "/app/cron.py:677" - migrate_domain_trash() -  - create 0 DomainDeletedAlias
2026-07-08 22:37:47,273 - SL - DEBUG - 3839 - "/app/cron.py:680" - migrate_domain_trash() -  - delete 0 DeletedAlias
2026-07-08 22:37:47,274 - SL - DEBUG - 3839 - "/app/cron.py:773" - sanity_check() -  - fix custom domain for alias
2026-07-08 22:37:47,276 - SL - DEBUG - 3839 - "/app/cron.py:776" - sanity_check() -  - check mailbox valid domain
2026-07-08 22:37:47,276 - SL - DEBUG - 3839 - "/app/cron.py:779" - sanity_check() -  - check mailbox valid PGP keys
2026-07-08 22:37:47,277 - SL - DEBUG - 3839 - "/app/cron.py:782" - sanity_check() -  - check if there's an email that starts with "‏" (right-to-left mark (RLM))
2026-07-08 22:37:47,278 - SL - DEBUG - 3839 - "/app/cron.py:794" - sanity_check() -  - Finish sanity check
exit=0
```

Every line is emitted from a self-identifying `cron.py:<lineno>` `LOG` statement; `sanity_check()` walks its consistency checks (`cron.py:725`–`cron.py:782`), the `migrate_domain_trash()` helper reports `create 0`/`delete 0` (empty DB), and it finishes at `cron.py:794`. The process then exits `0` — it binds no port, so it is never counted among the 7 listening sockets. (The `cron.py:782` line contains a literal U+200F right-to-left mark between the double quotes, reproduced verbatim.)

#### 3.3.3 `event_listener.py` — `python event_listener.py listener` (LISTEN/NOTIFY consumer; no TCP port)

`def main(mode, dry_run, max_retries)` at `event_listener.py:29`; `__main__` at `event_listener.py:94`. It consumes PostgreSQL LISTEN/NOTIFY and binds no TCP port. It uses `EVENT_LISTENER_DB_URI`, which defaults to `DB_URI` (`app/config.py:637`). Exact start command and **complete** captured stdout (default, non-dry-run):

```
$ docker exec -w /app sl-app bash -c \
    'PYTHONUNBUFFERED=1 setsid /app/venv/bin/python event_listener.py listener > /tmp/q2_ev.log 2>&1 < /dev/null & echo $!'
3976
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/wbpvlnctwyzvibwurqfl
Upload files to local dir
>>> init logging <<<
2026-07-08 22:38:29,200 - SL - DEBUG - 3976 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-08 22:38:29,319 - SL - INFO - 3976 - "/app/event_listener.py:34" - main() -  - Using PostgresEventSource
2026-07-08 22:38:29,322 - SL - INFO - 3976 - "/app/event_listener.py:43" - main() -  - Starting with HttpEventSink
2026-07-08 22:38:29,322 - SL - INFO - 3976 - "/app/events/event_source.py:49" - __listen() -  - Starting to listen to events
```

No-port proof — socket count unchanged by this start:

```
$ docker exec sl-app bash -c "cat /proc/net/tcp /proc/net/tcp6 | awk '\$4==\"0A\"' | wc -l"   # after starting PID 3976
7
```

**Honest correction vs. the inferred banner.** The default sink is **`HttpEventSink`** (`event_listener.py:43`, assigned at `:44`), *not* `ConsoleEventSink`. `ConsoleEventSink` (logged at `event_listener.py:40`) is only selected under `if dry_run:` (`event_listener.py:39`). Exact command and **complete** captured stdout with the flag:

```
$ docker exec -w /app sl-app bash -c \
    'PYTHONUNBUFFERED=1 setsid /app/venv/bin/python event_listener.py listener --dry-run > /tmp/q2_ev_dry.log 2>&1 < /dev/null & echo $!'
3582
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/yxxxyaqicxyiqjjuprmo
Upload files to local dir
>>> init logging <<<
2026-07-08 22:31:42,084 - SL - DEBUG - 3582 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-08 22:31:42,201 - SL - INFO - 3582 - "/app/event_listener.py:34" - main() -  - Using PostgresEventSource
2026-07-08 22:31:42,204 - SL - INFO - 3582 - "/app/event_listener.py:40" - main() -  - Starting with ConsoleEventSink
2026-07-08 22:31:42,205 - SL - INFO - 3582 - "/app/events/event_source.py:49" - __listen() -  - Starting to listen to events
```

Only the sink line differs between default (`HttpEventSink`, `:43`) and `--dry-run` (`ConsoleEventSink`, `:40`); both then enter `events/event_source.py:49` `__listen()` and begin listening.

### 3.4 Essential-vs-auxiliary rationale

The **web app** (accepts HTTP on 7777) and the **email handler** (accepts SMTP on 20381) are the two Python processes that actually *accept connections*, and each needs **PostgreSQL** (all ORM access) to function; **Redis** completes the documented production picture for sessions/rate-limiting (though unused by the web app in the default dev config — §3.2.4). These four constitute basic operation. `CONTRIBUTING.md:145-147` names the process roles, e.g. `email_handler.py: the email handler.` at `CONTRIBUTING.md:146`.

The **auxiliary** processes — `job_runner.py`, `cron.py`, `event_listener.py` — perform deferred/background processing (job queue, scheduled maintenance, event fan-out). They bind no port and are not required for the core request/mail flow to accept connections; the system will serve HTTP and receive/route mail without them, merely deferring background work.

### 3.5 Repeatability

Every service was started/probed **at least twice** and the readiness evidence was byte-stable — only per-process values (timestamp, PID, and the random `GNUPGHOME` temp-dir name) differ. Captured runs:

**Essential services (2 probes each) — all stable:**

```
$ for r in 1 2; do \
    docker exec sl-app curl -sS -w ' [code=%{http_code}]\n' http://127.0.0.1:7777/health; \
    docker exec sl-postgres pg_isready -h 127.0.0.1 -p 5432 -U myuser -d simplelogin; \
    docker exec sl-redis redis-cli ping; done
success [code=200]
127.0.0.1:5432 - accepting connections
PONG
success [code=200]
127.0.0.1:5432 - accepting connections
PONG
```

**Email handler startup (2 restarts, PIDs 4066 then 4099)** — both emit the identical two-line banner (modulo timestamp/PID) and both accept a TCP connection on 20381:

```
run #1 (PID 4066):
2026-07-08 22:39:45,889 - SL - INFO - 4066 - "/app/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-08 22:39:45,891 - SL - DEBUG - 4066 - "/app/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
run #2 (PID 4099):
2026-07-08 22:39:50,298 - SL - INFO - 4099 - "/app/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-08 22:39:50,300 - SL - DEBUG - 4099 - "/app/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381

$ # diff of the two banners after masking timestamp+PID:
BANNERS IDENTICAL after stripping timestamp+PID
```

**`cron.py -j sanity_check` (2 runs)** — both produced 23 lines and `exit=0`; masking timestamp+PID, the only difference is the random `GNUPGHOME` path:

```
$ strip() { sed -E 's/^[0-9-]+ [0-9:,]+ - SL - ([A-Z]+) - [0-9]+ -/TS - SL - \1 - PID -/'; }
$ diff <(strip < run1.log) <(strip < run2.log)
4c4
< WARNING: Use a temp directory for GNUPGHOME /tmp/giqztffvosnqbvjrodnc
---
> WARNING: Use a temp directory for GNUPGHOME /tmp/jdyrudobsuqblptrbxic
```

**`job_runner.py` (2 restarts, PIDs 3892/3934) and `event_listener.py listener` (2 restarts, PIDs 3976/4014)** — each restart's listening-socket count was `before=7, after=7` (no port bound), and the app-init/readiness lines matched line-for-line (modulo timestamp/PID/GNUPGHOME).

---

## 4. Q3 — Unconfigured-domain rejection

> **Question (verbatim):** "If I run the migrations but don't run `init_app.py`, the `SLDomain` table should be empty (no email domains configured). If I then start the email handler and simulate an incoming email addressed to any `@sl.local` address, what happens? What SMTP status code does the email handler return to the sender, and what does it log to explain the rejection?"

### 4.1 Short answer

The email handler returns SMTP status **`550 SL E515 Email not exist`** to the sender, and logs the rejection line:

```
alias e1@sl.local cannot be created on-the-fly, return 550
```

emitted by `handle_forward()` at **`email_handler.py:551`**. The `550` string is the constant `E515 = "550 SL E515 Email not exist"` at `app/email/status.py:51`, returned as `status.E515` at `email_handler.py:555`.

### 4.2 BEFORE state — migrations run, `init_app.py` skipped, `public_domain` empty

Reach the migrated-but-uninitialized state (255 alembic scripts; **no** `init_app.py`, **no** `flask dummy-data`):

```
# migrations applied and captured in §1.4 (alembic upgrade head > /tmp/q3_alembic.log 2>&1)
$ docker exec sl-app grep -c "Running upgrade" /tmp/q3_alembic.log
255

$ docker exec sl-postgres psql -U myuser -d simplelogin -c 'SELECT count(*) FROM public_domain;'
 count 
-------
     0
(1 row)
```

`class SLDomain` (`app/models.py:3116`) maps to `__tablename__ = "public_domain"` (`app/models.py:3119`), so the empty count proves the `SLDomain` table has no rows. Running `init_app.py` *would* add `sl.local` because `ALIAS_DOMAINS` defaults to include `EMAIL_DOMAIN` — `app/config.py:160` (`ALIAS_DOMAINS = OTHER_ALIAS_DOMAINS + [EMAIL_DOMAIN]`, with `EMAIL_DOMAIN = "sl.local"` from `example.env:22`); skipping it leaves `public_domain` empty.

### 4.3 Start the email handler and inject via `swaks`

Email handler started as in §3.2.2 (readiness banner `Listen for port 20381` / `Start mail controller 0.0.0.0 20381`). Injection through the real SMTP entry point using `swaks` (`CONTRIBUTING.md:186`, command form at `CONTRIBUTING.md:218`):

```
$ docker exec sl-app swaks --to e1@sl.local --from hey@google.com --server 127.0.0.1:20381 ; echo "swaks exit=$?"
```

Full, unedited `swaks` transcript (the `<**` prefix is swaks marking a non-2xx/3xx reply; the greeting shows aiosmtpd `Python SMTP 1.4.2`):

```
=== Trying 127.0.0.1:20381...
=== Connected to 127.0.0.1.
<-  220 reverse-code-generator-64e0e612-hfqqw Python SMTP 1.4.2
 -> EHLO reverse-code-generator-64e0e612-hfqqw
<-  250-reverse-code-generator-64e0e612-hfqqw
<-  250-SIZE 33554432
<-  250-8BITMIME
<-  250-SMTPUTF8
<-  250 HELP
 -> MAIL FROM:<hey@google.com>
<-  250 OK
 -> RCPT TO:<e1@sl.local>
<-  250 OK
 -> DATA
<-  354 End data with <CR><LF>.<CR><LF>
 -> Date: Wed, 08 Jul 2026 22:24:15 +0000
 -> To: e1@sl.local
 -> From: hey@google.com
 -> Subject: test Wed, 08 Jul 2026 22:24:15 +0000
 -> Message-Id: <20260708222415.003212@reverse-code-generator-64e0e612-hfqqw>
 -> X-Mailer: swaks v20201014.0 jetmore.org/john/code/swaks/
 -> 
 -> This is a test mailing
 -> 
 -> 
 -> .
<** 550 SL E515 Email not exist
 -> QUIT
<-  221 Bye
=== Connection closed with remote host.

swaks exit=26
```

The **SMTP status code returned to the sender** is therefore, byte-accurate: **`550 SL E515 Email not exist`**.

### 4.4 What the handler logs to explain the rejection

Complete, unedited handler stdout for **this exact message** (`/tmp/q3_email.log`), captured as the delta produced by injection #1. Every line carries the real per-message id (a UUID, `email_handler.py:2339` `message_id = str(uuid.uuid4())`) — here `fe7348de-5653-484f-b744-2945ccdcf230` — set via `set_message_id()` (`app/log.py:22-25`); the `app/log.py:14` format self-identifies `"pathname:lineno" - funcName() - message_id - message`. No placeholders and no ellipsis; the elapsed time is the full float as emitted:

```
2026-07-08 22:24:15,037 - SL - DEBUG - 3139 - "/app/app/log.py:24" - set_message_id() -  - set message_id fe7348de-5653-484f-b744-2945ccdcf230
2026-07-08 22:24:15,037 - SL - DEBUG - 3139 - "/app/email_handler.py:2342" - _handle() - fe7348de-5653-484f-b744-2945ccdcf230 - ====>=====>====>====>====>====>====>====>
2026-07-08 22:24:15,037 - SL - INFO - 3139 - "/app/email_handler.py:2343" - _handle() - fe7348de-5653-484f-b744-2945ccdcf230 - New message, mail from hey@google.com, rctp tos ['e1@sl.local'] 
2026-07-08 22:24:15,038 - SL - INFO - 3139 - "/app/email_handler.py:1956" - handle() - fe7348de-5653-484f-b744-2945ccdcf230 - Set CONTENT_TRANSFER_ENCODING
2026-07-08 22:24:15,038 - SL - DEBUG - 3139 - "/app/email_handler.py:1963" - handle() - fe7348de-5653-484f-b744-2945ccdcf230 - Cannot parse Postfix queue ID from None None
2026-07-08 22:24:15,158 - SL - DEBUG - 3139 - "/app/email_handler.py:1980" - handle() - fe7348de-5653-484f-b744-2945ccdcf230 - ==>> Handle mail_from:hey@google.com, rcpt_tos:['e1@sl.local'], header_from:hey@google.com, header_to:e1@sl.local, cc:None, reply-to:None, message_id:<20260708222415.003212@reverse-code-generator-64e0e612-hfqqw>, client_ip:None, headers:[('Date', 'Wed, 08 Jul 2026 22:24:15 +0000'), ('To', 'e1@sl.local'), ('From', 'hey@google.com'), ('Subject', 'test Wed, 08 Jul 2026 22:24:15 +0000'), ('Message-Id', '<20260708222415.003212@reverse-code-generator-64e0e612-hfqqw>'), ('X-Mailer', 'swaks v20201014.0 jetmore.org/john/code/swaks/'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-08 22:24:15,162 - SL - DEBUG - 3139 - "/app/email_handler.py:2202" - handle() - fe7348de-5653-484f-b744-2945ccdcf230 - Forward phase hey@google.com(hey@google.com) -> e1@sl.local
2026-07-08 22:24:15,172 - SL - DEBUG - 3139 - "/app/email_handler.py:545" - handle_forward() - fe7348de-5653-484f-b744-2945ccdcf230 - alias e1@sl.local not exist. Try to see if it can be created on the fly
2026-07-08 22:24:15,181 - SL - INFO - 3139 - "/app/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - fe7348de-5653-484f-b744-2945ccdcf230 - Cannot auto-create custom domain alias for e1@sl.local because there's no custom domain for sl.local
2026-07-08 22:24:15,182 - SL - INFO - 3139 - "/app/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - fe7348de-5653-484f-b744-2945ccdcf230 - Cannot auto-create e1@sl.local since it has no directory separator
2026-07-08 22:24:15,182 - SL - DEBUG - 3139 - "/app/email_handler.py:551" - handle_forward() - fe7348de-5653-484f-b744-2945ccdcf230 - alias e1@sl.local cannot be created on-the-fly, return 550
2026-07-08 22:24:15,183 - SL - INFO - 3139 - "/app/email_handler.py:2367" - _handle() - fe7348de-5653-484f-b744-2945ccdcf230 - Finish mail_from hey@google.com, rcpt_tos ['e1@sl.local'], takes 0.14559197425842285 seconds with return code '550 SL E515 Email not exist'<<===
```

The primary rejection line is **`email_handler.py:551`** (`alias e1@sl.local cannot be created on-the-fly, return 550`); the corroborating final-status line at **`email_handler.py:2367`** records the full elapsed time (`0.14559197425842285` seconds) and the return code `'550 SL E515 Email not exist'`. The two intermediate `INFO` lines at `app/alias_utils.py:104` and `app/alias_utils.py:165` explain *why* neither on-the-fly path applies (no `CustomDomain` for `sl.local`; no directory separator in the local-part).

### 4.5 Decision chain (traced through the real code)

1. `aiosmtpd` invokes `MailHandler.handle_DATA` (`email_handler.py:2289`) → `self._handle(envelope, msg)` (`email_handler.py:2335`).
2. `_handle` logs "New message" (`email_handler.py:2343`) and computes `return_status = handle(envelope, msg)` (`email_handler.py:2353`).
3. `handle()` (`email_handler.py:1945`) enters the forward phase, iterating `for is_delivered, smtp_status in handle_forward(...)` (`email_handler.py:2208`) and, absent a success, returns `res[0][1]` (`email_handler.py:2233`).
4. `handle_forward()` (`email_handler.py:536`): `alias = Alias.get_by(email=alias_address)` misses (`email_handler.py:543`); logs the not-exist line (`:545`); calls `alias = try_auto_create(alias_address)` (`email_handler.py:549`; `app/alias_utils.py:202`).
5. `try_auto_create` (`app/alias_utils.py:202`) tries `try_auto_create_via_domain` (`app/alias_utils.py:274`) → `check_if_alias_can_be_auto_created_for_custom_domain` (`app/alias_utils.py:92`), which finds no `CustomDomain` for `sl.local` and logs at `app/alias_utils.py:104`; then `try_auto_create_directory` (`app/alias_utils.py:227`) → `check_if_alias_can_be_auto_created_for_a_directory` (`app/alias_utils.py:145`), which logs "no directory separator" at `app/alias_utils.py:165`. Both return `None`.
6. Back in `handle_forward`, `if not alias:` (`email_handler.py:550`) logs the rejection at `email_handler.py:551` and, since `should_ignore_bounce(envelope.mail_from)` is false for a normal sender (`email_handler.py:552`), executes `return [(False, status.E515)]` at `email_handler.py:555`. (The ignore-bounce branch would instead `return [(True, status.E207)]` at `email_handler.py:553`.)

### 4.6 SPF / E216 caveat — observed, not assumed

`_handle` contains an SPF override: `spamd_result = SpamdResult.extract_from_headers(msg)` (`email_handler.py:2356`), then `if return_status[0] == "5":` (`email_handler.py:2357`) and if `spamd_result` shows an SPF `fail`/`soft_fail`, the `550` is **replaced** by `E216 = "250 SL E216 Handled spf policy"` (`app/email/status.py:24`) via `return_status = status.E216` (`email_handler.py:2365`) and the mail is black-holed. For a plain `swaks` injection with no spamd/SPF headers, `spamd_result` is `None`, so the `if spamd_result and ...` condition is false and the replacement does **not** occur. **Confirmed at runtime:** the sender saw `550 SL E515 Email not exist` (not `250 SL E216`), consistent with the SPF branch not firing.

### 4.7 Causal correction — the empty `SLDomain` is contextual, not the direct cause (honest finding)

The AAP inferred that the rejection resolves through `is_valid_alias_address_domain()` (`app/email_utils.py:557`, which checks `SLDomain.get_by(domain=...)` at `:560`). **Runtime tracing shows this function is NOT on the `handle_forward` auto-create path** — it is called at `email_handler.py:1000` on an *existing* `alias.email`. The auto-create path instead checks `CustomDomain` (`app/alias_utils.py:92-104`), never `SLDomain`.

To test whether the empty `SLDomain` is the direct cause, `init_app.py` was run to populate it and the same message re-injected. Complete captured output — the `init_app.py` SL-domain log line, the `public_domain` row it created, and the full re-injection transcript:

```
$ docker exec -w /app sl-app bash -c '/app/venv/bin/python init_app.py > /tmp/q_initapp.log 2>&1; echo "exit=$?"'
exit=0
$ docker exec sl-app grep "SL domain" /tmp/q_initapp.log
2026-07-08 22:25:25,615 - SL - INFO - 3285 - "/app/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain

$ docker exec sl-postgres psql -U myuser -d simplelogin -c 'SELECT id, domain FROM public_domain;'
 id |  domain  
----+----------
  1 | sl.local
(1 row)

$ docker exec sl-app swaks --to e1@sl.local --from hey@google.com --server 127.0.0.1:20381 ; echo "swaks exit=$?"
swaks exit=26
=== Trying 127.0.0.1:20381...
=== Connected to 127.0.0.1.
<-  220 reverse-code-generator-64e0e612-hfqqw Python SMTP 1.4.2
 -> EHLO reverse-code-generator-64e0e612-hfqqw
<-  250-reverse-code-generator-64e0e612-hfqqw
<-  250-SIZE 33554432
<-  250-8BITMIME
<-  250-SMTPUTF8
<-  250 HELP
 -> MAIL FROM:<hey@google.com>
<-  250 OK
 -> RCPT TO:<e1@sl.local>
<-  250 OK
 -> DATA
<-  354 End data with <CR><LF>.<CR><LF>
 -> Date: Wed, 08 Jul 2026 22:25:26 +0000
 -> To: e1@sl.local
 -> From: hey@google.com
 -> Subject: test Wed, 08 Jul 2026 22:25:26 +0000
 -> Message-Id: <20260708222526.003302@reverse-code-generator-64e0e612-hfqqw>
 -> X-Mailer: swaks v20201014.0 jetmore.org/john/code/swaks/
 -> 
 -> This is a test mailing
 -> 
 -> 
 -> .
<** 550 SL E515 Email not exist
 -> QUIT
<-  221 Bye
=== Connection closed with remote host.
```

**With `SLDomain` populated (`sl.local` present), the reply is still `550 SL E515 Email not exist` via the identical chain.** So a bare, non-existent public-domain address is rejected regardless of whether `sl.local` is a configured SL domain: public-domain aliases are created via the UI/API, not auto-created from inbound mail. The empty `SLDomain` is the *context the question describes*, but the direct cause of the `550` is that no `Alias` row exists and neither auto-create path (custom-domain nor directory) applies. This is reported honestly rather than forcing the inferred SLDomain-decision narrative.

### 4.8 AFTER state and repeatability

**AFTER state** — measured immediately after the empty-`SLDomain` rejections of §4.3–§4.4 (i.e. before the §4.7 populate step). Actual `psql` output rows confirm the rejection created nothing (`public_domain` still empty because `init_app.py` was skipped; `alias` still empty because no on-the-fly alias was created):

```
$ docker exec sl-postgres psql -U myuser -d simplelogin -c 'SELECT count(*) FROM public_domain;'
 count 
-------
     0
(1 row)

$ docker exec sl-postgres psql -U myuser -d simplelogin -c 'SELECT count(*) FROM alias;'
 count 
-------
     0
(1 row)
```

**Repeatability.** The injection was repeated three times against the same running handler; for each run the SMTP reply seen by the sender, the `email_handler.py:551` rejection log, and the `email_handler.py:2367` finish return-code were captured. Exact command and output:

```
$ docker exec sl-app bash -c '
for i in 1 2 3; do
  before=$(wc -l < /tmp/q3_email.log)
  reply=$(swaks --to e1@sl.local --from hey@google.com --server 127.0.0.1:20381 2>&1 | grep -E "<\*\*" | sed "s/^[^0-9]*//")
  rej=$(tail -n +$((before+1)) /tmp/q3_email.log | grep -m1 -oE "alias e1@sl.local cannot be created on-the-fly, return 550")
  fin=$(tail -n +$((before+1)) /tmp/q3_email.log | grep -m1 -oE "return code .550 SL E515 Email not exist.")
  echo "run $i: SMTP_reply=[$reply] | reject_log=[$rej] | finish=[$fin]"
done'
run 1: SMTP_reply=[550 SL E515 Email not exist] | reject_log=[alias e1@sl.local cannot be created on-the-fly, return 550] | finish=[return code '550 SL E515 Email not exist']
run 2: SMTP_reply=[550 SL E515 Email not exist] | reject_log=[alias e1@sl.local cannot be created on-the-fly, return 550] | finish=[return code '550 SL E515 Email not exist']
run 3: SMTP_reply=[550 SL E515 Email not exist] | reject_log=[alias e1@sl.local cannot be created on-the-fly, return 550] | finish=[return code '550 SL E515 Email not exist']
```

All three runs were byte-identical: SMTP reply `550 SL E515 Email not exist`, the `email_handler.py:551` rejection log, and the `email_handler.py:2367` finish line with return code `'550 SL E515 Email not exist'`.

### 4.9 Status constants for context

- `E200 = "250 Message accepted for delivery"` (`app/email/status.py:2`) — the accept case.
- `E207` (`app/email/status.py:12` → the literal string `"250 SL E207 No bounce report"`) — returned as `[(True, status.E207)]` at `email_handler.py:553` only when `should_ignore_bounce(...)` is true.
- `E216 = "250 SL E216 Handled spf policy"` (`app/email/status.py:24`) — the SPF-override replacement (§4.6).
- `E502 = "550 SL E502 Email not exist"` (`app/email/status.py:39`) — a sibling `550` used elsewhere; distinct from the `E515` returned here.
- `E515 = "550 SL E515 Email not exist"` (`app/email/status.py:51`) — the observed reply.

---

## 5. Coverage pass and external-tool validation

### 5.1 Dev-vs-prod SMTP port

Dev binds **20381** (`email_handler.py:2399`, argparse `default=20381`); production binds the privileged port **25**. All Q2/Q3 observations here used the dev port 20381 so they are reproducible without privileged binding. (MailHog on port 1080 — `CONTRIBUTING.md:221` — is available for viewing dev mail but is not required for the rejection path exercised in Q3.)

### 5.2 External-tool validation (web-searched, authoritative docs)

- **`swaks`** (Swiss Army Knife SMTP — the all-purpose SMTP transaction tester) — used to inject the Q3 message. The target server is selected with the `-s`/`--server [host[:port]]` option; per the manpage this option tells swaks to "use network sockets and specify the hostname or IP address to which to connect", with the TCP port either appended as `host:port` or supplied separately via `-p`/`--port`. It is used here as `--server 127.0.0.1:20381`. By default swaks prints a faithful transcript of the SMTP dialog to STDOUT/STDERR, annotating each line with a *transaction hint*: `->` marks an expected line **sent by swaks to the target server**, `<-` marks an expected line **received from the target server**, and `<**` marks an **unexpected line received from the target server** — which is exactly how a 5xx rejection such as `550 SL E515 Email not exist` is flagged (swaks expected a 2xx/3xx greeting to `RCPT TO`/`DATA` and instead received a 5xx). The server's reply is therefore captured directly at the sender. Authoritative sources: official project page <https://jetmore.org/john/code/swaks/>; reference documentation <https://jetmore.org/john/code/swaks/latest/doc/ref.txt> (see "OUTPUT OPTIONS" → the transaction-hints table for `->`/`<-`/`<**`, and the transport/network options for `--server`); Ubuntu manpage <https://manpages.ubuntu.com/manpages/jammy/man1/swaks.1.html>.
- **`aiosmtpd`** — the async SMTP server library that powers `email_handler.py`. Per its documented handler-hook contract, `handle_DATA(server, session, envelope) -> str` returns the "Response message to be sent to the client", and hooks "must return a string status"; the docs further state that if a hook returns `None` or raises, aiosmtpd logs an exception and returns a `451` code to the client. This validates that the `550 SL E515 Email not exist` seen by swaks is the **authentic protocol reply**, not a log artifact. The exact return path, verified at runtime: SimpleLogin's `MailHandler.handle_DATA` (`email_handler.py:2289`) calls `ret = self._handle(envelope, msg)` (`email_handler.py:2292`) and `return ret` (`email_handler.py:2293`); `_handle` (`email_handler.py:2335`) computes `return_status = handle(envelope, msg)` (`email_handler.py:2353`) and `return return_status` (`email_handler.py:2378`); the module-level `handle()` (`email_handler.py:1945`, declared `-> str`) routes to `handle_forward()`, which returns `status.E515` — the literal string `"550 SL E515 Email not exist"` (`app/email/status.py:51`). aiosmtpd transmits that exact string back as the DATA reply, which is what swaks records. (SimpleLogin's `handle_DATA` never returns `None`: its outer `except Exception` returns `status.E404` at `email_handler.py:2332`, so aiosmtpd's `451`-on-`None` fallback is never reached in this path.) Authoritative sources: handlers doc <https://aiosmtpd.aio-libs.org/en/latest/handlers.html>; controller / programmatic-usage doc <https://aiosmtpd.aio-libs.org/en/latest/controller.html>; docs source <https://github.com/aio-libs/aiosmtpd/blob/master/aiosmtpd/docs/handlers.rst>.
- **Alembic / Flask-Migrate** — `Migrate()` is not registered on the running entry points; `shell.py`'s `flask_migrate.upgrade()` (`shell.py:20`) is dead code inside an `if False:` block (`shell.py:11`), and `__main__` (`shell.py:72`) only calls `embed()`. Therefore `alembic upgrade head` (`alembic.ini:5` `script_location = migrations`) is the reliable migration command, matching the `CONTRIBUTING.md:106` quickstart. `migrations/env.py` imports `Base` (`:26`) and `DB_URI` (`:27`) and sets `target_metadata = Base.metadata` (`:28`).

### 5.3 Named-item coverage (re-reading each question)

**Q1** — every named item addressed:
- **"empty database (no tables, migrations not run)"** → BEFORE-state proof, §2.2 (`\dt` → "Did not find any relations", `information_schema.tables` count 0).
- **`python server.py`** → real entry point, §2.3 (`__main__` `server.py:598` → `local_main` `:572` → `app.run(debug=True, port=7777)` `:588`).
- **"open the login page in the browser"** → `GET /auth/login` = HTTP 200, no DB (§2.3); failure only on POST (§2.3–2.4).
- **"the full Python exception that gets raised"** → complete unedited traceback, §2.4 (`sqlalchemy.exc.ProgrammingError` wrapping `psycopg2.errors.UndefinedTable`, `relation "users" does not exist`).

**Q2** — every named item addressed:
- **"database migrations ... have been run"** → `alembic upgrade head` (255 scripts), §3.1.
- **`init_app.py`** → run with captured output (`Add sl.local to SL domain`, `init_app.py:44`), §3.1.
- **"which Python services must be running"** → two tiers enumerated, §3.2 (essential: web app, email handler, PostgreSQL, Redis) and §3.3 (auxiliary: `job_runner.py`, `cron.py`, `event_listener.py`).
- **"exact log output ... startup messages from stdout/stderr"** → captured banner for each service, §3.2–3.3.
- **"binding to their ports"** → web app `/health` = `success` on 7777 (§3.2.1); email handler port-connect proof on 20381 (§3.2.2); PostgreSQL `pg_isready` 5432 (§3.2.3); Redis `PONG` 6379 (§3.2.4); auxiliary processes explicitly bind no TCP port (§3.3).

**Q3** — every named item addressed:
- **"run migrations but don't run `init_app.py`"** → migrated-but-uninitialized state, §4.2.
- **"`SLDomain` table should be empty"** → `SELECT count(*) FROM public_domain` = 0 (`class SLDomain` → `public_domain`, `app/models.py:3116-3119`), §4.2.
- **"start the email handler"** → `python email_handler.py` :20381 with readiness banner, §4.3.
- **"incoming email addressed to any `@sl.local` address"** → `swaks --to e1@sl.local ...`, full transcript, §4.3.
- **"What SMTP status code does the email handler return to the sender"** → **`550 SL E515 Email not exist`** (`app/email/status.py:51`), §4.1/§4.3.
- **"what does it log to explain the rejection"** → `alias e1@sl.local cannot be created on-the-fly, return 550` at `email_handler.py:551`, plus finish line at `email_handler.py:2367`, §4.4.

### 5.4 Observed-vs-inferred summary

Every behavioral value in §2–§4 (the Q1 traceback and 500 render path; the Q2 readiness banners, `/health`, `pg_isready`, `PONG`, `connected_clients:1`, listening-socket counts, `HttpEventSink` default vs `ConsoleEventSink` under `--dry-run`; the Q3 `swaks` transcript, `550 SL E515`, the rejection log, before/after counts, and the SLDomain-populated re-injection) was **observed at runtime** and transcribed from captured bytes. The single item explicitly labeled **(inferred)** and not exercised is the Q1 alternate trigger via `load_user` / an authenticated session cookie (§2.7). All `file:line` citations were cross-checked against the running source at HEAD `2cd6ee777f8c`.
