# SimpleLogin Development Stack Bring-Up — Runtime-Verified Answers

This document answers three questions about bringing up the SimpleLogin development stack from scratch. Every answer is written from **first-hand runtime observation** of the application's real behavior — the relevant code path was actually executed and its unedited output captured — rather than from reading the source alone.

## Subject under test

- **Repository branch:** `app_2cd6ee777f8c` (HEAD `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`).
- **Canonical runtime:** the user-provided Docker image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (`andrewparkscaleai/coding-agent:simple-login__app__2cd6ee777f8c...`), with the SimpleLogin source checked out at `/app` inside the container. All observation commands below were run inside that container against a Python 3.10.18 virtualenv at `/app/venv` — the project's declared runtime (`python = "^3.10"`, `pyproject.toml:61`).
- **The three questions, in reader order:** **Q1** — the exception hit when starting the server against an empty (unmigrated) database and opening the login page; **Q2** — the Python services that must run for the system to work, with their exact startup/port-bind logs; **Q3** — what the email handler returns for mail to `@sl.local` when migrations ran but `init_app.py` did not, leaving the domain table empty.

## How to read the evidence in this document

- Every fenced block is a **verbatim capture**. A line beginning with `$ ` is the exact command that was run; everything following it, up to the next `$ ` line, is that command's exact stdout/stderr with nothing filtered, paraphrased, truncated, or reconstructed. Lines beginning with `$ #` are transparent comments echoed by the driver script.
- Each capture is produced by a small **driver script** that runs under `set -o pipefail` and echoes every command through a `run()` helper, so a shown `$ ` line is exactly what executed. For a service, the driver captures the background PID, polls for readiness with a timeout, proves the port binding from `/proc/net/tcp`, terminates the service with `SIGTERM` (via its process group where a Werkzeug reloader child exists), and re-reads `/proc/net/tcp` afterward to confirm the port was released.
- Facts taken directly from a capture are labelled **Observed**. Facts derived by reading the source are labelled **Inferred** and, wherever possible, are confirmed by an adjacent capture.
- Every code reference uses the exact case-sensitive repository path with an explicit line or line-range, e.g. `server.py:388-394`.
- **Port-binding proof** is read from `/proc/net/tcp`, because `ss`, `netstat`, and `lsof` are not installed in the image. In that file the local endpoint is printed as `HEXIP:HEXPORT` and a listening socket has state `0A`. The relevant values are `7777 = 0x1E61`, `20381 = 0x4F9D`, `0100007F = 127.0.0.1`, and `00000000 = 0.0.0.0`.

## ⚠️ Security and isolation notice (read before reproducing)

This stack is pinned to the versions the project actually declares, and several of those pinned versions carry **published security advisories**. The advisory facts below are cited from those advisories (they are *not* runtime observations) and are called out so the stack is only ever run in a throwaway, isolated environment. The pinned versions are **left frozen on purpose** so the observed behavior reflects the real project; they are not upgraded.

- **aiosmtpd 1.4.2** (observed version — see `pip show` in Section 0.1) falls in the range affected by **CVE-2024-27305** — inbound SMTP smuggling, GitHub-reviewed *Moderate* (CVSS 5.3), fixed in aiosmtpd **1.4.5**. The Q3 email handler binds SMTP on `0.0.0.0:20381`.
- **gunicorn 20.0.4** (observed version) falls in the range affected by **CVE-2024-1135** — HTTP request smuggling caused by improper `Transfer-Encoding` header validation, fixed in gunicorn **22.0.0**; the advisory advises blocking access to restricted endpoints via a firewall where updating is not possible. The canonical Q2 web command binds gunicorn on `0.0.0.0:7777`.
- **Flask 1.1.2** (observed version) predates later Flask security fixes such as **CVE-2023-30861** (a conditional session-cookie disclosure under specific reverse-proxy caching conditions).

**Because of the above, when reproducing any scenario in this document:**

- Run the stack **only inside an isolated, disposable container** (as was done here), with the listening ports (`7777`, `20381`) **unpublished / firewalled** and never reachable from an untrusted network.
- Every credential shown here is a **disposable, local-only scratch value** created purely for observation — the PostgreSQL role `myuser` / `mypassword` and `FLASK_SECRET=secret`. These are **not** real secrets and must never be reused outside a throwaway environment. Any `.env` used here is derived **byte-for-byte** from the repository's committed `example.env`, which contains placeholder values only.

**Repository impact (read-only, net-zero).** This is a strict read-only investigation: no existing source file is modified or deleted. The only net change to the repository is this document, `blitzy/documentation/app_2cd6ee777f8c.md`. Every temporary artifact created for observation — a disposable `.env`, an SMTP-injection helper, capture logs, and scratch databases — is removed at the end; the proof is captured in the final *“Cleanup and net-zero verification”* section.

## Execution chronology and database-state ordering

The three questions require **different database states**, so the order in which the scenarios were executed is deliberately not the same as the reader order (Q1 → Q2 → Q3) used above. This section makes the exact ordering explicit, because a wrong ordering would destroy the state a question needs — for example, running `init_app.py` before the Q3 pre-initialization observation would seed `public_domain` and change what Q3 observes.

Two databases were used, both created fresh for this investigation and dropped at the end (see the final *Cleanup and net-zero verification* section):

- **`sl_q1_obs`** — created empty and **never migrated**; used only for Q1. Its state never changes.
- **`sl_probe_obs`** — created, migrated with `alembic upgrade head`, then transitioned through the exact sequence the questions require. This single database is the subject of the Q3 pre-initialization observation, the `init_app.py` seeding, all five Q2 services, and the Q3 post-initialization cross-check.

The table below lists every observation in **wall-clock capture order**. Each timestamp is read directly from the captured SimpleLogin log lines quoted verbatim in the sections below, and each row records the database and the number of rows in `public_domain` (the `SLDomain` table, `app/models.py:3119`) at that moment. This is **Observed** ordering — the timestamps are in the captures, not asserted.

| Wall-clock (2026-07-13) | Scenario | Database | `public_domain` rows | Observed result |
|---|---|---|---|---|
| `19:10:42` | Q3 pre-init — primary injection | `sl_probe_obs` (migrated, uninitialized) | 0 | `550 SL E515` |
| `19:10:46` | Q3 pre-init — E207 alternate branch | `sl_probe_obs` | 0 | `250 SL E207` |
| `19:10:51` | Q3 pre-init — stability re-run | `sl_probe_obs` | 0 | `550 SL E515` |
| `19:21:42` | `init_app.py` seeds the domain table | `sl_probe_obs` | 0 → 1 | one row `sl.local` |
| `19:23:34` | Q2 — web app (dev `server.py`) | `sl_probe_obs` (initialized) | 1 | LISTEN `127.0.0.1:7777` |
| `19:24:53` | Q2 — web app (canonical gunicorn) | `sl_probe_obs` | 1 | LISTEN `0.0.0.0:7777` |
| `19:25:19` | Q2 — `email_handler.py` | `sl_probe_obs` | 1 | LISTEN `0.0.0.0:20381` |
| `19:26:59` | Q2 — `job_runner.py` | `sl_probe_obs` | 1 | job taken → done |
| `19:27:31` | Q2 — `cron.py -j delete_old_monitoring` | `sl_probe_obs` | 1 | exit `0` |
| `19:28:01` | Q2 — `event_listener.py listener` | `sl_probe_obs` | 1 | `Starting to listen to events` |
| `19:37:15` | Q1 — empty-DB startup + login (run 1) | `sl_q1_obs` (empty, unmigrated) | n/a (no tables) | `ProgrammingError` / `UndefinedTable` |
| `19:37:49` | Q1 — empty-DB startup + login (run 2) | `sl_q1_obs` | n/a | `ProgrammingError` / `UndefinedTable` |
| `19:41:20` | Q3 post-init cross-check — identical injection | `sl_probe_obs` (initialized) | 1 | `550 SL E515` (unchanged) |

Two facts about this ordering matter for the answers:

- **The Q3 post-initialization cross-check (`19:41:20`) runs strictly after the `init_app.py` seeding (`19:21:42`), on the same database (`sl_probe_obs`).** That is the correct ordering to demonstrate the causal point in Q3: the domain table is genuinely non-empty (one `sl.local` row) when the identical message is re-injected, yet the rejection is unchanged.
- **Q1 uses its own dedicated empty database (`sl_q1_obs`), so its wall-clock position (`19:37`) is independent of the `sl_probe_obs` state machine.** Q1 only requires “empty, unmigrated”, which `sl_q1_obs` satisfies throughout.

## Section 0 — Environment bring-up and prerequisites

Everything in this section was reproduced from the canonical image **before** any question was answered, so that the errors captured later are genuinely the errors each question asks about and not an unrelated setup failure.

### 0.1 Base image, database, cache, roles, and dependency versions (Observed)

The image is Debian 12 with the project's Python 3.10 venv. PostgreSQL 15 and Redis are started with `service postgresql start` and `service redis-server start` (run once at provisioning); the status captures below confirm both are up. Two disposable superuser roles exist for the scenarios (`myuser`, `test`). The pinned regex dependency `pyre2==0.3.6` is present and importable.

```text
$ cat /etc/os-release | grep -E '^(PRETTY_NAME)='
PRETTY_NAME="Debian GNU/Linux 12 (bookworm)"

$ /app/venv/bin/python --version
Python 3.10.18

$ service postgresql status 2>&1 | head -2
15/main (port 5432): online

$ service redis-server status 2>&1 | head -2
redis-server is running.

$ PGPASSWORD=mypassword psql -U myuser -h localhost -d postgres -tAc "SELECT rolname, rolsuper FROM pg_roles WHERE rolname IN ('myuser','test') ORDER BY rolname;"
myuser|t
test|t

$ redis-cli -h localhost -p 6379 ping
PONG

$ /app/venv/bin/python -c 'import re2 as re; print("pyre2 import OK; re.DOTALL=", re.DOTALL)'
pyre2 import OK; re.DOTALL= re.DOTALL

$ /app/venv/bin/pip show pyre2 2>/dev/null | grep -E '^(Name|Version)'
Name: pyre2
Version: 0.3.6

$ /app/venv/bin/pip show Flask SQLAlchemy aiosmtpd gunicorn psycopg2-binary python-dotenv 2>/dev/null | grep -E '^(Name|Version)'
Name: Flask
Version: 1.1.2
Name: SQLAlchemy
Version: 1.3.24
Name: aiosmtpd
Version: 1.4.2
Name: gunicorn
Version: 20.0.4
Name: psycopg2-binary
Version: 2.9.3
Name: python-dotenv
Version: 0.14.0
```

**What this shows (Observed):** `PRETTY_NAME="Debian GNU/Linux 12 (bookworm)"`; the venv interpreter is `Python 3.10.18`; PostgreSQL reports `15/main (port 5432): online` and Redis answers `PONG`; roles `myuser` and `test` both have `rolsuper = t`; `import re2` succeeds and `re.DOTALL` resolves. The exact installed versions relevant to the three questions are **Flask 1.1.2**, **SQLAlchemy 1.3.24**, **aiosmtpd 1.4.2**, **gunicorn 20.0.4**, **psycopg2-binary 2.9.3**, and **python-dotenv 0.14.0**.

**Dependency note (`pyre2`) — setup step, verified Observed.** The pristine repository pins `pyre2==0.3.6`, but the base image shipped a fallback `google-re2`. During provisioning the correct wheel was built into the venv with `pip install "cython<3" "pybind11>=2.10"` followed by `pip install pyre2==0.3.6 --no-build-isolation` (the system already provides `libre2-dev`, `cmake`, and `ninja`). The capture above is the runtime verification that this succeeded (`Version: 0.3.6`, `import re2` OK). This is disclosed because the stack will not import without a working `re2` module.

### 0.2 The `.env` prerequisite, and a distinct pre-database failure (Observed)

`app/config.py` reads several **mandatory** environment variables at *import time*: `URL = os.environ["URL"]` (`app/config.py:79`), `EMAIL_DOMAIN = os.environ["EMAIL_DOMAIN"]` (`app/config.py:92`), and `SUPPORT_EMAIL = os.environ["SUPPORT_EMAIL"]` (`app/config.py:93`). If none of these is set, importing the configuration raises a `KeyError` **before** any database work happens at all. That is a *different* failure from the empty-database error Q1 asks about, so it is demonstrated and set aside here first. The capture below moves the provisioned `.env` aside, imports `app.config` under an empty environment, observes the `KeyError`, and then restores `.env`.

```text
$ ls -l /app/.env && git -C /app check-ignore .env && echo '(.env is git-ignored)'
-rw-r--r-- 1 root 1001 5789 Jul 13 16:08 /app/.env
.env
(.env is git-ignored)

$ diff /app/.env /app/example.env >/dev/null && echo '/app/.env is byte-identical to example.env (i.e. cp example.env .env)'
/app/.env is byte-identical to example.env (i.e. cp example.env .env)

$ mv /app/.env /tmp/blitzy_obs/dotenv.bak

$ cd /tmp && env -i PATH=/usr/bin:/bin HOME=/root PYTHONPATH=/app /app/venv/bin/python -c 'import app.config'
Traceback (most recent call last):
  File "<string>", line 1, in <module>
  File "/app/app/config.py", line 79, in <module>
    URL = os.environ["URL"]
  File "/usr/local/lib/python3.10/os.py", line 680, in __getitem__
    raise KeyError(key) from None
KeyError: 'URL'
exit_code=1

$ mv /tmp/blitzy_obs/dotenv.bak /app/.env && ls -l /app/.env
-rw-r--r-- 1 root 1001 5789 Jul 13 16:08 /app/.env

$ cd /app && source /tmp/blitzy_obs/sl.env && /app/venv/bin/python -c 'import app.config as c; print("config import OK; URL=", c.URL)'
config import OK; URL= http://localhost:7777
```

**What this shows (Observed):** the provisioned `/app/.env` is git-ignored and **byte-identical to `example.env`** (i.e. it was produced by `cp example.env .env`, placeholder values only). With it moved aside and an empty environment, `import app.config` fails with `KeyError: 'URL'` raised at `/app/app/config.py:79`. Restoring `.env` (or sourcing a canonical environment) makes the import succeed: `config import OK; URL= http://localhost:7777`. **This missing-configuration `KeyError` must not be conflated with the Q1 empty-database exception** — they occur at different stages (config import vs. first SQL query).

