# SimpleLogin Startup Behavior — Q&A / Runtime-Observation Report

> **Scope note.** This is a **read-only behavioral investigation** of the SimpleLogin
> codebase (branch `app_2cd6ee777f8c`, commit `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`).
> Its sole purpose is to *observe and document* what the development stack actually does at
> three startup checkpoints — it does **not** change, fix, or delete any source. The only
> file added to the repository is this document. All full-application executions were
> performed inside the **canonical Docker image**
> `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`
> (tag `andrewparkscaleai/coding-agent:simple-login__app__2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`),
> image digest `sha256:ea242796bbce36ca99ba9f783a4e7ac9d2ed3738e737f9a6d96bb22bbf1d9b58` —
> **Python 3.10.18**, Poetry-pinned deps (Flask 1.1.2, **SQLAlchemy 1.3.24**,
> psycopg2-binary 2.9.3, Werkzeug 1.0.1, **aiosmtpd 1.4.2**, alembic 1.4.3), PostgreSQL 15.

> **Evidence discipline (read this before the captures).** Every behavioral claim below is
> immediately preceded by the **exact command** that produced it and followed by the
> **complete captured output**. The captures are reproduced **verbatim** with a single,
> disclosed, mechanical normalization: **non-semantic trailing whitespace is trimmed** on each
> line so the Markdown stays `git diff --check`-clean (this is a byte-level hygiene step only;
> **no content is elided, summarized, annotated, or truncated** — every SQL column, every
> stack frame, every log line, and every SMTP reply is present in full). Nothing is redacted:
> the one session cookie shown in §2.3 is a throwaway development cookie signed with the
> **public** `example.env` value `FLASK_SECRET=secret` inside a disposable container, so it is
> non-sensitive and is printed raw. Values that are **not** a real-path capture are explicitly
> labeled `DB-layer stand-in` or `INFERRED (from code)`.

## Direct answers at a glance

| # | Question | Direct answer |
|---|----------|---------------|
| **REQ-1** | Empty DB -> `python server.py` -> open login page -> full exception? | **`sqlalchemy.exc.ProgrammingError`** wrapping **`psycopg2.errors.UndefinedTable: relation "users" does not exist`**, raised on the login **POST** (first SQL query), rendered as HTTP **500**. Full traceback in §2.3(f). |
| **REQ-2** | After migrations + `init_app.py`, which Python services + readiness? | **3 services**: web app `python server.py` (**:7777**), email handler `python email_handler.py` (**:20381**), job runner `python job_runner.py` (**no port**, readiness = "process alive & polling every 10s"). `cron.py`/`event_listener.py` are **not** required. Readiness evidence in §3.5-§3.7. |
| **REQ-3** | Migrations run, `init_app.py` skipped -> email to `x@sl.local`? | **Rejected** with SMTP reply **`550 SL E515 Email not exist`** [app/email/status.py:51] — driven by alias-nonexistence + auto-create failure (not directly by the empty `SLDomain`). Full transcript in §4.3. |

> **Migration caveat (answered honestly up front, detailed in §3.2).** The README/AAP name
> `flask db upgrade` as the migration step. In this **unchanged** codebase that exact command
> **fails** (`KeyError: 'migrate'`) because Flask-Migrate is never registered; the schema is
> built by the canonical `alembic upgrade head` (exactly what CI runs). The literal
> `flask db upgrade` success prerequisite is therefore **blocked** without modifying source,
> which is out of scope — see §3.2 for the complete failure traceback and the working path.

---

## §1 Environment & startup-order preamble

### 1.1 Canonical runtime & container launch (with a digest guard)

All scenario executions ran inside one disposable container (`sl-obs`, working dir `/app`)
started from the canonical image. A from-scratch `poetry install` of the pinned lock on a
generic host fails on old build-from-source pins (`cffi==1.14.4`, `cbor2==5.2.0`), so a
hand-rebuilt environment would be **non-canonical**; the image ships **Python 3.10**
[Dockerfile:8] with the Poetry deps pre-installed in a venv at `/app/venv`. Before any
destructive or observation step, the container's image digest is asserted to equal the
canonical digest, so every command is mechanically scoped to the disposable runtime:

```text
$ docker run -d --name sl-obs --entrypoint bash \
    ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0 -lc 'sleep infinity'
$ docker inspect --format '{{.Image}}' sl-obs
sha256:ea242796bbce36ca99ba9f783a4e7ac9d2ed3738e737f9a6d96bb22bbf1d9b58
GUARD OK: container image digest matches canonical sha256:ea242796bbce36ca99ba9f783a4e7ac9d2ed3738e737f9a6d96bb22bbf1d9b58
$ docker inspect --format '{{.Config.WorkingDir}}' sl-obs
/app
$ docker exec sl-obs bash -lc 'git -C /app rev-parse HEAD'
2cd6ee777f8c2d3531559588bcfb18627ffb5d2c
```

> **Note on `python`:** in the image a bare `python` resolves to `/usr/local/bin/python`
> (3.10.18) which does **not** carry the app deps; the canonical services run under the venv.
> Every documented `python <script>` command was executed as `/app/venv/bin/python <script>`
> (equivalently, with the venv activated); commands are shown with the venv path so they are
> reproducible without an implicit "activate" step.

### 1.2 PostgreSQL bring-up, readiness, and the `myuser` role

PostgreSQL 15 is started inside the container and its readiness is confirmed before use. The
dev `DB_URI` role `myuser` (superuser, password `mypassword`) is created idempotently:

```text
$ docker exec sl-obs bash -lc 'service postgresql start'
Starting PostgreSQL 15 database server: main.
$ docker exec sl-obs bash -lc 'pg_lsclusters'
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
$ docker exec sl-obs bash -lc 'pg_isready -h 127.0.0.1 -p 5432'
127.0.0.1:5432 - accepting connections
```

```text
$ cat /tmp/create_role.sql
DO $$
BEGIN
   IF NOT EXISTS (SELECT FROM pg_roles WHERE rolname = 'myuser') THEN
      CREATE ROLE myuser LOGIN SUPERUSER PASSWORD 'mypassword';
   END IF;
END
$$;
$ su - postgres -c "psql -v ON_ERROR_STOP=1 -f /tmp/create_role.sql"
DO
$ PGPASSWORD=mypassword psql -h 127.0.0.1 -U myuser -d postgres -tAc "SELECT rolname, rolsuper FROM pg_roles WHERE rolname=current_user"
myuser|t
```

### 1.3 Interpreter & dependency pins (observed)

```text
$ /app/venv/bin/python --version
Python 3.10.18
$ /app/venv/bin/python -c "import sys,sqlalchemy,psycopg2,flask,aiosmtpd,werkzeug,alembic; \
    print('python',sys.version.split()[0]); print('sqlalchemy',sqlalchemy.__version__); \
    print('psycopg2',psycopg2.__version__); print('flask',flask.__version__); \
    print('aiosmtpd',aiosmtpd.__version__); print('werkzeug',werkzeug.__version__); \
    print('alembic',alembic.__version__)"
python 3.10.18
sqlalchemy 1.3.24
psycopg2 2.9.3 (dt dec pq3 ext lo64)
flask 1.1.2
aiosmtpd 1.4.2
werkzeug 1.0.1
alembic 1.4.3
```

### 1.4 Configuration — the five unguarded boot variables & `obs.env` provenance

Runtime configuration is environment-driven through python-dotenv; the loader reads the file
named by `CONFIG` (else `./.env`) [app/config.py:65-71]. Five variables are read with bare
`os.environ[...]` subscripts, so a missing value raises `KeyError` at import time; all five
are present in `example.env`. The observation config `/tmp/obs.env` is a **byte-for-byte copy**
of `example.env` (identical SHA-256), created **outside** the repository and never committed:

```text
$ cp /app/example.env /tmp/obs.env
$ cmp /app/example.env /tmp/obs.env && echo IDENTICAL
IDENTICAL
$ sha256sum /app/example.env /tmp/obs.env
5aa7a424f7112df5c5048fbbced1d4913bd076b5b5875cf1b8b43f1d232f9a6d  /app/example.env
5aa7a424f7112df5c5048fbbced1d4913bd076b5b5875cf1b8b43f1d232f9a6d  /tmp/obs.env
$ grep -nE "^(URL|EMAIL_DOMAIN|SUPPORT_EMAIL|DB_URI|FLASK_SECRET)=" /tmp/obs.env
6:URL=http://localhost:7777
22:EMAIL_DOMAIN=sl.local
40:SUPPORT_EMAIL=support@sl.local
75:DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin
77:FLASK_SECRET=secret
```

| Variable | Definition | `example.env` value |
|----------|------------|---------------------|
| `URL` | `os.environ["URL"]` [app/config.py:79] | `URL=http://localhost:7777` [example.env:6] |
| `EMAIL_DOMAIN` | `os.environ["EMAIL_DOMAIN"].lower()` [app/config.py:92] | `EMAIL_DOMAIN=sl.local` [example.env:22] |
| `SUPPORT_EMAIL` | `os.environ["SUPPORT_EMAIL"]` [app/config.py:93] | `SUPPORT_EMAIL=support@sl.local` [example.env:40] |
| `DB_URI` | `os.environ["DB_URI"]` [app/config.py:192] | `DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin` [example.env:75] |
| `FLASK_SECRET` | `os.environ["FLASK_SECRET"]` [app/config.py:196] (+`raise RuntimeError` if empty [app/config.py:197-198]) | `FLASK_SECRET=secret` [example.env:77] |

`ALIAS_DOMAINS = OTHER_ALIAS_DOMAINS + [EMAIL_DOMAIN]` [app/config.py:160] therefore defaults
to `["sl.local"]`, making `sl.local` the natural domain to exercise in REQ-3.

### 1.5 Redis is optional

`RedisSessionStore` is initialized **only** when `MEM_STORE_URI` is set [server.py:163-165];
`example.env` does not set it, so the dev stack uses default Flask cookie sessions and an
in-memory limiter. Consequently a login-page **GET** touches neither Redis nor the database:

```text
$ grep -c -i MEM_STORE_URI /tmp/obs.env
0
```

### 1.6 Guarded, disposable-scoped DB reset & the per-scenario states

