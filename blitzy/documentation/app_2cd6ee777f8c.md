# Runtime Investigation: "Inconsistent behavior when emails are forwarded through SimpleLogin aliases"

**Repository:** SimpleLogin (`app`) · **Investigated commit / source branch:** `app_2cd6ee777f8c` (`git HEAD 2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`)
**Deliverable:** this document (`blitzy/documentation/app_2cd6ee777f8c.md`).
**Posture:** strictly **read-only** against the product. The forward path was driven live through the **real, canonical** `aiosmtpd` SMTP entry point (`python email_handler.py`, port `20381`) using `swaks`; evidence was captured at three surfaces — the handler's stdout, the delivered `.eml` at a local SMTP sink, and the Postgres tables. Every value below is shown with the exact command that produced it, the complete unedited output, a `file:line` reference, and an explicit **[OBSERVED]** or **[INFERRED]** label.

---

## 1. The reported issue and the four questions

A production report describes *"inconsistent behavior when emails are forwarded through SimpleLogin aliases."* Four precise questions were asked:

- **Q1 — Log message text (and level)** emitted when (a) an email is **successfully forwarded** vs. (b) forwarding **fails because the target alias does not exist**.
- **Q2 — "SL Message-ID" vs. the original `Message-ID`**: which message-id(s) are involved in a forward and how they differ from the sender's original `Message-ID`.
- **Q3 — Transformed `From` header** of the forwarded message, including the reverse-alias (reply-email) address format and its **actual local-part length**.
- **Q4 — Database records created by a single forward**, with actual record IDs and timestamps, for a first send vs. a repeat send.

### 1.1 TL;DR of findings

