# SimpleLogin Runtime Investigation — Web Server, Email Handler, and Job Runner

> An **evidence‑grounded** Q&A report. Every behavioural claim below is paired with the **exact command** that produced it and the **verbatim output** that was observed, plus a `file:line` citation into the source. Nothing here is written from reading code alone — the stack was **built, run, and observed first**, then documented.

## What SimpleLogin is, and what was tested

SimpleLogin is an open‑source email‑aliasing service implemented as a **Python/Flask monolith** — the Poetry project is literally named `name = "SimpleLogin"` `[pyproject.toml:L47]`. The single code package is launched through several independent process entry points. This report concerns the **three components named in the questions**:

| Component | Entry point | Port / cadence |
|-----------|-------------|----------------|
| **Web server** | `server.py` (dev) / `gunicorn wsgi:app` (prod) `[Dockerfile:L47]` | TCP **7777** |
| **Email handler** | `email_handler.py` (aiosmtpd) `[email_handler.py:L2381]` | TCP **20381** |
| **Job runner** | `job_runner.py` `[job_runner.py:L329]` | **10‑second** poll loop `[job_runner.py:L347]` |

All three are separate OS processes that **share the same database and configuration** because each builds the same lightweight Flask application context via `create_light_app()` (defined at `[server.py:L127]`, imported by `[email_handler.py:L177]` and `[job_runner.py:L24]`).

### How evidence was gathered

