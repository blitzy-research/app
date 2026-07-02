# SimpleLogin — First-Operator Runtime Verification Guide (branch app_2cd6ee777f8c)

This guide answers three runtime-observability questions for a first operator bringing up **SimpleLogin** — an open-source, Flask-based email alias/forwarding service. It is a **verification guide, not a code change**: every source reference below is read-only and cited by `file:line`.

The answers are written from a **real local run** of the system on source branch `app_2cd6ee777f8c` (HEAD `2cd6ee77`). Following the governing discipline, every behavioral claim is immediately followed by the specific captured evidence (in a fenced block) and an exact `file:line` citation — **one claim, one piece of evidence** — and local-environment nuances are reported honestly rather than idealized.

---

## How the system was run

The system is a multi-process Flask application (app-factory pattern). The three long-running processes were started per the documented local run mechanism and observed together.

- **Repository:** SimpleLogin backend + webapp. Branch `app_2cd6ee777f8c`, HEAD `2cd6ee77 chore: emit some missing contact audit logs (#2269)`.
- **Run mechanism** (from `CONTRIBUTING.md` §"Run the code locally" [CONTRIBUTING.md:L77]): `alembic upgrade head` → `flask dummy-data` → `python server.py` (webapp `:7777`) + `python email_handler.py` (SMTP `:20381`) + `python job_runner.py` (poller).
- Working `.env` copied from `example.env` with `DB_URI=postgresql://myuser:mypassword@localhost:15432/simplelogin`. Relevant defaults inherited from `example.env`: `URL=http://localhost:7777` [example.env:L6], `NOT_SEND_EMAIL=true` [example.env:L19], `DISABLE_ONBOARDING=true` [example.env:L150], `EMAIL_DOMAIN=sl.local` [example.env:L22].

**Runtime versions (captured).** The stack ran on **Python 3.10.18** (project constraint `python = "^3.10"` [pyproject.toml:L61]), **PostgreSQL 15.13** on `localhost:15432` (the development baseline is Postgres 13+; the local server was 15.13), role `myuser`, database `simplelogin`, and **Redis 7.0.15** on port `6379`. Produced by `python --version`, a PostgreSQL `SELECT version()`, and a Redis `INFO server` query:

```console
$ python --version
Python 3.10.18
$ PGPASSWORD=mypassword psql -h localhost -p 15432 -U myuser -d simplelogin -c 'SELECT version();'
                                                           version
-----------------------------------------------------------------------------------------------------------------------------
 PostgreSQL 15.13 (Debian 15.13-0+deb12u1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 12.2.0-14+deb12u1) 12.2.0, 64-bit
(1 row)
$ redis-cli -p 6379 INFO server | grep -E 'redis_version|tcp_port'
redis_version:7.0.15
tcp_port:6379
```

Other Poetry-locked dependencies relevant to the run: Flask `^1.1.2` [pyproject.toml:L62], SQLAlchemy `1.3.24` [pyproject.toml:L116], `aiosmtpd = "^1.2"` [pyproject.toml:L87] (resolved to `1.4.2`, matching the SMTP banner in Q1(b)), redis `^4.5.3` [pyproject.toml:L117], and alembic.

> **Shorthand:** in the fenced blocks below, `psql ...` abbreviates the full connection command shown above — `PGPASSWORD=mypassword psql -h localhost -p 15432 -U myuser -d simplelogin`. It marks the connection flags only; every query string and result row is shown verbatim.

**Three long-running processes (captured by PID and port).** `python server.py` was launched as **PID 2716** — because `app.run(debug=True, port=7777)` [server.py:L588] enables the Werkzeug debug reloader, that launcher owns the `:7777` listening socket while its reloader **worker PID 2747** actually serves requests (so every webapp log line below carries PID `2747`). The email handler is **PID 2723** (owns `:20381`) and the job runner is **PID 2730**. From `ps` and a `/proc/net/tcp` socket-inode → PID match:

```console
$ ps -eo pid,ppid,cmd | grep -E 'server\.py|email_handler\.py|job_runner\.py' | grep -v grep
   2716    2710 /app/.venv/bin/python server.py
   2723    2717 /app/.venv/bin/python email_handler.py
   2730    2724 /app/.venv/bin/python job_runner.py
   2747    2716 /app/.venv/bin/python /app/server.py
$ # map listening ports 7777 / 20381 to owning PIDs
port 7777  LISTEN -> PID 2716   (socket inode 219986040)
port 20381 LISTEN -> PID 2723   (socket inode 219985005)
```

**Log line format** — every quoted `SL` log line below has this shape (cited once so the reader can parse them all):

```text
{asctime} - SL - {levelname} - {PID} - "{pathname}:{lineno}" - {funcName}() - {message_id} - {message}
```

There is a single `"SL"` logger, configured in `app/log.py` with `LOG = _get_logger("SL")` [app/log.py:L79] and the shortcuts `LOG.d` (debug), `LOG.i` (info), `LOG.w` (warning), `LOG.e` (exception).

**Migration result:** `alembic upgrade head` left the schema at head revision `32f25cbf12f6` with **77 tables**, including `activation_code` and `sync_event`. Produced by `alembic current` and two `psql` queries:

```console
$ alembic current
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
32f25cbf12f6 (head)
$ psql ... -c "SELECT count(*) FROM information_schema.tables WHERE table_schema='public';"
 table_count
-------------
          77
(1 row)
$ psql ... -c "SELECT table_name FROM information_schema.tables WHERE table_name IN ('activation_code','sync_event');"
   table_name
-----------------
 activation_code
 sync_event
(2 rows)
```

**Seed result:** `flask dummy-data` (CLI entry point `@app.cli.command("dummy-data")` [server.py:L490]) created **two** demo users and **11 aliases**. The primary account is defined by `email="john@wick.com"` [app/fake_data.py:L45], `password="password"` [app/fake_data.py:L47], `activated=True` [app/fake_data.py:L48], with a random alias via `Alias.create_new_random(user)` [app/fake_data.py:L70]; a second user `winston@continental.com` is also seeded. The login walkthrough uses `john@wick.com` / `password`. Produced by two `psql` queries:

```console
$ psql ... -c 'SELECT id, email, activated, is_admin FROM users ORDER BY id;'
 id |          email          | activated | is_admin
----+-------------------------+-----------+----------
  1 | john@wick.com           | t         | t
  2 | winston@continental.com | t         | f
(2 rows)
$ psql ... -c 'SELECT count(*) AS alias_count FROM alias;'
 alias_count
-------------
          11
(1 row)
```

### Honest local-environment nuances (read this first)

These are reported up front because they materially change what is observable. Each is reinforced inline where it occurs.