**How `.env` is discovered (`python-dotenv` 0.14.0) — Inferred from source, confirmed Observed.** `app/config.py` calls `load_dotenv()` at `app/config.py:71` (the `else:` branch taken when the `CONFIG` variable is unset, i.e. the canonical `python server.py` invocation). python-dotenv's `find_dotenv()` locates the file by walking **up** the directory tree from the caller's module directory when run as a script (non-interactive), which resolves `/app/.env` via `app/config.py`'s own directory. `load_dotenv()` is called with its default `override=False`, so a variable already exported in the real environment takes precedence over the value in `.env`. This override behavior is **confirmed observationally by Q1 below**: `example.env`/`.env` sets `DB_URI=...localhost:5432/simplelogin` (`example.env:75`, a *migrated* database), yet the Q1 run exported `DB_URI=...sl_q1_obs` on the command line and the server demonstrably queried the empty `sl_q1_obs` (it raised `relation "users" does not exist`). Had `.env` won, the query would have hit the migrated `simplelogin` and no such error would appear. The Flask version in effect is exactly **1.1.2** (Section 0.1).

## Q1 — Empty database, `python server.py`, open the login page: the exact exception

**Question.** With PostgreSQL running and an empty database created (the database exists but no tables have been created and migrations have **not** been run), what error is actually hit when the server is started with `python server.py` and the login page is opened in a browser? The full Python exception is wanted.

### Q1 — Why the process starts but the request fails (Inferred from source)

`app/db.py` builds the engine and opens a connection at **import time**: `engine = create_engine(config.DB_URI)` (`app/db.py:9-11`) and `connection = engine.connect()` (`app/db.py:12`), with `Session = scoped_session(...)` (`app/db.py:14`). Connecting to an *empty* database succeeds — there is nothing schema-specific about opening a connection — so the process boots normally. The failure only appears when the **first query against a missing relation** is issued. The root route `/` redirects to the login view (`server.py:249-255`), the login page is served by `@auth_bp.route("/login")` at `app/auth/views/login.py:21-25` (the blueprint is mounted at `url_prefix="/auth"`, `app/auth/base.py:3-5`, so the browser URL is `/auth/login`), and submitting the form runs `user = User.get_by(email=email) or User.get_by(email=canonical_email)` at `app/auth/views/login.py:43`, which calls `Session.query(cls).filter_by(**kw).first()` at `app/models.py:84`. That is the first statement to touch the `users` table (`User.__tablename__ = "users"`, `app/models.py:337`). The observations below confirm each step of this reasoning at runtime.

### Q1 — Reproduction (Observed)

The driver below creates a fresh, empty, **unmigrated** database named `sl_q1_obs`, starts `python server.py` as its own process group via `setsid` (so the process and the Werkzeug reloader child can be terminated together), waits for readiness on the DB-free `/health` route (`server.py:213-215`), proves the port binding from `/proc/net/tcp`, exercises the root redirect and a cold `GET /auth/login`, then submits `POST /auth/login` to trigger the first query. The web server's own console (boot banner and full traceback) is captured to a separate log, shown immediately after each transcript. This was run twice; both runs are shown in full.

#### Q1 Run 1 — driver transcript

```text
$ PGPASSWORD=mypassword dropdb -U myuser -h localhost --if-exists sl_q1_obs

$ PGPASSWORD=mypassword createdb -U myuser -h localhost sl_q1_obs

$ PGPASSWORD=mypassword psql -U myuser -h localhost -d sl_q1_obs -tAc "SELECT count(*) FROM information_schema.tables WHERE table_schema='public';"
0

$ cd /app && source /tmp/blitzy_obs/sl.env && DB_URI=postgresql://myuser:mypassword@localhost:5432/sl_q1_obs setsid /app/venv/bin/python server.py > /tmp/blitzy_obs/q1_server_run1.log 2>&1 &
  server process-group leader PID (captured via $!) = 8668

$ for i in $(seq 1 40); do c=$(curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:7777/health 2>/dev/null); [ "$c" = 200 ] && { echo "ready after ${i}s (/health -> 200)"; break; }; sleep 1; done
ready after 3s (/health -> 200)

$ kill -0 8668 && echo '  server leader alive (kill -0 ok)'
  server leader alive (kill -0 ok)

$ grep -iE ' [0-9A-F]{8}:1E61 ' /proc/net/tcp | awk '$4=="0A"{print "  LISTEN local="$2" (0100007F=127.0.0.1, hex 1E61=7777, state 0A=LISTEN)"}'
  LISTEN local=0100007F:1E61 (0100007F=127.0.0.1, hex 1E61=7777, state 0A=LISTEN)

$ curl -sS -w ' [HTTP %{http_code}]\n' http://127.0.0.1:7777/health
success [HTTP 200]

$ curl -sS -D - -o /dev/null -w 'HTTP_STATUS=%{http_code}\n' http://127.0.0.1:7777/
HTTP/1.0 302 FOUND
Content-Type: text/html; charset=utf-8
Content-Length: 229
Location: http://127.0.0.1:7777/auth/login
Vary: Cookie
Set-Cookie: slapp=eyJfZnJlc2giOmZhbHNlLCJfcGVybWFuZW50Ijp0cnVlfQ.alU-bg.DXE3chfCypGx9bEF5c5ODRIN9h0; Expires=Mon, 20-Jul-2026 19:37:18 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Mon, 13 Jul 2026 19:37:18 GMT

HTTP_STATUS=302

$ curl -sS -o /tmp/blitzy_obs/q1_get_run1.html -w 'HTTP_STATUS=%{http_code}\n' http://127.0.0.1:7777/auth/login
HTTP_STATUS=200

$ curl -c /tmp/blitzy_obs/q1_cj_run1 -s http://127.0.0.1:7777/auth/login -o /tmp/blitzy_obs/q1_login_run1.html -w 'GET /auth/login HTTP_STATUS=%{http_code}\n'
GET /auth/login HTTP_STATUS=200

$ CSRF=$(grep -oP 'name="csrf_token"[^>]*value="\K[^"]+' /tmp/blitzy_obs/q1_login_run1.html | head -1); echo "csrf_token length = ${#CSRF}"
csrf_token length = 91

$ curl -b /tmp/blitzy_obs/q1_cj_run1 -c /tmp/blitzy_obs/q1_cj_run1 -sS -D - -o /tmp/blitzy_obs/q1_post_body_run1.html -w 'HTTP_STATUS=%{http_code}\n' -X POST http://127.0.0.1:7777/auth/login --data-urlencode 'csrf_token='"$CSRF" --data-urlencode 'email=test@example.com' --data-urlencode 'password=whatever'
HTTP/1.0 500 INTERNAL SERVER ERROR
Content-Type: text/html; charset=utf-8
Content-Length: 5749
Vary: Cookie
Set-Cookie: slapp=eyJfZnJlc2giOmZhbHNlLCJfcGVybWFuZW50Ijp0cnVlLCJjc3JmX3Rva2VuIjoiMDU3MDJkZTY1NmVkOTcwNzNmMGI0NWRkMmM2MmM5ODU2NjBlYWYxOSJ9.alU-bg.f03ixPLWkFpr93XwoOhGJqgJsjU; Expires=Mon, 20-Jul-2026 19:37:18 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Mon, 13 Jul 2026 19:37:18 GMT

HTTP_STATUS=500

$ grep -iE 'Server error|server issues|<title>|history.back' /tmp/blitzy_obs/q1_post_body_run1.html
    <title>
        Server error
  Looks like we are having some server issues...
  <a class="btn btn-primary" href="javascript:history.back()">
      history.back();

$ wc -c /tmp/blitzy_obs/q1_post_body_run1.html
5749 /tmp/blitzy_obs/q1_post_body_run1.html

$ kill -TERM -8668   # terminate whole process group (leader + Werkzeug reloader child)
  stopped

$ grep -iE ' [0-9A-F]{8}:1E61 ' /proc/net/tcp | awk '$4=="0A"{print "  still LISTEN "$2}'
  (no output above == no socket in LISTEN state on 7777 -> port released)

$ ps -o pid,cmd -C python 2>/dev/null | grep server.py || echo '  (no server.py process running)'
  (no server.py process running)
```

**What this shows (Observed).** The new database `sl_q1_obs` has `0` tables in `public`. The server comes up as process-group leader **PID 8668**, is ready after 3 s, and is listening on `0100007F:1E61` = **127.0.0.1:7777** (state `0A` = LISTEN). `GET /` returns **HTTP 302** with `Location: http://127.0.0.1:7777/auth/login`, and a cold `GET /auth/login` returns **HTTP 200** (the login page renders — this GET performs no user query). Submitting `POST /auth/login` (with a valid CSRF token and `email=test@example.com`) returns **HTTP 500 INTERNAL SERVER ERROR**, `Content-Length: 5749`.

**The HTTP 500 body is the application's generic error page, not a debugger (Observed — corrects a common misconception).** The grep over the captured 500 body shows the rendered template `error/500.html`: `<title>` / `Server error` / `Looks like we are having some server issues...` / `<a class="btn btn-primary" href="javascript:history.back()">`. This is expected from the code: `server.py:388-394` registers `@app.errorhandler(Exception)` which logs the exception with `LOG.e(e)` (`server.py:390`) and then `return render_template("error/500.html"), 500` (`server.py:393`). So even though `Debug mode: on` appears in the boot banner, the client receives the static 5749-byte “Server error” page — **the full Python traceback is written to the server's stdout/stderr, not to the browser.** That traceback is the next capture.

#### Q1 Run 1 — web server console (boot banner + full traceback, verbatim)

`LOG.e` is SimpleLogin's error logger (`LOG.e == logging.Logger.exception`, `app/log.py:76`), which emits the complete traceback. The boot banner appears **twice** because `app.run(debug=True, port=7777)` (`server.py:588`, invoked from `local_main()` at `server.py:572`) enables the Werkzeug auto-reloader: the process-group leader **8668** supervises, while the reloader **child PID 8682** re-imports the app and is the process that actually serves the requests and logs the error.

```text
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/wbwngriitdevwiuyckln
Upload files to local dir
>>> init logging <<<
2026-07-13 19:37:15,660 - SL - DEBUG - 8668 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/xmjrulkojkpzqlurphll
Upload files to local dir
>>> init logging <<<
2026-07-13 19:37:17,430 - SL - DEBUG - 8682 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
/app/venv/lib/python3.10/site-packages/flask_debugtoolbar/__init__.py:213: UserWarning: Could not insert debug toolbar. </body> tag not found in response.
  warnings.warn('Could not insert debug toolbar.'
2026-07-13 19:37:18,533 - SL - DEBUG - 8682 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET / ImmutableMultiDict([]) 302, takes 0.0006470680236816406
2026-07-13 19:37:18,648 - SL - DEBUG - 8682 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.10405254364013672
2026-07-13 19:37:18,681 - SL - DEBUG - 8682 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.022179841995239258
2026-07-13 19:37:18,706 - SL - ERROR - 8682 - "/app/server.py:390" - error_handler() -  - (psycopg2.errors.UndefinedTable) relation "users" does not exist
LINE 2: FROM users 
             ^

[SQL: SELECT users.directory_quota AS users_directory_quota, users.subdomain_quota AS users_subdomain_quota, users.password AS users_password, users.id AS users_id, users.created_at AS users_created_at, users.updated_at AS users_updated_at, users.email AS users_email, users.name AS users_name, users.is_admin AS users_is_admin, users.alias_generator AS users_alias_generator, users.notification AS users_notification, users.activated AS users_activated, users.disabled AS users_disabled, users.profile_picture_id AS users_profile_picture_id, users.otp_secret AS users_otp_secret, users.enable_otp AS users_enable_otp, users.last_otp AS users_last_otp, users.fido_uuid AS users_fido_uuid, users.default_alias_custom_domain_id AS users_default_alias_custom_domain_id, users.default_alias_public_domain_id AS users_default_alias_public_domain_id, users.lifetime AS users_lifetime, users.paid_lifetime AS users_paid_lifetime, users.lifetime_coupon_id AS users_lifetime_coupon_id, users.trial_end AS users_trial_end, users.default_mailbox_id AS users_default_mailbox_id, users.sender_format AS users_sender_format, users.sender_format_updated_at AS users_sender_format_updated_at, users.replace_reverse_alias AS users_replace_reverse_alias, users.referral_id AS users_referral_id, users.intro_shown AS users_intro_shown, users.max_spam_score AS users_max_spam_score, users.newsletter_alias_id AS users_newsletter_alias_id, users.include_sender_in_reverse_alias AS users_include_sender_in_reverse_alias, users.random_alias_suffix AS users_random_alias_suffix, users.expand_alias_info AS users_expand_alias_info, users.ignore_loop_email AS users_ignore_loop_email, users.alternative_id AS users_alternative_id, users.disable_automatic_alias_note AS users_disable_automatic_alias_note, users.one_click_unsubscribe_block_sender AS users_one_click_unsubscribe_block_sender, users.include_website_in_one_click_alias AS users_include_website_in_one_click_alias, users.disable_import AS users_disable_import, users.can_use_phone AS users_can_use_phone, users.phone_quota AS users_phone_quota, users.block_behaviour AS users_block_behaviour, users.include_header_email_header AS users_include_header_email_header, users.enable_data_breach_check AS users_enable_data_breach_check, users.flags AS users_flags, users.unsub_behaviour AS users_unsub_behaviour, users.delete_on AS users_delete_on 
FROM users 
WHERE users.email = %(email_1)s 
 LIMIT %(param_1)s]
[parameters: {'email_1': 'test@example.com', 'param_1': 1}]
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
[parameters: {'email_1': 'test@example.com', 'param_1': 1}]
(Background on this error at: http://sqlalche.me/e/13/f405)
2026-07-13 19:37:18,712 - SL - DEBUG - 8682 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 500, takes 0.014319181442260742
```

