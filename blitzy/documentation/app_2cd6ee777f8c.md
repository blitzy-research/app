# SimpleLogin — Runtime-Behavior Onboarding Q&A

This document answers five onboarding questions about **how the *running* SimpleLogin
system actually behaves at runtime** — not merely how the source reads. Every answer was
produced by **building and running the real code paths first**, then capturing the verbatim
output. Each claim is paired with (a) the exact command used, (b) the unedited observed
output, (c) the concrete observed value, (d) the responsible `file:line`, and (e) a short
cause→effect rationale.

**SimpleLogin** is an open-source, self-hostable email-aliasing / privacy service. It is a
**Flask/Python monolith** backed by **PostgreSQL** (SQLAlchemy ORM), served in production by
**Gunicorn** loading the WSGI callable `wsgi:app` [wsgi.py:3]. Application code lives under
`app/`; the REST API lives under `app/api/`.

## How to read this document

- **observed** — a value taken directly from real runtime output captured during this
  investigation (log line, HTTP response, JSON body, SQL row, or error text).
- **inferred** — a conclusion drawn by reading the source code, explicitly labeled as such.
- Citations are inline as `[file:line]` and refer to the repository at git commit
  `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c` (branch `app_2cd6ee777f8c`), which is the exact
  code that was executed.

## Questions answered

| # | Question | Direct answer (observed) |
|---|----------|--------------------------|
| Q1 | What TCP port does the app bind to on startup? | `0.0.0.0:7777` |
| Q2 | What do the startup/initialization logs look like? | Gunicorn arbiter INFO lines + per-worker app import banner incl. `>>> init logging <<<` |
| Q3 | What does the health-check endpoint return? | Body `success`, HTTP `200 OK`, `Content-Type: text/html; charset=utf-8`, `Content-Length: 7` |
| Q4 | Alias creation via REST API — JSON response + DB persistence? | HTTP `201`, a 17-key JSON object; a row is inserted into the `alias` table (API `id` == DB `id`) |
| Q5 | What happens if PostgreSQL is down at startup? | Master binds `:7777`, worker raises `sqlalchemy.exc.OperationalError` (Connection refused) at `app/db.py:12`; `Worker failed to boot`, exit code `3` |

---

## Environment / Build / Invocation

All observations were made inside the prescribed Docker image
`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`, using the **documented
Python 3.10 runtime** [Dockerfile:8], [CONTRIBUTING.md:23] and the project's Poetry-locked
dependencies — i.e., the canonical configuration, not a newer host interpreter.

**Runtime and dependency versions (observed):**

```
$ /app/venv/bin/python --version
Python 3.10.18

$ /app/venv/bin/pip list | grep -iE '^(gunicorn|Flask|Flask-Migrate|SQLAlchemy|psycopg2-binary|alembic|coloredlogs|arrow|Werkzeug|Jinja2) '
alembic                       1.4.3
arrow                         0.16.0
coloredlogs                   14.0
Flask                         1.1.2
Flask-Migrate                 2.5.3
gunicorn                      20.0.4
Jinja2                        2.11.3
psycopg2-binary               2.9.3
SQLAlchemy                    1.3.24
Werkzeug                      1.0.1
```

These match the pins in `pyproject.toml` (`python = "^3.10"` [pyproject.toml:61],
`flask = "^1.1.2"` [pyproject.toml:62], `gunicorn = "^20.0.4"` [pyproject.toml:66],
`psycopg2-binary = "^2.9.3"` [pyproject.toml:71], `SQLAlchemy = "1.3.24"` [pyproject.toml:116])
and `poetry.lock`.

**PostgreSQL version caveat (observed):** the PostgreSQL server used for these observations is
**15.13** (`15.13 (Debian 15.13-0+deb12u1)`), whereas the documented/CI target is **Postgres
13+** [CONTRIBUTING.md:25]. The behaviors under test here — the TCP port bind, the health
response, the alias JSON/row, and the connection-refused startup error — are
**database-version-independent**, so this difference does not affect the answers. This is
noted so as not to overstate environment fidelity.

**Configuration file (kept outside the repository).** The app reads all configuration from a
`CONFIG` env-file [app/config.py:65], [app/config.py:69]. A copy of `example.env` was placed
at an absolute `/tmp` path so the source tree stays byte-for-byte unchanged, with `DB_URI`
pointed at a throwaway database:

```
$ cp /code/example.env /tmp/sl_obs.env
$ sed -i 's#localhost:5432/simplelogin#localhost:5432/slobs#' /tmp/sl_obs.env
# resulting required keys:
URL=http://localhost:7777                                           # [example.env:6]
EMAIL_DOMAIN=sl.local                                               # [example.env:22]
SUPPORT_EMAIL=support@sl.local                                      # [example.env:40]
DB_URI=postgresql://myuser:mypassword@localhost:5432/slobs          # [example.env:75]
FLASK_SECRET=secret                                                 # [example.env:77]
WORDS_FILE_PATH=local_data/test_words.txt                           # [example.env:97]
```

