# SimpleLogin — What Happens When You Create a New Alias (Branch app_2cd6ee777f8c)

This document answers six questions about SimpleLogin's **"create a new alias"** flow from **observed runtime evidence** captured while the application was actually running in its **default, out-of-the-box development configuration**. Every behavioral claim is paired with (a) the actual captured output (real log lines, full HTTP request/response blocks, SQL statements, error text) and (b) a `file:line` citation naming the specific function/method that performs the work.

Labelling conventions used throughout:

- **OBSERVED** — the value was produced at runtime by exercising the real web entry point and captured directly. Unless a block is marked otherwise, it is OBSERVED in the default configuration.
- **INFERRED** — the statement is derived from reading the source code, not from a captured runtime observation. Every such statement is explicitly labelled.
- **NON-CANONICAL** — the observation was produced under a non-default configuration (e.g., a Redis store wired up by hand). Default-configuration values are always reported first; any non-default observation is explicitly labelled.

The six questions answered:

- **Q1 — Run and observe** (Section 2): bring the app up, log in, create an alias, watch the flow live.
- **Q2 — Frontend request** (Section 3): the exact HTTP method, URL, content type, body fields, and CSRF handling.
- **Q3 — Backend response** (Section 4): status code(s), payload, headers, redirect location, flash messaging.
- **Q4 — Database changes** (Section 5): which records are inserted/updated, how many tables, related entities.
- **Q5 — Logs & background work** (Section 6): background tasks, follow-up events, additional work beyond the initial write.
- **Q6 — Error paths** (Section 7): how validation, database, and network/service failures surface in the response, logs, and DB state.

> **The single most important architectural finding:** in the default development configuration, creating a random alias writes **3 tables** (`alias`, `daily_metric`, `alias_audit_log`), the domain event is a **logged no-op** (no `sync_event`, no `NOTIFY`), and the HTTP response is a **Post/Redirect/Get 302 + flash message**. A custom alias with a secondary mailbox writes **4 tables** (adds `alias_mailbox`). This canonical default behavior is cleanly separated from configuration-dependent behavior (a non-default setup with **both** a configured `EVENT_WEBHOOK` **and** a `PartnerUser` would add a `sync_event` INSERT + `NOTIFY` — see §6.3 for the exact three-gate sequence). The exact same creation core — `Alias.create` (`app/models.py:1627-1692`) — backs both the web form and the REST API; only the HTTP envelope differs (302 + flash vs. 201 JSON).

---

## Section 1 — Overview: Environment, Versions, and Bootstrap Commands

All facts in this section are **OBSERVED** from the running instance.

### 1.1 Repository and runtime

All values below were captured from the running instance inside the **mandated canonical Docker image** — this is the exact environment the task requires (AAP §0.8). No ad-hoc host interpreter and no hand-built virtualenv were used; the observation ran inside the image's own venv.

| Item | Value | How captured |
|------|-------|--------------|
| Mandated image (ref) | `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (published alias `andrewparkscaleai/coding-agent:simple-login__app__2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`) | AAP §0.8 / setup |
| Image id | `sha256:ea242796bbce36ca99ba9f783a4e7ac9d2ed3738e737f9a6d96bb22bbf1d9b58` | `docker image inspect` |
| Container / workdir | container `sl-app`; workdir **`/app`** | `docker inspect` |
| OS | Debian GNU/Linux 12 (bookworm) | `cat /etc/os-release` |
| Repo path (runtime) | **`/app`** (the repository, bind-mounted from the working clone) | `pwd` inside container |
| Working git branch | `blitzy-50c513ea-a8be-4059-9e27-0c097dfd57f3` | `git rev-parse --abbrev-ref HEAD` |
| HEAD commit (capture-time; moving) | authoring capture `872849980eb530187250c876c894e8b567732426`; the working-branch `HEAD` is a **moving reference** that successive doc-only review commits have since advanced — most recently to `ff16e389c15a0820924a4dec782842e5fa7fbb02` — and the commit that finalizes these corrections advances it once more, so a fresh checkout reports a **descendant** of the capture hash. Every commit between the base and `HEAD` touches only this deliverable. The read-only invariant is anchored to the immutable **base commit** (next row), not to any moving hash. | `git rev-parse HEAD` (capture-time) |
| Source/base commit | `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c` — ancestor of HEAD; the deliverable is named after the source branch `app_2cd6ee777f8c` | `git merge-base --is-ancestor` |
| Only tracked delta base→HEAD | `A blitzy/documentation/app_2cd6ee777f8c.md` (this file; **no source path is modified**) | `git diff --name-status` |
| Python | **3.10.18** (image venv at `/app/venv`; the alias-flow deps are present at their exact `poetry.lock` pins, with one image-level substitution *off* the flow — `pyre2`→`google-re2`, documented in §1.2) | `python --version` |
| Dev web server | **Werkzeug/1.0.1**, port **7777** (`server.py:588` — `app.run(debug=True, port=7777)`); every response `Server:` header reads `Werkzeug/1.0.1 Python/3.10.18` | response header capture |
| PostgreSQL | **15.13** (Debian 15.13-0+deb12u1) on `:5432` | `SELECT version()` |
| Redis | **7.0.15** on `:6379` (present but *not* wired into rate limiting by default — see §7 B4) | `redis-cli INFO server` |

The project targets Python `^3.10` (`pyproject.toml:61`, `CONTRIBUTING.md`). The mandated image satisfies this directly: it ships **Python 3.10.18** with a pre-built virtualenv at `/app/venv`. Every dependency exercised by the alias-creation flow (the twelve listed in §1.2) is present at its exact `poetry.lock` pin, so no interpreter substitution and no C-extension rebuild were necessary for the path under study. **One image-level dependency substitution does exist and is disclosed rather than glossed over (OBSERVED, off the alias-creation path):** `poetry.lock` pins `pyre2==0.3.6` but the venv ships `google-re2==1.1.20250805` in its place — the captured evidence and its (test-only) impact are in §1.2. Confirmed at runtime — the `Server:` header of the very first response is:

```
$ curl -sD - -o /dev/null http://localhost:7777/ | grep -i '^Server:'
Server: Werkzeug/1.0.1 Python/3.10.18
```

> **Provenance note (fresh canonical run — OBSERVED):** every value, HTTP block, SQL statement, and log line in this document was (re)captured by a **clean run inside the mandated image above**: services started, the database re-bootstrapped from empty (`alembic upgrade head` → `flask dummy-data`, §1.3), the dev server launched, and each condition exercised through its real web entry point. No value is carried over from any other environment or from reading the prompt.

### 1.2 Key pinned dependencies (from `poetry.lock`; versions verified live in the image venv)

The versions below were read back at runtime from `/app/venv` (not just from `poetry.lock`):

```
$ /app/venv/bin/python -c "import importlib.metadata as m; \
  [print(f'{p}=={m.version(p)}') for p in \
  ['flask','flask-login','flask-wtf','wtforms','flask-limiter','sqlalchemy', \
   'alembic','psycopg2-binary','redis','protobuf','arrow','email-validator']]"
flask==1.1.2
flask-login==0.5.0
flask-wtf==0.14.3
wtforms==2.3.3
flask-limiter==1.4
sqlalchemy==1.3.24
alembic==1.4.3
psycopg2-binary==2.9.3
redis==4.6.0
protobuf==5.27.1
arrow==0.16.0
email-validator==1.1.3
```

| Package | Version | Role in the alias-creation flow |
|---------|---------|--------------------------------|
| flask | 1.1.2 | routing, request/response, flash messaging |
| flask-login | 0.5.0 | `@login_required`, `current_user` |
| flask-wtf | 0.14.3 | CSRF-protected forms (`CSRFValidationForm`) |
| wtforms | 2.3.3 | form field definitions |
| flask-limiter | 1.4 | route-level rate limiting (`ALIAS_LIMIT`) |
| sqlalchemy | 1.3.24 | ORM, scoped `Session`, SQL emission |
| alembic | 1.4.3 | schema migrations (`alembic upgrade head`) |
| psycopg2-binary | 2.9.3 | PostgreSQL driver |
| redis | 4.6.0 | client backing the per-user token bucket + Redlock |
| protobuf | 5.27.1 | encoding of the `AliasCreated` domain event |
| arrow | 0.16.0 | timestamps on models |
| email-validator | 1.1.3 | alias/email address validation (custom-alias prefix path) |

**Dependency-pin discrepancy (OBSERVED — off the alias-creation path).** The twelve packages above were each verified present at their exact locked versions, but the venv is **not** a perfect mirror of `poetry.lock` for *every* one of its ~70 packages, so the blanket phrasing "all ~70 deps at exact `poetry.lock` pins" would be inaccurate. The lock pins `pyre2` (the Cython RE2 wrapper) at `0.3.6`, yet the image installs Google's own `google-re2` in its place and the locked `pyre2` is absent entirely. The substitute exposes a different API surface — it lacks the `re2.DOTALL` attribute the `pyre2` compatibility layer provides — so code written against the `pyre2` API raises `AttributeError` on that attribute. Captured live in the image venv:

```
$ grep -n -A1 '^name = "pyre2"' poetry.lock
2473:name = "pyre2"
2474-version = "0.3.6"
$ /app/venv/bin/pip show pyre2 2>&1 | head -1
WARNING: Package(s) not found: pyre2
$ /app/venv/bin/pip show google-re2 | grep -E '^(Name|Version):'
Name: google-re2
Version: 1.1.20250805
$ /app/venv/bin/python -c "import re2; print('re2 file =', re2.__file__); print('has DOTALL =', hasattr(re2, 'DOTALL'))"
re2 file = /app/venv/lib/python3.10/site-packages/re2/__init__.py
has DOTALL = False
$ /app/venv/bin/python -c "import re2; re2.DOTALL"
AttributeError: module 're2' has no attribute 'DOTALL'
```

**Impact on this investigation: none.** The only modules that `import re2` are `app/dashboard/views/referral.py`, `app/email_utils.py`, `app/spamassassin_utils.py`, and `app/regex_utils.py` (OBSERVED via `grep -rln 'import re2' app tests`) — none is on the alias-creation web path (`app/dashboard/views/index.py`, `app/dashboard/views/custom_alias.py`, `app/models.py`), which never imports `re2`. *(INFERRED, not re-run here: per the image's own dependency notes the substitution also breaks a small number of RE2-dependent tests — three `email_handler` modules at collection time plus one regex test — which is why the blanket "all ~70 deps at exact `poetry.lock` pins" phrasing is corrected to this precise statement.)*

### 1.3 Canonical bootstrap (default configuration)

The application was brought up exactly as the contributor guide prescribes — `CONTRIBUTING.md` gives the one-liner `alembic upgrade head && flask dummy-data && python3 server.py` and the login instruction (open `http://localhost:7777`, log in with `john@wick.com / password`). For a reproducible, from-empty capture, the `simplelogin` database was dropped and recreated first, then each step below was run **inside the mandated container** and its exit code and key output captured. `.env` is byte-identical to `example.env` (`diff -q .env example.env` → identical), so no config was set: `EVENT_WEBHOOK`, `MEM_STORE_URI`, `REDIS_URL`, and `DISABLE_RATE_LIMIT` are all **absent** (§1.4).

**Step 0 — datastores (default ports):**

```
$ pg_ctlcluster 15 main start        # -> exit 0
$ pg_isready -h localhost            # -> localhost:5432 - accepting connections   (exit 0)
$ redis-server --daemonize yes --dir /tmp ; redis-cli ping   # -> PONG
```

**Step 0b — clean-slate database (drop/create + role/credential context).** `psql` connects over TCP as the image's pre-provisioned superuser `myuser`, which requires a password, so `PGPASSWORD` is exported for the shell session — a *bare* `psql` would instead try to connect as OS user `root` (`psql: error: … FATAL: role "root" does not exist`) and omitting the password yields `psql: error: … fe_sendauth: no password supplied`. With the dev server not yet started (Step 3) there are no open connections, so the `simplelogin` database is dropped and recreated directly, against the maintenance database `postgres`, giving the migrations an empty schema (OBSERVED):

```
$ export PGPASSWORD=mypassword
$ psql -h localhost -U myuser -d postgres -tAc \
    "select 'myuser superuser='||rolsuper from pg_roles where rolname='myuser'"
myuser superuser=true
$ psql -h localhost -U myuser -d postgres -tAc "DROP DATABASE IF EXISTS simplelogin"
DROP DATABASE
$ psql -h localhost -U myuser -d postgres -tAc "CREATE DATABASE simplelogin OWNER myuser"
CREATE DATABASE
$ psql -h localhost -U myuser -d simplelogin -tAc \
    "select count(*) from information_schema.tables where table_schema='public'"   # -> 0 (empty)
0
```

*(If the database is instead re-bootstrapped while the dev server is live, first close its pooled connections — `SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname='simplelogin' AND pid<>pg_backend_pid();` — otherwise `DROP DATABASE` reports `database "simplelogin" is being accessed by other users`.)*

**Step 1 — schema (`alembic upgrade head`):** run as `cd /app && /app/venv/bin/alembic upgrade head` in the image venv. On startup the command prints the app banner (`>>> URL: http://localhost:7777`, then `MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value`) and then applies the project's **full Alembic migration chain** up to head `32f25cbf12f6`. The **contiguous tail** of that run — its final three `Running upgrade` lines and the captured exit code — is shown below (OBSERVED; `alembic upgrade` emits no further output after the last migration, so these are the genuine last lines):

```
INFO  [alembic.runtime.migration] Running upgrade 62afa3a10010 -> 91ed7f46dc81, alias_audit_log
INFO  [alembic.runtime.migration] Running upgrade 91ed7f46dc81 -> 7d7b84779837, user_audit_log
INFO  [alembic.runtime.migration] Running upgrade 7d7b84779837 -> 32f25cbf12f6, alias_audit_log_index_created_at
alembic exit=0
```

Verified afterward (OBSERVED): **77** tables in schema `public`; head revision **`32f25cbf12f6`**:

```
$ psql -h localhost -U myuser -d simplelogin -tAc \
    "select count(*) from information_schema.tables where table_schema='public'"   # -> 77
$ psql -h localhost -U myuser -d simplelogin -tAc 'select version_num from alembic_version'   # -> 32f25cbf12f6
```

**Step 2 — seed data (`flask dummy-data`):** the CLI command is defined at `server.py:490-497` and runs `fake_data()`, `add_sl_domains()`, and `add_proton_partner()`:

```python
    @app.cli.command("dummy-data")
    def dummy_data():
        from init_app import add_sl_domains, add_proton_partner

        LOG.w("reset db, add fake data")
        fake_data()
        add_sl_domains()
        add_proton_partner()
```

Run as `cd /app && FLASK_APP=server.py /app/venv/bin/flask dummy-data`; it completed with a captured **exit code of 0** (OBSERVED). The seeding is verbose (it creates the fake users, SL domains, seeded aliases, and OAuth clients, emitting many log lines). The investigation-relevant detail is that the seeding path itself already fires the **gated event no-op** at `app/events/event_dispatcher.py:62` — the same no-op documented for the alias-creation flow in Q5 / §6. Each occurrence is this verbatim line (a single complete occurrence shown, OBSERVED):

```
2026-07-13 18:34:31,082 - SL - INFO - 3501 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
```

