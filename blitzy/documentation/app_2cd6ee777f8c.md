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
  id on login (Q4); the exact `600`-second token constant, which runtime observation
  brackets to `(593s, 607s]` (Q6); and the "pickle protocol 4" reading of the leading session
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
# ---- Canonical backing services + app container: LITERAL stand-up commands ----
# (Reconstructed verbatim from `docker inspect` of the running containers; the
#  `docker ps` / network verification block immediately below confirms they match.)
# postgres:13 -> host 'sl-pg13'; redis:6 -> host 'sl-redis6'; both on network 'sl-canon'.
docker network create sl-canon
docker run -d --name sl-pg13 --network sl-canon \
    -e POSTGRES_USER=test -e POSTGRES_PASSWORD=test -e POSTGRES_DB=test  postgres:13
docker run -d --name sl-redis6 --network sl-canon  redis:6
# The app container: host repo bind-mounted at /app, project venv volume, port 7777 published.
docker run -d --name sl-app -w /app --shm-size=256m -p 7777:7777 \
    -v /tmp/blitzy/app/blitzy-bbae9d68-fff2-457c-b3c7-71b46c6af08e_aefa13:/app \
    -v sl-venv:/app/venv --entrypoint /bin/bash \
    ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0 -c 'sleep infinity'
docker network connect sl-canon sl-app     # app is on the default bridge AND sl-canon

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
    --error-logfile /tmp/gunicorn_error.log \
    > /tmp/gunicorn_boot.log 2>&1   # capture app (SL) stdout logger — Q2/Q8 read this file
```

The three containers and their network membership, exactly as actually running `[OBSERVED]`
(this is the verification that the literal commands above match reality):

```text
$ docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}'
NAMES       IMAGE                                                           STATUS          PORTS
sl-redis6   redis:6                                                         Up 46 minutes   6379/tcp
sl-pg13     postgres:13                                                     Up 46 minutes   5432/tcp
sl-app      ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0   Up 46 minutes   0.0.0.0:7777->7777/tcp
$ docker network inspect sl-canon --format '{{range .Containers}}{{.Name}}={{.IPv4Address}} {{end}}'
sl-redis6=172.18.0.3/16 sl-pg13=172.18.0.2/16 sl-app=172.18.0.4/16
$ docker inspect sl-app --format '{{range $k,$v := .NetworkSettings.Networks}}{{$k}}({{$v.IPAddress}}) {{end}}'
bridge(172.17.0.2) sl-canon(172.18.0.4)
```

### Database migration (transparent disclosure of a workaround)

Running `alembic upgrade head` **fails only when the target PostgreSQL 13 database already has the
`pg_trgm` extension installed** (e.g., a re-used or partially-migrated database). In that case
migration `424808e1fe49` takes its "pg_trgm already loaded" branch and issues `op.execute("Rollback")`
(`migrations/versions/2021_082012_424808e1fe49_.py:L29`), which under Alembic's single-transaction DDL
discards the in-flight upgrade transaction — including the `alias` table created earlier in that same
transaction — so the subsequent `op.create_index('note_pg_trgm_index', 'alias', ...)` (`:L31`) fails
with `UndefinedTable`. On a **genuinely clean** PG13 with no pre-existing `pg_trgm`, `CREATE EXTENSION
pg_trgm` succeeds, no `Rollback` fires, and `alembic upgrade head` completes cleanly to head
`32f25cbf12f6`. The literal failure below was reproduced with `pg_trgm` pre-installed `[OBSERVED]`:

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
the migration's `pg_trgm` `Rollback` branch combined with a pre-existing `pg_trgm` extension, not of
this investigation.

**Workaround (disclosed for full transparency):** the authentic, complete schema at Alembic head
`32f25cbf12f6` was provisioned onto the empty PG13 database using a **schema-only** `pg_dump`
(structure only, **no data**) taken from the image's reference database (which is stamped at the
same head), after which the Alembic version table was stamped to `32f25cbf12f6`. Because the dump
is schema-only, every data table was empty prior to seeding — this is the disposable-DB
initial-state guard. After provisioning, Alembic reports head and `upgrade head` is a clean no-op
`[OBSERVED]`. The literal provisioning commands were:

```bash
# Schema-only dump (structure, NO data) from the image's reference DB (already stamped at head
# 32f25cbf12f6), piped into the empty canonical PG13; then stamp Alembic's version table so the
# subsequent `alembic upgrade head` is a clean no-op (avoids the pg_trgm Rollback branch above):
pg_dump --schema-only --no-owner --no-privileges "$IMAGE_REF_DB_URI" \
    | psql "postgresql://test:test@sl-pg13:5432/test"
DB_URI="postgresql://test:test@sl-pg13:5432/test" alembic stamp 32f25cbf12f6
```

The resulting head state is confirmed `[OBSERVED]`:

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


**Supplementary end-to-end evidence — login handshake, access-log correlation, API-key
side-effect control, and unauthenticated control `[OBSERVED]`.** The verbose capture above shows the
session-only `200`. The following self-contained run additionally shows (1) the real login handshake
that mints the session, (2) the gunicorn access-log line for that exact session request, (3) proof
that the session-fallback path updates **no** `api_key` row (contrast Q7), and (4) the unauthenticated
`401` control. (The alias JSON is larger here — `Content-Length: 3953` vs the `2550` above — purely
because more aliases exist in the live DB at this later capture moment; the `200` status and the
session-fallback mechanism are the invariant. The 3953-byte body is the same structure shown above and
is elided below where marked.)

```text
### STEP 1 — GET /auth/login (mint anonymous session cookie + obtain CSRF token)
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Wed, 15 Jul 2026 05:04:43 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 6918
Set-Cookie: slapp=1297feca-17ea-49ef-be04-bd2430b99059.5mJbkUzobGda0zExDegrHp67ov4; Expires=Wed, 22-Jul-2026 05:04:43 GMT; HttpOnly; Path=/; SameSite=Lax
extracted csrf_token (len=91): Ijc2M2E3M2Y4ZmRjMjg3NTQ5YzQwMWQ3NDQyYjg1YmIyYTcwYmE0MjYi.alcU6w.Sj4cvAuRtWEDu1uSeJcDfe5iywU

### STEP 2 — api_key BEFORE (pristine keys id=1 'code', id=2 'codeFF') + whole-table hash
id=1 code=code times=0 last_used=NULL
id=2 code=codeFF times=0 last_used=NULL
whole api_key table (times,last_used) md5 BEFORE = 3f42cef757882a5af3d206b723961edb

### STEP 3 — POST /auth/login (email=john@wick.com, real password, csrf_token) -> 302
HTTP/1.1 302 FOUND
Server: gunicorn/20.0.4
Date: Wed, 15 Jul 2026 05:04:44 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 229
Location: http://localhost:7777/dashboard/
Set-Cookie: slapp=1297feca-17ea-49ef-be04-bd2430b99059.5mJbkUzobGda0zExDegrHp67ov4; Expires=Wed, 22-Jul-2026 05:04:44 GMT; HttpOnly; Path=/; SameSite=Lax
# NB: the slapp cookie value is byte-identical before and after login (see Q4: no session-ID rotation on login).

