# SimpleLogin — First-Run Verification Guide (local deployment)

> **Branch:** `app_2cd6ee777f8c` · **Method:** observation-first (every claim below is backed by a
> `file:line` reference **and/or** verbatim output captured from actually running the code).

This document answers a new user's three questions about a **locally-running** SimpleLogin
deployment:

- **Q1 — Liveness/readiness:** Once the app starts, how can I tell the **web server**, **email
  handler**, and **job runner** are actually up and responding? What confirms in the logs / dashboard
  that users can sign in and manage aliases?
- **Q2 — End-to-end actions:** Create an account, create an alias, and have that alias receive an
  email. What happens, and how does the system show each action was handled correctly?
- **Q3 — Background components:** Do the email handler and job runner come online automatically in the
  background? What behavior shows they function as intended across different situations?

Everything here was produced by **building and running** the three processes, exercising the real user
workflows, and capturing the actual output — not by reading the code alone. Each captured block is
shown immediately below the exact command/code that produced it.

## Environment used for these observations

| Item | Value (observed) |
|------|------------------|
| Python | `Python 3.10.20` (in-project virtualenv `./.venv`; the system `python3` is 3.13 and is **not** used) |
| Dependency manager | `Poetry (version 1.8.5)` |
| Database | PostgreSQL `13.23` — `/var/run/postgresql:5432 - accepting connections` |
| Cache/session store | Redis — `PONG` on `localhost:6379` |
| App config file | `example.env` (loaded via `CONFIG=example.env`) |
| Key local flags | `NOT_SEND_EMAIL=true` [`example.env:L19`], `EMAIL_DOMAIN=sl.local` [`example.env:L22`], `URL=http://localhost:7777` [`example.env:L6`], `DISABLE_ONBOARDING=true` [`example.env:L150`] |
| Demo account | `john@wick.com` / `password` (name "John Wick", `is_admin=True`, `activated=True`) [`app/fake_data.py:L45-L49`] |

> **Why `NOT_SEND_EMAIL=true` matters throughout.** Locally, SimpleLogin does **not** contact a real
> SMTP relay. `mail_sender.send()` short-circuits: `if config.NOT_SEND_EMAIL:` [`app/mail_sender.py:L130`]
> it **logs** the email and returns `True` [`app/mail_sender.py:L137`] instead of delivering it. So a
> "sent" or "forwarded" email manifests **as a log line plus a database row**, not as a message in an
> external inbox. This is stated wherever it is relevant below.

## The four evidence channels (legend)

Every factual claim in this guide is grounded in at least one of four channels, each cited with a
`file:line`:

1. **HTTP response** — status line, headers, and body (e.g. `curl -i`).
2. **UI flash message** — a toast rendered by the dashboard/auth pages (Flask `flash(...)`).
3. **`SL` stdout log line** — emitted by the `SL` logger [`app/log.py:L79`] with a format that embeds
   the source `pathname:lineno`, function name, and an email `message_id` correlation field
   [`app/log.py:L12-L15`]:

   ```text
   %(asctime)s - %(name)s - %(levelname)s - %(process)d - "%(pathname)s:%(lineno)d" - %(funcName)s() - %(message_id)s - %(message)s
   ```

   The `"pathname:lineno"` prefix is preserved in every quoted log line below — it *is* part of the
   evidence. Every process also prints the import banner `>>> init logging <<<` [`app/log.py:L67`] at
   startup, a useful "the app package imported cleanly" signal.
4. **Database row** — a row in PostgreSQL (`users`, `alias`, `email_log`, `job`, …) that records the
   durable effect of an action.

---

## How to run it locally (first-run sequence)

The three subject components are **three independent entry-point processes**. The canonical local
bring-up (per `CONTRIBUTING.md`: `alembic upgrade head && flask dummy-data && python3 server.py`) is:

1. **Apply the schema** (once): `CONFIG=example.env poetry run alembic upgrade head`. In this
   environment the schema was already applied — the `simplelogin` database contains **77** tables:

   ```bash
   $ docker exec sl-db psql -U myuser -d simplelogin -t \
       -c "SELECT count(*) FROM information_schema.tables WHERE table_schema='public';"
   ```
   ```text
   77
   ```

