# SimpleLogin — Local Bring-Up & Runtime Observation Report

**Branch / revision:** `app_2cd6ee777f8c` (HEAD `2cd6ee77`, commit `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c` — "chore: emit some missing contact audit logs (#2269)")
**Docker image:** `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (tag `simple-login__app__2cd6ee77…`), run as container `simplelogin-app`
**Configuration:** default/canonical — `.env` is a byte-identical copy of `example.env`
**Method:** every behavioral claim below was produced by **running the code through its real entry points** and capturing the **actual** runtime output. Each claim carries a `file:line` reference *and* the unedited command/log output that supports it. Statements that could only be read (not run) are explicitly labelled **(inferred)**.

This document answers three operator questions about bringing up and observing SimpleLogin locally:

- **Q1 — Startup & readiness signals:** what to notice in the logs or UI that shows the app is ready for user authentication and alias-based email activity.
- **Q2 — New-user walkthrough:** register → verify email → log in, and the visible behavior confirming each step and forwarding the user into the dashboard (plus edge/error cases).
- **Q3 — Behind-the-scenes background jobs & internal services:** what runs behind the flow, and the runtime evidence that background jobs / internal services supporting email forwarding and identity verification are active and communicating.

---

## Default-configuration values that shape every observation

The behavior reported here is coupled to these default values in `.env` (= `example.env`). They are stated up-front because they explain *why* the default local system behaves differently from production. None were modified.

| Key | Value | `file:line` | Effect on observed behavior |
|-----|-------|-------------|-----------------------------|
| `URL` | `http://localhost:7777` | `example.env:L6` | Base URL in activation links, logged at import as `>>> URL: http://localhost:7777`. |
| `NOT_SEND_EMAIL` | `true` | `example.env:L19` | Outbound mail is **not sent**; `mail_sender.send()` logs only subject/from/to metadata and returns `True` (see the important correction in Q2). |
| `EMAIL_DOMAIN` | `sl.local` | `example.env:L22` | Aliases live under `@sl.local`. |
| `# DISABLE_REGISTRATION` | commented out | `example.env:L58` | Registration is **enabled** by default. |
| `DISABLE_ONBOARDING` | `true` | `example.env:L150` | `User.create` logs `"Disable onboarding emails"` and returns **before** enqueuing onboarding jobs. |
| `MEM_STORE_URI` | unset | — | Flask uses cookie sessions; login works without Redis for the session store. |
| `EVENT_WEBHOOK` | unset | `app/config.py:L612` | The event dispatcher **short-circuits** in the normal flow (no `SyncEvent` persisted) — see Q3 for how the pipeline was still demonstrated end-to-end. |

**Log-line format (every process).** SimpleLogin uses a single `SL` logger [`app/log.py:L76`]. Console lines look like:

```
TIMESTAMP - SL - LEVEL - PID - "/app/<file>.py:LINE" - <func>() - <message_id> - <message>
```

---

## Direct answers (summary)

**Q1 — Readiness.** The operator should look, per process, for: the `>>> init logging <<<` banner printed at import [`app/log.py:L67`]; the webapp's Gunicorn lines `Starting gunicorn 20.0.4` → `Listening at: http://0.0.0.0:7777` → `Booting worker with pid: …` [`Dockerfile:L47`]; the email handler's `Listen for port 20381` + `Start mail controller 0.0.0.0 20381` [`email_handler.py:L2403,L2386`]; the event listener's `Using PostgresEventSource` / `Starting with HttpEventSink` / `Starting to listen to events` [`event_listener.py:L34,L43`, `events/event_source.py:L49`]; the job runner alive (init banner; its poll loop is silent while idle); a `200` + body `success` from `GET /health` [`server.py:L213-L215`]; the rendered login page at `GET /auth/login`; and per-request access-log lines from `after_request` [`server.py:L284`]. Together these confirm the app is ready for authentication and alias email activity.

**Q2 — New-user walkthrough.** Registering via `POST /auth/register` logs `create user <email>` [`register.py:L85`] and renders the "waiting for activation" page [`register.py:L104`]. Verifying via `GET /auth/activate?code=…` flips `users.activated` **False → True** [`activate.py:L49`], flashes **"Your account has been activated"** [`activate.py:L56`], logs the user in, and **302-redirects to `/dashboard/`** [`activate.py:L67`]. Logging in via `POST /auth/login` logs `log user … in` [`login_utils.py:L35`] and redirects to `/dashboard/` [`login_utils.py:L44`]. Every edge case (invalid/expired code, wrong password, not-activated login, resend, `DISABLE_REGISTRATION`, disabled account, scheduled-for-deletion) produces its own named message — all observed below.

**Q3 — Behind the scenes.** Yes — background jobs and internal services support the flow. `User.create` provisions a **verified default mailbox** and a **first alias** (`simplelogin-newsletter…`) and, in production, enqueues onboarding jobs [`app/models.py:L601-L668`]. A DB-backed `Job` queue is drained by `job_runner.py` on a **~10.02 s poll** (measured, stable across 2 runs) [`job_runner.py:L347`]. The `email_handler.py` `aiosmtpd` service on **:20381** forwards mail through `handle_forward` [`email_handler.py:L536`], ending with a `Finish mail_from …` completion line [`email_handler.py:L2367`]. The event pipeline persists a `SyncEvent` and issues a PostgreSQL `NOTIFY simplelogin_sync_events` [`app/events/event_dispatcher.py:L25-L26`] consumed by the running `event_listener.py` — demonstrated end-to-end below. New Relic telemetry (`app/events/auth_event.py`), a `yacron` scheduler (`crontab.yml`, observed firing every 5 min), and the Postfix-queue monitor (`monitoring.py`) round out the machinery.

---

## Exact build/run commands used

The provided image bundles PostgreSQL 15.13 and Redis 7.0.15 and runs the five SimpleLogin processes in-container. The canonical documented local run is:

```bash
# Datastores (per CONTRIBUTING.md:L100 for Postgres; Redis required on :6379)
#   docker run -e POSTGRES_PASSWORD=mypassword -e POSTGRES_USER=myuser \
#              -e POSTGRES_DB=simplelogin -p 5432:5432 postgres:13
# In this image they are started in-container:
service postgresql start
service redis-server start

# App config = default
cp example.env .env

# Schema + seed (CONTRIBUTING.md:L106)
export FLASK_APP=server.py
alembic upgrade head          # -> head 32f25cbf12f6
flask dummy-data              # creates john@wick.com/password (CONTRIBUTING.md:L109)

# The five runtime processes:
gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15   # webapp (Dockerfile:L47)   PID 525/526/527
python email_handler.py                               # aiosmtpd :20381           PID 593
python job_runner.py                                  # 10s poll loop             PID 594
python event_listener.py listener                     # Postgres LISTEN/NOTIFY    PID 637
yacron -c /app/crontab.yml                            # 15 scheduled jobs         PID 1238
```

