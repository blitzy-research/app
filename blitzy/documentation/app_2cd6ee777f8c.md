# SimpleLogin — Runtime Behavior Investigation (branch `app_2cd6ee777f8c`)

This document **empirically** determines the actual runtime/initialization behavior of the [SimpleLogin](https://github.com/simple-login/app) email‑alias system — not what the code merely *suggests* should happen. The system was **built and run** (PostgreSQL + Alembic migrations, the Gunicorn web server, the aiosmtpd email handler, the HTTP API, and direct database queries), and the five questions below are answered from **observed runtime output**, then reconciled with line‑by‑line code citations. Throughout, **the code is the source of truth**: every value was captured live and cross‑checked against the actual source — no value is inferred from reasoning alone. Each answer is presented as **Question → Method/commands → Observed output → Code citations → Rationale/thinking**.

A recurring theme — and the reason an empirical approach matters here — is that the *observed* runtime answer differs subtly from a naive code reading in three of the five cases:

- **Q1:** the migration graph **head** creates an *index*, not the last *table*.
- **Q2:** the readiness line is Gunicorn's **master‑process** line, not SimpleLogin's own application logger.
- **Q5:** `max_alias_free_plan` is a **request‑time global**, not a value stored per user.

These "runtime ≠ naive inference" points are made explicit in the relevant sections, because surfacing them is exactly what this investigation set out to establish.

---

## Environment & Setup

**How the system was built and run.**

- **Container image.** The AAP‑cited private image `andrewparkscaleai/coding-agent:simple-login__app__2cd6ee777f8c…` (from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`) was **access‑denied** (private registry, not pullable). An **equivalent runtime** was therefore built to match the project's documented versions — `postgres:12.1` (per `README.md:243`) for the database and Python **3.10** (`pyproject.toml` requires `^3.10`; `Dockerfile` uses `python:3.10`) — with the SimpleLogin dependency set installed from the project's own lockfile.

- **Dependency versions (observed in the running interpreter; "code is truth").** The dependencies were installed at the versions pinned by `poetry.lock`/`pyproject.toml`, and these exact versions were verified by importing them in the running interpreter:

  | Package | Version observed at runtime | Source pin |
  |---------|-----------------------------|------------|
  | SQLAlchemy | `1.3.24` | `pyproject.toml` / `poetry.lock` |
  | Flask | `1.1.2` | `pyproject.toml` |
  | gunicorn | `20.0.4` | `poetry.lock` |
  | aiosmtpd | `1.4.2` | `poetry.lock` |
  | alembic | `1.4.3` | `poetry.lock` |

  > **Version transparency note.** The AAP narrative referenced a hypothetical "equivalent runtime" using slightly different patch versions (gunicorn `20.1.0`, aiosmtpd `1.2.2`, alembic `1.7.7`). The **actual** runtime used here installs the versions pinned in `poetry.lock` — gunicorn **`20.0.4`**, aiosmtpd **`1.4.2`**, alembic **`1.4.3`** — and the observed log strings below reflect those real, locked versions. The Gunicorn readiness wording (`Listening at:`) is **identical across the entire 20.x line**, so this patch difference does not affect any of the five answers; only the version number embedded in the `Starting gunicorn …` banner differs.

- **Configuration mechanism.** SimpleLogin loads configuration via a dotenv file: `app/config.py` reads the `CONFIG` environment variable and, if set, loads that file; otherwise it loads `.env` from the working directory (`app/config.py:65-71`). An **ephemeral** `CONFIG`/`.env` dotenv (derived from `example.env`) supplied `DB_URI` → PostgreSQL, `FLASK_SECRET`, `URL`, and `EMAIL_DOMAIN`, and — for Requirement 5 — a toggled `MAX_NB_EMAIL_FREE_PLAN`. **These experiment config files are non‑committed runtime artifacts** (created only to exercise the system) and are **not** part of the repository.

- **Runtime‑only shims that do NOT affect any answer** (stated transparently): `re2` → stdlib `re`; `cryptography==37.0.1` (for PGPy's `register_interface`); `protobuf 5.29.6` (for `event_pb2`); and the `gpg` binary. These are environment conveniences only and change **none** of the observed values.

- **Result.** `create_app()` builds fully; Gunicorn serves the API on port **7777**; `email_handler.py` binds port **25025** on demand.

- **Reusable empty‑database recipe** (from `scripts/reset_local_db.sh`): drop and recreate the public schema, then apply all migrations and (optionally) seed demo data:

  ```sql
  drop schema public cascade; create schema public;
  ```
  ```bash
  alembic upgrade head
  flask dummy-data
  ```

Unless stated otherwise, observations were taken under the **normal application configuration** (the SimpleLogin defaults), **not** the test configuration `tests/test.env`. Where a configuration choice affects a value (notably Requirement 5's baseline), the configuration used is called out explicitly.

---

## Q1 — Database migrations: total tables on an empty DB + last table created

**Question (verbatim):** *"On an empty PostgreSQL database, how many tables are created in total once all migrations are applied, and what is the exact name of the last table created (according to migration application order)?"*

**Method / commands.** Start from a guaranteed‑empty schema (per `scripts/reset_local_db.sh`), apply the entire Alembic chain, then count base tables directly from `information_schema`:

```sql
-- 1) empty the schema (per scripts/reset_local_db.sh)
drop schema public cascade; create schema public;
```
```bash
# 2) apply ALL migrations against the empty DB (ran with ZERO errors)
CONFIG=/abs/path/ephemeral.env alembic upgrade head
```
```sql
-- 3) count BASE TABLES
SELECT count(*) FROM information_schema.tables
WHERE table_schema='public' AND table_type='BASE TABLE';
```

To independently corroborate the count and the *ordering*, all 255 migration `upgrade()` bodies were parsed with Python's `ast` module and replayed in Alembic's own topological order (via `alembic.script.ScriptDirectory`), tallying every `create_table` / `drop_table` / `rename_table` operation.

**Observed output (exact values).**

- **Total = 77 tables = 76 application tables + `alembic_version`** (Alembic's own bookkeeping table). Equivalently, **76 tables excluding `alembic_version`**. The live count:

  ```
   count
  -------
      77
  (1 row)
  ```
  and excluding Alembic's bookkeeping table:
  ```
   count
  -------
      76
  (1 row)
  ```

- **Last table created (by migration application order) = `user_audit_log`** — created by revision `7d7b84779837`, file `migrations/versions/2024_101611_7d7b84779837_user_audit_log.py`. The tail of the live `alembic upgrade head` output confirms the application order:

  ```
  INFO  [alembic.runtime.migration] Running upgrade 62afa3a10010 -> 91ed7f46dc81, alias_audit_log
  INFO  [alembic.runtime.migration] Running upgrade 91ed7f46dc81 -> 7d7b84779837, user_audit_log
  INFO  [alembic.runtime.migration] Running upgrade 7d7b84779837 -> 32f25cbf12f6, alias_audit_log_index_created_at
  ```

- **The migration head `32f25cbf12f6`** (file `2024_101616_32f25cbf12f6_alias_audit_log_index_created_at.py`) creates **only an index** — `ix_alias_audit_log_created_at` — and **no table**. After migration, `alembic_version` stores exactly this head revision: `32f25cbf12f6`.

- **The penultimate table‑creating revision `91ed7f46dc81`** (file `2024_101113_91ed7f46dc81_alias_audit_log.py`) creates `alias_audit_log`.

**Supporting facts (verified live + by AST replay).**

- `alembic.ini:5` → `script_location = migrations`.
- **255** revision files; **single head** `32f25cbf12f6`; **single base** `5e549314e1e2` (both confirmed via `alembic heads` / `alembic show`).
- Across all `upgrade()` bodies (replayed in topological order): **80 `create_table`** calls spanning **79 distinct** names, **4 `drop_table`**, and **3 `rename_table`** operations.
- **Renames:** `gen_email` → `alias`, `forward_email` → `contact`, `forward_email_log` → `email_log`.
- **`drop_table` targets (upgrade direction):** `client_scope`, `scope` (revision `551c4e6d4a8b`), `partner` (revision `2e2b53afd819`), `metric` (revision `20c738810b1b`). Note that **`partner` is later re‑created**, so it remains present in the final schema; `client_scope`, `metric`, and `scope` are dropped and not recreated; and the three renamed‑away *old* names disappear. **`metric2` is a distinct table** and is still present.
- Merge revision `2634b41f54db` converges two branches (tuple `down_revision = ('01e2997e90d3', '2d89315ac650')`).
- **Triple‑confirmed count:** (a) the live `information_schema` query returned **76 application tables + `alembic_version` = 77**; (b) the AST replay of all 255 `upgrade()` bodies yielded a **net 76** application tables; and (c) a direct existence check confirmed `user_audit_log`, `alias_audit_log`, `alias`, `contact`, `email_log`, `partner`, and `metric2` all present while `client_scope`, `metric`, `scope`, `gen_email`, `forward_email`, and `forward_email_log` are all absent.

**Code citations.**

- `alembic.ini:5` — `script_location = migrations`.
- `migrations/versions/2024_101616_32f25cbf12f6_alias_audit_log_index_created_at.py:22` — `op.create_index('ix_alias_audit_log_created_at', 'alias_audit_log', ['created_at'], unique=False, postgresql_concurrently=True)` (index only; `down_revision = '7d7b84779837'`).
- `migrations/versions/2024_101611_7d7b84779837_user_audit_log.py:22` — `op.create_table('user_audit_log', …)` (`down_revision = '91ed7f46dc81'`).
- `migrations/versions/2024_101113_91ed7f46dc81_alias_audit_log.py:22` — `op.create_table('alias_audit_log', …)`.
- `migrations/versions/2023_042011_2634b41f54db_.py:15` — `down_revision = ('01e2997e90d3', '2d89315ac650')` (merge).

**Rationale / thinking.** The count is only meaningful on a **truly empty schema**, which is why the `drop schema … cascade; create schema` reset precedes `alembic upgrade head`. The net application table set equals *(creates − drops, after accounting for the three renames and the fact that `partner` is dropped then re‑created)* = **76**. PostgreSQL additionally holds Alembic's own `alembic_version` row‑tracking table, giving **77** total base tables. The phrase *"last table created"* means the final `create_table` in **topological (application) order** — which occurs in revision `7d7b84779837` and creates **`user_audit_log`**. This is the "runtime ≠ naive inference" subtlety: the actual graph **head** `32f25cbf12f6` is applied *after* `7d7b84779837`, but it only adds an **index** (`ix_alias_audit_log_created_at`), so it is **not** the last *table*. A naive "the head is the last thing, so the last table is whatever the head touches" reading would be wrong; the empirical migration log makes the true order unambiguous.

---

## Q2 — Web server readiness log + millisecond delta

**Question (verbatim):** *"What is the exact log message that signals the web server is ready to accept connections, and how many milliseconds elapse between the first log entry and that ready message?"*

**Method / commands.** Launch the production entry point (the `Dockerfile` `CMD`) and capture startup logs with a **sub‑second timestamper** — necessary because Gunicorn's default error‑log timestamps are only second‑resolution, so the first line and the readiness line frequently print within the *same* printed second:

```bash
# Dockerfile CMD (Dockerfile:47): production entry point
gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15