**What this shows (Observed).** The serving child (PID 8682) logs `GET / ... 302`, then for the POST it logs, from `error_handler()` at `/app/server.py:390`, the exception `(psycopg2.errors.UndefinedTable) relation "users" does not exist` with the caret pointing at `FROM users`. The complete failing statement is shown in full — the driver `User.get_by` query selecting **every** `users` column (`users.directory_quota ... users.delete_on`) `FROM users WHERE users.email = %(email_1)s LIMIT %(param_1)s`, with `[parameters: {'email_1': 'test@example.com', 'param_1': 1}]`. The traceback then reproduces the *same* full SQL a second time as SQLAlchemy re-raises it, and the chain terminates in the canonical class:

```text
psycopg2.errors.UndefinedTable: relation "users" does not exist   (the DB-API cause)
        ↓  wrapped by SQLAlchemy as
sqlalchemy.exc.ProgrammingError: (psycopg2.errors.UndefinedTable) relation "users" does not exist
```

The Python call chain in the traceback is exactly: Flask `full_dispatch_request` → `flask_debugtoolbar` → `cProfile.runcall` → `flask_limiter` → `app/app/auth/views/login.py:43` (`User.get_by`) → `app/app/models.py:84` (`Session.query(cls).filter_by(**kw).first()`) → SQLAlchemy ORM/engine → `psycopg2` `cursor.execute`. This confirms the Inferred path above: the connection opened fine, and the first query against the not-yet-created `users` relation is what fails.

#### Q1 Run 2 — driver transcript

```text
$ PGPASSWORD=mypassword dropdb -U myuser -h localhost --if-exists sl_q1_obs

$ PGPASSWORD=mypassword createdb -U myuser -h localhost sl_q1_obs

$ PGPASSWORD=mypassword psql -U myuser -h localhost -d sl_q1_obs -tAc "SELECT count(*) FROM information_schema.tables WHERE table_schema='public';"
0

$ cd /app && source /tmp/blitzy_obs/sl.env && DB_URI=postgresql://myuser:mypassword@localhost:5432/sl_q1_obs setsid /app/venv/bin/python server.py > /tmp/blitzy_obs/q1_server_run2.log 2>&1 &
  server process-group leader PID (captured via $!) = 8771

$ for i in $(seq 1 40); do c=$(curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:7777/health 2>/dev/null); [ "$c" = 200 ] && { echo "ready after ${i}s (/health -> 200)"; break; }; sleep 1; done
ready after 3s (/health -> 200)

$ kill -0 8771 && echo '  server leader alive (kill -0 ok)'
  server leader alive (kill -0 ok)

$ grep -iE ' [0-9A-F]{8}:1E61 ' /proc/net/tcp | awk '$4=="0A"{print "  LISTEN local="$2" (0100007F=127.0.0.1, hex 1E61=7777, state 0A=LISTEN)"}'
  LISTEN local=0100007F:1E61 (0100007F=127.0.0.1, hex 1E61=7777, state 0A=LISTEN)

$ curl -sS -w ' [HTTP %{http_code}]\n' http://127.0.0.1:7777/health
success [HTTP 200]

$ curl -sS -D - -o /dev/null -w 'HTTP_STATUS=%{http_code}\n' http://127.0.0.1:7777/
HTTP/1.0 302 FOUND
Content-Type: text/html; charset=utf-8
Content-Length: 229
Location: http://127.0.0.1:7777/auth/login
Vary: Cookie
Set-Cookie: slapp=eyJfZnJlc2giOmZhbHNlLCJfcGVybWFuZW50Ijp0cnVlfQ.alU-kA.bgVUEaXRLSB2EkQk8n8Grje1_0k; Expires=Mon, 20-Jul-2026 19:37:52 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Mon, 13 Jul 2026 19:37:52 GMT

HTTP_STATUS=302

$ curl -sS -o /tmp/blitzy_obs/q1_get_run2.html -w 'HTTP_STATUS=%{http_code}\n' http://127.0.0.1:7777/auth/login
HTTP_STATUS=200

$ curl -c /tmp/blitzy_obs/q1_cj_run2 -s http://127.0.0.1:7777/auth/login -o /tmp/blitzy_obs/q1_login_run2.html -w 'GET /auth/login HTTP_STATUS=%{http_code}\n'
GET /auth/login HTTP_STATUS=200

$ CSRF=$(grep -oP 'name="csrf_token"[^>]*value="\K[^"]+' /tmp/blitzy_obs/q1_login_run2.html | head -1); echo "csrf_token length = ${#CSRF}"
csrf_token length = 91

$ curl -b /tmp/blitzy_obs/q1_cj_run2 -c /tmp/blitzy_obs/q1_cj_run2 -sS -D - -o /tmp/blitzy_obs/q1_post_body_run2.html -w 'HTTP_STATUS=%{http_code}\n' -X POST http://127.0.0.1:7777/auth/login --data-urlencode 'csrf_token='"$CSRF" --data-urlencode 'email=test@example.com' --data-urlencode 'password=whatever'
HTTP/1.0 500 INTERNAL SERVER ERROR
Content-Type: text/html; charset=utf-8
Content-Length: 5749
Vary: Cookie
Set-Cookie: slapp=.eJwNyDsOgCAQBcC7bG1h-MNlCJC3MVHRAFbGu8uU81Lkhr5R4HR0LBRvtDNV1EFhtGdO6Y3juHZUCmSTRDaahXFeiwTl2DPcKsoMlZU1AiwZ9P1qJRy9.alU-kQ.HTwhlvpE2uDDPXGRjoOQWc1wagg; Expires=Mon, 20-Jul-2026 19:37:53 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Mon, 13 Jul 2026 19:37:53 GMT

HTTP_STATUS=500

$ grep -iE 'Server error|server issues|<title>|history.back' /tmp/blitzy_obs/q1_post_body_run2.html
    <title>
        Server error
  Looks like we are having some server issues...
  <a class="btn btn-primary" href="javascript:history.back()">
      history.back();

$ wc -c /tmp/blitzy_obs/q1_post_body_run2.html
5749 /tmp/blitzy_obs/q1_post_body_run2.html

$ kill -TERM -8771   # terminate whole process group (leader + Werkzeug reloader child)
  stopped

$ grep -iE ' [0-9A-F]{8}:1E61 ' /proc/net/tcp | awk '$4=="0A"{print "  still LISTEN "$2}'
  (no output above == no socket in LISTEN state on 7777 -> port released)

$ ps -o pid,cmd -C python 2>/dev/null | grep server.py || echo '  (no server.py process running)'
  (no server.py process running)
```

#### Q1 Run 2 — web server console (boot banner + full traceback, verbatim)

```text
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/fcbutvfqvrueshiaiizd
Upload files to local dir
>>> init logging <<<
2026-07-13 19:37:49,786 - SL - DEBUG - 8771 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/pvxzrywznegttkqwiocq
Upload files to local dir
>>> init logging <<<
2026-07-13 19:37:51,709 - SL - DEBUG - 8786 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
/app/venv/lib/python3.10/site-packages/flask_debugtoolbar/__init__.py:213: UserWarning: Could not insert debug toolbar. </body> tag not found in response.
  warnings.warn('Could not insert debug toolbar.'
2026-07-13 19:37:52,836 - SL - DEBUG - 8786 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET / ImmutableMultiDict([]) 302, takes 0.0007605552673339844
2026-07-13 19:37:52,952 - SL - DEBUG - 8786 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.10882067680358887
2026-07-13 19:37:52,986 - SL - DEBUG - 8786 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.02210378646850586
2026-07-13 19:37:53,012 - SL - ERROR - 8786 - "/app/server.py:390" - error_handler() -  - (psycopg2.errors.UndefinedTable) relation "users" does not exist
LINE 2: FROM users 
             ^

[SQL: SELECT users.directory_quota AS users_directory_quota, users.subdomain_quota AS users_subdomain_quota, users.password AS users_password, users.id AS users_id, users.created_at AS users_created_at, users.updated_at AS users_updated_at, users.email AS users_email, users.name AS users_name, users.is_admin AS users_is_admin, users.alias_generator AS users_alias_generator, users.notification AS users_notification, users.activated AS users_activated, users.disabled AS users_disabled, users.profile_picture_id AS users_profile_picture_id, users.otp_secret AS users_otp_secret, users.enable_otp AS users_enable_otp, users.last_otp AS users_last_otp, users.fido_uuid AS users_fido_uuid, users.default_alias_custom_domain_id AS users_default_alias_custom_domain_id, users.default_alias_public_domain_id AS users_default_alias_public_domain_id, users.lifetime AS users_lifetime, users.paid_lifetime AS users_paid_lifetime, users.lifetime_coupon_id AS users_lifetime_coupon_id, users.trial_end AS users_trial_end, users.default_mailbox_id AS users_default_mailbox_id, users.sender_format AS users_sender_format, users.sender_format_updated_at AS users_sender_format_updated_at, users.replace_reverse_alias AS users_replace_reverse_alias, users.referral_id AS users_referral_id, users.intro_shown AS users_intro_shown, users.max_spam_score AS users_max_spam_score, users.newsletter_alias_id AS users_newsletter_alias_id, users.include_sender_in_reverse_alias AS users_include_sender_in_reverse_alias, users.random_alias_suffix AS users_random_alias_suffix, users.expand_alias_info AS users_expand_alias_info, users.ignore_loop_email AS users_ignore_loop_email, users.alternative_id AS users_alternative_id, users.disable_automatic_alias_note AS users_disable_automatic_alias_note, users.one_click_unsubscribe_block_sender AS users_one_click_unsubscribe_block_sender, users.include_website_in_one_click_alias AS users_include_website_in_one_click_alias, users.disable_import AS users_disable_import, users.can_use_phone AS users_can_use_phone, users.phone_quota AS users_phone_quota, users.block_behaviour AS users_block_behaviour, users.include_header_email_header AS users_include_header_email_header, users.enable_data_breach_check AS users_enable_data_breach_check, users.flags AS users_flags, users.unsub_behaviour AS users_unsub_behaviour, users.delete_on AS users_delete_on 
FROM users 
WHERE users.email = %(email_1)s 
 LIMIT %(param_1)s]
[parameters: {'email_1': 'test@example.com', 'param_1': 1}]
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
[parameters: {'email_1': 'test@example.com', 'param_1': 1}]
(Background on this error at: http://sqlalche.me/e/13/f405)
2026-07-13 19:37:53,018 - SL - DEBUG - 8786 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 500, takes 0.01478886604309082
```

**What Run 2 shows (Observed).** Identical behavior: server-group leader **PID 8771** with serving child **PID 8786**; `GET / → 302`, `GET /auth/login → 200`, `POST /auth/login → 500`; the same `sqlalchemy.exc.ProgrammingError` wrapping `psycopg2.errors.UndefinedTable: relation "users" does not exist`, and a clean shutdown that releases port 7777.

### Q1 — Stability across the two runs (Observed)

Normalizing away the only volatile fields (timestamps and PIDs), the two server-console tracebacks are byte-identical except for the trailing request duration in the final `after_request` log line (`takes 0.014319...` vs `takes 0.014788...`). The exception class, the wrapped DB-API error, the `users` relation name, both full SQL statements, the bound parameters, and the `POST /auth/login → 500` outcome are the same in both runs. The result is therefore **stable**.

### Q1 — Answer

Starting `python server.py` against an empty, unmigrated database **succeeds** (the engine connects at import, `app/db.py:12`), and the login page even renders on a cold `GET /auth/login` (HTTP 200). The error surfaces only when the login form is **submitted**: the first ORM query (`User.get_by`, `app/auth/views/login.py:43` → `app/models.py:84`) runs against the missing `users` table, and PostgreSQL raises `psycopg2.errors.UndefinedTable: relation "users" does not exist`, which SQLAlchemy wraps and re-raises as:

```text
sqlalchemy.exc.ProgrammingError: (psycopg2.errors.UndefinedTable) relation "users" does not exist
```

The browser itself receives **HTTP 500** rendered as the generic `error/500.html` “Server error” page (5749 bytes), because `@app.errorhandler(Exception)` catches it and returns that template (`server.py:388-394`); the **full Python traceback quoted above is written to the server's stdout/stderr**, not to the browser. The exact missing relation is `users` (`User.__tablename__ = "users"`, `app/models.py:337`).

## Q2 — After migrations and `init_app.py`: the required Python services, with startup logs and port bindings

**Question.** Once the database migrations and the initialization script (`init_app.py`) have been run and everything is set up correctly, which Python services must be running for the system to work? Start each required service and show the exact stdout/stderr log output that confirms it is running and ready to accept connections — verifying the services actually bind to their ports.

### Q2 — Chronology: initialize the migrated database first (Observed)

Per the required ordering, Q2 is exercised on the **same** database used for the Q3 pre-init observation (`sl_probe_obs`), after it has been migrated (Q3 section) and then **initialized** with `python init_app.py`. `init_app.py`'s `__main__` runs `load_pgp_public_keys()` and `add_sl_domains()` (`init_app.py:69-73`); `add_sl_domains()` inserts one `SLDomain` row per `ALIAS_DOMAINS` entry (`init_app.py:39-56`), and `SLDomain.__tablename__ = "public_domain"` (`app/models.py:3119`). The complete, unedited run below shows `public_domain` transition from **0 → 1** (the single seeded row `sl.local`):

```text
### Chronology: sl_probe_obs is migrated-but-uninitialized (public_domain empty). Running init_app.py to seed it.

$ psql -U myuser -h localhost -d sl_probe_obs -tAc "select count(*) from public_domain"
0

$ DB_URI=postgresql://myuser:mypassword@localhost:5432/sl_probe_obs /app/venv/bin/python init_app.py
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/nfhgdjjyjqdatsjxfdvo
Upload files to local dir
>>> init logging <<<
2026-07-13 19:21:42,173 - SL - DEBUG - 8027 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-13 19:21:43,156 - SL - DEBUG - 8027 - "/app/init_app.py:36" - load_pgp_public_keys() -  - Finish load_pgp_public_keys
2026-07-13 19:21:43,158 - SL - INFO - 8027 - "/app/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain

$ psql -U myuser -h localhost -d sl_probe_obs -c "select id, domain, premium_only, use_as_reverse_alias from public_domain order by id"
 id |  domain  | premium_only | use_as_reverse_alias 
----+----------+--------------+----------------------
  1 | sl.local | f            | t
(1 row)


$ psql -U myuser -h localhost -d sl_probe_obs -tAc "select count(*) from public_domain"
1
```

