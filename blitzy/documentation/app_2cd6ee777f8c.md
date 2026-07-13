# SimpleLogin — Running It Locally and Confirming the Three Core Components Work

> **Document type:** Run-first, evidence-backed Q&A runtime investigation.
> **Subject:** [SimpleLogin](https://github.com/simple-login/app) — a Python/Flask email-aliasing platform.
> **Methodology:** Every behavioral claim below was produced by **actually building, running, and driving** the software through its **real entry points** (an HTTP request to the web app, an SMTP delivery to the email handler, a genuinely enqueued `Job` drained by the job runner) and is accompanied by the **actual, unedited captured output** plus the command that produced it. Reading the source is used only to *locate and explain* each signal. **Any statement that was not directly observed at runtime is explicitly labeled `[INFERRED]`; any value obtained off the canonical path is labeled `[NON-CANONICAL]`.**

---

## 1. Summary

SimpleLogin lets a user hide their real email address behind **aliases**. Mail sent to an alias is forwarded to the user's real mailbox; the user can reply through the alias without revealing their address. The platform is a Python/Flask monorepo, and the three runtime components a newcomer asks about map **one-to-one** onto three root-level entry points:

| Component | Entry point | Role | Default port |
|-----------|-------------|------|--------------|
| **Web server** | `server.py` (prod: `wsgi.py` under Gunicorn) | Serves the dashboard UI, auth, OAuth/API clients | **7777** |
| **Email handler** | `email_handler.py` | `aiosmtpd` SMTP server implementing alias *forwarding* and *replying* | **20381** |
| **Job runner** | `job_runner.py` | Polls the `job` table and drains asynchronous work (onboarding emails, account deletion, batch import, …) | *(none — no listening socket)* |

This document answers three question clusters:

- **Q1 — Are the three components up?** How a reader confirms, from logs / HTTP responses / the dashboard, that the web server, email handler, and job runner are live and that users can sign in and manage aliases. → [§4](#4-q1--are-the-three-components-up-liveness--confirmation-signals)
- **Q2 — Core user actions & correct-handling evidence.** Creating an account, creating an alias, and having that alias receive an email — with the before / intermediate / after state at each boundary. → [§5](#5-q2--core-user-actions--correct-handling-evidence)
- **Q3 — Background auto-online behavior.** Whether the email handler and job runner come online in the background and keep working, plus their secondary/edge behavior (unknown jobs, retry accounting, reply/bounce SMTP codes, alias auto-creation). → [§6](#6-q3--background-auto-online-behavior)

A short orientation to the log format the reader will see everywhere is in [§3](#3-how-to-read-the-logs-the-sl-log-format). A coverage-pass table ([§7](#7-coverage-pass)), an observed-vs-inferred ledger ([§8](#8-observed-vs-inferred-ledger)), and a cleanup confirmation ([§9](#9-cleanup-confirmation)) close the document.

---

## 2. How to run it locally (Environment)

### 2.1 Canonical environment

All observations were taken against the **default, canonical configuration** in the user-provided Docker image:

- **Image:** `andrewparkscaleai/coding-agent:simple-login__app__2cd6ee777f8c2d3531559588bcfb18627ffb5d2c` (from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`).
- **Runtime:** Python **3.10** (`Dockerfile:L8`); the front-end asset build stage uses Node **10.17.0** (`Dockerfile:L2`). A pre-built virtualenv lives at `/app/venv`.
- **Datastores:** PostgreSQL **13** (canonical local host port **15432** → container `5432`, per `scripts/reset_local_db.sh` and `CONTRIBUTING.md:L100`) and Redis on **6379** (sessions + rate limiting, `CONTRIBUTING.md:L65`).
- **Repository branch:** `app_2cd6ee777f8c` (HEAD `2cd6ee77`).

The container was already provisioned with two co-operating containers on a Docker network `sl-net`:

```
$ docker ps --format '{{.Names}}\t{{.Image}}\t{{.Status}}'
sl-app      ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0   Up About an hour
sl-postgres postgres:13                                                     Up About an hour
```

`sl-app` publishes `7777` (web) and `20381` (SMTP) to the host; `sl-postgres` publishes `15432→5432`. Redis runs inside `sl-app` (`redis-server --daemonize yes`, `redis-cli ping` → `PONG`).

### 2.2 The `.env`

Configuration is environment-variable driven via `.env` (copied from `example.env`) and `app/config.py`. The effective, default `.env` inside `sl-app` includes:

```
URL=http://localhost:7777
NOT_SEND_EMAIL=true
EMAIL_DOMAIN=sl.local
DB_URI=postgresql://myuser:mypassword@sl-postgres:5432/simplelogin
```

> **Gotcha — `NOT_SEND_EMAIL` is presence-based.** `app/config.py:L91` reads `NOT_SEND_EMAIL = "NOT_SEND_EMAIL" in os.environ`. The *value* is ignored: even `NOT_SEND_EMAIL=false` would still disable outbound delivery because the key is *present*. To actually re-enable sending you must **comment the line out entirely**. This matters for the inbound-email test in [§5.4](#54-c4--inbound-email-to-the-alias-the-forward-path).

### 2.3 Bring-up sequence (migrate + seed)

```bash
# inside sl-app, from /app
./venv/bin/alembic upgrade head        # 255 migrations -> head 32f25cbf12f6 (77 tables)
FLASK_APP=wsgi ./venv/bin/flask dummy-data
```

`flask dummy-data` is the CLI command defined at `server.py:L490` which logs `LOG.w("reset db, add fake data")` (`server.py:L493`) and runs `app/fake_data.py:fake_data()` (`app/fake_data.py:L40`). It provisions:

- login account **`john@wick.com` / `password`**, created **already `activated=True`** (`app/fake_data.py:L45`, `L47`, `L48`);
- a second account `winston@continental.com`;
- **11 aliases** created via `Alias.create_new_random(user)` (`app/fake_data.py:L70`) — so alias local-parts are **random**, *not* a hard-coded `e1@sl.local`. (`e1@sl.local` in `CONTRIBUTING.md:L218` is only an illustrative `swaks` target; it *does* happen to exist here as alias id 5, but the local-part is not guaranteed.)
- the seed itself also creates **one `Contact`** (`app/fake_data.py:L86`) and **one `EmailLog`** (`app/fake_data.py:L93`), and **zero `Job` rows**. This is the true post-seed baseline — important for the cleanup check in [§9](#9-cleanup-confirmation).

Observed seed state (queried with `psql`):

```
$ PGPASSWORD=mypassword psql -h sl-postgres -U myuser -d simplelogin -c \
  "SELECT id,email,activated FROM users ORDER BY id;"
 id |          email           | activated
----+--------------------------+-----------
  1 | john@wick.com            | t
  2 | winston@continental.com  | t
```

### 2.4 Starting the three components (exact commands)

The three processes are started through their **real entry points**. (In the container, backgrounding with plain `&`/`nohup` dies on `docker exec` teardown; the durable technique is `setsid` + a small on-disk launcher script per process — an environmental detail, not part of the app.)

```bash
# 1) Web server — the Dockerfile CMD form (Gunicorn), Dockerfile:L47
./venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
#    (equivalently, the dev entry `python3 server.py` -> local_main() -> app.run(debug=True, port=7777), server.py:L572,L588)

# 2) Email handler — CONTRIBUTING.md:L212
./venv/bin/python email_handler.py

# 3) Job runner — CONTRIBUTING.md:L228
./venv/bin/python job_runner.py
```

`wsgi.py` is a one-liner (`from server import create_app; app = create_app()`), so Gunicorn and the dev server both build the same Flask app via `create_app()` (`server.py:L139`).

---

## 3. How to read the logs: the `SL` log format

Every application log line below is emitted by a single centralized logger. Understanding its format makes each subsequent line self-explanatory.

- **Format string** (`app/log.py:L12-L15`):
  ```
  %(asctime)s - %(name)s - %(levelname)s - %(process)d - "%(pathname)s:%(lineno)d" - %(funcName)s() - %(message_id)s - %(message)s
  ```
- **Logger name** is `"SL"` (`app/log.py:L79`, `LOG = _get_logger("SL")`).
- **Level shortcuts** (`app/log.py:L74-L77`): `LOG.d` → `debug`, `LOG.i` → `info`, `LOG.w` → `warning`, **`LOG.e` → `logging.Logger.exception`** (so `LOG.e` lines are emitted at **ERROR** level and append a traceback tail — even a bare `NoneType: None` when called outside an `except` block).
- **Import-time banner** `>>> init logging <<<` is printed once per process (`app/log.py:L67`).
- **werkzeug's default HTTP access log is disabled** (`app/log.py:L70-L71`). Consequently the web server produces **no** default per-request access line; per-request visibility comes from the app's own `after_request` hook (`server.py:L284-L292`).
- The **`message_id`** field (the second-to-last token) is minted per inbound email and threaded through the whole email lifecycle (`app/log.py:L18-L37`, `set_message_id` at `app/log.py:L22`), so a single email can be followed end-to-end by matching that id.

A concrete line, annotated:

```
2026-07-13 16:43:01,118 - SL - DEBUG - 1829 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET / ImmutableMultiDict([]) 302, takes 0.0002696514129638672
└─ asctime ────────────┘   │    │       │       └ pathname:lineno ─┘   └ funcName ┘ ↑           └──────────────── message ─────────────────────────────┘
                          name level  process(pid)                                  (empty message_id — not an email)
```

---

## 4. Q1 — Are the three components up? (liveness & confirmation signals)

Each component is started through its real entry point, then probed for its **definitive** liveness signal.

### 4.1 Web server (`server.py`, port 7777)

**Start it** (Gunicorn form). Captured startup output:

```
[2026-07-13 16:42:39 +0000] [1827] [INFO] Starting gunicorn 20.0.4
[2026-07-13 16:42:39 +0000] [1827] [INFO] Listening at: http://0.0.0.0:7777 (1827)
[2026-07-13 16:42:39 +0000] [1827] [INFO] Using worker: sync
[2026-07-13 16:42:39 +0000] [1828] [INFO] Booting worker with pid: 1828
[2026-07-13 16:42:39 +0000] [1829] [INFO] Booting worker with pid: 1829
>>> init logging <<<
2026-07-13 16:42:40,xxx - SL - DEBUG - 1828 - "/app/app/utils.py:17" - <module>() -  - load words file
```

The Gunicorn `Listening at: http://0.0.0.0:7777` line and the per-worker `>>> init logging <<<` banner (`app/log.py:L67`) confirm the process booted.

**Definitive liveness probe — `GET /health`.** The handler `healthcheck()` (`server.py:L213-L215`) returns the literal body `success` with HTTP `200`:

```
$ curl -i http://127.0.0.1:7777/health
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Mon, 13 Jul 2026 16:43:00 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 7

success
```

> **Note (worth stating to a newcomer):** `/health` is **deliberately excluded from request logging and profiling** (`server.py:L195`, `server.py:L281`), so it produces **no `SL` log line**. The definitive "web is up" signal is therefore the **HTTP `200 success` response itself**, not a log entry.

**The login page is served.** An unauthenticated `GET /` returns a `302` redirect to `/auth/login` (index view, `server.py:L250-L255`):

```
$ curl -i http://127.0.0.1:7777/
HTTP/1.1 302 FOUND
Location: http://127.0.0.1:7777/auth/login
Content-Type: text/html; charset=utf-8
```

**Per-request `SL` visibility.** Because werkzeug's access log is off, the app's own `after_request` hook (`server.py:L284-L292`, `LOG.d("%s %s %s %s %s, takes %s", …)`) is what shows each non-excluded request. The `GET /` above produced exactly this line (and `/health` produced none):

```
2026-07-13 16:43:01,118 - SL - DEBUG - 1829 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET / ImmutableMultiDict([]) 302, takes 0.0002696514129638672
```

### 4.2 Email handler (`email_handler.py`, SMTP 20381)

**Start it** (`./venv/bin/python email_handler.py`). The `__main__` block parses `--port` (default **20381**, `email_handler.py:L2399`) and logs `LOG.i("Listen for port %s", …)` (`email_handler.py:L2403`); `main()` (`email_handler.py:L2381`) builds `Controller(MailHandler(), hostname="0.0.0.0", port=port)`, calls `controller.start()` (`L2385`), and logs `LOG.d("Start mail controller %s %s", …)` (`L2386`), then holds the process open with `while True: time.sleep(2)` (`L2392-L2393`). Captured, verbatim:

```
>>> init logging <<<
2026-07-13 16:44:00,048 - SL - INFO  - 1917 - "/app/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-13 16:44:00,050 - SL - DEBUG - 1917 - "/app/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

**Confirm it is actually listening on `0.0.0.0:20381`.** (The image lacks `ss`/`nc`/`lsof`, so a Python `socket.connect_ex` probe was used — the canonical "is the port accepting connections" check.)

```
$ python3 -c "import socket;s=socket.socket();print('OPEN' if s.connect_ex(('127.0.0.1',20381))==0 else 'CLOSED')"
OPEN
```

Both the `Start mail controller 0.0.0.0 20381` log line **and** the open TCP port confirm the handler is up.

### 4.3 Job runner (`job_runner.py`, no port)

**Start it** (`./venv/bin/python job_runner.py`). Its `__main__` block is an infinite loop (`job_runner.py:L330`) that, per iteration, opens `create_light_app().app_context()` (`L332`), fetches work via `get_jobs_to_run()` (`L333`), logs `LOG.d("Take job %s", job)` (`L334`) **only when work exists**, and sleeps 10 seconds (`time.sleep(10)`, `L347`). With no jobs it idles quietly:

```
>>> init logging <<<
2026-07-13 16:44:37,xxx - SL - DEBUG - 1948 - "/app/app/utils.py:17" - <module>() -  - load words file
   (… no "Take job" lines while the job table is empty — by design …)
```

**Observing the poll cadence (Rule: observe true timing, ≥2 cycles).** To make the loop *do* something, three genuine `Job` rows (`name=onboarding-1`, `payload={"user_id": 1}`, `state=ready`) were enqueued. The runner picked them up on consecutive polls; the `Take job` timestamps are:

```
2026-07-13 16:47:48,862 - SL - DEBUG - 1948 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 2 onboarding-1 {'user_id': 1}>
2026-07-13 16:47:58,901 - SL - DEBUG - 1948 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 3 onboarding-1 {'user_id': 1}>
2026-07-13 16:48:08,935 - SL - DEBUG - 1948 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 4 onboarding-1 {'user_id': 1}>
```

Inter-poll deltas: **10.039 s** and **10.034 s** → the **10-second poll cadence is stable across two consecutive cycles**, matching `time.sleep(10)` (`job_runner.py:L347`). Each job also transitioned `ready → taken → done` (see [§6.2](#62-a-jobs-lifecycle-ready--taken--done-with-attempts-accounting)); the handler for `onboarding-1` logged:

```
2026-07-13 16:47:48,xxx - SL - DEBUG - 1948 - "/app/job_runner.py:196" - process_job() -  - send onboarding send-from-alias email to user <User 1 John Wick john@wick.com>
```

### 4.4 UI confirmation — sign-in and alias management (Q1 dashboard signals)

With the web server up, a **real HTTP sign-in** was performed as the seeded `john@wick.com / password` using a `requests` session (fetch the login page, extract the CSRF token, POST credentials to the real `login()` handler at `app/auth/views/login.py:L25`). The captured flow:

```
GET  /auth/login  -> 200   (login page served; CSRF token extracted)
POST /auth/login  -> 302   Location: http://127.0.0.1:7777/dashboard/
GET  /dashboard/  -> 200
```

- The `302 → /dashboard/` on POST is the successful-authentication signal.
- `/dashboard/` returning `200` confirms the session is authenticated — reachability is gated by the Flask-Login user loader `load_user()` (`server.py:L220-L230`), which returns `None` for a disabled (`L225-L226`) or inactive (`L227-L228`) user; John is active, so the dashboard renders.

**Alias-management confirmation.** The dashboard index view (`app/dashboard/views/index.py`) lists the user's aliases. The rendered `/dashboard/` HTML for John contained his alias addresses and the SimpleLogin branding — the visible proof that alias management works:

```
(aliases present in the /dashboard/ HTML for john@wick.com)
  e0@sl.local
  e1@sl.local
  e2@sl.local
  simplelogin-newsletter.test631@sl.local
  word_list460@sl.local
  … ("SimpleLogin" branding present in the page chrome)
```

> The dashboard route is `/dashboard/` **with a trailing slash**; `/` is the `server.py` index that only redirects.

---


## 5. Q2 — Core user actions & correct-handling evidence

Three actions were exercised end-to-end through real HTTP/SMTP, capturing **before / intermediate / after** state at each boundary. The test account used was `blitzyprobe1783961522@gmail.com` (assigned **User id 4**) and the alias it created was `word_list027@sl.local` (**Alias id 15**). All of this test data was removed afterward — see [§9](#9-cleanup-confirmation).

### 5.1 C1 — Create a new account (signup)

Registration goes through the real endpoint `POST /auth/register` (handler `register()`, `app/auth/views/register.py:L32`).

> **Gotcha — a mailbox-usability (MX) gate applies at signup.** `email_can_be_used_as_mailbox()` requires the email's domain to have a resolvable MX record (`SKIP_MX_LOOKUP_ON_CHECK = False`, `app/config.py:L600`). In-container DNS works: `gmail.com` resolves real MX and is accepted, whereas `example.com` (no MX) is rejected. A real deliverable domain (`gmail.com`) was therefore used for the test address.

**BEFORE** — no such user:

```
$ psql ... -c "SELECT id,email,activated FROM users WHERE email='blitzyprobe1783961522@gmail.com';"
(0 rows)
```

**Register.** `GET /auth/register` → `200`; `POST /auth/register` → `200` (renders `auth/register_waiting_activation.html`, `register.py:L104`). The handler logged the create-user line at `register.py:L85`:

```
2026-07-13 16:52:03,242 - SL - DEBUG - 1828 - "/app/app/auth/views/register.py:85" - register() -  - create user blitzyprobe1783961522@gmail.com
```

**INTERMEDIATE** — the row now exists but is **not yet activated** (`activated` column defaults to `False`, `app/models.py:L358`); an `activation_code` row was created (`register.py:L120`, a 30-char code). The activation email ("Just one more step to join SimpleLogin", `register.py:L95` `send_activation_email`) was **short-circuited** by the default `NOT_SEND_EMAIL=true`:

```
$ psql ... -c "SELECT id,email,activated FROM users WHERE email='blitzyprobe1783961522@gmail.com';"
 id |               email                | activated
----+------------------------------------+-----------
  4 | blitzyprobe1783961522@gmail.com    | f

$ psql ... -c "SELECT length(code) FROM activation_code WHERE user_id=4;"
 length
--------
     30
```

**Activate through the real HTTP route** (not a DB flag-flip — that would be `[NON-CANONICAL]`). Because outbound email is off by default, the code is read from the DB and the canonical activation URL is hit — `GET /auth/activate?code=<code>` (`register.py:L124` builds `f"{URL}/auth/activate?code={activation.code}"`):

```
$ curl -i "http://127.0.0.1:7777/auth/activate?code=cqqahfaeinlkurfgfgeabeaejzatgp"
HTTP/1.1 302 FOUND
Location: http://127.0.0.1:7777/dashboard/
```
```
2026-07-13 16:52:xx,xxx - SL - DEBUG - 1828 - "/app/app/auth/views/activate.py:66" - activate() -  - redirect user to dashboard
```

**AFTER** — `activated` has flipped `false → true` via the canonical route:

```
$ psql ... -c "SELECT id,email,activated FROM users WHERE email='blitzyprobe1783961522@gmail.com';"
 id |               email                | activated
----+------------------------------------+-----------
  4 | blitzyprobe1783961522@gmail.com    | t
```

**Correct-handling evidence:** a `create user` log line, a persisted `User` row, and an `activated` transition `false → true` driven only by the real signup + activation HTTP routes.

> **Note on onboarding jobs.** `[INFERRED from config]` The AAP expected signup to auto-enqueue onboarding work for the job runner, but this environment has `DISABLE_ONBOARDING` set (`app/config.py:L401`, presence-based; `example.env:L150`). Registration accordingly logged that onboarding emails are disabled, and **no** onboarding `Job` was enqueued by signup. The job runner's liveness/cadence was therefore demonstrated with explicitly enqueued jobs instead ([§4.3](#43-job-runner-job_runnerpy-no-port), [§6.2](#62-a-jobs-lifecycle-ready--taken--done-with-attempts-accounting)).

### 5.2 C2 — Sign in

`POST /auth/login` was exercised for **both** the seeded account and the freshly-activated one; each produced a `302 → /dashboard/` and a subsequent `GET /dashboard/ → 200`:

```
# seeded account
POST /auth/login (john@wick.com / password)                 -> 302  Location: /dashboard/
GET  /dashboard/                                             -> 200   (aliases e0/e1/e2@sl.local … visible)

# newly-activated test account
POST /auth/login (blitzyprobe1783961522@gmail.com / …)       -> 302  Location: /dashboard/
GET  /dashboard/                                             -> 200   (only its own alias simplelogin-newsletter.list851@sl.local visible)
```

The new account seeing only *its own* alias (and not John's) is visible evidence that per-user data isolation is handled correctly.

### 5.3 C3 — Create an alias (canonical dashboard HTTP path)

A **random** alias is created by `POST /dashboard/` with form field `form-name=create-random-email` (handler `index()`, `app/dashboard/views/index.py:L67`), which calls `Alias.create_new_random(user=current_user, scheme=…)` (`index.py:L104`), logs `LOG.d("create new random alias %s for user %s", …)` (`index.py:L110`), and flashes `f"Alias {alias.email} has been created"` (`index.py:L111`).

> The alias-creation endpoint is `/dashboard/` (the dashboard blueprint's index route), **not** `/` (the `server.py` index, which only redirects).

**BEFORE** — the test user has exactly one alias:

```
$ psql ... -c "SELECT id,email,mailbox_id FROM alias WHERE user_id=4 ORDER BY id;"
 id |                  email                   | mailbox_id
----+------------------------------------------+------------
 14 | simplelogin-newsletter.list851@sl.local  |          6
(1 row)
```

**Create.** `POST /dashboard/` (`form-name=create-random-email`) → `302` with a highlight redirect, and the flash confirming the new address:

```
POST /dashboard/  ->  302  Location: /dashboard/?highlight_alias_id=15&query=&sort=&filter=
flash: "Alias word_list027@sl.local has been created"     (index.py:L111)
```
```
2026-07-13 16:53:56,903 - SL - DEBUG - 1829 - "/app/app/dashboard/views/index.py:110" - index() -  - create new random alias <Alias 15 word_list027@sl.local> for user <User 4 ...>
```

**AFTER** — a new `Alias` row (`app/models.py:L1469`) exists and the count went 1 → 2:

```
$ psql ... -c "SELECT id,email,mailbox_id,enabled,created_at FROM alias WHERE user_id=4 ORDER BY id;"
 id |                  email                   | mailbox_id | enabled |         created_at
----+------------------------------------------+------------+---------+----------------------------
 14 | simplelogin-newsletter.list851@sl.local  |          6 | t       | ...
 15 | word_list027@sl.local                    |          6 | t       | 2026-07-13 16:53:56...
(2 rows)
```

The new alias `word_list027@sl.local` also appeared in the `/dashboard/` listing (the same UI confirmation mechanism as [§4.4](#44-ui-confirmation--sign-in-and-alias-management-q1-dashboard-signals)). **Correct-handling evidence:** the create-alias log line, the flash message, the new persisted row, and its appearance in the dashboard.

### 5.4 C4 — Inbound email to the alias (the forward path)

The alias `word_list027@sl.local` (id 15, mailbox 6 = the test user's own mailbox) was sent a real message over the **canonical SMTP entry point** — a plain `smtplib` client to `127.0.0.1:20381` (functionally identical to the `swaks --server 127.0.0.1:20381` in `CONTRIBUTING.md:L218`; the image ships no `swaks`).

#### Part 1 — default config (`NOT_SEND_EMAIL=true`): the forward is *processed* and logged

SMTP transcript (sender `hey@google.com`):

```
EHLO ...                      -> 250 06a79fe3b90d ...
MAIL FROM:<hey@google.com>    -> 250 OK
RCPT TO:<word_list027@sl.local> -> 250 OK
DATA ... .                    -> 250 Message accepted for delivery
```

**Full `message_id`-correlated lifecycle** (message id `6a07d144-378f-4614-841a-7b5c3a897109`), captured contiguously from the handler log — from the boundary marker through the `Finish` line (nothing truncated before the relevant events):

```
2026-07-13 16:55:14,xxx - SL - DEBUG - 1917 - "/app/email_handler.py:2342" - _handle() - 6a07d144-378f-4614-841a-7b5c3a897109 - ====>=====>=====>=====>=====>=====>=====>
2026-07-13 16:55:14,xxx - SL - INFO  - 1917 - "/app/email_handler.py:2343" - _handle() - 6a07d144-378f-4614-841a-7b5c3a897109 - New message, mail from hey@google.com, rctp tos ['word_list027@sl.local']
2026-07-13 16:55:14,xxx - SL - INFO  - 1917 - "/app/email_handler.py:1980" - handle() - 6a07d144-378f-4614-841a-7b5c3a897109 - Handle mail_from hey@google.com, rcpt_tos ['word_list027@sl.local']
2026-07-13 16:55:14,xxx - SL - DEBUG - 1917 - "/app/email_handler.py:2202" - handle() - 6a07d144-378f-4614-841a-7b5c3a897109 - Forward phase hey@google.com -> word_list027@sl.local
2026-07-13 16:55:14,xxx - SL - DEBUG - 1917 - "/app/email_handler.py:580"  - get_or_create_contact() - 6a07d144-... - Create or get contact for from_header:hey@google.com
2026-07-13 16:55:14,811 - SL - INFO  - 1917 - "/app/app/contact_utils.py:110" - create_contact() - 6a07d144-... - Created contact <Contact 3 hey@google.com 15> for alias <Alias 15 word_list027@sl.local>
2026-07-13 16:55:14,xxx - SL - DEBUG - 1917 - "/app/email_handler.py:688"  - forward_email_to_mailbox() - 6a07d144-... - Forward <Contact 3 hey@google.com 15> -> <Alias 15 word_list027@sl.local> -> <Mailbox 6 blitzyprobe1783961522@gmail.com>
2026-07-13 16:55:14,830 - SL - DEBUG - 1917 - "/app/email_handler.py:740"  - forward_email_to_mailbox() - 6a07d144-... - Create <EmailLog 3> for <Contact 3 hey@google.com 15>, <User 4 ...>, <Mailbox 6 ...>
2026-07-13 16:55:14,xxx - SL - DEBUG - 1917 - "/app/app/mail_sender.py:131" - send() - 6a07d144-... - send email with subject 'Blitzy inbound forward test' ... (NOT_SEND_EMAIL -> not actually sent)
2026-07-13 16:55:14,xxx - SL - INFO  - 1917 - "/app/email_handler.py:2367" - _handle() - 6a07d144-... - Finish mail_from hey@google.com, rcpt_tos ['word_list027@sl.local'], takes 0.19637... seconds with return code '250 Message accepted for delivery'<<===
```

Key lines and their grounding:
- `Forward <Contact 3 …> -> <Alias 15 …> -> <Mailbox 6 …>` — the forward decision, `LOG.d("Forward %s -> %s -> %s", …)` at `email_handler.py:L688` (inside `forward_email_to_mailbox()`, reached from `handle_forward()` at `email_handler.py:L536`).
- `Create <EmailLog 3> for …` — the audit-row creation, `LOG.d("Create %s for %s, %s, %s", …)` at `email_handler.py:L740`.
- `Finish … return code '250 Message accepted for delivery'<<===` — the completion line, `email_handler.py:L2367`, echoing the SMTP return code.

**State evidence (new rows).** A `Contact` (`app/models.py:L1863`) for the sender and an `EmailLog` (`app/models.py:L2060`) for the forward:

```
$ psql ... -c "SELECT id,contact_id,alias_id,mailbox_id,user_id,is_reply,blocked,bounced FROM email_log WHERE user_id=4 ORDER BY id;"
 id | contact_id | alias_id | mailbox_id | user_id | is_reply | blocked | bounced
----+------------+----------+------------+---------+----------+---------+---------
  3 |          3 |       15 |          6 |       4 | f        | f       | f

$ psql ... -c "SELECT id,website_email,alias_id,user_id FROM contact WHERE user_id=4;"
 id | website_email  | alias_id | user_id
----+----------------+----------+---------
  3 | hey@google.com |       15 |       4
```

Because `is_reply=f, blocked=f, bounced=f`, `EmailLog.get_action()` (`app/models.py:L2132`) returns `"forward"` for this row.

**Dashboard activity counter (before / after).** The per-user counters shown by the dashboard come from `get_stats()` (`app/dashboard/views/index.py:L32-L52`), which **counts `EmailLog` rows** — `nb_forward` = count where `is_reply=False, blocked=False, bounced=False`. It incremented by exactly 1:

```
# nb_forward for user 4, replicating the get_stats() count query
BEFORE inbound email:  0
AFTER  inbound email:  1
```

> **Correction embedded (important):** `nb_forward`/`nb_block`/`nb_reply` at `app/models.py:L3238-L3240` are columns of the **`Metric2`** class (`app/models.py:L3211`) — a **global daily-metrics table explicitly commented as obsolete** ("only for the last 14 days"). They are **not** the per-user/per-alias activity counter. The real counter is the `EmailLog`-derived `get_stats()` value shown above.

#### Part 2 — documented email-test toggle: capturing the forwarded-mail artifact

With `NOT_SEND_EMAIL=true` the outbound delivery is short-circuited, so **no message reaches a mailbox** even though the forward is fully processed. To capture the actual forwarded artifact, the **documented** email-test configuration (`CONTRIBUTING.md:L200-L221`) was applied. **This is a temporary observation config in the git-ignored `.env`, not a source-tree change** (backed up and reverted afterward — see [§9](#9-cleanup-confirmation)):

- comment out `NOT_SEND_EMAIL` (so sending is re-enabled — recall it is presence-based);
- set `POSTFIX_SERVER=127.0.0.1` and `POSTFIX_PORT=1025`;
- run a local SMTP sink on `127.0.0.1:1025` (an `aiosmtpd` sink — the functional equivalent of the documented mailcatcher/MailHog, whose viewer is `http://localhost:1080/`);
- restart the email handler.

Re-sending the same message now performs a **real** outbound send (the short-circuit is gone) — the handler log shows the SMTP-connection and `Sendmail` lines from `app/mail_sender.py` instead:

```
... - "/app/app/mail_sender.py:156" - send() - ... - getting a smtp connection takes ... seconds
... - "/app/app/mail_sender.py:163" - send() - ... - Sendmail mail_from:sl.lmycy...@sl.local, rcpt_to:blitzyprobe1783961522@gmail.com
... - "/app/email_handler.py:2367" - _handle() - 55ac51ef-... - Finish ... return code '250 Message accepted for delivery'<<===
```

**Captured mail artifact** at the `:1025` sink — the forwarded message visibly arrived, addressed to the user's real mailbox, with SimpleLogin's forwarding headers intact:

```
envelope.mail_from : sl.lmycyibufqqdemzygi3ton25.r3zmgaqlidgu4@sl.local     (VERP reverse-path)
envelope.rcpt_tos  : ['blitzyprobe1783961522@gmail.com']                    (the real mailbox)
--- headers ---
X-SimpleLogin-Type          : Forward
X-SimpleLogin-EmailLog-ID   : 4
X-SimpleLogin-Envelope-From : hey@google.com
X-SimpleLogin-Envelope-To   : word_list027@sl.local
From                        : "hey at google.com" <hey_at_google_com_onicpwwelu@sl.local>   (reverse-alias)
To                          : word_list027@sl.local
Subject                     : Blitzy MTA-capture test
(body preserved)
```

The rewritten `From:` (a **reverse-alias** `hey_at_google_com_onicpwwelu@sl.local`) is what lets the user reply through the alias without exposing their address. This second send created `EmailLog id 4`.

**Correct-handling evidence for inbound email:** a fully-correlated `New message → Forward → Create EmailLog → Finish 250` log block, new `Contact` and `EmailLog` rows, an incremented `nb_forward`, and — with the documented test toggle — the forwarded message captured at the recipient mailbox with the expected SimpleLogin headers.

---


## 6. Q3 — Background auto-online behavior

### 6.1 Do the email handler and job runner stay online in the background?

**Yes — both are perpetual loops that keep accepting/processing work without any manual re-trigger.**

- **Email handler.** After `controller.start()`, `main()` holds the process open with `while True: time.sleep(2)` (`email_handler.py:L2392-L2393`). `[INFERRED from code]` for the specific `sleep(2)` keep-alive line; **observed** at runtime: the same handler process accepted messages repeatedly across the investigation (16:55, 16:57, and again after a restart at 16:58, 17:01, 17:02) while continuously reporting `Listen for port 20381`.
- **Job runner.** The loop opens a **fresh** `create_light_app().app_context()` **per iteration** (`job_runner.py:L332`) and polls every 10 s (`L347`). **Observed:** a single job-runner process (pid 1948, started 16:44:37) was still running with an elapsed time of `14:17` when checked, having drained jobs across 16:47–16:48 and again at 16:59–17:00 — i.e. it stayed online and kept polling on its own.

This continuous-loop design is precisely the mechanism behind "the components automatically come online in the background and keep working."

### 6.2 (A) Job's lifecycle: ready → taken → done, with attempts accounting

A batch of **8** genuine `Job` rows (`name=onboarding-1`, `state=ready`) was enqueued to observe the **intermediate** state, not just the endpoints. The job-runner drains them sequentially inside its for-loop (`job_runner.py:L333-L345`): each job is marked `taken=True` (`L337`), `state=JobState.taken.value` (`L339`), `attempts += 1` (`L340`), then after `process_job()` returns, `state=JobState.done.value` (`L344`). `JobState` is `ready=0 / taken=1 / done=2` (`app/models.py:L253-L256`).

Sampling the state distribution while the runner worked through the batch:

```
BEFORE        : {ready: 8, taken: 0, done: 0}
INTERMEDIATE  : {ready: 7, taken: 1, done: 0}   ->  {6,1,1}  ->  {5,1,2}  ->  {4,1,3}   (exactly ONE job 'taken' at a time)
AFTER         : {ready: 0, taken: 0, done: 8}   (every row: state=2, attempts=1, taken=t)
```

The "exactly one job in `taken` at a time" reflects the sequential per-job commit at `job_runner.py:L341`. This is the observed `ready(0) → taken(1) → done(2)` transition with `attempts` incrementing `0 → 1`. Timing corroborated the 10-second cadence from [§4.3](#43-job-runner-job_runnerpy-no-port) (≥2 cycles).

### 6.3 (B) Edge: unknown job name

A `Job` with a name **not** handled by `process_job()` (`job_runner.py:L188`) was enqueued. The dispatcher falls through to `LOG.e("Unknown job name %s", job.name)` (`job_runner.py:L304`). Captured, verbatim (note the ERROR level and the `NoneType: None` traceback tail, because `LOG.e` is `logging.Logger.exception`, `app/log.py:L77`):

```
2026-07-13 16:59:49,917 - SL - DEBUG - 1948 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 13 blitzy-unknown-job-xyz {'note': 'edge test'}>
2026-07-13 16:59:49,920 - SL - ERROR - 1948 - "/app/job_runner.py:304" - process_job() -  - Unknown job name blitzy-unknown-job-xyz
NoneType: None
```

**Post-state (observed):** the job still ends up `state=2 (done)`, `attempts=1`. Because `LOG.e` logs but does **not** raise, the loop proceeds to mark the job `done` after `process_job()` returns — an unknown job is logged-and-consumed, not retried.

### 6.4 (C) Edge: job retry / attempts gate

`get_jobs_to_run()` (`job_runner.py:L307`) will re-take a `taken` job only when **both** `taken_at < now − JOB_TAKEN_RETRY_WAIT_MINS` (30 minutes) **and** `attempts < JOB_MAX_ATTEMPTS` (5). Constants: `JOB_MAX_ATTEMPTS = 5` (`app/config.py:L564`), `JOB_TAKEN_RETRY_WAIT_MINS = 30` (`app/config.py:L565`).

Rather than wait 30 real minutes, three `taken` jobs were seeded with controlled `taken_at`/`attempts` to observe the **gate boundary** directly (the 30-minute wall-clock itself is thus `[INFERRED]` from the seeded timestamps; the gate *logic* is **observed**):

| Job | `taken_at` | `attempts` | Eligible? | Observed outcome |
|-----|-----------|-----------|-----------|------------------|
| id 14 | −31 min | 1 (`< 5`) | **yes** (both conditions met) | **RE-TAKEN** → `Take job <Job 14 onboarding-1>`, `attempts 1 → 2`, `state → done(2)` |
| id 15 | −5 min | 1 | no (too recent) | not re-taken → stays `state=1`, `attempts=1` |
| id 16 | −31 min | 5 (`not < 5`) | no (attempt ceiling) | not re-taken → stays `state=1`, `attempts=5` |

This directly demonstrates both halves of the retry gate and the `attempts` ceiling.

### 6.5 (D) Edge: email-handler phases & SMTP return-code mapping

The dispatcher `handle()` (`email_handler.py:L1945`) routes each message to one of three phases — **forward** (`handle_forward()`, `L536`), **reply** (`handle_reply()`, `L966`), or **bounce** (`handle_bounce()`, `L1851`) — and `handle_DATA()` (`email_handler.py:L2289`) maps failures to SMTP return codes. Observed outcomes:

**Forward (happy path)** → `250 Message accepted for delivery` (observed in [§5.4](#54-c4--inbound-email-to-the-alias-the-forward-path)).

**Send to a non-existent alias** (`nonexistent-blitzy-xyz@sl.local`) → the alias doesn't exist and can't be auto-created, so the handler returns **`550 SL E515 Email not exist`**:

```
... - "/app/email_handler.py:545" - handle_forward() - 74193471-... - alias nonexistent-blitzy-xyz@sl.local not exist. Try to see if it can be created on the fly
... - "/app/email_handler.py:551" - handle_forward() - 74193471-... - alias nonexistent-blitzy-xyz@sl.local cannot be created on-the-fly, return 550
... - "/app/email_handler.py:2367" - _handle() - 74193471-... - Finish ... with return code '550 SL E515 Email not exist'<<===
```
SMTP transcript: `RCPT/DATA -> 550 SL E515 Email not exist`.

**Reply phase from an unauthorized sender** — a stranger (`stranger@external.com`) sending to a **reverse-alias** (`hey_at_google_com_onicpwwelu@sl.local`) is routed to the reply phase, rejected because only the owning mailbox may use a reverse-alias, and the owner is notified. Returns **`250 SL E214 Unauthorized for using reverse alias`**:

```
... - "/app/email_handler.py:2196" - handle() - c033f56c-... - Reply phase stranger@external.com -> hey_at_google_com_onicpwwelu@sl.local
... - "/app/email_handler.py:1393" - handle_unknown_mailbox() - c033f56c-... - Reply email can only be used by mailbox. Actual mail_from: stranger@external.com ...
... (security notification "Attempt to use your alias word_list027@sl.local from stranger@external.com" sent to the mailbox owner)
... - "/app/email_handler.py:2367" - _handle() - c033f56c-... - Finish ... with return code '250 SL E214 Unauthorized for using reverse alias'<<===
```

**The other mapped codes were verified in source but not triggered at runtime** (they require contrived reverse-alias/VERP/malformed inputs). `[INFERRED]` from `email_handler.py:L2289-L2332`:
- `CannotCreateContactForReverseAlias` → `status.E524` → `"550 SL E524 Wrong use of reverse-alias"` (`L2307`).
- `(VERPReply | VERPForward | VERPTransactional)` → `status.E213` → `"250 SL E213 Unknown email ignored"` (`L2318`).
- generic `Exception` → `LOG.e(...)` → `status.E404` → `"421 SL E404 Unexpected error - Retry later"` (`L2332`).
- The **bounce** phase (`handle_bounce()`, `email_handler.py:L1851`) requires a `MAILER-DAEMON` bounce message to reach it and was **not** triggered — `[INFERRED]` from source.

### 6.6 (E) Secondary: alias auto-creation branches (alternate inbound paths)

When a message targets a non-existent alias, the handler tries to auto-create it via two branches dispatched from `try_auto_create()` (`app/alias_utils.py:L202`): `try_auto_create_via_domain()` (`L274`) and `try_auto_create_directory()` (`L227`). Both branches were **observed executing and declining** during the non-existent-alias send above (there is no custom domain or directory in the default seed):

```
... - "/app/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - ... - Cannot auto-create custom domain alias for nonexistent-blitzy-xyz@sl.local because there's no custom domain for sl.local
... - "/app/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - ... - Cannot auto-create nonexistent-blitzy-xyz@sl.local since it has no directory separator
```

The **success** logs — `LOG.d("create alias %s for directory %s", …)` (`alias_utils.py:L238`) and `LOG.d("create alias %s for domain %s", …)` (`alias_utils.py:L299`) — require a verified `CustomDomain` or a directory-separated address matching a `Directory`, neither present in the default seed, so those success paths are `[INFERRED]` from source.

---


## 7. Coverage pass

Every named item from the three questions and the reference set, and where it is answered. **✅ = observed at runtime; ⚠️ = inferred from source (with reason in [§8](#8-observed-vs-inferred-ledger)).**

| # | Named item | Where answered | Grounding | Status |
|---|-----------|----------------|-----------|--------|
| 1 | **Web server** up | §4.1 | `GET /health` → `200 "success"` | ✅ |
| 2 | **Email handler** up | §4.2 | `Start mail controller 0.0.0.0 20381`; port 20381 OPEN | ✅ |
| 3 | **Job runner** up | §4.3 | poll loop + `Take job` + 10 s cadence (×2) | ✅ |
| 4 | **Sign-in** | §4.4, §5.2 | `POST /auth/login` → `302 /dashboard/`; `GET /dashboard/` → `200` | ✅ |
| 5 | **Alias management** (UI) | §4.4, §5.3 | alias listing renders; flash "…has been created" | ✅ |
| 6 | **Receiving email** | §5.4 | `New message → Forward → Create EmailLog → Finish 250`; artifact at `:1025` | ✅ |
| 7 | **Background auto-online** | §6.1 | perpetual loops observed running unattended | ✅ |
| 8 | `healthcheck()` `server.py:L213-L215` | §4.1 | body `success`, `200` | ✅ |
| 9 | `/health` excluded from logs `server.py:L195,L281` | §3, §4.1 | no `SL` line for `/health` | ✅ |
| 10 | `after_request` access log `server.py:L284-L292` | §3, §4.1 | `… GET / … 302, takes …` | ✅ |
| 11 | `load_user()` `server.py:L220-L230` | §4.4 | `/dashboard/` gated, returns `200` for active user | ✅ |
| 12 | `local_main()` / port 7777 `server.py:L572,L588` | §2.4, §4.1 | Gunicorn `Listening at … :7777` | ✅ |
| 13 | `Listen for port %s` `email_handler.py:L2403` | §4.2 | `Listen for port 20381` | ✅ |
| 14 | `Start mail controller %s %s` `email_handler.py:L2386` | §4.2 | verbatim log | ✅ |
| 15 | keep-alive `while True: time.sleep(2)` `L2392-L2393` | §6.1 | handler stays online (loop line ⚠️) | ✅/⚠️ |
| 16 | `_handle()` / `message_id` `L2335-L2373` | §5.4 | correlated block, id `6a07d144…` | ✅ |
| 17 | `handle_DATA()` `L2289` + code map E524/E213/E404 | §6.5 | E404 map ⚠️; E515/E214 observed | ⚠️/✅ |
| 18 | `handle_forward()` `L536` / `Forward %s -> %s -> %s` `L688` | §5.4 | forward log line | ✅ |
| 19 | `EmailLog` create `L740` / model `L2060` | §5.4 | `Create <EmailLog 3> …`; DB row | ✅ |
| 20 | `handle_reply()` `L966` | §6.5 | reply phase → `250 SL E214` | ✅ |
| 21 | `handle_bounce()` `L1851` | §6.5 | needs MAILER-DAEMON message | ⚠️ |
| 22 | `Finish … return code '%s'<<===` `L2367` | §5.4, §6.5 | verbatim | ✅ |
| 23 | job loop `while True` / `time.sleep(10)` `L330,L347` | §4.3, §6.1 | 10.039 s / 10.034 s deltas | ✅ |
| 24 | `get_jobs_to_run()` + retry gate `L307-L326` | §6.4 | boundary table (30-min wall-clock ⚠️) | ✅/⚠️ |
| 25 | `Take job %s` `L334` | §4.3, §6.2, §6.3 | verbatim | ✅ |
| 26 | `process_job()` unknown-job `L304` `LOG.e` | §6.3 | `ERROR … Unknown job name …` + `NoneType: None` | ✅ |
| 27 | `JobState` ready/taken/done `models.py:L253-L256` | §6.2 | state `{8}→…→{done:8}` | ✅ |
| 28 | `Job` state defaults ready `models.py:L2697-L2698` | §6.2 | enqueued rows start `ready(0)` | ✅ |
| 29 | `register()` + `create user %s` `register.py:L32,L85` | §5.1 | verbatim log | ✅ |
| 30 | `User.activated` false→true `models.py:L358` | §5.1 | 3 DB snapshots | ✅ |
| 31 | canonical activation route `register.py:L124` | §5.1 | `GET /auth/activate?code=…` → `302` | ✅ |
| 32 | `login()` `login.py:L25` | §5.2 | `302 /dashboard/` | ✅ |
| 33 | random alias `index.py:L104,L110,L111` | §5.3 | log + flash + new row | ✅ |
| 34 | custom alias `custom_alias.py:L34,L139` | §7 note | not separately exercised (random path covers creation) | ⚠️ |
| 35 | `Contact` `models.py:L1863` | §5.4 | `Created contact <Contact 3 …>`; DB row | ✅ |
| 36 | `get_stats()` `nb_forward` `index.py:L32-L52` | §5.4 | `0 → 1` (EmailLog-derived) | ✅ |
| 37 | `Metric2` obsolete counters `models.py:L3211,L3238-L3240` | §5.4 | flagged as obsolete, not used as the counter | ✅ |
| 38 | `try_auto_create*` `alias_utils.py:L202,L227,L274` | §6.6 | both branches observed declining | ✅ (success ⚠️) |
| 39 | `delete_alias()` `alias_utils.py:L336` | §9 | used in cleanup; `L346/L368` logs | ✅ |
| 40 | `NOT_SEND_EMAIL` presence-based `config.py:L91` | §2.2, §5.4 | short-circuit observed; toggle re-enabled send | ✅ |
| 41 | `message_id` correlation `log.py:L18-L37` | §3, §5.4 | one id across the block | ✅ |
| 42 | `SL` format / logger / `LOG.d/i/w/e` `log.py:L12-L15,L74-L79` | §3 | annotated line | ✅ |
| 43 | Ports 7777 / 20381 / 15432 / 6379 / 1025 / 1080 | §2, §4, §5.4 | probed / used | ✅ |

---

## 8. Observed-vs-inferred ledger

Every `[INFERRED]` (⚠️) claim in this document and why it could not be directly observed:

1. **Email-handler keep-alive `while True: time.sleep(2)` (`email_handler.py:L2392-L2393`).** The *line* is read from source; what was **observed** is that the handler process stays listening and processes messages repeatedly over many minutes and across restarts (§6.1). Only the literal `sleep(2)` statement is inferred.
2. **30-minute retry wall-clock (`JOB_TAKEN_RETRY_WAIT_MINS=30`, `config.py:L565`).** The gate *logic* and boundary were **observed** by seeding `taken_at` at −31 min / −5 min (§6.4); the literal 30-minute elapsed wait was not sat through, so the exact wall-clock value is inferred from source.
3. **SMTP codes `E524` / `E213` / `E404` (`email_handler.py:L2307/L2318/L2332`).** These require contrived reverse-alias / VERP-bounce / malformed inputs. The forward `250`, the non-existent-alias `550 SL E515`, and the reply `250 SL E214` were **observed**; the three above are inferred from the verified exception→code map.
4. **Bounce phase `handle_bounce()` (`email_handler.py:L1851`).** Reaching it requires an inbound `MAILER-DAEMON` bounce; not produced in this run — inferred from source.
5. **Alias auto-create *success* logs (`alias_utils.py:L238` directory, `L299` domain).** Both auto-create branches were **observed executing and declining** (§6.6); success needs a verified `CustomDomain` or a `Directory`-matching address, neither in the default seed — success paths inferred.
6. **Custom-alias creation path (`custom_alias.py:L34,L139`).** Alias creation was exercised through the **random** dashboard path (§5.3); the custom-alias endpoint was not separately driven — its handler is cited from source.
7. **Onboarding-job auto-enqueue on signup.** `[INFERRED from config]` This environment sets `DISABLE_ONBOARDING` (`config.py:L401`), so signup did **not** enqueue onboarding jobs; the job runner was exercised with explicitly enqueued jobs instead (§5.1 note, §6.2).

No value in this document was taken off the canonical path; there are **no `[NON-CANONICAL]`** values. (Where a DB flag-flip would have been non-canonical — activating the new account — the real `GET /auth/activate` route was used instead; §5.1.)

---

## 9. Cleanup confirmation

Per the read-only mandate, all temporary test data was removed so the database returns to its post-seed baseline, and no source file was modified.

### 9.1 Test-data deletion (canonical path)

The test account was removed via the app's own `User.delete()` (`app/models.py:L671`), which iterates the user's aliases calling `delete_alias()` (`app/alias_utils.py:L336`) and then FK-cascades contacts / email_logs / mailbox / activation_code. Test `Job` rows were deleted (baseline job count is 0). Captured:

```
=== BEFORE cleanup ===
users = 3 | jobs = 15
Deleted 15 Job row(s).
2026-07-13 17:08:31,437 - SL - INFO - 2620 - "/app/app/alias_utils.py:346" - delete_alias() -  - User <User 4 blitzyprobe1783961522@gmail.com …> has deleted alias <Alias 14 simplelogin-newsletter.list851@sl.local>
2026-07-13 17:08:31,442 - SL - INFO - 2620 - "/app/app/alias_utils.py:368" - delete_alias() -  - Moving <Alias 14 …> to global trash <Deleted Alias simplelogin-newsletter.list851@sl.local>
2026-07-13 17:08:31,448 - SL - INFO - 2620 - "/app/app/alias_utils.py:346" - delete_alias() -  - User <User 4 …> has deleted alias <Alias 15 word_list027@sl.local>
2026-07-13 17:08:31,452 - SL - INFO - 2620 - "/app/app/alias_utils.py:368" - delete_alias() -  - Moving <Alias 15 …> to global trash <Deleted Alias word_list027@sl.local>
Deleted test User id=4 via canonical User.delete().
=== AFTER cleanup ===
users = 2 | jobs = 0
```

`delete_alias()` records deleted aliases in a global trash (`DeletedAlias`); those 2 residue rows for the test aliases were then removed to fully restore the baseline:

```
$ psql ... -c "DELETE FROM deleted_alias WHERE email IN ('simplelogin-newsletter.list851@sl.local','word_list027@sl.local');"
DELETE 2
```

### 9.2 Database restored to the pristine post-seed baseline

```
$ psql ... (row counts after cleanup)
      tbl        | count
-----------------+-------
 users           |     2      (john id1, winston id2 — both activated)
 alias           |    11      (ids 1-11; test aliases 14/15 gone)
 contact         |     1      (id1 — the fake_data.py:L86 seed contact)
 email_log       |     1      (id1 — the fake_data.py:L93 seed row)
 mailbox         |     4      (ids 1-4)
 job             |     0
 activation_code |     0
 deleted_alias   |     0
```

This exactly matches the state produced by `flask dummy-data` (which itself seeds 1 contact + 1 email_log + 0 jobs).

### 9.3 Configuration reverted; source tree untouched

The temporary email-test toggle (§5.4 Part 2) lived only in the **git-ignored** `.env` (`git check-ignore .env` → `.env`) and was reverted to defaults:

```
$ grep -nE "NOT_SEND_EMAIL|POSTFIX_SERVER|POSTFIX_PORT" /app/.env
19:NOT_SEND_EMAIL=true              # active again (presence-based)
69:# POSTFIX_SERVER=my-postfix.com  # original commented template (my 127.0.0.1 override removed)
154:# POSTFIX_PORT=1025             # original commented template (my 1025 override removed)
```

The running application's tracked source tree was never modified by this investigation (the only entries below are **pre-existing image-setup artifacts**, present before the investigation began — key material and a rebuilt lockfile — not changes made here):

```
$ git -C /app status --porcelain
 M app/spamassassin_utils.py
 M local_data/dkim.key
 M local_data/jwtRS256.key
 M local_data/jwtRS256.key.pub
 M local_data/paddle.key.pub
 M local_data/private-pgp.asc
 M local_data/test_words.txt
 M static/package-lock.json
```

### 9.4 The only committed artifact is this document

In the deliverable repository, the single new path is this file:

```
$ git status --porcelain
?? blitzy/

$ git add -n blitzy/
add 'blitzy/documentation/app_2cd6ee777f8c.md'
```

No tracked source file is added, modified, or deleted — the sole committed change is `blitzy/documentation/app_2cd6ee777f8c.md`.

---

*End of investigation. All behavioral claims above are backed by the captured runtime output shown alongside them; items marked `[INFERRED]`/⚠️ are enumerated in [§8](#8-observed-vs-inferred-ledger).*

