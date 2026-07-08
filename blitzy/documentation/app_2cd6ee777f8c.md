# SimpleLogin — Local Bring-Up & Runtime Observation Report

**Source branch:** `app_2cd6ee777f8c` (HEAD `2cd6ee77`) · **Deliverable:** `blitzy/documentation/app_2cd6ee777f8c.md`

This document answers three operator questions about bringing up and observing the SimpleLogin
back-end locally, **from live runtime observation** of the canonical Docker image
`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (tag `simple-login__app__2cd6ee77…`):

- **Q1 — Startup & readiness.** After the system starts, what in the **logs or the UI** shows the
  application is ready to handle authentication and alias-based email activity?
- **Q2 — New-user walkthrough.** Walk through **register → verify email → log in** and identify the
  visible behavior confirming each step and the forward into the dashboard.
- **Q3 — Behind the scenes.** What does the app do behind the scenes during that flow — are there
  **background jobs or internal services** for email forwarding / identity verification, and what at
  runtime shows they are active and communicating?

Every behavioral claim below is paired with the **exact command that produced it** and the
**complete, unedited output** captured at runtime, plus a `file:line` citation into the source at
branch `app_2cd6ee777f8c`. Statements that were **inferred from reading** (not exercised at runtime)
are explicitly labeled `(inferred)`. Two pieces of evidence are explicitly labeled
**NON-CANONICAL DIAGNOSTIC** / **observation-setup** where a mechanism could not be reached through a
product entry point under the default configuration (Q3 events, job cadence) — see those sections.

The investigation created temporary users/aliases only to exercise the flows and **removed all of
them**; the source tree is left byte-for-byte unchanged (only this document is added). See
**§6 Build/Run, Read-only & Cleanup Proof**.

---

## Command conventions (reproducibility)

All observations were captured with `docker exec simplelogin-app …`. Inside the container the five
SimpleLogin processes were already running (see §6) and write to `/app/logs/{webapp,email_handler,job_runner,event_listener,yacron}.log`.

Database inspection uses the verbatim `.env` DB credentials
(`DB_URI` → `postgres://myuser:mypassword@localhost:5432/simplelogin` [`example.env`]):

```
PGPASSWORD=mypassword psql -h localhost -U myuser -d simplelogin -c "<SQL>"
```

Authenticated POSTs are CSRF-protected (Flask-WTF). Every such command below uses a per-session
cookie jar `$JAR` and a `$CSRF` shell variable acquired from the form's hidden
`csrf_token` field with this exact, reproducible helper (run against the same cookie jar
immediately before the POST):

```
JAR=$(mktemp)
CSRF=$(curl -sS -c "$JAR" http://localhost:7777/auth/login \
       | grep -oP 'name="csrf_token" type="hidden" value="\K[^"]+')
```

Ephemeral session secrets (the `csrf_token` value and the `slapp=` session-cookie blob) are shown as
captured where they appear in a raw HTTP transcript; they belong to temporary users that were
subsequently deleted (see §6) and are inert. In command lines the token is passed as the `$CSRF`
variable (its acquisition is documented above) — this is the actual, reproducible invocation.

---

## Summary — direct answers

**Q1 (ready?).** Readiness is visible in the **logs** as five independent signals, one per process,
plus a live HTTP probe: the webapp prints the Gunicorn boot sequence ending
`Listening at: http://0.0.0.0:7777` and `Booting worker with pid: …` [`Dockerfile:L47`], each process
prints `>>> init logging <<<` when the `SL` logger initializes [`app/log.py:L67`], the email handler
prints `Listen for port 20381` + `Start mail controller 0.0.0.0 20381` [`email_handler.py:L2403,L2386`],
the event listener prints `Using PostgresEventSource` + `Starting with HttpEventSink` +
`Starting to listen to events` [`event_listener.py:L34,L43`; `events/event_source.py:L49`], the job
runner prints its init banner then idles silently, and `yacron` logs each scheduled spawn. The live
readiness confirmation is `GET /health` → `HTTP/1.1 200 OK` body `success`
[`server.py:L213-L215`]. In the **UI**, `GET /auth/login` returns `HTTP 200` rendering the
`Welcome back!` login card with the `email`/`password`/`csrf_token` form. Each handled request (except
`/health`, which is excluded [`server.py:L281`]) is logged by `after_request` [`server.py:L284`].

**Q2 (walkthrough).** `POST /auth/register` creates the user (`create user …`
[`app/auth/views/register.py:L85`]) and renders the `register_waiting_activation.html`
"check your inbox" page. **Default-config note (see §2.2):** under `NOT_SEND_EMAIL=true`
[`example.env:L19`] the activation **link is NOT printed to the log** — only the email
subject/from/to is [`app/mail_sender.py:L130-L137`]; the AAP's expectation of a printed activation
link is **not reproduced**. `GET /auth/activate?code=<code>` flips `users.activated` from `f`→`t`,
deletes the code, flashes **"Your account has been activated"** and **302-redirects to
`/dashboard/`** [`app/auth/views/activate.py:L49,L53,L56,L66`]. `POST /auth/login` logs
`log user <User …> in` and redirects to `/dashboard/`
[`app/auth/views/login_utils.py:L35,L44`]. All edge/error paths were also exercised (invalid code →
400, expired code → 400, wrong password → 200, not-activated login → 200, resend, disabled account,
scheduled-deletion, and `DISABLE_REGISTRATION` → 302) — see §2.6.

**Q3 (behind the scenes).** Yes — the flow is backed by internal services that are observably active
and communicating. On registration, `User.create` provisions a **verified default mailbox** and a
**first alias**, and (under `DISABLE_ONBOARDING=true` [`example.env:L150`]) logs
`Disable onboarding emails` and enqueues **no** onboarding jobs [`app/models.py:L611,L636,L646-L648`].
Real product paths (registration and dashboard alias creation) reach the **event dispatcher**, which
under the default config (no `EVENT_WEBHOOK`) short-circuits at
`Not sending events because webhook is not configured…` [`app/events/event_dispatcher.py:L62`]; the
full persist→`NOTIFY`→consume mechanism is demonstrated via a clearly-labeled diagnostic (§3.3). The
**email handler** on `:20381` forwards a real test message external-sender → alias → owning mailbox,
finishing `Finish mail_from … '250 Message accepted for delivery'`
[`email_handler.py:L2367`]. The **job runner** drains work on a **~10.02s** poll
[`job_runner.py:L347`], demonstrated end-to-end through the real `POST /dashboard/delete_account`
entry point (enqueues `delete-account`, runner deletes the user + cascade). `yacron` runs
`send_undelivered_mails` every **~300s** (`*/5` schedule) [`crontab.yml:L80`].

### Default local configuration that shapes what is observed

The container runs the canonical `example.env` copied to `.env`, unmodified. The values that change
observable behavior versus production:

| Key | Value | `file:line` | Effect on what you observe |
|-----|-------|-------------|----------------------------|
| `URL` | `http://localhost:7777` | `example.env:L6` | Base URL in logs/links |
| `NOT_SEND_EMAIL` | `true` | `example.env:L19` | Email is **printed** (subject/from/to only), not sent — activation link is **not** logged [`app/mail_sender.py:L130-L137`] |
| `EMAIL_DOMAIN` | `sl.local` | `example.env:L22` | Alias domain (e.g. `…@sl.local`) |
| `DISABLE_ONBOARDING` | `true` | `example.env:L150` | `User.create` skips onboarding jobs [`app/models.py:L646-L648`]; job runner idles |
| `DISABLE_REGISTRATION` | *(commented/unset)* | `example.env:L58`; `app/config.py:L138` | Registration **open** by default |
| `EVENT_WEBHOOK` | *(unset)* | `app/config.py:L612` | Event pipeline short-circuits at dispatcher gate [`app/events/event_dispatcher.py:L62`] |
| `MEM_STORE_URI` | *(unset)* | `app/config.py:L568` | Cookie sessions; login works without Redis |

---

## Q1 — Startup & readiness signals (logs and UI)

**Direct answer.** Readiness shows up as **five per-process log signals** and **two live HTTP
signals**. Each of the five long-running processes prints an unmistakable "I'm up" line; the webapp
additionally answers `GET /health` with `200 success` and renders the login page. Below is the
complete captured evidence for each, with the exact command.

### Q1.1 Webapp — Gunicorn boot sequence + `>>> init logging <<<`

The production launch command is `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15`
[`Dockerfile:L47`] with `EXPOSE 7777` [`Dockerfile:L44`]; `wsgi:app` is the WSGI entry [`wsgi.py`].
Command: `docker exec simplelogin-app sed -n '1,20p' /app/logs/webapp.log`

```
[2026-07-08 03:56:08 +0000] [525] [INFO] Starting gunicorn 20.0.4
[2026-07-08 03:56:08 +0000] [525] [INFO] Listening at: http://0.0.0.0:7777 (525)
[2026-07-08 03:56:08 +0000] [525] [INFO] Using worker: sync
[2026-07-08 03:56:08 +0000] [526] [INFO] Booting worker with pid: 526
[2026-07-08 03:56:08 +0000] [527] [INFO] Booting worker with pid: 527
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/puzgyljzwhbicgjxrrxl
Upload files to local dir
>>> init logging <<<
2026-07-08 03:56:08,654 - SL - DEBUG - 526 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/swhkcdxhugsgcdasjsew
Upload files to local dir
>>> init logging <<<
```

Readiness markers: `Listening at: http://0.0.0.0:7777` and one `Booting worker with pid:` per worker
(two workers, `-w 2`). The `>>> init logging <<<` line is `print(">>> init logging <<<")`
[`app/log.py:L67`], emitted once per worker process as the `SL` logger (`LOG = _get_logger("SL")`
[`app/log.py:L79`]) initializes; it appears in **every** SimpleLogin process below.