1. **The default Flask "Running on http://…" banner does NOT appear**, because `app/log.py` disables the Werkzeug access logger: `log = logging.getLogger("werkzeug")` [app/log.py:L70] then `log.disabled = True` [app/log.py:L71]. Readiness is instead evidenced by the init banner, `/health`, and the port-binding logs.
2. **`NOT_SEND_EMAIL=true`** means outbound mail is **logged (subject/from/to only), not sent** [app/mail_sender.py:L130-L137], so the activation link is never printed — it was read directly from the `activation_code` table in PostgreSQL.
3. **`DISABLE_ONBOARDING=true`** means a normal registration enqueues **zero** onboarding jobs — the job runner idles. A near-term job was enqueued deliberately to observe a live dispatch.
4. **`RegisterEvent`/`LoginEvent` emit New Relic custom events** (a no-op locally), NOT PostgreSQL events — reported here because it corrects a common misconception.
5. **DMARC and SpamAssassin checks are disabled locally** (unset `DMARC_CHECK_ENABLED` / `SPAMASSASSIN_HOST`).

---

## Q1 — Readiness / Startup

> **Q1:** "After starting the system what should I notice in the logs or UI that shows the app is ready to handle user authentication and alias based email activity?"

### Q1(a) Ready for user authentication

**Claim: logging bootstraps at import — the first readiness signal.** Produced by `python -c "import app.log"`:

```text
>>> init logging <<<
```

Cite the unconditional module-level `print(">>> init logging <<<")` [app/log.py:L67].

**Claim: the usual Flask "Running on http://…" banner is intentionally absent — do not expect it.**

```bash
$ grep -c "Running on" webapp.log
0
```

Cite `log = logging.getLogger("werkzeug")` [app/log.py:L70] and `log.disabled = True` [app/log.py:L71].

**Claim: the webapp liveness probe returns success.** Command `curl -sS -i http://localhost:7777/health`:

```http
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 7
Set-Cookie: slapp=5298473a-d327-45f0-9718-28e0ffe715d3.XCsnCbC2huM4o783RZYfQOVCtf0; Expires=Wed, 08-Jul-2026 23:39:58 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Wed, 01 Jul 2026 23:39:58 GMT

success
```

Cite `@app.route("/health", methods=["GET"])` [server.py:L213], `def healthcheck():` [server.py:L214], and `return "success", 200` [server.py:L215]. Note the session cookie name `slapp` (the full signed value is shown verbatim), the literal body `success`, and `Content-Length: 7`. (`/health` is deliberately excluded from the per-request access log — `not request.path.startswith("/health")` [server.py:L281] — so it produces no `after_request` line.)

**Claim: the authentication UI is served — the login form is live.** Command `curl -sS -D - -o /tmp/login.html http://localhost:7777/auth/login` (headers to stdout, body saved), then a `grep` of the body for the form fields:

```console
$ curl -sS -D - -o /tmp/login.html http://localhost:7777/auth/login
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 235552
Set-Cookie: slapp=b087a668-fc3e-4bc7-bc16-3be278d767f4.HjM2ZnR3Yl4Bk86xII4xqyBtPns; Expires=Thu, 09-Jul-2026 06:05:52 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Thu, 02 Jul 2026 06:05:52 GMT
$ grep -oE 'name="(csrf_token|email|password)"|>Log in<' /tmp/login.html
name="csrf_token"
name="email"
name="password"
>Log in<
```

This confirms the `auth` blueprint is mounted and the login page renders with a CSRF-protected `email`/`password` form on port `7777`. Cite the route `@auth_bp.route("/login", methods=["GET", "POST"])` [app/auth/views/login.py:L21] → `def login():` [app/auth/views/login.py:L25], whose `return render_template(` [app/auth/views/login.py:L74] uses `"auth/login.html",` [app/auth/views/login.py:L75]; in that template the form is `<form method="post">` [templates/auth/login.html:L16] with `{{ form.csrf_token }}` [templates/auth/login.html:L17] and the email field at [templates/auth/login.html:L20]. Note on the body size: the dev server runs with `debug=True` (`app.run(debug=True, ...)` [server.py:L588]), which injects the Flask Debug Toolbar into every HTML page — the login body is therefore ~235 KB, and its exact `Content-Length` varies by a few hundred bytes per request because the toolbar embeds per-request timing/profiler values and a fresh CSRF token (repeated captures observed sizes such as `235257`, `235462`, and the `235552` shown above). The stable readiness signal is the reproducible `HTTP/1.0 200 OK` plus the CSRF-protected `email`/`password` form, not the precise byte count.

**Claim: unauthenticated access to the root is redirected to login — authentication gates the app.** Command `curl -sS -i http://localhost:7777/`:

```http
HTTP/1.0 302 FOUND
Content-Type: text/html; charset=utf-8
Content-Length: 229
Location: http://localhost:7777/auth/login
Set-Cookie: slapp=69bd55ff-57b0-45bd-8670-53de54160561.NdATa10YhavGD3H6V9TDhJ6QQzs; Expires=Wed, 08-Jul-2026 23:39:58 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Wed, 01 Jul 2026 23:39:58 GMT
```

Cite the root route `def set_index_page(app):` [server.py:L249] → `@app.route("/", methods=["GET", "POST"])` [server.py:L250] → `def index():` [server.py:L251]: when `if current_user.is_authenticated:` [server.py:L252] is false, the `else:` [server.py:L254] branch returns `redirect(url_for("auth.login"))` [server.py:L255] (an authenticated user would instead hit `return redirect(url_for("dashboard.index"))` [server.py:L253]).

### Q1(b) Ready for alias-based email activity

**Claim: the SMTP email handler binds and announces its port.** From the email handler's log at startup (PID `2723`):

```text
2026-07-01 23:38:01,250 - SL - INFO - 2723 - "/app/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-01 23:38:01,251 - SL - DEBUG - 2723 - "/app/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

Cite `LOG.i("Listen for port %s", args.port)` [email_handler.py:L2403] (default port `20381` at [email_handler.py:L2399]) and `LOG.d("Start mail controller %s %s", controller.hostname, controller.port)` [email_handler.py:L2386], where the controller is built with `Controller(MailHandler(), hostname="0.0.0.0", port=port)` [email_handler.py:L2383] inside `def main(port: int)` [email_handler.py:L2381].

**Claim: the SMTP server actually accepts connections (real handshake).** A raw socket connect to `127.0.0.1:20381` sending `EHLO probe.local` then `QUIT` (server lines shown; `220`/`250-…`/`221` are the server's, `EHLO probe.local` is the line we sent):

```text
220 05788680d14b Python SMTP 1.4.2
EHLO probe.local
250-05788680d14b
250-SIZE 33554432
250-8BITMIME
250-SMTPUTF8
250 HELP
221 Bye
```

This is the aiosmtpd `1.4.2` greeting (the hostname `05788680d14b` is the container's own hostname) plus the advertised ESMTP capabilities (`SIZE 33554432`, `8BITMIME`, `SMTPUTF8`, `HELP`), and the `221 Bye` closing response.

**Claim: the background job runner is alive and polling (it backs the async alias/email machinery).** The poll loop runs `while True:` [job_runner.py:L330] under `create_light_app().app_context()`, sleeping between passes with `time.sleep(10)` [job_runner.py:L347]. When there are no due jobs it is silent by design — so liveness is confirmed by the running process (PID `2730`) rather than a log line. `ps` for PID `2730`:

```console
$ ps -o pid,etime,cmd -p 2730
    PID     ELAPSED CMD
   2730       02:37 /app/.venv/bin/python job_runner.py
