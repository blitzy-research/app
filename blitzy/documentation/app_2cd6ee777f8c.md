# SimpleLogin — Runtime Verification of the Web Server, Email Handler, and Job Runner

This document answers three question groups about the SimpleLogin back-end by **running the code first and writing from what was observed**. Every behavioral claim is paired with the exact command that produced it and the complete, unedited output, plus a `file:line` citation into the source under test. Anything not directly observed is explicitly labeled **INFERRED**.

- **Q1 — Startup / Health:** how to tell the web server, email handler, and job runner are up and responding, and what confirms in the logs / dashboard UI that users can sign in and manage aliases.
- **Q2 — User actions & correctness:** create an account, create an alias, and have that alias receive an email; what happens and how the system shows it was handled correctly.
- **Q3 — Background components:** whether the email handler and job runner come online in the background, and what behavior shows they are functioning across situations.

## Legend & Ground Rules

- **OBSERVED** — the statement is backed by captured runtime output shown in this document.
- **INFERRED** — the statement is read from source (with a `file:line` citation) but was not directly executed in this pass; preferred only where running is not applicable.
- **Redaction:** one-time secrets (activation codes) are replaced with `<activation_code_redacted>`. See §3.2. CSRF tokens are per-request, single-use nonces that expire immediately; they are shown unedited as captured because they are not reusable credentials.
- **Public fixture:** `john@wick.com` / `password` is the **public demo login seeded by `flask dummy-data`** (OBSERVED in §1); it is not a secret and is used only to render the authenticated dashboard in §2.5.
- **Read-only scope:** no source file was modified. The only artifact produced is this document. Temporary accounts, aliases, emails, jobs, and observation scripts were created solely to observe behavior and were removed afterward (§6).

## Environment Identity (the exact code and runtime under test)

- **Runtime image (mandated by setup instructions):** alias `andrewparkscaleai/coding-agent:simple-login__app__2cd6ee772d3531559588bcfb18627ffb5d2c`.
- **OBSERVED image RepoTag actually running** (`docker inspect`): `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`.
- **OBSERVED source commit under test:** the image's `/app` git `HEAD` is `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c` — the immutable source-parent commit of this branch (the exact code being documented). Every commit this branch adds sits on top of `2cd6ee777f8c...` and changes only this documentation file: `git diff --name-only 2cd6ee777f8c2d3531559588bcfb18627ffb5d2c HEAD` lists exactly `blitzy/documentation/app_2cd6ee777f8c.md` and nothing else (absolute-hash references are used deliberately so this identity claim stays correct regardless of how many documentation commits are later stacked on the branch). The image therefore ships the exact code being documented.
- **Runtime:** Python 3.10.18 in the image's virtualenv at `/app/venv`. Key pins (from `poetry.lock`): Flask 1.1.2, Flask-Login 0.5.0, gunicorn 20.0.4, SQLAlchemy 1.3.24, aiosmtpd 1.4.2, redis 4.6.0, yacron 0.11.2.
- **Backing services:** PostgreSQL 13 (`sl-db`) and Redis 7 (`sl-redis`) on a shared Docker network; the app container is `sl-app`.

---

# 1. Environment & Setup

This section records the canonical build/run environment so every value reported downstream is a default, non-artifact value.

**Image alias, running RepoTag, and container topology** (OBSERVED):

```
$ docker inspect sl-app --format '{{json .Config.Image}}'   # via image RepoTags
["ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0"]

$ docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Ports}}'
NAMES      IMAGE                                                           PORTS
sl-app     ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0   0.0.0.0:7777->7777/tcp, 0.0.0.0:20381->20381/tcp
sl-redis   redis:7                                                         0.0.0.0:6379->6379/tcp
sl-db      postgres:13                                                     0.0.0.0:15432->5432/tcp
```

**Canonical config — copy the template and apply only the local network deltas** (OBSERVED). The app reads configuration from `.env` in the working directory via `load_dotenv()` unless `CONFIG` names an explicit file (`app/config.py:L65-L71`); `DB_URI` is required (`app/config.py:L192`). The container's `.env` is a verbatim copy of the repo-tracked `example.env` with only two gitignored, environment-specific lines changed — the Postgres host (`localhost` → `sl-db`) and an appended Redis URI:

```
$ cp example.env .env
$ diff example.env .env
75c75
< DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin
---
> DB_URI=postgresql://myuser:mypassword@sl-db:5432/simplelogin
198a199
> MEM_STORE_URI=redis://sl-redis:6379
```

The `.env` file is gitignored and is never committed; the tracked `example.env` is untouched.

**Database migration + seed data** (OBSERVED). Migrations are applied with Alembic and the local login fixtures are seeded with `flask dummy-data`:

```
$ FLASK_APP=server.py /app/venv/bin/flask db upgrade   # alembic upgrade head (idempotent)
Upload files to local dir
>>> init logging <<<
2026-07-08 05:14:35,141 - SL - DEBUG - 1628 - "/code/app/utils.py:17" - <module>() -  - load words file: /code/local_data/test_words.txt
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.

$ /app/venv/bin/alembic current
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
32f25cbf12f6 (head)

$ psql -h sl-db -U myuser -d simplelogin -c "SELECT id,email,activated FROM users ORDER BY id;"
 id |          email          | activated 
----+-------------------------+-----------
  1 | john@wick.com           | t
  2 | winston@continental.com | t
(2 rows)
```

The schema is at head revision `32f25cbf12f6`, and the seed created two **public demo logins**, `john@wick.com` (id 1) and `winston@continental.com` (id 2), both already activated. `john@wick.com` / `password` is used only in §2.5 to render the authenticated dashboard.

**Exact invocation commands** used throughout (each component is its own real entry point):

| Component | Command | Port | Source of default |
|---|---|---|---|
| Web (production) | `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` | 7777 | `Dockerfile:L47` (container default `CMD`) |
| Web (development) | `python server.py` | 7777 | `server.py:L588` — `app.run(debug=True, port=7777)` |
| Email handler | `python email_handler.py` | 20381 | `email_handler.py:L2381` / `:L2398-L2403` (argparse default) |
| Job runner | `python job_runner.py` | — | `job_runner.py:L329-L347` (infinite poll loop) |

---

# 2. Q1 — Startup & Health Verification

## 2.1 Web server (production, gunicorn) — the canonical default

**Startup log** (OBSERVED). Started with the container's default command `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` (`Dockerfile:L47`). gunicorn prints its master banner, the bound address, and one line per worker; the app then prints its own boot banner (`>>> init logging <<<`, `app/log.py:L67`) once per worker:

```
$ cat /tmp/web.log
[2026-07-08 05:15:08 +0000] [1679] [INFO] Starting gunicorn 20.0.4
[2026-07-08 05:15:08 +0000] [1679] [INFO] Listening at: http://0.0.0.0:7777 (1679)
[2026-07-08 05:15:08 +0000] [1679] [INFO] Using worker: sync
[2026-07-08 05:15:08 +0000] [1680] [INFO] Booting worker with pid: 1680
[2026-07-08 05:15:08 +0000] [1681] [INFO] Booting worker with pid: 1681
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/vukiarwscbicywwjqawf
Upload files to local dir
>>> init logging <<<
2026-07-08 05:15:09,486 - SL - DEBUG - 1680 - "/code/app/utils.py:17" - <module>() -  - load words file: /code/local_data/test_words.txt
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/whdrxyryaumwlkhnhnae
Upload files to local dir
>>> init logging <<<
2026-07-08 05:15:09,543 - SL - DEBUG - 1681 - "/code/app/utils.py:17" - <module>() -  - load words file: /code/local_data/test_words.txt
```

**Up-signals in this output** (OBSERVED): `Starting gunicorn 20.0.4`, `Listening at: http://0.0.0.0:7777 (1679)`, and two workers booted (`pid: 1680`, `pid: 1681`). The master PID is **1679**; workers are **1680** and **1681**. The `>>> init logging <<<` banner is emitted by `app/log.py:L67`; `MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value` reflects the free-plan alias limit default of 5 (`app/config.py:L124`), which §3.6 exercises.

**Health endpoint** (OBSERVED). The health route `@app.route("/health")` (`server.py:L213`) is served by the `healthcheck()` view function (`server.py:L214`), whose body is `return "success", 200` (`server.py:L215`):

```
$ curl -i http://localhost:7777/health
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 05:16:01 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 7
Set-Cookie: slapp=4a5844c0-1a99-43cc-b934-451eb4e75dce.9GgcHBRWUdAeVQoXt_kc8ySjuQA; Expires=Wed, 15-Jul-2026 05:16:01 GMT; HttpOnly; Path=/; SameSite=Lax

success
```

The response is `HTTP/1.1 200 OK`, `Server: gunicorn/20.0.4`, body `success` (`Content-Length: 7`). This is the single most direct "web server is up" signal.

**Index redirect** (OBSERVED). The index route `@app.route("/")` (`server.py:L250-L251`) redirects anonymous visitors to the login page:

```
$ curl -i http://localhost:7777/
HTTP/1.1 302 FOUND
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 05:16:01 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 229
Location: http://localhost:7777/auth/login
Set-Cookie: slapp=582519c8-ae20-4273-af71-1dfb577110cc.If32hkVWwoE5H0gAaf6855lF4h0; Expires=Wed, 15-Jul-2026 05:16:01 GMT; HttpOnly; Path=/; SameSite=Lax

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<title>Redirecting...</title>
<h1>Redirecting...</h1>
<p>You should be redirected automatically to target URL: <a href="/auth/login">/auth/login</a>.  If not click the link.
```

`HTTP/1.1 302 FOUND` with `Location: http://localhost:7777/auth/login` confirms the routing layer is live and that unauthenticated users land on the sign-in page.

## 2.2 Web server (development mode) — contrast with production

