# SimpleLogin — Runtime Behavior Q&A (branch `app_2cd6ee777f8c`)

This document answers five behavioral questions about the SimpleLogin Flask
application **from real, captured runtime output — not from reading the source
alone**. Every answer was produced *run-first*: the relevant code path was built
and executed, the actual stdout/stderr/HTTP/DB output was captured, and only then
was the prose written around that observed evidence. Each value the questions ask
for is quoted as a literal and grounded with a `file:line` citation into this
repository.

**Exact environment used for capture** (the project's pinned stack):

- Docker image `andrewparkscaleai/coding-agent:simple-login__app__2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`
  (from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`), repository checked out at
  commit `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c` (branch `app_2cd6ee777f8c`).
- Python **3.10.18** (`Dockerfile:L8` `FROM python:3.10`; `pyproject.toml:L61` `python = "^3.10"`).
- **SQLAlchemy 1.3.24** (`pyproject.toml:L116`), **psycopg2-binary 2.9.3** (`pyproject.toml:L71`, `libpq` 140001),
  **Flask 1.1.2** (`pyproject.toml:L62`), **gunicorn 20.0.4** (`pyproject.toml:L66`), **redis 4.6.0** client
  (`pyproject.toml:L117` `redis = "^4.5.3"`).
- Backing services present in the image: **PostgreSQL 15.13** and **Redis 7.0.15**.
  > Note: the deployment baseline nominally targets PostgreSQL 13 / Redis 6; the capture image ships
  > PostgreSQL 15.13 / Redis 7.0.15. This does not change any answer below — every quoted value is what
  > the running system actually emitted.
- Timestamps in the captured output show the year **2026** because the capture container's clock is set to
  `2026-07-01`. The dates are quoted exactly as observed.

---

## How this was captured

The application is configured entirely through environment variables. The config
profile is selected by the `CONFIG` variable; when set, `app/config.py:L66-L69`
loads that env file (and prints `load config file <abs-path>` at `app/config.py:L68`),
otherwise the `else` branch `load_dotenv()` at `app/config.py:L71` runs.

The runtime used the `tests/test.env` profile. The only adjustment versus the file
is the database port: `tests/test.env:L17` declares
`DB_URI=postgresql://test:test@localhost:15432/test`, but the live PostgreSQL in the
image listens on **5432**, so a working `DB_URI` on port 5432 was exported before boot.
Because `python-dotenv`'s `load_dotenv()` uses `override=False` by default, a
pre-exported `DB_URI` takes precedence over the value in `tests/test.env`, so the app
connects to the live database while still printing the `load config file` banner.

```bash
# Inside the container (image tag above); repo at /app, virtualenv at /app/venv.
cd /app
. /tmp/sl_env.sh            # exports URL, FLASK_SECRET, MEM_STORE_URI, DB_URI=...:5432, etc.
. /app/venv/bin/activate
export CONFIG=tests/test.env  # drives app/config.py -> prints "load config file ..."

# Schema was already applied with Alembic (creates users, api_key, mailbox, alias, ...):
#   CONFIG=tests/test.env FLASK_APP=server.py flask db upgrade
# Verified: alembic head = 32f25cbf12f6, 77 tables, incl. alias / api_key / mailbox / users.

# Confirm the branch / deliverable filename:
git rev-parse --abbrev-ref HEAD   # -> app_2cd6ee777f8c (source branch this file is named after)
```

Key config literals used, all from `tests/test.env` (verified):
`URL=http://localhost` (`L2`), `EMAIL_DOMAIN=sl.local` (`L8`),
`MAX_NB_EMAIL_FREE_PLAN=3` (`L13`), `DB_URI=...localhost:15432/test` (`L17`),
`FLASK_SECRET=secret` (`L20`), `WORDS_FILE_PATH=local_data/test_words.txt` (`L38`),
`MEM_STORE_URI=redis://localhost` (`L78`).

For the alias API (Q4) a throwaway actor was seeded — a `User` (which
auto-provisions a default `Mailbox`) plus an `ApiKey` — using a temporary script
outside the repository, which was removed afterward. For Q5 the database was made
unreachable by pointing `DB_URI` at the closed port 15432.

---

## Q1 — "What port does the Flask app bind to?"

**Answer:** port **`7777`** in both runtimes — the production Gunicorn server binds it
on **all interfaces** (`0.0.0.0:7777`), and the development Flask/Werkzeug server binds
it on **loopback only** (`127.0.0.1:7777`).

### Command run (production, Gunicorn)

```bash
CONFIG=tests/test.env gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
```

### Observed output (verbatim)

```text
[2026-07-01 04:29:50 +0000] [2876] [INFO] Starting gunicorn 20.0.4
[2026-07-01 04:29:50 +0000] [2876] [INFO] Listening at: http://0.0.0.0:7777 (2876)
[2026-07-01 04:29:50 +0000] [2876] [INFO] Using worker: sync
[2026-07-01 04:29:50 +0000] [2884] [INFO] Booting worker with pid: 2884
[2026-07-01 04:29:50 +0000] [2885] [INFO] Booting worker with pid: 2885
```

The `Listening at: http://0.0.0.0:7777 (2876)` line is the direct runtime proof of
the bound address and port. A follow-up request confirmed the socket was serving:

```bash
$ curl -s -o /dev/null -w "HTTP %{http_code}\n" http://127.0.0.1:7777/health
HTTP 200
```

> The image does not ship `ss`/`netstat`, so the bound socket is evidenced by
> Gunicorn's own `Listening at:` line plus the successful `curl` above rather than a
> socket-table dump.

### Explanation (file:line)

- **Production / Gunicorn — `0.0.0.0:7777` (all interfaces):**
  - `Dockerfile:L44` `EXPOSE 7777`
  - `Dockerfile:L47` `CMD ["gunicorn","wsgi:app","-b","0.0.0.0:7777","-w","2","--timeout","15"]`
    (a commented `--log-level DEBUG` variant sits at `Dockerfile:L46`)
  - `wsgi.py:L1` `from server import create_app`; `wsgi.py:L3` `app = create_app()` — the WSGI
    callable Gunicorn loads. The `-b 0.0.0.0:7777` flag is what produced the `Listening at` line above.
- **Development / Flask built-in server — `127.0.0.1:7777` (loopback):**
  - `server.py:L588` `app.run(debug=True, port=7777)`, inside `def local_main():` (`server.py:L572`),
    guarded by `server.py:L598` `if __name__ == "__main__":` → `server.py:L599` `local_main()`.
  - Because `app.run(...)` is called with **no `host=` argument**, Flask/Werkzeug defaults the host to
    `127.0.0.1` (loopback only). This is the key contrast: same port `7777`, but the dev server is
    reachable only from inside the host, whereas Gunicorn's explicit `-b 0.0.0.0:7777` exposes all interfaces.

### Rationale

The container `EXPOSE`s `7777` and Gunicorn binds `0.0.0.0:7777` so a fronting reverse
proxy (Nginx) can reach the app across the container network. The development server
intentionally stays on loopback (`127.0.0.1`) for safety, since it is meant only for
local development, not network exposure.

---

## Q2 — "What do the startup logs look like when it initializes?"

**Answer:** boot emits, in order, three import-time banners
(`load config file <abs-path>`, `>>> URL: <url>`, `>>> init logging <<<`) plus a
first `SL`-logger DEBUG line, all preceded (under Gunicorn) by Gunicorn's own boot
lines. The dedicated `SL` logger runs at `DEBUG`; the Flask/Werkzeug request logger
is **disabled**, so there are no per-request access log lines.

### Command run

```bash
# Booting under Gunicorn prints the banners once per worker (-w 2 => twice):
CONFIG=tests/test.env gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
# The same import-time banners can be seen by simply importing the WSGI module:
CONFIG=tests/test.env python -c "import wsgi"
```

### Observed output (verbatim — full Gunicorn boot log)

```text
[2026-07-01 04:29:50 +0000] [2876] [INFO] Starting gunicorn 20.0.4
[2026-07-01 04:29:50 +0000] [2876] [INFO] Listening at: http://0.0.0.0:7777 (2876)
[2026-07-01 04:29:50 +0000] [2876] [INFO] Using worker: sync
[2026-07-01 04:29:50 +0000] [2884] [INFO] Booting worker with pid: 2884
[2026-07-01 04:29:50 +0000] [2885] [INFO] Booting worker with pid: 2885
load config file /app/tests/test.env
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-01 04:29:50,867 - SL - DEBUG - 2884 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
load config file /app/tests/test.env
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-01 04:29:50,894 - SL - DEBUG - 2885 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

(The banner block appears **twice** because `-w 2` starts two workers `2884` and
`2885`, each importing the app once.)

### Explanation (file:line)

The four import-time lines map to exact `print`/log statements:

- `load config file /app/tests/test.env` — `app/config.py:L68` `print("load config file", config_file)`,
  where the path was made absolute at `app/config.py:L67`. This banner prints **only** when `CONFIG` is
  set (`app/config.py:L66`); with `CONFIG` unset the `else` `load_dotenv()` at `app/config.py:L71` runs and
  this line does **not** appear.
- `>>> URL: http://localhost` — `app/config.py:L80` `print(">>> URL:", URL)`, with
  `URL = os.environ["URL"]` at `app/config.py:L79` (value from `tests/test.env:L2`).
- `Upload files to local dir` — `app/config.py:L328` `print("Upload files to local dir")`, reached because
  `tests/test.env:L3` sets `LOCAL_FILE_UPLOAD=1`.
- `>>> init logging <<<` — `app/log.py:L67` `print(">>> init logging <<<")`.

**Ordering insight (why config banners precede the logging banner):** `app/log.py:L7-L9`
does `from app.config import (COLOR_LOG,)`, so importing `app.log` forces `app.config` to
fully import first. That is why `load config file …` and `>>> URL: …` (from `app/config.py`)
are printed **before** `>>> init logging <<<` (from `app/log.py`).

The line immediately after the banners is a real `SL`-logger record:

```text
2026-07-01 04:29:50,867 - SL - DEBUG - 2884 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

- The logger is named `SL` and created at `DEBUG`: `app/log.py:L79` `LOG = _get_logger("SL")`;
  level set at `app/log.py:L51` `logger.setLevel(logging.DEBUG)`.
- The exact format string is defined at `app/log.py:L12-L15`:
  ```text
  "%(asctime)s - %(name)s - %(levelname)s - %(process)d - "
  '"%(pathname)s:%(lineno)d" - %(funcName)s() - %(message_id)s - %(message)s'
  ```
  So a line reads `<asctime> - SL - DEBUG - <pid> - "<pathname>:<lineno>" - <func>() - <message_id> - <message>`.
  `message_id` is empty by default, which is why there is a visible `-  -` gap (two spaces) before the message.
  `asctime` is rendered in GMT because `app/log.py:L43` sets `console_handler.formatter.converter = time.gmtime`.
  This first record originates from `app/utils.py:L17` `LOG.d("load words file: %s", WORDS_FILE_PATH)`.

**Werkzeug request logger is disabled** — `app/log.py:L70` `log = logging.getLogger("werkzeug")`
and `app/log.py:L71` `log.disabled = True`. Consequently the startup/steady-state output contains
**no** per-request access lines such as `127.0.0.1 - - [..] "GET /health HTTP/1.1" 200`. This is a
distinctive, intentional behavior and explains why hitting `/health` (Q3) produces no access-log entry.

The Gunicorn boot lines themselves (emitted by the Gunicorn master before the app import banners) are:
`Starting gunicorn 20.0.4`, `Listening at: http://0.0.0.0:7777 (<pid>)`, `Using worker: sync`,
and one `Booting worker with pid: <pid>` per worker.

> Additional observed banners depend on the environment. When importing without the pre-exported
> `GNUPGHOME`, an extra `WARNING: Use a temp directory for GNUPGHOME /tmp/...` line appears, and an
> `SL - INFO` line `... "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled`
> was also observed. Only lines that actually appeared are quoted here; none are invented.

### Rationale

The banners are plain module-level `print`/`LOG.d` calls that run **at import time**, so they fire the
moment the app package is loaded (once per Gunicorn worker). Silencing the `werkzeug` logger keeps the
application's own structured `SL` logs clean and free of Flask's default access-log noise.


---

## Q3 — "Health check: what response body AND status code does it return?"

**Answer:** the response body is exactly `success` (**7 bytes**) and the status code is
**`200`**. The default `Content-Type` is `text/html; charset=utf-8` and `Content-Length: 7`.

### Command run

```bash
curl -i http://127.0.0.1:7777/health
```

### Observed output (verbatim)

```text
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Wed, 01 Jul 2026 04:30:10 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 7
Set-Cookie: slapp=1d593535-ae92-43f6-a11b-6976895af7d2.vPMGuXGpU6zbYvQwOK6ZMCPHXoc; Expires=Wed, 08-Jul-2026 04:30:10 GMT; HttpOnly; Path=/; SameSite=Lax

success
```

Byte count of the body, measured independently:

```bash
$ BODY=$(curl -s http://127.0.0.1:7777/health); printf "body=[%s] len=%s\n" "$BODY" "$(printf %s "$BODY" | wc -c)"
body=[success] len=7
```

### Explanation (file:line)

- The endpoint is defined at `server.py:L213` `@app.route("/health", methods=["GET"])`,
  `server.py:L214` `def healthcheck():`, and `server.py:L215` `return "success", 200`.
- Flask interprets the returned tuple `("success", 200)` as `(body, status)`. Therefore:
  - **status code = `200`** (the tuple's second element),
  - **body = `success`** — the literal string from `server.py:L215`, which is **7 bytes**
    (`s`,`u`,`c`,`c`,`e`,`s`,`s`), matching the observed `Content-Length: 7`,
  - **`Content-Type: text/html; charset=utf-8`** — Flask's default content type for a bare string body.

The additional `Server: gunicorn/20.0.4` header comes from the WSGI server, and the
`Set-Cookie: slapp=...` header is the app's session cookie; neither is part of the
handler's return value.

### Rationale

Returning a bare Python string makes Flask fall back to its default `text/html; charset=utf-8`
content type, and the second element of the tuple explicitly overrides the HTTP status to `200`.
The endpoint is a trivial liveness probe — a fixed 7-byte body and a `200` are all a health check needs.


---

## Q4 — Creating an alias through the API

**Question:** "When a user creates an alias through the API: (a) what does the actual
JSON response look like, and (b) what ends up in the database — which table, and what values?"

The endpoint exercised is `POST /api/alias/random/new`. It was chosen over the custom
endpoint because it needs no `signed_suffix`, minimizing seeding. Its full path is
`/api/alias/random/new`: the blueprint prefix is `url_prefix="/api"`
(`app/api/base.py:L11`) and the route decorator is
`app/api/views/new_random_alias.py:L21` `@api_bp.route("/alias/random/new", methods=["POST"])`.

### Prerequisites / seeding (temporary, removed afterward)

The endpoint is guarded by `@require_api_auth` (`app/api/views/new_random_alias.py:L23`,
defined at `app/api/base.py:L52`). Auth reads the `Authentication` header
(`app/api/base.py:L17` `request.headers.get("Authentication")`), resolves it via
`ApiKey.get_by(code=api_code)` (`app/api/base.py:L18`), and sets `g.user = api_key.user`
(`app/api/base.py:L34`).

A `User` was created; `User.create(...)` (`app/models.py:L602`) auto-provisions the
default mailbox — `app/models.py:L611` `mb = Mailbox.create(user_id=user.id, email=user.email, verified=True)`
and `app/models.py:L613` `user.default_mailbox_id = mb.id`. An `ApiKey` was then created
(`ApiKey.create` at `app/models.py:L2365`, whose `code = random_string(60)` at
`app/models.py:L2366`). The free-plan cap `MAX_NB_EMAIL_FREE_PLAN=3` (`tests/test.env:L13`)
is enforced by `can_create_new_alias()` (`app/models.py:L867`); a fresh user with 0 aliases passes.

```bash
CONFIG=tests/test.env python /tmp/seed.py     # temporary script, deleted after capture
```

Observed seed output (the 60-char API code is redacted here as it is a credential):

```text
API_CODE: <redacted 60-char ApiKey.code — generated by random_string(60), app/models.py:L2366>
USER_ID: 1632
DEFAULT_MAILBOX_ID: 1938
USER_EMAIL: doc_demo@mailbox.test
ALIAS_GENERATOR: 1
```

### Q4a — the actual JSON response

**Answer:** HTTP **`201 CREATED`** with `Content-Type: application/json`, and a JSON body
containing the newly created alias's full v2 info plus a top-level `alias` field.

#### Command run

```bash
curl -i -X POST http://127.0.0.1:7777/api/alias/random/new \
     -H "Authentication: <API_CODE>" \
     -H "Content-Type: application/json" -d '{}'
```

#### Observed output (verbatim — status line + headers)

```text
HTTP/1.1 201 CREATED
Server: gunicorn/20.0.4
Date: Wed, 01 Jul 2026 04:30:47 GMT
Connection: close
Content-Type: application/json
Content-Length: 430
Access-Control-Allow-Origin: *
Set-Cookie: slapp=7544f352-b9b5-499d-a1f2-9591914b2be6.gDgxfwkYbe1XvRuBtPJbOZ9zvAk; Expires=Wed, 08-Jul-2026 04:30:47 GMT; HttpOnly; Path=/; SameSite=Lax
```

#### Observed output (verbatim — JSON body, pretty-printed)

```json
{
  "alias": "aweigh_taints940@sl.local",
  "creation_date": "2026-07-01 04:30:47+00:00",
  "creation_timestamp": 1782880247,
  "disable_pgp": false,
  "email": "aweigh_taints940@sl.local",
  "enabled": true,
  "id": 2677,
  "latest_activity": null,
  "mailbox": {
    "email": "doc_demo@mailbox.test",
    "id": 1938
  },
  "mailboxes": [
    {
      "email": "doc_demo@mailbox.test",
      "id": 1938
    }
  ],
  "name": null,
  "nb_block": 0,
  "nb_forward": 0,
  "nb_reply": 0,
  "note": null,
  "pinned": false,
  "support_pgp": false
}
```

#### Field enumeration (file:line)

The response is built at `app/api/views/new_random_alias.py:L114-L117`:
`return (jsonify(alias=alias.email, **serialize_alias_info_v2(get_alias_info_v2(alias))), 201,)`.
The `alias` key is added by the `jsonify(alias=alias.email, ...)` call; every other key comes
from `serialize_alias_info_v2` (`app/api/serializer.py:L55`, field dict `app/api/serializer.py:L56-L79`):

- `alias`: the new alias email (equals `email`) — from `jsonify(alias=alias.email, ...)`
- `id`: alias primary key
- `email`: the alias address
- `name`: `null` by default
- `enabled`: `true`
- `note`: `null` (no note sent)
- `creation_date`: `alias.created_at.format()` → `"2026-07-01 04:30:47+00:00"`
- `creation_timestamp`: `alias.created_at.timestamp` → `1782880247`
- `nb_forward`: `0` (`nb_forward`)
- `nb_block`: `0` (from `nb_blocked`)
- `nb_reply`: `0`
- `mailbox`: `{id, email}` of the alias's mailbox → `{"id": 1938, "email": "doc_demo@mailbox.test"}`
- `mailboxes`: list of `{id, email}` (at least one) → `[{"id": 1938, "email": "doc_demo@mailbox.test"}]`
- `support_pgp`: `false` (`alias.mailbox_support_pgp()`)
- `disable_pgp`: `false`
- `latest_activity`: `null` (no activity yet)
- `pinned`: `false`

This matches the documented contract: `docs/api.md:L399` `#### POST /api/alias/random/new`,
`docs/api.md:L412` "If success, 201 with the new alias info", and the v2 alias field list at
`docs/api.md:L430-L458`.

> The alias domain is `@sl.local`, which is `EMAIL_DOMAIN` (`tests/test.env:L8`), the default used
> when the user has no custom/public alias domain configured. The word-style local part
> (`aweigh_taints940`) follows from `ALIAS_GENERATOR=1` on the seeded user.

### Q4b — what ends up in the database (which table, what values)

**Answer:** the row is inserted into the **`alias`** table. Its persisted values match the
JSON response exactly (same `email`), and `mailbox_id` equals the user's `default_mailbox_id`.

#### Command run

```bash
psql -h localhost -p 5432 -U test -d test -x -c \
"SELECT id, email, user_id, mailbox_id, enabled, flags, note, name, disable_pgp, \
        pinned, automatic_creation, custom_domain_id, directory_id, created_at, updated_at \
 FROM alias WHERE id = 2677;"
# table-name confirmation:
psql -h localhost -p 5432 -U test -d test -tAc "SELECT to_regclass('public.alias');"
```

#### Observed output (verbatim)

```text
-[ RECORD 1 ]------+---------------------------
id                 | 2677
email              | aweigh_taints940@sl.local
user_id            | 1632
mailbox_id         | 1938
enabled            | t
flags              | 0
note               |
name               |
disable_pgp        | f
pinned             | f
automatic_creation | f
custom_domain_id   |
directory_id       |
created_at         | 2026-07-01 04:30:47.964418
updated_at         |

-- to_regclass('public.alias') -> alias
-- mailbox 1938 -> doc_demo@mailbox.test (user_id 1632)
```

#### Column-by-column explanation (file:line)

Destination table is **`alias`** — `app/models.py:L1469` `class Alias(Base, ModelMixin):`,
`app/models.py:L1470` `__tablename__ = "alias"`. The row is created by
`Alias.create_new_random(...)` (`app/models.py:L1721`), which calls
`Alias.create(user_id=user.id, email=random_email, mailbox_id=user.default_mailbox_id, note=note)`
(`app/models.py:L1750-L1754`), and is then committed at
`app/api/views/new_random_alias.py:L107` `Session.commit()`.

- `id = 2677` — primary key from `ModelMixin` (`app/models.py:L63`).
- `email = aweigh_taints940@sl.local` — `app/models.py:L1477` `email = sa.Column(sa.String(128), unique=True, nullable=False)`.
  **Identical to the JSON `email`/`alias`** returned to the client.
- `user_id = 1632` — `app/models.py:L1474` FK to `users.id`; the seeded user.
- `mailbox_id = 1938` — `app/models.py:L1506`; equals the user's `default_mailbox_id`
  (mailbox `1938` = `doc_demo@mailbox.test`), because `create_new_random` passes
  `mailbox_id=user.default_mailbox_id`.
- `enabled = t` (true) — `app/models.py:L1482` `enabled = sa.Column(sa.Boolean(), default=True, nullable=False)`.
- `flags = 0` — `app/models.py:L1483` (default `0`).
- `note =` (NULL) — `app/models.py:L1503` `note = sa.Column(sa.Text, default=None, nullable=True)`; no note was sent.
- `name =` (NULL) — `app/models.py:L1480` (`nullable=True, default=None`).
- `disable_pgp = f` (false) — `app/models.py:L1516`.
- `pinned = f` (false) — `app/models.py:L1545` (`default=False, server_default="0"`).
- `automatic_creation = f` (false) — `app/models.py:L1494`.
- `custom_domain_id =` (NULL), `directory_id =` (NULL) — none set for a random alias on the default domain.
- `created_at = 2026-07-01 04:30:47.964418` — `ModelMixin` `app/models.py:L64`
  (`ArrowType`, `default=arrow.utcnow`, `nullable=False`). This is the source of the JSON
  `creation_date` (`created_at.format()`) and `creation_timestamp` (`created_at.timestamp`).
- `updated_at =` (NULL) — `ModelMixin` `app/models.py:L65` (`default=None, onupdate=arrow.utcnow`);
  NULL on insert because no update has occurred.

### Rationale

The random-alias handler resolves an alias address, calls `Alias.create_new_random(...)`
(`app/models.py:L1721`) followed by `Session.commit()` (`app/api/views/new_random_alias.py:L107`),
then serializes the freshly-persisted row via `serialize_alias_info_v2`. Because both the JSON body
and the DB row are derived from the same `Alias` object, the client-visible `email` and the stored
`email` are guaranteed identical, and `mailbox_id` is the user's default mailbox by construction.


---

## Q5 — "If PostgreSQL isn't available when the app starts, what error appears and where does it break?"

**Answer:** the app raises **`sqlalchemy.exc.OperationalError`** wrapping the driver's
**`psycopg2.OperationalError`** (`Connection refused`). Critically, it breaks at
**module import time — before `create_app()` ever runs** — at
`app/db.py:L12` `connection = engine.connect()`, because the database connection is
established **eagerly** at module load. The SQLAlchemy help token is
`http://sqlalche.me/e/13/e3q8` (the `/e/13/` confirms the SQLAlchemy 1.3 line).

### Command run (database unreachable)

```bash
CONFIG=tests/test.env    # DB_URI from tests/test.env:L17 -> localhost:15432 (nothing listening)
unset DB_URI             # ensure the dead port 15432 is used, not a working override
python -c "import wsgi"  # wsgi.py:L1 does "from server import create_app"
```

Port 15432 was confirmed closed beforehand (`ConnectionRefusedError: [Errno 111] Connection refused`).

### Observed output (verbatim — banners, then the full traceback)

Import-time banners print first, then the connect fails:

```text
load config file /app/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/ceqrcotamiiwohkykcsd
Upload files to local dir
>>> init logging <<<
```

```text
Traceback (most recent call last):
  File "/app/venv/lib/python3.10/site-packages/sqlalchemy/engine/base.py", line 2336, in _wrap_pool_connect
    return fn()
  ...
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
  File "<string>", line 1, in <module>
  File "/app/wsgi.py", line 1, in <module>
    from server import create_app
  File "/app/server.py", line 31, in <module>
    from app.admin_model import (
  File "/app/app/admin_model.py", line 11, in <module>
    from app import models, s3
  File "/app/app/models.py", line 32, in <module>
    from app.db import Session
  File "/app/app/db.py", line 12, in <module>
    connection = engine.connect()
  ...
  File "/app/venv/lib/python3.10/site-packages/psycopg2/__init__.py", line 122, in connect
    conn = _connect(dsn, connection_factory=connection_factory, **kwasync)
sqlalchemy.exc.OperationalError: (psycopg2.OperationalError) connection to server at "localhost" (::1), port 15432 failed: Connection refused
	Is the server running on that host and accepting TCP/IP connections?
connection to server at "localhost" (127.0.0.1), port 15432 failed: Connection refused
	Is the server running on that host and accepting TCP/IP connections?

(Background on this error at: http://sqlalche.me/e/13/e3q8)
```

Programmatic confirmation of the exception identity and break location:

```text
EXC_TYPE           : sqlalchemy.exc.OperationalError
HAS_ORIG           : True
ORIG_TYPE          : psycopg2.OperationalError
SA_CODE_URL        : e3q8                     # -> http://sqlalche.me/e/13/e3q8
BREAK_FILE         : /app/app/db.py
BREAK_LINE         : 12
BREAK_CODE         : connection = engine.connect()
CREATE_APP_REACHED : False (exception raised during import, before wsgi.py:3 app = create_app())
```

### Explanation (file:line)

The failure is triggered by an **eager, module-scope** database connection:

- `app/db.py:L9-L11` builds the engine:
  `engine = create_engine(config.DB_URI, connect_args={"application_name": config.DB_CONN_NAME})`.
- `app/db.py:L12` `connection = engine.connect()` — this executes **at import time** (module scope),
  not lazily and not per request. **This is exactly where execution fails** (`BREAK_LINE : 12`).
- The import that reaches `app/db.py` is triggered from the WSGI entrypoint: `wsgi.py:L1`
  `from server import create_app`. As the observed traceback shows, `server.py` imports
  `app.db` transitively *before* `create_app()` is called — the chain is
  `wsgi.py:L1` → `server.py:L31` (`from app.admin_model import ...`) →
  `app/admin_model.py:L11` (`from app import models, s3`) → `app/models.py:L32`
  (`from app.db import Session`) → `app/db.py:L12` (`engine.connect()`).
  (`server.py:L76` also imports `from app.db import Session`, but the eager connect has already
  fired at the earlier `app/models.py:L32` import, so that is the first import to reach it.)

Exception identity:

- The driver raises `psycopg2.OperationalError` (message `... failed: Connection refused`).
  SQLAlchemy intercepts the DBAPI error and re-wraps it as `sqlalchemy.exc.OperationalError`,
  keeping the original accessible via `e.orig` (`ORIG_TYPE : psycopg2.OperationalError`).
- The help token `http://sqlalche.me/e/13/e3q8` — the `/e/13/` path segment corresponds to the
  **SQLAlchemy 1.3** line, matching the pinned `SQLAlchemy = "1.3.24"` (`pyproject.toml:L116`).

**libpq phrasing variance (documented, not over-asserted):** the exact `psycopg2` wording depends on
the image's bundled libpq. This image ships **libpq 140001 (v14, newer)**, which emits
`connection to server at "..." ... failed: Connection refused`. Older libpq versions instead emit
`could not connect to server: Connection refused`. Both are `psycopg2.OperationalError`; the wrapping
`sqlalchemy.exc.OperationalError` and the break location (`app/db.py:L12`) are unchanged.

### Rationale

Because the connection is opened at **import** (module scope in `app/db.py`), a missing database is a
**hard boot failure**: the process dies while importing the app, before `create_app()` runs and before
any request is served — rather than failing on the first HTTP request. That is the key behavioral
takeaway. (Per the read-only scope of this task, the eager `engine.connect()` at `app/db.py:L12` is
**documented here, not changed**.)


---

## Coverage pass

Every question and sub-part is answered with quoted, observed runtime evidence and
`file:line` grounding:

- [x] **Q1 — bind port.** `7777` for both runtimes; Gunicorn `0.0.0.0:7777` (all interfaces,
  proven by `Listening at: http://0.0.0.0:7777 (2876)`) vs dev server `127.0.0.1:7777`
  (loopback, `server.py:L588` `app.run(debug=True, port=7777)` with no `host=`).
  Anchors: `Dockerfile:L44,L47`, `wsgi.py:L1,L3`, `server.py:L588,L598-L599`.
- [x] **Q2 — startup logs.** Verbatim banners `load config file /app/tests/test.env`
  (`app/config.py:L68`), `>>> URL: http://localhost` (`app/config.py:L80`),
  `Upload files to local dir` (`app/config.py:L328`), `>>> init logging <<<` (`app/log.py:L67`),
  plus a real `SL - DEBUG` line (format `app/log.py:L12-L15`); werkzeug logger disabled
  (`app/log.py:L70-L71`); Gunicorn boot lines quoted; ordering explained
  (`app/log.py:L7-L9` imports `app.config`).
- [x] **Q3 — health body AND status.** Body `success` (**7 bytes**, `Content-Length: 7`),
  status **`200`**, `Content-Type: text/html; charset=utf-8` (`server.py:L213-L215`).
- [x] **Q4a — JSON response.** HTTP **`201 CREATED`**, `Content-Type: application/json`,
  full v2 body enumerated (`app/api/views/new_random_alias.py:L114-L117`,
  `app/api/serializer.py:L55-L79`); corroborated by `docs/api.md:L399-L412,L430-L458`.
- [x] **Q4b — database write.** Table **`alias`** (`app/models.py:L1470`); every persisted
  column value quoted from the real row (id `2677`, `email` matches the JSON exactly,
  `mailbox_id=1938` = user's `default_mailbox_id`, `enabled=t`, `flags=0`, `note`/`name`/`updated_at` NULL,
  `disable_pgp=f`, `pinned=f`, `automatic_creation=f`); insert path
  `app/models.py:L1721,L1750-L1754` + `app/api/views/new_random_alias.py:L107`.
- [x] **Q5 — DB unavailable at startup.** `sqlalchemy.exc.OperationalError` wrapping
  `psycopg2.OperationalError`, raised at **import time before `create_app()`** at
  `app/db.py:L12` `connection = engine.connect()`; help token `http://sqlalche.me/e/13/e3q8`
  (SQLAlchemy 1.3, `pyproject.toml:L116`); libpq phrasing-variance noted.
- [x] **Q6 — actual runtime output.** Every answer quotes captured evidence (log lines,
  HTTP responses + headers, DB rows, tracebacks) with the exact command that produced it.

**Read-only / cleanup note.** This task added exactly one file —
`blitzy/documentation/app_2cd6ee777f8c.md` — and modified no existing source, config, test,
or migration file. All temporary observation scripts and captures created during the
investigation were removed afterward. The eager `engine.connect()` at `app/db.py:L12` was
**documented, not changed**. After completion, `git status --porcelain` reports only this new
document (and its new `blitzy/documentation/` directory).

