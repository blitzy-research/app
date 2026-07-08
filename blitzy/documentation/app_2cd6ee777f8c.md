# SimpleLogin — Runtime Behavior Onboarding (Observed)

This document answers four onboarding questions about how the SimpleLogin Flask
application **actually behaves when it is running**. Every value reported below was
**observed from the running application** under the canonical configuration — it was
captured from real execution of the real code paths, not inferred from reading source.
Where a statement is derived from code rather than from observed output, it is labeled
**inferred**. To capture this evidence, temporary observation scripts were created under
`/tmp`, used to boot the app and probe it, and then removed; **no repository file was
modified or added except this single document** (`git status --porcelain` shows only this
`.md`).

Each question section is laid out the same way: (1) the **direct answer** first, (2) the
**exact values with `file:line` citations** naming the function/handler that does the
work, (3) the **exact command** that was run, (4) the **verbatim, unedited observed
output** in a fenced block, and finally a short **Why (cause → effect)** note explaining
the mechanism. Volatile fields (process IDs, timestamps, randomly generated alias
local-parts, row identifiers) are inherently run-specific; they are reproduced exactly as
captured and are called out as volatile in the stability notes.

## Canonical run context

All observations below were captured under the repository's own canonical configuration:

- **Configuration**: `CONFIG=tests/test.env` — a single dotenv file selected by the
  `CONFIG` environment variable and resolved at `app/config.py:L65-L69`
  (`config_file = os.environ.get("CONFIG")` → `print("load config file", config_file)` →
  `load_dotenv(...)`).
- **Runtime**: **Python 3.10.18** — observed via `python --version` in the canonical
  warmed image. `Dockerfile:L8` declares `FROM python:3.10`, a **floating** tag whose
  exact patch version resolves by build date (so it may differ across builds); the CI
  matrix pins `python-version: ["3.10"]` at `.github/workflows/main.yml:L40`, and
  `pyproject.toml:L61` declares `python = "^3.10"`.
- **Schema**: applied with `CONFIG=tests/test.env alembic upgrade head`, producing
  **77 tables** (Alembic head `32f25cbf12f6`).
- **PostgreSQL**: reachable on port **15432**, from
  `tests/test.env:L17` `DB_URI=postgresql://test:test@localhost:15432/test`.
- **Redis**: backs Flask-Limiter rate limiting and session storage, from
  `tests/test.env:L78` `MEM_STORE_URI=redis://localhost`.
- **Domains / URL**: `EMAIL_DOMAIN=sl.local` (`tests/test.env:L8`) and
  `URL=http://localhost` (`tests/test.env:L2`).
- **Canonical dependency versions** (each pinned in `poetry.lock` at the cited line):
  Flask `1.1.2` (`poetry.lock:L911`), Werkzeug `1.0.1` (`poetry.lock:L3424`), gunicorn
  `20.0.4` (`poetry.lock:L1448`), Flask-SQLAlchemy `2.5.1` (`poetry.lock:L1076`),
  SQLAlchemy `1.3.24` (`poetry.lock:L3054`), psycopg2-binary `2.9.3`
  (`poetry.lock:L2221`), arrow `0.16.0` (`poetry.lock:L202`).

Canonical invocations used throughout:

- Dev server: `CONFIG=tests/test.env python server.py`
- Production server: `CONFIG=tests/test.env gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15`
- Schema: `CONFIG=tests/test.env alembic upgrade head`

---

## Q1 — What port does the Flask app bind to, and what do the startup logs look like?

### Direct answer

The application always uses **port 7777**, but the **bind host differs by entry point**:

- **Dev server** — `python server.py` → `local_main()` → `app.run(debug=True, port=7777)`:
  binds **`127.0.0.1:7777`** (loopback only).
- **Production server** — `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15`:
  binds **`0.0.0.0:7777`** (all interfaces).

Both entry points listen on **7777**; they differ only in the bind **host**
(`127.0.0.1` vs `0.0.0.0`).

A notable observed detail: the Werkzeug **`* Running on http://127.0.0.1:7777/` banner is
ABSENT**, and there are **no per-request access log lines** — even though the socket is
genuinely bound (proven by the kernel socket table below). This is caused by the
`werkzeug` logger being disabled (see **Why**).

### Exact values and citations

