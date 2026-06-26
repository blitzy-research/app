# SimpleLogin — Runtime Behavior Analysis (branch `app_2cd6ee777f8c` @ `2cd6ee77`)

This document answers three runtime-behavior questions about the **SimpleLogin** codebase. Each answer is grounded strictly in the source ("code-as-truth") and is backed by precise `file:line` citations, followed by an explicit *Rationale / Thinking* block that explains **why** the system behaves the way it does. All line numbers correspond to the repository at branch `app_2cd6ee777f8c`, HEAD commit `2cd6ee77`.

## Environment & Methodology

The analysis is grounded in **static source-code tracing**, which is the primary and sufficient method for these questions; every claim below was verified by reading the source at branch `app_2cd6ee777f8c`, commit `2cd6ee77`.

- **Runtime target:** Python 3.10 — the highest version pinned across the project: `python = "^3.10"` (`pyproject.toml:61`), `FROM python:3.10` (`Dockerfile:8`), and `python-version: '3.10'` in the CI workflow (`.github/workflows/main.yml:17`).
- **A full boot** would require **PostgreSQL + Redis + an SMTP path**, available via the user-supplied Docker image `andrewparkscaleai/coding-agent:simple-login__app__2cd6ee777f8c…` (from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`). This bullet describes the optional external runtime environment, not a code-as-truth behavioral claim; **runtime corroboration is optional** for these questions and leaves no artifacts behind, as the conclusions are derived from the code itself.
- **Persistence stack:** PostgreSQL accessed via SQLAlchemy (pinned to `1.3.24`, `poetry.lock:3053-3054`) using the `Session` pattern with explicit `Session.commit()`; time arithmetic via `arrow`; object storage via S3 (boto3) with a local-disk fallback gated on `config.LOCAL_FILE_UPLOAD` (`app/s3.py:28-58`).
- **Conventions to keep in mind while reading citations:** models inherit a `ModelMixin` (`app/models.py:62-91`) exposing `.get()`/`.filter()` plus `created_at`/`updated_at`, and `.create()`/`.delete()` (`app/models.py:116-139`); enums subclass `EnumE` (`app/models.py:167`); audit trails are written via `emit_user_audit_log(...)` (`app/user_audit_log_utils.py:35`); SMTP status codes are centralized in `app/email/status.py` (e.g. `E200` at `app/email/status.py:2`).

---

## Q1 — Mailbox verification code retry / lockout

### Question

What observable state changes occur when a user repeatedly submits an **incorrect** mailbox verification code? How are failed attempts tracked, what numeric limit applies, and what mechanism ultimately prevents further submission of that code?

### Answer (citation-backed)

**The data model.** Each mailbox has at most one *current* `MailboxActivation` row (`app/models.py:2828-2835`). Its columns are:

- `mailbox_id` — foreign key to `Mailbox.id` with `ondelete="cascade"`, `nullable=False`, `index=True` (`app/models.py:2831-2833`).
- `code` — `sa.String(32)`, `nullable=False`, `index=True` (`app/models.py:2834`).
- `tries` — `sa.Integer`, `default=0`, `nullable=False` (`app/models.py:2835`).
- `id`, `created_at`, and `updated_at` are supplied by `ModelMixin`.

**The numeric limit is 3.** It is defined by the module-level constant `MAX_ACTIVATION_TRIES = 3` (`app/mailbox_utils.py:43`).

**The verification function** is `verify_mailbox_code(user, mailbox_id, code)` (`app/mailbox_utils.py:166-220`). Its guards execute in the following strict order:

1. `Mailbox.get(mailbox_id)`; if the mailbox does not exist → `raise MailboxError("Invalid mailbox")` (`app/mailbox_utils.py:167-172`).
2. If `mailbox.verified` is already true → `clear_activation_codes_for_mailbox(mailbox)` then `return mailbox` (`app/mailbox_utils.py:173-178`).
3. If `mailbox.user_id != user.id` → `raise MailboxError("Invalid mailbox")` (`app/mailbox_utils.py:179-183`).
4. Load the **most recent** activation: `MailboxActivation.filter(MailboxActivation.mailbox_id == mailbox_id).order_by(MailboxActivation.created_at.desc()).first()` (`app/mailbox_utils.py:185-189`).
5. If there is no activation → `raise MailboxError("Invalid code")` (`app/mailbox_utils.py:190-194`).
6. **Lockout check:** `if activation.tries >= MAX_ACTIVATION_TRIES:` → `clear_activation_codes_for_mailbox(mailbox)` then `raise CannotVerifyError("Invalid activation code. Please request another code.")` (`app/mailbox_utils.py:195-198`).
7. **15-minute expiry:** `if activation.created_at < arrow.now().shift(minutes=-15):` → clear codes + the same `CannotVerifyError` (`app/mailbox_utils.py:199-204`).
8. **Wrong-code branch:** `if code != activation.code:` → `activation.tries = activation.tries + 1; Session.commit()` then `raise CannotVerifyError("Invalid activation code")` (`app/mailbox_utils.py:205-211`). **This is the observable state change on each wrong attempt: the `tries` column is incremented by 1 and the increment is committed.**
9. **Success:** `mailbox.verified = True`, `emit_user_audit_log(user, UserAuditLogAction.VerifyMailbox, …)`, `clear_activation_codes_for_mailbox(mailbox)`, `return mailbox` (`app/mailbox_utils.py:212-220`).

**Code clearing.** `clear_activation_codes_for_mailbox(mailbox)` (`app/mailbox_utils.py:159-163`) runs `Session.query(MailboxActivation).filter(MailboxActivation.mailbox_id == mailbox.id).delete()` followed by `Session.commit()` — i.e., it **deletes every activation row** for that mailbox.

**The `tries` progression (state-by-state — this is the crux):**

- Because the lockout check (`>= 3`) is evaluated *before* the increment, three wrong submissions move `tries` **0 → 1 → 2 → 3**, each raising `CannotVerifyError("Invalid activation code")` (`app/mailbox_utils.py:205-211`).
- The **4th** submission enters with `tries == 3`, so it takes the **lockout branch** (`activation.tries >= MAX_ACTIVATION_TRIES`), which **deletes the activation row** via `clear_activation_codes_for_mailbox()` and raises `CannotVerifyError("Invalid activation code. Please request another code.")` (`app/mailbox_utils.py:195-198`).
- Any **subsequent** submission then finds **no activation** (guard 5) and fails with `MailboxError("Invalid code")` (`app/mailbox_utils.py:190-194`) until the user requests a brand-new code.
- A subtle ordering consequence: even a *correct* code submitted on the 4th attempt is rejected, because the lockout check (`app/mailbox_utils.py:195`) precedes the code comparison (`app/mailbox_utils.py:205`).

**The mechanism that ultimately prevents further submission of a given code** is therefore the **lockout branch deleting the activation row** (`app/mailbox_utils.py:195-198`, calling `app/mailbox_utils.py:159-163`) — not a counter that merely blocks. Once the row is deleted, the code no longer exists to match against.

**Code generation.** `generate_activation_code(mailbox, use_digit_code=False)` (`app/mailbox_utils.py:223-239`) first clears any existing codes, then sets `code = secrets.token_urlsafe(16)` by default (`app/mailbox_utils.py:233`), or a six-digit `"{:06d}".format(random.randint(1, 999999))` when `use_digit_code` is set (`app/mailbox_utils.py:231`) — unless `config.MAILBOX_VERIFICATION_OVERRIDE_CODE` is configured (`app/mailbox_utils.py:228-229`). The new row is created with `tries=0` (`app/mailbox_utils.py:234-238`).

**The HTTP route.** `/mailbox_verify` is handled by `mailbox_verify()` (`app/dashboard/views/mailbox.py:120-135`):

- It is decorated **only** with `@dashboard_bp.route("/mailbox_verify")` (`app/dashboard/views/mailbox.py:120`) and `@login_required` (`app/dashboard/views/mailbox.py:121`) — **there is no `@parallel_limiter.lock` and no Flask-Limiter decorator.** With no `methods=` argument, the route is GET-only.
- It reads `mailbox_id` and `code` from `request.args`; if `code` is absent it falls back to the legacy `verify_with_signed_secret(mailbox_id)` (`app/dashboard/views/mailbox.py:125-127`).
- It calls `mailbox_utils.verify_mailbox_code(...)` inside a `try` and catches `except mailbox_utils.MailboxError` (`app/dashboard/views/mailbox.py:130`), flashing the error and redirecting. Because **`CannotVerifyError` is a subclass of `MailboxError`** (`app/mailbox_utils.py:38` extends the base at `app/mailbox_utils.py:28`), the wrong-code, lockout, and expiry errors are all caught here and surfaced as a flash message + redirect.
- **Contrast:** the `mailbox_route` handler does carry a limiter — `@parallel_limiter.lock(only_when=lambda: request.method == "POST")` (`app/dashboard/views/mailbox.py:38`). This makes the absence of a limiter on `/mailbox_verify` concrete: it is a deliberate asymmetry, not an oversight of the framework being unavailable.

**Legacy path.** `verify_with_signed_secret(request)` (`app/dashboard/views/mailbox.py:138-175`; the file is exactly 175 lines) handles older verification links. It builds an `itsdangerous.TimestampSigner(MAILBOX_SECRET)` (`app/dashboard/views/mailbox.py:139`) and calls `s.unsign(..., max_age=900)` — a 15-minute window (`app/dashboard/views/mailbox.py:142`) — then base64-decodes a `[mailbox_id, email]` JSON payload and, on success, sets `mailbox.verified = True`, emits an audit log, and commits. This path has no `tries` counter; its protection is the time-limited signature.

### Rationale / Thinking

- **Protection is purely application-level, not transport/HTTP-level.** There is no rate-limit decorator on `/mailbox_verify` (`app/dashboard/views/mailbox.py:120-121`). The defense rests on three pillars: (1) the **3-try ceiling** on `tries` (`app/mailbox_utils.py:43`, enforced at `:195`), (2) the **15-minute expiry** (`app/mailbox_utils.py:199`), and (3) a **high-entropy, short-lived code** — `secrets.token_urlsafe(16)` is ~128 bits of entropy (`app/mailbox_utils.py:233`), or a six-digit code for the digit variant (`app/mailbox_utils.py:231`). With at most three guesses against a 128-bit secret (or even a six-digit space), brute force is infeasible within the code's lifetime, so an HTTP throttle is not strictly required for confidentiality of the code.
- **Why deletion rather than a permanent "locked" flag.** Clearing the row (`app/mailbox_utils.py:159-163`) keeps the schema simple — at most one live code per mailbox — and forces the user onto the "request another code" path, which regenerates a fresh high-entropy secret with `tries` reset to 0 (`app/mailbox_utils.py:223-239`). A burned code can therefore never be retried; there is no stale locked state to reason about or to reset.
- **Why "most recent activation" ordering matters.** `generate_activation_code()` deletes prior rows before inserting (`app/mailbox_utils.py:226`), so normally only one activation exists. The defensive `order_by(created_at.desc())` (`app/mailbox_utils.py:187`) makes the logic robust even if multiple rows were ever to coexist — the newest code wins.
- **Strict budget.** Because the lockout check precedes the code comparison (`app/mailbox_utils.py:195` before `:205`), the 3-try budget is strict: a correct guess on the fourth attempt is still rejected and the code is destroyed. This is a conservative, fail-closed posture.

---

## Q2 — Background task (Job) lifecycle and error/retry

### Question

For a scheduled background task (a `Job` row) later picked up for execution, trace the full lifecycle from creation through completion, and determine the recovery/retry behavior when the task errors mid-execution, plus the observable state that reflects failure.

### Answer (citation-backed)

**The model and enum.**

- `Job` (`app/models.py:2683-2707`, `__tablename__ = "job"` at `:2686`) has: `name` `sa.String(128)` not-null (`:2688`); `payload` `sa.JSON` (`:2689`); `taken` `sa.Boolean` `default=False` not-null (`:2692`); `run_at` `ArrowType` (`:2693`); `state` `sa.Integer` not-null with `server_default=str(JobState.ready.value)`, `default=JobState.ready.value`, `index=True` (`:2694-2700`); `attempts` `sa.Integer` not-null `server_default="0"`, `default=0` (`:2701`); and `taken_at` `ArrowType`, nullable (`:2702`). A composite index `ix_state_run_at_taken_at` over `(state, run_at, taken_at)` backs the selection query (`:2704`).
- `JobState(EnumE)` (`app/models.py:253-257`) defines `ready = 0`, `taken = 1`, `done = 2`, `error = 3`.

**Creation.** A new job is simply a `Job` row whose `state` defaults to `JobState.ready` (0), `attempts` defaults to 0, and `taken` defaults to `False` (per the column defaults at `app/models.py:2692-2701`), with a `name`, an optional `payload`, and an optional `run_at` schedule time.

**The runner loop** lives at `job_runner.py:329-347`, guarded by `if __name__ == "__main__":` (`:329`), looping `while True:` (`:330`), and wrapping each iteration in `with create_light_app().app_context():` (`:332`). For each job returned by `get_jobs_to_run()` (`:333`), the following happen in this exact order:

```python
job.taken = True                      # job_runner.py:337
job.taken_at = arrow.now()            # job_runner.py:338
job.state = JobState.taken.value      # job_runner.py:339
job.attempts += 1                     # job_runner.py:340  <- incremented BEFORE execution
Session.commit()                      # job_runner.py:341  <- committed BEFORE dispatch
process_job(job)                      # job_runner.py:342  <- the actual work
job.state = JobState.done.value       # job_runner.py:344  <- only reached on success
Session.commit()                      # job_runner.py:345
```

After the `for` loop, `time.sleep(10)` (`job_runner.py:347`) makes the runner wake roughly every 10 seconds.

**The dispatcher** `process_job(job)` (`job_runner.py:188-304`) is a large `if/elif` on `job.name` (matching config constants such as `config.JOB_ONBOARDING_1`, `JOB_ONBOARDING_2`, `JOB_ONBOARDING_4`, and so on) that reads `job.payload` and calls the appropriate handler.

**Selection.** `get_jobs_to_run()` (`job_runner.py:307-326`) computes `taken_at_earliest = arrow.now().shift(minutes=-config.JOB_TAKEN_RETRY_WAIT_MINS)` (`:311`) and `run_at_earliest = arrow.now().shift(minutes=+10)` (`:312`), then returns `Job` rows matching:

- `state == ready` **OR** (`state == taken` **AND** `taken_at < taken_at_earliest` **AND** `attempts < JOB_MAX_ATTEMPTS`) — `job_runner.py:315-322`, **AND**
- (`run_at IS NULL` **OR** `run_at <= run_at_earliest`) — `job_runner.py:323`.

The function's own header comment restates these exact conditions (`job_runner.py:308-310`).

**Constants** (`app/config.py`): `JOB_MAX_ATTEMPTS = 5` (`:564`) and `JOB_TAKEN_RETRY_WAIT_MINS = 30` (`:565`).

**Error / retry semantics (the core of Q2).**

- There is **no `try`/`except` anywhere in `job_runner.py`** (a file-wide search for `try:`/`except` returns zero matches).
- Consequently, if `process_job()` (`job_runner.py:342`) raises, the `job.state = JobState.done.value` assignment at `job_runner.py:344` is **never reached**. The exception propagates out of the loop and **terminates the runner process**; an external supervisor (process manager) is responsible for restarting it.
- The job is left in `state = taken` with `attempts` **already incremented** — because the increment and commit at `job_runner.py:340-341` happened *before* dispatch — and `taken_at` set to the pickup time.
- That job is **re-selected only after 30 minutes** (`taken_at < now − JOB_TAKEN_RETRY_WAIT_MINS`, `job_runner.py:319`) **and only while `attempts < JOB_MAX_ATTEMPTS` (5)** (`job_runner.py:320`). After five attempts it is no longer selected by `get_jobs_to_run()`.

**The `JobState.error` quirk.** `JobState.error` (value 3, `app/models.py:257`) is **never assigned by the runner**. Outside its enum definition, it is referenced only in `tasks/cleanup_old_jobs.py:15` (production) and in `tests/tasks/test_cleanup_old_jobs.py` (tests). `job_runner.py` only ever sets `JobState.taken` (`:339`) and `JobState.done` (`:344`).

**Cleanup.** `cleanup_old_jobs(oldest_allowed)` (`tasks/cleanup_old_jobs.py:10-24`) deletes `Job` rows where `state == done` (`:14`) **OR** `state == error` (`:15`) **OR** (`state == taken` **AND** `attempts >= JOB_MAX_ATTEMPTS`) (`:16-19`), **AND** `updated_at < oldest_allowed` (`:21`), then commits (`:23`). This confirms the terminal-state contract: stale, exhausted `taken` jobs are reaped by their **attempts count**, not by ever being transitioned to `error`.

**Observable failure state (direct answer).** On a mid-execution error, the row's `state` **stays `taken`**, `attempts` **climbs toward 5**, `taken_at` holds the last pickup time, and `state` **never reaches `done`**. There is no transition to `error` from the runner.

### Rationale / Thinking

- **At-least-once execution with bounded, time-spaced retries.** Incrementing `attempts` and committing *before* dispatch (`job_runner.py:340-341`) guarantees that a crashed, killed, or infinitely-looping job is still *counted*. A poison job therefore cannot retry forever — once `attempts` reaches `JOB_MAX_ATTEMPTS` (5), it drops out of `get_jobs_to_run()` selection (`job_runner.py:320`).
- **The 30-minute gap** (`JOB_TAKEN_RETRY_WAIT_MINS`, `app/config.py:565`, applied at `job_runner.py:319`) prevents tight retry loops and gives transient failures — e.g., a downstream outage — time to clear before the next attempt.
- **No `try/except` is a deliberate "fail fast / let the supervisor recover" posture.** A single failing job takes down the whole runner until the supervisor restarts it. The flip side is that, because success is the *only* path that sets `state = done` (`job_runner.py:344`), failures remain visible as lingering `taken` rows with elevated `attempts`.
- **The defined-but-unused `JobState.error` is a genuine code quirk.** Cleanup is written to purge `error` jobs (`tasks/cleanup_old_jobs.py:15`), yet nothing in the runner ever sets that state. It is reported here as an observation, not corrected.

---


## Q3 — Email forwarding bounce address (VERP) & forward-vs-reply handling

### Question

During forwarding through an alias, what is the exact format of the special bounce-handling return-path (envelope sender)? When a bounce arrives at that address, how does the system identify the original email, what state changes are recorded, and how does handling differ by direction (forward phase vs reply phase)?

### Answer (citation-backed)

**Address generation.** `generate_verp_email(verp_type, object_id, sender_domain=None)` (`app/email_utils.py:1438-1464`) builds the return-path as follows:

```python
data = [
    verp_type.value,                              # app/email_utils.py:1447
    object_id or 0,                               # app/email_utils.py:1448
    int((time.time() - VERP_TIME_START) / 60),    # app/email_utils.py:1449
]                                                 #   -> [type, id, minutes-since-2022-01-01]
json_payload = json.dumps(data).encode("utf-8")   # app/email_utils.py:1451
payload_hmac = hmac.new(
    config.VERP_EMAIL_SECRET.encode("utf-8"),
    json_payload, VERP_HMAC_ALGO,                 # app/email_utils.py:1454-1455
).digest()[:8]                                    # first 8 bytes only -> :1456
encoded_payload   = base64.b32encode(json_payload).rstrip(b"=").decode("utf-8")   # :1457
encoded_signature = base64.b32encode(payload_hmac).rstrip(b"=").decode("utf-8")   # :1458
return "{}.{}.{}@{}".format(
    config.VERP_PREFIX, encoded_payload, encoded_signature,
    sender_domain or config.EMAIL_DOMAIN,
).lower()                                          # app/email_utils.py:1459-1464
```

The time base `VERP_TIME_START = 1640995200` (2022-01-01) is at `app/email_utils.py:68`, and the HMAC algorithm `VERP_HMAC_ALGO = "sha3-224"` is at `app/email_utils.py:69`. With `VERP_PREFIX = "sl"` (`app/config.py:500`), the **exact address format is**:

```text
sl.<base32-payload>.<base32-hmac>@<domain>
```

The whole address is lowercased; `<domain>` is the passed `sender_domain` or the default `EMAIL_DOMAIN`. The payload encodes `[verp_type, object_id, minutes-since-2022]` and the signature is the first 8 bytes of an SHA3-224 HMAC over that payload, both base32-encoded with `=` padding stripped.

**Address decoding / verification.** `get_verp_info_from_email(email)` (`app/email_utils.py:1467-1498`) reverses the process:

- It splits the local-part (before `@`) on `.`, and requires exactly **3 fields** with `fields[0] == VERP_PREFIX`, else returns `None` (`app/email_utils.py:1471-1477`).
- It base32-decodes the payload and signature (restoring `=` padding), guarded by `except binascii.Error: return None` (`app/email_utils.py:1478-1486`).
- It **recomputes the HMAC and rejects on mismatch:** `if expected_signature != signature: return None` (`app/email_utils.py:1487-1491`) — this makes the token **tamper-evident**.
- It applies a **future-timestamp upper-bound (sanity) check**: `if data[2] > (time.time() + config.VERP_MESSAGE_LIFETIME - VERP_TIME_START) / 60: return None` (`app/email_utils.py:1496`), where `VERP_MESSAGE_LIFETIME = 5 * 86400` (5 days, `app/config.py:499`). Here `data[2]` is the token's timestamp in *minutes-since-2022* and the right-hand side is *now + 5 days* in the same units, so the condition rejects **only** tokens whose embedded timestamp lies more than 5 days in the **future** (a clock-sanity bound). It does **not** reject old/stale tokens — a bounce address generated days or weeks ago still validates — so despite the `…_LIFETIME` name this condition does **not** expire aged addresses (see [Documented Quirks](#documented-quirks-observed-not-fixed) §5).
- On success it returns `(VerpType(data[0]), data[1])` — i.e. `(VerpType, object_id)` (`app/email_utils.py:1498`).

**`VerpType(EnumE)`** (`app/models.py:247-250`) is `bounce_forward = 0`, `bounce_reply = 1`, `transactional = 2`.

**What `object_id` is (an important nuance).** For the two **bounce** directions, `object_id` is the **`EmailLog.id`**:

- Forward delivery sets the envelope sender via `generate_verp_email(VerpType.bounce_forward, email_log.id, contact_domain)` (`email_handler.py:905`).
- Reply delivery uses `generate_verp_email(VerpType.bounce_reply, email_log.id, alias_domain)` (`email_handler.py:1225`).
- For completeness, the **transactional** variant encodes a different id: `generate_verp_email(VerpType.transactional, transaction.id, alias_domain)` (`email_handler.py:1290`), where `transaction.id` is a `TransactionalEmail.id`, **not** an `EmailLog.id`. So the precise statement is: *for `bounce_forward` and `bounce_reply` the embedded id is the `EmailLog.id`; the transactional VERP encodes a `TransactionalEmail.id`.*

**Inbound identification** happens in `handle(envelope, msg)` (`email_handler.py:1945+`):

- The recipient is decoded first: `verp_info = get_verp_info_from_email(rcpt_tos[0])` (`email_handler.py:2035`).
- For a **forward** bounce (`verp_info[0] == VerpType.bounce_forward`, `email_handler.py:2061`): `email_log_id = (verp_info and verp_info[1]) or parse_id_from_bounce(rcpt_tos[0])` (`:2062`), then `email_log = EmailLog.get(email_log_id)` (`:2063`) — an **O(1) primary-key lookup**. If missing → `return status.E512` (`:2067`); if it is a bounce → `return handle_bounce(envelope, email_log, msg)` (`:2070`).
- For a **reply** bounce (`verp_info[0] == VerpType.bounce_reply`, `email_handler.py:2080`): symmetric logic at `email_handler.py:2082-2091`.
- iCloud-style bounces, where the VERP token arrives in `mail_from` rather than `rcpt_to`, are handled by a second decode `get_verp_info_from_email(mail_from[0])` (`email_handler.py:2101`).

**The unified dispatcher** `handle_bounce(envelope, email_log, msg) -> str` (`email_handler.py:1851-1914`):

- `if not email_log: return status.E512` (`email_handler.py:1856-1858`).
- `if not email_log.user.is_active(): return status.E510` (`email_handler.py:1869-1871`).
- It then **branches on `email_log.is_reply`** (`email_handler.py:1873`):
  - **Reply (`is_reply` true):** an **auto-reply special case is checked first** — `content_type = msg.get_content_type().lower()`; `if content_type != "multipart/report" or envelope.mail_from != "<>":` the message is treated as an auto-reply: it sets `email_log.auto_replied = True; Session.commit()` (`email_handler.py:1887-1888`), rewrites the `To` header and `envelope.rcpt_tos` to `alias.email` (`:1891-1892`), re-delivers through `handle_forward(...)` (`:1899`), and returns that delivery's SMTP status. Otherwise it is a true reply-phase bounce → `handle_bounce_reply_phase(envelope, msg, email_log)` then `return status.E212` (`email_handler.py:1910-1911`).
  - **Forward (`else`):** `handle_bounce_forward_phase(msg, email_log)` then `return status.E211` (`email_handler.py:1912-1914`).

**Forward-phase handler** `handle_bounce_forward_phase(msg, email_log)` (`email_handler.py:1432-1593`):

- Records a `Bounce` keyed on **`mailbox.email`**: `Bounce.create(email=mailbox.email, info=…, commit=True)` (`email_handler.py:1449-1454`).
- Uploads the **full bounce report** to S3 at `refused-emails/full-{random_name}.eml` (`email_handler.py:1463-1466`) and, when the original message can be parsed from the bounce, the **original message** at `refused-emails/{random_name}.eml` (`email_handler.py:1482-1485`), via `s3.upload_email_from_bytesio(...)`.
- Creates a `RefusedEmail.create(path=…, full_report_path=…, user_id=user.id)` (`email_handler.py:1487-1489`).
- Sets on the `email_log`: `bounced = True` (`:1493`), `refused_email_id = refused_email.id` (`:1494`), `bounced_mailbox_id = mailbox.id` (`:1495`), then `Session.commit()` (`:1496`).
- Performs a **conditional alias auto-disable**: `alias_will_be_disabled, reason = should_disable(alias)` (`email_handler.py:1500`); if true → `change_alias_status(alias, enabled=False, …)` (`:1505`), create a disable `Notification` (`:1509`), and `send_email_with_rate_control(user, ALERT_BOUNCE_EMAIL, user.email, …)` (`:1519-1521`). Otherwise → create a bounce `Notification` (`:1551`) and alert `user.email` via `ALERT_BOUNCE_EMAIL` (`:1565-1567`).

**Reply-phase handler** `handle_bounce_reply_phase(envelope, msg, email_log)` (`email_handler.py:1595-1687`):

- Records a `Bounce` keyed on **`sanitize_email(contact.website_email, not_lower=True)`** (`email_handler.py:1609-1617`).
- Performs the same S3 uploads (`refused-emails/full-{random_name}.eml` at `:1624` and `refused-emails/{random_name}.eml` at `:1632`) and `RefusedEmail.create(...)` (`email_handler.py:1637`).
- Sets `email_log.bounced = True` (`:1642`), `refused_email_id` (`:1643`), `bounced_mailbox_id` (`:1645`), then `Session.commit()` (`:1647`).
- Performs **no alias auto-disable** — there is no `should_disable` call in this handler.
- Creates a `Notification` (`email_handler.py:1657`) and sends `send_email_with_rate_control(user, ALERT_BOUNCE_EMAIL_REPLY_PHASE, mailbox.email, …)` (`email_handler.py:1668-1670`).

**`should_disable(alias)`** (`app/email_utils.py:1166`) returns a tuple `(bool, str)` — whether to disable plus a reason. It bypasses when `alias.cannot_be_disabled` (`:1171-1173`) or `not config.ALIAS_AUTOMATIC_DISABLE` (`:1175-1176`); otherwise it counts **forward-phase** bounces (`EmailLog.bounced == True AND EmailLog.is_reply == False`, `:1182-1183`) and disables when there are more than 12 in 24h (`:1190-1191`), or more than 5 in 24h combined with more than 10 over 7 days (`:1194-1211`), among further day-frequency tiers (`:1212-1230`).

**SMTP status strings** (`app/email/status.py`, quoted verbatim):

- `E200 = "250 Message accepted for delivery"` (`app/email/status.py:2`).
- `E211 = "250 SL E211 Bounce Forward phase handled"` (`app/email/status.py:19`).
- `E212 = "250 SL E212 Bounce Reply phase handled"` (`app/email/status.py:20`).
- `E510 = "550 SL E510 so such user"` (`app/email/status.py:47`) — note the **source typo**: it reads "**so** such user" (not "no such user").
- `E512 = "550 SL E512 No such email log"` (`app/email/status.py:49`).

**Supporting models and storage.**

- `EmailLog` (`app/models.py:2060+`) — relevant columns: `is_reply` (`:2075`), `bounced` (`:2082`), `auto_replied` (`:2085`), `refused_email_id` (FK with `ondelete="SET NULL"`, `:2097`), `mailbox_id` (`:2103`), `bounced_mailbox_id` (`:2109`), plus the `get_phase()` helper (`:2143`).
- `RefusedEmail` (`app/models.py:2858-2884`) — `full_report_path` (`sa.String(128)`, `:2864`) and `path` (`sa.String(128)`, `:2867`), `delete_at` (default `_expiration_7d`, ~7 days, `:2872`), and `get_url()` (`:2877`).
- `Bounce` (`app/models.py:3290-3297`) — `email` (`sa.String(256)`, indexed, `:3294`) and `info` (`sa.Text`, `:3295`); the docstring reads "Record all bounces. Deleted after 7 days" (`:3291`).
- `app/s3.py` — `upload_email_from_bytesio(path, bs, filename)` (`:47`) and `upload_from_bytesio(key, bs, …)` (`:28`) both branch on `config.LOCAL_FILE_UPLOAD` (local-disk fallback at `:50` / `:31`) and otherwise call `_get_s3client().put_object(...)` (boto3, at `:58` / `:39`).

### Forward-vs-Reply comparison (table)

| Aspect | Forward phase | Reply phase |
|---|---|---|
| Handler | `handle_bounce_forward_phase()` (`email_handler.py:1432-1593`) | `handle_bounce_reply_phase()` (`email_handler.py:1595-1687`) |
| Trigger | Inbound mail to the alias cannot be delivered to the user's mailbox | The user's reply from the alias cannot be delivered to the external contact |
| `VerpType` | `bounce_forward` (0) | `bounce_reply` (1) |
| `Bounce.email` keyed on | `mailbox.email` (`email_handler.py:1449-1454`) | `sanitize_email(contact.website_email, not_lower=True)` (`email_handler.py:1609-1617`) |
| Alias auto-disable | Yes, if `should_disable(alias)` (`app/email_utils.py:1166`) → `change_alias_status(enabled=False)` (`email_handler.py:1500-1505`) | No (no `should_disable` call) |
| Alert recipient / config | `user.email` via `ALERT_BOUNCE_EMAIL` (`email_handler.py:1519-1521`, `:1565-1567`) | `mailbox.email` via `ALERT_BOUNCE_EMAIL_REPLY_PHASE` (`email_handler.py:1668-1670`) |
| SMTP status returned | `E211` "Bounce Forward phase handled" (`app/email/status.py:19`) | `E212` "Bounce Reply phase handled" (`app/email/status.py:20`) |
| Special case | — | Non-`multipart/report` body OR non-empty `mail_from` → treated as auto-reply: `email_log.auto_replied = True`, re-forwarded to the alias via `handle_forward` (`email_handler.py:1873-1908`) |
| Common to both | Persist a `Bounce` row; upload full report + original message to S3 (`refused-emails/full-{random_name}.eml`, `refused-emails/{random_name}.eml`); create a `RefusedEmail`; set `email_log.bounced=True` + `refused_email_id` + `bounced_mailbox_id`; create a `Notification`; send a rate-controlled alert | (same) |

### Rationale / Thinking

- **Self-describing, signed, time-bounded token.** The return-path is a VERP token that carries everything needed to route a bounce. Because the `EmailLog.id` is embedded directly in the address, inbound identification is **O(1)** — a single `EmailLog.get(id)` (`email_handler.py:2063`) — with **no need to parse the human-readable bounce body**, which is notoriously MTA-specific and unreliable.
- **Security properties.** The **SHA3-224 HMAC truncated to the first 8 bytes** (`app/email_utils.py:1454-1456`) makes the token tamper-evident: a forged or edited address fails the `expected_signature != signature` check (`app/email_utils.py:1490-1491`) and is rejected. The **`VERP_MESSAGE_LIFETIME` check** (`app/config.py:499`, applied at `app/email_utils.py:1496`) is a *future*-timestamp upper bound: it rejects only tokens whose embedded timestamp is more than 5 days **ahead** of the current clock, so it acts as a clock-sanity guard and does **not** block replay of *stale* bounce addresses — an aged but validly-signed token is still accepted (see [Documented Quirks](#documented-quirks-observed-not-fixed) §5). The only freshness-like protections that actually hold are therefore the **HMAC signature** (forgery resistance) and the short-lived nature of the underlying `EmailLog`/S3 artifacts. **Base32** (rather than base64) is used deliberately (see the inline comment at `app/email_utils.py:1452-1453`) so the address contains only DNS/email-safe characters and is case-insensitive after `.lower()` (`app/email_utils.py:1464`).
- **Why the forward/reply split exists.** The *failing party* differs by direction — the user's own mailbox in the forward direction versus the external contact in the reply direction. That is why the recorded `Bounce.email`, the alert recipient, and — crucially — the **protective action** differ. Only the **forward** direction can auto-disable the alias (via `should_disable`, which counts forward-phase bounces, `app/email_utils.py:1182-1183`), because repeated undeliverable inbound mail indicates a bad alias/mailbox pairing; reply-phase failures point at the external contact and must **not** disable the user's alias.
- **The reply-phase auto-reply special case** (`email_handler.py:1876`) exists to distinguish genuine delivery failures — a `multipart/report` message with an empty `mail_from` (i.e. `<>`) — from vacation/auto-responder messages. The latter are re-injected to the alias via `handle_forward` (`email_handler.py:1899`) rather than being mistaken for bounces, so an out-of-office reply reaches the user instead of silently disappearing into bounce handling.

---


## Documented Quirks (observed, NOT fixed)

The following are real behaviors observed in the source. They are reported here as findings for transparency; **none of them is changed or proposed for fix** as part of this analysis.

1. **`JobState.error` is defined but never assigned by the runner.** The enum value is defined at `app/models.py:257`; the only production reference outside the enum is `tasks/cleanup_old_jobs.py:15` (plus tests). `job_runner.py` only ever sets `JobState.taken` (`:339`) and `JobState.done` (`:344`), so a failed job never reaches the `error` state.
2. **`/mailbox_verify` has no rate-limit decorator.** The route carries only `@dashboard_bp.route("/mailbox_verify")` and `@login_required` (`app/dashboard/views/mailbox.py:120-121`), in contrast to the `@parallel_limiter.lock(...)` applied to `mailbox_route` (`app/dashboard/views/mailbox.py:38`). Protection relies entirely on the application-level `tries` ceiling, the 15-minute expiry, and the high-entropy code.
3. **No `try`/`except` around `process_job()`.** The dispatch at `job_runner.py:342` is unguarded (a file-wide search for `try:`/`except` returns zero matches), so an exception propagates out and terminates the runner; restart is delegated to an external supervisor.
4. **SMTP `E510` string typo.** `app/email/status.py:47` reads `"550 SL E510 so such user"` — "so" appears where "no" was presumably intended.
5. **`VERP_MESSAGE_LIFETIME` does not expire stale bounce addresses.** Despite the name, the only check that uses it (`app/email_utils.py:1496`) is `if data[2] > (time.time() + config.VERP_MESSAGE_LIFETIME - VERP_TIME_START) / 60: return None`, which compares the token's embedded timestamp (`data[2]`, minutes-since-2022) against *now + 5 days* in the same units. It therefore rejects only timestamps set **more than 5 days in the future** (a clock-sanity upper bound) and **never** rejects **old** timestamps, so a VERP bounce address generated long ago still validates. No condition in `get_verp_info_from_email()` (`app/email_utils.py:1467-1498`) enforces a *lower* time bound. Reported as an observation; the source is unchanged.

## Reference Files

All ten files below were consulted **read-only** (traced and cited, never modified). `app/models.py` and `app/config.py` are shared across questions.

- **Q1 (mailbox verification lockout):** `app/mailbox_utils.py`, `app/models.py`, `app/dashboard/views/mailbox.py`.
- **Q2 (background task lifecycle and error/retry):** `job_runner.py`, `app/config.py`, `tasks/cleanup_old_jobs.py`, `app/models.py`.
- **Q3 (VERP bounce address and forward-vs-reply handling):** `app/email_utils.py`, `email_handler.py`, `app/email/status.py`, `app/s3.py`, `app/config.py`, `app/models.py`.