**Schema build (observed).** A fresh database was created and migrated to head with Alembic.
Note the fresh database must **not** have `pg_trgm` pre-created (see the reproduction note at
the end of Q4/Environment) — the migration creates it itself:

```
$ dropdb --if-exists slobs && createdb slobs        # fresh DB, no pg_trgm pre-created
$ cd /code && CONFIG=/tmp/sl_obs.env /app/venv/bin/alembic upgrade head
load config file /tmp/sl_obs.env
>>> URL: http://localhost:7777
...
INFO  [alembic.runtime.migration] Running upgrade  -> 5e549314e1e2, empty message
...
INFO  [alembic.runtime.migration] Running upgrade 7d7b84779837 -> 32f25cbf12f6, alias_audit_log_index_created_at
# 255 "Running upgrade" steps; exit code 0

$ CONFIG=/tmp/sl_obs.env /app/venv/bin/alembic current
32f25cbf12f6 (head)
```

**Seed a real user + API key through the model layer (observed).** Q4 requires a valid API
key. A throwaway `/tmp` script used the real model layer (legitimate data setup, not a bypass
of the API under test). `User.create` auto-provisions a **verified default mailbox**
[app/models.py:611], [app/models.py:613] and a first "newsletter" alias, so the seeded user
starts with exactly one alias (`id=1`):

```
# /tmp/blitzy_seed.py (removed after use)
user = User.create(email="onboard@sl.local", password="onboarding-password", activated=True)
Session.commit()
api_key = ApiKey.create(user_id=user.id, name="onboarding")
Session.commit()

# observed output:
SEED_USER_ID: 1
SEED_DEFAULT_MAILBOX_ID: 1
SEED_API_KEY_CODE: <60-char code>        # [app/models.py:2365] code = random_string(60) — redacted here
SEED_API_KEY_LEN: 60
SEED_ALIAS_COUNT: 1
SEED_ALIAS_ROW: 1 simplelogin-newsletter.aboard076@sl.local | note= "This is your first alias. ..."
```

> The 60-character API key `code` is a credential; it is redacted in this document and passed
> to `curl` below via a `$API_KEY` shell variable. It is a locally-generated, throwaway value
> for a disposable database.

**Canonical run command (observed, used for Q1–Q3 and the Q4/Q5 boots):**

```
$ cd /code && CONFIG=/tmp/sl_obs.env /app/venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
```

This is exactly the Docker image's production command
`CMD ["gunicorn","wsgi:app","-b","0.0.0.0:7777","-w","2","--timeout","15"]` [Dockerfile:47].

---

## Q1 — What TCP port does the application bind to on startup?

**Direct answer (observed):** the application binds to **`0.0.0.0:7777`** (all interfaces,
TCP port 7777).

**(a) Command used**

```
$ cd /code && CONFIG=/tmp/sl_obs.env /app/venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
```

**(b) Verbatim observed output** (Gunicorn arbiter, stderr):

```
[2026-07-13 16:29:16 +0000] [1153] [INFO] Starting gunicorn 20.0.4
[2026-07-13 16:29:16 +0000] [1153] [INFO] Listening at: http://0.0.0.0:7777 (1153)
[2026-07-13 16:29:16 +0000] [1153] [INFO] Using worker: sync
[2026-07-13 16:29:16 +0000] [1154] [INFO] Booting worker with pid: 1154
[2026-07-13 16:29:17 +0000] [1155] [INFO] Booting worker with pid: 1155
```

The **development server** path was also exercised, and it binds the same port:

```
$ cd /code && CONFIG=/tmp/sl_obs.env /app/venv/bin/python server.py
 * Serving Flask app "server" (lazy loading)
 * Environment: production
 * Debug mode: on
$ curl -s -i http://127.0.0.1:7777/health | head -1
HTTP/1.0 200 OK          # dev server answered on 7777
```

**(c) Concrete observed value:** `0.0.0.0:7777`. The arbiter line `Listening at:
http://0.0.0.0:7777 (1153)` is emitted by the master process (PID `1153` here). Confirmed
**stable across ≥2 starts**: a second run produced `Listening at: http://0.0.0.0:7777 (1269)`
— identical bind address; only the PID changed.

**(d) Responsible `file:line`:**
- `Dockerfile:47` → `CMD ["gunicorn","wsgi:app","-b","0.0.0.0:7777","-w","2","--timeout","15"]` (the `-b 0.0.0.0:7777` flag).
- `Dockerfile:44` → `EXPOSE 7777` (the image documents the same port).
- `server.py:588` → `app.run(debug=True, port=7777)` (the dev-server bind).
- `example.env:6` → `URL=http://localhost:7777` (the configured public URL).

**(e) Cause→effect:** the port is supplied by the `-b 0.0.0.0:7777` argument on the Gunicorn
command line [Dockerfile:47]; Gunicorn's master process opens the listening socket and logs
`Listening at: http://0.0.0.0:7777`. It is **not** read from `URL` in the config — `URL` is
the app's public base URL, not the bind address (**inferred** from the fact that the bind
comes from the `-b` flag while `URL` is a separate config key [example.env:6]).

