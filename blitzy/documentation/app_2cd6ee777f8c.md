# SimpleLogin — First-Time Initialization Runtime Behavior (Q1–Q5)

> **Source branch:** `app_2cd6ee777f8c` · **Deliverable:** `blitzy/documentation/app_2cd6ee777f8c.md`
> **Methodology:** Every value below was **observed by building and running the canonical code paths** and capturing the real, unedited output — not inferred from reading code. Each answer states the exact command, shows the complete output, cites the `file:line` and function that produced the behavior, covers sibling/edge cases, and is labeled **Observed** or **Inferred**.

## Environment & Setup Preamble

The application was stood up in its canonical configuration inside a container built from the project's own base (`FROM python:3.10`, matching the CI `python-version: 3.10` in `.github/workflows/main.yml` and `pyproject.toml`'s `python = "^3.10"`). Backing services run as sibling containers on a shared Docker network.

**Observed tool/dependency versions** (installed via Poetry from the pinned `poetry.lock`):

| Component | Version (observed) | Manifest constraint |
|---|---|---|
| Python | 3.10.20 | `^3.10` (`pyproject.toml`) |
| gunicorn | 20.0.4 | `^20.0.4` |
| alembic | 1.4.3 | via Flask-Migrate `^2.5.3` |
| Flask | 1.1.2 | `^1.1.2` |
| SQLAlchemy | 1.3.24 | `1.3.24` |
| aiosmtpd | 1.4.2 | `^1.2` |
| redis (client) | 4.6.0 | `^4.5.3` |
| bcrypt | 3.2.0 | `^3.2.0` |
| PostgreSQL (service) | 13 | `.github/workflows/main.yml` |
| Redis (service) | 6 | `.github/workflows/main.yml` |

**Canonical commands used** (stated per the "use the default, canonical build/configuration" rule):
- Dependency install: `poetry install` (from the pinned `poetry.lock`).
- Migrations (Q1): `alembic upgrade head` — equivalent to the canonical `poetry run alembic upgrade head` at `scripts/reset_local_db.sh:6` (dependencies are installed globally in the image, so the bare and `poetry run` forms are equivalent).
- Web server (Q2): `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` — the exact production command from `Dockerfile:47`.
- Email handler (Q3): `python email_handler.py --port 25025`.
- API/DB observation (Q4/Q5): `curl` against the running gunicorn server and `psql` against the PostgreSQL database.

**Default configuration** (a `.env` derived from `example.env`): `NOT_SEND_EMAIL=true` (`example.env:19`) so registration completes without a real MTA; registration left OPEN (`DISABLE_REGISTRATION` commented out, `example.env:58`); `MAX_NB_EMAIL_FREE_PLAN` at its default of `5` (`example.env:55`); `NAMESERVERS` at the canonical default.