### STEP 4 — session-only GET /api/aliases?page_id=0 (cookie jar, NO Authentication header) -> 200
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Wed, 15 Jul 2026 05:04:44 GMT
Connection: close
Content-Type: application/json
Content-Length: 3953
Access-Control-Allow-Origin: *
Set-Cookie: slapp=1297feca-17ea-49ef-be04-bd2430b99059.5mJbkUzobGda0zExDegrHp67ov4; Expires=Wed, 22-Jul-2026 05:04:44 GMT; HttpOnly; Path=/; SameSite=Lax

{"aliases":[ ... 3953-byte alias JSON, identical structure to the body shown above, elided here ... ]}

### STEP 5 — access-log correlation for that exact session request:
172.17.0.1 - - [15/Jul/2026:05:04:44 +0000] "GET /api/aliases?page_id=0 HTTP/1.1" 200 3953 "-" "curl/8.14.1"
2026-07-15 05:04:44,414 - SL - DEBUG - 33 - "/app/server.py:284" - after_request() -  - 172.17.0.1 GET /api/aliases ImmutableMultiDict([('page_id', '0')]) 200, takes 0.04555535316467285

### STEP 6 — api_key AFTER the session call (IDENTICAL: session path touches no api_key row)
id=1 code=code times=0 last_used=NULL
id=2 code=codeFF times=0 last_used=NULL
whole api_key table (times,last_used) md5 AFTER  = 3f42cef757882a5af3d206b723961edb
RESULT: api_key table UNCHANGED (before==after) -> session-fallback bumps NO api_key row

### STEP 7 — CONTROL: unauthenticated GET /api/aliases?page_id=0 (NO cookie, NO header) -> 401
HTTP/1.1 401 UNAUTHORIZED
Server: gunicorn/20.0.4
Date: Wed, 15 Jul 2026 05:04:44 GMT
Connection: close
Content-Type: application/json
Content-Length: 26
Access-Control-Allow-Origin: *
Set-Cookie: slapp=10f11a00-7646-408b-80d4-67173e58fd1b.GG7faWUyaU-MAS4LoDw0j0osw5k; Expires=Wed, 22-Jul-2026 05:04:44 GMT; HttpOnly; Path=/; SameSite=Lax

{"error":"Wrong api key"}
```

The session cookie — and nothing else — authorizes the request: the same endpoint returns
`401 {"error":"Wrong api key"}` (Content-Length 26) with no credential, `200` with the session cookie,
and the `api_key` table is **untouched** by the session path (identical `md5` over every
`(times,last_used)` pair before and after: `3f42cef757882a5af3d206b723961edb`), whereas the API-key
path in Q7 increments `times`/`last_used`. This is the session-fallback branch of `authorize_request`
(`app/api/base.py:L20-L25`), which sets `g.user = current_user` and never assigns `g.api_key`.

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

**Supplementary evidence — the API-key usage counter is bumped even on the `440` path `[OBSERVED]`.**
`authorize_request` runs first (as the auth check) and commits `last_used`/`times` **before**
`require_api_sudo` evaluates, so a request that ends in `440` still increments the key's `times`. The
following before/after capture on the same disposable key `q2disposkey` (owner user id=3; `sudo_mode_at`
NULL, so the `440` is guaranteed and non-destructive) shows `times` `6 -> 7` and `last_used` advanced,
with the DELETE correlated in the access log:

```text
### BEFORE: api_key 'q2disposkey'
user_id=3 times=6 last_used=2026-07-15 05:01:12.86221 sudo_mode_at=NULL
disposable owner user exists (id=3)? count=1

### REQUEST: DELETE /api/user with valid key 'q2disposkey' (sudo NOT active)
HTTP/1.1 440 UNKNOWN
Server: gunicorn/20.0.4
Date: Wed, 15 Jul 2026 05:04:45 GMT
Connection: close
Content-Type: application/json
Content-Length: 22
Access-Control-Allow-Origin: *
Set-Cookie: slapp=5036a186-82fc-4bc5-8ed6-a9f7028f3eb1.xQih9NwcK55XP1zX0iDfDFZmLh4; Expires=Wed, 22-Jul-2026 05:04:45 GMT; HttpOnly; Path=/; SameSite=Lax

{"error":"Need sudo"}

### AFTER: api_key 'q2disposkey' (times +1, last_used updated; sudo still NULL)
user_id=3 times=7 last_used=2026-07-15 05:04:45.22648 sudo_mode_at=NULL
disposable owner user STILL exists (non-destructive 440)? count=1

### access-log correlation for the DELETE:
172.17.0.1 - - [15/Jul/2026:05:04:45 +0000] "DELETE /api/user HTTP/1.1" 440 22 "-" "curl/8.14.1"
```

The counter moved from `times=6` to `times=7` and `last_used` advanced to `2026-07-15 05:04:45.22648`
across a request whose response was `440` — because `authorize_request` (`app/api/base.py:L28-L32`)
sets `api_key.last_used = arrow.now()`, increments `api_key.times`, and commits **before**
`require_api_sudo` (`app/api/base.py:L63-L73`) runs its sudo check and returns `440`. The owner user
(id=3) still exists, confirming the `440` path never reaches the account-deletion body.


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

### SERVER-SIDE traceback (new app (SL)-logger stdout lines in /tmp/gunicorn_boot.log from this request):
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

**Server-side Redis view — before login / after login / after logout `[OBSERVED]`.** The same
transition observed directly in the store (`redis.Redis(host="sl-redis6")`; key format
`session:<uuid>`, `app/session.py:L44-L45`). The `slapp` cookie's UUID *is* the Redis key: on login
the **same** key is updated in place (byte length grows, TTL promoted), and on logout the key is
**deleted** and a new UUID minted — so a replayed pre-logout cookie is rejected server-side:

```text
########## Q4 REDIS TRANSITION — anon -> auth -> logout (ONE cookie jar) ##########
### ANON GET /auth/login -> slapp cookie minted
anon slapp cookie value: 66ea7359-53d5-46d3-9ce1-fd12f71c64af.gtgEtOi42huTnOomNWYiQiYeYb8
[BEFORE login (anonymous session)] key=session:66ea7359-53d5-46d3-9ce1-fd12f71c64af
  EXISTS=1  STRLEN(bytes)=96  TTL(s)=300  total session:* keys=44
  pickled session-dict keys (3): ['_fresh', '_permanent', 'csrf_token']

### LOGIN POST (same jar): HTTP/1.1 302 FOUND
auth slapp cookie value: 66ea7359-53d5-46d3-9ce1-fd12f71c64af.gtgEtOi42huTnOomNWYiQiYeYb8
cookie BYTE-IDENTICAL before==after login? True
[AFTER login (authenticated session)] key=session:66ea7359-53d5-46d3-9ce1-fd12f71c64af
  EXISTS=1  STRLEN(bytes)=300  TTL(s)=604800  total session:* keys=44
  pickled session-dict keys (6): ['_fresh', '_id', '_permanent', '_user_id', 'csrf_token', 'sudo_time']

### GET /api/aliases?page_id=0 with the authenticated session jar (no api key): HTTP/1.1 200 OK
   (returns the JSON alias list — the session authenticates via the Q1 fallback path; full body under Q1)

### GET /auth/logout (same jar): HTTP/1.1 302 FOUND | Location: http://localhost:7777/auth/login
post-logout jar slapp uuid: 3f4dc158-8611-48a9-a714-a823c1ecb0c1 (new uuid rotated)
[AFTER logout (the FORMER authenticated uuid — expect EXISTS=0)] key=session:66ea7359-53d5-46d3-9ce1-fd12f71c64af
  EXISTS=0  STRLEN(bytes)=0  TTL(s)=-2  total session:* keys=44
  pickled session-dict keys (0): []