2. **Seed baseline records + demo account** with `flask dummy-data`, which runs `fake_data()` +
   `add_sl_domains()` + `add_proton_partner()` [`server.py:L490-L497`]:

   ```bash
   $ CONFIG=example.env PYTHONPATH=. FLASK_APP=server.py poetry run flask dummy-data
   ```
   ```text
   >>> init logging <<<
   2026-07-01 04:12:35,462 - SL - WARNING - 41336 - ".../server.py:494" - dummy_data() -  - reset db, add fake data
   2026-07-01 04:12:35,462 - SL - DEBUG - 41336 - ".../app/fake_data.py:41" - fake_data() -  - create fake data
   2026-07-01 04:12:36,231 - SL - INFO - 41336 - ".../init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain
   ```

   The demo account is now present (`activated=t`, `is_admin=t`):

   ```bash
   $ docker exec sl-db psql -U myuser -d simplelogin \
       -c "SELECT id,email,activated,is_admin,name FROM users WHERE email='john@wick.com';"
   ```
   ```text
    id |     email     | activated | is_admin |   name
   ----+---------------+-----------+----------+-----------
     1 | john@wick.com | t         | t        | John Wick
   (1 row)
   ```

3. **Start the three processes**, each in its own terminal / background job (they do **not** start
   each other — see Q3):

   ```bash
   # Web server (production form = Dockerfile CMD)
   CONFIG=example.env PYTHONPATH=. poetry run gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
   # Email handler (aiosmtpd SMTP daemon)
   CONFIG=example.env PYTHONPATH=. poetry run python email_handler.py
   # Job runner (10-second poll loop)
   CONFIG=example.env PYTHONPATH=. poetry run python job_runner.py
   ```

   `wsgi.py` is the Gunicorn target — `from server import create_app` then `app = create_app()`.

---

## Q1 — Are the web server, email handler, and job runner up?

Answered per component below: **(1)** how to start it, **(2)** the exact startup log signature to look
for, **(3)** the liveness probe and its exact result, and **(4)** the sign-in / alias-management
signals. Reasoning follows each.

### Q1.1 — Web server (Flask served by Gunicorn / `wsgi:app`)

**Start it** (production form, matching the `Dockerfile` CMD `["gunicorn","wsgi:app","-b","0.0.0.0:7777","-w","2","--timeout","15"]` [`Dockerfile:L47`]):

```bash
$ CONFIG=example.env PYTHONPATH=. poetry run gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
```

**Startup signature** (captured verbatim):

```text
[2026-07-01 04:13:18 +0000] [42160] [INFO] Starting gunicorn 20.0.4
[2026-07-01 04:13:18 +0000] [42160] [INFO] Listening at: http://0.0.0.0:7777 (42160)
[2026-07-01 04:13:18 +0000] [42160] [INFO] Using worker: sync
[2026-07-01 04:13:18 +0000] [42170] [INFO] Booting worker with pid: 42170
[2026-07-01 04:13:18 +0000] [42171] [INFO] Booting worker with pid: 42171
```

The two `Booting worker` lines correspond to `-w 2` (two workers). The port `7777` matches the local
dev default `app.run(debug=True, port=7777)` [`server.py:L588`] and the Gunicorn bind.

**Liveness probe — the health endpoint.** The route is `@app.route("/health", methods=["GET"])`
[`server.py:L213`] → `def healthcheck():` [`server.py:L214`] → `return "success", 200` [`server.py:L215`]:

```bash
$ curl -i http://localhost:7777/health
```
```http
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Wed, 01 Jul 2026 04:13:29 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 7
Vary: Cookie
Set-Cookie: slapp=<redacted-session-cookie>; Expires=Wed, 08-Jul-2026 04:13:29 GMT; HttpOnly; Path=/; SameSite=Lax

success
```

The exact result is **status `200`** and **body `success`** (`Content-Length: 7` — the 7 bytes of
`success`). This is the definitive "web server is up and responding" signal.

**Routing is alive — root redirect.** An unauthenticated `GET /` redirects to the login page
(`redirect(url_for("auth.login"))`) [`server.py:L250-L255`]:

```bash
$ curl -i http://localhost:7777/
```
```http
HTTP/1.1 302 FOUND
Server: gunicorn/20.0.4
Content-Length: 229
Location: http://localhost:7777/auth/login
...
```

The `Set-Cookie: slapp=...` header above matches `SESSION_COOKIE_NAME = "slapp"` [`app/config.py:L199`],
confirming the session layer is wired up. Breadth of service is registered by `register_blueprints(app)`
[`server.py:L233`], which mounts `auth_bp, monitor_bp, dashboard_bp, developer_bp, phone_bp, oauth_bp`
(at both `/oauth` and `/oauth2`), `onboarding_bp, discover_bp, internal_bp, api_bp` [`server.py:L234-L246`].