**Reasoning (cause → effect):** `server.py`'s `__main__` calls `app.run(debug=True, port=7777)` (`server.py:L588`), which starts Flask's built-in Werkzeug development server (with the debug reloader) instead of gunicorn. The observable difference is the HTTP protocol version and `Server` header.

**Dev-server startup + health** (OBSERVED). After stopping gunicorn to free port 7777:

```
$ python server.py     # /tmp/devserver.log (startup, unedited)
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/tvdwpdezjgxpswemkyrn
Upload files to local dir
>>> init logging <<<
2026-07-08 05:40:16,857 - SL - DEBUG - 2264 - "/code/app/utils.py:17" - <module>() -  - load words file: /code/local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/qhvosjxedgakfoafjfki
Upload files to local dir
>>> init logging <<<
2026-07-08 05:40:18,834 - SL - DEBUG - 2270 - "/code/app/utils.py:17" - <module>() -  - load words file: /code/local_data/test_words.txt

$ curl -i http://127.0.0.1:7777/health
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 7
Set-Cookie: slapp=ae10a589-827e-4217-b29a-194f31f2ceaf.194DA_fgxlmUWSfNqV1MucVcPTE; Expires=Wed, 15-Jul-2026 05:40:22 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Wed, 08 Jul 2026 05:40:22 GMT

success
```

**Distinction** (OBSERVED): dev mode emits the Werkzeug banner (`* Serving Flask app "server"`, `* Debug mode: on`, the "development server" warning) and answers `/health` as `HTTP/1.0 200 OK` with `Server: Werkzeug/1.0.1 Python/3.10.18`, whereas production answers `HTTP/1.1 200 OK` with `Server: gunicorn/20.0.4` (§2.1). The debug reloader forks a child process (parent 2264, reloaded child 2270). gunicorn was restored afterward for the remaining observations.

## 2.3 Email handler startup

**Startup log** (OBSERVED). Started as its own process with `python email_handler.py`. `main(port)` builds an aiosmtpd `Controller(MailHandler(), hostname="0.0.0.0", port=port)` (`email_handler.py:L2381,L2383`), calls `controller.start()` (`:L2385`) then logs the controller and the bound port:

```
$ cat /tmp/eh.log
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/fnjdrbsxwimcbtcpikrz
Upload files to local dir
>>> init logging <<<
2026-07-08 05:16:31,217 - SL - DEBUG - 1805 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-08 05:16:31,811 - SL - INFO - 1805 - "/app/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-08 05:16:31,813 - SL - DEBUG - 1805 - "/app/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

**Up-signals** (OBSERVED): `Listen for port 20381` (`email_handler.py:L2403`) and `Start mail controller 0.0.0.0 20381` (`email_handler.py:L2386`). The handler runs as PID **1805** and binds `0.0.0.0:20381`. (The handler is launched from `/app` — see the re2 note in §4.1 — hence the `/app/...` paths in these lines.)

## 2.4 Job runner startup

**Startup log** (OBSERVED). Started as its own process with `python job_runner.py`. It imports the Flask app factory `from server import create_light_app` (`job_runner.py:L24`) and then enters its poll loop:

```
$ cat /tmp/jr.log
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/hjmqnpqbfolegrfxbfzd
Upload files to local dir
>>> init logging <<<
2026-07-08 05:16:31,045 - SL - DEBUG - 1806 - "/code/app/utils.py:17" - <module>() -  - load words file: /code/local_data/test_words.txt
```

**Up-signals** (OBSERVED): the same `>>> init logging <<<` boot banner, running as PID **1806**. When there is work to do, the loop prints `Take job %s` (`job_runner.py:L334`) — demonstrated with real cadence in §4.3. With an empty `Job` table the loop simply sleeps for 10 seconds between polls (`job_runner.py:L347`), so no per-poll line is printed; the running-quietly state is itself the healthy idle signal.

## 2.5 Dashboard UI — sign-in and alias-management signals

**Login page** (OBSERVED). The rendered `/auth/login` page contains the sign-in form. The template source is `templates/auth/login.html` (form `:L16`, `{{ form.csrf_token }}` `:L17`, password field `:L25`); the live-rendered line numbers differ because of the surrounding layout:

```
$ curl -sS http://localhost:7777/auth/login | grep -nE "<form method|csrf_token|name=\"email\"|type=\"password\"|/auth/register"
113:      <form method="post">
114:        <input id="csrf_token" name="csrf_token" type="hidden" value="IjIxMTRlZGNkOTYxN2M0MzNjNTVhYzcxNzkxOGZmNWM1MjhmZmVkNWQi.ak3dIA.JujbYcxfO4JK31RvZLFYgTr8E5w">
117:          <input autofocus="true" class="form-control" id="email" name="email" required type="email" value="">
124:          <input class="form-control" id="password" name="password" required type="password" value="">
142:    <a href="/auth/register">Sign up</a>
```

This confirms the sign-in surface renders: a POST form with a CSRF token, an email field, a password field, and a "Sign up" link to `/auth/register`.

**Authenticated dashboard (alias-management surface)** (OBSERVED). Logging in as the public fixture `john@wick.com` / `password` and fetching `/dashboard/` returns the alias-management page:

```
$ python dash.py     # logs in as john@wick.com, GETs /dashboard/
[login] POST /auth/login -> HTTP 302 Location=http://127.0.0.1:7777/dashboard/
[dashboard] GET /dashboard/ -> HTTP 200 (99694 bytes)

[rendered markers present in served /dashboard/ HTML]
  'title="Create a custom alias"'                    -> True
  'fa fa-random"></i> Random Alias'                  -> True
  'placeholder="Enter to search for alias"'          -> True

[stats <div class="h1 m-0"> numbers] (nb_alias is the alias-count stat) = ['10', '1', '0', '0']
[distinct alias addresses rendered in list] count=5
   - e0@sl.local
   - e1@sl.local
   - e2@sl.local
   - newsletter.foible812@sl.local
   - tonics_helots826@sl.local
```

The exact rendered lines carrying those markers (OBSERVED):

```
$ curl -sS --cookie <session> http://localhost:7777/dashboard/ | grep -nE "Create a custom alias|Random Alias|Enter to search|h1 m-0"
342:                      title="Create a custom alias"
354:                  <i class="fa fa-random"></i> Random Alias
429:            <div class="h1 m-0">10</div>
443:            <div class="h1 m-0">1</div>
457:            <div class="h1 m-0">0</div>
471:            <div class="h1 m-0">0</div>
521:                 placeholder="Enter to search for alias"
```

**Sign-in + management signals** (OBSERVED): login returns `HTTP 302` to `/dashboard/`; the dashboard returns `HTTP 200` (99694 bytes) and renders the "Create a custom alias" control (template `templates/dashboard/index.html:L44`, rendered L342), the "Random Alias" button (template `:L56`, rendered L354), the alias search box (template `:L215`, rendered L521), and the alias-count statistic `stats.nb_alias` = **10** (template `:L131`, rendered L429). The alias list itself is the `{% for alias_info in alias_infos %}` loop (`templates/dashboard/index.html:L233`). Seeing this page with the count and list is the UI confirmation that a user can sign in and manage aliases.

---

# 3. Q2 — User Actions & Correctness Signals

## 3.1 Inbound-mail data flow (for §3.4–§3.7)

```mermaid
flowchart LR
    Sender["External sender (swaks)"] -->|SMTP DATA| EH["email_handler.py<br/>handle_DATA :2289"]
    EH --> HF["handle_forward :536"]
    HF --> GC["get/create contact<br/>contact_utils.create_contact :110"]
    HF --> FWD["forward_email_to_mailbox :679"]
    FWD --> EL["EmailLog.create :740"]
    FWD --> MB["User mailbox (send path)"]
    HF -->|SMTP reply| RESP["E200 '250 Message accepted for delivery'<br/>app/email/status.py:2"]
    RESP --> Sender
```

All user actions below were driven against a **fresh temporary account** through the real blueprint routes and the real SMTP entry point (no debug hooks). Temp account: `blitzyqa_tmp_1783488005@outlook.com` (users.id **6**). It was fully removed in §6.

## 3.2 Account creation → activation → login

**Driver output** (OBSERVED). A `requests`-based script fetched CSRF tokens and POSTed to the real routes `POST /auth/register` (`app/auth/views/register.py:L32`), `GET /auth/activate` (`app/auth/views/activate.py:L17`), and `POST /auth/login` (`app/auth/views/login.py:L21-L25`):

```
$ /app/venv/bin/python /tmp/driver.py
======================================================================
[TEMP ACCOUNT] blitzyqa_tmp_1783488005@outlook.com
======================================================================
[1] POST /auth/register -> HTTP 200
[2] response page <title>: Activation Email Sent
      | SimpleLogin