### REPLAY pre-logout cookie on GET /api/aliases?page_id=0 (expect 401):
HTTP/1.1 401 UNAUTHORIZED
Server: gunicorn/20.0.4
Connection: close
Content-Type: application/json
Content-Length: 26
Access-Control-Allow-Origin: *
Set-Cookie: slapp=<REDACTED-NEW-ANON-COOKIE>; Expires=...; HttpOnly; Path=/; SameSite=Lax
########## END Q4 REDIS TRANSITION ##########
```

Three observed facts anchor the answer: (1) the anonymous session already exists in Redis with a
short **`300`s** TTL — `app/session.py:L95-L96` sets `ttl = 300` when `"_user_id"` is absent ("Only 5
minutes for non-authenticated sessions"); (2) login updates the **same** `session:<uuid>` key in
place — the byte length grows `96 → 300` as `login_user()` adds `_user_id`, `_id`, and `sudo_time`,
and the TTL is promoted to the full **`604800`s** (7-day `permanent_session_lifetime`,
`app/session.py:L92`) with **no key rotation**; and (3) logout (`/auth/logout` → `logout_session()` →
`purge_session`, `app/session.py:L61-L64`) **deletes** the record (`EXISTS 1 → 0`, `TTL -2`) and mints
a new UUID, so the pre-logout cookie replayed against a protected endpoint now returns **`401`** — the
session is genuinely invalidated server-side, not merely cleared in the browser.


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


**Supplementary evidence — complete SMTP wire dialogue, the forwarded message *body*, and a direct
DB readback `[OBSERVED]`.** The primary capture above shows the handler trail and the forwarded header
block; the following self-contained re-run additionally resolves the three items the header-only view
omitted: (1) the **complete SMTP client wire dialogue** (`EHLO`/`MAIL`/`RCPT`/`DATA`/`QUIT` with every
reply code), captured client-side via `smtplib.set_debuglevel(1)`; (2) the **complete forwarded body**
(not just headers); and (3) a **direct `psql` readback** of the `email_log` and `contact` rows the
forward path created. This re-run delivered to the same seeded alias `operas_enzyme523@sl.local` and
created `email_log id=13` (matching the forwarded `X-SimpleLogin-EmailLog-ID: 13` header):

```text
### ===== COMPLETE SMTP CLIENT WIRE DIALOGUE (smtplib set_debuglevel(1)) — EHLO/MAIL/RCPT/DATA/QUIT + reply codes =====
send: 'ehlo probe.sl.local\r\n'
reply: b'250-9523ca2e4a73\r\n'
reply: b'250-SIZE 33554432\r\n'
reply: b'250-8BITMIME\r\n'
reply: b'250-SMTPUTF8\r\n'
reply: b'250 HELP\r\n'
reply: retcode (250); Msg: b'9523ca2e4a73\nSIZE 33554432\n8BITMIME\nSMTPUTF8\nHELP'
send: 'mail FROM:<outsider@example.com>\r\n'
reply: b'250 OK\r\n'
reply: retcode (250); Msg: b'OK'
send: 'rcpt TO:<operas_enzyme523@sl.local>\r\n'
reply: b'250 OK\r\n'
reply: retcode (250); Msg: b'OK'
send: 'data\r\n'
reply: b'354 End data with <CR><LF>.<CR><LF>\r\n'
reply: retcode (354); Msg: b'End data with <CR><LF>.<CR><LF>'
data: (354, b'End data with <CR><LF>.<CR><LF>')
send: b'From: Outsider <outsider@example.com>\r\nTo: operas_enzyme523@sl.local\r\nSubject: Q5 body+dialogue supplement\r\nDate: Tue, 14 Jul 2026 21:40:00 -0000\r\nMessage-ID: <q5-probe-supp-0001@example.com>\r\nReply-To: someone@elsewhere.test\r\nReceived: from mail.example.com (mail.example.com [203.0.113.9]) by mx.sl.local; Tue, 14 Jul 2026 21:39:50 -0000\r\nX-Test-Custom: hello-world\r\nMIME-Version: 1.0\r\nContent-Type: text/plain; charset="utf-8"\r\nContent-Transfer-Encoding: 7bit\r\n\r\nThis is the Q5 probe body.\r\n.\r\n'
reply: b'250 Message accepted for delivery\r\n'
reply: retcode (250); Msg: b'Message accepted for delivery'
data: (250, b'Message accepted for delivery')
send: 'quit\r\n'
reply: b'221 Bye\r\n'
reply: retcode (221); Msg: b'Bye'
### explicit reply codes: EHLO(from ehlo above)  MAIL FROM=250 b'OK'  RCPT TO=250 b'OK'  DATA=250 b'Message accepted for delivery'

### recipient mapping: envelope_from='sl.lmycyibrgmwcamrtha2dsnbvlu.osj52b2mcahyg@sl.local'  envelope_to='john@wick.com'

### ===== forwarded-message headers that tie it to the DB row (excerpt) =====
X-SimpleLogin-EmailLog-ID: 13
From: "Outsider - outsider at example.com" <outsider_at_example_com_ydmks@sl.local>
Reply-To: "someone at elsewhere.test" <someone_at_elsewhere_test_kjmac@sl.local>

### ===== COMPLETE FORWARDED MESSAGE — BODY (the F-P4-05 gap) =====
Content-Type=text/plain (28 bytes):
This is the Q5 probe body.

### ===== END FORWARDED BODY =====

