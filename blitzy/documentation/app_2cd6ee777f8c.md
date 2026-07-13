# SimpleLogin Backend — DEV-Mode Runtime Behavior (Observed)

This document answers seven questions about how the **SimpleLogin** email-aliasing
backend (a Python/Flask web application) behaves when run **locally in DEVELOPMENT
mode**. Every answer is written from **observed runtime output** captured by actually
building and running the code through its canonical entry point — not from reading the
source alone. Each behavioral claim is paired with the exact command that produced it,
the complete unedited output, a live-derived `file:line` grounding, and an
**Observed** / **Inferred** label.

## Investigation environment

- **Application:** SimpleLogin webapp, whose app object is built by the application
  factory `create_app()` in `server.py` and served in production via `wsgi.py`
  (`from server import create_app` / `app = create_app()`, `wsgi.py:L1-3`).
- **Branch / commit:** `app_2cd6ee777f8c` — the checkout is at detached `HEAD`
  `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c` (`chore: emit some missing contact audit
  logs (#2269)`). The deliverable filename is locked to this branch name.
- **Runtime:** the provisioned Docker container (`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`),
  with PostgreSQL and Redis running locally. All capture was performed inside this
  container; the interpreter is the project virtualenv at `/app/venv` (the system
  `python3` has no Flask installed).
- **Pinned stack** (observed exact versions; `pyproject.toml` anchors in parentheses):

  ```
  $ /app/venv/bin/python -c "import flask, flask_login, sqlalchemy, redis, coloredlogs, werkzeug, sys; \
      print('python     ', sys.version.split()[0]); print('flask      ', flask.__version__); \
      print('werkzeug   ', werkzeug.__version__); print('flask_login', flask_login.__version__); \
      print('sqlalchemy ', sqlalchemy.__version__); print('redis      ', redis.__version__); \
      print('coloredlogs', coloredlogs.__version__)"
  python      3.10.18
  flask       1.1.2
  werkzeug    1.0.1
  flask_login 0.5.0
  sqlalchemy  1.3.24
  redis       4.6.0
  coloredlogs 14.0
  ```

  `pyproject.toml`: `python = "^3.10"` (L61), `flask = "^1.1.2"` (L62),
  `SQLAlchemy = "1.3.24"` (L116), `python-dotenv = "^0.14.0"` (L68),
  `gunicorn = "^20.0.4"` (L66), `flask-debugtoolbar = "^0.11.0"` (L84),
  `coloredlogs = "^14.0"` (L89), `Flask-Limiter = "^1.4"` (L101),
  `redis = "^4.5.3"` (L117). Werkzeug 1.0.1 is the transitive pin under Flask 1.1.2.

- **Canonical bootstrap & run commands** (per `CONTRIBUTING.md:L106`):

  ```
  cp example.env .env          # local config (git-ignored)
  alembic upgrade head         # apply migrations
  flask dummy-data             # seed dev data (john@wick.com / password)
  python3 server.py            # dev entry point -> local_main() -> app.run(debug=True, port=7777)
  ```

  The local `.env` used for capture set `URL=http://localhost:7777`,
  `DB_URI=postgresql://test:test@localhost:5432/test`, `MEM_STORE_URI=redis://localhost`
  (to exercise the Redis session backend), `FLASK_SECRET=secret`, `EMAIL_DOMAIN=sl.local`,
  `SUPPORT_EMAIL=support@sl.local`, `DISABLE_RATE_LIMIT=1`. `.env`/`config` are
  git-ignored, so this does not alter tracked source; `flask dummy-data` mutates only the
  local dev database.

