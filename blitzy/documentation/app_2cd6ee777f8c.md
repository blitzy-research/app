# SimpleLogin (self‑hosted) — First‑Time Operator Runtime Q&A

This document answers three questions a first‑time operator asked about a freshly launched, self‑hosted **SimpleLogin** instance: (Q1) what confirms the app is ready, (Q2) what a new user sees while registering → verifying → logging in → landing on the dashboard, and (Q3) what the background/internal services do behind the scenes and how you can tell they are talking to each other.

Every behavioural statement below sits next to the **actual runtime output** that demonstrates it (a verbatim log line, an HTTP response/header, or a measured value), plus a `file:line` reference into the source that produced it. Statements I could not observe at runtime are explicitly labelled **inferred**; anything obtained outside the default configuration or outside the real entry point is labelled **non‑default** / **non‑canonical**.

---

## Intro — how the instance was built, run, and observed

### Run environment

| Component | Value | Evidence |
|-----------|-------|----------|
| Application runtime | Python **3.10.18** (Poetry range `python = "^3.10"`) | `pyproject.toml:L61`; `python --version` in the venv |
| Web/WSGI server | Gunicorn **20.0.4** | startup banner (below); `pyproject.toml:L66` |
| Database | PostgreSQL (migration head `32f25cbf12f6`, 77 tables, seeded) | `psql` query (below); `CONTRIBUTING.md:L25` |
| Cache / rate‑limit / session store | Redis (responds `PONG`) | `redis-cli ping` |
| Front‑end assets | Prebuilt (Node not required at runtime) | container image |
| Config source | `/app/.env`, a **byte‑for‑byte copy of `example.env`** | `diff -q .env example.env` → identical |

The instance runs inside the canonical Docker image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` in its **default configuration**. Config is read from `/app/.env`, which `app/config.py` auto‑loads at import via `load_dotenv()` [`app/config.py:L71`]. I verified the config is the shipped default:

```console
$ diff -q .env example.env
# (no output) → .env is IDENTICAL to example.env
```

The application database was already migrated and seeded (the canonical recipe is `alembic upgrade head && flask dummy-data && python3 server.py` [`CONTRIBUTING.md:L106`], which creates the demo login `john@wick.com / password` [`CONTRIBUTING.md:L109`]):

```console
$ psql -U myuser -h localhost -d simplelogin -tAc "SELECT version_num FROM alembic_version;"
32f25cbf12f6
$ psql -U myuser -h localhost -d simplelogin -tAc "SELECT id,email,activated FROM users ORDER BY id LIMIT 3;"
1|john@wick.com|t
2|winston@continental.com|t
```

### Exact invocation commands (the five process entry points)

SimpleLogin's runtime is **five independent processes** that cooperate through shared PostgreSQL state. I started each from `/app` with the venv interpreter (`.env` auto‑loaded by `app/config.py:L71`):

```bash
# 1. Web app — canonical production entry point (Dockerfile:L47), bound to 0.0.0.0 so the host can reach it
/app/venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15

# 2. Inbound SMTP handler (listens on 20381)
/app/venv/bin/python email_handler.py

# 3. Background job runner (10-second polling loop)
/app/venv/bin/python job_runner.py

# 4. Event listener (the "listener" subcommand is REQUIRED)
/app/venv/bin/python event_listener.py listener

# 5. Cron job (yacron dispatches these per crontab.yml; run on-demand here)
/app/venv/bin/python cron.py -j stats
```

> **Note on the web entry point.** `python server.py` (the Flask dev server) binds to `127.0.0.1` only and is not reachable from outside the container, so I used the canonical production form `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` [`Dockerfile:L47`]. The WSGI object it serves is `app = create_app()` [`wsgi.py:L1,L3`].

### The uniform log format (so every line below is attributable)

Every SimpleLogin log line follows one format string [`app/log.py:L12-L15`]:

```
%(asctime)s - %(name)s - %(levelname)s - %(process)d - "%(pathname)s:%(lineno)d" - %(funcName)s() - %(message_id)s - %(message)s
```

That is: **timestamp – logger name – level – pid – `"pathname:lineno"` – `funcName()` – message‑id – message**. The logger name is always `SL` [`app/log.py:L79`], and each process prints a plain `>>> init logging <<<` banner when logging boots [`app/log.py:L67`]. Because the format embeds `pathname:lineno` and `funcName()`, every observed line can be attributed to the exact source location that emitted it — which is how the `file:line` citations below are grounded.

### Two default‑configuration flags that shape what you observe

Two shipped defaults materially change the observable behaviour, so they are called out up‑front:

1. **`NOT_SEND_EMAIL=true`** [`example.env:L19`]. Transactional email is **not delivered**; instead the mailer logs only the message metadata. Observed for the activation email:

   ```
   2026-07-03 00:05:28,436 - SL - DEBUG - 2141 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'Just one more step to join SimpleLogin', from '"noreply@sl.local" <noreply@sl.local>' to 'blitzytestuser@example.com'
   ```

   > **Observed reality vs. expectation.** A common assumption is that the activation *link* is printed to the logs. It is **not** in this build: `mail_sender.send()` under `NOT_SEND_EMAIL` logs only **subject / from / to** [`app/mail_sender.py:L130-L136`], never the body or the link. To complete the verification step canonically, I read the activation code from the real `activation_code` DB row created by the registration (see Q2), then visited the real `/auth/activate` endpoint.

2. **`DISABLE_ONBOARDING=true`** [`example.env:L150`]. Onboarding `Job` rows are **not** enqueued on self‑hosted registration, so the background‑job demonstration in Q3 uses a job type that does run (`send-alias-creation-events`) rather than onboarding jobs.

A third default is decisive for Q3's event pipeline: **`EVENT_WEBHOOK` is unset** (there is no `EVENT_WEBHOOK=` line in `example.env`; `POSTFIX_PORT` is likewise commented out at `example.env:L154`). As shown in Q3, this leaves the event‑delivery sink in a "skip" state by default.

### Scope / hygiene

This was a read‑only investigation. Temporary users, aliases, jobs, and observation scripts were created only to elicit runtime output and are removed afterward; no source file was modified. The seeded demo account is `john@wick.com / password` [`CONTRIBUTING.md:L109`].

---

## Q1 — Readiness / startup indicators

> *"After starting the system what should I notice in the logs or UI that shows the app is ready to handle user authentication and alias based email activity?"*

**Short answer:** watch for five things — the web app's `>>> init logging <<<` banner and gunicorn's `Listening at: http://0.0.0.0:7777`, a `GET /health` that returns `success`/`200`, the SMTP handler's `Listen for port 20381`, the job runner and event listener bootstrapping, and the login/registration UI answering on `:7777`. Each is shown verbatim below.

