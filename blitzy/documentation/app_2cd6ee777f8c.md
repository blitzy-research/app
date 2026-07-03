# SimpleLogin (self‑hosted) — First‑Time Operator Runtime Q&A

This document answers three questions a first‑time operator asked about a freshly launched, self‑hosted **SimpleLogin** instance: (Q1) what confirms the app is ready, (Q2) what a new user sees while registering → verifying → logging in → landing on the dashboard, and (Q3) what the background/internal services do behind the scenes and how you can tell they are talking to each other.

Every behavioural statement below sits next to the **actual runtime output** that demonstrates it (a verbatim log line, an HTTP response/header, or a measured value), plus a `file:line` reference into the source that produced it. Statements I could not observe at runtime are explicitly labelled **inferred**; anything obtained outside the default configuration or outside the real entry point is labelled **non‑default** / **non‑canonical**.

> **Citation convention.** Every `file:line` reference uses the **full path from the repository root** (e.g. `app/auth/views/register.py:L85`, `events/event_sink.py:L18`, `app/events/event_dispatcher.py:L24`). Root‑level scripts are cited by their bare name (e.g. `email_handler.py:L2343`, `job_runner.py:L334`, `server.py:L215`) because they live at the repository root. There are **two** distinct event packages and both are cited by their real paths: `events/` (`events/event_sink.py`, `events/event_source.py`, `events/runner.py`) and `app/events/` (`app/events/event_dispatcher.py`). All line numbers were verified against the checked‑out source (`HEAD`).

> **Observation session.** All evidence below was captured in a single consistent run on **2026‑07‑03, 01:12–01:36 UTC**, inside the canonical container. Process ids are therefore stable across the quotes: web workers `3398`/`3399` (gunicorn master `3396`), SMTP handler `3419`, job runner `3420`, event listener `3421`/`4138`. Raw output was captured to log files and is quoted **verbatim** (no ellipses inside observed‑output fences; any redaction is marked `[REDACTED]` and explained). The single exception is the two-worker Gunicorn startup fence in Q1 §1 below: because each worker prints an identical preamble block, that fence shows the block once (collapsed for brevity), and the full, un-collapsed two-worker preamble is then quoted verbatim in the supplementary capture that follows it.

---

## Intro — how the instance was built, run, and observed

### Run environment (documented vs. observed)

| Component | Documented (source of truth) | Observed at runtime | Evidence |
|-----------|------------------------------|---------------------|----------|
| Application runtime | Python `^3.10` (Poetry range) / base image `python:3.10` | **Python 3.10.18** | `pyproject.toml:L61`; `Dockerfile:L8`; `python --version` |
| Front‑end build toolchain | **Node v10** (`node:10.17.0-alpine`) for asset build only | Node **v18.19.0** present; **not used at runtime** (assets prebuilt) | `Dockerfile:L2`; `CONTRIBUTING.md:L24`; `node --version` |
| Web / WSGI server | Gunicorn `^20.0.4` (`gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15`) | Gunicorn **20.0.4** | `pyproject.toml:L66`; `Dockerfile:L47` |
| Database | PostgreSQL **13+** | **PostgreSQL 15.13** (migration head `32f25cbf12f6`, 77 tables, seeded) | `CONTRIBUTING.md:L25`; `psql` output (below) |
| Cache / rate‑limit / session store | Redis | **Redis 7.0.15** (responds `PONG`) | `CONTRIBUTING.md:L65`; `redis-cli` output (below) |
| Config source | `example.env` defaults | `/app/.env` is a **byte‑for‑byte copy of `example.env`** | `diff -q .env example.env` → identical |

The instance runs inside the canonical Docker image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` in its **default configuration**. Config is read from `/app/.env`, which `app/config.py` auto‑loads at import via `load_dotenv()` [`app/config.py:L71`]. I verified the config is the shipped default and captured the exact runtime versions:

```console
$ diff -q .env example.env
# (no output) → .env is IDENTICAL to example.env

$ /app/venv/bin/python --version
Python 3.10.18

$ node --version
v18.19.0

$ psql "postgresql://myuser:mypassword@localhost:5432/simplelogin" -tAc "show server_version;"
15.13 (Debian 15.13-0+deb12u1)

$ redis-cli ping
PONG
$ redis-server --version | grep -o "v=[0-9.]*"
v=7.0.15
```

> **Node note.** The project's front‑end is built with **Node v10** (`FROM node:10.17.0-alpine AS npm` [`Dockerfile:L2`]; "Node v10 for front-end." [`CONTRIBUTING.md:L24`]). In this image the static assets are **prebuilt**, so Node is not needed at runtime — the `v18.19.0` present on `PATH` plays no part in serving requests. This is stated for completeness of the build environment.

### Provisioning / migration / seed (exact commands + verification)

The documented canonical bring‑up recipe is `alembic upgrade head && flask dummy-data && python3 server.py` [`CONTRIBUTING.md:L106`], which creates the demo login `john@wick.com / password` [`CONTRIBUTING.md:L109`]. In this environment the PostgreSQL database and Redis were **pre‑provisioned and pre‑seeded** by the image's setup (`sl_setup`) before observation; I verified that state with the exact queries below (Alembic head, table count, and the seeded users):

```console
$ psql "postgresql://myuser:mypassword@localhost:5432/simplelogin" -tAc "SELECT version_num FROM alembic_version;"
32f25cbf12f6
$ psql "postgresql://myuser:mypassword@localhost:5432/simplelogin" -tAc "SELECT count(*) FROM information_schema.tables WHERE table_schema=(SELECT current_schema());"
77
$ psql "postgresql://myuser:mypassword@localhost:5432/simplelogin" -tAc "SELECT id,email,activated FROM users ORDER BY id LIMIT 3;"
1|john@wick.com|t
2|winston@continental.com|t
$ grep -E "^DB_URI|^URL=" .env
URL=http://localhost:7777
DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin
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

# 5. Cron — scheduled by yacron reading crontab.yml (see Q1 §6 for the real yacron dispatch)
/app/venv/bin/yacron -c <crontab.yml-derived config>   # yacron then runs: python cron.py -j <job>
```

> **Note on the web entry point.** `python server.py` (the Flask dev server) binds to `127.0.0.1` only — `app.run(debug=True, port=7777)` with no `host=` argument [`server.py:L588`] — and is not reachable from outside the container, so I used the canonical production form `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` [`Dockerfile:L47`]. The WSGI object it serves is `app = create_app()` [`wsgi.py:L1,L3`].

### The uniform log format (so every line below is attributable)

Every SimpleLogin log line follows one format string [`app/log.py:L12-L15`]:

```
%(asctime)s - %(name)s - %(levelname)s - %(process)d - "%(pathname)s:%(lineno)d" - %(funcName)s() - %(message_id)s - %(message)s
```

That is: **timestamp – logger name – level – pid – `"pathname:lineno"` – `funcName()` – message‑id – message**. The logger name is always `SL` [`app/log.py:L79`], and each process prints a plain `>>> init logging <<<` banner when logging boots [`app/log.py:L67`]. Because the format embeds `pathname:lineno` and `funcName()`, every observed line can be attributed to the exact source location that emitted it — which is how the `file:line` citations below are grounded.

### Two default‑configuration flags that shape what you observe

Two shipped defaults materially change the observable behaviour, so they are called out up‑front:

1. **`NOT_SEND_EMAIL=true`** [`example.env:L19`]. Transactional email is **not delivered**; instead the mailer logs only the message metadata. Observed for the activation email during a real registration:

   ```
   2026-07-03 01:20:36,925 - SL - DEBUG - 3399 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'Just one more step to join SimpleLogin', from '"noreply@sl.local" <noreply@sl.local>' to 'blitzytmp_reg@example.com'
   ```

   > **Observed reality vs. expectation.** A common assumption is that the activation *link* is printed to the logs. It is **not**: `mail_sender.send()` under `NOT_SEND_EMAIL` logs only **subject / from / to** and returns immediately [`app/mail_sender.py:L130-L137`], never the body or the link. Consequently, "read the activation link from the logs" is **not satisfiable in the default configuration**. In Q2 I therefore (a) use the real `activation_code` DB row as a **non‑canonical** fallback to drive the real `/auth/activate` endpoint, and (b) separately capture the **real rendered email body containing the link** by running a **non‑default** MTA (with `NOT_SEND_EMAIL` disabled) — both clearly labelled.

2. **`DISABLE_ONBOARDING=true`** [`example.env:L150`]. Onboarding `Job` rows are **not** enqueued on self‑hosted registration, so the background‑job demonstration in Q3 uses a different observable job path rather than onboarding jobs.

3. **`EVENT_WEBHOOK` is unset** (there is no `EVENT_WEBHOOK=` line in `example.env`; `EVENT_WEBHOOK = os.environ.get("EVENT_WEBHOOK", None)` [`app/config.py:L612`]). As shown in Q3, this leaves the event‑delivery pipeline **wired but dormant** by default: the canonical alias‑creation trigger short‑circuits before any `SyncEvent`/`NOTIFY` is produced.

### Scope / hygiene

This was a **read‑only** investigation. Temporary users, aliases, jobs, `SyncEvent`s, observation scripts, and short‑lived helper services (a local MTA catcher, a local webhook receiver, a non‑default web instance) were created only to elicit runtime output and are **removed afterward** (verified in the final "Cleanup" section); no source file was modified. The seeded demo account is `john@wick.com / password` [`CONTRIBUTING.md:L109`].

---

## Q1 — After starting the system, what confirms the app is ready for authentication and alias/email activity?

> *"After starting the system what should I notice in the logs or UI that shows the app is ready to handle user authentication and alias based email activity?"*

Readiness is confirmed by the **startup output of five processes** plus a health probe and the initial UI state. Each process prints the `>>> init logging <<<` banner [`app/log.py:L67`] and then a process‑specific "I am listening / running" signal. Below, each signal is quoted verbatim from the captured logs.

### 1. Web application (the authentication surface) — Gunicorn on `0.0.0.0:7777`

Command: `/app/venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15`. Startup from the observation session (abridged — the identical second-worker preamble block is collapsed here for brevity; each of the two workers actually prints the full preamble, and the complete un-collapsed two-worker output is quoted verbatim in the supplementary capture that follows the notes below):

```
[2026-07-03 01:12:27 +0000] [3396] [INFO] Starting gunicorn 20.0.4
[2026-07-03 01:12:27 +0000] [3396] [INFO] Listening at: http://0.0.0.0:7777 (3396)
[2026-07-03 01:12:27 +0000] [3396] [INFO] Using worker: sync
[2026-07-03 01:12:27 +0000] [3398] [INFO] Booting worker with pid: 3398
[2026-07-03 01:12:27 +0000] [3399] [INFO] Booting worker with pid: 3399
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/jmroyuavohitjmdlrpln
Upload files to local dir
>>> init logging <<<
2026-07-03 01:12:28,056 - SL - DEBUG - 3398 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
>>> init logging <<<
2026-07-03 01:12:28,119 - SL - DEBUG - 3399 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

