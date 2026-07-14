# SimpleLogin Backend — DEV-Mode Runtime Behavior (Observed)

This document answers seven questions about how the **SimpleLogin** email-aliasing backend
(a Python/Flask email service) behaves when run **locally in DEVELOPMENT mode**. Every answer
is written from **observed runtime output** captured by actually building and running the code
through its canonical entry point (`python3 server.py`) — not from reading source alone. Each
question is answered in the mandated five-stage order: **(1) direct answer → (2) command(s) run
→ (3) complete output (secret-masked only where a `<REDACTED:…>` marker is shown) → (4) `file:line`
grounding → (5) Observed / Inferred label.**

> **Evidence discipline.** Every fenced output block below is the stdout/stderr of the command
> shown immediately above it, reproduced from a capture file. Nothing inside an output block is
> truncated, re-ordered, paraphrased, or summarized, and all interpretation lives *outside* the
> blocks; the **only** in-block modifications are the **two byte-level normalizations** defined
> next (and nothing else). Consequently, a block that contains a `<REDACTED:…>` marker is
> **complete and structurally verbatim but secret-masked** — its header is labelled *"Complete
> output (secret-masked)"* rather than *"unedited"* — whereas blocks with no marker are genuinely
> unedited. **(1) Line endings:**
> `curl -i` HTTP responses are **CRLF-terminated on the wire** (RFC 7230) and are shown here
> **LF-normalized** — the status line, headers and body are otherwise verbatim. **(2) Secret
> masking:** the random secret *bytes* of security-sensitive fields (session-cookie HMAC signatures,
> the signed-cookie payload blob, acquired API-key values, and CSRF tokens) are replaced in place
> with a labelled `<REDACTED:…>` marker while every surrounding structural and non-secret decoded
> field is preserved verbatim — see the **Secret-handling convention** note below. Where a value can vary run-to-run (PIDs, timestamps, the session id, the elapsed
> `takes`, the Debugger PIN), the same unchanged path was exercised **at least twice** and the
> distribution is reported honestly (stable vs variable). Runs are labelled with **Run IDs**
> (RUN‑A, RUN‑B, RUN‑Q5‑REDIS, RUN‑Q5‑COOKIE‑1/2, RUN‑Q6, RUN‑Q7) so HTTP responses, Redis keys,
> DB rows and logger lines can be correlated to the exact server process that produced them.

> **Security scope (read this first).** Every credential, key and setting shown in this document
> — the seed login `john@wick.com` / `password`, `FLASK_SECRET=secret`, the DB credentials
> `test:test`, the seeded API keys `code` / `codeFF`, session cookies, `DISABLE_RATE_LIMIT=1`,
> `debug=True`, and `OAUTHLIB_INSECURE_TRANSPORT=1` — is **disposable, DEVELOPMENT-only**, created
> by `flask dummy-data` in a throwaway local container. **None of it is production guidance.** Each
> sensitive value is re-flagged as DEV-only beside its use below.