### Q1.2 Email handler — `Listen for port 20381` / `Start mail controller`

The `aiosmtpd` controller entry point is `python email_handler.py`.
Command: `docker exec simplelogin-app sed -n '1,9p' /app/logs/email_handler.log`

```
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/grtndrypthtkdczcjkyy
Upload files to local dir
>>> init logging <<<
2026-07-08 03:57:06,173 - SL - DEBUG - 593 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-08 03:57:06,719 - SL - INFO - 593 - "/app/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-08 03:57:06,720 - SL - DEBUG - 593 - "/app/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

Readiness markers: `Listen for port 20381` [`email_handler.py:L2403`] and
`Start mail controller 0.0.0.0 20381` [`email_handler.py:L2386`] (PID 593). This is the alias
receive/forward SMTP service Postfix relays inbound mail to.

### Q1.3 Event listener — source/sink wiring + `Starting to listen to events`

Entry point `python event_listener.py listener`.
Command: `docker exec simplelogin-app sed -n '1,12p' /app/logs/event_listener.log`

```
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/imfmngfjxqototqszhpe
Upload files to local dir
>>> init logging <<<
2026-07-08 03:58:24,968 - SL - DEBUG - 637 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-08 03:58:25,087 - SL - INFO - 637 - "/app/event_listener.py:34" - main() -  - Using PostgresEventSource
2026-07-08 03:58:25,095 - SL - INFO - 637 - "/app/event_listener.py:43" - main() -  - Starting with HttpEventSink
2026-07-08 03:58:25,095 - SL - INFO - 637 - "/app/events/event_source.py:49" - __listen() -  - Starting to listen to events
2026-07-08 04:51:52,905 - SL - DEBUG - 637 - "/app/events/event_source.py:55" - __listen() -  - Got NOTIFY: pid=4943 channel=simplelogin_sync_events payload=1
2026-07-08 04:51:52,975 - SL - WARNING - 637 - "/app/events/event_sink.py:19" - process() -  - Skipping sending event because there is no webhook configured
```

Readiness markers: `Using PostgresEventSource` [`event_listener.py:L34`],
`Starting with HttpEventSink` [`event_listener.py:L43`], and `Starting to listen to events`
[`events/event_source.py:L49`] (PID 637). The listener runs in `LISTENER` mode; in `DEAD_LETTER` mode
it would print `Using DeadLetterEventSource` and with `--dry-run` it would use `ConsoleEventSink`
[`event_listener.py:L30,L33,L40`] `(inferred — those modes were not launched; the running instance
is LISTENER + HttpEventSink as shown)`. The two trailing lines are the listener consuming a Postgres
`NOTIFY` on channel `simplelogin_sync_events` and skipping delivery because no webhook is configured
(default) — the same channel exercised in §3.3.

### Q1.4 Job runner — init banner, then a silent idle poll

Entry point `python job_runner.py`.
Command: `docker exec simplelogin-app sed -n '1,6p' /app/logs/job_runner.log`

```
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/juvcksevuerahqxswdez
Upload files to local dir
>>> init logging <<<
```

Readiness marker: the `>>> init logging <<<` banner (PID 594). The runner then enters its poll loop
and is **silent while idle** — it only logs when it takes a job (`Take job …` [`job_runner.py:L334`]).
Under `DISABLE_ONBOARDING=true` registration enqueues nothing, so at rest the log shows only the
banner. The live "it's polling" proof is in §3.5 (measured ~10.02s cadence).

### Q1.5 Cron (`yacron`) — scheduled-job spawn lines

Entry point `yacron -c /app/crontab.yml`.
Command: `docker exec simplelogin-app sed -n '1,6p' /app/logs/yacron.log`

```
INFO:yacron:Starting job SimpleLogin Notify HIBP breaches
INFO:yacron:Job SimpleLogin Notify HIBP breaches spawned
INFO:yacron:Starting job SimpleLogin send unsent emails
INFO:yacron:Job SimpleLogin send unsent emails spawned
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
```

Readiness marker: `yacron` logs `Starting job … / Job … spawned` for each scheduled command; the
subprocesses print the SL banner. The full 15-job schedule and the measured 5-minute cadence are in
§3.7.

### Q1.6 Live readiness probe — `GET /health` → `200 success`

The healthcheck route returns `"success", 200` [`server.py:L213-L215`].
Command: `docker exec simplelogin-app curl -sS -i http://localhost:7777/health`

```
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 05:37:18 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 7
Vary: Cookie
Set-Cookie: slapp=eyJfcGVybWFuZW50Ijp0cnVlfQ.ak3iDg.n1JbIQI7nNtL4N7XRVOd6JK-qPM; Expires=Wed, 15-Jul-2026 05:37:18 GMT; HttpOnly; Path=/; SameSite=Lax

success
```

`HTTP/1.1 200 OK`, `Content-Length: 7`, body `success`, served by `gunicorn/20.0.4`. (The
`Set-Cookie: slapp=` is an empty anonymous session assigned to the unauthenticated probe.)

### Q1.7 UI readiness — `GET /auth/login` renders the login card

Command:
`docker exec simplelogin-app curl -sS -o /tmp/login.html -w "HTTP %{http_code} bytes=%{size_download}\n" http://localhost:7777/auth/login`

```
HTTP 200 bytes=6918
```

The relevant rendered form elements (complete lines) from `/tmp/login.html`:

```
112:      <h1 class="card-title">Welcome back!</h1>
114:        <input id="csrf_token" name="csrf_token" type="hidden" value="ImYxMDhhYTJmOTQ2YWUyMzlhODhkZmY1NThjNDQ3ZjllMTVmNmJjMzci.ak3iJA.46gjcKSK7UGOyw5kG-Ts32Jj1EA">
117:          <input autofocus="true" class="form-control" id="email" name="email" required type="email" value="">
124:          <input class="form-control" id="password" name="password" required type="password" value="">
133:          <button type="submit" class="btn btn-primary btn-block">Log in</button>
```

The `Welcome back!` card with `email`, `password`, and hidden `csrf_token` inputs and the `Log in`
button confirms the auth UI is being served. The `auth` blueprint mounts at `url_prefix="/auth"`
[`app/auth/base.py:L4`].

### Q1.8 Per-request access log + `/health` exclusion

`after_request` [`server.py:L273`] logs one line per handled request via `LOG.d` [`server.py:L284`],
but skips `/static`, `/admin/static`, `/_debug_toolbar`, `/git`, `/favicon.ico`, and **`/health`**
[`server.py:L276-L282`]. Demonstration — three requests were issued in sequence
(`/health`, `/auth/login`, `/health`) and the new `webapp.log` lines captured:

```
=== New webapp.log lines produced by the 3-request sequence (health, login, health) ===
2026-07-08 05:37:51,273 - SL - DEBUG - 527 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.0013191699981689453
```

Only the `/auth/login` request produced an access-log line; both `/health` probes were excluded
[`server.py:L281`]. The line format is `remote_addr method path args status, takes <seconds>`.

### Q1.9 Route resolution + blueprint registration

Command (real unauthenticated HTTP to each route, `-o /dev/null -w '%{http_code}'`):

```
=== Runtime route resolution (real HTTP, unauthenticated) ===
GET /health -> HTTP 200
GET /auth/login -> HTTP 200
GET /auth/register -> HTTP 200
GET /auth/activate -> HTTP 400
GET /dashboard/ -> HTTP 302
```

`/dashboard/` 302-redirects an unauthenticated caller to login; `/auth/activate` with no `code`
returns 400. Blueprints are registered in `register_blueprints` [`server.py:L234-L246`]: `auth_bp`,
`monitor_bp`, `dashboard_bp`, `developer_bp`, `phone_bp`, `oauth_bp` (mounted twice at `/oauth` and
`/oauth2`), `onboarding_bp`, `discover_bp`, `internal_bp`, `api_bp`. The `dashboard` blueprint mounts
at `url_prefix="/dashboard"` [`app/dashboard/base.py:L6`].

### Q1.10 Boot-time seeding — SL domains + PGP keys (`init_app.py`)

`init_app.py` seeds SL domains and loads mailbox PGP public keys at boot [`init_app.py:L13-L73`]. Run
through its real entry point (idempotent):
Command: `docker exec simplelogin-app bash -lc 'cd /app && source venv/bin/activate && python init_app.py'`

```
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/ckrxvemnrmzknnuhulet
Upload files to local dir
>>> init logging <<<
2026-07-08 05:38:50,929 - SL - DEBUG - 5805 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-08 05:38:51,714 - SL - DEBUG - 5805 - "/app/init_app.py:16" - load_pgp_public_keys() -  - Load PGP key for mailbox <Mailbox 2 pgp@example.org>
2026-07-08 05:38:51,730 - SL - DEBUG - 5805 - "/app/init_app.py:36" - load_pgp_public_keys() -  - Finish load_pgp_public_keys
2026-07-08 05:38:51,732 - SL - DEBUG - 5805 - "/app/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
```

`Load PGP key for mailbox <Mailbox 2 pgp@example.org>` [`init_app.py:L16`], `Finish load_pgp_public_keys`
[`init_app.py:L36`], and `sl.local is already a SL domain` [`init_app.py:L42`] (idempotent — on the
first boot this line is `Add sl.local to SL domain` [`init_app.py:L44`], observed in the initial boot
log at 03:54:51).

---

## Q2 — New-user walkthrough: register → verify → log in → dashboard

