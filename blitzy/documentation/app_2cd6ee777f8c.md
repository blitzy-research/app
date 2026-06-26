# SimpleLogin — Runtime Behavior Onboarding Q&A

This document answers four onboarding questions about the **SimpleLogin** (Flask + PostgreSQL) codebase by **observing the application actually running** — not just by reading what the code *says* should happen. Each answer was produced by building and running the real, pinned stack, exercising it (HTTP requests, database `SELECT`s, a deliberate failure injection), capturing the **verbatim** output, and then tracing that observed behavior back to the responsible source line.

**Methodology (one-liner).** The pinned stack was built and run in Docker — `flask==1.1.2`, `Werkzeug==1.0.1`, `SQLAlchemy==1.3.24`, `psycopg2-binary==2.9.3`, `gunicorn==20.0.4`, `arrow==0.16.0`, `Flask-Migrate==2.5.3`, `Flask-Limiter==1.4`, `python-dotenv==0.14.0`, `sqlalchemy-utils==0.36.8` on Python 3.10 with PostgreSQL — the schema applied with `alembic upgrade head` and baseline data seeded with `flask dummy-data`, following the canonical local recipe `alembic upgrade head && flask dummy-data && python3 server.py` [CONTRIBUTING.md:L106].

**No source files were modified.** This markdown document is the only artifact produced. The application's container image runs from working directory `/code` (`WORKDIR /code` [Dockerfile:L3]), which is why the captured console paths read `/code/...`. All investigation scripts were transient and lived outside the repository tree; after capturing evidence the repository was verified to be byte-for-byte unmodified (clean `git status`).

**Environment variables** were taken from `example.env`: `URL=http://localhost:7777` [example.env:L6], `EMAIL_DOMAIN=sl.local` [example.env:L22], `SUPPORT_EMAIL=support@sl.local` [example.env:L40], `EMAIL_SERVERS_WITH_PRIORITY=[(10, "email.hostname.")]` [example.env:L61], `DB_URI=postgresql://...:5432/simplelogin` [example.env:L75], and `FLASK_SECRET=secret` [example.env:L77].

> **A note on run-specific values.** Some captured values are inherently specific to a single run — the randomly generated alias local-part, the autoincrement `id`, timestamps, process IDs (pids), and the per-import `GNUPGHOME` temp-directory paths. These are labeled **observed-in-this-run** below. The *behavior, structure, table, columns, status codes, and error types* are stable and reproducible; only those particular values change between runs (this was confirmed by re-running the stack).

---

## The four questions

1. *"What port does the Flask app bind to, and what do the startup logs look like when it initializes?"*
2. *"What response body and HTTP status code does the health check return?"*
3. *"When a user creates an alias through the API, what does the actual JSON response look like, and what ends up in the database — which table, and what values?"*
4. *"If PostgreSQL isn't available when the app starts, what error appears, and where does it break?"*

---

## Q1 — Port & startup logs

> *"What port does the Flask app bind to, and what do the startup logs look like when it initializes?"*

**Answer (one line).** The app binds to **port 7777** on both run paths — `127.0.0.1:7777` for the development server and `0.0.0.0:7777` for the production Gunicorn server.

### (a) Commands executed

```bash
# Development path
python3 server.py

# Production path
gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
```

### (b) Verbatim stdout/stderr — DEV path (`python3 server.py`)

```
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/onhlhtqfjdvivyegfsvu
Upload files to local dir
>>> init logging <<<
2026-06-26 19:50:08,591 - SL - DEBUG - 1984 - "/code/app/utils.py:17" - <module>() -  - load words file: /code/local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/lnivykodhbqjfflnyjpa
Upload files to local dir
>>> init logging <<<
2026-06-26 19:50:12,409 - SL - DEBUG - 1990 - "/code/app/utils.py:17" - <module>() -  - load words file: /code/local_data/test_words.txt
```