---

## Q2 — What do the startup / initialization logs look like?

**Direct answer (observed):** startup output has two distinct streams — (1) **Gunicorn arbiter
`[INFO]` lines** on stderr, and (2) the **application's import-time stdout**, printed **once
per worker** (so twice with `-w 2`), including the literal banner `>>> init logging <<<`
[app/log.py:67]. The Werkzeug per-request logger is **disabled** [app/log.py:70-71], so no
`"GET ... HTTP/1.1" 200` request lines ever appear.

**(a) Command used** (same canonical run as Q1); full stdout+stderr captured to a file.

**(b) Verbatim observed output** (complete startup, one full boot):

```
[2026-07-13 16:29:16 +0000] [1153] [INFO] Starting gunicorn 20.0.4
[2026-07-13 16:29:16 +0000] [1153] [INFO] Listening at: http://0.0.0.0:7777 (1153)
[2026-07-13 16:29:16 +0000] [1153] [INFO] Using worker: sync
[2026-07-13 16:29:16 +0000] [1154] [INFO] Booting worker with pid: 1154
[2026-07-13 16:29:17 +0000] [1155] [INFO] Booting worker with pid: 1155
load config file /tmp/sl_obs.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/wxwaylglndfblnvzmxhm
Upload files to local dir
>>> init logging <<<
2026-07-13 16:29:17,832 - SL - DEBUG - 1154 - "/code/app/utils.py:17" - <module>() -  - load words file: /code/local_data/test_words.txt
load config file /tmp/sl_obs.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/cftqgtlqaohteelrmblh
Upload files to local dir
>>> init logging <<<
2026-07-13 16:29:17,874 - SL - DEBUG - 1155 - "/code/app/utils.py:17" - <module>() -  - load words file: /code/local_data/test_words.txt
```

**(c) Concrete observed values:**
- **Arbiter INFO lines (stderr):** `Starting gunicorn 20.0.4`, `Listening at:
  http://0.0.0.0:7777 (<pid>)`, `Using worker: sync`, and one `Booting worker with pid:
  <pid>` per worker (two here, because `-w 2`).
- **App import-time stdout (once per worker):** `load config file /tmp/sl_obs.env`,
  `>>> URL: http://localhost:7777`, `MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default
  value`, `Paddle param not set`, `WARNING: Use a temp directory for GNUPGHOME /tmp/<random>`,
  `Upload files to local dir`, the banner `>>> init logging <<<`, and a `SL - DEBUG - ... load
  words file` line.
- The block appears **twice** — once for worker `1154` and once for worker `1155` — because
  each worker imports `wsgi:app` separately.

**(d) Responsible `file:line`:**
- `app/log.py:67` → `print(">>> init logging <<<")` (the import-time banner).
- `app/log.py:70-71` → `log = logging.getLogger("werkzeug")` / `log.disabled = True` (why no
  per-request access-log lines are printed).
- The arbiter lines (`Starting gunicorn 20.0.4`, `Listening at`, `Using worker: sync`,
  `Booting worker with pid`) originate from Gunicorn 20.0.4 itself.