### 1. Web application is up and serving

**Gunicorn is listening on 7777.** Observed on web‑app start:

```
[2026-07-03 00:01:23 +0000] [2138] [INFO] Starting gunicorn 20.0.4
[2026-07-03 00:01:23 +0000] [2138] [INFO] Listening at: http://0.0.0.0:7777 (2138)
[2026-07-03 00:01:23 +0000] [2138] [INFO] Using worker: sync
[2026-07-03 00:01:23 +0000] [2140] [INFO] Booting worker with pid: 2140
[2026-07-03 00:01:23 +0000] [2141] [INFO] Booting worker with pid: 2141
```

This is the canonical production command `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` [`Dockerfile:L47`], and `EXPOSE 7777` [`Dockerfile:L44`] documents the port.

**Logging booted.** Each worker prints the bootstrap banner from `app/log.py:L67` (once per worker, pids 2140 and 2141):

```
>>> init logging <<<
```

Immediately after, the first `SL`‑formatted line appears, confirming the log format from `app/log.py:L12-L15` is in effect:

```
2026-07-03 00:01:23,902 - SL - DEBUG - 2140 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

**Health check.** The clearest readiness probe is `GET /health`. The handler is `return "success", 200` [`server.py:L215`] on route `@app.route("/health", methods=["GET"])` [`server.py:L213`]:

```console
$ curl -i http://localhost:7777/health
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Fri, 03 Jul 2026 00:01:44 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 7
Vary: Cookie
Set-Cookie: slapp=...; HttpOnly; Path=/; SameSite=Lax

success
```

The exact status line is `HTTP/1.1 200 OK`, the body is `success` (7 bytes — note `Content-Length: 7`), matching `"success", 200` [`server.py:L215`].

**Per‑request access log.** Every non‑static request is logged by the after‑request hook using the format string `"%s %s %s %s %s, takes %s"` [`server.py:L285`] (hook at `server.py:L284-L292`). Triggering `GET /auth/login` produced:

```
2026-07-03 00:01:44,438 - SL - DEBUG - 2141 - "/app/server.py:284" - after_request() -  - 172.17.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.019622325897216797
```

That line reads: client IP `172.17.0.1`, method `GET`, path `/auth/login`, request args `ImmutableMultiDict([])`, status `200`, and `takes 0.0196…` seconds. **Note:** `/health` deliberately produces **no** access‑log line — it is in the ignore list [`server.py:L195`] and explicitly skipped (`not request.path.startswith("/health")`) [`server.py:L281`]; indeed only the non‑health routes appeared in the access log.

### 2. The login / registration UI is reachable (UI landing state)

A first‑time operator can immediately reach the auth UI. Observed HTTP statuses:

```console
$ curl -s -o /dev/null -w "GET /auth/login -> HTTP %{http_code}\n" http://localhost:7777/auth/login
GET /auth/login -> HTTP 200
$ curl -s -o /dev/null -w "GET / -> HTTP %{http_code} ; redirect->%{redirect_url}\n" http://localhost:7777/
GET / -> HTTP 302 ; redirect->http://localhost:7777/auth/login
$ curl -s -o /dev/null -w "GET /auth/register -> HTTP %{http_code}\n" http://localhost:7777/auth/register
GET /auth/register -> HTTP 200
```

So the login page answers `200`, the registration page answers `200`, and the unauthenticated root `/` redirects (`302`) to `http://localhost:7777/auth/login` — the expected landing for a fresh visitor.

### 3. The alias / email subsystem is active (SMTP handler)

Starting `python email_handler.py` produced the two readiness lines that confirm the inbound SMTP server is bound:

```
2026-07-03 00:02:06,260 - SL - INFO - 2171 - "/app/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-03 00:02:06,262 - SL - DEBUG - 2171 - "/app/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

`Listen for port 20381` is `LOG.i("Listen for port %s", args.port)` [`email_handler.py:L2403`] and `Start mail controller 0.0.0.0 20381` is `LOG.d("Start mail controller %s %s", ...)` [`email_handler.py:L2386`] (the aiosmtpd `Controller` is created at `email_handler.py:L2383`). Port **20381** is the default local SMTP port.

### 4. The background job runner is up

`python job_runner.py` bootstraps (banner + first `SL` line, pid 2172) and then enters its polling loop:

```
>>> init logging <<<
2026-07-03 00:02:05,381 - SL - DEBUG - 2172 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

