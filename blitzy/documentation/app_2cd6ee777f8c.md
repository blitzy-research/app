# SimpleLogin — First‑Run / Initialization Investigation (branch `app_2cd6ee777f8c`)

This document answers five first‑run / initialization questions about the SimpleLogin
email‑aliasing application. Every reported value was produced by **building and running the
relevant code path first**, capturing the **complete, unedited output**, and only then writing
the answer. Each question below leads with the **direct answer**, followed by the **exact
command(s)**, the **complete unedited output**, the **`file:line` grounding** (naming the
specific function/method that performs the work), and the **rationale**.

> **Read‑only source repository.** No file in the SimpleLogin source working copy was modified,
> added, or deleted. All experimentation used ephemeral configuration and throwaway state that
> lives **outside** the repository (inside the runtime container at `/root/…`) and was removed
> afterward. The working copy is left byte‑for‑byte unchanged.

---

## Environment & build

**Working copy (source of truth, read-only):** `/tmp/blitzy/app/app_2cd6ee777f8c_36621f`
- git branch: `app_2cd6ee777f8c` (this document's filename equals the branch name)
- HEAD commit: `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`
- `git status --porcelain` at start **and** end: empty (0 lines — clean).

**Runtime stack — the canonical versions the project pins.** The five answers were produced on the
**exact service versions the repository declares as canonical** in its CI matrix
(`.github/workflows/main.yml`): **PostgreSQL 13** (`:47` `image: postgres:13`), **Redis 6**
(`:94` `redis-version: 6`), and **Python 3.10** (`:17`/`:40` `python-version: '3.10'`). These were
stood up with the official `postgres:13` and `redis:6` Docker images on a private bridge network,
and the application ran from the project's own pinned virtualenv (built from `poetry.lock`) inside
the shipped app image. The read-only working copy was bind-mounted at `/host_repo`:

```
docker network create sl-canon-net

# Canonical PostgreSQL 13 (matches .github/workflows/main.yml:47 image: postgres:13,
# with the same test/test/test credentials from :52/:54/:58)
docker run -d --name sl-pg13 --network sl-canon-net \
  -e POSTGRES_USER=test -e POSTGRES_PASSWORD=test -e POSTGRES_DB=test \
  postgres:13 -c fsync=off -c full_page_writes=off

# Canonical Redis 6 (matches .github/workflows/main.yml:94 redis-version: 6)
docker run -d --name sl-redis6 --network sl-canon-net redis:6

# Application container (project's pinned Python 3.10 venv from poetry.lock),
# source working copy bind-mounted READ-ONLY at /host_repo
docker run -d --name sl-canon --network sl-canon-net \
  -v /tmp/blitzy/app/app_2cd6ee777f8c_36621f:/host_repo:ro \
  --entrypoint sleep simple-login-app-fixed:app_2cd6ee777f8c infinity
```

**Observed versions (verbatim from the running canonical environment):**

| Component | Observed version | Canonical reference |
|-----------|------------------|---------------------|
| Python | **3.10.18** | `Dockerfile:8` `FROM python:3.10`; `pyproject.toml` `python = "^3.10"`; `.github/workflows/main.yml:17,:40` `python-version: '3.10'` |
| PostgreSQL | **13.23** (Debian 13.23-1.pgdg13+1), image `postgres:13` | `.github/workflows/main.yml:47` `image: postgres:13` |
| Redis | **6.2.22**, image `redis:6` | `.github/workflows/main.yml:94` `redis-version: 6` |
| gunicorn | **20.0.4** | `poetry.lock:1447-1448` `name = "gunicorn"` / `version = "20.0.4"` |
| Flask / Werkzeug | **1.1.2 / 1.0.1** | `poetry.lock` |
| SQLAlchemy / Alembic / Flask-Migrate | **1.3.24 / 1.4.3 / 2.5.3** | `poetry.lock` |
| aiosmtpd | **1.4.2** | `poetry.lock` |
| psycopg2-binary | **2.9.3** | `poetry.lock` |
| Flask-Limiter / redis-py | **1.4 / 4.6.0** | `poetry.lock` |
| dnspython / bcrypt | **2.0.0 / 3.2.0** | `poetry.lock` |

The exact version strings were read from the running services:

```
$ docker exec sl-canon bash -lc 'source /app/venv/bin/activate && python --version'
Python 3.10.18
$ docker exec sl-pg13 postgres --version
postgres (PostgreSQL) 13.23 (Debian 13.23-1.pgdg13+1)
$ docker exec sl-redis6 redis-server --version
Redis server v=6.2.22 sha=00000000:0 malloc=jemalloc-5.1.0 bits=64 build=bec41ce62cfcb8e
```

> **Note on the plan's Docker assumption.** The Agent Action Plan speculated that Docker might be
> unusable in the investigation shell and that an apt-installed PostgreSQL would be needed. In
> practice Docker **was** available, so the strictly canonical `postgres:13` and `redis:6` images
> were used directly — matching the CI-declared versions exactly rather than substituting a
> different major version. Every value below therefore reflects the default, canonical stack.

**Ephemeral configuration (outside the repository; removed at cleanup).** `app/config.py` loads its
`.env` from the path in the `CONFIG` environment variable (`config.py:65` `config_file =
os.environ.get("CONFIG")`, then `:69` `load_dotenv(get_abs_path(config_file))`) and reads many
variables at import time. Two ephemeral env files were derived from the repository's own
`tests/test.env` template and stored at `/root` inside the container (never in the repo):

- `/root/run_base.env` — `MAX_NB_EMAIL_FREE_PLAN` **removed** (so the code default of 5 applies),
  `DISABLE_RATE_LIMIT=1`, `NOT_SEND_EMAIL=true`, `DB_URI=postgresql://test:test@sl-pg13:5432/test`,
  `MEM_STORE_URI=redis://sl-redis6`. Used for Q1-Q4 and the Q5 "before" measurement.
- `/root/run_after.env` — identical **plus** `MAX_NB_EMAIL_FREE_PLAN=10`. Used for the Q5 "after"
  measurement.

The **same** `DB_URI` (`postgresql://test:test@sl-pg13:5432/test`) was used for the migration run,
the API server, and every direct `psql` query, so migrated tables and registered users are
consistently observable. (This is the canonical `postgres:13` container's own port `5432` on the
private network; the CI workflow publishes the identical service on host port `15432` via
`.github/workflows/main.yml:60` `ports: - 15432:5432`.) `PYTHONDONTWRITEBYTECODE=1` was set for all
runtime commands to avoid writing `.pyc` files into the working copy.

**Canonical invocation pattern used throughout:**

```
docker exec sl-canon bash -lc 'source /app/venv/bin/activate && cd /host_repo \
  && export PYTHONDONTWRITEBYTECODE=1 && export CONFIG=/root/run_base.env && <command>'
```

---

## Q1 — Database migrations on an empty PostgreSQL database

### Direct answer
- **(a) Total number of tables created: `77`.** (76 ORM/model tables + Alembic's own
  `alembic_version` bookkeeping table.)
- **(b) The last table created, by migration execution order, is `user_audit_log`.**

Both values were observed on the **canonical stack — PostgreSQL 13.23** (`docker run postgres:13`,
matching `.github/workflows/main.yml:47`) — and are stable across two independent reset -> migrate
runs. The **complete, unedited** logs of **both** runs are embedded verbatim in **Appendix A**.

### Exact commands
Mirroring the repository's own reset pattern (`scripts/reset_test_db.sh`: `drop schema public
cascade; create schema public;` then `alembic upgrade head`). Executed in the app container against
canonical PostgreSQL 13 with `CONFIG=/root/run_base.env`, `cwd=/host_repo`,
`DB=postgresql://test:test@sl-pg13:5432/test`:

```
echo "drop schema public cascade; create schema public;" | psql "$DB"
alembic upgrade head
alembic current
psql "$DB" -c "SELECT count(*) FROM information_schema.tables WHERE table_schema='public';"
psql "$DB" -c "SELECT table_name FROM information_schema.tables WHERE table_schema='public'
               AND table_name IN ('user_audit_log','alias_audit_log','alembic_version')
               ORDER BY table_name;"
```

The **complete unedited output** of this exact sequence, for **both** runs, is in **Appendix A**
(Run 1 = fresh/empty start; Run 2 = reset of the now-populated DB, whose `drop schema … cascade`
NOTICE enumerates all 82 other objects — 77 tables + 4 enum types + 1 `pg_trgm` extension). The `alembic upgrade head` step emits the full **255**
`Running upgrade` lines in each run — none are elided. The tail of Run 1 (the last two migration
steps, `alembic current`, and both `information_schema` queries) is:

```
INFO  [alembic.runtime.migration] Running upgrade 62afa3a10010 -> 91ed7f46dc81, alias_audit_log
INFO  [alembic.runtime.migration] Running upgrade 91ed7f46dc81 -> 7d7b84779837, user_audit_log
INFO  [alembic.runtime.migration] Running upgrade 7d7b84779837 -> 32f25cbf12f6, alias_audit_log_index_created_at
----- step 3: alembic current -----
load config file /root/run_base.env
>>> URL: http://localhost
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
WARNING: Use a temp directory for GNUPGHOME /tmp/evymtuqdqpbvmpwojzwu
Upload files to local dir
>>> init logging <<<
2026-07-08 06:07:44,567 - SL - DEBUG - 45 - "/host_repo/app/utils.py:17" - <module>() -  - load words file: /host_repo/local_data/test_words.txt
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
32f25cbf12f6 (head)
----- step 4: table count (information_schema) -----
 count 
-------
    77
(1 row)

----- step 5: confirm last-created + bookkeeping tables present -----
   table_name    
-----------------
 alembic_version
 alias_audit_log
 user_audit_log
(3 rows)
```

### Proof that `user_audit_log` is the last `CREATE TABLE` (exact SQL-echo command + config)
To pinpoint the last `CREATE TABLE` deterministically, the upgrade was re-run once with SQLAlchemy
engine SQL echo enabled. Echo is turned on by an **ephemeral** Alembic config stored **outside** the
repo at `/root/alembic_echo.ini` — byte-identical to the repo's `alembic.ini` except two lines:
`script_location = /host_repo/migrations` (absolute, so it runs from any cwd) and, under
`[logger_sqlalchemy]` (whose `qualname = sqlalchemy.engine`), `level = INFO` instead of `WARN`.
The exact command was:

```
echo "drop schema public cascade; create schema public;" | psql "$DB"
alembic -c /root/alembic_echo.ini upgrade head      # sqlalchemy.engine=INFO -> echoes every SQL statement
```

That echo run emitted exactly **81** `CREATE TABLE` statements (the total captured log-line volume is environment-dependent — it varies by config preamble and capture method — and is not itself a Q1 answer value). The
**complete ordered list** of every `CREATE TABLE` emitted (execution order, via
`grep -oE 'CREATE TABLE [a-z_]+'` over the echo log) is:

```
 1  CREATE TABLE alembic_version
 2  CREATE TABLE file
 3  CREATE TABLE scope
 4  CREATE TABLE users
 5  CREATE TABLE activation_code
 6  CREATE TABLE client
 7  CREATE TABLE gen_email
 8  CREATE TABLE authorization_code
 9  CREATE TABLE client_scope
10  CREATE TABLE client_user
11  CREATE TABLE oauth_token
12  CREATE TABLE redirect_uri
13  CREATE TABLE reset_password_code
14  CREATE TABLE partner
15  CREATE TABLE forward_email
16  CREATE TABLE subscription
17  CREATE TABLE forward_email_log
18  CREATE TABLE deleted_alias
19  CREATE TABLE email_change
20  CREATE TABLE api_key
21  CREATE TABLE alias_used_on
22  CREATE TABLE custom_domain
23  CREATE TABLE lifetime_coupon
24  CREATE TABLE directory
25  CREATE TABLE job
26  CREATE TABLE mailbox
27  CREATE TABLE manual_subscription
28  CREATE TABLE social_auth
29  CREATE TABLE account_activation
30  CREATE TABLE refused_email
31  CREATE TABLE referral
32  CREATE TABLE apple_subscription
33  CREATE TABLE sent_alert
34  CREATE TABLE alias_mailbox
35  CREATE TABLE recovery_code
36  CREATE TABLE domain_deleted_alias
37  CREATE TABLE notification
38  CREATE TABLE fido
39  CREATE TABLE mfa_browser
40  CREATE TABLE directory_mailbox
41  CREATE TABLE public_domain
42  CREATE TABLE domain_mailbox
43  CREATE TABLE monitoring
44  CREATE TABLE batch_import
45  CREATE TABLE authorized_address
46  CREATE TABLE coinbase_subscription
47  CREATE TABLE metric
48  CREATE TABLE bounce
49  CREATE TABLE transactional_email
50  CREATE TABLE metric
51  CREATE TABLE payout
52  CREATE TABLE hibp
53  CREATE TABLE alias_hibp
54  CREATE TABLE ignored_email
55  CREATE TABLE coupon
56  CREATE TABLE hibp_notified_alias
57  CREATE TABLE ignore_bounce_sender
58  CREATE TABLE auto_create_rule
59  CREATE TABLE auto_create_rule__mailbox
60  CREATE TABLE message_id_matching
61  CREATE TABLE deleted_directory
62  CREATE TABLE deleted_subdomain
63  CREATE TABLE phone_country
64  CREATE TABLE phone_number
65  CREATE TABLE phone_message
66  CREATE TABLE phone_reservation
67  CREATE TABLE invalid_mailbox_domain
68  CREATE TABLE admin_audit_log
69  CREATE TABLE provider_complaint
70  CREATE TABLE partner
71  CREATE TABLE partner_api_token
72  CREATE TABLE partner_user
73  CREATE TABLE partner_subscription
74  CREATE TABLE newsletter
75  CREATE TABLE newsletter_user
76  CREATE TABLE api_cookie_token
77  CREATE TABLE daily_metric
78  CREATE TABLE sync_event
79  CREATE TABLE mailbox_activation
80  CREATE TABLE alias_audit_log
81  CREATE TABLE user_audit_log
```

The **last** `CREATE TABLE` emitted is **`user_audit_log`** (#81). There are 81 `CREATE TABLE`
statements but only 77 live tables because several early tables were later renamed or dropped by
subsequent migrations (e.g. `gen_email` -> `alias`, `forward_email` -> `contact`, and the duplicate
`partner`/`metric` creations), so the authoritative **live** count is the `information_schema`
value (77), not the count of historical `create_table` calls. The complete DDL for
`user_audit_log` and **everything emitted after it** — only its three `CREATE INDEX` statements,
then the HEAD revision `32f25cbf12f6` (a `CREATE INDEX CONCURRENTLY`, i.e. **index-only**), then
the final `alembic_version` bump — proving that no later `CREATE TABLE` exists:

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


INFO  [sqlalchemy.engine.base.Engine] {}
INFO  [sqlalchemy.engine.base.Engine] CREATE INDEX ix_user_audit_log_user_email ON user_audit_log (user_email)
INFO  [sqlalchemy.engine.base.Engine] {}
INFO  [sqlalchemy.engine.base.Engine] CREATE INDEX ix_user_audit_log_user_id ON user_audit_log (user_id)
INFO  [sqlalchemy.engine.base.Engine] {}
INFO  [sqlalchemy.engine.base.Engine] CREATE INDEX ix_user_audit_log_created_at ON user_audit_log (created_at)
INFO  [sqlalchemy.engine.base.Engine] {}
INFO  [sqlalchemy.engine.base.Engine] UPDATE alembic_version SET version_num='7d7b84779837' WHERE alembic_version.version_num = '91ed7f46dc81'
INFO  [sqlalchemy.engine.base.Engine] {}
INFO  [alembic.runtime.migration] Running upgrade 7d7b84779837 -> 32f25cbf12f6, alias_audit_log_index_created_at
INFO  [sqlalchemy.engine.base.Engine] COMMIT
INFO  [sqlalchemy.engine.base.Engine] CREATE INDEX CONCURRENTLY ix_alias_audit_log_created_at ON alias_audit_log (created_at)
INFO  [sqlalchemy.engine.base.Engine] {}
INFO  [sqlalchemy.engine.base.Engine] COMMIT
INFO  [sqlalchemy.engine.base.Engine] BEGIN (implicit)
INFO  [sqlalchemy.engine.base.Engine] UPDATE alembic_version SET version_num='32f25cbf12f6' WHERE alembic_version.version_num = '7d7b84779837'
INFO  [sqlalchemy.engine.base.Engine] {}
INFO  [sqlalchemy.engine.base.Engine] COMMIT
```

### Stability (run twice)
The full reset -> `alembic upgrade head` -> count sequence was executed **twice** against canonical
PostgreSQL 13. Both runs ended at `alembic current` = `32f25cbf12f6 (head)`, both ran **255**
`Running upgrade` steps, and both reported `count = 77`, with the same final two `create_table`
steps (`alias_audit_log` then `user_audit_log`):

```
run1 (empty start):     count=77   head=32f25cbf12f6 (head)   upgrade_steps=255
run2 (populated reset): count=77   head=32f25cbf12f6 (head)   upgrade_steps=255   (drop-cascade: 82 objects)
```

The count is observed via `information_schema`, not inferred. Complete logs: **Appendix A**.

### `file:line` grounding
- `alembic.ini:5` — `script_location = migrations` (migration root).
- `migrations/env.py:27` — `from app.config import DB_URI`; `:28` — `target_metadata =
  Base.metadata`; `:35` — `config.set_main_option('sqlalchemy.url', DB_URI)`; `:69-73` —
  `engine_from_config(..., poolclass=pool.NullPool)` (NullPool on `:72`); `:80-81` —
  `with context.begin_transaction(): context.run_migrations()` (online mode, inside
  `run_migrations_online()` defined at `:62`); `:84-87` — the offline/online dispatch that calls
  `run_migrations_online()`.
- HEAD revision `32f25cbf12f6` — file
  `migrations/versions/2024_101616_32f25cbf12f6_alias_audit_log_index_created_at.py`: `:14`
  `revision = '32f25cbf12f6'`, `:15` `down_revision = '7d7b84779837'`, `:22`
  `op.create_index('ix_alias_audit_log_created_at', 'alias_audit_log', ['created_at'], ...)` —
  **index only, no `create_table`**. It is the true HEAD (no revision has
  `down_revision='32f25cbf12f6'`).
- Revision `7d7b84779837` — file `migrations/versions/2024_101611_7d7b84779837_user_audit_log.py`:
  `:15` `down_revision = '91ed7f46dc81'`, `:22` `op.create_table('user_audit_log', ...)` — the
  **last `create_table`** in execution order.
- Revision `91ed7f46dc81` — file `migrations/versions/2024_101113_91ed7f46dc81_alias_audit_log.py`:
  `:22` `op.create_table('alias_audit_log', ...)` — the immediately preceding table create.
- `app/models.py` — declares **76** ORM tables (`grep -c __tablename__ app/models.py` -> 76); the
  77th live table is Alembic's own `alembic_version`.

### Rationale
"Last table created" is an **execution-order** question, not a declaration-order one. Alembic
applies migrations along the linear revision chain; the true HEAD `32f25cbf12f6` only adds an index
(`CREATE INDEX CONCURRENTLY`), so the final `CREATE TABLE` executed is `user_audit_log` from
`7d7b84779837`. The total is the 76 model tables plus Alembic's own `alembic_version` table = **77**,
confirmed by querying `information_schema.tables` rather than by counting `create_table` statements
(the echo run shows 81 historical `CREATE TABLE` emissions, several of whose tables were later
dropped/renamed — so the live count is authoritative). Flask-Limiter uses Redis (`MEM_STORE_URI`),
not a database table, so it adds nothing to the count.

## Q2 — Web server startup readiness

### Direct answer
- **(a) The log message that signals readiness to accept connections is
  `Listening at: http://0.0.0.0:7777 (<pid>)`** (INFO, emitted by the gunicorn master/arbiter).
  The very first log line is `Starting gunicorn 20.0.4`.
- **(b) The elapsed time between the first log entry and the ready message is ~`0.20 ms`**
  (sub-millisecond), stable across four runs (measured range **0.187 – 0.231 ms**).

### Exact command
The canonical production entry point is gunicorn (`Dockerfile:47`). Because gunicorn's own log
timestamps are **second**-resolution, the stderr stream was wrapped with an external
high-resolution (millisecond) timestamper — a small Python wrapper using `time.perf_counter()`
that spawns the **exact** canonical command
(`gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15`) and stamps each line with its offset in ms
from the first line. Run in the app container with `CONFIG=/root/run_base.env`, `cwd=/host_repo`:

```
python /root/q2_timeit.py     # prints [wallclock] [+offset ms] <gunicorn log line>
```

### Exact timestamper wrapper (`/root/q2_timeit.py`, ephemeral, outside the repo)

```
#!/usr/bin/env python3
"""High-resolution (millisecond) timestamper for gunicorn startup (Q2).

Gunicorn's own log timestamps are second-resolution, so this wrapper spawns the EXACT canonical
command and stamps every stderr line with a monotonic offset (time.perf_counter) from the first
line. It prints, for each line:  [HH:MM:SS.mmm] [+<offset> ms] <original gunicorn log line>
It then reports the FIRST line, the READY line ("Listening at:"), and their delta in ms, and
terminates gunicorn once the startup sequence has been observed.
"""
import subprocess, sys, time, os, signal

CMD = ["gunicorn", "wsgi:app", "-b", "0.0.0.0:7777", "-w", "2", "--timeout", "15"]

proc = subprocess.Popen(
    CMD, stdout=subprocess.PIPE, stderr=subprocess.STDOUT,
    bufsize=1, universal_newlines=True, preexec_fn=os.setsid,
)

t0 = None
first_line = None
ready_line = None
ready_off = None
first_off = None
lines_after_ready = 0
try:
    for raw in proc.stdout:
        line = raw.rstrip("\n")
        now = time.perf_counter()
        if t0 is None:
            t0 = now
        off_ms = (now - t0) * 1000.0
        wall = time.strftime("%H:%M:%S") + (".%03d" % int((time.time() % 1) * 1000))
        print("[%s] [+%9.3f ms] %s" % (wall, off_ms, line), flush=True)
        if first_line is None:
            first_line = line; first_off = off_ms
        if ready_line is None and "Listening at:" in line:
            ready_line = line; ready_off = off_ms
        if ready_line is not None:
            lines_after_ready += 1
            # capture a couple lines past ready (Using worker / Booting worker) then stop
            if lines_after_ready >= 3:
                break
finally:
    try:
        os.killpg(os.getpgid(proc.pid), signal.SIGTERM)
    except Exception:
        pass
    try:
        proc.wait(timeout=5)
    except Exception:
        try: os.killpg(os.getpgid(proc.pid), signal.SIGKILL)
        except Exception: pass

print("")
print("FIRST LINE : %s" % first_line)
print("READY LINE : %s" % ready_line)
if ready_off is not None and first_off is not None:
    print("DELTA first->ready = %.3f ms" % (ready_off - first_off))
else:
    print("DELTA first->ready = N/A (ready line not observed)")
```

### Complete unedited output — all four runs
Each run's **complete, unedited** timestamped stream is shown below (the wrapper terminates
gunicorn a few lines after the ready line, so each run's full captured output is exactly these four
gunicorn lines plus the trailing FIRST/READY/DELTA summary the wrapper prints):

**Run 1 (complete, unedited):**

```
[06:14:31.466] [+    0.000 ms] [2026-07-08 06:14:31 +0000] [198] [INFO] Starting gunicorn 20.0.4
[06:14:31.466] [+    0.202 ms] [2026-07-08 06:14:31 +0000] [198] [INFO] Listening at: http://0.0.0.0:7777 (198)
[06:14:31.466] [+    0.218 ms] [2026-07-08 06:14:31 +0000] [198] [INFO] Using worker: sync
[06:14:31.468] [+    2.486 ms] [2026-07-08 06:14:31 +0000] [199] [INFO] Booting worker with pid: 199

FIRST LINE : [2026-07-08 06:14:31 +0000] [198] [INFO] Starting gunicorn 20.0.4
READY LINE : [2026-07-08 06:14:31 +0000] [198] [INFO] Listening at: http://0.0.0.0:7777 (198)
DELTA first->ready = 0.202 ms
```

**Run 2 (complete, unedited):**

```
[06:14:34.951] [+    0.000 ms] [2026-07-08 06:14:34 +0000] [225] [INFO] Starting gunicorn 20.0.4
[06:14:34.951] [+    0.187 ms] [2026-07-08 06:14:34 +0000] [225] [INFO] Listening at: http://0.0.0.0:7777 (225)
[06:14:34.951] [+    0.217 ms] [2026-07-08 06:14:34 +0000] [225] [INFO] Using worker: sync
[06:14:34.954] [+    2.371 ms] [2026-07-08 06:14:34 +0000] [226] [INFO] Booting worker with pid: 226

FIRST LINE : [2026-07-08 06:14:34 +0000] [225] [INFO] Starting gunicorn 20.0.4
READY LINE : [2026-07-08 06:14:34 +0000] [225] [INFO] Listening at: http://0.0.0.0:7777 (225)
DELTA first->ready = 0.187 ms
```

**Run 3 (complete, unedited):**

```
[06:14:38.437] [+    0.000 ms] [2026-07-08 06:14:38 +0000] [252] [INFO] Starting gunicorn 20.0.4
[06:14:38.437] [+    0.231 ms] [2026-07-08 06:14:38 +0000] [252] [INFO] Listening at: http://0.0.0.0:7777 (252)
[06:14:38.437] [+    0.248 ms] [2026-07-08 06:14:38 +0000] [252] [INFO] Using worker: sync
[06:14:38.439] [+    2.394 ms] [2026-07-08 06:14:38 +0000] [253] [INFO] Booting worker with pid: 253

FIRST LINE : [2026-07-08 06:14:38 +0000] [252] [INFO] Starting gunicorn 20.0.4
READY LINE : [2026-07-08 06:14:38 +0000] [252] [INFO] Listening at: http://0.0.0.0:7777 (252)
DELTA first->ready = 0.231 ms
```

**Run 4 (complete, unedited):**

```
[06:14:41.723] [+    0.000 ms] [2026-07-08 06:14:41 +0000] [279] [INFO] Starting gunicorn 20.0.4
[06:14:41.723] [+    0.199 ms] [2026-07-08 06:14:41 +0000] [279] [INFO] Listening at: http://0.0.0.0:7777 (279)
[06:14:41.723] [+    0.216 ms] [2026-07-08 06:14:41 +0000] [279] [INFO] Using worker: sync
[06:14:41.726] [+    2.721 ms] [2026-07-08 06:14:41 +0000] [280] [INFO] Booting worker with pid: 280

FIRST LINE : [2026-07-08 06:14:41 +0000] [279] [INFO] Starting gunicorn 20.0.4
READY LINE : [2026-07-08 06:14:41 +0000] [279] [INFO] Listening at: http://0.0.0.0:7777 (279)
DELTA first->ready = 0.199 ms
```

First->ready delta across all four runs (stable, sub-millisecond):

```
Run 1: DELTA first->ready = 0.202 ms   ([2026-07-08 06:14:31 +0000] [198] [INFO] Listening at: http://0.0.0.0:7777 (198))
Run 2: DELTA first->ready = 0.187 ms   ([2026-07-08 06:14:34 +0000] [225] [INFO] Listening at: http://0.0.0.0:7777 (225))
Run 3: DELTA first->ready = 0.231 ms   ([2026-07-08 06:14:38 +0000] [252] [INFO] Listening at: http://0.0.0.0:7777 (252))
Run 4: DELTA first->ready = 0.199 ms   ([2026-07-08 06:14:41 +0000] [279] [INFO] Listening at: http://0.0.0.0:7777 (279))
min 0.187 ms · max 0.231 ms · ~0.20 ms typical (4 runs)
```

(Each run's `Booting worker` line carries a larger offset — ~2.4 ms — but that is worker boot, which
occurs **after** the ready line and does not affect the first->ready readiness delta.)

### `file:line` grounding
- `Dockerfile:47` — `CMD ["gunicorn","wsgi:app","-b","0.0.0.0:7777","-w","2","--timeout","15"]`;
  `Dockerfile:44` — `EXPOSE 7777`.
- `wsgi.py:1` — `from server import create_app`; `wsgi.py:3` — `app = create_app()` (the gunicorn
  application object).
- `poetry.lock:1447-1448` — `name = "gunicorn"` / `version = "20.0.4"` (the `Listening at:` line is
  gunicorn's arbiter readiness signal in the 20.x series; here observed verbatim as
  `Listening at: http://0.0.0.0:7777 (<pid>)`).
- `app/log.py:12` — start of the `SL` log format `_log_format`; `:51` — `logger.setLevel(logging.DEBUG)`;
  `:67` — `print(">>> init logging <<<")` on import; `:70-71` — `log = logging.getLogger("werkzeug")`
  then `log.disabled = True` (the werkzeug request logger is explicitly disabled); `:79` —
  `LOG = _get_logger("SL")`.

### Rationale
The readiness signal is gunicorn's arbiter `Listening at:` line: the listening socket is bound at
that moment, so the server can accept connections. The first->ready delta is tiny (~0.20 ms)
because, with the default sync worker and **no** `--preload`, gunicorn's master binds the socket and
logs `Listening at:` **before** forking workers; the application itself (`create_app()`, which
prints `>>> init logging <<<` and `>>> URL: …`) is imported later **inside** each worker, after the
`Booting worker` line. Thus the measured first->ready interval reflects only the master's
socket-bind/log latency, not application import time. The external millisecond timestamper is
required because gunicorn's own timestamps are second-resolution (all four lines in each run share
the same second, e.g. `06:14:31`). The observed sequence
(`Starting gunicorn 20.0.4` -> `Listening at: http://0.0.0.0:7777 (<pid>)` -> `Using worker: sync`
-> `Booting worker with pid`) is the canonical gunicorn 20.x startup order.

