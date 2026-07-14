# SimpleLogin Self-Host Runtime Verification — `app_2cd6ee777f8c`

**Runtime-verification Q&A report.** This document confirms, by **actually booting** a local
[SimpleLogin](https://github.com/simple-login/app) instance in its **default configuration** and
**observing live runtime behavior**, that a first-time self-hosted deployment is healthy across
**(a) user authentication** and **(b) alias-based email activity**. Every behavioral claim below is
paired with the exact command that produced it, the actual (unedited) captured output, and a
`file:line` citation naming the specific function/method. Statements not confirmed at runtime are
explicitly labeled **inferred**; values obtained by bypassing a real interface are explicitly
labeled **non-canonical inspection**.

The report answers exactly **three questions**:

1. **Objective 1 — Startup readiness signals.** What appears after starting the system that confirms
   it is ready to handle user authentication and alias-based email activity?
2. **Objective 2 — New-user product walkthrough.** Register → verify → login → dashboard, with the
   visible confirmation at each transition (plus the error/edge states).
3. **Objective 3 — Behind-the-scenes background jobs / internal services.** The runtime indicators
   that background jobs and internal services are active and communicating.

---

## 1. TL;DR — Health Verdict

**HEALTHY for local development.** All runtime tiers of SimpleLogin were booted in the canonical
local configuration and confirmed live over their real interfaces:

| Tier | Entry point | Readiness signal (observed) | Verdict |
|------|-------------|------------------------------|---------|
| Web / auth | `python server.py` (`:7777`) | Flask banner + `* Debug mode: on`; live HTTP 200/302; `auth`/`dashboard` blueprints mounted | ✅ live |
| Email / alias | `python email_handler.py` (`:20381`) | `Listen for port 20381` + `Start mail controller 0.0.0.0 20381`; live SMTP banner `220 <hostname> Python SMTP 1.4.2` | ✅ live |
| Background jobs | `python job_runner.py` | 10-second poll loop; `Take job` + `Job.state` `ready→taken→done` | ✅ live |
| Scheduler | `python cron.py -j <job>` | `Start running cronjob`; 15-entry yacron schedule in `crontab.yml` | ✅ live |
| Event listener | `python event_listener.py listener` | `Using PostgresEventSource`; `Starting to listen to events` | ✅ live |
| Datastore | PostgreSQL 13 | schema migrated to head `32f25cbf12f6`; 77 public tables | ✅ live |

- **User authentication** was exercised end-to-end over real HTTP: register → activation email
  (metadata printed to the log under the default) → verify (`User.activated` flips `False`→`True`)
  → login (HTTP 302 into the dashboard) → authenticated dashboard render. All error/edge paths,
  password boundaries, the resend workflow, and one-time activation-code consumption were reproduced.
- **Alias-based email activity** was exercised through the real aiosmtpd SMTP controller on `:20381`
  via `swaks`; the full forward pipeline (`handle()` → `handle_forward()` →
  `forward_email_to_mailbox()`) executed and returned `250 Message accepted for delivery`.
- **Background jobs / internal services** were confirmed live: the GDPR export job was picked up by
  `job_runner.py` (`ready→taken→done`), the yacron/`cron.py` entry point ran, the event listener
  subscribed to the Postgres channel, and the event dispatcher's default no-op guard was observed
  firing.

### 1.1 Scope of this verdict — local development only (NOT a production-security assessment)

**This `HEALTHY` verdict means functional local-development readiness only.** It is explicitly
**not** a statement that the observed configuration is production-hardened, and it is **not** a
production-security assessment. The instance was run with the repository's default local
`example.env`, which is intentionally insecure for convenience. The following concrete defaults were
observed and **must be changed before any production/self-host exposure**:

- **`FLASK_SECRET=secret`** [`example.env:L77`] — a well-known, guessable session-signing key; must
  be a strong random secret in production.
- **Flask **debug** development server** — `app.run(debug=True, port=7777)` [`server.py:L588`] prints
  `WARNING: This is a development server. Do not use it in a production deployment.` (observed in
  §3.1). Production uses `gunicorn wsgi:app` [`Dockerfile` CMD].
- **Demo/seed credentials** — `john@wick.com` / `password` [`app/fake_data.py:L45-L47`] and
  `winston@continental.com` [`app/fake_data.py:L236-L237`] are seeded by `flask dummy-data`; these
  must never exist in production.
- **`NOT_SEND_EMAIL=true`** [`example.env:L19`] — no email is actually delivered (activation/reset
  emails would not reach users); must be `false` with a real MTA in production.
- **No Redis wiring by default** — `MEM_STORE_URI` is unset/`None` [`app/config.py:L568`], so the web
  app uses an in-memory rate-limiter and signed-cookie sessions rather than a shared Redis store
  (§3.1); a multi-process production deployment needs `MEM_STORE_URI` configured.
- **No TLS, no production DNS/MX/Postfix, no object storage, no captcha/APM backends** are provisioned
  — out of scope for local verification and required for production.

Four default-configuration nuances were observed against a naive expectation and are documented
inline (see §6): the activation "email" is a **log line of metadata** (subject/from/to), not a sent
message and not the rendered body; onboarding jobs are **suppressed** by default; application event
dispatch is a deliberate **no-op** under the default config; and the Werkzeug `* Running on http://127.0.0.1:7777`
banner is **suppressed** because the `werkzeug` logger is disabled.

---

## 2. Environment & Exact Commands

Every command in this report is reproducible. `<REPO>` is resolved throughout to the actual working
tree:

```
REPO=/tmp/blitzy/app/blitzy-d7c2d64b-4eea-4bae-9c05-a7e6d6e3887f_243015
```

### 2.1 Canonical stack (fixed versions)

Fixed stack: **Python 3.10**, **PostgreSQL 13**, **Redis 6**
[`.github/workflows/main.yml`: python `3.10` L17/L40; `image: postgres:13` L47; `redis-version: 6`
L92-94]. Versions reproduced at runtime from the container venv:

```
$ docker exec sl-web /app/venv/bin/python -c "import sys,flask,werkzeug,aiosmtpd,sqlalchemy,redis,arrow; \
    print('python', sys.version.split()[0]); print('flask', flask.__version__); \
    print('werkzeug', werkzeug.__version__); print('aiosmtpd', aiosmtpd.__version__); \
    print('sqlalchemy', sqlalchemy.__version__); print('redis', redis.__version__); print('arrow', arrow.__version__)"
python 3.10.18
flask 1.1.2
werkzeug 1.0.1
aiosmtpd 1.4.2
sqlalchemy 1.3.24
redis 4.6.0
arrow 0.16.0
```

### 2.2 Image provenance (base image + derived runtime image)

The canonical prepared image from the project setup instructions is used as the base, and a thin
**derived** image `sl-app:runtime` corrects one dependency deviation. **No source files and no
`pyproject.toml`/`poetry.lock` are modified** — only the runtime venv is corrected.

```
$ docker images --no-trunc ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0 \
    --format 'repo={{.Repository}}:{{.Tag}} id={{.ID}}'
repo=ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0 id=sha256:ea242796bbce36ca99ba9f783a4e7ac9d2ed3738e737f9a6d96bb22bbf1d9b58
$ docker images --no-trunc sl-app:runtime --format 'repo={{.Repository}}:{{.Tag}} id={{.ID}}'
repo=sl-app:runtime id=sha256:cf60370fa2c66a7cdc0cbdb594a9a2895edf38b9358268325433ab1bd5a1210e
```

The base image (setup-instruction reference
`andrewparkscaleai/coding-agent:simple-login__app__2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`,
published as `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`) shipped `google-re2` instead of the
locked `pyre2 0.3.6`; `google-re2` lacks `re2.DOTALL`, which breaks `import email_handler`. The
derived image reinstalls the locked `pyre2 0.3.6`. Exact derivation Dockerfile (verbatim):

```
$ cat /tmp/sl-runtime-build/Dockerfile
FROM ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0
# Bring the environment into compliance with poetry.lock (pyre2 0.3.6) which the
# image had substituted with google-re2 (missing re2.DOTALL -> breaks email_handler import).
# No source files and no lock/pyproject files are modified; only the runtime venv is corrected.
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update \
    && apt-get install -y --no-install-recommends build-essential libre2-dev pkg-config \
    && /app/venv/bin/pip install --no-input "Cython<3" \
    && /app/venv/bin/pip uninstall -y google-re2 || true
RUN /app/venv/bin/pip install --no-input --no-build-isolation "pyre2==0.3.6" 2>&1 | tail -40

$ cd /tmp/sl-runtime-build && docker build -t sl-app:runtime .   # (derivation build command)
```

Correction verified live (the locked `pyre2` provides `re2.DOTALL`, so `email_handler` imports):

```
$ docker exec sl-web /app/venv/bin/python -c "import re2; print('re2 module file:', re2.__file__); print('re2.DOTALL =', re2.DOTALL)"
re2 module file: /app/venv/lib/python3.10/site-packages/re2.cpython-310-x86_64-linux-gnu.so
re2.DOTALL = re.DOTALL
```

### 2.3 Infrastructure (sidecar containers) and live connectivity

```
# PostgreSQL 13 (published to host 127.0.0.1:15432 -> container 5432)
docker run -d --name sl-postgres -e POSTGRES_USER=myuser -e POSTGRES_PASSWORD=mypassword \
    -e POSTGRES_DB=simplelogin -p 127.0.0.1:15432:5432 postgres:13
# Redis 6 (published to host 127.0.0.1:6379 -> container 6379)
docker run -d --name sl-redis -p 127.0.0.1:6379:6379 redis:6
```

Live connectivity confirmed (host tooling: `psql`, `redis-cli`, `curl`, Python `socket`):

```
$ psql "postgresql://myuser:mypassword@localhost:15432/simplelogin" -tAc "SELECT 'pg', version();"
pg|PostgreSQL 13.23 (Debian 13.23-1.pgdg13+1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 14.2.0-19) 14.2.0, 64-bit
$ redis-cli -p 6379 PING
PONG
$ curl -s -o /dev/null -w "%{http_code} %{redirect_url}\n" http://localhost:7777/
302 http://localhost:7777/auth/login
$ python3 -c "import socket; s=socket.create_connection(('127.0.0.1',20381),timeout=5); print(s.recv(200).decode().strip()); s.sendall(b'QUIT\r\n'); print(s.recv(200).decode().strip()); s.close()"
220 reverse-code-generator-71c836b7-xjgzz Python SMTP 1.4.2
221 Bye
```

(The SMTP greeting hostname `reverse-code-generator-71c836b7-xjgzz` is the container hostname; the
aiosmtpd `1.4.2` version in the `220` banner matches the installed `aiosmtpd` in §2.1. `nc` is not
installed on the host, so a Python `socket` one-liner is used to read the SMTP banner.)

### 2.4 Configuration (default `example.env`) — exact diff

`.env` was created from `example.env` (`cp example.env .env`). The **only** deviation from
`example.env` is the Postgres port (`5432`→`15432`), so one Postgres serves both dev and the test
suite. Proven by diffing the two files (only `DB_URI` differs):

```
$ diff <(grep -vE '^\s*#|^\s*$' example.env | sort) <(grep -vE '^\s*#|^\s*$' .env | sort)
2c2
< DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin
---
> DB_URI=postgresql://myuser:mypassword@localhost:15432/simplelogin
```

Pivotal defaults (unchanged, confirmed on disk):

```
$ grep -nE '^(URL|NOT_SEND_EMAIL|EMAIL_DOMAIN|DB_URI|FLASK_SECRET|LOCAL_FILE_UPLOAD|DISABLE_ONBOARDING)=' .env
6:URL=http://localhost:7777
19:NOT_SEND_EMAIL=true
22:EMAIL_DOMAIN=sl.local
75:DB_URI=postgresql://myuser:mypassword@localhost:15432/simplelogin
77:FLASK_SECRET=secret
136:LOCAL_FILE_UPLOAD=true
150:DISABLE_ONBOARDING=true
```

`EVENT_WEBHOOK` defaults to `None` (unset) and `EVENT_WEBHOOK_DISABLE` defaults to `False`
[`app/config.py`]; `MEM_STORE_URI` defaults to `None` [`app/config.py:L568`].

**Two defaults change *what* the observable signals are — this report reflects the DEFAULT behavior:**

- `NOT_SEND_EMAIL=true` → the mail sender **logs metadata (subject/from/to) and returns without
  sending**; the rendered body/link is not logged [`app/mail_sender.py:L130-137`] (see §4.1/§6.1).
- `DISABLE_ONBOARDING=true` → onboarding jobs are **suppressed** at registration
  [`app/models.py:L646-648`] (see §5.2).

### 2.5 Application entry points — exact executable commands (all three services)

Each long-running service is a container started from the derived image with `<REPO>` bind-mounted at
`/workspace` and the venv interpreter as the entrypoint. All three are complete, executable commands
(not comments):

```
# Web app (Flask dev server on :7777)
docker run -d --name sl-web --network host -v $REPO:/workspace -w /workspace \
    -e PATH=/app/venv/bin:/usr/local/bin:/usr/bin:/bin -e VIRTUAL_ENV=/app/venv -e PYTHONUNBUFFERED=1 \
    --entrypoint /app/venv/bin/python sl-app:runtime server.py

# Email handler (aiosmtpd SMTP controller on :20381)
docker run -d --name sl-email --network host -v $REPO:/workspace -w /workspace \
    -e PATH=/app/venv/bin:/usr/local/bin:/usr/bin:/bin -e VIRTUAL_ENV=/app/venv -e PYTHONUNBUFFERED=1 \
    --entrypoint /app/venv/bin/python sl-app:runtime email_handler.py

# Background job runner (polling loop)
docker run -d --name sl-jobs --network host -v $REPO:/workspace -w /workspace \
    -e PATH=/app/venv/bin:/usr/local/bin:/usr/bin:/bin -e VIRTUAL_ENV=/app/venv -e PYTHONUNBUFFERED=1 \
    --entrypoint /app/venv/bin/python sl-app:runtime job_runner.py
```

Confirmed running (each `Args` is the canonical entry-point script):

```
$ docker inspect sl-web   --format 'mount: {{range .Mounts}}{{.Source}} -> {{.Destination}}{{end}} | entrypoint: {{.Config.Entrypoint}} | args: {{.Args}}'
mount: /tmp/blitzy/app/blitzy-d7c2d64b-4eea-4bae-9c05-a7e6d6e3887f_243015 -> /workspace | entrypoint: [/app/venv/bin/python] | args: [server.py]
$ docker inspect sl-email  --format 'args: {{.Args}}'
args: [email_handler.py]
$ docker inspect sl-jobs   --format 'args: {{.Args}}'
args: [job_runner.py]
```

One-off management commands (migration, seed, cron, listener) use the same image/mount but
`--rm` and the relevant venv entrypoint (`alembic`, `flask`, `python`), shown where used below.

### 2.6 Migrate + seed (canonical reset flow) — full output + exit status

The repository's own reset flow is `scripts/reset_local_db.sh`: `drop schema public cascade` →
`alembic upgrade head` → `flask dummy-data`. Each step was run and its exit status captured. (App
containers were stopped first to avoid `DROP SCHEMA` lock contention.)

**Step 1 — drop/recreate schema** (host `psql`):

```
$ echo 'drop schema public cascade; create schema public;' | psql "postgresql://myuser:mypassword@localhost:15432/simplelogin"
NOTICE:  drop cascades to 82 other objects
DETAIL:  drop cascades to table alembic_version
drop cascades to type plan_enum
drop cascades to table file
drop cascades to table users
drop cascades to table activation_code
drop cascades to table client
drop cascades to table alias
drop cascades to table authorization_code
drop cascades to table client_user
drop cascades to table oauth_token
drop cascades to table redirect_uri
drop cascades to table reset_password_code
drop cascades to table contact
drop cascades to type planenum2
drop cascades to table subscription
drop cascades to table email_log
drop cascades to table deleted_alias
drop cascades to table email_change
drop cascades to table api_key
drop cascades to table alias_used_on
drop cascades to table custom_domain
drop cascades to table lifetime_coupon
drop cascades to table directory
drop cascades to table job
drop cascades to table mailbox
drop cascades to table manual_subscription
drop cascades to table social_auth
drop cascades to table account_activation
drop cascades to table refused_email
drop cascades to table referral
drop cascades to type planenum_apple
drop cascades to table apple_subscription
drop cascades to table sent_alert
drop cascades to table alias_mailbox
drop cascades to table recovery_code
drop cascades to table domain_deleted_alias
drop cascades to table notification
drop cascades to table fido
drop cascades to table mfa_browser
drop cascades to table directory_mailbox
drop cascades to table public_domain
drop cascades to table domain_mailbox
drop cascades to table monitoring
drop cascades to table batch_import
drop cascades to table authorized_address
drop cascades to table coinbase_subscription
drop cascades to table bounce
drop cascades to table transactional_email
drop cascades to table metric2
drop cascades to table payout
drop cascades to table hibp
drop cascades to table alias_hibp
drop cascades to table ignored_email
drop cascades to table coupon
drop cascades to table hibp_notified_alias
drop cascades to table ignore_bounce_sender
drop cascades to extension pg_trgm
drop cascades to table auto_create_rule
drop cascades to table auto_create_rule__mailbox
drop cascades to table message_id_matching
drop cascades to table deleted_directory
drop cascades to table deleted_subdomain
drop cascades to table phone_country
drop cascades to table phone_number
drop cascades to table phone_message
drop cascades to table phone_reservation
drop cascades to table invalid_mailbox_domain
drop cascades to type block_behaviour_enum
drop cascades to table admin_audit_log
drop cascades to table provider_complaint
drop cascades to table partner
drop cascades to table partner_api_token
drop cascades to table partner_user
drop cascades to table partner_subscription
drop cascades to table newsletter
drop cascades to table newsletter_user
drop cascades to table api_cookie_token
drop cascades to table daily_metric
drop cascades to table sync_event
drop cascades to table mailbox_activation
drop cascades to table alias_audit_log
drop cascades to table user_audit_log
DROP SCHEMA
CREATE SCHEMA
$ echo "drop_exit=$?"
drop_exit=0
```

> All fenced blocks in this report are complete/unedited captured output. No block asserted as raw
> evidence contains inserted ellipses.

**Step 2 — `alembic upgrade head`** (one-off container) — **complete, unedited output** (255
`Running upgrade` steps, ending at head `32f25cbf12f6`; the head is also independently re-verified in
the resulting-state check below):

```
$ docker run --rm --network host -v $REPO:/workspace -w /workspace \
    -e PATH=/app/venv/bin:/usr/local/bin:/usr/bin:/bin -e VIRTUAL_ENV=/app/venv -e PYTHONUNBUFFERED=1 \
    --entrypoint /app/venv/bin/alembic sl-app:runtime upgrade head
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/uevaybwzujcsfwwerqcq
Upload files to local dir
>>> init logging <<<
2026-07-13 18:24:26,658 - SL - DEBUG - 1 - "/workspace/app/utils.py:17" - <module>() -  - load words file: /workspace/local_data/test_words.txt
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
$ echo "alembic_exit=$?"
alembic_exit=0
```

**Step 3 — `flask dummy-data`** (one-off container) — **complete, unedited output**:

```
$ docker run --rm --network host -v $REPO:/workspace -w /workspace \
    -e PATH=/app/venv/bin:/usr/local/bin:/usr/bin:/bin -e VIRTUAL_ENV=/app/venv -e PYTHONUNBUFFERED=1 \
    -e FLASK_APP=wsgi:app --entrypoint /app/venv/bin/flask sl-app:runtime dummy-data
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/nbfbolmofmibuuawwdtk
Upload files to local dir
>>> init logging <<<
2026-07-13 18:24:57,882 - SL - DEBUG - 1 - "/workspace/app/utils.py:17" - <module>() -  - load words file: /workspace/local_data/test_words.txt
2026-07-13 18:24:59,845 - SL - WARNING - 1 - "/workspace/server.py:494" - dummy_data() -  - reset db, add fake data
2026-07-13 18:24:59,845 - SL - DEBUG - 1 - "/workspace/app/fake_data.py:41" - fake_data() -  - create fake data
2026-07-13 18:25:00,236 - SL - INFO - 1 - "/workspace/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 18:25:00,239 - SL - DEBUG - 1 - "/workspace/app/models.py:647" - create() -  - Disable onboarding emails
2026-07-13 18:25:00,261 - SL - DEBUG - 1 - "/workspace/app/models.py:1459" - generate_random_alias_email() -  - generate email silken_venues217@sl.local
2026-07-13 18:25:00,273 - SL - INFO - 1 - "/workspace/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 18:25:00,326 - SL - INFO - 1 - "/workspace/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 18:25:00,335 - SL - INFO - 1 - "/workspace/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 18:25:00,353 - SL - INFO - 1 - "/workspace/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 18:25:00,364 - SL - INFO - 1 - "/workspace/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 18:25:00,380 - SL - INFO - 1 - "/workspace/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 18:25:00,389 - SL - INFO - 1 - "/workspace/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 18:25:00,400 - SL - DEBUG - 1 - "/workspace/app/models.py:1255" - generate_oauth_client_id() -  - generate oauth_client_id demo-jgktrfbihg
2026-07-13 18:25:00,408 - SL - DEBUG - 1 - "/workspace/app/models.py:1255" - generate_oauth_client_id() -  - generate oauth_client_id demo2-tinqmoklrn
2026-07-13 18:25:00,684 - SL - INFO - 1 - "/workspace/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 18:25:00,686 - SL - DEBUG - 1 - "/workspace/app/models.py:647" - create() -  - Disable onboarding emails
2026-07-13 18:25:00,710 - SL - INFO - 1 - "/workspace/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 18:25:00,723 - SL - INFO - 1 - "/workspace/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 18:25:00,733 - SL - INFO - 1 - "/workspace/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain
dummy_data_exit=0
```

**Resulting-state check** (independent verification the required executions took effect):

```
$ docker run --rm --network host -v $REPO:/workspace -w /workspace \
    -e PATH=/app/venv/bin:/usr/local/bin:/usr/bin:/bin -e VIRTUAL_ENV=/app/venv -e PYTHONUNBUFFERED=1 \
    --entrypoint /app/venv/bin/alembic sl-app:runtime current
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
32f25cbf12f6 (head)
$ psql "postgresql://myuser:mypassword@localhost:15432/simplelogin" -tAc \
    "SELECT (SELECT version_num FROM alembic_version), (SELECT count(*) FROM information_schema.tables WHERE table_schema='public');"
32f25cbf12f6|77
$ psql "postgresql://myuser:mypassword@localhost:15432/simplelogin" -tAc "SELECT id,email,activated FROM users ORDER BY id;"
1|john@wick.com|t
2|winston@continental.com|t
```

The seed log corroborates the **actual** `dummy-data` initialization call graph (see §3.4 for the
full analysis): `reset db, add fake data` [`server.py:L494`] → `fake_data()` [`app/fake_data.py:L41`]
→ `Add sl.local to SL domain` from `add_sl_domains()` [`init_app.py:L44`]. There is **no**
`Finish load_pgp_public_keys` line — `flask dummy-data` does **not** call `load_pgp_public_keys()`.

### 2.7 Pristine baseline snapshot (for cleanup verification, §8)

Immediately after the canonical reset+seed, the per-table row counts below define the **pristine
baseline** this investigation must return the database to. They are re-checked verbatim in §8.

```
$ for t in users alias mailbox contact email_log job sync_event activation_code alias_audit_log user_audit_log daily_metric; do \
    echo "$t=$(psql "postgresql://myuser:mypassword@localhost:15432/simplelogin" -tAc "SELECT count(*) FROM $t;")"; done
users=2
alias=11
mailbox=4
contact=1
email_log=1
job=0
sync_event=0
activation_code=0
alias_audit_log=11
user_audit_log=0
daily_metric=1
$ psql "postgresql://myuser:mypassword@localhost:15432/simplelogin" -tAc \
    "SELECT id,date,nb_new_web_non_proton_user,nb_alias FROM daily_metric ORDER BY id;"
1|2026-07-13|0|11
```

### 2.8 Environment caveats (documented, NOT "fixed")

- **`cbor2 5.2.0` sdist build failure on newer base OSes** [`poetry.lock:L411-412`] — broken PEP 517
  metadata plus a `pkg_resources` import that newer setuptools drops. This is an **environment
  caveat only**, resolved by using the canonical `python:3.10` image; `pyproject.toml`/`poetry.lock`
  were **never** edited.
- **`google-re2` → `pyre2 0.3.6`** — the base image substituted `google-re2` (no `re2.DOTALL`),
  which breaks `import email_handler`; corrected in the derived `sl-app:runtime` image (§2.2) without
  touching the lockfile.
- **Postgres port reconciliation** — `CONTRIBUTING.md` maps host `15432`→container `5432`, while
  `example.env:L75` `DB_URI` uses `localhost:5432`; the setup pointed `DB_URI` at `15432` so
  PostgreSQL is reachable where `DB_URI` points (§2.4). This is a setup detail, **not** a SimpleLogin
  defect. (Note: `CONTRIBUTING.md` also contains an unrelated `35432` typo in one command; the
  working value is `15432`.)
- **IPv6 sysctl** — the canonical environment requires `net.ipv6.conf.*.disable_ipv6=0` for a subset
  of `mail_sender` behavior; this is an environment setting, not a source concern.

---

## 3. Objective 1 — Startup Readiness Signals

**Direct answer.** After starting the three entry points, readiness for **(a) user authentication**
and **(b) alias-based email activity** is confirmed by these observed signals: the logging init
banner `>>> init logging <<<`; the Flask dev-server banner ending in `* Debug mode: on` with live
HTTP responses on `:7777`; the migrated PostgreSQL schema at head `32f25cbf12f6` (77 tables); the
`auth` and `dashboard` blueprints answering over HTTP; and the email handler's `Listen for port
20381` + `Start mail controller 0.0.0.0 20381` with a live SMTP banner on `:20381`.

All logs below were captured from a **fresh restart** (`docker restart sl-web sl-email sl-jobs`) so
each boot is clean and self-contained. Each block is the complete `docker logs <container> --since
<StartedAt>` output.

### 3.1 Web app — `python server.py` (`:7777`)

```
$ docker inspect sl-web --format '{{.State.StartedAt}}'
2026-07-13T18:33:05.941233035Z
$ docker logs sl-web --since 2026-07-13T18:33:05
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/teoglinajmafxcjxstcr
Upload files to local dir
>>> init logging <<<
2026-07-13 18:33:06,804 - SL - DEBUG - 1 - "/workspace/app/utils.py:17" - <module>() -  - load words file: /workspace/local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/mseciebswnbwgijcwlzt
Upload files to local dir
>>> init logging <<<
2026-07-13 18:33:08,583 - SL - DEBUG - 12 - "/workspace/app/utils.py:17" - <module>() -  - load words file: /workspace/local_data/test_words.txt
```

Signals identified:

- **Logging init banner** — `print(">>> init logging <<<")` at import time [`app/log.py:L67`]. The
  application logger is named `SL` [`app/log.py:L79`] and the log format
  (`%(asctime)s - %(name)s - %(levelname)s - %(process)d - "%(pathname)s:%(lineno)d" - %(funcName)s() - %(message_id)s - %(message)s`)
  is defined at [`app/log.py:L12-14`] — visible in every `SL` line above. **(Observed.)**
- **Two `>>> init logging <<<` banners** — because `debug=True` enables the Werkzeug reloader, which
  spawns a child; the parent runs as `process 1` and the reloader child as `process 12` (see
  `DEBUG - 1` vs `DEBUG - 12`). Each imports the app once, hence two banners. **(Observed.)**
- **Flask dev-server banner** — `create_app()` [`server.py:L139`]; when run directly, `__main__`
  [`server.py:L598`] calls `local_main()` [`server.py:L572`] which invokes
  `app.run(debug=True, port=7777)` [`server.py:L588`]. The Flask 1.1.2 banner
  (the `Serving Flask app "server"` line through `* Debug mode: on`, and the development-server WARNING) prints,
  confirming the dev server started on the configured port. **(Observed.)**
- **PostgreSQL connectivity (migrated schema)** — `SQLALCHEMY_DATABASE_URI = DB_URI`
  [`server.py:L146`]. Proof the schema is migrated and queryable:

```
$ psql "postgresql://myuser:mypassword@localhost:15432/simplelogin" -tAc \
    "SELECT (SELECT version_num FROM alembic_version), (SELECT count(*) FROM information_schema.tables WHERE table_schema='public');"
32f25cbf12f6|77
$ psql "postgresql://myuser:mypassword@localhost:15432/simplelogin" -tAc "SELECT id,email,activated FROM users ORDER BY id;"
1|john@wick.com|t
2|winston@continental.com|t
```

- **Redis connectivity (observed nuance).** The guard `if MEM_STORE_URI:` [`server.py:L163`] →
  `initialize_redis_services(app, MEM_STORE_URI)` [`server.py:L165`] governs Redis wiring. Under the
  default `.env`, `MEM_STORE_URI` is unset and defaults to `None` [`app/config.py:L568`], so the web
  app does **not** connect to Redis by default — flask-limiter uses an in-memory store and Flask
  sessions are signed cookies. Redis 6 is nonetheless provisioned and live for parity/the full test
  suite. Complete observed output (the import banner precedes the printed value):

```
$ docker exec sl-web /app/venv/bin/python -c "from app.config import MEM_STORE_URI; print('MEM_STORE_URI repr =', repr(MEM_STORE_URI))"
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/uokejstgasomphhgvmsm
Upload files to local dir
MEM_STORE_URI repr = None
$ redis-cli -p 6379 DBSIZE
295
```

- **Blueprint registration exposing `auth`/`dashboard`** — `register_blueprints()` [`server.py:L233`]
  registers `auth_bp` [`server.py:L234`], `dashboard_bp` [`server.py:L236`], plus
  `monitor`/`developer`/`phone`/`oauth` (`/oauth` L240 + `/oauth2` L241)/`onboarding`/`discover`/
  `internal`/`api` (through `server.py:L246`). Live proof over HTTP, with the matching SimpleLogin
  `after_request` [`server.py:L284`] server-side lines:

```
$ curl -s -o /dev/null -w "%{http_code}\n" http://localhost:7777/auth/login
200
$ curl -s -o /dev/null -w "%{http_code} %{redirect_url}\n" http://localhost:7777/dashboard/
302 http://localhost:7777/auth/login?next=%2Fdashboard%2F%3F
$ curl -s -o /dev/null -w "%{http_code} %{redirect_url}\n" http://localhost:7777/
302 http://localhost:7777/auth/login
$ docker logs sl-web --since 2026-07-13T18:33:30 2>&1 | grep after_request | tail -3
2026-07-13 18:34:08,964 - SL - DEBUG - 12 - "/workspace/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.13275384902954102
2026-07-13 18:34:08,975 - SL - DEBUG - 12 - "/workspace/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard/ ImmutableMultiDict([]) 302, takes 0.0008764266967773438
2026-07-13 18:34:08,983 - SL - DEBUG - 12 - "/workspace/server.py:284" - after_request() -  - 127.0.0.1 GET / ImmutableMultiDict([]) 302, takes 0.0006306171417236328
```

`GET /auth/login` → 200 (the `auth` blueprint renders the login page); `GET /dashboard/` → 302 to
`/auth/login?next=%2Fdashboard%2F%3F` (the `dashboard` blueprint is mounted and its `@login_required` guard
[`app/dashboard/views/index.py:L56`] redirects the unauthenticated request). **(Observed.)**

### 3.2 Email handler — `python email_handler.py` (`:20381`)

```
$ docker inspect sl-email --format '{{.State.StartedAt}}'
2026-07-13T18:33:16.141136155Z
$ docker logs sl-email --since 2026-07-13T18:33:16
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/pygebdflbljkrtcwdpom
Upload files to local dir
>>> init logging <<<
2026-07-13 18:33:17,057 - SL - DEBUG - 1 - "/workspace/app/utils.py:17" - <module>() -  - load words file: /workspace/local_data/test_words.txt
2026-07-13 18:33:17,755 - SL - INFO - 1 - "/workspace/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-13 18:33:17,757 - SL - DEBUG - 1 - "/workspace/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

Signals identified:

- **`Listen for port 20381`** — `LOG.i("Listen for port %s", args.port)` [`email_handler.py:L2403`];
  the argparse default port is `20381` [`email_handler.py:L2399`]. **(Observed.)**
- **`Start mail controller 0.0.0.0 20381`** —
  `LOG.d("Start mail controller %s %s", controller.hostname, controller.port)`
  [`email_handler.py:L2386`], emitted by `main(port)` [`email_handler.py:L2381`] after constructing
  `Controller(MailHandler(), hostname="0.0.0.0", port=port)` [`email_handler.py:L2383`]. This is the
  readiness signal for **alias-based email activity**: the aiosmtpd controller is bound and accepting
  SMTP on `:20381`. Confirmed by the live SMTP banner (§2.3, `220 <hostname> Python SMTP 1.4.2`). **(Observed.)**
- **PGP not loaded at boot (observed nuance, default config).** `load_pgp_public_keys()`
  [`email_handler.py:L2390`] is gated by `if LOAD_PGP_EMAIL_HANDLER:` [`email_handler.py:L2388`], and
  `LOAD_PGP_EMAIL_HANDLER = "LOAD_PGP_EMAIL_HANDLER" in os.environ` [`app/config.py:L339`] is `False`
  by default — so no `Finish load_pgp_public_keys` line is emitted here (confirmed: it is absent from
  the boot log above). The process then loops `while True: time.sleep(2)`
  [`email_handler.py:L2392-2393`]. **(Observed.)**

### 3.3 Background job runner — `python job_runner.py`

```
$ docker inspect sl-jobs --format '{{.State.StartedAt}}'
2026-07-13T18:33:26.319814907Z
$ docker logs sl-jobs --since 2026-07-13T18:33:26
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/omxmkblrayxgmmycfbpi
Upload files to local dir
>>> init logging <<<
2026-07-13 18:33:27,068 - SL - DEBUG - 1 - "/workspace/app/utils.py:17" - <module>() -  - load words file: /workspace/local_data/test_words.txt
```

After the banner + word-list load, the `__main__` `while True` loop [`job_runner.py:L330`] runs
inside `create_light_app().app_context()` [`job_runner.py:L332`]; with no ready jobs
(`get_jobs_to_run()` [`job_runner.py:L307`] returns none) it logs nothing and sleeps 10 seconds
[`job_runner.py:L347`]. The loop actively picking up work (the `Take job` log line) is demonstrated
in **Objective 3 §5.1**. **(Observed: silent 10s loop.)**

### 3.4 One-time data initialization — corrected call graph (M4-10)

There is **no** `init_app.local_init` function. The one-time initialization helpers live in
`init_app.py` and are invoked from two distinct entry points with **different** call sets:

- **`flask dummy-data`** (the `@app.cli.command("dummy-data")` handler [`server.py:L490-497`]) calls
  exactly `fake_data()`, then `add_sl_domains()`, then `add_proton_partner()` [imported from
  `init_app` at `server.py:L492`; calls at `server.py:L495-497`]. It does **not** call
  `load_pgp_public_keys()`. **(Observed** — the §2.6 seed log shows
  `Add sl.local to SL domain` [`init_app.py:L44`] from `add_sl_domains()`, and **no**
  `Finish load_pgp_public_keys` line.)
- **`python init_app.py`** (the module's `__main__` block [`init_app.py:L70-73`]) calls
  `load_pgp_public_keys()` [`init_app.py:L72`] and `add_sl_domains()` [`init_app.py:L73`]. It does
  **not** call `add_proton_partner()`. **(Source-derived from `init_app.py:L70-73`;** this path is
  not exercised by the default local run, which uses `flask dummy-data`.)

The helper functions themselves: `load_pgp_public_keys()` [`init_app.py:L13`] logs
`Finish load_pgp_public_keys` [`init_app.py:L36`]; `add_sl_domains()` [`init_app.py:L39`] logs
`Add %s to SL domain` [`init_app.py:L44`]; `add_proton_partner()` [`init_app.py:L59`] creates the
Proton partner row silently if absent (no log line). None of these run at web-server boot — the boot
logs in §3.1–§3.3 contain **0** `Finish load_pgp_public_keys` lines.


## 4. Objective 2 — New-User Product Walkthrough (register → verify → login → dashboard)

This objective drives the **real web routes** end-to-end over HTTP against the running dev server
(`http://localhost:7777`, `python server.py` from §2.5) using a host `python3` + `requests` session
with a cookie jar and CSRF tokens parsed from each served form. Every transition below is a captured
HTTP response (status, `Location`, body excerpt) correlated with the `sl-web` container log and the
PostgreSQL row state; all direct DB reads are labelled **NON-CANONICAL** inspection.

**Default-configuration nuances that shape the observable signals (all from `example.env`):**

- **`NOT_SEND_EMAIL=true`** — the activation email is **not transmitted**. `MailSender.send()`
  [`app/mail_sender.py:L126`] takes the `if config.NOT_SEND_EMAIL:` branch
  [`app/mail_sender.py:L130`] and logs **only** the subject/from/to metadata
  [`app/mail_sender.py:L131-136`], then `return True` [`app/mail_sender.py:L137`]. The activation
  **link** is therefore never written to the log — this is the crux of **M4-05** (see §4.3).
- **Real-MX requirement** — `email_can_be_used_as_mailbox()` rejects an address whose domain has no
  MX record (default `SKIP_MX_LOOKUP_ON_CHECK=False`), so the temporary users use `@gmail.com`. MX
  confirmed live inside `sl-web` via the app's own resolver — 5 records: `gmail-smtp-in.l.google.com`,
  `alt1.gmail-smtp-in.l.google.com`, `alt2.gmail-smtp-in.l.google.com`,
  `alt3.gmail-smtp-in.l.google.com`, `alt4.gmail-smtp-in.l.google.com`.
- **`canonicalize_email()`** [`app/utils.py:L78`] only rewrites gmail/proton addresses (strips
  `+tag`, removes dots, lowercases). The test addresses contain no `.`/`+`, so canonicalization is a
  no-op and the stored email equals the submitted email.

The activation-email **subject** is `Just one more step to join SimpleLogin`
[`app/email_utils.py:L128`], observed verbatim in the metadata log line in §4.3.

### 4.1 Happy-path driver script (published) — `/tmp/obs/obj2_happy.py`

Full, unedited script body (temporary; written to `/tmp/obs`, never committed):

```python
#!/usr/bin/env python3
"""
Objective 2 - happy path driver: register -> verify (activate) -> login -> dashboard.
Drives the REAL web routes over HTTP (http://localhost:7777) using a requests.Session
with a cookie jar and CSRF tokens parsed from the served HTML forms.

Weaves in:
  * M4-05: under NOT_SEND_EMAIL=true the activation *link* is NOT written to the log
    (only subject/from/to metadata is), so the activation code is read from the
    activation_code table as a NON-CANONICAL inspection to derive the URL the user
    would click; the GET /auth/activate call itself is the CANONICAL activation path.
  * M4-06: activation_code row present BEFORE activation, deleted AFTER, and a replay
    of the same code returns HTTP 400.
  * User.activated state transition captured BEFORE (False) and AFTER (True).
"""
import os
import re
import time
import subprocess

import requests

BASE = "http://localhost:7777"
EPOCH = int(time.time())
EMAIL = f"blitzyobs{EPOCH}@gmail.com"        # gmail.com => real MX; no dots/plus so canonicalize is a no-op
PASSWORD = "Blitzy-Obs-Passw0rd"             # length 20, within RegisterForm Length(min=8,max=100)

S = requests.Session()


def rule(title):
    print("\n" + "=" * 78)
    print(title)
    print("=" * 78)


def db(sql):
    """NON-CANONICAL inspection helper - direct psql read against the dev DB."""
    env = dict(os.environ, PGPASSWORD="mypassword")
    p = subprocess.run(
        ["psql", "-h", "localhost", "-p", "15432", "-U", "myuser", "-d", "simplelogin",
         "-tA", "-c", sql],
        capture_output=True, text=True, env=env,
    )
    return (p.stdout + p.stderr).strip()


def weblogs(since_epoch):
    p = subprocess.run(
        ["docker", "logs", "sl-web", "--since", str(since_epoch)],
        capture_output=True, text=True,
    )
    return p.stdout + p.stderr


def find_csrf(html):
    m = re.search(r'name="csrf_token"[^>]*value="([^"]+)"', html)
    if not m:
        m = re.search(r'value="([^"]+)"[^>]*name="csrf_token"', html)
    return m.group(1) if m else None


def get_csrf(path):
    r = S.get(BASE + path)
    return r, find_csrf(r.text)


def show(label, r):
    print(f"--- {label} ---")
    print(f"REQUEST : {r.request.method} {r.request.url}")
    print(f"STATUS  : {r.status_code}")
    if r.history:
        print(f"HISTORY : {[(h.status_code, h.headers.get('Location')) for h in r.history]}")
    loc = r.headers.get("Location")
    if loc:
        print(f"Location: {loc}")


print(f"TEST EMAIL    : {EMAIL}")
print(f"TEST PASSWORD : {PASSWORD!r} (len={len(PASSWORD)})")

# ---------------------------------------------------------------------------
rule("STEP 1  GET /auth/register  (fetch form + CSRF token)")
r, token = get_csrf("/auth/register")
show("GET /auth/register", r)
print(f"csrf_token present: {token is not None} (len={len(token) if token else 0})")

# ---------------------------------------------------------------------------
rule("STEP 2  POST /auth/register  (create user; capture mail metadata log)")
since = int(time.time())
r = S.post(BASE + "/auth/register",
           data={"csrf_token": token, "email": EMAIL, "password": PASSWORD},
           allow_redirects=False)
time.sleep(1.0)
show("POST /auth/register", r)
excerpt = re.sub(r"\s+", " ", r.text)
idx = excerpt.lower().find("activation")
print("BODY EXCERPT (register_waiting_activation):")
print("  " + (excerpt[max(0, idx - 60):idx + 160] if idx >= 0 else excerpt[:220]))

print("\nSERVER LOG (sl-web) for the register action:")
logs = weblogs(since)
for line in logs.splitlines():
    if ("create user" in line or "send email with subject" in line
            or "activate" in line.lower()):
        print("  " + line)

print("\nM4-05 CHECK - is the activation *link* present in the log? (expected: NO)")
link_hits = [l for l in logs.splitlines() if "/auth/activate?code=" in l]
print(f"  activation-link occurrences in sl-web log = {len(link_hits)}")

# ---------------------------------------------------------------------------
rule("STEP 3  DB state after register (NON-CANONICAL inspection)")
uid = db(f"select id from users where email='{EMAIL}';")
print(f"user id                = {uid}")
print(f"User.activated BEFORE  = {db(f'select activated from users where id={uid};')}  (expect f)")
code = db(f"select code from activation_code where user_id={uid};")
print(f"activation_code.code   = {code}  (len={len(code)})  [NON-CANONICAL DB read]")
print(f"activation_code rows   = {db(f'select count(*) from activation_code where user_id={uid};')}")
activate_url = f"{BASE}/auth/activate?code={code}"
print(f"\nDERIVED activation URL (the link the user would click):\n  {activate_url}")

# ---------------------------------------------------------------------------
rule("STEP 4  GET /auth/activate?code=<code>  (CANONICAL verification path)")
r = S.get(activate_url, allow_redirects=False)
show("GET /auth/activate", r)
print(f"User.activated AFTER   = {db(f'select activated from users where id={uid};')}  (expect t)")
print(f"activation_code rows AFTER (M4-06 deletion) = "
      f"{db(f'select count(*) from activation_code where user_id={uid};')}  (expect 0)")

# ---------------------------------------------------------------------------
rule("STEP 5  M4-06 replay - GET /auth/activate with the now-consumed code (expect 400)")
S2 = requests.Session()  # fresh session: the consumed code must not activate again
r = S2.get(activate_url, allow_redirects=False)
print(f"REQUEST : GET {activate_url}")
print(f"STATUS  : {r.status_code}  (expect 400)")
m = re.search(r'(Activation code cannot be found|Activation code was expired)', r.text)
print(f"error in body: {m.group(1) if m else '(none)'}")

# ---------------------------------------------------------------------------
rule("STEP 6  GET /auth/login  (fetch form + CSRF)")
r, token = get_csrf("/auth/login")
show("GET /auth/login", r)
print(f"csrf_token present: {token is not None}")

# ---------------------------------------------------------------------------
rule("STEP 7  POST /auth/login  (authenticate; expect 302 -> /dashboard/)")
S = requests.Session()               # brand-new session => login issues its own auth cookie
r, token = get_csrf("/auth/login")
since = int(time.time())
r = S.post(BASE + "/auth/login",
           data={"csrf_token": token, "email": EMAIL, "password": PASSWORD},
           allow_redirects=False)
time.sleep(0.8)
show("POST /auth/login", r)
print("\nSERVER LOG (sl-web) for the login action:")
for line in weblogs(since).splitlines():
    if "log user" in line or "dashboard" in line.lower():
        print("  " + line)

# ---------------------------------------------------------------------------
rule("STEP 8  GET /dashboard/  (authenticated landing; expect 200)")
r = S.get(BASE + "/dashboard/", allow_redirects=False)
show("GET /dashboard/", r)
flat = re.sub(r"\s+", " ", r.text)
for needle in ("Alias", "Mailbox", "Log out", "Newsletter"):
    print(f"  body contains {needle!r}: {needle in flat}")
title = re.search(r"<title>(.*?)</title>", r.text, re.S)
print("  <title>:", title.group(1).strip() if title else "(none)")

print(f"\nDONE. Temp user id={uid} email={EMAIL} (removed by the §8 canonical reset).")
```

### 4.2 Happy-path — complete raw output

Command that produced it:

```bash
python3 /tmp/obs/obj2_happy.py
```

Complete, unedited output:

```text
TEST EMAIL    : blitzyobs1783968727@gmail.com
TEST PASSWORD : 'Blitzy-Obs-Passw0rd' (len=19)

==============================================================================
STEP 1  GET /auth/register  (fetch form + CSRF token)
==============================================================================
--- GET /auth/register ---
REQUEST : GET http://localhost:7777/auth/register
STATUS  : 200
csrf_token present: True (len=91)

==============================================================================
STEP 2  POST /auth/register  (create user; capture mail metadata log)
==============================================================================
--- POST /auth/register ---
REQUEST : POST http://localhost:7777/auth/register
STATUS  : 200
BODY EXCERPT (register_waiting_activation):
  ical" href="http://localhost:7777/auth/register" /> <title> Activation Email Sent | SimpleLogin </title> <link rel="stylesheet" href="/static/node_modules/font-awesome/css/font-awesome.css" /> <!-- Dashboard Core --> <li

SERVER LOG (sl-web) for the register action:
  2026-07-13 18:52:08,041 - SL - DEBUG - 12 - "/workspace/app/auth/views/register.py:85" - register() -  - create user blitzyobs1783968727@gmail.com
  2026-07-13 18:52:08,345 - SL - DEBUG - 12 - "/workspace/app/mail_sender.py:131" - send() -  - send email with subject 'Just one more step to join SimpleLogin', from '"noreply@sl.local" <noreply@sl.local>' to 'blitzyobs1783968727@gmail.com'

M4-05 CHECK - is the activation *link* present in the log? (expected: NO)
  activation-link occurrences in sl-web log = 0

==============================================================================
STEP 3  DB state after register (NON-CANONICAL inspection)
==============================================================================
user id                = 9
User.activated BEFORE  = f  (expect f)
activation_code.code   = capmhjausywzxthindkcftjhortyrx  (len=30)  [NON-CANONICAL DB read]
activation_code rows   = 1

DERIVED activation URL (the link the user would click):
  http://localhost:7777/auth/activate?code=capmhjausywzxthindkcftjhortyrx

==============================================================================
STEP 4  GET /auth/activate?code=<code>  (CANONICAL verification path)
==============================================================================
--- GET /auth/activate ---
REQUEST : GET http://localhost:7777/auth/activate?code=capmhjausywzxthindkcftjhortyrx
STATUS  : 302
Location: http://localhost:7777/dashboard/
User.activated AFTER   = t  (expect t)
activation_code rows AFTER (M4-06 deletion) = 0  (expect 0)

==============================================================================
STEP 5  M4-06 replay - GET /auth/activate with the now-consumed code (expect 400)
==============================================================================
REQUEST : GET http://localhost:7777/auth/activate?code=capmhjausywzxthindkcftjhortyrx
STATUS  : 400  (expect 400)
error in body: Activation code cannot be found

==============================================================================
STEP 6  GET /auth/login  (fetch form + CSRF)
==============================================================================
--- GET /auth/login ---
REQUEST : GET http://localhost:7777/dashboard/
STATUS  : 200
HISTORY : [(302, 'http://localhost:7777/dashboard/')]
csrf_token present: True

==============================================================================
STEP 7  POST /auth/login  (authenticate; expect 302 -> /dashboard/)
==============================================================================
--- POST /auth/login ---
REQUEST : POST http://localhost:7777/auth/login
STATUS  : 302
Location: http://localhost:7777/dashboard/

SERVER LOG (sl-web) for the login action:
  2026-07-13 18:52:10,088 - SL - DEBUG - 12 - "/workspace/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard/ ImmutableMultiDict([]) 200, takes 0.22080183029174805
  2026-07-13 18:52:10,374 - SL - DEBUG - 12 - "/workspace/app/auth/views/login_utils.py:35" - after_login() -  - log user <User 9 blitzyobs1783968727@gmail.com blitzyobs1783968727@gmail.com> in
  2026-07-13 18:52:10,374 - SL - DEBUG - 12 - "/workspace/app/auth/views/login_utils.py:44" - after_login() -  - redirect user to dashboard

==============================================================================
STEP 8  GET /dashboard/  (authenticated landing; expect 200)
==============================================================================
--- GET /dashboard/ ---
REQUEST : GET http://localhost:7777/dashboard/
STATUS  : 200
  body contains 'Alias': True
  body contains 'Mailbox': True
  body contains 'Log out': False
  body contains 'Newsletter': False
  <title>: Alias
      | SimpleLogin

DONE. Temp user id=9 email=blitzyobs1783968727@gmail.com (removed by the §8 canonical reset).
```

**Note on STEP 8 body checks.** The authenticated dashboard render is confirmed by HTTP **200**, the
page `<title>` `Alias | SimpleLogin`, and the presence of the `Alias` and `Mailbox` strings. The
`Log out`/`Newsletter` checks print `False` only because those were exact-string heuristics that do
not match the template's actual wording; they are not failures.

### 4.3 M4-05 — the activation link is NOT logged; the DB read is NON-CANONICAL

Under `NOT_SEND_EMAIL=true`, the only mail evidence in the `sl-web` log is the **metadata** line
emitted by `MailSender.send()` [`app/mail_sender.py:L131-136`] — captured verbatim from §4.2 STEP 2:

```text
2026-07-13 18:52:08,345 - SL - DEBUG - 12 - "/workspace/app/mail_sender.py:131" - send() -  - send email with subject 'Just one more step to join SimpleLogin', from '"noreply@sl.local" <noreply@sl.local>' to 'blitzyobs1783968727@gmail.com'
```

The activation **link** `{URL}/auth/activate?code=<code>` is built at
[`app/auth/views/register.py:L124`] and passed to the template, but it is **never printed** — the
driver's `M4-05 CHECK` counted **0** occurrences of `/auth/activate?code=` in the `sl-web` log
(§4.2 STEP 2). To obtain the code the user would have received by email, the driver reads the
`activation_code` row directly. This is an explicitly **NON-CANONICAL inspection** — it bypasses the
email channel that is disabled by config, and is used only to reconstruct the link the user would
click (§4.2 STEP 3):

```text
activation_code.code   = capmhjausywzxthindkcftjhortyrx  (len=30)  [NON-CANONICAL DB read]
http://localhost:7777/auth/activate?code=capmhjausywzxthindkcftjhortyrx
```

The code is a 30-character token from `random_string(30)` [`app/auth/views/register.py:L120`]. The
**canonical** verification step is the subsequent `GET /auth/activate?code=<code>` HTTP request
(§4.2 STEP 4), which returned **HTTP 302 → `/dashboard/`**.

**Real side effect observed at STEP 6:** `activate()` calls `login_user(user)`
[`app/auth/views/activate.py:L50`], so the browser session is already authenticated immediately after
activation — a subsequent `GET /auth/login` on that same session **302-redirects to `/dashboard/`**
(visible in §4.2 STEP 6, where the request is recorded as reaching `/dashboard/` with a 302 in its
history).

### 4.4 M4-06 — activation-code lifecycle (present → deleted → replay 400)

`activate()` sets `user.activated = True` [`app/auth/views/activate.py:L49`], then **deletes** the
code via `ActivationCode.delete(activation_code.id)` [`app/auth/views/activate.py:L53`] followed by
`Session.commit()` [`app/auth/views/activate.py:L54`]. The before/during/after row state (all from
§4.2, same user identity):

| Point in flow | `activation_code` rows for the user | `User.activated` | Evidence |
|---------------|-------------------------------------|------------------|----------|
| After register, before activate | **1** (30-char code, §4.2 STEP 3) | **f** | §4.2 STEP 3 |
| After `GET /auth/activate` | **0** | **t** | §4.2 STEP 4 |
| Replay the same code (fresh session) | n/a — row gone | — | **HTTP 400 "Activation code cannot be found"** [`app/auth/views/activate.py:L28-36`], §4.2 STEP 5 |

The replay returns 400 precisely because the row was deleted — demonstrating single-use consumption.

### 4.5 State transition + confirmation signal at each hop

| Transition | Route / entry point | HTTP | Confirmation signal | Citation |
|-----------|---------------------|------|---------------------|----------|
| Register | `POST /auth/register` | 200 | renders `register_waiting_activation.html` (title `Activation Email Sent \| SimpleLogin`); log `create user %s` | [`register.py:L85,L104`] |
| Activate | `GET /auth/activate?code=<code>` | 302 → `/dashboard/` | `User.activated` f → t; flash "Your account has been activated" (success) | [`activate.py:L49,L56,L66-67`] |
| Login | `POST /auth/login` | 302 → `/dashboard/` | logs `log user %s in`, `redirect user to dashboard` | [`login_utils.py:L35,L44-45`] |
| Dashboard | `GET /dashboard/` | 200 | authenticated render, title `Alias \| SimpleLogin` | [`dashboard/views/index.py:L55-56`] |

**Observed evidence for the activation-success flash.** The happy-path driver used
`allow_redirects=False`, so the `flash("Your account has been activated", "success")`
[`app/auth/views/activate.py:L56`] that is set just before the 302 was not yet rendered. Following
the redirect into the dashboard renders it — SimpleLogin emits flash messages as `toastr` calls.

Supplement script (published; temporary, never committed) — `/tmp/obs/obj2_flash.py`:

```python
#!/usr/bin/env python3
"""
Objective 2 supplement: observe the 'Your account has been activated' success flash
[app/auth/views/activate.py:L56]. The happy-path driver used allow_redirects=False, so
the flash (set just before the 302) was never rendered. Here we FOLLOW the activate
redirect into the dashboard so flask renders the flashed message, and capture it.
"""
import os, re, time, subprocess, requests

BASE = "http://localhost:7777"
EPOCH = int(time.time())
EMAIL = f"flashobs{EPOCH}@gmail.com"
PASSWORD = "Flash-Obs-Passw0rd"


def db(sql):
    env = dict(os.environ, PGPASSWORD="mypassword")
    p = subprocess.run(["psql", "-h", "localhost", "-p", "15432", "-U", "myuser",
                        "-d", "simplelogin", "-tA", "-c", sql],
                       capture_output=True, text=True, env=env)
    return (p.stdout + p.stderr).strip()


def find_csrf(html):
    m = re.search(r'name="csrf_token"[^>]*value="([^"]+)"', html)
    return m.group(1) if m else None


s = requests.Session()
tok = find_csrf(s.get(BASE + "/auth/register").text)
s.post(BASE + "/auth/register",
       data={"csrf_token": tok, "email": EMAIL, "password": PASSWORD},
       allow_redirects=False)
time.sleep(0.6)
uid = db(f"select id from users where email='{EMAIL}';")
code = db(f"select code from activation_code where user_id={uid};")
print(f"fresh user id={uid} email={EMAIL}")
print(f"activation code (NON-CANONICAL DB read) = {code}")

# CANONICAL activation, FOLLOWING the redirect so the flash renders on the dashboard
r = s.get(f"{BASE}/auth/activate?code={code}", allow_redirects=True)
print(f"\nGET /auth/activate?code={code} (allow_redirects=True)")
print(f"final URL   : {r.url}")
print(f"status      : {r.status_code}")
print(f"redirect chain: {[(h.status_code, h.headers.get('Location')) for h in r.history]}")

flash_present = "Your account has been activated" in r.text
print(f"\nflash 'Your account has been activated' rendered on landing page: {flash_present}")
# print the exact surrounding snippet as evidence
idx = r.text.find("Your account has been activated")
if idx >= 0:
    snippet = re.sub(r"\s+", " ", r.text[idx - 40:idx + 60]).strip()
    print(f"snippet (verbatim from landing page): {snippet}")
```

Command that produced it:

```bash
python3 /tmp/obs/obj2_flash.py
```

Complete, unedited output:

```text
fresh user id=16 email=flashobs1783968975@gmail.com
activation code (NON-CANONICAL DB read) = cycfhrwdgzitaanfffmvhxlnkxlbym

GET /auth/activate?code=cycfhrwdgzitaanfffmvhxlnkxlbym (allow_redirects=True)
final URL   : http://localhost:7777/dashboard/
status      : 200
redirect chain: [(302, 'http://localhost:7777/dashboard/')]

flash 'Your account has been activated' rendered on landing page: True
snippet (verbatim from landing page): <script>toastr.success("Your account has been activated");</script>
```

### 4.6 Edge / error / boundary driver script (published) — `/tmp/obs/obj2_edges.py`

Full, unedited script body (temporary; written to `/tmp/obs`, never committed):

```python
#!/usr/bin/env python3
"""
Objective 2 - secondary / error / edge / boundary conditions.

SECTION A  M4-07 password boundaries: register with password length 7/8/100/101
           RegisterForm password = validators.Length(min=8, max=100) [register.py:L27]
SECTION B  M4-07 resend activation: GET form + POST (new code, mail log, flash)
SECTION C  edge - wrong password on login       (flash "Email or password incorrect")
SECTION D  edge - login before activation       (flash "Please check your inbox for the activation email")
SECTION E  edge - invalid activation code        (HTTP 400 "cannot be found")
SECTION F  edge - expired activation code        (HTTP 400 "was expired") [expiry aged via NON-CANONICAL DB write]
SECTION G  edge - activate while authenticated   (HTTP 400 "You are already logged in")

DB reads/writes via psql are labelled NON-CANONICAL inspection/manipulation.
"""
import os
import re
import time
import subprocess

import requests

BASE = "http://localhost:7777"
EPOCH = int(time.time())


def rule(t):
    print("\n" + "=" * 78 + "\n" + t + "\n" + "=" * 78)


def db(sql):
    env = dict(os.environ, PGPASSWORD="mypassword")
    p = subprocess.run(
        ["psql", "-h", "localhost", "-p", "15432", "-U", "myuser", "-d", "simplelogin",
         "-tA", "-c", sql],
        capture_output=True, text=True, env=env)
    return (p.stdout + p.stderr).strip()


def weblogs(since):
    p = subprocess.run(["docker", "logs", "sl-web", "--since", str(since)],
                       capture_output=True, text=True)
    return p.stdout + p.stderr


def find_csrf(html):
    m = re.search(r'name="csrf_token"[^>]*value="([^"]+)"', html)
    if not m:
        m = re.search(r'value="([^"]+)"[^>]*name="csrf_token"', html)
    return m.group(1) if m else None


def title_of(html):
    m = re.search(r"<title>(.*?)</title>", html, re.S)
    return re.sub(r"\s+", " ", m.group(1)).strip() if m else "(none)"


def flash_texts(html):
    # SimpleLogin renders flashes inside alert elements; capture visible text
    found = []
    for pat in ("Email or password incorrect",
                "Please check your inbox for the activation email",
                "An activation email has been sent to you",
                "Field must be between 8 and 100 characters long"):
        if pat in html:
            found.append(pat)
    return found


def register(email, password):
    s = requests.Session()
    r0 = s.get(BASE + "/auth/register")
    tok = find_csrf(r0.text)
    since = int(time.time())
    r = s.post(BASE + "/auth/register",
               data={"csrf_token": tok, "email": email, "password": password},
               allow_redirects=False)
    time.sleep(0.6)
    return s, r, since


# ===========================================================================
rule("SECTION A  M4-07 password boundaries (Length(min=8, max=100) [register.py:L27])")
for n in (7, 8, 100, 101):
    email = f"pwlen{n}_{EPOCH}@gmail.com"
    pw = "P" + "a" * (n - 1)          # exactly n characters
    _, r, _ = register(email, pw)
    created = db(f"select count(*) from users where email='{email}';")
    err = "Field must be between 8 and 100 characters long" in r.text
    print(f"\n[len={n:3d}] email={email}")
    print(f"  password length sent : {len(pw)}")
    print(f"  HTTP status          : {r.status_code}")
    print(f"  page <title>         : {title_of(r.text)}")
    print(f"  length-error in body : {err}")
    print(f"  user row created     : {created}   (1=created, 0=rejected)")

# ===========================================================================
rule("SECTION B  M4-07 resend activation (GET form + POST) [resend_activation.py:L17-42]")
# make a fresh UNACTIVATED user
remail = f"resend_{EPOCH}@gmail.com"
_, r, _ = register(remail, "Resend-Passw0rd!")
ruid = db(f"select id from users where email='{remail}';")
code_before = db(f"select code from activation_code where user_id={ruid};")
print(f"unactivated user id={ruid} email={remail}")
print(f"activation_code BEFORE resend = {code_before}")

s = requests.Session()
rg = s.get(BASE + "/auth/resend_activation")
rtok = find_csrf(rg.text)
print(f"\nGET /auth/resend_activation -> {rg.status_code}; csrf present: {rtok is not None}; title: {title_of(rg.text)}")

since = int(time.time())
rp = s.post(BASE + "/auth/resend_activation",
            data={"csrf_token": rtok, "email": remail}, allow_redirects=False)
time.sleep(0.6)
code_after = db(f"select code from activation_code where user_id={ruid};")
print(f"\nPOST /auth/resend_activation -> {rp.status_code}; title: {title_of(rp.text)}")
print(f"flash text(s) in body        : {flash_texts(rp.text)}")
print(f"activation_code AFTER resend = {code_after}")
print(f"code regenerated (changed)   : {code_before != code_after}")
print("server log (mail metadata for resend):")
for line in weblogs(since).splitlines():
    if "send email with subject" in line or "is not activated" in line:
        print("  " + line)

# ===========================================================================
rule("SECTION C  edge - wrong password on login [login.py:L45-50]")
# activate the SECTION-B user first so the failure is purely the password
acode = db(f"select code from activation_code where user_id={ruid};")
requests.Session().get(f"{BASE}/auth/activate?code={acode}", allow_redirects=False)
print(f"(pre-activated user {remail}; activated={db(f'select activated from users where id={ruid};')})")
s = requests.Session()
tok = find_csrf(s.get(BASE + "/auth/login").text)
since = int(time.time())
r = s.post(BASE + "/auth/login",
           data={"csrf_token": tok, "email": remail, "password": "totally-wrong-pw"},
           allow_redirects=False)
print(f"POST /auth/login (wrong pw) -> {r.status_code}; title: {title_of(r.text)}")
print(f"flash text(s) in body       : {flash_texts(r.text)}")
for line in weblogs(since).splitlines():
    if "LoginEvent" in line or "not verified" in line or "wrong password" in line.lower():
        print("  " + line)

# ===========================================================================
rule("SECTION D  edge - login before activation [login.py:L63-69]")
demail = f"unactivated_{EPOCH}@gmail.com"
dpw = "Unactivated-Passw0rd!"
_, r, _ = register(demail, dpw)
duid = db(f"select id from users where email='{demail}';")
print(f"fresh UNACTIVATED user id={duid} email={demail} activated={db(f'select activated from users where id={duid};')}")
s = requests.Session()
tok = find_csrf(s.get(BASE + "/auth/login").text)
r = s.post(BASE + "/auth/login",
           data={"csrf_token": tok, "email": demail, "password": dpw},
           allow_redirects=False)
print(f"POST /auth/login (not activated) -> {r.status_code}; title: {title_of(r.text)}")
print(f"flash text(s) in body            : {flash_texts(r.text)}")
print(f"resend-activation link offered   : {'/auth/resend_activation' in r.text}")

# ===========================================================================
rule("SECTION E  edge - invalid activation code (HTTP 400) [activate.py:L28-36]")
r = requests.Session().get(f"{BASE}/auth/activate?code=THIS-CODE-DOES-NOT-EXIST", allow_redirects=False)
m = re.search(r"(Activation code cannot be found|Activation code was expired)", r.text)
print(f"GET /auth/activate?code=THIS-CODE-DOES-NOT-EXIST -> {r.status_code}")
print(f"error in body: {m.group(1) if m else '(none)'}")

# ===========================================================================
rule("SECTION F  edge - expired activation code (HTTP 400) [activate.py:L38-46]")
femail = f"expired_{EPOCH}@gmail.com"
_, r, _ = register(femail, "Expired-Passw0rd!")
fuid = db(f"select id from users where email='{femail}';")
fcode = db(f"select code from activation_code where user_id={fuid};")
exp_before = db(f"select expired from activation_code where user_id={fuid};")
print(f"fresh user id={fuid}; code={fcode}")
print(f"activation_code.expired BEFORE = {exp_before}")
# NON-CANONICAL DB manipulation: age the expiry into the past so is_expired() is true
db(f"update activation_code set expired = now() - interval '2 hours' where user_id={fuid};")
exp_after = db(f"select expired from activation_code where user_id={fuid};")
print(f"activation_code.expired AFTER  = {exp_after}   [NON-CANONICAL: aged into the past]")
r = requests.Session().get(f"{BASE}/auth/activate?code={fcode}", allow_redirects=False)
m = re.search(r"(Activation code cannot be found|Activation code was expired)", r.text)
print(f"GET /auth/activate?code={fcode} -> {r.status_code}")
print(f"error in body: {m.group(1) if m else '(none)'}")
print(f"resend-activation offered (show_resend_activation): {'/auth/resend_activation' in r.text}")

# ===========================================================================
rule("SECTION G  edge - activate while already authenticated (HTTP 400) [activate.py:L18-22]")
# log in the SECTION-C user (now activated) to get an authenticated session
s = requests.Session()
tok = find_csrf(s.get(BASE + "/auth/login").text)
s.post(BASE + "/auth/login",
       data={"csrf_token": tok, "email": remail, "password": "Resend-Passw0rd!"},
       allow_redirects=False)
r = s.get(f"{BASE}/auth/activate?code=anything", allow_redirects=False)
print(f"authenticated session: GET /auth/activate?code=anything -> {r.status_code}")
print(f"'already logged in' in body: {'already logged in' in r.text}")

# ===========================================================================
rule("CREATED USERS (for §8 cleanup accounting)")
print(db(f"select id,email,activated from users where email like '%{EPOCH}@gmail.com' order by id;"))
```

### 4.7 Edge / boundary — complete raw output

Command that produced it:

```bash
python3 /tmp/obs/obj2_edges.py
```

Complete, unedited output:

```text

==============================================================================
SECTION A  M4-07 password boundaries (Length(min=8, max=100) [register.py:L27])
==============================================================================

[len=  7] email=pwlen7_1783968741@gmail.com
  password length sent : 7
  HTTP status          : 200
  page <title>         : Register | SimpleLogin
  length-error in body : True
  user row created     : 0   (1=created, 0=rejected)

[len=  8] email=pwlen8_1783968741@gmail.com
  password length sent : 8
  HTTP status          : 200
  page <title>         : Activation Email Sent | SimpleLogin
  length-error in body : False
  user row created     : 1   (1=created, 0=rejected)

[len=100] email=pwlen100_1783968741@gmail.com
  password length sent : 100
  HTTP status          : 200
  page <title>         : Activation Email Sent | SimpleLogin
  length-error in body : False
  user row created     : 1   (1=created, 0=rejected)

[len=101] email=pwlen101_1783968741@gmail.com
  password length sent : 101
  HTTP status          : 200
  page <title>         : Register | SimpleLogin
  length-error in body : True
  user row created     : 0   (1=created, 0=rejected)

==============================================================================
SECTION B  M4-07 resend activation (GET form + POST) [resend_activation.py:L17-42]
==============================================================================
unactivated user id=12 email=resend_1783968741@gmail.com
activation_code BEFORE resend = ltmllzxzcfyzjfkrijqthwtbqxnojd

GET /auth/resend_activation -> 200; csrf present: True; title: Resend activation email | SimpleLogin

POST /auth/resend_activation -> 200; title: Activation Email Sent | SimpleLogin
flash text(s) in body        : ['An activation email has been sent to you']
activation_code AFTER resend = inizxmthqkrtplqgyliblyxhdihdff
code regenerated (changed)   : True
server log (mail metadata for resend):
  2026-07-13 18:52:26,208 - SL - DEBUG - 12 - "/workspace/app/auth/views/resend_activation.py:36" - resend_activation() -  - user <User 12 resend_1783968741@gmail.com resend_1783968741@gmail.com> is not activated
  2026-07-13 18:52:26,237 - SL - DEBUG - 12 - "/workspace/app/mail_sender.py:131" - send() -  - send email with subject 'Just one more step to join SimpleLogin', from '"noreply@sl.local" <noreply@sl.local>' to 'resend_1783968741@gmail.com'

==============================================================================
SECTION C  edge - wrong password on login [login.py:L45-50]
==============================================================================
(pre-activated user resend_1783968741@gmail.com; activated=t)
POST /auth/login (wrong pw) -> 200; title: Login | SimpleLogin
flash text(s) in body       : ['Email or password incorrect']

==============================================================================
SECTION D  edge - login before activation [login.py:L63-69]
==============================================================================
fresh UNACTIVATED user id=13 email=unactivated_1783968741@gmail.com activated=f
POST /auth/login (not activated) -> 200; title: Login | SimpleLogin
flash text(s) in body            : ['Please check your inbox for the activation email']
resend-activation link offered   : True

==============================================================================
SECTION E  edge - invalid activation code (HTTP 400) [activate.py:L28-36]
==============================================================================
GET /auth/activate?code=THIS-CODE-DOES-NOT-EXIST -> 400
error in body: Activation code cannot be found

==============================================================================
SECTION F  edge - expired activation code (HTTP 400) [activate.py:L38-46]
==============================================================================
fresh user id=14; code=dkoqazhajtucdetbthzeyicbygcmsk
activation_code.expired BEFORE = 2026-07-13 19:52:29.524062
activation_code.expired AFTER  = 2026-07-13 16:52:30.437274   [NON-CANONICAL: aged into the past]
GET /auth/activate?code=dkoqazhajtucdetbthzeyicbygcmsk -> 400
error in body: Activation code was expired
resend-activation offered (show_resend_activation): True

==============================================================================
SECTION G  edge - activate while already authenticated (HTTP 400) [activate.py:L18-22]
==============================================================================
authenticated session: GET /auth/activate?code=anything -> 400
'already logged in' in body: True

==============================================================================
CREATED USERS (for §8 cleanup accounting)
==============================================================================
10|pwlen8_1783968741@gmail.com|f
11|pwlen100_1783968741@gmail.com|f
12|resend_1783968741@gmail.com|t
13|unactivated_1783968741@gmail.com|f
14|expired_1783968741@gmail.com|f
```

### 4.8 M4-07 — password boundaries + resend activation

**Password length boundary.** `RegisterForm.password` is validated by
`validators.Length(min=8, max=100)` [`app/auth/views/register.py:L27`]. The four boundary points
either side of `min` and `max` were exercised (§4.7 SECTION A). On rejection the form re-renders
(title `Register | SimpleLogin`) with the WTForms error `Field must be between 8 and 100 characters
long` and **no** user row is written; on acceptance the waiting-activation page renders
(title `Activation Email Sent | SimpleLogin`) and exactly one user row is created:

| Password length | HTTP | Page title (from §4.7) | Length error shown | User row created | Verdict |
|-----------------|------|------------------------|--------------------|------------------|---------|
| 7 (below min) | 200 | `Register` | **yes** | **0** | rejected |
| 8 (at min) | 200 | `Activation Email Sent` | no | **1** | accepted |
| 100 (at max) | 200 | `Activation Email Sent` | no | **1** | accepted |
| 101 (above max) | 200 | `Register` | **yes** | **0** | rejected |

**Resend activation** — `resend_activation()` [`app/auth/views/resend_activation.py:L17`] (§4.7 SECTION B):

- `GET /auth/resend_activation` → **200**, form served (title `Resend activation email | SimpleLogin`).
- `POST` for an unactivated user → **200**, renders `register_waiting_activation.html`
  [`resend_activation.py:L42`]; flashes **"An activation email has been sent to you. Please check
  your inbox/spam folder."** (category `warning`) [`resend_activation.py:L37-40`]; logs
  `user %s is not activated` [`resend_activation.py:L36`] followed by the same `NOT_SEND_EMAIL`
  metadata line as registration.
- The activation code is **regenerated** (the old and new 30-char codes are both shown in §4.7
  SECTION B) because `send_activation_email()` deletes prior codes before creating a new one
  [`app/auth/views/register.py:L119-120`].

### 4.9 Login / activate edge cases

Each row is a captured HTTP response with its body signal — the first five from §4.7 SECTIONS
C–G, and the three already-authenticated rows from a dedicated live re-verification (complete
unedited capture below the table). Note that **only** `GET /auth/activate` returns `400` for an
already-authenticated session; `register` and `login` short-circuit to a `302` redirect to the
dashboard *before* any form/CSRF processing:

| Case | Entry point | HTTP | Signal | Citation |
|------|-------------|------|--------|----------|
| Wrong password | `POST /auth/login` | 200 | flash "Email or password incorrect"; `LoginEvent.failed` | [`login.py:L45-50`] |
| Login before activation | `POST /auth/login` | 200 | flash "Please check your inbox for the activation email. You can also have this email re-sent"; resend link shown; `LoginEvent.not_activated` | [`login.py:L63-69`] |
| Invalid code | `GET /auth/activate?code=THIS-CODE-DOES-NOT-EXIST` | **400** | "Activation code cannot be found" | [`activate.py:L28-36`] |
| Expired code | `GET /auth/activate?code=<code>` (expiry aged via NON-CANONICAL DB write) | **400** | "Activation code was expired"; resend shown | [`activate.py:L38-46`] |
| Already authenticated — **activate** | `GET /auth/activate?code=anything` (logged-in session) | **400** | "You are already logged in" | [`activate.py:L18-22`] |
| Already authenticated — **register** | `GET` / `POST /auth/register` (logged-in session) | **302** | redirect to `/dashboard/`; warning flash "You are already logged in" | [`register.py:L33-36`] |
| Already authenticated — **login** | `GET` / `POST /auth/login` (logged-in session) | **302** | redirect to `/dashboard/` (no flash) | [`login.py:L28-34`] |

The three already-authenticated rows above were re-verified live against the running `:7777` web
app through the canonical HTTP entry points (seed user `john@wick.com`), confirming that the
`400 "You are already logged in"` response is emitted **only** by `activate` [`activate.py:L18-22`],
whereas `register` [`register.py:L33-36`] and `login` [`login.py:L28-34`] return a `302` redirect to
`/dashboard/` because their `current_user.is_authenticated` guard runs *before* form/CSRF handling.
Complete unedited capture:

```text
STEP 2  POST /auth/login  (john@wick.com / password)       -> HTTP 302 ; Location: http://localhost:7777/dashboard/   [session now authenticated]
STEP 3  GET  /auth/register      (authenticated session)   -> HTTP 302 ; Location: http://localhost:7777/dashboard/
STEP 4  POST /auth/register      (authenticated session)   -> HTTP 302 ; Location: http://localhost:7777/dashboard/   [redirect precedes form processing]
STEP 5  GET  /auth/login         (authenticated session)   -> HTTP 302 ; Location: http://localhost:7777/dashboard/
STEP 6  GET  /auth/activate?code=anything (authenticated)  -> HTTP 400 ; body contains "You are already logged in": YES
STEP 7  follow register 302 to /dashboard/                 -> warning flash "You are already logged in" present: YES
```

Read-only proof that the authenticated `POST /auth/register` (STEP 4) created **no** row — the
guard returns before `RegisterForm` is ever processed, so the demonstration leaves the DB pristine:

```text
$ psql -U myuser -d simplelogin -c "SELECT COUNT(*) FROM users WHERE email='should-not-be-created@example.com';"
 count
-------
     0
$ psql -U myuser -d simplelogin -c "SELECT id,email,activated FROM users ORDER BY id;"
 id |          email          | activated
----+-------------------------+-----------
  1 | john@wick.com           | t
  2 | winston@continental.com | t
```

**Inferred vs observed.** Every row in §4.5, §4.8, and §4.9 is **Observed** — a captured HTTP status
plus a body/flash string, a DB row-state read, and (where applicable) a `sl-web` log line. The only
non-canonical operations are the DB reads used to obtain the activation code (the email channel is
disabled by `NOT_SEND_EMAIL=true`) and the single DB write that ages the `expired` column for the
expired-code case — both explicitly labelled **NON-CANONICAL** at the point of use. The
`LoginEvent`/`RegisterEvent` analytics [`app/events/auth_event.py`] execute on every branch but are a
no-op without a New Relic backend (**inferred**; no dedicated log line is emitted under the default
configuration).

## 5. Objective 3 — Behind-the-Scenes Background Jobs & Internal Services

This objective exercises the three background tiers that support email forwarding and identity
verification, and the shared substrate through which they communicate. Each signal below is captured
from a real entry point: the `job_runner.py` polling loop, the `email_handler.py` aiosmtpd controller
on `:20381`, the `event_listener.py` Postgres listener, and the `cron.py` yacron job dispatcher. All
direct `psql` reads are labelled **NON-CANONICAL** inspection; the triggers and workers are canonical.

**Default-configuration nuances that shape these signals:**

- **`DISABLE_ONBOARDING=true`** — new-user registration does **not** enqueue onboarding jobs, so to
  observe `job_runner.py` acting on real work we trigger the canonical GDPR **"export data"** flow,
  which always enqueues a `send-user-report` job (§5.1).
- **Event dispatch is a default no-op** — with no partner and no webhook, `EventDispatcher.send_event`
  stops at **guard #2** and never creates a `SyncEvent` row nor issues `NOTIFY` (§5.4). This observable
  negative is itself part of the correct answer.

### 5.1 Background job runner — `python job_runner.py` (canonical GDPR export job)

The runner's `__main__` loop wraps each cycle in `create_light_app().app_context()`
[`job_runner.py:L332`], calls `get_jobs_to_run()` [`job_runner.py:L307`], logs
`Take job %s` [`job_runner.py:L334`], moves the job `ready → taken` (`JobState.taken`)
[`job_runner.py:L337-L339`], dispatches through `process_job()` [`job_runner.py:L188`] (the
`send-user-report` branch at [`job_runner.py:L285`] runs `ExportUserDataJob`), moves it to
`done` [`job_runner.py:L344`], and sleeps `time.sleep(10)` between polls [`job_runner.py:L347`].

The job is enqueued canonically from the dashboard: `POST /dashboard/account_setting` with
`form-name=send-full-user-report` [`app/dashboard/views/account_setting.py:L130`] calls
`ExportUserDataJob(current_user).store_job_in_db()` [`account_setting.py:L131`], which inserts a
`Job(name="send-user-report", run_at=now())` [`app/jobs/export_user_data_job.py:L186-190`;
`JOB_SEND_USER_REPORT="send-user-report"` at `app/config.py:L309`].

Driver script (published; temporary, never committed) — `/tmp/obs/obj3_export.py`:

```python
#!/usr/bin/env python3
"""
Objective 3 - canonical background job: the GDPR "export data" flow.

Triggers the real dashboard action `POST /dashboard/account_setting` with
form-name=send-full-user-report [app/dashboard/views/account_setting.py:L130-131], which enqueues a
`Job(name="send-user-report")` via ExportUserDataJob.store_job_in_db()
[app/jobs/export_user_data_job.py:L186-190] with run_at=now(). The running job_runner.py loop then
picks it up: LOG.d("Take job %s") [job_runner.py:L334], state ready(0)->taken(1)
[job_runner.py:L337-L339], process_job() dispatch [job_runner.py:L285], state ->done(2)
[job_runner.py:L344]. Loop sleeps time.sleep(10) between polls [job_runner.py:L347].

Logs in as the canonical seed demo account john@wick.com / password. Polls the Job row every 0.1s to
capture the ready->taken->done transition, and measures the pickup latency twice (M4-08). All psql
reads are NON-CANONICAL inspection; the trigger and the runner are the canonical entry points.
"""
import os, re, time, subprocess, requests

BASE = "http://localhost:7777"
JOHN_ID = 1
STATE = {"0": "ready", "1": "taken", "2": "done", "3": "error", "": "(none)"}


def db(sql):
    env = dict(os.environ, PGPASSWORD="mypassword")
    p = subprocess.run(["psql", "-h", "localhost", "-p", "15432", "-U", "myuser",
                        "-d", "simplelogin", "-tA", "-c", sql],
                       capture_output=True, text=True, env=env)
    return (p.stdout + p.stderr).strip()


def jobrunner_logs(since):
    p = subprocess.run(["docker", "logs", "sl-jobs", "--since", str(since)],
                       capture_output=True, text=True)
    return p.stdout + p.stderr


def find_csrf(html):
    m = re.search(r'name="csrf_token"[^>]*value="([^"]+)"', html)
    return m.group(1) if m else None


def login(email, password):
    s = requests.Session()
    tok = find_csrf(s.get(BASE + "/auth/login").text)
    r = s.post(BASE + "/auth/login",
               data={"csrf_token": tok, "email": email, "password": password},
               allow_redirects=False)
    assert r.status_code == 302, f"login failed: {r.status_code}"
    return s


def snap_job(job_id):
    row = db(f"select state,taken,coalesce(taken_at::text,'NULL'),attempts from job where id={job_id};")
    if not row:
        return None
    st, taken, taken_at, attempts = row.split("|")
    return {"state": st, "taken": taken, "taken_at": taken_at, "attempts": attempts}


def run_once(s, label):
    print("\n" + "-" * 74)
    print(f"{label}: enqueue export job + observe ready->taken->done")
    print("-" * 74)
    # fetch account_setting page for its CSRF token
    tok = find_csrf(s.get(BASE + "/dashboard/account_setting").text)
    since = int(time.time())
    t0 = time.time()
    r = s.post(BASE + "/dashboard/account_setting",
               data={"csrf_token": tok, "form-name": "send-full-user-report"},
               allow_redirects=False)
    print(f"POST /dashboard/account_setting (send-full-user-report) -> {r.status_code}")
    # the job row for john (send-user-report)
    time.sleep(0.2)
    job_id = db(f"select id from job where name='send-user-report' and (payload->>'user_id')='{JOHN_ID}' order by id desc limit 1;")
    print(f"enqueued Job.id = {job_id}  name=send-user-report  (payload user_id={JOHN_ID})")
    init = snap_job(job_id)
    print(f"run_at = {db(f'select run_at from job where id={job_id};')}")
    print(f"T0 (enqueue) state snapshot: state={STATE.get(init['state'])}({init['state']}) "
          f"taken={init['taken']} taken_at={init['taken_at']} attempts={init['attempts']}")

    # poll every 0.1s up to 15s, log each distinct (state,taken) transition
    seen = []
    last = None
    take_ts = None
    deadline = time.time() + 15
    while time.time() < deadline:
        cur = snap_job(job_id)
        key = (cur["state"], cur["taken"])
        if key != last:
            elapsed = time.time() - t0
            seen.append((round(elapsed, 3), cur))
            print(f"  t+{elapsed:6.3f}s  state={STATE.get(cur['state']):5s}({cur['state']}) "
                  f"taken={cur['taken']} taken_at={cur['taken_at']} attempts={cur['attempts']}")
            last = key
        if cur["state"] == "2":   # done
            break
        time.sleep(0.1)

    # pickup latency from the runner's own "Take job" log timestamp
    logs = jobrunner_logs(since)
    take_lines = [l for l in logs.splitlines() if "Take job" in l and f"Job {job_id}" in l]
    if not take_lines:
        take_lines = [l for l in logs.splitlines() if "Take job" in l]
    print("sl-jobs 'Take job' log line(s):")
    for l in take_lines:
        print("  " + l)
    # also show the export/report processing lines
    for l in logs.splitlines():
        if "user report" in l.lower() or "send-user-report" in l or "send email with subject" in l:
            print("  " + l)
    return job_id


s = login("john@wick.com", "password")
print("logged in as john@wick.com (canonical seed demo account)")

j1 = run_once(s, "OBSERVATION 1")
# wait until job 1 fully done + taken before second enqueue (store_job_in_db blocks duplicate untaken)
time.sleep(11)
j2 = run_once(s, "OBSERVATION 2")

print("\n=== FINAL job rows (send-user-report) ===")
print(db("select id,name,state,taken,taken_at,attempts,run_at from job where name='send-user-report' order by id;"))
print("\n(M4-08) exact inter-poll sleep is time.sleep(10) [job_runner.py:L347] (source-derived);")
print("observed pickup latency above corroborates it is bounded by 10s. No non-canonical probe jobs used.")
```

Command that produced it:

```bash
python3 /tmp/obs/obj3_export.py
```

Complete, unedited output:

```text
logged in as john@wick.com (canonical seed demo account)

--------------------------------------------------------------------------
OBSERVATION 1: enqueue export job + observe ready->taken->done
--------------------------------------------------------------------------
POST /dashboard/account_setting (send-full-user-report) -> 200
enqueued Job.id = 1  name=send-user-report  (payload user_id=1)
run_at = 2026-07-13 19:03:26.133211
T0 (enqueue) state snapshot: state=ready(0) taken=f taken_at=NULL attempts=0
  t+ 0.483s  state=ready(0) taken=f taken_at=NULL attempts=0
  t+ 4.266s  state=taken(1) taken=t taken_at=2026-07-13 19:03:30.382064 attempts=1
  t+ 4.397s  state=done (2) taken=t taken_at=2026-07-13 19:03:30.382064 attempts=1
sl-jobs 'Take job' log line(s):
  2026-07-13 19:03:30,381 - SL - DEBUG - 1 - "/workspace/job_runner.py:334" - <module>() -  - Take job <Job 1 send-user-report {'user_id': 1}>
  2026-07-13 19:03:30,381 - SL - DEBUG - 1 - "/workspace/job_runner.py:334" - <module>() -  - Take job <Job 1 send-user-report {'user_id': 1}>
  2026-07-13 19:03:30,437 - SL - DEBUG - 1 - "/workspace/app/mail_sender.py:131" - send() -  - send email with subject 'Your SimpleLogin data', from '"SimpleLogin (noreply)" <noreply@sl.local>' to 'john@wick.com'

--------------------------------------------------------------------------
OBSERVATION 2: enqueue export job + observe ready->taken->done
--------------------------------------------------------------------------
POST /dashboard/account_setting (send-full-user-report) -> 200
enqueued Job.id = 2  name=send-user-report  (payload user_id=1)
run_at = 2026-07-13 19:03:41.659844
T0 (enqueue) state snapshot: state=ready(0) taken=f taken_at=NULL attempts=0
  t+ 0.463s  state=ready(0) taken=f taken_at=NULL attempts=0
  t+ 8.952s  state=done (2) taken=t taken_at=2026-07-13 19:03:50.466875 attempts=1
sl-jobs 'Take job' log line(s):
  2026-07-13 19:03:50,466 - SL - DEBUG - 1 - "/workspace/job_runner.py:334" - <module>() -  - Take job <Job 2 send-user-report {'user_id': 1}>
  2026-07-13 19:03:50,466 - SL - DEBUG - 1 - "/workspace/job_runner.py:334" - <module>() -  - Take job <Job 2 send-user-report {'user_id': 1}>
  2026-07-13 19:03:50,505 - SL - DEBUG - 1 - "/workspace/app/mail_sender.py:131" - send() -  - send email with subject 'Your SimpleLogin data', from '"SimpleLogin (noreply)" <noreply@sl.local>' to 'john@wick.com'

=== FINAL job rows (send-user-report) ===
1|send-user-report|2|t|2026-07-13 19:03:30.382064|1|2026-07-13 19:03:26.133211
2|send-user-report|2|t|2026-07-13 19:03:50.466875|1|2026-07-13 19:03:41.659844

(M4-08) exact inter-poll sleep is time.sleep(10) [job_runner.py:L347] (source-derived);
observed pickup latency above corroborates it is bounded by 10s. No non-canonical probe jobs used.
```

### 5.2 Job state model + poll cadence (M4-08)

The `Job.state` column moves through the `JobState` enum `ready=0 → taken=1 → done=2`
(`error=3` on failure) [`app/models.py:L253-257`]; the `Job` model is defined at
[`app/models.py:L2683`]. Observed transition for the two export jobs above:

| Stage | `state` | `taken` | `taken_at` | `attempts` | Evidence |
|-------|---------|---------|-----------|-----------|----------|
| Enqueued (T0) | `ready` (0) | `f` | NULL | 0 | §5.1 OBS 1/2 "T0 snapshot" |
| Picked up | `taken` (1) | `t` | set | 1 | §5.1 OBS 1 "t+4.266s" + `Take job` log |
| Finished | `done` (2) | `t` | set | 1 | §5.1 "FINAL job rows" |

**Poll cadence.** The exact inter-poll interval is `time.sleep(10)` [`job_runner.py:L347`] — this is
**source-derived**. The observed pickup latency corroborates it: across two canonical enqueues the
runner picked up the job **~4.25 s** and **~8.81 s** after `run_at`, both **bounded by 10 s** (the
value depends on where in the 10-second cycle the enqueue landed). No non-canonical "unknown-job"
probes are used — both jobs are real `send-user-report` jobs dispatched by `process_job()`.

### 5.3 Email forwarding pipeline — `email_handler.py` (SMTP `:20381`)

A message sent to the seed alias `e1@sl.local` (owner `john@wick.com`) is accepted by the aiosmtpd
controller and driven through `MailHandler.handle_DATA()` [`email_handler.py:L2289`] →
`handle()` [`email_handler.py:L1945`] → `handle_forward()` [`email_handler.py:L536`]. Under
`NOT_SEND_EMAIL=true` the final delivery is logged rather than transmitted. The message was injected
with `swaks` through the real SMTP port.

Command (SMTP client):

```bash
swaks --to e1@sl.local --from probe-sender@example.com --server 127.0.0.1:20381 \
      --header "Subject: Obj3 forwarding pipeline probe" \
      --body "Objective 3: exercising handle() -> handle_forward() through the aiosmtpd controller."
```

Complete, unedited `swaks` client transcript (swaks prefixes each line it sends with `->`; only
display-artifact trailing whitespace on otherwise-empty lines has been trimmed so the committed
document passes `git diff --check` — no content byte, timestamp, ID, or field is altered):

```text
=== Trying 127.0.0.1:20381...
=== Connected to 127.0.0.1.
<-  220 reverse-code-generator-71c836b7-xjgzz Python SMTP 1.4.2
 -> EHLO reverse-code-generator-71c836b7-xjgzz
<-  250-reverse-code-generator-71c836b7-xjgzz
<-  250-SIZE 33554432
<-  250-8BITMIME
<-  250-SMTPUTF8
<-  250 HELP
 -> MAIL FROM:<probe-sender@example.com>
<-  250 OK
 -> RCPT TO:<e1@sl.local>
<-  250 OK
 -> DATA
<-  354 End data with <CR><LF>.<CR><LF>
 -> Date: Mon, 13 Jul 2026 19:07:52 +0000
 -> To: e1@sl.local
 -> From: probe-sender@example.com
 -> Subject: Obj3 forwarding pipeline probe
 -> Message-Id: <20260713190752.119109@reverse-code-generator-71c836b7-xjgzz>
 -> X-Mailer: swaks v20240103.0 jetmore.org/john/code/swaks/
 ->
 -> Objective 3: exercising handle() -> handle_forward() through the aiosmtpd controller.
 ->
 ->
 -> .
<-  250 Message accepted for delivery
 -> QUIT
<-  221 Bye
=== Connection closed with remote host.
```

Complete, unedited `sl-email` handler log for that message (`docker logs sl-email --since <send>`):

```text
2026-07-13 19:07:52,111 - SL - DEBUG - 1 - "/workspace/app/log.py:24" - set_message_id() -  - set message_id 0e94bf3b-c88c-4d66-83ea-a9cc8161a842
2026-07-13 19:07:52,111 - SL - DEBUG - 1 - "/workspace/email_handler.py:2342" - _handle() - 0e94bf3b-c88c-4d66-83ea-a9cc8161a842 - ====>=====>====>====>====>====>====>====>
2026-07-13 19:07:52,111 - SL - INFO - 1 - "/workspace/email_handler.py:2343" - _handle() - 0e94bf3b-c88c-4d66-83ea-a9cc8161a842 - New message, mail from probe-sender@example.com, rctp tos ['e1@sl.local']
2026-07-13 19:07:52,112 - SL - INFO - 1 - "/workspace/email_handler.py:1956" - handle() - 0e94bf3b-c88c-4d66-83ea-a9cc8161a842 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 19:07:52,112 - SL - DEBUG - 1 - "/workspace/email_handler.py:1963" - handle() - 0e94bf3b-c88c-4d66-83ea-a9cc8161a842 - Cannot parse Postfix queue ID from None None
2026-07-13 19:07:52,276 - SL - DEBUG - 1 - "/workspace/email_handler.py:1980" - handle() - 0e94bf3b-c88c-4d66-83ea-a9cc8161a842 - ==>> Handle mail_from:probe-sender@example.com, rcpt_tos:['e1@sl.local'], header_from:probe-sender@example.com, header_to:e1@sl.local, cc:None, reply-to:None, message_id:<20260713190752.119109@reverse-code-generator-71c836b7-xjgzz>, client_ip:None, headers:[('Date', 'Mon, 13 Jul 2026 19:07:52 +0000'), ('To', 'e1@sl.local'), ('From', 'probe-sender@example.com'), ('Subject', 'Obj3 forwarding pipeline probe'), ('Message-Id', '<20260713190752.119109@reverse-code-generator-71c836b7-xjgzz>'), ('X-Mailer', 'swaks v20240103.0 jetmore.org/john/code/swaks/'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 19:07:52,282 - SL - DEBUG - 1 - "/workspace/email_handler.py:2202" - handle() - 0e94bf3b-c88c-4d66-83ea-a9cc8161a842 - Forward phase probe-sender@example.com(probe-sender@example.com) -> e1@sl.local
2026-07-13 19:07:52,299 - SL - DEBUG - 1 - "/workspace/email_handler.py:580" - handle_forward() - 0e94bf3b-c88c-4d66-83ea-a9cc8161a842 - Create or get contact for from_header:probe-sender@example.com
2026-07-13 19:07:52,326 - SL - DEBUG - 1 - "/workspace/app/contact_utils.py:110" - create_contact() - 0e94bf3b-c88c-4d66-83ea-a9cc8161a842 - Created contact <Contact 2 probe-sender@example.com 5> for alias <Alias 5 e1@sl.local> with email probe-sender@example.com invalid_email=False
2026-07-13 19:07:52,327 - SL - INFO - 1 - "/workspace/app/handler/dmarc.py:33" - apply_dmarc_policy_for_forward_phase() - 0e94bf3b-c88c-4d66-83ea-a9cc8161a842 - DMARC check disabled
2026-07-13 19:07:52,335 - SL - DEBUG - 1 - "/workspace/email_handler.py:688" - forward_email_to_mailbox() - 0e94bf3b-c88c-4d66-83ea-a9cc8161a842 - Forward <Contact 2 probe-sender@example.com 5> -> <Alias 5 e1@sl.local> -> <Mailbox 1 john@wick.com>
2026-07-13 19:07:52,339 - SL - DEBUG - 1 - "/workspace/email_handler.py:740" - forward_email_to_mailbox() - 0e94bf3b-c88c-4d66-83ea-a9cc8161a842 - Create <EmailLog 2> for <Contact 2 probe-sender@example.com 5>, <User 1 John Wick john@wick.com>, <Mailbox 1 john@wick.com>
2026-07-13 19:07:52,344 - SL - DEBUG - 1 - "/workspace/email_handler.py:867" - forward_email_to_mailbox() - 0e94bf3b-c88c-4d66-83ea-a9cc8161a842 - From header, new:"probe-sender at example.com" <probe-sender_at_example_com_vvceytezmb@sl.local>, old:probe-sender@example.com
2026-07-13 19:07:52,344 - SL - DEBUG - 1 - "/workspace/email_handler.py:316" - replace_header_when_forward() - 0e94bf3b-c88c-4d66-83ea-a9cc8161a842 - Delete Cc header, old value None
2026-07-13 19:07:52,345 - SL - DEBUG - 1 - "/workspace/email_handler.py:313" - replace_header_when_forward() - 0e94bf3b-c88c-4d66-83ea-a9cc8161a842 - Replace To header, old: e1@sl.local, new: e1@sl.local
2026-07-13 19:07:52,345 - SL - INFO - 1 - "/workspace/app/handler/unsubscribe_generator.py:36" - _generate_header_with_original_behaviour() - 0e94bf3b-c88c-4d66-83ea-a9cc8161a842 - Email has no unsubscribe header
2026-07-13 19:07:52,345 - SL - DEBUG - 1 - "/workspace/email_handler.py:893" - forward_email_to_mailbox() - 0e94bf3b-c88c-4d66-83ea-a9cc8161a842 - Forward mail from probe-sender@example.com to john@wick.com, mail_options:[], rcpt_options:[]
2026-07-13 19:07:52,345 - SL - DEBUG - 1 - "/workspace/app/mail_sender.py:131" - send() - 0e94bf3b-c88c-4d66-83ea-a9cc8161a842 - send email with subject 'Obj3 forwarding pipeline probe', from '"probe-sender at example.com" <probe-sender_at_example_com_vvceytezmb@sl.local>' to 'e1@sl.local'
2026-07-13 19:07:52,345 - SL - INFO - 1 - "/workspace/email_handler.py:2367" - _handle() - 0e94bf3b-c88c-4d66-83ea-a9cc8161a842 - Finish mail_from probe-sender@example.com, rcpt_tos ['e1@sl.local'], takes 0.23475217819213867 seconds with return code '250 Message accepted for delivery'<<===
```

**Correct paths (M4-01).** The DMARC check runs from `app/handler/dmarc.py:33`
(`apply_dmarc_policy_for_forward_phase()`, logged `DMARC check disabled` under the local default) —
**not** `app/dmarc.py`. The forward is driven by `forward_email_to_mailbox()`
(defined at [`email_handler.py:L679`]); it logs the route
`Forward <Contact 2 probe-sender@example.com 5> -> <Alias 5 e1@sl.local> -> <Mailbox 1 john@wick.com>`
at [`email_handler.py:L688`], creates the reverse-alias `Contact` via `create_contact()`
[`app/contact_utils.py:L110`], and inserts an `EmailLog` (`EmailLog.create` at
[`email_handler.py:L732`], logged `Create <EmailLog 2>` at [`email_handler.py:L740`]). Observed DB
side effects: `email_log` 1 → 2 and `contact` 1 → 2. The pipeline finished in `_handle()`
[`email_handler.py:L2367`], whose final log line (reproduced verbatim in the block above) ends with
`return code '250 Message accepted for delivery'` and reports the elapsed
`takes 0.23475217819213867 seconds`.

### 5.4 Event system under the default self-host — dispatch stops at guard #2 (M4-12)

Creating an alias calls `EventDispatcher.send_event(user, EventContent(alias_created=event))`
[`app/models.py:L1687`]. `send_event()` [`app/events/event_dispatcher.py:L49`] evaluates three guards
in order: **#1** `EVENT_WEBHOOK_DISABLE` [`event_dispatcher.py:L57`], **#2** `not EVENT_WEBHOOK`
[`event_dispatcher.py:L61-64`], **#3** `not partner_user` [`event_dispatcher.py:L66-69`]; only on
passing all three does it reach the success path — `PostgresDispatcher.send()`, which creates a `SyncEvent` row and
issues `NOTIFY simplelogin_sync_events` [`event_dispatcher.py:L23-26`] and logs
`Sent event to the dispatcher` [`event_dispatcher.py:L84`].

Under the default config (`EVENT_WEBHOOK=None` [`app/config.py:L612`], `EVENT_WEBHOOK_DISABLE=False`
[`app/config.py:L616`]) dispatch **stops at guard #2** — so guards #1/#3 and the entire success path
(the `SyncEvent` row and the `NOTIFY`) are **source-derived / not exercised** under default.

Driver script (published; temporary, never committed) — `/tmp/obs/obj3_event.py`:

```python
#!/usr/bin/env python3
"""
Objective 3 - the event system under the DEFAULT self-host configuration (M4-12).

CANONICAL observation: creating an alias calls
  EventDispatcher.send_event(user, EventContent(alias_created=event))  [app/models.py:L1687]
Under the default config (EVENT_WEBHOOK=None, EVENT_WEBHOOK_DISABLE=False) send_event() reaches
GUARD #2 [app/events/event_dispatcher.py:L61-64] and returns WITHOUT creating a SyncEvent row or
issuing NOTIFY. We trigger a real alias creation via HTTP registration (User.create provisions the
default newsletter alias -> Alias.create -> send_event), capture the fresh guard #2 log line, and
show the sync_event table stays empty (no dispatch).

NON-CANONICAL substrate probe: we then demonstrate the raw Postgres LISTEN/NOTIFY channel
`simplelogin_sync_events` [app/events/event_dispatcher.py:L14] works at the database level - but this
is NOT the app dispatching; it is a manual substrate probe, explicitly labelled non-canonical.
"""
import os, re, time, subprocess, requests

BASE = "http://localhost:7777"
EPOCH = int(time.time())
EMAIL = f"eventobs{EPOCH}@gmail.com"
PASSWORD = "Event-Obs-Passw0rd"


def db(sql):
    env = dict(os.environ, PGPASSWORD="mypassword")
    p = subprocess.run(["psql", "-h", "localhost", "-p", "15432", "-U", "myuser",
                        "-d", "simplelogin", "-tA", "-c", sql],
                       capture_output=True, text=True, env=env)
    return (p.stdout + p.stderr).strip()


def weblogs(since):
    p = subprocess.run(["docker", "logs", "sl-web", "--since", str(since)],
                       capture_output=True, text=True)
    return p.stdout + p.stderr


def find_csrf(html):
    m = re.search(r'name="csrf_token"[^>]*value="([^"]+)"', html)
    return m.group(1) if m else None


print("=" * 74)
print("CANONICAL: alias creation -> EventDispatcher.send_event -> guard #2 (default config)")
print("=" * 74)
print(f"sync_event rows BEFORE = {db('select count(*) from sync_event;')}  (default: no dispatch)")

s = requests.Session()
tok = find_csrf(s.get(BASE + "/auth/register").text)
since = int(time.time())
r = s.post(BASE + "/auth/register",
           data={"csrf_token": tok, "email": EMAIL, "password": PASSWORD},
           allow_redirects=False)
time.sleep(1.0)
uid = db(f"select id from users where email='{EMAIL}';")
newsletter = db(f"select email from alias where user_id={uid};")
print(f"registered temp user id={uid}; User.create provisioned newsletter alias: {newsletter}")
print("\nsl-web log - the send_event GUARD #2 line emitted by the alias_created dispatch:")
for line in weblogs(since).splitlines():
    if "Not sending events" in line or "event_dispatcher" in line:
        print("  " + line)
print(f"\nsync_event rows AFTER  = {db('select count(*) from sync_event;')}  "
      f"(still 0: no SyncEvent created, no NOTIFY issued under default)")

print("\nRuntime config confirming the guard (read inside sl-web):")
cfg = subprocess.run(["docker", "exec", "sl-web", "/app/venv/bin/python", "-c",
                      "from app import config; print('EVENT_WEBHOOK=%r' % config.EVENT_WEBHOOK); "
                      "print('EVENT_WEBHOOK_DISABLE=%r' % config.EVENT_WEBHOOK_DISABLE)"],
                     capture_output=True, text=True)
for line in cfg.stdout.splitlines():
    if line.startswith("EVENT_WEBHOOK"):
        print("  " + line)

print("\n" + "=" * 74)
print("NON-CANONICAL substrate probe: raw Postgres LISTEN/NOTIFY on simplelogin_sync_events")
print("=" * 74)
print("This proves the CHANNEL works at the DB substrate level. It is NOT the app dispatching")
print("(the app stops at guard #2 above). Explicitly labelled NON-CANONICAL.")
probe = r'''
import time, psycopg2
from app.config import DB_URI
CH = "simplelogin_sync_events"
# two separate connections: mirrors the real dispatcher (one conn) / listener (another conn)
lis = psycopg2.connect(DB_URI); lis.set_isolation_level(psycopg2.extensions.ISOLATION_LEVEL_AUTOCOMMIT)
lcur = lis.cursor(); lcur.execute("LISTEN %s;" % CH)
print("listener conn: LISTEN %s registered" % CH)
noti = psycopg2.connect(DB_URI); noti.set_isolation_level(psycopg2.extensions.ISOLATION_LEVEL_AUTOCOMMIT)
ncur = noti.cursor(); ncur.execute("NOTIFY %s, 'non-canonical-substrate-probe';" % CH)
print("notifier conn: NOTIFY %s sent (payload=non-canonical-substrate-probe)" % CH)
got = False
deadline = time.time() + 5
while time.time() < deadline and not got:
    lis.poll()
    while lis.notifies:
        n = lis.notifies.pop(0)
        print("listener conn RECEIVED NOTIFY: pid=%s channel=%s payload=%s" % (n.pid, n.channel, n.payload))
        got = True
    time.sleep(0.2)
print("channel delivered:", got)
lcur.close(); lis.close(); ncur.close(); noti.close()
'''
pr = subprocess.run(["docker", "exec", "-w", "/workspace", "sl-web", "/app/venv/bin/python", "-c", probe],
                    capture_output=True, text=True)
for line in (pr.stdout + pr.stderr).splitlines():
    if any(k in line for k in ("LISTEN", "NOTIFY", "RECEIVED", "channel delivered", "conn:")):
        print("  " + line)
print(f"\ntemp user id={uid} email={EMAIL} (removed by the §8 canonical reset)")
```

Command that produced it:

```bash
python3 /tmp/obs/obj3_event.py
```

Complete, unedited output:

```text
==========================================================================
CANONICAL: alias creation -> EventDispatcher.send_event -> guard #2 (default config)
==========================================================================
sync_event rows BEFORE = 0  (default: no dispatch)
registered temp user id=19; User.create provisioned newsletter alias: simplelogin-newsletter.avails857@sl.local

sl-web log - the send_event GUARD #2 line emitted by the alias_created dispatch:
  2026-07-13 19:06:07,205 - SL - INFO - 12 - "/workspace/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty

sync_event rows AFTER  = 0  (still 0: no SyncEvent created, no NOTIFY issued under default)

Runtime config confirming the guard (read inside sl-web):
  EVENT_WEBHOOK=None
  EVENT_WEBHOOK_DISABLE=False

==========================================================================
NON-CANONICAL substrate probe: raw Postgres LISTEN/NOTIFY on simplelogin_sync_events
==========================================================================
This proves the CHANNEL works at the DB substrate level. It is NOT the app dispatching
(the app stops at guard #2 above). Explicitly labelled NON-CANONICAL.
  listener conn: LISTEN simplelogin_sync_events registered
  notifier conn: NOTIFY simplelogin_sync_events sent (payload=non-canonical-substrate-probe)
  listener conn RECEIVED NOTIFY: pid=1843 channel=simplelogin_sync_events payload=non-canonical-substrate-probe
  channel delivered: True

temp user id=19 email=eventobs1783969566@gmail.com (removed by the §8 canonical reset)
```

The `NOTIFY simplelogin_sync_events` channel [`app/events/event_dispatcher.py:L14`] is the intended
substrate between the web app (dispatcher) and `event_listener.py` (listener). The two-connection
`LISTEN`/`NOTIFY` block above proves the channel works at the **database substrate level**, but it is
explicitly **NON-CANONICAL** — the application itself issues no `NOTIFY` under the default config
(guard #2, `sync_event` count stays 0).

### 5.5 Internal service: `event_listener.py listener` — bounded invocation (M7-01)

`event_listener.py` in `listener` mode is a long-running internal service: `main()`
[`event_listener.py:L29`] selects `PostgresEventSource` [`event_listener.py:L34-35`] and an
`HttpEventSink` [`event_listener.py:L43`], then `__listen()` [`events/event_source.py:L41`] logs
`Starting to listen to events` [`event_source.py:L49`] and enters a `while True` loop
[`event_source.py:L50`] around `select.select([self.__connection], [], [], 5)`
[`event_source.py:L51`] and **blocks forever** (the outer `run()` wraps this in another `while True`
[`event_source.py:L33`]). To capture its complete startup output and a definitive termination, it was run
under a hard wall-clock bound with `timeout`.

Command (bounded run; `timeout -k 5 15` sends `SIGTERM` at 15 s, `SIGKILL` 5 s later):

```bash
docker run --rm --network host -v "$REPO":/workspace -w /workspace \
  -e PATH=/app/venv/bin:/usr/local/bin:/usr/bin:/bin -e VIRTUAL_ENV=/app/venv -e PYTHONUNBUFFERED=1 \
  --entrypoint timeout sl-app:runtime -k 5 15 /app/venv/bin/python event_listener.py listener
echo "LISTENER_EXIT_CODE=$?"
```

Complete, unedited output:

```text
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/plmvkrkijvfcuszqijqu
Upload files to local dir
>>> init logging <<<
2026-07-13 19:07:15,263 - SL - DEBUG - 7 - "/workspace/app/utils.py:17" - <module>() -  - load words file: /workspace/local_data/test_words.txt
2026-07-13 19:07:15,383 - SL - INFO - 7 - "/workspace/event_listener.py:34" - main() -  - Using PostgresEventSource
2026-07-13 19:07:15,387 - SL - INFO - 7 - "/workspace/event_listener.py:43" - main() -  - Starting with HttpEventSink
2026-07-13 19:07:15,387 - SL - INFO - 7 - "/workspace/events/event_source.py:49" - __listen() -  - Starting to listen to events
LISTENER_EXIT_CODE=124
```

The listener reached `Starting to listen to events` [`events/event_source.py:L49`] — the correct path
is `events/event_source.py` (M4-01), **not** `app/events/event_source.py` — and then blocked in the
`select` loop with no further lines until `timeout` fired. `LISTENER_EXIT_CODE=124` is the shell's
canonical "command timed out" status, proving the process was still running (blocked) at the bound
and was terminated by the injected `SIGTERM`. This is the definitive lifecycle evidence the original
document lacked: complete startup banner, the exact final log line, and an explicit exit status.

### 5.6 Internal service: `cron.py` / yacron scheduler

`cron.py` is one of the three canonical entry points named in `CONTRIBUTING.md` (see §6). It is a
one-shot dispatcher selected by `-j/--job` [`cron.py:L1265-1272`], wrapped in
`create_light_app().app_context()`. The yacron daemon reads `crontab.yml` and invokes
`python cron.py -j <job>` on each schedule. A single job was exercised canonically (`stats`) under a
wall-clock bound.

Command (bounded run):

```bash
docker run --rm --network host -v "$REPO":/workspace -w /workspace \
  -e PATH=/app/venv/bin:/usr/local/bin:/usr/bin:/bin -e VIRTUAL_ENV=/app/venv -e PYTHONUNBUFFERED=1 \
  --entrypoint timeout sl-app:runtime 60 /app/venv/bin/python cron.py -j stats
echo "CRON_EXIT_CODE=$?"
```

Complete, unedited output:

```text
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/uboyuutdujuimefmguzy
Upload files to local dir
>>> init logging <<<
2026-07-13 19:08:23,126 - SL - DEBUG - 7 - "/workspace/app/utils.py:17" - <module>() -  - load words file: /workspace/local_data/test_words.txt
2026-07-13 19:08:25,243 - SL - DEBUG - 7 - "/workspace/cron.py:1263" - <module>() -  - Start running cronjob
2026-07-13 19:08:25,245 - SL - DEBUG - 7 - "/workspace/cron.py:1275" - <module>() -  - Compute growth and daily monitoring stats
2026-07-13 19:08:25,245 - SL - WARNING - 7 - "/workspace/cron.py:540" - stats() -  - ADMIN_EMAIL not set, nothing to do
CRON_EXIT_CODE=0
```

The run logged `Start running cronjob` [`cron.py:L1263`], dispatched to the `stats` branch
`Compute growth and daily monitoring stats` [`cron.py:L1275`], and `stats()` short-circuited with
`ADMIN_EMAIL not set, nothing to do` [`cron.py:L540`] — the correct default-config behavior since no
admin recipient is configured locally. `CRON_EXIT_CODE=0` confirms a clean one-shot completion (the
job finished well within the 60 s bound; `timeout` did not fire).

The full yacron schedule — **15 jobs** — is defined in `crontab.yml`; the most frequent is
`send_undelivered_mails` at `*/5 * * * *` (every 5 minutes). Complete, unedited file:

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

`crontab-all-hosts.yml` is the multi-host variant of the same schedule. Under the local default no
yacron daemon is started (jobs are only dispatched on demand via `python cron.py -j <job>` as shown
above), which is why nothing in `crontab.yml` fires automatically during a plain `server.py` boot.

## 6. Inferred vs Observed — evidence classification & canonical entry points

### 6.1 Canonical entry points (M4-11)

`CONTRIBUTING.md` names **exactly three** canonical entry points [`CONTRIBUTING.md:L143-147`]:

```text
The repo consists of the three following entry points:

- wsgi.py and server.py: the webapp.
- email_handler.py: the email handler.
- cron.py: the cronjob.
```

Accordingly, the three canonical entry points are **`wsgi.py`/`server.py`** (the webapp),
**`email_handler.py`** (the email handler), and **`cron.py`** (the cronjob). `job_runner.py` is
**not** one of the three — it is documented separately under its own `## Job runner` heading
[`CONTRIBUTING.md:L226-229`] as an auxiliary handler required for features such as GDPR data export
(exactly the flow exercised in §5.1). `event_listener.py` is not mentioned in `CONTRIBUTING.md` at
all; it is an internal service described here from its own source (`event_listener.py` +
`events/event_source.py`). This document therefore attributes the "three canonical entry points"
strictly to `wsgi.py`/`server.py`, `email_handler.py`, and `cron.py`, and presents `job_runner.py`
and `event_listener.py` as additional internal services, each described from its own source file.

### 6.2 Evidence classification ledger

Every behavioral claim in this document is classified as **Observed** (captured live at runtime),
**Source-derived** (read from the code and *not* exercised at runtime under the default config), or
**Non-canonical** (produced by a deliberate runtime manipulation — a direct DB read/write or a raw
`psql` probe — that bypasses the normal application entry point and is therefore *not* an observation
of the application behaving on its own).

**Observed at runtime (canonical):**

| # | Behavior / signal | Where | Objective |
|---|-------------------|-------|-----------|
| 1 | `>>> init logging <<<` banner + Flask boot + `* Debug mode: on` + dev-server warning | §3.1 | 1 |
| 2 | Live HTTP: `/` → 302, `/auth/login` → 200, `/auth/register` → 200, `/dashboard/` → 302 (unauth) | §3.1 | 1 |
| 3 | Email handler `Listen for port 20381` + `Start mail controller 0.0.0.0 20381` + live SMTP banner | §3.2 | 1 |
| 4 | `job_runner.py` boot into the poll loop | §3.3 | 1 |
| 5 | `alembic upgrade head` + `flask dummy-data` full output + exit 0 + schema head `32f25cbf12f6` | §2.6 | 1 |
| 6 | Register `POST /auth/register` → `create user` + activation-email metadata log + waiting-activation render | §4.2 | 2 |
| 7 | `User.activated` `False` → `True` across `GET /auth/activate` | §4.5 | 2 |
| 8 | `activation_code` row present → deleted → replay `400 Activation code cannot be found` | §4.4 | 2 |
| 9 | `POST /auth/login` → `302` → `/dashboard/` → authenticated dashboard `200` | §4.2 | 2 |
| 10 | Activation-success flash `Your account has been activated` rendered on the dashboard | §4.5 | 2 |
| 11 | Edge paths: wrong password, login-before-activation, invalid code `400`, expired code `400`, already-authenticated activate `400` (register / login redirect `302`) | §4.9 | 2 |
| 12 | Resend `GET` `200` + `POST` `200` flash `An activation email has been sent to you` + code regenerated | §4.8 | 2 |
| 13 | Password boundaries: `7` rejected, `8` accepted, `100` accepted, `101` rejected | §4.8 | 2 |
| 14 | GDPR export `Job.state` `ready` → `taken` → `done` + observed pickup latencies (~4.25 s, ~8.81 s) | §5.1, §5.2 | 3 |
| 15 | Forward pipeline `handle()` → `handle_forward()` → `forward_email_to_mailbox()` → `EmailLog` → `250 Message accepted for delivery` | §5.3 | 3 |
| 16 | Event dispatch stops at guard #2 (`Not sending events because webhook is not configured and allowed to be empty`); `sync_event` count stays `0` | §5.4 | 3 |
| 17 | `event_listener.py listener` startup banner → blocks in `select` loop → `LISTENER_EXIT_CODE=124` | §5.5 | 3 |
| 18 | `cron.py -j stats` run → `ADMIN_EMAIL not set, nothing to do` → `CRON_EXIT_CODE=0` | §5.6 | 3 |

**Source-derived (inferred; NOT exercised at runtime under the default config):**

| # | Behavior | Basis (file:line) | Why not exercised |
|---|----------|-------------------|-------------------|
| 1 | Event **success** path — guards #1 (`EVENT_WEBHOOK_DISABLE`) and #3 (`not partner_user`) passing → `PostgresDispatcher.send()` → `SyncEvent.create` + `NOTIFY simplelogin_sync_events` + `Sent event to the dispatcher` | `app/events/event_dispatcher.py:L57,L66-69,L23-25,L84` | Default config has `EVENT_WEBHOOK=None`, so dispatch stops at guard #2 (§5.4) — the success path is never reached |
| 2 | MFA branches in `after_login()` — FIDO and OTP redirects | `app/auth/views/login_utils.py:L19-27,L28-33` | Test/seed users have neither `fido_enabled()` nor `enable_otp`, so the non-MFA branch is taken |
| 3 | Analytics recording backend — `LoginEvent.send()` / `RegisterEvent.send()` call `newrelic.agent.record_custom_event` | `app/events/auth_event.py:L21-24` | No New Relic agent is configured locally, so the call records nowhere observable (functional no-op) |
| 4 | Onboarding jobs at registration | `app/models.py:L647-664` | `DISABLE_ONBOARDING=true` — `User.create()` logs `Disable onboarding emails` and skips scheduling |
| 5 | `cron.py` jobs other than `stats` (e.g. `send_undelivered_mails`, `delete_logs`) | `crontab.yml` | Only the `stats` job was driven canonically; the others are schedule-defined but not individually run |
| 6 | Email handler **reply** and **bounce** phases | `email_handler.py:L966` (`handle_reply`), `email_handler.py:L1851` (`handle_bounce`) | Only the forward phase was driven via `swaks` (§5.3) |

**Non-canonical (deliberate runtime manipulation — labeled, NOT an observation of app behavior):**

| # | Manipulation | Where | Why used |
|---|--------------|-------|----------|
| 1 | Direct `activation_code` table **read** to obtain the code | §4.3 | Under `NOT_SEND_EMAIL=true` the code/link is never logged, so it is read from the DB purely to continue the canonical `GET /auth/activate` flow |
| 2 | Direct `activation_code` **write** to age `expired` into the past | §4.9 | To reach the `Activation code was expired` `400` branch without waiting one hour |
| 3 | Raw two-connection `psql` `LISTEN`/`NOTIFY` **substrate probe** | §5.4 | Proves the Postgres channel works at the DB level; the application itself issues no `NOTIFY` under the default (guard #2) |

## 7. Coverage matrix — every named item, with evidence status

Each of the three questions is decomposed into every named item (mechanism, function, condition,
file, flag, port, log line). The **Evidence status** column classifies each row as **Observed**,
**Observed (negative)** (an observed *absence*, itself the correct default answer), **Source-derived**
(read from code, not exercised under the default), or **Non-canonical** (produced by a deliberate
runtime manipulation). There is **no blanket "all observed" claim** — each row is classified on its
own.

### 7.1 Objective 1 — startup readiness signals

| Named item (file:line) | § | Evidence status | Concrete value / signal |
|------------------------|---|-----------------|-------------------------|
| `>>> init logging <<<` banner [`app/log.py:L67`] | §3.1 | Observed | Printed at import on every entry point boot |
| Flask dev-server bind on `:7777` [`server.py:L588`] | §3.1 | Observed | `app.run(debug=True, port=7777)`; `* Debug mode: on` + dev-server warning |
| Werkzeug `* Running on http://127.0.0.1:7777` line | §3.1 | Observed (negative) | **Suppressed** — the `werkzeug` logger is disabled [`app/log.py:L70-71`]; readiness is confirmed instead by the live HTTP probes |
| PostgreSQL connectivity via migrated schema | §2.6, §3.1 | Observed | `alembic upgrade head` → head `32f25cbf12f6`; 77 public tables; live `SELECT` probes |
| Redis for sessions / rate-limiting [`server.py:L163-165`] | §3.1 | Observed (negative) / Source-derived | `MEM_STORE_URI=None` [`app/config.py:L568`] under default → web app uses **in-memory** limiter + signed-cookie sessions; the Redis-backed path is source-derived (not wired by default) |
| aiosmtpd controller on `:20381` [`email_handler.py:L2386,L2403`] | §3.2 | Observed | `Listen for port 20381` + `Start mail controller 0.0.0.0 20381` + live SMTP banner |
| Blueprint registration (`auth`, `dashboard`) [`server.py:L233-246`] | §3.1 | Observed | `/auth/login` → 200, `/auth/register` → 200, `/dashboard/` → 302 (login-required redirect) |
| One-time data init call graph | §3.4 | Observed + Source-derived | `flask dummy-data` output observed (§2.6); the exact `fake_data`/`add_sl_domains`/`add_proton_partner` vs `init_app.__main__` call graph is source-derived (M4-10 correction) |

### 7.2 Objective 2 — new-user walkthrough (register → verify → login → dashboard)

| Named item (file:line) | § | Evidence status | Concrete value / signal |
|------------------------|---|-----------------|-------------------------|
| `POST /auth/register` [`register.py:L32`] | §4.2 | Observed | HTTP 200 `Activation Email Sent`; log `create user` [`register.py:L85`] |
| `ActivationCode` generation [`register.py:L120`] | §4.3 | Observed + Non-canonical | Row created (observed via DB); the 30-char code read from DB is **Non-canonical** (not logged under `NOT_SEND_EMAIL`) |
| Activation email under `NOT_SEND_EMAIL=true` [`app/mail_sender.py:L131`] | §4.2, §4.3 | Observed (negative) | Only **metadata** logged (`send email with subject 'Just one more step to join SimpleLogin'`); body/link **not** logged/sent |
| `GET /auth/activate` → `User.activated` `True` [`activate.py:L49`] | §4.4, §4.5 | Observed | `activated` `False` → `True`; HTTP 302 → `/dashboard/` |
| Activation-code consumption [`activate.py:L53`] | §4.4 | Observed | Row present → deleted; replay → `400 Activation code cannot be found` |
| `POST /auth/login` [`login.py:L25`] | §4.2, §4.5 | Observed | HTTP 302 → `/dashboard/` |
| `after_login()` 302 into `dashboard.index` [`login_utils.py:L40-45`] | §4.5 | Observed | `Location: /dashboard/`; dashboard `200` `Alias \| SimpleLogin` |
| MFA branches (FIDO / OTP) [`login_utils.py:L19-33`] | §6.2 | Source-derived | Seed/test users have no MFA → non-MFA branch taken |
| Flash messages | §4.5 | Observed | `Your account has been activated` (success) rendered on dashboard |
| Redirect `Location` headers | §4.2, §4.5 | Observed | `302` with `Location: /dashboard/` captured verbatim |
| Edge: wrong password [`login.py:L45-50`] | §4.9 | Observed | Flash `Email or password incorrect` |
| Edge: login-before-activation [`login.py:L63-69`] | §4.9 | Observed | Flash `Please check your inbox for the activation email` |
| Edge: invalid activation code [`activate.py:L28-36`] | §4.9 | Observed | HTTP `400 Activation code cannot be found` |
| Edge: expired activation code [`activate.py:L38-46`] | §4.9 | Observed + Non-canonical | HTTP `400 Activation code was expired` (expiry aged via **Non-canonical** DB write) |
| Edge: already-authenticated activate [`activate.py:L18-22`] | §4.9 | Observed | HTTP `400` `You are already logged in` |
| Edge: already-authenticated register / login [`register.py:L33-36`, `login.py:L28-34`] | §4.9 | Observed | HTTP `302` → `/dashboard/` (register adds warning flash `You are already logged in`; login has no flash) |
| Resend activation `GET` + `POST` [`resend_activation.py`] | §4.8 | Observed | `GET` 200 `Resend activation email`; `POST` 200 flash `An activation email has been sent to you`; code regenerated |
| Password boundaries `Length(min=8,max=100)` [`register.py:L27`] | §4.8 | Observed | `7` rejected, `8` accepted, `100` accepted, `101` rejected |

### 7.3 Objective 3 — behind-the-scenes background jobs & internal services

| Named item (file:line) | § | Evidence status | Concrete value / signal |
|------------------------|---|-----------------|-------------------------|
| `job_runner.py` polling loop [`job_runner.py:L329-347`] | §5.1 | Observed | Boots into loop; `Take job` [`L334`] |
| `Job`-table dispatch via `process_job()` [`job_runner.py:L188`] | §5.1 | Observed | GDPR export → `send-user-report` branch [`L285`] |
| `JobState` `ready`→`taken`→`done` [`app/models.py:L253-257`] | §5.1, §5.2 | Observed | Job `id=1`: `ready(0)` → `taken(1)` → `done(2)` captured mid-flight |
| Poll cadence `time.sleep(10)` [`job_runner.py:L347`] | §5.2 | Source-derived (corroborated) | Exact 10 s is source-derived; two observed pickups (~4.25 s, ~8.81 s) are ≤ 10 s and corroborate it |
| GDPR export job [`app/jobs/export_user_data_job.py:L186-190`] | §5.1 | Observed | `POST /dashboard/account_setting` → `Job.create(name="send-user-report")` → export ran (`send email with subject 'Your SimpleLogin data'`) |
| Onboarding jobs at registration [`app/models.py:L647-664`] | §6.2 | Source-derived | Suppressed by `DISABLE_ONBOARDING=true` (`Disable onboarding emails`) |
| `email_handler.py` forwarding pipeline [`email_handler.py:L1945,L536`] | §5.3 | Observed | `swaks` → `handle_forward` → `forward_email_to_mailbox` → `EmailLog` → `250 Message accepted for delivery` |
| `handle_reply` / `handle_bounce` phases [`email_handler.py:L966,L1851`] | §6.2 | Source-derived | Only the forward phase was driven |
| `NOTIFY simplelogin_sync_events` channel [`app/events/event_dispatcher.py:L14`] | §5.4 | Source-derived + Non-canonical | App issues **no** `NOTIFY` under default (guard #2); channel proven only via a **Non-canonical** `psql` probe |
| Event dispatch default no-op [`app/events/event_dispatcher.py:L61-64`] | §5.4 | Observed (negative) | `Not sending events because webhook is not configured and allowed to be empty`; `sync_event` = 0 |
| Event **success** path (`SyncEvent` + `NOTIFY` + `Sent event to the dispatcher`) [`event_dispatcher.py:L23-26,L84`] | §6.2 | Source-derived | Never reached (default stops at guard #2) |
| `event_listener.py listener` [`event_listener.py:L29,L34-43`] | §5.5 | Observed | `Using PostgresEventSource` → `Starting to listen to events` [`events/event_source.py:L49`] → blocks → exit `124` |
| `cron.py` / yacron scheduler [`cron.py:L1263`, `crontab.yml`] | §5.6 | Observed + Source-derived | `cron.py -j stats` observed (exit 0); the 15-job schedule + other jobs are source-derived from `crontab.yml` |
| `send_undelivered_mails` every 5 min (`*/5 * * * *`) | §5.6 | Source-derived | Most frequent schedule entry in `crontab.yml`; not individually run |
| DB + Redis as inter-process substrate | §5.1-5.4 | Observed + Source-derived | Shared Postgres tables (`Job`, `SyncEvent`) observed; Redis substrate is source-derived (not wired by default, §7.1) |

## 8. Cleanup & read-only guarantee (M4-13)

Every temporary entity created during this investigation (test users id 3–19, their newsletter
aliases/mailboxes/activation codes, the two GDPR-export `Job` rows, and the `swaks` forward's
`Contact`/`EmailLog`/`user_audit_log` rows) is removed by performing the **canonical full DB reset**
— the same `alembic upgrade head` + `flask dummy-data` flow used to build the environment (§2.6),
preceded by a `DROP SCHEMA public CASCADE`. This is a bulletproof restore: it guarantees the database
returns to the exact pristine baseline recorded in §2.7 rather than relying on row-by-row deletes of
irreversible sequences. Before/after SQL is published below.

### 8.1 Before — accumulated test residue (complete SQL)

Command:

```bash
export PGPASSWORD=mypassword
psql -h localhost -p 15432 -U myuser -d simplelogin -tA -c "SELECT 'users', count(*) FROM users
  UNION ALL SELECT 'alias', count(*) FROM alias
  UNION ALL SELECT 'mailbox', count(*) FROM mailbox
  UNION ALL SELECT 'contact', count(*) FROM contact
  UNION ALL SELECT 'email_log', count(*) FROM email_log
  UNION ALL SELECT 'job', count(*) FROM job
  UNION ALL SELECT 'sync_event', count(*) FROM sync_event
  UNION ALL SELECT 'activation_code', count(*) FROM activation_code
  UNION ALL SELECT 'alias_audit_log', count(*) FROM alias_audit_log
  UNION ALL SELECT 'user_audit_log', count(*) FROM user_audit_log
  UNION ALL SELECT 'daily_metric', count(*) FROM daily_metric ORDER BY 1;"
```

Complete, unedited output:

```text
-- BEFORE cleanup (2026-07-13T19:24:28Z)
activation_code|11
alias|28
alias_audit_log|28
contact|2
daily_metric|1
email_log|2
job|2
mailbox|21
sync_event|0
user_audit_log|1
users|19
-- daily_metric detail:
1|2026-07-13|17|28
```

The residue (19 users vs the baseline 2, 28 aliases vs 11, 2 jobs, 11 activation codes, a
`daily_metric` of `17/28` vs `0/11`, etc.) confirms the environment was genuinely dirty — matching
what M4-13 flagged. `sync_event` stayed `0` throughout, corroborating the default event no-op (§5.4).

### 8.2 Canonical reset — stop app containers, drop schema, migrate, seed

Step 1 — stop the three application containers (they hold DB connections; `sl-postgres`/`sl-redis`
stay up):

```bash
docker stop sl-web sl-email sl-jobs
```

```text
sl-web
sl-email
sl-jobs
```

Step 2 — terminate lingering backends, then drop and recreate the `public` schema:

```bash
export PGPASSWORD=mypassword
psql -h localhost -p 15432 -U myuser -d simplelogin -tA \
  -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname='simplelogin' AND pid <> pg_backend_pid();"
psql -h localhost -p 15432 -U myuser -d simplelogin -tA \
  -c "DROP SCHEMA public CASCADE; CREATE SCHEMA public; GRANT ALL ON SCHEMA public TO myuser; GRANT ALL ON SCHEMA public TO public;"
```

Complete, unedited output:

```text
NOTICE:  drop cascades to 82 other objects
DETAIL:  drop cascades to table alembic_version
drop cascades to type plan_enum
drop cascades to table file
drop cascades to table users
drop cascades to table activation_code
drop cascades to table client
drop cascades to table alias
drop cascades to table authorization_code
drop cascades to table client_user
drop cascades to table oauth_token
drop cascades to table redirect_uri
drop cascades to table reset_password_code
drop cascades to table contact
drop cascades to type planenum2
drop cascades to table subscription
drop cascades to table email_log
drop cascades to table deleted_alias
drop cascades to table email_change
drop cascades to table api_key
drop cascades to table alias_used_on
drop cascades to table custom_domain
drop cascades to table lifetime_coupon
drop cascades to table directory
drop cascades to table job
drop cascades to table mailbox
drop cascades to table manual_subscription
drop cascades to table social_auth
drop cascades to table account_activation
drop cascades to table refused_email
drop cascades to table referral
drop cascades to type planenum_apple
drop cascades to table apple_subscription
drop cascades to table sent_alert
drop cascades to table alias_mailbox
drop cascades to table recovery_code
drop cascades to table domain_deleted_alias
drop cascades to table notification
drop cascades to table fido
drop cascades to table mfa_browser
drop cascades to table directory_mailbox
drop cascades to table public_domain
drop cascades to table domain_mailbox
drop cascades to table monitoring
drop cascades to table batch_import
drop cascades to table authorized_address
drop cascades to table coinbase_subscription
drop cascades to table bounce
drop cascades to table transactional_email
drop cascades to table metric2
drop cascades to table payout
drop cascades to table hibp
drop cascades to table alias_hibp
drop cascades to table ignored_email
drop cascades to table coupon
drop cascades to table hibp_notified_alias
drop cascades to table ignore_bounce_sender
drop cascades to extension pg_trgm
drop cascades to table auto_create_rule
drop cascades to table auto_create_rule__mailbox
drop cascades to table message_id_matching
drop cascades to table deleted_directory
drop cascades to table deleted_subdomain
drop cascades to table phone_country
drop cascades to table phone_number
drop cascades to table phone_message
drop cascades to table phone_reservation
drop cascades to table invalid_mailbox_domain
drop cascades to type block_behaviour_enum
drop cascades to table admin_audit_log
drop cascades to table provider_complaint
drop cascades to table partner
drop cascades to table partner_api_token
drop cascades to table partner_user
drop cascades to table partner_subscription
drop cascades to table newsletter
drop cascades to table newsletter_user
drop cascades to table api_cookie_token
drop cascades to table daily_metric
drop cascades to table sync_event
drop cascades to table mailbox_activation
drop cascades to table alias_audit_log
drop cascades to table user_audit_log
DROP SCHEMA
CREATE SCHEMA
GRANT
GRANT
```

Step 3 — re-run the canonical migrate + seed (`ALEMBIC_EXIT=0`, `SEED_EXIT=0`). This is the identical
flow whose complete transcript is embedded verbatim in §2.6; the full migration log is reproduced
again below for this second (cleanup) run:

```bash
REPO=/tmp/blitzy/app/blitzy-d7c2d64b-4eea-4bae-9c05-a7e6d6e3887f_243015
COMMON="--rm --network host -v $REPO:/workspace -w /workspace \
  -e PATH=/app/venv/bin:/usr/local/bin:/usr/bin:/bin -e VIRTUAL_ENV=/app/venv -e PYTHONUNBUFFERED=1"
docker run $COMMON --entrypoint /app/venv/bin/alembic sl-app:runtime upgrade head
docker run $COMMON -e FLASK_APP=wsgi:app --entrypoint /app/venv/bin/flask sl-app:runtime dummy-data
```

Complete, unedited `alembic upgrade head` output:

```text
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/ohxvtrhuvxgdfdqgmsgt
Upload files to local dir
>>> init logging <<<
2026-07-13 19:24:56,346 - SL - DEBUG - 1 - "/workspace/app/utils.py:17" - <module>() -  - load words file: /workspace/local_data/test_words.txt
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

Complete, unedited `flask dummy-data` output:

```text
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/uvyycwuezfmfqefykqen
Upload files to local dir
>>> init logging <<<
2026-07-13 19:24:58,802 - SL - DEBUG - 1 - "/workspace/app/utils.py:17" - <module>() -  - load words file: /workspace/local_data/test_words.txt
2026-07-13 19:25:00,747 - SL - WARNING - 1 - "/workspace/server.py:494" - dummy_data() -  - reset db, add fake data
2026-07-13 19:25:00,747 - SL - DEBUG - 1 - "/workspace/app/fake_data.py:41" - fake_data() -  - create fake data
2026-07-13 19:25:01,134 - SL - INFO - 1 - "/workspace/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 19:25:01,138 - SL - DEBUG - 1 - "/workspace/app/models.py:647" - create() -  - Disable onboarding emails
2026-07-13 19:25:01,158 - SL - DEBUG - 1 - "/workspace/app/models.py:1459" - generate_random_alias_email() -  - generate email tautly_fungal677@sl.local
2026-07-13 19:25:01,169 - SL - INFO - 1 - "/workspace/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 19:25:01,240 - SL - INFO - 1 - "/workspace/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 19:25:01,249 - SL - INFO - 1 - "/workspace/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 19:25:01,268 - SL - INFO - 1 - "/workspace/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 19:25:01,280 - SL - INFO - 1 - "/workspace/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 19:25:01,295 - SL - INFO - 1 - "/workspace/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 19:25:01,303 - SL - INFO - 1 - "/workspace/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 19:25:01,314 - SL - DEBUG - 1 - "/workspace/app/models.py:1255" - generate_oauth_client_id() -  - generate oauth_client_id demo-hzgudzjrik
2026-07-13 19:25:01,322 - SL - DEBUG - 1 - "/workspace/app/models.py:1255" - generate_oauth_client_id() -  - generate oauth_client_id demo2-izabrlzjyj
2026-07-13 19:25:01,595 - SL - INFO - 1 - "/workspace/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 19:25:01,597 - SL - DEBUG - 1 - "/workspace/app/models.py:647" - create() -  - Disable onboarding emails
2026-07-13 19:25:01,620 - SL - INFO - 1 - "/workspace/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 19:25:01,632 - SL - INFO - 1 - "/workspace/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 19:25:01,642 - SL - INFO - 1 - "/workspace/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain
```

### 8.3 After — pristine baseline restored (complete SQL)

The same count query as §8.1 was re-run:

```text
-- AFTER cleanup (2026-07-13T19:25:14Z) — schema head + counts
32f25cbf12f6
activation_code|0
alias|11
alias_audit_log|11
contact|1
daily_metric|1
email_log|1
job|0
mailbox|4
sync_event|0
user_audit_log|0
users|2
-- users list:
1|john@wick.com|t
2|winston@continental.com|t
-- daily_metric detail:
1|2026-07-13|0|11
```

**This matches the §2.7 pristine baseline exactly** — `users=2` (only `john@wick.com` and
`winston@continental.com`), `alias=11`, `mailbox=4`, `contact=1`, `email_log=1`, `job=0`,
`sync_event=0`, `activation_code=0`, `alias_audit_log=11`, `user_audit_log=0`, `daily_metric=1` with
row `1|2026-07-13|0|11`, at alembic head `32f25cbf12f6`. All temporary rows are gone.

### 8.4 Read-only guarantee — git proof

The application containers were restarted so the environment is left live:

```bash
docker start sl-web sl-email sl-jobs
docker ps --format '{{.Names}}\t{{.Status}}' | grep sl- | sort
```

```text
sl-web
sl-email
sl-jobs
sl-email	Up About a minute
sl-jobs	Up About a minute
sl-postgres	Up 3 hours
sl-redis	Up 3 hours
sl-web	Up About a minute
```

The source tree is unchanged apart from this single documentation file:

```bash
git status --porcelain
git diff --stat
git ls-files --others --exclude-standard   # untracked, excluding .gitignore
```

```text
--- git status --porcelain ---
 M blitzy/documentation/app_2cd6ee777f8c.md
--- git diff --stat ---
 blitzy/documentation/app_2cd6ee777f8c.md | 3660 +++++++++++++++++++++++-------
 1 file changed, 2857 insertions(+), 803 deletions(-)
--- untracked (excluding ignored) ---
(none)
```

Exactly one tracked file is modified — `blitzy/documentation/app_2cd6ee777f8c.md` — and there are no
untracked files in the repository. (The `git diff --stat` insertion/deletion counts above are a
point-in-time snapshot captured while this section was being written; the file grew slightly as §8
was appended. The dispositive, stable fact is the `git status --porcelain` line — a single modified
file.) No SimpleLogin source, test, configuration, `pyproject.toml`, or `poetry.lock` file was
changed. The `.env` used for the run is git-ignored (§2.4). All temporary
observation scripts live under `/tmp/obs` (outside the repository) and are removed at the end of the
investigation. The read-only requirement of rule `SWE-AtlasQnA-Repo` is satisfied.
