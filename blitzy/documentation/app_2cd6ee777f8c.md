# SimpleLogin Runtime Investigation — Web Server, Email Handler, and Job Runner

> An **evidence‑grounded** Q&A report. Every behavioural claim below is paired with the **exact command** that produced it and the **verbatim output** that was observed, plus a `file:line` citation into the source. Nothing here is written from reading code alone — the stack was **built, run, and observed first**, then documented.

## What SimpleLogin is, and what was tested

SimpleLogin is an open‑source email‑aliasing service implemented as a **Python/Flask monolith** — the Poetry project is literally named `name = "SimpleLogin"` `[pyproject.toml:L47]`. The single code package is launched through several independent process entry points. This report concerns the **three components named in the questions**:

| Component | Entry point | Port / cadence |
|-----------|-------------|----------------|
| **Web server** | `server.py` (dev) / `gunicorn wsgi:app` (prod) `[Dockerfile:L47]` | TCP **7777** |
| **Email handler** | `email_handler.py` (aiosmtpd) `[email_handler.py:L2381]` | TCP **20381** |
| **Job runner** | `job_runner.py` `[job_runner.py:L329]` | **10‑second** poll loop `[job_runner.py:L347]` |

All three are separate OS processes that **share the same database and configuration** because each builds the same lightweight Flask application context via `create_light_app()` (defined at `[server.py:L127]`, imported by `[email_handler.py:L177]` and `[job_runner.py:L24]`).

### How evidence was gathered