- **Dev entry point**: `server.py:L572-L588` `def local_main()`. It sets
  `config.COLOR_LOG = True`, builds `app = create_app()`, attaches the Flask Debug
  Toolbar, sets `app.debug = True`, and finally calls `app.run(debug=True, port=7777)` at
  **`server.py:L588`**. It is entered via the module guard
  `if __name__ == "__main__": local_main()` at **`server.py:L598-L599`**.
- Because **no `host` argument** is passed to `app.run(...)`, Flask defaults the bind host
  to loopback. This is grounded in the **installed Flask `1.1.2` source** in the canonical
  environment (`site-packages/flask/app.py`): the `run()` docstring documents the default at
  `flask/app.py:L918-L921` — "*Defaults to `'127.0.0.1'` or the host in the `SERVER_NAME`
  config variable if present*" — and the resolution logic is `_host = "127.0.0.1"` at
  `flask/app.py:L969` followed by `host = host or sn_host or _host` at `flask/app.py:L977`.
  With `host=None` (none passed) and no `SERVER_NAME` host configured, that expression
  evaluates to `"127.0.0.1"`, which is why the dev server binds loopback. Binding `0.0.0.0`
  instead, per the official Flask documentation, "This tells your operating system to listen
  on all public IPs" (Flask Quickstart,
  <https://flask.palletsprojects.com/en/stable/quickstart/>) — which is exactly what the
  gunicorn production invocation does with its explicit `-b 0.0.0.0:7777`.
- **Production entry point**: `Dockerfile:L47`
  `CMD ["gunicorn","wsgi:app","-b","0.0.0.0:7777","-w","2","--timeout","15"]`; the exposed
  port is `Dockerfile:L44` `EXPOSE 7777`; the canonical Python is `Dockerfile:L8`
  `FROM python:3.10`. The gunicorn WSGI target is `wsgi.py`
  (`from server import create_app` at `wsgi.py:L1`; `app = create_app()` at `wsgi.py:L3`).

### Exact commands

- Dev bind capture (boot, then read the listening socket from the kernel):
  `CONFIG=tests/test.env python server.py` followed by inspecting `/proc/net/tcp`
  (equivalently `ss -ltnp`).
- Production bind capture:
  `CONFIG=tests/test.env gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15`.

### Verbatim observed output

Dev-server kernel socket (loopback bind):

```
local=127.0.0.1:7777 state=LISTEN
```

Dev-server full stdout (one representative capture; because `debug=True` enables the
Werkzeug reloader, the import runs twice, so the SL bootstrap block appears **twice** —
once in the reloader parent and once in the reloaded child; the `/tmp/...` GNUPGHOME paths
and the timestamps are volatile between runs):

```
load config file /app/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/wnrfjqwdeakzftyohyfr
Upload files to local dir
>>> init logging <<<
2026-07-08 08:28:15,012 - SL - DEBUG - 188 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
load config file /app/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/djftrotxxfkwhplxqils
Upload files to local dir
>>> init logging <<<
2026-07-08 08:28:16,889 - SL - DEBUG - 201 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

Two observed facts in this capture matter for the answer: (1) the Flask CLI preamble lines
(`* Serving Flask app "server" (lazy loading)`, `* Environment: production`, the two
development-server `WARNING:` lines, and `* Debug mode: on`) **are** printed — they are
emitted via `click`/`print`, not through the `werkzeug` logger; but (2) the Werkzeug
`* Running on http://127.0.0.1:7777/` banner is **absent** (see **Why** below). That
contrast — preamble present, banner and access logs absent — is itself the evidence that
only the `werkzeug`-logger-emitted output is suppressed, while the bind and the plain
`print`/`click` output are unaffected.

Production (gunicorn) kernel socket:

```
local=0.0.0.0:7777 state=LISTEN
```

Production (gunicorn) startup INFO lines:

```
[2026-07-07 22:23:36 +0000] [21596] [INFO] Starting gunicorn 20.0.4
[2026-07-07 22:23:36 +0000] [21596] [INFO] Listening at: http://0.0.0.0:7777 (21596)
[2026-07-07 22:23:36 +0000] [21596] [INFO] Using worker: sync
[2026-07-07 22:23:36 +0000] [21598] [INFO] Booting worker with pid: 21598
[2026-07-07 22:23:36 +0000] [21599] [INFO] Booting worker with pid: 21599
```

The two worker PIDs (`21598`, `21599`) reflect `-w 2` (two sync workers); the master PID
is `21596`. PIDs and the timestamp are volatile between runs; the structure (one master +
two workers, `Starting gunicorn 20.0.4`, `Listening at: http://0.0.0.0:7777`) is stable.

