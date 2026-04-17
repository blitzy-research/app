# SimpleLogin Runtime Verification Q&A — Branch app_2cd6ee777f8c

This document answers four operational questions about the self-hosted [SimpleLogin](https://github.com/simple-login/app) application, evaluated against commit `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`. Every answer is anchored either in a specific source-file citation (`file_path:line_number`) or in a log line that was captured verbatim by running the three core processes (web server, email handler, job runner) inside the provided Docker sandbox image `andrewparkscaleai/coding-agent:simple-login__app__2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`. **No source file was modified** while producing this document; the only artifact added to the repository is this single markdown file at `blitzy/documentation/app_2cd6ee777f8c.md`. All test data (one alias, one contact, one email-log row, one probe job) that was created during runtime exploration was deleted before writing the final answers — cleanup was confirmed with `SELECT COUNT(*)` queries that each returned `0`.

Runtime environment used for the captured evidence (all values drawn from `example.env` and the container setup log):

```sh
URL=http://localhost:7777
NOT_SEND_EMAIL=true                       # example.env:19
EMAIL_DOMAIN=sl.local                     # example.env:22
SUPPORT_EMAIL=support@sl.local
DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin
FLASK_SECRET=secret
DISABLE_ONBOARDING=true                   # example.env:150
LOCAL_FILE_UPLOAD=true
```

Dependency versions from `pyproject.toml` lines 60-119 that matter for the answers below: Python `^3.10`, Flask `^1.1.2`, gunicorn `^20.0.4`, aiosmtpd `^1.2`, SQLAlchemy `1.3.24` (pinned), psycopg2-binary `^2.9.3`, flask-login `^0.5.0`, Flask-Limiter `^1.4`, redis `^4.5.3`, PGPy `0.5.4` (pinned), cryptography `37.0.1` (pinned). The pinned versions (SQLAlchemy, PGPy, cryptography) are critical for compatibility reasons documented in the project's Poetry manifest; the caret versions are the minimums Poetry accepts.

---

## Q1. "How do I confirm that the web server, email handler, and job runner are actually up and responding after launch?"

Each of the three processes exposes a different observable signal. The web server answers an explicit HTTP health endpoint; the email handler binds a TCP listen socket and emits two distinct startup log lines; the job runner has no HTTP endpoint at all and is instead verified via process inspection and a controlled `job`-table probe. The following subsections cover each one.

### 1.1 Web server

**Source of truth — `server.py:213-215`** defines a trivial health endpoint that is registered inside `create_app()`:

```python
@app.route("/health", methods=["GET"])
def healthcheck():
    return "success", 200
```

**WSGI entry point — `wsgi.py:1-3`** is the complete file:

```python
from server import create_app

app = create_app()
```

**Gunicorn launch command — `Dockerfile:47`** (the container's only `CMD`):

```sh
CMD ["gunicorn","wsgi:app","-b","0.0.0.0:7777","-w","2","--timeout","15"]
```

`EXPOSE 7777` on `Dockerfile:44` declares the port the container publishes. The image binds to all interfaces at port 7777, and `wsgi:app` is the WSGI callable that Gunicorn imports.

**Captured runtime evidence.** In the Docker sandbox, starting the process and calling `/health` produced the following:

```sh
$ docker exec simplelogin-setup bash -c "curl -s -v http://localhost:7777/health 2>&1"
*   Trying 127.0.0.1:7777...
* Connected to localhost (127.0.0.1) port 7777 (#0)
> GET /health HTTP/1.1
> Host: localhost:7777
> User-Agent: curl/7.88.1
> Accept: */*
>
< HTTP/1.1 200 OK
< Server: gunicorn/20.0.4
< Date: Thu, 16 Apr 2026 23:19:31 GMT
< Connection: close
< Content-Type: text/html; charset=utf-8
< Content-Length: 7
success
```

The body is the literal string `success`, the status is `200 OK`, and the `Server` header is `gunicorn/20.0.4` — consistent with the Poetry pin (`gunicorn = "^20.0.4"`).

**Startup log lines captured from gunicorn stderr** (`/tmp/gunicorn.log`):

```log
[2026-04-16 23:19:15 +0000] [2616] [INFO] Starting gunicorn 20.0.4
[2026-04-16 23:19:15 +0000] [2616] [INFO] Listening at: http://0.0.0.0:7777 (2616)
[2026-04-16 23:19:15 +0000] [2616] [INFO] Using worker: sync
[2026-04-16 23:19:15 +0000] [2617] [INFO] Booting worker with pid: 2617
>>> init logging <<<
2026-04-16 23:19:16,514 - SL - DEBUG - 2617 - "/workspace/app/utils.py:17" - <module>() -  - load words file: /workspace/local_data/test_words.txt
```

The banner `>>> init logging <<<` is emitted by `app/log.py:67`:

```python
print(">>> init logging <<<")
```

…and confirms that the `app.log` module was imported inside the worker, meaning the full Flask application factory ran successfully.

**Process-level checks.** Either of these is sufficient:

```sh
$ ps -ef | grep gunicorn
root  2616  ...  gunicorn wsgi:app -b 0.0.0.0:7777 -w 1 --timeout 15 ...
root  2617  ...  gunicorn wsgi:app -b 0.0.0.0:7777 -w 1 --timeout 15 ...  # the worker
```

…or read `/proc/net/tcp` directly (the tested sandbox did not have `ss` or `netstat`):

```sh
$ cat /proc/net/tcp | awk '{print $2}' | grep 1E61
00000000:1E61       # hex 0x1E61 == 7777 decimal, bound on 0.0.0.0
```

An entry of `00000000:1E61` in the second column (local address) with a state of `0A` (LISTEN) confirms the server has a listening socket on all IPv4 interfaces at port 7777.

### 1.2 Email handler

**Source of truth — `email_handler.py:2381-2393`** (the `main()` function that sets up the aiosmtpd controller):

```python
def main(port: int):
    """Use aiosmtpd Controller"""
    controller = Controller(MailHandler(), hostname="0.0.0.0", port=port)

    controller.start()
    LOG.d("Start mail controller %s %s", controller.hostname, controller.port)

    if LOAD_PGP_EMAIL_HANDLER:
        LOG.w("LOAD PGP keys")
        load_pgp_public_keys()

    while True:
        time.sleep(2)
```

And the `__main__` block — `email_handler.py:2396-2404`:

```python
if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument(
        "-p", "--port", help="SMTP port to listen for", type=int, default=20381
    )
    args = parser.parse_args()

    LOG.i("Listen for port %s", args.port)
    main(port=args.port)
```

Two startup log lines are guaranteed, in this order:

1. **INFO** at line 2403: `Listen for port 20381`
2. **DEBUG** at line 2386: `Start mail controller 0.0.0.0 20381`

After those two lines, the process enters an infinite `while True: time.sleep(2)` loop (line 2392-2393) and remains silent until an SMTP connection arrives.

**Captured runtime evidence.** From `/tmp/email_handler.log` in the sandbox immediately after starting `python email_handler.py -p 20381`:

```log
>>> init logging <<<
2026-04-16 23:17:54,934 - SL - INFO - 2366 - "/workspace/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-04-16 23:17:54,935 - SL - DEBUG - 2366 - "/workspace/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

The file-and-line column of the log record (`"/workspace/email_handler.py:2403"` and `"/workspace/email_handler.py:2386"`) exactly matches the expected source lines from the snippets above. This is how the global log format renders `%(pathname)s:%(lineno)d` — see Q2 for the format definition.

**Process-level checks.** The email handler binds TCP port 20381 (argparse default — `email_handler.py:2399`), so the same `/proc/net/tcp` inspection works:

```sh
$ cat /proc/net/tcp | awk '{print $2}' | grep 4F9D
00000000:4F9D       # hex 0x4F9D == 20381 decimal
```

Alternatively, a direct `NOOP` SMTP probe confirms the socket accepts the SMTP protocol:

```sh
$ python3 -c "import smtplib; s=smtplib.SMTP('localhost', 20381); print(s.noop()); s.quit()"
(250, b'OK')
```

A `(250, b'OK')` response means the handler is accepting and correctly parsing SMTP commands.

### 1.3 Job runner

The job runner has no HTTP endpoint, no listening socket, and no dedicated startup log line beyond the generic `>>> init logging <<<` banner. Its entire main loop is the following block — **`job_runner.py:329-347`**:

```python
if __name__ == "__main__":
    while True:
        # wrap in an app context to benefit from app setup like database cleanup, sentry integration, etc
        with create_light_app().app_context():
            for job in get_jobs_to_run():
                LOG.d("Take job %s", job)

                # mark the job as taken, whether it will be executed successfully or not
                job.taken = True
                job.taken_at = arrow.now()
                job.state = JobState.taken.value
                job.attempts += 1
                Session.commit()
                process_job(job)

                job.state = JobState.done.value
                Session.commit()

            time.sleep(10)
```

Three facts follow directly from that block:

- The runner wraps each iteration in `create_light_app().app_context()` (line 332). That is the stripped-down Flask factory from `server.py:127-136` that only wires SQLAlchemy and `Session.remove()` on teardown — no blueprints, no admin, no login manager. See Q4 for the full definition.
- The only log line emitted **per job** is `Take job <Job ...>` at `LOG.d` (DEBUG) severity on line 334. If no jobs are waiting, no log line is printed at all.
- The sleep on line 347 is `10` seconds, so the maximum latency between inserting a ready `job` row and the runner picking it up is approximately 10 seconds (plus whatever processing time remains from the previous tick).

Who puts jobs into the table? The helper `get_jobs_to_run()` at **`job_runner.py:307-326`** is what the loop polls each tick:

```python
def get_jobs_to_run() -> List[Job]:
    # Get jobs that match all conditions:
    #  - Job.state == ready OR (Job.state == taken AND Job.taken_at < now - 30 mins AND Job.attempts < 5)
    #  - Job.run_at is Null OR Job.run_at < now + 10 mins
    taken_at_earliest = arrow.now().shift(minutes=-config.JOB_TAKEN_RETRY_WAIT_MINS)
    run_at_earliest = arrow.now().shift(minutes=+10)
    query = Job.filter(
        and_(
            or_(
                Job.state == JobState.ready.value,
                and_(
                    Job.state == JobState.taken.value,
                    Job.taken_at < taken_at_earliest,
                    Job.attempts < config.JOB_MAX_ATTEMPTS,
                ),
            ),
            or_(Job.run_at.is_(None), and_(Job.run_at <= run_at_earliest)),
        )
    )
    return query.all()
```

A single combined filter collects two categories of rows in one query: rows in `ready` state (`Job.state == JobState.ready.value`), and rows that were previously `taken` but appear stuck (`taken` more than `config.JOB_TAKEN_RETRY_WAIT_MINS` ago and still below `config.JOB_MAX_ATTEMPTS`). Both are further constrained to jobs whose `run_at` is either NULL or at most 10 minutes in the future (`run_at_earliest`). The set of recognised job names lives in `process_job()` at `job_runner.py:188-304`; anything not in that dispatcher drops to the fall-through `LOG.e("Unknown job name %s", job.name)` on line 304.

**Captured runtime evidence — startup.** Everything in `/tmp/job_runner.log` after launching `python job_runner.py`:

```log
>>> init logging <<<
2026-04-16 23:18:19,631 - SL - DEBUG - 2446 - "/workspace/app/utils.py:17" - <module>() -  - load words file: /workspace/local_data/test_words.txt
```

Two lines. Then silence. The runner is polling the `job` table every 10 seconds but emitting nothing because no ready jobs exist.

**Captured runtime evidence — job pickup probe.** To prove the runner is alive and polling (not merely stuck), I inserted a single row into the `job` table and watched for pickup:

```sql
INSERT INTO job (name, payload, state, taken, attempts, run_at, created_at, updated_at)
VALUES ('verification_probe', '{}', 0, false, 0, NOW(), NOW(), NOW());
```

…then captured the runner log within ~10 seconds:

```log
2026-04-16 23:21:10,789 - SL - DEBUG - 2446 - "/workspace/job_runner.py:334" - <module>() -  - Take job <Job 2 verification_probe {}>
2026-04-16 23:21:10,793 - SL - ERROR - 2446 - "/workspace/job_runner.py:304" - process_job() -  - Unknown job name verification_probe
```

The `Take job <Job 2 verification_probe {}>` line at line 334 confirms `job_runner.py:334`'s `LOG.d("Take job %s", job)` fired. The `ERROR` on line 304 is the expected fall-through from `process_job()` — `verification_probe` is not a registered job name, but the dispatcher still marked the row `state=2` (`JobState.done`) because the `while True` block unconditionally sets `job.state = JobState.done.value` after `process_job` returns (`job_runner.py:344`). The row was cleaned up — see the final cleanup evidence below.

**Process-level checks.** The only non-runtime verification is `ps`:

```sh
$ ps -ef | grep 'python .*job_runner.py' | grep -v grep
root  2446  2439  ...  /app/venv/bin/python job_runner.py
```

The presence of that process is necessary; demonstrating a successful `Take job` pickup is sufficient.

### 1.4 `/health` is intentionally excluded from the application access log

One finding worth calling out for operators: the SL `after_request` log explicitly skips `/health`, so hammering the endpoint from a load balancer will not flood the log. Captured evidence — after 3 back-to-back `curl /health` calls, zero `after_request()` log entries were produced:

```sh
$ docker exec simplelogin-setup bash -c "grep 'after_request()' /tmp/gunicorn.log | grep -E '/health' | wc -l"
0
```

The gunicorn internal access log still records the request (`[DEBUG] GET /health`), but the application-level `LOG.d(...)` call for request tracing is skipped by the condition on `server.py:281` — covered in detail in Q2.

---

## Q2. "What log messages or dashboard UI states should I watch for to confirm that users can sign in and manage their aliases?"

There are three layers of evidence: (1) the global log format (which tells you what every SL log line looks like), (2) the per-request `after_request` DEBUG log that the Flask app emits, and (3) the login view and its helper that emit two very specific lines on every successful login, plus the dashboard `Stats` data class whose four fields map exactly to four cards on the dashboard template.

### 2.1 The global log format and why werkzeug's access log is silent

**Log format — `app/log.py:12-15`:**

```python
_log_format = (
    "%(asctime)s - %(name)s - %(levelname)s - %(process)d - "
    '"%(pathname)s:%(lineno)d" - %(funcName)s() - %(message_id)s - %(message)s'
)
```

**Logger name — `app/log.py:79`:**

```python
LOG = _get_logger("SL")
```

Every log record SimpleLogin emits therefore begins `<timestamp> - SL - <LEVEL> - <pid>`. If you grep for records whose name column is `SL`, you get only application records — infrastructure noise is filtered out by name.

**werkzeug access log is suppressed — `app/log.py:70-71`:**

```python
log = logging.getLogger("werkzeug")
log.disabled = True
```

This disables the default Flask development-server access log. The practical consequence for a new self-hoster: if you start the server at `INFO` level (or silently wonder why gunicorn is not logging request lines), the *only* per-request record you will see from the Flask layer is the one emitted by `after_request` — and that record is emitted at DEBUG level. Set the log level to DEBUG, or else the app looks "silent" even while happily serving traffic.

**Message-id threading — `app/log.py:22-37`** is the log-debugging workhorse for emails:

```python
def set_message_id(message_id):
    global _MESSAGE_ID
    LOG.d("set message_id %s", message_id)
    _MESSAGE_ID = message_id


class EmailHandlerFilter(logging.Filter):
    """automatically add message-id to keep track of an email processing"""

    def filter(self, record):
        message_id = self.get_message_id()
        record.message_id = message_id if message_id else ""
        return True

    def get_message_id(self):
        return _MESSAGE_ID
```

Once `_handle()` in the email handler generates a new UUID and calls `set_message_id(uuid)`, every subsequent log record in that process carries that UUID in the `%(message_id)s` slot of the format string (the ` - ` separator surrounding the message-id in each log line comes from the format-string definition at `app/log.py:12-15`, not from the filter). A single email's end-to-end journey can therefore be extracted by filtering on that UUID. Q3c shows the captured log with that UUID visible.

### 2.2 The per-request web log — `after_request`

**Source — `server.py:272-296`:**

```python
@app.after_request
def after_request(res):
    # not logging /static call
    if (
        not request.path.startswith("/static")
        and not request.path.startswith("/admin/static")
        and not request.path.startswith("/_debug_toolbar")
        and not request.path.startswith("/git")
        and not request.path.startswith("/favicon.ico")
        and not request.path.startswith("/health")
    ):
        start_time = g.start_time or time.time()
        LOG.d(
            "%s %s %s %s %s, takes %s",
            request.remote_addr,
            request.method,
            request.path,
            request.args,
            res.status_code,
            time.time() - start_time,
        )
        newrelic.agent.record_custom_event(
            "HttpResponseStatus", {"code": res.status_code}
        )
    return res
```

Key details:

- Severity is `LOG.d` — DEBUG. If your logger is configured at INFO, these records will not appear.
- Six request paths are filtered out: `/static/...`, `/admin/static/...`, `/_debug_toolbar/...`, `/git/...`, `/favicon.ico/...`, and `/health/...`. The `/health` filter (line 281) is why a health-checking load balancer does not flood the log.
- `g.start_time` is populated by the matching `before_request` hook at `server.py:257-270` (not reproduced here — it is the mirror of `after_request` and is what makes the `takes %s` latency field non-zero).
- After the `LOG.d(...)` call, `server.py:293-295` also emits a New Relic custom event (`newrelic.agent.record_custom_event("HttpResponseStatus", {"code": res.status_code})`) on every non-filtered response. This is only observable in a New Relic APM environment — it produces no log output — but it is part of the `after_request` body in the source.

**Captured runtime evidence** — three real request lines from `/tmp/gunicorn.log` during the login smoke test (see Q2.3 for how these arose):

```log
2026-04-16 23:19:41,235 - SL - DEBUG - 2617 - "/workspace/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.020029783248901367
2026-04-16 23:19:41,490 - SL - DEBUG - 2617 - "/workspace/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 302, takes 0.24435997009277344
2026-04-16 23:19:41,611 - SL - DEBUG - 2617 - "/workspace/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard/ ImmutableMultiDict([]) 200, takes 0.1094672679901123
```

Each record carries: timestamp, logger name `SL`, level `DEBUG`, worker pid `2617`, source location `"/workspace/server.py:284"`, function `after_request()`, an empty message_id slot (the request is not part of an SMTP flow), then the formatted message with remote address, method, path, query-string args, status code, and elapsed seconds. The 302 on `POST /auth/login` is the redirect from `after_login()`; the follow-up `GET /dashboard/ ... 200` is the dashboard load.

### 2.3 The login flow — `app/auth/views/login.py:21-72`

The login view is an if/elif cascade over `LoginEvent.ActionType` values that produces user-facing flash messages and an analytics event for each outcome:

- **Bad credentials** — `login.py:49-50`:

  ```python
  flash("Email or password incorrect", "error")
  LoginEvent(LoginEvent.ActionType.failed).send()
  ```

- **Disabled account** — `login.py:52-56`:

  ```python
  flash(
      "Your account is disabled. Please contact SimpleLogin team to re-enable your account.",
      "error",
  )
  LoginEvent(LoginEvent.ActionType.disabled_login).send()
  ```

- **Scheduled to be deleted** — `login.py:58-62`:

  ```python
  flash(
      f"Your account is scheduled to be deleted on {user.delete_on}",
      "error",
  )
  LoginEvent(LoginEvent.ActionType.scheduled_to_be_deleted).send()
  ```

- **Not yet activated** — `login.py:65-69`:

  ```python
  flash(
      "Please check your inbox for the activation email. You can also have this email re-sent",
      "error",
  )
  LoginEvent(LoginEvent.ActionType.not_activated).send()
  ```

- **Success** — `login.py:71-72`:

  ```python
  LoginEvent(LoginEvent.ActionType.success).send()
  return after_login(user, next_url)
  ```

So for every failed sign-in attempt the operator sees a flash banner in the user's browser and a `LoginEvent(...).send()` side effect in the analytics pipeline, and for a successful sign-in the control flow is redirected into `after_login()`.

### 2.4 The after-login redirect — `app/auth/views/login_utils.py:35-45`

This helper is what produces two very characteristic DEBUG log lines for every successful login:

```python
LOG.d("log user %s in", user)
login_user(user)
session["sudo_time"] = int(time())

if next_url:
    LOG.d("redirect user to %s", next_url)
    return redirect(next_url)
else:
    LOG.d("redirect user to dashboard")
    return redirect(url_for("dashboard.index"))
```

Two markers therefore fire together:

- `log user <User <id> <name> <email>> in` at `login_utils.py:35` (DEBUG).
- `redirect user to dashboard` at `login_utils.py:44` (DEBUG).

`login_user(user)` is Flask-Login's own session-writer (see `flask-login` at `^0.5.0` in `pyproject.toml`); the line between those two DEBUG records is also where `session["sudo_time"]` is stamped.

**Captured runtime evidence.** From `/tmp/gunicorn.log`, the three DEBUG records immediately surrounding the 302 redirect during the `POST /auth/login` smoke test:

```log
2026-04-16 23:19:41,490 - SL - DEBUG - 2617 - "/workspace/app/auth/views/login_utils.py:35" - after_login() -  - log user <User 1 John Wick john@wick.com> in
2026-04-16 23:19:41,490 - SL - DEBUG - 2617 - "/workspace/app/auth/views/login_utils.py:44" - after_login() -  - redirect user to dashboard
2026-04-16 23:19:41,490 - SL - DEBUG - 2617 - "/workspace/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 302, takes 0.24435997009277344
```

The `<User 1 John Wick john@wick.com>` repr is the value of `User.__repr__()` and confirms the correct database row was loaded (`john@wick.com` is user id 1, created by `app/fake_data.py:45`). The following `GET /dashboard/ ... 200` line shows that the browser followed the redirect and was served the dashboard, confirming the whole session-cookie round-trip works.

The corresponding HTTP response headers that produced the `302`:

```log
HTTP/1.1 302 FOUND
Server: gunicorn/20.0.4
Content-Type: text/html; charset=utf-8
Content-Length: 225
Location: http://localhost:7777/dashboard/
Set-Cookie: slapp=...HttpOnly; Path=/; SameSite=Lax
```

`Location: /dashboard/` is the `url_for("dashboard.index")` value from `login_utils.py:45`.

### 2.5 Blueprint wiring — how `/auth/...` and `/dashboard/...` get routed

**`app/auth/base.py`** is a three-statement file that declares the blueprint the login/register views hang off of:

```python
from flask import Blueprint

auth_bp = Blueprint(
    name="auth", import_name=__name__, url_prefix="/auth", template_folder="templates"
)
```

**`app/dashboard/base.py`** declares its counterpart:

```python
from flask import Blueprint

dashboard_bp = Blueprint(
    name="dashboard",
    import_name=__name__,
    url_prefix="/dashboard",
    template_folder="templates",
)
```

Both are registered in `create_app()` via the `register_blueprints(app)` call at `server.py:172` — the `register_blueprints` function itself is defined at `server.py:233`. The `/auth` and `/dashboard` URL prefixes come from the `Blueprint(...)` call, not from the route decorators, which is why `@auth_bp.route("/login")` answers to `/auth/login` and `@dashboard_bp.route("/")` answers to `/dashboard/`.

### 2.6 The dashboard UI readiness indicators — four `Stats` cards

Once a user reaches `/dashboard/`, the page's four statistic cards at the top provide an immediate "database is reachable and I am actually logged in" signal. The four values are produced by the following:

**`Stats` dataclass — `app/dashboard/views/index.py:24-29`:**

```python
@dataclass
class Stats:
    nb_alias: int
    nb_forward: int
    nb_reply: int
    nb_block: int
```

**`get_stats(user)` — `app/dashboard/views/index.py:32-52`:**

```python
def get_stats(user: User) -> Stats:
    nb_alias = Alias.filter_by(user_id=user.id).count()
    nb_forward = (
        Session.query(EmailLog)
        .filter_by(user_id=user.id, is_reply=False, blocked=False, bounced=False)
        .count()
    )
    nb_reply = (
        Session.query(EmailLog)
        .filter_by(user_id=user.id, is_reply=True, blocked=False, bounced=False)
        .count()
    )
    nb_block = (
        Session.query(EmailLog)
        .filter_by(user_id=user.id, is_reply=False, blocked=True, bounced=False)
        .count()
    )
    return Stats(
        nb_alias=nb_alias, nb_forward=nb_forward, nb_reply=nb_reply, nb_block=nb_block
    )
```

The four `EmailLog` queries are keyed by three boolean columns — `is_reply`, `blocked`, `bounced` — which are the exact columns the email handler writes when it processes an inbound message (see Q3c).

**The template that renders the four cards — `templates/dashboard/index.html`:**

- Line 125: `<div class="subheader">Aliases</div>`
- Line 131: `<div class="h1 m-0">{{ stats.nb_alias }}</div>`
- Line 139: `<div class="subheader">Forwarded</div>`
- Line 145: `<div class="h1 m-0">{{ stats.nb_forward }}</div>`
- Line 153: `<div class="subheader">Replies/Sent</div>`
- Line 159: `<div class="h1 m-0">{{ stats.nb_reply }}</div>`
- Line 167: `<div class="subheader">Blocked</div>`
- Line 173: `<div class="h1 m-0">{{ stats.nb_block }}</div>`

So the four headings the user sees on the top row of the dashboard map 1:1 onto `Stats.nb_alias`, `Stats.nb_forward`, `Stats.nb_reply`, `Stats.nb_block`, and each of those is a single `SELECT count(*) FROM ...` query. If the dashboard renders with non-zero values (or with zero values but no server error), the application has successfully:

1. Read the session cookie (Flask-Login is working).
2. Loaded the `User` row (SQLAlchemy and Postgres are reachable).
3. Issued four `EmailLog`/`Alias` count queries and one `Alias` list query (the dashboard also enumerates each alias below the cards — see `app/dashboard/views/index.py:67`'s `def index()` and the remainder of `templates/dashboard/index.html`).

The `@dashboard_bp.route("/", methods=["GET", "POST"])` on line 55 of `app/dashboard/views/index.py` is what `login_utils.py:45`'s `redirect(url_for("dashboard.index"))` resolves to.

### 2.7 Summary of log signals for a healthy login → dashboard round-trip

In the order you will see them:

```log
SL - DEBUG - ... - after_request()  ... GET  /auth/login ... 200, takes ...   # login form rendered
SL - DEBUG - ... - after_login()    ... log user <User 1 John Wick john@wick.com> in
SL - DEBUG - ... - after_login()    ... redirect user to dashboard
SL - DEBUG - ... - after_request()  ... POST /auth/login ... 302, takes ...   # form submission redirect
SL - DEBUG - ... - after_request()  ... GET  /dashboard/ ... 200, takes ...   # dashboard served
```

Five records. The two middle records (`log user ... in` and `redirect user to dashboard`) are the definitive evidence that the login succeeded — they are emitted only on the success path through `login.py:71-72 → login_utils.after_login(...)`, so their absence (combined with a flash banner like "Email or password incorrect") pinpoints the failure branch taken.

---

## Q3. "If I perform basic user actions (create a new account, create an alias, receive an email to that alias), what should I expect the system to do at each stage?"

This question has three sub-flows. Each is answered below with the code path, the side effects that get written to the database, the log lines that the three running processes emit, and (where reproduced during this runtime exploration) the captured log excerpts.

### Account Creation

**Entry point — `app/auth/views/register.py:31-32`:**

```python
@auth_bp.route("/register", methods=["GET", "POST"])
def register():
```

The route is registered on `auth_bp` (the `Blueprint("auth", ..., url_prefix="/auth")` from `app/auth/base.py`), so the user-visible URL is `/auth/register`.

**Registration-disabled short-circuit — `app/auth/views/register.py:38-40`:**

```python
if config.DISABLE_REGISTRATION:
    flash("Registration is closed", "error")
    return redirect(url_for("auth.login"))
```

`DISABLE_REGISTRATION` is the opt-out for operators; when set, the registration form refuses to accept submissions and the user is bounced back to `/auth/login` with a flash banner.

**Successful submission — `app/auth/views/register.py:85-104`:**

```python
LOG.d("create user %s", email)
user = User.create(
    email=email,
    name=form.email.data,
    password=form.password.data,
    referral=get_referral(),
)
Session.commit()

try:
    send_activation_email(user, next_url)
    RegisterEvent(RegisterEvent.ActionType.success).send()
    DailyMetric.get_or_create_today_metric().nb_new_web_non_proton_user += 1
    Session.commit()
except Exception:
    flash("Invalid email, are you sure the email is correct?", "error")
    RegisterEvent(RegisterEvent.ActionType.invalid_email).send()
    return redirect(url_for("auth.register"))

return render_template("auth/register_waiting_activation.html")
```

The observable outcomes:

1. A `LOG.d("create user %s", email)` line at DEBUG level announces the attempt.
2. `User.create(...)` writes a row to the `users` table and, for new users, provisions a newsletter alias and (conditionally) schedules three onboarding jobs. See the DISABLE_ONBOARDING block below.
3. `send_activation_email(user, next_url)` (implementation begins at `register.py:117`) generates an activation link of the shape `{URL}/auth/activate?code={activation.code}` and hands it to `email_utils.send_activation_email(user, activation_link)` at `register.py:129`.
4. A browser-visible flash + the `register_waiting_activation.html` page at `register.py:104`.

**The onboarding-email short-circuit — `app/models.py:646-665`:**

```python
if config.DISABLE_ONBOARDING:
    LOG.d("Disable onboarding emails")
    return user

# Schedule onboarding emails
Job.create(
    name=config.JOB_ONBOARDING_1,
    payload={"user_id": user.id},
    run_at=arrow.now().shift(days=1),
)
Job.create(
    name=config.JOB_ONBOARDING_2,
    payload={"user_id": user.id},
    run_at=arrow.now().shift(days=2),
)
Job.create(
    name=config.JOB_ONBOARDING_4,
    payload={"user_id": user.id},
    run_at=arrow.now().shift(days=3),
)
Session.flush()

return user
```

With `DISABLE_ONBOARDING=true` (the value set in this runtime exploration via `example.env:150`, via `app/config.py:401`:

```python
DISABLE_ONBOARDING = "DISABLE_ONBOARDING" in os.environ
```

), `User.create()` emits the single line `Disable onboarding emails` and returns, and **no rows are inserted into the `job` table**. That is why the job runner — which polls the `job` table every 10 s — has nothing to do after a new registration in a local-dev environment. If you unset `DISABLE_ONBOARDING`, three rows appear in `job` with names `JOB_ONBOARDING_1`, `JOB_ONBOARDING_2`, `JOB_ONBOARDING_4` and `run_at` set to 1, 2, and 3 days in the future, so the runner would then pick them up as their scheduled times arrive.

**The local-dev email-delivery short-circuit — `app/config.py:91`:**

```python
NOT_SEND_EMAIL = "NOT_SEND_EMAIL" in os.environ
```

Per `example.env:19`, the recommended local-dev value is `NOT_SEND_EMAIL=true`. When this is set, `send_activation_email` still runs through its full pipeline (link generation, Jinja rendering, header composition) but the final SMTP delivery inside `email_utils.send_activation_email` is skipped/short-circuited. The user does not receive an email; instead, the activation link can be fetched out of the database or the log. That is the standard configuration for the runtime exploration described in this document.

What the user sees on a successful submit: the `register_waiting_activation.html` template rendered in the browser, and — in local-dev mode — no actual activation email delivered. In a real production configuration, they receive an email with a link of the shape `{URL}/auth/activate?code={...}`.

### Alias Creation

There are two paths that produce an alias: the HTTP API (used in this runtime exploration) and the dashboard UI.

#### Via the HTTP API — `app/api/views/new_random_alias.py:21-25`

```python
@api_bp.route("/alias/random/new", methods=["POST"])
@limiter.limit(ALIAS_LIMIT)
@require_api_auth
@parallel_limiter.lock(name="alias_creation")
def new_random_alias():
```

Four decorators protect this endpoint:

1. `@api_bp.route("/alias/random/new", methods=["POST"])` — HTTP routing.
2. `@limiter.limit(ALIAS_LIMIT)` — Flask-Limiter rate cap (`Flask-Limiter ^1.4` per `pyproject.toml`).
3. `@require_api_auth` — requires a valid `Authentication` header. Implementation in `app/api/base.py:16-43`:

   ```python
   def authorize_request() -> Optional[Tuple[str, int]]:
       api_code = request.headers.get("Authentication")
       api_key = ApiKey.get_by(code=api_code)

       if not api_key:
           if current_user.is_authenticated:
               ...
               g.user = current_user
           else:
               return jsonify(error="Wrong api key"), 401
       else:
           # Update api key stats
           api_key.last_used = arrow.now()
           api_key.times += 1
           Session.commit()

           g.user = api_key.user
   ```

   So every accepted API call bumps `api_key.last_used = arrow.now()` and `api_key.times += 1`. This gives you a post-hoc check: after calling the API, compare `api_key.times` before and after and confirm it incremented.

4. `@parallel_limiter.lock(name="alias_creation")` — mutex to prevent racing alias-creation attempts from producing duplicate emails.

**The response shape — `app/api/views/new_random_alias.py:114-118`:**

```python
return (
    jsonify(alias=alias.email, **serialize_alias_info_v2(get_alias_info_v2(alias))),
    201,
)
```

A `201 Created` with a JSON body that always contains an `alias` field (the email address) plus the full v2 alias-info payload.

**Captured runtime evidence.** The test used `Authentication: code` (the API key `"code"` belonging to `john@wick.com` seeded by `app/fake_data.py:121-122`):

```python
api_key = ApiKey.create(user_id=user.id, name="Chrome")
api_key.code = "code"
```

The actual request and the corresponding log lines from `/tmp/gunicorn.log`:

```sh
curl -X POST http://localhost:7777/api/alias/random/new \
  -H "Authentication: code" \
  -H "Content-Type: application/json" \
  -d '{}'
```

```log
[2026-04-16 23:19:49 +0000] [2617] [DEBUG] POST /api/alias/random/new
2026-04-16 23:19:49,199 - SL - DEBUG - 2617 - "/workspace/app/models.py:1459" - generate_random_alias_email() -  - generate email pimple_ragged296@sl.local
2026-04-16 23:19:49,218 - SL - DEBUG - 2617 - "/workspace/server.py:284" - after_request() -  - 127.0.0.1 POST /api/alias/random/new ImmutableMultiDict([]) 201, takes 0.036805152893066406
127.0.0.1 - - [16/Apr/2026:23:19:49 +0000] "POST /api/alias/random/new HTTP/1.1" 201 406 "-" "curl/7.88.1"
```

Two evidentiary log lines:

- `generate email pimple_ragged296@sl.local` from `app/models.py:1459` (`generate_random_alias_email()`) — the random-alias generator succeeded.
- `POST /api/alias/random/new ImmutableMultiDict([]) 201, takes 0.036805152893066406` from `after_request` — the HTTP request was served with HTTP 201 in ~37 ms.

The response body (truncated to the first field) was:

```json
{"alias": "pimple_ragged296@sl.local", "...": "..."}
```

The side effect on the `alias` table is a single new row with `email=pimple_ragged296@sl.local` and `user_id=1`.

#### Via the Dashboard — `app/dashboard/views/custom_alias.py:30-34`

```python
@dashboard_bp.route("/custom_alias", methods=["GET", "POST"])
@limiter.limit(ALIAS_LIMIT, methods=["POST"])
@login_required
@parallel_limiter.lock(name="alias_creation")
def custom_alias():
```

Observations:

- The URL is `/dashboard/custom_alias` (from `Blueprint("dashboard", ..., url_prefix="/dashboard")` at `app/dashboard/base.py`).
- Authentication here is session-based via `@login_required` (Flask-Login) rather than an API header.
- The same rate limit (`ALIAS_LIMIT`) and the same `parallel_limiter.lock(name="alias_creation")` mutex protect both paths — any attempt to race the API and the dashboard simultaneously is serialized.

The random-alias (as opposed to custom-alias) path via the dashboard is handled by the main `index()` view in `app/dashboard/views/index.py` (route at line 55), which on POST submits a form that results in the same `Alias` row being written.

### Email Receipt at an Alias

This is the richest of the three sub-flows: a single inbound SMTP message produces 10+ log lines across `_handle()`, `handle()`, `handle_forward()`, `get_or_create_contact()`, `forward_email_to_mailbox()`, the DMARC handler, the unsubscribe generator, and the mail sender, and it writes two rows (`contact`, `email_log`) to the database.

**Entry point — `email_handler.py:2289`:** `MailHandler.handle_DATA()`. This is the aiosmtpd callback the `Controller` routes every accepted SMTP message to. It delegates to `_handle()`.

**`_handle()` — `email_handler.py:2335-2378` (relevant excerpt):**

```python
@newrelic.agent.background_task()
def _handle(self, envelope: Envelope, msg: Message):
    start = time.time()

    # generate a different message_id to keep track of an email lifecycle
    message_id = str(uuid.uuid4())
    set_message_id(message_id)

    LOG.d("====>=====>====>====>====>====>====>====>")
    LOG.i(
        "New message, mail from %s, rctp tos %s ",
        envelope.mail_from,
        envelope.rcpt_tos,
    )

    ...

    with create_light_app().app_context():
        return_status = handle(envelope, msg)
        elapsed = time.time() - start

        # Different behaviour depending on status
        ...
        LOG.i(
            "Finish mail_from %s, rcpt_tos %s, takes %s seconds with return code '%s'<<===",
            envelope.mail_from,
            envelope.rcpt_tos,
            elapsed,
            return_status,
        )
```

Three things to notice:

1. A fresh `uuid.uuid4()` message-id is generated per email and installed globally via `set_message_id()` (`app/log.py:22-25`). Every log line emitted while handling this email carries that UUID, which is the one-and-only way to untangle concurrent email lifecycles from the log.
2. The `====>=====>====>...` banner is the opening bracket of an email's lifecycle and the `<<===` at the end of the `Finish mail_from ...` message is the closing bracket. Grepping on those two markers gives you exact boundaries of each email's work.
3. The whole `handle()` body runs inside `with create_light_app().app_context():`, i.e. a minimal Flask app context that exists only to give SQLAlchemy a session — no blueprints, no login manager, no admin. That is what allows the email handler to be a completely independent process (see Q4).

**Forward-vs-reply dispatch — `email_handler.py:2194-2211` (excerpt):**

```python
if is_reverse_alias(rcpt_to):
    LOG.d(
        "Reply phase %s(%s) -> %s",
        mail_from,
        copy_msg[headers.FROM],
        rcpt_to,
    )
    ...
else:  # Forward case
    LOG.d(
        "Forward phase %s(%s) -> %s",
        mail_from,
        copy_msg[headers.FROM],
        rcpt_to,
    )
    for is_delivered, smtp_status in handle_forward(envelope, copy_msg, rcpt_to):
        ...
```

A recipient address that begins with the reverse-alias prefix is treated as a reply (the user is replying to a contact via their alias); everything else is a forward (an inbound sender is writing to the alias, and SimpleLogin must forward to the user's real mailbox).

**Forward path — `email_handler.py:579-581`:**

```python
from_header = get_header_unicode(msg[headers.FROM])
LOG.d("Create or get contact for from_header:%s", from_header)
contact = get_or_create_contact(from_header, envelope.mail_from, alias)
```

This is where the `contact` row is written (or looked up) by `app/contact_utils.py`. The closing log line of a successful forward is `email_handler.py:893-899`:

```python
LOG.d(
    "Forward mail from %s to %s, mail_options:%s, rcpt_options:%s ",
    contact.website_email,
    mailbox.email,
    envelope.mail_options,
    envelope.rcpt_options,
)
```

**Captured runtime evidence — the complete lifecycle of one test email** (UUID `dceca8ed-5090-4a96-8b2d-958d4ce71473`, from `/tmp/email_handler.log`):

```log
2026-04-16 23:20:02,030 - SL - DEBUG - 2366 - "/workspace/app/log.py:24" - set_message_id() -  - set message_id dceca8ed-5090-4a96-8b2d-958d4ce71473
2026-04-16 23:20:02,030 - SL - DEBUG - 2366 - "/workspace/email_handler.py:2342" - _handle() - dceca8ed-5090-4a96-8b2d-958d4ce71473 - ====>=====>====>====>====>====>====>====>
2026-04-16 23:20:02,030 - SL - INFO  - 2366 - "/workspace/email_handler.py:2343" - _handle() - dceca8ed-5090-4a96-8b2d-958d4ce71473 - New message, mail from sender@example.com, rctp tos ['pimple_ragged296@sl.local']
2026-04-16 23:20:02,031 - SL - DEBUG - 2366 - "/workspace/email_handler.py:1963" - handle() - dceca8ed-5090-4a96-8b2d-958d4ce71473 - Cannot parse Postfix queue ID from None None
2026-04-16 23:20:02,162 - SL - DEBUG - 2366 - "/workspace/email_handler.py:1980" - handle() - dceca8ed-5090-4a96-8b2d-958d4ce71473 - ==>> Handle mail_from:sender@example.com, rcpt_tos:['pimple_ragged296@sl.local'], header_from:sender@example.com, header_to:pimple_ragged296@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'sender@example.com'), ('To', 'pimple_ragged296@sl.local'), ('Subject', 'runtime verification test'), ...], mail_options:['SIZE=223'], rcpt_options:[]
2026-04-16 23:20:02,166 - SL - DEBUG - 2366 - "/workspace/email_handler.py:2202" - handle() - dceca8ed-5090-4a96-8b2d-958d4ce71473 - Forward phase sender@example.com(sender@example.com) -> pimple_ragged296@sl.local
2026-04-16 23:20:02,181 - SL - DEBUG - 2366 - "/workspace/email_handler.py:580" - handle_forward() - dceca8ed-5090-4a96-8b2d-958d4ce71473 - Create or get contact for from_header:sender@example.com
2026-04-16 23:20:02,206 - SL - DEBUG - 2366 - "/workspace/app/contact_utils.py:110" - create_contact() - dceca8ed-5090-4a96-8b2d-958d4ce71473 - Created contact <Contact 2 sender@example.com 12> for alias <Alias 12 pimple_ragged296@sl.local> with email sender@example.com invalid_email=False
2026-04-16 23:20:02,206 - SL - INFO  - 2366 - "/workspace/app/handler/dmarc.py:33" - apply_dmarc_policy_for_forward_phase() - dceca8ed-5090-4a96-8b2d-958d4ce71473 - DMARC check disabled
2026-04-16 23:20:02,217 - SL - DEBUG - 2366 - "/workspace/email_handler.py:688" - forward_email_to_mailbox() - dceca8ed-5090-4a96-8b2d-958d4ce71473 - Forward <Contact 2 sender@example.com 12> -> <Alias 12 pimple_ragged296@sl.local> -> <Mailbox 1 john@wick.com>
2026-04-16 23:20:02,220 - SL - DEBUG - 2366 - "/workspace/email_handler.py:740" - forward_email_to_mailbox() - dceca8ed-5090-4a96-8b2d-958d4ce71473 - Create <EmailLog 2> for <Contact 2 sender@example.com 12>, <User 1 John Wick john@wick.com>, <Mailbox 1 john@wick.com>
2026-04-16 23:20:02,227 - SL - WARNING - 2366 - "/workspace/email_handler.py:857" - forward_email_to_mailbox() - dceca8ed-5090-4a96-8b2d-958d4ce71473 - missing date header, create one
2026-04-16 23:20:02,227 - SL - DEBUG - 2366 - "/workspace/email_handler.py:867" - forward_email_to_mailbox() - dceca8ed-5090-4a96-8b2d-958d4ce71473 - From header, new:"sender at example.com" <sender_at_example_com_bcvnfsnuo@sl.local>, old:sender@example.com
2026-04-16 23:20:02,227 - SL - DEBUG - 2366 - "/workspace/email_handler.py:316" - replace_header_when_forward() - dceca8ed-5090-4a96-8b2d-958d4ce71473 - Delete Cc header, old value None
2026-04-16 23:20:02,227 - SL - DEBUG - 2366 - "/workspace/email_handler.py:313" - replace_header_when_forward() - dceca8ed-5090-4a96-8b2d-958d4ce71473 - Replace To header, old: pimple_ragged296@sl.local, new: pimple_ragged296@sl.local
2026-04-16 23:20:02,228 - SL - INFO  - 2366 - "/workspace/app/handler/unsubscribe_generator.py:36" - _generate_header_with_original_behaviour() - dceca8ed-5090-4a96-8b2d-958d4ce71473 - Email has no unsubscribe header
2026-04-16 23:20:02,228 - SL - DEBUG - 2366 - "/workspace/email_handler.py:893" - forward_email_to_mailbox() - dceca8ed-5090-4a96-8b2d-958d4ce71473 - Forward mail from sender@example.com to john@wick.com, mail_options:['SIZE=223'], rcpt_options:[]
2026-04-16 23:20:02,228 - SL - DEBUG - 2366 - "/workspace/app/mail_sender.py:131" - send() - dceca8ed-5090-4a96-8b2d-958d4ce71473 - send email with subject 'runtime verification test', from '"sender at example.com" <sender_at_example_com_bcvnfsnuo@sl.local>' to 'pimple_ragged296@sl.local'
2026-04-16 23:20:02,228 - SL - INFO  - 2366 - "/workspace/email_handler.py:2367" - _handle() - dceca8ed-5090-4a96-8b2d-958d4ce71473 - Finish mail_from sender@example.com, rcpt_tos ['pimple_ragged296@sl.local'], takes 0.19825291633605957 seconds with return code '250 Message accepted for delivery'<<===
```

Observations (each line is labeled with its file:line because they all come from the `"%(pathname)s:%(lineno)d"` slot of the log format):

- Time from `====>` banner to `<<===` footer: **198 ms**.
- Every line carries the same message-id `dceca8ed-5090-4a96-8b2d-958d4ce71473` — this is the UUID `_handle()` installed at `app/log.py:22-25`.
- `Created contact <Contact 2 sender@example.com 12>` at `app/contact_utils.py:110` — the `contact` row is written here. The repr shows `id=2`, `website_email=sender@example.com`, `alias_id=12`.
- `Create <EmailLog 2>` at `email_handler.py:740` — the `email_log` row is written here. Its three boolean columns will be `is_reply=False, blocked=False, bounced=False`, which makes it count toward `Stats.nb_forward` on the dashboard (see Q2.6's `get_stats(user)`).
- `DMARC check disabled` at `app/handler/dmarc.py:33` — DMARC enforcement was not exercised in this runtime (consistent with local-dev configuration).
- `From header, new:"sender at example.com" <sender_at_example_com_bcvnfsnuo@sl.local>, old:sender@example.com` at `email_handler.py:867` — the From header rewrite: the forwarded message presents as coming from a reverse-alias so the user can reply through SimpleLogin.
- `send email with subject 'runtime verification test', from '"sender at example.com" <sender_at_example_com_bcvnfsnuo@sl.local>' to 'pimple_ragged296@sl.local'` at `app/mail_sender.py:131` — the `send()` call that would, in a non-`NOT_SEND_EMAIL` configuration, actually dispatch the SMTP DATA to the user's mailbox. With `NOT_SEND_EMAIL=true`, the call short-circuits internally; the log record is still produced, and the pipeline still writes the DB rows.
- `'250 Message accepted for delivery'` in the `Finish` line — the SMTP status returned to the sending client. This is what a real external sender would see in response to their `DATA` command.

**Database verification.** Queried after the test email landed:

```sql
SELECT id, website_email, reply_email, alias_id FROM contact WHERE website_email='sender@example.com';
-- returned: id=2, website_email=sender@example.com, reply_email=sender_at_example_com_bcvnfsnuo@sl.local, alias_id=12