Verification that the foundation is up:

```console
$ git rev-parse --abbrev-ref HEAD          # (host working tree)
blitzy-19312d7e-6cc6-4b7b-b329-47abb71fdf6c
$ git rev-parse --short HEAD
2cd6ee77
# container /app is a detached checkout at the same commit:
$ docker exec simplelogin-app git -C /app rev-parse HEAD
2cd6ee777f8c2d3531559588bcfb18627ffb5d2c

$ redis-cli ping
PONG
$ alembic current
32f25cbf12f6 (head)
# Seed users:
id | email                     | activated | is_admin
 1 | john@wick.com             | t         | t
 2 | winston@continental.com   | t         | f
```

> **Note on the webapp entry point.** Production uses Gunicorn (`wsgi:app`, `Dockerfile:L47`), which is what runs in this image and what the readiness evidence below is captured from. The repository also ships a Flask dev-server entry `local_main()` [`server.py:L572`] → `app.run(debug=True, port=7777)` [`server.py:L588`] invoked by `python3 server.py`; both bind port 7777 and emit the same `SL` log lines.

---

## Q1 — Startup & readiness signals (logs and UI)

**Direct answer.** After start-up, the readiness signals — in the order an operator sees them per process — are: (1) the `>>> init logging <<<` banner; (2) the webapp's Gunicorn `Listening at: http://0.0.0.0:7777` + `Booting worker with pid` lines; (3) the email handler's `Listen for port 20381` + `Start mail controller 0.0.0.0 20381`; (4) the event listener's `Using PostgresEventSource` / `Starting with HttpEventSink` / `Starting to listen to events`; (5) the job runner's init banner (its poll loop is intentionally silent while the queue is empty); (6) `GET /health` → `200` / body `success`; (7) the rendered login page at `GET /auth/login`; and (8) per-request access-log lines. All items below are **observed** (none inferred).

### 1. `>>> init logging <<<` banner — every process [`app/log.py:L67`]

`app/log.py:L67` is a literal `print(">>> init logging <<<")` executed at import; the `SL` logger and the `LOG.d/i/w/e` shortcuts are defined just below it [`app/log.py:L76`]. The banner (preceded by a few import-time `print`s) appears in each process's log. From the webapp log (one block per Gunicorn worker):

```
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/puzgyljzwhbicgjxrrxl
Upload files to local dir
>>> init logging <<<
2026-07-08 03:56:08,654 - SL - DEBUG - 526 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

### 2. Webapp readiness — Gunicorn lines [`Dockerfile:L47`, `EXPOSE 7777` at `Dockerfile:L44`]

The production launch command is `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` [`Dockerfile:L47`]; `wsgi.py` exposes `app = create_app()` [`wsgi.py:L1,L3`]. The canonical readiness lines captured from `webapp.log`:

```
[2026-07-08 03:56:08 +0000] [525] [INFO] Starting gunicorn 20.0.4
[2026-07-08 03:56:08 +0000] [525] [INFO] Listening at: http://0.0.0.0:7777 (525)
[2026-07-08 03:56:08 +0000] [525] [INFO] Using worker: sync
[2026-07-08 03:56:08 +0000] [526] [INFO] Booting worker with pid: 526
[2026-07-08 03:56:08 +0000] [527] [INFO] Booting worker with pid: 527
```

`Listening at: http://0.0.0.0:7777` and the two `Booting worker with pid` lines (one per `-w 2` worker) are the definitive "webapp is ready" signal.

### 3. Blueprints registered [`server.py:L233-L246`]

`register_blueprints(app)` mounts the HTTP surface. Confirmed at runtime by constructing the app and inspecting the URL map — the registered blueprints include `auth`, `monitor`, `dashboard`, `developer`, `phone`, `oauth` (mounted at both `/oauth` and `/oauth2`), `onboarding`, `discover`, `internal`, and `api`. Representative route resolution (observed):

```
/auth/login     -> True
/auth/register  -> True
/auth/activate  -> True
/dashboard/     -> True
/health         -> True
```

The `auth` blueprint is mounted under `/auth` [`app/auth/base.py`] and the dashboard under `/dashboard` [`app/dashboard/base.py:L3-L8`].

### 4. Boot-time SL-domain seeding + PGP key load [`init_app.py`]

`init_app.py` seeds the SimpleLogin domains and loads PGP public keys at boot. On the **first** boot (captured during `flask dummy-data`), the domain seed line appears [`init_app.py:L44`]:

```
Add sl.local to SL domain
```

Running `init_app.py` again (idempotent) shows the PGP load path [`init_app.py:L13,L16,L36`] and the idempotent domain check [`init_app.py:L42`]:

```
... - SL - DEBUG - ... "/app/init_app.py:16" - load_pgp_public_keys() - Load PGP key for mailbox <Mailbox 2 pgp@example.org>
... - SL - DEBUG - ... "/app/init_app.py:36" - load_pgp_public_keys() - Finish load_pgp_public_keys
... - SL - INFO  - ... "/app/init_app.py:42" - add_sl_domains()      - sl.local is already a SL domain
```

### 5. Email handler readiness — `:20381` [`email_handler.py:L2403,L2386`]

`main(port)` builds an `aiosmtpd` `Controller(MailHandler(), hostname="0.0.0.0", port=port)` [`email_handler.py:L2383`] with the argparse default port `20381` [`email_handler.py:L2399`]. Captured from `email_handler.log`:

```
2026-07-08 03:57:06,719 - SL - INFO - 593 - "/app/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-08 03:57:06,720 - SL - DEBUG - 593 - "/app/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

These two lines are the "ready to receive alias email on 20381" signal.

### 6. Job runner alive [`job_runner.py:L329-L347`]

The job runner prints the init banner at start; its poll loop calls `get_jobs_to_run()` [`job_runner.py:L307`] and then `time.sleep(10)` [`job_runner.py:L347`]. **Observed:** while the queue is empty the loop is **silent** — `time.sleep(10)` emits nothing and `LOG.d("Take job %s", job)` [`job_runner.py:L334`] fires only when a job is present. From `job_runner.log` at start (idle):

```
>>> init logging <<<
2026-07-08 03:57:06,036 - SL - DEBUG - 594 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

(The poll cadence and `Take job` lines are demonstrated with an enqueued job in **Q3**.)

### 7. Event listener alive [`event_listener.py:L34,L43`, `events/event_source.py:L49`]