**Sign-in signal (dashboard/UI).** Exercising a real login with the seeded `john@wick.com / password`
(CSRF handled). Success redirects to `dashboard.index` (`return redirect(url_for("dashboard.index"))`)
[`app/auth/views/login.py:L34`]:

```text
POST /auth/login -> HTTP 302
Location: http://localhost:7777/dashboard/
```

A **deliberately wrong** password renders the login page (HTTP 200) with the flash
`"Email or password incorrect"` [`app/auth/views/login.py:L49`], which surfaces in the page as a toastr
error:

```text
POST /auth/login (wrong pw) -> HTTP 200
... <script>toastr.error("Email or password incorrect");</script>
```

**Alias-management signal.** Once authenticated, `GET /` redirects to `/dashboard/`, and the dashboard
index (rendered from `templates/dashboard/index.html`) is the alias-management page:

```text
GET / (authenticated) -> HTTP 302 Location: http://localhost:7777/dashboard/
GET /dashboard/ (authenticated) -> HTTP 200
<title> -> Alias | SimpleLogin
  contains 'Random alias': True
  contains 'Create': True
  contains 'New custom alias': True
```

**Reasoning.** `200 success` on `/health` proves the WSGI worker is serving requests; the `302` on `/`
proves routing + session are functional; the login `302` to `/dashboard/` proves authentication works;
and the dashboard page (title "Alias | SimpleLogin", with "Random alias" / "New custom alias" controls)
proves a signed-in user can manage aliases.

### Q1.2 — Email handler (`aiosmtpd` SMTP daemon)

**Start it:**

```bash
$ CONFIG=example.env PYTHONPATH=. poetry run python email_handler.py
```

**Startup signature** (captured verbatim — these lines prove the SMTP listener is bound):

```text
>>> init logging <<<
2026-07-01 04:13:43,454 - SL - INFO - 42585 - ".../email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-01 04:13:43,457 - SL - DEBUG - 42585 - ".../email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

- `Listen for port 20381` is `LOG.i("Listen for port %s", args.port)` [`email_handler.py:L2403`]; the
  `--port` argument defaults to **`20381`** [`email_handler.py:L2399`].
- `Start mail controller 0.0.0.0 20381` is `LOG.d("Start mail controller %s %s", controller.hostname,
  controller.port)` [`email_handler.py:L2386`], printed after the controller
  `Controller(MailHandler(), hostname="0.0.0.0", port=port)` [`email_handler.py:L2383`] starts. The two
  values are the bind host `0.0.0.0` and port `20381`.

**Liveness probe — the socket is actually listening.** Independent TCP connect check:

```bash
$ python3 -c "import socket; s=socket.socket(); print('connect_ex =', s.connect_ex(('127.0.0.1',20381)))"
```
```text
port 20381 connect_ex = 0 (0=open/listening)
```

`connect_ex == 0` means the port accepts connections — the handler is live. (A full end-to-end SMTP
delivery to this port, returning `250 Message accepted for delivery`, is shown in **Q2.3**.)

**Reasoning.** The `Listen for port 20381` + `Start mail controller 0.0.0.0 20381` pair is the handler's
"I am bound and ready" signature; the `connect_ex = 0` TCP probe independently confirms the socket is
open. Together they prove the email handler is up.

### Q1.3 — Job runner (background poll loop)

**Start it:**

```bash
$ CONFIG=example.env PYTHONPATH=. poetry run python job_runner.py
```

**Startup signature + healthy idle behavior.** After the `>>> init logging <<<` banner the runner enters
its loop `if __name__ == "__main__": while True:` [`job_runner.py:L329-L330`]. When **no** jobs are
queued it is **silent** — it fetches ready jobs via `get_jobs_to_run()` [`job_runner.py:L307`], finds
none, and `time.sleep(10)` [`job_runner.py:L347`]. So the healthy **idle** signature is: startup banner,
then no output, re-polling every **10 seconds**.

**Liveness probe — force a real cycle.** To capture an actual poll, a throwaway `Job` row was enqueued
(then deleted). The runner picked it up on its next 10-second poll and logged `LOG.d("Take job %s", job)`
[`job_runner.py:L334`]:

```text
2026-07-01 04:15:06,633 - SL - DEBUG - 42864 - ".../job_runner.py:334" - <module>() -  - Take job <Job 1 blitzy_probe_unknown_job {'probe': True}>
```

The job then transitioned `ready(0) → done(2)` (`JobState` is `ready=0, taken=1, done=2, error=3`
[`app/models.py:L253-L257`]):

```bash
$ docker exec sl-db psql -U myuser -d simplelogin \
    -c "SELECT id,name,state,attempts,taken FROM job WHERE name='blitzy_probe_unknown_job';"