### The SL bootstrap block (printed at startup / by each worker)

In addition to the gunicorn INFO lines, the SimpleLogin app prints a bootstrap block at
import/startup. Each line and its exact origin:

- `load config file .../tests/test.env` — `app/config.py:L68`
  (`print("load config file", config_file)`).
- `>>> URL: http://localhost` — `app/config.py:L80` (`print(">>> URL:", URL)`).
- `WARNING: Use a temp directory for GNUPGHOME /tmp/...` — `app/config.py:L262`
  (`print("WARNING: Use a temp directory for GNUPGHOME", GNUPGHOME)`).
- `Upload files to local dir` — `app/config.py:L328` (`print("Upload files to local dir")`).
- `>>> init logging <<<` — `app/log.py:L67` (`print(">>> init logging <<<")`).
- A `load words file: ...` DEBUG line — `app/utils.py:L17`
  (`LOG.d("load words file: %s", WORDS_FILE_PATH)`) — the first SL-logger DEBUG line
  emitted at import.

The SL log **format** is fixed at `app/log.py:L12-L14`, and timestamps are in **UTC**
because `app/log.py:L43` sets `console_handler.formatter.converter = time.gmtime`.

### Why (cause → effect): the absent Werkzeug banner

Lead with the observed result: the dev server's usual **`* Running on http://127.0.0.1:7777/`
banner is absent**, and there are **no per-request access log lines**, even though the
socket is bound to `127.0.0.1:7777` (proven by the `local=127.0.0.1:7777 state=LISTEN`
kernel line above).

The cause is that the `werkzeug` logger is **disabled** at `app/log.py:L70-L71`.
Reproduced exactly (comment and both statements, no elision):

```
# Disable flask logs such as 127.0.0.1 - - [15/Feb/2013 10:52:22] "GET /index.html HTTP/1.1" 200
log = logging.getLogger("werkzeug")
log.disabled = True
```

Cause → effect: Werkzeug emits both its startup "Running on ..." banner and its
per-request access lines through the `werkzeug` logger. Because that logger's `disabled`
attribute is set to `True` at import time (`app/log.py:L70-L71`), those records are
suppressed — so the banner never appears and no access logs are printed, **despite** the
socket being correctly bound. The bind itself is unaffected; only the logging output is
suppressed. This is why the direct answer leads with the observed "banner absent" result
and then supplies the causal reason.

---

## Q2 — What does the health check return (body and status)?

### Direct answer

`GET /health` returns HTTP status **200** with a body that is **exactly** the raw ASCII
string `success` — **not** JSON, and with **no trailing newline** — and a
`Content-Type: text/html; charset=utf-8`.

### Exact values and citations

The handler is `healthcheck()` registered at `server.py:L213-L215`:

```python
    @app.route("/health", methods=["GET"])
    def healthcheck():
        return "success", 200
```

Flask interprets the returned tuple `("success", 200)` as `(body, status)`. Because the
body is a bare Python `str`, Flask applies its default response `Content-Type` of
`text/html; charset=utf-8` (observed in the output below — not inferred).

### Exact command

`curl -sS -i http://127.0.0.1:7777/health` (equivalently, a temporary Werkzeug
test-client probe capturing `status`, `content_type`, and `body`).

### Verbatim observed output

```
GET /health  -> status=200 content_type='text/html; charset=utf-8' body=b'success'
```

The body is the byte string `b'success'` — 7 bytes, no trailing newline, not wrapped in
JSON.

### Request-timing exclusion (observed, with causal reason)

`/health` emits **NO** request-timing `after_request` log line. This was verified **by
contrast**: a `GET /live` request produced an `after_request` timing line, while
`GET /health` produced none.

The cause is the `after_request` handler at `server.py:L272-L296`, whose guard explicitly
excludes `/health` (among other paths) at `server.py:L281`; the timing line itself is the
`LOG.d(...)` call at `server.py:L284`. The full handler body, reproduced with all its
clauses and logic (no elision):

```python
        # not logging /static call
        if (
            not request.path.startswith("/static")
            and not request.path.startswith("/admin/static")
            and not request.path.startswith("/_debug_toolbar")
            and not request.path.startswith("/git")
            and not request.path.startswith("/favicon.ico")
            and not request.path.startswith("/health")
        ):
            start_time = g.start_time or time.time()
            LOG.d(
                "%s %s %s %s %s, takes %s",
                request.remote_addr,
                request.method,
                request.path,
                request.args,
                res.status_code,
                time.time() - start_time,
            )
            newrelic.agent.record_custom_event(
                "HttpResponseStatus", {"code": res.status_code}
            )
        return res
```