On the **first** `GET /health` request only, the dev server additionally emitted this one stderr warning (noted here separately — it is **not** part of startup):

```
/usr/local/lib/python3.10/site-packages/flask_debugtoolbar/__init__.py:213: UserWarning: Could not insert debug toolbar. </body> tag not found in response.
```

### (b′) Verbatim stdout/stderr — PROD path (Gunicorn)

```
[2026-06-26 19:52:18 +0000] [2089] [INFO] Starting gunicorn 20.0.4
[2026-06-26 19:52:18 +0000] [2089] [INFO] Listening at: http://0.0.0.0:7777 (2089)
[2026-06-26 19:52:18 +0000] [2089] [INFO] Using worker: sync
[2026-06-26 19:52:18 +0000] [2090] [INFO] Booting worker with pid: 2090
[2026-06-26 19:52:18 +0000] [2091] [INFO] Booting worker with pid: 2091
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/esykudpgxsxgidnagbpf
Upload files to local dir
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/mqzaetutatveczzgciix
Upload files to local dir
>>> init logging <<<
>>> init logging <<<
2026-06-26 19:52:20,726 - SL - DEBUG - 2090 - "/code/app/utils.py:17" - <module>() -  - load words file: /code/local_data/test_words.txt
2026-06-26 19:52:20,798 - SL - DEBUG - 2091 - "/code/app/utils.py:17" - <module>() -  - load words file: /code/local_data/test_words.txt
```

### (e) Rationale & citations

This is the most instructive part of Q1: **what the startup logs actually look like is *not* the textbook Flask/Werkzeug banner.**

**Port 7777 (dev).** `local_main()` calls `app.run(debug=True, port=7777)` [server.py:L588], invoked under the `if __name__ == "__main__":` → `local_main()` guard [server.py:L598-L599]. Flask's development server defaults its host to `127.0.0.1`, so the dev server binds **`127.0.0.1:7777`** (confirmed by connecting a socket to that address).

**Port 7777 (prod).** The image declares `EXPOSE 7777` [Dockerfile:L44] and runs `CMD ["gunicorn","wsgi:app","-b","0.0.0.0:7777","-w","2","--timeout","15"]` [Dockerfile:L47], so Gunicorn binds **`0.0.0.0:7777`** with two `sync` workers. The `wsgi:app` target resolves through `wsgi.py`, which is exactly `from server import create_app` followed by `app = create_app()` (three lines); the factory is `def create_app() -> Flask:` [server.py:L139].

**Where the SimpleLogin prints come from.**

- `>>> URL: http://localhost:7777` is `print(">>> URL:", URL)` [app/config.py:L80], where `URL = os.environ["URL"]` [app/config.py:L79].
- `MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value` [app/config.py:L123], `Paddle param not set` [app/config.py:L217], and `Upload files to local dir` [app/config.py:L328] are default-value notices printed by `app/config.py` at import time.
- `WARNING: Use a temp directory for GNUPGHOME /tmp/<random>` is printed when `GNUPGHOME` is unset [app/config.py:L262]; a fresh temp directory is created and named **on each import**, which is why the path **differs** between the two import passes.
- `>>> init logging <<<` is `print(">>> init logging <<<")` [app/log.py:L67].
- The `... - SL - DEBUG - <pid> - "/code/app/utils.py:17" - <module>() -  - load words file: ...` line is the **"SL" application logger** writing at `DEBUG` to stdout, using the format string `"%(asctime)s - %(name)s - %(levelname)s - %(process)d - \"%(pathname)s:%(lineno)d\" - %(funcName)s() - %(message_id)s - %(message)s"` [app/log.py:L12-L15]; the logger is created via `LOG = _get_logger("SL")` [app/log.py:L79].