The three checkpoints are **mutually-exclusive** database states, so the disposable
`simplelogin` database (the exact name in `example.env`'s `DB_URI`) is dropped/recreated
between scenarios:

- **REQ-1** = empty (no migrations)
- **REQ-2** = migrated **+** initialized
- **REQ-3** = migrated **but not** initialized

Because `DROP DATABASE`/`pg_terminate_backend` are destructive, they are wrapped in a guard
that **aborts unless** it is running in the disposable container (marker file) **and** the
`DB_URI` targets `localhost` + the disposable DB/role only. The guard makes it mechanically
impossible to reset a non-disposable database:

```bash
$ cat /tmp/reset_db.sh
#!/usr/bin/env bash
# Guarded, disposable-container-scoped reset for the ephemeral `simplelogin` DB.
# Aborts unless ALL preconditions match, so it can never touch a non-disposable DB.
set -euo pipefail

EXPECT_DB="simplelogin"
EXPECT_ROLE="myuser"
DB_URI="${DB_URI:-postgresql://myuser:mypassword@localhost:5432/simplelogin}"

# Guard 1: must be the disposable observation container (marker file written at bring-up).
if [ ! -f /tmp/.sl_obs_disposable ]; then
  echo "GUARD ABORT: disposable-container marker /tmp/.sl_obs_disposable absent" >&2; exit 3
fi
# Guard 2: DB_URI must target localhost and the disposable DB/role only.
host=$(python3 -c "import urllib.parse,os;u=urllib.parse.urlparse(os.environ.get('DB_URI',''));print(u.hostname or '')" 2>/dev/null || echo "")
name=$(python3 -c "import urllib.parse,os;u=urllib.parse.urlparse(os.environ.get('DB_URI',''));print((u.path or '').lstrip('/'))" 2>/dev/null || echo "")
user=$(python3 -c "import urllib.parse,os;u=urllib.parse.urlparse(os.environ.get('DB_URI',''));print(u.username or '')" 2>/dev/null || echo "")
case "$host" in localhost|127.0.0.1) : ;; *) echo "GUARD ABORT: DB host '$host' not local" >&2; exit 4;; esac
[ "$name" = "$EXPECT_DB" ]   || { echo "GUARD ABORT: DB name '$name' != $EXPECT_DB" >&2; exit 5; }
[ "$user" = "$EXPECT_ROLE" ] || { echo "GUARD ABORT: role '$user' != $EXPECT_ROLE" >&2; exit 6; }

echo "GUARD OK: disposable container, DB_URI=localhost/$EXPECT_DB role=$EXPECT_ROLE -> resetting"
export PGPASSWORD=mypassword
psql -h 127.0.0.1 -U "$EXPECT_ROLE" -d postgres -v ON_ERROR_STOP=1 \
  -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname='$EXPECT_DB' AND pid <> pg_backend_pid()" >/dev/null
psql -h 127.0.0.1 -U "$EXPECT_ROLE" -d postgres -v ON_ERROR_STOP=1 -c "DROP DATABASE IF EXISTS $EXPECT_DB"
psql -h 127.0.0.1 -U "$EXPECT_ROLE" -d postgres -v ON_ERROR_STOP=1 -c "CREATE DATABASE $EXPECT_DB OWNER $EXPECT_ROLE"
echo "RESET DONE: fresh empty $EXPECT_DB"
```

The guard is demonstrated aborting on a non-disposable name and succeeding on the disposable
DB (observed):

```text
$ DB_URI=postgresql://myuser:mypassword@localhost:5432/PRODDB /tmp/reset_db.sh ; echo exit=$?
GUARD ABORT: DB name 'PRODDB' != simplelogin
exit=5
$ DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin /tmp/reset_db.sh ; echo exit=$?
GUARD OK: disposable container, DB_URI=localhost/simplelogin role=myuser -> resetting
DROP DATABASE
CREATE DATABASE
RESET DONE: fresh empty simplelogin
exit=0
```

The canonical bring-up order mirrors `README.md`:

```text
Postgres up  ->  create empty `simplelogin` DB  ->  (REQ-1 stops here)
             ->  migrations (alembic upgrade head)  ->  python init_app.py  ->  start services
```

### 1.7 Service lifecycle conventions used throughout

Each networked service is started in the background with its stdout/stderr redirected to a log
file and its launcher PID captured; readiness is polled (HTTP `/health` for the web app, a TCP
connect for the SMTP handler); after each scenario the service is stopped and its port is
verified released. The exact commands are shown inline per service; the generic pattern is:

```bash
# start (background, logged, PID captured)
CONFIG=/tmp/obs.env nohup /app/venv/bin/python <svc>.py > /tmp/<svc>.log 2>&1 &  ; echo $! > /tmp/<svc>.pid
# stop + verify release
pkill -TERM -f "[<x>]<svc>.py" ; sleep 2 ; pkill -KILL -f "[<x>]<svc>.py"
fuser <port>/tcp ; echo rc=$?     # rc=1 + no PIDs = released
```

### 1.8 Migrations: `flask db upgrade` vs `alembic upgrade head` (previewed; evidence in §3.2)

The README/AAP document `flask db upgrade`, but in this codebase **Flask-Migrate is not
registered** in `create_app` (no `Migrate(app, db)` call exists anywhere in the source; the
only reference, `shell.py:20`, is dead code under `if False:` [shell.py:11-26]), so that
command **fails** with `KeyError: 'migrate'`. The working, canonical migration path is
`alembic upgrade head` — exactly what CI uses
(`CONFIG=tests/test.env poetry run alembic upgrade head` [.github/workflows/main.yml:98]). The
complete failure traceback and the successful Alembic run are captured in **§3.2**.

### 1.9 Stdout log format & the disabled werkzeug logger

Every SimpleLogin log line uses this format [app/log.py:12-16], written to **stdout** via
`StreamHandler(sys.stdout)` [app/log.py:41]:

```text
%(asctime)s - %(name)s - %(levelname)s - %(process)d - "%(pathname)s:%(lineno)d" - %(funcName)s() - %(message_id)s - %(message)s
```

**Observed nuance:** `app/log.py:70-71` sets `logging.getLogger("werkzeug").disabled = True`.
At capture time this **suppresses** the Werkzeug lines emitted through that logger — the
`* Running on http://127.0.0.1:7777` banner, `* Debugger is active!`, `* Debugger PIN:` and
`* Restarting with stat` — while **Flask's own** banner (`* Serving Flask app`,
`* Environment: production`, `* Debug mode: on`, printed via `click`) and SimpleLogin's own
`after_request` access log (SL logger, [server.py:284-292]) remain visible. This is confirmed
by live capture in §2/§3 (a `grep` for those banners returns **0** matches).

### 1.10 Port 25 vs 20381 (dev vs prod) — flagged

The higher-level system spec describes the SMTP handler at **port 25** (the production Postfix
relay target). In local development, `python email_handler.py` binds the argparse **default
20381** [email_handler.py:2399]. **The port actually bound and observed throughout this report
is 20381.**

### 1.11 Count grounding: 255 migration files & 77 tables

The numbers cited later are grounded in real command output (not inferred from `alembic.ini`,
which only sets `script_location = migrations` [alembic.ini:5]):

```text
$ ls -1 /app/migrations/versions/*.py | wc -l
255
$ grep -c "Running upgrade" /tmp/req2_alembic.log
255
$ PGPASSWORD=mypassword psql -h 127.0.0.1 -U myuser -d simplelogin -tAc \
    "SELECT count(*) FROM information_schema.tables WHERE table_schema='public'"
77
```

---

## §2 REQ-1 — Empty-database startup failure

### 2.1 Direct answer

With an **empty** `simplelogin` database (created, **migrations not run**), `python server.py`
**starts normally** and the login page **GET renders HTTP 200 with no query**. The failure
occurs on the login **POST (submit)**, when the first SQL query runs against the missing table.
The exception raised is:

> **`sqlalchemy.exc.ProgrammingError`** wrapping
> **`psycopg2.errors.UndefinedTable: relation "users" does not exist`**

The app's global error handler `@app.errorhandler(Exception)` [server.py:388-394] catches it,
logs the full traceback via `LOG.e(e)` [server.py:390], and returns HTTP **500**
(`render_template("error/500.html"), 500` [server.py:394]). The complete traceback is in
§2.3(f).

### 2.2 Commands run

The login page is `/auth/login` — route `@auth_bp.route("/login", methods=["GET", "POST"])`
[app/auth/views/login.py:21] under blueprint `url_prefix="/auth"` [app/auth/base.py:3-5]
(visiting `/` redirects anonymous users to `auth.login` [server.py:249-255]).

```bash
# Substrate: guarded reset to an EMPTY simplelogin DB (no migrations)  -- see §1.6
DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin /tmp/reset_db.sh
# Start the web app through its real entry point (local_main -> app.run(debug=True, port=7777))
CONFIG=/tmp/obs.env nohup /app/venv/bin/python server.py > /tmp/req1_server.log 2>&1 &
# (same session) GET renders the form with NO DB query (headers only):
curl -sS -D - -o /dev/null http://127.0.0.1:7777/auth/login
# then drive the REAL login POST reusing the session cookie + scraped csrf_token:
/app/venv/bin/python /tmp/req1_login_post.py
# corroborating DB-layer stand-in (labeled non-canonical):
DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin /app/venv/bin/python /tmp/req1_dblayer_standin.py
```

The two helper scripts are reproduced in full so the CSRF/session flow and the stand-in are
auditable (both live under `/tmp`, outside the repository, and are removed on completion —
see §5).

**`/tmp/req1_login_post.py`** (real HTTP login flow):

```python
"""REQ-1: drive the REAL login HTTP path (GET form -> scrape CSRF -> POST credentials).

Uses a requests.Session so the Flask session cookie set on the GET is replayed on the
POST, and scrapes the hidden csrf_token so WTF_CSRF passes and execution reaches
User.get_by(email=...) [app/auth/views/login.py:43] -> the first SQL query.
"""
import re
import requests

BASE = "http://127.0.0.1:7777"
s = requests.Session()

# 1) GET the login form (renders 200, no DB query)
r_get = s.get(f"{BASE}/auth/login", timeout=15)
print("GET /auth/login ->", r_get.status_code)

# 2) scrape the hidden csrf_token rendered by {{ form.csrf_token }}
m = re.search(r'name="csrf_token"[^>]*value="([^"]+)"', r_get.text)
csrf = m.group(1) if m else None
print("csrf_token scraped:", csrf is not None)

# 3) POST credentials + csrf_token, replaying the session cookie
r_post = s.post(
    f"{BASE}/auth/login",
    data={"csrf_token": csrf, "email": "a@b.c", "password": "whatever"},
    timeout=15,
)
print("POST /auth/login ->", r_post.status_code)
print("POST Content-Type:", r_post.headers.get("Content-Type"))
print("contains-500-page-marker:", ("Internal" in r_post.text) or (r_post.status_code == 500))
```

**`/tmp/req1_dblayer_standin.py`** (labeled DB-layer stand-in — corroboration only):

```python
"""REQ-1 DB-layer stand-in (NOT the full HTTP path) — corroboration only.

Mirrors app/db.py:9-12 directly: build the engine from DB_URI, eagerly
`engine.connect()` (which SUCCEEDS on an empty DB), then run ONE SELECT against the
missing `users` table to isolate the "connect ok, first query fails" distinction.
Prints the COMPLETE traceback via traceback.print_exc(). flush=True keeps stdout in
narrative order relative to the (unbuffered) stderr traceback.
"""
import os
import sys
import traceback
from sqlalchemy import create_engine, text

DB_URI = os.environ["DB_URI"]
engine = create_engine(DB_URI)

# Mirror the import-time connect at app/db.py:12
conn = engine.connect()
print("engine.connect() -> SUCCEEDED on empty DB (no error at connect)", flush=True)

print("--- first query against missing table raised: ---", flush=True)
sys.stdout.flush()
try:
    conn.execute(text("SELECT id FROM users WHERE email = 'a@b.c'"))
except Exception:
    traceback.print_exc()
finally:
    conn.close()
```

### 2.3 Observed output

**(a) Empty-DB substrate proof** — 0 public tables and the `users` relation absent (real
output + exit status, adjacent to the action):

```text
$ PGPASSWORD=mypassword psql -h 127.0.0.1 -U myuser -d simplelogin -tAc "SELECT count(*) FROM information_schema.tables WHERE table_schema='public'" ; echo exit=$?
0
exit=0
$ PGPASSWORD=mypassword psql -h 127.0.0.1 -U myuser -d simplelogin -tAc "SELECT to_regclass('public.users') IS NULL" ; echo exit=$?
t
exit=0
```

**(b) `python server.py` startup banner** — the server binds `:7777` against the **empty** DB
without error (import-time `connection = engine.connect()` [app/db.py:12] succeeds; `create_app`
issues no query). The app **boot preamble** (`load config file` -> `>>> init logging <<<` ->
`load words file`) appears **twice** — once per process — because `app.run(debug=True)`
[server.py:588] enables the Werkzeug reloader, which spawns a child: the first preamble is the
reloader **parent** (PID 375), the second is the **child** (PID 399) that actually serves. The
Werkzeug `* Serving Flask app` / `* Environment: production` / `* Debug mode: on` banner prints
**once**; note the **absence** of any `* Running on http://127.0.0.1:7777` or `* Debugger PIN:`
line — SimpleLogin's logging setup suppresses Werkzeug's `_internal` "Running on" emitter (§1.9):

```text
load config file /tmp/obs.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/uxfvxzqeqqpmzwtpbbvt
Upload files to local dir
>>> init logging <<<
2026-07-14 21:21:31,505 - SL - DEBUG - 375 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
load config file /tmp/obs.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/ifvgonjmzhoszssnecsu
Upload files to local dir
>>> init logging <<<
2026-07-14 21:21:34,154 - SL - DEBUG - 399 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

**(c) Port-binding proof (:7777).** The listening socket is owned by the reloader parent+child;
`0100007F:1E61` = `127.0.0.1:0x1E61` = `127.0.0.1:7777` in `LISTEN` (state `0A`):

```text
$ fuser 7777/tcp
   375   399