- **Q1 [OBSERVED]:** Success ends with an **INFO** summary line carrying `250 Message accepted for delivery` (`E200`); a non-existent alias emits two **DEBUG** lines (`alias … not exist …` / `alias … cannot be created on-the-fly, return 550`) and an **INFO** summary carrying `550 SL E515 Email not exist` (`E515`).
- **Q2 [OBSERVED]:** A pure forward **preserves the sender's original `Message-ID` byte-for-byte** (delivered header == sent header). No `sl_message_id` is generated and **no `message_id_matching` row is created during a forward** — that identifier is a *reply-phase* artifact. A separate *internal log-tracing id* (a fresh `uuid4`) is stamped into every log line and **changes on every message**. These three distinct "message ids" are the crux of the perceived inconsistency.
- **Q3 [OBSERVED]:** The `From` header becomes `"hey at google.com" <hey_at_google_com_uyhxkb@sl.local>`. The reverse-alias here uses the **sender-included** format (`{sanitized-sender}_{random}`) — because the seeded user `john@wick.com` has `include_sender_in_reverse_alias = True` (the User model's Python-side default). The observed local part is **24 characters** with a **6-character** random suffix; across brand-new contacts the random suffix ranged 6–9 chars (`random.randint(5, 10)`). **This is not the prefix-less 20–50 char default** — reported exactly as observed.
- **Q4 [OBSERVED]:** A **first** forward from a new `(alias, sender)` pair creates exactly **three** rows — `Contact`, `UserAuditLog`, `EmailLog` — in that order. A **repeat** forward from the same sender→alias reuses the `Contact` and creates **only** a new `EmailLog`. `message_id_matching` gains **no** row in either case.
- **Root cause of "inconsistency" [INFERRED]:** the report most plausibly conflates (a) genuinely per-message-variable values (autoincrement `EmailLog.id`, `created_at`, the log-tracing `uuid4`, the random reverse-alias suffix on brand-new contacts) with (b) the fact that **Message-ID handling differs between the forward phase (original preserved) and the reply phase (original replaced by a generated `sl_message_id`)**. Neither is a defect; both are by design.

---

## 2. Environment & Methodology

### 2.1 Disciplines followed

- **Run first, then write.** All code paths were executed and captured *before* any answer sentence was written.
- **Canonical entry point only.** Messages entered through the real `aiosmtpd` listener (`Controller(MailHandler(), hostname="0.0.0.0", port=port)` — `email_handler.py:2383`, with the argparse default `port=20381` at `email_handler.py:2399`). No value in this report comes from a bypassing harness or a direct `handle_forward()` call. **No NON-CANONICAL value appears in this document.**
- **Reproduce, don't stabilize.** The same unchanged success input was replayed **3 times** (plus additional brand-new-sender sends) and the observed distribution is reported in §7.

### 2.2 Canonical runtime

| Component | Value | Source |
|-----------|-------|--------|
| Python | 3.10.18 | `pyproject.toml:61` → `python = "^3.10"` |
| Database | PostgreSQL 15 (port `15432`, DB `simplelogin`, user `myuser`) | provisioned per setup |
| Cache | Redis 7.0.15 (port `6379`) | provisioned per setup |
| SMTP sink | local `aiosmtpd` sink on `127.0.0.1:1025` (captures the forwarded `.eml`) | temporary, deleted afterward |
| Container | `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` | user-specified image |

Runtime confirmation:

```console
$ /app/venv/bin/python --version
Python 3.10.18
$ cd /app && git rev-parse HEAD
2cd6ee777f8c2d3531559588bcfb18627ffb5d2c
```

### 2.3 `.env` deltas from `example.env` (default local config a normal user would use)

Per `CONTRIBUTING.md` "Test sending email", the only deltas required to emit a real forward and capture it are:

| Setting | Value used | Why | `file:line` |
|---------|-----------|-----|-------------|
| `NOT_SEND_EMAIL` | **commented out** | **presence-based** flag `NOT_SEND_EMAIL = "NOT_SEND_EMAIL" in os.environ`; setting it to `false` is *still truthy*, so it must be removed entirely | `app/config.py:91`; `example.env:19` (`NOT_SEND_EMAIL=true`) |
| `POSTFIX_SERVER` | `localhost` | point delivery at the local sink (default is `240.0.0.1`) | `app/config.py:136`; `.env` |
| `POSTFIX_PORT` | `1025` | local sink port (default is `25`) | `app/config.py:149`; `example.env:154` (`# POSTFIX_PORT=1025`) |
| `EMAIL_DOMAIN` | `sl.local` | alias/reverse-alias domain | `app/config.py:92`; `example.env:22` |
| `DB_URI` | `postgresql://myuser:mypassword@localhost:15432/simplelogin` | app database | `.env` |

⚠️ **`NOT_SEND_EMAIL` is presence-based** — verified at runtime that the flag resolves to `False` so a genuine forward is emitted (not the `LOG.d`-only short-circuit in `app/mail_sender.py:129-137`):

```console
$ CONFIG=/app/.env /app/venv/bin/python -c "from app import config; \
  print('NOT_SEND_EMAIL =', config.NOT_SEND_EMAIL); \
  print('POSTFIX_SERVER =', config.POSTFIX_SERVER); \
  print('POSTFIX_PORT   =', config.POSTFIX_PORT); \
  print('EMAIL_DOMAIN   =', config.EMAIL_DOMAIN); \
  print('ENABLE_SPAM_ASSASSIN =', config.ENABLE_SPAM_ASSASSIN); \
  print('MAX_SPAM_SCORE =', config.MAX_SPAM_SCORE)"
NOT_SEND_EMAIL = False
POSTFIX_SERVER = localhost
POSTFIX_PORT   = 1025
EMAIL_DOMAIN   = sl.local
ENABLE_SPAM_ASSASSIN = False
MAX_SPAM_SCORE = 5.5
```

**[OBSERVED]** `NOT_SEND_EMAIL is False`, so the real send path `_send_to_smtp → SMTP(config.POSTFIX_SERVER, config.POSTFIX_PORT, …)` runs (`app/mail_sender.py:147-151`).
**[INFERRED]** `ENABLE_SPAM_ASSASSIN = False` (presence-based, `app/config.py:450`) means the SpamAssassin branch in `forward_email_to_mailbox` is skipped by default; no spam rejection appears in any captured log, confirming the forward path is not gated by spam here.

### 2.4 Seeding deterministic fixtures

```console
$ CONFIG=/app/.env /app/venv/bin/alembic upgrade head        # schema (head 32f25cbf12f6, 77 tables)
$ CONFIG=/app/.env FLASK_APP=wsgi:app /app/venv/bin/flask dummy-data
```

`flask dummy-data` (`server.py:490` → `dummy_data()` → `fake_data()`) creates user `john@wick.com` (`app/fake_data.py:45`) and aliases `e0/e1/e2@sl.local` (`app/fake_data.py:140-151`). **`e1@sl.local` (alias id 5) is the success target.**

```console
$ psql … -c "SELECT id,email FROM alias ORDER BY id;"
 id |                  email                  | user_id
----+-----------------------------------------+---------
  1 | simplelogin-newsletter.test838@sl.local |       1
  2 | test_test005@sl.local                   |       1
  3 | example@example.com                     |       1
  4 | e0@sl.local                             |       1
  5 | e1@sl.local                             |       1
  6 | e2@sl.local                             |       1
  ...
```

**Q4 pre-seed nuance [OBSERVED]:** `fake_data()` pre-seeds a `Contact` for `hey@google.com` (`reply_email=rep@sl.local`) but binds it to a **random** alias (`Alias.create_new_random(user)`, `app/fake_data.py:70`, `86-92`), i.e. `alias_id=2`, **not** `e1` (id 5). Contact uniqueness is `(alias_id, website_email)` (`app/contact_utils.py:85`), so the first forward `hey@google.com → e1` still creates a brand-new `Contact`:

```console
$ psql … -c "SELECT id,alias_id,website_email,reply_email FROM contact ORDER BY id;"   # BEFORE any forward
 id | alias_id | website_email  | reply_email
----+----------+----------------+--------------
  1 |        2 | hey@google.com | rep@sl.local
(1 row)
```

### 2.5 Canonical entry point + injector

```console
$ CONFIG=/app/.env /app/venv/bin/python email_handler.py     # aiosmtpd Controller on 0.0.0.0:20381
2026-07-13 17:02:03,729 - SL - INFO  - 836 - "/app/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-13 17:02:03,731 - SL - DEBUG - 836 - "/app/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

Injection used the documented recipe (`CONTRIBUTING.md` "Test sending email"):

```console
$ swaks --to e1@sl.local --from hey@google.com --server 127.0.0.1:20381 …
```

**Log-level legend** (`app/log.py:77-80`): `LOG.d`=**DEBUG**, `LOG.i`=**INFO**, `LOG.w`=**WARNING**, `LOG.e`=**ERROR**. Every handler line is prefixed by the per-message log-tracing id via the `%(message_id)s` field in the formatter (`app/log.py:14`), injected by `EmailHandlerFilter` (`app/log.py:28-37`). Logger level is DEBUG on a stdout handler (`app/log.py:41,51`).

---

## 3. Q1 — Success vs Non-Existent-Alias log messages

### 3.1 SUCCESS — full evidence

**Command (canonical injection):**

```console
$ swaks --to e1@sl.local --from hey@google.com --server 127.0.0.1:20381 \
        --header "Message-ID: <probe-run1@test.local>" \
        --header "Subject: Forward probe run1" --body "Hello from run 1"
```

**swaks client transcript (unedited, tail):**

```
 -> MAIL FROM:<hey@google.com>
<-  250 OK
 -> RCPT TO:<e1@sl.local>
<-  250 OK
 -> DATA
<-  354 End data with <CR><LF>.<CR><LF>
 ...
 -> .
<-  250 Message accepted for delivery
 -> QUIT
<-  221 Bye
=== Connection closed with remote host.
```

**Handler stdout for this message (unedited):**

```
2026-07-13 17:02:25,536 - SL - DEBUG - 836 - "/app/app/log.py:24" - set_message_id() -  - set message_id 75c0de02-f224-4223-840c-a4d9d0a9a078
2026-07-13 17:02:25,536 - SL - DEBUG - 836 - "/app/email_handler.py:2342" - _handle() - 75c0de02-f224-4223-840c-a4d9d0a9a078 - ====>=====>====>====>====>====>====>====>
2026-07-13 17:02:25,536 - SL - INFO - 836 - "/app/email_handler.py:2343" - _handle() - 75c0de02-f224-4223-840c-a4d9d0a9a078 - New message, mail from hey@google.com, rctp tos ['e1@sl.local']
2026-07-13 17:02:25,537 - SL - INFO - 836 - "/app/email_handler.py:1956" - handle() - 75c0de02-f224-4223-840c-a4d9d0a9a078 - Set CONTENT_TRANSFER_ENCODING
2026-07-13 17:02:25,537 - SL - DEBUG - 836 - "/app/email_handler.py:1963" - handle() - 75c0de02-f224-4223-840c-a4d9d0a9a078 - Cannot parse Postfix queue ID from None None
2026-07-13 17:02:25,676 - SL - DEBUG - 836 - "/app/email_handler.py:1980" - handle() - 75c0de02-f224-4223-840c-a4d9d0a9a078 - ==>> Handle mail_from:hey@google.com, rcpt_tos:['e1@sl.local'], header_from:hey@google.com, header_to:e1@sl.local, cc:None, reply-to:None, message_id:<20260713170225.000857@04e9f907815a>, client_ip:None, headers:[('Date', 'Mon, 13 Jul 2026 17:02:25 +0000'), ('To', 'e1@sl.local'), ('From', 'hey@google.com'), ('Subject', 'Forward probe run1'), ('Message-Id', '<20260713170225.000857@04e9f907815a>'), ('X-Mailer', 'swaks v20201014.0 jetmore.org/john/code/swaks/'), ('Message-ID', '<probe-run1@test.local>'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-13 17:02:25,680 - SL - DEBUG - 836 - "/app/email_handler.py:2202" - handle() - 75c0de02-f224-4223-840c-a4d9d0a9a078 - Forward phase hey@google.com(hey@google.com) -> e1@sl.local
2026-07-13 17:02:25,696 - SL - DEBUG - 836 - "/app/email_handler.py:580" - handle_forward() - 75c0de02-f224-4223-840c-a4d9d0a9a078 - Create or get contact for from_header:hey@google.com
2026-07-13 17:02:25,721 - SL - DEBUG - 836 - "/app/app/contact_utils.py:110" - create_contact() - 75c0de02-f224-4223-840c-a4d9d0a9a078 - Created contact <Contact 2 hey@google.com 5> for alias <Alias 5 e1@sl.local> with email hey@google.com invalid_email=False
2026-07-13 17:02:25,721 - SL - INFO - 836 - "/app/app/handler/dmarc.py:33" - apply_dmarc_policy_for_forward_phase() - 75c0de02-f224-4223-840c-a4d9d0a9a078 - DMARC check disabled
2026-07-13 17:02:25,729 - SL - DEBUG - 836 - "/app/email_handler.py:688" - forward_email_to_mailbox() - 75c0de02-f224-4223-840c-a4d9d0a9a078 - Forward <Contact 2 hey@google.com 5> -> <Alias 5 e1@sl.local> -> <Mailbox 1 john@wick.com>
2026-07-13 17:02:25,733 - SL - DEBUG - 836 - "/app/email_handler.py:740" - forward_email_to_mailbox() - 75c0de02-f224-4223-840c-a4d9d0a9a078 - Create <EmailLog 2> for <Contact 2 hey@google.com 5>, <User 1 John Wick john@wick.com>, <Mailbox 1 john@wick.com>
2026-07-13 17:02:25,738 - SL - DEBUG - 836 - "/app/email_handler.py:867" - forward_email_to_mailbox() - 75c0de02-f224-4223-840c-a4d9d0a9a078 - From header, new:"hey at google.com" <hey_at_google_com_uyhxkb@sl.local>, old:hey@google.com
2026-07-13 17:02:25,739 - SL - DEBUG - 836 - "/app/email_handler.py:893" - forward_email_to_mailbox() - 75c0de02-f224-4223-840c-a4d9d0a9a078 - Forward mail from hey@google.com to john@wick.com, mail_options:[], rcpt_options:[]
2026-07-13 17:02:25,740 - SL - DEBUG - 836 - "/app/app/mail_sender.py:163" - _send_to_smtp() - 75c0de02-f224-4223-840c-a4d9d0a9a078 - Sendmail mail_from:sl.lmycyibsfqqdemzygi3tqms5.eh3ne3oe6fasy@sl.local, rcpt_to:john@wick.com, header_from:"hey at google.com" <hey_at_google_com_uyhxkb@sl.local>, header_to:e1@sl.local, header_cc:None
2026-07-13 17:02:25,743 - SL - INFO - 836 - "/app/email_handler.py:2367" - _handle() - 75c0de02-f224-4223-840c-a4d9d0a9a078 - Finish mail_from hey@google.com, rcpt_tos ['e1@sl.local'], takes 0.20715069770812988 seconds with return code '250 Message accepted for delivery'<<===
```

**Extracted success log lines (each named, with level and `file:line`) [OBSERVED]:**

| Level | Message (rendered) | Emitter / `file:line` |
|-------|--------------------|-----------------------|
| DEBUG | `Forward phase hey@google.com(hey@google.com) -> e1@sl.local` | `handle()` — `email_handler.py:2202` (template `"Forward phase %s(%s) -> %s"`, L2202-2207) |
| DEBUG | `Created contact <Contact 2 hey@google.com 5> for alias <Alias 5 e1@sl.local> with email hey@google.com invalid_email=False` | `create_contact()` — `app/contact_utils.py:110` |
| DEBUG | `Create <EmailLog 2> for <Contact 2 hey@google.com 5>, <User 1 John Wick john@wick.com>, <Mailbox 1 john@wick.com>` | `forward_email_to_mailbox()` — `email_handler.py:740` (template `"Create %s for %s, %s, %s"`) |
| DEBUG | `From header, new:"hey at google.com" <hey_at_google_com_uyhxkb@sl.local>, old:hey@google.com` | `forward_email_to_mailbox()` — `email_handler.py:867` (template `"From header, new:%s, old:%s"`) |
| **INFO** | `Finish mail_from hey@google.com, rcpt_tos ['e1@sl.local'], takes 0.207… seconds with return code '250 Message accepted for delivery'<<===` | `_handle()` — `email_handler.py:2367` (template L2367-2373) |

The terminating SMTP status `250 Message accepted for delivery` is the constant **`E200`** — `app/email/status.py:2`:

```python
E200 = "250 Message accepted for delivery"
```

### 3.2 FAILURE — non-existent alias — full evidence

**Command:**

```console
$ swaks --to doesnotexist@sl.local --from hey@google.com --server 127.0.0.1:20381 \
        --header "Subject: Failure probe" --body "should 550"
```

**swaks client transcript (unedited, tail):**

```
 -> RCPT TO:<doesnotexist@sl.local>
<-  250 OK
 -> DATA
<-  354 End data with <CR><LF>.<CR><LF>
 ...
 -> .
<** 550 SL E515 Email not exist
 -> QUIT
<-  221 Bye
=== Connection closed with remote host.
```

**Handler stdout for this message (unedited):**

```
2026-07-13 17:04:23,434 - SL - DEBUG - 836 - "/app/app/log.py:24" - set_message_id() - 75c0de02-f224-4223-840c-a4d9d0a9a078 - set message_id 2d447f99-d076-438b-8292-eea02d1ed3fe
2026-07-13 17:04:23,434 - SL - DEBUG - 836 - "/app/email_handler.py:2342" - _handle() - 2d447f99-d076-438b-8292-eea02d1ed3fe - ====>=====>====>====>====>====>====>====>
2026-07-13 17:04:23,434 - SL - INFO - 836 - "/app/email_handler.py:2343" - _handle() - 2d447f99-d076-438b-8292-eea02d1ed3fe - New message, mail from hey@google.com, rctp tos ['doesnotexist@sl.local']
2026-07-13 17:04:23,435 - SL - INFO - 836 - "/app/email_handler.py:1956" - handle() - 2d447f99-d076-438b-8292-eea02d1ed3fe - Set CONTENT_TRANSFER_ENCODING
2026-07-13 17:04:23,435 - SL - DEBUG - 836 - "/app/email_handler.py:1963" - handle() - 2d447f99-d076-438b-8292-eea02d1ed3fe - Cannot parse Postfix queue ID from None None
2026-07-13 17:04:23,437 - SL - DEBUG - 836 - "/app/email_handler.py:1980" - handle() - 2d447f99-d076-438b-8292-eea02d1ed3fe - ==>> Handle mail_from:hey@google.com, rcpt_tos:['doesnotexist@sl.local'], header_from:hey@google.com, header_to:doesnotexist@sl.local, cc:None, reply-to:None, message_id:<20260713170423.000918@04e9f907815a>, client_ip:None, headers:[...], mail_options:[], rcpt_options:[]
2026-07-13 17:04:23,441 - SL - DEBUG - 836 - "/app/email_handler.py:2202" - handle() - 2d447f99-d076-438b-8292-eea02d1ed3fe - Forward phase hey@google.com(hey@google.com) -> doesnotexist@sl.local
2026-07-13 17:04:23,447 - SL - DEBUG - 836 - "/app/email_handler.py:545" - handle_forward() - 2d447f99-d076-438b-8292-eea02d1ed3fe - alias doesnotexist@sl.local not exist. Try to see if it can be created on the fly
2026-07-13 17:04:23,453 - SL - INFO - 836 - "/app/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - 2d447f99-d076-438b-8292-eea02d1ed3fe - Cannot auto-create custom domain alias for doesnotexist@sl.local because there's no custom domain for sl.local
2026-07-13 17:04:23,453 - SL - INFO - 836 - "/app/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - 2d447f99-d076-438b-8292-eea02d1ed3fe - Cannot auto-create doesnotexist@sl.local since it has no directory separator
2026-07-13 17:04:23,453 - SL - DEBUG - 836 - "/app/email_handler.py:551" - handle_forward() - 2d447f99-d076-438b-8292-eea02d1ed3fe - alias doesnotexist@sl.local cannot be created on-the-fly, return 550
2026-07-13 17:04:23,454 - SL - INFO - 836 - "/app/email_handler.py:2367" - _handle() - 2d447f99-d076-438b-8292-eea02d1ed3fe - Finish mail_from hey@google.com, rcpt_tos ['doesnotexist@sl.local'], takes 0.019916772842407227 seconds with return code '550 SL E515 Email not exist'<<===
```

**Extracted failure log lines (each named, with level and `file:line`) [OBSERVED]:**

| Level | Message (rendered) | Emitter / `file:line` |
|-------|--------------------|-----------------------|
| DEBUG | `alias doesnotexist@sl.local not exist. Try to see if it can be created on the fly` | `handle_forward()` — `email_handler.py:545` (template L545-548) |
| INFO | `Cannot auto-create custom domain alias for doesnotexist@sl.local because there's no custom domain for sl.local` | `check_if_alias_can_be_auto_created_for_custom_domain()` — `app/alias_utils.py:104` (auto-create attempt) |
| INFO | `Cannot auto-create doesnotexist@sl.local since it has no directory separator` | `check_if_alias_can_be_auto_created_for_a_directory()` — `app/alias_utils.py:165` (auto-create attempt) |
| DEBUG | `alias doesnotexist@sl.local cannot be created on-the-fly, return 550` | `handle_forward()` — `email_handler.py:551` |
| **INFO** | `Finish mail_from hey@google.com, rcpt_tos ['doesnotexist@sl.local'], takes 0.0199… seconds with return code '550 SL E515 Email not exist'<<===` | `_handle()` — `email_handler.py:2367` |

The path returns `[(False, status.E515)]` (`email_handler.py:555`); the terminating SMTP status `550 SL E515 Email not exist` is the constant **`E515`** — `app/email/status.py:51`:

```python
E515 = "550 SL E515 Email not exist"
```

**Q1 answer [OBSERVED]:** the *distinguishing* success line is the **INFO** `Finish … return code '250 Message accepted for delivery'` (`E200`); the *distinguishing* failure lines are the two **DEBUG** `handle_forward` lines (`… not exist …` / `… cannot be created on-the-fly, return 550`) followed by the **INFO** `Finish … return code '550 SL E515 Email not exist'` (`E515`). All other pipeline lines (routing, `Handle mail_from`, tracing-id) are common to both.


---

## 4. Q2 — "SL Message-ID" vs. the original `Message-ID`

This is the subtle question. There are **three distinct "message id" concepts** in play; conflating them is the most likely source of the "inconsistent behavior" perception.

### 4.1 Identifier #1 — the original `Message-ID` header (PRESERVED during a forward)

During the forward phase, `Message-ID` is in `headers_to_keep` (`email_handler.py:793-807`, with the comment `# do not delete original message id`):

```python
headers_to_keep = [
    headers.FROM, headers.TO, headers.CC, headers.SUBJECT, headers.DATE,
    # do not delete original message id
    headers.MESSAGE_ID,
    headers.REFERENCES, headers.IN_REPLY_TO, ...
]
```

The only rewrite of threading headers in the forward phase is `replace_sl_message_id_by_original_message_id(msg)` (called at `email_handler.py:860`; defined `email_handler.py:931-963`) — it touches **`In-Reply-To`** and **`References`** only, **never** `Message-ID`.

**[OBSERVED]** The delivered message's `Message-ID` equals the value that entered over SMTP. In run #1 I sent both a swaks-auto `Message-Id` and an explicit `Message-ID` header; both survived byte-for-byte in the delivered `.eml` captured at the sink:

```console
$ grep -iE "^Message-Id|^Message-ID" /tmp/sink/msg_2.eml          # run #1 delivered
Message-Id: <20260713170225.000857@04e9f907815a>
Message-ID: <probe-run1@test.local>
```

The `swaks` transcript shows exactly those two headers were sent:

```
 -> Message-Id: <20260713170225.000857@04e9f907815a>
 -> Message-ID: <probe-run1@test.local>
```

Runs #2 and #3 (swaks auto `Message-Id` only) show the same preservation:

```console
$ grep -i "^Message-Id" /tmp/sink/msg_3.eml    # run #2:  <20260713170450.000931@04e9f907815a>  (== sent)
$ grep -i "^Message-Id" /tmp/sink/msg_4.eml    # run #3:  <20260713170513.000949@04e9f907815a>  (== sent)
```

The `EmailLog.message_id` column stores this same original value (`message_id=str(msg[headers.MESSAGE_ID])`, `email_handler.py:737`) — see the `email_log` snapshot in §6.

**Conclusion #1 [OBSERVED]:** during a pure forward, the `Message-ID` header is **unchanged** — the delivered header is identical to the sender's original. It differs from the original by **zero** bytes.

### 4.2 Identifier #2 — the internal log-tracing id (a fresh `uuid4`, VARIABLE)

Every handler log line is prefixed by the `%(message_id)s` field (`app/log.py:14`). Because these runs arrive via `aiosmtpd` (no Postfix `Received` header with a queue id), `handle()` fails to parse a Postfix queue id and falls back to the `uuid4` generated in `_handle()`:

```python
# email_handler.py:1959-1963 (handle)          # email_handler.py:2339-2340 (_handle)
postfix_queue_id = get_queue_id(msg)            message_id = str(uuid.uuid4())
if postfix_queue_id:                            set_message_id(message_id)
    set_message_id(postfix_queue_id)
else:
    LOG.d("Cannot parse Postfix queue ID ...")
```

**[OBSERVED]** the "Cannot parse Postfix queue ID from None None" line (`email_handler.py:1963`) appears on every message, and the tracing id is a different `uuid4` each time:

| Run | Log-tracing id (from `set message_id …` / line prefix) |
|-----|--------------------------------------------------------|
| success #1 | `75c0de02-f224-4223-840c-a4d9d0a9a078` |
| failure    | `2d447f99-d076-438b-8292-eea02d1ed3fe` |
| success #2 | `687adbdb-d602-4b22-8a5d-b40af0df6fb8` |
| success #3 | `39dfbd66-6d6d-45c0-b516-4b9ac17b97cf` |

**Conclusion #2 [OBSERVED]:** this id is an **internal tracing id, not an email header**, and it is **VARIABLE run-to-run** (a fresh `uuid4` per message). It never appears in the delivered `.eml`.

### 4.3 Identifier #3 — the real `sl_message_id` (a REPLY-PHASE artifact, NOT written on a forward)

The "SL Message-ID" proper is produced only in the **reply** path by `replace_original_message_id()`:

```python
# email_handler.py:1311-1313
sl_message_id = make_msgid(str(email_log.id), get_email_domain_part(alias.email))
# email_handler.py:1316-1321 → persisted in message_id_matching; email_log.sl_message_id set at L1341
# email_handler.py:1338-1339 → del msg[MESSAGE_ID]; msg[MESSAGE_ID] = sl_message_id   (REPLACES Message-ID — reply only)
```

**[OBSERVED]** a pure forward creates **no** `message_id_matching` row — the table stayed empty across all forwards (before, between, and after every send):

```console
$ psql … -c "SELECT count(*) FROM message_id_matching;"
 count
-------
     0
```

**Conclusion #3 [OBSERVED]:** the generated `sl_message_id` (and its `message_id_matching` row and the `del msg[MESSAGE_ID]; msg[MESSAGE_ID] = sl_message_id` replacement) belong to the **reply phase**, not the forward. A forward neither generates nor stores an `sl_message_id`.

### 4.4 Q2 synthesis — how they "differ"

| Concept | Where it lives | On a forward | Value class |
|---------|----------------|--------------|-------------|
| **Original `Message-ID`** | email header (delivered `.eml`) | **preserved unchanged** | equal to sender's original |
| **Log-tracing id** | log lines only (`%(message_id)s`) | fresh `uuid4` per message | **VARIABLE** |
| **`sl_message_id`** | `message_id_matching` + `email_log.sl_message_id` + reply's `Message-ID` | **not created** | reply-phase only |

**[INFERRED]** The perception of "inconsistent behavior" most plausibly arises because **Message-ID handling differs between phases**: on the **forward** the original `Message-ID` is preserved, whereas on a **reply** the original is *replaced* by a generated `sl_message_id` (`email_handler.py:1338-1339`) and recorded in `message_id_matching`. An observer comparing a forwarded message's headers to a replied message's headers (or to log lines carrying the per-message `uuid4`) would see three different "ids" and could reasonably conclude the behavior is inconsistent, when in fact each id is deterministic *for its phase*.


---

## 5. Q3 — Transformed `From` header and reverse-alias (reply-email) format

### 5.1 The delivered `From` header

The forward replaces the `From` header with `contact.new_addr()` (`email_handler.py:864-867`):

```python
old_from_header = msg[headers.FROM]
new_from_header = contact.new_addr()
add_or_replace_header(msg, "From", new_from_header)
LOG.d("From header, new:%s, old:%s", new_from_header, old_from_header)
```

**[OBSERVED]** delivered `.eml` (run #1), raw:

```console
$ grep -iE "^From:|^X-SimpleLogin" /tmp/sink/msg_2.eml
X-SimpleLogin-Type: Forward
X-SimpleLogin-EmailLog-ID: 2
X-SimpleLogin-Envelope-From: hey@google.com
X-SimpleLogin-Original-From: hey@google.com
X-SimpleLogin-Envelope-To: e1@sl.local
From: "hey at google.com" <hey_at_google_com_uyhxkb@sl.local>
```

So the transformed header is:

```
From: "hey at google.com" <hey_at_google_com_uyhxkb@sl.local>
```

### 5.2 Anatomy of the value

**Display name** — `Contact.new_addr()` (`app/models.py:2008-2046`) uses the default `SenderFormatEnum.AT` (`app/models.py:2019`, `sender_format = user.sender_format if user else SenderFormatEnum.AT.value`; enum defined at `app/models.py:203-208`, `AT = 0`). The AT branch (`app/models.py:2028-2034`) computes `website_email.replace("@", " at ")` → for `hey@google.com` this yields the display name **`hey at google.com`**. The final address is `sl_formataddr((new_name, self.reply_email))` (`app/models.py:2045`).

> Note the correct enum name is **`SenderFormatEnum.AT`** (`app/models.py:203-208`), not `ContactFormatEnum.AT`.

**Reverse-alias (reply-email)** — the address-part `hey_at_google_com_uyhxkb@sl.local` is `Contact.reply_email`, generated by `generate_reply_email()` (`app/email_utils.py:1103-1153`).

### 5.3 ⚠️ Which branch produced it, and the ACTUAL local-part length

The observed local part is **not** the prefix-less random string one might expect from the function's default. It is the **sender-included** format `{sanitized-sender}_{random}`. The reason is a **user setting**, verified at runtime:

```console
$ psql … -tAc "SELECT id,email,include_sender_in_reverse_alias FROM users ORDER BY id;"
1|john@wick.com|t
2|winston@continental.com|t
```

`john@wick.com` has `include_sender_in_reverse_alias = t (TRUE)`. **[INFERRED, grounded]** this is the User model's Python-side default: `include_sender_in_reverse_alias = sa.Column(sa.Boolean, default=True, nullable=False, server_default="0")` (`app/models.py:455-457`). `fake_data()` creates the user through the ORM (`User.create(...)`, `app/fake_data.py:44`) without setting the field, so the Python `default=True` applies (the `server_default="0"` would only apply to non-ORM inserts). `fake_data.py` does **not** set this field (confirmed by grep — no match).

Consequently `generate_reply_email()` takes the **include-sender** branch, not the default `else` branch:

```python
# app/email_utils.py
include_sender_in_reverse_alias = False                       # L1112 (initial)
if user.include_sender_in_reverse_alias is not None:          # L1116  ← True for john
    include_sender_in_reverse_alias = user.include_sender_in_reverse_alias   # L1117 → True
...
if include_sender_in_reverse_alias and contact_email:         # L1137  ← taken
    random_length = random.randint(5, 10)                     # L1138  ← 5–10, not 20–50
    reply_email = f"{contact_email}_{random_string(random_length)}@{reply_domain}"   # L1142
else:
    random_length = random.randint(20, 50)                    # L1145  (NOT taken here)
    reply_email = f"{random_string(random_length)}@{reply_domain}"                    # L1148
```

`contact_email` is the sanitized sender (`hey@google.com` → lower/ascii, `@`→`_at_`, `.`→`_`, alphanumeric) = `hey_at_google_com`. `random_string` uses `string.ascii_lowercase` (`app/utils.py:41-47`). The legacy `ra+`/`reply+` prefix is commented out (`app/email_utils.py:1141,1147`).

**Actual local-part length — measured directly [OBSERVED]:**

```console
$ psql … -tAc "SELECT id,website_email,reply_email, length(split_part(reply_email,'@',1)) AS localpart_len FROM contact ORDER BY id;"
1	hey@google.com	rep@sl.local	3
2	hey@google.com	hey_at_google_com_uyhxkb@sl.local	24
3	alice@example.org	alice_at_example_org_lsjfxkj@sl.local	28
4	bob@example.org	bob_at_example_org_nhjekthri@sl.local	28
5	carol@example.org	carol_at_example_org_hmugjhrg@sl.local	29
```

For `hey@google.com → e1`, the reverse-alias local part **`hey_at_google_com_uyhxkb`** is exactly **24 characters**, of which the random suffix `uyhxkb` is **6 characters** (within `random.randint(5, 10)`). Across brand-new contacts the random suffix length varied — 6 (`uyhxkb`), 7 (`lsjfxkj`), 9 (`nhjekthri`), 8 (`hmugjhrg`) — i.e. the 5–10 range, and total local-part length tracks the sender length.

### 5.4 Cross-check against the database

**[OBSERVED]** the delivered `From`'s reverse-alias exactly equals `Contact.reply_email` in Postgres for the new contact (id 2):

```console
$ psql … -tAc "SELECT id,alias_id,website_email,reply_email FROM contact WHERE reply_email LIKE 'hey_at_google_com%';"
2|5|hey@google.com|hey_at_google_com_uyhxkb@sl.local
```

Delivered `From` address-part `hey_at_google_com_uyhxkb@sl.local` == `contact.reply_email` — **match**. Membership as a reverse-alias is confirmed by `is_reverse_alias()` via `Contact.get_by(reply_email=address)` (`app/email_utils.py:1156-1160`).

**Q3 answer [OBSERVED]:** the forwarded `From` is `"hey at google.com" <hey_at_google_com_uyhxkb@sl.local>` — an `AT`-format display name (`SenderFormatEnum.AT`) plus a **sender-included** reverse-alias whose local part is **24 chars** (6-char random suffix), because the seeded user's `include_sender_in_reverse_alias` is `True`. The value matches `Contact.reply_email` exactly.


---

## 6. Q4 — Database records created by a single forward

Snapshots were taken with a temporary read-only script (`/tmp/snap.sql`, deleted afterward) over the four relevant tables:

```console
$ PGPASSWORD=mypassword psql -h localhost -p 15432 -U myuser -d simplelogin -f /tmp/snap.sql
# SELECTs id + key columns + created_at from contact, email_log, user_audit_log, message_id_matching
```

### 6.1 BEFORE any forward (post-seed baseline)

```
 contact | email_log | user_audit_log | message_id_matching
---------+-----------+----------------+---------------------
       1 |         1 |              0 |                   0
```

The single pre-existing `contact` (id 1) and `email_log` (id 1) are the `fake_data` pre-seed on the **random** alias (id 2), unrelated to `e1` — see §2.4.

### 6.2 AFTER the FIRST forward (`hey@google.com → e1@sl.local`, run #1)

```
=== contact ===
 id | alias_id | website_email  |            reply_email            | name |         created_at
----+----------+----------------+-----------------------------------+------+----------------------------
  1 |        2 | hey@google.com | rep@sl.local                      |      | 2026-07-13 16:49:39.346843
  2 |        5 | hey@google.com | hey_at_google_com_uyhxkb@sl.local |      | 2026-07-13 17:02:25.711751
=== email_log ===
 id | contact_id | user_id | mailbox_id | alias_id |              message_id              |         created_at
----+------------+---------+------------+----------+--------------------------------------+----------------------------
  1 |          1 |       1 |            |        2 |                                      | 2026-07-13 16:49:39.352415
  2 |          2 |       1 |          1 |        5 | <20260713170225.000857@04e9f907815a> | 2026-07-13 17:02:25.730863
=== user_audit_log ===
 id | user_id |  user_email   |     action     |              message               |         created_at
----+---------+---------------+----------------+------------------------------------+----------------------------
  1 |       1 | john@wick.com | create_contact | Created contact 2 (hey@google.com) | 2026-07-13 17:02:25.718282
=== message_id_matching ===
 (0 rows)
=== counts ===
 contact | email_log | user_audit_log | message_id_matching
---------+-----------+----------------+---------------------
       2 |         2 |              1 |                   0
```

**The FIRST forward created exactly THREE rows [OBSERVED]:**

| Record | id | Key columns | created_at | Emitter / `file:line` |
|--------|----|-----------  |-----------|-----------------------|
| `Contact` | **2** | `alias_id=5` (e1), `website_email=hey@google.com`, `reply_email=hey_at_google_com_uyhxkb@sl.local` | `2026-07-13 17:02:25.711751` | `Contact.create(...)` — `app/contact_utils.py:92-103` |
| `UserAuditLog` | **1** | `user_id=1`, `user_email=john@wick.com`, `action=create_contact`, `message="Created contact 2 (hey@google.com)"` | `2026-07-13 17:02:25.718282` | `emit_user_audit_log(CreateContact)` — `app/contact_utils.py:104-109` → `UserAuditLog.create` `app/user_audit_log_utils.py:35-38` |
| `EmailLog` | **2** | `contact_id=2`, `user_id=1`, `mailbox_id=1`, `alias_id=5`, `message_id=<20260713170225.000857@04e9f907815a>` | `2026-07-13 17:02:25.730863` | `EmailLog.create(...)` — `email_handler.py:732-739` |
| `message_id_matching` | — | (none created) | — | reply-phase only (`email_handler.py:1316-1321`) |

The creation **order** is visible in the timestamps: `Contact` (…711) → `UserAuditLog` (…718) → `EmailLog` (…730), all within ~19 ms — matching the code order (contact + audit inside `create_contact`, then `EmailLog` back in `forward_email_to_mailbox`).

### 6.3 AFTER a REPEAT forward (same sender→alias, runs #2 and #3)

Counts after run #2, then run #3:

```
run #2:  contact=2 | email_log=3 | user_audit_log=1 | message_id_matching=0
run #3:  contact=2 | email_log=4 | user_audit_log=1 | message_id_matching=0
```

The handler log for run #2/#3 shows the contact is **reused** (there is no `Created contact` line; only `Create <EmailLog 3>` / `Create <EmailLog 4>` for the existing `<Contact 2 …>`):

```
… email_handler.py:740 … Create <EmailLog 3> for <Contact 2 hey@google.com 5>, <User 1 …>, <Mailbox 1 …>
… email_handler.py:740 … Create <EmailLog 4> for <Contact 2 hey@google.com 5>, <User 1 …>, <Mailbox 1 …>
```

`email_log` rows added by the repeats:

```
 id | contact_id | user_id | mailbox_id | alias_id |              message_id              |         created_at
----+------------+---------+------------+----------+--------------------------------------+----------------------------
  3 |          2 |       1 |          1 |        5 | <20260713170450.000931@04e9f907815a> | 2026-07-13 17:04:50.819776
  4 |          2 |       1 |          1 |        5 | <20260713170513.000949@04e9f907815a> | 2026-07-13 17:05:13.11832
```

**The REPEAT forward created only ONE row [OBSERVED]:** a new `EmailLog` (id 3, then id 4), both `contact_id=2` (reused). **No** new `Contact`, **no** new `UserAuditLog`, **no** `message_id_matching`. This is enforced by the contact uniqueness on `(alias_id, website_email)` (`app/contact_utils.py:85`): `create_contact` finds the existing contact and returns it without inserting (`__update_contact_if_needed`).

### 6.4 Provenance of `id` and `created_at`

Every record inherits an autoincrement primary key `id` and an `arrow`-typed `created_at` from `ModelMixin` (`app/models.py:62-152`). **[OBSERVED]** ids increment monotonically (`EmailLog` 2 → 3 → 4) and `created_at` advances every send — both are therefore **VARIABLE** across runs.

**Q4 answer [OBSERVED]:** a first forward from a new `(alias, sender)` pair creates **`Contact` + `UserAuditLog` + `EmailLog`** (ids 2 / 1 / 2 with the timestamps above); a repeat forward from the same sender→alias creates **only `EmailLog`** (ids 3, 4); `message_id_matching` is never written by a forward.


---

## 7. Run-to-run stability (directly addressing "inconsistent behavior")

The **same unchanged** success input (`swaks --to e1@sl.local --from hey@google.com --server 127.0.0.1:20381`) was replayed **3 times** (runs #1–#3). The table classifies each observed value as **STABLE** or **VARIABLE** with the concrete evidence.

| Observed value | Run #1 | Run #2 | Run #3 | Class | Why |
|----------------|--------|--------|--------|-------|-----|
| Success log templates | identical | identical | identical | **STABLE** | fixed format strings (`email_handler.py:2202,740,867,2367`) |
| Terminating SMTP status | `250 …accepted…` | same | same | **STABLE** | constant `E200` (`app/email/status.py:2`) |
| `From` header format | `"hey at google.com" <…>` | same | same | **STABLE** | `SenderFormatEnum.AT` (`app/models.py:2019,2028`) |
| Reverse-alias **value** | `hey_at_google_com_uyhxkb@sl.local` | same | same | **STABLE** | `Contact` reused on repeat sends (uniqueness `app/contact_utils.py:85`) |
| `EmailLog.id` | 2 | 3 | 4 | **VARIABLE** | autoincrement PK (`ModelMixin`, `app/models.py:62-152`) |
| `EmailLog.created_at` | 17:02:25.730 | 17:04:50.819 | 17:05:13.118 | **VARIABLE** | `arrow.utcnow()` per insert |
| Log-tracing id (`uuid4`) | `75c0de02…` | `687adbdb…` | `39dfbd66…` | **VARIABLE** | fresh `uuid4` (`email_handler.py:2339`) |
| Original `Message-ID` (delivered) | `<…170225.000857@…>` | `<…170450.000931@…>` | `<…170513.000949@…>` | preserved-but-input-varies | swaks mints a new `Message-Id` each send; forward preserves whatever arrives |
| Envelope return-path (VERP) | `sl.lmycyibsfqqdemzygi3tqms5.eh3ne3oe6fasy@sl.local` | `sl.lmycyibtfqqdemzygi3tqnc5.g2t6hphhbc57o@sl.local` | (varies) | **VARIABLE** | per-`EmailLog` bounce-handling return path (`_send_to_smtp`, `app/mail_sender.py:163`) |
| `message_id_matching` rows | 0 | 0 | 0 | **STABLE** | not written on a forward |

Additional variability across **brand-new** contacts (different senders → e1) — the reverse-alias **random suffix** changes each time a new `Contact` is created:

| Sender | reverse-alias | random suffix (len) |
|--------|--------------|---------------------|
| `hey@google.com` | `hey_at_google_com_uyhxkb@sl.local` | `uyhxkb` (6) |
| `alice@example.org` | `alice_at_example_org_lsjfxkj@sl.local` | `lsjfxkj` (7) |
| `bob@example.org` | `bob_at_example_org_nhjekthri@sl.local` | `nhjekthri` (9) |
| `carol@example.org` | `carol_at_example_org_hmugjhrg@sl.local` | `hmugjhrg` (8) |

### 7.1 Conclusion about the report

**[OBSERVED]** For an *identical* input, the *forward behavior* is deterministic in every user-visible way that matters: the log templates, SMTP status, `From`/reverse-alias format, and (on contact reuse) the reverse-alias value are all stable. The only things that change run-to-run are the intrinsically per-record values: `EmailLog.id`, `created_at`, the internal `uuid4` tracing id, the VERP return-path, and — when a brand-new `Contact` is minted — the random reverse-alias suffix.

**[INFERRED]** The reported "inconsistent behavior" is best explained as a combination of:
1. **Expected per-record variation** — autoincrement ids, timestamps, the log-tracing `uuid4`, and the random reverse-alias suffix for a *new* sender. None of these indicate a defect.
2. **Phase-dependent Message-ID handling** — the forward phase *preserves* the original `Message-ID`, while the reply phase *replaces* it with a generated `sl_message_id` and records a `message_id_matching` row (§4.3–4.4). An observer comparing forwarded vs. replied headers (or reading the per-message `uuid4` in logs) would see three different "message ids" and could describe the system as inconsistent, though each id is deterministic within its own phase.

No genuine run-to-run inconsistency in the *forward path itself* was observed across the identical replays; the distribution above is reported exactly as measured, not smoothed.

---

## 8. Coverage pass

- **Q1 (success log)** — ✅ full swaks transcript + full handler stdout (§3.1); distinguishing **INFO** `Finish … '250 Message accepted for delivery'` (`email_handler.py:2367`, `E200` `app/email/status.py:2`) plus the DEBUG routing/create/from lines.
- **Q1 (non-existent-alias log)** — ✅ full transcript + stdout (§3.2); two DEBUG `handle_forward` lines (`email_handler.py:545`, `:551`) + **INFO** `Finish … '550 SL E515 Email not exist'` (`E515` `app/email/status.py:51`).
- **Q2 (SL Message-ID vs original)** — ✅ three ids disambiguated (§4): original **preserved** (delivered == sent), tracing **uuid4 variable**, `sl_message_id` **reply-phase only** (`message_id_matching` count 0 on forward); inferred forward-vs-reply difference stated.
- **Q3 (From + reverse-alias format + length)** — ✅ delivered `From` raw (§5.1); `SenderFormatEnum.AT` display name; observed reverse-alias **sender-included** branch with **exact 24-char** local part / 6-char suffix; branch reason (`include_sender_in_reverse_alias=True`) grounded; cross-checked vs `Contact.reply_email`.
- **Q4 (DB records + ids + timestamps)** — ✅ before/after snapshots (§6); first send = **Contact(2)+UserAuditLog(1)+EmailLog(2)** with timestamps; repeat = **EmailLog only (3,4)**; `message_id_matching` unchanged; fake_data pre-seed nuance covered.
- **Run-to-run** — ✅ 3 identical replays + brand-new-sender variability; STABLE/VARIABLE table (§7).
- **Named items** — E200/E515 strings ✅; `handle`/`handle_forward`/`forward_email_to_mailbox`/`_handle`/`create_contact`/`generate_reply_email`/`new_addr`/`replace_original_message_id` all named with `file:line` ✅; three CRITICAL NUANCES honored (observed reverse-alias length reported, not assumed 5–10; `SenderFormatEnum.AT`; `NOT_SEND_EMAIL` commented/`False`) ✅.
- **Canonical discipline** — ✅ every value came through the `aiosmtpd` entry point via swaks; **no NON-CANONICAL value appears** in this document.

---

## 9. Cleanup confirmation

The investigation is read-only against the product. After capturing evidence, the temporary observation artifacts and the spun-up container/services are removed; the repository's only lasting change is this document.

Commands used to clean up (executed during finalization):

```console
# temp observation scripts / captured .eml dumps (inside the container)
$ rm -f /tmp/blitzy_adhoc_test_sink.py /tmp/snap.sql; rm -rf /tmp/sink
# host-side temp evidence copy (never committed)
$ rm -rf blitzy_evidence_tmp
# tear down the investigation container (Postgres + Redis + SMTP sink live inside it)
$ docker rm -f sl_inv
```

Expected final repository state — the only change is the new document:

```console
$ git status --porcelain
?? blitzy/documentation/app_2cd6ee777f8c.md
```

(The SimpleLogin source tree is left byte-for-byte unchanged; `.env` is gitignored and was never committed. The `spamassassin_utils.py` file inside the *container image* was restored to its canonical `import re2 as re` before the runs, and `pyre2==0.3.10` was installed into the container venv so the canonical source imports cleanly — neither touches the deliverable repository.)

