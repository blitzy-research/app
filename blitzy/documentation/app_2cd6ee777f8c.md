# SimpleLogin — First-Operator Runtime Verification Guide (branch app_2cd6ee777f8c)

This guide answers three runtime-observability questions for a first operator bringing up **SimpleLogin** — an open-source, Flask-based email alias/forwarding service. It is a **verification guide, not a code change**: every source reference below is read-only and cited by `file:line`.

The answers are written from a **real local run** of the system on source branch `app_2cd6ee777f8c` (HEAD `2cd6ee77`). Following the governing discipline, every behavioral claim is immediately followed by the specific captured evidence (in a fenced block) and an exact `file:line` citation — **one claim, one piece of evidence** — and local-environment nuances are reported honestly rather than idealized.

---

## How the system was run

The system is a multi-process Flask application (app-factory pattern). The three long-running processes were started per the documented local run mechanism and observed together.

- **Repository:** SimpleLogin backend + webapp. Branch `app_2cd6ee777f8c`, HEAD `2cd6ee77 chore: emit some missing contact audit logs (#2269)`.
- **Python 3.10.20** (project constraint `python = "^3.10"`), Poetry-locked dependencies (Flask `^1.1.2`, SQLAlchemy `1.3.24`, aiosmtpd `1.4.2`, redis, alembic).
- **PostgreSQL 16.14** on `localhost:35432` (the development baseline is Postgres 13+; a newer server was used locally), role `myuser`, database `simplelogin`.
- **Redis 7.0.15** on port `6379`.
- Working `.env` copied from `example.env` with `DB_URI=postgresql://myuser:mypassword@localhost:35432/simplelogin`. Relevant defaults inherited from `example.env`: `URL=http://localhost:7777` [example.env:L6], `NOT_SEND_EMAIL=true` [example.env:L19], `DISABLE_ONBOARDING=true` [example.env:L150], `EMAIL_DOMAIN=sl.local` [example.env:L22].
- **Run mechanism** (from `CONTRIBUTING.md` §"Run the code locally"): `alembic upgrade head` → `flask dummy-data` → `python server.py` (webapp `:7777`) + `python email_handler.py` (SMTP `:20381`) + `python job_runner.py` (poller).
- **Three long-running processes observed by PID:** webapp **PID 33076** (`server.py` `:7777`), email handler **PID 33077** (`email_handler.py` `:20381`), job runner **PID 33078** (`job_runner.py`).

**Log line format** — every quoted `SL` log line below has this shape (cited once so the reader can parse them all):

```text
{asctime} - SL - {levelname} - {PID} - "{pathname}:{lineno}" - {funcName}() - {message_id} - {message}
```

There is a single `"SL"` logger, configured in `app/log.py` with `LOG = _get_logger("SL")` [app/log.py:L79] and the shortcuts `LOG.d` (debug), `LOG.i` (info), `LOG.w` (warning), `LOG.e` (exception).

**Migration result:** `alembic upgrade head` exited `0` and produced **77 tables**, including `activation_code` and `sync_event`.

**Seed result:** `flask dummy-data` exited `0` and created the demo account — DB row `id=1 | john@wick.com | activated=t | is_admin=t` — plus **11 aliases**. The account is defined by `email="john@wick.com"` [app/fake_data.py:L45], `password="password"` [app/fake_data.py:L47], `activated=True` [app/fake_data.py:L48], with a random alias via `Alias.create_new_random(user)` [app/fake_data.py:L70]; the CLI entry point is `@app.cli.command("dummy-data")` [server.py:L490]. The login walkthrough uses `john@wick.com` / `password`.

### Honest local-environment nuances (read this first)

These are reported up front because they materially change what is observable. Each is reinforced inline where it occurs.

1. **The default Flask "Running on http://…" banner does NOT appear**, because `app/log.py` disables the Werkzeug access logger: `log = logging.getLogger("werkzeug")` [app/log.py:L70] then `log.disabled = True` [app/log.py:L71]. Readiness is instead evidenced by the init banner, `/health`, and the port-binding logs.
2. **`NOT_SEND_EMAIL=true`** means outbound mail is **logged (subject/from/to only), not sent** [app/mail_sender.py:L130-L137], so the activation link is never printed — it was read directly from the `activation_code` table in PostgreSQL.
3. **`DISABLE_ONBOARDING=true`** means a normal registration enqueues **zero** onboarding jobs — the job runner idles. A near-term job was enqueued deliberately to observe a live dispatch.
4. **`RegisterEvent`/`LoginEvent` emit New Relic custom events** (a no-op locally), NOT PostgreSQL events — reported here because it corrects a common misconception.
5. **DMARC and SpamAssassin checks are disabled locally** (unset `DMARC_CHECK_ENABLED` / `SPAMASSASSIN_HOST`).
6. One **ephemeral environment-only accommodation** — a stdlib `re` shim standing in for the optional `re2` package — was used to make dependencies import. It is NOT a repository edit and does not affect any observed auth/email/job/event behavior; it is mentioned once here for full honesty.

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