> **Secret-handling convention (mask-before-emission).** Although every value shown here is a
> disposable DEV-only throwaway (each captured session was flushed and each acquired API key was
> revoked at cleanup — proven in *Cleanup and read-only guarantee*), this document does **not**
> persist raw secret key material. In every evidence block the random secret *bytes* are masked in
> place with a labelled marker, while all structure and every non-secret field is shown verbatim so
> that no evidentiary value is lost:
>
> - **Session-cookie HMAC signature** — the `slapp` cookie value `session-id.signature` keeps the
>   real session-id (needed to prove the id is retained across a login and to correlate with the
>   Redis `session:<id>` key) and masks the signature as `<REDACTED:slapp-hmac-signature>`.
> - **Signed-cookie payload** — the fallback-backend cookie `.payload.timestamp.signature` keeps the
>   leading `.eJw` zlib marker and the real timestamp segment, and masks the compressed payload
>   (`<REDACTED:zlib+base64url-session-payload>`) and signature (`<REDACTED:slapp-hmac-signature>`);
>   the *decoded* JSON is shown below each blob with only its `csrf_token` masked, which still proves
>   the payload decodes **without any secret key** (⇒ signed-not-encrypted) and still shows the
>   `_user_id`/`_id`/`_fresh`/`_permanent`/`sudo_time` fields.
> - **Acquired API key** — the 60-character value returned by `POST /api/auth/login` is masked as
>   `<REDACTED:api_key-value>` (these keys are revoked at cleanup and never re-used in a later
>   request; the calls below bind the key via a shell variable or use the seeded `code` key).
> - **CSRF token** — the 40-hex `csrf_token` value is masked as `<REDACTED:csrf_token-value>`.
>
> **Retained verbatim** (not secrets, required as evidence): the session-id UUID, `_user_id`
> (= the user's `alternative_id`), `_id` (a SHA-512 of `remote_addr|user_agent`, not secret-derived),
> `_fresh`, `_permanent`, `sudo_time`, the signed-cookie timestamp, all cookie flags
> (`HttpOnly`/`Path`/`SameSite`/`Expires`/`Max-Age`), all HTTP statuses/headers/bodies, all `SL` log
> lines, and the **public, non-bearer DEV defaults**: the seed login `john@wick.com` / `password`,
> `FLASK_SECRET=secret`, the DB credentials `test:test`, and the seed API keys `code` / `codeFF`.
> These defaults are not private secrets — they are published verbatim in the project's own
> `example.env`, `CONTRIBUTING.md`, and the container's setup script — so masking them would remove
> required Q2/Q5 configuration-and-identity evidence (the AAP mandates showing the actual observed
> identity values, e.g. proving `session["_user_id"]` **equals** the user's `alternative_id`) while
> adding no security value, because none of them is a capturable bearer token. This is a deliberate
> policy, not an oversight: it reconciles the run-first *complete-evidence* rule with
> mask-before-emission — **only capturable bearer material** (session-cookie HMAC signatures, the
> signed-cookie payload blob, acquired 60-char API-key values, CSRF tokens, and the
> `RECOVERY_CODE_HMAC_SECRET`, which is **never printed anywhere** in this document) is hidden;
> opaque random bytes carry no evidentiary meaning, whereas the public defaults and identity UUIDs
> above do and are therefore kept.

## Investigation environment

- **Application.** The SimpleLogin webapp object is built by the application factory
  `create_app()` in `server.py` and served in production by `wsgi.py`
  (`from server import create_app` / `app = create_app()`, `wsgi.py:L1,L3`). The dev entry point is
  `local_main()` (`server.py:L572-595`), reached by `if __name__ == "__main__": local_main()`
  (`server.py:L598-599`) when running `python3 server.py`.

- **Branch / commit.** The active branch checkout is at detached `HEAD`
  `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`; the deliverable filename `app_2cd6ee777f8c.md` is
  locked to the branch name. Command and complete output:

  ```
  $ git rev-parse HEAD; git log -1 --oneline
  ```
  ```
  git rev-parse HEAD:
  2cd6ee777f8c2d3531559588bcfb18627ffb5d2c
  git log -1 --oneline:
  2cd6ee77 chore: emit some missing contact audit logs (#2269)
  ```

- **Runtime.** All capture was performed inside the provisioned Docker container
  (`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`). The interpreter is the
  project virtualenv **`/app/venv`** — the *system* `python3` has no Flask, so every command uses
  the venv interpreter/tools (`/app/venv/bin/python`, `/app/venv/bin/alembic`,
  `/app/venv/bin/flask`). PostgreSQL and Redis run locally in the container. The webapp binds
  loopback `127.0.0.1:7777` inside the container, so HTTP is exercised with
  `docker exec ... curl http://localhost:7777/...`.

- **Datastore provenance** (complete output):

  ```
  $ pg_isready
  $ psql -h localhost -U test -d test -t -c "select version()"
  ```
  ```
  $ pg_isready
  /var/run/postgresql:5432 - accepting connections

  $ psql -c "select version()"
   PostgreSQL 15.13 (Debian 15.13-0+deb12u1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 12.2.0-14+deb12u1) 12.2.0, 64-bit
  ```
  ```
  $ redis-cli ping
  $ redis-cli INFO server | grep redis_version
  ```
  ```
  $ redis-cli ping
  PONG
  $ redis-cli INFO server | grep redis_version
  redis_version:7.0.15
  ```

- **Pinned stack** (observed exact versions):

  ```
  $ /app/venv/bin/python -c "import flask, flask_login, sqlalchemy, redis, coloredlogs, werkzeug, dotenv, alembic, sys; \
      print('python      ', sys.version.split()[0]); print('flask       ', flask.__version__); \
      print('werkzeug    ', werkzeug.__version__); print('flask_login ', flask_login.__version__); \
      print('sqlalchemy  ', sqlalchemy.__version__); print('redis       ', redis.__version__); \
      print('coloredlogs ', coloredlogs.__version__); print('alembic     ', alembic.__version__)"
  $ /app/venv/bin/pip show python-dotenv | grep -i '^Version'
  ```
  ```
  python       3.10.18
  flask        1.1.2
  werkzeug     1.0.1
  flask_login  0.5.0
  sqlalchemy   1.3.24
  redis        4.6.0
  coloredlogs  14.0
  alembic      1.4.3
  python-dotenv Version: 0.14.0
  ```

  Werkzeug **1.0.1** is the transitive pin under Flask 1.1.2 (`pyproject.toml`: `flask = "^1.1.2"`
  L62, `SQLAlchemy = "1.3.24"` L116, `python-dotenv = "^0.14.0"` L68, `coloredlogs = "^14.0"` L89,
  `redis = "^4.5.3"` L117). The pinned Flask/Werkzeug pair is what makes the Q3 readiness question
  version-sensitive.

- **Canonical bootstrap** (per `CONTRIBUTING.md:L106`). Every command is prefixed with the
  container's mandated env sourcing, which exports config into the process environment before the
  app is imported:

  ```
  $ cd /app && set -a && . /tmp/sl_env.sh && set +a && unset EVENT_WEBHOOK_DISABLE && export GNUPGHOME=/tmp/sl_clean_gnupg
  $ /app/venv/bin/alembic upgrade head      # apply migrations
  $ /app/venv/bin/flask  dummy-data          # seed dev data (john@wick.com / password) — DEV-only
  ```

  Migrations were applied from an empty schema (261 lines of migration output — 255 `Running
  upgrade …` steps — ending at revision `32f25cbf12f6`). The idempotent re-run and `alembic current` confirm the DB is at
  head — complete output of the exact venv command:

  ```
  $ /app/venv/bin/alembic upgrade head
  ```
  ```
  >>> URL: http://localhost
  Upload files to local dir
  >>> init logging <<<
  2026-07-13 18:18:17,051 - SL - DEBUG - 4868 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
  INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
  INFO  [alembic.runtime.migration] Will assume transactional DDL.
  ```
  ```
  $ /app/venv/bin/alembic current
  ```
  ```
  >>> URL: http://localhost
  Upload files to local dir
  >>> init logging <<<
  2026-07-13 18:17:35,396 - SL - DEBUG - 4848 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
  INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
  INFO  [alembic.runtime.migration] Will assume transactional DDL.
  32f25cbf12f6 (head)
  ```

  The seed (`flask dummy-data` → `dummy_data()` `server.py:L491-497` → `fake_data()`) — complete
  unedited output:

  ```
  $ /app/venv/bin/flask dummy-data
  ```
  ```
  >>> URL: http://localhost
  Upload files to local dir
  >>> init logging <<<
  2026-07-13 18:17:32,798 - SL - DEBUG - 4831 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
  2026-07-13 18:17:33,621 - SL - WARNING - 4831 - "/app/server.py:494" - dummy_data() -  - reset db, add fake data
  2026-07-13 18:17:33,621 - SL - DEBUG - 4831 - "/app/app/fake_data.py:41" - fake_data() -  - create fake data
  2026-07-13 18:17:33,906 - SL - INFO - 4831 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
  2026-07-13 18:17:33,931 - SL - DEBUG - 4831 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email test_list620@sl.local
  2026-07-13 18:17:33,942 - SL - INFO - 4831 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
  2026-07-13 18:17:33,992 - SL - INFO - 4831 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
  2026-07-13 18:17:34,002 - SL - INFO - 4831 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
  2026-07-13 18:17:34,023 - SL - INFO - 4831 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
  2026-07-13 18:17:34,035 - SL - INFO - 4831 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
  2026-07-13 18:17:34,050 - SL - INFO - 4831 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
  2026-07-13 18:17:34,058 - SL - INFO - 4831 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
  2026-07-13 18:17:34,069 - SL - DEBUG - 4831 - "/app/app/models.py:1255" - generate_oauth_client_id() -  - generate oauth_client_id demo-lxoqekszui
  2026-07-13 18:17:34,075 - SL - DEBUG - 4831 - "/app/app/models.py:1255" - generate_oauth_client_id() -  - generate oauth_client_id demo2-lflynsksor
  2026-07-13 18:17:34,349 - SL - INFO - 4831 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
  2026-07-13 18:17:34,375 - SL - INFO - 4831 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
  2026-07-13 18:17:34,388 - SL - INFO - 4831 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
  2026-07-13 18:17:34,397 - SL - INFO - 4831 - "/app/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain
  ```

  Seed verification — the dev account and its API keys created by `fake_data()`
  (`app/fake_data.py:L44-54`, `is_admin=True`; API keys `app/fake_data.py:L121-125`). **DEV-only
  seed values:**

  ```
  $ psql -h localhost -U test -d test -x -c "select id,email,activated,disabled,is_admin,delete_on,enable_otp,alternative_id from users where email='john@wick.com'"
  $ psql -h localhost -U test -d test    -c "select id,user_id,name,code from api_key where user_id=1 order by id"
  ```
  ```
  -[ RECORD 1 ]--+-------------------------------------
  id             | 1
  email          | john@wick.com
  activated      | t
  disabled       | f
  is_admin       | t
  delete_on      | 
  enable_otp     | f
  alternative_id | bfb8024e-6c00-4c33-ac2c-e60bf3b7003d

   id | user_id |  name   |  code  
  ----+---------+---------+--------
    1 |       1 | Chrome  | code
    2 |       1 | Firefox | codeFF
  (2 rows)
  ```

  The freshly seeded `alternative_id` is **`bfb8024e-6c00-4c33-ac2c-e60bf3b7003d`**; it is used to
  verify web identity in Q5. (It is a fresh UUID4 per seed — `user.alternative_id = str(uuid.uuid4())`
  `app/models.py:L617` — so it is *stable within a seed* but *regenerated by a re-seed*.)

- **Effective, observed config** (canonical load with the env sourced). Note **`URL` resolves to
  `http://localhost`, not `http://localhost:7777`** — this is a real read-vs-run result explained
  in Q2 (the shell env var wins over `./.env` because `python-dotenv` uses `override=False`):

  ```
  $ /app/venv/bin/python -c "import app.config as c; print(repr(c.URL)); print(repr(c.SESSION_COOKIE_NAME)); print(repr(c.MEM_STORE_URI)); print(repr(c.DISABLE_RATE_LIMIT)); print(repr(c.DB_URI)); print(repr(c.DB_CONN_NAME))"
  ```
  ```
  >>> URL: http://localhost
  Upload files to local dir
  'http://localhost'
  'slapp'
  'redis://localhost'
  True
  'postgresql://test:test@localhost:5432/test'
  'webapp'
  ```

  `MEM_STORE_URI=redis://localhost` means the **server-side Redis session backend** is active for
  the primary runs; a separate run with `MEM_STORE_URI=""` exercises the signed-cookie backend
  (Q5). `.env`/`config` and the local dev DB are git-ignored, so none of this alters tracked source.

---

## Q1 — How does the backend start up in development mode?

### Direct answer (Observed)

Running `python3 server.py` executes the `if __name__ == "__main__": local_main()` guard
(`server.py:L598-599`). `local_main()` (`server.py:L572-595`) does, in order: sets
`config.COLOR_LOG = True` (L573) — which, as Q3 proves, has **no retroactive effect** on the
already-built `SL` logger; builds the app with the factory `create_app()` (L574); enables the
Flask-DebugToolbar (`from flask_debugtoolbar import DebugToolbarExtension` L577;
`DebugToolbarExtension(app)` L582); sets `app.debug = True` (L581); and calls
`app.run(debug=True, port=7777)` (L588). Because `debug=True` and `use_reloader` is not disabled,
Werkzeug's **stat reloader** forks a serving child, so the whole module — including its import-time
`print` banners — is imported **twice** (reloader parent, then serving child). The server then
binds **`127.0.0.1:7777`** and serves via Werkzeug's development WSGI server. Flask labels the run
`Environment: production` even with debug on (explained in the Observed section). Production is
different: `wsgi.py` exposes `app = create_app()` served by gunicorn (`Dockerfile:L47`) — that path
is **Inferred** (not run here).

### Command(s) run

```
$ cd /app && set -a && . /tmp/sl_env.sh && set +a && unset EVENT_WEBHOOK_DISABLE && export GNUPGHOME=/tmp/sl_clean_gnupg
# bounded lifecycle (one shell): launch, wait 5s for BOTH reloader imports (NO request yet),
# snapshot the PURE startup stream, then probe /health, then terminate parent+child by PID.
$ /app/venv/bin/python server.py >/tmp/q1.log 2>&1 &   PARENT=$!
$ sleep 5 ; cp /tmp/q1.log q1_startup.log              # pure startup, before any HTTP
$ ps -eo pid,ppid,args | grep "[s]erver.py"
$ curl -s -i http://localhost:7777/health              # first HTTP request (readiness)
$ kill -TERM $CHILD $PARENT ; kill -KILL $CHILD $PARENT
```

### Complete, unedited output (RUN‑A)

**Pure startup stream** — the complete stdout+stderr captured **before any HTTP request was made**
(so nothing here is request-triggered). Every banner block appears **twice** (reloader parent PID
5058, then serving child PID 5070):

```
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-13 18:23:21,160 - SL - DEBUG - 5058 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-13 18:23:22,753 - SL - DEBUG - 5070 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

The reloader parent and its serving child, confirmed as two separate OS processes (child PPID =
parent PID 5058):

```
   5058    5052 /app/venv/bin/python server.py
   5070    5058 /app/venv/bin/python /app/server.py
```

**Request-triggered output (NOT part of startup).** The Flask-DebugToolbar
`Could not insert debug toolbar. </body> tag not found` warning is emitted by the toolbar's
`after_request` processing of a *response*, not at startup. It is **absent** from the pure startup
stream above and appears **only after** the first HTTP response (`/health` returns the 7-byte body
`success`, which has no `</body>`). These are the exact two extra lines that appear after the
`/health` request:

```
/app/venv/lib/python3.10/site-packages/flask_debugtoolbar/__init__.py:213: UserWarning: Could not insert debug toolbar. </body> tag not found in response.
  warnings.warn('Could not insert debug toolbar.'
```

**Reproducibility (RUN‑A vs RUN‑B).** The pure startup stream is **identical across both runs
except the PIDs and the timestamps**: RUN‑A parent/child PIDs `5058`/`5070` at `18:23:21`/`18:23:22`;
RUN‑B parent/child PIDs `5101`/`5113` at `18:23:30`/`18:23:31`. Both are variable; the line content,
ordering and the double-import are stable. Both runs bound and then released `127.0.0.1:7777`
cleanly (no LISTEN socket after termination); and the resulting parent(`NLWP=1`)/child(`NLWP=2`)
thread topology then stays **flat for the life of the process** — see Q6, where two further runs
sample it at t=0/30/65 s and find it unchanged over a `>=60 s` window. The complete RUN‑B pure-startup transcript
(`q1_runB_startup.log`, captured before any HTTP request) and its process tree follow — compare
line-for-line with RUN‑A above; only the PID and timestamp fields change:

```
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-13 18:23:30,291 - SL - DEBUG - 5101 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-13 18:23:31,837 - SL - DEBUG - 5113 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

The two RUN‑B processes (child PPID = parent PID 5101):

```
   5101    5095 /app/venv/bin/python server.py
   5113    5101 /app/venv/bin/python /app/server.py
```

### File:line grounding

- Entry guard: `if __name__ == "__main__":` (`server.py:L598`) → `local_main()` (`server.py:L599`).
- `def local_main():` `server.py:L572`: `config.COLOR_LOG = True` (L573); `app = create_app()` (L574);
  `from flask_debugtoolbar import DebugToolbarExtension` (L577); `app.config["DEBUG_TB_PROFILER_ENABLED"] = True`
  (L579); `app.config["DEBUG_TB_INTERCEPT_REDIRECTS"] = False` (L580); `app.debug = True` (L581);
  `DebugToolbarExtension(app)` (L582); `app.run(debug=True, port=7777)` (L588).
- Factory `def create_app() -> Flask:` `server.py:L139`; the **webapp** `app = Flask(__name__)` is at
  `server.py:L140` (the `create_light_app()` app at L128 is a different, request-less app);
  `app.wsgi_app = ProxyFix(app.wsgi_app, x_for=1, x_host=1)` (L142); `app.url_map.strict_slashes = False`
  (L144); `app.secret_key = FLASK_SECRET` (L151); `register_blueprints(app)` (L172); `/health` (L213-215).
- Production contrast (**Inferred**, not run): `wsgi.py:L1` `from server import create_app`; `wsgi.py:L3`
  `app = create_app()`; `Dockerfile:L8` `FROM python:3.10`; `Dockerfile:L44` `EXPOSE 7777`; `Dockerfile:L47`
  `CMD ["gunicorn","wsgi:app","-b","0.0.0.0:7777","-w","2","--timeout","15"]`.

### Observed vs Inferred

- **Observed:** the full dev startup sequence and its ordering; the double import (reloader parent
  PID 5058 + serving child PID 5070 as separate processes); the `Environment: production` /
  `Debug mode: on` banner; that the DebugToolbar warning is *request-triggered* (absent from pure
  startup, present only after `/health`); the port-7777 bind (confirmed in Q3/Q4); and the
  PID/timestamp-only variance across two runs.
- **Inferred:** the production `gunicorn wsgi:app` binding `0.0.0.0:7777` — read from the
  `Dockerfile` and **not** run; the dev-vs-prod contrast is code-derived.

---

## Q2 — How is configuration loaded?

### Direct answer (Observed)

Configuration is resolved **entirely at import time** of `app/config.py` — the first SimpleLogin
module `server.py` imports (`server.py:L30` `from app import config, constants`). The module uses
**python-dotenv 0.14.0**. At `config.py:L65` it reads `config_file = os.environ.get("CONFIG")`; when
`CONFIG` **is set** it prints `load config file <abs-path>` (L68) and calls
`load_dotenv(get_abs_path(config_file))` (L69); when `CONFIG` **is not set** it silently calls
`load_dotenv()` (L71), which loads `./.env`. `load_dotenv` runs with dotenv's default
**`override=False`**, so any variable already present in the process environment **wins over** the
file. That is exactly why the canonical dev run observes **`URL=http://localhost`** (the value
exported by `/tmp/sl_env.sh`) even though `/app/.env:L53` sets `URL=http://localhost:7777` — the
shell/process value takes precedence and the file value is ignored.

Five variables are read with **bracket access** `os.environ["…"]` and therefore **hard-fail with
`KeyError`** when absent, evaluated in this top-to-bottom order: `URL` (L79), `EMAIL_DOMAIN` (L92),
`SUPPORT_EMAIL` (L93), `DB_URI` (L192), `FLASK_SECRET` (L196). A **present-but-empty** `FLASK_SECRET`
instead raises **`RuntimeError("FLASK_SECRET is empty. Please define it.")`** (L197-198). A sixth
value is also required but fails differently: `EMAIL_SERVERS_WITH_PRIORITY` (L172) is loaded via
`sl_getenv("EMAIL_SERVERS_WITH_PRIORITY")` with **no `default_factory`**, so when it is absent
`sl_getenv` calls `default_factory()` — i.e. `None()` — raising **`TypeError: 'NoneType' object is
not callable`** at `config.py:L33`.

### Command(s) run

```
# ---- DEFAULT (else) branch + override=False precedence: canonical env, no CONFIG ----
$ cd /app && set -a && . /tmp/sl_env.sh && set +a && unset EVENT_WEBHOOK_DISABLE && export GNUPGHOME=/tmp/sl_clean_gnupg
$ unset CONFIG
$ /app/venv/bin/python -c "import app.config"          # app.config is the exact module server.py imports at L30

# ---- CONFIG branch: a controlled file whose URL is a distinctive :9999 ----
# full.env = a byte-copy of /app/.env with ONLY the URL line changed to http://localhost:9999
$ env -i PATH=/usr/bin:/bin CONFIG=/tmp/blitzy_cap/q2/full.env \
      /app/venv/bin/python -c "import app.config; print('[import ok] app.config.URL =', app.config.URL)"

# ---- the two conflicting URL definitions (precedence proof) ----
$ grep -nE '^URL='        /app/.env        # the .env FILE value (loses to process env)
$ grep -nE '^export URL=' /tmp/sl_env.sh   # the shell/process-env value (wins)

# ---- required-variable hard-fails through the REAL entry point: python3 server.py ----
# each no_<VAR>.env is /app/.env with exactly ONE required line removed; run in a clean env
# (env -i) so ONLY the CONFIG file supplies values and the removed var is genuinely absent.
$ env -i PATH=/usr/bin:/bin CONFIG=/tmp/blitzy_cap/q2/no_url.env            /app/venv/bin/python server.py
$ env -i PATH=/usr/bin:/bin CONFIG=/tmp/blitzy_cap/q2/no_email_domain.env   /app/venv/bin/python server.py
$ env -i PATH=/usr/bin:/bin CONFIG=/tmp/blitzy_cap/q2/no_support_email.env  /app/venv/bin/python server.py
$ env -i PATH=/usr/bin:/bin CONFIG=/tmp/blitzy_cap/q2/no_db_uri.env         /app/venv/bin/python server.py
$ env -i PATH=/usr/bin:/bin CONFIG=/tmp/blitzy_cap/q2/no_flask_secret.env   /app/venv/bin/python server.py
# present-but-empty FLASK_SECRET → RuntimeError (not KeyError):
$ env -i PATH=/usr/bin:/bin CONFIG=/tmp/blitzy_cap/q2/empty_flask_secret.env /app/venv/bin/python server.py
# sixth required value, loaded via sl_getenv() with no default_factory → TypeError:
$ env -i PATH=/usr/bin:/bin CONFIG=/tmp/blitzy_cap/q2/no_email_servers.env  /app/venv/bin/python server.py
```

### Complete, unedited output

**(a) DEFAULT (`else`) branch + `override=False` precedence** — canonical env sourced, `CONFIG`
unset. No `load config file` line is printed (the `else` branch at L71 is taken), and the observed
URL is `http://localhost` — the process-env value, **not** the `/app/.env` value `:7777`:

```
>>> URL: http://localhost
Upload files to local dir
```

**(b) `CONFIG` branch** — a controlled file is loaded; the `load config file <abs-path>` banner
(L68) fires and the file's distinctive `URL=http://localhost:9999` is what the module ends up with,
proving the CONFIG file is the source:

```
load config file /tmp/blitzy_cap/q2/full.env
>>> URL: http://localhost:9999
Upload files to local dir
[import ok] app.config.URL = http://localhost:9999
```

**(c) The two conflicting `URL` definitions** (precedence proof — the file loses, the process env
wins):

```
--- /app/.env line 53 (the file value, LOSES) ---
53:URL=http://localhost:7777
--- /tmp/sl_env.sh line 4 (the shell/process-env value, WINS) ---
4:export URL="http://localhost"
```

**(d) Required-variable hard-fails through `python3 server.py`.** Each aborts at the exact line that
reads the missing variable; note the `>>> URL:` print (L80) appears for every case *after* the first
(because `URL` is resolved first at L79):

`no_url.env` → `KeyError: 'URL'` at `config.py:L79`:

```
load config file /tmp/blitzy_cap/q2/no_url.env
Traceback (most recent call last):
  File "/app/server.py", line 30, in <module>
    from app import config, constants
  File "/app/app/config.py", line 79, in <module>
    URL = os.environ["URL"]
  File "/usr/local/lib/python3.10/os.py", line 680, in __getitem__
    raise KeyError(key) from None
KeyError: 'URL'
```

`no_email_domain.env` → `KeyError: 'EMAIL_DOMAIN'` at `config.py:L92`:

```
load config file /tmp/blitzy_cap/q2/no_email_domain.env
>>> URL: http://localhost:9999
Traceback (most recent call last):
  File "/app/server.py", line 30, in <module>
    from app import config, constants
  File "/app/app/config.py", line 92, in <module>
    EMAIL_DOMAIN = os.environ["EMAIL_DOMAIN"].lower()
  File "/usr/local/lib/python3.10/os.py", line 680, in __getitem__
    raise KeyError(key) from None
KeyError: 'EMAIL_DOMAIN'
```

`no_support_email.env` → `KeyError: 'SUPPORT_EMAIL'` at `config.py:L93`:

```
load config file /tmp/blitzy_cap/q2/no_support_email.env
>>> URL: http://localhost:9999
Traceback (most recent call last):
  File "/app/server.py", line 30, in <module>
    from app import config, constants
  File "/app/app/config.py", line 93, in <module>
    SUPPORT_EMAIL = os.environ["SUPPORT_EMAIL"]
  File "/usr/local/lib/python3.10/os.py", line 680, in __getitem__
    raise KeyError(key) from None
KeyError: 'SUPPORT_EMAIL'
```

`no_db_uri.env` → `KeyError: 'DB_URI'` at `config.py:L192`:

```
load config file /tmp/blitzy_cap/q2/no_db_uri.env
>>> URL: http://localhost:9999
Traceback (most recent call last):
  File "/app/server.py", line 30, in <module>
    from app import config, constants
  File "/app/app/config.py", line 192, in <module>
    DB_URI = os.environ["DB_URI"]
  File "/usr/local/lib/python3.10/os.py", line 680, in __getitem__
    raise KeyError(key) from None
KeyError: 'DB_URI'
```

`no_flask_secret.env` → `KeyError: 'FLASK_SECRET'` at `config.py:L196`:

```
load config file /tmp/blitzy_cap/q2/no_flask_secret.env
>>> URL: http://localhost:9999
Traceback (most recent call last):
  File "/app/server.py", line 30, in <module>
    from app import config, constants
  File "/app/app/config.py", line 196, in <module>
    FLASK_SECRET = os.environ["FLASK_SECRET"]
  File "/usr/local/lib/python3.10/os.py", line 680, in __getitem__
    raise KeyError(key) from None
KeyError: 'FLASK_SECRET'
```

`empty_flask_secret.env` (`FLASK_SECRET=` present but blank) → `RuntimeError` at `config.py:L198`:

```
load config file /tmp/blitzy_cap/q2/empty_flask_secret.env
>>> URL: http://localhost:9999
Traceback (most recent call last):
  File "/app/server.py", line 30, in <module>
    from app import config, constants
  File "/app/app/config.py", line 198, in <module>
    raise RuntimeError("FLASK_SECRET is empty. Please define it.")
RuntimeError: FLASK_SECRET is empty. Please define it.
```

**(e) Additional required value with a different failure mode** — `no_email_servers.env`
(`EMAIL_SERVERS_WITH_PRIORITY` removed) → `TypeError` from `sl_getenv`'s `default_factory()` call at
`config.py:L33`:

```
load config file /tmp/blitzy_cap/q2/no_email_servers.env
>>> URL: http://localhost:9999
Traceback (most recent call last):
  File "/app/server.py", line 30, in <module>
    from app import config, constants
  File "/app/app/config.py", line 172, in <module>
    EMAIL_SERVERS_WITH_PRIORITY = sl_getenv("EMAIL_SERVERS_WITH_PRIORITY")
  File "/app/app/config.py", line 33, in sl_getenv
    return default_factory()
TypeError: 'NoneType' object is not callable
```

### File:line grounding

- `server.py:L30` `from app import config, constants` — the first SL import; triggers all of
  `app/config.py` at module load.
- `app/config.py:L65` `config_file = os.environ.get("CONFIG")`; `L66` `if config_file:`; `L68`
  `print("load config file", config_file)`; `L69` `load_dotenv(get_abs_path(config_file))`; `L70-71`
  `else:` / `load_dotenv()` (default `override=False`).
- `app/config.py:L17-18` `get_abs_path` returns absolute paths unchanged (`if file_path.startswith("/"): return file_path`).
- Required bracket variables: `L79` `URL = os.environ["URL"]`; `L80` `print(">>> URL:", URL)`; `L92`
  `EMAIL_DOMAIN = os.environ["EMAIL_DOMAIN"].lower()`; `L93` `SUPPORT_EMAIL = os.environ["SUPPORT_EMAIL"]`;
  `L192` `DB_URI = os.environ["DB_URI"]`; `L196` `FLASK_SECRET = os.environ["FLASK_SECRET"]`.
- Empty-secret guard: `L197` `if not FLASK_SECRET:`; `L198` `raise RuntimeError("FLASK_SECRET is empty. Please define it.")`.
- `sl_getenv` required value: `L23` `def sl_getenv(env_var: str, default_factory: Callable = None):`;
  `L31-33` `value = os.getenv(env_var)` / `if value is None:` / `return default_factory()`; used at
  `L172` `EMAIL_SERVERS_WITH_PRIORITY = sl_getenv("EMAIL_SERVERS_WITH_PRIORITY")`.
- `app/config.py:L328` `print("Upload files to local dir")` — printed because `LOCAL_FILE_UPLOAD=1`
  is set in both the canonical env and the copied `full.env`.
- `app/config.py:L199` `SESSION_COOKIE_NAME = "slapp"` (the cookie name confirmed live in Q3/Q5).

### Observed vs Inferred

- **Observed:** the `else`/`CONFIG` branch selection and their prints; the `override=False`
  precedence (`URL=http://localhost` wins over the file's `:7777`); the CONFIG file being
  authoritative (`URL=http://localhost:9999`); and every required-variable failure with its exact
  exception type and `config.py` line (`KeyError` for the five bracket vars at L79/L92/L93/L192/L196,
  `RuntimeError` for empty `FLASK_SECRET` at L198, `TypeError` for `EMAIL_SERVERS_WITH_PRIORITY` at
  L33/L172).
- **Canonical-entry-point notes:** the five `KeyError` cases, the empty-secret `RuntimeError`, and the
  `TypeError` case were exercised through the **real** entry point `python3 server.py`. The default-branch
  and `CONFIG`-branch success probes used `import app.config` — the **same module object** `server.py`
  imports at `L30`; the identical `>>> URL: http://localhost` value is independently corroborated by
  the canonical `python3 server.py` startup captured in Q1/Q3.
- **Inferred:** nothing in this section is inferred — every branch and failure was run and its
  complete output captured above.
- **Security note (DEVELOPMENT only):** the values shown here (`FLASK_SECRET=secret`, local DB URI,
  the seeded `.env`) are the container's throwaway development configuration. They are not production
  secrets and must never be reused outside this local investigation environment.

---

## Q3 — Which log messages actually signal that the server is ready?

### Direct answer (Observed)

**There is no explicit "server is ready / listening" log line in dev mode.** SimpleLogin disables
the `werkzeug` logger at import (`log = logging.getLogger("werkzeug")` / `log.disabled = True`,
`app/log.py:L70-71`), which **suppresses Werkzeug's entire standard startup banner** on this pinned
stack (Werkzeug **1.0.1** / Flask **1.1.2**): the familiar `* Running on http://127.0.0.1:7777/`
and `Press CTRL+C to quit`, and (because `debug=True` + reloader) `* Restarting with stat`,
`* Debugger is active!`, and `* Debugger PIN: NNN-NNN-NNN` — **none of these appear** (verified by
capturing the actual stream in Q1). The only startup markers actually printed are:

1. SimpleLogin's own `print()` banners — `>>> URL: http://localhost` (`app/config.py:L80`) and
   `>>> init logging <<<` (`app/log.py:L67`) — each printed **twice** (reloader parent + child); and
2. Flask's CLI banner — `* Serving Flask app "server" (lazy loading)`, `* Environment: production`,
   the two-line development-server `WARNING`, and `* Debug mode: on` — which Flask emits through
   `click.echo` (its `show_server_banner`), **not** through the `werkzeug` logger, so it survives.

Because there is no readiness line, practical readiness must be confirmed out-of-band; the
`/health` endpoint returning **`HTTP/1.0 200 OK`** with body `success` is the reliable signal. The
**last observable startup marker before the socket accepts is the second (serving-child) `SL`
"load words file" line** (`app/utils.py:L17`, emitted during the child's app import) — it prints
immediately *after* the child's second `>>> init logging <<<`, and nothing else is logged between it
and the first accepted request (the launch→first-`200` correlation below shows the child's
"load words file" line and the first `/health` `200` landing in the **same second**). Separately,
`local_main()` runs `config.COLOR_LOG = True` (`server.py:L573`) but this does **not** colorize the
logs — see the COLOR_LOG sub-finding below.

### Command(s) run

```
$ /app/venv/bin/python -c "import werkzeug, flask; print('werkzeug', werkzeug.__version__, '| flask', flask.__version__)"
# full startup stream captured as in Q1 (reproduced below), then out-of-band readiness probe:
$ curl -s -i http://localhost:7777/health
# COLOR_LOG behaviour probe (NON-CANONICAL: imports modules directly to mimic local_main L573):
$ /app/venv/bin/python /tmp/blitzy_cap/colorlog_probe.py
```

### Complete output (secret-masked)

_(One `Set-Cookie` HMAC signature is masked below per the Evidence-discipline note; the block is otherwise structurally verbatim.)_

Werkzeug/Flask version pairing:

```
werkzeug 1.0.1 | flask 1.1.2
```

The complete **pure startup stream** (RUN‑A, before any request) — this is the evidence for which
readiness lines appear. Note the **absence** of every Werkzeug-logger line and the presence of only
the two SL `print` markers (×2) and the Flask CLI banner:

```
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-13 18:23:21,160 - SL - DEBUG - 5058 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-13 18:23:22,753 - SL - DEBUG - 5070 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

Out-of-band readiness confirmation while the server was up (RUN‑A):

```
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 7
Set-Cookie: slapp=c44ff3a2-7a92-4b69-8a61-c1b58ab5332d.<REDACTED:slapp-hmac-signature>; Expires=Mon, 20-Jul-2026 18:23:25 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Mon, 13 Jul 2026 18:23:25 GMT

success
```

**First-success timing correlation (Observed).** To tie the startup stream to the first request the
socket actually accepts, the canonical dev server was launched (`python server.py`, recording `T0`)
and `/health` polled every 50 ms until the first `200` (recording `T1`); then the last import-time
`SL` lines and the first `/health` `Date` header were printed. The serving-child's final import-time
marker — the `SL` "load words file" line (`app/utils.py:L17`) — lands at `05:03:29,186` (child PID
`38647`), and the first successful `/health` carries `Date: Tue, 14 Jul 2026 05:03:29 GMT`: the socket
begins accepting in the **same second** the child finishes importing. Measured launch→first-`200` was
**3.097 s**.

```bash
T0=$(date +%s.%N); setsid /app/venv/bin/python server.py > q3.log 2>&1 &
for i in $(seq 1 600); do
  [ "$(curl -s -o /dev/null -w '%{http_code}' http://localhost:7777/health)" = 200 ] \
    && { first=$(date +%s.%N); break; }; sleep 0.05
done
awk -v a=$T0 -v b=$first 'BEGIN{printf "elapsed_launch_to_first200 = %.3f s\n", b-a}'
curl -s -i http://localhost:7777/health | grep -iE '^HTTP/|^Date:'
grep -E ">>> init logging <<<|load words file|Debug mode: on" q3.log | tail -5
```

The concatenated output of that sequence — the elapsed timing, then the first `/health` status line
and `Date` header, then the tail of the import-time markers for parent PID `38554` and serving-child
PID `38647`:

```text
elapsed_launch_to_first200 = 3.097 s
HTTP/1.0 200 OK
Date: Tue, 14 Jul 2026 05:03:29 GMT
>>> init logging <<<
2026-07-14 05:03:27,627 - SL - DEBUG - 38554 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
 * Debug mode: on
>>> init logging <<<
2026-07-14 05:03:29,186 - SL - DEBUG - 38647 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

**COLOR_LOG sub-finding (why `local_main()`'s `config.COLOR_LOG = True` does not colorize logs).**
The `SL` logger is built at import time in `app/log.py`, which imports `COLOR_LOG` *by value*
(`from app.config import (COLOR_LOG,)` L7-9) and only calls `coloredlogs.install(...)` inside
`_get_logger()` when that import-time value is truthy (L61-62). `local_main()` sets
`config.COLOR_LOG = True` **after** `app/log.py` has already been imported and `LOG` already built,
so it neither changes `app.log.COLOR_LOG` nor re-installs coloredlogs. This probe demonstrates it
(and matches the plain, un-colored SL lines in the startup stream above — a `cat -A` of the stream
shows no ANSI escape codes):

```
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
import-time : config.COLOR_LOG=False  log.COLOR_LOG=False
after set   : config.COLOR_LOG=True  log.COLOR_LOG=False
SL handlers : ['logging.StreamHandler']
SL formatter: 'logging.Formatter'
```

### File:line grounding

- **Appears:** `print(">>> init logging <<<")` `app/log.py:L67` (×2); `print(">>> URL:", URL)`
  `app/config.py:L80` (×2). The Flask CLI banner lines are produced by Flask's `show_server_banner`
  (via `click.echo`, `flask/cli.py`), independent of the `werkzeug` logger.
- **Suppressor:** `log = logging.getLogger("werkzeug")` (`app/log.py:L70`); `log.disabled = True`
  (`app/log.py:L71`) — this is why the entire Werkzeug banner is absent.
- **SL logger construction (import time):** `from app.config import (COLOR_LOG,)` (`app/log.py:L7-9`);
  `_log_format` (`app/log.py:L12-15`); console handler on `sys.stdout` with UTC `converter = time.gmtime`
  (`app/log.py:L43`); `logger.setLevel(logging.DEBUG)` (`app/log.py:L51`); `if COLOR_LOG: coloredlogs.install(...)`
  (`app/log.py:L61-62`); `LOG = _get_logger("SL")` (`app/log.py:L79`).
- **Suppressed lines' emitters (code-grounded / Inferred, not run):** `* Running on ...` and
  `Press CTRL+C to quit` are logged by Werkzeug 1.0.1 `serving.py` `run_simple` via the `werkzeug`
  logger; `* Restarting with stat` by `werkzeug/_reloader.py`; `* Debugger is active!` / `* Debugger PIN:`
  by `werkzeug/debug/__init__.py`. Their **absence** is Observed; that the `werkzeug` logger is their
  channel is code-grounded.
- `/health` handler returns `("success", 200)` `server.py:L214-215` (`healthcheck()` inside `create_app`).

### Observed vs Inferred

- **Observed:** exactly which lines appear (the two `print` markers ×2 and the Flask CLI banner) and
  which are absent (the entire Werkzeug banner including the Debugger PIN) on Werkzeug 1.0.1 /
  Flask 1.1.2; the `/health` `200`/`success` as the practical readiness signal
  (`Server: Werkzeug/1.0.1 Python/3.10.18`); and that `config.COLOR_LOG=True` leaves
  `log.COLOR_LOG=False` with a plain `StreamHandler`/`Formatter` (no colorization). The Debugger PIN
  is run-to-run-variable in general, but here its **absence** is the stable cross-run observation.
- **Inferred / code-grounded (not run):** the specific Werkzeug source functions that would emit the
  suppressed lines. The absence itself is Observed; the causal attribution to the disabled
  `werkzeug` logger is grounded in `app/log.py:L70-71` and the Werkzeug source.

---

## Q4 — Which ports and endpoints does it expose?

### Direct answer (Observed)

In development the server listens on **one TCP port, `7777`, bound to the loopback interface
`127.0.0.1`** — set by `app.run(debug=True, port=7777)` (`server.py:L588`) and confirmed live in
`/proc/net/tcp` as `0100007F:1E61` (127.0.0.1:7777) in state `0A` (LISTEN). (`EXPOSE 7777` in
`Dockerfile:L44` documents the same port for the production `gunicorn` bind `0.0.0.0:7777`, which is
**Inferred** — not run here.) The app built by the canonical factory `create_app()` exposes a route
map of **exactly 292 rules** (stable across two independent builds). Those 292 rules break down as:

| Mount / prefix | Rules | Source |
|----------------|------:|--------|
| `/admin` (Flask-Admin UI) | 122 | `Admin(...)` mount in `create_app()` |
| `/api` (`api_bp`) | 52 | `app/api/base.py:L11` `url_prefix="/api"` |
| `/dashboard` (`dashboard_bp`) | 51 | `app/dashboard/base.py:L6` |
| `/auth` (`auth_bp`) | 23 | `app/auth/base.py:L4` |
| `/developer` (`developer_bp`) | 7 | `app/developer/base.py:L6` |
| `/onboarding` (`onboarding_bp`) | 6 | `app/onboarding/base.py:L6` |
| `/phone` (`phone_bp`) | 5 | `app/phone/base.py:L6` |
| `/oauth` (`oauth_bp`) | 5 | `server.py:L240` `url_prefix="/oauth"` |
| `/oauth2` (`oauth_bp` again) | 5 | `server.py:L241` `url_prefix="/oauth2"` |
| `/internal` (`internal_bp`) | 2 | `app/internal/base.py:L6` |
| `/static` | 1 | Flask default static route |
| direct + `monitor_bp` routes | 13 | see below |

`register_blueprints(app)` (`server.py:L233-246`) performs **11 registrations across 10 distinct
blueprints** — `oauth_bp` is registered **twice** (at `/oauth` and `/oauth2`). The 13 remaining rules
are direct (non-blueprint) app routes plus the `monitor_bp` routes: `/` (`index`),
`/health` (`healthcheck`), `/jwks`, `/favicon.ico`, `/dnt` (`do_not_track`),
`/.well-known/openid-configuration` (`openid_config`), `/paddle`, `/paddle_coupon`,
`/coinbase` (`coinbase_webhook`), `/discover` (`discover.index`), and the three `monitor_bp` routes
`/git` (`monitor.git_sha1`), `/live` (`monitor.live`), `/exception` (`monitor.test_exception`).

**Discrepancy (Observed, code-authoritative):** `monitor_bp` is mounted at **`url_prefix="/"`**
(`app/monitor/base.py:L3`), so its routes are `/git`, `/live`, `/exception` — **not** under a
`/monitor` prefix as an earlier tech-spec draft (§5.2.1) suggested. The running route map is
authoritative.

### Command(s) run

```
$ cd /app && set -a && . /tmp/sl_env.sh && set +a && unset EVENT_WEBHOOK_DISABLE && export GNUPGHOME=/tmp/sl_clean_gnupg

# (1) enumerate the live route table from the SAME factory local_main() uses (create_app):
$ cat dump_routes.py
from server import create_app
app = create_app()
rules = list(app.url_map.iter_rules())
print("TOTAL RULES:", len(rules))
def methods_of(r):
    return ",".join(sorted(m for m in r.methods if m not in ("HEAD","OPTIONS")))
for r in sorted(rules, key=lambda r: (str(r), r.endpoint)):
    print(f"{methods_of(r):22s} {str(r):55s} -> {r.endpoint}")
$ /app/venv/bin/python dump_routes.py           # run twice; the route portion is identical

# (2) confirm the listening socket while a real server is up, then probe readiness:
$ /app/venv/bin/python server.py >/tmp/q4srv.log 2>&1 &   PARENT=$!
$ sleep 5
$ head -1 /proc/net/tcp ; grep -iE ':1E61 [0-9A-F:]+ 0A ' /proc/net/tcp   # :7777 LISTEN
$ curl -s -i http://localhost:7777/health
$ kill -TERM $CHILD $PARENT ; kill -KILL $CHILD $PARENT
$ grep -iE ':1E61 [0-9A-F:]+ 0A ' /proc/net/tcp || echo PORT FREE   # after termination
```

### Complete output (secret-masked)

_(One `Set-Cookie` HMAC signature is masked below per the Evidence-discipline note; the block is otherwise structurally verbatim.)_

**Listening socket** (`/proc/net/tcp`, header + the single `:1E61`/state-`0A` line for this run;
parent/child PIDs recorded):

```
### /proc/net/tcp header + the :7777 (hex 1E61) LISTEN (state 0A) line:
  sl  local_address rem_address   st tx_queue rx_queue tr tm->when retrnsmt   uid  timeout inode                                                     
   2: 0100007F:1E61 00000000:0000 0A 00000000:00000000 00:00000000 00000000     0        0 879068313 1 0000000000000000 100 0 0 10 0                 

### decode: local_address 0100007F:1E61 = 127.0.0.1:7777 ; st 0A = LISTEN
### server PIDs: parent=5507 child=5520
```

**Readiness probe** — `GET /health` returns `200`/`success` on the loopback bind:

```
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 7
Set-Cookie: slapp=71782d44-deb0-4725-afa3-52c158f17fe7.<REDACTED:slapp-hmac-signature>; Expires=Mon, 20-Jul-2026 18:42:37 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Mon, 13 Jul 2026 18:42:37 GMT

success
```

**Full live route map** — the complete, unedited enumeration of all 292 rules (methods with
`HEAD`/`OPTIONS` elided for readability of the *methods column only*; the rule paths and endpoints are
verbatim). Identical across both builds:

```
TOTAL RULES: 292
GET,POST               /                                                       -> index
GET                    /.well-known/openid-configuration                       -> openid_config
GET                    /admin/                                                 -> admin.index
GET                    /admin/adminauditlog/                                   -> adminauditlog.index_view
POST                   /admin/adminauditlog/action/                            -> adminauditlog.action_view
GET                    /admin/adminauditlog/ajax/lookup/                       -> adminauditlog.ajax_lookup
POST                   /admin/adminauditlog/ajax/update/                       -> adminauditlog.ajax_update
POST                   /admin/adminauditlog/delete/                            -> adminauditlog.delete_view
GET                    /admin/adminauditlog/details/                           -> adminauditlog.details_view
GET,POST               /admin/adminauditlog/edit/                              -> adminauditlog.edit_view
GET                    /admin/adminauditlog/export/<export_type>/              -> adminauditlog.export
GET,POST               /admin/adminauditlog/new/                               -> adminauditlog.create_view
GET                    /admin/alias/                                           -> alias.index_view
POST                   /admin/alias/action/                                    -> alias.action_view
GET                    /admin/alias/ajax/lookup/                               -> alias.ajax_lookup
POST                   /admin/alias/ajax/update/                               -> alias.ajax_update
POST                   /admin/alias/delete/                                    -> alias.delete_view
GET                    /admin/alias/details/                                   -> alias.details_view
GET,POST               /admin/alias/edit/                                      -> alias.edit_view
GET                    /admin/alias/export/<export_type>/                      -> alias.export
GET,POST               /admin/alias/new/                                       -> alias.create_view
GET                    /admin/coupon/                                          -> coupon.index_view
POST                   /admin/coupon/action/                                   -> coupon.action_view
GET                    /admin/coupon/ajax/lookup/                              -> coupon.ajax_lookup
POST                   /admin/coupon/ajax/update/                              -> coupon.ajax_update
POST                   /admin/coupon/delete/                                   -> coupon.delete_view
GET                    /admin/coupon/details/                                  -> coupon.details_view
GET,POST               /admin/coupon/edit/                                     -> coupon.edit_view
GET                    /admin/coupon/export/<export_type>/                     -> coupon.export
GET,POST               /admin/coupon/new/                                      -> coupon.create_view
GET                    /admin/customdomain/                                    -> customdomain.index_view
POST                   /admin/customdomain/action/                             -> customdomain.action_view
GET                    /admin/customdomain/ajax/lookup/                        -> customdomain.ajax_lookup
POST                   /admin/customdomain/ajax/update/                        -> customdomain.ajax_update
POST                   /admin/customdomain/delete/                             -> customdomain.delete_view
GET                    /admin/customdomain/details/                            -> customdomain.details_view
GET,POST               /admin/customdomain/edit/                               -> customdomain.edit_view
GET                    /admin/customdomain/export/<export_type>/               -> customdomain.export
GET,POST               /admin/customdomain/new/                                -> customdomain.create_view
GET                    /admin/dailymetric/                                     -> dailymetric.index_view
POST                   /admin/dailymetric/action/                              -> dailymetric.action_view
GET                    /admin/dailymetric/ajax/lookup/                         -> dailymetric.ajax_lookup
POST                   /admin/dailymetric/ajax/update/                         -> dailymetric.ajax_update
POST                   /admin/dailymetric/delete/                              -> dailymetric.delete_view
GET                    /admin/dailymetric/details/                             -> dailymetric.details_view
GET,POST               /admin/dailymetric/edit/                                -> dailymetric.edit_view
GET                    /admin/dailymetric/export/<export_type>/                -> dailymetric.export
GET,POST               /admin/dailymetric/new/                                 -> dailymetric.create_view
GET,POST               /admin/email_search/                                    -> email_search.index
GET                    /admin/invalidmailboxdomain/                            -> invalidmailboxdomain.index_view
POST                   /admin/invalidmailboxdomain/action/                     -> invalidmailboxdomain.action_view
GET                    /admin/invalidmailboxdomain/ajax/lookup/                -> invalidmailboxdomain.ajax_lookup
POST                   /admin/invalidmailboxdomain/ajax/update/                -> invalidmailboxdomain.ajax_update
POST                   /admin/invalidmailboxdomain/delete/                     -> invalidmailboxdomain.delete_view
GET                    /admin/invalidmailboxdomain/details/                    -> invalidmailboxdomain.details_view
GET,POST               /admin/invalidmailboxdomain/edit/                       -> invalidmailboxdomain.edit_view
GET                    /admin/invalidmailboxdomain/export/<export_type>/       -> invalidmailboxdomain.export
GET,POST               /admin/invalidmailboxdomain/new/                        -> invalidmailboxdomain.create_view
GET                    /admin/mailbox/                                         -> mailbox.index_view
POST                   /admin/mailbox/action/                                  -> mailbox.action_view
GET                    /admin/mailbox/ajax/lookup/                             -> mailbox.ajax_lookup
POST                   /admin/mailbox/ajax/update/                             -> mailbox.ajax_update
POST                   /admin/mailbox/delete/                                  -> mailbox.delete_view
GET                    /admin/mailbox/details/                                 -> mailbox.details_view
GET,POST               /admin/mailbox/edit/                                    -> mailbox.edit_view
GET                    /admin/mailbox/export/<export_type>/                    -> mailbox.export
GET,POST               /admin/mailbox/new/                                     -> mailbox.create_view
GET                    /admin/manualsubscription/                              -> manualsubscription.index_view
POST                   /admin/manualsubscription/action/                       -> manualsubscription.action_view
GET                    /admin/manualsubscription/ajax/lookup/                  -> manualsubscription.ajax_lookup
POST                   /admin/manualsubscription/ajax/update/                  -> manualsubscription.ajax_update
POST                   /admin/manualsubscription/delete/                       -> manualsubscription.delete_view
GET                    /admin/manualsubscription/details/                      -> manualsubscription.details_view
GET,POST               /admin/manualsubscription/edit/                         -> manualsubscription.edit_view
GET                    /admin/manualsubscription/export/<export_type>/         -> manualsubscription.export
GET,POST               /admin/manualsubscription/new/                          -> manualsubscription.create_view
GET                    /admin/metric2/                                         -> metric2.index_view
POST                   /admin/metric2/action/                                  -> metric2.action_view
GET                    /admin/metric2/ajax/lookup/                             -> metric2.ajax_lookup
POST                   /admin/metric2/ajax/update/                             -> metric2.ajax_update
POST                   /admin/metric2/delete/                                  -> metric2.delete_view
GET                    /admin/metric2/details/                                 -> metric2.details_view
GET,POST               /admin/metric2/edit/                                    -> metric2.edit_view
GET                    /admin/metric2/export/<export_type>/                    -> metric2.export
GET,POST               /admin/metric2/new/                                     -> metric2.create_view
GET                    /admin/newsletter/                                      -> newsletter.index_view
POST                   /admin/newsletter/action/                               -> newsletter.action_view
GET                    /admin/newsletter/ajax/lookup/                          -> newsletter.ajax_lookup
POST                   /admin/newsletter/ajax/update/                          -> newsletter.ajax_update
POST                   /admin/newsletter/delete/                               -> newsletter.delete_view
GET                    /admin/newsletter/details/                              -> newsletter.details_view
GET,POST               /admin/newsletter/edit/                                 -> newsletter.edit_view
GET                    /admin/newsletter/export/<export_type>/                 -> newsletter.export
GET,POST               /admin/newsletter/new/                                  -> newsletter.create_view
GET                    /admin/newsletteruser/                                  -> newsletteruser.index_view
POST                   /admin/newsletteruser/action/                           -> newsletteruser.action_view
GET                    /admin/newsletteruser/ajax/lookup/                      -> newsletteruser.ajax_lookup
POST                   /admin/newsletteruser/ajax/update/                      -> newsletteruser.ajax_update
POST                   /admin/newsletteruser/delete/                           -> newsletteruser.delete_view
GET                    /admin/newsletteruser/details/                          -> newsletteruser.details_view
GET,POST               /admin/newsletteruser/edit/                             -> newsletteruser.edit_view
GET                    /admin/newsletteruser/export/<export_type>/             -> newsletteruser.export
GET,POST               /admin/newsletteruser/new/                              -> newsletteruser.create_view
GET                    /admin/providercomplaint/                               -> providercomplaint.index_view
POST                   /admin/providercomplaint/action/                        -> providercomplaint.action_view
GET                    /admin/providercomplaint/ajax/lookup/                   -> providercomplaint.ajax_lookup
POST                   /admin/providercomplaint/ajax/update/                   -> providercomplaint.ajax_update
POST                   /admin/providercomplaint/delete/                        -> providercomplaint.delete_view
GET                    /admin/providercomplaint/details/                       -> providercomplaint.details_view
GET                    /admin/providercomplaint/download_eml                   -> providercomplaint.download_eml
GET,POST               /admin/providercomplaint/edit/                          -> providercomplaint.edit_view
GET                    /admin/providercomplaint/export/<export_type>/          -> providercomplaint.export
GET                    /admin/providercomplaint/mark_ok                        -> providercomplaint.mark_ok
GET,POST               /admin/providercomplaint/new/                           -> providercomplaint.create_view
GET                    /admin/static/<path:filename>                           -> admin.static
GET                    /admin/user/                                            -> user.index_view
POST                   /admin/user/action/                                     -> user.action_view
GET                    /admin/user/ajax/lookup/                                -> user.ajax_lookup
POST                   /admin/user/ajax/update/                                -> user.ajax_update
POST                   /admin/user/delete/                                     -> user.delete_view
GET                    /admin/user/details/                                    -> user.details_view
GET,POST               /admin/user/edit/                                       -> user.edit_view
GET                    /admin/user/export/<export_type>/                       -> user.export
GET,POST               /admin/user/new/                                        -> user.create_view
POST                   /api/alias/random/new                                   -> api.new_random_alias
GET,POST               /api/aliases                                            -> api.get_aliases
DELETE                 /api/aliases/<int:alias_id>                             -> api.delete_alias
GET                    /api/aliases/<int:alias_id>                             -> api.get_alias
PATCH,PUT              /api/aliases/<int:alias_id>                             -> api.update_alias
GET                    /api/aliases/<int:alias_id>/activities                  -> api.get_alias_activities
POST                   /api/aliases/<int:alias_id>/contacts                    -> api.create_contact_route
GET                    /api/aliases/<int:alias_id>/contacts                    -> api.get_alias_contacts_route
POST                   /api/aliases/<int:alias_id>/toggle                      -> api.toggle_alias
POST                   /api/api_key                                            -> api.create_api_key
POST                   /api/apple/process_payment                              -> api.apple_process_payment
GET,POST               /api/apple/update_notification                          -> api.apple_update_notification
POST                   /api/auth/activate                                      -> api.auth_activate
POST                   /api/auth/facebook                                      -> api.auth_facebook
POST                   /api/auth/forgot_password                               -> api.forgot_password
POST                   /api/auth/google                                        -> api.auth_google
POST                   /api/auth/login                                         -> api.auth_login
POST                   /api/auth/mfa                                           -> api.auth_mfa
POST                   /api/auth/reactivate                                    -> api.auth_reactivate
POST                   /api/auth/register                                      -> api.auth_register
DELETE                 /api/contacts/<int:contact_id>                          -> api.delete_contact
POST                   /api/contacts/<int:contact_id>/toggle                   -> api.toggle_contact
GET                    /api/custom_domains                                     -> api.get_custom_domains
PATCH                  /api/custom_domains/<int:custom_domain_id>              -> api.update_custom_domain
GET                    /api/custom_domains/<int:custom_domain_id>/trash        -> api.get_custom_domain_trash
GET                    /api/export/aliases                                     -> api.export_aliases
GET                    /api/export/data                                        -> api.export_data
GET                    /api/logout                                             -> api.logout
POST                   /api/mailboxes                                          -> api.create_mailbox
GET                    /api/mailboxes                                          -> api.get_mailboxes
DELETE                 /api/mailboxes/<int:mailbox_id>                         -> api.delete_mailbox
PUT                    /api/mailboxes/<int:mailbox_id>                         -> api.update_mailbox
GET                    /api/notifications                                      -> api.get_notifications
POST                   /api/notifications/<int:notification_id>/read           -> api.mark_as_read
GET,POST               /api/phone/reservations/<int:reservation_id>            -> api.phone_messages
GET                    /api/setting                                            -> api.get_setting
PATCH                  /api/setting                                            -> api.update_setting
GET                    /api/setting/domains                                    -> api.get_available_domains_for_random_alias
DELETE                 /api/setting/unlink_proton_account                      -> api.unlink_proton_account
GET                    /api/stats                                              -> api.user_stats
PATCH                  /api/sudo                                               -> api.enter_sudo
DELETE                 /api/user                                               -> api.delete_user
GET                    /api/user/cookie_token                                  -> api.get_api_session_token
PATCH                  /api/user_info                                          -> api.update_user_info
GET                    /api/user_info                                          -> api.user_info
POST                   /api/v2/alias/custom/new                                -> api.new_custom_alias_v2
GET,POST               /api/v2/aliases                                         -> api.get_aliases_v2
GET                    /api/v2/mailboxes                                       -> api.get_mailboxes_v2
GET                    /api/v2/setting/domains                                 -> api.get_available_domains_for_random_alias_v2
POST                   /api/v3/alias/custom/new                                -> api.new_custom_alias_v3
GET                    /api/v4/alias/options                                   -> api.options_v4
GET                    /api/v5/alias/options                                   -> api.options_v5
GET,POST               /auth/activate                                          -> auth.activate
GET                    /auth/api_to_cookie                                     -> auth.api_to_cookie
GET,POST               /auth/change_email                                      -> auth.change_email
GET                    /auth/facebook/callback                                 -> auth.facebook_callback
GET                    /auth/facebook/login                                    -> auth.facebook_login
GET,POST               /auth/fido                                              -> auth.fido
GET,POST               /auth/forgot_password                                   -> auth.forgot_password
GET                    /auth/github/callback                                   -> auth.github_callback
GET                    /auth/github/login                                      -> auth.github_login
GET                    /auth/google/callback                                   -> auth.google_callback
GET                    /auth/google/login                                      -> auth.google_login
GET,POST               /auth/login                                             -> auth.login
GET                    /auth/logout                                            -> auth.logout
GET,POST               /auth/mfa                                               -> auth.mfa
GET                    /auth/oidc/callback                                     -> auth.oidc_callback
GET                    /auth/oidc/login                                        -> auth.oidc_login
GET                    /auth/proton/callback                                   -> auth.proton_callback
GET                    /auth/proton/login                                      -> auth.proton_login
GET,POST               /auth/recovery                                          -> auth.recovery_route
GET,POST               /auth/register                                          -> auth.register
GET,POST               /auth/resend_activation                                 -> auth.resend_activation
GET,POST               /auth/reset_password                                    -> auth.reset_password
GET,POST               /auth/social                                            -> auth.social
POST                   /coinbase                                               -> coinbase_webhook
GET,POST               /dashboard/                                             -> dashboard.index
GET,POST               /dashboard/account_setting                              -> dashboard.account_setting
GET,POST               /dashboard/alias_contact_manager/<int:alias_id>/        -> dashboard.alias_contact_manager
GET                    /dashboard/alias_export                                 -> dashboard.alias_export_route
GET                    /dashboard/alias_log/<int:alias_id>                     -> dashboard.alias_log
GET                    /dashboard/alias_log/<int:alias_id>/<int:page_id>       -> dashboard.alias_log
GET,POST               /dashboard/alias_transfer/receive                       -> dashboard.alias_transfer_receive_route
GET,POST               /dashboard/alias_transfer/send/<int:alias_id>/          -> dashboard.alias_transfer_send_route
GET,POST               /dashboard/api_key                                      -> dashboard.api_key
GET,POST               /dashboard/app                                          -> dashboard.app_route
GET,POST               /dashboard/batch_import                                 -> dashboard.batch_import_route
GET,POST               /dashboard/billing                                      -> dashboard.billing
GET,POST               /dashboard/block_contact/<int:contact_id>               -> dashboard.block_contact
GET,POST               /dashboard/cancel_email_change                          -> dashboard.cancel_email_change
GET                    /dashboard/coinbase_checkout                            -> dashboard.coinbase_checkout_route
GET,POST               /dashboard/contact/<int:contact_id>/                    -> dashboard.contact_detail_route
POST                   /dashboard/contacts/<int:contact_id>/toggle             -> dashboard.toggle_contact
GET,POST               /dashboard/coupon                                       -> dashboard.coupon_route
GET,POST               /dashboard/custom_alias                                 -> dashboard.custom_alias
GET,POST               /dashboard/custom_domain                                -> dashboard.custom_domain
GET,POST               /dashboard/delete_account                               -> dashboard.delete_account
GET,POST               /dashboard/directory                                    -> dashboard.directory
GET,POST               /dashboard/domains/<int:custom_domain_id>/auto-create   -> dashboard.domain_detail_auto_create
GET,POST               /dashboard/domains/<int:custom_domain_id>/dns           -> dashboard.domain_detail_dns
GET,POST               /dashboard/domains/<int:custom_domain_id>/info          -> dashboard.domain_detail
GET,POST               /dashboard/domains/<int:custom_domain_id>/trash         -> dashboard.domain_detail_trash
GET,POST               /dashboard/enter_sudo                                   -> dashboard.enter_sudo
GET,POST               /dashboard/fido_manage                                  -> dashboard.fido_manage
GET,POST               /dashboard/fido_setup                                   -> dashboard.fido_setup
GET,POST               /dashboard/lifetime_licence                             -> dashboard.lifetime_licence
GET,POST               /dashboard/mailbox                                      -> dashboard.mailbox_route
GET,POST               /dashboard/mailbox/<int:mailbox_id>/                    -> dashboard.mailbox_detail_route
GET,POST               /dashboard/mailbox/<int:mailbox_id>/cancel_email_change -> dashboard.cancel_mailbox_change_route
GET                    /dashboard/mailbox/confirm_change                       -> dashboard.mailbox_confirm_change_route
GET                    /dashboard/mailbox_verify                               -> dashboard.mailbox_verify
GET,POST               /dashboard/mfa_cancel                                   -> dashboard.mfa_cancel
GET,POST               /dashboard/mfa_setup                                    -> dashboard.mfa_setup
GET,POST               /dashboard/notification/<notification_id>               -> dashboard.notification_route
GET,POST               /dashboard/notifications                                -> dashboard.notifications_route
GET,POST               /dashboard/pricing                                      -> dashboard.pricing
GET,POST               /dashboard/referral                                     -> dashboard.referral_route
GET,POST               /dashboard/refused_email                                -> dashboard.refused_email_route
GET,POST               /dashboard/resend_email_change                          -> dashboard.resend_email_change
GET,POST               /dashboard/setting                                      -> dashboard.setting
GET,POST               /dashboard/setup_done                                   -> dashboard.setup_done
GET,POST               /dashboard/subdomain                                    -> dashboard.subdomain_route
GET                    /dashboard/subscription_success                         -> dashboard.subscription_success
GET,POST               /dashboard/support                                      -> dashboard.support_route
POST                   /dashboard/unlink_proton_account                        -> dashboard.unlink_proton_account
GET,POST               /dashboard/unsubscribe/<int:alias_id>                   -> dashboard.unsubscribe
GET                    /dashboard/unsubscribe/encoded/<encoded_request>        -> dashboard.encoded_unsubscribe
GET,POST               /developer/                                             -> developer.index
GET,POST               /developer/clients/<client_id>                          -> developer.client_detail
GET,POST               /developer/clients/<client_id>/advanced                 -> developer.client_detail_advanced
GET,POST               /developer/clients/<client_id>/oauth_endpoint           -> developer.client_detail_oauth_endpoint
GET,POST               /developer/clients/<client_id>/oauth_setting            -> developer.client_detail_oauth_setting
GET,POST               /developer/clients/<client_id>/referral                 -> developer.client_detail_referral
GET,POST               /developer/new_client                                   -> developer.new_client
GET,POST               /discover/                                              -> discover.index
GET                    /dnt                                                    -> do_not_track
GET                    /exception                                              -> monitor.test_exception
GET                    /favicon.ico                                            -> favicon
GET                    /git                                                    -> monitor.git_sha1
GET                    /health                                                 -> healthcheck
GET                    /internal/exit-sudo-mode                                -> internal.exit_sudo_mode
GET                    /internal/integrations/proton                           -> internal.set_enable_proton_cookie
GET                    /jwks                                                   -> jwks
GET                    /live                                                   -> monitor.live
GET,POST               /oauth/authorize                                        -> oauth.authorize
GET                    /oauth/me                                               -> oauth.user_info
GET,POST               /oauth/token                                            -> oauth.token
GET                    /oauth/user_info                                        -> oauth.user_info
GET                    /oauth/userinfo                                         -> oauth.user_info
GET,POST               /oauth2/authorize                                       -> oauth.authorize
GET                    /oauth2/me                                              -> oauth.user_info
GET,POST               /oauth2/token                                           -> oauth.token
GET                    /oauth2/user_info                                       -> oauth.user_info
GET                    /oauth2/userinfo                                        -> oauth.user_info
GET                    /onboarding/                                            -> onboarding.index
GET                    /onboarding/account_activated                           -> onboarding.account_activated
GET                    /onboarding/extension_redirect                          -> onboarding.extension_redirect
GET,POST               /onboarding/final                                       -> onboarding.final
GET                    /onboarding/setup                                       -> onboarding.setup
GET,POST               /onboarding/setup_done                                  -> onboarding.setup_done
GET,POST               /paddle                                                 -> paddle
GET,POST               /paddle_coupon                                          -> paddle_coupon
GET,POST               /phone/                                                 -> phone.index
GET,POST               /phone/provider1/sms                                    -> phone.provider1_sms
GET,POST               /phone/provider2/sms                                    -> phone.provider2_sms
GET,POST               /phone/reservation/<int:reservation_id>                 -> phone.reservation_route
GET,POST               /phone/twilio/sms                                       -> phone.twilio_sms
GET                    /static/<path:filename>                                 -> static
```

### File:line grounding

- Port: `server.py:L588` `app.run(debug=True, port=7777)`; live `/proc/net/tcp` local address
  `0100007F:1E61` = `127.0.0.1:7777`, state `0A` = LISTEN. `Dockerfile:L44` `EXPOSE 7777`;
  `Dockerfile:L47` prod `CMD ["gunicorn","wsgi:app","-b","0.0.0.0:7777",...]` (**Inferred**, not run).
- `/health`: `server.py:L213-215` `@app.route("/health")` / `def healthcheck():` / `return "success", 200`.
- Blueprint registration: `server.py:L233-246` `register_blueprints(app)`:
  `auth_bp` (L234), `monitor_bp` (L235), `dashboard_bp` (L236), `developer_bp` (L237),
  `phone_bp` (L238), `oauth_bp` @ `/oauth` (L240), `oauth_bp` @ `/oauth2` (L241),
  `onboarding_bp` (L242), `discover_bp` (L244), `internal_bp` (L245), `api_bp` (L246).
- Blueprint prefixes: `app/api/base.py:L11` (`/api`), `app/auth/base.py:L4` (`/auth`),
  `app/dashboard/base.py:L6` (`/dashboard`), `app/developer/base.py:L6` (`/developer`),
  `app/discover/base.py:L6` (`/discover`), `app/internal/base.py:L6` (`/internal`),
  `app/monitor/base.py:L3` (`/` — the discrepancy), `app/oauth/base.py:L4` (`/oauth`),
  `app/onboarding/base.py:L6` (`/onboarding`), `app/phone/base.py:L6` (`/phone`).

### Observed vs Inferred

- **Observed:** the single loopback listener on `127.0.0.1:7777` (raw `/proc/net/tcp` line); the
  `/health` `200`/`success`; the exact route count **292** and the complete route map, stable across
  two independent `create_app()` builds; the `monitor_bp` `/`-prefix producing `/git`, `/live`,
  `/exception`; and `oauth_bp` being registered twice (`/oauth` + `/oauth2`).
- **Inferred:** the production `gunicorn` bind to `0.0.0.0:7777` — read from the `Dockerfile`, not
  run. The route map above is the dev app from `create_app()`; production uses the **same** factory
  (`wsgi.py:L3`), so the route set is expected to be identical, but that was not separately run.

## Q5 — How is a single authenticated request handled? (PRIMARY)

### Direct answer (Observed)

A request first reaches the app through the WSGI middleware `ProxyFix(app.wsgi_app, x_for=1,
x_host=1)` (`server.py:L142`), which rewrites `request.remote_addr` from `X-Forwarded-For` and the
URL host from `X-Forwarded-Host` (both proven below). Flask then runs the **three app-wide
`before_request` hooks in this observed order** — `[0] Limiter.__check_request_limit` (Flask-Limiter),
`[1] set_index_page.<locals>.before_request` (`server.py:L258`, which stamps `g.start_time`),
`[2] make_session_permanent` (`server.py:L205`) — before dispatching to the view.

**Identity is determined by one of two independent mechanisms**, chosen by how the request
authenticates:

- **Web session (Flask-Login).** The signed session carries `_user_id`, whose value is the user's
  `alternative_id` **UUID** (not the integer primary key) because `User.get_id()` returns
  `alternative_id` when set (`app/models.py:L595-599`). On each request Flask-Login calls
  `load_user(alternative_id)` (`server.py:L221`), which does `User.get_by(alternative_id=...)`, sets
  the Sentry user identity (`L224` `sentry_sdk.set_user({"email": ..., "id": ...})`), and returns
  `None` if the user is `disabled` (`L225-226`) or not `is_active()` (`L227-228`), otherwise
  populating the `current_user` proxy.
  Observed: after login, `_user_id = bfb8024e-6c00-4c33-ac2c-e60bf3b7003d`, which equals john's
  `alternative_id` in the DB (`1|bfb8024e-6c00-4c33-ac2c-e60bf3b7003d`).
- **API key (`Authentication` header).** `authorize_request()` (`app/api/base.py:L16`) reads the
  `Authentication` header (`L17`), looks up `ApiKey.get_by(code=...)` (`L18`); on a valid key it
  **first updates usage stats** (`last_used = arrow.now()`, `times += 1`, `Session.commit()`,
  `L30-32`) and **only then** enforces guards (`disabled → 403`, `not is_active() → 401`), finally
  setting `g.user = api_key.user` (`L34`). With no/invalid key it falls back to the web session
  (`if current_user.is_authenticated:` `L21` → `g.user = current_user` `L25`) or returns
  `401 {"error": "Wrong api key"}` (`L27`). `g.api_key` is assigned for both branches (`L42`), so it
  is `None` on the session-fallback path.

**Propagation:** web view handlers read `current_user` (the Flask-Login proxy, backed by the session
+ `load_user`); the API view `/api/user_info` reads `g.user`. The helper `get_current_user()`
(`server.py:L332-336`) returns `g.user` else `current_user`, but at runtime its **only** consumer is
the 429 rate-limit logger (`server.py:L367`) — it is *not* the general convergence point for normal
requests. Each non-excluded request ends with the `after_request` logger (`server.py:L273-296`,
`LOG.d(` at `L284`) emitting one `SL` line, then the registered `teardown_appcontext` callback runs `Session.remove()`
(Inferred — the callback registration is observed, and Flask fires teardown callbacks by guarantee,
but the per-request invocation itself is not separately logged).

**Session storage has two backends, both exercised.** With `MEM_STORE_URI` set (`server.py:L163-165`
→ `initialize_redis_services`), a server-side `RedisSessionStore` keeps the data under
`session:<id>` and the cookie carries only the **signed session id**; anonymous sessions get
`ttl=300`, authenticated sessions `ttl=604800` (7 days). With `MEM_STORE_URI` unset, Flask's default
**signed-cookie** backend stores the whole session payload *inside the cookie*, zlib-compressed and
HMAC-signed — readable with **no secret key** (⇒ signed/tamper-evident, **not** encrypted); the
`_user_id` there is the same `alternative_id` UUID.

### Command(s) run

The core web/API evidence comes from one bounded server run, **RUN-Q5-REDIS** (Redis backend,
`parent=5660 child=5673`); a supplementary run captured the `X-Forwarded-Host` proof
(`parent=5844 child=5856`); the signed-cookie backend was exercised twice as **RUN-Q5-COOKIE**
(`MEM_STORE_URI=""`); the canonical owner-revoke lifecycle of subsection **(H′)** was captured as
**RUN-Q5-REVOKE-CANONICAL** (Redis backend); and the session-lifecycle edge behaviours of subsection
**(I)** — login non-rotation (Redis), signed-cookie logout replay (`MEM_STORE_URI=""`), and negative
CSRF — were captured alongside them. Every server is launched through the canonical dev entry point
`/app/venv/bin/python server.py`. Disposable users D1 (disabled), D2 (`delete_on` in the future) and
D3 (web login-then-mutate) are created solely for the guard tests — through the app's own model layer,
a **supplemental, non-canonical** fixture (there is no self-service HTTP path to disable an account
or schedule a future deletion) — and are deleted at the end of the phase; **john@wick.com is never
mutated**.

```bash
# ---- prefix sourced in every command (per setup) ----
cd /app && set -a && . /tmp/sl_env.sh && set +a && unset EVENT_WEBHOOK_DISABLE \
  && export GNUPGHOME=/tmp/sl_clean_gnupg
BASE=http://localhost:7777

# (A) app-wide before_request hook ORDER (same factory local_main() uses):
/app/venv/bin/python - <<'PY'
from server import create_app
app = create_app()
print("app.before_request_funcs (None key = app-wide, runs for every request):")
for key, funcs in app.before_request_funcs.items():
    print("  blueprint key:", key)
    for i, fn in enumerate(funcs):
        print("    [%d] %s  (module %s)" % (i, getattr(fn,"__qualname__",fn.__name__), fn.__module__))
PY

# (A) ProxyFix proof on a LOGGED endpoint (X-Forwarded-For + X-Forwarded-Host):
curl -s -D - -o /dev/null -H 'X-Forwarded-For: 203.0.113.7' \
  -H 'X-Forwarded-Host: proxied.example.test' "$BASE/auth/login"          # -> proxyfix_headers.txt
# supplementary anonymous /dashboard/ with forwarded host (redirect host proves x_host=1):
curl -s -D - -o /dev/null -H 'X-Forwarded-For: 198.51.100.9' \
  -H 'X-Forwarded-Host: proxied.example.test' "$BASE/dashboard/"          # -> proxyfix_xhost_headers.txt

# (B) WEB PATH (john@wick.com / password) with a disposable cookie jar:
curl -s -D web_get_login_headers.txt -o body.html -c cj_john.txt "$BASE/auth/login"   # anon GET -> CSRF + anon session
CSRF=$(grep -oE 'name="csrf_token"[^>]*value="[^"]+"' body.html | head -1 | sed -E 's/.*value="([^"]+)".*/\1/')
# (Redis session dump BEFORE login) -> redis_anon.txt
curl -s -D web_post_login_headers.txt -o /dev/null -b cj_john.txt -c cj_john.txt \
  --data-urlencode "email=john@wick.com" --data-urlencode "password=password" \
  --data-urlencode "csrf_token=$CSRF" "$BASE/auth/login"                  # POST -> 302 /dashboard/
# (Redis session dump AFTER login) -> redis_auth.txt
psql -h localhost -U test -d test -t -A -F'|' \
  -c "select id, alternative_id from users where email='john@wick.com';"  # -> db_john_identity.txt
curl -s -D web_dashboard_headers.txt -o /dev/null -b cj_john.txt "$BASE/dashboard/"    # authenticated GET -> 200

# (C) API PATH:
curl -s -i -H 'Content-Type: application/json' \
  -d '{"email":"john@wick.com","password":"password","device":"blitzy-q5-device"}' \
  "$BASE/api/auth/login"                                                  # canonical key acquisition -> api_auth_login.txt
curl -s -i -H 'Authentication: code'                "$BASE/api/user_info" # [A] valid  -> 200  (+ stats before/after)
curl -s -i -H 'Authentication: this-is-a-wrong-key' "$BASE/api/user_info" # [B] wrong  -> 401
curl -s -i                                          "$BASE/api/user_info" # [C] absent -> 401
curl -s -i -b cj_john.txt                           "$BASE/api/user_info" # [D] session fallback -> 200 (g.api_key=None)
# --- SUPPLEMENTAL (non-canonical) fixture setup for the account-guard tests ---------------------
# The `disabled` state and a *future* `delete_on` have NO self-service HTTP path (disabling is an
# admin/DB action; scheduled deletion is set by a separate internal flow), so the disposable guard
# users D1/D2/D3 and one API key each are created through the app's OWN model layer, and the guard
# state is then set directly in the DB. These writes are a FIXTURE, not the behaviour under test;
# the CANONICAL signal under test is the HTTP status the endpoint returns on that state.
/app/venv/bin/python - <<'PY' > /tmp/guard_fixture.env
from app.db import Session
from app.models import User, ApiKey
import uuid
lines = []
for tag in ("D1", "D2", "D3"):
    u = User.create(email=f"blitzy-{tag}-{uuid.uuid4().hex[:8]}@sl.local",
                    password="password", name=tag, activated=True); Session.commit()
    k = ApiKey.create(user_id=u.id, name=f"blitzy-{tag}-key"); Session.commit()
    lines += [f"{tag}UID={u.id}", f"{tag}KEY={k.code}"]
print("\n".join(lines))
PY
set -a; . /tmp/guard_fixture.env; set +a   # binds D1UID/D1KEY/D2UID/D2KEY/D3UID/D3KEY (60-char codes; values never printed into this doc)
PGP="PGPASSWORD=test psql -h localhost -U test -d test"   # test:test = public DEV creds (see Security scope)

$PGP -c "update users set disabled=true where id=$D1UID;"                # D1 -> disabled (fixture)
curl -s -i -H "Authentication: $D1KEY"              "$BASE/api/user_info" # (d-1) disabled account -> 403 "Disabled account"
$PGP -c "update users set delete_on=(now() + interval '7 days') where id=$D2UID;"  # D2 -> future delete_on (fixture)
curl -s -i -H "Authentication: $D2KEY"              "$BASE/api/user_info" # (d-2) inactive account -> 401 "Account does not exist"

# (D) WEB load_user guards on disposable D3 (uid $D3UID; resolved to 5 in RUN-Q5-REDIS, as the output
#     blocks below show). The DB writes are a SUPPLEMENTAL fixture (same no-self-service-path reason
#     as above); the CANONICAL signal is the HTTP status of the *identical* GET /dashboard/ issued on
#     the same cookie jar before/after each mutation:
$PGP -c "update users set disabled=true  where id=$D3UID;"   # -> GET /dashboard/ 302 login
$PGP -c "update users set disabled=false where id=$D3UID;"   # -> GET /dashboard/ 200
$PGP -c "update users set delete_on=(now() + interval '7 days') where id=$D3UID;"  # -> GET /dashboard/ 302 login

# (F) signed-cookie backend (twice): disable Redis sessions, log in, decode the cookie:
export MEM_STORE_URI=""              # Flask default signed-cookie interface
/app/venv/bin/python server.py &     # RUN-Q5-COOKIE-N (fresh server on :7777)
# web login exactly as in (B): GET /auth/login (-c cj_cookie.txt) -> scrape csrf_token ->
#   POST /auth/login with email/password/csrf_token (-b -c cj_cookie.txt) -> 302, then decode:
curl -s -D cookie_login_headers.txt -o cbody.html -c cj_cookie.txt "$BASE/auth/login"
CSRF=$(grep -oE 'name="csrf_token"[^>]*value="[^"]+"' cbody.html | head -1 | sed -E 's/.*value="([^"]+)".*/\1/')
curl -s -D cookie_post.txt -o /dev/null -b cj_cookie.txt -c cj_cookie.txt \
  --data-urlencode "email=john@wick.com" --data-urlencode "password=password" \
  --data-urlencode "csrf_token=$CSRF" "$BASE/auth/login"                 # -> 302 /dashboard/
# take the 'slapp' cookie value (.eJw<payload>.<ts>.<sig>) and decode the payload with NO secret key:
SLAPP=$(awk -F'\t' '$6=="slapp"{print $7}' cj_cookie.txt | tail -1)
/app/venv/bin/python - "$SLAPP" <<'PY'   # base64url + zlib, no key -> proves signed-not-encrypted
import sys, base64, zlib, json
blob = sys.argv[1].split('.')[0]            # the .eJw... payload segment before the timestamp/sig
if blob.startswith('.'): blob = blob[1:]
raw = zlib.decompress(base64.urlsafe_b64decode(blob + '=' * (-len(blob) % 4)))
print(json.dumps(json.loads(raw)))          # csrf_token masked in the doc's output block
PY

# (G) WEB LOGOUT lifecycle (fresh login on jar cj_logout.txt, then logout + post-logout probe):
curl -s -D - -o login_body.html -c cj_logout.txt "$BASE/auth/login"                    # anon GET -> CSRF
CSRF=$(grep -oE 'name="csrf_token"[^>]*value="[^"]+"' login_body.html | head -1 | sed -E 's/.*value="([^"]+)".*/\1/')
curl -s -o /dev/null -b cj_logout.txt -c cj_logout.txt \
  --data-urlencode "email=john@wick.com" --data-urlencode "password=password" \
  --data-urlencode "csrf_token=$CSRF" "$BASE/auth/login"                               # POST -> 302 /dashboard/ (login)
curl -s -D web_before_logout_dashboard.txt -o /dev/null -b cj_logout.txt "$BASE/dashboard/"  # BEFORE -> 200
curl -s -D web_logout.txt -o /dev/null -b cj_logout.txt -c cj_logout.txt "$BASE/auth/logout" # -> 302 + Set-Cookie Max-Age=0 (slapp/mfa/dark-mode)
curl -s -D web_after_logout_dashboard.txt -o /dev/null -b cj_logout.txt "$BASE/dashboard/"   # AFTER -> 302 login
# (Redis session dump BEFORE vs AFTER logout -> redis_logout.txt; the authenticated key is purged)

# (H) API REVOKED-KEY over HTTP on a concrete endpoint (/api/user_info):
#     acquire + use + reuse are over HTTP; the revoke step here is a SUPPLEMENTAL direct-SQL DELETE
#     (labelled non-canonical). The canonical owner-revoke transition itself is exercised over HTTP
#     in subsection (H') below; the observed signal in BOTH is the 401 the reused key receives.
curl -s -i -H 'Content-Type: application/json' \
  -d '{"email":"john@wick.com","password":"password","device":"blitzy-q5-revoke"}' \
  "$BASE/api/auth/login" -o api_revoke_acquire.txt                                     # acquire (HTTP)
K=$(/app/venv/bin/python -c "import json,sys,re; b=open('api_revoke_acquire.txt').read().split('\r\n\r\n',1)[-1]; print(json.loads(b)['api_key'])")  # bind K (value never printed into this doc)
curl -s -i -H "Authentication: $K" "$BASE/api/user_info"                               # use (HTTP)   -> 200  api_revoke_use.txt
PGP="PGPASSWORD=test psql -h localhost -U test -d test"                                # test:test = public DEV creds
$PGP -c "delete from api_key where code='$K';"                                         # revoke: SUPPLEMENTAL SQL DELETE (non-canonical; see (H') for the canonical HTTP revoke)
curl -s -i -H "Authentication: $K" "$BASE/api/user_info"                               # reuse (HTTP) -> 401 Wrong api key  api_revoke_after.txt
```


### Complete output (secret-masked)

_(Bearer material — `Set-Cookie` HMAC signatures, the signed-cookie payload blob, acquired API-key
values, and CSRF tokens — is masked below per the Evidence-discipline note; every block is otherwise
structurally verbatim, and all non-secret fields, statuses, headers, bodies and log lines are shown.)_

> HTTP header blocks below are shown LF-normalized (`tr -d '\r'`) per the global note in the
> *Investigation environment* section; the on-disk captures use CRLF line endings as curl emitted them.

#### (A) Common entry — `before_request` hook order + `ProxyFix`

`before_request_funcs.txt` (the app-wide hooks, in run order, from the same `create_app()` factory
`local_main()` uses):

```text
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-13 18:54:39,065 - SL - DEBUG - 5688 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
app.before_request_funcs (None key = app-wide, runs for every request):
  blueprint key: None
    [0] Limiter.__check_request_limit  (module flask_limiter.extension)
    [1] set_index_page.<locals>.before_request  (module server)
    [2] create_app.<locals>.make_session_permanent  (module server)
```

`ProxyFix` proof #1 — `proxyfix_headers.txt` (request carried `X-Forwarded-For: 203.0.113.7`); the
corresponding `SL` log line (below, PID 5673) shows `remote_addr = 203.0.113.7`, so `x_for=1` is
honored:

```text
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 342103
Set-Cookie: slapp=24878969-dfd3-4f8a-9717-2a376213144a.<REDACTED:slapp-hmac-signature>; Expires=Mon, 20-Jul-2026 18:54:40 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Mon, 13 Jul 2026 18:54:40 GMT
```

`ProxyFix` proof #2 — `proxyfix_xhost_headers.txt` (anonymous `GET /dashboard/` with
`X-Forwarded-Host: proxied.example.test`); the `302` redirect `Location` host is
`proxied.example.test`, so `x_host=1` is honored (the redirect itself is Flask-Login's
`unauthorized()` → `login_view=auth.login`):

```text
HTTP/1.0 302 FOUND
Content-Type: text/html; charset=utf-8
Content-Length: 277
Location: http://proxied.example.test/auth/login?next=%2Fdashboard%2F%3F
Set-Cookie: slapp=08dc93ff-1093-4b51-a64d-e088e1e9527c.<REDACTED:slapp-hmac-signature>; Expires=Mon, 20-Jul-2026 18:56:48 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Mon, 13 Jul 2026 18:56:48 GMT
```

#### (B) Web-session path — identity determination & propagation

**(b-1) Anonymous `GET /auth/login`** — `web_get_login_headers.txt`. A fresh `slapp` cookie is set
(value = `session_id.signature`; the data lives server-side in Redis):

```text
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 215139
Set-Cookie: slapp=83213b1a-73bf-4dc8-85d5-47896a8f8307.<REDACTED:slapp-hmac-signature>; Expires=Mon, 20-Jul-2026 18:54:40 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Mon, 13 Jul 2026 18:54:40 GMT
```

Redis session state **BEFORE** login — `redis_anon.txt` (both anonymous sessions carry only
`csrf_token`, no `_user_id`, and `ttl=300`):

```text
session:24878969-dfd3-4f8a-9717-2a376213144a  ttl=300
    decoded: {'_permanent': True, '_fresh': False, 'csrf_token': '<REDACTED:csrf_token-value>'}
session:83213b1a-73bf-4dc8-85d5-47896a8f8307  ttl=300
    decoded: {'_permanent': True, '_fresh': False, 'csrf_token': '<REDACTED:csrf_token-value>'}
```

**(b-2) `POST /auth/login`** (john@wick.com / password) — `web_post_login_headers.txt`. `302` to
`/dashboard/`, the **same** `slapp` session id is retained:

```text
HTTP/1.0 302 FOUND
Content-Type: text/html; charset=utf-8
Content-Length: 229
Location: http://localhost:7777/dashboard/
Set-Cookie: slapp=83213b1a-73bf-4dc8-85d5-47896a8f8307.<REDACTED:slapp-hmac-signature>; Expires=Mon, 20-Jul-2026 18:54:40 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Mon, 13 Jul 2026 18:54:40 GMT
```

Redis session state **AFTER** login — `redis_auth.txt`. The **same** session id `83213b1a…`
transitioned `ttl=300 → ttl=604800` (7 days) and gained `_user_id`, `_fresh: True`, `_id`,
`sudo_time`; the untouched anon session `24878969…` merely ticked down to `ttl=299`:

```text
session:24878969-dfd3-4f8a-9717-2a376213144a  ttl=299
    decoded: {'_permanent': True, '_fresh': False, 'csrf_token': '<REDACTED:csrf_token-value>'}
session:83213b1a-73bf-4dc8-85d5-47896a8f8307  ttl=604800
    decoded: {'_permanent': True, '_fresh': True, 'csrf_token': '<REDACTED:csrf_token-value>', '_user_id': 'bfb8024e-6c00-4c33-ac2c-e60bf3b7003d', '_id': 'b03643a8a515eae10966eb5799d0b928b1fae8daeb0b0a33ba33f35b5fd7e09d574a2b38cd3af3ce53bdb464d28d2c515533674c8b087cba7d49abfa2cc9de57', 'sudo_time': 1783968880}
```

**Identity proof** — `db_john_identity.txt` (`select id, alternative_id …`). The session's
`_user_id` (`bfb8024e-…-003d`) equals john's **`alternative_id`**, confirming `get_id()` stores the
UUID, **not** the integer primary key `1`:

```text
1|bfb8024e-6c00-4c33-ac2c-e60bf3b7003d
```

**(b-3) Authenticated `GET /dashboard/`** — `web_dashboard_headers.txt` (`200`; `current_user` is
john, resolved via `load_user`):

```text
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 762858
Set-Cookie: slapp=83213b1a-73bf-4dc8-85d5-47896a8f8307.<REDACTED:slapp-hmac-signature>; Expires=Mon, 20-Jul-2026 18:54:41 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Mon, 13 Jul 2026 18:54:41 GMT
```


#### (C) API path — `Authentication` header

**(c-1) Canonical key acquisition** `POST /api/auth/login` — `api_auth_login.txt`. The response is
pretty-printed JSON (note the genuine trailing spaces after each comma — real bytes, not edited):

```text
HTTP/1.0 200 OK
Content-Type: application/json
Content-Length: 178
Access-Control-Allow-Origin: *
Set-Cookie: slapp=491f6af4-c9cd-4dae-a8de-93227c898d21.<REDACTED:slapp-hmac-signature>; Expires=Mon, 20-Jul-2026 18:54:41 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Mon, 13 Jul 2026 18:54:41 GMT

{
  "api_key": "<REDACTED:api_key-value>", 
  "email": "john@wick.com", 
  "mfa_enabled": false, 
  "mfa_key": null, 
  "name": "John Wick"
}
```

**(c-2) `[A]` valid key + stats side effect** — the seeded key `code` is used so the before/after
usage counters land on a known row. `api_stats_before.txt` then `api_valid.txt` (`200`) then
`api_stats_after.txt`. The stats row moved `times 0 → 1` and `last_used ∅ → 2026-07-13
18:54:41.483957`, confirming the commit happens **before** the guards:

```text
1|code|0|
```
```text
HTTP/1.0 200 OK
Content-Type: application/json
Content-Length: 279
Access-Control-Allow-Origin: *
Set-Cookie: slapp=0b692651-8010-4892-8a4c-7aedb594c26d.<REDACTED:slapp-hmac-signature>; Expires=Mon, 20-Jul-2026 18:54:41 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Mon, 13 Jul 2026 18:54:41 GMT

{
  "can_create_reverse_alias": true, 
  "connected_proton_address": null, 
  "email": "john@wick.com", 
  "in_trial": false, 
  "is_premium": true, 
  "max_alias_free_plan": 3, 
  "name": "John Wick", 
  "profile_picture_url": "http://localhost/static/upload/profile_pic.svg"
}
```
```text
1|code|1|2026-07-13 18:54:41.483957
```

**(c-3) `[B]` wrong key** — `api_wrong.txt` (`401`, body from `app/api/base.py:L27`, **not** the
generic 401 errorhandler which would say `"Unauthorized"`):

```text
HTTP/1.0 401 UNAUTHORIZED
Content-Type: application/json
Content-Length: 31
Access-Control-Allow-Origin: *
Set-Cookie: slapp=5bc49879-7749-4ef1-95a1-0f1e0b2bb86d.<REDACTED:slapp-hmac-signature>; Expires=Mon, 20-Jul-2026 18:54:41 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Mon, 13 Jul 2026 18:54:41 GMT

{
  "error": "Wrong api key"
}
```

**(c-4) `[C]` absent** (no header, no cookie) — `api_absent.txt` (`401`, same `Wrong api key`):

```text
HTTP/1.0 401 UNAUTHORIZED
Content-Type: application/json
Content-Length: 31
Access-Control-Allow-Origin: *
Set-Cookie: slapp=cdf09b42-9f22-4741-bd30-b5a7357533b7.<REDACTED:slapp-hmac-signature>; Expires=Mon, 20-Jul-2026 18:54:41 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Mon, 13 Jul 2026 18:54:41 GMT

{
  "error": "Wrong api key"
}
```

**(c-5) `[D]` session fallback** (no `Authentication` header, **with** john's web cookie) —
`api_session_fallback.txt` (`200`; `g.user = current_user`, and `g.api_key = None` because the
valid-key branch that would set it never ran):

```text
HTTP/1.0 200 OK
Content-Type: application/json
Content-Length: 279
Access-Control-Allow-Origin: *
Set-Cookie: slapp=83213b1a-73bf-4dc8-85d5-47896a8f8307.<REDACTED:slapp-hmac-signature>; Expires=Mon, 20-Jul-2026 18:54:41 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Mon, 13 Jul 2026 18:54:41 GMT

{
  "can_create_reverse_alias": true, 
  "connected_proton_address": null, 
  "email": "john@wick.com", 
  "in_trial": false, 
  "is_premium": true, 
  "max_alias_free_plan": 3, 
  "name": "John Wick", 
  "profile_picture_url": "http://localhost/static/upload/profile_pic.svg"
}
```

#### (D) Account guards — API (`authorize_request`) and web (`load_user`)

**(d-1) API disabled account** (D1 key) — `api_disabled.txt` (`403 {"error": "Disabled account"}`,
`app/api/base.py:L37`):

```text
HTTP/1.0 403 FORBIDDEN
Content-Type: application/json
Content-Length: 34
Access-Control-Allow-Origin: *
Set-Cookie: slapp=3860bfb9-8a64-415d-882b-b7b93152bfbc.<REDACTED:slapp-hmac-signature>; Expires=Mon, 20-Jul-2026 18:54:41 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Mon, 13 Jul 2026 18:54:41 GMT

{
  "error": "Disabled account"
}
```

**(d-2) API inactive account** (D2 key, future `delete_on`) — `api_inactive.txt`
(`401 {"error": "Account does not exist"}`, `app/api/base.py:L40`):

```text
HTTP/1.0 401 UNAUTHORIZED
Content-Type: application/json
Content-Length: 40
Access-Control-Allow-Origin: *
Set-Cookie: slapp=af52c895-f954-448a-890f-578d96f893b3.<REDACTED:slapp-hmac-signature>; Expires=Mon, 20-Jul-2026 18:54:41 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Mon, 13 Jul 2026 18:54:41 GMT

{
  "error": "Account does not exist"
}
```

**(d-3) Web `load_user` guards** — D3 logs in, then the DB row is mutated between identical
`GET /dashboard/` requests on the same cookie jar. `load_user` returns `None` when the user is
`disabled` (`server.py:L225-226`) or not `is_active()` (`L227-228`), which Flask-Login turns into a
`302` to the login page; restoring the row returns `200`. The five blocks below are the complete
responses (`curl -s -D <file>`), secret-masked per the Evidence-discipline note, in sequence — the disposable D3 login, then the
baseline / disabled / restored / inactive `GET /dashboard/` on the same cookie jar. The `slapp`
cookie is the same disposable Redis session throughout (`b8c1f1f6-…`, flushed in the end-of-phase
cleanup); the interpretation of each block is stated in the prose line preceding it.

D3 login → `302` to `/dashboard/` (`d3_post_login_headers.txt`):

```text
HTTP/1.0 302 FOUND
Content-Type: text/html; charset=utf-8
Content-Length: 229
Location: http://localhost:7777/dashboard/
Set-Cookie: slapp=b8c1f1f6-d45a-48cc-b900-105a22c08b5b.<REDACTED:slapp-hmac-signature>; Expires=Mon, 20-Jul-2026 18:54:41 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Mon, 13 Jul 2026 18:54:41 GMT
```

Baseline — row active → `load_user` returns the user → `200 OK` (`d3_before_headers.txt`):

```text
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 683139
Set-Cookie: slapp=b8c1f1f6-d45a-48cc-b900-105a22c08b5b.<REDACTED:slapp-hmac-signature>; Expires=Mon, 20-Jul-2026 18:54:42 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Mon, 13 Jul 2026 18:54:42 GMT
```

`update users set disabled=true where id=5` → `load_user` returns `None` (`server.py:L225-226`) →
`302` to `auth.login` (`d3_disabled_headers.txt`):

```text
HTTP/1.0 302 FOUND
Content-Type: text/html; charset=utf-8
Content-Length: 277
Location: http://localhost:7777/auth/login?next=%2Fdashboard%2F%3F
Set-Cookie: slapp=b8c1f1f6-d45a-48cc-b900-105a22c08b5b.<REDACTED:slapp-hmac-signature>; Expires=Mon, 20-Jul-2026 18:54:42 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Mon, 13 Jul 2026 18:54:42 GMT
```

`update users set disabled=false where id=5` (restored) → `200 OK` (`d3_restored_headers.txt`):

```text
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 559418
Set-Cookie: slapp=b8c1f1f6-d45a-48cc-b900-105a22c08b5b.<REDACTED:slapp-hmac-signature>; Expires=Mon, 20-Jul-2026 18:54:42 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Mon, 13 Jul 2026 18:54:42 GMT
```

`update users set delete_on=(now() + interval '7 days') where id=5` → `is_active()` false
(`server.py:L227-228`) → `302` to `auth.login` (`d3_inactive_headers.txt`):

```text
HTTP/1.0 302 FOUND
Content-Type: text/html; charset=utf-8
Content-Length: 277
Location: http://localhost:7777/auth/login?next=%2Fdashboard%2F%3F
Set-Cookie: slapp=b8c1f1f6-d45a-48cc-b900-105a22c08b5b.<REDACTED:slapp-hmac-signature>; Expires=Mon, 20-Jul-2026 18:54:42 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Mon, 13 Jul 2026 18:54:42 GMT
```


#### (E) The `SL` per-request log (`after_request`), correlated by PID 5673

`sl_request_logs.txt` — every non-excluded request from RUN-Q5-REDIS in emission order, all bearing
PID `5673`. Format: `<ts> - SL - DEBUG - <PID> - "/app/server.py:284" - after_request() -  -
<remote_addr> <method> <path> <request.args> <status>, takes <seconds>`. `/health`, `/static`,
`/git`, `/favicon.ico`, `/_debug_toolbar`, `/admin/static` never appear (excluded at
`server.py:L276-281`). The first line's `203.0.113.7` is the `X-Forwarded-For` from the ProxyFix
probe:

```text
2026-07-13 18:54:40,256 - SL - DEBUG - 5673 - "/app/server.py:284" - after_request() -  - 203.0.113.7 GET /auth/login ImmutableMultiDict([]) 200, takes 0.11127161979675293
2026-07-13 18:54:40,290 - SL - DEBUG - 5673 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.021816015243530273
2026-07-13 18:54:40,668 - SL - DEBUG - 5673 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 302, takes 0.24659156799316406
2026-07-13 18:54:41,165 - SL - DEBUG - 5673 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard/ ImmutableMultiDict([]) 200, takes 0.33506131172180176
2026-07-13 18:54:41,429 - SL - DEBUG - 5673 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/auth/login ImmutableMultiDict([]) 200, takes 0.2528386116027832
2026-07-13 18:54:41,494 - SL - DEBUG - 5673 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /api/user_info ImmutableMultiDict([]) 200, takes 0.012177705764770508
2026-07-13 18:54:41,547 - SL - DEBUG - 5673 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /api/user_info ImmutableMultiDict([]) 401, takes 0.0029587745666503906
2026-07-13 18:54:41,558 - SL - DEBUG - 5673 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /api/user_info ImmutableMultiDict([]) 401, takes 0.002178668975830078
2026-07-13 18:54:41,578 - SL - DEBUG - 5673 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /api/user_info ImmutableMultiDict([]) 200, takes 0.010585308074951172
2026-07-13 18:54:41,592 - SL - DEBUG - 5673 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /api/user_info ImmutableMultiDict([]) 403, takes 0.005698442459106445
2026-07-13 18:54:41,605 - SL - DEBUG - 5673 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /api/user_info ImmutableMultiDict([]) 401, takes 0.005246400833129883
2026-07-13 18:54:41,635 - SL - DEBUG - 5673 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.0214688777923584
2026-07-13 18:54:41,887 - SL - DEBUG - 5673 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 302, takes 0.24013495445251465
2026-07-13 18:54:42,064 - SL - DEBUG - 5673 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard/ ImmutableMultiDict([]) 200, takes 0.1648879051208496
2026-07-13 18:54:42,120 - SL - DEBUG - 5673 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard/ ImmutableMultiDict([]) 302, takes 0.003975391387939453
2026-07-13 18:54:42,306 - SL - DEBUG - 5673 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard/ ImmutableMultiDict([]) 200, takes 0.13887619972229004
2026-07-13 18:54:42,363 - SL - DEBUG - 5673 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard/ ImmutableMultiDict([]) 302, takes 0.004217624664306641
```

The supplementary `X-Forwarded-Host` run (PID 5856) produced the matching log line with
`remote_addr = 198.51.100.9` (its `X-Forwarded-For`), confirming `x_for=1` again:

```text
2026-07-13 18:56:48,236 - SL - DEBUG - 5856 - "/app/server.py:284" - after_request() -  - 198.51.100.9 GET /dashboard/ ImmutableMultiDict([]) 302, takes 0.0011358261108398438
```

#### (F) Session backends — Redis (server-side) vs signed cookie

**Redis backend** (`MEM_STORE_URI=redis://localhost`): the `slapp` cookie is just the **signed
session id** — the data is in Redis (shown in (b)). `cj_john.txt` (curl cookie jar):

```text
# Netscape HTTP Cookie File
# https://curl.se/docs/http-cookies.html
# This file was generated by libcurl! Edit at your own risk.

#HttpOnly_localhost	FALSE	/	FALSE	1784573680	slapp	83213b1a-73bf-4dc8-85d5-47896a8f8307.<REDACTED:slapp-hmac-signature>
```

**Signed-cookie backend** (`MEM_STORE_URI=""`, RUN-Q5-COOKIE, twice): login still succeeds
(`HTTP/1.0 302 FOUND`, `Location: http://localhost:7777/dashboard/`) but **no** Redis `session:*`
key is created — the `session:*` count is unchanged across the login in both runs
(`before == after == 10`; those 10 are stale keys left from RUN-Q5-REDIS and are cleaned up at the
end of the phase):

```text
run 1: before=10 after=10
run 2: before=10 after=10
```

The complete `POST /auth/login` response (`cookie_1_post_headers.txt`, RUN-Q5-COOKIE run 1) shows
the full `slapp` payload being written in the `Set-Cookie` header — note `Vary: Cookie` (present
here, absent from the Redis backend where the cookie is only an opaque id) and the leading `.` on
the value marking a zlib-compressed payload. Run 2 is identical apart from the payload bytes,
`Date`, and signature (its decoded form is shown below):

```text
HTTP/1.0 302 FOUND
Content-Type: text/html; charset=utf-8
Content-Length: 229
Location: http://localhost:7777/dashboard/
Vary: Cookie
Set-Cookie: slapp=.eJw<REDACTED:zlib+base64url-session-payload>.alU1KQ.<REDACTED:slapp-hmac-signature>; Expires=Mon, 20-Jul-2026 18:57:45 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Mon, 13 Jul 2026 18:57:45 GMT
```

Now the cookie carries the **whole session payload**. `cookie_1_jar.txt` (the leading `.` marks a
zlib-compressed payload; the three dot-separated parts are `payload.timestamp.signature`):

```text
#HttpOnly_localhost	FALSE	/	FALSE	1784573865	slapp	.eJw<REDACTED:zlib+base64url-session-payload>.alU1KQ.<REDACTED:slapp-hmac-signature>
```

Decoding the payload with **base64url + zlib and no secret key** yields readable JSON — proving the
cookie is **signed (tamper-evident), not encrypted**. Both runs decode to the same `_user_id`
(john's `alternative_id` UUID) and `_id`; the `csrf_token`/`sudo_time`/signature differ per login.
`cookie_1_decoded.txt` and `cookie_2_decoded.txt`:

```text
raw cookie value: .eJw<REDACTED:zlib+base64url-session-payload>.alU1KQ.<REDACTED:slapp-hmac-signature>
zlib_compressed: True
base64-decoded payload (no secret key used => readable => signed, NOT encrypted):
{"_fresh":true,"_id":"b03643a8a515eae10966eb5799d0b928b1fae8daeb0b0a33ba33f35b5fd7e09d574a2b38cd3af3ce53bdb464d28d2c515533674c8b087cba7d49abfa2cc9de57","_permanent":true,"_user_id":"bfb8024e-6c00-4c33-ac2c-e60bf3b7003d","csrf_token":"<REDACTED:csrf_token-value>","sudo_time":1783969065}
```
```text
raw cookie value: .eJw<REDACTED:zlib+base64url-session-payload>.alU1MQ.<REDACTED:slapp-hmac-signature>
zlib_compressed: True
base64-decoded payload (no secret key used => readable => signed, NOT encrypted):
{"_fresh":true,"_id":"b03643a8a515eae10966eb5799d0b928b1fae8daeb0b0a33ba33f35b5fd7e09d574a2b38cd3af3ce53bdb464d28d2c515533674c8b087cba7d49abfa2cc9de57","_permanent":true,"_user_id":"bfb8024e-6c00-4c33-ac2c-e60bf3b7003d","csrf_token":"<REDACTED:csrf_token-value>","sudo_time":1783969073}
```


#### (G) Web logout — session destruction & post-logout rejection

The web-session lifecycle is completed by **logout**. `GET /auth/logout` routes to `logout()`
(`app/auth/views/logout.py:L8-9`), which calls `logout_session()` (`app/session.py:L117-121` —
Flask-Login's `logout_user()` at `L118`, and, for the Redis backend, `purge_session()` at
`L119-121`), then returns a `302` to `auth.login` while **deleting** the `slapp`, `mfa`, and
`dark-mode` cookies (`app/auth/views/logout.py:L13-15`; `SESSION_COOKIE_NAME = "slapp"` at
`app/config.py:L199`). Captured on one cookie jar (`cj_logout.txt`); the session id / CSRF shown are
dynamic per run.

**(g-1) BEFORE — authenticated `GET /dashboard/`** (logged in as john@wick.com) —
`web_before_logout_dashboard.txt` (`200`, the "before" state):

```text
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 753966
Set-Cookie: slapp=eb6071a0-37ea-43d7-a806-f5767a8433d4.<REDACTED:slapp-hmac-signature>; Expires=Mon, 20-Jul-2026 23:26:39 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Mon, 13 Jul 2026 23:26:39 GMT
```

**(g-2) `GET /auth/logout`** — `web_logout.txt`. `302` to `/auth/login`; the response **deletes**
`slapp`/`mfa`/`dark-mode` (each `Expires=Thu, 01-Jan-1970 00:00:00 GMT; Max-Age=0`) and, in the same
response, opens a **fresh anonymous** `slapp` session id (Flask starts a new empty session once the
authenticated one is cleared):

```text
HTTP/1.0 302 FOUND
Content-Type: text/html; charset=utf-8
Content-Length: 229
Location: http://localhost:7777/auth/login
Set-Cookie: slapp=; Expires=Thu, 01-Jan-1970 00:00:00 GMT; Max-Age=0; Path=/
Set-Cookie: mfa=; Expires=Thu, 01-Jan-1970 00:00:00 GMT; Max-Age=0; Path=/
Set-Cookie: dark-mode=; Expires=Thu, 01-Jan-1970 00:00:00 GMT; Max-Age=0; Path=/
Set-Cookie: slapp=11e32c66-a9e5-4a0b-9fbc-d992485346de.<REDACTED:slapp-hmac-signature>; Expires=Mon, 20-Jul-2026 23:26:39 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Mon, 13 Jul 2026 23:26:39 GMT
```

Server-side session purge (Redis) — `redis_logout.txt`. The authenticated session key existed with
`ttl=604800` (7 days) **before** logout and is **gone** afterwards (`purge_session()` deleted it):

```text
--- redis BEFORE logout ---
session:eb6071a0-37ea-43d7-a806-f5767a8433d4
604800
--- redis AFTER logout ---
keys matching session:eb6071a0-37ea-43d7-a806-f5767a8433d4 => 0
```

**(g-3) AFTER — post-logout `GET /dashboard/`** on the **same** cookie jar —
`web_after_logout_dashboard.txt`. `302` to `/auth/login?next=%2Fdashboard%2F%3F`: the jar now carries
only the fresh anonymous session (no `_user_id`), so the protected page no longer authenticates:

```text
HTTP/1.0 302 FOUND
Content-Type: text/html; charset=utf-8
Content-Length: 277
Location: http://localhost:7777/auth/login?next=%2Fdashboard%2F%3F
Set-Cookie: slapp=11e32c66-a9e5-4a0b-9fbc-d992485346de.<REDACTED:slapp-hmac-signature>; Expires=Mon, 20-Jul-2026 23:26:39 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Mon, 13 Jul 2026 23:26:39 GMT
```

#### (H) API revoked key over HTTP — a concrete `/api/user_info` request

The **acquire → use → reuse** steps here are all real HTTP calls; the **revoke** step in *this*
block is a **supplemental, non-canonical** direct-SQL `DELETE` of the key's row (used only to
produce the revoked state compactly). The genuinely canonical owner-revocation transition — the
CSRF-protected, sudo-gated dashboard `POST /dashboard/api_key` `form-name=delete` — is exercised
**entirely over HTTP** in subsection **(H′)** immediately below. In both blocks the observed signal
is identical: on the revoked key, `ApiKey.get_by(code=...)` (`app/api/base.py:L18`) returns `None`,
so `authorize_request()` takes the no-key branch (`if not api_key:` `L20`); with no web session on
the request it returns `401 {"error": "Wrong api key"}` (`L27`). The key value below is a disposable
DEVELOPMENT-only key created solely for this test and deleted at the revoke step; it is dynamic per
run.

**(h-1) ACQUIRE — `POST /api/auth/login`** (device `blitzy-q5-revoke`) — `api_revoke_acquire.txt`
(`200`, returns the `api_key`):

```text
HTTP/1.0 200 OK
Content-Type: application/json
Content-Length: 178
Access-Control-Allow-Origin: *
Set-Cookie: slapp=ec5b849b-9256-4595-a1f8-19317416c382.<REDACTED:slapp-hmac-signature>; Expires=Mon, 20-Jul-2026 23:28:02 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Mon, 13 Jul 2026 23:28:02 GMT

{
  "api_key": "<REDACTED:api_key-value>", 
  "email": "john@wick.com", 
  "mfa_enabled": false, 
  "mfa_key": null, 
  "name": "John Wick"
}
```

The acquired key is persisted as an **owned** `api_key` row (`user_id=1` = john) — `revoke_db.txt`
(top block):

```text
--- api_key row for acquired key BEFORE revoke (id|name|user_id|times) ---
8|blitzy-q5-revoke|1|0
--- rows matching that code AFTER revoke (expect 0) ---
0
```

**(h-2) USE — valid key `GET /api/user_info`** — `api_revoke_use.txt` (`200`, john's user JSON):

```text
HTTP/1.0 200 OK
Content-Type: application/json
Content-Length: 279
Access-Control-Allow-Origin: *
Set-Cookie: slapp=d2e62c8a-cb83-4f55-8386-7712c1d976e7.<REDACTED:slapp-hmac-signature>; Expires=Mon, 20-Jul-2026 23:28:02 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Mon, 13 Jul 2026 23:28:02 GMT

{
  "can_create_reverse_alias": true, 
  "connected_proton_address": null, 
  "email": "john@wick.com", 
  "in_trial": false, 
  "is_premium": true, 
  "max_alias_free_plan": 3, 
  "name": "John Wick", 
  "profile_picture_url": "http://localhost/static/upload/profile_pic.svg"
}
```

**(h-3) REVOKE** — the key row is deleted; the `revoke_db.txt` bottom block above confirms `0` rows
remain matching that `code`.

**(h-4) REUSE over HTTP — the same, now-revoked key `GET /api/user_info`** — `api_revoke_after.txt`
(`401`, body `Wrong api key` from `app/api/base.py:L27` — **not** the generic 401 errorhandler that
would say `"Unauthorized"`):

```text
HTTP/1.0 401 UNAUTHORIZED
Content-Type: application/json
Content-Length: 31
Access-Control-Allow-Origin: *
Set-Cookie: slapp=64175680-b23c-4ada-abf2-2f69cbd73b67.<REDACTED:slapp-hmac-signature>; Expires=Mon, 20-Jul-2026 23:28:02 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Mon, 13 Jul 2026 23:28:02 GMT

{
  "error": "Wrong api key"
}
```

#### (H′) Canonical owner revocation over HTTP — the sudo-gated dashboard delete (RUN-Q5-REVOKE-CANONICAL)

To exercise the **canonical** revoke transition (not the SQL `DELETE` fixture used in (H)), a fresh
Redis-backed server was driven entirely through the owner UI: web login → **enter-sudo** (the delete
action is `@sudo_required`, `app/dashboard/views/enter_sudo.py:L71-81`, `_SUDO_GAP = 120` s) →
`POST /dashboard/api_key` `form-name=create` to mint a key → use it once → `POST /dashboard/api_key`
`form-name=delete` (`app/dashboard/views/api_key.py:L60-76`: `ApiKey.delete(api_key_id)` +
`Session.commit()`) → reuse the same key. Every step is a real CSRF-protected HTTP request; the only
DB touches are the read-back of the row id and the final `count(*)`.

```bash
# fresh Redis server on :7777; jar=cj_canon.txt; PY=/app/venv/bin/python; PGP="PGPASSWORD=test psql -h localhost -U test -d test"
# scrape() = grep the csrf_token hidden input value from an HTML file
curl -s -c cj -o l.html  "$BASE/auth/login";        C=$(scrape l.html)                 # anon GET
curl -s -b cj -c cj --data-urlencode email=john@wick.com --data-urlencode password=password \
     --data-urlencode "csrf_token=$C" "$BASE/auth/login"                                # 1) login  -> 302 /dashboard/
curl -s -b cj "$BASE/dashboard/"                                                        # 2) authed -> 200
curl -s -b cj -c cj -o s.html "$BASE/dashboard/enter_sudo"; C=$(scrape s.html)          # sudo GET  -> 200
curl -s -b cj -c cj --data-urlencode password=password --data-urlencode "csrf_token=$C" \
     "$BASE/dashboard/enter_sudo"                                                        # 3) sudo   -> 302
curl -s -b cj -c cj -o a.html "$BASE/dashboard/api_key"; C=$(scrape a.html)             # sudo-gated GET -> 200
curl -s -b cj -c cj -o new.html --data-urlencode form-name=create \
     --data-urlencode name=blitzy-canonical-revoke --data-urlencode "csrf_token=$C" \
     "$BASE/dashboard/api_key"                                                           # 4) create -> 200 (new key in new_api_key.html)
NK=$(scrape_clipboard new.html)                                                          #   bind new key (value never printed here)
curl -s -H "Authentication: $NK" "$BASE/api/user_info"                                   # 5) use    -> 200
KID=$($PGP -t -A -c "select id from api_key where name='blitzy-canonical-revoke';")      #   id read-back
curl -s -b cj -c cj --data-urlencode form-name=delete --data-urlencode "api-key-id=$KID" \
     --data-urlencode "csrf_token=$C" "$BASE/dashboard/api_key"                          # 6) REVOKE -> 302 (canonical HTTP delete)
curl -s -H "Authentication: $NK" "$BASE/api/user_info"                                   # 7) reuse  -> 401
$PGP -t -A -c "select count(*) from api_key where name='blitzy-canonical-revoke';"       # 8) readback -> 0
```

Complete (secret-masked) outcomes:

```text
1) POST /auth/login                    -> HTTP/1.0 302 FOUND   Location: http://localhost:7777/dashboard/
2) GET  /dashboard/                    -> HTTP/1.0 200 OK
3) POST /dashboard/enter_sudo          -> HTTP/1.0 302 FOUND   Location: http://localhost:7777/dashboard/
4) POST /dashboard/api_key (create)    -> HTTP/1.0 200 OK
   Set-Cookie: slapp=6e4f6e6c-5010-459b-8843-91403a443e9a.<REDACTED:slapp-hmac-signature>; Expires=Tue, 21-Jul-2026 04:07:34 GMT; HttpOnly; Path=/; SameSite=Lax
5) GET  /api/user_info (NEW key)       -> HTTP/1.0 200 OK
   {
     "can_create_reverse_alias": true,
     "connected_proton_address": null,
     "email": "john@wick.com",
     "in_trial": false,
     "is_premium": true,
     "max_alias_free_plan": 3,
     "name": "John Wick",
     "profile_picture_url": "http://localhost/static/upload/profile_pic.svg"
   }
6) api-key-id (DB read-back)           = 35
7) POST /dashboard/api_key (delete)    -> HTTP/1.0 302 FOUND   Location: http://localhost:7777/dashboard/api_key
8) GET  /api/user_info (REVOKED key)   -> HTTP/1.0 401 UNAUTHORIZED
   {
     "error": "Wrong api key"
   }
9) SELECT count(*) FROM api_key WHERE name='blitzy-canonical-revoke'  = 0
```

**Result (Observed):** the canonical, sudo-gated UI delete removes the row (`count(*) = 0`) and the
reused key then receives the very same `401 {"error": "Wrong api key"}` as the SQL-fixture path in
(H) — confirming revocation is a genuine HTTP/UI transition (not merely a database artefact) and
that the two paths converge on the identical observed signal.

#### (I) Session-lifecycle edge behaviours (Observed) — login rotation, signed-cookie logout, negative CSRF

Three further authenticated-request edge behaviours were exercised at runtime; each is a property of
the pinned stack and the app's own session/auth wiring, and each is reported as **Observed** DEV
behaviour. Where a behaviour would warrant a code change, that change is **out of scope** here — the
AAP mandates a read-only investigation and forbids modifying application source (AAP §0.5.2 /
§0.7.7) — so only the observation and its code grounding are given. They are recorded because Q5
asks *how an authenticated request is handled*, and these are the observed boundaries of that handling.

**(i-1) The session id is NOT rotated on login (Redis backend).** With one cookie jar, the anonymous
session id issued by `GET /auth/login` is unchanged after `POST /auth/login` succeeds; only the Redis
TTL and stored payload change. `RedisSessionStore.save_session` (`app/session.py:L82-114`) always
writes `self._get_key(session.session_id)` — the *same* id — and re-signs that same id into the
cookie; regeneration happens only in `purge_session` (`app/session.py:L61-66`), which is called by
`logout_session` (`app/session.py:L117-121`), not on login. `after_login`
(`app/auth/views/login_utils.py:L36-37`) calls `login_user(user)` and sets `session` keys but does
not rotate the id. Observed:

```text
anon session id (cookie)  = 4eca6bbb-e655-4ca1-b960-1d35b535a48b   redis TTL=300     _user_id absent
POST /auth/login          -> HTTP/1.0 302 FOUND   (1 Set-Cookie; re-signs the SAME id)
GET  /dashboard/          -> HTTP/1.0 200 OK
post-login session id     = 4eca6bbb-e655-4ca1-b960-1d35b535a48b   redis TTL=604800  _user_id present
=> session id UNCHANGED across login; TTL 300 -> 604800; payload gains _user_id.
```

**(i-2) Signed-cookie logout cannot revoke a captured cookie (fallback backend, `MEM_STORE_URI=""`).**
In signed-cookie mode the whole session lives inside the cookie, so logout only deletes the client's
copy (`response.delete_cookie(...)`, `app/auth/views/logout.py:L13-15`) — there is no server-side
record to invalidate. `logout()` (`app/auth/views/logout.py:L9-15`) calls `logout_session()`; with
the default Flask `SecureCookieSessionInterface` there is no `purge_session`, so nothing server-side
is revoked (contrast the Redis backend of (G), where `purge_session` deletes `session:<id>` and the
post-logout probe is `302`). A copy taken *before* logout keeps working for its full 7-day lifetime.
Observed (Client B replays Client A's pre-logout cookie *after* A logged out):

```text
Client A: login 302 -> /dashboard/ 200 -> logout 302 (Set-Cookie slapp Max-Age=0) -> /dashboard/ 302 (A rejected)
Client B (stale pre-logout cookie), AFTER A's logout:
  GET /dashboard/    replay #1/#2/#3 -> 200 / 200 / 200
  GET /api/user_info replay #1/#2/#3 -> 200 / 200 / 200   (session fallback: g.user = current_user)
control: one-byte-tampered cookie -> GET /dashboard/ 302 (rejected — the signature check holds)
```

**(i-3) Negative CSRF on the login form is rejected (no authentication).** Posting valid credentials
with a **missing** or **invalid** `csrf_token` does not authenticate: Flask-WTF's
`form.validate_on_submit()` (`app/auth/views/login.py:L40`) returns `False`, so the view falls
through to re-render the login page (`200`) and establishes no session. Observed:

```text
POST /auth/login (no csrf_token)      -> HTTP/1.0 200 OK   then GET /dashboard/ -> 302 (not authenticated)
POST /auth/login (invalid csrf_token) -> HTTP/1.0 200 OK   then GET /dashboard/ -> 302 (not authenticated)
```

> **Scope note.** (i-1) and (i-2) are session-fixation- and logout-revocation-relevant and would, in
> a hardening pass, motivate rotating the session id on login and adding a server-side session
> version; both are **source changes and therefore out of scope** for this read-only investigation.
> (i-3) confirms the CSRF protection on the primary web-auth form is effective.

### File:line grounding

- **Entry / middleware:** `server.py:L142` `app.wsgi_app = ProxyFix(app.wsgi_app, x_for=1, x_host=1)`
  (import at `L28`).
- **App-wide `before_request` hooks (observed order):**
  `[0]` Flask-Limiter `Limiter.__check_request_limit`; `[1]` `set_index_page.<locals>.before_request`
  defined at `server.py:L258` (inside `set_index_page(app)` `L249`, sets `g.start_time`);
  `[2]` `make_session_permanent` at `server.py:L205`.
- **Rate-limit keying / disable:** `app/extensions.py:L14-19` `__key_func` (userid when
  `current_user.is_authenticated`, else IP), `L23` `limiter = Limiter(key_func=__key_func)`,
  `L26-28` `disable_rate_limit` returns `config.DISABLE_RATE_LIMIT` (set to `1` here).
- **Web login manager:** `app/extensions.py:L7` `login_manager = LoginManager()`, `L8`
  `login_manager.session_protection = "strong"`.
- **Web login view → `login_user`:** `app/auth/views/login.py:L21-25` route + `L40`
  `form.validate_on_submit()`; `app/auth/views/login_utils.py:L12` `after_login(...)`, `L36`
  `login_user(user)` (stores `get_id()` in `session["_user_id"]`).
- **Identity token:** `app/models.py:L595-599` `get_id(self)` returns `self.alternative_id` when set
  else `str(self.id)`; `alternative_id` column at `app/models.py:L482`.
- **Web per-request identity:** `server.py:L221` `def load_user(alternative_id):` →
  `User.get_by(alternative_id=alternative_id)` (`L222`); when a user is found (`if user:` `L223`) it
  **sets the Sentry user identity** `sentry_sdk.set_user({"email": user.email, "id": user.id})`
  (`L224`), then returns `None` if `user.disabled` (`L225-226`) or `not user.is_active()`
  (`L227-228`); `is_active` at `app/models.py:L766` (`True` iff `delete_on is None` or in the past).
- **API auth:** `app/api/base.py:L16` `def authorize_request()`; `L17`
  `api_code = request.headers.get("Authentication")`; `L18` `api_key = ApiKey.get_by(code=api_code)`;
  no-key branch `L20-27` (`if current_user.is_authenticated:` `L21` → `g.user = current_user` `L25`,
  else `return jsonify(error="Wrong api key"), 401` `L27`); valid-key stats `L30-32`
  (`last_used`, `times += 1`, `Session.commit()`); guards `L36-37` (`disabled → 403`),
  `L39-40` (`not is_active() → 401`); `L34` `g.user = api_key.user`; `L42` `g.api_key = api_key`.
- **Propagation helper (correction):** `server.py:L332-336` `get_current_user()` returns `g.user`
  else `current_user`; its **only** runtime consumer is the 429 handler `server.py:L367`
  (`LOG.w(..., get_current_user())`). Normal web views use `current_user`; `/api/user_info`
  (`app/api/views/user_info.py:L50-51`, `@require_api_auth`) reads `g.user`.
- **Per-request log + teardown:** `server.py:L273` `def after_request(res):`; the `LOG.d(` call opens
  at `L284` (hence the emitted `"/app/server.py:284"`); skip-list `L276-281`; the registered `teardown_appcontext`
  callback runs `Session.remove()` (`server.py:L209-211`) — **Inferred** (framework-guaranteed: the
  callback registration is observed, but its per-request invocation is not separately logged).
- **Session backend selection:** `server.py:L163` `if MEM_STORE_URI:` → `L165`
  `initialize_redis_services(app, MEM_STORE_URI)` (`app/redis_services.py:L9`, which sets
  `app.session_interface = RedisSessionStore(...)`).
- **`RedisSessionStore`:** `app/session.py:L18` `SESSION_PREFIX = "session"`; `L31` class;
  `L38` `_get_signer` (`itsdangerous.Signer(..., salt="session", key_derivation="hmac")`);
  `L45` key = `session:<id>`; `L82` `save_session`; `L92`
  `ttl = int(app.permanent_session_lifetime.total_seconds())`; `L95-96`
  `if "_user_id" not in session: ttl = 300`; `L97` `setex`; `L102` `sign(...)` (cookie = signed id).
- **401/403 errorhandlers (distinct from the API bodies above):** `server.py:L347-353`
  `unauthorized(e)` — the `/api/*` branch returns `jsonify(error="Unauthorized"), 401` (`L350`),
  while the web branch returns `redirect(url_for("auth.login", next=request.full_path))` (**`L353`**,
  the exact redirect the review flagged); `server.py:L355-360` `forbidden(e)` — `/api/*` →
  `jsonify(error="Forbidden"), 403` (`L358`), web → `render_template("error/403.html"), 403` (`L360`).
- **Web logout (subsection G):** `app/auth/views/logout.py:L8` `@auth_bp.route("/logout")`, `L9`
  `def logout():`; `L10` `logout_session()` (`app/session.py:L117-121` — `logout_user()` `L118`,
  and for the Redis backend `purge_session()` `L119-121`); `L12`
  `make_response(redirect(url_for("auth.login")))`; `L13-15` `response.delete_cookie(...)` for
  `SESSION_COOKIE_NAME` (=`"slapp"`, `app/config.py:L199`), `"mfa"`, and `"dark-mode"`; `L17`
  `return response`.
- **API key revocation over HTTP (subsection H):** once the `api_key` row is deleted, the
  `authorize_request()` lookup `ApiKey.get_by(code=api_code)` (`app/api/base.py:L18`) returns `None`,
  so the no-key branch `if not api_key:` (`L20`) is taken and — absent a web session — it returns
  `jsonify(error="Wrong api key"), 401` (`L27`), i.e. the revoked key follows the identical code path
  as any wrong key.

### Observed vs Inferred

- **Observed:** the three-hook `before_request` order; ProxyFix honoring both `X-Forwarded-For`
  (`remote_addr` 203.0.113.7 / 198.51.100.9 in the `SL` log) and `X-Forwarded-Host` (redirect
  `Location` host `proxied.example.test`); the full web login flow (anon `ttl=300` → authenticated
  `ttl=604800`, `_user_id` = john's `alternative_id` UUID, DB-confirmed); the API flow — canonical
  key acquisition, valid (`200` + stats `times 0→1`, `last_used` set), wrong (`401`), absent
  (`401`), session fallback (`200`); both API guards (`403 Disabled account`, `401 Account does not
  exist`); both web `load_user` guards (disabled → `302`, inactive → `302`, restore → `200`); the 17
  correlated `SL` request-log lines (PID 5673) with `/health`, `/static`, `/git`, `/favicon.ico`
  excluded; and both session backends (Redis server-side vs signed-cookie, the latter creating no
  Redis key and decoding — without any secret — to the same `_user_id`); the **web logout**
  lifecycle (subsection G) — authenticated `/dashboard/` `200` (before) → `GET /auth/logout` `302`
  with `slapp`/`mfa`/`dark-mode` deleted (`Max-Age=0`) and the server-side Redis session purged
  (`ttl=604800` → key gone) → post-logout `/dashboard/` `302` to `/auth/login` (after); and the
  **API-key revocation over HTTP** (subsection H) — canonical acquire (`200`) → use (`200`) → the
  `api_key` row deleted → the same key reused over HTTP yields `401 {"error": "Wrong api key"}` on
  the concrete endpoint `/api/user_info`.
- **Inferred:** the generic 401/403 **errorhandler** bodies `{"error": "Unauthorized"}` /
  `{"error": "Forbidden"}` (`server.py:L350,L358`) were **not** triggered here — the API returns its
  own `authorize_request` JSON directly, so those handler bodies are read from source, not observed.
  The New Relic custom-event recording in `after_request` (`server.py`) is present in source but not
  independently verified (no New Relic backend in dev). The `teardown_appcontext` callback's
  `Session.remove()` (`server.py:L209-211`) is likewise **Inferred at the per-request level**: the
  callback's registration is observed (`app.teardown_appcontext_funcs` contains
  `create_app.<locals>.cleanup`) and Flask guarantees teardown callbacks fire, but the callback body
  emits no log/counter, so its per-request invocation is not separately observed.


## Q6 — Does the webapp auto-start any background jobs or schedulers?

### Direct answer (Observed)

**No.** Starting the webapp the canonical way (`python3 server.py`) starts **no** scheduler, cron,
or job-runner thread. Observed at runtime: after `create_app()`, **none** of the background-job
modules (`cron`, `job_runner`, `event_listener`, `email_handler`) are present in `sys.modules`, and
`server.py` contains **no import** of any of them. The running dev server has exactly two OS threads
in its worker (child) process and one in the reloader (parent) process, and **both worker threads
belong to the Werkzeug dev-server/reloader machinery, not to any application scheduler**:

- **Parent (reloader) process** — `NLWP=1`: blocked in `StatReloaderLoop.restart_with_reloader()`
  waiting on the subprocess.
- **Child (worker) process** — `NLWP=2`: per `werkzeug/_reloader.py:run_with_reloader` (L325), when
  `WERKZEUG_RUN_MAIN == "true"` (L332) the actual WSGI server (`main_func`) is started in a **daemon
  thread** (`threading.Thread(target=main_func)`, L334; `setDaemon(True)`, L335) while
  `reloader.run()` — the file-watching stat loop — runs in the **main thread** (L337). Neither is an
  APScheduler/cron/job thread.

The scheduled work lives in **separate processes with their own `if __name__ == "__main__"` guards**,
run independently (and, in production, driven by an external `yacron` reading `crontab.yml` — see
below): `cron.py`, `job_runner.py`, `email_handler.py`, and `event_listener.py`. The first three
build a lightweight app context via `create_light_app()`; `event_listener.py` does **not** use
`create_light_app` at all (it dispatches `LISTENER`/`DEAD_LETTER`/`debug`/`run` subcommands).

### Command(s) run

```bash
cd /app && set -a && . /tmp/sl_env.sh && set +a && unset EVENT_WEBHOOK_DISABLE \
  && export GNUPGHOME=/tmp/sl_clean_gnupg

# (1) bounded lifecycle, run TWICE independently (RUN-Q6-A, RUN-Q6-B) to confirm cross-run stability
#     and that NO scheduler thread/process appears over a >=60s window. Each run inspects the thread
#     topology at t=0s (before), t=30s (intermediate) and t=65s (after the >=60s window), then
#     terminates by exact PID.
run_once() {                                            # $1 = RUN-Q6-A | RUN-Q6-B
  /app/venv/bin/python server.py > /tmp/q6_$1.log 2>&1 &
  PARENT=$!; sleep 7
  CHILD=$(ps -eo pid,ppid,args | awk -v p="$PARENT" '$2==p && /server.py/{print $1}')
  echo "$1 parent=$PARENT child=$CHILD"
  inspect() {                                           # one thread-topology sample
    curl -s -o /dev/null -w "health_$2=%{http_code}\n" http://localhost:7777/health
    echo "=== $1 inspection $3 ==="
    ps -o pid,ppid,nlwp,args -p "$PARENT","$CHILD"       # NLWP = OS thread count
    echo "-- parent thread ids (/proc/$PARENT/task) --"
    for t in /proc/$PARENT/task/*; do echo "  tid=$(basename "$t") comm=$(cat "$t/comm")"; done
    echo "-- child thread ids (/proc/$CHILD/task) --"
    for t in /proc/$CHILD/task/*;  do echo "  tid=$(basename "$t") comm=$(cat "$t/comm")"; done; }
  inspect "$1" t0  "t=0s (before)"
  sleep 30; inspect "$1" t30 "t=30s (intermediate)"
  sleep 35; inspect "$1" t65 "t=65s (after >=60s window)"   # 7+30+35 >= 60s elapsed since child came up
  ps -eo pid,ppid,nlwp,args | grep "[p]ython"           # every python process (live server + zombies)
  kill -TERM $CHILD $PARENT; sleep 1; kill -KILL $CHILD $PARENT; sleep 1
  curl -s -o /dev/null -w "health_after=%{http_code}\n" --max-time 3 http://localhost:7777/health \
    || echo "PORT FREE ($1)"
}
run_once RUN-Q6-A ; sleep 3 ; run_once RUN-Q6-B
# process-population summary: histogram of python process states (S=live sleeping, Z=zombie/defunct)
ps -eo stat,comm | awk '/python/{c[substr($1,1,1)]++} END{for(k in c) print k"  count="c[k]}'

# (2) runtime import check — are any job modules pulled in by the webapp?
/app/venv/bin/python - <<'PY'
import sys
from server import create_app
app = create_app()
mods = [m for m in ("cron","job_runner","event_listener","email_handler") if m in sys.modules]
print("background-job modules present in sys.modules after create_app():", mods if mods else "NONE")
PY

# (3) does server.py import any job module? per-script __main__ guard + create_light_app usage:
grep -nE '^import (cron|job_runner|event_listener|email_handler)|^from (cron|job_runner|event_listener|email_handler)' server.py
for f in cron.py job_runner.py email_handler.py event_listener.py; do
  grep -n '__main__' "$f"; grep -n 'create_light_app' "$f"; done

# (4) external scheduler config (read by yacron in production, not by the webapp):
cat crontab.yml
```

### Complete, unedited output

**(1) Lifecycle over a `>=60 s` window — two independent runs (RUN-Q6-A, RUN-Q6-B).** Each run
samples the thread topology at **t=0 s (before)**, **t=30 s (intermediate)** and **t=65 s (after the
`>=60 s` window)**. RUN-Q6-A output:

```text
########## RUN-Q6-A parent=37063 child=37075 ##########
health_t0=200
=== RUN-Q6-A inspection t=0s (before) ===
    PID    PPID NLWP COMMAND
  37063   37042    1 /app/venv/bin/python server.py
  37075   37063    2 /app/venv/bin/python /app/server.py
-- parent thread ids (/proc/37063/task) --
  tid=37063 comm=python
-- child thread ids (/proc/37075/task) --
  tid=37075 comm=python
  tid=37086 comm=python
health_t30=200
=== RUN-Q6-A inspection t=30s (intermediate) ===
    PID    PPID NLWP COMMAND
  37063   37042    1 /app/venv/bin/python server.py
  37075   37063    2 /app/venv/bin/python /app/server.py
-- parent thread ids (/proc/37063/task) --
  tid=37063 comm=python
-- child thread ids (/proc/37075/task) --
  tid=37075 comm=python
  tid=37086 comm=python
health_t65=200
=== RUN-Q6-A inspection t=65s (after >=60s window) ===
    PID    PPID NLWP COMMAND
  37063   37042    1 /app/venv/bin/python server.py
  37075   37063    2 /app/venv/bin/python /app/server.py
-- parent thread ids (/proc/37063/task) --
  tid=37063 comm=python
-- child thread ids (/proc/37075/task) --
  tid=37075 comm=python
  tid=37086 comm=python
health_after=000
PORT FREE (RUN-Q6-A)
```

RUN-Q6-B output (fresh process, unchanged invocation):

```text
########## RUN-Q6-B parent=37137 child=37150 ##########
health_t0=200
=== RUN-Q6-B inspection t=0s (before) ===
    PID    PPID NLWP COMMAND
  37137   37042    1 /app/venv/bin/python server.py
  37150   37137    2 /app/venv/bin/python /app/server.py
-- parent thread ids (/proc/37137/task) --
  tid=37137 comm=python
-- child thread ids (/proc/37150/task) --
  tid=37150 comm=python
  tid=37161 comm=python
health_t30=200
=== RUN-Q6-B inspection t=30s (intermediate) ===
    PID    PPID NLWP COMMAND
  37137   37042    1 /app/venv/bin/python server.py
  37150   37137    2 /app/venv/bin/python /app/server.py
-- parent thread ids (/proc/37137/task) --
  tid=37137 comm=python
-- child thread ids (/proc/37150/task) --
  tid=37150 comm=python
  tid=37161 comm=python
health_t65=200
=== RUN-Q6-B inspection t=65s (after >=60s window) ===
    PID    PPID NLWP COMMAND
  37137   37042    1 /app/venv/bin/python server.py
  37150   37137    2 /app/venv/bin/python /app/server.py
-- parent thread ids (/proc/37137/task) --
  tid=37137 comm=python
-- child thread ids (/proc/37150/task) --
  tid=37150 comm=python
  tid=37161 comm=python
health_after=000
PORT FREE (RUN-Q6-B)
```

In **both** runs the topology is **flat across the whole `>=60 s` window and identical between runs**:
parent `NLWP=1` (the reloader), child `NLWP=2` (daemon WSGI thread + `reloader.run()` main thread),
the very same two child tids at every sample, `/health = 200` throughout, and after killing the exact
PIDs the `:7777` listener is gone (`health_after=000` ⇒ `PORT FREE`). **No third thread and no extra
process ever appears** — i.e. no APScheduler/cron/job thread is spawned over time.

The per-run `ps … | grep "[p]ython"` listing additionally showed **only** each run's own parent+child
as *live* processes; the other python entries it printed were **36 `<defunct>` (zombie, state `Z`)
processes** — dead remnants of earlier short-lived dev-server launches in this long-lived (~12 h)
container, reparented to PID 1 and awaiting reap — **not** live schedulers. The steady-state
process-state histogram makes the live-vs-zombie split explicit (`S` = live sleeping, `Z` = zombie):

```text
Z  count=36
S  count=2
```

i.e. exactly **two** live python processes exist (the reloader parent + worker child, confirmed
`Ss` and `Sl` — the `l` flag = multithreaded child); every other python entry is a dead zombie, so
no background scheduler process is running.

**(2) Runtime import check — `import_check.txt`:**

```text
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-13 19:12:10,177 - SL - DEBUG - 6812 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
background-job modules present in sys.modules after create_app(): NONE
```


**(3) `server.py` imports + per-script `__main__`/`create_light_app` — `scripts_arch.txt`:**

```text
########## server.py: does it import the 4 job modules? ##########
(no imports of cron/job_runner/event_listener/email_handler in server.py)

########## cron.py : __main__ guard ##########
1262:if __name__ == "__main__":
----- cron.py : create_light_app usage -----
65:from server import create_light_app
1273:    with create_light_app().app_context():

########## job_runner.py : __main__ guard ##########
329:if __name__ == "__main__":
----- job_runner.py : create_light_app usage -----
24:from server import create_light_app
332:        with create_light_app().app_context():

########## email_handler.py : __main__ guard ##########
2396:if __name__ == "__main__":
----- email_handler.py : create_light_app usage -----
177:from server import create_light_app
2352:        with create_light_app().app_context():

########## event_listener.py : __main__ guard ##########
94:if __name__ == "__main__":
----- event_listener.py : create_light_app usage -----
  (does NOT use create_light_app)
```

`event_listener.py`'s `__main__` block (it parses a subcommand rather than building an app context)
— `eventlistener_cron.txt`:

```text
if __name__ == "__main__":
    if len(argv) < 2:
        print("Invalid usage. Pass a valid subcommand as argument")
        exit(1)

    args = args()

    if args.command in [Mode.LISTENER.value, Mode.DEAD_LETTER.value]:
        main(
            mode=Mode.from_str(args.command),
            dry_run=args.dry_run,
            max_retries=args.max_retries,
        )
    elif args.command == "debug":
        debug_event(args.event_id)
    elif args.command == "run":
        run_event(args.event_id, args.delete_on_success)
    else:
        print("Invalid command")
        exit(1)
```

**(4) External scheduler config — `crontab.yml`** (each job shells out to `python /code/cron.py -j
<job>` on a cron schedule; production runs this under `yacron`, entirely outside the webapp process):

```yaml
jobs:
  - name: SimpleLogin growth stats
    command: python /code/cron.py -j stats
    shell: /bin/bash
    schedule: "0 0 * * *"
    captureStderr: true

  - name: SimpleLogin Delete Old Monitoring records
    command: python /code/cron.py -j delete_old_monitoring
    shell: /bin/bash
    schedule: "15 1 * * *"
    captureStderr: true

  - name: SimpleLogin Custom Domain check
    command: python /code/cron.py -j check_custom_domain
    shell: /bin/bash
    schedule: "15 2 * * *"
    captureStderr: true

  - name: SimpleLogin HIBP check
    command: python /code/cron.py -j check_hibp
    shell: /bin/bash
    schedule: "15 3 * * *"
    captureStderr: true
    concurrencyPolicy: Forbid

  - name: SimpleLogin Notify HIBP breaches
    command: python /code/cron.py -j notify_hibp
    shell: /bin/bash
    schedule: "15 4 * * *"
    captureStderr: true
    concurrencyPolicy: Forbid

  - name: SimpleLogin Delete Logs
    command: python /code/cron.py -j delete_logs
    shell: /bin/bash
    schedule: "15 5 * * *"
    captureStderr: true

  - name: SimpleLogin Delete Old data
    command: python /code/cron.py -j delete_old_data
    shell: /bin/bash
    schedule: "30 5 * * *"
    captureStderr: true

  - name: SimpleLogin Poll Apple Subscriptions
    command: python /code/cron.py -j poll_apple_subscription
    shell: /bin/bash
    schedule: "15 6 * * *"
    captureStderr: true

  - name: SimpleLogin Notify Trial Ends
    command: python /code/cron.py -j notify_trial_end
    shell: /bin/bash
    schedule: "15 8 * * *"
    captureStderr: true

  - name: SimpleLogin Notify Manual Subscription Ends
    command: python /code/cron.py -j notify_manual_subscription_end
    shell: /bin/bash
    schedule: "15 9 * * *"
    captureStderr: true

  - name: SimpleLogin Notify Premium Ends
    command: python /code/cron.py -j notify_premium_end
    shell: /bin/bash
    schedule: "15 10 * * *"
    captureStderr: true

  - name: SimpleLogin delete users scheduled to be deleted
    command: python /code/cron.py -j delete_scheduled_users
    shell: /bin/bash
    schedule: "15 11 * * *"
    captureStderr: true
    concurrencyPolicy: Forbid

  - name: SimpleLogin send unsent emails
    command: python /code/cron.py -j send_undelivered_mails
    shell: /bin/bash
    schedule: "*/5 * * * *"
    captureStderr: true
    concurrencyPolicy: Forbid

  - name: SimpleLogin clear alias_audit_log old entries
    command: python /code/cron.py -j clear_alias_audit_log
    shell: /bin/bash
    schedule: "0 * * * *" # Once every hour
    captureStderr: true
    concurrencyPolicy: Forbid

  - name: SimpleLogin clear user_audit_log old entries
    command: python /code/cron.py -j clear_user_audit_log
    shell: /bin/bash
    schedule: "0 * * * *" # Once every hour
    captureStderr: true
    concurrencyPolicy: Forbid
```

### File:line grounding

- **No import in the webapp:** `server.py` has no `import cron|job_runner|event_listener|email_handler`
  (grep above); confirmed at runtime by `sys.modules` after `create_app()` = `NONE`.
- **Thread model (library source):** `werkzeug/_reloader.py:L325` `run_with_reloader(main_func, …)`;
  `L332` `if os.environ.get("WERKZEUG_RUN_MAIN") == "true":`; `L334`
  `t = threading.Thread(target=main_func, args=())`; `L335` `t.setDaemon(True)`; `L337`
  `reloader.run()`; parent branch `L339` `sys.exit(reloader.restart_with_reloader())`;
  `WERKZEUG_RUN_MAIN` is set for the child at `werkzeug/_reloader.py:L182`.
- **Background scripts are independent processes:**
  `cron.py:L1262` `if __name__ == "__main__":`, `create_light_app` import `L65`, context `L1273`;
  `job_runner.py:L329` guard, import `L24`, context `L332`;
  `email_handler.py:L2396` guard, import `L177`, context `L2352`;
  `event_listener.py:L94` guard — **no** `create_light_app`; dispatches
  `Mode.LISTENER`/`Mode.DEAD_LETTER`/`debug`/`run`.
- **External scheduling:** `crontab.yml` — 15 jobs, each `command: python /code/cron.py -j <job>` on
  a cron `schedule`, run by `yacron` (a separate process), not by the Flask app.

### Observed vs Inferred

- **Observed:** the webapp starting with no job module in `sys.modules`; the stable
  parent(`NLWP=1`)/child(`NLWP=2`) thread counts sampled at **t=0 s, t=30 s and t=65 s** and found
  **identical across a `>=60 s` window in two independent runs** (RUN-Q6-A, RUN-Q6-B); the
  process-state histogram showing exactly **two** live python processes (`S count=2`) and no live
  scheduler (the other 36 being `Z`/zombie remnants); `/health = 200` throughout each window; the
  port freeing after termination; `server.py` importing none of the four scripts; and each script's
  `__main__` guard and `create_light_app` usage (or absence, for `event_listener.py`).
- **Inferred:** the *identity* of the two child threads (daemon server thread + `reloader.run()` main
  thread) is read from Werkzeug 1.0.1 source (`_reloader.py`), not from thread names (which all read
  `comm=python`); the production use of `yacron` to execute `crontab.yml` is read from the config
  file, not run here (no `yacron` in this dev container).


## Q7 — Anything else going on? (import-time side effects)

### Direct answer (Observed)

Beyond request handling, starting the app has four notable **import-time** side effects:

1. **A PostgreSQL connection is opened at import.** `app/db.py:L12` runs `connection =
   engine.connect()` at module import (not lazily), so merely importing the app opens a real DB
   connection with `application_name = "webapp"` (`config.DB_CONN_NAME`, `app/config.py:L193`). This
   is a **single process-wide connection**: `app/db.py:L14` binds `Session =
   scoped_session(sessionmaker(bind=connection))` to that one `connection` object (not to the engine
   pool), so **all threads share it** — which, under the threaded dev server, lets concurrent requests
   race on one transaction (demonstrated in *Further observed DEV-mode behaviours* (a) below).
2. **That connection happens *twice* under the dev reloader.** Because `debug=True` enables the
   Werkzeug reloader, the app module is imported in **both** the reloader (parent) process **and** the
   worker (child) process — so `pg_stat_activity` shows **two** `webapp` connections while the dev
   server is up, and the import-time `print` banners appear **twice** in the startup stream (with the
   Flask/Werkzeug `* Serving … * Environment: production … * Debug mode: on` banner in between).
3. **`OAUTHLIB_INSECURE_TRANSPORT` is force-set to `"1"` at import.** `server.py:L124`
   (`os.environ["OAUTHLIB_INSECURE_TRANSPORT"] = "1"`) is an **unconditional module-level**
   statement — it runs on *any* import of `server`, including the production `wsgi.py` import, not
   just in dev. Observed: the variable is `None` before importing `server` and `'1'` after. Its
   purpose (comment `server.py:L123`) is that the app is served behind nginx over http, i.e. it is a
   trusted-proxy framing — not a claim that public traffic is plaintext.
4. **Import fails fast if the DB is unreachable.** Because the connect is at import, pointing
   `DB_URI` at a dead port makes `python server.py` abort during import with a full
   `sqlalchemy.exc.OperationalError` traceback and exit code `1` — the server never binds a port.

Three **further DEV-mode behaviours** — request concurrency racing on the single shared connection,
the Flask Debug Toolbar (and the forgeability of the signed session given the public `FLASK_SECRET`),
and the non-idempotency of `flask dummy-data` — are exercised and reported in *Further observed
DEV-mode behaviours* at the end of this section.

### Command(s) run

```bash
cd /app && set -a && . /tmp/sl_env.sh && set +a && unset EVENT_WEBHOOK_DISABLE \
  && export GNUPGHOME=/tmp/sl_clean_gnupg
PSQL="psql -h localhost -U test -d test -A -F| -P pager=off"

# (1) canonical: launch the real server, count webapp DB connections before / during / after:
PGPASSWORD=test $PSQL -c "select count(*) as webapp_conns_before from pg_stat_activity where application_name='webapp';"
/app/venv/bin/python server.py > q7_server.log 2>&1 &   # reloader => parent + child both import app.db
PARENT=$!; sleep 6
CHILD=$(ps -eo pid,ppid,args | awk -v p="$PARENT" '$2==p && /server.py/{print $1}')   # RUN-Q7 parent=7067 child=7079
PGPASSWORD=test $PSQL -c "select pid, application_name, state, backend_type, client_addr, client_port from pg_stat_activity where application_name='webapp' order by pid;"
kill -TERM $CHILD $PARENT 2>/dev/null; sleep 1; kill -KILL $CHILD $PARENT 2>/dev/null; sleep 2
PGPASSWORD=test $PSQL -c "select count(*) as webapp_conns_after from pg_stat_activity where application_name='webapp';"

# (2) NON-CANONICAL helper: prove the connect happens at IMPORT of app.db:
/app/venv/bin/python -c "import app.db; print('app.db imported; connection =', app.db.connection); print('closed?', app.db.connection.closed)"

# (3) OAUTHLIB flag flips None -> '1' by importing server (server.py:L124):
/app/venv/bin/python -c "import os; print('before import server:', repr(os.environ.get('OAUTHLIB_INSECURE_TRANSPORT'))); import server; print('after  import server:', repr(os.environ.get('OAUTHLIB_INSECURE_TRANSPORT')))"

# (4) fail-fast: canonical entry point with DB_URI on a dead port (15432 = nothing listening):
export DB_URI="postgresql://test:test@localhost:15432/test"
/app/venv/bin/python server.py    # aborts at import with full traceback, exit 1
```

### Complete, unedited output

**(1) `webapp` DB connections before / during / after — `pg_before.txt`, `pids.txt`,
`pg_during.txt`, `pg_after.txt`** (RUN-Q7). Two connections while up (one per process — the PG `pid`
column is the server-side backend pid, and the two `client_port`s distinguish the reloader parent
and worker child); zero before and after:

```text
webapp_conns_before
0
(1 row)
```
```text
RUN-Q7 parent=7067 child=7079
```
```text
pid|application_name|state|backend_type|client_addr|client_port
7069|webapp|idle|client backend|::1|51390
7080|webapp|idle|client backend|::1|51400
(2 rows)
```
```text
webapp_conns_after
0
(1 row)
```

**(2) Reloader double-import — `q7_server.log`.** The import-time markers (`>>> URL: …`,
`Upload files to local dir`, `>>> init logging <<<`, the `SL` "load words file" line) print **once
for the parent (PID 7067)**, then the Flask/Werkzeug banner, then **again for the child (PID 7079)**:

```text
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-13 19:17:05,546 - SL - DEBUG - 7067 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-13 19:17:07,082 - SL - DEBUG - 7079 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

**(3) Connect-at-import (NON-CANONICAL helper) — `import_probe.txt`.** This is *not* the canonical
entry point; it is a bare `python -c "import app.db"` used only to isolate the import-time connect.
The connection object exists and is open (`closed? False`) purely from importing the module:

```text
>>> URL: http://localhost
Upload files to local dir
app.db imported; connection = <sqlalchemy.engine.base.Connection object at 0x7954ddb8a5f0>
closed? False
```

**(4) `OAUTHLIB_INSECURE_TRANSPORT` flip — `oauthlib.txt`** (`None` before importing `server`, `'1'`
after):

```text
before import server: None
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-13 19:17:14,831 - SL - DEBUG - 7103 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
after  import server: '1'
```


**(5) Fail-fast on unreachable DB — `failfast.txt`** (canonical `python server.py`, `DB_URI` pointed
at dead port 15432). The complete, unedited traceback; the direct cause chain ends at
`app/db.py:L12 connection = engine.connect()`, and the process exits `1` without ever binding a port:

```text
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
Traceback (most recent call last):
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 2336, in _wrap_pool_connect
    return fn()
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/pool/base.py", line 304, in unique_connection
    return _ConnectionFairy._checkout(self)
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/pool/base.py", line 778, in _checkout
    fairy = _ConnectionRecord.checkout(pool)
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/pool/base.py", line 495, in checkout
    rec = pool._do_get()
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/pool/impl.py", line 139, in _do_get
    with util.safe_reraise():
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/util/langhelpers.py", line 68, in __exit__
    compat.raise_(
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/util/compat.py", line 182, in raise_
    raise exception
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/pool/impl.py", line 137, in _do_get
    return self._create_connection()
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/pool/base.py", line 309, in _create_connection
    return _ConnectionRecord(self)
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/pool/base.py", line 440, in __init__
    self.__connect(first_connect_check=True)
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/pool/base.py", line 660, in __connect
    with util.safe_reraise():
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/util/langhelpers.py", line 68, in __exit__
    compat.raise_(
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/util/compat.py", line 182, in raise_
    raise exception
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/pool/base.py", line 656, in __connect
    connection = pool._invoke_creator(self)
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/strategies.py", line 114, in connect
    return dialect.connect(*cargs, **cparams)
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/default.py", line 508, in connect
    return self.dbapi.connect(*cargs, **cparams)
  File "/app/venv/lib/python3.10/site-packages/psycopg2/__init__.py", line 122, in connect
    conn = _connect(dsn, connection_factory=connection_factory, **kwasync)
psycopg2.OperationalError: connection to server at "localhost" (::1), port 15432 failed: Connection refused
	Is the server running on that host and accepting TCP/IP connections?
connection to server at "localhost" (127.0.0.1), port 15432 failed: Connection refused
	Is the server running on that host and accepting TCP/IP connections?


The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "/app/server.py", line 31, in <module>
    from app.admin_model import (
  File "/app/app/admin_model.py", line 11, in <module>
    from app import models, s3
  File "/app/app/models.py", line 32, in <module>
    from app.db import Session
  File "/app/app/db.py", line 12, in <module>
    connection = engine.connect()
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 2263, in connect
    return self._connection_cls(self, **kwargs)
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 104, in __init__
    else engine.raw_connection()
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 2369, in raw_connection
    return self._wrap_pool_connect(
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 2339, in _wrap_pool_connect
    Connection._handle_dbapi_exception_noconnection(
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1583, in _handle_dbapi_exception_noconnection
    util.raise_(
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/util/compat.py", line 182, in raise_
    raise exception
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 2336, in _wrap_pool_connect
    return fn()
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/pool/base.py", line 304, in unique_connection
    return _ConnectionFairy._checkout(self)
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/pool/base.py", line 778, in _checkout
    fairy = _ConnectionRecord.checkout(pool)
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/pool/base.py", line 495, in checkout
    rec = pool._do_get()
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/pool/impl.py", line 139, in _do_get
    with util.safe_reraise():
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/util/langhelpers.py", line 68, in __exit__
    compat.raise_(
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/util/compat.py", line 182, in raise_
    raise exception
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/pool/impl.py", line 137, in _do_get
    return self._create_connection()
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/pool/base.py", line 309, in _create_connection
    return _ConnectionRecord(self)
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/pool/base.py", line 440, in __init__
    self.__connect(first_connect_check=True)
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/pool/base.py", line 660, in __connect
    with util.safe_reraise():
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/util/langhelpers.py", line 68, in __exit__
    compat.raise_(
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/util/compat.py", line 182, in raise_
    raise exception
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/pool/base.py", line 656, in __connect
    connection = pool._invoke_creator(self)
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/strategies.py", line 114, in connect
    return dialect.connect(*cargs, **cparams)
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/default.py", line 508, in connect
    return self.dbapi.connect(*cargs, **cparams)
  File "/app/venv/lib/python3.10/site-packages/psycopg2/__init__.py", line 122, in connect
    conn = _connect(dsn, connection_factory=connection_factory, **kwasync)
sqlalchemy.exc.OperationalError: (psycopg2.OperationalError) connection to server at "localhost" (::1), port 15432 failed: Connection refused
	Is the server running on that host and accepting TCP/IP connections?
connection to server at "localhost" (127.0.0.1), port 15432 failed: Connection refused
	Is the server running on that host and accepting TCP/IP connections?

(Background on this error at: http://sqlalche.me/e/13/e3q8)
exit_code=1
```

### File:line grounding

- **DB connect at import:** `app/db.py:L9-11` `engine = create_engine(config.DB_URI,
  connect_args={"application_name": config.DB_CONN_NAME})`; `app/db.py:L12` `connection =
  engine.connect()` (executed at import, not lazily); `app/db.py:L14` `Session =
  scoped_session(sessionmaker(bind=connection))`. `application_name` value: `config.DB_CONN_NAME`
  = `os.environ.get("DB_CONN_NAME", "webapp")` (`app/config.py:L193`) = `"webapp"` (observed).
- **Import chain that triggers it (from the traceback):** `server.py:L31` `from app.admin_model
  import (…)` → `app/admin_model.py:L11` `from app import models, s3` → `app/models.py:L32`
  `from app.db import Session` → `app/db.py:L12` `connection = engine.connect()`.
- **Reloader double-import:** the module is imported once in the reloader parent and once in the
  worker child (`WERKZEUG_RUN_MAIN`); see Q6 grounding (`werkzeug/_reloader.py:L182,L325-339`) and
  the two `SL` "load words file" lines (PID 7067 then 7079) above.
- **`OAUTHLIB_INSECURE_TRANSPORT`:** `server.py:L124`
  `os.environ["OAUTHLIB_INSECURE_TRANSPORT"] = "1"` — unconditional, module-level (preceding comment
  `server.py:L123` "the app is served behind nginx which uses http and not https"). It therefore
  also runs when `wsgi.py` imports `server` in production (`wsgi.py:L1,L3`).

### Observed vs Inferred

- **Observed:** the two `webapp` connections while the dev server is up (and zero before/after); the
  import-time banners printing twice with distinct PIDs (7067 parent, 7079 child); the connection
  being open immediately after a bare `import app.db`; the `OAUTHLIB_INSECURE_TRANSPORT` flip
  `None → '1'`; and the complete fail-fast traceback with exit code `1` when `DB_URI` points at a
  dead port.
- **Inferred:** that `OAUTHLIB_INSECURE_TRANSPORT` is likewise set on the production `wsgi.py` import
  path — read from the unconditional placement at `server.py:L124` (production `wsgi` was not run
  here). The bare `import app.db` probe in (3) is explicitly **non-canonical** (not the
  `python server.py` entry point); it is used only to isolate the connect-at-import, whose canonical
  manifestation is the two `pg_stat_activity` rows in (1).

### Further observed DEV-mode behaviours (beyond import-time)

Three additional runtime behaviours were exercised while answering "anything else going on". Each is
reported **Observed**. Where a fix would require changing application source, that is flagged
**out of scope** for this read-only investigation (this document only adds an answer file; it does
not modify source — AAP §0.5.2 / §0.7.7).

#### (a) Concurrent requests race on the single shared DB connection

Because `app/db.py:L12` opens **one** module-global `connection = engine.connect()` and `L14` binds
`Session` to **that** connection (not to the engine pool), and because the dev server is **threaded
by default** (Flask 1.1.2 `Flask.run` executes `options.setdefault("threaded", True)`), two
overlapping requests share a single DB transaction and race. Fired 60 concurrent authenticated
`GET /api/user_info` (rate-limiting is off via `DISABLE_RATE_LIMIT=1`, so nothing throttles them):

```bash
export PGPASSWORD=test
PGP="psql -h localhost -U test -d test -tA"
# per run: record the log position, capture the counter BEFORE, fire 60 concurrent authed
# GET /api/user_info, then capture the status distribution, the counter AFTER, and only THIS
# run's new server-log lines (from the recorded position onward):
mark=$(wc -l < /tmp/sl_run.log)
$PGP -c "select times from api_key where code='code';"                 # counter BEFORE
for i in $(seq 1 60); do
  ( curl -s -o /dev/null -w "%{http_code}\n" -H "Authentication: code" \
      http://localhost:7777/api/user_info >> /tmp/q7_conc.txt ) &
done; wait
sort /tmp/q7_conc.txt | uniq -c                                        # HTTP status distribution
$PGP -c "select times from api_key where code='code';"                 # counter AFTER
tail -n +$((mark+1)) /tmp/sl_run.log \
  | grep -E "This transaction is inactive|InvalidRequestError" | sort | uniq -c
```

The exact split varies run-to-run (it is a race), so the **same unchanged burst was run twice under
the same server process** (child pid `38146`); the race is present in **both**. HTTP status
distribution (`sort /tmp/q7_conc.txt | uniq -c`), run A then run B:

```text
     48 200
     12 500
```
```text
     52 200
      8 500
```

The `code` key usage counter (`select times from api_key where code='code';`) BEFORE then AFTER each
burst — run A, then run B:

```text
12
19
```
```text
19
23
```

THIS-run server-log lines matching the transaction error (`tail -n +$((mark+1)) /tmp/sl_run.log |
grep -E "This transaction is inactive|InvalidRequestError" | sort | uniq -c`) — run A (12 failures):

```text
     12     raise exc.InvalidRequestError("This transaction is inactive")
      1 2026-07-14 04:55:51,172 - SL - ERROR - 38146 - "/app/server.py:390" - error_handler() -  - This transaction is inactive
      1 2026-07-14 04:55:51,194 - SL - ERROR - 38146 - "/app/server.py:390" - error_handler() -  - This transaction is inactive
      1 2026-07-14 04:55:51,221 - SL - ERROR - 38146 - "/app/server.py:390" - error_handler() -  - This transaction is inactive
      2 2026-07-14 04:55:51,424 - SL - ERROR - 38146 - "/app/server.py:390" - error_handler() -  - This transaction is inactive
      1 2026-07-14 04:55:51,425 - SL - ERROR - 38146 - "/app/server.py:390" - error_handler() -  - This transaction is inactive
      1 2026-07-14 04:55:51,426 - SL - ERROR - 38146 - "/app/server.py:390" - error_handler() -  - This transaction is inactive
      1 2026-07-14 04:55:51,431 - SL - ERROR - 38146 - "/app/server.py:390" - error_handler() -  - This transaction is inactive
      1 2026-07-14 04:55:51,446 - SL - ERROR - 38146 - "/app/server.py:390" - error_handler() -  - This transaction is inactive
      1 2026-07-14 04:55:51,497 - SL - ERROR - 38146 - "/app/server.py:390" - error_handler() -  - This transaction is inactive
      1 2026-07-14 04:55:51,548 - SL - ERROR - 38146 - "/app/server.py:390" - error_handler() -  - This transaction is inactive
      1 2026-07-14 04:55:51,595 - SL - ERROR - 38146 - "/app/server.py:390" - error_handler() -  - This transaction is inactive
     12 sqlalchemy.exc.InvalidRequestError: This transaction is inactive
```

run B (8 failures):

```text
      8     raise exc.InvalidRequestError("This transaction is inactive")
      1 2026-07-14 04:55:53,833 - SL - ERROR - 38146 - "/app/server.py:390" - error_handler() -  - This transaction is inactive
      1 2026-07-14 04:55:53,843 - SL - ERROR - 38146 - "/app/server.py:390" - error_handler() -  - This transaction is inactive
      1 2026-07-14 04:55:53,846 - SL - ERROR - 38146 - "/app/server.py:390" - error_handler() -  - This transaction is inactive
      1 2026-07-14 04:55:53,873 - SL - ERROR - 38146 - "/app/server.py:390" - error_handler() -  - This transaction is inactive
      1 2026-07-14 04:55:54,130 - SL - ERROR - 38146 - "/app/server.py:390" - error_handler() -  - This transaction is inactive
      1 2026-07-14 04:55:54,139 - SL - ERROR - 38146 - "/app/server.py:390" - error_handler() -  - This transaction is inactive
      1 2026-07-14 04:55:54,141 - SL - ERROR - 38146 - "/app/server.py:390" - error_handler() -  - This transaction is inactive
      1 2026-07-14 04:55:54,239 - SL - ERROR - 38146 - "/app/server.py:390" - error_handler() -  - This transaction is inactive
      8 sqlalchemy.exc.InvalidRequestError: This transaction is inactive
```

Interpretation (outside the blocks): in **both** runs a subset of the concurrent requests returned
`500`, each carrying a full server-side `sqlalchemy.exc.InvalidRequestError: This transaction is
inactive` traceback logged by `error_handler` (`server.py:L390`); and in **both** runs the usage
counter advanced far less than the number of `200`s (run A `12 → 19`, i.e. 7 increments against 48
successes; run B `19 → 23`, i.e. 4 increments against 52 successes), so dozens of increments were
lost. Both are direct consequences of many threads sharing the single `app/db.py:L14` `Session` bound
to the single `app/db.py:L12` `connection`; the failure count (12 vs 8) varies because it is
timing-dependent.

- **Grounding:** `app/db.py:L12` `connection = engine.connect()`; `app/db.py:L14` `Session =
  scoped_session(sessionmaker(bind=connection))`; threaded default in `flask/app.py`
  `Flask.run` (`options.setdefault("threaded", True)`, observed above); the `500` path is the
  catch-all `@app.errorhandler(Exception)` (`server.py:L388`) → `error_handler` (`L389`) → `LOG.e(e)`
  (`L390`, the SL-ERROR line above) → `jsonify(error="Internal error"), 500` for `/api/` (`L392-393`).
- **Observed.** **Out of scope to fix:** binding `Session` to the engine/pool instead of a single
  connection is an application-source change.

#### (b) The Flask Debug Toolbar is enabled, and the signed session is forgeable given the public `FLASK_SECRET`

`local_main()` enables the Debug Toolbar in dev (`server.py:L577` `from flask_debugtoolbar import
DebugToolbarExtension`; `L581` `app.debug = True`; `L582` `DebugToolbarExtension(app)`; `L588`
`app.run(debug=True, port=7777)`). Observed consequences:

```bash
# (b1) the toolbar injects into every authenticated HTML response. Fetch the dashboard with a valid
#      logged-in cookie jar (cj), then scan the returned HTML for toolbar markers:
curl -s -b cj http://localhost:7777/dashboard/ -o dash.html          # 200
grep -oiE '_debug_toolbar/static/[^"]*|id="flDebug|Flask-Debug' dash.html | sort -u
```

```text
Flask-Debug
_debug_toolbar/static/'</script>
_debug_toolbar/static/js/jquery.js
_debug_toolbar/static/js/jquery.tablesorter.js
_debug_toolbar/static/js/toolbar.js
flask-debug
id="flDebug
```

```bash
# the toolbar's static-asset namespace is served live:
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:7777/_debug_toolbar/static/js/toolbar.js
```

```text
200
```

```bash
# the toolbar's ConfigVars panel is embedded in the page, exposing app-config rows:
grep -oiE 'ConfigVarsPanel|<td>SECRET_KEY</td>|SQLALCHEMY_DATABASE_URI' dash.html | sort -u
```

```text
<td>SECRET_KEY</td>
ConfigVarsPanel
SQLALCHEMY_DATABASE_URI
```

```bash
# (b2) NON-CANONICAL forge (signed-cookie mode, MEM_STORE_URI=""): mint a Flask-Login session cookie
#      WITHOUT logging in, signed with the public example.env default FLASK_SECRET="secret".
#      forge.py replicates flask_login._create_identifier() and signs via SecureCookieSessionInterface;
#      it prints ONLY the cookie length, never the cookie value.
ALT=<john_alt_id>                                              # john's alternative_id (= _user_id)
C1=$(python /tmp/forge.py "$ALT" secret      forge 127.0.0.1 goodid)    # correct secret + correct _id
C2=$(python /tmp/forge.py "$ALT" WRONGSECRET forge 127.0.0.1 goodid)    # wrong signing secret
C3=$(python /tmp/forge.py "$ALT" secret      forge 127.0.0.1 wrongid)   # correct secret, wrong client _id
echo "forged cookie length (correct secret+_id) = ${#C1}"
curl -s -A forge -b "slapp=$C1" -o d1.html -w "  [1] correct secret + correct _id : /dashboard/ = %{http_code}\n" http://localhost:7777/dashboard/
echo "  [1] identity shown on the forged dashboard page: $(grep -oiE 'John Wick' d1.html | head -1)"
curl -s -A forge -b "slapp=$C2" -o /dev/null -w "  [2] WRONG secret       + correct _id : /dashboard/ = %{http_code}\n" http://localhost:7777/dashboard/
curl -s -A forge -b "slapp=$C3" -o /dev/null -w "  [3] correct secret + WRONG _id (strong-prot) : /dashboard/ = %{http_code}\n" http://localhost:7777/dashboard/
```

```text
forged cookie length (correct secret+_id) = 262
  [1] correct secret + correct _id : /dashboard/ = 200
  [1] identity shown on the forged dashboard page: John Wick
  [2] WRONG secret       + correct _id : /dashboard/ = 302
  [3] correct secret + WRONG _id (strong-prot) : /dashboard/ = 200
```

The forge script prints only the cookie *length* (`262`); the signed cookie value itself is never
emitted, so no bearer material appears in the output above.

So the toolbar is served on every authenticated HTML page and its **ConfigVars panel exposes app
config rows including `SECRET_KEY` and `SQLALCHEMY_DATABASE_URI`**; and because `FLASK_SECRET` is the
public `example.env` default `"secret"`, a cookie **signed with that public secret authenticates as
the admin without any login** (`[1]` → `200`, "John Wick"), while a wrong signature is rejected
(`[2]` → `302`). Probe `[3]` shows Flask-Login's configured `session_protection = "strong"`
(`app/extensions.py:L8`) does **not** bind the session to the client here: because SimpleLogin marks
**every** session permanent (`make_session_permanent`, `server.py:L205-206` `session.permanent =
True`), the strong-protection check degrades to *basic* — `login_manager.py:L351`
`if mode == 'basic' or sess.permanent:` → `L352` `sess['_fresh'] = False` (mark non-fresh, **no
logout**); the session-clearing `elif mode == 'strong'` branch (`L355`) is never reached for a
permanent session. A mismatched client `_id` therefore does not invalidate a forged cookie.

**Why this is not a reachable remote-code-execution path in the canonical dev config** (each cause is
code/observation-grounded, not assumed):

- **Loopback-only bind.** `app.run(..., port=7777)` binds `127.0.0.1:7777` (observed in Q1/Q4 via
  `/proc/net/tcp`); the port is not reachable off-host.
- **Interactive debugger is PIN-gated and the PIN is never emitted.** Werkzeug's evalex console
  requires the `WERKZEUG_DEBUG_PIN`, which is printed through the `werkzeug` logger — and that logger
  is **disabled** (`app/log.py:L70-71`, shown in Q3), so no PIN appears in the captured startup
  stream.
- **A catch-all error handler intercepts before the debugger.** `@app.errorhandler(Exception)`
  (`server.py:L388-395`) converts any unhandled exception into a plain `500` response (JSON for
  `/api/`, `error/500.html` otherwise). Observed directly: the concurrency failures in (a) returned
  **status `500`, not an interactive Werkzeug debugger page**.

- **Grounding:** `server.py:L577,L581-582,L588`; `app/extensions.py:L8`; `server.py:L205-206`;
  `flask_login/login_manager.py:L351-352,L355`; `app/log.py:L70-71`; `server.py:L388-395`.
- **Observed** (the forge is explicitly **NON-CANONICAL** — it bypasses the `POST /auth/login` entry
  point by minting a cookie directly). **Out of scope to fix:** disabling the toolbar in
  `local_main()` and requiring a non-default `FLASK_SECRET` are application-source/configuration
  changes; `FLASK_SECRET="secret"` is a disposable DEV-only default (see the *Security scope* note).

#### (c) `flask dummy-data` is not idempotent

The seed command logs a "reset db" banner but does **not** drop or recreate tables: `dummy_data()`
(`server.py:L491-497`) logs `LOG.w("reset db, add fake data")` (`L494`) and then calls `fake_data()`
(`L495`), which at `app/fake_data.py:L44-56` immediately does `User.create(email="john@wick.com", …)`
+ `Session.commit()`. Run against the already-seeded dev DB, the first insert violates the
`users.email` unique constraint:

```bash
export PGPASSWORD=test
PGP="psql -h localhost -U test -d test -tA"          # -t tuples-only, -A unaligned => bare scalar
# BEFORE: user count, then john's id
$PGP -c "select count(*) from users;"
$PGP -c "select id from users where email='john@wick.com';"
```

```text
2
1
```

```bash
# re-run the seed against the already-populated dev DB; grep the salient lines and capture exit code
/app/venv/bin/flask dummy-data 2>&1 \
  | grep -E "reset db, add fake data|create fake data|sqlalchemy.exc.IntegrityError|already exists"
echo "exit_code=${PIPESTATUS[0]}"
```

```text
2026-07-14 04:54:14,347 - SL - WARNING - 38181 - "/app/server.py:494" - dummy_data() -  - reset db, add fake data
2026-07-14 04:54:14,347 - SL - DEBUG - 38181 - "/app/app/fake_data.py:41" - fake_data() -  - create fake data
DETAIL:  Key (email)=(john@wick.com) already exists.
sqlalchemy.exc.IntegrityError: (psycopg2.errors.UniqueViolation) duplicate key value violates unique constraint "users_email_key"
DETAIL:  Key (email)=(john@wick.com) already exists.
exit_code=1
```

```bash
# AFTER: user count, then john's id
$PGP -c "select count(*) from users;"
$PGP -c "select id from users where email='john@wick.com';"
```

```text
2
1
```

So `flask dummy-data` is a **one-shot seed for a fresh/empty schema**, not a reset: on a populated DB
it aborts at the very first row with `exit 1` and leaves the existing data intact (verified: user
count `2` before and after, `john@wick.com` id `1` untouched).

- **Grounding:** `server.py:L491` `def dummy_data():`, `L494` `LOG.w("reset db, add fake data")`,
  `L495` `fake_data()`; `app/fake_data.py:L44-56` `User.create(email="john@wick.com", …)` +
  `Session.commit()`; unique constraint `users_email_key` on `users.email`.
- **Observed.** **Out of scope to fix:** making the seed idempotent (drop/recreate, or upsert/guard)
  is an application-source change.



## Coverage-pass checklist

Final pass re-reading each question and every named item. Each box is checked **only** where the
body contains observed runtime evidence for it (with the section that carries the evidence).

**Scope of this checklist (per AAP §0.7.4).** This is the coverage pass over the seven questions and
their named sub-items — the "each distinct thing and named item" the prompt asks for — **not** a
general security or dependency audit. Topics that no question raises — CORS / security-response
headers, CRLF / log injection, exhaustive malformed / boundary API fuzzing, and third-party
dependency CVE advisories — are **outside the question scope** and are deliberately excluded per
AAP §0.5.2 / §0.7.7 (read-only, no source changes) and §0.8 ("do not expand into a full security
audit"); they are intentionally absent as checklist items rather than checked-but-unevidenced. Where
a behaviour flagged elsewhere *does* fall inside a question it is evidenced and listed here — the
Debug Toolbar and the shared-connection concurrency race under Q7 *(b)/(a)*, and the session-lifecycle
items (no rotation, stale-cookie replay, negative CSRF) under Q5 *(I)*.

**Q1 — startup in dev mode**
- [x] Canonical entry `python3 server.py` → `local_main()` runs — Q1 output (RUN‑A/RUN‑B)
- [x] `config.COLOR_LOG = True` set by `local_main` — Q1/Q3 grounding + Q3 colorlog probe
- [x] `create_app()` builds the app — Q1 grounding (`server.py:L574`)
- [x] Flask‑DebugToolbar enabled, `app.debug = True` — Q1 grounding (`server.py:L577,L581-582`)
- [x] `app.run(debug=True, port=7777)` — Q1 output + Q4 port evidence

**Q2 — configuration loading**
- [x] `CONFIG` env var branch (`load config file …` print) — Q2 output (`full.env` run)
- [x] Default `./.env` branch (`load_dotenv()`) — Q2 output (default import)
- [x] `python-dotenv` `override=False` precedence (shell env wins) — Q2 output (URL precedence)
- [x] Required‑var hard‑fail for `URL`, `EMAIL_DOMAIN`, `SUPPORT_EMAIL`, `DB_URI`, `FLASK_SECRET` — Q2 output (5 KeyError runs)
- [x] Present‑but‑empty `FLASK_SECRET` → `RuntimeError` — Q2 output
- [x] `EMAIL_SERVERS_WITH_PRIORITY` (sixth required, `sl_getenv`) → `TypeError` — Q2 output

**Q3 — readiness log messages**
- [x] `>>> init logging <<<` — Q3 output
- [x] `>>> URL: http://localhost` — Q3 output
- [x] `werkzeug` logger disabled → standard access log suppressed — Q3 grounding + startup stream
- [x] Werkzeug `* Serving … * Environment … * Debug mode: on` banner (what actually appears) — Q3/Q1 output
- [x] `/health` is a request‑time probe, not a startup marker — Q3 output (out‑of‑band probe)

**Q4 — ports and endpoints**
- [x] Listening port `127.0.0.1:7777` (raw `/proc/net/tcp`) — Q4 output
- [x] `/health` → `200 success` — Q4 output
- [x] All 10 blueprints + prefixes (`auth`,`api`,`dashboard`,`developer`,`phone`,`monitor`@`/`,`oauth`,`oauth2`,`onboarding`,`discover`,`internal`) — Q4 route dump + grounding
- [x] Direct routes (`/`, `/health`, `/favicon.ico`, `/dnt`, `/.well-known/openid-configuration`, `/jwks`) and `/admin` mount — Q4 route dump
- [x] Exact live route count (292) stable across two builds — Q4 output
- [x] `monitor_bp` `/` discrepancy noted — Q4 grounding

**Q5 — authenticated request (PRIMARY)**
- [x] (a) entry: `ProxyFix` + the three `before_request` hooks in order — Q5 (A)
- [x] `ProxyFix` `x_for` and `x_host` both honored — Q5 (A)
- [x] (b) identity, web: `_user_id` = `alternative_id` UUID via `load_user` — Q5 (B) + DB proof
- [x] (b) identity, API: `Authentication` header → `authorize_request` → `g.user` — Q5 (C)
- [x] API valid / wrong / absent / session‑fallback — Q5 (C)
- [x] API usage‑stats side effect (commit before guards); `g.api_key=None` on fallback — Q5 (C)
- [x] Account guards: API disabled(403)/inactive(401); web `load_user` disabled/inactive(302) — Q5 (D)
- [x] (c) propagation: `current_user` (web) / `g.user` (API); `get_current_user()` only in 429 handler — Q5 (E) + grounding
- [x] Per‑request `after_request` `SL` log, `/health` & `/static` excluded — Q5 (E)
- [x] Both session backends (Redis server‑side vs signed cookie); before/intermediate/after TTLs — Q5 (B),(F)
- [x] Signed cookie is signed (readable w/o secret), not encrypted — Q5 (F)
- [x] 401 edge: web → 302 login redirect; API JSON — Q5 (A),(C) + grounding
- [x] Web logout: `GET /auth/logout` → `302` + `slapp`/`mfa`/`dark-mode` deleted (`Max-Age=0`) + Redis session purged; post-logout `/dashboard/` → `302` (before `200` / after `302`) — Q5 (G)
- [x] API key revoked over HTTP on a concrete endpoint: acquire `200` → use `200` → revoke → reuse `401 {"error": "Wrong api key"}` on `/api/user_info` — Q5 (H′) canonical sudo‑gated UI delete; (H) SQL‑fixture supplemental
- [x] Session ID **not** rotated across the login boundary (session‑fixation observation; same sid, TTL `300`→`604800`) — Q5 (I) (i‑1)
- [x] Stale signed‑cookie replay after logout in signed‑cookie mode (empty `MEM_STORE_URI`): the logged‑out cookie still authenticates; tampering → `302` — Q5 (I) (i‑2)
- [x] Negative CSRF: `POST` without a valid token re‑renders the form (no state change) — Q5 (I) (i‑3)

**Q6 — background jobs / schedulers**
- [x] Webapp starts no scheduler thread (thread counts, `sys.modules` = NONE) — Q6 output
- [x] `cron.py`, `job_runner.py`, `email_handler.py`, `event_listener.py` independent `__main__` — Q6 output
- [x] `create_light_app` used by first three; not by `event_listener.py` — Q6 output
- [x] External `crontab.yml` scheduling (yacron) — Q6 output

**Q7 — anything else (import‑time side effects)**
- [x] DB connection opened at import (`app/db.py:L12`) — Q7 output (pg_stat + probe)
- [x] Reloader double‑import → two `webapp` DB connections + doubled banners — Q7 output
- [x] `OAUTHLIB_INSECURE_TRANSPORT` set to `"1"` at import, unconditional (prod too) — Q7 output
- [x] Fail‑fast on unreachable DB (full traceback, exit 1) — Q7 output
- [x] Concurrency: single shared `connection`/`Session` (`db.py:L12,L14`) + threaded dev server → mixed `200`/`500` (run A 48×200/12×500, run B 52×200/8×500), "This transaction is inactive", lost usage‑counter increments — Q7 *Further …* (a)
- [x] Debug Toolbar enabled in dev + ConfigVars panel exposes `SECRET_KEY`/`SQLALCHEMY_DATABASE_URI`; signed session forgeable given public `FLASK_SECRET` (NON‑CANONICAL forge → 200; wrong secret → 302); strong→basic degradation for permanent sessions; not a reachable RCE (loopback/PIN‑disabled/errorhandler‑intercept) — Q7 *Further …* (b)
- [x] `flask dummy-data` not idempotent → `users_email_key` UniqueViolation on seeded DB, exit 1, no corruption — Q7 *Further …* (c)


## Cleanup and read-only guarantee

The investigation was run-first and strictly read-only: the only tracked change is this answer
document. All observation was performed inside the throwaway container, and all temporary artifacts
lived under the container's ephemeral `/tmp/blitzy_cap/` (never inside the tracked repository tree).

**Runtime state after the investigation** (no lingering server, no server-side session state, and
the disposable guard-test users removed — `john@wick.com` was never mutated):

```text
server_processes=0
port_7777=FREE
redis_session_keys=0
```
```text
1|john@wick.com
2|winston@continental.com
```

**Bearer-material invalidation.** The flushed Redis sessions (`redis_session_keys=0` above) make
every captured `slapp` **session-id** cookie and its paired CSRF token dead. The one API key
acquired canonically during Q5 (`blitzy-q5-device` — the value captured in `api_auth_login.txt`,
shown there as `<REDACTED:api_key-value>` per the secret-handling convention above) was deleted at
completion; re-running the exact lookup `authorize_request` performs — `ApiKey.get_by(code=…)`
(`app/api/base.py:L18`), keyed on that same acquired value (printed truncated as `rvys...` in the
capture below) — now returns `None`, so any request bearing that key takes the `Wrong api key` `401`
path (`app/api/base.py:L27`) and the key is non-reusable:

```text
before delete: ApiKey.get_by(code=rvys...) => FOUND id=5 name=blitzy-q5-device
after  delete: ApiKey.get_by(code=rvys...) => None
remaining john api_keys: [(1, 'Chrome', 'code', 1), (2, 'Firefox', 'codeFF', 0)]
```

The additional disposable key created for the subsection-(H) revocation demonstration
(`blitzy-q5-revoke`) is deleted at its revoke step — subsection (H)'s `revoke_db.txt` shows `0` rows
remaining for that `code` — so it too is non-reusable (its reuse over HTTP is the `401 Wrong api key`
shown in (h-4)). The seeded `code`/`codeFF` keys remain as standard `flask dummy-data` seed values
(the `code` key's `times` counter increases monotonically with each valid-key authentication used in
testing — a benign seed side-effect, never reset so as not to fabricate state); they are disposable
DEVELOPMENT-only credentials in the git-ignored throwaway dev database, never production material.

**Read-only proof — across the whole branch since the source baseline, exactly one path differs:
this document.** The commit-independent check is the branch delta against the pre-Blitzy source
baseline `2cd6ee77` (`chore: emit some missing contact audit logs (#2269)`); it lists a single path
whether or not this deliverable has been committed yet:

```bash
git rev-parse --abbrev-ref HEAD                    # blitzy-a91b3111-6deb-4252-be6e-91d062c6ed20
git diff --name-only 2cd6ee77 --                   # paths differing from the source baseline
git diff --name-only 2cd6ee77 -- | wc -l
```
```text
blitzy/documentation/app_2cd6ee777f8c.md
1
```

The **working-tree** status, by contrast, changes with the commit and must be read accordingly.
*Before* this deliverable is committed — the state the investigation itself leaves — `git status
--porcelain` reports the one modified path:

```bash
git status --porcelain          # investigation end state, before the deliverable is committed
```
```text
 M blitzy/documentation/app_2cd6ee777f8c.md
```

*After* the deliverable is committed (the repository's final state) the working tree is **clean** —
`git status --porcelain` prints nothing — and the single change is recorded as that same one file in
the commit, confirmable with `git show --stat HEAD` (`1 file changed`,
`blitzy/documentation/app_2cd6ee777f8c.md`). An empty post-commit `git status` is therefore the
expected end state, not a discrepancy: exactly one file ever changes — either as a working-tree
modification (pre-commit) or as a one-file commit (post-commit).

**Note on `git diff --check`.** The terminal blank line at EOF flagged by the original review has
been removed (one final newline is retained). The remaining `git diff --check` notices are all
**trailing spaces that exist inside verbatim evidence blocks**, and come from three distinct
tool-emitted sources: (1) `psql` column padding in the seeded-account verification (the `\x`
expanded `delete_on      |` NULL field and the aligned `api_key` table header in the
Investigation-environment section); (2) the kernel's fixed-width column padding in the
`/proc/net/tcp` listener dump (Q4); and (3) the trailing space after each field in SimpleLogin's
**pretty-printed** API JSON (Q5, e.g. `"email": "john@wick.com", `). These bytes are reproduced
exactly as the tools emitted them; stripping them would violate the complete-and-unedited evidence
rule (and would re-introduce the very "compact vs pretty-print" gap the review called out). They are
therefore preserved intentionally.

Temporary observation scripts and capture files under `/tmp/blitzy_cap/` in the container are
deleted at completion; because they never resided in the tracked tree, their removal leaves the
repository byte-for-byte unchanged apart from this document.
