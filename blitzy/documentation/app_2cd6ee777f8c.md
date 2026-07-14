# SimpleLogin Runtime Behavioral Investigation — `app_2cd6ee777f8c`

A read-only, **evidence-based runtime investigation** of the SimpleLogin email-aliasing Flask
application. Every answer below was produced by **actually building, running, and exercising the
system through its real, canonical entry points** and capturing the command and its complete,
untruncated output. Source anchors (`file:line`) are given as the *causal explanation*; the value
reported for each question is the value captured from the running system.

### Labelling convention — `[OBSERVED]` vs `[INFERRED]`

- **`[OBSERVED]`** — a value captured directly from the running system, shown next to the exact
  command that produced it.
- **`[INFERRED]`** — a statement derived from reading source code or an authoritative external
  reference, used where the claim itself is not a directly-capturable runtime signal. `[INFERRED]`
  is used substantively in this document, specifically for: the Microsoft-IIS provenance of HTTP
  status `440` (Q2); the internal reason Flask-Login 0.5.0 does not rotate the server-side session
  id on login (Q4); the delivery of the failed-login `LoginEvent` to NewRelic via
  `record_custom_event` (Q8); the exact `600`-second token constant, which runtime observation
  brackets to `(597s, 607s]` (Q6); and the "pickle protocol 4" reading of the leading session
  bytes (Q3).

This document does **not** claim that every value is observed; source-derived semantics are
labelled `[INFERRED]` and kept separate from captured output.

---

## Methodology

### Runtime stack (canonical build/run)

The application is the SimpleLogin Flask/Python monolith. It was stood up in the **canonical**
configuration required by the AAP: served with **gunicorn** on port **7777** with
**`--timeout 15`** and **`-w 2`**, backed by **PostgreSQL 13** and **Redis 6**, with the
Redis-backed server-side session store wired in (`MEM_STORE_URI` set). The application code and
virtualenv run inside the provided image container (`sl-app`, bind-mounting the repository at
`/app`); PostgreSQL 13 (`sl-pg13`, image `postgres:13`) and Redis 6 (`sl-redis6`, image
`redis:6`) run as separate containers on a shared Docker network, so the stack matches the
frozen AAP rather than the image's bundled PostgreSQL 15 / Redis 7.

Observed component versions (from the venv actually running the app) `[OBSERVED]`:

```text
=== component versions (from the venv actually running) ===
Python 3.10.18
flask==1.1.2
flask-login==0.5.0
flask-wtf==0.14.3
aiosmtpd==1.4.2
redis==4.6.0
itsdangerous==1.1.0
sqlalchemy==1.3.24
arrow==0.16.0
gunicorn==20.0.4
newrelic==8.8.0
psycopg2-binary==2.9.3
cryptography==37.0.1
```

Containers and image digest, and the exact `DB_URI`/`MEM_STORE_URI` wiring of the running app
`[OBSERVED]`:

```text
=== Docker containers (canonical stack) ===
NAMES       IMAGE                                                           PORTS
sl-redis6   redis:6                                                         6379/tcp
sl-pg13     postgres:13                                                     5432/tcp
sl-app      ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0   0.0.0.0:7777->7777/tcp

=== Image digest (sl-app runtime image) ===
ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0
REPOSITORY                   TAG                                  DIGEST                                                                    IMAGE ID       CREATED        SIZE
ghcr.io/scaleapi/swe-atlas   swe_atlas_QnA_simple-login_app_1.0   sha256:b82cb15631e92ade58b8cf10493550f03a54dc5a8f25d3f31ef41afc186ee2c1   ea242796bbce   5 months ago   2.14GB

=== MEM_STORE_URI wiring proof (from running app env) ===
export DB_URI="postgresql://test:test@sl-pg13:5432/test"
export MEM_STORE_URI="redis://sl-redis6"
exec gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15 \
```

Backing-service versions and application health, captured live from the canonical stack
`[OBSERVED]`:

```text
### Backing service versions (canonical stack, queried live):
$ docker exec sl-pg13 psql -U test -d test -tAc 'select version();'
PostgreSQL 13.23 (Debian 13.23-1.pgdg13+1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 14.2.0-19) 14.2.0, 64-bit
$ docker exec sl-redis6 redis-server --version
Redis server v=6.2.22 sha=00000000:0 malloc=jemalloc-5.1.0 bits=64 build=16fd36f46dbcd9b6

### App health check (canonical gunicorn on :7777):
$ curl -sS -o /dev/null -w 'GET /auth/login -> HTTP %{http_code}\n' http://localhost:7777/auth/login
GET /auth/login -> HTTP 200
$ curl -sS -o /dev/null -w 'GET /api/aliases (no auth) -> HTTP %{http_code}\n' http://localhost:7777/api/aliases?page_id=0
GET /api/aliases (no auth) -> HTTP 401

### Gunicorn worker processes (canonical -w 2 --timeout 15):
$ docker exec sl-app bash -lc "ps -eo pid,cmd | grep '[g]unicorn' | head"
   2480 python gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15 --access-logfile /tmp/gunicorn_access.log --error-logfile /tmp/gunicorn_error.log
   2481 python gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15 --access-logfile /tmp/gunicorn_access.log --error-logfile /tmp/gunicorn_error.log
   2482 python gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15 --access-logfile /tmp/gunicorn_access.log --error-logfile /tmp/gunicorn_error.log
```

> **Why the health check shows two codes.** `GET /auth/login` returns `200` (the app is up).
> `GET /api/aliases?page_id=0` with **no credential at all** returns `401` — the endpoint requires
> authorization by default. Q1 exercises the same endpoint with a *browser-session cookie* and
> observes `200`, which is precisely the session-fallback behavior under investigation.

> **`MEM_STORE_URI` is mandatory for Q3/Q4.** `server.py` wires the `RedisSessionStore` (via
> `initialize_redis_services`) only when `MEM_STORE_URI` is set (`server.py:L163-L165`,
> `app/config.py:L568`). It was set to `redis://sl-redis6`, so sessions are stored in Redis and
> Q3/Q4 are observable canonically. The signing secret is loaded from `/app/.env`
> (`FLASK_SECRET`, `app/config.py:L196`); its value is **redacted** throughout this document.

### Exact commands used to stand up the system

```bash
# Canonical backing services (separate containers, shared docker network 'sl-canon'):
#   postgres:13  -> host 'sl-pg13'   redis:6 -> host 'sl-redis6'
# The app container (sl-app) is attached to the same network.

# Activate the project virtualenv; config.py auto-loads /app/.env via python-dotenv.
cd /app && . venv/bin/activate

# Point the app at the canonical PG13 + Redis6 services (override the image's bundled PG15/Redis7):
export DB_URI="postgresql://test:test@sl-pg13:5432/test"
export MEM_STORE_URI="redis://sl-redis6"
export PYTHONPATH=/app

# Database schema provisioning + migrations (see "Database migration" below).
alembic current          # -> 32f25cbf12f6 (head)
alembic upgrade head     # -> clean no-op (already at head)

# Seed a verified user / mailbox / alias / API key on the disposable empty DB:
FLASK_APP=wsgi:app flask dummy-data       # runs fake_data() + add_sl_domains()

# Canonical run command (AAP: gunicorn wsgi:app, port 7777, -w 2, --timeout 15):
gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15 \
    --access-logfile /tmp/gunicorn_access.log \
    --error-logfile /tmp/gunicorn_error.log
```

### Database migration (transparent disclosure of a workaround)

Running `alembic upgrade head` against a **brand-new, empty PostgreSQL 13** database **fails**
with an upstream migration-ordering defect: a data migration that builds a trigram index runs
before the table it targets exists. The literal failure `[OBSERVED]`:

```text
### DB_URI=postgresql://test:test@sl-pg13:5432/test
### alembic current (empty DB, before upgrade):
>>> URL: http://localhost:7777
WARNING: Use a temp directory for GNUPGHOME /tmp/ltoiztkbbmlkwfptfdxp
Upload files to local dir
>>> init logging <<<
2026-07-14 21:27:40,488 - SL - DEBUG - 2400 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
### alembic upgrade head:
    operations.impl.create_index(idx)
  File "/app/venv/lib/python3.10/site-packages/alembic/ddl/impl.py", line 283, in create_index
    self._exec(schema.CreateIndex(index))
  File "/app/venv/lib/python3.10/site-packages/alembic/ddl/impl.py", line 141, in _exec
    return conn.execute(construct, *multiparams, **params)
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1011, in execute
    return meth(self, multiparams, params)
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/sql/ddl.py", line 72, in _execute_on_connection
    return connection._execute_ddl(self, multiparams, params)
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 1068, in _execute_ddl
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
sqlalchemy.exc.ProgrammingError: (psycopg2.errors.UndefinedTable) relation "alias" does not exist

[SQL: CREATE INDEX note_pg_trgm_index ON alias USING gin (note gin_trgm_ops)]
(Background on this error at: http://sqlalche.me/e/13/f405)
### alembic current (after upgrade):
>>> URL: http://localhost:7777
WARNING: Use a temp directory for GNUPGHOME /tmp/gqyuiasiwnpvnnwqryoc
Upload files to local dir
>>> init logging <<<
2026-07-14 21:27:42,981 - SL - DEBUG - 2404 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
```

The failing statement is `CREATE INDEX note_pg_trgm_index ON alias USING gin (note gin_trgm_ops)`,
raised as `psycopg2.errors.UndefinedTable: relation "alias" does not exist`. This is a property of
the migration history, not of this investigation.

**Workaround (disclosed for full transparency):** the authentic, complete schema at Alembic head
`32f25cbf12f6` was provisioned onto the empty PG13 database using a **schema-only** `pg_dump`
(structure only, **no data**) taken from the image's reference database (which is stamped at the
same head), after which the Alembic version table was stamped to `32f25cbf12f6`. Because the dump
is schema-only, every data table was empty prior to seeding — this is the disposable-DB
initial-state guard. After provisioning, Alembic reports head and `upgrade head` is a clean no-op
`[OBSERVED]`:

```text
### alembic current:
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
32f25cbf12f6 (head)
### alembic upgrade head:
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
```

The schema so provisioned is byte-for-byte the canonical head schema (77 tables); the
session/pickle, alias-token, API-key, and email-forwarding behaviors probed here are
application-level and independent of how the schema rows were created.

### Canonical seed data

> **`flask dummy-data` is a one-time, non-idempotent, destructive fixture command.** It logs
> `"reset db, add fake data"` (`server.py:L494`) and calls `fake_data()` (`app/fake_data.py:L41`),
> which creates many records and writes local upload artifacts (e.g. `static/upload/profile_pic.svg`).
> It must only be run against a **disposable, empty** test database (satisfied here by the
> schema-only load above). The local-upload artifacts it generates are removed during cleanup
> (see the final section).

