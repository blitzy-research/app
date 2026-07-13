# SimpleLogin Runtime Behavior Investigation — `app_2cd6ee777f8c`

**Repository:** SimpleLogin (`app/`) • **Branch:** `app_2cd6ee777f8c` • **Method:** runtime observation inside the canonical Docker container.

This document answers three runtime-behavior questions about the SimpleLogin
codebase. Every behavioral claim was produced by running the real code path
through its canonical entry point and capturing the actual, unedited output —
not by reading source alone. Each claim carries (1) the exact command, (2) the
complete unedited output of that command, (3) a `file:line` reference, and
(4) an explicit evidence label (defined below).

> **Path note (source-verified):** the SMTP inbound handler is at the
> **repository root** — `email_handler.py`. There is **no** `app/email_handler.py`.
> All Q3 handler references point at the root file.

## The three questions (verbatim)

1. **Q1 —** When a mailbox verification code is submitted incorrectly several
   times in succession, what observable state changes and enforcement
   mechanisms does the running system apply? Trace (a) what limit(s) exist,
   (b) how failed attempts are tracked (which counter/state field, and its
   progression), and (c) what condition ultimately prevents further submissions.

2. **Q2 —** For background tasks that are scheduled and later picked up for
   execution, trace the complete lifecycle from initial creation through final
   completion. When a task encounters an error during execution, determine
   (a) what recovery or retry behavior is applied, and (b) what observable state
   reflects that failure.

3. **Q3 —** During email forwarding through an alias, what is the exact
   **format** of the special address generated to handle delivery failures
   (the bounce / return-path / VERP address)? When a failure notification
   arrives at this address, trace (a) how the system identifies the original
   email, (b) what state changes are recorded, and (c) how the handling behavior
   **differs depending on the direction** of the original message (forward
   versus reply).

---

## Evidence labels (how to read this document)

Every claim below is tagged with exactly one of three labels. This distinction
is applied deliberately: printing source with `sed`/`grep` is **not** a runtime
observation and is never labeled `[OBSERVED]`.

- **`[OBSERVED]`** — the value or behavior was produced by **executing the real
  code path at runtime**, and the captured stdout/stderr/database state is shown
  verbatim.
- **`[SOURCE-VERIFIED]`** — the statement is a structural fact confirmed by
  **reading the source** (e.g. a decorator list, an exact source line, a `grep`
  that returns nothing). It is corroborating, not a runtime observation.
- **`[INFERRED]`** — a conclusion drawn from reading that was **not exercised at
  runtime** (for example, a code path that exists but is not wired to the entry
  point under test). Inferred claims are prefer-run wherever possible; where a
  path genuinely could not be exercised, that is stated.

Every scratch script used below is reproduced **in full** inside this document
(so the setup, authentication, monkeypatches, filtering, and assertions are all
auditable), even though the temporary files themselves were deleted afterwards
(see the final "Repository integrity & cleanup" section). Where a command
applies a filter (e.g. `grep -v` to drop a specific noisy log line), the filter
is shown **as part of the command**, so the displayed output is the complete
output *of the displayed command*.

---

## Environment (canonical container) — proof

All observations were produced inside the canonical container `sl_canon`. The
next three blocks prove the container's identity (the required GHCR image), the
project virtual-env / dependency context, and the running data services. The
host checkout is a different Python with no app dependencies and is **not**
canonical; it is used only to hold this document.

### Proof 1 — the container runs the required GHCR image (`docker inspect`)

**Command (run on the host):**

```bash
docker inspect sl_canon --format 'container={{.Name}} state={{.State.Status}} running={{.State.Running}}
image_ref={{.Config.Image}}
image_id={{.Image}}'
docker image inspect ea242796bbce --format '{{range .RepoTags}}{{.}}{{end}}  id={{.Id}}'
```

**Complete unedited output:**

```
container=/sl_canon state=running running=true
image_ref=ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0
image_id=sha256:ea242796bbce36ca99ba9f783a4e7ac9d2ed3738e737f9a6d96bb22bbf1d9b58
ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0  id=sha256:ea242796bbce36ca99ba9f783a4e7ac9d2ed3738e737f9a6d96bb22bbf1d9b58
```

The container is `Up`/`running` and its image is exactly
`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`
(id `sha256:ea242796bbce…`). **[OBSERVED]**

### Proof 2 — the canonical project virtual-env and pinned dependencies

**Command:**

```bash
docker exec sl_canon bash -lc '
cd /app
/app/venv/bin/python --version
/app/venv/bin/python -c "import sys; print(\"executable:\", sys.executable); print(\"prefix:\", sys.prefix)"
ls -l /app/venv/bin/python
/app/venv/bin/python -c "import flask, sqlalchemy, aiosmtpd, flanker; print(\"flask\", flask.__version__, \"| sqlalchemy\", sqlalchemy.__version__, \"| aiosmtpd\", aiosmtpd.__version__)"
python3 -c "import flask" 2>&1 | tail -1
git rev-parse HEAD
grep -E "^name|^version|^python " pyproject.toml | head -5'
```

**Complete unedited output:**

```
Python 3.10.18
executable: /app/venv/bin/python
prefix: /app/venv
lrwxrwxrwx 1 root 1001 7 Aug 26  2025 /app/venv/bin/python -> python3
flask 1.1.2 | sqlalchemy 1.3.24 | aiosmtpd 1.4.2
ModuleNotFoundError: No module named 'flask'
2cd6ee777f8c2d3531559588bcfb18627ffb5d2c
name = "SimpleLogin"
version = "0.1.0"
python = "^3.10"
```

The runtime is the project venv at `/app/venv` (Python 3.10.18), holding the
pinned deps (`flask 1.1.2`, `sqlalchemy 1.3.24`, `aiosmtpd 1.4.2`) that match
`poetry.lock`; the **bare** `python3` cannot import `flask`, confirming every
run below must use `/app/venv/bin/python`. `pyproject.toml` declares the project
`SimpleLogin` with `python = "^3.10"`. Git HEAD is
`2cd6ee777f8c…`, matching the source branch name `app_2cd6ee777f8c`. **[OBSERVED]**

### Proof 3 — data services and the runtime constants the answers depend on

**Command:**

```bash
docker exec sl_canon bash -lc '
cd /app
service postgresql status | head -1
service redis-server status | head -1
redis-cli ping
export CONFIG=tests/test.env PYTHONPATH=/app DB_URI="postgresql://test:test@localhost:5432/test"
/app/venv/bin/python - <<PY
import psycopg2, os
c=psycopg2.connect("postgresql://test:test@localhost:5432/test"); cur=c.cursor()
cur.execute("select count(*) from information_schema.tables where table_schema=%s",("public",))
print("public tables:", cur.fetchone()[0])
cur.execute("select version()"); print(cur.fetchone()[0].split(",")[0])
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
print("VERP_MESSAGE_LIFETIME =", config.VERP_MESSAGE_LIFETIME, "(=", config.VERP_MESSAGE_LIFETIME/86400, "days)")
print("NOT_SEND_EMAIL =", config.NOT_SEND_EMAIL)
print("LOCAL_FILE_UPLOAD =", config.LOCAL_FILE_UPLOAD)
print("UPLOAD_DIR =", config.UPLOAD_DIR)
print("DISABLE_RATE_LIMIT env =", os.environ.get("DISABLE_RATE_LIMIT"), "| config =", config.DISABLE_RATE_LIMIT)
PY'
```