**Why the SimpleLogin prints appear *twice* (dev).** In debug mode the Werkzeug reloader spawns a child process (`WERKZEUG_RUN_MAIN=true`) that **re-imports** the application module. Consequently `>>> URL:`, the `GNUPGHOME` notice (with a *different* temp path), `>>> init logging <<<`, and the SL `DEBUG` line all repeat. This duplication is reproduced literally above rather than collapsed.

**Why the Flask banner lines appear but the classic dev-server lines do *not* (the key finding).**

- **Present** (printed once, by the parent only): `* Serving Flask app "server" (lazy loading)`, `* Environment: production`, the two-line development-server `WARNING`, and `* Debug mode: on`. These come from Flask's `flask.cli.show_server_banner(...)` via `click.echo`/`click.secho` — **not** through the logging system — so SimpleLogin's logger configuration cannot suppress them. (In the reloader child they are skipped because `show_server_banner` returns early when `WERKZEUG_RUN_MAIN == "true"`.)
- **Absent** (this contradicts the "idealized" Werkzeug banner): `* Running on http://127.0.0.1:7777/ (Press CTRL+C to quit)`, `* Restarting with stat`, `* Debugger is active!`, and `* Debugger PIN: nnn-nnn-nnn` were **not printed at runtime**, even though the server *was* serving on `127.0.0.1:7777`. The reason: Werkzeug emits all four through its `_log()` helper, which uses `logging.getLogger("werkzeug")`, and SimpleLogin disables that logger at import — `log = logging.getLogger("werkzeug")` then `log.disabled = True` [app/log.py:L70-L71]. A disabled logger drops those records, so the lines never reach stdout.
- **Per-request access lines are also suppressed** for the same reason: the usual `127.0.0.1 - - [..] "GET /health HTTP/1.1" 200` line did **not** appear after hitting `/health`, because the `werkzeug` logger is disabled [app/log.py:L70-L71].

**Production logs differ in origin.** Gunicorn's `Starting gunicorn 20.0.4`, `Listening at: http://0.0.0.0:7777`, `Using worker: sync`, and `Booting worker with pid: ...` come from Gunicorn's own logger (not `werkzeug`), so they appear normally; SimpleLogin's import prints then appear **once per worker** (pids `2090`/`2091`).

**Takeaway.** "What the startup logs look like" at runtime is the SimpleLogin prints plus a *partial* Flask banner. The `* Running on ...`, reloader, and Debugger-PIN lines are silenced by SimpleLogin's deliberate disabling of the `werkzeug` logger [app/log.py:L70-L71] — precisely the gap between "what the code says should happen" and what actually happens.

> **Observed-in-this-run:** the pids (`1984`/`1990` dev; `2089`/`2090`/`2091` prod), the timestamps, and the `GNUPGHOME` temp-directory names are specific to this capture and differ on every run.

---

## Q2 — Health check

> *"What response body and HTTP status code does the health check return?"*

**Answer (one line).** `GET /health` returns the body **`success`** (plain text, 7 bytes) with HTTP status **`200`**.

### (a) Command executed

```bash
curl -i http://127.0.0.1:7777/health   # run against both the dev and prod servers
```

### (c) HTTP response — verbatim (status line + key headers + body)

DEV (Werkzeug):

```
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 7
Server: Werkzeug/1.0.1 Python/3.10.20

success
```

PROD (Gunicorn):

```
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Content-Length: 7
Content-Type: text/html; charset=utf-8

success
```

### (e) Rationale & citations

The route is `@app.route("/health", methods=["GET"])` with `def healthcheck(): return "success", 200` [server.py:L213-L215]. Flask interprets the returned `(str, int)` tuple as `(body, status_code)`, producing status **`200`** and body `success`. Because the handler sets no explicit content type, Flask applies its default `text/html; charset=utf-8`, and `Content-Length: 7` matches the seven ASCII bytes of `success`.