**Direct answer.** The three steps are driven through the real `/auth` endpoints. **Register**
(`POST /auth/register`) creates the user and renders the "check your inbox" waiting page.
**Verify** (`GET /auth/activate?code=…`) flips `users.activated` `f`→`t`, deletes the code, flashes
*"Your account has been activated"* and **redirects to `/dashboard/`**. **Log in**
(`POST /auth/login`) logs the user in and redirects to `/dashboard/`. Two temporary users were used:
**User A** `blitzy-temp-usera@example.com` (id 5, verified — the happy path) and **User B**
`blitzy-temp-userb@example.com` (id 6, left unverified — the not-activated/resend/expired edges). Both
were deleted afterward (§6).

### Q2.1 Register — `POST /auth/register`

Exact command:

```
curl -sS -c $JAR -b $JAR -w "HTTP %{http_code}\n" \
  --data-urlencode "csrf_token=$CSRF" \
  --data-urlencode "email=blitzy-temp-userA@example.com" \
  --data-urlencode "password=BlitzyTempA2026!" \
  http://localhost:7777/auth/register
```

Response + the page that was rendered (grep of the response body):

```
HTTP 200
Activation
check your inbox
```

Complete register-time `webapp.log` delta:

```
2026-07-08 05:40:13,518 - SL - DEBUG - 527 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/register ImmutableMultiDict([]) 200, takes 0.0011970996856689453
2026-07-08 05:40:13,568 - SL - DEBUG - 527 - "/app/app/auth/views/register.py:85" - register() -  - create user blitzy-temp-usera@example.com
2026-07-08 05:40:13,825 - SL - INFO - 527 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 05:40:13,827 - SL - DEBUG - 527 - "/app/app/models.py:647" - create() -  - Disable onboarding emails
2026-07-08 05:40:13,846 - SL - DEBUG - 527 - "/app/app/email_utils.py:303" - send_email() -  - send email to blitzy-temp-usera@example.com, subject 'Just one more step to join SimpleLogin'
2026-07-08 05:40:13,847 - SL - DEBUG - 527 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'Just one more step to join SimpleLogin', from '"noreply@sl.local" <noreply@sl.local>' to 'blitzy-temp-usera@example.com'
2026-07-08 05:40:13,850 - SL - DEBUG - 527 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/register ImmutableMultiDict([]) 200, takes 0.3110225200653076
```

Visible confirmation: `create user blitzy-temp-usera@example.com` [`app/auth/views/register.py:L85`]
(note the email is canonicalized to lowercase), the `register_waiting_activation.html` "check your
inbox" page [`app/auth/views/register.py:L104`], and — in the same delta — the two behind-the-scenes
signals detailed in Q3: the event-dispatcher gate [`app/events/event_dispatcher.py:L62`] and
`Disable onboarding emails` [`app/models.py:L647`].

### Q2.2 Default-config reconciliation: `NOT_SEND_EMAIL` — the activation link is **NOT** printed

**Direct answer (correcting the AAP expectation).** The AAP (§0.1.1/§0.3.5) anticipated that under
`NOT_SEND_EMAIL=true` the **activation link** would be printed to the webapp log. **That is not what
the code does and not what was observed.** Under `NOT_SEND_EMAIL`, `mail_sender.send()` logs **only**
the email `subject`, `from`, and `to`, then returns `True` without rendering or logging the body/link
[`app/mail_sender.py:L130-L137`]. The activation link is therefore **never** written to the log.

Proof — grep the complete register-time log delta for the activation URL:

```
Command: grep -c "activate?code=" <register-delta>
0
```

Zero matches: the link is not present anywhere in `webapp.log`. **The AAP-expected "activation link
printed to log" signal is NOT REPRODUCED under the default configuration.**

To continue the walkthrough, the activation code was read directly from the database. **This DB
lookup is a FALLBACK / non-canonical shortcut** (a normal user would click the link in the delivered
email; here no email is delivered) — it is *not* a product signal:

```
 id |             email             | activated
----+-------------------------------+-----------
  5 | blitzy-temp-usera@example.com | f
(1 row)

 id | user_id |              code              |          expired
----+---------+--------------------------------+----------------------------
  4 |       5 | cmfiggvmsdzofhdlrqshuinnmbmewd | 2026-07-08 06:40:13.831586
(1 row)
```

User 5 is `activated=f` immediately after registration; activation code `cmfiggvmsdzofhdlrqshuinnmbmewd`
is valid for ~1 hour (`expired` one hour after creation). (This temp user was deleted in §6.)

### Q2.3 Verify email — `GET /auth/activate?code=…` (state before → during → after)

**BEFORE** — `users.activated` for id 5:

```
f
```

**DURING** — hit the real verification endpoint (complete `curl -i`):

```
Command: curl -sS -i -c $JAR -b $JAR "http://localhost:7777/auth/activate?code=cmfiggvmsdzofhdlrqshuinnmbmewd"
HTTP/1.1 302 FOUND
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 05:40:46 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 229
Location: http://localhost:7777/dashboard/
Vary: Cookie
Set-Cookie: slapp=.eJw9jstqwzAURH_FaB2DrKun_6SUEO5LONA6wZK7Cfn3CgpdDMPM4nBe5la_sG3azPr5MlMfZdrJrK2Zi_l4nMeEzI9z79OGbSLVfRz9_oNdxVzf18sgHNo2s_bj1LHuYlZDFqIHzBiWoKiLLTEqhVSKWCou01JRs6CSJYsANFIhUKiS1BYJyaMjyCyAFVgDkJCPXlwWxwMaAGLynMnmxIRJfEGq6JiLaEjD_fbU4xt33fu_2tn0-PNDIbSBYNZqy-wV45xDtPOSnC7RISebzfsXhGNXLQ.ak3i3g.vgEGC0lUmipvDwTKkBXtpNvxWZ0; Expires=Wed, 15-Jul-2026 05:40:46 GMT; HttpOnly; Path=/; SameSite=Lax

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<title>Redirecting...</title>
<h1>Redirecting...</h1>
<p>You should be redirected automatically to target URL: <a href="/dashboard/">/dashboard/</a>.  If not click the link.
```

**AFTER** — `users.activated` for id 5:

```
t
```

**Activation code deleted on success** (expect 0 rows for user 5):

```
 id | user_id | code
----+---------+------
(0 rows)
```

Complete activate `webapp.log` delta:

```
2026-07-08 05:40:46,439 - SL - DEBUG - 526 - "/app/app/email_utils.py:303" - send_email() -  - send email to simplelogin-newsletter.word910@sl.local, subject 'Welcome to SimpleLogin'
2026-07-08 05:40:46,440 - SL - DEBUG - 526 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'Welcome to SimpleLogin', from '"noreply@sl.local" <noreply@sl.local>' to 'simplelogin-newsletter.word910@sl.local'
2026-07-08 05:40:46,440 - SL - DEBUG - 526 - "/app/app/auth/views/activate.py:66" - activate() -  - redirect user to dashboard
2026-07-08 05:40:46,441 - SL - DEBUG - 526 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/activate ImmutableMultiDict([('code', 'cmfiggvmsdzofhdlrqshuinnmbmewd')]) 302, takes 0.033589839935302734
```

Confirmations: `HTTP/1.1 302 FOUND` → `Location: http://localhost:7777/dashboard/`; `activated`
flipped `f`→`t` [`app/auth/views/activate.py:L49`]; the code row was deleted
[`app/auth/views/activate.py:L53`]; a "Welcome to SimpleLogin" email is printed to the user's first
alias; and `redirect user to dashboard` [`app/auth/views/activate.py:L66`].

### Q2.4 Forwarded into the dashboard

Following the redirect with the activated session:

```
Command: curl -sS -b $JAR -o /tmp/dashA.html -w "HTTP %{http_code} bytes=%{size_download}\n" http://localhost:7777/dashboard/
HTTP 200 bytes=37536
```

The activation flash and the first alias are present in the rendered dashboard (complete greps):

```
Your account has been activated
simplelogin-newsletter.word910@sl.local
```

`HTTP 200`, the flash *"Your account has been activated"* [`app/auth/views/activate.py:L56`], and the
first alias listed on the dashboard index [`app/dashboard/views/index.py`]. The new user has been
forwarded into the dashboard exactly as expected.

### Q2.5 Log in — `POST /auth/login`

Exact command:

```
curl -sS -i -c $JAR -b $JAR \
  --data-urlencode "csrf_token=$CSRF" \
  --data-urlencode "email=blitzy-temp-usera@example.com" \
  --data-urlencode "password=BlitzyTempA2026!" \
  http://localhost:7777/auth/login
```

Response status + Location, and the complete login `webapp.log` delta:

```
HTTP/1.1 302 FOUND
Location: http://localhost:7777/dashboard/
```
```
2026-07-08 05:41:12,115 - SL - DEBUG - 527 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.0012717247009277344
2026-07-08 05:41:12,373 - SL - DEBUG - 527 - "/app/app/auth/views/login_utils.py:35" - after_login() -  - log user <User 5 blitzy-temp-userA@example.com blitzy-temp-usera@example.com> in
2026-07-08 05:41:12,374 - SL - DEBUG - 527 - "/app/app/auth/views/login_utils.py:44" - after_login() -  - redirect user to dashboard
2026-07-08 05:41:12,374 - SL - DEBUG - 527 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 302, takes 0.2388136386871338
```

Confirmations: `log user <User 5 …> in` [`app/auth/views/login_utils.py:L35`] and `redirect user to
dashboard` [`app/auth/views/login_utils.py:L44`] → `302 … Location: /dashboard/`. (`after_login`
also branches to FIDO/OTP when enabled [`app/auth/views/login_utils.py:L20,L28`] `(inferred — not
configured for this user; the direct dashboard redirect at L44 was taken, as shown)`.)

### Q2.6 Edge & error conditions (each exercised through the real endpoint)

