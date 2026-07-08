# SimpleLogin Runtime Investigation — Evidence-Backed Answers

This document answers three runtime-behavior questions about the SimpleLogin codebase by
**running the real code paths first** and capturing their actual, unedited output. Every factual
claim is paired with a `file:line` citation **and** the observed runtime evidence that produced it.
Statements derived only from reading code (not observed at runtime) are marked `[inferred]`; any
value obtained from a non-canonical accommodation is marked `[NON-CANONICAL]`.

| | |
|---|---|
| **Repository** | SimpleLogin (`app`) — Flask monolith, MIT licensed |
| **Branch** | `app_2cd6ee777f8c` |
| **Source baseline commit** (all `file:line` citations pinned here) | `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c` |
| **Canonical interpreter** | Python 3.10 (`pyproject.toml:61` `python = "^3.10"`; `Dockerfile:8` `FROM python:3.10`) |
| **Services required** | PostgreSQL + Redis |

The three questions target three distinct process entry points of the monolith: the **web
application** (Q1, `GET /dashboard/mailbox_verify`), the standalone **job runner** (Q2,
`job_runner.py`), and the **inbound SMTP daemon** (Q3, `email_handler.py` bounce handling).

## Methodology

1. **Run first.** The canonical runtime (Python 3.10 + PostgreSQL + Redis + the locked
   dependencies) was built, the schema migrated with Alembic, and each question's real code path
   was exercised with temporary observation scripts and the project's own `pytest` harnesses.
2. **Evidence next.** Counter values, job-state transitions, the exact VERP address strings, HMAC
   round-trips, SMTP status codes, and resulting database rows were captured **before, during, and
   after** every state change.
3. **Prose last.** The findings below are written from what was observed. Each question section
   leads with the **direct answer**, then the **evidence**, then the **causal reasoning**.
4. **Read-only.** No repository source file was modified. The only net-new artifact is this
   document. All temporary observation scripts were removed after capture (see the closing section
   for the final `git status` proof).

### Non-canonical accommodations (labeled throughout)

This investigation ran inside the project's canonical Docker image toolchain. Two environment
substitutions were unavoidable and are explicitly labeled wherever they affect a result:

- **`google-re2` substitutes the canonical `pyre2`.** SimpleLogin imports `import re2 as re`
  (`app/email_utils.py:23`, `app/spamassassin_utils.py:8`). The `pyre2` wheel fails to build against
  the sandbox's newer `libre2`, so `google-re2` (which also exposes a module named `re2`) is used
  instead. `google-re2` lacks the `re`-compatible flag constants (e.g. `re2.DOTALL`), which only
  affects Q3's import of `email_handler` — addressed with a labeled shim in the disposable Q3 script
  (details in the Q3 section). The bounce/VERP logic itself runs real and unmodified.
- **PostgreSQL 17 / Redis 8.** The canonical stack pins only the Python client libraries
  (SQLAlchemy `1.3.24`, `redis` client `4.6.0`); the server versions available in the sandbox are
  PostgreSQL 17.10 and Redis 8.0.2, which serve the code paths without incident. The test database
  listens on port **15432** to match `tests/test.env:17`.

## Environment build (exact commands + output)

The canonical interpreter, services, and locked dependencies were established as follows. Package
versions are taken from `poetry.lock`; the client-library versions verified at runtime are shown
below.

```bash
$ python --version
Python 3.10.20
$ python -c "import sys; print(sys.executable)"
/tmp/slvenv/bin/python
```

```bash
$ psql "postgresql://test:test@localhost:15432/test" -tAc "select version();"
PostgreSQL 17.10 (Ubuntu 17.10-0ubuntu0.25.10.1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0, 64-bit
$ pg_isready -h localhost -p 15432
localhost:15432 - accepting connections
$ redis-cli ping
PONG
$ redis-server --version
Redis server v=8.0.2 sha=00000000:0 malloc=jemalloc-5.3.0 bits=64 build=d14bb9989612b22e
```

Mandatory configuration comes from `tests/test.env` (the project's own CI config, loaded by
`app/config.py`). Notably `DB_URI=postgresql://test:test@localhost:15432/test`
(`tests/test.env:17`), `MEM_STORE_URI=redis://localhost` (`tests/test.env:78`), and
`FLASK_SECRET=secret` (`tests/test.env:20`). `VERP_EMAIL_SECRET` is not set explicitly, so
`app/config.py:502-504` derives it from `FLASK_SECRET` at import time; the derived value is 36
characters long and passes the ≥32-char guard at `app/config.py:505-508`. To honor the safety
constraint, the raw derived value is **not** reproduced anywhere in this document — only its length
and a one-way SHA-256 digest are shown. The digest is sufficient to reproduce and verify the exact
VERP HMAC signatures observed in Question 3 without disclosing the secret:

```bash
$ CONFIG=tests/test.env python -c "import hashlib; from app import config; s=config.VERP_EMAIL_SECRET; print('VERP_EMAIL_SECRET length =', len(s), 'chars'); print('sha256(VERP_EMAIL_SECRET) =', hashlib.sha256(s.encode()).hexdigest()); print('passes >=32-char guard:', len(s) >= 32)"
VERP_EMAIL_SECRET length = 36 chars
sha256(VERP_EMAIL_SECRET) = 262d39ac671bae1918c41de1363470149f60cf319c50255ad637a5878de2d216
passes >=32-char guard: True
```

(The distributed template `example.env:75` ships
`DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin`.)

> **Note on credentials:** the values shown in this document — `FLASK_SECRET=secret` and the
> `test:test` database password — are non-sensitive placeholder values that already ship publicly in
> the repository's own `tests/test.env` (SimpleLogin is MIT-licensed open source); neither is a real
> credential. The derived `VERP_EMAIL_SECRET` is deliberately **redacted**: only its length (36) and
> the one-way SHA-256 digest above appear anywhere in this document — never the raw value.

The ~90 locked dependencies are **pre-installed in the provided canonical image's** virtual
environment (`/tmp/slvenv`, built with `uv`, which therefore exposes no in-venv `pip`). Their
presence and exact locked versions were confirmed via `importlib.metadata` — no install/upgrade was
performed. There are 180 installed distributions in total; the packages material to the three
questions match `poetry.lock` exactly:

```bash
$ python -c "import importlib.metadata as m; names=['Flask','Flask-Login','Flask-Limiter','SQLAlchemy','psycopg2-binary','redis','arrow','aiosmtpd','boto3','yacron','Flask-Migrate','gunicorn','gevent','pytest','alembic']; [print(f'{n}=={m.version(n)}') for n in names]; print('TOTAL distributions installed:', len(list(m.distributions())))"
Flask==1.1.2
Flask-Login==0.5.0
Flask-Limiter==1.4
SQLAlchemy==1.3.24
psycopg2-binary==2.9.3
redis==4.6.0
arrow==0.16.0
aiosmtpd==1.4.2
boto3==1.35.37
yacron==0.11.2
Flask-Migrate==2.5.3
gunicorn==20.0.4
gevent==22.10.2
pytest==7.3.1
alembic==1.4.3
TOTAL distributions installed: 180
```

As a second, independent cross-check, the versions actually **loaded at runtime** (by importing the
modules) match the installed locked set above:

```bash
$ python -c "import flask,sqlalchemy,arrow,aiosmtpd,boto3,redis; print('flask',flask.__version__,'| sqlalchemy',sqlalchemy.__version__,'| arrow',arrow.__version__,'| aiosmtpd',aiosmtpd.__version__,'| boto3',boto3.__version__,'| redis',redis.__version__)"
flask 1.1.2 | sqlalchemy 1.3.24 | arrow 0.16.0 | aiosmtpd 1.4.2 | boto3 1.35.37 | redis 4.6.0
```

