# SimpleLogin — Runtime-Behavior Onboarding Q&A

This document answers five onboarding questions about **how the *running* SimpleLogin system
actually behaves at runtime** — not merely how the source reads. Every answer was produced by
**building and running the real code paths first**, then capturing the verbatim output. Each
claim is paired with (a) the exact command used, (b) the unedited observed output, (c) the
concrete observed value, (d) the responsible `file:line`, and (e) a short cause→effect
rationale. Where a statement is derived by reading code rather than observing runtime output,
it is explicitly labeled **inferred**.

**SimpleLogin** is an open-source, self-hostable email-aliasing / privacy service. It is a
**Flask/Python monolith** backed by **PostgreSQL** (SQLAlchemy ORM), served in production by
**Gunicorn** loading the WSGI callable `wsgi:app` [wsgi.py:3]. Application code lives under
`app/`; the REST API lives under `app/api/`.

## How to read this document

- **observed** — a value taken directly from real runtime output captured during this
  investigation (a log line, HTTP response, JSON body, SQL row, or error text).
- **inferred** — a conclusion drawn by reading the source code, explicitly labeled as such.
- Citations are inline as `[file:line]` and refer to the repository at git commit
  `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c` (branch `app_2cd6ee777f8c`), which is the exact
  code that was executed (baked into the image at `/app`, `git rev-parse HEAD` =
  `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`).
- **Credential hygiene.** All observations ran against a **disposable container** and a
  **throwaway database**. Secret values — the database password, `FLASK_SECRET`, the seeded
  user's password, and the API key — are **redacted** in this document (shown as `***` or a
  named variable). They existed only inside the container's environment and `mode-600` `/tmp`
  files, which were destroyed when the container was removed (see the final *Cleanup* section).
  No secret value is committed to the repository.

## Questions answered (direct answers, observed)

| # | Question | Direct answer (observed) |
|---|----------|--------------------------|
| Q1 | What TCP port does the app bind to on startup? | Port **`7777`** on both paths, on different interfaces: the canonical Gunicorn server binds `0.0.0.0:7777` (`-b 0.0.0.0:7777`); the development server binds `127.0.0.1:7777` (`app.run(..., port=7777)`, Werkzeug default `host`) |
| Q2 | What do the startup / initialization logs look like? | Gunicorn arbiter `[INFO]` lines on **stderr** + the app's import-time **stdout** printed **once per worker**, incl. the banner `>>> init logging <<<`. The `werkzeug` logger is disabled. |
| Q3 | What does the health-check endpoint return? | Body `success`, HTTP `200 OK`, `Content-Type: text/html; charset=utf-8`, `Content-Length: 7` |
| Q4 | Alias creation via REST API — JSON response + DB persistence? | HTTP `201`, a 17-key JSON object; a row is inserted into the `alias` table (API `id` == DB `id`) |
| Q5 | What happens if PostgreSQL is down at startup? | Master binds `:7777`; each worker raises `sqlalchemy.exc.OperationalError` (Connection refused) at `app/db.py:12`; `Reason: Worker failed to boot.`, exit code `3` |

---

## Environment / Build / Invocation

All observations were made inside the prescribed SimpleLogin Docker image, using the
**documented Python 3.10 runtime** [Dockerfile:8], [CONTRIBUTING.md:23] and the project's
Poetry-locked dependencies — i.e., the canonical configuration, not a newer host interpreter.

### Container identity and exact invocation

A **fresh, disposable** container was launched with an **init/reaper** (`--init`, so PID 1 is
`docker-init`/tini and reaps any orphaned child — no zombies accumulate). The port is not
published to the host; every request below is issued from **inside** the container.

```
$ docker inspect ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0 \
    --format 'RepoDigests={{json .RepoDigests}}'
RepoDigests=["ghcr.io/scaleapi/swe-atlas@sha256:b82cb15631e92ade58b8cf10493550f03a54dc5a8f25d3f31ef41afc186ee2c1"]

$ docker --version
Docker version 28.5.2, build ecc6942

$ docker run -d --init --name sl-fix --entrypoint sleep \
    ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0 infinity

$ docker exec sl-fix cat /proc/1/comm     # PID 1 is the reaper
docker-init

$ docker exec sl-fix bash -lc 'cd /app && git rev-parse HEAD'
2cd6ee777f8c2d3531559588bcfb18627ffb5d2c
```

The repository is baked into the image at **`/app`**; the Poetry virtualenv is at
`/app/venv`. The remaining commands are run inside this container (via `docker exec`).

### Runtime and dependency versions (observed)

```
$ /app/venv/bin/python --version
Python 3.10.18

$ /root/.local/bin/poetry --version
Poetry (version 2.1.4)

$ /app/venv/bin/pip list | grep -iE '^(alembic|arrow|coloredlogs|Flask|Flask-Migrate|gunicorn|Jinja2|psycopg2-binary|SQLAlchemy|Werkzeug) '
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
`psycopg2-binary = "^2.9.3"` [pyproject.toml:71], `Flask-Migrate = "^2.5.3"`
[pyproject.toml:77], `coloredlogs = "^14.0"` [pyproject.toml:89],
`arrow = "^0.16.0"` [pyproject.toml:74], `SQLAlchemy = "1.3.24"` [pyproject.toml:116]) and
`poetry.lock`. Poetry itself (2.1.4) is only the dependency manager; the app is run directly
via `/app/venv/bin/{python,gunicorn,alembic}`.

### PostgreSQL provisioning (observed)

The image ships PostgreSQL 15 (cluster `15/main`). It is started, then a dedicated login role
and a **fresh, isolated throwaway database** (`slobs`) are created for this investigation. The
fresh database is **not** given the `pg_trgm` extension — the migration creates it (see the
migration note in Q4).

```
$ postgres --version   # /usr/lib/postgresql/15/bin/postgres
postgres (PostgreSQL) 15.13 (Debian 15.13-0+deb12u1)
$ psql --version
psql (PostgreSQL) 15.13 (Debian 15.13-0+deb12u1)

$ pg_ctlcluster 15 main start        # start the cluster on :5432
$ pg_isready -h localhost -p 5432
localhost:5432 - accepting connections

# create the login role (password taken from the example.env template, value not shown)
# and a fresh, isolated throwaway database owned by that role:
$ su postgres -c "psql -tAc \"CREATE ROLE myuser LOGIN PASSWORD '***';\""     # value redacted
$ su postgres -c "psql -tAc \"DROP DATABASE IF EXISTS slobs;\""
$ su postgres -c "psql -tAc \"CREATE DATABASE slobs OWNER myuser;\""

