# SimpleLogin local run — runtime‑grounded Q&A

This document confirms that a freshly started **local SimpleLogin** instance behaves correctly by **actually running the system and observing its runtime behaviour**, not by reading the code alone. Every factual claim below is backed by (a) the exact command or code that produced it, (b) the **verbatim** output that was observed, and (c) an exact `file:line` citation into the source at branch `app_2cd6ee777f8c` (HEAD commit `2cd6ee77`). Random values (activation code, alias suffix, user id, timestamps, process ids) are specific to this run; the invariants (HTTP statuses, redirect targets, log `file:line`, the 30‑character activation code, the e‑mail canonicalisation) are not.

It answers three questions asked by a first‑time self‑hoster:

- **Q1 — Readiness.** After the system starts, what appears in the **logs or UI** that proves the app is ready to handle **user authentication** and **alias‑based e‑mail** activity?
- **Q2 — New‑user walkthrough.** Walking through the product as a brand‑new user who **registers**, **verifies** their address and **logs in** — what **visible behaviour** confirms each step succeeds and that the user is **forwarded into the dashboard**?
- **Q3 — Behind the scenes.** What indicators show that **background jobs or internal services** support **e‑mail forwarding** and **identity verification**, and what proves at runtime that these pieces are **active and talking to each other**?

No application source file was modified. The only artefact produced is this document. Every temporary user, alias, job, throwaway database and observation script created during the investigation was deleted afterward, and `git status --porcelain` was verified empty (see the **Cleanup and read‑only verification** section at the end).

---

## Setup summary — how the stack was run

The investigation ran inside the project’s own Docker image (`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`), which checks out this repository at `/app` on the same commit (`2cd6ee77`) and ships a Python 3.10 virtualenv at `/app/venv`.

**Runtime versions actually observed:**

```text
Python 3.10.18                       # /app/venv/bin/python --version
PostgreSQL 15.13 (Debian 15.13-0+deb12u1)   # SHOW server_version
Redis 7.0.15                         # redis-cli INFO server -> redis_version
```

Python **3.10** is the version the project pins — `python = "^3.10"` at `pyproject.toml:61` and `FROM python:3.10` at `Dockerfile:8`; `CONTRIBUTING.md:236` explicitly notes 3.12 does not work (`# we haven't managed to make python 3.12 work`). A newer interpreter was deliberately avoided.

**Non‑mutating configuration technique.** The app is configured from a file **outside the repository**, `/root/sl.env`, referenced through the `CONFIG` environment variable — `app/config.py:65` does `config_file = os.environ.get("CONFIG")` and `app/config.py:69` does `load_dotenv(get_abs_path(config_file))`. This lets the full stack boot without editing any tracked file. The relevant values (a copy of `example.env` plus a local `DB_URI`/`MEM_STORE_URI`) are:

```text
URL=http://localhost:7777            # example.env:6
NOT_SEND_EMAIL=true                  # example.env:19
EMAIL_DOMAIN=sl.local                # example.env:22
DISABLE_ONBOARDING=true              # example.env:150
DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin
MEM_STORE_URI=redis://localhost
FLASK_SECRET=<redacted-local-dev-secret>
```

**Commands run** (mirroring the documented local path in `CONTRIBUTING.md:88`, `:106`, `:109`, `:212`, `:228`):

```bash
# PostgreSQL + Redis are started as local services (already online in the image)
# Schema (already at head 32f25cbf12f6) and baseline seed:
CONFIG=/root/sl.env /app/venv/bin/alembic upgrade head
CONFIG=/root/sl.env FLASK_APP=server.py /app/venv/bin/flask dummy-data   # seeds john@wick.com / winston@continental.com
# Web app (dev server on http://localhost:7777):
CONFIG=/root/sl.env /app/venv/bin/python server.py
# Standalone services observed for Q3:
CONFIG=/root/sl.env /app/venv/bin/python email_handler.py   # inbound SMTP on :20381
CONFIG=/root/sl.env /app/venv/bin/python job_runner.py      # 10s poll loop
```

