# SimpleLogin Runtime Behavior Investigation — `app_2cd6ee777f8c`

**Repository:** SimpleLogin (`app/`) &nbsp;•&nbsp; **Branch:** `app_2cd6ee777f8c` &nbsp;•&nbsp; **Method:** runtime observation inside the canonical Docker container.

This document answers three runtime-behavior questions about the SimpleLogin codebase. **Every** behavioral claim below was produced by *running the real code path through its canonical entry point* and capturing the actual, unedited output — not by reading source alone. Each claim carries: (1) the exact command, (2) the complete unedited output, (3) a `file:line` reference, and (4) an **[OBSERVED]** or **[INFERRED]** label.

> **Path note (verified):** the SMTP inbound handler is at the **repository root**: `email_handler.py`. There is **no** `app/email_handler.py`. All Q3 handler references below point at the root file.

## The three questions (verbatim)

1. **Q1 —** When a mailbox verification code is submitted incorrectly several times in succession, what observable state changes and enforcement mechanisms does the running system apply? Trace (a) what limit(s) exist, (b) how failed attempts are tracked (which counter/state field, and its progression), and (c) what condition ultimately prevents further submissions.

2. **Q2 —** For background tasks that are scheduled and later picked up for execution, trace the complete lifecycle from initial creation through final completion. When a task encounters an error during execution, determine (a) what recovery or retry behavior is applied, and (b) what observable state reflects that failure.

3. **Q3 —** During email forwarding through an alias, what is the exact **format** of the special address generated to handle delivery failures (the bounce / return-path / VERP address)? When a failure notification arrives at this address, trace (a) how the system identifies the original email, (b) what state changes are recorded, and (c) how the handling behavior **differs depending on the direction** of the original message (forward versus reply).

---

## Environment (canonical container)