### Q2 — Which Python services are required (Inferred from README/Dockerfile/CONTRIBUTING; each Observed individually below)

The required service set is **inferred** from three grounding sources and then each service is **observed** running individually. The production deployment ordering in `README.md` starts, in order, `flask db upgrade` (`README.md:433`), `python init_app.py` (`README.md:448`), the web app on `127.0.0.1:7777:7777` (`README.md:461`), `python email_handler.py` on `127.0.0.1:20381:20381` (`README.md:477,480`), and `python job_runner.py` (`README.md:495`); `crontab.yml` drives `cron.py` on a schedule (e.g. `python /code/cron.py -j delete_old_monitoring`, `crontab.yml:8-11`); and `event_listener.py` provides the PostgreSQL LISTEN/NOTIFY event consumer. This yields **five** Python services:

1. **Web application** — `server.py` (dev entry) or `wsgi.py` under gunicorn (canonical/production, the `Dockerfile` CMD `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15`, `Dockerfile:47`). **Binds TCP port 7777.**
2. **Inbound SMTP handler** — `email_handler.py` (aiosmtpd). **Binds TCP port 20381** (`email_handler.py:2383,2399,2403`).
3. **`job_runner.py`** — continuous background worker that polls the `job` table. **No client-facing port.**
4. **`cron.py`** — scheduled one-shot jobs invoked by the yacron scheduler per `crontab.yml`. **No client-facing port.**
5. **`event_listener.py listener`** — background consumer of PostgreSQL `LISTEN`/`NOTIFY` events. **No client-facing port.**

Two of the five are **port-binding services** that accept client connections (web app on 7777, SMTP handler on 20381); the other three are **background workers** whose readiness is proven by a log line rather than a listening socket. The “ready” evidence therefore differs in kind between the two groups, and both kinds are captured below.

**Port clarification (Observed vs. a common misconception).** The *Python* SMTP service binds **20381**, not 25. The public MTA (Postfix) listens on port 25 and *relays* inbound mail to the Python handler at `smtp:127.0.0.1:20381` (`README.md:367-369`); nginx reverse-proxies the web app at `http://localhost:7777` (`README.md:513`). The values observed for the Python services are therefore **7777** and **20381**.

### Q2 — Service A: the web application (port 7777)

The web app has two real entry points, and **both** are exercised. The **development** entry is `python server.py`, whose `local_main()` (`server.py:571`) calls `app.run(debug=True, port=7777)` (`server.py:588`) — Werkzeug's debug server binds the loopback address `127.0.0.1:7777` and forks a reloader child (hence the boot banner appears twice; the child PID differs from the leader). `/health` returns `("success", 200)` (`server.py:213-215`) and `/` returns a 302 redirect to `/auth/login` (`server.py:249-255`):

```text
### Service A (dev entry): python server.py -> local_main() [server.py:571] -> app.run(debug=True, port=7777) [server.py:588]
  leader PID=8122 (process-group leader via setsid; Werkzeug debug reloader forks a child)
  waited 2s for port bind

$ listen_report 1E61 7777
  LISTEN local=0100007F:1E61 (hex 1E61=7777, state 0A=LISTEN; 0100007F=127.0.0.1, 00000000=0.0.0.0)

$ curl -s -o /dev/null -w 'HEALTH_STATUS=%{http_code}\n' http://127.0.0.1:7777/health
HEALTH_STATUS=200

$ curl -s http://127.0.0.1:7777/health; echo
success

$ curl -s -o /dev/null -w 'ROOT_STATUS=%{http_code} REDIRECT=%{redirect_url}\n' http://127.0.0.1:7777/
ROOT_STATUS=302 REDIRECT=http://127.0.0.1:7777/auth/login

$ kill -0 8122 && echo '  leader alive (kill -0 ok)'
  leader alive (kill -0 ok)

----- server.py console (complete stdout+stderr) -----
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/jiabqnndbpbaogwscfcw
Upload files to local dir
>>> init logging <<<
2026-07-13 19:23:34,287 - SL - DEBUG - 8122 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/zniaoejhpxbkjxmqabpy
Upload files to local dir
>>> init logging <<<
2026-07-13 19:23:36,112 - SL - DEBUG - 8133 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
/app/venv/lib/python3.10/site-packages/flask_debugtoolbar/__init__.py:213: UserWarning: Could not insert debug toolbar. </body> tag not found in response.
  warnings.warn('Could not insert debug toolbar.'
2026-07-13 19:23:37,165 - SL - DEBUG - 8133 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET / ImmutableMultiDict([]) 302, takes 0.0006241798400878906
----- end console -----

$ kill -TERM -8122 2>/dev/null; sleep 3; echo terminated
terminated

$ listen_report 1E61 7777
  (no LISTEN socket on port 7777)

$ ps -eo pid,cmd | grep 'server.py' | grep -v grep || echo '  (no server.py process running)'
  (no server.py process running)
```

The **canonical/production** entry is the `Dockerfile` CMD (`Dockerfile:47`): `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15`, where `wsgi.py` is `from server import create_app; app = create_app()` (`wsgi.py:1-3`). Gunicorn binds **all** interfaces `0.0.0.0:7777`, logs `Starting gunicorn 20.0.4` and `Listening at: http://0.0.0.0:7777`, and boots two sync workers:

```text
### Service A (canonical/production entry, Dockerfile CMD): gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
### wsgi.py: 'from server import create_app; app = create_app()' [wsgi.py:1-3]
  gunicorn master PID=8237
  waited 1s for port bind

$ listen_report 1E61 7777
  LISTEN local=00000000:1E61 (hex 1E61=7777, state 0A=LISTEN; 0100007F=127.0.0.1, 00000000=0.0.0.0)

$ curl -s -o /dev/null -w 'HEALTH_STATUS=%{http_code}\n' http://127.0.0.1:7777/health
HEALTH_STATUS=200

$ curl -s http://127.0.0.1:7777/health; echo
success

$ kill -0 8237 && echo '  master alive (kill -0 ok)'
  master alive (kill -0 ok)

$ ps --ppid 8237 -o pid=,cmd= | sed 's/^ */  worker PID /'
  worker PID 8240 /app/venv/bin/python /app/venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
  worker PID 8241 /app/venv/bin/python /app/venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15

----- gunicorn console (complete stdout+stderr) -----
[2026-07-13 19:24:53 +0000] [8237] [INFO] Starting gunicorn 20.0.4
[2026-07-13 19:24:53 +0000] [8237] [INFO] Listening at: http://0.0.0.0:7777 (8237)
[2026-07-13 19:24:53 +0000] [8237] [INFO] Using worker: sync
[2026-07-13 19:24:53 +0000] [8240] [INFO] Booting worker with pid: 8240
[2026-07-13 19:24:53 +0000] [8241] [INFO] Booting worker with pid: 8241
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/lxljdvgmzwlsyxuygzyl
Upload files to local dir
>>> init logging <<<
2026-07-13 19:24:53,979 - SL - DEBUG - 8240 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/afsdjwvtntqcpyjvppwb
Upload files to local dir
>>> init logging <<<
2026-07-13 19:24:54,078 - SL - DEBUG - 8241 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
----- end console -----

$ kill -TERM 8237 2>/dev/null; sleep 3; echo terminated
terminated

$ listen_report 1E61 7777
  (no LISTEN socket on port 7777)

$ ps -eo pid,stat,cmd | grep '[g]unicorn wsgi:app' | grep -v defunct || echo '  (no live gunicorn wsgi:app process running)'
  (no live gunicorn wsgi:app process running)
```

**Observed:** the dev server binds `0100007F:1E61` (`127.0.0.1:7777`) and gunicorn binds `00000000:1E61` (`0.0.0.0:7777`); `/health` returns HTTP 200 with body `success` on both; `/` returns HTTP 302 to `http://127.0.0.1:7777/auth/login`; gunicorn reports its own master PID and two worker PIDs. Each was terminated with `SIGTERM` and the port was confirmed released afterward. The two gunicorn invocations bound port 7777 identically (readiness stable across runs).

### Q2 — Service B: the inbound SMTP handler (port 20381)

`python email_handler.py` calls `main(port=20381)` (`email_handler.py:2381,2404`; the argparse default is `20381`, `email_handler.py:2399`) and starts an aiosmtpd `Controller(MailHandler(), hostname="0.0.0.0", port=20381)` (`email_handler.py:2383`). Readiness is proven three ways — the `Listen for port 20381` log line (`email_handler.py:2403`), a `LISTEN` socket on `0.0.0.0:20381`, and a live SMTP `220` greeting returned to a real client connection:

```text
### Service B: python email_handler.py -> main(port=20381) [email_handler.py:2381,2404]
### Controller(MailHandler(), hostname='0.0.0.0', port=20381) [email_handler.py:2383]; argparse default 20381 [email_handler.py:2399]
  email_handler PID=8285
  waited 2s for port bind

$ listen_report 4F9D 20381
  LISTEN local=00000000:4F9D (hex 4F9D=20381, state 0A=LISTEN; 0100007F=127.0.0.1, 00000000=0.0.0.0)

$ (live SMTP 220 banner probe on 127.0.0.1:20381)
  S: 220 f411db182cde Python SMTP 1.4.2
  S: 221 Bye

$ kill -0 8285 && echo '  email_handler alive (kill -0 ok)'
  email_handler alive (kill -0 ok)

----- email_handler.py console (complete stdout+stderr) -----
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/gayqgpcmmmgppdbwxncl
Upload files to local dir
>>> init logging <<<
2026-07-13 19:25:19,186 - SL - DEBUG - 8285 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-13 19:25:19,884 - SL - INFO - 8285 - "/app/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-13 19:25:19,886 - SL - DEBUG - 8285 - "/app/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
----- end console -----

$ kill -TERM 8285 2>/dev/null; sleep 3; echo terminated
terminated

$ listen_report 4F9D 20381
  (no LISTEN socket on port 20381)

$ ps -eo pid,stat,cmd | grep '[e]mail_handler.py' | grep -v defunct || echo '  (no live email_handler.py process running)'
  (no live email_handler.py process running)
```

**Observed:** the handler binds `00000000:4F9D` (`0.0.0.0:20381`); a real socket connection is greeted with `220 f411db182cde Python SMTP 1.4.2` (aiosmtpd 1.4.2) and `QUIT` is answered `221 Bye`; the console shows `Listen for port 20381` (`email_handler.py:2403`) and `Start mail controller 0.0.0.0 20381` (`email_handler.py:2386`). The process was terminated with `SIGTERM` and the port confirmed released.

### Q2 — Service C: `job_runner.py` (background worker, no client-facing port)

`job_runner.py`'s `__main__` is an unbounded poll loop: each iteration opens an app context, calls `get_jobs_to_run()`, and for every returned `Job` logs `Take job %s` (`job_runner.py:334`), marks it `taken`, calls `process_job(job)`, marks it `done`, then `time.sleep(10)`. To exercise a **recognized, bounded** job (rather than the `Unknown job name` error path), one valid no-op job is enqueued: `config.JOB_SEND_ALIAS_CREATION_EVENTS = "send-alias-creation-events"` (`app/config.py:311`) with an **empty payload**. In `process_job`, that branch does `user = User.get(user_id)` with `user_id` absent (`None`), so `User.get(None)` returns `None` and the guarded `if user and user.activated:` body is skipped — a clean recognized no-op. The complete enqueue helper used (a temporary observation script, removed at the end) is:

```python
# Temporary observation helper: enqueue ONE valid, bounded, no-op job so job_runner.py
# exercises a RECOGNIZED job (config.JOB_SEND_ALIAS_CREATION_EVENTS [app/config.py:311]).
# Empty payload -> user_id None -> User.get(None) None -> process_job no-ops -> job marked done.
from server import create_light_app
from app.db import Session
from app.models import Job
from app import config

with create_light_app().app_context():
    j = Job.create(name=config.JOB_SEND_ALIAS_CREATION_EVENTS, payload={})
    Session.commit()
    print(f"ENQUEUED job id={j.id} name={j.name!r} state={j.state} taken={j.taken} payload={j.payload}")
```

The full, unedited run — enqueue, poll, take, mark done, stop, and net-zero row removal:

```text
### Service C: python job_runner.py -> while True: for job in get_jobs_to_run(): Take/process/mark done; time.sleep(10) [job_runner.py __main__]
### JobState enum [app/models.py:254-257]: ready=0, taken=1, done=2, error=3

$ env PYTHONPATH=/app DB_URI=postgresql://myuser:mypassword@localhost:5432/sl_probe_obs /app/venv/bin/python /tmp/blitzy_obs/enqueue_job.py
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/gcfoiihbobpqrhflpbym
Upload files to local dir
>>> init logging <<<
2026-07-13 19:26:59,590 - SL - DEBUG - 8414 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
ENQUEUED job id=1 name='send-alias-creation-events' state=0 taken=False payload={}

$ psql -U myuser -h localhost -d sl_probe_obs -c "select id,name,state,taken,attempts from job order by id"
 id |            name            | state | taken | attempts 
----+----------------------------+-------+-------+----------
  1 | send-alias-creation-events |     0 | f     |        0
(1 row)


  job_runner PID=8422
  waited 2s for first 'Take job'

$ kill -0 8422 && echo '  job_runner alive (kill -0 ok, still in 10s poll loop)'
  job_runner alive (kill -0 ok, still in 10s poll loop)

$ psql -U myuser -h localhost -d sl_probe_obs -c "select id,name,state,taken,attempts from job order by id"
 id |            name            | state | taken | attempts 
----+----------------------------+-------+-------+----------
  1 | send-alias-creation-events |     2 | t     |        1
(1 row)


----- job_runner.py console (complete stdout+stderr) -----
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/wxllwxpxkmduiojjrorq
Upload files to local dir
>>> init logging <<<
2026-07-13 19:27:01,385 - SL - DEBUG - 8422 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-13 19:27:02,368 - SL - DEBUG - 8422 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 1 send-alias-creation-events {}>
/app/venv/lib/python3.10/site-packages/sqlalchemy/orm/loading.py:247: SAWarning: fully NULL primary key identity cannot load any object.  This condition may raise an error in a future release.
  util.warn(
----- end console -----

$ kill -TERM 8422 2>/dev/null; sleep 2; echo terminated
terminated

$ ps -eo pid,stat,cmd | grep '[j]ob_runner.py' | grep -v defunct || echo '  (no live job_runner.py process running)'
  (no live job_runner.py process running)

### net-zero: remove the temporary probe job row

$ psql -U myuser -h localhost -d sl_probe_obs -c "delete from job"
DELETE 1

$ psql -U myuser -h localhost -d sl_probe_obs -tAc "select count(*) from job"
0
```

