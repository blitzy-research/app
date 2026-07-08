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
| **HEAD commit** | `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c` |
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
PostgreSQL 17.10 (Ubuntu 17.10-0ubuntu0.25.10.1) on x86_64-pc-linux-gnu, ...
$ pg_isready -h localhost -p 15432
localhost:15432 - accepting connections
$ redis-cli ping
PONG
$ redis-server --version
Redis server v=8.0.2
```

Mandatory configuration comes from `tests/test.env` (the project's own CI config, loaded by
`app/config.py`). Notably `DB_URI=postgresql://test:test@localhost:15432/test`
(`tests/test.env:17`), `MEM_STORE_URI=redis://localhost` (`tests/test.env:78`), and
`FLASK_SECRET=secret` (`tests/test.env:20`). `VERP_EMAIL_SECRET` is not set explicitly, so
`app/config.py:502-504` derives it as `FLASK_SECRET + "pleasegenerateagoodrandomtoken"` =
`"secretpleasegenerateagoodrandomtoken"` (36 chars), which passes the ≥32-char guard at
`app/config.py:505-508`. (The distributed template `example.env:75` ships
`DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin`.)

> **Note on credentials:** every secret shown in this document — `FLASK_SECRET=secret`, the derived
> `VERP_EMAIL_SECRET`, and the `test:test` database password — is a non-sensitive, placeholder value
> that already ships publicly in the repository's own `tests/test.env` (SimpleLogin is MIT-licensed
> open source). None is a real credential; they are reproduced only because the exact derived
> `VERP_EMAIL_SECRET` determines the VERP HMAC signatures observed in Question 3.

The schema was migrated with Alembic so every model is queryable:

```bash
$ CONFIG=tests/test.env alembic current
32f25cbf12f6 (head)
```

Runtime client-library versions (from the locked set) confirm the canonical stack:

```bash
$ python -c "import flask,sqlalchemy,arrow,aiosmtpd,boto3,redis; print('flask',flask.__version__,'| sqlalchemy',sqlalchemy.__version__,'| arrow',arrow.__version__,'| aiosmtpd',aiosmtpd.__version__,'| boto3',boto3.__version__,'| redis',redis.__version__)"
flask 1.1.2 | sqlalchemy 1.3.24 | arrow 0.16.0 | aiosmtpd 1.4.2 | boto3 1.35.37 | redis 4.6.0
```

Baseline git state (established before any observation, re-verified after cleanup — see the closing
section):

```bash
$ git rev-parse HEAD
2cd6ee777f8c2d3531559588bcfb18627ffb5d2c
$ git branch --show-current
blitzy-8e1c5efc-dd15-4f3e-81d1-742bff0fa928
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
created mailbox id=20, correct code='5KEjCxfXKBZY1uE2EsQj1g', submitting wrong code='5KEjCxfXKBZY1uE2EsQj1gWRONG'
INITIAL: MailboxActivation.tries=0  row_exists=True
... "app/mailbox_utils.py:206" - verify_mailbox_code() - ... failed to verify mailbox 20 because code does not match
attempt 1: BEFORE tries=0 exists=True  ->  CannotVerifyError(msg='Invalid activation code')  ->  AFTER tries=1 exists=True
... "app/mailbox_utils.py:206" - ... failed to verify mailbox 20 because code does not match
attempt 2: BEFORE tries=1 exists=True  ->  CannotVerifyError(msg='Invalid activation code')  ->  AFTER tries=2 exists=True
... "app/mailbox_utils.py:206" - ... failed to verify mailbox 20 because code does not match
attempt 3: BEFORE tries=2 exists=True  ->  CannotVerifyError(msg='Invalid activation code')  ->  AFTER tries=3 exists=True
... "app/mailbox_utils.py:196" - ... failed to verify mailbox 20 more than 3 times
attempt 4: BEFORE tries=3 exists=True  ->  CannotVerifyError(msg='Invalid activation code. Please request another code.')  ->  AFTER tries=None exists=False
... "app/mailbox_utils.py:191" - ... failed to verify mailbox 20 because there is no activation
attempt 5: BEFORE tries=None exists=False  ->  MailboxError('Invalid code')  ->  AFTER tries=None exists=False
```