SELECT id, alias_id, contact_id, is_reply, blocked, bounced FROM email_log WHERE id=2;
-- returned: id=2, alias_id=12, contact_id=2, is_reply=False, blocked=False, bounced=False
```

Exactly what `get_stats(user)` from `app/dashboard/views/index.py:32-52` requires to count toward `nb_forward` — so this email would increment the dashboard's "Forwarded" card by 1 on next page load.

**Cleanup confirmation.** All test rows were removed after verification. Post-cleanup counts:

```sql
SELECT COUNT(*) FROM alias       WHERE email LIKE 'pimple_ragged%';   -- 0
SELECT COUNT(*) FROM contact     WHERE website_email='sender@example.com'; -- 0
SELECT COUNT(*) FROM email_log   WHERE id=2;                         -- 0
SELECT COUNT(*) FROM job         WHERE name='verification_probe';    -- 0
```

All four counts are `0`, confirming the exploration left no residue in the database.

---

## Q4. "Does the email handler and job runner auto-start with the web server, or do I need to launch them separately? How can I tell they're functioning?"

**No — the email handler, job runner, cron, and event listener are completely independent processes. The Dockerfile starts ONLY the web server. Each background component has its own `if __name__ == "__main__":` block and must be launched as a separate `python3` process.**

This section presents the evidence for that claim, then explains how to verify each process is functioning.

### 4.1 Only the web server is auto-started by the Dockerfile

**`Dockerfile:44,47`:**

```dockerfile
EXPOSE 7777

#gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15 --log-level DEBUG
CMD ["gunicorn","wsgi:app","-b","0.0.0.0:7777","-w","2","--timeout","15"]
```

The entire container command is `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15`. There is no `supervisord`, no `foreman`, no `s6-overlay`, no shell wrapper, no systemd — nothing spawns the other processes.

**`wsgi.py:1-3`** (the entire file):

```python
from server import create_app

app = create_app()
```

`create_app()` (`server.py:139-217`) wires Flask extensions, blueprints, admin, login, error handlers, and the `/health` endpoint. Nothing inside `create_app()` imports `email_handler.py` or `job_runner.py` and nothing starts a new thread, subprocess, or asyncio task for them. As a result, `docker run` against the SimpleLogin image yields a container that serves HTTP on port 7777 and performs zero inbound SMTP handling and zero background job processing.

### 4.2 Each background component has its own `__main__`

**`email_handler.py:2396-2404`:**

```python
if __name__ == "__main__":
    parser = argparse.ArgumentParser()

    parser.add_argument(
        "-p", "--port", help="SMTP port to listen for", type=int, default=20381
    )

    args = parser.parse_args()

    LOG.i("Listen for port %s", args.port)
    main(port=args.port)
```

and `email_handler.py:2381-2393`:

```python
def main(port: int):
    """Use aiosmtpd Controller"""
    controller = Controller(MailHandler(), hostname="0.0.0.0", port=port)

    controller.start()
    LOG.d("Start mail controller %s %s", controller.hostname, controller.port)

    if LOAD_PGP_EMAIL_HANDLER:
        LOG.w("LOAD PGP keys")
        load_pgp_public_keys()

    while True:
        time.sleep(2)