## Q3 — Email handler on a custom port (25025)

### Direct answer
- **(a) Yes — the startup log confirms the handler is listening on port `25025`.**
- **(b) Two lines confirm it (both formatted by the `SL` logger):**
  - `Listen for port 25025` (INFO)
  - `Start mail controller 0.0.0.0 25025` (DEBUG)

### Exact command
The real entry point is `email_handler.py`'s `argparse` CLI. `main()` ends in an intentional
`while True: time.sleep(2)` loop (`email_handler.py:2392-2393`), so `timeout` is used purely to let
the process self-terminate after the startup lines are emitted (it does not alter the startup
behavior being observed). Run in the app container with `CONFIG=/root/run_base.env`, `cwd=/host_repo`;
`echo "exit=$?"` is appended so the wrapper's exit status is captured verbatim:

```
timeout --signal=INT 5 python email_handler.py --port 25025 ; echo "exit=$?"
```

### Complete unedited output

```
load config file /root/run_base.env
>>> URL: http://localhost
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
WARNING: Use a temp directory for GNUPGHOME /tmp/ilcmhqcopusbosbnsvnl
Upload files to local dir
>>> init logging <<<
2026-07-08 06:17:07,415 - SL - DEBUG - 351 - "/host_repo/app/utils.py:17" - <module>() -  - load words file: /host_repo/local_data/test_words.txt
2026-07-08 06:17:08,236 - SL - INFO - 351 - "/host_repo/email_handler.py:2403" - <module>() -  - Listen for port 25025
2026-07-08 06:17:08,238 - SL - DEBUG - 351 - "/host_repo/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 25025
Traceback (most recent call last):
  File "/host_repo/email_handler.py", line 2404, in <module>
    main(port=args.port)
  File "/host_repo/email_handler.py", line 2393, in main
    time.sleep(2)
KeyboardInterrupt
exit=124
```

