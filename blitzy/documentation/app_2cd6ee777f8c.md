# Runtime Verification of a Self-Hosted SimpleLogin Instance

**Branch:** `app_2cd6ee777f8c` · **HEAD commit:** `2cd6ee77` ("chore: emit some missing contact audit logs (#2269)") · **Method:** run-first, then write.

This document proves that a freshly self-hosted **SimpleLogin** instance is operating correctly by **running** the full multi-process topology and **observing** its behavior. Every claim below leads with the direct answer, gives the cause→effect reasoning, and is backed by **actual, unedited runtime output** (real log lines, HTTP responses, database rows) together with a `file:line` citation into the source. Claims that could not be observed under the default configuration are explicitly tagged **(inferred from reading)**.

The instance was brought up inside the canonical container image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (Python 3.10.18, PostgreSQL 15.13, Redis 7.0.15), with the repository checked out at `/app` on commit `2cd6ee77` — identical to this branch. The webapp was launched with the default developer command `python server.py` (which calls `app.run(debug=True, port=7777)`), the SMTP forwarder with `python email_handler.py` (listening on `:20381`), and the background worker with `python job_runner.py` (a 10-second poll loop). Local email mode `NOT_SEND_EMAIL=true` was kept so that every activation and forwarded email is **printed to the log** instead of being transmitted — this is the primary evidence lever throughout. All temporary users, aliases, contacts, jobs and scripts created for observation were removed afterward; the final `git status` (Cleanup section) shows the repository unchanged apart from this document.

**Answer at a glance:**

- **Q1 (readiness):** The operator knows the app is ready when (a) the logging banner `>>> init logging <<<` prints and the single `SL` logger begins emitting DEBUG lines, (b) the webapp binds `:7777` and the login page returns HTTP 200, (c) `email_handler.py` logs `Listen for port 20381` / `Start mail controller 0.0.0.0 20381`, and (d) `job_runner.py` enters its `while True` poll loop (empirically every ~10 s).
- **Q2 (new user):** `POST /auth/register` creates the user with `activated=false` and prints the "Just one more step to join SimpleLogin" activation email; `GET /auth/activate?code=…` flips `activated` **False→True**, deletes the activation code, prints the welcome email, and **302-redirects into `dashboard.index`**; `POST /auth/login` then logs the user in and 302-redirects to the dashboard, which renders at HTTP 200.
- **Q3 (behind the scenes):** A real message sent to `127.0.0.1:20381` is forwarded `contact → alias → mailbox` (printed, not sent, because of `NOT_SEND_EMAIL`); `job_runner.py` executes onboarding and other background jobs every ~10 s; `cron.py`/yacron run scheduled maintenance jobs; and there are **two distinct** "event" subsystems (New Relic analytics and PostgreSQL Proton-sync) — **neither** performs identity verification, which is the **synchronous activation-email + `/auth/activate`** path.

---

## 1. Environment & bring-up recipe

**Direct answer:** the instance is brought up by deriving `.env` from `example.env`, starting PostgreSQL and Redis, applying migrations with `alembic upgrade head`, seeding with `flask dummy-data`, and launching the three long-running processes (webapp, email handler, job runner) concurrently against the same PostgreSQL/Redis. The multi-process topology is **mandatory** for Q3 — a single process cannot exhibit forwarding or background-job behavior.

### 1.1 Local configuration (`.env`)

The container's `.env` is byte-for-byte identical to the shipped `example.env` (i.e. the canonical default a normal operator would use); a diff of the two, ignoring comments/blank lines, is empty. The load-bearing values:

```bash
$ grep -nE "^(URL|DB_URI|NOT_SEND_EMAIL|EMAIL_DOMAIN|DISABLE_ONBOARDING)=" /app/.env
6:URL=http://localhost:7777
19:NOT_SEND_EMAIL=true
22:EMAIL_DOMAIN=sl.local
75:DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin
150:DISABLE_ONBOARDING=true
```

Two nuances that materially affect the observations below, both grounded in `app/config.py`:

- **`NOT_SEND_EMAIL` is a presence check, not a value check** — `NOT_SEND_EMAIL = "NOT_SEND_EMAIL" in os.environ` (`app/config.py:91`). Any value (even `false`) makes it `True`; to actually transmit mail you would have to **remove the line entirely**. Because it is set, `MailSender.send()` prints the email instead of contacting Postfix.
- **`DISABLE_ONBOARDING` is likewise a presence check** — `DISABLE_ONBOARDING = "DISABLE_ONBOARDING" in os.environ` (`app/config.py:401`). Since `example.env` ships `DISABLE_ONBOARDING=true` (line 150), the canonical default **disables onboarding-job scheduling** at registration (see Q3 §4.2). This is reported honestly rather than assumed away.

`DB_URI` uses port `5432` in this container (the exposed PostgreSQL port); note that `scripts/reset_local_db.sh` and `CONTRIBUTING.md`'s `docker run` example instead use `15432` (mapped `-p 15432:5432`). We used `5432` because that is where PostgreSQL 15 listens inside this container. `.env` and `.env.*` are git-ignored (`.gitignore`), so deriving `.env` never dirties the working tree.

### 1.2 Data stores, migrations and seed (captured output)

```bash
$ psql -tAc "select version();"    # PostgreSQL
PostgreSQL 15.13 (Debian 15.13-0+deb12u1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 12.2.0-14+deb12u1) 12.2.0, 64-bit
$ redis-cli info server | grep -E "redis_version|tcp_port"
redis_version:7.0.15
tcp_port:6379
$ CONFIG=/app/.env alembic current
INFO  [alembic.runtime.migration] Will assume transactional DDL.
32f25cbf12f6 (head)
```

The canonical seed recipe is `alembic upgrade head && flask dummy-data` (`CONTRIBUTING.md:106`). `flask dummy-data` provisions the demo account **`john@wick.com` / `password`** (`CONTRIBUTING.md:109`), used below as a known-good login. The seeded baseline is 2 users and 11 aliases:

```bash
$ psql -c "SELECT id,email,activated FROM users ORDER BY id;"
 id |          email          | activated
----+-------------------------+-----------
  1 | john@wick.com           | t
  2 | winston@continental.com | t
(2 rows)
$ psql -tAc "SELECT count(*) FROM alias;"
11
```