```
```text
 id |           name           | state | attempts | taken
----+--------------------------+-------+----------+-------
  1 | blitzy_probe_unknown_job |     2 |        1 | t
```

**Reasoning.** A `Take job …` line followed by the row advancing to `state=2` (done) proves the runner is
actively consuming work; the silent 10-second re-poll is the healthy idle heartbeat (inference grounded
in `time.sleep(10)` [`job_runner.py:L347`]). (This particular job's name was unknown, which also
demonstrates graceful error handling — see **Q3**.)

---


## Q2 — End-to-end user actions

Three actions were performed against the running stack: **create an account**, **create an alias**, and
**have that alias receive an email**. For each: what was done, what the system did (quoted
HTTP/flash/log/SMTP/DB evidence), and why that evidence proves correct handling.

### Q2.1 — Create a new account

**What was done** — submit the registration form (POST to `/auth/register`) with a temporary email.
The route calls `User.create(email=..., name=..., password=..., referral=...)` [`app/auth/views/register.py:L86`]
→ `send_activation_email(user, next_url)` [`app/auth/views/register.py:L95`] →
`render_template("auth/register_waiting_activation.html")` [`app/auth/views/register.py:L104`].

```bash
$ # POST /auth/register  (csrf_token, email=blitzy-temp-user@gmail.com, password=...)
```

**What the system did:**

1. **HTTP + waiting-activation page.** The response was `HTTP 200` rendering
   `templates/auth/register_waiting_activation.html`, whose literal UI text is the confirmation signal:

   ```text
   POST /auth/register -> HTTP 200
     page contains 'Activation Email Sent': True                              (block title)
     page contains 'An email to validate your email is on its way.': True     (<h1>)
     page contains 'Please check your inbox/spam folder.': True               (<p>)
   ```

2. **Activation email — logged, not sent** (because `NOT_SEND_EMAIL=true`). The web log shows the
   `create user` line and then the activation email being logged by `mail_sender.send()`
   [`app/mail_sender.py:L131`]:

   ```text
   ... ".../app/auth/views/register.py:85" - register() -  - create user blitzy-temp-user@gmail.com
   ... ".../app/mail_sender.py:131" - send() -  - send email with subject 'Just one more step to join SimpleLogin', from '"noreply@sl.local" <noreply@sl.local>' to 'blitzy-temp-user@gmail.com'
   ```

3. **Database row.** A new `users` row exists, correctly **unactivated** (`activated=f`) until the
   activation link is followed:

   ```bash
   $ docker exec sl-db psql -U myuser -d simplelogin \
       -c "SELECT id,email,activated,is_admin,name FROM users WHERE email='blitzy-temp-user@gmail.com';"
   ```
   ```text
    id |           email            | activated | is_admin |            name
   ----+----------------------------+-----------+----------+----------------------------
     3 | blitzy-temp-user@gmail.com | f         | f        | blitzy-temp-user@gmail.com
   ```

**Why this proves correct handling.** The `create user …` log confirms the `User.create` path ran; the
`send email with subject 'Just one more step to join SimpleLogin' …` log confirms the activation email was
generated (and, locally, logged rather than delivered — exactly the `NOT_SEND_EMAIL` behavior); the
waiting-activation page is the user-facing confirmation; and the persisted `users` row with `activated=f`
is the correct state for an account awaiting email verification.

> **Note on activated vs. unactivated.** To exercise alias creation and inbound mail (which require an
> activated account with a verified mailbox), the seeded **`john@wick.com`** account
> (`activated=t`, mailbox `verified=t`) is used below, rather than activating the temporary user.

### Q2.2 — Create an alias

Exercised **two** ways.

**(a) Dashboard random-alias control** (signed in as `john@wick.com`). The dashboard POST with
`form-name=create-random-email` triggers `Alias.create_new_random(user=current_user, scheme=scheme)`
[`app/dashboard/views/index.py:L104`] and then the success flash
`flash(f"Alias {alias.email} has been created", "success")` [`app/dashboard/views/index.py:L111`]:

```text
POST /dashboard/ (create-random-email) -> HTTP 302
Location: http://localhost:7777/dashboard/?highlight_alias_id=13&query=&sort=&filter=
FLASH (rendered): toastr.success("Alias shorty_shower125@sl.local has been created")
```

The web log confirms the create with the exact alias and owner:

```text
... ".../app/dashboard/views/index.py:110" - index() -  - create new random alias <Alias 13 shorty_shower125@sl.local> for user <User 1 John Wick john@wick.com>
```

**(b) REST API** — `POST /api/alias/random/new` (`new_random_alias`) returns **HTTP `201`** with
`jsonify(alias=alias.email, ...)` [`app/api/views/new_random_alias.py:L115-L116`]:

```bash
$ curl -X POST http://localhost:7777/api/alias/random/new -H "Authentication: <api-key>"
```
```text
POST /api/alias/random/new -> HTTP 201
{"alias":"beside_pander301@sl.local","creation_date":"2026-07-01 04:19:10+00:00","creation_timestamp":1782879550,"disable_pgp":false,"email":"beside_pander301@sl.local","enabled":true,"id":14, ...}
```

**Database rows** (note the domain is `sl.local`, from `EMAIL_DOMAIN` [`example.env:L22`]):

```bash
$ docker exec sl-db psql -U myuser -d simplelogin \
    -c "SELECT id,email,user_id,mailbox_id,enabled FROM alias WHERE email LIKE '%@sl.local' ORDER BY id DESC LIMIT 2;"