Cause → effect: because the timing `LOG.d(...)` at `server.py:L284` is only reached when
the request path is **not** one of the excluded prefixes — and `/health` is excluded at
`server.py:L281` — a health probe never produces a timing line. `/live` is **not** in the
exclusion list, so it does produce one; that contrast is the observed proof.

### Sibling monitor endpoints (for completeness)

The monitor blueprint is registered at `app/monitor/base.py:L3`
(`monitor_bp = Blueprint(name="monitor", import_name=__name__, url_prefix="/")`), and its
handlers live in `app/monitor/views.py`:

- `GET /live -> b'live'` — handler `def live(): return "live"` at
  `app/monitor/views.py:L10-L12` (route `@monitor_bp.route("/live")`).
- `GET /git -> b'dev'` — handler `def git_sha1(): return SHA1` at
  `app/monitor/views.py:L5-L7` (route `@monitor_bp.route("/git")`); the build SHA1 defaults
  to `dev` in a local, non-CI build (`SHA1` imported from `app.build_info` at
  `app/monitor/views.py:L1`).
- `/exception` — deliberately raises an exception, handler `def test_exception():` at
  `app/monitor/views.py:L15-L17` (route `@monitor_bp.route("/exception")`), used to
  exercise error reporting.


---

## Q3 — What does the alias-creation API return, and what is persisted (which table, what values)?

This question has **two** endpoints that both create aliases; both are covered: the
primary **random** endpoint and the sibling **custom** endpoint.

### Direct answer

- `POST /api/alias/random/new` returns HTTP **201** with a JSON body describing the new
  alias. The fields are shaped by `serialize_alias_info_v2`, plus a top-level `alias` key
  added by the endpoint.
- The new alias is persisted to the database table named **`alias`**
  (`Alias.__tablename__ = "alias"` at `app/models.py:L1470`).
- The sibling `POST /api/v3/alias/custom/new` returns the **identical 201 JSON shape** and
  persists to the **same `alias` table**, differing only in how the local-part is chosen.

### Seeding (prerequisites) — via the real model factories

Creating an alias requires an authenticated user, an API key, and a default mailbox.
`User.create` (`app/models.py:L602-L613`) **auto-provisions a verified default mailbox**:
at `app/models.py:L611` it calls `Mailbox.create(user_id=..., email=..., verified=True)`,
and at `app/models.py:L613` it sets `user.default_mailbox_id = mb.id`. An API key is
created via `ApiKey.create` (`app/models.py:L2365`). These mirror the canonical test
seeding helpers `create_new_user` (`tests/utils.py:L17-L31`) and `random_token`
(`tests/utils.py:L66-L67`), and the `create_app` usage in `tests/conftest.py`
(`from server import create_app` at `tests/conftest.py:L20`; `app = create_app()` at
`tests/conftest.py:L23`). This is why the seeded user already has a usable default mailbox
(`id=1`) when the alias is created.

### Authentication

The API authenticates via the `Authentication` header carrying the API key code:
`app/api/base.py:L16-L18` reads `api_code = request.headers.get("Authentication")` (at
`app/api/base.py:L17`) and looks the key up with `ApiKey.get_by(code=api_code)`. So the
request sends the header `Authentication: <api_key.code>`.

### Exact values and citations (random endpoint)

- Route: `app/api/views/new_random_alias.py:L21`
  `@api_bp.route("/alias/random/new", methods=["POST"])` (the whole handler spans
  `L21-L117`); auth is enforced by the `@require_api_auth` decorator at
  `app/api/views/new_random_alias.py:L23`.
- Response assembly at `app/api/views/new_random_alias.py:L114-L117`:

```python
    return (
        jsonify(alias=alias.email, **serialize_alias_info_v2(get_alias_info_v2(alias))),
        201,
    )
```

- The JSON shape is defined by `serialize_alias_info_v2` at `app/api/serializer.py:L55-L93`;
  the top-level `alias` key is added by the endpoint (the `jsonify(alias=alias.email, ...)`
  above).