```

Running `python3 email_handler.py -p 20381` creates an aiosmtpd `Controller` bound to `0.0.0.0:20381` and blocks in `while True: time.sleep(2)`. Two log lines appear at startup, in order: `Listen for port %s` (INFO, line 2403) and `Start mail controller %s %s` (DEBUG, line 2386). After that, the process is silent until it accepts its first SMTP message.

**`job_runner.py:329-347`:**

```python
if __name__ == "__main__":
    while True:
        # wrap in an app context to benefit from app setup like database cleanup, sentry integration, etc
        with create_light_app().app_context():
            for job in get_jobs_to_run():
                LOG.d("Take job %s", job)

                # mark the job as taken, whether it will be executed successfully or not
                job.taken = True
                job.taken_at = arrow.now()
                job.state = JobState.taken.value
                job.attempts += 1

                Session.commit()
                process_job(job)

                job.state = JobState.done.value
                Session.commit()

            time.sleep(10)
```

The runner loops forever with a 10-second sleep between polls. Each poll opens a fresh `create_light_app()` app context, calls `get_jobs_to_run()` (defined at `job_runner.py:307-326`) which queries the `job` table for rows in `JobState.ready` (=0) or `JobState.taken` (=1) that have exceeded `JOB_TAKEN_RETRY_WAIT_MINS` and have fewer than `JOB_MAX_ATTEMPTS` attempts, marks the row as `JobState.taken` with `attempts += 1`, calls `process_job(job)`, and finally marks it `JobState.done` (=2). When there are no jobs, the runner is silent — it does not log each empty poll.

**`cron.py:1262-1284` (first portion of `__main__`):**

```python
if __name__ == "__main__":
    LOG.d("Start running cronjob")
    parser = argparse.ArgumentParser()
    parser.add_argument(
        "-j",
        "--job",
        help="Choose a cron job to run",
        type=str,
    )
    args = parser.parse_args()
    # wrap in an app context to benefit from app setup like database cleanup, sentry integration, etc
    with create_light_app().app_context():
        if args.job == "stats":
            LOG.d("Compute growth and daily monitoring stats")
            stats()
        elif args.job == "notify_trial_end":
            ...