# prove the fresh DB has NO pg_trgm pre-created (only the default plpgsql):
$ su postgres -c "psql -d slobs -tAc \"SELECT extname FROM pg_extension ORDER BY 1;\""
plpgsql
```

**Version note (observed vs documented).** The PostgreSQL server/client used here is
**15.13** (`PostgreSQL 15.13 (Debian 15.13-0+deb12u1)`), whereas the project documents
**Postgres 13+** [CONTRIBUTING.md:25]. The values reported below (port bind, health response,
alias JSON/row, and the connection-refused startup error) were observed **only** on 15.13; that
they are identical on other supported major versions is **inferred**, not observed here, and is
stated as a limited inference rather than a cross-version claim.

### Configuration file (CONFIG, kept outside the repository)

The app reads all configuration from a `CONFIG` env-file [app/config.py:65],
[app/config.py:69]. A copy of `example.env` was placed at an absolute `/tmp` path (mode `600`,
**outside** the repository so the source tree stays byte-for-byte unchanged), with `DB_URI`
repointed at the throwaway `slobs` database. The required keys, with **secret values redacted**:

```
$ cp /app/example.env /tmp/obs/sl_obs.env && chmod 600 /tmp/obs/sl_obs.env
$ sed -i 's#^DB_URI=.*#DB_URI=<throwaway slobs URI>#' /tmp/obs/sl_obs.env
# required keys (secret values redacted; keys are set from the example.env template):
URL=http://localhost:7777                                    # [example.env:6],  [app/config.py:79]
EMAIL_DOMAIN=sl.local                                        # [example.env:22], [app/config.py:92]
SUPPORT_EMAIL=support@sl.local                               # [example.env:40], [app/config.py:93]
WORDS_FILE_PATH=local_data/test_words.txt                    # [example.env:97]
DB_URI=postgresql://***:***@localhost:5432/slobs   (redacted) # [example.env:75]
FLASK_SECRET=***                                   (redacted) # [example.env:77], [app/config.py:196]
```

> Reproducibility & safety conventions used throughout: every observation script runs under
> `set -uo pipefail`; server processes are started in the background with their **exact PID
> captured** (`pid=$!`), probed for readiness against `/health`, then stopped with
> `kill -TERM "$pid"` followed by `wait "$pid"` so no process is orphaned; secrets are read
> from `mode-600` `/tmp` files into shell variables (`$DB_URI`, `$API_KEY`) and never echoed;
> and the whole run is confined to a disposable container plus a dedicated `slobs` database.

### Schema build (Alembic) — complete output

The full schema is built by running Alembic to head against the fresh `slobs` database. The
**complete, unedited** output follows (no ellipses); the step count is then measured with real
commands rather than asserted.

```
$ cd /app && CONFIG=/tmp/obs/sl_obs.env /app/venv/bin/alembic upgrade head
load config file /tmp/obs/sl_obs.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/pkkiiiwkriutvsjpuwey
Upload files to local dir
>>> init logging <<<
2026-07-13 17:20:05,256 - SL - DEBUG - 224 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade  -> 5e549314e1e2, empty message
INFO  [alembic.runtime.migration] Running upgrade 5e549314e1e2 -> 3cd10cfce8c3, empty message
INFO  [alembic.runtime.migration] Running upgrade 3cd10cfce8c3 -> 0256244cd7c8, empty message
INFO  [alembic.runtime.migration] Running upgrade 0256244cd7c8 -> 213fcca48483, empty message
INFO  [alembic.runtime.migration] Running upgrade 213fcca48483 -> f234688f5ebd, empty message
INFO  [alembic.runtime.migration] Running upgrade f234688f5ebd -> d03e433dc248, empty message
INFO  [alembic.runtime.migration] Running upgrade d03e433dc248 -> 2fe19381f386, empty message
INFO  [alembic.runtime.migration] Running upgrade 2fe19381f386 -> b20ee72fd9a4, empty message
INFO  [alembic.runtime.migration] Running upgrade b20ee72fd9a4 -> 590d89f981c0, empty message
INFO  [alembic.runtime.migration] Running upgrade 590d89f981c0 -> 551c4e6d4a8b, empty message
INFO  [alembic.runtime.migration] Running upgrade 551c4e6d4a8b -> c6e7fc37ad42, empty message
INFO  [alembic.runtime.migration] Running upgrade c6e7fc37ad42 -> 1b7d161d1012, empty message
INFO  [alembic.runtime.migration] Running upgrade 1b7d161d1012 -> 507afb2632cc, empty message
INFO  [alembic.runtime.migration] Running upgrade 507afb2632cc -> 4fac8c8a704c, empty message
INFO  [alembic.runtime.migration] Running upgrade 4fac8c8a704c -> c79c702a1f23, empty message
INFO  [alembic.runtime.migration] Running upgrade c79c702a1f23 -> 5fa68bafae72, empty message
INFO  [alembic.runtime.migration] Running upgrade 5fa68bafae72 -> 4a640c170d02, empty message
INFO  [alembic.runtime.migration] Running upgrade 4a640c170d02 -> 2e2b53afd819, empty message
INFO  [alembic.runtime.migration] Running upgrade 2e2b53afd819 -> 6bbda4685999, empty message
INFO  [alembic.runtime.migration] Running upgrade 6bbda4685999 -> d68a2d971b70, empty message
INFO  [alembic.runtime.migration] Running upgrade d68a2d971b70 -> 0a89c670fc7a, empty message
INFO  [alembic.runtime.migration] Running upgrade 0a89c670fc7a -> 3ebfbaeb76c0, empty message
INFO  [alembic.runtime.migration] Running upgrade 3ebfbaeb76c0 -> 83f4dbe125c4, empty message
INFO  [alembic.runtime.migration] Running upgrade 83f4dbe125c4 -> e505cb517589, empty message
INFO  [alembic.runtime.migration] Running upgrade e505cb517589 -> e83298198ca5, empty message
INFO  [alembic.runtime.migration] Running upgrade e83298198ca5 -> 3a87573bf8a8, empty message
INFO  [alembic.runtime.migration] Running upgrade 3a87573bf8a8 -> a8d8aa307b8b, empty message
INFO  [alembic.runtime.migration] Running upgrade a8d8aa307b8b -> 0b28518684ae, empty message
INFO  [alembic.runtime.migration] Running upgrade 0b28518684ae -> 5e868298fee7, empty message
INFO  [alembic.runtime.migration] Running upgrade 5e868298fee7 -> 2d2fc3e826af, empty message
INFO  [alembic.runtime.migration] Running upgrade 2d2fc3e826af -> 0c7f1a48aac9, empty message
INFO  [alembic.runtime.migration] Running upgrade 0c7f1a48aac9 -> 18e934d58f55, empty message
INFO  [alembic.runtime.migration] Running upgrade 18e934d58f55 -> 9e1b06b9df13, empty message
INFO  [alembic.runtime.migration] Running upgrade 9e1b06b9df13 -> d4e4488a0032, empty message
INFO  [alembic.runtime.migration] Running upgrade d4e4488a0032 -> e409f6214b2b, empty message
INFO  [alembic.runtime.migration] Running upgrade e409f6214b2b -> 696e17c13b8b, empty message
INFO  [alembic.runtime.migration] Running upgrade 696e17c13b8b -> a8b996f0be40, empty message
INFO  [alembic.runtime.migration] Running upgrade a8b996f0be40 -> 10ad2dbaeccf, empty message
INFO  [alembic.runtime.migration] Running upgrade 10ad2dbaeccf -> 01f808f15b2e, empty message
INFO  [alembic.runtime.migration] Running upgrade 01f808f15b2e -> d29cca963221, empty message
INFO  [alembic.runtime.migration] Running upgrade d29cca963221 -> ba6f13ccbabb, empty message
INFO  [alembic.runtime.migration] Running upgrade ba6f13ccbabb -> 7c39ba4ec38d, empty message
INFO  [alembic.runtime.migration] Running upgrade 7c39ba4ec38d -> 9c976df9b9c4, empty message
INFO  [alembic.runtime.migration] Running upgrade 9c976df9b9c4 -> b9f849432543, empty message
INFO  [alembic.runtime.migration] Running upgrade b9f849432543 -> 6664d75ce3d4, empty message
INFO  [alembic.runtime.migration] Running upgrade 6664d75ce3d4 -> 3c9542fc54e9, empty message
INFO  [alembic.runtime.migration] Running upgrade 3c9542fc54e9 -> 3fa3a648c8e7, empty message
INFO  [alembic.runtime.migration] Running upgrade 3fa3a648c8e7 -> 903ec5f566e8, empty message
INFO  [alembic.runtime.migration] Running upgrade 903ec5f566e8 -> f580030d9beb, empty message
INFO  [alembic.runtime.migration] Running upgrade f580030d9beb -> e3cb44b953f2, empty message
INFO  [alembic.runtime.migration] Running upgrade e3cb44b953f2 -> 75093e7ded27, empty message
INFO  [alembic.runtime.migration] Running upgrade 75093e7ded27 -> 5f191273d067, empty message
INFO  [alembic.runtime.migration] Running upgrade 5f191273d067 -> 7eef64ffb398, empty message
INFO  [alembic.runtime.migration] Running upgrade 7eef64ffb398 -> 235355381f53, empty message
INFO  [alembic.runtime.migration] Running upgrade 235355381f53 -> 628a5438295c, empty message
INFO  [alembic.runtime.migration] Running upgrade 628a5438295c -> 11a35b448f83, empty message
INFO  [alembic.runtime.migration] Running upgrade 11a35b448f83 -> 9081f1a90939, empty message
INFO  [alembic.runtime.migration] Running upgrade 9081f1a90939 -> 91b69dfad2f1, empty message
INFO  [alembic.runtime.migration] Running upgrade 91b69dfad2f1 -> 7744c5c16159, empty message
INFO  [alembic.runtime.migration] Running upgrade 7744c5c16159 -> 14167121af69, empty message
INFO  [alembic.runtime.migration] Running upgrade 14167121af69 -> 6e061eb84167, empty message
INFO  [alembic.runtime.migration] Running upgrade 6e061eb84167 -> e9395fe234a4, empty message
INFO  [alembic.runtime.migration] Running upgrade e9395fe234a4 -> 0809266d08ca, empty message
INFO  [alembic.runtime.migration] Running upgrade 0809266d08ca -> f4b8232fa17e, empty message
INFO  [alembic.runtime.migration] Running upgrade f4b8232fa17e -> dbd80d290f04, empty message
INFO  [alembic.runtime.migration] Running upgrade dbd80d290f04 -> 4e4a759ac4b5, empty message
INFO  [alembic.runtime.migration] Running upgrade 4e4a759ac4b5 -> 30c13ca016e4, empty message
INFO  [alembic.runtime.migration] Running upgrade 30c13ca016e4 -> 541ce53ab6e9, empty message
INFO  [alembic.runtime.migration] Running upgrade 541ce53ab6e9 -> 67c61eead8d2, empty message
INFO  [alembic.runtime.migration] Running upgrade 67c61eead8d2 -> 224fd8963462, empty message
INFO  [alembic.runtime.migration] Running upgrade 224fd8963462 -> 92baf66b268b, empty message
INFO  [alembic.runtime.migration] Running upgrade 92baf66b268b -> 497cfd2a02e2, empty message
INFO  [alembic.runtime.migration] Running upgrade 497cfd2a02e2 -> ea30c0b5b2e3, empty message
INFO  [alembic.runtime.migration] Running upgrade ea30c0b5b2e3 -> bfd7b2302903, empty message
INFO  [alembic.runtime.migration] Running upgrade bfd7b2302903 -> 57ef03f3ac34, empty message
INFO  [alembic.runtime.migration] Running upgrade 57ef03f3ac34 -> dd911f880b75, empty message
INFO  [alembic.runtime.migration] Running upgrade dd911f880b75 -> bd05eac83f5f, empty message
INFO  [alembic.runtime.migration] Running upgrade bd05eac83f5f -> b4146f7d5277, empty message
INFO  [alembic.runtime.migration] Running upgrade b4146f7d5277 -> f939d67374e4, empty message
INFO  [alembic.runtime.migration] Running upgrade f939d67374e4 -> de1b457472e0, empty message
INFO  [alembic.runtime.migration] Running upgrade de1b457472e0 -> ae94fe5c4e9f, empty message
INFO  [alembic.runtime.migration] Running upgrade ae94fe5c4e9f -> 026e7a782ed6, empty message
INFO  [alembic.runtime.migration] Running upgrade 026e7a782ed6 -> 925b93d92809, empty message
INFO  [alembic.runtime.migration] Running upgrade 925b93d92809 -> bdf76f4b65a2, empty message
INFO  [alembic.runtime.migration] Running upgrade bdf76f4b65a2 -> a3a7c518ea70, empty message
INFO  [alembic.runtime.migration] Running upgrade a3a7c518ea70 -> a5e3c6693dc6, empty message
INFO  [alembic.runtime.migration] Running upgrade a5e3c6693dc6 -> bf11ab2f0a7a, empty message
INFO  [alembic.runtime.migration] Running upgrade bf11ab2f0a7a -> 1759f73274ee, empty message
INFO  [alembic.runtime.migration] Running upgrade 1759f73274ee -> 552d735a2f1f, empty message
INFO  [alembic.runtime.migration] Running upgrade 552d735a2f1f -> 5cad8fa84386, empty message
INFO  [alembic.runtime.migration] Running upgrade 5cad8fa84386 -> c31cdf879ee3, empty message
INFO  [alembic.runtime.migration] Running upgrade c31cdf879ee3 -> 659d979b64ce, empty message
INFO  [alembic.runtime.migration] Running upgrade 659d979b64ce -> ce15cf3467b4, empty message
INFO  [alembic.runtime.migration] Running upgrade ce15cf3467b4 -> 0e08145f0499, empty message
INFO  [alembic.runtime.migration] Running upgrade 0e08145f0499 -> 00532ac6d4bc, empty message
INFO  [alembic.runtime.migration] Running upgrade 00532ac6d4bc -> f680032cc361, empty message
INFO  [alembic.runtime.migration] Running upgrade f680032cc361 -> 10a7947fda6b, empty message
INFO  [alembic.runtime.migration] Running upgrade 10a7947fda6b -> 4a7d35941602, empty message
INFO  [alembic.runtime.migration] Running upgrade 4a7d35941602 -> cfc013b6461a, empty message
INFO  [alembic.runtime.migration] Running upgrade cfc013b6461a -> b2d51e4d94c8, empty message
INFO  [alembic.runtime.migration] Running upgrade b2d51e4d94c8 -> 749c2b85d20f, empty message
INFO  [alembic.runtime.migration] Running upgrade 749c2b85d20f -> a5b4dc311a89, empty message
INFO  [alembic.runtime.migration] Running upgrade a5b4dc311a89 -> a3c9a43e41f4, empty message
INFO  [alembic.runtime.migration] Running upgrade a3c9a43e41f4 -> 7128f87af701, empty message
INFO  [alembic.runtime.migration] Running upgrade 7128f87af701 -> 270d598c51e3, empty message
INFO  [alembic.runtime.migration] Running upgrade 270d598c51e3 -> b77ab8c47cc7, empty message
INFO  [alembic.runtime.migration] Running upgrade b77ab8c47cc7 -> a2b95b04d1f7, empty message
INFO  [alembic.runtime.migration] Running upgrade a2b95b04d1f7 -> 63fd3b240583, empty message
INFO  [alembic.runtime.migration] Running upgrade 63fd3b240583 -> 95938a93ea14, empty message
INFO  [alembic.runtime.migration] Running upgrade 95938a93ea14 -> b82bcad9accf, empty message
INFO  [alembic.runtime.migration] Running upgrade b82bcad9accf -> 84471852b610, empty message
INFO  [alembic.runtime.migration] Running upgrade 84471852b610 -> b0e9a389939a, empty message
INFO  [alembic.runtime.migration] Running upgrade b0e9a389939a -> 198c3aca9d8d, empty message
INFO  [alembic.runtime.migration] Running upgrade 198c3aca9d8d -> 58ad4df8583e, empty message
INFO  [alembic.runtime.migration] Running upgrade 58ad4df8583e -> 1abfc9e14d7e, empty message
INFO  [alembic.runtime.migration] Running upgrade 1abfc9e14d7e -> 32b00d06d892, empty message
INFO  [alembic.runtime.migration] Running upgrade 32b00d06d892 -> b17afc77ba83, empty message
INFO  [alembic.runtime.migration] Running upgrade b17afc77ba83 -> 54ca2dbf89c0, empty message
INFO  [alembic.runtime.migration] Running upgrade 54ca2dbf89c0 -> eef0c404b531, empty message
INFO  [alembic.runtime.migration] Running upgrade eef0c404b531 -> 84dec6c29c48, empty message
INFO  [alembic.runtime.migration] Running upgrade 84dec6c29c48 -> d0f197979bd9, empty message
INFO  [alembic.runtime.migration] Running upgrade d0f197979bd9 -> 9dc16e591f88, empty message
INFO  [alembic.runtime.migration] Running upgrade 9dc16e591f88 -> ac41029fb329, empty message
INFO  [alembic.runtime.migration] Running upgrade ac41029fb329 -> d1edb3cadec8, empty message
INFO  [alembic.runtime.migration] Running upgrade d1edb3cadec8 -> 623662ea0e7e, empty message
INFO  [alembic.runtime.migration] Running upgrade 623662ea0e7e -> 56c790ec8ab4, empty message
INFO  [alembic.runtime.migration] Running upgrade 56c790ec8ab4 -> c0d91ff18f77, empty message
INFO  [alembic.runtime.migration] Running upgrade c0d91ff18f77 -> 780a8344914b, empty message
INFO  [alembic.runtime.migration] Running upgrade 780a8344914b -> a20aeb9b0eac, empty message
INFO  [alembic.runtime.migration] Running upgrade a20aeb9b0eac -> 0af2c2e286a7, empty message
INFO  [alembic.runtime.migration] Running upgrade 0af2c2e286a7 -> 1919f1859215, empty message
INFO  [alembic.runtime.migration] Running upgrade 1919f1859215 -> f66ca777f409, empty message
INFO  [alembic.runtime.migration] Running upgrade f66ca777f409 -> 7c0dbd378cdb, empty message
INFO  [alembic.runtime.migration] Running upgrade 7c0dbd378cdb -> e99989e6ad56, empty message
INFO  [alembic.runtime.migration] Running upgrade e99989e6ad56 -> 1b54995bc086, empty message
INFO  [alembic.runtime.migration] Running upgrade 1b54995bc086 -> 2779eb90c6c4, empty message
INFO  [alembic.runtime.migration] Running upgrade 2779eb90c6c4 -> 74906d31d994, empty message
INFO  [alembic.runtime.migration] Running upgrade 74906d31d994 -> 85d0655d42c0, empty message
INFO  [alembic.runtime.migration] Running upgrade 85d0655d42c0 -> de7aa5280210, empty message
INFO  [alembic.runtime.migration] Running upgrade de7aa5280210 -> e831a883153a, empty message
INFO  [alembic.runtime.migration] Running upgrade e831a883153a -> d1236c4dff71, empty message
INFO  [alembic.runtime.migration] Running upgrade d1236c4dff71 -> 94f14eb0fe5b, empty message
INFO  [alembic.runtime.migration] Running upgrade 94f14eb0fe5b -> 9d6adad83936, empty message
INFO  [alembic.runtime.migration] Running upgrade 9d6adad83936 -> f398b261d9c6, empty message
INFO  [alembic.runtime.migration] Running upgrade f398b261d9c6 -> 517b79c56088, empty message
INFO  [alembic.runtime.migration] Running upgrade 517b79c56088 -> 48b991e9de06, empty message
INFO  [alembic.runtime.migration] Running upgrade 48b991e9de06 -> e11c3dd48a6f, empty message
INFO  [alembic.runtime.migration] Running upgrade e11c3dd48a6f -> 4912f3bd5ba2, empty message
INFO  [alembic.runtime.migration] Running upgrade 4912f3bd5ba2 -> f5133dc851ee, empty message
INFO  [alembic.runtime.migration] Running upgrade f5133dc851ee -> 5c77d685df87, empty message
INFO  [alembic.runtime.migration] Running upgrade 5c77d685df87 -> 6cc7f073b358, empty message
INFO  [alembic.runtime.migration] Running upgrade 6cc7f073b358 -> 68e2f38e33f4, empty message
INFO  [alembic.runtime.migration] Running upgrade 68e2f38e33f4 -> fc2eb1d7e4fc, empty message
INFO  [alembic.runtime.migration] Running upgrade fc2eb1d7e4fc -> a5e643d562c9, empty message
INFO  [alembic.runtime.migration] Running upgrade a5e643d562c9 -> 29ea13ed76f9, empty message
INFO  [alembic.runtime.migration] Running upgrade 29ea13ed76f9 -> 8e70205a5308, empty message
INFO  [alembic.runtime.migration] Running upgrade 8e70205a5308 -> f3f19998b755, empty message
INFO  [alembic.runtime.migration] Running upgrade f3f19998b755 -> c31a081eab74, empty message
INFO  [alembic.runtime.migration] Running upgrade c31a081eab74 -> 78403c7b8089, empty message
INFO  [alembic.runtime.migration] Running upgrade 78403c7b8089 -> 5662122eac21, empty message
INFO  [alembic.runtime.migration] Running upgrade 5662122eac21 -> 20c738810b1b, empty message
INFO  [alembic.runtime.migration] Running upgrade 20c738810b1b -> dfee471558bd, empty message
INFO  [alembic.runtime.migration] Running upgrade dfee471558bd -> 05e3af59929a, empty message
INFO  [alembic.runtime.migration] Running upgrade 05e3af59929a -> c3470e2d3224, empty message
INFO  [alembic.runtime.migration] Running upgrade c3470e2d3224 -> ffa75d04e6ef, empty message
INFO  [alembic.runtime.migration] Running upgrade ffa75d04e6ef -> 9014cca7097c, empty message
INFO  [alembic.runtime.migration] Running upgrade 9014cca7097c -> d4392342465f, empty message
INFO  [alembic.runtime.migration] Running upgrade d4392342465f -> 424808e1fe49, empty message
INFO  [alembic.runtime.migration] Running upgrade 424808e1fe49 -> 916a5257d18c, empty message
INFO  [alembic.runtime.migration] Running upgrade 916a5257d18c -> 4d3f91ddf3e9, empty message
INFO  [alembic.runtime.migration] Running upgrade 4d3f91ddf3e9 -> d8c55e79da54, empty message
INFO  [alembic.runtime.migration] Running upgrade d8c55e79da54 -> cf1e8c1bc737, empty message
INFO  [alembic.runtime.migration] Running upgrade cf1e8c1bc737 -> 7a105bfc0cd0, empty message
INFO  [alembic.runtime.migration] Running upgrade 7a105bfc0cd0 -> bc75acacc98e, empty message
INFO  [alembic.runtime.migration] Running upgrade bc75acacc98e -> b8b4f9598240, empty message
INFO  [alembic.runtime.migration] Running upgrade b8b4f9598240 -> 5ee767807344, empty message
INFO  [alembic.runtime.migration] Running upgrade 5ee767807344 -> 4913cb3f5a05, empty message
INFO  [alembic.runtime.migration] Running upgrade 4913cb3f5a05 -> 0b1c9ea11aef, empty message
INFO  [alembic.runtime.migration] Running upgrade 0b1c9ea11aef -> 2fbcad5527d7, empty message
INFO  [alembic.runtime.migration] Running upgrade 2fbcad5527d7 -> d750d578b068, empty message
INFO  [alembic.runtime.migration] Running upgrade d750d578b068 -> 2f1b3c759773, empty message
INFO  [alembic.runtime.migration] Running upgrade 2f1b3c759773 -> 99d9e329b27f, empty message
INFO  [alembic.runtime.migration] Running upgrade 99d9e329b27f -> a06066e3fbeb, empty message
INFO  [alembic.runtime.migration] Running upgrade a06066e3fbeb -> d67eab226ecd, empty message
INFO  [alembic.runtime.migration] Running upgrade d67eab226ecd -> bbedc353f90c, empty message
INFO  [alembic.runtime.migration] Running upgrade bbedc353f90c -> 0b9150eb309d, Increase message_id length manually
INFO  [alembic.runtime.migration] Running upgrade 0b9150eb309d -> 6204e57b4bc4, empty message
INFO  [alembic.runtime.migration] Running upgrade 6204e57b4bc4 -> 37feaba7c45d, empty message
INFO  [alembic.runtime.migration] Running upgrade 37feaba7c45d -> ff6c04869029, empty message
INFO  [alembic.runtime.migration] Running upgrade ff6c04869029 -> fdb02bd105a8, empty message
INFO  [alembic.runtime.migration] Running upgrade fdb02bd105a8 -> dd278f96ca83, empty message
INFO  [alembic.runtime.migration] Running upgrade dd278f96ca83 -> 1076b5795b08, empty message
INFO  [alembic.runtime.migration] Running upgrade 1076b5795b08 -> 5639ad89ee50, empty message
INFO  [alembic.runtime.migration] Running upgrade 5639ad89ee50 -> 11ba83e2dd71, empty message
INFO  [alembic.runtime.migration] Running upgrade 11ba83e2dd71 -> ccbfb61eda0d, empty message
INFO  [alembic.runtime.migration] Running upgrade ccbfb61eda0d -> a5013ff0a00a, empty message
INFO  [alembic.runtime.migration] Running upgrade a5013ff0a00a -> e6e8e12f5a13, empty message
INFO  [alembic.runtime.migration] Running upgrade e6e8e12f5a13 -> 9031c9e28510, empty message
INFO  [alembic.runtime.migration] Running upgrade 9031c9e28510 -> b8fd175c084a, empty message
INFO  [alembic.runtime.migration] Running upgrade b8fd175c084a -> e7d7ebcea26c, empty message
INFO  [alembic.runtime.migration] Running upgrade e7d7ebcea26c -> d0ccd9d7ac0c, empty message
INFO  [alembic.runtime.migration] Running upgrade d0ccd9d7ac0c -> 4b483a762fed, empty message
INFO  [alembic.runtime.migration] Running upgrade 4b483a762fed -> ad467baf7ec8, empty message
INFO  [alembic.runtime.migration] Running upgrade ad467baf7ec8 -> d8a3dfe674f2, empty message
INFO  [alembic.runtime.migration] Running upgrade d8a3dfe674f2 -> 3d05479d0d11, empty message
INFO  [alembic.runtime.migration] Running upgrade 3d05479d0d11 -> 753d2ed92d41, empty message
INFO  [alembic.runtime.migration] Running upgrade 753d2ed92d41 -> 698424c429e9, empty message
INFO  [alembic.runtime.migration] Running upgrade 698424c429e9 -> 07b870d7cc86, empty message
INFO  [alembic.runtime.migration] Running upgrade 07b870d7cc86 -> 9282e982bc05, Add block_behaviour setting for user
INFO  [alembic.runtime.migration] Running upgrade 9282e982bc05 -> 5047fcbd57c7, empty message
INFO  [alembic.runtime.migration] Running upgrade 5047fcbd57c7 -> 4729b7096d12, empty message
INFO  [alembic.runtime.migration] Running upgrade 4729b7096d12 -> b500363567e3, Create admin audit log
INFO  [alembic.runtime.migration] Running upgrade b500363567e3 -> 28b9b14c9664, store provider complaints
INFO  [alembic.runtime.migration] Running upgrade 28b9b14c9664 -> 0aaad1740797, store provider complaints
INFO  [alembic.runtime.migration] Running upgrade 0aaad1740797 -> e866ad0e78e1, Add partner tables
INFO  [alembic.runtime.migration] Running upgrade e866ad0e78e1 -> 088f23324464, add flags to the user model
INFO  [alembic.runtime.migration] Running upgrade 088f23324464 -> 2b1d3cd93e4b, update partner_api_token token length
INFO  [alembic.runtime.migration] Running upgrade 2b1d3cd93e4b -> 82d3c7109ffb, partner_user and partner_subscription
INFO  [alembic.runtime.migration] Running upgrade 82d3c7109ffb -> 36646e5dc6d9, make external_user_id non nullable
INFO  [alembic.runtime.migration] Running upgrade 36646e5dc6d9 -> a7bcb872c12a, Add alias transfer token expiration
INFO  [alembic.runtime.migration] Running upgrade a7bcb872c12a -> 673a074e4215, empty message
INFO  [alembic.runtime.migration] Running upgrade 673a074e4215 -> d1fb679f7eec, Add sudo expiration for ApiKeys
INFO  [alembic.runtime.migration] Running upgrade d1fb679f7eec -> bfebc2d5c719, Add state to job
INFO  [alembic.runtime.migration] Running upgrade bfebc2d5c719 -> 516c21ea7d87, empty message
INFO  [alembic.runtime.migration] Running upgrade 516c21ea7d87 -> bd7d032087b2, empty message
INFO  [alembic.runtime.migration] Running upgrade bd7d032087b2 -> b0101a66bb77, Add unsubscribe behaviour
INFO  [alembic.runtime.migration] Running upgrade b0101a66bb77 -> 89081a00fc7d, default_unsub_behaviour
INFO  [alembic.runtime.migration] Running upgrade 89081a00fc7d -> c66f2c5b6cb1, empty message
INFO  [alembic.runtime.migration] Running upgrade c66f2c5b6cb1 -> 9cc0f0712b29, Add api to cookie token
INFO  [alembic.runtime.migration] Running upgrade 9cc0f0712b29 -> bd95b2b4217f, Updated recovery code string length
INFO  [alembic.runtime.migration] Running upgrade bd95b2b4217f -> 2c2093c82bc0, empty message
INFO  [alembic.runtime.migration] Running upgrade 2c2093c82bc0 -> 5f4a5625da66, empty message
INFO  [alembic.runtime.migration] Running upgrade 5f4a5625da66 -> 893c0d18475f, empty message
INFO  [alembic.runtime.migration] Running upgrade 893c0d18475f -> bc496c0a0279, empty message
INFO  [alembic.runtime.migration] Running upgrade bc496c0a0279 -> 2d89315ac650, empty message
INFO  [alembic.runtime.migration] Running upgrade 893c0d18475f -> 01e2997e90d3, empty message
INFO  [alembic.runtime.migration] Running upgrade 01e2997e90d3, 2d89315ac650 -> 2634b41f54db, empty message
INFO  [alembic.runtime.migration] Running upgrade 2634b41f54db -> 01827104004b, empty message
INFO  [alembic.runtime.migration] Running upgrade 01827104004b -> 0a5701a4f5e4, empty message
INFO  [alembic.runtime.migration] Running upgrade 0a5701a4f5e4 -> ec7fdde8da9f, empty message
INFO  [alembic.runtime.migration] Running upgrade ec7fdde8da9f -> 46ecb648a47e, empty message
INFO  [alembic.runtime.migration] Running upgrade 46ecb648a47e -> 4bc54632d9aa, empty message
INFO  [alembic.runtime.migration] Running upgrade 4bc54632d9aa -> 818b0a956205, empty message
INFO  [alembic.runtime.migration] Running upgrade 818b0a956205 -> 52510a633d6f, empty message
INFO  [alembic.runtime.migration] Running upgrade 52510a633d6f -> fa2f19bb4e5a, empty message
INFO  [alembic.runtime.migration] Running upgrade fa2f19bb4e5a -> 06a9a7133445, Create sync_event table
INFO  [alembic.runtime.migration] Running upgrade 06a9a7133445 -> d608b8e48082, empty message
INFO  [alembic.runtime.migration] Running upgrade d608b8e48082 -> 56d08955fcab, add retry count to sync event
INFO  [alembic.runtime.migration] Running upgrade 56d08955fcab -> 1c14339aae90, empty message
INFO  [alembic.runtime.migration] Running upgrade 1c14339aae90 -> 2441b7ff5da9, Custom Domain partner id
INFO  [alembic.runtime.migration] Running upgrade 2441b7ff5da9 -> 88dd7a0abf54, contact.flags and custom_domain.pending_deletion
INFO  [alembic.runtime.migration] Running upgrade 88dd7a0abf54 -> 62afa3a10010, custom domain indices
INFO  [alembic.runtime.migration] Running upgrade 62afa3a10010 -> 91ed7f46dc81, alias_audit_log
INFO  [alembic.runtime.migration] Running upgrade 91ed7f46dc81 -> 7d7b84779837, user_audit_log
INFO  [alembic.runtime.migration] Running upgrade 7d7b84779837 -> 32f25cbf12f6, alias_audit_log_index_created_at
```

```
$ CONFIG=/tmp/obs/sl_obs.env /app/venv/bin/alembic upgrade head > /tmp/obs/alembic_upgrade.txt 2>&1 ; echo "exit=$?"
exit=0
$ wc -l < /tmp/obs/alembic_upgrade.txt
265
$ grep -c 'Running upgrade' /tmp/obs/alembic_upgrade.txt
255
```

So the migration completed with **exit code 0**, producing **265** output lines: 8 lines of
the app's import-time stdout (see Q2), 2 Alembic context lines, and **255** `Running upgrade`
steps — the last advancing to `32f25cbf12f6`. `alembic current` confirms head:

```
$ CONFIG=/tmp/obs/sl_obs.env /app/venv/bin/alembic current
load config file /tmp/obs/sl_obs.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/tynnsvtbcylbbccstwky
Upload files to local dir
>>> init logging <<<
2026-07-13 17:20:07,423 - SL - DEBUG - 231 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
32f25cbf12f6 (head)
```

### Schema verification — the `alias` table exists

```
$ psql "$DB_URI" -c "\d alias"
                                                                   Table "public.alias"
            Column            |            Type             | Collation | Nullable |                               Default