- Flow: API auth (`app/api/base.py:L16-L18`) → route → `Alias.create_new_random`
  (`app/models.py:L1721`), whose `mailbox_id` comes from `user.default_mailbox_id`
  (`app/models.py:L1753`) → serialize (`app/api/serializer.py:L55-L93`) → JSON; the row is
  committed to the `alias` table. The random local-part is generated by
  `generate_random_alias_email`, whose **definition is at `app/models.py:L1435`** (the AAP
  references `app/models.py:L1459`, which is a line **within** that function's body — the
  `LOG.d("generate email %s", random_email)` call; the `def` line is `L1435`).

### Exact command (random endpoint)

`curl -sS -X POST -H "Authentication: <api_key.code>" http://127.0.0.1:7777/api/alias/random/new`
(equivalently, a temporary Werkzeug test-client POST used to capture the response).

### Verbatim observed output — random endpoint JSON

```
STATUS: 201
{
  "alias": "rebels_cosies753@sl.local",
  "creation_date": "2026-07-07 22:19:53+00:00",
  "creation_timestamp": 1783462793,
  "disable_pgp": false,
  "email": "rebels_cosies753@sl.local",
  "enabled": true,
  "id": 2,
  "latest_activity": null,
  "mailbox": {"email": "probe_orv0pbfx@example.com", "id": 1},
  "mailboxes": [{"email": "probe_orv0pbfx@example.com", "id": 1}],
  "name": null,
  "nb_block": 0,
  "nb_forward": 0,
  "nb_reply": 0,
  "note": "onboarding test alias",
  "pinned": false,
  "support_pgp": false
}
```

### Persisted row — the `alias` table

The new alias is stored in the table named **`alias`**. The observed column values for the
new row (`id=2`):

```
id=2  email='rebels_cosies753@sl.local'  user_id=1  mailbox_id=1
enabled=True  note='onboarding test alias'  name=None
created_at=<Arrow [2026-07-07T22:19:53.776515+00:00]>  updated_at=None
automatic_creation=False  pinned=False  disable_pgp=False
```

How the persisted columns map to the JSON response fields:

- `email` ↔ JSON `email` **and** the top-level `alias` (both are `alias.email`).
- `note` ↔ JSON `note` (`onboarding test alias`).
- `created_at` ↔ JSON `creation_date` (string form) **and** `creation_timestamp`
  (Unix seconds); e.g. `created_at=<Arrow [2026-07-07T22:19:53.776515+00:00]>` corresponds
  to `creation_date="2026-07-07 22:19:53+00:00"` and `creation_timestamp=1783462793`.
- `mailbox_id=1` ↔ JSON `mailbox.id` (and the single entry in `mailboxes`), whose `email`
  is the seeded default mailbox `probe_orv0pbfx@example.com`.
- `enabled=True` ↔ JSON `enabled: true`; `pinned=False` ↔ `pinned: false`;
  `disable_pgp=False` ↔ `disable_pgp: false`; `name=None` ↔ `name: null`.
- `user_id=1` and `automatic_creation=False` are persisted columns not surfaced directly
  in this JSON body.

### Sibling custom endpoint

- Route: `POST /api/v3/alias/custom/new` at `app/api/views/new_custom_alias.py:L115`;
  handler `def new_custom_alias_v3():` at `app/api/views/new_custom_alias.py:L119`. The v2
  sibling route is at `app/api/views/new_custom_alias.py:L28`.
- It returns the **identical 201 JSON shape** and persists into the **same `alias` table**.
  Observed alias email: `email='my-custom-prefix.b7n3s9wh@sl.local'`.
- Difference vs the random endpoint — **local-part selection**: the **random** endpoint
  generates the local-part from server-side random words
  (`generate_random_alias_email`, `app/models.py:L1435`), whereas the **custom** endpoint
  uses a **client-provided prefix plus a signed suffix**. The suffix is signed by the
  `TimestampSigner` at `app/alias_suffix.py:L11`
  (`signer = itsdangerous.TimestampSigner(config.CUSTOM_ALIAS_SECRET)`); building a
  canonical custom request therefore requires that signed suffix.
- Illustrative command:
  `curl -sS -X POST -H "Authentication: <api_key.code>" -H "Content-Type: application/json" -d '{"alias_prefix":"my-custom-prefix","signed_suffix":"<signed>","mailbox_ids":[1]}' http://127.0.0.1:7777/api/v3/alias/custom/new`.

### Stability note