```
```text
 id |           email           | user_id | mailbox_id | enabled
----+---------------------------+---------+------------+---------
 14 | beside_pander301@sl.local |       1 |          1 | t
 13 | shorty_shower125@sl.local |       1 |          1 | t
```

**Why this proves correct handling.** The flash `Alias shorty_shower125@sl.local has been created`
(dashboard) and the `201` + JSON `alias`/`email` fields (REST) are the two API contracts for "alias
created"; the `create new random alias <Alias 13 …>` log confirms the code path executed; and the new
`alias` rows (`enabled=t`, owned by `user_id=1`, bound to `mailbox_id=1`) are the durable proof. Both
generated addresses are on the `sl.local` domain, as configured.

### Q2.3 — Have that alias receive an email

**What was done** — deliver a message over SMTP to the email handler on port **`20381`**, `MAIL FROM` an
external sender, `RCPT TO` the alias created above (`shorty_shower125@sl.local`, owned by John whose
mailbox `1` is `verified=t`):

```python
# temporary observation script (smtplib) — delivered to 127.0.0.1:20381
import smtplib
from email.mime.text import MIMEText
msg = MIMEText("Hello alias, this is a Blitzy inbound test message.\n")
msg["Subject"] = "Blitzy inbound test"
msg["From"] = "external-sender@example.com"
msg["To"]   = "shorty_shower125@sl.local"
with smtplib.SMTP("127.0.0.1", 20381, timeout=30) as smtp:
    smtp.ehlo("example.com"); smtp.mail("external-sender@example.com")
    smtp.rcpt("shorty_shower125@sl.local")
    print(smtp.data(msg.as_string().encode()))