**(a) Invalid activation code → 400.**
```
Command: curl -sS -o /tmp/inv.html -w "HTTP %{http_code}\n" "http://localhost:7777/auth/activate?code=blitzy-bogus-does-not-exist"
HTTP 400
Activation code cannot be found
```
Flash *"Activation code cannot be found"* [`app/auth/views/activate.py:L33`].

**(b) Wrong password → 200 + flash.**
```
Command: curl -sS -c $JAR -b $JAR -o /tmp/wp.html -w "HTTP %{http_code}\n" \
  --data-urlencode "csrf_token=$CSRF" --data-urlencode "email=blitzy-temp-usera@example.com" \
  --data-urlencode "password=BlitzyWRONGpassword999" http://localhost:7777/auth/login
HTTP 200
Email or password incorrect
```
```
2026-07-08 05:41:30,726 - SL - DEBUG - 527 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 200, takes 0.2402806282043457
```
Flash *"Email or password incorrect"* [`app/auth/views/login.py:L49-L50`] (200, no redirect).

**(c) Not-activated user login → 200 + check-inbox prompt.** User B (id 6), registered and left
unverified:
```
register userB -> HTTP 200

 id |             email             | activated
----+-------------------------------+-----------
  6 | blitzy-temp-userb@example.com | f
(1 row)

 id | user_id |              code              |          expired
----+---------+--------------------------------+----------------------------
  5 |       6 | inszibyotzcivlkqwucbuhjjmgpjbi | 2026-07-08 06:41:45.123282
(1 row)
```
```
=== EDGE: Not-activated user login → HTTP 200 + check-inbox prompt ===
HTTP 200
Please check your inbox for the activation email. You can also have this email re-sent
```
Flash prompting re-send [`app/auth/views/login.py:L63-L69`].

**(d) Resend activation → 200.**
```
=== EDGE: Resend activation ===
HTTP 200
```
```
2026-07-08 05:42:01,669 - SL - DEBUG - 526 - "/app/app/auth/views/resend_activation.py:36" - resend_activation() -  - user <User 6 blitzy-temp-userB@example.com blitzy-temp-userb@example.com> is not activated
2026-07-08 05:42:01,686 - SL - DEBUG - 526 - "/app/app/email_utils.py:303" - send_email() -  - send email to blitzy-temp-userb@example.com, subject 'Just one more step to join SimpleLogin'
2026-07-08 05:42:01,687 - SL - DEBUG - 526 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'Just one more step to join SimpleLogin', from '"noreply@sl.local" <noreply@sl.local>' to 'blitzy-temp-userb@example.com'
2026-07-08 05:42:01,688 - SL - DEBUG - 526 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/resend_activation ImmutableMultiDict([]) 200, takes 0.022855520248413086
```
`user <User 6 …> is not activated` [`app/auth/views/resend_activation.py:L36`]; a fresh code is
generated and (printed) re-sent.

**(e) Expired activation code → 400 (state before → during → after).** The resend created a fresh
code (id 6); its `expired` timestamp was moved into the past to exercise the `is_expired()` branch:
```
--- Current activation code for user B (resend created a fresh one) ---
 id | user_id |              code              |          expired
----+---------+--------------------------------+----------------------------
  6 |       6 | veytllpwljjyfrwwxtupzfuoybfynd | 2026-07-08 06:42:01.670709
(1 row)

--- BEFORE: expired timestamp (in the future = still valid) ---
2026-07-08 06:42:01.670709
--- DURING (state change): set the temp code's expiry into the past ---
Command: UPDATE activation_code SET expired='2020-01-01 00:00:00' WHERE user_id=6;
UPDATE 1
--- AFTER the update: expired timestamp now in the past ---
2020-01-01 00:00:00
--- Hit the REAL endpoint with the (now-expired) code ---
Command: curl -sS -o /tmp/exp.html -w "HTTP %{http_code}\n" "http://localhost:7777/auth/activate?code=veytllpwljjyfrwwxtupzfuoybfynd"
HTTP 400
Activation code was expired
```
Flash *"Activation code was expired"* [`app/auth/views/activate.py:L42-L43`].

**(f) Disabled account → 200 + flash (before → during → after, with reset).** Exercised on User A
(id 5), reversibly:
```
--- BEFORE: users.disabled ---
f
--- STATE CHANGE: disable the temp account ---
Command: UPDATE users SET disabled=true WHERE id=5;
UPDATE 1
--- DURING (disabled=true): attempt real login ---
HTTP 200
Your account is disabled. Please contact SimpleLogin team to re-enable your account.
--- RESET: re-enable ---
Command: UPDATE users SET disabled=false WHERE id=5;
UPDATE 1
--- AFTER reset: users.disabled ---
f
```
Flash *"Your account is disabled…"* [`app/auth/views/login.py:L51-L56`].

**(g) Scheduled-for-deletion → 200 + flash (before → during → after, with reset).** On User A:
```
--- BEFORE: users.delete_on ---

(empty = NULL)
--- STATE CHANGE: schedule deletion ---
Command: UPDATE users SET delete_on='2030-01-01 00:00:00' WHERE id=5;
UPDATE 1
--- DURING (delete_on set): attempt real login ---
HTTP 200
Your account is scheduled to be deleted on 2030-01-01T00:00:00+00:00
--- RESET: unschedule ---
Command: UPDATE users SET delete_on=NULL WHERE id=5;
UPDATE 1
--- AFTER reset: users.delete_on (empty = NULL) ---

(empty = NULL)
```
Flash *"Your account is scheduled to be deleted on …"* [`app/auth/views/login.py:L57-L62`].

**(h) `DISABLE_REGISTRATION` → 302 to login (default is OFF).** Because the default `.env` must not
be modified, a **throwaway** instance was launched on `:7778` with `DISABLE_REGISTRATION=1`
(`"DISABLE_REGISTRATION" in os.environ` [`app/config.py:L138`]); the committed `.env` was untouched
and the instance was stopped afterward:
```
=== Users count BEFORE (baseline) === 
4
=== Launch THROWAWAY instance on :7778 with DISABLE_REGISTRATION=1 ===
Command: DISABLE_REGISTRATION=1 gunicorn wsgi:app -b 0.0.0.0:7778 -w 1 --timeout 15 (backgrounded)
throwaway master PID=6066
=== GET /auth/register on :7778 → 302 to /auth/login ===
HTTP/1.1 302 FOUND
Location: http://localhost:7778/auth/login
=== POST /auth/register on :7778 → also blocked ===
HTTP/1.1 302 FOUND
Location: http://localhost:7778/auth/login
=== Follow redirect WITH cookie jar so the session flash persists ===
Command: curl -sS -L -c jar -b jar -o closed.html -w "final HTTP %{http_code}\n" http://localhost:7778/auth/register
final HTTP 200
--- closed-registration flash on the login page ---
Registration is closed
=== Blocked user was NOT created ===
 should_be_zero
----------------
              0
(1 row)
=== Stop the throwaway instance ===
throwaway :7778 stopped (connection refused)
```
Both `GET` and `POST /auth/register` 302-redirect to `/auth/login` with the flash *"Registration is
closed"* [`app/auth/views/register.py:L38-L40`]; the blocked registration created **0** users
(count `4` before and after), and the throwaway `:7778` instance was stopped (connection refused).

---

## Q3 — Behind the scenes: background jobs & internal services

**Direct answer.** Yes. The new-user flow sets in motion several internal services that are
observably active and communicating: (1) `User.create` provisions a **verified default mailbox** and
a **first alias** and — under `DISABLE_ONBOARDING=true` — deliberately enqueues **no** onboarding
jobs; (2) real product paths reach the **event dispatcher**, which under the default config (no
`EVENT_WEBHOOK`) short-circuits at a gate (the full persist→`NOTIFY`→consume mechanism is shown via a
labeled diagnostic); (3) the **email handler** on `:20381` forwards a real message alias→mailbox;
(4) the **job runner** drains work on a ~10.02s poll, demonstrated through the real account-deletion
entry point; and (5) **cron** runs `send_undelivered_mails` every ~300s. Each is evidenced below.

### Q3.1 Identity provisioning by `User.create` (verified mailbox + first alias + onboarding skip)

Inspecting User A (id 5) right after registration:

```
--- User row: default_mailbox_id + newsletter_alias_id [app/models.py:L613,L643] ---
-[ RECORD 1 ]-------+------------------------------
id                  | 5
email               | blitzy-temp-usera@example.com
activated           | t
default_mailbox_id  | 7
newsletter_alias_id | 14

--- Default mailbox (verified=t) [app/models.py:L611] ---
-[ RECORD 1 ]---------------------------
id       | 7
user_id  | 5
email    | blitzy-temp-usera@example.com
verified | t

--- First alias (prefix simplelogin-newsletter) [app/models.py:L636] ---
-[ RECORD 1 ]--------------------------------------------------------------------------------------------------------------------
id      | 14
user_id | 5
email   | simplelogin-newsletter.word910@sl.local
note    | This is your first alias. It's used to receive SimpleLogin communications like new features announcements, newsletters.

--- Onboarding jobs for user 5 (DISABLE_ONBOARDING=true → expect 0) [app/models.py:L646-648] ---
 onboarding_jobs
-----------------
               0
(1 row)
```

`User.create` set `default_mailbox_id=7` (a mailbox with `verified=t` [`app/models.py:L611`]) and
`newsletter_alias_id=14` (first alias, prefix `simplelogin-newsletter` [`app/models.py:L636`]). Under
`DISABLE_ONBOARDING=true` it logged `Disable onboarding emails` (seen in the Q2.1 delta) and enqueued
**0** onboarding jobs [`app/models.py:L646-L648`]. `(inferred)` In production
(`DISABLE_ONBOARDING` off) the same method enqueues **three** onboarding jobs — `onboarding-1`
[`app/models.py:L652`], `onboarding-2` [`app/models.py:L657`], and `onboarding-4` [`app/models.py:L662`]
(enclosing span [`app/models.py:L650-L665`]); note `onboarding-3` is *defined* [`app/config.py:L303`]
but is **not** enqueued by `User.create` — not reproduced here by design.