**Complete unedited output** (the `load config file …`, `>>> URL:`, GNUPGHOME
`WARNING`, `>>> init logging <<<`, and `SL - DEBUG … load words file` lines are
the application's own startup output, shown verbatim):

```
15/main (port 5432): online
redis-server is running.
PONG
public tables: 77
PostgreSQL 15.13 (Debian 15.13-0+deb12u1) on x86_64-pc-linux-gnu
load config file /app/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/zuetylxjmaxbkcyqquil
Upload files to local dir
>>> init logging <<<
2026-07-13 18:04:54,905 - SL - DEBUG - 4554 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
imports OK: app.mailbox_utils, job_runner, app.email_utils, email_handler
MAX_ACTIVATION_TRIES = 3
JOB_MAX_ATTEMPTS = 5
JOB_TAKEN_RETRY_WAIT_MINS = 30
VERP_PREFIX = 'sl'
EMAIL_DOMAIN = 'sl.local'
VERP_HMAC_ALGO = 'sha3-224'
VERP_TIME_START = 1640995200
VERP_MESSAGE_LIFETIME = 432000 (= 5.0 days)
NOT_SEND_EMAIL = True
LOCAL_FILE_UPLOAD = True
UPLOAD_DIR = /app/static/upload
DISABLE_RATE_LIMIT env = None | config = False
```

Postgres 15.13 is online with the full 77-table schema; Redis answers `PONG`;
the three subsystems import cleanly; and every constant the answers rely on is
read from the running process. **[OBSERVED]**

### Canonical run environment and its transparent substitutes (boundary disclosure)

Every script and test below exports this environment first:

```bash
export CONFIG=tests/test.env PYTHONPATH=/app DB_URI="postgresql://test:test@localhost:5432/test"
# every Python is invoked as /app/venv/bin/python (the venv holds the pinned deps)
```

The default test configuration (`tests/test.env`) enables two **transparent
substitutes** that bound what "sending" and "storing" mean in this
investigation. These are the standard, canonical test-config values — not
mocks injected by this document — but every downstream claim is scoped to them:

- **`NOT_SEND_EMAIL=true`.** `MailSender.send()` does **not** open an SMTP
  connection; it logs the message and returns `True`
  [`app/mail_sender.py:130`]. So wherever this document says a message was
  "re-forwarded" or "sent", what is **observed** is that the app reached the
  send/dispatch call successfully — **not** real outbound SMTP delivery.
- **`LOCAL_FILE_UPLOAD=1`.** S3 is replaced by local-disk storage:
  `upload_from_bytesio()` writes under `UPLOAD_DIR = /app/static/upload`
  instead of calling `boto3` [`app/s3.py:31`]. So wherever this document says a
  `RefusedEmail` was "stored", what is **observed** is a local file on disk (its
  path is printed), **not** an S3 object.

These two lines are the only substitution of external I/O; all in-process logic
(routing, state machine, HMAC, DB writes) is the real canonical code.

**`DISABLE_RATE_LIMIT` — precise statement (correcting a common misreading).**
`tests/test.env` does **not** set `DISABLE_RATE_LIMIT`; at runtime
`os.environ["DISABLE_RATE_LIMIT"]` is `None` and `config.DISABLE_RATE_LIMIT` is
`False` (shown in Proof 3 above). Rate limiting is disabled **only inside the
Flask test client**, by the canonical `flask_client` fixture which executes
`config.DISABLE_RATE_LIMIT = True` [`tests/conftest.py`]. The Q1 HTTP script
below performs the **same** documented override and labels it explicitly, so the
login route's limiter does not interfere with the multi-request test.
**[OBSERVED]** (env/config values) / **[SOURCE-VERIFIED]** (fixture behavior).

---

## Q1 — Mailbox verification-code enforcement under repeated incorrect attempts

### Direct answer

**The limit.** `MAX_ACTIVATION_TRIES = 3` [`app/mailbox_utils.py:43`].
**[OBSERVED]** (printed at runtime).

**How failed attempts are tracked.** The integer column
`MailboxActivation.tries` (default `0`) [`app/models.py:2835`] on the latest
activation row for the mailbox. **[SOURCE-VERIFIED]** (model column) /
**[OBSERVED]** (its runtime progression, below).

**The function that enforces it.** `verify_mailbox_code(user, mailbox_id, code)`
[`app/mailbox_utils.py:166`], which the canonical route
`GET /dashboard/mailbox_verify` calls directly at
[`app/dashboard/views/mailbox.py:129`]. **[SOURCE-VERIFIED]**

**Progression and the condition that stops submissions.** On each wrong code,
`tries` is incremented, committed, and `CannotVerifyError("Invalid activation
code")` is raised [`app/mailbox_utils.py:209-211`]. The observed progression is
`tries` **0 → 1 → 2 → 3**. The condition that ultimately prevents further
submissions is the guard at the **top** of the function: once
`activation.tries >= MAX_ACTIVATION_TRIES`, it deletes **all** activation rows
for the mailbox via `clear_activation_codes_for_mailbox`
[`app/mailbox_utils.py:197`] and raises `CannotVerifyError("Invalid activation
code. Please request another code.")` [`app/mailbox_utils.py:198`]. A subsequent
submission then finds no activation and raises `MailboxError("Invalid code")`
[`app/mailbox_utils.py:194`]. **[OBSERVED]**

**Enforcement is a per-activation counter plus code deletion — not an HTTP 429.**
`/dashboard/mailbox_verify` (blueprint-local rule `/mailbox_verify`, prefixed by
the dashboard blueprint's `url_prefix="/dashboard"` [`app/dashboard/base.py:6`])
carries **no** rate-limiter decorator, unlike the `/mailbox` POST route. Every
wrong submission returns **HTTP 302**, never 429. **[OBSERVED]**

> **Note on route naming (consistency).** The externally reachable URL is
> `/dashboard/mailbox_verify`. The blueprint-local route *rule* is
> `/mailbox_verify`; this document uses the full external path
> `/dashboard/mailbox_verify` throughout.

### Evidence A — the real `verify_mailbox_code` (primary + all branches)

Script `obs_q1_direct.py` (reproduced in full) creates one throwaway premium
user, then drives the **exact** function the route calls, reading
`MailboxActivation.tries` from the DB after each call. It is cleaned up by
deleting the user (cascade), and prints before/after row counts to prove the run
is net-zero.

```python
"""Q1 - drive the REAL verify_mailbox_code enforcement function.

Isolation strategy: a single throwaway premium User is created; all scenarios
attach to it; at the end the User is deleted (ON DELETE CASCADE removes its
mailboxes / activations / aliases). Before/after row counts prove the run is
net-zero, so the database is left unchanged (auditable cleanup)."""
from server import create_app
app = create_app()
from app.db import Session
from app.models import User, Mailbox, MailboxActivation
from tests.utils import create_new_user
import app.mailbox_utils as mu
import arrow

def counts():
    return (Session.query(User).count(),
            Session.query(Mailbox).count(),
            Session.query(MailboxActivation).count())

def tries_of(mb_id):
    a = (MailboxActivation.filter(MailboxActivation.mailbox_id == mb_id)
         .order_by(MailboxActivation.created_at.desc()).first())
    return None if a is None else a.tries

def act_count(mb_id):
    return MailboxActivation.filter(MailboxActivation.mailbox_id == mb_id).count()

print("MAX_ACTIVATION_TRIES =", mu.MAX_ACTIVATION_TRIES)
with app.app_context():
    before = counts()
    user = create_new_user()
    user.lifetime = True          # premium, so create_mailbox is allowed
    Session.commit()
    uid = user.id
    try:
        # ---- SCENARIO A: repeated WRONG code -> tries progression + lockout ----
        print()
        print("=== SCENARIO A: repeated WRONG code -> tries progression + lockout ===")
        mb_id = mu.create_mailbox(user, "q1_scenarioA@example.com", send_email=False).mailbox.id
        act = (MailboxActivation.filter(MailboxActivation.mailbox_id == mb_id)
               .order_by(MailboxActivation.created_at.desc()).first())
        print("mailbox_id =", mb_id)
        print("correct code =", repr(act.code))
        print("BEFORE any submission: tries =", tries_of(mb_id))
        for i in range(1, 4):
            try:
                mu.verify_mailbox_code(user, mb_id, "wrong-code-%d" % i)
            except mu.MailboxError as e:
                print("submission %d: %s(msg=%r) -> tries now = %s"
                      % (i, type(e).__name__, e.msg, tries_of(mb_id)))
        print("--- 4th submission (tries>=3 triggers lockout) ---")
        try:
            mu.verify_mailbox_code(user, mb_id, "wrong-code-4")
        except mu.MailboxError as e:
            print("submission 4: %s(msg=%r)" % (type(e).__name__, e.msg))
        print("activation row after lockout:", tries_of(mb_id))
        print("activation row count after lockout:", act_count(mb_id))
        print("--- 5th submission (no activation -> MailboxError) ---")
        try:
            mu.verify_mailbox_code(user, mb_id, "wrong-code-5")
        except mu.MailboxError as e:
            print("submission 5: %s(msg=%r)" % (type(e).__name__, e.msg))

        # ---- SCENARIO B: 15-minute expiry edge branch ----
        print()
        print("=== SCENARIO B: 15-minute expiry edge branch ===")
        mb2_id = mu.create_mailbox(user, "q1_scenarioB@example.com", send_email=False).mailbox.id
        act2 = (MailboxActivation.filter(MailboxActivation.mailbox_id == mb2_id)
                .order_by(MailboxActivation.created_at.desc()).first())
        backdated = arrow.now().shift(minutes=-16)
        act2.created_at = backdated
        Session.commit()
        print("mailbox_id =", mb2_id, "| tries =", tries_of(mb2_id),
              "| created_at backdated to", backdated.isoformat())
        try:
            mu.verify_mailbox_code(user, mb2_id, act2.code)   # even the CORRECT code
        except mu.MailboxError as e:
            print("expiry: %s(msg=%r)" % (type(e).__name__, e.msg))
        print("activation row count after expiry:", act_count(mb2_id))

        # ---- SCENARIO C: success control (correct code) ----
        print()
        print("=== SCENARIO C: success control path (correct code) ===")
        mb3 = mu.create_mailbox(user, "q1_scenarioC@example.com", send_email=False).mailbox
        mb3_id = mb3.id
        act3 = (MailboxActivation.filter(MailboxActivation.mailbox_id == mb3_id)
                .order_by(MailboxActivation.created_at.desc()).first())
        print("mailbox_id =", mb3_id, "| verified BEFORE =", mb3.verified,
              "| activation count BEFORE =", act_count(mb3_id))
        out = mu.verify_mailbox_code(user, mb3_id, act3.code)
        Session.refresh(out)
        print("verify returned Mailbox id =", out.id)
        print("verified AFTER =", out.verified, "| activation count AFTER =", act_count(mb3_id))
    finally:
        # ---- auditable cleanup: delete the throwaway user (cascades) ----
        User.delete(uid)
        Session.commit()
        after = counts()
        print()
        print("=== cleanup: deleted user %d; (User,Mailbox,MailboxActivation) "
              "before=%s after=%s net-zero=%s ===" % (uid, before, after, before == after))
```

**Command:**

```bash
docker exec sl_canon bash -lc '
export CONFIG=tests/test.env PYTHONPATH=/app DB_URI="postgresql://test:test@localhost:5432/test"
cd /app && /app/venv/bin/python /tmp/blitzy_obs/obs_q1_direct.py'
```

**Complete unedited output — RUN 1** (the `SL - INFO/DEBUG` lines are the app's
own logger, shown verbatim; each pinpoints the branch that fired via its
`file:line`):

```
load config file /app/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/gwmgscldykdtqcxhpoyn
Upload files to local dir
>>> init logging <<<
2026-07-13 18:31:18,563 - SL - DEBUG - 5029 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
MAX_ACTIVATION_TRIES = 3
2026-07-13 18:31:19,893 - SL - INFO - 5029 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty

=== SCENARIO A: repeated WRONG code -> tries progression + lockout ===
2026-07-13 18:31:19,938 - SL - INFO - 5029 - "/app/app/mailbox_utils.py:88" - create_mailbox() -  - User <User 485 Test User user_6n37054mok@mailbox.test> has created mailbox with q1_scenarioA@example.com
2026-07-13 18:31:19,942 - SL - INFO - 5029 - "/app/app/mailbox_utils.py:93" - create_mailbox() -  - Skipping sending validation email for mailbox <Mailbox 639 q1_scenarioa@example.com>
mailbox_id = 639
correct code = 'N9puqbrKafo7GWGarbyM7A'
BEFORE any submission: tries = 0
2026-07-13 18:31:19,947 - SL - INFO - 5029 - "/app/app/mailbox_utils.py:206" - verify_mailbox_code() -  - User <User 485 Test User user_6n37054mok@mailbox.test> failed to verify mailbox 639 because code does not match
submission 1: CannotVerifyError(msg='Invalid activation code') -> tries now = 1
2026-07-13 18:31:19,952 - SL - INFO - 5029 - "/app/app/mailbox_utils.py:206" - verify_mailbox_code() -  - User <User 485 Test User user_6n37054mok@mailbox.test> failed to verify mailbox 639 because code does not match
submission 2: CannotVerifyError(msg='Invalid activation code') -> tries now = 2
2026-07-13 18:31:19,956 - SL - INFO - 5029 - "/app/app/mailbox_utils.py:206" - verify_mailbox_code() -  - User <User 485 Test User user_6n37054mok@mailbox.test> failed to verify mailbox 639 because code does not match
submission 3: CannotVerifyError(msg='Invalid activation code') -> tries now = 3
--- 4th submission (tries>=3 triggers lockout) ---
2026-07-13 18:31:19,961 - SL - INFO - 5029 - "/app/app/mailbox_utils.py:196" - verify_mailbox_code() -  - User <User 485 Test User user_6n37054mok@mailbox.test> failed to verify mailbox 639 more than 3 times
submission 4: CannotVerifyError(msg='Invalid activation code. Please request another code.')
activation row after lockout: None
activation row count after lockout: 0
--- 5th submission (no activation -> MailboxError) ---
2026-07-13 18:31:19,966 - SL - INFO - 5029 - "/app/app/mailbox_utils.py:191" - verify_mailbox_code() -  - User <User 485 Test User user_6n37054mok@mailbox.test> failed to verify mailbox 639 because there is no activation
submission 5: MailboxError(msg='Invalid code')

=== SCENARIO B: 15-minute expiry edge branch ===
2026-07-13 18:31:19,996 - SL - INFO - 5029 - "/app/app/mailbox_utils.py:88" - create_mailbox() -  - User <User 485 Test User user_6n37054mok@mailbox.test> has created mailbox with q1_scenarioB@example.com
2026-07-13 18:31:20,000 - SL - INFO - 5029 - "/app/app/mailbox_utils.py:93" - create_mailbox() -  - Skipping sending validation email for mailbox <Mailbox 640 q1_scenariob@example.com>
mailbox_id = 640 | tries = 0 | created_at backdated to 2026-07-13T18:15:20.001163+00:00
2026-07-13 18:31:20,005 - SL - INFO - 5029 - "/app/app/mailbox_utils.py:200" - verify_mailbox_code() -  - User <User 485 Test User user_6n37054mok@mailbox.test> failed to verify mailbox 640 because code is too old
expiry: CannotVerifyError(msg='Invalid activation code. Please request another code.')
activation row count after expiry: 0

=== SCENARIO C: success control path (correct code) ===
2026-07-13 18:31:20,038 - SL - INFO - 5029 - "/app/app/mailbox_utils.py:88" - create_mailbox() -  - User <User 485 Test User user_6n37054mok@mailbox.test> has created mailbox with q1_scenarioC@example.com
2026-07-13 18:31:20,042 - SL - INFO - 5029 - "/app/app/mailbox_utils.py:93" - create_mailbox() -  - Skipping sending validation email for mailbox <Mailbox 641 q1_scenarioc@example.com>
mailbox_id = 641 | verified BEFORE = False | activation count BEFORE = 1
2026-07-13 18:31:20,046 - SL - INFO - 5029 - "/app/app/mailbox_utils.py:212" - verify_mailbox_code() -  - User <User 485 Test User user_6n37054mok@mailbox.test> has verified mailbox 641
verify returned Mailbox id = 641
verified AFTER = True | activation count AFTER = 0
2026-07-13 18:31:20,051 - SL - INFO - 5029 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 18:31:20,054 - SL - INFO - 5029 - "/app/app/alias_utils.py:346" - delete_alias() -  - User <User 485 Test User user_6n37054mok@mailbox.test> has deleted alias <Alias 802 simplelogin-newsletter.accrue154@sl.local>
2026-07-13 18:31:20,059 - SL - INFO - 5029 - "/app/app/alias_utils.py:368" - delete_alias() -  - Moving <Alias 802 simplelogin-newsletter.accrue154@sl.local> to global trash <Deleted Alias simplelogin-newsletter.accrue154@sl.local>
2026-07-13 18:31:20,062 - SL - INFO - 5029 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty

=== cleanup: deleted user 485; (User,Mailbox,MailboxActivation) before=(172, 264, 42) after=(172, 264, 42) net-zero=True ===
```

**Complete unedited output — RUN 2** (stability; the same command run a second
time — identical behavior; only the auto-generated user/mailbox/alias ids and
the random code differ):

```
load config file /app/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/ppxizqnxepyhiexdfnmx
Upload files to local dir
>>> init logging <<<
2026-07-13 18:31:28,558 - SL - DEBUG - 5053 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
MAX_ACTIVATION_TRIES = 3
2026-07-13 18:31:29,918 - SL - INFO - 5053 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty

=== SCENARIO A: repeated WRONG code -> tries progression + lockout ===
2026-07-13 18:31:29,985 - SL - INFO - 5053 - "/app/app/mailbox_utils.py:88" - create_mailbox() -  - User <User 486 Test User user_halznwv3qo@mailbox.test> has created mailbox with q1_scenarioA@example.com
2026-07-13 18:31:29,990 - SL - INFO - 5053 - "/app/app/mailbox_utils.py:93" - create_mailbox() -  - Skipping sending validation email for mailbox <Mailbox 643 q1_scenarioa@example.com>
mailbox_id = 643
correct code = 'g9sDQ8CqnecTSAQOqONXkA'
BEFORE any submission: tries = 0
2026-07-13 18:31:29,995 - SL - INFO - 5053 - "/app/app/mailbox_utils.py:206" - verify_mailbox_code() -  - User <User 486 Test User user_halznwv3qo@mailbox.test> failed to verify mailbox 643 because code does not match
submission 1: CannotVerifyError(msg='Invalid activation code') -> tries now = 1
2026-07-13 18:31:29,999 - SL - INFO - 5053 - "/app/app/mailbox_utils.py:206" - verify_mailbox_code() -  - User <User 486 Test User user_halznwv3qo@mailbox.test> failed to verify mailbox 643 because code does not match
submission 2: CannotVerifyError(msg='Invalid activation code') -> tries now = 2
2026-07-13 18:31:30,004 - SL - INFO - 5053 - "/app/app/mailbox_utils.py:206" - verify_mailbox_code() -  - User <User 486 Test User user_halznwv3qo@mailbox.test> failed to verify mailbox 643 because code does not match
submission 3: CannotVerifyError(msg='Invalid activation code') -> tries now = 3
--- 4th submission (tries>=3 triggers lockout) ---
2026-07-13 18:31:30,008 - SL - INFO - 5053 - "/app/app/mailbox_utils.py:196" - verify_mailbox_code() -  - User <User 486 Test User user_halznwv3qo@mailbox.test> failed to verify mailbox 643 more than 3 times
submission 4: CannotVerifyError(msg='Invalid activation code. Please request another code.')
activation row after lockout: None
activation row count after lockout: 0
--- 5th submission (no activation -> MailboxError) ---
2026-07-13 18:31:30,014 - SL - INFO - 5053 - "/app/app/mailbox_utils.py:191" - verify_mailbox_code() -  - User <User 486 Test User user_halznwv3qo@mailbox.test> failed to verify mailbox 643 because there is no activation
submission 5: MailboxError(msg='Invalid code')

=== SCENARIO B: 15-minute expiry edge branch ===
2026-07-13 18:31:30,044 - SL - INFO - 5053 - "/app/app/mailbox_utils.py:88" - create_mailbox() -  - User <User 486 Test User user_halznwv3qo@mailbox.test> has created mailbox with q1_scenarioB@example.com
2026-07-13 18:31:30,048 - SL - INFO - 5053 - "/app/app/mailbox_utils.py:93" - create_mailbox() -  - Skipping sending validation email for mailbox <Mailbox 644 q1_scenariob@example.com>
mailbox_id = 644 | tries = 0 | created_at backdated to 2026-07-13T18:15:30.049091+00:00
2026-07-13 18:31:30,053 - SL - INFO - 5053 - "/app/app/mailbox_utils.py:200" - verify_mailbox_code() -  - User <User 486 Test User user_halznwv3qo@mailbox.test> failed to verify mailbox 644 because code is too old
expiry: CannotVerifyError(msg='Invalid activation code. Please request another code.')
activation row count after expiry: 0

=== SCENARIO C: success control path (correct code) ===
2026-07-13 18:31:30,085 - SL - INFO - 5053 - "/app/app/mailbox_utils.py:88" - create_mailbox() -  - User <User 486 Test User user_halznwv3qo@mailbox.test> has created mailbox with q1_scenarioC@example.com
2026-07-13 18:31:30,089 - SL - INFO - 5053 - "/app/app/mailbox_utils.py:93" - create_mailbox() -  - Skipping sending validation email for mailbox <Mailbox 645 q1_scenarioc@example.com>
mailbox_id = 645 | verified BEFORE = False | activation count BEFORE = 1
2026-07-13 18:31:30,094 - SL - INFO - 5053 - "/app/app/mailbox_utils.py:212" - verify_mailbox_code() -  - User <User 486 Test User user_halznwv3qo@mailbox.test> has verified mailbox 645
verify returned Mailbox id = 645
verified AFTER = True | activation count AFTER = 0
2026-07-13 18:31:30,100 - SL - INFO - 5053 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 18:31:30,103 - SL - INFO - 5053 - "/app/app/alias_utils.py:346" - delete_alias() -  - User <User 486 Test User user_halznwv3qo@mailbox.test> has deleted alias <Alias 803 simplelogin-newsletter.brings553@sl.local>
2026-07-13 18:31:30,107 - SL - INFO - 5053 - "/app/app/alias_utils.py:368" - delete_alias() -  - Moving <Alias 803 simplelogin-newsletter.brings553@sl.local> to global trash <Deleted Alias simplelogin-newsletter.brings553@sl.local>
2026-07-13 18:31:30,110 - SL - INFO - 5053 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty

=== cleanup: deleted user 486; (User,Mailbox,MailboxActivation) before=(172, 264, 42) after=(172, 264, 42) net-zero=True ===
```

**What this proves (before / intermediate / after):**

- **Limit + counter + progression [OBSERVED].** `MAX_ACTIVATION_TRIES = 3`;
  `tries` advances **0 → 1 → 2 → 3** across submissions 1–3, each raising
  `CannotVerifyError(msg='Invalid activation code')`. The logger marker
  `mailbox_utils.py:206` confirms the "code does not match" branch; the
  increment/commit/raise are at `app/mailbox_utils.py:209-211`.
- **Lockout condition [OBSERVED].** Submission 4 (with `tries` already `3`) hits
  the top-of-function guard (`mailbox_utils.py:196`, "more than 3 times") and
  raises `CannotVerifyError('Invalid activation code. Please request another
  code.')`; the activation row is then gone (`None`, count `0`), proving
  `clear_activation_codes_for_mailbox` [`app/mailbox_utils.py:197`] deleted it.
- **Post-lockout secondary path [OBSERVED].** Submission 5 raises
  `MailboxError('Invalid code')` via the no-activation branch
  (`mailbox_utils.py:191`, raise at `:194`).
- **15-minute expiry edge branch [OBSERVED].** With `created_at` back-dated 16
  minutes, even the correct code is rejected (`mailbox_utils.py:200`, "code is
  too old"), codes cleared (count `0`).
- **Success control path [OBSERVED].** A correct code flips `verified`
  `False → True` and clears the activation rows (count `0`) — `mailbox_utils.py:212`.
- **Net-zero cleanup [OBSERVED].** Both runs report
  `before=(172, 264, 42) after=(172, 264, 42) net-zero=True`.


### Evidence B — the canonical HTTP entry point `GET /dashboard/mailbox_verify`

Evidence A drove the enforcement function directly. Evidence B drives the **real
HTTP route** through a **real `/auth/login` session** (no session injection, no
debug shortcut), proving the route returns **HTTP 302** on every wrong
submission — never a 429 — and reporting each attempt's flash message read
directly from the Flask session.

Rate limiting is disabled exactly as the canonical `flask_client` test fixture
does it — `config.DISABLE_RATE_LIMIT = True` — and that override is stated
explicitly here because, as established in the front matter, **`tests/test.env`
does not set `DISABLE_RATE_LIMIT`** (the process value is `None` → config
`False`). This override does not affect the outcome: `/dashboard/mailbox_verify`
has no limiter to disable; it is set only so the surrounding `/auth/login` and
`/dashboard/` requests behave like the canonical fixture.

Script `obs_q1_http.py` (reproduced in full):

```python
"""Q1 - exercise the CANONICAL HTTP entry point GET /dashboard/mailbox_verify.

Authentication uses the REAL /auth/login route (same POST the test-suite login
helper uses): we POST email+password, observe the 302 redirect to /dashboard,
then GET a protected page and confirm it returns 200 containing the
authenticated marker b"/auth/logout" (i.e. the real login session -- not a
session injection or debug shortcut). Rate limiting is disabled the same way the
canonical flask_client fixture does it -- config.DISABLE_RATE_LIMIT = True --
and that override is stated explicitly (it is NOT set by tests/test.env).
The flash message is read directly from the Flask session and CLEARED after each
request, so each attempt's exact flash is reported separately (no HTML/regex
scraping); attempts 4 (lockout) and 5 (no activation) are printed individually."""
from server import create_app
app = create_app()
app.config["TESTING"] = True
app.config["WTF_CSRF_ENABLED"] = False
app.config["SERVER_NAME"] = "sl.test"
from app import config
config.DISABLE_RATE_LIMIT = True   # documented fixture override; test.env does NOT set this
from app.db import Session
from app.models import User, Mailbox, MailboxActivation
from tests.utils import create_new_user
from flask import url_for
import app.mailbox_utils as mu

def counts():
    return (Session.query(User).count(), Session.query(Mailbox).count(),
            Session.query(MailboxActivation).count())

def tries_of(mb_id):
    Session.expire_all()
    a = (MailboxActivation.filter(MailboxActivation.mailbox_id == mb_id)
         .order_by(MailboxActivation.created_at.desc()).first())
    return None if a is None else a.tries

def act_count(mb_id):
    Session.expire_all()
    return MailboxActivation.filter(MailboxActivation.mailbox_id == mb_id).count()

def read_and_clear_flashes(client):
    with client.session_transaction() as sess:
        fl = list(sess.get("_flashes", []))
        sess["_flashes"] = []
    return fl

with app.app_context():
    before = counts()
    user = create_new_user()      # password="password", activated=True
    user.lifetime = True
    Session.commit()
    uid = user.id
    email = user.email
    try:
        mb_id = mu.create_mailbox(user, "q1_http@example.com", send_email=False).mailbox.id
        act = (MailboxActivation.filter(MailboxActivation.mailbox_id == mb_id)
               .order_by(MailboxActivation.created_at.desc()).first())
        print("logged-in user =", email, "| mailbox_id =", mb_id,
              "| correct code =", repr(act.code))

        client = app.test_client()

        with app.test_request_context():
            login_url = url_for("auth.login")
            dash_url = url_for("dashboard.index")
        # real login: POST /auth/login -> 302 to dashboard
        r0 = client.post(login_url, data={"email": email, "password": "password"},
                         follow_redirects=False)
        # confirm authenticated session: protected page returns 200 + logout marker
        r1 = client.get(dash_url, follow_redirects=False)
        print()
        print("=== REAL /auth/login transcript ===")
        print("POST %s (email + password) -> HTTP %d | Location=%s"
              % (login_url, r0.status_code, r0.headers.get("Location")))
        print("GET  %s (protected) -> HTTP %d | b'/auth/logout' present = %s"
              % (dash_url, r1.status_code, (b"/auth/logout" in r1.data)))

        with app.test_request_context():
            verify_path = url_for("dashboard.mailbox_verify")
        print()
        print("=== authenticated GET", verify_path, "with WRONG code x5 (expect 302, never 429) ===")
        print("tries BEFORE =", tries_of(mb_id))
        for i in range(1, 6):
            url = "%s?mailbox_id=%d&code=wrong-http-%d" % (verify_path, mb_id, i)
            resp = client.get(url, follow_redirects=False)
            loc = resp.headers.get("Location")
            flash_i = read_and_clear_flashes(client)   # isolate THIS attempt's flash
            tag = ""
            if i == 4:
                tag = "   <- lockout (tries>=3): clears codes"
            elif i == 5:
                tag = "   <- no activation remains"
            print("attempt %d: GET %s -> HTTP %d | Location=%s | tries=%s%s"
                  % (i, url, resp.status_code, loc, tries_of(mb_id), tag))
            print("           flash = %s" % (flash_i,))
        print("activation count after wrong HTTP submissions:", act_count(mb_id))
    finally:
        User.delete(uid)
        Session.commit()
        after = counts()
        print()
        print("=== cleanup: deleted user %d; (User,Mailbox,MailboxActivation) "
              "before=%s after=%s net-zero=%s ===" % (uid, before, after, before == after))
```

**Command** (the trailing `grep -v` filter is declared as part of the command;
it removes only the app's own boilerplate logger/init lines — `SL - DEBUG/INFO`
markers, config-load, GNUPGHOME warning, upload-dir notice, init-logging banner
— and the output below is the complete output of this exact command):

```bash
docker exec sl_canon bash -lc '
export CONFIG=tests/test.env PYTHONPATH=/app DB_URI="postgresql://test:test@localhost:5432/test"
cd /app && /app/venv/bin/python /tmp/blitzy_obs/obs_q1_http.py 2>&1 | grep -v -E "SL - (DEBUG|INFO) -|load config file|>>> URL|WARNING: Use a temp|Upload files to local|>>> init logging"'
```

**Complete output — RUN 1:**

```
logged-in user = user_pmmjaoy1g6@mailbox.test | mailbox_id = 635 | correct code = 'z1X3Fr6ubQv9Ml2rwkYvKQ'

=== REAL /auth/login transcript ===
POST /auth/login (email + password) -> HTTP 302 | Location=http://sl.test/dashboard/
GET  /dashboard/ (protected) -> HTTP 200 | b'/auth/logout' present = True

=== authenticated GET /dashboard/mailbox_verify with WRONG code x5 (expect 302, never 429) ===
tries BEFORE = 0
attempt 1: GET /dashboard/mailbox_verify?mailbox_id=635&code=wrong-http-1 -> HTTP 302 | Location=http://sl.test/dashboard/mailbox | tries=1
           flash = [('error', 'Cannot verify mailbox: Invalid activation code')]
attempt 2: GET /dashboard/mailbox_verify?mailbox_id=635&code=wrong-http-2 -> HTTP 302 | Location=http://sl.test/dashboard/mailbox | tries=2
           flash = [('error', 'Cannot verify mailbox: Invalid activation code')]
attempt 3: GET /dashboard/mailbox_verify?mailbox_id=635&code=wrong-http-3 -> HTTP 302 | Location=http://sl.test/dashboard/mailbox | tries=3
           flash = [('error', 'Cannot verify mailbox: Invalid activation code')]
attempt 4: GET /dashboard/mailbox_verify?mailbox_id=635&code=wrong-http-4 -> HTTP 302 | Location=http://sl.test/dashboard/mailbox | tries=None   <- lockout (tries>=3): clears codes
           flash = [('error', 'Cannot verify mailbox: Invalid activation code. Please request another code.')]
attempt 5: GET /dashboard/mailbox_verify?mailbox_id=635&code=wrong-http-5 -> HTTP 302 | Location=http://sl.test/dashboard/mailbox | tries=None   <- no activation remains
           flash = [('error', 'Cannot verify mailbox: Invalid code')]
activation count after wrong HTTP submissions: 0

=== cleanup: deleted user 483; (User,Mailbox,MailboxActivation) before=(172, 264, 42) after=(172, 264, 42) net-zero=True ===
```

**Complete output — RUN 2** (stability; identical behavior, ids/code differ):

```
logged-in user = user_6rz7nyv0p1@mailbox.test | mailbox_id = 637 | correct code = 'houuypnH8NQxPVPz1kc58g'

=== REAL /auth/login transcript ===
POST /auth/login (email + password) -> HTTP 302 | Location=http://sl.test/dashboard/
GET  /dashboard/ (protected) -> HTTP 200 | b'/auth/logout' present = True

=== authenticated GET /dashboard/mailbox_verify with WRONG code x5 (expect 302, never 429) ===
tries BEFORE = 0
attempt 1: GET /dashboard/mailbox_verify?mailbox_id=637&code=wrong-http-1 -> HTTP 302 | Location=http://sl.test/dashboard/mailbox | tries=1
           flash = [('error', 'Cannot verify mailbox: Invalid activation code')]
attempt 2: GET /dashboard/mailbox_verify?mailbox_id=637&code=wrong-http-2 -> HTTP 302 | Location=http://sl.test/dashboard/mailbox | tries=2
           flash = [('error', 'Cannot verify mailbox: Invalid activation code')]
attempt 3: GET /dashboard/mailbox_verify?mailbox_id=637&code=wrong-http-3 -> HTTP 302 | Location=http://sl.test/dashboard/mailbox | tries=3
           flash = [('error', 'Cannot verify mailbox: Invalid activation code')]
attempt 4: GET /dashboard/mailbox_verify?mailbox_id=637&code=wrong-http-4 -> HTTP 302 | Location=http://sl.test/dashboard/mailbox | tries=None   <- lockout (tries>=3): clears codes
           flash = [('error', 'Cannot verify mailbox: Invalid activation code. Please request another code.')]
attempt 5: GET /dashboard/mailbox_verify?mailbox_id=637&code=wrong-http-5 -> HTTP 302 | Location=http://sl.test/dashboard/mailbox | tries=None   <- no activation remains
           flash = [('error', 'Cannot verify mailbox: Invalid code')]
activation count after wrong HTTP submissions: 0

=== cleanup: deleted user 484; (User,Mailbox,MailboxActivation) before=(172, 264, 42) after=(172, 264, 42) net-zero=True ===
```

**What this proves [OBSERVED]:**

- **Real authenticated session.** `POST /auth/login` returns `302` to
  `http://sl.test/dashboard/`, and the subsequent `GET /dashboard/` returns
  `200` containing `b'/auth/logout'` — the canonical login route established the
  session, not a debug hook.
- **Never 429 / no lockout by HTTP status.** All five wrong submissions return
  `HTTP 302` to `…/dashboard/mailbox`. Enforcement is entirely the `tries`
  counter and code deletion.
- **`tries` progression over HTTP:** `0 → 1 → 2 → 3`, then `None` (row deleted)
  on attempts 4 and 5 — matching Evidence A through the real route.
- **Cleanly isolated flashes (each read-and-cleared per attempt):** attempts 1–3
  → `Cannot verify mailbox: Invalid activation code`; attempt 4 (lockout) →
  `Cannot verify mailbox: Invalid activation code. Please request another code.`;
  attempt 5 (no activation) → `Cannot verify mailbox: Invalid code`. These are
  the route's flash strings (`app/dashboard/views/mailbox.py:132`), formatted as
  `f"Cannot verify mailbox: {e.msg}"`.

### Verbatim source — the route and its (absent) limiter

```python
# app/dashboard/views/mailbox.py  (L120-135)
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
```

The `/mailbox_verify` rule carries exactly two decorators — `@dashboard_bp.route`
and `@login_required` — and **no** limiter. By contrast the `/mailbox` route adds
a third decorator:

```python
# app/dashboard/views/mailbox.py  (L36-38)
@dashboard_bp.route("/mailbox", methods=["GET", "POST"])
@login_required
@parallel_limiter.lock(only_when=lambda: request.method == "POST")
```

Grep confirming the decorator inventory (the only limiter decorator in the file
sits on `/mailbox`, line 38; none appears between the `/mailbox_verify` route on
line 120 and its handler on line 122):

```
$ grep -n -E "route\(|login_required|limiter|parallel_limiter" app/dashboard/views/mailbox.py
7:from flask_login import login_required, current_user
13:from app import parallel_limiter, mailbox_utils, user_settings
36:@dashboard_bp.route("/mailbox", methods=["GET", "POST"])
37:@login_required
38:@parallel_limiter.lock(only_when=lambda: request.method == "POST")
39:def mailbox_route():
120:@dashboard_bp.route("/mailbox_verify")
121:@login_required
```

**[SOURCE-VERIFIED]** (decorator inventory by reading + grep — not runtime).
The blueprint's `url_prefix="/dashboard"` [`app/dashboard/base.py:6`] is what
makes the external path `/dashboard/mailbox_verify`.

### Verbatim source — the enforcement function

`verify_mailbox_code` — the full enforcement logic (`app/mailbox_utils.py`,
L166-220; `MAX_ACTIVATION_TRIES = 3` is defined at L43):

```python
def verify_mailbox_code(user: User, mailbox_id: int, code: str) -> Mailbox:
    mailbox = Mailbox.get(mailbox_id)
    if not mailbox:
        LOG.i(
            f"User {user} failed to verify mailbox {mailbox_id} because it does not exist"
        )
        raise MailboxError("Invalid mailbox")
    if mailbox.verified:
        LOG.i(
            f"User {user} failed to verify mailbox {mailbox_id} because it's already verified"
        )
        clear_activation_codes_for_mailbox(mailbox)
        return mailbox
    if mailbox.user_id != user.id:
        LOG.i(
            f"User {user} failed to verify mailbox {mailbox_id} because it's owned by another user"
        )
        raise MailboxError("Invalid mailbox")

    activation = (
        MailboxActivation.filter(MailboxActivation.mailbox_id == mailbox_id)
        .order_by(MailboxActivation.created_at.desc())
        .first()
    )
    if not activation:
        LOG.i(
            f"User {user} failed to verify mailbox {mailbox_id} because there is no activation"
        )
        raise MailboxError("Invalid code")
    if activation.tries >= MAX_ACTIVATION_TRIES:
        LOG.i(f"User {user} failed to verify mailbox {mailbox_id} more than 3 times")
        clear_activation_codes_for_mailbox(mailbox)
        raise CannotVerifyError("Invalid activation code. Please request another code.")
    if activation.created_at < arrow.now().shift(minutes=-15):
        LOG.i(
            f"User {user} failed to verify mailbox {mailbox_id} because code is too old"
        )
        clear_activation_codes_for_mailbox(mailbox)
        raise CannotVerifyError("Invalid activation code. Please request another code.")
    if code != activation.code:
        LOG.i(
            f"User {user} failed to verify mailbox {mailbox_id} because code does not match"
        )
        activation.tries = activation.tries + 1
        Session.commit()
        raise CannotVerifyError("Invalid activation code")
    LOG.i(f"User {user} has verified mailbox {mailbox_id}")
    mailbox.verified = True
    emit_user_audit_log(
        user=user,
        action=UserAuditLogAction.VerifyMailbox,
        message=f"Verify mailbox {mailbox_id} ({mailbox.email})",
    )
    clear_activation_codes_for_mailbox(mailbox)
    return mailbox
```

**[SOURCE-VERIFIED]** (exact lines shown by `sed`); the branch each observed
run took is additionally **[OBSERVED]** via the `mailbox_utils.py:<line>` logger
markers in Evidence A's output (`:206` code-mismatch, `:196` lockout, `:200`
expiry, `:191` no-activation, `:212` success).

### Q1 file:line reference table

| Fact | Value | Location | Label |
|------|-------|----------|-------|
| Max attempts | `MAX_ACTIVATION_TRIES = 3` | `app/mailbox_utils.py:43` | [OBSERVED] + [SOURCE-VERIFIED] |
| Attempt counter | `MailboxActivation.tries` (Integer, default 0) | `app/models.py:2835` | [OBSERVED] (progression) |
| Activation code column | `code = sa.Column(sa.String(32) …)` | `app/models.py:2834` | [SOURCE-VERIFIED] |
| Enforcement function | `verify_mailbox_code(user, mailbox_id, code)` | `app/mailbox_utils.py:166` | [OBSERVED] |
| Canonical HTTP route | `GET /dashboard/mailbox_verify` | `app/dashboard/views/mailbox.py:120-122` | [OBSERVED] |
| Blueprint prefix | `url_prefix="/dashboard"` | `app/dashboard/base.py:6` | [SOURCE-VERIFIED] |
| Route → function call | `verify_mailbox_code(current_user, …)` | `app/dashboard/views/mailbox.py:129` | [OBSERVED] |
| No-activation branch | `raise MailboxError("Invalid code")` | `app/mailbox_utils.py:194` | [OBSERVED] |
| Lockout guard | `if activation.tries >= MAX_ACTIVATION_TRIES` | `app/mailbox_utils.py:195` | [OBSERVED] |
| Lockout clears codes | `clear_activation_codes_for_mailbox(mailbox)` | `app/mailbox_utils.py:197` | [OBSERVED] |
| Lockout exception | `CannotVerifyError("… Please request another code.")` | `app/mailbox_utils.py:198` | [OBSERVED] |
| 15-min expiry guard | `activation.created_at < arrow.now().shift(minutes=-15)` | `app/mailbox_utils.py:199` | [OBSERVED] |
| Wrong-code increment | `activation.tries = activation.tries + 1; Session.commit()` | `app/mailbox_utils.py:209-210` | [OBSERVED] |
| Wrong-code exception | `CannotVerifyError("Invalid activation code")` | `app/mailbox_utils.py:211` | [OBSERVED] |
| Success | `mailbox.verified = True` | `app/mailbox_utils.py:213` | [OBSERVED] |
| Limiter on `/mailbox` POST only | `@parallel_limiter.lock(...)` | `app/dashboard/views/mailbox.py:38` | [SOURCE-VERIFIED] |
| `/mailbox_verify` has no limiter | (only `route` + `login_required`) | `app/dashboard/views/mailbox.py:120-121` | [SOURCE-VERIFIED] |
| Generic bucket limiter (not applied here) | `check_bucket_limit` raises `TooManyRequests` at `max_hits` | `app/rate_limiter.py:19-42` | [SOURCE-VERIFIED] |

### Q1 coverage confirmation

- **(a) what limit(s) exist** → `MAX_ACTIVATION_TRIES = 3` **[OBSERVED]**.
- **(b) how failed attempts are tracked** → `MailboxActivation.tries`, observed
  advancing `0 → 1 → 2 → 3` **[OBSERVED]**.
- **(c) the condition that ultimately prevents further submissions** → the
  `tries >= 3` guard deletes all activation codes and raises "Please request
  another code."; the next submission finds no activation and raises "Invalid
  code" **[OBSERVED]**. Not an HTTP 429 — the route has no limiter; every wrong
  submission returns HTTP 302 **[OBSERVED]**.


---

## Q2 — Background-task (Job) lifecycle and error behavior

### Direct answer

**The mechanism.** SimpleLogin has no Celery/RQ/Arq. Background tasks are rows in
the `job` table (`Job` model [`app/models.py:2683`]) drained by a standalone
daemon `job_runner.py`. A job's `state` is a `JobState` enum
[`app/models.py:253-257`]: **`ready=0`, `taken=1`, `done=2`, `error=3`**.
**[OBSERVED]** (enum values printed at runtime).

**Normal lifecycle (creation → completion).** A job is created in `state=ready(0)`
(e.g. `Job.create(name=…, payload=…)`). The daemon's main loop
[`job_runner.py:329-347`] selects eligible jobs via `get_jobs_to_run()`
[`job_runner.py:307`], then for each job: sets `taken=True`, `taken_at=now`,
`state=taken(1)`, **increments `attempts`**, commits, calls `process_job(job)`
[`job_runner.py:188`], and **only after `process_job` returns** sets
`state=done(2)` and commits. Observed transition for a valid `onboarding-1` job:
`state 0 → 2`, `attempts 0 → 1`. **[OBSERVED]**

**Error behavior (the key finding).** There is **no `try/except` anywhere in
`job_runner.py`**. When `process_job` raises, the exception propagates out of the
`for` loop and the `while True` loop, and **the runner process exits** — the
`state = done` assignment [`job_runner.py:344`] is never reached. The job is left
at **`state=taken(1)` with `attempts` incremented — it is NOT set to
`error(3)`.** In fact `JobState.error` is **never assigned by `job_runner.py`**;
it is written only in `cleanup_old_jobs` (as a delete filter) and in tests.
Observed on a failing `batch-import` job: process exits with a Python traceback
and **exit code 1** (not the timeout code 124), and the persisted job is
`state=1, attempts=1, done=False, error=False`. **[OBSERVED]**

**Recovery / retry.** There is no in-process retry. Recovery happens only when the
daemon is **restarted**: `get_jobs_to_run()` re-selects a `taken` job once its
`taken_at` is older than `JOB_TAKEN_RETRY_WAIT_MINS = 30` minutes
[`app/config.py:565`] **and** `attempts < JOB_MAX_ATTEMPTS = 5`
[`app/config.py:564`]. So a failing job is retried on subsequent daemon starts,
at most 5 attempts, each 30-minutes-stale apart. **[OBSERVED]** (eligibility
matrix below).

**End-of-life.** `cleanup_old_jobs(oldest_allowed)` [`tasks/cleanup_old_jobs.py:10`]
deletes jobs whose `updated_at < oldest_allowed` **and** whose state is `done(2)`
or `error(3)` **or** (`taken(1)` with `attempts >= JOB_MAX_ATTEMPTS`). It is
invoked by `cron.py delete_old_data()` [`cron.py:1245-1248`], scheduled in
`crontab.yml` as "SimpleLogin Delete Old data" at `30 5 * * *`
[`crontab.yml:40-44`]. Note `job_runner.py` itself is **not** in `crontab.yml`
(it is a long-running daemon, not a cron entry). **[OBSERVED]** + **[SOURCE-VERIFIED]**.

### Configuration constants (printed at runtime)

| Constant | Value | Location | Label |
|----------|-------|----------|-------|
| `JobState.ready` | `0` | `app/models.py:253-257` | [OBSERVED] |
| `JobState.taken` | `1` | `app/models.py:253-257` | [OBSERVED] |
| `JobState.done` | `2` | `app/models.py:253-257` | [OBSERVED] |
| `JobState.error` | `3` | `app/models.py:253-257` | [OBSERVED] |
| `JOB_MAX_ATTEMPTS` | `5` | `app/config.py:564` | [OBSERVED] |
| `JOB_TAKEN_RETRY_WAIT_MINS` | `30` | `app/config.py:565` | [OBSERVED] |

### Verbatim source — the daemon main loop (no try/except)

```python
# job_runner.py  (L329-347)
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

The `state = JobState.done.value` at L344 executes **only if `process_job(job)`
at L342 returns normally**. There is no exception handling; a raised exception
skips L344-345 and terminates the process. **[SOURCE-VERIFIED]**

### Verbatim source — the eligibility selector

```python
# job_runner.py  (L307-326)
def get_jobs_to_run() -> List[Job]:
    # Get jobs that match all conditions:
    #  - Job.state == ready OR (Job.state == taken AND Job.taken_at < now - 30 mins AND Job.attempts < 5)
    #  - Job.run_at is Null OR Job.run_at < now + 10 mins
    taken_at_earliest = arrow.now().shift(minutes=-config.JOB_TAKEN_RETRY_WAIT_MINS)
    run_at_earliest = arrow.now().shift(minutes=+10)
    query = Job.filter(
        and_(
            or_(
                Job.state == JobState.ready.value,
                and_(
                    Job.state == JobState.taken.value,
                    Job.taken_at < taken_at_earliest,
                    Job.attempts < config.JOB_MAX_ATTEMPTS,
                ),
            ),
            or_(Job.run_at.is_(None), and_(Job.run_at <= run_at_earliest)),
        )
    )
    return query.all()
```

**[SOURCE-VERIFIED]**

### Verbatim source — `process_job` dispatch (valid names vs unknown)

Head of `process_job` (`job_runner.py`, L188-190) — verbatim:

```python
def process_job(job: Job):
    if job.name == config.JOB_ONBOARDING_1:
        user_id = job.payload.get("user_id")
```

Lines 191-302 are one dispatch branch per recognized job name (`JOB_ONBOARDING_1`,
`JOB_BATCH_IMPORT`, and the other `JOB_*` constants in `app/config.py`), each
calling its handler directly. The final `else` branch (`job_runner.py`, L303-304)
— verbatim — is what an **unrecognized** name hits: it is only logged, never
executed.

```python
    else:
        LOG.e("Unknown job name %s", job.name)
```

`config.JOB_ONBOARDING_1 == "onboarding-1"` and `config.JOB_BATCH_IMPORT ==
"batch-import"` are **recognized** names. A name matching none of the branches
falls through to the `else` and is only logged (`"Unknown job name …"`,
[`job_runner.py:304`]) — it performs no work. `handle_batch_import(None)` raises
`AttributeError` at `app/import_utils.py:23` (`user = batch_import.user`).
**[SOURCE-VERIFIED]** + **[OBSERVED]** (both exercised below).

### Verbatim source — end-of-life cleanup and its schedule

```python
# tasks/cleanup_old_jobs.py  (L10-24)
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
```

```python
# cron.py  (import L69; delete_old_data L1245-1248)
from tasks.cleanup_old_jobs import cleanup_old_jobs
def delete_old_data():
    oldest_valid = arrow.now().shift(days=-config.KEEP_OLD_DATA_DAYS)
    cleanup_old_imports(oldest_valid)
    cleanup_old_jobs(oldest_valid)
```

```yaml
# crontab.yml  (L40-44)
  - name: SimpleLogin Delete Old data
    command: python /code/cron.py -j delete_old_data
    shell: /bin/bash
    schedule: "30 5 * * *"
    captureStderr: true
```

`cleanup_old_jobs` returns `None`; the deleted count is emitted only in its log
line `"Deleted N jobs"` [`tasks/cleanup_old_jobs.py:24`]. **[SOURCE-VERIFIED]**

### Grep evidence — no try/except; `JobState.error` never written by the runner

```
$ grep -n "try:\|except" job_runner.py
(no output)

$ grep -rn "JobState.error" --include=*.py job_runner.py tasks/ app/ tests/
tasks/cleanup_old_jobs.py:15:            Job.state == JobState.error.value,
tests/tasks/test_cleanup_old_jobs.py:21:            state=JobState.error.value,
tests/tasks/test_cleanup_old_jobs.py:46:            state=JobState.error.value,

$ grep -n "job_runner" crontab.yml
(no output)
```

`JobState.error` appears only as a **delete filter** in `cleanup_old_jobs` and as
**test seed data** — never as an assignment in `job_runner.py`. **[SOURCE-VERIFIED]**


### Evidence A — normal lifecycle: a VALID job driven to `done(2)` by the real daemon

This drives the **fully canonical entry point**: the real `job_runner.py`
`__main__` daemon. A single eligible `onboarding-1` job is seeded, the real
daemon is run (it drains the job, then idles in its `time.sleep(10)` loop until
the 8-second `timeout` sends SIGTERM), and the job's state is read back. The
outgoing onboarding email is suppressed by `NOT_SEND_EMAIL` (a bounded
substitute — see the front matter; only enqueue/dispatch is exercised, not real
SMTP). `create_new_user()` also enqueues an `onboarding-1/2/4` triad as a side
effect, so cleanup deletes every job tied to the test user for a net-zero run.

Seeder `obs_q2_seed_valid.py`:

```python
"""Q2 FLOW A (seed) - seed exactly ONE eligible, VALID recognized job.

JOB_ONBOARDING_1 ("onboarding-1") is a real job name recognised by
process_job [job_runner.py:189]. We attach it to a fresh user whose
notification & activated flags are True so the handler's guarded body runs
(the outgoing email itself is suppressed by NOT_SEND_EMAIL - a bounded
substitute, disclosed in the front matter). run_at is left NULL so the job is
eligible immediately. The job id is written to a file so the post-run reader
(a separate process) can look it up after the real daemon has run."""
from server import create_app
app = create_app()
from app.db import Session
from app.models import User, Job, JobState
from app import config
from tests.utils import create_new_user

with app.app_context():
    jobs_before = Session.query(Job).count()
    users_before = Session.query(User).count()
    user = create_new_user()          # activated=True by default
    user.notification = True
    Session.commit()
    job = Job.create(
        name=config.JOB_ONBOARDING_1,
        payload={"user_id": user.id},
        run_at=None,                  # eligible immediately
        commit=True,
    )
    with open("/tmp/blitzy_obs/q2_valid_ids.txt", "w") as fh:
        fh.write("%d %d\n" % (job.id, user.id))
    print("JOB_MAX_ATTEMPTS =", config.JOB_MAX_ATTEMPTS,
          "| JOB_TAKEN_RETRY_WAIT_MINS =", config.JOB_TAKEN_RETRY_WAIT_MINS)
    print("JobState enum: ready=%d taken=%d done=%d error=%d"
          % (JobState.ready.value, JobState.taken.value,
             JobState.done.value, JobState.error.value))
    print("seeded VALID job id =", job.id, "| name =", repr(job.name),
          "| user_id =", user.id)
    print("BEFORE daemon: state =", job.state, "(ready=0) | attempts =",
          job.attempts, "| taken =", job.taken, "| taken_at =", job.taken_at)
    print("jobs_before =", jobs_before, "| users_before =", users_before)
```

Reader/cleanup `obs_q2_read_valid.py`:

```python
"""Q2 FLOW A (read + cleanup) - read the seeded VALID job's state AFTER the real
job_runner.py daemon drained it, then delete every job tied to the test user
(the seeded job PLUS the onboarding triad that create_new_user enqueues) and the
user itself, so the run is net-zero."""
from server import create_app
app = create_app()
from app.db import Session
from app.models import User, Job, JobState

with open("/tmp/blitzy_obs/q2_valid_ids.txt") as fh:
    job_id, user_id = [int(x) for x in fh.read().split()]

with app.app_context():
    job = Job.get(job_id)
    print("AFTER daemon: job id =", job_id, "| state =", job.state,
          "(done=2) | attempts =", job.attempts, "| taken =", job.taken)
    print("state==JobState.done ? ->", job.state == JobState.done.value)
    # cleanup: delete ALL jobs whose payload user_id is this test user
    # (seeded job + the onboarding-1/2/4 triad create_new_user enqueues), then user
    victims = [j.id for j in Session.query(Job).all()
               if isinstance(j.payload, dict) and str(j.payload.get("user_id")) == str(user_id)]
    for jid in victims:
        Job.delete(jid)
    User.delete(user_id)
    Session.commit()
    jobs_after = Session.query(Job).count()
    users_after = Session.query(User).count()
    print("cleanup: deleted jobs", sorted(victims), "+ user", user_id)
    print("jobs_after =", jobs_after, "| users_after =", users_after)
```

**Command** (the daemon step's `grep -E` filter is declared as part of the
command; it keeps the loop/dispatch/traceback lines and drops the per-loop app
re-init banner):

```bash
docker exec sl_canon bash -lc '
export CONFIG=tests/test.env PYTHONPATH=/app DB_URI="postgresql://test:test@localhost:5432/test"
cd /app
echo "########## STEP 1: seed VALID job ##########"
/app/venv/bin/python /tmp/blitzy_obs/obs_q2_seed_valid.py 2>&1 | grep -v -E "SL - (DEBUG|INFO) -|load config|>>> URL|WARNING: Use|Upload files|>>> init"
echo
echo "########## STEP 2: run REAL job_runner.py daemon (timeout 8s) ##########"
timeout 8 /app/venv/bin/python job_runner.py 2>&1 | grep -E "job_runner.py:33[0-9]|process_job|onboarding|Traceback|Error"
echo "daemon exit code = ${PIPESTATUS[0]}  (124 = SIGTERM at timeout while sleeping = drained OK then idle)"
echo
echo "########## STEP 3: read AFTER-state + net-zero cleanup ##########"
/app/venv/bin/python /tmp/blitzy_obs/obs_q2_read_valid.py 2>&1 | grep -v -E "SL - (DEBUG|INFO) -|load config|>>> URL|WARNING: Use|Upload files|>>> init"'
```

**Complete output — RUN 1:**

```
########## STEP 1: seed VALID job ##########
JOB_MAX_ATTEMPTS = 5 | JOB_TAKEN_RETRY_WAIT_MINS = 30
JobState enum: ready=0 taken=1 done=2 error=3
seeded VALID job id = 1549 | name = 'onboarding-1' | user_id = 488
BEFORE daemon: state = 0 (ready=0) | attempts = 0 | taken = False | taken_at = None
jobs_before = 55 | users_before = 172

########## STEP 2: run REAL job_runner.py daemon (timeout 8s) ##########
2026-07-13 18:40:05,711 - SL - DEBUG - 5317 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 1549 onboarding-1 {'user_id': 488}>
2026-07-13 18:40:05,719 - SL - DEBUG - 5317 - "/app/job_runner.py:196" - process_job() -  - send onboarding send-from-alias email to user <User 488 Test User user_1wv509t9mj@mailbox.test>
daemon exit code = 124  (124 = SIGTERM at timeout while sleeping = drained OK then idle)

########## STEP 3: read AFTER-state + net-zero cleanup ##########
AFTER daemon: job id = 1549 | state = 2 (done=2) | attempts = 1 | taken = True
state==JobState.done ? -> True
cleanup: deleted jobs [1546, 1547, 1548, 1549] + user 488
jobs_after = 55 | users_after = 172
```

**Complete output — RUN 2** (stability; ids differ):

```
########## STEP 1: seed VALID job ##########
JOB_MAX_ATTEMPTS = 5 | JOB_TAKEN_RETRY_WAIT_MINS = 30
JobState enum: ready=0 taken=1 done=2 error=3
seeded VALID job id = 1553 | name = 'onboarding-1' | user_id = 489
BEFORE daemon: state = 0 (ready=0) | attempts = 0 | taken = False | taken_at = None
jobs_before = 55 | users_before = 172

########## STEP 2: run REAL job_runner.py daemon (timeout 8s) ##########
2026-07-13 18:40:26,358 - SL - DEBUG - 5369 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 1553 onboarding-1 {'user_id': 489}>
2026-07-13 18:40:26,366 - SL - DEBUG - 5369 - "/app/job_runner.py:196" - process_job() -  - send onboarding send-from-alias email to user <User 489 Test User user_mooizz7h7w@mailbox.test>
daemon exit code = 124  (124 = SIGTERM at timeout while sleeping = drained OK then idle)

########## STEP 3: read AFTER-state + net-zero cleanup ##########
AFTER daemon: job id = 1553 | state = 2 (done=2) | attempts = 1 | taken = True
state==JobState.done ? -> True
cleanup: deleted jobs [1550, 1551, 1552, 1553] + user 489
jobs_after = 55 | users_after = 172
```

**What this proves [OBSERVED]:** the real daemon takes the job
(`job_runner.py:334`), dispatches it through `process_job` to the `onboarding-1`
branch (`job_runner.py:196`), and the job ends at **`state=2 (done)`,
`attempts=1`**. The daemon then idles (exit 124 = SIGTERM at the 8-second
timeout, i.e. it had drained the queue and was sleeping). Both runs identical;
net-zero (`jobs 55→55`, `users 172→172`).


### Evidence B — error path (KEY FINDING): a failing job stays `taken(1)` and the runner process exits

This is the fully-canonical demonstration of the error behavior. A single
eligible `batch-import` job with a non-existent `batch_import_id` is seeded, the
**real `job_runner.py` daemon** is run, and its exit code, traceback, and the
job's persisted state are captured.

Seeder `obs_q2_seed_fail.py`:

```python
"""Q2 FLOW B (seed) - seed exactly ONE eligible job that will FAIL during
process_job. JOB_BATCH_IMPORT dispatches to handle_batch_import(batch_import)
[job_runner.py:222-225]; with a non-existent batch_import_id, BatchImport.get
returns None and handle_batch_import(None) raises AttributeError on
`batch_import.user` [app/import_utils.py:24]. The main loop has NO try/except, so
the exception propagates and the runner PROCESS exits; state=done is never set."""
from server import create_app
app = create_app()
from app.db import Session
from app.models import Job, JobState
from app import config

with app.app_context():
    jobs_before = Session.query(Job).count()
    job = Job.create(
        name=config.JOB_BATCH_IMPORT,
        payload={"batch_import_id": 999999999},   # non-existent -> None -> AttributeError
        run_at=None,
        commit=True,
    )
    with open("/tmp/blitzy_obs/q2_fail_id.txt", "w") as fh:
        fh.write("%d\n" % job.id)
    print("seeded FAILING job id =", job.id, "| name =", repr(job.name),
          "| payload =", job.payload)
    print("BEFORE daemon: state =", job.state, "(ready=0) | attempts =",
          job.attempts, "| taken =", job.taken, "| taken_at =", job.taken_at)
    print("jobs_before =", jobs_before)
```

Reader/cleanup `obs_q2_read_fail.py`:

```python
"""Q2 FLOW B (read + cleanup) - read the failing job's persisted state AFTER the
runner crashed, prove it is stuck at taken(1) with attempts incremented (NOT
done, NOT error), then delete it (net-zero)."""
from server import create_app
app = create_app()
from app.db import Session
from app.models import Job, JobState

with open("/tmp/blitzy_obs/q2_fail_id.txt") as fh:
    job_id = int(fh.read().strip())

with app.app_context():
    job = Job.get(job_id)
    print("AFTER crash: job id =", job_id, "| state =", job.state,
          "| attempts =", job.attempts, "| taken =", job.taken,
          "| taken_at =", job.taken_at)
    print("state==taken(1)? ->", job.state == JobState.taken.value,
          "| state==done(2)? ->", job.state == JobState.done.value,
          "| state==error(3)? ->", job.state == JobState.error.value)
    Job.delete(job_id)
    Session.commit()
    print("cleanup: deleted job", job_id, "| jobs_after =", Session.query(Job).count())
```

**Command** (the daemon's full stdout+stderr is captured to a log file, then the
segment from the "Take job" line to end-of-file is shown — this is the complete,
unedited traceback):

```bash
docker exec sl_canon bash -lc '
export CONFIG=tests/test.env PYTHONPATH=/app DB_URI="postgresql://test:test@localhost:5432/test"
cd /app
echo "########## STEP 1: seed FAILING job ##########"
/app/venv/bin/python /tmp/blitzy_obs/obs_q2_seed_fail.py 2>&1 | grep -v -E "SL - (DEBUG|INFO) -|load config|>>> URL|WARNING: Use|Upload files|>>> init"
echo
echo "########## STEP 2: run REAL job_runner.py daemon (timeout 20s; expect it to CRASH and exit on its own) ##########"
timeout 20 /app/venv/bin/python job_runner.py > /tmp/blitzy_obs/q2_fail_daemon.log 2>&1
echo "daemon exit code = $?  (NOT 124 => process exited ON ITS OWN due to the exception, before the timeout)"
echo "--- daemon output from the Take-job line through end of traceback (complete) ---"
sed -n "/Take job/,\$p" /tmp/blitzy_obs/q2_fail_daemon.log
echo
echo "########## STEP 3: read AFTER-state + cleanup ##########"
/app/venv/bin/python /tmp/blitzy_obs/obs_q2_read_fail.py 2>&1 | grep -v -E "SL - (DEBUG|INFO) -|load config|>>> URL|WARNING: Use|Upload files|>>> init"'
```

**Complete output — RUN 1:**

```
########## STEP 1: seed FAILING job ##########
seeded FAILING job id = 1554 | name = 'batch-import' | payload = {'batch_import_id': 999999999}
BEFORE daemon: state = 0 (ready=0) | attempts = 0 | taken = False | taken_at = None
jobs_before = 55

########## STEP 2: run REAL job_runner.py daemon (timeout 20s; expect it to CRASH and exit on its own) ##########
daemon exit code = 1  (NOT 124 => process exited ON ITS OWN due to the exception, before the timeout)
--- daemon output from the Take-job line through end of traceback (complete) ---
2026-07-13 18:41:10,588 - SL - DEBUG - 5422 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 1554 batch-import {'batch_import_id': 999999999}>
Traceback (most recent call last):
  File "/app/job_runner.py", line 342, in <module>
    process_job(job)
  File "/app/job_runner.py", line 225, in process_job
    handle_batch_import(batch_import)
  File "/app/app/import_utils.py", line 23, in handle_batch_import
    user = batch_import.user
AttributeError: 'NoneType' object has no attribute 'user'

########## STEP 3: read AFTER-state + cleanup ##########
AFTER crash: job id = 1554 | state = 1 | attempts = 1 | taken = True | taken_at = 2026-07-13T18:41:10.588574+00:00
state==taken(1)? -> True | state==done(2)? -> False | state==error(3)? -> False
cleanup: deleted job 1554 | jobs_after = 55
```

**Complete output — RUN 2** (stability; ids differ):

```
########## STEP 1: seed FAILING job ##########
seeded FAILING job id = 1555 | name = 'batch-import' | payload = {'batch_import_id': 999999999}
BEFORE daemon: state = 0 (ready=0) | attempts = 0 | taken = False | taken_at = None
jobs_before = 55

########## STEP 2: run REAL job_runner.py daemon (timeout 20s; expect CRASH + exit on its own) ##########
daemon exit code = 1  (NOT 124 => exited on its own due to the exception)
--- daemon output from the Take-job line through end of traceback (complete) ---
2026-07-13 18:41:32,999 - SL - DEBUG - 5474 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 1555 batch-import {'batch_import_id': 999999999}>
Traceback (most recent call last):
  File "/app/job_runner.py", line 342, in <module>
    process_job(job)
  File "/app/job_runner.py", line 225, in process_job
    handle_batch_import(batch_import)
  File "/app/app/import_utils.py", line 23, in handle_batch_import
    user = batch_import.user
AttributeError: 'NoneType' object has no attribute 'user'

########## STEP 3: read AFTER-state + cleanup ##########
AFTER crash: job id = 1555 | state = 1 | attempts = 1 | taken = True | taken_at = 2026-07-13T18:41:32.999867+00:00
state==taken(1)? -> True | state==done(2)? -> False | state==error(3)? -> False
cleanup: deleted job 1555 | jobs_after = 55
```

**What this proves [OBSERVED]:**

- The real daemon takes the job (`job_runner.py:334`), then **exits with code 1**
  (not the timeout code 124) — i.e. the **process terminated on its own** because
  of the unhandled exception.
- The traceback is exactly `job_runner.py:342` (`process_job(job)`) →
  `job_runner.py:225` (`handle_batch_import`) → `app/import_utils.py:23`
  (`user = batch_import.user`) → `AttributeError: 'NoneType' object has no
  attribute 'user'`.
- The persisted job is **`state=1 (taken)`, `attempts=1`**, with `state==done`
  **False** and `state==error` **False** — the failing job is left `taken` with
  its attempt counter incremented; it is **never** transitioned to `error(3)`,
  confirming the finding that `job_runner.py` has no error-state handling and the
  `state=done` line (L344) is never reached. Both runs identical; net-zero
  (`jobs 55→55`).

> **Finding (documented, not fixed — per AAP §0.3.2).** Because there is no
> `try/except` around `process_job`, a single failing job crashes the entire
> `job_runner.py` process; the job is not marked `error`, and no further jobs are
> processed until the daemon is restarted. This is reported as an observed
> finding; the AAP explicitly places fixing this behavior out of scope.


### Evidence C — retry eligibility and end-of-life cleanup (incl. the `error(3)` row)

This drives the real `get_jobs_to_run()` selector and the real
`cleanup_old_jobs()` deleter in-process, printing exact before/after rows. All
seeded rows are tracked and deleted at the end (net-zero); `updated_at`/`taken_at`
are back-dated with raw SQL so the ORM `onupdate` hook does not reset them.

Script `obs_q2_eligibility_cleanup.py`:

```python
"""Q2 FLOW C - two read-only demonstrations against the REAL functions:

Part 1: retry eligibility via the real get_jobs_to_run() [job_runner.py:307].
        Seed jobs covering each branch and print which the selector returns.
Part 2: end-of-life deletion via the real cleanup_old_jobs() [tasks/cleanup_old_jobs.py:10],
        with EXACT before/after rows - including an old JobState.error(3) row.

All seeded rows are tracked by id and deleted at the end (net-zero). updated_at /
taken_at are back-dated with raw SQL so the ORM onupdate hook does not reset them."""
from server import create_app
app = create_app()
from app.db import Session
from app.models import Job, JobState
from app import config
from tasks.cleanup_old_jobs import cleanup_old_jobs
from sqlalchemy import text
import arrow

def backdate(job_id, updated_at=None, taken_at=None):
    if updated_at is not None:
        Session.execute(text("UPDATE job SET updated_at=:v WHERE id=:i"),
                        {"v": updated_at.naive, "i": job_id})
    if taken_at is not None:
        Session.execute(text("UPDATE job SET taken_at=:v WHERE id=:i"),
                        {"v": taken_at.naive, "i": job_id})
    Session.commit()

def row(job_id):
    r = Session.execute(text(
        "SELECT id, state, attempts, updated_at FROM job WHERE id=:i"),
        {"i": job_id}).fetchone()
    return None if r is None else tuple(r)

with app.app_context():
    seeded = []
    jobs_before = Session.query(Job).count()
    try:
        now = arrow.now()
        stale = now.shift(minutes=-(config.JOB_TAKEN_RETRY_WAIT_MINS + 1))  # 31 min ago
        fresh = now.shift(minutes=-5)
        old = now.shift(days=-10)
        future = now.shift(days=+1)

        # -------- Part 1: get_jobs_to_run eligibility matrix --------
        a = Job.create(name="blitzy-q2c-a", state=JobState.ready.value, run_at=None, commit=True); seeded.append(a.id)
        b = Job.create(name="blitzy-q2c-b", state=JobState.taken.value, taken=True, attempts=1, run_at=None, commit=True); seeded.append(b.id)
        backdate(b.id, taken_at=stale)
        c = Job.create(name="blitzy-q2c-c", state=JobState.taken.value, taken=True, attempts=config.JOB_MAX_ATTEMPTS, run_at=None, commit=True); seeded.append(c.id)
        backdate(c.id, taken_at=stale)
        d = Job.create(name="blitzy-q2c-d", state=JobState.taken.value, taken=True, attempts=1, run_at=None, commit=True); seeded.append(d.id)
        backdate(d.id, taken_at=fresh)
        e = Job.create(name="blitzy-q2c-e", state=JobState.ready.value, run_at=future.naive, commit=True); seeded.append(e.id)

        import job_runner
        eligible_ids = {j.id for j in job_runner.get_jobs_to_run()}
        print("=== Part 1: get_jobs_to_run() eligibility (JOB_TAKEN_RETRY_WAIT_MINS=%d, JOB_MAX_ATTEMPTS=%d) ==="
              % (config.JOB_TAKEN_RETRY_WAIT_MINS, config.JOB_MAX_ATTEMPTS))
        print("(a) ready, run_at=NULL                       -> eligible:", a.id in eligible_ids, "(expect True)")
        print("(b) taken, taken_at=31min ago, attempts=1    -> eligible:", b.id in eligible_ids, "(expect True)")
        print("(c) taken, taken_at=31min ago, attempts=5    -> eligible:", c.id in eligible_ids, "(expect False: attempts>=max)")
        print("(d) taken, taken_at=5min ago,  attempts=1    -> eligible:", d.id in eligible_ids, "(expect False: not stale yet)")
        print("(e) ready, run_at=+1 day                     -> eligible:", e.id in eligible_ids, "(expect False: run_at gate)")

        # -------- Part 2: cleanup_old_jobs before/after (incl. old error row) --------
        f = Job.create(name="blitzy-q2c-f", state=JobState.done.value, commit=True); seeded.append(f.id); backdate(f.id, updated_at=old)
        g = Job.create(name="blitzy-q2c-g", state=JobState.error.value, commit=True); seeded.append(g.id); backdate(g.id, updated_at=old)
        h = Job.create(name="blitzy-q2c-h", state=JobState.taken.value, taken=True, attempts=config.JOB_MAX_ATTEMPTS, commit=True); seeded.append(h.id); backdate(h.id, updated_at=old)
        i = Job.create(name="blitzy-q2c-i", state=JobState.done.value, commit=True); seeded.append(i.id); backdate(i.id, updated_at=now)  # recent -> survives
        j = Job.create(name="blitzy-q2c-j", state=JobState.taken.value, taken=True, attempts=1, commit=True); seeded.append(j.id); backdate(j.id, updated_at=old)  # taken attempts<max -> survives
        k = Job.create(name="blitzy-q2c-k", state=JobState.ready.value, commit=True); seeded.append(k.id); backdate(k.id, updated_at=old)  # ready -> survives

        oldest_allowed = now.shift(days=-1)
        print()
        print("=== Part 2: cleanup_old_jobs(oldest_allowed = now-1day = %s) ===" % oldest_allowed.format("YYYY-MM-DD HH:mm"))
        labels = {f.id:"(f) done(2)   old   -> DELETE", g.id:"(g) error(3)  old   -> DELETE",
                  h.id:"(h) taken(1) att=5 old -> DELETE", i.id:"(i) done(2)  recent(now) -> keep",
                  j.id:"(j) taken(1) att=1 old -> keep", k.id:"(k) ready(0)  old   -> keep"}
        print("--- BEFORE (id, state, attempts, updated_at) ---")
        for jid in [f.id,g.id,h.id,i.id,j.id,k.id]:
            print("   ", labels[jid], "->", row(jid))
        ret = cleanup_old_jobs(oldest_allowed)
        print("--- cleanup_old_jobs returned:", ret, "(it returns None; the deleted count appears in its own log line \"Deleted N jobs\" at cleanup_old_jobs.py:24) ---")
        print("--- AFTER (None = row was deleted) ---")
        for jid in [f.id,g.id,h.id,i.id,j.id,k.id]:
            print("   ", labels[jid], "->", row(jid))
    finally:
        # net-zero: delete every surviving seeded row
        for jid in seeded:
            if row(jid) is not None:
                Session.execute(text("DELETE FROM job WHERE id=:i"), {"i": jid})
        Session.commit()
        jobs_after = Session.query(Job).count()
        print()
        print("=== cleanup: jobs_before =", jobs_before, "| jobs_after =", jobs_after,
              "| net-zero =", jobs_before == jobs_after, "===")
```

**Command:**

```bash
docker exec sl_canon bash -lc '
export CONFIG=tests/test.env PYTHONPATH=/app DB_URI="postgresql://test:test@localhost:5432/test"
cd /app && /app/venv/bin/python /tmp/blitzy_obs/obs_q2_eligibility_cleanup.py 2>&1 | grep -v -E "SL - (DEBUG) -|load config|>>> URL|WARNING: Use|Upload files|>>> init"'
```

**Complete output — RUN 1** (the two `SL - INFO` lines are `cleanup_old_jobs`'s
own log output, shown verbatim):

```
=== Part 1: get_jobs_to_run() eligibility (JOB_TAKEN_RETRY_WAIT_MINS=30, JOB_MAX_ATTEMPTS=5) ===
(a) ready, run_at=NULL                       -> eligible: True (expect True)
(b) taken, taken_at=31min ago, attempts=1    -> eligible: True (expect True)
(c) taken, taken_at=31min ago, attempts=5    -> eligible: False (expect False: attempts>=max)
(d) taken, taken_at=5min ago,  attempts=1    -> eligible: False (expect False: not stale yet)
(e) ready, run_at=+1 day                     -> eligible: False (expect False: run_at gate)

=== Part 2: cleanup_old_jobs(oldest_allowed = now-1day = 2026-07-12 18:43) ===
--- BEFORE (id, state, attempts, updated_at) ---
    (f) done(2)   old   -> DELETE -> (1572, 2, 0, datetime.datetime(2026, 7, 3, 18, 43, 2, 948899))
    (g) error(3)  old   -> DELETE -> (1573, 3, 0, datetime.datetime(2026, 7, 3, 18, 43, 2, 948899))
    (h) taken(1) att=5 old -> DELETE -> (1574, 1, 5, datetime.datetime(2026, 7, 3, 18, 43, 2, 948899))
    (i) done(2)  recent(now) -> keep -> (1575, 2, 0, datetime.datetime(2026, 7, 13, 18, 43, 2, 948899))
    (j) taken(1) att=1 old -> keep -> (1576, 1, 1, datetime.datetime(2026, 7, 3, 18, 43, 2, 948899))
    (k) ready(0)  old   -> keep -> (1577, 0, 0, datetime.datetime(2026, 7, 3, 18, 43, 2, 948899))
2026-07-13 18:43:02,984 - SL - INFO - 5525 - "/app/tasks/cleanup_old_jobs.py:11" - cleanup_old_jobs() -  - Deleting jobs older than 2026-07-12T18:43:02.948899+00:00
2026-07-13 18:43:02,986 - SL - INFO - 5525 - "/app/tasks/cleanup_old_jobs.py:24" - cleanup_old_jobs() -  - Deleted 3 jobs
--- cleanup_old_jobs returned: None (it returns None; the deleted count appears in its own log line "Deleted N jobs" at cleanup_old_jobs.py:24) ---
--- AFTER (None = row was deleted) ---
    (f) done(2)   old   -> DELETE -> None
    (g) error(3)  old   -> DELETE -> None
    (h) taken(1) att=5 old -> DELETE -> None
    (i) done(2)  recent(now) -> keep -> (1575, 2, 0, datetime.datetime(2026, 7, 13, 18, 43, 2, 948899))
    (j) taken(1) att=1 old -> keep -> (1576, 1, 1, datetime.datetime(2026, 7, 3, 18, 43, 2, 948899))
    (k) ready(0)  old   -> keep -> (1577, 0, 0, datetime.datetime(2026, 7, 3, 18, 43, 2, 948899))

=== cleanup: jobs_before = 55 | jobs_after = 55 | net-zero = True ===
```

**Complete output — RUN 2** (stability; ids differ):

```
=== Part 1: get_jobs_to_run() eligibility (JOB_TAKEN_RETRY_WAIT_MINS=30, JOB_MAX_ATTEMPTS=5) ===
(a) ready, run_at=NULL                       -> eligible: True (expect True)
(b) taken, taken_at=31min ago, attempts=1    -> eligible: True (expect True)
(c) taken, taken_at=31min ago, attempts=5    -> eligible: False (expect False: attempts>=max)
(d) taken, taken_at=5min ago,  attempts=1    -> eligible: False (expect False: not stale yet)
(e) ready, run_at=+1 day                     -> eligible: False (expect False: run_at gate)

=== Part 2: cleanup_old_jobs(oldest_allowed = now-1day = 2026-07-12 18:43) ===
--- BEFORE (id, state, attempts, updated_at) ---
    (f) done(2)   old   -> DELETE -> (1583, 2, 0, datetime.datetime(2026, 7, 3, 18, 43, 59, 190766))
    (g) error(3)  old   -> DELETE -> (1584, 3, 0, datetime.datetime(2026, 7, 3, 18, 43, 59, 190766))
    (h) taken(1) att=5 old -> DELETE -> (1585, 1, 5, datetime.datetime(2026, 7, 3, 18, 43, 59, 190766))
    (i) done(2)  recent(now) -> keep -> (1586, 2, 0, datetime.datetime(2026, 7, 13, 18, 43, 59, 190766))
    (j) taken(1) att=1 old -> keep -> (1587, 1, 1, datetime.datetime(2026, 7, 3, 18, 43, 59, 190766))
    (k) ready(0)  old   -> keep -> (1588, 0, 0, datetime.datetime(2026, 7, 3, 18, 43, 59, 190766))
2026-07-13 18:43:59,226 - SL - INFO - 5572 - "/app/tasks/cleanup_old_jobs.py:11" - cleanup_old_jobs() -  - Deleting jobs older than 2026-07-12T18:43:59.190766+00:00
2026-07-13 18:43:59,228 - SL - INFO - 5572 - "/app/tasks/cleanup_old_jobs.py:24" - cleanup_old_jobs() -  - Deleted 3 jobs
--- cleanup_old_jobs returned: None (it returns None; the deleted count appears in its own log line "Deleted N jobs" at cleanup_old_jobs.py:24) ---
--- AFTER (None = row was deleted) ---
    (f) done(2)   old   -> DELETE -> None
    (g) error(3)  old   -> DELETE -> None
    (h) taken(1) att=5 old -> DELETE -> None
    (i) done(2)  recent(now) -> keep -> (1586, 2, 0, datetime.datetime(2026, 7, 13, 18, 43, 59, 190766))
    (j) taken(1) att=1 old -> keep -> (1587, 1, 1, datetime.datetime(2026, 7, 3, 18, 43, 59, 190766))
    (k) ready(0)  old   -> keep -> (1588, 0, 0, datetime.datetime(2026, 7, 3, 18, 43, 59, 190766))

=== cleanup: jobs_before = 55 | jobs_after = 55 | net-zero = True ===
```

**What this proves [OBSERVED]:**

- **Retry eligibility (before/after a failure).** `get_jobs_to_run()` returns a
  `taken` job again only once it is **stale** (`taken_at` older than 30 min) and
  **under the attempt ceiling** (`attempts < 5`): case (b) is eligible; case (c)
  (attempts = 5) and case (d) (only 5 min stale) are not; a future `run_at`
  (case e) is gated out. This is exactly how a failed (`taken`, `attempts`
  incremented) job is re-picked-up after the daemon restarts.
- **End-of-life cleanup incl. the `error(3)` row.** `cleanup_old_jobs` deleted
  the old `done(2)` (f), the old **`error(3)` (g)**, and the old `taken` with
  `attempts=5` (h) — the AFTER rows are `None`. The controls survived: recent
  `done` (i), old `taken` with `attempts=1` (j), and old `ready` (k). The real
  function logged `"Deleted 3 jobs"` (`tasks/cleanup_old_jobs.py:24`). Both runs
  identical; net-zero.

### Evidence D — the canonical harness tests pass (exit code 0)

The canonical Q2 test harnesses are run with coverage disabled (via
`-o addopts=""`, which drops the repo's `--cov` gate) so the **exit code reflects
the tests only**, not a coverage threshold.

**Command:**

```bash
docker exec sl_canon bash -lc '
export CONFIG=tests/test.env PYTHONPATH=/app DB_URI="postgresql://test:test@localhost:5432/test" PYTEST_ADDOPTS=""
cd /app
/app/venv/bin/python -m pytest tests/jobs/test_job_runner.py tests/tasks/test_cleanup_old_jobs.py -o addopts="" -p no:cacheprovider -v 2>&1 | grep -E "^tests/|passed|failed|error" | grep -v Warning
echo "PYTEST EXIT CODE = ${PIPESTATUS[0]}"'
```

**Complete output:**

```
tests/jobs/test_job_runner.py::test_get_jobs_to_run PASSED               [ 50%]
tests/tasks/test_cleanup_old_jobs.py::test_cleanup_old_jobs PASSED       [100%]
======================== 2 passed, 18 warnings in 0.04s ========================
PYTEST EXIT CODE = 0
```

**[OBSERVED]** both canonical harness tests pass; **exit code 0**. (The 18
warnings are pre-existing third-party `DeprecationWarning`s from `pkg_resources`,
`flask_limiter`, and `gnupg`, unrelated to Q2.)

### Q2 file:line reference table

| Fact | Value | Location | Label |
|------|-------|----------|-------|
| Job model | `Job` | `app/models.py:2683` | [SOURCE-VERIFIED] |
| State enum | ready=0, taken=1, done=2, error=3 | `app/models.py:253-257` | [OBSERVED] |
| Daemon main loop | `while True: … for job in get_jobs_to_run()` | `job_runner.py:329-347` | [SOURCE-VERIFIED] |
| Mark taken + attempts++ | `state=taken; attempts+=1; commit` | `job_runner.py:337-341` | [OBSERVED] |
| Work dispatcher | `process_job(job)` | `job_runner.py:342`, def at `:188` | [OBSERVED] |
| Set done (only on success) | `job.state = JobState.done.value` | `job_runner.py:344` | [OBSERVED] |
| No try/except | (grep: no output) | `job_runner.py` | [SOURCE-VERIFIED] |
| Eligibility selector | `get_jobs_to_run()` | `job_runner.py:307-326` | [OBSERVED] |
| Retry wait | `JOB_TAKEN_RETRY_WAIT_MINS = 30` | `app/config.py:565` | [OBSERVED] |
| Max attempts | `JOB_MAX_ATTEMPTS = 5` | `app/config.py:564` | [OBSERVED] |
| Failing dispatch | `handle_batch_import(None)` → `AttributeError` | `app/import_utils.py:23` | [OBSERVED] |
| Unknown-name branch | `LOG.e("Unknown job name %s", …)` | `job_runner.py:304` | [SOURCE-VERIFIED] |
| End-of-life cleanup | `cleanup_old_jobs(oldest_allowed)` | `tasks/cleanup_old_jobs.py:10-24` | [OBSERVED] |
| Cleanup invoked by cron | `delete_old_data()` → `cleanup_old_jobs` | `cron.py:1245-1248` | [SOURCE-VERIFIED] |
| Cron schedule | `"30 5 * * *"` "SimpleLogin Delete Old data" | `crontab.yml:40-44` | [SOURCE-VERIFIED] |
| `JobState.error` never set by runner | only in cleanup filter + tests | grep evidence | [SOURCE-VERIFIED] |

### Q2 coverage confirmation

- **Complete lifecycle from creation to completion** → `ready(0)` at creation →
  daemon sets `taken(1)` + `attempts++` → `process_job` → `done(2)` on success;
  observed `state 0→2`, `attempts 0→1` on a real `onboarding-1` job **[OBSERVED]**.
- **Error during execution — (a) recovery/retry behavior** → no in-process retry;
  the process exits on the unhandled exception **[OBSERVED]**. Retry is therefore
  only possible after the daemon restarts: `get_jobs_to_run()` re-selects a
  `taken` job once it is 30-min stale and `attempts < 5`, and refuses it at
  `attempts = 5` — both boundaries observed in Evidence C **[OBSERVED]**; that a
  restarted daemon then carries such a job through to completion across up to 5
  attempts follows from those observations but was not run end-to-end to
  exhaustion **[INFERRED]**.
- **Error during execution — (b) observable state reflecting the failure** →
  the job stays **`state=taken(1)` with `attempts` incremented** (observed
  `state=1, attempts=1`); it is **not** set to `error(3)` **[OBSERVED]**.
- **End-of-life** → `cleanup_old_jobs` deletes old `done`/`error`/max-attempt
  `taken` jobs, scheduled via `crontab.yml` **[OBSERVED]**.


## Q3 — Email-forwarding bounce handling (VERP address format and direction-dependent behavior)

### Direct answer

**The bounce address format.** During forwarding, SimpleLogin generates the
return-path (envelope `MAIL FROM`, i.e. the VERP / bounce address) with the
function `generate_verp_email(verp_type, object_id, sender_domain)`
[app/email_utils.py:1438]. The exact, lower-cased format is **[OBSERVED]**:

```
{VERP_PREFIX}.{base32(payload)}.{base32(hmac_signature)}@{domain}
```

where, from the running code [app/email_utils.py:1446-1464] **[SOURCE-VERIFIED]**:

- `payload = [verp_type.value, object_id, minutes_since_2022]` — a JSON list;
  `object_id` is the `EmailLog.id`; `minutes_since_2022` is
  `int((time.time() - VERP_TIME_START) / 60)` with `VERP_TIME_START = 1640995200`
  (2022-01-01) [app/email_utils.py:68,1449].
- `hmac_signature` = first 8 bytes of `HMAC(VERP_EMAIL_SECRET, payload, "sha3-224")`
  (`VERP_HMAC_ALGO = "sha3-224"` [app/email_utils.py:69]).
- both `payload` and `hmac_signature` are base32-encoded with `=` padding stripped.
- `VERP_PREFIX = "sl"` [app/config.py:500] and `domain` is the contact domain in
  the forward phase and the alias domain in the reply phase.

A concrete generated pair (object_id `987654`) **[OBSERVED]**:

```
bounce_forward -> sl.lmycyibzha3tmnjufqqdemzygi4tcm25.rund5goiafprk@contact-domain.com
bounce_reply   -> sl.lmysyibzha3tmnjufqqdemzygi4tcm25.6wxzoxfwuqhlu@sl.local
```

A **legacy** form is still recognized on receipt (not generated): forward =
`BOUNCE_PREFIX + email_log.id + BOUNCE_SUFFIX` = `bounce+987654+@sl.local`, parsed
by `parse_id_from_bounce` [app/email_utils.py:1258]; reply legacy prefix is
`BOUNCE_PREFIX_FOR_REPLY_PHASE = "bounce_reply"` (no trailing `+`)
[app/config.py:100-101,108-110] **[OBSERVED]**.

**(a) How the system identifies the original email.** When a failure notification
arrives, `handle()` calls `get_verp_info_from_email(rcpt_tos[0])`
[email_handler.py:2035, app/email_utils.py:1467], which splits the address,
**verifies the HMAC signature** (`expected_signature != signature -> return None`
[app/email_utils.py:1490]), and returns `(VerpType, object_id)`. The `object_id`
is the `EmailLog.id`; `handle()` then loads that `EmailLog` and recovers the
contact, alias, mailbox and user from it. Round-trip decode **[OBSERVED]**:

```
decode(forward) -> (<VerpType.bounce_forward: 0>, 987654)
decode(reply)   -> (<VerpType.bounce_reply: 1>, 987654)
decode(tampered signature) -> None
```

**Signature/lifetime nuance (corrects a common misread).** The only time-related
check is at [app/email_utils.py:1496]:

```python
if data[2] > (time.time() + config.VERP_MESSAGE_LIFETIME - VERP_TIME_START) / 60:
    return None
```

This rejects a token **only when its embedded timestamp is more than
`VERP_MESSAGE_LIFETIME` (5 days, `432000 s`) in the _future_** — a clock-sanity /
future-bound guard, **not an expiry**. There is **no lower bound**: an old signed
token (even 100 days old) still decodes and is accepted. Observed directly by
shifting only the generator's wall clock and decoding at the real time
**[OBSERVED]**:

```
6-day-OLD     token decode -> (<VerpType.bounce_forward: 0>, 987654)   # accepted
100-day-OLD   token decode -> (<VerpType.bounce_forward: 0>, 987654)   # accepted
6-day-FUTURE  token decode -> None                                     # rejected (>5d future)
4-day-FUTURE  token decode -> (<VerpType.bounce_forward: 0>, 987654)   # accepted (<5d future)
```

Tamper-resistance therefore comes from the **HMAC signature**, not from an age
limit; `VERP_MESSAGE_LIFETIME` bounds only how far into the future a token may be
dated.

**(b) + (c) State changes recorded, and how handling differs by direction.**
Direction is decided by `EmailLog.is_reply` inside `handle_bounce`
[email_handler.py:1851,1873]; a real bounce is one where
`is_bounce(envelope, msg)` is True — i.e. `envelope.mail_from == "<>"` **and** the
message is `multipart/report` [email_handler.py:1813]. Driving the canonical
`email_handler.handle()` with a real `multipart/report` DSN produced
**[OBSERVED]**:

| Aspect | Forward phase (`is_reply=False`) | Reply phase (`is_reply=True`) |
|--------|----------------------------------|-------------------------------|
| Handler | `handle_bounce_forward_phase` [L1432] | `handle_bounce_reply_phase` [L1595] |
| SMTP status returned | **`E211`** = `250 SL E211 Bounce Forward phase handled` | **`E212`** = `250 SL E212 Bounce Reply phase handled` |
| `Bounce.create(email=…)` keyed on | **`mailbox.email`** | **`contact.website_email`** (sanitized) |
| `RefusedEmail` stored | yes (local file under `UPLOAD_DIR`) | yes (local file under `UPLOAD_DIR`) |
| `email_log.bounced` | set `True` | set `True` |
| `email_log.refused_email_id` | set | set |
| `email_log.bounced_mailbox_id` | set to the mailbox id | set to the mailbox id |
| user `Notification` | **always** — a delivery-failure notice, or a "disabled due to multiple bounces" notice when `should_disable()` is True | **always** — "Email cannot be sent to … from your alias …" |

The two directions therefore differ in **which SMTP status is returned**
(`E211` vs `E212`), **which email the `Bounce` row is keyed on** (the mailbox that
could not receive vs the external contact that rejected the reply), and the
**wording of the user `Notification`**.

**Edge branch — auto-disable after repeated bounces.** In the forward phase, after
recording the bounce the handler calls `should_disable(alias)`
[app/email_utils.py]; when `nb_bounced_last_24h > 12` it returns
`(True, "+12 bounces in the last 24h")`, and the handler flips the alias to
`enabled=False` and creates a "disabled due to multiple bounces" `Notification`.
Observed the exact transition `enabled: True -> (12 bounces) True -> (13th bounce)
False` **[OBSERVED]**.

**Edge branch — the auto-reply exception (`email_log.auto_replied`).** A modern,
**HMAC-signed reply-VERP** address that carries a message that is **not** a bounce
(not `multipart/report`, or `mail_from != "<>"`) does **not** enter an auto-reply;
`handle()`'s reply-VERP branch takes its `else` clause and raises `VERPReply`
[email_handler.py:2095], which `handle_DATA` catches and converts to **`E213`**
(`250 SL E213 Unknown email ignored`) [email_handler.py:2308-2318], leaving
`email_log.auto_replied = False`. The `auto_replied = True` path
[email_handler.py:1887] is reached **only** when `handle_bounce` is invoked on
a reply `EmailLog` with a non-report message **without** the `is_bounce` gate — in
production the **iCloud un-gated caller** [email_handler.py:2101-2116], which
calls `handle_bounce(envelope, email_log, msg)` directly. Both outcomes observed
**[OBSERVED]**:

```
(a) modern reply-VERP + non-report via handle_DATA -> '250 SL E213 …' ; auto_replied=False
(b) iCloud legacy mail_from via handle_DATA         -> '250 Message accepted …' ; auto_replied=True
```

The evidence, embedded scripts, and complete unedited output follow.


### Evidence A — VERP address format and decode semantics (`generate_verp_email` / `get_verp_info_from_email`)

This drives the **real** generator and decoder. For the lifetime scenarios it
shifts **only the wall clock seen by the generator** (via a `time.time` mock) so
the real function emits a token dated in the past/future; the real decoder then
runs at the true current time. The clock-shift is the only non-default input and
is disclosed here **[boundary disclosure]**.

Embedded script (`/tmp/blitzy_obs/obs_q3_verp.py`, complete):

```python
"""Q3 Part 1 - VERP address FORMAT + decode semantics, driving the REAL
generate_verp_email and get_verp_info_from_email [app/email_utils.py:1438,1467].

Read-only (no DB writes). For the lifetime scenarios we shift ONLY the wall clock
seen by the real generator (via mock on time.time) so the REAL function emits a
token dated in the past/future; the REAL decoder then runs at the true current
time. This isolates exactly what the L1496 check does. The clock-shift is
disclosed and is the only non-default input."""
from server import create_app
app = create_app()
import time as _time
from unittest import mock
from app.email_utils import (generate_verp_email, get_verp_info_from_email,
                             parse_id_from_bounce, VERP_TIME_START, VERP_HMAC_ALGO)
from app.models import VerpType
from app import config

OBJ = 987654
DAY = 86400

with app.app_context():
    print("VERP_PREFIX =", repr(config.VERP_PREFIX), "| EMAIL_DOMAIN =", repr(config.EMAIL_DOMAIN))
    print("VERP_HMAC_ALGO =", repr(VERP_HMAC_ALGO), "| VERP_TIME_START =", VERP_TIME_START)
    print("VERP_MESSAGE_LIFETIME =", config.VERP_MESSAGE_LIFETIME, "s =",
          config.VERP_MESSAGE_LIFETIME / DAY, "days")

    print()
    print("=== (1) exact address FORMAT for BOTH directions (generated NOW), object_id =", OBJ, "===")
    fwd = generate_verp_email(VerpType.bounce_forward, OBJ, "contact-domain.com")
    rep = generate_verp_email(VerpType.bounce_reply, OBJ, config.EMAIL_DOMAIN)
    print("bounce_forward ->", fwd)
    print("bounce_reply   ->", rep)
    print("structure = {VERP_PREFIX}.{base32(payload)}.{base32(hmac_sig)}@{domain}, lowercased")
    print("  forward username split by '.' =", fwd.split("@")[0].split("."))
    print("  forward domain =", fwd.split("@")[1], "| reply domain =", rep.split("@")[1])

    print()
    print("=== (2) round-trip decode via get_verp_info_from_email ===")
    print("decode(forward) ->", get_verp_info_from_email(fwd))
    print("decode(reply)   ->", get_verp_info_from_email(rep))

    print()
    print("=== (3) tamper the signature -> HMAC check fails -> None (L1490) ===")
    user, dom = fwd.split("@")
    p, payload_b32, sig_b32 = user.split(".")
    bad_sig = ("aa" + sig_b32[2:]) if not sig_b32.startswith("aa") else ("bb" + sig_b32[2:])
    tampered = "{}.{}.{}@{}".format(p, payload_b32, bad_sig, dom)
    print("tampered addr ->", tampered)
    print("decode(tampered) ->", get_verp_info_from_email(tampered))

    print()
    print("=== (4) LIFETIME check L1496 - old tokens ACCEPTED, only >5-day-FUTURE rejected ===")
    real_now = _time.time()
    def gen_at(offset_days):
        # the REAL generator, but with its wall clock shifted by offset_days
        with mock.patch("time.time", return_value=real_now + offset_days * DAY):
            return generate_verp_email(VerpType.bounce_forward, OBJ, config.EMAIL_DOMAIN)
    for label, off in [("6-day-OLD    ", -6), ("100-day-OLD  ", -100),
                       ("6-day-FUTURE ", +6), ("4-day-FUTURE ", +4)]:
        tok = gen_at(off)
        # decode runs at the REAL current time (no patch)
        print("%s token decode ->" % label, get_verp_info_from_email(tok))

    print()
    print("=== (5) legacy BOUNCE_PREFIX form parsed by parse_id_from_bounce (L1258) ===")
    legacy_fwd = config.BOUNCE_PREFIX + str(OBJ) + config.BOUNCE_SUFFIX
    print("legacy forward addr =", legacy_fwd)
    print("parse_id_from_bounce ->", parse_id_from_bounce(legacy_fwd))
    print("BOUNCE_PREFIX =", repr(config.BOUNCE_PREFIX), "| BOUNCE_SUFFIX =", repr(config.BOUNCE_SUFFIX))
    print("BOUNCE_PREFIX_FOR_REPLY_PHASE =", repr(config.BOUNCE_PREFIX_FOR_REPLY_PHASE), "(no trailing '+')")
```

Exact command (RUN 1), including the declared noise filter:

```bash
docker exec sl_canon bash -lc '
export CONFIG=tests/test.env PYTHONPATH=/app DB_URI="postgresql://test:test@localhost:5432/test"
FILT="SL - (DEBUG|INFO) -|load config file|>>> URL|WARNING: Use a temp|Upload files to local|>>> init logging"
/app/venv/bin/python /tmp/blitzy_obs/obs_q3_verp.py 2>&1 | grep -vE "$FILT"'
```

Complete unedited output (RUN 1):

```text
VERP_PREFIX = 'sl' | EMAIL_DOMAIN = 'sl.local'
VERP_HMAC_ALGO = 'sha3-224' | VERP_TIME_START = 1640995200
VERP_MESSAGE_LIFETIME = 432000 s = 5.0 days

=== (1) exact address FORMAT for BOTH directions (generated NOW), object_id = 987654 ===
bounce_forward -> sl.lmycyibzha3tmnjufqqdemzygi4tcm25.rund5goiafprk@contact-domain.com
bounce_reply   -> sl.lmysyibzha3tmnjufqqdemzygi4tcm25.6wxzoxfwuqhlu@sl.local
structure = {VERP_PREFIX}.{base32(payload)}.{base32(hmac_sig)}@{domain}, lowercased
  forward username split by '.' = ['sl', 'lmycyibzha3tmnjufqqdemzygi4tcm25', 'rund5goiafprk']
  forward domain = contact-domain.com | reply domain = sl.local

=== (2) round-trip decode via get_verp_info_from_email ===
decode(forward) -> (<VerpType.bounce_forward: 0>, 987654)
decode(reply)   -> (<VerpType.bounce_reply: 1>, 987654)

=== (3) tamper the signature -> HMAC check fails -> None (L1490) ===
tampered addr -> sl.lmycyibzha3tmnjufqqdemzygi4tcm25.aand5goiafprk@contact-domain.com
decode(tampered) -> None

=== (4) LIFETIME check L1496 - old tokens ACCEPTED, only >5-day-FUTURE rejected ===
6-day-OLD     token decode -> (<VerpType.bounce_forward: 0>, 987654)
100-day-OLD   token decode -> (<VerpType.bounce_forward: 0>, 987654)
6-day-FUTURE  token decode -> None
4-day-FUTURE  token decode -> (<VerpType.bounce_forward: 0>, 987654)

=== (5) legacy BOUNCE_PREFIX form parsed by parse_id_from_bounce (L1258) ===
legacy forward addr = bounce+987654+@sl.local
parse_id_from_bounce -> 987654
BOUNCE_PREFIX = 'bounce+' | BOUNCE_SUFFIX = '+@sl.local'
BOUNCE_PREFIX_FOR_REPLY_PHASE = 'bounce_reply' (no trailing '+')
```

**Two-run stability (F1).** The script is DB-independent and its behavioral output
— the address grammar, the decode round-trips, the tamper→None result, the lifetime
accept/reject boundary, the legacy parse, and the printed constants — is identical on
every run. RUN 1 and RUN 2 were issued in the same clock-minute, so even the address
generated "now" in step (1) — which encodes the generation time to minute
granularity — matched, making the two runs **byte-for-byte identical**, verified by
capturing both to files and diffing:

```bash
diff /tmp/blitzy_obs/q3_verp_run1.txt /tmp/blitzy_obs/q3_verp_run2.txt && echo "IDENTICAL (F1 deterministic)"
# -> IDENTICAL (F1 deterministic)
```


### Evidence B — direction-dependent dispatch through canonical `email_handler.handle()`

This drives the **canonical** module-level entry `email_handler.handle(envelope, msg)`
with a real `multipart/report` DSN (the committed fixture
`local_data/email_tests/bounce.eml`) and `envelope.mail_from = "<>"`, so
`is_bounce(...)` is True and the message routes to `handle_bounce`. It seeds a real
`User`/`Alias`/`Contact`/`EmailLog` for each direction (`is_reply=False` then
`is_reply=True`), records before/after state, and cleans up with an explicit
deletion cascade (net-zero counts printed at the end).

**Boundary disclosure (F9):** with the default `tests/test.env`, `NOT_SEND_EMAIL=True`
suppresses the outbound alert email (no real SMTP send) and `LOCAL_FILE_UPLOAD=True`
stores each `RefusedEmail` to a **local file** under `UPLOAD_DIR` instead of S3; the
exact local paths are printed and the files this script creates are deleted at the end.

Embedded script (`/tmp/blitzy_obs/obs_q3_handler.py`, complete):

```python
"""
obs_q3_handler.py — Q3 direction-dependent bounce dispatch, driven through the
CANONICAL entry point email_handler.handle(envelope, msg).

Forward-phase bounce  -> handle_bounce_forward_phase -> status E211
Reply-phase   bounce  -> handle_bounce_reply_phase   -> status E212

Boundary substitutes (disclosed adjacent to the claims, F9):
  - NOT_SEND_EMAIL=True   : the alert email send is suppressed (no real SMTP out).
  - LOCAL_FILE_UPLOAD=True : RefusedEmail bodies are written to local files under
                             UPLOAD_DIR instead of S3; the exact local paths are shown.

Cleanup is explicit deletion-cascade (NOT rollback): every row and every local
file this script creates is removed, and net-zero counts are printed at the end.
"""
import os
import email

from server import create_app

app = create_app()

from app import config
from app.db import Session
from app.models import (
    User,
    Alias,
    Contact,
    EmailLog,
    Mailbox,
    Bounce,
    RefusedEmail,
    Notification,
    Job,
    VerpType,
)
from app.email_utils import generate_verp_email
from app.email import status
import email_handler
from tests.utils import create_new_user

ROOT = "/app"
BOUNCE_EML = os.path.join(ROOT, "local_data", "email_tests", "bounce.eml")


def load_dsn():
    """Fresh multipart/report DSN each call (handle() mutates msg headers)."""
    with open(BOUNCE_EML, "rb") as f:
        return email.message_from_bytes(f.read())


def counts():
    return {
        "Bounce": Session.query(Bounce).count(),
        "RefusedEmail": Session.query(RefusedEmail).count(),
        "Notification": Session.query(Notification).count(),
        "EmailLog": Session.query(EmailLog).count(),
        "User": Session.query(User).count(),
        "Job": Session.query(Job).count(),
    }


def refused_files():
    d = os.path.join(config.UPLOAD_DIR, "refused-emails")
    if not os.path.isdir(d):
        return []
    return sorted(os.listdir(d))


with app.app_context():
    print("=" * 78)
    print("BOUNDARY DISCLOSURE (F9) — default test.env configuration in effect:")
    print("  config.NOT_SEND_EMAIL  =", config.NOT_SEND_EMAIL,
          "  -> outbound alert email is SUPPRESSED (no real SMTP send)")
    print("  config.LOCAL_FILE_UPLOAD =", config.LOCAL_FILE_UPLOAD,
          "-> RefusedEmail stored to LOCAL file, not S3")
    print("  config.UPLOAD_DIR      =", config.UPLOAD_DIR)
    print("  config.EMAIL_DOMAIN    =", config.EMAIL_DOMAIN)
    print("=" * 78)

    base_counts = counts()
    base_max_bounce = Session.query(Bounce).order_by(Bounce.id.desc()).first()
    base_max_bounce_id = base_max_bounce.id if base_max_bounce else 0
    base_max_ref = Session.query(RefusedEmail).order_by(RefusedEmail.id.desc()).first()
    base_max_ref_id = base_max_ref.id if base_max_ref else 0
    base_max_job = Session.query(Job).order_by(Job.id.desc()).first()
    base_max_job_id = base_max_job.id if base_max_job else 0
    base_files = refused_files()
    print("BASELINE counts:", base_counts)
    print("BASELINE refused-emails file count:", len(base_files))
    print()

    created_user_ids = []
    created_file_paths = []  # absolute local paths to remove at cleanup

    # ================================================================
    # FORWARD PHASE  (email_log.is_reply = False)  -> expect E211
    # ================================================================
    print("#" * 78)
    print("# FORWARD-PHASE BOUNCE  (EmailLog.is_reply = False)")
    print("#" * 78)
    uf = create_new_user()
    created_user_ids.append(uf.id)
    mbf = uf.default_mailbox
    aliasf = Alias.create_new_random(uf)
    Session.commit()
    contactf = Contact.create(
        user_id=uf.id,
        alias_id=aliasf.id,
        website_email="fwd-victim@example.com",
        reply_email="rep-fwd@sl.local",
        commit=True,
    )
    elf = EmailLog.create(
        user_id=uf.id,
        contact_id=contactf.id,
        alias_id=aliasf.id,
        mailbox_id=mbf.id,
        is_reply=False,
        commit=True,
    )
    print("[SETUP] user=%d alias=%s mailbox.email=%s contact.website_email=%s email_log.id=%d is_reply=%s"
          % (uf.id, aliasf.email, mbf.email, contactf.website_email, elf.id, elf.is_reply))
    print("[BEFORE] email_log.bounced=%s refused_email_id=%s bounced_mailbox_id=%s"
          % (elf.bounced, elf.refused_email_id, elf.bounced_mailbox_id))

    verp_fwd = generate_verp_email(VerpType.bounce_forward, elf.id)
    print("[VERP rcpt] bounce_forward address =", verp_fwd)

    from aiosmtpd.smtp import Envelope
    env = Envelope()
    env.mail_from = "<>"
    env.rcpt_tos = [verp_fwd]
    msg = load_dsn()
    print("[msg content-type] =", msg.get_content_type(), " mail_from =", env.mail_from)

    res_fwd = email_handler.handle(env, msg)
    print("[RESULT] handle() ->", repr(res_fwd))
    print("[ASSERT] res == status.E211 :", res_fwd == status.E211, "  (E211 =", repr(status.E211), ")")

    Session.expire(elf)
    print("[AFTER ] email_log.bounced=%s refused_email_id=%s bounced_mailbox_id=%s"
          % (elf.bounced, elf.refused_email_id, elf.bounced_mailbox_id))
    print("[AFTER ] bounced_mailbox_id == forward mailbox.id (%d) :" % mbf.id,
          elf.bounced_mailbox_id == mbf.id)

    new_bounces_f = Session.query(Bounce).filter(Bounce.id > base_max_bounce_id).all()
    for b in new_bounces_f:
        print("[Bounce] id=%d email=%r  == mailbox.email(%r)? %s"
              % (b.id, b.email, mbf.email, b.email == mbf.email))
    new_ref_f = Session.query(RefusedEmail).filter(RefusedEmail.id > base_max_ref_id).all()
    for r in new_ref_f:
        fp_full = os.path.join(config.UPLOAD_DIR, r.full_report_path) if r.full_report_path else None
        fp_orig = os.path.join(config.UPLOAD_DIR, r.path) if r.path else None
        print("[RefusedEmail] id=%d full_report_path=%s path=%s" % (r.id, r.full_report_path, r.path))
        print("   [F9 LOCAL_FILE_UPLOAD] full_report file exists=%s @ %s"
              % (os.path.exists(fp_full) if fp_full else None, fp_full))
        if fp_orig:
            print("   [F9 LOCAL_FILE_UPLOAD] orig file exists=%s @ %s"
                  % (os.path.exists(fp_orig), fp_orig))
        if fp_full:
            created_file_paths.append(fp_full)
        if fp_orig:
            created_file_paths.append(fp_orig)
    notifs_f = Session.query(Notification).filter(Notification.user_id == uf.id).all()
    n_notif_f = len(notifs_f)
    print("[Notification] count for forward user =", n_notif_f,
          "  (forward phase ALWAYS creates a user Notification: a delivery-failure notice,")
    print("               or a 'disabled due to multiple bounces' notice when should_disable() is True)")
    for nt in notifs_f:
        print("   [Notification.title] =", repr(nt.title))
    print()

    # ================================================================
    # REPLY PHASE  (email_log.is_reply = True)  -> expect E212
    # ================================================================
    print("#" * 78)
    print("# REPLY-PHASE BOUNCE  (EmailLog.is_reply = True)")
    print("#" * 78)
    ur = create_new_user()
    created_user_ids.append(ur.id)
    mbr = ur.default_mailbox
    aliasr = Alias.create_new_random(ur)
    Session.commit()
    contactr = Contact.create(
        user_id=ur.id,
        alias_id=aliasr.id,
        website_email="reply-victim@example.com",
        reply_email="rep-rep@sl.local",
        commit=True,
    )
    elr = EmailLog.create(
        user_id=ur.id,
        contact_id=contactr.id,
        alias_id=aliasr.id,
        mailbox_id=mbr.id,
        is_reply=True,
        commit=True,
    )
    print("[SETUP] user=%d alias=%s mailbox.email=%s contact.website_email=%s email_log.id=%d is_reply=%s"
          % (ur.id, aliasr.email, mbr.email, contactr.website_email, elr.id, elr.is_reply))
    print("[BEFORE] email_log.bounced=%s refused_email_id=%s bounced_mailbox_id=%s"
          % (elr.bounced, elr.refused_email_id, elr.bounced_mailbox_id))

    verp_rep = generate_verp_email(VerpType.bounce_reply, elr.id)
    print("[VERP rcpt] bounce_reply address =", verp_rep)

    env2 = Envelope()
    env2.mail_from = "<>"
    env2.rcpt_tos = [verp_rep]
    msg2 = load_dsn()
    print("[msg content-type] =", msg2.get_content_type(), " mail_from =", env2.mail_from)

    base_max_bounce2 = Session.query(Bounce).order_by(Bounce.id.desc()).first()
    base_max_bounce2_id = base_max_bounce2.id if base_max_bounce2 else 0
    base_max_ref2 = Session.query(RefusedEmail).order_by(RefusedEmail.id.desc()).first()
    base_max_ref2_id = base_max_ref2.id if base_max_ref2 else 0

    res_rep = email_handler.handle(env2, msg2)
    print("[RESULT] handle() ->", repr(res_rep))
    print("[ASSERT] res == status.E212 :", res_rep == status.E212, "  (E212 =", repr(status.E212), ")")

    Session.expire(elr)
    print("[AFTER ] email_log.bounced=%s refused_email_id=%s bounced_mailbox_id=%s"
          % (elr.bounced, elr.refused_email_id, elr.bounced_mailbox_id))
    print("[AFTER ] refused_email_id set? %s   bounced_mailbox_id == reply mailbox.id(%d)? %s"
          % (elr.refused_email_id is not None, mbr.id, elr.bounced_mailbox_id == mbr.id))

    new_bounces_r = Session.query(Bounce).filter(Bounce.id > base_max_bounce2_id).all()
    for b in new_bounces_r:
        print("[Bounce] id=%d email=%r  == contact.website_email(%r)? %s"
              % (b.id, b.email, contactr.website_email, b.email == contactr.website_email))
    new_ref_r = Session.query(RefusedEmail).filter(RefusedEmail.id > base_max_ref2_id).all()
    for r in new_ref_r:
        fp_full = os.path.join(config.UPLOAD_DIR, r.full_report_path) if r.full_report_path else None
        fp_orig = os.path.join(config.UPLOAD_DIR, r.path) if r.path else None
        print("[RefusedEmail] id=%d full_report_path=%s path=%s" % (r.id, r.full_report_path, r.path))
        print("   [F9 LOCAL_FILE_UPLOAD] full_report file exists=%s @ %s"
              % (os.path.exists(fp_full) if fp_full else None, fp_full))
        if fp_orig:
            print("   [F9 LOCAL_FILE_UPLOAD] orig file exists=%s @ %s"
                  % (os.path.exists(fp_orig), fp_orig))
        if fp_full:
            created_file_paths.append(fp_full)
        if fp_orig:
            created_file_paths.append(fp_orig)
    notifs_r = Session.query(Notification).filter(Notification.user_id == ur.id).all()
    print("[Notification] count for reply user =", len(notifs_r),
          "  (reply phase ALWAYS creates a user Notification)")
    for nt in notifs_r:
        print("   [Notification.title] =", repr(nt.title))
    print()

    # ================================================================
    # DIRECTION COMPARISON SUMMARY
    # ================================================================
    print("#" * 78)
    print("# DIRECTION-DEPENDENT SUMMARY (observed)")
    print("#" * 78)
    print("  FORWARD (is_reply=False): status=%r" % (res_fwd,))
    print("     Bounce.email = mailbox.email (%r)" % (mbf.email,))
    print("     Notification(s)=%d  title=%r" % (n_notif_f, notifs_f[0].title if notifs_f else None))
    print("  REPLY   (is_reply=True) : status=%r" % (res_rep,))
    print("     Bounce.email = contact.website_email (%r)" % (contactr.website_email,))
    print("     Notification(s)=%d  title=%r" % (len(notifs_r), notifs_r[0].title if notifs_r else None))
    print()

    # ================================================================
    # CLEANUP — explicit deletion cascade + local file removal
    # ================================================================
    print("#" * 78)
    print("# CLEANUP (explicit deletion cascade — NOT rollback)")
    print("#" * 78)
    # 1) remove local refused-email files this script created
    for fp in created_file_paths:
        try:
            if os.path.exists(fp):
                os.remove(fp)
                print("  removed local file:", fp)
        except OSError as e:
            print("  WARN could not remove", fp, e)
    # 2) delete users (cascades Alias/Contact/EmailLog/RefusedEmail rows/Notification)
    for uid in created_user_ids:
        User.delete(uid)
    Session.commit()
    # 3) delete Bounce rows created (Bounce has no user FK)
    Session.query(Bounce).filter(Bounce.id > base_max_bounce_id).delete(synchronize_session=False)
    # 4) delete onboarding Job rows created by create_new_user
    Session.query(Job).filter(Job.id > base_max_job_id).delete(synchronize_session=False)
    Session.commit()

    final_counts = counts()
    final_files = refused_files()
    print("FINAL counts:   ", final_counts)
    print("BASELINE counts:", base_counts)
    print("NET-ZERO (counts identical)?", final_counts == base_counts)
    print("FINAL refused-emails file count:", len(final_files),
          " BASELINE:", len(base_files),
          " NET-ZERO?", len(final_files) == len(base_files))
```

Exact command (RUN 1), including the declared noise filter:

```bash
docker exec sl_canon bash -lc '
export CONFIG=tests/test.env PYTHONPATH=/app DB_URI="postgresql://test:test@localhost:5432/test"
FILT="SL - (DEBUG|INFO) -|load config file|>>> URL|WARNING: Use a temp|Upload files to local|>>> init logging|add missing content-transfer-encoding|id F09806333D"
/app/venv/bin/python /tmp/blitzy_obs/obs_q3_handler.py 2>&1 | grep -vE "$FILT"'
```

Complete unedited output (RUN 1):

```text
==============================================================================
BOUNDARY DISCLOSURE (F9) — default test.env configuration in effect:
  config.NOT_SEND_EMAIL  = True   -> outbound alert email is SUPPRESSED (no real SMTP send)
  config.LOCAL_FILE_UPLOAD = True -> RefusedEmail stored to LOCAL file, not S3
  config.UPLOAD_DIR      = /app/static/upload
  config.EMAIL_DOMAIN    = sl.local
==============================================================================
BASELINE counts: {'Bounce': 9, 'RefusedEmail': 0, 'Notification': 0, 'EmailLog': 1, 'User': 26, 'Job': 0}
BASELINE refused-emails file count: 14

##############################################################################
# FORWARD-PHASE BOUNCE  (EmailLog.is_reply = False)
##############################################################################
[SETUP] user=506 alias=comfit_clomps867@sl.local mailbox.email=user_jt9j5qbfh3@mailbox.test contact.website_email=fwd-victim@example.com email_log.id=413 is_reply=False
[BEFORE] email_log.bounced=False refused_email_id=None bounced_mailbox_id=None
[VERP rcpt] bounce_forward address = sl.lmycyibugezsyibsgm4deojrgfoq.vrstqojvv4z7q@sl.local
[msg content-type] = multipart/report  mail_from = <>
[RESULT] handle() -> '250 SL E211 Bounce Forward phase handled'
[ASSERT] res == status.E211 : True   (E211 = '250 SL E211 Bounce Forward phase handled' )
[AFTER ] email_log.bounced=True refused_email_id=24 bounced_mailbox_id=665
[AFTER ] bounced_mailbox_id == forward mailbox.id (665) : True
[Bounce] id=21 email='user_jt9j5qbfh3@mailbox.test'  == mailbox.email('user_jt9j5qbfh3@mailbox.test')? True
[RefusedEmail] id=24 full_report_path=refused-emails/full-23bb6ba9-c0da-4b22-a996-eb143cf8eb47.eml path=refused-emails/23bb6ba9-c0da-4b22-a996-eb143cf8eb47.eml
   [F9 LOCAL_FILE_UPLOAD] full_report file exists=True @ /app/static/upload/refused-emails/full-23bb6ba9-c0da-4b22-a996-eb143cf8eb47.eml
   [F9 LOCAL_FILE_UPLOAD] orig file exists=True @ /app/static/upload/refused-emails/23bb6ba9-c0da-4b22-a996-eb143cf8eb47.eml
[Notification] count for forward user = 1   (forward phase ALWAYS creates a user Notification: a delivery-failure notice,
               or a 'disabled due to multiple bounces' notice when should_disable() is True)
   [Notification.title] = 'Email from fwd-victim@example.com to comfit_clomps867@sl.local cannot be delivered to user_jt9j5qbfh3@mailbox.test'

##############################################################################
# REPLY-PHASE BOUNCE  (EmailLog.is_reply = True)
##############################################################################
[SETUP] user=507 alias=doctor_refits114@sl.local mailbox.email=user_ginz4xrzun@mailbox.test contact.website_email=reply-victim@example.com email_log.id=414 is_reply=True
[BEFORE] email_log.bounced=False refused_email_id=None bounced_mailbox_id=None
[VERP rcpt] bounce_reply address = sl.lmysyibuge2cyibsgm4deojrgfoq.hiha3it6x34ro@sl.local
[msg content-type] = multipart/report  mail_from = <>
[RESULT] handle() -> '250 SL E212 Bounce Reply phase handled'
[ASSERT] res == status.E212 : True   (E212 = '250 SL E212 Bounce Reply phase handled' )
[AFTER ] email_log.bounced=True refused_email_id=25 bounced_mailbox_id=666
[AFTER ] refused_email_id set? True   bounced_mailbox_id == reply mailbox.id(666)? True
[Bounce] id=22 email='reply-victim@example.com'  == contact.website_email('reply-victim@example.com')? True
[RefusedEmail] id=25 full_report_path=refused-emails/full-ddbb18f6-1f0c-48e6-b538-daa60ce89583.eml path=refused-emails/ddbb18f6-1f0c-48e6-b538-daa60ce89583.eml
   [F9 LOCAL_FILE_UPLOAD] full_report file exists=True @ /app/static/upload/refused-emails/full-ddbb18f6-1f0c-48e6-b538-daa60ce89583.eml
   [F9 LOCAL_FILE_UPLOAD] orig file exists=True @ /app/static/upload/refused-emails/ddbb18f6-1f0c-48e6-b538-daa60ce89583.eml
[Notification] count for reply user = 1   (reply phase ALWAYS creates a user Notification)
   [Notification.title] = 'Email cannot be sent to reply-victim@example.com from your alias doctor_refits114@sl.local'

##############################################################################
# DIRECTION-DEPENDENT SUMMARY (observed)
##############################################################################
  FORWARD (is_reply=False): status='250 SL E211 Bounce Forward phase handled'
     Bounce.email = mailbox.email ('user_jt9j5qbfh3@mailbox.test')
     Notification(s)=1  title='Email from fwd-victim@example.com to comfit_clomps867@sl.local cannot be delivered to user_jt9j5qbfh3@mailbox.test'
  REPLY   (is_reply=True) : status='250 SL E212 Bounce Reply phase handled'
     Bounce.email = contact.website_email ('reply-victim@example.com')
     Notification(s)=1  title='Email cannot be sent to reply-victim@example.com from your alias doctor_refits114@sl.local'

##############################################################################
# CLEANUP (explicit deletion cascade — NOT rollback)
##############################################################################
  removed local file: /app/static/upload/refused-emails/full-23bb6ba9-c0da-4b22-a996-eb143cf8eb47.eml
  removed local file: /app/static/upload/refused-emails/23bb6ba9-c0da-4b22-a996-eb143cf8eb47.eml
  removed local file: /app/static/upload/refused-emails/full-ddbb18f6-1f0c-48e6-b538-daa60ce89583.eml
  removed local file: /app/static/upload/refused-emails/ddbb18f6-1f0c-48e6-b538-daa60ce89583.eml
FINAL counts:    {'Bounce': 9, 'RefusedEmail': 0, 'Notification': 0, 'EmailLog': 1, 'User': 26, 'Job': 0}
BASELINE counts: {'Bounce': 9, 'RefusedEmail': 0, 'Notification': 0, 'EmailLog': 1, 'User': 26, 'Job': 0}
NET-ZERO (counts identical)? True
FINAL refused-emails file count: 14  BASELINE: 14  NET-ZERO? True
```

**Two-run stability (F5).** RUN 2 (same command) shows the identical behavior —
`E211`/`E212`, `Bounce` keyed on `mailbox.email`/`contact.website_email`, and
net-zero — differing only in run-specific identifiers (user/alias/email_log ids,
random alias names, and RefusedEmail uuids). Captured to files and diffed:

```bash
diff /tmp/blitzy_obs/q3_handler_run1.txt /tmp/blitzy_obs/q3_handler_run2.txt | grep -cE '^[<>]'
# -> 50   (all 50 differing lines are run-specific ids/aliases/uuids; behavior identical)
grep 'NET-ZERO' /tmp/blitzy_obs/q3_handler_run2.txt
# -> NET-ZERO (counts identical)? True
# -> FINAL refused-emails file count: 14  BASELINE: 14  NET-ZERO? True
```

### Evidence C — `email_log.auto_replied` reachability (F13), through canonical entries

This proves the corrected F13 semantics. **(a)** A modern HMAC-signed reply-VERP
carrying a **non-report** message is driven through the canonical async SMTP entry
`MailHandler.handle_DATA`; `is_bounce` is False, so `handle()`'s reply-VERP branch
raises `VERPReply`, which `handle_DATA` catches and converts to `E213`, leaving
`auto_replied=False`. **(b1)** The same entry with a **legacy** `mail_from =
bounce+{id}+@domain` drives the iCloud un-gated block, which calls `handle_bounce`
directly → the auto-reply branch sets `auto_replied=True`. **(b2)** Calling
`handle_bounce` directly on a reply `EmailLog` + non-report message exercises that
branch in isolation.

**Boundary disclosure (F9):** `NOT_SEND_EMAIL=True` suppresses the auto-reply
re-forward SMTP send. `auto_replied` is committed to the DB inside the handler and
re-read here with a fresh `SELECT` (after `Session.rollback()`) because
`handle_DATA` runs its own `create_light_app()` app-context.

Embedded script (`/tmp/blitzy_obs/obs_q3_autoreply.py`, complete):

```python
"""
obs_q3_autoreply.py — Q3 F13 reachability of email_log.auto_replied.

(a) Modern reply-VERP + NON-report message, driven through the CANONICAL async
    SMTP entry MailHandler.handle_DATA:
      is_bounce == False  -> reply-VERP 'else' branch -> raise VERPReply
      -> handle_DATA catches (email_handler.py:2308) -> returns status.E213
      -> email_log.auto_replied stays False.
    (Corrects the earlier claim that a non-report reply-VERP enters auto-reply.)

(b) email_log.auto_replied becomes True only when handle_bounce() runs on a reply
    EmailLog with a NON-report msg WITHOUT the is_bounce gate. In production this
    is the iCloud un-gated caller (email_handler.py:2101-2116) calling
    handle_bounce(envelope, email_log, msg) directly. Shown two canonical ways:
      (b1) handle_DATA with legacy mail_from = bounce+{id}+@domain (drives the iCloud block)
      (b2) direct handle_bounce() on a reply EmailLog + non-report msg (the branch itself)

Boundary substitute (F9): NOT_SEND_EMAIL=True -> the re-forward SMTP send is suppressed.
auto_replied is committed to the DB inside the handler; it is re-read here with a fresh
SELECT (Session.rollback first) because handle_DATA runs its own create_light_app context.
Cleanup is explicit deletion cascade; net-zero counts printed at the end.
"""
import os
import asyncio
from email.message import EmailMessage

from server import create_app

app = create_app()

from app import config
from app.db import Session
from app.models import User, Alias, Contact, EmailLog, Bounce, RefusedEmail, Notification, Job, VerpType
from app.email_utils import generate_verp_email
from app.email import status
import email_handler
from email_handler import MailHandler, is_bounce, handle_bounce
from tests.utils import create_new_user
from aiosmtpd.smtp import Envelope


def make_plain_msg(subject="hi"):
    m = EmailMessage()
    m["From"] = "external-sender@example.com"
    m["To"] = "someone@sl.local"
    m["Subject"] = subject
    m.set_content("this is a normal (non multipart/report) message body")
    return m


def counts():
    return {
        "Bounce": Session.query(Bounce).count(),
        "RefusedEmail": Session.query(RefusedEmail).count(),
        "Notification": Session.query(Notification).count(),
        "EmailLog": Session.query(EmailLog).count(),
        "User": Session.query(User).count(),
        "Job": Session.query(Job).count(),
    }


def auto_replied_of(email_log_id):
    """Fresh SELECT of the committed value (handler used its own session)."""
    Session.rollback()
    return Session.query(EmailLog.auto_replied).filter(EmailLog.id == email_log_id).scalar()


def make_reply_emaillog():
    u = create_new_user()
    a = Alias.create_new_random(u)
    Session.commit()
    c = Contact.create(user_id=u.id, alias_id=a.id,
                       website_email="c-%d@example.com" % u.id, reply_email="r-%d@sl.local" % u.id, commit=True)
    el = EmailLog.create(user_id=u.id, contact_id=c.id, alias_id=a.id,
                         mailbox_id=u.default_mailbox_id, is_reply=True, commit=True)
    return u, a, c, el


with app.app_context():
    print("=" * 78)
    print("BOUNDARY DISCLOSURE (F9): config.NOT_SEND_EMAIL =", config.NOT_SEND_EMAIL,
          "-> the auto-reply re-forward SMTP send is SUPPRESSED")
    print("=" * 78)
    base_counts = counts()
    base_max_job = Session.query(Job).order_by(Job.id.desc()).first()
    base_max_job_id = base_max_job.id if base_max_job else 0
    base_max_bounce = Session.query(Bounce).order_by(Bounce.id.desc()).first()
    base_max_bounce_id = base_max_bounce.id if base_max_bounce else 0
    print("BASELINE counts:", base_counts)
    print()

    created_user_ids = []

    # ================================================================
    # (a) Modern reply-VERP + NON-report -> E213, auto_replied stays False
    # ================================================================
    print("#" * 78)
    print("# (a) MODERN reply-VERP + non-report via canonical handle_DATA")
    print("#" * 78)
    ua, aa, ca, ea = make_reply_emaillog(); created_user_ids.append(ua.id)
    ea_id = ea.id
    print("[SETUP] reply EmailLog id=%d is_reply=True auto_replied(before)=%s" % (ea_id, ea.auto_replied))
    verp_a = generate_verp_email(VerpType.bounce_reply, ea_id)
    msg_a = make_plain_msg()
    env_a = Envelope()
    env_a.mail_from = "external-sender@example.com"     # NOT "<>"
    env_a.rcpt_tos = [verp_a]
    env_a.original_content = msg_a.as_bytes()
    print("[VERP rcpt] reply-VERP =", verp_a)
    print("[is_bounce(envelope,msg)] =", is_bounce(env_a, msg_a),
          " (needs mail_from=='<>' AND multipart/report; here both false)")
    res_a = asyncio.run(MailHandler().handle_DATA(None, None, env_a))
    print("[RESULT] handle_DATA ->", repr(res_a))
    print("[ASSERT] res == status.E213 :", res_a == status.E213, "  (E213 =", repr(status.E213), ")")
    a_auto = auto_replied_of(ea_id)
    print("[AFTER ] email_log.auto_replied =", a_auto, " (EXPECTED False — VERPReply raised, not auto-reply)")
    print()

    # ================================================================
    # (b1) iCloud un-gated path via canonical handle_DATA (legacy mail_from)
    # ================================================================
    print("#" * 78)
    print("# (b1) iCloud un-gated caller via canonical handle_DATA (mail_from = bounce+{id}+@domain)")
    print("#" * 78)
    ub, ab, cb, eb = make_reply_emaillog(); created_user_ids.append(ub.id)
    eb_id = eb.id
    print("[SETUP] reply EmailLog id=%d is_reply=True auto_replied(before)=%s" % (eb_id, eb.auto_replied))
    legacy_mail_from = "{}{}{}".format(config.BOUNCE_PREFIX, eb_id, config.BOUNCE_SUFFIX)
    ab_email = ab.email
    print("[legacy mail_from] =", legacy_mail_from, " (BOUNCE_PREFIX + id + BOUNCE_SUFFIX)")
    msg_b = make_plain_msg()
    env_b = Envelope()
    env_b.mail_from = legacy_mail_from       # NOT "<>"  -> iCloud block, un-gated handle_bounce
    env_b.rcpt_tos = [ab_email]
    env_b.original_content = msg_b.as_bytes()
    res_b = asyncio.run(MailHandler().handle_DATA(None, None, env_b))
    print("[RESULT] handle_DATA ->", repr(res_b))
    b_auto = auto_replied_of(eb_id)
    print("[AFTER ] email_log.auto_replied =", b_auto, " (EXPECTED True — un-gated handle_bounce auto-reply branch)")
    print()

    # ================================================================
    # (b2) direct handle_bounce() on reply EmailLog + non-report msg
    # ================================================================
    print("#" * 78)
    print("# (b2) direct handle_bounce() — the auto-reply branch in isolation")
    print("#" * 78)
    uc, ac, cc, ec = make_reply_emaillog(); created_user_ids.append(uc.id)
    ec_id = ec.id
    print("[SETUP] reply EmailLog id=%d is_reply=True auto_replied(before)=%s" % (ec_id, ec.auto_replied))
    msg_c = make_plain_msg()
    env_c = Envelope()
    env_c.mail_from = "external-sender@example.com"      # NOT "<>"
    env_c.rcpt_tos = [ac.email]
    res_c = handle_bounce(env_c, ec, msg_c)
    print("[RESULT] handle_bounce ->", repr(res_c), " (a forward smtp status; send suppressed by NOT_SEND_EMAIL)")
    c_auto = auto_replied_of(ec_id)
    print("[AFTER ] email_log.auto_replied =", c_auto, " (EXPECTED True)")
    print()

    # ================================================================
    # SUMMARY
    # ================================================================
    print("#" * 78)
    print("# F13 REACHABILITY SUMMARY (observed)")
    print("#" * 78)
    print("  (a) modern reply-VERP + non-report via handle_DATA -> %r ; auto_replied=%s" % (res_a, a_auto))
    print("  (b1) iCloud legacy mail_from via handle_DATA        -> %r ; auto_replied=%s" % (res_b, b_auto))
    print("  (b2) direct handle_bounce (auto-reply branch)       -> %r ; auto_replied=%s" % (res_c, c_auto))
    print()

    # ================================================================
    # CLEANUP
    # ================================================================
    print("#" * 78)
    print("# CLEANUP (explicit deletion cascade)")
    print("#" * 78)
    Session.rollback()
    for r in Session.query(RefusedEmail).filter(RefusedEmail.user_id.in_(created_user_ids)).all():
        for p in (r.full_report_path, r.path):
            if p:
                fp = os.path.join(config.UPLOAD_DIR, p)
                if os.path.exists(fp):
                    os.remove(fp); print("  removed local file:", fp)
    for uid in created_user_ids:
        User.delete(uid)
    Session.commit()
    Session.query(Bounce).filter(Bounce.id > base_max_bounce_id).delete(synchronize_session=False)
    Session.query(Job).filter(Job.id > base_max_job_id).delete(synchronize_session=False)
    Session.commit()
    final_counts = counts()
    print("FINAL counts:   ", final_counts)
    print("BASELINE counts:", base_counts)
    print("NET-ZERO (counts identical)?", final_counts == base_counts)
```

Exact command (RUN 1). The filter keeps the meaningful proof WARNINGs — the
`VERPReply` log at `email_handler.py:2309` and the `iCloud bounces` log at
`email_handler.py:2110` — and drops only routine noise:

```bash
docker exec sl_canon bash -lc '
export CONFIG=tests/test.env PYTHONPATH=/app DB_URI="postgresql://test:test@localhost:5432/test"
FILT_AUTO="SL - (DEBUG|INFO) -|load config file|>>> URL|WARNING: Use a temp|Upload files to local|>>> init logging|missing date header|add missing content-transfer|forward_email_to_mailbox"
/app/venv/bin/python /tmp/blitzy_obs/obs_q3_autoreply.py 2>&1 | grep -vE "$FILT_AUTO"'
```

Complete unedited output (RUN 1):

```text
==============================================================================
BOUNDARY DISCLOSURE (F9): config.NOT_SEND_EMAIL = True -> the auto-reply re-forward SMTP send is SUPPRESSED
==============================================================================
BASELINE counts: {'Bounce': 9, 'RefusedEmail': 0, 'Notification': 0, 'EmailLog': 1, 'User': 26, 'Job': 0}

##############################################################################
# (a) MODERN reply-VERP + non-report via canonical handle_DATA
##############################################################################
[SETUP] reply EmailLog id=453 is_reply=True auto_replied(before)=False
[VERP rcpt] reply-VERP = sl.lmysyibuguzsyibsgm4deojrgvoq.jyoltakwdsgga@sl.local
[is_bounce(envelope,msg)] = False  (needs mail_from=='<>' AND multipart/report; here both false)
2026-07-13 19:15:14,071 - SL - WARNING - 6692 - "/app/email_handler.py:2309" - handle_DATA() - 52d22211-08e0-4573-9e15-381230634942 - email handling fail with error:VERPReply cannot handle email sent to reply VERP, <Alias 863 kibosh_bummer266@sl.local> -> <Contact 179 c-518@example.com 863> (<EmailLog 453>, <User 518 Test User user_pcj63fwpe8@mailbox.test> mail_from:external-sender@example.com, rcpt_tos:['sl.lmysyibuguzsyibsgm4deojrgvoq.jyoltakwdsgga@sl.local'], header_from:external-sender@example.com, header_to:someone@sl.local
[RESULT] handle_DATA -> '250 SL E213 Unknown email ignored'
[ASSERT] res == status.E213 : True   (E213 = '250 SL E213 Unknown email ignored' )
[AFTER ] email_log.auto_replied = False  (EXPECTED False — VERPReply raised, not auto-reply)

##############################################################################
# (b1) iCloud un-gated caller via canonical handle_DATA (mail_from = bounce+{id}+@domain)
##############################################################################
[SETUP] reply EmailLog id=454 is_reply=True auto_replied(before)=False
[legacy mail_from] = bounce+454+@sl.local  (BOUNCE_PREFIX + id + BOUNCE_SUFFIX)
2026-07-13 19:15:14,369 - SL - WARNING - 6692 - "/app/email_handler.py:2110" - handle() - af0380a7-7c71-4631-9577-b24df3027264 - iCloud bounces <EmailLog 454> <Alias 865 oldish_polios041@sl.local>, saved to
[RESULT] handle_DATA -> '250 Message accepted for delivery'
[AFTER ] email_log.auto_replied = True  (EXPECTED True — un-gated handle_bounce auto-reply branch)

##############################################################################
# (b2) direct handle_bounce() — the auto-reply branch in isolation
##############################################################################
[SETUP] reply EmailLog id=456 is_reply=True auto_replied(before)=False
[RESULT] handle_bounce -> '250 Message accepted for delivery'  (a forward smtp status; send suppressed by NOT_SEND_EMAIL)
[AFTER ] email_log.auto_replied = True  (EXPECTED True)

##############################################################################
# F13 REACHABILITY SUMMARY (observed)
##############################################################################
  (a) modern reply-VERP + non-report via handle_DATA -> '250 SL E213 Unknown email ignored' ; auto_replied=False
  (b1) iCloud legacy mail_from via handle_DATA        -> '250 Message accepted for delivery' ; auto_replied=True
  (b2) direct handle_bounce (auto-reply branch)       -> '250 Message accepted for delivery' ; auto_replied=True

##############################################################################
# CLEANUP (explicit deletion cascade)
##############################################################################
FINAL counts:    {'Bounce': 9, 'RefusedEmail': 0, 'Notification': 0, 'EmailLog': 1, 'User': 26, 'Job': 0}
BASELINE counts: {'Bounce': 9, 'RefusedEmail': 0, 'Notification': 0, 'EmailLog': 1, 'User': 26, 'Job': 0}
NET-ZERO (counts identical)? True
```

**Two-run stability (F5).** RUN 2 (same command) produces the identical outcomes —
`(a)` `E213` with `auto_replied=False`; `(b1)` and `(b2)` `auto_replied=True`; and
net-zero — differing only in run-specific ids. Verified:

```bash
grep -E 'auto_replied=' /tmp/blitzy_obs/q3_autoreply_run2.txt | tail -3
# -> (a) modern reply-VERP + non-report via handle_DATA -> '250 SL E213 Unknown email ignored' ; auto_replied=False
# -> (b1) iCloud legacy mail_from via handle_DATA        -> '250 Message accepted for delivery' ; auto_replied=True
# -> (b2) direct handle_bounce (auto-reply branch)       -> '250 Message accepted for delivery' ; auto_replied=True
grep 'NET-ZERO' /tmp/blitzy_obs/q3_autoreply_run2.txt
# -> NET-ZERO (counts identical)? True
```

### Evidence D — edge branch: forward bounce that auto-disables the alias

This drives the canonical `email_handler.handle()` for the forward phase and
observes the `should_disable()` threshold transition. It seeds **12** pre-bounced
forward `EmailLog` rows (at which point `should_disable` is still `False`), then
sends **one live forward bounce** through `handle()`; the handler marks the 13th
log bounced → `nb_bounced_last_24h = 13 > 12` → the alias is flipped to
`enabled=False` and a "disabled due to multiple bounces" `Notification` is created.

**Boundary disclosure (F9):** `NOT_SEND_EMAIL=True` suppresses the alert email;
`LOCAL_FILE_UPLOAD=True` writes the `RefusedEmail` to a local file (removed at cleanup).

Embedded script (`/tmp/blitzy_obs/obs_q3_disable.py`, complete):

```python
"""
obs_q3_disable.py — Q3 edge branch: forward-phase bounce that tips an alias over the
auto-disable threshold, driven through the CANONICAL entry email_handler.handle().

should_disable() (app/email_utils.py) returns True when nb_bounced_last_24h > 12
(i.e. 13+ forward bounces in 24h). We seed 12 pre-bounced forward EmailLogs
(should_disable == False at 12), then drive ONE live forward bounce through
handle(); the handler sets bounced=True on the 13th log -> count 13 > 12 ->
change_alias_status(enabled=False) + a 'disabled due to multiple bounces' Notification.

Observed transition: alias.enabled  True -> (12 bounces: still True) -> (13th bounce: False)
Boundary substitute (F9): NOT_SEND_EMAIL=True suppresses the alert email; LOCAL_FILE_UPLOAD
writes RefusedEmail to a local file. Cleanup is explicit deletion cascade; net-zero at end.
"""
import os
import email

from server import create_app

app = create_app()

from app import config
from app.db import Session
from app.models import User, Alias, Contact, EmailLog, Bounce, RefusedEmail, Notification, Job, VerpType
from app.email_utils import generate_verp_email, should_disable
from app.email import status
import email_handler
from aiosmtpd.smtp import Envelope
from tests.utils import create_new_user

BOUNCE_EML = "/app/local_data/email_tests/bounce.eml"


def counts():
    return {
        "Bounce": Session.query(Bounce).count(),
        "RefusedEmail": Session.query(RefusedEmail).count(),
        "Notification": Session.query(Notification).count(),
        "EmailLog": Session.query(EmailLog).count(),
        "User": Session.query(User).count(),
        "Job": Session.query(Job).count(),
    }


with app.app_context():
    print("=" * 78)
    print("BOUNDARY DISCLOSURE (F9): NOT_SEND_EMAIL =", config.NOT_SEND_EMAIL,
          "  LOCAL_FILE_UPLOAD =", config.LOCAL_FILE_UPLOAD)
    print("config.ALIAS_AUTOMATIC_DISABLE =", config.ALIAS_AUTOMATIC_DISABLE,
          " (must be True for should_disable to fire)")
    print("=" * 78)
    base = counts()
    base_max_bounce = Session.query(Bounce).order_by(Bounce.id.desc()).first()
    base_max_bounce_id = base_max_bounce.id if base_max_bounce else 0
    base_max_job = Session.query(Job).order_by(Job.id.desc()).first()
    base_max_job_id = base_max_job.id if base_max_job else 0
    print("BASELINE counts:", base)
    print()

    u = create_new_user()
    mb = u.default_mailbox
    alias = Alias.create_new_random(u)
    Session.commit()
    contact = Contact.create(user_id=u.id, alias_id=alias.id,
                             website_email="disable-victim@example.com",
                             reply_email="rdis@sl.local", commit=True)
    uid = u.id
    print("[SETUP] user=%d alias=%s alias.enabled(BEFORE)=%s" % (uid, alias.email, alias.enabled))

    # seed 12 pre-bounced FORWARD email logs (within 24h)
    for _ in range(12):
        EmailLog.create(user_id=u.id, contact_id=contact.id, alias_id=alias.id,
                        mailbox_id=mb.id, is_reply=False, bounced=True, commit=True)
    Session.commit()
    dis12, reason12 = should_disable(alias)
    print("[INTERMEDIATE] after seeding 12 forward bounces: should_disable=%s reason=%r" % (dis12, reason12))
    print("[INTERMEDIATE] alias.enabled=%s (still enabled at 12 bounces; threshold is >12)" % alias.enabled)

    # the 13th, LIVE bounce driven through the canonical handler
    el13 = EmailLog.create(user_id=u.id, contact_id=contact.id, alias_id=alias.id,
                           mailbox_id=mb.id, is_reply=False, commit=True)
    verp = generate_verp_email(VerpType.bounce_forward, el13.id)
    env = Envelope()
    env.mail_from = "<>"
    env.rcpt_tos = [verp]
    with open(BOUNCE_EML, "rb") as f:
        msg = email.message_from_bytes(f.read())
    print("[LIVE BOUNCE] 13th forward bounce via handle(), rcpt=%s" % verp)
    res = email_handler.handle(env, msg)
    print("[RESULT] handle() ->", repr(res), " (E211 =", repr(status.E211), ")")

    Session.expire(alias)
    dis13, reason13 = should_disable(alias)
    print("[AFTER ] should_disable=%s reason=%r" % (dis13, reason13))
    print("[AFTER ] alias.enabled=%s  (EXPECTED False — auto-disabled by the 13th bounce)" % alias.enabled)
    disable_notifs = Session.query(Notification).filter(
        Notification.user_id == uid,
        Notification.title.like("%disabled due to multiple bounces%"),
    ).all()
    print("[AFTER ] 'disabled' Notification count =", len(disable_notifs))
    for nt in disable_notifs:
        print("   [Notification.title] =", repr(nt.title))
    print()
    print("TRANSITION (observed): alias.enabled  True -> (12 bounces) %s -> (13th bounce) %s"
          % (dis12 is False, alias.enabled))
    print()

    # ================================================================
    # CLEANUP
    # ================================================================
    print("#" * 78)
    print("# CLEANUP (explicit deletion cascade)")
    print("#" * 78)
    for r in Session.query(RefusedEmail).filter(RefusedEmail.user_id == uid).all():
        for p in (r.full_report_path, r.path):
            if p:
                fp = os.path.join(config.UPLOAD_DIR, p)
                if os.path.exists(fp):
                    os.remove(fp); print("  removed local file:", fp)
    User.delete(uid)
    Session.commit()
    Session.query(Bounce).filter(Bounce.id > base_max_bounce_id).delete(synchronize_session=False)
    Session.query(Job).filter(Job.id > base_max_job_id).delete(synchronize_session=False)
    Session.commit()
    final = counts()
    print("FINAL counts:   ", final)
    print("BASELINE counts:", base)
    print("NET-ZERO (counts identical)?", final == base)
```

Exact command (RUN 1). The filter keeps the proof WARNING — the `Disable alias …
+12 bounces in the last 24h` log at `email_handler.py:1502` — and drops only noise:

```bash
docker exec sl_canon bash -lc '
export CONFIG=tests/test.env PYTHONPATH=/app DB_URI="postgresql://test:test@localhost:5432/test"
FILT_DIS="SL - (DEBUG|INFO) -|load config file|>>> URL|WARNING: Use a temp|Upload files to local|>>> init logging|missing date header|add missing content-transfer|forward_email_to_mailbox|^[[:space:]]*id F0"
/app/venv/bin/python /tmp/blitzy_obs/obs_q3_disable.py 2>&1 | grep -vE "$FILT_DIS"'
```

Complete unedited output (RUN 1):

```text
==============================================================================
BOUNDARY DISCLOSURE (F9): NOT_SEND_EMAIL = True   LOCAL_FILE_UPLOAD = True
config.ALIAS_AUTOMATIC_DISABLE = True  (must be True for should_disable to fire)
==============================================================================
BASELINE counts: {'Bounce': 9, 'RefusedEmail': 0, 'Notification': 0, 'EmailLog': 1, 'User': 26, 'Job': 0}

[SETUP] user=524 alias=emerge_firmed807@sl.local alias.enabled(BEFORE)=True
[INTERMEDIATE] after seeding 12 forward bounces: should_disable=False reason=''
[INTERMEDIATE] alias.enabled=True (still enabled at 12 bounces; threshold is >12)
[LIVE BOUNCE] 13th forward bounce via handle(), rcpt=sl.lmycyibug42syibsgm4deojrgvoq.3uosct4m7pvmi@sl.local
2026-07-13 19:15:20,499 - SL - WARNING - 6722 - "/app/email_handler.py:1502" - handle_bounce_forward_phase() -  - Disable alias <Alias 875 emerge_firmed807@sl.local> because +12 bounces in the last 24h. [<Mailbox 683 user_6kedeh49uu@mailbox.test>] <User 524 Test User user_6kedeh49uu@mailbox.test>. Last contact <Contact 189 disable-victim@example.com 875>
[RESULT] handle() -> '250 SL E211 Bounce Forward phase handled'  (E211 = '250 SL E211 Bounce Forward phase handled' )
[AFTER ] should_disable=True reason='+12 bounces in the last 24h'
[AFTER ] alias.enabled=False  (EXPECTED False — auto-disabled by the 13th bounce)
[AFTER ] 'disabled' Notification count = 1
   [Notification.title] = 'emerge_firmed807@sl.local has been disabled due to multiple bounces'

TRANSITION (observed): alias.enabled  True -> (12 bounces) True -> (13th bounce) False

##############################################################################
# CLEANUP (explicit deletion cascade)
##############################################################################
  removed local file: /app/static/upload/refused-emails/full-07db654f-18e4-4752-aee7-cff2fdc7d612.eml
  removed local file: /app/static/upload/refused-emails/07db654f-18e4-4752-aee7-cff2fdc7d612.eml
FINAL counts:    {'Bounce': 9, 'RefusedEmail': 0, 'Notification': 0, 'EmailLog': 1, 'User': 26, 'Job': 0}
BASELINE counts: {'Bounce': 9, 'RefusedEmail': 0, 'Notification': 0, 'EmailLog': 1, 'User': 26, 'Job': 0}
NET-ZERO (counts identical)? True
```

**Two-run stability (F5).** RUN 2 (same command) shows the identical transition
(`enabled: True → (12 bounces) True → (13th) False`, reason `"+12 bounces in the
last 24h"`, one disable `Notification`) and net-zero, differing only in ids:

```bash
grep -E 'should_disable|alias.enabled=False|TRANSITION|NET-ZERO' /tmp/blitzy_obs/q3_disable_run2.txt
```

### Verbatim source — VERP generation and decoding

`generate_verp_email` — builds the bounce address (`app/email_utils.py`, L1438-1464):

```python
def generate_verp_email(
    verp_type: VerpType, object_id: int, sender_domain: Optional[str] = None
) -> str:
    """Generates an email address with the verp type, object_id and domain encoded in the address
    and signed with hmac to prevent tampering
    """
    # Encoded as a list to minimize size of email address
    # Time is in minutes granularity and start counting on 2022-01-01 to reduce bytes to represent time
    data = [
        verp_type.value,
        object_id or 0,
        int((time.time() - VERP_TIME_START) / 60),
    ]
    json_payload = json.dumps(data).encode("utf-8")
    # Signing without itsdangereous because it uses base64 that includes +/= symbols and lower and upper case letters.
    # We need to encode in base32
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

`get_verp_info_from_email` — decodes + HMAC-verifies + the future-bound check at
L1496 (`app/email_utils.py`, L1467-1498). Note there is **no** lower time bound:

```python
def get_verp_info_from_email(email: str) -> Optional[Tuple[VerpType, int]]:
    """This method processes the email address, checks if it's a signed verp email generated by us to receive bounces
    and extracts the type of verp email and associated email log id/transactional email id stored as object_id
    """
    idx = email.find("@")
    if idx == -1:
        return None
    username = email[:idx]
    fields = username.split(".")
    if len(fields) != 3 or fields[0] != config.VERP_PREFIX:
        return None
    try:
        padding = (8 - (len(fields[1]) % 8)) % 8
        payload = base64.b32decode(fields[1].encode("utf-8").upper() + (b"=" * padding))
        padding = (8 - (len(fields[2]) % 8)) % 8
        signature = base64.b32decode(
            fields[2].encode("utf-8").upper() + (b"=" * padding)
        )
    except binascii.Error:
        return None
    expected_signature = hmac.new(
        config.VERP_EMAIL_SECRET.encode("utf-8"), payload, VERP_HMAC_ALGO
    ).digest()[:8]
    if expected_signature != signature:
        return None
    data = json.loads(payload)
    # verp type, object_id, time
    if len(data) != 3:
        return None
    if data[2] > (time.time() + config.VERP_MESSAGE_LIFETIME - VERP_TIME_START) / 60:
        return None
    return VerpType(data[0]), data[1]
```

`is_bounce` — a real bounce requires empty `MAIL FROM` **and** `multipart/report`
(`email_handler.py`, L1813-1818):

```python
def is_bounce(envelope: Envelope, msg: Message):
    """Detect whether an email is a Delivery Status Notification"""
    return (
        envelope.mail_from == "<>"
        and msg.get_content_type().lower() == "multipart/report"
    )
```

`handle_bounce` — direction dispatch on `email_log.is_reply`, the auto-reply branch
(sets `auto_replied=True` at L1887), and the `E211`/`E212` returns
(`email_handler.py`, L1851-1914):

```python
def handle_bounce(envelope, email_log: EmailLog, msg: Message) -> str:
    """
    Return SMTP status, e.g. "500 Error"
    """

    if not email_log:
        LOG.w("No such email log")
        return status.E512

    contact: Contact = email_log.contact
    alias = contact.alias
    LOG.d(
        "handle bounce for %s, phase=%s, contact=%s, alias=%s",
        email_log,
        email_log.get_phase(),
        contact,
        alias,
    )
    if not email_log.user.is_active():
        LOG.d(f"User {email_log.user} is not active")
        return status.E510

    if email_log.is_reply:
        content_type = msg.get_content_type().lower()

        if content_type != "multipart/report" or envelope.mail_from != "<>":
            # forward the email again to the alias
            LOG.i(
                "Handle auto reply %s %s",
                content_type,
                envelope.mail_from,
            )

            contact: Contact = email_log.contact
            alias = contact.alias

            email_log.auto_replied = True
            Session.commit()

            # replace the BOUNCE_EMAIL by alias in To field
            add_or_replace_header(msg, "To", alias.email)
            envelope.rcpt_tos = [alias.email]

            # same as handle()
            # result of all deliveries
            # each element is a couple of whether the delivery is successful and the smtp status
            res: [(bool, str)] = []

            for is_delivered, smtp_status in handle_forward(envelope, msg, alias.email):
                res.append((is_delivered, smtp_status))

            for is_success, smtp_status in res:
                # Consider all deliveries successful if 1 delivery is successful
                if is_success:
                    return smtp_status

            # Failed delivery for all, return the first failure
            return res[0][1]

        handle_bounce_reply_phase(envelope, msg, email_log)
        return status.E212
    else:  # forward phase
        handle_bounce_forward_phase(msg, email_log)
        return status.E211
```

`handle()` reply-VERP dispatch — a non-bounce, non-OOO reply-VERP raises `VERPReply`
(the `else` at L2095), which `handle_DATA` maps to `E213`
(`email_handler.py`, L2079-2095):

```python
        and rcpt_tos[0].startswith(f"{BOUNCE_PREFIX_FOR_REPLY_PHASE}+")
        or (verp_info and verp_info[0] == VerpType.bounce_reply)
    ):
        email_log_id = (verp_info and verp_info[1]) or parse_id_from_bounce(rcpt_tos[0])
        email_log = EmailLog.get(email_log_id)

        if not email_log:
            LOG.w("No such email log")
            return status.E512

        # bounce by contact
        if is_bounce(envelope, msg):
            return handle_bounce(envelope, email_log, msg)
        elif is_automatic_out_of_office(msg):
            handle_out_of_office_reply_phase(email_log, envelope, msg, rcpt_tos)
        else:
            raise VERPReply(
```

`handle()` iCloud un-gated caller — calls `handle_bounce` directly (no `is_bounce`
gate), the only production path that can set `auto_replied=True` for a non-report
reply (`email_handler.py`, L2098-2116):

```python
            )

    # iCloud returns the bounce with mail_from=bounce+{email_log_id}+@simplelogin.co, rcpt_to=alias
    verp_info = get_verp_info_from_email(mail_from[0])
    if (
        len(rcpt_tos) == 1
        and mail_from.startswith(BOUNCE_PREFIX)
        and mail_from.endswith(BOUNCE_SUFFIX)
    ) or (verp_info and verp_info[0] == VerpType.bounce_forward):
        email_log_id = (verp_info and verp_info[1]) or parse_id_from_bounce(mail_from)
        email_log = EmailLog.get(email_log_id)
        alias = Alias.get_by(email=rcpt_tos[0])
        LOG.w(
            "iCloud bounces %s %s, saved to%s",
            email_log,
            alias,
            save_email_for_debugging(msg, file_name_prefix="icloud_bounce_"),
        )
        return handle_bounce(envelope, email_log, msg)
```

### Q3 file:line reference table

| Item | Location | What it is |
|------|----------|-----------|
| `generate_verp_email` (address builder) | `app/email_utils.py:1438-1464` | Emits `{sl}.{base32(payload)}.{base32(hmac)}@{domain}` |
| `payload = [verp_type, object_id, minutes_since_2022]` | `app/email_utils.py:1446-1450` | The signed payload |
| `VERP_TIME_START = 1640995200` | `app/email_utils.py:68` | Epoch base (2022-01-01) for the minute counter |
| `VERP_HMAC_ALGO = "sha3-224"` | `app/email_utils.py:69` | HMAC algorithm; signature = first 8 bytes |
| `get_verp_info_from_email` (decoder) | `app/email_utils.py:1467-1498` | Splits, HMAC-verifies, returns `(VerpType, object_id)` |
| HMAC verification (`!= -> None`) | `app/email_utils.py:1490` | Tamper rejection |
| Future-bound check (**not** expiry) | `app/email_utils.py:1496` | Rejects only tokens dated `> now + 5 days`; no lower bound |
| `parse_id_from_bounce` (legacy) | `app/email_utils.py:1258` | Parses id from `bounce+{id}+@…` |
| `VERP_PREFIX = "sl"` | `app/config.py:500` | Modern VERP prefix |
| `VERP_MESSAGE_LIFETIME = 5*86400` | `app/config.py:499` | 5-day future bound (`432000 s`) |
| `BOUNCE_PREFIX`/`BOUNCE_SUFFIX` | `app/config.py:100-101` | Legacy forward: `bounce+` / `+@{EMAIL_DOMAIN}` |
| `BOUNCE_PREFIX_FOR_REPLY_PHASE = "bounce_reply"` | `app/config.py:108-110` | Legacy reply prefix (no trailing `+`) |
| `VerpType` (bounce_forward=0, bounce_reply=1, transactional=2) | `app/models.py:247-251` | Direction encoded in the address |
| `handle()` decode + VERP dispatch | `email_handler.py:2035,2058-2095` | Recovers `EmailLog`, branches by `VerpType` |
| `is_bounce` (`<>` + `multipart/report`) | `email_handler.py:1813-1818` | Distinguishes a real bounce |
| `handle_bounce` (branch on `is_reply`) | `email_handler.py:1851-1914` | Forward→`E211`, reply→`E212`; auto-reply branch |
| `email_log.auto_replied = True` | `email_handler.py:1887` | Set only in the un-gated auto-reply branch |
| `handle_bounce_forward_phase` | `email_handler.py:1432` | `Bounce(mailbox.email)`, RefusedEmail, `bounced`, Notification, `should_disable` |
| `handle_bounce_reply_phase` | `email_handler.py:1595` | `Bounce(contact.website_email)`, RefusedEmail, `bounced`, Notification |
| `VERPReply` raised (reply else) | `email_handler.py:2095` | Modern non-report reply-VERP path |
| iCloud un-gated caller | `email_handler.py:2098-2116` | Calls `handle_bounce` directly |
| `handle_DATA` catches VERP* → `E213` | `email_handler.py:2308-2318` | `250 SL E213 Unknown email ignored` |
| Status strings `E211`/`E212`/`E213` | `app/email/status.py:19-21` | Exact SMTP replies |
| `EmailLog.is_reply` / `bounced` / `auto_replied` / `refused_email_id` / `bounced_mailbox_id` | `app/models.py:2075,2082,2085,2097,2109` | Observed state fields |

### Q3 coverage confirmation

- **Exact format of the bounce/return-path/VERP address** →
  `{VERP_PREFIX}.{base32(payload)}.{base32(hmac_signature)}@{domain}` lowercased,
  with `VERP_PREFIX="sl"`; observed for **both directions**
  (`sl.…@contact-domain.com` forward, `sl.…@sl.local` reply) **[OBSERVED]**.
- **(a) How the system identifies the original email** → `get_verp_info_from_email`
  splits the address, HMAC-verifies it, and returns `(VerpType, object_id)` where
  `object_id = EmailLog.id`; round-trip decode observed for both directions, and a
  tampered signature decodes to `None` **[OBSERVED]**.
- **(b) State changes recorded** → `email_log.bounced=True`, `refused_email_id` set,
  `bounced_mailbox_id` set; a `Bounce` row created; a `RefusedEmail` stored (local
  file under `UPLOAD_DIR`); a user `Notification` created — all observed before→after
  **[OBSERVED]**.
- **(c) Direction-dependent behavior (forward vs reply)** → forward returns `E211`
  with `Bounce` keyed on `mailbox.email`; reply returns `E212` with `Bounce` keyed on
  `contact.website_email`; direction decided by `EmailLog.is_reply` **[OBSERVED]**.
- **Auto-reply exception** → a modern non-report reply-VERP raises `VERPReply` →
  `E213`, `auto_replied` stays `False`; `auto_replied=True` only via the iCloud
  un-gated caller path **[OBSERVED]**.
- **Edge — auto-disable after repeated bounces** → the 13th forward bounce
  (`>12` in 24h) flips `alias.enabled` `True→False` and creates a "disabled due to
  multiple bounces" `Notification` **[OBSERVED]**.
- **VERP hardening (from research)** → tamper-resistance is the sha3-224 HMAC, and
  `VERP_MESSAGE_LIFETIME` bounds how far in the **future** a token may be dated; it
  is **not** an age-based expiry — old signed tokens still decode **[OBSERVED]**.

---

## Methodology note — reproducibility invariants and point-in-time baselines

This investigation runs real code against a live PostgreSQL database inside the
canonical container. Two properties of that setup shape how the evidence above
should be read, and what "reproducibility" means here.

**1. What is reproducible (the invariants).** Every claim rests on one or more of
three invariants that hold across runs:

- **Behavioral outcome** — the observable behavior is stable: the counter
  progression `tries 0 -> 1 -> 2 -> 3` and lockout (Q1); the transitions
  `ready(0) -> taken(1) -> done(2)` and the failing-job outcome (stays
  `taken(1)`, `attempts` incremented, process exits non-zero) (Q2); the VERP
  address grammar and the `E211` / `E212` / `E213` status replies (Q3). This is
  the behavior each question asks about, and it repeats identically.
- **Net-zero database effect** — each DB-touching script prints its own row
  counts *before* and *after* and removes exactly the rows it created via an
  explicit deletion cascade, so `before == after` within the same run. Net-zero
  is asserted as this within-run equality, which holds against **any** baseline.
- **Two-run stability** — every scenario is executed twice; both runs are
  reported in full and agree on all of the above.

**2. What is NOT byte-reproducible (and why that is expected).** The absolute row
counts printed as a baseline (for example `User=26`) are a **point-in-time
snapshot** of the container's Postgres, which is ephemeral, is not tracked by git,
and accumulates rows across many investigation runs (a freshly migrated database
has essentially no application rows). Those absolute numbers are therefore not the
thing being demonstrated and are not expected to match between runs or sessions.
Likewise, run-specific identifiers — user ids, alias local-parts, the base32
payload inside a freshly generated VERP address, and UUID file names — are
**generated per run by construction**, so the two-run outputs differ on exactly
those tokens while agreeing on behavior (each two-run block states this
explicitly, e.g. "all differing lines are run-specific ids/aliases/uuids;
behavior identical").

**3. The strongest case — `obs_q3_verp.py`.** This script (Q3 Evidence A) is
DB-independent, and its behavioral output — the address grammar, the decode
round-trips, the tamper→None result, the lifetime accept/reject boundary, and the
printed constants — is fully deterministic and identical on every run. The one
run-varying element is the address generated "now" in step (1) (and the tampered
variant derived from it): because a VERP address encodes the generation time to
minute granularity, those bytes are identical between two runs in the same
clock-minute (as the embedded RUN 1 / RUN 2 pair was — giving an empty diff) and
shift only in the time-encoded portion for runs in different minutes. It is the
strongest reproducibility evidence for the CRITICAL VERP-format / semantics answer:
the format and the accept/reject behavior repeat identically regardless of when the
script runs.

**4. Scope of the guarantee.** None of the above touches the source repository:
the container's database and its `static/upload` directory are runtime state, not
tracked files (see the repository-integrity section below). Every *source-read*
fact in this document — line numbers, constants, control flow — is independent of
database state and is stable regardless of accumulated rows.

---

## Repository integrity and cleanup

Per the read-only rule set, the only tracked change this task introduces is the
single deliverable document. This section shows, with exact commands and their
complete output, that (a) the temporary runtime artifacts the investigation
produced were removed, (b) the database was left with a net-zero effect, (c) all
temporary observation scripts and host build-scratch were deleted, and (d) the
git working tree contains exactly one modified file — this document — and nothing
else.

### 1. Temporary refused-email artifacts removed

Driving the real bounce path in Q3 causes `email_handler` to persist refused
messages under `static/upload/refused-emails/` (a `LOCAL_FILE_UPLOAD=1` boundary,
disclosed in Q3). Fourteen such files had accumulated from investigation runs.
This directory is runtime state and is **git-ignored** (`.gitignore` excludes
`static/upload/`), so its contents never appear in the repository diff; they are
removed here so the container is left clean.

```
$ docker exec sl_canon ls -la /app/static/upload/refused-emails/
total 64
drwxr-sr-x 2 root 1001 4096 Jul 13 19:15 .
drwxr-sr-x 1 root 1001 4096 Jul 13 16:59 ..
-rw-r--r-- 1 root 1001  105 Jul 13 16:59 27e6fd30-9cbe-4b89-9eec-96ddaa660f8b.eml
-rw-r--r-- 1 root 1001  105 Jul 13 16:59 444248bb-6061-4da8-9f67-eb0736cca147.eml
-rw-r--r-- 1 root 1001  105 Jul 13 17:10 79223903-ee68-433b-a7c0-b13cf017755c.eml
-rw-r--r-- 1 root 1001  105 Jul 13 17:03 cce1804b-b216-4677-b93f-deac2d5cf5a8.eml
-rw-r--r-- 1 root 1001  105 Jul 13 17:03 e2a9f84c-0177-41de-9a39-75b412c891b8.eml
-rw-r--r-- 1 root 1001  105 Jul 13 17:10 e7519247-bf37-4ead-a8ec-06565f12a40c.eml
-rw-r--r-- 1 root 1001  840 Jul 13 16:59 full-27e6fd30-9cbe-4b89-9eec-96ddaa660f8b.eml
-rw-r--r-- 1 root 1001  548 Jul 13 17:11 full-376a8803-9442-4a56-ab28-53aeb2bdd159.eml
-rw-r--r-- 1 root 1001  844 Jul 13 16:59 full-444248bb-6061-4da8-9f67-eb0736cca147.eml
-rw-r--r-- 1 root 1001  548 Jul 13 17:04 full-50df0cf0-1991-4d20-97b2-154d203e0a3e.eml
-rw-r--r-- 1 root 1001  844 Jul 13 17:10 full-79223903-ee68-433b-a7c0-b13cf017755c.eml
-rw-r--r-- 1 root 1001  844 Jul 13 17:03 full-cce1804b-b216-4677-b93f-deac2d5cf5a8.eml
-rw-r--r-- 1 root 1001  840 Jul 13 17:03 full-e2a9f84c-0177-41de-9a39-75b412c891b8.eml
-rw-r--r-- 1 root 1001  840 Jul 13 17:10 full-e7519247-bf37-4ead-a8ec-06565f12a40c.eml

$ docker exec sl_canon sh -c "ls -1A /app/static/upload/refused-emails/ | wc -l"
14
```

Removing them and confirming the directory is empty:

```
$ docker exec sl_canon sh -c "rm -f /app/static/upload/refused-emails/*.eml"
(rm exit=0)

$ docker exec sl_canon sh -c "ls -1A /app/static/upload/refused-emails/ | wc -l"   # AFTER: expect 0
0

$ docker exec sl_canon ls -la /app/static/upload/refused-emails/   # AFTER: empty dir
total 8
drwxr-sr-x 2 root 1001 4096 Jul 13 19:26 .
drwxr-sr-x 1 root 1001 4096 Jul 13 16:59 ..
```

### 2. Database net-zero effect

Deleting the disk artifacts touches no database rows. The counts below are the
point-in-time baseline of the ephemeral container database (see the methodology
note); the load-bearing claim is that they are **identical before and after** the
cleanup, i.e. net-zero. `RefusedEmail=0` also confirms the fourteen files above
were pure orphans with no surviving DB rows referencing them.

Before:

```
$ docker exec sl_canon bash -c 'cd /app && export CONFIG=tests/test.env PYTHONPATH=/app DB_URI="postgresql://test:test@localhost:5432/test" && /app/venv/bin/python /tmp/blitzy_obs/db_counts.py 2>/dev/null'
load config file /app/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/yggwpcgfdhacoxlflntx
Upload files to local dir
>>> init logging <<<
2026-07-13 19:26:35,947 - SL - DEBUG - 6926 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
User=26
Mailbox=67
MailboxActivation=27
Alias=35
Contact=11
EmailLog=1
Job=0
Bounce=9
RefusedEmail=0
Notification=0
```

After (same command; counts unchanged):

```
$ docker exec sl_canon bash -c 'cd /app && export CONFIG=tests/test.env PYTHONPATH=/app DB_URI="postgresql://test:test@localhost:5432/test" && /app/venv/bin/python /tmp/blitzy_obs/db_counts.py 2>/dev/null'
load config file /app/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/adxjajuvjqdyxejghswp
Upload files to local dir
>>> init logging <<<
2026-07-13 19:26:52,267 - SL - DEBUG - 6956 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
User=26
Mailbox=67
MailboxActivation=27
Alias=35
Contact=11
EmailLog=1
Job=0
Bounce=9
RefusedEmail=0
Notification=0
```

### 3. Temporary observation scripts and build-scratch removed

All observation scripts were authored on the host and copied into the container
at `/tmp/blitzy_obs/` for execution; host-side build directories were used to
assemble the byte-exact embeddings in this document. None of these are inside the
repository tree. They are removed here so nothing temporary survives.

```
$ docker exec sl_canon sh -c "ls -1 /tmp/blitzy_obs/ | wc -l"   # container scratch scripts+outputs
26

$ ls -1d /tmp/q2src /tmp/q3src /tmp/q3build /tmp/blitzy_stage   # host scratch dirs
/tmp/blitzy_stage
/tmp/q2src
/tmp/q3build
/tmp/q3src
```

Removing both the container scratch and the host build-scratch:

```
# Remove container-side observation scripts + captured outputs (where all scripts were authored & executed)
$ docker exec sl_canon sh -c "rm -rf /tmp/blitzy_obs"
(rm exit=0)
$ docker exec sl_canon sh -c "ls -la /tmp/blitzy_obs 2>&1 || echo REMOVED"
ls: cannot access '/tmp/blitzy_obs': No such file or directory
REMOVED

# Remove host-side build scratch (used to assemble byte-exact embeddings)
$ rm -rf /tmp/q2src /tmp/q3src /tmp/q3build /tmp/blitzy_stage
(rm exit=0)
$ ls -1d /tmp/q2src /tmp/q3src /tmp/q3build /tmp/blitzy_stage 2>&1 || echo ALL_REMOVED
ls: cannot access '/tmp/q2src': No such file or directory
ls: cannot access '/tmp/q3src': No such file or directory
ls: cannot access '/tmp/q3build': No such file or directory
ls: cannot access '/tmp/blitzy_stage': No such file or directory
ALL_REMOVED
```

### 4. No lingering investigation processes; services healthy

No investigation daemon (`job_runner`, `email_handler`, `gunicorn`) is left
running. A number of defunct (zombie, state `Z`) entries remain in the container
process table: these are inert PID-table remnants of earlier `docker exec` and
short-lived daemon runs whose parent is the container's PID 1 (`sleep infinity`,
which does not reap children). They hold no CPU, memory, file, socket, or
database resources and are cleared only when the container is torn down; they are
reported here honestly rather than hidden. The active investigation-daemon check
returns zero lines.

```
# (a) Active (non-defunct) investigation daemons? Expect NONE.
$ docker exec sl_canon sh -c "ps -eo pid,ppid,state,cmd | grep -E 'job_runner|email_handler|gunicorn|wsgi|obs_q' | grep -v grep | grep -v defunct"
(exit=1 ; grep exit 1 = zero active investigation-daemon lines)

# (b) Any defunct/zombie remnants from earlier daemon runs (state Z, reaped only on container teardown):
$ docker exec sl_canon sh -c "ps -eo pid,ppid,state,cmd | awk '\$3==\"Z\"'"
    525       1 Z [bash] <defunct>
    527       1 Z [bash] <defunct>
    528       1 Z [python] <defunct>
    995       1 Z [python] <defunct>
    996       1 Z [head] <defunct>
   1034       1 Z [pkill] <defunct>
   1197       1 Z [bash] <defunct>
   1688       1 Z [bash] <defunct>
   1690       1 Z [gunicorn] <defunct>
   1789       1 Z [pkill] <defunct>
   2099       1 Z [redis-server] <defunct>
```

### 5. Git working tree — exactly one file changed

The decisive integrity check: the working tree shows exactly one modified path,
this deliverable, and nothing else. The refused-email deletions from step 1 do
not appear because `static/upload/` is git-ignored.

```
$ git rev-parse --abbrev-ref HEAD   # current branch
blitzy-a5e337a6-db0a-49aa-932f-e2facb1909f6

$ git status --porcelain            # ONLY the deliverable should appear
 M blitzy/documentation/app_2cd6ee777f8c.md

$ git diff --name-status            # nature of the change
M	blitzy/documentation/app_2cd6ee777f8c.md

$ git diff --stat                   # scope of the change
 blitzy/documentation/app_2cd6ee777f8c.md | 3311 +++++++++++++++++++++++++-----
 1 file changed, 2747 insertions(+), 564 deletions(-)
```

The load-bearing invariant is the **scope** shown by `git status --porcelain` and
`git diff --name-status`: a single modified file,
`blitzy/documentation/app_2cd6ee777f8c.md`, with no other repository file created,
modified, or deleted. The `git diff --stat` insertion/deletion magnitude is a
point-in-time value captured during cleanup; it necessarily grows as this section
(the methodology note, this repository-integrity section, and the coverage pass
below) is appended — the document records its own finalization. The single-file
scope is re-confirmed at commit time. No source, configuration, dependency,
migration, schema, or test file is touched.

---

## Coverage pass — every named sub-part answered

Each question carries its own coverage confirmation in its section above. This
consolidated pass re-reads the three questions one final time, enumerates every
distinct thing and every named item each one asks for, and points to where in
this document that item is answered and with what evidence. References are by
section name (stable), and every item carries its evidence label.

### Q1 — repeated incorrect mailbox verification codes

- [x] **The limit that exists** — `MAX_ACTIVATION_TRIES = 3`
  (`app/mailbox_utils.py:43`). *(Q1 Direct answer + Verbatim enforcement;
  SOURCE-VERIFIED, and OBSERVED via the counter reaching the bound.)*
- [x] **How failed attempts are tracked (which counter / state field)** — the
  `tries` integer column on the latest `MailboxActivation` row
  (`app/models.py`), incremented and committed on each wrong code by
  `verify_mailbox_code`. *(Q1 Evidence A; OBSERVED.)*
- [x] **Its progression (before / intermediate / after)** — `tries` advances
  `0 -> 1 -> 2 -> 3` across successive wrong submissions. *(Q1 Evidence A,
  captured after each submission, two runs; OBSERVED.)*
- [x] **The condition that ultimately prevents further submissions** — on the
  attempt that reaches the limit, `clear_activation_codes_for_mailbox` deletes
  the activation row(s) (code invalidation) and `CannotVerifyError("Invalid
  activation code. Please request another code.")` is raised; a later submission
  then finds no activation and raises `MailboxError("Invalid code")`.
  *(Q1 Evidence A; OBSERVED.)*
- [x] **Named alternative — lockout / code invalidation / HTTP rejection** — it
  is **code invalidation** (deletion), not an HTTP 429: `/dashboard/mailbox_verify`
  carries no Flask-Limiter / `parallel_limiter` decorator, so the authenticated
  HTTP transcript returns `302` on every attempt, never `429`. The generic
  `check_bucket_limit` exists but is not applied to this route.
  *(Q1 Evidence B, real login + 5 authenticated requests, two runs; OBSERVED +
  SOURCE-VERIFIED.)*
- [x] **Edge branch — code expiry** — the separate 15-minute age guard invalidates
  an old code independently of `tries`. *(Q1 Evidence A expiry scenario;
  OBSERVED.)*
- [x] **Canonical entry point exercised** — the real HTTP route
  `/dashboard/mailbox_verify` (delegating to `mailbox_utils.verify_mailbox_code`),
  reached through a real `/auth/login` session. *(Q1 Evidence B; OBSERVED.)*

### Q2 — background-task lifecycle and error behavior

- [x] **Complete lifecycle from creation through completion** — `Job.create(...)`
  writes `state=ready(0)`; `get_jobs_to_run()` selects it; the main loop marks it
  `taken(1)`, sets `taken_at`, increments `attempts`, commits, calls
  `process_job`, then sets `state=done(2)`; `cleanup_old_jobs` later deletes it.
  *(Q2 Evidence A end-to-end for a valid recognized job, two runs; OBSERVED.)*
- [x] **A valid recognized job reaches final completion** — a job with a
  recognized name runs to `state=done(2)`. *(Q2 Evidence A; OBSERVED.)*
- [x] **Recovery / retry behavior on error** — there is **no** `try/except`
  around `process_job` in the main loop, so a raised exception propagates and the
  runner **process exits non-zero**; retry happens only after the process is
  restarted and the `taken_at` age exceeds `JOB_TAKEN_RETRY_WAIT_MINS = 30`,
  bounded by `JOB_MAX_ATTEMPTS = 5`. *(Q2 Evidence B — real `job_runner.py` run
  with captured traceback + exit code — and Evidence C retry-eligibility
  boundaries, two runs; grep evidence for the missing `try/except`. The full
  multi-restart retry-to-exhaustion cycle was not run end-to-end and is INFERRED
  from those observations; OBSERVED + SOURCE-VERIFIED + INFERRED.)*
- [x] **Observable state that reflects the failure** — the job **stays
  `taken(1)` with `attempts` incremented**; it is **not** moved to
  `error(3)`. `JobState.error(3)` is never assigned by `job_runner.py` (it
  appears only in `cleanup_old_jobs` and tests). *(Q2 Evidence B before/after
  rows; grep evidence; OBSERVED + SOURCE-VERIFIED.)*
- [x] **Retry-eligibility branches** — immediate `ready`, stale `taken` past the
  30-minute window, and exhausted `attempts >= 5` (ineligible). *(Q2 Evidence A
  eligibility scenarios; OBSERVED.)*
- [x] **End-of-life cleanup** — `cleanup_old_jobs` deletes old jobs in `done(2)`
  or `error(3)`, and old `taken(1)` jobs with `attempts >= JOB_MAX_ATTEMPTS`,
  shown with before/after rows (including a seeded `error(3)` row). *(Q2
  Evidence C; OBSERVED.)*
- [x] **Canonical entry point exercised** — the real `job_runner.py` drain loop
  (not a mock), plus the scheduled `cron.py` / `crontab.yml` invocation path for
  cleanup. *(Q2 Evidence B / D; OBSERVED + SOURCE-VERIFIED.)*

### Q3 — VERP bounce-address format and direction-dependent handling

- [x] **Exact format of the special bounce / return-path / VERP address** —
  lowercased `{VERP_PREFIX}.{base32(payload)}.{base32(signature)}@{domain}`,
  where `VERP_PREFIX = "sl"`, `payload = [verp_type, email_log_id,
  minutes_since_2022]`, and `signature` is the first 8 bytes of an
  HMAC-`sha3-224` over the payload (base32, `=` padding stripped). Built by
  `generate_verp_email`. *(Q3 Evidence A, byte-identical across two runs;
  OBSERVED + SOURCE-VERIFIED.)*
- [x] **Legacy address forms still recognized** — forward
  `BOUNCE_PREFIX + id + BOUNCE_SUFFIX` and reply
  `BOUNCE_PREFIX_FOR_REPLY_PHASE + "+" + id`, parsed by `parse_id_from_bounce`.
  *(Q3 Evidence A legacy round-trip; OBSERVED + SOURCE-VERIFIED.)*
- [x] **VERP validity semantics (CRITICAL — corrected)** — the lifetime check is
  a **future-directed clock-sanity guard, not an age-based expiry**: a token
  whose embedded timestamp is more than `VERP_MESSAGE_LIFETIME` in the **future**
  is rejected, but an **old** signed token still decodes successfully. Shown by
  decoding tokens back-dated 6 days and 100 days (both **ACCEPTED**), a token
  6 days in the future (**rejected -> None**), and one 4 days in the future
  (**ACCEPTED**). *(Q3 Evidence A, two runs; OBSERVED + SOURCE-VERIFIED at
  `app/email_utils.py` lifetime-check line.)*
- [x] **Tamper resistance** — mutating the signed address makes
  `get_verp_info_from_email` return `None` (HMAC verification fails). *(Q3
  Evidence A tamper scenario; OBSERVED.)*
- [x] **How the system identifies the original email** — `handle()` calls
  `get_verp_info_from_email(rcpt_tos[0])` (HMAC-verified) to recover the
  `EmailLog` id, then loads that `EmailLog`. *(Q3 Evidence B; OBSERVED +
  SOURCE-VERIFIED.)*
- [x] **State changes recorded** — a `Bounce` row is created, a `RefusedEmail`
  is stored, and `email_log.bounced = True` with `bounced_mailbox_id` set; the
  reply phase additionally sets `refused_email_id` and creates a user
  `Notification`. *(Q3 Evidence B before/after rows for both directions;
  OBSERVED.)*
- [x] **How handling differs by DIRECTION — forward vs reply** — direction is
  taken from `EmailLog.is_reply` in `handle_bounce`. **Forward phase**
  (`handle_bounce_forward_phase`): `Bounce` keyed on the **mailbox** email,
  may auto-disable the alias after repeated bounces, returns SMTP **`E211`**.
  **Reply phase** (`handle_bounce_reply_phase`): `Bounce` keyed on the
  **contact's website email**, creates a user `Notification`, returns SMTP
  **`E212`**. *(Q3 Evidence B, real `email_handler.handle()` driven for both
  directions with the captured status lines, two runs; OBSERVED +
  SOURCE-VERIFIED.)*
- [x] **Forward-phase auto-disable modifier** — after enough accumulated
  forward bounces, the next forward bounce flips `alias.enabled` from `True` to
  `False` and creates a `Notification`. *(Q3 Evidence D, seeded to the
  threshold; OBSERVED.)*
- [x] **Edge branch — the auto-reply exception (with reachability stated)** — a
  bounce-shaped message that is not `multipart/report` (or has a non-empty
  `MAIL FROM`) is treated as an auto-reply: the modern reply-VERP path raises
  `VERPReply` and returns `E213` **without** setting `auto_replied`, whereas the
  legacy/iCloud caller path sets `email_log.auto_replied = True` and re-forwards.
  The reachability constraint of each path is stated explicitly rather than
  assumed. *(Q3 Evidence C; OBSERVED + SOURCE-VERIFIED.)*
- [x] **Canonical entry point exercised** — the real `email_handler.handle()`
  SMTP path driven with the repository's own `local_data/email_tests/bounce.eml`
  DSN and an empty envelope sender, for both directions. *(Q3 Evidence B;
  OBSERVED.)*

### Confirmation

Every distinct thing and every named item in the three questions — including the
examples the questions offer as candidate answers (lockout / code invalidation /
HTTP rejection for Q1; recovery/retry behavior and observable failure state for
Q2; the bounce / return-path / VERP address and the forward-versus-reply
direction for Q3) — appears above and is answered from observed runtime evidence,
grounded in exact `file:line` references, and labeled OBSERVED, SOURCE-VERIFIED,
or INFERRED.