The two relevant lines both carry the port `25025` verbatim (`Listen for port 25025` and
`Start mail controller 0.0.0.0 25025`). The trailing `KeyboardInterrupt` traceback is `timeout`'s
`SIGINT` (from `--signal=INT`) interrupting the intentional `while True: time.sleep(2)` loop. The
final line — captured verbatim from the appended `echo "exit=$?"` — is **`exit=124`**, which is
GNU `timeout`'s standard "command timed out" exit code (the process was still running its loop when
the 5-second limit was reached, so `timeout` terminated it and reported 124). The exit code was
observed to be `124` on repeated runs.

### `file:line` grounding
- `email_handler.py:2397-2401` — the `argparse` parser; `:2399` — argument
  `"-p", "--port", help="SMTP port to listen for", type=int, default=20381` (the default is 20381;
  `--port 25025` overrides it); `:2401` — `args = parser.parse_args()`.
- `email_handler.py:2403` — `LOG.i("Listen for port %s", args.port)` -> emits `Listen for port 25025`.
- `email_handler.py:2404` — `main(port=args.port)`.
- `email_handler.py:2381` — `def main(port: int)`; `:2383` — `controller = Controller(MailHandler(),
  hostname="0.0.0.0", port=port)`; `:2385` — `controller.start()`; `:2386` —
  `LOG.d("Start mail controller %s %s", controller.hostname, controller.port)` -> emits
  `Start mail controller 0.0.0.0 25025`; `:2392-2393` — `while True: time.sleep(2)` (the loop the
  `KeyboardInterrupt` interrupts).
- `app/log.py:51` — `logger.setLevel(logging.DEBUG)`, which is why **both** the INFO line and the
  DEBUG line appear.

### Rationale
The `--port 25025` value flows straight through: `argparse` parses it (`:2401`), `LOG.i("Listen for
port %s", args.port)` echoes it before the controller starts (`:2403`), and inside `main()` the
aiosmtpd `Controller` is constructed with `port=port` and started (`:2383`, `:2385`), after which
`LOG.d("Start mail controller %s %s", controller.hostname, controller.port)` reports the actual
bound host/port (`0.0.0.0 25025`, `:2386`). Because the `SL` logger is at `DEBUG` level
(`app/log.py:51`), the DEBUG confirmation line is shown in addition to the INFO line. This was
exercised through the real CLI entry point — a canonical observation, not a stand-in. The `exit=124`
is the wrapper's (`timeout`'s) status, now shown verbatim in the output, not a claim about
`email_handler.py` itself.

## Q4 — Registration and pre‑activation login

### Direct answer
- **(a) Exact JSON error on pre‑activation login: `{"error":"Account not activated"}`.**
- **(b) HTTP status code: `422 UNPROCESSABLE ENTITY`.**
- **(c) Full curl output:** shown below for both the register call (HTTP `200`,
  `{"msg":"User needs to confirm their account"}`) and the pre‑activation login call (HTTP `422`).
- **(d) Direct database query for the user:** `activated = f` (false) and `notification = t` (true).

> **Note on the anticipated MX‑lookup blocker (reported honestly).** The plan warned that
> `auth_register` calls `email_can_be_used_as_mailbox`, which rejects domains with no MX record
> unless `SKIP_MX_LOOKUP_ON_CHECK` is true — a flag hardcoded `False` and not
> environment‑overridable. **On the canonical stack this blocker did NOT manifest**, so the register
> call succeeded through the **canonical** endpoint with **no** flag toggling and **no**
> non‑canonical workaround. The reason, observed in‑process (below), is that the MX lookup for
> `example.com` returned a non‑empty list (`['']`), so the gate did not trip. The flag was verified
> `False` throughout.

### Exact commands
Server running the canonical entry point under `/root/run_base.env` (`DISABLE_RATE_LIMIT=1`,
`NOT_SEND_EMAIL=true`) against canonical PostgreSQL 13. `curl -sS -i` prints the complete HTTP
response (status line + all headers + body) without curl's TTY progress meter:

```
gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15        # canonical entry point (Dockerfile:47)

curl -sS -i -X POST http://localhost:7777/api/auth/register -H 'Content-Type: application/json' \
     -d '{"email":"testuser@example.com","password":"testpass123"}'

curl -sS -i -X POST http://localhost:7777/api/auth/login    -H 'Content-Type: application/json' \
     -d '{"email":"testuser@example.com","password":"testpass123"}'

psql "postgresql://test:test@sl-pg13:5432/test" \
     -c "SELECT activated, notification FROM users WHERE email='testuser@example.com';"
```

### Complete unedited output

Registration (canonical, HTTP 200):

```
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 06:19:27 GMT
Connection: close
Content-Type: application/json
Content-Length: 46
Access-Control-Allow-Origin: *
Set-Cookie: slapp=1b6544fb-c5cf-4e9c-a47e-893664905886.RUde5O5d_OimQlirU-8yt9VG6Nk; Expires=Wed, 15-Jul-2026 06:19:27 GMT; HttpOnly; Path=/; SameSite=Lax

{"msg":"User needs to confirm their account"}
```

Login **before** activation (HTTP 422):

```
HTTP/1.1 422 UNPROCESSABLE ENTITY
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 06:19:28 GMT
Connection: close
Content-Type: application/json
Content-Length: 34
Access-Control-Allow-Origin: *
Set-Cookie: slapp=832a9250-6625-40f1-9853-38bf7ff9b880.aXKs4dBKbAA-r8iK2_iKIr26KxQ; Expires=Wed, 15-Jul-2026 06:19:28 GMT; HttpOnly; Path=/; SameSite=Lax

{"error":"Account not activated"}
```

Direct database query (part d):

```
 activated | notification 
-----------+--------------
 f         | t
(1 row)
```

> **On the `Set-Cookie: slapp=…` values above.** These are **local, disposable session cookies**
> minted by the Flask session for this single throwaway investigation run against the ephemeral
> local database; they are not credentials, API keys, or production secrets, and they cease to
> exist when the disposable container/DB is torn down. They are shown only because the task requires
> the **complete, unedited** HTTP response (headers included).

In‑process confirmation that the register success was canonical (flag untoggled; MX lookup returned
a non‑empty list, so the gate did not reject):

```
SKIP_MX_LOOKUP_ON_CHECK= False
mx_domains(example.com)= ['']
email_can_be_used_as_mailbox(testuser@example.com)= True
```

### `file:line` grounding
- `app/api/views/auth.py:31` — `def auth_login`; `:30` — `@limiter.limit("10/minute")`. The login
  checks run in order: `:64` wrong password (`:66` -> 400), `:67` disabled (`:69` -> 400), `:70`
  scheduled deletion (`:74` -> 400), then **`:75` `elif not user.activated:` -> `:77`
  `return jsonify(error="Account not activated"), 422`**.
- `app/api/views/auth.py:89` — `def auth_register`; `:110` — `if not
  email_can_be_used_as_mailbox(email) or personal_email_already_used(email):`; `:114` — the
  `cannot use … as personal inbox`/400 branch (**not** taken here). On success it creates the `User`
  and `AccountActivation` and returns `{"msg":"User needs to confirm their account"}`.
- `app/email_utils.py:569` — `def email_can_be_used_as_mailbox`; `:604` — `mx_domains =
  get_mx_domain_list(domain)`; `:607` — `if not config.SKIP_MX_LOOKUP_ON_CHECK and not mx_domains:`;
  `:608` — `LOG.d("No MX record for domain %s", domain)`; `:609` — `return False` (the MX gate,
  **not** taken here because `mx_domains == ['']` is non‑empty).
- `app/config.py:600` — `SKIP_MX_LOOKUP_ON_CHECK = False` (preceded by `:599` `# Only used for
  tests`; not environment‑overridable); `:91` — `NOT_SEND_EMAIL = "NOT_SEND_EMAIL" in os.environ`;
  `:602` — `DISABLE_RATE_LIMIT = "DISABLE_RATE_LIMIT" in os.environ`.
- `app/models.py:354‑356` — the `notification` column; **`default=True` is on `:355`**
  (`sa.Boolean, default=True, nullable=False, server_default="1"`). `:358` — `activated =
  sa.Column(sa.Boolean, default=False, nullable=False, index=True)`. `User.create` sets
  `notification=False` only in the `from_partner` branch, which does not apply to an API
  registration — hence `activated=f, notification=t`.

### Rationale
Registration creates the account in an **unactivated** state (`activated` defaults to `False` on
`app/models.py:358`) and issues an activation code; the API responds
`{"msg":"User needs to confirm their account"}`. Attempting to log in before activation reaches the
`elif not user.activated:` branch of `auth_login` (`auth.py:75`), which returns
`jsonify(error="Account not activated"), 422` (`:77`). The direct query confirms the persisted
state: `activated=f` (the account is not yet confirmed) and `notification=t` (the `notification`
column defaults to `True`/`server_default="1"` on `app/models.py:355` and is only set false on the
partner‑signup path). The anticipated MX blocker did not fire because
`get_mx_domain_list("example.com")` returned `['']` — a non‑empty list — so
`not config.SKIP_MX_LOOKUP_ON_CHECK and not mx_domains` evaluates to
`True and (not ['']) = True and False = False`; the register therefore succeeded on the canonical
path with the flag verified `False`.

## Q5 — Dynamic alias limits

### Direct answer
- **Before the change** (default configuration): `GET /api/user_info` returns
  **`"max_alias_free_plan": 5`**.
- **After** setting `MAX_NB_EMAIL_FREE_PLAN=10` and **restarting** the server: **both** users
  return **`"max_alias_free_plan": 10`** — the pre‑existing user (created while the limit was 5)
  **and** the newly created user.
- **Therefore the new limit applies to _both_ users, not only the user created after the change.**
  The value is **global‑live**: it is computed per request from the global configuration, with no
  per‑user snapshot.

### Exact commands
Same canonical PostgreSQL 13 database throughout (`postgresql://test:test@sl-pg13:5432/test`). The
full flow was executed end‑to‑end by a single script; every command is shown inline (prefixed `$`)
immediately above its complete unedited output in the transcript below. The API‑key flow for each
user is: register -> read the activation code from the DB (email is suppressed by
`NOT_SEND_EMAIL=true`) -> activate -> login with a `device` (which mints an `ApiKey` and returns its
`code`) -> call `/api/user_info` with the `Authentication` header. Between the "before" and "after"
phases the server is **restarted** under a different config (`/root/run_base.env` -> `/root/run_after.env`)
because `MAX_NB_EMAIL_FREE_PLAN` is read once at import time. The 12 steps are:

```
# BEFORE (CONFIG=/root/run_base.env, MAX_NB_EMAIL_FREE_PLAN unset -> default 5)
(1)  curl -sS -i -X POST /api/auth/register   {testuser@example.com}
(2)  psql  SELECT aa.code FROM account_activation aa JOIN users u ON u.id=aa.user_id WHERE u.email='testuser@example.com'
(3)  curl -sS -i -X POST /api/auth/activate   {testuser@example.com, code}
(4)  curl -sS -i -X POST /api/auth/login      {testuser@example.com, testpass123, device:cli}  -> api_key
(5)  curl -sS -i        /api/user_info        -H 'Authentication: <api_key user #1>'              -> 5
# RESTART: CONFIG=/root/run_after.env (MAX_NB_EMAIL_FREE_PLAN=10); confirm value read at import; restart gunicorn
(6)  CONFIG=/root/run_after.env python -c "from app import config; print(config.MAX_NB_EMAIL_FREE_PLAN)"
# AFTER (CONFIG=/root/run_after.env, MAX_NB_EMAIL_FREE_PLAN=10)
(7)  curl -sS -i -X POST /api/auth/register   {q5user2@example.com}
(8)  psql  SELECT aa.code ... WHERE u.email='q5user2@example.com'
(9)  curl -sS -i -X POST /api/auth/activate   {q5user2@example.com, code}
(10) curl -sS -i -X POST /api/auth/login      {q5user2@example.com, testpass123, device:cli}  -> api_key
(11) curl -sS -i        /api/user_info        -H 'Authentication: <api_key user #1, pre-existing>' -> 10
(12) curl -sS -i        /api/user_info        -H 'Authentication: <api_key user #2, new>'          -> 10
```

### Complete unedited output (full end‑to‑end transcript)
The complete transcript of the run is embedded verbatim below. Each `$ …` line is the exact command
executed; the lines beneath it are its complete unedited output. The `(extracted CODE…=…)` /
`(extracted APIKEY…=…)` lines are the script echoing the values it parsed from the preceding output
so the subsequent step's command is fully reproducible and auditable.