The schema was migrated with Alembic so every model is queryable. `alembic upgrade head` is
idempotent here (the image's database was already provisioned to head), so it re-affirms the head
revision without applying new steps; `alembic current` then confirms the head revision id. The
leading lines are the SimpleLogin app-startup banner emitted by `alembic/env.py` importing the app
(the ephemeral `GNUPGHOME` temp path, PID, and timestamp vary per invocation):

```bash
$ CONFIG=tests/test.env alembic upgrade head
load config file /tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/ybvofniosmszsbyfiein
Upload files to local dir
>>> init logging <<<
2026-07-08 06:41:10,443 - SL - DEBUG - 110382 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/utils.py:17" - <module>() -  - load words file: /tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/local_data/test_words.txt
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
$ CONFIG=tests/test.env alembic current
32f25cbf12f6 (head)
```

Git baseline. All `file:line` citations in this document are pinned to the **immutable source
baseline commit** `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c` — the commit this working branch was
created from (per the project rule, the source tree is read-only). Before any observation the
working tree was clean; the closing section re-verifies a clean tree after temp-script cleanup. The
read-only guarantee is proven by diffing the current branch head against that source baseline —
exactly one file is added and nothing else is touched:

```bash
$ git branch --show-current
blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928
$ git status --porcelain
$        # (empty output above == clean working tree at baseline)
$ git diff --name-status 2cd6ee777f8c2d3531559588bcfb18627ffb5d2c..HEAD
A	blitzy/documentation/app_2cd6ee777f8c.md
```

The `pytest` harnesses below were driven with the `flask_client` fixture (`tests/conftest.py:60`),
which wraps each test in a transaction it rolls back (`tests/conftest.py:61`) and sets
`config.DISABLE_RATE_LIMIT = True` (`tests/conftest.py:65`). Standalone scripts that needed to
commit to the real database (Q2's runner subprocess) ran outside that fixture.

---

# Question 1 — Mailbox verification-code brute-force enforcement

> *When a mailbox verification code is submitted incorrectly multiple times in succession, what
> observable state changes and enforcement mechanisms does the running system apply?*
> Specifically: **(a)** the attempt limit(s), **(b)** how failed attempts are tracked, **(c)** what
> ultimately prevents further submissions.

## Direct answer

- **(a) The limit is `MAX_ACTIVATION_TRIES = 3`** (`app/mailbox_utils.py:43`).
- **(b) Failed attempts are tracked by a per-record integer counter** that **counts up from 0**:
  `MailboxActivation.tries` (`app/models.py:2835`, class at `app/models.py:2828`).
- **(c) What ultimately prevents further submissions is deletion of the activation record.** Once
  `tries` reaches the cap, `clear_activation_codes_for_mailbox()` (`app/mailbox_utils.py:159`)
  deletes the `MailboxActivation` row, so subsequent submissions find no record to check.

The observed tolerance is exactly **three wrong guesses recorded** (`tries` goes `0→1→2→3`); the
**fourth** submission trips the guard and destroys the code. The verify route carries **no rate
limiter** — the per-record counter is the whole anti-brute-force control.

## Evidence

### The enforcement logic and its ordering

`verify_mailbox_code()` (`app/mailbox_utils.py:166`) fetches the latest activation and applies guards
in this order: the attempts cap **first** (`app/mailbox_utils.py:195`), then a 15-minute expiry
(`app/mailbox_utils.py:199`), then the code comparison, which increments `tries` on mismatch
(`app/mailbox_utils.py:209`). Crucially, the cap is compared **before** the increment
(`if activation.tries >= MAX_ACTIVATION_TRIES:` at `app/mailbox_utils.py:195`), which is why three
wrong values are recorded before the fourth submission is blocked.

### Q1-A — driving `verify_mailbox_code()` directly (counter before/during/after every submission)

```bash
$ pytest tests/blitzy_adhoc_test_q1.py -p no:cacheprovider --no-cov -s -q
```

```
===== Q1-A: direct verify_mailbox_code() — tries counter before/during/after each wrong submission =====
MAX_ACTIVATION_TRIES = 3
2026-07-08 06:11:06,590 - SL - INFO - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/mailbox_utils.py:88" - create_mailbox() -  - User <User 35 Test User user_02ov3q5vdg@mailbox.test> has created mailbox with dzruytqmmaxcytdtrlqc@dzruytqmmaxcytdtrlqc.com
2026-07-08 06:11:06,595 - SL - INFO - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/mailbox_utils.py:245" - send_verification_email() -  - Sending mailbox verification email to dzruytqmmaxcytdtrlqc@dzruytqmmaxcytdtrlqc.com with send link=True
2026-07-08 06:11:06,610 - SL - DEBUG - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/email_utils.py:303" - send_email() -  - send email to dzruytqmmaxcytdtrlqc@dzruytqmmaxcytdtrlqc.com, subject 'Please confirm your mailbox dzruytqmmaxcytdtrlqc@dzruytqmmaxcytdtrlqc.com'
2026-07-08 06:11:06,614 - SL - DEBUG - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/mail_sender.py:131" - send() -  - send email with subject 'Please confirm your mailbox dzruytqmmaxcytdtrlqc@dzruytqmmaxcytdtrlqc.com', from '"noreply@sl.local" <noreply@sl.local>' to 'dzruytqmmaxcytdtrlqc@dzruytqmmaxcytdtrlqc.com'
created mailbox id=77, correct code='wQ5LUAOYn-JQRin2Baq4-Q', submitting wrong code='wQ5LUAOYn-JQRin2Baq4-QWRONG'
INITIAL: MailboxActivation.tries=0  row_exists=True
2026-07-08 06:11:06,617 - SL - INFO - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/mailbox_utils.py:206" - verify_mailbox_code() -  - User <User 35 Test User user_02ov3q5vdg@mailbox.test> failed to verify mailbox 77 because code does not match
attempt 1: BEFORE tries=0 exists=True  ->  CannotVerifyError(msg='Invalid activation code')  ->  AFTER tries=1 exists=True
2026-07-08 06:11:06,622 - SL - INFO - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/mailbox_utils.py:206" - verify_mailbox_code() -  - User <User 35 Test User user_02ov3q5vdg@mailbox.test> failed to verify mailbox 77 because code does not match
attempt 2: BEFORE tries=1 exists=True  ->  CannotVerifyError(msg='Invalid activation code')  ->  AFTER tries=2 exists=True
2026-07-08 06:11:06,627 - SL - INFO - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/mailbox_utils.py:206" - verify_mailbox_code() -  - User <User 35 Test User user_02ov3q5vdg@mailbox.test> failed to verify mailbox 77 because code does not match
attempt 3: BEFORE tries=2 exists=True  ->  CannotVerifyError(msg='Invalid activation code')  ->  AFTER tries=3 exists=True
2026-07-08 06:11:06,632 - SL - INFO - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/mailbox_utils.py:196" - verify_mailbox_code() -  - User <User 35 Test User user_02ov3q5vdg@mailbox.test> failed to verify mailbox 77 more than 3 times
attempt 4: BEFORE tries=3 exists=True  ->  CannotVerifyError(msg='Invalid activation code. Please request another code.')  ->  AFTER tries=None exists=False
2026-07-08 06:11:06,637 - SL - INFO - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/mailbox_utils.py:191" - verify_mailbox_code() -  - User <User 35 Test User user_02ov3q5vdg@mailbox.test> failed to verify mailbox 77 because there is no activation
attempt 5: BEFORE tries=None exists=False  ->  MailboxError(msg='Invalid code')  ->  AFTER tries=None exists=False
```

Reading the transitions:

- **Attempts 1–3** each log `code does not match` (`app/mailbox_utils.py:206`) and increment the
  counter: `tries` `0→1→2→3`.
- **Attempt 4** finds `tries=3`, so the cap guard at `app/mailbox_utils.py:195` fires — it logs
  `failed to verify mailbox 77 more than 3 times` (`app/mailbox_utils.py:196`), calls
  `clear_activation_codes_for_mailbox(mailbox)` (`app/mailbox_utils.py:197`), and raises
  `CannotVerifyError("Invalid activation code. Please request another code.")`
  (`app/mailbox_utils.py:198`). The row is now **gone** (`exists=False`).
- **Attempt 5** finds no activation, logs `because there is no activation`
  (`app/mailbox_utils.py:191`), and raises the generic `MailboxError("Invalid code")`
  (`app/mailbox_utils.py:194`) — the **terminal prevention** is directly observable.

The terminal step is a bulk delete; `clear_activation_codes_for_mailbox()`
(`app/mailbox_utils.py:159-163`) runs
`Session.query(MailboxActivation).filter(MailboxActivation.mailbox_id == mailbox.id).delete()`.

### Q1-B — the real HTTP entry point `GET /dashboard/mailbox_verify`

The canonical route is `@dashboard_bp.route("/mailbox_verify")` / `def mailbox_verify()`
(`app/dashboard/views/mailbox.py:120-122`), guarded only by `@login_required`
(`app/dashboard/views/mailbox.py:121`). On a wrong code, `verify_mailbox_code()` raises
`CannotVerifyError`, which subclasses `MailboxError` (`app/mailbox_utils.py:38`); the route catches
`except mailbox_utils.MailboxError as e:` (`app/dashboard/views/mailbox.py:130`), logs the failure
(`app/dashboard/views/mailbox.py:131`), calls `flash(f"Cannot verify mailbox: {e.msg}", "error")`
(`app/dashboard/views/mailbox.py:132`), and **returns `redirect(url_for("dashboard.mailbox_route"))`**
(`app/dashboard/views/mailbox.py:133`). So every **failed** submission is an **HTTP 302 redirect** to
`/dashboard/mailbox` carrying a flashed error — the validation page
(`render_template("dashboard/mailbox_validation.html", mailbox=mailbox)`, `app/dashboard/views/mailbox.py:135`) is
reached **only on success**. Driving the real route through the `flask_client` with a logged-in user
reproduces the identical counter progression. With `follow_redirects=False` the raw `302` and its
`Location` header are visible; a final request with `follow_redirects=True` shows the client landing
on the mailbox page (`200`):

```
===== Q1-B: REAL HTTP entry point GET /dashboard/mailbox_verify (canonical) =====
2026-07-08 06:11:07,277 - SL - INFO - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/mailbox_utils.py:88" - create_mailbox() -  - User <User 36 Test User user_a7gcf1iaax@mailbox.test> has created mailbox with iarnvxydmxjsnkistqyi@iarnvxydmxjsnkistqyi.com
2026-07-08 06:11:07,280 - SL - INFO - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/mailbox_utils.py:245" - send_verification_email() -  - Sending mailbox verification email to iarnvxydmxjsnkistqyi@iarnvxydmxjsnkistqyi.com with send link=True
2026-07-08 06:11:07,293 - SL - DEBUG - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/email_utils.py:303" - send_email() -  - send email to iarnvxydmxjsnkistqyi@iarnvxydmxjsnkistqyi.com, subject 'Please confirm your mailbox iarnvxydmxjsnkistqyi@iarnvxydmxjsnkistqyi.com'
2026-07-08 06:11:07,297 - SL - DEBUG - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/mail_sender.py:131" - send() -  - send email with subject 'Please confirm your mailbox iarnvxydmxjsnkistqyi@iarnvxydmxjsnkistqyi.com', from '"noreply@sl.local" <noreply@sl.local>' to 'iarnvxydmxjsnkistqyi@iarnvxydmxjsnkistqyi.com'
logged-in user=user_a7gcf1iaax@mailbox.test, mailbox id=79
--- raw responses with follow_redirects=False (shows the redirect itself) ---
2026-07-08 06:11:07,301 - SL - INFO - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/mailbox_utils.py:206" - verify_mailbox_code() -  - User <User 36 Test User user_a7gcf1iaax@mailbox.test> failed to verify mailbox 79 because code does not match
2026-07-08 06:11:07,302 - SL - INFO - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/dashboard/views/mailbox.py:131" - mailbox_verify() -  - Cannot verify mailbox 79 because of Invalid activation code
2026-07-08 06:11:07,302 - SL - DEBUG - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard/mailbox_verify ImmutableMultiDict([('mailbox_id', '79'), ('code', '5aZb1jRsdZ_3-ULQAxFlzwWRONG')]) 302, takes 0.003803730010986328
  attempt 1: raw_status=302 Location='http://sl.test/dashboard/mailbox' BEFORE tries=0 exists=True AFTER tries=1 exists=True flash='Cannot verify mailbox: Invalid activation code'
2026-07-08 06:11:07,310 - SL - INFO - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/mailbox_utils.py:206" - verify_mailbox_code() -  - User <User 36 Test User user_a7gcf1iaax@mailbox.test> failed to verify mailbox 79 because code does not match
2026-07-08 06:11:07,310 - SL - INFO - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/dashboard/views/mailbox.py:131" - mailbox_verify() -  - Cannot verify mailbox 79 because of Invalid activation code
2026-07-08 06:11:07,310 - SL - DEBUG - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard/mailbox_verify ImmutableMultiDict([('mailbox_id', '79'), ('code', '5aZb1jRsdZ_3-ULQAxFlzwWRONG')]) 302, takes 0.003751516342163086
  attempt 2: raw_status=302 Location='http://sl.test/dashboard/mailbox' BEFORE tries=1 exists=True AFTER tries=2 exists=True flash='Cannot verify mailbox: Invalid activation code'
2026-07-08 06:11:07,317 - SL - INFO - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/mailbox_utils.py:206" - verify_mailbox_code() -  - User <User 36 Test User user_a7gcf1iaax@mailbox.test> failed to verify mailbox 79 because code does not match
2026-07-08 06:11:07,317 - SL - INFO - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/dashboard/views/mailbox.py:131" - mailbox_verify() -  - Cannot verify mailbox 79 because of Invalid activation code
2026-07-08 06:11:07,318 - SL - DEBUG - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard/mailbox_verify ImmutableMultiDict([('mailbox_id', '79'), ('code', '5aZb1jRsdZ_3-ULQAxFlzwWRONG')]) 302, takes 0.003654003143310547
  attempt 3: raw_status=302 Location='http://sl.test/dashboard/mailbox' BEFORE tries=2 exists=True AFTER tries=3 exists=True flash='Cannot verify mailbox: Invalid activation code'
2026-07-08 06:11:07,324 - SL - INFO - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/mailbox_utils.py:196" - verify_mailbox_code() -  - User <User 36 Test User user_a7gcf1iaax@mailbox.test> failed to verify mailbox 79 more than 3 times
2026-07-08 06:11:07,325 - SL - INFO - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/dashboard/views/mailbox.py:131" - mailbox_verify() -  - Cannot verify mailbox 79 because of Invalid activation code. Please request another code.
2026-07-08 06:11:07,325 - SL - DEBUG - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard/mailbox_verify ImmutableMultiDict([('mailbox_id', '79'), ('code', '5aZb1jRsdZ_3-ULQAxFlzwWRONG')]) 302, takes 0.003731250762939453
  attempt 4: raw_status=302 Location='http://sl.test/dashboard/mailbox' BEFORE tries=3 exists=True AFTER tries=None exists=False flash='Cannot verify mailbox: Invalid activation code. Please request another code.'
2026-07-08 06:11:07,331 - SL - INFO - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/mailbox_utils.py:191" - verify_mailbox_code() -  - User <User 36 Test User user_a7gcf1iaax@mailbox.test> failed to verify mailbox 79 because there is no activation
2026-07-08 06:11:07,331 - SL - INFO - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/dashboard/views/mailbox.py:131" - mailbox_verify() -  - Cannot verify mailbox 79 because of Invalid code
2026-07-08 06:11:07,331 - SL - DEBUG - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard/mailbox_verify ImmutableMultiDict([('mailbox_id', '79'), ('code', '5aZb1jRsdZ_3-ULQAxFlzwWRONG')]) 302, takes 0.0029802322387695312
  attempt 5: raw_status=302 Location='http://sl.test/dashboard/mailbox' BEFORE tries=None exists=False AFTER tries=None exists=False flash='Cannot verify mailbox: Invalid code'
--- one request with follow_redirects=True (client follows the 302 to the mailbox page -> 200) ---
2026-07-08 06:11:07,362 - SL - INFO - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/mailbox_utils.py:88" - create_mailbox() -  - User <User 36 Test User user_a7gcf1iaax@mailbox.test> has created mailbox with mnvpmmkovvjaxawjajat@mnvpmmkovvjaxawjajat.com
2026-07-08 06:11:07,365 - SL - INFO - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/mailbox_utils.py:245" - send_verification_email() -  - Sending mailbox verification email to mnvpmmkovvjaxawjajat@mnvpmmkovvjaxawjajat.com with send link=True
2026-07-08 06:11:07,378 - SL - DEBUG - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/email_utils.py:303" - send_email() -  - send email to mnvpmmkovvjaxawjajat@mnvpmmkovvjaxawjajat.com, subject 'Please confirm your mailbox mnvpmmkovvjaxawjajat@mnvpmmkovvjaxawjajat.com'
2026-07-08 06:11:07,382 - SL - DEBUG - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/mail_sender.py:131" - send() -  - send email with subject 'Please confirm your mailbox mnvpmmkovvjaxawjajat@mnvpmmkovvjaxawjajat.com', from '"noreply@sl.local" <noreply@sl.local>' to 'mnvpmmkovvjaxawjajat@mnvpmmkovvjaxawjajat.com'
2026-07-08 06:11:07,386 - SL - INFO - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/mailbox_utils.py:206" - verify_mailbox_code() -  - User <User 36 Test User user_a7gcf1iaax@mailbox.test> failed to verify mailbox 80 because code does not match
2026-07-08 06:11:07,386 - SL - INFO - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/dashboard/views/mailbox.py:131" - mailbox_verify() -  - Cannot verify mailbox 80 because of Invalid activation code
2026-07-08 06:11:07,386 - SL - DEBUG - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard/mailbox_verify ImmutableMultiDict([('mailbox_id', '80'), ('code', 'RRqfvfvogtfcMOK1i_CRbgWRONG')]) 302, takes 0.0037229061126708984
2026-07-08 06:11:07,406 - SL - DEBUG - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard/mailbox ImmutableMultiDict([]) 200, takes 0.018623828887939453
  followed: final_status=200 (landed on mailbox route after 302) flash=None
```

Every failed submission returns **`raw_status=302`** with **`Location='http://sl.test/dashboard/mailbox'`**
and a flash of the form `Cannot verify mailbox: <e.msg>` — never a rendered validation page. The flashed
text on attempt 4 (`Cannot verify mailbox: Invalid activation code. Please request another code.`) and
the row disappearing (`exists=False`) confirm the terminal deletion fires through the real web route; the
final `follow_redirects=True` request shows the browser being sent on to `GET /dashboard/mailbox` (`200`).
The `tries` progression (`0→1→2→3→deleted`) is byte-for-byte identical to the direct call in Q1-A, proving
the HTTP route and the underlying helper enforce the same counter.

### Two-run stability (Q1)

The whole Q1 harness was executed **twice** (`/tmp/blitzy_evidence/q1.run1.txt` and `q1.run2.txt`, each
`5 passed, 18 warnings`). Every magnitude/threshold observable is identical across the two runs; the only
differences are values that are inherently non-deterministic — the auto-increment database primary keys of
the freshly created mailbox rows, the randomly generated activation codes, wall-clock timestamps, the PID,
and the per-request `takes` durations. Normalizing those away, `diff` reports the runs as identical.

| Observable | Run 1 | Run 2 | Stable? |
|---|---|---|---|
| `MAX_ACTIVATION_TRIES` | `3` | `3` | ✓ |
| `tries` progression (Q1-A direct, Q1-B HTTP) | `0→1→2→3→deleted` | `0→1→2→3→deleted` | ✓ |
| Wrong submission that deletes the row | 4th | 4th | ✓ |
| Raw HTTP status of each failed submit (Q1-B) | `302` | `302` | ✓ |
| `Location` header (Q1-B) | `http://sl.test/dashboard/mailbox` | `http://sl.test/dashboard/mailbox` | ✓ |
| Followed status after the `302` (Q1-B) | `200` | `200` | ✓ |
| `AccountActivation` progression (Q1-D) | `3→2→1→deleted` | `3→2→1→deleted` | ✓ |
| `mailbox_verify` present in limiter registry (Q1-E) | absent | absent | ✓ |

### Sibling variant — the signed-link "Old way" path

When the route is hit **without** a `code` parameter it takes the signed-link branch
`return verify_with_signed_secret(mailbox_id)` (`app/dashboard/views/mailbox.py:127`). That function
is declared `def verify_with_signed_secret(request: str)` (`app/dashboard/views/mailbox.py:138`) and
immediately does `mailbox_verify_request = request.args.get("mailbox_id")`
(`app/dashboard/views/mailbox.py:140`).

**Observed (unexpected but real):** the canonical route returns **HTTP 500**, because the parameter
name `request` shadows Flask's global `request`, and the *string* the route passed in has no `.args`
attribute. The complete, contiguous section of the single `pytest` run (`[NON-CANONICAL]` direct call
labeled inline) is:

```
===== Q1-C: signed-link 'Old way' verify_with_signed_secret() (max_age=900 -> 15 min) =====
2026-07-08 06:11:07,666 - SL - INFO - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 06:11:07,909 - SL - DEBUG - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/auth/views/login_utils.py:35" - after_login() -  - log user <User 37 Test User user_czdo7f8lv2@mailbox.test> in
2026-07-08 06:11:07,909 - SL - DEBUG - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/auth/views/login_utils.py:44" - after_login() -  - redirect user to dashboard
2026-07-08 06:11:07,910 - SL - DEBUG - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 302, takes 0.23789477348327637
2026-07-08 06:11:07,914 - SL - DEBUG - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/dashboard/views/index.py:172" - index() -  - Show intro to <User 37 Test User user_czdo7f8lv2@mailbox.test>
2026-07-08 06:11:08,020 - SL - DEBUG - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard/ ImmutableMultiDict([]) 200, takes 0.10887694358825684
2026-07-08 06:11:08,052 - SL - INFO - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/mailbox_utils.py:88" - create_mailbox() -  - User <User 37 Test User user_czdo7f8lv2@mailbox.test> has created mailbox with dbdcehpcxwdpdfjhurnb@dbdcehpcxwdpdfjhurnb.com
2026-07-08 06:11:08,055 - SL - INFO - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/mailbox_utils.py:245" - send_verification_email() -  - Sending mailbox verification email to dbdcehpcxwdpdfjhurnb@dbdcehpcxwdpdfjhurnb.com with send link=True
2026-07-08 06:11:08,068 - SL - DEBUG - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/email_utils.py:303" - send_email() -  - send email to dbdcehpcxwdpdfjhurnb@dbdcehpcxwdpdfjhurnb.com, subject 'Please confirm your mailbox dbdcehpcxwdpdfjhurnb@dbdcehpcxwdpdfjhurnb.com'
2026-07-08 06:11:08,072 - SL - DEBUG - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/mail_sender.py:131" - send() -  - send email with subject 'Please confirm your mailbox dbdcehpcxwdpdfjhurnb@dbdcehpcxwdpdfjhurnb.com', from '"noreply@sl.local" <noreply@sl.local>' to 'dbdcehpcxwdpdfjhurnb@dbdcehpcxwdpdfjhurnb.com'
mailbox id=82 verified_before=False
2026-07-08 06:11:08,074 - SL - ERROR - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/server.py:390" - error_handler() -  - 'str' object has no attribute 'args'
Traceback (most recent call last):
  File "/tmp/slvenv/lib/python3.10/site-packages/flask/app.py", line 1950, in full_dispatch_request
    rv = self.dispatch_request()
  File "/tmp/slvenv/lib/python3.10/site-packages/flask/app.py", line 1936, in dispatch_request
    return self.view_functions[rule.endpoint](**req.view_args)
  File "/tmp/slvenv/lib/python3.10/site-packages/flask_login/utils.py", line 272, in decorated_view
    return func(*args, **kwargs)
  File "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/dashboard/views/mailbox.py", line 127, in mailbox_verify
    return verify_with_signed_secret(mailbox_id)
  File "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/dashboard/views/mailbox.py", line 140, in verify_with_signed_secret
    mailbox_verify_request = request.args.get("mailbox_id")
AttributeError: 'str' object has no attribute 'args'
2026-07-08 06:11:08,078 - SL - DEBUG - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard/mailbox_verify ImmutableMultiDict([('mailbox_id', '82')]) 500, takes 0.004930019378662109
(i) CANONICAL authenticated route GET (no code): status=500 (errorhandler server.py:389 renders 500 page)
(ii) verify_with_signed_secret(<str>) exactly as the route invokes it @L127 -> full traceback:
Traceback (most recent call last):
  File "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/tests/blitzy_adhoc_test_q1.py", line 122, in test_q1c_signed_link
    verify_with_signed_secret(str(mailbox.id))
  File "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/dashboard/views/mailbox.py", line 140, in verify_with_signed_secret
    mailbox_verify_request = request.args.get("mailbox_id")
AttributeError: 'str' object has no attribute 'args'

2026-07-08 06:11:08,081 - SL - DEBUG - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/dashboard/views/mailbox.py:173" - verify_with_signed_secret() -  - Mailbox <Mailbox 82 dbdcehpcxwdpdfjhurnb@dbdcehpcxwdpdfjhurnb.com> is verified
(iii) [NON-CANONICAL] verify_with_signed_secret(flask_request): resp_type=str verified_after=True
(iv) unsign(max_age=900) fresh token -> OK, recovered=b'WzgyLCAiZGJkY2VocGN4d2RwZGZqaHVybmJAZGJkY2VocGN4d2RwZGZqaHVybmIuY29tIl0='
(iv) unsign(max_age=900) on 1000s-old token -> SignatureExpired: Signature age 1000 > 900 seconds
```

The single captured section above (one contiguous run) shows all four facts: **(i)** the canonical
authenticated route `GET /dashboard/mailbox_verify?mailbox_id=82` (no `code`) returns **HTTP 500** —
the error handler at `server.py:390` logs `'str' object has no attribute 'args'` and `after_request`
records the `500`; **(ii)** calling `verify_with_signed_secret(str(mailbox.id))` exactly as the route does
(`app/dashboard/views/mailbox.py:127`) raises the same `AttributeError` at
`app/dashboard/views/mailbox.py:140`, because the parameter name `request` shadows Flask's global
`request` and a bare *string* has no `.args`; **(iii)** when the function is instead handed a real
Flask request object (a `[NON-CANONICAL]` direct call, to reveal the intended mechanics) it verifies
the mailbox (`verified_after=True`, log at `app/dashboard/views/mailbox.py:173`); and **(iv)** the
15-minute window is enforced by `TimestampSigner(MAILBOX_SECRET).unsign(mailbox_verify_request, max_age=900)`
(`app/dashboard/views/mailbox.py:139,142`) — a fresh token decodes to
`b'WzgyLCAiZGJkY2VocGN4d2RwZGZqaHVybmJAZGJkY2VocGN4d2RwZGZqaHVybmIuY29tIl0='`, while a token aged
1000 s raises `SignatureExpired: Signature age 1000 > 900 seconds`.

So the signed-link path implements a **15-minute** (`max_age=900`) window, but as invoked by the
canonical route (passing a bare string) it raises `AttributeError` and yields a 500. `[inferred]`
that this path is effectively dead as wired; the observed 500 is the runtime fact.

### Sibling variant — `AccountActivation` counts in the OPPOSITE direction

The mobile-signup counterpart, `AccountActivation` (`app/models.py:2838`), carries a `tries` column
that **defaults to 3** and **decrements** on each wrong attempt (column comment: "nb tries decrements
each time user enters wrong code"; `CheckConstraint(tries >= 0)`). Its enforcement lives in the API
route `POST /api/auth/activate` (`app/api/views/auth.py`): `account_activation.tries -= 1`
(`app/api/views/auth.py:178`) and deletion when `if account_activation.tries == 0:`
(`app/api/views/auth.py:181`). Observed:

```
===== Q1-D: AccountActivation counterpart — tries DECREMENTS 3->0 (opposite direction) =====
2026-07-08 06:11:08,343 - SL - INFO - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
AccountActivation created with tries=3 (model default=3)
2026-07-08 06:11:08,355 - SL - DEBUG - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/server.py:284" - after_request() -  - 127.0.0.1 POST /api/auth/activate ImmutableMultiDict([]) 400, takes 0.0035295486450195312
attempt 1: BEFORE tries=3 -> POST /api/auth/activate wrong code -> status=400 body={'error': 'Wrong email or code'} -> AFTER tries=2 exists=True
2026-07-08 06:11:08,362 - SL - DEBUG - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/server.py:284" - after_request() -  - 127.0.0.1 POST /api/auth/activate ImmutableMultiDict([]) 400, takes 0.003208160400390625
attempt 2: BEFORE tries=2 -> POST /api/auth/activate wrong code -> status=400 body={'error': 'Wrong email or code'} -> AFTER tries=1 exists=True
2026-07-08 06:11:08,369 - SL - DEBUG - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/server.py:284" - after_request() -  - 127.0.0.1 POST /api/auth/activate ImmutableMultiDict([]) 410, takes 0.0036661624908447266
attempt 3: BEFORE tries=1 -> POST /api/auth/activate wrong code -> status=410 body={'error': 'Too many wrong tries'} -> AFTER tries=None exists=False
2026-07-08 06:11:08,375 - SL - DEBUG - 98346 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/server.py:284" - after_request() -  - 127.0.0.1 POST /api/auth/activate ImmutableMultiDict([]) 400, takes 0.0020797252655029297
attempt 4: BEFORE tries=None -> POST /api/auth/activate wrong code -> status=400 body={'error': 'Wrong email or code'} -> AFTER tries=None exists=False
```

Contrast of the two mechanisms (both effectively tolerate 3 wrong guesses, opposite bookkeeping):

| | `MailboxActivation` (Q1 subject) | `AccountActivation` (mobile signup) |
|---|---|---|
| Start value | `tries` default **0** (`app/models.py:2835`) | `tries` default **3** (`app/models.py:2850`) |
| On wrong code | **increments** `+1` (`app/mailbox_utils.py:209`) | **decrements** `-1` (`app/api/views/auth.py:178`) |
| Terminal trigger | `tries >= 3` on the 4th submit (`app/mailbox_utils.py:195`) | `tries == 0` on the 3rd wrong (`app/api/views/auth.py:181`) |
| Terminal action | delete row (`app/mailbox_utils.py:159`) | delete row + `410 Too many wrong tries` |

### Sibling variant — rate-limit contrast (there is none on the verify route)

Runtime introspection of the Flask-Limiter registry shows the verify route is **absent** (no limit),
while a sibling mailbox route and the account-activation route **are** registered:

```
===== Q1-E: rate-limit contrast (verify route has NO limiter; mailbox_detail POST-only 20/minute) =====
_route_limits['app.dashboard.views.mailbox.mailbox_verify']: present=False limits=[]
_route_limits['app.dashboard.views.mailbox_detail.mailbox_detail_route']: present=True limits=["20 per 1 minute methods=['post']"]
_route_limits['app.api.views.auth.auth_activate']: present=True limits=['10 per 1 minute methods=None']
config.DISABLE_RATE_LIMIT during tests = True
```

The sibling limit is `@limiter.limit("20/minute", methods=["POST"])` (`app/dashboard/views/mailbox_detail.py:37`)
— note it is **POST-only**, and `mailbox_detail` is a different route from the verify route. The
conclusion: **the anti-brute-force control for mailbox verification is the per-record `tries`
counter and its terminal deletion, not a rate limiter.**

> Not to be conflated: the 30-character registration/login `ActivationCode` created in
> `app/auth/views/register.py` is a **different** mechanism and is out of scope for this question.

### Corroborating canonical harness

```bash
$ pytest tests/test_mailbox_utils.py -p no:cacheprovider --no-cov -q
23 passed, 18 warnings in 2.18s
```

`tests/test_mailbox_utils.py` includes `test_verify_fail` (asserts `activation.tries == i+1` after
each wrong attempt), `test_verify_too_may` (pre-sets `tries = MAX_ACTIVATION_TRIES` then expects
`CannotVerifyError`), and `test_verify_ok` (asserts the activation is `None` after success) — all
consistent with the transitions captured above.


---

# Question 2 — Background task lifecycle and retry/recovery

> *For background tasks that are scheduled and later picked up for execution, trace the complete
> lifecycle from initial creation through final completion*, including **(a)** the recovery/retry
> behavior when a task errors during execution, and **(b)** the observable state that reflects that
> failure.

## Direct answer

- **Lifecycle:** a `Job` (`app/models.py:2683`) moves `ready (0) → taken (1) → done (2)`. On
  creation `state=ready, attempts=0, taken=False, taken_at=None`; on pickup the runner sets
  `taken=True, taken_at=now, state=taken, attempts += 1` and commits **before** executing; on
  success it sets `state=done`.
- **(a) Recovery/retry** is *implicit and time-based*. There is **no `try/except`** around job
  execution (`job_runner.py:342`). A job that raises is simply left in `taken`; it becomes eligible
  again only once `taken_at` is older than **`JOB_TAKEN_RETRY_WAIT_MINS = 30`** minutes
  (`app/config.py:565`) **and** `attempts < JOB_MAX_ATTEMPTS = 5` (`app/config.py:564`). After 5
  attempts it is abandoned (never re-selected).
- **(b) The observable failure state** is a job **stuck in `state = taken (1)`** with `attempts`
  already incremented and `taken_at` set — `done` is never reached. `JobState.error (=3)` is
  **never written by the runner** (see the refinement below).

## Evidence

### The state enum and the runner loop

`JobState` (`app/models.py:253`) defines `ready = 0` (`app/models.py:254`), `taken = 1`
(`app/models.py:255`), `done = 2` (`app/models.py:256`), `error = 3` (`app/models.py:257`). The
runner's `__main__` loop (`job_runner.py:329-347`)
reads:

```python
job.taken = True                      # job_runner.py:337
job.taken_at = arrow.now()            # job_runner.py:338
job.state = JobState.taken.value      # job_runner.py:339
job.attempts += 1                     # job_runner.py:340
Session.commit()                      # job_runner.py:341
process_job(job)                      # job_runner.py:342  <-- NO try/except
job.state = JobState.done.value       # job_runner.py:344
Session.commit()                      # job_runner.py:345
```

Because the `taken`/`attempts`/`taken_at` update is committed at `job_runner.py:341` **before**
`process_job(job)` runs at `job_runner.py:342`, a crash inside `process_job` leaves those values
persisted while `done` (`job_runner.py:344`) is never reached.

### Failure path — the real runner (`python job_runner.py`) with a job that raises

A real `Job` named `batch-import` (matching `config.JOB_BATCH_IMPORT = "batch-import"`,
`app/config.py:305`) whose `payload` (a JSON column, `app/models.py:2689`) is the empty string
reaches the batch-import dispatch branch (`elif job.name == config.JOB_BATCH_IMPORT:`,
`job_runner.py:222`), whose very first line `batch_import_id = job.payload.get("batch_import_id")`
(`job_runner.py:223`) calls `.get()` on a `str` and raises
`AttributeError: 'str' object has no attribute 'get'`. Because there is **no `try/except`** around
`process_job(job)` (`job_runner.py:342`), that exception propagates out of the `__main__` loop and
crashes the process — `state = done` (`job_runner.py:344`) is never reached. Driving the real entry
point (helper commits a `ready` row to the real DB, then the real `python job_runner.py` loop picks
it up):

```
==================== Q2-FAILURE PATH ====================
$ python tests/blitzy_adhoc_test_q2_helper.py insert_fail
[insert_fail] created (real handler branch='batch-import', payload="") -> Job id=233 name='batch-import' state=0(ready) attempts=0 taken=False taken_at=None run_at=None
--- BEFORE (real runner not yet run) ---
$ python tests/blitzy_adhoc_test_q2_helper.py show BEFORE-fail
[show BEFORE-fail] 1 job(s) in DB:
   Job id=233 name='batch-import' state=0(ready) attempts=0 taken=False taken_at=None run_at=None
--- DRIVE REAL RUNNER ---
$ timeout -s INT 30 python job_runner.py
runner exit code = 1   (1 = crashed on unhandled exception, state=done never reached)
--- runner output from job pickup onward (COMPLETE traceback, unedited) ---
2026-07-08 06:34:46,509 - SL - DEBUG - 108240 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/job_runner.py:334" - <module>() -  - Take job <Job 233 batch-import >
Traceback (most recent call last):
  File "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/job_runner.py", line 342, in <module>
    process_job(job)
  File "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/job_runner.py", line 223, in process_job
    batch_import_id = job.payload.get("batch_import_id")
AttributeError: 'str' object has no attribute 'get'
--- AFTER (job STUCK in taken; done never reached) ---
$ python tests/blitzy_adhoc_test_q2_helper.py show AFTER-fail
[show AFTER-fail] 1 job(s) in DB:
   Job id=233 name='batch-import' state=1(taken) attempts=1 taken=True taken_at=2026-07-08T06:34:46.509862+00:00 run_at=None
```

The process exits with code **1** (a genuine crash — a timeout kill would be `124`), confirming there
is no error handling around `process_job`. The traceback's frames pin the crash exactly: the runner's
`__main__` loop at `job_runner.py:342` called `process_job(job)`, which raised inside
`job_runner.py:223`.

**Observable failure state (before → during → after):** the row starts `state=0(ready) attempts=0
taken=False taken_at=None`; the loop commits `taken=True, taken_at=now, state=1(taken), attempts=1`
**before** dispatch (`job_runner.py:337-341`); after the crash it is left exactly there —
`state=1(taken)`, `attempts=1`, `taken=True`, `taken_at` set — and `state` is **never** `2(done)` and
**never** `3(error)`.

### Success path — a job that completes reaches `done (2)`

An unknown job name falls through to the final `else:` (`job_runner.py:303`) whose body is
`LOG.e("Unknown job name %s", job.name)` (`job_runner.py:304`), which does **not** raise, so the loop
proceeds to `state = done` (`job_runner.py:344`). Here the job has the empty name `''` — an unknown
name — so it exercises exactly that branch:

```
==================== Q2-SUCCESS PATH ====================
$ python tests/blitzy_adhoc_test_q2_helper.py insert_success
[insert_success] created (unknown job name '' -> no-op handler) -> Job id=234 name='' state=0(ready) attempts=0 taken=False taken_at=None run_at=None
--- BEFORE ---
$ python tests/blitzy_adhoc_test_q2_helper.py show BEFORE-success
[show BEFORE-success] 1 job(s) in DB:
   Job id=234 name='' state=0(ready) attempts=0 taken=False taken_at=None run_at=None
--- DRIVE REAL RUNNER (unknown-name no-op reaches done, then polls forever) ---
$ timeout -s TERM 15 python job_runner.py
runner exit code = 124   (124 = timeout-killed after job reached done)
--- runner log from job pickup onward ---
2026-07-08 06:34:54,253 - SL - DEBUG - 108304 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/job_runner.py:334" - <module>() -  - Take job <Job 234  >
2026-07-08 06:34:54,256 - SL - ERROR - 108304 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/job_runner.py:304" - process_job() -  - Unknown job name 
NoneType: None
--- AFTER (job reached done) ---
$ python tests/blitzy_adhoc_test_q2_helper.py show AFTER-success
[show AFTER-success] 1 job(s) in DB:
   Job id=234 name='' state=2(done) attempts=1 taken=True taken_at=2026-07-08T06:34:54.253906+00:00 run_at=None
```

Here the runner is stopped with `timeout -s TERM 15` (exit `124`) only because the `while True:` loop
polls forever (`time.sleep(10)`, `job_runner.py:347`) once the queue is drained — the job itself had
already reached `state=2(done)` before the kill. So the cross-product is observed: **failure ⇒ stuck
in `taken (1)`; success ⇒ `done (2)`** — in both cases `attempts` ends at `1`.

### Retry eligibility — the real `get_jobs_to_run()` across the 30-min window and 5-attempt cap

`get_jobs_to_run()` (`job_runner.py:307-326`) re-selects a job when
`state == ready` **OR** (`state == taken` **AND** `taken_at < now - 30 min` **AND** `attempts < 5`),
further gated by `run_at` being null or `<= now + 10 min` (`job_runner.py:323`). The constants are
confirmed at runtime by the helper's `constants` subcommand:

```
==================== Q2-CONSTANTS ====================
$ python tests/blitzy_adhoc_test_q2_helper.py constants
config.JOB_MAX_ATTEMPTS          = 5  (app/config.py:564)
config.JOB_TAKEN_RETRY_WAIT_MINS = 30  (app/config.py:565)
JobState: ready=0 taken=1 done=2 error=3  (app/models.py:253-257)
```

The `eligibility` subcommand seeds seven scratch `Job` rows spanning the full cross-product of the
selection predicate (`state`, `taken_at` relative to the 30-minute window, `attempts` relative to the
5-attempt cap, and a future `run_at`), then calls the **real** `get_jobs_to_run()` and prints which
rows it returns. The seed values (`taken_at` back-dating, `attempts` edits, future `run_at`) are
`[TEST FIXTURE]` manipulations of scratch DB state, not source changes. The exact command and its
complete, unedited output:

```
==================== Q2-ELIGIBILITY (retry-window / attempts-cap) ====================
$ python tests/blitzy_adhoc_test_q2_helper.py eligibility
[eligibility] JOB_TAKEN_RETRY_WAIT_MINS=30 JOB_MAX_ATTEMPTS=5
[eligibility] get_jobs_to_run() returned 4 job(s):
   [ELIGIBLE] ready, run_at=None                               -> Job id=226 name='' state=0(ready) attempts=0 taken=False taken_at=None run_at=None
   [ELIGIBLE] ready, run_at=now                                -> Job id=227 name='' state=0(ready) attempts=0 taken=False taken_at=None run_at=2026-07-08T06:34:40.862590+00:00
   [ELIGIBLE] taken, taken_at=-40m, attempts=0                 -> Job id=228 name='' state=1(taken) attempts=0 taken=False taken_at=2026-07-08T05:54:40.862590+00:00 run_at=None
   [ELIGIBLE] taken, taken_at=-40m, attempts=4                 -> Job id=229 name='' state=1(taken) attempts=4 taken=False taken_at=2026-07-08T05:54:40.862590+00:00 run_at=None
   [not-elig] taken, taken_at=-5m (within retry window)        -> Job id=230 name='' state=1(taken) attempts=0 taken=False taken_at=2026-07-08T06:29:40.862590+00:00 run_at=None
   [not-elig] taken, taken_at=-40m, attempts=5 (cap reached)   -> Job id=231 name='' state=1(taken) attempts=5 taken=False taken_at=2026-07-08T05:54:40.862590+00:00 run_at=None
   [not-elig] ready, run_at=+2h (future)                       -> Job id=232 name='' state=0(ready) attempts=0 taken=False taken_at=2026-07-08T08:34:40.862590+00:00 run_at=2026-07-08T08:34:40.862590+00:00
[eligibility] expected ELIGIBLE ids=[226, 227, 228, 229] got=[226, 227, 228, 229] MATCH=True
[eligibility] cleaned up harness jobs
```

This directly demonstrates each clause of the predicate: a `ready` job is always eligible (whether
`run_at` is null or already due), but a `ready` job with a **future** `run_at` (+2h) is **not**
selected; a `taken` job is **not** retried within the 30-minute window (`taken_at=-5m`); it becomes
eligible once `taken_at` crosses the window (`taken_at=-40m`, `attempts=0` and `attempts=4` both
selected); and it is **abandoned** at `attempts >= 5` (the boundary `4 < 5` is eligible, `5` is not).
The runner's assertion line confirms the selected set exactly:
`expected ELIGIBLE ids=[226, 227, 228, 229] got=[226, 227, 228, 229] MATCH=True`.

### Two-run stability (Q2)

The whole Q2 harness — the `constants` and `eligibility` subcommands plus the failure path
(`insert_fail` + the real `python job_runner.py` loop) and the success path (`insert_success` + the
real `python job_runner.py` loop) — was executed **twice**
(`/tmp/blitzy_evidence/q2.run1.txt` and `q2.run2.txt`). Every constant, threshold, state and
attempts observable is identical across the two runs; the only differences are values that are
inherently non-deterministic — the auto-increment database primary keys of the freshly created `Job`
rows (run 1 seeded ids `226`–`234`, run 2 seeded ids `235`–`243`), the `taken_at`/`run_at` wall-clock
timestamps, and the runner process PIDs. Normalizing those away, `diff` reports the two runs as
identical:

```
$ diff <(norm q2.run1.txt) <(norm q2.run2.txt)
*** Q2 run1 == run2 after PK/timestamp/PID normalization -> BEHAVIORALLY IDENTICAL ***
```

| Observable | Run 1 | Run 2 | Stable? |
|---|---|---|---|
| `JOB_MAX_ATTEMPTS` (`app/config.py:564`) | `5` | `5` | ✓ |
| `JOB_TAKEN_RETRY_WAIT_MINS` (`app/config.py:565`) | `30` | `30` | ✓ |
| `get_jobs_to_run()` returned count | `4` | `4` | ✓ |
| Eligible set `MATCH` (expected == got) | `True` | `True` | ✓ |
| Eligible rows (positionally, of the 7 seeded) | first 4 (ready×2, taken-past-under-cap×2) | first 4 (ready×2, taken-past-under-cap×2) | ✓ |
| Excluded rows (positionally, of the 7 seeded) | last 3 (within-window, cap-reached, future `run_at`) | last 3 (within-window, cap-reached, future `run_at`) | ✓ |
| Failure path — runner exit code | `1` | `1` | ✓ |
| Failure path — AFTER `state` | `1(taken)` | `1(taken)` | ✓ |
| Failure path — AFTER `attempts` | `1` | `1` | ✓ |
| Success path — runner exit code | `124` | `124` | ✓ |
| Success path — AFTER `state` | `2(done)` | `2(done)` | ✓ |
| Success path — AFTER `attempts` | `1` | `1` | ✓ |

### `JobState.error` — defined, but never written by the runner (refinement)

The runner only ever writes `taken` and `done`:

```bash
$ grep -n 'job.state *=' job_runner.py
339:                job.state = JobState.taken.value
344:                job.state = JobState.done.value
$ grep -n 'JobState.error' job_runner.py
   # (no output) -> job_runner.py never references JobState.error
```

`JobState.error (=3)` **is** referenced elsewhere — but only by a **separate** maintenance entry
point, `tasks/cleanup_old_jobs.py:15` (a query filter that deletes old `done`/`error`/stuck-`taken`
jobs), and it is set only in that task's test fixtures (`tests/tasks/test_cleanup_old_jobs.py:21,46`).
The **runner never transitions a job to `error`** — a runner-failed job lingers in `taken`. (This
refines the working assumption that `error` is "never assigned anywhere": it is assigned, just never
by the runner.)

### Corroborating canonical harness

```bash
$ pytest tests/jobs/test_job_runner.py -p no:cacheprovider --no-cov -q
1 passed, 18 warnings in 0.02s
```

`tests/jobs/test_job_runner.py::test_get_jobs_to_run` (`tests/jobs/test_job_runner.py:8`) creates
jobs in each state/`taken_at`/`attempts`/`run_at` combination and asserts the count returned by
`get_jobs_to_run()`, corroborating the eligibility rule captured above.

### Contrast — the `yacron`/`cron.py` scheduler is NOT the `Job` table

Time-based maintenance runs via a **separate** subsystem: `crontab.yml` defines `yacron` schedules
that invoke `python /code/cron.py -j <name>`. The complete set of scheduled jobs (all 15 entries in
`crontab.yml`) is: `stats` (`0 0 * * *`), `delete_old_monitoring` (`15 1 * * *`), `check_custom_domain`
(`15 2 * * *`), `check_hibp` (`15 3 * * *`), `notify_hibp` (`15 4 * * *`), `delete_logs`
(`15 5 * * *`), `delete_old_data` (`30 5 * * *`), `poll_apple_subscription` (`15 6 * * *`),
`notify_trial_end` (`15 8 * * *`), `notify_manual_subscription_end` (`15 9 * * *`), `notify_premium_end`
(`15 10 * * *`), `delete_scheduled_users` (`15 11 * * *`), `send_undelivered_mails` (`*/5 * * * *`),
`clear_alias_audit_log` (`0 * * * *`), and `clear_user_audit_log` (`0 * * * *`). `cron.py` dispatches
via `argparse -j`. This is a distinct process from the `Job` DB table that Question 2 concerns.


---

# Question 3 — VERP bounce-address format and directional handling

> *During email forwarding through an alias, what exact format is used for the special address
> generated to handle delivery failures?* Answer: **(a)** the exact address format, **(b)** how an
> inbound failure notification is decoded to identify the original email, **(c)** what state changes
> are recorded, and **(d)** how handling differs by DIRECTION (forward phase vs reply phase).

## Direct answer

- **(a) Exact format:** `sl.<base32-payload>.<base32-hmac>@<domain>`, **lowercased**, produced by the
  single return expression in `generate_verp_email()` (`app/email_utils.py:1459-1464`). The prefix is
  `VERP_PREFIX = "sl"` (`app/config.py:500`); the payload is a base32-encoded JSON list
  `[verp_type, object_id, minutes_since_VERP_TIME_START]`; the signature is a base32-encoded, 8-byte
  truncated `HMAC-sha3-224` over that payload.
- **(b) Inbound decode:** `get_verp_info_from_email()` (`app/email_utils.py:1467`) base32-decodes the
  payload, recomputes and compares the HMAC (`app/email_utils.py:1490`), then applies a
  **future-timestamp upper-bound guard** (`app/email_utils.py:1496`) that returns `None` only when the
  address's embedded timestamp is **more than `VERP_MESSAGE_LIFETIME` (5 days, `app/config.py:499`) in
  the future** of decode time — there is **no lower bound, so a stale/old address is still accepted** —
  and otherwise returns `(VerpType, EmailLog.id)`. A legacy `bounce+<id>+@` form is decoded by
  `parse_id_from_bounce()` (`app/email_utils.py:1258`).
- **(c) State changes (both directions):** a `Bounce` row is created; the full report + original
  message are archived to S3; a `RefusedEmail` row is created; and the `EmailLog` gets
  `bounced=True`, `refused_email_id`, and `bounced_mailbox_id` set, alongside a user `Notification`.
- **(d) Direction differences:** the forward phase (`handle_bounce_forward_phase()`,
  `email_handler.py:1432`) returns SMTP `E211`, records `Bounce.email = mailbox.email`, alerts the
  **user's** address, and may auto-disable the alias; the reply phase
  (`handle_bounce_reply_phase()`, `email_handler.py:1595`) returns `E212`, records
  `Bounce.email = contact.website_email`, alerts the **mailbox** address, and never disables the
  alias. The direction is chosen inside `handle_bounce()` (`email_handler.py:1851`) from
  `EmailLog.is_reply`.

> **[CORRECTION vs AAP]** The bounce handlers live in the repository **root** `email_handler.py`,
> **not** `app/email_handler.py` (no such file exists). All citations below use root
> `email_handler.py`.

## Evidence

### (a) Exact format — `generate_verp_email()` (`app/email_utils.py:1438`)

The construction (`app/email_utils.py:1446-1464`) builds the JSON payload
`[verp_type.value, object_id or 0, int((time.time() - VERP_TIME_START) / 60)]`
(`app/email_utils.py:1446-1450`), signs it with `hmac.new(secret, payload, "sha3-224").digest()[:8]`
(`app/email_utils.py:1454-1456`), base32-encodes both with the `=` padding stripped
(`app/email_utils.py:1457-1458`), and returns:

```python
return "{}.{}.{}@{}".format(                       # app/email_utils.py:1459-1464
    config.VERP_PREFIX,                            # "sl"  (app/config.py:500)
    encoded_payload,
    encoded_signature,
    sender_domain or config.EMAIL_DOMAIN,          # "sl.local" in this env
).lower()
```

Runtime constants confirmed under the Flask app context:

```
VERP_PREFIX='sl'  EMAIL_DOMAIN='sl.local'  VERP_EMAIL_SECRET len=36
VERP_MESSAGE_LIFETIME=432000 (5 days)  VERP_TIME_START=1640995200  VERP_HMAC_ALGO='sha3-224'
VerpType: bounce_forward=0, bounce_reply=1, transactional=2   (app/models.py:247-250)
```

Calling `generate_verp_email()` for all three `VerpType` values with `object_id=100` prints the
**exact** addresses (unedited), plus the decoded payload list showing the `[type, id, minutes]`
structure:

```bash
$ pytest tests/blitzy_adhoc_test_q3_verp.py -s   # exercises generate_verp_email() directly
```

```
--- (a) EXACT VERP addresses (object_id=100) ---
  bounce_forward: EXACT ADDRESS: sl.lmycyibrgaycyibsgm3tiojtgjoq.ehplnr2q6k4fq@sl.local
     decoded payload list [verp_type, object_id, minutes_since_2022-01-01] = [0, 100, 2374932]
  bounce_reply: EXACT ADDRESS: sl.lmysyibrgaycyibsgm3tiojtgjoq.fohccdjhc652u@sl.local
     decoded payload list [verp_type, object_id, minutes_since_2022-01-01] = [1, 100, 2374932]
  transactional: EXACT ADDRESS: sl.lmzcyibrgaycyibsgm3tiojtgjoq.awbcfv2fi6qoc@sl.local
     decoded payload list [verp_type, object_id, minutes_since_2022-01-01] = [2, 100, 2374932]
```

The three addresses share prefix `sl.`, the same trailing `@sl.local`, and the same time term
(`2374932` minutes since `VERP_TIME_START`); they differ only in the leading payload byte
(`verp_type` 0/1/2) and the resulting signature — exactly matching the
`sl.<payload>.<sig>@<domain>` format.

### (b) Inbound decode — `get_verp_info_from_email()` (`app/email_utils.py:1467`)

Round-tripping freshly generated addresses (this time with `object_id=4242`) recovers the exact
`(VerpType, id)` tuple; a tampered signature and a random (non-3-field) address both return `None`.
Complete, unedited output from `tests/blitzy_adhoc_test_q3_verp.py`:

```
  bounce_forward: address=sl.lmycyibugi2delbagiztonbzgmzf2.vpi7gmwftjo4g@sl.local
     -> get_verp_info_from_email() = (<VerpType.bounce_forward: 0>, 4242)  (match: type=True id=True)
  bounce_reply: address=sl.lmysyibugi2delbagiztonbzgmzf2.x3opfh7p25h54@sl.local
     -> get_verp_info_from_email() = (<VerpType.bounce_reply: 1>, 4242)  (match: type=True id=True)
  transactional: address=sl.lmzcyibugi2delbagiztonbzgmzf2.usoa5nowc5d2u@sl.local
     -> get_verp_info_from_email() = (<VerpType.transactional: 2>, 4242)  (match: type=True id=True)
  tampered signature: sl.lmycyibugi2delbagiztonbzgmzf2.vpi7gmwftjo4a@sl.local
     -> get_verp_info_from_email() = None  (expect None; HMAC mismatch at app/email_utils.py:1490)
  random@sl.local -> None  (expect None; not 3 dotted fields)
```

This confirms the HMAC comparison at `app/email_utils.py:1490` (a tampered signature yields `None`)
and the structural check requiring exactly three dotted fields prefixed by `sl`
(`app/email_utils.py:1476`).

**[CORRECTION — the guard at `app/email_utils.py:1496` is a *future-timestamp upper bound*, not a
staleness/expiry gate.]** The exact source line is:

```python
if data[2] > (time.time() + config.VERP_MESSAGE_LIFETIME - VERP_TIME_START) / 60:   # app/email_utils.py:1496
    return None
```

Generation embeds `data[2] = int((time.time() - VERP_TIME_START) / 60)` (`app/email_utils.py:1449`),
so substituting and cancelling `VERP_TIME_START` reduces the guard to *reject iff*
`t_generated > t_now + VERP_MESSAGE_LIFETIME` — it rejects only addresses whose embedded timestamp
claims to be **more than 5 days in the future** of decode time. There is **no lower bound**, so an
arbitrarily **old** address is still accepted. Driving all four boundary cases through the real
generator + decoder (the `[TEST FIXTURE]` label marks where `time.time()` is mocked inside the real
`generate_verp_email()` only to place the *embedded* timestamp; decoding always runs at real "now"):

```
  guard: `if data[2] > (time.time() + VERP_MESSAGE_LIFETIME - VERP_TIME_START)/60: return None`
  => rejects payloads whose embedded minute-timestamp is MORE THAN 5 days in the FUTURE of decode time.
  decode-time threshold (max acceptable data[2]) now = 2382132 minutes
  [current/now] embedded data[2]=2374932 min  -> decode=(<VerpType.bounce_forward: 0>, 4242)  => ACCEPTED  (expected ACCEPTED)
  [stale: -365 days (far past)] embedded data[2]=1849332 min  -> decode=(<VerpType.bounce_forward: 0>, 4242)  => ACCEPTED  (expected ACCEPTED (NOT a staleness gate))
  [future: +4 days (within 5-day window)] embedded data[2]=2380692 min  -> decode=(<VerpType.bounce_forward: 0>, 4242)  => ACCEPTED  (expected ACCEPTED)
  [future: +6 days (beyond 5-day window)] embedded data[2]=2383572 min  -> decode=None  => REJECTED(None)  (expected REJECTED(None))
```

Observed exactly as predicted: the far-past (`-365 days`) address is **ACCEPTED** (no staleness
expiry), the two future cases split precisely at the 5-day window (`+4 days` ACCEPTED, `+6 days`
REJECTED as `None`), and the decode-time threshold is `2382132` minutes.
`VERP_MESSAGE_LIFETIME = 432000` (5 days, `app/config.py:499`) therefore bounds how far into the
*future* an embedded timestamp may be, **not** how old a bounce address may be.

**Legacy fallback — `parse_id_from_bounce()` (`app/email_utils.py:1258`).** The legacy `bounce+`
form encodes the id between two `+` characters; `parse_id_from_bounce()` recovers it via
`int(email_address[email_address.find("+") : email_address.rfind("+")])` (`app/email_utils.py:1259`)
— note the slice starts at the first `+`, and `int()` tolerates the leading `+` sign
(`int("+123") == 123`):

```
  legacy forward bounce address='bounce+123+@sl.local' -> parse_id_from_bounce()=123
  legacy forward bounce address='bounce+987654+@sl.local' -> parse_id_from_bounce()=987654
```

The full set of legacy/transactional prefixes is defined in `app/config.py`:
`BOUNCE_PREFIX = "bounce+"` (`app/config.py:100`), `BOUNCE_SUFFIX = "+@{EMAIL_DOMAIN}"`
(`app/config.py:101`), `BOUNCE_PREFIX_FOR_REPLY_PHASE = "bounce_reply"` (`app/config.py:108`), and
`TRANSACTIONAL_BOUNCE_PREFIX = "transactional+"` (`app/config.py:113`). The reply-phase and transactional legacy
forms are decoded by the same integer-extraction logic. `[inferred]` for the reply/transactional
legacy forms — only the modern signed form and the legacy forward form were exercised at runtime;
the prefixes above are cited from source.

### (c)+(d) Directional handling — driving `handle_bounce()` down both branches

`handle_bounce()` (`email_handler.py:1851`) dispatches by direction. The
forward-vs-reply divergence, reproduced at runtime:

| Aspect | Forward (`bounce_forward`) | Reply (`bounce_reply`) |
|---|---|---|
| Handler | `handle_bounce_forward_phase()` `email_handler.py:1432` | `handle_bounce_reply_phase()` `email_handler.py:1595` |
| `Bounce.email` | `mailbox.email` | `contact.website_email` |
| Alert recipient | `user.email` — `ALERT_BOUNCE_EMAIL = "bounce"` (`app/config.py:359`) | `mailbox.email` — `ALERT_BOUNCE_EMAIL_REPLY_PHASE = "bounce-when-reply"` (`app/config.py:361`) |
| Alias auto-disable | Yes, if `should_disable(alias)` (`app/email_utils.py:1166`) | No |
| SMTP status | `E211` | `E212` |
| Special branch | — | Auto-reply re-forward when NOT a true DSN (`email_handler.py:1876`) |

The bounce driver builds real `EmailLog` rows (forward: `is_reply=False`; reply: `is_reply=True`)
and calls `handle_bounce()`, capturing the state before/after:

```bash
$ pytest tests/blitzy_adhoc_test_q3_bounce.py -s   # drives handle_bounce() both directions
```

**Forward phase — fresh alias (`should_disable` False):**

```
[SETUP] alias='cougar_sultan009@sl.local' mailbox='user_v9xyg7mmuy@mailbox.test' contact.website_email='sender-c0744745@remote.example'
[get_phase] email_log.get_phase() -> 'forward'   (app/models.py:2143-2147)
[INPUT] bounce.eml content_type='multipart/report' walk_parts=7
[BEFORE]  bounced=False refused_email_id=None bounced_mailbox_id=None alias.enabled=True
2026-07-08 06:26:05,861 - SL - DEBUG - 104724 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/email_handler.py:1862" - handle_bounce() -  - handle bounce for <EmailLog 121>, phase=forward, contact=<Contact 15 sender-c0744745@remote.example 63>, alias=<Alias 63 cougar_sultan009@sl.local>
2026-07-08 06:26:05,864 - SL - DEBUG - 104724 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/email_handler.py:1456" - handle_bounce_forward_phase() -  - Handle forward bounce <Contact 15 sender-c0744745@remote.example 63> -> <Alias 63 cougar_sultan009@sl.local> -> <Mailbox 97 user_v9xyg7mmuy@mailbox.test>. <EmailLog 121>
2026-07-08 06:26:05,872 - SL - DEBUG - 104724 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/email_handler.py:1491" - handle_bounce_forward_phase() -  - Create refused email <Refused Email 7 refused-emails/b6405849-9751-4e90-9e80-52f9892a6afc.eml 2026-07-15T06:26:05.871743+00:00>
[RESULT]  handle_bounce -> '250 SL E211 Bounce Forward phase handled'   (expect E211='250 SL E211 Bounce Forward phase handled')
[AFTER]   bounced=True refused_email_id=7 bounced_mailbox_id=97 alias.enabled=True
[STATE Bounce] rows_for(mailbox.email) 0 -> 1; latest.email='user_v9xyg7mmuy@mailbox.test' info_len=517
[STATE S3] 2 upload call(s):
    path='refused-emails/full-b6405849-9751-4e90-9e80-52f9892a6afc.eml' filename='full-b6405849-9751-4e90-9e80-52f9892a6afc' bytes=3803
    path='refused-emails/b6405849-9751-4e90-9e80-52f9892a6afc.eml' filename='b6405849-9751-4e90-9e80-52f9892a6afc' bytes=1382
[STATE RefusedEmail] id=7 path='refused-emails/b6405849-9751-4e90-9e80-52f9892a6afc.eml' full_report_path='refused-emails/full-b6405849-9751-4e90-9e80-52f9892a6afc.eml' user_id=48
```

Forward phase returns **`E211`** and records the complete common state-change set:
`Bounce.email == mailbox.email` (`user_v9xyg7mmuy@mailbox.test`); **two S3 uploads** via
`s3.upload_email_from_bytesio()` (`app/s3.py:47`) — the full DSN report to
`refused-emails/full-<uuid>.eml` (`bytes=3803`) and the original message to
`refused-emails/<uuid>.eml` (`bytes=1382`, uploaded at `email_handler.py:1483-1485`); a `RefusedEmail`
row created with `path=file_path, full_report_path=full_report_path, user_id=user.id`
(`email_handler.py:1487-1489`, logged at `email_handler.py:1491`); and the `EmailLog` flags
`bounced=True / refused_email_id=7 / bounced_mailbox_id=97` (`email_handler.py:1493-1495`), alerting
the **user's** address. With `should_disable=(False, '')` the alias stays enabled
(`alias.enabled=True`).

**Forward phase — alias with >12 prior bounces (`should_disable` True) → alias auto-disabled:**

```
[BEFORE] alias.enabled=True prior_bounced_forward_logs=12 should_disable=(False, '')
2026-07-08 06:26:06,269 - SL - DEBUG - 104724 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/email_handler.py:1491" - handle_bounce_forward_phase() -  - Create refused email <Refused Email 8 refused-emails/f6e45738-2cd3-441a-986c-0f003fde938d.eml 2026-07-15T06:26:06.268662+00:00>
2026-07-08 06:26:06,277 - SL - WARNING - 104724 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/email_handler.py:1502" - handle_bounce_forward_phase() -  - Disable alias <Alias 65 faffed_bottom859@sl.local> because +12 bounces in the last 24h. [<Mailbox 98 user_jcfj9omfi2@mailbox.test>] <User 49 Test User user_jcfj9omfi2@mailbox.test>. Last contact <Contact 16 sender-19ca68a5@remote.example 65>
2026-07-08 06:26:06,277 - SL - INFO - 104724 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/app/alias_utils.py:553" - change_alias_status() -  - Changing alias <Alias 65 faffed_bottom859@sl.local> enabled to False
[RESULT] handle_bounce -> '250 SL E211 Bounce Forward phase handled' (expect E211)
[should_disable AFTER] -> (True, '+12 bounces in the last 24h')   (app/email_utils.py:1166; >12 branch :1190)
[AFTER] alias.enabled=False
[STATE Notification] count=1 titles=['faffed_bottom859@sl.local has been disabled due to multiple bounces']
```

The alias started with exactly `12` prior forward-phase bounce logs, so `should_disable` was
`(False, '')` *before* handling; the current bounce makes it the 13th, so after processing
`should_disable(alias)` (`app/email_utils.py:1166`) returns `(True, '+12 bounces in the last 24h')`
(the `nb_bounced_last_24h > 12` branch, `app/email_utils.py:1190`). The forward handler then disables
the alias via `change_alias_status()` (`app/alias_utils.py:553`), transitioning
`alias.enabled: True → False`, and creates one `Notification` with the "disabled due to multiple
bounces" title. SMTP status is still `E211`. (Requires `ALIAS_AUTOMATIC_DISABLE=true`, present in
`tests/test.env:62`.)

**Reply phase (multipart/report DSN with `MAIL FROM:<>`):**

```
[SETUP] alias='sirens_nimbus281@sl.local' mailbox='user_6fj3oviw07@mailbox.test' contact.website_email='sender-7bb79468@remote.example'
[get_phase] email_log.get_phase() -> 'reply'
[INPUT] content_type='multipart/report' envelope.mail_from='<>'  -> true-DSN branch
[BEFORE] bounced=False refused_email_id=None auto_replied=False alias.enabled=True
2026-07-08 06:26:06,595 - SL - DEBUG - 104724 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/email_handler.py:1862" - handle_bounce() -  - handle bounce for <EmailLog 135>, phase=reply, contact=<Contact 17 sender-7bb79468@remote.example 67>, alias=<Alias 67 sirens_nimbus281@sl.local>
2026-07-08 06:26:06,602 - SL - DEBUG - 104724 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/email_handler.py:1640" - handle_bounce_reply_phase() -  - Create refused email <Refused Email 9 refused-emails/a7a60839-5433-460d-a4f2-79365ba4e28b.eml 2026-07-15T06:26:06.601989+00:00>
[RESULT] handle_bounce -> '250 SL E212 Bounce Reply phase handled'   (expect E212='250 SL E212 Bounce Reply phase handled')
[STATE Bounce] latest.email='sender-7bb79468@remote.example'   (== contact.website_email 'sender-7bb79468@remote.example')
[STATE S3] 2 upload call(s):
    path='refused-emails/full-a7a60839-5433-460d-a4f2-79365ba4e28b.eml' filename='a7a60839-5433-460d-a4f2-79365ba4e28b' bytes=3803
    path='refused-emails/a7a60839-5433-460d-a4f2-79365ba4e28b.eml' filename='a7a60839-5433-460d-a4f2-79365ba4e28b' bytes=1382
[STATE RefusedEmail] id=9 path='refused-emails/a7a60839-5433-460d-a4f2-79365ba4e28b.eml' full_report_path='refused-emails/full-a7a60839-5433-460d-a4f2-79365ba4e28b.eml' user_id=50
[AFTER] bounced=True bounced_mailbox_id=99 alias.enabled=True
```

Reply phase returns **`E212`** and records the same common state set — a `Bounce` row, two S3 uploads
(`bytes=3803` full report + `bytes=1382` original), a `RefusedEmail`
(`email_handler.py:1640`), and `EmailLog.bounced=True / bounced_mailbox_id=99` — but with three
directional divergences from the forward phase: (1) `Bounce.email == contact.website_email`
(`sender-7bb79468@remote.example`), **not** `mailbox.email`; (2) the full-report S3 upload's
`filename` argument is the bare UUID `a7a60839-5433-460d-a4f2-79365ba4e28b` (**no `full-` prefix**),
whereas the forward phase
passes `full-<uuid>` — though the stored object `path` and `RefusedEmail.full_report_path` still carry
the `refused-emails/full-<uuid>.eml` form in both directions; and (3) the alias is **never
auto-disabled** — `alias.enabled` stays `True`. The alert goes to the **mailbox** address.

**Reply-phase auto-reply RE-FORWARD branch (message that is NOT a real DSN):** when the inbound
message's `content_type != "multipart/report"` **or** `envelope.mail_from != "<>"`
(`email_handler.py:1876`), it is treated as an auto-reply and re-forwarded instead of processed as a
bounce:

```
[BEFORE] auto_replied=False
[INPUT] content_type='text/plain' envelope.mail_from='sender-02a17a5f@remote.example'
        branch condition: content_type != 'multipart/report' OR mail_from != '<>'  -> True
2026-07-08 06:26:07,001 - SL - DEBUG - 104724 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/email_handler.py:1862" - handle_bounce() -  - handle bounce for <EmailLog 136>, phase=reply, contact=<Contact 18 sender-02a17a5f@remote.example 69>, alias=<Alias 69 wanton_quaint452@sl.local>
[DURING/AFTER] email_log.auto_replied -> True   (set at app/email_handler.py:1887)
[RESULT] handle_bounce -> '250 Message accepted for delivery'   err=None
[RE-FORWARD] envelope.rcpt_tos -> ['wanton_quaint452@sl.local']   (replaced with alias.email 'wanton_quaint452@sl.local')
[STORED MAIL] 1 message(s) captured by mail_sender store-mode
```

The branch condition `content_type != "multipart/report" or envelope.mail_from != "<>"`
(`email_handler.py:1876`) is `True` here (a `text/plain` message from a real sender), so the message
is treated as an auto-reply rather than a DSN: `EmailLog.auto_replied` is set `True`
(`email_handler.py:1887`), the `To` header is rewritten to `alias.email`
(`add_or_replace_header(msg, "To", alias.email)`, `email_handler.py:1891`), the envelope recipients
are replaced with `[alias.email]` (`email_handler.py:1892` — observed as
`envelope.rcpt_tos -> ['wanton_quaint452@sl.local']`), and the message is re-forwarded via
`handle_forward()` (`email_handler.py:1899`); the return is `250 Message accepted for delivery` —
**not** a bounce status. (The `[DURING/AFTER]` annotation above is the disposable harness's own label;
it writes the module path loosely as `app/email_handler.py`, but the authoritative module is the
repository-root `email_handler.py`, as the absolute path in every framework log line confirms —
`<repo>/email_handler.py:NNNN`.)

### Edge statuses — `E512` (unknown email log) and `E510` (inactive user)

```
2026-07-08 06:26:07,039 - SL - WARNING - 104724 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/email_handler.py:1857" - handle_bounce() -  - No such email log
[RESULT] handle_bounce(None) -> '550 SL E512 No such email log'   (expect E512='550 SL E512 No such email log')
[SETUP] user.delete_on=2026-07-09T06:26:07.322750+00:00 user.is_active()=False
2026-07-08 06:26:07,329 - SL - DEBUG - 104724 - "/tmp/blitzy/app/blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928_a9d9a1/email_handler.py:1870" - handle_bounce() -  - User <User 52 Test User user_718xyzdp9p@mailbox.test> is not active
[RESULT] handle_bounce -> '550 SL E510 so such user'   (expect E510='550 SL E510 so such user')
```

A missing `EmailLog` logs "No such email log" (`email_handler.py:1857`) and yields `550 SL E512`
(`email_handler.py:1858`); an inactive user (`User.is_active()` returns `False` when `delete_on` is
set to a future time, `app/models.py:766`) logs that the user `is not active` (`email_handler.py:1870`)
and yields `550 SL E510` (`email_handler.py:1871`).

### Direction discriminator — `EmailLog.get_phase()` (`app/models.py:2143-2147`)

```
EmailLog(is_reply=False).get_phase()='forward'; EmailLog(is_reply=True).get_phase()='reply'
```

`get_phase()` returns `"reply"` when `self.is_reply` is set, else `"forward"`
(`app/models.py:2143-2147`) — the value `handle_bounce()` keys on to choose the phase handler.

### Two-run stability (Q3)

Both Q3 harnesses were executed **twice**. `tests/blitzy_adhoc_test_q3_verp.py`
(`/tmp/blitzy_evidence/q3verp.run1.txt` and `q3verp.run2.txt`, each `1 passed, 18 warnings`) is
**byte-identical across the two runs in its entire evidence region** — the two runs fell within the
same wall-clock minute, so even the minute-quantized VERP time term (`2374932`), and therefore the
exact addresses, matched. (That time term is `int((time.time() - VERP_TIME_START) / 60)`,
`app/email_utils.py:1449`, so it advances by 1 every minute; the *format* `sl.<payload>.<sig>@<domain>`
and the decode/boundary semantics are invariant regardless of the minute.)
`tests/blitzy_adhoc_test_q3_bounce.py` (`q3bounce.run1.txt` / `q3bounce.run2.txt`, each
`7 passed, 18 warnings`) is **behaviorally identical** across the two runs; the only differences are
inherently non-deterministic identifiers — auto-increment primary keys, per-message UUID filenames,
randomly generated alias/mailbox/contact names, and wall-clock timestamps.

| Observable | Run 1 | Run 2 | Stable? |
|---|---|---|---|
| VERP format | `sl.<b32-payload>.<b32-sig>@<domain>` | `sl.<b32-payload>.<b32-sig>@<domain>` | ✓ |
| Decode round-trip recovers `(VerpType, id)` | yes (all 3 types) | yes (all 3 types) | ✓ |
| Tampered signature / random address | `None` / `None` | `None` / `None` | ✓ |
| Boundary: current / −365d / +4d / +6d | ACCEPT / ACCEPT / ACCEPT / REJECT | ACCEPT / ACCEPT / ACCEPT / REJECT | ✓ |
| Forward SMTP status | `E211` | `E211` | ✓ |
| Reply SMTP status | `E212` | `E212` | ✓ |
| `Bounce.email` forward / reply | `mailbox.email` / `contact.website_email` | `mailbox.email` / `contact.website_email` | ✓ |
| S3 uploads per bounce (full-report / original bytes) | 2 (`3803` / `1382`) | 2 (`3803` / `1382`) | ✓ |
| `should_disable` at 13th forward bounce | `(True, '+12 bounces in the last 24h')` | `(True, '+12 bounces in the last 24h')` | ✓ |
| Forward auto-disables / reply never disables | yes / no | yes / no | ✓ |
| Auto-reply re-forward (non-DSN) return | `250 Message accepted for delivery` | `250 Message accepted for delivery` | ✓ |
| Edge: unknown log / inactive user | `E512` / `E510` | `E512` / `E510` | ✓ |

### Background standard (framing only)

A Delivery Status Notification (DSN) per **RFC 3464 / RFC 3462** is a MIME message whose top-level
content type is `multipart/report`, and genuine non-delivery reports carry an empty envelope return
path (`MAIL FROM:<>`); auto-replies (RFC 3834) do not. This is exactly the discriminator at
`email_handler.py:1876`. **VERP** (Variable Envelope Return Path) is the technique of embedding a
unique token in the envelope return path so an inbound failure can be attributed to the exact
message that caused it; SimpleLogin implements a *signed* variant (HMAC-`sha3-224`). The repository
code remains the authoritative source for every value above.

### Canonical corroboration and an honest environment finding

```bash
$ pytest tests/test_email_utils.py -k "verp or bounce or parse_id" -p no:cacheprovider --no-cov -q
14 passed
```

The 14 passing cases include `test_generate_verp_email` (`tests/test_email_utils.py:849`, parametrized
over ids 1/10/100/1000/10000), `test_generate_verp_email_forward_reply_phase`
(`tests/test_email_utils.py:858`), and `test_parse_id_from_bounce`, corroborating the format and
decode logic captured above.

> **[NON-CANONICAL ENV]** `tests/test_email_handler.py` **ERRORS at collection** in this environment:
> `AttributeError: module 're2' has no attribute 'DOTALL'`, raised from `app/spamassassin_utils.py:13`.
> This is because the environment substitutes **google-re2** for the canonical **pyre2** (the latter
> exposes `re2.DOTALL`; the former does not). To exercise the bounce flow without touching any
> repository source, the disposable harness `tests/blitzy_adhoc_test_q3_bounce.py` injects a clearly
> labeled `[NON-CANONICAL ENV SHIM]` (defining the missing `re2` flag constants and delegating
> integer-flag `compile()` to the stdlib `re`) **before** importing `email_handler`. The shim affects
> only the SpamAssassin regex path, which is unrelated to the VERP/bounce logic under test; no
> repository file was modified.


---

# Coverage & Cleanup

## Coverage pass — every named sub-part and sibling variant

**Question 1 — mailbox verification brute-force**

| Sub-part / variant | Answer | `file:line` | Observed |
|---|---|---|---|
| (a) attempt limit | `MAX_ACTIVATION_TRIES = 3` | `app/mailbox_utils.py:43` | ✅ tries 0→1→2→3 then deletion |
| (b) tracking | `MailboxActivation.tries` (counts **up** from 0) | `app/models.py:2835` (class `:2828`) | ✅ printed before/after each attempt |
| (c) terminal prevention | record deleted by `clear_activation_codes_for_mailbox()` | `app/mailbox_utils.py:159` | ✅ row gone; next attempt "no activation" |
| guard order (cap→expiry→compare) | cap `:195`, expiry `:199`, compare+`tries+=1` `:209` | `app/mailbox_utils.py` | ✅ CannotVerifyError texts captured |
| real HTTP entry point | `GET /dashboard/mailbox_verify` (no limiter) | `app/dashboard/views/mailbox.py:120-122` | ✅ HTTP 200 + flash messages |
| signed-link sibling | `verify_with_signed_secret()` `max_age=900` | `app/dashboard/views/mailbox.py:138-142` | ✅ SignatureExpired + shadowing-`request` 500 captured |
| `AccountActivation` counterpart | `default=3`, **decrements** to 0 | `app/models.py:2838`; `app/api/views/auth.py:178,181` | ✅ 400/400/410 captured |
| rate-limit contrast | verify route has none; `mailbox_detail` `20/minute` **POST-only** | `app/dashboard/views/mailbox_detail.py:37` | ✅ limiter registry present=False/True |
| do-not-conflate | 30-char registration `ActivationCode` is a different mechanism | `app/auth/views/register.py` | out of scope (noted) |
| harness | `tests/test_mailbox_utils.py` | — | ✅ 23 passed |

**Question 2 — Job lifecycle & retry**

| Sub-part / variant | Answer | `file:line` | Observed |
|---|---|---|---|
| lifecycle | `ready(0)→taken(1)→done(2)` | `app/models.py:253-257`; `job_runner.py:337-345` | ✅ both success & failure |
| (a) retry behavior | implicit, time-based; **no try/except** @`:342` | `job_runner.py:342` | ✅ crash exit code 1 |
| retry window | `JOB_TAKEN_RETRY_WAIT_MINS = 30` | `app/config.py:565` | ✅ eligible only past 30 min |
| attempt cap | `JOB_MAX_ATTEMPTS = 5` | `app/config.py:564` | ✅ abandoned at `attempts>=5`, boundary `4<5` |
| (b) failure state | stuck in `taken`, `attempts++`, `taken_at` set, `done` never reached | `job_runner.py:337-342` | ✅ `state=1 attempts=1` after crash |
| `JobState.error` refinement | never written by runner; only in `tasks/cleanup_old_jobs.py:15` | `job_runner.py` (grep NONE) | ✅ grep captured |
| success cross-product | unknown name → `done(2)` | `job_runner.py:303-304` | ✅ `state=2` |
| cron/yacron contrast | separate subsystem, not the `Job` table | `crontab.yml`, `cron.py` | ✅ schedules listed |
| harness | `tests/jobs/test_job_runner.py::test_get_jobs_to_run` | `:8` | ✅ 1 passed |

**Question 3 — VERP bounce format & directional handling**

| Sub-part / variant | Answer | `file:line` | Observed |
|---|---|---|---|
| (a) exact format | `sl.<b32-payload>.<b32-hmac>@<domain>` lowercased | `app/email_utils.py:1459-1464` | ✅ 3 exact addresses printed |
| payload / signature | `[type,id,mins]`; HMAC-`sha3-224` `[:8]` | `app/email_utils.py:1446-1458` | ✅ decoded lists shown |
| (b) decode | `get_verp_info_from_email()` → `(VerpType,id)` | `app/email_utils.py:1467` | ✅ round-trip match; tamper→None |
| decode guard (NOT a staleness gate) | **future-timestamp upper bound only** — rejects timestamps > `VERP_MESSAGE_LIFETIME` (5 days, `=432000`s) in the **future** of decode time; **no lower bound**, so stale/old addresses are accepted | `app/config.py:499`; `app/email_utils.py:1496` (embed `:1449`) | ✅ 4-case boundary: current / −365d / +4d **ACCEPTED**, +6d **REJECTED(None)**; threshold=2382132 min |
| legacy fallback | `parse_id_from_bounce()` `bounce+<id>+@` | `app/email_utils.py:1258`; `app/config.py:100-113` | ✅ 123 / 987654 (reply/txn legacy `[inferred]`) |
| (c) state changes | `Bounce` + S3 archive + `RefusedEmail` + `EmailLog` flags + `Notification` | `email_handler.py:1432-1683` | ✅ flags & Bounce row captured |
| (d) forward | `E211`, `Bounce.email=mailbox.email`, alert `user.email`, auto-disable | `email_handler.py:1432`; `app/config.py:359` | ✅ both should_disable F/T |
| (d) reply | `E212`, `Bounce.email=contact.website_email`, alert `mailbox.email`, no disable | `email_handler.py:1595`; `app/config.py:361` | ✅ enabled stays True |
| auto-reply re-forward | non-DSN → `auto_replied=True`, re-forward, `250` | `email_handler.py:1876,1887,1899` | ✅ captured |
| edge `E512`/`E510` | unknown log / inactive user | `email_handler.py:1856-1858,1869-1871` | ✅ both captured |
| `get_phase()` | `is_reply`→`reply` else `forward` | `app/models.py:2143-2147` | ✅ both printed |
| transactional variant | `VerpType.transactional=2` | `app/models.py:249` | ✅ address + round-trip |
| harness | `tests/test_email_utils.py` (verp/bounce/parse_id) | `:849,:858` | ✅ 14 passed (`test_email_handler.py` env-blocked, documented) |

Every behavioral claim above is paired with **both** a `file:line` citation **and** captured runtime
output; statements derived from reading rather than observation are labeled `[inferred]`, and every
environment accommodation is labeled `[NON-CANONICAL ENV]`.

## Cleanup & read-only guarantee

All runtime observation was performed through the project's own `pytest` harnesses and through
temporary, disposable scripts. The four temporary observation scripts —
`tests/blitzy_adhoc_test_q1.py`, `tests/blitzy_adhoc_test_q2_helper.py`,
`tests/blitzy_adhoc_test_q3_verp.py`, and `tests/blitzy_adhoc_test_q3_bounce.py` — together with
their `tests/__pycache__/` byte-code, the Redis `dump.rdb` file, and the scratch `.eml` objects that
the `LOCAL_FILE_UPLOAD` path wrote under the git-ignored `static/upload/refused-emails/` directory
(`.gitignore:11`), were all deleted after evidence capture. The scratch PostgreSQL rows created
during the investigation are transient test-DB state outside the repository tree. No repository
source file was modified, added to, or deleted — the **only** net-new artifact is this document.

The read-only guarantee is proven by diffing the working tree against the pristine **source
baseline** commit `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c` — a stable anchor regardless of any
platform checkpoint commit layered on top. Exactly one file (this document) differs, and no source
file appears. Captured after deleting every temporary artifact, immediately before committing the
deliverable:

```bash
$ git diff 2cd6ee777f8c2d3531559588bcfb18627ffb5d2c --stat
 blitzy/documentation/app_2cd6ee777f8c.md | 1195 ++++++++++++++++++++++++++++++
 1 file changed, 1195 insertions(+)
$ git diff 2cd6ee777f8c2d3531559588bcfb18627ffb5d2c --name-only
blitzy/documentation/app_2cd6ee777f8c.md
$ git status --porcelain -uall
 M blitzy/documentation/app_2cd6ee777f8c.md
```

The single `git status` entry is this deliverable; it shows ` M` (modified) rather than `??`
(untracked) because a platform checkpoint commit already contains an earlier revision of it, and
this line reflects the current pass's edits to that same one file. The diff against the source
baseline lists that identical single file with **zero** source-file changes — confirming the
read-only constraint was honored end to end.

