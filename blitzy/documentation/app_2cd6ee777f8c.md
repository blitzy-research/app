# SimpleLogin Backend — Actual Runtime Behavior in Local Development Mode

This document explains what the SimpleLogin backend **actually does — and actually prints — at runtime** when it is started locally in development mode on branch `app_2cd6ee777f8c`. It deliberately answers *"what gets printed when you run it,"* not *"what the code says should happen."* Every claim is traceable to a specific source location (inline `file:line` citations such as `server.py:L588`) and, where it concerns printed output, to **literal stdout captured from a real run** performed inside the provided container image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (Python `3.10`; code at `/app` at the same commit as this branch; the virtualenv carries the exact pins `flask 1.1.2` / `werkzeug 1.0.1` / `click 8.0.3` / `sqlalchemy 1.3.24` / `flask_login 0.5.0`). The complete captured logs are reproduced verbatim in [Appendix A](#appendix-a--verbatim-captured-output), and a claim-to-citation map is in [Appendix B](#appendix-b--evidence-index).

---

## TL;DR

- **Dev entry point.** `python3 server.py` runs the `__main__` guard (`server.py:L598-599`), which calls `local_main()` (`server.py:L572-595`); that builds the app via `create_app()` and launches **Werkzeug's development server** with `app.run(debug=True, port=7777)` (`server.py:L588`). Production instead runs `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` (`Dockerfile:L47`). There is **one** construction path (`create_app()`), **two** launch modes.
- **Configuration loads eagerly at import.** `app/config.py` runs top-to-bottom on import: `load_dotenv()` reads `.env` (`app/config.py:L71`), required variables like `URL` and `FLASK_SECRET` are resolved immediately and raise if absent (`app/config.py:L79`, `L196-198`), and `print(">>> URL:", URL)` fires at `app/config.py:L80`.
- **What "ready" actually looks like.** The readiness-adjacent lines that *do* print are the import-time `print()`s — `>>> URL: http://localhost:7777` (`app/config.py:L80`) and `>>> init logging <<<` (`app/log.py:L67`) — plus the Flask startup banner ending in `* Debug mode: on`. The familiar Werkzeug `* Running on http://127.0.0.1:7777/` line is **NOT printed** — it is suppressed because the `werkzeug` logger is disabled at import (`app/log.py:L70-71`). Readiness is therefore effectively *silent*; it was confirmed empirically via `GET /health → 200`.
- **Port.** The server listens on **7777** (`server.py:L588`; `Dockerfile:L44`).
- **Identity is dual-path.** Browser sessions resolve `current_user` through the Flask-Login `user_loader` keyed on the UUID `alternative_id` (`server.py:L220-230` + `app/models.py:L595-599`), **not** the numeric primary key. API calls resolve `g.user` from the `Authentication` request header (`app/api/base.py:L16-43`).
- **No background jobs run in the web process.** The dev server serves HTTP only. Every periodic / asynchronous / SMTP task is a **separate top-level `__main__` script** (`job_runner.py`, `cron.py`, `event_listener.py`, `email_handler.py`). Most of these wrap their work in a minimal Flask app context built by `create_light_app()` (`job_runner.py`, `cron.py`, `init_app.py`, and — per processed message — `email_handler.py`), **but `event_listener.py` does *not*** — it runs its own Postgres event-source → `Runner` loop with no Flask app or app context (`event_listener.py:L5`, `L46-47`).
- **Other import-time activity.** An **eager database connection** is opened at import (`app/db.py:L9-14`) — but *not* before the import-time prints. Configuration is imported and printed first (`>>> URL:`, `app/config.py:L80`), then `>>> init logging <<<` (`app/log.py:L67`), and only *afterwards* does `app.db`'s `engine.connect()` (`app/db.py:L12`) run; so an unreachable Postgres aborts startup *after* those two lines have already printed, but *before* app creation, the Flask banner, and readiness. Also at import: optional Sentry init only if `SENTRY_DSN` is set (`server.py:L111-121`); a Flask-Limiter rate limiter is wired up (`app/extensions.py:L14-23`); and `OAUTHLIB_INSECURE_TRANSPORT=1` is set (`server.py:L124`).

---

## Methodology & Rationale

This document was produced with a disciplined **investigate → run → capture → write** workflow that left the source tree unchanged.

**1. Investigate (static analysis).** The web bootstrap and request path were traced across `server.py`, `wsgi.py`, `app/config.py`, `app/log.py`, `app/db.py`, `app/extensions.py`, `app/session.py`, `app/redis_services.py`, `app/api/base.py`, and `app/models.py`, plus the separate worker entry points `job_runner.py`, `cron.py` (with `crontab.yml`), `event_listener.py`, `email_handler.py`, and `init_app.py`. Exact `file:line` locations were recorded for every emitted line and every identity-resolution branch.

**2. Run (empirical reproduction).** Because the user wants *"what actually gets printed when you run it,"* the app was executed inside the provided container image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (Python `3.10`, code at `/app` at this branch's commit, virtualenv carrying the exact legacy pins). A PostgreSQL instance was provisioned, a **throwaway `.env`** was created from `example.env`, and the documented sequence `alembic upgrade head && flask dummy-data && python3 server.py` (`CONTRIBUTING.md:L106`) was run.

**3. Capture.** Literal stdout was captured up to readiness, then one authenticated **web** request (log in as `john@wick.com` / `password`, `CONTRIBUTING.md:L109`, then fetch a dashboard route) and one authenticated **API** request (carrying the `Authentication` header) were exercised, along with a deliberately bad API key. The raw logs are in [Appendix A](#appendix-a--verbatim-captured-output).

**4. Write.** This document answers objectives **O1–O7**, each with an **(a) Observed** subsection (the literal captured output) and a **(b) Why — code rationale** subsection (the `file:line` citations that explain it). Afterward, the throwaway `.env` and helper scripts were removed and the source tree was verified pristine (`git status` empty); the only artifact added is this Markdown file.

Three distinctions are maintained throughout, because they are essential to answering the questions honestly:

- **Dev vs. prod** — the Werkzeug development server (`server.py` / `app.run`) is kept separate from the production WSGI server (`gunicorn wsgi:app`).
- **Web vs. workers** — the single web process is kept separate from the distinct background processes (job runner, cron, event listener, email handler).
- **Code says vs. actually prints** — the headline nuance: the code *would* emit a Werkzeug `* Running on ...` banner, but because the `werkzeug` logger is disabled (`app/log.py:L70-71`), that line **never actually prints**. This is confirmed empirically in [Appendix A](#appendix-a--verbatim-captured-output).

> **Versions referenced.** All behavior is described for the project-pinned stack, not newer releases: Flask `1.1.2`, Werkzeug `1.0.1` (transitive, pinned via `poetry.lock`), click `8.0.3`, flask_login `0.5.0`, SQLAlchemy `1.3.24`, gunicorn `20.0.4`, gevent `22.10.2`, redis `^4.5.3`, newrelic `8.8.0`, coloredlogs `14.0`, python-dotenv `^0.14.0`, Python `3.10` (`pyproject.toml`; `poetry.lock`).

---

## O1 — Dev-mode startup sequence (and the production contrast)

### (a) Observed

Running `python3 server.py` produces the following ordered stdout up to readiness (canonical capture, real `.env`, `COLOR_LOG` unset — see [Appendix A](#appendix-a--verbatim-captured-output)):

```
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/ficswppbuayxvsommeoy
Upload files to local dir
>>> init logging <<<
2026-06-26 19:47:35,485 - SL - DEBUG - 7 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
```

Then the **entire block repeats** (the import-time prints and the `load words file` log line appear a second time under a different process id), after which the server is listening. This doubling — and the absence of the usual `* Running on ...` line — is explained in [O3](#o3--readiness-log-messages-what-actually-prints).

The production launch (`gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15`) prints a different, gunicorn-owned banner instead (full capture in [Appendix A](#appendix-a--verbatim-captured-output)):

```
[2026-06-26 19:50:44 +0000] [7] [INFO] Starting gunicorn 20.0.4
[2026-06-26 19:50:44 +0000] [7] [INFO] Listening at: http://0.0.0.0:7777 (7)
[2026-06-26 19:50:44 +0000] [7] [INFO] Using worker: sync
[2026-06-26 19:50:44 +0000] [12] [INFO] Booting worker with pid: 12
[2026-06-26 19:50:44 +0000] [13] [INFO] Booting worker with pid: 13
```

### (b) Why — code rationale

- **The dev entry point is the `__main__` guard.** `if __name__ == "__main__": local_main()` at `server.py:L598-599` runs only when the file is executed directly (`python3 server.py`), not when imported.
- **`local_main()` configures and launches the Werkzeug dev server.** At `server.py:L572-595` it: forces `config.COLOR_LOG = True` (`server.py:L573`); builds the app with `create_app()` (`server.py:L574`); imports and attaches the **Flask-DebugToolbar** (`from flask_debugtoolbar import DebugToolbarExtension` … `DebugToolbarExtension(app)`, `server.py:L577-582`); sets `app.debug = True` (`server.py:L581`); and calls **`app.run(debug=True, port=7777)`** (`server.py:L588`). Passing `debug=True` turns on Werkzeug's interactive debugger **and** the auto-reloader.
- **The app is built by a single factory.** `create_app()` at `server.py:L139-217` constructs `Flask(__name__)` (`server.py:L140`), wraps the WSGI app in `ProxyFix(app.wsgi_app, x_for=1, x_host=1)` (`server.py:L142`) because SimpleLogin runs behind NGINX, disables strict slashes (`server.py:L144`), sets `SQLALCHEMY_DATABASE_URI = DB_URI` (`server.py:L146`) and `app.secret_key = FLASK_SECRET` (`server.py:L151`), configures the session cookie (`server.py:L159-162`), conditionally enables Redis-backed services when `MEM_STORE_URI` is set (`server.py:L163-165`), then wires up the limiter, error pages, extensions, blueprints, admin, payment callbacks, and the `/health` route, finally `return app` (`server.py:L217`).
- **Production reuses the same factory through `wsgi.py`.** `wsgi.py` is just `from server import create_app` (`wsgi.py:L1`) and `app = create_app()` (`wsgi.py:L3`); gunicorn imports this `app`. The production command, port, and base image are fixed by the Dockerfile: `CMD ["gunicorn","wsgi:app","-b","0.0.0.0:7777","-w","2","--timeout","15"]` (`Dockerfile:L47`), `EXPOSE 7777` (`Dockerfile:L44`), `FROM python:3.10` (`Dockerfile:L8`).
- **Key narrative — one construction, two launch modes.** Both dev and prod build the identical app via `create_app()`. **Dev** adds three things prod does not have: the interactive debugger, the auto-reloader (both from `debug=True`), and the Flask-DebugToolbar (`server.py:L577-582`). **Prod** runs two synchronous gunicorn workers and none of those dev conveniences.
- **`create_light_app()` is a different, smaller app — and it is NOT the web server.** `create_light_app()` at `server.py:L127-136` builds a minimal `Flask(__name__)` with just the SQLAlchemy URI and a teardown hook. It is used by **several of the background workers** (see [O6](#o6--background-jobs-and-schedulers-multi-process-architecture)) — though not all of them — and never by the dev or prod web path.

---

## O2 — Configuration loading

### (a) Observed

Configuration is **fully resolved at import time**, before any HTTP server starts. With a `.env` produced from `example.env`, the resolved `URL` is echoed immediately:

```
>>> URL: http://localhost:7777
```

and four further import-time `print()`s appear in the canonical capture as a direct consequence of evaluating `app/config.py`:

```
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/ficswppbuayxvsommeoy
Upload files to local dir
```

If a required variable is missing the process aborts at import with a `KeyError`/`RuntimeError` before any banner prints (e.g. an empty `FLASK_SECRET` raises `RuntimeError("FLASK_SECRET is empty. Please define it.")`).

### (b) Why — code rationale

- **`.env` is read eagerly via `load_dotenv()`.** `app/config.py` imports `from dotenv import load_dotenv` and then branches at module top level: if a `CONFIG` environment variable is set it prints `load config file <path>` (`app/config.py:L68`) and calls `load_dotenv(config_file)` (`app/config.py:L69`); **otherwise it calls `load_dotenv()`** which reads `.env` from the working directory (`app/config.py:L71`). The canonical run uses the plain `.env` path, so the `load config file` line does **not** appear (see the footnote in [Appendix A](#appendix-a--verbatim-captured-output)).
- **`COLOR_LOG` is a presence check.** `COLOR_LOG = "COLOR_LOG" in os.environ` (`app/config.py:L73`) — it is true only if the variable exists in the environment at import time. In `example.env` it is commented out (`example.env:L16`), so by default it is false.
- **Required variables are resolved immediately and raise if absent.** `URL = os.environ["URL"]` (`app/config.py:L79`) followed by `print(">>> URL:", URL)` (`app/config.py:L80`); `EMAIL_DOMAIN = os.environ["EMAIL_DOMAIN"].lower()` (`app/config.py:L92`); `SUPPORT_EMAIL = os.environ["SUPPORT_EMAIL"]` (`app/config.py:L93`); `DB_URI = os.environ["DB_URI"]` (`app/config.py:L192`); `DB_CONN_NAME = os.environ.get("DB_CONN_NAME", "webapp")` (`app/config.py:L193`); `FLASK_SECRET = os.environ["FLASK_SECRET"]` (`app/config.py:L196`) with `if not FLASK_SECRET: raise RuntimeError(...)` (`app/config.py:L197-198`). Each bare `os.environ[...]` access raises `KeyError` at import if the variable is unset — which is why a reachable, fully populated configuration must exist *before* import.
- **Fixed and optional settings.** `SESSION_COOKIE_NAME = "slapp"` is a constant (`app/config.py:L199`) — this is the cookie name observed during the login flow. `MEM_STORE_URI = os.environ.get("MEM_STORE_URI", None)` (`app/config.py:L568`) gates Redis; `DISABLE_RATE_LIMIT = "DISABLE_RATE_LIMIT" in os.environ` (`app/config.py:L602`) gates the limiter.
- **The other import-time prints come from config evaluation, too.** `MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value` (`app/config.py:L123`), `Paddle param not set` (`app/config.py:L217`), `WARNING: Use a temp directory for GNUPGHOME <path>` (`app/config.py:L262`), and `Upload files to local dir` (`app/config.py:L328`) are all plain `print()` statements that execute as `app/config.py` is imported. The GNUPGHOME path is a freshly created temp directory, which is why it differs between runs.
- **The `.env` used for the run.** It was built from `example.env`: `URL=http://localhost:7777` (`example.env:L6`), `NOT_SEND_EMAIL=true` (`example.env:L19`), `EMAIL_DOMAIN=sl.local` (`example.env:L22`), `SUPPORT_EMAIL=support@sl.local` (`example.env:L40`), `DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin` (`example.env:L75`), `FLASK_SECRET=secret` (`example.env:L77`). Note that `CONTRIBUTING.md` documents alternate Postgres host ports for docker-based local setups (e.g. the `-p 15432:5432` mapping at `CONTRIBUTING.md:L100`), so a developer's `.env` `DB_URI` must point at whatever port their Postgres actually listens on; inside the provided container Postgres is on `5432`, matching `example.env`.


---

## O3 — Readiness log messages (what actually prints)

This section answers the user's primary curiosity directly: **which exact lines indicate the server is ready, and which familiar lines do *not* appear.**

### (a) Observed

The only lines that print around readiness are:

1. The import-time plain `print()`s:
   ```
   >>> URL: http://localhost:7777
   >>> init logging <<<
   ```
2. The first `SL` logger line (emitted while loading the words file at import):
   ```
   2026-06-26 19:47:35,485 - SL - DEBUG - 7 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
   ```
3. The Flask startup banner:
   ```
    * Serving Flask app "server" (lazy loading)
    * Environment: production
      WARNING: This is a development server. Do not use it in a production deployment.
      Use a production WSGI server instead.
    * Debug mode: on
   ```

**Critical finding — what is NOT printed.** There is **no** `* Running on http://127.0.0.1:7777/` line, **no** `* Restarting with stat`, **no** `* Debugger is active!`, and **no** `* Debugger PIN: ...` line. The development server is nevertheless listening — readiness was confirmed out-of-band with `GET /health` returning `200` (`READY=1 after 4s (via GET /health == 200)` in [Appendix A](#appendix-a--verbatim-captured-output)). In practical terms, **readiness is effectively silent**: the last thing you see before the server accepts requests is the `* Debug mode: on` banner line.

**Reloader doubling.** Because `debug=True` enables the auto-reloader, the entire import-time block prints **twice** — once in the parent process (process id **7** in the capture) and once in the reloader child (process id **23**):

```
>>> init logging <<<
2026-06-26 19:47:35,485 - SL - DEBUG - 7  - ... - load words file: ...   <- parent (pid 7)
...
>>> init logging <<<
2026-06-26 19:47:38,188 - SL - DEBUG - 23 - ... - load words file: ...   <- reloader child (pid 23)
```

The child (pid 23) is the process that actually serves requests, which is why every per-request `after_request` line in the capture carries pid 23.

**Per-request logging is the app's own, not Werkzeug's.** No standard Werkzeug access lines (e.g. `127.0.0.1 - - [..] "GET / HTTP/1.1" 200 -`) appear. Instead, the application's `after_request` hook emits structured `SL` lines like:

```
2026-06-26 19:47:39,146 - SL - DEBUG - 23 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.1112067699432373
```

### (b) Why — code rationale

- **The two `>>>` lines are bare `print()`s.** `print(">>> URL:", URL)` lives at `app/config.py:L80` (runs during config import) and `print(">>> init logging <<<")` lives at `app/log.py:L67` (runs during logging import). They are not routed through any logger, so they always print regardless of logger configuration.
- **The `SL` logger.** The application logger is named `SL` and created by `LOG = _get_logger("SL")` (`app/log.py:L79`). It logs at `DEBUG` (`app/log.py:L51`), does not propagate to the root logger (`app/log.py:L59`), writes to `sys.stdout` (`app/log.py:L41`), uses UTC timestamps via `time.gmtime` (`app/log.py:L43`), and uses the fixed multi-field format at `app/log.py:L12-15`:
  ```
  %(asctime)s - %(name)s - %(levelname)s - %(process)d - "%(pathname)s:%(lineno)d" - %(funcName)s() - %(message_id)s - %(message)s
  ```
  This is exactly the shape of every `SL` line in the capture — including the `%(process)d` field that reveals the pid (7 vs 23).
- **The Flask banner is printed by click, not by a logger.** In Flask `1.1.2`, `app.run()` calls Werkzeug's `run_simple`, and Flask's CLI prints the `* Serving Flask app ... / * Environment: production / WARNING: ... / * Debug mode: on` banner via `click.echo` (Flask's `cli.show_server_banner`). Because it does not go through the `werkzeug` logger, it survives even though that logger is disabled. (`* Environment: production` is shown because `FLASK_ENV` is not set to `development`; it is purely informational and does not change that `debug=True` is active.)
- **The missing lines are suppressed by a disabled logger — the headline "code says X but Y prints" nuance.** Werkzeug `1.0.1` emits the `* Running on ...`, `* Restarting with stat`, `* Debugger is active!`, and `* Debugger PIN: ...` lines through `logging.getLogger("werkzeug")`. At import time SimpleLogin disables that logger:
  ```python
  # app/log.py:L70-71
  log = logging.getLogger("werkzeug")
  log.disabled = True
  ```
  The comment right above it (`app/log.py:L69`) states the intent: silence Flask's default `127.0.0.1 - - [...] "GET ..." 200` access lines. A side effect is that the dev-server "Running on" / debugger lines are silenced too — so the code *would* announce its address, but at runtime it does not.
- **Per-request lines come from `after_request`.** The structured request line is emitted by `LOG.d("%s %s %s %s %s, takes %s", request.remote_addr, request.method, request.path, request.args, res.status_code, time.time() - start_time)` at `server.py:L284-292`. Because `after_request` skips `/health` (`server.py:L281`), the health probe used to confirm readiness does not itself produce a log line — consistent with the silent readiness.
- **The Flask-DebugToolbar `UserWarning` is dev-only.** The capture's `UserWarning: Could not insert debug toolbar. </body> tag not found in response.` originates from the toolbar attached in `local_main()` (`server.py:L577-582`); it is harmless and appears only because of the dev toolbar.
- **Color/timestamp variation.** With `COLOR_LOG` unset (the `example.env` default), `SL` timestamps include milliseconds (`19:47:35,485`). With `COLOR_LOG=true` present at import, `coloredlogs.install()` runs (`app/log.py:L61-62`) and reformats timestamps without milliseconds (`19:49:38`), applying ANSI colors only on a TTY. Note that `local_main()` sets `config.COLOR_LOG = True` (`server.py:L573`) *after* `LOG` was already constructed at import (`app/log.py:L79`), so that assignment is inert for the `SL` logger — coloring is governed by whether `COLOR_LOG` was in the environment at import time. Both variants are shown in [Appendix A](#appendix-a--verbatim-captured-output).

---

## O4 — Exposed ports and endpoints

### (a) Observed

The server listens on **port 7777**. The captured request flow confirms the live HTTP surface — for example `GET /` redirects (`302`) to `/auth/login`, `GET /auth/login` returns `200`, `POST /auth/login` redirects (`302`) to `/dashboard/`, `GET /dashboard/` returns `200`, and `GET /api/user_info` returns `200` (good key) or `401` (bad key):

```
127.0.0.1 GET / ImmutableMultiDict([]) 302, takes ...
127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes ...
127.0.0.1 POST /auth/login ImmutableMultiDict([]) 302, takes ...
127.0.0.1 GET /dashboard/ ImmutableMultiDict([]) 200, takes ...
127.0.0.1 GET /api/user_info ImmutableMultiDict([]) 200, takes ...
127.0.0.1 GET /api/user_info ImmutableMultiDict([]) 401, takes ...
```

### (b) Why — code rationale

- **Port 7777.** Set by `app.run(..., port=7777)` (`server.py:L588`) in dev; `EXPOSE 7777` (`Dockerfile:L44`) and `-b 0.0.0.0:7777` (`Dockerfile:L47`) in prod.
- **Blueprints** are registered in `register_blueprints()` at `server.py:L233-246`:
  - `auth_bp` (`server.py:L234`), `monitor_bp` (`server.py:L235`), `dashboard_bp` (`server.py:L236`), `developer_bp` (`server.py:L237`), `phone_bp` (`server.py:L238`).
  - `oauth_bp` is mounted **twice**, at `/oauth` (`server.py:L240`) **and** `/oauth2` (`server.py:L241`).
  - `onboarding_bp` (`server.py:L242`), `discover_bp` (`server.py:L244`), `internal_bp` (`server.py:L245`).
  - `api_bp` (`server.py:L246`), whose `url_prefix="/api"` is defined at `app/api/base.py:L11`.
- **Direct routes and mounts** (registered inside `create_app()`):
  - `/health` → `"success", 200` (`server.py:L213-215`).
  - `/` index, which redirects to `dashboard.index` when authenticated, else `auth.login` (`set_index_page`, `server.py:L249-255`).
  - `/.well-known/openid-configuration` (`server.py:L300`) and `/jwks` (`server.py:L325`) from `setup_openid_metadata` (`server.py:L299-329`).
  - The **Flask-Admin** console, mounted at the default path **`/admin`**, set up by `init_admin` (`server.py:L441-458`): `Admin(name="SimpleLogin", template_mode="bootstrap4")` (`server.py:L442`) is attached via `admin.init_app(app, index_view=SLAdminIndexView())` (`server.py:L444`), followed by many `add_view(...)` registrations. Because no `url=` argument is passed to `Admin(...)`, Flask-Admin (`1.5.7`, pinned) uses its **default `/admin` mount**; this is corroborated in-code by `SLAdminIndexView.index` redirecting to `/admin/user` (`app/admin_model.py:L116`) and by the `/admin/static` path guards in the request hooks (`server.py:L262`, `L277`).
  - `/favicon.ico` → redirect to `/static/favicon.ico` (`setup_favicon_route`, `server.py:L397-400`).
  - Payment webhooks, defined inside helper functions invoked by `create_app()`: `/paddle` (`@app.route("/paddle", methods=["GET", "POST"])` at `app/payments/paddle.py:L25`) and `/paddle_coupon` (`@app.route("/paddle_coupon", methods=["GET", "POST"])` at `app/payments/paddle.py:L260`), both wired up by `setup_paddle_callback(app)` (`server.py:L180`); and `/coinbase` (`@app.route("/coinbase", methods=["POST"])` at `app/payments/coinbase.py:L19`), wired up by `setup_coinbase_commerce(app)` (`server.py:L181`).
  - `/dnt` (`setup_do_not_track`, `server.py:L553-555`).
  - `/static` assets (Flask's default static route).
- **CORS** is enabled for `/api/*` via `CORS(app, resources={r"/api/*": {"origins": "*"}})` (`server.py:L200`).


---

## O5 — Single authenticated request lifecycle

This is the user's **primary concern**: where a request first enters the application, how the user identity is determined at runtime, and how the authentication context is propagated through request handling. SimpleLogin uses a **dual-path** identity model — a browser/session path and an API-key path.

### (a) Observed

The captured run exercises one authenticated **web** request and one authenticated **API** request (full transcript in [Appendix A](#appendix-a--verbatim-captured-output)):

```
=== R2: GET / (anonymous) ===
status=302 location=http://localhost:7777/auth/login
=== R3: GET /auth/login (anonymous) ===
status=200
csrf extracted len=91
=== R4: POST /auth/login (john@wick.com / password) ===
status=302 location=http://localhost:7777/dashboard/
cookie jar (names):
by slapp
=== R5: GET /dashboard/ (AUTHENTICATED web request, session cookie) ===
status=200
=== R6: GET /api/user_info (AUTHENTICATED API, header Authentication: code) ===
{
  "can_create_reverse_alias": true,
  "connected_proton_address": null,
  "email": "john@wick.com",
  "in_trial": false,
  "is_premium": true,
  "max_alias_free_plan": 5,
  "name": "John Wick",
  "profile_picture_url": "http://localhost:7777/static/upload/profile_pic.svg"
}
status=200
=== R7: GET /api/user_info (BAD key -> expect 401) ===
{
  "error": "Wrong api key"
}
status=401
```

Observed facts: anonymous `GET /` is redirected to the login page; a successful `POST /auth/login` sets a cookie named **`slapp`** and redirects to `/dashboard/`; the subsequent `GET /dashboard/` with that cookie returns `200` (the **web-session** path); `GET /api/user_info` with a valid `Authentication` header returns `200` and the user's JSON profile (the **API-key** path); the same call with a bad key returns `401 {"error": "Wrong api key"}`. The cookie name `slapp` matches `SESSION_COOKIE_NAME = "slapp"` (`app/config.py:L199`).

The corresponding server-side `after_request` lines (note all carry pid 23, the reloader child) are:

```
2026-06-26 19:47:39,025 - SL - DEBUG - 23 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET / ImmutableMultiDict([]) 302, takes ...
2026-06-26 19:47:39,146 - SL - DEBUG - 23 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes ...
2026-06-26 19:47:39,406 - SL - DEBUG - 23 - "/app/app/auth/views/login_utils.py:35" - after_login() -  - log user <User 1 John Wick john@wick.com> in
2026-06-26 19:47:39,406 - SL - DEBUG - 23 - "/app/app/auth/views/login_utils.py:44" - after_login() -  - redirect user to dashboard
2026-06-26 19:47:39,407 - SL - DEBUG - 23 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 302, takes ...
2026-06-26 19:47:39,764 - SL - DEBUG - 23 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard/ ImmutableMultiDict([]) 200, takes ...
2026-06-26 19:47:39,790 - SL - DEBUG - 23 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /api/user_info ImmutableMultiDict([]) 200, takes ...
2026-06-26 19:47:39,800 - SL - DEBUG - 23 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /api/user_info ImmutableMultiDict([]) 401, takes ...
```

### (b) Why — code rationale

**Entry.** Every request enters through the WSGI callable `wsgi:app` (= `create_app()`, `wsgi.py:L1-3`) wrapped by `ProxyFix(app.wsgi_app, x_for=1, x_host=1)` (`server.py:L142`), which normalizes `X-Forwarded-For`/`X-Forwarded-Host` from the NGINX proxy, then proceeds into Flask routing. In dev, the same `create_app()` output is served directly by Werkzeug via `app.run` (`server.py:L588`).

**`before_request` hooks (two of them).**
1. `make_session_permanent` sets `session.permanent = True` and `app.permanent_session_lifetime = timedelta(days=7)` (`server.py:L204-207`), giving authenticated sessions a 7-day lifetime.
2. A second `before_request` (`server.py:L257-270`) records `g.start_time = time.time()` (`server.py:L265`) for request-duration logging and, if the URL has a `?slref=<code>` referral parameter, stores it in `session["slref"]` (`server.py:L268-270`). It skips `/static`, `/admin/static`, and `/_debug_toolbar` paths (`server.py:L260-264`).

**Identity path (a) — browser/session via Flask-Login.** When a request carries the `slapp` session cookie, Flask-Login resolves the user through the registered `user_loader`:

```python
# server.py:L220-230
@login_manager.user_loader
def load_user(alternative_id):
    user = User.get_by(alternative_id=alternative_id)
    if user:
        sentry_sdk.set_user({"email": user.email, "id": user.id})
        if user.disabled:
            return None
        if not user.is_active():
            return None
    return user
```

The critical detail is **what the session stores as the identity key**. Flask-Login calls `User.get_id()` to serialize identity, and SimpleLogin overrides it to return the UUID `alternative_id`, not the numeric primary key:

```python
# app/models.py:L594-599  (comment: implement flask-login "alternative token")
def get_id(self):
    if self.alternative_id:
        return self.alternative_id
    else:
        return str(self.id)
```

`alternative_id` is a unique `String(128)` column (`app/models.py:L482`) generated at user creation as `str(uuid.uuid4())` (`app/models.py:L617`, only when not explicitly provided — `if "alternative_id" not in kwargs`, `app/models.py:L616`). Storing the **UUID rather than the PK** means a server-side session can be invalidated by rotating the user's `alternative_id` — every previously issued cookie then fails to resolve to a user. The `user_loader` also defensively returns `None` for disabled or inactive users (`server.py:L225-228`), and tags the Sentry scope with the user (`server.py:L224`).

**Identity path (b) — API key via the `Authentication` header.** API routes call `authorize_request()`:

```python
# app/api/base.py:L16-43
def authorize_request() -> Optional[Tuple[str, int]]:
    api_code = request.headers.get("Authentication")     # L17
    api_key = ApiKey.get_by(code=api_code)               # L18
    if not api_key:
        if current_user.is_authenticated:                # L21
            g.user = current_user                        # L25  (fallback to the session user)
        else:
            return jsonify(error="Wrong api key"), 401   # L26-27
    else:
        api_key.last_used = arrow.now()                  # L30
        api_key.times += 1                               # L31
        Session.commit()                                 # L32
        g.user = api_key.user                            # L34
    if g.user.disabled:
        return jsonify(error="Disabled account"), 403    # L36-37
    if not g.user.is_active():
        return jsonify(error="Account does not exist"), 401  # L39-40
    g.api_key = api_key                                  # L42
    return None
```

So an API request is identified by the **`Authentication` request header** (`app/api/base.py:L17`); the key is looked up (`app/api/base.py:L18`), its usage stats are updated and committed (`app/api/base.py:L30-32`), and `g.user` is set to the key's owner (`app/api/base.py:L34`). If no key is supplied but the caller already has a valid session, the session user is used instead (`app/api/base.py:L20-25`); otherwise the request is rejected with `401 {"error": "Wrong api key"}` — exactly the body observed in R7 (`app/api/base.py:L26-27`). Disabled accounts get `403` (`app/api/base.py:L36-37`) and inactive ones `401` (`app/api/base.py:L39-40`). Endpoints opt in via the `require_api_auth` decorator (`app/api/base.py:L52-60`); privileged endpoints add `require_api_sudo`, which returns `440 {"error": "Need sudo"}` outside the sudo window (`app/api/base.py:L63-73`, `SUDO_MODE_MINUTES_VALID = 5` at `app/api/base.py:L13`).

**Context propagation.** The resolved identity is carried through the request via three mechanisms:
- **`flask.g`** holds `g.user` and `g.api_key` (set by `authorize_request`) and `g.start_time` (set by the second `before_request`). The helper `get_current_user()` (`server.py:L332-336`) prefers `g.user` and falls back to Flask-Login's `current_user`.
- **The Flask-Login `current_user` proxy** is available everywhere for the session path, with `session_protection = "strong"` (`app/extensions.py:L8`) for added cookie-theft resistance.
- **The session record** (the `slapp` cookie, optionally Redis-backed). When `MEM_STORE_URI` is set, `initialize_redis_services` swaps the session interface to `RedisSessionStore` (`server.py:L163-165` → `app/redis_services.py:L9-14`). Server-side session TTLs differ by authentication state: the permanent lifetime (7 days) for authenticated sessions, but only **300 seconds** for anonymous sessions — kept solely so the CSRF token survives (`app/session.py:L92`, `L95-96`).

**Per-request bookkeeping and teardown.** On the way out, `after_request` (`server.py:L272-296`) logs the structured `SL` request line (`server.py:L284-292`) and records a NewRelic custom event `HttpResponseStatus` with the status code (`server.py:L293-295`), while skipping noise paths including `/health` (`server.py:L275-281`). The SQLAlchemy scoped session is then removed per request by the `teardown_appcontext` hook `cleanup` → `Session.remove()` (`server.py:L209-211`). The rate-limiter key is computed per request as `userid:{current_user.id}` when authenticated, else `ip:{ip}` (`app/extensions.py:L14-19`), with the limiter disabled when `config.DISABLE_RATE_LIMIT` is set (`app/extensions.py:L26-28`).


---

## O6 — Background jobs and schedulers (multi-process architecture)

### (a) Observed

The development web process (`python3 server.py`) starts **no** schedulers, threads, or background loops. Its captured stdout contains only: import-time prints, the Flask banner, the reloader-doubled import block, the dev-toolbar warning, and per-request `after_request` lines (see [Appendix A](#appendix-a--verbatim-captured-output)). No worker startup, no periodic-task logging, and no SMTP-listener output appears — because none of that runs in the web process. Background work is launched by running **separate executables** (e.g. `python job_runner.py`, `python cron.py -j <job>`, `python event_listener.py`, `python email_handler.py`), each of which is its own process.

### (b) Why — code rationale

- **The web factory imports no workers and starts no concurrency primitives.** A scan of `server.py` finds no module-level import of `job_runner`, `event_listener`, `email_handler`, or `cron`, and no use of `threading`, `multiprocessing`, `asyncio`, or any scheduler (`APScheduler`/`BackgroundScheduler`). The only reference to `init_app` in `server.py` is a **lazy import inside the `flask dummy-data` CLI command** (`from init_app import add_sl_domains, add_proton_partner` at `server.py:L492`, within the command defined at `server.py:L490-497`) — that runs only when an operator explicitly invokes `flask dummy-data`, and is not part of the web boot or the request path.
- **All periodic / asynchronous / SMTP work lives in separate top-level `__main__` scripts**, each distinct from the web factory. *Most* of them — `job_runner.py`, `cron.py`, `init_app.py`, and `email_handler.py` (the last one only per processed message) — obtain a minimal Flask **app context** via `create_light_app()` (`server.py:L127-136`) so they can use the ORM. **`event_listener.py` is the exception: it builds no Flask app or app context at all** and does not call `create_light_app()` (detailed below):
  - **`job_runner.py`** — `from server import create_light_app` (`job_runner.py:L24`); under `if __name__ == "__main__":` (`job_runner.py:L329`) it runs an infinite loop `while True:` (`job_runner.py:L330`) that drains the `Job` table inside `with create_light_app().app_context():` (`job_runner.py:L332`), sleeping `time.sleep(10)` between passes (`job_runner.py:L347`).
  - **`cron.py`** — `from server import create_light_app` (`cron.py:L65`); under `__main__` (`cron.py:L1262`) it parses arguments with `argparse.ArgumentParser()` (`cron.py:L1264`) and runs the selected job inside `create_light_app().app_context()` (`cron.py:L1273`). It is driven by `crontab.yml`, whose entries invoke `python /code/cron.py -j <job>` for jobs such as `stats`, `delete_old_monitoring`, `check_custom_domain`, `check_hibp`, `notify_hibp`, `delete_logs`, `delete_old_data`, `poll_apple_subscription`, `notify_trial_end`, `notify_manual_subscription_end`, `notify_premium_end`, `delete_scheduled_users`, `send_undelivered_mails`, `clear_alias_audit_log`, and `clear_user_audit_log`.
  - **`event_listener.py`** — **does *not* import or call `create_light_app()`, and (unlike the other workers) builds no Flask app or app context.** It imports `from app.config import EVENT_LISTENER_DB_URI` (`event_listener.py:L5`), `from events.runner import Runner` (`event_listener.py:L8`), and `from events.event_source import DeadLetterEventSource, PostgresEventSource` (`event_listener.py:L9`). An `argparse.ArgumentParser(description="Run event listener")` (`event_listener.py:L69`) selects a mode under `__main__` (`event_listener.py:L94-113`); `main()` (`event_listener.py:L29-47`) constructs the event source — `PostgresEventSource(EVENT_LISTENER_DB_URI)` for the `listener` mode or `DeadLetterEventSource(...)` for `dead_letter` (`event_listener.py:L30-35`) — and a sink (`ConsoleEventSink`/`HttpEventSink`, `event_listener.py:L39-44`), then runs `Runner(source=source, sink=sink)` / `runner.run()` (`event_listener.py:L46-47`). It is a PostgreSQL event consumer (`listener` / `dead_letter` / `run` / `debug` modes) that reaches the database through its own source objects and a dedicated `EVENT_LISTENER_DB_URI`, rather than through the web app's ORM session.
  - **`email_handler.py`** — `import argparse` (`email_handler.py:L33`) and `from aiosmtpd.controller import Controller` (`email_handler.py:L48`); under `__main__` (`email_handler.py:L2396`) it starts an SMTP server with `Controller(MailHandler(), hostname="0.0.0.0", port=port)` (`email_handler.py:L2383`) and stays alive with `while True:` (`email_handler.py:L2392`). It imports `create_light_app` (`email_handler.py:L177`) but, unlike the loop-level workers above, enters the app context **per processed message** — `with create_light_app().app_context():` inside its message handler (`email_handler.py:L2352`) — rather than wrapping the whole process in a single context. This is an inbound SMTP forwarding server, entirely separate from the HTTP server.
  - **`init_app.py`** — `from server import create_light_app` (`init_app.py:L10`); under `__main__` (`init_app.py:L69`) it runs one-time initialization (`load_pgp_public_keys`, `add_sl_domains`, `add_proton_partner`) inside `with create_light_app().app_context():` (`init_app.py:L71`). It is a setup script, not a server.
- **The documentation agrees.** `CONTRIBUTING.md` lists the primary entry points as the webapp (`wsgi.py`/`server.py`, `CONTRIBUTING.md:L145`), `email_handler.py` (`CONTRIBUTING.md:L146`), and `cron.py` (`CONTRIBUTING.md:L147`), and separately documents running the job runner via `python job_runner.py` (`CONTRIBUTING.md:L228`).

**Conclusion for O6:** the dev server serves HTTP only. Schedulers and asynchronous/SMTP work are deliberately *out-of-process* — they are started by running the dedicated scripts above, not by the web bootstrap.

---

## O7 — Other runtime activity at startup/during requests

### (a) Observed

Beyond serving HTTP, the most consequential startup activity is **establishing a database connection at import time**. This is invisible in stdout when it succeeds, and it is a hard prerequisite for reaching readiness — but it is *not* the very first side effect of startup. The import-time **config prints come first** (`>>> URL:` and the four others), then **`>>> init logging <<<`**, and only *after* those does `app.db` open its connection (`app/db.py:L12`). So if Postgres is unreachable, the process fails during import **after** `>>> URL:` and `>>> init logging <<<` have already printed, but **before** app creation, the Flask banner, and readiness (the exact import order is traced in (b) below). In the successful canonical capture, the database connect happened silently and startup proceeded to readiness. No Sentry/profiler lines appear because those features are not enabled in the dev `.env`.

### (b) Why — code rationale

- **Eager, import-time database connection.** `app/db.py` builds the engine and **opens a connection immediately at import**:
  ```python
  # app/db.py:L9-14
  engine = create_engine(
      config.DB_URI, connect_args={"application_name": config.DB_CONN_NAME}
  )
  connection = engine.connect()
  Session = scoped_session(sessionmaker(bind=connection))
  ```
  `engine.connect()` (`app/db.py:L12`) runs as soon as `app.db` is first imported in the module graph, which is why a reachable PostgreSQL instance is a precondition for the server to even reach readiness. The scoped session created here (`app/db.py:L14`) is the one torn down per request by `Session.remove()` (`server.py:L211`).
- **Where the DB connect falls in the import order (and why it is *not* the first side effect).** Running `python3 server.py`, imports execute top-to-bottom. `server.py:L30` (`from app import config, constants`) evaluates `app/config.py` first, emitting `>>> URL:` (`app/config.py:L80`) and the four other config prints. Next, `server.py:L31` (`from app.admin_model import (...)`) reaches `app/admin_model.py:L11` (`from app import models, s3`); inside `app/models.py`, the line `from app import config, rate_limiter` (`app/models.py:L30`) imports `app/rate_limiter.py`, whose `from app.log import LOG` (`app/rate_limiter.py:L9`) causes `app/log.py` to print `>>> init logging <<<` (`app/log.py:L67`). Only *then* — two lines later, at `app/models.py:L32` (`from app.db import Session`) — is `app/db.py` imported and `engine.connect()` (`app/db.py:L12`) executed. (`app/db.py` itself opens with `from app import config` at `app/db.py:L6`, so configuration is necessarily fully loaded — and `>>> URL:` already printed — before any connection is attempted.) This ordering matches the canonical capture in [Appendix A.1](#a1--canonical-dev-startup-python3-serverpy-real-env-color_log-unset): `>>> URL:` and `>>> init logging <<<` precede everything else, while the connection itself emits no line of its own.
- **Optional Sentry error monitoring.** `sentry_sdk.init(...)` runs only when `SENTRY_DSN` is set (`server.py:L111-121`); it is not required to boot and is absent from the dev capture.
- **Rate limiter.** A Flask-Limiter `Limiter(key_func=__key_func)` is constructed (`app/extensions.py:L23`) with the per-user/per-IP key function (`app/extensions.py:L14-19`) and attached to the app via `limiter.init_app(app)` (`server.py:L167`); it is disabled when `config.DISABLE_RATE_LIMIT` is set (`app/extensions.py:L26-28`).
- **`OAUTHLIB_INSECURE_TRANSPORT=1` is set at import.** `os.environ["OAUTHLIB_INSECURE_TRANSPORT"] = "1"` (`server.py:L124`) lets the OAuth flows work over plain HTTP locally (the app is normally fronted by NGINX terminating TLS). This is a development convenience and is **not** appropriate for a direct production deployment.
- **Optional profiler and per-response NewRelic event.** `flask_profiler` is initialized only if `FLASK_PROFILER_PATH` is set (`server.py:L185-197`); on every non-skipped response, `after_request` records a NewRelic custom event `HttpResponseStatus` (`server.py:L293-295`).


---

## Appendix A — Verbatim captured output

All output below was captured from a real run inside the provided container image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`. Timestamps, process ids (parent **7**, reloader child **23**), and the random `GNUPGHOME` temp paths are part of the literal capture and are preserved exactly.

### A.1 — Canonical dev startup (`python3 server.py`, real `.env`, `COLOR_LOG` unset)

Full captured stdout to readiness plus the per-request lines:

```
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/ficswppbuayxvsommeoy
Upload files to local dir
>>> init logging <<<
2026-06-26 19:47:35,485 - SL - DEBUG - 7 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/kinlkwkzyfxmxndnlanu
Upload files to local dir
>>> init logging <<<
2026-06-26 19:47:38,188 - SL - DEBUG - 23 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
/app/venv/lib/python3.10/site-packages/flask_debugtoolbar/__init__.py:213: UserWarning: Could not insert debug toolbar. </body> tag not found in response.
  warnings.warn('Could not insert debug toolbar.'
2026-06-26 19:47:39,025 - SL - DEBUG - 23 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET / ImmutableMultiDict([]) 302, takes 0.0008857250213623047
2026-06-26 19:47:39,146 - SL - DEBUG - 23 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.1112067699432373
2026-06-26 19:47:39,406 - SL - DEBUG - 23 - "/app/app/auth/views/login_utils.py:35" - after_login() -  - log user <User 1 John Wick john@wick.com> in
2026-06-26 19:47:39,406 - SL - DEBUG - 23 - "/app/app/auth/views/login_utils.py:44" - after_login() -  - redirect user to dashboard
2026-06-26 19:47:39,407 - SL - DEBUG - 23 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 302, takes 0.2468245029449463
2026-06-26 19:47:39,764 - SL - DEBUG - 23 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard/ ImmutableMultiDict([]) 200, takes 0.3451509475708008
2026-06-26 19:47:39,790 - SL - DEBUG - 23 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /api/user_info ImmutableMultiDict([]) 200, takes 0.01493525505065918
2026-06-26 19:47:39,800 - SL - DEBUG - 23 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /api/user_info ImmutableMultiDict([]) 401, takes 0.002149820327758789
```

Reading guide: lines 1–6 are import-time `print()`s; line 7 is the first `SL` log (loading the words file) under the parent pid **7**; lines 8–12 are the Flask click banner; lines 13–19 are the **exact repeat** emitted by the reloader child pid **23**; lines 20–21 are the dev-only Flask-DebugToolbar `UserWarning`; the remaining lines are per-request `after_request` `SL` lines (all pid 23). There is **no** `* Running on ...` line and **no** debugger-PIN line.

### A.2 — Request outcomes (captured separately)

```
READY=1 after 4s (via GET /health == 200)
=== R2: GET / (anonymous) ===
status=302 location=http://localhost:7777/auth/login
=== R3: GET /auth/login (anonymous) ===
status=200
csrf extracted len=91
=== R4: POST /auth/login (john@wick.com / password) ===
status=302 location=http://localhost:7777/dashboard/
cookie jar (names):
by slapp
=== R5: GET /dashboard/ (AUTHENTICATED web request, session cookie) ===
status=200
=== R6: GET /api/user_info (AUTHENTICATED API, header Authentication: code) ===
{
  "can_create_reverse_alias": true,
  "connected_proton_address": null,
  "email": "john@wick.com",
  "in_trial": false,
  "is_premium": true,
  "max_alias_free_plan": 5,
  "name": "John Wick",
  "profile_picture_url": "http://localhost:7777/static/upload/profile_pic.svg"
}
status=200
=== R7: GET /api/user_info (BAD key -> expect 401) ===
{
  "error": "Wrong api key"
}
status=401
```

The cookie name `slapp` matches `SESSION_COOKIE_NAME = "slapp"` (`app/config.py:L199`); the `401 {"error": "Wrong api key"}` body matches `app/api/base.py:L26-27`.

### A.3 — `COLOR_LOG=true` variant (note the timestamp format difference)

```
2026-06-26 19:49:38 - SL - DEBUG - 7 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/aytfrusjocvlpasyswzq
Upload files to local dir
>>> init logging <<<
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
2026-06-26 19:49:41 - SL - DEBUG - 23 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
/app/venv/lib/python3.10/site-packages/flask_debugtoolbar/__init__.py:213: UserWarning: Could not insert debug toolbar. </body> tag not found in response.
  warnings.warn('Could not insert debug toolbar.'
2026-06-26 19:49:42 - SL - DEBUG - 23 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.10759758949279785
```

With `COLOR_LOG` unset (the `example.env` default) the `SL` timestamps include milliseconds (`19:47:35,485`); with `COLOR_LOG=true` present at import, `coloredlogs.install()` runs (`app/log.py:L61-62`) and timestamps are reformatted without milliseconds (`19:49:38`), with ANSI colors applied only on a TTY (none here, since output is redirected to a file/pipe). Ordering between plain `print()` and `SL` logging output can interleave depending on stdout buffering (pipe vs TTY); the A.1 capture (`server_run.log`) is the canonical reference.

### A.4 — Production contrast (`gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15`)

```
[2026-06-26 19:50:44 +0000] [7] [INFO] Starting gunicorn 20.0.4
[2026-06-26 19:50:44 +0000] [7] [INFO] Listening at: http://0.0.0.0:7777 (7)
[2026-06-26 19:50:44 +0000] [7] [INFO] Using worker: sync
[2026-06-26 19:50:44 +0000] [12] [INFO] Booting worker with pid: 12
[2026-06-26 19:50:44 +0000] [13] [INFO] Booting worker with pid: 13
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/tugsmeqpalhlnyddxgcv
Upload files to local dir
>>> init logging <<<
2026-06-26 19:50:45,082 - SL - DEBUG - 13 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/ufgqzxvwawvivbmtccxb
Upload files to local dir
>>> init logging <<<
2026-06-26 19:50:45,083 - SL - DEBUG - 12 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-06-26 19:50:46,930 - SL - DEBUG - 12 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.019021272659301758
```

In production, **gunicorn's own logger** prints the readiness signal `Listening at: http://0.0.0.0:7777` — this is **not** suppressed, because it belongs to gunicorn, not to the disabled `werkzeug` logger. The app's import-time prints appear **once per worker** (pids **12** and **13** — the two `-w 2` sync workers each importing the app, which is different from dev's single-tree parent+reloader doubling). There is **no** Flask `* Serving Flask app / * Environment / * Debug mode` banner, because `app.run()` is never called in production. The `after_request` `SL` line is still emitted on requests.

### A.5 — Footnote: the `CONFIG`-file startup artifact

An earlier startup-only capture used `CONFIG=/work/run.env` and therefore *also* printed a leading line `load config file /work/run.env` (`app/config.py:L68`). The documented `.env` flow — the `else` branch `load_dotenv()` at `app/config.py:L71` — does **not** print that line. The canonical `.env`-based capture in A.1 (no `CONFIG`) is authoritative; this artifact is noted only to explain the conditional print at `app/config.py:L68`.

---

## Appendix B — Evidence index

Every factual claim in this document maps to a source location below (and, where it concerns printed output, to the captures in Appendix A).

| Area | Citation |
|---|---|
| Dev entry / `__main__` | `server.py:L598-599` |
| `local_main()` / `app.run(debug=True, port=7777)` | `server.py:L572-595`, `server.py:L588` |
| `create_app()` factory | `server.py:L139-217` |
| `ProxyFix` | `server.py:L142` |
| `secret_key = FLASK_SECRET` | `server.py:L151` |
| Blueprints | `server.py:L233-246` |
| `/health` | `server.py:L213-215` |
| Index redirect | `server.py:L249-255` |
| Flask-Admin `/admin` mount (default; no `url=`) | `server.py:L442`, `L444`; corroborated `app/admin_model.py:L116`, `server.py:L262`, `L277` |
| Payment webhooks `/paddle`, `/paddle_coupon` | `app/payments/paddle.py:L25`, `L260` (registered via `server.py:L180`) |
| Payment webhook `/coinbase` | `app/payments/coinbase.py:L19` (registered via `server.py:L181`) |
| `before_request` (session permanent, 7-day) | `server.py:L204-207` |
| `before_request` (`g.start_time`, `slref`) | `server.py:L257-270` |
| `after_request` (SL log + NewRelic) | `server.py:L272-296`, esp. `L284-292`, `L293-295` |
| teardown → `Session.remove()` | `server.py:L209-211` |
| Flask-Login `user_loader` | `server.py:L220-230` |
| `User.get_id()` / `alternative_id` | `app/models.py:L595-599`, `L482`, `L616-617` |
| API auth (`Authentication` header) | `app/api/base.py:L16-43`, header at `L17`, 401 at `L26-27` |
| API decorators (`require_api_auth` / `require_api_sudo`) | `app/api/base.py:L52-60`, `L63-73` |
| `OAUTHLIB_INSECURE_TRANSPORT` | `server.py:L124` |
| Sentry optional | `server.py:L111-121` |
| `create_light_app()` definition (workers only; never the web path) | `server.py:L127-136` |
| Workers that use the `create_light_app()` app context | `job_runner.py:L24`, `L332`; `cron.py:L65`, `L1273`; `init_app.py:L10`, `L71`; `email_handler.py:L177`, `L2352` (per message) |
| `event_listener.py` builds NO Flask app (does not call `create_light_app()`) | `event_listener.py:L5`, `L8-9`, `L29-47`, `L94-113` |
| `wsgi:app` | `wsgi.py:L1-3` |
| Config import-time / `>>> URL:` | `app/config.py:L71`, `L79-80` |
| `FLASK_SECRET` required | `app/config.py:L196-198` |
| `SESSION_COOKIE_NAME = "slapp"` | `app/config.py:L199` |
| `>>> init logging <<<` / werkzeug logger disabled | `app/log.py:L67`, `L70-71` |
| SL log format / logger name | `app/log.py:L12-15`, `L79` |
| Eager DB connect | `app/db.py:L9-14` |
| Flask-Login strong protection | `app/extensions.py:L8` |
| Limiter key function | `app/extensions.py:L14-19`, `L23`, `L26-28` |
| Redis session activation | `app/redis_services.py:L9-14` |
| Session TTLs (7 days / 300 s) | `app/session.py:L92`, `L95-96` |
| Prod CMD / port / base image | `Dockerfile:L47`, `L44`, `L8` |
| Dev run command / login creds | `CONTRIBUTING.md:L106`, `L109` |
| Worker entry points | `job_runner.py:L24`, `L329-347`; `cron.py:L65`, `L1262-1273`; `event_listener.py:L5`, `L8-9`, `L29-47`, `L94-113`; `email_handler.py:L48`, `L177`, `L2352`, `L2383-2396`; `init_app.py:L10`, `L69-71` |
| `dummy-data` lazy `init_app` import | `server.py:L490-497` (import at `L492`) |
| Version pins | `pyproject.toml:L61-117`; `poetry.lock` (werkzeug 1.0.1, flask 1.1.2, click 8.0.3) |

---

*This document is the sole artifact of this task. The SimpleLogin source tree was investigated read-only; no source file was created, modified, or deleted, and all temporary run artifacts (the throwaway `.env` and helper scripts) were removed afterward.*