The response **shape is stable across ≥2 runs**. Only volatile fields differ between runs:
the randomly generated local-part (via `generate_random_alias_email`, `app/models.py:L1435`),
the `id`, the timestamps (`creation_date` / `creation_timestamp` / `created_at`), and the
`note`. The set of keys, their types, the `201` status, and the target `alias` table are
invariant.

### Historical schema rename (observed)

The live `alias` schema reveals a historical rename from `gen_email` → `alias`: the `id`
primary key still defaults from the sequence `gen_email_id_seq`, and the primary-key
constraint is still named `gen_email_pkey`, even though the table itself is now named
`alias` (`Alias.__tablename__ = "alias"` at `app/models.py:L1470`). These `gen_email_*`
names are **not** written anywhere in `app/models.py`; they are auto-generated by
PostgreSQL and were read directly from the running database catalog.

Exact command (run against the live schema on PostgreSQL `15.13`):

```
PGPASSWORD=test psql -h localhost -p 15432 -U test -d test \
  -c "SELECT pg_get_serial_sequence('alias','id') AS id_sequence;" \
  -c "SELECT conname AS pk_constraint FROM pg_constraint WHERE conrelid='alias'::regclass AND contype='p';" \
  -c "SELECT column_name, column_default FROM information_schema.columns WHERE table_name='alias' AND column_name='id';"
```

Verbatim observed output:

```
       id_sequence
-------------------------
 public.gen_email_id_seq
(1 row)

 pk_constraint
----------------
 gen_email_pkey
(1 row)

 column_name |            column_default
-------------+---------------------------------------
 id          | nextval('gen_email_id_seq'::regclass)
(1 row)
```

Why (cause → effect): the table was **created** as `gen_email` — with a serial `id` column
and a primary key — at `migrations/versions/5e549314e1e2_.py:L92-L101`
(`op.create_table('gen_email', sa.Column('id', sa.Integer(), autoincrement=True,
nullable=False), ..., sa.PrimaryKeyConstraint('id'), ...)`). PostgreSQL derives the implicit
object names from the table name at creation time, which is what produced the sequence
`gen_email_id_seq` (backing the serial `id`) and the constraint `gen_email_pkey`. A later
migration, `migrations/versions/2020_031711_e9395fe234a4_.py:L20-L21`
(`def upgrade(): op.rename_table("gen_email", "alias")`), renames **only the table** — not
its owned sequence, nor its primary-key constraint — so both retain their original
`gen_email_*` names, exactly as observed above. This is disclosed as an observed schema
detail.

### Why (cause → effect)

The `201` and the JSON body are produced by the `return (jsonify(...), 201)` tuple at
`app/api/views/new_random_alias.py:L114-L117`; the field set comes from
`serialize_alias_info_v2` (`app/api/serializer.py:L55-L93`) with the extra top-level
`alias` key merged in by the endpoint. The row lands in `alias` because
`Alias.__tablename__ = "alias"` (`app/models.py:L1470`), and `mailbox_id=1` is resolved
from `user.default_mailbox_id` (`app/models.py:L1753`) — which exists only because
`User.create` auto-created a verified default mailbox at `app/models.py:L611-L613`. The
custom endpoint reaches the same table and shape because it converges on the same
`Alias` model and serializer; only the local-part source differs.


---

## Q4 — If PostgreSQL is unavailable at startup, what error appears, and where does it break?

### Direct answer

With PostgreSQL stopped, the application **cannot even finish importing** — it breaks
**eagerly** at **`app/db.py:L12`** (`connection = engine.connect()`), a module-level
connection opened at **import time**. It fails **before** the body of `create_app()` ever
runs. This is a **hard, non-recoverable startup dependency**, not a lazily deferred error:
merely importing `server` (which the WSGI/dev entry points both do) is enough to trigger
it.

### Exact values and citations (the break point and the import chain)

- **Break point**: `app/db.py:L12` `connection = engine.connect()`. The engine is created
  just above at `app/db.py:L9-L11`:
  `engine = create_engine(config.DB_URI, connect_args={"application_name": config.DB_CONN_NAME})`.
- **Import chain** that reaches the eager connect (each arrow is one module import frame):

  1. `server.py:L31` — `from app.admin_model import (` →
  2. `app/admin_model.py:L11` — `from app import models, s3` →
  3. `app/models.py:L32` — `from app.db import Session` →
  4. `app/db.py:L12` — `connection = engine.connect()`  ← **fails here**

  Because step 4 executes at module top level (not inside a function), the exception
  propagates all the way back up the import chain and aborts the process during import of
  `server`.