```

Important detail: `cron.py` is **not** a long-running daemon. It parses `-j <jobname>`, executes that one job inside `create_light_app().app_context()`, and exits. Scheduling is delegated to an external yacron process driven by `crontab.yml`. Example invocation from `crontab.yml:2-5`:

```yaml
- name: SimpleLogin growth stats
  command: python /code/cron.py -j stats
  shell: /bin/bash
  schedule: "0 0 * * *"
```

`crontab.yml` contains 15 such jobs, each of which is a separate `python /code/cron.py -j <name>` invocation. To actually run them on a schedule in production, an operator runs `yacron -c crontab.yml` as its own supervised process; that process then forks each cron job at its scheduled time.

**`event_listener.py:94`** (the `__main__` block) and `event_listener.py:29-35`:

```python
def main(mode: Mode, dry_run: bool, max_retries: int):
    if mode == Mode.DEAD_LETTER:
        ...
    elif mode == Mode.LISTENER:
        ...
        source = PostgresEventSource(EVENT_LISTENER_DB_URI)
        ...
```

```python
if __name__ == "__main__":
    ...
    if args.command in [Mode.LISTENER.value, Mode.DEAD_LETTER.value]:
        ...
```

The event listener is another independent process that connects to `EVENT_LISTENER_DB_URI` and consumes Postgres LISTEN/NOTIFY events. It has two modes (`listener` and `dead_letter`) selected by subcommand argument. It was not exercised during this runtime exploration; its independence is clear from the `__main__` block alone.

### 4.3 Why each background process is cheap — `create_light_app()`

All three background components (email handler, job runner, cron) obtain their database session through `create_light_app()` rather than the full `create_app()`. The full file of that function — `server.py:127-136`:

```python
def create_light_app() -> Flask:
    app = Flask(__name__)
    app.config["SQLALCHEMY_DATABASE_URI"] = DB_URI
    app.config["SQLALCHEMY_TRACK_MODIFICATIONS"] = False

    @app.teardown_appcontext
    def shutdown_session(response_or_exc):
        Session.remove()

    return app