------------------------------+-----------------------------+-----------+----------+----------------------------------------------------------------------
 id                           | integer                     |           | not null | nextval('gen_email_id_seq'::regclass)
 created_at                   | timestamp without time zone |           | not null |
 updated_at                   | timestamp without time zone |           |          |
 user_id                      | integer                     |           | not null |
 email                        | character varying(128)      |           | not null |
 enabled                      | boolean                     |           | not null |
 custom_domain_id             | integer                     |           |          |
 automatic_creation           | boolean                     |           | not null | false
 directory_id                 | integer                     |           |          |
 note                         | text                        |           |          |
 mailbox_id                   | integer                     |           | not null |
 name                         | character varying(128)      |           |          |
 disable_pgp                  | boolean                     |           | not null | false
 cannot_be_disabled           | boolean                     |           | not null | false
 disable_email_spoofing_check | boolean                     |           | not null | false
 batch_import_id              | integer                     |           |          |
 pinned                       | boolean                     |           | not null | false
 original_owner_id            | integer                     |           |          |
 transfer_token               | character varying(64)       |           |          |
 hibp_last_check              | timestamp without time zone |           |          |
 ts_vector                    | tsvector                    |           |          | generated always as (to_tsvector('english'::regconfig, note)) stored
 transfer_token_expiration    | timestamp without time zone |           |          |
 last_email_log_id            | integer                     |           |          |
 flags                        | bigint                      |           | not null | '0'::bigint