```

**Claim: aliases exist and use the configured domain.** The seeded aliases live on the `sl.local` domain (`EMAIL_DOMAIN=sl.local` [example.env:L22]). Queried directly from the `alias` table (`john@wick.com`, user_id 1):

```console
$ psql ... -c 'SELECT id, email, enabled FROM alias WHERE user_id=1 AND email LIKE '"'"'%@sl.local'"'"' ORDER BY id;'
 id |                   email                   | enabled
----+-------------------------------------------+---------
  1 | simplelogin-newsletter.hereon338@sl.local | t
  2 | limber_redeem561@sl.local                 | t
  4 | e0@sl.local                               | f
  5 | e1@sl.local                               | t
  6 | e2@sl.local                               | t
(5 rows)
```

---

## Q2 — New-User Journey (register → verify → login → dashboard)

> **Q2:** "Walking through the typical product experience as if you were a new user signing up for the first time. What happens when a user registers, verifies their address, and tries to log in? What visible behavior confirms that the system is handling every step correctly and forwarding the user into the dashboard as expected?"

The walkthrough used a synthetic test address `blitzy-qna-probe-1782949284@proton.me` (POSTed to `/auth/register`, created as user `id=3`; a separate never-activated address `blitzy-qna-probe-notact-1782949366@proton.me` was used for the not-activated login path in Q2(c)). All probe rows were removed at cleanup so the repository and database are left unchanged.

### Q2(a) Register

**Claim: registration renders the "waiting for activation" screen.** `POST /auth/register` → `HTTP 200`, response `<title>`:

```text
Activation Email Sent | SimpleLogin
```

Cite `render_template("auth/register_waiting_activation.html")` [app/auth/views/register.py:L104].

**Claim: the server logs user creation.**

```text
2026-07-01 23:41:24,817 - SL - DEBUG - 2747 - "/app/app/auth/views/register.py:85" - register() -  - create user blitzy-qna-probe-1782949284@proton.me
```

Cite `LOG.d("create user %s", email)` [app/auth/views/register.py:L85]; the row is created by `User.create(...)` [app/auth/views/register.py:L86]. (The log PID is `2747` — the Werkzeug reloader worker that serves requests, as explained in "How the system was run".)

**Claim: the activation email is logged, not sent (`NOT_SEND_EMAIL`), so no link is printed.**

```text
2026-07-01 23:41:25,142 - SL - DEBUG - 2747 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'Just one more step to join SimpleLogin', from '"noreply@sl.local" <noreply@sl.local>' to 'blitzy-qna-probe-1782949284@proton.me'
```

Cite the guard `if config.NOT_SEND_EMAIL:` [app/mail_sender.py:L130], which logs subject/from/to and then `return True` [app/mail_sender.py:L137]. The activation email subject literal is `Just one more step to join SimpleLogin`.

**Claim: the password policy is enforced (minimum length 8).** Registering with a 5-character password `short` → `HTTP 200` with the page error:

```text
Field must be between 8 and 100 characters long
```

And no user row is created (rejected before `User.create`) — DB count of `blitzy-qna-probe-short-%` users is `0`:

```console
$ psql ... -c "SELECT count(*) FROM users WHERE email LIKE 'blitzy-qna-probe-short-%';"
 count
-------
     0