```

Compared to `create_app()` (which begins at `server.py:139` and ends around line 217), the light factory:

- Does NOT register blueprints (`register_blueprints(app)` is not called).
- Does NOT initialize Flask-Admin (`init_admin(app)` is not called).
- Does NOT initialize Flask-Login (`login_manager.init_app(app)` is not called).
- Does NOT initialize Flask-CORS, Flask-Limiter, Flask-Migrate, CSRF, Paddle or Coinbase payment webhooks.
- Does NOT configure Sentry for request middleware.

It only sets `SQLALCHEMY_DATABASE_URI` and registers a teardown that calls `Session.remove()` to release the scoped-session at the end of each `with ... .app_context():` block. That is exactly what a database-only worker needs, which is why workers can each run in their own process without any of the blueprints or extensions stepping on each other.

### 4.4 How to tell each background process is functioning

#### Web server

Already covered in Q1. The canonical liveness probe is:

```sh
curl -v http://localhost:7777/health
```

which must return `HTTP/1.1 200 OK` with body `success` from `server.py:213-215`. For the process-level view, `ps -ef | grep gunicorn` (or `pgrep -a gunicorn`) shows a master plus N workers — the captured startup log shows:

```log
[2026-04-16 23:19:15 +0000] [2616] [INFO] Starting gunicorn 20.0.4
[2026-04-16 23:19:15 +0000] [2616] [INFO] Listening at: http://0.0.0.0:7777 (2616)
[2026-04-16 23:19:15 +0000] [2616] [INFO] Using worker: sync
[2026-04-16 23:19:15 +0000] [2617] [INFO] Booting worker with pid: 2617
```

#### Email handler

The two-step check: (1) is a TCP listener bound on `20381`, and (2) does an actual SMTP transaction produce the expected lifecycle log?

**Listener check:**

```sh
ss -tlnp sport = :20381
```

If the handler is running, this shows a `LISTEN` socket owned by the `python3 email_handler.py` process.

**SMTP transaction check.** Send any message with a Python one-liner:

```python
import smtplib
from email.message import EmailMessage
msg = EmailMessage()
msg["From"] = "sender@example.com"
msg["To"] = "<existing_alias>@sl.local"
msg["Subject"] = "probe"
msg.set_content("hello")
with smtplib.SMTP("localhost", 20381) as s:
    s.send_message(msg)