Reading the transitions:

- **Attempts 1–3** each log `code does not match` (`app/mailbox_utils.py:206`) and increment the
  counter: `tries` `0→1→2→3`.
- **Attempt 4** finds `tries=3`, so the cap guard at `app/mailbox_utils.py:195` fires — it logs
  `... more than 3 times` (`app/mailbox_utils.py:196`), calls
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
(`app/dashboard/views/mailbox.py:121`). Driving it through the `flask_client` with a logged-in user
reproduces the identical counter progression; each response renders the validation page (HTTP 200)
with a flashed error:

```
===== Q1-B: REAL HTTP entry point GET /dashboard/mailbox_verify (canonical) =====
logged-in user=user_467d7zxue6@mailbox.test, mailbox id=22
GET http://sl.test/dashboard/mailbox_verify?mailbox_id=22&code=FCwNBo6aRDkNuuCwnyMzcwWRONG
  attempt 1: status=200 BEFORE tries=0 exists=True AFTER tries=1 exists=True flashed~='Invalid activation code'
  attempt 2: status=200 BEFORE tries=1 exists=True AFTER tries=2 exists=True flashed~='Invalid activation code'
  attempt 3: status=200 BEFORE tries=2 exists=True AFTER tries=3 exists=True flashed~='Invalid activation code'
  attempt 4: status=200 BEFORE tries=3 exists=True AFTER tries=None exists=False flashed~='Invalid activation code. Please request another code.'
  attempt 5: status=200 BEFORE tries=None exists=False AFTER tries=None exists=False flashed~='Invalid code'
```

The flashed text on attempt 4 (`Invalid activation code. Please request another code.`) and the row
disappearing (`exists=False`) confirm the same enforcement fires through the real web route.

### Sibling variant — the signed-link "Old way" path

When the route is hit **without** a `code` parameter it takes the signed-link branch
`return verify_with_signed_secret(mailbox_id)` (`app/dashboard/views/mailbox.py:127`). That function
is declared `def verify_with_signed_secret(request: str)` (`app/dashboard/views/mailbox.py:138`) and
immediately does `mailbox_verify_request = request.args.get("mailbox_id")`
(`app/dashboard/views/mailbox.py:140`).

**Observed (unexpected but real):** the canonical route returns **HTTP 500**, because the parameter
name `request` shadows Flask's global `request`, and the *string* the route passed in has no `.args`
attribute:

```
===== Q1-C: signed-link 'Old way' verify_with_signed_secret() (max_age=900 -> 15 min) =====
mailbox id=24 verified_before=False
... server.py:390 - error_handler() - 'str' object has no attribute 'args'
Traceback (most recent call last):
  ...
  File ".../app/dashboard/views/mailbox.py", line 127, in mailbox_verify
    return verify_with_signed_secret(mailbox_id)
  File ".../app/dashboard/views/mailbox.py", line 140, in verify_with_signed_secret
    mailbox_verify_request = request.args.get("mailbox_id")
AttributeError: 'str' object has no attribute 'args'
(i) CANONICAL route GET (no code): status=500
(ii) verify_with_signed_secret(<str>) [as route does @L127]: AttributeError: 'str' object has no attribute 'args'
```

When the function is instead handed a real Flask request object (a `[NON-CANONICAL]` direct call, to
reveal the intended mechanics), the signed link verifies the mailbox, and the 15-minute window is
enforced by `TimestampSigner(MAILBOX_SECRET).unsign(..., max_age=900)`
(`app/dashboard/views/mailbox.py:139,142`):