```
########## BEFORE — server under CONFIG=/root/run_base.env (default limit 5) ##########
  (alembic upgrade exit=0; public tables now: 77)
----- (1) register user #1 (testuser@example.com) -----
$ curl -sS -i -X POST http://localhost:7777/api/auth/register -H 'Content-Type: application/json' -d '{"email":"testuser@example.com","password":"testpass123"}'
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 06:23:14 GMT
Connection: close
Content-Type: application/json
Content-Length: 46
Access-Control-Allow-Origin: *
Set-Cookie: slapp=61dd7ca0-7032-488e-a190-25d3dd2b237a.lMpHx2Uj1dt37zdilbeSEj-_Ngo; Expires=Wed, 15-Jul-2026 06:23:14 GMT; HttpOnly; Path=/; SameSite=Lax

{"msg":"User needs to confirm their account"}

----- (2) read activation code for user #1 from DB (email suppressed by NOT_SEND_EMAIL) -----
$ psql "postgresql://test:test@sl-pg13:5432/test" -c "SELECT aa.code FROM account_activation aa JOIN users u ON u.id=aa.user_id WHERE u.email='testuser@example.com';"
  code  
--------
 864803
(1 row)


(extracted CODE1=864803)
----- (3) activate user #1 -----
$ curl -sS -i -X POST http://localhost:7777/api/auth/activate -H 'Content-Type: application/json' -d '{"email":"testuser@example.com","code":"864803"}'
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 06:23:14 GMT
Connection: close
Content-Type: application/json
Content-Length: 51
Access-Control-Allow-Origin: *
Set-Cookie: slapp=10aba855-99e8-4e8d-b8f8-a2e62b6af87c.fe545-zsUJwsKMrUjwrYg1k-cDE; Expires=Wed, 15-Jul-2026 06:23:14 GMT; HttpOnly; Path=/; SameSite=Lax

{"msg":"Account is activated, user can login now"}

----- (4) login user #1 with device=cli (mints api_key) -----
$ curl -sS -i -X POST http://localhost:7777/api/auth/login -H 'Content-Type: application/json' -d '{"email":"testuser@example.com","password":"testpass123","device":"cli"}'
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 06:23:14 GMT
Connection: close
Content-Type: application/json
Content-Length: 171
Access-Control-Allow-Origin: *
Set-Cookie: slapp=30210d11-250c-48fd-9372-35c01f34e01f.mFdaZrGA03rfQp4bzhW4_HwOmd4; Expires=Wed, 15-Jul-2026 06:23:14 GMT; HttpOnly; Path=/; SameSite=Lax

{"api_key":"igrimqymragjtzgxyxulvfgdtydtcbdgptenvvwldwoeccknoiaimqhugugr","email":"testuser@example.com","mfa_enabled":false,"mfa_key":null,"name":"testuser@example.com"}

(extracted APIKEY1=igrimqymragjtzgxyxulvfgdtydtcbdgptenvvwldwoeccknoiaimqhugugr)
----- (5) GET /api/user_info for user #1 (BEFORE; expect max_alias_free_plan=5) -----
$ curl -sS -i http://localhost:7777/api/user_info -H 'Authentication: igrimqymragjtzgxyxulvfgdtydtcbdgptenvvwldwoeccknoiaimqhugugr'
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 06:23:15 GMT
Connection: close
Content-Type: application/json
Content-Length: 212
Access-Control-Allow-Origin: *
Set-Cookie: slapp=f37f4fdb-7db5-46fd-99cd-49c1defdf371.ozYPxjQeFrarIettV3AR2QTrmK8; Expires=Wed, 15-Jul-2026 06:23:15 GMT; HttpOnly; Path=/; SameSite=Lax

{"can_create_reverse_alias":true,"connected_proton_address":null,"email":"testuser@example.com","in_trial":true,"is_premium":true,"max_alias_free_plan":5,"name":"testuser@example.com","profile_picture_url":null}

(stopped BEFORE server pid 902)

########## RESTART — value is read at import time, so restart under CONFIG=/root/run_after.env ##########
----- (6) confirm the new config value is read at import under run_after.env -----
$ CONFIG=/root/run_after.env python -c "from app import config; print('MAX_NB_EMAIL_FREE_PLAN =', config.MAX_NB_EMAIL_FREE_PLAN)"
load config file /root/run_after.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/qrryywnqibgpqzphjiih
Upload files to local dir
MAX_NB_EMAIL_FREE_PLAN = 10

(started AFTER server pid 1000 under run_after.env)

########## AFTER — limit now 10; create user #2 then read BOTH users ##########
----- (7) register user #2 (q5user2@example.com) -----
$ curl -sS -i -X POST http://localhost:7777/api/auth/register -H 'Content-Type: application/json' -d '{"email":"q5user2@example.com","password":"testpass123"}'
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 06:23:18 GMT
Connection: close
Content-Type: application/json
Content-Length: 46
Access-Control-Allow-Origin: *
Set-Cookie: slapp=918e9496-1245-4fdd-a422-7610e057b524.6kHMB2guk0uL9WbkfpIGDWKrpFQ; Expires=Wed, 15-Jul-2026 06:23:18 GMT; HttpOnly; Path=/; SameSite=Lax

{"msg":"User needs to confirm their account"}

----- (8) read activation code for user #2 -----
$ psql "postgresql://test:test@sl-pg13:5432/test" -c "SELECT aa.code FROM account_activation aa JOIN users u ON u.id=aa.user_id WHERE u.email='q5user2@example.com';"
  code  
--------
 438611
(1 row)


(extracted CODE2=438611)
----- (9) activate user #2 -----
$ curl -sS -i -X POST http://localhost:7777/api/auth/activate -H 'Content-Type: application/json' -d '{"email":"q5user2@example.com","code":"438611"}'
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 06:23:18 GMT
Connection: close
Content-Type: application/json
Content-Length: 51
Access-Control-Allow-Origin: *
Set-Cookie: slapp=e8086c70-c6ff-47bb-8359-5bd539b3c9a9.9SPQNOFeG7nLSO_gF-jQ057Xp9M; Expires=Wed, 15-Jul-2026 06:23:18 GMT; HttpOnly; Path=/; SameSite=Lax

{"msg":"Account is activated, user can login now"}

----- (10) login user #2 with device=cli (mints api_key) -----
$ curl -sS -i -X POST http://localhost:7777/api/auth/login -H 'Content-Type: application/json' -d '{"email":"q5user2@example.com","password":"testpass123","device":"cli"}'
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 06:23:18 GMT
Connection: close
Content-Type: application/json
Content-Length: 169
Access-Control-Allow-Origin: *
Set-Cookie: slapp=fdf1acc0-986a-42d0-a3d5-eb761dff5599._MBloyp4fkrwp3WhKecYPanES3M; Expires=Wed, 15-Jul-2026 06:23:18 GMT; HttpOnly; Path=/; SameSite=Lax

{"api_key":"fpudojhljfuosivfvultrkbfhnjvhpgnqjoosscwyqmgrngukshqldhagxkn","email":"q5user2@example.com","mfa_enabled":false,"mfa_key":null,"name":"q5user2@example.com"}

(extracted APIKEY2=fpudojhljfuosivfvultrkbfhnjvhpgnqjoosscwyqmgrngukshqldhagxkn)
----- (11) GET /api/user_info for user #1 (PRE-EXISTING, created while limit was 5; expect 10) -----
$ curl -sS -i http://localhost:7777/api/user_info -H 'Authentication: igrimqymragjtzgxyxulvfgdtydtcbdgptenvvwldwoeccknoiaimqhugugr'
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 06:23:19 GMT
Connection: close
Content-Type: application/json
Content-Length: 213
Access-Control-Allow-Origin: *
Set-Cookie: slapp=c83ee467-4f95-45f6-9647-ce52e3eb0a5a.ovlo1Df_ro3vSkhdisN_4HHeH7I; Expires=Wed, 15-Jul-2026 06:23:19 GMT; HttpOnly; Path=/; SameSite=Lax

{"can_create_reverse_alias":true,"connected_proton_address":null,"email":"testuser@example.com","in_trial":true,"is_premium":true,"max_alias_free_plan":10,"name":"testuser@example.com","profile_picture_url":null}

----- (12) GET /api/user_info for user #2 (NEW, created after change; expect 10) -----
$ curl -sS -i http://localhost:7777/api/user_info -H 'Authentication: fpudojhljfuosivfvultrkbfhnjvhpgnqjoosscwyqmgrngukshqldhagxkn'
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 06:23:19 GMT
Connection: close
Content-Type: application/json
Content-Length: 211
Access-Control-Allow-Origin: *
Set-Cookie: slapp=3382b177-326c-4e52-9fae-4cb95cde4230.aggQRTzCQ0abxxgt5uBDzbTu1cI; Expires=Wed, 15-Jul-2026 06:23:19 GMT; HttpOnly; Path=/; SameSite=Lax

{"can_create_reverse_alias":true,"connected_proton_address":null,"email":"q5user2@example.com","in_trial":true,"is_premium":true,"max_alias_free_plan":10,"name":"q5user2@example.com","profile_picture_url":null}

(stopped AFTER server pid 1000)
```

> **On the `api_key`, activation `code`, and `Set-Cookie: slapp=…` values above.** These are
> **local, disposable credentials** minted for a single throwaway investigation run against the
> ephemeral local PostgreSQL 13 database; they are **not** production secrets and cease to exist when
> the disposable container/DB is torn down. They are shown **in full, unredacted**, to satisfy the
> requirement for complete unedited output and to make the register -> activate -> login -> API‑key ->
> `/api/user_info` flow fully auditable end‑to‑end.

### State transition summary (same pre‑existing user #1 across the restart)

```
user #1 (testuser@example.com)  max_alias_free_plan:  5  (before)  ->  10  (after restart)
user #2 (q5user2@example.com)   max_alias_free_plan: 10  (created after the change)
=> BOTH users report 10  (global-live, computed per request; not a per-user snapshot)
```

### `file:line` grounding
- `app/api/views/user_info.py:50` — `@api_bp.route("/user_info")`; `:51` — `@require_api_auth`;
  `:52` — `def user_info()`; `:65` — `user = g.user`; `:67` — `return jsonify(user_to_dict(user))`;
  `:28` — `def user_to_dict(user)`; `:34` — `"max_alias_free_plan": user.max_alias_for_free_account()`.
- `app/models.py:858` — `def max_alias_for_free_account(self)`; `:859-862` — the
  `self.FLAG_FREE_OLD_ALIAS_LIMIT == self.flags & self.FLAG_FREE_OLD_ALIAS_LIMIT` test; `:863` —
  `return config.MAX_NB_EMAIL_OLD_FREE_PLAN` (old‑plan branch, **not** taken); `:865` —
  `return config.MAX_NB_EMAIL_FREE_PLAN` (the branch taken). `:341` — `FLAG_FREE_OLD_ALIAS_LIMIT =
  1 << 2`; `:339` — `FLAG_DISABLE_CREATE_CONTACTS = 1 << 0`; `:546` — `flags` default
  `FLAG_DISABLE_CREATE_CONTACTS`. A normal new user's `flags` = 1, and `1 & 4 == 0`, so the method
  returns `config.MAX_NB_EMAIL_FREE_PLAN`.
- `app/config.py:121` — `MAX_NB_EMAIL_FREE_PLAN = int(os.environ["MAX_NB_EMAIL_FREE_PLAN"])`;
  `:124` — `MAX_NB_EMAIL_FREE_PLAN = 5` (the `except` default). Read at **import** time (inside the
  `try/except` at `:120-124`), hence a restart is required for a change to take effect.
- `app/api/base.py:11` — `api_bp = Blueprint(name="api", import_name=__name__, url_prefix="/api")`
  (so the route is `/api/user_info`); `:16` — `def authorize_request`; `:17` —
  `api_code = request.headers.get("Authentication")`; `:18` — `api_key = ApiKey.get_by(code=api_code)`;
  `:34` — `g.user = api_key.user`; `:52` — `def require_api_auth`.
- `app/api/views/auth.py:146` — `def auth_activate`; `:189` — `user.activated = True`; `:342` —
  `return jsonify(**auth_payload(user, device)), 200`; `:345` — `def auth_payload(user, device)`;
  `:354-357` — `api_key = ApiKey.get_by(...)` / `ApiKey.create(user.id, device)`; `:360` —
  `ret["api_key"] = api_key.code`. Activation codes live in `account_activation.code`
  (`app/models.py:2841` `__tablename__ = "account_activation"`).

### Rationale
`user_info()` serializes the user via `user_to_dict()`, whose `max_alias_free_plan` field calls
`user.max_alias_for_free_account()` **on every request** (`user_info.py:34`). That method reads the
module‑level `config.MAX_NB_EMAIL_FREE_PLAN` (a normal user lacks the `FLAG_FREE_OLD_ALIAS_LIMIT`
bit, so the "old plan" branch at `models.py:863` is not taken; `:865` returns the free‑plan value).
Because `config.MAX_NB_EMAIL_FREE_PLAN` is read from the environment at **import** time
(`config.py:120-124`), the value only changes after a process restart — which is why the server was
restarted under `run_after.env`, and why step (6) prints `MAX_NB_EMAIL_FREE_PLAN = 10` from a fresh
import. Since the limit is a single global value consulted live per request (not stored per user at
signup), raising it to 10 makes **both** the pre‑existing user #1 (whose value went 5 -> 10 across the
restart) and the newly created user #2 report `10`. The observed run‑to‑run behavior with the same
unchanged inputs is fully consistent: 5 before, 10 for both users after.

## Coverage checklist

- **Q1(a)** total tables created = **77** — ✅ (observed via `information_schema`, stable over 2 runs).
- **Q1(b)** last table created by execution order = **`user_audit_log`** — ✅ (migration chain +
  SQL‑echo `CREATE TABLE` evidence).
- **Q2(a)** exact ready message = **`Listening at: http://0.0.0.0:7777 (<pid>)`** — ✅.
- **Q2(b)** ms between first log line and ready message = **~0.20 ms** (0.187–0.231 ms) — ✅
  (external ms timestamper, stable over 4 runs).
- **Q3(a)** startup log confirms listening on 25025 = **Yes** — ✅.
- **Q3(b)** exact messages = **`Listen for port 25025`** (INFO) and **`Start mail controller
  0.0.0.0 25025`** (DEBUG) — ✅.
- **Q4(a)** exact JSON error = **`{"error":"Account not activated"}`** — ✅.
- **Q4(b)** HTTP status = **`422`** — ✅.
- **Q4(c)** full curl output for register (200) and pre‑activation login (422) — ✅.
- **Q4(d)** DB columns for the user = **`activated=f`, `notification=t`** — ✅. Anticipated
  MX‑lookup blocker reported honestly (did not manifest; canonical path succeeded, flag verified
  `False`).
- **Q5 before** = **`"max_alias_free_plan": 5`** — ✅.
- **Q5 after** = **`"max_alias_free_plan": 10`** for **both** users — ✅.
- **Q5 both‑vs‑only** = **both** users reflect the new limit (global‑live, computed per request) — ✅.

_All values above were produced by running the canonical code paths on the canonical stack —
PostgreSQL **13.23** (image `postgres:13`), Redis **6.2.22** (image `redis:6`), and Python
**3.10.18**, exactly as declared in `.github/workflows/main.yml` (`:47`, `:94`, `:17`/`:40`) — and
captured verbatim. The one anticipated risk that did not materialize (the Q4 MX-lookup blocker) is
reported honestly in Q4; it did not affect any answer._

---

## Appendix A — Complete unedited Q1 migration logs (both runs, canonical PostgreSQL 13.23)

This appendix contains the **complete, unedited** output of the exact Q1 sequence
(`drop schema public cascade; create schema public;` -> `alembic upgrade head` -> `alembic current`
-> two `information_schema` queries) for **both** stability runs. Nothing is elided; every one of the
255 `Running upgrade` lines is present. The interleaved `load config file …`, `>>> URL: …`,
`MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value`, `>>> init logging <<<`, and GNUPGHOME
lines are the application's own stdout/stderr on import and are shown verbatim.

### Appendix A.1 — Run 1 (fresh/empty database; reset finds nothing to cascade)