**(e) Cause→effect:** the app-side lines are emitted at **module import time**, which happens
**once per worker** as each worker lazily imports `wsgi:app` (default Gunicorn worker loading,
no `--preload`) — this is why the banner block appears once per worker (**inferred** from the
worker-per-block correspondence and Gunicorn's default lazy loading). Because the `werkzeug`
logger is disabled [app/log.py:70-71], request access logs are suppressed.

**Stability across ≥2 runs (observed):** the *set* of lines is stable. A second boot produced
the same arbiter lines and the same per-worker banner block. The only differences are the
**PIDs** (run 2 master `1269`, workers `1270`/`1271`) and the **random GNUPGHOME path** (run 1
workers used `/tmp/wxwaylglndfblnvzmxhm` and `/tmp/cftqgtlqaohteelrmblh`; run 2 used
`/tmp/rndfnooocheutxwoexrf` and `/tmp/voktxpbvendcebtnfuqx`) — expected per-process variation,
not a change in behavior.

---

## Q3 — What does the health-check endpoint return?

**Direct answer (observed):** `GET /health` returns the body **`success`** (7 bytes) with
status **`200 OK`**, `Content-Type: text/html; charset=utf-8`, and `Content-Length: 7`.

**(a) Command used**

```
$ curl -i http://127.0.0.1:7777/health
```

**(b) Verbatim observed output:**

```
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Mon, 13 Jul 2026 16:29:41 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 7
Vary: Cookie
Set-Cookie: slapp=<flask-session-cookie>; Expires=Mon, 20-Jul-2026 16:29:41 GMT; HttpOnly; Path=/; SameSite=Lax

success
```

A byte count confirms the body length:

```
$ curl -s http://127.0.0.1:7777/health | wc -c
7
```

**(c) Concrete observed values:** status `200 OK`; body `success` (exactly 7 bytes);
`Content-Type: text/html; charset=utf-8`; `Content-Length: 7`; `Vary: Cookie`; a `slapp`
session cookie via `Set-Cookie`. Confirmed **stable across ≥2 requests** (identical response).

**(d) Responsible `file:line`:** `server.py:213-215`:

```python
@app.route("/health", methods=["GET"])   # server.py:213
def healthcheck():                        # server.py:214
    return "success", 200                 # server.py:215
```

**(e) Cause→effect:** the view returns the literal 2-tuple `("success", 200)`
[server.py:215]. Flask interprets a bare `str` return as the response body and serializes it
with the default `Content-Type: text/html; charset=utf-8` (hence `text/html`, not JSON), and
the `200` as the status code. The `slapp` session cookie is attached by the app's session
interface on the response (**inferred**: the cookie name `slapp` and the `Vary: Cookie` header
are added by Flask's session machinery, not by the health view itself, which only returns the
tuple). The cookie value is an anonymous session (its payload decodes to `{"_permanent":
true}`); its signed value is elided as `<flask-session-cookie>` above for credential hygiene.

---


## Q4 — Creating an alias through the REST API: JSON response and database persistence

This question has **two sides** — the HTTP/JSON side and the database side — and both were
exercised. The endpoint is `POST /api/alias/random/new` [app/api/views/new_random_alias.py:21]
(the blueprint prefix `url_prefix="/api"` [app/api/base.py:11] makes the full path
`/api/alias/random/new`). Authentication is via the **`Authentication`** request header
[app/api/base.py:17].

### Q4a — API side (the JSON response)

**(a) Command used** (`$API_KEY` = the 60-char seeded key `code`, redacted):

```
$ curl -i -X POST http://127.0.0.1:7777/api/alias/random/new \
    -H "Authentication: $API_KEY" \
    -H "Content-Type: application/json" \
    -d '{"note":"onboarding demo alias"}'
```

**(b) Verbatim observed output:**

```
HTTP/1.1 201 CREATED
Server: gunicorn/20.0.4
Date: Mon, 13 Jul 2026 16:30:00 GMT
Connection: close
Content-Type: application/json
Content-Length: 430
Access-Control-Allow-Origin: *
Vary: Cookie
Set-Cookie: slapp=<flask-session-cookie>; Expires=Mon, 20-Jul-2026 16:30:00 GMT; HttpOnly; Path=/; SameSite=Lax

{"alias":"dwarfs_mallet609@sl.local","creation_date":"2026-07-13 16:30:00+00:00","creation_timestamp":1783960200,"disable_pgp":false,"email":"dwarfs_mallet609@sl.local","enabled":true,"id":2,"latest_activity":null,"mailbox":{"email":"onboard@sl.local","id":1},"mailboxes":[{"email":"onboard@sl.local","id":1}],"name":null,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":"onboarding demo alias","pinned":false,"support_pgp":false}
```

Pretty-printed for readability (same bytes):

```json
{
  "alias": "dwarfs_mallet609@sl.local",
  "creation_date": "2026-07-13 16:30:00+00:00",
  "creation_timestamp": 1783960200,
  "disable_pgp": false,
  "email": "dwarfs_mallet609@sl.local",
  "enabled": true,
  "id": 2,
  "latest_activity": null,
  "mailbox": {"email": "onboard@sl.local", "id": 1},
  "mailboxes": [{"email": "onboard@sl.local", "id": 1}],
  "name": null,
  "nb_block": 0,
  "nb_forward": 0,
  "nb_reply": 0,
  "note": "onboarding demo alias",
  "pinned": false,
  "support_pgp": false
}
```

**(c) Concrete observed values:**
- **Status:** `201 CREATED`; `Content-Type: application/json`; `Content-Length: 430`;
  `Access-Control-Allow-Origin: *`.
- **17 keys**, sorted alphabetically: `alias`, `creation_date`, `creation_timestamp`,
  `disable_pgp`, `email`, `enabled`, `id`, `latest_activity`, `mailbox`, `mailboxes`, `name`,
  `nb_block`, `nb_forward`, `nb_reply`, `note`, `pinned`, `support_pgp`.
- **`alias` == `email`** == `dwarfs_mallet609@sl.local` (the endpoint adds a top-level `alias`
  key **in addition to** the serialized `email` key).
- `id` = `2`; `latest_activity` = `null` (brand-new alias, no activity yet); `name` = `null`;
  `nb_block`/`nb_forward`/`nb_reply` = `0`; `enabled` = `true`; `pinned`/`disable_pgp`/
  `support_pgp` = `false`; `note` echoes the request body (`"onboarding demo alias"`);
  `mailbox`/`mailboxes` reflect the seeded verified default mailbox (`onboard@sl.local`, id 1).
- These are **live** values: the random `email`, `id`, `creation_date`, and
  `creation_timestamp` are generated at runtime (a different run yields a different email such
  as `barbel_erring251@sl.local`). Only the **shape/keys/types** are fixed.

**(d) Responsible `file:line`:**
- `app/api/views/new_random_alias.py:21` → route `@api_bp.route("/alias/random/new", methods=["POST"])`.
- `app/api/views/new_random_alias.py:114-117` → `return (jsonify(alias=alias.email, **serialize_alias_info_v2(get_alias_info_v2(alias))), 201)` (the `201` and the extra `alias` key).
- `app/api/serializer.py:55` → `serialize_alias_info_v2(...)` and `:252` → `get_alias_info_v2(...)` (the 16 serialized keys).
- `app/api/base.py:17` → `api_code = request.headers.get("Authentication")` (the auth header).

**(e) Cause→effect:** `serialize_alias_info_v2` builds a dict of 16 keys [app/api/serializer.py:55];
the endpoint spreads that dict and adds `alias=alias.email`, giving 17 keys, and returns it
with status `201` [app/api/views/new_random_alias.py:114-117]. Flask 1.1.2's `jsonify` sorts
keys alphabetically (`JSON_SORT_KEYS` defaults to `True`), which is why the observed key order
is alphabetical (**inferred** from the Flask default combined with the observed ordering).

### Q4b — Database side (what is persisted)

The row is written to the **`alias`** table [app/models.py:1470].

**(a) Commands used** (before and after the API call):

```
$ psql "$DB_URI" -c "SELECT id,user_id,email,name,enabled,note,mailbox_id,flags,pinned,automatic_creation,created_at FROM alias ORDER BY id;"
```

**(b) Verbatim observed output — BEFORE the API call** (only the seeded newsletter alias):

```
 id | user_id |                   email                   | name | enabled |    note (truncated)   | mailbox_id | flags | pinned | automatic_creation |         created_at
----+---------+-------------------------------------------+------+---------+-----------------------+------------+-------+--------+--------------------+----------------------------
  1 |       1 | simplelogin-newsletter.aboard076@sl.local |      | t       | This is your first... |          1 |     0 | f      | f                  | 2026-07-13 16:28:15.225906
(1 row)
```

**Verbatim observed output — AFTER the API call** (the new row `id=2` is present):

```
 id | user_id |                   email                   | name | enabled |         note          | mailbox_id | flags | pinned | automatic_creation |         created_at
----+---------+-------------------------------------------+------+---------+-----------------------+------------+-------+--------+--------------------+----------------------------
  1 |       1 | simplelogin-newsletter.aboard076@sl.local |      | t       | This is your first... |          1 |     0 | f      | f                  | 2026-07-13 16:28:15.225906
  2 |       1 | dwarfs_mallet609@sl.local                 |      | t       | onboarding demo alias |          1 |     0 | f      | f                  | 2026-07-13 16:30:00.537212
(2 rows)
```

**(c) Concrete observed values (the created row):** `id=2`, `user_id=1`,
`email=dwarfs_mallet609@sl.local`, `name=NULL` (shown blank), `enabled=t`,
`note='onboarding demo alias'`, `mailbox_id=1`, `flags=0`, `pinned=f`,
`automatic_creation=f`, `created_at=2026-07-13 16:30:00.537212`.

**Key check — API `id` == DB primary-key `id`:** the JSON reported `"id": 2` and the persisted
row's primary key is `id = 2`. They are **equal** (`2 == 2`). The DB `created_at`
(`16:30:00.537212`, microsecond precision) corresponds to the JSON `creation_date`
(`2026-07-13 16:30:00+00:00`) and `creation_timestamp` (`1783960200`).

**(d) Responsible `file:line`:**
- `app/models.py:1469-1470` → `class Alias(Base, ModelMixin):` / `__tablename__ = "alias"`.
- `app/models.py:1721` → `Alias.create_new_random(...)` (called from the endpoint at
  `app/api/views/new_random_alias.py:106`).
- `app/models.py:116-118` → `ModelMixin.create` with `commit = kw.pop("commit", False)` — i.e.
  it defaults to **not** committing.
- `app/api/views/new_random_alias.py:107` → `Session.commit()` (the explicit commit).

**(e) Cause→effect:** the endpoint calls `Alias.create_new_random`
[app/api/views/new_random_alias.py:106] → `ModelMixin.create` which **defaults to
`commit=False`** [app/models.py:116-118], so the object is flushed but not yet durably
committed; the endpoint then calls `Session.commit()` explicitly
[app/api/views/new_random_alias.py:107], which is what durably persists the row. The
before/after snapshots show the state change: the `alias` table goes from 1 row to 2 rows,
and the new row's values match the JSON exactly.

### Q4c — Negative / auth check (proves the canonical path is exercised)

Repeating the call with a **missing** or **wrong** `Authentication` header returns `401` and
creates **no** row:

```
$ curl -i -X POST http://127.0.0.1:7777/api/alias/random/new -H "Content-Type: application/json" -d '{"note":"no auth"}'
HTTP/1.1 401 UNAUTHORIZED
{"error":"Wrong api key"}

$ curl -i -X POST http://127.0.0.1:7777/api/alias/random/new -H "Authentication: totally-wrong-key" -H "Content-Type: application/json" -d '{"note":"bad auth"}'
HTTP/1.1 401 UNAUTHORIZED
{"error":"Wrong api key"}

# alias count unchanged (no row created by the rejected calls):
$ psql "$DB_URI" -tAc "SELECT count(*) FROM alias;"
3
```

This matches `app/api/base.py:27` → `return jsonify(error="Wrong api key"), 401`, guarded by
the `require_api_auth` decorator [app/api/base.py:52]. It confirms the real
`Authentication`-authenticated path is genuinely being exercised (a bad key is rejected before
any alias is created).

### Q4d — Stability

A second successful `POST` created another alias — `id=3`, `email=barbel_erring251@sl.local`,
`note="second onboarding alias"` — so the primary key **increments** (`2 → 3`) and both rows
persist (3 rows total: the newsletter alias plus the two API-created aliases). The behavior is
stable across repeated calls.

> **Reproduction note (documented, not fixed):** on PostgreSQL, Alembic runs all migrations in
> a single transaction. The migration `migrations/versions/2021_082012_424808e1fe49_.py` runs
> `CREATE EXTENSION pg_trgm` [migration:24] inside a `try/except` that, on a duplicate, issues
> `op.execute("Rollback")`. If `pg_trgm` is **pre-created**, that rollback unwinds the whole
> transaction (dropping the just-created `alias` table) and the later
> `CREATE INDEX 'note_pg_trgm_index'` [migration:31] fails. The fix is environmental — build
> the schema on a **fresh** database and let the migration create the extension — not a source
> change.

---


## Q5 — What happens if PostgreSQL is unavailable at startup?

**Direct answer (observed):** the Gunicorn **master still binds `:7777` and logs `Listening
at: http://0.0.0.0:7777`**, but each **worker fails to boot** while importing the WSGI app.
The worker raises `sqlalchemy.exc.OperationalError` wrapping `psycopg2.OperationalError:
... port 5432 failed: Connection refused`. The break point is the **module-level eager
connection** `connection = engine.connect()` at **`app/db.py:12`**. Gunicorn then logs
`Reason: Worker failed to boot.` and the process exits with **code `3`**.

**(a) Commands used** — stop PostgreSQL, confirm it is down, then boot the canonical server:

```
$ su postgres -c '/usr/lib/postgresql/15/bin/pg_ctl -D /var/lib/postgresql/data -m fast stop'
waiting for server to shut down.... done
server stopped

$ pg_isready -h localhost -p 5432
localhost:5432 - no response

$ psql "$DB_URI" -tAc "SELECT 1"
psql: error: connection to server at "localhost" (::1), port 5432 failed: Connection refused
	Is the server running on that host and accepting TCP/IP connections?
connection to server at "localhost" (127.0.0.1), port 5432 failed: Connection refused

$ cd /code && CONFIG=/tmp/sl_obs.env /app/venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15 ; echo "exit=$?"
```

**(b) Verbatim observed output** (master binds first; then one worker's complete traceback;
the second worker's traceback is byte-for-byte identical and is omitted here; then the master
shutdown):

```
[2026-07-13 16:32:07 +0000] [1374] [INFO] Starting gunicorn 20.0.4
[2026-07-13 16:32:07 +0000] [1374] [INFO] Listening at: http://0.0.0.0:7777 (1374)
[2026-07-13 16:32:07 +0000] [1374] [INFO] Using worker: sync
[2026-07-13 16:32:07 +0000] [1375] [INFO] Booting worker with pid: 1375
[2026-07-13 16:32:07 +0000] [1376] [INFO] Booting worker with pid: 1376
[2026-07-13 16:32:07 +0000] [1376] [ERROR] Exception in worker process
Traceback (most recent call last):
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 2336, in _wrap_pool_connect
    return fn()
  ... (SQLAlchemy pool/connect frames) ...
  File "/app/venv/lib/python3.10/site-packages/psycopg2/__init__.py", line 122, in connect
    conn = _connect(dsn, connection_factory=connection_factory, **kwasync)
psycopg2.OperationalError: connection to server at "localhost" (::1), port 5432 failed: Connection refused
	Is the server running on that host and accepting TCP/IP connections?
connection to server at "localhost" (127.0.0.1), port 5432 failed: Connection refused
	Is the server running on that host and accepting TCP/IP connections?


The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "/app/venv/lib/python3.10/site-packages/gunicorn/arbiter.py", line 583, in spawn_worker
    worker.init_process()
  File "/app/venv/lib/python3.10/site-packages/gunicorn/workers/base.py", line 119, in init_process
    self.load_wsgi()
  File "/app/venv/lib/python3.10/site-packages/gunicorn/workers/base.py", line 144, in load_wsgi
    self.wsgi = self.app.wsgi()
  File "/app/venv/lib/python3.10/site-packages/gunicorn/app/base.py", line 67, in wsgi
    self.callable = self.load()
  File "/app/venv/lib/python3.10/site-packages/gunicorn/app/wsgiapp.py", line 49, in load
    return self.load_wsgiapp()
  File "/app/venv/lib/python3.10/site-packages/gunicorn/app/wsgiapp.py", line 39, in load_wsgiapp
    return util.import_app(self.app_uri)
  File "/app/venv/lib/python3.10/site-packages/gunicorn/util.py", line 358, in import_app
    mod = importlib.import_module(module)
  File "/usr/local/lib/python3.10/importlib/__init__.py", line 126, in import_module
    return _bootstrap._gcd_import(name[level:], package, level)
  ...
  File "/code/wsgi.py", line 1, in <module>
    from server import create_app
  File "/code/server.py", line 31, in <module>
    from app.admin_model import (
  File "/code/app/admin_model.py", line 11, in <module>
    from app import models, s3
  File "/code/app/models.py", line 32, in <module>
    from app.db import Session
  File "/code/app/db.py", line 12, in <module>
    connection = engine.connect()
  ... (SQLAlchemy connect frames) ...
sqlalchemy.exc.OperationalError: (psycopg2.OperationalError) connection to server at "localhost" (::1), port 5432 failed: Connection refused
	Is the server running on that host and accepting TCP/IP connections?
connection to server at "localhost" (127.0.0.1), port 5432 failed: Connection refused
	Is the server running on that host and accepting TCP/IP connections?

(Background on this error at: http://sqlalche.me/e/13/e3q8)
[2026-07-13 16:32:07 +0000] [1376] [INFO] Worker exiting (pid: 1376)
load config file /tmp/sl_obs.env
>>> URL: http://localhost:7777
...
>>> init logging <<<
[2026-07-13 16:32:08 +0000] [1375] [INFO] Worker exiting (pid: 1375)
[2026-07-13 16:32:08 +0000] [1374] [INFO] Shutting down: Master
[2026-07-13 16:32:08 +0000] [1374] [INFO] Reason: Worker failed to boot.
exit=3
```

**(c) Concrete observed values:**
- The master (PID `1374`) **binds and logs** `Listening at: http://0.0.0.0:7777 (1374)`
  **before** any worker fails — the port is opened even though the DB is down.
- The raised exception is `sqlalchemy.exc.OperationalError` wrapping
  `psycopg2.OperationalError: connection to server at "localhost" (::1), port 5432 failed:
  Connection refused` (with the IPv4 `(127.0.0.1)` follow-on).
- Gunicorn logs `Exception in worker process`, then `Worker exiting`, then `Shutting down:
  Master`, then `Reason: Worker failed to boot.`, and the process **exits with code `3`**.

**(d) Responsible `file:line` — the break point:** `app/db.py:12` → `connection =
engine.connect()`. This module-level, **eager** connection is opened at import time. The
traceback shows the exact import chain that reaches it:

```
/code/wsgi.py:1            from server import create_app
  → /code/server.py:31        from app.admin_model import (
    → /code/app/admin_model.py:11   from app import models, s3
      → /code/app/models.py:32        from app.db import Session
        → /code/app/db.py:12            connection = engine.connect()   ← break point
```

Every step of this chain is **observed** directly in the captured traceback (the file:line of
each frame is printed).

**(e) Cause→effect (inferred where noted):** Gunicorn's default worker model (no `--preload`,
matching [Dockerfile:47]) has the **master** open the listening socket first and then fork
workers that each import the WSGI app — this is why the port is bound before the worker fails
(**inferred** from Gunicorn's default lazy-loading model, consistent with the observed order:
`Listening at ...` precedes the worker error). Because `app/db.py` opens the database
connection at **import time — before the Flask app object exists** [app/db.py:12], a missing
database is a **hard boot failure** with no graceful degradation: the import raises, the
worker cannot load `wsgi:app`, and after the workers fail, the arbiter shuts down with `Reason:
Worker failed to boot.` and exit code `3`.