```
(iii) [NON-CANONICAL] verify_with_signed_secret(flask_request): resp_type=str verified_after=True
(iv) unsign(max_age=900) fresh token -> OK, recovered b'WzI0LCAibmlwcHlsd2F6cnFs'...
(iv) unsign(max_age=900) on 1000s-old token -> SignatureExpired: Signature age 1000 > 900 seconds
```

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
AccountActivation created with tries=3 (model default=3)
attempt 1: BEFORE tries=3 -> POST /api/auth/activate wrong code -> status=400 body={'error': 'Wrong email or code'} -> AFTER tries=2 exists=True
attempt 2: BEFORE tries=2 -> POST /api/auth/activate wrong code -> status=400 body={'error': 'Wrong email or code'} -> AFTER tries=1 exists=True
attempt 3: BEFORE tries=1 -> POST /api/auth/activate wrong code -> status=410 body={'error': 'Too many wrong tries'} -> AFTER tries=None exists=False
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
_route_limits['app.dashboard.views.mailbox_detail.mailbox_detail_route']: present=True limits=['<flask_limiter.wrappers.Limit object at 0x...>']
_route_limits['app.api.views.auth.auth_activate']: present=True limits=['<flask_limiter.wrappers.Limit object at 0x...>']
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

`JobState` (`app/models.py:253`) defines `ready = 0` (`:254`), `taken = 1` (`:255`),
`done = 2` (`:256`), `error = 3` (`:257`). The runner's `__main__` loop (`job_runner.py:329-347`)
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

A job named `batch-import` (`config.JOB_BATCH_IMPORT`, `app/config.py:305`) with a nonexistent
`batch_import_id` reaches the dispatch branch (`job_runner.py:222-225`), where
`BatchImport.get(bad_id)` returns `None` and `handle_batch_import(None)` dereferences
`batch_import.user` (`app/import_utils.py:23`) → `AttributeError`. Driving the real entry point:

```bash
$ python /tmp/blitzy_evidence/q2_helper.py insert_fail    # commit a ready job
inserted FAIL job (name=batch-import, bad batch_import_id): id=74 name='batch-import' state=0(ready) attempts=0 taken=False taken_at=None
$ python /tmp/blitzy_evidence/q2_helper.py show_all        # BEFORE pickup
   id=74 name='batch-import' state=0(ready) attempts=0 taken=False taken_at=None
$ timeout 30 python job_runner.py                          # the REAL runner __main__ loop
```

```
... "job_runner.py:334" - Take job <Job 74 batch-import {'batch_import_id': 99999999}>
Traceback (most recent call last):
  File ".../job_runner.py", line 342, in <module>
    process_job(job)
  File ".../job_runner.py", line 225, in process_job
    handle_batch_import(batch_import)
  File ".../app/import_utils.py", line 23, in handle_batch_import
    user = batch_import.user
AttributeError: 'NoneType' object has no attribute 'user'
job_runner.py exit code = 1   # process CRASHED (not a timeout kill)
```

The process exits with code **1** (a genuine crash — a timeout kill would be 124), confirming there
is no error handling around `process_job`. Querying the row **after** the crash:

```bash
$ python /tmp/blitzy_evidence/q2_helper.py show_all        # AFTER crash
   id=74 name='batch-import' state=1(taken) attempts=1 taken=True taken_at=2026-07-08T05:11:23.487642+00:00
```

**Observable failure state:** `state=1(taken)`, `attempts=1`, `taken=True`, `taken_at` set, and
`state` is **never** `2(done)` and **never** `3(error)`.

### Success path — a job that completes reaches `done (2)`

An unknown job name falls through to `else: LOG.e("Unknown job name %s", job.name)`
(`job_runner.py:303-304`), which does **not** raise, so the loop proceeds to `state = done`:

```bash
$ python /tmp/blitzy_evidence/q2_helper.py insert_success   # id=75, state=ready
$ timeout 9 python job_runner.py
... "job_runner.py:334" - Take job <Job 75 blitzy-nonexistent-job {}>
... "job_runner.py:304" - process_job() - Unknown job name blitzy-nonexistent-job
job_runner.py exit code = 124                               # timeout kill (loop kept running; no crash)
$ python /tmp/blitzy_evidence/q2_helper.py show_all         # AFTER
   id=75 name='blitzy-nonexistent-job' state=2(done) attempts=1 taken=True taken_at=2026-07-08T05:11:54.857124+00:00
```

So the cross-product is observed: **failure ⇒ stuck in `taken`; success ⇒ `done`.**

### Retry eligibility — the real `get_jobs_to_run()` across the 30-min window and 5-attempt cap