**Observed reality:** while the `Job` table is empty the loop is **silent** — the `for job in get_jobs_to_run(): LOG.d("Take job %s", job)` body [`job_runner.py:L333-L334`] emits nothing per cycle, and the loop simply calls `time.sleep(10)` [`job_runner.py:L347`]. So "readiness" for the job runner is the live process; the `Take job` line and the 10‑second cadence are demonstrated under load in **Q3**.

### 5. The event listener is up (internal event pipeline)

`python event_listener.py listener` (the `listener` subcommand is required [`event_listener.py:L17`]) selected its source and sink and began listening:

```
2026-07-03 00:02:05,481 - SL - INFO - 2173 - "/app/event_listener.py:34" - main() -  - Using PostgresEventSource
2026-07-03 00:02:05,489 - SL - INFO - 2173 - "/app/event_listener.py:43" - main() -  - Starting with HttpEventSink
2026-07-03 00:02:05,490 - SL - INFO - 2173 - "/app/events/event_source.py:49" - __listen() -  - Starting to listen to events
```

`Using PostgresEventSource` [`event_listener.py:L34`] and `Starting with HttpEventSink` [`event_listener.py:L43`] are the default source/sink; the `Runner(source=source, sink=sink)` orchestrator is built at `event_listener.py:L46`. (With `--dry-run` it would instead log `Starting with ConsoleEventSink` [`event_listener.py:L40`].)

### 6. Scheduled maintenance (cron)

Scheduled jobs are driven by yacron reading `crontab.yml`, which dispatches commands of the form `python /code/cron.py -j <job>` — e.g. the growth‑stats job:

```yaml
  - name: SimpleLogin growth stats
    command: python /code/cron.py -j stats
    schedule: "0 0 * * *"
```

Running that job on demand confirmed the dispatcher works (exit code `0`):

```
2026-07-03 00:03:05,057 - SL - DEBUG - 2217 - "/app/cron.py:1263" - <module>() -  - Start running cronjob
2026-07-03 00:03:05,059 - SL - DEBUG - 2217 - "/app/cron.py:1275" - <module>() -  - Compute growth and daily monitoring stats
2026-07-03 00:03:05,059 - SL - WARNING - 2217 - "/app/cron.py:540" - stats() -  - ADMIN_EMAIL not set, nothing to do
```

`crontab.yml` also schedules `delete_old_monitoring`, `check_custom_domain`, `check_hibp`, `notify_hibp`, `delete_logs`, `delete_old_data`, `poll_apple_subscription`, `notify_trial_end`, and `notify_manual_subscription_end` — all invoked as `python /code/cron.py -j <job>`.

### Readiness — at a glance

| Subsystem | Ready signal (observed) | Source |
|-----------|-------------------------|--------|
| Web app | `Listening at: http://0.0.0.0:7777` + `>>> init logging <<<` | `Dockerfile:L47`; `app/log.py:L67` |
| Auth serving | `GET /health` → `HTTP/1.1 200 OK`, body `success` | `server.py:L213-L215` |
| UI landing | `/auth/login`→200, `/`→302→`/auth/login`, `/auth/register`→200 | `server.py` routes |
| SMTP / alias email | `Listen for port 20381` + `Start mail controller 0.0.0.0 20381` | `email_handler.py:L2403,L2386` |
| Job runner | process alive; enters `time.sleep(10)` loop | `job_runner.py:L330-L347` |
| Event listener | `Using PostgresEventSource` + `Starting with HttpEventSink` | `event_listener.py:L34,L43` |
| Cron | `Start running cronjob` (exit 0) | `cron.py:L1263`; `crontab.yml` |

---


## Q2 — The new‑user journey (register → verify → log in → dashboard)

> *"walking through the typical product experience as if you were a new user signing up for the first time. What happens when a user registers verifies their address and tries to log in. What visible behavior confirms that the system is handling every step correctly and forwarding the user into the dashboard as expected?"*

I drove the **real HTTP endpoints on `:7777`** with a fresh temporary user (`blitzytestuser2@example.com`, plus `blitzytestuser@example.com` for the negative cases), including the CSRF token each form requires, and cleaned them up afterward. Each step below shows the HTTP status, redirect target, flashed UI message, DB row, and log line.

### Step 1 — Register (`POST /auth/register` [`register.py:L31`])

Submitting the registration form created the account and rendered the "check your email" page:

```
1) POST /auth/register -> 200 | title: Activation Email Sent | SimpleLogin
```

The server logged the account creation, `LOG.d("create user %s", email)` [`register.py:L85`]:

```
2026-07-03 00:08:38,947 - SL - DEBUG - 2141 - "/app/app/auth/views/register.py:85" - register() -  - create user blitzytestuser2@example.com
```

**Observed reality vs. expectation.** A successful registration returns **HTTP 200 rendering `auth/register_waiting_activation.html`** — *not* a 302 redirect. The code returns the template directly: `return render_template("auth/register_waiting_activation.html")` [`register.py:L104`], and the rendered page's `<title>` is `Activation Email Sent`. The access log confirms the 200:

```
2026-07-03 00:08:39,378 - SL - DEBUG - 2141 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/register ... 200, takes 0.33...
```