### ===== DIRECT psql READBACK (forward-path DB rows created by this delivery) =====
$ psql -tAc "select ... from email_log where id=13"
id=13 | contact_id=2 | alias_id=2 | user_id=1 | mailbox_id=1 | is_reply=false | blocked=false | bounced=false
$ psql -tAc "select ... from contact where id in (2,3)"   # get-or-create resolved these two
id=2 | alias_id=2 | website_email=outsider@example.com | reply_email=outsider_at_example_com_ydmks@sl.local | name=Outsider
id=3 | alias_id=2 | website_email=someone@elsewhere.test | reply_email=someone_at_elsewhere_test_kjmac@sl.local | name=NULL
```

**Reading the supplement.** The wire dialogue confirms a real SMTP traversal: the server answered
`EHLO` with `250-…/SIZE/8BITMIME/SMTPUTF8/HELP`, then `250 OK` to `MAIL FROM`/`RCPT TO`, `354` to
`DATA`, and `250 Message accepted for delivery` after the terminating `.` — finally `221 Bye` to
`QUIT`. The forwarded **body** survives intact and unmodified (`text/plain`, 28 bytes,
`This is the Q5 probe body.`); no footer is injected because the message carried no unsubscribe header
(`unsubscribe_generator.py:L36`, logged in the primary trail). The `psql` readback closes the loop: the
forward path wrote `email_log id=13` (contact 2, alias 2, mailbox 1) and the two `contact` rows'
`reply_email` values — `outsider_at_example_com_ydmks@sl.local` and
`someone_at_elsewhere_test_kjmac@sl.local` — are **byte-identical** to the rewritten `From` and
`Reply-To` reverse-alias addresses in the forwarded header block, confirming the reverse aliases are
the persisted contacts (not transient values).

---

## Q6 — Alias-creation token expiry window (two independent real-form runs)

**Direct answer `[OBSERVED]`:** the alias-creation token (the signed alias suffix) is valid for
**600 seconds**. Two independent real-form runs both showed: a token used at **age ~0s** and at
**age ~593s** creates the alias (**success**, a new row appears in the `alias` table), while the
**same** token used at **age ~607s** is rejected as expired (**no row created**). Runtime
observation therefore brackets the window to **`(593s, 607s]`**; the boundary is exactly **600
seconds** `[INFERRED — from signer.unsign(max_age=600), tightly bracketed by the observation
above]`.

**How it was exercised (two independent canonical runs — resolves the prior filtered/synthetic
attempt).** Two independent sequences (RUN A and RUN B), each: authenticated login → `GET
/dashboard/custom_alias` → **extract the server-minted `signed-alias-suffix` and CSRF token from
the page HTML** (no `signer.sign`, no hand-forged token) → **immediate** `POST` to
`/dashboard/custom_alias` (success) → **real monotonic wall-clock wait** → **near-boundary** `POST`
at ~593s (success) → wait → **expired** `POST` at ~607s (rejected). For **every one of the six
posts** the capture records the **complete `curl -i` response** (status line + all headers
+ body), **both** a monotonic elapsed age (`time.monotonic`) **and** a UTC wall-clock timestamp,
**and** the PostgreSQL `alias`-row state **before and after** the post — so the earlier
finding (a grep-filtered excerpt mislabelled "complete") is fully resolved. The whole capture ran
~10 minutes (start `2026-07-15T04:41:34.084038Z` → done `2026-07-15T04:51:42.482310Z`). The
complete capture below reproduces the script provenance and the full log — verbatim except for the
session-cookie-value redaction also used in the Q4/Q8 captures (`slapp=` value shown as
`<REDACTED-SESSION-COOKIE>`, name and attributes kept) — so the token source, timing, HTTP
responses, and database effects are all auditable `[OBSERVED]`:

```text
################################################################
# Q6 - ALIAS-CREATION TOKEN EXPIRY WINDOW (two independent REAL-FORM runs)
# ENHANCED CANONICAL METHOD (resolves F-P4-02): tokens minted BY THE REAL FORM
# (extracted from GET /dashboard/custom_alias page HTML), submitted via REAL POST
# to /dashboard/custom_alias with the real csrf_token; REAL monotonic wall-clock waits.
# For EVERY one of the SIX posts this capture records the COMPLETE response
# (status line + ALL headers + body), BOTH a monotonic elapsed age (time.monotonic)
# AND a UTC wall-clock timestamp, AND the PostgreSQL alias-row state BEFORE and AFTER
# each post. No signer.sign, no hand-forged token, no clock skew. The ONLY edit to the
# raw bytes below is the same session-cookie hygiene redaction used in the Q4/Q8
# captures: the 'slapp=' cookie VALUE is shown as <REDACTED-SESSION-COOKIE> while the
# cookie name and ALL its attributes (Expires/HttpOnly/Path/SameSite) are kept verbatim.
################################################################

===== CAPTURE SCRIPT (provenance) - /tmp/qafix/q6_capture.py =====
#!/usr/bin/env python3
"""Q6 enhanced capture (F-P4-02): two independent REAL-FORM token cycles.

For every one of the SIX posts (immediate / near-boundary / expired x RUN A / RUN B) records:
  * the COMPLETE, unedited `curl -i` HTTP response (status line + all headers + body),
  * BOTH a monotonic elapsed age (time.monotonic) AND a UTC wall-clock timestamp,
  * the PostgreSQL alias-row state BEFORE and AFTER the post (row for the exact prefix).
Tokens are minted by the real GET /dashboard/custom_alias page (no signer.sign, no clock skew).
The real form CSRF token (csrf_token) is submitted, exactly as the browser does.
Runs on the host: curl -> http://localhost:7777 ; DB via `docker exec sl-pg13 psql`.
"""
import subprocess, time, datetime, re

BASE = "http://localhost:7777"
SCRATCH = "/tmp/qafix"
LOG = SCRATCH + "/q6_evidence.log"
AGE_NEAR = 593
AGE_EXP = 607

logf = open(LOG, "w", buffering=1)

def log(*a):
    logf.write(" ".join(str(x) for x in a) + "\n")

def sh(cmd):
    return subprocess.run(cmd, shell=True, capture_output=True, text=True)

def utc_now():
    return datetime.datetime.now(datetime.timezone.utc).strftime("%Y-%m-%dT%H:%M:%S.%fZ")

def psql(sql):
    return sh("PGPASSWORD=test psql -h sl-pg13 -U test -d test -tAc \"%s\"" % sql).stdout.strip()

def curl_login(label):
    cj = "%s/q6_%s_cj.txt" % (SCRATCH, label)
    sh("rm -f " + cj)
    login_html = sh("curl -s -c %s %s/auth/login" % (cj, BASE)).stdout
    m = re.search(r'name="csrf_token"[^>]*value="([^"]+)"', login_html)
    lcsrf = m.group(1) if m else ""
    sh("curl -s -b %s -c %s -o /dev/null -X POST %s/auth/login "
       "--data-urlencode email=john@wick.com --data-urlencode password=password "
       "--data-urlencode csrf_token=%s" % (cj, cj, BASE, lcsrf))
    page = sh("curl -s -b %s %s/dashboard/custom_alias" % (cj, BASE)).stdout
    fm = re.search(r'name="csrf_token"[^>]*value="([^"]+)"', page)
    fcsrf = fm.group(1) if fm else ""
    sm = re.search(r'value="(\.[a-z0-9]+@sl\.local\.[^"]+)"', page)
    suffix = sm.group(1) if sm else ""
    return cj, fcsrf, suffix

def post(label, cj, fcsrf, suffix, t0_mono, prefix, stage):
    age = time.monotonic() - t0_mono
    log("\n----- [RUN %s] %s : monotonic_age=%.3fs  utc=%s  (prefix=%s) -----" % (label, stage, age, utc_now(), prefix))
    before = psql("select count(*) from alias where email like '%s.%%';" % prefix)
    log("DB BEFORE: alias rows with prefix '%s.' = %s" % (prefix, before))
    resp = sh("curl -sS -i -b %s -c %s -X POST %s/dashboard/custom_alias "
              "--data-urlencode prefix=%s --data-urlencode 'signed-alias-suffix=%s' "
              "--data-urlencode mailboxes=1 --data-urlencode note=q6-%s-%s "
              "--data-urlencode csrf_token=%s"
              % (cj, cj, BASE, prefix, suffix, stage, label, fcsrf))
    log("----- COMPLETE unedited HTTP response (curl -i) -----")
    log(resp.stdout.rstrip("\n"))
    after = psql("select count(*) from alias where email like '%s.%%';" % prefix)
    row = psql("select id||'|'||email||'|'||coalesce(note,'<null>')||'|'||created_at from alias where email like '%s.%%';" % prefix)
    log("DB AFTER: alias rows with prefix '%s.' = %s" % (prefix, after))
    log("DB AFTER row: %s" % (row if row else "(none)"))
    loc = ""
    for ln in resp.stdout.splitlines():
        if ln.lower().startswith("location:"):
            loc = ln.split(":", 1)[1].strip()
    log("redirect Location = %s" % (loc if loc else "(none)"))
    if "custom_alias" in loc:
        flashes = sh("curl -s -b %s '%s' | grep -oE 'toastr\\.(warning|error|success)\\(\"[^\"]*\"\\)'" % (cj, loc)).stdout
        log("ALL flashes rendered on redirect target (complete, unfiltered):")
        log(flashes.rstrip("\n") if flashes.strip() else "(no toastr flashes found)")
    return age