`get_jobs_to_run()` (`job_runner.py:307-326`) re-selects a job when
`state == ready` **OR** (`state == taken` **AND** `taken_at < now - 30 min` **AND** `attempts < 5`),
further gated by `run_at` being null or within `now + 10 min` (`job_runner.py:313-323`). The
constants were confirmed at runtime:

```bash
$ python -c "from app import config; print(config.JOB_MAX_ATTEMPTS, config.JOB_TAKEN_RETRY_WAIT_MINS)"
5 30
```

Driving the real query against one job (id=77), reporting eligibility **before/after** each change
(`taken_at` back-dating and `attempts` edits are labeled `[TEST FIXTURE]` manipulations of scratch
DB state, not source changes):

```
READY job (id=76):                         get_jobs_to_run() -> 1 job(s) ids=[76]
taken 5 min ago, attempts=1:               get_jobs_to_run() -> 0 job(s) ids=[]      # within 30-min window
[TEST FIXTURE] backdate id=77 to 31 min:   get_jobs_to_run() -> 1 job(s) ids=[77]    # past window, attempts<5
[TEST FIXTURE] set id=77 attempts=5:       get_jobs_to_run() -> 0 job(s) ids=[]      # attempts>=5 -> ABANDONED
[TEST FIXTURE] set id=77 attempts=4:       get_jobs_to_run() -> 1 job(s) ids=[77]    # boundary: 4<5 -> eligible
```

This directly demonstrates: a fresh `taken` job is **not** retried within 30 minutes; it becomes
eligible once `taken_at` crosses the window; and it is **abandoned** at `attempts >= 5` (the
boundary `4 < 5` is eligible, `5` is not).

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
that invoke `python /code/cron.py -j <name>` (e.g. `stats` on `0 0 * * *`, `delete_old_monitoring`,
`check_custom_domain`, `check_hibp`, `notify_hibp`, `delete_logs`, `delete_old_data`,
`poll_apple_subscription`, `notify_trial_end`, `notify_manual_subscription_end`, …), and `cron.py`
dispatches via `argparse -j`. This is unrelated to the `Job` DB table that Question 2 concerns.


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
  payload, recomputes and compares the HMAC (`app/email_utils.py:1490`), enforces a 5-day lifetime
  (`app/email_utils.py:1496`; `VERP_MESSAGE_LIFETIME = 432000` = 5 days, `app/config.py:499`), and
  returns `(VerpType, EmailLog.id)`. A legacy `bounce+<id>+@` form is decoded by
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
(`:1446-1450`), signs it with `hmac.new(secret, payload, "sha3-224").digest()[:8]` (`:1454-1456`),
base32-encodes both with the `=` padding stripped (`:1457-1458`), and returns:

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
  EXACT ADDRESS: sl.lmycyibrgaycyibsgm3tiobxgzoq.ataxfel77s3e4@sl.local
  decoded payload list [verp_type, object_id, minutes_since_2022-01-01] = [0, 100, 2374876]
  EXACT ADDRESS: sl.lmysyibrgaycyibsgm3tiobxgzoq.5lpsjv5ibnoou@sl.local
  decoded payload list [verp_type, object_id, minutes_since_2022-01-01] = [1, 100, 2374876]
  EXACT ADDRESS: sl.lmzcyibrgaycyibsgm3tiobxgzoq.it4x2ebnfdnja@sl.local
  decoded payload list [verp_type, object_id, minutes_since_2022-01-01] = [2, 100, 2374876]
```

The three addresses share prefix `sl.`, the same trailing `@sl.local`, and the same time term
(`2374876` minutes since `VERP_TIME_START`); they differ only in the leading payload byte
(`verp_type` 0/1/2) and the resulting signature — exactly matching the
`sl.<payload>.<sig>@<domain>` format.

### (b) Inbound decode — `get_verp_info_from_email()` (`app/email_utils.py:1467`)

Round-tripping freshly generated addresses (this time with `object_id=4242`) recovers the exact
`(VerpType, id)` tuple; a tampered signature and a random address both return `None`:

```
bounce_forward: address=sl.lmycyibugi2delbagiztonbyg43f2.juu7qmd3kt6pm@sl.local
   -> get_verp_info_from_email() = (<VerpType.bounce_forward: 0>, 4242)  (match: type=True id=True)