**Duplicate‑registration guard.** Re‑submitting the same email flashes the "already used" error, `flash(f"Email {email} already used", "error")` [`register.py:L82`]. The flash string was present verbatim in the response body:

```
POST /auth/register (dup) -> 200
  flash present: 'Email blitzytestuser@example.com already used'
```

### Step 2 — Verify the address (email activation)

Registration created a real `ActivationCode` row (`send_activation_email` at `register.py:L117`, code = `random_string(30)`). Because `NOT_SEND_EMAIL=true` [`example.env:L19`] the email is **not delivered and the link is not logged** (only subject/from/to are logged — see Intro). The activation link's canonical shape is `f"{URL}/auth/activate?code={activation.code}"` [`register.py:L124`] with `URL=http://localhost:7777` [`example.env:L6`]. I therefore read the code from the real DB row it created and reconstructed the canonical link:

```console
$ psql ... -c "SELECT ac.code FROM activation_code ac JOIN users u ON u.id=ac.user_id WHERE u.email='blitzytestuser2@example.com';"
ncwerxakspsgxhfehuehmwrlhjsgtg
# canonical link (register.py:L124 format):
# http://localhost:7777/auth/activate?code=ncwerxakspsgxhfehuehmwrlhjsgtg
```

Visiting the **real** `GET /auth/activate?code=…` endpoint [`activate.py:L13`] activated the account and forwarded to the dashboard:

```
3) GET /auth/activate?code=<real> (follow) -> final 200 | final URL: http://localhost:7777/dashboard/
   activated flash present: 'Your account has been activated'
   dashboard title: Alias | SimpleLogin
```

- The success flash is `flash("Your account has been activated", "success")` [`activate.py:L56`] — present verbatim in the rendered page.
- The redirect is logged as `LOG.d("redirect user to dashboard")` [`activate.py:L66`] → `redirect(url_for("dashboard.index"))` [`activate.py:L67`]:

  ```
  2026-07-03 00:08:39,273 - SL - DEBUG - 2141 - "/app/app/auth/views/activate.py:66" - activate() -  - redirect user to dashboard
  ```
- The raw (unfollowed) response is a `302` to `http://localhost:7777/dashboard/`, per the access log:

  ```
  2026-07-03 00:08:39,273 - SL - DEBUG - 2141 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/activate ImmutableMultiDict([('code', 'ncwerxakspsgxhfehuehmwrlhjsgtg')]) 302, takes 0.03...
  ```
- The DB row flips to activated (`activated` `f`→`t`), confirming verification really happened (checked for the first temp user):

  ```console
  $ psql ... -c "SELECT id,email,activated FROM users WHERE email='blitzytestuser@example.com';"
  5|blitzytestuser@example.com|t
  ```

**Invalid activation code** returns **HTTP 400** with the error `Activation code cannot be found` [`activate.py:L33`]:

```
GET /auth/activate?code=INVALID -> 400
  flash/error present: 'Activation code cannot be found'
```

The access log confirms the 400 status:

```
2026-07-03 00:07:13,857 - SL - DEBUG - 2140 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/activate ImmutableMultiDict([('code', 'THIS_CODE_DOES_NOT_EXIST_123')]) 400, takes 0.005...
```

The **expired‑code** branch returns **HTTP 400** with the error `Activation code was expired` [`activate.py:L42`]. Codes carry a 1‑hour TTL — the `expired` column defaults to now+1h (`expired = sa.Column(ArrowType, nullable=False, default=_expiration_1h)` [`app/models.py:L1212`]) and `is_expired()` returns `self.expired < arrow.now()` [`app/models.py:L1214-L1215`]. To observe this **through the real endpoint**, I registered a temporary user, back‑dated its activation code's `expired` timestamp to 2 hours in the past (a temporary DB fixture — equivalent to letting the TTL lapse; only the *precondition* is set this way, the branch itself is exercised over real HTTP), then hit the **real** `GET /auth/activate` with a fresh, unauthenticated session:

```
REGISTER -> HTTP 200
ACTIVATION CODE (from DB): ctjwzyirspsnkcsfxapovblybmszyt
BACK-DATED expired = 2026-07-02 22:30:54.23429
ACTIVATE(expired) -> HTTP 400
error text present: 'Activation code was expired' -> True
rendered snippet: '>\n        Activation code was expired\n      </d'
```

The access log confirms the **400** status from the real handler:

```
2026-07-03 00:30:54,281 - SL - DEBUG - 2140 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/activate ImmutableMultiDict([('code', 'ctjwzyirspsnkcsfxapovblybmszyt')]) 400, takes 0.0026259422302246094
```

### Step 3 — Log in (`POST /auth/login` [`login.py:L21`])

With correct credentials, login succeeded and issued a `302` to the dashboard:

```
4) POST /auth/login (correct) -> 302 | Location: http://localhost:7777/dashboard/
```

The success branch runs `LoginEvent(LoginEvent.ActionType.success).send()` [`login.py:L71`] then `after_login(user, next_url)` [`login.py:L72`]; `after_login` logs `LOG.d("log user %s in", user)` [`login_utils.py:L35`] and calls `login_user(user)` [`login_utils.py:L36`]. That log line is the observable proof the success branch executed (it is only reachable after `login.py:L71-L72`):

```
2026-07-03 00:08:39,623 - SL - DEBUG - 2141 - "/app/app/auth/views/login_utils.py:35" - after_login() -  - log user <User 6 blitzytestuser2@example.com blitzytestuser2@example.com> in
```

**Negative branches** (for completeness):