### Q3.2 Event pipeline — REAL entry point (canonical): dispatcher gate under default config

**Canonical, observed.** A logged-in user creating an alias is a real product path that reaches
`EventDispatcher.send_event`. Exercised via `POST /dashboard/` with `form-name=create-random-email`
(the dashboard "random alias" action):

```
Command: curl -sS -c $JAR -b $JAR -i \
  --data-urlencode "csrf_token=$CSRF" --data-urlencode "form-name=create-random-email" \
  http://localhost:7777/dashboard/
HTTP/1.1 302 FOUND
Location: http://localhost:7777/dashboard/?highlight_alias_id=16&query=&sort=&filter=
```
```
2026-07-08 05:45:52,509 - SL - DEBUG - 527 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email word_list345@sl.local
2026-07-08 05:45:52,516 - SL - INFO - 527 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 05:45:52,519 - SL - DEBUG - 527 - "/app/app/dashboard/views/index.py:110" - index() -  - create new random alias <Alias 16 word_list345@sl.local> for user <User 5 blitzy-temp-userA@example.com blitzy-temp-usera@example.com>
```

Alias creation (`create new random alias <Alias 16 …>` [`app/dashboard/views/index.py:L110`]) reaches
the dispatcher, which **short-circuits** at `Not sending events because webhook is not configured and
allowed to be empty` [`app/events/event_dispatcher.py:L62`]. This is the **canonical** runtime
behavior under the default config (`EVENT_WEBHOOK` unset [`app/config.py:L612`]): the gate
[`app/events/event_dispatcher.py:L47-L65`] returns before persisting. The identical gate line also
appears in the Q2.1 registration delta — i.e. the very first alias created during registration hits
the same real path.

### Q3.3 Event pipeline — MECHANISM (NON-CANONICAL DIAGNOSTIC — labeled non-canonical per AAP §0.7 real-entry-point rule)

Because the canonical path short-circuits before persisting when `EVENT_WEBHOOK` is unset, the full
persist→`NOTIFY`→consume mechanism cannot be reached through a product entry point under the default
config. The mechanism below was therefore exercised with a **standalone diagnostic script that calls
`PostgresDispatcher().send(...)` directly** — this is **explicitly NOT a product entry point** and is
labeled non-canonical. It demonstrates that the `event_listener` service (PID 637, already running)
is live and consuming.

Diagnostic script (`/tmp/blitzy_evtdiag.py`, since removed — §6) core:
`PostgresDispatcher().send(b"blitzy-q3-diagnostic-event")` then `Session.commit()`
[`app/events/event_dispatcher.py:L23-L26`, channel `simplelogin_sync_events` at L14]. Complete output:

```
=== Run diagnostic (PYTHONPATH=/app) — COMPLETE output ===
DIAGNOSTIC: channel=simplelogin_sync_events
DIAGNOSTIC: PostgresDispatcher.send persisted SyncEvent id=2, content=b'blitzy-q3-diagnostic-event', retry_count=0

=== Running event_listener.py (PID 637) COMPLETE delta — it CONSUMED the NOTIFY ===
2026-07-08 05:46:36,034 - SL - DEBUG - 637 - "/app/events/event_source.py:55" - __listen() -  - Got NOTIFY: pid=6211 channel=simplelogin_sync_events payload=2
2026-07-08 05:46:36,057 - SL - WARNING - 637 - "/app/events/event_sink.py:19" - process() -  - Skipping sending event because there is no webhook configured

=== The persisted SyncEvent row (diagnostic) ===
 id |                        content                         | retry_count
----+--------------------------------------------------------+-------------
  2 | \x626c69747a792d71332d646961676e6f737469632d6576656e74 |           1
(1 row)
```

The dispatcher persisted a `SyncEvent` row and issued `NOTIFY simplelogin_sync_events`; the
independently-running listener received `Got NOTIFY: … channel=simplelogin_sync_events payload=2`
[`events/event_source.py:L55`] and (no webhook) logged `Skipping sending event because there is no
webhook configured` [`events/event_sink.py:L19`]. The `retry_count` advanced to 1 because the
`Runner.__on_event` handler [`events/runner.py:L20`] (of class `Runner` [`events/runner.py:L11`],
which pumps source → sink) re-queued the undeliverable event via `event.retry_count =
event.retry_count + 1` then `Session.commit()` [`events/runner.py:L42-L43`]. This proves the
LISTEN/NOTIFY channel and the listener are active and communicating end-to-end. `(inferred)` For
manual replay/inspection of a persisted event outside the live loop, the operator utility
`events/event_debugger.py` provides `debug_event` [`events/event_debugger.py:L6`] and `run_event`
[`events/event_debugger.py:L31`] (a debug/replay tool, not part of the live flow). (The `SyncEvent
id=2` row was deleted in §6.)

### Q3.4 Email forwarding on `:20381` (external sender → alias → owning mailbox)

A real message was injected into the email handler with `swaks`, addressed to User A's first alias.
Exact command + complete SMTP transcript:

```
swaks --to simplelogin-newsletter.word910@sl.local --from blitzy-external-sender@example.com \
      --server 127.0.0.1:20381 --header "Subject: Blitzy Q3 forward test" --body "Blitzy Q3 forward test body"

=== Trying 127.0.0.1:20381...
=== Connected to 127.0.0.1.
<-  220 164d4ad66104 Python SMTP 1.4.2
 -> EHLO 164d4ad66104
<-  250-164d4ad66104
<-  250-SIZE 33554432
<-  250-8BITMIME
<-  250-SMTPUTF8
<-  250 HELP
 -> MAIL FROM:<blitzy-external-sender@example.com>
<-  250 OK
 -> RCPT TO:<simplelogin-newsletter.word910@sl.local>
<-  250 OK
 -> DATA
<-  354 End data with <CR><LF>.<CR><LF>
 -> Date: Wed, 08 Jul 2026 05:47:01 +0000
 -> To: simplelogin-newsletter.word910@sl.local
 -> From: blitzy-external-sender@example.com
 -> Subject: Blitzy Q3 forward test
 -> Message-Id: <20260708054701.006229@164d4ad66104>
 -> X-Mailer: swaks v20201014.0 jetmore.org/john/code/swaks/
 -> 
 -> Blitzy Q3 forward test body
 -> 
 -> 
 -> .
<-  250 Message accepted for delivery
 -> QUIT
<-  221 Bye
=== Connection closed with remote host.
```

Complete `email_handler.log` forward-path delta (PID 593, message-id `9a49c31a-4d0a-4896-a60a-38a8f3726bca`):