The running process is `event_listener.py listener` (LISTENER mode). Captured from `event_listener.log`:

```
2026-07-08 03:58:25,087 - SL - INFO - 637 - "/app/event_listener.py:34" - main() -  - Using PostgresEventSource
2026-07-08 03:58:25,095 - SL - INFO - 637 - "/app/event_listener.py:43" - main() -  - Starting with HttpEventSink
2026-07-08 03:58:25,095 - SL - INFO - 637 - "/app/events/event_source.py:49" - __listen() -  - Starting to listen to events
```

`Starting to listen to events` means the process is attached to the PostgreSQL `LISTEN simplelogin_sync_events` channel and ready.

### 8. `/health` → `200` / `success` [`server.py:L213-L215`]

```console
$ curl -i http://localhost:7777/health
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Content-Type: text/html; charset=utf-8
Content-Length: 7
...
success
```

**Observed subtlety:** `/health` is deliberately **excluded** from the access log [`server.py:L281`]. Verified by firing `/health`, `/auth/login`, `/health` in sequence — only the `/auth/login` request produced a new `after_request` line; neither `/health` call did.

### 9. Rendered login page — UI readiness [`templates/auth/login.html`]

```console
$ curl -s -o /tmp/login.html -w "HTTP %{http_code} bytes=%{size_download}\n" http://localhost:7777/auth/login
HTTP 200 bytes=6918
```

The page contains the heading and form fields:

```html
<h1 class="card-title">Welcome back!</h1>
... <input ... name="csrf_token" ...>
    <input ... name="email" ...>
    <input ... name="password" ...>
    <button ...>Log in</button>
```

### 10. Per-request access log [`server.py:L284`]

`after_request` logs every non-`/health` request with the format `remote_addr method path args status, takes elapsed` [`server.py:L284-L291`] and also records a New Relic `HttpResponseStatus` custom event [`server.py:L293-L295`] (inferred to reach New Relic only when configured; the log line itself is observed). Captured line:

```
2026-07-08 04:37:35,408 - SL - DEBUG - 527 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.0020742416381835938
```

---


## Q2 — New-user walkthrough: register → verify → log in → dashboard

**Direct answer.** Driving the flow through the real `/auth` endpoints:
`POST /auth/register` → logs `create user <email>` [`register.py:L85`] and renders `register_waiting_activation.html` [`register.py:L104`];
`GET /auth/activate?code=…` → flips `users.activated` **False → True** [`activate.py:L49`], logs the user in [`activate.py:L50`], flashes **"Your account has been activated"** [`activate.py:L56`], and **302-redirects to `/dashboard/`** [`activate.py:L67`];
`POST /auth/login` → logs `log user … in` [`login_utils.py:L35`] and **302-redirects to `/dashboard/`** [`login_utils.py:L44`].
All steps and edge cases below are **observed** through the real HTTP entry points (never debug hooks or direct model calls). A temporary user `blitzy-temp-newuser@example.com` (id=3, password `SecretPass123!`) and a second unverified user `blitzy-temp-unverified@example.com` (id=4) were created for the walkthrough and removed in cleanup.

> **Important observed correction about `NOT_SEND_EMAIL`.** The activation **link is NOT printed to the log** in the default config. Under `config.NOT_SEND_EMAIL` [`app/mail_sender.py:L130-L137`], `send()` logs only the subject/from/to metadata and then `return True` — the rendered email **body** (which carries the `…/auth/activate?code=…` link) is neither sent nor logged. This was verified: grepping the register-time log delta for `activate?code=` returned **0 matches**. To obtain the code in the default config, it was read from the `activation_code` table (in production the email would be delivered to the user). This corrects the common assumption that the link is printed to stdout.

### Happy path — Register [`app/auth/views/register.py`]

`POST /auth/register` → **HTTP 200**, rendering `auth/register_waiting_activation.html` [`register.py:L104`] (page shows "Activation" / "check your inbox"). The webapp log delta:

```
2026-07-08 04:39:31,085 - SL - DEBUG - 527 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/register ImmutableMultiDict([]) 200, takes 0.0052449703216552734
2026-07-08 04:39:31,289 - SL - DEBUG - 527 - "/app/app/auth/views/register.py:85" - register() -  - create user blitzy-temp-newuser@example.com
2026-07-08 04:39:31,554 - SL - INFO - 527 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 04:39:31,557 - SL - DEBUG - 527 - "/app/app/models.py:647" - create() -  - Disable onboarding emails
2026-07-08 04:39:31,576 - SL - DEBUG - 527 - "/app/app/email_utils.py:303" - send_email() -  - send email to blitzy-temp-newuser@example.com, subject 'Just one more step to join SimpleLogin'
2026-07-08 04:39:31,577 - SL - DEBUG - 527 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'Just one more step to join SimpleLogin', from '"noreply@sl.local" <noreply@sl.local>' to 'blitzy-temp-newuser@example.com'
2026-07-08 04:39:31,581 - SL - DEBUG - 527 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/register ImmutableMultiDict([]) 200, takes 0.32558107376098633
```

Note the visible confirmations: `create user …` [`register.py:L85`], the activation-email dispatch [`register.py:L117-L129` → `send_activation_email` → `email_utils.py:L303` → `mail_sender.py:L131`], the default `Disable onboarding emails` [`models.py:L647`], and the `Not sending events …` gate [`event_dispatcher.py:L62`]. The real activation code created by this flow (read from the DB, since the link is not logged):

```
activation_code: id=… user_id=3 code=rlxjjqokiucpufhaaeasukyfqtxjic  (activated=f BEFORE, valid 1h per register.py:L118)
```

### Happy path — Verify email [`app/auth/views/activate.py`]

State transition observed **before/after** hitting the real endpoint:

```console
# BEFORE
$ psql ... -tAc "SELECT activated FROM users WHERE id=3;"
f
# Hit the real verification endpoint with the code from registration
$ curl -i "http://localhost:7777/auth/activate?code=rlxjjqokiucpufhaaeasukyfqtxjic"
HTTP/1.1 302 FOUND
Location: http://localhost:7777/dashboard/
# AFTER
$ psql ... -tAc "SELECT activated FROM users WHERE id=3;"
t
```

The webapp log delta shows the welcome-email send [`activate.py:L58`], the redirect log [`activate.py:L66`], and the `302` access line with the `code` arg:

```
2026-07-08 04:40:44,081 - SL - DEBUG - 526 - "/app/app/email_utils.py:303" - send_email() -  - send email to simplelogin-newsletter.list527@sl.local, subject 'Welcome to SimpleLogin'
2026-07-08 04:40:44,083 - SL - DEBUG - 526 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'Welcome to SimpleLogin', from '"noreply@sl.local" <noreply@sl.local>' to 'simplelogin-newsletter.list527@sl.local'
2026-07-08 04:40:44,083 - SL - DEBUG - 526 - "/app/app/auth/views/activate.py:66" - activate() -  - redirect user to dashboard
2026-07-08 04:40:44,083 - SL - DEBUG - 526 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/activate ImmutableMultiDict([('code', 'rlxjjqokiucpufhaaeasukyfqtxjic')]) 302, takes 0.04079866409301758
```

Inside the handler: `user.activated = True` [`activate.py:L49`], `login_user(user)` [`activate.py:L50`], `ActivationCode.delete` [`activate.py:L53`], flash **"Your account has been activated"** [`activate.py:L56`], `send_welcome_email(user)` [`activate.py:L58`], and `redirect(url_for("dashboard.index"))` [`activate.py:L67`]. **Corroboration:** because the code is deleted on success [`activate.py:L53`], a later `SELECT … FROM activation_code WHERE user_id=3` returns **no row** — observed.

**Forwarded into the dashboard** — following the redirect (the `login_user` at activation established the session):

```console
$ curl -s -b jar -o /tmp/dash.html -w "HTTP %{http_code} bytes=%{size_download}\n" http://localhost:7777/dashboard/
HTTP 200 bytes=37541
$ grep -o "Your account has been activated" /tmp/dash.html   # the flash
Your account has been activated
$ grep -o "simplelogin-newsletter" /tmp/dash.html            # first alias present
simplelogin-newsletter
```

The dashboard index [`app/dashboard/views/index.py:L55,L67`] renders 200, shows the activation flash, and lists the user's first alias — proving the new user landed on the dashboard.

### Happy path — Log in [`app/auth/views/login.py`, `login_utils.py`]

With a fresh (returning-user) cookie jar, `POST /auth/login` with the verified credentials → **HTTP 302**, `Location: /dashboard/`. The webapp log delta:

```
2026-07-08 04:41:18,761 - SL - DEBUG - 526 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.0013332366943359375
2026-07-08 04:41:19,205 - SL - DEBUG - 527 - "/app/app/auth/views/login_utils.py:35" - after_login() -  - log user <User 3 blitzy-temp-newuser@example.com blitzy-temp-newuser@example.com> in
2026-07-08 04:41:19,205 - SL - DEBUG - 527 - "/app/app/auth/views/login_utils.py:44" - after_login() -  - redirect user to dashboard
2026-07-08 04:41:19,205 - SL - DEBUG - 527 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 302, takes 0.2407398223876953
```

On success `login()` calls `LoginEvent(LoginEvent.ActionType.success).send()` [`login.py:L71`] then `after_login` [`login.py:L72` → `login_utils.py:L12`], which logs `log user … in` [`login_utils.py:L35`] and redirects to the dashboard [`login_utils.py:L44-L45`]. (For accounts with 2FA, `after_login` would branch to FIDO [`login_utils.py:L20`] or OTP [`login_utils.py:L28`] before the dashboard — **(inferred)** from code; the temp account has no 2FA so the direct dashboard redirect was observed.)

### Edge / error conditions (each exercised through the real endpoint)

**Invalid activation code → HTTP 400 "Activation code cannot be found"** [`activate.py:L33`]:

```console
$ curl -s -o /tmp/a.html -w "HTTP %{http_code}\n" "http://localhost:7777/auth/activate?code=bogus-does-not-exist"
HTTP 400
$ grep -o "Activation code cannot be found" /tmp/a.html
Activation code cannot be found
```

**Expired activation code → HTTP 400 "Activation code was expired"** + resend offered [`activate.py:L38,L42-L43`]. User 4's code expiry was set to `2020-01-01` (a data-only change on a temp row; the real handler path was exercised):

```console
$ curl -s -o /tmp/e.html -w "HTTP %{http_code}\n" "http://localhost:7777/auth/activate?code=iunqzeuvhewmiyfbywggccqbzquwcn"
HTTP 400
$ grep -o "Activation code was expired" /tmp/e.html
Activation code was expired
```

`is_expired()` [`activate.py:L38`] triggers the `flash(..., "error")` + `render_template("auth/activate.html", show_resend_activation=True), 400` at [`activate.py:L42-L43`].

**Wrong password → HTTP 200 "Email or password incorrect"** [`login.py:L49`] + `LoginEvent(...failed)` [`login.py:L50`]:

```console
$ curl -s -o /tmp/w.html -w "HTTP %{http_code}\n" -c jar -b jar --data "csrf_token=…&email=blitzy-temp-newuser@example.com&password=WRONGpassword" http://localhost:7777/auth/login
HTTP 200
$ grep -o "Email or password incorrect" /tmp/w.html
Email or password incorrect
```

**Login by a not-yet-activated user → HTTP 200, prompt to check inbox** [`login.py:L63-L69`]. User 4 (never verified):

```console
$ grep -o "Please check your inbox for the activation email. You can also have this email re-sent" /tmp/u4.html
Please check your inbox for the activation email. You can also have this email re-sent
```

This is the `elif not user.activated:` branch [`login.py:L63`] with `show_resend_activation=True` and the flash at [`login.py:L65-L68`], followed by `LoginEvent(...not_activated).send()` [`login.py:L69`].

**Resend activation** [`app/auth/views/resend_activation.py`]. `POST /auth/resend_activation` for user 4 → HTTP 200 waiting page; log:

```
... - SL - ... "/app/app/auth/views/resend_activation.py:36" - resend_activation() -  - user <User 4 ...> is not activated
... - SL - DEBUG - ... "/app/app/email_utils.py:303" - send_email() -  - send email to blitzy-temp-unverified@example.com, subject 'Just one more step to join SimpleLogin'
```

**`DISABLE_REGISTRATION` (non-default toggle) → registration closed** [`register.py:L38-L40`]. The committed `.env` was **not** modified; instead a throwaway Gunicorn instance was launched on port **7778** with `DISABLE_REGISTRATION=1` in its environment [`app/config.py` reads `"DISABLE_REGISTRATION" in os.environ`]. `POST /auth/register` → **HTTP 302 → `/auth/login`**, and the followed page flashes **"Registration is closed"**; no user row was created (count = 0). The throwaway instance was then stopped.

> **Citation correction (observed):** on this branch the `DISABLE_REGISTRATION` path **redirects to `auth.login`** with a flash, rather than re-rendering the register page with an inline error — `flash("Registration is closed", "error")` + `redirect(url_for("auth.login"))` [`register.py:L39-L40`].