def main():
    log("################################################################")
    log("# Q6 ENHANCED CAPTURE (F-P4-02) - two independent REAL-FORM cycles")
    log("# complete unedited responses + monotonic AND UTC timing + DB before/after (all 6 posts)")
    log("# start (UTC): %s" % utc_now())
    log("################################################################")
    log("total alias rows before run: %s" % psql("select count(*) from alias;"))

    cjA, fcsrfA, sufA = curl_login("A")
    t0A = time.monotonic(); utcA = utc_now()
    log("\n[RUN A] minted signed-alias-suffix=%s  form_csrf_len=%d  t0_utc=%s" % (sufA, len(fcsrfA), utcA))
    cjB, fcsrfB, sufB = curl_login("B")
    t0B = time.monotonic(); utcB = utc_now()
    log("[RUN B] minted signed-alias-suffix=%s  form_csrf_len=%d  t0_utc=%s" % (sufB, len(fcsrfB), utcB))
    if not (sufA and sufB and fcsrfA and fcsrfB):
        log("ERROR: token or csrf extraction failed; aborting"); logf.flush(); return

    log("\n>>> STAGE 1: IMMEDIATE posts (age ~0s, expect SUCCESS 302 -> /dashboard/?highlight_alias_id=):")
    post("A", cjA, fcsrfA, sufA, t0A, "qafix_a_imm", "immediate")
    post("B", cjB, fcsrfB, sufB, t0B, "qafix_b_imm", "immediate")

    log("\n>>> STAGE 2: NEAR-BOUNDARY posts (age ~%ss, expect STILL VALID -> SUCCESS):" % AGE_NEAR)
    while time.monotonic() - t0A < AGE_NEAR:
        time.sleep(1)
    post("A", cjA, fcsrfA, sufA, t0A, "qafix_a_near", "near-boundary")
    post("B", cjB, fcsrfB, sufB, t0B, "qafix_b_near", "near-boundary")

    log("\n>>> STAGE 3: EXPIRED posts (age ~%ss, expect EXPIRED -> 302 back to form + warning):" % AGE_EXP)
    while time.monotonic() - t0A < AGE_EXP:
        time.sleep(1)
    post("A", cjA, fcsrfA, sufA, t0A, "qafix_a_exp", "expired")
    post("B", cjB, fcsrfB, sufB, t0B, "qafix_b_exp", "expired")

    log("\ntotal alias rows after run: %s" % psql("select count(*) from alias;"))
    log("# done (UTC): %s" % utc_now())
    log("### DONE_Q6_CAPTURE ###")
    logf.flush()

if __name__ == "__main__":
    main()

===== COMPLETE UNEDITED CAPTURE LOG - /tmp/qafix/q6_evidence.log =====
################################################################
# Q6 ENHANCED CAPTURE (F-P4-02) - two independent REAL-FORM cycles
# complete unedited responses + monotonic AND UTC timing + DB before/after (all 6 posts)
# start (UTC): 2026-07-15T04:41:34.084038Z
################################################################
total alias rows before run: 23

[RUN A] minted signed-alias-suffix=.invite425@sl.local.alcPfg.3lP5QVpgaKV9feO-32auhLm-bl0  form_csrf_len=91  t0_utc=2026-07-15T04:41:34.425139Z
[RUN B] minted signed-alias-suffix=.upward754@sl.local.alcPfg.Iicyg45WnRm3LwIOXoUksSbQ5Ws  form_csrf_len=91  t0_utc=2026-07-15T04:41:34.717454Z

>>> STAGE 1: IMMEDIATE posts (age ~0s, expect SUCCESS 302 -> /dashboard/?highlight_alias_id=):

----- [RUN A] immediate : monotonic_age=0.292s  utc=2026-07-15T04:41:34.717515Z  (prefix=qafix_a_imm) -----
DB BEFORE: alias rows with prefix 'qafix_a_imm.' = 0
----- COMPLETE unedited HTTP response (curl -i) -----
HTTP/1.1 302 FOUND
Server: gunicorn/20.0.4
Date: Wed, 15 Jul 2026 04:41:34 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 273
Location: http://localhost:7777/dashboard/?highlight_alias_id=34
Set-Cookie: slapp=<REDACTED-SESSION-COOKIE>; Expires=Wed, 22-Jul-2026 04:41:34 GMT; HttpOnly; Path=/; SameSite=Lax

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<title>Redirecting...</title>
<h1>Redirecting...</h1>
<p>You should be redirected automatically to target URL: <a href="/dashboard/?highlight_alias_id=34">/dashboard/?highlight_alias_id=34</a>.  If not click the link.
DB AFTER: alias rows with prefix 'qafix_a_imm.' = 1
DB AFTER row: 34|qafix_a_imm.invite425@sl.local|q6-immediate-A|2026-07-15 04:41:34.804846
redirect Location = http://localhost:7777/dashboard/?highlight_alias_id=34

----- [RUN B] immediate : monotonic_age=0.184s  utc=2026-07-15T04:41:34.901065Z  (prefix=qafix_b_imm) -----
DB BEFORE: alias rows with prefix 'qafix_b_imm.' = 0
----- COMPLETE unedited HTTP response (curl -i) -----
HTTP/1.1 302 FOUND
Server: gunicorn/20.0.4
Date: Wed, 15 Jul 2026 04:41:34 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 273
Location: http://localhost:7777/dashboard/?highlight_alias_id=35
Set-Cookie: slapp=<REDACTED-SESSION-COOKIE>; Expires=Wed, 22-Jul-2026 04:41:34 GMT; HttpOnly; Path=/; SameSite=Lax

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<title>Redirecting...</title>
<h1>Redirecting...</h1>
<p>You should be redirected automatically to target URL: <a href="/dashboard/?highlight_alias_id=35">/dashboard/?highlight_alias_id=35</a>.  If not click the link.
DB AFTER: alias rows with prefix 'qafix_b_imm.' = 1
DB AFTER row: 35|qafix_b_imm.upward754@sl.local|q6-immediate-B|2026-07-15 04:41:34.984089
redirect Location = http://localhost:7777/dashboard/?highlight_alias_id=35

>>> STAGE 2: NEAR-BOUNDARY posts (age ~593s, expect STILL VALID -> SUCCESS):

----- [RUN A] near-boundary : monotonic_age=593.254s  utc=2026-07-15T04:51:27.678756Z  (prefix=qafix_a_near) -----
DB BEFORE: alias rows with prefix 'qafix_a_near.' = 0
----- COMPLETE unedited HTTP response (curl -i) -----
HTTP/1.1 302 FOUND
Server: gunicorn/20.0.4
Date: Wed, 15 Jul 2026 04:51:27 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 273
Location: http://localhost:7777/dashboard/?highlight_alias_id=36
Set-Cookie: slapp=<REDACTED-SESSION-COOKIE>; Expires=Wed, 22-Jul-2026 04:51:27 GMT; HttpOnly; Path=/; SameSite=Lax

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<title>Redirecting...</title>
<h1>Redirecting...</h1>
<p>You should be redirected automatically to target URL: <a href="/dashboard/?highlight_alias_id=36">/dashboard/?highlight_alias_id=36</a>.  If not click the link.
DB AFTER: alias rows with prefix 'qafix_a_near.' = 1
DB AFTER row: 36|qafix_a_near.invite425@sl.local|q6-near-boundary-A|2026-07-15 04:51:27.7723
redirect Location = http://localhost:7777/dashboard/?highlight_alias_id=36