```

**SMTP server reply** — the handler accepted the message with the exact success code
`E200 = "250 Message accepted for delivery"` [`app/email/status.py:L2`]:

```text
EHLO -> 250
MAIL FROM -> 250 OK
RCPT TO -> 250 OK
DATA (final SMTP reply) -> 250 Message accepted for delivery
```

**Handler log chain** (verbatim; note the single `message_id` `9cf3291f-aa1f-480a-9406-f485e932fddb`
threaded through every line — that is the log format's `message_id` correlation field in action):

```text
... ".../email_handler.py:2343" - _handle() - 9cf3291f-... - New message, mail from external-sender@example.com, rctp tos ['shorty_shower125@sl.local']
... ".../email_handler.py:2202" - handle()  - 9cf3291f-... - Forward phase external-sender@example.com(external-sender@example.com) -> shorty_shower125@sl.local
... ".../email_handler.py:688"  - forward_email_to_mailbox() - 9cf3291f-... - Forward <Contact 2 external-sender@example.com 13> -> <Alias 13 shorty_shower125@sl.local> -> <Mailbox 1 john@wick.com>
... ".../app/mail_sender.py:131" - send() - 9cf3291f-... - send email with subject 'Blitzy inbound test', from '"external-sender at example.com" <external-sender_at_example_com_fukfhtl@sl.local>' to 'shorty_shower125@sl.local'
... ".../email_handler.py:2367" - _handle() - 9cf3291f-... - Finish mail_from external-sender@example.com, rcpt_tos ['shorty_shower125@sl.local'], takes 0.18215370178222656 seconds with return code '250 Message accepted for delivery'<<===
```

Key lines:
- **`New message, mail from … rctp tos …`** [`email_handler.py:L2343`] — note the (sic) spelling
  `rctp tos` and the trailing space, quoted exactly.
- **`Forward <Contact 2 …> -> <Alias 13 shorty_shower125@sl.local> -> <Mailbox 1 john@wick.com>`** is
  `LOG.d("Forward %s -> %s -> %s", contact, alias, mailbox)` [`email_handler.py:L688`] — the
  **contact → alias → mailbox** forwarding chain.
- **`send email with subject 'Blitzy inbound test' …`** [`app/mail_sender.py:L131`] — because
  `NOT_SEND_EMAIL=true`, the forwarded message is **logged, not delivered** to a real inbox. The `From`
  is rewritten to a reverse-alias address (`external-sender_at_example_com_...@sl.local`).
- **`Finish mail_from … takes 0.18215370178222656 seconds with return code '250 Message accepted for delivery'`**
  [`email_handler.py:L2367`] — the measured processing time (**0.182 s**) and the returned SMTP status.

**Database row — how the message was handled.** A new `email_log` row records the forward. `EmailLog` is
`class EmailLog(Base, ModelMixin)` [`app/models.py:L2060`] with the columns `is_reply`
[`app/models.py:L2075`], `blocked` [`app/models.py:L2078`], and `bounced` [`app/models.py:L2082`]:

```bash
$ docker exec sl-db psql -U myuser -d simplelogin \
    -c "SELECT id,user_id,alias_id,is_reply,blocked,bounced FROM email_log ORDER BY id DESC LIMIT 1;"
```
```text
 id | user_id | alias_id | is_reply | blocked | bounced
----+---------+----------+----------+---------+---------
  2 |       1 |       13 | f        | f       | f
```

**Why this proves correct handling.** The SMTP `250 Message accepted for delivery` is the protocol-level
acknowledgement that the handler accepted the message; the `New message … → Forward contact→alias→mailbox
→ Finish … return code '250 …'` log chain shows the message flowed through the pipeline to John's mailbox;
and the `email_log` row with `is_reply=f, blocked=f, bounced=f` is the durable record of a **clean,
normal forward** (not a reply, not blocked, not bounced). Because `NOT_SEND_EMAIL=true`, "the alias
received the email" locally = **this log chain + this `email_log` row**, not a message landing in an
external mailbox.

**What happens when a new account interacts with aliases or tries to receive mail?** A newly-registered
account starts `activated=f` (Q2.1) — until activated it cannot sign in to manage aliases. An **activated**
account (like John) can create aliases (Q2.2) and its aliases forward inbound mail to its verified mailbox
(Q2.3), each action leaving the HTTP/flash/log/DB evidence shown above. If the alias is **disabled**, or
the mailbox is **unverified**, the message is handled but **not** forwarded — see Q3.

---


## Q3 — Background components (auto-start + healthy behavior across scenarios)

### Q3.1 — Do the email handler and job runner come online automatically? → **No.**

Each is an **independent `__main__` entry point** that must be launched on its own; the web app neither
imports nor spawns them. Three independent proofs:

**Proof 1 — by reading.** The web server sources contain **no reference** to the other two components:

```bash
$ grep -n 'email_handler\|job_runner' server.py wsgi.py
```
```text
>>> NONE — web app never imports/spawns email_handler or job_runner
```

**Proof 2 — process tree.** The Gunicorn master's only children are its two workers (not the handler or
runner); the three entry-points are distinct top-level processes:

```text
-- gunicorn master 42160 and descendants:
   child:   42170 .../.venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
   child:   42171 .../.venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
-- the 3 entry-points are distinct top-level processes:
   42160 .../.venv/bin/gunicorn wsgi:app ...
   42585 .../.venv/bin/python email_handler.py
   42864 .../.venv/bin/python job_runner.py
