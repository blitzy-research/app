# SimpleLogin local run — runtime‑grounded Q&A

This document confirms that a freshly started **local SimpleLogin** instance behaves correctly by **actually running the system and observing its runtime behaviour**, not by reading the code alone. Every factual claim below is backed by (a) the exact command or code that produced it, (b) the **verbatim** output that was observed, and (c) an exact `file:line` citation into the source at branch `app_2cd6ee777f8c` (HEAD commit `2cd6ee77`). Random values (activation code, alias suffix, user id, timestamps, process ids) are specific to this run; the invariants (HTTP statuses, redirect targets, log `file:line`, the 30‑character activation code, the e‑mail canonicalisation) are not.

It answers three questions asked by a first‑time self‑hoster:

- **Q1 — Readiness.** After the system starts, what appears in the **logs or UI** that proves the app is ready to handle **user authentication** and **alias‑based e‑mail** activity?
- **Q2 — New‑user walkthrough.** Walking through the product as a brand‑new user who **registers**, **verifies** their address and **logs in** — what **visible behaviour** confirms each step succeeds and that the user is **forwarded into the dashboard**?
- **Q3 — Behind the scenes.** What indicators show that **background jobs or internal services** support **e‑mail forwarding** and **identity verification**, and what proves at runtime that these pieces are **active and talking to each other**?

No application source file was modified. The only artefact produced is this document. Every temporary user, alias, job, throwaway database and observation script created during the investigation was deleted afterward; the final working tree `git status --porcelain` is empty, and the baseline‑to‑HEAD diff contains only this new document (see the **Cleanup and read‑only verification** section at the end).

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
DB_URI=postgresql://myuser:<redacted>@localhost:5432/simplelogin
MEM_STORE_URI=redis://localhost
FLASK_SECRET=<redacted-local-dev-secret>
```

The loaded `DB_URI` is what Flask‑SQLAlchemy actually binds to: `app.config["SQLALCHEMY_DATABASE_URI"] = DB_URI` at `server.py:146` inside `create_app()` (`server.py:139`), with the identical wiring at `server.py:129` inside `create_light_app()` (the lighter factory the standalone `job_runner.py`, `cron.py` and `email_handler.py` processes use). The DB password is redacted above; it exists only in the out‑of‑repo `/root/sl.env` and is never committed.

**Commands run** (mirroring the documented local path in `CONTRIBUTING.md:88`, `:106`, `:109`, `:212`, `:228`):

```bash
# PostgreSQL + Redis are started as local services (already online in the image)
# Schema (already at head 32f25cbf12f6) and baseline seed:
CONFIG=/root/sl.env /app/venv/bin/alembic upgrade head
CONFIG=/root/sl.env FLASK_APP=server.py /app/venv/bin/flask dummy-data   # seeds john@wick.com / winston@continental.com
# Web app (dev server on http://localhost:7777). WERKZEUG_RUN_MAIN=true only suppresses the
# debug reloader's child fork so the single process is cleanly stoppable — behaviour is identical:
WERKZEUG_RUN_MAIN=true CONFIG=/root/sl.env /app/venv/bin/python server.py
# Standalone services observed for Q3:
CONFIG=/root/sl.env /app/venv/bin/python email_handler.py   # inbound SMTP on :20381
CONFIG=/root/sl.env /app/venv/bin/python job_runner.py      # 10s poll loop
```

**Dependency transparency and required scratch‑environment disclosure.** The runtime is a throwaway scratch environment that lives entirely outside the repository and installs the locked dependency set; the versions actually exercised by the flows in this document match `poetry.lock` exactly — for example `Flask 1.1.2` (the `create_app()` framework at `server.py:139`), `SQLAlchemy 1.3.24`, `aiosmtpd 1.4.2`, `redis 4.6.0`, `bcrypt 3.2.0`. Two substitutions were made **in the scratch environment only** so the stack would build and boot. Both are disclosed here in full, and **neither is a change to the repository's dependencies — `pyproject.toml` and `poetry.lock` are untouched** — and neither touches the register/verify/login/e‑mail flows documented below:

1. **`cbor2 5.2.0 → 5.6.5`** — the locked `cbor2 5.2.0` source distribution fails to build in the sandbox, so the `5.6.5` wheel was substituted. `cbor2` is a WebAuthn/FIDO2 (de)serialization dependency; it is not imported by any of the register/verify/login/e‑mail code paths exercised here.
2. **`pyre2` → a `re2` shim onto Python's stdlib `re`** — `pyre2` (which supplies the compiled `re2` module) is awkward to build in the sandbox, so in the scratch environment the single source line that imports it was shimmed to stdlib `re`. To be exact about the difference between the *repository source* and the *scratch environment*: the **committed repository source** imports the compiled binding — `import re2 as re` at `app/spamassassin_utils.py:8`, used as `re.DOTALL` at `app/spamassassin_utils.py:13` — whereas the scratch environment ran that one module with `import re` instead. `spamassassin_utils.py` participates only in inbound spam scanning and in none of the flows below.

Crucially, the **deliverable repository** (the one this document is committed to) is left byte‑for‑byte unchanged: this Markdown file is the only addition in the baseline‑to‑HEAD diff, and the final working tree `git status --porcelain` is empty.

**Startup banner actually captured** when booting the web app — the plain `print()` lines emitted before logging initialises, followed by the first structured log line:

```text
load config file /root/sl.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/ugrbkhmbpawqctoppijh
Upload files to local dir
>>> init logging <<<
2026-07-01 05:36:50,761 - SL - DEBUG - 164 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