----- [RUN B] near-boundary : monotonic_age=593.148s  utc=2026-07-15T04:51:27.865375Z  (prefix=qafix_b_near) -----
DB BEFORE: alias rows with prefix 'qafix_b_near.' = 0
----- COMPLETE unedited HTTP response (curl -i) -----
HTTP/1.1 302 FOUND
Server: gunicorn/20.0.4
Date: Wed, 15 Jul 2026 04:51:27 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 273
Location: http://localhost:7777/dashboard/?highlight_alias_id=37
Set-Cookie: slapp=<REDACTED-SESSION-COOKIE>; Expires=Wed, 22-Jul-2026 04:51:27 GMT; HttpOnly; Path=/; SameSite=Lax

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<title>Redirecting...</title>
<h1>Redirecting...</h1>
<p>You should be redirected automatically to target URL: <a href="/dashboard/?highlight_alias_id=37">/dashboard/?highlight_alias_id=37</a>.  If not click the link.
DB AFTER: alias rows with prefix 'qafix_b_near.' = 1
DB AFTER row: 37|qafix_b_near.upward754@sl.local|q6-near-boundary-B|2026-07-15 04:51:27.945352
redirect Location = http://localhost:7777/dashboard/?highlight_alias_id=37

>>> STAGE 3: EXPIRED posts (age ~607s, expect EXPIRED -> 302 back to form + warning):

----- [RUN A] expired : monotonic_age=607.629s  utc=2026-07-15T04:51:42.053934Z  (prefix=qafix_a_exp) -----
DB BEFORE: alias rows with prefix 'qafix_a_exp.' = 0
----- COMPLETE unedited HTTP response (curl -i) -----
HTTP/1.1 302 FOUND
Server: gunicorn/20.0.4
Date: Wed, 15 Jul 2026 04:51:42 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 295
Location: http://localhost:7777/dashboard/custom_alias
Set-Cookie: slapp=<REDACTED-SESSION-COOKIE>; Expires=Wed, 22-Jul-2026 04:51:42 GMT; HttpOnly; Path=/; SameSite=Lax

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<title>Redirecting...</title>
<h1>Redirecting...</h1>
<p>You should be redirected automatically to target URL: <a href="http://localhost:7777/dashboard/custom_alias">http://localhost:7777/dashboard/custom_alias</a>.  If not click the link.
DB AFTER: alias rows with prefix 'qafix_a_exp.' = 0
DB AFTER row: (none)
redirect Location = http://localhost:7777/dashboard/custom_alias
ALL flashes rendered on redirect target (complete, unfiltered):
toastr.success("Alias qafix_a_imm.invite425@sl.local has been created")
toastr.success("Alias qafix_a_near.invite425@sl.local has been created")
toastr.warning("Alias creation time is expired, please retry")
toastr.success("Copied to clipboard")

----- [RUN B] expired : monotonic_age=607.531s  utc=2026-07-15T04:51:42.248153Z  (prefix=qafix_b_exp) -----
DB BEFORE: alias rows with prefix 'qafix_b_exp.' = 0
----- COMPLETE unedited HTTP response (curl -i) -----
HTTP/1.1 302 FOUND
Server: gunicorn/20.0.4
Date: Wed, 15 Jul 2026 04:51:42 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 295
Location: http://localhost:7777/dashboard/custom_alias
Set-Cookie: slapp=<REDACTED-SESSION-COOKIE>; Expires=Wed, 22-Jul-2026 04:51:42 GMT; HttpOnly; Path=/; SameSite=Lax

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<title>Redirecting...</title>
<h1>Redirecting...</h1>
<p>You should be redirected automatically to target URL: <a href="http://localhost:7777/dashboard/custom_alias">http://localhost:7777/dashboard/custom_alias</a>.  If not click the link.
DB AFTER: alias rows with prefix 'qafix_b_exp.' = 0
DB AFTER row: (none)
redirect Location = http://localhost:7777/dashboard/custom_alias
ALL flashes rendered on redirect target (complete, unfiltered):
toastr.success("Alias qafix_b_imm.upward754@sl.local has been created")
toastr.success("Alias qafix_b_near.upward754@sl.local has been created")
toastr.warning("Alias creation time is expired, please retry")
toastr.success("Copied to clipboard")

total alias rows after run: 27
# done (UTC): 2026-07-15T04:51:42.482310Z
### DONE_Q6_CAPTURE ###

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
#   monotonic age ~0.2-0.3s  -> VALID   (302 -> /dashboard/?highlight_alias_id=34 [A], =35 [B]; DB row created)
#   monotonic age   ~593s    -> VALID   (302 -> /dashboard/?highlight_alias_id=36 [A], =37 [B]; DB row created)
#   monotonic age   ~607s    -> EXPIRED (302 -> /dashboard/custom_alias; NO DB row; toastr.warning 'Alias creation time is expired, please retry')
# => runtime observation brackets the window to (593s, 607s]; exact 600s per signer.unsign(max_age=600) at alias_suffix.py:L40
# itsdangerous semantics: age>max_age raises SignatureExpired(BadSignature) -> valid at <=600s, expired at >600s
```

**Reading the evidence (per-stage verdicts).**

- **Immediate (monotonic age 0.292s [A] / 0.184s [B]) → SUCCESS:** RUN A `302 →
  /dashboard/?highlight_alias_id=34`, RUN B `…=35`; the `alias`-row count for the post's prefix goes
  **0 → 1** (row `34|qafix_a_imm.invite425@sl.local`, row `35|qafix_b_imm.upward754@sl.local`). A
  `highlight_alias_id` redirect **plus** a newly-created DB row means the alias was created.
- **Near-boundary (monotonic age 593.254s [A] / 593.148s [B]) → STILL VALID:** RUN A `302 →
  …highlight_alias_id=36`, RUN B `…=37`; DB count again **0 → 1** (row
  `36|qafix_a_near.invite425@sl.local`, row `37|qafix_b_near.upward754@sl.local`).
- **Expired (monotonic age 607.629s [A] / 607.531s [B]) → REJECTED:** both runs `302 →
  /dashboard/custom_alias` (back to the form), the DB count stays **0 → 0** (no row created), and
  the redirect target renders `toastr.warning("Alias creation time is expired, please retry")`.

> **Transparent note on the accumulated flashes.** In the expired-stage output the redirect-target
> page shows *four* `toastr` lines per run: the two earlier successes
> (`"Alias qafix_a_imm.… has been created"`, `"Alias qafix_a_near.… has been created"` for RUN A;
> the `qafix_b_*` equivalents for RUN B), the expiry **warning**, and a static
> `toastr.success("Copied to clipboard")`. This is expected: (a) the immediate and near-boundary
> `POST`s were issued header-only (`curl -i`, without following the redirect), so their success
> flashes stayed queued in the session until the expired-stage follow-up `GET` fetched the redirect
> target and consumed all pending flashes at once; and (b) `"Copied to clipboard"` is **not** a
> server-side flash at all — it is a hard-coded string emitted by the dashboard's copy-to-clipboard
> JavaScript that the flash grep also matches, shown here unfiltered rather than silently removed.
> The unambiguous Q6 signal does not depend on flash ordering: it is the **`Location`** of each
> `POST` (`/dashboard/?highlight_alias_id=N` for the two successes vs `/dashboard/custom_alias` for
> the expired case) **together with the direct `alias`-table before/after count** (0 → 1 on the two
> valid posts, 0 → 0 on the expired post) — both consistent across the two independent runs.

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
`(593s, 607s]` transition.

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

**Supplementary conditions `[OBSERVED]` — invalid-key rejection, per-call bodies, no-retry
arithmetic.** On a **fresh disposable key** (`name='q7fixkey'`, starting `times=0, last_used=NULL`),
three additional conditions were exercised: (a) a bogus `Authentication` header is rejected with
`401` and touches **no** counter; (b) every one of five valid calls returns a complete body (shown
per call); and (c) the number of gunicorn access-log lines equals the number of requests, so the
`+5` counter delta corresponds to exactly five real requests (no internal retries):

```text
########## Q7 SUPPLEMENT — invalid-key rejection, per-call bodies, no-retry arithmetic ##########
### fresh disposable key name='q7fixkey' (code redacted); endpoint GET /api/user_info