[3] users.id=6  activated(before)=False  activation_code=CAPTURED(len=30) [REDACTED in doc]
[4] GET /auth/activate?code=<activation_code_redacted> -> HTTP 302 Location=http://127.0.0.1:7777/dashboard/
[5] post-activation dashboard flash(es): [('success', 'Your account has been activated'), ('success', 'Copied to clipboard')]
[6] activated(after)=True
[7] POST /auth/login -> HTTP 302 Location=http://127.0.0.1:7777/dashboard/
[8] GET /dashboard/ (authenticated) -> HTTP 200 (37296 bytes)
[9] POST /dashboard/ form-name=create-random-email -> HTTP 200
[10] alias-creation flash(es): [('success', 'Alias doling_clinks024@sl.local has been created'), ('success', 'Copied to clipboard')]
[11] alias count BEFORE=1  AFTER=2
[12] newest alias row (id,email) = (21, 'doling_clinks024@sl.local')
[RESULT] uid=6 alias_id=21 alias_email=doling_clinks024@sl.local
```

**Security redaction (F-critical):** the registration activation code is a 30-character one-time secret created by `ActivationCode.create(user_id=user.id, code=random_string(30))` and embedded in the activation link `f"{URL}/auth/activate?code={activation.code}"` (`app/auth/views/register.py:L120,L124`). It is **redacted** here (`<activation_code_redacted>`) rather than printed. The safe, sufficient proof that activation worked is the **state transition** `activated(before)=False` → `activated(after)=True` (lines [3] and [6]) plus the success flash `Your account has been activated` (line [5]) and the `302 → /dashboard/` redirect (line [4]).

**Correctness signals** (OBSERVED): registration returns `HTTP 200` with page title "Activation Email Sent"; the account exists as `users.id=6` initially unactivated; after visiting the activation link, `activated` flips to `True` and the app redirects to `/dashboard/`; login then returns `302 → /dashboard/` and the authenticated dashboard returns `HTTP 200`.

**Server-side log for the same run** (OBSERVED), with the activation code redacted where the debug `after_request` logger echoes the query string (`server.py:L284`):

```
$ tail -n +22 /tmp/web.log     # new lines produced by the driver run
2026-07-08 05:20:05,091 - SL - DEBUG - 1681 - "/code/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/register ImmutableMultiDict([]) 200, takes 0.005547523498535156
2026-07-08 05:20:05,136 - SL - DEBUG - 1681 - "/code/app/auth/views/register.py:85" - register() -  - create user blitzyqa_tmp_1783488005@outlook.com
2026-07-08 05:20:05,411 - SL - INFO - 1681 - "/code/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 05:20:05,414 - SL - DEBUG - 1681 - "/code/app/models.py:647" - create() -  - Disable onboarding emails
2026-07-08 05:20:05,437 - SL - DEBUG - 1681 - "/code/app/email_utils.py:303" - send_email() -  - send email to blitzyqa_tmp_1783488005@outlook.com, subject 'Just one more step to join SimpleLogin'
2026-07-08 05:20:05,438 - SL - DEBUG - 1681 - "/code/app/mail_sender.py:131" - send() -  - send email with subject 'Just one more step to join SimpleLogin', from '"noreply@sl.local" <noreply@sl.local>' to 'blitzyqa_tmp_1783488005@outlook.com'
2026-07-08 05:20:05,442 - SL - DEBUG - 1681 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/register ImmutableMultiDict([]) 200, takes 0.3476221561431885
2026-07-08 05:20:05,513 - SL - DEBUG - 1681 - "/code/app/email_utils.py:303" - send_email() -  - send email to simplelogin-newsletter.mishit719@sl.local, subject 'Welcome to SimpleLogin'
2026-07-08 05:20:05,514 - SL - DEBUG - 1681 - "/code/app/mail_sender.py:131" - send() -  - send email with subject 'Welcome to SimpleLogin', from '"noreply@sl.local" <noreply@sl.local>' to 'simplelogin-newsletter.mishit719@sl.local'
2026-07-08 05:20:05,514 - SL - DEBUG - 1681 - "/code/app/auth/views/activate.py:66" - activate() -  - redirect user to dashboard
2026-07-08 05:20:05,514 - SL - DEBUG - 1681 - "/code/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/activate ImmutableMultiDict([('code', '<activation_code_redacted>')]) 302, takes 0.03565716743469238
2026-07-08 05:20:05,523 - SL - DEBUG - 1681 - "/code/app/dashboard/views/index.py:172" - index() -  - Show intro to <User 6 blitzyqa_tmp_1783488005@outlook.com blitzyqa_tmp_1783488005@outlook.com>
2026-07-08 05:20:05,627 - SL - DEBUG - 1681 - "/code/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard/ ImmutableMultiDict([]) 200, takes 0.10880613327026367
2026-07-08 05:20:05,690 - SL - DEBUG - 1680 - "/code/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.03223371505737305
2026-07-08 05:20:05,939 - SL - DEBUG - 1680 - "/code/app/auth/views/login_utils.py:35" - after_login() -  - log user <User 6 blitzyqa_tmp_1783488005@outlook.com blitzyqa_tmp_1783488005@outlook.com> in
2026-07-08 05:20:05,939 - SL - DEBUG - 1680 - "/code/app/auth/views/login_utils.py:44" - after_login() -  - redirect user to dashboard
2026-07-08 05:20:05,939 - SL - DEBUG - 1680 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 302, takes 0.24609613418579102
2026-07-08 05:20:06,061 - SL - DEBUG - 1680 - "/code/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard/ ImmutableMultiDict([]) 200, takes 0.1166677474975586
2026-07-08 05:20:06,137 - SL - DEBUG - 1680 - "/code/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard/ ImmutableMultiDict([]) 200, takes 0.049488067626953125
2026-07-08 05:20:06,157 - SL - DEBUG - 1680 - "/code/app/models.py:1459" - generate_random_alias_email() -  - generate email doling_clinks024@sl.local
2026-07-08 05:20:06,172 - SL - INFO - 1680 - "/code/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 05:20:06,176 - SL - DEBUG - 1680 - "/code/app/dashboard/views/index.py:110" - index() -  - create new random alias <Alias 21 doling_clinks024@sl.local> for user <User 6 blitzyqa_tmp_1783488005@outlook.com blitzyqa_tmp_1783488005@outlook.com>
2026-07-08 05:20:06,179 - SL - DEBUG - 1680 - "/code/server.py:284" - after_request() -  - 127.0.0.1 POST /dashboard/ ImmutableMultiDict([]) 302, takes 0.038811683654785156
2026-07-08 05:20:06,234 - SL - DEBUG - 1680 - "/code/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard/ ImmutableMultiDict([('highlight_alias_id', '21'), ('query', ''), ('sort', ''), ('filter', '')]) 200, takes 0.05081629753112793
```

The `create user blitzyqa_tmp_1783488005@outlook.com` line (`app/auth/views/register.py:L85`) and `redirect user to dashboard` (`app/auth/views/activate.py:L66`) corroborate the driver's account-creation and activation results. The `GET /auth/activate` line shows the code position **redacted**.

## 3.3 Alias creation

**Reasoning:** the dashboard "Random Alias" action POSTs `form-name=create-random-email` to `/dashboard/`. The random branch checks the quota with `can_create_new_alias()` (`app/dashboard/views/index.py:L98`), creates the alias via `Alias.create_new_random(...)` (`:L104`), and logs it (`:L110`).

**Same-run consistency — flash, log line, and DB row all name the same alias** (OBSERVED):

- Flash (driver line [10]): `Alias doling_clinks024@sl.local has been created`.
- Log (`app/dashboard/views/index.py:L110`): `create new random alias <Alias 21 doling_clinks024@sl.local> for user <User 6 blitzyqa_tmp_1783488005@outlook.com ...>` (preceded by `generate_random_alias_email() ... generate email doling_clinks024@sl.local`, `app/models.py:L1459`).
- DB row (driver line [12]): newest alias `(21, 'doling_clinks024@sl.local')`.

All three reference the same address `doling_clinks024@sl.local` and the same id **21** owned by user **6**.

**Before/after alias count** (OBSERVED, driver line [11]): the account's alias count went from **1 → 2** (the initial onboarding alias, then the newly created random alias). This is the persistence-layer confirmation that the create action took effect.

## 3.4 Alias receives an email (default configuration → E200)

**Reasoning:** an external sender delivers over SMTP to the handler's real entry point `handle_DATA` (`email_handler.py:L2289`), which routes to `handle_forward` (`:L536`); a `Contact` is created (`app/contact_utils.py:L110`), the message is forwarded via `forward_email_to_mailbox` (`:L679`) which creates an `EmailLog` (`:L740`), and the protocol reply is `E200 = "250 Message accepted for delivery"` (`app/email/status.py:L2`).

**SMTP transcript + handler forward log** (OBSERVED). Sent with `swaks` directly to the handler on port 20381:

```
$ swaks --server 127.0.0.1:20381 --from qa-sender-ext@example.com --to doling_clinks024@sl.local \
        --h-Subject "Blitzy QA inbound test" --body "Hello doling_clinks024, this is an inbound test via swaks."
=== Trying 127.0.0.1:20381...
=== Connected to 127.0.0.1.
<-  220 3cb75349d424 Python SMTP 1.4.2
 -> EHLO 3cb75349d424
<-  250-3cb75349d424
<-  250-SIZE 33554432
<-  250-8BITMIME
<-  250-SMTPUTF8
<-  250 HELP
 -> MAIL FROM:<qa-sender-ext@example.com>
<-  250 OK
 -> RCPT TO:<doling_clinks024@sl.local>
<-  250 OK
 -> DATA
<-  354 End data with <CR><LF>.<CR><LF>
 -> Date: Wed, 08 Jul 2026 05:21:27 +0000
 -> To: doling_clinks024@sl.local
 -> From: qa-sender-ext@example.com
 -> Subject: Blitzy QA inbound test
 -> Message-Id: <20260708052127.001945@3cb75349d424>
 -> X-Mailer: swaks v20201014.0 jetmore.org/john/code/swaks/
 -> 
 -> Hello doling_clinks024, this is an inbound test via swaks.
 -> 
 -> 
 -> .
<-  250 Message accepted for delivery
 -> QUIT
