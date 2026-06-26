# Verifying a Locally‑Running SimpleLogin Deployment

> **Source branch:** `app_2cd6ee777f8c` · **git HEAD:** `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`
> **Audience:** A newcomer to the SimpleLogin codebase who has just started the system and wants to confirm it is actually working.
> **Method:** Every answer below is grounded in the **source code as the single source of truth** (cited as `path:line`) **and** corroborated by **live runtime observation** captured from a running deployment (a reproducible command plus the signal it produced). Nothing here is inferred without evidence.

This document answers three questions a newcomer typically asks after bringing the project up:

1. **Service Health (Q1)** — How do I tell that the three runtime components — the **web server**, the **email handler**, and the **job runner** — are up and responding? What should I see in the logs or the dashboard to confirm users can sign in and manage aliases?
2. **Basic User Actions (Q2)** — Once it is running, how do I create an account, create an alias, and have that alias receive an email? What concrete runtime signals (HTTP responses, database rows, log lines, dashboard state) prove these actions were handled correctly?
3. **Background Components (Q3)** — Do the email handler and job runner come online "in the background" to support email activity while the app is active, and what behavior demonstrates they function as intended?

Each section gives **the answer**, **the code evidence**, **the observed runtime signal**, and **the rationale** ("why this proves it").

> **A note on how the logs cite themselves.** SimpleLogin's logger format embeds the call site `"%(pathname)s:%(lineno)d"` in every line ([`app/log.py:12‑14`](#citations-appendix)). That means the runtime log output literally prints the source file and line that emitted it. Throughout this document you will see observed log lines such as `"/app/email_handler.py:2386"` — these are not annotations added by hand; they are emitted by the running program, so the runtime evidence and the code citation are one and the same.

---

## Table of Contents