**Disabled account → HTTP 200 "Your account is disabled…"** [`login.py:L52-L55`]. Temporarily setting `users.disabled=true` on the temp user and logging in produced:

```
Your account is disabled. Please contact SimpleLogin team to re-enable your account.
```

with `disabled_login` telemetry [`login.py:L56`]; the flag was reset to `false` afterward. **(Observed** — triggered via a reversible data change on the temp user, then restored.)

**Scheduled-for-deletion → HTTP 200 "Your account is scheduled to be deleted on …"** [`login.py:L58-L61`]. Temporarily setting `users.delete_on=2030-01-01` produced:

```
Your account is scheduled to be deleted on 2030-01-01T00:00:00+00:00
```

with `scheduled_to_be_deleted` telemetry [`login.py:L62`]; `delete_on` was reset to `NULL` afterward. **(Observed** — reversible data change on the temp user, then restored.)

### UI signals captured

Rendered HTML was saved for each step: `login.html` ("Welcome back!" + email/password/csrf inputs + "Log in" button), `register_waiting_activation.html` ("Activation" / "check your inbox"), `activate.html` error variants ("Activation code cannot be found" / "Activation code was expired" with a resend link), and the dashboard landing page (activation flash + the `simplelogin-newsletter` first alias).

---


## Q3 — Behind-the-scenes background jobs & internal services

**Direct answer.** Yes. The new-user flow sets several internal services in motion, and they were observed active and communicating: (a) `User.create` provisions a **verified default mailbox** and a **first alias** and (in production) enqueues onboarding jobs [`app/models.py:L601-L668`]; (b) a **DB-backed `Job` queue** is drained by `job_runner.py` on a **measured ~10.02 s poll**, stable across two runs [`job_runner.py:L307,L334,L347`]; (c) the **`email_handler.py` `aiosmtpd` service on :20381** forwards mail via `handle_forward` [`email_handler.py:L536`], ending with the `Finish mail_from …` completion line [`email_handler.py:L2367`]; and (d) the **event pipeline** persists a `SyncEvent` and issues a PostgreSQL `NOTIFY simplelogin_sync_events` [`app/events/event_dispatcher.py:L25-L26`] that the running `event_listener.py` consumes — demonstrated end-to-end. Supporting machinery: New Relic telemetry [`app/events/auth_event.py`], the `yacron` scheduler [`crontab.yml`] (observed firing every 5 min), and the Postfix-queue monitor [`monitoring.py`].

### 1. Identity provisioning inside `User.create` [`app/models.py:L601-L668`]

For the new user (id=3), `User.create` created a **verified default mailbox** [`models.py:L611` `Mailbox.create(..., verified=True)`] and a **first alias** with `prefix="simplelogin-newsletter"` [`models.py:L636`], and wired `default_mailbox_id` / `newsletter_alias_id` [`models.py:L613,L643`]. Observed via `psql`:

```
 id |               email                | activated | default_mailbox_id | newsletter_alias_id
----+------------------------------------+-----------+--------------------+---------------------
  3 | blitzy-temp-newuser@example.com    | t         |                  5 |                  12

 id | user_id |              email              | verified
----+---------+---------------------------------+----------
  5 |       3 | blitzy-temp-newuser@example.com | t

 id | user_id |                  email                  | note60
----+---------+-----------------------------------------+--------------------------------------------------------------
 12 |       3 | simplelogin-newsletter.list527@sl.local | This is your first alias. It's used to receive SimpleLogin c
```

**Onboarding jobs — default vs production.** Under the default `DISABLE_ONBOARDING=true`, `User.create` logs `Disable onboarding emails` and **returns before enqueuing** [`models.py:L646-L648`]; observed 0 onboarding jobs:

```console
$ psql ... -tAc "SELECT count(*) FROM job WHERE name LIKE 'onboarding%';"
0
```

**Production contrast (inferred from code):** with the flag unset, the `else` branch [`models.py:L650-L665`] would enqueue `JOB_ONBOARDING_1` / `_2` / `_4` (`onboarding-1/2/4` per `app/config.py:L301-L304`). The `create user …` register log delta above already shows the `Disable onboarding emails` line firing in the default config.

### 2. DB-backed `Job` queue drained by `job_runner.py` — timing measured [`job_runner.py:L307,L334,L347`]

The `__main__` loop calls `get_jobs_to_run()` [`job_runner.py:L307`], logs `Take job %s` [`job_runner.py:L334`] for each, dispatches via `process_job` [`job_runner.py:L188-L304`], marks the job done, and sleeps `time.sleep(10)` [`job_runner.py:L347`]. Jobs were enqueued through the **real** `Job.create(...)` API and picked up by the already-running runner (PID 594).

**Magnitude/frequency claim — poll cadence.** Eight jobs were enqueued and drained back-to-back. Measuring the delta between consecutive `Take job` timestamps over **~70 s** (jobs 2→9), split into two runs of three intervals each:

| Run | Jobs (Take-job times) | Intervals (s) | Mean |
|-----|-----------------------|---------------|------|
| RUN 1 | Job2 04:48:40.700 → Job3 :50.719 → Job4 04:49:00.740 → Job5 :10.759 | 10.019, 10.021, 10.019 | **10.02** |
| RUN 2 | Job6 04:49:20.772 → Job7 :30.790 → Job8 :40.811 → Job9 :50.828 | 10.018, 10.021, 10.017 | **10.02** |

The cadence is **stable at ~10.02 s across both runs** (the extra ~0.02 s over the literal `sleep(10)` is the `get_jobs_to_run` query + dispatch). Raw `Take job` + dispatch lines (excerpt from `job_runner.log`):

```
2026-07-08 04:48:40,700 - SL - DEBUG - 594 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 2 send-alias-creation-events {'user_id': 3}>
2026-07-08 04:48:40,704 - SL - DEBUG - 594 - "/app/job_runner.py:299" - process_job() -  - Sending alias creation events for <User 3 blitzy-temp-newuser@example.com blitzy-temp-newuser@example.com>
2026-07-08 04:48:40,704 - SL - INFO - 594 - "/app/app/jobs/event_jobs.py:17" - send_alias_creation_events_for_user() -  - Sending alias create events for user {user}
2026-07-08 04:48:40,706 - SL - INFO - 594 - "/app/app/jobs/event_jobs.py:44" - send_alias_creation_events_for_user() -  - Sending 1 alias create event for <User 3 blitzy-temp-newuser@example.com blitzy-temp-newuser@example.com>
2026-07-08 04:48:50,719 - SL - DEBUG - 594 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 3 send-alias-creation-events {'user_id': 3}>
2026-07-08 04:49:00,740 - SL - DEBUG - 594 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 4 send-alias-creation-events {'user_id': 3}>
2026-07-08 04:49:10,759 - SL - DEBUG - 594 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 5 send-alias-creation-events {'user_id': 3}>
2026-07-08 04:49:20,772 - SL - DEBUG - 594 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 6 send-alias-creation-events {'user_id': 3}>
2026-07-08 04:49:30,790 - SL - DEBUG - 594 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 7 send-alias-creation-events {'user_id': 3}>
2026-07-08 04:49:40,811 - SL - DEBUG - 594 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 8 send-alias-creation-events {'user_id': 3}>
2026-07-08 04:49:50,828 - SL - DEBUG - 594 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 9 send-alias-creation-events {'user_id': 3}>
```

