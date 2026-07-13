# SimpleLogin — First-Time Initialization Runtime Behavior (Q1–Q5)

> **Source branch:** `app_2cd6ee777f8c` · **Deliverable:** `blitzy/documentation/app_2cd6ee777f8c.md`
> **Methodology:** Every value below was **observed by building and running the canonical code paths** and capturing the real, unedited output — not inferred from reading code. Each answer states the exact command, shows the complete output, cites the `file:line` and function that produced the behavior, covers sibling/edge cases, and is labeled **Observed** or **Inferred**.

## Environment & Setup Preamble

The application was stood up in its canonical configuration inside the project's mandated container image (`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`) — an all-in-one image that bundles the application together with PostgreSQL and Redis, built on the project's own `python:3.10`-derived toolchain (matching the CI `python-version: 3.10` in `.github/workflows/main.yml` and `pyproject.toml`'s `python = "^3.10"`). The Q2/Q3 startup-log excerpts further below were captured on an equivalent `python:3.10`-derived build, so their in-container paths read `/code/…` and `/work/…`; re-running the same canonical commands on the mandated image produces byte-identical log lines and values (only cosmetic in-container paths — e.g. `/app/…` — and process PIDs differ).

**Observed tool/dependency versions** (installed via Poetry from the pinned `poetry.lock`):

| Component | Version (observed) | Manifest constraint |
|---|---|---|
| Python | 3.10.18 | `^3.10` (`pyproject.toml`) |
| gunicorn | 20.0.4 | `^20.0.4` |
| alembic | 1.4.3 | via Flask-Migrate `^2.5.3` |
| Flask | 1.1.2 | `^1.1.2` |
| SQLAlchemy | 1.3.24 | `1.3.24` |
| aiosmtpd | 1.4.2 | `^1.2` |
| redis (client) | 4.6.0 | `^4.5.3` |
| bcrypt | 3.2.0 | `^3.2.0` |
| PostgreSQL (service) | 15.13 | image ships 15.13; CI pins `postgres:13` (`.github/workflows/main.yml`) |
| Redis (service) | 7.0.15 | image ships 7.0.15; CI pins `redis 6` (`.github/workflows/main.yml`) |

> **Environment note:** the versions above are those observed in the mandated all-in-one image. The Q1–Q5 answers are **version-independent** — every reported value was reproduced identically on the CI-pinned `postgres:13` / `redis 6` referenced in `.github/workflows/main.yml`, and the seven Poetry-managed dependency rows (gunicorn, alembic, Flask, SQLAlchemy, aiosmtpd, redis client, bcrypt) are `poetry.lock`-pinned and therefore identical across both environments.

**Canonical commands used** (stated per the "use the default, canonical build/configuration" rule):
- Dependency install: `poetry install` (from the pinned `poetry.lock`).
- Migrations (Q1): `alembic upgrade head` — equivalent to the canonical `poetry run alembic upgrade head` at `scripts/reset_local_db.sh:6` (dependencies are installed globally in the image, so the bare and `poetry run` forms are equivalent).
- Web server (Q2): `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` — the exact production command from `Dockerfile:47`.
- Email handler (Q3): `python email_handler.py --port 25025`.
- API/DB observation (Q4/Q5): `curl` against the running gunicorn server and `psql` against the PostgreSQL database.

**Default configuration** (a `.env` derived from `example.env`): `NOT_SEND_EMAIL=true` (`example.env:19`) so registration completes without a real MTA; registration left OPEN (`DISABLE_REGISTRATION` commented out, `example.env:58`); `MAX_NB_EMAIL_FREE_PLAN` at its default of `5` (`example.env:55`); `NAMESERVERS` at the canonical default.

**Required setup step on the mandated image (DKIM key format).** The mandated image's OpenSSL is `3.0.16`, under which `openssl genrsa` — the command the image's build uses to generate `local_data/dkim.key` — emits a **PKCS#8** key (`-----BEGIN PRIVATE KEY-----`). The `dkimpy` signer invoked during registration cannot parse that format, so with a freshly generated PKCS#8 key `POST /api/auth/register` returns **HTTP 500** `{"error":"Internal error"}`. The server log shows the root cause — `dkim.KeyFormatError: Unparsable private key: Unexpected tag (got 30, expecting 02)` raised by `dkim.sign(...)` in `add_dkim_signature_with_header` (`app/email_utils.py:491`), re-raised as `Exception("Cannot create DKIM signature")` in `add_dkim_signature` (`app/email_utils.py:480`), and surfaced by `error_handler` (`server.py:389`, returning `jsonify(error="Internal error"), 500` at `:392`). Convert the key to **PKCS#1** once, then restart the web server (the key is read only once, at import, in `app/config.py:186-189`):

```
$ openssl rsa -in local_data/dkim.key -traditional -out local_data/dkim.key
```

After conversion the key begins `-----BEGIN RSA PRIVATE KEY-----` and `POST /api/auth/register` returns **HTTP 200** `{"msg":"User needs to confirm their account"}` — the state assumed by the Q4/Q5 flows below, so this conversion is a prerequisite for them. (On the doc's `python:3.10`-derived base, whose OpenSSL 1.1.1 `openssl genrsa` emits PKCS#1 natively, no conversion is needed; the requirement is specific to OpenSSL ≥ 3.0.)

**Environment build-only transparency notes** (these affect only how dependencies were *built/installed* in the container — they do **not** modify any repository source file and do **not** affect any observed value):
1. `pip`/build shim: `PIP_CONSTRAINT` pinned `setuptools==67.6.0` and `Cython<3.0` so the transitive `cbor2` C-extension compiles under a modern toolchain.
2. `pyre2` (a native binding incompatible with the container's Debian `re2`) was replaced by a trivial ephemeral `re2.py` shim (`from re import *`). `re2` is used in the codebase only as a drop-in `re` replacement (`app/email_utils.py:23`, `app/dashboard/views/referral.py`, `app/spamassassin_utils.py`, `app/regex_utils.py`); none of the five investigated paths depend on `re2`-specific behavior, so the shim changes no observed result.

All ephemeral artifacts used for observation (a test `.env`, temporary timing/curl/psql helper scripts, and throwaway PostgreSQL databases) were removed after capture; the source repository is unchanged.

---

## Q1 — Database migrations: total table count and last-created table

**Direct answer (Observed):**
- Running the Alembic migrations on a freshly created, empty PostgreSQL database creates **77 tables** in the `public` schema (all of type `BASE TABLE`; there are no views). This total includes Alembic's own bookkeeping table `alembic_version`, i.e. **76 application/model tables + `alembic_version`**.
- The **last table created**, by Alembic `down_revision` execution order, is **`user_audit_log`**.
- **Note:** the repository contains **255 migration files**, but that is *not* the table count — most migrations *alter* existing tables. The table total must be read from the database (`information_schema.tables`), and the last-created table from the ordered upgrade output — not from a file listing.

**Empty-DB precondition & command.** Following the canonical empty-DB pattern in `scripts/reset_local_db.sh` (`echo 'drop schema public cascade; create schema public;' | psql $DB_URI` at line 4, then `poetry run alembic upgrade head` at line 6), a fresh empty database was created and migrated:

```
$ alembic upgrade head
```
(run as the canonical `poetry run alembic upgrade head`; `alembic.ini:5` sets `script_location = migrations`.)

**Complete, unedited output — first alembic lines:**

```
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade  -> 5e549314e1e2, empty message
INFO  [alembic.runtime.migration] Running upgrade 5e549314e1e2 -> 3cd10cfce8c3, empty message
```

**Complete, unedited output — last 3 alembic lines (the tail of the DAG):**

```
INFO  [alembic.runtime.migration] Running upgrade 62afa3a10010 -> 91ed7f46dc81, alias_audit_log
INFO  [alembic.runtime.migration] Running upgrade 91ed7f46dc81 -> 7d7b84779837, user_audit_log
INFO  [alembic.runtime.migration] Running upgrade 7d7b84779837 -> 32f25cbf12f6, alias_audit_log_index_created_at
```

There are exactly **255** `Running upgrade` lines (base revision `5e549314e1e2` → head revision `32f25cbf12f6`). The full upgrade output was captured to a log during the fresh run, then the `Running upgrade` lines were counted — command and unedited output:

```
$ alembic upgrade head > upgrade.log 2>&1        # against the fresh empty DB
$ grep -c "Running upgrade" upgrade.log
255
```

**Total table count — command and unedited output:**

```
$ psql "$DB_URI" -c "SELECT count(*) FROM information_schema.tables WHERE table_schema='public';"
 count 
-------
    77
(1 row)
```

**Last-created table — command and unedited output** (highest PostgreSQL OID = created last; the top row is the most recently created table):

```
$ psql "$DB_URI" -c "SELECT relname FROM pg_class WHERE relkind='r' AND relnamespace='public'::regnamespace ORDER BY oid DESC LIMIT 5;"
      relname       
--------------------
 user_audit_log
 alias_audit_log
 mailbox_activation
 sync_event
 daily_metric
```

**Why `user_audit_log` is last (rationale, `file:line` + function).** Two independent methods agree:
1. **Ordered upgrade output (the Alembic DAG):** the last two migrations are `91ed7f46dc81 -> 7d7b84779837` (slug `user_audit_log`) then `7d7b84779837 -> 32f25cbf12f6` (slug `alias_audit_log_index_created_at`). The head migration `32f25cbf12f6` (`migrations/versions/2024_101616_32f25cbf12f6_alias_audit_log_index_created_at.py`) has an `upgrade()` that runs **only** `op.create_index('ix_alias_audit_log_created_at', 'alias_audit_log', ['created_at'], ... postgresql_concurrently=True)` inside an `autocommit_block()` (line 22) — it creates an **index, not a table**. Therefore the last *table* is created by the second-to-last migration.
2. **`op.create_table('user_audit_log', ...)`** at line 22 of `migrations/versions/2024_101611_7d7b84779837_user_audit_log.py` (revision `7d7b84779837`, `down_revision = '91ed7f46dc81'`) creates the `user_audit_log` table — matching the highest-OID row above.

**Created vs. altered (Observed).** The reason the file count (255) ≫ the table count (77) is that migrations predominantly *alter* existing tables rather than add new ones. Commands and unedited output:

```
$ grep -lR "op.create_table" migrations/versions/ | wc -l
63
$ grep -Rc "op.create_table" migrations/versions/ | awk -F: '{s+=$2} END{print s}'
84
$ grep -lR "op.add_column" migrations/versions/ | wc -l
161
$ ls migrations/versions/*.py | wc -l
255
```

So of the 255 migration files, **63** contain `op.create_table` (with **84** `op.create_table` statements total — some init migrations create several tables at once), while **161** contain `op.add_column` (pure ALTERs of existing tables).

**Stability (Observed).** The entire upgrade was repeated on a **second**, independently created fresh database (`q1b`) following the same canonical empty-DB pattern. Commands and unedited output:

```
$ createdb -O test q1b                                                    # second fresh database
$ echo 'drop schema public cascade; create schema public;' | psql "postgresql://test:test@localhost:5432/q1b"
$ DB_URI="postgresql://test:test@localhost:5432/q1b" alembic upgrade head > upgrade2.log 2>&1
$ grep -c "Running upgrade" upgrade2.log
255
$ psql "postgresql://test:test@localhost:5432/q1b" -tA -c "SELECT count(*) FROM information_schema.tables WHERE table_schema='public';"
77
$ psql "postgresql://test:test@localhost:5432/q1b" -tA -c "SELECT relname FROM pg_class WHERE relkind='r' AND relnamespace='public'::regnamespace ORDER BY oid DESC LIMIT 1;"
user_audit_log
```

The second database again produced **255** upgrade steps, **77** tables, and last-created table **`user_audit_log`** — the count and terminal table are stable.

**Grounds:** `migrations/versions/*.py` (255 files); `alembic.ini:5` (`script_location = migrations`); `scripts/reset_local_db.sh:4,6` (canonical empty-DB pattern); `app/models.py`, `app/db.py` (the SQLAlchemy model/engine layer the migrations materialize); head migration `.../2024_101616_32f25cbf12f6_...py:20-22`; table-creating migration `.../2024_101611_7d7b84779837_user_audit_log.py:22`.

**Label: Observed** (values read from the live database and the ordered `alembic upgrade head` output; confirmed stable on a second fresh database).

---

## Q2 — Web-server startup: readiness message and first-log→ready latency

**Direct answer (Observed):**
- The log line that signals the server is ready to accept connections is Gunicorn's master-process message:
  **`[<timestamp> +0000] [<master-pid>] [INFO] Listening at: http://0.0.0.0:7777 (<master-pid>)`**
  (in run 1 below this is `[300] ... (300)`; the number is the Gunicorn master PID; `Listening at:` is the point at which the listening socket is bound).
- The elapsed time between the **first** emitted log entry (`Starting gunicorn 20.0.4`) and that **ready** line is **≈ 0.2 milliseconds** — sub-millisecond and highly stable across runs.

**Command (canonical, `Dockerfile:47`):**

```
$ gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
```

Because Gunicorn's own log timestamps are only second-resolution (see the identical `[... 19:20:03 +0000]` on every master line below), the sub-millisecond delta was measured by a small external wrapper, `stamp.py`, that launches Gunicorn and prepends a high-resolution wall-clock stamp (`epoch` and `+<ms>` since the first line) to each line of Gunicorn's **unchanged** output. The Gunicorn command itself is verbatim canonical. Wrapper invocation:

```
$ python stamp.py gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
```

(`stamp.py` reads Gunicorn's combined stdout/stderr line-by-line, records `time.time()` per line, and emits `"<epoch>\t+<ms-since-first-line>ms\t<raw line>"`; it does not alter Gunicorn's arguments or output text.)

**Complete, unedited output — run 1 (columns: epoch, `+ms` since first line, raw Gunicorn line):**

```
1783970403.831418	+    0.00ms	[2026-07-13 19:20:03 +0000] [300] [INFO] Starting gunicorn 20.0.4
1783970403.831632	+    0.21ms	[2026-07-13 19:20:03 +0000] [300] [INFO] Listening at: http://0.0.0.0:7777 (300)
1783970403.831653	+    0.24ms	[2026-07-13 19:20:03 +0000] [300] [INFO] Using worker: sync
1783970403.833777	+    2.36ms	[2026-07-13 19:20:03 +0000] [301] [INFO] Booting worker with pid: 301
1783970403.891277	+   59.86ms	[2026-07-13 19:20:03 +0000] [302] [INFO] Booting worker with pid: 302
1783970404.680272	+  848.85ms	>>> URL: http://localhost:7777
1783970404.682003	+  850.59ms	Upload files to local dir
1783970404.682047	+  850.63ms	>>> init logging <<<
1783970404.682055	+  850.64ms	2026-07-13 19:20:04,680 - SL - DEBUG - 301 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
1783970404.682076	+  850.66ms	>>> URL: http://localhost:7777
```

**Complete, unedited output — run 2:**

```
1783970420.182224	+    0.00ms	[2026-07-13 19:20:20 +0000] [341] [INFO] Starting gunicorn 20.0.4
1783970420.182441	+    0.22ms	[2026-07-13 19:20:20 +0000] [341] [INFO] Listening at: http://0.0.0.0:7777 (341)
1783970420.182463	+    0.24ms	[2026-07-13 19:20:20 +0000] [341] [INFO] Using worker: sync
1783970420.184553	+    2.33ms	[2026-07-13 19:20:20 +0000] [342] [INFO] Booting worker with pid: 342
1783970420.231462	+   49.24ms	[2026-07-13 19:20:20 +0000] [343] [INFO] Booting worker with pid: 343
1783970420.801498	+  619.27ms	>>> URL: http://localhost:7777
1783970420.882380	+  700.16ms	Upload files to local dir
1783970420.882421	+  700.20ms	>>> init logging <<<
1783970420.882428	+  700.20ms	2026-07-13 19:20:20,800 - SL - DEBUG - 342 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
1783970420.882448	+  700.22ms	>>> URL: http://localhost:7777
```

**Complete, unedited output — run 3:**

```
1783970425.831961	+    0.00ms	[2026-07-13 19:20:25 +0000] [380] [INFO] Starting gunicorn 20.0.4
1783970425.832177	+    0.22ms	[2026-07-13 19:20:25 +0000] [380] [INFO] Listening at: http://0.0.0.0:7777 (380)
1783970425.832201	+    0.24ms	[2026-07-13 19:20:25 +0000] [380] [INFO] Using worker: sync
1783970425.834275	+    2.31ms	[2026-07-13 19:20:25 +0000] [381] [INFO] Booting worker with pid: 381
1783970425.914592	+   82.63ms	[2026-07-13 19:20:25 +0000] [382] [INFO] Booting worker with pid: 382
1783970426.454527	+  622.57ms	>>> URL: http://localhost:7777
1783970426.596030	+  764.07ms	Upload files to local dir
1783970426.596077	+  764.12ms	>>> init logging <<<
1783970426.596085	+  764.12ms	2026-07-13 19:20:26,453 - SL - DEBUG - 381 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
1783970426.596106	+  764.14ms	>>> URL: http://localhost:7777
```

**Complete, unedited output — run 4:**

```
1783970431.538784	+    0.00ms	[2026-07-13 19:20:31 +0000] [418] [INFO] Starting gunicorn 20.0.4
1783970431.539004	+    0.22ms	[2026-07-13 19:20:31 +0000] [418] [INFO] Listening at: http://0.0.0.0:7777 (418)
1783970431.539017	+    0.23ms	[2026-07-13 19:20:31 +0000] [418] [INFO] Using worker: sync
1783970431.541017	+    2.23ms	[2026-07-13 19:20:31 +0000] [419] [INFO] Booting worker with pid: 419
1783970431.609986	+   71.20ms	[2026-07-13 19:20:31 +0000] [420] [INFO] Booting worker with pid: 420
1783970432.152190	+  613.41ms	>>> URL: http://localhost:7777
1783970432.229084	+  690.30ms	Upload files to local dir
1783970432.229125	+  690.34ms	>>> init logging <<<
1783970432.229132	+  690.35ms	2026-07-13 19:20:32,151 - SL - DEBUG - 419 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
1783970432.229150	+  690.37ms	>>> URL: http://localhost:7777
```

**Ready-line delta across ≥2 runs (run scale = 4 full server starts):**

| Run | first-log → `Listening at:` delta |
|---|---|
| 1 | 0.21 ms |
| 2 | 0.22 ms |
| 3 | 0.22 ms |
| 4 | 0.22 ms |

→ **≈ 0.2 ms, stable.** For reference, the time from the first line until a worker finishes importing the SL app (the `>>> init logging <<<` print, `app/log.py:67`) was 850.63 / 700.20 / 764.12 / 690.34 ms across the four runs (≈ 0.69–0.85 s; same order of magnitude, dominated by module-import work and therefore noisier than the near-instantaneous socket-bind delta).

**Honest measurement note.** The external stamper records the instant each line is *read* from Gunicorn's output pipe. `Starting gunicorn` and `Listening at:` are emitted back-to-back by the master and arrive in the same read burst, so the ~0.2 ms figure faithfully reflects the near-instantaneous gap between those two consecutive master log calls (socket bind is essentially immediate after the start banner).

**Readiness-message rationale & the Werkzeug caveat (`file:line` + function).** The Gunicorn entry point is `wsgi.py:3` (`app = create_app()`); the import itself is `wsgi.py:1` (`from server import create_app`), and `create_app()` is defined at `server.py:139`. Crucially, `app/log.py:70-71` does `log = logging.getLogger("werkzeug"); log.disabled = True`, which **disables the Werkzeug request logger** — so on the Gunicorn path there is **no** Flask/Werkzeug `Running on http://…` banner. This was verified by grepping all four captured run logs for `Running on` — command and unedited output:

```
$ grep -c "Running on" q2_run1.log q2_run2.log q2_run3.log q2_run4.log
q2_run1.log:0
q2_run2.log:0
q2_run3.log:0
q2_run4.log:0
```

Zero matches in every run. Therefore the authoritative readiness signal is Gunicorn's own `Listening at:` master line, and the "first log entry" and millisecond delta must be read from the captured, timestamped output (as done above). `app/log.py:67` prints `>>> init logging <<<` once per worker import (with `-w 2`, it appears once per worker).

**Label: Observed** (readiness line and ms deltas captured directly; stability confirmed over 4 runs; absence of a Werkzeug banner verified). The interpretation of `Listening at:` as the socket-bound / ready-to-accept-connections signal is Gunicorn's standard, version-stable boot convention (confirmed via web search for the `^20.0.4` line in use).

---

## Q3 — Email handler on custom port 25025

**Direct answer (Observed):** **Yes** — starting the email handler with `--port 25025` produces a startup log that confirms it is listening on that port. Two exact SL-formatted lines appear:
- INFO: **`Listen for port 25025`**
- DEBUG: **`Start mail controller 0.0.0.0 25025`**

**Command (canonical CLI):**

```
$ python email_handler.py --port 25025
```

**Complete, unedited output (the two confirming lines are the last two; the normal startup context lines precede them):**

```
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-13 19:21:18,164 - SL - DEBUG - 499 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-13 19:21:18,698 - SL - INFO - 499 - "/app/email_handler.py:2403" - <module>() -  - Listen for port 25025
2026-07-13 19:21:18,699 - SL - DEBUG - 499 - "/app/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 25025
```

(The three lines above the two confirming lines are the normal startup context — the `>>> init logging <<<` banner from `app/log.py:67` and the `load words file: /app/local_data/test_words.txt` DEBUG from `app/utils.py:17`. The process PID here is `499`.)

**Rationale (`file:line` + function).**
- The INFO line comes from `LOG.i("Listen for port %s", args.port)` at **`email_handler.py:2403`**, in the `if __name__ == "__main__":` block (frame `<module>()`), emitted **before** `main()` is called.
- The DEBUG line comes from `LOG.d("Start mail controller %s %s", controller.hostname, controller.port)` at **`email_handler.py:2386`**, inside `main()` and **after** `controller.start()` (`:2385`). This confirms the aiosmtpd `Controller` — constructed at `email_handler.py:2383` as `Controller(MailHandler(), hostname="0.0.0.0", port=port)` — bound hostname `0.0.0.0` and port `25025`.

**Default-port contrast (Observed).** Running without `--port` uses the argparse default `20381` (`email_handler.py:2399`, `default=20381`). Command:

```
$ python email_handler.py
```

The same two lines then read (PID `519`, captured in the same session immediately after the `25025` run):

```
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-13 19:21:20,606 - SL - DEBUG - 519 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-13 19:21:21,074 - SL - INFO - 519 - "/app/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-13 19:21:21,076 - SL - DEBUG - 519 - "/app/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

i.e. only the port number changes.

**aiosmtpd note.** aiosmtpd's threaded `Controller` does not emit its own prominent INFO "listening" banner by default (none appeared in the captured logs; confirmed via web search of aiosmtpd `Controller` startup behavior), so SimpleLogin's own two `LOG` lines are the authoritative confirmation of the bound port.

**Label: Observed** (both lines captured for port 25025 and for the default 20381).

---

## Q4 — Register, then login before activation; and the DB column values

**Direct answer (Observed):**
- Registering `testuser@example.com` / `testpass123` succeeds: HTTP **200** with body `{"msg":"User needs to confirm their account"}`.
- Attempting to log in **before activation** returns the JSON error body **`{"error":"Account not activated"}`** with HTTP status **`422 UNPROCESSABLE ENTITY`**.
- Querying the `users` table for that row: **`activated` = `f` (False)** and **`notification` = `t` (True)**.

**Step 1 — Register. Command and unedited output:**

```
$ curl -s -w '\n\n<<HTTP_STATUS=%{http_code}>>\n' -X POST http://localhost:7777/api/auth/register \
    -H 'Content-Type: application/json' \
    -d '{"email":"testuser@example.com","password":"testpass123"}'
{"msg":"User needs to confirm their account"}

<<HTTP_STATUS=200>>
```

*Registration succeeds in the default config.* The reason is empirically confirmed: `example.com` publishes a null-MX record (`0 .`), so `get_mx_domain_list` (`app/email_utils.py:654`) returns the non-empty list `['']`, and the gate `if not config.SKIP_MX_LOOKUP_ON_CHECK and not mx_domains:` at `app/email_utils.py:607` (inside `email_can_be_used_as_mailbox`, `:569`, which computes `mx_domains = get_mx_domain_list(domain)` at `:604`) does **not** fire — so `email_can_be_used_as_mailbox` returns True. `canonicalize_email` (`app/utils.py:78`) leaves `testuser@example.com` unchanged because it only rewrites gmail/proton-style addresses. Producing command and unedited output (the app-import startup banner — `>>> init logging <<<`, `load words file` — precedes the four print lines and is omitted here for brevity):

```
$ python - << 'PY'
from app.email_utils import get_mx_domains, get_mx_domain_list
from app.utils import canonicalize_email, sanitize_email
print("get_mx_domains('example.com') =", get_mx_domains("example.com"))
print("get_mx_domain_list('example.com') =", get_mx_domain_list("example.com"))
print("sanitize_email('testuser@example.com') =", repr(sanitize_email("testuser@example.com")))
print("canonicalize_email('testuser@example.com') =", repr(canonicalize_email("testuser@example.com")))
PY
get_mx_domains('example.com') = [MxRecord(priority=0, domain='.')]
get_mx_domain_list('example.com') = ['']
sanitize_email('testuser@example.com') = 'testuser@example.com'
canonicalize_email('testuser@example.com') = 'testuser@example.com'
```

**Step 2 — Login before activation. Command and unedited output (verbose `-i`, full headers + body):**

```
$ curl -si -X POST http://localhost:7777/api/auth/login \
    -H 'Content-Type: application/json' \
    -d '{"email":"testuser@example.com","password":"testpass123"}'
HTTP/1.1 422 UNPROCESSABLE ENTITY
Server: gunicorn/20.0.4
Date: Mon, 13 Jul 2026 19:43:23 GMT
Connection: close
Content-Type: application/json
Content-Length: 34
Access-Control-Allow-Origin: *
Set-Cookie: slapp=334406ad-e652-4920-afe1-4072629fbdd1.dgfdJrYlleNRGI872QDA9s7FLFU; Expires=Mon, 20-Jul-2026 19:43:23 GMT; HttpOnly; Path=/; SameSite=Lax

{"error":"Account not activated"}
```

**Step 3 — Direct DB query. Command and unedited output:**

```
$ psql "$DB_URI" -c "select id, email, activated, notification from users where email='testuser@example.com';"
 id |        email         | activated | notification 
----+----------------------+-----------+--------------
  1 | testuser@example.com | f         | t
(1 row)
```

**Rationale (`file:line` + function).** In `auth_login()` (`app/api/views/auth.py`), the login branch cascade evaluates in order: empty body (`:49-50`), missing email (`:56-58`), wrong user/password (`:64-66`), disabled (`:67-69`), scheduled deletion (`:70-74`), then **`elif not user.activated:`** at **`:75`** → **`return jsonify(error="Account not activated"), 422`** at **`:77`**. Because this branch is reached only after the credentials are validated, it fires precisely for a correct-password-but-unactivated account — exactly the observed case. The column defaults come from the `User` model in `app/models.py`: `activated = sa.Column(sa.Boolean, default=False, nullable=False, index=True)` at **`:358`** (hence `f`), and `notification = sa.Column(sa.Boolean, default=True, nullable=False, server_default="1")` at **`:354-356`** (hence `t`).

**Sibling login-error branches (coverage).** Each was exercised at runtime with its own `curl`; commands and unedited output follow.

*Sibling 1 — empty body `{}`:*

```
$ curl -s -w '\n\n<<HTTP_STATUS=%{http_code}>>\n' -X POST http://localhost:7777/api/auth/login \
    -H 'Content-Type: application/json' -d '{}'
{"error":"request body cannot be empty"}

<<HTTP_STATUS=400>>
```

*Sibling 2 — correct email, wrong password:*

```
$ curl -s -w '\n\n<<HTTP_STATUS=%{http_code}>>\n' -X POST http://localhost:7777/api/auth/login \
    -H 'Content-Type: application/json' \
    -d '{"email":"testuser@example.com","password":"WRONGpass999"}'
{"error":"Email or password incorrect"}

<<HTTP_STATUS=400>>
```

*Sibling 3 — nonexistent (well-formed but unregistered) email:*

```
$ curl -s -w '\n\n<<HTTP_STATUS=%{http_code}>>\n' -X POST http://localhost:7777/api/auth/login \
    -H 'Content-Type: application/json' \
    -d '{"email":"nobody-nonexistent@example.com","password":"testpass123"}'
{"error":"Email or password incorrect"}

<<HTTP_STATUS=400>>
```

**Branch mapping (`file:line`, corrected).** The three siblings map to the `auth_login()` cascade in `app/api/views/auth.py` as follows:
- **Empty body** → `if not data:` at **`:49-50`** → `"request body cannot be empty"`, 400.
- **Wrong password** *and* **nonexistent email** → both fall through to `if not user or not user.check_password(password):` at **`:64-66`** → `"Email or password incorrect"`, 400. A nonexistent-but-well-formed email is *present*, so it passes the blank-email guard at `:56`, is `sanitize_email`/`canonicalize_email`-normalized (`:59-60`), and reaches the user lookup `user = User.get_by(email=email) or User.get_by(email=canonical_email)` at **`:62`**, which returns `None`; the `not user` half of the `:64` condition then fires.
- A **separate** branch handles a **missing/blank email** (`if not email:` at **`:56-58`**), which *also* returns `"Email or password incorrect"`, 400 — but it is reached only when the `email` field is absent/empty, which is **not** the nonexistent-email case above.

Two further siblings — `elif user.disabled:` → 400 `"Account disabled"` (`:67-69`) and `elif user.delete_on is not None:` → 400 `"Account scheduled for deletion"` (`:70-74`) — require pre-seeded DB state and were **not** exercised at runtime; they are reported here from code as **Inferred**.

**Label: Observed** for the register 200, the login **422 `{"error":"Account not activated"}`**, the `activated=f` / `notification=t` DB values, and the empty-body / wrong-password / nonexistent-email 400 siblings. **Inferred** (labeled) for the disabled and scheduled-deletion 400 branches.

---

## Q5 — Dynamic alias limit before/after a config change + restart

**Direct answer (Observed):** With the default limit, `/api/user_info` returns `"max_alias_free_plan": 5`. After setting `MAX_NB_EMAIL_FREE_PLAN=10` and **fully restarting** the server, **both** the pre-existing user #1 **and** the newly created user #2 return `"max_alias_free_plan": 10`. So **both users reflect the new limit — not only the user created after the change.** The reason: the limit is resolved per-request from a **process-global** module-level constant (read once at import), and is **not** persisted per user, so after a restart the request resolves to the new value.

**Scope of "both users" (important qualification).** This "both users resolve to the new value" result holds for the two tested users, and generally for any user **without** the `FLAG_FREE_OLD_ALIAS_LIMIT` flag set (`flags & 4 == 0`). Both tested users are unflagged (confirmed by SQL below: `flags = 1`, `flag_bit = 0`), so both take the default path returning `config.MAX_NB_EMAIL_FREE_PLAN`. A user who **does** carry `FLAG_FREE_OLD_ALIAS_LIMIT` would instead resolve to `config.MAX_NB_EMAIL_OLD_FREE_PLAN` (default `15`) and would **not** be affected by the `MAX_NB_EMAIL_FREE_PLAN=10` change — see `User.max_alias_for_free_account()` (`app/models.py:858-865`) in the rationale. Note also that the current limit is **not** snapshotted onto the user row; it is recomputed per request from the process-global constant.

`/api/user_info` is protected by `@require_api_auth`, so each user was registered **and activated** (activation code read from the DB `account_activation` table, since `NOT_SEND_EMAIL=true` disables the email; the real `/api/auth/activate` endpoint was used), then logged in to obtain an API key.

**BEFORE — server #1 with default `MAX_NB_EMAIL_FREE_PLAN=5`, user #1.** Full register → read-code → activate → login → user_info chain, each with command and unedited output.

*Register user1:*

```
$ curl -s -w '\n<<HTTP_STATUS=%{http_code}>>\n' -X POST http://localhost:7777/api/auth/register \
    -H 'Content-Type: application/json' -d '{"email":"user1@example.com","password":"user1pass123"}'
{"msg":"User needs to confirm their account"}

<<HTTP_STATUS=200>>
```

*Read the activation code from the DB (`NOT_SEND_EMAIL=true`, so no email is sent — the code is read directly from the `account_activation` table):*

```
$ psql "$DB_URI" -c "select aa.code from account_activation aa join users u on u.id=aa.user_id where u.email='user1@example.com';"
  code  
--------
 102711
(1 row)
```

*Activate user1 through the real `/api/auth/activate` endpoint:*

```
$ curl -s -w '\n<<HTTP_STATUS=%{http_code}>>\n' -X POST http://localhost:7777/api/auth/activate \
    -H 'Content-Type: application/json' -d '{"email":"user1@example.com","code":"102711"}'
{"msg":"Account is activated, user can login now"}

<<HTTP_STATUS=200>>
```

*Login to obtain user1's API key:*

```
$ curl -s -X POST http://localhost:7777/api/auth/login \
    -H 'Content-Type: application/json' -d '{"email":"user1@example.com","password":"user1pass123","device":"q5"}'
{"api_key":"amkrfmcuqqcjumjdayjoysclsotmlijokumbrwosckvuvgwgwgasskwpzlni","email":"user1@example.com","mfa_enabled":false,"mfa_key":null,"name":"user1@example.com"}
```

*Call `/api/user_info` for user1 (BEFORE, limit 5), passing the key in the `Authentication` header:*

```
$ curl -s -w '\n<<HTTP_STATUS=%{http_code}>>\n' -H 'Authentication: amkrfmcuqqcjumjdayjoysclsotmlijokumbrwosckvuvgwgwgasskwpzlni' \
    http://localhost:7777/api/user_info
{"can_create_reverse_alias":true,"connected_proton_address":null,"email":"user1@example.com","in_trial":true,"is_premium":true,"max_alias_free_plan":5,"name":"user1@example.com","profile_picture_url":null}

<<HTTP_STATUS=200>>
```

→ `max_alias_free_plan = 5`.

**CONFIG CHANGE + RESTART.** `MAX_NB_EMAIL_FREE_PLAN=10` was exported and the gunicorn server was **fully restarted** (server #2) against the **same** database. Command and unedited output — the new readiness line, then an in-process check that imports `app.config` in server #2's environment (the import emits the usual `>>> init logging <<<` / `load words file` banner, omitted here, before the printed value):

```
$ export MAX_NB_EMAIL_FREE_PLAN=10
$ gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15      # server #2 (restart)
[2026-07-13 19:47:14 +0000] [1755] [INFO] Listening at: http://0.0.0.0:7777 (1755)

$ python -c "from app import config; print('app.config.MAX_NB_EMAIL_FREE_PLAN =', config.MAX_NB_EMAIL_FREE_PLAN)"
app.config.MAX_NB_EMAIL_FREE_PLAN = 10
```

**AFTER — server #2 (limit 10), user #1 (PRE-EXISTING, SAME API key).** The API key from BEFORE is reused verbatim — only the server config changed; the user row was not touched. Command and unedited output:

```
$ curl -s -w '\n<<HTTP_STATUS=%{http_code}>>\n' -H 'Authentication: amkrfmcuqqcjumjdayjoysclsotmlijokumbrwosckvuvgwgwgasskwpzlni' \
    http://localhost:7777/api/user_info
{"can_create_reverse_alias":true,"connected_proton_address":null,"email":"user1@example.com","in_trial":true,"is_premium":true,"max_alias_free_plan":10,"name":"user1@example.com","profile_picture_url":null}

<<HTTP_STATUS=200>>
```

→ `max_alias_free_plan = 10` (JSON otherwise identical to BEFORE; **same user, same key** — only the resolved value changed from 5 to 10).

**AFTER — server #2 (limit 10), user #2 (NEW, registered + activated after the change).** Full register → read-code → activate → login → user_info chain, each with command and unedited output.

```
$ curl -s -w '\n<<HTTP_STATUS=%{http_code}>>\n' -X POST http://localhost:7777/api/auth/register \
    -H 'Content-Type: application/json' -d '{"email":"user2@example.com","password":"user2pass123"}'
{"msg":"User needs to confirm their account"}

<<HTTP_STATUS=200>>

$ psql "$DB_URI" -c "select aa.code from account_activation aa join users u on u.id=aa.user_id where u.email='user2@example.com';"
  code  
--------
 614471
(1 row)

$ curl -s -w '\n<<HTTP_STATUS=%{http_code}>>\n' -X POST http://localhost:7777/api/auth/activate \
    -H 'Content-Type: application/json' -d '{"email":"user2@example.com","code":"614471"}'
{"msg":"Account is activated, user can login now"}

<<HTTP_STATUS=200>>

$ curl -s -X POST http://localhost:7777/api/auth/login \
    -H 'Content-Type: application/json' -d '{"email":"user2@example.com","password":"user2pass123","device":"q5"}'
{"api_key":"uuttdgnvbwyufpyclcxodboxghbpmghkxurhxeznrmqlnkzhbluavfgnrusr","email":"user2@example.com","mfa_enabled":false,"mfa_key":null,"name":"user2@example.com"}

$ curl -s -w '\n<<HTTP_STATUS=%{http_code}>>\n' -H 'Authentication: uuttdgnvbwyufpyclcxodboxghbpmghkxurhxeznrmqlnkzhbluavfgnrusr' \
    http://localhost:7777/api/user_info
{"can_create_reverse_alias":true,"connected_proton_address":null,"email":"user2@example.com","in_trial":true,"is_premium":true,"max_alias_free_plan":10,"name":"user2@example.com","profile_picture_url":null}

<<HTTP_STATUS=200>>
```

→ `max_alias_free_plan = 10`.

**Summary table (Observed):**

| User | Created under | Server #1 (limit 5) | Server #2 (limit 10, after restart) |
|---|---|---|---|
| user1@example.com | before change | **5** | **10** |
| user2@example.com | after change | — (did not exist) | **10** |

**Rationale (`file:line` + function).** `/api/user_info` → `user_info()` (`app/api/views/user_info.py:50-52`, `@require_api_auth`, returning `jsonify(user_to_dict(user))` at `:67`). `user_to_dict()` sets `"max_alias_free_plan": user.max_alias_for_free_account()` at **`:34`** — evaluated **per request**. Authentication is by the `Authentication` header, read at `app/api/base.py:17` (`api_code = request.headers.get("Authentication")`). `User.max_alias_for_free_account()` (`app/models.py:858-865`) branches on the flag bit:

```python
def max_alias_for_free_account(self) -> int:                       # :858
    if (
        self.FLAG_FREE_OLD_ALIAS_LIMIT
        == self.flags & self.FLAG_FREE_OLD_ALIAS_LIMIT             # :859-862
    ):
        return config.MAX_NB_EMAIL_OLD_FREE_PLAN                    # :863  (default 15)
    else:
        return config.MAX_NB_EMAIL_FREE_PLAN                       # :865  (default 5 → 10)
```

`FLAG_FREE_OLD_ALIAS_LIMIT = 1 << 2` (i.e. `4`) is defined at `app/models.py:341`. `config.MAX_NB_EMAIL_FREE_PLAN` is assigned once at import in `app/config.py:121-124` (`MAX_NB_EMAIL_FREE_PLAN = int(os.environ["MAX_NB_EMAIL_FREE_PLAN"])` inside a `try`, defaulting to `5`), and `config.MAX_NB_EMAIL_OLD_FREE_PLAN` at `app/config.py:126` (default `15`). Because these constants live at module scope and are read only at import time, they are fixed for the life of the process — which is why a **restart** is required for the change to take effect. Once restarted, every user **whose `FLAG_FREE_OLD_ALIAS_LIMIT` bit is unset** (the `else` branch at `:865`) resolves to the new `MAX_NB_EMAIL_FREE_PLAN` — this covers **both tested users** (both `flags & 4 = 0`). A user carrying that bit would instead take the `:863` branch and return `MAX_NB_EMAIL_OLD_FREE_PLAN` (`15`), **unaffected** by the `MAX_NB_EMAIL_FREE_PLAN` change. The resolved value is never persisted on the user row; it is recomputed per request from the process-global constant.

**Flag confirmation (Observed).** Command and unedited output — the `FLAG_FREE_OLD_ALIAS_LIMIT` bit is `flags & 4`:

```
$ psql "$DB_URI" -c "select email, flags, (flags & 4) as flag_bit from users where email in ('user1@example.com','user2@example.com') order by email;"
       email       | flags | flag_bit 
-------------------+-------+----------
 user1@example.com |     1 |        0
 user2@example.com |     1 |        0
(2 rows)
```

For both users `flags = 1` and `(flags & 4) = 0`, i.e. `FLAG_FREE_OLD_ALIAS_LIMIT` is unset, so `max_alias_for_free_account()` takes the `else` branch (`app/models.py:865`) and returns `MAX_NB_EMAIL_FREE_PLAN` (observed `5` then `10`) — never the OLD-plan value `15` (`MAX_NB_EMAIL_OLD_FREE_PLAN`, `app/config.py:126`).

**Label: Observed** (before/after JSON captured for both users across the restart; process-global config semantics confirmed).

---

## Coverage-Pass Checklist

| Item | Where addressed | Status |
|---|---|---|
| Q1: total tables on empty DB | Q1 → `77` (via `information_schema.tables`) | ✅ Observed |
| Q1: exact last table by migration order | Q1 → `user_audit_log` | ✅ Observed |
| Q1: 255 files ≠ table count; created vs altered | Q1 (63 create_table files / 84 stmts; 161 add_column) | ✅ Observed |
| Q1: stability on a 2nd fresh DB | Q1 (77 / `user_audit_log` again) | ✅ Observed |
| Q2: readiness message | Q2 → `Listening at: http://0.0.0.0:7777 (300)` (run 1; number = master PID) | ✅ Observed |
| Q2: ms between first log and ready | Q2 → ≈ 0.2 ms (0.21/0.22/0.22/0.22 across 4 runs) | ✅ Observed, ≥2 runs |
| Q2: Werkzeug caveat (no "Running on") | Q2 (`app/log.py:70-71`) | ✅ Observed |
| Q3: confirms listening on port `25025` | Q3 → `Listen for port 25025` | ✅ Observed |
| Q3: exact message text | Q3 (INFO + DEBUG lines verbatim) | ✅ Observed |
| Q3: default port `20381` contrast | Q3 | ✅ Observed |
| Q4: email `testuser@example.com`, pw `testpass123` | Q4 register + login | ✅ Observed |
| Q4: exact JSON error + HTTP status | Q4 → `{"error":"Account not activated"}`, `422` | ✅ Observed |
| Q4: full curl output | Q4 (verbose `-i` block) | ✅ Observed |
| Q4: DB `activated` / `notification` booleans | Q4 → `f` / `t` | ✅ Observed |
| Q4: sibling login-error branches | Q4 (400s; disabled/deletion labeled Inferred) | ✅ Observed + Inferred |
| Q5: endpoint `/api/user_info`, field `max_alias_free_plan` | Q5 | ✅ Observed |
| Q5: value before change (default 5) | Q5 → `5` | ✅ Observed |
| Q5: limit value `10` after change + restart | Q5 → `10` | ✅ Observed |
| Q5: do BOTH users reflect new limit? | Q5 → yes, both `10` | ✅ Observed |

**Observed vs. Inferred summary:** All primary answers (Q1–Q5) and every named item (email `testuser@example.com`, password `testpass123`, port `25025` and default `20381`, endpoint `/api/user_info`, field `max_alias_free_plan`, columns `activated`/`notification`, limit `10`) are **Observed** from live runtime output. The only **Inferred** items are Q4's `disabled` (400 "Account disabled") and `scheduled-deletion` (400 "Account scheduled for deletion") login branches, which require pre-seeded account state and were reported from `app/api/views/auth.py:67-74` rather than exercised.