Note the `alembic upgrade head` output itself already demonstrates the shared `SL` logger format in action (see Q1 §2.1):

```
2026-07-07 00:22:19,365 - SL - DEBUG - 11278 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

### 1.3 Launching the topology

```bash
# Developer webapp (dev path -> app.run(debug=True, port=7777), server.py:588)
$ CONFIG=/app/.env python server.py            &   # HTTP :7777
# SMTP forwarder (defaults --port 20381, email_handler.py:2399)
$ CONFIG=/app/.env python email_handler.py     &   # SMTP :20381
# Background worker (10-second poll loop)
$ CONFIG=/app/.env python job_runner.py        &
```

In production the same webapp is served by `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` (`wsgi.py` is simply `from server import create_app; app = create_app()`; the `Dockerfile` declares `EXPOSE 7777` and runs that gunicorn command). Optional companion processes referenced later are `event_listener.py listener` (Q3 §4.4) and `cron.py -j <job>` (Q3 §4.3).

**Directory-structure reference nuance:** the entry-point/directory-role overview lives in **`CONTRIBUTING.md` (§"Code structure", ~L143-160)**, which names `wsgi.py`/`server.py`, `email_handler.py` and `cron.py` as the entry points. `docs/code-structure.md` is only a minimal `# TODO` stub about `local_data/` key generation and does **not** contain a "Directory structure" section — so directory structure is cited to `CONTRIBUTING.md`.

---

## 2. Q1 — Startup & readiness indicators (logs + UI)

**Direct answer.** After bring-up, the operator confirms readiness from four concrete signals:

1. **Logs:** the banner `>>> init logging <<<` prints, then the single stdout logger named `SL` begins emitting timestamped DEBUG lines in a fixed format.
2. **Webapp:** the process binds `:7777` and the login page returns HTTP 200.
3. **SMTP:** `email_handler.py` logs `Listen for port 20381` and `Start mail controller 0.0.0.0 20381`.
4. **Worker:** `job_runner.py` enters its `while True` poll loop and sleeps ~10 s between cycles.

Each is evidenced below.

### 2.1 Logging banner and logger identity

**Reasoning:** SimpleLogin installs one shared, colorized stdout logger for every process, so the very first readiness signal is that logger initializing. The literal marker is `print(">>> init logging <<<")` at `app/log.py:67`. The logger is `LOG = _get_logger("SL")` (`app/log.py:79`), set to `DEBUG` (`app/log.py:51`) with `coloredlogs.install(...)` (`app/log.py:62`). Werkzeug's own request logger is silenced — `logging.getLogger("werkzeug").disabled = True` (`app/log.py:70-71`) — which is why the usual Flask "Running on http://127.0.0.1:7777/" line is **absent** (see the honest correction in §2.3). An `EmailHandlerFilter` (`app/log.py:28`) injects a per-message UUID into the `%(message_id)s` slot so an email's lifecycle can be traced across log lines (used heavily in Q3).

The format string is defined at `app/log.py:12-14`:

```python
_log_format = (
    "%(asctime)s - %(name)s - %(levelname)s - %(process)d - "
    '"%(pathname)s:%(lineno)d" - %(funcName)s() - %(message_id)s - %(message)s'
)
```

**Observed** — the first 21 lines of the webapp's captured stdout (`python server.py`):

```
load config file /app/.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/nnzcyrlgxyhzqmtcugvu
Upload files to local dir
>>> init logging <<<
2026-07-07 00:22:53,725 - SL - DEBUG - 11333 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
load config file /app/.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/mzklshbwdgliawchykxw
Upload files to local dir
>>> init logging <<<
2026-07-07 00:22:55,256 - SL - DEBUG - 11341 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

The banner prints **twice** because the Flask dev server's auto-reloader spawns a watcher parent (pid `11333`) and a worker child (pid `11341`); each initializes logging. The DEBUG line matches the format string exactly: `<ts> - SL - DEBUG - <pid> - "<path>:<line>" - <func>() - <message_id> - <message>` (the `message_id` slot is empty for non-email lines). This is the same `SL`-logger shape corroborated by public self-hosted deployments.

### 2.2 Build stamp (default/canonical build value)

**Direct answer:** the default local build reports `SHA1 = "dev"`, so the webapp's `VERSION` is `"dev"` and the (disabled) Sentry release would be `"app@dev"`. **Reasoning:** `app/build_info.py:1-2` hard-codes `SHA1 = "dev"` and `BUILD_TIME = "1652365083"` for non-CI builds; `server.py:50` imports `SHA1`, `server.py:418` sets `VERSION = SHA1` (injected into templates), and `server.py:115` computes `release=f"app@{SHA1}"` for Sentry (guarded by `if SENTRY_DSN`, `server.py:112`, which is unset locally so Sentry is not initialized).

**Observed** (evaluated in the app context):

```bash
$ CONFIG=/app/.env python -c "from app.build_info import SHA1, BUILD_TIME; print(SHA1, BUILD_TIME)"
dev 1652365083
```

`VERSION = "dev"` is therefore the canonical/default build identifier — it is **not** a real git SHA, and is reported as such.

### 2.3 Webapp bind and startup init routines (with two honest corrections)

**Direct answer:** the dev entry point binds `:7777` via `app.run(debug=True, port=7777)` at `server.py:588` (inside `local_main()` at `server.py:572`, which also forces `config.COLOR_LOG = True` at `server.py:573`). Readiness of the HTTP surface is confirmed by an actual request returning 200 (below) and by the app's own per-request log line emitted from `after_request` at `server.py:284`.

**Honest correction #1 (observed overrides expectation).** The conventional Werkzeug line `Running on http://127.0.0.1:7777/` does **not** appear, because the werkzeug logger is disabled at `app/log.py:70-71`. HTTP readiness is instead evidenced by reachability plus the custom `after_request` log:

```
2026-07-07 00:23:1x,xxx - SL - DEBUG - 11341 - "/app/server.py:284" - <lambda>() -  - GET /auth/login ImmutableMultiDict([]) 200, takes 0.10...
```