```
===== run1: reset -> alembic upgrade head -> information_schema (canonical PG13) =====
----- step 1: reset to empty schema (mirror scripts/reset_test_db.sh) -----
DROP SCHEMA
CREATE SCHEMA
----- step 2: alembic upgrade head (COMPLETE, unedited) -----
load config file /root/run_base.env
>>> URL: http://localhost
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
WARNING: Use a temp directory for GNUPGHOME /tmp/uetjyzwrfdufmjxbutgs
Upload files to local dir
>>> init logging <<<
2026-07-08 06:07:42,943 - SL - DEBUG - 44 - "/host_repo/app/utils.py:17" - <module>() -  - load words file: /host_repo/local_data/test_words.txt
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade  -> 5e549314e1e2, empty message
INFO  [alembic.runtime.migration] Running upgrade 5e549314e1e2 -> 3cd10cfce8c3, empty message
INFO  [alembic.runtime.migration] Running upgrade 3cd10cfce8c3 -> 0256244cd7c8, empty message
INFO  [alembic.runtime.migration] Running upgrade 0256244cd7c8 -> 213fcca48483, empty message
INFO  [alembic.runtime.migration] Running upgrade 213fcca48483 -> f234688f5ebd, empty message
INFO  [alembic.runtime.migration] Running upgrade f234688f5ebd -> d03e433dc248, empty message
INFO  [alembic.runtime.migration] Running upgrade d03e433dc248 -> 2fe19381f386, empty message
INFO  [alembic.runtime.migration] Running upgrade 2fe19381f386 -> b20ee72fd9a4, empty message
INFO  [alembic.runtime.migration] Running upgrade b20ee72fd9a4 -> 590d89f981c0, empty message
INFO  [alembic.runtime.migration] Running upgrade 590d89f981c0 -> 551c4e6d4a8b, empty message
INFO  [alembic.runtime.migration] Running upgrade 551c4e6d4a8b -> c6e7fc37ad42, empty message
INFO  [alembic.runtime.migration] Running upgrade c6e7fc37ad42 -> 1b7d161d1012, empty message
INFO  [alembic.runtime.migration] Running upgrade 1b7d161d1012 -> 507afb2632cc, empty message
INFO  [alembic.runtime.migration] Running upgrade 507afb2632cc -> 4fac8c8a704c, empty message
INFO  [alembic.runtime.migration] Running upgrade 4fac8c8a704c -> c79c702a1f23, empty message
INFO  [alembic.runtime.migration] Running upgrade c79c702a1f23 -> 5fa68bafae72, empty message
INFO  [alembic.runtime.migration] Running upgrade 5fa68bafae72 -> 4a640c170d02, empty message
INFO  [alembic.runtime.migration] Running upgrade 4a640c170d02 -> 2e2b53afd819, empty message
INFO  [alembic.runtime.migration] Running upgrade 2e2b53afd819 -> 6bbda4685999, empty message
INFO  [alembic.runtime.migration] Running upgrade 6bbda4685999 -> d68a2d971b70, empty message
INFO  [alembic.runtime.migration] Running upgrade d68a2d971b70 -> 0a89c670fc7a, empty message
INFO  [alembic.runtime.migration] Running upgrade 0a89c670fc7a -> 3ebfbaeb76c0, empty message
INFO  [alembic.runtime.migration] Running upgrade 3ebfbaeb76c0 -> 83f4dbe125c4, empty message
INFO  [alembic.runtime.migration] Running upgrade 83f4dbe125c4 -> e505cb517589, empty message
INFO  [alembic.runtime.migration] Running upgrade e505cb517589 -> e83298198ca5, empty message
INFO  [alembic.runtime.migration] Running upgrade e83298198ca5 -> 3a87573bf8a8, empty message
INFO  [alembic.runtime.migration] Running upgrade 3a87573bf8a8 -> a8d8aa307b8b, empty message
INFO  [alembic.runtime.migration] Running upgrade a8d8aa307b8b -> 0b28518684ae, empty message
INFO  [alembic.runtime.migration] Running upgrade 0b28518684ae -> 5e868298fee7, empty message
INFO  [alembic.runtime.migration] Running upgrade 5e868298fee7 -> 2d2fc3e826af, empty message
INFO  [alembic.runtime.migration] Running upgrade 2d2fc3e826af -> 0c7f1a48aac9, empty message
INFO  [alembic.runtime.migration] Running upgrade 0c7f1a48aac9 -> 18e934d58f55, empty message
INFO  [alembic.runtime.migration] Running upgrade 18e934d58f55 -> 9e1b06b9df13, empty message
INFO  [alembic.runtime.migration] Running upgrade 9e1b06b9df13 -> d4e4488a0032, empty message
INFO  [alembic.runtime.migration] Running upgrade d4e4488a0032 -> e409f6214b2b, empty message
INFO  [alembic.runtime.migration] Running upgrade e409f6214b2b -> 696e17c13b8b, empty message
INFO  [alembic.runtime.migration] Running upgrade 696e17c13b8b -> a8b996f0be40, empty message
INFO  [alembic.runtime.migration] Running upgrade a8b996f0be40 -> 10ad2dbaeccf, empty message
INFO  [alembic.runtime.migration] Running upgrade 10ad2dbaeccf -> 01f808f15b2e, empty message
INFO  [alembic.runtime.migration] Running upgrade 01f808f15b2e -> d29cca963221, empty message
INFO  [alembic.runtime.migration] Running upgrade d29cca963221 -> ba6f13ccbabb, empty message
INFO  [alembic.runtime.migration] Running upgrade ba6f13ccbabb -> 7c39ba4ec38d, empty message
INFO  [alembic.runtime.migration] Running upgrade 7c39ba4ec38d -> 9c976df9b9c4, empty message
INFO  [alembic.runtime.migration] Running upgrade 9c976df9b9c4 -> b9f849432543, empty message
INFO  [alembic.runtime.migration] Running upgrade b9f849432543 -> 6664d75ce3d4, empty message
INFO  [alembic.runtime.migration] Running upgrade 6664d75ce3d4 -> 3c9542fc54e9, empty message
INFO  [alembic.runtime.migration] Running upgrade 3c9542fc54e9 -> 3fa3a648c8e7, empty message
INFO  [alembic.runtime.migration] Running upgrade 3fa3a648c8e7 -> 903ec5f566e8, empty message
INFO  [alembic.runtime.migration] Running upgrade 903ec5f566e8 -> f580030d9beb, empty message
INFO  [alembic.runtime.migration] Running upgrade f580030d9beb -> e3cb44b953f2, empty message
INFO  [alembic.runtime.migration] Running upgrade e3cb44b953f2 -> 75093e7ded27, empty message
INFO  [alembic.runtime.migration] Running upgrade 75093e7ded27 -> 5f191273d067, empty message
INFO  [alembic.runtime.migration] Running upgrade 5f191273d067 -> 7eef64ffb398, empty message
INFO  [alembic.runtime.migration] Running upgrade 7eef64ffb398 -> 235355381f53, empty message
INFO  [alembic.runtime.migration] Running upgrade 235355381f53 -> 628a5438295c, empty message
INFO  [alembic.runtime.migration] Running upgrade 628a5438295c -> 11a35b448f83, empty message
INFO  [alembic.runtime.migration] Running upgrade 11a35b448f83 -> 9081f1a90939, empty message
INFO  [alembic.runtime.migration] Running upgrade 9081f1a90939 -> 91b69dfad2f1, empty message
INFO  [alembic.runtime.migration] Running upgrade 91b69dfad2f1 -> 7744c5c16159, empty message
INFO  [alembic.runtime.migration] Running upgrade 7744c5c16159 -> 14167121af69, empty message
INFO  [alembic.runtime.migration] Running upgrade 14167121af69 -> 6e061eb84167, empty message
INFO  [alembic.runtime.migration] Running upgrade 6e061eb84167 -> e9395fe234a4, empty message
INFO  [alembic.runtime.migration] Running upgrade e9395fe234a4 -> 0809266d08ca, empty message
INFO  [alembic.runtime.migration] Running upgrade 0809266d08ca -> f4b8232fa17e, empty message
INFO  [alembic.runtime.migration] Running upgrade f4b8232fa17e -> dbd80d290f04, empty message
INFO  [alembic.runtime.migration] Running upgrade dbd80d290f04 -> 4e4a759ac4b5, empty message
INFO  [alembic.runtime.migration] Running upgrade 4e4a759ac4b5 -> 30c13ca016e4, empty message
INFO  [alembic.runtime.migration] Running upgrade 30c13ca016e4 -> 541ce53ab6e9, empty message
INFO  [alembic.runtime.migration] Running upgrade 541ce53ab6e9 -> 67c61eead8d2, empty message
INFO  [alembic.runtime.migration] Running upgrade 67c61eead8d2 -> 224fd8963462, empty message
INFO  [alembic.runtime.migration] Running upgrade 224fd8963462 -> 92baf66b268b, empty message
INFO  [alembic.runtime.migration] Running upgrade 92baf66b268b -> 497cfd2a02e2, empty message
INFO  [alembic.runtime.migration] Running upgrade 497cfd2a02e2 -> ea30c0b5b2e3, empty message
INFO  [alembic.runtime.migration] Running upgrade ea30c0b5b2e3 -> bfd7b2302903, empty message
INFO  [alembic.runtime.migration] Running upgrade bfd7b2302903 -> 57ef03f3ac34, empty message
INFO  [alembic.runtime.migration] Running upgrade 57ef03f3ac34 -> dd911f880b75, empty message
INFO  [alembic.runtime.migration] Running upgrade dd911f880b75 -> bd05eac83f5f, empty message
INFO  [alembic.runtime.migration] Running upgrade bd05eac83f5f -> b4146f7d5277, empty message
INFO  [alembic.runtime.migration] Running upgrade b4146f7d5277 -> f939d67374e4, empty message
INFO  [alembic.runtime.migration] Running upgrade f939d67374e4 -> de1b457472e0, empty message
INFO  [alembic.runtime.migration] Running upgrade de1b457472e0 -> ae94fe5c4e9f, empty message
INFO  [alembic.runtime.migration] Running upgrade ae94fe5c4e9f -> 026e7a782ed6, empty message
INFO  [alembic.runtime.migration] Running upgrade 026e7a782ed6 -> 925b93d92809, empty message
INFO  [alembic.runtime.migration] Running upgrade 925b93d92809 -> bdf76f4b65a2, empty message
INFO  [alembic.runtime.migration] Running upgrade bdf76f4b65a2 -> a3a7c518ea70, empty message
INFO  [alembic.runtime.migration] Running upgrade a3a7c518ea70 -> a5e3c6693dc6, empty message
INFO  [alembic.runtime.migration] Running upgrade a5e3c6693dc6 -> bf11ab2f0a7a, empty message
INFO  [alembic.runtime.migration] Running upgrade bf11ab2f0a7a -> 1759f73274ee, empty message
INFO  [alembic.runtime.migration] Running upgrade 1759f73274ee -> 552d735a2f1f, empty message
INFO  [alembic.runtime.migration] Running upgrade 552d735a2f1f -> 5cad8fa84386, empty message
INFO  [alembic.runtime.migration] Running upgrade 5cad8fa84386 -> c31cdf879ee3, empty message
INFO  [alembic.runtime.migration] Running upgrade c31cdf879ee3 -> 659d979b64ce, empty message
INFO  [alembic.runtime.migration] Running upgrade 659d979b64ce -> ce15cf3467b4, empty message
INFO  [alembic.runtime.migration] Running upgrade ce15cf3467b4 -> 0e08145f0499, empty message
INFO  [alembic.runtime.migration] Running upgrade 0e08145f0499 -> 00532ac6d4bc, empty message
INFO  [alembic.runtime.migration] Running upgrade 00532ac6d4bc -> f680032cc361, empty message
INFO  [alembic.runtime.migration] Running upgrade f680032cc361 -> 10a7947fda6b, empty message
INFO  [alembic.runtime.migration] Running upgrade 10a7947fda6b -> 4a7d35941602, empty message
INFO  [alembic.runtime.migration] Running upgrade 4a7d35941602 -> cfc013b6461a, empty message
INFO  [alembic.runtime.migration] Running upgrade cfc013b6461a -> b2d51e4d94c8, empty message
INFO  [alembic.runtime.migration] Running upgrade b2d51e4d94c8 -> 749c2b85d20f, empty message
INFO  [alembic.runtime.migration] Running upgrade 749c2b85d20f -> a5b4dc311a89, empty message
INFO  [alembic.runtime.migration] Running upgrade a5b4dc311a89 -> a3c9a43e41f4, empty message
INFO  [alembic.runtime.migration] Running upgrade a3c9a43e41f4 -> 7128f87af701, empty message
INFO  [alembic.runtime.migration] Running upgrade 7128f87af701 -> 270d598c51e3, empty message
INFO  [alembic.runtime.migration] Running upgrade 270d598c51e3 -> b77ab8c47cc7, empty message
INFO  [alembic.runtime.migration] Running upgrade b77ab8c47cc7 -> a2b95b04d1f7, empty message
INFO  [alembic.runtime.migration] Running upgrade a2b95b04d1f7 -> 63fd3b240583, empty message
INFO  [alembic.runtime.migration] Running upgrade 63fd3b240583 -> 95938a93ea14, empty message
INFO  [alembic.runtime.migration] Running upgrade 95938a93ea14 -> b82bcad9accf, empty message
INFO  [alembic.runtime.migration] Running upgrade b82bcad9accf -> 84471852b610, empty message
INFO  [alembic.runtime.migration] Running upgrade 84471852b610 -> b0e9a389939a, empty message
INFO  [alembic.runtime.migration] Running upgrade b0e9a389939a -> 198c3aca9d8d, empty message
INFO  [alembic.runtime.migration] Running upgrade 198c3aca9d8d -> 58ad4df8583e, empty message
INFO  [alembic.runtime.migration] Running upgrade 58ad4df8583e -> 1abfc9e14d7e, empty message
INFO  [alembic.runtime.migration] Running upgrade 1abfc9e14d7e -> 32b00d06d892, empty message
INFO  [alembic.runtime.migration] Running upgrade 32b00d06d892 -> b17afc77ba83, empty message
INFO  [alembic.runtime.migration] Running upgrade b17afc77ba83 -> 54ca2dbf89c0, empty message
INFO  [alembic.runtime.migration] Running upgrade 54ca2dbf89c0 -> eef0c404b531, empty message
INFO  [alembic.runtime.migration] Running upgrade eef0c404b531 -> 84dec6c29c48, empty message
INFO  [alembic.runtime.migration] Running upgrade 84dec6c29c48 -> d0f197979bd9, empty message
INFO  [alembic.runtime.migration] Running upgrade d0f197979bd9 -> 9dc16e591f88, empty message
INFO  [alembic.runtime.migration] Running upgrade 9dc16e591f88 -> ac41029fb329, empty message
INFO  [alembic.runtime.migration] Running upgrade ac41029fb329 -> d1edb3cadec8, empty message
INFO  [alembic.runtime.migration] Running upgrade d1edb3cadec8 -> 623662ea0e7e, empty message
INFO  [alembic.runtime.migration] Running upgrade 623662ea0e7e -> 56c790ec8ab4, empty message
INFO  [alembic.runtime.migration] Running upgrade 56c790ec8ab4 -> c0d91ff18f77, empty message
INFO  [alembic.runtime.migration] Running upgrade c0d91ff18f77 -> 780a8344914b, empty message
INFO  [alembic.runtime.migration] Running upgrade 780a8344914b -> a20aeb9b0eac, empty message
INFO  [alembic.runtime.migration] Running upgrade a20aeb9b0eac -> 0af2c2e286a7, empty message
INFO  [alembic.runtime.migration] Running upgrade 0af2c2e286a7 -> 1919f1859215, empty message
INFO  [alembic.runtime.migration] Running upgrade 1919f1859215 -> f66ca777f409, empty message
INFO  [alembic.runtime.migration] Running upgrade f66ca777f409 -> 7c0dbd378cdb, empty message
INFO  [alembic.runtime.migration] Running upgrade 7c0dbd378cdb -> e99989e6ad56, empty message
INFO  [alembic.runtime.migration] Running upgrade e99989e6ad56 -> 1b54995bc086, empty message
INFO  [alembic.runtime.migration] Running upgrade 1b54995bc086 -> 2779eb90c6c4, empty message
INFO  [alembic.runtime.migration] Running upgrade 2779eb90c6c4 -> 74906d31d994, empty message
INFO  [alembic.runtime.migration] Running upgrade 74906d31d994 -> 85d0655d42c0, empty message
INFO  [alembic.runtime.migration] Running upgrade 85d0655d42c0 -> de7aa5280210, empty message
INFO  [alembic.runtime.migration] Running upgrade de7aa5280210 -> e831a883153a, empty message
INFO  [alembic.runtime.migration] Running upgrade e831a883153a -> d1236c4dff71, empty message
INFO  [alembic.runtime.migration] Running upgrade d1236c4dff71 -> 94f14eb0fe5b, empty message
INFO  [alembic.runtime.migration] Running upgrade 94f14eb0fe5b -> 9d6adad83936, empty message
INFO  [alembic.runtime.migration] Running upgrade 9d6adad83936 -> f398b261d9c6, empty message
INFO  [alembic.runtime.migration] Running upgrade f398b261d9c6 -> 517b79c56088, empty message
INFO  [alembic.runtime.migration] Running upgrade 517b79c56088 -> 48b991e9de06, empty message
INFO  [alembic.runtime.migration] Running upgrade 48b991e9de06 -> e11c3dd48a6f, empty message
INFO  [alembic.runtime.migration] Running upgrade e11c3dd48a6f -> 4912f3bd5ba2, empty message
INFO  [alembic.runtime.migration] Running upgrade 4912f3bd5ba2 -> f5133dc851ee, empty message
INFO  [alembic.runtime.migration] Running upgrade f5133dc851ee -> 5c77d685df87, empty message
INFO  [alembic.runtime.migration] Running upgrade 5c77d685df87 -> 6cc7f073b358, empty message
INFO  [alembic.runtime.migration] Running upgrade 6cc7f073b358 -> 68e2f38e33f4, empty message
INFO  [alembic.runtime.migration] Running upgrade 68e2f38e33f4 -> fc2eb1d7e4fc, empty message
INFO  [alembic.runtime.migration] Running upgrade fc2eb1d7e4fc -> a5e643d562c9, empty message
INFO  [alembic.runtime.migration] Running upgrade a5e643d562c9 -> 29ea13ed76f9, empty message
INFO  [alembic.runtime.migration] Running upgrade 29ea13ed76f9 -> 8e70205a5308, empty message
INFO  [alembic.runtime.migration] Running upgrade 8e70205a5308 -> f3f19998b755, empty message
INFO  [alembic.runtime.migration] Running upgrade f3f19998b755 -> c31a081eab74, empty message
INFO  [alembic.runtime.migration] Running upgrade c31a081eab74 -> 78403c7b8089, empty message
INFO  [alembic.runtime.migration] Running upgrade 78403c7b8089 -> 5662122eac21, empty message
INFO  [alembic.runtime.migration] Running upgrade 5662122eac21 -> 20c738810b1b, empty message
INFO  [alembic.runtime.migration] Running upgrade 20c738810b1b -> dfee471558bd, empty message
INFO  [alembic.runtime.migration] Running upgrade dfee471558bd -> 05e3af59929a, empty message
INFO  [alembic.runtime.migration] Running upgrade 05e3af59929a -> c3470e2d3224, empty message
INFO  [alembic.runtime.migration] Running upgrade c3470e2d3224 -> ffa75d04e6ef, empty message
INFO  [alembic.runtime.migration] Running upgrade ffa75d04e6ef -> 9014cca7097c, empty message
INFO  [alembic.runtime.migration] Running upgrade 9014cca7097c -> d4392342465f, empty message
INFO  [alembic.runtime.migration] Running upgrade d4392342465f -> 424808e1fe49, empty message
INFO  [alembic.runtime.migration] Running upgrade 424808e1fe49 -> 916a5257d18c, empty message
INFO  [alembic.runtime.migration] Running upgrade 916a5257d18c -> 4d3f91ddf3e9, empty message
INFO  [alembic.runtime.migration] Running upgrade 4d3f91ddf3e9 -> d8c55e79da54, empty message
INFO  [alembic.runtime.migration] Running upgrade d8c55e79da54 -> cf1e8c1bc737, empty message
INFO  [alembic.runtime.migration] Running upgrade cf1e8c1bc737 -> 7a105bfc0cd0, empty message
INFO  [alembic.runtime.migration] Running upgrade 7a105bfc0cd0 -> bc75acacc98e, empty message
INFO  [alembic.runtime.migration] Running upgrade bc75acacc98e -> b8b4f9598240, empty message
INFO  [alembic.runtime.migration] Running upgrade b8b4f9598240 -> 5ee767807344, empty message
INFO  [alembic.runtime.migration] Running upgrade 5ee767807344 -> 4913cb3f5a05, empty message
INFO  [alembic.runtime.migration] Running upgrade 4913cb3f5a05 -> 0b1c9ea11aef, empty message
INFO  [alembic.runtime.migration] Running upgrade 0b1c9ea11aef -> 2fbcad5527d7, empty message
INFO  [alembic.runtime.migration] Running upgrade 2fbcad5527d7 -> d750d578b068, empty message
INFO  [alembic.runtime.migration] Running upgrade d750d578b068 -> 2f1b3c759773, empty message
INFO  [alembic.runtime.migration] Running upgrade 2f1b3c759773 -> 99d9e329b27f, empty message
INFO  [alembic.runtime.migration] Running upgrade 99d9e329b27f -> a06066e3fbeb, empty message
INFO  [alembic.runtime.migration] Running upgrade a06066e3fbeb -> d67eab226ecd, empty message
INFO  [alembic.runtime.migration] Running upgrade d67eab226ecd -> bbedc353f90c, empty message
INFO  [alembic.runtime.migration] Running upgrade bbedc353f90c -> 0b9150eb309d, Increase message_id length manually
INFO  [alembic.runtime.migration] Running upgrade 0b9150eb309d -> 6204e57b4bc4, empty message
INFO  [alembic.runtime.migration] Running upgrade 6204e57b4bc4 -> 37feaba7c45d, empty message
INFO  [alembic.runtime.migration] Running upgrade 37feaba7c45d -> ff6c04869029, empty message
INFO  [alembic.runtime.migration] Running upgrade ff6c04869029 -> fdb02bd105a8, empty message
INFO  [alembic.runtime.migration] Running upgrade fdb02bd105a8 -> dd278f96ca83, empty message
INFO  [alembic.runtime.migration] Running upgrade dd278f96ca83 -> 1076b5795b08, empty message
INFO  [alembic.runtime.migration] Running upgrade 1076b5795b08 -> 5639ad89ee50, empty message
INFO  [alembic.runtime.migration] Running upgrade 5639ad89ee50 -> 11ba83e2dd71, empty message
INFO  [alembic.runtime.migration] Running upgrade 11ba83e2dd71 -> ccbfb61eda0d, empty message
INFO  [alembic.runtime.migration] Running upgrade ccbfb61eda0d -> a5013ff0a00a, empty message
INFO  [alembic.runtime.migration] Running upgrade a5013ff0a00a -> e6e8e12f5a13, empty message
INFO  [alembic.runtime.migration] Running upgrade e6e8e12f5a13 -> 9031c9e28510, empty message
INFO  [alembic.runtime.migration] Running upgrade 9031c9e28510 -> b8fd175c084a, empty message
INFO  [alembic.runtime.migration] Running upgrade b8fd175c084a -> e7d7ebcea26c, empty message
INFO  [alembic.runtime.migration] Running upgrade e7d7ebcea26c -> d0ccd9d7ac0c, empty message
INFO  [alembic.runtime.migration] Running upgrade d0ccd9d7ac0c -> 4b483a762fed, empty message
INFO  [alembic.runtime.migration] Running upgrade 4b483a762fed -> ad467baf7ec8, empty message
INFO  [alembic.runtime.migration] Running upgrade ad467baf7ec8 -> d8a3dfe674f2, empty message
INFO  [alembic.runtime.migration] Running upgrade d8a3dfe674f2 -> 3d05479d0d11, empty message
INFO  [alembic.runtime.migration] Running upgrade 3d05479d0d11 -> 753d2ed92d41, empty message
INFO  [alembic.runtime.migration] Running upgrade 753d2ed92d41 -> 698424c429e9, empty message
INFO  [alembic.runtime.migration] Running upgrade 698424c429e9 -> 07b870d7cc86, empty message
INFO  [alembic.runtime.migration] Running upgrade 07b870d7cc86 -> 9282e982bc05, Add block_behaviour setting for user
INFO  [alembic.runtime.migration] Running upgrade 9282e982bc05 -> 5047fcbd57c7, empty message
INFO  [alembic.runtime.migration] Running upgrade 5047fcbd57c7 -> 4729b7096d12, empty message
INFO  [alembic.runtime.migration] Running upgrade 4729b7096d12 -> b500363567e3, Create admin audit log
INFO  [alembic.runtime.migration] Running upgrade b500363567e3 -> 28b9b14c9664, store provider complaints
INFO  [alembic.runtime.migration] Running upgrade 28b9b14c9664 -> 0aaad1740797, store provider complaints
INFO  [alembic.runtime.migration] Running upgrade 0aaad1740797 -> e866ad0e78e1, Add partner tables
INFO  [alembic.runtime.migration] Running upgrade e866ad0e78e1 -> 088f23324464, add flags to the user model
INFO  [alembic.runtime.migration] Running upgrade 088f23324464 -> 2b1d3cd93e4b, update partner_api_token token length
INFO  [alembic.runtime.migration] Running upgrade 2b1d3cd93e4b -> 82d3c7109ffb, partner_user and partner_subscription
INFO  [alembic.runtime.migration] Running upgrade 82d3c7109ffb -> 36646e5dc6d9, make external_user_id non nullable
INFO  [alembic.runtime.migration] Running upgrade 36646e5dc6d9 -> a7bcb872c12a, Add alias transfer token expiration
INFO  [alembic.runtime.migration] Running upgrade a7bcb872c12a -> 673a074e4215, empty message
INFO  [alembic.runtime.migration] Running upgrade 673a074e4215 -> d1fb679f7eec, Add sudo expiration for ApiKeys
INFO  [alembic.runtime.migration] Running upgrade d1fb679f7eec -> bfebc2d5c719, Add state to job
INFO  [alembic.runtime.migration] Running upgrade bfebc2d5c719 -> 516c21ea7d87, empty message
INFO  [alembic.runtime.migration] Running upgrade 516c21ea7d87 -> bd7d032087b2, empty message
INFO  [alembic.runtime.migration] Running upgrade bd7d032087b2 -> b0101a66bb77, Add unsubscribe behaviour
INFO  [alembic.runtime.migration] Running upgrade b0101a66bb77 -> 89081a00fc7d, default_unsub_behaviour
INFO  [alembic.runtime.migration] Running upgrade 89081a00fc7d -> c66f2c5b6cb1, empty message
INFO  [alembic.runtime.migration] Running upgrade c66f2c5b6cb1 -> 9cc0f0712b29, Add api to cookie token
INFO  [alembic.runtime.migration] Running upgrade 9cc0f0712b29 -> bd95b2b4217f, Updated recovery code string length
INFO  [alembic.runtime.migration] Running upgrade bd95b2b4217f -> 2c2093c82bc0, empty message
INFO  [alembic.runtime.migration] Running upgrade 2c2093c82bc0 -> 5f4a5625da66, empty message
INFO  [alembic.runtime.migration] Running upgrade 5f4a5625da66 -> 893c0d18475f, empty message
INFO  [alembic.runtime.migration] Running upgrade 893c0d18475f -> bc496c0a0279, empty message
INFO  [alembic.runtime.migration] Running upgrade bc496c0a0279 -> 2d89315ac650, empty message
INFO  [alembic.runtime.migration] Running upgrade 893c0d18475f -> 01e2997e90d3, empty message
INFO  [alembic.runtime.migration] Running upgrade 01e2997e90d3, 2d89315ac650 -> 2634b41f54db, empty message
INFO  [alembic.runtime.migration] Running upgrade 2634b41f54db -> 01827104004b, empty message
INFO  [alembic.runtime.migration] Running upgrade 01827104004b -> 0a5701a4f5e4, empty message
INFO  [alembic.runtime.migration] Running upgrade 0a5701a4f5e4 -> ec7fdde8da9f, empty message
INFO  [alembic.runtime.migration] Running upgrade ec7fdde8da9f -> 46ecb648a47e, empty message
INFO  [alembic.runtime.migration] Running upgrade 46ecb648a47e -> 4bc54632d9aa, empty message
INFO  [alembic.runtime.migration] Running upgrade 4bc54632d9aa -> 818b0a956205, empty message
INFO  [alembic.runtime.migration] Running upgrade 818b0a956205 -> 52510a633d6f, empty message
INFO  [alembic.runtime.migration] Running upgrade 52510a633d6f -> fa2f19bb4e5a, empty message
INFO  [alembic.runtime.migration] Running upgrade fa2f19bb4e5a -> 06a9a7133445, Create sync_event table
INFO  [alembic.runtime.migration] Running upgrade 06a9a7133445 -> d608b8e48082, empty message
INFO  [alembic.runtime.migration] Running upgrade d608b8e48082 -> 56d08955fcab, add retry count to sync event
INFO  [alembic.runtime.migration] Running upgrade 56d08955fcab -> 1c14339aae90, empty message
INFO  [alembic.runtime.migration] Running upgrade 1c14339aae90 -> 2441b7ff5da9, Custom Domain partner id
INFO  [alembic.runtime.migration] Running upgrade 2441b7ff5da9 -> 88dd7a0abf54, contact.flags and custom_domain.pending_deletion
INFO  [alembic.runtime.migration] Running upgrade 88dd7a0abf54 -> 62afa3a10010, custom domain indices
INFO  [alembic.runtime.migration] Running upgrade 62afa3a10010 -> 91ed7f46dc81, alias_audit_log
INFO  [alembic.runtime.migration] Running upgrade 91ed7f46dc81 -> 7d7b84779837, user_audit_log
INFO  [alembic.runtime.migration] Running upgrade 7d7b84779837 -> 32f25cbf12f6, alias_audit_log_index_created_at
----- step 3: alembic current -----
load config file /root/run_base.env
>>> URL: http://localhost
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
WARNING: Use a temp directory for GNUPGHOME /tmp/evymtuqdqpbvmpwojzwu
Upload files to local dir
>>> init logging <<<
2026-07-08 06:07:44,567 - SL - DEBUG - 45 - "/host_repo/app/utils.py:17" - <module>() -  - load words file: /host_repo/local_data/test_words.txt
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
32f25cbf12f6 (head)
----- step 4: table count (information_schema) -----
 count 
-------
    77
(1 row)

----- step 5: confirm last-created + bookkeeping tables present -----
   table_name    
-----------------
 alembic_version
 alias_audit_log
 user_audit_log
(3 rows)
```

