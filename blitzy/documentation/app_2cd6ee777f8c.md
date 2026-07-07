# SimpleLogin — Runtime Verification Q&A

**How do I know a locally running SimpleLogin instance is healthy, how do its basic user flows behave, and do its background components come online on their own?**

This document answers those three questions **from directly observed runtime behavior**. SimpleLogin is a self-hostable email-alias service: users create aliases, and mail sent to an alias is forwarded to the user's real mailbox while replies go back out *as the alias*, hiding the real address. The service is a single Python/Flask codebase that is run as several **independent** processes. This investigation focuses on the three named in the question:

| Component | Entry point | Role | Default port |
|-----------|-------------|------|--------------|
| **Web server** | `python server.py` | Dashboard + REST API + health endpoints | `7777` |
| **Email handler** | `python email_handler.py` | aiosmtpd SMTP server that forwards/replies/bounces mail | `20381` |
| **Job runner** | `python job_runner.py` | Polls the `job` table every ~10 s and runs queued work | (none) |

Every factual claim below is backed by three things: **(a)** the exact command that produced the evidence, **(b)** the complete, unedited captured output, and **(c)** a `file:line` citation into this repository. All commands were run through each component's real entry point — no bypasses or stand-ins.

> **Method (run-first).** The stack was built and run first; temporary observation scripts were executed and their output captured *before* this prose was written. Every temporary account, alias, contact, email-log, job and script created during the investigation was removed afterward, and the database was verified back at its exact seed baseline — see **§6 (Cleanup and read-only proof)**, which shows the deletion commands, the before/after row counts, and a clean `git status`. The repository is left unchanged except for this document.

---

## 1. How the stack was run

### 1.1 Runtime and provisioning