Everything was observed inside the canonical Docker runtime container `sl-runtime` (image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`, **Python 3.10.18**). Commands were run through `docker exec sl-runtime …`. The container ships `curl` and Python (`smtplib`, `socket`) but **not** `lsof`/`ss`/`netstat`/`nc`/`swaks`, so port‑binding was proven from `/proc/net/tcp` and inbound mail was injected with Python's `smtplib`. Each component's stdout was redirected to its own log file under `/tmp/blitzy_obs/` so lines could be quoted verbatim; those temporary files and all test data were removed afterward.

Observed supporting services: **PostgreSQL 15.13**, **Redis 7.0.15** (`redis-cli ping` → `PONG`), and **MailHog v1.0.1** as the local mail sink (SMTP `:1025`, HTTP API `:8025`). The runtime `/app/.env` is configured for local mail observation: `URL=http://localhost:7777` `[example.env:L6]`, `EMAIL_DOMAIN=sl.local` `[example.env:L22]`, `DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin` `[example.env:L75]`, `POSTFIX_SERVER=localhost`, `POSTFIX_PORT=1025` `[example.env:L154]`, and `NOT_SEND_EMAIL` `[example.env:L19]` is commented out so mail actually reaches the sink.

> **A note on runtime vs. canonical values.** `[CONTRIBUTING.md:L100]` documents Postgres 13 mapped `-p 15432:5432`; the running image actually provides Postgres **15.13** reached on the in‑container port **5432** (matching `DB_URI` `[example.env:L75]`). Where a value below is a canonical instruction it is cited to `CONTRIBUTING.md`/`example.env`; where it is a measured runtime value it is quoted from captured output.

---

## Runtime setup (commands actually run)

**Baseline before observation** (clean seed): two seeded users, an empty job queue, and an empty mail sink.

```bash
$ docker exec sl-runtime su postgres -c "psql -d simplelogin -c \"SELECT id, email FROM users ORDER BY id;\""
 id |          email
----+-------------------------
  1 | john@wick.com
  2 | winston@continental.com

$ docker exec sl-runtime su postgres -c "psql -d simplelogin -c \"SELECT id,name,state FROM job ORDER BY id;\""
(0 rows)

$ docker exec sl-runtime curl -s http://localhost:8025/api/v2/messages   # MailHog
total: 0
```

The seeded account `john@wick.com` / `password` comes from `flask dummy-data` `[server.py:L490-497]`, which calls `fake_data()` `[app/fake_data.py:L40]` seeding `email="john@wick.com"` `[app/fake_data.py:L45]` and `password="password"` `[app/fake_data.py:L47]`. The canonical bring‑up sequence is `alembic upgrade head && flask dummy-data && python3 server.py` `[CONTRIBUTING.md:L106]`, and the documented login is `john@wick.com / password` `[CONTRIBUTING.md:L109]`.

### Shared logging is initialized on import (`>>> init logging <<<`)

Every component imports `app/log.py`, which prints a one‑time marker on import and installs a shared stdout formatter under the logger name `SL`.

```bash
$ docker exec sl-runtime bash -lc 'cd /app && CONFIG=/app/.env /app/venv/bin/python -c "import app.log" 2>&1 | grep -n ">>> init logging <<<"'
7:>>> init logging <<<
```

That marker is `print(">>> init logging <<<")` at `[app/log.py:L67]`. The shared format string `[app/log.py:L12-15]` is:

```
%(asctime)s - %(name)s - %(levelname)s - %(process)d - "%(pathname)s:%(lineno)d" - %(funcName)s() - %(message_id)s - %(message)s
```

It is emitted to `sys.stdout` via `logging.StreamHandler(sys.stdout)` `[app/log.py:L41]`, the Werkzeug request logger is disabled (`log.disabled = True`) `[app/log.py:L70-71]`, the shortcuts `LOG.d/i/w/e` map to `debug/info/warning/exception` `[app/log.py:L74-77]`, and the logger is `LOG = _get_logger("SL")` `[app/log.py:L79]`. A useful consequence: **every SL log line embeds its own `"pathname:lineno"`**, so the runtime output itself confirms the `file:line` citations used throughout this report.

### Starting the three processes (stdout captured)

```bash
# Web server (production entry) — Dockerfile:L47
$ docker exec sl-runtime bash -lc 'cd /app && setsid /app/venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15 > /tmp/blitzy_obs/web.log 2>&1 < /dev/null &'

# Email handler — default --port 20381 (email_handler.py:L2399)
$ docker exec sl-runtime bash -lc 'cd /app && setsid /app/venv/bin/python email_handler.py > /tmp/blitzy_obs/email.log 2>&1 < /dev/null &'

# Job runner — 10s poll loop (job_runner.py:L329)
$ docker exec sl-runtime bash -lc 'cd /app && setsid /app/venv/bin/python job_runner.py > /tmp/blitzy_obs/job.log 2>&1 < /dev/null &'
```

`wsgi.py` exposes the callable Gunicorn loads (`from server import create_app` / `app = create_app()`), and `EXPOSE 7777` is declared at `[Dockerfile:L44]`. The captured startup output for each process is quoted in the Q1 section below.

---

## Q1 — "After the app starts, how can I tell the web server, email handler, and job runner are actually up and responding? What should I see in logs or the dashboard UI that confirms users can sign in and manage their aliases?"

There are three complementary signals per component: **startup log markers**, **a bound listening port**, and **a live response** (HTTP for the web server, an SMTP banner for the handler, the poll cadence for the runner). Then the **dashboard UI** confirms users can sign in and manage aliases.

### 1. Web server — startup markers, `/health`, `/live`, `/git`, bound port

**Claim: the web server boots and announces it is listening on 7777.** Captured from `web.log`:

```
[2026-07-01 22:04:07 +0000] [2009] [INFO] Starting gunicorn 20.0.4
[2026-07-01 22:04:07 +0000] [2009] [INFO] Listening at: http://0.0.0.0:7777 (2009)
[2026-07-01 22:04:07 +0000] [2009] [INFO] Using worker: sync
[2026-07-01 22:04:07 +0000] [2011] [INFO] Booting worker with pid: 2011
[2026-07-01 22:04:07 +0000] [2012] [INFO] Booting worker with pid: 2012
>>> init logging <<<
2026-07-01 22:04:08,362 - SL - DEBUG - 2011 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

*Why this confirms it:* these are the standard Gunicorn readiness markers — `Starting gunicorn`, `Listening at: http://0.0.0.0:7777`, the worker type, and one `Booting worker with pid` per worker (two, matching `-w 2` `[Dockerfile:L47]`). The version `gunicorn 20.0.4` matches the pinned dependency (`gunicorn ^20.0.4`, `[pyproject.toml]`). The `>>> init logging <<<` line and the following `SL`‑formatted line confirm the shared logger is live inside each worker.

**Claim: `GET /health` returns HTTP 200 with the body `success`.**

```bash
$ docker exec sl-runtime curl -i -s http://localhost:7777/health
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Content-Type: text/html; charset=utf-8
Content-Length: 7
Vary: Cookie
Set-Cookie: slapp=<REDACTED_SESSION_COOKIE>; Expires=...; HttpOnly; Path=/; SameSite=Lax

success
```

*Why this confirms it:* the route is `@app.route("/health", methods=["GET"])` returning `return "success", 200` `[server.py:L213-215]`. The status line is `HTTP/1.1 200 OK`, and `Content-Length: 7` equals `len("success")`. (The `Set-Cookie: slapp=…` value is a signed Flask session, not a credential; it is redacted here.)

**Claim: `GET /live` returns the body `live`.**

```bash
$ docker exec sl-runtime curl -i -s http://localhost:7777/live
HTTP/1.1 200 OK
Content-Length: 4
...
live
```

*Why this confirms it:* the route `@monitor_bp.route("/live")` returns `return "live"` `[app/monitor/views.py:L10-12]`; `Content-Length: 4` equals `len("live")`.

**Claim: `GET /git` returns the build SHA.**

```bash
$ docker exec sl-runtime curl -i -s http://localhost:7777/git
HTTP/1.1 200 OK
Content-Length: 3
...
dev
```

*Why this confirms it:* `@monitor_bp.route("/git")` returns the SHA1 build string `[app/monitor/views.py:L5-7]`; in this image the value is `dev`. The related `/exception` route `[app/monitor/views.py:L15-18]` deliberately raises `Exception("to make sure sentry works")` — it is a Sentry self‑test, **not** a liveness endpoint, and is named here only for completeness.

**Claim: port 7777 is bound and LISTENING.** Proven from the container's `/proc/net/tcp` (no `lsof`/`ss` available). Port `7777` is `0x1E61`, and the TCP `LISTEN` state is `0A`:

```bash
$ docker exec sl-runtime bash -lc 'grep -i "1E61" /proc/net/tcp'
   1: 00000000:1E61 00000000:0000 0A 00000000:00000000 00:00000000 ...
```

*Why this confirms it:* `00000000:1E61` with state `0A` is a socket listening on `0.0.0.0:7777`. (Extra `1E61` rows with state `06` are `TIME_WAIT` connections left by the `curl` calls above.)

### 2. Email handler — startup markers, bound port, SMTP banner

**Claim: the handler boots, listens on 20381, and starts the aiosmtpd controller.** Captured from `email.log`:

```
>>> init logging <<<
2026-07-01 22:04:26,184 - SL - INFO - 2035 - "/app/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-01 22:04:26,186 - SL - DEBUG - 2035 - "/app/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

*Why this confirms it:* `Listen for port 20381` is `LOG.i("Listen for port %s", args.port)` `[email_handler.py:L2403]` with the argparse default `--port 20381` `[email_handler.py:L2399]`; `Start mail controller 0.0.0.0 20381` is `LOG.d("Start mail controller %s %s", controller.hostname, controller.port)` `[email_handler.py:L2386]`, emitted right after `controller.start()` `[email_handler.py:L2385]` on the `Controller(MailHandler(), hostname="0.0.0.0", port=port)` created at `[email_handler.py:L2383]` inside `main()` `[email_handler.py:L2381]`. Note the log lines' own `pathname:lineno` fields read `"/app/email_handler.py:2403"` and `:2386` — the runtime output literally confirms the citations.

**Claim: port 20381 is bound, and the server answers with a `220` SMTP banner.** Port `20381` is `0x4F9D`:

```bash
$ docker exec sl-runtime bash -lc 'grep -i "4F9D" /proc/net/tcp'
   0: 00000000:4F9D 00000000:0000 0A 00000000:00000000 00:00000000 ...
```

```bash
$ docker exec sl-runtime /app/venv/bin/python - <<'PY'
import socket
s = socket.create_connection(("localhost", 20381), timeout=5)
print("220-banner:", repr(s.recv(1024).decode().strip()))
s.sendall(b"EHLO observer.local\r\n"); print("EHLO-resp :", repr(s.recv(1024).decode().splitlines()[0]))
s.sendall(b"QUIT\r\n"); s.close()
PY
220-banner: '220 b05f70854303 Python SMTP 1.4.2'
EHLO-resp : '250-b05f70854303'
```

*Why this confirms it:* state `0A` on `0x4F9D` proves it is listening on `0.0.0.0:20381`; the `220 … Python SMTP 1.4.2` greeting and the `250-` EHLO reply prove the aiosmtpd server is not just bound but actively speaking SMTP. (`Python SMTP 1.4.2` is the aiosmtpd version, satisfying `aiosmtpd ^1.2` `[pyproject.toml]`.) Connecting a client and reading the `220` banner is the standard way to verify an aiosmtpd server.

### 3. Job runner — poll cadence on stdout

The runner logs nothing per cycle when the queue is empty (its work only executes when jobs exist), so liveness is shown by the **poll cadence** once jobs are present. Three jobs were made eligible one‑per‑cycle; the runner picked each up on its own poll:

```
2026-07-01 22:09:34,412 - SL - DEBUG - 2053 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 1 onboarding-1 {'user_id': 999999}>
2026-07-01 22:09:44,450 - SL - DEBUG - 2053 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 2 onboarding-1 {'user_id': 999999}>
2026-07-01 22:09:54,469 - SL - DEBUG - 2053 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 3 onboarding-1 {'user_id': 999999}>
```

Measured deltas between consecutive `Take job` timestamps: **10.038 s** and **10.019 s**.

*Why this confirms it:* `Take job %s` is `LOG.d("Take job %s", job)` `[job_runner.py:L334]`, printed once per job the loop picks up; the ~10 s spacing is the `time.sleep(10)` at the bottom of the loop `[job_runner.py:L347]`. Observing **more than one** interval (two deltas across three cycles) demonstrates the cadence rather than inferring it. (`{'user_id': 999999}` is a throwaway payload used only to measure timing; see the cleanup note. These jobs then transitioned `ready(0) → taken(1) → done(2)`, confirming the full lifecycle at `[job_runner.py:L344]`.)

### 4. Dashboard UI — sign‑in and alias management signals

**Claim: an unauthenticated visitor is redirected to the login page.**

```bash
$ docker exec sl-runtime curl -s -o /dev/null -w "HTTP %{http_code} -> %{redirect_url}\n" http://localhost:7777/
HTTP 302 -> http://localhost:7777/auth/login
```

**Claim: signing in with the seeded credentials lands the user on the dashboard.**

```bash
# GET /auth/login to obtain the CSRF token + session cookie, then POST credentials
$ curl ... --data-urlencode "email=john@wick.com" --data-urlencode "password=password" \
        --data-urlencode "csrf_token=<REDACTED_CSRF>" http://localhost:7777/auth/login
POST /auth/login -> HTTP 302, redirect -> http://localhost:7777/dashboard/

$ curl -b <authenticated-cookie> -s -o /dev/null -w "GET / -> HTTP %{http_code} -> %{redirect_url}\n" http://localhost:7777/
GET / -> HTTP 302 -> http://localhost:7777/dashboard/
```

*Why this confirms it:* the sign‑in route is `@auth_bp.route("/login", methods=["GET", "POST"])` `[app/auth/views/login.py:L21]` / `def login()` `[app/auth/views/login.py:L25]`; on success the browser is redirected, and an already‑authenticated request to `/` is sent to the dashboard by the root route `[server.py:L251-256]` (authenticated → `dashboard.index`, otherwise → `auth.login`).

**Claim: the dashboard shows the user's aliases and a per‑user alias count — the alias‑management surface.**

```bash
$ curl -b <authenticated-cookie> -s http://localhost:7777/dashboard/ -w "GET /dashboard/ -> HTTP %{http_code}\n" -o dashboard.html
GET /dashboard/ -> HTTP 200
$ grep -oE "e[0-9]+@sl\.local" dashboard.html | sort -u
e0@sl.local
e1@sl.local
e2@sl.local
$ grep -o "john@wick.com" dashboard.html | head -1
john@wick.com
```

Cross‑checked against the database (the same count the dashboard computes):

```bash
$ docker exec sl-runtime su postgres -c "psql -d simplelogin -tAc \"SELECT count(*) FROM alias WHERE user_id=1;\""
10
```

*Why this confirms it:* the dashboard renders the signed‑in identity (`john@wick.com`) and the seeded aliases `e0@sl.local`, `e1@sl.local`, `e2@sl.local` (sample aliases `e{i}@{EMAIL_DOMAIN}` with `EMAIL_DOMAIN=sl.local` `[example.env:L22]`). The per‑user count is `nb_alias = Alias.filter_by(user_id=user.id).count()` `[app/dashboard/views/index.py:L33]`, which equals the `10` measured directly in the DB. Seeing the alias list plus this count on an HTTP 200 dashboard is the UI signal that a user can sign in and manage aliases.

---

## Q2 — "Perform basic actions — create a new account, create an alias, and have that alias receive an email. What happens when a new account interacts with aliases or tries to receive mail? How does the system show these actions were handled correctly? What should I see at runtime that confirms expected behavior?"

### Action 1 — Create a new account

A fresh account was created through the same `User.create(...)` path the app uses:

```bash
$ docker exec sl-runtime /app/venv/bin/python /tmp/blitzy_obs/create_account.py
2026-07-01 22:14:48,747 - SL - DEBUG - 2523 - "/app/app/models.py:647" - create() -  - Disable onboarding emails
Created account: <User 3 Blitzy Test blitzy_newacct@example.com>  activated=True
Default verified mailbox: blitzy_newacct@example.com verified=True
```

*What happened and why it confirms correct handling:* `User.create` `[app/models.py:L601-668]` created `<User 3 …>` and, per `mb = Mailbox.create(user_id=user.id, email=user.email, verified=True)` `[app/models.py:L611]`, gave it a **verified default mailbox** equal to its own email — a new account is immediately able to receive forwarded mail.

**Notable, and reported honestly:** the log line `Disable onboarding emails` comes from the guard `if config.DISABLE_ONBOARDING: LOG.d("Disable onboarding emails"); return user` `[app/models.py:L646-648]`, which **returns early before** the three onboarding `Job.create(...)` calls at `[app/models.py:L650-665]`. In this runtime `DISABLE_ONBOARDING=true` `[.env:L150]` and `DISABLE_ONBOARDING = "DISABLE_ONBOARDING" in os.environ` `[app/config.py:L401]` is truthy by mere presence — so **account creation enqueued zero onboarding jobs here**. When onboarding is enabled, those jobs are created with `run_at=arrow.now().shift(days=1|2|3)` `[app/models.py:L654,L659,L664]`, i.e. scheduled 1–3 days out, which `get_jobs_to_run()` (window `run_at <= now + 10 min`, `[job_runner.py:L307-326]`) would not pick up during a short observation regardless. (The seeded interactive account for sign‑in remains `john@wick.com` / `password` `[app/fake_data.py:L45,L47]`.)

### Action 2 — Create an alias

Signed in as `john@wick.com`, a random alias was created from the dashboard (`POST /dashboard/` with `form-name=create-random-email`):

**Success flash (what the UI shows):**

```
Alias test_test485@sl.local has been created
```

**Server‑side log line (index.py:L110), captured from `web.log`:**

```
2026-07-01 22:12:59,170 - SL - DEBUG - 2012 - "/app/app/dashboard/views/index.py:110" - index() -  - create new random alias <Alias 12 test_test485@sl.local> for user <User 1 John Wick john@wick.com>
```

**Alias count incremented (10 → 11):**

```bash
$ docker exec sl-runtime su postgres -c "psql -d simplelogin -tAc \"SELECT count(*) FROM alias WHERE user_id=1;\""
11
```

*Why this confirms correct handling:* the random‑alias branch runs `alias = Alias.create_new_random(user=current_user, scheme=scheme)` `[app/dashboard/views/index.py:L104]`, sets its mailbox, commits, logs `LOG.d("create new random alias %s for user %s", …)` `[app/dashboard/views/index.py:L110]`, and flashes `flash(f"Alias {alias.email} has been created", "success")` `[app/dashboard/views/index.py:L111]`. The exact flash text and the new alias `test_test485@sl.local` (`<Alias 12>`) appear together, and the persisted count `nb_alias` `[app/dashboard/views/index.py:L33]` rose from 10 to 11. (The alternate manual path is `@dashboard_bp.route("/custom_alias", methods=["GET", "POST"])` `[app/dashboard/views/custom_alias.py:L30]` / `def custom_alias()` `[app/dashboard/views/custom_alias.py:L34]`.)

### Action 3 — Have the alias receive an email

A message was sent to the new alias on port 20381 using Python's `smtplib` (`swaks` is absent):

```bash
$ docker exec sl-runtime /app/venv/bin/python - <<'PY'
import smtplib
from email.mime.text import MIMEText
msg = MIMEText("Blitzy runtime observation: inbound delivery test to a SimpleLogin alias.")
msg["Subject"] = "Blitzy inbound test"; msg["From"] = "tester@example.com"; msg["To"] = "test_test485@sl.local"
with smtplib.SMTP("localhost", 20381, timeout=15) as s:
    print("sendmail returned:", s.sendmail("tester@example.com", ["test_test485@sl.local"], msg.as_string()))
PY
sendmail returned: {}
```

*An empty dict from `sendmail` means every recipient was accepted (a 2xx result).*

**Handler `New message` line (email_handler.py:L2343‑2344) — quoted verbatim, including the source typo `rctp tos`:**

```
2026-07-01 22:13:15,126 - SL - INFO - 2035 - "/app/email_handler.py:2343" - _handle() - 5893219c-8918-40c7-9f15-54f2126fe6db - New message, mail from tester@example.com, rctp tos ['test_test485@sl.local'] 
```

**Handler `Finish` line (email_handler.py:L2367‑2368) — measured elapsed time and SMTP return code:**

```
2026-07-01 22:13:15,331 - SL - INFO - 2035 - "/app/email_handler.py:2367" - _handle() - 5893219c-8918-40c7-9f15-54f2126fe6db - Finish mail_from tester@example.com, rcpt_tos ['test_test485@sl.local'], takes 0.20510578155517578 seconds with return code '250 Message accepted for delivery'<<===
```

*Why this confirms correct handling:* the inbound coroutine is `async def handle_DATA(self, server, session, envelope)` `[email_handler.py:L2289]`. The receipt log is `LOG.i("New message, mail from %s, rctp tos %s ", …)` `[email_handler.py:L2343-2344]` — the literal really is spelled **`rctp tos`** in the source, quoted here exactly. The completion log is `LOG.i("Finish mail_from %s, rcpt_tos %s, takes %s seconds with return code '%s'<<===", …)` `[email_handler.py:L2367-2368]`. The two **measured** values are the elapsed **`0.20510578155517578` seconds** and the return code **`250 Message accepted for delivery`** — a `2xx` code denotes success (a `5xx` would denote failure). The shared identifier `5893219c‑8918‑40c7‑9f15‑54f2126fe6db` in both lines is the `%(message_id)s` field of the log format `[app/log.py:L12-15]`.

**End‑to‑end delivery landed in the mail sink (MailHog):**

```bash
$ docker exec sl-runtime curl -s http://localhost:8025/api/v2/messages   # parsed:
MailHog total: 1
  Envelope MAIL FROM : sl.<verp>@sl.local          # SimpleLogin return-path
  Envelope RCPT TO   : john@wick.com               # forwarded to john's verified mailbox
  Subject            : Blitzy inbound test
  X-SimpleLogin-Type : Forward
  From               : "tester at example.com" <tester_at_example_com_gsjij@sl.local>
  To                 : test_test485@sl.local
```

*Why this is the end‑to‑end proof:* the message the external sender addressed to the alias `test_test485@sl.local` was **forwarded by the handler to John's verified mailbox `john@wick.com`** and captured by the sink. The `X-SimpleLogin-Type: Forward` header marks it as a forward, and the `From` was rewritten to a reverse‑alias (`tester_at_example_com_gsjij@sl.local`) so a reply would route back through SimpleLogin. This is exactly what a new account interacting with an alias to receive mail should produce.

### Negative / edge behaviour (what a failure looks like)

**Bad credentials** are rejected with an explicit flash:

```bash
$ curl ... --data-urlencode "email=john@wick.com" --data-urlencode "password=WRONGPASSWORD" http://localhost:7777/auth/login
# resulting page contains:
Email or password incorrect
```

*Why:* the login view flashes `flash("Email or password incorrect", "error")` `[app/auth/views/login.py:L49]` on a failed authentication (an already‑authenticated user is instead redirected to `dashboard.index` `[app/auth/views/login.py:L34]`).

The following inbound‑mail error signals were **not triggered** by the successful test above (which returned `250`); they are grounded in code and named here as the expected negative‑path responses (labeled *not directly observed*):

- **Recipient cannot receive mail** — `if not user.can_send_or_receive(): LOG.i(f"User {user} cannot receive emails")` `[email_handler.py:L563-564]`, returning `status.E207` (`"250 SL E207 No bounce report"` `[app/email/status.py:L12]`) when the sender is an ignorable bounce, otherwise `status.E504` (`"550 SL E504 Account disabled"` `[app/email/status.py:L41]`).
- **Reverse‑alias misuse** — `return status.E524` (`"550 SL E524 Wrong use of reverse-alias"` `[app/email/status.py:L62]`) at `[email_handler.py:L2306]` inside `handle_DATA`.
- **VERP error conditions** — `return status.E213` (`"250 SL E213 Unknown email ignored"` `[app/email/status.py:L21]`) at `[email_handler.py:L2318]`.

The `User`, `Alias`, `Contact`, and `Job` models that back all of this observed state are defined at `[app/models.py:L336]`, `[app/models.py:L1469]`, `[app/models.py:L1863]`, and `[app/models.py:L2683]` respectively.

---

## Q3 — "While the application is active, do the email handler and job runner automatically come online in the background to support email activity or data handling? What behavior suggests these internal components are functioning as intended across different situations?"

**Yes.** Both are launched as their own long‑lived OS processes and run unattended in the background. After the single startup command each, they required no further intervention across the whole investigation:

```bash
$ docker exec sl-runtime ps -o pid,etime,cmd -p 2009,2035,2053
    PID     ELAPSED CMD
   2009       13:34 /app/venv/bin/python /app/venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
   2035       13:17 /app/venv/bin/python email_handler.py
   2053       12:59 /app/venv/bin/python job_runner.py
```

They share one consistent database/config because **all three build the same lightweight app context**: `create_light_app()` is defined at `[server.py:L127]`, imported by the handler at `[email_handler.py:L177]` (used at `[email_handler.py:L2352]`) and by the runner at `[job_runner.py:L24]` (used at `[job_runner.py:L332]`). That shared context is why the web‑created alias and the enqueued jobs above were visible to the handler and the runner.

### Email handler — auto‑accepts and processes every message

The handler's `main()` keeps the aiosmtpd controller alive with a keepalive loop `while True: time.sleep(2)` `[email_handler.py:L2392-2393]`. It accepts inbound connections and processes each message with **no per‑message intervention** — demonstrated in Q2, where sending to the alias caused the `New message …` / `Finish … return code '250 Message accepted for delivery'` pair to appear on the handler's stdout automatically. That automatic pair is the behaviour that shows the handler is functioning.

### Job runner — polls and dispatches automatically, and does so "across different situations"

The main loop is: `while True:` `[job_runner.py:L330]` → `with create_light_app().app_context():` `[job_runner.py:L332]` → `for job in get_jobs_to_run():` `[job_runner.py:L333]` → `LOG.d("Take job %s", job)` `[job_runner.py:L334]` → `process_job(job)` `[job_runner.py:L342]` → `job.state = JobState.done.value` `[job_runner.py:L344]` → `time.sleep(10)` `[job_runner.py:L347]`.

**Situation A — idle (no jobs):** from startup (22:04:43) until the first job was enqueued (~22:09:33), roughly five minutes, the runner produced **no per‑cycle log output and zero errors** — it polls silently (the `for` body runs only when `get_jobs_to_run()` returns rows) yet stays alive. *This is the expected quiet‑but‑ready behaviour.*

**Situation B — a burst of jobs across consecutive cycles:** the three `Take job` lines quoted in Q1 (22:09:34 / :44 / :54) with measured **10.038 s** and **10.019 s** gaps show it dispatching work on the fixed 10‑second cadence `[job_runner.py:L347]`.

**Situation C — a single real job, dispatched to its handler branch:** one `onboarding-1` job for the freshly‑created, activated User 3 was made eligible and the runner picked it up automatically on the next poll:

```
2026-07-01 22:15:34,891 - SL - DEBUG - 2053 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 4 onboarding-1 {'user_id': 3}>
2026-07-01 22:15:34,895 - SL - DEBUG - 2053 - "/app/job_runner.py:196" - process_job() -  - send onboarding send-from-alias email to user <User 3 Blitzy Test blitzy_newacct@example.com>
```

*Why this confirms it:* the first line is the poll pickup (`Take job`, `[job_runner.py:L334]`); the second is the **`JOB_ONBOARDING_1` branch of `process_job` actually executing** — `LOG.d("send onboarding send-from-alias email to user %s", user)` `[job_runner.py:L196]` — for a real activated user (its `pathname` field reads `"/app/job_runner.py:196"`). The job then transitioned to `done` `[job_runner.py:L344]`. Together, A/B/C show the runner behaving consistently whether idle, under a burst, or handling a single job.

**Selection logic** — `get_jobs_to_run()` `[job_runner.py:L307-326]` returns jobs where `state == ready` (0) **or** a `taken` (1) job gone stale (`taken_at` older than the retry window and `attempts` under the max), **and** whose `run_at` is `NULL` or `<= now + 10 min`. That is exactly why immediate (`run_at=now`) jobs were eligible at once, while account‑creation onboarding jobs (scheduled days ahead) would not be.

**Full `process_job` dispatch inventory** `[job_runner.py:L188-304]` — the runner branches on `job.name`. Directly observed: **`JOB_ONBOARDING_1`** (`"onboarding-1"` `[app/config.py:L301]`) at `[job_runner.py:L189]`. The remaining branches are grounded in code and were **not directly observed** in this run (described here for completeness):

| `job.name` | Line | Purpose |
|------------|------|---------|
| `JOB_ONBOARDING_1` = `"onboarding-1"` | `[job_runner.py:L189]` | send‑from‑alias onboarding email *(observed)* |
| `JOB_ONBOARDING_2` = `"onboarding-2"` | `[job_runner.py:L198]` | mailbox onboarding email |
| `JOB_ONBOARDING_4` = `"onboarding-4"` | `[job_runner.py:L207]` | PGP onboarding email |
| `JOB_BATCH_IMPORT` | `[job_runner.py:L222]` | batch alias import |
| `JOB_DELETE_ACCOUNT` | `[job_runner.py:L226]` | account deletion |
| `JOB_DELETE_MAILBOX` | `[job_runner.py:L245]` | mailbox deletion |
| `JOB_DELETE_DOMAIN` | `[job_runner.py:L248]` | custom‑domain deletion |
| `JOB_SEND_USER_REPORT` | `[job_runner.py:L285]` | user‑data export |
| `JOB_SEND_PROTON_WELCOME_1` | `[job_runner.py:L289]` | Proton welcome email |
| `JOB_SEND_ALIAS_CREATION_EVENTS` | `[job_runner.py:L295]` | alias‑creation events |
| *(fallback)* | `[job_runner.py:L304]` | `LOG.e("Unknown job name %s", job.name)` |

The onboarding constants are defined at `[app/config.py:L301]` (`onboarding-1`), `[app/config.py:L302]` (`onboarding-2`), and `[app/config.py:L304]` (`onboarding-4`).

---

## Coverage pass

Every named item and every "e.g. / such as / including" example across the three questions, and whether it was **directly observed at runtime (RT)** or **grounded in code (code)**.

**The three components / ports**
- Web server on **7777** — RT (gunicorn `Listening at: http://0.0.0.0:7777`; `/proc/net/tcp` `1E61` state `0A`).
- Email handler on **20381** — RT (`Listen for port 20381`; `Start mail controller 0.0.0.0 20381`; `4F9D` state `0A`; `220` banner).
- Job runner **10‑second** poll — RT (measured deltas 10.038 s, 10.019 s; `time.sleep(10)` `[job_runner.py:L347]`).

**Web endpoints**
- `/health` → `success` / `200` `[server.py:L213-215]` — RT. `/live` → `live` `[app/monitor/views.py:L10-12]` — RT. `/git` → `dev` `[app/monitor/views.py:L5-7]` — RT. `/exception` `[app/monitor/views.py:L15-18]` — code (Sentry self‑test; not a liveness route).

**Web entry / config**
- `create_app` `[server.py:L139]`, `create_light_app` `[server.py:L127]`, dev launcher `app.run(debug=True, port=7777)` `[server.py:L588]`, gunicorn CMD + `EXPOSE 7777` `[Dockerfile:L44,L47]`, `wsgi.py` (`create_app`), root redirect `[server.py:L251-256]`, `dummy-data` CLI `[server.py:L490-497]` — all cited; startup and redirect behaviour RT.

**Email handler**
- `main` `[email_handler.py:L2381]`, `Controller` `[email_handler.py:L2383]`, `controller.start()` `[email_handler.py:L2385]`, `Start mail controller` `[email_handler.py:L2386]` — RT. `--port` default 20381 `[email_handler.py:L2399]`, `Listen for port` `[email_handler.py:L2403]` — RT. Keepalive `[email_handler.py:L2392-2393]` — code. `handle_DATA` `[email_handler.py:L2289]`, `New message … rctp tos` `[email_handler.py:L2343-2344]` (verbatim typo), `Finish … return code` `[email_handler.py:L2367-2368]` — RT. `create_light_app` use `[email_handler.py:L177,L2352]` — code. cannot‑receive `[email_handler.py:L563-564]`, `E524` `[email_handler.py:L2306]`, `E213` `[email_handler.py:L2318]` — code (negative paths not triggered).

**Job runner**
- import `[job_runner.py:L24]`, `while True` `[job_runner.py:L330]`, app_context `[job_runner.py:L332]`, `for job` `[job_runner.py:L333]` — code. `Take job` `[job_runner.py:L334]`, `process_job` `[job_runner.py:L342]`, `done` `[job_runner.py:L344]`, `time.sleep(10)` `[job_runner.py:L347]` — RT. `get_jobs_to_run` `[job_runner.py:L307-326]` — code. `process_job` branches `[job_runner.py:L188-304]` + `Unknown job name` `[job_runner.py:L304]` — `onboarding-1` RT; all others code.

**User flows / seed**
- login route/def `[app/auth/views/login.py:L21,L25]` — RT. authed redirect `[app/auth/views/login.py:L34]` — code. bad‑creds flash `[app/auth/views/login.py:L49]` — RT. `nb_alias` `[app/dashboard/views/index.py:L33]` — RT (10 then 11). random‑alias block `[app/dashboard/views/index.py:L104-111]` + success flash `[app/dashboard/views/index.py:L111]` — RT. custom alias `[app/dashboard/views/custom_alias.py:L30-34]` — code. seed `john@wick.com` / `password` `[app/fake_data.py:L45,L47]` — RT. models `User/Alias/Contact/Job` `[app/models.py:L336,L1469,L1863,L2683]` — code.

**Logging**
- format `[app/log.py:L12-15]`, stdout handler `[app/log.py:L41]`, `>>> init logging <<<` `[app/log.py:L67]`, werkzeug disabled `[app/log.py:L70-71]`, `LOG.d/i/w/e` `[app/log.py:L74-77]`, `SL` logger `[app/log.py:L79]` — marker + format + `SL` name RT; the rest code.

**Config / run**
- `URL` `[example.env:L6]`, `NOT_SEND_EMAIL` `[example.env:L19]`, `EMAIL_DOMAIN` `[example.env:L22]`, `DB_URI` `[example.env:L75]`, `POSTFIX_PORT` `[example.env:L154]`, run sequence `[CONTRIBUTING.md:L106]`, login `[CONTRIBUTING.md:L109]`, MailHog API `:8025` (UI mapped to host `:1080`), dep versions `[pyproject.toml]` — cited; service liveness (Postgres/Redis/MailHog) RT.

**"e.g. / such as" example clauses (treated as required)**
- "web server, email handler, and job runner" — each answered individually in Q1/Q3.
- "create a new account, create an alias, and have that alias receive an email" — each answered individually in Q2.
- "in logs or the dashboard UI" — logs (startup + handler + runner) and dashboard UI (sign‑in redirect, alias list, `nb_alias`) both answered in Q1.
- "across different situations" — idle / burst / single‑job situations answered in Q3.

### Explicitly flagged — not verified by running

1. **Onboarding jobs are NOT auto‑enqueued on account creation in this runtime.** `DISABLE_ONBOARDING=true` `[.env:L150]` triggers the early return at `[app/models.py:L646-648]`; the `Disable onboarding emails` log was observed. The onboarding `Job.create` calls at `[app/models.py:L650-665]` (with `run_at=now+1/2/3 days`) were therefore **not exercised via account creation**. The `onboarding-1` dispatch shown in Q3 was driven by directly enqueuing an eligible (`run_at=now`) job for the new activated user.
2. **`process_job` branches other than `onboarding-1`** (`onboarding-2`, `onboarding-4`, batch import, delete account/mailbox/domain, user report, Proton welcome, alias‑creation events, and the `Unknown job name` fallback) were **not directly observed**; they are described from source with their `file:line`.
3. **Inbound negative paths** (`E504`/`E207` cannot‑receive, `E524` reverse‑alias, `E213` VERP) were **not triggered** — the successful test returned `250 Message accepted for delivery`. Their code and status literals are cited from `email_handler.py` and `app/email/status.py`.
4. **The `onboarding-1` branch's outbound email was not confirmed in MailHog** — the branch‑dispatch log `[job_runner.py:L196]` proves the branch executed, but the sink total did not increase for it, so no MailHog delivery is claimed for that job (only the branch execution is claimed). The Q2 inbound‑forward message *was* confirmed in MailHog.
5. **Runtime service versions differ from the canonical docs** — observed **PostgreSQL 15.13** / **Redis 7.0.15** vs. the Postgres 13 / Redis v6 referenced in `CONTRIBUTING.md`; this does not affect any behaviour cited above.

---

*All temporary observation scripts, captured log files, and test data (the temporary account, the temporary alias, the timing jobs, and the captured mail) were removed after evidence capture; the repository's only change is this document.*