- **The app is bound and accepting connections.** `Listening at: http://0.0.0.0:7777 (3396)` — Gunicorn is bound to all interfaces on the port SimpleLogin exposes (`EXPOSE 7777` [`Dockerfile:L44`]; canonical `CMD` [`Dockerfile:L47`]). The WSGI object is `app = create_app()` [`wsgi.py:L1,L3`].
- **Two workers are up** (`-w 2`): `Booting worker with pid: 3398` and `Booting worker with pid: 3399`. Each worker independently boots the app, hence the banner block prints twice.
- **Logging is initialised** per worker: `>>> init logging <<<` [`app/log.py:L67`], immediately followed by the first `SL` log line, confirming the uniform format [`app/log.py:L12-L15`] and logger name `SL` [`app/log.py:L79`] are live.
- The `MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value` and `Paddle param not set` lines are benign default‑config notices printed by `app/config.py`; `Upload files to local dir` confirms local file storage (no S3) — all consistent with a default self‑hosted bring‑up.

**Supplementary capture — full two-worker preamble, verbatim (fresh run of the identical command).** Because `-w 2` starts two independent workers and each boots the app in its own process, the entire preamble block — including a *worker-specific* `GNUPGHOME` temp directory — prints once per worker. The session fence above collapses the identical second block for brevity; the capture below, taken from a fresh run of the exact same command, shows both blocks in full with nothing removed:

```
[2026-07-03 05:20:16 +0000] [8575] [INFO] Starting gunicorn 20.0.4
[2026-07-03 05:20:16 +0000] [8575] [INFO] Listening at: http://0.0.0.0:7777 (8575)
[2026-07-03 05:20:16 +0000] [8575] [INFO] Using worker: sync
[2026-07-03 05:20:16 +0000] [8577] [INFO] Booting worker with pid: 8577
[2026-07-03 05:20:16 +0000] [8578] [INFO] Booting worker with pid: 8578
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/sqnfdybtephchttnofmd
Upload files to local dir
>>> init logging <<<
2026-07-03 05:20:17,085 - SL - DEBUG - 8577 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/rvcqhccunqqonkejxjnn
Upload files to local dir
>>> init logging <<<
2026-07-03 05:20:17,109 - SL - DEBUG - 8578 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
[2026-07-03 05:20:22 +0000] [8575] [INFO] Handling signal: term
[2026-07-03 05:20:22 +0000] [8578] [INFO] Worker exiting (pid: 8578)
[2026-07-03 05:20:22 +0000] [8577] [INFO] Worker exiting (pid: 8577)
[2026-07-03 05:20:22 +0000] [8575] [INFO] Shutting down: Master
```

Across two fresh runs of the identical command, each run emitted the preamble exactly twice — two `>>> URL` lines, two `GNUPGHOME` temp-directory warnings (with *distinct* temp paths), and two `>>> init logging <<<` lines per run — confirming the duplication is stable and not a one-off, which is why the note above states the banner block prints twice.

### 2. Health check — `GET /health` returns `success, 200`

```console
$ curl -sS -i http://localhost:7777/health
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Fri, 03 Jul 2026 01:50:23 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 7
Vary: Cookie
Set-Cookie: slapp=[REDACTED]; Expires=Fri, 10-Jul-2026 01:50:23 GMT; HttpOnly; Path=/; SameSite=Lax

success
```

- **`HTTP/1.1 200 OK`** with body **`success`** (`Content-Length: 7`) is exactly what the handler returns: `return "success", 200` [`server.py:L215`] on the `@app.route("/health", methods=["GET"])` route [`server.py:L213`]. This is the single most direct "web app is ready" signal.
- **`Server: gunicorn/20.0.4`** confirms the request was served by the canonical production server, not the Flask dev server.
- The **`Set-Cookie: slapp=`** value is a live Flask session cookie; its payload is **`[REDACTED]`** here for safety (it is a session token). The cookie flags `HttpOnly`, `Path=/`, `SameSite=Lax` are shown verbatim.

### 3. Initial UI landing state

Hitting the site's entry points as an unauthenticated visitor produces the access log below (the per‑request logger is `after_request()` at `server.py:L284`):

```
2026-07-03 01:13:20,335 - SL - DEBUG - 3398 - "/app/server.py:284" - after_request() -  - 172.17.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.019712209701538086
2026-07-03 01:13:20,342 - SL - DEBUG - 3398 - "/app/server.py:284" - after_request() -  - 172.17.0.1 GET / ImmutableMultiDict([]) 302, takes 0.00022554397583007812
2026-07-03 01:13:20,368 - SL - DEBUG - 3399 - "/app/server.py:284" - after_request() -  - 172.17.0.1 GET /auth/register ImmutableMultiDict([]) 200, takes 0.01916670799255371
```

- **`GET /auth/login`** returned **`200`** — the login page renders; the authentication surface is live.
- **`GET /`** returned **`302`** — the site root **redirects** an unauthenticated visitor (to `/auth/login`), the expected landing behaviour before login.
- **`GET /auth/register`** returned **`200`** — the registration page renders; new‑user sign‑up is available.
- The access‑log line is emitted by a single `LOG.d` call whose format string is `"%s %s %s %s %s, takes %s"` [`server.py:L284-L292`], binding client IP, method, path, args, status, and elapsed seconds in that order. Note `/health` is **intentionally excluded** from this access log by the guard `not request.path.startswith("/health")` [`server.py:L281`], which is why the health probe above produced no access‑log line.

### 4. Inbound SMTP handler (the alias‑email surface) — port `20381`

Command: `/app/venv/bin/python email_handler.py`. Verbatim startup:

```
>>> init logging <<<
2026-07-03 01:12:45,039 - SL - DEBUG - 3419 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-03 01:12:45,729 - SL - INFO - 3419 - "/app/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-03 01:12:45,731 - SL - DEBUG - 3419 - "/app/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

- **`Listen for port 20381`** [`email_handler.py:L2403`] and **`Start mail controller 0.0.0.0 20381`** [`email_handler.py:L2386`] confirm the aiosmtpd controller is bound and ready to receive mail addressed to aliases. Port `20381` is the local SMTP port (mapped from `25` in production). This is the "alias based email activity" readiness signal.

### 5. Background job runner

Command: `/app/venv/bin/python job_runner.py`. Verbatim startup (the loop is idle until a `Job` row appears):

```
>>> init logging <<<
2026-07-03 01:12:44,875 - SL - DEBUG - 3420 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

- After the banner the runner enters its polling loop — a `while True:` block [`job_runner.py:L330`] whose every iteration ends with `time.sleep(10)` [`job_runner.py:L347`]. It logs nothing further until work arrives, at which point it prints `Take job %s` [`job_runner.py:L334`] (demonstrated with cadence measurements in Q3 §1). A silent job runner after the banner is the **healthy idle state**.

### 6. Event listener (internal event pipeline)

Command: `/app/venv/bin/python event_listener.py listener` (the `listener` subcommand is required). Verbatim startup:

```
>>> init logging <<<
2026-07-03 01:12:44,850 - SL - DEBUG - 3421 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-03 01:12:44,967 - SL - INFO - 3421 - "/app/event_listener.py:34" - main() -  - Using PostgresEventSource
2026-07-03 01:12:44,975 - SL - INFO - 3421 - "/app/event_listener.py:43" - main() -  - Starting with HttpEventSink
2026-07-03 01:12:44,975 - SL - INFO - 3421 - "/app/events/event_source.py:49" - __listen() -  - Starting to listen to events
```

- **`Using PostgresEventSource`** [`event_listener.py:L34`] — the listener will consume events via PostgreSQL `LISTEN/NOTIFY`.
- **`Starting with HttpEventSink`** [`event_listener.py:L43`] — successfully‑consumed events will be delivered to the configured webhook (the sink; behaviour and default‑config caveats are shown in Q3 §3).
- **`Starting to listen to events`** [`events/event_source.py:L49`] — the `LISTEN simplelogin_sync_events` subscription is active and the listener is blocked waiting for notifications. This is the "internal services are up" readiness signal.

### 7. Cron scheduler (yacron) — real scheduled dispatch of `cron.py`

The shipped schedule is defined in `crontab.yml`, e.g. the growth‑stats job: `command: python /code/cron.py -j stats` [`crontab.yml:L3`] on a daily `schedule: "0 0 * * *"` [`crontab.yml:L5`]. To **observe an actual yacron dispatch within the session** (rather than wait until midnight), I ran real yacron with a temporary config using the schedule `* * * * *` (every minute) and the command `cd /app && /app/venv/bin/python cron.py -j stats`. That dispatched command is **behaviorally equivalent to the shipped `stats` cron entry** — both invoke `cron.py -j stats` — but it is **not byte‑identical** to `crontab.yml`'s `command: python /code/cron.py -j stats` [`crontab.yml:L3`]: the shipped entry uses the image install path `/code` and a bare `python`, whereas in this container the code lives at `/app` and the interpreter is the venv `/app/venv/bin/python`. The freshly re‑observed argv confirms the exact form of the command actually dispatched: `will execute argv ['/bin/bash', '-c', 'cd /app && /app/venv/bin/python cron.py -j stats']`. Verbatim yacron output for one dispatch:

```
DEBUG:yacron:Job SimpleLogin growth stats (* * * * *) is scheduled for now
DEBUG:yacron:Job SimpleLogin growth stats retry config: {'maximumRetries': 0, 'initialDelay': 1, 'maximumDelay': 300, 'backoffMultiplier': 2}
INFO:yacron:Starting job SimpleLogin growth stats
DEBUG:yacron:SimpleLogin growth stats: will execute argv ['/bin/bash', '-c', 'cd /app && /app/venv/bin/python cron.py -j stats']
INFO:yacron:Job SimpleLogin growth stats spawned
[SimpleLogin growth stats stdout] >>> init logging <<<
[SimpleLogin growth stats stdout] 2026-07-03 01:14:08,568 - SL - DEBUG - 3508 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
[SimpleLogin growth stats stdout] 2026-07-03 01:14:09,430 - SL - DEBUG - 3508 - "/app/cron.py:1263" - <module>() -  - Start running cronjob
[SimpleLogin growth stats stdout] 2026-07-03 01:14:09,431 - SL - DEBUG - 3508 - "/app/cron.py:1275" - <module>() -  - Compute growth and daily monitoring stats
[SimpleLogin growth stats stdout] 2026-07-03 01:14:09,431 - SL - WARNING - 3508 - "/app/cron.py:540" - stats() -  - ADMIN_EMAIL not set, nothing to do
INFO:yacron:Job SimpleLogin growth stats exit code 0; has stdout: true, has stderr: false; fail_reason: None
INFO:yacron:Cron job SimpleLogin growth stats: reporting success
```

- **yacron scheduled and spawned the job**: `is scheduled for now` → `Starting job SimpleLogin growth stats` → `will execute argv ['/bin/bash', '-c', 'cd /app && /app/venv/bin/python cron.py -j stats']` → `Job SimpleLogin growth stats spawned`. This is the real scheduler dispatching `cron.py -j <job>` following the same invocation pattern that `crontab.yml` prescribes [`crontab.yml:L3`] (behaviorally equivalent to the shipped `stats` entry; the container‑specific path form is noted above).
- **The spawned `cron.py` ran and exited cleanly**: its own SL log lines are captured on yacron's stdout — `Start running cronjob` [`cron.py:L1263`], `Compute growth and daily monitoring stats` [`cron.py:L1275`], and the benign default‑config `ADMIN_EMAIL not set, nothing to do` [`cron.py:L540`] — then yacron reports `exit code 0` and `reporting success`. (Across the run yacron dispatched the job twice — at `01:14:09` pid `3508` and `01:15:02` pid `3518` — confirming the schedule fires repeatably.)

> **Label — non‑default vs. on‑demand.** The **schedule** used above (`* * * * *`) is a non‑default value chosen only so a dispatch could be observed live; the shipped schedule is `"0 0 * * *"` [`crontab.yml:L5`]. For comparison, invoking the job **on‑demand** — `/app/venv/bin/python cron.py -j stats` — produced the identical three SL cron lines (`Start running cronjob` / `Compute growth and daily monitoring stats` / `ADMIN_EMAIL not set, nothing to do`), but that path bypasses the scheduler and is therefore **not** scheduler evidence on its own.

### Q1 readiness summary

| Process | Command | Verbatim readiness signal | Source |
|---------|---------|---------------------------|--------|
| Web app | `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` | `Listening at: http://0.0.0.0:7777 (3396)` + `>>> init logging <<<` | `Dockerfile:L47`; `app/log.py:L67` |
| Health probe | `curl -i /health` | `HTTP/1.1 200 OK` / body `success` | `server.py:L213-L215` |
| UI landing | `GET /auth/login`, `GET /`, `GET /auth/register` | `200`, `302`, `200` | `server.py:L284` |
| SMTP handler | `python email_handler.py` | `Listen for port 20381` + `Start mail controller 0.0.0.0 20381` | `email_handler.py:L2403,L2386` |
| Job runner | `python job_runner.py` | banner then idle 10‑s poll loop | `job_runner.py:L330,L347` |
| Event listener | `python event_listener.py listener` | `Using PostgresEventSource` / `Starting with HttpEventSink` / `Starting to listen to events` | `event_listener.py:L34,L43`; `events/event_source.py:L49` |
| Cron scheduler | `yacron` → `python cron.py -j stats` | `Job SimpleLogin growth stats spawned` / `exit code 0` / `reporting success` | `crontab.yml:L3`; `cron.py:L1263` |

---

## Q2 — The new‑user journey: register → verify → login → dashboard

> *"walking through the typical product experience as if you were a new user signing up for the first time. What happens when a user registers verifies their address and tries to log in. What visible behavior confirms that the system is handling every step correctly and forwarding the user into the dashboard as expected?"*

Every step below was exercised through the **real HTTP entry points** on `http://localhost:7777` with a temporary user (`blitzytmp_reg@example.com`, later cleaned up). Server‑side log lines and per‑request access‑log lines are quoted verbatim.

### Step 1 — Register (`POST /auth/register`)

The registration created the user and returned the "check your email" waiting page (HTTP 200):

```
2026-07-03 01:20:36,633 - SL - DEBUG - 3399 - "/app/app/auth/views/register.py:85" - register() -  - create user blitzytmp_reg@example.com
2026-07-03 01:20:36,928 - SL - DEBUG - 3399 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/register ImmutableMultiDict([]) 200, takes 0.3382911682128906
```