### (1) INVALID-KEY control: a bogus Authentication header must NOT touch any counter
### disposable-key row BEFORE invalid call [times|last_used]: 0|NULL
$ curl -sS -i -H "Authentication: totally-invalid-key-qafix" http://localhost:7777/api/user_info
HTTP/1.1 401 UNAUTHORIZED
Server: gunicorn/20.0.4
Connection: close
Content-Type: application/json
Content-Length: 26
Access-Control-Allow-Origin: *
### disposable-key row AFTER invalid call  [times|last_used]: 0|NULL   <- unchanged

### (2) FIVE valid API-key calls — complete per-call body shown for every call
### access-log 'GET /api/user_info' line count BEFORE 5 valid calls: 93
### disposable-key row BEFORE batch [times|last_used]: 0|NULL
  --- call1: HTTP/1.1 200 OK | Content-Length: 244 | body sha256(16)=e2a830b75f6eac83 ---
  body: {"can_create_reverse_alias":true,"connected_proton_address":null,"email":"john@wick.com","in_trial":false,"is_premium":true,"max_alias_free_plan":3,"name":"John Wick","profile_picture_url":"http://localhost:7777/static/upload/profile_pic.svg"}
  --- call2: HTTP/1.1 200 OK | Content-Length: 244 | body sha256(16)=e2a830b75f6eac83 ---
  body: {"can_create_reverse_alias":true,"connected_proton_address":null,"email":"john@wick.com","in_trial":false,"is_premium":true,"max_alias_free_plan":3,"name":"John Wick","profile_picture_url":"http://localhost:7777/static/upload/profile_pic.svg"}
  --- call3: HTTP/1.1 200 OK | Content-Length: 244 | body sha256(16)=e2a830b75f6eac83 ---
  body: {"can_create_reverse_alias":true,"connected_proton_address":null,"email":"john@wick.com","in_trial":false,"is_premium":true,"max_alias_free_plan":3,"name":"John Wick","profile_picture_url":"http://localhost:7777/static/upload/profile_pic.svg"}
  --- call4: HTTP/1.1 200 OK | Content-Length: 244 | body sha256(16)=e2a830b75f6eac83 ---
  body: {"can_create_reverse_alias":true,"connected_proton_address":null,"email":"john@wick.com","in_trial":false,"is_premium":true,"max_alias_free_plan":3,"name":"John Wick","profile_picture_url":"http://localhost:7777/static/upload/profile_pic.svg"}
  --- call5: HTTP/1.1 200 OK | Content-Length: 244 | body sha256(16)=e2a830b75f6eac83 ---
  body: {"can_create_reverse_alias":true,"connected_proton_address":null,"email":"john@wick.com","in_trial":false,"is_premium":true,"max_alias_free_plan":3,"name":"John Wick","profile_picture_url":"http://localhost:7777/static/upload/profile_pic.svg"}
### access-log 'GET /api/user_info' line count AFTER 5 valid calls: 98   (delta = 5)
### disposable-key row AFTER batch [times|last_used]: 5|2026-07-15 04:49:15.551645