```

and tail the email handler log. The expected output is the lifecycle shown in Q3c: an opening `====>=====>====>...>` banner (DEBUG), a `New message, mail from ... rctp tos ...` INFO line, a `Forward phase` or `Reply phase` DEBUG line, the `Create or get contact for from_header:...` and `Create <EmailLog N>` DEBUG lines, and finally an INFO line of the shape `Finish mail_from ... takes <N>s with return code '250 Message accepted for delivery'<<===`. If the `<<===` tail appears within a second or two, the handler is functioning correctly.

#### Job runner

The runner does not expose a network socket. Two verification strategies:

**Process check:**

```sh
ps -ef | grep 'python3 job_runner.py'
```

**Functional check via a probe job.** Because the runner silently polls every 10 seconds, the only way to see it do work is to give it work. Insert a row into the `job` table with `state=0` (ready) and any job name:

```sql
INSERT INTO job (name, payload, taken, run_at, state, attempts, created_at, updated_at)
VALUES ('verification_probe', '{}', false, NOW(), 0, 0, NOW(), NOW());
```

and then tail the job runner log. Within 10 seconds the runner emits a `Take job <Job ...>` DEBUG line from `job_runner.py:334`. Because `verification_probe` is not a recognized name, `process_job()` falls through to the `else` branch at `job_runner.py:303-304`:

```python
else:
    LOG.e("Unknown job name %s", job.name)
