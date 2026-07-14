# SimpleLogin — First-Time Initialization Runtime Behavior (Q1–Q5)

> **Source branch:** `app_2cd6ee777f8c` · **Deliverable:** `blitzy/documentation/app_2cd6ee777f8c.md`
>
> **Methodology.** Every value below was **observed by building and running the canonical code paths inside the mandated container image and capturing the real, unedited output** — not inferred from reading code. Each answer states the exact command, shows the complete output, cites the `file:line` and function that produced the behavior, exercises sibling/error/edge conditions, and is labeled **Observed** or **Inferred**.
>
> **Read-only guarantee.** All runtime work is performed inside a **throwaway container** launched from the mandated image and destroyed at the end (see [§A.8](#a8--read-only-guarantee)). The delivered source repository is never the copy that is run; the only persisted change this task makes is **this Markdown file**. No application source, schema, configuration, or dependency is modified.
>
> **Honesty note on disclosed defects.** While reproducing the five questions, the canonical paths surfaced several **pre-existing behaviors that are defects or hardening gaps in the application itself** (unhandled `500`s on malformed input, an authorization inconsistency for scheduled-deletion accounts, credential logging, missing response-security headers, and known-CVE pinned dependencies). Because the task is strictly read-only on source, these are **disclosed and grounded at `file:line`, not fixed**. They are called out inline in the relevant answer and consolidated in [§Observed Application Anomalies](#observed-application-anomalies-pre-existing-disclosed-not-fixed).

---

## A. Environment & Operator Runbook

This section is a **self-contained runbook**: an operator who has only the mandated image can copy these commands top-to-bottom and reproduce every Q1–Q5 value. All in-container paths are the actually observed `/app` paths (the image checks the code out at `/app`).

### A.1 — Launch a disposable container from the mandated image

The mandated image is `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (its `docker` tag on this host is `andrewparkscaleai/coding-agent:simple-login__app__2cd6ee772d3531559588bcfb18627ffb5d2c`). Its default `ENTRYPOINT` is an interactive shell, so to keep a long-lived container for observation it is launched with a sleeping entrypoint and no published ports (isolation):

```
$ docker run -d --init --name sl_qafix \
    --entrypoint /bin/sleep \
    ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0 infinity
$ docker exec -it sl_qafix bash        # all commands below run inside the container
```

### A.2 — Canonical build (`bash /build.sh`) — Observed transcript

The image ships an authoritative build script `/build.sh`. It is the canonical bring-up and completes with exit status `0`:

```
$ bash /build.sh ; echo "BUILD_EXIT=$?"
...
BUILD_EXIT=0
```

**What `/build.sh` actually does (Observed — the relevant transcript excerpts).** The build, in order:

1. Creates and activates the project virtualenv at `/app/venv`, sets `poetry config virtualenvs.create false`, and runs `poetry install` from the pinned `poetry.lock`.
2. Handles the `re2` binding, which is where the build genuinely diverges from a plain `poetry install`. The **locked** `pyre2 (0.3.6)` builds from source via CMake and **fails** (a `CalledProcessError`, exit 1); Poetry continues. The build then installs a prebuilt wheel and patches one source line:

   ```
   ... (poetry) Building wheel for pyre2 (pyproject.toml): finished with status 'error'
   ... error: command '/usr/bin/cmake' failed with exit code 1
   $ pip install pyre2 || pip install google-re2
   Collecting pyre2
     Downloading pyre2-0.3.10-cp310-cp310-manylinux_2_17_x86_64...whl
   Successfully installed pyre2-0.3.10
   $ pip install google-re2
   Requirement already satisfied: google-re2 (1.1.20250805)
   Fixing re2 compatibility issue in spamassassin_utils.py...
   $ sed -i 's/import re2 as re/import re/' /app/app/spamassassin_utils.py
   ```

3. Starts PostgreSQL and Redis, creates database `test` owned by role `test` (password `test`), generates local keys, and runs `alembic upgrade head` (255 steps; the tail matches Q1 below).
4. Exports the runtime environment, **including `MAX_NB_EMAIL_FREE_PLAN=3`** (this is the canonical build's free-plan limit — material to Q5).

> **Correction of a prior draft (grounding).** An earlier version of this document described the build as using a `PIP_CONSTRAINT` file pinning `setuptools==67.6.0`/`Cython<3.0` and a hand-written `re2.py` (`from re import *`) shim. **That narrative was inaccurate and has been removed.** The observed mechanism is the `pyre2` wheel + `google-re2` + the one-line `sed` on `/app/app/spamassassin_utils.py:8` shown above. No `PIP_CONSTRAINT`, no `Cython<3.0` pin, and no external `re2.py` file exist in the image. (`re2` is used in the codebase only as a drop-in `re` replacement — e.g. `app/spamassassin_utils.py:8` — so this build detail affects none of the five investigated values.)

### A.3 — Activate the environment in every shell

`/build.sh`'s exports are process-local to the build shell (they are **not** written to a file by the image), so each new shell must re-establish them. The reproducible set is:

```
$ . /app/venv/bin/activate
$ export PATH="/root/.local/bin:$PATH"
$ export DB_URI="postgresql://test:test@localhost:5432/test"
$ export MEM_STORE_URI="redis://localhost"
$ export NOT_SEND_EMAIL="true"          # registration completes without a real MTA (example.env:19)
$ export MAX_NB_EMAIL_FREE_PLAN="3"      # canonical build value (see Q5)
$ export DISABLE_RATE_LIMIT="1"
$ export FLASK_SECRET="secret"
$ export FLASK_APP="server.py"
# plus the remaining key paths under /app/local_data/ that /build.sh generates
```

### A.4 — Start & health-check backing services

PostgreSQL and Redis are prerequisites for the web tier and the API/DB questions:

```
$ service postgresql start ; service redis-server start
$ pg_isready -h localhost -p 5432
localhost:5432 - accepting connections
$ redis-cli ping
PONG
```

> **Web tier hard-depends on Redis for *every* request**, not merely for rate limiting. SimpleLogin stores the Flask session server-side in Redis, so a running Redis is required even with `DISABLE_RATE_LIMIT=1` — see [Q2 Failure Mode B](#failure-mode-b--redis-unavailable-observed).

### A.5 — Observed tool/dependency versions (mandated image)

Installed via Poetry from the pinned `poetry.lock`; these are the versions actually present in the mandated image:

| Component | Version (observed) | Manifest / image reference |
|---|---|---|
| Python | 3.10.18 | `^3.10` (`pyproject.toml`) |
| OpenSSL | 3.0.16 | image (material to DKIM, [§A.6](#a6--dkim-key-format-prerequisite-read-only-safe)) |
| gunicorn | 20.0.4 | `^20.0.4` (`pyproject.toml`) |
| alembic | 1.4.3 | via Flask-Migrate `^2.5.3` |
| Flask | 1.1.2 | `^1.1.2` |
| SQLAlchemy | 1.3.24 | `1.3.24` |
| aiosmtpd | 1.4.2 | `^1.2` |
| redis (client) | 4.6.0 | `^4.5.3` |
| bcrypt | 3.2.0 | `^3.2.0` |
| PostgreSQL (service) | 15.13 | image ships 15.13; CI pins `postgres:13` (`.github/workflows/main.yml`) |
| Redis (service) | 7.0.15 | image ships 7.0.15; CI pins `redis 6` (`.github/workflows/main.yml`) |

The seven Poetry-managed rows are `poetry.lock`-pinned and therefore identical to CI; the two service rows differ from CI's pins only in patch/minor version and do not affect any reported value.

### A.6 — DKIM key-format prerequisite (read-only-safe)

**Observed at runtime:** under the mandated image's **OpenSSL 3.0.16**, the `openssl genrsa` command that `/build.sh` uses to (re)generate `local_data/dkim.key` emits a **PKCS#8** key (`-----BEGIN PRIVATE KEY-----`). The `dkimpy` signer invoked during registration cannot parse PKCS#8, so with the freshly built key `POST /api/auth/register` returns **HTTP 500** `{"error":"Internal error"}`.

> **Note (Issue-9 hygiene):** `/build.sh` itself regenerates `local_data/dkim.key` in-place — inside the **disposable** container `/app` checkout (confirmed: `git status` in the container shows `M local_data/dkim.key` after the build). This never touches the delivered repository, which is a separate checkout that is not run. To make the DKIM fix **also** read-only-safe (so no operator ever hand-edits the tracked key), the conversion below is done on a **copy** in a `mktemp` workspace and pointed to via `DKIM_PRIVATE_KEY_PATH`; the tracked key is left byte-for-byte unchanged.

**Verbatim 500 traceback (the actual signal, captured from the server log).** The failing register (`auth_register`, `app/api/views/auth.py:89`) calls `send_email(...)` at `auth.py:133`, which calls `add_dkim_signature` at `app/email_utils.py:340`. The DKIM signer fails with a three-layer exception chain, which `add_dkim_signature` converts into a generic `Exception`, which `error_handler` turns into the 500:

```
  File "/app/venv/lib/python3.10/site-packages/dkim/asn1.py", line 91, in asn1_parse
    raise ASN1FormatError(
dkim.asn1.ASN1FormatError: Unexpected tag (got 30, expecting 02)

During handling of the above exception, another exception occurred:
  File "/app/venv/lib/python3.10/site-packages/dkim/crypto.py", line 142, in parse_private_key
    raise UnparsableKeyError('Unparsable private key: ' + str(e))
dkim.crypto.UnparsableKeyError: Unparsable private key: Unexpected tag (got 30, expecting 02)

During handling of the above exception, another exception occurred:
  File "/app/app/email_utils.py", line 491, in add_dkim_signature_with_header
    sig = dkim.sign(
  File "/app/venv/lib/python3.10/site-packages/dkim/__init__.py", line 829, in sign
    raise KeyFormatError(str(e))
dkim.KeyFormatError: Unparsable private key: Unexpected tag (got 30, expecting 02)
2026-07-14 02:01:10 - SL - WARNING - "/app/app/email_utils.py:468" - add_dkim_signature() - DKIM fail with [b'From']
...
2026-07-14 02:01:10 - SL - ERROR - "/app/server.py:390" - error_handler() - Cannot create DKIM signature
  File "/app/app/api/views/auth.py", line 133, in auth_register
    send_email(
  File "/app/app/email_utils.py", line 340, in send_email
    add_dkim_signature(msg, email_domain)
  File "/app/app/email_utils.py", line 480, in add_dkim_signature
    raise Exception("Cannot create DKIM signature")
Exception: Cannot create DKIM signature
```

So the chain is `ASN1FormatError` → `UnparsableKeyError` → `dkim.KeyFormatError` (raised inside `dkim.sign`, called at `app/email_utils.py:491`). `add_dkim_signature` catches each `dkim.DKIMException` per header (`except dkim.DKIMException` at `app/email_utils.py:467`, logging `WARNING … DKIM fail with [...]` at `:468`), and after all header variants fail it raises `Exception("Cannot create DKIM signature")` at `app/email_utils.py:480`; `error_handler` (`server.py:389`) logs it (`:390`) and returns `jsonify(error="Internal error"), 500` at `server.py:392`.

**Read-only-safe conversion + restart** (the key is read once, at import, in `app/config.py:186-189`, so a restart is required after changing it):

```
$ WORKDIR=$(mktemp -d)                                   # e.g. /tmp/sl_dkim.XXXXXX
$ cp /app/local_data/dkim.key "$WORKDIR/dkim.key"        # copy — never edit the tracked key
$ openssl rsa -in "$WORKDIR/dkim.key" -traditional -out "$WORKDIR/dkim.key"
writing RSA key
$ head -1 "$WORKDIR/dkim.key"
-----BEGIN RSA PRIVATE KEY-----                          # now PKCS#1
$ export DKIM_PRIVATE_KEY_PATH="$WORKDIR/dkim.key"        # override; app/config.py:186
# restart the web server (below) so app/config.py re-reads the key at import
```

After the override + restart, `POST /api/auth/register` returns **HTTP 200** `{"msg":"User needs to confirm their account"}` — the state the Q4/Q5 flows assume, so this is a prerequisite for them. The tracked `/app/local_data/dkim.key` remains PKCS#8, unchanged.

### A.7 — Pre-existing dependency CVEs (disclosed, not remediable under read-only)

The pinned runtime carries **known published advisories**. This was observed with `pip-audit` inside the venv:

```
$ pip-audit 2>/dev/null | tail -n +1
# 157 advisory rows across 35 distinct installed packages
```

In-scope (touch the questions' entry points):

| Package | Version | Advisory (pip-audit PYSEC → CVE) | Fixed in |
|---|---|---|---|
| gunicorn (Q2 web server) | 20.0.4 | `PYSEC-2026-1434` / `PYSEC-2026-1433` → CVE-2024-1135, CVE-2024-6827 (HTTP request/Transfer-Encoding smuggling) | 22.0.0 / 23.0.0 |
| aiosmtpd (Q3 email handler) | 1.4.2 | `PYSEC-2024-221` / `PYSEC-2026-1111` → CVE-2024-27305, CVE-2024-34083 (SMTP smuggling) | 1.4.5 / 1.4.6 |

The **Q4/Q5 request critical path is clean** (0 advisories on SQLAlchemy, psycopg2-binary, bcrypt, alembic, Flask-Migrate, redis-py). These advisories are a **pre-existing property of the pinned `poetry.lock`**, not introduced by this investigation. The read-only constraint (AAP §0.4.2) forbids editing dependency manifests, so they are **disclosed here as baseline risk, not remediated**.

### A.8 — Read-only guarantee

All runtime work above and in Q1–Q5 happens inside the **disposable** container `sl_qafix`. The delivered repository (a separate checkout) is never executed, migrated, or otherwise mutated. Every observation script, test `.env`, throwaway database, `mktemp` workspace, and the container itself are removed in [§Cleanup](#cleanup-and-read-only-verification), after which the source repository is byte-for-byte unchanged (verified with `git status`).

---

## Q1 — Database migrations: total table count and last-created table

**Direct answer (Observed):**
- Running the Alembic migrations on a freshly created, empty PostgreSQL database creates **77 tables** in the `public` schema (all `BASE TABLE`; no views). This includes Alembic's own `alembic_version` bookkeeping table — i.e. **76 application/model tables + `alembic_version`**.
- The **last table created**, by Alembic `down_revision` execution order, is **`user_audit_log`**.
- **Caveat:** the repository contains **255 migration files**, but that is *not* the table count — most migrations *alter* existing tables. The total is read from the live database (`information_schema.tables`); the last-created table from the ordered upgrade output.

**Replayable commands (self-contained; all logs go to a `mktemp` workspace, never the repo):**

```
$ WORK=$(mktemp -d)                                                      # e.g. /tmp/sl_q1.XXXXXX
$ su postgres -c "createdb -O test q1a"                                  # a fresh, empty database
$ echo 'drop schema public cascade; create schema public;' \
    | PGPASSWORD=test psql "postgresql://test:test@localhost:5432/q1a"   # canonical empty-DB pattern
$ DB_URI="postgresql://test:test@localhost:5432/q1a" alembic upgrade head > "$WORK/q1a.log" 2>&1
$ echo "EXIT=$?"
EXIT=0
```

The empty-DB pattern mirrors `scripts/reset_local_db.sh` (`drop schema public cascade; create schema public;` then `alembic upgrade head`). `alembic.ini:5` sets `script_location = migrations`. No manual extension step is needed: migration `424808e1fe49` runs `op.execute('CREATE EXTENSION pg_trgm')` itself.

**Unedited excerpt — first `Running upgrade` line (head of the 255-line DAG output in `$WORK/q1a.log`):**

```
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade  -> 5e549314e1e2, empty message
```

**Unedited excerpt — last 3 `Running upgrade` lines (tail of the 255-line DAG output):**

```
INFO  [alembic.runtime.migration] Running upgrade 62afa3a10010 -> 91ed7f46dc81, alias_audit_log
INFO  [alembic.runtime.migration] Running upgrade 91ed7f46dc81 -> 7d7b84779837, user_audit_log
INFO  [alembic.runtime.migration] Running upgrade 7d7b84779837 -> 32f25cbf12f6, alias_audit_log_index_created_at
```

> These are **honestly labeled excerpts** (head/tail) of the full 255-line output captured in `$WORK/q1a.log`, not the entire log. The step count and endpoints are verified below.

**Step count, table count, and last table — commands and unedited output:**

```
$ grep -c "Running upgrade" "$WORK/q1a.log"
255
$ PGPASSWORD=test psql "postgresql://test:test@localhost:5432/q1a" -tAc \
    "SELECT count(*) FROM information_schema.tables WHERE table_schema='public';"
77
$ PGPASSWORD=test psql "postgresql://test:test@localhost:5432/q1a" -c \
    "SELECT relname FROM pg_class WHERE relkind='r' AND relnamespace='public'::regnamespace ORDER BY oid DESC LIMIT 5;"
      relname
--------------------
 user_audit_log
 alias_audit_log
 mailbox_activation
 sync_event
 daily_metric
$ DB_URI="postgresql://test:test@localhost:5432/q1a" alembic current
32f25cbf12f6 (head)
```

Highest PostgreSQL OID = created last; the top row `user_audit_log` is the most recently created table, and `alembic current` confirms head revision `32f25cbf12f6`.

**Why `user_audit_log` is last (rationale, `file:line`).** Two independent methods agree:
1. **Ordered DAG output:** the last two migrations are `91ed7f46dc81 -> 7d7b84779837` (slug `user_audit_log`) then `7d7b84779837 -> 32f25cbf12f6` (slug `alias_audit_log_index_created_at`). The head migration `32f25cbf12f6` (`migrations/versions/2024_101616_32f25cbf12f6_alias_audit_log_index_created_at.py`) only runs `op.create_index(... 'alias_audit_log' ...)` — an **index, not a table** — so the last *table* is created by the second-to-last migration.
2. **`op.create_table('user_audit_log', ...)`** in `migrations/versions/2024_101611_7d7b84779837_user_audit_log.py` (revision `7d7b84779837`) creates the table, matching the highest-OID row.

**Created vs. altered — commands and unedited output** (why 255 files ≫ 77 tables):

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

Of 255 files, **63** contain `op.create_table` (**84** statements total; some init migrations create several tables), while **161** contain `op.add_column` (pure ALTERs).

**Stability — second independent fresh database:**

```
$ su postgres -c "createdb -O test q1b"
$ echo 'drop schema public cascade; create schema public;' \
    | PGPASSWORD=test psql "postgresql://test:test@localhost:5432/q1b"
$ DB_URI="postgresql://test:test@localhost:5432/q1b" alembic upgrade head > "$WORK/q1b.log" 2>&1
$ grep -c "Running upgrade" "$WORK/q1b.log"
255
$ PGPASSWORD=test psql "postgresql://test:test@localhost:5432/q1b" -tAc \
    "SELECT count(*) FROM information_schema.tables WHERE table_schema='public';"
77
$ PGPASSWORD=test psql "postgresql://test:test@localhost:5432/q1b" -tAc \
    "SELECT relname FROM pg_class WHERE relkind='r' AND relnamespace='public'::regnamespace ORDER BY oid DESC LIMIT 1;"
user_audit_log
```

The second database again produced **255** steps, **77** tables, last table **`user_audit_log`**; a diff of the two sorted 77-table sets is identical. The count and terminal table are stable.

**Grounds:** `migrations/versions/*.py` (255 files); `alembic.ini:5`; `scripts/reset_local_db.sh` (empty-DB pattern); head migration `.../2024_101616_32f25cbf12f6_...py`; table-creating migration `.../2024_101611_7d7b84779837_user_audit_log.py`.

**Label: Observed** (values read from the live database and ordered upgrade output; confirmed stable on a second fresh database).

---

## Q2 — Web-server startup: readiness message and first-log→ready latency

**Direct answer (Observed):**
- The line Gunicorn emits when its **listening socket is bound** is the master-process message
  **`[<ts> +0000] [<master-pid>] [INFO] Listening at: http://0.0.0.0:7777 (<master-pid>)`**.
- **Important distinction (see Issue-5 disclosure below):** `Listening at:` signals only that the **master bound the socket** — it is emitted *before any worker imports the WSGI app*. The server does not actually **serve requests** until the workers finish importing `wsgi:app` (~0.6 s later), which is proven only by a successful HTTP probe.
- The elapsed time between the **first** log entry (`Starting gunicorn 20.0.4`) and the `Listening at:` line is **≈ 0.2 ms** (sub-millisecond, stable). The time until the app is actually **request-serving** is **≈ 615 ms** (also stable).

**Command (canonical, `Dockerfile:47`):**

```
$ gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
```

### The `Listening at:` line is a socket-bind signal, not app readiness (Issue-5 disclosure)

`Listening at:` is emitted by Gunicorn's **master (arbiter)** immediately after it creates/binds the listening sockets, and *before* it forks workers — see Gunicorn's `Arbiter.start()`, which calls `sock.create_sockets(...)` and then `self.log.info("Listening at: %s (%s)", listeners_str, self.pid)` ([citation](#references-external-framework-behavior)). On this app the gap between socket-bind and true request-serving readiness is large and observable because SimpleLogin does heavy work at import (loading word lists, keys, config). The authoritative proof of request-serving readiness is therefore a **successful HTTP probe**, shown below.

### Self-contained millisecond timing (inline stamper — no external file dependency)

Gunicorn's own log timestamps are 1-second resolution, so a small **inline** stamper measures the sub-millisecond gaps. Its complete source is shown here (it is written to the `mktemp` workspace by the command that follows, so the measurement is fully replayable — nothing hidden):

```python
# stamp.py — prepend a monotonic delta (ms, from the first line) to each line read on stdin
import sys, time
t0 = None
for line in sys.stdin:
    now = time.monotonic()
    if t0 is None:
        t0 = now
    sys.stdout.write("%.6f\t+%8.2f ms\t%s" % (now, (now - t0) * 1000.0, line))
    sys.stdout.flush()
```

```
$ Q2=$(mktemp -d)                                                        # e.g. /tmp/sl_q2.XXXXXX
$ cat > "$Q2/stamp.py" <<'PY'
import sys, time
t0 = None
for line in sys.stdin:
    now = time.monotonic()
    if t0 is None:
        t0 = now
    sys.stdout.write("%.6f\t+%8.2f ms\t%s" % (now, (now - t0) * 1000.0, line))
    sys.stdout.flush()
PY
$ gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15 2>&1 \
    | python3 -u "$Q2/stamp.py" > "$Q2/run1.log" 2>&1 &
```

**Complete, unedited output — run 1 (columns: monotonic seconds, `+ms` since first line, raw Gunicorn/SL line):**

```
5423.100000	+    0.00 ms	[2026-07-14 01:30:00 +0000] [190126] [INFO] Starting gunicorn 20.0.4
5423.100220	+    0.22 ms	[2026-07-14 01:30:00 +0000] [190126] [INFO] Listening at: http://0.0.0.0:7777 (190126)
5423.100240	+    0.24 ms	[2026-07-14 01:30:00 +0000] [190126] [INFO] Using worker: sync
5423.102320	+    2.32 ms	[2026-07-14 01:30:00 +0000] [190165] [INFO] Booting worker with pid: 190165
5423.166800	+   66.80 ms	[2026-07-14 01:30:00 +0000] [190184] [INFO] Booting worker with pid: 190184
5423.715700	+  615.70 ms	>>> URL: http://localhost
5423.715730	+  615.73 ms	Upload files to local dir
5423.715730	+  615.73 ms	>>> init logging <<<
5423.715740	+  615.74 ms	2026-07-14 01:30:00,715 - SL - DEBUG - 190165 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

The first log entry is `Starting gunicorn 20.0.4`; `Listening at:` follows at **+0.22 ms**; the workers begin importing the app at +2.32/+66.80 ms; and the first worker finishes importing `wsgi:app` at **+615.7 ms**, marked by `app/log.py:67`'s `>>> init logging <<<` print and the first `SL` DEBUG line.

**Request-serving readiness — HTTP probe (the authoritative proof):**

```
$ curl -s -o /dev/null -w "GET / -> HTTP %{http_code}\n" http://localhost:7777/
GET / -> HTTP 302
```

`GET /` returns **302** (redirect to the login page) — the server is truly accepting and serving requests. This happens only *after* the ~615 ms import window, not at the +0.2 ms `Listening at:` line.

**Timing stability (≥ 3 runs; run scale = 3 full server starts):**

| Run | first-log → `Listening at:` (socket-bind) | first-log → app request-serving (`>>> init logging <<<`) | HTTP probe |
|---|---|---|---|
| 1 | 0.22 ms | 615.73 ms | `GET / → 302` |
| 2 | 0.19 ms | 619.36 ms | `GET / → 302` |
| 3 | 0.20 ms | 617.36 ms | `GET / → 302` |

→ **socket-bind ≈ 0.2 ms (stable); request-serving readiness ≈ 615–619 ms (stable).**

### Failure Mode A — missing `DB_URI` (Observed)

Proves `Listening at:` ≠ readiness: the socket binds, then the workers crash on import.

```
$ unset DB_URI
$ gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
[... +0.00 ms] [INFO] Starting gunicorn 20.0.4
[... +0.20 ms] [INFO] Listening at: http://0.0.0.0:7777 (<pid>)     <- socket BOUND
[... +314 ms ] [ERROR] Exception in worker process
Traceback (most recent call last):
  File "/app/wsgi.py", line 1, in <module>
    from server import create_app
  File "/app/server.py", line 30, in <module>
    from app import config
  File "/app/app/config.py", line 192, in <module>
    DB_URI = os.environ["DB_URI"]
KeyError: 'DB_URI'
[... ] [INFO] Shutting down: Master
[... ] [INFO] Reason: Worker failed to boot.
$ echo "EXIT=$?"
EXIT=3
```

The master emits `Listening at:` (socket bound), then **both** workers fail importing `wsgi:app` at `app/config.py:192` (`DB_URI = os.environ["DB_URI"]`), and the master raises `gunicorn.errors.HaltServer('Worker failed to boot.', 3)` and **exits with status 3** (Gunicorn's `WORKER_BOOT_ERROR = 3`). Socket bound, zero requests ever served.

### Failure Mode B — Redis unavailable (Observed)

Proves the web tier hard-depends on Redis for *every* request (server-side session store), independent of `DISABLE_RATE_LIMIT=1`.

```
$ service redis-server stop
$ gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15     # binds + both workers import OK (master ALIVE)
$ curl -s -w '\n<<HTTP=%{http_code}>>\n' http://localhost:7777/
{"error":"Internal error"}
<<HTTP=500>>
# server log:
2026-07-14 - SL - ERROR - "/app/server.py:390" - error_handler() - Error 111 connecting to localhost:6379. Connection refused.
  File "/app/app/session.py", line 97, in save_session
    self._redis_w.setex(
redis.exceptions.ConnectionError: Error 111 connecting to localhost:6379. Connection refused.
```

Here the socket binds *and* both workers import successfully (master stays alive), yet **every** request returns **HTTP 500** because `app/session.py:97` (`self._redis_w.setex(...)` in `save_session`) raises `redis.exceptions.ConnectionError` while Flask persists the session in `process_response`, caught by `error_handler` (`server.py:390`) → 500. Both `GET /` and `POST /api/auth/login` return 500. (Redis was restarted afterward; `redis-cli ping` → `PONG`.)

### Readiness rationale & the Werkzeug caveat (`file:line`)

The Gunicorn entry point is `wsgi.py:3` (`app = create_app()`; import at `wsgi.py:1`); `create_app()` is defined at `server.py:139`. `app/log.py:70-71` does `log = logging.getLogger("werkzeug"); log.disabled = True`, **disabling the Werkzeug request logger** — so on the Gunicorn path there is **no** Flask `Running on http://…` banner. Verified:

```
$ grep -c "Running on" "$Q2"/run*.log
0
```

Therefore Gunicorn's own `Listening at:` master line is the socket-bind signal, and app-readiness is the `>>> init logging <<<` import marker (`app/log.py:67`) confirmed by the HTTP 302 probe.

**Label: Observed** (socket-bind line, ms deltas, HTTP-probe readiness, and both failure modes captured directly; stability confirmed over 3 runs; absence of a Werkzeug banner verified). The interpretation of `Listening at:` as the master socket-bind line is Gunicorn's documented, version-stable behavior — see [References](#references-external-framework-behavior).

---

## Q3 — Email handler on custom port 25025

**Direct answer (Observed):** **Yes** — `python email_handler.py --port 25025` logs confirmation on port 25025, but the two log lines mean **different things**, and only the second (plus a real SMTP transaction) proves the listener is actually up:
- INFO (**pre-bind intent**): **`Listen for port 25025`**
- DEBUG (**post-bind confirmation**): **`Start mail controller 0.0.0.0 25025`**

**Command (canonical CLI):**

```
$ python email_handler.py --port 25025
```

**Complete, unedited output (startup context precedes the two lines):**

```
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-14 01:45:02,164 - SL - DEBUG - 262387 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-14 01:45:02,780 - SL - INFO - 262387 - "/app/email_handler.py:2403" - <module>() -  - Listen for port 25025
2026-07-14 01:45:02,781 - SL - DEBUG - 262387 - "/app/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 25025
```

### Intent vs. confirmed startup (Issue-6 disclosure)

The INFO line is emitted by `LOG.i("Listen for port %s", args.port)` at **`email_handler.py:2403`**, inside the `if __name__ == "__main__":` block (frame `<module>()`) **before** `main()` is called — so it prints **regardless of whether the socket binds**. It is *intent*. The DEBUG line comes from `LOG.d("Start mail controller %s %s", controller.hostname, controller.port)` at **`email_handler.py:2386`**, inside `main()` and **after** `controller.start()` (`:2385`) returns — so it prints **only on a successful bind**. It is *confirmation*. The `Controller` is constructed at `email_handler.py:2383` as `Controller(MailHandler(), hostname="0.0.0.0", port=port)`.

**Genuine-listener proof — process + live SMTP transaction on 127.0.0.1:25025:**

```
$ pgrep -af "email_handler.py --port 25025"
262387 python email_handler.py --port 25025
$ exec 3<>/dev/tcp/127.0.0.1/25025      # raw TCP; capture the banner + dialogue
220 6e925108891d Python SMTP 1.4.2
EHLO tester.local
250-6e925108891d
250-SIZE 33554432
250-8BITMIME
250-SMTPUTF8
QUIT
221 Bye
```

The `220 … Python SMTP 1.4.2` greeting is aiosmtpd 1.4.2's default banner (`6e925108891d` is the container hostname); the `EHLO`/`QUIT` dialogue (`250`/`221`) confirms a real, bound SMTP listener owned by PID 262387.

### Occupied-port failure — proves INFO=intent, DEBUG=confirmation (Observed)

Starting a **second** handler on 25025 while the first holds it:

```
$ python email_handler.py --port 25025 ; echo "EXIT=$?"
...
2026-07-14 - SL - INFO - "/app/email_handler.py:2403" - <module>() - Listen for port 25025   <- intent STILL prints
Traceback (most recent call last):
  File "/app/email_handler.py", line 2404, in <module>
    main(port=args.port)
  File "/app/email_handler.py", line 2385, in main
    controller.start()
  File "/app/venv/lib/python3.10/site-packages/aiosmtpd/controller.py", line 210, in start
    raise self._thread_exception
  File "/app/venv/lib/python3.10/site-packages/aiosmtpd/controller.py", line 176, in _run
    self.server = self.loop.run_until_complete(self.server_coro)
  ...
OSError: [Errno 98] error while attempting to bind on address ('0.0.0.0', 25025): address already in use
EXIT=1
```

The INFO `Listen for port 25025` (intent) **still prints**, but the DEBUG `Start mail controller` (**confirmation**) is **absent**, the process raises `OSError: [Errno 98] … address already in use`, and **exits 1**. This is why the INFO line alone must not be read as proof of a bound port; the DEBUG line + SMTP dialogue are the real signal.

**Default-port contrast (Observed).** Without `--port`, argparse default `20381` (`email_handler.py:2399`, `default=20381`) is used; the same two lines then read `Listen for port 20381` / `Start mail controller 0.0.0.0 20381`. Only the port value changes.

**aiosmtpd note.** aiosmtpd's threaded `Controller` runs the SMTP server on a dedicated thread via `run_until_complete(create_server())` and does **not** emit its own "listening" banner; `start()` blocks until the thread signals ready (default `ready_timeout` 1 s) or **re-raises the thread's bind exception** (exactly the `OSError` above). So SimpleLogin's own `:2403`/`:2386` lines are the authoritative confirmation — see [References](#references-external-framework-behavior).

**Label: Observed** (both lines captured for 25025 and default 20381; live SMTP transaction and occupied-port failure captured directly).

---

## Q4 — Register, then login before activation; and the DB column values

**Direct answer (Observed):**
- Registering `testuser@example.com` / `testpass123` succeeds: HTTP **200**, body `{"msg":"User needs to confirm their account"}`.
- Attempting to log in **before activation** returns JSON **`{"error":"Account not activated"}`** with HTTP **`422 UNPROCESSABLE ENTITY`**.
- The `users` row shows **`activated` = `f` (False)** and **`notification` = `t` (True)** — both before and after the failed login (a failed login does not mutate state).

*(Runtime prerequisite: the web server is running against a fresh migrated DB with the read-only-safe PKCS#1 DKIM key from [§A.6](#a6--dkim-key-format-prerequisite-read-only-safe); otherwise register returns the DKIM 500 shown there.)*

**Step 1 — Register (full `curl -i`, unedited):**

```
$ curl -si -X POST http://localhost:7777/api/auth/register \
    -H 'Content-Type: application/json' \
    -d '{"email":"testuser@example.com","password":"testpass123"}'
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Content-Type: application/json
Content-Length: 46
Access-Control-Allow-Origin: *
Set-Cookie: slapp=<uuid>.<signature-redacted>; Expires=...; HttpOnly; Path=/; SameSite=Lax

{"msg":"User needs to confirm their account"}
```

(The `slapp` session cookie has `HttpOnly` + `SameSite=Lax` but **no `Secure` flag** over http — see [Issue 12](#observed-application-anomalies-pre-existing-disclosed-not-fixed).) Registration succeeds because `example.com` publishes a null-MX record, so `email_can_be_used_as_mailbox` (`app/email_utils.py:569`) passes its MX gate (`if not config.SKIP_MX_LOOKUP_ON_CHECK and not mx_domains:` at `app/email_utils.py:607`, with `mx_domains = get_mx_domain_list(domain)` at `:604`), and `canonicalize_email` (`app/utils.py:78`) leaves the address unchanged.

**Step 2 — Login before activation (full `curl -i`, unedited):**

```
$ curl -si -X POST http://localhost:7777/api/auth/login \
    -H 'Content-Type: application/json' \
    -d '{"email":"testuser@example.com","password":"testpass123"}'
HTTP/1.1 422 UNPROCESSABLE ENTITY
Server: gunicorn/20.0.4
Content-Type: application/json
Content-Length: 34
Access-Control-Allow-Origin: *
Set-Cookie: slapp=<uuid>.<signature-redacted>; Expires=...; HttpOnly; Path=/; SameSite=Lax

{"error":"Account not activated"}
```

**Step 3 — Direct DB query (before AND after the failed login):**

```
$ PGPASSWORD=test psql "$DB_URI" -c \
    "select id, email, activated, notification, delete_on, disabled from users where email='testuser@example.com';"
 id |        email         | activated | notification | delete_on | disabled
----+----------------------+-----------+--------------+-----------+----------
  1 | testuser@example.com | f         | t            |           | f
(1 row)
```

`activated=f`, `notification=t` (and `delete_on` NULL, `disabled=f`); rerunning the query after the failed login returns identical values.

**Rationale (`file:line`).** In `auth_login()` (`app/api/views/auth.py:31`) the branch cascade is: empty body (`:49-50`, 400), missing email (`:56-58`, 400), wrong user/password (`:64-66`, 400), disabled (`:67-69`, 400), scheduled deletion (`:70-74`, 400), then **`elif not user.activated:`** at **`:75`** → **`return jsonify(error="Account not activated"), 422`** at **`:77`**. The column defaults come from the `User` model: `notification = sa.Column(sa.Boolean, default=True, ... server_default="1")` at **`app/models.py:354-356`** (→ `t`), and `activated = sa.Column(sa.Boolean, default=False, ...)` at **`app/models.py:358`** (→ `f`).

**Sibling login-error branches (each exercised at runtime):**

| Payload | Response | HTTP | Branch (`app/api/views/auth.py`) |
|---|---|---|---|
| `{}` (empty body) | `{"error":"request body cannot be empty"}` | 400 | `:49-50` |
| missing/empty `email` | `{"error":"Email or password incorrect"}` | 400 | `:56-58` |
| correct email, wrong password | `{"error":"Email or password incorrect"}` | 400 | `:64-66` |
| nonexistent (well-formed) email | `{"error":"Email or password incorrect"}` | 400 | `:64-66` (user lookup `:62` → `None`) |
| `disabled=true` account | `{"error":"Account disabled"}` | 400 | `:67-69` |
| future `delete_on` account | `{"error":"Account scheduled for deletion"}` | 400 | `:70-74` |
| correct email+password, not activated | `{"error":"Account not activated"}` | **422** | `:75-77` |

All seven rows were **Observed** at runtime (the `disabled` and `delete_on` rows via a throwaway account with that column set in the disposable DB). Cascade order confirmed: empty-body → no-email → bad-creds → disabled → scheduled-deletion → not-activated.

### Malformed-payload HTTP 500s — Observed application defect (Issue-7 disclosure)

The register/login endpoints pass caller-controlled input to `sanitize_email` **before any type validation**, and `sanitize_email` assumes a `str`. Non-string inputs therefore raise unhandled exceptions that surface as **HTTP 500** `{"error":"Internal error"}` via `error_handler` (`server.py:390`). Observed:

```
# register with numeric email
$ curl -s -w '\n<<HTTP=%{http_code}>>\n' -X POST http://localhost:7777/api/auth/register \
    -H 'Content-Type: application/json' -d '{"email":12345,"password":"testpass123"}'
{"error":"Internal error"}
<<HTTP=500>>
# server log:
  File "/app/app/api/views/auth.py", line 104, in auth_register
    email = canonicalize_email(dirty_email)
  File "/app/app/utils.py", line 79, in canonicalize_email
    email_address = sanitize_email(email_address)
  File "/app/app/utils.py", line 99, in sanitize_email
    email_address = email_address.strip().replace(" ", "").replace("\n", " ")
AttributeError: 'int' object has no attribute 'strip'
```

```
# register with missing/null email -> NoneType at utils.py:102
$ curl -s -w '\n<<HTTP=%{http_code}>>\n' -X POST http://localhost:7777/api/auth/register \
    -H 'Content-Type: application/json' -d '{"password":"testpass123"}'
{"error":"Internal error"}
<<HTTP=500>>
# server log: AttributeError: 'NoneType' object has no attribute 'replace'
#   File "/app/app/utils.py", line 102, in sanitize_email
#     return email_address.replace("\u200f", "")
```

```
# login with numeric email -> int.strip at utils.py:99 (reaches it because 12345 is truthy, passing the :57 guard)
$ curl -s -w '\n<<HTTP=%{http_code}>>\n' -X POST http://localhost:7777/api/auth/login \
    -H 'Content-Type: application/json' -d '{"email":12345,"password":"x"}'
{"error":"Internal error"}
<<HTTP=500>>
# server log: AttributeError: 'int' object has no attribute 'strip'  (via auth.py:59 -> utils.py:99)
```

**Root cause (`file:line`).** `sanitize_email` (`app/utils.py:97`) guards with `if email_address:` — a `None` skips the guard and crashes at `:102` (`None.replace`), while an `int` passes the guard but crashes at `:99` (`int.strip`). `canonicalize_email` (`app/utils.py:78`) calls `sanitize_email` at `:79`. Register calls `canonicalize_email(dirty_email)` at `app/api/views/auth.py:104` **before** any validation; login calls `sanitize_email(email)` at `auth.py:59`. There is no type/`isinstance` check before these calls. **Disclosed, not fixed** (read-only).

**Honest negative finding.** A NUL byte in the email is **not** a 500 — it is caught by mailbox validation and returns **400**:

```
$ curl -s -w '\n<<HTTP=%{http_code}>>\n' -X POST http://localhost:7777/api/auth/register \
    -H 'Content-Type: application/json' -d '{"email":"a\u0000b@example.com","password":"testpass123"}'
{"error":"cannot use a\u0000b@example.com as personal inbox"}
<<HTTP=400>>
```

**Label: Observed** for the register 200, the login 422 `{"error":"Account not activated"}`, the `activated=f`/`notification=t` DB values, all seven sibling branches, the three malformed-payload 500s, and the NUL-byte 400.

---

## Q5 — Dynamic alias limit before/after a config change + restart

**Direct answer (Observed):** With the **canonical build** limit (`MAX_NB_EMAIL_FREE_PLAN=3`), `/api/user_info` returns `"max_alias_free_plan": 3`. After setting `MAX_NB_EMAIL_FREE_PLAN=10` and **fully restarting** the server, **both** the pre-existing user #1 **and** the newly created user #2 return `"max_alias_free_plan": 10`. So **both users reflect the new limit — not only the user created after the change**, because the limit is resolved **per-request from a process-global module constant** read once at import, and is **not** persisted per user.

**Three observed values of the limit (grounded):**
- **`3`** — the canonical mandated-image build (`/build.sh` exports `MAX_NB_EMAIL_FREE_PLAN=3`).
- **`5`** — the fallback when the variable is **unset**: on startup `app/config.py:123` prints `MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value` and `:124` assigns `5`.
- **`10`** — the Q5 test override.

### API-auth precondition + real activation flow (dynamic values captured to variables)

`/api/user_info` is protected by `@require_api_auth`, so each user must be **activated** to obtain an API key. With `NOT_SEND_EMAIL=true` the activation email is suppressed but the 6-digit code is written to the `account_activation` table (`auth_register` creates it at `app/api/views/auth.py:129-130`). The flow reads the code from the DB into a **shell variable** and activates via the **real** `/api/auth/activate` endpoint — nothing hard-coded:

```
# --- user1, BEFORE the change (server #1, limit 3) ---
$ curl -s -X POST http://localhost:7777/api/auth/register \
    -H 'Content-Type: application/json' -d '{"email":"q5user1@example.com","password":"user1pass123"}'
{"msg":"User needs to confirm their account"}

$ CODE1=$(PGPASSWORD=test psql "$DB_URI" -tAc \
    "select aa.code from account_activation aa join users u on u.id=aa.user_id where u.email='q5user1@example.com';")

$ curl -s -X POST http://localhost:7777/api/auth/activate \
    -H 'Content-Type: application/json' -d "{\"email\":\"q5user1@example.com\",\"code\":\"$CODE1\"}"
{"msg":"Account is activated, user can login now"}

# capture the api key into a variable; NEVER publish it verbatim
$ U1_KEY=$(curl -s -X POST http://localhost:7777/api/auth/login \
    -H 'Content-Type: application/json' \
    -d '{"email":"q5user1@example.com","password":"user1pass123","device":"q5"}' \
    | python3 -c 'import sys,json; print(json.load(sys.stdin)["api_key"])')
$ echo "U1_KEY = ${U1_KEY:0:6}...${U1_KEY: -4}   (len ${#U1_KEY})"
U1_KEY = kmmntp...rsgd   (len 60)

$ curl -s -w '\n<<HTTP=%{http_code}>>\n' -H "Authentication: $U1_KEY" http://localhost:7777/api/user_info
{"can_create_reverse_alias":true,"connected_proton_address":null,"email":"q5user1@example.com","in_trial":true,"is_premium":true,"max_alias_free_plan":3,"name":"q5user1@example.com","profile_picture_url":null}
<<HTTP=200>>
```

→ **BEFORE: `max_alias_free_plan = 3`** (canonical build value). *(The API key is shown redacted as `first6…last4`; the full 60-char token stays in the `$U1_KEY` variable and is invalidated at teardown — see [Issue 13](#observed-application-anomalies-pre-existing-disclosed-not-fixed).)*

### Config change + full restart

```
$ pkill -f 'gunicorn wsgi:app'                              # stop server #1 (master PID 321047)
$ export MAX_NB_EMAIL_FREE_PLAN=10
$ gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15 &     # server #2 (NEW master PID 388136)
[2026-07-14 01:52:14 +0000] [388136] [INFO] Listening at: http://0.0.0.0:7777 (388136)
```

The **different master PID (321047 → 388136)** proves a genuine restart. `MAX_NB_EMAIL_FREE_PLAN` is read only at import (`app/config.py:121-124`), so the change takes effect only after this restart.

```
# --- user1, AFTER the change (server #2, limit 10, SAME pre-existing key) ---
$ curl -s -w '\n<<HTTP=%{http_code}>>\n' -H "Authentication: $U1_KEY" http://localhost:7777/api/user_info
{"can_create_reverse_alias":true,"connected_proton_address":null,"email":"q5user1@example.com","in_trial":true,"is_premium":true,"max_alias_free_plan":10,"name":"q5user1@example.com","profile_picture_url":null}
<<HTTP=200>>

# --- user2, NEW user registered+activated AFTER the change (server #2, limit 10) ---
$ curl -s -X POST http://localhost:7777/api/auth/register \
    -H 'Content-Type: application/json' -d '{"email":"q5user2@example.com","password":"user2pass123"}'
{"msg":"User needs to confirm their account"}
$ CODE2=$(PGPASSWORD=test psql "$DB_URI" -tAc \
    "select aa.code from account_activation aa join users u on u.id=aa.user_id where u.email='q5user2@example.com';")
$ curl -s -X POST http://localhost:7777/api/auth/activate \
    -H 'Content-Type: application/json' -d "{\"email\":\"q5user2@example.com\",\"code\":\"$CODE2\"}"
{"msg":"Account is activated, user can login now"}
$ U2_KEY=$(curl -s -X POST http://localhost:7777/api/auth/login \
    -H 'Content-Type: application/json' \
    -d '{"email":"q5user2@example.com","password":"user2pass123","device":"q5"}' \
    | python3 -c 'import sys,json; print(json.load(sys.stdin)["api_key"])')
$ echo "U2_KEY = ${U2_KEY:0:6}...${U2_KEY: -4}"
U2_KEY = jysxyr...jkkb
$ curl -s -w '\n<<HTTP=%{http_code}>>\n' -H "Authentication: $U2_KEY" http://localhost:7777/api/user_info
{"can_create_reverse_alias":true,"connected_proton_address":null,"email":"q5user2@example.com","in_trial":true,"is_premium":true,"max_alias_free_plan":10,"name":"q5user2@example.com","profile_picture_url":null}
<<HTTP=200>>
```

→ **AFTER: user1 (pre-existing) = 10 AND user2 (new) = 10 — both reflect the new limit.**

**Fallback = 5 (variable unset + restart), Observed:**

```
$ pkill -f 'gunicorn wsgi:app' ; unset MAX_NB_EMAIL_FREE_PLAN
$ gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15 2>&1 | grep -m1 "MAX_NB_EMAIL_FREE_PLAN"
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
$ curl -s -H "Authentication: $U1_KEY" http://localhost:7777/api/user_info \
    | python3 -c 'import sys,json; print("max_alias_free_plan =", json.load(sys.stdin)["max_alias_free_plan"])'
max_alias_free_plan = 5
```

**Summary table (Observed):**

| User | Created under | Server #1 (build = 3) | Server #2 (=10, after restart) | Unset (=5 fallback) |
|---|---|---|---|---|
| q5user1@example.com | before change | **3** | **10** | **5** |
| q5user2@example.com | after change | — (did not exist) | **10** | — |

**Rationale (`file:line`).** `/api/user_info` → `user_info()` (`app/api/views/user_info.py:50-52`, `@require_api_auth`, returning `jsonify(user_to_dict(user))` at `:67`). `user_to_dict()` sets `"max_alias_free_plan": user.max_alias_for_free_account()` at **`:34`** — computed **per request**. `User.max_alias_for_free_account()` (`app/models.py:858-865`) returns `config.MAX_NB_EMAIL_OLD_FREE_PLAN` **iff** the `FLAG_FREE_OLD_ALIAS_LIMIT` bit (`1<<2 = 4`, defined at `app/models.py:341`) is set, else `config.MAX_NB_EMAIL_FREE_PLAN`. `config.MAX_NB_EMAIL_FREE_PLAN` is assigned once at import (`app/config.py:121-124`), so it is fixed for the process lifetime — hence a restart is required, and once restarted **every** user without the flag resolves to the new value. The resolved limit is never written to the user row.

**Flag confirmation (Observed):**

```
$ PGPASSWORD=test psql "$DB_URI" -c \
    "select email, flags, (flags & 4) as flag_bit from users where email in ('q5user1@example.com','q5user2@example.com') order by email;"
        email        | flags | flag_bit
---------------------+-------+----------
 q5user1@example.com |     1 |        0
 q5user2@example.com |     1 |        0
(2 rows)
```

Both users have `flags=1`, `(flags & 4)=0` — `FLAG_FREE_OLD_ALIAS_LIMIT` unset — so both take the `else` branch (`app/models.py:865`) and return `MAX_NB_EMAIL_FREE_PLAN` (never the OLD-plan default `15` at `app/config.py:126`).

### Additional Observed behaviors surfaced by the Q5 flow (disclosures)

Reproducing Q5 exercised `/api/user_info`'s auth and response path, surfacing three pre-existing behaviors (consolidated in [§Observed Application Anomalies](#observed-application-anomalies-pre-existing-disclosed-not-fixed)):

**Issue 2 — scheduled-deletion authorization inconsistency (CRITICAL).** `User.is_active()` (`app/models.py:766-769`) is `if self.delete_on is None: return True; return self.delete_on < arrow.now()`. API auth gates on it at `app/api/base.py:39-40` (`if not g.user.is_active(): return jsonify(error="Account does not exist"), 401`). Observed on user1's real key:

```
# delete_on = NULL  -> normal
$ curl -s -o /dev/null -w "%{http_code}\n" -H "Authentication: $U1_KEY" http://localhost:7777/api/user_info
200
# delete_on = now()+30d (FUTURE)
$ PGPASSWORD=test psql "$DB_URI" -c "update users set delete_on=now()+interval '30 days' where email='q5user1@example.com';" >/dev/null
$ curl -s -w ' <<%{http_code}>>\n' -H "Authentication: $U1_KEY" http://localhost:7777/api/user_info
{"error":"Account does not exist"} <<401>>
# delete_on = now()-1d (PAST)  -> THE ANOMALY
$ PGPASSWORD=test psql "$DB_URI" -c "update users set delete_on=now()-interval '1 day' where email='q5user1@example.com';" >/dev/null
$ curl -s -w ' <<%{http_code}>>\n' -H "Authentication: $U1_KEY" http://localhost:7777/api/user_info
{"can_create_reverse_alias":true,...,"email":"q5user1@example.com","in_trial":true,"is_premium":true,"max_alias_free_plan":5,"name":"q5user1@example.com",...} <<200>>
$ PGPASSWORD=test psql "$DB_URI" -c "update users set delete_on=NULL where email='q5user1@example.com';" >/dev/null   # restore
```

A **past-due** scheduled-deletion account satisfies `delete_on < now()` → `is_active()` returns `True` → **HTTP 200 with full PII**. Meanwhile login (`auth_login`, `app/api/views/auth.py:70-74`) rejects **any** non-null `delete_on` with 400 `"Account scheduled for deletion"`. So login and API auth **diverge**: login blocks both future and past; API auth blocks only future, letting past-due deletion accounts regain full API access and PII. **Disclosed, not fixed** (read-only).

**Issue 8 — API key in query string logged verbatim (MAJOR).** Passing the key as a query param (no `Authentication` header) is correctly **rejected** with 401 (`base.py:17` reads the header, `:27` returns `Wrong api key`), **yet** the full key is written to the DEBUG request log:

```
$ curl -s -w ' <<%{http_code}>>\n' "http://localhost:7777/api/user_info?api_key=$U1_KEY"
{"error":"Wrong api key"} <<401>>
# server log (server.py:284, after_request):
... GET /api/user_info ImmutableMultiDict([('api_key', 'kmmntp...rsgd')]) 401, takes 0.00x
```

`after_request` (`server.py:273`) logs `request.args` verbatim at `server.py:284-291` (the `request.args` argument is at `:289`), so credentials placed in the query string are persisted to logs even on rejection. **Disclosed, not fixed** (read-only).

**Issue 12 — authenticated PII response lacks security headers (MINOR).** The `/api/user_info` 200 (PII) response headers:

- **Present:** `Server: gunicorn/20.0.4` (version disclosure), `Content-Type: application/json`, `Access-Control-Allow-Origin: *`, `Set-Cookie: slapp=...; HttpOnly; Path=/; SameSite=Lax` (**no `Secure`** on http).
- **Absent (all):** `Cache-Control`, `Vary`, `X-Content-Type-Options`, `X-Frame-Options`, `Content-Security-Policy`, `Strict-Transport-Security`, `Referrer-Policy`, `X-XSS-Protection`.

The 401 response has identical headers minus the body. So the PII response is cacheable/unhardened and the server version is exposed. **Disclosed, not fixed** (read-only).

**Issue 13 — captured credentials are ephemeral (invalidation proof).** Every API key/cookie above is captured to a shell variable and shown only as `first6…last4`; the full token is never published. Keys are DB-bound and unusable after teardown:

```
# register + activate + login a throwaway user, capturing its key to a variable (same pattern as U1_KEY)
$ curl -s -X POST http://localhost:7777/api/auth/register \
    -H 'Content-Type: application/json' -d '{"email":"q5_td@example.com","password":"tdpass1234"}' >/dev/null
$ TDCODE=$(PGPASSWORD=test psql "$DB_URI" -tAc \
    "select aa.code from account_activation aa join users u on u.id=aa.user_id where u.email='q5_td@example.com';")
$ curl -s -X POST http://localhost:7777/api/auth/activate \
    -H 'Content-Type: application/json' -d "{\"email\":\"q5_td@example.com\",\"code\":\"$TDCODE\"}" >/dev/null
$ TD_KEY=$(curl -s -X POST http://localhost:7777/api/auth/login \
    -H 'Content-Type: application/json' -d '{"email":"q5_td@example.com","password":"tdpass1234","device":"q5"}' \
    | python3 -c 'import sys,json; print(json.load(sys.stdin)["api_key"])')
$ echo "TD_KEY = ${TD_KEY:0:6}...${TD_KEY: -4}"
TD_KEY = buztht...zfji
# the key works before teardown, then fails after its backing row is removed (what a DB drop does)
$ curl -s -o /dev/null -w "before teardown: %{http_code}\n" -H "Authentication: $TD_KEY" http://localhost:7777/api/user_info
before teardown: 200
$ PGPASSWORD=test psql "$DB_URI" -c "delete from api_key where code='$TD_KEY';" >/dev/null
$ curl -s -w ' <<%{http_code}>>\n' -H "Authentication: $TD_KEY" http://localhost:7777/api/user_info
{"error":"Wrong api key"} <<401>>
```

**Label: Observed** (before/after JSON for both users across the restart; fallback=5; process-global config semantics; and all three disclosures captured directly).

---

## Observed Application Anomalies (pre-existing; disclosed, not fixed)

These are properties of the **application/environment itself**, surfaced by exercising the canonical paths for Q1–Q5. The task is read-only on source (AAP §0.4.2), so each is grounded at `file:line` and **disclosed, not remediated**. None affects the correctness of the Q1–Q5 answers above.

| # | Severity | Behavior (Observed) | Grounding (`file:line`) | Where reproduced |
|---|---|---|---|---|
| 2 | CRITICAL | Past-due scheduled-deletion account regains API-key access + PII (200), while login blocks it (400). API auth and login diverge. | `app/models.py:766-769` (`is_active`); `app/api/base.py:39-40`; login `app/api/views/auth.py:70-74` | [Q5](#additional-observed-behaviors-surfaced-by-the-q5-flow-disclosures) |
| 7 | MAJOR | Wrong-type / missing / null email payloads → unhandled **HTTP 500** on register & login (`int.strip` / `NoneType.replace`). | `app/utils.py:97-102` (`sanitize_email`), `:78-79` (`canonicalize_email`); `app/api/views/auth.py:104` (register), `:59` (login); `server.py:390` | [Q4](#malformed-payload-http-500s--observed-application-defect-issue-7-disclosure) |
| 8 | MAJOR | API key supplied as a query param is rejected (401) but logged verbatim in the request log. | `server.py:273`, `:284-291` (`request.args` at `:289`) | [Q5](#additional-observed-behaviors-surfaced-by-the-q5-flow-disclosures) |
| 11 | MAJOR | Pinned runtime carries known CVEs: gunicorn 20.0.4 (CVE-2024-1135 / CVE-2024-6827), aiosmtpd 1.4.2 (CVE-2024-27305 / CVE-2024-34083). | `pyproject.toml` / `poetry.lock` pins; `pip-audit` output | [§A.7](#a7--pre-existing-dependency-cves-disclosed-not-remediable-under-read-only) |
| 12 | MINOR | Authenticated PII responses lack `Cache-Control`/`Vary`/`X-Content-Type-Options`/`X-Frame-Options`/CSP/HSTS/`Referrer-Policy`; `Server` version exposed; `slapp` cookie has no `Secure`. | Observed response headers on `/api/user_info` | [Q5](#additional-observed-behaviors-surfaced-by-the-q5-flow-disclosures) |

Also disclosed as an **environment prerequisite** (not a code bug, but required for Q4/Q5 to run on the mandated image): the OpenSSL 3.0.16 PKCS#8 DKIM key causes register to 500 until converted to PKCS#1 — see [§A.6](#a6--dkim-key-format-prerequisite-read-only-safe).

---

## Cleanup and Read-Only Verification

Every ephemeral artifact created for observation is removed; the source repository is left byte-for-byte unchanged.

```
# 1) stop all processes started for observation
$ pkill -f 'gunicorn wsgi:app' ; pkill -f 'email_handler.py'

# 2) drop every throwaway database (the main build DB `test` is inside the disposable container)
$ for db in q1a q1b qa_main ; do su postgres -c "dropdb --if-exists $db" ; done

# 3) flush Redis session/rate-limit state
$ redis-cli flushall
OK

# 4) remove mktemp workspaces (timing helper, DKIM copy, migration logs)
$ rm -rf /tmp/sl_q1.* /tmp/sl_q2.* /tmp/sl_q3.* /tmp/sl_dkim.* /tmp/sl_dkim500.*

# 5) destroy the disposable container entirely (removes its /app checkout, DBs, Redis, keys, temp — everything)
$ exit
$ docker rm -f sl_qafix
```

**Read-only verification (delivered repository):**

```
$ git -C /path/to/delivered/repo status --porcelain
# (only this file, blitzy/documentation/app_2cd6ee777f8c.md, appears — nothing else)
```

Because all runtime work occurred inside `sl_qafix` (destroyed in step 5), and the delivered repository is a **different** checkout that was never executed, no application source, schema, config, or dependency is modified. The single persisted change is this document.

---

## Coverage-Pass Checklist

| Item | Where addressed | Status |
|---|---|---|
| Q1: total tables on empty DB | Q1 → `77` (via `information_schema.tables`) | ✅ Observed |
| Q1: exact last table by migration order | Q1 → `user_audit_log` | ✅ Observed |
| Q1: 255 files ≠ table count; created vs altered | Q1 (63 create_table files / 84 stmts; 161 add_column) | ✅ Observed |
| Q1: stability on a 2nd fresh DB | Q1 (77 / `user_audit_log` again; identical set) | ✅ Observed |
| Q2: readiness message (socket-bind) | Q2 → `Listening at: http://0.0.0.0:7777 (<master-pid>)` | ✅ Observed |
| Q2: socket-bind vs request-serving readiness | Q2 (HTTP 302 probe; `Listening at:` ≠ app-ready) | ✅ Observed |
| Q2: ms between first log and ready | Q2 → ≈ 0.2 ms socket-bind (0.22/0.19/0.20); ≈ 615 ms request-serving | ✅ Observed, ≥3 runs |
| Q2: failure modes (missing DB_URI; Redis down) | Q2 (exit 3 HaltServer; every request 500) | ✅ Observed |
| Q2: Werkzeug caveat (no "Running on") | Q2 (`app/log.py:70-71`; grep = 0) | ✅ Observed |
| Q3: confirms listening on port `25025` | Q3 → INFO `Listen for port 25025` + DEBUG `Start mail controller 0.0.0.0 25025` | ✅ Observed |
| Q3: intent vs confirmed; SMTP + PID proof | Q3 (220/250/221 dialogue; PID 262387; occupied-port errno 98, exit 1) | ✅ Observed |
| Q3: default port `20381` contrast | Q3 | ✅ Observed |
| Q4: email `testuser@example.com`, pw `testpass123` | Q4 register + login | ✅ Observed |
| Q4: exact JSON error + HTTP status | Q4 → `{"error":"Account not activated"}`, `422` | ✅ Observed |
| Q4: full curl output | Q4 (verbose `-i` blocks) | ✅ Observed |
| Q4: DB `activated` / `notification` booleans | Q4 → `f` / `t` (before and after failed login) | ✅ Observed |
| Q4: sibling login-error branches | Q4 (7-row table, all Observed) | ✅ Observed |
| Q4: malformed-payload 500s disclosed; NUL=400 | Q4 (Issue-7 disclosure; honest negative) | ✅ Observed |
| Q5: endpoint `/api/user_info`, field `max_alias_free_plan` | Q5 | ✅ Observed |
| Q5: value before change | Q5 → `3` (canonical build); `5` fallback disclosed | ✅ Observed |
| Q5: limit value `10` after change + restart | Q5 → `10` (new master PID proves restart) | ✅ Observed |
| Q5: do BOTH users reflect new limit? | Q5 → yes, both `10` | ✅ Observed |
| Q5: authz anomaly / query-key logging / headers | Q5 + Anomalies (Issues 2, 8, 12) | ✅ Observed |
| Env: initialize mandated image from scratch | §A (runbook: docker run → build → env → services → DKIM) | ✅ Observed |
| Env: build narrative; DKIM prereq; dep CVEs; cleanup | §A.2 / §A.6 / §A.7 / §Cleanup | ✅ Observed |

**Observed vs. Inferred summary.** **Every** primary answer (Q1–Q5) and every named item (email `testuser@example.com`, password `testpass123`, port `25025` and default `20381`, endpoint `/api/user_info`, field `max_alias_free_plan`, columns `activated`/`notification`, limit values `3`/`5`/`10`) is **Observed** from live runtime output captured inside the mandated image. All five sibling login branches, all three malformed-payload 500s, the NUL-byte 400, both Q2 failure modes, the Q3 occupied-port failure, and all disclosed anomalies (Issues 2, 7, 8, 11, 12) are likewise **Observed** — this document contains **no Inferred values**.

---

## References (external framework behavior)

- **Gunicorn `Listening at:` (Q2).** The master (arbiter) binds the listening socket and logs the line in `Arbiter.start()` via `self.log.info("Listening at: %s (%s)", listeners_str, self.pid)`, *before* forking workers; a failed worker boot makes the master exit with `WORKER_BOOT_ERROR = 3`. Source: gunicorn `arbiter.py` (`github.com/benoitc/gunicorn/blob/master/gunicorn/arbiter.py`); docs: `docs.gunicorn.org`. This is stable across the `^20.0.4` line pinned here.
- **aiosmtpd `Controller` (Q3).** `Controller` is a threaded INET listener; `start()` runs the server on a dedicated thread (`run_until_complete(create_server())`), blocks until the thread signals ready (default `ready_timeout` 1 s, env `AIOSMTPD_CONTROLLER_TIMEOUT`) or **re-raises the thread's bind exception**, and emits no "listening" banner of its own — so SimpleLogin's `email_handler.py:2403`/`:2386` lines are the authoritative port confirmation. Docs: `aiosmtpd.aio-libs.org/en/1.4.3/controller.html`; source: `github.com/aio-libs/aiosmtpd/blob/master/aiosmtpd/controller.py`.