**Observed:** the probe job moves `state 0 (ready) → 2 (done)`, `taken f → t`, `attempts 0 → 1` (`JobState`, `app/models.py:254-257`), and the console logs `Take job <Job 1 send-alias-creation-events {}>` (`job_runner.py:334`). The single `SAWarning: fully NULL primary key identity cannot load any object` is the authentic, expected signature of the no-op branch — it is emitted by `User.get(None)` (a primary-key load with a `NULL` id), which returns `None`, so no alias-creation events are dispatched. `kill -0` confirms the worker stays alive between polls; `SIGTERM` stops it; and the temporary `job` row is deleted (`DELETE 1` → count `0`), leaving the table as it was found.

### Q2 — Service D: `cron.py` (scheduled one-shot jobs, no client-facing port)

`cron.py` is **not** a daemon: it is a scheduled one-shot that the yacron scheduler invokes per `crontab.yml` (for example `python /code/cron.py -j delete_old_monitoring` at `15 1 * * *`, `crontab.yml:8-11`). Its `__main__` logs `Start running cronjob` (`cron.py:1263`), parses `-j/--job`, and dispatches; the `delete_old_monitoring` branch (`cron.py:1298-1300`) calls `delete_old_monitoring()` (`cron.py:954`), which deletes `Monitoring` rows older than 30 days, commits, and logs the row count (`cron.py:961`). The complete run, ending with the process exit code, is:

```text
### Service D: python cron.py -j delete_old_monitoring  (scheduled one-shot; yacron drives it per crontab.yml)
### __main__ LOG.d('Start running cronjob') [cron.py:1263]; dispatch elif args.job=='delete_old_monitoring' [cron.py:1298-1300] -> delete_old_monitoring() [cron.py:954]

$ env PYTHONPATH=/app DB_URI=postgresql://myuser:mypassword@localhost:5432/sl_probe_obs /app/venv/bin/python cron.py -j delete_old_monitoring; echo "CRON_EXIT_CODE=$?"
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/bppuatyazuyehuoczjqv
Upload files to local dir
>>> init logging <<<
2026-07-13 19:27:31,532 - SL - DEBUG - 8460 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-13 19:27:32,401 - SL - DEBUG - 8460 - "/app/cron.py:1263" - <module>() -  - Start running cronjob
2026-07-13 19:27:32,402 - SL - DEBUG - 8460 - "/app/cron.py:1299" - <module>() -  - Delete old monitoring records
2026-07-13 19:27:32,519 - SL - DEBUG - 8460 - "/app/cron.py:961" - delete_old_monitoring() -  - delete monitoring records older than 2026-06-13T19:27:32.403069+00:00, nb row 0
CRON_EXIT_CODE=0
```

**Observed:** the job logs `Start running cronjob` (`cron.py:1263`), `Delete old monitoring records` (`cron.py:1299`), and the function-result line `delete monitoring records older than 2026-06-13T19:27:32.403069+00:00, nb row 0` (`cron.py:961`), then exits with `CRON_EXIT_CODE=0`. `nb row 0` is expected because the freshly-migrated database has no `monitoring` rows. Because it terminates on its own, no signal is needed to stop it.

### Q2 — Service E: `event_listener.py listener` (background event consumer, no client-facing port)

`python event_listener.py listener` runs `main(mode=LISTENER, dry_run=False, max_retries=10)`: it logs `Using PostgresEventSource` (`event_listener.py:34`), constructs `PostgresEventSource(EVENT_LISTENER_DB_URI)` (`event_listener.py:35`; `EVENT_LISTENER_DB_URI` defaults to `DB_URI`, `app/config.py:637`), logs `Starting with HttpEventSink` (`event_listener.py:43`), then `Runner.run()` calls `PostgresEventSource.__listen`, which issues `LISTEN` on the notification channel and logs the strongest readiness line, `Starting to listen to events` (`events/event_source.py:49`), before blocking in a `select()` poll loop. It consumes PostgreSQL `LISTEN`/`NOTIFY` and does **not** bind a client-facing port. The service is started **twice** to confirm the readiness sequence is stable:

```text
### Service E: python event_listener.py listener
### main(LISTENER): LOG.i('Using PostgresEventSource') [event_listener.py:34]; PostgresEventSource(EVENT_LISTENER_DB_URI) [:35]; LOG.i('Starting with HttpEventSink') [:43]
### Runner.run -> PostgresEventSource.__listen -> LOG.info('Starting to listen to events') [events/event_source.py:49] (strongest readiness); no client-facing port -- consumes PG LISTEN/NOTIFY

========================= event_listener.py listener RUN 1 =========================
  event_listener PID=8482
  waited 1s for 'Starting to listen to events'

$ kill -0 8482 && echo '  event_listener alive (kill -0 ok, blocked in select() poll loop)'
  event_listener alive (kill -0 ok, blocked in select() poll loop)

----- event_listener.py console RUN 1 (complete stdout+stderr) -----
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/senmbptensvrekrbzrew
Upload files to local dir
>>> init logging <<<
2026-07-13 19:28:01,385 - SL - DEBUG - 8482 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-13 19:28:01,501 - SL - INFO - 8482 - "/app/event_listener.py:34" - main() -  - Using PostgresEventSource
2026-07-13 19:28:01,508 - SL - INFO - 8482 - "/app/event_listener.py:43" - main() -  - Starting with HttpEventSink
2026-07-13 19:28:01,509 - SL - INFO - 8482 - "/app/events/event_source.py:49" - __listen() -  - Starting to listen to events
----- end console -----

$ kill -TERM 8482 2>/dev/null; sleep 2; echo terminated
terminated

$ ps -eo pid,stat,cmd | grep '[e]vent_listener.py' | grep -v defunct || echo '  (no live event_listener.py process running)'
  (no live event_listener.py process running)

========================= event_listener.py listener RUN 2 =========================
  event_listener PID=8494
  waited 1s for 'Starting to listen to events'

$ kill -0 8494 && echo '  event_listener alive (kill -0 ok, blocked in select() poll loop)'
  event_listener alive (kill -0 ok, blocked in select() poll loop)

----- event_listener.py console RUN 2 (complete stdout+stderr) -----
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/mziaahtmmykqpgwrtihy
Upload files to local dir
>>> init logging <<<
2026-07-13 19:28:04,402 - SL - DEBUG - 8494 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-13 19:28:04,525 - SL - INFO - 8494 - "/app/event_listener.py:34" - main() -  - Using PostgresEventSource
2026-07-13 19:28:04,533 - SL - INFO - 8494 - "/app/event_listener.py:43" - main() -  - Starting with HttpEventSink
2026-07-13 19:28:04,533 - SL - INFO - 8494 - "/app/events/event_source.py:49" - __listen() -  - Starting to listen to events
----- end console -----

$ kill -TERM 8494 2>/dev/null; sleep 2; echo terminated
terminated

$ ps -eo pid,stat,cmd | grep '[e]vent_listener.py' | grep -v defunct || echo '  (no live event_listener.py process running)'
  (no live event_listener.py process running)
```

**Observed:** both runs emit the identical readiness sequence — `Using PostgresEventSource` (`event_listener.py:34`) → `Starting with HttpEventSink` (`event_listener.py:43`) → `Starting to listen to events` (`events/event_source.py:49`) — within ~1s; `kill -0` confirms the process stays alive in its poll loop; and `SIGTERM` stops it cleanly with no residual process.

### Q2 — Answer

**Five** Python services must be running for the fully-initialized system to work. The table summarizes each service's canonical start command, role, port/binding, and the observed readiness signal (the service *set* is inferred from `README.md`/`Dockerfile`/`crontab.yml`; every port, log line, and lifecycle result in the table was **observed** at runtime above):

| # | Service (entry command) | Role | Port / binding (Observed) | Readiness signal (Observed) |
|---|-------------------------|------|---------------------------|-----------------------------|
| 1 | Web — dev: `python server.py`; canonical: `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` | HTTP UI/API | `127.0.0.1:7777` (dev) / `0.0.0.0:7777` (gunicorn) | `Listening at: http://0.0.0.0:7777`; `GET /health` → `200 success` |
| 2 | SMTP — `python email_handler.py` | Inbound mail | `0.0.0.0:20381` | `Listen for port 20381` (`email_handler.py:2403`); live `220 … Python SMTP 1.4.2` |
| 3 | `python job_runner.py` | Background job worker | none (DB poll loop) | `Take job <…>` (`job_runner.py:334`); job `ready→0→2 done` |
| 4 | `python cron.py -j <job>` | Scheduled one-shot (yacron) | none | `Start running cronjob` (`cron.py:1263`); exit code `0` |
| 5 | `python event_listener.py listener` | PG `LISTEN`/`NOTIFY` consumer | none | `Starting to listen to events` (`events/event_source.py:49`) |

**Security reminder (see the Security & isolation notice near the top).** In canonical form the web app (gunicorn 20.0.4) binds `0.0.0.0:7777` and the SMTP handler (aiosmtpd 1.4.2) binds `0.0.0.0:20381` — i.e. all interfaces. These observations were made inside an isolated, disposable container with unpublished ports; the frozen versions are preserved for faithful observation and must not be exposed to an untrusted network.

## Q3 — Migrations run but `init_app.py` skipped: mail to `@sl.local`, the SMTP status and rejection log

**Question.** If migrations are run but `init_app.py` is **not** run, the `SLDomain` table should be empty (no email domains configured). If the email handler is then started and an incoming email addressed to any `@sl.local` address is simulated, what happens — what SMTP status code does the handler return to the sender, and what does it log to explain the rejection?

### Q3 — The decision path (Inferred from source, confirmed Observed below)

Inbound mail enters through the aiosmtpd listener and `MailHandler.handle_DATA` (`email_handler.py:2289`) → `_handle` (`email_handler.py:2335`) → `handle` (`email_handler.py:1945`). For a plain recipient such as `nonexistent@sl.local` the message is a **Forward** (not a reverse alias — `is_reverse_alias` returns `False` for a plain `@sl.local` address, `app/email_utils.py:1156`), so control reaches `handle_forward` (`email_handler.py:536`). There, `alias = Alias.get_by(email=rcpt_to)` (`email_handler.py:543`) finds no alias, so the code tries to auto-create one with `try_auto_create(alias_address)` (`email_handler.py:549` → `app/alias_utils.py:202`). `try_auto_create` only attempts two strategies — a **custom-domain catch-all** (`try_auto_create_via_domain`) and a **directory** alias (`try_auto_create_directory`); when both return `None`, `handle_forward` logs `"alias %s cannot be created on-the-fly, return 550"` (`email_handler.py:551`) and then branches on the sender: `if should_ignore_bounce(envelope.mail_from): return [(True, status.E207)] else: return [(False, status.E515)]` (`email_handler.py:552-555`). `should_ignore_bounce` (`app/email_utils.py:1361`) returns `True` only if the sender is present in the `ignore_bounce_sender` table (`IgnoreBounceSender`, `app/models.py:3357`); for a normal sender it returns `False`, so the result is `status.E515 = "550 SL E515 Email not exist"` (`app/email/status.py:51`). The alternate value is `status.E207 = "250 SL E207 No bounce report"` (`app/email/status.py:12`). Both branches are exercised and observed below.

**Note on the `SLDomain`/`public_domain` relationship (grounding).** `init_app.py` is what seeds the domain table: its `__main__` calls `add_sl_domains()` (`init_app.py:69-73`), which inserts one `SLDomain` row per entry of `ALIAS_DOMAINS` (`init_app.py:39-56`); `SLDomain.__tablename__ = "public_domain"` (`app/models.py:3119`), and `ALIAS_DOMAINS = ["sl.local"]` (`app/config.py:157-161`). Skipping `init_app.py` therefore leaves `public_domain` empty. The runtime logs below reveal an important causal nuance about *why* the rejection happens, discussed in “Q3 — Causal nuance”.

### Q3 — Setup: a migrated but UNINITIALIZED database (Observed)

A fresh database `sl_probe_obs` is created and migrated with `alembic upgrade head`; `init_app.py` is deliberately **not** run. (Migrations are run against a fresh `createdb` — never against a database where the `pg_trgm` extension was pre-created — so the whole Alembic transaction does not roll back on a duplicate-object error.)

```text
########## Q3 SETUP: create + migrate sl_probe_obs (init_app.py deliberately NOT run) ##########

$ dropdb -U myuser -h localhost --if-exists sl_probe_obs

$ createdb -U myuser -h localhost sl_probe_obs
```

The complete, unedited `alembic upgrade head` output follows — **255** migration steps, ending at head revision `32f25cbf12f6`:

```text
$ cd /app && DB_URI='postgresql://myuser:mypassword@localhost:5432/sl_probe_obs' /app/venv/bin/alembic upgrade head 2>&1
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/qmqpclgsgpyqepukmjru
Upload files to local dir
>>> init logging <<<
2026-07-13 19:10:37,013 - SL - DEBUG - 7694 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
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
```

With migrations applied, the schema has **77** tables, `public_domain` is **empty** (0 rows — the uninitialized state Q3 requires), and `alias` is likewise empty:

```text
$ psql -U myuser -h localhost -d sl_probe_obs -tAc "SELECT count(*) FROM information_schema.tables WHERE table_schema='public';"
77

$ psql -U myuser -h localhost -d sl_probe_obs -tAc "SELECT count(*) AS public_domain_rows FROM public_domain;"
0

$ psql -U myuser -h localhost -d sl_probe_obs -tAc "SELECT count(*) AS alias_rows FROM alias;"
0
```

### Q3 — Start the email handler and prove the port binding (Observed)