The same run also logs `Disable onboarding emails` (`app/models.py:647`) and `Add sl.local to SL domain` (`init_app.py:44` — a repo-root module, imported as `from init_app import …` at `server.py:492`; the `/app/…` prefix in the runtime log line is the container's bind-mount path, not a Python package path); the resulting seed state (2 users, 11 aliases) is verified immediately below.

Verified afterward (OBSERVED): 2 users (`john@wick.com` id=1 admin, `winston@continental.com` id=2) and **11** seeded aliases (`select count(*),max(id) from alias` → `11|11`, so the next created alias will have **id=12**).

> **Note on `dummy-data` re-runs:** `fake_data()` `INSERT`s without a prior `drop_all`, so re-running it on an already-seeded database fails on the duplicate `john@wick.com`. That is why Step 0 drops/recreates the database first — this is a clean-slate capture.

**Step 3 — dev server:** started from `/app` in the image venv, reloader disabled for a stable single PID (identical entry point and default config as `python3 server.py`; both call `app.run(debug=True, port=7777)` at `server.py:588`):

```
$ cd /app && WERKZEUG_RUN_MAIN=true FLASK_APP=server.py /app/venv/bin/python server.py
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
 * Serving Flask app "server" (lazy loading)
 * Running on http://127.0.0.1:7777/ (Press CTRL+C to quit)
```

The server binds `127.0.0.1:7777` (container loopback). To reach it from the host, a `socat` bridge is started **inside the container**, forwarding the container's published interface (`eth0` = `172.17.0.2`) to that loopback; a host `curl` then confirms the entry point answers with the unauthenticated redirect to the login page (OBSERVED):

```
$ socat TCP-LISTEN:7777,bind=172.17.0.2,fork,reuseaddr TCP:127.0.0.1:7777 &
$ curl -s -o /dev/null -w '%{http_code}\n' http://localhost:7777/     # from the host
302
```

All request/response captures in this document hit that real web entry point.

### 1.4 Default configuration keys (all from `example.env`, which the run used verbatim)

| Key | Value | `file:line` |
|-----|-------|-------------|
| `URL` | `http://localhost:7777` | `example.env:6` |
| `EMAIL_DOMAIN` | `sl.local` | `example.env:22` |
| `DB_URI` | `postgresql://myuser:mypassword@localhost:5432/simplelogin` | `example.env:75` |
| `FLASK_SECRET` | `secret` | `example.env:77` |
| `EVENT_WEBHOOK` | **absent** → defaults to `None` | `app/config.py:612` — `EVENT_WEBHOOK = os.environ.get("EVENT_WEBHOOK", None)` |
| `MEM_STORE_URI` | **absent** → defaults to `None` | `app/config.py:568` — `MEM_STORE_URI = os.environ.get("MEM_STORE_URI", None)` |
| `DISABLE_RATE_LIMIT` | **absent** → defaults to `False` | `app/config.py:602` — `DISABLE_RATE_LIMIT = "DISABLE_RATE_LIMIT" in os.environ` |

**OBSERVED:** `.env` is byte-identical to `example.env`; `EVENT_WEBHOOK`, `MEM_STORE_URI`, and `DISABLE_RATE_LIMIT` do not appear in `example.env` at all, so all three take their unset defaults. These three defaults determine the three headline findings: the event dispatch is a logged no-op (`EVENT_WEBHOOK=None`), the per-user Redis token bucket / Redlock guard are no-ops (`MEM_STORE_URI=None`), and route-level rate limiting is active (`DISABLE_RATE_LIMIT=False`).

### 1.5 Seeded and used accounts (OBSERVED)

Premium/quota status was read back live (`User.can_create_new_alias`, `app/models.py:867`):

```
$ PYTHONPATH=/app /app/venv/bin/python -c "from app.models import User; u=User.get(1); \
  print(u.email,'is_admin=',u.is_admin,'is_premium=',u.is_premium(),'can_create_new_alias=',u.can_create_new_alias())"
john@wick.com is_admin= True is_premium= True can_create_new_alias= True
```

- **id=1 `john@wick.com` / `password`** — `is_admin`, and **PREMIUM** (active subscription seeded by `dummy-data`). OBSERVED live: `is_premium=True`, `can_create_new_alias=True`, so john **bypasses the free-plan quota** and uses the **PAID** rate-limit buckets. Primary happy-path user for §§2–6.
- **id=2 `winston@continental.com`** — also premium (OBSERVED live: `is_premium=True`, `can_create_new_alias=True`); not usable for the quota test.
- A genuinely **FREE** fixture user (`max_alias_for_free_account=5`) is created at runtime **specifically for the quota test** — its creation evidence and exact quota behavior are in **§7 B1**.
- A disposable **API key** for `john` is created at runtime for the web-vs-API contrast; its creation, single use, and deletion appear in **§8** with the key value **redacted** (only a short prefix and its length are printed — never the full secret).

### 1.6 Observation taps (external, NO source instrumentation)

The investigation is fully non-invasive — **no source file was modified** to capture evidence; every tap is external to the application:

- **PostgreSQL statement log:** `log_statement=all` + `log_min_duration_statement=0`, enabled per-database with `ALTER DATABASE simplelogin SET …` + `SELECT pg_reload_conf()` and **reset afterward** (Appendix A). This records every `BEGIN`/`INSERT`/`UPDATE`/`SELECT`/`COMMIT` the SQLAlchemy engine (`app/db.py`) emits. The exact log destination and the enable/reset commands are shown in **§5**.
- **Dev-server stdout** → `/tmp/blitzy_server.log` (the structured `LOG` object, `app/log.py`). (`/tmp` here is container scratch, **not** the repo bind-mount.)
- **A `requests` (2.31.0) HTTP client**, run in the image venv, drove the live web entry point (login → form submission); `curl` drove the API contrast (§8).

Note: the in-memory flask-limiter route budget (5 create-POST/min per §1.4 `ALIAS_LIMIT`; 10 GET/min on `/dashboard/`, `app/dashboard/views/index.py:62`) is ephemeral in-process state, so the dev server was restarted between capture batches to reset the counters.

---

## Section 2 — Q1: Run-and-Observe Walkthrough (OBSERVED, default config)

This is the "watch it live" narrative: log in as the seeded user, then create the first alias.

### 2.1 Login (OBSERVED)

The dashboard is behind Flask-Login (`@login_required`, `app/dashboard/views/index.py:56`), so a session is established first. The exchange was driven by a `requests` client against the real loopback entry point. Command and complete output:

```
$ /app/venv/bin/python /tmp/obs_login.py
======================================================================
STEP 1  GET /auth/login  (fetch login form + csrf_token)
======================================================================
$ GET http://127.0.0.1:7777/auth/login
-> HTTP 200 OK; Content-Length=220839
-> extracted csrf_token (redacted length=91): <present>
-> session cookie present: True

======================================================================
STEP 2  POST /auth/login  {csrf_token, email, password}
======================================================================
$ POST http://127.0.0.1:7777/auth/login  (form: csrf_token=<redacted>&email=john@wick.com&password=<redacted>)
-> HTTP 302 FOUND
-> Location: http://127.0.0.1:7777/dashboard/
-> Set-Cookie: slapp=<authenticated session cookie — value redacted>; Expires=Mon, 20-Jul-2026 18:38:12 GMT; HttpOnly; Path=/; SameSite=Lax
-> Content-Length: 229

======================================================================
STEP 3  GET /dashboard/  (confirm authenticated session)
======================================================================
$ GET http://127.0.0.1:7777/dashboard/
-> HTTP 200 OK; Content-Length=779080
-> page contains /auth/logout link (authenticated): True
-> page contains create-random-email form: True
```

**OBSERVED:** `POST /auth/login` returns **HTTP 302** to `/dashboard/` and sets the authenticated `slapp` session cookie (`HttpOnly; Path=/; SameSite=Lax`); the follow-up `GET /dashboard/` returns **HTTP 200** whose body contains both an `/auth/logout` link and the `create-random-email` form — confirming an authenticated session ready to create aliases. (The literal session-cookie value is redacted; it is a live credential.)

### 2.2 Creating the first alias (OBSERVED)

With the session established, submitting the dashboard's **random-alias** control (a plain HTML `POST` form; exact fields and template lines in **§3**) creates an alias and lands the browser back on the dashboard with a **success flash** rendered as a green toastr ("Alias … has been created"; mechanism in **§4.3**). The live capture of that single click — request, response, and the followed redirect:

```
$ /app/venv/bin/python /tmp/obs_walkthrough.py
======================================================================
POST /dashboard/  (form-name=create-random-email)  -- the single click
======================================================================
--- REQUEST (as sent by the client) ---
POST http://127.0.0.1:7777/dashboard/
Content-Type: application/x-www-form-urlencoded
Content-Length: 132
Cookie: slapp=<redacted>
body: form-name=create-random-email&csrf_token=<redacted>

--- RESPONSE ---
HTTP 302 FOUND
Content-Type: text/html; charset=utf-8
Content-Length: 339
Location: http://127.0.0.1:7777/dashboard/?highlight_alias_id=12&query=&sort=&filter=
Vary: Cookie
Set-Cookie: slapp=<redacted>; Expires=Mon, 20-Jul-2026 18:46:31 GMT; HttpOnly; Path=/; SameSite=Lax

highlight_alias_id in Location: 12

--- FOLLOW REDIRECT: GET the Location ---
GET http://127.0.0.1:7777/dashboard/?highlight_alias_id=12&query=&sort=&filter=
-> HTTP 200 OK; Content-Length=660215
-> body contains 'has been created' flash: True
```

Reading the capture against the code — behind that single click:

- the browser issues `POST /dashboard/` with `form-name=create-random-email` and a `csrf_token`, content type `application/x-www-form-urlencoded` (full request detail in **§3**);
- the `dashboard.index` view (`app/dashboard/views/index.py:55`) validates CSRF and quota, calls `Alias.create_new_random`, and commits once (**§§4–5**);
- this was the **first** alias created after the 11 seeded rows, so it received **id=12** (visible as `highlight_alias_id=12` in the redirect target) — the exact write set is detailed in **§5**;
- the application emits a small fixed set of log lines, including the gated event no-op (exact lines and count in **§6**);
- the view responds **HTTP 302** (Post/Redirect/Get) to `…/dashboard/?highlight_alias_id=12&query=&sort=&filter=` (**§4**);
- the browser follows the redirect with `GET /dashboard/` (**HTTP 200**, body contains the "has been created" flash).

Error and edge conditions (quota, CSRF, rate limit, custom-alias validation, DB integrity) are catalogued in **Section 7**.

---

## Section 3 — Q2: The Frontend Request (OBSERVED, default config)

The two creation controls differ on the client side:

- The **random-alias** controls are **plain HTML `POST` forms** — a click submits immediately with **no JavaScript**; the browser sends the default encoding **`application/x-www-form-urlencoded`**.
- The **custom-alias** control is an HTML `POST` form **with a client-side JavaScript submit handler** (`templates/dashboard/custom_alias.html:112-129`) and Parsley field validation on the prefix (`:33`). The handler validates locally (mailbox chosen, prefix non-empty) and only then calls `form.submit()`; the wire request is still `application/x-www-form-urlencoded` (no JSON/XHR). This distinction matters in §7: several "invalid input" failures are blocked in the browser and are only reachable by a **crafted** request (labelled NON-CANONICAL there).

### 3.1 Form fields

**Random-alias forms** — rendered by `templates/dashboard/index.html`. There are **three** sibling `POST` forms (all posting to `/dashboard/`), each with its own hidden `csrf_token` and `form-name=create-random-email`:

| Random form | `csrf_token` | `form-name` | `generator_scheme` |
|-------------|--------------|-------------|--------------------|
| default button (uses user's generator) | `:51` | `:52` | (none) |
| word variant | `:69` | `:70` | `:72` |
| uuid variant | `:79` | `:80` | `:82` |

The `csrf_token` at `templates/dashboard/index.html:41` belongs to the separate **create-custom-email** button (the "Create custom alias" link that redirects to `/dashboard/custom_alias`), **not** the random forms — that was the previous draft's citation error. The `generator_scheme` values come from `AliasGeneratorEnum` (`app/models.py:211-213`):

```python
class AliasGeneratorEnum(EnumE):
    word = 1  # aliases are generated based on random words
    uuid = 2  # aliases are generated based on uuid
```

The default button form (`:51-52`) carries **no** `generator_scheme`, so the route falls back to the user's default generator (route chain below).

**Custom-alias form** — rendered by `templates/dashboard/custom_alias.html`; exact `name=` attributes:

- `prefix` — text input (`:29`) with a Parsley pattern `[0-9a-z-_.]{1,}` and `maxlength="40"` (`:33-39`);
- `signed-alias-suffix` — `<select>` (`:42`) whose option values are **server-signed** suffixes (`{{ alias_suffix.signed_suffix }}`, `:45`);
- `mailboxes` — a **multi**-`<select>` (`:69-74`; posts as a repeated field, read via `request.form.getlist("mailboxes")`);
- `note` — textarea (`:85`);
- `csrf_token` (`:93`).

The custom form has **no `form-name`** field (its route does not branch on `form-name`, unlike `/dashboard/`).

### 3.2 RANDOM alias — `POST /dashboard/` — route `app/dashboard/views/index.py:55` (`dashboard.index`)

All three generator schemes were exercised. Command and complete output:

```
$ /app/venv/bin/python /tmp/obs_random_schemes.py
======================================================================
RANDOM default (uses current_user.alias_generator)  (generator_scheme=None)
======================================================================
--- REQUEST ---
POST http://127.0.0.1:7777/dashboard/
Content-Type: application/x-www-form-urlencoded
body: form-name=create-random-email&csrf_token=<redacted>
--- RESPONSE ---
HTTP 302 FOUND
Content-Length: 339
Location: http://127.0.0.1:7777/dashboard/?highlight_alias_id=15&query=&sort=&filter=
highlight_alias_id: 15

======================================================================
RANDOM word  (generator_scheme=1)
======================================================================
--- REQUEST ---
POST http://127.0.0.1:7777/dashboard/
Content-Type: application/x-www-form-urlencoded
body: form-name=create-random-email&csrf_token=<redacted>&generator_scheme=1
--- RESPONSE ---
HTTP 302 FOUND
Content-Length: 339
Location: http://127.0.0.1:7777/dashboard/?highlight_alias_id=16&query=&sort=&filter=
highlight_alias_id: 16

======================================================================
RANDOM uuid  (generator_scheme=2)
======================================================================
--- REQUEST ---
POST http://127.0.0.1:7777/dashboard/
Content-Type: application/x-www-form-urlencoded
body: form-name=create-random-email&csrf_token=<redacted>&generator_scheme=2
--- RESPONSE ---
HTTP 302 FOUND
Content-Length: 339
Location: http://127.0.0.1:7777/dashboard/?highlight_alias_id=17&query=&sort=&filter=
highlight_alias_id: 17

created alias ids this run: ['15', '16', '17']
```

Mapping the ids to `alias.email` in the DB confirms each scheme's address style:

```
$ psql -tAc 'select id,email from alias where id between 15 and 17 order by id'
15|mammon_tongue253@sl.local                        <- default (john's alias_generator = word)
16|cowmen_meting741@sl.local                        <- generator_scheme=1 (word)
17|67b9edef-7a36-4ae8-b3a7-7ef6e39c67e0@sl.local    <- generator_scheme=2 (uuid)
```

**Two-run stability (OBSERVED):** the 3-scheme batch was run twice (server restarted between runs to reset the in-memory rate limiter). Both runs produced the identical response shape — `HTTP 302 FOUND`, `Content-Length: 339`, `Location: …/dashboard/?highlight_alias_id=<id>&query=&sort=&filter=` — differing only in the incrementing `highlight_alias_id` (run 1 → 15/16/17, run 2 → 18/19/20). The random address domain `sl.local` is `EMAIL_DOMAIN` (`example.env:22`).

**Route handling chain** (each step cited):

- `app/dashboard/views/index.py:57-60` — the route-level rate limiter, applied only to the create-random POST:
  ```python
  @limiter.limit(
      ALIAS_LIMIT,
      methods=["POST"],
      exempt_when=lambda: request.form.get("form-name") != "create-random-email",
  )
  ```
- `app/dashboard/views/index.py:85` — `csrf_form = CSRFValidationForm()` is built, and `:88-89` rejects an invalid CSRF before any write.
- `app/dashboard/views/index.py:97-104` — the `create-random-email` branch checks `current_user.can_create_new_alias()`, computes the scheme, and calls the creation core:
  ```python
  elif request.form.get("form-name") == "create-random-email":
      if current_user.can_create_new_alias():
          scheme = int(
              request.form.get("generator_scheme") or current_user.alias_generator
          )
          if not scheme or not AliasGeneratorEnum.has_value(scheme):
              scheme = current_user.alias_generator
          alias = Alias.create_new_random(user=current_user, scheme=scheme)
  ```
  This confirms the variant behavior above: `scheme = int(generator_scheme or current_user.alias_generator)`, and an out-of-range value falls back to `current_user.alias_generator`.
- `app/dashboard/views/index.py:108` — `Session.commit()` (the single commit for the whole write set).
- `app/dashboard/views/index.py:113-121` — `redirect(url_for("dashboard.index", highlight_alias_id=alias.id, query=query, sort=sort, filter=alias_filter))` → the HTTP 302 in the response block above.

### 3.3 CUSTOM alias — `POST /dashboard/custom_alias` — route `app/dashboard/views/custom_alias.py:30`

The custom form is read from a `GET` first. Its live option set (OBSERVED):

```
$ /app/venv/bin/python /tmp/obs_suffix_options.py
signed-alias-suffix options (value -> label):
  value='@old.com.alU07g.7iJ_WOdZ9adR7GxWj_YHrDKZvjQ'
    label='@old.com (your domain)'
  value='.striae399@premium.com.alU07g.m658nlHeZ4dQ82VwpbA3T6u6-sI'
    label='.striae399@premium.com (Premium domain)'
  value='.sensed706@sl.local.alU07g._3R2R_DzLbEr4n0kKm7mr-dpFKI'
    label='.sensed706@sl.local (Public domain)'
mailboxes options (value -> label):
  1 -> 'john@wick.com'
  2 -> 'pgp@example.org'
```

Creating with a single mailbox and with two mailboxes (command + complete output):

```
$ /app/venv/bin/python /tmp/obs_custom.py
parsed: mailbox_ids=['1', '2']  suffix(option value, verbatim)='@old.com.alUzuQ.6M6CzYd8Fnx1PlnGI9BYMIWLt4I'
        csrf length=91

======================================================================
CASE 1  custom, SINGLE mailbox -> expect 302 success
======================================================================
--- CASE1: REQUEST ---
POST http://127.0.0.1:7777/dashboard/custom_alias
Content-Type: application/x-www-form-urlencoded
body: prefix=obs-single-1783968697&signed-alias-suffix=%40old.com.alUzuQ.6M6CzYd8Fnx1PlnGI9BYMIWLt4I&note=obs+custom+single&csrf_token=<redacted>&mailboxes=1
--- CASE1: RESPONSE ---
HTTP 302 FOUND
Content-Type: text/html; charset=utf-8
Content-Length: 273
Location: http://127.0.0.1:7777/dashboard/?highlight_alias_id=13
highlight_alias_id: 13
follow -> HTTP 200; 'has been created' present: True

======================================================================
CASE 2  custom, TWO mailboxes -> expect 302 success + alias_mailbox rows
======================================================================
--- CASE2: REQUEST ---
POST http://127.0.0.1:7777/dashboard/custom_alias
Content-Type: application/x-www-form-urlencoded
body: prefix=obs-twombx-1783968697&signed-alias-suffix=%40old.com.alUzug.daTt8naRW3WWafR027IP8hF7eXE&note=obs+custom+two+mbx&csrf_token=<redacted>&mailboxes=1&mailboxes=2
--- CASE2: RESPONSE ---
HTTP 302 FOUND
Content-Type: text/html; charset=utf-8
Content-Length: 273
Location: http://127.0.0.1:7777/dashboard/?highlight_alias_id=14
highlight_alias_id: 14
follow -> HTTP 200; 'has been created' present: True
```

Results (from the DB): `13 obs-single-1783968697@old.com` (1 mailbox) and `14 obs-twombx-1783968697@old.com` (2 mailboxes; the second mailbox becomes an `alias_mailbox` row — §5.3). The two-mailbox request posts `mailboxes` **twice** (`mailboxes=1&mailboxes=2`), read with `request.form.getlist("mailboxes")` (`app/dashboard/views/custom_alias.py:61`). The custom success `Location` carries only `highlight_alias_id` (no `query/sort/filter`), produced at `app/dashboard/views/custom_alias.py:161`.

**The `signed-alias-suffix` value** is an `itsdangerous` `TimestampSigner` signature (signer created at `app/alias_suffix.py:11`):

```python
signer = itsdangerous.TimestampSigner(config.CUSTOM_ALIAS_SECRET)
```

It is **`<value>.<timestamp>.<signature>` parsed from the right** — **not** "three dot-separated parts". The leading `<value>` is itself a suffix that already contains dots (and an `@`). For the observed option `@old.com.alUzuQ.6M6CzYd8Fnx1PlnGI9BYMIWLt4I`, `itsdangerous` right-splits into signature `6M6CzYd8Fnx1PlnGI9BYMIWLt4I`, timestamp `alUzuQ`, and value `@old.com` (which contains the `old`.`com` dot); the `.striae399@premium.com....` option's value contains two dots. Server-side verification is `signer.unsign(signed_suffix, max_age=600)` in `check_suffix_signature` (`app/alias_suffix.py:37-42`) — the **600-second** window is why a stale form fails "expired", and a modified value fails "tampered" (both in §7 B5).

### 3.4 CSRF handling (Q2 sub-question)

Both web forms embed a `csrf_token` (Flask-WTF). The backend validates it via `CSRFValidationForm` **before any write** (`app/utils.py:157-158`):

```python
class CSRFValidationForm(FlaskForm):
    pass
```

- Random path: `app/dashboard/views/index.py:85` builds `CSRFValidationForm()`, and `:88-90` does `if not csrf_form.validate(): flash("Invalid request", "warning"); return redirect(request.url)`.
- Custom path: `app/dashboard/views/custom_alias.py:52` builds `CSRFValidationForm()`, and `:56-58` does the same `flash("Invalid request", "warning")` + redirect.

An empty `CSRFValidationForm` subclass is sufficient because Flask-WTF injects and validates the `csrf_token` field automatically. The invalid-CSRF runtime behavior is captured in §7 B2.

---

## Section 4 — Q3: The Backend Response (OBSERVED, default config)

### 4.1 Post/Redirect/Get — on success (not every mutation)

On **success**, both creation paths use Post/Redirect/Get: the `POST` returns **HTTP 302** with a success flash, and the browser follows the `Location` with a `GET` that returns **HTTP 200** (§§2–3). It is **not** true that "every mutation returns 302": several custom-alias branches flash an error and **fall through to `render_template(...)` at `app/dashboard/views/custom_alias.py:166`, returning HTTP 200** (a re-render of the form). The full outcome map:

| Outcome | HTTP | Where |
|---------|------|-------|
| Random success | **302** → `dashboard.index` | `app/dashboard/views/index.py:113-121` |
| Custom success | **302** → `dashboard.index` | `app/dashboard/views/custom_alias.py:161` |
| Custom **duplicate** ("You already have this alias"), **domain-deleted**, **deleted**, or **"hacked"** branch | **200** (re-render of `custom_alias.html`) | flash at `:120`/`:122`/`:128-132`/`:135`/`:164`, no `return`, fall through to `render_template` at `:166` |
| Custom validation errors (bad prefix, tampered mailbox, expired/tampered suffix, invalid email, `..`) and invalid CSRF | **302** → `redirect(request.url)` | `:58`/`:70`/`:82`/`:87`/`:94`/`:98`/`:105`/`:113` |
| Custom `IntegrityError` | **302** → `redirect(dashboard.custom_alias)` | `:150` |
| Quota gate (free user — even on `GET`) | **302** → `dashboard.index` | `:36-42` (§7 B1) |

Live proof of the **200** re-render (duplicate custom alias — full error catalogue in §7 B5):

```
======================================================================
CASE 3  DUPLICATE of CASE 1 -> expect HTTP 200 re-render (M9)
======================================================================
--- CASE3: REQUEST ---
POST http://127.0.0.1:7777/dashboard/custom_alias
Content-Type: application/x-www-form-urlencoded
body: prefix=obs-single-1783968697&signed-alias-suffix=%40old.com.alUzug.daTt8naRW3WWafR027IP8hF7eXE&note=obs+custom+dup&csrf_token=<redacted>&mailboxes=1
--- CASE3: RESPONSE ---
HTTP 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 551695
body contains 'You already have this alias': True
```

The two success `Location` values differ between routes:

- **Random success**: `…/dashboard/?highlight_alias_id=<id>&query=&sort=&filter=` — `redirect(url_for("dashboard.index", highlight_alias_id=alias.id, query=query, sort=sort, filter=alias_filter))` at `app/dashboard/views/index.py:113-121`.
- **Custom success**: `…/dashboard/?highlight_alias_id=<id>` (no `query/sort/filter`) — `redirect(url_for("dashboard.index", highlight_alias_id=alias.id))` at `app/dashboard/views/custom_alias.py:161`:
  ```python
                  Session.commit()
                  flash(f"Alias {full_alias} has been created", "success")

                  return redirect(url_for("dashboard.index", highlight_alias_id=alias.id))
  ```

### 4.2 Response metadata (OBSERVED — complete raw headers)

Complete, unedited raw response headers for a random-create success (curl; the `slapp` value is a live credential and is redacted):

```
$ bash /tmp/obs_raw_headers.sh
===== RAW RESPONSE HEADERS: POST /dashboard/ (create-random-email) =====
HTTP/1.0 302 FOUND
Content-Type: text/html; charset=utf-8
Content-Length: 339
Location: http://127.0.0.1:7777/dashboard/?highlight_alias_id=21&query=&sort=&filter=
Vary: Cookie
Set-Cookie: slapp=<redacted>; Expires=Mon, 20-Jul-2026 18:55:26 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Mon, 13 Jul 2026 18:55:26 GMT
```

Side-by-side (both success 302s; every header field shown):

| Metadata | Random `POST /dashboard/` | Custom `POST /dashboard/custom_alias` |
|----------|---------------------------|----------------------------------------|
| Status line | `HTTP/1.0 302 FOUND` | `HTTP/1.0 302 FOUND` |
| `Content-Type` | `text/html; charset=utf-8` | `text/html; charset=utf-8` |
| `Content-Length` | `339` | `273` |
| `Location` | `…/dashboard/?highlight_alias_id=<id>&query=&sort=&filter=` | `…/dashboard/?highlight_alias_id=<id>` |
| `Set-Cookie` | `slapp=…; Expires=…; HttpOnly; Path=/; SameSite=Lax` | same attributes |
| `Vary` | `Cookie` | `Cookie` |
| `Server` | `Werkzeug/1.0.1 Python/3.10.18` | `Werkzeug/1.0.1 Python/3.10.18` |

Both success `Content-Length` values are **alias-id-digit dependent** (OBSERVED): the Werkzeug 302 body embeds the redirect `Location` (which carries `highlight_alias_id=<id>`) **twice** — once as the `href` and once as the visible link text — so `Content-Length` grows by **2 bytes per additional digit in the alias id**. The `339`/`273` shown above were captured for a **2-digit** id (`highlight_alias_id=21`); re-running the same creates with a **3-digit** id yields `341` for random (`highlight_alias_id=176`) and `275` for custom (`highlight_alias_id=179`). The duplicate-custom **200** re-render instead carries `Content-Length: 551695` (a full HTML page), as shown in §4.1. `Set-Cookie` refreshes the session on every response.

### 4.3 Flash-message mechanism

The flash is delivered via the session cookie and rendered at the top of the next page by `templates/base.html:98-103`:

```html
        {% with messages = get_flashed_messages(with_categories=true) %}
          <!-- Categories: success (green), info (blue), warning (yellow), danger (red) -->
          {% if messages %}

            {% for category, message in messages %}<script>toastr.{{category }}("{{ message }}");</script>{% endfor %}
          {% endif %}
        {% endwith %}
```

`get_flashed_messages(with_categories=true)` yields `(category, message)` pairs rendered as `<script>toastr.{{category}}("{{message}}");</script>` at `:102` — i.e. **the Flask flash category is used verbatim as the toastr method name**. The alias-creation code uses exactly three categories (the line-only references in the table below are within `app/dashboard/views/index.py` and `app/dashboard/views/custom_alias.py`):

| Flask flash category (in code) | Rendered call | toastr method / color |
|--------------------------------|---------------|-----------------------|
| `success` (`index.py:111`, `custom_alias.py:159`) | `toastr.success(...)` | green |
| `warning` (`index.py:89`/`:95`/`:123`, `custom_alias.py:57`/`:81`/`:93`) | `toastr.warning(...)` | orange |
| `error` (`custom_alias.py:68`/`:86`/`:97`/`:104`/`:112`/`:120`/`:122`/`:135`/`:149`) | `toastr.error(...)` | red |

**Accuracy note (m2):** the `base.html:99` HTML comment reads "…warning (yellow), danger (red)", but that label is **stale** — the alias code **never** flashes a `danger` category. Its red messages use the `error` category, which renders `toastr.error(...)` (toastr's red method is `error`; `danger` is a Bootstrap alert class, not a toastr method). The custom form's client-side JS also raises `toastr.error("…","Error")` directly (`custom_alias.html:118`/`:123`), which is not a Flask flash at all.

Observed success flashes, captured verbatim from the followed `GET 200` bodies:

```
$ /app/venv/bin/python /tmp/obs_flash_line.py
RANDOM followed-GET flash line(s):
  <script>toastr.success("Alias refund_salsas148@sl.local has been created");</script>
CUSTOM followed-GET flash line(s):
  <script>toastr.success("Alias obs-flash-1783969123@old.com has been created");</script>
```

- Random matches `flash(f"Alias {alias.email} has been created", "success")` at `app/dashboard/views/index.py:111`.
- Custom matches `flash(f"Alias {full_alias} has been created", "success")` at `app/dashboard/views/custom_alias.py:159`.

**Accuracy note:** a `toastr.success("Copied to clipboard")` call also exists (`templates/base.html:170`), but it is a **static JS string** wired to the copy button — **not** a flash, and unrelated to alias creation.

---

## Section 5 — Q4: Database Changes (OBSERVED via PostgreSQL `log_statement=all` + psycopg2 before/after row snapshots; default config)

**How captured (producing command).** With the server running in the canonical image, per-database statement logging was enabled temporarily (reset in the cleanup step, Appendix A) so PostgreSQL 15 would log every statement:

```bash
# enable full statement logging on the simplelogin DB (temporary, reset afterwards)
export PGPASSWORD=mypassword
psql -h localhost -U myuser -d simplelogin -c "alter database simplelogin set log_statement='all'"
psql -h localhost -U myuser -d simplelogin -c "alter database simplelogin set log_min_duration_statement=0"
psql -h localhost -U myuser -d postgres    -tAc "select pg_reload_conf()"   # -> t
# then restart the dev server so pooled connections pick up the setting
```

PostgreSQL logs to `/var/log/postgresql/postgresql-15-main.log` (OBSERVED: `logging_collector => off`, so statements go to that file via stderr). A temporary harness (`obs_db_capture.py`, removed afterwards) logged in as `john@wick.com`, took a psycopg2 row snapshot **before** each create, issued the real form `POST`, took a snapshot **after**, and sliced the PostgreSQL log by line-count marks around the single request (backend PID isolates the request). Each variant was run **twice** to confirm stability. This Section-5 statement-logging pass is a **dedicated capture session**, distinct from the Section 2/6 walkthrough; consequently its pooled backend PID (**36580**) and calendar date (**2026-07-14 UTC**) differ from the PIDs and date shown in those sections, and its alias ids (**25–28**) simply continue the sequence past the earlier runs — none of which affects the observed write set or deltas.

### 5.1 The single creation core: `Alias.create`

Both entry points ultimately call `Alias.create` (`app/models.py:1627-1692`), which performs, in ONE call: the per-user Redis rate check → email sanitize → global-trash lookups → `Session.add(new_alias)` (INSERT `alias`) → `DailyMetric.get_or_create_today_metric().nb_alias += 1` (UPDATE `daily_metric`) → **(gated)** `EventDispatcher.send_event(...)` (protobuf built at `app/models.py:1680-1686`) → `emit_alias_audit_log(new_alias, AliasAuditLogAction.CreateAlias, "New alias created")` (INSERT `alias_audit_log`). The complete tail of the method, from the `alias` insert onward (`app/models.py:1660-1692`, unedited):

```python
        Session.add(new_alias)
        DailyMetric.get_or_create_today_metric().nb_alias += 1

        if (
            new_alias.flags & cls.FLAG_PARTNER_CREATED > 0
            and new_alias.user.flags & User.FLAG_CREATED_ALIAS_FROM_PARTNER == 0
        ):
            user.flags = user.flags | User.FLAG_CREATED_ALIAS_FROM_PARTNER

        if commit:
            Session.commit()

        if flush:
            Session.flush()

        # Internal import to avoid global import cycles
        from app.alias_audit_log_utils import AliasAuditLogAction, emit_alias_audit_log
        from app.events.event_dispatcher import EventDispatcher
        from app.events.generated.event_pb2 import AliasCreated, EventContent

        event = AliasCreated(
            id=new_alias.id,
            email=new_alias.email,
            note=new_alias.note,
            enabled=True,
            created_at=int(new_alias.created_at.timestamp),
        )
        EventDispatcher.send_event(user, EventContent(alias_created=event))
        emit_alias_audit_log(
            new_alias, AliasAuditLogAction.CreateAlias, "New alias created"
        )

        return new_alias
```

The `commit`/`flush` kwargs are both `False` by default here (the dashboard callers do not pass them), so the writes are staged, not committed, inside `Alias.create`. The view then issues a single `Session.commit()` (`app/dashboard/views/index.py:108` for random; `app/dashboard/views/custom_alias.py:158` for custom), so **all writes land in one transaction**. The `FLAG_PARTNER_CREATED` branch (`app/models.py:1663-1667`) is inert here because a dashboard-created alias has `flags=0` (OBSERVED in the `INSERT INTO alias … flags … 0 …` above).

### 5.2 RANDOM alias (default scheme) — 3 tables

**Producing command:** `POST /dashboard/` with body `csrf_token=<…>&form-name=create-random-email` (see §3.2), as `john@wick.com`.

**Before/during/after snapshot** (psycopg2, two runs; OBSERVED). Run 1 created alias id=25 (`misers_amnion824@sl.local`); run 2 created id=26 (`volley_tildes080@sl.local`):

| Table | BEFORE (run 1) | AFTER (run 1) | Delta | Operation |
|-------|----------------|---------------|-------|-----------|
| `alias` | count=24, max_id=24 | count=25, max_id=25 | **+1 row** | INSERT |
| `daily_metric` | id=1, nb_alias=**24** | id=1, nb_alias=**25** | **+0 rows / nb_alias +1** | UPDATE (today's row already existed) |
| `alias_audit_log` | count=24, max_id=24 | count=25, max_id=25 | **+1 row** | INSERT |
| `alias_mailbox` | count=1, max_id=1 | count=1, max_id=1 | 0 | — (not touched) |
| `sync_event` | count=0 | count=0 | 0 | — (not touched; see §6) |

**Two-run stability (OBSERVED):** both runs produced the identical delta — `alias +1, alias_audit_log +1, alias_mailbox +0, daily_metric.nb_alias +1`.

The whole operation is ONE `BEGIN..COMMIT` on backend PID 36580. The complete, unedited 73-line log slice for run 1 is below **in full** (this is the entire single `POST /dashboard/` — the create transaction `BEGIN`→`COMMIT`, then a second read-only `BEGIN`→`ROLLBACK` that reloads the alias/user for the flash message at request teardown). All 19 `statement:` lines are shown verbatim with their `duration:` lines and full column-projection lists; the three **write** statements are `INSERT alias`, `INSERT alias_audit_log`, and `UPDATE daily_metric`:

```
2026-07-14 13:01:36.399 UTC [36580] myuser@simplelogin LOG:  statement: BEGIN
2026-07-14 13:01:36.399 UTC [36580] myuser@simplelogin LOG:  duration: 0.069 ms
2026-07-14 13:01:36.399 UTC [36580] myuser@simplelogin LOG:  statement: SELECT users.directory_quota AS users_directory_quota, users.subdomain_quota AS users_subdomain_quota, users.password AS users_password, users.id AS users_id, users.created_at AS users_created_at, users.updated_at AS users_updated_at, users.email AS users_email, users.name AS users_name, users.is_admin AS users_is_admin, users.alias_generator AS users_alias_generator, users.notification AS users_notification, users.activated AS users_activated, users.disabled AS users_disabled, users.profile_picture_id AS users_profile_picture_id, users.otp_secret AS users_otp_secret, users.enable_otp AS users_enable_otp, users.last_otp AS users_last_otp, users.fido_uuid AS users_fido_uuid, users.default_alias_custom_domain_id AS users_default_alias_custom_domain_id, users.default_alias_public_domain_id AS users_default_alias_public_domain_id, users.lifetime AS users_lifetime, users.paid_lifetime AS users_paid_lifetime, users.lifetime_coupon_id AS users_lifetime_coupon_id, users.trial_end AS users_trial_end, users.default_mailbox_id AS users_default_mailbox_id, users.sender_format AS users_sender_format, users.sender_format_updated_at AS users_sender_format_updated_at, users.replace_reverse_alias AS users_replace_reverse_alias, users.referral_id AS users_referral_id, users.intro_shown AS users_intro_shown, users.max_spam_score AS users_max_spam_score, users.newsletter_alias_id AS users_newsletter_alias_id, users.include_sender_in_reverse_alias AS users_include_sender_in_reverse_alias, users.random_alias_suffix AS users_random_alias_suffix, users.expand_alias_info AS users_expand_alias_info, users.ignore_loop_email AS users_ignore_loop_email, users.alternative_id AS users_alternative_id, users.disable_automatic_alias_note AS users_disable_automatic_alias_note, users.one_click_unsubscribe_block_sender AS users_one_click_unsubscribe_block_sender, users.include_website_in_one_click_alias AS users_include_website_in_one_click_alias, users.disable_import AS users_disable_import, users.can_use_phone AS users_can_use_phone, users.phone_quota AS users_phone_quota, users.block_behaviour AS users_block_behaviour, users.include_header_email_header AS users_include_header_email_header, users.enable_data_breach_check AS users_enable_data_breach_check, users.flags AS users_flags, users.unsub_behaviour AS users_unsub_behaviour, users.delete_on AS users_delete_on
	FROM users
	WHERE users.alternative_id = '6aba9f6b-4be2-4a5a-880b-38fec0e8b328'
	 LIMIT 1
2026-07-14 13:01:36.399 UTC [36580] myuser@simplelogin LOG:  duration: 0.389 ms
2026-07-14 13:01:36.403 UTC [36580] myuser@simplelogin LOG:  statement: SELECT subscription.id AS subscription_id_1, subscription.created_at AS subscription_created_at, subscription.updated_at AS subscription_updated_at, subscription.cancel_url AS subscription_cancel_url, subscription.update_url AS subscription_update_url, subscription.subscription_id AS subscription_subscription_id, subscription.event_time AS subscription_event_time, subscription.next_bill_date AS subscription_next_bill_date, subscription.cancelled AS subscription_cancelled, subscription.plan AS subscription_plan, subscription.user_id AS subscription_user_id
	FROM subscription
	WHERE subscription.user_id = 1
	 LIMIT 1
2026-07-14 13:01:36.403 UTC [36580] myuser@simplelogin LOG:  duration: 0.167 ms
2026-07-14 13:01:36.411 UTC [36580] myuser@simplelogin LOG:  statement: SELECT anon_1.alias_id AS anon_1_alias_id, anon_1.alias_created_at AS anon_1_alias_created_at, anon_1.alias_updated_at AS anon_1_alias_updated_at, anon_1.alias_user_id AS anon_1_alias_user_id, anon_1.alias_email AS anon_1_alias_email, anon_1.alias_name AS anon_1_alias_name, anon_1.alias_enabled AS anon_1_alias_enabled, anon_1.alias_flags AS anon_1_alias_flags, anon_1.alias_custom_domain_id AS anon_1_alias_custom_domain_id, anon_1.alias_automatic_creation AS anon_1_alias_automatic_creation, anon_1.alias_directory_id AS anon_1_alias_directory_id, anon_1.alias_note AS anon_1_alias_note, anon_1.alias_mailbox_id AS anon_1_alias_mailbox_id, anon_1.alias_disable_pgp AS anon_1_alias_disable_pgp, anon_1.alias_cannot_be_disabled AS anon_1_alias_cannot_be_disabled, anon_1.alias_disable_email_spoofing_check AS anon_1_alias_disable_email_spoofing_check, anon_1.alias_batch_import_id AS anon_1_alias_batch_import_id, anon_1.alias_original_owner_id AS anon_1_alias_original_owner_id, anon_1.alias_pinned AS anon_1_alias_pinned, anon_1.alias_transfer_token AS anon_1_alias_transfer_token, anon_1.alias_transfer_token_expiration AS anon_1_alias_transfer_token_expiration, anon_1.alias_hibp_last_check AS anon_1_alias_hibp_last_check, anon_1.alias_ts_vector AS anon_1_alias_ts_vector, anon_1.alias_last_email_log_id AS anon_1_alias_last_email_log_id, mailbox_1.id AS mailbox_1_id, mailbox_1.created_at AS mailbox_1_created_at, mailbox_1.updated_at AS mailbox_1_updated_at, mailbox_1.user_id AS mailbox_1_user_id, mailbox_1.email AS mailbox_1_email, mailbox_1.verified AS mailbox_1_verified, mailbox_1.force_spf AS mailbox_1_force_spf, mailbox_1.new_email AS mailbox_1_new_email, mailbox_1.pgp_public_key AS mailbox_1_pgp_public_key, mailbox_1.pgp_finger_print AS mailbox_1_pgp_finger_print, mailbox_1.disable_pgp AS mailbox_1_disable_pgp, mailbox_1.nb_failed_checks AS mailbox_1_nb_failed_checks, mailbox_1.disabled AS mailbox_1_disabled, mailbox_1.generic_subject AS mailbox_1_generic_subject, mailbox_2.id AS mailbox_2_id, mailbox_2.created_at AS mailbox_2_created_at, mailbox_2.updated_at AS mailbox_2_updated_at, mailbox_2.user_id AS mailbox_2_user_id, mailbox_2.email AS mailbox_2_email, mailbox_2.verified AS mailbox_2_verified, mailbox_2.force_spf AS mailbox_2_force_spf, mailbox_2.new_email AS mailbox_2_new_email, mailbox_2.pgp_public_key AS mailbox_2_pgp_public_key, mailbox_2.pgp_finger_print AS mailbox_2_pgp_finger_print, mailbox_2.disable_pgp AS mailbox_2_disable_pgp, mailbox_2.nb_failed_checks AS mailbox_2_nb_failed_checks, mailbox_2.disabled AS mailbox_2_disabled, mailbox_2.generic_subject AS mailbox_2_generic_subject
	FROM (SELECT alias.id AS alias_id, alias.created_at AS alias_created_at, alias.updated_at AS alias_updated_at, alias.user_id AS alias_user_id, alias.email AS alias_email, alias.name AS alias_name, alias.enabled AS alias_enabled, alias.flags AS alias_flags, alias.custom_domain_id AS alias_custom_domain_id, alias.automatic_creation AS alias_automatic_creation, alias.directory_id AS alias_directory_id, alias.note AS alias_note, alias.mailbox_id AS alias_mailbox_id, alias.disable_pgp AS alias_disable_pgp, alias.cannot_be_disabled AS alias_cannot_be_disabled, alias.disable_email_spoofing_check AS alias_disable_email_spoofing_check, alias.batch_import_id AS alias_batch_import_id, alias.original_owner_id AS alias_original_owner_id, alias.pinned AS alias_pinned, alias.transfer_token AS alias_transfer_token, alias.transfer_token_expiration AS alias_transfer_token_expiration, alias.hibp_last_check AS alias_hibp_last_check, alias.ts_vector AS alias_ts_vector, alias.last_email_log_id AS alias_last_email_log_id
	FROM alias
	WHERE alias.email = 'misers_amnion824@sl.local'
	 LIMIT 1) AS anon_1 LEFT OUTER JOIN (alias_mailbox AS alias_mailbox_1 JOIN mailbox AS mailbox_1 ON mailbox_1.id = alias_mailbox_1.mailbox_id) ON anon_1.alias_id = alias_mailbox_1.alias_id LEFT OUTER JOIN mailbox AS mailbox_2 ON mailbox_2.id = anon_1.alias_mailbox_id
2026-07-14 13:01:36.412 UTC [36580] myuser@simplelogin LOG:  duration: 0.668 ms
2026-07-14 13:01:36.415 UTC [36580] myuser@simplelogin LOG:  statement: SELECT contact.id AS contact_id, contact.created_at AS contact_created_at, contact.updated_at AS contact_updated_at, contact.user_id AS contact_user_id, contact.alias_id AS contact_alias_id, contact.name AS contact_name, contact.website_email AS contact_website_email, contact.website_from AS contact_website_from, contact.reply_email AS contact_reply_email, contact.is_cc AS contact_is_cc, contact.pgp_public_key AS contact_pgp_public_key, contact.pgp_finger_print AS contact_pgp_finger_print, contact.mail_from AS contact_mail_from, contact.invalid_email AS contact_invalid_email, contact.block_forward AS contact_block_forward, contact.automatic_created AS contact_automatic_created, contact.flags AS contact_flags
	FROM contact
	WHERE contact.reply_email = 'misers_amnion824@sl.local'
	 LIMIT 1
2026-07-14 13:01:36.415 UTC [36580] myuser@simplelogin LOG:  duration: 0.160 ms
2026-07-14 13:01:36.417 UTC [36580] myuser@simplelogin LOG:  statement: SELECT deleted_alias.id AS deleted_alias_id, deleted_alias.created_at AS deleted_alias_created_at, deleted_alias.updated_at AS deleted_alias_updated_at, deleted_alias.email AS deleted_alias_email, deleted_alias.reason AS deleted_alias_reason
	FROM deleted_alias
	WHERE deleted_alias.email = 'misers_amnion824@sl.local'
	 LIMIT 1
2026-07-14 13:01:36.417 UTC [36580] myuser@simplelogin LOG:  duration: 0.348 ms
2026-07-14 13:01:36.419 UTC [36580] myuser@simplelogin LOG:  statement: SELECT subscription.id AS subscription_id_1, subscription.created_at AS subscription_created_at, subscription.updated_at AS subscription_updated_at, subscription.cancel_url AS subscription_cancel_url, subscription.update_url AS subscription_update_url, subscription.subscription_id AS subscription_subscription_id, subscription.event_time AS subscription_event_time, subscription.next_bill_date AS subscription_next_bill_date, subscription.cancelled AS subscription_cancelled, subscription.plan AS subscription_plan, subscription.user_id AS subscription_user_id
	FROM subscription
	WHERE subscription.user_id = 1
	 LIMIT 1
2026-07-14 13:01:36.419 UTC [36580] myuser@simplelogin LOG:  duration: 0.117 ms
2026-07-14 13:01:36.420 UTC [36580] myuser@simplelogin LOG:  statement: SELECT deleted_alias.id AS deleted_alias_id, deleted_alias.created_at AS deleted_alias_created_at, deleted_alias.updated_at AS deleted_alias_updated_at, deleted_alias.email AS deleted_alias_email, deleted_alias.reason AS deleted_alias_reason
	FROM deleted_alias
	WHERE deleted_alias.email = 'misers_amnion824@sl.local'
	 LIMIT 1
2026-07-14 13:01:36.420 UTC [36580] myuser@simplelogin LOG:  duration: 0.101 ms
2026-07-14 13:01:36.421 UTC [36580] myuser@simplelogin LOG:  statement: SELECT domain_deleted_alias.id AS domain_deleted_alias_id, domain_deleted_alias.created_at AS domain_deleted_alias_created_at, domain_deleted_alias.updated_at AS domain_deleted_alias_updated_at, domain_deleted_alias.email AS domain_deleted_alias_email, domain_deleted_alias.domain_id AS domain_deleted_alias_domain_id, domain_deleted_alias.user_id AS domain_deleted_alias_user_id, domain_deleted_alias.reason AS domain_deleted_alias_reason
	FROM domain_deleted_alias
	WHERE domain_deleted_alias.email = 'misers_amnion824@sl.local'
	 LIMIT 1
2026-07-14 13:01:36.422 UTC [36580] myuser@simplelogin LOG:  duration: 0.281 ms
2026-07-14 13:01:36.427 UTC [36580] myuser@simplelogin LOG:  statement: SELECT public_domain.id AS public_domain_id, public_domain.created_at AS public_domain_created_at, public_domain.updated_at AS public_domain_updated_at, public_domain.domain AS public_domain_domain, public_domain.premium_only AS public_domain_premium_only, public_domain.can_use_subdomain AS public_domain_can_use_subdomain, public_domain.partner_id AS public_domain_partner_id, public_domain.hidden AS public_domain_hidden, public_domain."order" AS public_domain_order, public_domain.use_as_reverse_alias AS public_domain_use_as_reverse_alias
	FROM public_domain
	WHERE public_domain.domain = 'sl.local'
	 LIMIT 1
2026-07-14 13:01:36.427 UTC [36580] myuser@simplelogin LOG:  duration: 0.127 ms
2026-07-14 13:01:36.431 UTC [36580] myuser@simplelogin LOG:  statement: INSERT INTO alias (created_at, updated_at, user_id, email, name, enabled, flags, custom_domain_id, automatic_creation, directory_id, note, mailbox_id, disable_pgp, cannot_be_disabled, disable_email_spoofing_check, batch_import_id, original_owner_id, pinned, transfer_token, transfer_token_expiration, hibp_last_check, last_email_log_id) VALUES ('2026-07-14T13:01:36.431098'::timestamp, NULL, 1, 'misers_amnion824@sl.local', NULL, true, 0, NULL, false, NULL, NULL, 1, false, false, false, NULL, NULL, false, NULL, '2026-07-14T13:01:36.431135'::timestamp, NULL, NULL) RETURNING alias.id
2026-07-14 13:01:36.433 UTC [36580] myuser@simplelogin LOG:  duration: 1.759 ms
2026-07-14 13:01:36.434 UTC [36580] myuser@simplelogin LOG:  statement: SELECT daily_metric.id AS daily_metric_id, daily_metric.created_at AS daily_metric_created_at, daily_metric.updated_at AS daily_metric_updated_at, daily_metric.date AS daily_metric_date, daily_metric.nb_new_web_non_proton_user AS daily_metric_nb_new_web_non_proton_user, daily_metric.nb_alias AS daily_metric_nb_alias
	FROM daily_metric
	WHERE daily_metric.date = '2026-07-14'::date
	 LIMIT 1
2026-07-14 13:01:36.435 UTC [36580] myuser@simplelogin LOG:  duration: 0.388 ms
2026-07-14 13:01:36.437 UTC [36580] myuser@simplelogin LOG:  statement: INSERT INTO alias_audit_log (created_at, updated_at, user_id, alias_id, alias_email, action, message) VALUES ('2026-07-14T13:01:36.437413'::timestamp, NULL, 1, 25, 'misers_amnion824@sl.local', 'create', 'New alias created') RETURNING alias_audit_log.id
2026-07-14 13:01:36.437 UTC [36580] myuser@simplelogin LOG:  duration: 0.373 ms
2026-07-14 13:01:36.438 UTC [36580] myuser@simplelogin LOG:  statement: UPDATE daily_metric SET updated_at='2026-07-14T13:01:36.438722'::timestamp, nb_alias=25 WHERE daily_metric.id = 1
2026-07-14 13:01:36.439 UTC [36580] myuser@simplelogin LOG:  duration: 0.145 ms
2026-07-14 13:01:36.439 UTC [36580] myuser@simplelogin LOG:  statement: COMMIT
2026-07-14 13:01:36.440 UTC [36580] myuser@simplelogin LOG:  duration: 0.987 ms
2026-07-14 13:01:36.441 UTC [36580] myuser@simplelogin LOG:  statement: BEGIN
2026-07-14 13:01:36.441 UTC [36580] myuser@simplelogin LOG:  duration: 0.019 ms
2026-07-14 13:01:36.441 UTC [36580] myuser@simplelogin LOG:  statement: SELECT alias.id AS alias_id, alias.created_at AS alias_created_at, alias.updated_at AS alias_updated_at, alias.user_id AS alias_user_id, alias.email AS alias_email, alias.name AS alias_name, alias.enabled AS alias_enabled, alias.flags AS alias_flags, alias.custom_domain_id AS alias_custom_domain_id, alias.automatic_creation AS alias_automatic_creation, alias.directory_id AS alias_directory_id, alias.note AS alias_note, alias.mailbox_id AS alias_mailbox_id, alias.disable_pgp AS alias_disable_pgp, alias.cannot_be_disabled AS alias_cannot_be_disabled, alias.disable_email_spoofing_check AS alias_disable_email_spoofing_check, alias.batch_import_id AS alias_batch_import_id, alias.original_owner_id AS alias_original_owner_id, alias.pinned AS alias_pinned, alias.transfer_token AS alias_transfer_token, alias.transfer_token_expiration AS alias_transfer_token_expiration, alias.hibp_last_check AS alias_hibp_last_check, alias.ts_vector AS alias_ts_vector, alias.last_email_log_id AS alias_last_email_log_id
	FROM alias
	WHERE alias.id = 25
2026-07-14 13:01:36.442 UTC [36580] myuser@simplelogin LOG:  duration: 0.132 ms
2026-07-14 13:01:36.444 UTC [36580] myuser@simplelogin LOG:  statement: SELECT users.directory_quota AS users_directory_quota, users.subdomain_quota AS users_subdomain_quota, users.password AS users_password, users.id AS users_id, users.created_at AS users_created_at, users.updated_at AS users_updated_at, users.email AS users_email, users.name AS users_name, users.is_admin AS users_is_admin, users.alias_generator AS users_alias_generator, users.notification AS users_notification, users.activated AS users_activated, users.disabled AS users_disabled, users.profile_picture_id AS users_profile_picture_id, users.otp_secret AS users_otp_secret, users.enable_otp AS users_enable_otp, users.last_otp AS users_last_otp, users.fido_uuid AS users_fido_uuid, users.default_alias_custom_domain_id AS users_default_alias_custom_domain_id, users.default_alias_public_domain_id AS users_default_alias_public_domain_id, users.lifetime AS users_lifetime, users.paid_lifetime AS users_paid_lifetime, users.lifetime_coupon_id AS users_lifetime_coupon_id, users.trial_end AS users_trial_end, users.default_mailbox_id AS users_default_mailbox_id, users.sender_format AS users_sender_format, users.sender_format_updated_at AS users_sender_format_updated_at, users.replace_reverse_alias AS users_replace_reverse_alias, users.referral_id AS users_referral_id, users.intro_shown AS users_intro_shown, users.max_spam_score AS users_max_spam_score, users.newsletter_alias_id AS users_newsletter_alias_id, users.include_sender_in_reverse_alias AS users_include_sender_in_reverse_alias, users.random_alias_suffix AS users_random_alias_suffix, users.expand_alias_info AS users_expand_alias_info, users.ignore_loop_email AS users_ignore_loop_email, users.alternative_id AS users_alternative_id, users.disable_automatic_alias_note AS users_disable_automatic_alias_note, users.one_click_unsubscribe_block_sender AS users_one_click_unsubscribe_block_sender, users.include_website_in_one_click_alias AS users_include_website_in_one_click_alias, users.disable_import AS users_disable_import, users.can_use_phone AS users_can_use_phone, users.phone_quota AS users_phone_quota, users.block_behaviour AS users_block_behaviour, users.include_header_email_header AS users_include_header_email_header, users.enable_data_breach_check AS users_enable_data_breach_check, users.flags AS users_flags, users.unsub_behaviour AS users_unsub_behaviour, users.delete_on AS users_delete_on
	FROM users
	WHERE users.id = 1
2026-07-14 13:01:36.444 UTC [36580] myuser@simplelogin LOG:  duration: 0.184 ms
2026-07-14 13:01:36.446 UTC [36580] myuser@simplelogin LOG:  statement: ROLLBACK
2026-07-14 13:01:36.446 UTC [36580] myuser@simplelogin LOG:  duration: 0.031 ms
```

> The slice above is the complete PostgreSQL `log_statement=all` output for run 1, verbatim — all 19 `statement:` lines, all 19 `duration:` lines, and the full ~40-column `SELECT` projection lists are included with **no elision**. The **only** normalization applied is stripping end-of-line whitespace from the tab-indented `FROM`/`WHERE` continuation lines that SQLAlchemy emits (a purely cosmetic trailing-space removal that changes no SQL token). **Note:** there is **no `SELECT count(*) FROM alias`** (quota) statement — `john` is a premium user, so `User.can_create_new_alias` (`app/models.py:867`) short-circuits without counting; the two `subscription` SELECTs are the premium checks.

### 5.3 CUSTOM alias with 2 mailboxes — 4 tables

**Producing command:** `POST /dashboard/custom_alias` with body `csrf_token=<…>&prefix=obscap…&signed-alias-suffix=<…>&mailboxes=1&mailboxes=2&note=` (see §3.3), as `john@wick.com` — two mailboxes selected (id 1 = `john@wick.com`, id 2 = `pgp@example.org`).

**Before/during/after snapshot** (psycopg2, two runs; OBSERVED). Run 1 created alias id=27 (`obscap169773@old.com`); run 2 created id=28 (`obscap269774@old.com`):

| Table | BEFORE (run 1) | AFTER (run 1) | Delta | Operation |
|-------|----------------|---------------|-------|-----------|
| `alias` | count=26, max_id=26 | count=27, max_id=27 | **+1 row** | INSERT (`custom_domain_id=2`, `note=''`) |
| `daily_metric` | id=1, nb_alias=**26** | id=1, nb_alias=**27** | **+0 rows / nb_alias +1** | UPDATE |
| `alias_audit_log` | count=26, max_id=26 | count=27, max_id=27 | **+1 row** | INSERT |
| `alias_mailbox` | count=1, max_id=1 | count=2, max_id=2 | **+1 row** | INSERT (secondary mailbox only) |
| `sync_event` | count=0 | count=0 | 0 | — (not touched; see §6) |

**Two-run stability (OBSERVED):** both runs produced the identical delta — `alias +1, alias_audit_log +1, alias_mailbox +1, daily_metric.nb_alias +1`.

**Crucial fact — the "2 mailboxes" produce only ONE `alias_mailbox` row.** The **primary** mailbox (`mailboxes[0]`, id=1) is stored on the `alias.mailbox_id` column; only the **secondary** mailbox (id=2) becomes an `alias_mailbox` row. So *N* selected mailboxes ⇒ 1 primary on `alias` + (*N*−1) `alias_mailbox` rows. The primary is passed into `Alias.create`; the secondaries are looped into `alias_mailbox`. The complete block — including its `IntegrityError` guard (the B6 rollback path documented in §7), the `Session.commit()`, and the success flash + redirect — is `app/dashboard/views/custom_alias.py:138-161`, verbatim and unedited:

```python
                try:
                    alias = Alias.create(
                        user_id=current_user.id,
                        email=full_alias,
                        note=alias_note,
                        mailbox_id=mailboxes[0].id,
                    )
                    Session.flush()
                except IntegrityError:
                    LOG.w("Alias %s already exists", full_alias)
                    Session.rollback()
                    flash("Unknown error, please retry", "error")
                    return redirect(url_for("dashboard.custom_alias"))

                for i in range(1, len(mailboxes)):
                    AliasMailbox.create(
                        alias_id=alias.id,
                        mailbox_id=mailboxes[i].id,
                    )

                Session.commit()
                flash(f"Alias {full_alias} has been created", "success")

                return redirect(url_for("dashboard.index", highlight_alias_id=alias.id))
```

The complete, unedited 142-line log slice for run 1 is below **in full** (the create transaction `BEGIN`→`COMMIT` on backend PID 36580, then the read-only `BEGIN`→`ROLLBACK` reload). All 35 `statement:` lines are shown verbatim with their `duration:` lines and full column-projection lists; the **four write** statements are `INSERT alias`, `UPDATE daily_metric`, `INSERT alias_audit_log`, and `INSERT alias_mailbox` (same end-of-line-whitespace normalization as §5.2, no SQL token changed):

```
2026-07-14 13:03:27.322 UTC [36580] myuser@simplelogin LOG:  statement: BEGIN
2026-07-14 13:03:27.322 UTC [36580] myuser@simplelogin LOG:  duration: 0.059 ms
2026-07-14 13:03:27.322 UTC [36580] myuser@simplelogin LOG:  statement: SELECT users.directory_quota AS users_directory_quota, users.subdomain_quota AS users_subdomain_quota, users.password AS users_password, users.id AS users_id, users.created_at AS users_created_at, users.updated_at AS users_updated_at, users.email AS users_email, users.name AS users_name, users.is_admin AS users_is_admin, users.alias_generator AS users_alias_generator, users.notification AS users_notification, users.activated AS users_activated, users.disabled AS users_disabled, users.profile_picture_id AS users_profile_picture_id, users.otp_secret AS users_otp_secret, users.enable_otp AS users_enable_otp, users.last_otp AS users_last_otp, users.fido_uuid AS users_fido_uuid, users.default_alias_custom_domain_id AS users_default_alias_custom_domain_id, users.default_alias_public_domain_id AS users_default_alias_public_domain_id, users.lifetime AS users_lifetime, users.paid_lifetime AS users_paid_lifetime, users.lifetime_coupon_id AS users_lifetime_coupon_id, users.trial_end AS users_trial_end, users.default_mailbox_id AS users_default_mailbox_id, users.sender_format AS users_sender_format, users.sender_format_updated_at AS users_sender_format_updated_at, users.replace_reverse_alias AS users_replace_reverse_alias, users.referral_id AS users_referral_id, users.intro_shown AS users_intro_shown, users.max_spam_score AS users_max_spam_score, users.newsletter_alias_id AS users_newsletter_alias_id, users.include_sender_in_reverse_alias AS users_include_sender_in_reverse_alias, users.random_alias_suffix AS users_random_alias_suffix, users.expand_alias_info AS users_expand_alias_info, users.ignore_loop_email AS users_ignore_loop_email, users.alternative_id AS users_alternative_id, users.disable_automatic_alias_note AS users_disable_automatic_alias_note, users.one_click_unsubscribe_block_sender AS users_one_click_unsubscribe_block_sender, users.include_website_in_one_click_alias AS users_include_website_in_one_click_alias, users.disable_import AS users_disable_import, users.can_use_phone AS users_can_use_phone, users.phone_quota AS users_phone_quota, users.block_behaviour AS users_block_behaviour, users.include_header_email_header AS users_include_header_email_header, users.enable_data_breach_check AS users_enable_data_breach_check, users.flags AS users_flags, users.unsub_behaviour AS users_unsub_behaviour, users.delete_on AS users_delete_on
	FROM users
	WHERE users.alternative_id = '6aba9f6b-4be2-4a5a-880b-38fec0e8b328'
	 LIMIT 1
2026-07-14 13:03:27.322 UTC [36580] myuser@simplelogin LOG:  duration: 0.305 ms
2026-07-14 13:03:27.325 UTC [36580] myuser@simplelogin LOG:  statement: SELECT subscription.id AS subscription_id_1, subscription.created_at AS subscription_created_at, subscription.updated_at AS subscription_updated_at, subscription.cancel_url AS subscription_cancel_url, subscription.update_url AS subscription_update_url, subscription.subscription_id AS subscription_subscription_id, subscription.event_time AS subscription_event_time, subscription.next_bill_date AS subscription_next_bill_date, subscription.cancelled AS subscription_cancelled, subscription.plan AS subscription_plan, subscription.user_id AS subscription_user_id
	FROM subscription
	WHERE subscription.user_id = 1
	 LIMIT 1
2026-07-14 13:03:27.325 UTC [36580] myuser@simplelogin LOG:  duration: 0.113 ms
2026-07-14 13:03:27.330 UTC [36580] myuser@simplelogin LOG:  statement: SELECT custom_domain.id AS custom_domain_id, custom_domain.created_at AS custom_domain_created_at, custom_domain.updated_at AS custom_domain_updated_at, custom_domain.user_id AS custom_domain_user_id, custom_domain.domain AS custom_domain_domain, custom_domain.name AS custom_domain_name, custom_domain.verified AS custom_domain_verified, custom_domain.dkim_verified AS custom_domain_dkim_verified, custom_domain.spf_verified AS custom_domain_spf_verified, custom_domain.dmarc_verified AS custom_domain_dmarc_verified, custom_domain.catch_all AS custom_domain_catch_all, custom_domain.random_prefix_generation AS custom_domain_random_prefix_generation, custom_domain.nb_failed_checks AS custom_domain_nb_failed_checks, custom_domain.ownership_verified AS custom_domain_ownership_verified, custom_domain.ownership_txt_token AS custom_domain_ownership_txt_token, custom_domain.is_sl_subdomain AS custom_domain_is_sl_subdomain, custom_domain.partner_id AS custom_domain_partner_id, custom_domain.pending_deletion AS custom_domain_pending_deletion, mailbox_1.id AS mailbox_1_id, mailbox_1.created_at AS mailbox_1_created_at, mailbox_1.updated_at AS mailbox_1_updated_at, mailbox_1.user_id AS mailbox_1_user_id, mailbox_1.email AS mailbox_1_email, mailbox_1.verified AS mailbox_1_verified, mailbox_1.force_spf AS mailbox_1_force_spf, mailbox_1.new_email AS mailbox_1_new_email, mailbox_1.pgp_public_key AS mailbox_1_pgp_public_key, mailbox_1.pgp_finger_print AS mailbox_1_pgp_finger_print, mailbox_1.disable_pgp AS mailbox_1_disable_pgp, mailbox_1.nb_failed_checks AS mailbox_1_nb_failed_checks, mailbox_1.disabled AS mailbox_1_disabled, mailbox_1.generic_subject AS mailbox_1_generic_subject
	FROM custom_domain LEFT OUTER JOIN (domain_mailbox AS domain_mailbox_1 JOIN mailbox AS mailbox_1 ON mailbox_1.id = domain_mailbox_1.mailbox_id) ON custom_domain.id = domain_mailbox_1.domain_id
	WHERE custom_domain.user_id = 1 AND custom_domain.ownership_verified = true ORDER BY custom_domain.domain ASC
2026-07-14 13:03:27.330 UTC [36580] myuser@simplelogin LOG:  duration: 0.440 ms
2026-07-14 13:03:27.334 UTC [36580] myuser@simplelogin LOG:  statement: SELECT custom_domain.id AS custom_domain_id, custom_domain.created_at AS custom_domain_created_at, custom_domain.updated_at AS custom_domain_updated_at, custom_domain.user_id AS custom_domain_user_id, custom_domain.domain AS custom_domain_domain, custom_domain.name AS custom_domain_name, custom_domain.verified AS custom_domain_verified, custom_domain.dkim_verified AS custom_domain_dkim_verified, custom_domain.spf_verified AS custom_domain_spf_verified, custom_domain.dmarc_verified AS custom_domain_dmarc_verified, custom_domain.catch_all AS custom_domain_catch_all, custom_domain.random_prefix_generation AS custom_domain_random_prefix_generation, custom_domain.nb_failed_checks AS custom_domain_nb_failed_checks, custom_domain.ownership_verified AS custom_domain_ownership_verified, custom_domain.ownership_txt_token AS custom_domain_ownership_txt_token, custom_domain.is_sl_subdomain AS custom_domain_is_sl_subdomain, custom_domain.partner_id AS custom_domain_partner_id, custom_domain.pending_deletion AS custom_domain_pending_deletion, mailbox_1.id AS mailbox_1_id, mailbox_1.created_at AS mailbox_1_created_at, mailbox_1.updated_at AS mailbox_1_updated_at, mailbox_1.user_id AS mailbox_1_user_id, mailbox_1.email AS mailbox_1_email, mailbox_1.verified AS mailbox_1_verified, mailbox_1.force_spf AS mailbox_1_force_spf, mailbox_1.new_email AS mailbox_1_new_email, mailbox_1.pgp_public_key AS mailbox_1_pgp_public_key, mailbox_1.pgp_finger_print AS mailbox_1_pgp_finger_print, mailbox_1.disable_pgp AS mailbox_1_disable_pgp, mailbox_1.nb_failed_checks AS mailbox_1_nb_failed_checks, mailbox_1.disabled AS mailbox_1_disabled, mailbox_1.generic_subject AS mailbox_1_generic_subject
	FROM custom_domain LEFT OUTER JOIN (domain_mailbox AS domain_mailbox_1 JOIN mailbox AS mailbox_1 ON mailbox_1.id = domain_mailbox_1.mailbox_id) ON custom_domain.id = domain_mailbox_1.domain_id
	WHERE custom_domain.user_id = 1 AND custom_domain.ownership_verified = true ORDER BY custom_domain.domain ASC
2026-07-14 13:03:27.334 UTC [36580] myuser@simplelogin LOG:  duration: 0.314 ms
2026-07-14 13:03:27.335 UTC [36580] myuser@simplelogin LOG:  statement: SELECT subscription.id AS subscription_id_1, subscription.created_at AS subscription_created_at, subscription.updated_at AS subscription_updated_at, subscription.cancel_url AS subscription_cancel_url, subscription.update_url AS subscription_update_url, subscription.subscription_id AS subscription_subscription_id, subscription.event_time AS subscription_event_time, subscription.next_bill_date AS subscription_next_bill_date, subscription.cancelled AS subscription_cancelled, subscription.plan AS subscription_plan, subscription.user_id AS subscription_user_id
	FROM subscription
	WHERE subscription.user_id = 1
	 LIMIT 1
2026-07-14 13:03:27.335 UTC [36580] myuser@simplelogin LOG:  duration: 0.098 ms
2026-07-14 13:03:27.337 UTC [36580] myuser@simplelogin LOG:  statement: SELECT public_domain.id AS public_domain_id, public_domain.created_at AS public_domain_created_at, public_domain.updated_at AS public_domain_updated_at, public_domain.domain AS public_domain_domain, public_domain.premium_only AS public_domain_premium_only, public_domain.can_use_subdomain AS public_domain_can_use_subdomain, public_domain.partner_id AS public_domain_partner_id, public_domain.hidden AS public_domain_hidden, public_domain."order" AS public_domain_order, public_domain.use_as_reverse_alias AS public_domain_use_as_reverse_alias
	FROM public_domain
	WHERE public_domain.hidden = false AND (public_domain.partner_id IS NULL) ORDER BY public_domain."order"
2026-07-14 13:03:27.337 UTC [36580] myuser@simplelogin LOG:  duration: 0.104 ms
2026-07-14 13:03:27.338 UTC [36580] myuser@simplelogin LOG:  statement: SELECT subscription.id AS subscription_id_1, subscription.created_at AS subscription_created_at, subscription.updated_at AS subscription_updated_at, subscription.cancel_url AS subscription_cancel_url, subscription.update_url AS subscription_update_url, subscription.subscription_id AS subscription_subscription_id, subscription.event_time AS subscription_event_time, subscription.next_bill_date AS subscription_next_bill_date, subscription.cancelled AS subscription_cancelled, subscription.plan AS subscription_plan, subscription.user_id AS subscription_user_id
	FROM subscription
	WHERE subscription.user_id = 1
	 LIMIT 1
2026-07-14 13:03:27.338 UTC [36580] myuser@simplelogin LOG:  duration: 0.077 ms
2026-07-14 13:03:27.339 UTC [36580] myuser@simplelogin LOG:  statement: SELECT public_domain.id AS public_domain_id, public_domain.created_at AS public_domain_created_at, public_domain.updated_at AS public_domain_updated_at, public_domain.domain AS public_domain_domain, public_domain.premium_only AS public_domain_premium_only, public_domain.can_use_subdomain AS public_domain_can_use_subdomain, public_domain.partner_id AS public_domain_partner_id, public_domain.hidden AS public_domain_hidden, public_domain."order" AS public_domain_order, public_domain.use_as_reverse_alias AS public_domain_use_as_reverse_alias
	FROM public_domain
	WHERE public_domain.id IS NULL AND public_domain.hidden = false
	 LIMIT 1
2026-07-14 13:03:27.339 UTC [36580] myuser@simplelogin LOG:  duration: 0.073 ms
2026-07-14 13:03:27.341 UTC [36580] myuser@simplelogin LOG:  statement: SELECT mailbox.id AS mailbox_id, mailbox.created_at AS mailbox_created_at, mailbox.updated_at AS mailbox_updated_at, mailbox.user_id AS mailbox_user_id, mailbox.email AS mailbox_email, mailbox.verified AS mailbox_verified, mailbox.force_spf AS mailbox_force_spf, mailbox.new_email AS mailbox_new_email, mailbox.pgp_public_key AS mailbox_pgp_public_key, mailbox.pgp_finger_print AS mailbox_pgp_finger_print, mailbox.disable_pgp AS mailbox_disable_pgp, mailbox.nb_failed_checks AS mailbox_nb_failed_checks, mailbox.disabled AS mailbox_disabled, mailbox.generic_subject AS mailbox_generic_subject
	FROM mailbox
	WHERE mailbox.user_id = 1 AND mailbox.verified = true
2026-07-14 13:03:27.341 UTC [36580] myuser@simplelogin LOG:  duration: 0.100 ms
2026-07-14 13:03:27.343 UTC [36580] myuser@simplelogin LOG:  statement: SELECT mailbox.id AS mailbox_id, mailbox.created_at AS mailbox_created_at, mailbox.updated_at AS mailbox_updated_at, mailbox.user_id AS mailbox_user_id, mailbox.email AS mailbox_email, mailbox.verified AS mailbox_verified, mailbox.force_spf AS mailbox_force_spf, mailbox.new_email AS mailbox_new_email, mailbox.pgp_public_key AS mailbox_pgp_public_key, mailbox.pgp_finger_print AS mailbox_pgp_finger_print, mailbox.disable_pgp AS mailbox_disable_pgp, mailbox.nb_failed_checks AS mailbox_nb_failed_checks, mailbox.disabled AS mailbox_disabled, mailbox.generic_subject AS mailbox_generic_subject
	FROM mailbox
	WHERE mailbox.id = '1'
2026-07-14 13:03:27.343 UTC [36580] myuser@simplelogin LOG:  duration: 0.077 ms
2026-07-14 13:03:27.344 UTC [36580] myuser@simplelogin LOG:  statement: SELECT mailbox.id AS mailbox_id, mailbox.created_at AS mailbox_created_at, mailbox.updated_at AS mailbox_updated_at, mailbox.user_id AS mailbox_user_id, mailbox.email AS mailbox_email, mailbox.verified AS mailbox_verified, mailbox.force_spf AS mailbox_force_spf, mailbox.new_email AS mailbox_new_email, mailbox.pgp_public_key AS mailbox_pgp_public_key, mailbox.pgp_finger_print AS mailbox_pgp_finger_print, mailbox.disable_pgp AS mailbox_disable_pgp, mailbox.nb_failed_checks AS mailbox_nb_failed_checks, mailbox.disabled AS mailbox_disabled, mailbox.generic_subject AS mailbox_generic_subject
	FROM mailbox
	WHERE mailbox.id = '2'
2026-07-14 13:03:27.344 UTC [36580] myuser@simplelogin LOG:  duration: 0.076 ms
2026-07-14 13:03:27.347 UTC [36580] myuser@simplelogin LOG:  statement: SELECT custom_domain.id AS custom_domain_id, custom_domain.created_at AS custom_domain_created_at, custom_domain.updated_at AS custom_domain_updated_at, custom_domain.user_id AS custom_domain_user_id, custom_domain.domain AS custom_domain_domain, custom_domain.name AS custom_domain_name, custom_domain.verified AS custom_domain_verified, custom_domain.dkim_verified AS custom_domain_dkim_verified, custom_domain.spf_verified AS custom_domain_spf_verified, custom_domain.dmarc_verified AS custom_domain_dmarc_verified, custom_domain.catch_all AS custom_domain_catch_all, custom_domain.random_prefix_generation AS custom_domain_random_prefix_generation, custom_domain.nb_failed_checks AS custom_domain_nb_failed_checks, custom_domain.ownership_verified AS custom_domain_ownership_verified, custom_domain.ownership_txt_token AS custom_domain_ownership_txt_token, custom_domain.is_sl_subdomain AS custom_domain_is_sl_subdomain, custom_domain.partner_id AS custom_domain_partner_id, custom_domain.pending_deletion AS custom_domain_pending_deletion, mailbox_1.id AS mailbox_1_id, mailbox_1.created_at AS mailbox_1_created_at, mailbox_1.updated_at AS mailbox_1_updated_at, mailbox_1.user_id AS mailbox_1_user_id, mailbox_1.email AS mailbox_1_email, mailbox_1.verified AS mailbox_1_verified, mailbox_1.force_spf AS mailbox_1_force_spf, mailbox_1.new_email AS mailbox_1_new_email, mailbox_1.pgp_public_key AS mailbox_1_pgp_public_key, mailbox_1.pgp_finger_print AS mailbox_1_pgp_finger_print, mailbox_1.disable_pgp AS mailbox_1_disable_pgp, mailbox_1.nb_failed_checks AS mailbox_1_nb_failed_checks, mailbox_1.disabled AS mailbox_1_disabled, mailbox_1.generic_subject AS mailbox_1_generic_subject
	FROM custom_domain LEFT OUTER JOIN (domain_mailbox AS domain_mailbox_1 JOIN mailbox AS mailbox_1 ON mailbox_1.id = domain_mailbox_1.mailbox_id) ON custom_domain.id = domain_mailbox_1.domain_id
	WHERE custom_domain.user_id = 1 AND custom_domain.ownership_verified = true ORDER BY custom_domain.domain ASC
2026-07-14 13:03:27.347 UTC [36580] myuser@simplelogin LOG:  duration: 0.265 ms
2026-07-14 13:03:27.349 UTC [36580] myuser@simplelogin LOG:  statement: SELECT subscription.id AS subscription_id_1, subscription.created_at AS subscription_created_at, subscription.updated_at AS subscription_updated_at, subscription.cancel_url AS subscription_cancel_url, subscription.update_url AS subscription_update_url, subscription.subscription_id AS subscription_subscription_id, subscription.event_time AS subscription_event_time, subscription.next_bill_date AS subscription_next_bill_date, subscription.cancelled AS subscription_cancelled, subscription.plan AS subscription_plan, subscription.user_id AS subscription_user_id
	FROM subscription
	WHERE subscription.user_id = 1
	 LIMIT 1
2026-07-14 13:03:27.349 UTC [36580] myuser@simplelogin LOG:  duration: 0.093 ms
2026-07-14 13:03:27.350 UTC [36580] myuser@simplelogin LOG:  statement: SELECT public_domain.id AS public_domain_id, public_domain.created_at AS public_domain_created_at, public_domain.updated_at AS public_domain_updated_at, public_domain.domain AS public_domain_domain, public_domain.premium_only AS public_domain_premium_only, public_domain.can_use_subdomain AS public_domain_can_use_subdomain, public_domain.partner_id AS public_domain_partner_id, public_domain.hidden AS public_domain_hidden, public_domain."order" AS public_domain_order, public_domain.use_as_reverse_alias AS public_domain_use_as_reverse_alias
	FROM public_domain
	WHERE public_domain.hidden = false AND (public_domain.partner_id IS NULL) ORDER BY public_domain."order"
2026-07-14 13:03:27.350 UTC [36580] myuser@simplelogin LOG:  duration: 0.089 ms
2026-07-14 13:03:27.353 UTC [36580] myuser@simplelogin LOG:  statement: SELECT custom_domain.id AS custom_domain_id, custom_domain.created_at AS custom_domain_created_at, custom_domain.updated_at AS custom_domain_updated_at, custom_domain.user_id AS custom_domain_user_id, custom_domain.domain AS custom_domain_domain, custom_domain.name AS custom_domain_name, custom_domain.verified AS custom_domain_verified, custom_domain.dkim_verified AS custom_domain_dkim_verified, custom_domain.spf_verified AS custom_domain_spf_verified, custom_domain.dmarc_verified AS custom_domain_dmarc_verified, custom_domain.catch_all AS custom_domain_catch_all, custom_domain.random_prefix_generation AS custom_domain_random_prefix_generation, custom_domain.nb_failed_checks AS custom_domain_nb_failed_checks, custom_domain.ownership_verified AS custom_domain_ownership_verified, custom_domain.ownership_txt_token AS custom_domain_ownership_txt_token, custom_domain.is_sl_subdomain AS custom_domain_is_sl_subdomain, custom_domain.partner_id AS custom_domain_partner_id, custom_domain.pending_deletion AS custom_domain_pending_deletion, mailbox_1.id AS mailbox_1_id, mailbox_1.created_at AS mailbox_1_created_at, mailbox_1.updated_at AS mailbox_1_updated_at, mailbox_1.user_id AS mailbox_1_user_id, mailbox_1.email AS mailbox_1_email, mailbox_1.verified AS mailbox_1_verified, mailbox_1.force_spf AS mailbox_1_force_spf, mailbox_1.new_email AS mailbox_1_new_email, mailbox_1.pgp_public_key AS mailbox_1_pgp_public_key, mailbox_1.pgp_finger_print AS mailbox_1_pgp_finger_print, mailbox_1.disable_pgp AS mailbox_1_disable_pgp, mailbox_1.nb_failed_checks AS mailbox_1_nb_failed_checks, mailbox_1.disabled AS mailbox_1_disabled, mailbox_1.generic_subject AS mailbox_1_generic_subject
	FROM custom_domain LEFT OUTER JOIN (domain_mailbox AS domain_mailbox_1 JOIN mailbox AS mailbox_1 ON mailbox_1.id = domain_mailbox_1.mailbox_id) ON custom_domain.id = domain_mailbox_1.domain_id
	WHERE custom_domain.user_id = 1 AND custom_domain.ownership_verified = true ORDER BY custom_domain.domain ASC
2026-07-14 13:03:27.353 UTC [36580] myuser@simplelogin LOG:  duration: 0.256 ms
2026-07-14 13:03:27.354 UTC [36580] myuser@simplelogin LOG:  statement: SELECT subscription.id AS subscription_id_1, subscription.created_at AS subscription_created_at, subscription.updated_at AS subscription_updated_at, subscription.cancel_url AS subscription_cancel_url, subscription.update_url AS subscription_update_url, subscription.subscription_id AS subscription_subscription_id, subscription.event_time AS subscription_event_time, subscription.next_bill_date AS subscription_next_bill_date, subscription.cancelled AS subscription_cancelled, subscription.plan AS subscription_plan, subscription.user_id AS subscription_user_id
	FROM subscription
	WHERE subscription.user_id = 1
	 LIMIT 1
2026-07-14 13:03:27.355 UTC [36580] myuser@simplelogin LOG:  duration: 0.082 ms
2026-07-14 13:03:27.356 UTC [36580] myuser@simplelogin LOG:  statement: SELECT public_domain.id AS public_domain_id, public_domain.created_at AS public_domain_created_at, public_domain.updated_at AS public_domain_updated_at, public_domain.domain AS public_domain_domain, public_domain.premium_only AS public_domain_premium_only, public_domain.can_use_subdomain AS public_domain_can_use_subdomain, public_domain.partner_id AS public_domain_partner_id, public_domain.hidden AS public_domain_hidden, public_domain."order" AS public_domain_order, public_domain.use_as_reverse_alias AS public_domain_use_as_reverse_alias
	FROM public_domain
	WHERE public_domain.hidden = false AND (public_domain.partner_id IS NULL) ORDER BY public_domain."order"
2026-07-14 13:03:27.356 UTC [36580] myuser@simplelogin LOG:  duration: 0.077 ms
2026-07-14 13:03:27.364 UTC [36580] myuser@simplelogin LOG:  statement: SELECT anon_1.alias_id AS anon_1_alias_id, anon_1.alias_created_at AS anon_1_alias_created_at, anon_1.alias_updated_at AS anon_1_alias_updated_at, anon_1.alias_user_id AS anon_1_alias_user_id, anon_1.alias_email AS anon_1_alias_email, anon_1.alias_name AS anon_1_alias_name, anon_1.alias_enabled AS anon_1_alias_enabled, anon_1.alias_flags AS anon_1_alias_flags, anon_1.alias_custom_domain_id AS anon_1_alias_custom_domain_id, anon_1.alias_automatic_creation AS anon_1_alias_automatic_creation, anon_1.alias_directory_id AS anon_1_alias_directory_id, anon_1.alias_note AS anon_1_alias_note, anon_1.alias_mailbox_id AS anon_1_alias_mailbox_id, anon_1.alias_disable_pgp AS anon_1_alias_disable_pgp, anon_1.alias_cannot_be_disabled AS anon_1_alias_cannot_be_disabled, anon_1.alias_disable_email_spoofing_check AS anon_1_alias_disable_email_spoofing_check, anon_1.alias_batch_import_id AS anon_1_alias_batch_import_id, anon_1.alias_original_owner_id AS anon_1_alias_original_owner_id, anon_1.alias_pinned AS anon_1_alias_pinned, anon_1.alias_transfer_token AS anon_1_alias_transfer_token, anon_1.alias_transfer_token_expiration AS anon_1_alias_transfer_token_expiration, anon_1.alias_hibp_last_check AS anon_1_alias_hibp_last_check, anon_1.alias_ts_vector AS anon_1_alias_ts_vector, anon_1.alias_last_email_log_id AS anon_1_alias_last_email_log_id, mailbox_1.id AS mailbox_1_id, mailbox_1.created_at AS mailbox_1_created_at, mailbox_1.updated_at AS mailbox_1_updated_at, mailbox_1.user_id AS mailbox_1_user_id, mailbox_1.email AS mailbox_1_email, mailbox_1.verified AS mailbox_1_verified, mailbox_1.force_spf AS mailbox_1_force_spf, mailbox_1.new_email AS mailbox_1_new_email, mailbox_1.pgp_public_key AS mailbox_1_pgp_public_key, mailbox_1.pgp_finger_print AS mailbox_1_pgp_finger_print, mailbox_1.disable_pgp AS mailbox_1_disable_pgp, mailbox_1.nb_failed_checks AS mailbox_1_nb_failed_checks, mailbox_1.disabled AS mailbox_1_disabled, mailbox_1.generic_subject AS mailbox_1_generic_subject, mailbox_2.id AS mailbox_2_id, mailbox_2.created_at AS mailbox_2_created_at, mailbox_2.updated_at AS mailbox_2_updated_at, mailbox_2.user_id AS mailbox_2_user_id, mailbox_2.email AS mailbox_2_email, mailbox_2.verified AS mailbox_2_verified, mailbox_2.force_spf AS mailbox_2_force_spf, mailbox_2.new_email AS mailbox_2_new_email, mailbox_2.pgp_public_key AS mailbox_2_pgp_public_key, mailbox_2.pgp_finger_print AS mailbox_2_pgp_finger_print, mailbox_2.disable_pgp AS mailbox_2_disable_pgp, mailbox_2.nb_failed_checks AS mailbox_2_nb_failed_checks, mailbox_2.disabled AS mailbox_2_disabled, mailbox_2.generic_subject AS mailbox_2_generic_subject
	FROM (SELECT alias.id AS alias_id, alias.created_at AS alias_created_at, alias.updated_at AS alias_updated_at, alias.user_id AS alias_user_id, alias.email AS alias_email, alias.name AS alias_name, alias.enabled AS alias_enabled, alias.flags AS alias_flags, alias.custom_domain_id AS alias_custom_domain_id, alias.automatic_creation AS alias_automatic_creation, alias.directory_id AS alias_directory_id, alias.note AS alias_note, alias.mailbox_id AS alias_mailbox_id, alias.disable_pgp AS alias_disable_pgp, alias.cannot_be_disabled AS alias_cannot_be_disabled, alias.disable_email_spoofing_check AS alias_disable_email_spoofing_check, alias.batch_import_id AS alias_batch_import_id, alias.original_owner_id AS alias_original_owner_id, alias.pinned AS alias_pinned, alias.transfer_token AS alias_transfer_token, alias.transfer_token_expiration AS alias_transfer_token_expiration, alias.hibp_last_check AS alias_hibp_last_check, alias.ts_vector AS alias_ts_vector, alias.last_email_log_id AS alias_last_email_log_id
	FROM alias
	WHERE alias.email = 'obscap169773@old.com'
	 LIMIT 1) AS anon_1 LEFT OUTER JOIN (alias_mailbox AS alias_mailbox_1 JOIN mailbox AS mailbox_1 ON mailbox_1.id = alias_mailbox_1.mailbox_id) ON anon_1.alias_id = alias_mailbox_1.alias_id LEFT OUTER JOIN mailbox AS mailbox_2 ON mailbox_2.id = anon_1.alias_mailbox_id
2026-07-14 13:03:27.364 UTC [36580] myuser@simplelogin LOG:  duration: 0.561 ms
2026-07-14 13:03:27.367 UTC [36580] myuser@simplelogin LOG:  statement: SELECT domain_deleted_alias.id AS domain_deleted_alias_id, domain_deleted_alias.created_at AS domain_deleted_alias_created_at, domain_deleted_alias.updated_at AS domain_deleted_alias_updated_at, domain_deleted_alias.email AS domain_deleted_alias_email, domain_deleted_alias.domain_id AS domain_deleted_alias_domain_id, domain_deleted_alias.user_id AS domain_deleted_alias_user_id, domain_deleted_alias.reason AS domain_deleted_alias_reason
	FROM domain_deleted_alias
	WHERE domain_deleted_alias.email = 'obscap169773@old.com'
	 LIMIT 1
2026-07-14 13:03:27.367 UTC [36580] myuser@simplelogin LOG:  duration: 0.122 ms
2026-07-14 13:03:27.368 UTC [36580] myuser@simplelogin LOG:  statement: SELECT deleted_alias.id AS deleted_alias_id, deleted_alias.created_at AS deleted_alias_created_at, deleted_alias.updated_at AS deleted_alias_updated_at, deleted_alias.email AS deleted_alias_email, deleted_alias.reason AS deleted_alias_reason
	FROM deleted_alias
	WHERE deleted_alias.email = 'obscap169773@old.com'
	 LIMIT 1
2026-07-14 13:03:27.368 UTC [36580] myuser@simplelogin LOG:  duration: 0.096 ms
2026-07-14 13:03:27.369 UTC [36580] myuser@simplelogin LOG:  statement: SELECT subscription.id AS subscription_id_1, subscription.created_at AS subscription_created_at, subscription.updated_at AS subscription_updated_at, subscription.cancel_url AS subscription_cancel_url, subscription.update_url AS subscription_update_url, subscription.subscription_id AS subscription_subscription_id, subscription.event_time AS subscription_event_time, subscription.next_bill_date AS subscription_next_bill_date, subscription.cancelled AS subscription_cancelled, subscription.plan AS subscription_plan, subscription.user_id AS subscription_user_id
	FROM subscription
	WHERE subscription.user_id = 1
	 LIMIT 1
2026-07-14 13:03:27.369 UTC [36580] myuser@simplelogin LOG:  duration: 0.109 ms
2026-07-14 13:03:27.370 UTC [36580] myuser@simplelogin LOG:  statement: SELECT deleted_alias.id AS deleted_alias_id, deleted_alias.created_at AS deleted_alias_created_at, deleted_alias.updated_at AS deleted_alias_updated_at, deleted_alias.email AS deleted_alias_email, deleted_alias.reason AS deleted_alias_reason
	FROM deleted_alias
	WHERE deleted_alias.email = 'obscap169773@old.com'
	 LIMIT 1
2026-07-14 13:03:27.370 UTC [36580] myuser@simplelogin LOG:  duration: 0.090 ms
2026-07-14 13:03:27.371 UTC [36580] myuser@simplelogin LOG:  statement: SELECT domain_deleted_alias.id AS domain_deleted_alias_id, domain_deleted_alias.created_at AS domain_deleted_alias_created_at, domain_deleted_alias.updated_at AS domain_deleted_alias_updated_at, domain_deleted_alias.email AS domain_deleted_alias_email, domain_deleted_alias.domain_id AS domain_deleted_alias_domain_id, domain_deleted_alias.user_id AS domain_deleted_alias_user_id, domain_deleted_alias.reason AS domain_deleted_alias_reason
	FROM domain_deleted_alias
	WHERE domain_deleted_alias.email = 'obscap169773@old.com'
	 LIMIT 1
2026-07-14 13:03:27.371 UTC [36580] myuser@simplelogin LOG:  duration: 0.093 ms
2026-07-14 13:03:27.372 UTC [36580] myuser@simplelogin LOG:  statement: SELECT public_domain.id AS public_domain_id, public_domain.created_at AS public_domain_created_at, public_domain.updated_at AS public_domain_updated_at, public_domain.domain AS public_domain_domain, public_domain.premium_only AS public_domain_premium_only, public_domain.can_use_subdomain AS public_domain_can_use_subdomain, public_domain.partner_id AS public_domain_partner_id, public_domain.hidden AS public_domain_hidden, public_domain."order" AS public_domain_order, public_domain.use_as_reverse_alias AS public_domain_use_as_reverse_alias
	FROM public_domain
	WHERE public_domain.domain = 'old.com'
	 LIMIT 1
2026-07-14 13:03:27.372 UTC [36580] myuser@simplelogin LOG:  duration: 0.100 ms
2026-07-14 13:03:27.377 UTC [36580] myuser@simplelogin LOG:  statement: SELECT anon_1.custom_domain_id AS anon_1_custom_domain_id, anon_1.custom_domain_created_at AS anon_1_custom_domain_created_at, anon_1.custom_domain_updated_at AS anon_1_custom_domain_updated_at, anon_1.custom_domain_user_id AS anon_1_custom_domain_user_id, anon_1.custom_domain_domain AS anon_1_custom_domain_domain, anon_1.custom_domain_name AS anon_1_custom_domain_name, anon_1.custom_domain_verified AS anon_1_custom_domain_verified, anon_1.custom_domain_dkim_verified AS anon_1_custom_domain_dkim_verified, anon_1.custom_domain_spf_verified AS anon_1_custom_domain_spf_verified, anon_1.custom_domain_dmarc_verified AS anon_1_custom_domain_dmarc_verified, anon_1.custom_domain_catch_all AS anon_1_custom_domain_catch_all, anon_1.custom_domain_random_prefix_generation AS anon_1_custom_domain_random_prefix_generation, anon_1.custom_domain_nb_failed_checks AS anon_1_custom_domain_nb_failed_checks, anon_1.custom_domain_ownership_verified AS anon_1_custom_domain_ownership_verified, anon_1.custom_domain_ownership_txt_token AS anon_1_custom_domain_ownership_txt_token, anon_1.custom_domain_is_sl_subdomain AS anon_1_custom_domain_is_sl_subdomain, anon_1.custom_domain_partner_id AS anon_1_custom_domain_partner_id, anon_1.custom_domain_pending_deletion AS anon_1_custom_domain_pending_deletion, mailbox_1.id AS mailbox_1_id, mailbox_1.created_at AS mailbox_1_created_at, mailbox_1.updated_at AS mailbox_1_updated_at, mailbox_1.user_id AS mailbox_1_user_id, mailbox_1.email AS mailbox_1_email, mailbox_1.verified AS mailbox_1_verified, mailbox_1.force_spf AS mailbox_1_force_spf, mailbox_1.new_email AS mailbox_1_new_email, mailbox_1.pgp_public_key AS mailbox_1_pgp_public_key, mailbox_1.pgp_finger_print AS mailbox_1_pgp_finger_print, mailbox_1.disable_pgp AS mailbox_1_disable_pgp, mailbox_1.nb_failed_checks AS mailbox_1_nb_failed_checks, mailbox_1.disabled AS mailbox_1_disabled, mailbox_1.generic_subject AS mailbox_1_generic_subject
	FROM (SELECT custom_domain.id AS custom_domain_id, custom_domain.created_at AS custom_domain_created_at, custom_domain.updated_at AS custom_domain_updated_at, custom_domain.user_id AS custom_domain_user_id, custom_domain.domain AS custom_domain_domain, custom_domain.name AS custom_domain_name, custom_domain.verified AS custom_domain_verified, custom_domain.dkim_verified AS custom_domain_dkim_verified, custom_domain.spf_verified AS custom_domain_spf_verified, custom_domain.dmarc_verified AS custom_domain_dmarc_verified, custom_domain.catch_all AS custom_domain_catch_all, custom_domain.random_prefix_generation AS custom_domain_random_prefix_generation, custom_domain.nb_failed_checks AS custom_domain_nb_failed_checks, custom_domain.ownership_verified AS custom_domain_ownership_verified, custom_domain.ownership_txt_token AS custom_domain_ownership_txt_token, custom_domain.is_sl_subdomain AS custom_domain_is_sl_subdomain, custom_domain.partner_id AS custom_domain_partner_id, custom_domain.pending_deletion AS custom_domain_pending_deletion
	FROM custom_domain
	WHERE custom_domain.domain = 'old.com'
	 LIMIT 1) AS anon_1 LEFT OUTER JOIN (domain_mailbox AS domain_mailbox_1 JOIN mailbox AS mailbox_1 ON mailbox_1.id = domain_mailbox_1.mailbox_id) ON anon_1.custom_domain_id = domain_mailbox_1.domain_id
2026-07-14 13:03:27.378 UTC [36580] myuser@simplelogin LOG:  duration: 0.369 ms
2026-07-14 13:03:27.380 UTC [36580] myuser@simplelogin LOG:  statement: INSERT INTO alias (created_at, updated_at, user_id, email, name, enabled, flags, custom_domain_id, automatic_creation, directory_id, note, mailbox_id, disable_pgp, cannot_be_disabled, disable_email_spoofing_check, batch_import_id, original_owner_id, pinned, transfer_token, transfer_token_expiration, hibp_last_check, last_email_log_id) VALUES ('2026-07-14T13:03:27.380647'::timestamp, NULL, 1, 'obscap169773@old.com', NULL, true, 0, 2, false, NULL, '', 1, false, false, false, NULL, NULL, false, NULL, '2026-07-14T13:03:27.380677'::timestamp, NULL, NULL) RETURNING alias.id
2026-07-14 13:03:27.381 UTC [36580] myuser@simplelogin LOG:  duration: 0.653 ms
2026-07-14 13:03:27.382 UTC [36580] myuser@simplelogin LOG:  statement: SELECT daily_metric.id AS daily_metric_id, daily_metric.created_at AS daily_metric_created_at, daily_metric.updated_at AS daily_metric_updated_at, daily_metric.date AS daily_metric_date, daily_metric.nb_new_web_non_proton_user AS daily_metric_nb_new_web_non_proton_user, daily_metric.nb_alias AS daily_metric_nb_alias
	FROM daily_metric
	WHERE daily_metric.date = '2026-07-14'::date
	 LIMIT 1
2026-07-14 13:03:27.382 UTC [36580] myuser@simplelogin LOG:  duration: 0.127 ms
2026-07-14 13:03:27.383 UTC [36580] myuser@simplelogin LOG:  statement: UPDATE daily_metric SET updated_at='2026-07-14T13:03:27.383359'::timestamp, nb_alias=27 WHERE daily_metric.id = 1
2026-07-14 13:03:27.383 UTC [36580] myuser@simplelogin LOG:  duration: 0.108 ms
2026-07-14 13:03:27.384 UTC [36580] myuser@simplelogin LOG:  statement: INSERT INTO alias_audit_log (created_at, updated_at, user_id, alias_id, alias_email, action, message) VALUES ('2026-07-14T13:03:27.383879'::timestamp, NULL, 1, 27, 'obscap169773@old.com', 'create', 'New alias created') RETURNING alias_audit_log.id
2026-07-14 13:03:27.384 UTC [36580] myuser@simplelogin LOG:  duration: 0.128 ms
2026-07-14 13:03:27.385 UTC [36580] myuser@simplelogin LOG:  statement: INSERT INTO alias_mailbox (created_at, updated_at, alias_id, mailbox_id) VALUES ('2026-07-14T13:03:27.385295'::timestamp, NULL, 27, 2) RETURNING alias_mailbox.id
2026-07-14 13:03:27.385 UTC [36580] myuser@simplelogin LOG:  duration: 0.417 ms
2026-07-14 13:03:27.386 UTC [36580] myuser@simplelogin LOG:  statement: COMMIT
2026-07-14 13:03:27.387 UTC [36580] myuser@simplelogin LOG:  duration: 1.003 ms
2026-07-14 13:03:27.388 UTC [36580] myuser@simplelogin LOG:  statement: BEGIN
2026-07-14 13:03:27.388 UTC [36580] myuser@simplelogin LOG:  duration: 0.024 ms
2026-07-14 13:03:27.388 UTC [36580] myuser@simplelogin LOG:  statement: SELECT alias.id AS alias_id, alias.created_at AS alias_created_at, alias.updated_at AS alias_updated_at, alias.user_id AS alias_user_id, alias.email AS alias_email, alias.name AS alias_name, alias.enabled AS alias_enabled, alias.flags AS alias_flags, alias.custom_domain_id AS alias_custom_domain_id, alias.automatic_creation AS alias_automatic_creation, alias.directory_id AS alias_directory_id, alias.note AS alias_note, alias.mailbox_id AS alias_mailbox_id, alias.disable_pgp AS alias_disable_pgp, alias.cannot_be_disabled AS alias_cannot_be_disabled, alias.disable_email_spoofing_check AS alias_disable_email_spoofing_check, alias.batch_import_id AS alias_batch_import_id, alias.original_owner_id AS alias_original_owner_id, alias.pinned AS alias_pinned, alias.transfer_token AS alias_transfer_token, alias.transfer_token_expiration AS alias_transfer_token_expiration, alias.hibp_last_check AS alias_hibp_last_check, alias.ts_vector AS alias_ts_vector, alias.last_email_log_id AS alias_last_email_log_id
	FROM alias
	WHERE alias.id = 27
2026-07-14 13:03:27.388 UTC [36580] myuser@simplelogin LOG:  duration: 0.165 ms
2026-07-14 13:03:27.390 UTC [36580] myuser@simplelogin LOG:  statement: ROLLBACK
2026-07-14 13:03:27.390 UTC [36580] myuser@simplelogin LOG:  duration: 0.032 ms
```

Note the two OBSERVED differences from a random alias in the `INSERT INTO alias` values: `custom_domain_id=2` (because `@old.com` is one of `john`'s custom domains — random aliases have `custom_domain_id=NULL`) and `note=''` (empty string from the submitted `note=` field — a random alias has `note=NULL`).

**OBSERVED ordering nuance (not a stability concern).** The write **set** (which tables, which rows) is identical and stable across runs. The intra-transaction *ordering* of the `UPDATE daily_metric` vs the `INSERT alias_audit_log` varies run to run: across the four captured transactions, random run 1 (spliced above) emitted `INSERT alias_audit_log` then `UPDATE daily_metric`, whereas random run 2 emitted them in the opposite order (`UPDATE daily_metric` first); custom run 1 (spliced above) emitted `UPDATE daily_metric` first, and custom run 2 emitted `INSERT alias_audit_log` first. **INFERRED** cause: SQLAlchemy 1.3.24 unit-of-work flush ordering. This is an implementation detail; the canonical, stable fact is the set of tables/rows written in one transaction.

### 5.4 Key facts (cited)

- **`daily_metric` is a GLOBAL per-day counter, NOT per-user** (`app/models.py:3262-3287`; `get_or_create_today_metric` at `:3280`). Its schema has `date` UNIQUE and columns `id, created_at, updated_at, date, nb_new_web_non_proton_user, nb_alias` — there is **no `user_id`**:
  ```python
  class DailyMetric(Base, ModelMixin):
      __tablename__ = "daily_metric"
      date = sa.Column(sa.Date, nullable=False, unique=True)
      # users who sign up via web without using "Login with Proton"
      nb_new_web_non_proton_user = sa.Column(
          sa.Integer, nullable=False, server_default="0", default=0
      )
      nb_alias = sa.Column(sa.Integer, nullable=False, server_default="0", default=0)

      @staticmethod
      def get_or_create_today_metric() -> DailyMetric:
          today = arrow.utcnow().date()
          daily_metric = DailyMetric.get_by(date=today)
          if not daily_metric:
              daily_metric = DailyMetric.create(
                  date=today, nb_new_web_non_proton_user=0, nb_alias=0
              )
          return daily_metric
  ```
  The **first** alias of a calendar day INSERTs the row; **every later one** UPDATEs `nb_alias += 1`. In the runs above the row already existed (id=1), so both creations were UPDATEs.
- **`alias_audit_log` is written SYNCHRONOUSLY in the same transaction** (`app/models.py:3810` defines the model), via `emit_alias_audit_log` (`app/alias_audit_log_utils.py:18-32`) with `action='create'`, `message='New alias created'`:
  ```python
  def emit_alias_audit_log(alias, action, message, user_id=None, commit=False):
      AliasAuditLog.create(
          user_id=user_id or alias.user_id,
          alias_id=alias.id,
          alias_email=alias.email,
          action=action.value,   # AliasAuditLogAction.CreateAlias = "create"
          message=message,
          commit=commit,
      )
  ```
- **`sync_event` is never written and no `NOTIFY` fires** (OBSERVED). The `sync_event` row count was `count=0, max_id=0` both before and after every create above (delta **+0**), and a dedicated `LISTEN simplelogin_sync_events` connection held open across the creates received **0** notifications (see §6.3 for the direct capture). This independently corroborates the Q5 finding (§6) that the domain event is a no-op in the default configuration.

### 5.5 Table-count summary

- **Default random alias = 3 tables:** `alias` (INSERT), `daily_metric` (UPDATE, or INSERT on the day's first alias), `alias_audit_log` (INSERT).
- **Custom alias with a secondary mailbox = 4 tables:** the above three **plus** `alias_mailbox` (INSERT, one row per secondary mailbox). A single-mailbox custom alias writes the same 3 tables as a random alias.

---

## Section 6 — Q5: Logs & Background Work (OBSERVED, default config)

**How captured (producing commands).** The dev server writes its application log to `/tmp/blitzy_server.log` (the file the run redirects stdout/stderr to; see §1.6). A temporary harness (`obs_logs_events.py`, removed afterwards) marked the log line count, issued one isolated `POST`, marked the line count again, and sliced the log. In parallel it held open a dedicated `LISTEN simplelogin_sync_events` psycopg2 connection and snapshotted the `sync_event` row count before/after, to directly observe whether any domain event was dispatched. The single-process server ran as PID 4333 throughout.

### 6.1 Complete application log for ONE random `POST /dashboard/`

**Exactly 4 lines** (OBSERVED, verbatim from `/tmp/blitzy_server.log`):

```
2026-07-13 19:13:05,176 - SL - DEBUG - 4333 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email artery_lapses149@sl.local
2026-07-13 19:13:05,183 - SL - INFO - 4333 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 19:13:05,186 - SL - DEBUG - 4333 - "/app/app/dashboard/views/index.py:110" - index() -  - create new random alias <Alias 29 artery_lapses149@sl.local> for user <User 1 John Wick john@wick.com>
2026-07-13 19:13:05,191 - SL - DEBUG - 4333 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /dashboard/ ImmutableMultiDict([]) 302, takes 0.036722660064697266
```

The four lines are: (1) the random-email generator (`app/models.py:1459`, `generate_random_alias_email()`); (2) the gated event no-op (`app/events/event_dispatcher.py:62` — see §6.3); (3) the create-confirmation `LOG.d` (`app/dashboard/views/index.py:110`, showing `<Alias 29 …>` and `<User 1 John Wick …>`); (4) the request-timing line from `after_request` (`server.py:284`), here `takes 0.036722660064697266` (≈37 ms total request wall-clock).

### 6.2 Complete application log for ONE custom `POST /dashboard/custom_alias`

**Exactly 2 lines** — the custom view emits **no** dedicated create-confirmation `LOG` line (only the random view logs `"create new random alias …"` at `app/dashboard/views/index.py:110`), and there is no random-email generation:

```
2026-07-13 19:13:05,795 - SL - INFO - 4333 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 19:13:05,801 - SL - DEBUG - 4333 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /dashboard/custom_alias ImmutableMultiDict([]) 302, takes 0.0802762508392334
```

The custom request here logged `takes 0.0802762508392334` (≈80 ms). **OBSERVED timing note (resolves any "~25 ms" claim):** the `after_request` "takes" value is the *total per-request wall-clock* logged at `server.py:284`; observed values across runs were `0.0367` (random) and `0.0803 / 0.0710 / 0.0658` (custom). Custom is consistently higher because it runs more `SELECT`s (suffix, mailbox, and duplicate checks). These are single measurements and vary naturally run to run.

### 6.3 Findings

**The domain event is a GATED NO-OP — and it is the FIRST gate (webhook), not the partner gate, that fires.** `EventDispatcher.send_event` (`app/events/event_dispatcher.py:47-84`) has **three** early-return gates before it would ever dispatch:

```python
        if config.EVENT_WEBHOOK_DISABLE:                                   # Gate 1 (L57-59)
            LOG.i("Not sending events because webhook is disabled")
            return

        if not config.EVENT_WEBHOOK and skip_if_webhook_missing:           # Gate 2 (L61-65)
            LOG.i(
                "Not sending events because webhook is not configured and allowed to be empty"
            )
            return

        partner_user = EventDispatcher.__partner_user(user.id)             # Gate 3 (L67-70)
        if not partner_user:
            LOG.i(f"Not sending events because there's no partner user for user {user}")
            return

        event = event_pb2.Event(                                           # dispatch: reached only if all 3 gates pass
            user_id=user.id,
            external_user_id=partner_user.external_user_id,
            partner_id=partner_user.partner_id,
            content=content,
        )

        serialized = event.SerializeToString()
        dispatcher.send(serialized)
```

The call from `Alias.create` (`app/models.py:1687`) is `EventDispatcher.send_event(user, EventContent(alias_created=event))` — it uses the default `skip_if_webhook_missing=True`. Because `EVENT_WEBHOOK` is unset in the default configuration (`app/config.py:612` — `EVENT_WEBHOOK = os.environ.get("EVENT_WEBHOOK", None)`), **Gate 2 fires first and returns**, so the partner check (Gate 3) is never reached. The OBSERVED log line proves exactly which gate ran — it is the Gate-2 message from `app/events/event_dispatcher.py:62`:

```
2026-07-13 19:13:05,183 - SL - INFO - 4333 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
```

So a fully-configured dispatch requires **both** a non-empty `EVENT_WEBHOOK` **AND** a `PartnerUser` for the user (it is an **AND**, not an OR). The `AliasCreated` protobuf is still built in `Alias.create` (`app/models.py:1680-1686`) and then **discarded**.

**No `sync_event` row and no `NOTIFY` (DIRECT OBSERVATION).** Only if all three gates pass does `PostgresDispatcher.send` (`app/events/event_dispatcher.py:24-26`) run `SyncEvent.create(content=event, flush=True)` + `Session.execute("NOTIFY simplelogin_sync_events, '<id>'")`. Captured evidence that this did **not** happen:

```
sync_event BEFORE: count=0, max_id=0
NOTIFY poll: select() timed out -> no notifications pending
NOTIFY notifications received on simplelogin_sync_events: 0
sync_event AFTER : count=0, max_id=0
sync_event DELTA : +0
```

**No background worker is running (OBSERVED via `ps -eo pid,ppid,args`).** The **only** SimpleLogin process is the dev server (`4333 /app/venv/bin/python server.py`, `ppid=0`). Everything else is infrastructure: PostgreSQL 15 (`3393`) and its internal workers (checkpointer, background writer, walwriter, autovacuum launcher, logical-replication launcher), `redis-server` (`3369`), the `socat` host-bridge (`3615/3621`), `gpg-agent` daemons, and one idle pooled PG backend (`4339`). A targeted search confirms none of the SimpleLogin async workers are up:

```
$ ps -eo args | grep -iE 'event_listener|cron|job_runner|gunicorn|celery' | grep -v grep
(none found)
```

Isolation greps over the full log corroborate the per-request findings (producing commands shown):

```
$ grep -c 'Not sending events because webhook is not configured' /tmp/blitzy_server.log
7                                    # one gated no-op per successful create
$ grep -c 'create new random alias' /tmp/blitzy_server.log
4                                    # only the RANDOM path logs (app/dashboard/views/index.py:110)
```

- **The `alias_audit_log` write is SYNCHRONOUS** within the request transaction (see §5) — it is **not** a background job.
- **Q5 answer (stated plainly):** in the default configuration, alias creation triggers **NO background tasks and NO follow-up events**. The complete extra work beyond the `alias` INSERT is: one `daily_metric` UPDATE, one `alias_audit_log` INSERT, and one logged-then-discarded event — all inside the same synchronous request transaction.
- **NON-CANONICAL contrast (INFERRED):** with `EVENT_WEBHOOK` set **and** a `PartnerUser` present, `send_event` would pass all three gates, serialize the protobuf (`event.SerializeToString()`), and `PostgresDispatcher.send` would `INSERT sync_event` + `NOTIFY simplelogin_sync_events` for the external `event_listener.py` consumer. This path was **not run** (it is non-default) and is therefore inferred from the code, not observed.

---

## Section 7 — Q6: Error / Edge-Path Catalog (OBSERVED live unless labelled; default config)

**How these were captured.** Every branch below was exercised against the canonical running server (PID varied across captures — `5030`/`5356`/`5433` for most branches, with the B3 route-429 evidence re-captured in a separate session under app PID `36792` — on the mandated image, host-reachable via the in-container `socat` bridge on `127.0.0.1:7777`). For each condition the producing command is a scripted `requests` client; the response status/`Location`/flash was read by following the 302 to its target (or from the 200 body); `alias`-row deltas were measured with `psycopg2` `SELECT count(*)` before/after; log evidence was sliced from `/tmp/blitzy_server.log`. Each failing branch performs **NO database write unless explicitly stated** — the success write-set (§5) never occurs on a rejected request (every "delta +0" below is a direct measurement).

**Canonical vs. adversarial (labelling policy).** The two browser forms validate some inputs client-side before submitting: `templates/dashboard/custom_alias.html` runs a click handler (`:112`) that requires a prefix and at least one mailbox and blocks the submit otherwise, and the random form only ever submits `form-name`/`csrf_token`/`generator_scheme`. Conditions that a real browser cannot produce (an **invalid-charset or over-length prefix** — the prefix input's Parsley `data-parsley-pattern="[0-9a-z-_.]{1,}"` and `maxlength=40` block these client-side, OBSERVED in §7 B5 — a tampered/expired/missing signed suffix, a non-existent `mailboxes` id, a non-numeric `generator_scheme`, a suffix for an unavailable domain) are therefore reached only by POSTing crafted fields **directly**, bypassing that client-side validation; those inputs are labelled **NON-CANONICAL** and the browser's own behaviour is stated first. In every such case the **route response itself is a real, canonical observation** — only the crafted *input* is non-canonical.

### B1 — Quota exceeded (OBSERVED)

**Setup (runtime, not repo):** a free, quota-maxed user `freeobs@example.com` (id=3, `trial_end=None`, `lifetime=False`) topped up to exactly 5 aliases. The gate is `User.can_create_new_alias` (`app/models.py:867-884`), which for a non-subscription user returns `Alias.filter_by(user_id=self.id).count() < self.max_alias_for_free_account()` (`MAX_NB_EMAIL_FREE_PLAN=5`, runtime-confirmed):

```python
        if self.lifetime_or_active_subscription():
            return True
        else:
            return (
                Alias.filter_by(user_id=self.id).count()
                < self.max_alias_for_free_account()
            )
```

OBSERVED at `freeobs` count = 5 (`can_create_new_alias()` returned `False`, runtime-confirmed), alias count **5 → 5 (+0)** in every case:

| Entry point | HTTP | `Location` | Flash (category, message) |
|---|---|---|---|
| **RANDOM** `POST /dashboard/` | **302** | `…/dashboard/?query=&sort=&filter=&page=0` (CL 309; **no `highlight_alias_id` ⇒ no write**) | `warning` — `You need to upgrade your plan to create new alias.` |
| **CUSTOM** `POST /dashboard/custom_alias` | **302** | `…/dashboard/` (CL 229) | `warning` — `You have reached free plan limit, please upgrade to create new aliases` |
| **CUSTOM** `GET /dashboard/custom_alias` | **302** | `…/dashboard/` (CL 229) | `warning` — `You have reached free plan limit, please upgrade to create new aliases` |

- **RANDOM** falls through the `else` at `app/dashboard/views/index.py:122-123` (`flash("You need to upgrade your plan to create new alias.", "warning")`, **no return**) to the common redirect.
- **CUSTOM** gates at **function entry** — *before any page is rendered* — at `app/dashboard/views/custom_alias.py:35-43`:
  ```python
  def custom_alias():
      # check if user has not exceeded the alias quota
      if not current_user.can_create_new_alias():
          LOG.d("%s can't create new alias", current_user)
          flash(
              "You have reached free plan limit, please upgrade to create new aliases",
              "warning",
          )
          return redirect(url_for("dashboard.index"))
  ```
  This applies to **both** `GET` and `POST`: a maxed-out free user requesting the custom-alias page is **302-redirected to `/dashboard/`** and never sees the form — OBSERVED, the maxed-free custom page does **not render** (so it never reaches the suffix/mailbox selectors); it redirects at function entry.

### B2 — Invalid / missing CSRF (OBSERVED)

Both routes validate a `CSRFValidationForm` (`app/utils.py:157-158`) before any write and, on failure, flash `"Invalid request"` (category `warning`) and `redirect(request.url)` — the random route at `app/dashboard/views/index.py:88-90`, the custom route at `app/dashboard/views/custom_alias.py:56-58`:

```python
    if request.method == "POST":
        if not csrf_form.validate():
            flash("Invalid request", "warning")
            return redirect(request.url)
```

OBSERVED cross-product (user `john`; **wrong** = a syntactically-valid-but-incorrect token, **missing** = no `csrf_token` field). Alias count **29 → 29 (+0)** in all four cases; because the handler uses `redirect(request.url)`, each route redirects **to itself**:

| Route | csrf_token | HTTP | `Location` | Flash |
|---|---|---|---|---|
| `POST /dashboard/` (random) | wrong | **302** | `…/dashboard/` | `warning` — `Invalid request` |
| `POST /dashboard/` (random) | missing | **302** | `…/dashboard/` | `warning` — `Invalid request` |
| `POST /dashboard/custom_alias` | wrong | **302** | `…/dashboard/custom_alias` | `warning` — `Invalid request` |
| `POST /dashboard/custom_alias` | missing | **302** | `…/dashboard/custom_alias` | `warning` — `Invalid request` |

### B3 — Route rate-limit 429 (OBSERVED)

Flask-Limiter is **ACTIVE by default** (`DISABLE_RATE_LIMIT=False`, `app/config.py:602`) with `ALIAS_LIMIT="100/day;50/hour;5/minute"` (`app/config.py:448`) and **in-memory** storage (`MEM_STORE_URI` unset — so restarting the server clears the counters, which the capture relies on). The limiter key is `userid:{current_user.id}` when authenticated (`app/extensions.py` `__key_func`). The decorators differ per route:

- **Random** `app/dashboard/views/index.py:57-61`: `@limiter.limit(ALIAS_LIMIT, methods=["POST"], exempt_when=lambda: request.form.get("form-name") != "create-random-email")` — so the 5/minute cap applies **only** to a `POST` whose `form-name` is `create-random-email` (GET and the delete/disable POSTs are exempt). A second decorator at `:62` caps `GET` at `10/minute` keyed by `current_user.id`.
- **Custom** `app/dashboard/views/custom_alias.py:31`: `@limiter.limit(ALIAS_LIMIT, methods=["POST"])` — the 5/minute cap applies to **all** POSTs (GET is not capped).

OBSERVED, fresh in-memory counter, firing rapid POSTs as `john`:

| Route | POST #1–5 | POST #6 | 429 `Content-Type` / `Content-Length` |
|---|---|---|---|
| `POST /dashboard/` (random) | **302** created ids 29,30,31,32,33 (CL 339 each) | **429** | `text/html; charset=utf-8` / **6067** bytes |
| `POST /dashboard/custom_alias` | **302** created ids 34,35,36,37,38 (CL 273 each) | **429** | `text/html; charset=utf-8` / **6039** bytes |

> **Byte-count nuance (OBSERVED).** The random 429 body is a **stable 6067 bytes** across runs because the word-generator localpart is a fixed shape (`word_word###@sl.local`, 5 queued flashes). The **custom** 429 body size **varies with the user-chosen prefix lengths** (a non-canonical, input-dependent value): the **6039 bytes** here were produced with prefixes `c429test1`–`c429test6`. So only the random count is canonical and reproducible; the custom count depends on the crafted prefixes.

`LOG.w` at `server.py:364` (both routes), verbatim:

```
2026-07-14 13:06:30,751 - SL - WARNING - 36792 - "/app/server.py:364" - rate_limited() -  - Client hit rate limit on path /dashboard/, user:<User 1 John Wick john@wick.com>
2026-07-14 13:07:32,044 - SL - WARNING - 36792 - "/app/server.py:364" - rate_limited() -  - Client hit rate limit on path /dashboard/custom_alias, user:<User 1 John Wick john@wick.com>
```

The 429 handler (`server.py:362-372`) branches HTML vs. JSON by path prefix — the web routes render `templates/error/429.html` (which `{% extends "error.html" %}`); the API path (§8) returns JSON:

```python
    @app.errorhandler(429)
    def rate_limited(e):
        LOG.w(
            "Client hit rate limit on path %s, user:%s",
            request.path,
            get_current_user(),
        )
        if request.path.startswith("/api/"):
            return jsonify(error="Rate limit exceeded"), 429
        else:
            return render_template("error/429.html"), 429
```

The **complete 6067-byte** random-route 429 response body — verbatim and unedited (only end-of-line whitespace stripped from 13 wrapped lines; no markup changed) — is reproduced **in full** below. It is a complete dashboard-chrome HTML page (the same `<head>` rendered in §2); its `<body>` carries the five queued `toastr.success` flashes from POST #1–5 (Flask carries pending flashes to the next rendered page) on one line, then the `error/429.html` block (`{% extends "error.html" %}`: the large **429**, *"Whoa, slow down there, pardner!"*, and the **Home Page** button → `/`), then the standard footer scripts:

```html

<!DOCTYPE html>
<html lang="en"
      dir="ltr"
      data-theme="">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport"
          content="width=device-width, user-scalable=no, initial-scale=1.0, maximum-scale=1.0, minimum-scale=1.0" />
    <meta http-equiv="X-UA-Compatible" content="ie=edge" />
    <meta http-equiv="Content-Language" content="en" />
    <meta name="msapplication-TileColor" content="#2d89ef" />
    <meta name="theme-color" content="#4188c9" />
    <meta name="apple-mobile-web-app-status-bar-style"
          content="black-translucent" />
    <meta name="apple-mobile-web-app-capable" content="yes" />
    <meta name="mobile-web-app-capable" content="yes" />
    <meta name="HandheldFriendly" content="True" />
    <meta name="MobileOptimized" content="320" />
    <meta name="referrer" content="no-referrer" />
    <!-- Bing -->
    <meta name="msvalidate.01" content="2A313A69CBFD1A378C3B91734DC221A8" />
    <!-- Yandex -->
    <meta name="yandex-verification" content="c9e5d4d68bc983a1" />
    <meta name="description"
          content="Protect your email address with email ALIAS. Create a different email alias for each website. No more phishing, or spam." />
    <link rel="icon" href="/static/favicon.ico" type="image/x-icon" />
    <link rel="shortcut icon" type="image/x-icon" href="/static/favicon.ico" />
    <link rel="canonical" href="http://localhost:7777/dashboard/" />
    <title>

      | SimpleLogin
    </title>
    <link rel="stylesheet"
          href="/static/node_modules/font-awesome/css/font-awesome.css" />
    <!-- Dashboard Core -->
    <link href="/static/assets/css/dashboard.css" rel="stylesheet" />
    <!-- Tabler JS -->
    <script src="/static/assets/js/vendors/jquery-3.2.1.min.js"></script>
    <script src="/static/assets/js/vendors/bootstrap.bundle.min.js"></script>
    <script src="/static/assets/js/vendors/jquery.sparkline.min.js"></script>
    <script src="/static/assets/js/vendors/selectize.min.js"></script>
    <script src="/static/assets/js/vendors/jquery.tablesorter.min.js"></script>
    <script src="/static/assets/js/vendors/jquery-jvectormap-2.0.3.min.js"></script>
    <script src="/static/assets/js/vendors/jquery-jvectormap-de-merc.js"></script>
    <script src="/static/assets/js/vendors/jquery-jvectormap-world-mill.js"></script>
    <script src="/static/assets/js/vendors/circle-progress.min.js"></script>
    <script src="/static/assets/js/core.js"></script>
    <!-- ClipboardJS -->
    <script src="/static/vendor/clipboard.min.js"></script>
    <!-- IntroJS -->
    <link rel="stylesheet"
          type="text/css"
          href="/static/node_modules/intro.js/minified/introjs.min.css" />
    <script src="/static/node_modules/intro.js/minified/intro.min.js"></script>
    <!-- Sentry -->
    <script src="/static/node_modules/%40sentry/browser/build/bundle.min.js"></script>
    <link rel="stylesheet" href="/static/vendor/bootstrap-social.min.css" />
    <!-- Toastr library -->
    <link rel="stylesheet"
          href="/static/node_modules/toastr/build/toastr.min.css" />
    <script src="/static/node_modules/toastr/build/toastr.min.js"></script>
    <script src="/static/node_modules/bootbox/dist/bootbox.min.js"></script>
    <!-- Multiple-select library -->
    <link rel="stylesheet"
          href="/static/node_modules/multiple-select/dist/multiple-select.min.css" />
    <script src="/static/node_modules/multiple-select/dist/multiple-select.min.js"></script>
    <!-- Parseley library -->
    <script src="/static/node_modules/parsleyjs/dist/parsley.min.js"></script>
    <script src="/static/node_modules/parsleyjs/dist/i18n/en.js"></script>
    <script src="/static/node_modules/htmx.org/dist/htmx.min.js"></script>

    <link rel="stylesheet"
          href="/static/darkmode.css?v=dev" />
    <link rel="stylesheet"
          type="text/css"
          href="/static/style.css?v=dev" />
    <script src="/static/js/theme.js"></script>
    <script>toastr.options.closeButton = true;</script>
    <!-- For additional head -->

  </head>
  <body>
    <div class="page">


      <div class="container">
        <!-- For flash messages -->

          <!-- Categories: success (green), info (blue), warning (yellow), danger (red) -->


            <script>toastr.success("Alias squirm_bunkum571@sl.local has been created");</script><script>toastr.success("Alias lasing_slowly638@sl.local has been created");</script><script>toastr.success("Alias header_blowup482@sl.local has been created");</script><script>toastr.success("Alias waxing_boxing456@sl.local has been created");</script><script>toastr.success("Alias yuccas_loveys294@sl.local has been created");</script>


      </div>


  <div class="page-content">
    <div class="container text-center">
      <div class="display-3 text-muted mb-5">
        <i class="si si-exclamation"></i>
        429
      </div>
      <h3 class="h3 mb-4">
        Whoa, slow down there, pardner!
      </h3>


  <a class="btn btn-primary" href="/">
    <i class="fe fe-home mr-2"></i>Home Page
  </a>

    </div>
  </div>

    </div>
    <script>


  // default options for bootbox
  bootbox.setDefaults({
    closeButton: false,
    backdrop: true
  })

  var clipboard = new ClipboardJS('.clipboard');

  clipboard.on('success', function (e) {
    toastr.success("Copied to clipboard");
    e.clearSelection();
  });

  // Handle back or close button
  $('.back-or-close').on("click", function () {
    // the window is actually a popup, in this case just close it
    if (history.length == 1) {
      window.close();
    } else {
      history.back();
    }
  });

  document.body.addEventListener('htmx:responseError', function(evt) {
    toastr.error("Sorry for the inconvenience! Could you refresh the page & retry please?", "Unknown Error");
  });

    </script>
    <script src="/static/local-storage-polyfill.js"></script>
    <script src="/static/js/an.js?v=2"></script>
    <!-- For additional script -->

  </body>
</html>
```

### B3′ — Non-numeric `generator_scheme` (NON-CANONICAL input) → HTTP 500 (OBSERVED)

The random form only ever submits `generator_scheme` values `1` or `2` (word / uuid). A **crafted** `generator_scheme=abc` (NON-CANONICAL — the browser dropdown cannot produce it) reaches `app/dashboard/views/index.py:99` where the raw form value is passed to `int(...)`:

```python
                scheme = int(
                    request.form.get("generator_scheme") or current_user.alias_generator
                )
```

OBSERVED: **HTTP 500** (CL 5749), alias count delta **+0**, rendered `templates/error/500.html` (via the generic `Exception` handler `server.py:388-395`). Server-log tail, verbatim:

```
  File "/app/app/dashboard/views/index.py", line 99, in index
    scheme = int(
ValueError: invalid literal for int() with base 10: 'abc'
```

### B4 — Per-user token bucket + Redlock parallel limiter (OBSERVED: both are NO-OPS by default)

There is a **second** rate-limit mechanism inside `Alias.create` (`app/models.py:1636-1642`) — a per-user token bucket (`app/rate_limiter.py`) — plus a Redlock concurrency guard (`@parallel_limiter.lock`, `app/parallel_limiter.py`). **Both are wired to Redis only when `MEM_STORE_URI` is set**, via the gate at `server.py:163-165`:

```python
    if MEM_STORE_URI:
        app.config[flask_limiter.extension.C.STORAGE_URL] = MEM_STORE_URI
        initialize_redis_services(app, MEM_STORE_URI)
```

`initialize_redis_services` (`app/redis_services.py:9-14`) is what calls `set_redis_concurrent_lock(...)` for **both** modules. In the canonical default `MEM_STORE_URI=None`, so this block is **skipped** and both module globals stay `None`. OBSERVED (via the same app-init path the server uses):

```
MEM_STORE_URI                = None
initialize_redis_services invoked at startup? -> False
rate_limiter.lock_redis      = None
parallel_limiter.lock_redis  = None
```

Consequences in default config:
- **Token bucket is a no-op** — `check_bucket_limit` returns immediately at `app/rate_limiter.py:28-29` (`if not lock_redis: return`). OBSERVED: 60 sequential calls with `max_hits=50` raised nothing and returned `None`.
- **Redlock guard is a pass-through** — `@parallel_limiter.lock`'s inner `decorated` returns the wrapped function directly at `app/parallel_limiter.py:51-52` (`if not lock_redis: return f(*args, **kwargs)`). (This is why the B6 race below is *possible* at all.)
- **Behavioral confirmation:** across **every** alias created in this whole investigation, the server log contains **0** occurrences of `"Cannot connect to redis"` or `"Rate limit hit for alias_create"` (`grep -c` = 0) — i.e. the bucket never even *attempts* a Redis call canonically.

Thresholds (parsed, runtime-confirmed; used only when the bucket is armed): `ALIAS_CREATE_RATE_LIMIT_FREE = [(10, 900), (50, 3600)]`, `ALIAS_CREATE_RATE_LIMIT_PAID = [(50, 900), (200, 3600)]` (`app/config.py:554-558`); the key is `f"alias_create_{seconds}d:{user.id}"`.

**NON-CANONICAL demo A — bucket ENFORCES when armed** (crafted: `set_redis_concurrent_lock(RedisStorage("redis://localhost:6379"))`, `max_hits=5`). This exercises the `value > max_hits` branch (`app/rate_limiter.py:30-40`). OBSERVED, verbatim:

```
  call 1: OK (under limit)
  call 2: OK (under limit)
  call 3: OK (under limit)
  call 4: OK (under limit)
  call 5: OK (under limit)
2026-07-13 19:45:34,214 - SL - INFO - 5356 - "/app/app/rate_limiter.py:33" - check_bucket_limit() -  - Rate limit hit for alias_create_demo:1783971934 (bucket id 1783969200) -> 6/5
  call 6: RAISED werkzeug.exceptions.TooManyRequests (429)
2026-07-13 19:45:34,214 - SL - INFO - 5356 - "/app/app/rate_limiter.py:33" - check_bucket_limit() -  - Rate limit hit for alias_create_demo:1783971934 (bucket id 1783969200) -> 7/5
  call 7: RAISED werkzeug.exceptions.TooManyRequests (429)
```

**NON-CANONICAL demo B — FAIL-OPEN when Redis is unreachable** (crafted: a temporary observation script set `app.rate_limiter.lock_redis` to a client pointed at a dead `redis://localhost:6399`, then called `check_bucket_limit(...)`). This exercises the `except (redis.exceptions.RedisError, AttributeError)` clause at `app/rate_limiter.py:41-42`. OBSERVED: the `lock_redis.incr(...)` call at `app/rate_limiter.py:31` raised `redis.exceptions.ConnectionError: Error 111 connecting to localhost:6399. Connection refused.` (a `RedisError` subclass); the `except` caught it, logged one `ERROR` line via `LOG.e("Cannot connect to redis")` (`:42`), and the function then **returned `None` with no exception propagated** (fail-open) — the script's own outcome assertion confirmed `check_bucket_limit` returned `None`. The handler's `ERROR` log line, complete and verbatim (OBSERVED):

```
2026-07-13 19:45:34,214 - SL - ERROR - 5356 - "/app/app/rate_limiter.py:42" - check_bucket_limit() -  - Cannot connect to redis
```

### B5 — Custom-alias validations (OBSERVED; 14 branches)

The custom route validates in this **order** (`app/dashboard/views/custom_alias.py:55-172`): (1) CSRF → (2) read `prefix` (`.strip()…` at `:59`) → (3) `check_alias_prefix` → (4) **mailbox loop** (`:73-83`) → (5) "no mailboxes" (`:85`) → (6) suffix signature (`:88-98`) → (7) `verify_prefix_suffix` (`:100`) → (8) `".."` (`:103`) → (9) `validate_email` (`:107`) → (10) duplicate/trash checks (`:117-135`) → (11) create + `Session.flush()` (`:138-145`) → success (`:158-161`). Note the mailbox loop runs **before** the suffix check.

**Browser client-side gate first (M8; OBSERVED in Chrome).** The prefix `<input>` carries a **Parsley** validator `data-parsley-pattern="[0-9a-z-_.]{1,}"` together with `maxlength="40"` and HTML5 `required` (`templates/dashboard/custom_alias.html:33-39`), and the `#create` click handler (`:112-129`) additionally blocks the submit (with a `toastr.error`, `return`) when the mailbox list is empty (`"You must select at least a mailbox"`) or the prefix is empty (`"Alias cannot be empty"`) (the mailbox `<select>` is also `required`, `:74`). So the browser **does** validate the prefix **charset** (the Parsley pattern), **length** (`maxlength=40`), and **presence** (`required` + the empty-prefix handler) before any request leaves the page — directly OBSERVED via Chrome DevTools below. What the client gate does **not** catch is **consecutive dots** (the pattern permits `.`, so `a..b` passes Parsley), the **signed suffix**, or **mailbox-id** validity — the latter two are supplied from `<select>`s of valid options. Rows below marked **(NC)** are only reachable by POSTing crafted fields that bypass this gate; rows marked **(C)** a real browser can produce. In all rows the **route response is a genuine observation**; alias delta is **+0** except the baseline create.

| # | Condition | | HTTP | Target | Flash (category, message) | Δ |
|---|---|---|---|---|---|---|
| 1 | bad prefix `bad!x` (invalid char — **crafted**; the browser's Parsley gate blocks it, OBSERVED below) | NC | **302** | →`custom_alias` | `error` — `Only lowercase letters, numbers, dashes (-), dots (.) and underscores (_) are currently supported for alias prefix. Cannot be more than 40 letters` | +0 |
| 2 | **missing `prefix` field** | NC | **500** | (500 page) | — (`AttributeError`, see below) | +0 |
| 3 | empty prefix `""` | NC | **302** | →`custom_alias` | `error` — *(same charset message as #1)* | +0 |
| 4 | long prefix (41 chars, crafted — the `prefix` input's `maxlength="40"` truncates typed input in-browser) | NC | **302** | →`custom_alias` | `error` — *(same charset message as #1)* | +0 |
| 5 | consecutive dots `a..b` | C | **302** | →`custom_alias` | `error` — `Your alias can't contain 2 consecutive dots (..)` | +0 |
| 6 | **tampered suffix** (flip a sig char) | NC | **302** | →`custom_alias` | `warning` — `Alias creation time is expired, please retry` | +0 |
| 7 | **missing suffix field** | NC | **302** | →`custom_alias` | `error` — `Unknown error, refresh the page` | +0 |
| 8 | empty suffix `""` | NC | **302** | →`custom_alias` | `warning` — `Alias creation time is expired, please retry` | +0 |
| 9 | **expired suffix** (ts backdated 700s>600s) | NC | **302** | →`custom_alias` | `warning` — `Alias creation time is expired, please retry` | +0 |
| 10 | missing `mailboxes` field | NC | **302** | →`custom_alias` | `error` — `At least one mailbox must be selected` | +0 |
| 11 | tampered mailbox id `99999` | NC | **302** | →`custom_alias` | `warning` — `Something went wrong, please retry` | +0 |
| 12 | verify_prefix_suffix=False (suffix domain not available) | NC | **200** | (re-render) | `warning` — `something went wrong` | +0 |
| 13a | baseline create `dupobs71594` | C | **302** | →`dashboard.index?highlight_alias_id=36` | `success` — `Alias dupobs71594@old.com has been created` | **+1** |
| 13b | duplicate (own) — re-POST same prefix | C | **200** | (re-render) | `error` — `You already have this alias dupobs71594@old.com` | +0 |

**Browser validation of `bad!x` — OBSERVED in Chrome DevTools (canonical entry point).** Logged in as `john@wick.com`, navigating to `/dashboard/custom_alias`, typing `bad!x` into the prefix field and clicking **Create** issues **no request at all**: Parsley rejects the `!` against `data-parsley-pattern="[0-9a-z-_.]{1,}"` and renders the inline error **`Only lowercase letters, dots, numbers, dashes (-) and underscores (_) are currently supported.`** (the field's `data-parsley-error-message`), the input gains class `parsley-error`, and the URL stays on `/dashboard/custom_alias`. `list_network_requests` for the interaction shows **no `POST /dashboard/custom_alias`** — only the page-load `GET /dashboard/custom_alias` (200) and the background `GET /api/notifications?page=0` (200) — and the database is unchanged (`SELECT count(*) FROM alias` = 39, `max(id)` = 39, identical to the pre-click baseline). A control prefix `validprefix123` flips the input to class `parsley-success` with the error list empty. Screenshot: `blitzy/screenshots/docfix_doc2_custom_bad_prefix_parsley_blocked.png`. **Therefore row 1's `302` is a NON-CANONICAL observation** — reachable only by a crafted `POST` (e.g. `curl`) that skips the browser; the server-side `check_alias_prefix` (message at `custom_alias.py:65`) then returns the longer server message shown in the table. The client (Parsley) and server messages differ verbatim: Parsley ends *"…dashes (-) and underscores (_) are currently supported."*, whereas the server adds *"…for alias prefix. Cannot be more than 40 letters"*.

**Counter-intuitive finding (empirically confirmed).** The flash text does *not* map to a naive "tampered vs. expired" model. `check_suffix_signature` (`app/alias_suffix.py:37-42`) catches `itsdangerous.BadSignature` — which subsumes **both** a bad signature **and** `SignatureExpired` — and returns `None`, so the route's `:89` branch fires ("…expired") for a **tampered string** (row 6), an **empty** string (row 8) *and* a genuinely **expired** suffix (row 9). The route's `:95 except` "tampered" branch (`"Unknown error, refresh the page"`) is reached **only when `check_suffix_signature` raises** a non-`BadSignature` — e.g. a **missing** suffix field, where `signer.unsign(None)` raises inside the library (row 7). The accompanying `LOG.w` lines (verbatim) prove which branch each hit:

```
2026-07-13 19:38:53,197 - SL - WARNING - 5030 - "/app/app/dashboard/views/custom_alias.py:92" - custom_alias() -  - Alias creation time expired for <User 1 John Wick john@wick.com>
2026-07-13 19:38:53,334 - SL - WARNING - 5030 - "/app/app/dashboard/views/custom_alias.py:96" - custom_alias() -  - Alias suffix is tampered, user <User 1 John Wick john@wick.com>
2026-07-13 19:38:53,479 - SL - WARNING - 5030 - "/app/app/dashboard/views/custom_alias.py:92" - custom_alias() -  - Alias creation time expired for <User 1 John Wick john@wick.com>
2026-07-13 19:39:54,101 - SL - WARNING - 5030 - "/app/app/dashboard/views/custom_alias.py:92" - custom_alias() -  - Alias creation time expired for <User 1 John Wick john@wick.com>
```

**Row 2 — missing `prefix` field → HTTP 500 (OBSERVED).** `request.form.get("prefix")` is `None`, and `None.strip()` raises. Server-log tail (with the full middleware chain that wraps the view), verbatim:

```
  File "/app/venv/lib/python3.10/site-packages/flask_debugtoolbar/__init__.py", line 125, in dispatch_request
    return view_func(**req.view_args)
  File "/usr/local/lib/python3.10/cProfile.py", line 110, in runcall
    return func(*args, **kw)
  File "/app/venv/lib/python3.10/site-packages/flask_limiter/extension.py", line 702, in __inner
    return obj(*a, **k)
  File "/app/venv/lib/python3.10/site-packages/flask_login/utils.py", line 272, in decorated_view
    return func(*args, **kwargs)
  File "/app/app/parallel_limiter.py", line 52, in decorated
    return f(*args, **kwargs)
  File "/app/app/dashboard/views/custom_alias.py", line 59, in custom_alias
    alias_prefix = request.form.get("prefix").strip().lower().replace(" ", "")
AttributeError: 'NoneType' object has no attribute 'strip'
```

**HTTP-code partition of the custom route (OBSERVED):** `302 redirect(request.url)` (→ `/dashboard/custom_alias`) for prefix/dots/suffix/mailbox validation failures; **500** for a missing `prefix` field; **200 `render_template`** for the fall-through cases (duplicate own/other, `DomainDeletedAlias`, `DeletedAlias`, `verify_prefix_suffix=False`); **302 redirect** to `dashboard.index` on success. This confirms the §4 finding that **200 re-renders occur only on the custom route** — the random route is fully Post/Redirect/Get.

**Row 12 — `verify_prefix_suffix` returns `False` (OBSERVED, NON-CANONICAL input).** A validly-signed suffix for a domain **not** in `current_user.available_alias_domains` (crafted `@notmydomain.xyz`) passes the signature check but fails `verify_prefix_suffix` (`:100`), so the `else` at `:162-164` (`# only happen if the request has been "hacked"`) flashes the lowercase `"something went wrong"` (`warning`) and falls through to `render_template` → **HTTP 200**. (Distinct from the mailbox-tampered `"Something went wrong, please retry"` at `:81`.)

**"Recreate DELETED" (DomainDeletedAlias) — OBSERVED end-to-end (CANONICAL).** Create → delete → recreate the same custom-domain alias:

```
1) CREATE deltest72502@old.com: HTTP 302 id=57 flash=[('success', 'Alias deltest72502@old.com has been created')]
2) DELETE id=57: HTTP 302 flash=[('success', 'Alias deltest72502@old.com has been deleted')]  domain_deleted_alias 0->1 (+1)
   in trash? domain_deleted_alias row for deltest72502@old.com: 1
3) RECREATE deltest72502@old.com: HTTP 200 OK flash=[('error', 'You have deleted this alias before. You can restore it on old.com &#39;Deleted Alias&#39; page')]  (alias delta +0)
```

The delete (`form-name=delete-alias` + `alias-id`, `app/dashboard/views/index.py:125-155`, **exempt** from `ALIAS_LIMIT`) calls `alias_utils.delete_alias` (`app/alias_utils.py:336-378`), which for a custom-domain alias inserts a `domain_deleted_alias` row (`:348-360`) — making the recreate hit the `DomainDeletedAlias.get_by` branch (`custom_alias.py:123-132`, flash at `:128`) → **HTTP 200 re-render**.

**Line-level source anchors** (strings verified verbatim against source): charset error `:65`; consecutive-dots `:104` (block `:103-105`; source uses a literal apostrophe, rendered HTML-escaped as `can&#39;t`); suffix "expired" `:91-93` (`check_suffix_signature` returns `None`, `app/alias_suffix.py:41-42`); suffix "tampered" `except Exception` → `LOG.w :96` + flash `:97`; "no mailboxes" `:86`; mailbox-tampered `:81`; duplicate-own `:120` (`Alias.get_by`); `DomainDeletedAlias` `:128`.

### B6 — IntegrityError → rollback (OBSERVED: real concurrent race + deterministic component repro)

The intended DB-level duplicate/race handler is `app/dashboard/views/custom_alias.py:138-150`:

```python
                try:
                    alias = Alias.create(
                        user_id=current_user.id,
                        email=full_alias,
                        note=alias_note,
                        mailbox_id=mailboxes[0].id,
                    )
                    Session.flush()
                except IntegrityError:
                    LOG.w("Alias %s already exists", full_alias)
                    Session.rollback()
                    flash("Unknown error, please retry", "error")
                    return redirect(url_for("dashboard.custom_alias"))
```

This differs from the **application-level** duplicate check in B5 (which catches an existing alias *before* the INSERT → HTTP 200 + `"You already have this alias …"`): the `IntegrityError` path only fires when a duplicate **slips past** that check concurrently and is stopped by the `gen_email_email_key` UNIQUE constraint (`alias.email`, `app/models.py:1477`).

**Why a real race is reachable in default config.** `app.run(debug=True, port=7777)` runs the Werkzeug dev server **threaded** (Flask forces `options.setdefault("threaded", True)`, `flask/app.py:983`), and `@parallel_limiter.lock` is a **no-op** by default (B4), so nothing serializes concurrent `create-custom` requests. Crucially, `app/db.py` opens **one** module-level connection — `connection = engine.connect()` (`:12`) — and binds the scoped session to it — `Session = scoped_session(sessionmaker(bind=connection))` (`:14`). So all concurrent request threads share **one** PostgreSQL connection/transaction.

**REAL CANONICAL RACE (OBSERVED).** Firing K=5 simultaneous POSTs (a `threading.Barrier`) with an identical brand-new prefix, on 3 separate attempts (re-captured fresh on **2026-07-14**, app PID **38143**, to supply the complete unedited failure output below), the `IntegrityError` branch was reached every time — `LOG.w "Alias … already exists"` at `custom_alias.py:147` fired on all 3. Attempt 0 verbatim:

```
ATTEMPT 0 prefix=b6race1784038383_0
   thread0: HTTP 302 Loc=http://127.0.0.1:7777/dashboard/custom_alias
   thread1: HTTP 500 Loc=-
   thread2: HTTP 500 Loc=-
   thread3: HTTP 500 Loc=-
   thread4: HTTP 500 Loc=-
   #302=1 #500=4 #429=0
```

The HTTP split across the K=5 threads is **timing-dependent, not fixed**. The number of threads that return a **302** varies run to run: the three attempts produced **1, 1, and 2 of 5** returning 302 respectively (tabulated below). A returned 302 here is the `IntegrityError` branch's `redirect(url_for("dashboard.custom_alias"))` at `:150` (every observed 302 carried `Location: …/dashboard/custom_alias`); the remaining threads return **HTTP 500** because the duplicate INSERT aborts the **shared** transaction, so their subsequent statements fail. The invariant that holds on *every* run is the database one: the `gen_email_email_key` UNIQUE constraint (`alias.email`, `app/models.py:1477`) permits **at most one** surviving row. The two 500-side failure modes follow, each **complete and unedited** (only end-of-line whitespace stripped from the wrapped `[SQL: …]` continuation lines; no token changed). The first is a **reader** thread: its `DeletedAlias.get_by(...)` duplicate-check read at `custom_alias.py:134` runs a `SELECT deleted_alias …` on the already-aborted shared transaction and raises `InFailedSqlTransaction`:

```
2026-07-14 14:13:05,487 - SL - ERROR - 38143 - "/app/server.py:390" - error_handler() -  - (psycopg2.errors.InFailedSqlTransaction) current transaction is aborted, commands ignored until end of transaction block

[SQL: SELECT deleted_alias.id AS deleted_alias_id, deleted_alias.created_at AS deleted_alias_created_at, deleted_alias.updated_at AS deleted_alias_updated_at, deleted_alias.email AS deleted_alias_email, deleted_alias.reason AS deleted_alias_reason
FROM deleted_alias
WHERE deleted_alias.email = %(email_1)s
 LIMIT %(param_1)s]
[parameters: {'email_1': 'b6race1784038383_0@old.com', 'param_1': 1}]
(Background on this error at: http://sqlalche.me/e/13/2j85)
Traceback (most recent call last):
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1276, in _execute_context
    self.dialect.do_execute(
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/default.py", line 608, in do_execute
    cursor.execute(statement, parameters)
psycopg2.errors.InFailedSqlTransaction: current transaction is aborted, commands ignored until end of transaction block


The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "/app/venv/lib/python3.10/site-packages/flask/app.py", line 1950, in full_dispatch_request
    rv = self.dispatch_request()
  File "/app/venv/lib/python3.10/site-packages/flask_debugtoolbar/__init__.py", line 125, in dispatch_request
    return view_func(**req.view_args)
  File "/usr/local/lib/python3.10/cProfile.py", line 110, in runcall
    return func(*args, **kw)
  File "/app/venv/lib/python3.10/site-packages/flask_limiter/extension.py", line 702, in __inner
    return obj(*a, **k)
  File "/app/venv/lib/python3.10/site-packages/flask_login/utils.py", line 272, in decorated_view
    return func(*args, **kwargs)
  File "/app/app/parallel_limiter.py", line 52, in decorated
    return f(*args, **kwargs)
  File "/app/app/dashboard/views/custom_alias.py", line 134, in custom_alias
    elif DeletedAlias.get_by(email=full_alias):
  File "/app/app/models.py", line 84, in get_by
    return Session.query(cls).filter_by(**kw).first()
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/orm/query.py", line 3429, in first
    ret = list(self[0:1])
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/orm/query.py", line 3203, in __getitem__
    return list(res)
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/orm/query.py", line 3535, in __iter__
    return self._execute_and_instances(context)
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/orm/query.py", line 3560, in _execute_and_instances
    result = conn.execute(querycontext.statement, self._params)
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1011, in execute
    return meth(self, multiparams, params)
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/sql/elements.py", line 298, in _execute_on_connection
    return connection._execute_clauseelement(self, multiparams, params)
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1124, in _execute_clauseelement
    ret = self._execute_context(
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1316, in _execute_context
    self._handle_dbapi_exception(
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1510, in _handle_dbapi_exception
    util.raise_(
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/util/compat.py", line 182, in raise_
    raise exception
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1276, in _execute_context
    self.dialect.do_execute(
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/default.py", line 608, in do_execute
    cursor.execute(statement, parameters)
sqlalchemy.exc.InternalError: (psycopg2.errors.InFailedSqlTransaction) current transaction is aborted, commands ignored until end of transaction block

[SQL: SELECT deleted_alias.id AS deleted_alias_id, deleted_alias.created_at AS deleted_alias_created_at, deleted_alias.updated_at AS deleted_alias_updated_at, deleted_alias.email AS deleted_alias_email, deleted_alias.reason AS deleted_alias_reason
FROM deleted_alias
WHERE deleted_alias.email = %(email_1)s
 LIMIT %(param_1)s]
[parameters: {'email_1': 'b6race1784038383_0@old.com', 'param_1': 1}]
(Background on this error at: http://sqlalche.me/e/13/2j85)
```

The other 500-side mode is a **committer** thread: it reached `Session.commit()` at `custom_alias.py:158`, but the shared transaction had already been rolled back beneath it, so `commit()` raised `InvalidRequestError`:

```
2026-07-14 14:13:05,494 - SL - ERROR - 38143 - "/app/server.py:390" - error_handler() -  - This transaction is inactive
Traceback (most recent call last):
  File "/app/venv/lib/python3.10/site-packages/flask/app.py", line 1950, in full_dispatch_request
    rv = self.dispatch_request()
  File "/app/venv/lib/python3.10/site-packages/flask_debugtoolbar/__init__.py", line 125, in dispatch_request
    return view_func(**req.view_args)
  File "/usr/local/lib/python3.10/cProfile.py", line 110, in runcall
    return func(*args, **kw)
  File "/app/venv/lib/python3.10/site-packages/flask_limiter/extension.py", line 702, in __inner
    return obj(*a, **k)
  File "/app/venv/lib/python3.10/site-packages/flask_login/utils.py", line 272, in decorated_view
    return func(*args, **kwargs)
  File "/app/app/parallel_limiter.py", line 52, in decorated
    return f(*args, **kwargs)
  File "/app/app/dashboard/views/custom_alias.py", line 158, in custom_alias
    Session.commit()
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/orm/scoping.py", line 163, in do
    return getattr(self.registry(), name)(*args, **kwargs)
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/orm/session.py", line 1046, in commit
    self.transaction.commit()
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/orm/session.py", line 508, in commit
    t[1].commit()
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1761, in commit
    raise exc.InvalidRequestError("This transaction is inactive")
sqlalchemy.exc.InvalidRequestError: This transaction is inactive
```

The single **302** thread is the one that caught its *own* `IntegrityError` at `Session.flush()` and took the `:146-150` branch; its `LOG.w` line at `custom_alias.py:147`, verbatim:

```
2026-07-14 14:13:05,489 - SL - WARNING - 38143 - "/app/app/dashboard/views/custom_alias.py:147" - custom_alias() -  - Alias b6race1784038383_0@old.com already exists
```

> **Completeness note** (mirroring §5.2 and §7 B3): the three blocks above are the **complete, unedited** server-log records for the two 500-side threads and the 302 thread — the full five-column `SELECT deleted_alias.id … deleted_alias.reason FROM deleted_alias` projection and both entire tracebacks are reproduced end to end; the only normalization is end-of-line whitespace stripped from the wrapped `[SQL: …]` continuation lines (no token changed). **Nothing is elided.** The `psycopg2.errors.InFailedSqlTransaction` "current transaction is aborted" message, the `LOG.w` line at `custom_alias.py:147`, the failing `Session.commit()` at `custom_alias.py:158`, and the terminal `sqlalchemy.exc.InvalidRequestError: This transaction is inactive` are all present verbatim above.

`after_request` logged the mix (the three 500 threads recorded, e.g., `… POST /dashboard/custom_alias … 500, takes 0.347` / `0.354` / `0.360`, alongside the 302 thread). The number of **surviving DB rows** for the raced email was **exactly 1 on all three attempts** in this session (a commit landed before the abort cascade each time). The invariant that holds on *every* run is the UNIQUE constraint — **never more than one** row survives; a **0**-survivor outcome is also possible under different interleaving (**INFERRED** — because all threads share one connection, a losing thread's `Session.rollback()` can undo the winner's already-flushed INSERT). The per-attempt distribution, spaced >60s apart to clear the `5/minute` route budget:

```
attempt | #302 | #500 | #429 | surviving_rows
   0    |   1  |   4  |   0  |       1
   1    |   1  |   4  |   0  |       1
   2    |   2  |   3  |   0  |       1
```

This is the true, directly-observed canonical behavior: under concurrent identical submissions the single shared connection turns most losers into **HTTP 500** (a run-varying **3–4 of 5**), a run-varying **1–2 of 5** into a 302 via the `IntegrityError` branch, and persists **at most one** row — the `gen_email_email_key` UNIQUE constraint (`app/models.py:1477`) guarantees the surviving count is **never more than one** (it was exactly **1** on each of the three fresh attempts above).

**DETERMINISTIC COMPONENT REPRO (OBSERVED; NON-CANONICAL input).** To isolate the `:146-150` branch cleanly (bypassing the `get_by` guard by inserting the same email twice on one session), verbatim:

```
1st Alias.create committed: id=54 email=compdup1783972315@sl.local
IntegrityError RAISED at Session.flush():
    (raised as a result of Query-invoked autoflush; consider using a session.no_autoflush block if this flush is occurring prematurely)
route branch: LOG.w('Alias %s already exists'); Session.rollback(); flash('Unknown error, please retry','error'); redirect(dashboard.custom_alias)
rows for compdup1783972315@sl.local after rollback of the 2nd = 1 (UNIQUE constraint held; exactly one persisted)
```

**HEALTH CHECK (OBSERVED).** Immediately after the race, a normal random create via the route returned `HTTP 302 … highlight_alias_id=27` — the shared connection recovers per-request (the request teardown calls `Session.remove()`), so the messy race leaves the server healthy.

### Remaining sub-branches NOT reproduced at runtime (labelled INFERRED)

- **`validate_email` raising `EmailNotValidError`** → `flash(str(e), "error")` at `app/dashboard/views/custom_alias.py:107-113`. **INFERRED** — attempted but not reached canonically: `check_alias_prefix` (`:64`) already rejects any prefix outside `[a-z0-9._-]{1,40}`, and a valid prefix + a valid signed suffix always forms a deliverable address, so `validate_email(full_alias, check_deliverability=False, allow_smtputf8=False)` does not raise for reachable inputs.
- **Global `DeletedAlias` (SL-public-domain) recreation** → `flash(general_error_msg, "error")` (`f"{full_alias} cannot be used"`) at `app/dashboard/views/custom_alias.py:134-135`. **INFERRED** — the custom-domain analogue *was* observed (B5, `DomainDeletedAlias`); the public-domain branch is not deterministically reachable through the canonical custom form because the SL-public suffix carries a fresh random token on every page load, so a recreate produces a **different** `full_alias` than the trashed one and never matches `DeletedAlias.get_by(email=full_alias)`.

---

## Section 8 — Web vs. API Contrast (OBSERVED)

Both entry points share the **same `Alias.create` core** (`app/models.py:1627-1692`) — the same three-table write-set observed in §5 (`INSERT INTO alias` + `UPDATE daily_metric` + `INSERT INTO alias_audit_log`). **However, the API write-set is NOT identical to the web write-set.** An authenticated API request first passes through `authorize_request` (`app/api/base.py:16-32`), which performs an **additional `UPDATE api_key` (usage-stat bump) in its own separately-committed transaction** *before* the alias core runs (`app/api/base.py:30-32`). The web (session-cookie) path has no API key and never touches the `api_key` table. This extra write was captured live below (§8.2, finding M6). Beyond that one extra write, the alias core and its side effects are identical; the HTTP envelope also differs.

| Aspect | WEB `POST /dashboard/` (`app/dashboard/views/index.py:55`) | API `POST /api/alias/random/new` (`app/api/views/new_random_alias.py:21`) |
|--------|-------------------------------------------------------------|----------------------------------------------------------------------------|
| Auth | session cookie `slapp` + CSRF token | header `Authentication: <key>` (`app/api/base.py:17-18`), **NO CSRF** |
| Auth side-effect (DB write) | none — a cookie session has no `api_key` row | **`UPDATE api_key` (`last_used`, `times`)** in its own committed txn (`app/api/base.py:30-32`) |
| Success status | **HTTP 302** (Post/Redirect/Get) | **HTTP 201 CREATED** |
| Success body | redirect + `Location …highlight_alias_id=<id>` + toastr flash | JSON alias object (with `note` echoed) |
| Tables written on success | **3** — `alias`, `daily_metric`, `alias_audit_log` (§5) | **4** — the same 3 **plus** `api_key` |
| Response serialization | none (302 redirect; empty body) | extra **read-only** txn (`get_alias_info_v2`) ending in `ROLLBACK` |
| Rate-limit response | **HTTP 429 HTML** (`templates/error/429.html`) | **HTTP 429 JSON** `{"error": "Rate limit exceeded"}` |
| Quota response | **HTTP 302** + warning flash | **HTTP 400 JSON** |

### 8.1 API happy path (OBSERVED, verbatim — fresh canonical capture)

Captured with a **disposable local API key** created via the ORM for `john@wick.com` (id=1) and deleted afterward (§8.4); the key value is redacted as `<APIKEY>` and the `slapp` cookie value as `<REDACTED>`.

Producing command:

```
docker exec sl-app curl -sS -i -X POST http://127.0.0.1:7777/api/alias/random/new \
  -H "Authentication: <APIKEY>" \
  -H "Content-Type: application/json" \
  -d '{"note":"api-obs-note"}'
```

Full, unedited response:

```
HTTP/1.0 201 CREATED
Content-Type: application/json
Content-Length: 547
Access-Control-Allow-Origin: *
Vary: Cookie
Set-Cookie: slapp=<REDACTED>; Expires=Mon, 20-Jul-2026 20:09:28 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Mon, 13 Jul 2026 20:09:28 GMT

{
  "alias": "gibber_gonged860@sl.local",
  "creation_date": "2026-07-13 20:09:28+00:00",
  "creation_timestamp": 1783973368,
  "disable_pgp": false,
  "email": "gibber_gonged860@sl.local",
  "enabled": true,
  "id": 58,
  "latest_activity": null,
  "mailbox": {
    "email": "john@wick.com",
    "id": 1
  },
  "mailboxes": [
    {
      "email": "john@wick.com",
      "id": 1
    }
  ],
  "name": null,
  "nb_block": 0,
  "nb_forward": 0,
  "nb_reply": 0,
  "note": "api-obs-note",
  "pinned": false,
  "support_pgp": false
}
```

The success return is `jsonify(alias=alias.email, **serialize_alias_info_v2(get_alias_info_v2(alias))), 201` at `app/api/views/new_random_alias.py:114-116`. Two observed facts to note:

- **`"note": "api-obs-note"` echoes the request input** (finding M7). The request body supplied `{"note":"api-obs-note"}`, and the response serializes it back via `serialize_alias_info_v2` at `app/api/serializer.py:63` (`"note": alias_info.alias.note`). A `null` note is impossible for a request that supplied one — the value is round-tripped through the `alias.note` column (see the `INSERT INTO alias (..., 'api-obs-note', 1, ...)` in §8.2).
- The `Server:` header reports **`Werkzeug/1.0.1 Python/3.10.18`** — the canonical image runtime (`python3 -V` → `Python 3.10.18`, §1), matching every other capture in this document.

### 8.2 The API `api_key` write side-effect (OBSERVED — finding M6)

Every authenticated API call is wrapped by `@require_api_auth` (`app/api/base.py:52`), whose `authorize_request` (`app/api/base.py:16`) looks the key up with `ApiKey.get_by(code=api_code)` (`app/api/base.py:18`) and, when it matches, **bumps the key's usage stats and commits them in a dedicated transaction** before the view body runs:

```
api_key.last_used = arrow.now()   # app/api/base.py:30
api_key.times += 1                # app/api/base.py:31
Session.commit()                  # app/api/base.py:32
```

To observe this, PostgreSQL statement logging (`log_statement='all'`, §5) was already active; a disposable key was created for `john@wick.com`, one happy-path call was issued, and the per-request statement delta was filtered to transaction-control + write statements.

Producing command (filter of the request's slice of `/var/log/postgresql/postgresql-15-main.log`):

```
sed -n '12026,12119p' $PGLOG | grep -aoE 'statement: (BEGIN|COMMIT|ROLLBACK|INSERT INTO [a-z_]+|UPDATE [a-z_]+)'
```

Complete output of that command (three transactions):

```
statement: BEGIN
statement: UPDATE api_key
statement: COMMIT
statement: BEGIN
statement: INSERT INTO alias
statement: UPDATE daily_metric
statement: INSERT INTO alias_audit_log
statement: COMMIT
statement: BEGIN
statement: ROLLBACK
```

- **Txn 1** (`authorize_request`, `app/api/base.py:16-32`): the `UPDATE api_key` stat bump, committed on its own. Verbatim statement (PostgreSQL backend PID 5949):
  ```
  2026-07-13 20:13:49.162 UTC [5949] myuser@simplelogin LOG:  statement: UPDATE api_key SET updated_at='2026-07-13T20:13:49.162256'::timestamp, last_used='2026-07-13T20:13:49.161745'::timestamp, times=1 WHERE api_key.id = 5
  ```
- **Txn 2** (the alias core inside `new_random_alias`, `app/api/views/new_random_alias.py:106-107`): `INSERT INTO alias` → `UPDATE daily_metric` → `INSERT INTO alias_audit_log` → `COMMIT` — byte-for-byte the same three-table write-set captured for the web path in §5 (the intervening read-only pre-check `SELECT`s are identical to §5 and omitted by the grep filter above).
- **Txn 3** (response serialization, `get_alias_info_v2` → `serialize_alias_info_v2`): a **read-only** transaction that issues only `SELECT`s and ends in `ROLLBACK` (no writes). The web path has no analogue because it returns a 302 redirect with an empty body.

The `api_key` row before/after the single call (ORM view; key value redacted):

| | `id` | `last_used` | `times` |
|--|------|-------------|---------|
| **Before** | 5 | `None` | `0` |
| **After** | 5 | `Arrow [2026-07-13T20:13:49.161745+00:00]` | `1` |

**Conclusion (M6):** the API success write-set is **four** tables — `api_key` (Txn 1) **plus** the same `alias` / `daily_metric` / `alias_audit_log` three (Txn 2) — whereas the web write-set is **three** (no `api_key`). The earlier claim that the write-sets were "identical" was incorrect.

### 8.3 API rate-limit (429) and quota (400) (OBSERVED, verbatim)

**API rate-limit → HTTP 429 JSON.** After a server restart (clean in-memory limiter), six rapid `POST`s were issued with john's disposable key. Requests #1–#5 returned `HTTP 201`; request #6 returned `HTTP 429`:

```
request #1 -> HTTP 201
request #2 -> HTTP 201
request #3 -> HTTP 201
request #4 -> HTTP 201
request #5 -> HTTP 201
request #6 -> HTTP 429
```

Full, unedited 429 response (cookie value redacted):

```
HTTP/1.0 429 TOO MANY REQUESTS
Content-Type: application/json
Content-Length: 37
Access-Control-Allow-Origin: *
Vary: Cookie
Set-Cookie: slapp=<REDACTED>; Expires=Mon, 20-Jul-2026 20:11:05 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Mon, 13 Jul 2026 20:11:05 GMT

{
  "error": "Rate limit exceeded"
}
```

This body comes from the `/api/`-prefixed branch of the shared 429 handler — `if request.path.startswith("/api/"): return jsonify(error="Rate limit exceeded"), 429` (`server.py:362-372`) — in direct contrast to the web route, which renders the HTML `templates/error/429.html` page (§7, branch B3). The handler also emits a `LOG.w` (`server.py:364`); note that for an API request `current_user` is **anonymous** (the limiter keys by IP, not user id):

```
2026-07-13 20:11:05,018 - SL - WARNING - 5943 - "/app/server.py:364" - rate_limited() -  - Client hit rate limit on path /api/alias/random/new, user:<flask_login.mixins.AnonymousUserMixin object at 0x7f5b361242e0>
```

**API quota exceeded → HTTP 400 JSON.** Issued with a disposable key for the maxed-out free user (`freeobs@example.com`, id=3, at the 5-alias limit; see §7, branch B1):

Producing command:

```
docker exec sl-app curl -sS -i -X POST http://127.0.0.1:7777/api/alias/random/new \
  -H "Authentication: <FREE_APIKEY>" \
  -H "Content-Type: application/json" \
  -d '{"note":"quota-test"}'
```

Full, unedited response (real `MAX_NB_EMAIL_FREE_PLAN` value = **5**, not a placeholder):

```
HTTP/1.0 400 BAD REQUEST
Content-Type: application/json
Content-Length: 146
Access-Control-Allow-Origin: *
Vary: Cookie
Set-Cookie: slapp=<REDACTED>; Expires=Mon, 20-Jul-2026 20:10:15 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Mon, 13 Jul 2026 20:10:15 GMT

{
  "error": "You have reached the limitation of a free account with the maximum of 5 aliases, please upgrade your plan to create more aliases"
}
```

This is the `return jsonify(error=...), 400` at `app/api/views/new_random_alias.py:37-43`, guarded by `if not user.can_create_new_alias()` at `:35`. Because `authorize_request` runs *before* the view body, the free user's `api_key` row is still stat-bumped and committed (Txn 1) even though the request is rejected: the request's statement delta showed `UPDATE api_key ... times=1 WHERE api_key.id = 4` → `COMMIT`, followed by a second transaction of `SELECT`s (users + all subscription tables + a `count(*)` for the quota) ending in `ROLLBACK` — **no** `alias` / `daily_metric` / `alias_audit_log` writes.

### 8.4 Disposal of the observation API keys (M15 — state left clean)

All API keys used above were **disposable, runtime-only keys created via the ORM** (named `blitzy-obs-temp-*`) and were deleted at the end of the investigation; none are committed to the repository (they live only in the ephemeral `api_key` table). Deletion evidence:

```
BEFORE delete: 2 disposable key(s):
  id=3 user_id=1 name='blitzy-obs-temp-john' times=6 last_used=<Arrow [2026-07-13T20:11:04.885880+00:00]>
  id=4 user_id=3 name='blitzy-obs-temp-free' times=1 last_used=<Arrow [2026-07-13T20:10:15.928613+00:00]>
Deleted ids: [3, 4]
AFTER delete: disposable key count = 0
ApiKey.get_by(code=<redacted ...rofh>) -> None
ApiKey.get_by(code=<redacted ...wwnl>) -> None
```

The third key used for the §8.2 exhibit (`id=5`, `blitzy-obs-temp-m6`) was likewise deleted (`deleted id=5; remaining m6 keys = 0`). After deletion, the exact lookup `authorize_request` performs — `ApiKey.get_by(code=...)` (`app/api/base.py:18`) — returns `None`, so reuse of a disposed key **falls through to** `return jsonify(error="Wrong api key"), 401` (`app/api/base.py:27`). This 401 path was **exercised directly and is OBSERVED, not inferred**. An unknown API code — behaviorally identical to reusing a disposed key, since `ApiKey.get_by(code=...)` (`app/api/base.py:18`) returns `None` in both cases — produces a `401`:

```
curl -sS -i -X POST http://127.0.0.1:7777/api/alias/random/new -H 'Authentication: blitzy-nonexistent-deleted-code-xyz' -H 'Content-Type: application/json' -d '{}'
```

Response (status line, content headers, and body reproduced verbatim; the volatile `Set-Cookie`/`Date` headers are not shown):

```
HTTP/1.0 401 UNAUTHORIZED
Content-Type: application/json
Content-Length: 31

{
  "error": "Wrong api key"
}
```

Omitting the `Authentication` header entirely produces the **byte-identical** `401` (`HTTP/1.0 401 UNAUTHORIZED`, `Content-Length: 31`, `{"error": "Wrong api key"}`), because the `if not api_key:` guard (`app/api/base.py:20`) is reached whether `api_code` is `None` (header absent) or an unknown code. (Aside: `times=6` on john's key = 1 happy-path call + 5 successful rate-limit-flood calls; the two throttled `429` requests did **not** increment `times`, because `@limiter.limit` (`app/api/views/new_random_alias.py:22`) runs *before* `@require_api_auth` (`:23`), so `authorize_request` never executed for them.)

---

## Section 9 — Observed vs. Inferred, Canonical vs. Non-canonical, and Coverage (Summary)

### 9.1 Evidence provenance (fresh canonical run — explicit disclosure)

Every value, log line, HTTP response, SQL statement, and error string in this document was **captured fresh, by this investigation, inside the mandated canonical Docker image** — `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (image id `sha256:ea242796bbce…`), running the repository bind-mounted at `/app` on **Python 3.10.18, PostgreSQL 15.13, Redis 7.0.15** (§1). Each capture is shown with its exact producing command and, where relevant, the live process/backend PID (for example PostgreSQL backend PID `36580` in §5's SQL slices, the dev-server/app PID `36792` in §7 B3 and `38143` in §7 B6, and the §8 API session's dev-server PID `5943` (§8.3) with PostgreSQL backend `5949` (§8.2)). Because the investigation spanned several canonical runs — with the database re-bootstrapped (`alembic upgrade head` + `flask dummy-data`) between some of them — per-section timestamps, process PIDs, and alias-id ranges differ (for example §5's `alias` ids 25-28 and §6's `<Alias 29>` come from different runs, each with its own reset id sequence); every value shown is the real, unedited output of the run that produced it. No evidence was borrowed from a supplied capture package. Any earlier-draft values that reflected a **non-canonical environment** (for example a `Server: Werkzeug/1.0.1 Python/3.10.20` banner or a PostgreSQL 16 datastore) were **discarded and are not relied upon**; apart from this one sentence, which names them solely to document their exclusion, no such non-canonical value is used as evidence anywhere in this document.

### 9.2 OBSERVED (default / canonical) — captured at runtime

- **Q1:** login → dashboard (302 → 200), then a single click creates an alias and lands back on the dashboard with a green toastr success flash.
- **Q2:** the frontend sends `POST` with `Content-Type: application/x-www-form-urlencoded`; random body `csrf_token`, `form-name=create-random-email`, optional `generator_scheme`; custom body `csrf_token`, `prefix`, `signed-alias-suffix`, `mailboxes`, `note` (no `form-name`).
- **Q3:** **HTTP 302** Post/Redirect/Get + toastr flash; `Location …highlight_alias_id=<id>` (random adds `&query=&sort=&filter=`); `Set-Cookie: slapp=…; HttpOnly; Path=/; SameSite=Lax`; `Content-Length` 339 (random) / 273 (custom) for a 2-digit alias id, growing 2 bytes per additional id digit (e.g. 341 / 275 for a 3-digit id — see §4.2).
- **Q4:** write-set = **3 tables** for random (`alias` INSERT, `daily_metric` UPDATE, `alias_audit_log` INSERT) and **4 tables** for a custom alias with a secondary mailbox (adds `alias_mailbox` INSERT); `daily_metric` is a **global per-day** counter; `alias_audit_log` is written **synchronously**; **0** `sync_event` rows and **0** `NOTIFY`s across the whole log.
- **Q5:** the domain event is a **gated no-op** (one `INFO` log line); **no background workers** run (`event_listener.py`/`job_runner.py`/`cron.py` all absent from `ps`); the audit log is synchronous, not a job.
- **Q6:** the B1 (quota), B2 (CSRF), B3 (route 429 HTML), B5 (custom-alias validations), and B6 (IntegrityError rollback) outputs; and the **B4 canonical default** finding that the Redis token bucket + Redlock are no-ops (`MEM_STORE_URI=None`).
- **§8:** the API happy path returning **HTTP 201** JSON with the request's `note` **echoed** in the body; the API-only **`api_key` UPDATE + commit** performed by `authorize_request` (making the API success write-set **4 tables** vs. the web's 3); and the API rate-limit / quota responses as **JSON** (`429 {"error": "Rate limit exceeded"}` / `400` with the real value `5`); and the API **401** `{"error": "Wrong api key"}` returned by `authorize_request` (`app/api/base.py:20,27`) for an invalid or missing `Authentication` key (§8.4, exercised directly).

### 9.3 INFERRED (code-derived, not observed) — each grounded in `file:line`

- The intra-transaction **statement ordering** difference between the random and custom paths (SQLAlchemy flush ordering) — the *set* of tables is the canonical, observed fact (§5).
- The **webhook-configured** `sync_event` INSERT + `NOTIFY simplelogin_sync_events` path (`app/events/event_dispatcher.py:71-79` → `PostgresDispatcher.send`, `app/events/event_dispatcher.py:23-25`) — not exercised because the default configuration leaves `EVENT_WEBHOOK` unset (§6, finding M12).
- The `validate_email` → `EmailNotValidError` → `flash(str(e), "error")` branch (`app/dashboard/views/custom_alias.py:109-114`) — requires a prefix+suffix pair that passes `verify_prefix_suffix` yet fails RFC email validation, which the canonical signed suffixes do not produce.
- The **SL-public-domain** (global `DeletedAlias`) recreation branch — the symmetric counterpart of the **observed** custom-domain `DomainDeletedAlias` recreation (§7 B5); the public-domain variant routes through `Alias.create`'s `DeletedAlias` lookup rather than the domain-scoped `domain_deleted_alias` table.

(Three items that earlier drafts listed as INFERRED are now **OBSERVED** and have moved to §9.2: the `verify_prefix_suffix`→`False` "hacked" branch returning **HTTP 200** — `app/dashboard/views/custom_alias.py:162-166`, captured in §7 B5; the exact **API quota 400 JSON body** with the real value `5`, captured in §8.3; and the **API 401 `{"error": "Wrong api key"}`** path for an invalid or missing `Authentication` key — the §8.4 prose earlier described only what the code "would" return, but this path is now **exercised directly** and captured verbatim (§8.4).)

### 9.4 NON-CANONICAL (non-default / crafted inputs, explicitly labelled)

- **Adversarial direct `POST`s that bypass the browser** (finding S3): several §7 error branches (B2 CSRF, parts of B5 custom-alias validation, and the B6 race) were driven by scripted HTTP clients that skip the dashboard's client-side JavaScript guard (`templates/dashboard/custom_alias.html:112-127`). §7 reports the **browser behavior first** and labels each such direct post NON-CANONICAL; they exercise server-side validation that a normal browser session would not reach the same way.
- **Non-numeric `generator_scheme`** (§7 B3′): a crafted `generator_scheme=abc` produces an uncaught `ValueError` → **HTTP 500** at `app/dashboard/views/index.py:99` (`int(...)`). The dashboard dropdown only ever submits `1` (word) or `2` (uuid) — and the plain "Random Alias" button submits no `generator_scheme` at all (the server then applies the user's default) — so a non-numeric value is crafted and non-canonical.
- The **Redis-backed token-bucket demos** in §7 B4:
  - **CASE A** — a live `RedisStorage("redis://localhost:6379")` with `max_hits=5` producing `TooManyRequests` on call #6 (`… -> 6/5`).
  - **CASE B** — a dead Redis on `:6399` producing `ERROR app/rate_limiter.py:42 - Cannot connect to redis` and then **failing open** (returning normally with no exception).

  Both are non-canonical because the default configuration has `MEM_STORE_URI=None`, which makes the bucket a no-op (`app/rate_limiter.py:28-29`). They are included only to demonstrate the real behavior of the `check_bucket_limit` code when a store is present.

### 9.5 Final named-item coverage matrix

Every part of the six questions, and every mechanism / function / file / flag / condition they name, mapped to where it is answered and its primary repository-relative `file:line`.

**9.5.1 — Question sub-parts**

| Question sub-part | Answered in | Primary `file:line` |
|-------------------|-------------|---------------------|
| Q1 — bring up the app (standard dev setup) | §1 | `server.py:588`; `CONTRIBUTING.md:106` |
| Q1 — log in the test user `john@wick.com` | §2 | `app/auth/views/login.py:21-25`; creds per `CONTRIBUTING.md:109` |
| Q1 — create a new alias & watch the flow | §2 | `app/dashboard/views/index.py:55` |
| Q2 — HTTP method (`POST`) | §3 | `app/dashboard/views/index.py:55` |
| Q2 — URL / path (`/dashboard/`, `/dashboard/custom_alias`) | §3 | `app/dashboard/views/index.py:55`; `app/dashboard/views/custom_alias.py:30` |
| Q2 — content type (`application/x-www-form-urlencoded`) | §3 | — (observed request header) |
| Q2 — body fields, random (`form-name`, `csrf_token`, `generator_scheme`) | §3 | `templates/dashboard/index.html:51,69,72,79,82` |
| Q2 — body fields, custom (`prefix`, `signed-alias-suffix`, `mailboxes`, `note`, `csrf_token`) | §3 | `templates/dashboard/custom_alias.html:94,112-127` |
| Q2 — CSRF token handling (+ client-side JS, M8) | §3 | `app/utils.py:157`; `app/dashboard/views/index.py:85,88`; `templates/dashboard/custom_alias.html:112-127` |
| Q3 — status code(s) (302 success; 200 re-render; 400/429/500 error) | §4, §7, §8 | `app/dashboard/views/index.py:113`; `app/dashboard/views/custom_alias.py:161,166` |
| Q3 — payload (redirect body / JSON) | §4, §8 | — (observed responses) |
| Q3 — metadata: headers | §4 | — (observed responses) |
| Q3 — metadata: redirect `Location` (`highlight_alias_id`) | §4 | `app/dashboard/views/index.py:113-121`; `app/dashboard/views/custom_alias.py:161` |
| Q3 — metadata: flash messaging (toastr) | §4 | `templates/base.html:98,102,185` |
| Q4 — which records inserted/updated | §5 | `app/models.py:1627-1692` |
| Q4 — how many tables (3 random / 4 custom+secondary mbx) | §5.5 | — (observed SQL) |
| Q4 — related entities (`daily_metric`, `alias_audit_log`, `alias_mailbox`) | §5 | `app/models.py:3262,3810,2939`; `app/alias_audit_log_utils.py:18-32` |
| Q5 — background tasks (none relevant run) | §6 | `ps` evidence (§6) |
| Q5 — follow-up events (gated no-op) | §6 | `app/events/event_dispatcher.py:49-79` |
| Q5 — additional work (synchronous audit log) | §6 | `app/alias_audit_log_utils.py:18-32` |
| Q6 — validation problems (+ response/logs/DB state) | §7 B5 | `app/dashboard/views/custom_alias.py:64,86,100,103-104` |
| Q6 — database errors (IntegrityError / rollback) | §7 B6 | `app/dashboard/views/custom_alias.py:146-150`; `app/db.py:12-14` |
| Q6 — network/service issues (Redis down) | §7 B4 | `app/rate_limiter.py:28-29,41-42` |

**9.5.2 — Named functions / methods**

| Function / method | Documented in | `file:line` |
|-------------------|---------------|-------------|
| `Alias.create` | §5, §8.2 | `app/models.py:1627-1692` |
| `Alias.create_new_random` | §2, §5 | `app/models.py:1721` |
| `Alias.create_new` | §5 (custom) | `app/models.py:1695` |
| `ModelMixin.create` | §5 | `app/models.py:115` |
| `User.can_create_new_alias` | §7 B1 | `app/models.py:867-884` |
| `DailyMetric.get_or_create_today_metric` | §5 | `app/models.py:3280` |
| `emit_alias_audit_log` / `AliasAuditLogAction.CreateAlias` | §5, §6 | `app/alias_audit_log_utils.py:18-32` |
| `EventDispatcher.send_event` | §6 | `app/events/event_dispatcher.py:49-79` |
| `check_bucket_limit` | §7 B4 | `app/rate_limiter.py:19-42` |
| `CSRFValidationForm` | §3, §7 B2 | `app/utils.py:157` |
| `authorize_request` (API auth) | §8.2 | `app/api/base.py:16-32` |
| `serialize_alias_info_v2` (API) | §8.1 | `app/api/serializer.py:55-93` |

**9.5.3 — Named configuration flags**

| Flag / key (default) | Documented in | `file:line` |
|----------------------|---------------|-------------|
| `ALIAS_LIMIT` (`100/day;50/hour;5/minute`) | §7 B3 | `app/config.py:448` |
| `ALIAS_CREATE_RATE_LIMIT_FREE` / `_PAID` | §7 B4 | `app/config.py:554-558` |
| `EVENT_WEBHOOK` (unset) / `EVENT_WEBHOOK_DISABLE` | §6 | `app/config.py:612`; `app/events/event_dispatcher.py:57-64` |
| `MEM_STORE_URI` (unset → bucket & Redlock no-op) | §7 B4 | `server.py:163-165`; `app/rate_limiter.py:28-29` |
| `DISABLE_RATE_LIMIT` (`False`) | §7 B3/B4 | `app/config.py:602`; `app/extensions.py:23,28` |
| `MAX_NB_EMAIL_FREE_PLAN` (`5`) | §7 B1, §8.3 | `app/config.py:121-124` |

**9.5.4 — Conditions / variants exercised**

| Condition / variant | Section | Observed result |
|---------------------|---------|-----------------|
| Random alias — word scheme (`generator_scheme=1`) | §3 | 302, created |
| Random alias — uuid scheme (`=2`) | §3 | 302, created |
| Random alias — default (no `generator_scheme`) | §3 | 302, created |
| Custom alias — single mailbox | §3, §5 | 302, 3-table write |
| Custom alias — secondary mailbox | §5 | 302, 4-table (adds `alias_mailbox`) |
| API random alias (happy path) | §8.1 | 201 JSON, `note` echoed |
| Quota exceeded — web random & custom | §7 B1 | 302 redirect + flash |
| Quota exceeded — API | §8.3 | 400 JSON (real `5`) |
| CSRF wrong / missing — random & custom (cross-product) | §7 B2 | 302 "Invalid request" |
| Route 429 — web random & custom | §7 B3 | 429 HTML (`error/429.html`) |
| Route 429 — API | §8.3 | 429 JSON |
| API — invalid / missing `Authentication` key | §8.4 | **401 JSON** `{"error":"Wrong api key"}` (OBSERVED) |
| Redis token bucket + Redlock — default | §7 B4 | no-op (canonical) |
| Redis bucket — live / dead store (crafted) | §7 B4, §9.4 | enforce / fail-open (NON-CANONICAL) |
| Custom prefix — bad-charset (`bad!x`) via browser | §7 B5 | Parsley `pattern` blocks client-side, **no POST** (OBSERVED) |
| Custom prefix — bad / empty / long / consecutive-dots (crafted direct POST) | §7 B5 | 302 error (NON-CANONICAL — bypasses the browser gate) |
| Custom prefix — missing field | §7 B5 | 500 (`NoneType.strip`) |
| Custom suffix — tampered / empty / expired | §7 B5 | 302 "…expired" warning |
| Custom suffix — missing field | §7 B5 | 302 "Unknown error" |
| Custom mailbox — missing / tampered | §7 B5 | 302 error / warning |
| `verify_prefix_suffix=False` ("hacked") | §7 B5 | 200 re-render |
| Duplicate own alias | §7 B5 | 200 re-render, "already have" |
| Deleted-alias recreation (custom domain) | §7 B5 | 200, "restore on Deleted Alias page" |
| Non-numeric `generator_scheme` (crafted) | §7 B3′, §9.4 | 500 (NON-CANONICAL) |
| IntegrityError concurrent race (real route) | §7 B6 | one 302 + shared-connection 500 cascade |
| Web vs. API contrast | §8 | HTTP envelope + write-set differ |

### 9.6 Review-findings closure matrix (this checkpoint)

**Auditable-report availability (M17):** one review report is present on disk in this workspace — `/tmp/blitzy/qa/reports/50c513ea-a8be-4059-9e27-0c097dfd57f3/blitzy_documentation_app_2cd6ee777f8c.md/cr/review/dest_1783966928953_b4afbc9e.md`. Its 22 numbered findings (2 Critical, 17 Major, 3 Minor) plus the 4 security items (S1–S4) are closed below with the section that carries the resolving evidence. Earlier-checkpoint reports were not present in this workspace to audit; this matrix is the auditable closure evidence for the report that exists.

| ID | Severity | Category | Status | Resolution evidence |
|----|----------|----------|--------|---------------------|
| C1 | CRITICAL | Runtime/Infra (wrong env) | ✅ RESOLVED | §1 re-run in canonical image `ea242796` (Py 3.10.18 / PG 15.13 / Redis 7.0.15); §9.1 provenance; whole-doc wrong-env markers = 0 |
| C2 / S1 | CRITICAL / Security | Repo integrity (cleanup) | ✅ RESOLVED | Appendix A — repository final-state proof (`git status --porcelain` / `--ignored`, `.env` & `__pycache__`/`.pyc` removed, PG logging reset) |
| M1 | MAJOR | Evidence provenance | ✅ RESOLVED | §9.1 explicit fresh-canonical-run disclosure |
| M2 | MAJOR | Evidence (cmd + output) | ✅ RESOLVED | every behavioral claim carries its exact producing command **and** its complete, unedited output beside it (§2–§8); §8.2's DB-log slice now uses a real, verified line range (`sed -n '12026,12119p'`), replacing the earlier non-executable placeholder command |
| M3 | MAJOR | Evidence (elisions) | ✅ RESOLVED | verbatim blocks are complete with **zero `…` elisions** — the full random/custom SQL slices (§5.2/§5.3), the complete 6067-byte web-429 body (§7 B3), the full `IntegrityError` tracebacks (§7 B6), and the untruncated API JSON (§8); the only angle-bracket tokens remaining are **deliberate secret redactions** (`<APIKEY>`, `<FREE_APIKEY>`, `<REDACTED>`) masking credentials in command/header lines — not incomplete-work placeholders |
| M4 | MAJOR | Perf (before/after, 2-run) | ✅ RESOLVED | §5 before/after row values + two-run stability |
| M5 | MAJOR | Infra (bootstrap) | ✅ RESOLVED | §1 full sequence with exit codes (image, PG/Redis, role/DB, alembic, dummy-data, socat, health/login) |
| M6 | MAJOR | API / DB (write-set) | ✅ RESOLVED | §8.2 `api_key` UPDATE/commit exhibit; §8 contrast table (4 vs 3 tables) |
| M7 | MAJOR | API (note / quota) | ✅ RESOLVED | §8.1 full 201 with `note` echoed; §8.3 quota 400 with real `5` |
| M8 | MAJOR | Frontend (JS / CSRF cite) | ✅ RESOLVED | §3 custom-form JS (`custom_alias.html:112-127`); random `csrf_token` cited at `index.html:51,69,79`; adversarial posts labelled §7 / §9.4 |
| M9 | MAJOR | HTTP (302-only claim) | ✅ RESOLVED | §4 PRG scoped to **success**; §7 documents 200 re-renders |
| M10 | MAJOR | Backend (maxed-free GET) | ✅ RESOLVED | §7 B1 — maxed-free GET → 302 redirect (`app/dashboard/views/custom_alias.py:36-42`), not "0 suffix options" |
| M11 | MAJOR | DB error (race mislabel) | ✅ RESOLVED | §7 B6 — real route race + deterministic component repro, each labelled |
| M12 | MAJOR | Events (gating) | ✅ RESOLVED | §6 three-gate sequence (webhook **and** partner), `event_dispatcher.py:57-69` |
| M13 | MAJOR | Error paths (missing) | ✅ RESOLVED | §7 B2 CSRF cross-products, B3′ non-numeric scheme, custom-route 429, browser-vs-crafted |
| M14 | MAJOR | Observability | ✅ RESOLVED | §6 exact count/isolation commands, `ps`, `sync_event=0`/`NOTIFY=0`, "no relevant SL worker" |
| M15 / S2 | MAJOR / Security | Credential handling | ✅ RESOLVED | §8.1 / §8.4 — key redacted, disposable-local, deletion + revocation proof |
| M16 | MAJOR | Docs / citations | ✅ RESOLVED | §9.5 named-item coverage matrix; INFERRED labels (§9.3); repo-relative `file:line` throughout |
| M17 | MAJOR | Process (closure) | ✅ RESOLVED | §9.6 this matrix + report-availability note |
| m1 | MINOR | Exactness (signer) | ✅ RESOLVED | §3 signer described as original value + timestamp + signature parsed from the right |
| m2 | MINOR | UI (toastr) | ✅ RESOLVED | §4 `toastr.error` (category) vs. Bootstrap colour distinction (`base.html:102,185`) |
| m3 | MINOR | Perf clarity | ✅ RESOLVED | §6 inter-log span vs. total request duration reconciled |
| S3 | Security | Adversarial labelling | ✅ RESOLVED | §7 browser-behavior-first + NON-CANONICAL labels; §9.4 |
| S4 | Security | Race classification | ✅ RESOLVED | §7 B6 — real route race distinguished from the component-level repro |

---

## Appendix A — Repository Final State (Cleanup Proof) (OBSERVED)

This section closes findings **C2 / S1** (repository integrity). The investigation is read-only: the sole intended change to the repository is the one new deliverable, `blitzy/documentation/app_2cd6ee777f8c.md`. Below is the captured proof that (a) the single non-default runtime setting enabled for observation was restored, (b) every temporary observation process, script, and artifact was removed, and (c) the only **tracked** change is this one file — `git diff <base> --name-status` reports a single added path — while the untracked `blitzy/screenshots/` and `blitzy/screen_recordings/` observation captures retained for the reader, and the gitignored dependency/runtime directories, are never committed (§A.4). Every block shows the producing command and its complete, unedited output (M2/M3).

> **Read-only invariant (OBSERVED):** `git diff <base> --name-status` reports exactly one path — this deliverable (an added file) — so no source, template, migration, test, configuration, or dependency file is added, modified, or deleted. `git status --porcelain` shows this file plus two **untracked** observation-artifact directories (`blitzy/screenshots/`, `blitzy/screen_recordings/`) that are retained for the reader and never `git add`-ed; they never enter a commit and never affect the one-path tracked delta (§A.4).

### A.1 Temporary runtime setting restored — PostgreSQL statement logging

The only non-default runtime setting enabled during the investigation was per-database SQL statement logging on `simplelogin` (used in §5 to capture the exact `INSERT`/`UPDATE` statements). It is reset to the built-in cluster defaults:

```
$ docker exec -e PGPASSWORD=mypassword sl-app psql -h localhost -U myuser -d simplelogin \
    -c "ALTER DATABASE simplelogin RESET log_statement;" \
    -c "ALTER DATABASE simplelogin RESET log_min_duration_statement;" \
    -c "SELECT pg_reload_conf();"
ALTER DATABASE
ALTER DATABASE
 pg_reload_conf
----------------
 t
(1 row)
```

Verified on a fresh connection — both settings are back to the defaults (`log_statement=none`, `log_min_duration_statement=-1`) and no per-database override row remains:

```
$ docker exec -e PGPASSWORD=mypassword sl-app psql -h localhost -U myuser -d simplelogin -tAc \
    "SELECT name||'='||setting FROM pg_settings WHERE name IN ('log_statement','log_min_duration_statement') ORDER BY name;"
log_min_duration_statement=-1
log_statement=none
$ docker exec -e PGPASSWORD=mypassword sl-app psql -h localhost -U myuser -d simplelogin -tAc \
    "SELECT count(*) FROM pg_db_role_setting s JOIN pg_database d ON d.oid=s.setdatabase WHERE d.datname='simplelogin';"
0
```

### A.2 Observation processes stopped

The dev server (`server.py`) and the `socat` host bridge started for observation are stopped by killing their specific container PIDs (never a host-wide `pkill`):

```
$ docker exec sl-app kill 5943 3621 3615
$ docker exec sl-app ps -eo pid,ppid,stat,cmd | grep -E 'server\.py|socat' | grep -v grep
   3621       1 Z    [socat] <defunct>
$ curl -sS -m5 -o /dev/null -w 'http_code=%{http_code}\n' http://localhost:7777/
curl: (56) Recv failure: Connection reset by peer
http_code=000
```

`server.py` is fully gone and the `socat` listener is closed — port 7777 is no longer served (`curl` receives a connection reset, `http_code=000`). The residual `socat` entry is a harmless zombie (`Z <defunct>`) reparented to the container's PID 1 (`sleep infinity`); it holds no port, socket, or memory and is container-local, not a repository artifact. *(INFERRED: PID 1 `sleep` does not `wait()`-reap children, so the process-table entry lingers until the container stops; it has no effect on the repository or the deliverable.)*

### A.3 Temporary observation artifacts removed

**Host scratch** — observation scripts and captured evidence written outside the repository under `/tmp` (the shared workspace root `/tmp/blitzy` is never touched):

```
$ rm -f /tmp/obs_api_*.py && rm -rf /tmp/blitzy_evidence
$ ls -d /tmp/blitzy_evidence 2>&1; ls /tmp/obs_api_*.py 2>&1
ls: cannot access '/tmp/blitzy_evidence': No such file or directory
ls: cannot access '/tmp/obs_api_*.py': No such file or directory
```

**Container scratch** — scripts `docker cp`-ed in plus the capture directory and logs, all under the container's ephemeral `/tmp` (which is *not* the `/app` bind mount):

```
$ docker exec sl-app rm -rf /tmp/blitzy_evidence /tmp/blitzy_server.log /tmp/blitzy_socat.log
$ docker exec sl-app rm -f /tmp/obs_*.py /tmp/obs_*.sh /tmp/obs_*.txt /tmp/api_body_*.txt \
      /tmp/create_alias.py /tmp/diag.py /tmp/login_test.py /tmp/lt.py
$ docker exec sl-app bash -c "ls /tmp/obs_*.py /tmp/api_body_*.txt /tmp/create_alias.py 2>&1 | tail -3"
ls: cannot access '/tmp/obs_*.py': No such file or directory
ls: cannot access '/tmp/api_body_*.txt': No such file or directory
ls: cannot access '/tmp/create_alias.py': No such file or directory
```

**Repository working-tree artifacts** flagged by the finding (`.env`, all `__pycache__` directories, all `*.pyc`) plus the runtime `static/upload` directory created by `flask dummy-data`, removed from the clone (excluding the `venv/` mount and `.git/`) — with before/after counts:

```
$ echo "BEFORE: pyc=$(find . -type f -name '*.pyc' -not -path './venv/*' -not -path './.git/*' | wc -l)" \
       "pycache=$(find . -type d -name '__pycache__' -not -path './venv/*' -not -path './.git/*' | wc -l)" \
       "env=$(test -f .env && echo present || echo absent)" \
       "upload=$(test -d static/upload && echo present || echo absent)"
BEFORE: pyc=583 pycache=46 env=present upload=present
$ find . -type f -name '*.pyc'      -not -path './venv/*' -not -path './.git/*' -delete
$ find . -type d -name '__pycache__' -not -path './venv/*' -not -path './.git/*' -exec rm -rf {} +
$ rm -f .env && rm -rf static/upload
$ echo "AFTER: pyc=$(find . -type f -name '*.pyc' -not -path './venv/*' -not -path './.git/*' | wc -l)" \
       "pycache=$(find . -type d -name '__pycache__' -not -path './venv/*' -not -path './.git/*' | wc -l)" \
       "env=$(test -f .env && echo present || echo absent)" \
       "upload=$(test -d static/upload && echo present || echo absent)"
AFTER: pyc=0 pycache=0 env=absent upload=absent
```

The shared Python virtualenv is intentionally **retained**: it is the image-provided runtime at `/opt/sl-venv`, mounted at the container's `/app/venv`, is gitignored via the `venv/` rule, and never appears in `git status`. It is confirmed untouched and functional (its own byte-compiled cache was excluded from the deletions above):

```
$ docker exec sl-app bash -c "find /app/venv -type f -name '*.pyc' | wc -l"
2711
$ docker exec sl-app /app/venv/bin/python -c "import flask; print('flask', flask.__version__)"
flask 1.1.2
```

### A.4 Final working-tree proof

```
$ git rev-parse HEAD
ff16e389c15a0820924a4dec782842e5fa7fbb02
$ git status --porcelain
 M blitzy/documentation/app_2cd6ee777f8c.md
?? blitzy/screen_recordings/
?? blitzy/screenshots/
$ git status --ignored --porcelain
 M blitzy/documentation/app_2cd6ee777f8c.md
?? blitzy/screen_recordings/
?? blitzy/screenshots/
!! static/node_modules/
$ git diff 2cd6ee777f8c2d3531559588bcfb18627ffb5d2c --name-status
A	blitzy/documentation/app_2cd6ee777f8c.md
```

Interpretation:

- `git diff <base> --name-status` against the immutable pre-existing base commit `2cd6ee77` (before this file existed) reports a single **`A`** (added) path — this deliverable — with **no source path modified or deleted**. This is the timeless read-only invariant, and it holds independently of any untracked or gitignored local file (`2cd6ee77` verified an ancestor of `HEAD` via `git merge-base --is-ancestor`).
- `git status --porcelain` lists this deliverable (` M` = modified relative to the current `HEAD` `ff16e389`, which already carries an earlier draft of the same file; the session's edits are committed as the final step, advancing `HEAD` by one — see the capture-timing note) **plus two untracked observation-artifact directories**, `blitzy/screen_recordings/` and `blitzy/screenshots/`. These hold the runtime and visual-verification captures referenced throughout this document and the QA review (for example the Parsley-blocked custom-prefix screenshot cited in §7 B5, `blitzy/screenshots/docfix_doc2_custom_bad_prefix_parsley_blocked.png`); they are produced during the investigation, retained for the reader as evidence, and **deliberately never `git add`-ed**, so they never enter a commit.
- `git status --ignored --porcelain` additionally lists `static/node_modules/` — the gitignored npm dependency directory (`.gitignore:9`), retained like the `venv/` runtime mount (`.gitignore:12`, which git does not enumerate). Every temporary scratch and build artifact flagged by the finding (`.env` `.gitignore:4`, `__pycache__/`, `*.pyc` `.gitignore:2`, `static/upload/` `.gitignore:11`) was removed in §A.3 and is confirmed **absent**; the only artifacts remaining are the intended deliverable, the reader-facing observation captures, and the gitignored dependencies — none of which is ever committed.

> **Capture timing (honest disclosure):** because this is a **self-documenting file**, the `HEAD` hash shown above (`ff16e389`) is a point-in-time value: the working-branch `HEAD` advances by one with **each doc-only review commit** (`87284998` → `1c10c966` → `b0b4dd6e` → `96ccbaa5` → `a4467a8a` → `ff16e389` → the commit that finalizes these corrections), so after the final commit a live `git rev-parse HEAD` on the delivered branch reports a **descendant** of `ff16e389`. The timeless read-only invariant does **not** depend on this moving hash — `git diff 2cd6ee777f8c2d3531559588bcfb18627ffb5d2c --name-status` reports the single added deliverable at **every** one of those HEADs, because it is measured against the immutable base commit `2cd6ee77` (verified an ancestor of `HEAD` via `git merge-base --is-ancestor`). Likewise, the untracked `blitzy/screenshots/` and `blitzy/screen_recordings/` capture directories and the gitignored `static/node_modules/`, the regenerable `.env`, and the `venv/` mount are **never tracked**, so they never alter that one-file tracked delta. Appending this Appendix A to the deliverable is the only change made *after* the git state was captured; it adds lines to the single, already-tracked file and creates no new path — re-verified immediately before the final commit. Database rows created while exercising the flow live only in the PostgreSQL data directory (inside the container, outside the `/app` bind mount) and are never part of the repository working tree.



---

*End of investigation. The default-configuration behavior (3-table write, gated event no-op, 302 Post/Redirect/Get + flash) is the canonical answer; configuration-dependent behavior (a non-default setup with **both** a configured `EVENT_WEBHOOK` **and** a `PartnerUser` adds a `sync_event` INSERT + `NOTIFY`) and non-default component demonstrations are labelled INFERRED / NON-CANONICAL above.*