bounce_reply: address=sl.lmysyibugi2delbagiztonbyg43f2.oaeyfaex4ergm@sl.local
   -> get_verp_info_from_email() = (<VerpType.bounce_reply: 1>, 4242)  (match: type=True id=True)
transactional: address=sl.lmzcyibugi2delbagiztonbyg43f2.kpgyfke63drao@sl.local
   -> get_verp_info_from_email() = (<VerpType.transactional: 2>, 4242)  (match: type=True id=True)
tampered signature: sl.lmycyibvguwcamrtg42dqnzwlu.ggyh6cd46b7ta@sl.local
   -> get_verp_info_from_email() = None (expect None; HMAC mismatch @L1490)
random@sl.local -> None (expect None)
VERP_MESSAGE_LIFETIME=432000s = 5 days (lifetime gate @app/email_utils.py:L1496)
```

This confirms the HMAC comparison at `app/email_utils.py:1490` (a tampered signature yields `None`)
and the 5-day lifetime gate at `app/email_utils.py:1496`.

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
`BOUNCE_PREFIX = "bounce+"` (`:100`), `BOUNCE_SUFFIX = "+@{EMAIL_DOMAIN}"` (`:101`),
`BOUNCE_PREFIX_FOR_REPLY_PHASE = "bounce_reply"` (`:108`), and
`TRANSACTIONAL_BOUNCE_PREFIX = "transactional+"` (`:113`). The reply-phase and transactional legacy
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
===== Q3 (c)/(d) FORWARD phase — fresh alias (should_disable False) =====
EmailLog id=1 is_reply=False get_phase()='forward'
should_disable(alias)=(False, '')
RETURN='250 SL E211 Bounce Forward phase handled'   (E211='250 SL E211 Bounce Forward phase handled')
AFTER: bounced=True refused_email_id=1 bounced_mailbox_id=52 (mailbox.id=52) alias.enabled=True
Bounce row: email='fybicrklgpkykpryqwsx@fybicrklgpkykpryqwsx.com'  (== mailbox.email 'fybicrklgpkykpryqwsx@fybicrklgpkykpryqwsx.com')
alerts (envelope_to, subject): [('user_qb03dbgaem@mailbox.test', 'An email sent to trumps_carted827@sl.local cannot be delivered to your mailbox')]
```

Forward phase returns **`E211`**, sets `Bounce.email == mailbox.email`, sets the `EmailLog` flags
`bounced=True / refused_email_id / bounced_mailbox_id`, and alerts the **user's** address. With
`should_disable=(False, '')` the alias stays enabled (`alias.enabled=True`).

**Forward phase — alias with >12 prior bounces (`should_disable` True) → alias auto-disabled:**

```
===== Q3 (c)/(d) FORWARD phase — alias with >12 prior bounces (should_disable True) =====
should_disable(alias)=(True, '+12 bounces in the last 24h')  alias.enabled(before)=True
RETURN='250 SL E211 Bounce Forward phase handled' (E211)  alias.enabled(after)=False  (expect disabled=False)
alerts (envelope_to, subject): [('user_qh1qk8luce@mailbox.test', 'Alias voiced_soviet838@sl.local has been disabled due to multiple bounces')]
```

After seeding 13 prior bounces, `should_disable(alias)` (`app/email_utils.py:1166`) returns
`(True, '+12 bounces in the last 24h')` (the `nb_bounced_last_24h > 12` branch,
`app/email_utils.py:1190`), so the alias transitions `enabled: True → False` and the alert subject
changes to the "disabled due to multiple bounces" template. (Requires `ALIAS_AUTOMATIC_DISABLE=true`,
present in `tests/test.env:62`.)

**Reply phase (multipart/report DSN with `MAIL FROM:<>`):**