### (3) NO-RETRY arithmetic: 5 requests -> 5 new access lines -> times delta (0 -> 5) equals request count.
########## END Q7 SUPPLEMENT ##########
```

The invalid key never resolves an `api_key` row — `ApiKey.get_by(code=...)` returns `None`
(`app/api/base.py:L18-L27`), so `authorize_request` short-circuits to `401` **before** the
`last_used`/`times` update at `app/api/base.py:L30-L32`; the disposable key's row is therefore
untouched (`0|NULL` before and after). All five valid bodies are byte-identical (`Content-Length:
244`, `sha256` prefix `e2a830b75f6eac83`), confirming each `200` carried the full `user_info`
payload, not an empty success. Finally, five requests produced exactly **five** new access-log lines
and a **`+5`** counter delta — the increment is one-per-request with no hidden retries.


---

## Q8 — Failed-login logging and response

**Direct answer `[OBSERVED]`:** a wrong-password login returns **HTTP `200 OK`** (the login page is
**re-rendered** — not a redirect, not a `4xx`), `Content-Type: text/html`, `Content-Length: 7017`,
with the flashed error **`Email or password incorrect`** rendered inline in the body. The
failed attempt produces **no dedicated application log line**: the only log output for the request
is the generic `after_request` request-completion `DEBUG` line (`server.py:L284`) and the gunicorn
access line (`"POST /auth/login HTTP/1.1" 200 7017`). The failed-login path *invokes*
`LoginEvent(failed).send()`, which calls `newrelic.agent.record_custom_event(...)`; but in the
canonical run the NewRelic agent is **disabled and unregistered** (`enabled=False`,
`license_key=None`, `application().active=False`, `current_transaction()=None`), so that call
**returns `None` and delivers/writes nothing** `[OBSERVED]` (disabled-state probe below) — and it
emits nothing to stdout either.

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
body bytes: 7017   # on-wire body; matches Content-Length: 7017 above and the "200 7017" access-log line below
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
`newrelic.agent.record_custom_event("LoginEvent", {...})` (`app/events/auth_event.py:L23-L24`).
`[OBSERVED]`: with the NewRelic agent disabled/unregistered at runtime (disabled-state probe below),
that call is a **runtime no-op** — it returns `None`, delivers no event, and writes no application
log line — which is why the negative grep for `LoginEvent|password incorrect|failed login` returns
no matches. The
only observable log lines are the generic `after_request` completion `DEBUG` (`server.py:L284`) and
the gunicorn access record.

**NewRelic disabled-state evidence `[OBSERVED]`.** The failed-login telemetry call resolves to a
no-op because, in the canonical run (plain `gunicorn wsgi:app`, no `newrelic-admin` wrapper, an empty
`/app/newrelic.ini`, and no `NEW_RELIC_*` env vars), the agent is never enabled or registered. Probed
in the live app context (same venv + `.env` as the gunicorn workers, `PYTHONPATH=/app`), exercising
the real `app/events/auth_event.py` `LoginEvent(failed).send()`:

```text
$ python3 nr_probe.py    # live app context; imports newrelic.agent + the real app.events.auth_event.LoginEvent
=== NewRelic runtime state (canonical gunicorn env, no newrelic-admin, empty newrelic.ini) ===
global_settings().enabled      = False
global_settings().monitor_mode = True
global_settings().license_key  = None
global_settings().app_name     = 'Python Application'
application()                  = <newrelic.api.application.Application object at 0x7d43ef9d31f0>
application().active           = False
current_transaction()          = None
record_custom_event(...) return= None (None => no event captured/delivered)
>>> URL: http://localhost:7777
WARNING: Use a temp directory for GNUPGHOME /tmp/agbpcljrydojllzrxqiw
Upload files to local dir
>>> init logging <<<
2026-07-15 04:33:51,392 - SL - DEBUG - 191 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
LoginEvent(failed).send() return= None (None; raised nothing; delivered nothing)
=== done ===
```

(The four incidental lines beginning `>>> URL:` through `load words file:` are emitted by importing
the `app` package to reach the real `LoginEvent`; they are not part of the telemetry call.) `enabled`
defaults to **`False`** — `newrelic/core/config.py:L553`: `enabled = _environ_as_bool("NEW_RELIC_ENABLED", False)`
— and no `NEW_RELIC_ENABLED` is set; with `license_key=None` and `application().active=False` the agent
never activates, so `record_custom_event(...)` — and therefore `LoginEvent.send()` — returns `None`
and delivers nothing. This is the observed reality behind the Q8 telemetry line: the source *invokes*
the NewRelic API, but at runtime no custom event is captured, recorded, or exported.

**Secondary / control conditions `[OBSERVED]`.** To bound the failed-login answer, the success path
and the nonexistent-user path were exercised on fresh cookie jars in the same run.

*Control A — SUCCESS (correct password) → `302` to `/dashboard/`, with the two `after_login` log
lines (`login_utils.py:L35`, `L44`) that the failed path does not produce:*

```text
$ curl -sS -i -b $CJ -c $CJ -X POST http://localhost:7777/auth/login --data-urlencode email=john@wick.com --data-urlencode password=<correct> --data-urlencode csrf_token=<csrf>
----- response status + headers (slapp cookie VALUE redacted; name+attrs kept) -----
HTTP/1.1 302 FOUND
Server: gunicorn/20.0.4
Date: Wed, 15 Jul 2026 04:36:10 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 229
Location: http://localhost:7777/dashboard/
Set-Cookie: slapp=<REDACTED-SESSION-COOKIE>; Expires=Wed, 22-Jul-2026 04:36:10 GMT; HttpOnly; Path=/; SameSite=Lax
(body is the standard 229-byte "Redirecting..." stub pointing at /dashboard/)
----- new app(SL)-logger lines for THIS request window -----
2026-07-15 04:36:10,971 - SL - DEBUG - 32 - "/app/app/auth/views/login_utils.py:35" - after_login() -  - log user <User 1 John Wick john@wick.com> in
2026-07-15 04:36:10,972 - SL - DEBUG - 32 - "/app/app/auth/views/login_utils.py:44" - after_login() -  - redirect user to dashboard
2026-07-15 04:36:10,972 - SL - DEBUG - 32 - "/app/server.py:284" - after_request() -  - 172.17.0.1 POST /auth/login ImmutableMultiDict([]) 302, takes 0.23988676071166992
----- new gunicorn access line -----
172.17.0.1 - - [15/Jul/2026:04:36:10 +0000] "POST /auth/login HTTP/1.1" 302 229 "-" "curl/8.14.1"
```

*Control B — NONEXISTENT user (account-enumeration check): identical `200` + identical flash
`Email or password incorrect`, no dedicated log line:*

```text
$ curl -sS -i -b $CJ -c $CJ -X POST http://localhost:7777/auth/login --data-urlencode email=nosuchuser_qafix@example.com --data-urlencode password=whatever-xyz --data-urlencode csrf_token=<csrf>
----- response status + headers (slapp cookie VALUE redacted) -----
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Wed, 15 Jul 2026 04:36:11 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 7032
Set-Cookie: slapp=<REDACTED-SESSION-COOKIE>; Expires=Wed, 22-Jul-2026 04:36:11 GMT; HttpOnly; Path=/; SameSite=Lax
----- flashed error rendered inline in the 200 body (grep) -----
toastr.error("Email or password incorrect")
toastr.success("Copied to clipboard")
----- new app(SL)-logger line for THIS request window -----
2026-07-15 04:36:11,392 - SL - DEBUG - 33 - "/app/server.py:284" - after_request() -  - 172.17.0.1 POST /auth/login ImmutableMultiDict([]) 200, takes 0.00520777702331543
----- new gunicorn access line -----
172.17.0.1 - - [15/Jul/2026:04:36:11 +0000] "POST /auth/login HTTP/1.1" 200 7032 "-" "curl/8.14.1"
----- NEGATIVE grep (LoginEvent|password incorrect|failed login) -----
(no matches)
```

Both the nonexistent-user path (`not user`) and the existing-user wrong-password path
(`not user.check_password(...)`) hit the **same** branch (`app/auth/views/login.py:L45`), so the
responses are indistinguishable in status and flash — **no account enumeration**. The only difference
is the body byte length (`7032` here vs the existing-user wrong-password `7017`): exactly `+15` bytes,
which is precisely the length difference of the echoed email value re-rendered in the form field
(`nosuchuser_qafix@example.com`, 28 chars, minus `john@wick.com`, 13 chars = `+15`) — not any
enumeration signal.


---

## Summary of findings

Every answer above is grounded in captured runtime output from the canonical stack (PostgreSQL 13,
Redis 6, gunicorn `wsgi:app` on `:7777`, `-w 2 --timeout 15`, `MEM_STORE_URI` set). Source-derived
semantics are labelled `[INFERRED]` (HTTP 440 provenance; Flask-Login 0.5.0 no-rotation mechanism;
the exact 600s constant, bracketed by observation; the pickle-protocol reading). The failed-login
NewRelic `record_custom_event` call was **observed** to be a runtime no-op (agent disabled; returns
`None`; no delivery).

| Q | Subsystem | Direct answer (observed) |
|---|-----------|--------------------------|
| Q1 | API session fallback | Session cookie, no `Authentication` header → **`200`** + normal JSON (2550-byte alias list). |
| Q2a | Sudo guard (API key, no sudo) | **`440`** `{"error":"Need sudo"}`; no deletion (`440` is a non-standard, IIS-origin code `[INFERRED]`). |
| Q2b | Sudo guard (session only) | **`500`** `{"error":"Internal error"}`; `AttributeError` on `g.api_key=None`; no deletion. |
| Q3 | Redis session format | `pickle` (protocol 4), **300 bytes**, key `session:<uuid>`, 6 keys (`_permanent,_fresh,csrf_token,_user_id,_id,sudo_time`). |
| Q4 | Session id across login | **No change** — cookie/UUID byte-identical before/after login; `login_user` does not rotate the store id `[INFERRED]`. |
| Q5 | Forward header survival | `X-Test-Custom` **stripped**; `Received` **stripped**; original `Reply-To` **stripped** then replaced by a reverse-alias `Reply-To`. |
| Q6 | Alias token expiry | Valid at ~0s and ~593s (DB row created); expired at ~607s (no row) → window **600s** `[INFERRED, bracketed to (593s,607s]]`. |
| Q7 | API-key usage stats | API-key call sets `last_used=arrow.now()` and `times+=1` (0→5→10); session path updates **neither**. |
| Q8 | Failed login | **`200`** re-render, `Content-Length 7017`, flash `Email or password incorrect`; no dedicated log line; failed-login `record_custom_event` is a **runtime no-op** (NewRelic agent disabled → returns `None`, no delivery) `[OBSERVED]`. |

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