The status-line HTTP version differs only because of the server fronting the app — Werkzeug emits `HTTP/1.0` while Gunicorn emits `HTTP/1.1` — but the **body and status code are identical** on both paths. This endpoint is intentionally trivial: it performs **no database access**, so there is no database state to report for Q2 (and, as noted in Q1, the `werkzeug` logger being disabled [app/log.py:L70-L71] means hitting `/health` produces no per-request access-log line on the dev server).

---

## Q3 — Alias creation (API JSON + database row)

> *"When a user creates an alias through the API, what does the actual JSON response look like, and what ends up in the database — which table, and what values?"*

**Answer (one line).** A successful `POST /api/alias/random/new` returns **HTTP `201`** with a JSON object describing the new alias (plus an extra top-level `alias` key), and persists exactly one row into the **`alias`** table.

### Authentication precondition (a non-standard header)

Authentication uses the **`Authentication`** request header — **not** `Authorization`. The flow is `api_code = request.headers.get("Authentication")` [app/api/base.py:L17] → `api_key = ApiKey.get_by(code=api_code)` [app/api/base.py:L18] → on success `g.user = api_key.user` [app/api/base.py:L34]; the endpoint is guarded by the `require_api_auth` decorator [app/api/base.py:L52-L60]. A missing or wrong key returns `{"error":"Wrong api key"}` with **`401`** via `jsonify(error="Wrong api key"), 401` [app/api/base.py:L27]. The API blueprint is mounted at prefix `/api` [app/api/base.py:L11]. The seeded API key value used here is `code`, created by `flask dummy-data`.

### Seed prerequisites

`flask dummy-data` (the CLI command at [server.py:L490-L495] → `fake_data()` in [app/fake_data.py:L40]) creates the user `john@wick.com` (password `password`) [CONTRIBUTING.md:L109], an `ApiKey` with `code="code"` [app/fake_data.py:L121-L122], and a verified default `Mailbox` whose id becomes `user.default_mailbox_id`. This matters because `alias.mailbox_id` is a **non-null** foreign key [app/models.py:L1506-L1508], sourced from `user.default_mailbox_id` in `Alias.create_new_random` [app/models.py:L1721]. Seeding also pre-creates several aliases, so the `alias` table already held **11 rows (ids 1–11)** before the API call in this capture; the newly created alias is therefore **`id=12`** (autoincrement).

### (a) Commands executed (three cases, to demonstrate the auth behavior)

```bash
# 1) No header
curl -i -X POST http://127.0.0.1:7777/api/alias/random/new

# 2) Wrong key
curl -i -X POST http://127.0.0.1:7777/api/alias/random/new -H 'Authentication: WRONGKEY'

# 3) Correct key (success)
curl -i -X POST http://127.0.0.1:7777/api/alias/random/new \
     -H 'Authentication: code' \
     -H 'Content-Type: application/json' \
     -d '{"note":"created via API for Q3"}'
```

### (c) HTTP responses — verbatim

Cases 1 & 2 (no / wrong header):

```
HTTP/1.1 401 UNAUTHORIZED
Content-Type: application/json
Content-Length: 26

{"error":"Wrong api key"}
```

Case 3 (success) — status line + headers:

```
HTTP/1.1 201 CREATED
Content-Type: application/json
Content-Length: 426
Server: gunicorn/20.0.4
Access-Control-Allow-Origin: *
```

…and the literal JSON body (keys are alphabetically sorted because Flask `jsonify` defaults `JSON_SORT_KEYS=True`):

```json
{"alias":"gavels_rushes985@sl.local","creation_date":"2026-06-26 19:53:35+00:00","creation_timestamp":1782503615,"disable_pgp":false,"email":"gavels_rushes985@sl.local","enabled":true,"id":12,"latest_activity":null,"mailbox":{"email":"john@wick.com","id":1},"mailboxes":[{"email":"john@wick.com","id":1}],"name":null,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":"created via API for Q3","pinned":false,"support_pgp":false}
```

The same payload, pretty-printed for readability (semantics identical to the literal body above):