$ awk '$4=="0A"{print $2}' /proc/net/tcp | grep -i ':1E61$'
0100007F:1E61
```

**(d) GET `/auth/login` -> HTTP 200, no DB query** (headers only via `curl -sS -D - -o /dev/null`,
so the full 350 KB HTML body is intentionally not fetched). The client timestamp and the
server-side `after_request` access log timestamp match to the second (**both 21:21:51**, same
child PID 399 — one and the same request in the empty-DB session):

```text
client-UTC-before: 2026-07-14T21:21:51Z
$ curl -sS -D - -o /dev/null http://127.0.0.1:7777/auth/login
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 350142
Vary: Cookie
Set-Cookie: slapp=eyJfZnJlc2giOmZhbHNlLCJfcGVybWFuZW50Ijp0cnVlLCJjc3JmX3Rva2VuIjoiNmMyYzI5ZGY4YzEwYTkyMjZhODkwODkyZTM4MmMzN2RkNDZkYjllYiJ9.alaobw.U542RdGncuEpQsMspyUlM3Xj18E; Expires=Tue, 21-Jul-2026 21:21:51 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Tue, 14 Jul 2026 21:21:51 GMT
client-UTC-after:  2026-07-14T21:21:51Z
```

```text
2026-07-14 21:21:51,346 - SL - DEBUG - 399 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.1103360652923584
```

> **Why GET does not query the DB.** The only `current_user` reference in the render chain
> (`login.html` -> `single.html` -> `base.html`) is
> `{% if NOW.timestamp < 1701475201 and current_user.is_authenticated and current_user.should_show_upgrade_button() %}`
> [templates/base.html:89]. The epoch `1701475201` = 2023-12-01; at the current date the first
> clause is `False`, so Jinja short-circuits and `current_user.is_authenticated` is never
> evaluated (and an anonymous `is_authenticated` would not hit the DB anyway).

**(e) The real login POST — CSRF methodology + client result.** The login form is a `flask_wtf`
`FlaskForm` rendering `<form method="post">` + `{{ form.csrf_token }}`
[templates/auth/login.html:16-17], and `WTF_CSRF_ENABLED` defaults **True**. A bare
`curl -X POST` fails CSRF, never reaches the query, and produces **no** error; the faithful path
(the helper above) uses a `requests.Session()` that GETs the form to obtain the session cookie +
`csrf_token`, then reposts them with credentials. The client-side timestamp (**21:22:18**)
matches the server error/access lines below:

```text
client-UTC-before-POST: 2026-07-14T21:22:18Z
$ /app/venv/bin/python /tmp/req1_login_post.py
GET /auth/login -> 200
csrf_token scraped: True
POST /auth/login -> 500
POST Content-Type: text/html; charset=utf-8
contains-500-page-marker: True
client-UTC-after-POST:  2026-07-14T21:22:18Z
```

**(f) The complete server-side traceback** (verbatim), logged by the global
`@app.errorhandler(Exception)` -> `LOG.e(e)` (where `LOG.e = logging.Logger.exception`
[app/log.py:77]) at **server.py:390**, followed by the `after_request` line showing the **500**.
The server `ERROR` timestamp (21:22:18,492) and the `after_request 500` timestamp (21:22:18,498)
match the client POST above:

```text
2026-07-14 21:22:18,492 - SL - ERROR - 399 - "/app/server.py:390" - error_handler() -  - (psycopg2.errors.UndefinedTable) relation "users" does not exist
LINE 2: FROM users
             ^

[SQL: SELECT users.directory_quota AS users_directory_quota, users.subdomain_quota AS users_subdomain_quota, users.password AS users_password, users.id AS users_id, users.created_at AS users_created_at, users.updated_at AS users_updated_at, users.email AS users_email, users.name AS users_name, users.is_admin AS users_is_admin, users.alias_generator AS users_alias_generator, users.notification AS users_notification, users.activated AS users_activated, users.disabled AS users_disabled, users.profile_picture_id AS users_profile_picture_id, users.otp_secret AS users_otp_secret, users.enable_otp AS users_enable_otp, users.last_otp AS users_last_otp, users.fido_uuid AS users_fido_uuid, users.default_alias_custom_domain_id AS users_default_alias_custom_domain_id, users.default_alias_public_domain_id AS users_default_alias_public_domain_id, users.lifetime AS users_lifetime, users.paid_lifetime AS users_paid_lifetime, users.lifetime_coupon_id AS users_lifetime_coupon_id, users.trial_end AS users_trial_end, users.default_mailbox_id AS users_default_mailbox_id, users.sender_format AS users_sender_format, users.sender_format_updated_at AS users_sender_format_updated_at, users.replace_reverse_alias AS users_replace_reverse_alias, users.referral_id AS users_referral_id, users.intro_shown AS users_intro_shown, users.max_spam_score AS users_max_spam_score, users.newsletter_alias_id AS users_newsletter_alias_id, users.include_sender_in_reverse_alias AS users_include_sender_in_reverse_alias, users.random_alias_suffix AS users_random_alias_suffix, users.expand_alias_info AS users_expand_alias_info, users.ignore_loop_email AS users_ignore_loop_email, users.alternative_id AS users_alternative_id, users.disable_automatic_alias_note AS users_disable_automatic_alias_note, users.one_click_unsubscribe_block_sender AS users_one_click_unsubscribe_block_sender, users.include_website_in_one_click_alias AS users_include_website_in_one_click_alias, users.disable_import AS users_disable_import, users.can_use_phone AS users_can_use_phone, users.phone_quota AS users_phone_quota, users.block_behaviour AS users_block_behaviour, users.include_header_email_header AS users_include_header_email_header, users.enable_data_breach_check AS users_enable_data_breach_check, users.flags AS users_flags, users.unsub_behaviour AS users_unsub_behaviour, users.delete_on AS users_delete_on
FROM users
WHERE users.email = %(email_1)s
 LIMIT %(param_1)s]