Indexes:
    "gen_email_pkey" PRIMARY KEY, btree (id)
    "alias_transfer_token_key" UNIQUE CONSTRAINT, btree (transfer_token)
    "gen_email_email_key" UNIQUE CONSTRAINT, btree (email)
    "ix_alias_custom_domain_id" btree (custom_domain_id)
    "ix_alias_directory_id" btree (directory_id)
    "ix_alias_flags" btree (flags)
    "ix_alias_hibp_last_check" btree (hibp_last_check)
    "ix_alias_mailbox_id" btree (mailbox_id)
    "ix_alias_user_id" btree (user_id)
    "ix_video___ts_vector__" gin (ts_vector)
    "note_pg_trgm_index" gin (note gin_trgm_ops)
Foreign-key constraints:
    "alias_batch_import_id_fkey" FOREIGN KEY (batch_import_id) REFERENCES batch_import(id) ON DELETE SET NULL
    "alias_original_owner_id_fkey" FOREIGN KEY (original_owner_id) REFERENCES users(id) ON DELETE SET NULL
    "gen_email_custom_domain_id_fkey" FOREIGN KEY (custom_domain_id) REFERENCES custom_domain(id) ON DELETE CASCADE
    "gen_email_directory_id_fkey" FOREIGN KEY (directory_id) REFERENCES directory(id) ON DELETE CASCADE
    "gen_email_mailbox_id_fkey" FOREIGN KEY (mailbox_id) REFERENCES mailbox(id) ON DELETE CASCADE
    "gen_email_user_id_fkey" FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
Referenced by:
    TABLE "alias_hibp" CONSTRAINT "alias_hibp_alias_id_fkey" FOREIGN KEY (alias_id) REFERENCES alias(id) ON DELETE CASCADE
    TABLE "alias_mailbox" CONSTRAINT "alias_mailbox_alias_id_fkey" FOREIGN KEY (alias_id) REFERENCES alias(id) ON DELETE CASCADE
    TABLE "alias_used_on" CONSTRAINT "alias_used_on_alias_id_fkey" FOREIGN KEY (alias_id) REFERENCES alias(id) ON DELETE CASCADE
    TABLE "client_user" CONSTRAINT "client_user_alias_id_fkey" FOREIGN KEY (alias_id) REFERENCES alias(id) ON DELETE CASCADE
    TABLE "contact" CONSTRAINT "contact_alias_id_fkey" FOREIGN KEY (alias_id) REFERENCES alias(id) ON DELETE CASCADE
    TABLE "email_log" CONSTRAINT "email_log_alias_id_fkey" FOREIGN KEY (alias_id) REFERENCES alias(id) ON DELETE CASCADE
    TABLE "hibp_notified_alias" CONSTRAINT "hibp_notified_alias_alias_id_fkey" FOREIGN KEY (alias_id) REFERENCES alias(id) ON DELETE CASCADE
    TABLE "users" CONSTRAINT "users_newsletter_alias_id_fkey" FOREIGN KEY (newsletter_alias_id) REFERENCES alias(id) ON DELETE SET NULL
```

```
$ psql "$DB_URI" -tAc "SELECT extname FROM pg_extension ORDER BY 1;"   # after migration
pg_trgm
plpgsql
$ psql "$DB_URI" -tAc "SELECT count(*) FROM information_schema.tables WHERE table_schema='public';"
77
```

The `alias` table [app/models.py:1470] is present with the columns exercised in Q4 (`id`,
`user_id`, `email`, `name`, `enabled`, `note`, `mailbox_id`, `flags`, `pinned`,
`automatic_creation`, `created_at`), its primary key `gen_email_pkey (id)`, and the
`note_pg_trgm_index` GIN index created by the migration. The `pg_trgm` extension is present
**after** the migration (it was absent before — the migration created it).

### Seed a real user + API key through the model layer (observed)

Q4 requires a valid API key. A throwaway script uses the **real model layer** (legitimate data
setup, not a bypass of the API under test). `User.create` [app/models.py:602] sets the
password [app/models.py:606-607], auto-provisions a **verified default mailbox**
[app/models.py:611], [app/models.py:613], and creates a first "newsletter" alias
[app/models.py:634-640] — so the seeded user starts with exactly **one** alias (`id=1`). The
API key `code` is a 60-character random string [app/models.py:2350], [app/models.py:2366]
(`code = random_string(60)`). The key is written to a `mode-600` file and **never printed**;
only non-secret assertions are emitted.

```python
# /tmp/03_seed.py  (removed at teardown)   — CONFIG + PYTHONPATH=/app set in the environment
import os
from app.db import Session
from app.models import User, ApiKey, Alias

user = User.create(email="onboard@sl.local", password=os.environ["SEED_PASSWORD"], activated=True)
Session.commit()
api_key = ApiKey.create(user_id=user.id, name="onboarding")
Session.commit()

# write the key to a mode-600 file; never print its value
fd = os.open("/tmp/obs/.api_key", os.O_WRONLY | os.O_CREAT | os.O_TRUNC, 0o600)
with os.fdopen(fd, "w") as fh:
    fh.write(api_key.code)

aliases = Alias.filter_by(user_id=user.id).order_by(Alias.id).all()
print("SEED_USER_ID:", user.id)
print("SEED_DEFAULT_MAILBOX_ID:", user.default_mailbox_id)
print("SEED_ALIAS_COUNT:", len(aliases))
print("SEED_API_KEY_LEN:", len(api_key.code))
# ... (first-alias row printed too; see output)
```

Observed output (the random throwaway `SEED_PASSWORD` and the API-key value are **not**
printed; only assertions are):

```
SEED_USER_ID: 1
SEED_USER_EMAIL: onboard@sl.local
SEED_DEFAULT_MAILBOX_ID: 1
SEED_ALIAS_COUNT: 1
SEED_API_KEY_LEN: 60
SEED_API_KEY_NAME: onboarding
SEED_FIRST_ALIAS_ROW: id=1 email=simplelogin-newsletter.word441@sl.local note="This is your first alias. It's used t..."
[03] api key file perms: 600 /tmp/obs/.api_key
[03] api key file length (chars): 60
```

So: `SEED_USER_ID=1`, `SEED_DEFAULT_MAILBOX_ID=1`, one pre-existing alias
(`id=1`, `simplelogin-newsletter.word441@sl.local`, the first-alias note from
[app/models.py:638-640]), and a **60-character** API key stored in a `mode-600` file. The
`$API_KEY` shell variable used by `curl` in Q4 is loaded from that file
(`API_KEY="$(cat /tmp/obs/.api_key)"`) and is never echoed.

### Canonical run command (used for Q1–Q3 and the Q4/Q5 boots)

```
$ cd /app && CONFIG=/tmp/obs/sl_obs.env /app/venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
```

This is exactly the Docker image's production command
`CMD ["gunicorn","wsgi:app","-b","0.0.0.0:7777","-w","2","--timeout","15"]` [Dockerfile:47].

---

## Q1 — What TCP port does the application bind to on startup?

**Direct answer (observed):** the application binds to **TCP port 7777** on both paths, but on
different interfaces. The canonical Gunicorn server (the production path) binds
**`0.0.0.0:7777`** (all interfaces, via `-b 0.0.0.0:7777` [Dockerfile:47]); the development
server binds **`127.0.0.1:7777`** (localhost only) because `app.run(debug=True, port=7777)`
[server.py:588] omits the `host` argument, leaving Werkzeug's default `host="127.0.0.1"`.

**(a) Command used**

```
$ cd /app && CONFIG=/tmp/obs/sl_obs.env /app/venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
```

**(b) Verbatim observed output** — Gunicorn arbiter (stderr), boot #1, master PID `268`:

```
[2026-07-13 17:21:42 +0000] [268] [INFO] Starting gunicorn 20.0.4
[2026-07-13 17:21:42 +0000] [268] [INFO] Listening at: http://0.0.0.0:7777 (268)
[2026-07-13 17:21:42 +0000] [268] [INFO] Using worker: sync
[2026-07-13 17:21:42 +0000] [272] [INFO] Booting worker with pid: 272
[2026-07-13 17:21:42 +0000] [273] [INFO] Booting worker with pid: 273
[2026-07-13 17:21:45 +0000] [268] [INFO] Handling signal: term
[2026-07-13 17:21:45 +0000] [272] [INFO] Worker exiting (pid: 272)
[2026-07-13 17:21:45 +0000] [273] [INFO] Worker exiting (pid: 273)
[2026-07-13 17:21:46 +0000] [268] [INFO] Shutting down: Master
```

The line `Listening at: http://0.0.0.0:7777 (268)` is the master opening the listening socket.