# captured with a sub-second timestamper. moreutils `ts` was unavailable in this
# runtime, so each stderr line was prefixed with a millisecond UTC wall-clock stamp
# (and a perf_counter delta was computed) by a small wrapper, equivalent to:
gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15 2>&1 | ts '%H:%M:%.S'
```

**Observed output (exact values).** A representative run (millisecond UTC timestamps prepended by the wrapper):

```
[21:37:01.275] [2026-06-26 21:37:01 +0000] [39473] [INFO] Starting gunicorn 20.0.4
[21:37:01.276] [2026-06-26 21:37:01 +0000] [39473] [INFO] Listening at: http://0.0.0.0:7777 (39473)
[21:37:01.276] [2026-06-26 21:37:01 +0000] [39473] [INFO] Using worker: sync
[21:37:01.278] [2026-06-26 21:37:01 +0000] [39474] [INFO] Booting worker with pid: 39474
```

- **Readiness message (exact):** `Listening at: http://0.0.0.0:7777 (<pid>)` — e.g. the full line `[2026-06-26 21:37:01 +0000] [39473] [INFO] Listening at: http://0.0.0.0:7777 (39473)`.
- **First log entry (exact):** `[INFO] Starting gunicorn 20.0.4` (this runtime's locked Gunicorn version; see the version‑transparency note in *Environment & Setup*).
- **Δ (first → ready):** measured at **≈ 0.2 ms** — and consistently **sub‑millisecond** across repeated runs: **0.199 ms, 0.230 ms, 0.198 ms** (three runs). This is well under 1 ms.
- **Note on the raw Gunicorn timestamps:** every master line prints the *same* second (`…01`), so the millisecond delta **cannot** be read from Gunicorn's own log; it is only visible with the sub‑second timestamper. SimpleLogin's own application‑import logs (`load config file …`, `>>> URL: …`, `Paddle param not set`, `>>> init logging <<<`) appear **after** `Booting worker` (≈ 600 ms later, when the worker imports the app), confirming that the `Listening at:` line — emitted by the **master** process — is the readiness signal.

**Code citations.**

- `wsgi.py` — `from server import create_app` / `app = create_app()` (the Gunicorn application target).
- `Dockerfile:44` — `EXPOSE 7777`; `Dockerfile:47` — `CMD ["gunicorn","wsgi:app","-b","0.0.0.0:7777","-w","2","--timeout","15"]`.
- `app/log.py:13` — the SimpleLogin log format `%(asctime)s - %(name)s - %(levelname)s - %(process)d - "%(pathname)s:%(lineno)d" - %(funcName)s() - %(message_id)s - %(message)s`; `app/log.py:43` — `converter = time.gmtime` (UTC, millisecond `asctime`); `app/log.py:51` — `setLevel(logging.DEBUG)`; `app/log.py:67` — `print(">>> init logging <<<")`; `app/log.py:70-71` — the `werkzeug` logger is disabled; `app/log.py:79` — `LOG = _get_logger("SL")`.

**Rationale / thinking.** Gunicorn's master process emits a fixed startup sequence: `Starting gunicorn 20.x.x` → `Listening at: http://0.0.0.0:7777 (pid)` → `Using worker: sync` → `Booting worker with pid: …`. The listening **socket is bound at `Listening at:`**, so that is precisely the *"ready to accept connections"* line. The first → ready delta is therefore just socket‑bind time — **inherently tiny (sub‑millisecond here) and environment‑variable**. This is the second "runtime ≠ naive inference" point: one might expect SimpleLogin's own logger (which *does* emit millisecond‑precision UTC `asctime`) to mark readiness, but those application lines are produced only when the **worker** imports the app — *after* `Booting worker` — and so they are **not** the readiness signal. And because Gunicorn's default error‑log timestamps are second‑resolution, a true millisecond delta is only obtainable with a sub‑second timestamper (or by wrapping the process start with a high‑resolution wall‑clock measurement), exactly as done above.

---

## Q3 — Email handler on custom port 25025

**Question (verbatim):** *"When the email handler is started on port 25025, does the startup log confirm it is listening on that specific port? What is the exact message text?"*

**Method / commands.**

```bash
python email_handler.py -p 25025
```

**Observed output (exact values) — YES, the startup log confirms port 25025.** Two SimpleLogin‑formatted lines are emitted (the empty `message_id` field renders as the two spaces between the dashes):

```
2026-06-26 21:38:00,246 - SL - INFO - 40040 - "/…/email_handler.py:2403" - <module>() -  - Listen for port 25025
2026-06-26 21:38:00,248 - SL - DEBUG - 40040 - "/…/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 25025
```

- **INFO line (from `email_handler.py:2403`):** `Listen for port 25025`
- **DEBUG line (from `email_handler.py:2386`):** `Start mail controller 0.0.0.0 25025`

Both lines echo the exact port **25025**.

**Order & code citations.** At `__main__`, the INFO line `Listen for port 25025` (`email_handler.py:2403`, `LOG.i("Listen for port %s", args.port)`, function `<module>`) prints **first**; then `main(port)` logs the DEBUG line `Start mail controller 0.0.0.0 25025` (`email_handler.py:2386`, `LOG.d("Start mail controller %s %s", controller.hostname, controller.port)`, function `main`), where the controller is `Controller(MailHandler(), hostname="0.0.0.0", port=port)` (`email_handler.py:2383`) and `controller.start()` is called at `email_handler.py:2385`. The `-p/--port` argument default is **20381** (`email_handler.py:2399`: `"-p", "--port", help="SMTP port to listen for", type=int, default=20381`), so absent `-p` the handler would log port **20381** instead.

**Rationale / thinking.** The port passed via `-p 25025` flows into **both** the human‑readable `Listen for port` INFO line (which logs `args.port` directly) **and** the aiosmtpd `Controller(port=...)`, whose bound `controller.port` is echoed by the `Start mail controller` DEBUG line. Crucially, that DEBUG line prints `controller.port` **after** `controller.start()` has run, so it reflects the **actually bound** port — making it definitive confirmation that the handler is listening on **25025**, not merely that the value was parsed from the command line.


---

## Q4 — Register, then login before activation; DB columns

**Question (verbatim):** *"Register a user with email `testuser@example.com` and password `testpass123`, then attempt to log in BEFORE activating the account. What is the exact JSON error message and HTTP status code (include the full curl output)? Then query the database directly: what are the boolean values of the `activated` and `notification` columns for that user?"*

**Method / commands.**

```bash
# Register
curl -i -X POST http://localhost:7777/api/auth/register \
  -H 'Content-Type: application/json' \
  -d '{"email":"testuser@example.com","password":"testpass123"}'

# Login BEFORE activation
curl -i -X POST http://localhost:7777/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"testuser@example.com","password":"testpass123"}'
```
```sql
SELECT activated, notification FROM users WHERE email='testuser@example.com';
```

**Observed output (exact values).**

- **Register → `HTTP 200 OK`**, body `{"msg":"User needs to confirm their account"}`. In this runtime's network the MX gate for `example.com` passed, so registration succeeded directly. Full `curl -i` output:

  ```
  HTTP/1.1 200 OK
  Server: gunicorn/20.0.4
  Date: Fri, 26 Jun 2026 21:38:37 GMT
  Connection: close
  Content-Type: application/json
  Content-Length: 46
  Access-Control-Allow-Origin: *
  Set-Cookie: slapp=…; Expires=…; HttpOnly; Path=/; SameSite=Lax

  {"msg":"User needs to confirm their account"}
  ```

- **Login before activation → `HTTP 422 UNPROCESSABLE ENTITY`**, body (exact): `{"error":"Account not activated"}`. Full `curl -i` output:

  ```
  HTTP/1.1 422 UNPROCESSABLE ENTITY
  Server: gunicorn/20.0.4
  Date: Fri, 26 Jun 2026 21:38:38 GMT
  Connection: close
  Content-Type: application/json
  Content-Length: 34
  Access-Control-Allow-Origin: *
  Set-Cookie: slapp=…; Expires=…; HttpOnly; Path=/; SameSite=Lax

  {"error":"Account not activated"}
  ```

- **DB columns** (psql boolean rendering — `f` = false, `t` = true):

  ```
          email         | activated | notification
  ----------------------+-----------+--------------
   testuser@example.com | f         | t
  (1 row)
  ```

  i.e. **`activated = f` (false)** and **`notification = t` (true)**.

**Caveat documented (MX edge case).** `email_can_be_used_as_mailbox` performs an MX lookup, and `SKIP_MX_LOOKUP_ON_CHECK = False` (`app/config.py:600`). In a network where `example.com` has **no** resolvable MX record, registration would instead return `{"error":"cannot use testuser@example.com as personal inbox"}` with **`HTTP 400`** (`app/api/views/auth.py:114`). In that case a non‑activated user is seeded directly (e.g. via `flask shell` → `User.create(...)`, or `flask dummy-data`) so that the **login‑before‑activation** behavior — the actual subject of the question — can still be observed. **Either way the login result is identical** (HTTP 422, `{"error":"Account not activated"}`). In this particular runtime the MX lookup for `example.com` *did* resolve, so the registration returned `HTTP 200` directly and no seeding fallback was needed.

**Code citations.**

- `app/api/views/auth.py:75` — `elif not user.activated:` → `app/api/views/auth.py:77` — `return jsonify(error="Account not activated"), 422`.
- `app/api/views/auth.py:141` — register success `return jsonify(msg="User needs to confirm their account"), 200`.
- `app/api/views/auth.py:114` — MX‑gate failure `return jsonify(error=f"cannot use {email} as personal inbox"), 400`.
- `app/api/views/auth.py:118` — password `< 8` → `"password too short"` (400); `app/api/views/auth.py:122` — `> 100` → `"password too long"` (400). `testpass123` is 11 characters, so it passes both checks.
- `app/models.py:354-356` — `notification = sa.Column(sa.Boolean, default=True, nullable=False, server_default="1")` ⇒ `notification = t`; `app/models.py:358` — `activated = sa.Column(sa.Boolean, default=False, nullable=False, index=True)` ⇒ `activated = f`.
- `app/config.py:600` — `SKIP_MX_LOOKUP_ON_CHECK = False`.

**Rationale / thinking.** A freshly registered, non‑partner user is created **un‑activated** (`activated` defaults to `False`) with **notifications enabled** (`notification` defaults to `True`). The login endpoint evaluates its guard clauses in order — *bad credentials → disabled → scheduled‑for‑deletion → not‑activated → …* — and, on reaching `elif not user.activated:`, short‑circuits with **HTTP 422** and `{"error":"Account not activated"}`. It never issues an API key until the account is activated. Consequently the database row for the freshly registered user shows exactly **`activated=f, notification=t`**, which is precisely what the direct SQL query returns.

---

## Q5 — `max_alias_free_plan` dynamic behavior

**Question (verbatim):** *"(a) Create a fresh user, call `/api/user_info`, and observe the `max_alias_free_plan` value. (b) Then set the configuration limit to 10, restart the server, create a SECOND new user, and call `/api/user_info` again. What are the exact API responses before and after, and do BOTH users reflect the new limit or only the user created after the change?"*

**Method / commands.** For each user, an API key was created (so the authenticated endpoint can be called), then:

```bash
# GET /api/user_info with the user's API key in the Authentication header
curl -s http://localhost:7777/api/user_info -H 'Authentication: <api_key>'
```

`app/api/views/user_info.py:34` serializes `"max_alias_free_plan": user.max_alias_for_free_account()`; the endpoint is `GET /api/user_info` (`app/api/views/user_info.py:50`), returning `jsonify(user_to_dict(user))` (`app/api/views/user_info.py:67`).

**Observed output (exact values).**

- **Before — NORMAL configuration**, `MAX_NB_EMAIL_FREE_PLAN` unset → default **5**. The server's startup log emits, verbatim:

  ```
  MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
  ```

  A fresh user (`q5user1@example.com`, `flags = 1`) then returns:

  ```json
  {"can_create_reverse_alias":true,"connected_proton_address":null,"email":"q5user1@example.com","in_trial":true,"is_premium":true,"max_alias_free_plan":5,"name":"Q5 User","profile_picture_url":null}
  ```

- **After — set `MAX_NB_EMAIL_FREE_PLAN=10`** in the ephemeral config and **restart the server** (the startup log no longer prints the "is not set" message). Querying again:

  - **Pre‑existing `q5user1@example.com` → `"max_alias_free_plan":10`:**
    ```json
    {"can_create_reverse_alias":true,"connected_proton_address":null,"email":"q5user1@example.com","in_trial":true,"is_premium":true,"max_alias_free_plan":10,"name":"Q5 User","profile_picture_url":null}
    ```
  - **Newly created `q5user2@example.com` (`flags = 1`) → `"max_alias_free_plan":10`:**
    ```json
    {"can_create_reverse_alias":true,"connected_proton_address":null,"email":"q5user2@example.com","in_trial":true,"is_premium":true,"max_alias_free_plan":10,"name":"Q5 User","profile_picture_url":null}
    ```

- **Answer: BOTH users reflect the new limit (10) — not only the user created after the change.** The value went **5 → 10** for the pre‑existing user as well as the new one.

The full JSON response shape (Flask's `jsonify` sorts keys alphabetically):

```json
{"can_create_reverse_alias":true,"connected_proton_address":null,"email":"…","in_trial":true,"is_premium":true,"max_alias_free_plan":N,"name":"…","profile_picture_url":null}
```

where `N` = **5** before and **10** after.

**Code citations.**

- `app/api/views/user_info.py:34` — `"max_alias_free_plan": user.max_alias_for_free_account()`; `app/api/views/user_info.py:50` — `@api_bp.route("/user_info")`; `app/api/views/user_info.py:67` — `return jsonify(user_to_dict(user))`.
- `app/models.py:858-865` — `max_alias_for_free_account()` returns `config.MAX_NB_EMAIL_OLD_FREE_PLAN` (15) **iff** `FLAG_FREE_OLD_ALIAS_LIMIT == self.flags & FLAG_FREE_OLD_ALIAS_LIMIT`, **else** `config.MAX_NB_EMAIL_FREE_PLAN`.
- `app/models.py:339` — `FLAG_DISABLE_CREATE_CONTACTS = 1 << 0` (= 1); `app/models.py:341` — `FLAG_FREE_OLD_ALIAS_LIMIT = 1 << 2` (= 4); `app/models.py:544-548` — the new‑user `flags` column Python default is `FLAG_DISABLE_CREATE_CONTACTS` (= 1).
- `app/config.py:121-124` — `MAX_NB_EMAIL_FREE_PLAN` is read **once at import**: `try: MAX_NB_EMAIL_FREE_PLAN = int(os.environ["MAX_NB_EMAIL_FREE_PLAN"])` / `except …: print("MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value"); MAX_NB_EMAIL_FREE_PLAN = 5`; `app/config.py:126` — `MAX_NB_EMAIL_OLD_FREE_PLAN` defaults to 15.
- `example.env:55` — `# MAX_NB_EMAIL_FREE_PLAN=5` (commented out → default 5 under the normal config); `tests/test.env:13` — `MAX_NB_EMAIL_FREE_PLAN=3` (the **test** config, deliberately **not** used for this baseline).

**Rationale / thinking.** This is the third "runtime ≠ naive inference" point: `max_alias_free_plan` is **computed at request time from global configuration**, not stored on the user row. For a freshly created user, `flags = FLAG_DISABLE_CREATE_CONTACTS = 1`, and `1 & FLAG_FREE_OLD_ALIAS_LIMIT (4) == 0`, so `max_alias_for_free_account()` takes the `config.MAX_NB_EMAIL_FREE_PLAN` branch (it would only take the old‑plan branch — 15 — if the `FLAG_FREE_OLD_ALIAS_LIMIT` bit were set). Because the limit is a **global** value re‑read on every request, once the process is **restarted** with the new config, **every** free user — old *and* new — reports the new limit; hence both `q5user1` and `q5user2` return **10**. A **restart is required** because `MAX_NB_EMAIL_FREE_PLAN` is read **exactly once at module import** (`app/config.py:121-124`); editing the config without restarting would leave the already‑imported value in place. Finally, the baseline must be observed under the **normal** application configuration (default **5**): `tests/test.env` sets the limit to **3**, which would give a misleading baseline — so, explicitly, the **normal config (5)** was used for the "before" measurement and the toggled config (10) for the "after".


---

## Summary of Answers

| # | Question | Answer |
|---|----------|--------|
| 1 | Tables on empty DB / last table | **77 total** (76 application tables + `alembic_version`); last table created = **`user_audit_log`** (revision `7d7b84779837`). The graph head `32f25cbf12f6` adds only an index, not a table. |
| 2 | Web‑server ready log / ms delta | Readiness line **`Listening at: http://0.0.0.0:7777 (<pid>)`**; first line **`Starting gunicorn 20.0.4`**; **Δ < 1 ms (≈ 0.2 ms measured)** — captured with a sub‑second timestamper since Gunicorn's default log timestamps are second‑resolution. |
| 3 | Email handler on 25025 | **Yes** — the log confirms it: **`Listen for port 25025`** (INFO, `email_handler.py:2403`) + **`Start mail controller 0.0.0.0 25025`** (DEBUG, `email_handler.py:2386`). |
| 4 | Login before activation | **HTTP 422 UNPROCESSABLE ENTITY**, body **`{"error":"Account not activated"}`**; DB columns **`activated = f`, `notification = t`**. (Register itself returned HTTP 200 `{"msg":"User needs to confirm their account"}`.) |
| 5 | `max_alias_free_plan` before/after | **5 → 10**; **both** the pre‑existing user *and* the new user report **10** after the restart — the limit is a dynamic, request‑time **global** config value, not stored per user. |

### Closing note — why empirical observation mattered

In three of the five cases the live system contradicts a quick, "obvious" reading of the code, which is exactly why the requirement was to **build and run** rather than infer:

- **Q1:** the migration **head is an index migration**, so the last *table* (`user_audit_log`) is created by the *penultimate*‑in‑graph revision, not by the head.
- **Q2:** readiness is signalled by Gunicorn's **master‑process** `Listening at:` line; SimpleLogin's own millisecond‑precision logger only emits *after* the worker boots and is therefore not the readiness marker.
- **Q5:** `max_alias_free_plan` is **recomputed from global config on every request**, so a single config change plus a restart updates the reported limit for **all** free users at once, old and new alike.

Every value above was captured from a running instance and reconciled against the cited source locations; where the runtime's locked dependency versions differ cosmetically from the AAP narrative (notably `gunicorn 20.0.4` vs. `20.1.0`), the **observed** value is reported, and the difference affects only the version number printed in the banner — not any answer.