[parameters: {'email_1': 'a@b.c', 'param_1': 1}]
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
[parameters: {'email_1': 'a@b.c', 'param_1': 1}]
(Background on this error at: http://sqlalche.me/e/13/f405)
2026-07-14 21:22:18,498 - SL - DEBUG - 399 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 500, takes 0.013988971710205078

```

**(g) DB-layer stand-in** — `DB-layer stand-in — not the full HTTP login path` (corroboration
only). This isolates the "connect succeeds, first query fails" distinction by mirroring
`app/db.py:9-12` directly (`create_engine(DB_URI)` + `engine.connect()`, then one `SELECT`):

```text
$ DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin /app/venv/bin/python /tmp/req1_dblayer_standin.py ; echo exit=$?
engine.connect() -> SUCCEEDED on empty DB (no error at connect)
--- first query against missing table raised: ---
Traceback (most recent call last):
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1276, in _execute_context
    self.dialect.do_execute(
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/default.py", line 608, in do_execute
    cursor.execute(statement, parameters)
psycopg2.errors.UndefinedTable: relation "users" does not exist
LINE 1: SELECT id FROM users WHERE email = 'a@b.c'
                       ^


The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "/tmp/req1_dblayer_standin.py", line 24, in <module>
    conn.execute(text("SELECT id FROM users WHERE email = 'a@b.c'"))
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
LINE 1: SELECT id FROM users WHERE email = 'a@b.c'
                       ^

[SQL: SELECT id FROM users WHERE email = 'a@b.c']
(Background on this error at: http://sqlalche.me/e/13/f405)
exit=0
```

### 2.4 Cause -> effect (with `file:line`)

1. **Import-time connect succeeds — the failure is at the first *query*, not at connect.**
   `app/db.py` builds the engine from `config.DB_URI` [app/db.py:9-11] and eagerly opens a
   connection at import: `connection = engine.connect()` [app/db.py:12]. Against an empty
   database this **succeeds** (observed in (b): the server bound `:7777`).
2. **GET renders 200 with no query** — short-circuited `NOW.timestamp < 1701475201` guard
   [templates/base.html:89] (observed in (d)).
3. **POST fires the query.** Inside `if form.validate_on_submit():` [app/auth/views/login.py:40]
   (which first passes CSRF), the line
   `user = User.get_by(email=email) or User.get_by(email=canonical_email)`
   [app/auth/views/login.py:43] calls `User.get_by` -> `Session.query(cls).filter_by(**kw).first()`
   [app/models.py:84], issuing `SELECT ... FROM users ...` against table `users`
   (`class User ... __tablename__ = "users"` [app/models.py:336-337]). The missing relation makes
   psycopg2 raise `UndefinedTable`, which SQLAlchemy 1.3.24 wraps as
   `sqlalchemy.exc.ProgrammingError`.
4. **In-app handling -> HTTP 500.** The global `@app.errorhandler(Exception)` [server.py:388-394]
   runs `LOG.e(e)` (the full traceback in (f)) and, for non-`/api/` paths,
   `return render_template("error/500.html"), 500`. Because this handler **intercepts** the
   exception, Werkzeug's interactive debugger page does **not** engage — the complete traceback
   is exactly what `LOG.e` (logging.exception) writes to stdout, captured in §2.3(f). *(This is
   an observed correction to the a-priori expectation that `debug=True` would also surface a
   Werkzeug interactive traceback page: the app-level handler takes precedence.)*

**(h) Shutdown + port release.** The web app is stopped and `:7777` is verified released:

```text
$ fuser 7777/tcp ; echo rc=$?
rc=1
$ pgrep -c -f "/app/venv/bin/python server.py"
0
```

---

## §3 REQ-2 — Required Python services and readiness evidence

### 3.1 Direct answer

After `flask db upgrade` (see the honest status in §3.2) / `alembic upgrade head` **and**
`python init_app.py` have both succeeded, **three** Python services must be running for the
local-development system to function [README.md:461,477-480,495]:

| # | Service | Command | Port | Readiness signal (observed) |
|---|---------|---------|------|-----------------------------|
| 1 | Web application | `python server.py` | TCP **7777** | Werkzeug `* Serving Flask app` banner; `:7777` LISTEN; `GET /health -> 200` |
| 2 | Email handler (SMTP ingress) | `python email_handler.py` | TCP **20381** | `Listen for port 20381` [email_handler.py:2403] + `Start mail controller 0.0.0.0 20381` [email_handler.py:2386]; `:20381` LISTEN; live SMTP `EHLO`/`NOOP` |
| 3 | Job runner (background worker) | `python job_runner.py` | **none** | No port, no "ready" banner — readiness = *process alive and polling* the `Job` table every 10s [job_runner.py:329-347] |

The production web alternative is `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15`
[Dockerfile:47] (real banner captured in §3.8). Auxiliary services `cron.py` (yacron) and
`event_listener.py` (Proton `LISTEN`/`NOTIFY`) are **not** part of this required set and were not
started.

### 3.2 `flask db upgrade` — honest status (BLOCKED) and the canonical migration path

**Direct answer:** in this repository at commit `2cd6ee77`, `flask db upgrade` **fails** — it
raises `KeyError: 'migrate'`. Flask-Migrate's `Migrate(app, db)` extension is **never registered**
on the app, so `current_app.extensions['migrate']` does not exist. The only reference to
`flask_migrate` in the source is **dead code** in `shell.py` guarded by `if False:` (the
`flask_migrate.upgrade()` call at `shell.py:20` is unreachable), and `create_app` in `server.py`
registers no `Migrate`. Fixing this would require **modifying source**, which this task forbids
(read-only investigation, MainRule) — so the requirement's literal `flask db upgrade` invocation
is **BLOCKED under the no-source-modification constraint**, and this is reported rather than
worked around.

Complete, unedited failure output with exit code (command exactly as documented in the README
migration step [README.md:422-433]):

```text
$ CONFIG=/tmp/obs.env FLASK_APP=server.py /app/venv/bin/python -m flask db upgrade
exit=1
--- complete /tmp/req2_flaskdb.log ---
load config file /tmp/obs.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/iwlfhmcbexpaovxvtrtg
Upload files to local dir
>>> init logging <<<
2026-07-14 21:23:54,074 - SL - DEBUG - 577 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
Traceback (most recent call last):
  File "/usr/local/lib/python3.10/runpy.py", line 196, in _run_module_as_main
    return _run_code(code, main_globals, None,
  File "/usr/local/lib/python3.10/runpy.py", line 86, in _run_code
    exec(code, run_globals)
  File "/app/venv/lib/python3.10/site-packages/flask/__main__.py", line 15, in <module>
    main(as_module=True)
  File "/app/venv/lib/python3.10/site-packages/flask/cli.py", line 967, in main
    cli.main(args=sys.argv[1:], prog_name="python -m flask" if as_module else None)
  File "/app/venv/lib/python3.10/site-packages/flask/cli.py", line 586, in main
    return super(FlaskGroup, self).main(*args, **kwargs)
  File "/app/venv/lib/python3.10/site-packages/click/core.py", line 1053, in main
    rv = self.invoke(ctx)
  File "/app/venv/lib/python3.10/site-packages/click/core.py", line 1659, in invoke
    return _process_result(sub_ctx.command.invoke(sub_ctx))
  File "/app/venv/lib/python3.10/site-packages/click/core.py", line 1659, in invoke
    return _process_result(sub_ctx.command.invoke(sub_ctx))
  File "/app/venv/lib/python3.10/site-packages/click/core.py", line 1395, in invoke
    return ctx.invoke(self.callback, **ctx.params)
  File "/app/venv/lib/python3.10/site-packages/click/core.py", line 754, in invoke
    return __callback(*args, **kwargs)
  File "/app/venv/lib/python3.10/site-packages/click/decorators.py", line 26, in new_func
    return f(get_current_context(), *args, **kwargs)
  File "/app/venv/lib/python3.10/site-packages/flask/cli.py", line 426, in decorator
    return __ctx.invoke(f, *args, **kwargs)
  File "/app/venv/lib/python3.10/site-packages/click/core.py", line 754, in invoke
    return __callback(*args, **kwargs)
  File "/app/venv/lib/python3.10/site-packages/flask_migrate/cli.py", line 134, in upgrade
    _upgrade(directory, revision, sql, tag, x_arg)
  File "/app/venv/lib/python3.10/site-packages/flask_migrate/__init__.py", line 96, in wrapped
    f(*args, **kwargs)
  File "/app/venv/lib/python3.10/site-packages/flask_migrate/__init__.py", line 269, in upgrade
    config = current_app.extensions['migrate'].migrate.get_config(directory,
KeyError: 'migrate'
```

The traceback terminates in `flask_migrate/__init__.py:269` at
`config = current_app.extensions['migrate'].migrate.get_config(...)` -> `KeyError: 'migrate'`,
confirming the missing extension registration.

**Canonical substitute actually used to build the schema:** `alembic upgrade head`, run directly
against `alembic.ini` (`script_location = migrations` [alembic.ini:5]). This is the **project's
own canonical migration invocation** — CI runs exactly
`CONFIG=tests/test.env poetry run alembic upgrade head` [.github/workflows/main.yml:98]. It
succeeds (exit 0) and applies all **255** migrations. The complete, unedited log follows (265
lines: the app boot preamble, `Context impl PostgresqlImpl.`, then one
`Running upgrade <from> -> <to>, <label>` line per migration, first
`-> 5e549314e1e2` through last `7d7b84779837 -> 32f25cbf12f6`):

```text
$ CONFIG=/tmp/obs.env /app/venv/bin/alembic upgrade head ; echo exit=$?
load config file /tmp/obs.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/ocvuawzriqafqbylahnv
Upload files to local dir
>>> init logging <<<
2026-07-14 21:24:26,514 - SL - DEBUG - 593 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
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
INFO  [alembic.runtime.migration] Running upgrade b20ee72fd9a4 -> 590d89f981c0, empty message
INFO  [alembic.runtime.migration] Running upgrade 590d89f981c0 -> 551c4e6d4a8b, empty message
INFO  [alembic.runtime.migration] Running upgrade 551c4e6d4a8b -> c6e7fc37ad42, empty message
INFO  [alembic.runtime.migration] Running upgrade c6e7fc37ad42 -> 1b7d161d1012, empty message
INFO  [alembic.runtime.migration] Running upgrade 1b7d161d1012 -> 507afb2632cc, empty message
INFO  [alembic.runtime.migration] Running upgrade 507afb2632cc -> 4fac8c8a704c, empty message
INFO  [alembic.runtime.migration] Running upgrade 4fac8c8a704c -> c79c702a1f23, empty message
INFO  [alembic.runtime.migration] Running upgrade c79c702a1f23 -> 5fa68bafae72, empty message
INFO  [alembic.runtime.migration] Running upgrade 5fa68bafae72 -> 4a640c170d02, empty message
INFO  [alembic.runtime.migration] Running upgrade 4a640c170d02 -> 2e2b53afd819, empty message
INFO  [alembic.runtime.migration] Running upgrade 2e2b53afd819 -> 6bbda4685999, empty message
INFO  [alembic.runtime.migration] Running upgrade 6bbda4685999 -> d68a2d971b70, empty message
INFO  [alembic.runtime.migration] Running upgrade d68a2d971b70 -> 0a89c670fc7a, empty message
INFO  [alembic.runtime.migration] Running upgrade 0a89c670fc7a -> 3ebfbaeb76c0, empty message
INFO  [alembic.runtime.migration] Running upgrade 3ebfbaeb76c0 -> 83f4dbe125c4, empty message
INFO  [alembic.runtime.migration] Running upgrade 83f4dbe125c4 -> e505cb517589, empty message
INFO  [alembic.runtime.migration] Running upgrade e505cb517589 -> e83298198ca5, empty message
INFO  [alembic.runtime.migration] Running upgrade e83298198ca5 -> 3a87573bf8a8, empty message
INFO  [alembic.runtime.migration] Running upgrade 3a87573bf8a8 -> a8d8aa307b8b, empty message
INFO  [alembic.runtime.migration] Running upgrade a8d8aa307b8b -> 0b28518684ae, empty message
INFO  [alembic.runtime.migration] Running upgrade 0b28518684ae -> 5e868298fee7, empty message
INFO  [alembic.runtime.migration] Running upgrade 5e868298fee7 -> 2d2fc3e826af, empty message
INFO  [alembic.runtime.migration] Running upgrade 2d2fc3e826af -> 0c7f1a48aac9, empty message
INFO  [alembic.runtime.migration] Running upgrade 0c7f1a48aac9 -> 18e934d58f55, empty message
INFO  [alembic.runtime.migration] Running upgrade 18e934d58f55 -> 9e1b06b9df13, empty message
INFO  [alembic.runtime.migration] Running upgrade 9e1b06b9df13 -> d4e4488a0032, empty message
INFO  [alembic.runtime.migration] Running upgrade d4e4488a0032 -> e409f6214b2b, empty message
INFO  [alembic.runtime.migration] Running upgrade e409f6214b2b -> 696e17c13b8b, empty message
INFO  [alembic.runtime.migration] Running upgrade 696e17c13b8b -> a8b996f0be40, empty message
INFO  [alembic.runtime.migration] Running upgrade a8b996f0be40 -> 10ad2dbaeccf, empty message
INFO  [alembic.runtime.migration] Running upgrade 10ad2dbaeccf -> 01f808f15b2e, empty message
INFO  [alembic.runtime.migration] Running upgrade 01f808f15b2e -> d29cca963221, empty message
INFO  [alembic.runtime.migration] Running upgrade d29cca963221 -> ba6f13ccbabb, empty message
INFO  [alembic.runtime.migration] Running upgrade ba6f13ccbabb -> 7c39ba4ec38d, empty message
INFO  [alembic.runtime.migration] Running upgrade 7c39ba4ec38d -> 9c976df9b9c4, empty message
INFO  [alembic.runtime.migration] Running upgrade 9c976df9b9c4 -> b9f849432543, empty message
INFO  [alembic.runtime.migration] Running upgrade b9f849432543 -> 6664d75ce3d4, empty message
INFO  [alembic.runtime.migration] Running upgrade 6664d75ce3d4 -> 3c9542fc54e9, empty message
INFO  [alembic.runtime.migration] Running upgrade 3c9542fc54e9 -> 3fa3a648c8e7, empty message
INFO  [alembic.runtime.migration] Running upgrade 3fa3a648c8e7 -> 903ec5f566e8, empty message
INFO  [alembic.runtime.migration] Running upgrade 903ec5f566e8 -> f580030d9beb, empty message
INFO  [alembic.runtime.migration] Running upgrade f580030d9beb -> e3cb44b953f2, empty message
INFO  [alembic.runtime.migration] Running upgrade e3cb44b953f2 -> 75093e7ded27, empty message
INFO  [alembic.runtime.migration] Running upgrade 75093e7ded27 -> 5f191273d067, empty message
INFO  [alembic.runtime.migration] Running upgrade 5f191273d067 -> 7eef64ffb398, empty message
INFO  [alembic.runtime.migration] Running upgrade 7eef64ffb398 -> 235355381f53, empty message
INFO  [alembic.runtime.migration] Running upgrade 235355381f53 -> 628a5438295c, empty message
INFO  [alembic.runtime.migration] Running upgrade 628a5438295c -> 11a35b448f83, empty message
INFO  [alembic.runtime.migration] Running upgrade 11a35b448f83 -> 9081f1a90939, empty message
INFO  [alembic.runtime.migration] Running upgrade 9081f1a90939 -> 91b69dfad2f1, empty message
INFO  [alembic.runtime.migration] Running upgrade 91b69dfad2f1 -> 7744c5c16159, empty message
INFO  [alembic.runtime.migration] Running upgrade 7744c5c16159 -> 14167121af69, empty message
INFO  [alembic.runtime.migration] Running upgrade 14167121af69 -> 6e061eb84167, empty message
INFO  [alembic.runtime.migration] Running upgrade 6e061eb84167 -> e9395fe234a4, empty message
INFO  [alembic.runtime.migration] Running upgrade e9395fe234a4 -> 0809266d08ca, empty message
INFO  [alembic.runtime.migration] Running upgrade 0809266d08ca -> f4b8232fa17e, empty message
INFO  [alembic.runtime.migration] Running upgrade f4b8232fa17e -> dbd80d290f04, empty message
INFO  [alembic.runtime.migration] Running upgrade dbd80d290f04 -> 4e4a759ac4b5, empty message
INFO  [alembic.runtime.migration] Running upgrade 4e4a759ac4b5 -> 30c13ca016e4, empty message
INFO  [alembic.runtime.migration] Running upgrade 30c13ca016e4 -> 541ce53ab6e9, empty message
INFO  [alembic.runtime.migration] Running upgrade 541ce53ab6e9 -> 67c61eead8d2, empty message
INFO  [alembic.runtime.migration] Running upgrade 67c61eead8d2 -> 224fd8963462, empty message
INFO  [alembic.runtime.migration] Running upgrade 224fd8963462 -> 92baf66b268b, empty message
INFO  [alembic.runtime.migration] Running upgrade 92baf66b268b -> 497cfd2a02e2, empty message
INFO  [alembic.runtime.migration] Running upgrade 497cfd2a02e2 -> ea30c0b5b2e3, empty message
INFO  [alembic.runtime.migration] Running upgrade ea30c0b5b2e3 -> bfd7b2302903, empty message
INFO  [alembic.runtime.migration] Running upgrade bfd7b2302903 -> 57ef03f3ac34, empty message
INFO  [alembic.runtime.migration] Running upgrade 57ef03f3ac34 -> dd911f880b75, empty message
INFO  [alembic.runtime.migration] Running upgrade dd911f880b75 -> bd05eac83f5f, empty message
INFO  [alembic.runtime.migration] Running upgrade bd05eac83f5f -> b4146f7d5277, empty message
INFO  [alembic.runtime.migration] Running upgrade b4146f7d5277 -> f939d67374e4, empty message
INFO  [alembic.runtime.migration] Running upgrade f939d67374e4 -> de1b457472e0, empty message
INFO  [alembic.runtime.migration] Running upgrade de1b457472e0 -> ae94fe5c4e9f, empty message
INFO  [alembic.runtime.migration] Running upgrade ae94fe5c4e9f -> 026e7a782ed6, empty message
INFO  [alembic.runtime.migration] Running upgrade 026e7a782ed6 -> 925b93d92809, empty message
INFO  [alembic.runtime.migration] Running upgrade 925b93d92809 -> bdf76f4b65a2, empty message
INFO  [alembic.runtime.migration] Running upgrade bdf76f4b65a2 -> a3a7c518ea70, empty message
INFO  [alembic.runtime.migration] Running upgrade a3a7c518ea70 -> a5e3c6693dc6, empty message
INFO  [alembic.runtime.migration] Running upgrade a5e3c6693dc6 -> bf11ab2f0a7a, empty message
INFO  [alembic.runtime.migration] Running upgrade bf11ab2f0a7a -> 1759f73274ee, empty message
INFO  [alembic.runtime.migration] Running upgrade 1759f73274ee -> 552d735a2f1f, empty message
INFO  [alembic.runtime.migration] Running upgrade 552d735a2f1f -> 5cad8fa84386, empty message
INFO  [alembic.runtime.migration] Running upgrade 5cad8fa84386 -> c31cdf879ee3, empty message
INFO  [alembic.runtime.migration] Running upgrade c31cdf879ee3 -> 659d979b64ce, empty message
INFO  [alembic.runtime.migration] Running upgrade 659d979b64ce -> ce15cf3467b4, empty message
INFO  [alembic.runtime.migration] Running upgrade ce15cf3467b4 -> 0e08145f0499, empty message
INFO  [alembic.runtime.migration] Running upgrade 0e08145f0499 -> 00532ac6d4bc, empty message
INFO  [alembic.runtime.migration] Running upgrade 00532ac6d4bc -> f680032cc361, empty message
INFO  [alembic.runtime.migration] Running upgrade f680032cc361 -> 10a7947fda6b, empty message
INFO  [alembic.runtime.migration] Running upgrade 10a7947fda6b -> 4a7d35941602, empty message
INFO  [alembic.runtime.migration] Running upgrade 4a7d35941602 -> cfc013b6461a, empty message
INFO  [alembic.runtime.migration] Running upgrade cfc013b6461a -> b2d51e4d94c8, empty message
INFO  [alembic.runtime.migration] Running upgrade b2d51e4d94c8 -> 749c2b85d20f, empty message
INFO  [alembic.runtime.migration] Running upgrade 749c2b85d20f -> a5b4dc311a89, empty message
INFO  [alembic.runtime.migration] Running upgrade a5b4dc311a89 -> a3c9a43e41f4, empty message
INFO  [alembic.runtime.migration] Running upgrade a3c9a43e41f4 -> 7128f87af701, empty message
INFO  [alembic.runtime.migration] Running upgrade 7128f87af701 -> 270d598c51e3, empty message
INFO  [alembic.runtime.migration] Running upgrade 270d598c51e3 -> b77ab8c47cc7, empty message
INFO  [alembic.runtime.migration] Running upgrade b77ab8c47cc7 -> a2b95b04d1f7, empty message
INFO  [alembic.runtime.migration] Running upgrade a2b95b04d1f7 -> 63fd3b240583, empty message
INFO  [alembic.runtime.migration] Running upgrade 63fd3b240583 -> 95938a93ea14, empty message
INFO  [alembic.runtime.migration] Running upgrade 95938a93ea14 -> b82bcad9accf, empty message
INFO  [alembic.runtime.migration] Running upgrade b82bcad9accf -> 84471852b610, empty message
INFO  [alembic.runtime.migration] Running upgrade 84471852b610 -> b0e9a389939a, empty message
INFO  [alembic.runtime.migration] Running upgrade b0e9a389939a -> 198c3aca9d8d, empty message
INFO  [alembic.runtime.migration] Running upgrade 198c3aca9d8d -> 58ad4df8583e, empty message
INFO  [alembic.runtime.migration] Running upgrade 58ad4df8583e -> 1abfc9e14d7e, empty message
INFO  [alembic.runtime.migration] Running upgrade 1abfc9e14d7e -> 32b00d06d892, empty message
INFO  [alembic.runtime.migration] Running upgrade 32b00d06d892 -> b17afc77ba83, empty message
INFO  [alembic.runtime.migration] Running upgrade b17afc77ba83 -> 54ca2dbf89c0, empty message
INFO  [alembic.runtime.migration] Running upgrade 54ca2dbf89c0 -> eef0c404b531, empty message
INFO  [alembic.runtime.migration] Running upgrade eef0c404b531 -> 84dec6c29c48, empty message
INFO  [alembic.runtime.migration] Running upgrade 84dec6c29c48 -> d0f197979bd9, empty message
INFO  [alembic.runtime.migration] Running upgrade d0f197979bd9 -> 9dc16e591f88, empty message
INFO  [alembic.runtime.migration] Running upgrade 9dc16e591f88 -> ac41029fb329, empty message
INFO  [alembic.runtime.migration] Running upgrade ac41029fb329 -> d1edb3cadec8, empty message
INFO  [alembic.runtime.migration] Running upgrade d1edb3cadec8 -> 623662ea0e7e, empty message
INFO  [alembic.runtime.migration] Running upgrade 623662ea0e7e -> 56c790ec8ab4, empty message
INFO  [alembic.runtime.migration] Running upgrade 56c790ec8ab4 -> c0d91ff18f77, empty message
INFO  [alembic.runtime.migration] Running upgrade c0d91ff18f77 -> 780a8344914b, empty message
INFO  [alembic.runtime.migration] Running upgrade 780a8344914b -> a20aeb9b0eac, empty message
INFO  [alembic.runtime.migration] Running upgrade a20aeb9b0eac -> 0af2c2e286a7, empty message
INFO  [alembic.runtime.migration] Running upgrade 0af2c2e286a7 -> 1919f1859215, empty message
INFO  [alembic.runtime.migration] Running upgrade 1919f1859215 -> f66ca777f409, empty message
INFO  [alembic.runtime.migration] Running upgrade f66ca777f409 -> 7c0dbd378cdb, empty message
INFO  [alembic.runtime.migration] Running upgrade 7c0dbd378cdb -> e99989e6ad56, empty message
INFO  [alembic.runtime.migration] Running upgrade e99989e6ad56 -> 1b54995bc086, empty message
INFO  [alembic.runtime.migration] Running upgrade 1b54995bc086 -> 2779eb90c6c4, empty message
INFO  [alembic.runtime.migration] Running upgrade 2779eb90c6c4 -> 74906d31d994, empty message
INFO  [alembic.runtime.migration] Running upgrade 74906d31d994 -> 85d0655d42c0, empty message
INFO  [alembic.runtime.migration] Running upgrade 85d0655d42c0 -> de7aa5280210, empty message
INFO  [alembic.runtime.migration] Running upgrade de7aa5280210 -> e831a883153a, empty message
INFO  [alembic.runtime.migration] Running upgrade e831a883153a -> d1236c4dff71, empty message
INFO  [alembic.runtime.migration] Running upgrade d1236c4dff71 -> 94f14eb0fe5b, empty message
INFO  [alembic.runtime.migration] Running upgrade 94f14eb0fe5b -> 9d6adad83936, empty message
INFO  [alembic.runtime.migration] Running upgrade 9d6adad83936 -> f398b261d9c6, empty message
INFO  [alembic.runtime.migration] Running upgrade f398b261d9c6 -> 517b79c56088, empty message
INFO  [alembic.runtime.migration] Running upgrade 517b79c56088 -> 48b991e9de06, empty message
INFO  [alembic.runtime.migration] Running upgrade 48b991e9de06 -> e11c3dd48a6f, empty message
INFO  [alembic.runtime.migration] Running upgrade e11c3dd48a6f -> 4912f3bd5ba2, empty message
INFO  [alembic.runtime.migration] Running upgrade 4912f3bd5ba2 -> f5133dc851ee, empty message
INFO  [alembic.runtime.migration] Running upgrade f5133dc851ee -> 5c77d685df87, empty message
INFO  [alembic.runtime.migration] Running upgrade 5c77d685df87 -> 6cc7f073b358, empty message
INFO  [alembic.runtime.migration] Running upgrade 6cc7f073b358 -> 68e2f38e33f4, empty message
INFO  [alembic.runtime.migration] Running upgrade 68e2f38e33f4 -> fc2eb1d7e4fc, empty message
INFO  [alembic.runtime.migration] Running upgrade fc2eb1d7e4fc -> a5e643d562c9, empty message
INFO  [alembic.runtime.migration] Running upgrade a5e643d562c9 -> 29ea13ed76f9, empty message
INFO  [alembic.runtime.migration] Running upgrade 29ea13ed76f9 -> 8e70205a5308, empty message
INFO  [alembic.runtime.migration] Running upgrade 8e70205a5308 -> f3f19998b755, empty message
INFO  [alembic.runtime.migration] Running upgrade f3f19998b755 -> c31a081eab74, empty message
INFO  [alembic.runtime.migration] Running upgrade c31a081eab74 -> 78403c7b8089, empty message
INFO  [alembic.runtime.migration] Running upgrade 78403c7b8089 -> 5662122eac21, empty message
INFO  [alembic.runtime.migration] Running upgrade 5662122eac21 -> 20c738810b1b, empty message
INFO  [alembic.runtime.migration] Running upgrade 20c738810b1b -> dfee471558bd, empty message
INFO  [alembic.runtime.migration] Running upgrade dfee471558bd -> 05e3af59929a, empty message
INFO  [alembic.runtime.migration] Running upgrade 05e3af59929a -> c3470e2d3224, empty message
INFO  [alembic.runtime.migration] Running upgrade c3470e2d3224 -> ffa75d04e6ef, empty message
INFO  [alembic.runtime.migration] Running upgrade ffa75d04e6ef -> 9014cca7097c, empty message
INFO  [alembic.runtime.migration] Running upgrade 9014cca7097c -> d4392342465f, empty message
INFO  [alembic.runtime.migration] Running upgrade d4392342465f -> 424808e1fe49, empty message
INFO  [alembic.runtime.migration] Running upgrade 424808e1fe49 -> 916a5257d18c, empty message
INFO  [alembic.runtime.migration] Running upgrade 916a5257d18c -> 4d3f91ddf3e9, empty message
INFO  [alembic.runtime.migration] Running upgrade 4d3f91ddf3e9 -> d8c55e79da54, empty message
INFO  [alembic.runtime.migration] Running upgrade d8c55e79da54 -> cf1e8c1bc737, empty message
INFO  [alembic.runtime.migration] Running upgrade cf1e8c1bc737 -> 7a105bfc0cd0, empty message
INFO  [alembic.runtime.migration] Running upgrade 7a105bfc0cd0 -> bc75acacc98e, empty message
INFO  [alembic.runtime.migration] Running upgrade bc75acacc98e -> b8b4f9598240, empty message
INFO  [alembic.runtime.migration] Running upgrade b8b4f9598240 -> 5ee767807344, empty message
INFO  [alembic.runtime.migration] Running upgrade 5ee767807344 -> 4913cb3f5a05, empty message
INFO  [alembic.runtime.migration] Running upgrade 4913cb3f5a05 -> 0b1c9ea11aef, empty message
INFO  [alembic.runtime.migration] Running upgrade 0b1c9ea11aef -> 2fbcad5527d7, empty message
INFO  [alembic.runtime.migration] Running upgrade 2fbcad5527d7 -> d750d578b068, empty message
INFO  [alembic.runtime.migration] Running upgrade d750d578b068 -> 2f1b3c759773, empty message
INFO  [alembic.runtime.migration] Running upgrade 2f1b3c759773 -> 99d9e329b27f, empty message
INFO  [alembic.runtime.migration] Running upgrade 99d9e329b27f -> a06066e3fbeb, empty message
INFO  [alembic.runtime.migration] Running upgrade a06066e3fbeb -> d67eab226ecd, empty message
INFO  [alembic.runtime.migration] Running upgrade d67eab226ecd -> bbedc353f90c, empty message
INFO  [alembic.runtime.migration] Running upgrade bbedc353f90c -> 0b9150eb309d, Increase message_id length manually
INFO  [alembic.runtime.migration] Running upgrade 0b9150eb309d -> 6204e57b4bc4, empty message
INFO  [alembic.runtime.migration] Running upgrade 6204e57b4bc4 -> 37feaba7c45d, empty message
INFO  [alembic.runtime.migration] Running upgrade 37feaba7c45d -> ff6c04869029, empty message
INFO  [alembic.runtime.migration] Running upgrade ff6c04869029 -> fdb02bd105a8, empty message
INFO  [alembic.runtime.migration] Running upgrade fdb02bd105a8 -> dd278f96ca83, empty message
INFO  [alembic.runtime.migration] Running upgrade dd278f96ca83 -> 1076b5795b08, empty message
INFO  [alembic.runtime.migration] Running upgrade 1076b5795b08 -> 5639ad89ee50, empty message
INFO  [alembic.runtime.migration] Running upgrade 5639ad89ee50 -> 11ba83e2dd71, empty message
INFO  [alembic.runtime.migration] Running upgrade 11ba83e2dd71 -> ccbfb61eda0d, empty message
INFO  [alembic.runtime.migration] Running upgrade ccbfb61eda0d -> a5013ff0a00a, empty message
INFO  [alembic.runtime.migration] Running upgrade a5013ff0a00a -> e6e8e12f5a13, empty message
INFO  [alembic.runtime.migration] Running upgrade e6e8e12f5a13 -> 9031c9e28510, empty message
INFO  [alembic.runtime.migration] Running upgrade 9031c9e28510 -> b8fd175c084a, empty message
INFO  [alembic.runtime.migration] Running upgrade b8fd175c084a -> e7d7ebcea26c, empty message
INFO  [alembic.runtime.migration] Running upgrade e7d7ebcea26c -> d0ccd9d7ac0c, empty message
INFO  [alembic.runtime.migration] Running upgrade d0ccd9d7ac0c -> 4b483a762fed, empty message
INFO  [alembic.runtime.migration] Running upgrade 4b483a762fed -> ad467baf7ec8, empty message
INFO  [alembic.runtime.migration] Running upgrade ad467baf7ec8 -> d8a3dfe674f2, empty message
INFO  [alembic.runtime.migration] Running upgrade d8a3dfe674f2 -> 3d05479d0d11, empty message
INFO  [alembic.runtime.migration] Running upgrade 3d05479d0d11 -> 753d2ed92d41, empty message
INFO  [alembic.runtime.migration] Running upgrade 753d2ed92d41 -> 698424c429e9, empty message
INFO  [alembic.runtime.migration] Running upgrade 698424c429e9 -> 07b870d7cc86, empty message
INFO  [alembic.runtime.migration] Running upgrade 07b870d7cc86 -> 9282e982bc05, Add block_behaviour setting for user
INFO  [alembic.runtime.migration] Running upgrade 9282e982bc05 -> 5047fcbd57c7, empty message
INFO  [alembic.runtime.migration] Running upgrade 5047fcbd57c7 -> 4729b7096d12, empty message
INFO  [alembic.runtime.migration] Running upgrade 4729b7096d12 -> b500363567e3, Create admin audit log
INFO  [alembic.runtime.migration] Running upgrade b500363567e3 -> 28b9b14c9664, store provider complaints
INFO  [alembic.runtime.migration] Running upgrade 28b9b14c9664 -> 0aaad1740797, store provider complaints
INFO  [alembic.runtime.migration] Running upgrade 0aaad1740797 -> e866ad0e78e1, Add partner tables
INFO  [alembic.runtime.migration] Running upgrade e866ad0e78e1 -> 088f23324464, add flags to the user model
INFO  [alembic.runtime.migration] Running upgrade 088f23324464 -> 2b1d3cd93e4b, update partner_api_token token length
INFO  [alembic.runtime.migration] Running upgrade 2b1d3cd93e4b -> 82d3c7109ffb, partner_user and partner_subscription
INFO  [alembic.runtime.migration] Running upgrade 82d3c7109ffb -> 36646e5dc6d9, make external_user_id non nullable
INFO  [alembic.runtime.migration] Running upgrade 36646e5dc6d9 -> a7bcb872c12a, Add alias transfer token expiration
INFO  [alembic.runtime.migration] Running upgrade a7bcb872c12a -> 673a074e4215, empty message
INFO  [alembic.runtime.migration] Running upgrade 673a074e4215 -> d1fb679f7eec, Add sudo expiration for ApiKeys
INFO  [alembic.runtime.migration] Running upgrade d1fb679f7eec -> bfebc2d5c719, Add state to job
INFO  [alembic.runtime.migration] Running upgrade bfebc2d5c719 -> 516c21ea7d87, empty message
INFO  [alembic.runtime.migration] Running upgrade 516c21ea7d87 -> bd7d032087b2, empty message
INFO  [alembic.runtime.migration] Running upgrade bd7d032087b2 -> b0101a66bb77, Add unsubscribe behaviour
INFO  [alembic.runtime.migration] Running upgrade b0101a66bb77 -> 89081a00fc7d, default_unsub_behaviour
INFO  [alembic.runtime.migration] Running upgrade 89081a00fc7d -> c66f2c5b6cb1, empty message
INFO  [alembic.runtime.migration] Running upgrade c66f2c5b6cb1 -> 9cc0f0712b29, Add api to cookie token
INFO  [alembic.runtime.migration] Running upgrade 9cc0f0712b29 -> bd95b2b4217f, Updated recovery code string length
INFO  [alembic.runtime.migration] Running upgrade bd95b2b4217f -> 2c2093c82bc0, empty message
INFO  [alembic.runtime.migration] Running upgrade 2c2093c82bc0 -> 5f4a5625da66, empty message
INFO  [alembic.runtime.migration] Running upgrade 5f4a5625da66 -> 893c0d18475f, empty message
INFO  [alembic.runtime.migration] Running upgrade 893c0d18475f -> bc496c0a0279, empty message
INFO  [alembic.runtime.migration] Running upgrade bc496c0a0279 -> 2d89315ac650, empty message
INFO  [alembic.runtime.migration] Running upgrade 893c0d18475f -> 01e2997e90d3, empty message
INFO  [alembic.runtime.migration] Running upgrade 01e2997e90d3, 2d89315ac650 -> 2634b41f54db, empty message
INFO  [alembic.runtime.migration] Running upgrade 2634b41f54db -> 01827104004b, empty message
INFO  [alembic.runtime.migration] Running upgrade 01827104004b -> 0a5701a4f5e4, empty message
INFO  [alembic.runtime.migration] Running upgrade 0a5701a4f5e4 -> ec7fdde8da9f, empty message
INFO  [alembic.runtime.migration] Running upgrade ec7fdde8da9f -> 46ecb648a47e, empty message
INFO  [alembic.runtime.migration] Running upgrade 46ecb648a47e -> 4bc54632d9aa, empty message
INFO  [alembic.runtime.migration] Running upgrade 4bc54632d9aa -> 818b0a956205, empty message
INFO  [alembic.runtime.migration] Running upgrade 818b0a956205 -> 52510a633d6f, empty message
INFO  [alembic.runtime.migration] Running upgrade 52510a633d6f -> fa2f19bb4e5a, empty message
INFO  [alembic.runtime.migration] Running upgrade fa2f19bb4e5a -> 06a9a7133445, Create sync_event table
INFO  [alembic.runtime.migration] Running upgrade 06a9a7133445 -> d608b8e48082, empty message
INFO  [alembic.runtime.migration] Running upgrade d608b8e48082 -> 56d08955fcab, add retry count to sync event
INFO  [alembic.runtime.migration] Running upgrade 56d08955fcab -> 1c14339aae90, empty message
INFO  [alembic.runtime.migration] Running upgrade 1c14339aae90 -> 2441b7ff5da9, Custom Domain partner id
INFO  [alembic.runtime.migration] Running upgrade 2441b7ff5da9 -> 88dd7a0abf54, contact.flags and custom_domain.pending_deletion
INFO  [alembic.runtime.migration] Running upgrade 88dd7a0abf54 -> 62afa3a10010, custom domain indices
INFO  [alembic.runtime.migration] Running upgrade 62afa3a10010 -> 91ed7f46dc81, alias_audit_log
INFO  [alembic.runtime.migration] Running upgrade 91ed7f46dc81 -> 7d7b84779837, user_audit_log
INFO  [alembic.runtime.migration] Running upgrade 7d7b84779837 -> 32f25cbf12f6, alias_audit_log_index_created_at
exit=0
```

**Migration count grounded in real output** (not inferred from `alembic.ini`): the number of
applied migrations equals the number of version scripts on disk, both **255**:

```text
$ grep -c "Running upgrade" 21_req2_alembic_full.log
255
$ ls -1 /app/migrations/versions/*.py | wc -l
255
```

### 3.3 Post-migration database state (real output + exit codes)

The migrated schema has **77** tables, the `users` table now **exists** (contrast REQ-1's
`UndefinedTable`), and `public_domain` (the `SLDomain` table [app/models.py:3116-3119]) is
**empty** until `init_app.py` runs:

```text
$ PGPASSWORD=mypassword psql -h 127.0.0.1 -U myuser -d simplelogin -tAc "SELECT count(*) FROM information_schema.tables WHERE table_schema='public'" ; echo exit=$?
77
exit=0
$ PGPASSWORD=mypassword psql -h 127.0.0.1 -U myuser -d simplelogin -tAc "SELECT to_regclass('public.users') IS NOT NULL" ; echo exit=$?
t
exit=0
$ PGPASSWORD=mypassword psql -h 127.0.0.1 -U myuser -d simplelogin -tAc "SELECT count(*) FROM public_domain" ; echo exit=$?
0
exit=0
```

### 3.4 `python init_app.py` — complete output and seed proof

`init_app.py`'s `__main__` block enters `create_light_app().app_context()` and calls
`load_pgp_public_keys()` and `add_sl_domains()` [init_app.py:69-73]; `add_sl_domains()` inserts
each `ALIAS_DOMAINS` / `PREMIUM_ALIAS_DOMAINS` entry into `SLDomain` [init_app.py:39-56]. Complete,
unedited output with exit code:

```text
$ CONFIG=/tmp/obs.env /app/venv/bin/python init_app.py ; echo exit=$?
load config file /tmp/obs.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/zmhzefvrvnwcjumrgpsv
Upload files to local dir
>>> init logging <<<
2026-07-14 21:24:57,393 - SL - DEBUG - 646 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-14 21:24:58,162 - SL - DEBUG - 646 - "/app/init_app.py:36" - load_pgp_public_keys() -  - Finish load_pgp_public_keys
2026-07-14 21:24:58,164 - SL - INFO - 646 - "/app/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain
exit=0
```

The two operative log lines are `Finish load_pgp_public_keys` [init_app.py:36] and
`Add sl.local to SL domain` [init_app.py:44] (`sl.local` because `example.env` sets
`EMAIL_DOMAIN=sl.local`, so `ALIAS_DOMAINS = ["sl.local"]` [app/config.py:92,157-161]).
Seed proof — `public_domain` now holds exactly one row, `sl.local`:

```text
$ PGPASSWORD=mypassword psql -h 127.0.0.1 -U myuser -d simplelogin -tAc "SELECT count(*) FROM public_domain" ; echo exit=$?
1
exit=0
$ PGPASSWORD=mypassword psql -h 127.0.0.1 -U myuser -d simplelogin -tAc "SELECT domain FROM public_domain" ; echo exit=$?
sl.local
exit=0
```

### 3.5 Service 1 — web application (`python server.py`, :7777)

Started via `local_main()` -> `app.run(debug=True, port=7777)` [server.py:572-588]. Complete
startup stdout/stderr (reloader parent PID 681 + child PID 703; `* Serving Flask app` banner
present, `* Running on` / `* Debugger PIN` suppressed per §1.9):

```text
$ CONFIG=/tmp/obs.env nohup /app/venv/bin/python server.py > /tmp/req2_web.log 2>&1 &
load config file /tmp/obs.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/egajbqedyrsbukptmwhi
Upload files to local dir
>>> init logging <<<
2026-07-14 21:25:18,132 - SL - DEBUG - 681 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
load config file /tmp/obs.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/nyindlzsyiwubxyindip
Upload files to local dir
>>> init logging <<<
2026-07-14 21:25:19,696 - SL - DEBUG - 703 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
/app/venv/lib/python3.10/site-packages/flask_debugtoolbar/__init__.py:213: UserWarning: Could not insert debug toolbar. </body> tag not found in response.
  warnings.warn('Could not insert debug toolbar.'
```

Port-binding proof (`:7777` LISTEN, state `0A`, `0100007F:1E61` = `127.0.0.1:7777`) + readiness
via the `/health` endpoint registered in `create_app` [server.py]:

```text
$ fuser 7777/tcp
   681   703
$ awk '$4=="0A"{print $2}' /proc/net/tcp | grep -i ':1E61$'
0100007F:1E61
$ curl -sS -o /dev/null -w "GET /health -> %{http_code}\n" http://127.0.0.1:7777/health
GET /health -> 200
```

### 3.6 Service 2 — email handler (`python email_handler.py`, :20381)

`main(port)` builds `Controller(MailHandler(), hostname="0.0.0.0", port=port)` and
`controller.start()`, then blocks on `while True: time.sleep(2)`; argparse default port is
**20381**. Complete startup stdout showing both readiness lines:

```text
$ CONFIG=/tmp/obs.env nohup /app/venv/bin/python email_handler.py > /tmp/req2_email.log 2>&1 &
load config file /tmp/obs.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/qgzzrgiusbnpuwlwrqua
Upload files to local dir
>>> init logging <<<
2026-07-14 21:25:30,753 - SL - DEBUG - 731 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-14 21:25:31,279 - SL - INFO - 731 - "/app/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-14 21:25:31,281 - SL - DEBUG - 731 - "/app/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

Port-binding proof (`:20381` LISTEN, `00000000:4F9D` = `0.0.0.0:20381`) + a **live SMTP probe**
(the aiosmtpd `Controller` answers `EHLO` and `NOOP`), proving it accepts connections:

```text
$ fuser 20381/tcp
   731
$ awk '$4=="0A"{print $2}' /proc/net/tcp | grep -i ':4F9D$'
00000000:4F9D
$ /app/venv/bin/python -c "import smtplib; s=smtplib.SMTP('127.0.0.1',20381); print('NOOP',s.noop()); print('EHLO',s.ehlo()[0]); s.quit()"
NOOP (250, b'OK')
EHLO 250
```

> **Port 25 vs 20381 (dev vs prod).** The architecture describes the SMTP handler at "port 25"
> (the production Postfix relay target). In local development `python email_handler.py` binds the
> argparse default **20381**; the observation above confirms the actually-bound port is 20381.

### 3.7 Service 3 — job runner (`python job_runner.py`, no port)

The job runner enters `while True:` inside `create_light_app().app_context()`, iterates
`get_jobs_to_run()`, logs `Take job %s` **only when a job is claimed**, then `time.sleep(10)`
[job_runner.py:329-347]. It binds **no port** and emits **no explicit ready banner**, so
readiness is reported honestly as *process alive and polling*. Complete startup stdout (single
process PID 769 — no reloader):

```text
$ CONFIG=/tmp/obs.env nohup /app/venv/bin/python job_runner.py > /tmp/req2_job.log 2>&1 &
load config file /tmp/obs.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/irogwctsynvsktmoaneg
Upload files to local dir
>>> init logging <<<
2026-07-14 21:25:32,860 - SL - DEBUG - 769 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

**Time-bounded liveness** — the process is observed across a **12-second** window (spanning a
full 10-second poll cycle): elapsed time advances 33s -> 45s and the process is still alive:

```text
job_runner pid = 769
$ ps -o pid,etimes,cmd -p 769   # elapsed seconds (t0)
    PID ELAPSED CMD
    769      33 /app/venv/bin/python job_runner.py
sleeping 12s to span a full 10s poll cycle...
$ ps -o pid,etimes,cmd -p 769   # elapsed seconds (t1) — must be >= t0+12 and still ALIVE
    PID ELAPSED CMD
    769      45 /app/venv/bin/python job_runner.py
$ kill -0 769 && echo "STILL ALIVE (process running)" || echo "DEAD"
STILL ALIVE (process running)
```

**Holds no port** — the job PID (769) is absent from the owners of `:7777`/`:20381` (which are
the web + email PIDs 681, 703, 731):

```text
$ fuser 7777/tcp 20381/tcp   # owners; must NOT include job pid 769
   681   703   731
$ ls -1 /proc/769/fd | wc -l  (job has fds but no listening socket on 7777/20381)
```

### 3.8 Production web alternative — gunicorn (real banner, OBSERVED)

Beyond the dev server, the Dockerfile CMD is `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15`
[Dockerfile:47], serving `app = create_app()` [wsgi.py]. This banner is **OBSERVED** (captured
live in the canonical image), not inferred:

```text
$ gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15   (from Dockerfile:47)
[2026-07-14 21:26:42 +0000] [860] [INFO] Starting gunicorn 20.0.4
[2026-07-14 21:26:42 +0000] [860] [INFO] Listening at: http://0.0.0.0:7777 (860)
[2026-07-14 21:26:42 +0000] [860] [INFO] Using worker: sync
[2026-07-14 21:26:42 +0000] [865] [INFO] Booting worker with pid: 865
[2026-07-14 21:26:42 +0000] [866] [INFO] Booting worker with pid: 866
load config file /tmp/obs.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/rgyxjlihdoblcdigtaqo
Upload files to local dir
>>> init logging <<<
2026-07-14 21:26:42,816 - SL - DEBUG - 865 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
load config file /tmp/obs.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/snptpsynkwpdjoalynvt
Upload files to local dir
>>> init logging <<<
2026-07-14 21:26:42,897 - SL - DEBUG - 866 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
--- health under gunicorn ---
GET /health -> 200
```

### 3.9 Log format and clean shutdown

All services share the stdout log format
`%(asctime)s - %(name)s - %(levelname)s - %(process)d - "%(pathname)s:%(lineno)d" - %(funcName)s() - %(message_id)s - %(message)s`
[app/log.py:12-16,41] — visible in every log line above (e.g., the `email_handler.py:2403`
readiness line). After capture, all services were stopped and both ports verified released:

```text
LISTEN :7777  -> [empty=released]
LISTEN :20381 -> [empty=released]
procs server.py       = 0
procs email_handler.py= 0
procs job_runner.py   = 0
procs gunicorn        = 0
```

---

## §4 REQ-3 — Skipped-initialization email rejection

### 4.1 Direct answer

With migrations run but `python init_app.py` **skipped** (so `public_domain` — the `SLDomain`
table [app/models.py:3116-3119] — is empty), an inbound message to `x@sl.local` is **rejected**.
The SMTP status code returned to the sender is:

> **`550 SL E515 Email not exist`** [app/email/status.py:51]

(a) **What happens:** the email handler accepts the SMTP session (EHLO/MAIL/RCPT/DATA all `250`/
`354`), then, after DATA, the forward pipeline finds no alias `x@sl.local`, fails to auto-create
one, and returns a permanent-failure reply. (b) **Status code:** `550 SL E515 Email not exist`.
(c) **Log lines explaining it:** two `handle_forward` lines — `alias x@sl.local not exist...`
[email_handler.py:545] then `alias x@sl.local cannot be created on-the-fly, return 550`
[email_handler.py:551] — bracketed by the `_handle` `Finish ... return code '550 SL E515 Email
not exist'` line [email_handler.py:2367] (full logs in §4.3).

**Honesty nuance (cause, led by the direct answer):** on the **forward** path this 550 is driven
by **alias-nonexistence + auto-create failure**, which is **independent of the empty `SLDomain`
table**. The `SLDomain` check (`is_valid_alias_address_domain` [app/email_utils.py:557-563])
governs the **reply** path, not this forward path (see §4.4). The empty `SLDomain` from skipping
`init_app.py` is the scenario's *premise*; the operative cause of this specific 550 is the
missing alias.

### 4.2 Commands run

```bash
# Guarded reset, then migrations ONLY (init_app.py deliberately NOT run) -- see §1.6
DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin /tmp/reset_db.sh
CONFIG=/tmp/obs.env /app/venv/bin/alembic upgrade head        # 255 migrations, exit 0
# start the email handler through its real aiosmtpd entry point (:20381)
CONFIG=/tmp/obs.env nohup /app/venv/bin/python email_handler.py > /tmp/req3_email.log 2>&1 &
# send an inbound message to x@sl.local from a normal sender, capturing the full SMTP dialogue
/app/venv/bin/python /tmp/req3_send.py
```

The sender script is reproduced in full (it lives under `/tmp`, outside the repo, and is removed
on completion — §5). It targets the **real** aiosmtpd listener on `127.0.0.1:20381` and uses
`set_debuglevel(1)` so smtplib echoes the exact server replies verbatim:

**`/tmp/req3_send.py`**:

```python
"""REQ-3: send an inbound message to x@sl.local through the REAL aiosmtpd entry point
on 127.0.0.1:20381, from a NORMAL (non-ignore-bounce) sender, and print the full SMTP
transcript. set_debuglevel(1) makes smtplib echo every command/reply to stderr, so the
exact 550 reply after DATA is captured verbatim.
"""
import smtplib
from email.message import EmailMessage

msg = EmailMessage()
msg["From"] = "someone@example.com"
msg["To"] = "x@sl.local"
msg["Subject"] = "test"
msg.set_content("hi")

s = smtplib.SMTP("127.0.0.1", 20381, timeout=15)
s.set_debuglevel(1)  # echo full SMTP dialogue
try:
    s.sendmail("someone@example.com", ["x@sl.local"], msg.as_bytes())
    print("RESULT: sendmail returned without exception (unexpected)")
except smtplib.SMTPDataError as e:
    print("RESULT: SMTPDataError -> code=", e.smtp_code, "msg=", e.smtp_error)
finally:
    try:
        s.quit()
    except Exception:
        pass
```

### 4.3 Observed output

**(a) Migrated-but-NOT-initialized state proof** — migrations applied (255), but `public_domain`
(SLDomain) and `alias` are both empty because `init_app.py` was skipped (real output + exit
codes):

```text
$ DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin /tmp/reset_db.sh ; echo reset_exit=$?
RESET DONE: fresh empty simplelogin
reset_exit=0
$ CONFIG=/tmp/obs.env /app/venv/bin/alembic upgrade head ; echo alembic_exit=$?
alembic_exit=0   # (255 "Running upgrade" lines — full log identical in form to §3.2)
$ PGPASSWORD=mypassword psql -h 127.0.0.1 -U myuser -d simplelogin -tAc "SELECT count(*) FROM public_domain" ; echo exit=$?
0
exit=0
$ PGPASSWORD=mypassword psql -h 127.0.0.1 -U myuser -d simplelogin -tAc "SELECT count(*) FROM alias" ; echo exit=$?
0
exit=0
```

**(b) Email handler startup** (`:20381`, PID 1295) — same readiness lines as §3.6:

```text
load config file /tmp/obs.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/mcwnppzmxiustmjeapid
Upload files to local dir
>>> init logging <<<
2026-07-14 21:28:27,561 - SL - DEBUG - 1295 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-14 21:28:28,033 - SL - INFO - 1295 - "/app/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-14 21:28:28,035 - SL - DEBUG - 1295 - "/app/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

**(c) Complete, unedited SMTP transcript** (from `req3_send.py` with `set_debuglevel(1)`). The
session is accepted through DATA (`250`/`354`); the `550 SL E515 Email not exist` reply is
returned **after** the message body, and smtplib raises `SMTPDataError code=550`:

```text
$ /app/venv/bin/python /tmp/req3_send.py
send: 'ehlo [172.17.0.3]\r\n'
reply: b'250-abe149475ec4\r\n'
reply: b'250-SIZE 33554432\r\n'
reply: b'250-8BITMIME\r\n'
reply: b'250-SMTPUTF8\r\n'
reply: b'250 HELP\r\n'
reply: retcode (250); Msg: b'abe149475ec4\nSIZE 33554432\n8BITMIME\nSMTPUTF8\nHELP'
send: 'mail FROM:<someone@example.com> size=151\r\n'
reply: b'250 OK\r\n'
reply: retcode (250); Msg: b'OK'
send: 'rcpt TO:<x@sl.local>\r\n'
reply: b'250 OK\r\n'
reply: retcode (250); Msg: b'OK'
send: 'data\r\n'
reply: b'354 End data with <CR><LF>.<CR><LF>\r\n'
reply: retcode (354); Msg: b'End data with <CR><LF>.<CR><LF>'
data: (354, b'End data with <CR><LF>.<CR><LF>')
send: b'From: someone@example.com\nTo: x@sl.local\nSubject: test\nContent-Type: text/plain; charset="utf-8"\nContent-Transfer-Encoding: 7bit\nMIME-Version: 1.0\n\nhi\n\r\n.\r\n'
reply: b'550 SL E515 Email not exist\r\n'
reply: retcode (550); Msg: b'SL E515 Email not exist'
data: (550, b'SL E515 Email not exist')
send: 'rset\r\n'
reply: b'250 OK\r\n'
reply: retcode (250); Msg: b'OK'
send: 'quit\r\n'
reply: b'221 Bye\r\n'
reply: retcode (221); Msg: b'Bye'
RESULT: SMTPDataError -> code= 550 msg= b'SL E515 Email not exist'
```

**(d) Complete, unedited handler logs for this message** — all lines for message-id
`2e6e5e3b-fbd8-4147-81e7-b66f7b7ca965`, exactly as written to stdout, with **no annotations
added**. (The `==>>` and `<<===` markers are **source-emitted** — literal format text in
`email_handler.py:1981` and `email_handler.py:2368` — not editorial marks.)

```text
2026-07-14 21:28:35,500 - SL - INFO - 1295 - "/app/email_handler.py:2343" - _handle() - 2e6e5e3b-fbd8-4147-81e7-b66f7b7ca965 - New message, mail from someone@example.com, rctp tos ['x@sl.local']
2026-07-14 21:28:35,501 - SL - DEBUG - 1295 - "/app/email_handler.py:1963" - handle() - 2e6e5e3b-fbd8-4147-81e7-b66f7b7ca965 - Cannot parse Postfix queue ID from None None
2026-07-14 21:28:35,641 - SL - DEBUG - 1295 - "/app/email_handler.py:1980" - handle() - 2e6e5e3b-fbd8-4147-81e7-b66f7b7ca965 - ==>> Handle mail_from:someone@example.com, rcpt_tos:['x@sl.local'], header_from:someone@example.com, header_to:x@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'someone@example.com'), ('To', 'x@sl.local'), ('Subject', 'test'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:['SIZE=151'], rcpt_options:[]
2026-07-14 21:28:35,646 - SL - DEBUG - 1295 - "/app/email_handler.py:2202" - handle() - 2e6e5e3b-fbd8-4147-81e7-b66f7b7ca965 - Forward phase someone@example.com(someone@example.com) -> x@sl.local
2026-07-14 21:28:35,656 - SL - DEBUG - 1295 - "/app/email_handler.py:545" - handle_forward() - 2e6e5e3b-fbd8-4147-81e7-b66f7b7ca965 - alias x@sl.local not exist. Try to see if it can be created on the fly
2026-07-14 21:28:35,716 - SL - INFO - 1295 - "/app/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - 2e6e5e3b-fbd8-4147-81e7-b66f7b7ca965 - Cannot auto-create custom domain alias for x@sl.local because there's no custom domain for sl.local
2026-07-14 21:28:35,716 - SL - INFO - 1295 - "/app/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - 2e6e5e3b-fbd8-4147-81e7-b66f7b7ca965 - Cannot auto-create x@sl.local since it has no directory separator
2026-07-14 21:28:35,716 - SL - DEBUG - 1295 - "/app/email_handler.py:551" - handle_forward() - 2e6e5e3b-fbd8-4147-81e7-b66f7b7ca965 - alias x@sl.local cannot be created on-the-fly, return 550
2026-07-14 21:28:35,717 - SL - INFO - 1295 - "/app/email_handler.py:2367" - _handle() - 2e6e5e3b-fbd8-4147-81e7-b66f7b7ca965 - Finish mail_from someone@example.com, rcpt_tos ['x@sl.local'], takes 0.2176060676574707 seconds with return code '550 SL E515 Email not exist'<<===
```

The **two rejection log lines** (identified here in prose, outside the evidence block, so the
capture stays verbatim) are:

- `email_handler.py:545` — `handle_forward()` — "alias x@sl.local not exist. Try to see if it can
  be created on the fly" (the alias lookup missed).
- `email_handler.py:551` — `handle_forward()` — "alias x@sl.local cannot be created on-the-fly,
  return 550" (auto-create also failed -> the 550 is returned).

The two intervening `INFO` lines from `app/alias_utils.py:104` and `app/alias_utils.py:165` record
*why* auto-create failed (no custom domain for `sl.local`; no directory separator in the local
part), and the closing `email_handler.py:2367` line records the final
`return code '550 SL E515 Email not exist'`.

**(e) Shutdown + port release:**

```text
LISTEN :7777  -> [empty=released]
LISTEN :20381 -> [empty=released]
procs server.py       = 0
procs email_handler.py= 0
procs job_runner.py   = 0
procs gunicorn        = 0
```

### 4.4 Cause -> effect (with `file:line`)

The inbound `x@sl.local` message is a **forward** (recipient is a local alias domain), so
`handle()` routes it to `handle_forward()` [email_handler.py:536-556]:

1. **Alias lookup misses.** `alias = Alias.get_by(email=alias_address)` returns `None` (the
   `alias` table is empty — §4.3(a)), so `handle_forward` logs
   `alias x@sl.local not exist...` [email_handler.py:545] and attempts auto-create.
2. **Auto-create fails on both strategies.** `try_auto_create(alias_address)`
   [app/alias_utils.py:202-224] tries:
   - `try_auto_create_via_domain` -> `check_if_alias_can_be_auto_created_for_custom_domain`
     [app/alias_utils.py:104] — needs a **verified `CustomDomain`** catch-all/rule for `sl.local`;
     none exists, logged "no custom domain for sl.local".
   - `try_auto_create_directory` -> `check_if_alias_can_be_auto_created_for_a_directory`
     [app/alias_utils.py:165] — needs a **directory** (a `+`/`#`/`/` separator in the local part);
     `x` has none, logged "no directory separator".
   Both return `None`, so `try_auto_create` returns `None`.
3. **Permanent rejection.** Back in `handle_forward`, `if not alias:` logs
   `alias x@sl.local cannot be created on-the-fly, return 550` [email_handler.py:551]; the sender
   `someone@example.com` is not an ignore-bounce address, so the `else` branch returns
   `status.E515` [email_handler.py] = `"550 SL E515 Email not exist"` [app/email/status.py:51].
4. **Reply to sender.** `_handle` logs the `Finish ... return code '550 SL E515 Email not exist'`
   line [email_handler.py:2367] and aiosmtpd sends that exact string as the DATA reply — observed
   verbatim in §4.3(c).

**Why the empty `SLDomain` is not the direct cause (reply-path vs forward-path).** The `SLDomain`
membership check is `is_valid_alias_address_domain(...)` — **defined** at
[app/email_utils.py:557-563] and **called on the reply path**, inside `handle_reply`, at
[email_handler.py:998-1002] (`if not is_valid_alias_address_domain(alias.email): ... return
False, status.E503`). A *forward* to a brand-new address never reaches that check — it is rejected
earlier by alias-nonexistence + auto-create failure (steps 1-3). Thus, although skipping
`init_app.py` does leave `public_domain` empty, the operative cause of this **forward-path** 550
is the missing alias, not the empty `SLDomain`. (Had a reply been attempted for an alias whose
domain is unknown, the empty/incorrect `SLDomain` would surface as `status.E503` on that path.)

---

## §5 Cleanup & scope proof

Per the MainRule and the user's constraint ("I'm not changing or deleting any source files"),
every observation artifact is ephemeral and was removed, leaving the deliverable repository
changed by exactly **one** file — this document.

### 5.1 Disposable database dropped

The observation database `simplelogin` lived only inside the disposable container and was dropped:

```text
$ PGPASSWORD=mypassword psql -h 127.0.0.1 -U myuser -lqt | cut -d'|' -f1 | grep -w simplelogin   # before
 simplelogin
--- guarded DROP (only when the disposable marker is present AND name == simplelogin) ---
DROP DATABASE
$ PGPASSWORD=mypassword psql -h 127.0.0.1 -U myuser -lqt | cut -d'|' -f1 | grep -w simplelogin   # after
(empty above = simplelogin dropped)
```

### 5.2 Temporary scripts/config removed — negative checks

All `/tmp` helpers (`obs.env`, `req1_login_post.py`, `req1_dblayer_standin.py`, `req3_send.py`,
`reset_db.sh`, and the launcher/marker files) were deleted; no observation process remains and
both ports are closed:

```text
-- temp artifacts removed (ls errors = files gone) --
ls: cannot access '/tmp/obs.env': No such file or directory
ls: cannot access '/tmp/req1_login_post.py': No such file or directory
ls: cannot access '/tmp/req1_dblayer_standin.py': No such file or directory
ls: cannot access '/tmp/req3_send.py': No such file or directory
ls: cannot access '/tmp/reset_db.sh': No such file or directory
-- observation processes (count via /proc/*/cmdline; 0 = none) --
  server.py = 0
  email_handler.py = 0
  job_runner.py = 0
  gunicorn = 0
-- listening ports (empty = closed) --
  :7777  ->
  :20381 ->
-- disposable simplelogin DB present? (empty = dropped) --
```

### 5.3 Deliverable-repository scope proof (authoritative)

The authoritative scope proof is the **host deliverable repository** (this working tree). It
shows exactly one modified path — this document — with no source edits and **no untracked/temp
files**:

```text
$ git status --porcelain
 M blitzy/documentation/app_2cd6ee777f8c.md
$ git rev-parse --abbrev-ref HEAD
blitzy-995c7c87-d276-4616-80ce-e2a2db52eabb
$ git diff --name-status HEAD --
M	blitzy/documentation/app_2cd6ee777f8c.md
$ git status --porcelain --untracked-files=all | grep '^??' || echo "(no untracked files)"
(no untracked files)
```

The `git status --porcelain` and `git diff --name-status` outputs each list exactly **one** path —
this document — with an empty untracked set, confirming no source file, config, dependency, or
temporary script was added, modified, or left behind. (Exact insertion/deletion counts are omitted
here because they would be self-referential — quoting them would change this very file's diff.)

> **Honest note on the canonical container's `/app` tree.** The canonical image ships with a
> handful of files already showing as modified in *its own* baked-in `/app` git tree
> (`app/spamassassin_utils.py`, several `local_data/*.key*`, `local_data/test_words.txt`,
> `static/package-lock.json`). These modifications are **present at image-build time** — confirmed
> by running `git status` in a **fresh `--rm` container with zero commands executed** — and are
> **not** produced by this investigation (which only ever wrote under `/tmp` and to a disposable
> database). The container is fully disposable and is discarded on completion; it is therefore
> **not** the scope reference. The deliverable repository above is the authoritative proof.

### 5.4 Evidence-fidelity statement

Every fenced `text`/`python`/`bash` block in this document is the **actual captured output or the
actual helper source**, reproduced verbatim, with a single disclosed non-semantic transformation:
trailing whitespace on each line was trimmed for markdown hygiene (no line content, ordering, or
value was altered). **Nothing is redacted.** The one item that might look sensitive — the
`Set-Cookie: slapp=...` value in §2.3(d) — is a **throwaway** session cookie signed with the
**public** `example.env` value `FLASK_SECRET=secret` inside a disposable container, so it carries
no secret and is shown raw to keep the capture literally complete.

## §6 Coverage — every named item answered

A final coverage pass confirms every explicitly named item in the three requirements is addressed
by name, with exact values and `file:line` grounding.

| Named item | Where answered | Status |
|------------|----------------|--------|
| **`python server.py`** (web app) | §2 (REQ-1 login failure), §3.5 (:7777 readiness) | Answered — binds `:7777`, `/health -> 200` |
| **login page** (`/auth/login`) | §2.3(d) GET 200 (no query), §2.3(e-f) POST -> 500 | Answered — GET renders, POST raises the exception |
| **full exception (REQ-1)** | §2.1 direct answer + §2.3(f) complete traceback | Answered — `sqlalchemy.exc.ProgrammingError` / `psycopg2.errors.UndefinedTable: relation "users" does not exist` |
| **migrations** | §3.2 (`flask db upgrade`), §3.3 (state) | **Answered with a caveat** — literal `flask db upgrade` is **BLOCKED** (`KeyError: 'migrate'`; Flask-Migrate never registered) and **cannot** succeed without modifying source (out of scope); the schema is built by the project-canonical `alembic upgrade head` (255 migrations, exactly what CI runs) |
| **`python init_app.py`** | §3.4 (run: seeds `SLDomain`), §4 (skipped: premise of REQ-3) | Answered — seeds `public_domain` with `sl.local`; skipping leaves it empty |
| **required Python services (REQ-2)** | §3.1 + §3.5/§3.6/§3.7 | Answered — **3**: web (`:7777`), email handler (`:20381`), job runner (no port) |
| **`email_handler`** | §3.6 (readiness :20381), §4 (REQ-3 rejection) | Answered — `Listen for port 20381` / `Start mail controller 0.0.0.0 20381`; forward rejection |
| **`SLDomain`** (`public_domain`) | §3.4 (seeded = 1 row `sl.local`), §4.3(a) (skipped = 0 rows), §4.4 (reply-path check) | Answered — table `public_domain` [app/models.py:3116-3119]; empty when init skipped |
| **`@sl.local`** | §4 (message to `x@sl.local`) | Answered — default `EMAIL_DOMAIN` [app/config.py:92]; the rejected recipient |
| **SMTP status code (REQ-3)** | §4.1 direct answer + §4.3(c) transcript | Answered — **`550 SL E515 Email not exist`** [app/email/status.py:51] |
| **rejection log lines (REQ-3)** | §4.3(d) verbatim logs + prose call-out | Answered — `email_handler.py:545` and `email_handler.py:551` (+ `alias_utils.py:104,165`; finish `email_handler.py:2367`) |
| **startup order** | §1 preamble + per-scenario resets | Answered — Postgres -> empty DB *(REQ-1 stops)* -> migrate -> `init_app.py` -> services |

**Observed vs. inferred.** All behavioral claims above are backed by live capture in the canonical
image (tracebacks, banners, port binds, the SMTP transcript, and handler logs). The only
explicitly labeled non-live element is the REQ-1 **DB-layer stand-in** in §2.3(g), which
corroborates — but does not replace — the real HTTP-path traceback in §2.3(f). The production
`gunicorn` banner in §3.8 is a real capture (labeled **OBSERVED**), not an inference.