**Dependency transparency (what this environment actually contains).** Unlike a from‑scratch scratch‑venv, the Docker image’s virtualenv installs the **locked** dependency set: the versions observed match `poetry.lock` exactly — for example `Flask 1.1.2`, `SQLAlchemy 1.3.24`, `aiosmtpd 1.4.2`, `redis 4.6.0`, `cbor2 5.2.0` and `pyre2 0.3.6`. `pyre2` is present as a compiled module (`re2.cpython-310-x86_64-linux-gnu.so`) and `import re2` exposes `DOTALL`, so it behaves exactly as the lockfile intends. **No dependency substitution was required** for the flows in this document. For full transparency, two notes about how the base image was prepared — neither is a change made by this investigation, and neither touches the register/verify/login/e‑mail flows documented below: (1) `pyre2 0.3.6` was reinstalled to replace a `google-re2` build the base image had shipped, restoring the version `poetry.lock` pins (the committed source imports it as `import re2 as re` at `app/email_utils.py:23` and `app/dashboard/views/referral.py:1`); and (2) the base image left one unrelated file, `app/spamassassin_utils.py`, importing Python’s stdlib `re` instead of `re2` — that module participates only in inbound spam scanning and in none of the flows below. Crucially, the **deliverable repository** (the one this document is committed to) is left byte‑for‑byte unchanged: only this Markdown file is added, and `git status --porcelain` there reports nothing else.

**Startup banner actually captured** when booting the web app — the plain `print()` lines emitted before logging initialises, followed by the first structured log line:

```text
load config file /root/sl.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/ihreyxvoiigseuaaglmw
Upload files to local dir
>>> init logging <<<
2026-07-01 04:41:22,079 - SL - DEBUG - 938 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

The `load config file /root/sl.env` line is the `print("load config file", config_file)` at `app/config.py:68`; `>>> init logging <<<` is `print(">>> init logging <<<")` at `app/log.py:67`. Everything after that point is emitted by the single **`SL`** logger.

---

## Q1 — Readiness indicators (logs / UI)

> *"After starting the system what should I notice in the logs or UI that shows the app is ready to handle user authentication and alias based email activity?"*

**Short answer.** Four concrete signals: (1) the `>>> init logging <<<` marker plus the single, uniform **`SL`** log‑line format prove the logging subsystem is up; (2) the `/health` endpoint returns `success` with HTTP `200`; (3) every unauthenticated request is redirected to `/auth/login`, proving the authentication gate is live; and (4) the local alias domain `sl.local` is registered (logged as `Add sl.local to SL domain`), proving the platform is ready to create alias‑based e‑mail.

### 1) The logs show the app booted and the logging subsystem is ready

The very first evidence that the process reached a ready state is the logging‑init marker and the first structured log line (from the banner above):

```text
>>> init logging <<<
2026-07-01 04:41:22,079 - SL - DEBUG - 938 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

`print(">>> init logging <<<")` is emitted at `app/log.py:67`. Every subsequent line follows one uniform format, assembled at `app/log.py:12-15`:

```text
%(asctime)s - %(name)s - %(levelname)s - %(process)d - "%(pathname)s:%(lineno)d" - %(funcName)s() - %(message_id)s - %(message)s
```

That is why each observed line reads `<timestamp> - SL - <LEVEL> - <pid> - "<path>:<lineno>" - <func>() -  - <message>` (the empty segment ` -  - ` is the unused `%(message_id)s`). SimpleLogin also disables Werkzeug’s own access log — `log = logging.getLogger("werkzeug")` at `app/log.py:70` and `log.disabled = True` at `app/log.py:71` — so **the only** request logging you see is SimpleLogin’s own (see sub‑part 2). **Reasoning:** a consistent `SL`‑prefixed line that embeds `pathname:lineno` is proof the app’s logging is initialised and that the code path emitting the line actually executed.

### 2) The app can handle user authentication — `/health` and the auth gate

The explicit liveness probe is `/health`. It is declared with `@app.route("/health", methods=["GET"])` at `server.py:213`, `def healthcheck():` at `server.py:214`, and returns the literal `return "success", 200` at `server.py:215`:

```bash
curl -sS -i http://127.0.0.1:7777/health
```
```text
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 7
Set-Cookie: slapp=...; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Wed, 01 Jul 2026 04:41:39 GMT

success
```

`Content-Length: 7` is exactly `len("success")`, and `Server: Werkzeug/1.0.1 Python/3.10.18` confirms the Werkzeug dev server on the pinned interpreter.