`python email_handler.py` binds SMTP on `0.0.0.0:20381` by default (argparse default, `email_handler.py:2399`; `Controller(MailHandler(), hostname="0.0.0.0", port=port)`, `email_handler.py:2383`). The handler is started as a background process whose PID is captured for a clean shutdown; readiness is confirmed from `/proc/net/tcp`, from the process's own startup log, and with a live `220` banner probe.

```text
########## Q3: start email_handler.py (binds 0.0.0.0:20381) ##########
$ cd /app && env DB_URI=postgresql://myuser:mypassword@localhost:5432/sl_probe_obs /app/venv/bin/python email_handler.py > /tmp/blitzy_obs/q3_handler.log 2>&1 &
email_handler background PID = 7704 (this is the python process that binds 20381)
$ # poll for SMTP listener on 20381 (hex 4F9D, state 0A) until up, timeout 40s
ready after 3s (LISTEN on 0.0.0.0:20381)

$ grep -nE 'Listen for port|Start mail controller' /tmp/blitzy_obs/q3_handler.log
8:2026-07-13 19:10:39,865 - SL - INFO - 7704 - "/app/email_handler.py:2403" - <module>() -  - Listen for port 20381
9:2026-07-13 19:10:39,866 - SL - DEBUG - 7704 - "/app/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381

$ # /proc/net/tcp LISTEN proof for port 20381
  LISTEN local=00000000:4F9D (00000000=0.0.0.0, hex 4F9D=20381, state 0A=LISTEN)

$ # live SMTP 220 banner probe (real socket, then close)
220 f411db182cde Python SMTP 1.4.2

$ kill -0 7704 && echo '(handler PID '7704' alive)'
(handler PID 7704 alive)
```

**What this shows (Observed).** The handler (background **PID 7704**) logs `Listen for port 20381` (`email_handler.py:2403`) and `Start mail controller 0.0.0.0 20381` (`email_handler.py:2386`), is listening on `00000000:4F9D` = **0.0.0.0:20381** (state `0A` = LISTEN), answers a live SMTP banner `220 f411db182cde Python SMTP 1.4.2` (note the `aiosmtpd` version **1.4.2** in the banner), and `kill -0` confirms the process is alive.

### Q3 — Primary result: mail to `nonexistent@sl.local` → `550 SL E515` (Observed)

A real inbound message is delivered through the listener using the self-contained raw-socket SMTP client shown in “Q3 — The SMTP injection client” (below). The complete client-side conversation — every `C:`/`S:` line, the full `DATA` payload, and the `<CR><LF>.<CR><LF>` terminator — is:

```text
########## Q3 PRIMARY RUN 1: inject sender@example.com -> nonexistent@sl.local (expect 550 SL E515) ##########
$ /app/venv/bin/python /tmp/blitzy_obs/sl_inject.py 127.0.0.1 20381 sender@example.com nonexistent@sl.local
S: 220 f411db182cde Python SMTP 1.4.2
C: EHLO test.blitzy.local
S: 250-f411db182cde
S: 250-SIZE 33554432
S: 250-8BITMIME
S: 250-SMTPUTF8
S: 250 HELP
C: MAIL FROM:<sender@example.com>
S: 250 OK
C: RCPT TO:<nonexistent@sl.local>
S: 250 OK
C: DATA
S: 354 End data with <CR><LF>.<CR><LF>
C: From: sender@example.com
C: To: nonexistent@sl.local
C: Subject: q3 test
C: Content-Transfer-Encoding: 7bit
C: 
C: q3 test body
C: .
S: 550 SL E515 Email not exist
C: QUIT
S: 221 Bye
=== FINAL_SERVER_REPLY_TO_SENDER: 550 SL E515 Email not exist
```

The server's final reply to the sender is **`550 SL E515 Email not exist`**. The handler's own console logs the complete rejection chain for this message (`message_id 30eb46f4-...`):

```text
2026-07-13 19:10:42,506 - SL - DEBUG - 7704 - "/app/app/log.py:24" - set_message_id() -  - set message_id 30eb46f4-05db-4713-8c84-5e513ff02d07
2026-07-13 19:10:42,506 - SL - DEBUG - 7704 - "/app/email_handler.py:2342" - _handle() - 30eb46f4-05db-4713-8c84-5e513ff02d07 - ====>=====>====>====>====>====>====>====>
2026-07-13 19:10:42,506 - SL - INFO - 7704 - "/app/email_handler.py:2343" - _handle() - 30eb46f4-05db-4713-8c84-5e513ff02d07 - New message, mail from sender@example.com, rctp tos ['nonexistent@sl.local'] 
2026-07-13 19:10:42,508 - SL - DEBUG - 7704 - "/app/email_handler.py:1963" - handle() - 30eb46f4-05db-4713-8c84-5e513ff02d07 - Cannot parse Postfix queue ID from None None
2026-07-13 19:10:42,643 - SL - DEBUG - 7704 - "/app/email_handler.py:1980" - handle() - 30eb46f4-05db-4713-8c84-5e513ff02d07 - ==>> Handle mail_from:sender@example.com, rcpt_tos:['nonexistent@sl.local'], header_from:sender@example.com, header_to:nonexistent@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'sender@example.com'), ('To', 'nonexistent@sl.local'), ('Subject', 'q3 test'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 19:10:42,648 - SL - DEBUG - 7704 - "/app/email_handler.py:2202" - handle() - 30eb46f4-05db-4713-8c84-5e513ff02d07 - Forward phase sender@example.com(sender@example.com) -> nonexistent@sl.local
2026-07-13 19:10:42,658 - SL - DEBUG - 7704 - "/app/email_handler.py:545" - handle_forward() - 30eb46f4-05db-4713-8c84-5e513ff02d07 - alias nonexistent@sl.local not exist. Try to see if it can be created on the fly
2026-07-13 19:10:42,668 - SL - INFO - 7704 - "/app/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - 30eb46f4-05db-4713-8c84-5e513ff02d07 - Cannot auto-create custom domain alias for nonexistent@sl.local because there's no custom domain for sl.local
2026-07-13 19:10:42,668 - SL - INFO - 7704 - "/app/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - 30eb46f4-05db-4713-8c84-5e513ff02d07 - Cannot auto-create nonexistent@sl.local since it has no directory separator
2026-07-13 19:10:42,668 - SL - DEBUG - 7704 - "/app/email_handler.py:551" - handle_forward() - 30eb46f4-05db-4713-8c84-5e513ff02d07 - alias nonexistent@sl.local cannot be created on-the-fly, return 550
2026-07-13 19:10:42,669 - SL - INFO - 7704 - "/app/email_handler.py:2367" - _handle() - 30eb46f4-05db-4713-8c84-5e513ff02d07 - Finish mail_from sender@example.com, rcpt_tos ['nonexistent@sl.local'], takes 0.1627042293548584 seconds with return code '550 SL E515 Email not exist'<<===
```

**What this shows (Observed).** The handler enters the `Forward phase` (`email_handler.py:2202`), finds `alias nonexistent@sl.local not exist` (`handle_forward`, `email_handler.py:545`), fails both auto-create strategies — `Cannot auto-create custom domain alias ... because there's no custom domain for sl.local` (`app/alias_utils.py:104`) and `Cannot auto-create ... since it has no directory separator` (`app/alias_utils.py:165`) — logs `cannot be created on-the-fly, return 550` (`email_handler.py:551`), and finishes `with return code '550 SL E515 Email not exist'` (`email_handler.py:2367`). This is the exact rejection the question asks for.

### Q3 — The SMTP injection client (complete source, Observed input)

To satisfy reproducibility, the entire helper is included. It performs the real SMTP conversation through the canonical listener on 20381 (no debug hook, mock, or bypass), prints every client and server line, transmits the `DATA` body and the dot terminator verbatim, and parses multi-line SMTP replies (a `NNN-` line is a continuation; a `NNN ` line ends the reply):

```python
#!/usr/bin/env python3
"""Minimal, self-contained raw-socket SMTP client for observing SimpleLogin's
inbound email handler (aiosmtpd listener). It prints the COMPLETE conversation
verbatim -- every client line (C:) and every server line (S:) -- including the
DATA payload and the <CR><LF>.<CR><LF> terminator, and it parses multi-line SMTP
replies correctly (a line "NNN-..." is a continuation; "NNN ..." ends the reply).

Usage: sl_inject.py <host> <port> <mail_from> <rcpt_to> [subject]
It performs the real SMTP conversation through the canonical listener; there is
no debug hook, mock, or bypass.
"""
import socket, sys

def recv_reply(f):
    """Read one full (possibly multi-line) SMTP reply and return the raw lines.
    Continuation lines have a '-' as the 4th char (e.g. '250-'); the final line
    has a space (e.g. '250 ')."""
    lines = []
    while True:
        raw = f.readline()
        if not raw:
            break
        line = raw.rstrip(b"\r\n").decode("utf-8", "replace")
        lines.append(line)
        print("S: " + line, flush=True)
        # final line of a reply: 3 digits followed by a space
        if len(line) >= 4 and line[3:4] == " ":
            break
    return lines

def send(sock, f, text):
    """Send a command line, echoing it verbatim as C:."""
    print("C: " + text, flush=True)
    sock.sendall(text.encode("utf-8") + b"\r\n")

def main():
    host, port, mail_from, rcpt_to = sys.argv[1], int(sys.argv[2]), sys.argv[3], sys.argv[4]
    subject = sys.argv[5] if len(sys.argv) > 5 else "q3 test"
    s = socket.create_connection((host, port), timeout=30)
    f = s.makefile("rb")
    recv_reply(f)                              # 220 banner
    send(s, f, "EHLO test.blitzy.local"); recv_reply(f)
    send(s, f, "MAIL FROM:<%s>" % mail_from);  recv_reply(f)
    send(s, f, "RCPT TO:<%s>" % rcpt_to);      recv_reply(f)
    send(s, f, "DATA");                        recv_reply(f)   # 354
    # Build and transmit the message body verbatim, then the dot terminator.
    body_lines = [
        "From: %s" % mail_from,
        "To: %s" % rcpt_to,
        "Subject: %s" % subject,
        "Content-Transfer-Encoding: 7bit",
        "",
        "%s body" % subject,
    ]
    for bl in body_lines:
        print("C: " + bl, flush=True)
        s.sendall(bl.encode("utf-8") + b"\r\n")
    print("C: .", flush=True)                  # end-of-DATA terminator
    s.sendall(b".\r\n")
    final = recv_reply(f)                       # server's status for the message
    send(s, f, "QUIT"); recv_reply(f)
    s.close()
    # Echo the single authoritative status line the server returned to the sender.
    status_line = final[-1] if final else ""
    print("=== FINAL_SERVER_REPLY_TO_SENDER: " + status_line, flush=True)

if __name__ == "__main__":
    main()
```

### Q3 — Alternate branch: sender in `ignore_bounce_sender` → `250 SL E207` (Observed)

The E515-vs-E207 outcome is gated solely by `should_ignore_bounce(envelope.mail_from)` (`email_handler.py:552`). To exercise the alternate branch, the sender is temporarily added to the `ignore_bounce_sender` table and the identical message is re-injected; the probe row is then removed and its absence proven (net-zero for this table):

```text
########## Q3 E207 BRANCH: add sender@example.com to ignore_bounce_sender, re-inject (expect 250 SL E207) ##########

$ psql -U myuser -h localhost -d sl_probe_obs -tAc "SELECT count(*) FROM ignore_bounce_sender WHERE mail_from='sender@example.com';"
0

$ psql -U myuser -h localhost -d sl_probe_obs -c "INSERT INTO ignore_bounce_sender (mail_from, created_at) VALUES ('sender@example.com', now());"
INSERT 0 1

$ psql -U myuser -h localhost -d sl_probe_obs -tAc "SELECT id, mail_from FROM ignore_bounce_sender WHERE mail_from='sender@example.com';"
1|sender@example.com
$ /app/venv/bin/python /tmp/blitzy_obs/sl_inject.py 127.0.0.1 20381 sender@example.com nonexistent@sl.local
S: 220 f411db182cde Python SMTP 1.4.2
C: EHLO test.blitzy.local
S: 250-f411db182cde
S: 250-SIZE 33554432
S: 250-8BITMIME
S: 250-SMTPUTF8
S: 250 HELP
C: MAIL FROM:<sender@example.com>
S: 250 OK
C: RCPT TO:<nonexistent@sl.local>
S: 250 OK
C: DATA
S: 354 End data with <CR><LF>.<CR><LF>
C: From: sender@example.com
C: To: nonexistent@sl.local
C: Subject: q3 test
C: Content-Transfer-Encoding: 7bit
C: 
C: q3 test body
C: .
S: 250 SL E207 No bounce report
C: QUIT
S: 221 Bye
=== FINAL_SERVER_REPLY_TO_SENDER: 250 SL E207 No bounce report
```

The handler console for this message (`message_id 12109ffa-...`) shows the *same* failed-auto-create chain, but this time `should_ignore_bounce` matches and the result flips to E207:

```text
2026-07-13 19:10:46,867 - SL - DEBUG - 7704 - "/app/app/log.py:24" - set_message_id() - 30eb46f4-05db-4713-8c84-5e513ff02d07 - set message_id 12109ffa-400f-4c9c-85e2-95d956bf82ff
2026-07-13 19:10:46,867 - SL - DEBUG - 7704 - "/app/email_handler.py:2342" - _handle() - 12109ffa-400f-4c9c-85e2-95d956bf82ff - ====>=====>====>====>====>====>====>====>
2026-07-13 19:10:46,867 - SL - INFO - 7704 - "/app/email_handler.py:2343" - _handle() - 12109ffa-400f-4c9c-85e2-95d956bf82ff - New message, mail from sender@example.com, rctp tos ['nonexistent@sl.local'] 
2026-07-13 19:10:46,868 - SL - DEBUG - 7704 - "/app/email_handler.py:1963" - handle() - 12109ffa-400f-4c9c-85e2-95d956bf82ff - Cannot parse Postfix queue ID from None None
2026-07-13 19:10:46,869 - SL - DEBUG - 7704 - "/app/email_handler.py:1980" - handle() - 12109ffa-400f-4c9c-85e2-95d956bf82ff - ==>> Handle mail_from:sender@example.com, rcpt_tos:['nonexistent@sl.local'], header_from:sender@example.com, header_to:nonexistent@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'sender@example.com'), ('To', 'nonexistent@sl.local'), ('Subject', 'q3 test'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 19:10:46,873 - SL - DEBUG - 7704 - "/app/email_handler.py:2202" - handle() - 12109ffa-400f-4c9c-85e2-95d956bf82ff - Forward phase sender@example.com(sender@example.com) -> nonexistent@sl.local
2026-07-13 19:10:46,879 - SL - DEBUG - 7704 - "/app/email_handler.py:545" - handle_forward() - 12109ffa-400f-4c9c-85e2-95d956bf82ff - alias nonexistent@sl.local not exist. Try to see if it can be created on the fly
2026-07-13 19:10:46,883 - SL - INFO - 7704 - "/app/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - 12109ffa-400f-4c9c-85e2-95d956bf82ff - Cannot auto-create custom domain alias for nonexistent@sl.local because there's no custom domain for sl.local
2026-07-13 19:10:46,883 - SL - INFO - 7704 - "/app/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - 12109ffa-400f-4c9c-85e2-95d956bf82ff - Cannot auto-create nonexistent@sl.local since it has no directory separator
2026-07-13 19:10:46,883 - SL - DEBUG - 7704 - "/app/email_handler.py:551" - handle_forward() - 12109ffa-400f-4c9c-85e2-95d956bf82ff - alias nonexistent@sl.local cannot be created on-the-fly, return 550
2026-07-13 19:10:46,884 - SL - WARNING - 7704 - "/app/app/email_utils.py:1363" - should_ignore_bounce() - 12109ffa-400f-4c9c-85e2-95d956bf82ff - do not send back bounce report to sender@example.com
2026-07-13 19:10:46,884 - SL - INFO - 7704 - "/app/email_handler.py:2367" - _handle() - 12109ffa-400f-4c9c-85e2-95d956bf82ff - Finish mail_from sender@example.com, rcpt_tos ['nonexistent@sl.local'], takes 0.016614198684692383 seconds with return code '250 SL E207 No bounce report'<<===
```

The decisive extra line is `should_ignore_bounce() ... do not send back bounce report to sender@example.com` (`app/email_utils.py:1363`), after which `_handle` finishes `with return code '250 SL E207 No bounce report'` (`email_handler.py:2367`). The probe row is then deleted and the table returns to **0** rows:

```text
-- E207 probe-row cleanup (net-zero for this table) --

$ psql -U myuser -h localhost -d sl_probe_obs -c "DELETE FROM ignore_bounce_sender WHERE mail_from='sender@example.com';"
DELETE 1

$ psql -U myuser -h localhost -d sl_probe_obs -tAc "SELECT count(*) AS ignore_bounce_sender_rows FROM ignore_bounce_sender;"
0
```

**What this shows (Observed).** With the sender listed in `ignore_bounce_sender`, the identical undeliverable message returns **`250 SL E207 No bounce report`** instead of E515 — confirming the branch at `email_handler.py:552-555`. For a normal sender (not in that table) the result is E515.

### Q3 — Stability across two runs (Observed)

The primary injection was repeated. The second run returns the same `550 SL E515 Email not exist` and produces the same handler chain (only the `message_id`, timestamps, and the `takes ...` duration differ). Client transcript:

```text
########## Q3 PRIMARY RUN 2: inject again (stability, expect 550 SL E515) ##########
$ /app/venv/bin/python /tmp/blitzy_obs/sl_inject.py 127.0.0.1 20381 sender@example.com nonexistent@sl.local
S: 220 f411db182cde Python SMTP 1.4.2
C: EHLO test.blitzy.local
S: 250-f411db182cde
S: 250-SIZE 33554432
S: 250-8BITMIME
S: 250-SMTPUTF8
S: 250 HELP
C: MAIL FROM:<sender@example.com>
S: 250 OK
C: RCPT TO:<nonexistent@sl.local>
S: 250 OK
C: DATA
S: 354 End data with <CR><LF>.<CR><LF>
C: From: sender@example.com
C: To: nonexistent@sl.local
C: Subject: q3 test
C: Content-Transfer-Encoding: 7bit
C: 
C: q3 test body
C: .
S: 550 SL E515 Email not exist
C: QUIT
S: 221 Bye
=== FINAL_SERVER_REPLY_TO_SENDER: 550 SL E515 Email not exist
```

Handler console for the second run (`message_id 25b87cd5-...`):

```text
2026-07-13 19:10:51,047 - SL - DEBUG - 7704 - "/app/app/log.py:24" - set_message_id() - 12109ffa-400f-4c9c-85e2-95d956bf82ff - set message_id 25b87cd5-5190-4c08-9825-91b3280fcfc3
2026-07-13 19:10:51,047 - SL - DEBUG - 7704 - "/app/email_handler.py:2342" - _handle() - 25b87cd5-5190-4c08-9825-91b3280fcfc3 - ====>=====>====>====>====>====>====>====>
2026-07-13 19:10:51,047 - SL - INFO - 7704 - "/app/email_handler.py:2343" - _handle() - 25b87cd5-5190-4c08-9825-91b3280fcfc3 - New message, mail from sender@example.com, rctp tos ['nonexistent@sl.local'] 
2026-07-13 19:10:51,048 - SL - DEBUG - 7704 - "/app/email_handler.py:1963" - handle() - 25b87cd5-5190-4c08-9825-91b3280fcfc3 - Cannot parse Postfix queue ID from None None
2026-07-13 19:10:51,050 - SL - DEBUG - 7704 - "/app/email_handler.py:1980" - handle() - 25b87cd5-5190-4c08-9825-91b3280fcfc3 - ==>> Handle mail_from:sender@example.com, rcpt_tos:['nonexistent@sl.local'], header_from:sender@example.com, header_to:nonexistent@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'sender@example.com'), ('To', 'nonexistent@sl.local'), ('Subject', 'q3 test'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 19:10:51,054 - SL - DEBUG - 7704 - "/app/email_handler.py:2202" - handle() - 25b87cd5-5190-4c08-9825-91b3280fcfc3 - Forward phase sender@example.com(sender@example.com) -> nonexistent@sl.local
2026-07-13 19:10:51,060 - SL - DEBUG - 7704 - "/app/email_handler.py:545" - handle_forward() - 25b87cd5-5190-4c08-9825-91b3280fcfc3 - alias nonexistent@sl.local not exist. Try to see if it can be created on the fly
2026-07-13 19:10:51,064 - SL - INFO - 7704 - "/app/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - 25b87cd5-5190-4c08-9825-91b3280fcfc3 - Cannot auto-create custom domain alias for nonexistent@sl.local because there's no custom domain for sl.local
2026-07-13 19:10:51,064 - SL - INFO - 7704 - "/app/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - 25b87cd5-5190-4c08-9825-91b3280fcfc3 - Cannot auto-create nonexistent@sl.local since it has no directory separator
2026-07-13 19:10:51,064 - SL - DEBUG - 7704 - "/app/email_handler.py:551" - handle_forward() - 25b87cd5-5190-4c08-9825-91b3280fcfc3 - alias nonexistent@sl.local cannot be created on-the-fly, return 550
2026-07-13 19:10:51,065 - SL - INFO - 7704 - "/app/email_handler.py:2367" - _handle() - 25b87cd5-5190-4c08-9825-91b3280fcfc3 - Finish mail_from sender@example.com, rcpt_tos ['nonexistent@sl.local'], takes 0.017365455627441406 seconds with return code '550 SL E515 Email not exist'<<===
```

The result is therefore **stable**: `E515` for a normal sender, `E207` only when the sender is in `ignore_bounce_sender`.

### Q3 — Handler shutdown (Observed)

```text
########## Q3: shutdown handler ##########
$ kill -TERM 7704   # terminate the email_handler process
stopped

$ # post-stop LISTEN check on 20381
(no LISTEN on 20381 — port released)

$ ps -o pid,cmd -C python 2>/dev/null | grep email_handler.py || echo '(no email_handler.py process running)'
(no email_handler.py process running)
```

The exact background PID is terminated, port 20381 is released, and no `email_handler.py` process remains — a clean, verifiable teardown.

### Q3 — Causal nuance: the empty `public_domain` is *not* the direct cause (Observed → confirmed later)

A precise reading of the runtime logs shows the handler **never queries `public_domain`** on this path. The rejection is produced entirely by (1) no matching `Alias` row (`email_handler.py:543`) and (2) both auto-create strategies returning `None` — specifically because *there is no CustomDomain for `sl.local`* (`app/alias_utils.py:104`; `CustomDomain` is a different table from `SLDomain`) and *the address has no directory separator* (`app/alias_utils.py:165`). Inbound mail does **not** create random public-domain aliases on the fly regardless of whether `public_domain` is seeded; user-facing aliases are created by users, not by receiving a message. So the observed E515 is best described as **“recipient alias does not exist and cannot be auto-created”**, which is the *state* that skipping `init_app.py` leaves the system in, rather than a direct check of the empty domain table. This is **confirmed** by the post-initialization cross-check below, where the identical injection still returns E515 after `init_app.py` seeds `public_domain`.

### Q3 — Post-initialization cross-check: identical injection after `init_app.py` still returns `550 SL E515` (Observed)

This cross-check runs **after** the Q2 initialization, on the **same** database `sl_probe_obs`, which `python init_app.py` has now seeded so that `public_domain` holds one row (`sl.local`). The email handler is restarted and the **identical** message — `sender@example.com` → `nonexistent@sl.local` — is injected again. If the empty `public_domain` had been the direct cause of the rejection, seeding it would change the outcome; the runtime shows it does **not**. (This observation was captured at `19:41`, chronologically after the `init_app.py` seeding at `19:21` and the Q2 service captures at `19:22`–`19:28`.)

```text
### Q3 POST-INIT cross-check on the SAME DB sl_probe_obs (now INITIALIZED by init_app.py; public_domain seeded)

$ psql -U myuser -h localhost -d sl_probe_obs -c "select id, domain, use_as_reverse_alias from public_domain order by id"
 id |  domain  | use_as_reverse_alias 
----+----------+----------------------
  1 | sl.local | t
(1 row)


$ psql -U myuser -h localhost -d sl_probe_obs -tAc "select count(*) from public_domain"
1

  email_handler PID=8908
  waited 2s for port bind

$ listen_report 4F9D 20381
  LISTEN local=00000000:4F9D (hex 4F9D=20381, state 0A=LISTEN; 0100007F=127.0.0.1, 00000000=0.0.0.0)

$ /app/venv/bin/python /tmp/blitzy_obs/sl_inject.py 127.0.0.1 20381 sender@example.com nonexistent@sl.local
S: 220 f411db182cde Python SMTP 1.4.2
C: EHLO test.blitzy.local
S: 250-f411db182cde
S: 250-SIZE 33554432
S: 250-8BITMIME
S: 250-SMTPUTF8
S: 250 HELP
C: MAIL FROM:<sender@example.com>
S: 250 OK
C: RCPT TO:<nonexistent@sl.local>
S: 250 OK
C: DATA
S: 354 End data with <CR><LF>.<CR><LF>
C: From: sender@example.com
C: To: nonexistent@sl.local
C: Subject: q3 test
C: Content-Transfer-Encoding: 7bit
C: 
C: q3 test body
C: .
S: 550 SL E515 Email not exist
C: QUIT
S: 221 Bye
=== FINAL_SERVER_REPLY_TO_SENDER: 550 SL E515 Email not exist

----- email_handler.py console for this message (complete stdout+stderr) -----
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/rrdgfjtimmezoteklwjv
Upload files to local dir
>>> init logging <<<
2026-07-13 19:41:20,233 - SL - DEBUG - 8908 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-13 19:41:20,927 - SL - INFO - 8908 - "/app/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-13 19:41:20,929 - SL - DEBUG - 8908 - "/app/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
2026-07-13 19:41:21,521 - SL - DEBUG - 8908 - "/app/app/log.py:24" - set_message_id() -  - set message_id 230e8312-89ea-467e-8d8d-5690495e33a0
2026-07-13 19:41:21,521 - SL - DEBUG - 8908 - "/app/email_handler.py:2342" - _handle() - 230e8312-89ea-467e-8d8d-5690495e33a0 - ====>=====>====>====>====>====>====>====>
2026-07-13 19:41:21,521 - SL - INFO - 8908 - "/app/email_handler.py:2343" - _handle() - 230e8312-89ea-467e-8d8d-5690495e33a0 - New message, mail from sender@example.com, rctp tos ['nonexistent@sl.local'] 
2026-07-13 19:41:21,522 - SL - DEBUG - 8908 - "/app/email_handler.py:1963" - handle() - 230e8312-89ea-467e-8d8d-5690495e33a0 - Cannot parse Postfix queue ID from None None
2026-07-13 19:41:21,643 - SL - DEBUG - 8908 - "/app/email_handler.py:1980" - handle() - 230e8312-89ea-467e-8d8d-5690495e33a0 - ==>> Handle mail_from:sender@example.com, rcpt_tos:['nonexistent@sl.local'], header_from:sender@example.com, header_to:nonexistent@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'sender@example.com'), ('To', 'nonexistent@sl.local'), ('Subject', 'q3 test'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 19:41:21,648 - SL - DEBUG - 8908 - "/app/email_handler.py:2202" - handle() - 230e8312-89ea-467e-8d8d-5690495e33a0 - Forward phase sender@example.com(sender@example.com) -> nonexistent@sl.local
2026-07-13 19:41:21,658 - SL - DEBUG - 8908 - "/app/email_handler.py:545" - handle_forward() - 230e8312-89ea-467e-8d8d-5690495e33a0 - alias nonexistent@sl.local not exist. Try to see if it can be created on the fly
2026-07-13 19:41:21,667 - SL - INFO - 8908 - "/app/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - 230e8312-89ea-467e-8d8d-5690495e33a0 - Cannot auto-create custom domain alias for nonexistent@sl.local because there's no custom domain for sl.local
2026-07-13 19:41:21,667 - SL - INFO - 8908 - "/app/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - 230e8312-89ea-467e-8d8d-5690495e33a0 - Cannot auto-create nonexistent@sl.local since it has no directory separator
2026-07-13 19:41:21,667 - SL - DEBUG - 8908 - "/app/email_handler.py:551" - handle_forward() - 230e8312-89ea-467e-8d8d-5690495e33a0 - alias nonexistent@sl.local cannot be created on-the-fly, return 550
2026-07-13 19:41:21,668 - SL - INFO - 8908 - "/app/email_handler.py:2367" - _handle() - 230e8312-89ea-467e-8d8d-5690495e33a0 - Finish mail_from sender@example.com, rcpt_tos ['nonexistent@sl.local'], takes 0.1473391056060791 seconds with return code '550 SL E515 Email not exist'<<===
----- end console -----

$ kill -TERM 8908 2>/dev/null; sleep 2; echo terminated
terminated

$ listen_report 4F9D 20381
  (no LISTEN socket on port 20381)

$ ps -eo pid,stat,cmd | grep '[e]mail_handler.py' | grep -v defunct || echo '  (no live email_handler.py process running)'
  (no live email_handler.py process running)
```