**Environment build-only transparency notes** (these affect only how dependencies were *built/installed* in the container — they do **not** modify any repository source file and do **not** affect any observed value):
1. `pip`/build shim: `PIP_CONSTRAINT` pinned `setuptools==67.6.0` and `Cython<3.0` so the transitive `cbor2` C-extension compiles under a modern toolchain.
2. `pyre2` (a native binding incompatible with the container's Debian `re2`) was replaced by a trivial ephemeral `re2.py` shim (`from re import *`). `re2` is used in the codebase only as a drop-in `re` replacement (`app/email_utils.py:23`, `referral.py`, `spamassassin_utils.py`, `regex_utils.py`); none of the five investigated paths depend on `re2`-specific behavior, so the shim changes no observed result.

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

There are exactly **255** `Running upgrade` lines (base revision `5e549314e1e2` → head revision `32f25cbf12f6`).

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

**Created vs. altered (Observed).** Of the 255 migration files: **63** contain `op.create_table` (with **84** `op.create_table` statements total — some init migrations create several tables at once), and **161** contain `op.add_column` (pure ALTERs of existing tables). This is why the file count (255) ≫ the table count (77): migrations predominantly alter existing tables rather than add new ones.

**Stability (Observed).** The upgrade was repeated on a **second** freshly created database: the total was again **77** tables and the last-created table again **`user_audit_log`** — stable.

**Grounds:** `migrations/versions/*.py` (255 files); `alembic.ini:5` (`script_location = migrations`); `scripts/reset_local_db.sh:4,6` (canonical empty-DB pattern); `app/models.py`, `app/db.py` (the SQLAlchemy model/engine layer the migrations materialize); head migration `.../2024_101616_32f25cbf12f6_...py:20-22`; table-creating migration `.../2024_101611_7d7b84779837_user_audit_log.py:22`.

**Label: Observed** (values read from the live database and the ordered `alembic upgrade head` output; confirmed stable on a second fresh database).

---

## Q2 — Web-server startup: readiness message and first-log→ready latency

**Direct answer (Observed):**
- The log line that signals the server is ready to accept connections is Gunicorn's master-process message:
  **`[<timestamp> +0000] [7] [INFO] Listening at: http://0.0.0.0:7777 (7)`**
  (the `(7)` is the master PID; `Listening at:` is the point at which the listening socket is bound).
- The elapsed time between the **first** emitted log entry (`Starting gunicorn 20.0.4`) and that **ready** line is **≈ 0.2 milliseconds** — sub-millisecond and highly stable across runs.

**Command (canonical, `Dockerfile:47`):**

```
$ gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
```

Because Gunicorn's own log timestamps are only second-resolution (see the identical `[... 15:50:56 +0000]` on every master line below), the sub-millisecond delta was measured by an external wrapper that prepends a high-resolution wall-clock stamp (`epoch` and `+<ms>` since the first line) to Gunicorn's **unchanged** output. The Gunicorn command itself is verbatim canonical.

**Complete, unedited output — run 1 (columns: epoch, `+ms` since first line, raw Gunicorn line):**

```
1783957856.021554	+    0.00ms	[2026-07-13 15:50:56 +0000] [7] [INFO] Starting gunicorn 20.0.4
1783957856.021745	+    0.19ms	[2026-07-13 15:50:56 +0000] [7] [INFO] Listening at: http://0.0.0.0:7777 (7)
1783957856.021771	+    0.22ms	[2026-07-13 15:50:56 +0000] [7] [INFO] Using worker: sync
1783957856.024100	+    2.55ms	[2026-07-13 15:50:56 +0000] [9] [INFO] Booting worker with pid: 9
1783957856.077379	+   55.83ms	[2026-07-13 15:50:56 +0000] [10] [INFO] Booting worker with pid: 10
1783957856.484326	+  462.77ms	load config file /work/test.env
1783957856.486434	+  464.88ms	>>> URL: http://localhost:7777
```

**Ready-line delta across ≥2 runs (run scale = 4 full server starts):**

| Run | first-log → `Listening at:` delta |
|---|---|
| 1 | 0.19 ms |
| 2 | 0.21 ms |
| 3 | 0.22 ms |
| 4 | 0.21 ms |

→ **≈ 0.2 ms, stable.** For reference, the time from the first line until a worker finishes importing the SL app (the `>>> init logging <<<` print, `app/log.py:67`) was 798.55 / 793.60 / 797.19 / 795.27 ms across the four runs (~795 ms, also stable).

**Honest measurement note.** The external stamper records the instant each line is *read* from Gunicorn's output pipe. `Starting gunicorn` and `Listening at:` are emitted back-to-back by the master and arrive in the same read burst, so the ~0.2 ms figure faithfully reflects the near-instantaneous gap between those two consecutive master log calls (socket bind is essentially immediate after the start banner).

**Readiness-message rationale & the Werkzeug caveat (`file:line` + function).** The Gunicorn entry point is `wsgi.py:3` (`app = create_app()`, importing `create_app` from `server.py:1`); `create_app()` is defined at `server.py:139`. Crucially, `app/log.py:70-71` does `log = logging.getLogger("werkzeug"); log.disabled = True`, which **disables the Werkzeug request logger** — so on the Gunicorn path there is **no** Flask/Werkzeug `Running on http://…` banner. This was verified: grepping the captured output for `Running on` returns **zero** matches. Therefore the authoritative readiness signal is Gunicorn's own `Listening at:` master line, and the "first log entry" and millisecond delta must be read from the captured, timestamped output (as done above). `app/log.py:67` prints `>>> init logging <<<` once per worker import (with `-w 2`, it appears once per worker).

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

**Complete, unedited output — the two confirming lines:**

```
2026-07-13 15:53:46,355 - SL - INFO - 1 - "/code/email_handler.py:2403" - <module>() -  - Listen for port 25025
2026-07-13 15:53:46,357 - SL - DEBUG - 1 - "/code/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 25025
```

(Immediately preceding these, the handler also emits `>>> init logging <<<` and `load words file: /code/local_data/test_words.txt` — `app/utils.py` — as part of normal startup.)

**Rationale (`file:line` + function).**
- The INFO line comes from `LOG.i("Listen for port %s", args.port)` at **`email_handler.py:2403`**, in the `if __name__ == "__main__":` block (frame `<module>()`), emitted **before** `main()` is called.
- The DEBUG line comes from `LOG.d("Start mail controller %s %s", controller.hostname, controller.port)` at **`email_handler.py:2386`**, inside `main()` and **after** `controller.start()` (`:2385`). This confirms the aiosmtpd `Controller` — constructed at `email_handler.py:2383` as `Controller(MailHandler(), hostname="0.0.0.0", port=port)` — bound hostname `0.0.0.0` and port `25025`.

**Default-port contrast (Observed).** Running without `--port` uses the argparse default `20381` (`email_handler.py:2399`, `default=20381`); the same two lines then read:

```
2026-07-13 15:54:09,072 - SL - INFO - 1 - "/code/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-13 15:54:09,074 - SL - DEBUG - 1 - "/code/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
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

*Registration succeeds in the default config.* (Empirically, `example.com` publishes a null-MX record (`0 .`), so `get_mx_domain_list` yields a non-empty list and the MX gate at `app/email_utils.py:607` does not fire; `email_can_be_used_as_mailbox` returns True. `canonicalize_email` leaves `testuser@example.com` unchanged since it only rewrites gmail/proton addresses.)

**Step 2 — Login before activation. Command and unedited output (verbose `-i`, full headers + body):**

```
$ curl -si -X POST http://localhost:7777/api/auth/login \
    -H 'Content-Type: application/json' \
    -d '{"email":"testuser@example.com","password":"testpass123"}'
HTTP/1.1 422 UNPROCESSABLE ENTITY
Server: gunicorn/20.0.4
Date: Mon, 13 Jul 2026 15:56:06 GMT
Connection: close
Content-Type: application/json
Content-Length: 34
Access-Control-Allow-Origin: *
Set-Cookie: slapp=1c765ac8-c071-43f8-ada1-6f3b5830ae5f.AI92aeQTc0bESHujgDoiAUMM8kg; Expires=Mon, 20-Jul-2026 15:56:06 GMT; HttpOnly; Path=/; SameSite=Lax

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

**Sibling login-error branches (coverage).** Observed at runtime:

```
# empty body {}  →  400
{"error":"request body cannot be empty"}

<<HTTP_STATUS=400>>
```
```
# wrong password  →  400
{"error":"Email or password incorrect"}

<<HTTP_STATUS=400>>
```
```
# nonexistent email  →  400
{"error":"Email or password incorrect"}

<<HTTP_STATUS=400>>
```

These correspond to `auth.py:49-50` (`if not data:`), `auth.py:64-66` (`if not user or not user.check_password(password)`), and `auth.py:56-58` (`if not email:`). Two further siblings — `elif user.disabled:` → 400 `"Account disabled"` (`:67-69`) and `elif user.delete_on is not None:` → 400 `"Account scheduled for deletion"` (`:70-74`) — require pre-seeded DB state and were **not** exercised at runtime; they are reported here from code as **Inferred**.

**Label: Observed** for the register 200, the login **422 `{"error":"Account not activated"}`**, the `activated=f` / `notification=t` DB values, and the empty-body / wrong-password / nonexistent-email 400 siblings. **Inferred** (labeled) for the disabled and scheduled-deletion 400 branches.

---

## Q5 — Dynamic alias limit before/after a config change + restart

**Direct answer (Observed):** With the default limit, `/api/user_info` returns `"max_alias_free_plan": 5`. After setting `MAX_NB_EMAIL_FREE_PLAN=10` and **fully restarting** the server, **both** the pre-existing user #1 **and** the newly created user #2 return `"max_alias_free_plan": 10`. So **both users reflect the new limit — not only the user created after the change.** The reason: the limit is resolved per-request from a **process-global** module-level constant (read once at import), and is **not** persisted per user, so after a restart every request resolves to the new value.

`/api/user_info` is protected by `@require_api_auth`, so each user was registered **and activated** (activation code read from the DB `account_activation` table, since `NOT_SEND_EMAIL=true` disables the email; the real `/api/auth/activate` endpoint was used), then logged in to obtain an API key.

**BEFORE — server #1 with default `MAX_NB_EMAIL_FREE_PLAN=5`, user #1. Command and unedited output:**

```
$ curl -s -w '\n\n<<HTTP_STATUS=%{http_code}>>\n' -H 'Authentication: ytvgcwvwcbaajjfxsjfahzsfugxobayitrzeklqmifcchcrorwdoiblyktem' \
    http://localhost:7777/api/user_info
{"can_create_reverse_alias":true,"connected_proton_address":null,"email":"user1@example.com","in_trial":true,"is_premium":true,"max_alias_free_plan":5,"name":"user1@example.com","profile_picture_url":null}

<<HTTP_STATUS=200>>
```

→ `max_alias_free_plan = 5`. (user #1 activation code observed: `620916`.)

**CONFIG CHANGE + RESTART.** `MAX_NB_EMAIL_FREE_PLAN=10` was set in the test `.env`, and the gunicorn server was **fully restarted** (server #2) against the **same** database. (Verified in-container that `app.config.MAX_NB_EMAIL_FREE_PLAN == 10` in server #2.)

**AFTER — server #2, user #1 (PRE-EXISTING, same API key). Unedited output:**

```
{"can_create_reverse_alias":true,"connected_proton_address":null,"email":"user1@example.com","in_trial":true,"is_premium":true,"max_alias_free_plan":10,"name":"user1@example.com","profile_picture_url":null}

<<HTTP_STATUS=200>>
```

→ `max_alias_free_plan = 10` (JSON otherwise identical to BEFORE).

**AFTER — server #2, user #2 (NEW, registered+activated after the change; activation code `176762`). Unedited output:**

```
{"can_create_reverse_alias":true,"connected_proton_address":null,"email":"user2@example.com","in_trial":true,"is_premium":true,"max_alias_free_plan":10,"name":"user2@example.com","profile_picture_url":null}

<<HTTP_STATUS=200>>
```

→ `max_alias_free_plan = 10`.

**Summary table (Observed):**

| User | Created under | Server #1 (limit 5) | Server #2 (limit 10, after restart) |
|---|---|---|---|
| user1@example.com | before change | **5** | **10** |
| user2@example.com | after change | — (did not exist) | **10** |

**Rationale (`file:line` + function).** `/api/user_info` → `user_info()` (`app/api/views/user_info.py:50-52`, `@require_api_auth`, returning `jsonify(user_to_dict(user))` at `:67`). `user_to_dict()` sets `"max_alias_free_plan": user.max_alias_for_free_account()` at **`:34`** — evaluated **per request**. `User.max_alias_for_free_account()` (`app/models.py:858-865`) returns `config.MAX_NB_EMAIL_OLD_FREE_PLAN` when the `FLAG_FREE_OLD_ALIAS_LIMIT` bit is set, otherwise `config.MAX_NB_EMAIL_FREE_PLAN`. `config.MAX_NB_EMAIL_FREE_PLAN` is assigned once at import in `app/config.py:120-124` (`MAX_NB_EMAIL_FREE_PLAN = int(os.environ["MAX_NB_EMAIL_FREE_PLAN"])`, default `5`). Because that constant lives at module scope and is read only at import time, it is fixed for the life of the process — which is why a **restart** is required for the change to take effect, and why, once restarted, **every** user (old or new) resolves to the new value.

**Flag confirmation (Observed).** For both users `flags = 1` and `(flags & 4) = 0`, i.e. `FLAG_FREE_OLD_ALIAS_LIMIT` is unset, so `max_alias_for_free_account()` takes the `else` branch and returns `MAX_NB_EMAIL_FREE_PLAN` (observed `5` then `10`) — never the OLD-plan value `15` (`MAX_NB_EMAIL_OLD_FREE_PLAN`, `app/config.py:126`).

**Label: Observed** (before/after JSON captured for both users across the restart; process-global config semantics confirmed).

---

## Coverage-Pass Checklist

| Item | Where addressed | Status |
|---|---|---|
| Q1: total tables on empty DB | Q1 → `77` (via `information_schema.tables`) | ✅ Observed |
| Q1: exact last table by migration order | Q1 → `user_audit_log` | ✅ Observed |
| Q1: 255 files ≠ table count; created vs altered | Q1 (63 create_table files / 84 stmts; 161 add_column) | ✅ Observed |
| Q1: stability on a 2nd fresh DB | Q1 (77 / `user_audit_log` again) | ✅ Observed |
| Q2: readiness message | Q2 → `Listening at: http://0.0.0.0:7777 (7)` | ✅ Observed |
| Q2: ms between first log and ready | Q2 → ≈ 0.2 ms (0.19/0.21/0.22/0.21) | ✅ Observed, ≥2 runs |
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