Seeding ran through the application's own canonical Flask CLI command (not by hand-inserting
rows). The seed log `[OBSERVED]`:

```text
2026-07-14 21:29:35,782 - SL - WARNING - 2451 - "/app/server.py:494" - dummy_data() -  - reset db, add fake data
2026-07-14 21:29:35,782 - SL - DEBUG - 2451 - "/app/app/fake_data.py:41" - fake_data() -  - create fake data
2026-07-14 21:29:36,083 - SL - INFO - 2451 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-14 21:29:36,108 - SL - DEBUG - 2451 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email operas_enzyme523@sl.local
2026-07-14 21:29:36,119 - SL - INFO - 2451 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-14 21:29:36,168 - SL - INFO - 2451 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-14 21:29:36,177 - SL - INFO - 2451 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-14 21:29:36,195 - SL - INFO - 2451 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-14 21:29:36,207 - SL - INFO - 2451 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-14 21:29:36,223 - SL - INFO - 2451 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-14 21:29:36,231 - SL - INFO - 2451 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-14 21:29:36,241 - SL - DEBUG - 2451 - "/app/app/models.py:1255" - generate_oauth_client_id() -  - generate oauth_client_id demo-vpymygnsyo
2026-07-14 21:29:36,247 - SL - DEBUG - 2451 - "/app/app/models.py:1255" - generate_oauth_client_id() -  - generate oauth_client_id demo2-zudkymvots
2026-07-14 21:29:36,519 - SL - INFO - 2451 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-14 21:29:36,544 - SL - INFO - 2451 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-14 21:29:36,556 - SL - INFO - 2451 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-14 21:29:36,566 - SL - INFO - 2451 - "/app/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain
```

Reading the seeded entities back through `psql` — these are the exact identities used by the
answers below `[OBSERVED]`:

```text
### Canonical seed readback (psql against the disposable PG13 test DB):
# Note: flask dummy-data was applied ONCE to the freshly schema-loaded (schema-only pg_dump => data-free) PG13 DB.

$ psql -tAc "select id,email,activated,is_admin from users where email='john@wick.com';"
1|john@wick.com|t|t
$ psql -tAc "select id,email,note from alias where id=2;"   -- john's canonical random alias
2|operas_enzyme523@sl.local|
$ psql -tAc "select id,email,verified,user_id from mailbox where id=1;"
1|john@wick.com|t|1
$ psql -tAc "select id,code,name,times,last_used from api_key where code='code';"  -- canonical default key (untouched)
1|code|Chrome|0|

### Column types for the Q7 counter fields (information_schema):
$ psql -tAc "select column_name,data_type from information_schema.columns where table_name='api_key' and column_name in ('times','last_used','sudo_mode_at') order by column_name;"
last_used :: timestamp without time zone
sudo_mode_at :: timestamp without time zone
times :: integer
```

Canonical seed identities used throughout (all `[OBSERVED]`):

- verified user **`john@wick.com`** / password **`password`** (`activated=t`, `is_admin=t`), **id=1**,
  `alternative_id=277992d1-b91a-44bc-8bbf-a4d555d05e2e`;
- a second seed user **`winston@continental.com`** (id=2, `app/fake_data.py:L236`);
- verified **mailbox** `john@wick.com` (**id=1**, owner id=1) and `pgp@example.org` (id=2);
- john's canonical random **alias** `operas_enzyme523@sl.local` (**id=2**), forwarding to mailbox `john@wick.com`;
- default **API keys** `code="code"` (name "Chrome", **id=1**, `times=0`, `last_used=NULL`) and
  `code="codeFF"` (name "Firefox", id=2), both untouched by the investigation;
- public domain `sl.local` (`init_app.py:L44`).

Two **disposable fixtures** were created *by the investigation* (not by `fake_data()`) so the
destructive Q2 test and the counter-sensitive Q7 test never mutate the canonical user or the
canonical `code` key: user **`q2victim@example.com`** (id=3) with API key **`q2disposkey`** (Q2),
and API key **`q7isolatedkey`** owned by john (Q7). Both are removed during cleanup.

### Conventions & cleanup

- All temporary helper scripts — the observation probes (created under the container's `/tmp`,
  outside the repository working tree) and the transient document-assembly tooling used to inject
  the captured evidence into this file — were **deleted** after use. The only repository change is
  this document. The final section ("Repository cleanup & validation") shows the tracked *and*
  ignored repository state as literal command output.
- No existing source file was modified, added, or deleted.
- **Evidence fidelity vs. whitespace.** Fenced code blocks reproduce captured tool output
  **verbatim**, including any trailing whitespace and blank HTTP-header separator lines (e.g.
  curl's `> `/`< ` markers) and the exact bytes of the Q8 HTTP body (so it matches its
  `Content-Length`). Consequently `git diff --check` may report "trailing whitespace" on lines
  that fall **inside evidence fences** — this is intentional, required by the complete/unedited
  evidence mandate, and confined to verbatim output. The document prose itself carries no trailing
  whitespace, and the file ends with a single trailing newline (no blank line at end-of-file).

---

## Q1 — API authentication via a browser session (no API key)

**Direct answer `[OBSERVED]`:** a request to a `require_api_auth`-guarded endpoint made with a
**browser-session cookie and no `Authentication` header** returns **HTTP `200 OK`** with the
endpoint's **normal JSON payload** (here, the full `application/json` alias list,
`Content-Length: 2550`). The session cookie alone authenticates the request.

**How it was exercised.** A self-contained login established a browser session (cookie jar), then
`GET /api/aliases?page_id=0` was issued with that cookie and **no** `Authentication` header, under
`curl -v` so the exact outgoing request headers and full response headers are captured.

Verbose request + response headers `[OBSERVED]` (note the single `Cookie: slapp=…` request header
and the **absence** of any `Authentication` header):

```text
* Host localhost:7777 was resolved.
* IPv6: ::1
* IPv4: 127.0.0.1
*   Trying [::1]:7777...
* Immediate connect fail for ::1: Cannot assign requested address
*   Trying 127.0.0.1:7777...
* Connected to localhost (127.0.0.1) port 7777
* using HTTP/1.x
> GET /api/aliases?page_id=0 HTTP/1.1
> Host: localhost:7777
> User-Agent: curl/8.14.1
> Accept: */*
> Cookie: slapp=7c40e349-8183-458f-9dda-590c7be3fa22.t5OfS2ihgEMYImcFhnO254jIv4Q
> 
* Request completely sent off
< HTTP/1.1 200 OK
< Server: gunicorn/20.0.4
< Date: Tue, 14 Jul 2026 21:34:23 GMT
< Connection: close
< Content-Type: application/json
< Content-Length: 2550
< Access-Control-Allow-Origin: *
* Replaced cookie slapp="7c40e349-8183-458f-9dda-590c7be3fa22.t5OfS2ihgEMYImcFhnO254jIv4Q" for domain localhost, path /, expire 1784669663
< Set-Cookie: slapp=7c40e349-8183-458f-9dda-590c7be3fa22.t5OfS2ihgEMYImcFhnO254jIv4Q; Expires=Tue, 21-Jul-2026 21:34:23 GMT; HttpOnly; Path=/; SameSite=Lax
< 
{ [2550 bytes data]
* shutting down connection #0
```

The session cookie jar used for the request `[OBSERVED]`:

```text
# Netscape HTTP Cookie File
# https://curl.se/docs/http-cookies.html
# This file was generated by libcurl! Edit at your own risk.

#HttpOnly_localhost	FALSE	/	FALSE	1784669663	slapp	7c40e349-8183-458f-9dda-590c7be3fa22.t5OfS2ihgEMYImcFhnO254jIv4Q
```

The complete, untruncated JSON response body (exactly 2550 bytes, as returned) `[OBSERVED]`:

```json
{"aliases":[{"creation_date":"2026-07-14 21:32:33+00:00","creation_timestamp":1784064753,"email":"q6bimm.dryads032@sl.local","enabled":true,"id":14,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":"q6-immediate"},{"creation_date":"2026-07-14 21:32:33+00:00","creation_timestamp":1784064753,"email":"q6aimm.milieu915@sl.local","enabled":true,"id":13,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":"q6-immediate"},{"creation_date":"2026-07-14 21:31:37+00:00","creation_timestamp":1784064697,"email":"q6validate.calmer990@sl.local","enabled":true,"id":12,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":"q6-validate"},{"creation_date":"2026-07-14 21:29:36+00:00","creation_timestamp":1784064576,"email":"wick@example.com","enabled":true,"id":11,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null},{"creation_date":"2026-07-14 21:29:36+00:00","creation_timestamp":1784064576,"email":"john@example.com","enabled":true,"id":10,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null},{"creation_date":"2026-07-14 21:29:36+00:00","creation_timestamp":1784064576,"email":"second@ab.cd","enabled":true,"id":8,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null},{"creation_date":"2026-07-14 21:29:36+00:00","creation_timestamp":1784064576,"email":"first@ab.cd","enabled":true,"id":7,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null},{"creation_date":"2026-07-14 21:29:36+00:00","creation_timestamp":1784064576,"email":"e2@sl.local","enabled":true,"id":6,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null},{"creation_date":"2026-07-14 21:29:36+00:00","creation_timestamp":1784064576,"email":"e1@sl.local","enabled":true,"id":5,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null},{"creation_date":"2026-07-14 21:29:36+00:00","creation_timestamp":1784064576,"email":"e0@sl.local","enabled":false,"id":4,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null},{"creation_date":"2026-07-14 21:29:36+00:00","creation_timestamp":1784064576,"email":"example@example.com","enabled":true,"id":3,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null},{"creation_date":"2026-07-14 21:29:36+00:00","creation_timestamp":1784064576,"email":"operas_enzyme523@sl.local","enabled":true,"id":2,"nb_block":0,"nb_forward":1,"nb_reply":0,"note":null},{"creation_date":"2026-07-14 21:29:36+00:00","creation_timestamp":1784064576,"email":"simplelogin-newsletter.bereft978@sl.local","enabled":true,"id":1,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":"This is your first alias. It's used to receive SimpleLogin communications like new features announcements, newsletters."}]}
```

> The alias inventory reflects **live runtime state** at capture time: it contains the canonical
> seed aliases plus aliases created earlier in this investigation (e.g. the Q6 form-created
> `q6*` aliases). The Q1 signal is the **`200` status + normal JSON structure** returned to a
> session-only request, not the specific alias set.