```
2026-07-08 05:47:01,979 - SL - DEBUG - 593 - "/app/app/log.py:24" - set_message_id() - 1df32b94-f7ad-4abb-a367-220088065f16 - set message_id 9a49c31a-4d0a-4896-a60a-38a8f3726bca
2026-07-08 05:47:01,979 - SL - DEBUG - 593 - "/app/email_handler.py:2342" - _handle() - 9a49c31a-4d0a-4896-a60a-38a8f3726bca - ====>=====>====>====>====>====>====>====>
2026-07-08 05:47:01,979 - SL - INFO - 593 - "/app/email_handler.py:2343" - _handle() - 9a49c31a-4d0a-4896-a60a-38a8f3726bca - New message, mail from blitzy-external-sender@example.com, rctp tos ['simplelogin-newsletter.word910@sl.local'] 
2026-07-08 05:47:01,980 - SL - INFO - 593 - "/app/email_handler.py:1956" - handle() - 9a49c31a-4d0a-4896-a60a-38a8f3726bca - Set CONTENT_TRANSFER_ENCODING
2026-07-08 05:47:01,980 - SL - DEBUG - 593 - "/app/email_handler.py:1963" - handle() - 9a49c31a-4d0a-4896-a60a-38a8f3726bca - Cannot parse Postfix queue ID from None None
2026-07-08 05:47:01,982 - SL - DEBUG - 593 - "/app/email_handler.py:1980" - handle() - 9a49c31a-4d0a-4896-a60a-38a8f3726bca - ==>> Handle mail_from:blitzy-external-sender@example.com, rcpt_tos:['simplelogin-newsletter.word910@sl.local'], header_from:blitzy-external-sender@example.com, header_to:simplelogin-newsletter.word910@sl.local, cc:None, reply-to:None, message_id:<20260708054701.006229@164d4ad66104>, client_ip:None, headers:[('Date', 'Wed, 08 Jul 2026 05:47:01 +0000'), ('To', 'simplelogin-newsletter.word910@sl.local'), ('From', 'blitzy-external-sender@example.com'), ('Subject', 'Blitzy Q3 forward test'), ('Message-Id', '<20260708054701.006229@164d4ad66104>'), ('X-Mailer', 'swaks v20201014.0 jetmore.org/john/code/swaks/'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-08 05:47:01,985 - SL - DEBUG - 593 - "/app/email_handler.py:2202" - handle() - 9a49c31a-4d0a-4896-a60a-38a8f3726bca - Forward phase blitzy-external-sender@example.com(blitzy-external-sender@example.com) -> simplelogin-newsletter.word910@sl.local
2026-07-08 05:47:01,993 - SL - DEBUG - 593 - "/app/email_handler.py:580" - handle_forward() - 9a49c31a-4d0a-4896-a60a-38a8f3726bca - Create or get contact for from_header:blitzy-external-sender@example.com
2026-07-08 05:47:02,011 - SL - DEBUG - 593 - "/app/app/contact_utils.py:110" - create_contact() - 9a49c31a-4d0a-4896-a60a-38a8f3726bca - Created contact <Contact 4 blitzy-external-sender@example.com 14> for alias <Alias 14 simplelogin-newsletter.word910@sl.local> with email blitzy-external-sender@example.com invalid_email=False
2026-07-08 05:47:02,011 - SL - INFO - 593 - "/app/app/handler/dmarc.py:33" - apply_dmarc_policy_for_forward_phase() - 9a49c31a-4d0a-4896-a60a-38a8f3726bca - DMARC check disabled
2026-07-08 05:47:02,017 - SL - DEBUG - 593 - "/app/email_handler.py:688" - forward_email_to_mailbox() - 9a49c31a-4d0a-4896-a60a-38a8f3726bca - Forward <Contact 4 blitzy-external-sender@example.com 14> -> <Alias 14 simplelogin-newsletter.word910@sl.local> -> <Mailbox 7 blitzy-temp-usera@example.com>
2026-07-08 05:47:02,019 - SL - DEBUG - 593 - "/app/email_handler.py:740" - forward_email_to_mailbox() - 9a49c31a-4d0a-4896-a60a-38a8f3726bca - Create <EmailLog 4> for <Contact 4 blitzy-external-sender@example.com 14>, <User 5 blitzy-temp-userA@example.com blitzy-temp-usera@example.com>, <Mailbox 7 blitzy-temp-usera@example.com>
2026-07-08 05:47:02,024 - SL - DEBUG - 593 - "/app/email_handler.py:867" - forward_email_to_mailbox() - 9a49c31a-4d0a-4896-a60a-38a8f3726bca - From header, new:"blitzy-external-sender at example.com" <blitzy-external-sender_at_example_com_urkmrh@sl.local>, old:blitzy-external-sender@example.com
2026-07-08 05:47:02,024 - SL - DEBUG - 593 - "/app/email_handler.py:316" - replace_header_when_forward() - 9a49c31a-4d0a-4896-a60a-38a8f3726bca - Delete Cc header, old value None
2026-07-08 05:47:02,024 - SL - DEBUG - 593 - "/app/email_handler.py:313" - replace_header_when_forward() - 9a49c31a-4d0a-4896-a60a-38a8f3726bca - Replace To header, old: simplelogin-newsletter.word910@sl.local, new: simplelogin-newsletter.word910@sl.local
2026-07-08 05:47:02,025 - SL - INFO - 593 - "/app/app/handler/unsubscribe_generator.py:36" - _generate_header_with_original_behaviour() - 9a49c31a-4d0a-4896-a60a-38a8f3726bca - Email has no unsubscribe header
2026-07-08 05:47:02,025 - SL - DEBUG - 593 - "/app/email_handler.py:893" - forward_email_to_mailbox() - 9a49c31a-4d0a-4896-a60a-38a8f3726bca - Forward mail from blitzy-external-sender@example.com to blitzy-temp-usera@example.com, mail_options:[], rcpt_options:[] 
2026-07-08 05:47:02,025 - SL - DEBUG - 593 - "/app/app/mail_sender.py:131" - send() - 9a49c31a-4d0a-4896-a60a-38a8f3726bca - send email with subject 'Blitzy Q3 forward test', from '"blitzy-external-sender at example.com" <blitzy-external-sender_at_example_com_urkmrh@sl.local>' to 'simplelogin-newsletter.word910@sl.local'
2026-07-08 05:47:02,025 - SL - INFO - 593 - "/app/email_handler.py:2367" - _handle() - 9a49c31a-4d0a-4896-a60a-38a8f3726bca - Finish mail_from blitzy-external-sender@example.com, rcpt_tos ['simplelogin-newsletter.word910@sl.local'], takes 0.04571652412414551 seconds with return code '250 Message accepted for delivery'<<===
```

Confirmations of the alias receive→forward path: `New message …` [`email_handler.py:L2343`] →
`Forward phase …` [`email_handler.py:L2202`] → `handle_forward()` creates a contact
[`email_handler.py:L580`; `app/contact_utils.py:L110`] → `Forward <Contact 4> -> <Alias 14 …> ->
<Mailbox 7 …>` [`email_handler.py:L688`] → `Create <EmailLog 4>` [`email_handler.py:L740`] → the
**reverse-alias From-rewrite** `"…at example.com" <blitzy-external-sender_at_example_com_urkmrh@sl.local>`
[`email_handler.py:L867`] → `Finish mail_from … '250 Message accepted for delivery'`
[`email_handler.py:L2367`]. The counterpart alias **send/reply** path is `handle_reply`
[`email_handler.py:L966`] `(inferred — not exercised; reply requires a mailbox-originated message to a
reverse-alias)`. (The `Contact 4`/`EmailLog 4` created here were removed with User A in §6.)

### Q3.5 Job runner — poll cadence ~10.02s (Job.create is a labeled observation-setup)

`job_runner.py` polls every 10s (`time.sleep(10)` [`job_runner.py:L347`]), taking ready jobs
(`Take job …` [`job_runner.py:L334`]) and dispatching them (`process_job` [`job_runner.py:L188-L304`]).
To measure the cadence, `send-alias-creation-events` jobs were enqueued **one at a time** via
`Job.create` — **this enqueue is a labeled observation-setup**, but the measured signal (the runner's
`Take job` timestamps and its dispatch) is **canonical runner behavior**. Each new job was enqueued
only after the previous was taken, so each lands in a fresh sleep window.

**Run 1** (7 jobs; driver wall-clock 70 s) — complete interval computation + `JobState` proof:

```
Job#10: 2026-07-08 05:48:04,914  (first)
Job#11: 2026-07-08 05:48:14,936  interval=10.022s
Job#12: 2026-07-08 05:48:24,958  interval=10.022s
Job#13: 2026-07-08 05:48:34,980  interval=10.022s
Job#14: 2026-07-08 05:48:45,001  interval=10.021s
Job#15: 2026-07-08 05:48:55,021  interval=10.020s
Job#16: 2026-07-08 05:49:05,042  interval=10.021s

 id |            name            | state | attempts
----+----------------------------+-------+----------
 10 | send-alias-creation-events |     2 |        1
 11 | send-alias-creation-events |     2 |        1
 12 | send-alias-creation-events |     2 |        1
 13 | send-alias-creation-events |     2 |        1
 14 | send-alias-creation-events |     2 |        1
 15 | send-alias-creation-events |     2 |        1
 16 | send-alias-creation-events |     2 |        1
(7 rows)
```

**Run 2** (independent, 4 jobs; wall-clock 38 s) — complete `Take job` lines + intervals:

```
2026-07-08 05:50:45,175 - SL - DEBUG - 594 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 17 send-alias-creation-events {'user_id': 5}>
2026-07-08 05:50:55,196 - SL - DEBUG - 594 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 18 send-alias-creation-events {'user_id': 5}>
2026-07-08 05:51:05,217 - SL - DEBUG - 594 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 19 send-alias-creation-events {'user_id': 5}>
2026-07-08 05:51:15,238 - SL - DEBUG - 594 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 20 send-alias-creation-events {'user_id': 5}>

take[0]: 2026-07-08 05:50:45,175
take[1]: 2026-07-08 05:50:55,196  interval=10.021s
take[2]: 2026-07-08 05:51:05,217  interval=10.021s
take[3]: 2026-07-08 05:51:15,238  interval=10.021s
```

**Result:** the poll interval is **~10.02s** and **stable across two independent runs** (Run 1: six
intervals 10.020–10.022s; Run 2: three intervals 10.021s), matching `time.sleep(10)`
[`job_runner.py:L347`] plus a few ms of dispatch/query time. Every job reached `state=2` (**done**)
with `attempts=1`; the `JobState` enum is `ready=0, taken=1, done=2, error=3` [`app/models.py:L253-L257`].
When taken, each job dispatched to `send_alias_creation_events_for_user`
[`app/jobs/event_jobs.py:L17,L44`], which itself hit the event gate
[`app/events/event_dispatcher.py:L62`]. That function builds the protobuf sync-event payloads from
the generated schema module `app/events/generated/event_pb2.py` — specifically the message types
`AliasCreated` and `AliasCreatedList` (imported at [`app/jobs/event_jobs.py:L4`], constructed at
[`app/jobs/event_jobs.py:L24`] and [`app/jobs/event_jobs.py:L36,L47`]); those wire-format messages are
the schema all sync events are serialized from. (Job rows 10–20 were deleted in §6.)

### Q3.6 Job runner — REAL entry point (canonical): account deletion enqueues & drains `delete-account`

**Canonical, observed.** The real product path that enqueues a background job for a logged-in user is
account deletion: `POST /dashboard/delete_account` calls `Job.create(JOB_DELETE_ACCOUNT, …)`
[`app/dashboard/views/delete_account.py:L42`] (behind `@login_required` + `@sudo_required`; the sudo
window is 120s, set at login [`app/dashboard/views/enter_sudo.py:L17,L70-L80`;
`app/auth/views/login_utils.py:L37`]). A fresh login (to set `sudo_time`) then the delete POST:

```
=== STEP 1: fresh login User A (sets sudo_time) ===
HTTP/1.1 302 FOUND
Location: http://localhost:7777/dashboard/
=== STEP 3: POST /dashboard/delete_account (REAL ENTRY POINT) ===
Command: curl -sS -b $J -c $J -i -X POST http://localhost:7777/dashboard/delete_account \
  --data-urlencode "form-name=delete-account" --data-urlencode "csrf_token=$CSRF"
HTTP/1.1 302 FOUND
Location: http://localhost:7777/dashboard/setting
```
```
2026-07-08 05:52:40,623 - SL - WARNING - 527 - "/app/app/dashboard/views/delete_account.py:36" - delete_account() -  - schedule delete account job for <User 5 blitzy-temp-userA@example.com blitzy-temp-usera@example.com>
2026-07-08 05:52:40,627 - SL - DEBUG - 527 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /dashboard/delete_account ImmutableMultiDict([]) 302, takes 0.006890535354614258
```