```json
{
  "alias": "gavels_rushes985@sl.local",
  "creation_date": "2026-06-26 19:53:35+00:00",
  "creation_timestamp": 1782503615,
  "disable_pgp": false,
  "email": "gavels_rushes985@sl.local",
  "enabled": true,
  "id": 12,
  "latest_activity": null,
  "mailbox": {"email": "john@wick.com", "id": 1},
  "mailboxes": [{"email": "john@wick.com", "id": 1}],
  "name": null,
  "nb_block": 0,
  "nb_forward": 0,
  "nb_reply": 0,
  "note": "created via API for Q3",
  "pinned": false,
  "support_pgp": false
}
```

### (d) Database state — `SELECT * FROM alias WHERE id = 12;`

```
id                           | 12
created_at                   | 2026-06-26 19:53:35.352922
updated_at                   | (NULL)
user_id                      | 1
email                        | gavels_rushes985@sl.local
enabled                      | t
custom_domain_id             | (NULL)
automatic_creation           | f
directory_id                 | (NULL)
note                         | created via API for Q3
mailbox_id                   | 1
name                         | (NULL)
disable_pgp                  | f
cannot_be_disabled           | f
disable_email_spoofing_check | f
batch_import_id              | (NULL)
pinned                       | f
original_owner_id            | (NULL)
transfer_token               | (NULL)
hibp_last_check              | (NULL)
ts_vector                    | 'api':3 'creat':1 'q3':5 'via':2
transfer_token_expiration    | 2026-06-26 19:53:35.352939
last_email_log_id            | (NULL)
flags                        | 0
```

### (e) Rationale & citations

**Endpoint & status.** `POST /api/alias/random/new` returns `201` on success [app/api/views/new_random_alias.py:L21-L25]; the analogous custom-alias endpoints are `POST /api/v2/alias/custom/new` [app/api/views/new_custom_alias.py:L28-L32] and `POST /api/v3/alias/custom/new` [app/api/views/new_custom_alias.py:L115-L119]. Each stacks `@limiter.limit(ALIAS_LIMIT)` + `@require_api_auth` + `@parallel_limiter.lock(name="alias_creation")`.

**Why the JSON has an extra top-level `alias` key.** The view returns `jsonify(alias=alias.email, **serialize_alias_info_v2(get_alias_info_v2(alias))), 201` [app/api/views/new_random_alias.py:L114-L117]. So the response is the `serialize_alias_info_v2` object [app/api/serializer.py:L55-L93] (keys: `id, email, creation_date, creation_timestamp, enabled, note, name, nb_forward, nb_block, nb_reply, mailbox{id,email}, mailboxes[{id,email}], support_pgp, disable_pgp, latest_activity, pinned`) **plus** an extra `alias` key equal to the alias email. That is why `alias` and `email` carry the same value (`gavels_rushes985@sl.local`). This duplication is easy to miss from the serializer alone — it is only visible by inspecting the view's return statement and/or the live response.

**Which table.** The row is written to the **`alias`** table — `class Alias(Base, ModelMixin)` with `__tablename__ = "alias"` [app/models.py:L1469-L1470].

**How the values arise (each mapped to its source).** Persistence happens via `Alias.create_new_random(user, ...)` [app/models.py:L1721] → `ModelMixin.create(...)`, which does `Session.add(...)` and commits when a `commit` kwarg is set [app/models.py:L115-L130]; here the **view** performs the commit [app/api/views/new_random_alias.py:L107]. Column provenance:

- `id = 12` — autoincrement primary key [app/models.py:L63].
- `created_at = 2026-06-26 19:53:35.352922` — non-null, default `arrow.utcnow` [app/models.py:L64].
- `updated_at = NULL` — only set on update [app/models.py:L65].
- `user_id = 1` — the authenticated user (`john`).
- `email = gavels_rushes985@sl.local` — unique, non-null [app/models.py:L1477]; the local-part is randomly generated (word-based scheme), hence run-specific.
- `enabled = t` — default `True` [app/models.py:L1482].
- `flags = 0` — default `0` [app/models.py:L1483-L1485].
- `custom_domain_id = NULL` and `directory_id = NULL` — a *random* alias has neither [app/models.py:L1487, app/models.py:L1499].
- `automatic_creation = f` [app/models.py:L1494]; `disable_pgp = f` [app/models.py:L1516]; `cannot_be_disabled = f` [app/models.py:L1521]; `disable_email_spoofing_check = f` [app/models.py:L1528]; `pinned = f` [app/models.py:L1545]; `name = NULL` [app/models.py:L1480]; and `batch_import_id`, `original_owner_id`, `transfer_token`, `hibp_last_check`, `last_email_log_id` are all `NULL`.
- `mailbox_id = 1` — set to `user.default_mailbox_id` in `create_new_random` [app/models.py:L1721]; this is a non-null foreign key [app/models.py:L1506-L1508].
- `transfer_token_expiration = 2026-06-26 19:53:35.352939` — this column carries `default=arrow.utcnow` [app/models.py:L1549-L1551], so it is stamped at insert time even though `transfer_token` itself is `NULL`.
- `ts_vector = 'api':3 'creat':1 'q3':5 'via':2` — a **generated/persisted** PostgreSQL column, `to_tsvector('english', note)` [app/models.py:L1559-L1561]. The note `created via API for Q3` becomes the lexemes `creat`(1) `via`(2) `api`(3) `q3`(5): `created` is stemmed to `creat`, and the English stopword `for` (position 4) is dropped. This is visible proof that the column is **computed by PostgreSQL**, not stored as the raw note text.

**Reproducibility caveat.** The alias local-part (`gavels_rushes985`), the `id` (`12`), and the timestamps are specific to this run and seed state. On a fresh database the **table, columns, and their semantics are identical**, but those particular values differ (re-running the capture produced a different random local-part and the next autoincrement id, with all other column semantics unchanged).


---

## Q4 — PostgreSQL unavailable at startup

> *"If PostgreSQL isn't available when the app starts, what error appears, and where does it break?"*

**Answer (one line).** The process raises **`sqlalchemy.exc.OperationalError`** wrapping **`psycopg2.OperationalError: ... Connection refused`** and dies at **import time** at `connection = engine.connect()` [app/db.py:L12] — **before `create_app()`** [server.py:L139] ever runs.

### (a) Command executed (PostgreSQL unreachable)

```bash
export DB_URI="postgresql://myuser:mypassword@127.0.0.1:5999/simplelogin"   # nothing listening on 5999
python3 server.py
# => process exits with code 1
```

### (b) Verbatim output

The SimpleLogin import prints appear, **then** the traceback — there is no Flask banner, no `Running on`, and no request handling, because the process never gets that far.

Prints emitted before the crash:

```
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/<random>
Upload files to local dir
>>> init logging <<<
```

The traceback (internal SQLAlchemy pool frames abbreviated with `...`; the two `OperationalError` lines and the four user-code frames are verbatim — this is the decisive evidence):

```
Traceback (most recent call last):
  ...
psycopg2.OperationalError: connection to server at "127.0.0.1", port 5999 failed: Connection refused
	Is the server running on that host and accepting TCP/IP connections?


The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "/code/server.py", line 31, in <module>
    from app.admin_model import (
  File "/code/app/admin_model.py", line 11, in <module>
    from app import models, s3
  File "/code/app/models.py", line 32, in <module>
    from app.db import Session
  File "/code/app/db.py", line 12, in <module>
    connection = engine.connect()
  ...
sqlalchemy.exc.OperationalError: (psycopg2.OperationalError) connection to server at "127.0.0.1", port 5999 failed: Connection refused
	Is the server running on that host and accepting TCP/IP connections?

(Background on this error at: http://sqlalche.me/e/13/e3q8)
```