That the **authentication gate is live** is proved by routing: an unauthenticated request to `/` or to a protected page is redirected to `/auth/login`. The root route is `@app.route("/", ...)` at `server.py:250`, `def index():` at `server.py:251`, redirecting authenticated users to `dashboard.index` and everyone else to `auth.login` (`server.py:252-255`):

```text
GET /            -> HTTP/1.0 302 FOUND   Location: http://127.0.0.1:7777/auth/login
GET /dashboard/  -> HTTP/1.0 302 FOUND   Location: http://127.0.0.1:7777/auth/login?next=%2Fdashboard%2F%3F
```

Each such request also produces one SimpleLogin access‑log line from `after_request()` (`server.py:273`); the emitting `LOG.d(` opens at `server.py:284` with format `"%s %s %s %s %s, takes %s"` at `server.py:285`:

```text
2026-07-01 04:41:39,603 - SL - DEBUG - 938 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET / ImmutableMultiDict([]) 302, takes 0.0008058547973632812
2026-07-01 04:41:39,701 - SL - DEBUG - 938 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard/ ImmutableMultiDict([]) 302, takes 0.0009338855743408203
```

Note `/health` is **deliberately excluded** from the access log (`not request.path.startswith("/health")` at `server.py:281`) — and indeed the captured server log contained **zero** `GET /health` access lines, confirming that guard. **Reasoning:** a `302 → /auth/login` on protected paths is direct proof the login/authentication machinery is wired and enforcing access; the `after_request` line proves requests are being served and timed.

> Host note: the `Location` above reads `http://127.0.0.1:7777/...` because the probe used `curl` against `127.0.0.1:7777`. In Q2 the same redirects are exercised through Flask’s test client, whose default host is `localhost`, so those show `http://localhost/...`. Same routes, different observed host — nothing more.

### 3) The app is ready for alias‑based e‑mail — `sl.local` is registered

Alias readiness is governed by `add_sl_domains()` at `init_app.py:39`, which for each configured alias domain either logs that it already exists (`init_app.py:42`) or registers it: `LOG.i("Add %s to SL domain", alias_domain)` at `init_app.py:44` followed by `SLDomain.create(domain=alias_domain, use_as_reverse_alias=True)` at `init_app.py:45`. With `EMAIL_DOMAIN=sl.local`, the first‑registration line captured verbatim is:

```text
2026-07-01 04:39:54,396 - SL - INFO - 831 - "/app/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain
```

Because the seeded database already had `sl.local` registered, this genuine first‑run line was captured on a throwaway database (created, migrated with `alembic upgrade head`, observed, then dropped — see cleanup). The **current registered state** in the running database confirms the same fact directly (`SLDomain.__tablename__ = "public_domain"` at `app/models.py:3119`):

```bash
psql ... -c "SELECT id,domain,use_as_reverse_alias,premium_only FROM public_domain ORDER BY id"
```
```text
1|premium.com|f|t
2|sl.local|t|f
```

`sl.local` is present with `use_as_reverse_alias = t`, precisely what `init_app.py:45` sets. **Reasoning:** aliases can only be created on a registered `SLDomain`; the `Add sl.local to SL domain` log line and the `public_domain` row together prove the local alias domain (`EMAIL_DOMAIN=sl.local`, `example.env:22`) is ready, which is exactly what Q2’s activation step relies on when it mints the default alias `…@sl.local`.

### 4) UI / endpoint readiness signal