Job enqueued (state=0 ready), then the runner drains it (~within one poll):

```
 id |      name      | state |    payload
----+----------------+-------+----------------
 21 | delete-account |     0 | {"user_id": 5}
(1 row)
```
```
2026-07-08 05:52:45,338 - SL - DEBUG - 594 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 21 delete-account {'user_id': 5}>
2026-07-08 05:52:45,343 - SL - WARNING - 594 - "/app/job_runner.py:235" - process_job() -  - Delete user <User 5 blitzy-temp-userA@example.com blitzy-temp-usera@example.com>
```

The dispatch branch `JOB_DELETE_ACCOUNT` [`job_runner.py:L226`] logs `Delete user <User 5 …>`
[`job_runner.py:L235`], prints the account-deleted email (`NOT_SEND_EMAIL`), then calls
`User.delete()` + `Session.commit()`. DB **after** — user + all dependents cascade-deleted, job done:

```
--- user 5 (expect 0 rows) ---
 id | email
----+-------
(0 rows)

--- aliases 14,16 (expect 0 rows) ---
 id | email
----+-------
(0 rows)

--- mailbox 7 (expect 0 rows) ---
 id
----
(0 rows)

--- contact 4 (expect 0 rows) ---
 id
----
(0 rows)

--- email_log 4 (expect 0 rows) ---
 id
----
(0 rows)

--- job 21 final state (expect state=2 done) ---
 id |      name      | state
----+----------------+-------
 21 | delete-account |     2
(1 row)
```
```
2026-07-08 05:52:45,355 - SL - DEBUG - 594 - "/app/app/email_utils.py:303" - send_email() -  - send email to blitzy-temp-usera@example.com, subject 'Your SimpleLogin account has been deleted'
2026-07-08 05:52:45,356 - SL - DEBUG - 594 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'Your SimpleLogin account has been deleted', from '"noreply@sl.local" <noreply@sl.local>' to 'blitzy-temp-usera@example.com'
2026-07-08 05:52:45,363 - SL - INFO - 594 - "/app/app/alias_utils.py:346" - delete_alias() -  - User <User 5 blitzy-temp-userA@example.com blitzy-temp-usera@example.com> has deleted alias <Alias 14 simplelogin-newsletter.word910@sl.local>
2026-07-08 05:52:45,377 - SL - INFO - 594 - "/app/app/alias_utils.py:346" - delete_alias() -  - User <User 5 blitzy-temp-userA@example.com blitzy-temp-usera@example.com> has deleted alias <Alias 16 word_list345@sl.local>
```

End-to-end proof that the webapp and job runner communicate through the DB-backed `Job` queue: the
real dashboard action enqueued `delete-account` (state 0), the runner took it (`Take job <Job 21
delete-account>`), deleted User 5 and cascade (aliases 14/16, mailbox 7, contact 4, email_log 4 → all
0 rows), and marked the job `state=2`. This real entry point also served as the deletion of temp User
A (§6).

`(inferred)` `process_job` dispatches other job types beyond those exercised above — for example the
GDPR user-data export job: when `job.name == config.JOB_SEND_USER_REPORT` (`"send-user-report"`,
[`app/config.py:L309`]) the runner calls `ExportUserDataJob.create_from_job(job)` then `.run()`
[`job_runner.py:L285-L288`], where `class ExportUserDataJob` [`app/jobs/export_user_data_job.py:L41`]
and its `run` method [`app/jobs/export_user_data_job.py:L131`] build a ZIP of the user's data and email
it. This job is **not** triggered by the register/verify/login flow, so it was not exercised here.

### Q3.7 Cron — `send_undelivered_mails` every ~300s (`*/5` schedule)

`crontab.yml` defines **15** `yacron` jobs. `send_undelivered_mails` runs on `*/5 * * * *`
[`crontab.yml:L77-L82`], executing `python /code/cron.py -j send_undelivered_mails`, which logs
`Sending undelivered emails` [`cron.py:L1314`]. All executions over the container's uptime and their
intervals:

```
run[0]: 2026-07-08 04:15:01,746  (first)
run[1]: 2026-07-08 04:20:01,679  interval=299.9s
run[2]: 2026-07-08 04:25:01,938  interval=300.3s
run[3]: 2026-07-08 04:30:02,056  interval=300.1s
run[4]: 2026-07-08 04:35:02,238  interval=300.2s
run[5]: 2026-07-08 04:40:02,376  interval=300.1s
run[6]: 2026-07-08 04:45:01,508  interval=299.1s
run[7]: 2026-07-08 04:50:01,555  interval=300.0s
run[8]: 2026-07-08 04:55:01,722  interval=300.2s
run[9]: 2026-07-08 05:00:03,985  interval=302.3s
run[10]: 2026-07-08 05:05:02,001  interval=298.0s
run[11]: 2026-07-08 05:10:02,150  interval=300.1s
run[12]: 2026-07-08 05:15:02,381  interval=300.2s
run[13]: 2026-07-08 05:20:01,486  interval=299.1s
run[14]: 2026-07-08 05:25:01,632  interval=300.1s
run[15]: 2026-07-08 05:30:01,743  interval=300.1s
run[16]: 2026-07-08 05:35:01,971  interval=300.2s
run[17]: 2026-07-08 05:40:02,028  interval=300.1s
run[18]: 2026-07-08 05:45:02,168  interval=300.1s
run[19]: 2026-07-08 05:50:02,344  interval=300.2s

intervals: min=298.0s max=302.3s  (nominal 300s per */5 schedule)
```

**Result:** 20 executions over ~1h35m, 19 intervals all **~300s** (min 298.0s, max 302.3s) —
stable, matching the 5-minute `*/5` schedule [`crontab.yml:L80`]. The 15 scheduled jobs in
`crontab.yml` are: `stats` (daily), `delete_old_monitoring`, `check_custom_domain`, `check_hibp`,
`notify_hibp`, `delete_logs`, `delete_old_data`, `poll_apple_subscription`, `notify_trial_end`,
`notify_manual_subscription_end`, `notify_premium_end`, `delete_scheduled_users`,
**`send_undelivered_mails` (`*/5`)**, `clear_alias_audit_log` (hourly), and `clear_user_audit_log`
(hourly) — each `python /code/cron.py -j <name>` [`cron.py`].

### Q3.8 Monitoring / observability `(inferred — not exercised at runtime)`

`monitoring.py` runs a background task watching the Postfix queue [`monitoring.py:L1-L40`] and
`monitor/newrelic.py` / `monitor/metric_exporter.py` export metrics to New Relic (`newrelic` 8.8.0);
the exporter is the `MetricExporter` class [`monitor/metric_exporter.py:L7`].
`after_request` also records a `HttpResponseStatus` custom event per request [`server.py:L293-L294`].
The register/verify/login flow itself also emits New Relic **auth telemetry**: `class RegisterEvent`
[`app/events/auth_event.py:L28`] fires during registration and `class LoginEvent`
[`app/events/auth_event.py:L6`] fires during login (each `.send()` calls
`newrelic.agent.record_custom_event`). All of these require a New Relic account/Postfix and were
**not** exercised under the default local config; listed for completeness `(inferred from reading —
New Relic reporting is not observable under the default local config)`.

---

## 6. Build / Run commands, read-only status & cleanup proof

### 6.1 Exact bring-up (default, canonical configuration)

The canonical image is run as one long-lived container; the five processes are launched inside it.
The default `example.env` is copied to `.env` **unmodified**. Exact commands:

```
# 1. Canonical image, long-lived container, ports published
docker run -d --name simplelogin-app \
  --entrypoint bash \
  ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0 \
  -c "sleep infinity"          # image: andrewparkscaleai/coding-agent:simple-login__app__2cd6ee772d3531559588bcfb18627ffb5d2c

# 2. Inside the container: Python venv + Flask CLI + default config
source /app/venv/bin/activate          # Python 3.10.18, 182 pkgs
export FLASK_APP=server.py
cp -n /app/example.env /app/.env       # default config, auto-loaded via load_dotenv

# 3. Infrastructure
service postgresql start                # PostgreSQL 15.13 — role myuser/mypassword, db simplelogin, :5432
service redis-server start              # Redis 7.0.15 — :6379

# 4. Schema + seed
alembic upgrade head                    # head 32f25cbf12f6 → 77 tables
flask dummy-data                        # seeds john@wick.com/password (admin) + winston@continental.com

# 5. The five long-running processes (exact command lines as observed via `ps -eo pid,args`)
gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15      # webapp        (PIDs 525/526/527)
python email_handler.py                                  # aiosmtpd :20381 (PID 593)
python job_runner.py                                     # job runner    (PID 594)
python event_listener.py listener                        # event listener (PID 637)
yacron -c /app/crontab.yml                               # cron scheduler (PID 1238)
```

Resolved versions (from the running venv): Python **3.10.18**, flask **1.1.2**, SQLAlchemy
**1.3.24**, gunicorn **20.0.4**, redis-py **4.6.0**; PostgreSQL **15.13**, Redis **7.0.15** — matching
`pyproject.toml` / `poetry.lock`. Alembic head **`32f25cbf12f6`**, **77** tables. Seed login
`john@wick.com` / `password` [`CONTRIBUTING.md`].

### 6.2 Read-only status of the source tree

The investigation added **only** this document; no source file was modified. Captured on the host
working tree (branch `blitzy-19312d7e-6cc6-4b7b-b329-47abb71fdf6c`):

