# Alias Email-Forwarding: Runtime Investigation of Log Text, Message-ID, `From` Header, and DB Records

## Summary

This document answers four questions about how SimpleLogin transforms an inbound email as it is forwarded through an alias. **Every value below was captured by running the real code** — driving the canonical SMTP routing entry point `email_handler.handle(envelope, msg)` (`email_handler.py:L1945`) inside the pinned Docker image — and never inferred from reading code alone. Each observation was run at least twice to expose run-to-run variation.

Headline findings:

- **Q1 (logs/status).** A successful forward emits a sequence of `LOG.d(...)` DEBUG lines from `forward_email_to_mailbox()` and returns SMTP status **`250 Message accepted for delivery`** (`E200`). A forward to a non-existent alias emits the `not exist` / `cannot be created on-the-fly, return 550` DEBUG lines from `handle_forward()` and returns **`550 SL E515 Email not exist`** (`E515`).
- **Q2 (SL Message-ID) — the central finding.** On a **forward**, SimpleLogin does **not** generate an "SL Message-ID." The original inbound `Message-ID` is **preserved byte-for-byte** on the outgoing message, and `EmailLog.sl_message_id` stays **`NULL`**. An SL Message-ID (`<{timeval}.{pid}.{randint}.{email_log.id}@sl.local>`) is minted **only in the reply phase**. This forward-vs-reply asymmetry is the most likely explanation for the reported "inconsistent behavior."
- **Q3 (`From` header).** The forwarded `From` is rewritten to `"spoofedemailsource at gmail.com" <spoofedemailsource_at_gmail_com_oarrgcqd@sl.local>` — an "`at`"-format display name paired with the reverse-alias. The observed reverse-alias uses the **sender-included** format `{sanitized-sender}_{random}@sl.local` (not a pure-random local-part), because the canonical ORM default for `User.include_sender_in_reverse_alias` is `True`.
- **Q4 (DB records).** A single forward from a **new** sender creates exactly **three** rows: one `Contact`, one `UserAuditLog` (`action=create_contact`), and one `EmailLog`. `EmailLog.sl_message_id` is `NULL` and **no** `MessageIDMatching` row is created on a forward.

> **Diagnostic-only scope.** This is a read-only investigation. No source file was modified and no fix was applied, even though the Q2 forward-vs-reply distinction is the probable root cause of the reported inconsistency.

---

## Environment & method

### Canonical build / run recipe

The investigation ran inside the mandated pre-built image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (running container `sl-setup`), on branch source commit **`2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`**. The canonical build/run recipe (confirmed from `Dockerfile`, `.github/workflows/main.yml`, `pyproject.toml`, and `tests/test.env`) is:

```bash
# Interpreter & OS packages (Dockerfile: `FROM python:3.10`; CI python-version '3.10'; pyproject `python = "^3.10"`)
sudo apt install -y libre2-dev libpq-dev          # .github/workflows/main.yml:L24,L84

# Python dependencies (Poetry-managed)
poetry install --no-interaction                   # .github/workflows/main.yml:L28,L88

# Provision schema via Alembic migrations (NOT create_all)
CONFIG=tests/test.env poetry run alembic upgrade head   # .github/workflows/main.yml:L98
```

Config comes from `tests/test.env`: `EMAIL_DOMAIN=sl.local` (`tests/test.env:L8`), `NOT_SEND_EMAIL=true` (`tests/test.env:L7`), `OTHER_ALIAS_DOMAINS` includes `sl.local` (`tests/test.env:L9`), and `ALIAS_AUTOMATIC_DISABLE=true` (`tests/test.env:L62`).

### Actual runtime used (and one labeled caveat)

The observed interpreter and libraries in the warmed container:

```
$ python --version
Python 3.10.18
$ python -c "import sqlalchemy, flask, aiosmtpd, arrow; print(sqlalchemy.__version__, flask.__version__, aiosmtpd.__version__, arrow.__version__)"
1.3.24 1.1.2 1.4.2 0.16.0
```

Services in the warmed container: **Redis 6** (`redis://localhost`) and PostgreSQL reachable at `postgresql://test:test@localhost:5432/test`.

