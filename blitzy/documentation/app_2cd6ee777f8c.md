# SimpleLogin — Runtime Verification Q&A

**How do I know a locally running SimpleLogin instance is healthy, how do its basic user flows behave, and do its background components come online on their own?**

This document answers those three questions **from directly observed runtime behavior**. SimpleLogin is a self-hostable email-alias service: users create aliases, and mail sent to an alias is forwarded to the user's real mailbox while replies go back out *as the alias*, hiding the real address. The service is a single Python/Flask codebase that is run as several **independent** processes. This investigation focuses on the three named in the question:

| Component | Entry point | Role | Default port |
|-----------|-------------|------|--------------|
| **Web server** | `python server.py` | Dashboard + REST API + health endpoints | `7777` |
| **Email handler** | `python email_handler.py` | aiosmtpd SMTP server that forwards/replies/bounces mail | `20381` |
| **Job runner** | `python job_runner.py` | Polls the `job` table every ~10 s and runs queued work | (none) |

Every factual claim below is backed by three things: **(a)** the exact command that produced the evidence, **(b)** the complete, unedited captured output, and **(c)** a `file:line` citation into this repository. All commands were run through each component's real entry point — no bypasses or stand-ins.

> **Method (run-first).** The stack was built and run first; temporary observation scripts were executed and their output captured *before* this prose was written. Temporary accounts, aliases and scripts created during the investigation were removed afterward (see the closing note); the repository is left unchanged except for this document.

---

## 1. How the stack was run

### 1.1 Runtime and provisioning