### Exact command

Stop PostgreSQL, then attempt to import/start the app and capture stderr:
`CONFIG=tests/test.env python -c "import server"` (equivalently
`CONFIG=tests/test.env python server.py`).

### Verbatim observed output (both layers)

```
psycopg2.OperationalError: connection to server at "localhost" (::1), port 15432 failed: Connection refused
	Is the server running on that host and accepting TCP/IP connections?
connection to server at "localhost" (127.0.0.1), port 15432 failed: Connection refused
	Is the server running on that host and accepting TCP/IP connections?

sqlalchemy.exc.OperationalError: (psycopg2.OperationalError) connection to server at "localhost" (::1), port 15432 failed: Connection refused
	Is the server running on that host and accepting TCP/IP connections?
connection to server at "localhost" (127.0.0.1), port 15432 failed: Connection refused
	Is the server running on that host and accepting TCP/IP connections?

(Background on this error at: http://sqlalche.me/e/13/e3q8)
```

### Why (cause → effect)

The low-level driver error is `psycopg2.OperationalError`: the connect is attempted
against `localhost`, which resolves to **both** IPv6 `::1` and IPv4 `127.0.0.1`. In this
environment `/etc/hosts` lists **both** entries (`127.0.0.1 localhost` on the first line and
`::1 localhost` on the second), and the C-library resolver (`getaddrinfo`) returns the
**IPv6 address first** under its default RFC 3484 address-precedence rules — which is why
`::1` is attempted (and reported) before `127.0.0.1`, even though `127.0.0.1` is the first
line in `/etc/hosts`. In the canonical environment **both** address attempts fail with
**Connection refused**: there is no listener on port `15432`, and because `::1` is assigned
to the loopback interface (`/proc/net/if_inet6` shows `::1` on `lo`), `connect(::1:15432)`
returns
`ECONNREFUSED` ("Connection refused") rather than `EADDRNOTAVAIL`. SQLAlchemy then wraps
that driver exception into `sqlalchemy.exc.OperationalError` (note the
`(psycopg2.OperationalError)` prefix and the `http://sqlalche.me/e/13/e3q8` background
link — the `13` marks SQLAlchemy 1.3.x). Because the failing `engine.connect()` call runs
at **module import** (`app/db.py:L12`) rather than inside a request handler or inside
`create_app()`, the failure is **fatal to process startup** — the interpreter cannot even
finish importing `server`, so no HTTP server is ever started and no request is ever served.

The **per-address errno and the address ordering are environment-dependent** — they hinge
on whether `::1` is bound to the loopback interface and on the `localhost` resolution
order. On a host where `::1` is **not** assigned to loopback, the IPv6 attempt instead
returns **`Cannot assign requested address`** (`EADDRNOTAVAIL`) and IPv4 `127.0.0.1` may be
listed first. What is **invariant** across environments is the **two-layer
`psycopg2.OperationalError` → `sqlalchemy.exc.OperationalError`** wrapping, the **eager
break at `app/db.py:L12` at import time**, the port `15432`, and the
`sqlalche.me/e/13/e3q8` link.


---

## Environment caveats

Two environment caveats about dependency and service versions are recorded here for
reproducibility.
**Neither affects any of the four answers above** — none of them touches the port, health,
alias, or database-startup code paths.

- **`cbor2`.** The canonical warmed image ships the **exact lockfile pin `cbor2==5.2.0`**
  (verified: `import cbor2` succeeds and a `dumps`/`loads` round-trip works), so **no
  substitution is needed** in this environment. In a *from-scratch* Poetry build the
  `cbor2==5.2.0` sdist can instead fail to build — its `setuptools_scm`-derived version
  resolves to `0.0.0` and the sdist is rejected by pip/uv — in which case the nearest
  wheeled release `cbor2==5.4.6` is substituted; that scenario does **not** apply to the
  canonical warmed image. Either way, `cbor2` is used only for FIDO/WebAuthn paths and is
  **not** exercised by the port, health, alias, or database-startup code paths.
- **PostgreSQL version.** The investigation used **PostgreSQL 15.13** (on port `15432` to
  match the unmodified `tests/test.env` `DB_URI` — `SHOW server_version` returned
  `15.13 (Debian 15.13-0+deb12u1)`), whereas the CI canonical version is **PostgreSQL 13**
  (`.github/workflows/main.yml:L47` `image: postgres:13`). The observed alias schema, JSON
  responses, and the Q4 error text are not sensitive to this difference.