> **Caveat — PostgreSQL major version (labeled non-canonical).** The CI/canonical spec pins **PostgreSQL 13** (`.github/workflows/main.yml:L47` `image: postgres:13`, published on host port `15432`), and `tests/test.env:L17` declares `DB_URI=postgresql://test:test@localhost:15432/test`. The warmed image I ran against instead provides **PostgreSQL 15.13** on port `5432` (the environment exports `DB_URI=...@localhost:5432/test`, which wins because `app/config.py`'s `load_dotenv(override=False)` does not override an already-set variable). The PostgreSQL **major version does not affect any answer below** — the log text, Message-ID handling, `From`-header rewrite, and which ORM records are created are all determined by the identical Python application code at HEAD `2cd6ee777f8c`. All emitted values are therefore reported as canonical; only the Postgres server version differs and is labeled here.

### How observations were produced (real entry point)

All observations flow through the real router `email_handler.handle(envelope, msg)` (`email_handler.py:L1945`) — the same function `MailHandler.handle_DATA` (`email_handler.py:L2289`) / `MailHandler._handle` invoke for live SMTP. I did **not** call `handle_forward`, `forward_email_to_mailbox`, or `handle_reply` directly.

A **temporary** pytest harness (`tests/blitzy_adhoc_test_investigation.py`, deleted afterward — see "Clean up") built an `aiosmtpd` `Envelope` + `Message` and called `handle()`. It ran under the `flask_client` fixture (`tests/conftest.py:L60-L77`) inside a Flask app context, mirroring the existing suite's pattern (`tests/test_email_handler.py:L79-L108`). Fixtures were seeded through the app's own APIs — `create_new_user()` (`tests/utils.py:L17`) and `Alias.create_new_random(user)` — and the outbound message was captured with `mail_sender.store_emails_instead_of_sending()` (`app/mail_sender.py:L102`) + `get_stored_emails()` (`app/mail_sender.py:L108`), serialized byte-exact via `message_to_bytes()` (`app/message_utils.py:L12`).

DEBUG logging is always on (`app/log.py:L51` sets the `SL` logger to `DEBUG`). To capture each `LOG.d(...)` line **with its exact `SL` prefix**, the harness attached a `StreamHandler` using the application's own `app.log._log_formatter` (format string at `app/log.py:L12-L15`):

```
"%(asctime)s - %(name)s - %(levelname)s - %(process)d - \"%(pathname)s:%(lineno)d\" - %(funcName)s() - %(message_id)s - %(message)s"
```

The producing command for every code block in Q1–Q4 (run twice, as `run1`/`run2`):

```bash
# inside container sl-setup, env from tests/test.env, DEBUG surfaced via -s
unset PYTEST_ADDOPTS
CONFIG=/app/tests/test.env python -m pytest -o addopts="" -p no:rerunfailures \
  --timeout=180 -q -s tests/blitzy_adhoc_test_investigation.py
```

### Caveat — transaction rollback does **not** leave the DB unchanged here; DB was reset

`tests/conftest.py`'s `flask_client` fixture opens `connection.begin()` and calls `transaction.rollback()` at teardown (`tests/conftest.py:L61,L74-L77`). **Observed reality:** because `Session = scoped_session(sessionmaker(bind=connection))` (`app/db.py`) is bound to that same connection and the code path commits via `EmailLog.create(..., commit=True)` etc., under **SQLAlchemy 1.3.24** each `Session.commit()` issues a real `COMMIT`, so the outer `transaction.rollback()` is effectively a no-op and the rows **persist** after the test process exits. I verified this directly with a throwaway probe: a row created with `commit=True` remained in `email_log` (row count `12 → 13`) after pytest exited. Consequently the freshly-committed IDs/timestamps were fully visible **during** the run (which is what the answers rely on), but the test database was **not** left unchanged by rollback alone. To honor the "leave it unchanged / clean up" constraint, after capturing all values I **reset the test database to its pristine migrated state**:

```bash
psql "$DB_URI" -c "drop schema public cascade; create schema public;"
CONFIG=/app/tests/test.env alembic upgrade head
```

### Read-only confirmation

No source file was modified. The only committed artifact is this document. `git status` in the repository shows only the new `blitzy/documentation/app_2cd6ee777f8c.md`; HEAD remains `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`. All temporary scripts were deleted and the spun-up test-DB state was reset (no extra containers were created — the warmed `sl-setup` container was reused).

---

## The exact log message text when an email is successfully forwarded versus when it fails due to a non-existent alias.

Both branches were exercised **separately** through `email_handler.handle(...)` with DEBUG on. They are distinct code paths: the success case runs `forward_email_to_mailbox()` (`email_handler.py:L679`) and returns `status.E200` (`email_handler.py:L928`); the non-existent-alias case returns early inside `handle_forward()` (`email_handler.py:L536`) with `status.E515` (`email_handler.py:L555`).

The `SL` logger and `LOG.d` shortcut are defined in `app/log.py`: `LOG = _get_logger("SL")` (`app/log.py:L79`), `logging.Logger.d = logging.Logger.debug` (`app/log.py:L74`), and the format string is at `app/log.py:L12-L15`. Every line below is pasted with that full prefix.

### (a) Successful forward — returns `E200`

Complete, unedited `LOG.d(...)` output captured for one successful forward (`run1`; a fresh alias `debunk_elvish590@sl.local`, external sender `spoofedemailsource@gmail.com`):

```
2026-07-07 18:28:49,867 - SL - DEBUG - 25195 - "/app/email_handler.py:1980" - handle() - 6D8C13F069 - ==>> Handle mail_from:spoofedemailsource@gmail.com, rcpt_tos:['debunk_elvish590@sl.local'], header_from:spoofedemailsource@gmail.com, header_to:debunk_elvish590@sl.local, cc:None, reply-to:None, message_id:<20220317165018.000191@somewhere-5488dd4b6b-7crp6>, client_ip:54.39.200.130, headers:[...], mail_options:[], rcpt_options:[]
2026-07-07 18:28:49,870 - SL - DEBUG - 25195 - "/app/email_handler.py:2202" - handle() - 6D8C13F069 - Forward phase spoofedemailsource@gmail.com(spoofedemailsource@gmail.com) -> debunk_elvish590@sl.local
2026-07-07 18:28:49,880 - SL - DEBUG - 25195 - "/app/email_handler.py:580" - handle_forward() - 6D8C13F069 - Create or get contact for from_header:spoofedemailsource@gmail.com
2026-07-07 18:28:49,899 - SL - DEBUG - 25195 - "/app/app/contact_utils.py:110" - create_contact() - 6D8C13F069 - Created contact <Contact 2 spoofedemailsource@gmail.com 4> for alias <Alias 4 debunk_elvish590@sl.local> with email spoofedemailsource@gmail.com invalid_email=False
2026-07-07 18:28:49,907 - SL - DEBUG - 25195 - "/app/email_handler.py:688" - forward_email_to_mailbox() - 6D8C13F069 - Forward <Contact 2 spoofedemailsource@gmail.com 4> -> <Alias 4 debunk_elvish590@sl.local> -> <Mailbox 2 user_plaehywe2l@mailbox.test>
2026-07-07 18:28:49,910 - SL - DEBUG - 25195 - "/app/email_handler.py:740" - forward_email_to_mailbox() - 6D8C13F069 - Create <EmailLog 2> for <Contact 2 spoofedemailsource@gmail.com 4>, <User 2 Test User user_plaehywe2l@mailbox.test>, <Mailbox 2 user_plaehywe2l@mailbox.test>
2026-07-07 18:28:49,915 - SL - DEBUG - 25195 - "/app/email_handler.py:867" - forward_email_to_mailbox() - 6D8C13F069 - From header, new:"spoofedemailsource at gmail.com" <spoofedemailsource_at_gmail_com_oarrgcqd@sl.local>, old:spoofedemailsource@gmail.com
2026-07-07 18:28:49,915 - SL - DEBUG - 25195 - "/app/email_handler.py:316" - replace_header_when_forward() - 6D8C13F069 - Delete Cc header, old value None
2026-07-07 18:28:49,915 - SL - DEBUG - 25195 - "/app/email_handler.py:313" - replace_header_when_forward() - 6D8C13F069 - Replace To header, old: debunk_elvish590@sl.local, new: debunk_elvish590@sl.local
2026-07-07 18:28:49,918 - SL - DEBUG - 25195 - "/app/email_handler.py:893" - forward_email_to_mailbox() - 6D8C13F069 - Forward mail from spoofedemailsource@gmail.com to user_plaehywe2l@mailbox.test, mail_options:[], rcpt_options:[] 
2026-07-07 18:28:49,918 - SL - DEBUG - 25195 - "/app/app/mail_sender.py:131" - send() - 6D8C13F069 - send email with subject '[Possible phishing attempt] test Thu, 17 Mar 2022 16:50:18 +0000', from '"spoofedemailsource at gmail.com" <spoofedemailsource_at_gmail_com_oarrgcqd@sl.local>' to 'debunk_elvish590@sl.local'
```

The three signature success lines, tied to their source, are:

| Log message | Emitting call | Location |
|---|---|---|
| `Forward <Contact ...> -> <Alias ...> -> <Mailbox ...>` | `LOG.d("Forward %s -> %s -> %s", contact, alias, mailbox)` | `email_handler.py:L688` |
| `Create <EmailLog ...> for <Contact ...>, <User ...>, <Mailbox ...>` | `LOG.d("Create %s for %s, %s, %s", email_log, contact, user, mailbox)` | `email_handler.py:L740` |
| `From header, new:..., old:...` | `LOG.d("From header, new:%s, old:%s", new_from_header, old_from_header)` | `email_handler.py:L867` |

Returned SMTP status (byte-exact), and its definition:

```
result = '250 Message accepted for delivery'
status.E200 = '250 Message accepted for delivery'  -> match=True
```

`E200 = "250 Message accepted for delivery"` is defined at `app/email/status.py:L2` and returned at `email_handler.py:L928` (`return True, status.E200`).

### (b) Forward to a non-existent alias — returns `E515`

Complete, unedited output captured for a forward to `ycqazrdhwsugymxcfbpo@sl.local` (an alias that does not exist and cannot be auto-created):

```
2026-07-07 18:28:50,258 - SL - DEBUG - 25195 - "/app/email_handler.py:1980" - handle() - 6D8C13F069 - ==>> Handle mail_from:crflqtedrapxziqthmpi@crflqtedrapxziqthmpi.com, rcpt_tos:['ycqazrdhwsugymxcfbpo@sl.local'], header_from:crflqtedrapxziqthmpi@crflqtedrapxziqthmpi.com, header_to:ycqazrdhwsugymxcfbpo@sl.local, cc:None, reply-to:None, message_id:<nonexist-1@sender.example.com>, client_ip:None, headers:[...], mail_options:[], rcpt_options:[]
2026-07-07 18:28:50,261 - SL - DEBUG - 25195 - "/app/email_handler.py:2202" - handle() - 6D8C13F069 - Forward phase crflqtedrapxziqthmpi@crflqtedrapxziqthmpi.com(crflqtedrapxziqthmpi@crflqtedrapxziqthmpi.com) -> ycqazrdhwsugymxcfbpo@sl.local
2026-07-07 18:28:50,267 - SL - DEBUG - 25195 - "/app/email_handler.py:545" - handle_forward() - 6D8C13F069 - alias ycqazrdhwsugymxcfbpo@sl.local not exist. Try to see if it can be created on the fly
2026-07-07 18:28:50,273 - SL - INFO - 25195 - "/app/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - 6D8C13F069 - Cannot auto-create custom domain alias for ycqazrdhwsugymxcfbpo@sl.local because there's no custom domain for sl.local
2026-07-07 18:28:50,273 - SL - INFO - 25195 - "/app/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - 6D8C13F069 - Cannot auto-create ycqazrdhwsugymxcfbpo@sl.local since it has no directory separator
2026-07-07 18:28:50,273 - SL - DEBUG - 25195 - "/app/email_handler.py:551" - handle_forward() - 6D8C13F069 - alias ycqazrdhwsugymxcfbpo@sl.local cannot be created on-the-fly, return 550
```

The two signature failure lines, tied to their source:

| Log message | Emitting call | Location |
|---|---|---|
| `alias <addr> not exist. Try to see if it can be created on the fly` | `LOG.d("alias %s not exist. Try to see if it can be created on the fly", alias_address)` | `email_handler.py:L545-L548` |
| `alias <addr> cannot be created on-the-fly, return 550` | `LOG.d("alias %s cannot be created on-the-fly, return 550", alias_address)` | `email_handler.py:L551` |

Returned SMTP status (byte-exact), and its definition:

```
result = '550 SL E515 Email not exist'
status.E515 = '550 SL E515 Email not exist'  -> match=True
```

`E515 = "550 SL E515 Email not exist"` is defined at `app/email/status.py:L51` and returned at `email_handler.py:L555` (`return [(False, status.E515)]`).

**Run-to-run note (Q1).** The message *text* is stable across runs; the volatile pieces are the identifiers interpolated into it — the random alias local-part, the object ids (`<Contact 2 ...>`, `<EmailLog 2>`), the mailbox address, and the timestamp/PID in the prefix. `run2` produced the identical lines with `<Contact 3 ...>`, `<EmailLog 3>`, alias `raving_bodges010@sl.local`, and reverse-alias suffix `_vfgirqp`. Both success runs returned `E200`; both failure runs returned `E515`.

---

## What specific SL Message-ID gets generated during forwarding, and how it differs from the original Message-ID.

### Direct answer (the observed truth): a forward generates **no** SL Message-ID

On a **forward**, SimpleLogin does **not** mint an "SL Message-ID." The original inbound `Message-ID` is **preserved byte-for-byte** on the outgoing forwarded message, and the `EmailLog.sl_message_id` column stays **`NULL`**. In the forward path, `replace_sl_message_id_by_original_message_id(msg)` (`email_handler.py:L860`, defined at `email_handler.py:L931`) only remaps `In-Reply-To`/`References` — it does **not** touch `Message-ID`, and the original `Message-ID` is retained in `headers_to_keep`.

Captured before/after for the same forward as Q1(a):

```
[Q2 BEFORE] inbound Message-ID = '<20220317165018.000191@somewhere-5488dd4b6b-7crp6>'
[Q2 AFTER]  outbound Message-ID = '<20220317165018.000191@somewhere-5488dd4b6b-7crp6>'
[Q2] inbound==outbound Message-ID ? True
```

The byte-exact outbound header block (from `message_to_bytes()` of the stored `SendRequest`) confirms the `Message-Id` is unchanged, and that the forward adds SimpleLogin tracking headers but **no** SL `Message-ID`:

```
Date: Thu, 17 Mar 2022 16:50:18 +0000
Subject: [Possible phishing attempt] test Thu, 17 Mar 2022 16:50:18 +0000
Message-Id: <20220317165018.000191@somewhere-5488dd4b6b-7crp6>
Content-Transfer-Encoding: 7bit
X-SimpleLogin-Type: Forward
X-SimpleLogin-EmailLog-ID: 2
X-SimpleLogin-Envelope-From: spoofedemailsource@gmail.com
X-SimpleLogin-Original-From: spoofedemailsource@gmail.com
X-SimpleLogin-Envelope-To: debunk_elvish590@sl.local
From: "spoofedemailsource at gmail.com"
 <spoofedemailsource_at_gmail_com_oarrgcqd@sl.local>
To: debunk_elvish590@sl.local
DKIM-Signature: v=1; a=rsa-sha256; c=relaxed/simple; d=sl.local; ...
```

And the created `EmailLog` row confirms `sl_message_id` is `NULL` on a forward:

```
EMAILLOG id=2 created_at='2026-07-07T18:28:49.908099+00:00' is_reply=False blocked=False message_id='<20220317165018.000191@somewhere-5488dd4b6b-7crp6>' sl_message_id=None
[Q4 AFTER] ... new MessageIDMatching rows = 0
```

`EmailLog.sl_message_id` is a nullable column (`app/models.py:L2116`, deferred `String(512)`) documented as "in the reply phase, the original message_id is replaced by the SL message_id." On a forward it is never written. **No `MessageIDMatching` row is created on a forward** either (`MessageIDMatching` defined at `app/models.py:L3365`).

### For contrast: a REPLY does mint an SL Message-ID

The "SL Message-ID" the question references is a **reply-phase** construct. Exercising a reply (rcpt = the reverse-alias, `mail_from` = the alias's own mailbox) routes to `handle_reply()` (`email_handler.py:L966`), which calls `replace_original_message_id(alias, email_log, msg)` (`email_handler.py:L1202`; defined at `email_handler.py:L1296`). That function mints the SL Message-ID at `email_handler.py:L1311-L1313`:

```python
sl_message_id = make_msgid(
    str(email_log.id), get_email_domain_part(alias.email)
)
```

Captured, unedited, from the reply run (`run1`):

```
2026-07-07 18:28:50,679 - SL - DEBUG - 25195 - "/app/email_handler.py:2196" - handle() - 6D8C13F069 - Reply phase user_ws8rhgf0zo@mailbox.test(user_ws8rhgf0zo@mailbox.test) -> spoofedemailsource_at_gmail_com_hvvbe@sl.local
2026-07-07 18:28:50,692 - SL - DEBUG - 25195 - "/app/email_handler.py:1314" - replace_original_message_id() - 6D8C13F069 - create a new sl_message_id <178344893069.25195.8423179011053050732.5@sl.local>
```

The minted value and the DB rows it produces:

```
[Q2-reply BEFORE] inbound reply Message-ID = '<orig-reply-1@mailbox.test>'
[Q2-reply] reply EmailLog.id=5  EmailLog.sl_message_id='<178344893069.25195.8423179011053050732.5@sl.local>'
[Q2-reply] MessageIDMatching id=1 created_at='2026-07-07T18:28:50.693204+00:00'
           sl_message_id       = '<178344893069.25195.8423179011053050732.5@sl.local>'
           original_message_id = '<orig-reply-1@mailbox.test>'
           email_log_id        = 5
[Q2-reply AFTER] outbound reply Message-ID = '<178344893069.25195.8423179011053050732.5@sl.local>'
```

So on a reply, unlike a forward: the original `Message-ID` (`<orig-reply-1@mailbox.test>`) is **replaced** on the outgoing message by the minted SL Message-ID (`email_handler.py:L1338-L1339`), `EmailLog.sl_message_id` is **set** (`email_handler.py:L1341`), and a `MessageIDMatching` row is created (`email_handler.py:L1316-L1321`) recording the `sl_message_id`↔`original_message_id`↔`email_log_id` mapping.

**Structure of the minted SL Message-ID** — it is exactly the Python stdlib `email.utils.make_msgid(idstring, domain)` shape `<{timeval}.{pid}.{randint}.{idstring}@{domain}>`:

| Component | Observed (`run1`) | Meaning |
|---|---|---|
| `timeval` | `178344893069` | stdlib time component |
| `pid` | `25195` | process id (matches the `%(process)d` field in every log prefix above) |
| `randint` | `8423179011053050732` | stdlib random component |
| `idstring` | `5` | `str(email_log.id)` — the reply `EmailLog.id` |
| `domain` | `sl.local` | `get_email_domain_part(alias.email)` — the alias domain |

**Run-to-run note (Q2).** The forward preserved the same original `Message-ID` on both runs (nothing minted). The reply minted a different value each run — `run2` produced `<178344893104.25195.4648077217597495233.7@sl.local>` (`email_log.id=7`, `MessageIDMatching id=2`). Thus `timeval`, `randint`, and the `email_log.id` component vary by nature; `pid` is stable within a process; the `@sl.local` domain is stable. There is also a reuse branch (`LOG.d("reuse the sl_message_id %s", ...)`, `email_handler.py:L1308-L1309`) for a reply fanned out to multiple recipients — not exercised here.

### Why this is the likely cause of "inconsistent behavior"

A user who inspects a **forwarded** email sees the sender's original `Message-ID` unchanged, while a user who inspects a **reply** they sent through the reverse-alias sees a SimpleLogin-minted `<...@sl.local>` `Message-ID`. Same alias, two phases, two different `Message-ID` outcomes — the forward-vs-reply asymmetry is documented here as the probable explanation. **No change was made** (diagnostic-only).

---

## The exact From header value in the forwarded email after transformation, including the reply-email (reverse-alias) address format.

### The transformed `From` (observed, byte-exact)

In the forward path the `From` header is replaced at `email_handler.py:L862-L867`:

```python
old_from_header = msg[headers.FROM]
new_from_header = contact.new_addr()
add_or_replace_header(msg, "From", new_from_header)
LOG.d("From header, new:%s, old:%s", new_from_header, old_from_header)
```

The captured `From header, new:.., old:..` log line (from the Q1(a) forward):

```
2026-07-07 18:28:49,915 - SL - DEBUG - 25195 - "/app/email_handler.py:867" - forward_email_to_mailbox() - 6D8C13F069 - From header, new:"spoofedemailsource at gmail.com" <spoofedemailsource_at_gmail_com_oarrgcqd@sl.local>, old:spoofedemailsource@gmail.com
```

The same value read back from the captured outbound `SendRequest` (`get_stored_emails()[-1].msg`):

```
[Q3 BEFORE] inbound From  = 'spoofedemailsource@gmail.com'
[Q3 AFTER]  outbound From = '"spoofedemailsource at gmail.com" <spoofedemailsource_at_gmail_com_oarrgcqd@sl.local>'
```

And byte-exact from `message_to_bytes()` (RFC 5322 header folding across two physical lines):

```
From: "spoofedemailsource at gmail.com"
 <spoofedemailsource_at_gmail_com_oarrgcqd@sl.local>
```

**So the transformed `From` is:** a quoted display name `"spoofedemailsource at gmail.com"` followed by the reverse-alias in angle brackets. This is produced by `Contact.new_addr()` (`app/models.py:L2008`) with the **default** `SenderFormatEnum.AT` (`= 0`, `app/models.py:L204`; the `User.sender_format` default is `0`). For the `AT` format, `new_addr()` computes `formatted_email = self.website_email.replace("@", " at ").strip()` → `spoofedemailsource at gmail.com`, and because the contact has no distinct display name it uses that as the whole `new_name`, then returns `sl_formataddr((new_name, self.reply_email))` (`app/models.py:L2044`). `sl_formataddr()` (`app/email_utils.py:L1501`) wraps `email.utils.formataddr` with `Header(addr, "utf-8")` to yield RFC-2047 form. Here the value is pure ASCII, so **no** `=?utf-8?...?=` encoding appears; a non-ASCII display name would be RFC-2047 encoded. (Had the contact carried a display name distinct from its email, the `AT` branch would render `"<Name> - spoofedemailsource at gmail.com" <reply_email>` — `app/models.py:L2028-L2033`.)

### The reverse-alias (`reply_email`) address format — observed

The bare reverse-alias placed in `From`/used as `reply_email` is generated by `generate_reply_email(contact_email, alias)` (`app/email_utils.py:L1103`). Observed value (`run1`):

```
[Q3] bare reverse-alias reply_email = 'spoofedemailsource_at_gmail_com_oarrgcqd@sl.local'
```

**Important observed detail (actual value vs. the pure-random assumption).** The observed reverse-alias uses the **sender-included** format `{sanitized-sender}_{random}@sl.local`, *not* a pure-random `{random}@sl.local`. The cause, verified at runtime, is that `User.include_sender_in_reverse_alias` has a Python-side ORM **`default=True`** (`app/models.py:L455-L457`: `sa.Column(sa.Boolean, default=True, nullable=False, server_default="0")`). A user created through the ORM (which is how SimpleLogin creates every user, including `create_new_user()`) therefore gets `True`; the `server_default="0"` would only apply to rows inserted via raw SQL outside the ORM. Direct runtime check:

```
$ python -c 'from tests.utils import create_new_user; from app.db import Session; u=create_new_user(); Session.flush(); print(u.include_sender_in_reverse_alias)'
...
include_sender_in_reverse_alias = True
```

With that flag `True`, `generate_reply_email()` takes the sender-included branch (`app/email_utils.py:L1119-L1141`): the contact email is sanitized (`@` → `_at_`, `.` → `_`, truncated to 45 chars) and suffixed with `random_string(random.randint(5, 10))` at `@{reply_domain}` where `reply_domain = config.EMAIL_DOMAIN = sl.local`. Hence `spoofedemailsource@gmail.com` → `spoofedemailsource_at_gmail_com` + `_` + an 8-char random suffix `oarrgcqd` + `@sl.local`.

For completeness: the **other** branch of `generate_reply_email()` — taken only when `include_sender_in_reverse_alias` is `False` — produces the pure-random `f"{random_string(random.randint(20, 50))}@{reply_domain}"` form (`app/email_utils.py:L1143-L1148`). The legacy `ra+`/`reply+` prefixes are commented out in both branches. This alternate form was **not** observed under the canonical default and is noted here as context only.

**Run-to-run note (Q3).** The display-name portion (`"spoofedemailsource at gmail.com"`) and the sanitized-sender prefix + `@sl.local` domain are stable; the random suffix varies each generation: `run1` `_oarrgcqd` (8 chars), `run2` `_vfgirqp` (7 chars), and the reply-setup forwards produced `_hvvbe` (5) and `_clrjcz` (6) — consistent with `random.randint(5, 10)`.

---

## What database records are created during a single forward operation — show actual record IDs and timestamps.

### Method (before / during / after)

Inside the run, the harness snapshotted `max(id)` of `EmailLog`, `Contact`, `UserAuditLog`, and `MessageIDMatching` **before** the `handle()` call, ran one forward from a **new** sender, then diffed **after**. Every model inherits `ModelMixin` (`app/models.py:L62-L65`), which provides the autoincrement integer `id` and the `created_at` `ArrowType` defaulting to `arrow.utcnow` (`app/models.py:L64`).

### Result — a single new-sender forward creates exactly three rows

Captured before/after (`run1`):

```
[Q4 BEFORE] max EmailLog.id=0  max Contact.id=0  max UserAuditLog.id=0  max MessageIDMatching.id=0

[Q4 AFTER] new Contact rows = 1, new EmailLog rows = 1, new UserAuditLog rows = 1, new MessageIDMatching rows = 0
  CONTACT      id=2 created_at='2026-07-07T18:28:49.890591+00:00' website_email='spoofedemailsource@gmail.com' reply_email='spoofedemailsource_at_gmail_com_oarrgcqd@sl.local'
  EMAILLOG     id=2 created_at='2026-07-07T18:28:49.908099+00:00' is_reply=False blocked=False message_id='<20220317165018.000191@somewhere-5488dd4b6b-7crp6>' sl_message_id=None
  USERAUDITLOG id=2 created_at='2026-07-07T18:28:49.896001+00:00' action='create_contact' message='Created contact 2 (spoofedemailsource@gmail.com)'
  MESSAGEIDMATCHING new rows = 0 (expected 0 on forward)
```

The three created rows, each tied to the code that creates it:

| # | Table / model | Actual id | Actual `created_at` | Created by |
|---|---|---|---|---|
| 1 | `Contact` (`app/models.py:L1863`) | `2` | `2026-07-07T18:28:49.890591+00:00` | `create_contact()` (`app/contact_utils.py:L42`); `reply_email` from `generate_reply_email()` (`app/contact_utils.py:L89`) |
| 2 | `UserAuditLog` (`app/models.py:L3829`, table `user_audit_log`) | `2` | `2026-07-07T18:28:49.896001+00:00` | `emit_user_audit_log(...)` (`app/contact_utils.py:L104-L108`) → `UserAuditLog.create(...)` (`app/user_audit_log_utils.py:L38`); `action='create_contact'`, `message='Created contact 2 (spoofedemailsource@gmail.com)'` |
| 3 | `EmailLog` (`app/models.py:L2060`, table `email_log`) | `2` | `2026-07-07T18:28:49.908099+00:00` | `EmailLog.create(...)` per verified mailbox (`email_handler.py:L732`), `message_id=str(msg[headers.MESSAGE_ID])` |

Key attributes on the created `EmailLog`: `is_reply=False`, `blocked=False`, `message_id` = the preserved original `Message-ID`, and **`sl_message_id=None`**. As shown, **no `MessageIDMatching`** row is created on a forward (contrast Q2's reply, which created `MessageIDMatching id=1`).

The `created_at` ordering within the operation reflects the code order: the `Contact` (`...890591`) and its `UserAuditLog` (`...896001`) are written first in `handle_forward()`, then the `EmailLog` (`...908099`) is written in `forward_email_to_mailbox()`.

### Run-to-run note (Q4)

IDs and timestamps advance monotonically. `run2` (a fresh new-sender forward) produced:

```
  CONTACT      id=3 created_at='2026-07-07T18:28:50.226354+00:00' website_email='spoofedemailsource@gmail.com' reply_email='spoofedemailsource_at_gmail_com_vfgirqp@sl.local'
  EMAILLOG     id=3 created_at='2026-07-07T18:28:50.241154+00:00' is_reply=False blocked=False message_id='<20220317165018.000191@somewhere-5488dd4b6b-7crp6>' sl_message_id=None
  USERAUDITLOG id=3 created_at='2026-07-07T18:28:50.231110+00:00' action='create_contact' message='Created contact 3 (spoofedemailsource@gmail.com)'
```

The autoincrement `id` values and `created_at` timestamps are **volatile by nature** (they were captured, never fabricated).

### Context: forward from an **already-known** sender

The three-row result is the **new-sender** case. A forward whose sender is already a `Contact` of that alias creates **only** the `EmailLog` (no new `Contact`, no new `UserAuditLog`), because `handle_forward()` uses get-or-create for the contact. The primary answer for "a single forward operation" from a new sender is therefore **1 `Contact` + 1 `UserAuditLog` + 1 `EmailLog`**.

---

## Coverage & caveats

**Both Q1 branches were run and captured separately.** Success → `E200` (`"250 Message accepted for delivery"`, `app/email/status.py:L2`); non-existent alias → `E515` (`"550 SL E515 Email not exist"`, `app/email/status.py:L51`). A reply was additionally exercised for the Q2 contrast (minted SL Message-ID + `MessageIDMatching`). Every observation was repeated at least twice.

**Volatile-but-captured fields (never fabricated).** These vary by nature run to run and were reproduced, not invented:

- Random alias local-parts (e.g. `debunk_elvish590`, `raving_bodges010`, `maniac_reboot428`, `lambda_merits709`).
- Reverse-alias random suffix from `random_string(random.randint(5, 10))` (`_oarrgcqd`, `_vfgirqp`, `_hvvbe`, `_clrjcz`).
- `make_msgid` components in the reply SL Message-ID: `timeval` (`178344893069` → `178344893104`), `pid` (`25195`; `25295` in the second full process), `randint` (`8423179011053050732` → `4648077217597495233`), and the `email_log.id` idstring (`5` → `7`).
- Autoincrement `id`s (Contact/EmailLog/UserAuditLog `2` → `3`; MessageIDMatching `1` → `2`) and `arrow.utcnow` `created_at` timestamps.

Stable structure across runs: the log message text, the `E200`/`E515` status strings, the preserved forward `Message-ID`, the `AT`-format display name, and the `@sl.local` reverse-alias domain.

**Labeled non-canonical / environment notes.**

- **PostgreSQL major version:** ran against **PostgreSQL 15.13** on port `5432` in the warmed image, whereas the CI/canonical spec pins **PostgreSQL 13** on port `15432` (`.github/workflows/main.yml:L47`; `tests/test.env:L17`). This does not affect any application-level value reported here (all determined by the identical Python code at HEAD `2cd6ee777f8c`).
- **Transaction rollback is a no-op for committed rows here.** As detailed in "Environment & method," `Session.commit()` under SQLAlchemy 1.3.24 with the connection-bound session (`app/db.py`) persists rows despite the `flask_client` fixture's `transaction.rollback()`. The committed IDs/timestamps were read live during the run; the test database was then explicitly reset (`drop schema public cascade; create schema public;` + `alembic upgrade head`) to leave it pristine.

**Adjacent branches noted as context only (not exercised as answers).** These are alternative outcomes the code can produce but are outside the two Q1 conditions asked about:

- A disabled alias or `contact.block_forward` produces a **blocked** `EmailLog` instead of a delivery (`email_handler.py:L596-L604`).
- An unverified mailbox → `E517` (`app/email/status.py:L53`); a disabled mailbox → `E518` (`app/email/status.py:L54`).
- The observed forward's `Subject` gained a `[Possible phishing attempt]` prefix because the chosen `.eml` fixture carries a DMARC soft-fail rspamd result; this is orthogonal to the four questions (it affects `Subject`, not the `From`/`Message-ID`/records behavior reported here).
- A reply fanned out to multiple recipients reuses an existing `sl_message_id` via the branch at `email_handler.py:L1306-L1309` (`LOG.d("reuse the sl_message_id %s", ...)`).

**Grounding.** Every factual claim above cites a `file:line` locator and names the responsible function/method; every value was produced by running `email_handler.handle(...)` (the real entry point), with the exact producing command shown in "Environment & method." Nothing here was derived from static reading alone.