### Appendix A.2 — Run 2 (reset of the now-populated database; `drop schema … cascade` enumerates all 82 other objects)

```
===== run2: reset -> alembic upgrade head -> information_schema (canonical PG13) =====
----- step 1: reset to empty schema (mirror scripts/reset_test_db.sh) -----
NOTICE:  drop cascades to 82 other objects
DETAIL:  drop cascades to table alembic_version
drop cascades to type plan_enum
drop cascades to table file
drop cascades to table users
drop cascades to table activation_code
drop cascades to table client
drop cascades to table alias
drop cascades to table authorization_code
drop cascades to table client_user
drop cascades to table oauth_token
drop cascades to table redirect_uri
drop cascades to table reset_password_code
drop cascades to table contact
drop cascades to type planenum2
drop cascades to table subscription
drop cascades to table email_log
drop cascades to table deleted_alias
drop cascades to table email_change
drop cascades to table api_key
drop cascades to table alias_used_on
drop cascades to table custom_domain
drop cascades to table lifetime_coupon
drop cascades to table directory
drop cascades to table job
drop cascades to table mailbox
drop cascades to table manual_subscription
drop cascades to table social_auth
drop cascades to table account_activation
drop cascades to table refused_email
drop cascades to table referral
drop cascades to type planenum_apple
drop cascades to table apple_subscription
drop cascades to table sent_alert
drop cascades to table alias_mailbox
drop cascades to table recovery_code
drop cascades to table domain_deleted_alias
drop cascades to table notification
drop cascades to table fido
drop cascades to table mfa_browser
drop cascades to table directory_mailbox
drop cascades to table public_domain
drop cascades to table domain_mailbox
drop cascades to table monitoring
drop cascades to table batch_import
drop cascades to table authorized_address
drop cascades to table coinbase_subscription
drop cascades to table bounce
drop cascades to table transactional_email
drop cascades to table metric2
drop cascades to table payout
drop cascades to table hibp
drop cascades to table alias_hibp
drop cascades to table ignored_email
drop cascades to table coupon
drop cascades to table hibp_notified_alias
drop cascades to table ignore_bounce_sender
drop cascades to extension pg_trgm
drop cascades to table auto_create_rule
drop cascades to table auto_create_rule__mailbox
drop cascades to table message_id_matching
drop cascades to table deleted_directory
drop cascades to table deleted_subdomain
drop cascades to table phone_country
drop cascades to table phone_number
drop cascades to table phone_message
drop cascades to table phone_reservation
drop cascades to table invalid_mailbox_domain
drop cascades to type block_behaviour_enum
drop cascades to table admin_audit_log
drop cascades to table provider_complaint
drop cascades to table partner
drop cascades to table partner_api_token
drop cascades to table partner_user
drop cascades to table partner_subscription
drop cascades to table newsletter
drop cascades to table newsletter_user
drop cascades to table api_cookie_token
drop cascades to table daily_metric
drop cascades to table sync_event
drop cascades to table mailbox_activation
drop cascades to table alias_audit_log
drop cascades to table user_audit_log
DROP SCHEMA
CREATE SCHEMA
----- step 2: alembic upgrade head (COMPLETE, unedited) -----
load config file /root/run_base.env
>>> URL: http://localhost
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
WARNING: Use a temp directory for GNUPGHOME /tmp/qlwijfyztnnkghliovtd
Upload files to local dir
>>> init logging <<<
2026-07-08 06:09:01,541 - SL - DEBUG - 127 - "/host_repo/app/utils.py:17" - <module>() -  - load words file: /host_repo/local_data/test_words.txt
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade  -> 5e549314e1e2, empty message
INFO  [alembic.runtime.migration] Running upgrade 5e549314e1e2 -> 3cd10cfce8c3, empty message
INFO  [alembic.runtime.migration] Running upgrade 3cd10cfce8c3 -> 0256244cd7c8, empty message
INFO  [alembic.runtime.migration] Running upgrade 0256244cd7c8 -> 213fcca48483, empty message
INFO  [alembic.runtime.migration] Running upgrade 213fcca48483 -> f234688f5ebd, empty message
INFO  [alembic.runtime.migration] Running upgrade f234688f5ebd -> d03e433dc248, empty message
INFO  [alembic.runtime.migration] Running upgrade d03e433dc248 -> 2fe19381f386, empty message
INFO  [alembic.runtime.migration] Running upgrade 2fe19381f386 -> b20ee72fd9a4, empty message
INFO  [alembic.runtime.migration] Running upgrade b20ee72fd9a4 -> 590d89f981c0, empty message
INFO  [alembic.runtime.migration] Running upgrade 590d89f981c0 -> 551c4e6d4a8b, empty message
INFO  [alembic.runtime.migration] Running upgrade 551c4e6d4a8b -> c6e7fc37ad42, empty message
INFO  [alembic.runtime.migration] Running upgrade c6e7fc37ad42 -> 1b7d161d1012, empty message
INFO  [alembic.runtime.migration] Running upgrade 1b7d161d1012 -> 507afb2632cc, empty message
INFO  [alembic.runtime.migration] Running upgrade 507afb2632cc -> 4fac8c8a704c, empty message
INFO  [alembic.runtime.migration] Running upgrade 4fac8c8a704c -> c79c702a1f23, empty message
INFO  [alembic.runtime.migration] Running upgrade c79c702a1f23 -> 5fa68bafae72, empty message
INFO  [alembic.runtime.migration] Running upgrade 5fa68bafae72 -> 4a640c170d02, empty message
INFO  [alembic.runtime.migration] Running upgrade 4a640c170d02 -> 2e2b53afd819, empty message
INFO  [alembic.runtime.migration] Running upgrade 2e2b53afd819 -> 6bbda4685999, empty message
INFO  [alembic.runtime.migration] Running upgrade 6bbda4685999 -> d68a2d971b70, empty message
INFO  [alembic.runtime.migration] Running upgrade d68a2d971b70 -> 0a89c670fc7a, empty message
INFO  [alembic.runtime.migration] Running upgrade 0a89c670fc7a -> 3ebfbaeb76c0, empty message
INFO  [alembic.runtime.migration] Running upgrade 3ebfbaeb76c0 -> 83f4dbe125c4, empty message
INFO  [alembic.runtime.migration] Running upgrade 83f4dbe125c4 -> e505cb517589, empty message
INFO  [alembic.runtime.migration] Running upgrade e505cb517589 -> e83298198ca5, empty message
INFO  [alembic.runtime.migration] Running upgrade e83298198ca5 -> 3a87573bf8a8, empty message
INFO  [alembic.runtime.migration] Running upgrade 3a87573bf8a8 -> a8d8aa307b8b, empty message
INFO  [alembic.runtime.migration] Running upgrade a8d8aa307b8b -> 0b28518684ae, empty message
INFO  [alembic.runtime.migration] Running upgrade 0b28518684ae -> 5e868298fee7, empty message
INFO  [alembic.runtime.migration] Running upgrade 5e868298fee7 -> 2d2fc3e826af, empty message
INFO  [alembic.runtime.migration] Running upgrade 2d2fc3e826af -> 0c7f1a48aac9, empty message
INFO  [alembic.runtime.migration] Running upgrade 0c7f1a48aac9 -> 18e934d58f55, empty message
INFO  [alembic.runtime.migration] Running upgrade 18e934d58f55 -> 9e1b06b9df13, empty message
INFO  [alembic.runtime.migration] Running upgrade 9e1b06b9df13 -> d4e4488a0032, empty message
INFO  [alembic.runtime.migration] Running upgrade d4e4488a0032 -> e409f6214b2b, empty message
INFO  [alembic.runtime.migration] Running upgrade e409f6214b2b -> 696e17c13b8b, empty message
INFO  [alembic.runtime.migration] Running upgrade 696e17c13b8b -> a8b996f0be40, empty message
INFO  [alembic.runtime.migration] Running upgrade a8b996f0be40 -> 10ad2dbaeccf, empty message
INFO  [alembic.runtime.migration] Running upgrade 10ad2dbaeccf -> 01f808f15b2e, empty message
INFO  [alembic.runtime.migration] Running upgrade 01f808f15b2e -> d29cca963221, empty message
INFO  [alembic.runtime.migration] Running upgrade d29cca963221 -> ba6f13ccbabb, empty message
INFO  [alembic.runtime.migration] Running upgrade ba6f13ccbabb -> 7c39ba4ec38d, empty message
INFO  [alembic.runtime.migration] Running upgrade 7c39ba4ec38d -> 9c976df9b9c4, empty message
INFO  [alembic.runtime.migration] Running upgrade 9c976df9b9c4 -> b9f849432543, empty message
INFO  [alembic.runtime.migration] Running upgrade b9f849432543 -> 6664d75ce3d4, empty message
INFO  [alembic.runtime.migration] Running upgrade 6664d75ce3d4 -> 3c9542fc54e9, empty message
INFO  [alembic.runtime.migration] Running upgrade 3c9542fc54e9 -> 3fa3a648c8e7, empty message
INFO  [alembic.runtime.migration] Running upgrade 3fa3a648c8e7 -> 903ec5f566e8, empty message
INFO  [alembic.runtime.migration] Running upgrade 903ec5f566e8 -> f580030d9beb, empty message
INFO  [alembic.runtime.migration] Running upgrade f580030d9beb -> e3cb44b953f2, empty message
INFO  [alembic.runtime.migration] Running upgrade e3cb44b953f2 -> 75093e7ded27, empty message
INFO  [alembic.runtime.migration] Running upgrade 75093e7ded27 -> 5f191273d067, empty message
INFO  [alembic.runtime.migration] Running upgrade 5f191273d067 -> 7eef64ffb398, empty message
INFO  [alembic.runtime.migration] Running upgrade 7eef64ffb398 -> 235355381f53, empty message
INFO  [alembic.runtime.migration] Running upgrade 235355381f53 -> 628a5438295c, empty message
INFO  [alembic.runtime.migration] Running upgrade 628a5438295c -> 11a35b448f83, empty message
INFO  [alembic.runtime.migration] Running upgrade 11a35b448f83 -> 9081f1a90939, empty message
INFO  [alembic.runtime.migration] Running upgrade 9081f1a90939 -> 91b69dfad2f1, empty message
INFO  [alembic.runtime.migration] Running upgrade 91b69dfad2f1 -> 7744c5c16159, empty message
INFO  [alembic.runtime.migration] Running upgrade 7744c5c16159 -> 14167121af69, empty message
INFO  [alembic.runtime.migration] Running upgrade 14167121af69 -> 6e061eb84167, empty message
INFO  [alembic.runtime.migration] Running upgrade 6e061eb84167 -> e9395fe234a4, empty message
INFO  [alembic.runtime.migration] Running upgrade e9395fe234a4 -> 0809266d08ca, empty message
INFO  [alembic.runtime.migration] Running upgrade 0809266d08ca -> f4b8232fa17e, empty message
INFO  [alembic.runtime.migration] Running upgrade f4b8232fa17e -> dbd80d290f04, empty message
INFO  [alembic.runtime.migration] Running upgrade dbd80d290f04 -> 4e4a759ac4b5, empty message
INFO  [alembic.runtime.migration] Running upgrade 4e4a759ac4b5 -> 30c13ca016e4, empty message
INFO  [alembic.runtime.migration] Running upgrade 30c13ca016e4 -> 541ce53ab6e9, empty message
INFO  [alembic.runtime.migration] Running upgrade 541ce53ab6e9 -> 67c61eead8d2, empty message
INFO  [alembic.runtime.migration] Running upgrade 67c61eead8d2 -> 224fd8963462, empty message
INFO  [alembic.runtime.migration] Running upgrade 224fd8963462 -> 92baf66b268b, empty message
INFO  [alembic.runtime.migration] Running upgrade 92baf66b268b -> 497cfd2a02e2, empty message
INFO  [alembic.runtime.migration] Running upgrade 497cfd2a02e2 -> ea30c0b5b2e3, empty message
INFO  [alembic.runtime.migration] Running upgrade ea30c0b5b2e3 -> bfd7b2302903, empty message
INFO  [alembic.runtime.migration] Running upgrade bfd7b2302903 -> 57ef03f3ac34, empty message
INFO  [alembic.runtime.migration] Running upgrade 57ef03f3ac34 -> dd911f880b75, empty message
INFO  [alembic.runtime.migration] Running upgrade dd911f880b75 -> bd05eac83f5f, empty message
INFO  [alembic.runtime.migration] Running upgrade bd05eac83f5f -> b4146f7d5277, empty message
INFO  [alembic.runtime.migration] Running upgrade b4146f7d5277 -> f939d67374e4, empty message
INFO  [alembic.runtime.migration] Running upgrade f939d67374e4 -> de1b457472e0, empty message
INFO  [alembic.runtime.migration] Running upgrade de1b457472e0 -> ae94fe5c4e9f, empty message
INFO  [alembic.runtime.migration] Running upgrade ae94fe5c4e9f -> 026e7a782ed6, empty message
INFO  [alembic.runtime.migration] Running upgrade 026e7a782ed6 -> 925b93d92809, empty message
INFO  [alembic.runtime.migration] Running upgrade 925b93d92809 -> bdf76f4b65a2, empty message
INFO  [alembic.runtime.migration] Running upgrade bdf76f4b65a2 -> a3a7c518ea70, empty message
INFO  [alembic.runtime.migration] Running upgrade a3a7c518ea70 -> a5e3c6693dc6, empty message
INFO  [alembic.runtime.migration] Running upgrade a5e3c6693dc6 -> bf11ab2f0a7a, empty message
INFO  [alembic.runtime.migration] Running upgrade bf11ab2f0a7a -> 1759f73274ee, empty message
INFO  [alembic.runtime.migration] Running upgrade 1759f73274ee -> 552d735a2f1f, empty message
INFO  [alembic.runtime.migration] Running upgrade 552d735a2f1f -> 5cad8fa84386, empty message
INFO  [alembic.runtime.migration] Running upgrade 5cad8fa84386 -> c31cdf879ee3, empty message
INFO  [alembic.runtime.migration] Running upgrade c31cdf879ee3 -> 659d979b64ce, empty message
INFO  [alembic.runtime.migration] Running upgrade 659d979b64ce -> ce15cf3467b4, empty message
INFO  [alembic.runtime.migration] Running upgrade ce15cf3467b4 -> 0e08145f0499, empty message
INFO  [alembic.runtime.migration] Running upgrade 0e08145f0499 -> 00532ac6d4bc, empty message
INFO  [alembic.runtime.migration] Running upgrade 00532ac6d4bc -> f680032cc361, empty message
INFO  [alembic.runtime.migration] Running upgrade f680032cc361 -> 10a7947fda6b, empty message
INFO  [alembic.runtime.migration] Running upgrade 10a7947fda6b -> 4a7d35941602, empty message
INFO  [alembic.runtime.migration] Running upgrade 4a7d35941602 -> cfc013b6461a, empty message
INFO  [alembic.runtime.migration] Running upgrade cfc013b6461a -> b2d51e4d94c8, empty message
INFO  [alembic.runtime.migration] Running upgrade b2d51e4d94c8 -> 749c2b85d20f, empty message
INFO  [alembic.runtime.migration] Running upgrade 749c2b85d20f -> a5b4dc311a89, empty message
INFO  [alembic.runtime.migration] Running upgrade a5b4dc311a89 -> a3c9a43e41f4, empty message
INFO  [alembic.runtime.migration] Running upgrade a3c9a43e41f4 -> 7128f87af701, empty message
INFO  [alembic.runtime.migration] Running upgrade 7128f87af701 -> 270d598c51e3, empty message
INFO  [alembic.runtime.migration] Running upgrade 270d598c51e3 -> b77ab8c47cc7, empty message
INFO  [alembic.runtime.migration] Running upgrade b77ab8c47cc7 -> a2b95b04d1f7, empty message
INFO  [alembic.runtime.migration] Running upgrade a2b95b04d1f7 -> 63fd3b240583, empty message
INFO  [alembic.runtime.migration] Running upgrade 63fd3b240583 -> 95938a93ea14, empty message
INFO  [alembic.runtime.migration] Running upgrade 95938a93ea14 -> b82bcad9accf, empty message
INFO  [alembic.runtime.migration] Running upgrade b82bcad9accf -> 84471852b610, empty message
INFO  [alembic.runtime.migration] Running upgrade 84471852b610 -> b0e9a389939a, empty message
INFO  [alembic.runtime.migration] Running upgrade b0e9a389939a -> 198c3aca9d8d, empty message
INFO  [alembic.runtime.migration] Running upgrade 198c3aca9d8d -> 58ad4df8583e, empty message
INFO  [alembic.runtime.migration] Running upgrade 58ad4df8583e -> 1abfc9e14d7e, empty message
INFO  [alembic.runtime.migration] Running upgrade 1abfc9e14d7e -> 32b00d06d892, empty message
INFO  [alembic.runtime.migration] Running upgrade 32b00d06d892 -> b17afc77ba83, empty message
INFO  [alembic.runtime.migration] Running upgrade b17afc77ba83 -> 54ca2dbf89c0, empty message
INFO  [alembic.runtime.migration] Running upgrade 54ca2dbf89c0 -> eef0c404b531, empty message
INFO  [alembic.runtime.migration] Running upgrade eef0c404b531 -> 84dec6c29c48, empty message
INFO  [alembic.runtime.migration] Running upgrade 84dec6c29c48 -> d0f197979bd9, empty message
INFO  [alembic.runtime.migration] Running upgrade d0f197979bd9 -> 9dc16e591f88, empty message
INFO  [alembic.runtime.migration] Running upgrade 9dc16e591f88 -> ac41029fb329, empty message
INFO  [alembic.runtime.migration] Running upgrade ac41029fb329 -> d1edb3cadec8, empty message
INFO  [alembic.runtime.migration] Running upgrade d1edb3cadec8 -> 623662ea0e7e, empty message
INFO  [alembic.runtime.migration] Running upgrade 623662ea0e7e -> 56c790ec8ab4, empty message
INFO  [alembic.runtime.migration] Running upgrade 56c790ec8ab4 -> c0d91ff18f77, empty message
INFO  [alembic.runtime.migration] Running upgrade c0d91ff18f77 -> 780a8344914b, empty message
INFO  [alembic.runtime.migration] Running upgrade 780a8344914b -> a20aeb9b0eac, empty message
INFO  [alembic.runtime.migration] Running upgrade a20aeb9b0eac -> 0af2c2e286a7, empty message
INFO  [alembic.runtime.migration] Running upgrade 0af2c2e286a7 -> 1919f1859215, empty message
INFO  [alembic.runtime.migration] Running upgrade 1919f1859215 -> f66ca777f409, empty message
INFO  [alembic.runtime.migration] Running upgrade f66ca777f409 -> 7c0dbd378cdb, empty message
INFO  [alembic.runtime.migration] Running upgrade 7c0dbd378cdb -> e99989e6ad56, empty message
INFO  [alembic.runtime.migration] Running upgrade e99989e6ad56 -> 1b54995bc086, empty message
INFO  [alembic.runtime.migration] Running upgrade 1b54995bc086 -> 2779eb90c6c4, empty message
INFO  [alembic.runtime.migration] Running upgrade 2779eb90c6c4 -> 74906d31d994, empty message
INFO  [alembic.runtime.migration] Running upgrade 74906d31d994 -> 85d0655d42c0, empty message
INFO  [alembic.runtime.migration] Running upgrade 85d0655d42c0 -> de7aa5280210, empty message
INFO  [alembic.runtime.migration] Running upgrade de7aa5280210 -> e831a883153a, empty message
INFO  [alembic.runtime.migration] Running upgrade e831a883153a -> d1236c4dff71, empty message
INFO  [alembic.runtime.migration] Running upgrade d1236c4dff71 -> 94f14eb0fe5b, empty message
INFO  [alembic.runtime.migration] Running upgrade 94f14eb0fe5b -> 9d6adad83936, empty message
INFO  [alembic.runtime.migration] Running upgrade 9d6adad83936 -> f398b261d9c6, empty message
INFO  [alembic.runtime.migration] Running upgrade f398b261d9c6 -> 517b79c56088, empty message
INFO  [alembic.runtime.migration] Running upgrade 517b79c56088 -> 48b991e9de06, empty message
INFO  [alembic.runtime.migration] Running upgrade 48b991e9de06 -> e11c3dd48a6f, empty message
INFO  [alembic.runtime.migration] Running upgrade e11c3dd48a6f -> 4912f3bd5ba2, empty message
INFO  [alembic.runtime.migration] Running upgrade 4912f3bd5ba2 -> f5133dc851ee, empty message
INFO  [alembic.runtime.migration] Running upgrade f5133dc851ee -> 5c77d685df87, empty message
INFO  [alembic.runtime.migration] Running upgrade 5c77d685df87 -> 6cc7f073b358, empty message
INFO  [alembic.runtime.migration] Running upgrade 6cc7f073b358 -> 68e2f38e33f4, empty message
INFO  [alembic.runtime.migration] Running upgrade 68e2f38e33f4 -> fc2eb1d7e4fc, empty message
INFO  [alembic.runtime.migration] Running upgrade fc2eb1d7e4fc -> a5e643d562c9, empty message
INFO  [alembic.runtime.migration] Running upgrade a5e643d562c9 -> 29ea13ed76f9, empty message
INFO  [alembic.runtime.migration] Running upgrade 29ea13ed76f9 -> 8e70205a5308, empty message
INFO  [alembic.runtime.migration] Running upgrade 8e70205a5308 -> f3f19998b755, empty message
INFO  [alembic.runtime.migration] Running upgrade f3f19998b755 -> c31a081eab74, empty message
INFO  [alembic.runtime.migration] Running upgrade c31a081eab74 -> 78403c7b8089, empty message
INFO  [alembic.runtime.migration] Running upgrade 78403c7b8089 -> 5662122eac21, empty message
INFO  [alembic.runtime.migration] Running upgrade 5662122eac21 -> 20c738810b1b, empty message
INFO  [alembic.runtime.migration] Running upgrade 20c738810b1b -> dfee471558bd, empty message
INFO  [alembic.runtime.migration] Running upgrade dfee471558bd -> 05e3af59929a, empty message
INFO  [alembic.runtime.migration] Running upgrade 05e3af59929a -> c3470e2d3224, empty message
INFO  [alembic.runtime.migration] Running upgrade c3470e2d3224 -> ffa75d04e6ef, empty message
INFO  [alembic.runtime.migration] Running upgrade ffa75d04e6ef -> 9014cca7097c, empty message
INFO  [alembic.runtime.migration] Running upgrade 9014cca7097c -> d4392342465f, empty message
INFO  [alembic.runtime.migration] Running upgrade d4392342465f -> 424808e1fe49, empty message
INFO  [alembic.runtime.migration] Running upgrade 424808e1fe49 -> 916a5257d18c, empty message
INFO  [alembic.runtime.migration] Running upgrade 916a5257d18c -> 4d3f91ddf3e9, empty message
INFO  [alembic.runtime.migration] Running upgrade 4d3f91ddf3e9 -> d8c55e79da54, empty message
INFO  [alembic.runtime.migration] Running upgrade d8c55e79da54 -> cf1e8c1bc737, empty message
INFO  [alembic.runtime.migration] Running upgrade cf1e8c1bc737 -> 7a105bfc0cd0, empty message
INFO  [alembic.runtime.migration] Running upgrade 7a105bfc0cd0 -> bc75acacc98e, empty message
INFO  [alembic.runtime.migration] Running upgrade bc75acacc98e -> b8b4f9598240, empty message
INFO  [alembic.runtime.migration] Running upgrade b8b4f9598240 -> 5ee767807344, empty message
INFO  [alembic.runtime.migration] Running upgrade 5ee767807344 -> 4913cb3f5a05, empty message
INFO  [alembic.runtime.migration] Running upgrade 4913cb3f5a05 -> 0b1c9ea11aef, empty message
INFO  [alembic.runtime.migration] Running upgrade 0b1c9ea11aef -> 2fbcad5527d7, empty message
INFO  [alembic.runtime.migration] Running upgrade 2fbcad5527d7 -> d750d578b068, empty message
INFO  [alembic.runtime.migration] Running upgrade d750d578b068 -> 2f1b3c759773, empty message
INFO  [alembic.runtime.migration] Running upgrade 2f1b3c759773 -> 99d9e329b27f, empty message
INFO  [alembic.runtime.migration] Running upgrade 99d9e329b27f -> a06066e3fbeb, empty message
INFO  [alembic.runtime.migration] Running upgrade a06066e3fbeb -> d67eab226ecd, empty message
INFO  [alembic.runtime.migration] Running upgrade d67eab226ecd -> bbedc353f90c, empty message
INFO  [alembic.runtime.migration] Running upgrade bbedc353f90c -> 0b9150eb309d, Increase message_id length manually
INFO  [alembic.runtime.migration] Running upgrade 0b9150eb309d -> 6204e57b4bc4, empty message
INFO  [alembic.runtime.migration] Running upgrade 6204e57b4bc4 -> 37feaba7c45d, empty message
INFO  [alembic.runtime.migration] Running upgrade 37feaba7c45d -> ff6c04869029, empty message
INFO  [alembic.runtime.migration] Running upgrade ff6c04869029 -> fdb02bd105a8, empty message
INFO  [alembic.runtime.migration] Running upgrade fdb02bd105a8 -> dd278f96ca83, empty message
INFO  [alembic.runtime.migration] Running upgrade dd278f96ca83 -> 1076b5795b08, empty message
INFO  [alembic.runtime.migration] Running upgrade 1076b5795b08 -> 5639ad89ee50, empty message
INFO  [alembic.runtime.migration] Running upgrade 5639ad89ee50 -> 11ba83e2dd71, empty message
INFO  [alembic.runtime.migration] Running upgrade 11ba83e2dd71 -> ccbfb61eda0d, empty message
INFO  [alembic.runtime.migration] Running upgrade ccbfb61eda0d -> a5013ff0a00a, empty message
INFO  [alembic.runtime.migration] Running upgrade a5013ff0a00a -> e6e8e12f5a13, empty message
INFO  [alembic.runtime.migration] Running upgrade e6e8e12f5a13 -> 9031c9e28510, empty message
INFO  [alembic.runtime.migration] Running upgrade 9031c9e28510 -> b8fd175c084a, empty message
INFO  [alembic.runtime.migration] Running upgrade b8fd175c084a -> e7d7ebcea26c, empty message
INFO  [alembic.runtime.migration] Running upgrade e7d7ebcea26c -> d0ccd9d7ac0c, empty message
INFO  [alembic.runtime.migration] Running upgrade d0ccd9d7ac0c -> 4b483a762fed, empty message
INFO  [alembic.runtime.migration] Running upgrade 4b483a762fed -> ad467baf7ec8, empty message
INFO  [alembic.runtime.migration] Running upgrade ad467baf7ec8 -> d8a3dfe674f2, empty message
INFO  [alembic.runtime.migration] Running upgrade d8a3dfe674f2 -> 3d05479d0d11, empty message
INFO  [alembic.runtime.migration] Running upgrade 3d05479d0d11 -> 753d2ed92d41, empty message
INFO  [alembic.runtime.migration] Running upgrade 753d2ed92d41 -> 698424c429e9, empty message
INFO  [alembic.runtime.migration] Running upgrade 698424c429e9 -> 07b870d7cc86, empty message
INFO  [alembic.runtime.migration] Running upgrade 07b870d7cc86 -> 9282e982bc05, Add block_behaviour setting for user
INFO  [alembic.runtime.migration] Running upgrade 9282e982bc05 -> 5047fcbd57c7, empty message
INFO  [alembic.runtime.migration] Running upgrade 5047fcbd57c7 -> 4729b7096d12, empty message
INFO  [alembic.runtime.migration] Running upgrade 4729b7096d12 -> b500363567e3, Create admin audit log
INFO  [alembic.runtime.migration] Running upgrade b500363567e3 -> 28b9b14c9664, store provider complaints
INFO  [alembic.runtime.migration] Running upgrade 28b9b14c9664 -> 0aaad1740797, store provider complaints
INFO  [alembic.runtime.migration] Running upgrade 0aaad1740797 -> e866ad0e78e1, Add partner tables
INFO  [alembic.runtime.migration] Running upgrade e866ad0e78e1 -> 088f23324464, add flags to the user model
INFO  [alembic.runtime.migration] Running upgrade 088f23324464 -> 2b1d3cd93e4b, update partner_api_token token length
INFO  [alembic.runtime.migration] Running upgrade 2b1d3cd93e4b -> 82d3c7109ffb, partner_user and partner_subscription
INFO  [alembic.runtime.migration] Running upgrade 82d3c7109ffb -> 36646e5dc6d9, make external_user_id non nullable
INFO  [alembic.runtime.migration] Running upgrade 36646e5dc6d9 -> a7bcb872c12a, Add alias transfer token expiration
INFO  [alembic.runtime.migration] Running upgrade a7bcb872c12a -> 673a074e4215, empty message
INFO  [alembic.runtime.migration] Running upgrade 673a074e4215 -> d1fb679f7eec, Add sudo expiration for ApiKeys
INFO  [alembic.runtime.migration] Running upgrade d1fb679f7eec -> bfebc2d5c719, Add state to job
INFO  [alembic.runtime.migration] Running upgrade bfebc2d5c719 -> 516c21ea7d87, empty message
INFO  [alembic.runtime.migration] Running upgrade 516c21ea7d87 -> bd7d032087b2, empty message
INFO  [alembic.runtime.migration] Running upgrade bd7d032087b2 -> b0101a66bb77, Add unsubscribe behaviour
INFO  [alembic.runtime.migration] Running upgrade b0101a66bb77 -> 89081a00fc7d, default_unsub_behaviour
INFO  [alembic.runtime.migration] Running upgrade 89081a00fc7d -> c66f2c5b6cb1, empty message
INFO  [alembic.runtime.migration] Running upgrade c66f2c5b6cb1 -> 9cc0f0712b29, Add api to cookie token
INFO  [alembic.runtime.migration] Running upgrade 9cc0f0712b29 -> bd95b2b4217f, Updated recovery code string length
INFO  [alembic.runtime.migration] Running upgrade bd95b2b4217f -> 2c2093c82bc0, empty message
INFO  [alembic.runtime.migration] Running upgrade 2c2093c82bc0 -> 5f4a5625da66, empty message
INFO  [alembic.runtime.migration] Running upgrade 5f4a5625da66 -> 893c0d18475f, empty message
INFO  [alembic.runtime.migration] Running upgrade 893c0d18475f -> bc496c0a0279, empty message
INFO  [alembic.runtime.migration] Running upgrade bc496c0a0279 -> 2d89315ac650, empty message
INFO  [alembic.runtime.migration] Running upgrade 893c0d18475f -> 01e2997e90d3, empty message
INFO  [alembic.runtime.migration] Running upgrade 01e2997e90d3, 2d89315ac650 -> 2634b41f54db, empty message
INFO  [alembic.runtime.migration] Running upgrade 2634b41f54db -> 01827104004b, empty message
INFO  [alembic.runtime.migration] Running upgrade 01827104004b -> 0a5701a4f5e4, empty message
INFO  [alembic.runtime.migration] Running upgrade 0a5701a4f5e4 -> ec7fdde8da9f, empty message
INFO  [alembic.runtime.migration] Running upgrade ec7fdde8da9f -> 46ecb648a47e, empty message
INFO  [alembic.runtime.migration] Running upgrade 46ecb648a47e -> 4bc54632d9aa, empty message
INFO  [alembic.runtime.migration] Running upgrade 4bc54632d9aa -> 818b0a956205, empty message
INFO  [alembic.runtime.migration] Running upgrade 818b0a956205 -> 52510a633d6f, empty message
INFO  [alembic.runtime.migration] Running upgrade 52510a633d6f -> fa2f19bb4e5a, empty message
INFO  [alembic.runtime.migration] Running upgrade fa2f19bb4e5a -> 06a9a7133445, Create sync_event table
INFO  [alembic.runtime.migration] Running upgrade 06a9a7133445 -> d608b8e48082, empty message
INFO  [alembic.runtime.migration] Running upgrade d608b8e48082 -> 56d08955fcab, add retry count to sync event
INFO  [alembic.runtime.migration] Running upgrade 56d08955fcab -> 1c14339aae90, empty message
INFO  [alembic.runtime.migration] Running upgrade 1c14339aae90 -> 2441b7ff5da9, Custom Domain partner id
INFO  [alembic.runtime.migration] Running upgrade 2441b7ff5da9 -> 88dd7a0abf54, contact.flags and custom_domain.pending_deletion
INFO  [alembic.runtime.migration] Running upgrade 88dd7a0abf54 -> 62afa3a10010, custom domain indices
INFO  [alembic.runtime.migration] Running upgrade 62afa3a10010 -> 91ed7f46dc81, alias_audit_log
INFO  [alembic.runtime.migration] Running upgrade 91ed7f46dc81 -> 7d7b84779837, user_audit_log
INFO  [alembic.runtime.migration] Running upgrade 7d7b84779837 -> 32f25cbf12f6, alias_audit_log_index_created_at
----- step 3: alembic current -----
load config file /root/run_base.env
>>> URL: http://localhost
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
WARNING: Use a temp directory for GNUPGHOME /tmp/gadkrqcvtpspaasvfwwk
Upload files to local dir
>>> init logging <<<
2026-07-08 06:09:03,122 - SL - DEBUG - 128 - "/host_repo/app/utils.py:17" - <module>() -  - load words file: /host_repo/local_data/test_words.txt
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
32f25cbf12f6 (head)
----- step 4: table count (information_schema) -----
 count 
-------
    77
(1 row)

----- step 5: confirm last-created + bookkeeping tables present -----
   table_name    
-----------------
 alembic_version
 alias_audit_log
 user_audit_log
(3 rows)
```