The `load config file /root/sl.env` line is the `print("load config file", config_file)` at `app/config.py:68`; `>>> init logging <<<` is `print(">>> init logging <<<")` at `app/log.py:67`. Everything after that point is emitted by the single **`SL`** logger.

---

## Q1 — Readiness indicators (logs / UI)

> *"After starting the system what should I notice in the logs or UI that shows the app is ready to handle user authentication and alias based email activity?"*

**Short answer.** Four concrete signals: (1) the `>>> init logging <<<` marker plus the single, uniform **`SL`** log‑line format prove the logging subsystem is up; (2) the `/health` endpoint returns `success` with HTTP `200`; (3) the observed unauthenticated `/` route and protected `/dashboard/` route both redirect (`302`) to `/auth/login`, proving the authentication gate is live; and (4) the local alias domain `sl.local` is registered (logged as `Add sl.local to SL domain`), proving the platform is ready to create alias‑based e‑mail.

### 1) The logs show the app booted and the logging subsystem is ready

The very first evidence that the process reached a ready state is the logging‑init marker and the first structured log line (from the banner above):

```text
>>> init logging <<<
2026-07-01 05:36:50,761 - SL - DEBUG - 164 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
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
Set-Cookie: slapp=<redacted session id>; Expires=Wed, 08-Jul-2026 05:36:52 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Wed, 01 Jul 2026 05:36:52 GMT

success
```

`Content-Length: 7` is exactly `len("success")`, and `Server: Werkzeug/1.0.1 Python/3.10.18` confirms the Werkzeug dev server on the pinned interpreter.

That the **authentication gate is live** is proved by routing: an unauthenticated request to `/` or to a protected page is redirected to `/auth/login`. The root route is `@app.route("/", ...)` at `server.py:250`, `def index():` at `server.py:251`, redirecting authenticated users to `dashboard.index` and everyone else to `auth.login` (`server.py:252-255`):

```bash
curl -sS -i http://127.0.0.1:7777/            # unauthenticated root
curl -sS -i http://127.0.0.1:7777/dashboard/  # unauthenticated protected page
```
```text
GET /            -> HTTP/1.0 302 FOUND   Location: http://127.0.0.1:7777/auth/login
GET /dashboard/  -> HTTP/1.0 302 FOUND   Location: http://127.0.0.1:7777/auth/login?next=%2Fdashboard%2F%3F
```

Each such request also produces one SimpleLogin access‑log line from `after_request()` (`server.py:273`); the emitting `LOG.d(` opens at `server.py:284` with format `"%s %s %s %s %s, takes %s"` at `server.py:285`:

```text
2026-07-01 05:36:52,060 - SL - DEBUG - 164 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET / ImmutableMultiDict([]) 302, takes 0.0005950927734375
2026-07-01 05:36:52,068 - SL - DEBUG - 164 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard/ ImmutableMultiDict([]) 302, takes 0.0008509159088134766
```

Note `/health` is **deliberately excluded** from the access log (`not request.path.startswith("/health")` at `server.py:281`) — and indeed the captured server log contained **zero** `GET /health` access lines, confirming that guard. **Reasoning:** a `302 → /auth/login` on protected paths is direct proof the login/authentication machinery is wired and enforcing access; the `after_request` line proves requests are being served and timed.

> Host note: the `Location` above reads `http://127.0.0.1:7777/...` because the probe used `curl` against `127.0.0.1:7777`. In Q2 the same redirects are exercised through Flask’s test client, whose default host is `localhost`, so those show `http://localhost/...`. Same routes, different observed host — nothing more.

### 3) The app is ready for alias‑based e‑mail — `sl.local` is registered

Alias readiness is governed by `add_sl_domains()` at `init_app.py:39`, which for each configured alias domain either logs that it already exists (`init_app.py:42`) or registers it: `LOG.i("Add %s to SL domain", alias_domain)` at `init_app.py:44` followed by `SLDomain.create(domain=alias_domain, use_as_reverse_alias=True)` at `init_app.py:45`. With `EMAIL_DOMAIN=sl.local`, the first‑registration line was captured by pointing an out‑of‑repo config at a throwaway database and invoking `add_sl_domains()` against it:

```bash
PGPASSWORD=<redacted> createdb -h localhost -U myuser simplelogin_probe
CONFIG=/tmp/probe/probe.env /app/venv/bin/alembic upgrade head       # probe.env = sl.env with DB_URI -> simplelogin_probe
CONFIG=/tmp/probe/probe.env /app/venv/bin/python -c "from server import create_light_app
from init_app import add_sl_domains
with create_light_app().app_context(): add_sl_domains()"
```
```text
2026-07-01 05:37:29,253 - SL - INFO - 209 - "/app/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain
```

Because the seeded database already had `sl.local` registered, this genuine first‑run line was captured on a throwaway database (created, migrated with `alembic upgrade head`, observed, then dropped — see cleanup). The **current registered state** in the running database confirms the same fact directly (`SLDomain.__tablename__ = "public_domain"` at `app/models.py:3119`):