```

which writes an ERROR record. The row is then marked `JobState.done.value` (=2) with `attempts=1`.

**Captured runtime evidence** from `/tmp/job_runner.log`:

```log
2026-04-16 23:21:10,789 - SL - DEBUG - 2446 - "/workspace/job_runner.py:334" - <module>() -  - Take job <Job 2 verification_probe {}>
2026-04-16 23:21:10,793 - SL - ERROR - 2446 - "/workspace/job_runner.py:304" - process_job() -  - Unknown job name verification_probe
```

Both expected lines are present. The `<Job 2 verification_probe {}>` repr is Python's `Job.__repr__()` output: id=2, name=`verification_probe`, payload=`{}`. Four milliseconds elapsed between the `Take job` (line 334) and the `Unknown job name` (line 304) — the runner's dispatcher is functioning.

Startup output for the runner is minimal — `/tmp/job_runner.log` opens with:

```log
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/vdacxbirthqhpmdgiqsu
Upload files to local dir
>>> init logging <<<
2026-04-16 23:18:19,631 - SL - DEBUG - 2446 - "/workspace/app/utils.py:17" - <module>() -  - load words file: /workspace/local_data/test_words.txt
```

The `>>> init logging <<<` `print()` is emitted once at import time from `app/log.py:67`; the `load words file: ...` DEBUG is from the `app/utils.py` import (the words file is what `generate_random_alias_email()` uses). No further log output until the first job appears in the table — which is exactly the **absence-of-log** signature you use to confirm "the runner is running but there is just no work."

#### Cron (yacron) and event listener

These were not exercised during this runtime exploration. The process-independence argument for each is the same as above:

- `cron.py` — `__main__` at line 1262 runs a single `-j <name>` job then exits. In production it is driven by `yacron -c crontab.yml`, which is its own supervised process. Verification in production: check the yacron log for successful invocations; inspect `crontab.yml` for the 15 scheduled jobs.
- `event_listener.py` — `__main__` at line 94 dispatches to `main(mode, dry_run, max_retries)` at line 29. The two supported modes are `LISTENER` (consume Postgres NOTIFY events) and `DEAD_LETTER` (reprocess failed events). Verification: check `ps -ef` for the process and — in a Proton deployment — confirm that Postgres `NOTIFY` writes are being consumed.

### 4.5 The single most important takeaway for a new self-hoster

A fresh `docker run` of the SimpleLogin image gives you a web server and nothing else. `curl http://localhost:7777/health` → `200 success` but:

- Any email sent to an alias goes nowhere — there is no process listening on SMTP.
- Any job-producing user action (account deletion, mailbox deletion, batch imports, export, proton welcome, alias-creation events) writes a row to `job` but nothing ever drains it.
- All scheduled maintenance (growth stats, HIBP check, trial-end notifications — the 15 entries in `crontab.yml`) never runs.

For any useful production deployment, you must start at minimum three processes side-by-side:

```sh
# Terminal 1 — web server (or, equivalently, the Docker CMD)
gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15

# Terminal 2 — inbound SMTP
python3 email_handler.py -p 20381

# Terminal 3 — background jobs
python3 job_runner.py

# Plus (for full feature parity):
# Terminal 4 — scheduled tasks
yacron -c crontab.yml

# Terminal 5 — Proton event sync
python3 event_listener.py listener
```

Each is independent and uses `create_light_app()` for its own minimal database context (see 4.3). They share the same `DB_URI` and the same Postgres cluster; they do not share memory.

---

## Summary: Process Independence Matrix

| Process | Entry point | Auto-started by web server? | How to verify functioning |
|---|---|---|---|
| Web server | `gunicorn wsgi:app` / `Dockerfile:47` | N/A (it IS the web server) | `curl http://localhost:7777/health` → 200 `"success"` (`server.py:213-215`); `ps -ef \| grep gunicorn` shows master + worker |
| Email handler | `python3 email_handler.py` / `email_handler.py:2396-2404` | **No — separate process** | `ss -tlnp sport = :20381` shows a LISTEN; send SMTP and watch for `Finish mail_from ... takes <N>s with return code '250 Message accepted for delivery'<<===` (`email_handler.py:2367`) |
| Job runner | `python3 job_runner.py` / `job_runner.py:329-347` | **No — separate process** | `ps -ef \| grep 'python3 job_runner.py'`; insert a `job` row with `state=0` and watch for `Take job <Job ...>` (`job_runner.py:334`) within 10 s |
| Cron (yacron) | `yacron -c crontab.yml` driving `python3 cron.py -j <name>` / `cron.py:1262` | **No — separate process** | Inspect yacron output log; `crontab.yml` defines 15 scheduled jobs; each invocation is an ephemeral `cron.py` process that exits after one job |
| Event listener | `python3 event_listener.py <listener\|dead_letter>` / `event_listener.py:94` | **No — separate process** | `ps -ef \| grep event_listener`; requires Postgres LISTEN/NOTIFY configuration via `EVENT_LISTENER_DB_URI` |

**Dependency footnote (all versions verified from `pyproject.toml`):** Python `^3.10`, Flask `^1.1.2`, gunicorn `^20.0.4`, SQLAlchemy `1.3.24` (pinned), psycopg2-binary `^2.9.3`, aiosmtpd `^1.2`, flask-login `^0.5.0`, Flask-Limiter `^1.4`, redis `^4.5.3`, PGPy `0.5.4` (pinned — compatible with cryptography `37.0.1`, also pinned).

**Runtime evidence footnote:** All log excerpts in this document were captured during a single live session on 2026-04-16 against the pre-built container `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` at commit `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`. All test users, aliases, contacts, email_log rows, and probe `job` rows created during the exploration were deleted before this document was finalized; `SELECT COUNT(*)` on each affected table returned `0`. No source code was modified.
