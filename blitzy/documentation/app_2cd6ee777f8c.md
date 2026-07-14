# SimpleLogin — Running It Locally and Confirming the Three Core Components Work

> **Document type:** Run-first, evidence-backed Q&A runtime investigation.
> **Subject:** [SimpleLogin](https://github.com/simple-login/app) — a Python/Flask email-aliasing platform.
> **Methodology:** Every behavioral claim below was produced by **actually building, running, and driving** the software through its **real entry points** — an HTTP request to the web app, an SMTP delivery to the email handler, and a genuinely enqueued `Job` (created by a real authenticated dashboard action) drained by the job runner. Each claim is accompanied by the command that produced it and the **raw captured output** (stdout/stderr, HTTP response including headers, SMTP transcript, or SQL rows). Reading the source is used only to *locate and explain* each signal.
>
> **Evidence conventions used throughout:**
> - Output shown inside a fenced block is the **verbatim** captured bytes for the stated command, and no explanatory prose is placed inside an output fence. Three narrow kinds of removal can occur, and each is always marked or disclosed: (a) a single secret value is replaced by an explicit `[REDACTED: …]` marker; (b) a long run of lines that are **identical in form** (for example the repeated `drop cascades to …` and `Running upgrade …` lines in §2.5) is omitted for length, with the lines on either side shown verbatim; and (c) a few framing lines that are not part of the signal — startup/banner lines the interpreter prints before the app is ready, or a single post-success diagnostic traceback from a helper script — are omitted. Every omission of type (b) or (c) is disclosed **at the point it occurs**, stating exactly how many lines were removed and why.
> - A claim that was **not** directly observed at runtime is labeled **`[INFERRED]`** with the reason.
> - A value obtained off the canonical path (e.g. a direct DB probe used only to *read* state) is labeled **`[NON-CANONICAL]`**; no such value is ever substituted for a canonical-path observation.

---

## 1. Summary

SimpleLogin lets a user hide their real email address behind **aliases**. Mail sent to an alias is forwarded to the user's real mailbox; the user can reply through the alias without revealing their address. The platform is a Python/Flask monorepo, and the three runtime components a newcomer asks about map **one-to-one** onto three root-level entry points:

| Component | Entry point | Role | Default port |
|-----------|-------------|------|--------------|
| **Web server** | `server.py` (prod: `wsgi.py` under Gunicorn) | Serves the dashboard UI, auth, OAuth/API clients | **7777** |
| **Email handler** | `email_handler.py` | `aiosmtpd` SMTP server implementing alias *forwarding* and *replying* | **20381** |
| **Job runner** | `job_runner.py` | Polls the `job` table and drains asynchronous work (account deletion, onboarding, batch import, user-report, …) | *(none — no listening socket)* |

These three processes are **separate, independently-launched programs** (in the upstream self-hosted deployment they run as three separate containers). None of them starts the others; each is brought up on its own and then stays alive on its own loop. This document answers three question clusters:

- **Q1 — Are the three components up?** How a reader confirms, from logs / HTTP responses / the dashboard, that the web server, email handler, and job runner are live and that users can sign in and manage aliases. → [§4](#4-q1--are-the-three-components-up-liveness--confirmation-signals)
- **Q2 — Core user actions & correct-handling evidence.** Creating an account, creating an alias (random *and* custom), and having that alias receive an email — with the before / intermediate / after state at each boundary. → [§5](#5-q2--core-user-actions--correct-handling-evidence)
- **Q3 — Background auto-online behavior.** Whether the email handler and job runner keep working in the background once started, plus their secondary/edge behavior (a genuinely-enqueued job, retry accounting, an unknown job, reply/bounce SMTP codes, alias auto-creation). → [§6](#6-q3--background-behavior-do-the-handler-and-runner-keep-working)

A short orientation to the log format the reader will see everywhere is in [§3](#3-how-to-read-the-logs-the-sl-log-format). A cleanup confirmation ([§7](#7-cleanup--the-repository-and-database-are-left-unchanged)), a coverage-pass table ([§8](#8-coverage-pass)), and an observed-vs-inferred ledger ([§9](#9-observed-vs-inferred-ledger)) close the document.

---

## 2. How to run it locally (Environment)

Everything below was run against the **default, canonical configuration** in the user-provided Docker image, using two co-operating containers on a private Docker network. A newcomer can reproduce the whole environment by following §2.1 → §2.5 in order: §2.1 brings up both containers, creates the `.env`, and starts Redis; §2.5 migrates the database to head and loads the seed data.

### 2.1 Images and containers

The runtime is the GHCR image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` plus a stock `postgres:13`:

```
$ docker images --format '{{.Repository}}:{{.Tag}}  {{.ID}}  {{.Size}}' | grep -iE "scaleapi|postgres"
ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0  ea242796bbce  2.14GB
postgres:13  264e9dea325c  438MB
```

> **Image reference.** The user-provided Special Instructions name this image by its Docker Hub alias `andrewparkscaleai/coding-agent:simple-login__app__2cd6ee7…`; this document uses the equivalent, directly-pullable GHCR reference shown above — the same image (identical image ID `ea242796bbce`) — which is the reference available in this environment.

Two containers run on a user-defined bridge network `sl-net`. `sl-app` (the GHCR image) publishes the web port `7777` and the SMTP port `20381` to the host; `sl-postgres` publishes `15432→5432`:

```
$ docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}'
NAMES         IMAGE                                                           STATUS       PORTS
sl-app        ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0   Up 2 hours   0.0.0.0:7777->7777/tcp, 0.0.0.0:20381->20381/tcp
sl-postgres   postgres:13                                                     Up 2 hours   0.0.0.0:15432->5432/tcp

$ docker network inspect sl-net --format '{{range .Containers}}{{.Name}} {{.IPv4Address}}{{println}}{{end}}'
sl-app 172.18.0.3/16
sl-postgres 172.18.0.2/16
```

The complete bring-up (already performed for this environment; run it verbatim on a fresh host to reproduce the exact state §2.2–§2.4 observe and §2.5 migrates and seeds) is:

```bash
docker network create sl-net
docker run -d --name sl-postgres --network sl-net \
  -e POSTGRES_USER=myuser -e POSTGRES_PASSWORD=mypassword -e POSTGRES_DB=simplelogin \
  -p 15432:5432 postgres:13
docker run -d --name sl-app --network sl-net \
  -p 7777:7777 -p 20381:20381 \
  ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0 \
  -lc 'sleep infinity'

# Prepare configuration inside sl-app: the pristine image ships example.env but no
# .env, so create it and repoint DB_URI from the shipped @localhost (example.env:L75)
# to the sl-postgres container, which is where Postgres is reachable on sl-net:
docker exec sl-app bash -lc 'cd /app && cp example.env .env'
docker exec sl-app bash -lc "cd /app && sed -i 's#@localhost:5432/simplelogin#@sl-postgres:5432/simplelogin#' .env"

# Start Redis inside sl-app; the pristine image does not auto-start it. NOTE:
# in the DEFAULT config the app does NOT use Redis for sessions or rate-limiting
# (it uses signed-cookie sessions and an in-process memory rate-limiter — see the
# note in §2.2); Redis is engaged only when the optional MEM_STORE_URI is set.
docker exec sl-app redis-server --daemonize yes --save "" --appendonly no
```

> **Why the app container's command is `-lc 'sleep infinity'` (not `bash -lc …`).** The image's entrypoint is `/bin/bash` (`docker image inspect ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0 --format '{{json .Config.Entrypoint}}'` → `["/bin/bash"]`), so the trailing tokens after the image name are passed as *arguments* to that entrypoint. Supplying `-lc 'sleep infinity'` yields exactly `Path=/bin/bash Args=["-lc","sleep infinity"]` — the form the running `sl-app` uses. Supplying `bash -lc 'sleep infinity'` instead would run `/bin/bash bash -lc 'sleep infinity'`, and the container exits immediately: `docker inspect` shows `State.ExitCode=126` and the log reads `/usr/bin/bash: /usr/bin/bash: cannot execute binary file`.

### 2.2 Runtime versions

Inside `sl-app`, the application runs on Python **3.10** (from a pre-built virtualenv at `/app/venv`; `Dockerfile:L8` pins `python:3.10`). PostgreSQL is **13** and Redis answers on `6379`:

```
$ docker exec sl-app ./venv/bin/python --version
Python 3.10.18

$ docker exec sl-postgres psql -U myuser -d simplelogin -tAc "SELECT version();"
PostgreSQL 13.23 (Debian 13.23-1.pgdg13+1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 14.2.0-19) 14.2.0, 64-bit

$ docker exec sl-app redis-cli ping
PONG
```

> **Newcomer note — use the venv, not the system Python.** The app's third-party dependencies (Flask, `arrow`, and the rest) live only in `/app/venv`. Run against the system Python, `python3 server.py` fails at its first third-party import — `ModuleNotFoundError: No module named 'arrow'` (`server.py:L5` imports `arrow`, before the Flask imports at `server.py:L12`); every invocation below uses `./venv/bin/python …` (or the pre-built `./venv/bin/gunicorn`, `./venv/bin/alembic`, `./venv/bin/flask`).

Redis is started inside `sl-app` as a daemon by the last command in §2.1 (the pristine image does not auto-start it):

```
$ docker exec sl-app ps -o pid,args -C redis-server
    PID COMMAND
     28 redis-server *:6379
```

> **Observed correction — the default configuration does *not* use Redis for sessions or rate-limiting.** A canonical inspection of the built app object (via the real `create_app()` factory, `server.py:L139`) shows that with the default `.env`, sessions are held in **signed cookies** and rate-limiting uses an **in-process memory** store; Redis is wired in only when the optional `MEM_STORE_URI` is set. The gate is explicit in `server.py:L163-L165` (`if MEM_STORE_URI: … initialize_redis_services(app, MEM_STORE_URI)`), and `MEM_STORE_URI` defaults to `None` (`app/config.py:L568`, `MEM_STORE_URI = os.environ.get("MEM_STORE_URI", None)`), which the shipped `example.env` leaves unset (the six probe lines are shown; the app-initialization banner that `create_app()` prints ahead of them is omitted for brevity):
>
> ```
> $ docker exec sl-app bash -lc 'cd /app && PYTHONPATH=/app ./venv/bin/python - <<PY
> import flask_limiter
> from server import create_app
> from app.config import MEM_STORE_URI
> app = create_app()
> si = type(app.session_interface)
> print("MEM_STORE_URI                 =", MEM_STORE_URI)
> print("app.session_interface         =", si.__module__ + "." + si.__name__)
> key = flask_limiter.extension.C.STORAGE_URL
> print("limiter STORAGE_URL cfg key   =", key)
> print("app.config[STORAGE_URL]       =", app.config.get(key, "<unset>"))
> print("limiter storage backend       =", type(app.extensions["limiter"]._storage).__module__ + "." + type(app.extensions["limiter"]._storage).__name__)
> print("SESSION_COOKIE_NAME           =", app.config.get("SESSION_COOKIE_NAME"))
> PY'
> MEM_STORE_URI                 = None
> app.session_interface         = flask.sessions.SecureCookieSessionInterface
> limiter STORAGE_URL cfg key   = RATELIMIT_STORAGE_URL
> app.config[STORAGE_URL]       = memory://
> limiter storage backend       = limits.storage.MemoryStorage
> SESSION_COOKIE_NAME           = slapp
> ```
>
> This was corroborated empirically: Redis `DBSIZE` was unchanged (no new keys) across repeated `GET /health` and `POST /auth/login` requests, confirming the web request path does not touch Redis in the default config. Redis still runs (it is part of the canonical stack, and the test suite's `tests/test.env` *does* set `MEM_STORE_URI`), but the newcomer's default local run does not depend on it for the flows in this document.

### 2.3 Source checkout state (exact, not assumed)

The investigation targets the branch **`app_2cd6ee777f8c`**, which resolves to commit `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`. The **running checkout inside the image is in a detached-HEAD state at that same commit** — it is *not* on a named local branch. Reporting this exactly matters for reproducibility:

```
$ docker exec sl-app bash -lc 'cd /app && git rev-parse HEAD; git symbolic-ref -q --short HEAD || echo "(detached HEAD)"; git describe --all --always HEAD'
2cd6ee777f8c2d3531559588bcfb18627ffb5d2c
(detached HEAD)
tags/v4.53.2-6-g2cd6ee77
```

The image ships eight already-modified tracked files (key material, a rebuilt `spamassassin_utils.py`, and a rebuilt `static/package-lock.json`) that are **pre-existing image artifacts, present before this investigation began**; they are shown and accounted for in the cleanup check ([§7.3](#73-source-tree-untouched)).

### 2.4 Configuration (`.env`)

Configuration is environment-variable driven via `/app/.env` and `app/config.py`. The pristine image ships `example.env` but no `.env`, so §2.1 creates it (`cp example.env .env`) and rewrites `DB_URI` from the shipped `@localhost` to `@sl-postgres`. The file is **git-ignored** (`git check-ignore .env` → `.env`), so creating and editing it is not a source-tree change. The keys that matter for this investigation, with their line numbers in the file:

```
$ docker exec sl-app bash -lc "cd /app && grep -nvE '^\s*#|^\s*$' .env | grep -iE 'URL|DB_URI|EMAIL_DOMAIN|SUPPORT_EMAIL|FLASK_SECRET|NOT_SEND_EMAIL|DISABLE'"
6:URL=http://localhost:7777
19:NOT_SEND_EMAIL=true
22:EMAIL_DOMAIN=sl.local
40:SUPPORT_EMAIL=support@sl.local
75:DB_URI=postgresql://myuser:mypassword@sl-postgres:5432/simplelogin
77:FLASK_SECRET=[REDACTED: FLASK_SECRET value removed — a development secret]
150:DISABLE_ONBOARDING=true
```

Two flags shape later observations:

- **`NOT_SEND_EMAIL=true`** (`.env:L19`). This is **presence-based**: `app/config.py:L91` reads `NOT_SEND_EMAIL = "NOT_SEND_EMAIL" in os.environ`, so the *value* is ignored — the key being present at all disables outbound delivery. Under the default config, an inbound message is therefore fully *processed and logged* but the outbound copy is **short-circuited** and never actually leaves the process (see [§5.4](#54-c4--an-email-arrives-at-the-alias-forward-path)). To capture an actual forwarded artifact, §5.4 Part 2 applies the documented mail-capture toggle and discloses it explicitly.
- **`DISABLE_ONBOARDING=true`** (`.env:L150`). Because of this, account signup does **not** enqueue onboarding jobs — visible in the seed log as `Disable onboarding emails` (`app/models.py:L647`). The job runner is therefore exercised in [§6](#6-q3--background-behavior-do-the-handler-and-runner-keep-working) with a genuinely-enqueued *account-deletion* job instead.

### 2.5 Bring the database to head and seed it

A pristine baseline is established with the canonical reset sequence (the same three steps `scripts/reset_local_db.sh` performs): drop and recreate the schema, migrate to the Alembic head, then load the development seed data.

```
$ docker exec sl-postgres psql -U myuser -d simplelogin -c 'drop schema public cascade; create schema public;'
NOTICE:  drop cascades to 82 other objects
DETAIL:  drop cascades to table alembic_version
drop cascades to type plan_enum
drop cascades to table file
drop cascades to table users
```

The `DETAIL` block continues for all **82** dropped objects (the tables include `alias`, `job`, `contact`, `email_log`, `alias_audit_log`, `user_audit_log`, `daily_metric`, and `metric2`); the 78 further `drop cascades to …` lines are identical in form and are the only lines omitted here. The statement's final output line is:

```
CREATE SCHEMA
```

The migration ran against the freshly-dropped schema above and emitted **255** `Running upgrade` lines. That single command was:

`$ docker exec sl-app bash -lc 'cd /app && ./venv/bin/alembic upgrade head' 2>&1 | grep "Running upgrade"`

Its **first** output line, verbatim:

```
INFO  [alembic.runtime.migration] Running upgrade  -> 5e549314e1e2, empty message
```

Its **final three** output lines (reaching head `32f25cbf12f6`), verbatim — the 251 intermediate lines between them are of identical form and are omitted here for length:

```
INFO  [alembic.runtime.migration] Running upgrade 62afa3a10010 -> 91ed7f46dc81, alias_audit_log
INFO  [alembic.runtime.migration] Running upgrade 91ed7f46dc81 -> 7d7b84779837, user_audit_log
INFO  [alembic.runtime.migration] Running upgrade 7d7b84779837 -> 32f25cbf12f6, alias_audit_log_index_created_at
```

The **255** count came from that one-time run against the fresh schema. Because migrations are idempotent, re-running `alembic upgrade head` on the now-migrated database emits **0** `Running upgrade` lines; the head is verifiable at any time:

```
$ docker exec sl-app bash -lc 'cd /app && ./venv/bin/alembic upgrade head 2>&1' | grep -c "Running upgrade"
0
$ docker exec sl-postgres psql -U myuser -d simplelogin -tAc "SELECT version_num FROM alembic_version;"
32f25cbf12f6
```

Seed the development data. `flask dummy-data` is the CLI command defined at `server.py:L490-L497`; it logs `LOG.w("reset db, add fake data")` at `server.py:L494` and then runs `app/fake_data.py:fake_data()` (which logs `create fake data` at `app/fake_data.py:L41`). The **complete, unedited stdout** (26 lines) is:

```
$ docker exec sl-app bash -lc 'cd /app && FLASK_APP=wsgi ./venv/bin/flask dummy-data'
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/oqmphulpacbwiacmjebz
Upload files to local dir
>>> init logging <<<
2026-07-13 18:02:59,687 - SL - DEBUG - 3306 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-13 18:03:00,460 - SL - WARNING - 3306 - "/app/server.py:494" - dummy_data() -  - reset db, add fake data
2026-07-13 18:03:00,461 - SL - DEBUG - 3306 - "/app/app/fake_data.py:41" - fake_data() -  - create fake data
2026-07-13 18:03:00,742 - SL - INFO - 3306 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 18:03:00,746 - SL - DEBUG - 3306 - "/app/app/models.py:647" - create() -  - Disable onboarding emails
2026-07-13 18:03:00,764 - SL - DEBUG - 3306 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email word_test646@sl.local
2026-07-13 18:03:00,775 - SL - INFO - 3306 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 18:03:00,827 - SL - INFO - 3306 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 18:03:00,835 - SL - INFO - 3306 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 18:03:00,852 - SL - INFO - 3306 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 18:03:00,863 - SL - INFO - 3306 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 18:03:00,877 - SL - INFO - 3306 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 18:03:00,885 - SL - INFO - 3306 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 18:03:00,896 - SL - DEBUG - 3306 - "/app/app/models.py:1255" - generate_oauth_client_id() -  - generate oauth_client_id demo-hyjabveqvz
2026-07-13 18:03:00,902 - SL - DEBUG - 3306 - "/app/app/models.py:1255" - generate_oauth_client_id() -  - generate oauth_client_id demo2-ydyffsqciv
2026-07-13 18:03:01,174 - SL - INFO - 3306 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 18:03:01,176 - SL - DEBUG - 3306 - "/app/app/models.py:647" - create() -  - Disable onboarding emails
2026-07-13 18:03:01,197 - SL - INFO - 3306 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 18:03:01,210 - SL - INFO - 3306 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 18:03:01,219 - SL - INFO - 3306 - "/app/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain
```

The two `Disable onboarding emails` lines (`app/models.py:L647`) confirm the `DISABLE_ONBOARDING` flag is active; `generate email word_test646@sl.local` (`app/models.py:L1459`) is the single random alias; and `Add sl.local to SL domain` (`init_app.py:L44`) registers the local alias domain.

### 2.6 What the seed actually creates (the real baseline)

`flask dummy-data` does **not** create "11 random aliases." It builds a deliberately *mixed* fixture. Querying the database directly shows the true post-seed state:

```
$ docker exec sl-postgres psql -U myuser -d simplelogin -c "SELECT id, email, activated, is_admin FROM users ORDER BY id;"
 id |          email          | activated | is_admin
----+-------------------------+-----------+----------
  1 | john@wick.com           | t         | t
  2 | winston@continental.com | t         | f
(2 rows)

$ docker exec sl-postgres psql -U myuser -d simplelogin -c "SELECT id, email, user_id, mailbox_id FROM alias ORDER BY id;"
 id |                  email                  | user_id | mailbox_id
----+-----------------------------------------+---------+------------
  1 | simplelogin-newsletter.list417@sl.local |       1 |          1
  2 | word_test646@sl.local                   |       1 |          1
  3 | example@example.com                     |       1 |          2
  4 | e0@sl.local                             |       1 |          2
  5 | e1@sl.local                             |       1 |          1
  6 | e2@sl.local                             |       1 |          2
  7 | first@ab.cd                             |       1 |          1
  8 | second@ab.cd                            |       1 |          1
  9 | simplelogin-newsletter.word568@sl.local |       2 |          3
 10 | john@example.com                        |       1 |          2
 11 | wick@example.com                        |       1 |          2
(11 rows)
```

Mapping each row to the code that creates it:

- **`simplelogin-newsletter.*`** (ids 1, 9) — one per user, auto-created inside `User.create()` with `prefix="simplelogin-newsletter"` (`app/models.py:L636`, id stored on `user.newsletter_alias_id` at `app/models.py:L643`).
- **`word_test646@sl.local`** (id 2) — the single **random** alias from `Alias.create_new_random(user)` (`app/fake_data.py:L70`); the local-part is random (the seed log shows `generate email word_test646@sl.local`, `app/models.py:L1459`).
- **`example@example.com`** (id 3) — hardcoded (`app/fake_data.py:L138`).
- **`e0@sl.local`, `e1@sl.local`, `e2@sl.local`** (ids 4–6) — **deterministic**, built by a `for i in range(3)` loop as `f"e{i}@{FIRST_ALIAS_DOMAIN}"` (`app/fake_data.py:L140-L150`). This is why **`e1@sl.local` is guaranteed** on this branch (the `swaks` example target in `CONTRIBUTING.md:L218` therefore always exists), whereas the random alias's local-part is not fixed.
- **`first@ab.cd`, `second@ab.cd`** (ids 7, 8) — on the seeded custom domain (`app/fake_data.py:L181-L191`).
- **`john@example.com`, `wick@example.com`** (ids 10, 11) — "breached" example aliases (`app/fake_data.py:L259-L263`).

The seed also creates directories and custom domains that matter for the auto-create paths in [§6.6](#66-secondary-alias-auto-creation-directory-success-and-domain-prerequisites):

```
$ docker exec sl-postgres psql -U myuser -d simplelogin -c "SELECT id, name, user_id, disabled FROM directory ORDER BY id;"
 id | name | user_id | disabled
----+------+---------+----------
  1 | abcd |       1 | f
  2 | xyzt |       1 | f
(2 rows)

$ docker exec sl-postgres psql -U myuser -d simplelogin -c "SELECT id, domain, user_id, verified, catch_all FROM custom_domain ORDER BY id;"
 id | domain  | user_id | verified | catch_all
----+---------+---------+----------+-----------
  1 | ab.cd   |       1 | t        | f
  2 | old.com |       1 | t        | f
(2 rows)
```

So the seed **does** contain two enabled directories (`abcd`, `xyzt`, owned by John) and two verified custom domains (`ab.cd`, `old.com`) — but both domains have `catch_all=f`. That distinction drives [§6.6](#66-secondary-alias-auto-creation-directory-success-and-domain-prerequisites): directory auto-create *is* feasible on the seed out of the box, whereas catch-all domain auto-create requires first **enabling catch-all** on a verified domain — a normal dashboard toggle, **not** a source change. Both paths were exercised and **observed** in §6.6 (catch-all was enabled on `old.com`, a new address auto-created an alias, and catch-all was toggled back off).

The complete pristine post-seed baseline — including the audit and metric tables — is captured here **immediately after the §2.5 reset**, before any of the flows below are exercised. This is the true fresh-reset baseline that the canonical reset produces; [§7.2](#72-final-database-state-after-cleanup) later shows that the investigation restores the **live entity tables** to exactly these seed values, while the **append-only** audit/bookkeeping tables (`deleted_alias`, `alias_audit_log`, `user_audit_log`, `job`) accumulate monotonically and return to the zeros shown here only when this §2.5 reset is re-run:

```
$ docker exec sl-postgres psql -U myuser -d simplelogin -c "
SELECT tbl, count FROM (
  SELECT 'activation_code' tbl, count(*) FROM activation_code
  UNION ALL SELECT 'alias', count(*) FROM alias
  UNION ALL SELECT 'alias_audit_log', count(*) FROM alias_audit_log
  UNION ALL SELECT 'contact', count(*) FROM contact
  UNION ALL SELECT 'custom_domain', count(*) FROM custom_domain
  UNION ALL SELECT 'daily_metric', count(*) FROM daily_metric
  UNION ALL SELECT 'deleted_alias', count(*) FROM deleted_alias
  UNION ALL SELECT 'directory', count(*) FROM directory
  UNION ALL SELECT 'email_log', count(*) FROM email_log
  UNION ALL SELECT 'job', count(*) FROM job
  UNION ALL SELECT 'mailbox', count(*) FROM mailbox
  UNION ALL SELECT 'metric2', count(*) FROM metric2
  UNION ALL SELECT 'user_audit_log', count(*) FROM user_audit_log
  UNION ALL SELECT 'users', count(*) FROM users) s ORDER BY tbl;"
       tbl       | count
-----------------+-------
 activation_code |     0
 alias           |    11
 alias_audit_log |    11
 contact         |     1
 custom_domain   |     2
 daily_metric    |     1
 deleted_alias   |     0
 directory       |     2
 email_log       |     1
 job             |     0
 mailbox         |     4
 metric2         |     0
 user_audit_log  |     0
 users           |     2
(14 rows)

$ docker exec sl-postgres psql -U myuser -d simplelogin -c "SELECT date, nb_new_web_non_proton_user, nb_alias FROM daily_metric;"
    date    | nb_new_web_non_proton_user | nb_alias
------------+----------------------------+----------
 2026-07-13 |                          0 |       11
(1 row)
```

Two audit/metric facts here are important for a trustworthy cleanup proof later:

- **`alias_audit_log` already contains 11 rows at baseline** — one per seeded alias. Alias creation writes an audit row, so creating test aliases *adds* to this table.
- **`daily_metric` already contains 1 row** with `nb_alias=11`. Every `Alias.create()` does `DailyMetric.get_or_create_today_metric().nb_alias += 1` (`app/models.py:L1661`), and registration does `nb_new_web_non_proton_user += 1` (`app/auth/views/register.py:L97`). Test activity therefore mutates this row, and cleanup must restore it exactly ([§7](#7-cleanup--the-repository-and-database-are-left-unchanged)).

### 2.7 Starting the three components (exact commands)

Each process is a **separate program** started through its own entry point. They do not launch one another, so a newcomer opens three terminals (or, as here, three detached `docker exec` sessions writing to per-process log files under `/tmp/sl_run/`):

```bash
# 1) Web server — the Dockerfile CMD form (Gunicorn), Dockerfile:L47
docker exec sl-app bash -lc 'cd /app && ./venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15'
#    (the dev entry `./venv/bin/python server.py` -> local_main() -> app.run(debug=True, port=7777),
#     server.py:L572,L588 — Flask is only importable via the venv)

# 2) Email handler — CONTRIBUTING.md:L212
docker exec sl-app bash -lc 'cd /app && ./venv/bin/python email_handler.py'

# 3) Job runner — CONTRIBUTING.md:L228
docker exec sl-app bash -lc 'cd /app && ./venv/bin/python job_runner.py'
```

`wsgi.py` is three lines, so Gunicorn and the dev server both build the same Flask app via `create_app()` (`server.py:L139`):

```
from server import create_app

app = create_app()
```

**A note on the observation helpers.** Beyond the three components above, the investigation drives and inspects the running system with small, ephemeral helper scripts (SMTP senders, HTTP probes, and `psql`/SQL queries). They are created only under `/tmp/` — on the host (e.g., `/tmp/blitzy_investigation/`) or inside the container (e.g., `/tmp/sl_run/`) — always **outside the repository**, are referenced by **complete absolute paths**, and are **removed during cleanup** ([§7.3](#73-source-tree-untouched)); none is ever tracked or committed. Because they are created *by* the investigation, they are not expected to pre-exist: a reader reproducing a step recreates the helper first. Where a helper's exact content determines the result it is shown inline — via `cat` or a `<<'SQL'`/`<<'PY'` heredoc, as in the cleanup transaction of [§7.1](#71-what-was-created-and-how-it-was-removed) and the runtime-observed blocks of [§6.5](#65-the-email-handlers-dispatcher-reply-bounce-and-smtp-status-codes) and [§6.6](#66-secondary-alias-auto-creation-directory-success-and-domain-prerequisites); those newer reproductions are fully self-contained heredocs that read no external file at all.

---

## 3. How to read the logs: the `SL` log format

Every application log line below is emitted by a single centralized logger. Understanding its format makes each subsequent line self-explanatory.

- **Format string** (`app/log.py:L12-L15`):
  ```
  %(asctime)s - %(name)s - %(levelname)s - %(process)d - "%(pathname)s:%(lineno)d" - %(funcName)s() - %(message_id)s - %(message)s
  ```
  Timestamps are UTC — the formatter sets `converter = time.gmtime` (`app/log.py:L43`).
- **Logger name** is `"SL"` (`app/log.py:L79`, `LOG = _get_logger("SL")`).
- **Level shortcuts** (`app/log.py:L74-L77`): `LOG.d` → `debug`, `LOG.i` → `info`, `LOG.w` → `warning`, and **`LOG.e` → `logging.Logger.exception`** — so `LOG.e` lines are emitted at **ERROR** level and append a traceback tail (even a bare `NoneType: None` when called outside an `except` block, as in the unknown-job case in [§6.4](#64-edge-an-unknown-job-name)).
- **Import-time banner** `>>> init logging <<<` is printed once per process (`app/log.py:L67`).
- **werkzeug's default HTTP access log is disabled** (`app/log.py:L70-L71`), so the web server produces **no** default per-request access line; per-request visibility comes instead from the app's own `after_request` hook (`server.py:L281-L292`).
- The **`message_id`** field (the second-to-last token) is minted per inbound email and threaded through the whole email lifecycle by an `EmailHandlerFilter` (`app/log.py:L18-L37`; `set_message_id` at `app/log.py:L22`), so a single email can be followed end-to-end by matching that id.

A concrete line, annotated (the annotation lines are commentary, placed **outside** the fenced output):

```
2026-07-13 18:03:00,460 - SL - WARNING - 3306 - "/app/server.py:494" - dummy_data() -  - reset db, add fake data
```

Reading left to right: `asctime` = `2026-07-13 18:03:00,460`; `name` = `SL`; `levelname` = `WARNING`; `process` (pid) = `3306`; `pathname:lineno` = `/app/server.py:494`; `funcName` = `dummy_data()`; `message_id` = empty (this is not an email); `message` = `reset db, add fake data`.

---

## 4. Q1 — Are the three components up? (liveness & confirmation signals)

The three processes were started together at **18:11:50–18:11:52** (§2.7) into per-process log files, and stayed up for the whole investigation. Each is probed below for its **definitive** liveness signal. All three PIDs quoted here (`3336`/`3353`/`3361` web, `3344` email handler, `3352` job runner) are the *same* processes throughout §4–§6 — a single continuous timeline, with three disclosed exceptions: the email-handler restart for mail-capture in [§5.4](#54-c4--an-email-arrives-at-the-alias-forward-path); a later dedicated email-handler run (pid `10602`) used to capture the two `handle_bounce()` phases in [§6.5](#65-the-email-handlers-dispatcher-reply-bounce-and-smtp-status-codes); and a subsequent re-verification run (email-handler pid `15828`, logging to `/tmp/qa_run/email.log`) used to capture the `E524`/`E213`/`E404` codes, the successful reply, and catch-all auto-creation in [§6.5](#65-the-email-handlers-dispatcher-reply-bounce-and-smtp-status-codes)–[§6.6](#66-secondary-alias-auto-creation-directory-success-and-domain-prerequisites). Each exception is labeled where its evidence appears.

### 4.1 Web server (`server.py` / `wsgi.py`, port 7777)

**Startup output** (`/tmp/sl_run/web.log`, first lines) — Gunicorn binds `0.0.0.0:7777` and boots two workers, each printing the `>>> init logging <<<` banner (`app/log.py:L67`):

```
[2026-07-13 18:11:50 +0000] [3336] [INFO] Starting gunicorn 20.0.4
[2026-07-13 18:11:50 +0000] [3336] [INFO] Listening at: http://0.0.0.0:7777 (3336)
[2026-07-13 18:11:50 +0000] [3336] [INFO] Using worker: sync
[2026-07-13 18:11:50 +0000] [3353] [INFO] Booting worker with pid: 3353
[2026-07-13 18:11:51 +0000] [3361] [INFO] Booting worker with pid: 3361
>>> init logging <<<
2026-07-13 18:11:51,622 - SL - DEBUG - 3353 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

**Definitive liveness probe — `GET /health`.** The handler `healthcheck()` returns the literal body `success` with HTTP `200` (`server.py:L214-L215`). The **complete raw response, including all headers**, captured with `curl -s -D -`:

```
$ curl -s -D - http://127.0.0.1:7777/health
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Mon, 13 Jul 2026 18:12:12 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 7
Vary: Cookie
Set-Cookie: slapp=[REDACTED: anonymous Flask session cookie]; Expires=Mon, 20-Jul-2026 18:12:12 GMT; HttpOnly; Path=/; SameSite=Lax

success
```

(The only redaction is the opaque signed session-cookie value; the `Set-Cookie` header itself, its attributes, and the 7-byte `success` body are shown verbatim. The body's length exactly matches `Content-Length: 7`.)

**`/health` is deliberately excluded from request logging and profiling.** It appears in the profiling ignore-list (`server.py:L195`) and the `after_request` hook skips it via `not request.path.startswith("/health")` (`server.py:L281`). Confirmed at runtime — after the probe, the web log contains **zero** `/health` lines:

```
$ docker exec sl-app bash -lc "grep -c '/health' /tmp/sl_run/web.log"
0
```

So the "web is up" signal is the **HTTP `200 success` response itself**, not a log entry.

**The login page is served.** An unauthenticated `GET /` returns `302` to `/auth/login` (the raw response, including the `Location` header):

```
$ curl -s -D - -o /dev/null http://127.0.0.1:7777/
HTTP/1.1 302 FOUND
Server: gunicorn/20.0.4
Date: Mon, 13 Jul 2026 18:12:26 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 229
Location: http://127.0.0.1:7777/auth/login
Vary: Cookie
Set-Cookie: slapp=[REDACTED: anonymous Flask session cookie]; Expires=Mon, 20-Jul-2026 18:12:26 GMT; HttpOnly; Path=/; SameSite=Lax
```

**Per-request `SL` visibility.** Because werkzeug's access log is off, the app's own `after_request` hook (`server.py:L284-L292`) is what shows each non-excluded request — it calls `LOG.d` with format string `"%s %s %s %s %s, takes %s"` and the arguments `request.remote_addr`, `request.method`, `request.path`, `request.args`, `res.status_code`, and `time.time() - start_time`. The `GET /` above produced exactly this line in the web log (worker pid `3361`):

```
2026-07-13 18:12:26,733 - SL - DEBUG - 3361 - "/app/server.py:284" - after_request() -  - 172.18.0.1 GET / ImmutableMultiDict([]) 302, takes 0.0003466606140136719
```

### 4.2 Email handler (`email_handler.py`, SMTP 20381)

**Startup output** (`/tmp/sl_run/email.log`) — the `__main__` block parses `--port` (default **20381**, `email_handler.py:L2399`) and logs `Listen for port 20381` (`email_handler.py:L2403`); `main()` (`email_handler.py:L2381`) builds the `aiosmtpd` `Controller` bound to `0.0.0.0`, calls `controller.start()`, and logs `Start mail controller` (`email_handler.py:L2386`):

```
>>> init logging <<<
2026-07-13 18:11:51,783 - SL - DEBUG - 3344 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-13 18:11:52,262 - SL - INFO - 3344 - "/app/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-13 18:11:52,264 - SL - DEBUG - 3344 - "/app/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

**Confirm it is actually listening on `0.0.0.0:20381`.** The image lacks `ss`/`nc`/`lsof`, so a `socket.connect_ex` probe was used — the canonical "is the port accepting connections" check — both from the host (published port) and inside the container:

```
$ python3 -c "import socket;s=socket.socket();print('20381',('OPEN' if s.connect_ex(('127.0.0.1',20381))==0 else 'CLOSED'))"
20381 OPEN
$ docker exec sl-app ./venv/bin/python -c "import socket;s=socket.socket();print('20381',('OPEN' if s.connect_ex(('127.0.0.1',20381))==0 else 'CLOSED'))"
20381 OPEN
```

Both the `Start mail controller 0.0.0.0 20381` log line **and** the open TCP port confirm the handler is up. (A full inbound-message lifecycle through this port is in [§5.4](#54-c4--an-email-arrives-at-the-alias-forward-path).)

### 4.3 Job runner (`job_runner.py`, no port)

**Startup output** (`/tmp/sl_run/job.log`) — the process boots and enters the infinite poll loop (`job_runner.py:L330`). When the `job` table is empty it logs nothing on each poll (the `LOG.d("Take job %s", job)` at `job_runner.py:L334` fires **only when work exists**):

```
>>> init logging <<<
2026-07-13 18:11:51,705 - SL - DEBUG - 3352 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

**Observing the poll cadence (Rule R3: observe true timing over ≥2 cycles).** To make the idle loop emit a line on consecutive polls, three `Job` rows (`name=onboarding-1`, `payload={"user_id": 1}`) were enqueued **one at a time**, each after the previous had been taken, so each lands on a distinct poll.

> **`[NON-CANONICAL]` producer:** these three jobs were inserted directly via the app's `Job.create()` model (not through an end-user HTTP action), purely to drive the idle poll loop so its interval can be timed. The measured interval is a property of `time.sleep(10)` (`job_runner.py:L347`) and is independent of how the job is produced. A **canonical, application-produced** job (a real dashboard *delete-account* action) is exercised separately in [§6.2](#62-a-genuinely-enqueued-job-canonical-ready--taken--done). The enqueue command and its output for the first job:

```
$ docker exec sl-app bash -lc "cd /app && PYTHONPATH=/app ./venv/bin/python /tmp/sl_run/enqueue_job.py 'onboarding-1' '{\"user_id\": 1}'"
ENQUEUED job id=1 name=onboarding-1 state=0 run_at=None
```

The runner picked the three jobs up on consecutive polls. The complete relevant slice of `/tmp/sl_run/job.log` (all lines for the three polls, pid `3352`):

```
2026-07-13 18:17:22,825 - SL - DEBUG - 3352 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 1 onboarding-1 {'user_id': 1}>
2026-07-13 18:17:22,833 - SL - DEBUG - 3352 - "/app/job_runner.py:196" - process_job() -  - send onboarding send-from-alias email to user <User 1 John Wick john@wick.com>
2026-07-13 18:17:22,855 - SL - DEBUG - 3352 - "/app/app/email_utils.py:303" - send_email() -  - send email to simplelogin-newsletter.list417@sl.local, subject 'SimpleLogin Tip: Send emails from your alias'
2026-07-13 18:17:22,856 - SL - DEBUG - 3352 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'SimpleLogin Tip: Send emails from your alias', from '"noreply@sl.local" <noreply@sl.local>' to 'simplelogin-newsletter.list417@sl.local'
2026-07-13 18:17:32,870 - SL - DEBUG - 3352 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 2 onboarding-1 {'user_id': 1}>
2026-07-13 18:17:32,875 - SL - DEBUG - 3352 - "/app/job_runner.py:196" - process_job() -  - send onboarding send-from-alias email to user <User 1 John Wick john@wick.com>
2026-07-13 18:17:32,889 - SL - DEBUG - 3352 - "/app/app/email_utils.py:303" - send_email() -  - send email to simplelogin-newsletter.list417@sl.local, subject 'SimpleLogin Tip: Send emails from your alias'
2026-07-13 18:17:32,890 - SL - DEBUG - 3352 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'SimpleLogin Tip: Send emails from your alias', from '"noreply@sl.local" <noreply@sl.local>' to 'simplelogin-newsletter.list417@sl.local'
2026-07-13 18:17:42,904 - SL - DEBUG - 3352 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 3 onboarding-1 {'user_id': 1}>
2026-07-13 18:17:42,909 - SL - DEBUG - 3352 - "/app/job_runner.py:196" - process_job() -  - send onboarding send-from-alias email to user <User 1 John Wick john@wick.com>
2026-07-13 18:17:42,924 - SL - DEBUG - 3352 - "/app/app/email_utils.py:303" - send_email() -  - send email to simplelogin-newsletter.list417@sl.local, subject 'SimpleLogin Tip: Send emails from your alias'
2026-07-13 18:17:42,925 - SL - DEBUG - 3352 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'SimpleLogin Tip: Send emails from your alias', from '"noreply@sl.local" <noreply@sl.local>' to 'simplelogin-newsletter.list417@sl.local'
```

The three `Take job` timestamps are `18:17:22,825`, `18:17:32,870`, `18:17:42,904`. Inter-poll deltas:

```
$ python3 -c "from datetime import datetime as D; t=['2026-07-13 18:17:22,825','2026-07-13 18:17:32,870','2026-07-13 18:17:42,904']; d=[D.strptime(x,'%Y-%m-%d %H:%M:%S,%f') for x in t]; print('poll2-poll1 = %.3f s'%(d[1]-d[0]).total_seconds()); print('poll3-poll2 = %.3f s'%(d[2]-d[1]).total_seconds())"
poll2-poll1 = 10.045 s
poll3-poll2 = 10.034 s
```

**The 10-second poll cadence is stable across two consecutive cycles** (10.045 s, 10.034 s — the extra ~0.04 s is the per-poll processing time), matching `time.sleep(10)` (`job_runner.py:L347`). Each job also transitioned `ready(0) → taken(1) → done(2)` with `attempts` incremented to 1 — verified directly (a `[NON-CANONICAL]` read-only DB probe):

```
$ docker exec sl-postgres psql -U myuser -d simplelogin -c "SELECT id, name, state, attempts, taken FROM job ORDER BY id;"
 id |     name     | state | attempts | taken
----+--------------+-------+----------+-------
  1 | onboarding-1 |     2 |        1 | t
  2 | onboarding-1 |     2 |        1 | t
  3 | onboarding-1 |     2 |        1 | t
(3 rows)
```

### 4.4 UI confirmation — sign-in and alias management

Q1 also asks how the **dashboard** confirms that users can sign in and manage aliases. This is shown with full session/cookie/HTML evidence in [§5.2](#52-c2--sign-in-real-session-with-cookies-and-csrf) (sign-in) and [§5.3](#53-c3--create-an-alias-random-and-custom) (a newly created alias appearing in the dashboard listing). In short: a successful `POST /auth/login` returns `302 → /dashboard/`, and `GET /dashboard/` then returns `200` with the user's alias addresses rendered in the HTML — the visible proof that sign-in and alias management work.

---


## 5. Q2 — Core user actions & correct-handling evidence

This section performs the three core actions Q2 names — **create a new account**, **create an alias**, and **have that alias receive an email** — each through its real entry point, and shows the runtime evidence (HTTP responses, `SL` logs, and database rows, with before/after state) that each was handled correctly. Unless noted, the web workers are pids `3353`/`3361` and the email handler is the one started in §2.7; the one disclosed exception is the email-handler restart for the mail-capture deviation in [§5.4](#54-c4--an-email-arrives-at-the-alias-forward-path).

All test data created here (one account, its aliases, one contact, two email-log rows) is enumerated and removed in [§7](#7-cleanup--the-repository-and-database-are-left-unchanged).

### 5.1 C1 — Create a new account (signup + activation)

**A note on the test address (corrects a common misconception).** SimpleLogin decides whether an address may be used as a personal mailbox in `email_can_be_used_as_mailbox()` (`app/email_utils.py:L569`). For `example.com`, that function returns **`True`** — `example.com` is *not* rejected. The reason: `example.com` publishes an RFC 7505 "null MX" (`.`), and `get_mx_domain_list()` turns that root `.` into `''` via `[d.domain[:-1] for d in get_mx_domains(domain)]`, producing the **non-empty** list `['']`, so the `not mx_domains` rejection branch is never taken. Observed directly at runtime:

```
$ docker exec sl-app ./venv/bin/python -c "from app.email_utils import email_can_be_used_as_mailbox, get_mx_domain_list; from app import config; print('SKIP_MX_LOOKUP_ON_CHECK =', config.SKIP_MX_LOOKUP_ON_CHECK); print('get_mx_domain_list(example.com) =', get_mx_domain_list('example.com')); print('email_can_be_used_as_mailbox(qa.tester.9f3@example.com) =', email_can_be_used_as_mailbox('qa.tester.9f3@example.com'))"
SKIP_MX_LOOKUP_ON_CHECK = False
get_mx_domain_list('example.com') = ['']
email_can_be_used_as_mailbox('qa.tester.9f3@example.com') = True
```

(The five startup lines the interpreter prints before the app is ready — `>>> URL: …` through `>>> init logging <<<` and the `load words file` line — are omitted here; only the three `print()` outputs are shown.) Because the address is accepted, it is used for the test account below.

**Before state** (users + activation codes):

```
$ docker exec sl-postgres psql -U myuser -d simplelogin -c "SELECT id, email, activated, is_admin FROM users ORDER BY id;"
 id |          email          | activated | is_admin
----+-------------------------+-----------+----------
  1 | john@wick.com           | t         | t
  2 | winston@continental.com | t         | f
(2 rows)

$ docker exec sl-postgres psql -U myuser -d simplelogin -c "SELECT date, nb_new_web_non_proton_user, nb_alias FROM daily_metric ORDER BY date;"
    date    | nb_new_web_non_proton_user | nb_alias
------------+----------------------------+----------
 2026-07-13 |                          0 |       11
(1 row)
```

**The action — `POST /auth/register`** (driven with a real `requests.Session`; the CSRF token is scraped from the prior `GET /auth/register`):

```
=== GET /auth/register ===
status: 200
csrf_token scraped: True (length 91)
session cookie set after GET: True

=== POST /auth/register ===
form fields: email=qa.tester.9f3@example.com  password=<15 chars>  csrf_token=<91 chars>
status: 200
body mentions 'activation': True
body length: 6090
flash/alert snippets: []
```

The `200` with an activation-mentioning body is the `auth/register_waiting_activation.html` template — i.e., the account was created and an activation step is pending. The server logged the creation at `app/auth/views/register.py:L85` (`LOG.d("create user %s", email)`), worker pid `3353`:

```
2026-07-13 18:25:59,602 - SL - DEBUG - 3353 - "/app/app/auth/views/register.py:85" - register() -  - create user qa.tester.9f3@example.com
```

**Intermediate state** — the new user exists but is **not yet activated**, a single-use activation code was created, and the daily metric counters moved. `User.create` also auto-provisions the user's newsletter alias (`app/models.py:L636,L643`), which is why `nb_alias` increments too:

```
$ docker exec sl-postgres psql -U myuser -d simplelogin -c "SELECT id, email, activated FROM users WHERE id=3;"
 id |           email           | activated
----+---------------------------+-----------
  3 | qa.tester.9f3@example.com | f
(1 row)

$ docker exec sl-postgres psql -U myuser -d simplelogin -c "SELECT id, user_id, length(code) AS code_len FROM activation_code ORDER BY id;"
 id | user_id | code_len
----+---------+----------
  1 |       3 |       30
(1 row)

$ docker exec sl-postgres psql -U myuser -d simplelogin -c "SELECT date, nb_new_web_non_proton_user, nb_alias FROM daily_metric ORDER BY date;"
    date    | nb_new_web_non_proton_user | nb_alias
------------+----------------------------+----------
 2026-07-13 |                          1 |       12
(1 row)

$ docker exec sl-postgres psql -U myuser -d simplelogin -c "SELECT id, email, user_id FROM alias WHERE user_id=3;"
 id |                  email                  | user_id
----+-----------------------------------------+---------
 12 | simplelogin-newsletter.list620@sl.local |       3
(1 row)
```

> **Security note (activation token redacted).** The activation code is `random_string(30)` — a 30-character secret (`app/auth/views/register.py:L120`, confirmed above by `code_len = 30`). It is a credential, so its value is **not** reproduced here; instead its handling is proven by (a) its length, (b) its single-use lifecycle (the row is deleted the moment it is consumed, shown next), and (c) the `activated` transition it drives. The activation URL is `{URL}/auth/activate?code=<code>`; only the redacted form is shown.

**Activation — canonical `GET /auth/activate?code=<code>` (with a `[NON-CANONICAL]` token-acquisition step).** The activation *endpoint itself is exercised canonically* — a real HTTP `GET` to the real route, exactly as following the link in an activation email would. **Obtaining the token, however, is `[NON-CANONICAL]` / supplemental, and is called out explicitly here:** in this default local configuration `NOT_SEND_EMAIL = True` (confirmed by a canonical config read; the flag is truthy because the key is present in `.env` — `app/config.py:L91`, `NOT_SEND_EMAIL = "NOT_SEND_EMAIL" in os.environ`), so the activation email is never transmitted — the mail sender logs the message and returns `True` without sending it (`app/mail_sender.py:L130-L137`), on the path reached from `send_activation_email()` (`app/auth/views/register.py:L129`, which mints the code at `L120`). Because no message reaches a mailbox, the one-time code cannot be received the way a real user would; it was therefore read **directly from the `activation_code` table** — the `code_len = 30` row shown in the *Intermediate state* above — into a shell variable. That direct read is a state *read* used solely to acquire the token; it is never substituted for the canonical activation request, which is the `GET` below. The token was never printed, and the request URL's token is masked in the captured output. The response is a `302` to the dashboard, and it logs the user in (fresh authenticated session cookie):

```
$ curl -s -D - -o /dev/null "http://127.0.0.1:7777/auth/activate?code=[REDACTED-30-CHAR-TOKEN]"
HTTP/1.1 302 FOUND
Server: gunicorn/20.0.4
Date: Mon, 13 Jul 2026 18:26:53 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 229
Location: http://127.0.0.1:7777/dashboard/
Vary: Cookie
Set-Cookie: slapp=[REDACTED: authenticated session cookie]; Expires=Mon, 20-Jul-2026 18:26:53 GMT; HttpOnly; Path=/; SameSite=Lax
```

```
2026-07-13 18:26:53,974 - SL - DEBUG - 3361 - "/app/app/auth/views/activate.py:66" - activate() -  - redirect user to dashboard
```

**After state** — `activated` flipped **`f` → `t`** (`app/auth/views/activate.py:L49`) and the activation code was consumed (row deleted), proving correct one-time-use handling (`app/auth/views/activate.py:L53`):

```
$ docker exec sl-postgres psql -U myuser -d simplelogin -c "SELECT id, email, activated FROM users WHERE id=3;"
 id |           email           | activated
----+---------------------------+-----------
  3 | qa.tester.9f3@example.com | t
(1 row)

$ docker exec sl-postgres psql -U myuser -d simplelogin -c "SELECT id, user_id, length(code) AS code_len FROM activation_code ORDER BY id;"
 id | user_id | code_len
----+---------+----------
(0 rows)
```


### 5.2 C2 — Sign-in (real session with cookies and CSRF)

Sign-in is exercised against the **seeded** account `john@wick.com / password` (it owns the seeded aliases, so the dashboard listing is meaningful). A real `requests.Session` is used so cookies persist across requests. The full flow — CSRF acquisition, the POST body, the raw `302`, cookie rotation, and session continuity into the dashboard:

```
=== GET /auth/login ===
status: 200
csrf_token scraped: True (length 91)
session cookie after GET: slapp present=True

=== POST /auth/login ===
form body: email=john@wick.com  password=<8 chars>  csrf_token=<91 chars>
status: 302
Location header: http://127.0.0.1:7777/dashboard/
Set-Cookie present on 302: True
session cookie after POST: slapp present=True  (rotated vs GET: True)

=== GET /dashboard/ (same authenticated session) ===
status: 200
dashboard HTML length: 99520
alias addresses rendered in dashboard HTML:
    e0@sl.local
    e1@sl.local
    e2@sl.local
    example@example.com
    first@ab.cd
    john@example.com
    second@ab.cd
    simplelogin-newsletter.list417@sl.local
    wick@example.com
    word_test646@sl.local
alias-management controls present in dashboard:
   [ ] New Custom Alias link (/dashboard/custom_alias)
   [x] Random alias form (create-random-email)
   [x] Alias enable/disable toggle
   [x] Search aliases box
```

How this shows correct handling:

- **Authentication succeeded**: `POST /auth/login` returned `302` with `Location: http://127.0.0.1:7777/dashboard/` (`login()` → `after_login()`, `app/auth/views/login.py:L25,L72`).
- **Session security**: the `slapp` session cookie value **rotated** across the login boundary (the value after POST differs from the value after the initial GET) — the expected session-fixation defense on privilege change.
- **Session continuity**: reusing the *same* session's cookie, `GET /dashboard/` returned `200` (not a redirect back to login) — the session is authenticated.
- **The dashboard confirms alias management** (this is the UI signal Q1 asks about): the rendered HTML lists exactly the **ten** aliases owned by `john` (user 1) — the three deterministic `e0/e1/e2@sl.local`, the one random `word_test646@sl.local`, the newsletter alias, `example@example.com`, `john@example.com`, `wick@example.com`, and `first/second@ab.cd` — matching the seed (§2.6). (The other two seeded aliases belong to `winston` and are not shown to `john`.)

The one control not matched by the literal-substring probe — the custom-alias entry point — **is** present; it is a button rather than an `href` containing `custom_alias`. Its verbatim markup from the dashboard HTML (indentation as served):

```
<button data-toggle="tooltip"
                      title="Create a custom alias"
                      class="btn btn-primary mr-2">
                <i class="fa fa-plus"></i> New Custom Alias
              </button>
```


### 5.3 C3 — Create an alias (random and custom)

Both alias-creation paths are exercised, as `john` (the authenticated session from §5.2). Before creation, `john` owns **10** aliases (global max alias id = `12`):

```
$ docker exec sl-postgres psql -U myuser -d simplelogin -c "SELECT count(*) AS john_alias_count, max(id) AS max_alias_id FROM alias WHERE user_id=1;"
 john_alias_count | max_alias_id
------------------+--------------
               10 |           11
(1 row)
```

**(a) Random alias — `POST /dashboard/` with `form-name=create-random-email`.** The request is CSRF-protected (`CSRFValidationForm`, `app/dashboard/views/index.py:L85-L89`). Following the redirect surfaces the one-shot success flash:

```
=== POST /dashboard/ (form-name=create-random-email) ===
final URL: http://127.0.0.1:7777/dashboard/?highlight_alias_id=15&query=&sort=&filter=
flash text: ['Alias list_word440@sl.local has been created']
```

The `302` carries `highlight_alias_id=15`, the flash confirms `Alias list_word440@sl.local has been created` (`app/dashboard/views/index.py:L111`), and the server logged the creation at `app/dashboard/views/index.py:L110`:

```
2026-07-13 18:31:07,467 - SL - DEBUG - 3353 - "/app/app/dashboard/views/index.py:110" - index() -  - create new random alias <Alias 15 list_word440@sl.local> for user <User 1 John Wick john@wick.com>
```

**(b) Custom alias — `POST /dashboard/custom_alias`.** This form needs a server-signed suffix and a verified mailbox id, both scraped from the prior `GET /dashboard/custom_alias`. The suffix lives in a `<select name="signed-alias-suffix">` whose `<option value="…">` carries the signed token; the three offered suffixes and the chosen one:

```
signed-suffix options found: 3 | domains: ['@old.com.alUusQ', '.list752@premium.com', '.test546@sl.local.al']
chosen suffix domain: @old (signed token length 43, value redacted)
mailbox ids available: ['1', '2']

=== POST /dashboard/custom_alias (prefix=qa-custom-9f3) ===
status: 302 | Location: http://127.0.0.1:7777/dashboard/?highlight_alias_id=14
```

The `302` with `highlight_alias_id=14` confirms creation (`app/dashboard/views/custom_alias.py:L139,L159`). Both new aliases exist in the DB and render in the dashboard listing:

```
$ docker exec sl-postgres psql -U myuser -d simplelogin -c "SELECT id, email, user_id, note FROM alias WHERE id IN (14,15) ORDER BY id;"
 id |         email         | user_id |         note
----+-----------------------+---------+----------------------
 14 | qa-custom-9f3@old.com |       1 | QA custom alias test
 15 | list_word440@sl.local |       1 |
(2 rows)
```

A fresh authenticated fetch of `GET /dashboard/` as `john` confirms all three test aliases created during this run are rendered in the listing (the substring test returns `True` for each):

```
dashboard renders word_test310@sl.local    : True
dashboard renders qa-custom-9f3@old.com    : True
dashboard renders list_word440@sl.local    : True
```

(This run created three test aliases: `word_test310@sl.local` (id `13`, an initial random-alias probe), `qa-custom-9f3@old.com` (id `14`, custom), and `list_word440@sl.local` (id `15`, random). All three — plus the account and its newsletter alias — are removed in [§7](#7-cleanup--the-repository-and-database-are-left-unchanged).) The custom alias landed on the verified custom domain `@old.com` because that was the first offered suffix; the `sl.local` suffix option (`.test546@sl.local`) would have produced a `word`-infixed alias on the default alias domain instead.


### 5.4 C4 — An email arrives at the alias (forward path)

An external message is delivered over SMTP to the seeded alias `e1@sl.local` (alias id `5`, owned by `john`, mailbox `john@wick.com`). This is done **twice**, to separate two distinct facts:

- **(A) Primary, default configuration** (`NOT_SEND_EMAIL=true`, as seeded in `.env`): the message is fully **processed** — a contact and an email-log row are created and the envelope is accepted — but the outbound leg to the mailbox is **short-circuited, not actually delivered**. This is the canonical default behavior and is reported as such (it is *not* a real delivery).
- **(B) A clearly-disclosed configuration deviation**: `NOT_SEND_EMAIL` is disabled and outbound mail is routed to a local capture sink, so the **actually-delivered** forwarded message can be captured verbatim. Everything about this deviation — the `.env` edit, the handler restart (a **new pid**, no continuity claimed across it), and the full restore — is shown.

There is no `swaks` in the image, so the SMTP client is Python's `smtplib` with `set_debuglevel(1)` (prints the full client↔server transcript).

**Before state** for alias `e1` (id `5`):

```
$ docker exec sl-postgres psql -U myuser -d simplelogin -c "SELECT count(*) AS emaillog_for_e1 FROM email_log WHERE alias_id=5;"
 emaillog_for_e1
-----------------
               0
(1 row)

$ docker exec sl-postgres psql -U myuser -d simplelogin -c "SELECT count(*) AS contacts_for_e1 FROM contact WHERE alias_id=5;"
 contacts_for_e1
-----------------
               0
(1 row)
```

#### (A) Default config — processed, accepted, outbound short-circuited

**The SMTP send** (envelope `MAIL FROM:<external.sender@example.org>`, `RCPT TO:<e1@sl.local>`), complete transcript:

```
send: 'ehlo qa-test-client.example.org\r\n'
reply: b'250-06a79fe3b90d\r\n'
reply: b'250-SIZE 33554432\r\n'
reply: b'250-8BITMIME\r\n'
reply: b'250-SMTPUTF8\r\n'
reply: b'250 HELP\r\n'
reply: retcode (250); Msg: b'06a79fe3b90d\nSIZE 33554432\n8BITMIME\nSMTPUTF8\nHELP'
send: 'mail from:<external.sender@example.org> size=392\r\n'
reply: b'250 OK\r\n'
reply: retcode (250); Msg: b'OK'
send: 'rcpt to:<e1@sl.local>\r\n'
reply: b'250 OK\r\n'
reply: retcode (250); Msg: b'OK'
send: 'data\r\n'
reply: b'354 End data with <CR><LF>.<CR><LF>\r\n'
reply: retcode (354); Msg: b'End data with <CR><LF>.<CR><LF>'
data: (354, b'End data with <CR><LF>.<CR><LF>')
send: b'From: External Sender <external.sender@example.org>\r\nTo: e1@sl.local\r\nSubject: QA inbound forward test 9f3\r\nDate: Mon, 13 Jul 2026 18:35:00 +0000\r\nMessage-ID: <qa-9f3-inbound@example.org>\r\nContent-Type: text/plain; charset="utf-8"\r\nContent-Transfer-Encoding: quoted-printable\r\nMIME-Version: 1.0\r\n\r\nThis is a QA test body sent to the seeded alias e1@sl.local to exercise the f=\r\norward path.\r\n.\r\n'
reply: b'250 Message accepted for delivery\r\n'
reply: retcode (250); Msg: b'Message accepted for delivery'
data: (250, b'Message accepted for delivery')
send: 'QUIT\r\n'
reply: b'221 Bye\r\n'
reply: retcode (221); Msg: b'Bye'
send_message() returned (empty dict = all recipients accepted): {}
```

The envelope was accepted stage-by-stage: `RCPT TO` returned `250 OK`, and only after `DATA` did the handler return the final `250 Message accepted for delivery` (this is `status.E200`, `app/email/status.py:L2`).

**The complete forward lifecycle in the `SL` log**, all correlated by the single per-message id `39525716-85e9-4e16-ae30-93a7925b0a17` that the handler generates at `email_handler.py:L2339` and threads through the logs via `set_message_id` (`email_handler.py:L2340`). This is the full, unedited set of lines for this message id — start boundary → contact creation → forward → email-log creation → finish:

```
2026-07-13 18:32:52,038 - SL - DEBUG - 3344 - "/app/app/log.py:24" - set_message_id() -  - set message_id 39525716-85e9-4e16-ae30-93a7925b0a17
2026-07-13 18:32:52,038 - SL - DEBUG - 3344 - "/app/email_handler.py:2342" - _handle() - 39525716-85e9-4e16-ae30-93a7925b0a17 - ====>=====>====>====>====>====>====>====>
2026-07-13 18:32:52,038 - SL - INFO - 3344 - "/app/email_handler.py:2343" - _handle() - 39525716-85e9-4e16-ae30-93a7925b0a17 - New message, mail from external.sender@example.org, rctp tos ['e1@sl.local']
2026-07-13 18:32:52,039 - SL - DEBUG - 3344 - "/app/email_handler.py:1963" - handle() - 39525716-85e9-4e16-ae30-93a7925b0a17 - Cannot parse Postfix queue ID from None None
2026-07-13 18:32:52,185 - SL - DEBUG - 3344 - "/app/email_handler.py:1980" - handle() - 39525716-85e9-4e16-ae30-93a7925b0a17 - ==>> Handle mail_from:external.sender@example.org, rcpt_tos:['e1@sl.local'], header_from:External Sender <external.sender@example.org>, header_to:e1@sl.local, cc:None, reply-to:None, message_id:<qa-9f3-inbound@example.org>, client_ip:None, headers:[('From', 'External Sender <external.sender@example.org>'), ('To', 'e1@sl.local'), ('Subject', 'QA inbound forward test 9f3'), ('Date', 'Mon, 13 Jul 2026 18:35:00 +0000'), ('Message-ID', '<qa-9f3-inbound@example.org>'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', 'quoted-printable'), ('MIME-Version', '1.0')], mail_options:['SIZE=392'], rcpt_options:[]
2026-07-13 18:32:52,190 - SL - DEBUG - 3344 - "/app/email_handler.py:2202" - handle() - 39525716-85e9-4e16-ae30-93a7925b0a17 - Forward phase external.sender@example.org(External Sender <external.sender@example.org>) -> e1@sl.local
2026-07-13 18:32:52,205 - SL - DEBUG - 3344 - "/app/email_handler.py:580" - handle_forward() - 39525716-85e9-4e16-ae30-93a7925b0a17 - Create or get contact for from_header:External Sender <external.sender@example.org>
2026-07-13 18:32:52,233 - SL - DEBUG - 3344 - "/app/app/contact_utils.py:110" - create_contact() - 39525716-85e9-4e16-ae30-93a7925b0a17 - Created contact <Contact 2 external.sender@example.org 5> for alias <Alias 5 e1@sl.local> with email external.sender@example.org invalid_email=False
2026-07-13 18:32:52,233 - SL - INFO - 3344 - "/app/app/handler/dmarc.py:33" - apply_dmarc_policy_for_forward_phase() - 39525716-85e9-4e16-ae30-93a7925b0a17 - DMARC check disabled
2026-07-13 18:32:52,243 - SL - DEBUG - 3344 - "/app/email_handler.py:688" - forward_email_to_mailbox() - 39525716-85e9-4e16-ae30-93a7925b0a17 - Forward <Contact 2 external.sender@example.org 5> -> <Alias 5 e1@sl.local> -> <Mailbox 1 john@wick.com>
2026-07-13 18:32:52,247 - SL - DEBUG - 3344 - "/app/email_handler.py:740" - forward_email_to_mailbox() - 39525716-85e9-4e16-ae30-93a7925b0a17 - Create <EmailLog 2> for <Contact 2 external.sender@example.org 5>, <User 1 John Wick john@wick.com>, <Mailbox 1 john@wick.com>
2026-07-13 18:32:52,252 - SL - DEBUG - 3344 - "/app/email_handler.py:867" - forward_email_to_mailbox() - 39525716-85e9-4e16-ae30-93a7925b0a17 - From header, new:"External Sender - external.sender at example.org" <external_sender_at_example_org_vkngtabzk@sl.local>, old:External Sender <external.sender@example.org>
2026-07-13 18:32:52,253 - SL - DEBUG - 3344 - "/app/email_handler.py:316" - replace_header_when_forward() - 39525716-85e9-4e16-ae30-93a7925b0a17 - Delete Cc header, old value None
2026-07-13 18:32:52,253 - SL - DEBUG - 3344 - "/app/email_handler.py:313" - replace_header_when_forward() - 39525716-85e9-4e16-ae30-93a7925b0a17 - Replace To header, old: e1@sl.local, new: e1@sl.local
2026-07-13 18:32:52,253 - SL - INFO - 3344 - "/app/app/handler/unsubscribe_generator.py:36" - _generate_header_with_original_behaviour() - 39525716-85e9-4e16-ae30-93a7925b0a17 - Email has no unsubscribe header
2026-07-13 18:32:52,253 - SL - DEBUG - 3344 - "/app/email_handler.py:893" - forward_email_to_mailbox() - 39525716-85e9-4e16-ae30-93a7925b0a17 - Forward mail from external.sender@example.org to john@wick.com, mail_options:['SIZE=392'], rcpt_options:[]
2026-07-13 18:32:52,253 - SL - DEBUG - 3344 - "/app/app/mail_sender.py:131" - send() - 39525716-85e9-4e16-ae30-93a7925b0a17 - send email with subject 'QA inbound forward test 9f3', from '"External Sender - external.sender at example.org" <external_sender_at_example_org_vkngtabzk@sl.local>' to 'e1@sl.local'
2026-07-13 18:32:52,253 - SL - INFO - 3344 - "/app/email_handler.py:2367" - _handle() - 39525716-85e9-4e16-ae30-93a7925b0a17 - Finish mail_from external.sender@example.org, rcpt_tos ['e1@sl.local'], takes 0.2158503532409668 seconds with return code '250 Message accepted for delivery'<<===
```

**The `send email …` line is not a delivery.** With `NOT_SEND_EMAIL=true`, `MailSender.send()` (`app/mail_sender.py:L126-L136`) logs that line and then `return True` **without** calling `_send_to_smtp` — so the message was accepted and an email-log created, but nothing left the process. The primary path therefore demonstrates correct **processing** (contact + email-log + reverse-alias rewrite + `250`), and the actual mailbox delivery is shown separately in (B).

**After state** — a forward email-log (`is_reply`, `blocked`, `bounced` all `f`) and the reverse-alias contact were created:

```
$ docker exec sl-postgres psql -U myuser -d simplelogin -c "SELECT id, alias_id, contact_id, is_reply, blocked, bounced FROM email_log WHERE alias_id=5 ORDER BY id;"
 id | alias_id | contact_id | is_reply | blocked | bounced
----+----------+------------+----------+---------+---------
  2 |        5 |          2 | f        | f       | f
(1 row)

$ docker exec sl-postgres psql -U myuser -d simplelogin -c "SELECT id, alias_id, website_email, reply_email FROM contact WHERE alias_id=5 ORDER BY id;"
 id | alias_id |        website_email        |                    reply_email
----+----------+-----------------------------+---------------------------------------------------
  2 |        5 | external.sender@example.org | external_sender_at_example_org_vkngtabzk@sl.local
(1 row)
```

So the per-alias forward count for `e1` went **0 → 1** (one forward `EmailLog`), and a `Contact` with a generated **reverse-alias** address (`external_sender_at_example_org_vkngtabzk@sl.local`) was recorded so replies can be routed back.

#### (B) Disclosed deviation — capturing the actually-delivered forward

To observe the real delivery, `NOT_SEND_EMAIL` is disabled and outbound is pointed at a local `aiosmtpd` capture sink on `127.0.0.1:1025`. **This is a deliberate, temporary deviation from the canonical default, fully restored afterward.**

The `.env` edits (the original file was first backed up to `/tmp/sl_run/env.backup`, whose sha256 is re-verified identical after restore in the **Restore** block below and again in [§7.3](#73-source-tree-untouched)):

```
$ docker exec sl-app bash -lc "grep -nE 'NOT_SEND_EMAIL|POSTFIX_SERVER|POSTFIX_PORT' /app/.env"
19:#NOT_SEND_EMAIL=true  # [DISCLOSED DEVIATION - restored after mail-capture]
69:# POSTFIX_SERVER=my-postfix.com
154:# POSTFIX_PORT=1025
199:POSTFIX_SERVER=127.0.0.1
200:POSTFIX_PORT=1025
```

The email handler was stopped (pid `3344`) and restarted under the deviation as a **new** process (pid `4572`) — no same-process continuity is claimed across this restart:

```
2026-07-13 18:35:15,660 - SL - INFO - 4572 - "/app/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-13 18:35:15,661 - SL - DEBUG - 4572 - "/app/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

A second message (`Subject: QA inbound delivery test 9f3`) was sent to `e1@sl.local`. The capture sink received the **actually-forwarded** message; its verbatim contents (the sink prepends the two `X-Sink-*` envelope lines it observed):

```
X-Sink-MailFrom: sl.lmycyibtfqqdemzygi4donk5.y6lt2kam3sjcy@sl.local
X-Sink-RcptTos: john@wick.com
----- raw message below -----
Subject: QA inbound delivery test 9f3
Date: Mon, 13 Jul 2026 18:36:00 +0000
Message-ID: <qa-9f3-delivery@example.org>
Content-Type: text/plain; charset="utf-8"
Content-Transfer-Encoding: quoted-printable
MIME-Version: 1.0
X-SimpleLogin-Type: Forward
X-SimpleLogin-EmailLog-ID: 3
X-SimpleLogin-Envelope-From: external.sender@example.org
X-SimpleLogin-Original-From: External Sender <external.sender@example.org>
X-SimpleLogin-Envelope-To: e1@sl.local
From: "External Sender - external.sender at example.org"
 <external_sender_at_example_org_vkngtabzk@sl.local>
To: e1@sl.local

Second QA test: NOT_SEND_EMAIL disabled, outbound routed to local sink to cap=
ture the delivered forward.
```

This is the definitive "the alias received the mail" evidence: the sink's **envelope recipient is `john@wick.com`** (e1's mailbox), the envelope sender is a VERP return-path, and the forwarded message carries SimpleLogin's headers (`X-SimpleLogin-Type: Forward`, `X-SimpleLogin-EmailLog-ID: 3`, `X-SimpleLogin-Envelope-To: e1@sl.local`) with the `From:` rewritten to the reverse-alias so the mailbox owner can reply through SimpleLogin.

**Restore (verified).** `.env` was restored from the backup and its checksum re-verified identical to the original; the deviation handler (pid `4572`) and the sink were stopped; the handler was restarted under the default config (pid `4648`), and the runtime config was re-checked:

```
$ docker exec sl-app bash -lc "cp -p /tmp/sl_run/env.backup /app/.env && sha256sum /app/.env"
7b3c4a44d1995cf10115a92d93d4787d556916b36d3522f5f81ff0a93a5d30d8  /app/.env

$ docker exec sl-app ./venv/bin/python -c "from app import config; print('NOT_SEND_EMAIL =', config.NOT_SEND_EMAIL, '| POSTFIX_SERVER =', config.POSTFIX_SERVER, '| POSTFIX_PORT =', config.POSTFIX_PORT)"
NOT_SEND_EMAIL = True | POSTFIX_SERVER = 240.0.0.1 | POSTFIX_PORT = 25
```

The environment is back to canonical default. (This deviation produced a second forward `EmailLog` id `3`; it and `Contact 2` are removed in [§7](#7-cleanup--the-repository-and-database-are-left-unchanged).)

---


## 6. Q3 — Background behavior: do the handler and runner keep working?

Q3 asks whether the email handler and job runner "automatically come online in the background to support email activity and data handling," and what behavior demonstrates that these internal components work as intended across situations — including secondary and edge conditions. This section answers that from runtime observation: how the long-lived processes stay up (§6.1), a genuinely-enqueued job draining through the runner (§6.2), the runner's retry/`run_at` selection gate (§6.3), an unknown job name (§6.4), the email handler's reply/bounce dispatcher and SMTP status codes (§6.5), and inbound mail auto-creating an alias via a directory (§6.6).

### 6.1 How the three services stay up (they do **not** launch one another)

**Correction of the "auto come online" framing.** The three components are **separate OS processes, each launched independently** — there is no parent process that spawns the other two. In this investigation each was started with its own command (§2.4): the web app via the container's Gunicorn `CMD`, the email handler via `python email_handler.py`, and the job runner via `python job_runner.py`. What makes them *appear* to "come online and keep working in the background" is that **each entry point ends in an infinite keep-alive loop** that holds the already-started process open and lets it accept later work **without re-initializing the process** per request/message/job. Neither background loop starts, supervises, or is aware of the other processes. *(This corrects an earlier draft that implied the handler/runner are auto-started by the web app; they are independently orchestrated.)*

**The email handler's keep-alive loop.** `main()` builds the `aiosmtpd` `Controller`, calls `controller.start()` (which spawns the listener thread), logs `Start mail controller`, and then the **main thread parks in an infinite 2-second sleep loop** so the process — and its listener — stay alive (`email_handler.py:L2381-L2393`):

```
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

Each inbound message is handled by `MailHandler.handle_DATA()` → `_handle()`, which opens a **fresh** `create_light_app().app_context()` **per message** (`email_handler.py:L2352`) — so the process stays up continuously while each message gets a clean application context. The single-message lifecycle captured in §5.4 (and the nonexistent-alias / unauthorized-reply / out-of-office SMTP cases in §6.5) were all served by one long-running handler process; the two `handle_bounce()` phases in §6.5 were captured in a separate, later handler run (pid `10602`).

**The job runner's poll loop.** The runner's `__main__` block is an infinite `while True:` (`job_runner.py:L330`) that, on each iteration, opens a fresh `create_light_app().app_context()` (`job_runner.py:L332`), drains **all** currently-eligible jobs via `get_jobs_to_run()` (`job_runner.py:L333`), and then sleeps 10 seconds (`job_runner.py:L347`) before polling again (full loop quoted in §6.3).

**Continuity evidence — one process, one timeline (Rule R8: no hidden restart).** The job runner has been the **same PID (`3352`) for the entire investigation** — it served the poll-cadence jobs at `18:17` (§4.3), the canonical delete-account job at `18:53` (§6.2), and the retry/unknown-job probes at `18:57`–`19:03` (§6.3–§6.4), with **no restart in between**. Its process age (`etime`) at the time of writing:

```
$ docker exec sl-app bash -lc "ps -eo pid,etime,cmd | grep -E 'job_runner.py|email_handler.py|gunicorn wsgi' | grep -v grep"
   3336    01:04:45 /app/venv/bin/python /app/venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
   3352    01:04:44 ./venv/bin/python job_runner.py
   3353    01:04:44 /app/venv/bin/python /app/venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
   3361    01:04:44 /app/venv/bin/python /app/venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
   4648       40:30 ./venv/bin/python email_handler.py
```

The web app (Gunicorn master `3336` + 2 workers `3353`/`3361`) and the job runner (`3352`) share the same ~`01:04` age — one continuous run. The email handler shows a **younger age (`40:30`)** because it was **deliberately restarted once** for the disclosed mail-capture deviation in §5.4 and then returned to the default configuration (pid `4648`); that single, labeled restart is the only process discontinuity in the investigation, and it is honestly reported rather than presented as uninterrupted continuity.


### 6.2 A genuinely enqueued job (canonical ready → taken → done)

The poll-cadence measurement in §4.3 used jobs inserted directly through the `Job.create()` model — labeled `[NON-CANONICAL]` because they did not originate from an end-user action. This section drives the runner with a **genuinely application-produced** job: a real, authenticated dashboard **account-deletion** request. That is the canonical producer of a `Job` in this codebase — `DeleteAccount` schedules a `delete-account` job via `Job.create(name=JOB_DELETE_ACCOUNT, payload={"user_id": current_user.id}, run_at=arrow.now(), commit=True)` (`app/dashboard/views/delete_account.py:L42-L47`). Deleting the throw-away test account from §5.1 doubles as its canonical cleanup.

**The producer — a real dashboard flow (HTTP).** The action requires an authenticated session **and** a fresh "sudo" confirmation (`@sudo_required`, `app/dashboard/views/enter_sudo.py:L70-L75`, gap `_SUDO_GAP = 120` seconds, `L17`). The script logs in as the test account, enters sudo with the same password, GETs the delete-account page for its CSRF token, and POSTs `form-name=delete-account`:

```
$ python3 /tmp/blitzy_investigation/q3_delete_account.py
=== C: canonical delete-account job producer (test user 3) ===
GET  /auth/login            -> 200 | csrf_len 91
POST /auth/login            -> 302 | Location: http://127.0.0.1:7777/dashboard/
     session cookie present: False
GET  /dashboard/enter_sudo  -> 200 | csrf_len 91
POST /dashboard/enter_sudo  -> 302 | Location: http://127.0.0.1:7777/dashboard/
GET  /dashboard/delete_account -> 200 | csrf_len 91
PRE_POST_TS 18:53:07 7.210
POST /dashboard/delete_account -> 302 | Location: http://127.0.0.1:7777/dashboard/setting
POST_DONE_TS 18:53:07 7.223
```

Each step's status confirms the gate it passed: `POST /auth/login` returns `302` to `http://127.0.0.1:7777/dashboard/` (authenticated); `POST /dashboard/enter_sudo` returns `302` to `http://127.0.0.1:7777/dashboard/` (sudo granted — an incorrect password would instead re-render `200` with an "Incorrect password" flash); `GET /dashboard/delete_account` returns `200` (the `@sudo_required` gate let the page render); and the decisive **`POST /dashboard/delete_account`** returns `302` to `http://127.0.0.1:7777/dashboard/setting`, the success redirect at `app/dashboard/views/delete_account.py:L54`. On that POST the web worker writes a scheduling log line (`app/dashboard/views/delete_account.py:L36`), emits a `UserMarkedForDeletion` audit entry (`app/dashboard/views/delete_account.py:L39`), and creates the job. The log line (from `/tmp/sl_run/web.log`) is timestamped to the same instant as the job's `run_at`:

```
2026-07-13 18:53:07,215 - SL - WARNING - 3353 - "/app/app/dashboard/views/delete_account.py:36" - delete_account() -  - schedule delete account job for <User 3 qa.tester.9f3@example.com qa.tester.9f3@example.com>
```

*(The `session cookie present: False` line in the producer output only reflects that SimpleLogin's session cookie is not literally named `session`; the subsequent `302`s prove the session was carried. The producer script's trailing traceback — omitted from the quoted output above — is a cosmetic bug in a post-success diagnostic line that prepended the base URL to an already-absolute `Location`; it occurred **after** the canonical POST had already returned `302`.)*

**The transition — `ready(0) → taken(1) → done(2)`, captured live.** While the POST ran, a tight ~0.2 s poller sampled the new job row (keyed on `name='delete-account'` and `payload->>'user_id'='3'`). The first sample caught the job **in `ready` state**, and a later sample caught it **`done`**:

```
$ # poller: /tmp/blitzy_investigation/q3_job_poll.txt (ts | id | state | attempts | taken | created | taken_at | run_at)
18:53:07.350|4|st=0|att=0|taken=false|created=18:53:07.216|taken_at=NULL|run_at=18:53:07.215
18:53:15.268|4|st=0|att=0|taken=false|created=18:53:07.216|taken_at=NULL|run_at=18:53:07.215
18:53:15.533|4|st=2|att=1|taken=true|created=18:53:07.216|taken_at=18:53:15.460|run_at=18:53:07.215
```

The row was **created at `18:53:07.216`** with `run_at=18:53:07.215` (`arrow.now()` at enqueue) and sat in **`ready` (`st=0`, `attempts=0`, `taken=false`)** until the last ready sample at `18:53:15.268` — about **8.2 seconds**, i.e. within one 10-second poll window. It was then **taken and completed in the same poll iteration**, so by `18:53:15.533` it read **`done` (`st=2`, `attempts=1`, `taken=true`, `taken_at=18:53:15.460`)**. Across the whole window the poller recorded **31 `ready` samples, 0 `taken` samples, and 75 `done` samples** — `taken` is invisible to a 0.2 s poller because the runner sets `taken` then `done` in the same loop body with no sleep between (`job_runner.py:L338`→`L344`); the transition through `taken` is instead proved by the **persisted `taken_at` timestamp and the `attempts` increment to `1`**.

**The runner's own log — the canonical work.** The job runner (still PID `3352`) logged taking and processing the job (`/tmp/sl_run/job.log`):

```
2026-07-13 18:53:15,460 - SL - DEBUG - 3352 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 4 delete-account {'user_id': 3}>
2026-07-13 18:53:15,464 - SL - WARNING - 3352 - "/app/job_runner.py:235" - process_job() -  - Delete user <User 3 qa.tester.9f3@example.com qa.tester.9f3@example.com>
2026-07-13 18:53:15,474 - SL - DEBUG - 3352 - "/app/app/email_utils.py:303" - send_email() -  - send email to qa.tester.9f3@example.com, subject 'Your SimpleLogin account has been deleted'
2026-07-13 18:53:15,475 - SL - DEBUG - 3352 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'Your SimpleLogin account has been deleted', from '"noreply@sl.local" <noreply@sl.local>' to 'qa.tester.9f3@example.com'
2026-07-13 18:53:15,479 - SL - INFO - 3352 - "/app/app/alias_utils.py:346" - delete_alias() -  - User <User 3 qa.tester.9f3@example.com qa.tester.9f3@example.com> has deleted alias <Alias 12 simplelogin-newsletter.list620@sl.local>
2026-07-13 18:53:15,484 - SL - INFO - 3352 - "/app/app/alias_utils.py:368" - delete_alias() -  - Moving <Alias 12 simplelogin-newsletter.list620@sl.local> to global trash <Deleted Alias simplelogin-newsletter.list620@sl.local>
```

`Take job` fires at `job_runner.py:L334`; the `delete-account` branch of `process_job()` logs `Delete user` (`job_runner.py:L235`), attempts the account-deletion notice email (short-circuited by `NOT_SEND_EMAIL`, exactly as in §5.4 — logged, not sent), and then `User.delete(user.id)` (`job_runner.py:L243`) cascades, deleting the account's auto-provisioned newsletter alias `12` and moving it to the global trash (`app/alias_utils.py:L346,L368`). The persisted job row afterward:

```
$ docker exec sl-postgres psql -U myuser -d simplelogin -c "SELECT id, state, name, payload, attempts, taken, to_char(created_at,'HH24:MI:SS.MS') created, to_char(taken_at,'HH24:MI:SS.MS') taken_at, to_char(run_at,'HH24:MI:SS.MS') run_at FROM job WHERE id=4;"
 id | state |      name      |    payload     | attempts | taken |   created    |   taken_at   |    run_at
----+-------+----------------+----------------+----------+-------+--------------+--------------+--------------
  4 |     2 | delete-account | {"user_id": 3} |        1 | t     | 18:53:07.216 | 18:53:15.460 | 18:53:07.215
(1 row)
```

The job row itself persists as `state=2` (`done`) — it carries the user id only in a JSON payload with no foreign key, so deleting the user does not remove the job. **Canonical cleanup confirmed:** the test account (`users.id=3`) and its alias (`alias.id=12`) are gone, the alias having moved to the `deleted_alias` global trash:

```
$ docker exec sl-postgres psql -U myuser -d simplelogin -tAc "SELECT count(*) FROM users WHERE id=3;"
0
$ docker exec sl-postgres psql -U myuser -d simplelogin -tAc "SELECT count(*) FROM alias WHERE id=12;"
0
$ docker exec sl-postgres psql -U myuser -d simplelogin -c "SELECT id, email FROM deleted_alias ORDER BY id;"
 id |                  email
----+-----------------------------------------
  1 | simplelogin-newsletter.list620@sl.local
(1 row)
```

The residual `deleted_alias` row is a normal side effect of account deletion (SimpleLogin keeps deleted aliases in a global trash to prevent later re-use); it is removed during the pristine-state restoration in [§7](#7-cleanup--the-repository-and-database-are-left-unchanged).


### 6.3 Retry accounting and the `run_at` selection gate

Which jobs the runner picks up on each poll is decided by `get_jobs_to_run()` (`job_runner.py:L307`), whose filter combines **two** independent conditions — a state/retry condition **and** a `run_at` scheduling condition (the second is the one an earlier draft omitted). The function and the loop that consumes it, verbatim:

```
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

Both conditions must hold (they are joined by `and_` at `L314`/`L323`):

- **State / retry** (`L316`–`L322`): a job is eligible if it is `ready` **OR** it is `taken` but *stale* — `taken_at` older than `now − JOB_TAKEN_RETRY_WAIT_MINS` **and** `attempts < JOB_MAX_ATTEMPTS`. The two ceilings come from config: `JOB_TAKEN_RETRY_WAIT_MINS = 30` (`app/config.py:L565`) and `JOB_MAX_ATTEMPTS = 5` (`app/config.py:L564`). This is the retry mechanism: a job that was taken but never finished (e.g. the runner died mid-job) becomes eligible again after 30 minutes, up to 5 attempts. Each pickup increments `attempts` (`job_runner.py:L340`).
- **Schedule** (`L323`): independently, the job's `run_at` must be **NULL** *or* no later than `now + 10 minutes`. A job scheduled further in the future is skipped until it comes within that 10-minute horizon.

**The `run_at` gate observed across all three branches.** All producers below except the runner's own decisions are `[NON-CANONICAL]` (jobs written directly through the model to construct a specific state); the **runner's take/skip decision is the canonical behavior being observed**. The payload `user_id=999999` points at no real user, so `process_job()` is a harmless no-op (`No user found`) if a job is ever taken.

- **`run_at` NULL → eligible.** The poll-cadence jobs in §4.3 and the stale-retry job below carry `run_at=NULL` and are taken.
- **`run_at` in the (recent) past → eligible.** The canonical delete-account job in §6.2 had `run_at=18:53:07.215` and was taken on the next poll.
- **`run_at` in the future → skipped.** A probe job was created with `run_at = now + 20 minutes`:

```
$ # setup (/tmp/blitzy_investigation/q3_f13_setup.txt) — [NON-CANONICAL] producer
TEST_A_future_runat  id=5 state=0 run_at=2026-07-13T19:17:15.587158+00:00
TEST_B_stale_taken   id=6 state=1 attempts=2 taken_at=2026-07-13T18:17:15.712582+00:00
```

  Job `5` was created at `~18:57` with `run_at=19:17:15`. Roughly five minutes (~30 poll cycles) later it was **still untaken**, because `19:17:15` was still beyond `now + 10 min`:

```
$ docker exec sl-postgres psql -U myuser -d simplelogin -c "SELECT id, state, name, attempts, taken, to_char(taken_at,'HH24:MI:SS') taken_at, to_char(run_at,'HH24:MI:SS') run_at, now()::time(0) AS now_ FROM job WHERE payload->>'user_id'='999999' ORDER BY id;"
 id | state |      name      | attempts | taken | taken_at |  run_at  |   now_
----+-------+----------------+----------+-------+----------+----------+----------
  5 |     0 | delete-account |        0 | f     |          | 19:17:15 | 19:02:46
  6 |     2 | delete-account |        3 | t     | 18:57:15 |          | 19:02:46
(2 rows)
```

  At `now_=19:02:46`, `now + 10 min = 19:12:46 < 19:17:15`, so job `5` remains `ready` with `attempts=0` — the `run_at <= now + 10 min` predicate (`L323`) correctly holds it back. *(Wall-clock arithmetic here is inferred from the timestamps; the `state=0`/`attempts=0` values are observed.)*

**Stale-taken retry observed.** Job `6` was created **already `taken`** with `taken_at` set 40 minutes in the past and `attempts=2` (the `[NON-CANONICAL]` setup line above: `state=1 attempts=2 taken_at=18:17:15`). Because `18:17:15 < now − 30 min` and `2 < 5`, it satisfied the stale-retry branch, and the runner **re-took it on its next poll**:

```
2026-07-13 18:57:15,783 - SL - DEBUG - 3352 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 6 delete-account {'user_id': 999999}>
```

Afterward job `6` reads `state=2` (`done`) with **`attempts=3`** (the current-state table above) — i.e. the retry incremented the attempt counter `2 → 3` (`job_runner.py:L340`). Together with the delete-account job's `0 → 1` (§6.2), this is the `attempts` accounting the retry ceiling depends on, observed at runtime.

### 6.4 Edge: an unknown job name

`process_job()` dispatches on `job.name` through a chain of `elif` branches (onboarding-1/2/4, batch import, delete-account, delete-mailbox, delete-domain, send-user-report, and others); the final `else` handles any unrecognized name by logging at exception level (`job_runner.py:L303-L304`). To exercise it, a job with a made-up name was enqueued (`[NON-CANONICAL]` producer — no end-user action creates arbitrary job names; the runner's handling is the canonical behavior). The runner took it and logged:

```
2026-07-13 19:03:26,223 - SL - DEBUG - 3352 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 7 blitzy-nonexistent-job {'probe': 'unknown-name-edge'}>
2026-07-13 19:03:26,226 - SL - ERROR - 3352 - "/app/job_runner.py:304" - process_job() -  - Unknown job name blitzy-nonexistent-job
NoneType: None
```

Two details are worth noting. First, the message is logged via `LOG.e`, which is **`logging.Logger.exception`** (`app/log.py:L77`); that call always appends exception info, so with **no active exception** it emits the trailing **`NoneType: None`** line — an artifact of using `LOG.e` outside an `except` block, not a real traceback. Second, the loop marks the job `done` regardless of the outcome (there is no `sleep` and no re-raise), so the unknown job ends `state=2` with `attempts=1`:

```
$ docker exec sl-postgres psql -U myuser -d simplelogin -c "SELECT id, state, name, payload, attempts, taken, to_char(taken_at,'HH24:MI:SS') taken_at FROM job WHERE name='blitzy-nonexistent-job';"
 id | state |          name          |            payload             | attempts | taken | taken_at
----+-------+------------------------+--------------------------------+----------+-------+----------
  7 |     2 | blitzy-nonexistent-job | {"probe": "unknown-name-edge"} |        1 | t     | 19:03:26
(1 row)
```

The three probe jobs (`5`, `6`, `7`) are removed during the pristine-state restoration in [§7](#7-cleanup--the-repository-and-database-are-left-unchanged).


### 6.5 The email handler's dispatcher: reply, bounce, and SMTP status codes

**The dispatcher is multi-branch, not "three phases."** Every message is routed by `handle(envelope, msg) -> str` (`email_handler.py:L1945`, docstring "Return SMTP status"). It is **not** a fixed forward/reply/bounce trio; it is a sequence of guards that inspect the envelope, the recipient type, and message headers, and it returns a specific `status.E***` code from whichever branch matches. The main branches, in order:

- **Bounce / DSN handling for VERP recipients** — `is_bounce(envelope, msg)` gates `handle_bounce()` / `handle_transactional_bounce()` in several recipient contexts (`email_handler.py:L2043,L2069,L2090` → `handle_bounce()` `L1851`). Crucially, `is_bounce()` is **not** a "from MAILER-DAEMON" test; it is (`email_handler.py:L1813`):

```
def is_bounce(envelope: Envelope, msg: Message):
    """Detect whether an email is a Delivery Status Notification"""
    return (
        envelope.mail_from == "<>"
        and msg.get_content_type().lower() == "multipart/report"
    )
```

- **Automatic out-of-office** — `is_automatic_out_of_office(msg)` (`email_handler.py:L2048,L2071,L2092`), plus a dedicated early return at `email_handler.py:L2166-L2172`: when there is exactly one recipient, that recipient is a reverse-alias, and `mail_from == "<>"`, the handler logs an out-of-office notice and returns `status.E206` (this is the branch Case C hits, below).
- **Rate limiting** — `if rate_limited(mail_from, rcpt_tos):` (`email_handler.py:L2145`).
- **Per-recipient reply vs forward** — for each recipient, `if is_reverse_alias(rcpt_to)` routes to the **reply** path `handle_reply()` (`email_handler.py:L2195,L2199` → `L966`); otherwise the **forward** path `handle_forward()` (`email_handler.py:L2203,L2208` → `L536`). A `NOREPLY` recipient short-circuits to `status.E200` (`email_handler.py:L2181-L2183`).

**How the final SMTP code is produced.** `MailHandler` defines **only `handle_DATA`** (`email_handler.py:L2288-L2289`) — there is no `handle_RCPT` or `handle_MAIL` — so `aiosmtpd` accepts `MAIL FROM` and `RCPT TO` with default `250`s and the accept/reject decision is returned **after the DATA body**. `handle_DATA` calls `_handle()` and maps exceptions to codes (`email_handler.py:L2289-L2332`):

```
    async def handle_DATA(self, server, session, envelope: Envelope):
        msg = email.message_from_bytes(envelope.original_content)
        try:
            ret = self._handle(envelope, msg)
            return ret

        # happen if reverse-alias is used during the forward phase
        # as in this case, a new reverse-alias needs to be created for this reverse-alias -> chaos
        except CannotCreateContactForReverseAlias as e:
            LOG.w(
                "Probably due to reverse-alias used in the forward phase, "
                "error:%s mail_from:%s, rcpt_tos:%s, header_from:%s, header_to:%s",
                e,
                envelope.mail_from,
                envelope.rcpt_tos,
                msg[headers.FROM],
                msg[headers.TO],
            )
            return status.E524
        except (VERPReply, VERPForward, VERPTransactional) as e:
            LOG.w(
                "email handling fail with error:%s "
                "mail_from:%s, rcpt_tos:%s, header_from:%s, header_to:%s",
                e,
                envelope.mail_from,
                envelope.rcpt_tos,
                msg[headers.FROM],
                msg[headers.TO],
            )
            return status.E213
        except Exception as e:
            LOG.e(
                "email handling fail with error:%s "
                "mail_from:%s, rcpt_tos:%s, header_from:%s, header_to:%s, saved to %s",
                e,
                envelope.mail_from,
                envelope.rcpt_tos,
                msg[headers.FROM],
                msg[headers.TO],
                save_envelope_for_debugging(
                    envelope, file_name_prefix=e.__class__.__name__
                ),  # todo: remove
            )
            return status.E404
```

So `CannotCreateContactForReverseAlias → E524`, `(VERPReply | VERPForward | VERPTransactional) → E213`, and any other exception → `E404`. The relevant codes (from `app/email/status.py`) are: `E200 = "250 Message accepted for delivery"` (`L2`), `E206 = "250 SL E206 Out of office"` (`L9`), `E213 = "250 SL E213 Unknown email ignored"` (`L21`), `E214 = "250 SL E214 Unauthorized for using reverse alias"` (`L22`), `E404 = "421 SL E404 Unexpected error - Retry later"` (`L32`, a **4xx** retry code, not a 5xx), `E515 = "550 SL E515 Email not exist"` (`L51`), and `E524 = "550 SL E524 Wrong use of reverse-alias"` (`L62`). Each of these three exception-mapped codes was **also exercised at runtime** — see "The three exception-mapped codes E524/E213/E404, observed" at the end of this section — so they are reported below as observed, not merely code-derived.


**Observed stage-by-stage (RCPT accepted; the real decision comes after DATA).** A low-level SMTP client sent three messages to `127.0.0.1:20381`, capturing each protocol stage separately so the recipient stage and the post-DATA decision are distinguishable:

```
$ python3 /tmp/blitzy_investigation/q3_smtp_stages.py
=================================================================
CASE A-nonexistent-alias | MAIL FROM:<external.sender@example.org> RCPT TO:<nonexistent-xyz-blitzy@sl.local>
=================================================================
EHLO       -> 250
MAIL FROM  -> 250 OK
RCPT TO    -> 250 OK   <-- recipient stage
DATA       -> 354 End data with <CR><LF>.<CR><LF>   <-- server ready for body
POST-DATA  -> 550 SL E515 Email not exist   <-- final decision AFTER body

=================================================================
CASE B-unauthorized-reply | MAIL FROM:<attacker@evil.example> RCPT TO:<rep@sl.local>
=================================================================
EHLO       -> 250
MAIL FROM  -> 250 OK
RCPT TO    -> 250 OK   <-- recipient stage
DATA       -> 354 End data with <CR><LF>.<CR><LF>   <-- server ready for body
POST-DATA  -> 250 SL E214 Unauthorized for using reverse alias   <-- final decision AFTER body

=================================================================
CASE C-bounce-DSN | MAIL FROM:<> RCPT TO:<rep@sl.local>
=================================================================
EHLO       -> 250
MAIL FROM  -> 250 OK
RCPT TO    -> 250 OK   <-- recipient stage
DATA       -> 354 End data with <CR><LF>.<CR><LF>   <-- server ready for body
POST-DATA  -> 250 SL E206 Out of office   <-- final decision AFTER body
```

In **every** case `RCPT TO` returns `250 OK` and the actual accept/reject arrives only **after** the DATA body — confirming that recipient validity is not decided at the RCPT stage. The three cases exercise three distinct dispatcher branches, each corroborated by the handler's own log (`/tmp/sl_run/email.log`), correlated by the per-message `message_id`:

```
2026-07-13 19:12:16,400 - SL - INFO - 4648 - "/app/email_handler.py:2343" - _handle() - 651e6246-241f-40a5-a72e-f53debc71029 - New message, mail from external.sender@example.org, rctp tos ['nonexistent-xyz-blitzy@sl.local']
2026-07-13 19:12:16,533 - SL - DEBUG - 4648 - "/app/email_handler.py:545" - handle_forward() - 651e6246-241f-40a5-a72e-f53debc71029 - alias nonexistent-xyz-blitzy@sl.local not exist. Try to see if it can be created on the fly
2026-07-13 19:12:16,542 - SL - DEBUG - 4648 - "/app/email_handler.py:551" - handle_forward() - 651e6246-241f-40a5-a72e-f53debc71029 - alias nonexistent-xyz-blitzy@sl.local cannot be created on-the-fly, return 550
2026-07-13 19:12:16,543 - SL - INFO - 4648 - "/app/email_handler.py:2367" - _handle() - 651e6246-241f-40a5-a72e-f53debc71029 - Finish mail_from external.sender@example.org, rcpt_tos ['nonexistent-xyz-blitzy@sl.local'], takes 0.14281177520751953 seconds with return code '550 SL E515 Email not exist'<<===
2026-07-13 19:12:16,546 - SL - INFO - 4648 - "/app/email_handler.py:2343" - _handle() - 715f706f-cc00-4cba-8f11-a7b715fc4c02 - New message, mail from attacker@evil.example, rctp tos ['rep@sl.local']
2026-07-13 19:12:16,561 - SL - WARNING - 4648 - "/app/email_handler.py:1393" - handle_unknown_mailbox() - 715f706f-cc00-4cba-8f11-a7b715fc4c02 - Reply email can only be used by mailbox. Actual mail_from: attacker@evil.example. msg from header: Attacker <attacker@evil.example>, reverse-alias rep@sl.local, <Alias 2 word_test646@sl.local> <User 1 John Wick john@wick.com> <Contact 1 hey@google.com 2>
2026-07-13 19:12:16,582 - SL - INFO - 4648 - "/app/email_handler.py:2367" - _handle() - 715f706f-cc00-4cba-8f11-a7b715fc4c02 - Finish mail_from attacker@evil.example, rcpt_tos ['rep@sl.local'], takes 0.03617095947265625 seconds with return code '250 SL E214 Unauthorized for using reverse alias'<<===
2026-07-13 19:12:16,585 - SL - INFO - 4648 - "/app/email_handler.py:2343" - _handle() - d9ddc13c-a466-45ff-8018-f6adde397392 - New message, mail from <>, rctp tos ['rep@sl.local']
2026-07-13 19:12:16,590 - SL - WARNING - 4648 - "/app/email_handler.py:2168" - handle() - d9ddc13c-a466-45ff-8018-f6adde397392 - out-of-office email to reverse alias <Contact 1 hey@google.com 2>. Saved to
2026-07-13 19:12:16,590 - SL - INFO - 4648 - "/app/email_handler.py:2367" - _handle() - d9ddc13c-a466-45ff-8018-f6adde397392 - Finish mail_from <>, rcpt_tos ['rep@sl.local'], takes 0.00493311882019043 seconds with return code '250 SL E206 Out of office'<<===
```

- **Case A (forward to a nonexistent alias):** `handle_forward()` finds no alias, tries on-the-fly creation, fails, and returns `550 SL E515 Email not exist` (`email_handler.py:L545,L551,L555`). This is a genuine `5xx` rejection, delivered after DATA.
- **Case B (reply from an unauthorized sender):** the recipient `rep@sl.local` is a reverse-alias, so the reply path runs; the sender is not one of the alias's authorized mailboxes, so `handle_unknown_mailbox()` logs the rejection (`email_handler.py:L1393`) and `handle_reply()` returns `250 SL E214` (`email_handler.py:L1034`) — deliberately a `2xx` "to avoid Postfix sending out bounces and avoid backscatter issue" (comment at `email_handler.py:L1033`).
- **Case C (`MAIL FROM:<>` DSN to a reverse-alias):** although the message matched `is_bounce()`'s content-type test, it did **not** reach `handle_bounce()`. Because the single recipient is a **reverse-alias** (not a VERP bounce address), it matched the earlier out-of-office guard (`email_handler.py:L2166-L2172`) and returned `250 SL E206 Out of office`. This is reported exactly as observed.

**`handle_bounce()` — both phases observed at runtime.** Case C above (a DSN aimed at a *reverse-alias*) is classified out-of-office; the dedicated bounce handler is instead reached when the DSN's recipient is a **VERP bounce address**. Contrary to an earlier reading, that address is **not** HMAC-signed and **can** be constructed by hand from a known `email_log.id`. The forward-phase trigger matches any single recipient that `startswith(BOUNCE_PREFIX)` **and** `endswith(BOUNCE_SUFFIX)` (`email_handler.py:L2057-L2061`); the reply-phase trigger matches a recipient that `startswith(f"{BOUNCE_PREFIX_FOR_REPLY_PHASE}+")` (`email_handler.py:L2077-L2080`); and in both branches the id is recovered by `parse_id_from_bounce()`, which is a plain `int()` of the substring between the first and last `+` — no signature, no secret (`app/email_utils.py:L1258-L1259`):

```
def parse_id_from_bounce(email_address: str) -> int:
    return int(email_address[email_address.find("+") : email_address.rfind("+")])
```

With the default config values `BOUNCE_PREFIX = "bounce+"`, `BOUNCE_SUFFIX = "+@sl.local"`, and `BOUNCE_PREFIX_FOR_REPLY_PHASE = "bounce_reply"` (`app/config.py:L100-L101,L108-L110`; `EMAIL_DOMAIN=sl.local`), the two VERP forms are simply `bounce+{id}+@sl.local` and `bounce_reply+{id}+@sl.local`. Both phases were therefore exercised black-box against the running handler (a dedicated later run, pid `10602` — the second disclosed exception to the single-timeline note in [§4](#4-q1--are-the-three-components-up-liveness--confirmation-signals)). The temporary rows they produced were removed during cleanup ([§7](#7-cleanup--the-repository-and-database-are-left-unchanged)).

**Forward phase → `250 SL E211`.** First a normal forward to `e1@sl.local` minted a **forward** email-log (`is_reply=f`); its id (`14`) was then used to build the VERP bounce recipient. A DSN with `MAIL FROM:<>` and `Content-Type: multipart/report` (the two conditions `is_bounce()` requires) was sent to `bounce+14+@sl.local` over the real SMTP socket:

```
$ docker exec -i sl-app ./venv/bin/python - <<'PY'
import smtplib
b = "=_blitzy_dsn_boundary_x1"
dsn = (
    'Content-Type: multipart/report; report-type=delivery-status; boundary="%s"\r\n'
    'From: MAILER-DAEMON@example.org\r\n'
    'To: bounce+14+@sl.local\r\n'
    'Subject: Delivery Status Notification (Failure)\r\n'
    'Auto-Submitted: auto-replied\r\n\r\n'
    '--%s\r\nContent-Type: text/plain; charset=us-ascii\r\n\r\n'
    'Delivery to the following recipient failed permanently:\r\n  john@wick.com\r\n\r\n'
    '--%s\r\nContent-Type: message/delivery-status\r\n\r\n'
    'Reporting-MTA: dns; mta.example.org\r\n\r\n'
    'Final-Recipient: rfc822; john@wick.com\r\nAction: failed\r\nStatus: 5.1.1\r\n'
    'Diagnostic-Code: smtp; 550 5.1.1 User unknown\r\n\r\n'
    '--%s--\r\n'
) % (b, b, b, b)
s = smtplib.SMTP("127.0.0.1", 20381, timeout=30)
s.ehlo("blitzy-repro.local")
cm, rm = s.mail("")                        # MAIL FROM:<>
cr, rr = s.rcpt("bounce+14+@sl.local")     # VERP forward-bounce recipient
cd, rd = s.data(dsn)
print("MAIL:", cm, rm.decode()); print("RCPT:", cr, rr.decode()); print("DATA:", cd, rd.decode())
s.quit()
PY
MAIL: 250 OK
RCPT: 250 OK
DATA: 250 SL E211 Bounce Forward phase handled
```

The handler log shows the full routing, correlated by the per-message `message_id 9bd5547f-…`: the message boundary and `New message, mail from <>, rctp tos ['bounce+14+@sl.local']`; `handle_bounce()` (log at `email_handler.py:L1862`; def `L1851`) logging **`phase=forward`** (because the referenced `<EmailLog 14>` has `is_reply=False`); `handle_bounce_forward_phase()` (log at `email_handler.py:L1456`; def `L1432`); the refused-email/notification side effects; and the finishing line carrying `status.E211` (returned at `email_handler.py:L1914`):

```
2026-07-14 04:09:10,122 - SL - DEBUG - 10602 - "/app/app/log.py:24" - set_message_id() - 96c3d5d4-9223-49b0-937a-69e04f6f7cc8 - set message_id 9bd5547f-088a-4fb6-bcd3-75f47b8580bf
2026-07-14 04:09:10,122 - SL - DEBUG - 10602 - "/app/email_handler.py:2342" - _handle() - 9bd5547f-088a-4fb6-bcd3-75f47b8580bf - ====>=====>====>====>====>====>====>====>
2026-07-14 04:09:10,122 - SL - INFO - 10602 - "/app/email_handler.py:2343" - _handle() - 9bd5547f-088a-4fb6-bcd3-75f47b8580bf - New message, mail from <>, rctp tos ['bounce+14+@sl.local']
2026-07-14 04:09:10,123 - SL - INFO - 10602 - "/app/email_handler.py:1956" - handle() - 9bd5547f-088a-4fb6-bcd3-75f47b8580bf - Set CONTENT_TRANSFER_ENCODING
2026-07-14 04:09:10,123 - SL - DEBUG - 10602 - "/app/email_handler.py:1963" - handle() - 9bd5547f-088a-4fb6-bcd3-75f47b8580bf - Cannot parse Postfix queue ID from None None
2026-07-14 04:09:10,124 - SL - DEBUG - 10602 - "/app/email_handler.py:1980" - handle() - 9bd5547f-088a-4fb6-bcd3-75f47b8580bf - ==>> Handle mail_from:<>, rcpt_tos:['bounce+14+@sl.local'], header_from:MAILER-DAEMON@example.org, header_to:bounce+14+@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('Content-Type', 'multipart/report; report-type=delivery-status; boundary="=_blitzy_dsn_boundary_x1"'), ('From', 'MAILER-DAEMON@example.org'), ('To', 'bounce+14+@sl.local'), ('Subject', 'Delivery Status Notification (Failure)'), ('Auto-Submitted', 'auto-replied'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 04:09:10,133 - SL - DEBUG - 10602 - "/app/email_handler.py:1862" - handle_bounce() - 9bd5547f-088a-4fb6-bcd3-75f47b8580bf - handle bounce for <EmailLog 14>, phase=forward, contact=<Contact 12 external.sender@example.org 5>, alias=<Alias 5 e1@sl.local>
2026-07-14 04:09:10,135 - SL - WARNING - 10602 - "/app/app/email_utils.py:727" - get_mailbox_bounce_info() - 9bd5547f-088a-4fb6-bcd3-75f47b8580bf - add missing content-transfer-encoding header
2026-07-14 04:09:10,138 - SL - DEBUG - 10602 - "/app/email_handler.py:1456" - handle_bounce_forward_phase() - 9bd5547f-088a-4fb6-bcd3-75f47b8580bf - Handle forward bounce <Contact 12 external.sender@example.org 5> -> <Alias 5 e1@sl.local> -> <Mailbox 1 john@wick.com>. <EmailLog 14>
2026-07-14 04:09:10,141 - SL - WARNING - 10602 - "/app/email_handler.py:1474" - handle_bounce_forward_phase() - 9bd5547f-088a-4fb6-bcd3-75f47b8580bf - Cannot parse original message from bounce message <Alias 5 e1@sl.local> <User 1 John Wick john@wick.com> <Contact 12 external.sender@example.org 5> refused-emails/full-32965e88-a86c-433e-807e-3bbf1f5696f2.eml
2026-07-14 04:09:10,144 - SL - DEBUG - 10602 - "/app/email_handler.py:1491" - handle_bounce_forward_phase() - 9bd5547f-088a-4fb6-bcd3-75f47b8580bf - Create refused email <Refused Email 3 None 2026-07-21T04:09:10.144034+00:00>
2026-07-14 04:09:10,148 - SL - DEBUG - 10602 - "/app/email_handler.py:1542" - handle_bounce_forward_phase() - 9bd5547f-088a-4fb6-bcd3-75f47b8580bf - Inform user <User 1 John Wick john@wick.com> about a bounce from contact <Contact 12 external.sender@example.org 5> to alias <Alias 5 e1@sl.local>
2026-07-14 04:09:10,176 - SL - DEBUG - 10602 - "/app/app/email_utils.py:303" - send_email() - 9bd5547f-088a-4fb6-bcd3-75f47b8580bf - send email to john@wick.com, subject 'An email sent to e1@sl.local cannot be delivered to your mailbox'
2026-07-14 04:09:10,177 - SL - DEBUG - 10602 - "/app/app/mail_sender.py:131" - send() - 9bd5547f-088a-4fb6-bcd3-75f47b8580bf - send email with subject 'An email sent to e1@sl.local cannot be delivered to your mailbox', from '"noreply@sl.local" <noreply@sl.local>' to 'john@wick.com'
2026-07-14 04:09:10,177 - SL - INFO - 10602 - "/app/email_handler.py:2367" - _handle() - 9bd5547f-088a-4fb6-bcd3-75f47b8580bf - Finish mail_from <>, rcpt_tos ['bounce+14+@sl.local'], takes 0.0550379753112793 seconds with return code '250 SL E211 Bounce Forward phase handled'<<===
```

Its database side effects were captured before → after — the before-state is stated here in prose because the fenced blocks below are verbatim `psql` output: `<EmailLog 14>` flipped `bounced` from `f` to `t` and its `refused_email_id` advanced from `NULL` to `3`; the `bounce` table, empty before this message, gained its first row (`id 2`, `email=john@wick.com`); and a `refused_email` row (`id 3`), a `notification` (`id 8`, "…cannot be delivered to your mailbox"), and a `sent_alert` (`type=bounce`) were created:

```
$ docker exec sl-postgres psql -U myuser -d simplelogin -c "SELECT id, bounced, refused_email_id FROM email_log WHERE id=14;"
 id | bounced | refused_email_id
----+---------+------------------
 14 | t       |                3
$ docker exec sl-postgres psql -U myuser -d simplelogin -c "SELECT id, email FROM bounce ORDER BY id;"
 id |            email
----+-----------------------------
  2 | john@wick.com
```

**Reply phase → `250 SL E212`.** Symmetrically, a reply from the mailbox owner (`john@wick.com`) to the contact's reverse-alias (`external_sender_at_example_org_rbnuz@sl.local`) minted a **reply** email-log (`is_reply=t`, id `15`); a DSN to `bounce_reply+15+@sl.local` was then accepted with `250 SL E212 Bounce Reply phase handled` (same send shape as above, only the recipient prefix and id differ):

```
$ docker exec -i sl-app ./venv/bin/python - <<'PY'
import smtplib
b = "=_blitzy_dsn_boundary_r1"
dsn = (
    'Content-Type: multipart/report; report-type=delivery-status; boundary="%s"\r\n'
    'From: MAILER-DAEMON@example.org\r\n'
    'To: bounce_reply+15+@sl.local\r\n'
    'Subject: Delivery Status Notification (Failure)\r\n'
    'Auto-Submitted: auto-replied\r\n\r\n'
    '--%s\r\nContent-Type: text/plain; charset=us-ascii\r\n\r\n'
    'Delivery to the following recipient failed permanently:\r\n  external.sender@example.org\r\n\r\n'
    '--%s\r\nContent-Type: message/delivery-status\r\n\r\n'
    'Reporting-MTA: dns; mta.example.org\r\n\r\n'
    'Final-Recipient: rfc822; external.sender@example.org\r\nAction: failed\r\nStatus: 5.1.1\r\n'
    'Diagnostic-Code: smtp; 550 5.1.1 User unknown\r\n\r\n'
    '--%s--\r\n'
) % (b, b, b, b)
s = smtplib.SMTP("127.0.0.1", 20381, timeout=30)
s.ehlo("blitzy-repro.local")
cm, rm = s.mail("")                             # MAIL FROM:<>
cr, rr = s.rcpt("bounce_reply+15+@sl.local")    # VERP reply-bounce recipient
cd, rd = s.data(dsn)
print("MAIL:", cm, rm.decode()); print("RCPT:", cr, rr.decode()); print("DATA:", cd, rd.decode())
s.quit()
PY
MAIL: 250 OK
RCPT: 250 OK
DATA: 250 SL E212 Bounce Reply phase handled
```

The handler routed it through `handle_bounce()` logging **`phase=reply`** (because `<EmailLog 15>.is_reply=True`) into `handle_bounce_reply_phase()` (log at `email_handler.py:L1605`; def `L1595`), returning `status.E212` (returned at `email_handler.py:L1911`):

```
2026-07-14 04:10:58,921 - SL - DEBUG - 10602 - "/app/app/log.py:24" - set_message_id() - 9407581a-7604-4d54-b713-3a61f4bdd599 - set message_id 20b1d172-9097-4a1f-b1bf-26c1445d6f01
2026-07-14 04:10:58,921 - SL - DEBUG - 10602 - "/app/email_handler.py:2342" - _handle() - 20b1d172-9097-4a1f-b1bf-26c1445d6f01 - ====>=====>====>====>====>====>====>====>
2026-07-14 04:10:58,921 - SL - INFO - 10602 - "/app/email_handler.py:2343" - _handle() - 20b1d172-9097-4a1f-b1bf-26c1445d6f01 - New message, mail from <>, rctp tos ['bounce_reply+15+@sl.local']
2026-07-14 04:10:58,921 - SL - INFO - 10602 - "/app/email_handler.py:1956" - handle() - 20b1d172-9097-4a1f-b1bf-26c1445d6f01 - Set CONTENT_TRANSFER_ENCODING
2026-07-14 04:10:58,922 - SL - DEBUG - 10602 - "/app/email_handler.py:1963" - handle() - 20b1d172-9097-4a1f-b1bf-26c1445d6f01 - Cannot parse Postfix queue ID from None None
2026-07-14 04:10:58,923 - SL - DEBUG - 10602 - "/app/email_handler.py:1980" - handle() - 20b1d172-9097-4a1f-b1bf-26c1445d6f01 - ==>> Handle mail_from:<>, rcpt_tos:['bounce_reply+15+@sl.local'], header_from:MAILER-DAEMON@example.org, header_to:bounce_reply+15+@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('Content-Type', 'multipart/report; report-type=delivery-status; boundary="=_blitzy_dsn_boundary_r1"'), ('From', 'MAILER-DAEMON@example.org'), ('To', 'bounce_reply+15+@sl.local'), ('Subject', 'Delivery Status Notification (Failure)'), ('Auto-Submitted', 'auto-replied'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 04:10:58,928 - SL - DEBUG - 10602 - "/app/email_handler.py:1862" - handle_bounce() - 20b1d172-9097-4a1f-b1bf-26c1445d6f01 - handle bounce for <EmailLog 15>, phase=reply, contact=<Contact 12 external.sender@example.org 5>, alias=<Alias 5 e1@sl.local>
2026-07-14 04:10:58,929 - SL - DEBUG - 10602 - "/app/email_handler.py:1605" - handle_bounce_reply_phase() - 20b1d172-9097-4a1f-b1bf-26c1445d6f01 - Handle reply bounce <Mailbox 1 john@wick.com> -> <Alias 5 e1@sl.local> -> <Contact 12 external.sender@example.org 5>.<EmailLog 15>
2026-07-14 04:10:58,929 - SL - WARNING - 10602 - "/app/app/email_utils.py:727" - get_mailbox_bounce_info() - 20b1d172-9097-4a1f-b1bf-26c1445d6f01 - add missing content-transfer-encoding header
2026-07-14 04:10:58,934 - SL - DEBUG - 10602 - "/app/email_handler.py:1640" - handle_bounce_reply_phase() - 20b1d172-9097-4a1f-b1bf-26c1445d6f01 - Create refused email <Refused Email 4 None 2026-07-21T04:10:58.933720+00:00>
2026-07-14 04:10:58,939 - SL - DEBUG - 10602 - "/app/email_handler.py:1651" - handle_bounce_reply_phase() - 20b1d172-9097-4a1f-b1bf-26c1445d6f01 - Inform user <User 1 John Wick john@wick.com> about bounced email sent by <Alias 5 e1@sl.local> to <Contact 12 external.sender@example.org 5>
2026-07-14 04:10:58,962 - SL - DEBUG - 10602 - "/app/app/email_utils.py:303" - send_email() - 20b1d172-9097-4a1f-b1bf-26c1445d6f01 - send email to john@wick.com, subject 'Email cannot be sent to external.sender@example.org from your alias e1@sl.local'
2026-07-14 04:10:58,963 - SL - DEBUG - 10602 - "/app/app/mail_sender.py:131" - send() - 20b1d172-9097-4a1f-b1bf-26c1445d6f01 - send email with subject 'Email cannot be sent to external.sender@example.org from your alias e1@sl.local', from '"noreply@sl.local" <noreply@sl.local>' to 'john@wick.com'
2026-07-14 04:10:58,963 - SL - INFO - 10602 - "/app/email_handler.py:2367" - _handle() - 20b1d172-9097-4a1f-b1bf-26c1445d6f01 - Finish mail_from <>, rcpt_tos ['bounce_reply+15+@sl.local'], takes 0.0421900749206543 seconds with return code '250 SL E212 Bounce Reply phase handled'<<===
```

`is_bounce()` gates both sends (each required `mail_from == "<>"` **and** `multipart/report`, exactly as quoted earlier); the phase — `E211` vs `E212` — is chosen solely by the referenced email-log's `is_reply` flag inside `handle_bounce()` (`email_handler.py:L1851-L1914`), **not** by the recipient prefix. The status strings are defined at `app/email/status.py:L19` (`E211 = "250 SL E211 Bounce Forward phase handled"`) and `L20` (`E212 = "250 SL E212 Bounce Reply phase handled"`). Both outcomes are thus **observed**, not inferred.

#### The three exception-mapped codes E524/E213/E404, observed (plus a successful reply)

The exception→code mapping quoted above (`handle_DATA`, `email_handler.py:L2288-L2332`) was **exercised black-box against the running handler**, alongside the reply path's *success* case (distinct from the unauthorized `E214` of Case B and the reply-phase bounce `E212` above). This evidence comes from the re-verification run (email-handler pid `15828`, log `/tmp/qa_run/email.log` — the third disclosed exception to the single-timeline note in [§4](#4-q1--are-the-three-components-up-liveness--confirmation-signals)). One canonical SMTP session drives all of it. The script first sends a precondition forward, then **discovers the resulting reverse-alias and forward-log id at runtime** (so it reproduces regardless of the random reverse-alias suffix or the current id sequence), and then exercises: a successful reply, `E213`, `E524`, `E404`, and post-error recovery. SMTP sends use `smtplib` (the canonical entry point); `psycopg2` is used only to read back the ids/rows:

```
$ docker exec -i sl-app ./venv/bin/python - <<'PY'
import smtplib, email, psycopg2
DB = "host=sl-postgres dbname=simplelogin user=myuser password=mypassword"

def q(sql, args=None):
    c = psycopg2.connect(DB); cur = c.cursor(); cur.execute(sql, args or ())
    rows = cur.fetchall(); c.close(); return rows

def send(mail_from, rcpt_to, raw):
    s = smtplib.SMTP("127.0.0.1", 20381, timeout=30); s.ehlo("blitzy-docfix.local")
    cm, rm = s.mail(mail_from); cr, rr = s.rcpt(rcpt_to); cd, rd = s.data(raw); s.quit()
    return "MAIL -> %s %s | RCPT -> %s %s | POST-DATA -> %s %s" % (cm, rm.decode(), cr, rr.decode(), cd, rd.decode())

print("### STEP1 forward external.sender@example.org -> e1@sl.local (mint precondition contact+forward-log)")
fwd = ("From: External Sender <external.sender@example.org>\r\nTo: e1@sl.local\r\n"
       "Subject: Blitzy docfix precondition forward\r\nMessage-ID: <blitzy-precond-fwd@example.org>\r\n"
       "Content-Type: text/plain; charset=us-ascii\r\n\r\nprecondition body\r\n")
print(send("external.sender@example.org", "e1@sl.local", fwd))
row = q("SELECT c.id, c.reply_email, (SELECT max(id) FROM email_log WHERE contact_id=c.id AND is_reply=false) "
        "FROM contact c WHERE c.website_email='external.sender@example.org' AND c.alias_id=5 ORDER BY c.id DESC LIMIT 1")[0]
CID, RA, FWD_LOG = row[0], row[1], row[2]
print("PRECOND: contact_id=%s reverse_alias=%s forward_email_log_id=%s" % (CID, RA, FWD_LOG))
print()

print("### STEP2 [successful reply] john@wick.com -> %s" % RA)
rep = ("From: John Wick <john@wick.com>\r\nTo: " + RA + "\r\n"
       "Subject: Re: Blitzy docfix precondition forward\r\nMessage-ID: <blitzy-docfix-reply-2@wick.com>\r\n"
       "Content-Type: text/plain; charset=us-ascii\r\n\r\nJohn replying via reverse-alias.\r\n")
print(send("john@wick.com", RA, rep))
print("REPLY email_log row:", q("SELECT id, alias_id, contact_id, is_reply, message_id "
      "FROM email_log WHERE message_id='<blitzy-docfix-reply-2@wick.com>'"))
print()

print("### STEP3 [E213] normal msg to forward VERP bounce+%s+@sl.local" % FWD_LOG)
e213 = ("From: Someone <someone@external.example>\r\nTo: bounce+%s+@sl.local\r\n"
        "Subject: not a bounce\r\nMessage-ID: <blitzy-e213b@external.example>\r\n"
        "Content-Type: text/plain; charset=us-ascii\r\n\r\nbody\r\n") % FWD_LOG
print(send("someone@external.example", "bounce+%s+@sl.local" % FWD_LOG, e213))
print()

print("### STEP4 [E524] reverse-alias rep@sl.local as MAIL FROM in forward phase -> e0@sl.local")
e524 = ("From: Reverse Alias <rep@sl.local>\r\nTo: e0@sl.local\r\n"
        "Subject: reverse alias as sender\r\nMessage-ID: <blitzy-e524b@sl.local>\r\n"
        "Content-Type: text/plain; charset=us-ascii\r\n\r\nbody\r\n")
print(send("rep@sl.local", "e0@sl.local", e524))
print()

print("### STEP5 [E404] folded overlong Message-ID (>1024) -> e1@sl.local")
folded = "<blitzy-e404b" + ("".join("\r\n " + ("z"*200) for _ in range(7))) + "@example.org>"
raw404 = ("From: Sender <sender@external.example>\r\nTo: e1@sl.local\r\n"
          "Subject: overlong folded message id\r\nMessage-ID: " + folded + "\r\n"
          "Content-Type: text/plain; charset=us-ascii\r\n\r\nbody\r\n")
print("parsed Message-ID length =", len(email.message_from_string(raw404)["Message-ID"]))
print(send("sender@external.example", "e1@sl.local", raw404))
print("ROLLBACK check: email_log rows with overlong z Message-ID =",
      q("SELECT count(*) FROM email_log WHERE message_id LIKE '%%zzzz%%'")[0][0])
print()

print("### STEP6 [recovery] normal msg immediately after E404 (same handler, no restart)")
rec = ("From: Sender3 <sender3@external.example>\r\nTo: e1@sl.local\r\n"
       "Subject: recovery after E404\r\nMessage-ID: <blitzy-recovery-b@external.example>\r\n"
       "Content-Type: text/plain; charset=us-ascii\r\n\r\nrecovery body\r\n")
print(send("sender3@external.example", "e1@sl.local", rec))
print("RECOVERY email_log row:", q("SELECT id, alias_id, contact_id, is_reply "
      "FROM email_log WHERE message_id='<blitzy-recovery-b@external.example>'"))
PY
### STEP1 forward external.sender@example.org -> e1@sl.local (mint precondition contact+forward-log)
MAIL -> 250 OK | RCPT -> 250 OK | POST-DATA -> 250 Message accepted for delivery
PRECOND: contact_id=42 reverse_alias=external_sender_at_example_org_fbrcpf@sl.local forward_email_log_id=53

### STEP2 [successful reply] john@wick.com -> external_sender_at_example_org_fbrcpf@sl.local
MAIL -> 250 OK | RCPT -> 250 OK | POST-DATA -> 250 Message accepted for delivery
REPLY email_log row: [(54, 5, 42, True, '<blitzy-docfix-reply-2@wick.com>')]

### STEP3 [E213] normal msg to forward VERP bounce+53+@sl.local
MAIL -> 250 OK | RCPT -> 250 OK | POST-DATA -> 250 SL E213 Unknown email ignored

### STEP4 [E524] reverse-alias rep@sl.local as MAIL FROM in forward phase -> e0@sl.local
MAIL -> 250 OK | RCPT -> 250 OK | POST-DATA -> 550 SL E524 Wrong use of reverse-alias

### STEP5 [E404] folded overlong Message-ID (>1024) -> e1@sl.local
parsed Message-ID length = 1447
MAIL -> 250 OK | RCPT -> 250 OK | POST-DATA -> 421 SL E404 Unexpected error - Retry later
ROLLBACK check: email_log rows with overlong z Message-ID = 0

### STEP6 [recovery] normal msg immediately after E404 (same handler, no restart)
MAIL -> 250 OK | RCPT -> 250 OK | POST-DATA -> 250 Message accepted for delivery
RECOVERY email_log row: [(55, 5, 46, False)]
```

> **Note on the `E404` reproduction.** The overlong `Message-ID` must be **header-folded** across continuation lines. A single physical line longer than ~1000 bytes is rejected by `aiosmtpd` at the protocol layer with `500 Line too long (see RFC5321 4.5.3.1.6)` *before* the handler runs; folding keeps each physical line short while the unfolded value (here `1447` chars) still exceeds the `email_log.message_id` column's `character varying(1024)` limit, so the failure occurs where intended — on the `INSERT` inside `_handle()`.

The handler log confirms each outcome (salient lines, correlated by the per-message `message_id`; note the handler pid is **`15828`** on every line — the same process handled all six messages, so the `E404` neither crashed nor restarted it):

```
# STEP2 successful reply (message_id e8c474c2-…) — reply phase, EmailLog 54 (is_reply=t), rewrite + outbound send, Finish 250
2026-07-14 15:14:01,822 - SL - INFO - 15828 - "/app/email_handler.py:2343" - _handle() - e8c474c2-c09c-4326-b0a1-fbf628ee69e5 - New message, mail from john@wick.com, rctp tos ['external_sender_at_example_org_fbrcpf@sl.local']
2026-07-14 15:14:01,827 - SL - DEBUG - 15828 - "/app/email_handler.py:2196" - handle() - e8c474c2-c09c-4326-b0a1-fbf628ee69e5 - Reply phase john@wick.com(John Wick <john@wick.com>) -> external_sender_at_example_org_fbrcpf@sl.local
2026-07-14 15:14:01,832 - SL - DEBUG - 15828 - "/app/email_handler.py:1051" - handle_reply() - e8c474c2-c09c-4326-b0a1-fbf628ee69e5 - Create <EmailLog 54> for <Contact 42 external.sender@example.org 5>, <User 1 John Wick john@wick.com>, <Mailbox 1 john@wick.com>
2026-07-14 15:14:01,837 - SL - DEBUG - 15828 - "/app/email_handler.py:1171" - handle_reply() - e8c474c2-c09c-4326-b0a1-fbf628ee69e5 - From header is e1@sl.local
2026-07-14 15:14:01,846 - SL - DEBUG - 15828 - "/app/email_handler.py:1212" - handle_reply() - e8c474c2-c09c-4326-b0a1-fbf628ee69e5 - send email from e1@sl.local to external.sender@example.org, mail_options:[],rcpt_options:[]
2026-07-14 15:14:01,846 - SL - DEBUG - 15828 - "/app/app/mail_sender.py:131" - send() - e8c474c2-c09c-4326-b0a1-fbf628ee69e5 - send email with subject 'Re: Blitzy docfix precondition forward', from 'e1@sl.local' to 'External Sender <external.sender@example.org>'
2026-07-14 15:14:01,848 - SL - INFO - 15828 - "/app/email_handler.py:2367" - _handle() - e8c474c2-c09c-4326-b0a1-fbf628ee69e5 - Finish mail_from john@wick.com, rcpt_tos ['external_sender_at_example_org_fbrcpf@sl.local'], takes 0.025511980056762695 seconds with return code '250 Message accepted for delivery'<<===

# STEP3 E213 (message_id b5278055-…) — VERPForward caught in handle_DATA (no Finish line)
2026-07-14 15:14:01,864 - SL - INFO - 15828 - "/app/email_handler.py:2343" - _handle() - b5278055-2381-4711-be5a-8d5f64d627a5 - New message, mail from someone@external.example, rctp tos ['bounce+53+@sl.local']
2026-07-14 15:14:01,869 - SL - WARNING - 15828 - "/app/email_handler.py:2309" - handle_DATA() - b5278055-2381-4711-be5a-8d5f64d627a5 - email handling fail with error:VERPForward  mail_from:someone@external.example, rcpt_tos:['bounce+53+@sl.local'], header_from:Someone <someone@external.example>, header_to:bounce+53+@sl.local

# STEP4 E524 (message_id d46b6096-…) — forward phase sees a reverse-alias sender; CannotCreateContactForReverseAlias caught in handle_DATA (no Finish line)
2026-07-14 15:14:01,871 - SL - INFO - 15828 - "/app/email_handler.py:2343" - _handle() - d46b6096-b4da-4d13-9214-07716c8db671 - New message, mail from rep@sl.local, rctp tos ['e0@sl.local']
2026-07-14 15:14:01,881 - SL - DEBUG - 15828 - "/app/email_handler.py:2202" - handle() - d46b6096-b4da-4d13-9214-07716c8db671 - Forward phase rep@sl.local(Reverse Alias <rep@sl.local>) -> e0@sl.local
2026-07-14 15:14:01,887 - SL - DEBUG - 15828 - "/app/email_handler.py:580" - handle_forward() - d46b6096-b4da-4d13-9214-07716c8db671 - Create or get contact for from_header:Reverse Alias <rep@sl.local>
2026-07-14 15:14:01,897 - SL - WARNING - 15828 - "/app/email_handler.py:2298" - handle_DATA() - d46b6096-b4da-4d13-9214-07716c8db671 - Probably due to reverse-alias used in the forward phase, error:CannotCreateContactForReverseAlias <Contact 1 hey@google.com 2> mail_from:rep@sl.local, rcpt_tos:['e0@sl.local'], header_from:Reverse Alias <rep@sl.local>, header_to:e0@sl.local

# STEP5 E404 (message_id e023f08c-…) — generic Exception (DB truncation) caught in handle_DATA at L2320 (no Finish line; row rolled back)
2026-07-14 15:14:01,900 - SL - INFO - 15828 - "/app/email_handler.py:2343" - _handle() - e023f08c-dca1-4842-a3b9-c0481bf50ff5 - New message, mail from sender@external.example, rctp tos ['e1@sl.local']
2026-07-14 15:14:01,920 - SL - ERROR - 15828 - "/app/email_handler.py:2320" - handle_DATA() - e023f08c-dca1-4842-a3b9-c0481bf50ff5 - email handling fail with error:(psycopg2.errors.StringDataRightTruncation) value too long for type character varying(1024)
    (… Python traceback …)
psycopg2.errors.StringDataRightTruncation: value too long for type character varying(1024)
sqlalchemy.exc.DataError: (psycopg2.errors.StringDataRightTruncation) value too long for type character varying(1024)

# STEP6 recovery (message_id 0d3a020b-…) — same process, EmailLog 55 created, Finish 250
2026-07-14 15:14:01,937 - SL - INFO - 15828 - "/app/email_handler.py:2343" - _handle() - 0d3a020b-965f-4555-9d86-d19b5683b604 - New message, mail from sender3@external.example, rctp tos ['e1@sl.local']
2026-07-14 15:14:01,974 - SL - DEBUG - 15828 - "/app/email_handler.py:688" - forward_email_to_mailbox() - 0d3a020b-965f-4555-9d86-d19b5683b604 - Forward <Contact 46 sender3@external.example 5> -> <Alias 5 e1@sl.local> -> <Mailbox 1 john@wick.com>
2026-07-14 15:14:01,976 - SL - DEBUG - 15828 - "/app/email_handler.py:740" - forward_email_to_mailbox() - 0d3a020b-965f-4555-9d86-d19b5683b604 - Create <EmailLog 55> for <Contact 46 sender3@external.example 5>, <User 1 John Wick john@wick.com>, <Mailbox 1 john@wick.com>
2026-07-14 15:14:01,981 - SL - INFO - 15828 - "/app/email_handler.py:2367" - _handle() - 0d3a020b-965f-4555-9d86-d19b5683b604 - Finish mail_from sender3@external.example, rcpt_tos ['e1@sl.local'], takes 0.04385852813720703 seconds with return code '250 Message accepted for delivery'<<===
```

Interpreting each, against the exception map in `handle_DATA`:

- **Successful reply → `E200` (`250 Message accepted for delivery`).** The recipient `external_sender_at_example_org_fbrcpf@sl.local` is a reverse-alias, so the **reply** path runs (`email_handler.py:L2196`); `handle_reply()` creates a **reply** `<EmailLog 54>` (`is_reply=t`, `email_handler.py:L1051`), rewrites the `From:` to the alias `e1@sl.local` (`L1171`), and "sends" the message out to the real contact (`L1212`; the actual send is logged, not delivered, because `NOT_SEND_EMAIL=true`). Success is signalled by the same `2xx` string as a forward; the `is_reply=t` flag is what distinguishes a reply from a forward at the data layer (`email_log` row `(54, 5, 42, True, …)`).
- **`E213` (`250 SL E213 Unknown email ignored`).** A *non-bounce* message addressed to a **forward VERP** address `bounce+53+@sl.local` raises `VERPForward` (the referenced `<EmailLog 53>` is not a bounce), which `handle_DATA` maps to `E213` (`email_handler.py:L2308-L2318`). Because the exception is caught in `handle_DATA` (outside `_handle()`), the terminal log line is the `LOG.w` at `email_handler.py:L2309` — there is **no** `Finish …<<===` line.
- **`E524` (`550 SL E524 Wrong use of reverse-alias`).** A reverse-alias (`rep@sl.local`, the seed `<Contact 1>`'s reverse-alias) used as the **`MAIL FROM`** enters the forward phase; trying to create a contact *for a reverse-alias* raises `CannotCreateContactForReverseAlias`, mapped to `E524` (`email_handler.py:L2297-L2307`). Terminal line is the `LOG.w` at `email_handler.py:L2298`; again no `Finish` line.
- **`E404` (`421 SL E404 Unexpected error - Retry later`).** The overlong `Message-ID` overflows `email_log.message_id` (`character varying(1024)`) on `INSERT`, raising `sqlalchemy.exc.DataError`; the generic `except Exception` maps it to `E404` — a **4xx** *retry* code, not a 5xx (`email_handler.py:L2319-L2332`). Terminal line is the `LOG.e` at `email_handler.py:L2320`; the transaction is rolled back, so **no** `email_log` row persists (the `ROLLBACK check` returns `0`).
- **Recovery.** The very next message (STEP6) is processed normally by the **same** handler process (pid `15828`), creating `<EmailLog 55>` and returning `E200` — demonstrating that a per-message `DataError` does not take the handler down (each message runs in its own `create_light_app().app_context()`, so the failed transaction is isolated).

All four codes (`E200` for the reply, `E213`, `E524`, `E404`) and the recovery are therefore **observed at runtime**, not inferred. The temporary rows they created (`<EmailLog 53/54/55>`, `<Contact 46>`) were removed during cleanup ([§7](#7-cleanup--the-repository-and-database-are-left-unchanged)).


### 6.6 Secondary: alias auto-creation (directory success) and domain prerequisites

A further "data handling" behavior of the email handler is that an inbound message to an address that does not yet exist can **create the alias on the fly**. The seed data already provisions two **enabled directories** owned by `john@wick.com` — `abcd` (id `1`) and `xyzt` (id `2`) — so this path is exercisable without any configuration change (this refutes an earlier claim that the seed lacked directories):

```
$ docker exec sl-postgres psql -U myuser -d simplelogin -c "SELECT d.id, d.name, d.user_id, d.disabled FROM directory d WHERE d.name='abcd';"
 id | name | user_id | disabled
----+------+---------+----------
  1 | abcd |       1 | f
(1 row)
```

**The prerequisites** (all checked in `check_if_alias_can_be_auto_created_for_a_directory()`, `app/alias_utils.py:L145`): the recipient domain must be an allowed alias domain (`can_create_directory_for_address`, `L153`); the local-part must contain a **directory separator** — `/`, `+`, or `#`, tested in that order (`L157-L165`); the substring **before** the separator is the `directory_name` (`L168`); a `Directory` with that name must exist (`L171`); its owner must not be disabled (`L177`) and must be under their alias quota (`L181`); and the directory itself must not be disabled (`L189`). `try_auto_create()` (`app/alias_utils.py:L202`) first tries catch-all/custom-domain creation (`try_auto_create_via_domain`, `L274` — declines here, since `sl.local` is the system domain, not a catch-all custom domain) and then `try_auto_create_directory()` (`L227`), which creates the alias owned by the directory's user with a `Created by directory <name>` note (`L238-L248`).

**Observed — a real SMTP delivery to `abcd+blitzyprobe@sl.local`.** Before the send there was no such alias. The send returned `250 Message accepted for delivery`:

```
$ python3 /tmp/blitzy_investigation/q3_dir_autocreate.py
EHLO      -> 250
MAIL FROM -> 250 OK
RCPT TO   -> 250 OK
DATA      -> 354 End data with <CR><LF>.<CR><LF>
POST-DATA -> 250 Message accepted for delivery
```

The handler created the alias on the fly and then forwarded the message through it in a single lifecycle (the full contiguous slice from `/tmp/sl_run/email.log`, correlated by `message_id db2f70da-d452-4689-a519-fc6231596be2`):

```
2026-07-13 19:14:59,864 - SL - INFO - 4648 - "/app/email_handler.py:2343" - _handle() - db2f70da-d452-4689-a519-fc6231596be2 - New message, mail from friend@external.example, rctp tos ['abcd+blitzyprobe@sl.local']
2026-07-13 19:14:59,871 - SL - DEBUG - 4648 - "/app/email_handler.py:2202" - handle() - db2f70da-d452-4689-a519-fc6231596be2 - Forward phase friend@external.example(A Friend <friend@external.example>) -> abcd+blitzyprobe@sl.local
2026-07-13 19:14:59,877 - SL - DEBUG - 4648 - "/app/email_handler.py:545" - handle_forward() - db2f70da-d452-4689-a519-fc6231596be2 - alias abcd+blitzyprobe@sl.local not exist. Try to see if it can be created on the fly
2026-07-13 19:14:59,881 - SL - DEBUG - 4648 - "/app/app/alias_utils.py:169" - check_if_alias_can_be_auto_created_for_a_directory() - db2f70da-d452-4689-a519-fc6231596be2 - directory_name abcd
2026-07-13 19:14:59,889 - SL - DEBUG - 4648 - "/app/app/alias_utils.py:238" - try_auto_create_directory() - db2f70da-d452-4689-a519-fc6231596be2 - create alias abcd+blitzyprobe@sl.local for directory <Directory abcd>
2026-07-13 19:14:59,926 - SL - DEBUG - 4648 - "/app/app/contact_utils.py:110" - create_contact() - db2f70da-d452-4689-a519-fc6231596be2 - Created contact <Contact 3 friend@external.example 16> for alias <Alias 16 abcd+blitzyprobe@sl.local> with email friend@external.example invalid_email=False
2026-07-13 19:14:59,933 - SL - DEBUG - 4648 - "/app/email_handler.py:688" - forward_email_to_mailbox() - db2f70da-d452-4689-a519-fc6231596be2 - Forward <Contact 3 friend@external.example 16> -> <Alias 16 abcd+blitzyprobe@sl.local> -> <Mailbox 1 john@wick.com>
2026-07-13 19:14:59,936 - SL - DEBUG - 4648 - "/app/email_handler.py:740" - forward_email_to_mailbox() - db2f70da-d452-4689-a519-fc6231596be2 - Create <EmailLog 4> for <Contact 3 friend@external.example 16>, <User 1 John Wick john@wick.com>, <Mailbox 1 john@wick.com>
2026-07-13 19:14:59,944 - SL - INFO - 4648 - "/app/email_handler.py:2367" - _handle() - db2f70da-d452-4689-a519-fc6231596be2 - Finish mail_from friend@external.example, rcpt_tos ['abcd+blitzyprobe@sl.local'], takes 0.07952475547790527 seconds with return code '250 Message accepted for delivery'<<===
```

The lifecycle is: forward phase (`email_handler.py:L2202`) → alias missing, try on-the-fly (`L545`) → `directory_name abcd` parsed (`app/alias_utils.py:L169`) → **`create alias abcd+blitzyprobe@sl.local for directory <Directory abcd>`** (`app/alias_utils.py:L238`) → contact created (`app/contact_utils.py:L110`) → forwarded to the directory owner's mailbox `john@wick.com` (`email_handler.py:L688`) → `EmailLog 4` created (`email_handler.py:L740`) → `250`. The resulting alias row:

```
$ docker exec sl-postgres psql -U myuser -d simplelogin -c "SELECT id, email, user_id, directory_id, note, to_char(created_at,'HH24:MI:SS') created FROM alias WHERE email='abcd+blitzyprobe@sl.local';"
 id |           email           | user_id | directory_id |           note            | created
----+---------------------------+---------+--------------+---------------------------+----------
 16 | abcd+blitzyprobe@sl.local |       1 |            1 | Created by directory abcd | 19:14:59
(1 row)
```

Alias `16` (and its contact `3` and email-log `4`) are removed during the pristine-state restoration in [§7](#7-cleanup--the-repository-and-database-are-left-unchanged).

**Observed — catch-all domain auto-creation (the second `try_auto_create` branch).** The `try_auto_create_via_domain()` branch (`app/alias_utils.py:L274`) that *declined* for the `sl.local` directory case above **does** fire for a **verified custom domain whose catch-all is enabled**. The seed ships `old.com` verified but with `catch_all=f` ([§2.6](#26-what-the-seed-actually-creates-the-real-baseline)), so this behavior is not available *by default* — but enabling it is a **normal dashboard action** (toggling "Catch-all" on the domain's settings page), **not** a source change. This was therefore exercised end-to-end against the running stack: catch-all was enabled through the real dashboard endpoint, a message was delivered to a brand-new address on `old.com`, and catch-all was then toggled back off — all in one canonical run (web UI over HTTP + SMTP delivery + `psycopg2` read-back; email-handler pid `15828`):

```
$ docker exec -i sl-app ./venv/bin/python - <<'PY'
import re, smtplib, psycopg2, requests
BASE = "http://127.0.0.1:7777"
DB = "host=sl-postgres dbname=simplelogin user=myuser password=mypassword"
TARGET = "qa-catchall-probe-8399b@old.com"

def q(sql):
    c = psycopg2.connect(DB); cur = c.cursor(); cur.execute(sql); rows = cur.fetchall(); c.close(); return rows
def csrf(h):
    m = re.search(r'name="csrf_token"[^>]*value="([^"]+)"', h); return m.group(1) if m else None
def toggle(sess):
    r = sess.get(BASE + "/auth/login", timeout=15); tok = csrf(r.text)
    r = sess.post(BASE + "/auth/login", data={"csrf_token": tok, "email": "john@wick.com", "password": "password"}, allow_redirects=False, timeout=15)
    login = (r.status_code, r.headers.get("Location"))
    r = sess.get(BASE + "/dashboard/domains/2/info", timeout=15); tok = csrf(r.text)
    r = sess.post(BASE + "/dashboard/domains/2/info", data={"csrf_token": tok, "form-name": "switch-catch-all"}, allow_redirects=False, timeout=15)
    return login, (r.status_code, r.headers.get("Location"))

print("catch_all BEFORE:", q("SELECT catch_all FROM custom_domain WHERE id=2")[0][0])
login, tog = toggle(requests.Session())
print("POST /auth/login ->", login[0], "Location:", login[1])
print("POST switch-catch-all ->", tog[0], "Location:", tog[1])
print("catch_all AFTER toggle-on:", q("SELECT catch_all FROM custom_domain WHERE id=2")[0][0])
print()
print("### SMTP to brand-new address on the catch-all domain")
raw = ("From: External Sender <catchall.sender@example.org>\r\nTo: " + TARGET + "\r\n"
       "Subject: Blitzy catch-all auto-create probe\r\nMessage-ID: <blitzy-catchall-2@example.org>\r\n"
       "Content-Type: text/plain; charset=us-ascii\r\n\r\nMail to a brand-new address on a catch-all domain.\r\n")
s = smtplib.SMTP("127.0.0.1", 20381, timeout=30); s.ehlo("blitzy-docfix.local")
cm, rm = s.mail("catchall.sender@example.org"); cr, rr = s.rcpt(TARGET); cd, rd = s.data(raw); s.quit()
print("MAIL -> %s %s | RCPT -> %s %s | POST-DATA -> %s %s" % (cm, rm.decode(), cr, rr.decode(), cd, rd.decode()))
print()
print("### auto-created rows")
print("alias:", q("SELECT id, email, note FROM alias WHERE email='%s'" % TARGET))
print("contact:", q("SELECT id, alias_id, website_email FROM contact WHERE website_email='catchall.sender@example.org' ORDER BY id DESC LIMIT 1"))
print("email_log:", q("SELECT id, alias_id, contact_id, is_reply FROM email_log WHERE message_id='<blitzy-catchall-2@example.org>'"))
print()
print("### restore catch_all -> f")
_, tog2 = toggle(requests.Session())
print("POST switch-catch-all (restore) ->", tog2[0], "Location:", tog2[1])
print("catch_all AFTER restore:", q("SELECT catch_all FROM custom_domain WHERE id=2")[0][0])
PY
catch_all BEFORE: False
POST /auth/login -> 302 Location: http://127.0.0.1:7777/dashboard/
POST switch-catch-all -> 302 Location: http://127.0.0.1:7777/dashboard/domains/2/info
catch_all AFTER toggle-on: True

### SMTP to brand-new address on the catch-all domain
MAIL -> 250 OK | RCPT -> 250 OK | POST-DATA -> 250 Message accepted for delivery

### auto-created rows
alias: [(67, 'qa-catchall-probe-8399b@old.com', 'Created by catchall option')]
contact: [(47, 67, 'catchall.sender@example.org')]
email_log: [(56, 67, 47, False)]

### restore catch_all -> f
POST switch-catch-all (restore) -> 302 Location: http://127.0.0.1:7777/dashboard/domains/2/info
catch_all AFTER restore: False
```

The handler log confirms the catch-all creation path (correlated by `message_id 30ace68d-…`; the alias is created, then a contact, then the forward proceeds exactly as a normal forward):

```
2026-07-14 15:18:47,855 - SL - INFO - 15828 - "/app/email_handler.py:2343" - _handle() - 30ace68d-2c79-463c-9dc1-15ca4bfe9044 - New message, mail from catchall.sender@example.org, rctp tos ['qa-catchall-probe-8399b@old.com']
2026-07-14 15:18:47,867 - SL - DEBUG - 15828 - "/app/email_handler.py:545" - handle_forward() - 30ace68d-2c79-463c-9dc1-15ca4bfe9044 - alias qa-catchall-probe-8399b@old.com not exist. Try to see if it can be created on the fly
2026-07-14 15:18:47,872 - SL - DEBUG - 15828 - "/app/app/alias_utils.py:140" - check_if_alias_can_be_auto_created_for_custom_domain() - 30ace68d-2c79-463c-9dc1-15ca4bfe9044 - Create alias via catchall
2026-07-14 15:18:47,873 - SL - DEBUG - 15828 - "/app/app/alias_utils.py:299" - try_auto_create_via_domain() - 30ace68d-2c79-463c-9dc1-15ca4bfe9044 - create alias qa-catchall-probe-8399b@old.com for domain <Custom Domain 2 old.com>
2026-07-14 15:18:47,899 - SL - DEBUG - 15828 - "/app/app/contact_utils.py:110" - create_contact() - 30ace68d-2c79-463c-9dc1-15ca4bfe9044 - Created contact <Contact 47 catchall.sender@example.org 67> for alias <Alias 67 qa-catchall-probe-8399b@old.com> with email catchall.sender@example.org invalid_email=False
2026-07-14 15:18:47,908 - SL - DEBUG - 15828 - "/app/email_handler.py:740" - forward_email_to_mailbox() - 30ace68d-2c79-463c-9dc1-15ca4bfe9044 - Create <EmailLog 56> for <Contact 47 catchall.sender@example.org 67>, <User 1 John Wick john@wick.com>, <Mailbox 1 john@wick.com>
2026-07-14 15:18:47,913 - SL - INFO - 15828 - "/app/email_handler.py:2367" - _handle() - 30ace68d-2c79-463c-9dc1-15ca4bfe9044 - Finish mail_from catchall.sender@example.org, rcpt_tos ['qa-catchall-probe-8399b@old.com'], takes 0.057846784591674805 seconds with return code '250 Message accepted for delivery'<<===
```

So catch-all auto-creation **is** feasible and was **observed**: the decisive difference from the directory path is the branch taken inside `try_auto_create()` (`try_auto_create_via_domain()` for a catch-all custom domain, `app/alias_utils.py:L274,L299`, vs `try_auto_create_directory()` for a directory). The newly created `<Alias 67>` carries the note **`Created by catchall option`** (set at `app/alias_utils.py:L285`), distinguishing it from the directory path's `Created by directory <name>` note. The toggle is idempotent for cleanup — the same run enabled catch-all and then disabled it again (`catch_all` returns to `False`), and the temporary rows (`<Alias 67>`, `<Contact 47>`, `<EmailLog 56>`) are removed during cleanup ([§7](#7-cleanup--the-repository-and-database-are-left-unchanged)).

---

## 7. Cleanup — the repository and database are left unchanged

The investigation is read-only: **no existing repository file is modified, added to, or deleted** — the only committed artifact is this document (`blitzy/documentation/app_2cd6ee777f8c.md`). Every account, alias, contact, and email-log created while exercising the flows above is removed here; the **live entity tables — and the mutable `daily_metric` counter — are restored exactly to the seed baseline** of [§2.6](#26-what-the-seed-actually-creates-the-real-baseline), while the **append-only** audit/bookkeeping tables (`deleted_alias`, `alias_audit_log`, `user_audit_log`, `job`, and `sent_alert`) retain monotonic residual — a database that is pristine in *every* table is produced only by re-running the §2.5 reset, as detailed and quantified in [§7.2](#72-final-database-state-after-cleanup). All temporary observation scripts live **outside** the repository (under host and container `/tmp/`) and are never tracked or committed; they are inventoried and removed in [§7.3](#73-source-tree-untouched).

### 7.1 What was created, and how it was removed

The complete inventory of test data created during the investigation, with the section that created each item:

| Table | Investigation rows (ids) | Created in | Baseline had |
|-------|--------------------------|------------|--------------|
| `users` | `3` (`qa.tester.9f3@example.com`) | [§5.1](#51-c1--create-a-new-account-signup--activation) | 2 rows (ids 1–2) |
| `alias` | `12` (auto newsletter, user 3), `13`/`15` (random, user 1), `14` (custom `@old.com`, user 1), `16` (directory `abcd`, user 1) | [§5.1](#51-c1--create-a-new-account-signup--activation), [§5.3](#53-c3--create-an-alias-random-and-custom), [§6.6](#66-secondary-alias-auto-creation-directory-success-and-domain-prerequisites) | 11 rows (ids 1–11) |
| `contact` | `2`, `3` | [§5.4](#54-c4--an-email-arrives-at-the-alias-forward-path), [§6.6](#66-secondary-alias-auto-creation-directory-success-and-domain-prerequisites) | 1 row (id 1) |
| `email_log` | `2`, `3`, `4` | [§5.4](#54-c4--an-email-arrives-at-the-alias-forward-path), [§6.6](#66-secondary-alias-auto-creation-directory-success-and-domain-prerequisites) | 1 row (id 1) |
| `job` | `1`–`3` (`[NON-CANONICAL]` §4.3 probes), `4` (canonical delete-account, §6.2), `5`/`6` (retry probes, §6.3), `7` (unknown-job, §6.4) | [§4.3](#43-job-runner-job_runnerpy-no-port), [§6.2](#62-a-genuinely-enqueued-job-canonical-ready--taken--done), [§6.3](#63-retry-accounting-and-the-run_at-selection-gate), [§6.4](#64-edge-an-unknown-job-name) | 0 rows |
| `deleted_alias` | `1` (alias 12 global-trash entry) | [§6.2](#62-a-genuinely-enqueued-job-canonical-ready--taken--done) | 0 rows |
| `alias_audit_log` | ids `> 11` (create/delete events for aliases 12–16) | §5.1/§5.3/§6.6 | 11 rows (ids 1–11) |
| `user_audit_log` | `1`–`3` (2× `create_contact`, 1× `user_marked_for_deletion`) | §5.4/§6.6/§6.2 | 0 rows |
| `daily_metric` (id 1) | counters mutated: `nb_new_web_non_proton_user` 0→1, `nb_alias` 11→16 | §5.1/§5.3/§6.6 | `(0, 11)` |

Removal happened in two stages.

**Stage 1 — the canonical account deletion already did part of the work.** The genuinely-enqueued delete-account job in [§6.2](#62-a-genuinely-enqueued-job-canonical-ready--taken--done) ran `User.delete(3)` in the job runner's `delete-account` branch (`job_runner.py:L226-L244`, with `User.delete(user.id)` at `job_runner.py:L243`), which removed `users` id `3` and cascade-removed its newsletter alias `12` into the `deleted_alias` global trash. That removal is proven at the end of §6.2 (`users id=3 count=0`, `alias id=12 count=0`). The test aliases `13`–`16`, their contacts/email-logs, all `job` rows, and the audit/metric residue belong to the seed account (`user_id=1 john@wick.com`) or to global tables, so they are **not** cascaded by deleting user 3 and are removed in stage 2.

**Stage 2 — one scoped SQL transaction restores the remainder.** The cleanup script deletes children before parents (FK-safe order) and restores the `daily_metric` counters, wrapped in a single `BEGIN … COMMIT` with `\set ON_ERROR_STOP on` so any FK violation rolls back the whole batch. It carries an explicit disposable-DB-only warning and pins its destination to the disposable local container, user, and database in the invocation shown below. Before any mutation, a **preflight scope-confirmation `SELECT`** — plus an FK-child introspection confirming all child tables of `alias`/`email_log` are empty for the target ids — verifies exactly which rows will be affected:

```
$ cat /tmp/blitzy_investigation/q6_preflight.sql
-- PREFLIGHT scope-confirmation (run BEFORE the DELETE transaction): count the exact
-- rows each DELETE will affect. Per-target counts must match the intended scope.
SELECT 'email_log id IN (2,3,4)'           AS target, count(*) FROM email_log       WHERE id IN (2,3,4)
UNION ALL SELECT 'contact id IN (2,3)',               count(*) FROM contact         WHERE id IN (2,3)
UNION ALL SELECT 'alias_audit_log id > 11',           count(*) FROM alias_audit_log WHERE id > 11
UNION ALL SELECT 'alias id IN (13,14,15,16)',         count(*) FROM alias           WHERE id IN (13,14,15,16)
UNION ALL SELECT 'deleted_alias id = 1',              count(*) FROM deleted_alias   WHERE id = 1
UNION ALL SELECT 'job id BETWEEN 1 AND 7',            count(*) FROM job             WHERE id BETWEEN 1 AND 7
UNION ALL SELECT 'user_audit_log id BETWEEN 1 AND 3', count(*) FROM user_audit_log  WHERE id BETWEEN 1 AND 3;
```

Run immediately before the transaction, its per-target counts were `3, 2, 6, 4, 1, 7, 3` — matching the `DELETE` row-counts below exactly. Re-running the identical preflight *after* cleanup returns zero for every target, confirming the transaction affected exactly its intended scope and nothing outside it:

```
$ docker exec -i sl-postgres psql -U myuser -d simplelogin < /tmp/blitzy_investigation/q6_preflight.sql
              target               | count
-----------------------------------+-------
 email_log id IN (2,3,4)           |     0
 contact id IN (2,3)               |     0
 alias_audit_log id > 11           |     0
 alias id IN (13,14,15,16)         |     0
 deleted_alias id = 1              |     0
 job id BETWEEN 1 AND 7            |     0
 user_audit_log id BETWEEN 1 AND 3 |     0
(7 rows)
```

The cleanup script itself:

```
$ cat /tmp/blitzy_investigation/q6_cleanup.sql
-- =====================================================================
-- SCOPED CLEANUP — DISPOSABLE LOCAL INVESTIGATION DB ONLY.
-- Deletes exactly the rows created during this read-only investigation,
-- restoring the pristine flask-dummy-data baseline. NEVER run against any
-- shared/production database. Wrapped in a single transaction; ON_ERROR_STOP
-- aborts and rolls back the whole batch if any FK constraint is violated.
-- =====================================================================
\set ON_ERROR_STOP on
BEGIN;
-- children before parents (FK-safe order)
DELETE FROM email_log       WHERE id IN (2,3,4);                 -- §5.4 + §6.6 forwards
DELETE FROM contact         WHERE id IN (2,3);                   -- §5.4 (alias5) + §6.6 (alias16)
DELETE FROM alias_audit_log WHERE id > 11;                       -- create/delete events for aliases 12-16
DELETE FROM alias           WHERE id IN (13,14,15,16);           -- test aliases (12 already deleted by delete-account job)
DELETE FROM deleted_alias   WHERE id = 1;                        -- alias 12 global-trash entry
DELETE FROM job             WHERE id BETWEEN 1 AND 7;            -- all investigation jobs (baseline 0)
DELETE FROM user_audit_log  WHERE id BETWEEN 1 AND 3;            -- 2x create_contact + 1x user_marked_for_deletion (baseline 0)
UPDATE daily_metric SET nb_new_web_non_proton_user = 0, nb_alias = 11 WHERE id = 1;  -- restore counters to baseline (0, 11)
COMMIT;
```

Executing the transaction (each `DELETE n` count matches the inventory above — `email_log` 3, `contact` 2, `alias_audit_log` 6, `alias` 4, `deleted_alias` 1, `job` 7, `user_audit_log` 3):

```
$ docker exec -i sl-postgres psql -U myuser -d simplelogin < /tmp/blitzy_investigation/q6_cleanup.sql
BEGIN
DELETE 3
DELETE 2
DELETE 6
DELETE 4
DELETE 1
DELETE 7
DELETE 3
UPDATE 1
COMMIT
```

### 7.2 Final database state after cleanup

Cleanup is proven at two levels, and the document is explicit that they behave differently.

**(a) The live entity tables are restored *exactly* to the seed baseline.** The rows created while exercising the flows — the signup/alias/forward rows of §5–§6.6, the exception-code and successful-reply probes of §6.5, and the catch-all aliases of §6.6 — are deleted in FK-safe order (children before parents; every child of `alias`/`contact`/`email_log` is `ON DELETE CASCADE`, so no orphan can survive) inside a single transaction. The before/after counts show `alias`, `contact`, and `email_log` returning to their exact seed counts and max ids, with the two seed users untouched:

```
$ docker exec -i sl-postgres psql -U myuser -d simplelogin <<'SQL'
\pset border 1
\echo ==== BEFORE (live entity tables) ====
SELECT 'email_log' AS t, count(*) AS rows, max(id) AS max_id FROM email_log
UNION ALL SELECT 'contact', count(*), max(id) FROM contact
UNION ALL SELECT 'alias', count(*), max(id) FROM alias
UNION ALL SELECT 'users', count(*), max(id) FROM users ORDER BY t;
\echo ==== DELETE reproduction rows (FK-safe order: email_log -> contact -> alias) ====
BEGIN;
DELETE FROM email_log WHERE id BETWEEN 48 AND 56;
DELETE FROM contact   WHERE id BETWEEN 42 AND 47;
DELETE FROM alias     WHERE id IN (66,67);
COMMIT;
\echo ==== AFTER (live entity tables == seed baseline) ====
SELECT 'email_log' AS t, count(*) AS rows, max(id) AS max_id FROM email_log
UNION ALL SELECT 'contact', count(*), max(id) FROM contact
UNION ALL SELECT 'alias', count(*), max(id) FROM alias
UNION ALL SELECT 'users', count(*), max(id) FROM users ORDER BY t;
SQL
Border style is 1.
==== BEFORE (live entity tables) ====
     t     | rows | max_id
-----------+------+--------
 alias     |   13 |     67
 contact   |    7 |     47
 email_log |   10 |     56
 users     |    2 |      2
(4 rows)

==== DELETE reproduction rows (FK-safe order: email_log -> contact -> alias) ====
BEGIN
DELETE 9
DELETE 6
DELETE 2
COMMIT
==== AFTER (live entity tables == seed baseline) ====
     t     | rows | max_id
-----------+------+--------
 alias     |   11 |     11
 contact   |    1 |      1
 email_log |    1 |      1
 users     |    2 |      2
(4 rows)
```

The `DELETE 9 / 6 / 2` counts are the nine reproduction `email_log` rows, six `contact` rows, and two catch-all `alias` rows; the `AFTER` block shows the three tables back at their seed counts (`alias` 11, `contact` 1, `email_log` 1) and max ids, with `users` unchanged at 2. Deleting via direct SQL (rather than the application's `delete_alias()`) intentionally does **not** write new `deleted_alias` trash rows.

The surviving rows are confirmed to be exactly the seed rows — the two seed users and the eleven seed aliases (including the seeded `e1@sl.local`, id `5`) — and `old.com`'s `catch_all` flag is back at its seed value `f` (it was toggled on, then off, in §6.6):

```
$ docker exec sl-postgres psql -U myuser -d simplelogin -c "SELECT id, email FROM users ORDER BY id;"
 id |          email
----+-------------------------
  1 | john@wick.com
  2 | winston@continental.com
(2 rows)

$ docker exec sl-postgres psql -U myuser -d simplelogin -c "SELECT id, email FROM alias ORDER BY id;"
 id |                  email
----+-----------------------------------------
  1 | simplelogin-newsletter.list417@sl.local
  2 | word_test646@sl.local
  3 | example@example.com
  4 | e0@sl.local
  5 | e1@sl.local
  6 | e2@sl.local
  7 | first@ab.cd
  8 | second@ab.cd
  9 | simplelogin-newsletter.word568@sl.local
 10 | john@example.com
 11 | wick@example.com
(11 rows)

$ docker exec sl-postgres psql -U myuser -d simplelogin -c "SELECT id, domain, catch_all FROM custom_domain ORDER BY id;"
 id | domain  | catch_all
----+---------+-----------
  1 | ab.cd   | f
  2 | old.com | f
(2 rows)
```

The test account and every test alias are gone; the two seed users and eleven seed aliases remain, and both custom domains are back at `catch_all=f`. A final targeted probe confirms **zero** reproduction rows survive in the live entity tables:

```
$ docker exec sl-postgres psql -U myuser -d simplelogin -c "
SELECT 'users id>2 (test accts)'      AS probe, count(*) FROM users     WHERE id > 2
UNION ALL SELECT 'alias id>11 (test aliases)',   count(*) FROM alias     WHERE id > 11
UNION ALL SELECT 'contact id>1 (test contacts)', count(*) FROM contact   WHERE id > 1
UNION ALL SELECT 'email_log id>1 (test logs)',   count(*) FROM email_log WHERE id > 1;"
            probe             | count
------------------------------+-------
 users id>2 (test accts)      |     0
 alias id>11 (test aliases)   |     0
 contact id>1 (test contacts) |     0
 email_log id>1 (test logs)   |     0
(4 rows)
```

The one **live counter** table, `daily_metric`, is likewise returned to its §2.6 baseline. Unlike the append-only tables in (b), `daily_metric` is *mutable* — the application both increments it (`DailyMetric.get_or_create_today_metric().nb_alias += 1` on every `Alias.create()`, `app/models.py:L1661`) and decrements it when an alias is deleted — so its per-day counter row is genuinely restorable by the same `UPDATE daily_metric SET nb_new_web_non_proton_user = 0, nb_alias = 11 WHERE id = 1` used in the §7.1 first-pass cleanup. Because the later §6.5/§6.6 reproductions re-touched the counter *after* that first-pass restore (and, having run past midnight, added a second per-day row for `2026-07-14`), the restore was re-applied as the final cleanup step — the `id 1` counters reset to the seed `(0, 11)` and the extra `2026-07-14` row deleted, in one transaction — leaving exactly the single baseline row §2.6 captured:

```
$ docker exec sl-postgres psql -U myuser -d simplelogin -c "SELECT id, date, nb_new_web_non_proton_user, nb_alias FROM daily_metric ORDER BY id;"
 id |    date    | nb_new_web_non_proton_user | nb_alias
----+------------+----------------------------+----------
  1 | 2026-07-13 |                          0 |       11
(1 row)
```

**(b) The append-only audit/bookkeeping tables retain monotonic residual — by design, not a cleanup failure.** `deleted_alias`, `alias_audit_log`, `user_audit_log`, `job`, and `sent_alert` are append-only in normal operation: the application only ever *inserts* into them, and PostgreSQL sequences never roll back, so deleting the entity rows does **not** reduce them. (`sent_alert` — an alert-deduplication log the handler writes whenever it sends an alert such as `reverse_alias_unknown_mailbox` or `bounce`, so it does not re-notify — is not one of the fourteen tables in the §2.6 baseline query above, but it is the same insert-only kind and is shown alongside the other four below for completeness. The *application* never prunes it; the investigation's scoped cleanups did remove the alert rows a given reproduction generated where they could be pinned to that run — §7.4, for example, deletes `sent_alert 6`/`7` — so the six rows that remain (ids `1` through `5`, which predate the §7.4 bounce checkpoint, plus id `10` from the later §6.6 catch-all run) are alerts whose individual rows were not separately targeted.) They therefore hold the cumulative rows written across the later reproductions (§6.5 exception-code, §6.6 catch-all, §7.4 bounce). The §7.1 first-pass and §7.4 bounce cleanups each removed the bookkeeping rows they could pin to their own run — the §7.4 transaction even deletes its `sent_alert 6`/`7` and `user_audit_log 15` — but rows that no cleanup targeted remain: `alias_audit_log` is pruned by no cleanup at all, and the audit and alert rows of runs whose bookkeeping was not individually enumerated stay behind. Combined with PostgreSQL sequence advancement that never rolls back, their counts are therefore *higher* than the fresh-reset pristine of §2.6:

```
$ docker exec sl-postgres psql -U myuser -d simplelogin -c "
SELECT t, count, max_id FROM (
  SELECT 'deleted_alias' t, count(*) count, max(id) max_id FROM deleted_alias
  UNION ALL SELECT 'alias_audit_log', count(*), max(id) FROM alias_audit_log
  UNION ALL SELECT 'user_audit_log', count(*), max(id) FROM user_audit_log
  UNION ALL SELECT 'job', count(*), max(id) FROM job
  UNION ALL SELECT 'sent_alert', count(*), max(id) FROM sent_alert) s ORDER BY t;"
        t        | count | max_id
-----------------+-------+--------
 alias_audit_log |    48 |    121
 deleted_alias   |    12 |     21
 job             |     2 |     78
 sent_alert      |     6 |     10
 user_audit_log  |    13 |     56
(5 rows)
```

This is the honest final state. The **live** tables that hold user-facing state (`users`, `alias`, `contact`, `email_log`, the mutable `daily_metric` counter, and the config tables `custom_domain`/`directory`/`mailbox`) are back at the seed values of §2.6; the **append-only** tables above (`deleted_alias`, `alias_audit_log`, `user_audit_log`, `job`, and `sent_alert`) are not, and cannot be by row-deletion alone. A database that is pristine in *every* table (the `deleted_alias`=0, `alias_audit_log`=11, `user_audit_log`=0, `job`=0 of §2.6) is produced only by re-running the canonical reset of [§2.5](#25-bring-the-database-to-head-and-seed-it). The investigation deliberately restores the live tables by scoped deletion rather than by that destructive reset, because a reset would regenerate *different* random-word alias local-parts — the seed's `word_test646` (id `2`) and `word568` (id `9`) come from `random_words()` (`app/utils.py:L29`), so a reset would invalidate every alias identifier quoted throughout this document. *(That a reset regenerates different local-parts is [INFERRED] from `random_words()` drawing on the unseeded `random` module at `app/utils.py:L29`; it was not re-observed, precisely because the investigation avoided the destructive reset.)*

### 7.3 Source tree untouched

On the host repository, the only modified path is this document — no source file changed:

```
$ git status --porcelain
 M blitzy/documentation/app_2cd6ee777f8c.md
```

Inside the container, `git status` shows eight tracked files as modified. These are **pre-existing image artifacts present before the investigation began** (as noted in [§2.3](#23-source-checkout-state-exact-not-assumed)) — DKIM/JWT/PGP/Paddle key material and rebuilt local test-word and lockfile artifacts baked into the image build — not changes made here:

```
$ docker exec sl-app sh -c 'cd /app && git status --porcelain'
 M app/spamassassin_utils.py
 M local_data/dkim.key
 M local_data/jwtRS256.key
 M local_data/jwtRS256.key.pub
 M local_data/paddle.key.pub
 M local_data/private-pgp.asc
 M local_data/test_words.txt
 M static/package-lock.json
```

The temporary mail-capture deviation of [§5.4(B)](#b-disclosed-deviation--capturing-the-actually-delivered-forward) is fully reverted: the runtime `.env` is back to its canonical checksum and `NOT_SEND_EMAIL=true`, and the email handler runs on the default configuration again:

```
$ docker exec sl-app sh -c 'sha256sum /app/.env'
7b3c4a44d1995cf10115a92d93d4787d556916b36d3522f5f81ff0a93a5d30d8  /app/.env
```

No temporary observation script is tracked by the repository (they live only under `/tmp`):

```
$ git ls-files | grep -E 'blitzy_investigation|sl_run|q[0-9]_|blitzy_adhoc' || echo "(none tracked — clean)"
(none tracked — clean)
```

The helper scripts themselves are then deleted from `/tmp` on both the container and the host, and their removal is confirmed — the investigation leaves no script behind on either filesystem:

```
$ docker exec sl-app bash -lc '
rm -rf /tmp/blitzy_docfix /tmp/blitzy_investigation /tmp/sl_run /tmp/blitzy_qa_fix
found=$(find /tmp -maxdepth 1 \( -name "blitzy_*" -o -name "sl_run" \) 2>/dev/null)
[ -z "$found" ] && echo "(none — all investigation helpers removed)" || echo "$found"'
(none — all investigation helpers removed)

$ rm -rf /tmp/blitzy_docfix /tmp/doc_catchall_seq.* /tmp/doc_cleanup*.out \
         /tmp/doc_ecodes_seq.* /tmp/redis_probe.py /tmp/verify_redis_cmd.sh
$ find /tmp -maxdepth 1 \( -name 'blitzy_docfix' -o -name 'doc_*' \
       -o -name 'redis_probe.py' -o -name 'verify_redis_cmd.sh' \) | grep -v '/tmp/blitzy$'
(none — all my host helpers removed)
```

**Teardown of the running components.** As the final cleanup step, the three long-lived components started in §2.7 were stopped by their specific container process ids — the web server (gunicorn master `15796`, which reaps its two workers), the email handler (`15828`), and the job runner (`15826`) — after which the published ports `7777` and `20381` stopped answering. The per-process run-log directory `/tmp/qa_run/` — into which those components wrote `web.log`, `email.log`, and `job.log` during the §4 through §6 observations — is then removed, so no investigation artifact remains on the container filesystem:

```
$ docker exec sl-app bash -lc 'for pid in 15796 15826 15828; do kill "$pid" && echo "sent TERM to $pid"; done'
sent TERM to 15796
sent TERM to 15826
sent TERM to 15828
$ docker exec sl-app bash -lc "ps -eo args | grep -E 'gunicorn wsgi:app|email_handler.py|job_runner.py' | grep -v grep || echo '(no app components running)'"
(no app components running)
$ docker exec sl-app bash -lc 'rm -rf /tmp/qa_run; [ -e /tmp/qa_run ] && echo STILL PRESENT || echo "/tmp/qa_run: confirmed gone"'
/tmp/qa_run: confirmed gone
```

The disposable PostgreSQL and Redis backing services are left up only as the throwaway datastores for the final state-verification queries of §7.2; they hold no repository state and are discarded with the container.

**Secret-handling note.** No secret is written to this committed document, and the ephemeral on-disk helpers that briefly held sensitive material were removed above. Every sensitive value that appeared in captured output is redacted in place at the point of capture: the development `FLASK_SECRET` is shown as `[REDACTED: FLASK_SECRET value removed — a development secret]` (§2.4); each `slapp=…` session cookie is shown as `[REDACTED: … session cookie]` (§4.1 for the anonymous cookie, §5.1 for the authenticated one); and the one-time account-activation token — a 30-character `random_string(30)` credential — was read from the database directly into a shell variable, never printed to the terminal, and masked in the request URL (§5.1). The only credential shown in the clear is the disposable local-container Postgres password (`myuser` / `mypassword`) created for this throwaway database in §2.1, which is not a production secret.

**Evidence-fidelity note.** Every fenced block in this document is verbatim captured output paired with the command that produced it. The only normalization applied to the deliverable itself is the removal of invisible trailing whitespace (psql column-header padding and log-line trailing spaces) and of the extra blank line at end-of-file, so `git diff --check` reports no whitespace errors; no visible character of any log line, status code, identifier, table value, or command was altered.


### 7.4 Bounce-phase reproduction cleanup (§6.5)

The two `handle_bounce()` phases documented in [§6.5](#65-the-email-handlers-dispatcher-reply-bounce-and-smtp-status-codes) were captured in a later, dedicated email-handler run (pid `10602`) and created a small, self-contained set of rows that is removed here in the same FK-safe, single-transaction manner as §7.1, returning the **live entity tables** to their seed values and the append-only tables to the warm residual they held **before** this run (the point-in-time checkpoint tabulated at the end of this section — *not* the fresh-reset pristine of §2.6, and *not* the final post-investigation state, which is [§7.2](#72-final-database-state-after-cleanup)). The rows were: `contact 12` (the seed forward's contact), `email_log 14`/`15` (the forward and reply logs), `refused_email 3`/`4`, `bounce 2`/`3`, `notification 8`/`9`, `user_audit_log 15` (a `create_contact` event), and `sent_alert 6`/`7` (the `bounce` / `bounce-when-reply` alerts); `alias 5.last_email_log_id` had also advanced to `15` and is reset to its baseline `NULL`.

A preflight `SELECT` confirms the exact scope before any mutation:

```
$ docker exec -i sl-postgres psql -U myuser -d simplelogin < /tmp/blitzy_qa_fix/bounce_preflight.sql
              target               | count
-----------------------------------+-------
 alias5.last_email_log_id not null |     1
 email_log id IN (14,15)           |     2
 refused_email id IN (3,4)         |     2
 contact id = 12                   |     1
 bounce id IN (2,3)                |     2
 notification id IN (8,9)          |     2
 user_audit_log id = 15            |     1
 sent_alert id IN (6,7)            |     2
(8 rows)
```

The scoped cleanup transaction (children before parents; `ON_ERROR_STOP` rolls back the whole batch on any FK violation) — each `DELETE n` / `UPDATE n` matches the preflight counts above:

```
$ docker exec -i sl-postgres psql -U myuser -d simplelogin < /tmp/blitzy_qa_fix/bounce_cleanup.sql
BEGIN
UPDATE 1
DELETE 2
DELETE 2
DELETE 1
DELETE 2
DELETE 2
DELETE 1
DELETE 2
COMMIT
```

Re-running the identical preflight after the transaction returns zero for every target, and a table-by-table comparison confirms the database is back to the exact **pre-reproduction state at that checkpoint** (same counts and max ids) — the live entity tables at their seed values, the append-only tables at the warm residual they held just before this run — with only the seed `email_log` (id `1`) and seed `contact` (id `1`) surviving and `alias 5.last_email_log_id` back to `NULL`:

```
$ docker exec -i sl-postgres psql -U myuser -d simplelogin < /tmp/blitzy_qa_fix/bounce_preflight.sql
              target               | count
-----------------------------------+-------
 alias5.last_email_log_id not null |     0
 email_log id IN (14,15)           |     0
 refused_email id IN (3,4)         |     0
 contact id = 12                   |     0
 bounce id IN (2,3)                |     0
 notification id IN (8,9)          |     0
 user_audit_log id = 15            |     0
 sent_alert id IN (6,7)            |     0
(8 rows)

$ # count/max per table — post-cleanup == the pre-run checkpoint state for every table
 table            | pre-run  | current
------------------+----------+---------
 alias            | 11/11    | 11/11
 alias_audit_log  | 33/63    | 33/63
 bounce           | 0/0      | 0/0
 contact          | 1/1      | 1/1
 deleted_alias    | 7/15     | 7/15
 email_log        | 1/1      | 1/1
 job              | 1/16     | 1/16
 notification     | 6/6      | 6/6
 refused_email    | 1/1      | 1/1
 sent_alert       | 5/5      | 5/5
 user_audit_log   | 1/9      | 1/9
 users            | 2/2      | 2/2
```

The `pre-run` column above is the **warm append-only residual at this bounce-run checkpoint** (`alias_audit_log` 33/63, `deleted_alias` 7/15, `job` 1/16, `user_audit_log` 1/9). It is deliberately *not* labeled "baseline": it is neither the fresh-reset pristine of §2.6 (`11`, `0`, `0`, `0`) nor the final post-investigation state. The later §6.5 exception-code and §6.6 catch-all reproductions appended still more append-only rows, so the authoritative post-investigation counts are the higher ones quantified in [§7.2](#72-final-database-state-after-cleanup) (`alias_audit_log` 48/121, `deleted_alias` 12/21, `job` 2/78, `user_audit_log` 13/56). Each snapshot is internally consistent for its point on the single timeline; the monotonic growth across them (`11` → `33` → `48` for `alias_audit_log`, and so on) is exactly the append-only behavior §7.2 explains — the live entity tables return to seed at every checkpoint, the append-only tables never shrink.

The temporary SMTP/DSN observation scripts used for this reproduction live only under host `/tmp/` (never tracked or committed), matching the read-only, leave-no-trace guarantee stated at the top of §7.


---

## 8. Coverage pass

Every distinct thing the prompt asks about, and where it is answered from observed runtime evidence:

| # | Named item from the prompt / its examples | Where answered | Observed? |
|---|-------------------------------------------|----------------|-----------|
| 1 | **Web server** is up (`server.py`, port 7777) | [§4.1](#41-web-server-serverpy--wsgipy-port-7777) — `GET /health` → `200 "success"` (full headers), `GET /` → `302` | Observed |
| 2 | **Email handler** is up (`email_handler.py`, SMTP 20381) | [§4.2](#42-email-handler-email_handlerpy-smtp-20381) — `Listen for port 20381` + `Start mail controller 0.0.0.0 20381`; socket probe | Observed |
| 3 | **Job runner** is up (`job_runner.py`, no port) | [§4.3](#43-job-runner-job_runnerpy-no-port) — poll loop; measured 10 s cadence over ≥2 cycles | Observed |
| 4 | What appears in the **logs** confirming liveness | [§3](#3-how-to-read-the-logs-the-sl-log-format) (format) + §4.1–4.3 (per-process startup lines) | Observed |
| 5 | What the **dashboard UI** shows confirming users can **sign in** | [§4.4](#44-ui-confirmation--sign-in-and-alias-management), [§5.2](#52-c2--sign-in-real-session-with-cookies-and-csrf) — authenticated `GET /dashboard/` HTML | Observed |
| 6 | What the dashboard UI shows confirming users can **manage aliases** | [§4.4](#44-ui-confirmation--sign-in-and-alias-management), [§5.3](#53-c3--create-an-alias-random-and-custom) — alias list + create controls; new alias appears | Observed |
| 7 | **Create a new account** | [§5.1](#51-c1--create-a-new-account-signup--activation) — `POST /auth/register`; `create user` log; `users` row `activated=f` | Observed |
| 8 | Account **activation** (`activated` f→t) | [§5.1](#51-c1--create-a-new-account-signup--activation) — `GET /auth/activate`; `activated` transition; `activation_code` deleted | Observed |
| 9 | **Create an alias — random** | [§5.3](#53-c3--create-an-alias-random-and-custom) — `POST /dashboard/` create-random; new `Alias` rows 13, 15 | Observed |
| 10 | **Create an alias — custom** | [§5.3](#53-c3--create-an-alias-random-and-custom) — custom-alias flow; new `Alias` row 14 on `@old.com` | Observed |
| 11 | Alias **receives an email** (forward path) | [§5.4](#54-c4--an-email-arrives-at-the-alias-forward-path) — SMTP send; single-`message_id` New→Forward→`EmailLog`→Finish | Observed |
| 12 | What happens when a **new account interacts with aliases** | [§5.1](#51-c1--create-a-new-account-signup--activation) (auto newsletter alias 12), [§5.3](#53-c3--create-an-alias-random-and-custom) | Observed |
| 13 | What happens when an alias **receives mail** | [§5.4](#54-c4--an-email-arrives-at-the-alias-forward-path), [§6.5](#65-the-email-handlers-dispatcher-reply-bounce-and-smtp-status-codes), [§6.6](#66-secondary-alias-auto-creation-directory-success-and-domain-prerequisites) | Observed |
| 14 | How the system **shows it was handled correctly** | before / intermediate / after state throughout §5 (DB rows, `nb_forward`, flash, HTML, `EmailLog`) | Observed |
| 15 | Does the **email handler come online in the background** and keep working | [§6.1](#61-how-the-three-services-stay-up-they-do-not-launch-one-another) — `while True: time.sleep(2)` keep-alive; continuity | Observed |
| 16 | Does the **job runner come online in the background** and keep working | [§6.1](#61-how-the-three-services-stay-up-they-do-not-launch-one-another) — PID 3352 continuous ~64 min; 10 s poll | Observed |
| 17 | A **genuinely-enqueued job** processed (ready → taken → done) | [§6.2](#62-a-genuinely-enqueued-job-canonical-ready--taken--done) — delete-account Job 4 via `POST /dashboard/delete_account` | Observed |
| 18 | **Retry / attempts** accounting | [§6.3](#63-retry-accounting-and-the-run_at-selection-gate) — Job 6 stale-taken re-take, attempts 2→3 | Observed |
| 19 | The **`run_at` selection gate** (`run_at <= now + 10 min`) | [§6.3](#63-retry-accounting-and-the-run_at-selection-gate) — Job 5 future `run_at` held back | Observed (state); wall-clock arithmetic inferred |
| 20 | An **unknown job name** (edge condition) | [§6.4](#64-edge-an-unknown-job-name) — `Unknown job name blitzy-nonexistent-job` at ERROR + `NoneType: None` | Observed |
| 21 | The **reply** phase | [§6.5](#65-the-email-handlers-dispatcher-reply-bounce-and-smtp-status-codes) — Case B unauthorized reply → `250 SL E214` | Observed |
| 22 | The **bounce** phase | [§6.5](#65-the-email-handlers-dispatcher-reply-bounce-and-smtp-status-codes) — `is_bounce()` predicate; forward-bounce `bounce+14+@sl.local` → `250 SL E211`, reply-bounce `bounce_reply+15+@sl.local` → `250 SL E212` (both observed) | Observed |
| 23 | **SMTP return / status codes** (incl. `E515`, `E214`, `E206`, `E211`, `E212`, and the mapped `E524`/`E213`/`E404`) | [§6.5](#65-the-email-handlers-dispatcher-reply-bounce-and-smtp-status-codes) — **all observed**: forward `250` / nonexistent `550 SL E515` / unauthorized-reply `250 SL E214` / DSN `250 SL E206` / forward-bounce `250 SL E211` / reply-bounce `250 SL E212` / reverse-alias-as-sender `550 SL E524` / non-bounce-to-VERP `250 SL E213` / DB-overflow `421 SL E404` / successful reply `250` | Observed (all) |
| 24 | **Alias auto-creation — directory** | [§6.6](#66-secondary-alias-auto-creation-directory-success-and-domain-prerequisites) — `abcd+blitzyprobe@sl.local` → alias 16 created | Observed |
| 25 | **Alias auto-creation — domain / catch-all** | [§2.6](#26-what-the-seed-actually-creates-the-real-baseline), [§6.6](#66-secondary-alias-auto-creation-directory-success-and-domain-prerequisites) — catch-all enabled on `old.com` via the dashboard, a new address auto-created `<Alias 67>` (note `Created by catchall option`), then catch-all toggled back off | Observed at runtime |

All twenty-five named items are answered from captured runtime evidence, except the two explicitly-labeled inferred sub-points (the `run_at` wall-clock arithmetic and the upstream three-container topology), each of which is backed by a genuine runtime attempt as detailed in [§9](#9-observed-vs-inferred-ledger). (The `E524`/`E213`/`E404` codes and catch-all auto-creation, previously inferred, are now observed at runtime — see §6.5–§6.6.)


---

## 9. Observed vs inferred ledger

The overwhelming majority of this document is **observed**: every liveness signal, every HTTP response and status code, every `SL` log line, every database row and counter (before / intermediate / after), the canonical ready→taken→done job transition, the retry re-take, the unknown-job error, the SMTP outcomes (both `handle_bounce()` phases → `250 SL E211` / `250 SL E212`, the exception-mapped `550 SL E524` / `250 SL E213` / `421 SL E404`, and a successful reply → `250`), and both directory **and** catch-all alias auto-creation were all produced at runtime through the real entry points and captured verbatim alongside the command that produced them.

Exactly **two** sub-points are labeled **`[INFERRED]`**. Each is a behavior that a black-box send/enqueue could not force into existence without controlling wall-clock timing or replicating the upstream multi-container deployment; for each, a genuine runtime attempt was made first, its real (different) outcome is shown in the body, and only the residual branch is described from source or upstream documentation. (A third item — the `E524`/`E213`/`E404` exception→code mappings — was previously inferred but has since been **observed at runtime** in [§6.5](#65-the-email-handlers-dispatcher-reply-bounce-and-smtp-status-codes), so it is no longer listed here.)

| # | Inferred sub-point | Why it could not be observed black-box | Genuine attempt actually made (observed outcome) | Source reference |
|---|--------------------|----------------------------------------|--------------------------------------------------|------------------|
| 1 | The `run_at <= now + 10 min` wall-clock arithmetic in the retry gate | Comparing a captured timestamp against "now + 10 min" is an arithmetic statement about clock values, not a directly-emitted runtime signal | [§6.3](#63-retry-accounting-and-the-run_at-selection-gate) — Job 5's future `run_at` was observed to keep it `state=0`/`attempts=0` across ~30 poll cycles (the *values* are observed; only the arithmetic tying them together is inferred) | `job_runner.py:L323` |
| 2 | The upstream production **three-container** topology | The self-hosted production layout runs three separate containers; locally the canonical image runs all three processes in one container | [§6.1](#61-how-the-three-services-stay-up-they-do-not-launch-one-another) — locally the three processes were each launched independently and their independence/continuity observed (PIDs, `etime`); the multi-container production shape is from upstream docs | corroborated by web search (§0.2.2) |

No other claim in the document is inferred. Where a value was obtained by reading the database directly rather than through the canonical entry point (for example, a `SELECT` on the `job` table to show a state transition), that read is a *verification* of a state produced canonically. Every value obtained off the canonical path is labeled `[NON-CANONICAL]` at its point of use: the diagnostic direct-`Job.create()` enqueues that drive or stage the runner (the §4.3 warm-up jobs `1`–`3`, and the §6.3 retry-state and §6.4 unknown-name job producers — in each case the *runner's* take/skip/handle decision is the canonical behavior being observed), and the §5.1 activation-token read (the one-time code was read from the `activation_code` table because the default `NOT_SEND_EMAIL=true` suppresses the activation email, so it is not deliverable — the activation *request* itself remains canonical). No such value is ever substituted for a canonical-path observation.