```

**Proof 3 — empirical.** Stopping **only** the handler and runner leaves the web server fully functional,
but nothing then listens on `20381` and inbound SMTP is refused:

```bash
$ kill <email_handler_pid> <job_runner_pid>     # web server left running
$ curl -s -o /dev/null -w "GET /health -> HTTP %{http_code}\n" http://localhost:7777/health
$ python3 -c "import socket;print('20381 connect_ex =', socket.socket().connect_ex(('127.0.0.1',20381)))"
```
```text
GET /health -> HTTP 200, body content: success        # web server UNAFFECTED
port 20381 connect_ex = 111 (0=open, non-zero=closed/refused)
SMTP connect FAILED as expected: ConnectionRefusedError [Errno 111] Connection refused
```

Restarting them independently brings them back online (the handler re-logs its bind, the port reopens):

```text
... ".../email_handler.py:2403" - <module>() -  - Listen for port 20381
... ".../email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
port 20381 connect_ex = 0 (0=open again)
job_runner RUNNING (pid 48428)
```

**Reasoning / production mapping.** Because the web app does not start them, in production these run as
**separate containers** — `sl-app` (web, `7777`), `sl-email` (`python email_handler.py`, `20381`), and
`sl-job-runner` (`python job_runner.py`). The health of the web tier (`/health` = `200`) is therefore
**independent** of whether mail/jobs are being processed; you must verify each process separately.

### Q3.2 — What behavior confirms they function as intended across situations?

**Healthy steady-state.**
- **Job runner:** a steady **10-second** poll cadence — `get_jobs_to_run()` [`job_runner.py:L307`], a
  `Take job %s` line per job [`job_runner.py:L334`], process, commit, `time.sleep(10)` [`job_runner.py:L347`].
  Idle = silent between polls; busy = one `Take job …` line per job that then advances to `state=done`.
- **Email handler:** a persistent bind on `0.0.0.0:20381` and a per-message log chain (`New message …` →
  `Forward … -> … -> …` → `Finish … return code '…'`) correlated by a single `message_id` (Q2.3).

**Failure / edge scenarios (correct handling across situations):**

- **Unknown job name (exercised).** A queued job whose name matches none of the dispatcher's
  `config.JOB_*` cases hits the `else:` branch `LOG.e("Unknown job name %s", job.name)` [`job_runner.py:L304`]
  (dispatch in `process_job` [`job_runner.py:L188`]). Captured:

  ```text
  ... ".../job_runner.py:334" - <module>() -  - Take job <Job 1 blitzy_probe_unknown_job {'probe': True}>
  ... ".../job_runner.py:304" - process_job() -  - Unknown job name blitzy_probe_unknown_job
  ```

  The job is still marked `taken`, logged as an `ERROR`, and advanced to `state=done(2)` (`attempts=1`) —
  i.e. it fails **loudly but gracefully**, without wedging the loop or retrying forever.

- **Disabled alias (exercised).** Sending to a **disabled** alias (`blitzy-disabled-probe@sl.local`,
  `enabled=False`). The handler's `if not alias.enabled or contact.block_forward:` branch
  [`email_handler.py:L596`] logs and records a **blocked** `EmailLog` (`blocked=True`)
  [`email_handler.py:L601`]:

  ```text
  ... ".../email_handler.py:597" - handle_forward() - ... - <Alias 15 blitzy-disabled-probe@sl.local> is disabled, do not forward
  DATA (final SMTP reply) -> 250 Message accepted for delivery
  ```
  ```bash
  $ docker exec sl-db psql -U myuser -d simplelogin \
      -c "SELECT id,alias_id,is_reply,blocked,bounced FROM email_log WHERE alias_id=15;"
  ```
  ```text
   id | alias_id | is_reply | blocked | bounced
  ----+----------+----------+---------+---------
    3 |       15 | f        | t       | f
  ```

  The message is **accepted** (`250`) but **not forwarded**, and recorded with `blocked=t`. By design the
  handler returns a `2xx` (`status.E200`) rather than a `5xx` so the sender does not permanently fail —
  *"by default return 2\*\* instead of 5\*\* to allow user to receive emails again when alias is enabled"*
  — unless the user set `block_behaviour == return_5xx`, in which case it returns `E502`
  [`email_handler.py:L607-L612`].

- **Unverified mailbox (verified by reading, not run).** When forwarding targets an unverified mailbox
  the handler returns `status.E517` [`email_handler.py:L635`], defined as
  `E517 = "550 SL E517 unverified mailbox"` [`app/email/status.py:L53`]. (Related: `E516 = "550 SL E516
  invalid mailbox"` [`app/email/status.py:L52`], `E518 = "550 SL E518 Disabled mailbox"`
  [`app/email/status.py:L54`].)

- **Permanent rejection / SPF downgrade (verified by reading, not run).** Permanent failures return `5xx`
  codes. If a would-be `5xx` bounce has a return-path that fails SPF, `_handle` rewrites it to a `2xx`
  (`status.E216`) to black-hole it instead of bouncing — `if return_status[0] == "5"` and the SPF check is
  `fail`/`soft_fail`, it logs *"Replacing 5XX to 216 status because the return-path failed the spf check"*
  and sets `return_status = status.E216` [`email_handler.py:L2357-L2366`], where
  `E216 = "250 SL E216 Handled spf policy"` [`app/email/status.py:L24`].