All observations were produced inside the canonical container `sl_canon` (image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`), running **Python 3.10**, Postgres, and Redis. The deliverable itself is written to the host checkout; the *observed values* all come from the container. The local host checkout runs a different Python with no app dependencies and is **not** canonical.

**Command — container identity, Python version, git HEAD:**

```bash
docker exec sl_canon bash -lc 'cd /app && /app/venv/bin/python --version && git rev-parse HEAD && cat .git/HEAD'
```

```
Python 3.10.18
2cd6ee777f8c2d3531559588bcfb18627ffb5d2c
2cd6ee777f8c2d3531559588bcfb18627ffb5d2c
```

The HEAD commit `2cd6ee777f8c…` matches the source branch name `app_2cd6ee777f8c`. **[OBSERVED]**

**Command — Postgres + Redis reachable, schema present:**

```bash
docker exec sl_canon bash -lc '
service postgresql status | head -1; service redis-server status | head -1
export CONFIG=tests/test.env PYTHONPATH=/app DB_URI="postgresql://test:test@localhost:5432/test"
cd /app && /app/venv/bin/python - <<PY
import psycopg2
c=psycopg2.connect("postgresql://test:test@localhost:5432/test"); cur=c.cursor()
cur.execute("select count(*) from information_schema.tables where table_schema=%s",("public",))
print("public tables:", cur.fetchone()[0])
cur.execute("select version()"); print(cur.fetchone()[0].split(",")[0])
PY
redis-cli ping'
```

```
15/main (port 5432): online
redis-server is running.
public tables: 77
PostgreSQL 15.13 (Debian 15.13-0+deb12u1) on x86_64-pc-linux-gnu
PONG
```

Postgres 15.13 is online with the full 77-table schema; Redis answers `PONG`. **[OBSERVED]**

**Command — the three subsystems import cleanly and config constants resolve at runtime:**

```bash
docker exec sl_canon bash -lc '
export CONFIG=tests/test.env PYTHONPATH=/app DB_URI="postgresql://test:test@localhost:5432/test"
cd /app && /app/venv/bin/python - <<PY
import app.mailbox_utils, job_runner, app.email_utils, email_handler
from app import config
from app.mailbox_utils import MAX_ACTIVATION_TRIES
from app.email_utils import VERP_HMAC_ALGO, VERP_TIME_START
print("imports OK: app.mailbox_utils, job_runner, app.email_utils, email_handler")
print("MAX_ACTIVATION_TRIES =", MAX_ACTIVATION_TRIES)
print("JOB_MAX_ATTEMPTS =", config.JOB_MAX_ATTEMPTS)
print("JOB_TAKEN_RETRY_WAIT_MINS =", config.JOB_TAKEN_RETRY_WAIT_MINS)
print("VERP_PREFIX =", repr(config.VERP_PREFIX))
print("EMAIL_DOMAIN =", repr(config.EMAIL_DOMAIN))
print("VERP_HMAC_ALGO =", repr(VERP_HMAC_ALGO))
print("VERP_TIME_START =", VERP_TIME_START)
print("VERP_MESSAGE_LIFETIME =", config.VERP_MESSAGE_LIFETIME)
PY'
```

```
imports OK: app.mailbox_utils, job_runner, app.email_utils, email_handler
MAX_ACTIVATION_TRIES = 3
JOB_MAX_ATTEMPTS = 5
JOB_TAKEN_RETRY_WAIT_MINS = 30
VERP_PREFIX = 'sl'
EMAIL_DOMAIN = 'sl.local'
VERP_HMAC_ALGO = 'sha3-224'
VERP_TIME_START = 1640995200
VERP_MESSAGE_LIFETIME = 432000
```

Every constant the three answers depend on is confirmed from the running process: `MAX_ACTIVATION_TRIES = 3`, `JOB_MAX_ATTEMPTS = 5`, `JOB_TAKEN_RETRY_WAIT_MINS = 30`, `VERP_PREFIX = 'sl'`, `EMAIL_DOMAIN = 'sl.local'`, `VERP_HMAC_ALGO = 'sha3-224'`, `VERP_TIME_START = 1640995200`, `VERP_MESSAGE_LIFETIME = 432000` (= 5 days). **[OBSERVED]**

> **Canonical run environment** used for every script and test below (log lines of the form `… - SL - DEBUG/INFO - …` are the app's own logger and are elided from some captures for readability using `grep -v`, never altering the observed values):
>
> ```bash
> export CONFIG=tests/test.env PYTHONPATH=/app DB_URI="postgresql://test:test@localhost:5432/test"
> # all Python invoked as /app/venv/bin/python (the venv holds the pinned deps)
> ```
> `tests/test.env` supplies `EMAIL_DOMAIN=sl.local`, `DISABLE_RATE_LIMIT=1`, `NOT_SEND_EMAIL`, `LOCAL_FILE_UPLOAD=1`, `MEM_STORE_URI=redis://localhost`. Where a value depends on configuration (e.g. `VERP_PREFIX`, `EMAIL_DOMAIN`), the value reported is the one produced by this default test configuration; `example.env` documents the production-style defaults and is noted where relevant.

---

## Q1 — Mailbox verification-code enforcement under repeated incorrect attempts

### Direct answer

The limit is **`MAX_ACTIVATION_TRIES = 3`** [`app/mailbox_utils.py:43`]. Failed attempts are tracked by the integer column **`MailboxActivation.tries`** (default `0`) [`app/models.py:2835`] on the latest activation row for the mailbox. The function that performs all enforcement is **`verify_mailbox_code(user, mailbox_id, code)`** [`app/mailbox_utils.py:166`], which the canonical HTTP route `GET /mailbox_verify` calls directly [`app/dashboard/views/mailbox.py:129`]. On each wrong code the counter is incremented and committed, then `CannotVerifyError("Invalid activation code")` is raised [`app/mailbox_utils.py:209-211`]; the observed progression is **`tries` 0 → 1 → 2 → 3**. The condition that ultimately prevents further submissions is the guard at the **top** of the function: once `activation.tries >= MAX_ACTIVATION_TRIES`, the code invalidates **all** activation rows for the mailbox via `clear_activation_codes_for_mailbox` [`app/mailbox_utils.py:197`] and raises `CannotVerifyError("Invalid activation code. Please request another code.")` [`app/mailbox_utils.py:198`]. After that, a further submission finds no activation and raises `MailboxError("Invalid code")` [`app/mailbox_utils.py:194`]. Enforcement is thus the **per-activation counter plus code deletion — not an HTTP 429**: `/mailbox_verify` carries no rate-limiter decorator (unlike the `/mailbox` POST route).

### Evidence — driving the real `verify_mailbox_code` through the ORM (primary + all branches)

**Command:**

```bash
docker exec sl_canon bash -lc '
export CONFIG=tests/test.env PYTHONPATH=/app DB_URI="postgresql://test:test@localhost:5432/test"
cd /app && /app/venv/bin/python /tmp/obs/obs_q1.py'
```

The scratch script `obs_q1.py` pushes a real app context (`from server import create_app`), creates a real user and a real unverified mailbox via `mailbox_utils.create_mailbox`, and then calls the **exact** function the route calls — `mailbox_utils.verify_mailbox_code(user, mailbox_id, code)` — reading `MailboxActivation.tries` straight from the DB after each call. **Complete unedited output** (the app's `SL - INFO/DEBUG` logger lines are shown exactly as emitted; they include the `file:line` of each branch):

```
MAX_ACTIVATION_TRIES = 3

=== SCENARIO A: repeated WRONG code -> tries progression + lockout ===
2026-07-13 17:08:20,978 - SL - INFO - 3234 - "/app/app/mailbox_utils.py:88" - create_mailbox() -  - User <User 467 Test User user_x598yeysaq@mailbox.test> has created mailbox with lyjtbhmojaudhpahskfx@lyjtbhmojaudhpahskfx.com
mailbox_id = 596
correct code = 'l_sbjPaFrlS3ETibGJXEIg'
BEFORE any submission: tries = 0
2026-07-13 17:08:21,006 - SL - INFO - 3234 - "/app/app/mailbox_utils.py:206" - verify_mailbox_code() -  - User <User 467 Test User user_x598yeysaq@mailbox.test> failed to verify mailbox 596 because code does not match
submission 1: CannotVerifyError(msg='Invalid activation code') -> tries now = 1
2026-07-13 17:08:21,011 - SL - INFO - 3234 - "/app/app/mailbox_utils.py:206" - verify_mailbox_code() -  - User <User 467 Test User user_x598yeysaq@mailbox.test> failed to verify mailbox 596 because code does not match
submission 2: CannotVerifyError(msg='Invalid activation code') -> tries now = 2
2026-07-13 17:08:21,016 - SL - INFO - 3234 - "/app/app/mailbox_utils.py:206" - verify_mailbox_code() -  - User <User 467 Test User user_x598yeysaq@mailbox.test> failed to verify mailbox 596 because code does not match
submission 3: CannotVerifyError(msg='Invalid activation code') -> tries now = 3
--- 4th submission (tries>=3 triggers lockout) ---
2026-07-13 17:08:21,021 - SL - INFO - 3234 - "/app/app/mailbox_utils.py:196" - verify_mailbox_code() -  - User <User 467 Test User user_x598yeysaq@mailbox.test> failed to verify mailbox 596 more than 3 times
submission 4: CannotVerifyError(msg='Invalid activation code. Please request another code.')
activation row after lockout: None
activation row count after lockout: 0
--- 5th submission (no activation -> MailboxError) ---
2026-07-13 17:08:21,029 - SL - INFO - 3234 - "/app/app/mailbox_utils.py:191" - verify_mailbox_code() -  - User <User 467 Test User user_x598yeysaq@mailbox.test> failed to verify mailbox 596 because there is no activation
submission 5: MailboxError(msg='Invalid code')

=== SCENARIO B: 15-minute expiry edge branch ===
mailbox_id = 597 | tries = 0 | created_at backdated to 2026-07-13T16:52:21.085751+00:00
2026-07-13 17:08:21,090 - SL - INFO - 3234 - "/app/app/mailbox_utils.py:200" - verify_mailbox_code() -  - User <User 467 Test User user_x598yeysaq@mailbox.test> failed to verify mailbox 597 because code is too old
expiry: CannotVerifyError(msg='Invalid activation code. Please request another code.')
activation row count after expiry: 0

=== SCENARIO C: success control path (correct code) ===
mailbox_id = 598 | verified BEFORE = False | activation count BEFORE = 1
2026-07-13 17:08:21,150 - SL - INFO - 3234 - "/app/app/mailbox_utils.py:212" - verify_mailbox_code() -  - User <User 467 Test User user_x598yeysaq@mailbox.test> has verified mailbox 598
verify returned Mailbox id = 598
verified AFTER = True | activation count AFTER = 0
```

*(The `create_mailbox`/`send_verification_email`/`send_email` DEBUG lines emitted between scenarios are omitted here purely for length; the INFO lines that pinpoint each enforcement branch are shown verbatim.)*

**What this proves (before / intermediate / after):**

- **Limit + counter + progression [OBSERVED].** `MAX_ACTIVATION_TRIES = 3`; `tries` starts at `0` and advances **0 → 1 → 2 → 3** across submissions 1–3, each raising `CannotVerifyError(msg='Invalid activation code')`. The logger line `"/app/app/mailbox_utils.py:206"` confirms the "code does not match" branch fires on each wrong attempt; the increment/commit/raise are at `app/mailbox_utils.py:209-211`.
- **Lockout condition [OBSERVED].** Submission 4 (with `tries` already `3`) hits the top-of-function guard — logger line `"/app/app/mailbox_utils.py:196"` ("more than 3 times") — and raises `CannotVerifyError(msg='Invalid activation code. Please request another code.')`. The activation row is then **gone**: `activation row after lockout: None`, `activation row count after lockout: 0`, proving `clear_activation_codes_for_mailbox` [`app/mailbox_utils.py:197`] deleted it.
- **Post-lockout secondary path [OBSERVED].** Submission 5 now raises `MailboxError(msg='Invalid code')` via the no-activation branch — logger line `"/app/app/mailbox_utils.py:191"`, raise at `app/mailbox_utils.py:194`.
- **15-minute expiry edge branch [OBSERVED].** With `created_at` back-dated to 16 minutes ago, even the **correct** code is rejected: logger line `"/app/app/mailbox_utils.py:200"` ("code is too old"), `CannotVerifyError(msg='Invalid activation code. Please request another code.')`, and codes cleared (`count after expiry: 0`). Raise at `app/mailbox_utils.py:204`.
- **Success control path [OBSERVED].** A correct code flips `verified` `False → True` and clears the activation rows (`activation count AFTER = 0`) — logger line `"/app/app/mailbox_utils.py:212"` ("has verified"), `mailbox.verified = True` at `app/mailbox_utils.py:213`, clear at `app/mailbox_utils.py:219`.

### Evidence — the canonical HTTP entry point `GET /mailbox_verify` (proves no HTTP 429)

**Command:**

```bash
docker exec sl_canon bash -lc '
export CONFIG=tests/test.env PYTHONPATH=/app DB_URI="postgresql://test:test@localhost:5432/test"
cd /app && /app/venv/bin/python /tmp/obs/obs_q1_http.py'
```

`obs_q1_http.py` logs a real user in through the Flask test client and issues `GET /dashboard/mailbox_verify?...` with a wrong code four times, reading `tries` from the DB after each request. **Complete unedited output** (app logger lines elided):

```
mailbox_id = 600 | correct code = 'ouFcIEF2arGzsO8GLvTW7Q'
tries BEFORE = 0
GET http://sl.test/dashboard/mailbox_verify?mailbox_id=600&code=wrong-http-1 -> HTTP 302 | Location=http://sl.test/dashboard/mailbox | tries=1
GET http://sl.test/dashboard/mailbox_verify?mailbox_id=600&code=wrong-http-2 -> HTTP 302 | Location=http://sl.test/dashboard/mailbox | tries=2
GET http://sl.test/dashboard/mailbox_verify?mailbox_id=600&code=wrong-http-3 -> HTTP 302 | Location=http://sl.test/dashboard/mailbox | tries=3
GET http://sl.test/dashboard/mailbox_verify?mailbox_id=600&code=wrong-http-4 -> HTTP 302 | Location=http://sl.test/dashboard/mailbox | tries=None
activation count after 4 wrong HTTP submissions: 0
final GET (follow_redirects) -> HTTP 200 | flash='Cannot verify mailbox: Invalid activation code");'
```

**What this proves [OBSERVED].** Every wrong submission returns **HTTP 302** (a redirect to `/dashboard/mailbox`), **never HTTP 429** — even the 4th, which fires the lockout (`tries=None` because the row was deleted, `activation count … 0`). The route wraps the call in `try/except MailboxError` and flashes the error, redirecting [`app/dashboard/views/mailbox.py:130-133`]; following the redirect surfaces the flash `Cannot verify mailbox: Invalid activation code`. (The trailing `");` in the captured flash string is an artifact of the regex used to scrape the rendered HTML/JS, not part of the message text.)

### Evidence — the "no rate-limiter" nuance (route decorators + grep)

**Command:**

```bash
docker exec sl_canon bash -lc '
cd /app
echo "--- /mailbox_verify (L118-135) ---"; sed -n "118,135p" app/dashboard/views/mailbox.py
echo "--- /mailbox POST (L34-40) ---";     sed -n "34,40p"   app/dashboard/views/mailbox.py
echo "--- grep parallel_limiter/check_bucket_limit ---"; grep -n "parallel_limiter\|check_bucket_limit" app/dashboard/views/mailbox.py'
```

**Complete unedited output:**

```
--- /mailbox_verify (L118-135) ---
@dashboard_bp.route("/mailbox_verify")
@login_required
def mailbox_verify():
    mailbox_id = request.args.get("mailbox_id")
    code = request.args.get("code")
    if not code:
        # Old way
        return verify_with_signed_secret(mailbox_id)
    try:
        mailbox = mailbox_utils.verify_mailbox_code(current_user, mailbox_id, code)
    except mailbox_utils.MailboxError as e:
        LOG.i(f"Cannot verify mailbox {mailbox_id} because of {e}")
        flash(f"Cannot verify mailbox: {e.msg}", "error")
        return redirect(url_for("dashboard.mailbox_route"))
    LOG.d("Mailbox %s is verified", mailbox)
    return render_template("dashboard/mailbox_validation.html", mailbox=mailbox)
--- /mailbox POST (L34-40) ---
@dashboard_bp.route("/mailbox", methods=["GET", "POST"])
@login_required
@parallel_limiter.lock(only_when=lambda: request.method == "POST")
def mailbox_route():
    mailboxes = (
--- grep parallel_limiter/check_bucket_limit ---
13:from app import parallel_limiter, mailbox_utils, user_settings
38:@parallel_limiter.lock(only_when=lambda: request.method == "POST")
```

**What this proves [OBSERVED].** `/mailbox_verify` has only `@dashboard_bp.route("/mailbox_verify")` [`app/dashboard/views/mailbox.py:120`] and `@login_required` [`:121`] — no limiter. By contrast `/mailbox` POST carries `@parallel_limiter.lock(...)` [`:38`]. The grep shows `parallel_limiter` is only *imported* (`:13`) and applied at `:38` (the `/mailbox` route); it is **not** on `/mailbox_verify`, and `check_bucket_limit` does not appear at all. The generic Redis bucket limiter `check_bucket_limit` (which raises `werkzeug.exceptions.TooManyRequests()` at its `max_hits`) exists [`app/rate_limiter.py:19-42`, raise at `:40`] **[INFERRED from reading — not exercised]**, but it is not wired to this route, consistent with the observed HTTP 302 (never 429).

### Enforcement logic — source (for reference)

**Command:** `docker exec sl_canon bash -lc 'cd /app && sed -n "185,220p" app/mailbox_utils.py'`

```
    activation = (
        MailboxActivation.filter(MailboxActivation.mailbox_id == mailbox_id)
        .order_by(MailboxActivation.created_at.desc())
        .first()
    )
    if not activation:
        LOG.i(... "there is no activation")
        raise MailboxError("Invalid code")                                   # L194
    if activation.tries >= MAX_ACTIVATION_TRIES:                             # L195
        LOG.i(... "more than 3 times")
        clear_activation_codes_for_mailbox(mailbox)                          # L197
        raise CannotVerifyError("Invalid activation code. Please request another code.")  # L198
    if activation.created_at < arrow.now().shift(minutes=-15):               # L199
        LOG.i(... "code is too old")
        clear_activation_codes_for_mailbox(mailbox)                          # L203
        raise CannotVerifyError("Invalid activation code. Please request another code.")  # L204
    if code != activation.code:                                             # L208
        LOG.i(... "code does not match")
        activation.tries = activation.tries + 1                              # L209
        Session.commit()                                                     # L210
        raise CannotVerifyError("Invalid activation code")                   # L211
    LOG.i(... "has verified mailbox")                                        # L212
    mailbox.verified = True                                                  # L213
    ...
    clear_activation_codes_for_mailbox(mailbox)                              # L219
    return mailbox
```

### Q1 `file:line` reference table

| Value / behavior | `file:line` | Label |
|---|---|---|
| `MAX_ACTIVATION_TRIES = 3` | `app/mailbox_utils.py:43` | OBSERVED (runtime + source) |
| `verify_mailbox_code(...)` (enforcement fn) | `app/mailbox_utils.py:166` | OBSERVED |
| Route calls `verify_mailbox_code` | `app/dashboard/views/mailbox.py:129` | OBSERVED |
| No-activation → `MailboxError("Invalid code")` | `app/mailbox_utils.py:194` | OBSERVED |
| Lockout guard `tries >= MAX` → clear + raise | `app/mailbox_utils.py:195-198` | OBSERVED |
| 15-min expiry guard → clear + raise | `app/mailbox_utils.py:199-204` | OBSERVED |
| Wrong code: `tries += 1`; commit; raise | `app/mailbox_utils.py:209-211` | OBSERVED |
| Success: `verified = True`; clear | `app/mailbox_utils.py:213,219` | OBSERVED |
| `MailboxActivation.code` `String(32)` | `app/models.py:2834` | OBSERVED |
| `MailboxActivation.tries` `Integer` default 0 | `app/models.py:2835` | OBSERVED |
| `clear_activation_codes_for_mailbox` | `app/mailbox_utils.py:159-163` | OBSERVED |
| `/mailbox_verify` decorators (no limiter) | `app/dashboard/views/mailbox.py:120-121` | OBSERVED |
| `/mailbox` POST `@parallel_limiter.lock` | `app/dashboard/views/mailbox.py:38` | OBSERVED |
| Route try/except `MailboxError` → flash+redirect | `app/dashboard/views/mailbox.py:130-133` | OBSERVED |
| `check_bucket_limit` raises `TooManyRequests` (exists, not applied here) | `app/rate_limiter.py:19-42` (raise `:40`) | INFERRED |

### Stability

The tries progression `0 → 1 → 2 → 3`, the lockout, the post-lockout `MailboxError`, the expiry branch, and the success path reproduced **identically across two runs** of `obs_q1.py` (only the auto-generated ids/codes differ per run):

```
submission 1: CannotVerifyError(msg='Invalid activation code') -> tries now = 1
submission 2: CannotVerifyError(msg='Invalid activation code') -> tries now = 2
submission 3: CannotVerifyError(msg='Invalid activation code') -> tries now = 3
submission 4: CannotVerifyError(msg='Invalid activation code. Please request another code.')
activation row count after lockout: 0
submission 5: MailboxError(msg='Invalid code')
expiry: CannotVerifyError(msg='Invalid activation code. Please request another code.')
activation row count after expiry: 0
verified AFTER = True | activation count AFTER = 0
```

**[OBSERVED]**

---

## Q2 — Background-task (`Job`) lifecycle and error behavior

### Direct answer

Background tasks are rows in the **`Job`** table [`app/models.py:2683`], with a `state` column driven by the **`JobState`** enum (`ready=0`, `taken=1`, `done=2`, `error=3`) [`app/models.py:253-257`], plus `attempts` (default 0), `taken`, `taken_at`, and `run_at`. The daemon **`job_runner.py`** drains them: `get_jobs_to_run()` [`job_runner.py:307-326`] selects jobs that are `ready`, or `taken` with `taken_at` older than `JOB_TAKEN_RETRY_WAIT_MINS = 30` and `attempts < JOB_MAX_ATTEMPTS = 5` [`app/config.py:564-565`]. The `while True` loop [`job_runner.py:330`] marks each job `taken`, sets `taken_at`, sets `state = taken`, increments `attempts`, **commits**, calls `process_job(job)`, and only *then* sets `state = done` and commits [`job_runner.py:337-345`].

**Key finding (documented, not fixed):** there is **no `try/except` anywhere in `job_runner.py`**. So when `process_job` raises, the exception propagates out of the loop and the **runner process exits**; the `job.state = JobState.done.value` line at [`job_runner.py:344`] is never reached, the job remains at **`state = taken (1)` with `attempts` already incremented**, and **`JobState.error (3)` is never assigned by `job_runner.py`** (it appears only in `tasks/cleanup_old_jobs.py:15` and in tests). Recovery/retry therefore happens only after the process is restarted *and* the 30-minute window elapses, up to 5 attempts; end-of-life cleanup (`cleanup_old_jobs` [`tasks/cleanup_old_jobs.py:10-24`], invoked by `cron.py`'s `delete_old_data` on the `"30 5 * * *"` schedule in `crontab.yml`) deletes `done`/`error` jobs and `taken` jobs with `attempts >= 5`.

### Evidence — full lifecycle + error path via the REAL `get_jobs_to_run` and `process_job`

**Command:**

```bash
docker exec sl_canon bash -lc '
export CONFIG=tests/test.env PYTHONPATH=/app DB_URI="postgresql://test:test@localhost:5432/test"
cd /app && /app/venv/bin/python /tmp/obs/obs_q2.py'
```

`obs_q2.py` reproduces the loop body **verbatim** from `job_runner.py:337-345` (each line annotated with its source line) around the **real** `get_jobs_to_run()` and the **real** `process_job()`. The failing job is a real `JOB_BATCH_IMPORT` job with a non-existent `batch_import_id`, which makes the real `process_job` reach `handle_batch_import(None)` and raise. A `try/except` appears in the *harness only* so it can read the after-failure DB state; the note in the output makes explicit that the real daemon has no such guard. **Complete unedited output** (app logger lines elided except the one emitted by `process_job` itself):

```
JobState enum: {'ready': 0, 'taken': 1, 'done': 2, 'error': 3}
JOB_MAX_ATTEMPTS = 5 | JOB_TAKEN_RETRY_WAIT_MINS = 30

=== CREATE a failing job (JOB_BATCH_IMPORT, non-existent batch_import_id) ===
  [BEFORE (creation)] id=1467 name=batch-import state=0(ready) attempts=0 taken=False taken_at=None run_at=2026-07-13T17:09:04.631258+00:00

=== get_jobs_to_run() selects the fresh ready job? ===
  selected ids: [1467] | fresh job selected: True

=== Reproduce main-loop body job_runner.py:L337-L345 with REAL process_job ===
  [INTERMEDIATE (after commit L341, before process_job L342)] id=1467 name=batch-import state=1(taken) attempts=1 taken=True taken_at=2026-07-13T17:09:04.637011+00:00 run_at=2026-07-13T17:09:04.631258+00:00
  process_job RAISED: AttributeError: 'NoneType' object has no attribute 'user'
  => L344 (job.state=JobState.done.value) SKIPPED (in real daemon the process exits here)
  [AFTER-FAILURE (persisted state)] id=1467 name=batch-import state=1(taken) attempts=1 taken=True taken_at=2026-07-13T17:09:04.637011+00:00 run_at=2026-07-13T17:09:04.631258+00:00

=== Retry eligibility ===
  immediately after failure, get_jobs_to_run re-selects job? -> False (taken_at is recent, < 30min ago)
  after back-dating taken_at by 31 min, re-selected? -> True (attempts=1 < 5)
  with attempts=5 (>= JOB_MAX_ATTEMPTS=5), re-selected? -> False (retry stops)

=== SUCCESS control path (job that does NOT raise -> state=done(2)) ===
  [BEFORE] id=1468 name=unknown-noop-job-name state=0(ready) attempts=0 taken=False taken_at=None run_at=2026-07-13T17:09:04.647254+00:00
2026-07-13 17:09:04,670 - SL - ERROR - 3307 - "/app/job_runner.py:304" - process_job() -  - Unknown job name unknown-noop-job-name
NoneType: None
  [AFTER (success)] id=1468 name=unknown-noop-job-name state=2(done) attempts=1 taken=True taken_at=2026-07-13T17:09:04.649597+00:00 run_at=2026-07-13T17:09:04.647254+00:00

=== END-OF-LIFE: cleanup_old_jobs deletes done/error/(taken & attempts>=5) ===
  jobs before cleanup: [(1467, 'taken', 5), (1468, 'done', 1)]
  jobs after cleanup: []
```

**What this proves (creation → scheduling → pickup → execution → completion, with before/intermediate/after) [OBSERVED]:**

- **Enum + config [OBSERVED].** `JobState = {'ready':0,'taken':1,'done':2,'error':3}`; `JOB_MAX_ATTEMPTS = 5`; `JOB_TAKEN_RETRY_WAIT_MINS = 30`.
- **Creation (before) [OBSERVED].** A fresh job is `state=0(ready) attempts=0 taken=False taken_at=None`, with `run_at` set — matching the representative `Job.create(name=…, payload=…, run_at=arrow.now(), commit=True)` used across the app (e.g. `app/mailbox_utils.py:145-154`).
- **Scheduling/selection [OBSERVED].** The **real** `get_jobs_to_run()` returns `[1467]` — the fresh `ready` job is selected.
- **Pickup (intermediate) [OBSERVED].** After the loop's `Session.commit()` at `job_runner.py:341` and **before** `process_job`, the persisted row is `state=1(taken) attempts=1 taken=True taken_at=<set>`. This confirms the loop order: `taken=True` (`:337`) → `taken_at` (`:338`) → `state=taken` (`:339`) → `attempts += 1` (`:340`) → `commit` (`:341`).
- **Execution error → observable failure state (after) [OBSERVED].** `process_job` raised `AttributeError: 'NoneType' object has no attribute 'user'`; the `done` assignment at `job_runner.py:344` was skipped; the persisted row remained **`state=1(taken) attempts=1`** — **not** `error(3)`. This is the observable state that reflects the failure.
- **Recovery / retry behavior [OBSERVED].** Immediately after failure `get_jobs_to_run()` does **not** re-select the job (its `taken_at` is recent). After back-dating `taken_at` by 31 minutes (simulating the 30-minute wait — the *selection function itself is the real one*), it **is** re-selected while `attempts=1 < 5`. Once `attempts=5 (>= JOB_MAX_ATTEMPTS)` it is **no longer** re-selected — retries stop at 5.
- **Completion / success control [OBSERVED].** A job whose `process_job` does not raise ends at `state=2(done) attempts=1`. (Here the real `process_job` logs `Unknown job name` at `job_runner.py:304` and returns normally, so the harness reaches the `done` assignment.)
- **End-of-life [OBSERVED].** The **real** `cleanup_old_jobs(oldest_allowed)` deleted both a `taken` job with `attempts=5` and a `done` job: `before=[(1467,'taken',5),(1468,'done',1)]` → `after=[]`.

### Evidence — the REAL daemon exiting on an uncaught exception (strongest evidence)

Rather than only reproducing the loop body, the actual `job_runner.py` daemon was launched against a single failing job. **Command:**

```bash
docker exec sl_canon bash -lc '
export CONFIG=tests/test.env PYTHONPATH=/app DB_URI="postgresql://test:test@localhost:5432/test"
cd /app
/app/venv/bin/python /tmp/obs/obs_q2_setup.py create          # seed one failing job
timeout 25 /app/venv/bin/python job_runner.py; echo "daemon exit code = $?"
/app/venv/bin/python /tmp/obs/obs_q2_inspect.py                # read persisted state'
```

**Complete unedited output — RUN 1** (app logger lines elided; the traceback is the daemon's own stderr, verbatim):

```
CREATED job id=1469 state=0(ready) attempts=0
--- launching REAL daemon: timeout 25 /app/venv/bin/python job_runner.py ---
daemon exit code = 1
Traceback (most recent call last):
  File "/app/job_runner.py", line 342, in <module>
    process_job(job)
  File "/app/job_runner.py", line 225, in process_job
    handle_batch_import(batch_import)
  File "/app/app/import_utils.py", line 23, in handle_batch_import
    user = batch_import.user
AttributeError: 'NoneType' object has no attribute 'user'
PERSISTED job id=1469 name=batch-import state=1(taken) attempts=1 taken=True taken_at=2026-07-13T17:09:24.288557+00:00
```

**Complete unedited output — RUN 2 (stability):**

```
CREATED job id=1470 state=0(ready) attempts=0
daemon exit code = 1
Traceback (most recent call last):
  File "/app/job_runner.py", line 342, in <module>
    process_job(job)
  File "/app/job_runner.py", line 225, in process_job
    handle_batch_import(batch_import)
  File "/app/app/import_utils.py", line 23, in handle_batch_import
    user = batch_import.user
AttributeError: 'NoneType' object has no attribute 'user'
PERSISTED job id=1470 name=batch-import state=1(taken) attempts=1 taken=True taken_at=2026-07-13T17:09:39.542496+00:00
```

**What this proves [OBSERVED].** Running the **canonical** `job_runner.py` `__main__` daemon, an error inside `process_job` (at `job_runner.py:342` → `:225` → `app/import_utils.py:23`) propagates uncaught and the **process exits with code 1**. The persisted job is left at **`state=1(taken) attempts=1`** — never `error(3)`. Both runs are byte-for-byte identical apart from the auto-incremented job id and timestamp. This is the concrete, canonical demonstration of the key finding: **a failing job never transitions to `JobState.error`; the daemon dies and the job stays `taken` with `attempts` incremented.**

### Evidence — the key finding, by grep (no `try/except`; where `JobState.error` lives; not in cron)

**Command:**

```bash
docker exec sl_canon bash -lc '
cd /app
echo "(1)"; grep -n "try:\|except" job_runner.py; echo "exit=$?"
echo "(2)"; grep -rn "JobState.error" --include="*.py" .
echo "(3)"; grep -n "job_runner" crontab.yml; echo "exit=$?"
echo "(4)"; sed -n "329,347p" job_runner.py'
```

**Complete unedited output:**

```
(1)
exit=1
(2)
./tests/tasks/test_cleanup_old_jobs.py:21:            state=JobState.error.value,
./tests/tasks/test_cleanup_old_jobs.py:46:            state=JobState.error.value,
./tasks/cleanup_old_jobs.py:15:            Job.state == JobState.error.value,
(3)
exit=1
(4)
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

**What this proves [OBSERVED].**
1. `grep "try:\|except" job_runner.py` returns **nothing** (`exit=1`) — there is **no exception handling** in the daemon.
2. `JobState.error` is assigned **only** in `tasks/cleanup_old_jobs.py:15` (as a *deletion filter*) and in `tests/tasks/test_cleanup_old_jobs.py:21,46` — **never** in `job_runner.py`.
3. `grep job_runner crontab.yml` returns **nothing** (`exit=1`) — `job_runner.py` is a standalone `while True` daemon, **not** a cron-scheduled task.
4. The loop source confirms the exact order and the `time.sleep(10)` between passes, with `process_job(job)` sitting between the `state=taken` commit and the `state=done` commit — with nothing catching a raise from `process_job`.

### Evidence — end-of-life cleanup source + scheduling

**Command:**

```bash
docker exec sl_canon bash -lc '
cd /app
echo "--- cleanup_old_jobs (L10-24) ---"; sed -n "10,24p" tasks/cleanup_old_jobs.py
echo "--- cron.py delete_old_data (L1245-1248) ---"; sed -n "1245,1248p" cron.py
echo "--- crontab.yml (L40-44) ---"; sed -n "40,44p" crontab.yml'
```

**Complete unedited output:**

```
--- cleanup_old_jobs (L10-24) ---
def cleanup_old_jobs(oldest_allowed: arrow.Arrow):
    LOG.i(f"Deleting jobs older than {oldest_allowed}")
    count = Job.filter(
        or_(
            Job.state == JobState.done.value,
            Job.state == JobState.error.value,
            and_(
                Job.state == JobState.taken.value,
                Job.attempts >= config.JOB_MAX_ATTEMPTS,
            ),
        ),
        Job.updated_at < oldest_allowed,
    ).delete()
    Session.commit()
    LOG.i(f"Deleted {count} jobs")
--- cron.py delete_old_data (L1245-1248) ---
def delete_old_data():
    oldest_valid = arrow.now().shift(days=-config.KEEP_OLD_DATA_DAYS)
    cleanup_old_imports(oldest_valid)
    cleanup_old_jobs(oldest_valid)
--- crontab.yml (L40-44) ---
  - name: SimpleLogin Delete Old data
    command: python /code/cron.py -j delete_old_data
    shell: /bin/bash
    schedule: "30 5 * * *"
    captureStderr: true
```

**What this proves [OBSERVED].** `cleanup_old_jobs` deletes jobs whose `state` is `done` **or** `error`, **or** `taken` with `attempts >= JOB_MAX_ATTEMPTS`, and `updated_at < oldest_allowed` [`tasks/cleanup_old_jobs.py:10-24`]. It is invoked by `delete_old_data()` [`cron.py:1245-1248`, call at `:1248`], scheduled in `crontab.yml` as "SimpleLogin Delete Old data" running `python /code/cron.py -j delete_old_data` on `"30 5 * * *"` [`crontab.yml:40-44`].

### Evidence — canonical harnesses pass

**Command:**

```bash
docker exec sl_canon bash -lc '
export CONFIG=tests/test.env PYTHONPATH=/app DB_URI="postgresql://test:test@localhost:5432/test" PYTEST_ADDOPTS=""
cd /app && /app/venv/bin/python -m pytest -c pytest.ci.ini \
  tests/jobs/test_job_runner.py::test_get_jobs_to_run \
  tests/tasks/test_cleanup_old_jobs.py -p no:randomly'
```

**Tail of output:**

```
---------- coverage: platform linux, python 3.10.18-final-0 ----------
Coverage HTML written to dir htmlcov

FAIL Required test coverage of 55.0% not reached. Total coverage: 19.68%
======================== 2 passed, 20 warnings in 2.32s ========================
```

**What this proves [OBSERVED].** The canonical harnesses `test_get_jobs_to_run` [`tests/jobs/test_job_runner.py:8`] and `test_cleanup_old_jobs` [`tests/tasks/test_cleanup_old_jobs.py:8`] both pass (`2 passed`). The `FAIL Required test coverage…` line is the coverage-threshold gate from running just two tests in isolation — it is **not** a test failure.

### Q2 `file:line` reference table

| Value / behavior | `file:line` | Label |
|---|---|---|
| `Job` model (name, payload, taken, run_at, state, attempts, taken_at) | `app/models.py:2683-2707` | OBSERVED |
| `JobState` enum `ready=0/taken=1/done=2/error=3` | `app/models.py:253-257` | OBSERVED |
| `JOB_MAX_ATTEMPTS = 5` | `app/config.py:564` | OBSERVED |
| `JOB_TAKEN_RETRY_WAIT_MINS = 30` | `app/config.py:565` | OBSERVED |
| `get_jobs_to_run()` selection | `job_runner.py:307-326` | OBSERVED |
| Main loop (`__main__`, `while True`) | `job_runner.py:329-347` | OBSERVED |
| Loop order: taken/taken_at/state=taken/attempts+1/commit | `job_runner.py:337-341` | OBSERVED |
| `process_job(job)` call | `job_runner.py:342` | OBSERVED |
| `state = done` (skipped on failure) | `job_runner.py:344` | OBSERVED |
| `process_job` executor | `job_runner.py:188-304` | OBSERVED |
| Failure origin (`handle_batch_import`) | `app/import_utils.py:23` | OBSERVED |
| **No `try/except`** in daemon | `job_runner.py` (grep empty) | OBSERVED |
| `JobState.error` used only in cleanup + tests | `tasks/cleanup_old_jobs.py:15`; `tests/tasks/test_cleanup_old_jobs.py:21,46` | OBSERVED |
| `cleanup_old_jobs` deletion rule | `tasks/cleanup_old_jobs.py:10-24` | OBSERVED |
| `delete_old_data` invokes cleanup | `cron.py:1245-1248` | OBSERVED |
| Cron schedule `"30 5 * * *"` | `crontab.yml:40-44` | OBSERVED |
| `job_runner` **not** in crontab | `crontab.yml` (grep empty) | OBSERVED |

### Stability

Both the loop-body reproduction and the **real daemon** were run twice; RUN 1 and RUN 2 of the daemon are identical (exit code 1, same traceback through `job_runner.py:342 → :225 → app/import_utils.py:23`, persisted `state=taken(1) attempts=1`), differing only by the auto-incremented job id/timestamp. **[OBSERVED]**


---

## Q3 — Email-forwarding bounce handling (VERP address format + direction-dependent behavior)

### Direct answer

The bounce/return-path address is a **VERP (Variable Envelope Return Path)** address built by **`generate_verp_email(verp_type, object_id, sender_domain)`** [`app/email_utils.py:1438-1465`]. Its exact format is (all lowercased):

```
{VERP_PREFIX}.{base32(payload)}.{base32(hmac_signature)}@{domain}
```

where `VERP_PREFIX = "sl"` [`app/config.py:500`], `payload = [verp_type.value, object_id or 0, int((time.time() - VERP_TIME_START) / 60)]` (i.e. `[VerpType, email_log.id, minutes_since_2022]`) [`app/email_utils.py:1446-1450`], and the signature is the first 8 bytes of `HMAC(VERP_EMAIL_SECRET, json_payload, "sha3-224")` [`app/email_utils.py:1454-1456`, `VERP_HMAC_ALGO` at `:69`], both base32-encoded with `=` stripped [`:1457-1458`]. **Direction** is encoded by `VerpType` (`bounce_forward=0`, `bounce_reply=1`) [`app/models.py:247-251`]: the forward phase generates the address with `VerpType.bounce_forward` and the contact domain [`email_handler.py:905`], the reply phase with `VerpType.bounce_reply` and the alias domain [`email_handler.py:1225`].

When a failure notification arrives, `handle()` decodes the recipient with **`get_verp_info_from_email`** [`app/email_utils.py:1467-1499`], which HMAC-verifies and returns `(VerpType, email_log.id)` — that recovers **the original email** (the `EmailLog` id) and its direction. Then **`handle_bounce`** [`email_handler.py:1851`] branches on `email_log.is_reply` [`email_handler.py:1873`]:

- **Forward phase** → `handle_bounce_forward_phase` [`email_handler.py:1432`], returns **`status.E211`** ("`250 SL E211 Bounce Forward phase handled`") [`:1913-1914`]. It creates a `Bounce` keyed on the **mailbox** email, stores a `RefusedEmail`, sets `email_log.bounced = True`, `refused_email_id`, and `bounced_mailbox_id`, and may auto-disable the alias after repeated bounces.
- **Reply phase** → `handle_bounce_reply_phase` [`email_handler.py:1595`], returns **`status.E212`** ("`250 SL E212 Bounce Reply phase handled`") [`:1910-1911`]. It creates a `Bounce` keyed on the **contact's `website_email`**, stores a `RefusedEmail`, sets the same three `EmailLog` fields, and creates a user `Notification`.

An **auto-reply edge branch** [`email_handler.py:1876`] treats a message that is not `multipart/report` (or has a non-empty MAIL FROM) as an auto-reply — `email_log.auto_replied = True` [`:1887`] and re-forward — rather than a bounce.

### Evidence — the exact VERP address format, decode round-trip, tamper, lifetime, legacy

**Command:**

```bash
docker exec sl_canon bash -lc '
export CONFIG=tests/test.env PYTHONPATH=/app DB_URI="postgresql://test:test@localhost:5432/test"
cd /app && /app/venv/bin/python /tmp/obs/obs_q3_verp.py'
```

`obs_q3_verp.py` calls the **real** `generate_verp_email` for both directions, round-trips through the **real** `get_verp_info_from_email`, tampers a signature, forges an over-age payload with a valid HMAC, and parses the legacy forms with the **real** `parse_id_from_bounce`. **Complete unedited output:**

```
config: VERP_PREFIX='sl' EMAIL_DOMAIN='sl.local' VERP_HMAC_ALGO='sha3-224' VERP_TIME_START=1640995200 VERP_MESSAGE_LIFETIME=432000 (=5.0 days)

=== GENERATE (default domain = config.EMAIL_DOMAIN) ===
FORWARD (bounce_forward=0): sl.lmycyibzha3tmnjufqqdemzygi3tsmc5.6a7nfoyfxa442@sl.local
REPLY   (bounce_reply=1)  : sl.lmysyibzha3tmnjufqqdemzygi3tsmc5.qlfsjutjtzkis@sl.local

=== GENERATE (explicit domains, as the handler call sites do) ===
FORWARD @contactdomain.com: sl.lmycyibzha3tmnjufqqdemzygi3tsmc5.6a7nfoyfxa442@contactdomain.com
REPLY   @aliasdomain.com  : sl.lmysyibzha3tmnjufqqdemzygi3tsmc5.qlfsjutjtzkis@aliasdomain.com

=== ROUND-TRIP decode -> (VerpType, original email_log id) ===
decode FORWARD: (<VerpType.bounce_forward: 0>, 987654)
decode REPLY  : (<VerpType.bounce_reply: 1>, 987654)

=== SHOW the 3-field structure {prefix}.{b32 payload}.{b32 signature}@{domain} ===
field[0] (prefix)   = sl
field[1] (payload)  = lmycyibzha3tmnjufqqdemzygi3tsmc5
field[2] (signature)= 6a7nfoyfxa442
decoded payload JSON= [0, 987654, 2382790]  (=[verp_type.value, email_log.id, minutes_since_2022])

=== TAMPER: flip last char of signature -> HMAC verify fails -> None ===
tampered address: sl.lmycyibzha3tmnjufqqdemzygi3tsmc5.6a7nfoyfxa44a@contactdomain.com
decode tampered : None

=== LIFETIME: forge a payload dated >5 days in the FUTURE, sign it correctly -> None ===
future-dated (valid HMAC) address: sl.lmycyibzha3tmnjufqqdemzzgaydkmc5.2csftkwyaak3e@sl.local
decode over-age: None  (rejected: data[2] > (now + VERP_MESSAGE_LIFETIME - VERP_TIME_START)/60)

=== LEGACY forms parsed by parse_id_from_bounce ===
legacy forward: bounce+987654+@sl.local -> id 987654
legacy reply  : bounce_reply+987654+@sl.local -> id 987654
```

**What this proves [OBSERVED]:**

- **Exact format, both directions.** For `email_log.id = 987654`, the emitted addresses are
  - **forward** `sl.lmycyibzha3tmnjufqqdemzygi3tsmc5.6a7nfoyfxa442@sl.local`
  - **reply** `sl.lmysyibzha3tmnjufqqdemzygi3tsmc5.qlfsjutjtzkis@sl.local`

  Both match `{sl}.{base32 payload}.{base32 sig}@{domain}`, all lowercase. Only the payload's leading `VerpType` byte differs between directions (visible as the stable prefix distinction `lmycy…` for forward vs `lmysy…` for reply); with explicit domains the handler uses the **contact** domain for forward [`email_handler.py:905`] and the **alias** domain for reply [`email_handler.py:1225`].
- **3-field structure + payload contents.** `field[0]='sl'`, `field[1]` = base32 payload, `field[2]` = base32 signature; decoding `field[1]` gives the JSON list **`[0, 987654, 2382790]`** = `[verp_type.value, email_log.id, minutes_since_2022]`.
- **Identification of the original email (round-trip).** `get_verp_info_from_email` returns `(<VerpType.bounce_forward: 0>, 987654)` and `(<VerpType.bounce_reply: 1>, 987654)` — recovering both the **original `EmailLog` id** and the **direction**.
- **HMAC tamper-resistance.** Flipping the last signature character yields `decode tampered : None` — the HMAC comparison [`app/email_utils.py:1487-1491`] rejects it.
- **Lifetime enforcement.** A payload dated >5 days in the future *with a valid HMAC* still decodes to `None` — the lifetime check [`app/email_utils.py:1496-1497`] rejects it (`VERP_MESSAGE_LIFETIME = 432000` s = 5 days).
- **Legacy forms.** `parse_id_from_bounce` [`app/email_utils.py:1258-1259`] extracts `987654` from both `bounce+987654+@sl.local` (forward) and `bounce_reply+987654+@sl.local` (reply).

> **Time component note.** Because `payload[2]` is minutes-since-2022, `field[1]`/`field[2]` change slowly over time; a second run yielded payload minutes `2382790`→ higher and correspondingly different base32 tails, while the format, the leading `VerpType` byte, and the decoded `email_log.id` were identical. The `example.env` comments [`example.env:43-48`] document the production-style `BOUNCE_PREFIX`/`BOUNCE_SUFFIX`/`BOUNCE_PREFIX_FOR_REPLY_PHASE` defaults; the values above are from the default test config (`VERP_PREFIX='sl'`, `EMAIL_DOMAIN='sl.local'`).

### `generate_verp_email` source (for reference)

**Command:** `docker exec sl_canon bash -lc 'cd /app && sed -n "1446,1465p" app/email_utils.py'`

```
    data = [
        verp_type.value,
        object_id or 0,
        int((time.time() - VERP_TIME_START) / 60),
    ]
    json_payload = json.dumps(data).encode("utf-8")
    payload_hmac = hmac.new(
        config.VERP_EMAIL_SECRET.encode("utf-8"), json_payload, VERP_HMAC_ALGO
    ).digest()[:8]
    encoded_payload = base64.b32encode(json_payload).rstrip(b"=").decode("utf-8")
    encoded_signature = base64.b32encode(payload_hmac).rstrip(b"=").decode("utf-8")
    return "{}.{}.{}@{}".format(
        config.VERP_PREFIX,
        encoded_payload,
        encoded_signature,
        sender_domain or config.EMAIL_DOMAIN,
    ).lower()
```

### Evidence — direction-dependent dispatch through the REAL `email_handler.handle()`

**Command:**

```bash
docker exec sl_canon bash -lc '
export CONFIG=tests/test.env PYTHONPATH=/app DB_URI="postgresql://test:test@localhost:5432/test"
cd /app && /app/venv/bin/python /tmp/obs/obs_q3_handler.py'
```

`obs_q3_handler.py` builds a real user, alias, mailbox, and contact; creates one `EmailLog` with `is_reply=False` and one with `is_reply=True`; generates the matching VERP address for each; and drives the **real** `email_handler.handle(envelope, msg)` with a real `multipart/report` DSN (parsed from raw bytes, exactly as `handle_DATA` receives mail) and `envelope.mail_from = "<>"`. **Complete unedited output** (the forward/reply sections; the trailing auto-reply probe is discussed in the next subsection):

```
status.E211 = '250 SL E211 Bounce Forward phase handled'
status.E212 = '250 SL E212 Bounce Reply phase handled'

mailbox.email = user_yc1racjl2r@mailbox.test | contact.website_email = sender@remote.example
DSN content type = multipart/report

=== FORWARD DIRECTION (is_reply=False) -> handle_bounce_forward_phase -> E211 ===
  [BEFORE] email_log id=376 is_reply=False get_phase=forward bounced=False auto_replied=False refused_email_id=None bounced_mailbox_id=None
  bounce VERP rcpt_to: sl.lmycyibtg43cyibsgm4denzzgboq.p46jdehr2t3b2@mailbox.test
  handle() returned: '250 SL E211 Bounce Forward phase handled' | == status.E211 -> True
  [AFTER] email_log id=376 is_reply=False get_phase=forward bounced=True auto_replied=False refused_email_id=10 bounced_mailbox_id=601
  Bounce.email = 'user_yc1racjl2r@mailbox.test' (keyed on MAILBOX email); count 0->1
  RefusedEmail exists for refused_email_id=10 -> True

=== REPLY DIRECTION (is_reply=True) -> handle_bounce_reply_phase -> E212 ===
  [BEFORE] email_log id=377 is_reply=True get_phase=reply bounced=False auto_replied=False refused_email_id=None bounced_mailbox_id=None
  bounce VERP rcpt_to: sl.lmysyibtg43syibsgm4denzzgboq.ht34vuvwpqrpm@sl.local
  handle() returned: '250 SL E212 Bounce Reply phase handled' | == status.E212 -> True
  [AFTER] email_log id=377 is_reply=True get_phase=reply bounced=True auto_replied=False refused_email_id=11 bounced_mailbox_id=601
  Bounce.email = 'sender@remote.example' (keyed on CONTACT website_email); count 2->3
  RefusedEmail exists for refused_email_id=11 -> True
  Notification count for user 1->2 (reply phase notifies user)
```

**What this proves (before / after, per direction) [OBSERVED]:**

- **Status codes.** `status.E211 = '250 SL E211 Bounce Forward phase handled'` and `status.E212 = '250 SL E212 Bounce Reply phase handled'` — both are **`250`** success strings, differing only by phase label.
- **Forward direction.** With `is_reply=False` (`get_phase=forward`): BEFORE `bounced=False, refused_email_id=None, bounced_mailbox_id=None`; `handle()` returned **E211** (`== status.E211 -> True`); AFTER `bounced=True, refused_email_id=10, bounced_mailbox_id=601`. The `Bounce` row is keyed on the **mailbox** email `user_yc1racjl2r@mailbox.test` (count `0->1`) and a `RefusedEmail` exists. This confirms the forward phase **does** set `refused_email_id` (not only the reply phase).
- **Reply direction.** With `is_reply=True` (`get_phase=reply`): BEFORE `bounced=False`; `handle()` returned **E212** (`== status.E212 -> True`); AFTER `bounced=True, refused_email_id=11, bounced_mailbox_id=601`. The `Bounce` row is keyed on the **contact's `website_email`** `sender@remote.example` (count `2->3`), a `RefusedEmail` exists, and a user **`Notification`** was created (`count 1->2`).
- **Original-email identification.** The generated VERP `rcpt_to` is decoded back to the correct `EmailLog` id in each case (the handler loads `EmailLog.get(email_log_id)` from the decoded id at `email_handler.py:2062/2082`), which is why the correct `email_log` row (376 / 377) transitions state.

### `handle_bounce` branch + `handle()` VERP decode (source, for reference)

**Command:** `docker exec sl_canon bash -lc 'cd /app && sed -n "1873,1914p" email_handler.py'` (key lines):

```
    if email_log.is_reply:                                           # L1873
        ...
        if content_type != "multipart/report" or envelope.mail_from != "<>":   # L1876 (auto-reply edge)
            ...
            email_log.auto_replied = True                            # L1887
            ...
        ...
        handle_bounce_reply_phase(envelope, msg, email_log)          # L1910
        return status.E212                                           # L1911
    else:
        handle_bounce_forward_phase(msg, email_log)                  # L1913
        return status.E211                                           # L1914
```

The inbound `handle()` decodes the recipient at `email_handler.py:2035` (`verp_info = get_verp_info_from_email(rcpt_tos[0])`), matches the forward-VERP branch (`verp_info[0] == VerpType.bounce_forward`) or the reply-VERP branch (`… == VerpType.bounce_reply`), recovers `email_log_id = (verp_info and verp_info[1]) or parse_id_from_bounce(rcpt_tos[0])`, and calls `handle_bounce` when `is_bounce(...)` (which requires `mail_from == "<>"` **and** `multipart/report`, `email_handler.py:1813`) is true.

### Evidence — the auto-reply edge branch (guard vs the un-gated caller)

**Command:**

```bash
docker exec sl_canon bash -lc '
export CONFIG=tests/test.env PYTHONPATH=/app DB_URI="postgresql://test:test@localhost:5432/test"
cd /app && /app/venv/bin/python /tmp/obs/obs_q3_autoreply.py'
```

**Complete unedited output:**

```
status.E213 = '250 SL E213 Unknown email ignored'

=== (a) GUARD via REAL async handle_DATA: reply-VERP rcpt + text/plain + mail_from='<>' ===
  handle_DATA returned: '250 SL E213 Unknown email ignored' | == status.E213 -> True (VERPReply caught by callback)
  [AFTER (guard: NOT auto_replied, NOT bounced)] email_log id=379 is_reply=True bounced=False auto_replied=False

=== (b) AUTO-REPLY branch via iCloud/legacy caller [L2116] through REAL handle() ===
  [BEFORE] email_log id=380 is_reply=True bounced=False auto_replied=False
  mail_from='bounce+380+@sl.local' rcpt_to='edible_latish895@sl.local' content_type=text/plain
  handle() returned: '250 Message accepted for delivery'
  [AFTER (auto_replied=True set at L1887; bounced stays False => re-forwarded, not a bounce)] email_log id=380 is_reply=True bounced=False auto_replied=True
```

**What this proves [OBSERVED] — a runtime nuance discovered by exercising the real paths:**

- **(a) The standard reply-VERP path is guarded.** Driving the **real async** `MailHandler.handle_DATA` with a reply-VERP recipient, a `text/plain` (non-report) body, and `mail_from='<>'` returns **`status.E213`** ("`250 SL E213 Unknown email ignored`"), leaving `auto_replied=False, bounced=False`. This is because the standard branch only calls `handle_bounce` when `is_bounce()` is true; when the message is *not* a report, `handle()` raises `VERPReply` at `email_handler.py:2095` (confirmed by the traceback captured when calling `handle()` directly: `app.errors.VERPReply: VERPReply cannot handle email sent to reply VERP …`), which the `handle_DATA` callback catches and converts to E213. So the auto-reply branch inside `handle_bounce` is **not** reachable via the standard reply-VERP recipient path.
- **(b) The auto-reply branch is reached via the un-gated iCloud/legacy caller.** `handle()` has one caller that invokes `handle_bounce` **unconditionally** — the iCloud/legacy path at `email_handler.py:2116` (recognized by `mail_from = bounce+{id}+@domain`, `rcpt_to = alias`). Driving that path with a `text/plain` message hits the auto-reply edge: `handle()` returns **`'250 Message accepted for delivery'`** (the re-forward delivery status) and the `EmailLog` ends at **`auto_replied=True, bounced=False`** — i.e. it is re-forwarded to the alias, **not** recorded as a bounce. `auto_replied = True` is set at `email_handler.py:1887`.

### Evidence — forward-phase modifier: alias auto-disable after repeated bounces

**Command:**

```bash
docker exec sl_canon bash -lc '
export CONFIG=tests/test.env PYTHONPATH=/app DB_URI="postgresql://test:test@localhost:5432/test"
cd /app && /app/venv/bin/python /tmp/obs/obs_q3_disable.py'
```

**Complete unedited output:**

```
seeded bounced-forward count (24h): 13
alias.enabled BEFORE: True
handle() returned: '250 SL E211 Bounce Forward phase handled' | == status.E211 -> True
alias.enabled AFTER: False
Notification count 0->1
latest Notification.title: 'fliest_hunger093@sl.local has been disabled due to multiple bounces'
```

**What this proves [OBSERVED].** After seeding 13 bounced-forward `EmailLog`s in the last 24h for one alias, a real forward bounce through `handle()` (returning **E211**) flips **`alias.enabled` `True → False`** and creates a `Notification` titled `'… has been disabled due to multiple bounces'`. This is the forward-phase auto-disable modifier: `should_disable(alias)` [`app/email_utils.py:1166`] → `change_alias_status(enabled=False)` [`email_handler.py:1505`] → `Notification.create(...)` [`email_handler.py:1509`].

### Q3 `file:line` reference table

| Value / behavior | `file:line` | Label |
|---|---|---|
| `generate_verp_email` (address builder) | `app/email_utils.py:1438-1465` | OBSERVED |
| Payload `[verp_type, id, minutes_since_2022]` | `app/email_utils.py:1446-1450` | OBSERVED |
| HMAC first 8 bytes (`sha3-224`) | `app/email_utils.py:1454-1456` (`VERP_HMAC_ALGO` `:69`) | OBSERVED |
| base32 `rstrip(=)` payload + signature | `app/email_utils.py:1457-1458` | OBSERVED |
| Format `"{}.{}.{}@{}".lower()` | `app/email_utils.py:1460-1464` | OBSERVED |
| `VERP_TIME_START = 1640995200` | `app/email_utils.py:68` | OBSERVED |
| `VERP_PREFIX = "sl"` | `app/config.py:500` | OBSERVED |
| `VERP_MESSAGE_LIFETIME = 5*86400` (5 days) | `app/config.py:499` | OBSERVED |
| `get_verp_info_from_email` (decode + HMAC + lifetime) | `app/email_utils.py:1467-1499` (HMAC `:1487-1491`, lifetime `:1496-1497`) | OBSERVED |
| `parse_id_from_bounce` (legacy) | `app/email_utils.py:1258-1259` | OBSERVED |
| Legacy `BOUNCE_PREFIX="bounce+"` / `BOUNCE_SUFFIX` | `app/config.py:100-101` | OBSERVED |
| Legacy `BOUNCE_PREFIX_FOR_REPLY_PHASE="bounce_reply"` | `app/config.py:108-109` | OBSERVED |
| `VerpType` `bounce_forward=0/bounce_reply=1/transactional=2` | `app/models.py:247-251` | OBSERVED |
| Forward generation call site (`contact_domain`) | `email_handler.py:905` | OBSERVED |
| Reply generation call site (`alias_domain`) | `email_handler.py:1225` | OBSERVED |
| `handle()` inbound entry | `email_handler.py:1945` | OBSERVED |
| `handle()` VERP decode of `rcpt_tos[0]` | `email_handler.py:2035` | OBSERVED |
| Reply-VERP `raise VERPReply` (non-report) | `email_handler.py:2095` | OBSERVED |
| Un-gated iCloud/legacy `handle_bounce` call | `email_handler.py:2116` | OBSERVED |
| `handle_bounce` branches on `email_log.is_reply` | `email_handler.py:1851,1873` | OBSERVED |
| Auto-reply edge (`content_type != multipart/report …`) | `email_handler.py:1876` | OBSERVED |
| `email_log.auto_replied = True` | `email_handler.py:1887` | OBSERVED |
| Reply phase → `E212` | `email_handler.py:1910-1911` | OBSERVED |
| Forward phase → `E211` | `email_handler.py:1913-1914` | OBSERVED |
| `handle_bounce_forward_phase` | `email_handler.py:1432` | OBSERVED |
| `handle_bounce_reply_phase` | `email_handler.py:1595` | OBSERVED |
| `E211`/`E212`/`E213` status strings (all `250`) | `app/email/status.py:19-21` | OBSERVED |
| `EmailLog.is_reply/bounced/auto_replied/refused_email_id/bounced_mailbox_id/get_phase` | `app/models.py:2075,2082,2085,2097,2109,2143` | OBSERVED |
| `is_bounce` requires `<>` + `multipart/report` | `email_handler.py:1813` | OBSERVED |
| `should_disable(alias)` (forward auto-disable) | `app/email_utils.py:1166` | OBSERVED |
| `example.env` documents bounce prefixes | `example.env:43-48` | OBSERVED |

### Stability

The generate/decode round-trip, tamper→`None`, over-age→`None`, and legacy parsing reproduced identically across two runs of `obs_q3_verp.py` (only the time-dependent base32 tail changed, as expected); the direction-dependent dispatch (E211 forward / E212 reply, `Bounce` keying, reply `Notification`) reproduced identically across two runs of `obs_q3_handler.py`. **[OBSERVED]**

```
decode FORWARD: (<VerpType.bounce_forward: 0>, 987654)
decode REPLY  : (<VerpType.bounce_reply: 1>, 987654)
decode tampered : None
decode over-age: None  (rejected: data[2] > (now + VERP_MESSAGE_LIFETIME - VERP_TIME_START)/60)
legacy forward: bounce+987654+@sl.local -> id 987654
legacy reply  : bounce_reply+987654+@sl.local -> id 987654
```


---

## Coverage pass — every named sub-part answered

Each question was re-read and decomposed into its named sub-parts; each maps to the evidence above.

### Q1 — mailbox verification-code enforcement

| Named sub-part | Answered? | Where / observed value |
|---|---|---|
| The limit | ✓ | `MAX_ACTIVATION_TRIES = 3` [`app/mailbox_utils.py:43`] — runtime-printed |
| Counter/state field | ✓ | `MailboxActivation.tries` (Integer, default 0) [`app/models.py:2835`] |
| Counter progression | ✓ | `tries` **0 → 1 → 2 → 3** across submissions 1–3 (obs_q1.py) |
| Lockout condition (prevents further submissions) | ✓ | `tries >= 3` guard → clear codes + `CannotVerifyError("… Please request another code.")` [`:195-198`] |
| Code invalidation | ✓ | `activation row after lockout: None`, `count = 0` (rows deleted by `clear_activation_codes_for_mailbox` [`:197`]) |
| Post-lockout behavior | ✓ | further submission → `MailboxError("Invalid code")` [`:194`] |
| 15-minute expiry branch | ✓ | back-dated `created_at` → "code is too old" → `CannotVerifyError` [`:199-204`] |
| Success control path | ✓ | correct code → `verified False→True`, codes cleared [`:213,219`] |
| No-429 / no-limiter nuance | ✓ | HTTP **302** on every attempt (never 429); `/mailbox_verify` has no limiter vs `/mailbox` `@parallel_limiter.lock` [`:38`] |

### Q2 — background-task lifecycle and error behavior

| Named sub-part | Answered? | Where / observed value |
|---|---|---|
| Creation | ✓ | `Job.create(...)` → `state=0(ready) attempts=0 taken=False taken_at=None` |
| Scheduling / selection | ✓ | real `get_jobs_to_run()` selects the fresh `ready` job [`job_runner.py:307-326`] |
| Pickup (intermediate) | ✓ | after commit [`:341`]: `state=1(taken) attempts=1 taken=True taken_at=set` |
| Execution | ✓ | real `process_job` [`:342`]; failing job raises `AttributeError` at `app/import_utils.py:23` |
| Completion | ✓ | success control path → `state=2(done)` [`:344`] |
| Error recovery / retry behavior | ✓ | no `try/except`; retry only after process restart + `taken_at < now-30min` and `attempts < 5`; stops at `attempts >= 5` |
| Observable failure state | ✓ | job stays **`state=taken(1)` with `attempts` incremented — NOT `error(3)`**; real daemon exits with code 1 |
| End-of-life cleanup | ✓ | `cleanup_old_jobs` deletes `done`/`error`/`taken(attempts>=5)` [`tasks/cleanup_old_jobs.py:10-24`]; scheduled `"30 5 * * *"` [`crontab.yml:40-44`]; `job_runner` not in crontab |

### Q3 — VERP address format and direction-dependent bounce handling

| Named sub-part | Answered? | Where / observed value |
|---|---|---|
| VERP address format | ✓ | `{sl}.{base32(payload)}.{base32(hmac)}@{domain}`, lowercased [`app/email_utils.py:1460-1464`] |
| Forward format | ✓ | `sl.lmycy…@<contact-domain>` (`VerpType.bounce_forward=0`, call site `email_handler.py:905`) |
| Reply format | ✓ | `sl.lmysy…@<alias-domain>` (`VerpType.bounce_reply=1`, call site `email_handler.py:1225`) |
| HMAC signing | ✓ | first 8 bytes of HMAC `sha3-224` [`:1454-1456`]; tamper → `None` [`:1487-1491`] |
| Lifetime | ✓ | `VERP_MESSAGE_LIFETIME=5 days` [`app/config.py:499`]; over-age → `None` [`:1496-1497`] |
| Original-email identification | ✓ | `get_verp_info_from_email` → `(VerpType, email_log.id)`; e.g. `(<bounce_forward:0>, 987654)` |
| Legacy forms | ✓ | `parse_id_from_bounce("bounce+987654+@…")→987654`, `("bounce_reply+987654+@…")→987654` |
| State changes recorded | ✓ | `email_log.bounced=True`, `refused_email_id`, `bounced_mailbox_id`; `Bounce` + `RefusedEmail` rows |
| Direction difference (forward vs reply) | ✓ | forward→**E211**, `Bounce` keyed on **mailbox** email; reply→**E212**, `Bounce` keyed on **contact.website_email** + user `Notification` |
| Auto-reply exception | ✓ | non-`multipart/report` → `auto_replied=True` [`:1887`], re-forwarded (via un-gated caller `:2116`); standard reply-VERP path is guarded → **E213** |
| `E211` | ✓ | `'250 SL E211 Bounce Forward phase handled'` [`app/email/status.py:19`] |
| `E212` | ✓ | `'250 SL E212 Bounce Reply phase handled'` [`app/email/status.py:20`] |

### Method compliance

- **Canonical entry points exercised:** the real `verify_mailbox_code` + the real `GET /mailbox_verify` (Q1); the real `get_jobs_to_run`/`process_job` and the real `job_runner.py` `__main__` daemon (Q2); the real `generate_verp_email`/`get_verp_info_from_email`/`parse_id_from_bounce` and the real `email_handler.handle()`/`handle_DATA` (Q3). Where the loop body was reproduced (Q2 obs_q2.py), the *functions called* are the real ones and the loop is quoted verbatim; the definitive daemon-exit evidence comes from running the real `job_runner.py`.
- **Every claim** carries its command, complete unedited output, `file:line`, and an [OBSERVED]/[INFERRED] label. The only [INFERRED] item is that `check_bucket_limit` would raise `TooManyRequests` (it exists but is not wired to `/mailbox_verify`, so it could not be exercised on that route).
- **Stability:** each question's behavior was reproduced across **≥2 runs** with identical results (only auto-incremented ids/timestamps and the time-dependent VERP tail differ).
- **Read-only:** no existing repository file was modified; the only artifact added is this document. All temporary observation scripts were removed after evidence capture.