**Development server path (observed).** `python server.py` runs `app.run(debug=True, port=7777)`
[server.py:588] and answers on the same port. Its complete transcript (stdout+stderr combined)
follows:

```
$ CONFIG=/tmp/obs/sl_obs.env PYTHONPATH=/app /app/venv/bin/python server.py
load config file /tmp/obs/sl_obs.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/xsibhzduospvchurgigl
Upload files to local dir
>>> init logging <<<
2026-07-13 17:22:38,674 - SL - DEBUG - 354 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
load config file /tmp/obs/sl_obs.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/jredsnhovjitpvphnyfs
Upload files to local dir
>>> init logging <<<
2026-07-13 17:22:40,410 - SL - DEBUG - 370 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
/app/venv/lib/python3.10/site-packages/flask_debugtoolbar/__init__.py:213: UserWarning: Could not insert debug toolbar. </body> tag not found in response.
  warnings.warn('Could not insert debug toolbar.'
```

```
$ curl -s -i http://127.0.0.1:7777/health | head -1
HTTP/1.0 200 OK
```

The dev server answers on `127.0.0.1:7777` (localhost only; note `HTTP/1.0`, the Werkzeug dev
server, versus Gunicorn's `HTTP/1.1`). Two things are worth noting in that transcript, both **observed**: (1) the app's
import-time block appears **twice** — once for the reloader parent (PID `354`) and once for the
reloader child (PID `370`) — because `debug=True` enables the auto-reloader; and (2) the
Flask/Werkzeug banner shows `* Serving Flask app "server"`, `* Environment: production`, the
`WARNING: This is a development server. Do not use it in a production deployment.` line, and
`* Debug mode: on`, but **no** `* Running on http://127.0.0.1:7777/` or `* Restarting with
stat` line appears. That omission is explained in Q2: those specific lines are emitted through
the `werkzeug` logger, which the app disables [app/log.py:70-71].

**(c) Concrete observed value:** `0.0.0.0:7777` for the canonical Gunicorn (production) server;
`127.0.0.1:7777` for the development server. The arbiter line `Listening at:
http://0.0.0.0:7777 (268)` is emitted by the master process (PID `268` on boot #1). Confirmed
**stable across two starts**: boot #2 (shown in full in Q2) produced `Listening at:
http://0.0.0.0:7777 (291)` — identical bind address; only the PID changed (`268` → `291`).

**(d) Responsible `file:line`:**
- `Dockerfile:47` → `CMD ["gunicorn","wsgi:app","-b","0.0.0.0:7777","-w","2","--timeout","15"]` (the `-b 0.0.0.0:7777` flag).
- `Dockerfile:44` → `EXPOSE 7777` (the image documents the same port).
- `server.py:588` → `app.run(debug=True, port=7777)` (the dev-server bind).
- `example.env:6` → `URL=http://localhost:7777` (the configured public base URL — not the bind).

**(e) Cause→effect:** the port is supplied by the `-b 0.0.0.0:7777` argument on the Gunicorn
command line [Dockerfile:47]; Gunicorn's master opens the listening socket and logs `Listening
at: http://0.0.0.0:7777`. It is **not** read from `URL` in the config — `URL` is the app's
public base URL, a separate config key [example.env:6], [app/config.py:79] (**inferred**: the
bind comes from the `-b` flag on the command line while `URL` is only read into `config.URL`;
the two are independent).

---

## Q2 — What do the startup / initialization logs look like?

**Direct answer (observed):** startup output has **two distinct streams** — (1) **Gunicorn
arbiter `[INFO]` lines on stderr**, and (2) the **application's import-time stdout**, printed
**once per worker** (twice with `-w 2`), including the literal banner `>>> init logging <<<`
[app/log.py:67]. No Werkzeug or Gunicorn per-request **access** log lines appear (see the
logging breakdown below), but the application's *own* request logger does emit a line for
non-`/health` API calls (demonstrated in Q4).

**(a) Command used** — the same canonical run as Q1, with stdout and stderr captured to
**separate** files so the two streams can be shown independently:

```
$ CONFIG=/tmp/obs/sl_obs.env /app/venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15 \
    > /tmp/obs/q2_run1.stdout 2> /tmp/obs/q2_run1.stderr &
$ master=$!                      # exact PID captured
# poll /health for readiness, then: kill -TERM "$master"; wait "$master"
```

**(b) Verbatim observed output** — boot #1.

_stderr (Gunicorn arbiter):_

```
[2026-07-13 17:21:42 +0000] [268] [INFO] Starting gunicorn 20.0.4
[2026-07-13 17:21:42 +0000] [268] [INFO] Listening at: http://0.0.0.0:7777 (268)
[2026-07-13 17:21:42 +0000] [268] [INFO] Using worker: sync
[2026-07-13 17:21:42 +0000] [272] [INFO] Booting worker with pid: 272
[2026-07-13 17:21:42 +0000] [273] [INFO] Booting worker with pid: 273
[2026-07-13 17:21:45 +0000] [268] [INFO] Handling signal: term
[2026-07-13 17:21:45 +0000] [272] [INFO] Worker exiting (pid: 272)
[2026-07-13 17:21:45 +0000] [273] [INFO] Worker exiting (pid: 273)
[2026-07-13 17:21:46 +0000] [268] [INFO] Shutting down: Master
```

_stdout (application import-time output — the block appears once per worker):_

```
load config file /tmp/obs/sl_obs.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/uytrlchqsjrgkzbgjehe
Upload files to local dir
>>> init logging <<<
2026-07-13 17:21:43,761 - SL - DEBUG - 273 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
load config file /tmp/obs/sl_obs.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/osaqcvjymcdpcdsegiou
Upload files to local dir
>>> init logging <<<
2026-07-13 17:21:43,800 - SL - DEBUG - 272 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

**(c) Concrete observed values:**
- **Arbiter INFO lines (stderr):** `Starting gunicorn 20.0.4`, `Listening at:
  http://0.0.0.0:7777 (<pid>)`, `Using worker: sync`, and one `Booting worker with pid:
  <pid>` per worker (two, because `-w 2`); on shutdown, `Handling signal: term`, `Worker
  exiting (pid: <pid>)` per worker, and `Shutting down: Master`.
- **App import-time stdout (once per worker):** `load config file /tmp/obs/sl_obs.env`
  [app/config.py:68], `>>> URL: http://localhost:7777` [app/config.py:80],
  `MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value` [app/config.py:123],
  `Paddle param not set` [app/config.py:217], `WARNING: Use a temp directory for GNUPGHOME
  /tmp/<random>` [app/config.py:262], `Upload files to local dir` [app/config.py:328], the
  banner `>>> init logging <<<` [app/log.py:67], and a `SL - DEBUG - <pid> - ... load words
  file` line. The block appears **twice** — once for worker `272` and once for worker `273`.

**Request-logging breakdown (corrects a common misconception).** Three independent mechanisms
could produce per-request log lines; here is what each actually does, observed:

1. **Werkzeug's per-request access logger — disabled.** `app/log.py:70-71` sets
   `logging.getLogger("werkzeug").disabled = True`. Werkzeug emits its access lines *and* its
   dev-server startup lines (`* Running on ...`, `* Restarting with stat`, `* Debugger is
   active!`) through that same logger via its internal `_log()` helper
   (`werkzeug/serving.py:977,984`, `werkzeug/_reloader.py:165`,
   `werkzeug/debug/__init__.py:275,279`, all routed to `logging.getLogger("werkzeug")` in
   `werkzeug/_internal.py:105`). Disabling the logger therefore suppresses **both** — which is
   exactly why the Q1 dev transcript shows the Flask banner but **not** `* Running on` /
   `* Restarting with stat`.
2. **Gunicorn access logging — off by default.** The canonical command [Dockerfile:47] does
   not pass `--access-logfile`, so Gunicorn writes no access-log lines (only the arbiter
   `[INFO]` lines above).
3. **The application's own request logger — active for non-`/health` requests.** An
   `@app.after_request` hook [server.py:272-296] calls `LOG.d(...)` for every request whose
   path is not `/static`, `/admin/static`, `/_debug_toolbar`, `/git`, `/favicon.ico`, or
   **`/health`** [server.py:281]. So `GET /health` produces **no** application log line, but a
   `POST /api/alias/random/new` does — an actual captured example is shown in Q4
   (`... - SL - DEBUG - <pid> - "/app/server.py:284" - after_request() - 127.0.0.1 POST
   /api/alias/random/new ... 201, takes ...`).

**(d) Responsible `file:line`:**
- `app/log.py:67` → `print(">>> init logging <<<")` (the import-time banner).
- `app/log.py:70-71` → `log = logging.getLogger("werkzeug")` / `log.disabled = True`.
- `server.py:272-296` → `@app.after_request` custom request logger; `/health` excluded at `server.py:281`.
- The arbiter lines originate from Gunicorn 20.0.4 itself.

**(e) Cause→effect:** the app-side lines are emitted at **module import time**, which happens
**once per worker** because Gunicorn's default worker model loads the WSGI app **inside each
forked worker** (not in the master). This is the default `preload_app = False`
[gunicorn 20.0.4: `gunicorn/config.py:948-954`; verified `Config().preload_app == False`], and
the app is imported by `Worker.init_process()` → `load_wsgi()`
[gunicorn 20.0.4: `gunicorn/workers/base.py:119,142`], which runs after the arbiter forks the
worker [gunicorn 20.0.4: `gunicorn/arbiter.py:561`]. That the per-worker block corresponds
one-to-one with each worker is **observed** (one block per PID); the underlying preload-default
mechanism is grounded in the Gunicorn source cited above and is confirmed by the Q5 traceback,
which shows the import happening inside `init_process`.

**Stability across two runs (observed).** Boot #2 produced the same set of lines. Its complete
capture:

_stderr:_

```
[2026-07-13 17:21:46 +0000] [291] [INFO] Starting gunicorn 20.0.4
[2026-07-13 17:21:46 +0000] [291] [INFO] Listening at: http://0.0.0.0:7777 (291)
[2026-07-13 17:21:46 +0000] [291] [INFO] Using worker: sync
[2026-07-13 17:21:46 +0000] [295] [INFO] Booting worker with pid: 295
[2026-07-13 17:21:46 +0000] [296] [INFO] Booting worker with pid: 296
[2026-07-13 17:21:48 +0000] [291] [INFO] Handling signal: term
[2026-07-13 17:21:48 +0000] [295] [INFO] Worker exiting (pid: 295)
[2026-07-13 17:21:48 +0000] [296] [INFO] Worker exiting (pid: 296)
[2026-07-13 17:21:48 +0000] [291] [INFO] Shutting down: Master
```

_stdout:_

```
load config file /tmp/obs/sl_obs.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/johpxntljfhzwyxehrmz
Upload files to local dir
>>> init logging <<<
2026-07-13 17:21:47,228 - SL - DEBUG - 296 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
load config file /tmp/obs/sl_obs.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/hiwbzvgbpjrhhaarbubq
Upload files to local dir
>>> init logging <<<
2026-07-13 17:21:47,250 - SL - DEBUG - 295 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

Comparing boot #1 and boot #2: the **stable** parts are the exact set and order of arbiter
lines and the per-worker app-import block. The **variable** parts are the **PIDs** (master
`268`→`291`; workers `272`/`273`→`295`/`296`) and the **random GNUPGHOME path** printed by
`app/config.py:262` (`/tmp/uytrlchqsjrgkzbgjehe`, `/tmp/osaqcvjymcdpcdsegiou` on boot #1 vs
`/tmp/johpxntljfhzwyxehrmz`, `/tmp/hiwbzvgbpjrhhaarbubq` on boot #2) — expected per-process
variation, not a behavior change.

---

## Q3 — What does the health-check endpoint return?

**Direct answer (observed):** `GET /health` returns the body **`success`** (exactly **7 bytes**,
no trailing newline) with HTTP status **`200 OK`** and `Content-Type: text/html; charset=utf-8`.

**(a) Command used** (against the canonical Gunicorn server on port 7777):

```
$ curl -s -i http://127.0.0.1:7777/health
```

**(b) Verbatim observed output** — response #1 (complete: status line, all headers, and body):

```
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Mon, 13 Jul 2026 17:25:48 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 7
Vary: Cookie
Set-Cookie: slapp=eyJfcGVybWFuZW50Ijp0cnVlfQ.alUfnA.c9u11PqqbZ_9117VLVf3M6TRT9M; Expires=Mon, 20-Jul-2026 17:25:48 GMT; HttpOnly; Path=/; SameSite=Lax

success
```

Response #2 (a second, independent request):

```
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Mon, 13 Jul 2026 17:25:51 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 7
Vary: Cookie
Set-Cookie: slapp=eyJfcGVybWFuZW50Ijp0cnVlfQ.alUfnw.5DSg7Go5o3oMAKFuB9fHJOHNksI; Expires=Mon, 20-Jul-2026 17:25:51 GMT; HttpOnly; Path=/; SameSite=Lax

success
```

**Body byte count (observed):**

```
$ curl -s http://127.0.0.1:7777/health | wc -c
7
```

**(c) Concrete observed values and stable-vs-variable comparison.** The two responses are
**NOT byte-identical** — three fields legitimately vary between requests. Comparing response #1
and response #2:

| Field | Response #1 | Response #2 | Stable? |
|-------|-------------|-------------|---------|
| Status line | `HTTP/1.1 200 OK` | `HTTP/1.1 200 OK` | **stable** |
| `Server` | `gunicorn/20.0.4` | `gunicorn/20.0.4` | **stable** |
| `Content-Type` | `text/html; charset=utf-8` | `text/html; charset=utf-8` | **stable** |
| `Content-Length` | `7` | `7` | **stable** |
| `Vary` | `Cookie` | `Cookie` | **stable** |
| Body | `success` | `success` | **stable** |
| `Date` | `Mon, 13 Jul 2026 17:25:48 GMT` | `Mon, 13 Jul 2026 17:25:51 GMT` | *varies* (clock) |
| `Set-Cookie` (slapp value) | `...alUfnA.c9u11PqqbZ_9117VLVf3M6TRT9M` | `...alUfnw.5DSg7Go5o3oMAKFuB9fHJOHNksI` | *varies* (re-signed each response) |
| `Expires` (of the cookie) | `Mon, 20-Jul-2026 17:25:48 GMT` | `Mon, 20-Jul-2026 17:25:51 GMT` | *varies* (Date + 7 days) |

So the **answer to the question** (status + body + content type + length) is fully stable across
two runs; only the per-response `Date`, the session-cookie value, and the cookie `Expires`
change — expected variation, not a behavior difference.

**About the `slapp` cookie (observed, and why it is safe to show).** The `Set-Cookie: slapp=...`
header is a standard signed Flask session cookie. Its first dot-separated segment is the
base64url-encoded payload; decoding it shows the session carries only an anonymous "permanent"
flag — no user identity and no secret:

```
$ echo -n 'eyJfcGVybWFuZW50Ijp0cnVlfQ' | tr '_-' '/+' | base64 -d
{"_permanent":true}
```

Both responses carry the **same** payload segment `eyJfcGVybWFuZW50Ijp0cnVlfQ`
(→ `{"_permanent":true}`); they differ only in the timestamp segment (`alUfnA` vs `alUfnw`) and
the HMAC signature (`c9u11PqqbZ_9117VLVf3M6TRT9M` vs `5DSg7Go5o3oMAKFuB9fHJOHNksI`), because
Flask re-signs the cookie with a fresh timestamp on every response. The signature is an HMAC and
does **not** reveal `FLASK_SECRET`; the payload contains no credentials. The cookie is present on
`/health` because a `@app.before_request` hook marks every session permanent (below) — `/health`
itself sets no session data.

**(d) Responsible `file:line`:**
- `server.py:213-215` → `@app.route("/health", methods=["GET"])` / `def healthcheck():` / `return "success", 200` (the literal body + status).
- `server.py:204-207` → `@app.before_request def make_session_permanent()` sets `session.permanent = True` and `permanent_session_lifetime = timedelta(days=7)` — the source of the 7-day cookie `Expires`; the accompanying comment "the cookie is valid for 7 days" is at `server.py:202-203`.
- `app/config.py:199` → `SESSION_COOKIE_NAME = "slapp"` (the cookie's name), applied via `app.config["SESSION_COOKIE_NAME"] = SESSION_COOKIE_NAME` at `server.py:159`.

**(e) Cause→effect:** the handler returns the Python tuple `("success", 200)` [server.py:215];
Flask turns the string into a `text/html; charset=utf-8` body of length 7 and the integer into
the HTTP status. The `Vary: Cookie` header and the `slapp` `Set-Cookie` are added by Flask's
session machinery because `make_session_permanent` [server.py:204-207] runs on every request; the
7-day `Expires` is a direct consequence of `permanent_session_lifetime = timedelta(days=7)`
(**inferred**: the header-to-code mapping is deduced from Flask's documented session behavior,
while the body/status/length and the cookie payload are directly observed above).

---

## Q4 — Alias creation: API response + database persistence

**Direct answer (observed).** A `POST /api/alias/random/new` authenticated with the
`Authentication` header returns **HTTP 201 CREATED** with a JSON object of **17 keys** describing
the newly created alias (for example `"id": 2`, `"email": "test_test346@sl.local"`,
`"note": "onboarding demo alias"`). Exactly **one** row is inserted into the **`alias`** table per
successful call; the JSON `id` equals the database primary-key `id`, `user_id` is the calling
user, `mailbox_id` is that user's default mailbox, and `note` is the supplied note. A call that
fails authentication returns **HTTP 401** with body `{"error":"Wrong api key"}` and persists
**nothing**. All values below are observed from a live run; the code mappings are labelled where
inferred.

### Q4.0 — Canonical entry point, authentication, and the exact command

Requests enter through the Flask blueprint mounted at `/api` [app/api/base.py:11] and the route
`POST /api/alias/random/new` [app/api/views/new_random_alias.py:21], which is guarded by the
`@require_api_auth` decorator [app/api/views/new_random_alias.py:23], [app/api/base.py:52].

Authentication is resolved in `authorize_request()` [app/api/base.py:16]: the API key is read from
the **`Authentication`** request header [app/api/base.py:17] and looked up with
`ApiKey.get_by(code=api_code)` [app/api/base.py:18].

**The `Authentication` header is not universally required.** When no valid key is found
(`if not api_key:` [app/api/base.py:20]) the code first checks `if current_user.is_authenticated:`
[app/api/base.py:21] and, if a logged-in web session exists, falls back to that session user
(`g.user = current_user` [app/api/base.py:25]); only a caller that is **both** without a valid key
**and** anonymous (no authenticated session) is rejected with
`return jsonify(error="Wrong api key"), 401` [app/api/base.py:27]. Every `curl` command in this
section is anonymous (it sends no session cookie), so the 401 branch is what applies to the
negative cases below.

API keys are 60-character random strings produced by `code = random_string(60)`
[app/models.py:2366] on the `ApiKey` model [app/models.py:2350]. The key used below was created
through the model layer during seeding, stored in a mode-600 file, and read into the `$API_KEY`
shell variable — it is never printed.

Canonical command (the key is passed via `$API_KEY` and never echoed):

```bash
curl -s -i -X POST http://127.0.0.1:7777/api/alias/random/new \
     -H "Authentication: ${API_KEY}" \
     -H "Content-Type: application/json" \
     -d '{"note":"onboarding demo alias"}'
```

### Q4.1 — The API response (observed, both creations)

First successful creation — **HTTP 201 CREATED**, captured with `curl -s -i` (full status line,
headers, and JSON body):

```
HTTP/1.1 201 CREATED
Server: gunicorn/20.0.4
Date: Mon, 13 Jul 2026 17:49:51 GMT
Connection: close
Content-Type: application/json
Content-Length: 422
Access-Control-Allow-Origin: *
Vary: Cookie
Set-Cookie: slapp=eyJfZnJlc2giOmZhbHNlLCJfcGVybWFuZW50Ijp0cnVlfQ.alUlPw.7EXsDfESQFmZWh2twaKSLIVS-mM; Expires=Mon, 20-Jul-2026 17:49:51 GMT; HttpOnly; Path=/; SameSite=Lax

{"alias":"test_test346@sl.local","creation_date":"2026-07-13 17:49:51+00:00","creation_timestamp":1783964991,"disable_pgp":false,"email":"test_test346@sl.local","enabled":true,"id":2,"latest_activity":null,"mailbox":{"email":"onboard@sl.local","id":1},"mailboxes":[{"email":"onboard@sl.local","id":1}],"name":null,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":"onboarding demo alias","pinned":false,"support_pgp":false}
```

Second successful creation — a distinct alias with the identical response shape:

```
HTTP/1.1 201 CREATED
Server: gunicorn/20.0.4
Date: Mon, 13 Jul 2026 17:49:52 GMT
Connection: close
Content-Type: application/json
Content-Length: 424
Access-Control-Allow-Origin: *
Vary: Cookie
Set-Cookie: slapp=eyJfZnJlc2giOmZhbHNlLCJfcGVybWFuZW50Ijp0cnVlfQ.alUlQA.QrpbW7zvdT82JFBJtNjrAchilaE; Expires=Mon, 20-Jul-2026 17:49:52 GMT; HttpOnly; Path=/; SameSite=Lax

{"alias":"word_list831@sl.local","creation_date":"2026-07-13 17:49:52+00:00","creation_timestamp":1783964992,"disable_pgp":false,"email":"word_list831@sl.local","enabled":true,"id":3,"latest_activity":null,"mailbox":{"email":"onboard@sl.local","id":1},"mailboxes":[{"email":"onboard@sl.local","id":1}],"name":null,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":"second onboarding alias","pinned":false,"support_pgp":false}
```

The body is assembled by the endpoint's return statement [app/api/views/new_random_alias.py:114-116]:

```python
return (
    jsonify(alias=alias.email, **serialize_alias_info_v2(get_alias_info_v2(alias))),
    201,
)
```

`serialize_alias_info_v2` [app/api/serializer.py:55] — fed by `get_alias_info_v2`
[app/api/serializer.py:252], which populates an `AliasInfo` dataclass [app/api/serializer.py:22] —
contributes **16** keys: `id, email, creation_date, creation_timestamp, enabled, note, name,
nb_forward, nb_block, nb_reply, mailbox, mailboxes, support_pgp, disable_pgp, latest_activity,
pinned`. The endpoint then adds a **17th** top-level key, `alias` (set to `alias.email`), for a
total of **17 keys**. Flask's `jsonify` serializes the dict with `sort_keys` on, which alphabetizes
the output — that is why `alias` appears first and `support_pgp` last. In both responses the
`alias` and `email` keys carry the same value.

The two responses are **not byte-identical**: they describe different aliases (`"id"` 2 vs 3, a
different random `email`, different `creation_date`/`creation_timestamp`, and a different `note`),
and each carries a freshly re-signed `slapp` session cookie. What they share is the **same parsed
JSON content and structure** — the same 17 keys in the same (alphabetized) order. Both payloads are
shown above **verbatim**, exactly as Flask's `jsonify` emits them (compact single-line, not
pretty-printed). (This corrects an earlier characterization
of the two payloads as "the same bytes"; only the shape is identical, not the bytes.)

### Q4.2 — What is persisted (the `alias` table) and the persistence semantics

The `Alias` model [app/models.py:1469] maps to the **`alias`** table [app/models.py:1470]. Row
state was read straight from PostgreSQL with a `left(note, 40)` projection so the `note` column is
shown verbatim under a normal column header, with no client-side truncation and no ellipses:

```sql
select id, user_id, email, name, enabled, left(note, 40) as note,
       mailbox_id, flags, pinned, automatic_creation, created_at
  from alias order by id;
```

After the first successful POST (two rows — the seed alias plus the newly created one):

```
 id | user_id |                  email                  | name | enabled |                   note                   | mailbox_id | flags | pinned | automatic_creation |         created_at
----+---------+-----------------------------------------+------+---------+------------------------------------------+------------+-------+--------+--------------------+----------------------------
  1 |       1 | simplelogin-newsletter.word441@sl.local |      | t       | This is your first alias. It's used to r |          1 |     0 | f      | f                  | 2026-07-13 17:21:00.839761
  2 |       1 | test_test346@sl.local                   |      | t       | onboarding demo alias                    |          1 |     0 | f      | f                  | 2026-07-13 17:49:51.886376
(2 rows)

```

After the second successful POST (three rows):

```
 id | user_id |                  email                  | name | enabled |                   note                   | mailbox_id | flags | pinned | automatic_creation |         created_at
----+---------+-----------------------------------------+------+---------+------------------------------------------+------------+-------+--------+--------------------+----------------------------
  1 |       1 | simplelogin-newsletter.word441@sl.local |      | t       | This is your first alias. It's used to r |          1 |     0 | f      | f                  | 2026-07-13 17:21:00.839761
  2 |       1 | test_test346@sl.local                   |      | t       | onboarding demo alias                    |          1 |     0 | f      | f                  | 2026-07-13 17:49:51.886376
  3 |       1 | word_list831@sl.local                   |      | t       | second onboarding alias                  |          1 |     0 | f      | f                  | 2026-07-13 17:49:52.036742
(3 rows)

```

**Correlation (observed).** The JSON `id` equals the database primary key. POST #1 returned
`"id": 2` and produced row `id=2` (`test_test346@sl.local`, note `onboarding demo alias`); POST #2
returned `"id": 3` and produced row `id=3` (`word_list831@sl.local`, note `second onboarding
alias`). Every new row carries `user_id=1` (the calling user), `mailbox_id=1` (the user's default
mailbox), `name` NULL, `enabled=t`, `flags=0`, `pinned=f`, and `automatic_creation=f`, with a
`created_at` that matches the response's `creation_date` to the second
(`2026-07-13 17:49:51` / `.886376` for row 2; `2026-07-13 17:49:52` / `.036742` for row 3).

**Persistence semantics (corrected).** The endpoint calls `Alias.create_new_random(...)`
[app/api/views/new_random_alias.py:106], which builds the row via
`Alias.create(user_id=…, email=…, mailbox_id=…, note=…)` [app/models.py:1750-1755] — passing
**neither** `commit` **nor** `flush`. That call resolves to the **`Alias.create` override**
[app/models.py:1628-1692] — which shadows the base `ModelMixin.create` [app/models.py:116] for
the alias path — where **both** arguments default to `False`: `commit = kw.pop("commit", False)`
[app/models.py:1629] and `flush = kw.pop("flush", False)` [app/models.py:1630]. The override adds
the row with `Session.add(new_alias)` [app/models.py:1660], then commits or flushes only if those
flags are set; since both are `False` it does neither and returns the instance
[app/models.py:1692] in the SQLAlchemy **pending** state — no `INSERT` has been emitted yet. The row is not sent to PostgreSQL until the
endpoint issues an explicit `Session.commit()` [app/api/views/new_random_alias.py:107], which
flushes the pending `INSERT` and commits the transaction. (This corrects the earlier claim that
`create(commit=False)` had "already flushed" the alias: it had not, because `flush` is `False` by
default as well — the object stays pending until the endpoint's own commit.)

### Q4.3 — Negative (authentication) cases and stability

To make the effect unambiguous, the `alias` table was reset to a single seed row (`id=1`) before
this run and `select count(*) from alias` was sampled at every step, giving one consistent
timeline:

```
### t0 alias count (before any create) ###
1
### t1 after no-header call (rejected) ###
1
### t2 after wrong-key call (rejected) ###
1
### t4 after successful POST #1 ###
2
### t6 after successful POST #2 ###
3
```

That is: **1** row before any call; still **1** after the anonymous no-header call (t1); still **1**
after the wrong-key call (t2); **2** after the first successful POST (t4); **3** after the second
(t6). Each successful creation adds exactly one row; each rejected call adds none.

Anonymous call with **no** `Authentication` header — full response including headers — **HTTP 401**,
nothing persisted:

```
HTTP/1.1 401 UNAUTHORIZED
Server: gunicorn/20.0.4
Date: Mon, 13 Jul 2026 17:49:51 GMT
Connection: close
Content-Type: application/json
Content-Length: 26
Access-Control-Allow-Origin: *
Vary: Cookie
Set-Cookie: slapp=eyJfZnJlc2giOmZhbHNlLCJfcGVybWFuZW50Ijp0cnVlfQ.alUlPw.7EXsDfESQFmZWh2twaKSLIVS-mM; Expires=Mon, 20-Jul-2026 17:49:51 GMT; HttpOnly; Path=/; SameSite=Lax

{"error":"Wrong api key"}
```

Call with a **wrong** API key (`Authentication: this-is-not-a-valid-key`) — a byte-identical 401:

```
HTTP/1.1 401 UNAUTHORIZED
Server: gunicorn/20.0.4
Date: Mon, 13 Jul 2026 17:49:51 GMT
Connection: close
Content-Type: application/json
Content-Length: 26
Access-Control-Allow-Origin: *
Vary: Cookie
Set-Cookie: slapp=eyJfZnJlc2giOmZhbHNlLCJfcGVybWFuZW50Ijp0cnVlfQ.alUlPw.7EXsDfESQFmZWh2twaKSLIVS-mM; Expires=Mon, 20-Jul-2026 17:49:51 GMT; HttpOnly; Path=/; SameSite=Lax

{"error":"Wrong api key"}
```

Both negative responses carry `Content-Type: application/json`, `Content-Length: 26`, and body
`{"error":"Wrong api key"}` — the anonymous rejection branch at [app/api/base.py:27]. The
missing-header case reaches the same outcome because `request.headers.get("Authentication")`
[app/api/base.py:17] returns `None` and `ApiKey.get_by(code=None)` [app/api/base.py:18] finds no
key, so `if not api_key:` [app/api/base.py:20] is true and (the caller being anonymous) the 401 is
returned.

**Observability of these calls** (reinforces Q2 and finding on request logging): the application's
custom `after_request` logger [server.py:272-296] emitted exactly one DEBUG line per API request —
the two 401s and the two 201s — while the readiness `GET /health` probe produced **no** log line at
all, because `/health` is explicitly excluded from that logger [server.py:281]:

```
### app custom after_request LOG.d lines (server.py:284) — from stdout+stderr ###
2026-07-13 17:49:51,731 - SL - DEBUG - 964 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/alias/random/new ImmutableMultiDict([]) 401, takes 0.0033311843872070312
2026-07-13 17:49:51,783 - SL - DEBUG - 965 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/alias/random/new ImmutableMultiDict([]) 401, takes 0.0036230087280273438
2026-07-13 17:49:51,902 - SL - DEBUG - 965 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/alias/random/new ImmutableMultiDict([]) 201, takes 0.06981420516967773
2026-07-13 17:49:52,044 - SL - DEBUG - 965 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/alias/random/new ImmutableMultiDict([]) 201, takes 0.036386728286743164
### any /health request-log line? (expect NONE — excluded at server.py:281) ###
(none — /health is not logged)
```

Each line is emitted by the `LOG.d(...)` call at [server.py:284] and records the method, path,
form data, resulting status, and elapsed time — confirming (a) that disabling the Werkzeug logger
does **not** remove per-request logging (the app keeps its own), and (b) that `/health` is
deliberately silent.

### Q4.4 — Reproduction caveat: the `pg_trgm` migration (setup context, not the runtime answer)

**Labelled inferred / setup context.** Building the schema for Q4 requires a *fresh* database in
which the `pg_trgm` extension is created **by the migration itself** rather than pre-created.
Migration `2021_082012_424808e1fe49_` runs `op.execute('CREATE EXTENSION pg_trgm')`
[migrations/versions/2021_082012_424808e1fe49_.py:24]; if the extension already exists it catches
the duplicate-object error and runs `op.execute("Rollback")`
[migrations/versions/2021_082012_424808e1fe49_.py:29]. Because Alembic executes all migrations
inside a single PostgreSQL transaction, that `Rollback` unwinds everything done earlier in the same
transaction — including creation of the `alias` table — after which
`op.create_index('note_pg_trgm_index', 'alias', ['note'], …)`
[migrations/versions/2021_082012_424808e1fe49_.py:31] fails because the table no longer exists.
This is an **inferred** explanation of a reproduction pitfall (it is why the setup uses a brand-new
database and never pre-creates `pg_trgm`); it is **not** part of the observed Q4 runtime behaviour,
and per the read-only rule set and AAP §0.3.2 the underlying migration ordering is documented here,
not fixed.

---

## Q5 — Startup failure when PostgreSQL is unavailable

**Direct answer (observed).** With PostgreSQL stopped, the Gunicorn **master still binds the socket
and logs `Listening at: http://0.0.0.0:7777`** — then **both** workers fail to boot while *importing*
the WSGI application. The break point is the module-level eager connection
`connection = engine.connect()` [app/db.py:12], reached through the import chain
`wsgi.py:1 → server.py:31 → app/admin_model.py:11 → app/models.py:32 → app/db.py:12`. Each worker
raises **`sqlalchemy.exc.OperationalError`** wrapping **`psycopg2.OperationalError: … Connection
refused`** (tried both `::1` and `127.0.0.1`). After the second worker fails, the master logs
**`Shutting down: Master`** / **`Reason: Worker failed to boot.`** and the whole process exits with
**status code 3**. The failure is a hard boot failure with no graceful degradation, because the
connection is opened at *import* time — before the Flask app object even exists.

### Q5.0 — How the failure was produced (exact commands)

PostgreSQL was stopped through the Debian cluster manager, and the shutdown was verified before
booting the server:

```console
$ pg_ctlcluster 15 main stop -m fast
exit=0
$ pg_lsclusters (status after stop)
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 down   postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
--- port 5432 probe ---
5432 refused (as expected)
```

The canonical server was then booted exactly as in production (this is the same command as Q1/Q2),
capturing merged stdout+stderr and the process exit code:

```bash
CONFIG=<config> /app/venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15 > q5_run.log 2>&1
echo "exit=$?"
```

Observed exit codes for the two independent boots:

```
Q5B_RUN1_EXIT=3
Q5B_RUN2_EXIT=3
```

(After capture, PostgreSQL was restarted with `pg_ctlcluster 15 main start` — `restart_exit=0`,
`DB back up OK` — to restore the observation database.)

### Q5.1 — The import chain to the break point (observed frames + citations)

The application opens its database connection at *module import* time. `app/db.py` builds the engine
from `config.DB_URI` [app/db.py:9-11] and then immediately, at module scope, executes
`connection = engine.connect()` [app/db.py:12]. Because Gunicorn imports the WSGI target
`wsgi:app` inside each worker, that eager connect runs during import and fails when PostgreSQL is
down. The app-side frames — excerpted here from the complete boot #1 shown below in Q5.2 — walk the
exact chain:

```
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
```

That is: `wsgi.py:1` (`from server import create_app`) → `server.py:31`
(`from app.admin_model import …`) → `app/admin_model.py:11` (`from app import models, s3`) →
`app/models.py:32` (`from app.db import Session`) → `app/db.py:12` (`connection = engine.connect()`),
which is the observed break point.

### Q5.2 — Complete boot #1 (verbatim, exit code 3)

The full, unedited merged stdout+stderr of the first boot follows. Note the ordering: the master
logs `Listening at: http://0.0.0.0:7777` (line 2) **before** any worker fails; each of the two
workers (`-w 2`) then raises the same exception (the app's config stdout — `load config file`,
`>>> URL:`, … `>>> init logging <<<` — is interleaved because it is printed to stdout during the
same import that fails); finally the master shuts down. Both worker tracebacks and both app-import
stdout blocks are shown in full:

```
[2026-07-13 18:00:37 +0000] [1248] [INFO] Starting gunicorn 20.0.4
[2026-07-13 18:00:37 +0000] [1248] [INFO] Listening at: http://0.0.0.0:7777 (1248)
[2026-07-13 18:00:37 +0000] [1248] [INFO] Using worker: sync
[2026-07-13 18:00:37 +0000] [1249] [INFO] Booting worker with pid: 1249
[2026-07-13 18:00:37 +0000] [1250] [INFO] Booting worker with pid: 1250
[2026-07-13 18:00:38 +0000] [1249] [ERROR] Exception in worker process
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
  File "<frozen importlib._bootstrap>", line 1050, in _gcd_import
  File "<frozen importlib._bootstrap>", line 1027, in _find_and_load
  File "<frozen importlib._bootstrap>", line 1006, in _find_and_load_unlocked
  File "<frozen importlib._bootstrap>", line 688, in _load_unlocked
  File "<frozen importlib._bootstrap_external>", line 883, in exec_module
  File "<frozen importlib._bootstrap>", line 241, in _call_with_frames_removed
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
sqlalchemy.exc.OperationalError: (psycopg2.OperationalError) connection to server at "localhost" (::1), port 5432 failed: Connection refused
	Is the server running on that host and accepting TCP/IP connections?
connection to server at "localhost" (127.0.0.1), port 5432 failed: Connection refused
	Is the server running on that host and accepting TCP/IP connections?

(Background on this error at: http://sqlalche.me/e/13/e3q8)
[2026-07-13 18:00:38 +0000] [1249] [INFO] Worker exiting (pid: 1249)
load config file /tmp/obs/sl_obs.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/vitzachewavttaigfwhf
Upload files to local dir
>>> init logging <<<
[2026-07-13 18:00:38 +0000] [1250] [ERROR] Exception in worker process
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
  File "<frozen importlib._bootstrap>", line 1050, in _gcd_import
  File "<frozen importlib._bootstrap>", line 1027, in _find_and_load
  File "<frozen importlib._bootstrap>", line 1006, in _find_and_load_unlocked
  File "<frozen importlib._bootstrap>", line 688, in _load_unlocked
  File "<frozen importlib._bootstrap_external>", line 883, in exec_module
  File "<frozen importlib._bootstrap>", line 241, in _call_with_frames_removed
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
sqlalchemy.exc.OperationalError: (psycopg2.OperationalError) connection to server at "localhost" (::1), port 5432 failed: Connection refused
	Is the server running on that host and accepting TCP/IP connections?
connection to server at "localhost" (127.0.0.1), port 5432 failed: Connection refused
	Is the server running on that host and accepting TCP/IP connections?

(Background on this error at: http://sqlalche.me/e/13/e3q8)
[2026-07-13 18:00:38 +0000] [1250] [INFO] Worker exiting (pid: 1250)
load config file /tmp/obs/sl_obs.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/alyuoaobyjsvlwzhvmsq
Upload files to local dir
>>> init logging <<<
[2026-07-13 18:00:38 +0000] [1248] [INFO] Shutting down: Master
[2026-07-13 18:00:38 +0000] [1248] [INFO] Reason: Worker failed to boot.
```

### Q5.3 — Complete boot #2 (verbatim, exit code 3) and run-to-run stability

A second independent boot produced the identical failure (also exit code 3). Its full, unedited
output:

```
[2026-07-13 18:00:38 +0000] [1252] [INFO] Starting gunicorn 20.0.4
[2026-07-13 18:00:38 +0000] [1252] [INFO] Listening at: http://0.0.0.0:7777 (1252)
[2026-07-13 18:00:38 +0000] [1252] [INFO] Using worker: sync
[2026-07-13 18:00:38 +0000] [1253] [INFO] Booting worker with pid: 1253
[2026-07-13 18:00:38 +0000] [1254] [INFO] Booting worker with pid: 1254
[2026-07-13 18:00:39 +0000] [1253] [ERROR] Exception in worker process
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
  File "<frozen importlib._bootstrap>", line 1050, in _gcd_import
  File "<frozen importlib._bootstrap>", line 1027, in _find_and_load
  File "<frozen importlib._bootstrap>", line 1006, in _find_and_load_unlocked
  File "<frozen importlib._bootstrap>", line 688, in _load_unlocked
  File "<frozen importlib._bootstrap_external>", line 883, in exec_module
  File "<frozen importlib._bootstrap>", line 241, in _call_with_frames_removed
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
sqlalchemy.exc.OperationalError: (psycopg2.OperationalError) connection to server at "localhost" (::1), port 5432 failed: Connection refused
	Is the server running on that host and accepting TCP/IP connections?
connection to server at "localhost" (127.0.0.1), port 5432 failed: Connection refused
	Is the server running on that host and accepting TCP/IP connections?

(Background on this error at: http://sqlalche.me/e/13/e3q8)
[2026-07-13 18:00:39 +0000] [1253] [INFO] Worker exiting (pid: 1253)
load config file /tmp/obs/sl_obs.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/wyvvfnbyoxkomavbctin
Upload files to local dir
>>> init logging <<<
[2026-07-13 18:00:39 +0000] [1254] [ERROR] Exception in worker process
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
  File "<frozen importlib._bootstrap>", line 1050, in _gcd_import
  File "<frozen importlib._bootstrap>", line 1027, in _find_and_load
  File "<frozen importlib._bootstrap>", line 1006, in _find_and_load_unlocked
  File "<frozen importlib._bootstrap>", line 688, in _load_unlocked
  File "<frozen importlib._bootstrap_external>", line 883, in exec_module
  File "<frozen importlib._bootstrap>", line 241, in _call_with_frames_removed
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
sqlalchemy.exc.OperationalError: (psycopg2.OperationalError) connection to server at "localhost" (::1), port 5432 failed: Connection refused
	Is the server running on that host and accepting TCP/IP connections?
connection to server at "localhost" (127.0.0.1), port 5432 failed: Connection refused
	Is the server running on that host and accepting TCP/IP connections?

(Background on this error at: http://sqlalche.me/e/13/e3q8)
[2026-07-13 18:00:39 +0000] [1254] [INFO] Worker exiting (pid: 1254)
load config file /tmp/obs/sl_obs.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/xbizjyxsryyxacnjmpbj
Upload files to local dir
>>> init logging <<<
[2026-07-13 18:00:39 +0000] [1252] [INFO] Shutting down: Master
[2026-07-13 18:00:39 +0000] [1252] [INFO] Reason: Worker failed to boot.
```

**Stability (observed).** The two boots are **deterministic**. A line-by-line `diff` of the two
281-line captures differs *only* in the wall-clock timestamps, the process IDs (master and the two
workers), and the two ephemeral `GNUPGHOME` temp-directory names — every other line, including both
complete tracebacks, the `Listening at: http://0.0.0.0:7777` line, the whole import chain, the
`sqlalchemy.exc.OperationalError`, and the `Shutting down: Master` / `Reason: Worker failed to boot.`
lines, is byte-for-byte identical:

```
1,6c1,6
< [2026-07-13 18:00:37 +0000] [1248] [INFO] Starting gunicorn 20.0.4
< [2026-07-13 18:00:37 +0000] [1248] [INFO] Listening at: http://0.0.0.0:7777 (1248)
< [2026-07-13 18:00:37 +0000] [1248] [INFO] Using worker: sync
< [2026-07-13 18:00:37 +0000] [1249] [INFO] Booting worker with pid: 1249
< [2026-07-13 18:00:37 +0000] [1250] [INFO] Booting worker with pid: 1250
< [2026-07-13 18:00:38 +0000] [1249] [ERROR] Exception in worker process
---
> [2026-07-13 18:00:38 +0000] [1252] [INFO] Starting gunicorn 20.0.4
> [2026-07-13 18:00:38 +0000] [1252] [INFO] Listening at: http://0.0.0.0:7777 (1252)
> [2026-07-13 18:00:38 +0000] [1252] [INFO] Using worker: sync
> [2026-07-13 18:00:38 +0000] [1253] [INFO] Booting worker with pid: 1253
> [2026-07-13 18:00:38 +0000] [1254] [INFO] Booting worker with pid: 1254
> [2026-07-13 18:00:39 +0000] [1253] [ERROR] Exception in worker process
135c135
< [2026-07-13 18:00:38 +0000] [1249] [INFO] Worker exiting (pid: 1249)
---
> [2026-07-13 18:00:39 +0000] [1253] [INFO] Worker exiting (pid: 1253)
140c140
< WARNING: Use a temp directory for GNUPGHOME /tmp/vitzachewavttaigfwhf
---
> WARNING: Use a temp directory for GNUPGHOME /tmp/wyvvfnbyoxkomavbctin
143c143
< [2026-07-13 18:00:38 +0000] [1250] [ERROR] Exception in worker process
---
> [2026-07-13 18:00:39 +0000] [1254] [ERROR] Exception in worker process
272c272
< [2026-07-13 18:00:38 +0000] [1250] [INFO] Worker exiting (pid: 1250)
---
> [2026-07-13 18:00:39 +0000] [1254] [INFO] Worker exiting (pid: 1254)
277c277
< WARNING: Use a temp directory for GNUPGHOME /tmp/alyuoaobyjsvlwzhvmsq
---
> WARNING: Use a temp directory for GNUPGHOME /tmp/xbizjyxsryyxacnjmpbj
280,281c280,281
< [2026-07-13 18:00:38 +0000] [1248] [INFO] Shutting down: Master
< [2026-07-13 18:00:38 +0000] [1248] [INFO] Reason: Worker failed to boot.
---
> [2026-07-13 18:00:39 +0000] [1252] [INFO] Shutting down: Master
> [2026-07-13 18:00:39 +0000] [1252] [INFO] Reason: Worker failed to boot.
```

(For completeness, an earlier pair of boots captured during setup showed the same signature and the
same exit code 3 — four boots in total, all identical apart from PIDs/timestamps/temp-dirs.)

### Q5.4 — Why the master binds the port but the workers die (Gunicorn worker-loading)

**Observed.** In every boot the master's `Listening at: http://0.0.0.0:7777 (<pid>)` line appears
*before* any worker traceback, and the `OperationalError` is raised only inside the forked workers
(`[<worker-pid>] [ERROR] Exception in worker process`). So the listening socket is bound
successfully even though no worker ever serves a request.

**Authority (Gunicorn 20.0.4).** Gunicorn's `preload_app` setting defaults to **`False`**
[gunicorn/config.py:948-954, class `PreloadApp`; runtime-verified: `Config().preload_app == False`],
and the production command (`gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` [Dockerfile:47])
does **not** pass `--preload`. With preloading off, the master process does not import the WSGI
application itself; instead each worker imports it **after** being forked. The observed traceback
frames are exactly this path: `arbiter.py:583` (`spawn_worker` → `worker.init_process()`) →
`workers/base.py:119` (`init_process` → `self.load_wsgi()`) → `workers/base.py:144`
(`load_wsgi` → `self.wsgi = self.app.wsgi()`), which triggers the import of `wsgi:app`.

**Inferred (from the authority above + the observed ordering).** Because the master binds the
socket and forks workers before any application import happens, the port ends up bound (the master
succeeds) while every worker dies during its own import at `connection = engine.connect()`
[app/db.py:12]. This is why "the port is listening" and "the app cannot start" are simultaneously
true when PostgreSQL is down. (If the image had used `--preload`, the master would import the app
first and fail before binding — a different failure mode; that is not the configured behaviour
here.) The root cause — opening a database connection at import time [app/db.py:12] — is documented
here and, per the read-only rule set and AAP §0.3.2, deliberately not fixed.

---

## Cleanup & Repository-Cleanliness

The investigation was designed to leave **no trace** in the source repository: everything ran
inside a fresh, disposable container, and every artifact lived under `/tmp` inside that container —
never in the repo.

### Observation isolation

The container was launched with an init/reaper (`--init`, so PID 1 is `docker-init`/tini). All
observation scripts, the throwaway `CONFIG` file, every captured evidence file, and the mode-600
secret files (`.db_uri`, `.api_key`) resided under `/tmp/obs` inside that container — outside the
repository working tree at all times.

### Process teardown (no zombies, no orphaned workers)

Each canonical Gunicorn boot was either torn down explicitly (SIGTERM to the master, then `wait`)
or exited on its own (the Q5 failure boots exit `3`). Because PID 1 is a real init process, any
transient child is reaped — there are no defunct/zombie processes and no orphaned workers:

```console
$ ps -o pid=,comm= -p 1            # PID 1 = init/reaper
      1 docker-init
$ ps -eo stat=,pid=,comm= | awk '$1 ~ /Z/'   # zombie (defunct) processes
(no output — zero zombie processes; docker-init/tini reaps children)
$ pgrep -a gunicorn || echo "no gunicorn processes"   # orphaned workers?
no gunicorn processes
```

(The `postgres` processes that remain belong to the managed PostgreSQL cluster, which was
restarted after the Q5 experiment — they are the database server, not orphaned application
workers.)

### Source repository unchanged

The answer document is the **only** change to the working-tree repository. Before the final commit,
`git status --porcelain` lists exactly one modified path, and `git diff --check` reports no
whitespace or end-of-file errors:

```console
$ git status --porcelain
 M blitzy/documentation/app_2cd6ee777f8c.md
$ git diff --check
(no output — no trailing-whitespace or end-of-file errors)
```

After the deliverable is committed, `git status --porcelain` is empty. The disposable observation
container is then destroyed (`docker rm -f`); any files it generated at runtime (for example the
regenerated `local_data/*` keys) vanish with it and never reach the source tree, which stays
byte-for-byte unchanged apart from this document.

### A note on captured whitespace

All embedded command output is verbatim in its **visible** content. Some captures inherently carry
trailing whitespace — the CRLF carriage returns in `curl -i` HTTP responses, and the
column-alignment padding in `psql` result tables. That invisible trailing whitespace has been
trimmed so that `git diff --check` is clean for this file; no visible character was altered and no
line of output was removed.