Each job reached the terminal state `done` — verified against the `JobState` enum `ready=0, taken=1, done=2` [`app/models.py:L253-L256`]:

```
 id |            name            | taken | state | attempts
----+----------------------------+-------+-------+----------
  1 | send-alias-creation-events | t     |     2 |        1
  ...
  9 | send-alias-creation-events | t     |     2 |        1
(9 rows)
```

**Dispatched job types** handled by `process_job` [`job_runner.py:L188-L304`] and named in `app/config.py:L301-L311`: `JOB_ONBOARDING_1/2/4`, `JOB_BATCH_IMPORT`, `JOB_DELETE_ACCOUNT`, `JOB_DELETE_MAILBOX`, `JOB_DELETE_DOMAIN`, `JOB_SEND_USER_REPORT` (runs `ExportUserDataJob`), `JOB_SEND_PROTON_WELCOME_1`, and `JOB_SEND_ALIAS_CREATION_EVENTS` (`send-alias-creation-events`, the one exercised); anything else logs `Unknown job name` [`job_runner.py:L304`].

### 3. Email handler forwarding on :20381 [`email_handler.py:L536,L2289,L2367`]

A test message was injected into the running handler with `swaks` (per `CONTRIBUTING.md:L218`), addressed to the new user's alias so the resulting rows cascade-clean with the user:

```console
$ swaks --to simplelogin-newsletter.list527@sl.local --from external-sender@google.com \
        --server 127.0.0.1:20381 --header 'Subject: Blitzy Q3 forward test' --body '...'
...
<-  250 Message accepted for delivery
```

The complete forward path captured from `email_handler.log` (PID 593): `handle_DATA` [`email_handler.py:L2289`] → `New message …` [`L2343`] → `Forward phase …` [`L2202`] → `handle_forward()` [`L536`] creating the contact → `forward_email_to_mailbox` resolving alias→mailbox → reverse-alias `From` rewrite → the `Finish mail_from …` completion line [`L2367`]:

```
2026-07-08 04:50:17,086 - SL - INFO - 593 - "/app/email_handler.py:2343" - _handle() - ... - New message, mail from external-sender@google.com, rctp tos ['simplelogin-newsletter.list527@sl.local']
2026-07-08 04:50:17,094 - SL - DEBUG - 593 - "/app/email_handler.py:2202" - handle() - ... - Forward phase external-sender@google.com(external-sender@google.com) -> simplelogin-newsletter.list527@sl.local
2026-07-08 04:50:17,103 - SL - DEBUG - 593 - "/app/email_handler.py:580" - handle_forward() - ... - Create or get contact for from_header:external-sender@google.com
2026-07-08 04:50:17,122 - SL - DEBUG - 593 - "/app/app/contact_utils.py:110" - create_contact() - ... - Created contact <Contact 3 external-sender@google.com 12> for alias <Alias 12 simplelogin-newsletter.list527@sl.local> with email external-sender@google.com invalid_email=False
2026-07-08 04:50:17,130 - SL - DEBUG - 593 - "/app/email_handler.py:688" - forward_email_to_mailbox() - ... - Forward <Contact 3 external-sender@google.com 12> -> <Alias 12 simplelogin-newsletter.list527@sl.local> -> <Mailbox 5 blitzy-temp-newuser@example.com>
2026-07-08 04:50:17,132 - SL - DEBUG - 593 - "/app/email_handler.py:740" - forward_email_to_mailbox() - ... - Create <EmailLog 3> for <Contact 3 external-sender@google.com 12>, <User 3 ...>, <Mailbox 5 blitzy-temp-newuser@example.com>
2026-07-08 04:50:17,137 - SL - DEBUG - 593 - "/app/email_handler.py:867" - forward_email_to_mailbox() - ... - From header, new:"external-sender at google.com" <external-sender_at_google_com_tgckofhmpj@sl.local>, old:external-sender@google.com
2026-07-08 04:50:17,138 - SL - DEBUG - 593 - "/app/app/mail_sender.py:131" - send() - ... - send email with subject 'Blitzy Q3 forward test', from '"external-sender at google.com" <external-sender_at_google_com_tgckofhmpj@sl.local>' to 'simplelogin-newsletter.list527@sl.local'
2026-07-08 04:50:17,138 - SL - INFO - 593 - "/app/email_handler.py:2367" - _handle() - ... - Finish mail_from external-sender@google.com, rcpt_tos ['simplelogin-newsletter.list527@sl.local'], takes 0.052645206451416016 seconds with return code '250 Message accepted for delivery'<<===
```

This proves the alias-based receive→forward path works: an external sender's mail hitting the alias is resolved to the owning mailbox, a reverse-alias `From` is generated (`external-sender_at_google_com_…@sl.local`), and an `EmailLog` row is created. The reply-phase counterpart `handle_reply` [`email_handler.py:L966`] processes replies to that reverse-alias — **(inferred** from code; the reply phase was not exercised in this run).

### 4. Event pipeline — `SyncEvent` + `NOTIFY simplelogin_sync_events` [`app/events/event_dispatcher.py`, `event_listener.py`, `events/*`]

**Default-config gate (observed).** In the normal flow, `EventDispatcher.send_event` short-circuits [`app/events/event_dispatcher.py:L61-L65`] when `EVENT_WEBHOOK` is unset (`app/config.py:L612`), logging `Not sending events because webhook is not configured and allowed to be empty` [`event_dispatcher.py:L62`] — seen in both the register delta and every job dispatch above — so no `SyncEvent` is persisted through the job path by default. The pipeline is **running but idle** by default.

**Mechanism demonstrated end-to-end.** To exercise the real persist + notify + consume path, `PostgresDispatcher.send` [`app/events/event_dispatcher.py:L23-L26`] was invoked directly (the same call the code uses), which runs `SyncEvent.create(content=event, flush=True)` [`L25`] and `Session.execute("NOTIFY simplelogin_sync_events, '<id>';")` [`L26`, channel constant `L14`]. A separate `LISTEN` connection captured the notification, and the **running** `event_listener.py` (PID 637) consumed it. Observation-script output:

```
LISTENING on simplelogin_sync_events
DISPATCHED via PostgresDispatcher.send + commit
NOTIFY RECEIVED: channel=simplelogin_sync_events payload=1
```

The persisted row (content stored as protobuf `bytea`; the demo payload was the ASCII string `blitzy-q3-event-pipeline-demo`):

```
 id |                 content                  | retry_count
----+------------------------------------------+-------------
  1 | \x626c69747a792d71332d6576656e742d706970 |           1
```

The running listener's log shows it picked the event up and — with no webhook configured — declined to deliver, so the `Runner` increments `retry_count` instead of deleting the row [`events/runner.py:L20-L46`, `events/event_sink.py:L17-L20`]:

```
2026-07-08 04:51:52,905 - SL - DEBUG - 637 - "/app/events/event_source.py:55" - __listen() -  - Got NOTIFY: pid=4943 channel=simplelogin_sync_events payload=1
2026-07-08 04:51:52,975 - SL - WARNING - 637 - "/app/events/event_sink.py:19" - process() -  - Skipping sending event because there is no webhook configured
```

This closes the loop: **dispatch → PostgreSQL `NOTIFY` → listener consume** across two processes, proving the event machinery is active and communicating. The `main()` wiring is `PostgresEventSource` (LISTENER mode) → `HttpEventSink` via `Runner(source, sink).run()` [`event_listener.py:L29-L47`]; a `DeadLetterEventSource` (DEAD_LETTER mode) and `ConsoleEventSink` (dry-run) are the alternates.

### 5. Supporting components (named + cited)

- **Jobs:** `app/jobs/event_jobs.py` → `send_alias_creation_events_for_user` [`L9`] builds `AliasCreated`/`AliasCreatedList` from the protobuf schema; observed firing (`Sending 1 alias create event …`, `event_jobs.py:L44`). `app/jobs/export_user_data_job.py` → `class ExportUserDataJob` [`L41`], `run` [`L131`], `create_from_job` [`L167`] — GDPR user-report export, dispatched by `JOB_SEND_USER_REPORT`.
- **Event source / sink / runner:** `events/event_source.py` — `PostgresEventSource` [`L27`], `DeadLetterEventSource` [`L84`], `__listen` [`L41,L49,L55`]; `events/event_sink.py` — `HttpEventSink` [`L16-L40`], `ConsoleEventSink` [`L43-L46`]; `events/runner.py` — `Runner` [`L11`], `run` [`L16`], `__on_event` [`L20`]; plus `events/event_debugger.py`.
- **Protobuf schema:** `app/events/generated/event_pb2.py` — symbols `AliasCreated`, `AliasCreatedList`, `AliasDeleted`, `EventContent`, `UserPlanChanged`.
- **New Relic telemetry:** `app/events/auth_event.py` — `LoginEvent` [`L6`] / `RegisterEvent` [`L28`], each `.send()` → `newrelic.agent.record_custom_event(...)` [`L23-L24,L45-L46`] (these fire during the Q2 login/register above). `monitor/newrelic.py` (uses `newrelic_telemetry_sdk` `GaugeMetric`/`MetricClient`, host `metric-api.eu.newrelic.com`, `send` [`L12`]); `monitor/metric_exporter.py` — `class MetricExporter` [`L7`], `run` [`L14`].
- **Cron scheduler:** `yacron -c crontab.yml` (PID 1238) actively spawns jobs via `/code/cron.py` (`/code`→`/app` symlink), dispatched by `-j <job>` [`cron.py:L1265`]. **Cadence observed (magnitude/frequency):** `send_undelivered_mails` (`crontab.yml` schedule `*/5 * * * *`) ran at `04:15:01, 04:20:01, 04:25:01, 04:30:02, 04:35:02, 04:40:02, 04:45:01, 04:50:01` — **exactly every 5 minutes, stable across 8 runs.** Raw excerpt:

```
2026-07-08 04:15:01,673 - SL - DEBUG - 1264 - "/code/cron.py:1263" - <module>() -  - Start running cronjob
2026-07-08 04:15:01,844 - SL - DEBUG - 1264 - "/code/cron.py:1195" - notify_hibp() -  - Send new breaches found email to <User 1 John Wick john@wick.com> for 2 breaches aliases
INFO:yacron:Cron job SimpleLogin send unsent emails: reporting success
2026-07-08 04:20:01,678 - SL - DEBUG - 1315 - "/code/cron.py:1263" - <module>() -  - Start running cronjob
INFO:yacron:Cron job SimpleLogin send unsent emails: reporting success
2026-07-08 04:50:01,554 - SL - DEBUG - 4839 - "/code/cron.py:1263" - <module>() -  - Start running cronjob
INFO:yacron:Cron job SimpleLogin send unsent emails: reporting success
```

  `crontab.yml` defines 15 jobs in total, e.g. `stats` `0 0 * * *`, `notify_hibp` `15 4 * * *`, `send_undelivered_mails` `*/5 * * * *`, `clear_alias_audit_log`/`clear_user_audit_log` `0 * * * *`.
- **Postfix-queue monitor:** `monitoring.py` — `log_postfix_metrics` [`L39`] decorated `@newrelic.agent.background_task()` [`L38`], alerts when incoming+active queue `> 50` [`L23`] after 10 consecutive fails [`L20`], using `MetricExporter` [`L12`]. This is a separate optional process; it was **not running** in this container (**inferred** from code — cited, not executed here).

---


## Coverage checklist

Every named item across Q1/Q2/Q3, with its status. **Observed** = exercised at runtime with captured output; **(inferred)** = read from code (labelled as such in-text).

### Q1 — Startup & readiness

| Item | `file:line` | Status |
|------|-------------|--------|
| `>>> init logging <<<` banner | `app/log.py:L67` | Observed |
| Gunicorn `Starting/Listening/Using worker/Booting worker` | `Dockerfile:L47`, `wsgi.py:L1-L3` | Observed |
| Blueprints registered (auth, monitor, dashboard, developer, phone, oauth, onboarding, discover, internal, api) | `server.py:L233-L246` | Observed |
| `/auth` and `/dashboard` prefixes | `app/auth/base.py`, `app/dashboard/base.py:L3-L8` | Observed |
| init_app SL-domain seeding + PGP key load | `init_app.py:L16,L36,L42,L44` | Observed |
| Email handler `Listen for port 20381` / `Start mail controller` | `email_handler.py:L2403,L2386` | Observed |
| Job runner alive; idle poll silent | `job_runner.py:L329-L347` | Observed |
| Event listener `Using PostgresEventSource`/`HttpEventSink`/`Starting to listen` | `event_listener.py:L34,L43`, `events/event_source.py:L49` | Observed |
| `/health` → 200 / `success` | `server.py:L213-L215` | Observed |
| `/health` excluded from access log | `server.py:L281` | Observed |
| Rendered login page | `templates/auth/login.html` | Observed |
| Per-request access log | `server.py:L284` | Observed |
| New Relic `HttpResponseStatus` custom event | `server.py:L293-L295` | (inferred) delivery; log line observed |