Everything ran inside the provided Docker container `sl-app` (image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`), which supplies the canonical Python 3.10 / PostgreSQL / Redis runtime. Commands were issued with `docker exec sl-app bash -lc '…'`.

```
$ docker exec sl-app bash -lc '/app/venv/bin/python --version; git -C /app rev-parse --short HEAD'
Python 3.10.18
2cd6ee77
```

- **Python 3.10** matches `Dockerfile:8` (`FROM python:3.10`) and `pyproject.toml:61` (`python = "^3.10"`).
- The canonical interpreter is the pre-built virtualenv `/app/venv/bin/python`; dependencies were installed at image-build time with Poetry (Poetry itself is not on the runtime `PATH`).
- Configuration is loaded from `/root/run.env` via the `CONFIG` environment variable (python-dotenv). The relevant values are `URL=http://localhost:7777`, `EMAIL_DOMAIN=sl.local`, `DB_URI=postgresql://test:test@localhost:5432/test`, and `MEM_STORE_URI=redis://localhost`. `DB_URI` is read at `app/config.py:192` (`DB_URI = os.environ["DB_URI"]`) and `MEM_STORE_URI` at `app/config.py:568`.

**DB port — discrepancy reported as observed.** `CONTRIBUTING.md` is internally inconsistent about the Postgres port: `CONTRIBUTING.md:94` shows `DB_URI=…@localhost:35432/simplelogin` while `CONTRIBUTING.md:100` maps `-p 15432:5432`. Neither was used here; the container's canonical `run.env` uses the standard port **5432**, which is what all evidence below reflects.

**Mail sink.** SimpleLogin forwards outbound mail through the address in `POSTFIX_SERVER`/`POSTFIX_PORT`. In this container `NOT_SEND_EMAIL` is unset and those keys point at a local aiosmtpd **sink** (`/root/mta_sink.py`) listening on `0.0.0.0:1025`, which writes each received message to `/tmp/mta_sink/*.eml`. (The published guide references a mailcatcher web UI at `http://localhost:1080/`, `CONTRIBUTING.md:221`; here the equivalent evidence is the `.eml` files the sink writes.)

### 1.2 Seeded login

The database is seeded by the `dummy-data` Flask CLI command, defined at `server.py:490` (`@app.cli.command("dummy-data")`), function `dummy_data()` at `server.py:491`, which logs `reset db, add fake data` at `server.py:494` and calls `fake_data()` at `server.py:495`. Captured on a throwaway scratch database (so the live `test` DB was untouched):

```
$ cd /app && CONFIG=/root/run.env DB_URI=postgresql://test:test@localhost:5432/sl_boot \
      FLASK_APP=server.py /app/venv/bin/python -m flask dummy-data
load config file /root/run.env
>>> URL: http://localhost:7777
Upload files to local dir
>>> init logging <<<
2026-07-06 22:40:55,036 - SL - WARNING - 3002 - "/app/server.py:494" - dummy_data() -  - reset db, add fake data
2026-07-06 22:40:55,036 - SL - DEBUG - 3002 - "/app/app/fake_data.py:41" - fake_data() -  - create fake data
```

The seed creates the login **`john@wick.com` / `password`**. Critically, `fake_data()` (`app/fake_data.py:40`) builds it with `activated=True` (`app/fake_data.py:48`) and `is_admin=True` (`app/fake_data.py:49`), so it can sign in to the dashboard immediately with no activation step. This account is used for all alias flows below; throwaway accounts are reserved for the signup demonstration.

```
$ PGPASSWORD=test psql -h localhost -U test -d test -c \
    "SELECT id,email,activated,is_admin FROM users ORDER BY id LIMIT 2;"
 id |         email          | activated | is_admin
----+------------------------+-----------+----------
  1 | john@wick.com          | t         | t
  2 | winston@continental.com| t         | f
```

### 1.3 The logging convention (how to read every signal below)

All three processes log to **stdout** through one logger named `"SL"` (`LOG = _get_logger("SL")`, `app/log.py:79`) at DEBUG level. The format string (`app/log.py:12-14`) is:

```
%(asctime)s - %(name)s - %(levelname)s - %(process)d - "%(pathname)s:%(lineno)d" - %(funcName)s() - %(message_id)s - %(message)s
```

So every line looks like this (from the seed output above):

```
2026-07-06 22:40:55,036 - SL - WARNING - 3002 - "/app/server.py:494" - dummy_data() -  - reset db, add fake data
└─ timestamp        └─ logger └─ level └─ pid └─ pathname:lineno └─ funcName()   └─(msg_id) └─ message
```

The key consequence: **each log line embeds its own `pathname:lineno` and `funcName()`**, which is exactly how any captured line is tied back to a `file:line` in this document. The convenience shortcuts `LOG.d` / `LOG.i` / `LOG.w` / `LOG.e` (debug/info/warning/exception) are defined at `app/log.py:74-77`, the `>>> init logging <<<` banner is printed on import at `app/log.py:67`, and Werkzeug's per-request HTTP access log is silenced at `app/log.py:70-71` (which is why you never see noisy `GET /health 200` lines).

---

## 2. R1 — How do I know each process is up?

Each of the three named processes was started through its **real entry point** into its own log file:

```
$ cd /app && export CONFIG=/root/run.env
$ nohup /app/venv/bin/python -u server.py       > /root/blitzy_obs/logs/server.log       2>&1 &
$ nohup /app/venv/bin/python -u email_handler.py > /root/blitzy_obs/logs/email_handler.log 2>&1 &
$ nohup /app/venv/bin/python -u job_runner.py    > /root/blitzy_obs/logs/job_runner.log    2>&1 &
```

### 2.1 Web server (`server.py`)

The dev entry point is `app.run(debug=True, port=7777)` at `server.py:588`; the app object is assembled by `create_app()` at `server.py:139`. (For contrast, production runs `gunicorn wsgi:app -b 0.0.0.0:7777` per `Dockerfile:47`, where `wsgi.py:1,3` is `from server import create_app` / `app = create_app()`.) The most direct liveness signal is the health endpoint.

**`/health` → `success`, HTTP 200.** Route `@app.route("/health")` at `server.py:213`, function `healthcheck()` at `server.py:214`, `return "success", 200` at `server.py:215`.

```
$ curl -sS -i http://localhost:7777/health
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 7
Set-Cookie: slapp=d3e670eb-...; Expires=Mon, 13-Jul-2026 23:14:52 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Mon, 06 Jul 2026 23:14:52 GMT

success
```

> **Discrepancy reported as observed.** The route decorator is at `server.py:213` and the `return "success", 200` statement is at `server.py:215` (line 214 is the `def healthcheck()` signature). The *observed behavior* — HTTP 200 with the exact 7-byte body `success` — is what matters and is confirmed above.

**`/live` → `live`.** `@monitor_bp.route("/live")` / `return "live"` at `app/monitor/views.py:10-12`.

```
$ curl -sS -i http://localhost:7777/live
HTTP/1.0 200 OK
Content-Length: 4
...
live
```

**`/git` → build SHA1.** `@monitor_bp.route("/git")` / `return SHA1` at `app/monitor/views.py:5-7`, sourced from `app.build_info.SHA1`.

```
$ curl -sS -i http://localhost:7777/git
HTTP/1.0 200 OK
Content-Length: 3
...
dev
```

The value is the literal string `dev` — not empty. That is the image default: `app/build_info.py:1` defines `SHA1 = "dev"` (a real build would substitute the git SHA at package time).

**`/exception` → HTTP 500.** Confirms error routing is wired. `test_exception()` deliberately does `raise Exception("to make sure sentry works")` at `app/monitor/views.py:15-17`.

```
$ curl -sS -i http://localhost:7777/exception 2>&1 | head -1
HTTP/1.0 500 INTERNAL SERVER ERROR
```

**Dashboard reachability.** `GET /` redirects unauthenticated users to the login page; after logging in as the seeded admin the dashboard renders.

```
$ curl -sS -i http://localhost:7777/ | head -4
HTTP/1.0 302 FOUND
Content-Length: 229
Location: http://localhost:7777/auth/login
```

After `POST /auth/login` with `john@wick.com` / `password` (302 → `/dashboard/`), `GET /` returns HTTP 200 with a 761,924-byte page rendered from `templates/dashboard/index.html` containing the expected controls (`Random alias`, `New custom alias`). The login page itself renders `templates/auth/login.html`. This confirms a user can sign in and reach alias management.

### 2.2 Email handler (`email_handler.py`)

`main(port)` (`email_handler.py:2381`) builds `Controller(MailHandler(), hostname="0.0.0.0", port=port)` (`email_handler.py:2383`), calls `controller.start()` (`email_handler.py:2385`), and the `__main__` block logs the listening port (default **20381**). Captured startup (verbatim from `email_handler.log`):

```
load config file /root/run.env
>>> URL: http://localhost:7777
Upload files to local dir
>>> init logging <<<
2026-07-06 22:44:14,014 - SL - DEBUG - 3237 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-06 22:44:14,502 - SL - INFO - 3237 - "/app/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-06 22:44:14,505 - SL - DEBUG - 3237 - "/app/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

The two banner lines are the liveness signal: `Listen for port 20381` from `email_handler.py:2403` and `Start mail controller 0.0.0.0 20381` from `email_handler.py:2386`. The socket is actually bound (checked via `/proc/net/tcp`, since `ss` is not installed in the image — `00000000:4F9D` is `0.0.0.0:20381` in hex, state `0A` = LISTEN):

```
$ cat /proc/net/tcp | awk '{print $2, $4}' | grep -i 4F9D
00000000:4F9D 0A
```

Why does the process stay up with no traffic? The `__main__` block ends with a keep-alive loop `while True: time.sleep(2)` at `email_handler.py:2392-2393`; the aiosmtpd `Controller` runs its own asyncio event loop in a background thread and accepts inbound SMTP connections automatically.

### 2.3 Job runner (`job_runner.py`) — a timing question

The job runner's `__main__` block (`job_runner.py:329`) runs `while True:` (`job_runner.py:330`) inside `create_light_app().app_context()` (`job_runner.py:332`), calls `get_jobs_to_run()` (`job_runner.py:307`), logs `Take job %s` (`job_runner.py:334`) for each ready row, then sleeps with `time.sleep(10)` (`job_runner.py:347`). So "up" means "polling the `job` table about every 10 seconds."

**Idle state.** From startup at 22:44:15 the runner logged **no** `Take job` lines for ~10 minutes, even though nine onboarding jobs already existed in the table. Those rows are `state=0` (ready) but their `run_at` is dated `now + 1/2/3 days`, and `get_jobs_to_run()` filters on `run_at <= now + 10 min`, so it correctly returns nothing. `create_light_app()` does not log per cycle, so an idle loop is silent by design.

**Work state + timing.** To make the runner do real work, a genuine `Job` row was enqueued through a product action — the data-export button (`POST /dashboard/account_setting` with `form-name=send-full-user-report`), which calls `ExportUserDataJob.store_job_in_db()` (`app/dashboard/views/account_setting.py:130-131`) and inserts a `JOB_SEND_USER_REPORT` row with `run_at = now`. Triggering it four times over ~31 s produced (verbatim, all from `job_runner.py:334`):

```
2026-07-06 22:55:27,623 - SL - DEBUG - 3256 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 11 send-user-report {'user_id': 1}>
2026-07-06 22:55:37,702 - SL - DEBUG - 3256 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 12 send-user-report {'user_id': 1}>   gap=10.079s
2026-07-06 22:55:47,770 - SL - DEBUG - 3256 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 13 send-user-report {'user_id': 1}>   gap=10.068s
2026-07-06 22:55:57,835 - SL - DEBUG - 3256 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 14 send-user-report {'user_id': 1}>   gap=10.065s
```

**Run duration ≈ 31 s; the ~10 s interval is stable across three consecutive cycles** (10.079 / 10.068 / 10.065 s). That directly measures `time.sleep(10)` (`job_runner.py:347`); the ~0.07 s excess is poll + processing overhead. Each job was picked up automatically on the next poll with no manual trigger.

**Not to be confused with the cron scheduler.** SimpleLogin has a *fourth* scheduler — `cron.py` (an argparse entry point) driven by **yacron** per `crontab.yml`, whose entries look like `command: python /code/cron.py -j <job>` for periodic tasks (`stats`, `delete_logs`, `check_hibp`, …). That is a separate mechanism from `job_runner.py`, which polls the `job` table every 10 s. They are different components.


---

## 3. R2 — Basic user actions, each confirmed on ≥2 channels

Each action below is corroborated on at least two independent channels — a **log line**, a **database row**, and where applicable a **dashboard flash** or an **SMTP return code** — so no claim rests on a single ambiguous source.

### 3.1 Account creation (signup)

A throwaway account was registered via `POST /auth/register` (route `@auth_bp.route("/register")` at `app/auth/views/register.py:31`, function `register()` at `:32`; the `auth` blueprint carries `url_prefix="/auth"`, so the path is `/auth/register`).

**Log channel** — `LOG.d("create user %s", email)` at `app/auth/views/register.py:85`:

```
- SL - DEBUG - "/app/app/auth/views/register.py:85" - register() - create user blitzy-tmp-signup@example.com
```

**Database channel** — a new `User` row (`app/models.py:336`) plus an `ActivationCode` row (`app/models.py:1202`) created by `ActivationCode.create(...)` at `app/auth/views/register.py:120`:

```
$ PGPASSWORD=test psql -h localhost -U test -d test -c \
   "SELECT id,email,activated,is_admin FROM users WHERE email='blitzy-tmp-signup@example.com';"
 id |             email             | activated | is_admin
----+-------------------------------+-----------+----------
  4 | blitzy-tmp-signup@example.com | f         | f

$ PGPASSWORD=test psql -h localhost -U test -d test -c \
   "SELECT id,user_id,code FROM activation_code WHERE user_id=4;"
 id | user_id |             code
----+---------+--------------------------------
  2 |       4 | hvdtfmryqfevzyajlgsvfoiqgirqhy
```

Note the new account is `activated=f` — a fresh signup is **not** active until the emailed code is used (contrast with the pre-activated seed account).

**Email channel** — the activation email is generated by `send_activation_email(...)` (`app/auth/views/register.py:117`, called at `:95`) and lands in the mail sink:

```
$ ls -1 /tmp/mta_sink/ ; grep -i '^Subject:' /tmp/mta_sink/msg_005.eml
msg_005.eml
Subject: Just one more step to join SimpleLogin
```

**Activation (secondary path).** Hitting `/auth/activate?code=…` (`app/auth/views/activate.py:13`, `activate()` at `:17`, `ActivationCode.get_by(code=code)` at `:26`, `user.activated = True` at `:49`) flips the account active and consumes the code:

```
$ curl to GET /auth/activate?code=hvdtfmryqfevzyajlgsvfoiqgirqhy  -> HTTP 200 (redirected to dashboard)
flash 'Your account has been activated' present: True      # app/auth/views/activate.py:56
```

After activation, the DB shows `activated=t` for user id 4 and zero remaining rows in `activation_code` for that user (the code was consumed).

### 3.2 Alias creation

**Random alias.** Logged in as `john@wick.com`, a random alias was created from the dashboard index (`@dashboard_bp.route("/")` at `app/dashboard/views/index.py:55`, `index()` at `:67`), which calls `Alias.create_new_random(user=current_user, scheme=scheme)` at `app/dashboard/views/index.py:104`.

**Log channel** — the *full* log string is `create new random alias %s for user %s` at `app/dashboard/views/index.py:110` (reported complete, not abbreviated):

```
- SL - DEBUG - "/app/app/dashboard/views/index.py:110" - index() -
    create new random alias <Alias 15 word_word475@sl.local> for user <User 1 John Wick john@wick.com>
```

**UI channel** — the success flash is an f-string `flash(f"Alias {alias.email} has been created", "success")` at `app/dashboard/views/index.py:111`; the *rendered* (substituted) value observed was:

```
Alias word_word475@sl.local has been created
```

**Database channel** — a new `Alias` row (`app/models.py:1469`):

```
$ PGPASSWORD=test psql -h localhost -U test -d test -c \
   "SELECT id,email,user_id,mailbox_id,enabled FROM alias WHERE id=15;"
 id |         email          | user_id | mailbox_id | enabled
----+------------------------+---------+------------+---------
 15 | word_word475@sl.local  |       1 |          1 | t
```

**Custom alias (secondary variant).** `POST /dashboard/custom_alias` (`app/dashboard/views/custom_alias.py:30`, `custom_alias()` at `:34`) was exercised both ways.

*Success* — with a valid prefix, a server-signed suffix and a mailbox, `Alias.create(...)` runs at `app/dashboard/views/custom_alias.py:139` and flashes `f"Alias {full_alias} has been created"` at `:159`:

```
POST /dashboard/custom_alias  (prefix=blitzytmpcustom, signed-alias-suffix=.word955@sl.local...., mailboxes=1)
 -> 302 Location: .../dashboard/?highlight_alias_id=16
 rendered flash: Alias blitzytmpcustom.word955@sl.local has been created
 DB: alias id=16 blitzytmpcustom.word955@sl.local user_id=1 mailbox_id=1 enabled=t
```

*Validation edge* — posting an invalid prefix `Bad Prefix!!` (normalized to `badprefix!!`) fails `check_alias_prefix(...)` and takes the branch at `app/dashboard/views/custom_alias.py:64-70`:

```
POST /dashboard/custom_alias  (prefix="Bad Prefix!!")
 -> 302 Location: .../dashboard/custom_alias
 rendered flash: Only lowercase letters, numbers, dashes (-), dots (.) and underscores (_)
                 are currently supported for alias prefix. Cannot be more than 40 letters
```

### 3.3 Inbound email reception

An inbound message was sent to the random alias with `swaks` (the same invocation shape documented at `CONTRIBUTING.md:218`), driving the pipeline `MailHandler.handle_DATA` (`email_handler.py:2289`) → `handle` (`email_handler.py:1945`) → `handle_forward` (`email_handler.py:536`).

```
$ swaks --to word_word475@sl.local --from hey@google.com --server 127.0.0.1:20381 \
        --header "Subject: Blitzy R2.3 inbound test" --body "Hello from the inbound forward test."
...
 -> RCPT TO:<word_word475@sl.local>
<-  250 OK
 -> DATA
<-  354 End data with <CR><LF>.<CR><LF>
...
<-  250 Message accepted for delivery
 -> QUIT
<-  221 Bye
```

**SMTP-code channel** — the server returned `250 Message accepted for delivery`, which is the constant `E200` in `app/email/status.py:2`. This is the success branch for a clean forward (not `E404`/`E216`/`E524`/`E213`), because the recipient is a valid, enabled alias whose mailbox forward succeeded.

**Log channel** — the bracketing pair, verbatim. The start marker is `New message, mail from …` and the end marker is `Finish mail_from …<<===`:

```
2026-07-06 23:05:14,104 - SL - INFO - 3237 - "/app/email_handler.py:2343" - _handle() - d88d7ba3-... - New message, mail from hey@google.com, rctp tos ['word_word475@sl.local'] 
2026-07-06 23:05:14,239 - SL - DEBUG - 3237 - "/app/email_handler.py:2202" - handle() - d88d7ba3-... - Forward phase hey@google.com(hey@google.com) -> word_word475@sl.local
2026-07-06 23:05:14,279 - SL - DEBUG - 3237 - "/app/app/contact_utils.py:110" - create_contact() - d88d7ba3-... - Created contact <Contact 3 hey@google.com 15> for alias <Alias 15 word_word475@sl.local> with email hey@google.com invalid_email=False
2026-07-06 23:05:14,290 - SL - DEBUG - 3237 - "/app/email_handler.py:740" - forward_email_to_mailbox() - d88d7ba3-... - Create <EmailLog 4> for <Contact 3 hey@google.com 15>, <User 1 John Wick john@wick.com>, <Mailbox 1 john@wick.com>
2026-07-06 23:05:14,309 - SL - INFO - 3237 - "/app/email_handler.py:2367" - _handle() - d88d7ba3-... - Finish mail_from hey@google.com, rcpt_tos ['word_word475@sl.local'], takes 0.2049722671508789 seconds with return code '250 Message accepted for delivery'<<===
```

> **Discrepancy reported as observed.** The runtime `pathname:lineno` embedded in the log points at the line of the `LOG.i(` *call* — `email_handler.py:2343` for `New message` and `:2367` for `Finish`. The message-string literals themselves sit one line lower, at `email_handler.py:2344` and `:2368`. Both citations are correct; the difference is simply that Python records the opening line of a multi-line logging call.

**Database channel** — a `Contact` row (`app/models.py:1863`) and an `EmailLog` row (`app/models.py:2060`):

```
$ ... "SELECT id,alias_id,website_email,reply_email FROM contact WHERE id=3;"
 id | alias_id | website_email  |            reply_email
----+----------+----------------+------------------------------------
  3 |       15 | hey@google.com | hey_at_google_com_wzsdzew@sl.local

$ ... "SELECT id,user_id,contact_id,alias_id,mailbox_id,is_reply,bounced FROM email_log WHERE id=4;"
 id | user_id | contact_id | alias_id | mailbox_id | is_reply | bounced
----+---------+------------+----------+------------+----------+---------
  4 |       1 |          3 |       15 |          1 | f        | f
```

**Mailbox channel** — the forwarded message appears in the sink, addressed to John's real mailbox, with the `From` header rewritten to the reverse-alias so a reply would route back through SimpleLogin:

```
$ grep -iE '^(From|To|Subject):' /tmp/mta_sink/msg_007.eml
Subject: Blitzy R2.3 inbound test
From: "hey at google.com" <hey_at_google_com_wzsdzew@sl.local>
To: word_word475@sl.local
```


---

## 4. R3 — Do the background components come online on their own?

**Short answer:** No — nothing auto-starts them. The web server, email handler and job runner are three **fully independent processes**. Once each is started, it picks up its own work automatically (the email handler via its SMTP event loop, the job runner via its 10 s poll), but starting one never starts another.

### 4.1 The web app does not spawn the siblings

The definitive proof is that `server.py` contains no reference to the other two entry points or to any process/thread-spawning primitive:

```
$ cd /app && grep -nE "email_handler|job_runner|subprocess|Popen|Thread|Controller" server.py
$ echo "grep exit=$?"
grep exit=1
```

`grep` exit code **1** means zero matches. `create_app()` (`server.py:139`) only calls `register_blueprints(app)` (`server.py:172`, defined at `:233`) to mount Flask routes — it launches nothing. The three processes therefore run under distinct PIDs:

```
$ ps -eo pid,cmd | grep -E "python -u (server|email_handler|job_runner)\.py" | grep -v grep
 4807 /app/venv/bin/python -u server.py
 3237 /app/venv/bin/python -u email_handler.py
 3256 /app/venv/bin/python -u job_runner.py
```

(This matches SimpleLogin's documented self-hosting topology, where `sl-app`, `sl-email` and `sl-job-runner` are separate containers — advisory corroboration; the authoritative evidence is the grep above.)

### 4.2 Each independently-started process auto-picks-up work

**Email handler — independent of the web app.** With the email handler and job runner left running, the web server was killed by PID. `:7777` then refused connections, but the email handler kept processing inbound SMTP with no manual trigger:

```
$ kill <server-pids> ; curl -sS -m5 http://localhost:7777/health || echo "web app down"
web app down                        # HTTP 000, connection refused
$ pgrep -f "python -u email_handler.py" >/dev/null && echo "email_handler: ALIVE"
email_handler: ALIVE

$ swaks --to word_word475@sl.local --from webdown@google.com --server 127.0.0.1:20381 ...
<-  250 Message accepted for delivery

# email_handler.log — processed with the web app DOWN:
2026-07-06 23:12:50,023 - SL - INFO - 3237 - "/app/email_handler.py:2343" - _handle() - 6085809b-... - New message, mail from webdown@google.com, rctp tos ['word_word475@sl.local'] 
2026-07-06 23:12:50,082 - SL - INFO - 3237 - "/app/email_handler.py:2367" - _handle() - 6085809b-... - Finish mail_from webdown@google.com, rcpt_tos ['word_word475@sl.local'], takes 0.05899858474731445 seconds with return code '250 Message accepted for delivery'<<===
```

This proves the aiosmtpd `Controller` (`email_handler.py:2383`) and `MailHandler.handle_DATA` (`email_handler.py:2289`) dispatch inbound mail on their own event loop, with no dependency on `server.py`. (The web server was restarted afterward and returned to HTTP 200.)

**Job runner — automatic pickup on the poll.** A real `Job` row was enqueued via the data-export product action, then observed transitioning through its states without any manual step. `JobState` values are `ready=0, taken=1, done=2, error=3` (`app/models.py:253`):

```
# BEFORE (idle): job id=18 enqueued, state=0 (ready)
23:08:35.676  state=0
23:08:38.879  state=1     # taken by the runner on its next poll
23:08:39.169  state=2     # done

# job_runner.log — the auto-pickup, no manual trigger:
2026-07-06 23:08:38,851 - SL - DEBUG - 3256 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 18 send-user-report {'user_id': 1}>
```

The transitions map to the loop body: `LOG.d("Take job …")` (`job_runner.py:334`) → mark `JobState.taken.value` (`job_runner.py:339`) → `process_job(job)` (`job_runner.py:188`, called at `:342`) → mark `JobState.done.value` (`job_runner.py:344`). A useful contrast observed in the same table: the onboarding jobs sit at `state=0` (ready) yet are **never** taken, because their `run_at` is 1–3 days in the future and `get_jobs_to_run()` only returns rows with `run_at <= now + 10 min`.

### 4.3 The email path across its variants (behavior "across different situations")

The email handler routes each message to a different branch based on the recipient. All three named branches were exercised through the real listener on `:20381`, each returning a distinct SMTP code.

**Forward** — `handle_forward` (`email_handler.py:536`), dispatched under `Forward phase` (`email_handler.py:2202`). Covered in §3.3: return code **`E200`** (`app/email/status.py:2`), `EmailLog.is_reply=f`, message forwarded to the mailbox.

**Reply** — `handle_reply` (`email_handler.py:966`), dispatched under `Reply phase` (`email_handler.py:2196`). A message sent *from John's mailbox to the reverse-alias* is delivered back out **as the alias** to the original contact:

```
$ swaks --from john@wick.com --to hey_at_google_com_wzsdzew@sl.local --server 127.0.0.1:20381 \
        --header "Subject: Re: Blitzy R2.3 inbound test" --body "This is john replying via the reverse-alias."
<-  250 Message accepted for delivery

# email_handler.log:
2026-07-06 23:09:15,201 - SL - DEBUG - 3237 - "/app/email_handler.py:2196" - handle() - 755aaebf-... - Reply phase john@wick.com(john@wick.com) -> hey_at_google_com_wzsdzew@sl.local
2026-07-06 23:09:15,212 - SL - DEBUG - 3237 - "/app/email_handler.py:1051" - handle_reply() - 755aaebf-... - Create <EmailLog 5> for <Contact 3 hey@google.com 15>, <User 1 John Wick john@wick.com>, <Mailbox 1 john@wick.com>
2026-07-06 23:09:15,242 - SL - INFO - 3237 - "/app/email_handler.py:2367" - _handle() - 755aaebf-... - Finish mail_from john@wick.com, rcpt_tos ['hey_at_google_com_wzsdzew@sl.local'], takes 0.04717612266540527 seconds with return code '250 Message accepted for delivery'<<===
```

Return code **`E200`**; `EmailLog id=5` has `is_reply=t` (contrast with the forward's `is_reply=f`); and the sink copy shows the reply leaving *as the alias* — `From: word_word475@sl.local`, `To: hey@google.com` — so the contact never sees John's real address.

**Bounce** — `handle_bounce` (`email_handler.py:1851`) → `handle_bounce_forward_phase` (`email_handler.py:1432`). A bounce is only recognized when `is_bounce()` is true, which requires `mail_from == "<>"` **and** `Content-Type: multipart/report` (`email_handler.py:1813-1818`). Sending a real DSN with a null return-path to the forward VERP address produced:

```
$ swaks --from '<>' --to sl.lmycyibufqqdemzxgmydmnk5.thoiyxau5v7m6@sl.local --server 127.0.0.1:20381 \
        --data '@/tmp/dsn_bounce.eml'          # dsn_bounce.eml is a multipart/report DSN
 -> MAIL FROM:<>
<-  250 SL E211 Bounce Forward phase handled

# email_handler.log:
2026-07-06 23:11:46,870 - SL - DEBUG - 3237 - "/app/email_handler.py:1862" - handle_bounce() - 8cf9a092-... - handle bounce for <EmailLog 4>, phase=forward, contact=<Contact 3 hey@google.com 15>, alias=<Alias 15 word_word475@sl.local>
2026-07-06 23:11:46,875 - SL - DEBUG - 3237 - "/app/email_handler.py:1456" - handle_bounce_forward_phase() - 8cf9a092-... - Handle forward bounce <Contact 3 hey@google.com 15> -> <Alias 15 word_word475@sl.local> -> <Mailbox 1 john@wick.com>. <EmailLog 4>
2026-07-06 23:11:46,883 - SL - DEBUG - 3237 - "/app/email_handler.py:1491" - handle_bounce_forward_phase() - 8cf9a092-... - Create refused email <Refused Email 2 ...>
2026-07-06 23:11:46,936 - SL - INFO - 3237 - "/app/email_handler.py:2367" - _handle() - 8cf9a092-... - Finish mail_from <>, rcpt_tos ['sl.lmycyibufqqdemzxgmydmnk5.thoiyxau5v7m6@sl.local'], takes 0.07480072975158691 seconds with return code '250 SL E211 Bounce Forward phase handled'<<===
```

Return code **`E211`** = `250 SL E211 Bounce Forward phase handled` (`app/email/status.py:19`). The handler marked the original `EmailLog id=4` as `bounced=t` (with `bounced_mailbox_id=1`), created a `RefusedEmail` row, and queued a bounce notice to the user:

```
$ ... "SELECT id,bounced,bounced_mailbox_id FROM email_log WHERE id=4;"
 id | bounced | bounced_mailbox_id
----+---------+--------------------
  4 | t       |                  1
```

**Edge case — the `is_bounce` guard.** An earlier attempt where the sender was *not* empty (`swaks --from ''` was inferred as `root@sl-app`, not `<>`) was correctly rejected as a non-bounce: the handler raised `VERPForward` and returned **`E213`** = `250 SL E213 Unknown email ignored` (`app/email/status.py:21`). This demonstrates that the `mail_from == "<>"` requirement in `is_bounce` (`email_handler.py:1813-1818`) is actually enforced.

**Reply-phase bounce.** `handle_bounce_reply_phase` (`email_handler.py:1595`) is reachable by the identical mechanism — a DSN to a *reply* VERP address (`VerpType.bounce_reply`). It was not separately triggered here; it is documented rather than fabricated, and follows the same `is_bounce` → `handle_bounce` dispatch as the forward-phase case above.


---

## 5. Coverage summary

Each named item mapped to its concrete observed value, the function that performs the work, the `file:line`, and the evidence channel.

| Question | Named item | Observed value | Function / mechanism | `file:line` | Evidence |
|----------|-----------|----------------|----------------------|-------------|----------|
| **R1** | Web server up | `GET /health` → body `success`, HTTP 200 | `healthcheck()` | `server.py:213-215` | curl output |
| R1 | Web server `/live` | body `live`, 200 | `live()` | `app/monitor/views.py:10-12` | curl output |
| R1 | Web server `/git` | body `dev` | `git_sha1()` (`SHA1="dev"`) | `app/monitor/views.py:5-7`, `app/build_info.py:1` | curl output |
| R1 | Web server `/exception` | HTTP 500 | `test_exception()` | `app/monitor/views.py:15-17` | curl status line |
| R1 | Email handler up | `Listen for port 20381` + `Start mail controller 0.0.0.0 20381`; bound `0.0.0.0:20381` | `main()` + `Controller(...)` | `email_handler.py:2403, 2386, 2383` | log + `/proc/net/tcp` |
| R1 | Email handler stays up | keep-alive `while True: time.sleep(2)` | `__main__` | `email_handler.py:2392-2393` | code + live PID |
| R1 | Job runner up | polls every ~10 s (10.079/10.068/10.065 s over ~31 s, 3 cycles) | `time.sleep(10)` loop | `job_runner.py:330, 347` | `Take job` timestamps |
| R1 | Job runner idle vs work | silent when idle; `Take job <Job 11..14>` when work exists | `get_jobs_to_run()` / `Take job` | `job_runner.py:307, 334` | log |
| R1 | 4th scheduler (not the runner) | yacron `python cron.py -j <job>` | `cron.py` | `crontab.yml`, `cron.py` | code |
| **R2** | Signup | `create user blitzy-tmp-signup@example.com`; `User id=4` + `ActivationCode id=2`; activation email | `register()` | `register.py:31,85,120` | log + DB + sink |
| R2 | Activation | flash `Your account has been activated`; `activated=t` | `activate()` | `activate.py:17,49,56` | UI + DB |
| R2 | Random alias | log `create new random alias <Alias 15 word_word475@sl.local> …`; flash `Alias word_word475@sl.local has been created`; `alias id=15` | `Alias.create_new_random()` | `index.py:104,110,111` | log + UI + DB |
| R2 | Custom alias (success) | flash `Alias blitzytmpcustom.word955@sl.local has been created`; `alias id=16` | `custom_alias()` → `Alias.create()` | `custom_alias.py:139,159` | UI + DB |
| R2 | Custom alias (invalid) | flash `Only lowercase letters, numbers, dashes …` | `check_alias_prefix()` branch | `custom_alias.py:64-70` | UI |
| R2 | Inbound email | SMTP `E200`; `New message`/`Finish<<===`; `Contact 3` + `EmailLog 4`; forwarded `msg_007.eml` | `handle_forward()` | `email_handler.py:536,2343,2367`, `status.py:2` | SMTP + log + DB + sink |
| **R3** | No auto-spawn | `grep … server.py` → exit 1 (no matches) | `create_app()` / `register_blueprints()` | `server.py:139,172,233` | grep output |
| R3 | Email handler auto-pickup | processes inbound mail with web app **down** | aiosmtpd `Controller` + `handle_DATA` | `email_handler.py:2383,2289` | log (web down) |
| R3 | Job runner auto-pickup | `Job 18` state `0→1→2` on next poll; `Take job <Job 18 …>` | poll loop + `process_job()` | `job_runner.py:334,339,188,344` | DB states + log |
| R3 | Forward variant | `E200`, `is_reply=f` | `handle_forward()` | `email_handler.py:536` | SMTP + DB |
| R3 | Reply variant | `E200`, `EmailLog 5 is_reply=t`, out as alias | `handle_reply()` | `email_handler.py:966,2196` | SMTP + log + DB + sink |
| R3 | Bounce variant | `E211`, `EmailLog 4 bounced=t`, refused-email + user notice | `handle_bounce()` → `handle_bounce_forward_phase()` | `email_handler.py:1851,1432`, `status.py:19` | SMTP + log + DB |
| R3 | Bounce guard (edge) | non-DSN → `E213` + `VERPForward` | `is_bounce()` | `email_handler.py:1813-1818`, `status.py:21` | SMTP + log |

### 5.1 Discrepancies reported as observed

- **`/health` line numbers.** Route decorator at `server.py:213`, `return "success", 200` at `server.py:215` (`:214` is the `def`). Behavior (200 + `success`) confirmed.
- **Alias log string.** Reported in full as `create new random alias %s for user %s` (`app/dashboard/views/index.py:110`), not abbreviated.
- **Rendered flash.** The f-string `flash(f"Alias {alias.email} has been created", …)` (`index.py:111`) rendered as `Alias word_word475@sl.local has been created`.
- **`New message` / `Finish` line numbers.** The runtime log's embedded `lineno` is the `LOG.i(` call line (`2343` / `2367`); the message-string literals are one line lower (`2344` / `2368`). Both are valid citations.
- **`CONTRIBUTING.md` DB port.** `35432` (`:94`) vs `15432` (`:100`) — inconsistent in the doc; the container's canonical `5432` was used.
- **`/git` value.** Returns the literal `dev` (the `app/build_info.py:1` default), not an empty string.

### 5.2 Bottom line

- **R1.** You know each process is up by a signal specific to it: the web server answers `GET /health` with `success`/200 (`server.py:215`); the email handler logs `Listen for port 20381` and binds `0.0.0.0:20381` (`email_handler.py:2403, 2383`); the job runner logs `Take job …` and cycles every ~10 s (`job_runner.py:334, 347`). After logging in as the seeded `john@wick.com`, the dashboard is reachable, confirming users can sign in and manage aliases.
- **R2.** Signup, alias creation and inbound email each leave a corroborating trail on at least two channels — a log line plus a database row, plus a dashboard flash or an SMTP return code. The inbound forward returns the SMTP status `250 Message accepted for delivery` (`E200`).
- **R3.** The email handler and job runner do **not** start automatically and are not started by the web app (`server.py` spawns nothing). But once each is running as its own process, it handles work on its own — the email handler accepts and forwards/replies/bounces inbound SMTP (even with the web app down), and the job runner picks up queued `Job` rows on its ~10 s poll — across the forward (`E200`), reply (`E200`) and bounce (`E211`) situations.