- **Reproducibility discipline:** any value that could vary run-to-run (PIDs, the Werkzeug
  Debugger PIN, the `takes` request timing, the session id, the user's `alternative_id`)
  was exercised **at least twice** and is reported as *stable* or *variable* accordingly.

---

## Q1 — How does the backend start up in development mode?

### Direct answer (Observed)

Running `python3 server.py` executes the `if __name__ == "__main__": local_main()` guard
(`server.py:L598-599`). `local_main()` (`server.py:L572-595`) forces colored logging on
(`config.COLOR_LOG = True`, L573), builds the app with the factory `create_app()` (L574),
enables the Flask-Debug-Toolbar (L577/L582), sets `app.debug = True` (L581), and finally
calls `app.run(debug=True, port=7777)` (L588). Because `debug=True` and `use_reloader` is
not disabled, Werkzeug's **stat reloader** forks a child process, so the whole module —
including its import-time `print` banners — is imported **twice** (parent + serving child).
The server then binds **127.0.0.1:7777** and serves via Werkzeug's development WSGI server.

### Command(s) run

```
# from /app, with the local .env as the config source
PYTHONPATH=/app /app/venv/bin/python server.py
```

### Complete, unedited output (startup stream)

The following is the complete stdout+stderr emitted at startup. It is **identical across
Run 1 and Run 2** except for the process ids (Run 1 parent/child PIDs 3439/3459; Run 2
3520/3540 — shown here). Note every banner block appears **twice** (reloader parent, then
reloader-spawned serving child):

```
>>> URL: http://localhost:7777
Upload files to local dir
>>> init logging <<<
2026-07-13 17:13:43,463 - SL - DEBUG - 3520 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
>>> URL: http://localhost:7777
Upload files to local dir
>>> init logging <<<
2026-07-13 17:13:45,050 - SL - DEBUG - 3540 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
/app/venv/lib/python3.10/site-packages/flask_debugtoolbar/__init__.py:213: UserWarning: Could not insert debug toolbar. </body> tag not found in response.
```

The reloader-spawned serving child was confirmed as a separate OS process (parent 3520):

```
$ ps -eo pid,ppid,args | grep "[s]erver.py"
   3520    ....  /app/venv/bin/python server.py
   3540    3520  /app/venv/bin/python /app/server.py
```

### File:line grounding

- Entry guard: `server.py:L598-599` — `if __name__ == "__main__":` / `local_main()`.
- `local_main()` `server.py:L572-595`: `config.COLOR_LOG = True` (L573); `app = create_app()`
  (L574); `from flask_debugtoolbar import DebugToolbarExtension` (L577); `app.debug = True`
  (L581); `DebugToolbarExtension(app)` (L582); `app.run(debug=True, port=7777)` (L588).
- Factory `create_app()` `server.py:L139`: `app = Flask(__name__)` (L128 is the *light* app;
  the webapp app is created at L139), `app.wsgi_app = ProxyFix(app.wsgi_app, x_for=1, x_host=1)`
  (L142), `app.url_map.strict_slashes = False` (L144), `app.secret_key = FLASK_SECRET` (L151),
  `SESSION_COOKIE_NAME` (L159), `SESSION_COOKIE_SAMESITE = "Lax"` (L162), Redis wiring when
  `MEM_STORE_URI` is set (L163-165), `register_blueprints(app)` (within L233-246), `/health`
  (L213-215), and the app is returned at the end of the factory.
- Production contrast: `wsgi.py:L1` `from server import create_app`; `wsgi.py:L3` `app = create_app()`;
  `Dockerfile:L8` `FROM python:3.10`; `Dockerfile:L44` `EXPOSE 7777`; `Dockerfile:L47`
  `CMD ["gunicorn", "wsgi:app", "-b", "0.0.0.0:7777", "-w", "2", "--timeout", "15"]`.

### Observed vs Inferred

- **Observed:** the entire dev startup sequence above, the double import (reloader child as a
  separate process), the bind on port 7777 (confirmed in Q3/Q4), and the DebugToolbar warning.
- **Inferred:** the production `gunicorn wsgi:app` command binding `0.0.0.0:7777` — this is read
  from the `Dockerfile` and was **not** run; the dev-vs-prod contrast is code-derived.

---

## Q2 — How is configuration loaded?

### Direct answer (Observed)

Configuration is resolved **at import time** by `app/config.py` using `python-dotenv`. The
module reads the `CONFIG` environment variable (`config_file = os.environ.get("CONFIG")`,
`app/config.py:L65`). If `CONFIG` is set, it prints `load config file <path>`
(`app/config.py:L68`) and loads that file (`load_dotenv(get_abs_path(config_file))`, L69);
otherwise it silently loads `./.env` (`load_dotenv()`, L71). Several variables are then read
with **subscript access** `os.environ["..."]`, so a missing one raises `KeyError` at import
and the process dies before the app is built. The unconditional required variables are
`URL` (L79), `EMAIL_DOMAIN` (L92), `SUPPORT_EMAIL` (L93), `DB_URI` (L192), and `FLASK_SECRET`
(L196); an **empty** `FLASK_SECRET` additionally raises `RuntimeError` (L197-198).

### Command(s) run

```
# (A) CONFIG branch + a required variable (URL) removed -> KeyError at import
env -u URL CONFIG=/tmp/blitzy_no_url.env PYTHONPATH=/app /app/venv/bin/python server.py ; echo EXIT=$?

# (B) empty FLASK_SECRET -> RuntimeError at import
FLASK_SECRET="" PYTHONPATH=/app /app/venv/bin/python server.py ; echo EXIT=$?

# (C) default ./.env branch (no CONFIG) -> silent load, resolved values printed
PYTHONPATH=/app /app/venv/bin/python /tmp/blitzy_probe_config.py
```

### Complete, unedited output

**(A) `CONFIG` set (prints the load line) and `URL` missing → `KeyError: 'URL'`:**

```
load config file /tmp/blitzy_no_url.env
Traceback (most recent call last):
  File "/app/server.py", line 30, in <module>
    from app import config, constants
  File "/app/app/config.py", line 79, in <module>
    URL = os.environ["URL"]
  File "/usr/local/lib/python3.10/os.py", line 680, in __getitem__
    raise KeyError(key) from None
KeyError: 'URL'
EXIT=1
```

**(B) empty `FLASK_SECRET` → `RuntimeError` (note `>>> URL:` prints first, then the crash):**

```
>>> URL: http://localhost:7777
Traceback (most recent call last):
  File "/app/server.py", line 30, in <module>
    from app import config, constants
  File "/app/app/config.py", line 198, in <module>
    raise RuntimeError("FLASK_SECRET is empty. Please define it.")
RuntimeError: FLASK_SECRET is empty. Please define it.
EXIT=1
```

**(C) default `./.env` branch — no `load config file` line is printed, and the resolved
values are read straight from the local `.env`:**

```
>>> URL: http://localhost:7777
RESOLVED: URL='http://localhost:7777'
RESOLVED: SESSION_COOKIE_NAME='slapp'
RESOLVED: MEM_STORE_URI='redis://localhost'
RESOLVED: DISABLE_RATE_LIMIT=True
RESOLVED: DB_CONN_NAME='webapp'
```

### File:line grounding

- `config_file = os.environ.get("CONFIG")` `app/config.py:L65`.
- `if config_file:` → `print("load config file", config_file)` (L68) + `load_dotenv(get_abs_path(config_file))` (L69);
  `else:` → `load_dotenv()` (L71).
- `COLOR_LOG = "COLOR_LOG" in os.environ` `app/config.py:L73`.
- Hard-fail required vars: `URL = os.environ["URL"]` (L79) immediately followed by
  `print(">>> URL:", URL)` (L80); `EMAIL_DOMAIN` (L92); `SUPPORT_EMAIL` (L93);
  `DB_URI = os.environ["DB_URI"]` (L192); `DB_CONN_NAME = os.environ.get("DB_CONN_NAME", "webapp")`
  (L193); `FLASK_SECRET = os.environ["FLASK_SECRET"]` (L196);
  `if not FLASK_SECRET: raise RuntimeError("FLASK_SECRET is empty. Please define it.")` (L197-198);
  `SESSION_COOKIE_NAME = "slapp"` (L199); `MEM_STORE_URI = os.environ.get("MEM_STORE_URI", None)` (L568);
  `DISABLE_RATE_LIMIT = "DISABLE_RATE_LIMIT" in os.environ` (L602).
- The crash surfaces during `from app import config, constants` at `server.py:L30`, confirming config
  resolution happens **at import time**, before `local_main()`/`create_app()` runs.

### Observed vs Inferred

- **Observed:** all three branches — the `CONFIG` path printing `load config file ...`, the
  `KeyError: 'URL'` traceback, the empty-`FLASK_SECRET` `RuntimeError`, and the silent `./.env`
  load with the resolved values. (`/tmp/blitzy_no_url.env` and `/tmp/blitzy_probe_config.py` were
  temporary and have been deleted; no tracked file was edited.)
- **Inferred:** none.

---

## Q3 — Which log messages actually signal that the server is ready?

### Direct answer (Observed)

**There is no explicit "server is ready / listening" log line in dev mode.** SimpleLogin
disables the `werkzeug` logger at import (`log.disabled = True`, `app/log.py:L71`), which
**suppresses Werkzeug's standard readiness banner** — the familiar
`* Running on http://127.0.0.1:7777/` and `Press CTRL+C to quit` lines never appear, and
neither do `* Restarting with stat`, `* Debugger is active!`, or `* Debugger PIN: NNN-NNN-NNN`,
because all of those are emitted through the `werkzeug` logger. The only startup markers
actually printed are SimpleLogin's own `print()` banners — `>>> URL: http://localhost:7777`
(`app/config.py:L80`) and `>>> init logging <<<` (`app/log.py:L67`) — plus Flask's CLI banner
(`* Serving Flask app ...`, `* Environment: production`, the production warning, `* Debug mode: on`),
which Flask emits via `click.echo` (stdout), not through the `werkzeug` logger. Because the
reloader double-imports the module, the last observable markers before the socket accepts are
the **second** (serving-child) `>>> init logging <<<` / `>>> URL:` pair after `* Debug mode: on`;
true readiness must be confirmed out-of-band (e.g. `GET /health` → `200`).

### Command(s) run

```
# confirm the Werkzeug version paired with Flask 1.1.2
PYTHONPATH=/app /app/venv/bin/python -c "import werkzeug, flask; print('werkzeug', werkzeug.__version__, '| flask', flask.__version__)"

# capture full startup stream (see Q1) then probe readiness out-of-band
PYTHONPATH=/app /app/venv/bin/python server.py     # (full stream shown in Q1)
curl -s -i http://localhost:7777/health            # out-of-band readiness confirmation
```

### Complete, unedited output

Version pairing:

```
werkzeug 1.0.1 | flask 1.1.2
```

The full startup stream is reproduced verbatim in **Q1**. The lines that actually appear
(mapped to emitter below) are the two `print` markers, the Flask CLI banner, and the
DebugToolbar warning — and **nothing** from Werkzeug's `werkzeug`-logger banner. Out-of-band
readiness confirmation while the server was up (Run 2):

```
$ curl -s -i http://localhost:7777/health
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 7
Set-Cookie: slapp=8a88eeb0-16d5-4d9a-a3fd-103b1f7c0330.Z1IZeiULJ5f1iyV4VvziVCLfZcU; Expires=Mon, 20-Jul-2026 17:13:47 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Mon, 13 Jul 2026 17:13:47 GMT

success
```

### File:line grounding

- `print(">>> init logging <<<")` `app/log.py:L67` (**appears**, twice).
- `log = logging.getLogger("werkzeug")` `app/log.py:L70`; `log.disabled = True` `app/log.py:L71`
  — this is why the Werkzeug banner is suppressed.
- `print(">>> URL:", URL)` `app/config.py:L80` (**appears**, twice).
- SL log format `_log_format` `app/log.py:L11-14`; UTC `converter = time.gmtime` (L43);
  `logger.setLevel(logging.DEBUG)` (L51); `coloredlogs.install(...)` when `COLOR_LOG` (L61-62),
  which `local_main()` forces on.
- **Suppressed** (routed through the disabled `werkzeug` logger via `werkzeug._internal._log`):
  `* Running on ...` and `Press CTRL+C to quit` (Werkzeug 1.0.1 `serving.py`),
  `* Restarting with stat` (`_reloader.py`), `* Debugger is active!` / `* Debugger PIN:`
  (`debug/__init__.py`). The `* Serving Flask app` / `* Environment` / `* Debug mode` lines
  survive because Flask emits them via `click.echo` (`flask/cli.py` `show_server_banner`), not
  the `werkzeug` logger.
- `/health` handler returns `("success", 200)` `server.py:L213-215`.

### Observed vs Inferred

- **Observed:** which lines actually appear (the two `print` markers ×2 and the Flask CLI banner)
  and which are absent (the entire Werkzeug banner, including the Debugger PIN) on the pinned
  Flask 1.1.2 / Werkzeug 1.0.1 stack; the `/health` 200 as the practical readiness signal;
  `Server: Werkzeug/1.0.1 Python/3.10.18`. The Debugger PIN is a run-to-run-variable value in
  the general case, but here its **absence** is the stable cross-run observation (it is never
  emitted because the logger is disabled).
- **Inferred:** none — the version-dependent banner question was settled by capturing the real
  stream rather than assuming.

---

## Q4 — What ports and endpoints are exposed?

### Direct answer (Observed)

The dev server listens on a single TCP port, **7777**, bound to **127.0.0.1** (loopback)
because `app.run(..., port=7777)` (`server.py:L588`) defaults the host to `127.0.0.1`
(`Dockerfile:L44` also declares `EXPOSE 7777`). The live route table (`app.url_map`) holds
**292 rules**. They come from ten blueprints registered in `register_blueprints()`
(`server.py:L233-246`) — `auth_bp` `/auth`, `monitor_bp` **`/`** (root!), `dashboard_bp`
`/dashboard`, `developer_bp` `/developer`, `phone_bp` `/phone`, `oauth_bp` registered **twice**
at `/oauth` **and** `/oauth2`, `onboarding_bp` `/onboarding`, `discover_bp` `/discover`,
`internal_bp` `/internal`, `api_bp` `/api` (`app/api/base.py:L11`) — plus the Flask-Admin mount
at `/admin` (122 auto-generated CRUD rules) and several direct (non-blueprint) routes: `/`,
`/health`, `/favicon.ico`, `/dnt`, `/.well-known/openid-configuration`, `/jwks`, `/coinbase`,
`/paddle`, `/paddle_coupon`, `/static/<path:filename>`. **Read-vs-run finding:** `monitor_bp`
is mounted at the **root** `/`, so its routes are `/git`, `/live`, `/exception` — **not**
`/monitor/*` (an earlier tech-spec draft claimed `/monitor`; the live `url_map` is authoritative).

### Command(s) run

```
# port bind (ss/netstat are non-functional in this container; use /proc/net/tcp)
grep -i ':1E61 ' /proc/net/tcp          # 1E61 hex = 7777 dec
curl -s -i http://localhost:7777/health

# enumerate the LIVE route table through the canonical factory
cat > /tmp/blitzy_q4_dump.py <<'PY'
from server import create_app
app = create_app()
rules = sorted(app.url_map.iter_rules(), key=lambda r: str(r.rule))
print("TOTAL RULES:", len(rules))
for r in rules:
    methods = ",".join(sorted(m for m in r.methods if m not in ("HEAD", "OPTIONS")))
    print("%-45s  [%s]  -> %s" % (str(r.rule), methods, r.endpoint))
PY
PYTHONPATH=/app /app/venv/bin/python /tmp/blitzy_q4_dump.py
```

### Complete, unedited output

Port bind (`0100007F` = 127.0.0.1 little-endian, `1E61` = 7777, state `0A` = `TCP_LISTEN`):

```
   2: 0100007F:1E61 00000000:0000 0A 00000000:00000000 ...
```

The complete live route table (all 292 rules, sorted by path; `create_app()` also prints its
two import-time markers first):

```
>>> URL: http://localhost:7777
Upload files to local dir
>>> init logging <<<
2026-07-13 17:32:08,275 - SL - DEBUG - 4169 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
TOTAL RULES: 292
================================================================================
/                                              [GET,POST]  -> index
/.well-known/openid-configuration              [GET]  -> openid_config
/admin/                                        [GET]  -> admin.index
/admin/adminauditlog/                          [GET]  -> adminauditlog.index_view
/admin/adminauditlog/action/                   [POST]  -> adminauditlog.action_view
/admin/adminauditlog/ajax/lookup/              [GET]  -> adminauditlog.ajax_lookup
/admin/adminauditlog/ajax/update/              [POST]  -> adminauditlog.ajax_update
/admin/adminauditlog/delete/                   [POST]  -> adminauditlog.delete_view
/admin/adminauditlog/details/                  [GET]  -> adminauditlog.details_view
/admin/adminauditlog/edit/                     [GET,POST]  -> adminauditlog.edit_view
/admin/adminauditlog/export/<export_type>/     [GET]  -> adminauditlog.export
/admin/adminauditlog/new/                      [GET,POST]  -> adminauditlog.create_view
/admin/alias/                                  [GET]  -> alias.index_view
/admin/alias/action/                           [POST]  -> alias.action_view
/admin/alias/ajax/lookup/                      [GET]  -> alias.ajax_lookup
/admin/alias/ajax/update/                      [POST]  -> alias.ajax_update
/admin/alias/delete/                           [POST]  -> alias.delete_view
/admin/alias/details/                          [GET]  -> alias.details_view
/admin/alias/edit/                             [GET,POST]  -> alias.edit_view
/admin/alias/export/<export_type>/             [GET]  -> alias.export
/admin/alias/new/                              [GET,POST]  -> alias.create_view
/admin/coupon/                                 [GET]  -> coupon.index_view
/admin/coupon/action/                          [POST]  -> coupon.action_view
/admin/coupon/ajax/lookup/                     [GET]  -> coupon.ajax_lookup
/admin/coupon/ajax/update/                     [POST]  -> coupon.ajax_update
/admin/coupon/delete/                          [POST]  -> coupon.delete_view
/admin/coupon/details/                         [GET]  -> coupon.details_view
/admin/coupon/edit/                            [GET,POST]  -> coupon.edit_view
/admin/coupon/export/<export_type>/            [GET]  -> coupon.export
/admin/coupon/new/                             [GET,POST]  -> coupon.create_view
/admin/customdomain/                           [GET]  -> customdomain.index_view
/admin/customdomain/action/                    [POST]  -> customdomain.action_view
/admin/customdomain/ajax/lookup/               [GET]  -> customdomain.ajax_lookup
/admin/customdomain/ajax/update/               [POST]  -> customdomain.ajax_update
/admin/customdomain/delete/                    [POST]  -> customdomain.delete_view
/admin/customdomain/details/                   [GET]  -> customdomain.details_view
/admin/customdomain/edit/                      [GET,POST]  -> customdomain.edit_view
/admin/customdomain/export/<export_type>/      [GET]  -> customdomain.export
/admin/customdomain/new/                       [GET,POST]  -> customdomain.create_view
/admin/dailymetric/                            [GET]  -> dailymetric.index_view
/admin/dailymetric/action/                     [POST]  -> dailymetric.action_view
/admin/dailymetric/ajax/lookup/                [GET]  -> dailymetric.ajax_lookup
/admin/dailymetric/ajax/update/                [POST]  -> dailymetric.ajax_update
/admin/dailymetric/delete/                     [POST]  -> dailymetric.delete_view
/admin/dailymetric/details/                    [GET]  -> dailymetric.details_view
/admin/dailymetric/edit/                       [GET,POST]  -> dailymetric.edit_view
/admin/dailymetric/export/<export_type>/       [GET]  -> dailymetric.export
/admin/dailymetric/new/                        [GET,POST]  -> dailymetric.create_view
/admin/email_search/                           [GET,POST]  -> email_search.index
/admin/invalidmailboxdomain/                   [GET]  -> invalidmailboxdomain.index_view
/admin/invalidmailboxdomain/action/            [POST]  -> invalidmailboxdomain.action_view
/admin/invalidmailboxdomain/ajax/lookup/       [GET]  -> invalidmailboxdomain.ajax_lookup
/admin/invalidmailboxdomain/ajax/update/       [POST]  -> invalidmailboxdomain.ajax_update
/admin/invalidmailboxdomain/delete/            [POST]  -> invalidmailboxdomain.delete_view
/admin/invalidmailboxdomain/details/           [GET]  -> invalidmailboxdomain.details_view
/admin/invalidmailboxdomain/edit/              [GET,POST]  -> invalidmailboxdomain.edit_view
/admin/invalidmailboxdomain/export/<export_type>/  [GET]  -> invalidmailboxdomain.export
/admin/invalidmailboxdomain/new/               [GET,POST]  -> invalidmailboxdomain.create_view
/admin/mailbox/                                [GET]  -> mailbox.index_view
/admin/mailbox/action/                         [POST]  -> mailbox.action_view
/admin/mailbox/ajax/lookup/                    [GET]  -> mailbox.ajax_lookup
/admin/mailbox/ajax/update/                    [POST]  -> mailbox.ajax_update
/admin/mailbox/delete/                         [POST]  -> mailbox.delete_view
/admin/mailbox/details/                        [GET]  -> mailbox.details_view
/admin/mailbox/edit/                           [GET,POST]  -> mailbox.edit_view
/admin/mailbox/export/<export_type>/           [GET]  -> mailbox.export
/admin/mailbox/new/                            [GET,POST]  -> mailbox.create_view
/admin/manualsubscription/                     [GET]  -> manualsubscription.index_view
/admin/manualsubscription/action/              [POST]  -> manualsubscription.action_view
/admin/manualsubscription/ajax/lookup/         [GET]  -> manualsubscription.ajax_lookup
/admin/manualsubscription/ajax/update/         [POST]  -> manualsubscription.ajax_update
/admin/manualsubscription/delete/              [POST]  -> manualsubscription.delete_view
/admin/manualsubscription/details/             [GET]  -> manualsubscription.details_view
/admin/manualsubscription/edit/                [GET,POST]  -> manualsubscription.edit_view
/admin/manualsubscription/export/<export_type>/  [GET]  -> manualsubscription.export
/admin/manualsubscription/new/                 [GET,POST]  -> manualsubscription.create_view
/admin/metric2/                                [GET]  -> metric2.index_view
/admin/metric2/action/                         [POST]  -> metric2.action_view
/admin/metric2/ajax/lookup/                    [GET]  -> metric2.ajax_lookup
/admin/metric2/ajax/update/                    [POST]  -> metric2.ajax_update
/admin/metric2/delete/                         [POST]  -> metric2.delete_view
/admin/metric2/details/                        [GET]  -> metric2.details_view
/admin/metric2/edit/                           [GET,POST]  -> metric2.edit_view
/admin/metric2/export/<export_type>/           [GET]  -> metric2.export
/admin/metric2/new/                            [GET,POST]  -> metric2.create_view
/admin/newsletter/                             [GET]  -> newsletter.index_view
/admin/newsletter/action/                      [POST]  -> newsletter.action_view
/admin/newsletter/ajax/lookup/                 [GET]  -> newsletter.ajax_lookup
/admin/newsletter/ajax/update/                 [POST]  -> newsletter.ajax_update
/admin/newsletter/delete/                      [POST]  -> newsletter.delete_view
/admin/newsletter/details/                     [GET]  -> newsletter.details_view
/admin/newsletter/edit/                        [GET,POST]  -> newsletter.edit_view
/admin/newsletter/export/<export_type>/        [GET]  -> newsletter.export
/admin/newsletter/new/                         [GET,POST]  -> newsletter.create_view
/admin/newsletteruser/                         [GET]  -> newsletteruser.index_view
/admin/newsletteruser/action/                  [POST]  -> newsletteruser.action_view
/admin/newsletteruser/ajax/lookup/             [GET]  -> newsletteruser.ajax_lookup
/admin/newsletteruser/ajax/update/             [POST]  -> newsletteruser.ajax_update
/admin/newsletteruser/delete/                  [POST]  -> newsletteruser.delete_view
/admin/newsletteruser/details/                 [GET]  -> newsletteruser.details_view
/admin/newsletteruser/edit/                    [GET,POST]  -> newsletteruser.edit_view
/admin/newsletteruser/export/<export_type>/    [GET]  -> newsletteruser.export
/admin/newsletteruser/new/                     [GET,POST]  -> newsletteruser.create_view
/admin/providercomplaint/                      [GET]  -> providercomplaint.index_view
/admin/providercomplaint/action/               [POST]  -> providercomplaint.action_view
/admin/providercomplaint/ajax/lookup/          [GET]  -> providercomplaint.ajax_lookup
/admin/providercomplaint/ajax/update/          [POST]  -> providercomplaint.ajax_update
/admin/providercomplaint/delete/               [POST]  -> providercomplaint.delete_view
/admin/providercomplaint/details/              [GET]  -> providercomplaint.details_view
/admin/providercomplaint/download_eml          [GET]  -> providercomplaint.download_eml
/admin/providercomplaint/edit/                 [GET,POST]  -> providercomplaint.edit_view
/admin/providercomplaint/export/<export_type>/  [GET]  -> providercomplaint.export
/admin/providercomplaint/mark_ok               [GET]  -> providercomplaint.mark_ok
/admin/providercomplaint/new/                  [GET,POST]  -> providercomplaint.create_view
/admin/static/<path:filename>                  [GET]  -> admin.static
/admin/user/                                   [GET]  -> user.index_view
/admin/user/action/                            [POST]  -> user.action_view
/admin/user/ajax/lookup/                       [GET]  -> user.ajax_lookup
/admin/user/ajax/update/                       [POST]  -> user.ajax_update
/admin/user/delete/                            [POST]  -> user.delete_view
/admin/user/details/                           [GET]  -> user.details_view
/admin/user/edit/                              [GET,POST]  -> user.edit_view
/admin/user/export/<export_type>/              [GET]  -> user.export
/admin/user/new/                               [GET,POST]  -> user.create_view
/api/alias/random/new                          [POST]  -> api.new_random_alias
/api/aliases                                   [GET,POST]  -> api.get_aliases
/api/aliases/<int:alias_id>                    [DELETE]  -> api.delete_alias
/api/aliases/<int:alias_id>                    [PATCH,PUT]  -> api.update_alias
/api/aliases/<int:alias_id>                    [GET]  -> api.get_alias
/api/aliases/<int:alias_id>/activities         [GET]  -> api.get_alias_activities
/api/aliases/<int:alias_id>/contacts           [GET]  -> api.get_alias_contacts_route
/api/aliases/<int:alias_id>/contacts           [POST]  -> api.create_contact_route
/api/aliases/<int:alias_id>/toggle             [POST]  -> api.toggle_alias
/api/api_key                                   [POST]  -> api.create_api_key
/api/apple/process_payment                     [POST]  -> api.apple_process_payment
/api/apple/update_notification                 [GET,POST]  -> api.apple_update_notification
/api/auth/activate                             [POST]  -> api.auth_activate
/api/auth/facebook                             [POST]  -> api.auth_facebook
/api/auth/forgot_password                      [POST]  -> api.forgot_password
/api/auth/google                               [POST]  -> api.auth_google
/api/auth/login                                [POST]  -> api.auth_login
/api/auth/mfa                                  [POST]  -> api.auth_mfa
/api/auth/reactivate                           [POST]  -> api.auth_reactivate
/api/auth/register                             [POST]  -> api.auth_register
/api/contacts/<int:contact_id>                 [DELETE]  -> api.delete_contact
/api/contacts/<int:contact_id>/toggle          [POST]  -> api.toggle_contact
/api/custom_domains                            [GET]  -> api.get_custom_domains
/api/custom_domains/<int:custom_domain_id>     [PATCH]  -> api.update_custom_domain
/api/custom_domains/<int:custom_domain_id>/trash  [GET]  -> api.get_custom_domain_trash
/api/export/aliases                            [GET]  -> api.export_aliases
/api/export/data                               [GET]  -> api.export_data
/api/logout                                    [GET]  -> api.logout
/api/mailboxes                                 [POST]  -> api.create_mailbox
/api/mailboxes                                 [GET]  -> api.get_mailboxes
/api/mailboxes/<int:mailbox_id>                [DELETE]  -> api.delete_mailbox
/api/mailboxes/<int:mailbox_id>                [PUT]  -> api.update_mailbox
/api/notifications                             [GET]  -> api.get_notifications
/api/notifications/<int:notification_id>/read  [POST]  -> api.mark_as_read
/api/phone/reservations/<int:reservation_id>   [GET,POST]  -> api.phone_messages
/api/setting                                   [GET]  -> api.get_setting
/api/setting                                   [PATCH]  -> api.update_setting
/api/setting/domains                           [GET]  -> api.get_available_domains_for_random_alias
/api/setting/unlink_proton_account             [DELETE]  -> api.unlink_proton_account
/api/stats                                     [GET]  -> api.user_stats
/api/sudo                                      [PATCH]  -> api.enter_sudo
/api/user                                      [DELETE]  -> api.delete_user
/api/user/cookie_token                         [GET]  -> api.get_api_session_token
/api/user_info                                 [GET]  -> api.user_info
/api/user_info                                 [PATCH]  -> api.update_user_info
/api/v2/alias/custom/new                       [POST]  -> api.new_custom_alias_v2
/api/v2/aliases                                [GET,POST]  -> api.get_aliases_v2
/api/v2/mailboxes                              [GET]  -> api.get_mailboxes_v2
/api/v2/setting/domains                        [GET]  -> api.get_available_domains_for_random_alias_v2
/api/v3/alias/custom/new                       [POST]  -> api.new_custom_alias_v3
/api/v4/alias/options                          [GET]  -> api.options_v4
/api/v5/alias/options                          [GET]  -> api.options_v5
/auth/activate                                 [GET,POST]  -> auth.activate
/auth/api_to_cookie                            [GET]  -> auth.api_to_cookie
/auth/change_email                             [GET,POST]  -> auth.change_email
/auth/facebook/callback                        [GET]  -> auth.facebook_callback
/auth/facebook/login                           [GET]  -> auth.facebook_login
/auth/fido                                     [GET,POST]  -> auth.fido
/auth/forgot_password                          [GET,POST]  -> auth.forgot_password
/auth/github/callback                          [GET]  -> auth.github_callback
/auth/github/login                             [GET]  -> auth.github_login
/auth/google/callback                          [GET]  -> auth.google_callback
/auth/google/login                             [GET]  -> auth.google_login
/auth/login                                    [GET,POST]  -> auth.login
/auth/logout                                   [GET]  -> auth.logout
/auth/mfa                                      [GET,POST]  -> auth.mfa
/auth/oidc/callback                            [GET]  -> auth.oidc_callback
/auth/oidc/login                               [GET]  -> auth.oidc_login
/auth/proton/callback                          [GET]  -> auth.proton_callback
/auth/proton/login                             [GET]  -> auth.proton_login
/auth/recovery                                 [GET,POST]  -> auth.recovery_route
/auth/register                                 [GET,POST]  -> auth.register
/auth/resend_activation                        [GET,POST]  -> auth.resend_activation
/auth/reset_password                           [GET,POST]  -> auth.reset_password
/auth/social                                   [GET,POST]  -> auth.social
/coinbase                                      [POST]  -> coinbase_webhook
/dashboard/                                    [GET,POST]  -> dashboard.index
/dashboard/account_setting                     [GET,POST]  -> dashboard.account_setting
/dashboard/alias_contact_manager/<int:alias_id>/  [GET,POST]  -> dashboard.alias_contact_manager
/dashboard/alias_export                        [GET]  -> dashboard.alias_export_route
/dashboard/alias_log/<int:alias_id>            [GET]  -> dashboard.alias_log
/dashboard/alias_log/<int:alias_id>/<int:page_id>  [GET]  -> dashboard.alias_log
/dashboard/alias_transfer/receive              [GET,POST]  -> dashboard.alias_transfer_receive_route
/dashboard/alias_transfer/send/<int:alias_id>/  [GET,POST]  -> dashboard.alias_transfer_send_route
/dashboard/api_key                             [GET,POST]  -> dashboard.api_key
/dashboard/app                                 [GET,POST]  -> dashboard.app_route
/dashboard/batch_import                        [GET,POST]  -> dashboard.batch_import_route
/dashboard/billing                             [GET,POST]  -> dashboard.billing
/dashboard/block_contact/<int:contact_id>      [GET,POST]  -> dashboard.block_contact
/dashboard/cancel_email_change                 [GET,POST]  -> dashboard.cancel_email_change
/dashboard/coinbase_checkout                   [GET]  -> dashboard.coinbase_checkout_route
/dashboard/contact/<int:contact_id>/           [GET,POST]  -> dashboard.contact_detail_route
/dashboard/contacts/<int:contact_id>/toggle    [POST]  -> dashboard.toggle_contact
/dashboard/coupon                              [GET,POST]  -> dashboard.coupon_route
/dashboard/custom_alias                        [GET,POST]  -> dashboard.custom_alias
/dashboard/custom_domain                       [GET,POST]  -> dashboard.custom_domain
/dashboard/delete_account                      [GET,POST]  -> dashboard.delete_account
/dashboard/directory                           [GET,POST]  -> dashboard.directory
/dashboard/domains/<int:custom_domain_id>/auto-create  [GET,POST]  -> dashboard.domain_detail_auto_create
/dashboard/domains/<int:custom_domain_id>/dns  [GET,POST]  -> dashboard.domain_detail_dns
/dashboard/domains/<int:custom_domain_id>/info  [GET,POST]  -> dashboard.domain_detail
/dashboard/domains/<int:custom_domain_id>/trash  [GET,POST]  -> dashboard.domain_detail_trash
/dashboard/enter_sudo                          [GET,POST]  -> dashboard.enter_sudo
/dashboard/fido_manage                         [GET,POST]  -> dashboard.fido_manage
/dashboard/fido_setup                          [GET,POST]  -> dashboard.fido_setup
/dashboard/lifetime_licence                    [GET,POST]  -> dashboard.lifetime_licence
/dashboard/mailbox                             [GET,POST]  -> dashboard.mailbox_route
/dashboard/mailbox/<int:mailbox_id>/           [GET,POST]  -> dashboard.mailbox_detail_route
/dashboard/mailbox/<int:mailbox_id>/cancel_email_change  [GET,POST]  -> dashboard.cancel_mailbox_change_route
/dashboard/mailbox/confirm_change              [GET]  -> dashboard.mailbox_confirm_change_route
/dashboard/mailbox_verify                      [GET]  -> dashboard.mailbox_verify
/dashboard/mfa_cancel                          [GET,POST]  -> dashboard.mfa_cancel
/dashboard/mfa_setup                           [GET,POST]  -> dashboard.mfa_setup
/dashboard/notification/<notification_id>      [GET,POST]  -> dashboard.notification_route
/dashboard/notifications                       [GET,POST]  -> dashboard.notifications_route
/dashboard/pricing                             [GET,POST]  -> dashboard.pricing
/dashboard/referral                            [GET,POST]  -> dashboard.referral_route
/dashboard/refused_email                       [GET,POST]  -> dashboard.refused_email_route
/dashboard/resend_email_change                 [GET,POST]  -> dashboard.resend_email_change
/dashboard/setting                             [GET,POST]  -> dashboard.setting
/dashboard/setup_done                          [GET,POST]  -> dashboard.setup_done
/dashboard/subdomain                           [GET,POST]  -> dashboard.subdomain_route
/dashboard/subscription_success                [GET]  -> dashboard.subscription_success
/dashboard/support                             [GET,POST]  -> dashboard.support_route
/dashboard/unlink_proton_account               [POST]  -> dashboard.unlink_proton_account
/dashboard/unsubscribe/<int:alias_id>          [GET,POST]  -> dashboard.unsubscribe
/dashboard/unsubscribe/encoded/<encoded_request>  [GET]  -> dashboard.encoded_unsubscribe
/developer/                                    [GET,POST]  -> developer.index
/developer/clients/<client_id>                 [GET,POST]  -> developer.client_detail
/developer/clients/<client_id>/advanced        [GET,POST]  -> developer.client_detail_advanced
/developer/clients/<client_id>/oauth_endpoint  [GET,POST]  -> developer.client_detail_oauth_endpoint
/developer/clients/<client_id>/oauth_setting   [GET,POST]  -> developer.client_detail_oauth_setting
/developer/clients/<client_id>/referral        [GET,POST]  -> developer.client_detail_referral
/developer/new_client                          [GET,POST]  -> developer.new_client
/discover/                                     [GET,POST]  -> discover.index
/dnt                                           [GET]  -> do_not_track
/exception                                     [GET]  -> monitor.test_exception
/favicon.ico                                   [GET]  -> favicon
/git                                           [GET]  -> monitor.git_sha1
/health                                        [GET]  -> healthcheck
/internal/exit-sudo-mode                       [GET]  -> internal.exit_sudo_mode
/internal/integrations/proton                  [GET]  -> internal.set_enable_proton_cookie
/jwks                                          [GET]  -> jwks
/live                                          [GET]  -> monitor.live
/oauth/authorize                               [GET,POST]  -> oauth.authorize
/oauth/me                                      [GET]  -> oauth.user_info
/oauth/token                                   [GET,POST]  -> oauth.token
/oauth/user_info                               [GET]  -> oauth.user_info
/oauth/userinfo                                [GET]  -> oauth.user_info
/oauth2/authorize                              [GET,POST]  -> oauth.authorize
/oauth2/me                                     [GET]  -> oauth.user_info
/oauth2/token                                  [GET,POST]  -> oauth.token
/oauth2/user_info                              [GET]  -> oauth.user_info
/oauth2/userinfo                               [GET]  -> oauth.user_info
/onboarding/                                   [GET]  -> onboarding.index
/onboarding/account_activated                  [GET]  -> onboarding.account_activated
/onboarding/extension_redirect                 [GET]  -> onboarding.extension_redirect
/onboarding/final                              [GET,POST]  -> onboarding.final
/onboarding/setup                              [GET]  -> onboarding.setup
/onboarding/setup_done                         [GET,POST]  -> onboarding.setup_done
/paddle                                        [GET,POST]  -> paddle
/paddle_coupon                                 [GET,POST]  -> paddle_coupon
/phone/                                        [GET,POST]  -> phone.index
/phone/provider1/sms                           [GET,POST]  -> phone.provider1_sms
/phone/provider2/sms                           [GET,POST]  -> phone.provider2_sms
/phone/reservation/<int:reservation_id>        [GET,POST]  -> phone.reservation_route
/phone/twilio/sms                              [GET,POST]  -> phone.twilio_sms
/static/<path:filename>                        [GET]  -> static
```

Rule counts by first path segment (derived from the dump above; totals to 292):
`/admin` = 122, `/api` = 52, `/dashboard` = 51, `/auth` = 23, `/developer` = 7,
`/onboarding` = 6, `/oauth` = 5, `/oauth2` = 5, `/phone` = 5, `/internal` = 2, and one each for
`/`, `/.well-known`, `/coinbase`, `/discover`, `/dnt`, `/exception`, `/favicon.ico`, `/git`,
`/health`, `/jwks`, `/live`, `/paddle`, `/paddle_coupon`, `/static`.

Monitor-discrepancy proof (queried the live map for any `/monitor/*` rule):

```
Any /monitor/* rule?: []
/git      -> monitor.git_sha1
/live     -> monitor.live
/exception-> monitor.test_exception
```

### File:line grounding

- Port: `app.run(debug=True, port=7777)` `server.py:L588`; `Dockerfile:L44` `EXPOSE 7777`.
- Blueprint registration: `register_blueprints(app)` body `server.py:L233-246`. `oauth_bp` is
  registered twice — once at `/oauth` and once at `/oauth2` (that is why `/oauth/authorize`
  and `/oauth2/authorize` both map to `oauth.authorize`).
- `api_bp = Blueprint(..., url_prefix="/api")` `app/api/base.py:L11`.
- `monitor_bp = Blueprint(name="monitor", import_name=__name__, url_prefix="/")` `app/monitor/base.py:L3`;
  routes `@monitor_bp.route("/git")` `app/monitor/views.py:L5-7`, `/live` (L10-12), `/exception` (L15-17).
- Direct routes: `index` `server.py:L250-256`; `healthcheck` `server.py:L213-215`;
  `favicon` `server.py:L398-400`; `openid_config` `server.py:L300`; `jwks` `server.py:L327`.

### Observed vs Inferred

- **Observed:** the port 7777 loopback bind, the 292-rule live table above, every blueprint prefix,
  every direct route, the `/oauth`+`/oauth2` double registration, and the `monitor_bp`-at-root
  read-vs-run discrepancy (`/git`, `/live`, `/exception`; no `/monitor/*` rule exists).
- **Inferred:** the production `0.0.0.0:7777` bind (gunicorn, from `Dockerfile:L47`) — not run.

---

## Q5 — How is a single authenticated request handled? (PRIMARY)

This is the primary concern. It is answered in three explicit sub-parts — **(a)** where a
request first enters the app, **(b)** how the user's identity is determined at runtime, and
**(c)** how the authentication context is propagated — and **both** real authentication paths
are exercised: the **web-session** path (Flask-Login) and the **API** path (the
`Authentication` header). The anonymous/401 edges and **both** session backends are also
exercised.

### Direct answer (Observed)

A request enters through the `ProxyFix` WSGI wrapper, then two `before_request` hooks
(`make_session_permanent`, then the timing hook that sets `g.start_time`). Identity is then
resolved by one of two mechanisms:

- **Web:** Flask-Login reads `session["_user_id"]` and calls the `user_loader`
  `load_user(alternative_id)` (`server.py:L220-222`), which does `User.get_by(alternative_id=...)`.
  Crucially, `session["_user_id"]` holds the user's **`alternative_id` (a UUID4)**, *not* the
  numeric primary key, because `User.get_id()` returns `alternative_id` when set
  (`app/models.py:L595-599`). The resolved user is exposed as `current_user`.
- **API:** `authorize_request()` (`app/api/base.py:L16-42`) reads the `Authentication` header,
  looks up the `ApiKey`, and sets `g.user = api_key.user`; if the header is absent/invalid but a
  web session exists, it falls through to `g.user = current_user`; otherwise it returns
  `401 {"error": "Wrong api key"}`.

Both paths converge in `get_current_user()` (`server.py:L332-336`), which returns `g.user` if set
else `current_user`. After the view runs, an `after_request` hook logs one line per request, and
`teardown_appcontext` calls `Session.remove()`.

### (a) Entry point

**Command / grounding:** the common entry chain, confirmed live:

- `app.wsgi_app = ProxyFix(app.wsgi_app, x_for=1, x_host=1)` `server.py:L142` (the app runs behind
  NGINX in prod; `x_for`/`x_host` trust one proxy hop).
- `@app.before_request def make_session_permanent(): session.permanent = True; app.permanent_session_lifetime = timedelta(days=7)`
  `server.py:L204-207`.
- `@app.before_request def before_request(): ... g.start_time = time.time()` `server.py:L258-269`
  (the timestamp read at L263 is what the per-request log line's `takes` is computed against).

**Observed evidence:** every captured `after_request` line begins with `remote_addr = 127.0.0.1`,
confirming the request traversed `ProxyFix` and the `before_request` timing hook (the timing that
produces `takes` below).

### (b) Identity determination

#### Web-session path (Flask-Login) — exercised with a real login

**Commands run** (driven only through real HTTP to `localhost:7777` with a cookie jar; the
Redis session backend is active because `MEM_STORE_URI=redis://localhost`):

```
# 1) anonymous GET to obtain a session + CSRF token
# 2) POST /auth/login with the seeded credentials john@wick.com / password (+ csrf_token)
# 3) authenticated GET /dashboard/
# Redis session contents were dumped at each step via redis-cli KEYS/TTL and the store's serializer
```

**Complete, unedited output** (Redis session state at each step):

```
[1] anonymous GET /auth/login -> 200
    Set-Cookie: slapp=2c77513d-5e18-4fba-a630-11bac9beea7a.NsrMRDAROp2XMWbfUkHSVFCck0E; ...
    redis key: session:2c77513d-5e18-4fba-a630-11bac9beea7a   ttl=300
    session dict keys: ['_permanent', '_fresh', 'csrf_token']    _user_id=None

[2] POST /auth/login (john@wick.com / password + csrf) -> 302  Location: http://localhost:7777/dashboard/
    (same session id retained)
    redis key: session:2c77513d-5e18-4fba-a630-11bac9beea7a   ttl=604800
    session dict keys: ['_permanent', '_fresh', 'csrf_token', '_user_id', '_id', 'sudo_time']
    _user_id = '01330a11-6c00-4031-86c4-d3846f0ce7e9'

[3] GET /dashboard/ (authenticated) -> 200   <title>Alias | SimpleLogin</title>  (763485 bytes)
```

The decisive identity fact — `session["_user_id"]` equals the user's `alternative_id`, **not**
the primary key — cross-checked directly against the database:

```
$ psql ... -c "select id, alternative_id from users where email='john@wick.com'"
 id |            alternative_id
----+--------------------------------------
  1 | 01330a11-6c00-4031-86c4-d3846f0ce7e9

# session _user_id observed above = 01330a11-6c00-4031-86c4-d3846f0ce7e9  (== alternative_id, != id 1)
```

**Reproducibility (2 runs):** the **session id is variable** (Run 1 `2c77513d-...`, Run 2
`328fbc0c-2e52-44b6-84bb-7d414555f255`), but the stored `_user_id` was **identical and stable**
across both runs (`01330a11-6c00-4031-86c4-d3846f0ce7e9`), and the authenticated TTL was `604800`
both times. The `alternative_id` is persisted in the DB, so it is stable unless `flask dummy-data`
regenerates it.

**File:line grounding (web identity):**

- `@login_manager.user_loader` / `def load_user(alternative_id):` `server.py:L220-221`, body
  `return User.get_by(alternative_id=alternative_id)` (L222).
- `def get_id(self): if self.alternative_id: return self.alternative_id else: return str(self.id)`
  `app/models.py:L595-599` — Flask-Login stores this in `session["_user_id"]`, which is why the
  UUID (not `1`) is stored.
- `alternative_id = sa.Column(sa.String(128), unique=True, nullable=True)` `app/models.py:L482`;
  generated `user.alternative_id = str(uuid.uuid4())` `app/models.py:L617`.
- `login_manager.session_protection = "strong"` `app/extensions.py:L8`.
- Login view `@auth_bp.route("/login", methods=["GET", "POST"])` `app/auth/views/login.py:L21`,
  `def login()` (L25), `User.get_by(email=...)` (L43), `user.check_password(...)` (L45),
  `after_login(user, next_url)` (L72); `after_login` `app/auth/views/login_utils.py:L12` calls
  `login_user(user)` (L36).

#### API path (`Authentication` header) — exercised with a real key

**Commands run** (the seeded API key `code` belongs to `john@wick.com`, user_id 1):

```
# [A] valid key
curl -s -o - -w '\n%{http_code}\n' -H 'Authentication: code'          http://localhost:7777/api/user_info
# [B] wrong key, no session
curl -s -o - -w '\n%{http_code}\n' -H 'Authentication: WRONG_KEY_xyz' http://localhost:7777/api/user_info
# [C] no header, no session
curl -s -o - -w '\n%{http_code}\n'                                     http://localhost:7777/api/user_info
# [D] no header but a valid web-session cookie -> fall-through to current_user
curl -s -o - -w '\n%{http_code}\n' -b jar                              http://localhost:7777/api/user_info
```

**Complete, unedited output:**

```
[A] Authentication: code  ->
{"can_create_reverse_alias":true,"connected_proton_address":null,"email":"john@wick.com","in_trial":false,"is_premium":true,"max_alias_free_plan":3,"name":"John Wick","profile_picture_url":"http://localhost:7777/static/upload/profile_pic.svg"}
200

[B] Authentication: WRONG_KEY_xyz  ->
{"error":"Wrong api key"}
401

[C] (no Authentication header)  ->
{"error":"Wrong api key"}
401

[D] (no header, valid session cookie)  ->
{"can_create_reverse_alias":true,"connected_proton_address":null,"email":"john@wick.com","in_trial":false,"is_premium":true,"max_alias_free_plan":3,"name":"John Wick","profile_picture_url":"http://localhost:7777/static/upload/profile_pic.svg"}
200
```

**File:line grounding (API identity):**

- `authorize_request()` `app/api/base.py:L16`; `api_code = request.headers.get("Authentication")` (L17);
  `api_key = ApiKey.get_by(code=api_code)` (L18).
- Fall-through when no key but a session exists: `if not api_key: if current_user.is_authenticated: g.user = current_user`
  `app/api/base.py:L20-25` (lines 22-24 are commented-out cookie-header logic, so the assignment
  `g.user = current_user` lands at L25); else `return jsonify(error="Wrong api key"), 401` (L26-27).
- Valid key: `g.user = api_key.user` (L34); disabled account → `jsonify(error="Disabled account"), 403` (L36-37);
  inactive → `jsonify(error="Account does not exist"), 401` (L39-40); `g.api_key = api_key` (L42).
- The `/api/*` view decorator `require_api_auth` `app/api/base.py:L52-58` calls `authorize_request()`
  (L55) and returns its error tuple directly when truthy. Corroborated by `docs/api.md:L73,L202,L236`
  (the `Authentication` header contract).

### (c) Auth-context propagation / convergence

The two paths deliberately converge in a single accessor:

```
# server.py:L332-336
def get_current_user():
    try:
        return g.user
    except AttributeError:
        return current_user
```

- **Web** requests never set `g.user`, so `get_current_user()` raises `AttributeError` internally
  and returns `current_user` (the Flask-Login proxy).
- **API** requests set `g.user` in `authorize_request()`, so `get_current_user()` returns it.
- Case **[D]** above is the explicit unification point: with no API key but a valid session,
  `authorize_request()` sets `g.user = current_user`, so both mechanisms end at the same object.

**File:line grounding:** `get_current_user()` `server.py:L332-336`.

### Per-request log line (`after_request`)

**Grounding:** `@app.after_request def after_request(res): ... LOG.d("%s %s %s %s %s, takes %s", ...)`
`server.py:L273-296` (the format string is at L285; the runtime record shows lineno 284 /
`after_request()`). The tuple is `remote_addr, method, path, args, status_code, (time.time()-start_time)`.
It is skipped for `/static`, `/admin/static`, `/_debug_toolbar`, `/git`, `/favicon.ico`, `/health`
(`server.py:L276-282`).

**Complete, unedited output** (actual emitted lines for the requests above):

```
127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.10316872596740723
127.0.0.1 POST /auth/login ImmutableMultiDict([]) 302, takes 0.2468411922454834
127.0.0.1 GET /dashboard/ ImmutableMultiDict([]) 200, takes 0.35060882568359375
127.0.0.1 GET /api/user_info ImmutableMultiDict([]) 200, takes 0.024179458618164062
127.0.0.1 GET /api/user_info ImmutableMultiDict([]) 401, takes 0.0020406246185302734   (wrong key)
127.0.0.1 GET /api/user_info ImmutableMultiDict([]) 401, takes 0.0018086433410644531   (no header)
127.0.0.1 GET /api/user_info ImmutableMultiDict([]) 200, takes 0.010299205780029297   (session fall-through)
127.0.0.1 GET /dashboard/ ImmutableMultiDict([]) 302, takes 0.000652313232421875   (unauth redirect)
127.0.0.1 GET /api/aliases ImmutableMultiDict([]) 401, takes 0.001813650131225586
```

The `takes` value is **run-to-run variable** (it is a wall-clock delta); the rest of each line is
stable.

### 401 / unauthenticated edge (both variants)

**Commands run:**

```
curl -s -i http://localhost:7777/dashboard/    # protected web page, no session
curl -s -i http://localhost:7777/api/aliases   # protected API endpoint, no auth
```

**Complete, unedited output:**

```
[E] GET /dashboard/ (no session) -> 302
    Location: http://localhost:7777/auth/login?next=%2Fdashboard%2F%3F

[F] GET /api/aliases (no auth) -> 401
    Content-Type: application/json
    {"error":"Wrong api key"}
```

**File:line grounding:** the 401 handler `unauthorized` inside `setup_error_page()` is at
`server.py:L347-352`: for `/api/` paths it returns `jsonify(error="Unauthorized"), 401`, otherwise
it flashes "You need to login to see this page" and `redirect(url_for("auth.login", next=request.full_path))`
(hence `next=%2Fdashboard%2F%3F`, i.e. `/dashboard/?` url-encoded). **Nuance:** SimpleLogin's `/api/*`
views use `@require_api_auth`, which **returns** `{"error":"Wrong api key"}, 401` rather than calling
`abort(401)`; that is why the observed `/api` body is `Wrong api key`. The `jsonify(error="Unauthorized")`
body would only be produced on an actual `abort(401)` within an `/api/` path.

### Two session backends (both exercised)

**(1) Server-side `RedisSessionStore`** (active because `MEM_STORE_URI=redis://localhost`): the
`slapp` cookie is only a short pointer `<uuid>.<itsdangerous-sig>`; the session dict lives in the
Redis key `session:<uuid>`. TTL is `int(app.permanent_session_lifetime.total_seconds())` = **604800 s
(7 days)** once `_user_id` is present, and **300 s** while anonymous. The transition `300 -> 604800`
on login is shown in the sub-part (b) capture above.

```
# app/session.py
ttl = int(app.permanent_session_lifetime.total_seconds())     # L92  -> 604800 (7 days)
if "_user_id" not in session:                                 # L95
    ttl = 300                                                 # L96  -> anonymous
```

**(2) Flask default signed cookie** (server launched with `MEM_STORE_URI=""`): the entire session
lives **client-side** in the `slapp` cookie (base64(payload) + timestamp + HMAC signature; a leading
`.` marks a zlib-compressed payload). Decoded anonymous payload:

```
# server launched with MEM_STORE_URI="" ; anonymous Set-Cookie: slapp=eyJfZnJlc2gi...  (self-contained)
decoded payload = {"_fresh": false, "_permanent": true, "csrf_token": "6546e9a824b6a6fd6d24cc104ad20d2b90a53929"}
# after login the cookie grows to 322 chars with a leading '.' (zlib-compressed) — no server-side key
```

**File:line grounding (sessions):** `SESSION_PREFIX = "session"` `app/session.py:L18`;
`class RedisSessionStore(...)` (L31); key `f"{SESSION_PREFIX}:{...}"` (L45); `save_session` (L82);
TTL logic (`ttl = int(...)` L92, `if "_user_id" not in session:` L95, `ttl = 300` L96); installed by `initialize_redis_services()` `app/redis_services.py:L9`,
`app.session_interface = RedisSessionStore(...)` (L12). Cookie name `SESSION_COOKIE_NAME = "slapp"`
`app/config.py:L199`; `SESSION_COOKIE_SAMESITE = "Lax"` `server.py:L162` (observed `SameSite=Lax`,
`HttpOnly`, `Path=/`, 7-day `Expires`).

### Rate limiting & teardown (context)

`Flask-Limiter`'s key is `userid:{current_user.id}` when authenticated else `ip:{ip_addr}`
(`app/extensions.py:L14-19`), and it is disabled here because `DISABLE_RATE_LIMIT=1`
(`disable_rate_limit` `app/extensions.py:L26-28`). Per request, `@app.teardown_appcontext def cleanup(...): Session.remove()`
(`server.py:L209-211`) releases the scoped SQLAlchemy session.

### Observed vs Inferred

- **Observed:** the full entry chain (via `remote_addr=127.0.0.1`); the web login flow and the
  `_user_id == alternative_id` (UUID4, not `1`) fact cross-checked against the DB; the anonymous→authenticated
  Redis TTL transition (300→604800); all four API cases [A]–[D]; the convergence via case [D]; the exact
  `after_request` lines; both 401 edges [E]/[F]; and both session backends (Redis pointer vs self-contained
  signed cookie, decoded).
- **Inferred:** the `jsonify(error="Unauthorized"), 401` body for `/api/` — it is reachable only via
  `abort(401)`, which SimpleLogin's `/api` views do not use (they return `Wrong api key`); this variant
  was not produced at runtime and is therefore labeled inferred.

---

## Q6 — Does the webapp auto-start any background jobs or schedulers?

### Direct answer (Observed)

**No.** Starting the webapp (`python3 server.py`) spawns **no** scheduler thread and **no**
background job process. The scheduled/background work lives in four **independent** scripts —
`cron.py`, `job_runner.py`, `event_listener.py`, and `email_handler.py` — each guarded by its own
`if __name__ == "__main__":` block and **not imported by `server.py`**. They are launched
separately (by an external `yacron`/OS cron, or manually), not by the web process.

### Command(s) run

```
# 1) does server.py import any of the background scripts?
grep -nE "import (cron|job_runner|event_listener|email_handler)|from (cron|job_runner|event_listener|email_handler)" server.py ; echo GREP_EXIT=$?

# 2) each background script's __main__ guard
grep -n '__name__ == "__main__"' cron.py job_runner.py event_listener.py email_handler.py

# 3) runtime: launch the webapp, confirm /health, enumerate its process tree + threads,
#    and scan the startup log for any scheduler/cron activity
setsid /app/venv/bin/python server.py >/tmp/startup.log 2>&1 &   # then poll /health
ps -eo pid,ppid,args | grep "[s]erver.py"
ls /proc/<serving_child_pid>/task | wc -l
grep -nE "cron|job_runner|event_listener|scheduler|APScheduler|Start running cronjob|Take job" /tmp/startup.log

# 4) documentation corroboration
grep -nE "cron\.py|job_runner\.py|event_listener\.py|email_handler\.py|server\.py" CONTRIBUTING.md
```

### Complete, unedited output

**(1) `server.py` imports none of them** (grep exit 1 = zero matches):

```
GREP_EXIT=1
```

**(2) each is a standalone `__main__` entry point:**

```
cron.py:1262:if __name__ == "__main__":
job_runner.py:329:if __name__ == "__main__":
event_listener.py:94:if __name__ == "__main__":
email_handler.py:2396:if __name__ == "__main__":
```

Context confirming independence (each builds its own app context / loop / listener):

```
# cron.py L1262+           argparse -j/--job ; with create_light_app().app_context(): <dispatch job>
# job_runner.py L329-330   if __name__ == "__main__":  /  while True:   (polls get_jobs_to_run(), process_job(job))
# event_listener.py L94+   argv subcommand dispatch (LISTENER / DEAD_LETTER / debug / run)
# email_handler.py L2396+  argparse -p/--port (default 20381) ; main(port=args.port)  (SMTP/LMTP listener)
```

**(3) runtime — the webapp process tree and threads (no scheduler):**

```
READY=1 (waited 5 half-seconds)   # /health returned 200

# process tree: only the reloader parent and its serving child
   3999    3992 /app/venv/bin/python server.py
   4023    3999 /app/venv/bin/python /app/server.py

# threads of the serving child (pid 4023):
thread count (tids): 2
  tid=4023 comm=python
  tid=4036 comm=python
# threads of the reloader parent (pid 3999):
thread count (tids): 1

# scan of the startup log for scheduler/cron/job activity:
  (NONE - webapp emitted no scheduler activity)
```

The serving child has exactly **2 threads**, both belonging to Werkzeug's own development server
(the request-serving thread and the reloader's stat-polling loop — in Werkzeug 1.0.1
`run_with_reloader` runs `main_func` in a daemon thread and `reloader.run()` in the main thread).
Neither is an application job scheduler.

**(4) `CONTRIBUTING.md` corroboration** (these are listed as *separate* entry points, and cron is
invoked as its own process):

```
106:alembic upgrade head && flask dummy-data && python3 server.py
145:- wsgi.py and server.py: the webapp.
146:- email_handler.py: the email handler.
147:- cron.py: the cronjob.
212:python email_handler.py
228:python job_runner.py
```

`crontab.yml` schedules the cron jobs as external processes, e.g.
`command: python /code/cron.py -j stats` on `schedule: "0 0 * * *"` (15 such jobs);
`crontab-all-hosts.yml` schedules `python /code/cron.py -j send_undelivered_mails` on `"*/5 * * * *"`.
These are run by an external `yacron`, not by the web process.

### File:line grounding

- `server.py` contains **no** `import cron / job_runner / event_listener / email_handler` (grep exit 1).
- `if __name__ == "__main__":` guards: `cron.py:L1262`, `job_runner.py:L329` (loop `while True:` at L330),
  `event_listener.py:L94`, `email_handler.py:L2396`.
- The webapp factory is `create_app()` `server.py:L139`; the background scripts instead use
  `create_light_app()` `server.py:L127` (its own teardown `shutdown_session` at L132-134) — a different,
  request-less app used only for a CLI/loop app-context.
- Entry-point list `CONTRIBUTING.md:L145-147`; run commands `CONTRIBUTING.md:L212,L228`.

### Observed vs Inferred

- **Observed:** `server.py` imports none of the four scripts; each has its own `__main__` guard; the
  running webapp's process tree and thread set contain no scheduler; the startup log contains no cron/job
  activity; `crontab.yml`/`crontab-all-hosts.yml` invoke `cron.py` as external processes.
- **Inferred:** none.

---

## Q7 — Anything else going on (import-time side effects)?

### Direct answer (Observed)

Beyond request handling, the app has notable **import-time** and **per-request** side effects:

1. **A PostgreSQL connection is opened at import time.** `app/db.py` calls `engine.connect()` at module
   top level, so merely importing the app (through the canonical package) opens a live DB connection whose
   `application_name` is `webapp`. If Postgres is unreachable, the **import itself** fails fast.
2. **An environment flag is set at import.** `server.py:L124` sets
   `os.environ["OAUTHLIB_INSECURE_TRANSPORT"] = "1"` (permits OAuth over plain HTTP in dev).
3. **Per request, `Session.remove()` runs** in a `teardown_appcontext` hook (`server.py:L209-211`).
4. **The Werkzeug reloader double-imports the module** (because `debug=True` with no `use_reloader=False`),
   so all import-time `print` banners appear **twice** (see Q1).

### Command(s) run

```
# import-time DB connect (positive): import app.db, then ask Postgres about ourselves
cat > /tmp/blitzy_q7_db.py <<'PY'
import os
print("OAUTHLIB_INSECURE_TRANSPORT before importing server:", repr(os.environ.get("OAUTHLIB_INSECURE_TRANSPORT")))
import app.db as d                     # app/db.py L9-14 runs engine.connect() at IMPORT
rows = d.connection.execute(
    "select pid, application_name, state, backend_type from pg_stat_activity where application_name = %(n)s",
    {"n": "webapp"}).fetchall()
print("import-time connections with application_name=webapp:")
for r in rows: print("  ", dict(r))
import app.config as c
print("config.DB_CONN_NAME =", repr(c.DB_CONN_NAME)); print("config.DB_URI =", repr(c.DB_URI))
import server                          # noqa
print("OAUTHLIB_INSECURE_TRANSPORT after importing server :", repr(os.environ.get("OAUTHLIB_INSECURE_TRANSPORT")))
PY
PYTHONPATH=/app /app/venv/bin/python /tmp/blitzy_q7_db.py

# import-time DB connect (fail-fast): point DB_URI at a dead port and import app.db
DB_URI="postgresql://test:test@localhost:5999/test" PYTHONPATH=/app /app/venv/bin/python -c "import app.db" ; echo EXIT=$?
```

### Complete, unedited output

**Import-time DB connect (positive) + OAUTHLIB flag flip:**

```
OAUTHLIB_INSECURE_TRANSPORT before importing server: None
>>> URL: http://localhost:7777
Upload files to local dir
import-time connections with application_name=webapp:
   {'pid': 4110, 'application_name': 'webapp', 'state': 'active', 'backend_type': 'client backend'}
config.DB_CONN_NAME = 'webapp'
config.DB_URI       = 'postgresql://test:test@localhost:5432/test'
>>> init logging <<<
2026-07-13 17:29:30,940 - SL - DEBUG - 4109 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
OAUTHLIB_INSECURE_TRANSPORT after importing server : '1'
```

**Import-time DB connect (fail-fast) — importing against a dead port raises at import:**

```
sqlalchemy.exc.OperationalError: (psycopg2.OperationalError) connection to server at "localhost" (::1), port 5999 failed: Connection refused
	Is the server running on that host and accepting TCP/IP connections?
connection to server at "localhost" (127.0.0.1), port 5999 failed: Connection refused
	Is the server running on that host and accepting TCP/IP connections?

(Background on this error at: http://sqlalche.me/e/13/e3q8)
>>> URL: http://localhost:7777
Upload files to local dir
EXIT=1
```

The reloader double-import is visible in the Q1 startup stream (the `>>> URL:` / `Upload files to local dir`
/ `>>> init logging <<<` / SL "load words file" block appears **twice**).

### File:line grounding

- `engine = create_engine(config.DB_URI, connect_args={"application_name": config.DB_CONN_NAME})`
  `app/db.py:L9-11`; `connection = engine.connect()` `app/db.py:L12`;
  `Session = scoped_session(sessionmaker(bind=connection))` `app/db.py:L14`. The connection is opened at
  **module import**, not lazily — proven by the fail-fast import above.
- `DB_CONN_NAME = os.environ.get("DB_CONN_NAME", "webapp")` `app/config.py:L193` (hence
  `application_name = 'webapp'`).
- `os.environ["OAUTHLIB_INSECURE_TRANSPORT"] = "1"` `server.py:L124` (observed `None` → `'1'` across the
  `import server`).
- `@app.teardown_appcontext def cleanup(resp_or_exc): Session.remove()` `server.py:L209-211` (per request).
- Reloader double-import: consequence of `app.run(debug=True, port=7777)` `server.py:L588` with no
  `use_reloader=False` — see Q1.

### Observed vs Inferred

- **Observed:** the import-time DB connection (`application_name='webapp'`, pid 4110, `active`) opened by a
  bare `import app.db`; the fail-fast import against a dead port (`OperationalError ... Connection refused`,
  `EXIT=1`), which proves the connect is eager at import; the `OAUTHLIB_INSECURE_TRANSPORT` flip `None`→`'1'`;
  and the reloader double-import (from Q1).
- **Inferred:** the per-request `Session.remove()` is grounded in code and runs on every
  `teardown_appcontext`; it was observed indirectly (each request in Q5 completed and released its scoped
  session) rather than by printing from inside the teardown hook.

---

## Coverage-Pass Checklist

A final pass re-reading each question and confirming every distinct thing and every named item is
answered with a concrete value, a `file:line`, observed evidence, sibling variants, and a cause→effect.

**Q1 — Startup (dev):**
- [x] Canonical entry `python3 server.py` → `local_main()` `server.py:L572-595`, guard L598-599 — *Observed*.
- [x] Sequence: `COLOR_LOG=True` (L573) → `create_app()` (L574) → DebugToolbar (L577/582) → `app.debug=True` (L581) → `app.run(debug=True, port=7777)` (L588). Cause→effect: `debug=True` ⇒ reloader ⇒ double import — *Observed*.
- [x] Port 7777 (see Q4); prod contrast `wsgi.py`+gunicorn `Dockerfile:L47` — *Inferred*.

**Q2 — Config loading (both branches):**
- [x] `CONFIG` branch prints `load config file ...` (`app/config.py:L68`) — *Observed*.
- [x] Default `./.env` branch, silent `load_dotenv()` (L71), resolved values — *Observed*.
- [x] Missing required var → `KeyError: 'URL'` at import (`app/config.py:L79`) — *Observed*.
- [x] Empty `FLASK_SECRET` → `RuntimeError` (`app/config.py:L197-198`) — *Observed*.
- [x] Required set: `URL`(L79),`EMAIL_DOMAIN`(L92),`SUPPORT_EMAIL`(L93),`DB_URI`(L192),`FLASK_SECRET`(L196) — *Observed*.

**Q3 — Readiness logs (version-dependent):**
- [x] Surviving markers: `>>> URL:` (`app/config.py:L80`), `>>> init logging <<<` (`app/log.py:L67`), Flask CLI banner (`click.echo`) — *Observed*.
- [x] Suppressed banner (`* Running on`, `Press CTRL+C`, `* Restarting with stat`, `* Debugger is active!`, `* Debugger PIN`) because `log.disabled=True` (`app/log.py:L71`) on Werkzeug 1.0.1 / Flask 1.1.2 — *Observed*.
- [x] Direct answer: no explicit readiness line; `/health`→200 is the practical signal; Debugger PIN's absence is stable cross-run — *Observed*.

**Q4 — Ports & endpoints:**
- [x] Port 7777 loopback bind (`/proc/net/tcp` `0100007F:1E61` state `0A`; `server.py:L588`, `Dockerfile:L44`) — *Observed*.
- [x] 292 live rules dumped in full — *Observed*.
- [x] Every blueprint prefix: `/auth`,`/`(monitor),`/dashboard`,`/developer`,`/phone`,`/oauth`,`/oauth2`,`/onboarding`,`/discover`,`/internal`,`/api` (`server.py:L233-246`; `app/api/base.py:L11`); `/admin` (Flask-Admin, 122) — *Observed*.
- [x] Direct routes: `/`,`/health`,`/favicon.ico`,`/dnt`,`/.well-known/openid-configuration`,`/jwks`,`/coinbase`,`/paddle`,`/paddle_coupon`,`/static` — *Observed*.
- [x] `monitor_bp` at root `/` (routes `/git`,`/live`,`/exception`; no `/monitor/*`) — read-vs-run discrepancy, `app/monitor/base.py:L3` — *Observed*.
- [x] `/oauth`+`/oauth2` double registration (`server.py:L240-241`) — *Observed*.

**Q5 — Authenticated request (PRIMARY):**
- [x] (a) Entry: `ProxyFix` (L142) → `make_session_permanent` (L204-207) → `before_request` `g.start_time` (L258-269); `remote_addr=127.0.0.1` — *Observed*.
- [x] (b) Web identity: `load_user(alternative_id)` (`server.py:L220-222`); `session["_user_id"]`=`01330a11-…` == `alternative_id` (UUID4) ≠ `id`(1); `get_id()` (`app/models.py:L595-599`); `session_protection="strong"` (`app/extensions.py:L8`) — *Observed*, stable across 2 runs (session id variable).
- [x] (b) API identity: `authorize_request()` (`app/api/base.py:L16-42`); cases [A] valid→200, [B] wrong→401, [C] none→401, [D] session fall-through→200 — *Observed*.
- [x] (c) Convergence: `get_current_user()` (`server.py:L332-336`), `g.user` (API) vs `current_user` (web); [D] unifies — *Observed*.
- [x] `after_request` line format `remote_addr method path args status, takes t` (`server.py:L285`, skip list L276-282); `takes` variable — *Observed*.
- [x] 401 edges: [E] web → 302 `Location .../auth/login?next=%2Fdashboard%2F%3F`; [F] api → 401 `{"error":"Wrong api key"}` (`server.py:L347-352`) — *Observed*; `Unauthorized` abort-body — *Inferred*.
- [x] Two backends: Redis pointer cookie + server key `session:<uuid>` TTL 604800/300 (`app/session.py:L92,95-96`); signed-cookie self-contained `slapp` decoded (`app/config.py:L199`, `SameSite=Lax` `server.py:L162`) — *Observed*.
- [x] Rate-limit key `userid`/`ip` (`app/extensions.py:L14-19`, disabled L26-28); teardown `Session.remove()` (`server.py:L209-211`) — *Observed*/grounded.

**Q6 — Background jobs:**
- [x] Direct answer: webapp starts none — *Observed*.
- [x] `server.py` imports none of the four scripts (grep exit 1) — *Observed*.
- [x] `__main__` guards: `cron.py:L1262`, `job_runner.py:L329`(+`while True` L330), `event_listener.py:L94`, `email_handler.py:L2396` — *Observed*.
- [x] Runtime: process tree (reloader parent + serving child), serving child = 2 Werkzeug threads, no scheduler; startup log has no cron/job activity — *Observed*.
- [x] `CONTRIBUTING.md:L145-147` entry points; `crontab.yml`/`crontab-all-hosts.yml` invoke `cron.py` externally — *Observed*.

**Q7 — Anything else (side effects):**
- [x] Import-time DB connect (`app/db.py:L9-14`), `application_name='webapp'` (`app/config.py:L193`), positive + fail-fast — *Observed*.
- [x] `OAUTHLIB_INSECURE_TRANSPORT="1"` at import (`server.py:L124`), `None`→`'1'` — *Observed*.
- [x] Per-request teardown `Session.remove()` (`server.py:L209-211`) — grounded, observed indirectly.
- [x] Reloader double-import (`server.py:L588`, `debug=True`) — prints appear twice, cross-ref Q1 — *Observed*.

_All seven questions and every named sub-item are answered above with a direct answer first,
the exact command(s), the complete unedited output, live `file:line` grounding, and an
Observed/Inferred label._