- [Section 0 — Overview & How It Is Run Locally](#section-0--overview--how-it-is-run-locally)
  - [0.1 The multi‑process model](#01-the-multi-process-model)
  - [0.2 Bringing the system up locally](#02-bringing-the-system-up-locally)
  - [0.3 The `DB_URI` port nuance (resolved by observation)](#03-the-db_uri-port-nuance-resolved-by-observation)
  - [0.4 Production launch & supporting daemons](#04-production-launch--supporting-daemons)
  - [0.5 The centralized logger](#05-the-centralized-logger)
- [Q1 — Service Health](#q1--service-health)
  - [Q1.1 Web server (port 7777)](#q11-web-server-port-7777)
  - [Q1.2 Email handler (port 20381)](#q12-email-handler-port-20381)
  - [Q1.3 Job runner](#q13-job-runner)
- [Q2 — Basic User Actions](#q2--basic-user-actions)
  - [Q2.1 Create an account (register → activate)](#q21-create-an-account-register--activate)
  - [Q2.2 Create an alias](#q22-create-an-alias)
  - [Q2.3 Have the alias receive an email](#q23-have-the-alias-receive-an-email)
  - [Q2.4 The critical local‑mode nuance: `NOT_SEND_EMAIL`](#q24-the-critical-local-mode-nuance-not_send_email)
- [Q3 — Background Components](#q3--background-components)
- [Consolidated Rationale / Thinking Thread](#consolidated-rationale--thinking-thread)
- [Test Data Cleanup & Non‑Persistence](#test-data-cleanup--non-persistence)
- [Secrets Handling Note](#secrets-handling-note)
- [Citations Appendix](#citations-appendix)

---

## Section 0 — Overview & How It Is Run Locally

### 0.1 The multi‑process model

SimpleLogin is a **Python/Flask monolith** with a **multi‑process runtime**. The single most important thing for a newcomer to internalize is this:

> **The web server, the email handler, and the job runner are three *independent, long‑running processes*. None of them spawns the others. Each is started explicitly and then enters its own loop.**

This is not an assumption — it is visible directly in the code. There are separate top‑level entry‑point scripts at the repository root, each with its own `__main__`/`main()` and its own infinite loop:

| Process | Entry point | What it does | Loop |
|---|---|---|---|
| **Web server** | `server.py` (local) / `wsgi.py` (gunicorn) | Serves the Flask app, dashboard, API | Gunicorn worker request loop / `app.run()` |
| **Email handler** | `email_handler.py` | aiosmtpd SMTP listener on `0.0.0.0:20381` | `while True: time.sleep(2)` ([`email_handler.py:2392‑2393`](#citations-appendix)) |
| **Job runner** | `job_runner.py` | Polls the `job` table and processes jobs | `while True: ... time.sleep(10)` ([`job_runner.py:330,347`](#citations-appendix)) |

The web entry point only registers blueprints and runs the Flask/WSGI app — it contains **no code that launches** `email_handler.py` or `job_runner.py`. We verify this precisely in [Q3](#q3--background-components).

```mermaid
flowchart TD
    subgraph Verify["Q1: Service Health Checks"]
        H1["GET /health -> 'success' 200<br/>server.py:213-215"]
        H2["Email handler startup log<br/>'Listen for port 20381'<br/>email_handler.py:2403 + 2386"]
        H3["Job runner poll loop<br/>'Take job ...' every 10s<br/>job_runner.py:329-347"]
    end

    subgraph Flow["Q2: Basic User Actions"]
        A1["Register /register<br/>app/auth/views/register.py:31"] --> A2["Activate /activate<br/>app/auth/views/activate.py:13"]
        A2 --> A3["Create alias<br/>create_new_random()<br/>app/dashboard/views/index.py:104"]
        A3 --> A4["Inbound mail to alias :20381"]
        A4 --> A5["handle_forward() writes EmailLog<br/>email_handler.py:536"]
    end

    subgraph Bg["Q3: Independent Background Processes"]
        P1["Web server :7777 (PID master)"]
        P2["Email handler :20381 (own PID)"]
        P3["Job runner (10s poll, own PID)"]
        P1 -. "does NOT spawn" .-> P2
        P1 -. "does NOT spawn" .-> P3
    end

    Verify --> Doc["This document<br/>(answers + rationale + citations)"]
    Flow --> Doc
    Bg --> Doc
```

### 0.2 Bringing the system up locally

The authoritative local‑run procedure is `CONTRIBUTING.md`. The condensed sequence is:

```bash
# 1) Configuration
cp example.env .env          # then set DB_URI (see 0.3 for the port nuance)

# 2) Dependencies: start PostgreSQL 13 and Redis
#    (PostgreSQL is the primary datastore; Redis backs rate limiting / job support)

# 3) Migrate the schema, seed demo data, and start the web server
alembic upgrade head && flask dummy-data && python3 server.py     # CONTRIBUTING.md:106

# 4) In separate terminals/containers, start the other two processes
python email_handler.py      # binds 0.0.0.0:20381   (CONTRIBUTING.md:212)
python job_runner.py         # 10s poll loop         (CONTRIBUTING.md:228)
```

Then open `http://localhost:7777` and sign in with the seeded account **`john@wick.com` / `password`** ([`CONTRIBUTING.md:109`](#citations-appendix)). That account is created by `flask dummy-data` already **activated** and as an **admin** ([`app/fake_data.py:44‑49`](#citations-appendix)), so you can exercise alias/email flows immediately without going through registration first.

> In the Docker‑based environment used to capture the observations in this document, the three processes were launched inside the application container and each writes to its own log file (`/app/web.log`, `/app/email_handler.log`, `/app/job_runner.log`); in a self‑host deployment the same lines are read with `docker logs <container>` per `docs/troubleshooting.md`. SimpleLogin's logger writes to **stdout** ([`app/log.py:41`](#citations-appendix)), so whichever way the process is supervised, the container runtime captures the lines.

### 0.3 The `DB_URI` port nuance (resolved by observation)

A newcomer following the docs literally will hit a genuine **inconsistency in the documented PostgreSQL port**. Three places disagree:

| Source | Value | Port |
|---|---|---|
| `example.env:75` (the default `DB_URI`) | `postgresql://myuser:mypassword@localhost:5432/simplelogin` | **5432** |
| `CONTRIBUTING.md:94` (instruction text: "edit `DB_URI` to") | `…@localhost:35432/…` | **35432** |
| `CONTRIBUTING.md:100` (the `docker run` command) | `docker run … -p 15432:5432 postgres:13` | host **15432** → container 5432 |

**What actually worked (observed):** In the running deployment, the application's *effective* `DB_URI` was **`postgresql://myuser:<password>@sl-db:5432/simplelogin`** — i.e., it connects to the PostgreSQL **container by its service name `sl-db` on the standard internal port `5432`**. This is because `app/config.py` calls `load_dotenv()` **without** an `override` argument ([`app/config.py:69‑71`](#citations-appendix)); python‑dotenv's default `override=False` behavior means an environment variable injected by the container runtime (`-e DB_URI=…@sl-db:5432/…`) **wins over** the value copied from `example.env`. The host‑port numbers in the docs (5432 / 35432 / 15432) only matter when PostgreSQL is reached over a host port mapping; in a Docker‑network topology the app reaches the database container directly on `5432`.

> **Takeaway for a newcomer:** don't blindly copy one of the three port numbers. Confirm which host/port your PostgreSQL is actually reachable on and set `DB_URI` to match. If you run the app and the database in the same Docker network, address the DB container by name on `5432`.

### 0.4 Production launch & supporting daemons

In production the web app is served by Gunicorn rather than the Flask dev server:

- **Gunicorn command:** `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` ([`Dockerfile:47`](#citations-appendix)). `wsgi.py` exposes `app = create_app()`.
- **Base image:** `python:3.10` ([`Dockerfile:8`](#citations-appendix)); a front‑end asset build stage uses `node:10.17.0-alpine` ([`Dockerfile:2`](#citations-appendix)).
- **Exposed port:** `EXPOSE 7777` ([`Dockerfile:44`](#citations-appendix)).

Beyond the three core processes, SimpleLogin's documented architecture includes additional **background daemons**, each also launched explicitly or on a schedule (again, *not* spawned by the web app):

- **`cron.py`** — scheduled maintenance, invoked per‑job by `yacron` according to `crontab.yml` / `crontab-all-hosts.yml` (e.g. entries of the form `python /code/cron.py -j <job>`).
- **`event_listener.py`** — a PostgreSQL `LISTEN`/`NOTIFY` consumer with a `Mode` enum of `DEAD_LETTER` and `LISTENER` ([`event_listener.py:15‑26`](#citations-appendix)).
- **`monitoring.py`** — an infrastructure‑metrics daemon on a 60‑second loop (`sleep(60)`, [`monitoring.py:171`](#citations-appendix)).

Together these correspond to the project's documented six‑process model: Web App, SMTP Handler, Job Runner, Cron Scheduler, Event Listener, and a Monitoring Daemon.

### 0.5 The centralized logger

All processes use a single centralized logger, `LOG`, defined in `app/log.py`. Understanding its format is what lets a newcomer *read* the health signals:

- **Writes to stdout** via `logging.StreamHandler(sys.stdout)` ([`app/log.py:41`](#citations-appendix)), with UTC timestamps (`converter = time.gmtime`).
- **Format** ([`app/log.py:12‑14`](#citations-appendix)) includes: timestamp, logger name (`SL`), level, process id, the **call site** `"pathname:lineno"`, the function name, a **message‑id** correlation token, and the message.
- **Message‑id correlation** via `set_message_id` ([`app/log.py:22`](#citations-appendix)) — every log line emitted while processing one email shares the same id, so you can follow a single message end‑to‑end (demonstrated in [Q2.3](#q23-have-the-alias-receive-an-email)).
- **Colorized output** via `coloredlogs` when `COLOR_LOG` is set ([`app/log.py:61‑62`](#citations-appendix)). Note that `COLOR_LOG` is **commented out** in `example.env` (`# COLOR_LOG=true`, [`example.env:16`](#citations-appendix)), but the local dev entry point `local_main()` forces `config.COLOR_LOG = True` ([`server.py:573`](#citations-appendix)) — so running `python3 server.py` locally yields colored logs even though the env var is unset.

A real captured line looks like this (note the embedded `server.py:284` call site and empty message‑id for a normal web request):

```text
2026-06-26 18:43:25,265 - SL - DEBUG - 77 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.0214540958404541
```

---

## Q1 — Service Health

**Question restated:** How do I confirm the web server, the email handler, and the job runner are up and responding — and that users can sign in and manage aliases?

### Q1.1 Web server (port 7777)

**Answer.** The canonical, application‑level liveness probe is **`GET /health`**, which returns the body `"success"` with HTTP `200`. UI reachability (the `/` redirect, the login page, the dashboard) confirms that the sign‑in and alias‑management surfaces are live. Per‑request activity for real routes is logged by an `after_request` hook.

**Code evidence.**

- The health endpoint is defined in `server.py`, **not** in the monitor blueprint:
  ```python
  # server.py:213-215
  @app.route("/health", methods=["GET"])
  def healthcheck():
      return "success", 200
  ```
- The index route redirects based on authentication — authenticated users to the dashboard, anonymous users to the login page ([`server.py:250‑255`](#citations-appendix)):
  ```python
  @app.route("/", methods=["GET", "POST"])
  def index():
      if current_user.is_authenticated:
          return redirect(url_for("dashboard.index"))
      else:
          return redirect(url_for("auth.login"))
  ```
- Per‑request logging is emitted by the `after_request` hook via `LOG.d("%s %s %s %s %s, takes %s", …)` ([`server.py:284‑292`](#citations-appendix)).
- Supplementary monitor endpoints are mounted by the monitor blueprint at `url_prefix="/"` ([`app/monitor/base.py:3`](#citations-appendix)): `/git` → build SHA1 ([`app/monitor/views.py:5‑7`](#citations-appendix)), `/live` → `"live"` ([`app/monitor/views.py:10‑12`](#citations-appendix)), and `/exception` → deliberately raises to test Sentry ([`app/monitor/views.py:15‑18`](#citations-appendix)).
- The full blueprint set registered on the app is `auth`, `monitor`, `dashboard`, `developer`, `phone`, `oauth` (mounted at **both** `/oauth` and `/oauth2`), `onboarding`, `discover`, `internal`, and `api` ([`server.py:233‑246`](#citations-appendix)) — so a `200` from the app means this entire routing surface is mounted.

**Observed runtime signal (reproducible).**

```bash
$ curl -i http://localhost:7777/health
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
...
success                       # response body

$ curl -i http://localhost:7777/
HTTP/1.1 302 FOUND
Location: http://localhost:7777/auth/login

$ curl http://localhost:7777/live ;  # -> live
$ curl http://localhost:7777/git  ;  # -> dev   (the build SHA1; "dev" in this build)
```

Loading `http://localhost:7777/` in a browser redirected to `/auth/login` (page title *"Login | SimpleLogin"*) and, after signing in as `john@wick.com`, landed on `/dashboard/` showing the alias‑management UI (the **"+ New Custom Alias"** and **"Random Alias"** controls — see [Q2.2](#q22-create-an-alias)). That round trip is itself proof that the sign‑in and alias‑management surfaces are reachable.

**⚠ Accuracy refinement (a) — `/health` produces NO request‑log line.** `/health` is **deliberately excluded** from the `after_request` logging block (`not request.path.startswith("/health")`, [`server.py:281`](#citations-appendix)) and from the profiler's ignore list ([`server.py:195`](#citations-appendix)). So a health probe returns `200` but emits **no** per‑request log line. This was verified directly: after probing `/health` several times and `/auth/login` once, the web log contained **0** lines mentioning `GET /health` and **4** mentioning `GET /auth/login`:

```text
# /app/web.log — a real route IS logged (note the self-cited server.py:284):
2026-06-26 18:43:25,265 - SL - DEBUG - 77 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.0214540958404541
# …and there is NO corresponding "GET /health" line, by design.
```

> **Do not mistake the absence of a `/health` log line for a malfunction.** It is expected behavior — health‑probe noise is intentionally suppressed.

**Rationale (why this proves it).** `/health` returns a constant `("success", 200)` with no side effects and no database access. Therefore a `200` from it proves the WSGI app is *routing requests* and a Gunicorn worker is *alive and serving* — which is exactly what a liveness probe should isolate. The `302 → /auth/login` from `/` and a successful login landing on `/dashboard/` then prove the higher‑level claim a newcomer cares about: **sign‑in works and alias management is reachable.** In production, Gunicorn's own boot lines ("Starting gunicorn", "Listening at", "Booting worker with pid …") are the process‑level signal; `/health` is the canonical app‑level one.

### Q1.2 Email handler (port 20381)

**Answer.** The email handler is up when its startup log shows the aiosmtpd controller bound to `0.0.0.0:20381`. Two specific log lines confirm it, and the port answers the SMTP greeting.

**Code evidence.**

- `main(port)` builds and starts the controller and logs the "Start mail controller" line:
  ```python
  # email_handler.py:2381-2393
  def main(port: int):
      """Use aiosmtpd Controller"""
      controller = Controller(MailHandler(), hostname="0.0.0.0", port=port)   # :2383
      controller.start()                                                       # :2385
      LOG.d("Start mail controller %s %s", controller.hostname, controller.port)  # :2386
      ...
      while True:
          time.sleep(2)                                                        # :2392-2393
  ```
- The `__main__` block defaults the port to `20381` ([`email_handler.py:2399`](#citations-appendix)) and logs the "Listen for port" line just before calling `main()`:
  ```python
  # email_handler.py:2403-2404
  LOG.i("Listen for port %s", args.port)
  main(port=args.port)
  ```

**⚠ Accuracy refinement (b).** The exact startup‑signal lines are **`email_handler.py:2386`** ("Start mail controller") and **`email_handler.py:2403`** ("Listen for port"). These are the two lines to grep for.

**Observed runtime signal (reproducible).** The handler's log shows both lines — and because the logger embeds the call site, the lines *self‑cite* `email_handler.py:2403` and `email_handler.py:2386`:

```text
2026-06-26 17:34:14,749 - SL - INFO  - 63 - "/app/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-06-26 17:34:14,751 - SL - DEBUG - 63 - "/app/email_handler.py:2386" - main()    -  - Start mail controller 0.0.0.0 20381
```

Connecting to the port confirms the listener is accepting connections — it answers with the SMTP greeting from the `aiosmtpd` library:

```text
$ (probe 127.0.0.1:20381)
220 <hostname> Python SMTP 1.4.2
```

**Rationale (why this proves it).** `controller.start()` ([`email_handler.py:2385`](#citations-appendix)) binds and begins serving *before* the "Start mail controller" line is logged, so seeing that line means the bind succeeded. The `220 … Python SMTP 1.4.2` banner is the protocol‑level proof that the socket is open and the aiosmtpd server (version 1.4.2) is responding — independent of the application logs. (A definitive end‑to‑end proof — actually accepting and forwarding a message — is shown in [Q2.3](#q23-have-the-alias-receive-an-email).)

### Q1.3 Job runner

**Answer.** The job runner is up when its process is running its 10‑second poll loop. It logs `Take job …` **only when it actually consumes a job**; a quiet loop with no such lines is normal and healthy when no jobs are pending.

**Code evidence.**

```python
# job_runner.py:329-347
if __name__ == "__main__":
    while True:                                              # :330
        with create_light_app().app_context():
            for job in get_jobs_to_run():                    # :333
                LOG.d("Take job %s", job)                    # :334
                ...
                process_job(job)                             # :342  (dispatch defined at :188)
                job.state = JobState.done.value              # :344
                Session.commit()
            time.sleep(10)                                   # :347
```

A crucial nuance lives in `get_jobs_to_run()` ([`job_runner.py:307‑326`](#citations-appendix)): it selects jobs that are `ready` **or** `taken`‑but‑stale (state `taken` with `taken_at` older than `JOB_TAKEN_RETRY_WAIT_MINS` and `attempts < JOB_MAX_ATTEMPTS`), **and** whose `run_at` is **null or within the next 10 minutes** — the filter is `or_(Job.run_at.is_(None), Job.run_at <= arrow.now().shift(minutes=+10))` ([`job_runner.py:312,323`](#citations-appendix)). That 10‑minute look‑ahead is why a near‑future job can be consumed slightly before its scheduled time; an idle job runner with no eligible jobs legitimately logs nothing between polls.

**Observed runtime signal (reproducible).** To prove the loop is alive, enqueue a job and watch it get consumed within ~10 s. A deliberately unknown job name is used so there are **no side effects** — `process_job` simply logs `Unknown job name …` via its `else` branch ([`job_runner.py:303‑304`](#citations-appendix)):

```text
# after inserting a Job(name="blitzy-verification-noop", state=ready):
2026-06-26 18:50:19,504 - SL - DEBUG - 2993 - "/app/job_runner.py:334" - <module>()    -  - Take job <Job 1 blitzy-verification-noop {'note': 'blitzy runtime check'}>
2026-06-26 18:50:19,507 - SL - ERROR - 2993 - "/app/job_runner.py:304" - process_job() -  - Unknown job name blitzy-verification-noop
```

The job's row then transitioned to `state = 2` (`done`, per the `JobState` enum at [`app/models.py:253‑256`](#citations-appendix)).

**Rationale (why this proves it).** The pair of lines proves the *entire* loop body executed: the runner **selected** a due job (`get_jobs_to_run`), **logged taking it** (`job_runner.py:334`), **invoked `process_job`** (`job_runner.py:304` is inside it), and **marked it done** (`state=2`). Seeing this happen within ~10 s of enqueueing — and a quiet loop otherwise — is exactly what a healthy poll‑based worker looks like.


---

## Q2 — Basic User Actions

**Question restated:** After the app is running, how do I create an account, create an alias, and have that alias receive an email — and what runtime signals prove each step worked?

The end‑to‑end flow and its signals are summarized here, then detailed below:

| Step | Route / mechanism | Code | Proof signal observed |
|---|---|---|---|
| Register | `POST /auth/register` | `app/auth/views/register.py:31,86,95` | 200 → "waiting activation" page; `User(activated=False)` + `ActivationCode` rows |
| Activate | `GET /auth/activate?code=…` | `app/auth/views/activate.py:13,49,53,56` | 302 redirect; `users.activated=true`; single‑use code deleted |
| Create alias | dashboard `POST` (`form-name=create-random-email`) | `app/dashboard/views/index.py:97,104,110,111` | flash "Alias … has been created"; new `Alias` row; self‑cited log line |
| Receive email | inbound SMTP to `:20381` | `email_handler.py:1945,536,679,732` | "New message"/"Finish mail_from" log pair; new `EmailLog` row; `+1` forward |

### Q2.1 Create an account (register → activate)

**Answer.** A new account is created through `POST /auth/register`, which creates a `User` with `activated=False` and sends an activation email; clicking the activation link (`GET /auth/activate?code=…`) flips `activated` to `True`, logs the user in, and consumes the single‑use code.

**Code evidence — registration** ([`app/auth/views/register.py`](#citations-appendix)):

```python
@auth_bp.route("/register", methods=["GET", "POST"])     # :31
def register():
    ...
    LOG.d("create user %s", email)                       # :85
    user = User.create(                                  # :86
        email=email, name=form.email.data,
        password=form.password.data, referral=get_referral(),
    )
    Session.commit()
    try:
        send_activation_email(user, next_url)            # :95
        ...
    return render_template("auth/register_waiting_activation.html")   # :104
```

**Code evidence — activation** ([`app/auth/views/activate.py`](#citations-appendix)):

```python
@auth_bp.route("/activate", methods=["GET", "POST"])     # :13
def activate():
    ...
    activation_code: ActivationCode = ActivationCode.get_by(code=code)   # :26
    ...
    user = activation_code.user
    user.activated = True                                # :49
    login_user(user)                                     # :50
    ActivationCode.delete(activation_code.id)            # :53  (single-use)
    Session.commit()
    flash("Your account has been activated", "success")  # :56
```

**Verification shortcut.** Because the seeded `john@wick.com` account is created **already activated** and as an **admin** ([`app/fake_data.py:44‑49`](#citations-appendix)), you can demonstrate registration/activation as a *separate* exercise from alias/email testing.

**Observed runtime signal (reproducible).** Driving the real HTTP routes with a fresh session (anonymous, so registration is allowed):

```text
GET  /auth/register   -> 200   (CSRF token present in the form)
POST /auth/register   -> 200   ("waiting activation" page rendered)
```

The database then showed the new, **unactivated** user and a matching single‑use activation code (the test account used a disposable `blitzy_test_…@example.com` address; counts shown are illustrative of the delta):

```text
 users.email                          | activated
 -------------------------------------+-----------
 blitzy_test_…@example.com            | f             <-- created un-activated

 activation_code: 1 row for that user_id   (single-use code)
```

Because of local mode (see [Q2.4](#q24-the-critical-local-mode-nuance-not_send_email)), the activation email is **logged, not sent** — the log line self‑cites `app/mail_sender.py:131`:

```text
2026-06-26 18:46:47,175 - SL - DEBUG - 77 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'Just one more step to join SimpleLogin', from '"noreply@sl.local" <noreply@sl.local>' to 'blitzy_test_…@example.com'
```

After issuing `GET /auth/activate?code=<code>`:

```text
GET /auth/activate?code=<single-use code>  -> 302 FOUND

 users.email                          | activated
 -------------------------------------+-----------
 blitzy_test_…@example.com            | t             <-- now activated
 activation_code rows for that user   : 0             <-- single-use code deleted
```

**Rationale (why this proves it).** The transition `activated: f → t` together with the activation‑code row disappearing maps **exactly** to the code: `user.activated = True` ([`app/auth/views/activate.py:49`](#citations-appendix)) and `ActivationCode.delete(...)` ([`app/auth/views/activate.py:53`](#citations-appendix)). The `302` is the post‑login redirect that follows `login_user(user)` ([`app/auth/views/activate.py:50`](#citations-appendix)). The `200 → "waiting activation"` page maps to `render_template("auth/register_waiting_activation.html")` ([`app/auth/views/register.py:104`](#citations-appendix)), and the un‑activated `User` plus the `ActivationCode` row map to `User.create(...)` ([`app/auth/views/register.py:86`](#citations-appendix)) followed by `send_activation_email(...)` ([`app/auth/views/register.py:95`](#citations-appendix)). Every observed signal has a one‑to‑one source counterpart.

### Q2.2 Create an alias

**Answer.** From the dashboard, the **"Random Alias"** button issues a `POST` with `form-name=create-random-email`, which calls `Alias.create_new_random(...)`, commits a new `Alias` row, logs the creation, and flashes a confirmation.

**Code evidence** ([`app/dashboard/views/index.py`](#citations-appendix)):

```python
@dashboard_bp.route("/", methods=["GET", "POST"])        # :55
@login_required                                          # :56
...
elif request.form.get("form-name") == "create-random-email":   # :97
    if current_user.can_create_new_alias():
        ...
        alias = Alias.create_new_random(user=current_user, scheme=scheme)  # :104
        alias.mailbox_id = current_user.default_mailbox_id
        Session.commit()
        LOG.d("create new random alias %s for user %s", alias, current_user)  # :110
        flash(f"Alias {alias.email} has been created", "success")             # :111
```

The dashboard template renders the alias list and controls, and surfaces per‑account activity statistics — **"New Custom Alias"** ([`templates/dashboard/index.html:46`](#citations-appendix)) and **"Random Alias"** ([`templates/dashboard/index.html:56`](#citations-appendix)) controls, plus the stats `nb_alias` ([line 131](#citations-appendix)), `nb_forward` ([line 145](#citations-appendix)), `nb_reply` ([line 159](#citations-appendix)), and `nb_block` ([line 173](#citations-appendix)). A custom‑alias path also exists (`app/dashboard/views/custom_alias.py`), with shared logic in `app/alias_utils.py`.

**Observed runtime signal (reproducible).** Clicking **"Random Alias"** in the dashboard (logged in as `john@wick.com`) navigated to `/dashboard/?highlight_alias_id=13&…` and rendered a new, highlighted card **`word_list342@sl.local`** labeled *"Created just now."* The alias is on the `@sl.local` domain (matching `EMAIL_DOMAIN`, see [Q2.4](#q24-the-critical-local-mode-nuance-not_send_email)). The server log self‑cites `app/dashboard/views/index.py:110`:

```text
2026-06-26 18:47:44,834 - SL - DEBUG - 78 - "/app/app/dashboard/views/index.py:110" - index() -  - create new random alias <Alias 13 word_list342@sl.local> for user <User 1 John Wick john@wick.com>
```

…and the database held the corresponding row: `alias.id=13`, `email=word_list342@sl.local`, `user_id=1` (John), `enabled=true`.

**Rationale (why this proves it).** The new `Alias` row plus the self‑citing `app/dashboard/views/index.py:110` log line are the direct outputs of `Alias.create_new_random(...)` ([`app/dashboard/views/index.py:104`](#citations-appendix)) and its `LOG.d(...)` ([`app/dashboard/views/index.py:110`](#citations-appendix)); the `highlight_alias_id=13` query parameter on the redirect and the "Created just now" card are the user‑visible confirmation that the flash ([`app/dashboard/views/index.py:111`](#citations-appendix)) describes. The route is `@login_required` ([`app/dashboard/views/index.py:56`](#citations-appendix)), so reaching it at all confirms the authenticated session from Q1.

### Q2.3 Have the alias receive an email

**Answer.** Sending an inbound message to `<alias>@sl.local` on port `20381` is accepted by the email handler, routed by `handle()` to the forward phase `handle_forward()` → `forward_email_to_mailbox()`, which creates a `Contact` and an `EmailLog` row and (in local mode) **logs** the forwarded message. The handler brackets the whole operation with a "New message …" line on entry and a "Finish mail_from …" line on exit.

**Code evidence.**

- Routing hub: `def handle(envelope, msg) -> str` ([`email_handler.py:1945`](#citations-appendix)) classifies the message and dispatches.
- Forward phase: `def handle_forward(...)` ([`email_handler.py:536`](#citations-appendix)) → `def forward_email_to_mailbox(...)` ([`email_handler.py:679`](#citations-appendix)).
- Contact resolution: `get_or_create_contact(...)` ([`email_handler.py:581`](#citations-appendix)).
- `EmailLog` creation: `EmailLog.create(...)` spans [`email_handler.py:732‑739`](#citations-appendix), immediately followed by `LOG.d("Create %s for %s, %s, %s", email_log, …)` at [`email_handler.py:740`](#citations-appendix). (A separate blocked‑path `EmailLog.create(...)` with the alias disabled lives at [`email_handler.py:598`](#citations-appendix).)
- Per‑message brackets: the entry `LOG.i("New message, mail from %s, rctp tos %s ", …)` — **the source contains the typo `rctp` (for "rcpt"), preserved verbatim here** — is the statement spanning [`email_handler.py:2343‑2347`](#citations-appendix) (the format string is on line 2344; the `LOG.i(` call begins on line 2343). The exit `LOG.i("Finish mail_from %s, rcpt_tos %s, takes %s seconds with return code '%s'<<===", …)` is the statement spanning [`email_handler.py:2367‑2373`](#citations-appendix) (format string on line 2368; `LOG.i(` call begins on line 2367).

> **Why the cited line and the runtime line can differ by one.** Python's logging records the line where the `LOG.x(` *call* begins. For these two multi‑line calls that is `2343` and `2367`, whereas the human‑readable message *string literal* is on the next line (`2344` and `2368`). Both are reported here so the citation is exact either way. The same applies to the `EmailLog` `LOG.d` whose call is on `740` while the `EmailLog.create(...)` it describes spans `732‑739`.

**Observed runtime signal (reproducible).** A test message was sent with Python's `smtplib` to `word_list342@sl.local` via `127.0.0.1:20381` (the project's documented tool is `swaks --to <alias>@sl.local --from hey@google.com --server 127.0.0.1:20381`, [`CONTRIBUTING.md:218`](#citations-appendix); `swaks` is not present in this image, so `smtplib` was used to the same effect, and the SMTP server returned `250`). The handler produced a complete, single‑message trace in which **every line shares the same message‑id** (`f5101992‑…`), demonstrating the `set_message_id` correlation from [§0.5](#05-the-centralized-logger). The lines self‑cite their source:

```text
"/app/email_handler.py:2343" _handle()                New message, mail from hey@google.com, rctp tos ['word_list342@sl.local']
"/app/email_handler.py:2202" handle()                 Forward phase hey@google.com(hey@google.com) -> word_list342@sl.local
"/app/email_handler.py:580"  handle_forward()         Create or get contact for from_header:hey@google.com
"/app/app/handler/dmarc.py:33" ...forward_phase()     DMARC check disabled
"/app/email_handler.py:688"  forward_email_to_mailbox Forward <Contact 2 hey@google.com 13> -> <Alias 13 word_list342@sl.local> -> <Mailbox 1 john@wick.com>
"/app/email_handler.py:740"  forward_email_to_mailbox Create <EmailLog 2> for <Contact 2 hey@google.com 13>, <User 1 John Wick john@wick.com>, <Mailbox 1 john@wick.com>
"/app/email_handler.py:867"  forward_email_to_mailbox From header, new:"hey at google.com" <hey_at_google_com_…@sl.local>, old:hey@google.com
"/app/app/mail_sender.py:131" send()                  send email with subject 'Blitzy verification test message', from '"hey at google.com" <…@sl.local>' to 'word_list342@sl.local'
"/app/email_handler.py:2367" _handle()                Finish mail_from hey@google.com, rcpt_tos ['word_list342@sl.local'], takes 0.0516 seconds with return code '250 Message accepted for delivery'<<===
```

Database deltas confirmed a **new `Contact`** (`hey@google.com`, for alias 13) and a **new `EmailLog`** row (the `email_log` and `contact` counts each incremented by one). The `EmailLog`/`Contact`/`Alias` models live at [`app/models.py:2060 / 1863 / 1469`](#citations-appendix).

**Rationale (why this proves it).** The "New message" entry line ([`email_handler.py:2343‑2347`](#citations-appendix)) proves the SMTP server *accepted* the envelope; the "Finish mail_from … return code '250 …'" exit line ([`email_handler.py:2367‑2373`](#citations-appendix)) proves it *finished successfully*. Between them, the self‑citing trace shows the exact code path the question is about — `handle()` → forward phase → `forward_email_to_mailbox()` resolving `Contact → Alias → Mailbox` and writing `EmailLog 2` ([`email_handler.py:740`](#citations-appendix)). The persistent `EmailLog` row is the durable proof that the alias *received and accounted for* the message; it is also what increments the dashboard's `nb_forward` statistic. The shared message‑id ties every line to this one message, so the evidence is unambiguous.

### Q2.4 The critical local‑mode nuance: `NOT_SEND_EMAIL`

**This is the single most important thing to understand so you do not misread "no delivery" as a failure.**

**Answer.** In local configuration, SimpleLogin **skips outbound SMTP entirely and logs the send metadata instead of transmitting the message**. Concretely, the `MailSender.send()` `NOT_SEND_EMAIL` branch logs the message's `subject`, `from`, and `to` headers — **not** the full MIME body — and returns success ([`app/mail_sender.py:130‑137`](#citations-appendix)). So after a successful inbound test you will see that send‑metadata line *logged*, and the alias's `EmailLog` row created — but no mail leaves the machine. That is correct, expected behavior.

**Code evidence.**

- `example.env` sets `NOT_SEND_EMAIL=true` ([`example.env:19`](#citations-appendix), with the comment *"Only print email content, not sending it, for local development"* on line 18) and `EMAIL_DOMAIN=sl.local` ([`example.env:22`](#citations-appendix)).
- The flag is read in `app/config.py` as **membership**, not value: `NOT_SEND_EMAIL = "NOT_SEND_EMAIL" in os.environ` ([`app/config.py:91`](#citations-appendix)). **Subtlety:** the flag is enabled whenever the variable is *present at all* — even `NOT_SEND_EMAIL=false` would still enable log‑only mode, because only the *presence* of the key is checked.
- The decisive branch is in `MailSender.send()` ([`app/mail_sender.py:130‑137`](#citations-appendix)): when `config.NOT_SEND_EMAIL` is truthy it logs `LOG.d("send email with subject '%s', from '%s' to '%s'", …)` and `return True` **without** calling `_send_to_smtp`.

**Observed runtime signal.** Both the activation email ([Q2.1](#q21-create-an-account-register--activate)) and the forwarded test message ([Q2.3](#q23-have-the-alias-receive-an-email)) appeared as `send()` log lines self‑citing `app/mail_sender.py:131` — and no external delivery occurred. `EMAIL_DOMAIN=sl.local` is also why the test alias had to be `…@sl.local` for the inbound message to be routed to it.

**Rationale (why this proves it).** The `return True` after logging ([`app/mail_sender.py:137`](#citations-appendix)) means the application treats the send as successful while skipping the actual SMTP transmission. Therefore the *correct* success signal in local mode is the `send email with subject …` log line plus the persisted `EmailLog` row — **not** an outbound message. A newcomer who looks for delivered mail and finds none has not found a bug; they have found `NOT_SEND_EMAIL=true` working as designed.


---

## Q3 — Background Components

**Question restated:** Do the email handler and job runner come online "in the background" to support email activity and data handling while the app is active, and what behavior demonstrates they function as intended?

**Answer (stated precisely).** The email handler and job runner **do *not* auto‑spawn from the web server.** They are **separate, independent processes** — `python email_handler.py` and `python job_runner.py` — started either by the container/process supervisor or by explicit commands, and each then enters its own continuous loop. In that sense they run "in the background" relative to the web app, but the web server does **not** launch them; if you forget to start them, the web UI still works while email forwarding and job processing simply do not happen.

> This precision matters. It would be wrong to claim the web app "automatically brings up" the handler and job runner — the code does not do that. They come online because *they are started as their own processes*, not because the web server started them.

**Code evidence — each is its own program with its own loop.**

- **Email handler:** has its own `main()` and `__main__` block, and an infinite loop: `Controller(...).start()` then `while True: time.sleep(2)` ([`email_handler.py:2381‑2393`](#citations-appendix)), driven from the `__main__` entry at [`email_handler.py:2403‑2404`](#citations-appendix).
- **Job runner:** has its own `__main__` block with `while True: … time.sleep(10)` ([`job_runner.py:329‑347`](#citations-appendix)).
- **Web server:** `create_app()` only calls `register_blueprints(app)` ([`server.py:233‑246`](#citations-appendix)) and the local entry point `local_main()` only configures and calls `app.run(debug=True, port=7777)` ([`server.py:572‑587`](#citations-appendix)). **There is no `subprocess`, `Popen`, `os.fork`, thread, or import‑side launch of `email_handler.py` or `job_runner.py` anywhere in the web startup path.** The web app and the two workers share only the **PostgreSQL database** (and Redis) — that is the coupling, not process parenting.

**Observed runtime signal — process independence (the decisive proof).** Inspecting the process table with parent‑PID shows the relationships unambiguously:

```text
   PID   PPID  COMMAND
    55      0  gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15     <- web master
    77     55  gunicorn ... (worker)                                   <- child of 55
    78     55  gunicorn ... (worker)                                   <- child of 55
    63      0  python email_handler.py                                 <- NOT a child of 55
  2993   2985  python job_runner.py                                    <- NOT a child of 55
```

The web master (PID 55) has exactly two children — its own Gunicorn workers (77, 78). The email handler (PID 63) and job runner (PID 2993) have **different parents** and are therefore **not** spawned by the web server. This is the direct runtime confirmation of the code reading above.

**Observed runtime signal — they function while the app is active.**

- *Email handler functioning:* the complete inbound‑mail trace in [Q2.3](#q23-have-the-alias-receive-an-email) (entry "New message", `EmailLog 2` written, exit "Finish mail_from … 250") shows the handler accepting and forwarding live traffic on `:20381` while the web app served the dashboard.
- *Job runner functioning:* the [Q1.3](#q13-job-runner) demonstration — enqueue a `Job`, observe `Take job …` ([`job_runner.py:334`](#citations-appendix)) and the `process_job` dispatch ([`job_runner.py:304`](#citations-appendix)) within ~10 s, and the row reaching `state=done` — proves the runner is consuming enqueued `Job` rows on its poll cycle.

**Supporting background daemons (corroborating the documented six‑process model).** Three more processes exist, and — importantly — they are **also launched explicitly or on a schedule, not spawned by the web app**:

- `cron.py` — scheduled maintenance run by `yacron` per `crontab.yml` / `crontab-all-hosts.yml` (entries like `python /code/cron.py -j <job>`).
- `event_listener.py` — a PostgreSQL `LISTEN`/`NOTIFY` consumer with a `Mode` enum (`DEAD_LETTER`, `LISTENER`) ([`event_listener.py:15‑26`](#citations-appendix)).
- `monitoring.py` — a metrics daemon on a 60‑second loop ([`monitoring.py:171`](#citations-appendix)).

Together with the web app, SMTP handler, and job runner, these make up the documented six‑process model, which already includes the monitoring daemon.

**Models backing all of these observations** ([`app/models.py`](#citations-appendix)): `User` (line 336), `ActivationCode` (line 1202), `Alias` (line 1469), `Contact` (line 1863), `EmailLog` (line 2060), `Job` (line 2683).

**Rationale (why this proves it).** The question "do they come online in the background?" has a precise, code‑grounded answer: yes, they run as background processes — but they are **independent**, evidenced by (1) each having its own `__main__` and infinite loop, (2) the web startup path containing no spawn of them, and (3) the process table showing they are not children of the web master. "Functioning as intended" is then proved by *behavioral* evidence under two different situations: an inbound email produces a forward + `EmailLog` (handler), and an enqueued job is taken and completed within one poll interval (runner). The shared coupling is the database, which is exactly why all three can cooperate without any one launching another.


---

## Consolidated Rationale / Thinking Thread

The verification strategy throughout this document follows one discipline: **isolate the smallest signal that can only be produced if the component is actually working, and tie it to the exact code that produces it.**

- **Q1 — Health.** A liveness check should have *no side effects* so a positive result cannot be a false positive caused by some other working subsystem. `/health` is perfect for this: it returns a constant `("success", 200)` with no DB access ([`server.py:213‑215`](#citations-appendix)), so a `200` isolates "the WSGI app is routing and a worker is alive." The email handler and job runner have no HTTP surface, so their honest liveness signals are their **startup log lines** (`email_handler.py:2386/2403`) and their **loop behavior** (`job_runner.py:334`). The one subtlety a newcomer must know is that `/health` is intentionally *silent* in the request log ([`server.py:281`, `195`](#citations-appendix)) — silence there is health, not failure.
- **Q2 — Actions.** Each user action was driven through its **real route or real SMTP port**, and verified against a **durable artifact** (a database row) rather than a transient UI message alone — because rows are unambiguous and reproducible. Register → an un‑activated `User` + `ActivationCode`; activate → `activated=true` + code deleted; create alias → a new `Alias` row + self‑citing log; receive email → a new `EmailLog` + `Contact` and the "New message"/"Finish mail_from" bracket. The `NOT_SEND_EMAIL` nuance is called out explicitly because it is the most common way a correct local deployment is *mistaken* for a broken one.
- **Q3 — Background components.** The answer is deliberately phrased to avoid over‑claiming "automatic" startup. The proof is layered: a **code** reading (each has its own loop; the web app spawns nothing), a **structural** runtime observation (process table parentage), and **behavioral** runtime observations (a forwarded email; a consumed job). Three independent angles converging on the same conclusion is what makes the answer trustworthy.

In every case the runtime evidence and the code citation reinforce each other — and because the logger prints `pathname:lineno`, the running system effectively *cites its own source* as it works.

---

## Test Data Cleanup & Non‑Persistence

All verification data created while capturing the observations above were **ephemeral runtime database rows**, not repository files, and **every one was deleted** on completion using the application's own deletion logic:

- The test alias created for the seeded user was removed with `delete_alias(alias, user)` ([`app/alias_utils.py:336`](#citations-appendix)) — the project's required path, since `Alias.delete` deliberately raises ([`app/models.py:1717`](#citations-appendix)). This cascaded its `Contact` and `EmailLog` rows.
- The disposable test account was removed with `User.delete(id)` ([`app/models.py:671`](#citations-appendix)), which internally deletes the user's aliases and then the user.
- The no‑op verification `Job` row was deleted.
- The alias‑"trash" rows that `delete_alias` writes (`DeletedAlias`) for the two test aliases were also removed, leaving no residue.

**Verified end state:** the database counts returned **exactly** to their pre‑test baseline (`users`, `alias`, `email_log`, `contact`, `job`), no `blitzy_test_…` user, no test alias, and no test job remained, and the seeded `john@wick.com` account was left intact (still `activated`, still `admin`). Deletion events were no‑ops in local mode ("Not sending events because webhook is not configured"), so no event residue was created.

**Source‑repository integrity:** No existing repository file was modified, added, or deleted in service of this task. The **only** artifact produced is this single document, `blitzy/documentation/app_2cd6ee777f8c.md`. All "test data" lived solely in the running PostgreSQL instance and has been removed.

---

## Secrets Handling Note

No real secrets are reproduced in this document. Sensitive material is referenced **by role only**:

- **`FLASK_SECRET`** — Flask session signing key; role only, value never shown.
- **`local_data/` key material** — the local DKIM signing key, JWT RS256 key pair, and PGP private key used by the handler for signing/encryption; described by role, never reproduced.
- **Database credentials** — shown only as the public placeholders that already appear in the project's own published docs (`example.env` / `CONTRIBUTING.md`); runtime‑observed connection strings have the password redacted (`…:<password>@…`).
- **DKIM keys** — referenced as the mechanism by which forwarded mail is signed; key bytes never shown.

---

## Citations Appendix

Every claim above maps to one of the following `path:line` references (verified exact against git HEAD `2cd6ee777f8c`) or to a reproducible runtime observation described in‑line. Line numbers point at the relevant statement; where a logging call spans multiple lines, both the call‑site line and the message‑string line are noted in the text.

| # | Claim | Evidence (`path:line`) |
|---|---|---|
| 1 | Web liveness `GET /health` → `("success", 200)` | `server.py:213-215` |
| 2 | `/health` excluded from per‑request `after_request` log | `server.py:281` |
| 3 | `/health` excluded from the profiler ignore list | `server.py:195` |
| 4 | Per‑request activity logged by `after_request` hook | `server.py:284-292` |
| 5 | Index `/` redirects (auth → dashboard, anon → login) | `server.py:250-255` |
| 6 | Blueprints registered (auth, monitor, dashboard, …, oauth at `/oauth` & `/oauth2`, …, api) | `server.py:233-246` |
| 7 | Local entry `local_main()`; forces `COLOR_LOG=True`; `app.run(port=7777)` | `server.py:572-587` (COLOR_LOG at `573`) |
| 8 | Gunicorn WSGI entry `app = create_app()` | `wsgi.py` |
| 9 | Monitor blueprint mounted at `url_prefix="/"` | `app/monitor/base.py:3` |
| 10 | `/git` → SHA1, `/live` → `"live"`, `/exception` → raises | `app/monitor/views.py:5-7, 10-12, 15-18` |
| 11 | Logger format (incl. `pathname:lineno`, `message_id`) | `app/log.py:12-14` |
| 12 | `set_message_id` per‑email correlation | `app/log.py:22` |
| 13 | Logger writes to stdout (`StreamHandler(sys.stdout)`) | `app/log.py:41` |
| 14 | `coloredlogs` used when `COLOR_LOG` set | `app/log.py:61-62` |
| 15 | Email handler `Controller(...)`, `start()`, "Start mail controller" | `email_handler.py:2383, 2385, 2386` |
| 16 | Email handler `__main__`: default port 20381, "Listen for port" | `email_handler.py:2399, 2403-2404` |
| 17 | Email handler infinite loop `while True: time.sleep(2)` | `email_handler.py:2392-2393` |
| 18 | Routing hub `handle(envelope, msg)` | `email_handler.py:1945` |
| 19 | Forward phase `handle_forward(...)` | `email_handler.py:536` |
| 20 | `forward_email_to_mailbox(...)` | `email_handler.py:679` |
| 21 | `get_or_create_contact(...)` (forward) | `email_handler.py:581` (log at `580`) |
| 22 | `EmailLog.create(...)` (forward path) + creation log | `email_handler.py:732-739` (LOG.d at `740`) |
| 23 | Blocked‑path `EmailLog.create(blocked=True)` | `email_handler.py:598` |
| 24 | Entry "New message … rctp tos …" (typo verbatim) | `email_handler.py:2343-2347` (string at `2344`) |
| 25 | Exit "Finish mail_from … return code …" | `email_handler.py:2367-2373` (string at `2368`) |
| 26 | Job runner poll loop & `Take job` & `time.sleep(10)` | `job_runner.py:329-347` (`Take job` at `334`, sleep at `347`) |
| 27 | `process_job` dispatch + "Unknown job name" else‑branch | `job_runner.py:188, 303-304` |
| 28 | `get_jobs_to_run()` selects `ready` or stale‑`taken` jobs whose `run_at` is null or within the next 10 min (`run_at <= arrow.now().shift(minutes=+10)`) | `job_runner.py:307-326` (look‑ahead at `312`, comparison at `323`) |
| 29 | Register route, `User.create`, `send_activation_email`, waiting page | `app/auth/views/register.py:31, 86, 95, 104` |
| 30 | Activate route, `activated=True`, `login_user`, code delete, success flash | `app/auth/views/activate.py:13, 49, 50, 53, 56` |
| 31 | Seeded `john@wick.com` created activated + admin | `app/fake_data.py:44-49` |
| 32 | Dashboard route `@login_required`; random‑alias branch; `create_new_random`; log; flash | `app/dashboard/views/index.py:55-56, 97, 104, 110, 111` |
| 33 | Dashboard controls + stats (`nb_alias`/`nb_forward`/`nb_reply`/`nb_block`) | `templates/dashboard/index.html:46, 56, 131, 145, 159, 173` |
| 34 | `NOT_SEND_EMAIL=true` + comment; `EMAIL_DOMAIN=sl.local` | `example.env:18-19, 22` |
| 35 | `NOT_SEND_EMAIL` read as env‑membership | `app/config.py:91` |
| 36 | `MailSender.send()` logs & returns True when `NOT_SEND_EMAIL` | `app/mail_sender.py:130-137` |
| 37 | `COLOR_LOG` commented in `example.env` | `example.env:16` |
| 38 | `URL=http://localhost:7777`; default `DB_URI` (port 5432) | `example.env:6, 75` |
| 39 | Models: User/ActivationCode/Alias/Contact/EmailLog/Job; `JobState` | `app/models.py:336, 1202, 1469, 1863, 2060, 2683; 253-256` |
| 40 | `Alias.delete` raises; use `delete_alias`; `User.delete` cascade | `app/models.py:1717, 671; app/alias_utils.py:336` |
| 41 | `event_listener.py` `Mode` enum (LISTENER/DEAD_LETTER) | `event_listener.py:15-26` |
| 42 | `monitoring.py` 60‑second loop | `monitoring.py:171` |
| 43 | Docker: node build stage, python:3.10, EXPOSE 7777, gunicorn CMD | `Dockerfile:2, 8, 44, 47` |
| 44 | Local‑run procedure; DB_URI port disagreement; login; email_handler/swaks/job_runner | `CONTRIBUTING.md:94, 100, 106, 109, 212, 218, 228` |
| 45 | Operational runbook (`docker logs`, `swaks`) | `docs/troubleshooting.md` |

### Reproduce‑it‑yourself quick reference

```bash
# Q1 web
curl -i http://localhost:7777/health     # -> 200, body "success"
curl -i http://localhost:7777/           # -> 302, Location: /auth/login
curl    http://localhost:7777/live       # -> live

# Q1 email handler — look for the two startup lines (they self-cite their source):
#   "/app/email_handler.py:2403" ... Listen for port 20381
#   "/app/email_handler.py:2386" ... Start mail controller 0.0.0.0 20381

# Q2 inbound email (documented tool; substitute smtplib if swaks is absent):
swaks --to <alias>@sl.local --from hey@google.com --server 127.0.0.1:20381
#   -> "New message ... rctp tos ['<alias>@sl.local']"  and
#   -> "Finish mail_from ... return code '250 Message accepted for delivery'"
#   -> a new EmailLog row; with NOT_SEND_EMAIL the send metadata (subject/from/to) is LOGGED and outbound SMTP is skipped.

# Q3 process independence:
ps -eo pid,ppid,args | grep -E "gunicorn wsgi:app|email_handler.py|job_runner.py"
#   -> email_handler.py and job_runner.py are NOT children of the gunicorn master PID.
```

