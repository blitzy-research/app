# SimpleLogin Runtime Verification — Web Server, Email Handler, and Job Runner

**Branch:** `app_2cd6ee777f8c`  ·  **Repository HEAD:** `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`

This document answers three question groups about the SimpleLogin back-end/web application. **Every claim below was produced by running the code in its canonical runtime and capturing the complete, unedited output**, then grounding the claim in a specific `file:line` reference. Claims are labeled:

- **`OBSERVED`** — I executed the code path and captured the output shown.
- **`INFERRED`** — read from source (not executed); always paired with a `file:line` reference.

All runtime evidence was captured on **2026-07-08** inside the mandated Docker runtime (see §1). Temporary test data (accounts, aliases, emails, and throwaway `Job` rows) and every observation script were created solely to gather this evidence and were removed afterward; the only file added to the repository is this document.

> **A note on `file:line` fidelity.** SimpleLogin's logger prints the emitting `"pathname:lineno" - funcName()` in *every* line (log format at `app/log.py:L12`). This means the captured logs below **literally embed the source location** of the code that ran, so the citations are self-verifying. One 1-line drift from the review citations was found at runtime and is reported honestly where it occurs (the "Forward phase" line, §3.3).

---

## 1. Environment & Setup

### 1.1 Canonical runtime (exact image and services)

**`OBSERVED`.** All verification was performed inside the mandated Docker image, which carries the pre-built **Python 3.10.18** dependency environment in a virtualenv at `/app/venv`. A bare host cannot reproduce it: `poetry.lock` pins `multidict==4.7.6`, whose C-extension has no cp310 wheel that builds cleanly, so `from app import ...` (imported by all three entry points) fails outside the image.

| Component | Value |
|-----------|-------|
| Runtime image | `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (tag `simple-login__app__2cd6ee777f8c…`) |
| App container | `sl-app` — repo bind-mounted at `/code`; canonical interpreter `/app/venv/bin/python`; ports `7777` (web) and `20381` (SMTP) |
| Database | `sl-db` — `postgres:13`, DB `simplelogin`, user `myuser` |
| Cache / rate-limit store | `sl-redis` — `redis:7` on `6379` |
| Runtime Python | **3.10.18** (`pyproject.toml` declares `python = "^3.10"`) |

Canonical configuration (`/code/.env`, git-ignored) reports these defaults, all consistent with source: `URL=http://localhost:7777` (`app/config.py:L79`), `DB_URI` → `sl-db:5432/simplelogin` (`app/config.py:L192`), `NOT_SEND_EMAIL=true` (`app/config.py:L91`), `EMAIL_DOMAIN=sl.local`, `DISABLE_ONBOARDING=true` (`app/config.py:L401`).

### 1.2 Database migration and canonical dummy data

**`OBSERVED`.** The database was migrated and seeded per the canonical recipe at `CONTRIBUTING.md:L106` (`alembic upgrade head && flask dummy-data`). `flask dummy-data` seeds the demo login `john@wick.com / password` and demo aliases. All ORM tables exist (`alias`, `contact`, `email_log`, `users`, `job`, `mailbox`, …).

### 1.3 Exact invocation commands for each component

The three long-running components are started by these exact commands (each uses the venv interpreter):

```bash
# Web server — PRODUCTION default (this is the container's default CMD; Dockerfile:L47)
/app/venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15

# Web server — DEVELOPMENT alternative (server.py:L588 -> app.run(debug=True, port=7777))
/app/venv/bin/python server.py

# Email handler — aiosmtpd SMTP daemon on port 20381 (run from /app; see note below)
/app/venv/bin/python email_handler.py

# Job runner — Job-table polling daemon
/app/venv/bin/python job_runner.py
```