<-  221 Bye
=== Connection closed with remote host.
SWAKS_EXIT=0
```

The handler's log for this exact message id (OBSERVED):

```
2026-07-08 05:21:27,568 - SL - DEBUG - 1805 - "/app/app/log.py:24" - set_message_id() -  - set message_id 0583e8d0-ec71-4018-bc8a-de957906adc0
2026-07-08 05:21:27,568 - SL - DEBUG - 1805 - "/app/email_handler.py:2342" - _handle() - 0583e8d0-ec71-4018-bc8a-de957906adc0 - ====>=====>====>====>====>====>====>====>
2026-07-08 05:21:27,568 - SL - INFO - 1805 - "/app/email_handler.py:2343" - _handle() - 0583e8d0-ec71-4018-bc8a-de957906adc0 - New message, mail from qa-sender-ext@example.com, rctp tos ['doling_clinks024@sl.local'] 
2026-07-08 05:21:27,569 - SL - INFO - 1805 - "/app/email_handler.py:1956" - handle() - 0583e8d0-ec71-4018-bc8a-de957906adc0 - Set CONTENT_TRANSFER_ENCODING
2026-07-08 05:21:27,590 - SL - DEBUG - 1805 - "/app/email_handler.py:1963" - handle() - 0583e8d0-ec71-4018-bc8a-de957906adc0 - Cannot parse Postfix queue ID from None None
2026-07-08 05:21:27,740 - SL - DEBUG - 1805 - "/app/email_handler.py:1980" - handle() - 0583e8d0-ec71-4018-bc8a-de957906adc0 - ==>> Handle mail_from:qa-sender-ext@example.com, rcpt_tos:['doling_clinks024@sl.local'], header_from:qa-sender-ext@example.com, header_to:doling_clinks024@sl.local, cc:None, reply-to:None, message_id:<20260708052127.001945@3cb75349d424>, client_ip:None, headers:[('Date', 'Wed, 08 Jul 2026 05:21:27 +0000'), ('To', 'doling_clinks024@sl.local'), ('From', 'qa-sender-ext@example.com'), ('Subject', 'Blitzy QA inbound test'), ('Message-Id', '<20260708052127.001945@3cb75349d424>'), ('X-Mailer', 'swaks v20201014.0 jetmore.org/john/code/swaks/'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-08 05:21:27,745 - SL - DEBUG - 1805 - "/app/email_handler.py:2202" - handle() - 0583e8d0-ec71-4018-bc8a-de957906adc0 - Forward phase qa-sender-ext@example.com(qa-sender-ext@example.com) -> doling_clinks024@sl.local
2026-07-08 05:21:27,761 - SL - DEBUG - 1805 - "/app/email_handler.py:580" - handle_forward() - 0583e8d0-ec71-4018-bc8a-de957906adc0 - Create or get contact for from_header:qa-sender-ext@example.com
2026-07-08 05:21:27,787 - SL - DEBUG - 1805 - "/app/app/contact_utils.py:110" - create_contact() - 0583e8d0-ec71-4018-bc8a-de957906adc0 - Created contact <Contact 4 qa-sender-ext@example.com 21> for alias <Alias 21 doling_clinks024@sl.local> with email qa-sender-ext@example.com invalid_email=False
2026-07-08 05:21:27,788 - SL - INFO - 1805 - "/app/app/handler/dmarc.py:33" - apply_dmarc_policy_for_forward_phase() - 0583e8d0-ec71-4018-bc8a-de957906adc0 - DMARC check disabled
2026-07-08 05:21:27,796 - SL - DEBUG - 1805 - "/app/email_handler.py:688" - forward_email_to_mailbox() - 0583e8d0-ec71-4018-bc8a-de957906adc0 - Forward <Contact 4 qa-sender-ext@example.com 21> -> <Alias 21 doling_clinks024@sl.local> -> <Mailbox 8 blitzyqa_tmp_1783488005@outlook.com>
2026-07-08 05:21:27,800 - SL - DEBUG - 1805 - "/app/email_handler.py:740" - forward_email_to_mailbox() - 0583e8d0-ec71-4018-bc8a-de957906adc0 - Create <EmailLog 4> for <Contact 4 qa-sender-ext@example.com 21>, <User 6 blitzyqa_tmp_1783488005@outlook.com blitzyqa_tmp_1783488005@outlook.com>, <Mailbox 8 blitzyqa_tmp_1783488005@outlook.com>
2026-07-08 05:21:27,806 - SL - DEBUG - 1805 - "/app/email_handler.py:867" - forward_email_to_mailbox() - 0583e8d0-ec71-4018-bc8a-de957906adc0 - From header, new:"qa-sender-ext at example.com" <qa-sender-ext_at_example_com_fbtcv@sl.local>, old:qa-sender-ext@example.com
2026-07-08 05:21:27,806 - SL - DEBUG - 1805 - "/app/email_handler.py:316" - replace_header_when_forward() - 0583e8d0-ec71-4018-bc8a-de957906adc0 - Delete Cc header, old value None
2026-07-08 05:21:27,806 - SL - DEBUG - 1805 - "/app/email_handler.py:313" - replace_header_when_forward() - 0583e8d0-ec71-4018-bc8a-de957906adc0 - Replace To header, old: doling_clinks024@sl.local, new: doling_clinks024@sl.local
2026-07-08 05:21:27,806 - SL - INFO - 1805 - "/app/app/handler/unsubscribe_generator.py:36" - _generate_header_with_original_behaviour() - 0583e8d0-ec71-4018-bc8a-de957906adc0 - Email has no unsubscribe header
2026-07-08 05:21:27,806 - SL - DEBUG - 1805 - "/app/email_handler.py:893" - forward_email_to_mailbox() - 0583e8d0-ec71-4018-bc8a-de957906adc0 - Forward mail from qa-sender-ext@example.com to blitzyqa_tmp_1783488005@outlook.com, mail_options:[], rcpt_options:[] 
2026-07-08 05:21:27,806 - SL - DEBUG - 1805 - "/app/app/mail_sender.py:131" - send() - 0583e8d0-ec71-4018-bc8a-de957906adc0 - send email with subject 'Blitzy QA inbound test', from '"qa-sender-ext at example.com" <qa-sender-ext_at_example_com_fbtcv@sl.local>' to 'doling_clinks024@sl.local'
2026-07-08 05:21:27,807 - SL - INFO - 1805 - "/app/email_handler.py:2367" - _handle() - 0583e8d0-ec71-4018-bc8a-de957906adc0 - Finish mail_from qa-sender-ext@example.com, rcpt_tos ['doling_clinks024@sl.local'], takes 0.23906350135803223 seconds with return code '250 Message accepted for delivery'<<===
```

**Correctness signals across three layers** (OBSERVED):
1. **Protocol layer:** the sender received `250 Message accepted for delivery` (E200, `app/email/status.py:L2`); `SWAKS_EXIT=0`.
2. **Log layer:** the forward chain `New message` (`:L2343`) → `Forward phase` (`:L2202`) → `Create or get contact` (`:L580`) → `Created contact <Contact 4 ...>` (`app/contact_utils.py:L110`) → `Forward <Contact 4> -> <Alias 21 doling_clinks024@sl.local> -> <Mailbox 8 ...>` (`:L688`) → `Create <EmailLog 4>` (`:L740`) → `Finish ... return code '250 Message accepted for delivery'` (`:L2367`).
3. **Persistence layer:** new rows (below).

**Before/after database state** (OBSERVED). Immediately before the send, alias 21 had **0** `Contact` and **0** `EmailLog` rows, and the global totals were at the pristine `dummy-data` baseline of `contact=2, email_log=2` (independently corroborated by the restored baseline in §6). After the send:

```
$ psql -h sl-db -U myuser -d simplelogin -tA -c \
  "SELECT (SELECT count(*) FROM contact), (SELECT count(*) FROM contact WHERE alias_id=21), (SELECT count(*) FROM email_log), (SELECT count(*) FROM email_log WHERE user_id=6);"
3|1|3|1

$ psql -h sl-db -U myuser -d simplelogin -x -c "SELECT id,user_id,alias_id,website_email,reply_email FROM contact WHERE alias_id=21;"
-[ RECORD 1 ]-+--------------------------------------------
id            | 4
user_id       | 6
alias_id      | 21
website_email | qa-sender-ext@example.com
reply_email   | qa-sender-ext_at_example_com_fbtcv@sl.local

$ psql -h sl-db -U myuser -d simplelogin -x -c "SELECT id,user_id,contact_id,alias_id,mailbox_id,is_reply,blocked FROM email_log WHERE user_id=6;"
-[ RECORD 1 ]--
id         | 4
user_id    | 6
contact_id | 4
alias_id   | 21
mailbox_id | 8
is_reply   | f
blocked    | f
```

The totals moved `contact 2 → 3` and `email_log 2 → 3`; alias 21 now has exactly one `Contact` and user 6 one `EmailLog`. The new `Contact` (id 4, `app/models.py:L1863`) links `alias_id=21` with a reverse-alias `reply_email` `qa-sender-ext_at_example_com_fbtcv@sl.local`, and the new `EmailLog` (id 4, `app/models.py:L2060`) links `contact_id=4`, `alias_id=21`, `mailbox_id=8`, `is_reply=f`, `blocked=f` — matching exactly the `<Contact 4>` / `<EmailLog 4>` / `<Mailbox 8>` named in the log. This is end-to-end "handled correctly."

## 3.5 Physical delivery to a mailbox (NOT_SEND_EMAIL disabled, local sink)

**Reasoning:** in the default `.env`, `NOT_SEND_EMAIL` is present, so `MailSender.send()` short-circuits before the network send (`app/mail_sender.py:L130`) — §3.4 forwards logically but does not physically transmit. To demonstrate real physical delivery, an **isolated** handler instance was run with a temporary `CONFIG` file that removes `NOT_SEND_EMAIL` and points `POSTFIX_SERVER`/`POSTFIX_PORT` (`app/config.py:L136,L149`) at a local aiosmtpd "sink" (a MailHog-equivalent). The repo `.env` files were **not** modified.

**swaks → isolated handler (:20382) → sink (:1025)** (OBSERVED):

```
$ swaks --server 127.0.0.1:20382 --from sink-demo-sender@example.com --to doling_clinks024@sl.local \
        --h-Subject "Blitzy QA MailHog physical-delivery test" \
        --body "This message should be physically forwarded to the mailbox via the sink."
=== Trying 127.0.0.1:20382...
=== Connected to 127.0.0.1.
<-  220 3cb75349d424 Python SMTP 1.4.2
 -> EHLO 3cb75349d424
<-  250-3cb75349d424
<-  250-SIZE 33554432
<-  250-8BITMIME
<-  250-SMTPUTF8
<-  250 HELP
 -> MAIL FROM:<sink-demo-sender@example.com>
<-  250 OK
 -> RCPT TO:<doling_clinks024@sl.local>
<-  250 OK
 -> DATA
<-  354 End data with <CR><LF>.<CR><LF>
 -> Date: Wed, 08 Jul 2026 05:26:00 +0000
 -> To: doling_clinks024@sl.local
 -> From: sink-demo-sender@example.com
 -> Subject: Blitzy QA MailHog physical-delivery test
 -> Message-Id: <20260708052600.002083@3cb75349d424>
 -> X-Mailer: swaks v20201014.0 jetmore.org/john/code/swaks/
 -> 
 -> This message should be physically forwarded to the mailbox via the sink.
 -> 
 -> 
 -> .
<-  250 Message accepted for delivery
 -> QUIT
<-  221 Bye
=== Connection closed with remote host.
SWAKS_EXIT=0
```

Isolated handler log — this time the **real SMTP send path** runs (`_send_to_smtp`, `app/mail_sender.py:L144`) instead of the short-circuit (OBSERVED):

```
2026-07-08 05:26:00,799 - SL - DEBUG - 2049 - "/app/email_handler.py:2202" - handle() - 92b5e7cf-909d-4704-9a74-9bfdd87db37d - Forward phase sink-demo-sender@example.com(sink-demo-sender@example.com) -> doling_clinks024@sl.local
2026-07-08 05:26:00,849 - SL - DEBUG - 2049 - "/app/email_handler.py:688" - forward_email_to_mailbox() - 92b5e7cf-909d-4704-9a74-9bfdd87db37d - Forward <Contact 5 sink-demo-sender@example.com 21> -> <Alias 21 doling_clinks024@sl.local> -> <Mailbox 8 blitzyqa_tmp_1783488005@outlook.com>
2026-07-08 05:26:00,852 - SL - DEBUG - 2049 - "/app/email_handler.py:740" - forward_email_to_mailbox() - 92b5e7cf-909d-4704-9a74-9bfdd87db37d - Create <EmailLog 5> for <Contact 5 sink-demo-sender@example.com 21>, <User 6 blitzyqa_tmp_1783488005@outlook.com blitzyqa_tmp_1783488005@outlook.com>, <Mailbox 8 blitzyqa_tmp_1783488005@outlook.com>
2026-07-08 05:26:00,858 - SL - DEBUG - 2049 - "/app/email_handler.py:893" - forward_email_to_mailbox() - 92b5e7cf-909d-4704-9a74-9bfdd87db37d - Forward mail from sink-demo-sender@example.com to blitzyqa_tmp_1783488005@outlook.com, mail_options:[], rcpt_options:[] 
2026-07-08 05:26:00,859 - SL - DEBUG - 2049 - "/app/app/mail_sender.py:156" - _send_to_smtp() - 92b5e7cf-909d-4704-9a74-9bfdd87db37d - getting a smtp connection takes seconds 0.0011525154113769531
2026-07-08 05:26:00,859 - SL - DEBUG - 2049 - "/app/app/mail_sender.py:163" - _send_to_smtp() - 92b5e7cf-909d-4704-9a74-9bfdd87db37d - Sendmail mail_from:sl.lmycyibvfqqdemzxgq4dqns5.if7zjrvjfdcdw@sl.local, rcpt_to:blitzyqa_tmp_1783488005@outlook.com, header_from:"sink-demo-sender at example.com" <sink-demo-sender_at_example_com_guowaymt@sl.local>, header_to:doling_clinks024@sl.local, header_cc:None
2026-07-08 05:26:00,862 - SL - INFO - 2049 - "/app/email_handler.py:2367" - _handle() - 92b5e7cf-909d-4704-9a74-9bfdd87db37d - Finish mail_from sink-demo-sender@example.com, rcpt_tos ['doling_clinks024@sl.local'], takes 0.20368218421936035 seconds with return code '250 Message accepted for delivery'<<===
```

The message physically arrived at the sink (OBSERVED):

```
$ cat /tmp/sink.log
===== MESSAGE RECEIVED AT SINK (MailHog-equivalent) =====
envelope.mail_from: sl.lmycyibvfqqdemzxgq4dqns5.if7zjrvjfdcdw@sl.local
envelope.rcpt_tos: ['blitzyqa_tmp_1783488005@outlook.com']
----- raw RFC822 message -----
Date: Wed, 08 Jul 2026 05:26:00 +0000
Subject: Blitzy QA MailHog physical-delivery test
Message-Id: <20260708052600.002083@3cb75349d424>
Content-Transfer-Encoding: 7bit
X-SimpleLogin-Type: Forward
X-SimpleLogin-EmailLog-ID: 5
X-SimpleLogin-Envelope-From: sink-demo-sender@example.com
X-SimpleLogin-Original-From: sink-demo-sender@example.com
X-SimpleLogin-Envelope-To: doling_clinks024@sl.local
From: "sink-demo-sender at example.com"
 <sink-demo-sender_at_example_com_guowaymt@sl.local>
To: doling_clinks024@sl.local

This message should be physically forwarded to the mailbox via the sink.




===== END MESSAGE =====
```

**Physical-delivery confirmation** (OBSERVED): with `NOT_SEND_EMAIL` disabled, the handler took the real `_send_to_smtp` path (`getting a smtp connection` at `app/mail_sender.py:L156`; `Sendmail mail_from:...` at `:L163`) and the sink received the forwarded message with `envelope.rcpt_tos: ['blitzyqa_tmp_1783488005@outlook.com']` (the user's real mailbox) and SimpleLogin's forwarding headers `X-SimpleLogin-Type: Forward` and `X-SimpleLogin-EmailLog-ID: 5`. This is the mail landing in the destination mailbox, not merely a logical accept.

## 3.6 Edge — alias quota gate (free-plan limit)

**Reasoning:** the random-alias branch guards creation with `can_create_new_alias()` (`app/dashboard/views/index.py:L98`), which compares the user's alias count against `max_alias_for_free_account()` (`app/models.py:L858`, returning `MAX_NB_EMAIL_FREE_PLAN`, default **5**, `app/config.py:L124`). When the limit is reached, the view flashes the upgrade warning (`app/dashboard/views/index.py:L123`) and does not create an alias.

**Driving user 6 (already at 2 aliases) to the limit** (OBSERVED):

```
$ /app/venv/bin/python /tmp/quota.py
user id=6  free-plan alias limit (MAX_NB_EMAIL_FREE_PLAN)=5
[attempt 1] before=2 after=3 -> created :: flashes=[('success', 'Alias forget_mother612@sl.local has been created'), ('success', 'Copied to clipboard')]
[attempt 2] before=3 after=4 -> created :: flashes=[('success', 'Alias hobble_brooms658@sl.local has been created'), ('success', 'Copied to clipboard')]
[attempt 3] before=4 after=5 -> created :: flashes=[('success', 'Alias throat_ceases062@sl.local has been created'), ('success', 'Copied to clipboard')]
[attempt 4] before=5 after=5 -> BLOCKED :: flashes=[('warning', 'You need to upgrade your plan to create new alias.'), ('success', 'Copied to clipboard')]
[QUOTA GATE HIT] count stayed at 5 (== limit 5); HTTP 200; warning flash observed
```

**Edge behavior** (OBSERVED): attempts 1–3 succeed (count 2→3→4→5); attempt 4 is **blocked** — the count stays at 5 (the before/after count is unchanged: `before=5 after=5`) and the response carries `('warning', 'You need to upgrade your plan to create new alias.')`. The quota gate correctly prevents exceeding the free-plan limit.

## 3.7 Edge — email to a non-existent alias (E515 rejection)

**Reasoning:** when the recipient alias does not exist, `handle_forward` first tries on-the-fly auto-creation (`email_handler.py:L545`); for `sl.local` there is no custom domain (`app/alias_utils.py:L104`) and no directory separator (`app/alias_utils.py:L165`), so it cannot be created and the handler returns 550 (`email_handler.py:L551`) as `E515 = "550 SL E515 Email not exist"` (`app/email/status.py:L51`).

**E-code definitions and swaks rejection** (OBSERVED):

```
$ grep -nE "^E200|^E502|^E515|^E518" app/email/status.py
2:E200 = "250 Message accepted for delivery"
39:E502 = "550 SL E502 Email not exist"
51:E515 = "550 SL E515 Email not exist"
54:E518 = "550 SL E518 Disabled mailbox"

$ swaks --server 127.0.0.1:20381 --from qa-sender-ext@example.com --to no_such_alias_blitzy_zzz@sl.local \
        --h-Subject "to nonexistent alias" --body "should be rejected"
=== Trying 127.0.0.1:20381...
=== Connected to 127.0.0.1.
<-  220 3cb75349d424 Python SMTP 1.4.2
 -> EHLO 3cb75349d424
<-  250-3cb75349d424
<-  250-SIZE 33554432
<-  250-8BITMIME
<-  250-SMTPUTF8
<-  250 HELP
 -> MAIL FROM:<qa-sender-ext@example.com>
<-  250 OK
 -> RCPT TO:<no_such_alias_blitzy_zzz@sl.local>
<-  250 OK
 -> DATA
<-  354 End data with <CR><LF>.<CR><LF>
 -> Date: Wed, 08 Jul 2026 05:28:15 +0000
 -> To: no_such_alias_blitzy_zzz@sl.local
 -> From: qa-sender-ext@example.com
 -> Subject: to nonexistent alias
 -> Message-Id: <20260708052815.002136@3cb75349d424>
 -> X-Mailer: swaks v20201014.0 jetmore.org/john/code/swaks/
 -> 
 -> should be rejected
 -> 
 -> 
 -> .
<** 550 SL E515 Email not exist
 -> QUIT
<-  221 Bye
=== Connection closed with remote host.
SWAKS_EXIT=26
```

Handler reasoning for the same message (OBSERVED):

```
2026-07-08 05:28:15,249 - SL - DEBUG - 1805 - "/app/email_handler.py:2202" - handle() - c6620c78-963a-4bc1-9845-bce5fa33dfa7 - Forward phase qa-sender-ext@example.com(qa-sender-ext@example.com) -> no_such_alias_blitzy_zzz@sl.local
2026-07-08 05:28:15,255 - SL - DEBUG - 1805 - "/app/email_handler.py:545" - handle_forward() - c6620c78-963a-4bc1-9845-bce5fa33dfa7 - alias no_such_alias_blitzy_zzz@sl.local not exist. Try to see if it can be created on the fly
2026-07-08 05:28:15,261 - SL - INFO - 1805 - "/app/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - c6620c78-963a-4bc1-9845-bce5fa33dfa7 - Cannot auto-create custom domain alias for no_such_alias_blitzy_zzz@sl.local because there's no custom domain for sl.local
2026-07-08 05:28:15,261 - SL - INFO - 1805 - "/app/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - c6620c78-963a-4bc1-9845-bce5fa33dfa7 - Cannot auto-create no_such_alias_blitzy_zzz@sl.local since it has no directory separator
2026-07-08 05:28:15,261 - SL - DEBUG - 1805 - "/app/email_handler.py:551" - handle_forward() - c6620c78-963a-4bc1-9845-bce5fa33dfa7 - alias no_such_alias_blitzy_zzz@sl.local cannot be created on-the-fly, return 550
2026-07-08 05:28:15,262 - SL - INFO - 1805 - "/app/email_handler.py:2367" - _handle() - c6620c78-963a-4bc1-9845-bce5fa33dfa7 - Finish mail_from qa-sender-ext@example.com, rcpt_tos ['no_such_alias_blitzy_zzz@sl.local'], takes 0.01985764503479004 seconds with return code '550 SL E515 Email not exist'<<===
```

**Edge behavior** (OBSERVED): the sender received `550 SL E515 Email not exist` (`SWAKS_EXIT=26`, i.e. a permanent SMTP failure), and the handler log shows the exact decision path (`:L545` → `alias_utils:L104` → `alias_utils:L165` → `:L551`) ending in the E515 return. No `Contact`/`EmailLog` rows are created for a rejected recipient.

---

# 4. Q3 — Background Components: Auto-Start & Behavior

## 4.1 They are independent daemons, not side effects of the web server

**Reasoning (cause → effect):** the container's default `CMD` starts **only** the web server; nothing in it (or in the web app code) spawns the email handler or job runner. They are separate long-running processes that an operator starts independently, matching the project's published three-container self-host topology (`sl-app`, `sl-email`, `sl-job-runner`).

```
$ sed -n '44p;47p' Dockerfile
EXPOSE 7777
CMD ["gunicorn","wsgi:app","-b","0.0.0.0:7777","-w","2","--timeout","15"]
```

`Dockerfile:L44` exposes only 7777 (the web port); `Dockerfile:L47` runs only gunicorn. The email handler (port 20381) and job runner are started with their own commands (§1). (From setup notes, the email handler is launched from `/app` rather than `/code` to sidestep a dependency/runtime `re2` incompatibility in this image — INFERRED as the operator's rationale for the working-directory choice. To be precise, that working-directory change also changes *which* `app/spamassassin_utils.py` is imported: the two copies differ only at line 8 — `/code` has `import re2 as re` while the image's `/app` copy is pre-patched to `import re` (`app/spamassassin_utils.py:L8`, OBSERVED below). The branch's `re2` shim (google-re2) lacks `DOTALL` whereas the standard-library `re` provides it, so `re.compile(..., re.DOTALL)` (`app/spamassassin_utils.py:L13`) raises `AttributeError` when imported from `/code` but succeeds from `/app`. So the cwd choice is not purely cosmetic — it selects a different, `re`-compatible `spamassassin_utils.py` — but every other handler behavior shown above is unaffected, and `email_handler.py` itself is byte-identical between `/code` and `/app`.)

**Evidence for the `/app` vs `/code` distinction** (OBSERVED). The two `spamassassin_utils.py` copies differ only at their import line; the `re2` shim lacks `DOTALL` while stdlib `re` provides it; and `email_handler.py` is identical in both trees:

```
$ diff /code/app/spamassassin_utils.py /app/app/spamassassin_utils.py
8c8
< import re2 as re
---
> import re

$ /app/venv/bin/python -c 'import re2, re; print("re2 DOTALL:", hasattr(re2,"DOTALL"), "| stdlib re DOTALL:", hasattr(re,"DOTALL"))'
re2 DOTALL: False | stdlib re DOTALL: True

$ diff -q /code/email_handler.py /app/email_handler.py && echo IDENTICAL
IDENTICAL
```

This confirms the working-directory choice selects a different (`re`-compatible) `spamassassin_utils.py` — the reason the import succeeds from `/app` — while leaving the handler's own code unchanged.

## 4.2 Process trees — before vs. after starting the daemons

**BEFORE (web server only)** (OBSERVED). Immediately after starting just gunicorn, an explicit search for the two daemons finds nothing:

```
$ ps -eo pid,ppid,cmd | grep -E "gunicorn|email_handler|job_runner" | grep -v grep
   1679       1 /app/venv/bin/python /app/venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
   1680    1679 /app/venv/bin/python /app/venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
   1681    1679 /app/venv/bin/python /app/venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
--- explicit search for the two daemons (expect no matches) ---
(no email_handler.py or job_runner.py process)
```

Only gunicorn exists: master **1679** with workers **1680/1681** (children of 1679). No `email_handler.py` or `job_runner.py` is running — proving they do **not** auto-start with the web server.

**AFTER (all three started)** (OBSERVED):

```
$ ps -eo pid,ppid,cmd | grep -E "gunicorn|email_handler|job_runner" | grep -v grep
   1679       1 /app/venv/bin/python /app/venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
   1680    1679 /app/venv/bin/python /app/venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
   1681    1679 /app/venv/bin/python /app/venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
   1805       1 /app/venv/bin/python email_handler.py
   1806       1 /app/venv/bin/python job_runner.py
```

The email handler (**1805**) and job runner (**1806**) are now **separate top-level processes** (PPID **1**), not children of gunicorn (1679). Each is an independent daemon.

**The web server has no mechanism to spawn them** (OBSERVED):

```
$ grep -nE "email_handler|job_runner|subprocess|os\.system|Popen" server.py wsgi.py
(no matches — the web server never references or spawns the other daemons)
```

`server.py` and `wsgi.py` contain no reference to the daemons and no process-spawning calls, confirming independence at the code level.

## 4.3 Job runner behavior & cadence (poll every ~10s; ready → taken → done)

**Reasoning:** the job runner's `__main__` is an infinite loop (`job_runner.py:L330`) that, inside a fresh app context (`:L332`), fetches due jobs (`get_jobs_to_run`, `:L307`), logs `Take job %s` (`:L334`), marks each `taken` and increments attempts (`:L337-L341`), runs `process_job` (`:L342`), marks it `done` (`:L344-L345`), and then sleeps 10 seconds (`:L347`). `JobState` values are `ready=0, taken=1, done=2, error=3` (`app/models.py:L253`).

**Methodology** (OBSERVED): to drive the loop, temporary no-op `Job` rows were enqueued through the **real model path** `Job.create(name=<temp-name>, commit=True)` (`app/models.py:L116`) and every pickup was read back from the job runner's own log line `Take job %s` (`job_runner.py:L334`). Per the magnitude/timing rule, the cadence was measured across **two separate `python job_runner.py` lifecycles**: **Run 1** — the original investigation session (job ids 25–27, name `blitzy-qa-noop`, helper `/tmp/jobcadence.py`) — and **Run 2**, an independent confirmation lifecycle started later in a fresh process (job ids 48–51, name `blitzy-qa-cp1fix-noop`, helper `/tmp/qa_cadence.py`, job-runner PID 4401). Both runs' raw pickup timestamps and each run's total span/duration are reported so the ~10s interval can be confirmed **stable across the two runs**.

**Run 1 (original session) — enqueue/pickup timing** (OBSERVED). Three no-op jobs, timed pickups over two poll cycles:

```
$ /app/venv/bin/python /tmp/jobcadence.py
[insert 1] job id=25 state=ready(0) at t=05:29:56
[pickup 1] job id=25 left ready after 6.66s -> state now 2 (2=done)
[insert 2] job id=26 state=ready(0) at t=05:30:03
[pickup 2] job id=26 left ready after 9.83s -> state now 2 (2=done)
[insert 3] job id=27 state=ready(0) at t=05:30:13
[pickup 3] job id=27 left ready after 10.01s -> state now 2 (2=done)

=== poll-interval gaps between consecutive pickups ===
gap pickup1->pickup2: 9.86s
gap pickup2->pickup3: 10.03s

inserted job ids: [25, 26, 27]
```

**Run 1 — raw job-runner log for those pickups** (OBSERVED). Each `Take job` is ~10s after the previous, and each unrecognized name is reported by `process_job` (`job_runner.py:L304`):

```
2026-07-08 05:30:03,072 - SL - DEBUG - 1806 - "/code/job_runner.py:334" - <module>() -  - Take job <Job 25 blitzy-qa-noop None>
2026-07-08 05:30:03,076 - SL - ERROR - 1806 - "/code/job_runner.py:304" - process_job() -  - Unknown job name blitzy-qa-noop
2026-07-08 05:30:13,091 - SL - DEBUG - 1806 - "/code/job_runner.py:334" - <module>() -  - Take job <Job 26 blitzy-qa-noop None>
2026-07-08 05:30:13,094 - SL - ERROR - 1806 - "/code/job_runner.py:304" - process_job() -  - Unknown job name blitzy-qa-noop
2026-07-08 05:30:23,108 - SL - DEBUG - 1806 - "/code/job_runner.py:334" - <module>() -  - Take job <Job 27 blitzy-qa-noop None>
2026-07-08 05:30:23,111 - SL - ERROR - 1806 - "/code/job_runner.py:304" - process_job() -  - Unknown job name blitzy-qa-noop
```

**Run 1 — persisted state transitions** (OBSERVED):

```
$ psql -h sl-db -U myuser -d simplelogin -c \
  "SELECT id,name,state,taken,attempts, to_char(created_at,'HH24:MI:SS') created, to_char(taken_at,'HH24:MI:SS') taken_at FROM job WHERE id IN (25,26,27) ORDER BY id;"
 id |      name      | state | taken | attempts | created  | taken_at 
----+----------------+-------+-------+----------+----------+----------
 25 | blitzy-qa-noop |     2 | t     |        1 | 05:29:56 | 05:30:03
 26 | blitzy-qa-noop |     2 | t     |        1 | 05:30:03 | 05:30:13
 27 | blitzy-qa-noop |     2 | t     |        1 | 05:30:13 | 05:30:23
```

Run 1 spanned ~27s end to end (first insert `05:29:56` → last pickup `05:30:23`); its first pickup was a partial **6.66s** because that job was enqueued mid-poll-cycle, after which the steady-state gaps settled to ~10s (`9.86s`, `10.03s`).

**Run 2 — raw job-runner log** (OBSERVED), a **separate** `python job_runner.py > /tmp/qa_jr_run2.log 2>&1 &` lifecycle started after Run 1 was stopped:

```
$ grep -E 'Take job|Unknown job name' /tmp/qa_jr_run2.log
2026-07-08 07:18:32,585 - SL - DEBUG - 4401 - "/code/job_runner.py:334" - <module>() -  - Take job <Job 48 blitzy-qa-cp1fix-noop None>
2026-07-08 07:18:32,589 - SL - ERROR - 4401 - "/code/job_runner.py:304" - process_job() -  - Unknown job name blitzy-qa-cp1fix-noop
2026-07-08 07:18:42,603 - SL - DEBUG - 4401 - "/code/job_runner.py:334" - <module>() -  - Take job <Job 49 blitzy-qa-cp1fix-noop None>
2026-07-08 07:18:42,606 - SL - ERROR - 4401 - "/code/job_runner.py:304" - process_job() -  - Unknown job name blitzy-qa-cp1fix-noop
2026-07-08 07:18:52,621 - SL - DEBUG - 4401 - "/code/job_runner.py:334" - <module>() -  - Take job <Job 50 blitzy-qa-cp1fix-noop None>
2026-07-08 07:18:52,624 - SL - ERROR - 4401 - "/code/job_runner.py:304" - process_job() -  - Unknown job name blitzy-qa-cp1fix-noop
2026-07-08 07:19:02,638 - SL - DEBUG - 4401 - "/code/job_runner.py:334" - <module>() -  - Take job <Job 51 blitzy-qa-cp1fix-noop None>
2026-07-08 07:19:02,641 - SL - ERROR - 4401 - "/code/job_runner.py:304" - process_job() -  - Unknown job name blitzy-qa-cp1fix-noop
```

**Run 2 — enqueue/pickup helper (DB-observed gaps + total duration)** (OBSERVED):

```
$ /app/venv/bin/python /tmp/qa_cadence.py 2>&1 | sed -n '/\[insert 1\]/,$p'
[insert 1] job id=48 state=ready(0) at t=07:18:25.536
[pickup 1] job id=48 picked up at t=07:18:32.611 -> state now 2 (2=done)
[insert 2] job id=49 state=ready(0) at t=07:18:32.614
[pickup 2] job id=49 picked up at t=07:18:42.618 -> state now 2 (2=done)
[insert 3] job id=50 state=ready(0) at t=07:18:42.629
[pickup 3] job id=50 picked up at t=07:18:52.633 -> state now 2 (2=done)
[insert 4] job id=51 state=ready(0) at t=07:18:52.636
[pickup 4] job id=51 picked up at t=07:19:02.671 -> state now 2 (2=done)

=== poll-interval gaps between consecutive pickups (DB-observed) ===
gap pickup1->pickup2: 10.01s
gap pickup2->pickup3: 10.02s
gap pickup3->pickup4: 10.04s

inserted job ids: [48, 49, 50, 51]
total run duration (first insert -> last pickup): 37.13s
```

**Behavior confirmation across two runs** (OBSERVED): the ~10s cadence is **stable across both independent `job_runner.py` lifecycles**, run in separate sessions with different job-runner PIDs (Run 1 PID 1806, Run 2 PID 4401). Run 1's steady-state gaps were **9.86s and 10.03s** (run span ~27s; its first pickup was a partial 6.66s because that job was enqueued mid-cycle); Run 2's three gaps were **10.01s, 10.02s, 10.04s** (total run duration **37.13s**). Every steady-state gap matches the `time.sleep(10)` cadence (`job_runner.py:L347`), and each run's `Take job` timestamps (Run 1 `05:30:03/13/23`; Run 2 `07:18:32/42/52` then `07:19:02`) are exactly 10s apart. Each job progressed `ready(0) → taken(1) → done(2)` with `taken=t` and `attempts=1` — the transient `taken` state is written at `:L337-L341` and the terminal `done` at `:L344-L345`. The runner picks up work continuously without external prompting; that steady ~10s pickup, reproduced in two separate runs, is the "functioning as intended" signal. (Both runs used intentionally unregistered job names — `blitzy-qa-noop` (Run 1) and `blitzy-qa-cp1fix-noop` (Run 2) — so `process_job` logs `Unknown job name` at `:L304` while still exercising the full pickup/commit lifecycle. Run 1's temp jobs 25–27 are removed in §6; Run 2's temp jobs 48–51 were deleted immediately after measurement, both leaving the `job` table at its baseline of only the pre-existing id=1 `blitzy-smoke-noop`.)

## 4.4 The cron scheduler is a distinct mechanism from the job runner

**Reasoning:** the job runner (§4.3) polls the `Job` table every 10 seconds for on-demand work. Scheduled/periodic tasks are a **separate** subsystem: `cron.py` is invoked on cron schedules by **yacron** (0.11.2) per `crontab.yml`. They are different files, different triggers, and different cadences.

```
$ grep -cE 'cron\.py -j' crontab.yml            # total scheduled cron jobs
15

$ awk '/-j /{split($0,a," -j "); name=a[2]} /schedule:/{s=$0; sub(/^[[:space:]]*schedule:[[:space:]]*"/,"",s); sub(/".*/,"",s); printf "%-32s %s\n", name, s}' crontab.yml
stats                            0 0 * * *
delete_old_monitoring            15 1 * * *
check_custom_domain              15 2 * * *
check_hibp                       15 3 * * *
notify_hibp                      15 4 * * *
delete_logs                      15 5 * * *
delete_old_data                  30 5 * * *
poll_apple_subscription          15 6 * * *
notify_trial_end                 15 8 * * *
notify_manual_subscription_end   15 9 * * *
notify_premium_end               15 10 * * *
delete_scheduled_users           15 11 * * *
send_undelivered_mails           */5 * * * *
clear_alias_audit_log            0 * * * *
clear_user_audit_log             0 * * * *

$ grep -E 'schedule:' crontab.yml | grep -vcE '"[0-9]+ [0-9]+ \* \* \*"'   # schedules that are NOT fixed-daily
3
```

**Distinction** (OBSERVED from `crontab.yml`): the yacron schedule defines **fifteen** scheduled jobs (the count command above returns `15`), each running `python /code/cron.py -j <job>`. **Twelve run at fixed daily times** — `stats` (`crontab.yml:L3`, `0 0 * * *`), `delete_old_monitoring` (`:L9`, `15 1 * * *`), `check_custom_domain` (`:L15`, `15 2 * * *`), `check_hibp` (`:L21`, `15 3 * * *`), `notify_hibp` (`:L28`, `15 4 * * *`), `delete_logs` (`:L35`, `15 5 * * *`), `delete_old_data` (`:L41`, `30 5 * * *`), `poll_apple_subscription` (`:L47`, `15 6 * * *`), `notify_trial_end` (`:L53`, `15 8 * * *`), `notify_manual_subscription_end` (`:L59`, `15 9 * * *`), `notify_premium_end` (`:L65`, `15 10 * * *`), and `delete_scheduled_users` (`:L71`, `15 11 * * *`) — and the remaining **three are not daily**: `send_undelivered_mails` runs every 5 minutes (`:L78`, `*/5 * * * *`), while `clear_alias_audit_log` (`:L85`, `0 * * * *`) and `clear_user_audit_log` (`:L92`, `0 * * * *`) run hourly. Regardless of the exact count, the point stands: this is the scheduled-maintenance mechanism (`cron.py` + yacron), separate from the `Job`-table-polling `job_runner.py`.

---

# 5. Coverage Pass

Every named item across Q1–Q3, with the evidence location and label.

**Q1 — Startup & Health**

| Item | Signal / Evidence | Where | Label |
|---|---|---|---|
| Web server up | gunicorn banner, `Listening at: http://0.0.0.0:7777`, workers 1680/1681 | §2.1 | OBSERVED |
| Web `/health` | `HTTP/1.1 200 OK`, body `success` via `healthcheck()` (`server.py:L214`) | §2.1 | OBSERVED |
| Web index redirect | `302 → /auth/login` (`server.py:L250-L251`) | §2.1 | OBSERVED |
| Web dev mode | Werkzeug `HTTP/1.0`, `Server: Werkzeug/1.0.1` (`server.py:L588`) | §2.2 | OBSERVED |
| Email handler up | `Listen for port 20381` (`:L2403`), `Start mail controller 0.0.0.0 20381` (`:L2386`) | §2.3 | OBSERVED |
| Job runner up | `>>> init logging <<<` boot banner, PID 1806 | §2.4 | OBSERVED |
| Login page UI | form + csrf + email + password + Sign up (`templates/auth/login.html:L16,L17,L25`) | §2.5 | OBSERVED |
| Dashboard UI | `Create a custom alias` (L44), `Random Alias` (L56), search (L215), alias list (L233), `stats.nb_alias`=10 (L131) | §2.5 | OBSERVED |

**Q2 — User Actions & Correctness**

| Item | Signal / Evidence | Where | Label |
|---|---|---|---|
| Account creation | `POST /auth/register` 200; `create user ...` (`app/auth/views/register.py:L85`); `users.id=6` | §3.2 | OBSERVED |
| Account activation | `activated` False→True; success flash; `302 → /dashboard/` (`app/auth/views/activate.py:L66`); code redacted | §3.2 | OBSERVED |
| Login | `POST /auth/login` 302 → `/dashboard/`; `log user ... in` (`app/auth/views/login.py:L21-L25`) | §3.2 | OBSERVED |
| Alias creation | flash + log (`app/dashboard/views/index.py:L110`) + row all name `doling_clinks024@sl.local` id 21; count 1→2 | §3.3 | OBSERVED |
| Alias receives email | swaks E200 (`app/email/status.py:L2`); forward log (`email_handler.py:L688`, `:L740`); Contact 4 / EmailLog 4 rows | §3.4 | OBSERVED |
| Physical delivery | real `_send_to_smtp` (`app/mail_sender.py:L156,L163`); sink receives to real mailbox | §3.5 | OBSERVED |
| Edge: quota gate | limit 5; 4th attempt blocked, count unchanged, upgrade warning (`app/dashboard/views/index.py:L123`) | §3.6 | OBSERVED |
| Edge: non-existent alias | `550 SL E515` (`app/email/status.py:L51`); decision path `email_handler.py:L545`→`app/alias_utils.py`→`:L551` | §3.7 | OBSERVED |

**Q3 — Background Components**

| Item | Signal / Evidence | Where | Label |
|---|---|---|---|
| Default CMD = web only | `Dockerfile:L44` (EXPOSE 7777), `:L47` (gunicorn only) | §4.1 | OBSERVED |
| Do NOT auto-start | web-only process tree; no daemon match | §4.2 | OBSERVED |
| Independent daemons | email handler 1805 & job runner 1806 as PPID-1 processes | §4.2 | OBSERVED |
| No spawn mechanism | grep of `server.py`/`wsgi.py` → no matches | §4.2 | OBSERVED |
| Job runner cadence | ~10s poll stable across **2 runs** (Run 1 gaps 9.86/10.03s, span ~27s; Run 2 gaps 10.01/10.02/10.04s, dur 37.13s); `Take job` 10s apart (`:L334`, `:L347`) | §4.3 | OBSERVED |
| Job state machine | ready(0)→taken(1)→done(2) persisted (`app/models.py:L253`) | §4.3 | OBSERVED |
| Cron ≠ job runner | 15 yacron `cron.py -j` jobs (12 daily; 3 non-daily: `send_undelivered_mails` every 5 min, `clear_alias_audit_log`/`clear_user_audit_log` hourly) | §4.4 | OBSERVED |
| Email-handler `/app` cwd rationale | `/app` loads pre-patched `spamassassin_utils.py` (`import re` vs `/code` `import re2 as re`, `:L8`); `re2` lacks `DOTALL` → import succeeds from `/app`; `email_handler.py` identical | §4.1 | OBSERVED (diff) + INFERRED (operator rationale) |

---

# 6. Cleanup — Repository & Data Left Unchanged

Per the read-only constraint, all temporary data and observation scripts were removed. This section shows the exact deletion commands and their output.

**Temp-data cleanup** (OBSERVED). Most `users` foreign keys are `ON DELETE CASCADE` (verified: `contact`, `email_log`, `mailbox`, `alias` cascade), so deleting the temp user removes its owned aliases/contacts/email-logs/mailboxes; the temp `Job` rows are deleted explicitly:

```
$ PSQL='psql -h sl-db -U myuser -d simplelogin'
$ $PSQL -c "SELECT (SELECT count(*) FROM users) users, (SELECT count(*) FROM alias) alias, (SELECT count(*) FROM contact) contact, (SELECT count(*) FROM email_log) email_log, (SELECT count(*) FROM job) job;"   # BEFORE / AFTER snapshot query
$ $PSQL -c "DELETE FROM job WHERE id IN (25,26,27);"
$ $PSQL -c "DELETE FROM users WHERE id=6;"   # ON DELETE CASCADE removes its aliases/contacts/email_logs/mailboxes
================= [E23] TEMP-DATA CLEANUP =================
----- BEFORE: temp data present -----
 users | alias | contact | email_log | job 
-------+-------+---------+-----------+-----
     3 |    16 |       4 |         4 |   4
(1 row)

temp user 6 + owned rows, and temp jobs 25/26/27:
              temp_user              | u6_alias | u6_contact | u6_emaillog | temp_jobs 
-------------------------------------+----------+------------+-------------+-----------
 blitzyqa_tmp_1783488005@outlook.com |        5 |          2 |           2 |         3
(1 row)


----- DELETE temp jobs 25,26,27 -----
DELETE 3
----- DELETE temp user 6 (ON DELETE CASCADE removes its aliases/contacts/email_logs/mailboxes/activation_code) -----
DELETE 1

----- AFTER: baseline restored -----
 users | alias | contact | email_log | job 
-------+-------+---------+-----------+-----
     2 |    11 |       2 |         2 |   1
(1 row)

temp user 6 gone? + remaining users + remaining job:
 id |          email          | activated 
----+-------------------------+-----------
  1 | john@wick.com           | t
  2 | winston@continental.com | t
(2 rows)

 id |       name        | state 
----+-------------------+-------
  1 | blitzy-smoke-noop |     2
(1 row)
```

**Result** (OBSERVED): counts returned exactly to the `dummy-data` baseline (`users 3→2`, `alias 16→11`, `contact 4→2`, `email_log 4→2`, `job 4→1`); the temp account `blitzyqa_tmp_1783488005@outlook.com` and temp jobs 25/26/27 are gone; only the public fixtures (`john@wick.com`, `winston@continental.com`) and the pre-existing `blitzy-smoke-noop` job remain.

**Temp observation scripts removed and repository verified clean** (OBSERVED):

```
$ docker exec sl-app bash -lc 'rm -fv /tmp/driver.py /tmp/quota.py /tmp/jobcadence.py /tmp/sink.py /tmp/dash.py /tmp/mailhog.env; \
                               echo "remaining *.py in /tmp: $(ls /tmp/*.py 2>/dev/null | wc -l)"'
remaining *.py in /tmp: 0

$ git status --porcelain
                       # (empty output — working tree is clean)

$ git diff --stat 2cd6ee777f8c2d3531559588bcfb18627ffb5d2c HEAD
 blitzy/documentation/app_2cd6ee777f8c.md | 961 +++++++++++++++++++++++++++++++
 1 file changed, 961 insertions(+)
```

All observation scripts were removed (0 remaining). `git status --porcelain` produces **no output** (clean tree): the runtime investigation touched **zero** repository files — temp data lived only in the throwaway Postgres database and temp scripts only under `/tmp`, both outside the repo. The single change introduced on this branch versus the source parent is this documentation file.

---

## Summary of Direct Answers

- **Q1:** The web server is up when gunicorn logs `Listening at: http://0.0.0.0:7777` and `GET /health` returns `HTTP/1.1 200 OK`/`success` (`server.py:L213-L215`); the email handler is up when it logs `Listen for port 20381` / `Start mail controller 0.0.0.0 20381` (`email_handler.py:L2403,L2386`); the job runner is up after its `>>> init logging <<<` boot banner and thereafter its ~10s `Take job` polling (`job_runner.py:L334,L347`). In the UI, the `/auth/login` form and the authenticated `/dashboard/` (Create-custom-alias, Random-Alias, search box, alias list, `stats.nb_alias`) confirm users can sign in and manage aliases.
- **Q2:** Creating an account (`app/auth/views/register.py:L32`), activating it (`activated` False→True), logging in, creating a random alias (`app/dashboard/views/index.py:L104`, log `:L110`), and sending mail to that alias are all handled correctly — shown simultaneously by the SMTP `250 Message accepted for delivery` (E200, `app/email/status.py:L2`), the forward log chain (`email_handler.py:L688,L740`), the new `Contact`/`EmailLog` rows (`app/models.py:L1863,L2060`), and — with sending enabled — physical delivery to the mailbox. Edge conditions behave correctly too: the free-plan quota gate blocks the 6th alias with an upgrade warning, and mail to a non-existent alias is rejected with `550 SL E515`.
- **Q3:** The email handler and job runner do **not** auto-start with the web server (`Dockerfile:L47` runs only gunicorn; `server.py`/`wsgi.py` contain no spawn logic). They are independent daemons (PPID 1) that, once started, run continuously — the job runner polling every ~10s and driving jobs `ready → taken → done`. The yacron-driven `cron.py` scheduler (`crontab.yml`) is a separate, schedule-based mechanism, not the job runner.