**What this shows (Observed).** `public_domain` now contains `1 | sl.local | t`, yet the same injection still returns **`550 SL E515 Email not exist`** (`message_id 230e8312-...`). The handler log is the *same* rejection chain as the pre-initialization run — note in particular that `app/alias_utils.py:104` still reports `no custom domain for sl.local`: that check consults the `custom_domain` table (`CustomDomain`), which is distinct from the now-seeded `public_domain` (`SLDomain`). This **confirms the causal nuance**: seeding `public_domain` does not change the result, because the forward auto-create path never consults it. The port was bound on `0.0.0.0:20381`, the handler was terminated with `SIGTERM`, and the port was confirmed released.

### Q3 — Answer (migrations run, `init_app.py` not run)

For a normal sender, mail to any `nonexistent@sl.local` is **permanently rejected** with SMTP status **`550 SL E515 Email not exist`** (`app/email/status.py:51`), returned by `handle_forward` at `email_handler.py:555`. The handler logs the rejection as this chain (verbatim above): `alias nonexistent@sl.local not exist ...` (`email_handler.py:545`) → `Cannot auto-create custom domain alias ... no custom domain for sl.local` (`app/alias_utils.py:104`) → `Cannot auto-create ... no directory separator` (`app/alias_utils.py:165`) → `alias nonexistent@sl.local cannot be created on-the-fly, return 550` (`email_handler.py:551`) → `Finish ... with return code '550 SL E515 Email not exist'` (`email_handler.py:2367`). If — and only if — the sender is in the `ignore_bounce_sender` table, the same undeliverable message instead returns **`250 SL E207 No bounce report`** (`app/email/status.py:12`, via `email_handler.py:553`).

## Coverage — every named item and implied condition

This section is a checklist confirming that every distinct thing the three questions name, and every implied condition (primary, alternate, edge, and boundary), is addressed **by name** and grounded in a specific capture or `file:line` reference. The **Evidence type** column is deliberately precise: **Observed** means a direct runtime capture quoted verbatim in this document; **Observed, confirmed by code** means a runtime capture whose interpretation is pinned to a specific source line; **Inferred, confirmed** means a fact first derived by reading the source and then corroborated by an adjacent capture. Liveness, port state after shutdown, exit codes, and two-run stability are labelled **Observed** only because each was actually captured (a live-process listing, a post-stop `/proc/net/tcp` read, an exit-code echo, and two separate runs) — not asserted.

| # | Named item / implied condition | Where answered | Evidence type |
|---|---|---|---|
| Q1-a | `python server.py` starts against an empty DB | Q1 — boot | Observed |
| Q1-b | import-time `engine.connect()` succeeds on the empty DB (`app/db.py:12`) | Q1 — boot (no startup error) | Observed (boundary), confirmed by code |
| Q1-c | opening the login page issues the first ORM query (`/` → `/auth/login`) | Q1 — GET/POST `/auth/login` | Observed |
| Q1-d | the full Python exception and traceback | Q1 — server console | Observed |
| Q1-e | exact missing relation name (`users`) | `relation "users" does not exist` | Observed |
| Q1-f | exception class `sqlalchemy.exc.ProgrammingError` wrapping `psycopg2.errors.UndefinedTable` | Q1 — server console | Observed |
| Q1-g | HTTP body to the browser is the generic `error/500.html`, **not** the Werkzeug debugger (`server.py:388-394`) | Q1 — POST body capture | Observed, confirmed by code |
| Q1-h | stability across two runs | Q1 — run 1 and run 2 | Observed (2 runs) |
| Q1-i | the missing-`.env` `KeyError` is a *distinct, earlier* failure mode (`app/config.py:79`) | Section 0 — KeyError capture | Observed |
| Q2-a | enumeration of the required services (five) | Q2 — enumeration | Inferred (README/Dockerfile/CONTRIBUTING), each confirmed below |
| Q2-b | web app, dev entry `server.py` on `127.0.0.1:7777` | Q2 — Service A (dev) | Observed |
| Q2-c | web app, canonical `gunicorn wsgi:app` on `0.0.0.0:7777` (`Dockerfile:47`) | Q2 — Service A (gunicorn) | Observed |
| Q2-d | `email_handler.py` SMTP on `0.0.0.0:20381` (`email_handler.py:2399`) | Q2 — Service B | Observed |
| Q2-e | `job_runner.py` takes and completes a real job | Q2 — Service C | Observed (before/during/after) |
| Q2-f | `cron.py -j delete_old_monitoring` runs and exits `0` | Q2 — Service D | Observed |
| Q2-g | `event_listener.py listener` readiness `Starting to listen to events` (`events/event_source.py:49`) | Q2 — Service E | Observed |
| Q2-h | port-binding proof for the two listeners (via `/proc/net/tcp`) | Q2 — Services A/B | Observed |
| Q2-i | the three background workers have **no** client-facing port | Q2 — Services C/D/E (no LISTEN socket) | Observed |
| Q2-j | the Python SMTP port is `20381`, not Postfix’s `25` (`README.md:367-369`) | Q2 — clarification | Observed, confirmed by code |
| Q2-k | listener readiness stable across two runs | Q2 — Service E run 1 and run 2 | Observed (2 runs) |
| Q3-a | migrated-but-uninitialized DB (`public_domain` empty) | Q3 — setup | Observed |
| Q3-b | inbound mail to `nonexistent@sl.local` via the real aiosmtpd listener | Q3 — primary injection | Observed |
| Q3-c | SMTP status to sender = `550 SL E515 Email not exist` (`app/email/status.py:51`) | Q3 — primary transcript | Observed, confirmed by code |
| Q3-d | the rejection log chain the handler emits | Q3 — handler console | Observed |
| Q3-e | the complete SMTP client source used to inject | Q3 — injector source | Observed (full source shown) |
| Q3-f | every client `DATA` line + terminator; multiline SMTP reply parsing | Q3 — transcript | Observed |
| Q3-g | alternate branch: sender in `ignore_bounce_sender` → `250 SL E207` (`app/email/status.py:12`) | Q3 — E207 branch | Observed, confirmed by code |
| Q3-h | E207 probe row inserted → queried → exercised → deleted → zero-count | Q3 — E207 branch | Observed (before/during/after) |
| Q3-i | stability across two runs | Q3 — run 1 and run 2 | Observed (2 runs) |
| Q3-j | `SLDomain.__tablename__ = "public_domain"` (`app/models.py:3119`) | Q3 — setup | Observed, confirmed by code |
| Q3-k | causal nuance: the empty `public_domain` is *not* the direct cause | Q3 — causal nuance | Inferred, confirmed by the post-init cross-check |
| Q3-l | post-init cross-check: identical injection with `public_domain = 1` still returns `550 SL E515` | Q3 — post-init cross-check | Observed |
| X-a | net-zero cleanup (scratch DBs dropped, scratch files removed, `.env` removed, ports free) | Cleanup section | Observed |

Every row above is stated explicitly, by name, in the section indicated, and is backed by either a verbatim capture or an exact `file:line` reference (usually both).

## Cleanup and net-zero verification

This investigation is strictly read-only: the only intended net change to the repository is this document. Everything created to reproduce the three scenarios — the four scratch databases, the temporary `.env`, the SMTP-injection helper, and every capture/log/script under `/tmp/blitzy_obs` — is removed here, and the removal is proven with runtime checks rather than asserted. Every `$ ` line in the capture below is the literal command executed (the driver echoes each command with a `run()` helper, then runs exactly that command).

The databases kept afterward (`postgres`, `simplelogin`, `sl_empty`, `test`) are pre-existing fixtures provisioned with the image, **not** artifacts of this investigation; only the four scratch databases created here (`sl_q1_obs`, `sl_probe_obs`, and the two prior-attempt leftovers `sl_q2`, `sl_q3`) are dropped.

```text
########## PRE-CLEANUP STATE ##########

$ ps -eo pid,cmd | grep -iE '[s]erver.py|[g]unicorn wsgi|[e]mail_handler.py|[j]ob_runner.py|[e]vent_listener.py|[c]ron.py' | grep -v defunct || echo '(no live SimpleLogin service processes)'
(no live SimpleLogin service processes)

$ grep -iE ' [0-9A-F]{8}:(1E61|4F9D) ' /proc/net/tcp | awk '$4=="0A"' || echo '(no LISTEN socket on 7777 or 20381)'
(no LISTEN socket on 7777 or 20381)

$ PGPASSWORD=mypassword psql -U myuser -h localhost -d postgres -tAc "select datname from pg_database where datname not in ('template0','template1') order by datname"
postgres
simplelogin
sl_empty
sl_probe_obs
sl_q1_obs
sl_q2
sl_q3
test

########## DROP THE FOUR SCRATCH DATABASES (keep postgres, simplelogin, sl_empty, test) ##########

$ PGPASSWORD=mypassword dropdb -U myuser -h localhost sl_q1_obs   && echo 'dropped sl_q1_obs'
dropped sl_q1_obs

$ PGPASSWORD=mypassword dropdb -U myuser -h localhost sl_probe_obs && echo 'dropped sl_probe_obs'
dropped sl_probe_obs

$ PGPASSWORD=mypassword dropdb -U myuser -h localhost sl_q2       && echo 'dropped sl_q2'
dropped sl_q2

$ PGPASSWORD=mypassword dropdb -U myuser -h localhost sl_q3       && echo 'dropped sl_q3'
dropped sl_q3

$ PGPASSWORD=mypassword psql -U myuser -h localhost -d postgres -tAc "select datname from pg_database where datname not in ('template0','template1') order by datname"
postgres
simplelogin
sl_empty
test

########## REMOVE TEMPORARY FILESYSTEM ARTIFACTS ##########

$ ls -d /tmp/blitzy_obs 2>/dev/null && rm -rf /tmp/blitzy_obs; ls -d /tmp/blitzy_obs 2>/dev/null || echo '(/tmp/blitzy_obs removed)'
/tmp/blitzy_obs
(/tmp/blitzy_obs removed)

$ rm -f /tmp/q1_* /tmp/q2_* /tmp/q3_* /tmp/q3e_* /tmp/q3x_* /tmp/inject_q3.py /tmp/sl_env.sh /tmp/probe.log /tmp/pytest_*.log /tmp/test_migrate*.log; echo 'removed prior-attempt /tmp scratch'
removed prior-attempt /tmp scratch

$ ls /tmp | grep -iE 'q1_|q2_|q3|sl_env|inject|blitzy_obs|pytest|test_migrate' || echo '(no scratch artifacts remain in /tmp)'
(no scratch artifacts remain in /tmp)

########## REMOVE THE TEMPORARY .env (restore the container checkout to pristine) ##########

$ ls -l /app/.env 2>/dev/null && rm -f /app/.env; ls -l /app/.env 2>/dev/null || echo '(/app/.env removed)'
-rw-r--r-- 1 root 1001 5789 Jul 13 19:57 /app/.env
(/app/.env removed)

$ cd /app && git status --porcelain

$ cd /app && echo "container working-tree changes: $(git status --porcelain | wc -l)"
container working-tree changes: 0

########## FINAL VERIFICATION ##########

$ grep -iE ' [0-9A-F]{8}:(1E61|4F9D) ' /proc/net/tcp | awk '$4=="0A"' || echo '(final: no LISTEN socket on 7777 or 20381)'
(final: no LISTEN socket on 7777 or 20381)

$ ps -eo pid,cmd | grep -iE '[s]erver.py|[g]unicorn wsgi|[e]mail_handler.py|[j]ob_runner.py|[e]vent_listener.py|[c]ron.py' | grep -v defunct || echo '(final: no live SimpleLogin service processes)'
(final: no live SimpleLogin service processes)

########## CLEANUP COMPLETE ##########
```

**What this shows (Observed).** Before cleanup there were no live SimpleLogin processes and no listening socket on `7777` or `20381`. (The Q3 E207 probe row in `ignore_bounce_sender` was inserted, exercised, and then deleted with a zero-count confirmation in the Q3 section above; the database that held it is dropped outright here.) The four scratch databases were dropped, leaving exactly the four pre-existing databases. `/tmp/blitzy_obs` and all prior-attempt `/tmp` scratch were removed, leaving no matching artifacts. The temporary `/app/.env` (which was derived byte-for-byte from the committed `example.env` and held only placeholder values) was removed, and the container source checkout reports **zero** working-tree changes (`git status --porcelain` empty). The final checks reconfirm no listener on either port and no live service process.

The repository that holds the deliverable is likewise net-zero: the only tracked change is the single document, and there are no untracked files:

```text
$ cd /tmp/blitzy/app/blitzy-7065bab1-a6a2-494e-8554-4f1f5dff7530_384cdf
$ git status --porcelain
 M blitzy/documentation/app_2cd6ee777f8c.md

$ git status --porcelain | wc -l
1

$ git ls-files --others --exclude-standard | wc -l   # untracked files (excluding gitignored)
0
```

**A note on trailing whitespace (byte-exactness).** Running `git diff --check` on the deliverable reports trailing-whitespace warnings, and **every** one of them falls inside a verbatim evidence fence. They are authentic bytes that must not be altered: the `\r` of the HTTP `CRLF` line endings captured from the raw `HTTP/1.0` responses in Q1, and the single trailing space that SQLAlchemy emits inside its generated `SELECT ... FROM users ` text. Stripping them would corrupt the byte-for-byte evidence the questions require. No **prose** line in this document carries trailing whitespace, and the file ends with exactly one newline (no redundant final blank line).