**Claim: the webapp liveness probe returns success.** Command `curl -i http://localhost:7777/health`:

```http
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 7
Vary: Cookie
Set-Cookie: slapp=...; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.20

success
```

Cite `@app.route("/health", methods=["GET"])` [server.py:L213] and `return "success", 200` [server.py:L215]. Note the session cookie name `slapp`, the literal body `success`, and `Content-Length: 7`.

**Claim: the authentication UI is served — the login form is live.** Command `curl -i http://localhost:7777/auth/login`:

```text
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 342187
(form contains) name="csrf_token"   name="email"   name="password"   ...  "Log in"
```

This confirms the `auth` blueprint is mounted and rendering the login page with CSRF-protected `email`/`password` fields on port `7777`.

**Claim: unauthenticated access to the root is redirected to login — authentication gates the app.** Command `curl -i http://localhost:7777/`:

```http
HTTP/1.0 302 FOUND
Location: http://localhost:7777/auth/login
```

### Q1(b) Ready for alias-based email activity

**Claim: the SMTP email handler binds and announces its port.** From `email.log` at startup:

```text
... - SL - INFO - 33077 - ".../email_handler.py:2403" - <module>() -  - Listen for port 20381
... - SL - DEBUG - 33077 - ".../email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

Cite `LOG.i("Listen for port %s", args.port)` [email_handler.py:L2403] (default port `20381` at [email_handler.py:L2399]) and `LOG.d("Start mail controller %s %s", ...)` [email_handler.py:L2386], where the controller is built with `Controller(MailHandler(), hostname="0.0.0.0", port=port)` [email_handler.py:L2383] inside `def main(port: int)` [email_handler.py:L2381].

**Claim: the SMTP server actually accepts connections (real handshake).** A raw SMTP connect to `127.0.0.1:20381`:

```text
220 reverse-file-mapper-e0b0366d-dg82d Python SMTP 1.4.2
EHLO probe.local
250-SIZE 33554432
250-8BITMIME
250-SMTPUTF8
250 HELP
```

This is the aiosmtpd `1.4.2` greeting plus the advertised ESMTP capabilities (`SIZE 33554432`, `8BITMIME`, `SMTPUTF8`).

**Claim: the background job runner is alive and polling (it backs the async alias/email machinery).** The poll loop runs `while True:` [job_runner.py:L330] under `create_light_app().app_context()`, sleeping between passes with `time.sleep(10)` [job_runner.py:L347]. When there are no due jobs it is silent by design; PID `33078` confirmed the process was running.

**Claim: aliases exist and use the configured domain.** The 11 seed aliases were created on the `sl.local` domain (`EMAIL_DOMAIN=sl.local` [example.env:L22]); e.g., the dummy-data log line:

```text
generate email cinema_evaded043@sl.local
```

---

## Q2 — New-User Journey (register → verify → login → dashboard)

> **Q2:** "Walking through the typical product experience as if you were a new user signing up for the first time. What happens when a user registers, verifies their address, and tries to log in? What visible behavior confirms that the system is handling every step correctly and forwarding the user into the dashboard as expected?"

The walkthrough used a synthetic test address `blitzy-qna-probe-1782937619@proton.me` (POSTed to `/auth/register`; removed at cleanup so the repository and database are left unchanged).

### Q2(a) Register

**Claim: registration renders the "waiting for activation" screen.** `POST /auth/register` → `HTTP 200`, response `<title>`:

```text
Activation Email Sent | SimpleLogin
```

Cite `render_template("auth/register_waiting_activation.html")` [app/auth/views/register.py:L104].

**Claim: the server logs user creation.**

```text
2026-07-01 20:26:59,530 - SL - DEBUG - 33076 - ".../app/auth/views/register.py:85" - register() -  - create user blitzy-qna-probe-1782937619@proton.me
```

Cite `LOG.d("create user %s", email)` [app/auth/views/register.py:L85]; the row is created by `User.create(...)` [app/auth/views/register.py:L86].

**Claim: the activation email is logged, not sent (`NOT_SEND_EMAIL`), so no link is printed.**

```text
2026-07-01 20:26:59,865 - SL - DEBUG - 33076 - ".../app/mail_sender.py:131" - send() -  - send email with subject 'Just one more step to join SimpleLogin', from '"noreply@sl.local" <noreply@sl.local>' to 'blitzy-qna-probe-1782937619@proton.me'
```

Cite the guard `if config.NOT_SEND_EMAIL:` [app/mail_sender.py:L130], which logs subject/from/to and then `return True` [app/mail_sender.py:L137]. The activation email subject literal is `Just one more step to join SimpleLogin`.

**Claim: the password policy is enforced (minimum length 8).** Registering with a 5-character password `short` → `HTTP 200` with the page error:

```text
Field must be between 8 and 100 characters long
```

DB user rows for that email = **0** (rejected before `User.create`). Cite `validators.Length(min=8, max=100)` [app/auth/views/register.py:L27].

**Claim (honest): the success `RegisterEvent` produces NO local log line**, because it is New Relic telemetry (a no-op locally). Cite `RegisterEvent(RegisterEvent.ActionType.success).send()` [app/auth/views/register.py:L96], whose `send()` calls `newrelic.agent.record_custom_event(...)` [app/events/auth_event.py:L45]. The daily metric increment `nb_new_web_non_proton_user += 1` [app/auth/views/register.py:L97] was verified in the database as `daily_metric.nb_new_web_non_proton_user = 3`.

### Q2(b) Verify address

Because `NOT_SEND_EMAIL` suppresses the printed link, the activation code was **read directly from the `activation_code` table in PostgreSQL** — this is the method used to obtain the code.

**Claim: an activation code (30 random chars) exists for the new, not-yet-activated user.** DB read: user `id=3`, `activated=False`; `activation_code.code`:

```text
eueiwztiymdlhnbbrkyvkgwgwxjpqv        (length 30)
```

Cite `ActivationCode.create(user_id=user.id, code=random_string(30))` [app/auth/views/register.py:L120].

**Claim: the activation link format is deterministic.**

```text
http://localhost:7777/auth/activate?code=eueiwztiymdlhnbbrkyvkgwgwxjpqv
```

Cite `activation_link = f"{URL}/auth/activate?code={activation.code}"` [app/auth/views/register.py:L124].

**Claim: visiting the activation link activates the account and redirects to the dashboard.** `GET /auth/activate?code=...`:

```http
HTTP/1.0 302 FOUND
Location: http://localhost:7777/dashboard/
```

```text
... - ".../app/auth/views/activate.py:66" - activate() -  - redirect user to dashboard
```

Cite `user.activated = True` [app/auth/views/activate.py:L49], `login_user(user)` [app/auth/views/activate.py:L50], `LOG.d("redirect user to dashboard")` [app/auth/views/activate.py:L66], and `return redirect(url_for("dashboard.index"))` [app/auth/views/activate.py:L67].

**Claim: the database confirms activation flipped.** The `users.activated` column went `False → True` after the GET.

**Claim: a welcome email is (logged as) sent on activation.**

```text
... - ".../app/mail_sender.py:131" - send() -  - send email with subject 'Welcome to SimpleLogin', from '"noreply@sl.local" <noreply@sl.local>' to 'simplelogin-newsletter.orator388@sl.local'
```

Cite `email_utils.send_welcome_email(user)` [app/auth/views/activate.py:L58].

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
... - ".../app/auth/views/login_utils.py:35" - after_login() -  - log user <User 1 John Wick john@wick.com> in
... - ".../app/auth/views/login_utils.py:44" - after_login() -  - redirect user to dashboard
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

Additionally, the webapp emits a per-request access log (its own line, since the Werkzeug logger is disabled):

```text
... - ".../server.py:284" - after_request() -  - 127.0.0.1 GET /auth/activate ImmutableMultiDict([('code','...')]) 302, takes 0.05...
```

Cite the request-logging `LOG.d(...)` in `@app.after_request def after_request(res)` [server.py:L284].

### Q2(e) Forwarded into the dashboard

**Claim: the authenticated user lands on the Alias dashboard.** `GET /dashboard/` → `HTTP 200`, `<title>`:

```text
Alias | SimpleLogin
```

Cite `@dashboard_bp.route("/", methods=["GET", "POST"])` [app/dashboard/views/index.py:L55], `def index()` [app/dashboard/views/index.py:L67], and the `return render_template(` [app/dashboard/views/index.py:L215] with `"dashboard/index.html",` [app/dashboard/views/index.py:L216] (render call at L215, template-name literal at L216).

**Claim: the user's aliases render on the dashboard.** The body contains the first alias `simplelogin-newsletter`. This ties to the `User.create` side effects observed in the database: a default pre-verified mailbox `('blitzy-qna-probe-...@proton.me', verified=True)` — cite `Mailbox.create(user_id=user.id, email=user.email, verified=True)` [app/models.py:L611] — and the first alias `simplelogin-newsletter.orator388@sl.local` — cite `Alias.create_new(user, prefix="simplelogin-newsletter", ...)` [app/models.py:L636].

---

## Q3 — Behind the Scenes

> **Q3:** "What the application does behind the scenes during that flow. Are there any indicators that background jobs or internal services are doing their part to support email forwarding or identity verification? What should I expect to observe at runtime that tells me these moving pieces are active and talking to each other properly?"

### Q3(a) Are background jobs doing their part?

**Claim (honest idle behavior): with `DISABLE_ONBOARDING=true`, a fresh registration enqueues ZERO onboarding jobs and the runner idles.** From `User.create` during dummy-data / registration:

```text
... - ".../app/models.py:647" - create() -  - Disable onboarding emails
```

Cite `if config.DISABLE_ONBOARDING:` → `LOG.d("Disable onboarding emails")` [app/models.py:L647], immediately followed by an early `return user` [app/models.py:L648]. The `+1`/`+2`/`+3`-day onboarding jobs — `JOB_ONBOARDING_1` with `run_at=arrow.now().shift(days=1)` [app/models.py:L654], `JOB_ONBOARDING_2` at `+2 days` [app/models.py:L659], and `JOB_ONBOARDING_4` at `+3 days` [app/models.py:L664] — are enqueued **only** when onboarding is enabled, and even then would not fire immediately (see the selection window below). This is why the runner is silent right after signup; this is the real behavior, not an idealized one.

**Claim: the selection window only picks jobs due within ~10 minutes; retry/attempt limits gate re-runs.** Captured at runtime:

```text
get_jobs_to_run window: run_at <= now+10min = 2026-07-01 20:41:21 | taken retry after 30 min | JOB_MAX_ATTEMPTS 5
```

Cite `get_jobs_to_run()` [job_runner.py:L307], which selects jobs where `state == ready`, **or** `taken` older than `JOB_TAKEN_RETRY_WAIT_MINS` with `attempts < JOB_MAX_ATTEMPTS`, and `run_at` is NULL or `≤ now + 10 min` (`return query.all()` [job_runner.py:L326]); the limits are `JOB_TAKEN_RETRY_WAIT_MINS = 30` [app/config.py:L565] and `JOB_MAX_ATTEMPTS = 5` [app/config.py:L564].

**Claim: a live dispatch was observed by enqueuing a near-term job.** Enqueuing `Job(name="onboarding-1", payload={"user_id": 3}, run_at=now)`, the separate job-runner process picked it up within its 10-second poll:

```text
... - ".../job_runner.py:334" - <module>() -  - Take job <Job 1 onboarding-1 {'user_id': 3}>
... - ".../job_runner.py:196" - process_job() -  - send onboarding send-from-alias email to user <User 3 blitzy-qna-probe-...@proton.me>
... - ".../app/mail_sender.py:131" - send() -  - send email with subject 'SimpleLogin Tip: Send emails from your alias', from '"noreply@sl.local" <noreply@sl.local>' to 'simplelogin-newsletter.orator388@sl.local'
```

The final job state was `state=2` (done), `taken=True`, `attempts=1` — the full `ready(0) → taken(1) → done(2)` lifecycle. Cite `LOG.d("Take job %s", job)` [job_runner.py:L334] and the name-based dispatch in `process_job()` [job_runner.py:L188]: `onboarding-1` → `onboarding_send_from_alias` [job_runner.py:L196]; `onboarding-2` → mailbox onboarding [job_runner.py:L198,L205]; `onboarding-4` → PGP onboarding [job_runner.py:L207]; plus batch-import, delete-account/mailbox/domain, send-user-report, send-proton-welcome-1, and send-alias-creation-events; an unknown name falls to `else:` → `LOG.e("Unknown job name %s", job.name)` [job_runner.py:L303-L304]. The `JOB_ONBOARDING_1` literal is `"onboarding-1"` [app/config.py:L301].

### Q3(b) Internal services supporting email forwarding

A real message was sent over SMTP to `127.0.0.1:20381`; the email handler processed it end-to-end. **Every line below is correlated by the same `message_id` `0273b844-a843-4a78-89e7-474626fa9b94`**, which shows the pipeline is one coherent flow.

**Claim: a correlation `message_id` is set for the incoming mail.**

```text
... - ".../app/log.py:24" - set_message_id() -  - set message_id 0273b844-...
```

Cite `set_message_id(...)` (`LOG.d("set message_id %s", message_id)`) [app/log.py:L24].

**Claim: the handler router receives the mail.**

```text
... - ".../email_handler.py:1980" - handle() - 0273b844-... - ==>> Handle mail_from:outsider@example.com, rcpt_tos:['simplelogin-newsletter.fawned261@sl.local'], ... message_id:<...@example.com>
```

Cite the main router `def handle(envelope: Envelope, msg: Message) -> str` [email_handler.py:L1945] and the `==>> Handle` log [email_handler.py:L1980].

**Claim: the forward phase is selected (inbound → alias → owner).**

```text
... - ".../email_handler.py:2202" - handle() -  - Forward phase outsider@example.com(outsider@example.com) -> simplelogin-newsletter.fawned261@sl.local
```

Cite [email_handler.py:L2202].

**Claim: the sender contact is created/looked up.**

```text
... - ".../email_handler.py:580" - handle_forward() -  - Create or get contact for from_header:outsider@example.com
```

Cite `def handle_forward(envelope, msg, rcpt_to)` [email_handler.py:L536] and the contact log [email_handler.py:L580].

**Claim: the DMARC policy check runs but is disabled locally (reported honestly).**

```text
... - ".../app/handler/dmarc.py:33" - apply_dmarc_policy_for_forward_phase() -  - DMARC check disabled
```

Cite `def apply_dmarc_policy_for_forward_phase(...)` [app/handler/dmarc.py:L28] and the disabled guard `if not DMARC_CHECK_ENABLED or not spam_result:` [app/handler/dmarc.py:L32] → `LOG.i("DMARC check disabled")` [app/handler/dmarc.py:L33]. `DMARC_CHECK_ENABLED` is unset, and `SPAMASSASSIN_HOST` is unset so the spam scan is skipped (`MAX_SPAM_SCORE` default `5.5`).

**Claim: routing resolves Contact → Alias → Mailbox (the alias owner).**

```text
... - ".../email_handler.py:688" - forward_email_to_mailbox() -  - Forward <Contact 3 outsider@example.com 1> -> <Alias 1 simplelogin-newsletter.fawned261@sl.local> -> <Mailbox 1 john@wick.com>
```

Cite [email_handler.py:L688].

**Claim: an `EmailLog` row is written (audit of the forward).**

```text
... - ".../email_handler.py:740" - forward_email_to_mailbox() -  - Create <EmailLog 3> for <Contact 3...>, <User 1 John Wick...>, <Mailbox 1 john@wick.com>
```

Cite [email_handler.py:L740].

**Claim: the `From` header is rewritten to a reverse-alias (so replies route back through SimpleLogin).**

```text
... - ".../email_handler.py:867" - ... - From header, new:"outsider at example.com" <outsider_at_example_com_mjejrv@sl.local>, old:outsider@example.com
```

Cite [email_handler.py:L867], plus the related header rewrites `Replace ... header` / `Delete ... header` [email_handler.py:L313-L316] and `missing date header, create one` [email_handler.py:L857].

**Claim: the message is forwarded to the owner mailbox.**

```text
... - ".../email_handler.py:893" - forward_email_to_mailbox() -  - Forward mail from outsider@example.com to john@wick.com
... - ".../app/mail_sender.py:131" - send() - 0273b844-... - send email with subject 'Blitzy QnA enabled forward', from '"outsider at example.com" <outsider_at_example_com_mjejrv@sl.local>' to 'simplelogin-newsletter.fawned261@sl.local'
```

Cite [email_handler.py:L893]; the final send is at [app/mail_sender.py:L131]. With `NOT_SEND_EMAIL` off, the real relay target would be `POSTFIX_SERVER` default `240.0.0.1` [app/config.py:L136] on `POSTFIX_PORT` default `25` [app/config.py:L149] — bypassed here by `NOT_SEND_EMAIL`.

**Claim: a disabled alias is NOT forwarded (negative path).** Sending to the disabled alias `e0@sl.local`:

```text
... - ".../email_handler.py:597" - handle_forward() -  - <Alias 4 e0@sl.local> is disabled, do not forward
```

Cite [email_handler.py:L597].

**Named email-security helpers in the forwarding path (by name):**

- **DMARC** — `app/handler/dmarc.py` (`apply_dmarc_policy_for_forward_phase` [app/handler/dmarc.py:L28]); **disabled locally**.
- **SpamAssassin** — `app/spamassassin_utils.py` (`class SpamAssassin` [app/spamassassin_utils.py:L17], `def is_spam(self, level=5)` [app/spamassassin_utils.py:L136]); **disabled locally** (no `SPAMASSASSIN_HOST`).
- **DKIM** — via the `dkimpy` library.
- **SPF** — via the `pyspf` library.
- **PGP** — encryption via a temporary `GNUPGHOME` directory.

### Q3(c) Internal services supporting identity verification

All three identity-verification services, by name:

**(1) Registration activation code.** Subject `Just one more step to join SimpleLogin`; the `ActivationCode.code` is 30 chars via `random_string(30)` [app/auth/views/register.py:L120], and `GET /auth/activate?code=...` sets `activated=True` [app/auth/views/activate.py:L49] (full evidence in Q2(b)).

**(2) Mailbox verification (`app/mailbox_utils.py`) — observed live.** An unverified mailbox (id=8) was created, an activation code generated, and the code verified:

```text
MailboxActivation.code = 'bi5bThOvdzM79k-iVf6iAA'
... - ".../app/mailbox_utils.py:212" - verify_mailbox_code() -  - User <User 1 John Wick john@wick.com> has verified mailbox 8
after verify_mailbox_code -> mailbox.verified = True
```

Cite `def generate_activation_code(...)` [app/mailbox_utils.py:L223-L240] (code via `secrets.token_urlsafe(16)`), `def verify_mailbox_code(user, mailbox_id, code)` [app/mailbox_utils.py:L166], the verified log `LOG.i(f"User {user} has verified mailbox {mailbox_id}")` [app/mailbox_utils.py:L212], and `mailbox.verified = True` [app/mailbox_utils.py:L213]. Also note `def create_mailbox(...)` [app/mailbox_utils.py:L46] and that `delete_mailbox` enqueues a `Job` via `Job.create(...)` [app/mailbox_utils.py:L145], tying mailbox management to the background job system.

**(3) Custom-domain DNS validation (`app/custom_domain_validation.py`) — observed live** (a temporary domain, deleted afterward). The expected DNS records the operator must publish to prove domain ownership + mail routing:

```text
ownership TXT record : sl-verification=ewkwvcoxaaizvaxtgbylubvcldfxwl
expected MX records  : [(10, 'email.hostname.')]
expected SPF record  : v=spf1 include:sl.local ~all
expected DKIM records: {'dkim._domainkey': 'dkim._domainkey.sl.local', 'dkim02._domainkey': 'dkim02._domainkey.sl.local', 'dkim03._domainkey': 'dkim03._domainkey.sl.local'}
```

Cite `class CustomDomainValidation` [app/custom_domain_validation.py:L24], `get_ownership_verification_record` [app/custom_domain_validation.py:L40], `get_expected_mx_records` [app/custom_domain_validation.py:L54], the SPF methods `get_expected_spf_domain` [app/custom_domain_validation.py:L67] / `get_expected_spf_record` [app/custom_domain_validation.py:L73], `get_dkim_records` [app/custom_domain_validation.py:L77], and `validate_domain_ownership` [app/custom_domain_validation.py:L134].

### Q3(d) Runtime observations proving the pieces are active and communicating

**Claim: the genuine inter-process channel is PostgreSQL NOTIFY/LISTEN on `simplelogin_sync_events`.** A listener issued `LISTEN simplelogin_sync_events`; the dispatcher wrote a `SyncEvent` and executed `NOTIFY` (on commit):

```text
[listener] LISTEN simplelogin_sync_events
[sender] PostgresDispatcher().send(...) + commit done
[listener] Got NOTIFY: pid=39098 channel=simplelogin_sync_events payload=2
SyncEvent persisted by dispatcher: id=2 content=b'blitzy-qna-test-event-payload'
```

Cite the channel name `NOTIFICATION_CHANNEL = "simplelogin_sync_events"` [app/events/event_dispatcher.py:L14] and `PostgresDispatcher.send` [app/events/event_dispatcher.py:L24] — `SyncEvent.create(content=event, flush=True)` [app/events/event_dispatcher.py:L25] then `Session.execute(f"NOTIFY {NOTIFICATION_CHANNEL}, '{instance.id}';")` [app/events/event_dispatcher.py:L26]; on the consumer side `cursor.execute(f"LISTEN {NOTIFICATION_CHANNEL};")` [events/event_source.py:L47] and its `Got NOTIFY: pid=… channel=… payload=…` log [events/event_source.py:L56]. **Important honesty note:** the NOTIFY only delivers on transaction **COMMIT** — `flush` alone was insufficient; this was confirmed as an observed fact.

**Claim (honest): register/login do NOT use this Postgres channel — they emit New Relic telemetry.** During the entire register/login walkthrough, `sync_event` rows created = **0**. The partner-webhook dispatcher is also gated off locally:

```text
... - ".../app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
```

Cite the `EventDispatcher.send_event` gating [app/events/event_dispatcher.py:L62] (because `EVENT_WEBHOOK` defaults to `None` [app/config.py:L612]), and that `RegisterEvent`/`LoginEvent` call `newrelic.agent.record_custom_event(...)` [app/events/auth_event.py:L45]. The real callers of `send_event` are `event_jobs.py`, `app/models.py:L677` (`user_deleted`), `app/models.py:L1687` (`alias_created`), `alias_utils.py`, and `subscription_webhook.py` — **not** register/login. The two listener modes are defined in `event_listener.py`: `LISTENER → PostgresEventSource` and `DEAD_LETTER → DeadLetterEventSource`.

**Claim: cross-process cooperation is already visible via three PIDs sharing PostgreSQL.** The webapp (PID `33076`) writes users/aliases → the email handler (PID `33077`) reads `Alias`/`Mailbox` and writes an `EmailLog` during the forward → the webapp writes a `Job` that the job runner (PID `33078`) reads and dispatches (`Take job`). This is the concrete "moving pieces talking to each other" evidence: three separate processes cooperating through the shared database.

**Claim: the scheduler (yacron via `cron.py`) defines recurring maintenance jobs.** `crontab.yml` defines **15** jobs; reproduced verbatim from the file (short names are the `python cron.py -j <job>` keys):

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

Cite `crontab.yml` and the `cron.py` dispatcher, which selects the job from the `-j`/`--job` argument [cron.py:L1265-L1274]. Note: unlike the source-plan's earlier table, the actual `crontab.yml` marks **both** `clear_alias_audit_log` and `clear_user_audit_log` with `concurrencyPolicy: Forbid` — reproduced here exactly as observed in the file.

---

## Coverage pass (checklist mapping every named sub-part → evidence)

| Sub-part | Answered? | Key evidence (verbatim) | file:line |
|----------|-----------|--------------------------|-----------|
| Q1(a) Ready for user authentication | Yes | `>>> init logging <<<`; `HTTP/1.0 200 OK` + `success` + `Content-Length: 7`; `/` → `302` to `/auth/login` | app/log.py:L67; server.py:L213,L215 |
| Q1(b) Ready for alias-based email | Yes | `Listen for port 20381`; `Start mail controller 0.0.0.0 20381`; `220 reverse-file-mapper-e0b0366d-dg82d Python SMTP 1.4.2` | email_handler.py:L2403,L2386,L2383; job_runner.py:L330,L347 |
| Q2(a) Register | Yes | `Activation Email Sent \| SimpleLogin`; `create user blitzy-qna-probe-1782937619@proton.me`; `Field must be between 8 and 100 characters long` | register.py:L104,L85,L27; mail_sender.py:L130,L137 |
| Q2(b) Verify address | Yes | `eueiwztiymdlhnbbrkyvkgwgwxjpqv` (len 30); `redirect user to dashboard`; `Your account has been activated` | register.py:L120,L124; activate.py:L49,L56,L66,L67 |
| Q2(c) Log in (success + 2 negatives) | Yes | `log user <User 1 John Wick john@wick.com> in`; `Email or password incorrect`; `Please check your inbox for the activation email. You can also have this email re-sent` | login_utils.py:L35; login.py:L49,L66-L69,L71,L72 |
| Q2(d) Visible behavior each step | Yes | Per-request access log `... after_request() ... GET /auth/activate ... 302, takes 0.05...` | server.py:L284 |
| Q2(e) Forwarded into dashboard | Yes | `Alias \| SimpleLogin`; first alias `simplelogin-newsletter` | index.py:L55,L67,L215,L216; models.py:L611,L636 |
| Q3(a) Background jobs doing their part | Yes | `Disable onboarding emails`; `Take job <Job 1 onboarding-1 {'user_id': 3}>`; `ready(0) → taken(1) → done(2)` | models.py:L647,L654,L659,L664; job_runner.py:L334,L307,L303-L304; config.py:L564,L565,L301 |
| Q3(b) Email forwarding services | Yes | `Forward phase ... -> simplelogin-newsletter.fawned261@sl.local`; `DMARC check disabled`; `<Alias 4 e0@sl.local> is disabled, do not forward` | email_handler.py:L1945,L1980,L2202,L580,L688,L740,L867,L893,L597; dmarc.py:L28,L32,L33; spamassassin_utils.py:L17,L136 |
| Q3(c) Identity verification services | Yes | `bi5bThOvdzM79k-iVf6iAA`; `has verified mailbox 8`; `sl-verification=ewkwvcoxaaizvaxtgbylubvcldfxwl`; `v=spf1 include:sl.local ~all` | mailbox_utils.py:L166,L212,L213,L223-L240; custom_domain_validation.py:L24,L40,L54,L67,L73,L77,L134 |
| Q3(d) Pieces active & communicating | Yes | `Got NOTIFY: pid=39098 channel=simplelogin_sync_events payload=2`; `Not sending events because webhook is not configured and allowed to be empty` | event_dispatcher.py:L14,L24-L26,L62; event_source.py:L47,L56; config.py:L612 |

**Named-item confirmation (every named mechanism/flag/file/example addressed by name):** `/health`, `>>> init logging <<<`, Werkzeug banner suppression, `Listen for port 20381`, `Start mail controller 0.0.0.0 20381`, the SMTP `220` greeting, `create user`, `register_waiting_activation.html`, `NOT_SEND_EMAIL`, `Length(min=8, max=100)`, `ActivationCode`/`random_string(30)`, `activated=True`, `Your account has been activated`, `redirect user to dashboard`, `Email or password incorrect`, the not-activated flash, `log user … in`, `sudo_time`, `dashboard/index.html`, `DISABLE_ONBOARDING`, `Take job`, `get_jobs_to_run`, `JOB_MAX_ATTEMPTS`, `JOB_TAKEN_RETRY_WAIT_MINS`, `JOB_ONBOARDING_1`/`JOB_ONBOARDING_2`/`JOB_ONBOARDING_4`, `handle()`, `Forward phase`, DMARC, SpamAssassin, DKIM, SPF, PGP, the reverse-alias rewrite, `EmailLog`, `mailbox_utils`, `custom_domain_validation`, the `simplelogin_sync_events` NOTIFY/LISTEN channel, `event_dispatcher`/`event_source`/`event_listener`, `RegisterEvent`/`LoginEvent` (New Relic), and `cron.py`/`crontab.yml`.

---

## Honest caveats & local-environment notes

1. **The Flask "Running on http://…" banner is intentionally suppressed** (`app/log.py:L70-L71`); readiness is evidenced instead by `>>> init logging <<<`, `/health` → `success`/`200`, and the port-binding logs.
2. **`NOT_SEND_EMAIL=true` logs mail (subject/from/to only) rather than sending it** [app/mail_sender.py:L130-L137] — the activation link was read from the `activation_code` table in PostgreSQL.
3. **`DISABLE_ONBOARDING=true` means normal onboarding jobs are not enqueued** [app/models.py:L647-L648]; a near-term job was triggered to observe a live `Take job` dispatch.
4. **DMARC and SpamAssassin checks are disabled locally** (unset `DMARC_CHECK_ENABLED` [app/handler/dmarc.py:L32] / unset `SPAMASSASSIN_HOST`).
5. **`RegisterEvent`/`LoginEvent` go to New Relic (a no-op locally), NOT the PostgreSQL event dispatcher** [app/events/auth_event.py:L45] — so register/login create `0` `sync_event` rows, and the partner-webhook dispatcher is gated off [app/events/event_dispatcher.py:L62].
6. **A stdlib `re` shim standing in for the optional `re2` package** was an ephemeral environment accommodation only — no repository file was edited, and it is unrelated to the observed auth/email/job/event behavior.