Everything ran inside the provided Docker container `sl-app` (image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`), which supplies the canonical Python 3.10 / PostgreSQL / Redis runtime. Commands were issued with `docker exec sl-app bash -c '<command>'`.

A few generic names are used throughout so no host-specific paths or secrets are hard-coded into the commands (each is defined once, here):

| Name in commands | Meaning |
|------------------|---------|
| `$CONFIG` | the container's dotenv file, loaded by python-dotenv via the `CONFIG` env var |
| `$OBS` | a temporary directory that held the captured process logs and observation scripts (removed in §6) |
| `$SINK` | the local MTA sink's maildir — one `.eml` file per received message |
| `python` | the container's canonical Poetry-built virtualenv interpreter (already on `PATH` once the venv is active) |
| `psql` | connects with `-h localhost -U <user> -d <db>`; the password is supplied from `~/.pgpass` (`chmod 600`), so **no password ever appears on a command line** |

> Captured program output is shown **verbatim** and therefore contains absolute container paths — notably the repository root `/app`, which also appears inside every log line's `pathname:lineno` field, and the dotenv file path in the config-load banner. Those strings are the programs' own output, not added by this document.

```
$ python --version; git -C /app rev-parse --short HEAD
Python 3.10.18
2cd6ee77
```

- **Python 3.10** matches `Dockerfile:8` (`FROM python:3.10`) and `pyproject.toml:61` (`python = "^3.10"`).
- The canonical interpreter is the pre-built Poetry virtualenv; dependencies were installed at image-build time (Poetry itself is not on the runtime `PATH`).
- Configuration is loaded from `$CONFIG` via the `CONFIG` environment variable (python-dotenv). The relevant values are `URL=http://localhost:7777`, `EMAIL_DOMAIN=sl.local`, `DB_URI=postgresql://<user>:<redacted>@localhost:5432/<db>`, and `MEM_STORE_URI=redis://localhost`. `DB_URI` is read at `app/config.py:192` (`DB_URI = os.environ["DB_URI"]`) and `MEM_STORE_URI` at `app/config.py:568`.

**Mail sink.** SimpleLogin forwards outbound mail through the address in `POSTFIX_SERVER`/`POSTFIX_PORT`. In this container `NOT_SEND_EMAIL=False` and those keys point at a local aiosmtpd **sink** listening on `0.0.0.0:1025`, which writes each received message to `$SINK/*.eml`. (The published guide references a mailcatcher web UI at `http://localhost:1080/`, `CONTRIBUTING.md:221`; here the equivalent evidence is the `.eml` files the sink writes.)

### 1.2 Seeded login

The database is seeded by the `dummy-data` Flask CLI command, defined at `server.py:490` (`@app.cli.command("dummy-data")`), function `dummy_data()` at `server.py:491`, which logs `reset db, add fake data` at `server.py:494` and calls `fake_data()` at `server.py:495`. Captured on a throwaway scratch database (`sl_boot`) so the live database was untouched:

```
$ cd /app && CONFIG=$CONFIG DB_URI="postgresql://<user>:<redacted>@localhost:5432/sl_boot" \
      FLASK_APP=server.py python -m flask dummy-data
load config file /root/run.env
>>> URL: http://localhost:7777
Upload files to local dir
>>> init logging <<<
2026-07-06 23:52:23,306 - SL - DEBUG - 5624 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-06 23:52:24,150 - SL - WARNING - 5624 - "/app/server.py:494" - dummy_data() -  - reset db, add fake data
2026-07-06 23:52:24,150 - SL - DEBUG - 5624 - "/app/app/fake_data.py:41" - fake_data() -  - create fake data
2026-07-06 23:52:24,445 - SL - INFO - 5624 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-06 23:52:24,475 - SL - DEBUG - 5624 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email list_word311@sl.local
2026-07-06 23:52:24,487 - SL - INFO - 5624 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-06 23:52:24,539 - SL - INFO - 5624 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-06 23:52:24,549 - SL - INFO - 5624 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-06 23:52:24,572 - SL - INFO - 5624 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-06 23:52:24,585 - SL - INFO - 5624 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-06 23:52:24,600 - SL - INFO - 5624 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-06 23:52:24,609 - SL - INFO - 5624 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-06 23:52:24,620 - SL - DEBUG - 5624 - "/app/app/models.py:1255" - generate_oauth_client_id() -  - generate oauth_client_id demo-murwwkbnwv
2026-07-06 23:52:24,626 - SL - DEBUG - 5624 - "/app/app/models.py:1255" - generate_oauth_client_id() -  - generate oauth_client_id demo2-wfifyjymng
2026-07-06 23:52:24,903 - SL - INFO - 5624 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-06 23:52:24,930 - SL - INFO - 5624 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-06 23:52:24,942 - SL - INFO - 5624 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-06 23:52:24,952 - SL - INFO - 5624 - "/app/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain
```

The seed creates the login **`john@wick.com` / `password`**. Critically, `fake_data()` (`app/fake_data.py:40`) builds it with `activated=True` (`app/fake_data.py:48`) and `is_admin=True` (`app/fake_data.py:49`), so it can sign in to the dashboard immediately with no activation step. This account is used for all alias flows below; throwaway accounts are reserved for the signup demonstration.

```
$ psql -c \
    "SELECT id,email,activated,is_admin FROM users ORDER BY id LIMIT 2;"
 id |          email          | activated | is_admin
----+-------------------------+-----------+----------
  1 | john@wick.com           | t         | t
  2 | winston@continental.com | t         | f
```

### 1.3 The logging convention (how to read every signal below)

All three processes log to **stdout** through one logger named `"SL"` (`LOG = _get_logger("SL")`, `app/log.py:79`) at DEBUG level. The format is assembled by the `_log_format` tuple at `app/log.py:12-15`, which concatenates to:

```
%(asctime)s - %(name)s - %(levelname)s - %(process)d - "%(pathname)s:%(lineno)d" - %(funcName)s() - %(message_id)s - %(message)s
```

So every line looks like this (from the seed output above):

```
2026-07-06 23:52:24,150 - SL - WARNING - 5624 - "/app/server.py:494" - dummy_data() -  - reset db, add fake data
└─ timestamp        └─ logger └─ level └─ pid └─ pathname:lineno └─ funcName()   └─(msg_id) └─ message
```

The key consequence: **each log line embeds its own `pathname:lineno` and `funcName()`**, which is exactly how any captured line is tied back to a `file:line` in this document. The convenience shortcuts `LOG.d` / `LOG.i` / `LOG.w` / `LOG.e` (debug/info/warning/exception) are defined at `app/log.py:74-77`, the `>>> init logging <<<` banner is printed on import at `app/log.py:67`, and Werkzeug's per-request HTTP access log is silenced at `app/log.py:70-71` (which is why you never see noisy `GET /health 200` lines).

---

## 2. R1 — How do I know each process is up?

Each of the three named processes was started through its **real entry point** into its own log file under `$OBS`:

```
$ mkdir -p "$OBS" && cd /app && export CONFIG=$CONFIG
$ setsid nohup python -u server.py        > "$OBS/server.log"        2>&1 &
$ setsid nohup python -u email_handler.py > "$OBS/email_handler.log" 2>&1 &
$ setsid nohup python -u job_runner.py    > "$OBS/job_runner.log"    2>&1 &
```

(`python -u` is used because the image does not set `PYTHONUNBUFFERED`, so unbuffered stdout is needed to capture logs live.) The exact PIDs printed by `$!` at spawn were recorded so that later steps can signal *only* those processes: **server 5696** (with reloader child **5730**), **email handler 5694**, **job runner 5695**.

### 2.1 Web server (`server.py`)

The dev entry point is `app.run(debug=True, port=7777)` at `server.py:588`; the app object is assembled by `create_app()` at `server.py:139`. (For contrast, production runs `gunicorn wsgi:app -b 0.0.0.0:7777` per `Dockerfile:47`, where `wsgi.py:1,3` is `from server import create_app` / `app = create_app()`.) Its startup output shows the dev server coming up in debug mode (the second `>>> init logging <<<` is Werkzeug's reloader child re-importing the app — hence the parent/child PID pair 5696/5730):

```
load config file /root/run.env
>>> URL: http://localhost:7777
Upload files to local dir
>>> init logging <<<
2026-07-06 23:53:51,961 - SL - DEBUG - 5696 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
load config file /root/run.env
>>> URL: http://localhost:7777
Upload files to local dir
>>> init logging <<<
2026-07-06 23:53:53,516 - SL - DEBUG - 5730 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

The most direct liveness signal is the health endpoint.

**`/health` → `success`, HTTP 200.** Route `@app.route("/health")` at `server.py:213`, function `healthcheck()` at `server.py:214`, `return "success", 200` at `server.py:215`.

```
$ curl -sS -i http://localhost:7777/health
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 7
Set-Cookie: slapp=8031f068-bb2a-4dfc-b0ce-48a9403850c3.oV-BtEaOoXU7CCru7VdJLDSDz-U; Expires=Mon, 13-Jul-2026 23:54:38 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Mon, 06 Jul 2026 23:54:38 GMT

success
```

(The `slapp` value is a fresh **anonymous, unauthenticated** Flask session cookie minted for the request; it carries no login and no credentials.) `Content-Length: 7` is the exact 7-byte body `success`.

> **Discrepancy reported as observed.** The route decorator is at `server.py:213` and the `return "success", 200` statement is at `server.py:215` (line 214 is the `def healthcheck()` signature). The *observed behavior* — HTTP 200 with the exact 7-byte body `success` — is what matters and is confirmed above.

**`/live` → `live`.** `@monitor_bp.route("/live")` / `return "live"` at `app/monitor/views.py:10-12`.

```
$ curl -sS -i http://localhost:7777/live
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 4
Set-Cookie: slapp=fa54d5a3-5a70-479c-aeb6-282a5695bd6d.aNzEfR_C6yN07IU1_ojWOGLeDaM; Expires=Mon, 13-Jul-2026 23:54:38 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Mon, 06 Jul 2026 23:54:38 GMT

live
```

**`/git` → build SHA1.** `@monitor_bp.route("/git")` / `return SHA1` at `app/monitor/views.py:5-7`, sourced from `app.build_info.SHA1`.

```
$ curl -sS -i http://localhost:7777/git
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 3
Set-Cookie: slapp=2904d4fe-a123-4ea1-b3fd-5d0abf71853b.xaStGNS63Y8xj5XWQ6vCLZmmE3g; Expires=Mon, 13-Jul-2026 23:54:38 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Mon, 06 Jul 2026 23:54:38 GMT

dev
```

The value is the literal string `dev` (`Content-Length: 3`) — not empty. That is the image default: `app/build_info.py:1` defines `SHA1 = "dev"` (a real build would substitute the git SHA at package time).

**`/exception` → HTTP 500.** Confirms error routing is wired. `test_exception()` deliberately does `raise Exception("to make sure sentry works")` at `app/monitor/views.py:15-17`.

```
$ curl -sS -i http://localhost:7777/exception 2>&1 | head -1
HTTP/1.0 500 INTERNAL SERVER ERROR
```

**Dashboard reachability (a user can sign in and manage aliases).** `GET /` redirects unauthenticated users to the login page:

```
$ curl -sS -i http://localhost:7777/
HTTP/1.0 302 FOUND
Content-Type: text/html; charset=utf-8
Content-Length: 229
Location: http://localhost:7777/auth/login
Set-Cookie: slapp=f1f09ca3-4b4b-4c6c-ac81-8523616b9fa9.bTwxrqzXoDqy3ymNQUZF2FvXPn4; Expires=Mon, 13-Jul-2026 23:54:39 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Mon, 06 Jul 2026 23:54:39 GMT

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<title>Redirecting...</title>
<h1>Redirecting...</h1>
<p>You should be redirected automatically to target URL: <a href="/auth/login">/auth/login</a>.  If not click the link.
```

Logging in as the seeded admin then reaches the dashboard. The login form (`templates/auth/login.html`) posts `csrf_token`, `email`, `password`:

```
$ CSRF=$(curl -sS -c "$OBS/john.cookies" http://localhost:7777/auth/login \
         | grep -oE 'name="csrf_token"[^>]*value="[^"]+"' | grep -oE 'value="[^"]+"' \
         | head -1 | sed -E 's/value="([^"]+)"/\1/')

$ curl -sS -i -b "$OBS/john.cookies" -c "$OBS/john.cookies" -X POST http://localhost:7777/auth/login \
       --data-urlencode "csrf_token=$CSRF" --data-urlencode "email=john@wick.com" \
       --data-urlencode "password=password" | grep -iE '^HTTP|^Location:'
HTTP/1.0 302 FOUND
Location: http://localhost:7777/dashboard/

$ curl -sS -i -b "$OBS/john.cookies" http://localhost:7777/ | grep -iE '^HTTP|^Location:'
HTTP/1.0 302 FOUND
Location: http://localhost:7777/dashboard/

$ curl -sS -b "$OBS/john.cookies" -o /tmp/dash.html \
       -w 'HTTP %{http_code}  bytes %{size_download}\n' http://localhost:7777/dashboard/
HTTP 200  bytes 763450

$ grep -c 'Random Alias'     /tmp/dash.html   # -> 1
$ grep -c 'New Custom Alias' /tmp/dash.html   # -> 1
```

So once authenticated, `GET /` redirects to `/dashboard/`, which renders `templates/dashboard/index.html` as a 763,450-byte page containing the `Random Alias` and `New Custom Alias` controls — i.e., a user can sign in and reach alias management. (`password` here is the *documented* public seed credential from §1.2, not a real secret.)

### 2.2 Email handler (`email_handler.py`)

`main(port)` (`email_handler.py:2381`) builds `Controller(MailHandler(), hostname="0.0.0.0", port=port)` (`email_handler.py:2383`), calls `controller.start()` (`email_handler.py:2385`), and the `__main__` block logs the listening port (default **20381**). Captured startup, verbatim:

```
load config file /root/run.env
>>> URL: http://localhost:7777
Upload files to local dir
>>> init logging <<<
2026-07-06 23:53:52,000 - SL - DEBUG - 5694 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-06 23:53:52,530 - SL - INFO - 5694 - "/app/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-06 23:53:52,532 - SL - DEBUG - 5694 - "/app/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

The two banner lines are the liveness signal: `Listen for port 20381` from `email_handler.py:2403` and `Start mail controller 0.0.0.0 20381` from `email_handler.py:2386`. The socket is actually bound — checked via `/proc/net/tcp` since `ss` is not installed in the image (`4F9D` is `20381` in hex; local address `00000000:4F9D` is `0.0.0.0:20381`; state `0A` = `LISTEN`):

```
$ cat /proc/net/tcp | awk '{print $2, $4}' | grep -i 4F9D
00000000:4F9D 0A
```

Why does the process stay up with no traffic? The `__main__` block ends with a keep-alive loop `while True: time.sleep(2)` at `email_handler.py:2392-2393`; the aiosmtpd `Controller` runs its own asyncio event loop in a background thread and accepts inbound SMTP connections automatically.

### 2.3 Job runner (`job_runner.py`) — a timing question

The job runner's `__main__` block (`job_runner.py:329`) runs `while True:` (`job_runner.py:330`) inside `create_light_app().app_context()` (`job_runner.py:332`), iterates `get_jobs_to_run()` (`job_runner.py:307`), logs `Take job %s` (`job_runner.py:334`) for each ready row, then sleeps with `time.sleep(10)` (`job_runner.py:347`). So "up" means "polling the `job` table about every 10 seconds." Its startup is quiet by design (no per-cycle log):

```
load config file /root/run.env
>>> URL: http://localhost:7777
Upload files to local dir
>>> init logging <<<
2026-07-06 23:53:51,840 - SL - DEBUG - 5695 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

**Idle state.** From startup the runner logged **no** `Take job` lines for ~13 minutes, even though six onboarding jobs already existed in the table. Those rows are `state=0` (ready) but their `run_at` is dated days ahead, and `get_jobs_to_run()` only returns rows with `run_at IS NULL OR run_at <= now + 10 min` (`job_runner.py:312,323`), so it correctly returns nothing. `create_light_app()` does not log per cycle, so an idle loop is silent.

**Work state + the ~10 s cadence.** Real `Job` rows were enqueued through a genuine product action (the data-export button; details and the full state transition are in §4.2). Because the runner processes *all* ready rows in one cycle and then sleeps 10 s, enqueuing one job just after the previous one finishes makes each land in the *next* poll cycle — so the gaps between consecutive `Take job` lines measure the poll interval directly (all four lines from `job_runner.py:334`):

```
2026-07-07 00:09:53,842 - SL - DEBUG - 5695 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 26 send-user-report {'user_id': 1}>
2026-07-07 00:10:03,912 - SL - DEBUG - 5695 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 27 send-user-report {'user_id': 1}>
2026-07-07 00:10:13,984 - SL - DEBUG - 5695 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 28 send-user-report {'user_id': 1}>
2026-07-07 00:10:24,053 - SL - DEBUG - 5695 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 29 send-user-report {'user_id': 1}>
```

Consecutive gaps: **26→27 = 10.070 s, 27→28 = 10.072 s, 28→29 = 10.069 s**. The ~10 s interval is stable across three consecutive cycles, directly measuring `time.sleep(10)` (`job_runner.py:347`); the ~0.07 s excess is poll + processing overhead. Each job was picked up automatically on the next poll with no manual trigger.

**Not to be confused with the cron scheduler.** SimpleLogin has a *fourth* scheduler — `cron.py` (an argparse entry point) driven by **yacron** per `crontab.yml`, whose entries look like `command: python /code/cron.py -j <job>` for periodic tasks such as `stats`, `delete_logs`, and `check_hibp`. That is a separate mechanism from `job_runner.py`, which polls the `job` table every ~10 s. They are different components.

---

## 3. R2 — Basic user actions, and how the system shows each worked

This section walks the three fundamental flows a newcomer performs — **create an account**, **create an alias**, and **have that alias receive an email** — and shows each one confirmed on multiple independent channels: a **log line**, a **database row**, a **dashboard flash / UI change**, and (for mail) an **SMTP return code** and a copy in the mail sink. Every command below is reproducible; session-specific tokens are fetched exactly as shown for the login in §2.1 (a `csrf_token` is pulled from the form and posted back), and the created accounts/aliases are all removed afterward (§6).

### 3.1 Account creation (signup)

Registration is handled by `register()` at `app/auth/views/register.py:32` (route `@auth_bp.route("/register")` at `app/auth/views/register.py:31`). On a valid POST it logs `create user %s` at `app/auth/views/register.py:85`, then calls `send_activation_email(user, next_url)` at `app/auth/views/register.py:95`; that function (`app/auth/views/register.py:117`) creates the activation code with `ActivationCode.create(user_id=user.id, code=random_string(30))` at `app/auth/views/register.py:120`.

**Command + response (HTTP channel).** A successful signup returns 200 and renders the "waiting for activation" template:

```
$ curl -sS -i -b "$OBS/reg.cookies" -c "$OBS/reg.cookies" -X POST http://localhost:7777/auth/register \
       --data-urlencode "csrf_token=$CSRF" \
       --data-urlencode "email=blitzy-tmp-signup@example.com" \
       --data-urlencode "password=<chosen-password>"
HTTP/1.0 200 OK
    <h4>auth/register_waiting_activation.html</h4>
```

(The `<chosen-password>` is any password for this throwaway account; it is not a real secret and the account is deleted in §6. A 200 rendering `register_waiting_activation.html` means the signup was accepted.)

**Log channel** — the `create user` line at `app/auth/views/register.py:85`:

```
2026-07-06 23:56:46,195 - SL - DEBUG - 5730 - "/app/app/auth/views/register.py:85" - register() -  - create user blitzy-tmp-signup@example.com
2026-07-06 23:56:46,659 - SL - DEBUG - 5730 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/register ImmutableMultiDict([]) 200, takes 0.5060079097747803
```

**Database channel** — a new `User` row (`class User` at `app/models.py:336`) is created with `activated = f` (it cannot log in yet), plus one `ActivationCode` row (`class ActivationCode` at `app/models.py:1202`). The activation code value is a secret, so **only its metadata is selected — never the code itself**:

```
$ psql -c "SELECT id,email,activated,is_admin FROM users WHERE email='blitzy-tmp-signup@example.com';"
 id |             email             | activated | is_admin
----+-------------------------------+-----------+----------
  5 | blitzy-tmp-signup@example.com | f         | f
(1 row)

$ psql -c "SELECT id,user_id,length(code) AS code_len,created_at FROM activation_code WHERE user_id=5;"
 id | user_id | code_len |         created_at
----+---------+----------+----------------------------
  3 |       5 |       30 | 2026-07-06 23:56:46.486517
(1 row)
```

The `code` column is deliberately **not** selected; only `length(code)=30` is shown, matching `random_string(30)` at `app/auth/views/register.py:120`.

**Email channel** — the activation email is delivered to the mail sink (envelope shown from the sink's own log; the `.eml` body is on disk):

```
$ ls -1t $SINK | head -1        # newest message written by the sink
msg_001.eml
$ grep -iE '^(To|Subject):' "$SINK/msg_001.eml"
Subject: Just one more step to join SimpleLogin
To: blitzy-tmp-signup@example.com
```

So signup is confirmed four ways: HTTP 200 + waiting-activation page, the `create user` log line, the `activated=f` user row plus a 30-char activation code row, and the activation email arriving in the sink.

### 3.2 Account activation

A newly-registered user has `activated=f` and cannot sign in until the emailed code is consumed. Activation is handled by `activate()` at `app/auth/views/activate.py:17` (route at `app/auth/views/activate.py:13`), which looks the code up with `ActivationCode.get_by(code=code)` at `app/auth/views/activate.py:26`, sets `user.activated = True` at `app/auth/views/activate.py:49`, and flashes `Your account has been activated` (category `success`) at `app/auth/views/activate.py:56`.

Because the code is a secret, this document never prints it. Instead, activation is proven by **before/after database state** plus the **success flash** — the code is used at runtime but shown as `<redacted>`:

```
# BEFORE — the account is inactive and its single code is unused:
$ psql -tAc "SELECT activated FROM users WHERE id=6;"                    # -> f
$ psql -tAc "SELECT count(*) FROM activation_code WHERE user_id=6;"      # -> 1

# Consume the real code via the activation endpoint (code redacted here). The -c cookie
# jar matters: activate() calls login_user(user) (app/auth/views/activate.py:50), which
# sets the session cookie, and curl's cookie engine then resends it on the -L follow so
# the request reaches the @login_required /dashboard/ (HTTP 200) and renders the flash.
# Only -c (write) is used, not -b (read), so a newly-registered user's initial request
# never carries a stale prior session, which activate() would otherwise reject with
# HTTP 400 "You are already logged in" (app/auth/views/activate.py:18-22). The full response is
# saved once, then the status chain and flash line are grepped so each command reproduces:
$ curl -sS -i -L -c "$OBS/act.cookies" \
       "http://localhost:7777/auth/activate?code=<redacted>" > "$OBS/activate.out"
$ grep -iE '^HTTP/|^Location:' "$OBS/activate.out"
HTTP/1.0 302 FOUND
Location: http://localhost:7777/dashboard/
HTTP/1.0 200 OK
# the followed dashboard page (dashboard/index.html) renders the flash from app/auth/views/activate.py:56:
$ grep -o 'Your account has been activated' "$OBS/activate.out" | head -1
Your account has been activated

# AFTER — the account is active and the code has been consumed (deleted):
$ psql -tAc "SELECT activated FROM users WHERE id=6;"                    # -> t
$ psql -tAc "SELECT count(*) FROM activation_code WHERE user_id=6;"      # -> 0
```

The state transition is unambiguous: `users.activated` flips **f → t** and the `activation_code` count drops **1 → 0**, exactly as `activate()` does at `app/auth/views/activate.py:49` (and the code row is deleted on use). The `302 → /dashboard/` redirect plus the `Your account has been activated` flash are the UI confirmation. The account can now sign in.

**Observed discrepancy — cookie persistence matters.** Running the *same* activation URL **without** a cookie jar still activates the account server-side (the `activated f → t` and code `1 → 0` transitions above are byte-for-byte identical), but the redirect chain differs: `302 → /dashboard/`, then a second `302 → /auth/login?next=%2Fdashboard%2F%3F`, then a final `200` on the *login* page (`auth/login.html`), with the flash **not** rendered. That is because the session cookie set by `login_user` at `app/auth/views/activate.py:50` is discarded, so curl's `-L` follow to the `@login_required` `/dashboard/` is unauthenticated. Persisting the cookie — a browser, or `curl -c` as shown above — is what makes the dashboard-rendered flash reproduce; the server-side activation itself is unaffected either way.

### 3.3 Alias creation

Two creation paths were exercised as the seeded admin `john@wick.com`: a **random** alias and a **custom** alias, plus the custom-alias **validation edge**.

**Random alias.** Posting `form-name=create-random-email` to the dashboard index (`index()` at `app/dashboard/views/index.py`) calls `Alias.create_new_random(...)` at `app/dashboard/views/index.py:104`, logs `create new random alias %s for user %s` at `app/dashboard/views/index.py:110`, and flashes `Alias {alias.email} has been created` at `app/dashboard/views/index.py:111`:

```
$ curl -sS -i -b "$OBS/john.cookies" -c "$OBS/john.cookies" -X POST http://localhost:7777/dashboard/ \
       --data-urlencode "csrf_token=$CSRF" --data-urlencode "form-name=create-random-email"
HTTP/1.0 302 FOUND
Location: http://localhost:7777/dashboard/?highlight_alias_id=20&query=&sort=&filter=
```

```
# Log channel (app/dashboard/views/index.py:110):
2026-07-07 00:00:33,530 - SL - DEBUG - 5730 - "/app/app/dashboard/views/index.py:110" - index() -  - create new random alias <Alias 20 test_word970@sl.local> for user <User 1 John Wick john@wick.com>
```

```
# UI channel — success flash (app/dashboard/views/index.py:111):
Alias test_word970@sl.local has been created
```

```
# DB channel — new Alias row (class Alias at app/models.py:1469); the count grew by one:
$ psql -c "SELECT count(*) FROM alias WHERE user_id=1;"   # before=14  after=15
$ psql -c "SELECT id,email,user_id,mailbox_id,enabled FROM alias WHERE id=20;"
 id |         email         | user_id | mailbox_id | enabled
----+-----------------------+---------+------------+---------
 20 | test_word970@sl.local |       1 |          1 | t
(1 row)
```

The `highlight_alias_id=20` in the redirect, the `<Alias 20 test_word970@sl.local>` log line, the flash, and the DB row (id 20, enabled `t`) all agree.

**Custom alias.** Posting to `custom_alias()` (`app/dashboard/views/custom_alias.py:34`, route at `app/dashboard/views/custom_alias.py:30`) creates the alias with `Alias.create(...)` at `app/dashboard/views/custom_alias.py:139` and flashes `Alias {full_alias} has been created` at `app/dashboard/views/custom_alias.py:159`:

```
$ curl -sS -i -b "$OBS/john.cookies" -X POST http://localhost:7777/dashboard/custom_alias \
       --data-urlencode "csrf_token=$CSRF" --data-urlencode "prefix=blitzytmpcustom" \
       --data-urlencode "signed-alias-suffix=$SIGNED_SUFFIX" --data-urlencode "mailboxes=1"
HTTP/1.0 302 FOUND
Location: http://localhost:7777/dashboard/?highlight_alias_id=21
# flash (app/dashboard/views/custom_alias.py:159):
Alias blitzytmpcustom.word889@sl.local has been created
```

```
# DB channel — new Alias row (Alias.create at app/dashboard/views/custom_alias.py:139; model app/models.py:1469):
$ psql -c "SELECT id,email,user_id,mailbox_id,enabled FROM alias WHERE id=21;"
 id |              email               | user_id | mailbox_id | enabled
----+----------------------------------+---------+------------+---------
 21 | blitzytmpcustom.word889@sl.local |       1 |          1 | t
(1 row)
```

(The `signed-alias-suffix` value is a server-signed token issued in the custom-alias form; it is fetched from the page like the CSRF token. The resulting alias is `blitzytmpcustom.word889@sl.local`.)

**Validation edge (error branch).** Submitting an illegal prefix (`Bad Prefix!!`) hits the guard `if not check_alias_prefix(...)` and flashes the error at `app/dashboard/views/custom_alias.py:64-70`, redirecting back without creating anything:

```
$ curl -sS -i -b "$OBS/john.cookies" -X POST http://localhost:7777/dashboard/custom_alias \
       --data-urlencode "csrf_token=$CSRF" --data-urlencode "prefix=Bad Prefix!!" \
       --data-urlencode "signed-alias-suffix=$SIGNED_SUFFIX" --data-urlencode "mailboxes=1"
HTTP/1.0 302 FOUND
Location: http://localhost:7777/dashboard/custom_alias
# flash (app/dashboard/views/custom_alias.py:64-70):
Only lowercase letters, numbers, dashes (-), dots (.) and underscores (_) are currently supported for alias prefix. Cannot be more than 40 letters
```

The redirect goes back to `/dashboard/custom_alias` (not the index with a new `highlight_alias_id`), and no alias row is added — the happy path and the error path are both confirmed.

### 3.4 Inbound email reception (forward)

Now the core promise: mail sent to an alias is forwarded to the user's real mailbox. A message was sent to the random alias `test_word970@sl.local` via the email handler's real listener on `:20381` using `swaks`. The handler's `handle()` logs `Forward phase` at `email_handler.py:2202` and dispatches to `handle_forward()` at `email_handler.py:536`.

**SMTP channel** — the full `swaks` transcript; the server's final reply is the confirmation:

```
$ swaks --to test_word970@sl.local --from hey@google.com --server 127.0.0.1:20381 \
        --header "Subject: Blitzy R2.3 inbound test" --body "Hello from the inbound forward test."
=== Trying 127.0.0.1:20381...
=== Connected to 127.0.0.1.
<-  220 sl-app Python SMTP 1.4.2
 -> EHLO sl-app
<-  250-sl-app
<-  250-SIZE 33554432
<-  250-8BITMIME
<-  250-SMTPUTF8
<-  250 HELP
 -> MAIL FROM:<hey@google.com>
<-  250 OK
 -> RCPT TO:<test_word970@sl.local>
<-  250 OK
 -> DATA
<-  354 End data with <CR><LF>.<CR><LF>
 -> Date: Tue, 07 Jul 2026 00:02:22 +0000
 -> To: test_word970@sl.local
 -> From: hey@google.com
 -> Subject: Blitzy R2.3 inbound test
 -> Message-Id: <20260707000222.006369@sl-app>
 -> X-Mailer: swaks v20201014.0 jetmore.org/john/code/swaks/
 ->
 -> Hello from the inbound forward test.
 ->
 ->
 -> .
<-  250 Message accepted for delivery
 -> QUIT
<-  221 Bye
=== Connection closed with remote host.
```

The final `250 Message accepted for delivery` is the SMTP status **E200**, defined at `app/email/status.py:2` as `"250 Message accepted for delivery"`.

**Log channel** — the handler brackets every message between a `New message` line (`email_handler.py:2343`) and a `Finish … <<===` line (`email_handler.py:2367`). The **complete, unedited** handler log for this one message (message-id `0c882f95-6edd-4219-95e7-2a9a742c80cb`) is:

```
2026-07-07 00:02:22,071 - SL - DEBUG - 5694 - "/app/app/log.py:24" - set_message_id() -  - set message_id 0c882f95-6edd-4219-95e7-2a9a742c80cb
2026-07-07 00:02:22,071 - SL - DEBUG - 5694 - "/app/email_handler.py:2342" - _handle() - 0c882f95-6edd-4219-95e7-2a9a742c80cb - ====>=====>====>====>====>====>====>====>
2026-07-07 00:02:22,071 - SL - INFO - 5694 - "/app/email_handler.py:2343" - _handle() - 0c882f95-6edd-4219-95e7-2a9a742c80cb - New message, mail from hey@google.com, rctp tos ['test_word970@sl.local']
2026-07-07 00:02:22,072 - SL - INFO - 5694 - "/app/email_handler.py:1956" - handle() - 0c882f95-6edd-4219-95e7-2a9a742c80cb - Set CONTENT_TRANSFER_ENCODING
2026-07-07 00:02:22,073 - SL - DEBUG - 5694 - "/app/email_handler.py:1963" - handle() - 0c882f95-6edd-4219-95e7-2a9a742c80cb - Cannot parse Postfix queue ID from None None
2026-07-07 00:02:22,195 - SL - DEBUG - 5694 - "/app/email_handler.py:1980" - handle() - 0c882f95-6edd-4219-95e7-2a9a742c80cb - ==>> Handle mail_from:hey@google.com, rcpt_tos:['test_word970@sl.local'], header_from:hey@google.com, header_to:test_word970@sl.local, cc:None, reply-to:None, message_id:<20260707000222.006369@sl-app>, client_ip:None, headers:[('Date', 'Tue, 07 Jul 2026 00:02:22 +0000'), ('To', 'test_word970@sl.local'), ('From', 'hey@google.com'), ('Subject', 'Blitzy R2.3 inbound test'), ('Message-Id', '<20260707000222.006369@sl-app>'), ('X-Mailer', 'swaks v20201014.0 jetmore.org/john/code/swaks/'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-07 00:02:22,200 - SL - DEBUG - 5694 - "/app/email_handler.py:2202" - handle() - 0c882f95-6edd-4219-95e7-2a9a742c80cb - Forward phase hey@google.com(hey@google.com) -> test_word970@sl.local
2026-07-07 00:02:22,217 - SL - DEBUG - 5694 - "/app/email_handler.py:580" - handle_forward() - 0c882f95-6edd-4219-95e7-2a9a742c80cb - Create or get contact for from_header:hey@google.com
2026-07-07 00:02:22,244 - SL - DEBUG - 5694 - "/app/app/contact_utils.py:110" - create_contact() - 0c882f95-6edd-4219-95e7-2a9a742c80cb - Created contact <Contact 5 hey@google.com 20> for alias <Alias 20 test_word970@sl.local> with email hey@google.com invalid_email=False
2026-07-07 00:02:22,244 - SL - INFO - 5694 - "/app/app/handler/dmarc.py:33" - apply_dmarc_policy_for_forward_phase() - 0c882f95-6edd-4219-95e7-2a9a742c80cb - DMARC check disabled
2026-07-07 00:02:22,253 - SL - DEBUG - 5694 - "/app/email_handler.py:688" - forward_email_to_mailbox() - 0c882f95-6edd-4219-95e7-2a9a742c80cb - Forward <Contact 5 hey@google.com 20> -> <Alias 20 test_word970@sl.local> -> <Mailbox 1 john@wick.com>
2026-07-07 00:02:22,256 - SL - DEBUG - 5694 - "/app/email_handler.py:740" - forward_email_to_mailbox() - 0c882f95-6edd-4219-95e7-2a9a742c80cb - Create <EmailLog 7> for <Contact 5 hey@google.com 20>, <User 1 John Wick john@wick.com>, <Mailbox 1 john@wick.com>
2026-07-07 00:02:22,262 - SL - DEBUG - 5694 - "/app/email_handler.py:867" - forward_email_to_mailbox() - 0c882f95-6edd-4219-95e7-2a9a742c80cb - From header, new:"hey at google.com" <hey_at_google_com_rlrlui@sl.local>, old:hey@google.com
2026-07-07 00:02:22,262 - SL - DEBUG - 5694 - "/app/email_handler.py:316" - replace_header_when_forward() - 0c882f95-6edd-4219-95e7-2a9a742c80cb - Delete Cc header, old value None
2026-07-07 00:02:22,262 - SL - DEBUG - 5694 - "/app/email_handler.py:313" - replace_header_when_forward() - 0c882f95-6edd-4219-95e7-2a9a742c80cb - Replace To header, old: test_word970@sl.local, new: test_word970@sl.local
2026-07-07 00:02:22,262 - SL - INFO - 5694 - "/app/app/handler/unsubscribe_generator.py:36" - _generate_header_with_original_behaviour() - 0c882f95-6edd-4219-95e7-2a9a742c80cb - Email has no unsubscribe header
2026-07-07 00:02:22,271 - SL - DEBUG - 5694 - "/app/email_handler.py:893" - forward_email_to_mailbox() - 0c882f95-6edd-4219-95e7-2a9a742c80cb - Forward mail from hey@google.com to john@wick.com, mail_options:[], rcpt_options:[]
2026-07-07 00:02:22,278 - SL - DEBUG - 5694 - "/app/app/mail_sender.py:156" - _send_to_smtp() - 0c882f95-6edd-4219-95e7-2a9a742c80cb - getting a smtp connection takes seconds 0.0010619163513183594
2026-07-07 00:02:22,278 - SL - DEBUG - 5694 - "/app/app/mail_sender.py:163" - _send_to_smtp() - 0c882f95-6edd-4219-95e7-2a9a742c80cb - Sendmail mail_from:sl.lmycyibxfqqdemzxgmytems5.lbyz4wif6p7ku@sl.local, rcpt_to:john@wick.com, header_from:"hey at google.com" <hey_at_google_com_rlrlui@sl.local>, header_to:test_word970@sl.local, header_cc:None
2026-07-07 00:02:22,280 - SL - INFO - 5694 - "/app/email_handler.py:2367" - _handle() - 0c882f95-6edd-4219-95e7-2a9a742c80cb - Finish mail_from hey@google.com, rcpt_tos ['test_word970@sl.local'], takes 0.20916199684143066 seconds with return code '250 Message accepted for delivery'<<===
```

The `Forward phase` line (`email_handler.py:2202`) routes to `handle_forward()`; the two ORM-creation log lines are the database-write evidence: `Contact 5` (`class Contact` at `app/models.py:1863`) for alias 20 via `create_contact()` (`app/contact_utils.py:110`), and `EmailLog 7` (`class EmailLog` at `app/models.py:2060`) linking Contact 5 → User 1 → Mailbox 1 (`email_handler.py:740`). (Both rows were removed in the §6 cleanup.)

**Mailbox / sink channel** — the forwarded copy actually arrives at the mailbox `john@wick.com`, with the `From` rewritten to the reverse-alias `hey_at_google_com_rlrlui@sl.local` (`email_handler.py:867`) so a reply routes back through SimpleLogin:

```
[MTA-SINK 2026-07-07 00:02:22] #5 mail_from=sl.lmycyibxfqqdemzxgmytems5.lbyz4wif6p7ku@sl.local rcpt_tos=['john@wick.com'] From='"hey at google.com" <hey_at_google_com_rlrlui@sl.local>' To='test_word970@sl.local' Subject='Blitzy R2.3 inbound test' bytes=1040
```

So one inbound message is confirmed on four channels: SMTP `250` (E200), the bracketed handler log with Contact/EmailLog creation, and the rewritten copy delivered to the real mailbox.

### 3.5 On-the-fly alias auto-create (a message to a not-yet-existing alias)

SimpleLogin can create an alias *on receipt* when mail arrives at an address that matches a **directory** (a `prefix+directory@domain` pattern) or a registered custom domain. This is the auto-create path, and it is exercised through the same real `:20381` listener. In the seed, user `john` owns the directory `abcd`, so mail to `abcd+<anything>@sl.local` should auto-create an alias.

When `handle_forward()` finds the alias missing, it logs `alias %s not exist. Try to see if it can be created on the fly` at `email_handler.py:545` and calls `try_auto_create(alias_address)` at `email_handler.py:549` (function `try_auto_create` at `app/alias_utils.py:202`). That tries the custom-domain path first (`app/alias_utils.py:104` logs it is skipped — `sl.local` is not a custom domain), then the directory path via `try_auto_create_directory` (`app/alias_utils.py:227`), which logs the directory name at `app/alias_utils.py:169` and `create alias %s for directory %s` at `app/alias_utils.py:238`.

**SMTP channel:**

```
$ swaks --to abcd+blitzyauto@sl.local --from sender@partner.example --server 127.0.0.1:20381 \
        --header "Subject: Blitzy R2.3 auto-create test" --body "Testing on-the-fly directory auto-create."
=== Trying 127.0.0.1:20381...
=== Connected to 127.0.0.1.
<-  220 sl-app Python SMTP 1.4.2
 -> EHLO sl-app
<-  250-sl-app
<-  250-SIZE 33554432
<-  250-8BITMIME
<-  250-SMTPUTF8
<-  250 HELP
 -> MAIL FROM:<sender@partner.example>
<-  250 OK
 -> RCPT TO:<abcd+blitzyauto@sl.local>
<-  250 OK
 -> DATA
<-  354 End data with <CR><LF>.<CR><LF>
 -> Date: Tue, 07 Jul 2026 00:03:00 +0000
 -> To: abcd+blitzyauto@sl.local
 -> From: sender@partner.example
 -> Subject: Blitzy R2.3 auto-create test
 -> Message-Id: <20260707000300.006409@sl-app>
 -> X-Mailer: swaks v20201014.0 jetmore.org/john/code/swaks/
 ->
 -> Testing on-the-fly directory auto-create.
 ->
 ->
 -> .
<-  250 Message accepted for delivery
 -> QUIT
<-  221 Bye
=== Connection closed with remote host.
```

The final reply is again **E200** (`app/email/status.py:2`).

**Log channel** — the **complete, unedited** auto-create decision chain (message-id `51712cd3-85b1-4774-af82-f97dcf07005f`):

```
2026-07-07 00:03:00,147 - SL - DEBUG - 5694 - "/app/app/log.py:24" - set_message_id() -  - set message_id 51712cd3-85b1-4774-af82-f97dcf07005f
2026-07-07 00:03:00,147 - SL - DEBUG - 5694 - "/app/email_handler.py:2342" - _handle() - 51712cd3-85b1-4774-af82-f97dcf07005f - ====>=====>====>====>====>====>====>====>
2026-07-07 00:03:00,147 - SL - INFO - 5694 - "/app/email_handler.py:2343" - _handle() - 51712cd3-85b1-4774-af82-f97dcf07005f - New message, mail from sender@partner.example, rctp tos ['abcd+blitzyauto@sl.local']
2026-07-07 00:03:00,148 - SL - INFO - 5694 - "/app/email_handler.py:1956" - handle() - 51712cd3-85b1-4774-af82-f97dcf07005f - Set CONTENT_TRANSFER_ENCODING
2026-07-07 00:03:00,149 - SL - DEBUG - 5694 - "/app/email_handler.py:1963" - handle() - 51712cd3-85b1-4774-af82-f97dcf07005f - Cannot parse Postfix queue ID from None None
2026-07-07 00:03:00,150 - SL - DEBUG - 5694 - "/app/email_handler.py:1980" - handle() - 51712cd3-85b1-4774-af82-f97dcf07005f - ==>> Handle mail_from:sender@partner.example, rcpt_tos:['abcd+blitzyauto@sl.local'], header_from:sender@partner.example, header_to:abcd+blitzyauto@sl.local, cc:None, reply-to:None, message_id:<20260707000300.006409@sl-app>, client_ip:None, headers:[('Date', 'Tue, 07 Jul 2026 00:03:00 +0000'), ('To', 'abcd+blitzyauto@sl.local'), ('From', 'sender@partner.example'), ('Subject', 'Blitzy R2.3 auto-create test'), ('Message-Id', '<20260707000300.006409@sl-app>'), ('X-Mailer', 'swaks v20201014.0 jetmore.org/john/code/swaks/'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-07 00:03:00,154 - SL - DEBUG - 5694 - "/app/email_handler.py:2202" - handle() - 51712cd3-85b1-4774-af82-f97dcf07005f - Forward phase sender@partner.example(sender@partner.example) -> abcd+blitzyauto@sl.local
2026-07-07 00:03:00,160 - SL - DEBUG - 5694 - "/app/email_handler.py:545" - handle_forward() - 51712cd3-85b1-4774-af82-f97dcf07005f - alias abcd+blitzyauto@sl.local not exist. Try to see if it can be created on the fly
2026-07-07 00:03:00,166 - SL - INFO - 5694 - "/app/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - 51712cd3-85b1-4774-af82-f97dcf07005f - Cannot auto-create custom domain alias for abcd+blitzyauto@sl.local because there's no custom domain for sl.local
2026-07-07 00:03:00,166 - SL - DEBUG - 5694 - "/app/app/alias_utils.py:169" - check_if_alias_can_be_auto_created_for_a_directory() - 51712cd3-85b1-4774-af82-f97dcf07005f - directory_name abcd
2026-07-07 00:03:00,174 - SL - DEBUG - 5694 - "/app/app/alias_utils.py:238" - try_auto_create_directory() - 51712cd3-85b1-4774-af82-f97dcf07005f - create alias abcd+blitzyauto@sl.local for directory <Directory abcd>
2026-07-07 00:03:00,182 - SL - INFO - 5694 - "/app/app/events/event_dispatcher.py:58" - send_event() - 51712cd3-85b1-4774-af82-f97dcf07005f - Not sending events because webhook is disabled
2026-07-07 00:03:00,190 - SL - DEBUG - 5694 - "/app/email_handler.py:580" - handle_forward() - 51712cd3-85b1-4774-af82-f97dcf07005f - Create or get contact for from_header:sender@partner.example
2026-07-07 00:03:00,225 - SL - DEBUG - 5694 - "/app/app/contact_utils.py:110" - create_contact() - 51712cd3-85b1-4774-af82-f97dcf07005f - Created contact <Contact 6 sender@partner.example 22> for alias <Alias 22 abcd+blitzyauto@sl.local> with email sender@partner.example invalid_email=False
2026-07-07 00:03:00,225 - SL - INFO - 5694 - "/app/app/handler/dmarc.py:33" - apply_dmarc_policy_for_forward_phase() - 51712cd3-85b1-4774-af82-f97dcf07005f - DMARC check disabled
2026-07-07 00:03:00,231 - SL - DEBUG - 5694 - "/app/email_handler.py:688" - forward_email_to_mailbox() - 51712cd3-85b1-4774-af82-f97dcf07005f - Forward <Contact 6 sender@partner.example 22> -> <Alias 22 abcd+blitzyauto@sl.local> -> <Mailbox 1 john@wick.com>
2026-07-07 00:03:00,233 - SL - DEBUG - 5694 - "/app/email_handler.py:740" - forward_email_to_mailbox() - 51712cd3-85b1-4774-af82-f97dcf07005f - Create <EmailLog 8> for <Contact 6 sender@partner.example 22>, <User 1 John Wick john@wick.com>, <Mailbox 1 john@wick.com>
2026-07-07 00:03:00,238 - SL - DEBUG - 5694 - "/app/email_handler.py:867" - forward_email_to_mailbox() - 51712cd3-85b1-4774-af82-f97dcf07005f - From header, new:"sender at partner.example" <sender_at_partner_example_tawiuhotgf@sl.local>, old:sender@partner.example
2026-07-07 00:03:00,238 - SL - DEBUG - 5694 - "/app/email_handler.py:316" - replace_header_when_forward() - 51712cd3-85b1-4774-af82-f97dcf07005f - Delete Cc header, old value None
2026-07-07 00:03:00,239 - SL - DEBUG - 5694 - "/app/email_handler.py:313" - replace_header_when_forward() - 51712cd3-85b1-4774-af82-f97dcf07005f - Replace To header, old: abcd+blitzyauto@sl.local, new: abcd+blitzyauto@sl.local
2026-07-07 00:03:00,239 - SL - INFO - 5694 - "/app/app/handler/unsubscribe_generator.py:36" - _generate_header_with_original_behaviour() - 51712cd3-85b1-4774-af82-f97dcf07005f - Email has no unsubscribe header
2026-07-07 00:03:00,247 - SL - DEBUG - 5694 - "/app/email_handler.py:893" - forward_email_to_mailbox() - 51712cd3-85b1-4774-af82-f97dcf07005f - Forward mail from sender@partner.example to john@wick.com, mail_options:[], rcpt_options:[]
2026-07-07 00:03:00,249 - SL - DEBUG - 5694 - "/app/app/mail_sender.py:156" - _send_to_smtp() - 51712cd3-85b1-4774-af82-f97dcf07005f - getting a smtp connection takes seconds 0.0012810230255126953
2026-07-07 00:03:00,249 - SL - DEBUG - 5694 - "/app/app/mail_sender.py:163" - _send_to_smtp() - 51712cd3-85b1-4774-af82-f97dcf07005f - Sendmail mail_from:sl.lmycyibyfqqdemzxgmytem25.ia2vr7bs2nxy4@sl.local, rcpt_to:john@wick.com, header_from:"sender at partner.example" <sender_at_partner_example_tawiuhotgf@sl.local>, header_to:abcd+blitzyauto@sl.local, header_cc:None
2026-07-07 00:03:00,251 - SL - INFO - 5694 - "/app/email_handler.py:2367" - _handle() - 51712cd3-85b1-4774-af82-f97dcf07005f - Finish mail_from sender@partner.example, rcpt_tos ['abcd+blitzyauto@sl.local'], takes 0.10365486145019531 seconds with return code '250 Message accepted for delivery'<<===
```

**DB channel** — the newly auto-created alias is owned by the directory's user and carries a `directory_id`, which a normal (non-auto) alias does not:

```
$ psql -c "SELECT id,email,user_id,directory_id FROM alias WHERE id=22;"
 id |            email            | user_id | directory_id
----+-----------------------------+---------+--------------
 22 | abcd+blitzyauto@sl.local    |       1 |            1
(1 row)
```

**Mailbox / sink channel** — the message was also forwarded on to `john@wick.com` (sink `#6`), so auto-create both creates the alias *and* delivers the triggering message:

```
[MTA-SINK 2026-07-07 00:03:00] #6 mail_from=sl.lmycyibyfqqdemzxgmytem25.ia2vr7bs2nxy4@sl.local rcpt_tos=['john@wick.com'] From='"sender at partner.example" <sender_at_partner_example_tawiuhotgf@sl.local>' To='abcd+blitzyauto@sl.local' Subject='Blitzy R2.3 auto-create test' bytes=1092
```

So an email to a **non-existent but eligible** address auto-creates `Alias 22` (with `directory_id=1`, the distinguishing mark of a directory auto-create), creates `Contact 6` and `EmailLog 8`, forwards the message to the mailbox, and returns E200 — every step evidenced by the log chain, the DB row, and the sink copy. (Alias 22, Contact 6, and EmailLog 8 were removed in §6.)


---

## 4. R3 — Do the background components come online on their own?

The question is whether, while the app is active, the **email handler** and **job runner** automatically support email activity and data handling in the background — and what runtime behavior proves it across different situations. The short answer from the code and the runtime: **the three processes are fully independent — the web server never starts the other two — and once each is running, it picks up its own work automatically** (the email handler on every inbound SMTP connection, the job runner on every ~10 s poll). This section proves both halves.

### 4.1 The web server does not spawn the others (they are independent processes)

`create_app()` (`server.py:139`) builds only the Flask app. It contains no reference to the sibling processes and no process-spawning primitive. A search of `server.py` for the sibling entry points and every spawn mechanism returns nothing:

```
$ grep -nE "email_handler|job_runner|subprocess|Popen|multiprocessing|Thread|Controller" server.py
$ echo "exit=$?"
exit=1
```

`exit=1` means **zero matches** — `server.py` starts no thread, no subprocess, and no aiosmtpd `Controller`. Nor is either sibling imported anywhere the app loads:

```
$ grep -rn "import email_handler\|import job_runner" server.py wsgi.py app/ | wc -l
0
```

So the three are started independently (as in §2), which is exactly how SimpleLogin is deployed — three separate services. The decisive runtime proof is that **the email handler keeps working after the web server is killed.** Using the *exact* PIDs captured at spawn in §2 (parent `5696` + Werkzeug reloader child `5730`) — not a broad `pkill`/`pgrep` — the web server was stopped and an email was sent:

```
# [1] baseline — web app up:
$ curl -s -o /dev/null -w "HTTP %{http_code}\n" http://localhost:7777/health
HTTP 200

# [2] stop ONLY the two captured web-server PIDs (never pkill/pgrep):
$ kill 5730 5696

# [3] web app is now down (connection refused):
$ curl -s -o /dev/null -w "http_code=%{http_code}\n" --max-time 4 http://localhost:7777/health ; echo "curl_rc=$?"
http_code=000
curl_rc=7

# [4] the email handler (5694) and job runner (5695) are still alive:
$ ps -o pid,cmd -p 5694,5695
    PID CMD
   5694 python -u email_handler.py
   5695 python -u job_runner.py

# [5] send an inbound email WHILE THE WEB APP IS DOWN:
$ swaks --to test_word970@sl.local --from hey@google.com --server 127.0.0.1:20381 \
        --header "Subject: webdown-test" --body "sent while web app is down"
=== Trying 127.0.0.1:20381...
=== Connected to 127.0.0.1.
<-  220 sl-app Python SMTP 1.4.2
 -> EHLO sl-app
<-  250-sl-app
<-  250-SIZE 33554432
<-  250-8BITMIME
<-  250-SMTPUTF8
<-  250 HELP
 -> MAIL FROM:<hey@google.com>
<-  250 OK
 -> RCPT TO:<test_word970@sl.local>
<-  250 OK
 -> DATA
<-  354 End data with <CR><LF>.<CR><LF>
 -> Date: Tue, 07 Jul 2026 00:11:45 +0000
 -> To: test_word970@sl.local
 -> From: hey@google.com
 -> Subject: webdown-test
 -> Message-Id: <20260707001145.007121@sl-app>
 -> X-Mailer: swaks v20201014.0 jetmore.org/john/code/swaks/
 ->
 -> sent while web app is down
 ->
 ->
 -> .
<-  250 Message accepted for delivery
 -> QUIT
<-  221 Bye
=== Connection closed with remote host.
```

The handler accepted and processed it (message-id `ce8b0d8d-67af-4768-a4be-37f6c35d0477`):

```
2026-07-07 00:11:45,871 - SL - INFO - 5694 - "/app/email_handler.py:2343" - _handle() - ce8b0d8d-67af-4768-a4be-37f6c35d0477 - New message, mail from hey@google.com, rctp tos ['test_word970@sl.local']
2026-07-07 00:11:45,913 - SL - INFO - 5694 - "/app/email_handler.py:2367" - _handle() - ce8b0d8d-67af-4768-a4be-37f6c35d0477 - Finish mail_from hey@google.com, rcpt_tos ['test_word970@sl.local'], takes 0.04257011413574219 seconds with return code '250 Message accepted for delivery'<<===
```

…and it wrote a new `EmailLog` row **while the web app was down**, proving it operates independently:

```
$ psql -c "SELECT id,user_id,contact_id,is_reply,bounced FROM email_log WHERE id>8 ORDER BY id;"
 id | user_id | contact_id | is_reply | bounced
----+---------+------------+----------+---------
  9 |       1 |          5 | f        | f
(1 row)
```

Finally the web server was restarted through its real entry point (the canonical venv `python`), coming back on fresh PIDs, and `/health` returned to 200:

```
$ setsid nohup env CONFIG=$CONFIG python -u server.py > "$OBS/server_restart.log" 2>&1 &
new server parent PID=7178  reloader child PID=7190
$ curl -s -w "HTTP %{http_code}\n" http://localhost:7777/health
successHTTP 200
```

**Conclusion (R3, part 1):** the components are independent processes; the email handler continued to accept and record mail with the web server completely down. (`EmailLog 9` was removed in §6.)

### 4.2 The job runner picks up queued work automatically (on its own ~10 s poll)

§2.3 established that the runner loops every ~10 s. Here we show it **automatically picks up new work with no manual trigger**. Real jobs were enqueued by a genuine product action — the dashboard **"Export data"** button, which is behind *sudo mode* (`POST /dashboard/enter_sudo` first, then `POST /dashboard/account_setting` with `form-name=send-full-user-report`), inserting a `send-user-report` row into the `job` table. The runner then advances the job through its state machine `JobState` (`ready=0`, `taken=1`, `done=2`) at `app/models.py:253`.

Polling the job's `state` column shows it move from **ready (0)** to **done (2)** on its own within one poll cycle:

```
$ JID=25
$ for i in $(seq 1 40); do \
     s=$(psql -tAc "SELECT state FROM job WHERE id=$JID"); \
     echo "$(date +%H:%M:%S.%3N) state=$s"; [ "$s" = 2 ] && break; sleep 0.4; done
00:07:31.742 state=0
00:07:33.961 state=2
```

The `taken=1` state is transient: within a single loop iteration the runner sets `state = JobState.taken.value` (`job_runner.py:339`), runs `process_job(job)` (`job_runner.py:342`, function at `job_runner.py:188`), then sets `state = JobState.done.value` (`job_runner.py:344`) — all in well under one 0.4 s sample, so the poll observes `0 → 2`. The pickup itself is logged by `Take job %s` at `job_runner.py:334`:

```
2026-07-07 00:07:33,613 - SL - DEBUG - 5695 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 25 send-user-report {'user_id': 1}>
```

That the pickup is **automatic and periodic** (not triggered by anything the user did after enqueuing) is shown by the ~10 s spacing of consecutive `Take job` lines already measured in §2.3 (`26→27 = 10.070 s`, `27→28 = 10.072 s`, `28→29 = 10.069 s`), which is exactly `time.sleep(10)` at `job_runner.py:347`. So: enqueue work → the runner finds and completes it on the next poll, entirely on its own. (Jobs 25–29 were removed in §6.)

### 4.3 The email handler handles every situation automatically

"Across different situations" means more than the happy-path forward. The handler's dispatcher `handle()` routes each message by phase, and the real listener processes each variant automatically as it arrives. The forward (happy) path and the on-the-fly auto-create path were shown in §3.4 and §3.5 (both `E200`). Here are the remaining branches — **reply**, **forward-phase bounce**, **reply-phase bounce**, and the **bounce guard** — each returning its own distinct SMTP code from `app/email/status.py`.

**(a) Reply — mailbox → contact via the reverse-alias (`handle_reply`, `email_handler.py:966`).** Sending *from* the mailbox `john@wick.com` *to* the reverse-alias `hey_at_google_com_rlrlui@sl.local` makes SimpleLogin send outward as the alias:

```
$ swaks --from john@wick.com --to hey_at_google_com_rlrlui@sl.local --server 127.0.0.1:20381 \
        --header "Subject: my reply" --body "replying to the sender via the reverse-alias"
=== Trying 127.0.0.1:20381...
=== Connected to 127.0.0.1.
<-  220 sl-app Python SMTP 1.4.2
 -> EHLO sl-app
<-  250-sl-app
<-  250-SIZE 33554432
<-  250-8BITMIME
<-  250-SMTPUTF8
<-  250 HELP
 -> MAIL FROM:<john@wick.com>
<-  250 OK
 -> RCPT TO:<hey_at_google_com_rlrlui@sl.local>
<-  250 OK
 -> DATA
<-  354 End data with <CR><LF>.<CR><LF>
 -> Date: Tue, 07 Jul 2026 00:13:19 +0000
 -> To: hey_at_google_com_rlrlui@sl.local
 -> From: john@wick.com
 -> Subject: my reply
 -> Message-Id: <20260707001319.007238@sl-app>
 -> X-Mailer: swaks v20201014.0 jetmore.org/john/code/swaks/
 ->
 -> replying to the sender via the reverse-alias
 ->
 ->
 -> .
<-  250 Message accepted for delivery
 -> QUIT
<-  221 Bye
=== Connection closed with remote host.
```

The log shows the `Reply phase` dispatch (`email_handler.py:2196`) and a **reply** `EmailLog` created in `handle_reply()` (`email_handler.py:1051`), sending out *as the alias* `test_word970@sl.local`:

```
2026-07-07 00:13:19,260 - SL - INFO - 5694 - "/app/email_handler.py:2343" - _handle() - 9de5314d-2b4d-45bd-b7a6-fad88a9b1b13 - New message, mail from john@wick.com, rctp tos ['hey_at_google_com_rlrlui@sl.local']
2026-07-07 00:13:19,267 - SL - DEBUG - 5694 - "/app/email_handler.py:2196" - handle() - 9de5314d-2b4d-45bd-b7a6-fad88a9b1b13 - Reply phase john@wick.com(john@wick.com) -> hey_at_google_com_rlrlui@sl.local
2026-07-07 00:13:19,277 - SL - DEBUG - 5694 - "/app/email_handler.py:1051" - handle_reply() - 9de5314d-2b4d-45bd-b7a6-fad88a9b1b13 - Create <EmailLog 10> for <Contact 5 hey@google.com 20>, <User 1 John Wick john@wick.com>, <Mailbox 1 john@wick.com>
2026-07-07 00:13:19,293 - SL - DEBUG - 5694 - "/app/email_handler.py:1212" - handle_reply() - 9de5314d-2b4d-45bd-b7a6-fad88a9b1b13 - send email from test_word970@sl.local to hey@google.com, mail_options:[],rcpt_options:[]
2026-07-07 00:13:19,307 - SL - INFO - 5694 - "/app/email_handler.py:2367" - _handle() - 9de5314d-2b4d-45bd-b7a6-fad88a9b1b13 - Finish mail_from john@wick.com, rcpt_tos ['hey_at_google_com_rlrlui@sl.local'], takes 0.04703402519226074 seconds with return code '250 Message accepted for delivery'<<===
```

The new row has `is_reply = t` (a forward has `f`), confirming the reply branch:

```
$ psql -c "SELECT id,user_id,contact_id,is_reply,bounced FROM email_log WHERE id=10;"
 id | user_id | contact_id | is_reply | bounced
----+---------+------------+----------+---------
 10 |       1 |          5 | t        | f
(1 row)
```

Return code: `250 Message accepted for delivery` = **E200** (`app/email/status.py:2`).

**Bounce detection (the gate for the next two cases).** A message is treated as a bounce only when it is a real DSN: `is_bounce()` (`email_handler.py:1813-1818`) requires `envelope.mail_from == "<>"` **and** `Content-Type: multipart/report`. To emit a genuine null sender, `swaks --from "<>"` is used (which sends `MAIL FROM:<>`); the DSN body is supplied with `--data @<file>`. When a DSN arrives at a VERP address, `handle_bounce()` (`email_handler.py:1851`) routes it by the phase of the referenced `EmailLog`.

**(b) Forward-phase bounce → `E211` (`handle_bounce_forward_phase`, `email_handler.py:1432`).** After a fresh forward created `EmailLog 11` with forward return-path VERP `sl.lmycyibrgewcamrtg4ztcmzylu.njc6kmyymtkm6@sl.local` (built by `generate_verp_email(VerpType.bounce_forward, …)` at `email_handler.py:905`), a DSN with a null sender was sent to that VERP. The exact DSN body (the `--data` input) was:

```
From: Mail Delivery Subsystem <MAILER-DAEMON@sl.local>
To: sl.lmycyibrgewcamrtg4ztcmzylu.njc6kmyymtkm6@sl.local
Subject: Delivery Status Notification (Failure)
MIME-Version: 1.0
Content-Type: multipart/report; report-type=delivery-status; boundary="fb99"

--fb99
Content-Type: text/plain; charset=us-ascii

Delivery to the following recipient failed permanently:
     john@wick.com

--fb99
Content-Type: message/delivery-status

Reporting-MTA: dns; sl.local
Final-Recipient: rfc822; john@wick.com
Action: failed
Status: 5.2.2
Diagnostic-Code: smtp; 552 5.2.2 The mailbox is over quota.

--fb99
Content-Type: message/rfc822

From: fwdbounce_at_partner.example_xxxx@sl.local
To: john@wick.com
Subject: fwd-bounce-setup

to be bounced by the mailbox

--fb99--
```

The SMTP exchange, captured from the server's `EHLO`-response onward (the connection-open, `220` banner, and `-> EHLO sl-app` lines are identical to the complete transcripts shown above):

```
$ swaks --from "<>" --to sl.lmycyibrgewcamrtg4ztcmzylu.njc6kmyymtkm6@sl.local \
        --server 127.0.0.1:20381 --data @"$OBS/dsn_forward.eml"
<-  250-sl-app
<-  250-SIZE 33554432
<-  250-8BITMIME
<-  250-SMTPUTF8
<-  250 HELP
 -> MAIL FROM:<>
<-  250 OK
 -> RCPT TO:<sl.lmycyibrgewcamrtg4ztcmzylu.njc6kmyymtkm6@sl.local>
<-  250 OK
 -> DATA
<-  354 End data with <CR><LF>.<CR><LF>
<-  250 SL E211 Bounce Forward phase handled
 -> QUIT
<-  221 Bye
```

```
2026-07-07 00:19:14,613 - SL - INFO - 5694 - "/app/email_handler.py:2343" - _handle() - 2c248981-266b-44bc-a27b-e3a70d7240df - New message, mail from <>, rctp tos ['sl.lmycyibrgewcamrtg4ztcmzylu.njc6kmyymtkm6@sl.local']
2026-07-07 00:19:14,622 - SL - DEBUG - 5694 - "/app/email_handler.py:1862" - handle_bounce() - 2c248981-266b-44bc-a27b-e3a70d7240df - handle bounce for <EmailLog 11>, phase=forward, contact=<Contact 7 fwdbounce@partner.example 20>, alias=<Alias 20 test_word970@sl.local>
2026-07-07 00:19:14,625 - SL - DEBUG - 5694 - "/app/email_handler.py:1456" - handle_bounce_forward_phase() - 2c248981-266b-44bc-a27b-e3a70d7240df - Handle forward bounce <Contact 7 fwdbounce@partner.example 20> -> <Alias 20 test_word970@sl.local> -> <Mailbox 1 john@wick.com>. <EmailLog 11>
2026-07-07 00:19:14,632 - SL - DEBUG - 5694 - "/app/email_handler.py:1491" - handle_bounce_forward_phase() - 2c248981-266b-44bc-a27b-e3a70d7240df - Create refused email <Refused Email 4 None 2026-07-14T00:19:14.631909+00:00>
2026-07-07 00:19:14,686 - SL - INFO - 5694 - "/app/email_handler.py:2367" - _handle() - 2c248981-266b-44bc-a27b-e3a70d7240df - Finish mail_from <>, rcpt_tos ['sl.lmycyibrgewcamrtg4ztcmzylu.njc6kmyymtkm6@sl.local'], takes 0.07353615760803223 seconds with return code '250 SL E211 Bounce Forward phase handled'<<===
```

The DB effect: `EmailLog 11` is marked `bounced = t`, and a new `Bounce` row is created whose `email` is the **mailbox** `john@wick.com` (the forward phase records the mailbox that rejected the mail):

```
$ psql -c "SELECT id,is_reply,bounced,bounced_mailbox_id FROM email_log WHERE id=11;"
 id | is_reply | bounced | bounced_mailbox_id
----+----------+---------+--------------------
 11 | f        | t       |                  1
(1 row)
$ psql -c "SELECT id,email FROM bounce ORDER BY id DESC LIMIT 1;"
 id |     email
----+---------------
  3 | john@wick.com
(1 row)
```

Return code: **E211** = `250 SL E211 Bounce Forward phase handled` (`app/email/status.py:19`).

**(c) Reply-phase bounce → `E212` (`handle_bounce_reply_phase`, `email_handler.py:1595`).** This is the counterpart branch: a DSN for a *reply* `EmailLog`. The reply above (`EmailLog 10`) was sent outward with a reply return-path VERP `sl.lmysyibrgawcamrtg4ztcmztlu.2fojfpmqovaju@sl.local` (built by `generate_verp_email(VerpType.bounce_reply, …)` at `email_handler.py:1225`). A DSN with a null sender was sent to that VERP — the **complete** transcript including the full DSN body:

```
$ swaks --from "<>" --to sl.lmysyibrgawcamrtg4ztcmztlu.2fojfpmqovaju@sl.local \
        --server 127.0.0.1:20381 --data @"$OBS/dsn_reply.eml"
=== Trying 127.0.0.1:20381...
=== Connected to 127.0.0.1.
<-  220 sl-app Python SMTP 1.4.2
 -> EHLO sl-app
<-  250-sl-app
<-  250-SIZE 33554432
<-  250-8BITMIME
<-  250-SMTPUTF8
<-  250 HELP
 -> MAIL FROM:<>
<-  250 OK
 -> RCPT TO:<sl.lmysyibrgawcamrtg4ztcmztlu.2fojfpmqovaju@sl.local>
<-  250 OK
 -> DATA
<-  354 End data with <CR><LF>.<CR><LF>
 -> From: Mail Delivery Subsystem <MAILER-DAEMON@google.com>
 -> To: sl.lmysyibrgawcamrtg4ztcmztlu.2fojfpmqovaju@sl.local
 -> Subject: Delivery Status Notification (Failure)
 -> MIME-Version: 1.0
 -> Content-Type: multipart/report; report-type=delivery-status; boundary="b1b1b1"
 ->
 -> --b1b1b1
 -> Content-Type: text/plain; charset=us-ascii
 ->
 -> Delivery to the following recipient failed permanently:
 ->      hey@google.com
 ->
 -> --b1b1b1
 -> Content-Type: message/delivery-status
 ->
 -> Reporting-MTA: dns; google.com
 -> Arrival-Date: Tue, 07 Jul 2026 00:13:19 +0000
 ->
 -> Final-Recipient: rfc822; hey@google.com
 -> Action: failed
 -> Status: 5.1.1
 -> Diagnostic-Code: smtp; 550-5.1.1 The email account that you tried to reach does not exist.
 ->
 -> --b1b1b1
 -> Content-Type: message/rfc822
 ->
 -> From: test_word970@sl.local
 -> To: hey@google.com
 -> Subject: my reply
 -> Message-ID: <178338319928.5694.12110958476751986382.10@sl.local>
 ->
 -> replying to the sender via the reverse-alias
 ->
 -> --b1b1b1--
 ->
 -> .
<-  250 SL E212 Bounce Reply phase handled
 -> QUIT
<-  221 Bye
=== Connection closed with remote host.
```

The log shows `handle_bounce()` routing to the **reply** phase (`email_handler.py:1862` → `handle_bounce_reply_phase` at `email_handler.py:1605`), creating a refused email and informing the user:

```
2026-07-07 00:16:59,201 - SL - INFO - 5694 - "/app/email_handler.py:2343" - _handle() - ae3efa02-73a2-448e-9a6d-7bd60215bbb8 - New message, mail from <>, rctp tos ['sl.lmysyibrgawcamrtg4ztcmztlu.2fojfpmqovaju@sl.local']
2026-07-07 00:16:59,210 - SL - DEBUG - 5694 - "/app/email_handler.py:1862" - handle_bounce() - ae3efa02-73a2-448e-9a6d-7bd60215bbb8 - handle bounce for <EmailLog 10>, phase=reply, contact=<Contact 5 hey@google.com 20>, alias=<Alias 20 test_word970@sl.local>
2026-07-07 00:16:59,211 - SL - DEBUG - 5694 - "/app/email_handler.py:1605" - handle_bounce_reply_phase() - ae3efa02-73a2-448e-9a6d-7bd60215bbb8 - Handle reply bounce <Mailbox 1 john@wick.com> -> <Alias 20 test_word970@sl.local> -> <Contact 5 hey@google.com 20>.<EmailLog 10>
2026-07-07 00:16:59,219 - SL - DEBUG - 5694 - "/app/email_handler.py:1640" - handle_bounce_reply_phase() - ae3efa02-73a2-448e-9a6d-7bd60215bbb8 - Create refused email <Refused Email 3 refused-emails/da648225-1fe0-4eae-93dd-5e91832bd8af.eml 2026-07-14T00:16:59.217663+00:00>
2026-07-07 00:16:59,224 - SL - DEBUG - 5694 - "/app/email_handler.py:1651" - handle_bounce_reply_phase() - ae3efa02-73a2-448e-9a6d-7bd60215bbb8 - Inform user <User 1 John Wick john@wick.com> about bounced email sent by <Alias 20 test_word970@sl.local> to <Contact 5 hey@google.com 20>
2026-07-07 00:16:59,268 - SL - INFO - 5694 - "/app/email_handler.py:2367" - _handle() - ae3efa02-73a2-448e-9a6d-7bd60215bbb8 - Finish mail_from <>, rcpt_tos ['sl.lmysyibrgawcamrtg4ztcmztlu.2fojfpmqovaju@sl.local'], takes 0.06680798530578613 seconds with return code '250 SL E212 Bounce Reply phase handled'<<===
```

The DB effect distinguishes the reply phase from the forward phase: `EmailLog 10` becomes `bounced = t` with `refused_email_id` set, and the new `Bounce` row's `email` is the **contact** `hey@google.com` (not the mailbox), plus a user `Notification` is created:

```
$ psql -c "SELECT id,is_reply,bounced,bounced_mailbox_id,refused_email_id FROM email_log WHERE id=10;"
 id | is_reply | bounced | bounced_mailbox_id | refused_email_id
----+----------+---------+--------------------+------------------
 10 | t        | t       |                  1 |                3
(1 row)
$ psql -c "SELECT id,email FROM bounce WHERE id=2;"
 id |     email
----+----------------
  2 | hey@google.com
(1 row)
$ psql -c "SELECT id,user_id,left(title,52) AS title FROM notification WHERE id=8;"
 id | user_id |                        title
----+---------+------------------------------------------------------
  8 |       1 | Email cannot be sent to hey@google.com from your alias
(1 row)
```

Return code: **E212** = `250 SL E212 Bounce Reply phase handled` (`app/email/status.py:20`). This exercises the reply-phase bounce branch end to end.

**(d) The bounce guard → `E213` (a non-DSN sent to a VERP address).** If a message that is *not* a genuine DSN (real sender, not `multipart/report`) arrives at a VERP address, `is_bounce()` is false, `handle()` raises `VERPForward`, and `handle_DATA()` catches `(VERPReply, VERPForward, VERPTransactional)` (`email_handler.py:2308`, logging at `:2309`) and returns `status.E213` (`email_handler.py:2318`) — silently ignoring it:

The SMTP exchange, captured from the server's `EHLO`-response onward (opening lines identical to the complete transcripts above):

```
$ swaks --from stranger@spam.example --to sl.lmycyibrgewcamrtg4ztcmzylu.njc6kmyymtkm6@sl.local \
        --server 127.0.0.1:20381 --header "Subject: not a bounce" --body "plain message to a VERP address"
<-  250-sl-app
<-  250-SIZE 33554432
<-  250-8BITMIME
<-  250-SMTPUTF8
<-  250 HELP
 -> MAIL FROM:<stranger@spam.example>
<-  250 OK
 -> RCPT TO:<sl.lmycyibrgewcamrtg4ztcmzylu.njc6kmyymtkm6@sl.local>
<-  250 OK
 -> DATA
<-  354 End data with <CR><LF>.<CR><LF>
<-  250 SL E213 Unknown email ignored
 -> QUIT
<-  221 Bye
```

```
2026-07-07 00:19:41,106 - SL - INFO - 5694 - "/app/email_handler.py:2343" - _handle() - 29f5dc01-3275-4c91-a924-4dc940fc21ca - New message, mail from stranger@spam.example, rctp tos ['sl.lmycyibrgewcamrtg4ztcmzylu.njc6kmyymtkm6@sl.local']
2026-07-07 00:19:41,114 - SL - WARNING - 5694 - "/app/email_handler.py:2309" - handle_DATA() - 29f5dc01-3275-4c91-a924-4dc940fc21ca - email handling fail with error:VERPForward  mail_from:stranger@spam.example, rcpt_tos:['sl.lmycyibrgewcamrtg4ztcmzylu.njc6kmyymtkm6@sl.local']
```

Return code: **E213** = `250 SL E213 Unknown email ignored` (`app/email/status.py:21`). This proves VERP addresses accept *only* genuine DSNs.

**Situation matrix (all handled automatically by the running email handler).** Every branch was driven through the real `:20381` listener and returned its own distinct SMTP code:

| Situation | Handler function | SMTP code | Evidence |
|-----------|------------------|-----------|----------|
| Forward (mail → alias → mailbox) | `handle_forward` (`email_handler.py:536`) | E200 (`app/email/status.py:2`) | §3.4 (`EmailLog 7`) + web-down `EmailLog 9` |
| On-the-fly auto-create | `try_auto_create_directory` (`app/alias_utils.py:227`) | E200 | §3.5 (`Alias 22`, `EmailLog 8`) |
| Reply (mailbox → contact) | `handle_reply` (`email_handler.py:966`) | E200 | `EmailLog 10`, `is_reply=t` |
| Forward-phase bounce | `handle_bounce_forward_phase` (`email_handler.py:1432`) | E211 (`app/email/status.py:19`) | `EmailLog 11 bounced=t`, `Bounce 3` (mailbox) |
| Reply-phase bounce | `handle_bounce_reply_phase` (`email_handler.py:1595`) | E212 (`app/email/status.py:20`) | `EmailLog 10 bounced=t`, `Bounce 2` (contact), `Notification 8` |
| Non-DSN at a VERP (guard) | `handle_DATA` catch (`email_handler.py:2308,2318`) | E213 (`app/email/status.py:21`) | `WARNING VERPForward`, no DB change |

**Conclusion (R3, part 2):** once running, the email handler classifies and processes every situation on its own — forward, auto-create, reply, both bounce phases, and the guard — each with a distinct, observable SMTP status and matching DB state; and the job runner picks up and completes queued work on its own ~10 s poll. The background components come online independently and do their jobs without any prompting from the web app. (All temporary rows created here — `EmailLog 9/10/11`, `Bounce 2/3`, `RefusedEmail 3/4`, `Notification 8`, and jobs 25–29 — were removed in §6.)

---

## 5. Coverage summary and discrepancies

### 5.1 Coverage matrix — every named item, its mechanism, and its observed value

This is the final coverage pass over the three questions. Each named mechanism is answered by the specific function that performs the work (with a `file:line`), the concrete value observed at runtime, and the section that shows the producing command and complete, unedited output.

| Question / named item | Mechanism (`file:line`) | Observed value (HTTP / SMTP / DB) | Evidence |
|-----------------------|-------------------------|-----------------------------------|----------|
| **R1 — Web server up** | `healthcheck()` `server.py:213-215`; `/live` `app/monitor/views.py:10-12`; `/git` `app/monitor/views.py:5-7` | `GET /health` → body `success`, HTTP 200; `/live` → `live`; `/git` → `dev` | §2.1 |
| **R1 — Email handler up** | `main()` `email_handler.py:2381`; `Controller(...)` `email_handler.py:2383` | bound on `0.0.0.0:20381`; logs `Start mail controller` (`:2386`) and `Listen for port 20381` (`:2403`) | §2.2 |
| **R1 — Job runner up** | `__main__` loop `job_runner.py:329`; `time.sleep(10)` `job_runner.py:347` | polls the `job` table every ~10 s; logs `Take job …` (`:334`) | §2.3 |
| **R2 — Account creation** | `register()` `app/auth/views/register.py:31`; `ActivationCode.create(...)` `app/auth/views/register.py:120` | log `create user …` (`app/auth/views/register.py:85`); `User 5`, `User 6`; `ActivationCode 3`, `4` (`length(code)=30`) | §3.1 |
| **R2 — Account activation** | `activate()` `app/auth/views/activate.py:17`; `user.activated = True` `app/auth/views/activate.py:49` | `activated` f→t; code count 1→0; flash `Your account has been activated` (`app/auth/views/activate.py:56`) | §3.2 |
| **R2 — Alias creation (random)** | `Alias.create_new_random(...)` `app/dashboard/views/index.py:104` | log `create new random alias …` (`app/dashboard/views/index.py:110`); flash (`app/dashboard/views/index.py:111`); `Alias 20` `test_word970@sl.local` | §3.3 |
| **R2 — Alias creation (custom)** | `Alias.create(...)` `app/dashboard/views/custom_alias.py:139`; prefix guard `app/dashboard/views/custom_alias.py:64-70` | flash `Alias … has been created` (`app/dashboard/views/custom_alias.py:159`); `Alias 21`; validation-edge flash on a bad prefix | §3.3 |
| **R2 — Inbound email (forward)** | `handle_forward(...)` `email_handler.py:536`; `EmailLog` write `email_handler.py:740` | SMTP **E200** (`app/email/status.py:2`); `Contact 5`, `EmailLog 7`; sink copy delivered to `john@wick.com` | §3.4 |
| **R2 — On-the-fly auto-create** | `try_auto_create(...)` `app/alias_utils.py:202` → `try_auto_create_directory(...)` `app/alias_utils.py:227` | SMTP **E200**; `Alias 22` (`directory_id=1`), `Contact 6`, `EmailLog 8` | §3.5 |
| **R3 — No self-spawning** | `server.py` (searched) | no `email_handler`/`job_runner`/`subprocess`/`Popen`/`multiprocessing`/`Thread`/`Controller` — `grep` exit 1 | §4.1 |
| **R3 — Handler independent of web app** | `handle_forward(...)` `email_handler.py:536` | mail accepted (E200, `EmailLog 9`) while `GET /health` was refused (`curl_rc=7`) | §4.1 |
| **R3 — Job auto-pickup + cadence** | `Take job` `job_runner.py:334`; `time.sleep(10)` `job_runner.py:347` | `job 25` state 0→2; consecutive gaps `10.070`/`10.072`/`10.069 s` | §4.2, §2.3 |
| **R3 — Reply variant** | `handle_reply(...)` `email_handler.py:966` | SMTP **E200**; `EmailLog 10` `is_reply=t` | §4.3(a) |
| **R3 — Forward-phase bounce** | `handle_bounce_forward_phase(...)` `email_handler.py:1432` | SMTP **E211** (`app/email/status.py:19`); `EmailLog 11 bounced=t`; `Bounce 3` = mailbox `john@wick.com` | §4.3(b) |
| **R3 — Reply-phase bounce** | `handle_bounce_reply_phase(...)` `email_handler.py:1595` | SMTP **E212** (`app/email/status.py:20`); `EmailLog 10 bounced=t`; `Bounce 2` = contact `hey@google.com`; `Notification 8` | §4.3(c) |
| **R3 — Bounce guard (non-DSN at VERP)** | `handle_DATA(...)` catch `email_handler.py:2308,2318` | SMTP **E213** (`app/email/status.py:21`); `WARNING …VERPForward`; no DB change | §4.3(d) |

Every named mechanism the question raises — the **web server**, **email handler**, and **job runner** (R1); **signup**, **activation**, **random** and **custom** alias creation, **inbound email**, and **auto-create** (R2); **no-spawn**, **job pickup**, and the **reply / forward-bounce / reply-bounce / guard** situations (R3) — is addressed above with a concrete observed value and a `file:line`.

### 5.2 Discrepancies and points worth noting

A few observations differ from what a first glance at the code might suggest; each is reported **as observed**, not as the code appears to imply:

1. **`/health` line span.** The `@app.route("/health")` decorator is at `server.py:213`, the `def healthcheck()` signature at `server.py:214`, and the `return "success", 200` at `server.py:215`. The behavior — HTTP 200 with the exact 7-byte body `success` — is what confirms liveness (also noted at §2.1).
2. **`/git` returns the literal `dev`, not a commit SHA.** The endpoint returns `SHA1` from `app/build_info.py:1`, which in this image is `SHA1 = "dev"` (no build-time SHA was injected). So `dev` is the correct, expected value here — not a missing build.
3. **The job runner's `taken` state is transient.** `JobState` is `ready=0, taken=1, done=2` (`app/models.py:253`). Within one loop iteration the runner marks a job `taken` (`job_runner.py:339`), runs `process_job(...)` (`job_runner.py:342` → `job_runner.py:188`), then marks it `done` (`job_runner.py:344`) — all before the next `time.sleep(10)`. A poll therefore observes `0 → 2`; the `taken=1` state exists only for the sub-second processing window (this is why §4.2 shows `state=0` then `state=2`).
4. **Auto-create runtime path (both routes are tried, in order).** The inbound-mail auto-create is reached from `handle_forward` at `email_handler.py:549` (`try_auto_create`, defined at `app/alias_utils.py:202`). `try_auto_create` tries the **custom-domain** route *first* — `try_auto_create_via_domain` (`app/alias_utils.py:274`) → `check_if_alias_can_be_auto_created_for_custom_domain` (`app/alias_utils.py:92`), which logged `Cannot auto-create custom domain alias … because there's no custom domain` at `app/alias_utils.py:104` and returned `None` (`sl.local` is a directory domain, not a custom domain) — and only then fell through to the **directory** route: `try_auto_create_directory` (`app/alias_utils.py:227`) → `check_if_alias_can_be_auto_created_for_a_directory` (`app/alias_utils.py:145`), which logged the directory name at `app/alias_utils.py:169` and created the alias, logging `create alias … for directory …` at `app/alias_utils.py:238`. So the `via_domain` branch *is* executed (it emits the `:104` line) but returns `None`; the directory branch is what actually creates the alias. This matches the captured log chain in §3.5.
5. **The `E213` catch is at `:2308/:2318`, not `:2307`.** In `handle_DATA`, `email_handler.py:2307` is a *different* branch (`return status.E524` for `CannotCreateContactForReverseAlias`). The guard that returns `E213` is the `except (VERPReply, VERPForward, VERPTransactional)` at `email_handler.py:2308` (warning at `email_handler.py:2309`), whose `return status.E213` is at `email_handler.py:2318`.
6. **`Bounce.email` distinguishes the two bounce phases.** A forward-phase bounce records `Bounce.email` = the **mailbox** (`john@wick.com`, `Bounce 3`), whereas a reply-phase bounce records `Bounce.email` = the **contact** (`hey@google.com`, `Bounce 2`). This observed difference is the clearest database signal for telling the two phases apart.
7. **`CONTRIBUTING.md` documents a different — and internally inconsistent — Postgres port.** The self-hosting guide's `DB_URI` example uses host port `35432` (`CONTRIBUTING.md:94`), while the very next command publishes the database on a *different* host port — `docker run … -p 15432:5432 postgres:13` (`CONTRIBUTING.md:100`). This container instead runs Postgres on the standard `5432` (the `DB_URI` shown in §1.1), so neither documented port (`35432`/`15432`) applies here; the observed, working value is `5432`.


---

## 6. Cleanup and read-only proof

The investigation's read-only rule requires that every temporary account, alias, contact, email-log, job, bounce, notification and observation script created above be removed, and that the database and source tree be verified back at their original state. This section shows that, with the deletion commands and their before/after output.

### 6.1 Temporary database rows removed

All rows created during the investigation were deleted in a single transaction. The exact script that was run (`psql -f "$OBS/cleanup.sql"`):

```
\echo ===== BEFORE cleanup: counts (users, alias, contact, email_log, job, notification, bounce, refused_email) =====
SELECT count(*) FROM users; SELECT count(*) FROM alias; SELECT count(*) FROM contact; SELECT count(*) FROM email_log;
SELECT count(*) FROM job; SELECT count(*) FROM notification; SELECT count(*) FROM bounce; SELECT count(*) FROM refused_email;

BEGIN;
-- john (user 1) temp aliases -> cascades their contacts (5,6,7) + email_logs (7..11)
DELETE FROM alias WHERE id IN (19,20,21,22);
-- refused emails created by the two bounce tests (now unreferenced)
DELETE FROM refused_email WHERE id IN (3,4);
-- all post-seed bounces (seed has 0) incl. stale prior-run orphan id 1
DELETE FROM bounce WHERE id IN (1,2,3);
-- post-seed notifications (seed has 6: ids 1-6); remove stale 7 + my 8,9
DELETE FROM notification WHERE id IN (7,8,9);
-- onboarding jobs for temp users 5,6 + export-report jobs from the poll tests
DELETE FROM job WHERE id BETWEEN 19 AND 29;
-- temp accounts -> cascades their newsletter aliases (17,18), mailboxes, activation codes
DELETE FROM users WHERE id IN (5,6);
COMMIT;

\echo ===== AFTER cleanup: counts (must equal pristine seed 2 | 11 | 1 | 1 | 6 | 6 | 0 | 1) =====
SELECT count(*) FROM users; SELECT count(*) FROM alias; SELECT count(*) FROM contact; SELECT count(*) FROM email_log;
SELECT count(*) FROM job; SELECT count(*) FROM notification; SELECT count(*) FROM bounce; SELECT count(*) FROM refused_email;
```

The before/after counts it produced (`users | alias | contact | email_log | job | notification | bounce | refused_email`):

```
BEFORE:  4 | 17 |  4 |  6 | 20 |  9 |  3 |  3
AFTER :  2 | 11 |  1 |  1 |  6 |  6 |  0 |  1     (== pristine seed baseline)
```

Note that the `BEFORE` row is larger than just this session's additions: it also contains a few stale rows left by earlier exploratory runs (an orphan `bounce 1`, `notification 7`, and some onboarding jobs). The transaction deletes **all** post-seed rows so the database returns to the exact untouched seed state — `2 | 11 | 1 | 1 | 6 | 6 | 0 | 1` — which is the same baseline a clean `dummy-data` seed produces on the scratch database in §1.2.

A fresh re-verification confirms the counts are still at baseline and that every temporary entity is gone. This is idempotent — re-running the checks finds nothing left to delete:

```
$ psql -c "SELECT (SELECT count(*) FROM users) AS users, (SELECT count(*) FROM alias) AS alias,
                   (SELECT count(*) FROM contact) AS contact, (SELECT count(*) FROM email_log) AS email_log,
                   (SELECT count(*) FROM job) AS job, (SELECT count(*) FROM notification) AS notification,
                   (SELECT count(*) FROM bounce) AS bounce, (SELECT count(*) FROM refused_email) AS refused_email;"
 users | alias | contact | email_log | job | notification | bounce | refused_email 
-------+-------+---------+-----------+-----+--------------+--------+---------------
     2 |    11 |       1 |         1 |   6 |            6 |      0 |             1
(1 row)
```

Every temporary entity referenced earlier — `User 5/6`, `Alias 17..22`, `Contact 5/6/7`, `EmailLog 7..11` — returns **0 rows**:

```
$ psql -tAc "SELECT (SELECT count(*) FROM users     WHERE id IN (5,6)),
                    (SELECT count(*) FROM alias     WHERE id IN (17,18,19,20,21,22)),
                    (SELECT count(*) FROM contact   WHERE id IN (5,6,7)),
                    (SELECT count(*) FROM email_log WHERE id IN (7,8,9,10,11)),
                    (SELECT count(*) FROM bounce    WHERE id IN (2,3)),
                    (SELECT count(*) FROM refused_email WHERE id IN (3,4)),
                    (SELECT count(*) FROM notification  WHERE id IN (7,8,9)),
                    (SELECT count(*) FROM job       WHERE id BETWEEN 19 AND 29);"
0|0|0|0|0|0|0|0
```

### 6.2 Temporary observation scripts removed

The captured process logs, SQL helpers and DSN files lived in a single temporary directory (`$OBS`) that is **outside the repository** — container-local scratch under `/tmp`. It was removed at the end of the investigation (the check is written to avoid printing the absolute path):

```
$ ls -1 "$OBS" | wc -l
50
$ rm -rf "$OBS"
$ [ -e "$OBS" ] && echo "still present" || echo "removed"
removed
```

### 6.3 Source code untouched (read-only proof)

**Container source tree (`/app`).** `git status --porcelain` reports only eight files, and all eight are **pre-existing image-build artifacts** (generated keys, a word list, a lock file) — none touched by this investigation:

```
$ cd /app && git status --porcelain
 M app/spamassassin_utils.py
 M local_data/dkim.key
 M local_data/jwtRS256.key
 M local_data/jwtRS256.key.pub
 M local_data/paddle.key.pub
 M local_data/private-pgp.asc
 M local_data/test_words.txt
 M static/package-lock.json
```

Their modification times pre-date the start of this investigation (`2025-08-26` for the image build; `local_data/dkim.key` at `2026-07-06 21:52`, the environment-setup step). Decisively, **no `.py` source file was modified after the investigation began** (`23:00` on `2026-07-06`):

```
$ cd /app && find . -name "*.py" -newermt "2026-07-06 23:00" -not -path "./venv/*"
$        # (no output — no source .py file is newer than the session start)
```

**Deliverable repository.** In the repository where this document lives, exactly one path is changed — this document:

```
$ git status --porcelain
 M blitzy/documentation/app_2cd6ee777f8c.md
```

So the only persistent change produced by this entire investigation is this single markdown document; every temporary account, alias, row, job and script was removed, and no source code — in the container or in this repository — was modified.