```
branch: blitzy-19312d7e-6cc6-4b7b-b329-47abb71fdf6c
base:   2cd6ee777f8c2d3531559588bcfb18627ffb5d2c
--- git status --porcelain ---
(empty above = clean)
--- git diff 2cd6ee77 --name-status ---
A	blitzy/documentation/app_2cd6ee777f8c.md
--- git diff 2cd6ee77 --stat ---
 blitzy/documentation/app_2cd6ee777f8c.md | 1224 ++++++++++++++++++++++++++++
 1 file changed, 1224 insertions(+)
```

`git status --porcelain` is empty (clean tree); the **only** file that differs from base `2cd6ee77`
is `blitzy/documentation/app_2cd6ee777f8c.md` (status `A`, added). No other source, template, config,
or manifest file was modified. All temporary observation scripts (`/tmp/blitzy_*.py`, cookie jars) were removed from the
container; none reside in the repository working tree.

### 6.3 Temporary-data cleanup proof (before → after)

Temp entities created during observation and their removal:

- **User A** (id 5) — removed via the **real** account-deletion job (§3.6); cascade removed its
  aliases (14, 16), mailbox (7), contact (4), email_log (4).
- **User B** (id 6) — removed via the canonical `User.delete(6)` model method (the same one the
  deletion job uses); cascade removed its alias (15), mailbox (8), activation_code (6).
- The diagnostic `SyncEvent` row (id 2), the observation-setup `Job` rows (10–21), the global-trash
  `deleted_alias` rows for the three temp aliases, and the temp `user_audit_log`/`alias_audit_log`
  rows were all explicitly deleted; `users_id_seq` was reset to its baseline (4).

Final baseline verification (single query, complete output):

```
 users_total | users_temp | alias_total | mailbox_total | job_total | sync_event_total | activation_code_total | deleted_alias_total | contact_total | email_log_total | user_audit_total | alias_audit_total | users_id_seq
-------------+------------+-------------+---------------+-----------+------------------+-----------------------+---------------------+---------------+-----------------+------------------+-------------------+--------------
           2 |          0 |          11 |             4 |         0 |                0 |                     0 |                   0 |             1 |               1 |                1 |                11 |            4
```

And a targeted check that **no** `blitzy-temp` reference survives anywhere
(users / mailbox / alias / deleted_alias / contact / user_audit_log / alias_audit_log):

```
 u | mb | al | da | ct | ual | aal
---+----+----+----+----+-----+-----
 0 |  0 |  0 |  0 |  0 |   0 |   0
```

The database is back to the seed baseline: only the two seed users (`john@wick.com`,
`winston@continental.com`), 11 seed aliases, 4 seed mailboxes, and the single seed contact/email_log
(user 1) and seed audit rows remain; `job`, `sync_event`, `activation_code`, and `deleted_alias` are
all empty. **Temp password/activation-code values shown in §2 belong to these now-deleted users and
are inert.**

---

## 7. Coverage checklist

Every named item in Q1/Q2/Q3, with where it is evidenced and whether it was **Observed** at runtime or
`(inferred)` from reading / labeled as a non-canonical **diagnostic**.

**Q1 — startup & readiness**
- [x] Webapp Gunicorn boot (`Listening at :7777`, `Booting worker`) — Observed §1.1 [`Dockerfile:L47`]
- [x] `>>> init logging <<<` banner (all 5 processes) — Observed §1.1–1.5 [`app/log.py:L67`], SL logger [`app/log.py:L79`]
- [x] Email handler `Listen for port 20381` / `Start mail controller` — Observed §1.2 [`email_handler.py:L2403,L2386`]
- [x] Event listener source/sink + `Starting to listen to events` — Observed §1.3 [`event_listener.py:L34,L43`; `events/event_source.py:L49`]; DEAD_LETTER/ConsoleEventSink modes — (inferred) §1.3
- [x] Job runner init banner + silent idle — Observed §1.4 (live poll in §3.5)
- [x] `yacron` scheduled-spawn lines — Observed §1.5 (full schedule §3.7)
- [x] `GET /health` → `200 success` — Observed §1.6 [`server.py:L213-L215`]
- [x] Login UI renders (`Welcome back!` + form) — Observed §1.7
- [x] Per-request access log + `/health` exclusion — Observed §1.8 [`server.py:L284,L281`]
- [x] Route resolution + blueprint registration — Observed §1.9 [`server.py:L234-L246`]
- [x] Boot seeding: SL domains + PGP keys — Observed §1.10 [`init_app.py:L16,L36,L42/L44`]

**Q2 — register → verify → login → dashboard**
- [x] Register creates user + waiting page — Observed §2.1 [`app/auth/views/register.py:L85,L104`]
- [x] `NOT_SEND_EMAIL`: activation link **NOT** printed (grep=0); AAP expectation NOT reproduced; DB lookup labeled FALLBACK — Observed §2.2 [`app/mail_sender.py:L130-L137`]
- [x] Verify flips `activated` f→t, deletes code, flash, 302 → /dashboard/ (before/during/after) — Observed §2.3 [`app/auth/views/activate.py:L49,L53,L56,L66`]
- [x] Forwarded into dashboard (200 + flash + first alias) — Observed §2.4
- [x] Login logs `log user … in` + 302 → /dashboard/ — Observed §2.5 [`app/auth/views/login_utils.py:L35,L44`]; FIDO/OTP branches — (inferred) §2.5
- [x] Edge: invalid code → 400 — Observed §2.6a [`app/auth/views/activate.py:L33`]
- [x] Edge: wrong password → 200 — Observed §2.6b [`app/auth/views/login.py:L49-L50`]
- [x] Edge: not-activated login → 200 — Observed §2.6c [`app/auth/views/login.py:L63-L69`]
- [x] Edge: resend activation → 200 — Observed §2.6d [`app/auth/views/resend_activation.py:L36`]
- [x] Edge: expired code → 400 (before/during/after) — Observed §2.6e [`app/auth/views/activate.py:L42-L43`]
- [x] Edge: disabled account → 200 (before/during/after + reset) — Observed §2.6f [`app/auth/views/login.py:L51-L56`]
- [x] Edge: scheduled-deletion → 200 (before/during/after + reset) — Observed §2.6g [`app/auth/views/login.py:L57-L62`]
- [x] Edge: `DISABLE_REGISTRATION` → 302 (before/after user-count) — Observed §2.6h [`app/auth/views/register.py:L38-L40`; `app/config.py:L138`]

**Q3 — behind the scenes**
- [x] `User.create` verified mailbox + first alias + onboarding-skip (0 jobs) — Observed §3.1 [`app/models.py:L611,L636,L646-L648`]; prod enqueues `onboarding-1`/`-2`/`-4` (`onboarding-3` defined but not enqueued) — (inferred) §3.1 [`app/models.py:L652,L657,L662`; `app/config.py:L303`]
- [x] Event pipeline REAL entry point reaches dispatcher gate — Observed §3.2 [`app/events/event_dispatcher.py:L62`; `app/dashboard/views/index.py:L110`]
- [x] Event MECHANISM (SyncEvent + NOTIFY + listener consume + `Runner.__on_event` re-queue) — **NON-CANONICAL DIAGNOSTIC** §3.3 [`app/events/event_dispatcher.py:L14,L23-L26`; `events/event_source.py:L55`; `events/event_sink.py:L19`; `events/runner.py:L11,L20,L42-L43`]
- [x] Event replay/debug utility `events/event_debugger.py` (`debug_event`/`run_event`) — (inferred) §3.3 [`events/event_debugger.py:L6,L31`]
- [x] Protobuf sync-event schema `event_pb2` (`AliasCreated`/`AliasCreatedList`) built by the dispatched `send_alias_creation_events_for_user` — §3.5 [`app/events/generated/event_pb2.py`; `app/jobs/event_jobs.py:L4,L24,L36,L47`]
- [x] Email forward alias→mailbox on :20381 (full path + reverse-alias) — Observed §3.4 [`email_handler.py:L2343,L2202,L580,L688,L740,L867,L2367`]; `handle_reply` — (inferred) §3.4 [`email_handler.py:L966`]
- [x] Job runner ~10.02s cadence (2 runs) + JobState done — Observed §3.5 (Job.create = labeled observation-setup) [`job_runner.py:L334,L347`; `app/models.py:L253-L257`]
- [x] Job runner REAL entry point (account deletion enqueues + drains delete-account) — Observed §3.6 [`app/dashboard/views/delete_account.py:L42`; `job_runner.py:L226,L235`]
- [x] GDPR export job `ExportUserDataJob` dispatched via `JOB_SEND_USER_REPORT` — (inferred, not in register/verify/login flow) §3.6 [`app/jobs/export_user_data_job.py:L41,L131`; `job_runner.py:L285-L288`; `app/config.py:L309`]
- [x] Cron `send_undelivered_mails` every ~300s (20 runs) + 15-job schedule — Observed §3.7 [`crontab.yml:L80`; `cron.py:L1314`]
- [x] Monitoring / New Relic — `MetricExporter`, `HttpResponseStatus` custom event, and `LoginEvent`/`RegisterEvent` auth telemetry — (inferred, not exercised) §3.8 [`monitoring.py:L1-L40`; `monitor/metric_exporter.py:L7`; `server.py:L293-L294`; `app/events/auth_event.py:L6,L28`]

**Constraints**
- [x] Default canonical configuration; exact build/run commands stated — §6.1
- [x] Timing/frequency observed at scale, stable across ≥2 runs (job poll, cron) — §3.5, §3.7
- [x] Real entry points used; non-canonical items explicitly labeled — §3.2/§3.3, §3.5/§3.6
- [x] Read-only: source tree unchanged (only this doc added) — §6.2
- [x] Temporary users/aliases removed; DB restored to seed baseline — §6.3