```bash
PGPASSWORD=<redacted> psql -h localhost -U myuser -d simplelogin -tA -F'|' \
  -c "SELECT id,domain,use_as_reverse_alias,premium_only FROM public_domain ORDER BY id"
```
```text
1|premium.com|f|t
2|sl.local|t|f
```

`sl.local` is present with `use_as_reverse_alias = t`, precisely what `init_app.py:45` sets. **Reasoning:** aliases can only be created on a registered `SLDomain`; the `Add sl.local to SL domain` log line and the `public_domain` row together prove the local alias domain (`EMAIL_DOMAIN=sl.local`, `example.env:22`) is ready, which is exactly what Q2’s **registration** step relies on when `User.create()` mints the default `…@sl.local` alias (`app/models.py:634-643`).

### 4) UI / endpoint readiness signal

The endpoint‑level readiness signals are the two above: `GET /health → 200 success` (an explicit probe intended for exactly this purpose) and the `302 → /auth/login` redirect that renders the login UI for anonymous visitors. Opening `http://localhost:7777` in a browser therefore lands on the login page — the documented first screen (`CONTRIBUTING.md:109`: “open http://localhost:7777, you should be able to login with `john@wick.com / password`”).

---


## Q2 — New‑user walkthrough (register → verify → login → dashboard)

> *"What happens when a user registers verifies their address and tries to log in. What visible behavior confirms that the system is handling every step correctly and forwarding the user into the dashboard as expected?"*

**Short answer.** A brand‑new user posts to `/auth/register` and gets a `200` “waiting” page; **registration itself already creates the account's default `…@sl.local` newsletter alias** (inside `User.create()`); a 30‑character activation code is stored and an activation e‑mail is composed; clicking the activation link flips `User.activated` from `False` to `True`, auto‑logs the user in, sends a welcome e‑mail **to that already‑created newsletter alias**, and `302`‑redirects to `/dashboard/`; a fresh login with the correct password `302`‑redirects to `/dashboard/` (a wrong password stays on `200` with `Email or password incorrect`); and the authenticated `GET /dashboard/` returns `200`, confirming the user is forwarded in.

This flow was exercised end‑to‑end with a **temporary** user, `blitzy.qna.probe@gmail.com`, driven through the real view functions with Flask’s test client (CSRF disabled for the harness only; it changes none of the observed auth/e‑mail behaviour). The user was deleted afterward (see cleanup). The harness (a temporary script `docker cp`‑ed into the container and removed afterward) drove the flow like this:

```python
# /tmp/probe/q2_flow.py  (run: CONFIG=/root/sl.env PYTHONPATH=/app WTF_CSRF_ENABLED=False \
#                              /app/venv/bin/python /tmp/probe/q2_flow.py ; removed afterward)
from server import create_app
from app.db import Session
from app.models import User, ActivationCode, Alias
app = create_app()
client = app.test_client()                       # STEP 1 & 2 share one session
# ... each step's exact request is shown as a `CMD:` line above its output block ...
```

Every `CMD:` line below is the exact test‑client call that produced the log lines and status immediately under it (captured verbatim; process id `251` throughout).

### E‑mail canonicalisation (applies before anything is stored)

`register()` canonicalises the submitted address at `app/auth/views/register.py:73` (`email = canonicalize_email(form.email.data)`). `canonicalize_email()` at `app/utils.py:78-94` — for `gmail.com`, `protonmail.com`, `proton.me`, `pm.me` — strips any `+`‑suffix (`app/utils.py:88-89`), removes dots (`app/utils.py:93`) and lower/strips (`app/utils.py:94`); other domains are returned unchanged (`app/utils.py:85`). Called directly in the harness:

```python
from app.utils import canonicalize_email
print('canonicalize_email("blitzy.qna.probe@gmail.com") ->', canonicalize_email("blitzy.qna.probe@gmail.com"))
```
```text
canonicalize_email("blitzy.qna.probe@gmail.com") -> "blitzyqnaprobe@gmail.com"
```

So the account is stored and later queried under the canonical `blitzyqnaprobe@gmail.com`. **Reasoning:** knowing this is essential — the DB row does not carry the dotted address, and both `app/auth/views/register.py:73` and `app/auth/views/login.py:42` rely on the same canonicalisation.

### Step 1 — Register

`POST /auth/register` returns `200` and renders `auth/register_waiting_activation.html` (`return render_template("auth/register_waiting_activation.html")` at `app/auth/views/register.py:104`). The producer command, and the observed status plus the page’s three literal strings:

```python
r = client.post("/auth/register",
                data={"email": "blitzy.qna.probe@gmail.com", "password": "<pw>"})
print("REGISTER_STATUS:", r.status_code)
for s in ("Activation Email Sent",
          "An email to validate your email is on its way.",
          "Please check your inbox/spam folder."):
    print(f'Page contains "{s}" ->', s in r.get_data(as_text=True))
```
```text
REGISTER_STATUS: 200
Page contains "Activation Email Sent"                        -> True
Page contains "An email to validate your email is on its way." -> True
Page contains "Please check your inbox/spam folder."          -> True
```

Those strings come from the template: title `Activation Email Sent` at `templates/auth/register_waiting_activation.html:3`, the `<h1>` `An email to validate your email is on its way.` at `templates/auth/register_waiting_activation.html:8`, and `Please check your inbox/spam folder.` at `templates/auth/register_waiting_activation.html:9`.

The server‑side proof, logged verbatim (`LOG.d("create user %s", email)` at `app/auth/views/register.py:85`; the activation e‑mail via `send_email()` logging at `app/email_utils.py:303`; the local short‑circuit at `app/mail_sender.py:131`):

```text
2026-07-01 05:39:16,424 - SL - DEBUG - 251 - "/app/app/auth/views/register.py:85" - register() -  - create user blitzyqnaprobe@gmail.com
2026-07-01 05:39:16,720 - SL - DEBUG - 251 - "/app/app/email_utils.py:303" - send_email() -  - send email to blitzyqnaprobe@gmail.com, subject 'Just one more step to join SimpleLogin'
2026-07-01 05:39:16,721 - SL - DEBUG - 251 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'Just one more step to join SimpleLogin', from '"noreply@sl.local" <noreply@sl.local>' to 'blitzyqnaprobe@gmail.com'
```

Database side effects immediately after register — the user exists but is **not** activated, a **30‑character** activation code was created (`ActivationCode.create(user_id=user.id, code=random_string(30))` at `app/auth/views/register.py:120`; `random_string(30)` at `app/utils.py:41-47` returns 30 lowercase letters), **and the account's default newsletter alias already exists**. Producer queries:

```python
u = User.get_by(email="blitzyqnaprobe@gmail.com")
print(f"DB: User id={u.id}  activated={u.activated}")
ac = ActivationCode.get_by(user_id=u.id)
print(f"DB: ActivationCode = '{ac.code}'  len={len(ac.code)}")
aliases = Alias.filter_by(user_id=u.id).all()
print("DB: alias count AT REGISTER =", len(aliases))
print("DB: alias AT REGISTER =", aliases[0].email, f"(User.newsletter_alias_id={u.newsletter_alias_id})")
```
```text
DB: User id=4  activated=False
DB: ActivationCode = 'ungmfbgpiobdxtviuppylfvngtdoaa'  len=30
DB: alias count AT REGISTER = 1
DB: alias AT REGISTER = simplelogin-newsletter.list612@sl.local  (User.newsletter_alias_id=13)
```

The default `simplelogin-newsletter.…@sl.local` alias is created **here, during registration**, inside `User.create()`: `Alias.create_new(user, prefix="simplelogin-newsletter", …)` at `app/models.py:634-640`, immediately followed by `user.newsletter_alias_id = alias.id` at `app/models.py:643`. It is **not** created during activation. **Reasoning:** the `200` waiting page + the `create user` log + a stored 30‑character `ActivationCode` with `activated=False`, alongside the already‑present `newsletter_alias_id=13` alias, together prove registration was accepted, provisioned the account's default alias, and is pending verification — exactly the intended “check your inbox” state.

### Step 2 — Verify (activate)

Visiting the activation link (`GET /auth/activate?code=…`) returns `302` to the dashboard. `activate()` at `app/auth/views/activate.py:13-69` sets `user.activated = True` at `app/auth/views/activate.py:49`, auto‑logs the user in via `login_user(user)` at `app/auth/views/activate.py:50`, consumes the code (`ActivationCode.delete(...)` at `app/auth/views/activate.py:53`), flashes success at `app/auth/views/activate.py:56`, sends the welcome e‑mail at `app/auth/views/activate.py:58`, and finally `LOG.d("redirect user to dashboard")` at `app/auth/views/activate.py:66` before `return redirect(url_for("dashboard.index"))` at `app/auth/views/activate.py:67`. Note the welcome e‑mail is addressed to the `simplelogin-newsletter.list612@sl.local` alias that **registration already created** — activation does not create it, it merely triggers the welcome message to it:

```python
r = client.get(f"/auth/activate?code={ac.code}")   # same session as register
print("ACTIVATE_STATUS:", r.status_code, "  Location:", r.headers.get("Location"))
```
```text
ACTIVATE_STATUS: 302   Location: http://localhost/dashboard/
2026-07-01 05:39:16,786 - SL - DEBUG - 251 - "/app/app/email_utils.py:303" - send_email() -  - send email to simplelogin-newsletter.list612@sl.local, subject 'Welcome to SimpleLogin'
2026-07-01 05:39:16,787 - SL - DEBUG - 251 - "/app/app/auth/views/activate.py:66" - activate() -  - redirect user to dashboard
```

Database side effects after activation — the flag flipped; the alias count is **unchanged** (still the one alias created at registration, confirming activation does not create it):

```python
Session.expire_all()
u = User.get_by(email="blitzyqnaprobe@gmail.com")
print("DB: User.activated ->", u.activated)
aliases = Alias.filter_by(user_id=u.id).all()
print("DB: alias count AFTER ACTIVATE =", len(aliases))
print("DB: alias AFTER ACTIVATE =", aliases[0].email)
```
```text
DB: User.activated -> True
DB: alias count AFTER ACTIVATE = 1
DB: alias AFTER ACTIVATE = simplelogin-newsletter.list612@sl.local
```

The success flash `Your account has been activated` (`flash("Your account has been activated", "success")` at `app/auth/views/activate.py:56`) is rendered on the page reached after following the `302` into `/dashboard/`. **Reasoning:** the `302 → /dashboard/`, the `activated` flag going `False → True`, and the `Welcome to SimpleLogin` e‑mail sent to the account's newsletter alias are three independent confirmations that verification succeeded and that the account is now a fully provisioned, logged‑in user; the alias count staying at `1` (the same `simplelogin-newsletter.list612@sl.local` created during registration) confirms the default alias is a registration side effect, not an activation one.

### Step 3 — Log in (negative path, then success)

With a **fresh** (unauthenticated) session, a wrong password stays on the login page with a flash, and the correct password redirects into the dashboard. `login()` at `app/auth/views/login.py:21-82` flashes `Email or password incorrect` at `app/auth/views/login.py:49` on bad credentials; on success it calls `after_login(user, next_url)` (`app/auth/views/login.py:72`), which logs `LOG.d("log user %s in", user)` at `app/auth/views/login_utils.py:35` and `LOG.d("redirect user to dashboard")` at `app/auth/views/login_utils.py:44`:

```python
client2 = app.test_client()                       # fresh, unauthenticated session
rw = client2.post("/auth/login", data={"email": "blitzy.qna.probe@gmail.com", "password": "wrong"})
print("LOGIN(wrong) STATUS:", rw.status_code,
      "  contains 'Email or password incorrect' ->",
      "Email or password incorrect" in rw.get_data(as_text=True))
rc = client2.post("/auth/login", data={"email": "blitzy.qna.probe@gmail.com", "password": "<pw>"})
print("LOGIN(correct) STATUS:", rc.status_code, "  Location:", rc.headers.get("Location"))
```
```text
LOGIN(wrong) STATUS: 200   contains 'Email or password incorrect' -> True
LOGIN(correct) STATUS: 302   Location: http://localhost/dashboard/
2026-07-01 05:39:17,281 - SL - DEBUG - 251 - "/app/app/auth/views/login_utils.py:35" - after_login() -  - log user <User 4 blitzy.qna.probe@gmail.com blitzyqnaprobe@gmail.com> in
2026-07-01 05:39:17,281 - SL - DEBUG - 251 - "/app/app/auth/views/login_utils.py:44" - after_login() -  - redirect user to dashboard
```

**Reasoning:** the `200` + `Email or password incorrect` proves the negative path is handled without leaking whether the account exists; the `302 → /dashboard/` plus the `log user … in` line proves credential verification and session establishment succeeded.

### Step 4 — Forwarded into the dashboard

The authenticated `GET /dashboard/` returns `200` (the route `@dashboard_bp.route("/", ...)` at `app/dashboard/views/index.py:55` is guarded by `@login_required` at `app/dashboard/views/index.py:56`; `def index():` at `app/dashboard/views/index.py:67`, rendering `dashboard/index.html` at `app/dashboard/views/index.py:215`). First‑time users get the intro walkthrough, logged at `app/dashboard/views/index.py:172`:

```python
r = client2.get("/dashboard/")                    # still the authenticated session
print("DASHBOARD_STATUS:", r.status_code)
```
```text
DASHBOARD_STATUS: 200
2026-07-01 05:39:17,288 - SL - DEBUG - 251 - "/app/app/dashboard/views/index.py:172" - index() -  - Show intro to <User 4 blitzy.qna.probe@gmail.com blitzyqnaprobe@gmail.com>
```

Contrast this with the **unauthenticated** `GET /dashboard/ → 302 /auth/login` from Q1: same URL, but now — because the session is authenticated — it returns `200` and renders the alias‑management dashboard. **Reasoning:** the authenticated `200` together with the `Show intro to <User 4 …>` line is the definitive confirmation that the system forwarded the freshly verified user into the dashboard, exactly as expected.

---


## Q3 — Behind the scenes (background jobs and internal services)

> *"Are there any indicators that background jobs or internal services are doing their part to support email forwarding or identity verification. What should I expect to observe at runtime that tells me these moving pieces are active and talking to each other properly?"*

**Short answer.** SimpleLogin runs several processes besides the web app: the inbound **SMTP server** (`email_handler.py`, the e‑mail‑forwarding entrypoint), the **job runner** (`job_runner.py`, a 10‑second poll loop over the Postgres `job` table), the **scheduled‑task runner** (`cron.py`, driven by `crontab.yml`), and the **event listener** (`event_listener.py`) that consumes a Postgres `NOTIFY`/`LISTEN` channel. At runtime you can observe each starting and doing work. They all read/write the **same PostgreSQL** database — which is what proves they are active and communicating; **Redis** is used by the **web app only**, for its session store and rate‑limit counters (the standalone processes boot with a DB‑only app context and never initialize Redis).

### 1) Which internal services support the flow

| Service | Entrypoint | Role in the flow | App context | Boot / liveness citation |
|---|---|---|---|---|
| Web app | `server.py:139` `create_app()` | Serves register/activate/login/dashboard; writes Postgres; **only process that initializes Redis** | `create_app()` (DB + Redis) | banner + `/health` (Q1) |
| Inbound SMTP | `email_handler.py:2381` `main()` | Receives mail and **forwards** alias → mailbox | `create_light_app()` DB‑only (`email_handler.py:2352`) | `email_handler.py:2386`, `email_handler.py:2403` (below) |
| Job runner | `job_runner.py:330-347` | Executes async/onboarding jobs from the `job` table | `create_light_app()` DB‑only (`job_runner.py:332`) | `job_runner.py:334`, `job_runner.py:304` (below) |
| Scheduled tasks | `cron.py:1263` | Periodic maintenance (stats, cleanups) per `crontab.yml` | `create_light_app()` DB‑only (`cron.py:1273`) | `cron.py:1263` (below) |
| Event listener | `event_listener.py:35` | Consumes Postgres sync events (`LISTEN`) | own `PostgresEventSource`, DB‑only | `event_listener.py:15-17`, `event_listener.py:35` (below) |

**Identity‑verification support** is carried by the web app + e‑mail composition (the activation e‑mail in Q2), while **e‑mail forwarding** is the job of `email_handler.py`. The processes above all read/write the **same PostgreSQL**: the web app's DB URI is wired at `server.py:146` (`app.config["SQLALCHEMY_DATABASE_URI"] = DB_URI`), and the standalone processes reuse the identical wiring inside `create_light_app()` at `server.py:129`. **Redis is scoped to the web app only** — `initialize_redis_services(app, MEM_STORE_URI)` is called exclusively from `create_app()` at `server.py:165` (with `MEM_STORE_URI=redis://localhost`), and that function installs the Redis‑backed session store and rate‑limit storage at `app/redis_services.py:9-14`. The standalone processes (`email_handler.py`, `job_runner.py`, `cron.py`) boot via `create_light_app()` at `server.py:127-136`, which wires **only** the database and never touches Redis; `event_listener.py` uses neither. So Redis liveness is a **web‑app** signal, evidenced directly (`redis-cli` against the local instance at `MEM_STORE_URI=redis://localhost`):

```bash
redis-cli PING
```
```text
PONG
```

Together with the `Set-Cookie: slapp=…` header in the `/health` response (Q1), the `PONG` confirms the web app's Redis session/rate‑limit layer is up. Every DB side effect in Q1/Q2 (and the job‑row transition below) is the corresponding proof of the shared **PostgreSQL** link that the standalone processes also use.

### 2) Indicator that the e‑mail‑forwarding service is active

Booting the inbound SMTP server shows it binds and starts its controller. `email_handler.py` logs `LOG.i("Listen for port %s", args.port)` at `email_handler.py:2403` (default `20381` at `email_handler.py:2399`) and, inside `main()`, `LOG.d("Start mail controller %s %s", controller.hostname, controller.port)` at `email_handler.py:2386` (the controller itself is `Controller(MailHandler(), hostname="0.0.0.0", port=port)` at `email_handler.py:2383`). It was booted under a short `timeout` so it stops on its own and leaves nothing running:

```bash
timeout 8 env CONFIG=/root/sl.env /app/venv/bin/python email_handler.py 2>&1 | grep -E "Listen for port|Start mail controller"
```
```text
2026-07-01 05:41:21,937 - SL - INFO - 361 - "/app/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-01 05:41:21,939 - SL - DEBUG - 361 - "/app/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

The actual per‑message forwarding is logged by `LOG.d("Forward %s -> %s -> %s", contact, alias, mailbox)` at `email_handler.py:688`. That line only fires when a real inbound message is delivered to an alias, which is outside the register/verify/login flow (it needs an inbound SMTP delivery); it is cited here as the code path, and what was **observed at runtime** is that the forwarding service **is up and listening on `0.0.0.0:20381`**. **Reasoning:** `Listen for port 20381` + `Start mail controller 0.0.0.0 20381` prove the alias‑forwarding entrypoint is active and bound; the `email_handler.py:688` reference names exactly where a forward would be logged.

### 3) Indicator that background jobs run (and that they poll Postgres)

The job runner is a poll loop: `while True:` at `job_runner.py:330`, taking each due job (`LOG.d("Take job %s", job)` at `job_runner.py:334`), marking it taken, calling `process_job(job)` at `job_runner.py:342`, marking it `done` at `job_runner.py:344`, then `time.sleep(10)` at `job_runner.py:347` (the ≈10‑second cadence). Jobs come from the Postgres `job` table via `get_jobs_to_run()` at `job_runner.py:307-326`.

To observe it safely, a throwaway sentinel job with an unknown name was enqueued, then one poll cycle was run (the sentinel deliberately hits the “unknown job” branch so it has no side effects — `LOG.e("Unknown job name %s", job.name)` at `job_runner.py:304`). The producer steps — enqueue, inspect the row, run exactly one cycle under `timeout`, re‑inspect:

```python
# enqueue a throwaway sentinel (removed afterward); run inside create_light_app().app_context()
from app.models import Job
Job.create(name="blitzy-observation-noop", payload={"probe": True}, commit=True)
```
```bash
# inspect the row BEFORE the cycle
PGPASSWORD=<redacted> psql -h localhost -U myuser -d simplelogin -tA -F'|' \
  -c "SELECT id,name,state,taken,attempts FROM job ORDER BY id"
# run one poll pass, then self-terminate (the 10s sleep guarantees a single pass)
timeout 15 env CONFIG=/root/sl.env /app/venv/bin/python job_runner.py 2>&1 | grep -E "Take job|Unknown job name"
# inspect the row AFTER the cycle
PGPASSWORD=<redacted> psql -h localhost -U myuser -d simplelogin -tA -F'|' \
  -c "SELECT id,name,state,taken,attempts FROM job ORDER BY id"
```
```text
before:  1|blitzy-observation-noop|0|f|0

2026-07-01 05:40:50,005 - SL - DEBUG - 316 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 1 blitzy-observation-noop {'probe': True}>
2026-07-01 05:40:50,009 - SL - ERROR - 316 - "/app/job_runner.py:304" - process_job() -  - Unknown job name blitzy-observation-noop

after:   1|blitzy-observation-noop|2|t|1
```

The state literally moved `ready(0) → done(2)` with `taken=true`, `attempts=1` (`JobState.ready=0`, `taken=1`, `done=2` at `app/models.py:254-256`). **Reasoning:** the runner **took** a row from the Postgres `job` table, **processed** it and **closed** it on its poll loop — proof the background‑job machinery (which is what dispatches onboarding e‑mails and other async work in `process_job()` at `job_runner.py:188`) is active and talking to Postgres. The sentinel job was deleted afterward.

The scheduled‑task runner starts cleanly too — `LOG.d("Start running cronjob")` at `cron.py:1263` (dispatched by `-j/--job` and scheduled by `crontab.yml`). Because that log line is emitted before the argparse dispatch, booting it with a non‑matching job name is side‑effect‑free (no dispatch branch matches) yet still proves the entrypoint runs:

```bash
CONFIG=/root/sl.env /app/venv/bin/python cron.py -j blitzy-observation-noop 2>&1 | grep "Start running cronjob"
```
```text
2026-07-01 05:41:20,265 - SL - DEBUG - 342 - "/app/cron.py:1263" - <module>() -  - Start running cronjob
```

### 4) The event bus, and how to tell the pieces are talking

SimpleLogin has **two distinct** event mechanisms; distinguishing them is important:

- **Postgres sync‑event bus (LISTEN/NOTIFY).** `app/events/event_dispatcher.py` defines `NOTIFICATION_CHANNEL = "simplelogin_sync_events"` at `app/events/event_dispatcher.py:14`; `PostgresDispatcher.send()` issues `Session.execute(f"NOTIFY {NOTIFICATION_CHANNEL}, '{instance.id}';")` at `app/events/event_dispatcher.py:26`, and `event_listener.py` is the consumer that `LISTEN`s on that channel (its `Mode` enum `LISTENER`/`DEAD_LETTER` at `event_listener.py:15-17`, `PostgresEventSource(EVENT_LISTENER_DB_URI)` at `event_listener.py:35`). This line was captured by the **same Q2 test‑client harness** (process id `251`): during registration, `EventDispatcher.send_event()` (`app/events/event_dispatcher.py:49`) was reached, but locally it short‑circuits at `app/events/event_dispatcher.py:62` because no webhook is configured:

  ```text
  2026-07-01 05:39:16,696 - SL - INFO - 251 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
  ```

  That line (message string at `app/events/event_dispatcher.py:63`) is the expected, healthy local behaviour: the sync‑event path is wired and invoked, but it is quiet because no `EVENT_WEBHOOK` is set — so no partner‑sync `NOTIFY` is emitted in this local setup.

- **NewRelic custom events (analytics).** Separately, `app/events/auth_event.py` records product analytics: `class LoginEvent` at `app/events/auth_event.py:6` calls `newrelic.agent.record_custom_event("LoginEvent", …)` at `app/events/auth_event.py:23`, and `class RegisterEvent` at `app/events/auth_event.py:28` calls `newrelic.agent.record_custom_event("RegisterEvent", …)` at `app/events/auth_event.py:45`. These are **not** the Postgres sync‑event system; they are telemetry that is effectively a no‑op locally (no NewRelic license configured).

Finally, the reason the activation and welcome e‑mails are observable **as log lines** at all is the local short‑circuit: with `NOT_SEND_EMAIL=true`, `app/mail_sender.py:130` takes the `if config.NOT_SEND_EMAIL:` branch, logs the subject/from/to via `LOG.d(` at `app/mail_sender.py:131` and `return True` at `app/mail_sender.py:137` without contacting a real relay. That is why every e‑mail in Q2/Q3 appears as an `app/mail_sender.py:131` line rather than being delivered.

**Reasoning (all pieces active and communicating):** the web app writes Postgres and — uniquely among these processes — uses Redis for its sessions/rate‑limits; `email_handler.py` binds port `20381`; `job_runner.py` reads and closes a Postgres `job` row on its 10‑second loop; `cron.py` starts its runner; and the `simplelogin_sync_events` `NOTIFY`/`LISTEN` bus is wired (with the local “not sending events” log explaining its silence). Because these separate processes all operate against the **same PostgreSQL** (with the web app additionally backed by Redis), observing each one act against that shared database is exactly what confirms the moving parts are alive and talking to one another.

---


## Coverage pass

Re‑reading each question and confirming every sub‑part is answered with observed evidence:

**Q1 — “what should I notice in the logs or UI that shows the app is ready to handle user authentication and alias based email activity?”**

- [x] *Logs show it booted / logging ready* — `>>> init logging <<<` (`app/log.py:67`) + the uniform `SL` format (`app/log.py:12-15`). → Q1 §1.
- [x] *Ready for user authentication* — `GET /health → 200 success` (`server.py:215`) and unauthenticated `GET / , /dashboard/ → 302 /auth/login` (`server.py:250-255`), plus the `after_request` access log at `server.py:284`. → Q1 §2.
- [x] *Ready for alias‑based e‑mail* — `Add sl.local to SL domain` (`init_app.py:44`) and the `public_domain` row for `sl.local`. → Q1 §3.
- [x] *UI/endpoint readiness signal* — `/health` probe and the `302 → /auth/login` login screen. → Q1 §4.

**Q2 — “What happens when a user registers verifies their address and tries to log in. What visible behavior confirms that the system is handling every step correctly and forwarding the user into the dashboard as expected?”**

- [x] *Register* — `POST /auth/register → 200`, waiting page text (`templates/auth/register_waiting_activation.html:3,8,9`), `create user` (`app/auth/views/register.py:85`), 30‑char `ActivationCode` (`app/auth/views/register.py:120`), `activated=False`, and the account's **default newsletter alias `…@sl.local` created here** in `User.create()` (`app/models.py:634-643`). → Q2 Step 1.
- [x] *Verify* — `GET /auth/activate → 302 /dashboard/`, `activated False→True` (`app/auth/views/activate.py:49`), welcome e‑mail sent to the **registration‑created** newsletter alias (`app/auth/views/activate.py:58`), success flash (`app/auth/views/activate.py:56`); the alias count stays at 1 (activation does not create the alias). → Q2 Step 2.
- [x] *Log in* — wrong password `200` + `Email or password incorrect` (`app/auth/views/login.py:49`); correct password `302 /dashboard/` + `log user … in` (`app/auth/views/login_utils.py:35`). → Q2 Step 3.
- [x] *Visible behaviour confirming each step* — HTTP statuses, redirect `Location`s, flash/page text and DB side effects quoted at every step. → Q2 Steps 1–4.
- [x] *Forwarded into the dashboard* — authenticated `GET /dashboard/ → 200` + `Show intro to <User 4 …>` (`app/dashboard/views/index.py:172`), contrasted with the unauthenticated `302`. → Q2 Step 4.

**Q3 — “Are there any indicators that background jobs or internal services are doing their part to support email forwarding or identity verification. What should I expect to observe at runtime that tells me these moving pieces are active and talking to each other properly?”**

- [x] *Which background jobs / internal services* — web app, `email_handler.py`, `job_runner.py`, `cron.py`, `event_listener.py` (table + citations). → Q3 §1.
- [x] *Support for e‑mail forwarding* — `Listen for port 20381` + `Start mail controller 0.0.0.0 20381` (`email_handler.py:2403`, `email_handler.py:2386`); forward path at `email_handler.py:688`. → Q3 §2.
- [x] *Support for identity verification* — the activation/welcome e‑mails composed during the flow (`app/email_utils.py:303`) made observable by the `NOT_SEND_EMAIL` short‑circuit (`app/mail_sender.py:131`); job runner backs async/onboarding work. → Q3 §3–§4.
- [x] *Proof the pieces are active and communicating* — job `ready(0)→done(2)` on the Postgres `job` table (`job_runner.py:334`, `job_runner.py:304`), `Start running cronjob` (`cron.py:1263`), the `simplelogin_sync_events` NOTIFY/LISTEN bus with the local `Not sending events…` log (`app/events/event_dispatcher.py:62`), all against the shared **PostgreSQL** (with Redis a web‑app‑only concern, confirmed by `redis-cli PING → PONG`). Postgres sync‑events explicitly distinguished from NewRelic `LoginEvent`/`RegisterEvent` (`app/events/auth_event.py:23,45`). → Q3 §4.

**Explicitly noted as not runtime‑exercised (stated rather than asserted):** a full inbound alias‑forward that would emit `Forward … -> … -> …` (`email_handler.py:688`) was not triggered — it requires delivering an inbound message to the running SMTP listener, which is outside the register/verify/login scenario. What was verified at runtime is that the SMTP forwarding service starts and listens on `0.0.0.0:20381`. Likewise the Postgres `NOTIFY` payload was not emitted locally because no `EVENT_WEBHOOK` is configured (the app logged exactly that, `app/events/event_dispatcher.py:62`).

---

## Cleanup and read‑only verification

All temporary artefacts created for the investigation were removed, restoring the seeded baseline and leaving the repository byte‑for‑byte unchanged:

- Temporary user `blitzy.qna.probe@gmail.com` (id 4) deleted via `User.delete(user.id, commit=True)`, which cascades its registration‑created `simplelogin-newsletter.list612@sl.local` alias (`newsletter_alias_id=13`).
- Sentinel `Job` (`blitzy-observation-noop`) deleted; the `job` table returned to 0 rows.
- Throwaway database `simplelogin_probe` (used only to capture the first‑run `Add sl.local to SL domain` line) dropped; the out‑of‑repo probe config removed.
- All temporary observation scripts and captured‑output files (kept outside the repository, e.g. under `/tmp`) removed. The out‑of‑repo `CONFIG` file, the virtualenv, and the transient `GNUPGHOME` directories live entirely outside the repository.
- Database restored to the `flask dummy-data` baseline: `john@wick.com` and `winston@continental.com` only.
- After committing, the final working tree `git status --porcelain` is empty, and the baseline‑to‑HEAD diff (`git diff --name-status 2cd6ee77..HEAD`) contains only this new document `blitzy/documentation/app_2cd6ee777f8c.md`; no existing source file was modified, added or deleted (`*.pyc` is ignored per `.gitignore`).