(1 row)
```

Cite `validators.Length(min=8, max=100)` [app/auth/views/register.py:L27].

**Claim (honest): the success `RegisterEvent` produces NO local log line**, because it is New Relic telemetry (a no-op locally). Cite `RegisterEvent(RegisterEvent.ActionType.success).send()` [app/auth/views/register.py:L96], whose `send()` calls `newrelic.agent.record_custom_event(` [app/events/auth_event.py:L45]. The daily metric increment `DailyMetric.get_or_create_today_metric().nb_new_web_non_proton_user += 1` [app/auth/views/register.py:L97] was verified in the database — after this single successful web registration the counter for today (`2026-07-01`) is `1`:

```console
$ psql ... -c 'SELECT date, nb_new_web_non_proton_user FROM daily_metric;'
    date    | nb_new_web_non_proton_user
------------+----------------------------
 2026-07-01 |                          1
(1 row)
```

### Q2(b) Verify address

Because `NOT_SEND_EMAIL` suppresses the printed link, the activation code was **read directly from the `activation_code` table in PostgreSQL** — this is the method used to obtain the code.

**Claim: the new user starts not-yet-activated.** Pre-activation DB row for `id=3`:

```console
$ psql ... -c 'SELECT id, email, activated FROM users WHERE id=3;'
 id |                 email                 | activated
----+---------------------------------------+-----------
  3 | blitzy-qna-probe-1782949284@proton.me | f
(1 row)
```

**Claim: an activation code (30 random chars) exists for that user.** DB read of `activation_code.code` and its length for `user_id=3`:

```console
$ psql ... -tAc "SELECT code, length(code) FROM activation_code WHERE user_id=3;"
auusszqbwhjwqrdesyjhvokjmpgzmm|30
```

Cite `ActivationCode.create(user_id=user.id, code=random_string(30))` [app/auth/views/register.py:L120].

**Claim: the activation link format is deterministic.**

```text
http://localhost:7777/auth/activate?code=auusszqbwhjwqrdesyjhvokjmpgzmm
```

Cite `activation_link = f"{URL}/auth/activate?code={activation.code}"` [app/auth/views/register.py:L124].

**Claim: visiting the activation link activates the account and redirects to the dashboard.** `GET /auth/activate?code=auusszqbwhjwqrdesyjhvokjmpgzmm`:

```http
HTTP/1.0 302 FOUND
Location: http://localhost:7777/dashboard/
```

```text
2026-07-01 23:42:16,554 - SL - DEBUG - 2747 - "/app/app/auth/views/activate.py:66" - activate() -  - redirect user to dashboard
```

Cite `user.activated = True` [app/auth/views/activate.py:L49], `login_user(user)` [app/auth/views/activate.py:L50], `LOG.d("redirect user to dashboard")` [app/auth/views/activate.py:L66], and `return redirect(url_for("dashboard.index"))` [app/auth/views/activate.py:L67].

**Claim: the database confirms activation flipped.** The same `id=3` row, queried again after the `GET`, now shows `activated=t` (compare with the pre-activation `activated=f` row above — `False → True`):

```console
$ psql ... -c 'SELECT id, email, activated FROM users WHERE id=3;'
 id |                 email                 | activated
----+---------------------------------------+-----------
  3 | blitzy-qna-probe-1782949284@proton.me | t
(1 row)
```

**Claim: a welcome email is (logged as) sent on activation.**

```text
2026-07-01 23:42:16,553 - SL - DEBUG - 2747 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'Welcome to SimpleLogin', from '"noreply@sl.local" <noreply@sl.local>' to 'simplelogin-newsletter.odious999@sl.local'
```

Cite `email_utils.send_welcome_email(user)` [app/auth/views/activate.py:L58]. (The welcome email is addressed to the user's own first alias `simplelogin-newsletter.odious999@sl.local`, auto-created for the new account.)

**Claim: the success flash is visible on the dashboard.** The post-activation `GET /dashboard/` → `200`; the page contains:

```text
Your account has been activated
```

Cite `flash("Your account has been activated", "success")` [app/auth/views/activate.py:L56].

### Q2(c) Log in (success + two negative paths)

**Claim (SUCCESS): logging in with the seeded account redirects into the dashboard.** `POST /auth/login` (`john@wick.com` / `password`):

```http
HTTP/1.0 302 FOUND
Location: http://localhost:7777/dashboard/
```

```text
2026-07-01 23:42:46,427 - SL - DEBUG - 2747 - "/app/app/auth/views/login_utils.py:35" - after_login() -  - log user <User 1 John Wick john@wick.com> in
2026-07-01 23:42:46,428 - SL - DEBUG - 2747 - "/app/app/auth/views/login_utils.py:44" - after_login() -  - redirect user to dashboard
```

Cite the success branch `LoginEvent(LoginEvent.ActionType.success).send()` [app/auth/views/login.py:L71] → `return after_login(user, next_url)` [app/auth/views/login.py:L72]; then, inside `after_login()`, `LOG.d("log user %s in", user)` [app/auth/views/login_utils.py:L35], `login_user(user)` [app/auth/views/login_utils.py:L36], `session["sudo_time"] = int(time())` [app/auth/views/login_utils.py:L37] (a code fact, not a log line), `LOG.d("redirect user to dashboard")` [app/auth/views/login_utils.py:L44], and `return redirect(url_for("dashboard.index"))` [app/auth/views/login_utils.py:L45]. MFA routing exists but is not triggered for this account: the FIDO branch `if user.fido_enabled():` [app/auth/views/login_utils.py:L20] and the OTP branch `elif user.enable_otp:` [app/auth/views/login_utils.py:L28].

**Claim (NEGATIVE — wrong password): a clear error flash, no redirect.** `POST /auth/login` (`john@wick.com` / `WRONGpassword`) → `HTTP 200`, the page shows:

```text
Email or password incorrect
```

Cite the failure branch `if not user or not user.check_password(form.password.data):` [app/auth/views/login.py:L45] → `flash("Email or password incorrect", "error")` [app/auth/views/login.py:L49]. This branch emits `LoginEvent(...).failed` → New Relic, so there is no local log line.

**Claim (NEGATIVE — not activated): the app prompts to check the inbox / resend activation.** A fresh unverified user logging in → `HTTP 200` with the exact flash:

```text
Please check your inbox for the activation email. You can also have this email re-sent
```

A resend-activation form is present. Cite the not-activated branch [app/auth/views/login.py:L66-L69], which flashes the message and emits `LoginEvent(LoginEvent.ActionType.not_activated).send()` [app/auth/views/login.py:L69] → New Relic, so again there is no local log line.

### Q2(d) Visible behavior confirming each step

Each step's correctness is confirmed by a **paired signal — a visible HTTP/UI outcome AND a server log line**:

- **Register** → `HTTP 200` "Activation Email Sent" + `create user ...` [app/auth/views/register.py:L85].
- **Verify** → `HTTP 302` to `/dashboard/` + `redirect user to dashboard` [app/auth/views/activate.py:L66] + DB `activated=True` [app/auth/views/activate.py:L49] + the success flash [app/auth/views/activate.py:L56].
- **Log in** → `HTTP 302` to `/dashboard/` + `log user … in` [app/auth/views/login_utils.py:L35].

Additionally, the webapp emits a per-request access log (its own line, since the Werkzeug logger is disabled). The activation `GET` (302) and the login `POST` (200) access lines, captured verbatim:

```text
2026-07-01 23:42:16,554 - SL - DEBUG - 2747 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/activate ImmutableMultiDict([('code', 'auusszqbwhjwqrdesyjhvokjmpgzmm')]) 302, takes 0.05317044258117676
2026-07-01 23:42:47,595 - SL - DEBUG - 2747 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 200, takes 0.28017592430114746
```

Each line reports the client IP, method, path, request args (the activation `code` is echoed; the login form args are an empty `ImmutableMultiDict([])` because credentials are not logged), the HTTP status, and the request duration (`takes 0.05317044258117676` s for the activate redirect, `takes 0.28017592430114746` s for the login POST). Cite the request-logging `LOG.d(...)` in `@app.after_request def after_request(res)` [server.py:L284].

### Q2(e) Forwarded into the dashboard

**Claim: the authenticated user lands on the Alias dashboard.** Logging in as `john@wick.com` then `GET /dashboard/` → `HTTP 200` with `<title>` `Alias | SimpleLogin`. Captured by parsing the authenticated response:

```console
$ python walk_q2_dashboard.py   # logs in as john@wick.com, then GET /dashboard/
dashboard status: 200
title: Alias | SimpleLogin
```

Cite the route `@dashboard_bp.route("/", methods=["GET", "POST"])` [app/dashboard/views/index.py:L55], `def index()` [app/dashboard/views/index.py:L67], and the `return render_template(` [app/dashboard/views/index.py:L215] with `"dashboard/index.html",` [app/dashboard/views/index.py:L216] (render call at L215, template-name literal at L216). The observed title is composed from the page's own `{% block title %}Alias{% endblock %}` [templates/dashboard/index.html:L31] inserted into the base layout's `<title>{% block title %}{% endblock %} | SimpleLogin</title>` [templates/base.html:L30-L33] — hence `Alias | SimpleLogin`.

**Claim: the user's aliases render on the dashboard.** The body contains John's first alias `simplelogin-newsletter.hereon338@sl.local`, and the string `simplelogin-newsletter` appears `11` times in the rendered HTML. Captured from the same authenticated response:

```console
$ python walk_q2_dashboard.py   # (continued)
first alias in body: simplelogin-newsletter.hereon338@sl.local
count of 'simplelogin-newsletter' occurrences in body: 11
```

The `simplelogin-newsletter.*` alias is a `User.create` side effect: every new account gets a default pre-verified mailbox — cite `Mailbox.create(user_id=user.id, email=user.email, verified=True)` [app/models.py:L611] — and a first alias built from the `simplelogin-newsletter` prefix — cite `Alias.create_new(user, prefix="simplelogin-newsletter", ...)` [app/models.py:L636]. (John's suffix is `hereon338`; the probe user's equivalent first alias was `simplelogin-newsletter.odious999@sl.local`, seen in the welcome-email log in Q2(b).)

---

## Q3 — Behind the Scenes

> **Q3:** "What the application does behind the scenes during that flow. Are there any indicators that background jobs or internal services are doing their part to support email forwarding or identity verification? What should I expect to observe at runtime that tells me these moving pieces are active and talking to each other properly?"

### Q3(a) Are background jobs doing their part?

**Claim (honest idle behavior): with `DISABLE_ONBOARDING=true`, a fresh registration enqueues ZERO onboarding jobs and the runner idles.** From `User.create` during dummy-data / registration:

```text
2026-07-01 23:41:25,110 - SL - DEBUG - 2747 - "/app/app/models.py:647" - create() -  - Disable onboarding emails
```

Cite `if config.DISABLE_ONBOARDING:` → `LOG.d("Disable onboarding emails")` [app/models.py:L647], immediately followed by an early `return user` [app/models.py:L648]. The `+1`/`+2`/`+3`-day onboarding jobs — `JOB_ONBOARDING_1` with `run_at=arrow.now().shift(days=1)` [app/models.py:L654], `JOB_ONBOARDING_2` at `+2 days` [app/models.py:L659], and `JOB_ONBOARDING_4` at `+3 days` [app/models.py:L664] — are enqueued **only** when onboarding is enabled, and even then would not fire immediately (see the selection window below). This is why the runner is silent right after signup; this is the real behavior, not an idealized one.

**Claim: the selection window only picks jobs due within ~10 minutes; retry/attempt limits gate re-runs.** A small script computing the exact bounds `get_jobs_to_run()` applies produced these live values:

```console
$ python job_window.py
now                         = 2026-07-01 23:48:42 +00:00
run_at_earliest (now+10min) = 2026-07-01 23:58:42 +00:00
taken_at_earliest (now-30m) = 2026-07-01 23:18:42 +00:00
JOB_TAKEN_RETRY_WAIT_MINS   = 30
JOB_MAX_ATTEMPTS            = 5
```

Cite `get_jobs_to_run()` [job_runner.py:L307], which selects jobs where `state == ready`, **or** `taken` older than `JOB_TAKEN_RETRY_WAIT_MINS` with `attempts < JOB_MAX_ATTEMPTS`, and `run_at` is NULL or `≤ now + 10 min` (`return query.all()` [job_runner.py:L326]); the limits are `JOB_TAKEN_RETRY_WAIT_MINS = 30` [app/config.py:L565] and `JOB_MAX_ATTEMPTS = 5` [app/config.py:L564]. The `run_at_earliest` bound (`now + 10 min`) is exactly why the `+1/+2/+3`-day onboarding jobs would never be picked immediately after signup.

**Claim: a live dispatch was observed by enqueuing a near-term job.** Enqueuing `Job.create(name="onboarding-1", payload={"user_id": 3}, run_at=arrow.now())` — its initial persisted state is `state=0` (ready), `taken=False`, `attempts=0`:

```console
$ python enqueue_job.py
ENQUEUED: <Job 1 onboarding-1 {'user_id': 3}>
initial: id=1 state=0 taken=False attempts=0 run_at=2026-07-01T23:45:11.038276+00:00
```

The separate job-runner process (PID `2730`) picked it up within its 10-second poll and processed it, producing these verbatim log lines:

```text
2026-07-01 23:45:12,022 - SL - DEBUG - 2730 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 1 onboarding-1 {'user_id': 3}>
2026-07-01 23:45:12,031 - SL - DEBUG - 2730 - "/app/job_runner.py:196" - process_job() -  - send onboarding send-from-alias email to user <User 3 blitzy-qna-probe-1782949284@proton.me blitzy-qna-probe-1782949284@proton.me>
2026-07-01 23:45:12,051 - SL - DEBUG - 2730 - "/app/app/email_utils.py:303" - send_email() -  - send email to simplelogin-newsletter.odious999@sl.local, subject 'SimpleLogin Tip: Send emails from your alias'
2026-07-01 23:45:12,052 - SL - DEBUG - 2730 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'SimpleLogin Tip: Send emails from your alias', from '"noreply@sl.local" <noreply@sl.local>' to 'simplelogin-newsletter.odious999@sl.local'
```

**Claim: the job ran to completion.** Querying the `job` table afterward shows the terminal state — `state=2` (done), `taken=t`, `attempts=1` — completing the full `ready(0) → taken(1) → done(2)` lifecycle (compare `state=0` at enqueue above):

```console
$ psql ... -c 'SELECT id, name, state, taken, attempts FROM job WHERE id=1;'
 id |     name     | state | taken | attempts
----+--------------+-------+-------+----------
  1 | onboarding-1 |     2 | t     |        1
(1 row)
```

Cite `LOG.d("Take job %s", job)` [job_runner.py:L334] and the name-based dispatch in `process_job()` [job_runner.py:L188]: `onboarding-1` → `onboarding_send_from_alias` [job_runner.py:L196]; `onboarding-2` → mailbox onboarding [job_runner.py:L198,L205]; `onboarding-4` → PGP onboarding [job_runner.py:L207]; plus batch-import, delete-account/mailbox/domain, send-user-report, send-proton-welcome-1, and send-alias-creation-events; an unknown name falls to `else:` → `LOG.e("Unknown job name %s", job.name)` [job_runner.py:L303-L304]. The `JOB_ONBOARDING_1` literal is `"onboarding-1"` [app/config.py:L301].

### Q3(b) Internal services supporting email forwarding

A real message was sent over SMTP to `127.0.0.1:20381` (from `outsider@example.com` to the enabled alias `simplelogin-newsletter.hereon338@sl.local`); the email handler (PID `2723`) processed it end-to-end. **Every line below is correlated by the same `message_id` `44a9cc52-88ea-4e61-a661-f40e24ecf015`**, which shows the pipeline is one coherent flow.

**Claim: a correlation `message_id` is set for the incoming mail.**

```text
2026-07-01 23:45:56,376 - SL - DEBUG - 2723 - "/app/app/log.py:24" - set_message_id() -  - set message_id 44a9cc52-88ea-4e61-a661-f40e24ecf015
```

Cite `set_message_id(...)` (`LOG.d("set message_id %s", message_id)`) [app/log.py:L24].

**Claim: the handler router receives the mail.**

```text
2026-07-01 23:45:56,492 - SL - DEBUG - 2723 - "/app/email_handler.py:1980" - handle() - 44a9cc52-88ea-4e61-a661-f40e24ecf015 - ==>> Handle mail_from:outsider@example.com, rcpt_tos:['simplelogin-newsletter.hereon338@sl.local'], header_from:outsider@example.com, header_to:simplelogin-newsletter.hereon338@sl.local, cc:None, reply-to:None, message_id:<blitzy-qna-fwd-001@example.com>, client_ip:None, headers:[('From', 'outsider@example.com'), ('To', 'simplelogin-newsletter.hereon338@sl.local'), ('Subject', 'Blitzy QnA enabled forward'), ('Message-Id', '<blitzy-qna-fwd-001@example.com>'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:['SIZE=298'], rcpt_options:[]
```

Cite the main router `def handle(envelope: Envelope, msg: Message) -> str` [email_handler.py:L1945] and the `==>> Handle` log [email_handler.py:L1980].

**Claim: the forward phase is selected (inbound → alias → owner).**

```text
2026-07-01 23:45:56,497 - SL - DEBUG - 2723 - "/app/email_handler.py:2202" - handle() - 44a9cc52-88ea-4e61-a661-f40e24ecf015 - Forward phase outsider@example.com(outsider@example.com) -> simplelogin-newsletter.hereon338@sl.local
```

Cite [email_handler.py:L2202].

**Claim: the sender contact is created/looked up.**

```text
2026-07-01 23:45:56,512 - SL - DEBUG - 2723 - "/app/email_handler.py:580" - handle_forward() - 44a9cc52-88ea-4e61-a661-f40e24ecf015 - Create or get contact for from_header:outsider@example.com
```

Cite `def handle_forward(envelope, msg, rcpt_to)` [email_handler.py:L536] and the contact log [email_handler.py:L580].

**Claim: the DMARC policy check runs but is disabled locally (reported honestly).**

```text
2026-07-01 23:45:56,536 - SL - INFO - 2723 - "/app/app/handler/dmarc.py:33" - apply_dmarc_policy_for_forward_phase() - 44a9cc52-88ea-4e61-a661-f40e24ecf015 - DMARC check disabled
```

Cite `def apply_dmarc_policy_for_forward_phase(...)` [app/handler/dmarc.py:L28] and the disabled guard `if not DMARC_CHECK_ENABLED or not spam_result:` [app/handler/dmarc.py:L32] → `LOG.i("DMARC check disabled")` [app/handler/dmarc.py:L33]. `DMARC_CHECK_ENABLED` is unset, and `SPAMASSASSIN_HOST` is unset so the spam scan is skipped (`MAX_SPAM_SCORE` default `5.5`).

**Claim: routing resolves Contact → Alias → Mailbox (the alias owner).**

```text
2026-07-01 23:45:56,544 - SL - DEBUG - 2723 - "/app/email_handler.py:688" - forward_email_to_mailbox() - 44a9cc52-88ea-4e61-a661-f40e24ecf015 - Forward <Contact 2 outsider@example.com 1> -> <Alias 1 simplelogin-newsletter.hereon338@sl.local> -> <Mailbox 1 john@wick.com>
```

Cite [email_handler.py:L688].

**Claim: an `EmailLog` row is written (audit of the forward).**

```text
2026-07-01 23:45:56,548 - SL - DEBUG - 2723 - "/app/email_handler.py:740" - forward_email_to_mailbox() - 44a9cc52-88ea-4e61-a661-f40e24ecf015 - Create <EmailLog 2> for <Contact 2 outsider@example.com 1>, <User 1 John Wick john@wick.com>, <Mailbox 1 john@wick.com>
```

Cite [email_handler.py:L740].

**Claim: the `From` header is rewritten to a reverse-alias (so replies route back through SimpleLogin).**

```text
2026-07-01 23:45:56,553 - SL - DEBUG - 2723 - "/app/email_handler.py:867" - forward_email_to_mailbox() - 44a9cc52-88ea-4e61-a661-f40e24ecf015 - From header, new:"outsider at example.com" <outsider_at_example_com_eazdko@sl.local>, old:outsider@example.com
```

Cite [email_handler.py:L867]. The reverse-alias local-part (`outsider_at_example_com_eazdko`) is randomized per contact so replies route back through SimpleLogin.

**Claim: the message is forwarded to the owner mailbox.**

```text
2026-07-01 23:45:56,553 - SL - DEBUG - 2723 - "/app/email_handler.py:893" - forward_email_to_mailbox() - 44a9cc52-88ea-4e61-a661-f40e24ecf015 - Forward mail from outsider@example.com to john@wick.com, mail_options:['SIZE=298'], rcpt_options:[]
2026-07-01 23:45:56,554 - SL - DEBUG - 2723 - "/app/app/mail_sender.py:131" - send() - 44a9cc52-88ea-4e61-a661-f40e24ecf015 - send email with subject 'Blitzy QnA enabled forward', from '"outsider at example.com" <outsider_at_example_com_eazdko@sl.local>' to 'simplelogin-newsletter.hereon338@sl.local'
```

Cite [email_handler.py:L893]; the final send is at [app/mail_sender.py:L131]. With `NOT_SEND_EMAIL` off, the real relay target would be `POSTFIX_SERVER` default `240.0.0.1` [app/config.py:L136] on `POSTFIX_PORT` default `25` [app/config.py:L149] — bypassed here by `NOT_SEND_EMAIL`.

**Claim: a disabled alias is NOT forwarded (negative path).** Sending to the disabled alias `e0@sl.local`:

```text
2026-07-01 23:46:19,012 - SL - DEBUG - 2723 - "/app/email_handler.py:597" - handle_forward() - 5f04c3a1-78b6-4c70-8dd6-fc7490828436 - <Alias 4 e0@sl.local> is disabled, do not forward
```

Cite [email_handler.py:L597]. (This is a separate inbound message, so it carries its own `message_id` `5f04c3a1-78b6-4c70-8dd6-fc7490828436`; the alias `e0@sl.local` is `enabled=f`, per the alias table in Q1(b).)

**Named email-security helpers in the forwarding path (by name):**

- **DMARC** — `app/handler/dmarc.py` (`apply_dmarc_policy_for_forward_phase` [app/handler/dmarc.py:L28]); **disabled locally**.
- **SpamAssassin** — `app/spamassassin_utils.py` (`class SpamAssassin` [app/spamassassin_utils.py:L17], `def is_spam(self, level=5)` [app/spamassassin_utils.py:L136]); **disabled locally** (no `SPAMASSASSIN_HOST`).
- **DKIM** — via the `dkimpy` library, declared `dkimpy = "^1.0.5"` [pyproject.toml:L92] and imported as `import dkim` [app/email_utils.py:L22]; used to DKIM-sign forwarded/relayed mail (SimpleLogin's `add_dkim_signature`).
- **SPF** — via the `pyspf` library, declared `pyspf = "^2.0.14"` [pyproject.toml:L100] and imported as `import spf` [app/email_utils.py:L24]; used in the SPF/DMARC handling path.
- **PGP** — via `python-gnupg`, declared `python-gnupg = "^0.4.6"` [pyproject.toml:L98] and imported as `import gnupg` [app/pgp_utils.py:L5]; the module reads `from app.config import GNUPGHOME, PGP_SENDER_PRIVATE_KEY` [app/pgp_utils.py:L10] and initializes `gpg = gnupg.GPG(gnupghome=GNUPGHOME)` [app/pgp_utils.py:L14] for mailbox-key encryption of forwarded mail.

### Q3(c) Internal services supporting identity verification

All three identity-verification services, by name:

**(1) Registration activation code.** Subject `Just one more step to join SimpleLogin`; the `ActivationCode.code` is 30 chars via `random_string(30)` [app/auth/views/register.py:L120], and `GET /auth/activate?code=auusszqbwhjwqrdesyjhvokjmpgzmm` sets `activated=True` [app/auth/views/activate.py:L49] (full evidence in Q2(b)).

**(2) Mailbox verification (`app/mailbox_utils.py`) — observed live.** An unverified mailbox (id=7) was created, an activation code generated, and the code verified:

```text
created unverified mailbox id=7 verified=False
MailboxActivation.code = 'lROYky7xLLzuEnOCPqwRvA'  (len 22)
2026-07-01 23:46:50,135 - SL - INFO - 3654 - "/app/app/mailbox_utils.py:212" - verify_mailbox_code() -  - User <User 1 John Wick john@wick.com> has verified mailbox 7
after verify_mailbox_code -> mailbox.verified = True
```

Cite `def generate_activation_code(...)` [app/mailbox_utils.py:L223-L240] (code via `secrets.token_urlsafe(16)`), `def verify_mailbox_code(user, mailbox_id, code)` [app/mailbox_utils.py:L166], the verified log `LOG.i(f"User {user} has verified mailbox {mailbox_id}")` [app/mailbox_utils.py:L212], and `mailbox.verified = True` [app/mailbox_utils.py:L213]. Also note `def create_mailbox(...)` [app/mailbox_utils.py:L46] and that `delete_mailbox` enqueues a `Job` via `Job.create(...)` [app/mailbox_utils.py:L145], tying mailbox management to the background job system.

**(3) Custom-domain DNS validation (`app/custom_domain_validation.py`) — observed live** (a temporary domain, deleted afterward). The expected DNS records the operator must publish to prove domain ownership + mail routing:

```text
ownership TXT record : sl-verification=mlumbahjhifhmhbydzhnayxxlwumbx
expected MX records  : [(10, 'email.hostname.')]
expected SPF record  : v=spf1 include:sl.local ~all
expected DKIM records: {'dkim._domainkey': 'dkim._domainkey.sl.local', 'dkim02._domainkey': 'dkim02._domainkey.sl.local', 'dkim03._domainkey': 'dkim03._domainkey.sl.local'}
```

Cite `class CustomDomainValidation` [app/custom_domain_validation.py:L24], `get_ownership_verification_record` [app/custom_domain_validation.py:L40], `get_expected_mx_records` [app/custom_domain_validation.py:L54], the SPF methods `get_expected_spf_domain` [app/custom_domain_validation.py:L67] / `get_expected_spf_record` [app/custom_domain_validation.py:L73], `get_dkim_records` [app/custom_domain_validation.py:L77], and `validate_domain_ownership` [app/custom_domain_validation.py:L134].

### Q3(d) Runtime observations proving the pieces are active and communicating

**Claim: the genuine inter-process channel is PostgreSQL NOTIFY/LISTEN on `simplelogin_sync_events`.** A listener issued `LISTEN simplelogin_sync_events`; the dispatcher wrote a `SyncEvent` and executed `NOTIFY` (on commit). The verbatim stdout of the two-connection probe (the `[listener]`/`[sender]` prefixes are the probe's own status prints; the actual payload passed to `send()` is shown verbatim on the last line as `content=b'blitzy-qna-test-event-payload'`):

```text
[listener] LISTEN simplelogin_sync_events
[sender] PostgresDispatcher().send(...) + commit done
[listener] Got NOTIFY: pid=3714 channel=simplelogin_sync_events payload=1
SyncEvent persisted by dispatcher: id=1 content=b'blitzy-qna-test-event-payload'
```

Cite the channel name `NOTIFICATION_CHANNEL = "simplelogin_sync_events"` [app/events/event_dispatcher.py:L14] and `PostgresDispatcher.send` [app/events/event_dispatcher.py:L24] — `SyncEvent.create(content=event, flush=True)` [app/events/event_dispatcher.py:L25] then `Session.execute(f"NOTIFY {NOTIFICATION_CHANNEL}, '{instance.id}';")` [app/events/event_dispatcher.py:L26]; on the consumer side `cursor.execute(f"LISTEN {NOTIFICATION_CHANNEL};")` [events/event_source.py:L47] and its `Got NOTIFY: pid=… channel=… payload=…` log [events/event_source.py:L56]. **Important honesty note:** the NOTIFY only delivers on transaction **COMMIT** — `flush` alone was insufficient; this was confirmed as an observed fact.

**Claim (honest): register/login do NOT use this Postgres channel — they emit New Relic telemetry.** During the entire register/login walkthrough, `sync_event` rows created = **0**. The partner-webhook dispatcher is also gated off locally:

```text
2026-07-01 23:41:25,105 - SL - INFO - 2747 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
```

Cite the `EventDispatcher.send_event` gating [app/events/event_dispatcher.py:L62] (because `EVENT_WEBHOOK` defaults to `None` [app/config.py:L612]), and that `RegisterEvent`/`LoginEvent` call `newrelic.agent.record_custom_event(...)` [app/events/auth_event.py:L45]. The real callers of `send_event` are `event_jobs.py`, `app/models.py:L677` (`user_deleted`), `app/models.py:L1687` (`alias_created`), `alias_utils.py`, and `subscription_webhook.py` — **not** register/login. The two listener modes are defined in `event_listener.py`: `LISTENER → PostgresEventSource` and `DEAD_LETTER → DeadLetterEventSource`.

**Claim: cross-process cooperation is visible via three distinct PIDs sharing PostgreSQL.** The webapp worker (PID `2747`) writes the `users` table → the email handler (PID `2723`) reads `Alias`/`Mailbox` and writes an `EmailLog` during the forward → the job runner (PID `2730`) reads/updates the `job` table. Each of the three log lines below carries a different PID, and all three back onto the same database — the concrete "moving pieces talking to each other" evidence:

```text
2026-07-01 23:42:46,808 - SL - DEBUG - 2747 - "/app/app/auth/views/register.py:85" - register() -  - create user blitzy-qna-probe-notact-1782949366@proton.me
2026-07-01 23:45:56,548 - SL - DEBUG - 2723 - "/app/email_handler.py:740" - forward_email_to_mailbox() - 44a9cc52-88ea-4e61-a661-f40e24ecf015 - Create <EmailLog 2> for <Contact 2 outsider@example.com 1>, <User 1 John Wick john@wick.com>, <Mailbox 1 john@wick.com>
2026-07-01 23:45:12,022 - SL - DEBUG - 2730 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 1 onboarding-1 {'user_id': 3}>
```

And PostgreSQL confirms the concurrent backends — `5` connections open on the `simplelogin` database while the three processes (plus the psql probe and the event listener) were running:

```console
$ psql ... -c "SELECT count(*) AS backends, datname FROM pg_stat_activity WHERE datname='simplelogin' GROUP BY datname;"
 backends |   datname
----------+-------------
        5 | simplelogin
(1 row)
```

**Claim: the scheduler (yacron via `cron.py`) defines recurring maintenance jobs.** `crontab.yml` defines **15** jobs. The table below is a **normalized summary** — each row pairs a job's `python cron.py -j <job>` command key with its `schedule:` value (and notes the `concurrencyPolicy: Forbid` flag where present); it is not the raw YAML, and the exact schedule strings are preserved verbatim:

```text
stats                          0 0 * * *
delete_old_monitoring          15 1 * * *
check_custom_domain            15 2 * * *
check_hibp                     15 3 * * *    (concurrencyPolicy: Forbid)
notify_hibp                    15 4 * * *    (concurrencyPolicy: Forbid)
delete_logs                    15 5 * * *
delete_old_data                30 5 * * *
poll_apple_subscription        15 6 * * *
notify_trial_end               15 8 * * *
notify_manual_subscription_end 15 9 * * *
notify_premium_end             15 10 * * *
delete_scheduled_users         15 11 * * *   (concurrencyPolicy: Forbid)
send_undelivered_mails         */5 * * * *   (concurrencyPolicy: Forbid)
clear_alias_audit_log          0 * * * *     (concurrencyPolicy: Forbid)
clear_user_audit_log           0 * * * *     (concurrencyPolicy: Forbid)
```

For reference, the actual YAML of the first four jobs (verbatim from `crontab.yml`, showing the real `name:`/`command:`/`schedule:`/`concurrencyPolicy:` structure the summary above was extracted from):

```yaml
jobs:
  - name: SimpleLogin growth stats
    command: python /code/cron.py -j stats
    shell: /bin/bash
    schedule: "0 0 * * *"
    captureStderr: true

  - name: SimpleLogin Delete Old Monitoring records
    command: python /code/cron.py -j delete_old_monitoring
    shell: /bin/bash
    schedule: "15 1 * * *"
    captureStderr: true

  - name: SimpleLogin Custom Domain check
    command: python /code/cron.py -j check_custom_domain
    shell: /bin/bash
    schedule: "15 2 * * *"
    captureStderr: true

  - name: SimpleLogin HIBP check
    command: python /code/cron.py -j check_hibp
    shell: /bin/bash
    schedule: "15 3 * * *"
    concurrencyPolicy: Forbid
    captureStderr: true
```

Cite `crontab.yml` and the `cron.py` dispatcher, which selects the job from the `-j`/`--job` argument [cron.py:L1265-L1274]. Note: the actual `crontab.yml` marks **both** `clear_alias_audit_log` and `clear_user_audit_log` with `concurrencyPolicy: Forbid` — reproduced in the summary exactly as observed in the file.

---

## Coverage pass (checklist mapping every named sub-part → evidence)

| Sub-part | Answered? | Key evidence (verbatim) | file:line |
|----------|-----------|--------------------------|-----------|
| Q1(a) Ready for user authentication | Yes | `>>> init logging <<<`; `HTTP/1.0 200 OK` + `success` + `Content-Length: 7`; `/` → `302` to `/auth/login` | app/log.py:L67; server.py:L213,L215 |
| Q1(b) Ready for alias-based email | Yes | `Listen for port 20381`; `Start mail controller 0.0.0.0 20381`; `220 05788680d14b Python SMTP 1.4.2` | email_handler.py:L2403,L2386,L2383; job_runner.py:L330,L347 |
| Q2(a) Register | Yes | `Activation Email Sent \| SimpleLogin`; `create user blitzy-qna-probe-1782949284@proton.me`; `Field must be between 8 and 100 characters long` | app/auth/views/register.py:L104,L85,L27; app/mail_sender.py:L130,L137 |
| Q2(b) Verify address | Yes | `auusszqbwhjwqrdesyjhvokjmpgzmm` (len 30); `redirect user to dashboard`; `Your account has been activated` | app/auth/views/register.py:L120,L124; app/auth/views/activate.py:L49,L56,L66,L67 |
| Q2(c) Log in (success + 2 negatives) | Yes | `log user <User 1 John Wick john@wick.com> in`; `Email or password incorrect`; `Please check your inbox for the activation email. You can also have this email re-sent` | app/auth/views/login_utils.py:L35; app/auth/views/login.py:L49,L66-L69,L71,L72 |
| Q2(d) Visible behavior each step | Yes | `after_request()` per-request access log; exact literal `302, takes 0.05317044258117676` (activate GET) | server.py:L284 |
| Q2(e) Forwarded into dashboard | Yes | `Alias \| SimpleLogin`; first alias `simplelogin-newsletter.hereon338@sl.local` (11 occurrences in body) | app/dashboard/views/index.py:L55,L67,L215,L216; app/models.py:L611,L636; templates/dashboard/index.html:L31; templates/base.html:L30-L33 |
| Q3(a) Background jobs doing their part | Yes | `Disable onboarding emails`; `Take job <Job 1 onboarding-1 {'user_id': 3}>`; `ready(0) → taken(1) → done(2)` | app/models.py:L647,L654,L659,L664; job_runner.py:L334,L307,L303-L304; app/config.py:L564,L565,L301 |
| Q3(b) Email forwarding services | Yes | forward phase target `-> simplelogin-newsletter.hereon338@sl.local`; `DMARC check disabled`; `<Alias 4 e0@sl.local> is disabled, do not forward` | email_handler.py:L1945,L1980,L2202,L580,L688,L740,L867,L893,L597; app/handler/dmarc.py:L28,L32,L33; app/spamassassin_utils.py:L17,L136 |
| Q3(c) Identity verification services | Yes | `lROYky7xLLzuEnOCPqwRvA`; `has verified mailbox 7`; `sl-verification=mlumbahjhifhmhbydzhnayxxlwumbx`; `v=spf1 include:sl.local ~all` | app/mailbox_utils.py:L166,L212,L213,L223-L240; app/custom_domain_validation.py:L24,L40,L54,L67,L73,L77,L134 |
| Q3(d) Pieces active & communicating | Yes | `Got NOTIFY: pid=3714 channel=simplelogin_sync_events payload=1`; `Not sending events because webhook is not configured and allowed to be empty` | app/events/event_dispatcher.py:L14,L24-L26,L62; events/event_source.py:L47,L56; app/config.py:L612 |

**Named-item confirmation (every named mechanism/flag/file/example addressed by name):** `/health`, `>>> init logging <<<`, Werkzeug banner suppression, `Listen for port 20381`, `Start mail controller 0.0.0.0 20381`, the SMTP `220` greeting, `create user`, `register_waiting_activation.html`, `NOT_SEND_EMAIL`, `Length(min=8, max=100)`, `ActivationCode`/`random_string(30)`, `activated=True`, `Your account has been activated`, `redirect user to dashboard`, `Email or password incorrect`, the not-activated flash, `log user … in`, `sudo_time`, `dashboard/index.html`, `DISABLE_ONBOARDING`, `Take job`, `get_jobs_to_run`, `JOB_MAX_ATTEMPTS`, `JOB_TAKEN_RETRY_WAIT_MINS`, `JOB_ONBOARDING_1`/`JOB_ONBOARDING_2`/`JOB_ONBOARDING_4`, `handle()`, `Forward phase`, DMARC, SpamAssassin, DKIM, SPF, PGP, the reverse-alias rewrite, `EmailLog`, `mailbox_utils`, `custom_domain_validation`, the `simplelogin_sync_events` NOTIFY/LISTEN channel, `event_dispatcher`/`event_source`/`event_listener`, `RegisterEvent`/`LoginEvent` (New Relic), and `cron.py`/`crontab.yml`.

---

## Honest caveats & local-environment notes

1. **The Flask "Running on http://…" banner is intentionally suppressed** (`app/log.py:L70-L71`); readiness is evidenced instead by `>>> init logging <<<`, `/health` → `success`/`200`, and the port-binding logs.
2. **`NOT_SEND_EMAIL=true` logs mail (subject/from/to only) rather than sending it** [app/mail_sender.py:L130-L137] — the activation link was read from the `activation_code` table in PostgreSQL.
3. **`DISABLE_ONBOARDING=true` means normal onboarding jobs are not enqueued** [app/models.py:L647-L648]; a near-term job was triggered to observe a live `Take job` dispatch.
4. **DMARC and SpamAssassin checks are disabled locally** (unset `DMARC_CHECK_ENABLED` [app/handler/dmarc.py:L32] / unset `SPAMASSASSIN_HOST`).
5. **`RegisterEvent`/`LoginEvent` go to New Relic (a no-op locally), NOT the PostgreSQL event dispatcher** [app/events/auth_event.py:L45] — so register/login create `0` `sync_event` rows, and the partner-webhook dispatcher is gated off [app/events/event_dispatcher.py:L62].