Everything was observed inside the canonical Docker runtime container `sl-runtime` (image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`, **Python 3.10.18**). Commands were run through `docker exec sl-runtime …`. The container ships `curl` and Python (`smtplib`, `socket`) but **not** `lsof`/`ss`/`netstat`/`nc`/`swaks`, so port‑binding was proven from `/proc/net/tcp` and inbound mail was injected with Python's `smtplib`. Each component's stdout was redirected to its own log file under `/tmp/blitzy_obs/` so lines could be quoted verbatim; those temporary files and all test data were removed afterward.

The supporting services were probed directly, and their **observed versions** were captured verbatim:

```bash
$ docker exec sl-runtime su postgres -c "psql -d simplelogin -tAc \"SELECT version();\""
PostgreSQL 15.13 (Debian 15.13-0+deb12u1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 12.2.0-14+deb12u1) 12.2.0, 64-bit

$ docker exec sl-runtime bash -lc 'redis-cli INFO server | grep "^redis_version:"'
redis_version:7.0.15

$ docker exec sl-runtime bash -lc 'redis-cli ping'
PONG

$ docker exec sl-runtime /usr/local/bin/mailhog -version
MailHog version: 1.0.1
```

So the data store, cache, and mail sink are **PostgreSQL 15.13**, **Redis 7.0.15** (which answered `PONG`), and **MailHog 1.0.1** (SMTP `:1025`, HTTP API `:8025`). The runtime `/app/.env` — which `[CONTRIBUTING.md:L111]` notes "is ignored by git", so it is **not a tracked repository file** and cannot be cited as one — is configured for local mail observation. Its exact lines were read directly:

```bash
$ docker exec sl-runtime bash -lc 'grep -nE "^URL=|NOT_SEND_EMAIL|^EMAIL_DOMAIN=|^DB_URI=|^POSTFIX_SERVER=|^POSTFIX_PORT=|^DISABLE_ONBOARDING=" /app/.env'
6:URL=http://localhost:7777
19:# NOT_SEND_EMAIL=true  # commented for local mail-sink observation
22:EMAIL_DOMAIN=sl.local
69:POSTFIX_SERVER=localhost
75:DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin
150:DISABLE_ONBOARDING=true
154:POSTFIX_PORT=1025
```

Each runtime value mirrors the **tracked template** `example.env`, which is the correct file to cite: `URL` `[example.env:L6]`, `NOT_SEND_EMAIL` `[example.env:L19]` (commented out here so forwarded mail actually reaches the sink), `EMAIL_DOMAIN=sl.local` `[example.env:L22]`, `DB_URI` `[example.env:L75]`, and `POSTFIX_PORT=1025` `[example.env:L154]`. `POSTFIX_SERVER=localhost` with `POSTFIX_PORT=1025` directs forwarded mail at MailHog. The `DISABLE_ONBOARDING=true` line — `[example.env:L150]` in the template — is analysed in Q3, where its effect on account‑creation jobs is measured.

> **A note on runtime vs. canonical values.** `[CONTRIBUTING.md:L100]` documents Postgres 13 mapped `-p 15432:5432`; the running image actually provides Postgres **15.13** reached on the in‑container port **5432** (matching `DB_URI` `[example.env:L75]`). Where a value below is a canonical instruction it is cited to `CONTRIBUTING.md`/`example.env`; where it is a measured runtime value it is quoted from captured output.

---

## Runtime setup (commands actually run)

**Baseline before observation** (clean seed): two seeded users, an empty job queue, and an empty mail sink.

```bash
$ docker exec sl-runtime su postgres -c "psql -d simplelogin -c \"SELECT id, email FROM users ORDER BY id;\""
 id |          email
----+-------------------------
  1 | john@wick.com
  2 | winston@continental.com
(2 rows)

$ docker exec sl-runtime su postgres -c "psql -d simplelogin -c \"SELECT id,name,state FROM job ORDER BY id;\""
 id | name | state 
----+------+-------
(0 rows)

$ docker exec sl-runtime su postgres -c "psql -d simplelogin -tAc \"SELECT count(*) FROM alias WHERE user_id=1;\""
10

$ docker exec sl-runtime curl -s http://localhost:8025/api/v2/messages
{"total":0,"count":0,"start":0,"items":[]}
```

The seeded account `john@wick.com` / `password` comes from `flask dummy-data` `[server.py:L490-497]`, which calls `fake_data()` `[app/fake_data.py:L40]` seeding `email="john@wick.com"` `[app/fake_data.py:L45]` and `password="password"` `[app/fake_data.py:L47]`. The canonical bring‑up sequence is `alembic upgrade head && flask dummy-data && python3 server.py` `[CONTRIBUTING.md:L106]`, and the documented login is `john@wick.com / password` `[CONTRIBUTING.md:L109]`.

### Shared logging is initialized on import (`>>> init logging <<<`)

Every component imports `app/log.py`, which prints a one‑time marker on import and installs a shared stdout formatter under the logger name `SL`.

```bash
$ docker exec sl-runtime bash -lc 'cd /app && CONFIG=/app/.env /app/venv/bin/python -c "import app.log" 2>&1 | grep -n ">>> init logging <<<"'
7:>>> init logging <<<
```

That marker is `print(">>> init logging <<<")` at `[app/log.py:L67]`. The shared format string `[app/log.py:L12-15]` is:

```
%(asctime)s - %(name)s - %(levelname)s - %(process)d - "%(pathname)s:%(lineno)d" - %(funcName)s() - %(message_id)s - %(message)s
```

It is emitted to `sys.stdout` via `logging.StreamHandler(sys.stdout)` `[app/log.py:L41]`, the Werkzeug request logger is disabled (`log.disabled = True`) `[app/log.py:L70-71]`, the shortcuts `LOG.d/i/w/e` map to `debug/info/warning/exception` `[app/log.py:L74-77]`, and the logger is `LOG = _get_logger("SL")` `[app/log.py:L79]`. A useful consequence: **every SL log line embeds its own `"pathname:lineno"`**, so the runtime output itself confirms the `file:line` citations used throughout this report.

### Starting the three processes (stdout captured)

```bash
# Web server (production entry) — Dockerfile:L47
$ docker exec sl-runtime bash -lc 'cd /app && setsid /app/venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15 > /tmp/blitzy_obs/web.log 2>&1 < /dev/null &'

# Email handler — default --port 20381 (email_handler.py:L2399)
$ docker exec sl-runtime bash -lc 'cd /app && setsid /app/venv/bin/python email_handler.py > /tmp/blitzy_obs/email.log 2>&1 < /dev/null &'

# Job runner — 10s poll loop (job_runner.py:L329)
$ docker exec sl-runtime bash -lc 'cd /app && setsid /app/venv/bin/python job_runner.py > /tmp/blitzy_obs/job.log 2>&1 < /dev/null &'
```

`wsgi.py` exposes the callable Gunicorn loads (`from server import create_app` / `app = create_app()`), and `EXPOSE 7777` is declared at `[Dockerfile:L44]`. The captured startup output for each process is quoted in the Q1 section below.

---

## Q1 — "After the app starts, how can I tell the web server, email handler, and job runner are actually up and responding? What should I see in logs or the dashboard UI that confirms users can sign in and manage their aliases?"

There are three complementary signals per component: **startup log markers**, **a bound listening port**, and **a live response** (HTTP for the web server, an SMTP banner for the handler, the poll cadence for the runner). Then the **dashboard UI** confirms users can sign in and manage aliases.

### 1. Web server — startup markers, `/health`, `/live`, `/git`, bound port

**Claim: the web server boots and announces it is listening on 7777.** Captured from `web.log`:

```
[2026-07-01 23:53:32 +0000] [5253] [INFO] Starting gunicorn 20.0.4
[2026-07-01 23:53:32 +0000] [5253] [INFO] Listening at: http://0.0.0.0:7777 (5253)
[2026-07-01 23:53:32 +0000] [5253] [INFO] Using worker: sync
[2026-07-01 23:53:32 +0000] [5254] [INFO] Booting worker with pid: 5254
[2026-07-01 23:53:32 +0000] [5255] [INFO] Booting worker with pid: 5255
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/xxphmkwgnlsynadefwtz
Upload files to local dir
>>> init logging <<<
2026-07-01 23:53:33,041 - SL - DEBUG - 5255 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/luqdtiptgwapfxiyzgpa
Upload files to local dir
>>> init logging <<<
2026-07-01 23:53:33,043 - SL - DEBUG - 5254 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

*Why this confirms it:* these are the standard Gunicorn readiness markers — `Starting gunicorn`, `Listening at: http://0.0.0.0:7777`, the worker type, and one `Booting worker with pid` per worker (two workers, `5254` and `5255`, matching `-w 2` `[Dockerfile:L47]`). The version `gunicorn 20.0.4` matches the pinned dependency `gunicorn = "^20.0.4"` `[pyproject.toml:L66]`. Each worker then prints its module‑import banner (`>>> URL`, `MAX_NB_EMAIL_FREE_PLAN…`, `Paddle param not set`, the `GNUPGHOME` temp‑dir warning, `Upload files to local dir`) followed by `>>> init logging <<<` and an `SL`‑formatted `load words file` line — shown here for both workers (`5255` then `5254`), confirming the shared logger is live inside each worker.

**Claim: `GET /health` returns HTTP 200 with the body `success`.**

```bash
$ docker exec sl-runtime curl -s -o /dev/null -w 'HTTP %{http_code} | Content-Type: %{content_type} | Content-Length: %{size_download}\n' http://localhost:7777/health
HTTP 200 | Content-Type: text/html; charset=utf-8 | Content-Length: 7

$ docker exec sl-runtime curl -s http://localhost:7777/health
success
```

The response also carries the usual headers; to prove the response structure **without** publishing the signed session cookie, only the header *names* are printed (values withheld):

```bash
$ docker exec sl-runtime bash -lc "curl -s -D - -o /dev/null http://localhost:7777/health | tr -d '\r' | awk -F':' '/^HTTP/{print \$0; next} NF{print \$1}'"
HTTP/1.1 200 OK
Server
Date
Connection
Content-Type
Content-Length
Vary
Set-Cookie
```

*Why this confirms it:* the route is `@app.route("/health", methods=["GET"])` returning `return "success", 200` `[server.py:L213-215]`. The status is `HTTP 200`, and `Content-Length: 7` equals `len("success")`. A `Set-Cookie` header *name* is present (a signed Flask session, not a credential); its value is deliberately not printed — only header names are shown, so no secret is published.

**Claim: `GET /live` returns the body `live`.**

```bash
$ docker exec sl-runtime curl -s -o /dev/null -w 'HTTP %{http_code} | Content-Type: %{content_type} | Content-Length: %{size_download}\n' http://localhost:7777/live
HTTP 200 | Content-Type: text/html; charset=utf-8 | Content-Length: 4

$ docker exec sl-runtime curl -s http://localhost:7777/live
live
```

*Why this confirms it:* the route `@monitor_bp.route("/live")` returns `return "live"` `[app/monitor/views.py:L10-12]`; `Content-Length: 4` equals `len("live")`.

**Claim: `GET /git` returns the build SHA.**

```bash
$ docker exec sl-runtime curl -s -o /dev/null -w 'HTTP %{http_code} | Content-Type: %{content_type} | Content-Length: %{size_download}\n' http://localhost:7777/git
HTTP 200 | Content-Type: text/html; charset=utf-8 | Content-Length: 3

$ docker exec sl-runtime curl -s http://localhost:7777/git
dev
```

*Why this confirms it:* `@monitor_bp.route("/git")` returns the SHA1 build string `[app/monitor/views.py:L5-7]`; in this image the value is `dev`. The related `/exception` route `[app/monitor/views.py:L15-18]` deliberately raises `Exception("to make sure sentry works")` — it is a Sentry self‑test, **not** a liveness endpoint, and is named here only for completeness.

**Claim: port 7777 is bound and LISTENING.** Proven from the container's `/proc/net/tcp` (no `lsof`/`ss` available). Port `7777` is `0x1E61`, and the TCP `LISTEN` state is `0A`:

```bash
$ docker exec sl-runtime python3 -c 'print("0x1E61 =", 0x1E61)'
0x1E61 = 7777

$ docker exec sl-runtime bash -lc 'grep -i "1E61" /proc/net/tcp'
   1: 00000000:1E61 00000000:0000 0A 00000000:00000000 00:00000000 00000000     0        0 416578486 1 0000000000000000 100 0 0 10 0                 
   5: 0100007F:1E61 0100007F:AD40 06 00000000:00000000 03:00001728 00000000     0        0 0 3 0000000000000000                                      
   7: 0100007F:AD7A 0100007F:1E61 06 00000000:00000000 03:0000175F 00000000     0        0 0 3 0000000000000000                                      
   8: 0100007F:1E61 0100007F:AD62 06 00000000:00000000 03:00001746 00000000     0        0 0 3 0000000000000000                                      
   9: 0100007F:1E61 0100007F:AD58 06 00000000:00000000 03:0000173D 00000000     0        0 0 3 0000000000000000                                      
  10: 0100007F:AD40 0100007F:1E61 06 00000000:00000000 03:00001727 00000000     0        0 0 3 0000000000000000                                      
  11: 0100007F:1E61 0100007F:AD72 06 00000000:00000000 03:00001757 00000000     0        0 0 3 0000000000000000                                      
  12: 0100007F:1E61 0100007F:AD50 06 00000000:00000000 03:00001732 00000000     0        0 0 3 0000000000000000                                      
  13: 0100007F:AD70 0100007F:1E61 06 00000000:00000000 03:0000174F 00000000     0        0 0 3 0000000000000000                                      
  14: 0100007F:1E61 0100007F:AD7A 06 00000000:00000000 03:0000175F 00000000     0        0 0 3 0000000000000000                                      
```

*Why this confirms it:* row `1:` in state `0A` (TCP `LISTEN`) with local address `00000000:1E61` = `0.0.0.0:7777` proves the web server is bound and accepting on all interfaces. The remaining rows are in state `06` (`TIME_WAIT`) on loopback (`0100007F` = 127.0.0.1) — the just‑completed `curl` health checks above — additional evidence the port is actively serving.

### 2. Email handler — startup markers, bound port, SMTP banner

**Claim: the handler boots, listens on 20381, and starts the aiosmtpd controller.** Captured from `email.log`:

```
>>> init logging <<<
2026-07-01 23:53:35,203 - SL - DEBUG - 5277 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-01 23:53:35,690 - SL - INFO - 5277 - "/app/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-01 23:53:35,691 - SL - DEBUG - 5277 - "/app/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

*Why this confirms it:* `Listen for port 20381` is `LOG.i("Listen for port %s", args.port)` `[email_handler.py:L2403]` with the argparse default `--port 20381` `[email_handler.py:L2399]`; `Start mail controller 0.0.0.0 20381` is `LOG.d("Start mail controller %s %s", controller.hostname, controller.port)` `[email_handler.py:L2386]`, emitted right after `controller.start()` `[email_handler.py:L2385]` on the `Controller(MailHandler(), hostname="0.0.0.0", port=port)` created at `[email_handler.py:L2383]` inside `main()` `[email_handler.py:L2381]`. Note the log lines' own `pathname:lineno` fields read `"/app/email_handler.py:2403"` and `:2386` — the runtime output literally confirms the citations.

**Claim: port 20381 is bound, and the server answers with a `220` SMTP banner.** Port `20381` is `0x4F9D`:

```bash
$ docker exec sl-runtime python3 -c 'print("0x4F9D =", 0x4F9D)'
0x4F9D = 20381

$ docker exec sl-runtime bash -lc 'grep -i "4F9D" /proc/net/tcp'
   0: 00000000:4F9D 00000000:0000 0A 00000000:00000000 00:00000000 00000000     0        0 416698451 1 0000000000000000 100 0 0 10 0                 
   6: 0100007F:E9E6 0100007F:4F9D 06 00000000:00000000 03:00000987 00000000     0        0 0 3 0000000000000000                                      
```

The handler was then driven as an SMTP client. Because `swaks`/`nc` are absent, a small raw‑socket probe script was written into the container and run:

```bash
$ docker exec sl-runtime bash -lc 'cat > /tmp/blitzy_obs/smtp_probe.py <<PYEOF
import socket
s = socket.create_connection(("localhost", 20381), timeout=5)
print("220-banner:", repr(s.recv(1024).decode().strip()))
s.sendall(b"EHLO observer.local\r\n")
print("EHLO-resp :", repr(s.recv(1024).decode().splitlines()[0]))
s.sendall(b"QUIT\r\n"); s.close()
PYEOF
/app/venv/bin/python /tmp/blitzy_obs/smtp_probe.py'
220-banner: '220 b05f70854303 Python SMTP 1.4.2'
EHLO-resp : '250-b05f70854303'
```

*Why this confirms it:* row `0:` in state `0A` on `0x4F9D` proves the handler is listening on `0.0.0.0:20381` (row `6:` is a `TIME_WAIT` on loopback — `0100007F` = 127.0.0.1 — left by a local client connection to the port); the `220 … Python SMTP 1.4.2` greeting and the `250-` EHLO reply prove the aiosmtpd server is not just bound but actively speaking SMTP. (`Python SMTP 1.4.2` is the aiosmtpd runtime version, satisfying the pin `aiosmtpd = "^1.2"` `[pyproject.toml:L87]`.) Connecting a client and reading the `220` banner is the standard way to verify an aiosmtpd server.

### 3. Job runner — poll cadence on stdout

The runner logs nothing per cycle when the queue is empty (its work only executes when jobs exist), so liveness is shown by the **poll cadence** once jobs are present. As a labelled **timing probe**, three recognized jobs (`proton-welcome-1` for the real activated user 8) were enqueued one per ~10.5 s so each became eligible on a distinct poll; the runner picked each up on its own cycle (from `job.log`):

```
2026-07-02 00:02:48,509 - SL - DEBUG - 5293 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 22 proton-welcome-1 {'user_id': 8}>
2026-07-02 00:02:58,527 - SL - DEBUG - 5293 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 23 proton-welcome-1 {'user_id': 8}>
2026-07-02 00:03:08,546 - SL - DEBUG - 5293 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 24 proton-welcome-1 {'user_id': 8}>
```

Deltas between consecutive `Take job` timestamps, computed from the three timestamps above:

```
delta 1 (job->job): 10.018 s
delta 2 (job->job): 10.019 s
```

*Why this confirms it:* `Take job %s` is `LOG.d("Take job %s", job)` `[job_runner.py:L334]`, printed once per job the loop picks up; the ~10 s spacing is the `time.sleep(10)` at the bottom of the loop `[job_runner.py:L347]`. Observing **more than one** interval (two deltas of `10.018 s` and `10.019 s` across three cycles) demonstrates the cadence rather than inferring it. These are real, recognized `proton-welcome-1` jobs for the activated partner user 8 — not a throwaway payload — used here purely as a **timing** probe. The full *natural* enqueue → consume → `done` lifecycle for that user's job (including its `ready(0) → taken(1) → done(2)` DB state transition at `[job_runner.py:L344]`) is proven, with DB evidence, in Q3.

### 4. Dashboard UI — sign‑in and alias management signals

**Claim: an unauthenticated visitor is redirected to the login page.**

```bash
$ docker exec sl-runtime curl -s -o /dev/null -w "HTTP %{http_code} -> %{redirect_url}\n" http://localhost:7777/
HTTP 302 -> http://localhost:7777/auth/login
```

**Claim: signing in with the seeded credentials lands the user on the dashboard.** A cookie‑jar **file** (`cj.txt`) holds the session cookie and the parsed CSRF token across requests; neither value is ever printed, so no secret is published. The exact flow that was run:

```bash
$ docker exec sl-runtime bash -lc 'cat > /tmp/blitzy_obs/login_flow.sh <<"SH"
set -e
CJ=/tmp/blitzy_obs/cj.txt
BASE=http://localhost:7777
# 1) GET /auth/login -> session cookie saved to jar + CSRF token parsed from the page
curl -s -c "$CJ" "$BASE/auth/login" -o /tmp/blitzy_obs/login_page.html
CSRF=$(grep -oE "name=\"csrf_token\"[^>]*value=\"[^\"]+\"" /tmp/blitzy_obs/login_page.html | sed -E "s/.*value=\"([^\"]+)\".*/\1/")
# 2) POST credentials with the jar cookie + parsed CSRF; print only status + redirect target
echo -n "POST /auth/login  -> "
curl -s -b "$CJ" -c "$CJ" \
  --data-urlencode "email=john@wick.com" \
  --data-urlencode "password=password" \
  --data-urlencode "csrf_token=$CSRF" \
  -o /dev/null -w "HTTP %{http_code} -> %{redirect_url}\n" "$BASE/auth/login"
# 3) authenticated GET / -> redirected to the dashboard
echo -n "GET / (authed)    -> "
curl -s -b "$CJ" -o /dev/null -w "HTTP %{http_code} -> %{redirect_url}\n" "$BASE/"
# 4) GET /dashboard/ -> 200, saved to dashboard.html for the alias-list check below
echo -n "GET /dashboard/   -> "
curl -s -b "$CJ" -o /tmp/blitzy_obs/dashboard.html -w "HTTP %{http_code}\n" "$BASE/dashboard/"
SH
bash /tmp/blitzy_obs/login_flow.sh'
POST /auth/login  -> HTTP 302 -> http://localhost:7777/dashboard/
GET / (authed)    -> HTTP 302 -> http://localhost:7777/dashboard/
GET /dashboard/   -> HTTP 200
```

*Why this confirms it:* the sign‑in route is `@auth_bp.route("/login", methods=["GET", "POST"])` `[app/auth/views/login.py:L21]` / `def login()` `[app/auth/views/login.py:L25]`; on success the POST returns `HTTP 302 -> …/dashboard/`, and an already‑authenticated request to `/` is likewise sent to the dashboard by the root route `[server.py:L251-256]` (authenticated → `dashboard.index`, otherwise → `auth.login`). The session cookie and CSRF token live only in the `cj.txt` jar file and the `$CSRF` shell variable — neither is printed.

**Claim: the dashboard shows the user's aliases and a per‑user alias count — the alias‑management surface.** The dashboard HTML fetched in step 4 above (`dashboard.html`) was searched for the seeded aliases and the signed‑in identity:

```bash
$ docker exec sl-runtime bash -lc 'grep -oE "e[0-9]+@sl\.local" /tmp/blitzy_obs/dashboard.html | sort -u'
e0@sl.local
e1@sl.local
e2@sl.local

$ docker exec sl-runtime bash -lc 'grep -o "john@wick.com" /tmp/blitzy_obs/dashboard.html | head -1'
john@wick.com
```

Cross‑checked against the database (the same count the dashboard computes):

```bash
$ docker exec sl-runtime su postgres -c "psql -d simplelogin -tAc \"SELECT count(*) FROM alias WHERE user_id=1;\""
10
```

*Why this confirms it:* the dashboard renders the signed‑in identity (`john@wick.com`) and the seeded aliases `e0@sl.local`, `e1@sl.local`, `e2@sl.local` (sample aliases `e{i}@{EMAIL_DOMAIN}` with `EMAIL_DOMAIN=sl.local` `[example.env:L22]`). The per‑user count is `nb_alias = Alias.filter_by(user_id=user.id).count()` `[app/dashboard/views/index.py:L33]`, which equals the `10` measured directly in the DB. Seeing the alias list plus this count on an HTTP 200 dashboard is the UI signal that a user can sign in and manage aliases.

---

## Q2 — "Perform basic actions — create a new account, create an alias, and have that alias receive an email. What happens when a new account interacts with aliases or tries to receive mail? How does the system show these actions were handled correctly? What should I see at runtime that confirms expected behavior?"

### Action 1 — Create a new account

A fresh account was created through the same `User.create(...)` path the app uses. The exact observation script and the command that ran it (the app package needs `PYTHONPATH=/app` to import `server`):

```bash
$ docker exec sl-runtime bash -lc 'cat > /tmp/blitzy_obs/create_account.py <<"PYEOF"
from server import create_light_app
from app.models import User, Mailbox
from app.db import Session

with create_light_app().app_context():
    user = User.create(
        email="blitzy_newacct@example.com",
        name="Blitzy Test",
        password="password",
        activated=True,
        from_partner=False,
    )
    Session.commit()
    is_partner = bool(user.flags & User.FLAG_CREATED_FROM_PARTNER)
    mb = Mailbox.get(user.default_mailbox_id)
    print(f"Created account: {user!r}  activated={user.activated}  from_partner={is_partner}")
    print(f"Default verified mailbox: {mb.email} verified={mb.verified}")
    print(f"user_id={user.id}")
PYEOF
cd /app && PYTHONPATH=/app /app/venv/bin/python /tmp/blitzy_obs/create_account.py'
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/hpxetsvtkvugzdvkehfw
Upload files to local dir
>>> init logging <<<
2026-07-01 23:55:24,172 - SL - DEBUG - 5523 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-01 23:55:25,132 - SL - INFO - 5523 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-01 23:55:25,135 - SL - DEBUG - 5523 - "/app/app/models.py:647" - create() -  - Disable onboarding emails
Created account: <User 7 Blitzy Test blitzy_newacct@example.com>  activated=True  from_partner=False
Default verified mailbox: blitzy_newacct@example.com verified=True
user_id=7
```

*What happened and why it confirms correct handling:* `User.create` `[app/models.py:L601-668]` created `<User 7 …>` and, per `mb = Mailbox.create(user_id=user.id, email=user.email, verified=True)` `[app/models.py:L611]`, gave it a **verified default mailbox** equal to its own email (`blitzy_newacct@example.com verified=True`) — a new account is immediately able to receive forwarded mail.

**Notable, and reported honestly:** the log line `Disable onboarding emails` comes from the guard `if config.DISABLE_ONBOARDING: LOG.d("Disable onboarding emails"); return user` `[app/models.py:L646-648]`, which **returns early before** the three onboarding `Job.create(...)` calls at `[app/models.py:L650-665]`. In this runtime the value `DISABLE_ONBOARDING=true` is the exact runtime `/app/.env` line captured in the setup section above (`150:DISABLE_ONBOARDING=true` from that `grep`); because `/app/.env` is git‑ignored, the citable tracked source is the template line `[example.env:L150]`, and the code that consumes it is `DISABLE_ONBOARDING = "DISABLE_ONBOARDING" in os.environ` `[app/config.py:L401]` — truthy by mere presence. So **account creation enqueued zero onboarding jobs here** (confirmed by the empty job queue below). When onboarding is enabled, those jobs are created with `run_at=arrow.now().shift(days=1|2|3)` `[app/models.py:L654,L659,L664]`, i.e. scheduled 1–3 days out, which `get_jobs_to_run()` (window `run_at <= now + 10 min`, `[job_runner.py:L307-326]`) would not pick up during a short observation regardless. (The seeded interactive account for sign‑in remains `john@wick.com` / `password` `[app/fake_data.py:L45,L47]`.)

The two seeded users plus the new `<User 7>` and the still‑empty job queue were confirmed directly in the database:

```bash
$ docker exec sl-runtime su postgres -c "psql -d simplelogin -c \"SELECT id,email,activated FROM users ORDER BY id;\""
 id |           email            | activated 
----+----------------------------+-----------
  1 | john@wick.com              | t
  2 | winston@continental.com    | t
  7 | blitzy_newacct@example.com | t
(3 rows)

$ docker exec sl-runtime su postgres -c "psql -d simplelogin -c \"SELECT id,name,state,run_at FROM job ORDER BY id;\""
 id | name | state | run_at 
----+------+-------+--------
(0 rows)
```

*Why this confirms it:* the new row `7 | blitzy_newacct@example.com | t` is the persisted, activated account, and the `(0 rows)` job queue is direct evidence that standard account creation enqueued **no** background jobs under `DISABLE_ONBOARDING=true` — exactly matching the `Disable onboarding emails` early‑return above.

### Action 2 — Create an alias

Signed in as `john@wick.com` (reusing the cookie‑jar from Q1), a random alias was created from the dashboard. The alias count **before** creation was `10`:

```bash
$ docker exec sl-runtime su postgres -c "psql -d simplelogin -tAc \"SELECT count(*) FROM alias WHERE user_id=1;\""
10
```

The exact producing request — a `POST /dashboard/` with `form-name=create-random-email` plus the parsed CSRF token, carried by the cookie‑jar file so no secret is printed — and the flash it produced:

```bash
$ docker exec sl-runtime bash -lc 'cat > /tmp/blitzy_obs/create_alias.sh <<"SH"
set -e
CJ=/tmp/blitzy_obs/cj.txt
BASE=http://localhost:7777
# refresh the dashboard page to parse a fresh CSRF token
curl -s -b "$CJ" "$BASE/dashboard/" -o /tmp/blitzy_obs/dash_before.html
CSRF=$(grep -oE "name=\"csrf_token\"[^>]*value=\"[^\"]+\"" /tmp/blitzy_obs/dash_before.html | head -1 | sed -E "s/.*value=\"([^\"]+)\".*/\1/")
# POST the create-random-email form; follow the redirect to the resulting dashboard page
echo -n "POST /dashboard/ (create-random-email) -> "
curl -s -b "$CJ" -c "$CJ" -L \
  --data-urlencode "form-name=create-random-email" \
  --data-urlencode "csrf_token=$CSRF" \
  -o /tmp/blitzy_obs/dash_after.html -w "HTTP %{http_code}\n" "$BASE/dashboard/"
echo "--- flash on resulting page ---"
grep -oE "Alias [a-z0-9_]+@sl\.local has been created" /tmp/blitzy_obs/dash_after.html | head -1
SH
bash /tmp/blitzy_obs/create_alias.sh'
POST /dashboard/ (create-random-email) -> HTTP 200
--- flash on resulting page ---
Alias test_test850@sl.local has been created
```

**Server‑side log line (index.py:L110), captured from `web.log`:**

```
2026-07-01 23:56:08,333 - SL - DEBUG - 5254 - "/app/app/dashboard/views/index.py:110" - index() -  - create new random alias <Alias 18 test_test850@sl.local> for user <User 1 John Wick john@wick.com>
```

**Alias count incremented (10 → 11), and the new row persisted:**

```bash
$ docker exec sl-runtime su postgres -c "psql -d simplelogin -tAc \"SELECT count(*) FROM alias WHERE user_id=1;\""
11

$ docker exec sl-runtime su postgres -c "psql -d simplelogin -c \"SELECT id,email FROM alias WHERE user_id=1 ORDER BY id DESC LIMIT 1;\""
 id |         email         
----+-----------------------
 18 | test_test850@sl.local
(1 row)
```

*Why this confirms correct handling:* the random‑alias branch runs `alias = Alias.create_new_random(user=current_user, scheme=scheme)` `[app/dashboard/views/index.py:L104]`, sets its mailbox `alias.mailbox_id = current_user.default_mailbox_id` `[app/dashboard/views/index.py:L106]`, commits `[app/dashboard/views/index.py:L108]`, logs `LOG.d("create new random alias %s for user %s", …)` `[app/dashboard/views/index.py:L110]`, and flashes `flash(f"Alias {alias.email} has been created", "success")` `[app/dashboard/views/index.py:L111]`. The exact flash text `Alias test_test850@sl.local has been created` and the same alias `<Alias 18 test_test850@sl.local>` in the server log refer to one object, and the persisted count `nb_alias` `[app/dashboard/views/index.py:L33]` rose from 10 to 11 with row `18 | test_test850@sl.local` now present. (The alternate manual path is `@dashboard_bp.route("/custom_alias", methods=["GET", "POST"])` `[app/dashboard/views/custom_alias.py:L30]` / `def custom_alias()` `[app/dashboard/views/custom_alias.py:L34]`.)

### Action 3 — Have the alias receive an email

A message was sent to the new alias on port 20381 using Python's `smtplib` (`swaks`/`nc` are absent). The exact script and the command that ran it:

```bash
$ docker exec sl-runtime bash -lc 'cat > /tmp/blitzy_obs/send_inbound.py <<"PYEOF"
import smtplib
from email.mime.text import MIMEText
msg = MIMEText("Blitzy runtime observation: inbound delivery test to a SimpleLogin alias.")
msg["Subject"] = "Blitzy inbound test"
msg["From"] = "tester@example.com"
msg["To"] = "test_test850@sl.local"
with smtplib.SMTP("localhost", 20381, timeout=15) as s:
    print("sendmail returned:", s.sendmail("tester@example.com", ["test_test850@sl.local"], msg.as_string()))
PYEOF
/app/venv/bin/python /tmp/blitzy_obs/send_inbound.py'
sendmail returned: {}
```

*An empty dict from `sendmail` means every recipient was accepted (a 2xx result).*

**Handler `New message` line (email_handler.py:L2343‑2344) — grepped from `email.log`, quoted verbatim including the source typo `rctp tos`:**

```bash
$ docker exec sl-runtime bash -lc 'grep -F "New message, mail from" /tmp/blitzy_obs/email.log | tail -1'
2026-07-01 23:56:42,680 - SL - INFO - 5277 - "/app/email_handler.py:2343" - _handle() - ff7f9b3e-4a7a-4e6a-ae6a-47200967168f - New message, mail from tester@example.com, rctp tos ['test_test850@sl.local'] 
```

**Handler `Finish` line (email_handler.py:L2367‑2368) — grepped from `email.log`, with measured elapsed time and SMTP return code:**

```bash
$ docker exec sl-runtime bash -lc 'grep -F "Finish mail_from" /tmp/blitzy_obs/email.log | tail -1'
2026-07-01 23:56:42,885 - SL - INFO - 5277 - "/app/email_handler.py:2367" - _handle() - ff7f9b3e-4a7a-4e6a-ae6a-47200967168f - Finish mail_from tester@example.com, rcpt_tos ['test_test850@sl.local'], takes 0.20512962341308594 seconds with return code '250 Message accepted for delivery'<<===
```

*Why this confirms correct handling:* the inbound coroutine is `async def handle_DATA(self, server, session, envelope)` `[email_handler.py:L2289]`. The receipt log is `LOG.i("New message, mail from %s, rctp tos %s ", …)` `[email_handler.py:L2343-2344]` — the literal really is spelled **`rctp tos`** in the source, quoted here exactly. The completion log is `LOG.i("Finish mail_from %s, rcpt_tos %s, takes %s seconds with return code '%s'<<===", …)` `[email_handler.py:L2367-2368]`. The two **measured** values are the elapsed **`0.20512962341308594` seconds** and the return code **`250 Message accepted for delivery`** — a `2xx` code denotes success (a `5xx` would denote failure). The shared identifier `ff7f9b3e‑4a7a‑4e6a‑ae6a‑47200967168f` in both lines is the `%(message_id)s` field of the log format `[app/log.py:L12-15]`.

**End‑to‑end delivery landed in the mail sink (MailHog).** The raw MailHog API v2 message was read by a small extractor and its envelope plus key headers printed verbatim (MailHog stores the full RFC‑822 message under `Raw.Data`, parsed here with Python's `email` module):

```bash
$ docker exec sl-runtime bash -lc 'cat > /tmp/blitzy_obs/mailhog_show.py <<"PYEOF"
import json, urllib.request, email
d = json.load(urllib.request.urlopen("http://localhost:8025/api/v2/messages"))
print("MailHog total:", d["total"])
it = d["items"][0]
print("Envelope MAIL FROM:", it["Raw"]["From"])
print("Envelope RCPT TO  :", ", ".join(it["Raw"]["To"]))
m = email.message_from_string(it["Raw"]["Data"])
for hdr in ("Subject", "X-SimpleLogin-Type", "From", "To"):
    print(f"Header {hdr}:", m.get(hdr, ""))
PYEOF
/app/venv/bin/python /tmp/blitzy_obs/mailhog_show.py'
MailHog total: 1
Envelope MAIL FROM: sl.lmycyibufqqdemzwgu4tcns5.rdyoxocdmob2q@sl.local
Envelope RCPT TO  : john@wick.com
Header Subject: Blitzy inbound test
Header X-SimpleLogin-Type: Forward
Header From: "tester at example.com" <tester_at_example_com_zlgomrui@sl.local>
Header To: test_test850@sl.local
```

*Why this is the end‑to‑end proof:* the message the external sender addressed to the alias `test_test850@sl.local` (the `Header To`) was **forwarded by the handler to John's verified mailbox `john@wick.com`** (the envelope `RCPT TO`) and captured by the sink (`MailHog total: 1`). The envelope `MAIL FROM` is SimpleLogin's VERP return‑path `sl.lmycyibufqqdemzwgu4tcns5.rdyoxocdmob2q@sl.local`, the `X-SimpleLogin-Type: Forward` header marks it as a forward, and the `Header From` was rewritten to a reverse‑alias `tester_at_example_com_zlgomrui@sl.local` so a reply would route back through SimpleLogin. This is exactly what a new account interacting with an alias to receive mail should produce.

### Negative / edge behaviour (what a failure looks like)

**Bad credentials** are rejected with an explicit flash. The exact producing request (a fresh cookie‑jar, a deliberately wrong password) and the flash it produced:

```bash
$ docker exec sl-runtime bash -lc 'cat > /tmp/blitzy_obs/bad_login.sh <<"SH"
set -e
CJ=/tmp/blitzy_obs/cj_bad.txt
BASE=http://localhost:7777
# fresh jar + CSRF token from the login page
curl -s -c "$CJ" "$BASE/auth/login" -o /tmp/blitzy_obs/badlogin_page.html
CSRF=$(grep -oE "name=\"csrf_token\"[^>]*value=\"[^\"]+\"" /tmp/blitzy_obs/badlogin_page.html | sed -E "s/.*value=\"([^\"]+)\".*/\1/")
# POST a WRONG password; follow the redirect and grep the flash on the resulting page
echo -n "POST /auth/login (wrong password) -> "
curl -s -b "$CJ" -c "$CJ" -L \
  --data-urlencode "email=john@wick.com" \
  --data-urlencode "password=WRONGPASSWORD" \
  --data-urlencode "csrf_token=$CSRF" \
  -o /tmp/blitzy_obs/badlogin_after.html -w "HTTP %{http_code}\n" "$BASE/auth/login"
echo "--- flash on resulting page ---"
grep -oE "Email or password incorrect" /tmp/blitzy_obs/badlogin_after.html | head -1
SH
bash /tmp/blitzy_obs/bad_login.sh'
POST /auth/login (wrong password) -> HTTP 200
--- flash on resulting page ---
Email or password incorrect
```

*Why:* the login view flashes `flash("Email or password incorrect", "error")` `[app/auth/views/login.py:L49]` on a failed authentication (an already‑authenticated user is instead redirected to `dashboard.index` `[app/auth/views/login.py:L34]`). The `HTTP 200` (rather than a `302` redirect to the dashboard) plus the flash text is the runtime signal that authentication was refused.

The following inbound‑mail error signals were **not triggered** by the successful test above (which returned `250`); they are grounded in code and named here as the expected negative‑path responses (labeled *not directly observed*):

- **Recipient cannot receive mail** — `if not user.can_send_or_receive(): LOG.i(f"User {user} cannot receive emails")` `[email_handler.py:L563-564]`, returning `status.E207` (`"250 SL E207 No bounce report"` `[app/email/status.py:L12]`) when the sender is an ignorable bounce, otherwise `status.E504` (`"550 SL E504 Account disabled"` `[app/email/status.py:L41]`).
- **Reverse‑alias misuse** — `return status.E524` (`"550 SL E524 Wrong use of reverse-alias"` `[app/email/status.py:L62]`) at `[email_handler.py:L2307]` inside `handle_DATA`.
- **VERP error conditions** — `return status.E213` (`"250 SL E213 Unknown email ignored"` `[app/email/status.py:L21]`) at `[email_handler.py:L2318]`.

The `User`, `Alias`, `Contact`, and `Job` models that back all of this observed state are defined at `[app/models.py:L336]`, `[app/models.py:L1469]`, `[app/models.py:L1863]`, and `[app/models.py:L2683]` respectively.

---

## Q3 — "While the application is active, do the email handler and job runner automatically come online in the background to support email activity or data handling? What behavior suggests these internal components are functioning as intended across different situations?"

**Yes.** Both are launched as their own long‑lived OS processes and run unattended in the background. After the single startup command each, they required no further intervention across the whole investigation:

```bash
$ docker exec sl-runtime ps -o pid,etime,cmd -p 5253,5277,5293
    PID     ELAPSED CMD
   5253       10:22 /app/venv/bin/python /app/venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
   5277       10:20 /app/venv/bin/python email_handler.py
   5293       10:17 /app/venv/bin/python job_runner.py
```

*Why this confirms it:* the three PIDs are the web server (`5253`), the email handler (`5277`), and the job runner (`5293`) started once at the top of this session; ~10 minutes later they are all still alive with no restarts, i.e. they came online and stayed online in the background. They share one consistent database/config because **all three build the same lightweight app context**: `create_light_app()` is defined at `[server.py:L127]`, imported by the handler at `[email_handler.py:L177]` (used at `[email_handler.py:L2352]`) and by the runner at `[job_runner.py:L24]` (used at `[job_runner.py:L332]`). That shared context is why the alias created via the web dashboard in Q2 and the jobs enqueued in this section (below) are all seen against **one** database by the handler and the runner.

### Email handler — auto‑accepts and processes every message

The handler's `main()` keeps the aiosmtpd controller alive with a keepalive loop `while True: time.sleep(2)` `[email_handler.py:L2392-2393]`. It accepts inbound connections and processes each message with **no per‑message intervention** — demonstrated in Q2, where sending to the alias caused the `New message …` / `Finish … return code '250 Message accepted for delivery'` pair to appear on the handler's stdout automatically. That automatic pair is the behaviour that shows the handler is functioning.

### Job runner — polls and dispatches automatically, and does so "across different situations"

The main loop is: `while True:` `[job_runner.py:L330]` → `with create_light_app().app_context():` `[job_runner.py:L332]` → `for job in get_jobs_to_run():` `[job_runner.py:L333]` → `LOG.d("Take job %s", job)` `[job_runner.py:L334]` → `process_job(job)` `[job_runner.py:L342]` → `job.state = JobState.done.value` `[job_runner.py:L344]` → `time.sleep(10)` `[job_runner.py:L347]`.

**Situation A — idle (no jobs eligible):** the runner logs nothing per cycle when no job is eligible (the `for` body runs only when `get_jobs_to_run()` returns rows), yet it stays alive and keeps polling. Verified directly — after the last cadence job (`Job 24` at `00:03:08,546`) the runner went idle with no further eligible jobs; re‑checking its log, the last `Take job` line was **unchanged** (still `Job 24`), and the runner log held **zero** error markers:

```bash
$ docker exec sl-runtime bash -lc 'grep "Take job" /tmp/blitzy_obs/job.log | tail -1'   # while idle: last Take job unchanged
2026-07-02 00:03:08,546 - SL - DEBUG - 5293 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 24 proton-welcome-1 {'user_id': 8}>

$ docker exec sl-runtime bash -lc 'grep -cE " - ERROR - |Traceback|Unknown job name|Exception" /tmp/blitzy_obs/job.log'
0

$ docker exec sl-runtime bash -lc 'echo -n "web.log errors:   "; grep -cE " - ERROR - |Traceback" /tmp/blitzy_obs/web.log; echo -n "email.log errors: "; grep -cE " - ERROR - |Traceback" /tmp/blitzy_obs/email.log'
web.log errors:   0
email.log errors: 0
```

*This is the expected quiet‑but‑ready behaviour:* the runner polls silently on its 10‑second cadence and produces neither output nor errors until a job becomes eligible — and the web and email logs are likewise error‑free across the whole session.

**Situation B — a burst of jobs across consecutive cycles:** the three `Take job` lines quoted in Q1 (`Job 22`/`23`/`24` at `00:02:48` / `:58` / `00:03:08`) with measured **10.018 s** and **10.019 s** gaps show it dispatching work on the fixed 10‑second cadence `[job_runner.py:L347]`; all three then reached `done` (state 2), confirmed directly in the DB:

```bash
$ docker exec sl-runtime su postgres -c "psql -d simplelogin -c \"SELECT id,name,state FROM job WHERE id IN (22,23,24) ORDER BY id;\""
 id |       name       | state 
----+------------------+-------
 22 | proton-welcome-1 |     2
 23 | proton-welcome-1 |     2
 24 | proton-welcome-1 |     2
(3 rows)
```

**Situation C — a job that is the *natural consequence of account creation*, consumed end‑to‑end:** creating a **partner** account (`from_partner=True`) enqueues a `proton-welcome-1` job with `run_at=arrow.now()` at `[app/models.py:L625-628]` — this happens **before** the `DISABLE_ONBOARDING` guard `[app/models.py:L646-648]`, so it fires regardless of that flag and, being `run_at=now`, is immediately eligible (well inside the runner's `now + 10 min` window). The account was created and the job it enqueued was read straight back from the DB — atomically, before the runner's next poll:

```bash
$ docker exec sl-runtime bash -lc 'cat > /tmp/blitzy_obs/create_partner.py <<"PYEOF"
import arrow
from server import create_light_app
from app.models import User, Job
from app.db import Session

with create_light_app().app_context():
    user = User.create(
        email="blitzy_partner@example.com", name="Blitzy Partner",
        password="password", activated=True, from_partner=True,
    )
    Session.commit()
    is_partner = bool(user.flags & User.FLAG_CREATED_FROM_PARTNER)
    print(f"Created PARTNER account: {user!r}  from_partner={is_partner}  user_id={user.id}")
    j = Job.filter_by().order_by(Job.id.desc()).first()
    print(f"Naturally-enqueued job (atomically, before runner poll): id={j.id} name={j.name} state={j.state} payload={j.payload}")
    print(f"  run_at={j.run_at}  now={arrow.now()}")
PYEOF
cd /app && PYTHONPATH=/app /app/venv/bin/python /tmp/blitzy_obs/create_partner.py'
Created PARTNER account: <User 8 Blitzy Partner blitzy_partner@example.com>  from_partner=True  user_id=8
Naturally-enqueued job (atomically, before runner poll): id=15 name=proton-welcome-1 state=0 payload={'user_id': 8}
  run_at=2026-07-01T23:58:28.465721+00:00  now=2026-07-01T23:58:28.474057+00:00
```

The already‑running job runner then picked it up on its next poll and dispatched it through the correct `process_job` branch (from `job.log`):

```
2026-07-01 23:58:38,186 - SL - DEBUG - 5293 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 15 proton-welcome-1 {'user_id': 8}>
2026-07-01 23:58:38,193 - SL - DEBUG - 5293 - "/app/job_runner.py:293" - process_job() -  - Send proton welcome email to user <User 8 Blitzy Partner blitzy_partner@example.com>
```

…and the job's DB state transitioned to `done` (`state=2`) with `attempts=1`:

```bash
$ docker exec sl-runtime su postgres -c "psql -d simplelogin -c \"SELECT id,name,state,attempts FROM job WHERE id=15;\""
 id |       name       | state | attempts 
----+------------------+-------+----------
 15 | proton-welcome-1 |     2 |        1
(1 row)
```

*Why this is the natural‑consequence proof:* the enqueue was **not** performed by hand — `User.create(..., from_partner=True)` itself called `Job.create(name=config.JOB_SEND_PROTON_WELCOME_1, payload={"user_id": user.id}, run_at=arrow.now())` `[app/models.py:L625-628]` (`JOB_SEND_PROTON_WELCOME_1 = "proton-welcome-1"` `[app/config.py:L310]`). The runner then autonomously ran the full lifecycle: poll pickup `Take job` `[job_runner.py:L334]`, mark `taken` and `attempts += 1` `[job_runner.py:L337-341]`, execute the matching branch — `LOG.d("Send proton welcome email to user %s", user)` then `welcome_proton(user)` `[job_runner.py:L289-294]` (its `pathname` field reads `"/app/job_runner.py:293"`) — and finally set `job.state = JobState.done.value` `[job_runner.py:L344]`. The `state=2` (`JobState.done` `[app/models.py:L253-257]`) with a single `attempt` is the DB proof the job completed cleanly on the first try. Together, A/B/C show the runner behaving consistently whether idle, under a burst, or handling a job that account creation produced automatically.

**Natural consequence, the other half — standard‑account onboarding jobs are enqueued too, just scheduled days out.** A partner account produces an *immediately‑runnable* job; a *standard* account produces the three onboarding jobs — but with `run_at` one/two/three days in the future, so they sit deliberately outside the runner's short‑term window. To show they really are enqueued by account creation, `DISABLE_ONBOARDING` was **temporarily** commented out in the runtime `/app/.env` (a transient override, reverted immediately afterward), a standard account was created, and its jobs were read from the DB:

```bash
$ docker exec sl-runtime bash -lc 'grep -n "DISABLE_ONBOARDING" /app/.env'   # temporarily disabled
150:# DISABLE_ONBOARDING=true  # temporarily disabled to observe natural onboarding enqueue

$ docker exec sl-runtime su postgres -c "psql -d simplelogin -c \"SELECT id,name,state,run_at FROM job WHERE name LIKE 'onboarding%' ORDER BY id;\""
 id |     name     | state |           run_at           
----+--------------+-------+----------------------------
 19 | onboarding-1 |     0 | 2026-07-03 00:01:33.082699
 20 | onboarding-2 |     0 | 2026-07-04 00:01:33.082831
 21 | onboarding-4 |     0 | 2026-07-05 00:01:33.082902
(3 rows)
```

Creating the standard account (`<User 10 Blitzy Onboard blitzy_onboard@example.com>`) thus enqueued `onboarding-1`/`-2`/`-4` naturally, via the `Job.create(...)` calls at `[app/models.py:L651-665]` with `run_at=arrow.now().shift(days=1|2|3)` `[app/models.py:L654,L659,L664]` — hence the July 3/4/5 `run_at` values. When re‑queried after the running runner had completed at least one further 10‑second poll cycle (`time.sleep(10)` `[job_runner.py:L347]`), they were **still `ready` (state 0), not consumed**, because their `run_at` is far beyond the runner's `now + 10 min` window:

```bash
$ docker exec sl-runtime su postgres -c "psql -d simplelogin -c \"SELECT id,name,state FROM job WHERE name LIKE 'onboarding%' ORDER BY id;\""   # re-queried after a further poll cycle
 id |     name     | state 
----+--------------+-------
 19 | onboarding-1 |     0
 20 | onboarding-2 |     0
 21 | onboarding-4 |     0
(3 rows)
```

`DISABLE_ONBOARDING=true` was then restored in `/app/.env`:

```bash
$ docker exec sl-runtime bash -lc 'grep -n "DISABLE_ONBOARDING" /app/.env'   # restored
150:DISABLE_ONBOARDING=true
```

*What this establishes:* account creation **does** enqueue onboarding jobs as a natural consequence (three rows appeared with no manual `Job.create`), but by design they are scheduled 1–3 days ahead, so the runner correctly leaves them `ready` in the short term. The immediately‑runnable natural job — the partner `proton-welcome-1` above — is what demonstrates the complete enqueue → consume → `done` path within an observation window.

**Selection logic** — `get_jobs_to_run()` `[job_runner.py:L307-326]` returns jobs where `state == ready` (0) **or** a `taken` (1) job gone stale (`taken_at` older than the retry window and `attempts` under the max), **and** whose `run_at` is `NULL` or `<= now + 10 min` (`run_at_earliest = arrow.now().shift(minutes=+10)` `[job_runner.py:L312]`, filtered at `[job_runner.py:L323]`). That is exactly why the immediate (`run_at=now`) partner job was eligible at once, while the account‑creation onboarding jobs (scheduled days ahead) were left `ready`, as measured directly above.

**Full `process_job` dispatch inventory** `[job_runner.py:L188-304]` — the runner branches on `job.name`. Directly observed in this session: **`JOB_SEND_PROTON_WELCOME_1`** (`"proton-welcome-1"` `[app/config.py:L310]`) at `[job_runner.py:L289]` (Situation C). The remaining branches are grounded in code and were **not directly observed** in this run (described here for completeness):

| `job.name` | Line | Purpose |
|------------|------|---------|
| `JOB_ONBOARDING_1` = `"onboarding-1"` | `[job_runner.py:L189]` | send‑from‑alias onboarding email *(enqueued naturally above; scheduled +1 day)* |
| `JOB_ONBOARDING_2` = `"onboarding-2"` | `[job_runner.py:L198]` | mailbox onboarding email *(enqueued naturally above; scheduled +2 days)* |
| `JOB_ONBOARDING_4` = `"onboarding-4"` | `[job_runner.py:L207]` | PGP onboarding email *(enqueued naturally above; scheduled +3 days)* |
| `JOB_BATCH_IMPORT` | `[job_runner.py:L222]` | batch alias import |
| `JOB_DELETE_ACCOUNT` | `[job_runner.py:L226]` | account deletion |
| `JOB_DELETE_MAILBOX` | `[job_runner.py:L245]` | mailbox deletion |
| `JOB_DELETE_DOMAIN` | `[job_runner.py:L248]` | custom‑domain deletion |
| `JOB_SEND_USER_REPORT` | `[job_runner.py:L285]` | user‑data export |
| `JOB_SEND_PROTON_WELCOME_1` = `"proton-welcome-1"` | `[job_runner.py:L289]` | Proton welcome email *(observed — dispatched to `done` in Situation C)* |
| `JOB_SEND_ALIAS_CREATION_EVENTS` | `[job_runner.py:L295]` | alias‑creation events |
| *(fallback)* | `[job_runner.py:L304]` | `LOG.e("Unknown job name %s", job.name)` |

The onboarding constants are defined at `[app/config.py:L301]` (`onboarding-1`), `[app/config.py:L302]` (`onboarding-2`), and `[app/config.py:L304]` (`onboarding-4`).

---

## Coverage pass

Every named item and every "e.g. / such as / including" example across the three questions, and whether it was **directly observed at runtime (RT)** or **grounded in code (code)**.

**The three components / ports**
- Web server on **7777** — RT (gunicorn `Listening at: http://0.0.0.0:7777`; `/proc/net/tcp` `1E61` state `0A`).
- Email handler on **20381** — RT (`Listen for port 20381`; `Start mail controller 0.0.0.0 20381`; `4F9D` state `0A`; `220` banner).
- Job runner **10‑second** poll — RT (measured deltas 10.018 s, 10.019 s; `time.sleep(10)` `[job_runner.py:L347]`).

**Web endpoints**
- `/health` → `success` / `200` `[server.py:L213-215]` — RT. `/live` → `live` `[app/monitor/views.py:L10-12]` — RT. `/git` → `dev` `[app/monitor/views.py:L5-7]` — RT. `/exception` `[app/monitor/views.py:L15-18]` — code (Sentry self‑test; not a liveness route).

**Web entry / config**
- `create_app` `[server.py:L139]`, `create_light_app` `[server.py:L127]`, dev launcher `app.run(debug=True, port=7777)` `[server.py:L588]`, gunicorn CMD + `EXPOSE 7777` `[Dockerfile:L44,L47]`, `wsgi.py` (`create_app`), root redirect `[server.py:L251-256]`, `dummy-data` CLI `[server.py:L490-497]` — all cited; startup and redirect behaviour RT.

**Email handler**
- `main` `[email_handler.py:L2381]`, `Controller` `[email_handler.py:L2383]`, `controller.start()` `[email_handler.py:L2385]`, `Start mail controller` `[email_handler.py:L2386]` — RT. `--port` default 20381 `[email_handler.py:L2399]`, `Listen for port` `[email_handler.py:L2403]` — RT. Keepalive `[email_handler.py:L2392-2393]` — code. `handle_DATA` `[email_handler.py:L2289]`, `New message … rctp tos` `[email_handler.py:L2343-2344]` (verbatim typo), `Finish … return code` `[email_handler.py:L2367-2368]` — RT. `create_light_app` use `[email_handler.py:L177,L2352]` — code. cannot‑receive `[email_handler.py:L563-564]`, `E524` `[email_handler.py:L2307]`, `E213` `[email_handler.py:L2318]` — code (negative paths not triggered).

**Job runner**
- import `[job_runner.py:L24]`, `while True` `[job_runner.py:L330]`, app_context `[job_runner.py:L332]`, `for job` `[job_runner.py:L333]` — code. `Take job` `[job_runner.py:L334]`, `process_job` `[job_runner.py:L342]`, `done` `[job_runner.py:L344]`, `time.sleep(10)` `[job_runner.py:L347]` — RT. `get_jobs_to_run` `[job_runner.py:L307-326]` — code. `process_job` branches `[job_runner.py:L188-304]` + `Unknown job name` `[job_runner.py:L304]` — `proton-welcome-1` `[job_runner.py:L289]` dispatched to `done` RT; `onboarding-1`/`-2`/`-4` naturally enqueued RT but not dispatched (scheduled days out); all others code.

**User flows / seed**
- login route/def `[app/auth/views/login.py:L21,L25]` — RT. authed redirect `[app/auth/views/login.py:L34]` — code. bad‑creds flash `[app/auth/views/login.py:L49]` — RT. `nb_alias` `[app/dashboard/views/index.py:L33]` — RT (10 then 11). random‑alias block `[app/dashboard/views/index.py:L104-111]` + success flash `[app/dashboard/views/index.py:L111]` — RT. custom alias `[app/dashboard/views/custom_alias.py:L30-34]` — code. seed `john@wick.com` / `password` `[app/fake_data.py:L45,L47]` — RT. models `User/Alias/Contact/Job` `[app/models.py:L336,L1469,L1863,L2683]` — code.

**Logging**
- format `[app/log.py:L12-15]`, stdout handler `[app/log.py:L41]`, `>>> init logging <<<` `[app/log.py:L67]`, werkzeug disabled `[app/log.py:L70-71]`, `LOG.d/i/w/e` `[app/log.py:L74-77]`, `SL` logger `[app/log.py:L79]` — marker + format + `SL` name RT; the rest code.

**Config / run**
- `URL` `[example.env:L6]`, `NOT_SEND_EMAIL` `[example.env:L19]`, `EMAIL_DOMAIN` `[example.env:L22]`, `DB_URI` `[example.env:L75]`, `POSTFIX_PORT` `[example.env:L154]`, run sequence `[CONTRIBUTING.md:L106]`, login `[CONTRIBUTING.md:L109]`, MailHog API `:8025` (UI mapped to host `:1080`), dep versions (project `name` `[pyproject.toml:L47]`, `gunicorn = "^20.0.4"` `[pyproject.toml:L66]`, `aiosmtpd = "^1.2"` `[pyproject.toml:L87]`) — cited; service liveness (Postgres/Redis/MailHog) RT.

**"e.g. / such as" example clauses (treated as required)**
- "web server, email handler, and job runner" — each answered individually in Q1/Q3.
- "create a new account, create an alias, and have that alias receive an email" — each answered individually in Q2.
- "in logs or the dashboard UI" — logs (startup + handler + runner) and dashboard UI (sign‑in redirect, alias list, `nb_alias`) both answered in Q1.
- "across different situations" — idle / burst / single‑job situations answered in Q3.

### Explicitly flagged — not verified by running

1. **Standard‑account onboarding jobs under `DISABLE_ONBOARDING=true` are not consumed short‑term (by design).** With the runtime flag set (`150:DISABLE_ONBOARDING=true` from the `/app/.env` grep in the setup section; the tracked template line is `[example.env:L150]`, consumed by `DISABLE_ONBOARDING = "DISABLE_ONBOARDING" in os.environ` `[app/config.py:L401]`), standard account creation hits the early return at `[app/models.py:L646-648]` — the `Disable onboarding emails` log was observed and the job queue stayed empty (Q2, Action 1). With onboarding **temporarily enabled**, standard account creation **did** naturally enqueue `onboarding-1`/`-2`/`-4` (Q3, "the other half"), but with `run_at` 1–3 days out they fall outside the runner's `now + 10 min` window `[job_runner.py:L312,L323]` and so are not consumed within a short observation. The immediately‑runnable natural job that **was** consumed end‑to‑end is the partner `proton-welcome-1` (Q3, Situation C) — a genuine account‑creation consequence, not a hand‑enqueued job.
2. **`process_job` branch coverage.** The **`JOB_SEND_PROTON_WELCOME_1`** branch (`"proton-welcome-1"`) at `[job_runner.py:L289]` **was directly observed** — dispatched to `done` in Q3, Situation C. The three onboarding branches (`onboarding-1`/`-2`/`-4`) were **naturally enqueued** but **not dispatched** because they are scheduled 1–3 days out. The remaining branches (batch import, delete account/mailbox/domain, user report, alias‑creation events, and the `Unknown job name` fallback `[job_runner.py:L304]`) were **not directly observed**; they are described from source with their `file:line`.
3. **Inbound negative paths** (`E504`/`E207` cannot‑receive, `E524` reverse‑alias, `E213` VERP) were **not triggered** — the successful test returned `250 Message accepted for delivery`. Their code and status literals are cited from `email_handler.py` and `app/email/status.py`.
4. **The `proton-welcome-1` branch's outbound email was not separately confirmed in MailHog** — the branch‑dispatch log `[job_runner.py:L293]` proves the branch executed and the DB showed the job reaching `done` (Q3, Situation C), but `welcome_proton()` **early‑returns when the user has no Proton communication address** (`comm_email, _, _ = user.get_communication_email()` `[job_runner.py:L108]`; `if not comm_email: return` `[job_runner.py:L109-110]`), which is the case for this locally‑created test user; when it does send, it uses `ignore_smtp_error=True` `[job_runner.py:L126]`. Accordingly **only the branch execution and the `done` transition are claimed for that job, not a MailHog delivery.** The Q2 inbound‑forward message *was* confirmed in MailHog.
5. **Runtime service versions differ from the canonical docs** — observed **PostgreSQL 15.13** / **Redis 7.0.15** vs. the Postgres 13 / Redis v6 referenced in `CONTRIBUTING.md`; this does not affect any behaviour cited above.

---

### Cleanup (verified)

After evidence capture the runtime was torn down and the temporary state removed: the temporary accounts (`User 7`/`8`/`10`) were deleted (cascading their aliases and jobs), the temporary random alias on `john@wick.com` was deleted, the MailHog sink was cleared (`DELETE /api/v1/messages`), the temporary `DISABLE_ONBOARDING` override was reverted, the three processes were stopped, and the `/tmp/blitzy_obs` script directory was removed. The clean end‑state was then verified directly — each check with its exact command and verbatim output:

**Runtime data restored to the seeded baseline** (temporary `User 7`/`8`/`10`, their jobs, and the temporary alias all gone):

```bash
$ docker exec sl-runtime su postgres -c "psql -d simplelogin -c \"SELECT id,email FROM users ORDER BY id;\""
 id |          email          
----+-------------------------
  1 | john@wick.com
  2 | winston@continental.com
(2 rows)

$ docker exec sl-runtime su postgres -c "psql -d simplelogin -tAc \"SELECT count(*) FROM job;\""
0

$ docker exec sl-runtime su postgres -c "psql -d simplelogin -tAc \"SELECT count(*) FROM alias WHERE user_id=1;\""
10

$ docker exec sl-runtime su postgres -c "psql -d simplelogin -tAc \"SELECT count(*) FROM deleted_alias;\""
0
```

**Mail sink emptied and the config override reverted:**

```bash
$ docker exec sl-runtime bash -lc 'curl -s http://localhost:8025/api/v2/messages | head -c 200; echo'
{"total":0,"count":0,"start":0,"items":[]}

$ docker exec sl-runtime bash -lc 'grep -n "DISABLE_ONBOARDING" /app/.env'
150:DISABLE_ONBOARDING=true
```

**Temporary scripts removed and the processes stopped (both ports free):**

```bash
$ docker exec sl-runtime bash -lc 'ls -d /tmp/blitzy_obs 2>/dev/null && echo "STILL PRESENT" || echo "/tmp/blitzy_obs removed"'
/tmp/blitzy_obs removed

$ docker exec sl-runtime bash -lc 'ps -eo pid,cmd | grep -E "gunicorn wsgi:app|email_handler.py|job_runner.py" | grep -v grep || echo "(no live SL processes)"'
(no live SL processes)

$ docker exec sl-runtime bash -lc 'python3 - <<"PY"
def listens(hexport):
    n=0
    with open("/proc/net/tcp") as f:
        next(f)
        for ln in f:
            p=ln.split()
            if p[1].split(":")[1].upper()==hexport and p[3]=="0A": n+=1
    return n
print("7777 LISTEN rows:", listens("1E61"))
print("20381 LISTEN rows:", listens("4F9D"))
PY'
7777 LISTEN rows: 0
20381 LISTEN rows: 0
```

**Repository read‑only — the only change is this document, and no temporary artifact was left behind:**

```bash
$ git status --porcelain
 M blitzy/documentation/app_2cd6ee777f8c.md

$ git status --porcelain --ignored | grep -iE "blitzy_obs|create_account|send_inbound|create_partner|create_alias|bad_login|mailhog_show|\.log$" || echo "(no temp artifacts tracked/untracked in repo)"
(no temp artifacts tracked/untracked in repo)
```

*Why this proves the read‑only + cleanup obligation is met:* the database is back to exactly the two seeded users with an empty job queue, `john@wick.com`'s alias count is back to its seeded `10`, the `deleted_alias` trash table is empty, MailHog holds `0` messages, `/app/.env` is back to `150:DISABLE_ONBOARDING=true`, no SimpleLogin process is running and both ports are free, and `git status` shows the repository's **only** change is this single document with **no** stray temporary scripts or logs.