**Startup / failure flow (observed steps solid, inferred step labeled):**

```mermaid
graph TD
    A["gunicorn wsgi:app -b 0.0.0.0:7777 -w 2"] --> B["Master binds socket :7777 (observed)"]
    B --> C["Log: Listening at http://0.0.0.0:7777 (pid) (observed)"]
    C --> D["Worker forked, imports WSGI app — default lazy load (inferred)"]
    D --> E["wsgi.py:1 from server import create_app (observed in traceback)"]
    E --> F["server.py:31 from app.admin_model import ... (observed)"]
    F --> G["app/admin_model.py:11 from app import models, s3 (observed)"]
    G --> H["app/models.py:32 from app.db import Session (observed)"]
    H --> I["app/db.py:12 connection = engine.connect() (observed)"]
    I -->|PostgreSQL up| J["App boots and serves requests (observed: Q1–Q4)"]
    I -->|PostgreSQL down| K["OperationalError → Worker failed to boot, exit 3 (observed)"]
```

**Stability across ≥2 boots (observed):** a second boot with PostgreSQL still down produced
the same sequence — master `1387` logged `Listening at: http://0.0.0.0:7777 (1387)`, both
workers (`1388`, `1389`) raised `sqlalchemy.exc.OperationalError`, the `app/db.py:12` frame
(`connection = engine.connect()`) appeared in the traceback, and the process exited with code
`3` (`Reason: Worker failed to boot.`). Only the PIDs differed. PostgreSQL was restarted after
the test (`localhost:5432 - accepting connections`).