- **`create user blitzytmp_reg@example.com`** [`app/auth/views/register.py:L85`] — the `User` row is created.
- **`POST /auth/register`** returned **`200`** — the response renders `auth/register_waiting_activation.html` [`app/auth/views/register.py:L104`], whose page title is **`Activation Email Sent`** [`templates/auth/register_waiting_activation.html:L3`]. This is the visible "we've emailed you a verification link" confirmation.
- **Duplicate‑registration guard.** Re‑posting the same address flashes an error rather than creating a second user — the flashed string is `Email {email} already used` [`app/auth/views/register.py:L82`], observed as the exact message `Email blitzytmp_reg@example.com already used` on the returned page.

**Observed browser‑console behavior on the waiting page (benign, default self‑hosted config).** Driving the registration through a **real browser** (rather than `curl`) and inspecting the DevTools console after landing on the waiting page shows the page is **visually correct** — it renders the `Activation Email Sent` card with the heading "An email to validate your email is on its way." and no visible error — yet the console emits two messages from the analytics wiring:

```
[log]   Analytics should only be enabled in prod
[error] Uncaught ReferenceError: plausible is not defined
```

This is a **benign, non‑blocking** artifact of the default self‑hosted configuration, not a failure of any registration step (the `User`/`ActivationCode` rows are created and the waiting page is served with HTTP `200`, exactly as above; the subsequent verify → login → dashboard steps all succeed). Cause → effect: the analytics loader `static/js/an.js` bails out early on any non‑production host — `if (!window.location.host.endsWith('simplelogin.io')) { console.log("Analytics should only be enabled in prod"); return; }` [`static/js/an.js:L3-L5`] — so it never reaches the line that would define the global stub `window.plausible = window.plausible || function() {…}` [`static/js/an.js:L28`]; and `base.html` injects the real Plausible library only when `PLAUSIBLE_HOST`/`PLAUSIBLE_DOMAIN` are set [`templates/base.html:L72-L75`], which they are not by default. The waiting page's own inline script then calls `plausible('Complete registration')` **unconditionally** [`templates/auth/register_waiting_activation.html:L14`], and because `plausible` is undefined on `localhost` the browser throws the `ReferenceError`. Guarding that inline call (e.g. `if (window.plausible) plausible('Complete registration')`) would remove the console error, but modifying the template is **out of scope for this read‑only investigation**, so the behaviour is reported here as observed rather than changed.

### The verification email and its activation link (three clearly‑labelled evidence levels)

This is the step that verifies the address, so it deserves precise treatment. There are three distinct observations, each labelled:

**(a) Default configuration (`NOT_SEND_EMAIL=true`) logs only metadata — the link is NOT in the logs.** During the real registration, the mailer logged exactly this and nothing more about the message:

```
2026-07-03 01:20:36,925 - SL - DEBUG - 3399 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'Just one more step to join SimpleLogin', from '"noreply@sl.local" <noreply@sl.local>' to 'blitzytmp_reg@example.com'
```

`mail_sender.send()` under `NOT_SEND_EMAIL` logs only **subject / from / to** and returns immediately [`app/mail_sender.py:L130-L137`] — the message **body**, and therefore the **activation link**, is never written to the logs. **Conclusion: "read the activation link from the default logs" is not satisfiable** in the default configuration. The link's canonical construction is `activation_link = f"{URL}/auth/activate?code={activation.code}"` [`app/auth/views/register.py:L124`].

**(b) Non‑default capture of the real rendered link (labelled NON‑DEFAULT).** To observe the actual link a user would receive, I ran a **non‑default** setup: a throwaway `aiosmtpd` catcher on `127.0.0.1:2525` and a second web instance on `:7778` started with `CONFIG=<nondefault.env>` where `NOT_SEND_EMAIL` is **removed** and `POSTFIX_SERVER=localhost` / `POSTFIX_PORT=2525` point the mailer at the catcher. Registering `blitzytmp_mta@example.com` against `:7778` delivered a real message to the catcher, which logged verbatim:

```
CAUGHT from=sl.lmzcyibqfqqdemzwg42dins5.ebulfva62yep4@sl.local to=['blitzytmp_mta@example.com'] bytes=27219
```

The decoded message contains the link verbatim:

Message headers (verbatim):

```
Subject: Just one more step to join SimpleLogin
From: "noreply@sl.local" <noreply@sl.local>
To: blitzytmp_mta@example.com
```

Complete `text/plain` body (verbatim, in full — no elision):

```
Thank you for choosing SimpleLogin.

To get started, please confirm that blitzytmp_mta@example.com is your email address using this link http://localhost:7777/auth/activate?code=pjqllmuyqrlmxoedmgiynpoqosbqvb within 1 hour.

If it wasn't you, maybe someone entered your email by mistake. In this case you can ignore this mail.


Best,
SimpleLogin team.

Do you have a question? Contact us at https://app.simplelogin.io/dashboard/support
```

The activation link `http://localhost:7777/auth/activate?code=pjqllmuyqrlmxoedmgiynpoqosbqvb` appeared identically in both the `text/plain` (above) and `text/html` parts, and matches the canonical format from `app/auth/views/register.py:L124`. Crucially, that **captured link was then verified through the canonical `:7777` endpoint** — following it produced a `302` to `/dashboard/` with the activation flash and flipped the user's DB row to activated (`11|blitzytmp_mta@example.com|t`). This proves the link a user receives is genuine and works end‑to‑end.

**(c) Direct DB read of the code (labelled NON‑CANONICAL).** To drive the default‑config `/auth/activate` step for the primary temp user (`blitzytmp_reg@example.com`) without a captured email, I read the `activation_code` row directly from PostgreSQL (`code = ctudjrjpjatkxyekdjrtiwcgsjhjor`). Reading the DB is a **bypassing interface**, not the user‑facing path, so this value is **non‑canonical**; it was used only to obtain a code to submit to the real HTTP endpoint below.

### Step 2 — Verify the address (`GET /auth/activate?code=<code>`)

Submitting the code to the real endpoint activated the account and forwarded the user to the dashboard:

```
2026-07-03 01:20:37,009 - SL - DEBUG - 3399 - "/app/app/auth/views/activate.py:66" - activate() -  - redirect user to dashboard
2026-07-03 01:20:37,009 - SL - DEBUG - 3399 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/activate ImmutableMultiDict([('code', 'ctudjrjpjatkxyekdjrtiwcgsjhjor')]) 302, takes 0.032498836517333984
2026-07-03 01:20:37,016 - SL - DEBUG - 3399 - "/app/app/dashboard/views/index.py:172" - index() -  - Show intro to <User 8 blitzytmp_reg@example.com blitzytmp_reg@example.com>
2026-07-03 01:20:37,115 - SL - DEBUG - 3399 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard/ ImmutableMultiDict([]) 200, takes 0.10241079330444336
```

- **`GET /auth/activate?code=<code>`** returned **`302`** — the address is verified and the response redirects. The success flash shown to the user is `Your account has been activated` [`app/auth/views/activate.py:L56`], and the redirect is logged as `redirect user to dashboard` [`app/auth/views/activate.py:L66`].
- **The redirect lands on the dashboard**: the very next request is `GET /dashboard/` returning `200`, and the dashboard controller logs `Show intro to <User 8 blitzytmp_reg@example.com blitzytmp_reg@example.com>` [`app/dashboard/views/index.py:L172`] before rendering `dashboard/index.html` [`app/dashboard/views/index.py:L215-L216`]. This is the visible confirmation the user is "forwarded into the dashboard as expected."
- **Account is now activated** (verified in the DB): the `users` row for id 8 shows `activated = t`.

**Verification edge cases** (both exercised on the real endpoint):

```
2026-07-03 01:20:37,123 - SL - DEBUG - 3399 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/activate ImmutableMultiDict([('code', 'THIS_CODE_DOES_NOT_EXIST_123')]) 400, takes 0.0038590431213378906
2026-07-03 01:22:00,107 - SL - DEBUG - 3399 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/activate ImmutableMultiDict([('code', 'fkeewxseemkwizdkvktozqshfibazs')]) 400, takes 0.0017178058624267578
```