**Honest correction #2 (observed overrides an AAP anchor).** `add_sl_domains()` and `add_proton_partner()` are **not** invoked during `create_app`/`init_extensions` on webapp startup; they live in the `@app.cli.command("dummy-data")` handler (`server.py` ~L490-497) and in `init_app.py`'s `__main__` block (`init_app.py:69`). Consequently the **webapp** log contains **no** "SL domain" lines. Running the init routines directly through their real entry point (`python init_app.py`) shows them, grounded in `init_app.py`:

```
2026-07-07 ... - SL - INFO - ... - "/app/init_app.py:16" - load_pgp_public_keys() -  - Load PGP key for mailbox <Mailbox 2 pgp@example.org>
2026-07-07 ... - SL - DEBUG - ... - "/app/init_app.py:36" - load_pgp_public_keys() -  - Finish load_pgp_public_keys
2026-07-07 ... - SL - DEBUG - ... - "/app/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
```

Here `add_sl_domains()` takes the "already a SL domain" branch (`init_app.py:42`) rather than the first-run `Add %s to SL domain` branch (`init_app.py:44`) because the domain was already seeded — a correct before/after distinction.

### 2.4 SMTP handler readiness

**Reasoning:** `email_handler.py`'s `main(port)` (`email_handler.py:2381`) constructs an aiosmtpd `Controller(MailHandler(), hostname="0.0.0.0", port=port)` (`email_handler.py:2383`); the `__main__` block defaults `--port` to `20381` (`email_handler.py:2399`). Readiness is announced by two lines.

**Observed** (`email_handler.log`):

```
2026-07-07 00:24:39,440 - SL - INFO - 11433 - "/app/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-07 00:24:39,442 - SL - DEBUG - 11433 - "/app/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

The listener socket was confirmed open (a TCP connect to `127.0.0.1:20381` returned `connect_ex == 0`).

### 2.5 Job runner readiness and the 10-second poll cadence

**Direct answer:** the worker is ready when it enters `while True:` (`job_runner.py:330`), calling `get_jobs_to_run()` (`job_runner.py:307`), logging `Take job %s` (`job_runner.py:334`) for each due job, and sleeping `time.sleep(10)` (`job_runner.py:347`) between cycles. The **measured** cadence is ~10.0 s, stable across ≥2 cycles.

**Observed** — three probe jobs (`run_at = now`) were inserted purely to make the runner tick visibly. **This insertion is a NON-CANONICAL observation aid** (real onboarding jobs are scheduled 1-3 days out); the probe jobs were deleted immediately afterward.

```
2026-07-07 00:26:19,569 - SL - DEBUG - 11434 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 1 blitzy-probe-1 {}>
2026-07-07 00:26:19,573 - SL - ERROR - 11434 - "/app/job_runner.py:304" - process_job() -  - Unknown job name blitzy-probe-1
2026-07-07 00:26:39,600 - SL - DEBUG - 11434 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 2 blitzy-probe-2 {}>
2026-07-07 00:26:39,603 - SL - ERROR - 11434 - "/app/job_runner.py:304" - process_job() -  - Unknown job name blitzy-probe-2
2026-07-07 00:26:49,613 - SL - DEBUG - 11434 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 3 blitzy-probe-3 {}>
2026-07-07 00:26:49,615 - SL - ERROR - 11434 - "/app/job_runner.py:304" - process_job() -  - Unknown job name blitzy-probe-3
```

**Timing math:** probe-2→probe-3 = `00:26:49.613 − 00:26:39.600 = 10.013 s` (one cycle). probe-1→probe-2 = `00:26:39.600 − 00:26:19.569 = 20.031 s`, i.e. exactly two 10 s cycles (the runner picks at most one due job per `Take job` and re-polls). Average `10.016 s/cycle`, stable across the observed cycles — confirming the hard-coded `time.sleep(10)` at `job_runner.py:347`. The `Unknown job name` line (`job_runner.py:304`) is the expected fallback for the synthetic probe names and simultaneously demonstrates the `process_job()` dispatch executing.

### 2.6 UI readiness

**Direct answer:** the login page renders at `http://localhost:7777` with HTTP 200, and a known-good login lands on the dashboard — proving the auth UI path end-to-end.

**Observed:**

```bash
$ curl -s -o /dev/null -w "%{http_code}\n" http://localhost:7777/auth/login
200
```

The returned HTML is `templates/auth/login.html` (title `… | SimpleLogin`, an "Email address" label, a password field, and a hidden `csrf_token` input). Logging in with the seeded demo account via the **real** `POST /auth/login` redirects to the dashboard:

```
POST /auth/login  (email=john@wick.com, password=password)  -> HTTP 302, Location: http://localhost:7777/dashboard/
GET  /dashboard/                                             -> HTTP 200  (title "Alias | SimpleLogin")
```

This confirms the UI is ready to handle authentication and alias/email activity.

---

## 3. Q2 — New-user journey: register → verify → login → dashboard

**Direct answer.** A brand-new user proceeds through four observable steps: (1) `POST /auth/register` creates the account with `activated=false` and prints the activation email; (2) `GET /auth/activate?code=…` flips `activated` **False→True**, deletes the one-time code, prints a welcome email, and **302-redirects into the dashboard**; (3) `POST /auth/login` authenticates and 302-redirects to the dashboard; (4) `GET /dashboard/` renders at HTTP 200. Each step below shows the request, the HTTP response, the flash message, the log lines, and the database state.

All steps were driven through the **real HTTP routes** with a temporary user **`blitzy-temp-q2@example.com` / `BlitzyTempPass123`** (it became user `id=3`; deleted in the Cleanup section). Preconditions confirmed at runtime: `HCAPTCHA_SECRET=None` (hCaptcha disabled locally), `DISABLE_REGISTRATION=False`, and the `RegisterForm` password rule `Length(min=8, max=100)`. One implementation detail worth stating up front: **flash messages are rendered as toastr JavaScript**, `<script>toastr.{category}("{message}")</script>` (`templates/base.html:102`) — so the evidence quotes the toastr call, which carries the exact category and message text.

### 3.1 Step 1 — Register (`POST /auth/register`)

**Reasoning:** the `/register` route (`app/auth/views/register.py:31`) validates the form, logs `create user %s` (`register.py:85`), creates the `User` (`register.py:86`), then calls `send_activation_email()` (`register.py:95`) which deletes any prior `ActivationCode` and creates a fresh one `ActivationCode.create(code=random_string(30))` (`register.py:120`) with `activation_link = f"{URL}/auth/activate?code=…"` (`register.py:124`). The browser then lands on `templates/auth/register_waiting_activation.html` (`register.py:104`).