> This eager-connection design detail (`app/db.py:12`) is **documented, not remediated** — it
> is out of scope to change per the task's read-only rule.

---


## Coverage pass — every named item addressed

A final decomposition confirming each named mechanism, file, table, column, header, and status
code has been answered from observed runtime output:

| Named item | Where answered | Observed value |
|------------|----------------|----------------|
| TCP port `7777` | Q1 | Bind `0.0.0.0:7777` [Dockerfile:47] |
| `Listening at:` arbiter line | Q1 / Q2 / Q5 | `Listening at: http://0.0.0.0:7777 (<pid>)` |
| Dev-server bind | Q1 | `python server.py` → answers on `7777` [server.py:588] |
| `Starting gunicorn 20.0.4` / `Using worker: sync` / `Booting worker with pid` | Q2 | Arbiter INFO lines (Gunicorn 20.0.4) |
| `>>> init logging <<<` banner | Q2 | Printed once per worker [app/log.py:67] |
| Werkzeug request logger disabled | Q2 | No per-request access-log lines [app/log.py:70-71] |
| `GET /health` → body / status | Q3 | `success` / `200 OK` [server.py:213-215] |
| Health `Content-Type` / `Content-Length` | Q3 | `text/html; charset=utf-8` / `7` |
| `POST /api/alias/random/new` | Q4a | HTTP `201 CREATED` [new_random_alias.py:21,114-117] |
| `Authentication` header | Q4a / Q4c | Required; missing/wrong → `401` [app/api/base.py:17,27] |
| `201` JSON with `alias` + `email` keys | Q4a | 17 keys; `alias == email` [new_random_alias.py:115], [serializer.py:55] |
| `alias` table + column values | Q4b | Row `id=2,user_id=1,email=...,note=...,mailbox_id=1,flags=0,pinned=f,automatic_creation=f,created_at=...` [app/models.py:1470] |
| API `id` == DB `id` | Q4b | `2 == 2` (confirmed) |
| Explicit `Session.commit()` (commit=False default) | Q4b | [new_random_alias.py:107], [app/models.py:116-118] |
| `401` `{"error":"Wrong api key"}` | Q4c | Observed for missing and wrong header [app/api/base.py:27] |
| `app/db.py:12` break point | Q5 | `connection = engine.connect()` in traceback |
| `OperationalError` (connection refused) | Q5 | `sqlalchemy.exc.OperationalError` wrapping `psycopg2.OperationalError` |
| Import chain to the break point | Q5 | `wsgi.py:1 → server.py:31 → app/admin_model.py:11 → app/models.py:32 → app/db.py:12` |
| `Worker failed to boot` / exit code `3` | Q5 | `Reason: Worker failed to boot.`, `exit=3` |