```
===== Q3 (c)/(d) REPLY phase (multipart/report DSN, MAIL FROM:<>) =====
EmailLog id=16 is_reply=True get_phase()='reply'
RETURN='250 SL E212 Bounce Reply phase handled'   (E212='250 SL E212 Bounce Reply phase handled')
AFTER: bounced=True refused_email_id=3 bounced_mailbox_id=56 (mailbox.id=56) alias.enabled=True (expect True, NO disable)
Bounce row: email='isudjmtarqiewlrxtajf@isudjmtarqiewlrxtajf.com'  (== contact.website_email 'isudjmtarqiewlrxtajf@isudjmtarqiewlrxtajf.com')
alerts (envelope_to, subject): [('ocbbanwrvbgjaomaqmln@ocbbanwrvbgjaomaqmln.com', 'Email cannot be sent to isudjmtarqiewlrxtajf@isudjmtarqiewlrxtajf.com from your alias masker_beamed415@sl.local')]
```

Reply phase returns **`E212`**, sets `Bounce.email == contact.website_email` (not the mailbox),
alerts the **mailbox** address, and — critically — leaves `alias.enabled=True` (**no auto-disable**
in the reply direction).

**Reply-phase auto-reply RE-FORWARD branch (message that is NOT a real DSN):** when the inbound
message's `content_type != "multipart/report"` **or** `envelope.mail_from != "<>"`
(`email_handler.py:1876`), it is treated as an auto-reply and re-forwarded instead of processed as a
bounce:

```
===== Q3 (d) REPLY phase auto-reply RE-FORWARD branch (NOT a DSN) @L1876 =====
RETURN='250 Message accepted for delivery'
AFTER: auto_replied=True (set True @L1887 before re-forward)
```

Here `EmailLog.auto_replied` is set `True` (`email_handler.py:1887`), the `To` header is rewritten to
`alias.email` (`email_handler.py:1891`), the message is re-forwarded via `handle_forward()`
(`email_handler.py:1899`), and the return is `250 Message accepted for delivery` — **not** a bounce
status.

### Edge statuses — `E512` (unknown email log) and `E510` (inactive user)

```
handle_bounce(env, None, msg) -> '550 SL E512 No such email log' (E512='550 SL E512 No such email log')
user.is_active()=False (delete_on set 1h in future)
handle_bounce(env, el, msg) -> '550 SL E510 so such user' (E510='550 SL E510 so such user')
```

A missing `EmailLog` yields `550 SL E512` (`email_handler.py:1856-1858`); an inactive user
(`User.is_active()` returns `False` when `delete_on` is set to a future time, `app/models.py:766`)
yields `550 SL E510` (`email_handler.py:1869-1871`).

### Direction discriminator — `EmailLog.get_phase()` (`app/models.py:2143-2147`)

```
EmailLog(is_reply=False).get_phase()='forward'; EmailLog(is_reply=True).get_phase()='reply'
```

`get_phase()` returns `"reply"` when `self.is_reply` is set, else `"forward"`
(`app/models.py:2143-2147`) — the value `handle_bounce()` keys on to choose the phase handler.

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
| lifetime gate | 5 days (`VERP_MESSAGE_LIFETIME=432000`) | `app/config.py:499`; `app/email_utils.py:1496` | ✅ printed |
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
temporary, disposable scripts. The temporary observation scripts
(`tests/blitzy_adhoc_test_q1.py`, `tests/blitzy_adhoc_test_q3_verp.py`,
`tests/blitzy_adhoc_test_q3_bounce.py`) and the out-of-tree helper
(`/tmp/blitzy_evidence/q2_helper.py`) were deleted after evidence capture; the scratch PostgreSQL
rows created during the investigation are transient test-DB state outside the repository tree. No
repository source file was modified, added to, or deleted — the **only** net-new artifact is this
document.

The final working-tree state confirming the read-only guarantee (captured after deleting every
temporary script and the Redis `dump.rdb` artifact, before committing the deliverable):

```bash
$ git rev-parse HEAD
2cd6ee777f8c2d3531559588bcfb18627ffb5d2c
$ git status --porcelain
?? blitzy/
$ git status --porcelain -uall     # -uall expands the untracked directory to its files
?? blitzy/documentation/app_2cd6ee777f8c.md
```

The sole change in the working tree is the new file `blitzy/documentation/app_2cd6ee777f8c.md`
(plain `--porcelain` collapses it to the untracked directory `blitzy/`; `-uall` expands it to the
exact file), confirming the read-only constraint was honored — no existing repository source file
was modified, added to, or deleted.