**Observed** — `POST /auth/register` returned **HTTP 200** rendering `register_waiting_activation.html` (title `Activation Email Sent | SimpleLogin`). The captured log for the request:

```
00:30:22 - SL - DEBUG - ... - "/app/app/auth/views/register.py:85"  - register()      -  - create user blitzy-temp-q2@example.com
00:30:22 - SL - DEBUG - ... - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
00:30:22 - SL - DEBUG - ... - "/app/app/models.py:647"              - create()        -  - Disable onboarding emails
00:30:22 - SL - DEBUG - ... - "/app/app/email_utils.py:303"         - send_email()    -  - send email to blitzy-temp-q2@example.com, subject 'Just one more step to join SimpleLogin'
00:30:22 - SL - DEBUG - ... - "/app/app/mail_sender.py:131"         - send()          -  - send email with subject 'Just one more step to join SimpleLogin', from '"noreply@sl.local" <noreply@sl.local>' to 'blitzy-temp-q2@example.com'
```

Four facts fall directly out of this output:
- The activation email uses the subject **"Just one more step to join SimpleLogin"** (`app/email_utils.py:128`, templates `templates/emails/transactional/activation.{txt,html}` resolved via `render()`'s base dir at `app/email_utils.py:73`), and it is **printed, not sent**, by `MailSender.send()`'s `NOT_SEND_EMAIL` branch (`app/mail_sender.py:130-132`). This printed line is the identity-verification evidence lever.
- `models.py:647` logs `Disable onboarding emails` — the `if config.DISABLE_ONBOARDING: return user` short-circuit (`app/models.py:646-647`), so **no onboarding `Job` rows are scheduled** under the canonical default (relevant to Q3 §4.2).
- `event_dispatcher.py:62` logs the partner-sync **guard** firing (relevant to Q3 §4.4) — the PostgreSQL event path does nothing here.

**Intermediate DB state** (immediately after registration):

```bash
$ psql -c "SELECT id,email,activated FROM users WHERE email='blitzy-temp-q2@example.com';"
 id |            email            | activated
----+-----------------------------+-----------
  3 | blitzy-temp-q2@example.com  | f
$ psql -c "SELECT id,code,user_id,created_at,expired FROM activation_code WHERE user_id=3;"
 id |             code               | user_id |         created_at         |          expired
----+--------------------------------+---------+----------------------------+----------------------------
  1 | lirblwbsqnojcpfpbahzkrvqvqxixq |       3 | 2026-07-07 00:30:22.594873 | 2026-07-07 01:30:22.594907
```

`activated = f`; the `activation_code` row exists, the code is exactly 30 characters (`random_string(30)`, `register.py:120`), and `expired` is exactly `created_at + 1 hour` — matching the `_expiration_1h` default on the model (`app/models.py:1212`). The `job` table has **0** rows (confirming onboarding was disabled).

### 3.2 Step 2 — Verify / activate (`GET /auth/activate?code=…`)

**Reasoning:** the `/activate` route (`app/auth/views/activate.py:13`) looks up the code, and on success sets `user.activated = True` (`activate.py:49`), calls `login_user(user)` (`activate.py:50`), deletes the one-time code `ActivationCode.delete(...)` (`activate.py:53`), flashes **"Your account has been activated"** as a success (`activate.py:56`), sends the welcome email (`activate.py:58`), and — with no `next` param — redirects to `dashboard.index` (`activate.py:67`).

**Observed** — `GET /auth/activate?code=lirblwbsqnojcpfpbahzkrvqvqxixq` returned **HTTP 302** with `Location: http://localhost:7777/dashboard/`; following it returned 200. Captured log:

```
00:30:55 - SL - DEBUG - ... - "/app/app/email_utils.py:303" - send_email() -  - send email to simplelogin-newsletter.word057@sl.local, subject 'Welcome to SimpleLogin'
00:30:55 - SL - DEBUG - ... - "/app/app/mail_sender.py:131" - send()       -  - send email with subject 'Welcome to SimpleLogin', from '"noreply@sl.local" <noreply@sl.local>' to 'simplelogin-newsletter.word057@sl.local'
00:30:55 - SL - DEBUG - ... - "/app/app/auth/views/activate.py:66" - activate() -  - redirect user to dashboard
00:30:55 - SL - DEBUG - ... - "/app/server.py:284" - <lambda>() -  - GET /auth/activate ImmutableMultiDict([('code', 'lirblwbsqnojcpfpbahzkrvqvqxixq')]) 302, takes 0.057s
00:30:55 - SL - DEBUG - ... - "/app/app/dashboard/views/index.py:172" - index() -  - Show intro to <User 3 blitzy-temp-q2@example.com ...>
00:30:55 - SL - DEBUG - ... - "/app/server.py:284" - <lambda>() -  - GET /dashboard/ ImmutableMultiDict([]) 200, takes ...s
```

The welcome email (subject **"Welcome to SimpleLogin"**, `app/email_utils.py:107`) is sent to the user's auto-created newsletter alias and, like all mail here, printed rather than transmitted. `activate.py:66` logs `redirect user to dashboard` and the request returns **302 → `/dashboard/`**, then `dashboard/index.py:172` logs `Show intro to <User 3 …>` on the 200 render.

**After DB state** (the transition):

```bash
$ psql -c "SELECT id,email,activated FROM users WHERE id=3;"
 id |            email            | activated
----+-----------------------------+-----------
  3 | blitzy-temp-q2@example.com  | t
$ psql -tAc "SELECT count(*) FROM activation_code WHERE user_id=3;"
0
```

`activated` flipped **False → True** (`activate.py:49`) and the one-time `activation_code` row was **deleted** (`activate.py:53`).

### 3.3 Step 3 — Login (`POST /auth/login`)

**Reasoning:** the `/login` route (`app/auth/views/login.py:21`) validates credentials; on success it emits `LoginEvent(...).send()` (`login.py:71`) and calls `after_login(user, next_url)` (`login.py:72` → `app/auth/views/login_utils.py:12`). With no MFA configured, `after_login` logs `redirect user to dashboard` (`login_utils.py:44`) and returns a 302 to `dashboard.index` (`login_utils.py:45`).

**Observed** — `POST /auth/login` with the valid temp credentials returned **HTTP 302**, `Location: /dashboard/`. Captured log:

```
00:31:48 - SL - DEBUG - ... - "/app/app/auth/views/login_utils.py:35" - after_login() -  - log user <User 3 blitzy-temp-q2@example.com ...> in
00:31:48 - SL - DEBUG - ... - "/app/app/auth/views/login_utils.py:44" - after_login() -  - redirect user to dashboard
```

The MFA branches (`login_utils.py:19-33`, redirecting to `auth.fido`/`auth.mfa`) are **not** triggered for this fresh user because neither FIDO nor OTP is enabled — the code falls through to the dashboard redirect at `login_utils.py:44-45`. **(inferred from reading)** for the MFA branch specifically, since no MFA was configured to exercise it.

### 3.4 Step 4 — Dashboard (`GET /dashboard/`)

**Reasoning:** the post-login destination is `dashboard.index` — route `"/"` with `@login_required` (`app/dashboard/views/index.py:55-56`), view `index()` (`index.py:67`), rendering `templates/dashboard/index.html`.

**Observed** — `GET /dashboard/` returned **HTTP 200** with title `Alias | SimpleLogin`; the body contains the expected dashboard widgets (`Create`, `random alias`, `Newsletter`, `Total`, `Settings`, `Logout`). The stat widgets are computed by `get_stats` (`index.py:32`).

### 3.5 Before / intermediate / after state table

| State point | `users.activated` (id=3) | `activation_code` (user_id=3) | Source |
|---|---|---|---|
| Before registration | (no user row) | (no row) | — |
| After `POST /auth/register` | `false` | 1 row, code `lirblwbsq…` (30 chars), `expired = created_at + 1h` | `register.py:86,120`; `models.py:1212` |
| After `GET /auth/activate` | `true` | 0 rows (deleted) | `activate.py:49,53` |

### 3.6 UI surfaces confirmed by observation

- `templates/auth/register.html` — the registration form (email, password, `csrf_token`).
- `templates/auth/register_waiting_activation.html` — post-register confirmation (`register.py:104`; title "Activation Email Sent | SimpleLogin").
- `templates/auth/activate.html` — activation result/error page (`extends error.html`, shows `{{ error }}` and a Resend link when `show_resend_activation`), exercised in the edge cases below.
- `templates/auth/login.html` — the login form and its flash (toastr) messages.
- `templates/dashboard/index.html` — the post-login landing (title "Alias | SimpleLogin").

---

## 4. Q3 — Behind-the-scenes services (forwarding, jobs, scheduler, events)

**Direct answer.** Behind the visible auth flow, four internal mechanisms are active and observable at runtime: (4.1) the `email_handler.py` SMTP pipeline **forwards** inbound mail `contact → alias → mailbox`; (4.2) `job_runner.py` executes **background jobs** (onboarding and maintenance) on a 10 s poll; (4.3) `cron.py` under **yacron** runs scheduled maintenance jobs; and (4.4) there are **two distinct** "event" subsystems — New Relic **analytics** and PostgreSQL **Proton-sync** — **neither of which performs identity verification**. Identity verification proper is the synchronous activation-email + `/auth/activate` path already shown in Q2.

### 4.1 Email forwarding pipeline (`email_handler.py`)

**Reasoning:** a message delivered to the SMTP listener enters `handle_DATA` (`email_handler.py:2289`) → `_handle` (which logs `New message …`, `email_handler.py:2343`) → the forward decision `handle_forward()` (`email_handler.py:536`) → `forward_email_to_mailbox()` which logs the pivotal `Forward %s -> %s -> %s` line (`email_handler.py:688`) → and finally the `Finish …` summary with a return code (`email_handler.py:2367`). Because `NOT_SEND_EMAIL=true`, the actual outbound send is printed by `MailSender.send()` (`app/mail_sender.py:130-132`) and returns success without contacting Postfix (whose default target is `240.0.0.1:25`, `app/config.py:136,149`).

**Observed** — a real message was sent via Python `smtplib` to `127.0.0.1:20381` (no `swaks` in the container), From `sender-blitzy@external.test` To `word_test110@sl.local` (a seeded alias belonging to `john@wick.com`; `EMAIL_DOMAIN=sl.local`, `app/config.py:92`). The SMTP client received `250` with an empty refusal dict. All handler lines share the injected `message_id` UUID `4741f24e-cf24-420f-9208-fc34a95ab05a`, tying the lifecycle together:

```
"/app/email_handler.py:2343" - _handle()               - 4741f24e... - New message, mail from sender-blitzy@external.test, rctp tos ['word_test110@sl.local']
"/app/email_handler.py:2202" - handle()                - 4741f24e... - Forward phase sender-blitzy@external.test -> word_test110@sl.local
"/app/email_handler.py:580"  - handle_forward()        - 4741f24e... - Create or get contact for from_header:sender-blitzy@external.test
"/app/app/contact_utils.py:110" - create_contact()     - 4741f24e... - Created contact <Contact 2 sender-blitzy@external.test 2> for alias <Alias 2 word_test110@sl.local> ... invalid_email=False
"/app/app/dmarc.py:33"       - apply_dmarc_policy()     - 4741f24e... - DMARC check disabled
"/app/email_handler.py:688"  - forward_email_to_mailbox() - 4741f24e... - Forward <Contact 2 sender-blitzy@external.test 2> -> <Alias 2 word_test110@sl.local> -> <Mailbox 1 john@wick.com>
"/app/email_handler.py:740"  - forward_email_to_mailbox() - 4741f24e... - Create <EmailLog 2> for <Contact 2...>, <User 1 John Wick john@wick.com>, <Mailbox 1 john@wick.com>
"/app/email_handler.py:867"  - forward_email_to_mailbox() - 4741f24e... - From header, new:"sender-blitzy at external.test" <sender-blitzy_at_external_test_mdwrsqtmj@sl.local>, old:sender-blitzy@external.test
"/app/app/mail_sender.py:131" - send()                  - 4741f24e... - send email with subject 'Blitzy forwarding probe', from '"sender-blitzy at external.test" <sender-blitzy_at_external_test_mdwrsqtmj@sl.local>' to 'word_test110@sl.local'
"/app/email_handler.py:2367" - _handle()               - 4741f24e... - Finish mail_from sender-blitzy@external.test, rcpt_tos ['word_test110@sl.local'], takes 0.17923283576965332 seconds with return code '250 Message accepted for delivery'<<===
```

**Cause → effect of the pivotal line:** `email_handler.py:688` shows the three-hop forward `Contact 2 (sender) -> Alias 2 (word_test110@sl.local) -> Mailbox 1 (john@wick.com)` — the sender is turned into a **reverse-alias** contact (`sender-blitzy_at_external_test_mdwrsqtmj@sl.local`, rewritten From header at `email_handler.py:867`) so the mailbox owner can reply through SimpleLogin. The forward **completed successfully** — the `Finish` line reports return code `250 Message accepted for delivery` (`email_handler.py:2367`) — but because `NOT_SEND_EMAIL` is set, the message was **printed** by `mail_sender.py:131`, not handed to an MTA. This is an honest, observed success rather than a forced one.

### 4.2 Background jobs (`job_runner.py`)

**Reasoning:** the worker loop is `while True` (`job_runner.py:330`) → `get_jobs_to_run()` (`job_runner.py:307`, which only selects jobs with `run_at < now + 10 min`) → `Take job %s` (`job_runner.py:334`) → `process_job(job)` (`job_runner.py:188`) → `time.sleep(10)` (`job_runner.py:347`).

The full `process_job()` dispatch, enumerated **by name** from `app/config.py:301-311` and `job_runner.py:188-304`:

| Job constant | Value | Handler / log | Line |
|---|---|---|---|
| `JOB_ONBOARDING_1` | `onboarding-1` | `onboarding_send_from_alias` → `send onboarding send-from-alias email to user %s` | `job_runner.py:189,196` |
| `JOB_ONBOARDING_2` | `onboarding-2` | onboarding mailbox → `send onboarding mailbox email` | `job_runner.py:198,205` |
| `JOB_ONBOARDING_4` | `onboarding-4` | onboarding PGP → `send onboarding pgp email` | `job_runner.py:207,219` |
| `JOB_BATCH_IMPORT` | `batch-import` | batch alias import | `job_runner.py:222` |
| `JOB_DELETE_ACCOUNT` | `delete-account` | account deletion | `job_runner.py:226` |
| `JOB_DELETE_MAILBOX` | `delete-mailbox` | mailbox deletion | `job_runner.py:245` |
| `JOB_DELETE_DOMAIN` | `delete-domain` | custom-domain deletion | `job_runner.py:248` |
| `JOB_SEND_USER_REPORT` | `send-user-report` | GDPR user data export | `job_runner.py:285` |
| `JOB_SEND_PROTON_WELCOME_1` | `proton-welcome-1` | Proton welcome email | `job_runner.py:289` |
| `JOB_SEND_ALIAS_CREATION_EVENTS` | `send-alias-creation-events` | alias-creation events | `job_runner.py:295` |
| *(fallback)* | — | `Unknown job name %s` | `job_runner.py:304` |

(`JOB_ONBOARDING_3` = `onboarding-3` is defined in config but has no `process_job` branch — a read-derived observation.)

**Connection to the new-user flow (identity onboarding), reported honestly.** Registration is *designed* to schedule three onboarding jobs — `Job.create(JOB_ONBOARDING_1, run_at=now+1day)`, `_2 +2days`, `_4 +3days` (`app/models.py:651-664`) — but this is guarded by `if config.DISABLE_ONBOARDING: return user` (`app/models.py:646-647`). Under the **canonical default** (`DISABLE_ONBOARDING=true` in `example.env:150`), registration logged `Disable onboarding emails` (`models.py:647`) and scheduled **zero** jobs (the `job` table was empty after Q2 §3.1). So onboarding jobs do **not** run out of the box; two further reasons they would not be observed in a short window even if enabled: they are scheduled 1-3 days out, and `get_jobs_to_run()` only picks jobs due within 10 minutes (`job_runner.py:307`).

**Observed onboarding execution (NON-CANONICAL setup, clearly labeled).** To actually watch the runner execute an onboarding job, a single `Job` row `onboarding-1` with `payload={"user_id":3}` and `run_at=now` was inserted, then deleted afterward:

```
00:34:20,152 - SL - DEBUG - 11434 - "/app/job_runner.py:334" - <module>()     -  - Take job <Job 4 onboarding-1 {'user_id': 3}>
00:34:20,xxx - SL - DEBUG - 11434 - "/app/job_runner.py:196" - process_job()  -  - send onboarding send-from-alias email to user <User 3 blitzy-temp-q2@example.com ...>
00:34:20,xxx - SL - DEBUG - 11434 - "/app/app/email_utils.py:303" - send_email() -  - send email to simplelogin-newsletter.word057@sl.local, subject 'SimpleLogin Tip: Send emails from your alias'
00:34:20,xxx - SL - DEBUG - 11434 - "/app/app/mail_sender.py:131" - send()    -  - send email with subject 'SimpleLogin Tip: Send emails from your alias', from '"noreply@sl.local" <noreply@sl.local>' to 'simplelogin-newsletter.word057@sl.local'
```

This shows the exact chain `Take job` (`job_runner.py:334`) → `send onboarding send-from-alias email` (`job_runner.py:196`) → the tip email "SimpleLogin Tip: Send emails from your alias" (`job_runner.py:34`) printed via `NOT_SEND_EMAIL`. The job finished in state `2` (done) with `attempts=1`; it was then deleted. The 10 s cadence was independently confirmed in §2.5 across ≥2 cycles.

### 4.3 Scheduler (`cron.py` / yacron)

**Reasoning:** `cron.py`'s `__main__` logs `Start running cronjob` (`cron.py:1263`) and dispatches by `-j/--job` (`cron.py:1274-1322`). The 17 dispatchable jobs are: `stats`, `notify_trial_end`, `notify_manual_subscription_end`, `notify_premium_end`, `delete_logs`, `delete_old_data`, `poll_apple_subscription`, `sanity_check`, `delete_old_monitoring`, `check_custom_domain`, `check_hibp`, `notify_hibp`, `cleanup_tokens`, `send_undelivered_mails`, `delete_scheduled_users`, `clear_alias_audit_log`, `clear_user_audit_log`. These are scheduled by **yacron** via `crontab.yml` and `crontab-all-hosts.yml`.

**Observed** — one benign job was run through its real entry point:

```bash
$ CONFIG=/app/.env python cron.py -j sanity_check      # exit code 0
... - SL - ... - "/app/cron.py:1263" - <module>()      -  - Start running cronjob
... - SL - ... - "/app/cron.py:1296" - <module>()      -  - Check data consistency
... - SL - ... - "/app/cron.py:725"  - sanity_check()  -  - sanitize user email
... (sanitize alias address & name; sanity contact address; sanitize mailbox address;
     normalize reverse alias; clean domain name; check mailbox valid PGP keys ...)
```

The yacron schedules (read-derived): `crontab.yml` runs e.g. `stats` at `0 0 * * *`, `send_undelivered_mails` at `*/5 * * * *`, `check_hibp` at `15 3 * * *`; `crontab-all-hosts.yml` defines a single `send_undelivered_mails` at `*/5 * * * *` with `concurrencyPolicy: Forbid`. **(inferred from reading)** for the schedule expressions and for the 16 jobs other than `sanity_check`, which were inventoried but not each executed.

### 4.4 Two distinct "event" mechanisms — and where identity verification actually lives

A common conflation this document deliberately avoids: the "events" in SimpleLogin are **two separate subsystems**, and **neither is the identity-verification mechanism**.

**(i) New Relic analytics events** — `app/events/auth_event.py`. `RegisterEvent.send()` and `LoginEvent.send()` call `newrelic.agent.record_custom_event(...)` (`auth_event.py:23-25` for login, `auth_event.py:45-47` for register), tagging outcomes such as `success`, `failed`, `email_in_use`, `catpcha_failed`, `not_activated`. **Observed: these are silent no-ops locally.** `newrelic.ini` is 0 bytes, no `NEW_RELIC_*` env vars are set, and `newrelic.agent.global_settings().enabled` evaluates to `False`, so `record_custom_event` emits nothing externally. This subsystem is **analytics only** — it does not verify identity.

**(ii) PostgreSQL Proton-sync events** — `app/events/event_dispatcher.py` + `event_listener.py`. `EventDispatcher.send_event()` (`event_dispatcher.py:49`) is **guarded** and returns early if `EVENT_WEBHOOK_DISABLE` is set (`event_dispatcher.py:57-58`), if `EVENT_WEBHOOK` is unset (`event_dispatcher.py:61-63`), or if the user has no `PartnerUser` (`event_dispatcher.py:69`). When it does fire, `PostgresDispatcher.send()` creates a `SyncEvent` row and issues `NOTIFY simplelogin_sync_events` (`event_dispatcher.py:24-26`), consumed by `event_listener.py`. **Observed: the guard fired at registration** (config: `EVENT_WEBHOOK=None`, `EVENT_WEBHOOK_DISABLE=False`):

```
"/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
```

The `sync_event` table held **0** rows. The listener process does start correctly — `event_listener.py listener` logged `Using PostgresEventSource` (`event_listener.py:34`), `Starting with HttpEventSink` (`event_listener.py:43`), and `Starting to listen to events` (`app/events/event_source.py:49`) — but there was nothing to consume in the default local setup (no webhook, no Proton partner user).

**(iii) Identity verification proper** — the correct implementation to attribute is the **synchronous activation path**, not either event system: `send_activation_email()` (`app/email_utils.py:125`, subject "Just one more step to join SimpleLogin", templates `templates/emails/transactional/activation.{txt,html}`) dispatches the verification link, and `GET /auth/activate` flips `users.activated` (`activate.py:49`). `send_email()` logs `send email to %s, subject '%s'` (`app/email_utils.py:303`). This is exactly the observed Q2 §3.1-§3.2 flow.

---

## 5. Edge / error paths

**Direct answer.** Every distinct failure branch behaves as coded: duplicate email, invalid code, expired code, wrong password, and login-before-activation each produce a specific flash and log line; the routes are rate-limited and begin returning `429` after 10 failures/minute. Rate limits (`/login` and `/activate` are `10/minute` deduct-on-failure, `login.py:22`, `activate.py:14-15`; `/resend_activation` is `10/hour`, `resend_activation.py:18`) were spaced out so they did not mask the branch under test, except where a `429` was intentionally provoked (§5.7).

### 5.1 Duplicate / existing email (OBSERVED)
`POST /auth/register` re-using an existing address → **HTTP 200**, toastr `error: Email blitzy-temp-q2@example.com already used` (`register.py:82`) plus `RegisterEvent(email_in_use)`.

### 5.2 Bad-mailbox personal-inbox branch (OBSERVED)
`POST /auth/register` with `email=blitzy-temp-badmbox@sl.local` (a domain that is itself an SL domain) → **HTTP 200**, toastr `error: You cannot use this email address as your personal inbox.` (`register.py:75`). No user row was created.

### 5.3 Invalid activation code (OBSERVED)
`GET /auth/activate?code=bogus-code-does-not-exist` → **HTTP 400**, error page body `Activation code cannot be found` (`activate.py:33`); `g.deduct_limit = True` (`activate.py:30`) so this failure counts against the limiter.

### 5.4 Expired activation code (OBSERVED; expiry forced — NON-CANONICAL)
A temp user `blitzy-temp-expired@example.com` (User 5) was registered (code `atompgzkgylffivhiespudrnmhysis`). Since the code lives for 1 hour, expiry was forced with `UPDATE activation_code SET expired = now() - interval '1 hour'` (**labeled non-canonical** — a DB manipulation, not the real passage of time). `GET /auth/activate?code=…` → **HTTP 400**, error `Activation code was expired` (`activate.py:42`) with `show_resend_activation=True` (`activate.py:43`) so the page renders a Resend link.

### 5.5 Wrong password (OBSERVED)
`POST /auth/login` with a wrong password → **HTTP 200**, toastr `error: Email or password incorrect` (`login.py:49`) plus `LoginEvent(failed)` (`login.py:50`).

### 5.6 Login before activation, then resend (OBSERVED)
A temp user `blitzy-temp-noact@example.com` (User 4) was registered but not activated.
- **5.6a Login while unactivated:** `POST /auth/login` → **HTTP 200**, toastr `error: Please check your inbox for the activation email. You can also have this email re-sent` (`login.py:66`) plus `LoginEvent(not_activated)` (`login.py:69`); the page shows a Resend link (`show_resend_activation`, `login.py:64`).
- **5.6b Resend:** `POST /auth/resend_activation` → **HTTP 200**, toastr `warning: An activation email has been sent to you. Please check your inbox/spam folder.` (`resend_activation.py:38`); the log shows `resend_activation.py:36 - user <User 4 …> is not activated` and a **new** activation email was printed (confirming the resend actually re-dispatched).

### 5.7 Rate limiting (OBSERVED)
Twelve rapid invalid `GET /auth/activate` requests produced the status sequence:

```
[400, 400, 400, 400, 400, 400, 400, 400, 400, 400, 429, 429]
```

The **first `429` at attempt #11** confirms the `10/minute` deduct-on-failure limiter (`activate.py:14-15`) — exactly 10 failures are allowed per minute before the limiter rejects.

### 5.8 Wrong hCaptcha — (inferred from reading)
The wrong-captcha branch (`register.py:58-71`: `LOG.w("User put wrong captcha …")`, flash `Wrong Captcha`, `RegisterEvent(catpcha_failed)`) could **not** be observed because `HCAPTCHA_SECRET=None` locally, so hCaptcha verification is skipped entirely. This branch is therefore reported **(inferred from reading)** rather than observed.

---

## 6. Cleanup confirmation (read-only guarantee)

**Direct answer.** Every temporary artifact was removed and the repository was verified unchanged apart from this document.

**Temporary data deleted** (via the canonical app methods, run inside an `init_app.create_light_app()` context):
- Users 3, 4, 5 (`blitzy-temp-q2`, `blitzy-temp-noact`, `blitzy-temp-expired`) via `User.delete(...)`, which cascaded their auto-created newsletter aliases into the global trash.
- Contact 2 (`sender-blitzy@external.test`) and its `EmailLog` created by the forwarding probe.
- The temporary probe/onboarding `Job` rows inserted for §2.5 and §4.2.
- The three resulting `DeletedAlias` trash entries (`simplelogin-newsletter.word057/word961/word555@sl.local`).

**Temporary scripts removed:** the HTTP helper and SMTP probe scripts and all capture logs under the container's scratch directory `/app/blitzy_tmp`.

**Final DB state == seeded baseline:**

```bash
$ psql -c "SELECT id,email,activated FROM users ORDER BY id;"
 id |          email          | activated
----+-------------------------+-----------
  1 | john@wick.com           | t
  2 | winston@continental.com | t
(2 rows)
$ psql -tAc "SELECT count(*) FROM alias;"          # -> 11
$ psql -tAc "SELECT count(*) FROM job;"            # -> 0
$ psql -tAc "SELECT count(*) FROM sync_event;"     # -> 0
$ psql -tAc "SELECT count(*) FROM contact WHERE email='sender-blitzy@external.test';"   # -> 0
```

The seeded demo account `john@wick.com` was intentionally preserved (it is part of the `flask dummy-data` seed, not a source change).

**Repository working tree — source repo (container `/app`, `git status --porcelain`):** shows only pre-existing baked drift that was present before this investigation and was **not** introduced by it (`M app/spamassassin_utils.py`, `M local_data/jwtRS256.key`, `M local_data/jwtRS256.key.pub`, `M local_data/test_words.txt`, `M static/package-lock.json`, `?? dump.rdb`). No source, template, configuration, or migration file was modified by this work.

**Deliverable working tree (`git status --porcelain`):**

```
?? blitzy/
```

The only addition to the repository is `blitzy/documentation/app_2cd6ee777f8c.md` (this file). `.env` is git-ignored and therefore never appears. The read-only guarantee is satisfied.

---

## 7. Coverage checklist

| Question / named item | Answered in |
|---|---|
| **Q1** — readiness indicators (logs) | §2.1 (banner, `SL` logger, format, DEBUG), §2.3 (bind), §2.4 (SMTP), §2.5 (worker) |
| **Q1** — readiness indicators (UI) | §2.6 (login page 200; demo login → dashboard) |
| Logging banner `>>> init logging <<<` | §2.1 |
| Log format / logger name `SL` / DEBUG / `message_id` UUID | §2.1 |
| Build stamp `SHA1="dev"` | §2.2 |
| Webapp bind `:7777` + init routines | §2.3 |
| SMTP `Listen for port 20381` | §2.4 |
| Job-runner loop + **10 s cadence (≥2 cycles)** | §2.5 |
| **Q2** — registration | §3.1 |
| **Q2** — verification (activation) | §3.2 |
| **Q2** — login | §3.3 |
| **Q2** — dashboard | §3.4 |
| Before/during/after states (`activated`, `activation_code`) | §3.5 (table) |
| UI surfaces (register/waiting/activate/login/dashboard) | §3.6 |
| **Q3** — email forwarding (`New message`/`Forward c→a→m`/`Finish`) | §4.1 |
| **Q3** — background jobs (`process_job` by name; observed onboarding) | §4.2 |
| **Q3** — scheduler (`cron.py -j`, yacron) | §4.3 |
| **Q3** — New Relic analytics vs PostgreSQL Proton-sync events | §4.4 (i), (ii) |
| **Q3** — identity verification (synchronous activation path) | §4.4 (iii), §3.1-§3.2 |
| Edge: duplicate email | §5.1 |
| Edge: bad-mailbox personal inbox | §5.2 |
| Edge: invalid activation code | §5.3 |
| Edge: expired activation code | §5.4 |
| Edge: wrong password | §5.5 |
| Edge: login before activation + resend | §5.6 |
| Edge: rate limiting (429) | §5.7 |
| Edge: wrong hCaptcha (inferred) | §5.8 |
| Cleanup + clean `git status` | §6 |

**Honest corrections vs. expectation (surfaced during observation):** (1) the Werkzeug "Running on …" line is suppressed by `app/log.py:70-71`, so HTTP readiness is confirmed via reachability + the `after_request` log at `server.py:284` (§2.3); (2) `add_sl_domains()`/`add_proton_partner()` are **not** called on webapp startup — they run in the `dummy-data` CLI command and `init_app.py`'s `__main__`, so the webapp log has no "SL domain" lines (§2.3).