---

## Coverage checklist

Each named sub-part of the four questions, with its concrete value, `file:line`, observed
evidence, sibling variant, and causal reason:

- [x] **Port — both entry points.** Dev `127.0.0.1:7777` (`server.py:L588` `app.run(debug=True, port=7777)`, no `host` ⇒ Flask default `127.0.0.1`); prod `0.0.0.0:7777` (`Dockerfile:L47` `-b 0.0.0.0:7777`). Evidence: kernel socket lines `local=127.0.0.1:7777 state=LISTEN` and `local=0.0.0.0:7777 state=LISTEN`.
- [x] **Startup logs.** Gunicorn INFO lines (`Starting gunicorn 20.0.4`, `Listening at: http://0.0.0.0:7777`, `Using worker: sync`, two `Booting worker` lines) + the SL bootstrap block (`app/config.py:L68,L80,L262,L328`; `app/log.py:L67`; `app/utils.py:L17`). Werkzeug "Running on" banner **absent** — cause: `werkzeug` logger disabled at `app/log.py:L70-L71`.
- [x] **Health body.** Exactly `success` (raw ASCII, no trailing newline) — `server.py:L213-L215` `return "success", 200`. Evidence: `body=b'success'`.
- [x] **Health status + Content-Type.** `200` with `Content-Type: text/html; charset=utf-8` — observed `status=200 content_type='text/html; charset=utf-8'`. `/health` excluded from `after_request` timing at `server.py:L281` (timing `LOG.d` at `server.py:L284`), verified by contrast with `/live`.
- [x] **Alias JSON — both endpoints.** `201` JSON (verbatim 17-key body) from `POST /api/alias/random/new` (`app/api/views/new_random_alias.py:L21`, return tuple `L114-L117`, shape from `serialize_alias_info_v2` `app/api/serializer.py:L55-L93`). Sibling `POST /api/v3/alias/custom/new` (`app/api/views/new_custom_alias.py:L115`, handler `L119`) returns the identical shape; observed `email='my-custom-prefix.b7n3s9wh@sl.local'`.
- [x] **Table name.** `alias` — `Alias.__tablename__ = "alias"` at `app/models.py:L1470`; historical `gen_email` → `alias` rename observed in the live PostgreSQL catalog (sequence `gen_email_id_seq`, constraint `gen_email_pkey` — these auto-generated names are **not** in `app/models.py`; table created as `gen_email` at `migrations/versions/5e549314e1e2_.py:L92-L101`, renamed at `migrations/versions/2020_031711_e9395fe234a4_.py:L20-L21`).
- [x] **Persisted values.** The `id=2` row columns (`email`, `user_id=1`, `mailbox_id=1`, `enabled=True`, `note`, `name=None`, `created_at=<Arrow [2026-07-07T22:19:53.776515+00:00]>`, `updated_at=None`, `automatic_creation=False`, `pinned=False`, `disable_pgp=False`), mapped to the JSON fields. `mailbox_id=1` resolved from `user.default_mailbox_id` (`app/models.py:L1753`), created by `User.create` (`app/models.py:L611-L613`).
- [x] **PostgreSQL-down error.** Two-layer verbatim error: `psycopg2.OperationalError` (in the canonical environment **both** IPv6 `::1` and IPv4 `127.0.0.1` → Connection refused, `::1` listed first; port `15432`) wrapped by `sqlalchemy.exc.OperationalError` (`http://sqlalche.me/e/13/e3q8`). The per-address errno/ordering is environment-dependent (`::1` → `Cannot assign requested address` on a host where `::1` is not bound to loopback); the two-layer wrapping and the eager break at `app/db.py:L12` at import time are invariant.
- [x] **Break location.** `app/db.py:L12` `connection = engine.connect()` (eager, import-time), reached via `server.py:L31` → `app/admin_model.py:L11` → `app/models.py:L32` → `app/db.py:L12`.
- [x] **Both alias endpoints, both server entry points, happy + error paths.** Random + custom alias endpoints; dev + gunicorn entry points; health/alias happy paths + PostgreSQL-down error path — all exercised.
- [x] **Both environment caveats disclosed.** `cbor2==5.2.0` is present as the exact lockfile pin in the canonical warmed image (no substitution needed; a from-scratch build may substitute `5.4.6`); PostgreSQL `15.13` vs CI `13` (`.github/workflows/main.yml:L47`) — neither affects the four answers.