- **Invalid code → `400`** with the page error `Activation code cannot be found` [`app/auth/views/activate.py:L33`].
- **Expired code → `400`** with the page error `Activation code was expired` [`app/auth/views/activate.py:L42`]. This branch is reached because `activation_code.is_expired()` [`app/auth/views/activate.py:L38`] returns true; `is_expired()` compares the stored expiry to now [`app/models.py:L1214`]. (For a temporary code, I set its `expired` column two hours into the past to trigger the real branch; the code value shown is the temp user's, later cleaned up.)

### Step 3 — Log in (`POST /auth/login`)

With the account activated, a correct login redirects into the dashboard:

```
2026-07-03 01:21:58,988 - SL - DEBUG - 3399 - "/app/app/auth/views/login_utils.py:35" - after_login() -  - log user <User 8 blitzytmp_reg@example.com blitzytmp_reg@example.com> in
2026-07-03 01:21:58,989 - SL - DEBUG - 3399 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 302, takes 0.23847603797912598
```

- **`log user <User 8 blitzytmp_reg@example.com blitzytmp_reg@example.com> in`** [`app/auth/views/login_utils.py:L35`] confirms the session was established; a successful `LoginEvent(LoginEvent.ActionType.success)` is also emitted [`app/auth/views/login.py:L71`].
- **`POST /auth/login`** returned **`302`** — `after_login()` redirects to `dashboard.index` [`app/auth/views/login_utils.py:L45`], i.e. `http://localhost:7777/dashboard/`, the observed `Location`.

Negative branches (both exercised, both correctly return `200` and re‑render the login page with an error rather than redirecting):

```
2026-07-03 01:21:59,235 - SL - DEBUG - 3399 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 200, takes 0.2394695281982422
2026-07-03 01:21:59,789 - SL - DEBUG - 3399 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 200, takes 0.23837947845458984
```

- **Wrong password → `200`** with the flash `Email or password incorrect` [`app/auth/views/login.py:L49`].
- **Un‑activated account → `200`** with the flash `Please check your inbox for the activation email. You can also have this email re-sent` [`app/auth/views/login.py:L63,L66`] — the guard `elif not user.activated:` blocks login until verification is done.

### Step 4 — Land on the dashboard (`dashboard.index`)

The forward into the dashboard is already visible above: after both activation and login, the request to `GET /dashboard/` returns `200`, the controller logs `Show intro to <User 8 blitzytmp_reg@example.com blitzytmp_reg@example.com>` [`app/dashboard/views/index.py:L172`], and `dashboard/index.html` [`app/dashboard/views/index.py:L215-L216`] is rendered. For a brand‑new account this is the alias dashboard (page title `Alias | SimpleLogin`) with the first‑time intro — the definitive "the system forwarded me into the dashboard" signal.

### Rate limiting on the auth endpoints (observed reality)

Both `/auth/login` and `/auth/activate` carry a `10/minute` rate limit applied with `@limiter.limit` [`app/auth/views/login.py:L22-L24`; `app/auth/views/activate.py:L14-L16`], where the limit is only **deducted on a failed attempt** — the decorator's predicate is `deduct_when=lambda r: hasattr(g, "deduct_limit") and g.deduct_limit`. `/auth/register` has **no** limiter decorator [`app/auth/views/register.py:L31`]. Observed full status sequences **from one run in this session** (the exact positions are single‑run evidence, not stable thresholds — see the variability note below):

```
/auth/login  (15 wrong-password attempts):
[200, 200, 200, 200, 200, 200, 200, 200, 200, 200, 200, 200, 200, 429, 429]

/auth/activate (15 invalid-code attempts):
[400, 400, 400, 400, 400, 400, 400, 400, 400, 400, 400, 400, 429, 400, 429]

/auth/register (12 GET attempts):
[200, 200, 200, 200, 200, 200, 200, 200, 200, 200, 200, 200]
```

- **`/auth/login`**: in this run the first `429` appeared at attempt **#14** (failed logins return `200` re‑rendering the page). **This position is not stable** — it depends on accumulated per‑worker counter state, as the re‑verification below shows.
- **`/auth/activate`**: in this run the first `429` appeared at attempt **#13**, and the sequence interleaved (`429`, then `400`, then `429`). Again, this is a **single‑run** position, not a fixed attempt number.
- **`/auth/register`**: no `429` at all — this is a **structural** fact (not a per‑run artifact): the route has no limiter decorator [`app/auth/views/register.py:L31`].
- **Why the exact first‑`429` position varies (and why the effective threshold is ~2× the "10/minute" nominal).** The limiter storage is in‑memory per process: `MEM_STORE_URI` defaults to `None` [`app/config.py:L568`] and `limiter = Limiter(key_func=__key_func)` [`app/extensions.py:L23`] is constructed with no shared backend. With Gunicorn running `-w 2`, **each worker keeps its own counter**, so the effective ceiling is roughly two independent buckets of 10, and which worker serves a given request — plus whatever counter state prior requests already accumulated — determines whether any specific attempt is the one that trips the limit. The precise first‑`429` attempt number is therefore **non‑deterministic run to run**; only the presence of the `10/minute` limiter on `/auth/login` and `/auth/activate`, its absence on `/auth/register`, and the deduct‑on‑failure semantics are stable, source‑grounded facts. This is observed behaviour of the default configuration, reported as‑is.

> **Re‑verification of the non‑determinism (fresh runs, same session, default `-w 2`).** Re‑running the loops confirms the exact positions shift with counter state. Two back‑to‑back `/auth/login` runs (20 wrong‑password attempts each) produced first‑`429` at attempt **#11**, then at attempt **#2** on the immediately following run (the second run starts with the first run's counters already near/over the ceiling):
>
> ```
> RUN-1 /auth/login  (20 wrong-pw):    [200, 200, 200, 200, 200, 200, 200, 200, 200, 200, 429, 200, 200, 200, 200, 200, 200, 200, 200, 200]  first429_at=11
> RUN-2 /auth/login  (20 wrong-pw):    [200, 429, 429, 429, 429, 429, 429, 429, 429, 429, 429, 429, 429, 429, 429, 429, 429, 429, 429, 429]  first429_at=2
> ```
>
> and two `/auth/activate` runs (20 invalid‑code attempts each) produced first‑`429` at attempt **#17**, then at attempt **#1**:
>
> ```
> RUN-1 /auth/activate (20 invalid-code): [400, 400, 400, 400, 400, 400, 400, 400, 400, 400, 400, 400, 400, 400, 400, 400, 429, 429, 400, 429]  first429_at=17
> RUN-2 /auth/activate (20 invalid-code): [429, 400, 429, 400, 400, 429, 429, 429, 429, 429, 429, 429, 429, 429, 429, 429, 429, 429, 429, 429]  first429_at=1
> ```
>
> The first‑`429` attempt number moved from `#14`→`#11`→`#2` (login) and `#13`→`#17`→`#1` (activate) across runs — so the sequences above are **evidence of one run only**, and the stable, portable claims are the source‑level ones (limiter presence/absence and deduct‑on‑failure), not any particular attempt number.

---

## Q3 — Behind the scenes: background jobs and internal services

> *"what the application does behind the scenes during that flow. Are there any indicators that background jobs or internal services are doing their part to support email forwarding or identity verification. What should I expect to observe at runtime that tells me these moving pieces are active and talking to each other properly?"*

The "moving pieces" do **not** call each other by RPC — they communicate through **shared PostgreSQL state**: the web app enqueues `Job` rows that the job runner drains, and alias‑creation events become `SyncEvent` rows announced by PostgreSQL `NOTIFY` that the event listener consumes. Below, each mechanism is observed at runtime.

### 1. Background job runner — a stable 10‑second polling cadence

Because `DISABLE_ONBOARDING=true` [`example.env:L150`] suppresses onboarding `Job` rows on self‑hosted registration, I demonstrated the runner's cadence **and** its dispatch path with a controlled workload: I enqueued ten `Job` rows (name `blitzy-observe-unknown`) via the canonical `Job.create` path, one every ~4 seconds, while the running job runner (pid `3420`) drained them. The first `Take job` of each poll cycle:

```
2026-07-03 01:29:27,056 - SL - DEBUG - 3420 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 7 blitzy-observe-unknown {'i': 0}>
2026-07-03 01:29:37,074 - SL - DEBUG - 3420 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 8 blitzy-observe-unknown {'i': 1}>
2026-07-03 01:29:47,099 - SL - DEBUG - 3420 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 11 blitzy-observe-unknown {'i': 4}>
2026-07-03 01:29:57,119 - SL - DEBUG - 3420 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 13 blitzy-observe-unknown {'i': 6}>
2026-07-03 01:30:07,135 - SL - DEBUG - 3420 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 16 blitzy-observe-unknown {'i': 9}>
```

- **The polling interval is ~10 seconds, measured across 5 cycles over a ~40‑second window** (satisfying the "observe across ≥2 cycles" discipline). The gaps between the first `Take job` of consecutive cycles are **10.018 s, 10.025 s, 10.020 s, 10.016 s** (mean ≈ 10.02 s), which confirms the `time.sleep(10)` at the end of the loop [`job_runner.py:L347`]. Each drained job is announced by `Take job %s` [`job_runner.py:L334`].
- **Dispatch by job name is observable.** Each temporary job took the `process_job` fallback branch, logging `Unknown job name blitzy-observe-unknown` [`job_runner.py:L304`]:

  ```
  2026-07-03 01:29:27,059 - SL - ERROR - 3420 - "/app/job_runner.py:304" - process_job() -  - Unknown job name blitzy-observe-unknown
  ```

  Real job names dispatch to real handlers instead — e.g. the alias‑creation‑events handler keyed on `config.JOB_SEND_ALIAS_CREATION_EVENTS` (value `"send-alias-creation-events"` [`app/config.py:L311`]) [`job_runner.py:L295-L302`]. The `Unknown job name` line is the visible proof that `process_job` is actually inspecting `job.name` and routing on it.
- These ten jobs were temporary observation artifacts and are removed in the Cleanup section.

### 2. Email forwarding lifecycle — the SMTP handler on port `20381`

To watch a forward end‑to‑end I sent one real message through the SMTP handler (`smtplib` → `localhost:20381`) from `blitzytmp_sender@example.com` to the seeded alias `test_test832@sl.local` (owned by `john@wick.com`). The handler accepted it (`250`) and logged the full lifecycle under one message id `1097d942-3d0b-4134-8b3e-adaa285bbe78`:

```
2026-07-03 01:31:16,683 - SL - INFO - 3419 - "/app/email_handler.py:2343" - _handle() - 1097d942-3d0b-4134-8b3e-adaa285bbe78 - New message, mail from blitzytmp_sender@example.com, rctp tos ['test_test832@sl.local'] 
2026-07-03 01:31:16,798 - SL - DEBUG - 3419 - "/app/email_handler.py:1980" - handle() - 1097d942-3d0b-4134-8b3e-adaa285bbe78 - ==>> Handle mail_from:blitzytmp_sender@example.com, rcpt_tos:['test_test832@sl.local'], header_from:blitzytmp_sender@example.com, header_to:test_test832@sl.local, cc:None, reply-to:None, message_id:<178304227667.3996.5152915510063862033@example.com>, client_ip:None, headers:[('Content-Type', 'text/plain; charset="us-ascii"'), ('MIME-Version', '1.0'), ('Content-Transfer-Encoding', '7bit'), ('Subject', 'Blitzy observation test'), ('From', 'blitzytmp_sender@example.com'), ('To', 'test_test832@sl.local'), ('Message-ID', '<178304227667.3996.5152915510063862033@example.com>'), ('Date', 'Fri, 03 Jul 2026 01:31:16 +0000')], mail_options:['SIZE=381'], rcpt_options:[]
2026-07-03 01:31:16,803 - SL - DEBUG - 3419 - "/app/email_handler.py:2202" - handle() - 1097d942-3d0b-4134-8b3e-adaa285bbe78 - Forward phase blitzytmp_sender@example.com(blitzytmp_sender@example.com) -> test_test832@sl.local
2026-07-03 01:31:16,843 - SL - DEBUG - 3419 - "/app/app/contact_utils.py:110" - create_contact() - 1097d942-3d0b-4134-8b3e-adaa285bbe78 - Created contact <Contact 3 blitzytmp_sender@example.com 2> for alias <Alias 2 test_test832@sl.local> with email blitzytmp_sender@example.com invalid_email=False
2026-07-03 01:31:16,851 - SL - DEBUG - 3419 - "/app/email_handler.py:688" - forward_email_to_mailbox() - 1097d942-3d0b-4134-8b3e-adaa285bbe78 - Forward <Contact 3 blitzytmp_sender@example.com 2> -> <Alias 2 test_test832@sl.local> -> <Mailbox 1 john@wick.com>
2026-07-03 01:31:16,854 - SL - DEBUG - 3419 - "/app/email_handler.py:740" - forward_email_to_mailbox() - 1097d942-3d0b-4134-8b3e-adaa285bbe78 - Create <EmailLog 3> for <Contact 3 blitzytmp_sender@example.com 2>, <User 1 John Wick john@wick.com>, <Mailbox 1 john@wick.com>
2026-07-03 01:31:16,860 - SL - DEBUG - 3419 - "/app/email_handler.py:867" - forward_email_to_mailbox() - 1097d942-3d0b-4134-8b3e-adaa285bbe78 - From header, new:"blitzytmp_sender at example.com" <blitzytmp_sender_at_example_com_gapyhrhmsx@sl.local>, old:blitzytmp_sender@example.com
2026-07-03 01:31:16,860 - SL - DEBUG - 3419 - "/app/app/mail_sender.py:131" - send() - 1097d942-3d0b-4134-8b3e-adaa285bbe78 - send email with subject 'Blitzy observation test', from '"blitzytmp_sender at example.com" <blitzytmp_sender_at_example_com_gapyhrhmsx@sl.local>' to 'test_test832@sl.local'
2026-07-03 01:31:16,860 - SL - INFO - 3419 - "/app/email_handler.py:2367" - _handle() - 1097d942-3d0b-4134-8b3e-adaa285bbe78 - Finish mail_from blitzytmp_sender@example.com, rcpt_tos ['test_test832@sl.local'], takes 0.17782831192016602 seconds with return code '250 Message accepted for delivery'<<===
```

- **Inbound accepted**: `New message, mail from blitzytmp_sender@example.com, rctp tos ['test_test832@sl.local']` [`email_handler.py:L2343`].
- **Routed to the forward phase** [`email_handler.py:L2202`], which creates the sender→alias `Contact` [`app/contact_utils.py:L110`] and resolves `Contact → Alias → Mailbox` (`<Alias 2 test_test832@sl.local> -> <Mailbox 1 john@wick.com>`) [`email_handler.py:L688`].
- **An `EmailLog` row is created** for the forward [`email_handler.py:L740`], and the sender address is rewritten to a per‑contact reverse alias (`blitzytmp_sender_at_example_com_gapyhrhmsx@sl.local`) so a reply routes back through SimpleLogin [`email_handler.py:L867`].
- **Completed**: the final line reports `takes 0.17782831192016602 seconds with return code '250 Message accepted for delivery'` [`email_handler.py:L2367`] — the measured processing time and the final SMTP code, verbatim. (Because `NOT_SEND_EMAIL=true`, the actual outbound send is logged as metadata only [`app/mail_sender.py:L131`] rather than delivered.)
- The `Contact 3` and `EmailLog 3` rows created here are temporary observation artifacts (attached to a seeded alias) and are removed in the Cleanup section.

### 3. The internal event pipeline — `SyncEvent` → PostgreSQL `NOTIFY` → listener → `HttpEventSink`

Alias creation is the canonical trigger for the event pipeline: `Alias.create` calls `EventDispatcher.send_event(user, EventContent(alias_created=event))` [`app/models.py:L1687`]. What actually happens under different configurations is where the honest, observed picture matters — there are **three** distinct observations:

**(a) Default configuration (`EVENT_WEBHOOK` unset): the canonical pipeline is wired but dormant.** Creating a random alias through the **real dashboard** (`POST /dashboard/` with `form-name=create-random-email`, which returns `302`) reached `send_event`, which **short‑circuited** at the webhook‑missing gate before producing any `SyncEvent` or `NOTIFY`:

```
2026-07-03 01:32:59,705 - SL - DEBUG - 3398 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email entire_jibbed984@sl.local
2026-07-03 01:32:59,719 - SL - INFO - 3398 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-03 01:32:59,723 - SL - DEBUG - 3398 - "/app/app/dashboard/views/index.py:110" - index() -  - create new random alias <Alias 21 entire_jibbed984@sl.local> for user <User 8 blitzytmp_reg@example.com blitzytmp_reg@example.com>
```

The alias (`Alias 21`) is created normally, but `Not sending events because webhook is not configured and allowed to be empty` [`app/events/event_dispatcher.py:L62`] means **no `SyncEvent`/`NOTIFY` is emitted under default config**. So on a default self‑hosted instance the event listener sits idle during normal use — expected, not a fault.

**(b) With `EVENT_WEBHOOK` set (non‑default) but a normal self‑hosted user: a second gate blocks it.** I started a non‑default web instance on `:7779` (worker pid `4101`) with `EVENT_WEBHOOK=http://localhost:8929/` and repeated the real dashboard alias creation. It got **past** the webhook gate but stopped at the **partner‑user** gate:

```
2026-07-03 01:35:00,099 - SL - INFO - 4101 - "/app/app/events/event_dispatcher.py:69" - send_event() -  - Not sending events because there's no partner user for user <User 8 blitzytmp_reg@example.com blitzytmp_reg@example.com>
2026-07-03 01:35:00,102 - SL - DEBUG - 4101 - "/app/app/dashboard/views/index.py:110" - index() -  - create new random alias <Alias 22 apexes_spoilt143@sl.local> for user <User 8 blitzytmp_reg@example.com blitzytmp_reg@example.com>
```

`Not sending events because there's no partner user for user <User 8 blitzytmp_reg@example.com blitzytmp_reg@example.com>` [`app/events/event_dispatcher.py:L69`] — the alias‑creation event is only emitted for users linked to a partner (e.g. Proton). **A normal self‑hosted user therefore never emits an alias‑creation `SyncEvent`, even with a webhook configured**, and the temporary webhook receiver got nothing. This is the honest finding: **the canonical dashboard‑triggered alias‑creation pipeline was not observed to deliver under default config, and is gated for ordinary self‑hosted users even with a webhook.**

**(c) The listener → sink transport itself works (real `HttpEventSink` delivery; NON‑CANONICAL trigger).** To prove the transport mechanics are live and inter‑communicating, I restarted the event listener with `EVENT_WEBHOOK=http://localhost:8929/` (a temporary local receiver) and produced a real `SyncEvent` + `NOTIFY` by calling the production dispatcher directly — `PostgresDispatcher.send()` runs `SyncEvent.create(content=event, flush=True)` then `Session.execute(f"NOTIFY {NOTIFICATION_CHANNEL}, '{instance.id}';")` [`app/events/event_dispatcher.py:L24-L26`] — with a serialized `AliasCreated` protobuf (80 bytes). This call is a **non‑canonical trigger** (it bypasses the dashboard), but everything downstream is the real production path:

```
2026-07-03 01:36:15,144 - SL - DEBUG - 4138 - "/app/events/event_source.py:55" - __listen() -  - Got NOTIFY: pid=4195 channel=simplelogin_sync_events payload=3
2026-07-03 01:36:15,216 - SL - INFO - 4138 - "/app/events/event_sink.py:22" - process() -  - Sending event 3 to http://localhost:8929/
2026-07-03 01:36:15,220 - SL - INFO - 4138 - "/app/events/event_sink.py:39" - process() -  - Event 3 sent successfully to webhook
2026-07-03 01:36:15,221 - SL - INFO - 4138 - "/app/events/runner.py:28" - __on_event() -  - Marked 3 as done
```

And the temporary webhook receiver logged the delivery verbatim:

```
2026-07-03T01:36:15.219433 POST / ct=application/x-protobuf bytes=80
```

- **`Got NOTIFY: pid=4195 channel=simplelogin_sync_events payload=3`** [`events/event_source.py:L55`] — the listener received the PostgreSQL notification on the `LISTEN simplelogin_sync_events` channel it subscribed to at startup [`events/event_source.py:L47`]. This is the direct proof the web/dispatcher side and the listener side are **talking to each other** through Postgres.
- **`Sending event 3 to http://localhost:8929/`** [`events/event_sink.py:L22`] → **`Event 3 sent successfully to webhook`** [`events/event_sink.py:L39`] — the `HttpEventSink` POSTed the serialized event and got HTTP 200. The success path is gated by `if res.status_code != 200:` [`events/event_sink.py:L33`]: a non-200 response would take the `if` branch (log `Failed to send event to webhook` and return `False`), while the observed 200 falls through to the `else` branch that emits the `Event 3 sent successfully to webhook` line quoted above and returns `True`. The receiver confirms `ct=application/x-protobuf bytes=80` (matching the 80‑byte payload).
- **`Marked 3 as done`** [`events/runner.py:L28`] and the row is deleted on success [`events/runner.py:L27`]; a follow‑up `SELECT count(*) FROM sync_event` returned `0`, confirming the consume‑and‑delete lifecycle. (The alternative sink, `ConsoleEventSink`, simply logs `Handling event {id}` for local dry‑runs [`events/event_sink.py:L43-L46`].)

### Q3 summary

| Behind‑the‑scenes mechanism | Observed runtime indicator | Source |
|-----------------------------|----------------------------|--------|
| Job‑runner cadence | `Take job` every ~10 s (measured 10.018/10.025/10.020/10.016 s over 5 cycles) | `job_runner.py:L334,L347` |
| Job dispatch by name | `Unknown job name blitzy-observe-unknown` (fallback branch) | `job_runner.py:L304` |
| Email forwarding | `New message` → `Forward phase` → `Create <EmailLog 3>` → `Finish` (return code `250 Message accepted for delivery`) | `email_handler.py:L2343,L2202,L740,L2367` |
| Event trigger (default) | `Not sending events because webhook is not configured and allowed to be empty` (dormant) | `app/events/event_dispatcher.py:L62` |
| Event trigger (webhook set, self‑hosted user) | `Not sending events because there's no partner user for user <User 8 blitzytmp_reg@example.com blitzytmp_reg@example.com>` — gated | `app/events/event_dispatcher.py:L69` |
| Event transport (real) | `Got NOTIFY` → `Event 3 sent successfully to webhook` → `Marked 3 as done` | `events/event_source.py:L55`; `events/event_sink.py:L39`; `events/runner.py:L28` |

---


## Coverage pass — every named item in the question, and whether it was observed

This checklist decomposes the three question groups into every distinct mechanism, file, flag, and "e.g./such as/including" example the prompt names, and states for each whether it was **observed at runtime** (with the section that carries the adjacent evidence), or whether it is **gated/unavailable under default config** (labelled honestly rather than marked complete).

### Q1 — Readiness / startup indicators

- [x] **Web app (`server.py` / `wsgi.py`, port 7777)** — gunicorn startup banner + `Listening at: http://0.0.0.0:7777` observed. Evidence: Q1 §"Web application". [`wsgi.py:L1-L3`; `Dockerfile:L47`]
- [x] **`>>> init logging <<<` banner** — observed at logging bootstrap. Evidence: Q1 §"Web application". [`app/log.py:L67`]
- [x] **`GET /health` → `("success", 200)`** — full verbatim `curl -i` response observed. Evidence: Q1 §"Health probe". [`server.py:L213-L215`]
- [x] **Per‑request access log** — observed with the `"%s %s %s %s %s, takes %s"` format; `/health` intentionally excluded. Evidence: Q1 §"UI landing & access log". [`server.py:L281,L284-L292`]
- [x] **UI landing state** — `GET /auth/login` → `200`, `GET /` → `302`, `GET /auth/register` → `200` observed. Evidence: Q1 §"UI landing & access log".
- [x] **SMTP handler (`email_handler.py`, port 20381)** — `Listen for port 20381` and `Start mail controller 0.0.0.0 20381` observed. Evidence: Q1 §"SMTP handler". [`email_handler.py:L2403,L2386`]
- [x] **Job runner (`job_runner.py`)** — startup banner then silent 10‑second idle loop observed. Evidence: Q1 §"Job runner". [`job_runner.py:L330,L347`]
- [x] **Event listener (`event_listener.py`)** — `Using PostgresEventSource` / `Starting with HttpEventSink` / `Starting to listen to events` observed. Evidence: Q1 §"Event listener". [`event_listener.py:L34,L43`]
- [x] **Cron scheduler (`cron.py` + `crontab.yml`) — real yacron dispatch** — yacron scheduled and spawned `cd /app && /app/venv/bin/python cron.py -j stats`, exit code `0`, "reporting success" observed. On‑demand `cron.py -j stats` is included only as a comparison and is labelled on‑demand. Evidence: Q1 §"Cron scheduler". [`crontab.yml:L3,L5`; `cron.py:L1263`]

### Q2 — New‑user flow (register → verify → login → dashboard)

- [x] **`POST /auth/register`** — `create user %s` log + `200` rendering `register_waiting_activation.html` (title `Activation Email Sent`) observed. Evidence: Q2 §"Step 1". [`app/auth/views/register.py:L85,L104`]
- [x] **Duplicate registration** — `Email {email} already used` flash observed. Evidence: Q2 §"Step 1". [`app/auth/views/register.py:L79-L83`]
- [⚠] **Activation link "from the logs"** — **NOT satisfied under default config.** With `NOT_SEND_EMAIL=true`, `mail_sender.send()` logs only **subject / from / to** and returns [`app/mail_sender.py:L130-L137`]; the link is **not** in the logs. The real rendered link was captured only via a **non‑default** temporary MTA (labelled non‑canonical), and the DB‑derived `ActivationCode` is labelled a **non‑canonical fallback**. Evidence: Q2 §"Reading the activation link". [`example.env:L19`]
- [x] **`GET /auth/activate?code=<code>` → `302`** — `Your account has been activated` flash + `redirect user to dashboard` observed. Evidence: Q2 §"Step 2". [`app/auth/views/activate.py:L56,L66`]
- [x] **Invalid activation code → `400`** — `Activation code cannot be found` observed. Evidence: Q2 §"Negative activation cases". [`app/auth/views/activate.py:L28-L36`]
- [x] **Expired activation code → `400`** — `Activation code was expired` observed. Evidence: Q2 §"Negative activation cases". [`app/auth/views/activate.py:L38-L46`]
- [x] **`POST /auth/login` → `302`** — `log user %s in` + `LoginEvent(LoginEvent.ActionType.success)` observed. Evidence: Q2 §"Step 3". [`app/auth/views/login_utils.py:L35`; `app/auth/views/login.py:L71`]
- [x] **Wrong password** — `Email or password incorrect` observed. Evidence: Q2 §"Negative login cases". [`app/auth/views/login.py:L45-L50`]
- [x] **Not‑activated login** — "please check your inbox" branch observed. Evidence: Q2 §"Negative login cases". [`app/auth/views/login.py:L63-L69`]
- [x] **Forward into dashboard** — `GET /dashboard/` → `200`, `Show intro to %s`, render `dashboard/index.html` observed. Evidence: Q2 §"Step 4". [`app/dashboard/views/index.py:L172,L215-L216`]
- [x] **Rate limits on `/login`, `/activate`, `/register`** — full status sequences observed; the `10/minute` limit and its per‑worker in‑memory behaviour (`MEM_STORE_URI=None`, gunicorn `-w 2`) explained as cause→effect. Evidence: Q2 §"Rate limits". [`app/auth/views/login.py:L22-L24`; `app/auth/views/activate.py:L14-L16`]

### Q3 — Behind‑the‑scenes background / internal services

- [x] **Background jobs — `job_runner.py` draining the `Job` table on a 10‑second cadence** — measured `10.018 / 10.025 / 10.020 / 10.016 s` across 5 cycles observed. Evidence: Q3 §1. [`job_runner.py:L334,L347`]
- [x] **`process_job` dispatch by name (incl. `Unknown job name` fallback)** — `Unknown job name blitzy-observe-unknown` observed. Evidence: Q3 §1. [`job_runner.py:L304`]
- [x] **Email forwarding — `email_handler.py` message lifecycle** — the stages `New message`, forward phase, `Create <EmailLog 3>`, and `Finish` (final return code `250 Message accepted for delivery`) were observed, with the full internal message‑id and the full `0.17782831192016602 s` timing. Evidence: Q3 §2. [`email_handler.py:L2343,L2202,L740,L2367`]
- [⚠] **Internal services — alias‑creation event → `SyncEvent` + PostgreSQL `NOTIFY` via the canonical dashboard flow** — **NOT observed to deliver under default config.** Real dashboard alias creation short‑circuits at `app/events/event_dispatcher.py:L62` (webhook not configured) by default; with a non‑default `EVENT_WEBHOOK` it still short‑circuits at the **partner‑user** gate `app/events/event_dispatcher.py:L69` for an ordinary self‑hosted user. Both gates were observed via the **real dashboard flow**. Evidence: Q3 §3(a), §3(b). [`app/events/event_dispatcher.py:L62,L69`; `app/models.py:L1687`]
- [x] **Internal services — the listener → `HttpEventSink` transport (LISTEN/NOTIFY → webhook POST → consume‑and‑delete)** — proven live via a **non‑canonical** direct `PostgresDispatcher.send()` trigger: `Got NOTIFY` → `Event 3 sent successfully to webhook` → `Marked 3 as done`, with the receiver logging `POST / ct=application/x-protobuf bytes=80` and `sync_event` returning to `0`. Trigger labelled non‑canonical; downstream transport is the real production path. Evidence: Q3 §3(c). [`events/event_source.py:L55`; `events/event_sink.py:L39`; `events/runner.py:L27-L28`]
- [x] **Identity verification — `ActivationCode` issuance & consumption** — issuance at register and consumption at activate observed (via the activation flow in Q2). Evidence: Q2 §"Step 1", §"Step 2". [`app/auth/views/register.py:L117-L129`; `app/auth/views/activate.py:L49-L58`]
- [x] **Cron / internal‑services scheduling** — real yacron dispatch (see Q1 cron). Evidence: Q1 §"Cron scheduler". [`crontab.yml:L3`]

**Two items are honestly marked ⚠ rather than complete:** the activation link is not exposed in default logs (captured only via a non‑default MTA and a non‑canonical DB fallback), and the canonical dashboard‑triggered alias‑creation event pipeline does not deliver under default config (it is gated by `EVENT_WEBHOOK` being unset and, even when set, by the partner‑user check for ordinary self‑hosted users). The downstream listener→sink transport itself is proven live via a clearly labelled non‑canonical trigger. These reflect the software as shipped in default configuration.

---

## Final validation & cleanup

Every temporary user, alias, contact, email‑log, job, and `SyncEvent` created for observation was deleted, and the host repository was confirmed unchanged except for this document.

### Temporary runtime data removed (verified `0`)

The cleanup deletions reported `DELETE 1` (email_log), `DELETE 1` (contact), `DELETE 10` (jobs), `DELETE 4` (users, cascading temp aliases and activation codes). The post‑cleanup verification counts, run against the canonical app database `postgresql://myuser:mypassword@localhost:5432/simplelogin`, are all `0`, and the seeded baseline is intact:

```
=== temp users remaining (expect 0) ===
0
=== temp aliases remaining (expect 0) ===
0
=== temp contact blitzytmp_% remaining (expect 0) ===
0
=== temp jobs blitzy-observe-unknown remaining (expect 0) ===
0
=== sync_event count (expect 0) ===
0
=== seeded baseline intact: users + total alias count ===
1|john@wick.com
2|winston@continental.com
11
```

The seeded users `john@wick.com` and `winston@continental.com` remain, and the alias count returned to `11` (the pre‑observation seeded maximum), confirming temp aliases `17–22` were removed by the cascade.

### Temporary services stopped, temporary scripts removed

The temporary observation processes started for Q2/Q3 — the non‑default web instance on `:7778` (with `NOT_SEND_EMAIL` removed, pointing at the catcher) and its `aiosmtpd` mail catcher on `127.0.0.1:2525` (Q2 §(b)); the non‑default web instance on `:7779` with `EVENT_WEBHOOK` set (Q3 §3(b)); the temporary local webhook receiver on `:8929` and the extra event listener that delivered to it (Q3 §3(c)); and the temporary yacron config (Q1 §7) — were all stopped, and the default configuration restored (`EVENT_WEBHOOK` unset; `.env` byte‑for‑byte equal to `example.env`). No observation scripts or captured‑output files remain in the repository.

### Host repository left pristine (only the deliverable)

The read‑only mandate is satisfied: the only change this investigation makes to the host repository is this Q&A answer document under `blitzy/documentation/`; no source, config, dependency, migration, template, or test file was modified, added, or deleted.

**Author‑run evidence (historical — captured while authoring, before this document was committed).** While the document was still an in‑progress, un‑committed change, `git status --porcelain` in the host repository root (`/tmp/blitzy/app/blitzy-04138eae-fe6d-47c5-986f-bf39151cbea3_34759f`) showed only the in‑progress document and nothing else:

```
 M blitzy/documentation/app_2cd6ee777f8c.md
```

**Current repository state.** Once the document is committed it becomes part of `HEAD`, so the **current** working tree is clean: `git status --porcelain` returns **no output**, while `git ls-files blitzy/documentation/app_2cd6ee777f8c.md` still lists the tracked deliverable. The two states express the same guarantee — apart from this single added document, the repository is left byte‑for‑byte unchanged. (The `M` line above is therefore author‑run evidence of the change being introduced, not a claim about the current committed state.)