The endpoint‑level readiness signals are the two above: `GET /health → 200 success` (an explicit probe intended for exactly this purpose) and the `302 → /auth/login` redirect that renders the login UI for anonymous visitors. Opening `http://localhost:7777` in a browser therefore lands on the login page — the documented first screen (`CONTRIBUTING.md:109`: “open http://localhost:7777, you should be able to login with `john@wick.com / password`”).

---


## Q2 — New‑user walkthrough (register → verify → login → dashboard)

> *"What happens when a user registers verifies their address and tries to log in. What visible behavior confirms that the system is handling every step correctly and forwarding the user into the dashboard as expected?"*

**Short answer.** A brand‑new user posts to `/auth/register` and gets a `200` “waiting” page; a 30‑character activation code is stored and an activation e‑mail is composed; clicking the activation link flips `User.activated` from `False` to `True`, auto‑logs the user in, mints a default `…@sl.local` alias, sends a welcome e‑mail, and `302`‑redirects to `/dashboard/`; a fresh login with the correct password `302`‑redirects to `/dashboard/` (a wrong password stays on `200` with `Email or password incorrect`); and the authenticated `GET /dashboard/` returns `200`, confirming the user is forwarded in.

This flow was exercised end‑to‑end with a **temporary** user, `blitzy.qna.probe@gmail.com`, driven through the real view functions with Flask’s test client (CSRF disabled for the harness only; it changes none of the observed auth/e‑mail behaviour). The user was deleted afterward (see cleanup).

### E‑mail canonicalisation (applies before anything is stored)

`register()` canonicalises the submitted address at `register.py:73` (`email = canonicalize_email(form.email.data)`). `canonicalize_email()` at `app/utils.py:78-94` — for `gmail.com`, `protonmail.com`, `proton.me`, `pm.me` — strips any `+`‑suffix (`app/utils.py:88-89`), removes dots (`app/utils.py:93`) and lower/strips (`app/utils.py:94`); other domains are returned unchanged (`app/utils.py:85`). Observed literally:

```text
canonicalize_email("blitzy.qna.probe@gmail.com") -> "blitzyqnaprobe@gmail.com"
```

So the account is stored and later queried under the canonical `blitzyqnaprobe@gmail.com`. **Reasoning:** knowing this is essential — the DB row does not carry the dotted address, and both `register.py:73` and `login.py:42` rely on the same canonicalisation.

### Step 1 — Register

`POST /auth/register` returns `200` and renders `auth/register_waiting_activation.html` (`return render_template("auth/register_waiting_activation.html")` at `register.py:104`). The observed status and the page’s three literal strings:

```text
REGISTER_STATUS: 200
Page contains "Activation Email Sent"                        -> True
Page contains "An email to validate your email is on its way." -> True
Page contains "Please check your inbox/spam folder."          -> True
```

Those strings come from the template: title `Activation Email Sent` at `templates/auth/register_waiting_activation.html:3`, the `<h1>` `An email to validate your email is on its way.` at `:8`, and `Please check your inbox/spam folder.` at `:9`.

The server‑side proof, logged verbatim (`LOG.d("create user %s", email)` at `register.py:85`; the activation e‑mail via `send_email()` logging at `email_utils.py:303`; the local short‑circuit at `mail_sender.py:131`):

```text
2026-07-01 04:45:22,192 - SL - DEBUG - 1188 - "/app/app/auth/views/register.py:85" - register() -  - create user blitzyqnaprobe@gmail.com
2026-07-01 04:45:22,487 - SL - DEBUG - 1188 - "/app/app/email_utils.py:303" - send_email() -  - send email to blitzyqnaprobe@gmail.com, subject 'Just one more step to join SimpleLogin'
2026-07-01 04:45:22,488 - SL - DEBUG - 1188 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'Just one more step to join SimpleLogin', from '"noreply@sl.local" <noreply@sl.local>' to 'blitzyqnaprobe@gmail.com'
```

Database side effects immediately after register — the user exists but is **not** activated, and a **30‑character** activation code was created (`ActivationCode.create(user_id=user.id, code=random_string(30))` at `register.py:120`; `random_string(30)` at `app/utils.py:41-47` returns 30 lowercase letters):

```text
DB: User id=5  activated=False
DB: ActivationCode = 'aplatrbjqmvxrmmmjomuozselqerbb'   len=30
```

**Reasoning:** the `200` waiting page + the `create user` log + a stored 30‑character `ActivationCode` with `activated=False` together prove registration was accepted and is pending verification — exactly the intended “check your inbox” state.

### Step 2 — Verify (activate)

Visiting the activation link (`GET /auth/activate?code=…`) returns `302` to the dashboard. `activate()` at `activate.py:13-69` sets `user.activated = True` at `activate.py:49`, auto‑logs the user in via `login_user(user)` at `activate.py:50`, consumes the code (`ActivationCode.delete(...)` at `activate.py:53`), flashes success at `activate.py:56`, sends the welcome e‑mail at `activate.py:58`, and finally `LOG.d("redirect user to dashboard")` at `activate.py:66` before `return redirect(url_for("dashboard.index"))` at `activate.py:67`:

```text
ACTIVATE_STATUS: 302   Location: http://localhost/dashboard/
2026-07-01 04:45:22,542 - SL - DEBUG - 1188 - "/app/app/email_utils.py:303" - send_email() -  - send email to simplelogin-newsletter.test608@sl.local, subject 'Welcome to SimpleLogin'
2026-07-01 04:45:22,543 - SL - DEBUG - 1188 - "/app/app/auth/views/activate.py:66" - activate() -  - redirect user to dashboard
```

Database side effects after activation — the flag flipped and a default alias was minted on `sl.local`:

```text
DB: User.activated  False -> True
DB: default alias created: simplelogin-newsletter.test608@sl.local   (1 alias)
```

The success flash `Your account has been activated` (`flash("Your account has been activated", "success")` at `activate.py:56`) is rendered on the page reached after following the `302` into `/dashboard/`. **Reasoning:** the `302 → /dashboard/`, the `activated` flag going `False → True`, the `Welcome to SimpleLogin` e‑mail, and the freshly created `…@sl.local` alias are four independent confirmations that verification succeeded and that the account is now a fully provisioned, logged‑in user.

### Step 3 — Log in (negative path, then success)

With a **fresh** (unauthenticated) session, a wrong password stays on the login page with a flash, and the correct password redirects into the dashboard. `login()` at `login.py:21-82` flashes `Email or password incorrect` at `login.py:49` on bad credentials; on success it calls `after_login(user, next_url)` (`login.py:72`), which logs `LOG.d("log user %s in", user)` at `login_utils.py:35` and `LOG.d("redirect user to dashboard")` at `login_utils.py:44`:

```text
LOGIN (wrong password) -> 200   page contains "Email or password incorrect"   (login.py:49)
LOGIN (correct)        -> 302   Location: http://localhost/dashboard/
2026-07-01 04:45:23,039 - SL - DEBUG - 1188 - "/app/app/auth/views/login_utils.py:35" - after_login() -  - log user <User 5 blitzy.qna.probe@gmail.com blitzyqnaprobe@gmail.com> in
2026-07-01 04:45:23,039 - SL - DEBUG - 1188 - "/app/app/auth/views/login_utils.py:44" - after_login() -  - redirect user to dashboard
```

**Reasoning:** the `200` + `Email or password incorrect` proves the negative path is handled without leaking whether the account exists; the `302 → /dashboard/` plus the `log user … in` line proves credential verification and session establishment succeeded.

### Step 4 — Forwarded into the dashboard

The authenticated `GET /dashboard/` returns `200` (the route `@dashboard_bp.route("/", ...)` at `app/dashboard/views/index.py:55` is guarded by `@login_required` at `:56`; `def index():` at `:67`, rendering `dashboard/index.html` at `:215`). First‑time users get the intro walkthrough, logged at `index.py:172`:

```text
DASHBOARD_STATUS: 200
2026-07-01 04:45:23,047 - SL - DEBUG - 1188 - "/app/app/dashboard/views/index.py:172" - index() -  - Show intro to <User 5 blitzy.qna.probe@gmail.com blitzyqnaprobe@gmail.com>
```

Contrast this with the **unauthenticated** `GET /dashboard/ → 302 /auth/login` from Q1: same URL, but now — because the session is authenticated — it returns `200` and renders the alias‑management dashboard. **Reasoning:** the authenticated `200` together with the `Show intro to <User 5 …>` line is the definitive confirmation that the system forwarded the freshly verified user into the dashboard, exactly as expected.

---


## Q3 — Behind the scenes (background jobs and internal services)

> *"Are there any indicators that background jobs or internal services are doing their part to support email forwarding or identity verification. What should I expect to observe at runtime that tells me these moving pieces are active and talking to each other properly?"*

**Short answer.** SimpleLogin runs several processes besides the web app: the inbound **SMTP server** (`email_handler.py`, the e‑mail‑forwarding entrypoint), the **job runner** (`job_runner.py`, a 10‑second poll loop over the Postgres `job` table), the **scheduled‑task runner** (`cron.py`, driven by `crontab.yml`), and the **event listener** (`event_listener.py`) that consumes a Postgres `NOTIFY`/`LISTEN` channel. At runtime you can observe each starting and doing work, all sharing the same PostgreSQL and Redis — which is what proves they are active and communicating.

### 1) Which internal services support the flow

| Service | Entrypoint | Role in the flow | Boot / liveness citation |
|---|---|---|---|
| Web app | `server.py` `create_app()` `:139` | Serves register/activate/login/dashboard; writes Postgres; uses Redis sessions | banner + `/health` (Q1) |
| Inbound SMTP | `email_handler.py` `main()` `:2381` | Receives mail and **forwards** alias → mailbox | `:2386`, `:2403` (below) |
| Job runner | `job_runner.py` `:329-347` | Executes async/onboarding jobs from the `job` table | `:334`, `:304` (below) |
| Scheduled tasks | `cron.py` `:1262-1263` | Periodic maintenance (stats, cleanups) per `crontab.yml` | `:1263` (below) |
| Event listener | `event_listener.py` | Consumes Postgres sync events (`LISTEN`) | `:15-17`, `:35` (below) |

**Identity‑verification support** is carried by the web app + e‑mail composition (the activation e‑mail in Q2), while **e‑mail forwarding** is the job of `email_handler.py`. **Redis** underpins both by storing sessions and the rate‑limit counters: `initialize_redis_services(app, MEM_STORE_URI)` at `server.py:165` (with `MEM_STORE_URI=redis://localhost`). The web app read/writes **PostgreSQL** throughout (every DB side effect in Q1/Q2 is proof of that link, and Redis `PONG` + the `Set-Cookie: slapp=…` in the `/health` response confirm the session store).

### 2) Indicator that the e‑mail‑forwarding service is active

Booting the inbound SMTP server shows it binds and starts its controller. `email_handler.py` logs `LOG.i("Listen for port %s", args.port)` at `:2403` (default `20381` at `:2399`) and, inside `main()`, `LOG.d("Start mail controller %s %s", controller.hostname, controller.port)` at `:2386` (the controller itself is `Controller(MailHandler(), hostname="0.0.0.0", port=port)` at `:2383`):

```bash
CONFIG=/root/sl.env python email_handler.py
```
```text
2026-07-01 04:47:14,913 - SL - INFO - 1326 - "/app/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-01 04:47:14,915 - SL - DEBUG - 1326 - "/app/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

The actual per‑message forwarding is logged by `LOG.d("Forward %s -> %s -> %s", contact, alias, mailbox)` at `email_handler.py:688`. That line only fires when a real inbound message is delivered to an alias, which is outside the register/verify/login flow (it needs an inbound SMTP delivery); it is cited here as the code path, and what was **observed at runtime** is that the forwarding service **is up and listening on `0.0.0.0:20381`**. **Reasoning:** `Listen for port 20381` + `Start mail controller 0.0.0.0 20381` prove the alias‑forwarding entrypoint is active and bound; the `:688` reference names exactly where a forward would be logged.

### 3) Indicator that background jobs run (and that they poll Postgres)

The job runner is a poll loop: `while True:` at `job_runner.py:330`, taking each due job (`LOG.d("Take job %s", job)` at `:334`), marking it taken, calling `process_job(job)` at `:342`, marking it `done` at `:344`, then `time.sleep(10)` at `:347` (the ≈10‑second cadence). Jobs come from the Postgres `job` table via `get_jobs_to_run()` at `:307-326`.

To observe it safely, a throwaway sentinel job with an unknown name was enqueued, then one poll cycle was run (the sentinel deliberately hits the “unknown job” branch so it has no side effects — `LOG.e("Unknown job name %s", job.name)` at `:304`). Enqueue → run → the DB row transition:

```text
Enqueued: Job id=1  name='blitzy-observation-noop'  payload={'probe': True}  state=0(ready) taken=f attempts=0

2026-07-01 04:46:46,708 - SL - DEBUG - 1279 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 1 blitzy-observation-noop {'probe': True}>
2026-07-01 04:46:46,711 - SL - ERROR - 1279 - "/app/job_runner.py:304" - process_job() -  - Unknown job name blitzy-observation-noop

DB after one cycle:  id=1  name='blitzy-observation-noop'  state=2(done)  taken=t  attempts=1
```

The state literally moved `ready(0) → done(2)` with `taken=true`, `attempts=1` (`JobState.ready=0`, `taken=1`, `done=2` at `app/models.py:254-256`). **Reasoning:** the runner **took** a row from the Postgres `job` table, **processed** it and **closed** it on its poll loop — proof the background‑job machinery (which is what dispatches onboarding e‑mails and other async work in `process_job()` at `:188`) is active and talking to Postgres. The sentinel job was deleted afterward.

The scheduled‑task runner starts cleanly too — `LOG.d("Start running cronjob")` at `cron.py:1263` (dispatched by `-j/--job` and scheduled by `crontab.yml`). Booting it with a non‑matching job name is side‑effect‑free (no dispatch branch matches):

```text
2026-07-01 04:47:59,869 - SL - DEBUG - 1379 - "/app/cron.py:1263" - <module>() -  - Start running cronjob
```

### 4) The event bus, and how to tell the pieces are talking

SimpleLogin has **two distinct** event mechanisms; distinguishing them is important:

- **Postgres sync‑event bus (LISTEN/NOTIFY).** `app/events/event_dispatcher.py` defines `NOTIFICATION_CHANNEL = "simplelogin_sync_events"` at `:14`; `PostgresDispatcher.send()` issues `Session.execute(f"NOTIFY {NOTIFICATION_CHANNEL}, '{instance.id}';")` at `:26`, and `event_listener.py` is the consumer that `LISTEN`s on that channel (its `Mode` enum `LISTENER`/`DEAD_LETTER` at `event_listener.py:15-17`, `PostgresEventSource(EVENT_LISTENER_DB_URI)` at `:35`). During the Q2 flow, `EventDispatcher.send_event()` (`:49`) was reached, but locally it short‑circuits at `:62` because no webhook is configured:

  ```text
  2026-07-01 04:45:22,463 - SL - INFO - 1188 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
  ```

  That line (message string at `event_dispatcher.py:63`) is the expected, healthy local behaviour: the sync‑event path is wired and invoked, but it is quiet because no `EVENT_WEBHOOK` is set — so no partner‑sync `NOTIFY` is emitted in this local setup.

- **NewRelic custom events (analytics).** Separately, `app/events/auth_event.py` records product analytics: `class LoginEvent` at `:6` calls `newrelic.agent.record_custom_event("LoginEvent", …)` at `:23`, and `class RegisterEvent` at `:28` calls `newrelic.agent.record_custom_event("RegisterEvent", …)` at `:45`. These are **not** the Postgres sync‑event system; they are telemetry that is effectively a no‑op locally (no NewRelic license configured).

Finally, the reason the activation and welcome e‑mails are observable **as log lines** at all is the local short‑circuit: with `NOT_SEND_EMAIL=true`, `mail_sender.py:130` takes the `if config.NOT_SEND_EMAIL:` branch, logs the subject/from/to via `LOG.d(` at `:131` and `return True` at `:137` without contacting a real relay. That is why every e‑mail in Q2/Q3 appears as a `mail_sender.py:131` line rather than being delivered.

**Reasoning (all pieces active and communicating):** the web app writes Postgres and uses Redis; `email_handler.py` binds `:20381`; `job_runner.py` reads and closes a Postgres `job` row on its 10‑second loop; `cron.py` starts its runner; and the `simplelogin_sync_events` `NOTIFY`/`LISTEN` bus is wired (with the local “not sending events” log explaining its silence). Because these separate processes all operate against the **same** PostgreSQL and Redis, observing each one act is exactly what confirms the moving parts are alive and talking to one another.

---


## Coverage pass

Re‑reading each question and confirming every sub‑part is answered with observed evidence:

**Q1 — “what should I notice in the logs or UI that shows the app is ready to handle user authentication and alias based email activity?”**

- [x] *Logs show it booted / logging ready* — `>>> init logging <<<` (`app/log.py:67`) + the uniform `SL` format (`app/log.py:12-15`). → Q1 §1.
- [x] *Ready for user authentication* — `GET /health → 200 success` (`server.py:215`) and unauthenticated `GET / , /dashboard/ → 302 /auth/login` (`server.py:250-255`), plus the `after_request` access log at `server.py:284`. → Q1 §2.
- [x] *Ready for alias‑based e‑mail* — `Add sl.local to SL domain` (`init_app.py:44`) and the `public_domain` row for `sl.local`. → Q1 §3.
- [x] *UI/endpoint readiness signal* — `/health` probe and the `302 → /auth/login` login screen. → Q1 §4.

**Q2 — “What happens when a user registers verifies their address and tries to log in. What visible behavior confirms that the system is handling every step correctly and forwarding the user into the dashboard as expected?”**

- [x] *Register* — `POST /auth/register → 200`, waiting page text (`register_waiting_activation.html:3,8,9`), `create user` (`register.py:85`), 30‑char `ActivationCode` (`register.py:120`), `activated=False`. → Q2 Step 1.
- [x] *Verify* — `GET /auth/activate → 302 /dashboard/`, `activated False→True` (`activate.py:49`), welcome e‑mail (`activate.py:58`), default alias `…@sl.local`, success flash (`activate.py:56`). → Q2 Step 2.
- [x] *Log in* — wrong password `200` + `Email or password incorrect` (`login.py:49`); correct password `302 /dashboard/` + `log user … in` (`login_utils.py:35`). → Q2 Step 3.
- [x] *Visible behaviour confirming each step* — HTTP statuses, redirect `Location`s, flash/page text and DB side effects quoted at every step. → Q2 Steps 1–4.
- [x] *Forwarded into the dashboard* — authenticated `GET /dashboard/ → 200` + `Show intro to <User 5 …>` (`dashboard/views/index.py:172`), contrasted with the unauthenticated `302`. → Q2 Step 4.

**Q3 — “Are there any indicators that background jobs or internal services are doing their part to support email forwarding or identity verification. What should I expect to observe at runtime that tells me these moving pieces are active and talking to each other properly?”**

- [x] *Which background jobs / internal services* — web app, `email_handler.py`, `job_runner.py`, `cron.py`, `event_listener.py` (table + citations). → Q3 §1.
- [x] *Support for e‑mail forwarding* — `Listen for port 20381` + `Start mail controller 0.0.0.0 20381` (`email_handler.py:2403`, `:2386`); forward path at `:688`. → Q3 §2.
- [x] *Support for identity verification* — the activation/welcome e‑mails composed during the flow (`email_utils.py:303`) made observable by the `NOT_SEND_EMAIL` short‑circuit (`mail_sender.py:131`); job runner backs async/onboarding work. → Q3 §3–§4.
- [x] *Proof the pieces are active and communicating* — job `ready(0)→done(2)` on the Postgres `job` table (`job_runner.py:334`, `:304`), `Start running cronjob` (`cron.py:1263`), the `simplelogin_sync_events` NOTIFY/LISTEN bus with the local `Not sending events…` log (`event_dispatcher.py:62`), all against the shared Postgres/Redis. Postgres sync‑events explicitly distinguished from NewRelic `LoginEvent`/`RegisterEvent` (`auth_event.py:23,45`). → Q3 §4.

**Explicitly noted as not runtime‑exercised (stated rather than asserted):** a full inbound alias‑forward that would emit `Forward … -> … -> …` (`email_handler.py:688`) was not triggered — it requires delivering an inbound message to the running SMTP listener, which is outside the register/verify/login scenario. What was verified at runtime is that the SMTP forwarding service starts and listens on `0.0.0.0:20381`. Likewise the Postgres `NOTIFY` payload was not emitted locally because no `EVENT_WEBHOOK` is configured (the app logged exactly that, `event_dispatcher.py:62`).

---

## Cleanup and read‑only verification

All temporary artefacts created for the investigation were removed, restoring the seeded baseline and leaving the repository byte‑for‑byte unchanged:

- Temporary user `blitzy.qna.probe@gmail.com` (id 5) deleted via `User.delete(id, commit=True)`, which cascades its `simplelogin-newsletter.test608@sl.local` alias.
- Sentinel `Job` (`blitzy-observation-noop`) deleted; the `job` table returned to 0 rows.
- Throwaway database `simplelogin_probe` (used only to capture the first‑run `Add sl.local to SL domain` line) dropped; the out‑of‑repo probe config removed.
- All temporary observation scripts and captured‑output files (kept outside the repository, e.g. under `/tmp`) removed. The out‑of‑repo `CONFIG` file, the virtualenv, and the transient `GNUPGHOME` directories live entirely outside the repository.
- Database restored to the `flask dummy-data` baseline: `john@wick.com` and `winston@continental.com` only.
- `git status --porcelain` verified to contain only this new document (`blitzy/documentation/app_2cd6ee777f8c.md`); no existing source file was modified, added or deleted (`*.pyc` is ignored per `.gitignore`).