**Other independent background processes (context only — not the subject of the question).** SimpleLogin
has additional standalone entry points that likewise do not auto-start with the web app: `cron.py`
(scheduled maintenance tasks, driven by the `crontab.yml` yacron config present at the repo root),
`event_listener.py` (PostgreSQL LISTEN/NOTIFY event processing via `PostgresEventSource`), and
`monitoring.py` (metrics daemon). First-run seeding is `init_app.py` (`load_pgp_public_keys()` +
`add_sl_domains()` [`init_app.py:L71-L73`]).

**Reasoning.** "Functioning as intended" for these background components is not a single OK/response —
it is (a) the **steady cadence / persistent bind** during normal operation, plus (b) the **correct
per-situation signatures** above: a blocked `EmailLog` for a disabled alias, an `Unknown job name` error
with the job still completing, and the documented `E517`/`E502`/`E216` status codes for the mailbox/SPF
edge cases. Observing those signatures across success **and** failure paths is what confirms they work.

---

## Coverage pass (every sub-question mapped to its answer)

| Question | Sub-part | Answered in |
|----------|----------|-------------|
| **Q1** | Web server up & responding? | Q1.1 — `curl /health` → `200`/`success`; gunicorn startup |
| **Q1** | Email handler up? | Q1.2 — `Listen for port 20381` / `Start mail controller 0.0.0.0 20381`; `connect_ex=0` |
| **Q1** | Job runner up? | Q1.3 — `Take job …`; 10-second poll; row → `state=done` |
| **Q1** | Logs/UI confirming sign-in works | Q1.1 — login `302 → /dashboard/`; wrong-pw toastr `Email or password incorrect` |
| **Q1** | Logs/UI confirming alias management works | Q1.1 — `/dashboard/` → `200`, title "Alias \| SimpleLogin", Random/New-custom controls |
| **Q2** | Create a new account | Q2.1 — waiting-activation page; `create user` + activation-email log; `users` row `activated=f` |
| **Q2** | Create an alias | Q2.2 — flash `Alias …@sl.local has been created`; REST `201`; `alias` rows |
| **Q2** | Alias receives an email | Q2.3 — SMTP `250 Message accepted for delivery`; `New message`/`Forward`/`Finish` logs; `email_log` row |
| **Q2** | What happens when a new account interacts with aliases / tries to receive mail | Q2.1–Q2.3 + closing paragraph; disabled/unverified paths in Q3.2 |
| **Q2** | How does the system show it was handled correctly | Every action shows the HTTP/flash/log/SMTP/DB channel(s) |
| **Q3** | Do handler & job runner auto-start in background? | Q3.1 — **No**; three proofs (grep, process tree, empirical stop/restart) |
| **Q3** | Behavior confirming they work across situations | Q3.2 — steady cadence/bind + unknown-job, disabled-alias, `E517`, `E216` signatures |

## Caveats & explicitly unverifiable items

- **`NOT_SEND_EMAIL=true` (local):** activation and forwarded emails are **logged, not delivered**
  [`app/mail_sender.py:L130-L137`]. Claims about "receiving" mail are therefore proven by the log chain +
  `email_log` row, not by an external inbox.
- **Verified by reading, not run:** the **unverified-mailbox** (`E517` [`email_handler.py:L635`]),
  **invalid/disabled-mailbox** (`E516`/`E518`), and **SPF `5xx`→`E216` downgrade**
  [`email_handler.py:L2357-L2366`] paths were confirmed from source, not executed, and are labeled as such
  in Q3.2.
- **Line-number drift:** references were re-confirmed against the current source by grepping the quoted
  literal. Where a `LOG.i(...)`/`LOG.d(...)` call spans two lines, the runtime-reported `lineno` is the
  call line (e.g. `New message …` reports `email_handler.py:2343`; the string literal itself sits on the
  next line). Observed values take precedence over any prior citation.
- **Temporary test data:** the temp account (`blitzy-temp-user@gmail.com`), temp aliases
  (`shorty_shower125@sl.local`, `beside_pander301@sl.local`, `blitzy-disabled-probe@sl.local`), the temp
  `job`/`email_log` rows, and the temp API key created for these observations were **removed** after
  capture; only the standard `flask dummy-data` demo seed (`john@wick.com`) remains. No source file was
  modified.