**Causal explanation.** The authorization pipeline is `authorize_request()` in
`app/api/base.py:L16-L43`, invoked by the `require_api_auth` decorator (`app/api/base.py:L52-L60`).
It reads the API key from the `Authentication` header (`app/api/base.py:L17`); when that header is
absent, `api_key` is falsy and control enters the session-fallback branch — if
`current_user.is_authenticated` it sets `g.user = current_user` (`app/api/base.py:L20-L25`) and
returns `None` (`app/api/base.py:L43`), i.e. "authorized". The view then runs normally and returns
its `200` JSON. (With **no** credential at all the same endpoint returns `401`, as shown in the
health check above — confirming the cookie is what authorizes here.)

---

## Q2 — Privileged (sudo) operation — BOTH conditions

The single `require_api_sudo`-guarded endpoint is `DELETE /api/user` (`delete_user`,
`app/api/views/user.py:L12-L14`). Both required conditions were exercised against a **disposable**
user/key so the destructive path could never affect the canonical user.

### Condition (a) — valid API key, but sudo mode NOT active

**Direct answer `[OBSERVED]`:** **HTTP `440`** with body **`{"error":"Need sudo"}`** (exactly 22
bytes). The reason phrase is emitted as `UNKNOWN` because `440` has no registered phrase. No
account-deletion occurs: the disposable user still exists and **zero** `delete-account` jobs are
scheduled, both before and after.

The key's `sudo_mode_at` is asserted `NULL` immediately before the request, the full response is
captured, and the user + `delete-account` job count are read back after `[OBSERVED]`:

```text
########## Q2(a) — API key, sudo NOT active ##########
### BEFORE: assert api_key.sudo_mode_at IS NULL for the key being used
$ psql -tAc "select code,sudo_mode_at from api_key where code='q2disposkey';"
q2disposkey | sudo_mode_at=NULL
### BEFORE: disposable user present + delete-account jobs for this user
user_exists=1
delete_account_jobs=0

### REQUEST: DELETE /api/user with valid API key (no active sudo)
$ curl -sS -i -X DELETE http://localhost:7777/api/user -H 'Authentication: q2disposkey'
HTTP/1.1 440 UNKNOWN
Server: gunicorn/20.0.4
Date: Tue, 14 Jul 2026 21:35:28 GMT
Connection: close
Content-Type: application/json
Content-Length: 22
Access-Control-Allow-Origin: *
Set-Cookie: slapp=5e412337-4ccd-4cd7-bbe8-a363b5819919.D6wuF6bYHXZnvQJePIVM5E3w1VI; Expires=Tue, 21-Jul-2026 21:35:28 GMT; HttpOnly; Path=/; SameSite=Lax

{"error":"Need sudo"}


### AFTER: user still present? delete-account job created?
user_exists=1
delete_account_jobs=0
```

**Causal explanation.** `require_api_sudo` (`app/api/base.py:L63-L73`) calls
`check_sudo_mode_is_active(g.api_key)` (guard at `app/api/base.py:L69`); with a valid key whose
`sudo_mode_at` is `NULL`, the check is falsy, so the decorator returns
`jsonify(error="Need sudo"), 440` (`app/api/base.py:L70`) and the wrapped `delete_user` body
never runs — hence no `delete-account` job (`app/api/views/user.py:L26-L31`).

**Attribution for the non-standard `440` `[INFERRED]`.** HTTP `440` is **not** part of the
standard HTTP status-code set (it is absent from RFC 7231 and the IANA HTTP Status Code Registry).
It is a proprietary code originating in **Microsoft Internet Information Services (IIS)**,
conventionally named **"Login Time-out"**, used to signal an expired authenticated session that
requires re-login. SimpleLogin **repurposes** this non-standard code to mean "sudo re-authentication
required." Sources consulted: http.dev/440; sitechecker.pro/what-is-440-status-code; and
seoleaders.co.uk/http-status-code-440-login-time-out — all describe `440` as an unofficial,
IIS-specific "Login Time-out" code. This framing is source/reference-derived; the `440` status
itself and the `{"error":"Need sudo"}` body are `[OBSERVED]` above. The observed reason phrase
`UNKNOWN` corroborates that the underlying WSGI stack has no registered phrase for `440`.

### Condition (b) — browser session only (the edge case), no `Authentication` header

**Direct answer `[OBSERVED]`:** **HTTP `500`** with body **`{"error":"Internal error"}`** (exactly
27 bytes). This is **not** a clean `440`: on the session-only path the sudo guard raises an
unhandled `AttributeError` server-side. No account-deletion occurs (user still present, zero
`delete-account` jobs after).

Full response **and** the complete server-side traceback for this exact request `[OBSERVED]`:

```text
########## Q2(b) — browser session only (edge case) ##########
### Login as disposable user q2victim to get a browser session
logged in (session cookie stored)
### BEFORE: user present, delete-account jobs
user_exists=1
delete_account_jobs=0
error-log lines before: 5

### REQUEST: DELETE /api/user with session cookie only (NO Authentication header)
$ curl -sS -i -b $CJ -X DELETE http://localhost:7777/api/user
HTTP/1.1 500 INTERNAL SERVER ERROR
Server: gunicorn/20.0.4
Date: Tue, 14 Jul 2026 21:35:47 GMT
Connection: close
Content-Type: application/json
Content-Length: 27
Access-Control-Allow-Origin: *
Set-Cookie: slapp=4273396b-6315-4779-be3e-641cf387a4cb.FkNsD6696Vcm3LejmXgX_fRMWLs; Expires=Tue, 21-Jul-2026 21:35:47 GMT; HttpOnly; Path=/; SameSite=Lax

{"error":"Internal error"}


### AFTER: user still present? delete-account job created?
user_exists=1
delete_account_jobs=0

### SERVER-SIDE traceback (new gunicorn error-log lines from this request):
2026-07-14 21:35:47,206 - SL - ERROR - 2481 - "/app/server.py:390" - error_handler() -  - 'NoneType' object has no attribute 'sudo_mode_at'
Traceback (most recent call last):
  File "/app/venv/lib/python3.10/site-packages/flask/app.py", line 1950, in full_dispatch_request
    rv = self.dispatch_request()
  File "/app/venv/lib/python3.10/site-packages/flask/app.py", line 1936, in dispatch_request
    return self.view_functions[rule.endpoint](**req.view_args)
  File "/app/app/api/base.py", line 69, in decorated
    if not check_sudo_mode_is_active(g.api_key):
  File "/app/app/api/base.py", line 47, in check_sudo_mode_is_active
    return api_key.sudo_mode_at and g.api_key.sudo_mode_at >= arrow.now().shift(
AttributeError: 'NoneType' object has no attribute 'sudo_mode_at'
```

**Causal explanation.** On a session-only request no API key is present, so `g.api_key` is `None`
(the session-fallback branch of `authorize_request` never assigns it). `require_api_sudo` still
calls `check_sudo_mode_is_active(g.api_key)` (`app/api/base.py:L69`), and inside that function
`api_key.sudo_mode_at` is dereferenced (`app/api/base.py:L47`) on a `None` object, raising
`AttributeError: 'NoneType' object has no attribute 'sudo_mode_at'` — visible verbatim in the
traceback above. Flask's global error handler logs it (`LOG.e(e)`, `server.py:L390`) and, because
the path starts with `/api/`, returns the JSON `{"error":"Internal error"}, 500`
(`server.py:L391-L392`). The `440 "Need sudo"` return is never reached on this edge path.

---

## Q3 — Stored session data format (byte-exact)

**Direct answer `[OBSERVED]`:** an authenticated session is stored in Redis under the key
**`session:<uuid>`** (here `session:c97ef454-d292-4c02-b049-47490d5ae6c5`). Its value is a
**Python `pickle` stream** whose leading two bytes are `\x80\x04` — the pickle `PROTO` opcode
selecting **protocol 4** `[INFERRED reading of the opcode]`. The stored value is **300 bytes** and
deserializes to a **`dict` with exactly six keys**:

| key | value (byte-exact) |
|-----|--------------------|
| `_permanent` | `True` |
| `_fresh` | `True` |
| `csrf_token` | `'3a59489a17f600750143a4504457ac772d69375f'` |
| `_user_id` | `'277992d1-b91a-44bc-8bbf-a4d555d05e2e'` (john's `users.alternative_id`) |
| `_id` | `'27c186…c7f052'` (128 hex chars — a SHA-512 Flask-Login client fingerprint) |
| `sudo_time` | `1784065207` |

**Two distinct UUIDs — do not conflate them.** The **session-store key** UUID
(`c97ef454-…-47490d5ae6c5`) is the random session identifier; the **`_user_id`** value
(`277992d1-…-a4d555d05e2e`) is john's `users.alternative_id`. They are different UUIDs.

**How the key was discovered (acquisition chain).** The session-store UUID is exactly the value
carried in the `slapp` cookie, recovered by unsigning that cookie with the application's own
`itsdangerous.Signer(salt="session", key_derivation="hmac")`. That end-to-end
login → cookie-jar → signer-unsign chain is the Q4 capture below (see `SESSION_UUID_FOR_Q3=…` in
the Q4 decode); the same UUID keys the Redis record read here.

Raw value read **directly from Redis** (not through the app's deserialization), then the
byte-exact value + key enumeration via `redis-py` + `pickle` `[OBSERVED]`:

```text
########## Q3 EVIDENCE — stored session data format (byte-exact) ##########
### The authenticated session uuid (from Q4 decode) = c97ef454-d292-4c02-b049-47490d5ae6c5
### Redis key structure: SESSION_PREFIX ':' <uuid>  ->  session:c97ef454-d292-4c02-b049-47490d5ae6c5

### (1) redis-cli --no-raw GET against the canonical store (sl-redis6):
$ redis-cli -h sl-redis6 --no-raw GET "session:c97ef454-d292-4c02-b049-47490d5ae6c5"
"\x80\x04\x95!\x01\x00\x00\x00\x00\x00\x00}\x94(\x8c\n_permanent\x94\x88\x8c\x06_fresh\x94\x88\x8c\ncsrf_token\x94\x8c(3a59489a17f600750143a4504457ac772d69375f\x94\x8c\b_user_id\x94\x8c$277992d1-b91a-44bc-8bbf-a4d555d05e2e\x94\x8c\x03_id\x94\x8c\x8027c186e703c3b903e2e3fabda7c6ece370b3c884403ddc0afa46497ad655dac94205fc64dcee5a73311b4c0aa898489b18c4d05fe67c23fc941a9d87a2c7f052\x94\x8c\tsudo_time\x94J\xb7\xacVju."

### (2) byte-exact value via redis-py + pickle enumeration (connecting to redis://sl-redis6):
key: session:c97ef454-d292-4c02-b049-47490d5ae6c5
length: 300
repr (byte-exact, BEFORE any deserialization):
b'\x80\x04\x95!\x01\x00\x00\x00\x00\x00\x00}\x94(\x8c\n_permanent\x94\x88\x8c\x06_fresh\x94\x88\x8c\ncsrf_token\x94\x8c(3a59489a17f600750143a4504457ac772d69375f\x94\x8c\x08_user_id\x94\x8c$277992d1-b91a-44bc-8bbf-a4d555d05e2e\x94\x8c\x03_id\x94\x8c\x8027c186e703c3b903e2e3fabda7c6ece370b3c884403ddc0afa46497ad655dac94205fc64dcee5a73311b4c0aa898489b18c4d05fe67c23fc941a9d87a2c7f052\x94\x8c\tsudo_time\x94J\xb7\xacVju.'
--- first two bytes (pickle PROTO opcode + protocol number): b'\x80\x04'
deserialized type: dict
keys (sorted): ['_fresh', '_id', '_permanent', '_user_id', 'csrf_token', 'sudo_time']
  _fresh       = True
  _id          = '27c186e703c3b903e2e3fabda7c6ece370b3c884403ddc0afa46497ad655dac94205fc64dcee5a73311b4c0aa898489b18c4d05fe67c23fc941a9d87a2c7f052'
  _permanent   = True
  _user_id     = '277992d1-b91a-44bc-8bbf-a4d555d05e2e'
  csrf_token   = '3a59489a17f600750143a4504457ac772d69375f'
  sudo_time    = 1784065207
```

> **Renderer note (byte-identical).** `redis-cli --no-raw` renders byte `0x08` as `\b` (e.g.
> `\x8c\b_user_id`), whereas the Python `repr` renders the same byte as `\x08` (e.g.
> `\x8c\x08_user_id`). Both are the single byte `0x08` (a pickle `SHORT_BINUNICODE` length prefix);
> the two renderings describe identical bytes.

**Causal explanation.** The custom store is `RedisSessionStore` in `app/session.py`. The key is
built as `f"session:{id}"` (`_get_key`, `app/session.py:L43-L45`) from `SESSION_PREFIX="session"`
(`app/session.py:L18`). On save, the value is produced by `pickle.dumps(dict(session))`
(`save_session`, `app/session.py:L91`) — hence the pickle stream and its `\x80\x04` protocol-4
header. `csrf_token` is present because Flask-WTF stores the CSRF token in the session;
`_user_id`/`_fresh`/`_id` are written by Flask-Login; `sudo_time` is set by SimpleLogin when sudo
mode is activated.

---

## Q4 — Session identifier behavior across login (before/after)

**Direct answer `[OBSERVED]`:** the session identifier **does not change** across a successful
login. Captured from **one** client (one cookie jar), the `slapp` cookie is **byte-identical**
before and after login, and its decoded UUID is identical:

- BEFORE: `c97ef454-d292-4c02-b049-47490d5ae6c5.pfXr9mCTu-cCc8M785X251ciEyM`
- AFTER:  `c97ef454-d292-4c02-b049-47490d5ae6c5.pfXr9mCTu-cCc8M785X251ciEyM`
- decoded UUID identical? **True**

Literal before/login/after cookie-jar output from a single client `[OBSERVED]`:

```text
########## Q4 EVIDENCE — session id across login (ONE client / one cookie jar) ##########
### BEFORE login: unauthenticated GET /auth/login to obtain initial slapp cookie
$ curl -s -c $CJ http://localhost:7777/auth/login -o /dev/null ; cat $CJ
----- literal cookie-jar file BEFORE login -----
# Netscape HTTP Cookie File
# https://curl.se/docs/http-cookies.html
# This file was generated by libcurl! Edit at your own risk.

#HttpOnly_localhost	FALSE	/	FALSE	1784670007	slapp	c97ef454-d292-4c02-b049-47490d5ae6c5.pfXr9mCTu-cCc8M785X251ciEyM

### LOGIN on the SAME cookie jar
$ curl -s -b $CJ -c $CJ -i -X POST http://localhost:7777/auth/login --data email=john@wick.com --data password=password --data csrf_token=<csrf>
HTTP/1.1 302 FOUND
Location: http://localhost:7777/dashboard/
----- literal cookie-jar file AFTER login -----
# Netscape HTTP Cookie File
# https://curl.se/docs/http-cookies.html
# This file was generated by libcurl! Edit at your own risk.

#HttpOnly_localhost	FALSE	/	FALSE	1784670007	slapp	c97ef454-d292-4c02-b049-47490d5ae6c5.pfXr9mCTu-cCc8M785X251ciEyM

BEFORE slapp value: c97ef454-d292-4c02-b049-47490d5ae6c5.pfXr9mCTu-cCc8M785X251ciEyM
AFTER  slapp value: c97ef454-d292-4c02-b049-47490d5ae6c5.pfXr9mCTu-cCc8M785X251ciEyM
raw cookie identical? YES
```

Decoding both cookies to their underlying UUID with the app's own signer `[OBSERVED]`:

```text
### Q4 decode via app signer (itsdangerous.Signer salt=session, key_derivation=hmac):
decoded BEFORE uuid: c97ef454-d292-4c02-b049-47490d5ae6c5
decoded AFTER  uuid: c97ef454-d292-4c02-b049-47490d5ae6c5
decoded uuid identical? True
SESSION_UUID_FOR_Q3=c97ef454-d292-4c02-b049-47490d5ae6c5
```

**Causal explanation (precise lifecycle).** New server-side session UUIDs are minted in three
situations, none of which is "a successful login on an already-valid session":

1. **Missing or invalid incoming cookie** — `open_session` mints a fresh UUID
   (`app/session.py:L71`).
2. **Corrupt stored data** — `open_session` also mints a fresh UUID (`app/session.py:L80`).
3. **Logout** — `purge_session` deletes the Redis record and rotates to a new UUID
   (`app/session.py:L64`).

In the observed flow: the initial unauthenticated `GET /auth/login` had no incoming cookie, so
`open_session` minted `c97ef454…` (case 1) and `save_session` stored + signed it (the BEFORE
cookie). On the login `POST` over the **same** jar, the cookie was present and valid, so
`open_session` **reused** that UUID (`app/session.py:L77`); `login_user()` then wrote
`_user_id`/`_fresh`/`_id` **into the existing session dict**; and `save_session` re-stored the
record under the **same** UUID and re-signed the **same** UUID (`app/session.py:L102-L104`).
Because `itsdangerous` signing is a deterministic HMAC over `(uuid, secret)`, re-signing the same
UUID yields the byte-identical cookie observed above.

**Flask-Login 0.5.0 semantics `[INFERRED]`.** At the pinned `flask-login==0.5.0`, `login_user()`
sets `session["_user_id"]`, `session["_fresh"]`, and `session["_id"]` on the **existing** session
dict; it does **not** regenerate the server-side session identifier. (Flask's default session
exposes no `regenerate()`; server-side session-id regeneration — the standard session-fixation
mitigation — is a separate, explicit action a framework/app must take.) The `session["_id"]` that
`login_user` sets is a **client fingerprint** produced by `_create_identifier()` (a SHA-512 of
client IP + User-Agent) for `login_manager.session_protection = "strong"`
(`app/extensions.py:L8`) — it is *not* the Redis session key and does not cause the store key to
rotate. Source consulted: the Flask-Login 0.5.0 documentation/source
(flask-login.readthedocs.io/en/0.5.0), where `login_user` writes the user id/fresh/`_id` keys into
the session without a session-id regeneration step. This is why the observed UUID persists across
login.

---

## Q5 — Email forwarding header survival (each of three headers)

**Direct answers — each named header individually `[OBSERVED]`:**

- **Custom `X-Test-Custom` header → STRIPPED (does not survive).** Absent from the forwarded
  message (`present? False`).
- **`Received` header → STRIPPED (does not survive).** Absent from the forwarded message
  (`present? False`).
- **`Reply-To` header → original value STRIPPED; a *new* `Reply-To` is synthesized.** The inbound
  `Reply-To: someone@elsewhere.test` does **not** survive; the forwarded message instead carries
  `Reply-To: "someone at elsewhere.test" <someone_at_elsewhere_test_kjmac@sl.local>`, a
  SimpleLogin **reverse-alias** address. So a `Reply-To` header is present, but **not with the
  original value**.

In short: the original values of **all three** probe headers are removed by the forward-path
allow-list. `X-Test-Custom` and `Received` have no re-synthesis and are simply gone; `Reply-To` is
re-created pointing at a reverse alias (because the inbound message had one). For contrast, the
`From` header *is* in the allow-list, so it is kept and then rewritten — its rewrite log shows a
non-`None` old value, unlike `Reply-To`.

**How it was exercised (real SMTP traversal — resolves the prior non-canonical attempt).** A
**real `aiosmtpd` `Controller(MailHandler())`** listener was started on `127.0.0.1:20381` — the
canonical inbound entry `MailHandler.handle_DATA` (`email_handler.py:L2289`), which reads
`envelope.original_content` (`email_handler.py:L2290`). A complete RFC822 message carrying the
three probe headers (plus standard headers) was delivered over the wire with `smtplib` (`MAIL
FROM`/`RCPT TO`/`DATA`), to john's seeded alias `operas_enzyme523@sl.local`. The full forwarding
pipeline then ran for real — allow-list strip, `From`/`Reply-To` rewrite — and the resulting
forwarded `SendRequest` was captured at the canonical send boundary. Because `NOT_SEND_EMAIL=true`
in the canonical `.env`, the app does not relay outbound; the fully-processed message is captured
at `MailSender.send()` where the outgoing `SendRequest` is recorded (`app/mail_sender.py:L128-L129`,
which appends the request **before** the `NOT_SEND_EMAIL` branch at `L130`). This is the same
capture point SimpleLogin's own tests use — an observation of the final forwarded message, **not**
a bypass of any forwarding logic.

Complete capture — inbound raw message, the real handler log trail, the SMTP `250` wire responses,
the **complete forwarded header block**, recipient mapping, and per-header verdicts `[OBSERVED]`:

```text
>>> URL: http://localhost:7777
WARNING: Use a temp directory for GNUPGHOME /tmp/byghxhownjqopwyjzpor
Upload files to local dir
>>> init logging <<<
2026-07-14 21:44:53,284 - SL - DEBUG - 2747 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
### inbound raw message (the three probe headers are X-Test-Custom, Received, Reply-To):
From: Outsider <outsider@example.com>
To: operas_enzyme523@sl.local
Subject: Q5 header survival test
Date: Tue, 14 Jul 2026 21:40:00 -0000
Message-ID: <q5-probe-real-0001@example.com>
Reply-To: someone@elsewhere.test
Received: from mail.example.com (mail.example.com [203.0.113.9]) by mx.sl.local; Tue, 14 Jul 2026 21:39:50 -0000
X-Test-Custom: hello-world
MIME-Version: 1.0
Content-Type: text/plain; charset="utf-8"
Content-Transfer-Encoding: 7bit

This is the Q5 probe body.

### REAL aiosmtpd Controller(MailHandler()) started on 127.0.0.1:20381
2026-07-14 21:44:54,007 - SL - DEBUG - 2747 - "/app/app/log.py:24" - set_message_id() -  - set message_id e7520d45-a1a8-4044-ae73-f93e12249f4b
2026-07-14 21:44:54,007 - SL - DEBUG - 2747 - "/app/email_handler.py:2342" - _handle() - e7520d45-a1a8-4044-ae73-f93e12249f4b - ====>=====>====>====>====>====>====>====>
2026-07-14 21:44:54,008 - SL - INFO - 2747 - "/app/email_handler.py:2343" - _handle() - e7520d45-a1a8-4044-ae73-f93e12249f4b - New message, mail from outsider@example.com, rctp tos ['operas_enzyme523@sl.local'] 
2026-07-14 21:44:54,008 - SL - DEBUG - 2747 - "/app/email_handler.py:1963" - handle() - e7520d45-a1a8-4044-ae73-f93e12249f4b - Cannot parse Postfix queue ID from ['from mail.example.com (mail.example.com [203.0.113.9]) by mx.sl.local; Tue, 14 Jul 2026 21:39:50 -0000'] from mail.example.com (mail.example.com [203.0.113.9]) by mx.sl.local; Tue, 14 Jul 2026 21:39:50 -0000
2026-07-14 21:44:54,127 - SL - DEBUG - 2747 - "/app/email_handler.py:1980" - handle() - e7520d45-a1a8-4044-ae73-f93e12249f4b - ==>> Handle mail_from:outsider@example.com, rcpt_tos:['operas_enzyme523@sl.local'], header_from:Outsider <outsider@example.com>, header_to:operas_enzyme523@sl.local, cc:None, reply-to:someone@elsewhere.test, message_id:<q5-probe-real-0001@example.com>, client_ip:None, headers:[('From', 'Outsider <outsider@example.com>'), ('To', 'operas_enzyme523@sl.local'), ('Subject', 'Q5 header survival test'), ('Date', 'Tue, 14 Jul 2026 21:40:00 -0000'), ('Message-ID', '<q5-probe-real-0001@example.com>'), ('Reply-To', 'someone@elsewhere.test'), ('Received', 'from mail.example.com (mail.example.com [203.0.113.9]) by mx.sl.local; Tue, 14 Jul 2026 21:39:50 -0000'), ('X-Test-Custom', 'hello-world'), ('MIME-Version', '1.0'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-14 21:44:54,131 - SL - DEBUG - 2747 - "/app/email_handler.py:2202" - handle() - e7520d45-a1a8-4044-ae73-f93e12249f4b - Forward phase outsider@example.com(Outsider <outsider@example.com>) -> operas_enzyme523@sl.local
2026-07-14 21:44:54,146 - SL - DEBUG - 2747 - "/app/email_handler.py:580" - handle_forward() - e7520d45-a1a8-4044-ae73-f93e12249f4b - Create or get contact for from_header:Outsider <outsider@example.com>
2026-07-14 21:44:54,171 - SL - DEBUG - 2747 - "/app/app/contact_utils.py:110" - create_contact() - e7520d45-a1a8-4044-ae73-f93e12249f4b - Created contact <Contact 2 outsider@example.com 2> for alias <Alias 2 operas_enzyme523@sl.local> with email outsider@example.com invalid_email=False
2026-07-14 21:44:54,171 - SL - DEBUG - 2747 - "/app/email_handler.py:589" - handle_forward() - e7520d45-a1a8-4044-ae73-f93e12249f4b - Create or get contact for reply_to_header:someone@elsewhere.test
2026-07-14 21:44:54,191 - SL - DEBUG - 2747 - "/app/app/contact_utils.py:110" - create_contact() - e7520d45-a1a8-4044-ae73-f93e12249f4b - Created contact <Contact 3 someone@elsewhere.test 2> for alias <Alias 2 operas_enzyme523@sl.local> with email someone@elsewhere.test invalid_email=False
2026-07-14 21:44:54,192 - SL - INFO - 2747 - "/app/app/handler/dmarc.py:33" - apply_dmarc_policy_for_forward_phase() - e7520d45-a1a8-4044-ae73-f93e12249f4b - DMARC check disabled
2026-07-14 21:44:54,199 - SL - DEBUG - 2747 - "/app/email_handler.py:688" - forward_email_to_mailbox() - e7520d45-a1a8-4044-ae73-f93e12249f4b - Forward <Contact 2 outsider@example.com 2> -> <Alias 2 operas_enzyme523@sl.local> -> <Mailbox 1 john@wick.com>
2026-07-14 21:44:54,202 - SL - DEBUG - 2747 - "/app/email_handler.py:740" - forward_email_to_mailbox() - e7520d45-a1a8-4044-ae73-f93e12249f4b - Create <EmailLog 2> for <Contact 2 outsider@example.com 2>, <User 1 John Wick john@wick.com>, <Mailbox 1 john@wick.com>
2026-07-14 21:44:54,207 - SL - DEBUG - 2747 - "/app/email_handler.py:867" - forward_email_to_mailbox() - e7520d45-a1a8-4044-ae73-f93e12249f4b - From header, new:"Outsider - outsider at example.com" <outsider_at_example_com_ydmks@sl.local>, old:Outsider <outsider@example.com>
2026-07-14 21:44:54,208 - SL - DEBUG - 2747 - "/app/email_handler.py:873" - forward_email_to_mailbox() - e7520d45-a1a8-4044-ae73-f93e12249f4b - Reply-To header, new:"someone at elsewhere.test" <someone_at_elsewhere_test_kjmac@sl.local>, old:None
2026-07-14 21:44:54,208 - SL - DEBUG - 2747 - "/app/email_handler.py:316" - replace_header_when_forward() - e7520d45-a1a8-4044-ae73-f93e12249f4b - Delete Cc header, old value None
2026-07-14 21:44:54,209 - SL - DEBUG - 2747 - "/app/email_handler.py:313" - replace_header_when_forward() - e7520d45-a1a8-4044-ae73-f93e12249f4b - Replace To header, old: operas_enzyme523@sl.local, new: operas_enzyme523@sl.local
2026-07-14 21:44:54,209 - SL - INFO - 2747 - "/app/app/handler/unsubscribe_generator.py:36" - _generate_header_with_original_behaviour() - e7520d45-a1a8-4044-ae73-f93e12249f4b - Email has no unsubscribe header
2026-07-14 21:44:54,211 - SL - DEBUG - 2747 - "/app/email_handler.py:893" - forward_email_to_mailbox() - e7520d45-a1a8-4044-ae73-f93e12249f4b - Forward mail from outsider@example.com to john@wick.com, mail_options:[], rcpt_options:[] 
2026-07-14 21:44:54,211 - SL - DEBUG - 2747 - "/app/app/mail_sender.py:131" - send() - e7520d45-a1a8-4044-ae73-f93e12249f4b - send email with subject 'Q5 header survival test', from '"Outsider - outsider at example.com" <outsider_at_example_com_ydmks@sl.local>' to 'operas_enzyme523@sl.local'
2026-07-14 21:44:54,212 - SL - INFO - 2747 - "/app/email_handler.py:2367" - _handle() - e7520d45-a1a8-4044-ae73-f93e12249f4b - Finish mail_from outsider@example.com, rcpt_tos ['operas_enzyme523@sl.local'], takes 0.20447707176208496 seconds with return code '250 Message accepted for delivery'<<===
### SMTP wire responses: MAIL FROM=250  RCPT TO=250  DATA=250 Message accepted for delivery
### forwarded messages captured (real outgoing SendRequest): 1
### recipient mapping: envelope_from='sl.lmycyibsfqqdemzygq2tanc5.kr27mqmxkifk4@sl.local' envelope_to='john@wick.com' is_forward=True
### ===== COMPLETE FORWARDED HEADER BLOCK =====
Subject: Q5 header survival test
Date: Tue, 14 Jul 2026 21:40:00 -0000
Message-ID: <q5-probe-real-0001@example.com>
MIME-Version: 1.0
Content-Type: text/plain; charset="utf-8"
Content-Transfer-Encoding: 7bit
X-SimpleLogin-Type: Forward
X-SimpleLogin-EmailLog-ID: 2
X-SimpleLogin-Envelope-From: outsider@example.com
X-SimpleLogin-Original-From: Outsider <outsider@example.com>
X-SimpleLogin-Envelope-To: operas_enzyme523@sl.local
From: "Outsider - outsider at example.com" <outsider_at_example_com_ydmks@sl.local>
Reply-To: "someone at elsewhere.test" <someone_at_elsewhere_test_kjmac@sl.local>
To: operas_enzyme523@sl.local
DKIM-Signature: v=1; a=rsa-sha256; c=relaxed/simple; d=sl.local;  i=@sl.local; q=dns/txt; s=dkim; t=1784065494; h=message-id : date :  subject : from : to; bh=RaMFh1AWOD3bBu11XnCnTgCjXprFT7zGVvimYB2hXpc=;  b=fXLWpzzcsVqiNpYNgp5eQ+G34L6iSkgaPf5ISWt1IKt+21g2LFliJpfklxCcbPdWaPbWo  UAkbzoPI+9AW+oJfErfK25UYzGkgHY1tiI8QI84eIqWpoi+tJwEY1STI7LCic39h838Oww9  6FC6TI++uRVl/YIC4m10UIDJV6qdsxc= 
### ===== PER-HEADER VERDICTS =====
X-Test-Custom present? False (value=None)
Received      present? False (value=None)
Reply-To value in forwarded msg: '"someone at elsewhere.test" <someone_at_elsewhere_test_kjmac@sl.local>'

### ===== SOURCE ANCHOR CONFIRMATION (verified against running source) =====
# email_handler.py:
#   L679  def forward_email_to_mailbox(
#   L793-807  headers_to_keep = [FROM, TO, CC, SUBJECT, DATE, MESSAGE_ID, REFERENCES,
#                                 IN_REPLY_TO, SL_QUEUE_ID, LIST_UNSUBSCRIBE,
#                                 LIST_UNSUBSCRIBE_POST] + headers.MIME_HEADERS
#   L808-809  if user.include_header_email_header: append AUTHENTICATION_RESULTS
#   L810  delete_all_headers_except(msg, headers_to_keep)   <-- allow-list strip
#   L864  old_from_header = msg[headers.FROM]
#   L866  add_or_replace_header(msg, "From", new_from_header)
#   L867  LOG.d("From header, new:%s, old:%s", ...)          <-- old NOT None (FROM kept)
#   L869  if reply_to_contact:
#   L870  reply_to_header = msg[headers.REPLY_TO]            <-- reads AFTER strip -> None
#   L872  add_or_replace_header(msg, "Reply-To", new_reply_to_header)
#   L873  LOG.d("Reply-To header, new:%s, old:%s", ...)      <-- old:None (REPLY_TO stripped)
#   L2289 async def handle_DATA(self, server, session, envelope: Envelope):
#   L2290 msg = email.message_from_bytes(envelope.original_content)  <-- real SMTP entry
# app/email/headers.py: REPLY_TO="Reply-To" L13, RECEIVED="Received" L14 (NEITHER in allow-list);
#   MIME_HEADERS L44-51 = Mime-Version/Content-Type/Content-Disposition/Content-Transfer-Encoding only
# app/mail_sender.py: send() L126; capture at L128-129 (append BEFORE NOT_SEND_EMAIL branch L130);
#   store_emails_instead_of_sending() L101-102 -> canonical send-boundary interception (not a bypass)
```

**Reading the evidence.**

- **SMTP wire:** `MAIL FROM=250  RCPT TO=250  DATA=250 Message accepted for delivery` — the message
  really traversed the SMTP handler.
- **Recipient mapping:** `envelope_from='sl.…@sl.local'` (a SimpleLogin reverse-alias VERP bounce
  address), `envelope_to='john@wick.com'` (the alias's destination mailbox), `is_forward=True`.
- **Per-header verdicts (from the captured forwarded message):** `X-Test-Custom present? False`,
  `Received present? False`, and `Reply-To` present with the **reverse-alias** value, not
  `someone@elsewhere.test`.
- **The decisive log lines:** `email_handler.py:L867` logs the `From` rewrite with
  `old:Outsider <outsider@example.com>` (non-`None`, because `From` was **kept** by the allow-list),
  whereas `email_handler.py:L873` logs the `Reply-To` rewrite with **`old:None`** — proving the
  original `Reply-To` had already been **stripped** by the allow-list before the new one was added.

**Causal explanation.** Forwarding is performed by `forward_email_to_mailbox`
(`email_handler.py:L679`). Before delivery it applies an **allow-list**: `headers_to_keep`
(`email_handler.py:L793-L807`) = `[FROM, TO, CC, SUBJECT, DATE, MESSAGE_ID, REFERENCES,
IN_REPLY_TO, SL_QUEUE_ID, LIST_UNSUBSCRIBE, LIST_UNSUBSCRIBE_POST]` plus `headers.MIME_HEADERS`,
then `delete_all_headers_except(msg, headers_to_keep)` (`email_handler.py:L810`) deletes every
header not on that list (case-insensitive). In `app/email/headers.py`, `REPLY_TO = "Reply-To"`
(`L13`) and `RECEIVED = "Received"` (`L14`) are **neither** on the allow-list, and `MIME_HEADERS`
(`L44-L51`) covers only `Mime-Version`/`Content-Type`/`Content-Disposition`/
`Content-Transfer-Encoding` — so the custom `X-Test-Custom`, the `Received`, and the original
`Reply-To` are all deleted. The pipeline then rewrites sender-facing headers: `From` is re-added
as the reverse-alias sender (`email_handler.py:L864-L867`); and **because the inbound message had
a `Reply-To`**, a reverse-alias contact is created and a **new** `Reply-To` is set — the code reads
`msg[headers.REPLY_TO]` at `email_handler.py:L870` (already `None` post-strip) and adds the new
value, logging it at `email_handler.py:L873`. `X-Test-Custom` and `Received` match no rewrite rule
and therefore never reappear. (The added `DKIM-Signature` signs `h=message-id:date:subject:from:to`
— note `Reply-To` is not among the signed headers.)

---

## Q6 — Alias-creation token expiry window (two independent real-form runs)

**Direct answer `[OBSERVED]`:** the alias-creation token (the signed alias suffix) is valid for
**600 seconds**. Two independent real-form runs both showed: a token used at **age 0s** and at
**age 597s** creates the alias (**success**), while the **same** token used at **age 607s** is
rejected as expired. Runtime observation therefore brackets the window to **`(597s, 607s]`**; the
boundary is exactly **600 seconds** `[INFERRED — from signer.unsign(max_age=600), tightly
bracketed by the observation above]`.

**How it was exercised (two independent canonical runs — resolves the prior synthetic attempt).**
Two independent sequences (RUN A and RUN B), each: authenticated login → `GET
/dashboard/custom_alias` → **extract the server-minted `signed-alias-suffix` and CSRF token from
the page HTML** (no `signer.sign`, no hand-forged token) → **immediate** `POST` to
`/dashboard/custom_alias` (success) → **real wall-clock wait** → **near-boundary** `POST` at ~597s
(success) → wait → **expired** `POST` at ~607s (rejected). The waits are real elapsed seconds
(`date +%s` polling), and the whole capture ran ~10 minutes (start `21:32:32Z` → done
`21:42:40Z`). The complete capture below includes the script provenance, so the token source and
timing are auditable `[OBSERVED]`:

```text
################################################################
# Q6 — ALIAS-CREATION TOKEN EXPIRY WINDOW (two independent REAL-FORM runs)
# CANONICAL METHOD (resolves F2): tokens minted BY THE REAL FORM (extracted
# from GET /dashboard/custom_alias page HTML), submitted via REAL POST to
# /dashboard/custom_alias; REAL wall-clock waits (no signer.sign, no clock skew).
################################################################

===== CAPTURE SCRIPT (provenance) — /tmp/q6_bg.sh =====
#!/bin/bash
BASE=http://localhost:7777
LOGF=/tmp/q_evidence/q6_background.log
: > "$LOGF"

run_token() {
  local label=$1
  local CJ=/tmp/q6_${label}_cj.txt; rm -f "$CJ"
  local html csrf page fcsrf suffix
  html=$(curl -s -c "$CJ" $BASE/auth/login)
  csrf=$(echo "$html" | grep -oP 'name="csrf_token"[^>]*value="\K[^"]+' | head -1)
  curl -s -b "$CJ" -c "$CJ" -o /dev/null -X POST $BASE/auth/login \
    --data-urlencode "email=john@wick.com" --data-urlencode "password=password" \
    --data-urlencode "csrf_token=$csrf"
  page=$(curl -s -b "$CJ" "$BASE/dashboard/custom_alias")
  fcsrf=$(echo "$page" | grep -oP 'name="csrf_token"[^>]*value="\K[^"]+' | head -1)
  suffix=$(echo "$page" | grep -oE 'value="\.[a-z0-9]+@sl\.local\.[^"]+"' | head -1 | sed 's/^value="//; s/"$//')
  echo "$CJ"    > /tmp/q6_${label}_cjpath
  echo "$fcsrf" > /tmp/q6_${label}_csrf
  echo "$suffix"> /tmp/q6_${label}_suffix
  date +%s      > /tmp/q6_${label}_t0
  echo "[RUN $label] minted signed-alias-suffix=$suffix  (t0=$(cat /tmp/q6_${label}_t0))"
}

post_token() {
  local label=$1 prefix=$2 desc=$3
  local CJ=$(cat /tmp/q6_${label}_cjpath)
  local fcsrf=$(cat /tmp/q6_${label}_csrf)
  local suffix=$(cat /tmp/q6_${label}_suffix)
  local t0=$(cat /tmp/q6_${label}_t0)
  local now age resp loc flash
  now=$(date +%s); age=$((now - t0))
  echo "----- [RUN $label] $desc : measured token age = ${age}s (prefix=$prefix) -----"
  resp=$(curl -s -b "$CJ" -c "$CJ" -i -X POST "$BASE/dashboard/custom_alias" \
     --data-urlencode "prefix=$prefix" --data-urlencode "signed-alias-suffix=$suffix" \
     --data-urlencode "mailboxes=1" --data-urlencode "note=q6-$desc" \
     --data-urlencode "csrf_token=$fcsrf")
  echo "$resp" | grep -iE '^(HTTP/|Location:)'
  loc=$(echo "$resp" | grep -i '^Location:' | sed 's/^[Ll]ocation: //; s/\r//')
  if echo "$loc" | grep -q 'custom_alias'; then
    flash=$(curl -s -b "$CJ" "$loc" | grep -oE 'toastr\.(warning|error|success)\("[^"]*"\)' | head -3)
    echo "rendered flash on redirect target: $flash"
  fi
}

wait_until_age() { # $1 target age based on run A t0
  local target=$1 t0 now age
  t0=$(cat /tmp/q6_A_t0)
  while :; do now=$(date +%s); age=$((now - t0)); [ "$age" -ge "$target" ] && break; sleep 1; done
}

{
echo "=== Q6 canonical two-run form boundary capture — start $(date -u) ==="
run_token A
run_token B
echo ">>> IMMEDIATE posts (expect success 302 highlight_alias_id):"
post_token A q6aimm immediate
post_token B q6bimm immediate
echo ">>> NEAR-BOUNDARY posts (~597s, expect still valid -> success):"
wait_until_age 597
post_token A q6anear near-boundary
post_token B q6bnear near-boundary
echo ">>> EXPIRED posts (~607s, expect expired -> back to form + warning flash):"
wait_until_age 607
post_token A q6aexp expired
post_token B q6bexp expired
echo "=== Q6 canonical two-run form boundary capture — done $(date -u) ==="
} >> "$LOGF" 2>&1

===== COMPLETE CAPTURE LOG — /tmp/q_evidence/q6_background.log =====
=== Q6 canonical two-run form boundary capture — start Tue Jul 14 21:32:32 UTC 2026 ===
[RUN A] minted signed-alias-suffix=.milieu915@sl.local.alaq8Q.kOr7jZ7eLmz9uPyh0AVSXNCfdU4  (t0=1784064753)
[RUN B] minted signed-alias-suffix=.dryads032@sl.local.alaq8Q.4sSidcfheHvNYTeHBS6LvFpMZ-I  (t0=1784064753)
>>> IMMEDIATE posts (expect success 302 highlight_alias_id):
----- [RUN A] immediate : measured token age = 0s (prefix=q6aimm) -----
HTTP/1.1 302 FOUND
Location: http://localhost:7777/dashboard/?highlight_alias_id=13
----- [RUN B] immediate : measured token age = 0s (prefix=q6bimm) -----
HTTP/1.1 302 FOUND
Location: http://localhost:7777/dashboard/?highlight_alias_id=14
>>> NEAR-BOUNDARY posts (~597s, expect still valid -> success):
----- [RUN A] near-boundary : measured token age = 597s (prefix=q6anear) -----
HTTP/1.1 302 FOUND
Location: http://localhost:7777/dashboard/?highlight_alias_id=16
----- [RUN B] near-boundary : measured token age = 597s (prefix=q6bnear) -----
HTTP/1.1 302 FOUND
Location: http://localhost:7777/dashboard/?highlight_alias_id=17
>>> EXPIRED posts (~607s, expect expired -> back to form + warning flash):
----- [RUN A] expired : measured token age = 607s (prefix=q6aexp) -----
HTTP/1.1 302 FOUND
Location: http://localhost:7777/dashboard/custom_alias
rendered flash on redirect target: toastr.success("Alias q6aimm.milieu915@sl.local has been created")
toastr.success("Alias q6anear.milieu915@sl.local has been created")
toastr.warning("Alias creation time is expired, please retry")
----- [RUN B] expired : measured token age = 607s (prefix=q6bexp) -----
HTTP/1.1 302 FOUND
Location: http://localhost:7777/dashboard/custom_alias
rendered flash on redirect target: toastr.success("Alias q6bimm.dryads032@sl.local has been created")
toastr.success("Alias q6bnear.dryads032@sl.local has been created")
toastr.warning("Alias creation time is expired, please retry")
=== Q6 canonical two-run form boundary capture — done Tue Jul 14 21:42:40 UTC 2026 ===

===== SOURCE ANCHOR CONFIRMATION (verified against running source) =====
# app/alias_suffix.py:
#   L11  signer = itsdangerous.TimestampSigner(config.CUSTOM_ALIAS_SECRET)
#   L37  def check_suffix_signature(signed_suffix): ...
#   L38  # hypothesis: user will click on the button in the 600 secs
#   L40  return signer.unsign(signed_suffix, max_age=600).decode()   <-- 600s window
#   L41-42  except itsdangerous.BadSignature: return None
# app/dashboard/views/custom_alias.py:
#   L90  suffix = check_suffix_signature(signed_alias_suffix)
#   L91  if not suffix:
#   L92  LOG.w("Alias creation time expired for %s", current_user)
#   L93  flash("Alias creation time is expired, please retry", "warning")  <-- matches observed toastr.warning
#   L94  return redirect(request.url)   <-- 302 back to /dashboard/custom_alias

===== OBSERVED BOUNDARY (both runs A & B identical) =====
#   age =   0s  -> VALID   (302 -> /dashboard/?highlight_alias_id=13 [A], =14 [B])
#   age = 597s  -> VALID   (302 -> /dashboard/?highlight_alias_id=16 [A], =17 [B])
#   age = 607s  -> EXPIRED (302 -> /dashboard/custom_alias + toastr.warning 'Alias creation time is expired, please retry')
# => runtime observation brackets the window to (597s, 607s]; exact 600s per signer.unsign(max_age=600) at alias_suffix.py:L40
# itsdangerous semantics: age>max_age raises SignatureExpired(BadSignature) -> valid at <=600s, expired at >600s
```

**Reading the evidence (per-stage verdicts).**

- **Immediate (age 0s) → SUCCESS:** RUN A `302 → /dashboard/?highlight_alias_id=13`, RUN B
  `…=14`. A `highlight_alias_id` redirect means the alias was created.
- **Near-boundary (age 597s) → STILL VALID:** RUN A `302 → …highlight_alias_id=16`, RUN B `…=17`.
- **Expired (age 607s) → REJECTED:** both runs `302 → /dashboard/custom_alias` (back to the form)
  and the redirect target renders `toastr.warning("Alias creation time is expired, please retry")`.

> **Transparent note on the accumulated success flashes.** In the expired-stage output the
> redirect-target page shows *three* flashes: the two earlier successes
> (`"Alias q6aimm.… has been created"`, `"Alias q6anear.… has been created"`) **and** the expiry
> warning. This is expected: the immediate and near-boundary `POST`s were issued header-only
> (`curl -i`, without following the redirect), so their success flashes stayed queued in the
> session until the expired-stage follow-up `GET` fetched the redirect target and consumed all
> pending flashes at once. The unambiguous Q6 signal is the **`Location`** of each `POST`
> (`/dashboard/?highlight_alias_id=N` for the two successes vs `/dashboard/custom_alias` for the
> expired case) together with the presence of the **expiry warning** flash — both consistent
> across the two independent runs.

**Causal explanation.** The signed suffix is verified by `check_suffix_signature`
(`app/alias_suffix.py:L37-L42`), which calls `signer.unsign(signed_suffix, max_age=600)`
(`app/alias_suffix.py:L40`) on a module-level `itsdangerous.TimestampSigner(config.CUSTOM_ALIAS_SECRET)`
(`app/alias_suffix.py:L11`). `TimestampSigner` embeds a creation timestamp in the token; when the
token's age exceeds `max_age` it raises `SignatureExpired` (a subclass of `BadSignature`), which
the function catches to `return None` (`app/alias_suffix.py:L41-L42`). The view
`custom_alias` calls `suffix = check_suffix_signature(signed_alias_suffix)`
(`app/dashboard/views/custom_alias.py:L90`); on `None` it logs a warning
(`app/dashboard/views/custom_alias.py:L92`), flashes
`"Alias creation time is expired, please retry"`
(`app/dashboard/views/custom_alias.py:L93`) — matching the observed `toastr.warning` — and
redirects back to the form (`app/dashboard/views/custom_alias.py:L94`). Because `max_age=600`,
tokens are accepted at age ≤ 600s and rejected at age > 600s, exactly bracketing the observed
`(597s, 607s]` transition.

---

## Q7 — API key usage statistics (each field, before/after, two batches)

**Direct answer `[OBSERVED]`:** each authenticated **API-key** call updates **both** usage
fields on the `api_key` row:

- **`times`** (`integer`) is **incremented by 1** per call.
- **`last_used`** (`timestamp without time zone`, UTC, microsecond precision) is **set to the
  current server time** (`arrow.now()`).

Across **two** batches of five API-key calls each, on an isolated key (`q7isolatedkey`):

- `times`: **0 → 5 → 10** (deterministic +5, +5).
- `last_used`: **NULL → `2026-07-14 21:37:09.384472` → `2026-07-14 21:37:09.679098`**.

**Contrast — the browser-session path updates *neither* field.** Five session-fallback calls
(cookie, **no** `Authentication` header) leave `times` at `10` and `last_used` unchanged. A third
column, `sudo_mode_at`, exists on the model but is not touched by ordinary authenticated calls.

Before/after rows for both batches, full body of the fifth call, per-call statuses and request
timestamps, DB column types, and the session-path contrast `[OBSERVED]`:

```text
########## Q7 EVIDENCE (isolated key code='q7isolatedkey', endpoint GET /api/user_info) ##########
### DB column types: last_used = timestamp without time zone (UTC, microsecond precision); times = integer
last_used :: timestamp without time zone
times :: integer

### BEFORE batch1  [times|last_used]:
0|NULL
### 5 API-key calls (GET /api/user_info -H 'Authentication: q7isolatedkey'):
  call1: HTTP 200  at 21:37:09.328
  call2: HTTP 200  at 21:37:09.344
  call3: HTTP 200  at 21:37:09.358
  call4: HTTP 200  at 21:37:09.378
  call5: HTTP 200  at 21:37:09.392
### AFTER batch1   [times|last_used]:
5|2026-07-14 21:37:09.384472
### full body of call5 (complete, untruncated):
{"can_create_reverse_alias":true,"connected_proton_address":null,"email":"john@wick.com","in_trial":false,"is_premium":true,"max_alias_free_plan":3,"name":"John Wick","profile_picture_url":"http://localhost:7777/static/upload/profile_pic.svg"}


### BEFORE batch2  [times|last_used]:
5|2026-07-14 21:37:09.384472
### 5 more API-key calls:
  call1: HTTP 200  at 21:37:09.622
  call2: HTTP 200  at 21:37:09.639
  call3: HTTP 200  at 21:37:09.656
  call4: HTTP 200  at 21:37:09.672
  call5: HTTP 200  at 21:37:09.687
### AFTER batch2   [times|last_used]:
10|2026-07-14 21:37:09.679098

### CONTRAST: 5 session-fallback calls (cookie, NO Authentication header) to GET /api/aliases?page_id=0:
### BEFORE session batch [times|last_used]:
10|2026-07-14 21:37:09.679098
  sess-call1: HTTP 200
  sess-call2: HTTP 200
  sess-call3: HTTP 200
  sess-call4: HTTP 200
  sess-call5: HTTP 200
### AFTER session batch  [times|last_used]:
10|2026-07-14 21:37:09.679098
```

The session cookie jar used for the (non-updating) contrast calls `[OBSERVED]`:

```text
# Netscape HTTP Cookie File
# https://curl.se/docs/http-cookies.html
# This file was generated by libcurl! Edit at your own risk.

#HttpOnly_localhost	FALSE	/	FALSE	1784669829	slapp	565fcfb2-a657-4fc2-afc4-797527d93d48.93MWiGTv-QioeKbTdNFkn0st_As
```

> Calls were issued **sequentially** (no concurrency), so counter increments are uncorrupted.
> `last_used` is set **server-side** via `arrow.now()` while the request is processed, so its value
> falls within the client-side call window (e.g. batch-1 `last_used=…384472` sits among the
> batch-1 call timestamps). The +5/+5 counter deltas and the two `last_used` advances are stable
> across the two batches.

**Causal explanation.** In `authorize_request` (`app/api/base.py:L16-L43`), the API-key branch
(taken only when a valid `Authentication` header is present) sets `api_key.last_used = arrow.now()`
(`app/api/base.py:L30`), increments `api_key.times += 1` (`app/api/base.py:L31`), and commits
(`Session.commit()`, `app/api/base.py:L32`). The columns are defined on the `ApiKey` model:
`last_used` (`app/models.py:L2358`), `times` (`app/models.py:L2359`), and `sudo_mode_at`
(`app/models.py:L2360`). The session-fallback branch never reaches this code (no `api_key` is
resolved), which is why the cookie path updates neither field.

---

## Q8 — Failed-login logging and response

**Direct answer `[OBSERVED]`:** a wrong-password login returns **HTTP `200 OK`** (the login page is
**re-rendered** — not a redirect, not a `4xx`), `Content-Type: text/html`, `Content-Length: 7017`,
with the flashed error **`Email or password incorrect`** rendered inline in the body. The
failed attempt produces **no dedicated application log line**: the only log output for the request
is the generic `after_request` request-completion `DEBUG` line (`server.py:L284`) and the gunicorn
access line (`"POST /auth/login HTTP/1.1" 200 7017`). The failed-login `LoginEvent` is delivered
to NewRelic as a custom event `[INFERRED]` (see below); it emits nothing to stdout.

The **complete** response — status line, all headers (session cookie **value redacted**; name and
attributes preserved), and the full `7017`-byte HTML body `[OBSERVED]`:

```text
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Tue, 14 Jul 2026 21:37:55 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 7017
Set-Cookie: slapp=<REDACTED-SESSION-COOKIE>; Expires=Tue, 21-Jul-2026 21:37:55 GMT; HttpOnly; Path=/; SameSite=Lax


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
    <link rel="canonical" href="http://localhost:7777/auth/login" />
    <title>
      Login
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
          

            <script>toastr.error("Email or password incorrect");</script>
          
        
      </div>
      

  <div class="page-single">
    <div class="container">
      <div class="row">
        <div class="col mx-auto" style="max-width: 32rem">
          <div class="text-center mb-6">
            <a href="https://simplelogin.io">
              <img src="/static/logo.svg"
                   style="background-color: transparent;
                          height: 20px">
            </a>
          </div>
          

  
  <div class="card" style="border-radius: 2%">
    <div class="card-body p-6">
      <h1 class="card-title">Welcome back!</h1>
      <form method="post">
        <input id="csrf_token" name="csrf_token" type="hidden" value="IjZhNDhkMTQ0OWZlYjYxYzM4MjViOGNmYTJkNzFlMmJkMTNmYTYxZDci.alasMw.x8UuIGfHOYOF70eF2K0S6zKwUTs">
        <div class="form-group">
          <label class="form-label">Email address</label>
          <input autofocus="true" class="form-control" id="email" name="email" required type="email" value="john@wick.com">
          
  

        </div>
        <div class="form-group">
          <label class="form-label">Password</label>
          <input class="form-control" id="password" name="password" required type="password" value="">
          
  

          <div class="text-muted">
            <a href="/auth/forgot_password" class="small">I forgot my password</a>
          </div>
        </div>
        <div class="form-footer">
          <button type="submit" class="btn btn-primary btn-block">Log in</button>
        </div>
      </form>
      
      
    </div>
  </div>
  <div class="text-center text-muted mt-2">
    Don't have an account yet?
    <a href="/auth/register">Sign up</a>
  </div>

        </div>
      </div>
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

The bounded log-window capture for this exact request, the inline flash, and the **negative grep**
proving no dedicated failed-login log line `[OBSERVED]`:

```text
########## Q8 EVIDENCE — wrong-credentials login ##########
### DISABLE_RATE_LIMIT=1 in env (so we are below any limit); single wrong-password attempt
### Step 1: GET login page for CSRF
csrf extracted: IjZhNDhkMTQ0OWZl... (truncated for display only; full value used in request)

### Step 2: POST wrong password
$ curl -sS -i -b $CJ -c $CJ -X POST http://localhost:7777/auth/login --data email=john@wick.com --data password=WRONG-password-xyz --data csrf_token=<csrf>
----- response status + headers (slapp cookie VALUE redacted; name+attrs kept) -----
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Tue, 14 Jul 2026 21:37:55 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 7017
Set-Cookie: slapp=<REDACTED-SESSION-COOKIE>; Expires=Tue, 21-Jul-2026 21:37:55 GMT; HttpOnly; Path=/; SameSite=Lax

----- body byte length -----
body bytes: 7018
----- the flashed error rendered inline in the 200 body (grep) -----
toastr.error("Email or password incorrect")
toastr.success("Copied to clipboard")

### Step 3: bounded server-log window for THIS request (app stdout delta):
$ tail -n +82 /tmp/gunicorn_boot.log   # new app(SL)-logger lines
2026-07-14 21:37:55,644 - SL - DEBUG - 2482 - "/app/server.py:284" - after_request() -  - 172.17.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.001354217529296875
2026-07-14 21:37:55,896 - SL - DEBUG - 2482 - "/app/server.py:284" - after_request() -  - 172.17.0.1 POST /auth/login ImmutableMultiDict([]) 200, takes 0.23941826820373535
$ tail -n +42 /tmp/gunicorn_access.log   # new gunicorn access lines
172.17.0.1 - - [14/Jul/2026:21:37:55 +0000] "GET /auth/login HTTP/1.1" 200 6918 "-" "curl/8.14.1"
172.17.0.1 - - [14/Jul/2026:21:37:55 +0000] "POST /auth/login HTTP/1.1" 200 7017 "-" "curl/8.14.1"

### Step 4: NEGATIVE grep — any dedicated failed-login app log line in the window?
$ tail -n +82 /tmp/gunicorn_boot.log | grep -iE 'LoginEvent|password incorrect|failed login'
(no matches — failed-login event writes NO application log line)
```

> **Which `toastr` line is the flash?** Only the body line
> `<script>toastr.error("Email or password incorrect");</script>` is the flashed error (rendered
> by the template's flash loop). The other two matches in the page —
> `toastr.success("Copied to clipboard")` and
> `toastr.error("Sorry for the inconvenience! … ", "Unknown Error")` — are **static page
> JavaScript** present in every render (a copy-to-clipboard helper and a generic AJAX error
> handler), **not** flashes. Corroboration: the `GET` login page is `200` / `6918` bytes while the
> wrong-password `POST` re-render is `200` / `7017` bytes — the ~99-byte difference is exactly the
> injected flash `<script>`.

**Causal explanation.** In the login view, the wrong-credentials branch
`if not user or not user.check_password(...)` (`app/auth/views/login.py:L45`) flashes
`"Email or password incorrect"` (`app/auth/views/login.py:L49`), emits `LoginEvent(failed).send()`
(`app/auth/views/login.py:L50`), and falls through to re-render the login template
(`render_template("auth/login.html", …)`, `app/auth/views/login.py:L74-L82`) — a `200`, not a
redirect. `LoginEvent.send()` calls
`newrelic.agent.record_custom_event("LoginEvent", {...})` (`app/events/auth_event.py:L23-L24`)
`[INFERRED — from source]`: this writes a NewRelic custom event and no application log line, which
is why the negative grep for `LoginEvent|password incorrect|failed login` returns no matches. The
only observable log lines are the generic `after_request` completion `DEBUG` (`server.py:L284`) and
the gunicorn access record.

---

## Summary of findings

Every answer above is grounded in captured runtime output from the canonical stack (PostgreSQL 13,
Redis 6, gunicorn `wsgi:app` on `:7777`, `-w 2 --timeout 15`, `MEM_STORE_URI` set). Source-derived
semantics are labelled `[INFERRED]` (HTTP 440 provenance; Flask-Login 0.5.0 no-rotation mechanism;
the NewRelic custom-event delivery; the exact 600s constant, bracketed by observation; the
pickle-protocol reading).

| Q | Subsystem | Direct answer (observed) |
|---|-----------|--------------------------|
| Q1 | API session fallback | Session cookie, no `Authentication` header → **`200`** + normal JSON (2550-byte alias list). |
| Q2a | Sudo guard (API key, no sudo) | **`440`** `{"error":"Need sudo"}`; no deletion (`440` is a non-standard, IIS-origin code `[INFERRED]`). |
| Q2b | Sudo guard (session only) | **`500`** `{"error":"Internal error"}`; `AttributeError` on `g.api_key=None`; no deletion. |
| Q3 | Redis session format | `pickle` (protocol 4), **300 bytes**, key `session:<uuid>`, 6 keys (`_permanent,_fresh,csrf_token,_user_id,_id,sudo_time`). |
| Q4 | Session id across login | **No change** — cookie/UUID byte-identical before/after login; `login_user` does not rotate the store id `[INFERRED]`. |
| Q5 | Forward header survival | `X-Test-Custom` **stripped**; `Received` **stripped**; original `Reply-To` **stripped** then replaced by a reverse-alias `Reply-To`. |
| Q6 | Alias token expiry | Valid at 0s and 597s; expired at 607s → window **600s** `[INFERRED, bracketed to (597s,607s]]`. |
| Q7 | API-key usage stats | API-key call sets `last_used=arrow.now()` and `times+=1` (0→5→10); session path updates **neither**. |
| Q8 | Failed login | **`200`** re-render, `Content-Length 7017`, flash `Email or password incorrect`; no dedicated log line; NewRelic custom event `[INFERRED]`. |

---

## Repository cleanup & validation

Per the read-only mandate, the only repository change is this document. After all
observations were captured:

- every temporary observation helper script created under the container's `/tmp` was **deleted**;
- the local-upload artifacts generated by `flask dummy-data`/`fake_data()` (e.g.
  `static/upload/profile_pic.svg`) and other runtime byte-code/residue were **removed**;
- the two disposable fixtures (`q2victim@example.com` / `q2disposkey`, and `q7isolatedkey`) live
  only in the disposable PG13 test database, not in the repository;
- the authenticated Redis sessions published as Q3/Q4 evidence were **invalidated** (their
  `session:<uuid>` keys deleted), so the historical cookie/UUID values in this document no longer
  map to any live session.

Literal tracked **and** ignored repository state, targeted residue checks, and session-invalidation
proof `[OBSERVED]`:

```text
===== [1] git status --porcelain (sole tracked change) =====
 M blitzy/documentation/app_2cd6ee777f8c.md

===== [2] .gitignore coverage of runtime residue (relevant lines) =====
2:*.pyc
4:.env
11:static/upload
17:.env.*

===== [3] git diff --name-status (which tracked files changed) =====
M	blitzy/documentation/app_2cd6ee777f8c.md

===== [4] Targeted residue checks (post-cleanup) =====
generated files under static/upload/: 0
*.pyc in working tree: 0
__pycache__ dirs in working tree: 0
repo-side _docbuild/ present: no (removed)
temp observation scripts on host /tmp: 0

===== [5] Non-infrastructure gitignored entries remaining (excludes venv/ node_modules/) =====
entries excluding venv/, node_modules/, static/node_modules/ (the virtualenv + JS build dirs):
.env
  ^ .env is local runtime config (gitignore L4/L17); it is NOT a repository change and is never committed.

===== [6] Redis session invalidation proof (F12) =====
session:* keys before invalidation (baseline captured this phase): 8
session:* keys now (live query against sl-redis6): 0
  -> the Q3/Q4 session UUIDs documented above no longer map to any live session.
```

No existing source, configuration, dependency, migration, workflow, or test file was modified; the
sole tracked change is `blitzy/documentation/app_2cd6ee777f8c.md` (shown as ` M` — a modification
of the stale version introduced by the prior checkpoint commit — in the `git status` proof above).