**Summary of direct answers:**

1. **Q1 — Port:** the app binds **`0.0.0.0:7777`** (Gunicorn `-b 0.0.0.0:7777` [Dockerfile:47];
   dev server `port=7777` [server.py:588]).
2. **Q2 — Startup logs:** Gunicorn arbiter `[INFO]` lines (`Starting gunicorn 20.0.4`,
   `Listening at: http://0.0.0.0:7777 (<pid>)`, `Using worker: sync`, `Booting worker with
   pid: <n>` per worker) plus the app's import-time stdout printed once per worker, including
   `>>> init logging <<<` [app/log.py:67]; the Werkzeug request logger is disabled
   [app/log.py:70-71].
3. **Q3 — Health check:** `GET /health` → body **`success`** (7 bytes), status **`200 OK`**,
   `Content-Type: text/html; charset=utf-8`, `Content-Length: 7` [server.py:213-215].
4. **Q4 — Alias creation:** `POST /api/alias/random/new` (with the `Authentication` header) →
   HTTP **`201`** and a **17-key** JSON object where `alias == email`; a row is inserted into
   the **`alias`** table [app/models.py:1470] and the JSON `id` equals the DB primary-key `id`.
5. **Q5 — PostgreSQL down:** the master binds `:7777` and logs `Listening at ...`, then the
   worker raises `sqlalchemy.exc.OperationalError` (Connection refused) at **`app/db.py:12`**
   during the WSGI import; Gunicorn reports `Worker failed to boot.` and exits with code **`3`**.

