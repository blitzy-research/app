# SimpleLogin Runtime Q&A — Branch `app_2cd6ee777f8c`

This document answers **five runtime‑behavior questions** about the
[SimpleLogin](https://github.com/simple-login/app) email‑aliasing application on branch
`app_2cd6ee777f8c` (the repository `.version` file contains `dev`). It is written **run‑first**:
every answer below was produced by **actually building and running** the relevant code path
and **capturing the real output**. Log lines, timings (ms), counts, HTTP responses (headers,
status codes, bodies), and SQL result rows are quoted **verbatim** in fenced blocks alongside
the exact command that produced them. Every factual claim is grounded in a `file:line`
reference or in the observed output. Where a value depends on a chosen convention (e.g. whether
to count Alembic's bookkeeping table, or which line is the timing baseline), the choice is
stated explicitly.

## Environment actually used

All observations were captured inside the project's mandated Docker image
`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (container `sl-setup`), which
provides the project's pinned runtime. Values are specific to this environment and are **not**
generalized beyond it.

| Component | Version / value (observed) |
|-----------|----------------------------|
| Python | 3.10.18 (venv at `/app/venv`) — project targets `FROM python:3.10` (`Dockerfile:8`), `python = "^3.10"` (`pyproject.toml`) |
| PostgreSQL | 15.13, listening on host port `15432` |
| Redis | 7.0.15, listening on `6379` |
| Flask / Werkzeug / Gunicorn | 1.1.2 / 1.0.1 / 20.0.4 (observed `Starting gunicorn 20.0.4`) |
| SQLAlchemy / Alembic / Flask‑Migrate | 1.3.24 / 1.4.3 / 2.5.3 |
| aiosmtpd / psycopg2‑binary | 1.4.2 / 2.9.3 |
| DB connection | `postgresql://test:test@localhost:15432/test` |
| Web server bind | `0.0.0.0:7777`; email handler bind `0.0.0.0:25025` |
| Config selection | `CONFIG` env var → `.env` file, via `python-dotenv` (`app/config.py:65-71`) |

**Config note (important for R5):** the checked‑in `tests/test.env:13` sets
`MAX_NB_EMAIL_FREE_PLAN=3`. To observe the genuine code default of `5` for the R5 "before"
state, the runtime used a **throwaway** config `base.env` derived from `tests/test.env` with the
`MAX_NB_EMAIL_FREE_PLAN` line **removed** (so `app/config.py:120-124`'s default `5` applies),
and a throwaway `after.env` with `MAX_NB_EMAIL_FREE_PLAN=10`. No committed file was edited; both
throwaway files were deleted afterward (per the read‑only‑source rule).

**Read‑only scope:** no existing repository file was modified. The only committed change is this
document. All temporary scripts / `.env` files used for observation were removed afterward.

---

## R1 — Empty‑database migration footprint

**Question.** When migrations are run against a fresh, empty PostgreSQL database, **(a)** how many
tables are created in total, and **(b)** what is the exact name of the *last* table created (by
migration output order)?

### Commands executed

Mirroring the project's own empty‑DB bootstrap (`scripts/reset_local_db.sh:4,6`,
`scripts/run-test.sh:13`) but **skipping** `flask dummy-data` (`scripts/reset_local_db.sh:7`,
which inserts rows, not tables):

```bash
# 1) Provision an EMPTY database (drop + recreate the public schema)
echo 'drop schema public cascade; create schema public;' \
  | psql -h localhost -p 15432 -U test -d test        # scripts/reset_local_db.sh:4

# verify empty
psql ... -tAc "SELECT count(*) FROM information_schema.tables
               WHERE table_schema NOT IN ('pg_catalog','information_schema');"
# -> 0

# 2) Run all migrations to head with SQL echo so CREATE TABLE order is visible.
#    (throwaway alembic ini = copy of alembic.ini with script_location=/app/migrations
#     and [logger_sqlalchemy] level=INFO; the default alembic.ini logs sqlalchemy at WARN)
CONFIG=tests/test.env /app/venv/bin/alembic -c /tmp/inv/alembic_echo.ini upgrade head
# alembic exit code = 0
```

The migration environment binds `target_metadata = Base.metadata` (`migrations/env.py:28`), the
script location resolves to `migrations/` (`alembic.ini:5`), and the schema is built by walking
**255** revision files under `migrations/versions/` to head.

### (a) Total number of tables — from the live schema (authoritative)

```bash
psql -h localhost -p 15432 -U test -d test -tAc \
 "SELECT count(*) FROM information_schema.tables
  WHERE table_schema NOT IN ('pg_catalog','information_schema');"
```
```
77
```
Adding `AND table_type='BASE TABLE'` returns the same number:
```
77
```
The `alembic_version` bookkeeping table **is** present and **is** included in this count:
```bash
psql ... -tAc "SELECT table_name FROM information_schema.tables WHERE table_name='alembic_version';"
```
```
alembic_version
```

> **Answer R1(a): 77 tables** exist in the live `public` schema after `alembic upgrade head`.
> This number **includes** Alembic's internal `alembic_version` table, so it is
> **76 application tables + 1 `alembic_version` = 77**.

**Why the live count is authoritative (not the number of `CREATE TABLE` calls).** With SQL echo
on, the migration run emitted **81** `CREATE TABLE` statements, yet only 77 tables remain. The
first emitted `CREATE TABLE` (log line 35) is Alembic's own bookkeeping table, created by
Alembic's machinery (it first probes `pg_class` for `alembic_version`, then creates it):

```
INFO  [sqlalchemy.engine.base.Engine] 
CREATE TABLE alembic_version (
	version_num VARCHAR(32) NOT NULL, 
	CONSTRAINT alembic_version_pkc PRIMARY KEY (version_num)
)
```

So 80 `CREATE TABLE`s come from migration files + 1 from Alembic = 81 emitted, but only 77
survive to head. The difference is due to tables that are **created and later renamed/dropped**
across the 255‑revision chain (e.g. `gen_email`→`alias`, `forward_email`→`contact`,
`forward_email_log`→`email_log`; `metric` and `partner` each appear twice in the create stream;
`scope`/`client_scope` are dropped). This is exactly why the count must be read from the live
schema via `information_schema` rather than inferred from `op.create_table(...)` calls.

### (b) The last table created — from migration stdout order

The final `CREATE TABLE` emitted in the run (log line 2819), with its preceding Alembic
"Running upgrade" marker, is:

```
INFO  [alembic.runtime.migration] Running upgrade 91ed7f46dc81 -> 7d7b84779837, user_audit_log
INFO  [sqlalchemy.engine.base.Engine] 
CREATE TABLE user_audit_log (
	id SERIAL NOT NULL, 
	created_at TIMESTAMP WITHOUT TIME ZONE NOT NULL, 
	updated_at TIMESTAMP WITHOUT TIME ZONE, 
	user_id INTEGER NOT NULL, 
	user_email VARCHAR(255) NOT NULL, 
	action VARCHAR(255) NOT NULL, 
	message TEXT, 
	PRIMARY KEY (id)
)
```

The only later revision, `Running upgrade 7d7b84779837 -> 32f25cbf12f6,
alias_audit_log_index_created_at`, creates **no** table — it emits only
`CREATE INDEX CONCURRENTLY ix_alias_audit_log_created_at ON alias_audit_log (created_at)`. The
chain ends at head revision `32f25cbf12f6` (confirmed by `alembic current` → `32f25cbf12f6 (head)`).
`user_audit_log` is present in the final live schema.

> **Answer R1(b):** the last table created (by migration output order) is **`user_audit_log`**
> (`CREATE TABLE user_audit_log`, log line 2819; created by revision `91ed7f46dc81 -> 7d7b84779837`).

### Grounding
- Empty‑DB pattern + `alembic upgrade head`: `scripts/reset_local_db.sh:4,6`; `flask dummy-data` skipped: `scripts/reset_local_db.sh:7`; test invocation pattern `CONFIG=tests/test.env ... alembic upgrade head`: `scripts/run-test.sh:13`.
- Migration wiring: `target_metadata = Base.metadata` (`migrations/env.py:28`); `script_location = migrations` (`alembic.ini:5`); table models on the shared `Base.metadata` (`app/models.py`).
- Count technique: `information_schema.tables` filtered to exclude `pg_catalog`/`information_schema`; `alembic_version` is a real `public` table and is counted — stated explicitly above.

---

## R2 — Web server readiness signal and timing

**Question.** When the web server starts, **(a)** what exact log message indicates it is ready to
accept connections, and **(b)** how many milliseconds elapse between the first log entry and that
ready message?

Both launch paths are exercised: the **development** server (`app.run(debug=True, port=7777)`,
reached by `python server.py` → `local_main()` at `server.py:572`, call at `server.py:588`,
`__main__` at `server.py:598-599`) and the **production** server
(`gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15`, `Dockerfile:47`; `wsgi:app = create_app()`
at `wsgi.py:1-3`).

### Timing baseline (stated explicitly)

The literal first log artifact in the code is `print(">>> init logging <<<")`
(`app/log.py:67`) — a **bare `print`**, carrying **no timestamp**, and (as observed) it is **not**
even the first line on stdout. Therefore it cannot serve as a timestamped baseline. Instead, a
throwaway harness launched each server, read its combined stdout/stderr line‑by‑line, and stamped
**every line** with a monotonic clock at the moment it was read (with `PYTHONUNBUFFERED=1` for
accurate per‑line timing). The **"first log entry"** is defined as the **first line the process
emits** (offset `+0.000 ms`); the delta is `(ready‑line offset) − 0`. The harness also probes TCP
`127.0.0.1:7777` to record the empirical "ready to accept connections" moment. All timings are
**environment‑specific** and were measured at runtime.

### Path 1 — Production (Gunicorn)

```bash
CONFIG=tests/test.env /app/venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
```
Verbatim captured startup (each line prefixed `[UTC wallclock | +offset from first line]`):
```
[2026-07-01 04:27:39.026Z | +     0.000 ms] [2026-07-01 04:27:39 +0000] [2697] [INFO] Starting gunicorn 20.0.4
[2026-07-01 04:27:39.026Z | +     0.208 ms] [2026-07-01 04:27:39 +0000] [2697] [INFO] Listening at: http://0.0.0.0:7777 (2697)
[2026-07-01 04:27:39.026Z | +     0.231 ms] [2026-07-01 04:27:39 +0000] [2697] [INFO] Using worker: sync
[2026-07-01 04:27:39.029Z | +     2.650 ms] [2026-07-01 04:27:39 +0000] [2699] [INFO] Booting worker with pid: 2699
[2026-07-01 04:27:39.049Z | +    23.364 ms] [2026-07-01 04:27:39 +0000] [2700] [INFO] Booting worker with pid: 2700
[2026-07-01 04:27:39.358Z | +   332.350 ms] load config file /app/tests/test.env
### RESULT first_line_offset=0.000 ms
### RESULT ready_line_offset=0.208 ms   (DELTA first->ready-line)
### RESULT port_connectable_offset=0.951 ms   (empirical ready-to-accept-connections)
```

> **Answer R2(a) — Gunicorn:** the ready message is
> **`Listening at: http://0.0.0.0:7777 (2697)`** (full line:
> `[2026-07-01 04:27:39 +0000] [2697] [INFO] Listening at: http://0.0.0.0:7777 (2697)`). This is
> the canonical Gunicorn "ready to accept connections" indicator — the master logs it immediately
> after binding the listening socket, before forking workers.
>
> **Answer R2(b) — Gunicorn:** the first log entry is the master's `Starting gunicorn 20.0.4`
> (`+0.000 ms`); the ready line follows at **`+0.208 ms`** → **delta ≈ 0.208 ms**. A second run
> measured **0.231 ms** (port first connectable at 0.951 ms / 3.506 ms). The delta is sub‑millisecond
> because Gunicorn's arbiter logs "Starting gunicorn" and "Listening at:" back‑to‑back. Note the
> app's own `>>> init logging <<<` does **not** appear first here — Gunicorn does not preload the
> app, so `load config file …` (and later `>>> init logging <<<`) come from the **workers** at
> `~+332 ms`, well after the master's ready line.

### Path 2 — Development server (`app.run(debug=True, port=7777)`)

```bash
CONFIG=tests/test.env /app/venv/bin/python server.py
```
Verbatim captured startup:
```
[2026-07-01 04:26:27.957Z | +     0.000 ms] load config file /app/tests/test.env
[2026-07-01 04:26:27.958Z | +     1.269 ms] >>> URL: http://localhost
[2026-07-01 04:26:28.243Z | +   286.256 ms] >>> init logging <<<
[2026-07-01 04:26:28.310Z | +   353.467 ms] 2026-07-01 04:26:28 - SL - DEBUG - 2638 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
[2026-07-01 04:26:29.475Z | +  1517.914 ms]  * Serving Flask app "server" (lazy loading)
[2026-07-01 04:26:29.475Z | +  1517.980 ms]  * Environment: production
[2026-07-01 04:26:29.475Z | +  1517.993 ms]    WARNING: This is a development server. Do not use it in a production deployment.
[2026-07-01 04:26:29.475Z | +  1518.003 ms]    Use a production WSGI server instead.
[2026-07-01 04:26:29.475Z | +  1518.020 ms]  * Debug mode: on
[2026-07-01 04:26:29.924Z | +  1967.657 ms] load config file /app/tests/test.env      <- reloader restarts child
### RESULT first_line_offset=0.000 ms
### RESULT ready_line: NOT observed in output
### RESULT port_connectable_offset=1520.573 ms   (empirical ready-to-accept-connections)
```

> **Answer R2(a) — dev server:** the expected Werkzeug readiness banner
> **`* Running on http://127.0.0.1:7777/ (Press CTRL+C to quit)` is NOT emitted** on this path.
> This was confirmed across two runs (`ready_line: NOT observed`). **Root cause (verified):**
> Werkzeug 1.0.1 emits that banner through the **`werkzeug` logger** via its internal `_log()`
> helper (`werkzeug/_internal.py:94`, called from `werkzeug/serving.py`'s `log_startup`), and
> `app/log.py:70-71` sets that logger `disabled = True` (`log = logging.getLogger("werkzeug");
> log.disabled = True`), which suppresses it. The lines that **do** appear are Flask's CLI banner,
> printed via `click.echo` (not the logger): `* Serving Flask app "server" (lazy loading)`,
> `* Environment: production`, the development‑server `WARNING`, and `* Debug mode: on`. None of
> these literally states "ready to accept connections."
>
> **Answer R2(b) — dev server:** because no readiness **log line** is emitted, readiness was
> measured empirically: TCP port 7777 first accepted a connection at **`+1520.573 ms`** after the
> first log entry (`load config file …`); a second run measured **1446.200 ms**. The last banner
> line (`* Debug mode: on`) appeared at `+1518.020 ms` / `+1897.581 ms`. This path is slower than
> Gunicorn because `debug=True` enables the Werkzeug **reloader**, which double‑initializes the app
> (the parent binds the socket via fd‑inheritance ~1520 ms, then execs a child that re‑imports and
> serves). This is faithful to `server.py:588`.

### Grounding
- Launch paths: `server.py:572` (`local_main`), `server.py:588` (`app.run(debug=True, port=7777)`), `server.py:598-599` (`__main__` → `local_main()`); `Dockerfile:47` (Gunicorn command); `wsgi.py:1-3` (`app = create_app()`).
- Logging facts: UTC timestamps via `time.gmtime` (`app/log.py:43`); fixed format string (`app/log.py:12-15`); first `print(">>> init logging <<<")` (`app/log.py:67`); `werkzeug` logger disabled (`app/log.py:70-71`).
- Werkzeug banner mechanism (installed 1.0.1): emitted via the `werkzeug` logger in `werkzeug/_internal.py:94` (`_log`) — hence suppressed by the disabled logger.

---


## R3 — Email handler on a custom port (25025)

**Question.** When the email handler is started on port **25025**, **(a)** does the startup log
confirm it is listening on that port, and **(b)** what is the exact message shown?

### Command executed

```bash
CONFIG=tests/test.env /app/venv/bin/python email_handler.py --port 25025
```
The handler parses `-p/--port` with default `20381` (`email_handler.py:2398-2399`); `--port 25025`
overrides it.

### Verbatim captured output

```
[2026-07-01 04:28:49.353Z | +   980.584 ms] 2026-07-01 04:28:49 - SL - INFO - 2816 - "/app/email_handler.py:2403" - <module>() -  - Listen for port 25025
[2026-07-01 04:28:49.355Z | +   982.433 ms] 2026-07-01 04:28:49 - SL - DEBUG - 2816 - "/app/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 25025
### RESULT port_connectable_offset=985.153 ms   (empirical ready-to-accept-connections)
```
TCP port 25025 became connectable immediately afterward, confirming the aiosmtpd `Controller`
actually bound to `0.0.0.0:25025`.

> **Answer R3(a): YES** — the startup log confirms it is listening on port 25025. The literal
> `25025` appears in **both** startup lines above (and the port then accepted connections).
>
> **Answer R3(b):** two lines confirm it (shown within the standard `app/log.py` format —
> `asctime - name - levelname - process - "pathname:lineno" - funcName() - message_id - message`):
> - INFO (from `__main__`, `email_handler.py:2403`): **`Listen for port 25025`**
>   → `2026-07-01 04:28:49 - SL - INFO - 2816 - "/app/email_handler.py:2403" - <module>() -  - Listen for port 25025`
> - DEBUG (from `main()` after `controller.start()`, `email_handler.py:2386`):
>   **`Start mail controller 0.0.0.0 25025`**
>   → `2026-07-01 04:28:49 - SL - DEBUG - 2816 - "/app/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 25025`

**Rationale.** `Listen for port 25025` is `LOG.i("Listen for port %s", args.port)`
(`email_handler.py:2403`), logged from the `__main__` block **before** binding. `Start mail
controller 0.0.0.0 25025` is `LOG.d("Start mail controller %s %s", controller.hostname,
controller.port)` (`email_handler.py:2386`), logged **after** `controller.start()`
(`email_handler.py:2385`), where the controller is
`Controller(MailHandler(), hostname="0.0.0.0", port=port)` (`email_handler.py:2383`). Together
they confirm the handler both intended and actually started on `0.0.0.0:25025`.

### Grounding
`email_handler.py:2398-2399` (`-p/--port`, default `20381`), `:2403` (`Listen for port`), `:2381`
(`def main(port)`), `:2383` (`Controller(... port=port)`), `:2385` (`controller.start()`), `:2386`
(`Start mail controller`).

---

## R4 — Registration, then login *before* activation

**Question.** After creating user `testuser@example.com` / `testpass123`, then attempting to log
in *before* activating the account: **(a)** the exact JSON error, **(b)** the HTTP status code,
**(c)** the full curl command + complete output, and **(d)** the DB values of the `activated` and
`notification` columns for that user.

The web server was running under Gunicorn on `:7777`.

### (c) Full curl commands and complete output

**Step 1 — register** (`POST /api/auth/register`, `app/api/views/auth.py:87`):
```bash
curl -sS -i -X POST http://localhost:7777/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"testuser@example.com","password":"testpass123"}'
```
```
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Wed, 01 Jul 2026 04:30:10 GMT
Connection: close
Content-Type: application/json
Content-Length: 46
Access-Control-Allow-Origin: *
Set-Cookie: slapp=e582d7e4-100e-4fe5-b18a-0741fbe9fef8...; HttpOnly; Path=/; SameSite=Lax

{"msg":"User needs to confirm their account"}
```
`example.com` is accepted by `email_can_be_used_as_mailbox()`; the endpoint creates the user via
`User.create(email=email, name=dirty_email, password=password)` (`app/api/views/auth.py:125`,
preceded by `LOG.d("create user %s", email)` at `:124`) **without** setting `activated` (so it
defaults to `False`), issues a 6‑digit activation code, and
returns `msg="User needs to confirm their account"`. The account is **not** activated.

**Step 2 — login before activation** (`POST /api/auth/login`, `app/api/views/auth.py:29`):
```bash
curl -sS -i -X POST http://localhost:7777/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"testuser@example.com","password":"testpass123"}'
```
```
HTTP/1.1 422 UNPROCESSABLE ENTITY
Server: gunicorn/20.0.4
Date: Wed, 01 Jul 2026 04:30:21 GMT
Connection: close
Content-Type: application/json
Content-Length: 34
Access-Control-Allow-Origin: *
Set-Cookie: slapp=563beeb2-c0e1-4f5c-a16e-f1b75dd187d1...; HttpOnly; Path=/; SameSite=Lax

{"error":"Account not activated"}
```
Status confirmed independently with `curl -w`:
```bash
curl -sS -o /dev/null -w "HTTP_STATUS=%{http_code}\n" -X POST http://localhost:7777/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"testuser@example.com","password":"testpass123"}'
```
```
HTTP_STATUS=422
```

> **Answer R4(a):** the exact JSON error is **`{"error":"Account not activated"}`**.
>
> **Answer R4(b):** the HTTP status code is **`422`** (`422 UNPROCESSABLE ENTITY`) — demonstrated
> in both the `-i` headers and the `-w "%{http_code}"` probe (not assumed).
>
> **Answer R4(c):** the full curl commands and their complete output (request, response headers,
> status line, and body) are shown verbatim above.

### (d) Direct database query

```bash
psql -h localhost -p 15432 -U test -d test \
  -c "SELECT email, activated, notification FROM users WHERE email='testuser@example.com';"
```
```
        email         | activated | notification 
----------------------+-----------+--------------
 testuser@example.com | f         | t
(1 row)
```

> **Answer R4(d):** for `testuser@example.com`, **`activated = f`** (false) and
> **`notification = t`** (true).

**Rationale.** The `422` comes specifically from the not‑activated guard
`elif not user.activated:` → `return jsonify(error="Account not activated"), 422`
(`app/api/views/auth.py:75-77`) — distinct from the `400` branches above it (`Email or password
incorrect` at `:58`/`:66`, `Account disabled` at `:69`, `Account scheduled for deletion` at
`:74`) and the `403` FIDO branch at `:81`. The DB values reflect the **column defaults** because
registration never activates the account and never toggles notification:
`activated = sa.Column(sa.Boolean, default=False, …)` (`app/models.py:358`) → `f`;
`notification = sa.Column(sa.Boolean, default=True, …, server_default="1")`
(`app/models.py:354-356`) → `t`.

### Grounding
`app/api/views/auth.py:87` (register route), `:125` (`User.create(...)` without `activated`),
`:29` (login route), `:75-77` (not‑activated guard returning `422`); `app/models.py:358`
(`activated` default `False`), `:354-356` (`notification` default `True`).

---


## R5 — Dynamic alias‑limit behavior

**Question.** **(a)** Create a fresh user and call `/api/user_info` to observe
`max_alias_free_plan`; **(b)** change the configuration to set the limit to **10**, restart the
server, create a *second* new user, and call `/api/user_info` again; **(c)** capture the exact API
responses before and after; **(d)** do *both* users reflect the new limit, or only the user
created after the change?

`GET /api/user_info` (`app/api/views/user_info.py:50`, `@require_api_auth`) returns
`user_to_dict(user)`, whose `"max_alias_free_plan": user.max_alias_for_free_account()` field is at
`app/api/views/user_info.py:34`. For a standard free account,
`max_alias_for_free_account()` returns `config.MAX_NB_EMAIL_FREE_PLAN` (`app/models.py:858,865`),
which is read from the environment with default `5` (`app/config.py:120-124`). The API key is
supplied in the **`Authentication`** header (`app/api/base.py:17-18`). Users were created
(activated, with an API key) using a throwaway helper mirroring `tests/utils.py`'s
`create_new_user` (`User.create(..., activated=True, flush=True)` + `ApiKey.create`).

### (a) / (c) BEFORE — default limit

Server running with `CONFIG=/tmp/inv/base.env` (a throwaway copy of `tests/test.env` with the
`MAX_NB_EMAIL_FREE_PLAN` line removed, so the code default `5` applies). Creating USER‑A
(`usera@example.com`, id 3) even printed the default‑path message
`MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value` (`app/config.py:123`).

```bash
curl -sS -i http://localhost:7777/api/user_info -H 'Authentication: <USER-A key>'
```
```
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Content-Type: application/json
Content-Length: 194
Access-Control-Allow-Origin: *

{"can_create_reverse_alias":true,"connected_proton_address":null,"email":"usera@example.com","in_trial":true,"is_premium":true,"max_alias_free_plan":5,"name":"usera","profile_picture_url":null}
```

> **Answer R5(a):** before the change, USER‑A's `max_alias_free_plan` is **`5`** (the default,
> `app/config.py:124`; also documented as `"max_alias_free_plan": 5,` in `docs/api.md:213`).

### (b) Change the limit to 10 and restart

A throwaway `after.env` (copy of `tests/test.env` with `MAX_NB_EMAIL_FREE_PLAN=10`) was selected
via `CONFIG` and the server **restarted** (mandatory, because the value is read at **module import
time**, `app/config.py:120-124`). The restart log confirms the new config was loaded:
```
[2026-07-01 04:32:13 +0000] [3025] [INFO] Starting gunicorn 20.0.4
[2026-07-01 04:32:13 +0000] [3025] [INFO] Listening at: http://0.0.0.0:7777 (3025)
load config file /tmp/inv/after.env
```
Then USER‑B (`userb@example.com`, id 4) was created.

### (c) AFTER — both users queried

USER‑A (the **pre‑change** user):
```bash
curl -sS -i http://localhost:7777/api/user_info -H 'Authentication: <USER-A key>'
```
```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 195

{"can_create_reverse_alias":true,"connected_proton_address":null,"email":"usera@example.com","in_trial":true,"is_premium":true,"max_alias_free_plan":10,"name":"usera","profile_picture_url":null}
```

USER‑B (the **post‑change** user):
```bash
curl -sS -i http://localhost:7777/api/user_info -H 'Authentication: <USER-B key>'
```
```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 195

{"can_create_reverse_alias":true,"connected_proton_address":null,"email":"userb@example.com","in_trial":true,"is_premium":true,"max_alias_free_plan":10,"name":"userb","profile_picture_url":null}
```

> **Answer R5(c):** before → USER‑A reports `"max_alias_free_plan":5`; after the change+restart →
> USER‑A reports `"max_alias_free_plan":10` **and** USER‑B reports `"max_alias_free_plan":10`.

### (d) Do both users reflect the new limit?

> **Answer R5(d): YES — BOTH users reflect the new limit of `10`.** USER‑A was created while the
> limit was `5` and returned `5` beforehand, yet after changing the config to `10` and restarting,
> USER‑A returns `10` — the same as the freshly created USER‑B. It is **not** limited to only the
> user created after the change.

**Rationale.** `user_info` returns `user.max_alias_for_free_account()`
(`app/api/views/user_info.py:34`), which returns `config.MAX_NB_EMAIL_FREE_PLAN`
(`app/models.py:865`) — read **live from module‑level config at request time**, **not** persisted
on the user row. Consequently every request reflects whatever value the running process currently
holds. Because `MAX_NB_EMAIL_FREE_PLAN` is read once at **module import** (`app/config.py:120-124`),
a **restart** is required for a new value to take effect; the value is selected without editing any
committed file via the `CONFIG` env var → `load_dotenv(get_abs_path(config_file))`
(`app/config.py:65-71`). The config‑driven (non‑persisted) nature is corroborated by
`tests/test.env:13` (`MAX_NB_EMAIL_FREE_PLAN=3`) and `example.env:55`.

### Grounding
`app/api/views/user_info.py:34` (`max_alias_free_plan` field), `:50` (`/user_info` route +
`@require_api_auth`); `app/models.py:858,865` (`max_alias_for_free_account` → `config.MAX_NB_EMAIL_FREE_PLAN`);
`app/config.py:120-124` (env read, default `5`), `:65-71` (`CONFIG`‑driven `.env` loading);
`app/api/base.py:17-18` (API key via `Authentication` header); `example.env:55`, `tests/test.env:13`,
`docs/api.md:213`.

---

## Coverage pass

Every sub‑part of every question is answered above, each backed by a verbatim runtime‑output block.

| Requirement | Sub‑part | Answer (exact literal) | Backed by |
|-------------|----------|------------------------|-----------|
| **R1** | (a) total tables | **77** (incl. `alembic_version`; 76 application tables) | live `information_schema` count block |
| **R1** | (b) last table created | **`user_audit_log`** | final `CREATE TABLE` in migration stdout (log line 2819) |
| **R2** | (a) ready message — Gunicorn | **`Listening at: http://0.0.0.0:7777 (2697)`** | Gunicorn capture block |
| **R2** | (b) ms delta — Gunicorn | **≈ 0.208 ms** (run 2: 0.231 ms) | timestamped Gunicorn capture |
| **R2** | (a) ready message — dev server | **No `Running on` line is emitted** (Werkzeug logger disabled, `app/log.py:70-71`) | dev‑server capture block (2 runs) |
| **R2** | (b) ms delta — dev server | no ready log line; empirical port‑connectable **≈ 1520.573 ms** (run 2: 1446.200 ms) | dev‑server capture + port probe |
| **R3** | (a) confirms port 25025? | **YES** | both startup lines contain `25025`; port connectable |
| **R3** | (b) exact message(s) | **`Listen for port 25025`** and **`Start mail controller 0.0.0.0 25025`** | email‑handler capture block |
| **R4** | (a) JSON error | **`{"error":"Account not activated"}`** | login `curl -i` body |
| **R4** | (b) HTTP status | **`422`** (`422 UNPROCESSABLE ENTITY`) | `curl -i` status line + `curl -w` probe |
| **R4** | (c) full curl command + output | shown verbatim (register + login) | R4(c) blocks |
| **R4** | (d) `activated` / `notification` | **`activated = f`**, **`notification = t`** | `psql` result row |
| **R5** | (a) before value | **`5`** | USER‑A "before" JSON |
| **R5** | (b) change to 10 + restart + 2nd user | done (`after.env`, restart, USER‑B) | restart log + USER‑B creation |
| **R5** | (c) exact responses before/after | before `5`; after USER‑A `10`, USER‑B `10` | before/after JSON blocks |
| **R5** | (d) both users or only new? | **BOTH** reflect `10` | USER‑A + USER‑B "after" JSON |

**Items measured empirically rather than read from a log line:** R2(b) for the **dev server**,
because the Werkzeug readiness banner is genuinely suppressed by the disabled `werkzeug` logger
(`app/log.py:70-71`); the "ready to accept connections" instant was therefore measured by TCP port
connectivity and is stated as such. All other answers are direct log/HTTP/SQL observations.

**Environment honesty.** All timings are specific to the `sl-setup` container run and were measured
at runtime; they are not generalized. Every other value (counts, table name, ready messages, JSON
bodies, status code, `f`/`t`, `5`/`10`) is an exact literal reproduced from captured output.