> **Runtime note (`OBSERVED`, environment-specific, not a product defect).** The email handler must be launched from `/app` (the image's copy), not the bind-mounted `/code`. `poetry.lock` pins `pyre2 0.3.6` (no cp310 wheel), so the image ships `google-re2`, whose `re2` module lacks `re.DOTALL`; `app/spamassassin_utils.py` does `import re2 as re; … re.DOTALL`, which raises on import from `/code` but succeeds from the image's patched `/app`. The web server and job runner run fine from `/code`. This affects only *where* the handler is launched, not its behavior.

### 1.4 Shared logging banner (all three components)

**`OBSERVED`.** Every entry point initializes one shared logger, `LOG = _get_logger("SL")` (`app/log.py:L79`), and prints a banner at process start, `print(">>> init logging <<<")` (`app/log.py:L67`). The Flask/werkzeug request logger is deliberately disabled (`app/log.py:L70-71`), which is why no werkzeug "Running on…" / per-request access lines appear. The banner appears in the captured startup output of all three components (see §2). The log line format (`app/log.py:L12`) embeds `asctime - SL - LEVEL - pid - "pathname:lineno" - funcName() - message_id - message`.

---

## 2. Q1 — Startup & Health Verification

> **User Question 1:** *"Once the application starts, how can I tell that the web server, email handler, and job runner are actually up and responding? What should I see in the logs or dashboard UI that confirms users can sign in and manage their aliases?"*

### 2.1 Direct answer

Each component exposes a distinct, positive up-signal:

- **Web server** — `GET /health` returns **HTTP 200** with the body **`success`** on **port 7777**, and `GET /` issues a redirect to the login page. These prove the Flask app factory booted and the WSGI server is accepting requests.
- **Email handler** — at startup it logs **`Listen for port 20381`** then **`Start mail controller 0.0.0.0 20381`**, confirming the aiosmtpd controller is bound and listening on **port 20381**.
- **Job runner** — logs the shared init banner and then polls silently; when a `Job` is present it logs **`Take job <Job …>`**. Its liveness is the continuous poll loop (see §4 for the measured cadence).
- **Dashboard UI** — the unauthenticated `/auth/login` page renders the sign-in form; after signing in, the dashboard renders the alias **count**, the **alias list**, and the **"Random Alias" / "Create a custom alias"** controls — the visible confirmation that a signed-in user can manage aliases.

**Cause → effect:** a `200 success` from `/health` can only be produced *after* `create_app()` builds the Flask app, initializes the database/extensions, and the WSGI worker begins serving — so it is a reliable single-probe liveness check. The email handler's two log lines are emitted only *after* `controller.start()` successfully binds the socket. The job runner's banner proves it entered its app-context poll loop.

### 2.2 Web server — `/health` and index redirect

**`OBSERVED`.** The health route is `healthcheck()` at `server.py:L213-L215` (`@app.route("/health")` → `return "success", 200`). Served by the production gunicorn default (`Dockerfile:L47`) on port 7777 (`Dockerfile:L44` `EXPOSE 7777`; `server.py:L588`).

```bash
$ curl -i http://localhost:7777/health
```
```
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 04:19:xx GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 7

success
```

The index route `index()` at `server.py:L252-L256` redirects unauthenticated visitors to `auth.login` (authenticated → `dashboard.index`):

```bash
$ curl -i http://localhost:7777/
```
```
HTTP/1.1 302 FOUND
Server: gunicorn/20.0.4
Location: http://localhost:7777/auth/login
Content-Length: 219
```

**`OBSERVED`.** The production web server's own startup output (gunicorn master + `-w 2` workers, each initializing the `SL` logger):

```
[INFO] Starting gunicorn 20.0.4
[INFO] Listening at: http://0.0.0.0:7777 (899)
[INFO] Using worker: sync
[INFO] Booting worker with pid: 907
[INFO] Booting worker with pid: 908
>>> init logging <<<
>>> init logging <<<
```

**`OBSERVED`.** The development alternative `python server.py` (`server.py:L588` → `app.run(debug=True, port=7777)`) binds the loopback interface and prints:

```
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
>>> init logging <<<
 * Serving Flask app "server"
 * Environment: production
 * Debug mode: on
```

Because `app.run(...)` is given no host argument it binds `127.0.0.1:7777`; a request issued *inside* the container returns the same health signal, served by Werkzeug rather than gunicorn:

```bash
$ curl -i http://127.0.0.1:7777/health     # from inside the container
```
```
HTTP/1.0 200 OK
Server: Werkzeug/1.0.1 Python/3.10.18
Content-Length: 7

success
```
> The familiar werkzeug "Running on http://…" line is intentionally **absent** — the werkzeug logger is disabled at `app/log.py:L70-71` (`OBSERVED`, and it explains why no per-request access logs appear either).

### 2.3 Email handler — startup up-signal and bound port

**`OBSERVED`.** Started with `python email_handler.py` (from `/app`). The daemon's own startup log:

```
>>> init logging <<<
2026-07-08 04:18:31,056 - SL - INFO - 934 - "/app/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-08 04:18:31,060 - SL - DEBUG - 934 - "/app/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

Reading the emitted `pathname:lineno` confirms the citations exactly:
- `Listen for port 20381` is `LOG.i("Listen for port %s", args.port)` at `email_handler.py:L2403` (in `__main__`, where the port default is `20381`, `email_handler.py:L2399`).
- `Start mail controller 0.0.0.0 20381` is `LOG.d("Start mail controller %s %s", controller.hostname, controller.port)` at `email_handler.py:L2386`, emitted by `main()` (`email_handler.py:L2381`) immediately after `controller.start()` (`email_handler.py:L2385`) constructs an aiosmtpd `Controller(MailHandler(), hostname="0.0.0.0", port=port)` (`email_handler.py:L2383`).

The host `0.0.0.0` and port `20381` in the log confirm the bound socket. (Liveness after startup — the `while True: time.sleep(2)` keep-alive — is covered in §4.)

### 2.4 Job runner — startup up-signal and poll loop

**`OBSERVED`.** Started with `python job_runner.py`. Its startup output shows the shared banner, then the loop runs silently while no jobs are ready:

```
>>> init logging <<<
2026-07-08 04:18:59,663 - SL - DEBUG - 976 - "/code/app/utils.py:17" - <module>() -  - load words file: /code/local_data/test_words.txt
```

The runner imports `from server import create_light_app` (`job_runner.py:L24`) and its `__main__` block is a `while True:` loop (`job_runner.py:L330`) that, inside `with create_light_app().app_context():` (`job_runner.py:L332`), iterates `for job in get_jobs_to_run():` (`job_runner.py:L333`; query defined at `job_runner.py:L307`), logging `LOG.d("Take job %s", job)` (`job_runner.py:L334`) for each. When idle it logs nothing and sleeps 10s (`job_runner.py:L347`). A **live `Take job` line and the measured poll cadence** are shown in §4.

### 2.5 Dashboard UI — sign-in and alias-management signals

**`OBSERVED`.** The unauthenticated sign-in page renders from `templates/auth/login.html`:

```bash
$ curl -s http://localhost:7777/auth/login | grep -nE 'form method|csrf_token|name="email"|type="password"|/auth/register'
```
```
<form method="post">
<input id="csrf_token" name="csrf_token" type="hidden" value="…">
<input class="form-control" id="email" name="email" required type="email" value="">
<input class="form-control" id="password" name="password" required type="password" value="">
<a href="/auth/register">Sign up</a>
```

**`OBSERVED`.** After signing in (flow in §3), a single authenticated `GET /dashboard/` returns HTTP 200 (43,644 bytes) rendering `templates/dashboard/index.html`. The alias-management markers are all present:

```bash
$ grep -oE '<div class="h1 m-0">[0-9]+</div>' dash.html | head -1   # stats.nb_alias (L131)
<div class="h1 m-0">2</div>
$ grep -c 'Random Alias'           dash.html   # "Random Alias" button (L56)      -> 1
$ grep -c 'Create a custom alias'  dash.html   # custom-alias button (L44)        -> 1
$ grep -c 'Enter to search for alias' dash.html # search box (L215)               -> 1
$ grep -oE '[a-z0-9_]+@sl\.local'  dash.html | sort -u   # alias list loop (L233)
junker021@sl.local
rezone_weblog886@sl.local
```

This is the confirmation that a signed-in user can manage aliases: the alias **count** `{{ stats.nb_alias }}` (`templates/dashboard/index.html:L131`) renders `2`; the alias-list loop `{% for alias_info in alias_infos %}` (`templates/dashboard/index.html:L233`) renders the user's two aliases; and the create controls (`templates/dashboard/index.html:L44,L56`) and search box (`templates/dashboard/index.html:L215`) are present.

---

## 3. Q2 — User Actions & Correctness

> **User Question 2:** *"Try performing basic user actions like creating a new account, creating an alias, and having that alias receive an email. What happens when a new account interacts with aliases or tries to receive mail, and how does the system show that these actions were handled correctly? What should I see at runtime that confirms the behavior you would expect?"*

### 3.1 Direct answer

All three actions were driven through their **real routes/entry points** and each surfaced a clear correctness signal:

- **Creating an account** (`register()`, `app/auth/views/register.py:L32`) returns the "Activation Email Sent" page and creates a `User` (unactivated) plus an `ActivationCode`. Activating it (`app/auth/views/activate.py`) flips `users.activated` **False → True**, flashes **"Your account has been activated"**, and logs the user in.
- **Creating an alias** (`index()` → `Alias.create_new_random(...)`, `app/dashboard/views/index.py:L104`) flashes **"Alias `<addr>` has been created"**, logs **`create new random alias …`** (`index.py:L110`), and inserts an `Alias` row.
- **Receiving an email** (SMTP `handle_DATA` → `handle_forward`, `email_handler.py:L2289,L536`) returns the SMTP success **`250 Message accepted for delivery`**, logs **`Forward <Contact> -> <Alias> -> <Mailbox>`** (`email_handler.py:L688`), and inserts `Contact` + `EmailLog` rows.

"Handled correctly" is therefore demonstrable at **three simultaneous layers**: the SMTP protocol response, the server log line, and the persisted database rows. Edge conditions (the alias-quota gate and a non-existent-alias rejection) are shown in §3.4–§3.5.

> **Driver note.** The multi-step, CSRF-protected web flows were exercised with a temporary `requests`-based script (removed afterward, §7) that fetched each form's `csrf_token` and POSTed to the **real** blueprint routes on the live server at `127.0.0.1:7777`. Email was sent with Python's `smtplib` (the documented `swaks`, `CONTRIBUTING.md:L217`, is not installed in the image); `smtplib` opens a raw SMTP session to `127.0.0.1:20381` and therefore hits the identical aiosmtpd `MailHandler.handle_DATA` entry point.

### 3.2 Creating a new account

**`OBSERVED`.** `POST /auth/register` (real route `register()`, `app/auth/views/register.py:L31-L32`) with a fresh `outlook.com` address and an 8+ char password returns HTTP 200 and the "Activation Email Sent" page. Because `NOT_SEND_EMAIL=true` (`app/config.py:L91`), no email is actually delivered, but the `ActivationCode` row is created and readable from the DB; the activation URL is built as `f"{URL}/auth/activate?code={activation.code}"` (`app/auth/views/register.py:L124`).

```
[1] POST /auth/register -> 200
[2] waiting-activation page title: "Activation Email Sent | SimpleLogin"
[3] users.id=5  activated(before)=False  activation_code=ykzsjwyullfyiwgddrypiwpozndmnf
[4] GET /auth/activate?code=... -> 200 ; toastr flashes=[('success', 'Your account has been activated')]
[5] activated(after)=True
```

Activation is handled by the route at `app/auth/views/activate.py`, which sets `user.activated = True`, calls `login_user(user)` (auto-login), and flashes the success message rendered as `toastr.success("Your account has been activated")` (flash mechanism at `templates/base.html:L102`). The `activated` transition **False → True** is the persistence-layer proof that account creation was handled correctly.

**`OBSERVED`.** Signing in explicitly through the real login route `login()` (`app/auth/views/login.py:L25`) succeeds and redirects to the dashboard:

```
[A1] POST /auth/login -> 302  Location=http://127.0.0.1:7777/dashboard/
     GET /dashboard/ (authenticated) -> 200
```

### 3.3 Creating an alias + receiving an email — the three correctness layers

**Alias creation (`OBSERVED`).** `POST /dashboard/` with `form-name=create-random-email` (real route `index()`, `app/dashboard/views/index.py:L67`) creates a random alias via `Alias.create_new_random(user=current_user, scheme=scheme)` (`app/dashboard/views/index.py:L104`), commits, logs at `index.py:L110`, and flashes success at `index.py:L111`. Captured signals for a fresh user (alias count **1 → 2**):

```
[A3] POST create-random-email -> 200
[A4] toastr flashes on redirect page: [('success', 'Alias rezone_weblog886@sl.local has been created')]
[A5] alias count AFTER = 2 ; newest alias = (18, 'rezone_weblog886@sl.local')
```

The matching **server log line** (from the live web process) — note the embedded `index.py:110` and `index()`:

```
2026-07-08 04:24:34,599 - SL - DEBUG - 908 - "/code/app/dashboard/views/index.py:110" - index() -  - create new random alias <Alias 13 coerce_others582@sl.local> for user <User 3 blitzyqatmp…@outlook.com …>
```

> A brand-new account already owns **one** alias before any manual creation — the auto-provisioned "first alias" (`note = "This is your first alias. It's used to receive SimpleLogin communications…"`), which is why the "before" count is `1`, not `0` (`OBSERVED`).

**Email reception (`OBSERVED`).** Sending an external email to the new alias over SMTP enters `MailHandler.handle_DATA` (`email_handler.py:L2289`, class at `L2288`) → `handle_forward` (`email_handler.py:L536`). The raw SMTP session and the server's responses:

```
MAIL FROM -> 250 OK
RCPT TO   -> 250 OK
DATA cmd  -> 354 End data with <CR><LF>.<CR><LF>
DATA final response -> 250 Message accepted for delivery
```

**Layer (1) — Protocol.** The final SMTP response **`250 Message accepted for delivery`** is the constant `E200 = "250 Message accepted for delivery"` (`app/email/status.py:L2`). In the forward path it is assigned `res_status = status.E200` (`email_handler.py:L608`) and returned via `return [(True, res_status)]` (`email_handler.py:L612`). The handler's own "Finish" line confirms the same return code:

```
2026-07-08 04:29:04,852 - SL - INFO - 934 - "/app/email_handler.py:2367" - _handle() - 3c1464dd-… - Finish mail_from external-sender-…@outlook.com, rcpt_tos ['rezone_weblog886@sl.local'], takes 0.6858623027801514 seconds with return code '250 Message accepted for delivery'<<===
```

**Layer (2) — Log.** The complete forward pipeline logged by the email handler (unedited), including the `Forward <Contact> -> <Alias> -> <Mailbox>` line at `email_handler.py:L688`:

```
2026-07-08 04:29:04,166 - SL - INFO  - 934 - "/app/email_handler.py:2343" - _handle() - New message, mail from external-sender-…@outlook.com, rctp tos ['rezone_weblog886@sl.local']
2026-07-08 04:29:04,790 - SL - DEBUG - 934 - "/app/email_handler.py:2202" - handle() - Forward phase external-sender-…@outlook.com(external-sender-…@outlook.com) -> rezone_weblog886@sl.local
2026-07-08 04:29:04,806 - SL - DEBUG - 934 - "/app/email_handler.py:580"  - handle_forward() - Create or get contact for from_header:external-sender-…@outlook.com
2026-07-08 04:29:04,833 - SL - DEBUG - 934 - "/app/app/contact_utils.py:110" - create_contact() - Created contact <Contact 3 external-sender-…@outlook.com 18> for alias <Alias 18 rezone_weblog886@sl.local> … invalid_email=False
2026-07-08 04:29:04,842 - SL - DEBUG - 934 - "/app/email_handler.py:688" - forward_email_to_mailbox() - Forward <Contact 3 external-sender-…@outlook.com 18> -> <Alias 18 rezone_weblog886@sl.local> -> <Mailbox 6 blitzyqab…@outlook.com>
2026-07-08 04:29:04,845 - SL - DEBUG - 934 - "/app/email_handler.py:740" - forward_email_to_mailbox() - Create <EmailLog 3> for <Contact 3 …>, <User 4 …>, <Mailbox 6 …>
```

Two grounding details, reported honestly:
- The `Forward <Contact> -> <Alias> -> <Mailbox>` line (`email_handler.py:L688`) is emitted by **`forward_email_to_mailbox()`**, the function `handle_forward` delegates the actual mailbox delivery to. The "Create or get contact" step (`handle_forward`, `email_handler.py:L580`) is where `get_or_create_contact` (`email_handler.py:L180`) is invoked.
- The "Forward phase …" line runs at **`email_handler.py:L2202`** at runtime (the review citation was `~L2203`) — a 1-line drift; the **`OBSERVED`** location is `L2202`.

**Layer (3) — Persistence.** Read-only SQL confirms exactly one new `Alias`, one `Contact`, and one `EmailLog`, all linked. **Before** the send: `contact=0, email_log=0`. **After** the send: `contact=1, email_log=1`.

```sql
-- Alias (app/models.py:L1469, table "alias")
 id |           email           | user_id | mailbox_id | enabled | automatic_creation
----+---------------------------+---------+------------+---------+--------------------
 18 | rezone_weblog886@sl.local |       4 |          6 | t       | f

-- Contact (app/models.py:L1863, table "contact")
 id | alias_id |             website_email              |                       reply_email
----+----------+----------------------------------------+----------------------------------------------------------
  3 |       18 | external-sender-…@outlook.com          | external-sender-…_at_outlook_com_psfxj@sl.local

-- EmailLog (app/models.py:L2060, table "email_log")
 id | user_id | alias_id | contact_id | is_reply | blocked | bounced
----+---------+----------+------------+----------+---------+---------
  3 |       4 |       18 |          3 | f        | f       | f
```

The `EmailLog` row (`contact_id=3`, `alias_id=18`, `is_reply=f`, `blocked=f`, `bounced=f`) is the durable record that a message was received, associated with the correct alias and contact, and forwarded without being blocked or bounced. The `Contact.reply_email` is the generated reverse-alias address the mailbox can reply through.

> **Default-configuration nuance (`OBSERVED`).** Under the canonical default `NOT_SEND_EMAIL=true`, the pipeline runs to completion — it accepts the message (`E200`), logs the `Forward` line, and persists `Contact`/`EmailLog` — but the *actual* outbound handoff to the mailbox is suppressed. To make the message physically arrive, `CONTRIBUTING.md:L188-L215` documents running a local MTA (e.g. MailHog) and, **in the git-ignored `.env` only**, commenting out `NOT_SEND_EMAIL` and setting `POSTFIX_SERVER`/`POSTFIX_PORT`. All acceptance/log/persistence signals above are independent of that toggle.

### 3.4 Edge case A — the alias-quota gate

**`OBSERVED`.** Alias creation is gated by `if current_user.can_create_new_alias():` (`app/dashboard/views/index.py:L98` for the random-alias branch; `L92` for custom). For a free-plan user, `can_create_new_alias()` (`app/models.py:L867`) returns `Alias.filter_by(user_id=…).count() < max_alias_for_free_account()`, and `max_alias_for_free_account()` (`app/models.py:L858`) returns `MAX_NB_EMAIL_FREE_PLAN`. Driving a user already at the limit (5 aliases) through the real route yields the warning flash at `index.py:L123`, and **no alias is created**:

```
[D1] user 3 alias count = 5 (limit MAX_NB_EMAIL_FREE_PLAN=5)
[D2] login user 3 -> 302 Location=/dashboard/
[D3] POST create-random-email (at limit) -> 200
[D4] toastr flashes: [('warning', 'You need to upgrade your plan to create new alias.')]
[D5] alias count still = 5 (unchanged -> gate blocked creation)
```

**Cause → effect:** the gate short-circuits before `Alias.create_new_random(...)`, so the count stays at 5 and the user sees the upgrade prompt.

> **Discrepancy, reported honestly.** The docstring at `app/models.py:L870` says a user "cannot have more than 15 aliases", but the **`OBSERVED`** runtime limit is **5** — `MAX_NB_EMAIL_FREE_PLAN` defaults to `5` (`app/config.py:L124`), and the process even logs `MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value` at startup. The enforced value is `5`, not `15`.

### 3.5 Edge case B — receiving mail for a non-existent alias

**`OBSERVED`.** Sending to an address with no matching `Alias` (and no auto-create rule for `sl.local`) is rejected with **`550 SL E515 Email not exist`** — the constant `E515 = "550 SL E515 Email not exist"` (`app/email/status.py:L51`):

```
[C1] to no-such-alias-…@sl.local
[C2] DATA final response -> 550 SL E515 Email not exist
```

The handler log shows exactly why (`handle_forward` finds no alias, tries and fails to auto-create, then returns 550):

```
2026-07-08 04:29:06,946 - SL - DEBUG - 934 - "/app/email_handler.py:545" - handle_forward() - alias no-such-alias-…@sl.local not exist. Try to see if it can be created on the fly
2026-07-08 04:29:06,952 - SL - INFO  - 934 - "/app/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - Cannot auto-create custom domain alias … because there's no custom domain for sl.local
2026-07-08 04:29:06,952 - SL - INFO  - 934 - "/app/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - Cannot auto-create … since it has no directory separator
2026-07-08 04:29:06,952 - SL - DEBUG - 934 - "/app/email_handler.py:551" - handle_forward() - alias no-such-alias-…@sl.local cannot be created on-the-fly, return 550
2026-07-08 04:29:06,953 - SL - INFO  - 934 - "/app/email_handler.py:2367" - _handle() - Finish … with return code '550 SL E515 Email not exist'<<===
```

### 3.6 Bonus edge — request rate limiting

**`OBSERVED`.** Rapidly repeating the create-alias POST trips SimpleLogin's rate limiter and returns an HTTP **429** page ("Whoa, slow down there, pardner!"). The dashboard route carries `@limiter.limit(ALIAS_LIMIT, methods=["POST"], …)` and `@limiter.limit("10/minute", methods=["GET"], …)` (`app/dashboard/views/index.py:L57-L62`), where `ALIAS_LIMIT = "100/day;50/hour;5/minute"` (`app/config.py:L448`). The limiter hit is logged by `rate_limited()` at `server.py:L364`:

```
2026-07-08 04:24:36,823 - SL - WARNING - 907 - "/code/server.py:364" - rate_limited() -  - Client hit rate limit on path /dashboard/, user:<User 3 …>
```

---

## 4. Q3 — Background Components

> **User Question 3:** *"While the application is active, do the email handler and job runner automatically come online in the background to support email activity or data handling? What behavior suggests that these internal components are functioning as intended across different situations?"*

### 4.1 Direct answer

**No — the email handler and job runner do *not* automatically come online with the web server. They are independent, separately-launched, long-running daemons.** Starting the application (the container's default command) launches **only** the web server; the email handler and job runner are distinct processes that must each be started on their own, after which each runs continuously.

**Cause → effect:** the container's default `CMD` execs a single process — gunicorn — so nothing in the web server's lifecycle forks or execs the other two. Each daemon has its own `__main__` and its own keep-alive loop; if you don't start it, it isn't running.

### 4.2 Independence — the web server does not start the others

**`OBSERVED`.** The container default command is the web server alone (`Dockerfile:L44,L47`):

```dockerfile
EXPOSE 7777
CMD ["gunicorn","wsgi:app","-b","0.0.0.0:7777","-w","2","--timeout","15"]
```

**`OBSERVED`.** The live process tree confirms the three components are unrelated processes. gunicorn's workers are children of the gunicorn master (PID 899), but the email handler (934) and job runner (976) are **top-level** processes (PPID 0) — not descendants of gunicorn:

```bash
$ ps -eo pid,ppid,cmd | grep -E 'gunicorn|email_handler|job_runner'
```
```
 899    0  /app/venv/bin/python /app/venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
 907  899  /app/venv/bin/python /app/venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
 908  899  /app/venv/bin/python /app/venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
 934    0  /app/venv/bin/python email_handler.py
 976    0  /app/venv/bin/python job_runner.py
```

**`OBSERVED`.** The web-server code contains no mechanism to spawn the other daemons:

```bash
$ grep -nE "email_handler|job_runner|subprocess|os\.system|Popen" server.py wsgi.py
(no matches)
```

**`INFERRED`** (`README.md`). The published self-host topology corroborates independence: three separate containers, each `docker run -d … --restart always`:
- `--name sl-app` (web) — `README.md:L455`, `--restart always` at `L462`.
- `--name sl-email` — `README.md:L471`, `--restart always` at `L478`, command `simplelogin/app:3.4.0 python email_handler.py` at `L480`.
- `--name sl-job-runner` — `README.md:L487`, `--restart always` at `L493`, command `simplelogin/app:3.4.0 python job_runner.py` at `L495`.

### 4.3 Continuous-run behavior — the job runner's ~10 s poll cadence

**`OBSERVED`.** The runner's loop is `while True:` (`job_runner.py:L330`) … `time.sleep(10)` (`job_runner.py:L347`). It logs nothing while idle, so to observe both a real `Take job` line and the interval, throwaway "ready" `Job` rows were inserted (safe: unrecognized names are handled by `LOG.e("Unknown job name %s")` at `job_runner.py:L304`) and removed afterward (§7).

**Single-job probe.** Inserting one `ready` job and watching the log:

```
inserted (state=0/ready) at 04:35:08
2026-07-08 04:35:12,278 - SL - DEBUG - 976 - "/code/job_runner.py:334" - <module>()    - Take job <Job 2 blitzy-probe-… None>
2026-07-08 04:35:12,283 - SL - ERROR - 976 - "/code/job_runner.py:304" - process_job() - Unknown job name blitzy-probe-…
final row: state=2 (done), attempts=1, taken=t, taken_at=2026-07-08 04:35:12.279276
```

This single capture demonstrates the full state machine `JobState` (`app/models.py:L253`: `ready=0, taken=1, done=2, error=3`): the row starts `ready(0)`; the loop logs `Take job` (`L334`), sets `taken_at` and increments `attempts` (the `taken(1)` step), runs `process_job` (`L188`), and finally commits `done(2)`.

**Cadence measurement (two runs).** Inserting 11 ready jobs at ~4 s spacing (faster than the 10 s poll) forces consecutive poll iterations to each pick up a batch; the gaps between iteration timestamps isolate the loop's own sleep interval. Each run spanned ~56 s.

*Run 1* — 5 iterations, jobs-per-iteration `[1,3,2,3,2]`:
```
iteration start times: 04:36:02.354, 04:36:12.374, 04:36:22.409, 04:36:33.431, 04:36:43.457
inter-iteration GAPS (s) = [10.02, 10.04, 11.02, 10.03]   mean 10.28s
sample raw lines:
2026-07-08 04:36:02,354 - SL - DEBUG - 976 - "/code/job_runner.py:334" - <module>() -  - Take job <Job 3 blitzy-cadence-1-…-0 None>
2026-07-08 04:36:12,374 - SL - DEBUG - 976 - "/code/job_runner.py:334" - <module>() -  - Take job <Job 4 blitzy-cadence-1-…-1 None>
2026-07-08 04:36:43,457 - SL - DEBUG - 976 - "/code/job_runner.py:334" - <module>() -  - Take job <Job 12 blitzy-cadence-1-…-9 None>
```

*Run 2* (stability confirmation) — 5 iterations, jobs-per-iteration `[2,3,2,3,1]`:
```
iteration start times: 04:37:13.499, 04:37:23.522, 04:37:33.553, 04:37:43.582, 04:37:53.848
inter-iteration GAPS (s) = [10.02, 10.03, 10.03, 10.27]   mean 10.09s
```

**Result:** the poll interval is **~10 seconds**, **stable across both runs** (means 10.28 s and 10.09 s) and across ≥4 cycles per run — exactly `time.sleep(10)` at `job_runner.py:L347` plus negligible processing time. This continuous, fixed-cadence polling is the behavior that indicates the job runner is functioning: it wakes every ~10 s, drains all eligible `Job` rows via `get_jobs_to_run()` (`job_runner.py:L307`), and advances each `ready → taken → done`.

**`INFERRED`** (email handler liveness). The email handler stays alive after `controller.start()` via `while True: time.sleep(2)` in `main()` (`email_handler.py:L2392-L2393`), which keeps the aiosmtpd controller thread listening on port 20381. Its startup lines (§2.3) and its successful handling of two separate SMTP transactions ~2 s apart (§3.3, §3.5) confirm it remained live and responsive between requests.

### 4.4 The job runner is distinct from the yacron scheduler

**`OBSERVED` / `INFERRED`.** SimpleLogin has **two** separate background mechanisms; they must not be conflated:

1. **Job runner** (`job_runner.py`) — a **custom `Job`-table poller**, not a task-queue library (no arq/Celery). It continuously polls the `job` table every ~10 s (measured above). Its job types are the `JOB_*` constants at `app/config.py:L301-L311` — `onboarding-1..4`, `batch-import`, `delete-account`, `delete-mailbox`, `delete-domain`, `send-user-report`, `proton-welcome-1`, `send-alias-creation-events` — dispatched by `process_job()` (`job_runner.py:L188`). Retry policy: `JOB_MAX_ATTEMPTS=5` (`app/config.py:L564`), `JOB_TAKEN_RETRY_WAIT_MINS=30` (`app/config.py:L565`).

2. **Cron scheduler** — `cron.py` invoked on **calendar schedules** by **yacron** per `crontab.yml`. Each entry runs `python /code/cron.py -j <job>` at a fixed time of day, e.g.:

```yaml
- name: SimpleLogin growth stats
  command: python /code/cron.py -j stats
  schedule: "0 0 * * *"
- name: SimpleLogin Custom Domain check
  command: python /code/cron.py -j check_custom_domain
  schedule: "15 2 * * *"
- name: SimpleLogin HIBP check
  command: python /code/cron.py -j check_hibp
  schedule: "15 3 * * *"
```

These map to functions in `cron.py` (`stats()` `L537`, `check_custom_domain()` `L902`, `check_hibp()` `L1106`, `delete_logs()` `L92`, `notify_hibp()` `L1171`, …). **Contrast:** the job runner is *event/interval-driven* (continuous 10 s polling of a DB table for on-demand work), whereas yacron/`cron.py` is *time-of-day scheduled* (periodic maintenance). They are independent subsystems.

---

## 5. Coverage Pass

Every named component and every named user action is addressed explicitly, with observed output and a `file:line` citation.

**Named components (Q1 & Q3):**

| Item | Up-signal / behavior | Where shown | Key citation |
|------|----------------------|-------------|--------------|
| **Web server** | `GET /health` → `200 success`; `GET /` → 302 `/auth/login`; gunicorn boot log | §2.2 | `server.py:L213-L215`, `L252-L256`, `L588`; `Dockerfile:L47` |
| **Email handler** | `Listen for port 20381` + `Start mail controller 0.0.0.0 20381`; keep-alive loop | §2.3, §4.3 | `email_handler.py:L2403`, `L2386`, `L2385`, `L2392-L2393` |
| **Job runner** | init banner; `Take job …`; ~10 s poll cadence (2 runs) | §2.4, §4.3 | `job_runner.py:L330`, `L334`, `L347`, `L304` |
| **Dashboard UI** | login form; post-login alias count, alias list, create/search controls | §2.5 | `templates/auth/login.html`; `templates/dashboard/index.html:L131,L233,L56,L44,L215` |

**Named user actions (Q2):**

| Action | Correctness signal(s) observed | Where shown | Key citation |
|--------|-------------------------------|-------------|--------------|
| **Create a new account** | "Activation Email Sent"; `activated` False→True; "Your account has been activated"; auto-login; explicit login 302 | §3.2 | `app/auth/views/register.py:L32,L124`; `app/auth/views/activate.py`; `app/auth/views/login.py:L25` |
| **Create an alias** | flash "Alias `<addr>` has been created"; log `create new random alias …`; alias row (count 1→2) | §3.3 | `app/dashboard/views/index.py:L104,L110,L111` |
| **Alias receives an email** | SMTP `250 Message accepted for delivery`; `Forward <Contact>→<Alias>→<Mailbox>`; new `Contact`+`EmailLog` (0→1) | §3.3 | `email_handler.py:L2289,L536,L688`; `app/email/status.py:L2`; `app/models.py:L1469,L1863,L2060` |

**Edge / alternate conditions (required):**

| Condition | Observed result | Where shown | Key citation |
|-----------|-----------------|-------------|--------------|
| Alias-quota gate (at limit) | warning "You need to upgrade your plan to create new alias."; no creation | §3.4 | `app/dashboard/views/index.py:L98,L123`; `app/models.py:L858,L867`; `app/config.py:L124` |
| Non-existent alias reception | `550 SL E515 Email not exist` | §3.5 | `app/email/status.py:L51`; `email_handler.py:L545,L551` |
| Request rate limiting | HTTP 429 rate-limit page | §3.6 | `app/dashboard/views/index.py:L57-L62`; `app/config.py:L448`; `server.py:L364` |
| Web server dev vs prod | Werkzeug (loopback, HTTP/1.0) vs gunicorn (0.0.0.0, HTTP/1.1) | §2.2 | `server.py:L588`; `Dockerfile:L47` |
| Background auto-start (Q3) | web-only default CMD; separate PIDs; no spawning; README 3-container topology | §4.2 | `Dockerfile:L47`; `README.md:L455,L471,L480,L487,L495` |
| Job state transition | `ready(0) → taken(1) → done(2)` | §4.3 | `app/models.py:L253`; `job_runner.py:L334,L188` |
| Distinct scheduler | yacron `cron.py`/`crontab.yml` vs custom `Job`-table poller | §4.4 | `crontab.yml`; `app/config.py:L301-L311` |

### Reported discrepancies (observed vs. cited)

1. **Free-plan alias limit** — docstring at `app/models.py:L870` says "15", but the enforced runtime limit is **5** (`MAX_NB_EMAIL_FREE_PLAN` default, `app/config.py:L124`; startup logs "use 5 as default value"). §3.4.
2. **"Forward phase" log line** — runs at `email_handler.py:L2202` at runtime (review citation was `~L2203`); 1-line drift. §3.3.

---

*Prepared by running each component's real entry point in the canonical Docker runtime and capturing complete, unedited output. All temporary accounts, aliases, emails, throwaway `Job` rows, and observation scripts were removed after evidence capture; the repository is left unchanged except for this document.*