- Wrong credentials → `flash("Email or password incorrect", "error")` [`login.py:L49`]:

  ```
  POST /auth/login (wrong pw) -> 200
    flash present: 'Email or password incorrect'
  ```
- Un‑activated account (logging in before verifying) → the "check your inbox" flash [`login.py:L66`] and `LoginEvent.not_activated` [`login.py:L69`]:

  ```
  POST /auth/login (unactivated) -> 200
    flash present: 'Please check your inbox for the activation email'
  ```

### Step 4 — Forwarded into the dashboard

`after_login` finishes by redirecting to the dashboard when there is no `next_url`: `LOG.d("redirect user to dashboard")` [`login_utils.py:L44`] → `redirect(url_for("dashboard.index"))` [`login_utils.py:L45`]:

```
2026-07-03 00:08:39,623 - SL - DEBUG - 2141 - "/app/app/auth/views/login_utils.py:44" - after_login() -  - redirect user to dashboard
2026-07-03 00:08:39,623 - SL - DEBUG - 2141 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 302, takes 0.238...
```

The dashboard landing page `dashboard.index` [`dashboard/views/index.py:L67`, route at `L55`] then rendered with `HTTP 200`. It logs `LOG.d("Show intro to %s", current_user)` [`index.py:L172`] and renders `templates/dashboard/index.html` [`index.py:L215-L216`] (whose `<title>` is `Alias`); alias statistics come from `class Stats` [`index.py:L25`]:

```
2026-07-03 00:08:39,280 - SL - DEBUG - 2141 - "/app/app/dashboard/views/index.py:172" - index() -  - Show intro to <User 6 blitzytestuser2@example.com blitzytestuser2@example.com>
2026-07-03 00:08:39,378 - SL - DEBUG - 2141 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard/ ImmutableMultiDict([]) 200, takes 0.101...
```

So the full happy path is visible end‑to‑end: **register (200 → "Activation Email Sent") → activate (302 → `/dashboard/`, "Your account has been activated") → login (302 → `/dashboard/`, "log user … in") → dashboard (`GET /dashboard/` 200, "Show intro")**.

### Rate limits — reported exactly as observed

The three endpoints behave differently, and I verified each empirically by bursting requests:

| Endpoint | Decorator | Observed burst behaviour |
|----------|-----------|--------------------------|
| `/auth/login` | `@limiter.limit("10/minute", deduct_when=… g.deduct_limit)` [`login.py:L22-L24`] | 30 failed POSTs → **10× HTTP 429** (first 429 at attempt #19) |
| `/auth/activate` | `@limiter.limit("10/minute", deduct_when=… g.deduct_limit)` [`activate.py:L14-L16`] | 30 invalid GETs → 400s then **10× HTTP 429** (first at #20) |
| `/auth/register` | **none** (no `limiter` import in the module) | 15 duplicate POSTs → **all 200, zero 429** |

Observed status sequences:

```
/auth/login (30 failed POSTs):    [200×18, 429, 200, 200, 429, 429, ...]  -> 429 count: 10
/auth/activate (30 invalid GETs): [400×19, 429, 429, 400, 429, ...]       -> 429 count: 10
/auth/register (15 dup POSTs):    [200, 200, ... 200]                     -> 429 count: 0
```

Two important, observed nuances:

1. **`/register` is not rate‑limited per‑route** — the module `app/auth/views/register.py` does not import or apply `limiter` at all (confirmed: zero `limiter` references), so all 15 attempts returned `200`.
2. The `10/minute` cap on `/login` and `/activate` is **conditional**: it only counts a request when the handler sets `g.deduct_limit = True`, which happens **only on failure** — a failed login (`login.py:L47`) or an invalid activation code (`activate.py:L30`). Successful logins/activations are not counted. The first `429` appearing near attempt ~19–20 (rather than 11) reflects the default **in‑memory** limiter store (`MEM_STORE_URI` is absent from `.env`) being **per‑worker** across the **two** gunicorn workers, i.e. ~10 per worker before the limit trips.

---


## Q3 — Behind the scenes: background jobs & internal services

> *"what the application does behind the scenes during that flow. Are there any indicators that background jobs or internal services are doing their part to support email forwarding or identity verification. What should I expect to observe at runtime that tells me these moving pieces are active and talking to each other properly?"*

The key architectural fact — and the answer to "talking to each other" — is that the five processes **do not call each other via RPC**; they communicate through **shared PostgreSQL state**: the web app enqueues `Job` rows and the **job runner** drains them on a timer, while events are written as `SyncEvent` rows and announced with PostgreSQL `NOTIFY`, which the **event listener** consumes. Below is each moving piece, observed at runtime.

### A) Background jobs — the job runner drains the `Job` table every 10 seconds

To observe the cadence I enqueued four `send-alias-creation-events` jobs (`config.JOB_SEND_ALIAS_CREATION_EVENTS = "send-alias-creation-events"` [`config.py:L311`]) for `user_id=1`, one after each was taken, and read the timestamps of the consecutive `Take job` lines (`LOG.d("Take job %s", job)` [`job_runner.py:L334`]):

```
2026-07-03 00:14:17,228 - SL - DEBUG - 2172 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 2 send-alias-creation-events {'user_id': 1}>
2026-07-03 00:14:27,254 - SL - DEBUG - 2172 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 3 send-alias-creation-events {'user_id': 1}>
2026-07-03 00:14:37,270 - SL - DEBUG - 2172 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 4 send-alias-creation-events {'user_id': 1}>
2026-07-03 00:14:47,290 - SL - DEBUG - 2172 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 5 send-alias-creation-events {'user_id': 1}>
```

**Measured cadence** (run duration ≈ 30 s, four cycles): the intervals between consecutive `Take job` ticks were **10.026 s, 10.016 s, 10.020 s** — stable at ~10 s across more than two cycles. This is the `time.sleep(10)` at the bottom of the runner loop [`job_runner.py:L347`].

**Dispatch by job name.** Each taken job is routed by `process_job` [`job_runner.py:L188`]. For this job the branch is `elif job.name == config.JOB_SEND_ALIAS_CREATION_EVENTS:` [`job_runner.py:L295`], which logged:

```
2026-07-03 00:14:17,236 - SL - DEBUG - 2172 - "/app/job_runner.py:299" - process_job() -  - Sending alias creation events for <User 1 John Wick john@wick.com>
```

An unrecognised job name hits the fallback `LOG.e("Unknown job name %s", job.name)` [`job_runner.py:L304`]. I confirmed this at runtime by enqueuing a `Job` with a bogus name into the shared table; the **already‑running** job runner (pid 2172) took it and logged the error verbatim:

```
2026-07-03 00:32:28,550 - SL - ERROR - 2172 - "/app/job_runner.py:304" - process_job() -  - Unknown job name blitzy-bogus-job-name
```

**Default‑config note on onboarding.** With `DISABLE_ONBOARDING=true` [`example.env:L150`], onboarding jobs are not enqueued on self‑hosted registration, which is why this demonstration uses `send-alias-creation-events` rather than an onboarding job.

### B) Email forwarding lifecycle — the SMTP handler (port 20381)

To observe forwarding I sent one test message through the **real** SMTP handler on `:20381`, from an outside sender to the seeded alias `e1@sl.local` (which forwards to mailbox `john@wick.com`). The handler bracketed the transaction with an opening and a closing line, both carrying the same message‑id (`0660a925-…`).

**Opening** — `LOG.i("New message, mail from %s, rctp tos %s ", …)` (string literal at `email_handler.py:L2344`; the `LOG.i` call reports `email_handler.py:2343`). **The source misspells "rcpt" as `rctp tos` — quoted verbatim, not corrected:**

```
2026-07-03 00:15:20,617 - SL - INFO - 2171 - "/app/email_handler.py:2343" - _handle() - 0660a925-10f6-4d81-b6a9-563dc3a7ef7a - New message, mail from outside-sender@example.com, rctp tos ['e1@sl.local']
```

**The forward chain in between** shows the alias → contact → mailbox resolution — exactly the "moving pieces" of forwarding:

```
2026-07-03 00:15:20,745 - SL - DEBUG - 2171 - "/app/email_handler.py:2202" - handle() - 0660a925-... - Forward phase outside-sender@example.com(outside-sender@example.com) -> e1@sl.local
2026-07-03 00:15:20,793 - SL - DEBUG - 2171 - "/app/email_handler.py:688" - forward_email_to_mailbox() - 0660a925-... - Forward <Contact 2 outside-sender@example.com 5> -> <Alias 5 e1@sl.local> -> <Mailbox 1 john@wick.com>
2026-07-03 00:15:20,802 - SL - DEBUG - 2171 - "/app/email_handler.py:893" - forward_email_to_mailbox() - 0660a925-... - Forward mail from outside-sender@example.com to john@wick.com, mail_options:['SIZE=251']
```

**Closing** — `Finish mail_from %s, rcpt_tos %s, takes %s seconds with return code '%s'<<===` (string literal at `email_handler.py:L2368`; call reports `email_handler.py:2367`):

```
2026-07-03 00:15:20,803 - SL - INFO - 2171 - "/app/email_handler.py:2367" - _handle() - 0660a925-... - Finish mail_from outside-sender@example.com, rcpt_tos ['e1@sl.local'], takes 0.18664312362670898 seconds with return code '250 Message accepted for delivery'<<===
```

Measured: the transaction took **`0.18664312362670898` seconds** and finished with return code **`250 Message accepted for delivery`**. (The actual onward SMTP send is logged rather than delivered because `NOT_SEND_EMAIL=true`.)

### C) Internal event pipeline — `SyncEvent` → PostgreSQL `NOTIFY` → listener → sink

This is the clearest "moving pieces talking to each other" demonstration: a producer writes a `SyncEvent` row and fires `NOTIFY`, and a **separate process** (the event listener) receives it over PostgreSQL's LISTEN/NOTIFY and hands it to a sink.

The producer is `PostgresDispatcher.send()` [`app/events/event_dispatcher.py:L24-L26`]: it does `SyncEvent.create(content=event, flush=True)` then `Session.execute("NOTIFY simplelogin_sync_events, '<id>'")`, where the channel is `NOTIFICATION_CHANNEL = "simplelogin_sync_events"` [`event_dispatcher.py:L14`].

> **Default‑config caveat (important).** In the shipped configuration the canonical trigger (creating an alias) does **not** emit an event at all: `EventDispatcher.send_event` short‑circuits because `EVENT_WEBHOOK` is unset — observed during the job‑runner run above (`event_dispatcher.py:L62`):
>
> ```
> 2026-07-03 00:14:17,239 - SL - INFO - 2172 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
> ```
>
> So by default the event pipeline is wired but **dormant**. To exercise the pipeline mechanics I invoked the real `PostgresDispatcher.send()` code path **directly** — this is **labelled non‑canonical** (the alias‑creation trigger is gated off by default), but the listener + sink consuming it are the **real, running** default‑config pipeline.

**(a) Default `HttpEventSink` path (running listener, pid 2173).** Dispatching one event created `SyncEvent` id `1` and fired the NOTIFY; the listener received it and the default sink skipped delivery:

```
2026-07-03 00:16:51,525 - SL - DEBUG - 2173 - "/app/events/event_source.py:55" - __listen() -  - Got NOTIFY: pid=2610 channel=simplelogin_sync_events payload=1
2026-07-03 00:16:51,598 - SL - WARNING - 2173 - "/app/events/event_sink.py:19" - process() -  - Skipping sending event because there is no webhook configured
```

- `Got NOTIFY: pid=2610 channel=simplelogin_sync_events payload=1` is `PostgresEventSource.__listen` (string literal at `events/event_source.py:L56`, after `LISTEN {NOTIFICATION_CHANNEL}` at `L47`). The **payload is the `SyncEvent.id`** (`1`), and `channel=simplelogin_sync_events` matches the constant. Note the cross‑process handshake: the `NOTIFY` came from the dispatcher's DB backend `pid=2610`, and the listener OS process `pid 2173` received it — two processes, communicating purely through PostgreSQL.
- `Skipping sending event because there is no webhook configured` is `HttpEventSink.process` [`events/event_sink.py:L19`] taking the `if not EVENT_WEBHOOK:` branch [`L18`] and returning `False` [`L20`]. Its success gate is `if res.status_code != 200:` [`event_sink.py:L33`]. Because it returned `False`, `Runner.__on_event` incremented `retry_count` [`events/runner.py:L42`] rather than deleting the row (verified: `SyncEvent` id `1` had `retry_count = 1` afterward).

**(b) Success path via `--dry-run` `ConsoleEventSink` (labelled non‑default).** To show a *successful* hand‑off, I ran a second listener with `--dry-run` (which selects `ConsoleEventSink` [`event_listener.py:L40`]) and dispatched another event (`SyncEvent` id `2`):

```
2026-07-03 00:17:49,966 - SL - DEBUG - 2629 - "/app/events/event_source.py:55" - __listen() -  - Got NOTIFY: pid=2645 channel=simplelogin_sync_events payload=2
2026-07-03 00:17:50,036 - SL - INFO - 2629 - "/app/events/event_sink.py:45" - process() -  - Handling event 2
2026-07-03 00:17:50,038 - SL - INFO - 2629 - "/app/events/runner.py:28" - __on_event() -  - Marked 2 as done
```

Here `ConsoleEventSink.process` [`event_sink.py:L45`] returned `True`, so `Runner.__on_event` deleted the row (`SyncEvent.delete(...)` [`events/runner.py:L27`]) and logged `Marked 2 as done` [`runner.py:L28`]. Confirmed by the DB: `SyncEvent` id `2` was gone (only the lingering id `1` from the failure path remained). The default listener was then restored.

### Cause → effect summary (how the pieces connect)

```
                 enqueue Job row                        poll every 10s (job_runner.py:L347)
   Web app  ───────────────────────►  Postgres `Job`  ◄──────────────────────  Job runner
                                          table                                  (process_job → dispatch by name)

                 SyncEvent.create + NOTIFY 'simplelogin_sync_events'      LISTEN + Got NOTIFY (event_source.py:L47,L56)
   Producer ───────────────────────►  Postgres `SyncEvent` row  ────────────────────►  Event listener
                                                                                          │
                                                          HttpEventSink (default: skip, no webhook)  or  ConsoleEventSink (dry-run: Marked done)
```

- **Identity verification** is supported by the `ActivationCode` row created at registration and consumed at `/auth/activate` (Q2).
- **Email forwarding** is the SMTP handler's `New message … → Finish … return code '250 …'` lifecycle (Q3‑B).
- **Background jobs** are the `Job` rows the job runner drains every 10 s (Q3‑A).
- **Internal services talking to each other** is the `SyncEvent` + `NOTIFY simplelogin_sync_events` → listener → sink chain (Q3‑C) — dormant by default because `EVENT_WEBHOOK` is unset.

---


## Coverage pass — every named item, addressed with evidence

**Q1 — readiness / startup indicators**

- [x] `>>> init logging <<<` banner — observed (once per worker) — `app/log.py:L67`
- [x] "listening on 7777" — `Listening at: http://0.0.0.0:7777 (2138)` — `Dockerfile:L47`, `wsgi.py:L3`
- [x] `/health` → `success`, `200` — `HTTP/1.1 200 OK` + body `success` (`Content-Length: 7`) — `server.py:L213-L215`
- [x] Per‑request access log — `172.17.0.1 GET /auth/login … 200, takes 0.0196…` — `server.py:L285` (and `/health` excluded, `server.py:L195,L281`)
- [x] SMTP `Listen for port 20381` — observed — `email_handler.py:L2403`
- [x] SMTP `Start mail controller 0.0.0.0 20381` — observed — `email_handler.py:L2386`
- [x] Job runner idle loop / `Take job` — idle‑silent at startup; `Take job` shown under load in Q3 — `job_runner.py:L334,L347`
- [x] Event listener `Using PostgresEventSource` / `Starting with HttpEventSink` — observed — `event_listener.py:L34,L43`
- [x] Cron dispatch — `python /code/cron.py -j stats` → `Start running cronjob` (exit 0) — `crontab.yml`, `cron.py:L1263`
- [x] UI landing — `/auth/login`→200, `/`→302→`/auth/login`, `/auth/register`→200 — observed

**Q2 — new‑user flow**

- [x] Register `create user` — `create user blitzytestuser2@example.com` — `register.py:L85`
- [x] Register response — **200** rendering `register_waiting_activation.html` ("Activation Email Sent"), **not 302** — `register.py:L104`
- [x] Duplicate flash — `Email … already used` — `register.py:L82`
- [x] Activation link — **not in logs** by default (only subject/from/to logged); code read from real `activation_code` row; canonical link `register.py:L124` shape — `example.env:L19`, `app/mail_sender.py:L130-L136`
- [x] `Your account has been activated` — observed in rendered page — `activate.py:L56`
- [x] Redirect to dashboard — `redirect user to dashboard` + `302`→`/dashboard/` — `activate.py:L66-L67`
- [x] Invalid code — **HTTP 400** `Activation code cannot be found` — `activate.py:L33`
- [x] Expired code — **HTTP 400** `Activation code was expired` — observed via real `/auth/activate` (expiry precondition set by back‑dating the code's `expired` timestamp; 1h TTL) — `activate.py:L42`, `app/models.py:L1212-L1215`
- [x] Login success — `log user … in` (via `LoginEvent.success` → `after_login`) — `login.py:L71-L72`, `login_utils.py:L35`
- [x] Wrong credentials — `Email or password incorrect` — `login.py:L49`
- [x] Not‑activated — `Please check your inbox for the activation email` — `login.py:L66`, `L69`
- [x] Dashboard — `Show intro to <User 6 …>`, `GET /dashboard/` 200, renders `dashboard/index.html` — `index.py:L172,L215-L216`; `Stats` `index.py:L25`
- [x] Rate limits reported as observed — `/login` & `/activate` = 10/min on **failures** (429 seen); `/register` = **unthrottled** (no `limiter`) — `login.py:L22-L24`, `activate.py:L14-L16`, `register.py`

**Q3 — behind‑the‑scenes services**

- [x] 10‑second cadence, ≥2 cycles, duration stated — intervals 10.026/10.016/10.020 s over ~30 s (4 cycles) — `job_runner.py:L347`
- [x] `Take job` — `Take job <Job 2 send-alias-creation-events …>` — `job_runner.py:L334`
- [x] `process_job` dispatch — `Sending alias creation events for <User 1 …>` — `job_runner.py:L295,L299`; unknown‑name branch observed (`Unknown job name blitzy-bogus-job-name`) — `job_runner.py:L304`
- [x] `DISABLE_ONBOARDING` note — onboarding jobs off; used `send-alias-creation-events` — `example.env:L150`
- [x] SMTP `New message … rctp tos …` (typo verbatim) — observed — `email_handler.py:L2344` (call `L2343`)
- [x] SMTP `Finish … takes … seconds with return code '…'` — `takes 0.18664312362670898 seconds`, `250 Message accepted for delivery` — `email_handler.py:L2368` (call `L2367`)
- [x] `SyncEvent` → `NOTIFY` → listener — `Got NOTIFY: pid=2610 channel=simplelogin_sync_events payload=1` — `event_dispatcher.py:L14,L24-L26`; `event_source.py:L47,L56`
- [x] `HttpEventSink` + `EVENT_WEBHOOK`‑unset caveat — `Skipping sending event because there is no webhook configured` — `event_sink.py:L18-L20`
- [x] Success gate `status_code != 200` — `event_sink.py:L33`; on success delete (`runner.py:L27`), on failure `retry_count += 1` (`runner.py:L42`)
- [x] `ConsoleEventSink` dry‑run success path (labelled non‑default) — `Handling event 2` → `Marked 2 as done` (row deleted) — `event_sink.py:L45`, `runner.py:L28`
- [x] Shared‑PostgreSQL‑state cause→effect (not RPC) — `Job` + `SyncEvent` tables — documented above

**Default‑config disclosures stated**

- [x] `NOT_SEND_EMAIL=true` — activation email logged (metadata only), not delivered — `example.env:L19`
- [x] `DISABLE_ONBOARDING=true` — onboarding jobs not enqueued — `example.env:L150`
- [x] `EVENT_WEBHOOK` unset — event delivery skipped by default; dry‑run/console path labelled non‑default — `event_sink.py:L18-L20`

### Notes where observed reality differed from the initial expectation

1. **Successful registration returns HTTP 200** (rendering the waiting‑activation page), not a 302 redirect — `register.py:L104`.
2. **The activation link is not printed to the logs** under `NOT_SEND_EMAIL=true`; only subject/from/to are logged — `app/mail_sender.py:L130-L136`. The code was read from the real `activation_code` DB row.
3. **`/register` has no per‑route rate limit** (the module does not import `limiter`); the `10/minute` cap applies to `/login` and `/activate` and only to **failed** attempts — `login.py:L47`, `activate.py:L30`.
4. The **event pipeline is dormant by default** because `EVENT_WEBHOOK` is unset; the mechanics were exercised by invoking the real dispatcher directly (labelled non‑canonical) and via the `--dry-run` console sink (labelled non‑default).
5. `email_handler.py` log line numbers: the `New message` string literal is at `L2344` (the `LOG.i` call reports `L2343`), and the `Finish` literal is at `L2368` (call reports `L2367`); the source spelling `rctp tos` is a typo, quoted verbatim.