### (e) Rationale & citations — precisely *what* and *where*

**What error.** A *chained* exception. The direct cause is `psycopg2.OperationalError: ... Connection refused / Is the server running on that host and accepting TCP/IP connections?`, which SQLAlchemy re-raises as `sqlalchemy.exc.OperationalError: (psycopg2.OperationalError) ... Connection refused ... (Background on this error at: http://sqlalche.me/e/13/e3q8)`. The process exits with code **`1`**.

**Where it breaks (exact line).** `app/db.py` connects **eagerly at module import**. The lazy engine is built first via `engine = create_engine(config.DB_URI, connect_args={"application_name": config.DB_CONN_NAME})` [app/db.py:L9-L11], and the actual failing call is `connection = engine.connect()` on **`app/db.py:L12`** (followed by `Session = scoped_session(sessionmaker(bind=connection))` on `L14`). It is the `engine.connect()` call — not `create_engine(...)` — that opens the socket and therefore fails.

**When it breaks (the runtime-vs-code nuance).** The failure happens during **import**, before any application or server logic. Critically, the *actual* trigger observed at runtime is the import chain `from app.admin_model import (...)` [server.py:L31] → `from app import models, s3` [app/admin_model.py:L11] → `from app.db import Session` [app/models.py:L32] → `connection = engine.connect()` [app/db.py:L12]. In other words, `app.db` is first imported **transitively via `app.admin_model`** [server.py:L31], which fires *earlier* than the direct `from app.db import Session` [server.py:L76] that a quick read might lead you to expect. Both imports are module-level and both precede `create_app()` [server.py:L139], but the observed first-touch is the `server.py:L31` chain. The app never gets far enough to bind a port, print a Flask banner, or define routes — it dies while importing modules.

**Why eager connection matters.** Because the connection is established at import (not lazily on the first request), an unreachable PostgreSQL is fatal immediately at process startup: there is no degraded mode and no deferred retry. A reader scanning for `from app.db import Session` would land on [server.py:L76], but the live traceback proves the earlier transitive import is the real culprit — exactly the kind of detail that only running the code reveals.

> **Observed-in-this-run:** the `GNUPGHOME` temp-directory name varies per run; the chosen unreachable port (`5999`) is arbitrary (any port with nothing listening reproduces the identical error).

---

## Appendix — Reproducibility & environment

- **Run paths and binds.** Development: `python3 server.py` → `app.run(debug=True, port=7777)` [server.py:L588] binds `127.0.0.1:7777`. Production: `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` [Dockerfile:L47] binds `0.0.0.0:7777` with two `sync` workers. Both paths use **port 7777**.
- **Canonical local recipe.** `alembic upgrade head && flask dummy-data && python3 server.py` [CONTRIBUTING.md:L106]; afterwards one can log in as `john@wick.com / password` [CONTRIBUTING.md:L109].
- **Working directory.** The Docker image sets `WORKDIR /code` [Dockerfile:L3], so captured console paths read `/code/...`. Running the same code from a different working directory changes only that path prefix, not the behavior.
- **Run-specific values.** Alias local-part, autoincrement `id`, all timestamps, pids, and `GNUPGHOME` temp-directory names are specific to a single run and will differ on re-execution. The *structure, tables, columns, status codes, and error types* are stable and were confirmed reproducible across runs.
- **Optional accelerators (no behavioral effect).** The investigation environment substituted a faithful `re`-backed shim for the optional `pyre2` regex accelerator and used a newer `cbor2`; both are transitively optional and have **zero** effect on port binding, the health check, alias creation, or the database-failure path documented here.
- **No repository changes.** This document is the only artifact. No existing source file was modified, and every temporary investigation script lived outside the repository tree and was removed afterward; the source tree remains byte-for-byte unmodified (clean `git status`).