### Q2 — New-user walkthrough

| Item | `file:line` | Status |
|------|-------------|--------|
| `create user …` log | `register.py:L85` | Observed |
| `register_waiting_activation.html` render | `register.py:L104` | Observed |
| Activation-email dispatch path | `register.py:L117-L129`, `email_utils.py:L303`, `mail_sender.py:L131` | Observed |
| `NOT_SEND_EMAIL` → link NOT logged (correction) | `mail_sender.py:L130-L137` | Observed |
| `users.activated` False→True (before/after) | `activate.py:L49` | Observed |
| `login_user` at activation | `activate.py:L50` | Observed |
| Activation code deleted on success | `activate.py:L53` | Observed |
| Flash "Your account has been activated" | `activate.py:L56` | Observed |
| Welcome email | `activate.py:L58` | Observed |
| 302 → `/dashboard/` on verify | `activate.py:L67` | Observed |
| Dashboard landing (200 + flash + first alias) | `app/dashboard/views/index.py:L55,L67` | Observed |
| `log user … in` + 302 on login | `login_utils.py:L35,L44` | Observed |
| `LoginEvent(success)` | `login.py:L71` | Observed |
| FIDO / OTP 2FA branches | `login_utils.py:L20,L28` | (inferred) |
| Invalid code → 400 "Activation code cannot be found" | `activate.py:L33` | Observed |
| Expired code → 400 "Activation code was expired" + resend | `activate.py:L38,L42-L43` | Observed |
| Wrong password → "Email or password incorrect" + failed event | `login.py:L49,L50` | Observed |
| Not-activated login → check-inbox prompt + not_activated event | `login.py:L63-L69` | Observed |
| Resend activation | `resend_activation.py:L36` | Observed |
| `DISABLE_REGISTRATION` → 302 to login, "Registration is closed" (correction: redirects) | `register.py:L38-L40` | Observed |
| Disabled account → "Your account is disabled…" + disabled_login | `login.py:L52-L56` | Observed |
| Scheduled-for-deletion → "…scheduled to be deleted…" + event | `login.py:L58-L62` | Observed |

### Q3 — Behind-the-scenes

| Item | `file:line` | Status |
|------|-------------|--------|
| `User.create` verified default mailbox | `app/models.py:L611` | Observed |
| `User.create` first alias `simplelogin-newsletter…` | `app/models.py:L636` | Observed |
| Default onboarding skip (`Disable onboarding emails`, 0 jobs) | `app/models.py:L646-L648` | Observed |
| Production onboarding enqueue (`onboarding-1/2/4`) | `app/models.py:L650-L665`, `app/config.py:L301-L304` | (inferred) |
| Job queue drained; `Take job` | `job_runner.py:L334` | Observed |
| `process_job` dispatch (all job types named) | `job_runner.py:L188-L304`, `app/config.py:L301-L311` | Observed (send-alias-creation-events run) |
| **Poll cadence ~10.02 s, stable across 2 runs, ~70 s** | `job_runner.py:L347` | Observed |
| `JobState` done=2 for all 9 jobs | `app/models.py:L253-L256` | Observed |
| `send_alias_creation_events_for_user` | `app/jobs/event_jobs.py:L9,L17,L44` | Observed |
| `ExportUserDataJob` (GDPR report) | `app/jobs/export_user_data_job.py:L41,L131,L167` | (inferred) |
| Email forward via swaks → `handle_forward` | `email_handler.py:L536,L2289,L2202` | Observed |
| `Finish mail_from …` completion | `email_handler.py:L2367` | Observed |
| Reverse-alias `From` rewrite + `EmailLog` | `email_handler.py:L867,L740` | Observed |
| `handle_reply` (reply phase) | `email_handler.py:L966` | (inferred) |
| Event gate when `EVENT_WEBHOOK` unset | `event_dispatcher.py:L61-L65`, `config.py:L612` | Observed |
| `SyncEvent.create` + `NOTIFY simplelogin_sync_events` | `event_dispatcher.py:L14,L25,L26` | Observed |
| Listener consumes `Got NOTIFY …` | `events/event_source.py:L55` | Observed |
| `HttpEventSink` skip / `Runner` retry | `events/event_sink.py:L17-L20`, `events/runner.py:L20-L46` | Observed |
| Protobuf schema symbols | `app/events/generated/event_pb2.py` | Observed (symbols present) |
| New Relic `LoginEvent`/`RegisterEvent` | `app/events/auth_event.py:L6,L28` | Observed (fire in Q2) |
| `MetricExporter` / `monitor/newrelic.py` | `monitor/metric_exporter.py:L7,L14`, `monitor/newrelic.py:L12` | (inferred) |
| **Cron `send_undelivered_mails` every 5 min, stable across 8 runs** | `crontab.yml` (`*/5 * * * *`), `cron.py:L1263` | Observed |
| `notify_hibp` fired | `cron.py:L1195` | Observed |
| Postfix-queue monitor | `monitoring.py:L12,L20,L23,L38,L39` | (inferred) not running here |

---

## Repository left unchanged (read-only verification)

This was a strictly read-only investigation. No source file was modified; the only artifact added is this document. All temporary test data created for observation (users `blitzy-temp-newuser@example.com` id=3 and `blitzy-temp-unverified@example.com` id=4, plus their cascade rows — mailbox 5/6, alias 12/13, contact, email_log — and the 9 temporary `job` rows and 1 demo `sync_event` row) and all temporary observation scripts were removed on completion; no `.env` value was persistently changed (`DISABLE_REGISTRATION` was applied only as an environment variable to a throwaway instance on port 7778). The expected final state of the working tree is that `git status --porcelain` shows only this new untracked document and `git diff` is empty (no tracked file modified or deleted).

```console
$ git status --porcelain
?? blitzy/documentation/app_2cd6ee777f8c.md
$ git diff --stat
(empty)
```

*(All `file:line` citations and captured output correspond to branch `app_2cd6ee777f8c`, HEAD `2cd6ee77`, in the default `example.env` configuration.)*

