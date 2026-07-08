# SimpleLogin — How an inbound reply resolves to a `Contact` and a forwarding destination, and why a reply can be delivered to the WRONG user

> **Investigation type:** run-first, read-only. Every factual claim below is grounded either in a specific `file:line` reference or in **complete, unedited runtime output** captured from the canonical Docker runtime (`sl-work`, image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`). The exact command that produced each block appears immediately above that block. No source file was modified; the temporary observation scripts were removed afterward (see **Repository verification & cleanup** at the end).

---

## TL;DR — direct answer

- **R1 — reply-email derivation.** `handle_reply()` takes the reply address **verbatim from the SMTP envelope recipient** — `reply_email = rcpt_to` [email_handler.py:972] — then normalizes it with `normalize_reply_email()` [email_handler.py:984] [app/email_validation.py:25]. Observed: `rcpt_to='sender_at_external_test_ycnjnjlzcw@sl.local'`, and `normalize_reply_email(R) == R` (True).
- **R2 — contact identification.** The reply email is looked up with `Contact.get_by(reply_email=reply_email)` [email_handler.py:986], which is `Session.query(cls).filter_by(**kw).first()` [app/models.py:83-84] — a **`LIMIT 1` with NO `ORDER BY`** (confirmed both from the emitted SQL and from `EXPLAIN`).
- **R3 — forwarding destination.** The resolved **user** is `contact.alias.user` and the resolved **alias** is `contact.alias` [email_handler.py:994,1004]. The **mailbox** returned by `get_mailbox_from_mail_from(mail_from, alias)` [email_handler.py:1019,1364] is the **authorized *sending* mailbox** — it is used for the SPF/authorization decision and is recorded as `EmailLog.mailbox_id` (audit). It is **NOT** the outbound recipient. The **actual outbound recipient** of the forwarded reply is **`contact.website_email`** — the message is sent `from alias.email to contact.website_email` [email_handler.py:1212] via `sl_sendmail(..., contact.website_email, ...)` [email_handler.py:1224-1230].
- **R4 — repeated events.** With a single `Contact`, three consecutive **full `handle()` reply events** on the same input all resolve to the same contact/user/alias and each writes a distinct `EmailLog`; 10 repeated `get_by` lookups all return the same user.
- **R5 — divergence.** Because `reply_email` is **indexed but NOT unique** [app/models.py:1899] [migrations/versions/2021_071310_78403c7b8089_.py:22], a second `Contact` for a **different user** can carry the **same** `reply_email`. After the physical (heap) order changes, the unordered `.first()` **flips** to the other user — observed within the process, across three independent OS processes, and confirmed by `EXPLAIN`.
- **R5(c) — the honest same-input outcome.** Re-running the **identical unchanged reply** (`mail_from=userA.email`) through the real `handle()` **after** the flip returns **`250 SL E214 Unauthorized for using reverse alias`** — User A's own reply is **denied**, because the resolved alias now belongs to User B and User A is not an authorized sender for it. A cross-user *delivery* (reply routed to User B's contact) is reproducible only when the resolved alias has the real per-alias flag `disable_email_spoofing_check=True` (clearly labeled below).
- **R6 — race / uniqueness.** The creation path performs a check-then-insert (`available_sl_email()` [app/models.py:1425] → `Contact.create()`), and the only DB uniqueness on `contact` is `uq_contact(alias_id, website_email)` [app/models.py:1875]. Two concurrent workers both observe the reply email as "free" and both insert it (TOCTOU); the `IntegrityError` fallback in `create_contact()` re-fetches by `(alias_id, website_email)` only [app/contact_utils.py:113-118] and therefore cannot prevent a duplicate `reply_email`.
- **R7 — observed values.** `R='sender_at_external_test_ycnjnjlzcw@sl.local'`; original contact `id=206 alias_id=990 user_id=595`; duplicate contact `id=207 alias_id=992 user_id=596`; post-flip resolution `id=207 user_id=596`; cross-user delivery `EmailLog id=377 user_id=596 mailbox_id=711 outbound-recipient=other@external.test`.

**Root cause in one line:** a non-unique `reply_email` column + an unordered `.first()` lookup means the contact (hence user/alias/mailbox and the outbound `contact.website_email`) chosen for a reply is **whatever row PostgreSQL returns first**, which is not stable once duplicate rows exist.

---

## Environment, build & invocation

All observations were produced inside the **canonical** runtime — the user-specified Docker container `sl-work` (image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`). The working tree is bind-mounted at `/app`; the app is configured by `CONFIG=/app/tests/test.env` (`EMAIL_DOMAIN=sl.local` [tests/test.env:8], `NOT_SEND_EMAIL=true` [tests/test.env:7] so no real SMTP egress occurs, `DB_URI=postgresql://test:test@localhost:15432/test` [tests/test.env:17], `DMARC_CHECK_ENABLED=true` [tests/test.env:66]). The observation scripts obtain a live Flask application context via `server.create_app()` and exercise the **real** entry point `email_handler.handle()` (which dispatches to `handle_reply()` when `is_reverse_alias(rcpt_to)` is true [email_handler.py:2195,2199]).

> **Note on the default (non-canonical) shell.** The plain login shell of this workspace is Python 3.13 / SQLAlchemy 2.x with **no PostgreSQL** — it is *not* the runtime this app targets. Every command below is therefore run through `docker exec sl-work …` so the evidence reflects the canonical Python 3.10 / SQLAlchemy 1.3.24 / PostgreSQL 15 stack.

**Command (runtime & schema probe):**

```
docker exec sl-work bash -lc 'cd /app && echo "python:" && python --version && \
  echo "libs:" && python -c "import sqlalchemy,psycopg2,aiosmtpd;print(\"sqlalchemy\",sqlalchemy.__version__,\"psycopg2\",psycopg2.__version__.split(\" \")[0],\"aiosmtpd\",aiosmtpd.__version__)" && \
  echo ".version:" && cat .version && \
  echo "branch:" && git rev-parse --abbrev-ref HEAD && \
  echo "base-app-commit:" && git rev-parse HEAD && \
  echo "postgres:" && pg_lsclusters'
```

**Complete, unedited output:**

```
python:
Python 3.10.18
libs:
sqlalchemy 1.3.24 psycopg2 2.9.3 aiosmtpd 1.4.2
.version:
dev
branch:
blitzy-fa8339d5-38ba-4a32-a6f3-20c43c289bf8
base-app-commit:
2cd6ee777f8c2d3531559588bcfb18627ffb5d2c
postgres:
Ver Cluster Port  Status Owner    Data directory              Log file
15  main    15432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

The reply-resolution subsystem consists of these files (all read-only reference targets; only the answer document is created):

| File | Role | Key anchors |
|------|------|-------------|
| `email_handler.py` | SMTP inbound processor; `handle()` dispatch + `handle_reply()` | `handle_reply` def [:966]; dispatch `is_reverse_alias(rcpt_to)` [:2195] → `handle_reply(...)` [:2199]; `get_mailbox_from_mail_from` [:1364]; outbound `sl_sendmail(..., contact.website_email, ...)` [:1224-1230] |
| `app/models.py` | ORM `ModelMixin`, `Contact`, `available_sl_email()` | `get_by` = `filter_by(**kw).first()` [:83-84]; `uq_contact(alias_id, website_email)` [:1875]; `reply_email` column `index=True` (not unique) [:1899]; `available_sl_email` [:1425] |
| `app/email_validation.py` | Reply-email normalization | `_ALLOWED_CHARS` [:9]; `normalize_reply_email` [:25]; non-allowed char → `_` [:33-34] |
| `app/email_utils.py` | Reverse-alias generation / detection | `generate_reply_email` [:1103]; `is_reverse_alias` [:1156] |
| `app/contact_utils.py` | Contact creation + `IntegrityError` fallback | `create_contact` [:42]; fallback re-fetch by `(alias_id, website_email)` [:113-118] |
| `app/email/status.py` | SMTP status constants | `E201` [:3]; `E214` [:22]; `E501` [:38]; `E502` [:39]; `E503` [:40]; `E504` [:41]; `E506` [:43] |
| `migrations/versions/2021_071310_78403c7b8089_.py` | Creates the `reply_email` index | `create_index(op.f('ix_contact_reply_email'), 'contact', ['reply_email'], unique=False)` [:22] |

**The single main observation script** drives the whole R1→R7 narrative in one internally consistent process (one app boot, one DB session, monotonic IDs). Its complete output is reproduced section-by-section below; every R-section labeled *"(segment of the `obs_repro.py` run)"* is a contiguous slice of the one output produced by:

```
docker exec sl-work bash -lc 'cd /app && python /tmp/obs_repro.py'
```

The **boot + seed** prologue of that run (User A → Mailbox → Alias → Contact) is:

```
load config file /app/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/bcaniaitaynxnqptrltv
Upload files to local dir
>>> init logging <<<
2026-07-08 06:25:26,094 - SL - DEBUG - 7916 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-08 06:25:27,333 - SL - DEBUG - 7916 - "/app/init_app.py:42" - add_sl_domains() -  - d1.test is already a SL domain
2026-07-08 06:25:27,334 - SL - DEBUG - 7916 - "/app/init_app.py:42" - add_sl_domains() -  - d2.test is already a SL domain
2026-07-08 06:25:27,335 - SL - DEBUG - 7916 - "/app/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-08 06:25:27,336 - SL - DEBUG - 7916 - "/app/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
########## SEED (User A -> Mailbox -> Alias -> Contact) ##########
2026-07-08 06:25:27,649 - SL - INFO - 7916 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 06:25:27,668 - SL - DEBUG - 7916 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email encode_cosmic143@sl.local
2026-07-08 06:25:27,676 - SL - INFO - 7916 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 06:25:27,703 - SL - DEBUG - 7916 - "/app/app/contact_utils.py:110" - create_contact() -  - Created contact <Contact 206 sender@external.test 990> for alias <Alias 990 encode_cosmic143@sl.local> with email sender@external.test invalid_email=False
userA.id = 595 email = user_noqs7byvg0@mailbox.test
aliasA.id = 990 email = encode_cosmic143@sl.local
create_contact -> created = True error = None
contactA.id = 206 alias_id = 990 user_id = 595 website_email = sender@external.test
R (contactA.reply_email) = 'sender_at_external_test_ycnjnjlzcw@sl.local'
aliasA.mailbox.id = 710 email = user_noqs7byvg0@mailbox.test

```

This establishes the canonical entities used throughout: **User A** `id=595`, its default **Mailbox** `id=710`, **Alias A** `id=990` (`encode_cosmic143@sl.local`), **Contact A** `id=206` (`website_email=sender@external.test`), and the generated reverse alias **`R='sender_at_external_test_ycnjnjlzcw@sl.local'`** (`contactA.reply_email`, produced by `generate_reply_email()` [app/email_utils.py:1103]).

---

## R1 — How the reply email is derived from an inbound reply

**Direct answer:** the reply email is **not parsed out of any header** — it is taken **verbatim from the SMTP envelope recipient** (`RCPT TO`). In `handle_reply()` the first substantive statement is `reply_email = rcpt_to` [email_handler.py:972]; after a domain guard it is normalized in place, `reply_email = normalize_reply_email(reply_email)` [email_handler.py:984], which lower-cases and replaces every character outside `_ALLOWED_CHARS` [app/email_validation.py:9] with `_` [app/email_validation.py:33-34]. For a well-formed reverse alias, normalization is a no-op.

The reply payload driven through the real entry point:

- `envelope.mail_from = 'user_noqs7byvg0@mailbox.test'` (User A's mailbox address — an authorized sender)
- `envelope.rcpt_tos  = ['sender_at_external_test_ycnjnjlzcw@sl.local']` (= `R`, the reverse alias)

**Command:** `docker exec sl-work bash -lc 'cd /app && python /tmp/obs_repro.py'` — segment of that run:

```
########## R1: reply-email derivation (REAL entry point handle()) ##########
envelope.mail_from = 'user_noqs7byvg0@mailbox.test'
envelope.rcpt_tos  = ['sender_at_external_test_ycnjnjlzcw@sl.local']
2026-07-08 06:25:27,707 - SL - INFO - 7916 - "/app/email_handler.py:1956" - handle() -  - Set CONTENT_TRANSFER_ENCODING
2026-07-08 06:25:27,707 - SL - DEBUG - 7916 - "/app/email_handler.py:1963" - handle() -  - Cannot parse Postfix queue ID from None None
2026-07-08 06:25:27,709 - SL - DEBUG - 7916 - "/app/email_handler.py:1980" - handle() -  - ==>> Handle mail_from:user_noqs7byvg0@mailbox.test, rcpt_tos:['sender_at_external_test_ycnjnjlzcw@sl.local'], header_from:user_noqs7byvg0@mailbox.test, header_to:sender_at_external_test_ycnjnjlzcw@sl.local, cc:None, reply-to:None, message_id:<obs-r1@sl.local>, client_ip:None, headers:[('From', 'user_noqs7byvg0@mailbox.test'), ('To', 'sender_at_external_test_ycnjnjlzcw@sl.local'), ('Message-ID', '<obs-r1@sl.local>'), ('Subject', 'obs'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-08 06:25:27,712 - SL - DEBUG - 7916 - "/app/email_handler.py:2196" - handle() -  - Reply phase user_noqs7byvg0@mailbox.test(user_noqs7byvg0@mailbox.test) -> sender_at_external_test_ycnjnjlzcw@sl.local
2026-07-08 06:25:27,714 - SL - INFO - 7916 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() -  - DMARC check disabled
2026-07-08 06:25:27,718 - SL - DEBUG - 7916 - "/app/email_handler.py:1051" - handle_reply() -  - Create <EmailLog 372> for <Contact 206 sender@external.test 990>, <User 595 Test User user_noqs7byvg0@mailbox.test>, <Mailbox 710 user_noqs7byvg0@mailbox.test>
2026-07-08 06:25:27,724 - SL - DEBUG - 7916 - "/app/email_handler.py:1171" - handle_reply() -  - From header is encode_cosmic143@sl.local
2026-07-08 06:25:27,726 - SL - DEBUG - 7916 - "/app/email_handler.py:380" - replace_header_when_reply() -  - Replace To header, old: sender_at_external_test_ycnjnjlzcw@sl.local, new: sender@external.test
2026-07-08 06:25:27,726 - SL - DEBUG - 7916 - "/app/email_handler.py:383" - replace_header_when_reply() -  - delete the Cc header. Old value None
2026-07-08 06:25:27,728 - SL - DEBUG - 7916 - "/app/email_handler.py:1314" - replace_original_message_id() -  - create a new sl_message_id <178349192772.7916.16018191297407854388.372@sl.local>
2026-07-08 06:25:27,731 - SL - WARNING - 7916 - "/app/email_handler.py:1206" - handle_reply() -  - missing date header, add one
2026-07-08 06:25:27,734 - SL - DEBUG - 7916 - "/app/email_handler.py:1212" - handle_reply() -  - send email from encode_cosmic143@sl.local to sender@external.test, mail_options:[],rcpt_options:[]
2026-07-08 06:25:27,737 - SL - DEBUG - 7916 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'obs', from 'encode_cosmic143@sl.local' to 'sender@external.test'
email_handler.handle(env, msg) RETURNED: '250 Message accepted for delivery'
normalize_reply_email(R) = 'sender_at_external_test_ycnjnjlzcw@sl.local' | equal to R: True

---- R1 normalization collision (normalize_reply_email) ----
'ab#cd@sl.local' -> 'ab_cd@sl.local'
'ab$cd@sl.local' -> 'ab_cd@sl.local'
'ab cd@sl.local' -> 'ab_cd@sl.local'

```

**Reading the output (cause → effect):**
- `handle()` logs `Reply phase … -> sender_at_external_test_ycnjnjlzcw@sl.local` [email_handler.py:2196] — dispatch entered `handle_reply()` because `is_reverse_alias(R)` is true [email_handler.py:2195] [app/email_utils.py:1156].
- The derived value equals the envelope recipient exactly, and `normalize_reply_email(R) == R` (**True**) — for a canonical reverse alias there is no rewriting.
- The final `handle()` return is **`'250 Message accepted for delivery'`**, and the outbound send is logged `send email from encode_cosmic143@sl.local to sender@external.test` [email_handler.py:1212] — i.e. **to `contact.website_email`**, established here for the R3 discussion.

**Sibling variant — normalization collision (why derivation is lossy).** The three inputs `ab#cd@…`, `ab$cd@…`, and `ab cd@…` all normalize to the **same** `ab_cd@sl.local` (last three lines above). Because `#`, `$`, and space are all outside `_ALLOWED_CHARS`, each maps to `_` [app/email_validation.py:33-34]. Two distinct raw envelope recipients can therefore collapse to one `reply_email` string — an independent, second way a lookup on `reply_email` can point at the "wrong" contact.

---

## R2 — How the derived reply email identifies the `Contact`

**Direct answer:** identification is a single call, `contact = Contact.get_by(reply_email=reply_email)` [email_handler.py:986]. `ModelMixin.get_by` is defined as `return cls.filter_by(**kwargs).first()` [app/models.py:83-84]; `.first()` compiles to a `LIMIT 1` query with **no `ORDER BY`**. When two rows share the reply email, SQL does not promise *which* row is returned.

**Command:** `docker exec sl-work bash -lc 'cd /app && python /tmp/obs_repro.py'` — segment of that run (the SQL was captured live with a SQLAlchemy `before_cursor_execute` listener):

```
########## R2: contact identification by reply_email ##########
Contact.get_by(reply_email=R) -> id = 206 alias_id = 990 user_id = 595
---- R2 actual emitted SQL (before_cursor_execute listener) ----
SQL: SELECT contact.id AS contact_id, contact.created_at AS contact_created_at, contact.updated_at AS contact_updated_at, contact.user_id AS contact_user_id, contact.alias_id AS contact_alias_id, contact.name AS contact_name, contact.website_email AS contact_website_email, contact.website_from AS contact_website_from, contact.reply_email AS contact_reply_email, contact.is_cc AS contact_is_cc, contact.pgp_public_key AS contact_pgp_public_key, contact.pgp_finger_print AS contact_pgp_finger_print, contact.mail_from AS contact_mail_from, contact.invalid_email AS contact_invalid_email, contact.block_forward AS contact_block_forward, contact.automatic_created AS contact_automatic_created, contact.flags AS contact_flags 
FROM contact 
WHERE contact.reply_email = %(reply_email_1)s 
 LIMIT %(param_1)s
PARAMS: {'reply_email_1': 'sender_at_external_test_ycnjnjlzcw@sl.local', 'param_1': 1}
CONTAINS 'ORDER BY': False
CONTAINS 'LIMIT': True

```

**Reading the output:** the resolved contact is `id=206 alias_id=990 user_id=595` (User A). The emitted SQL filters `WHERE contact.reply_email = %(reply_email_1)s` and appends ` LIMIT %(param_1)s` with `param_1 = 1`; the programmatic checks confirm **`CONTAINS 'ORDER BY': False`** and **`CONTAINS 'LIMIT': True`**. This is the mechanical origin of the non-determinism: a `LIMIT 1` with no tie-break.

---

## R3 — Which user / alias / mailbox (and actual recipient) is chosen

**Direct answer — four distinct things, do not conflate them:**

1. **Resolved alias** = `contact.alias` [email_handler.py:994].
2. **Resolved user** = `alias.user` [email_handler.py:1004]. This is the account the reply is attributed to.
3. **Selected mailbox** = `get_mailbox_from_mail_from(mail_from, alias)` [email_handler.py:1019] (def [email_handler.py:1364]). This is the **authorized *sending* mailbox** — it exists to answer *"is the envelope `mail_from` allowed to send through this alias?"*, it drives the SPF/authorization decision, and it is stored as **`EmailLog.mailbox_id`** for audit. If `mail_from` is not an authorized address for the alias, this returns **`None`** and the handler rejects with E214.
4. **Actual outbound recipient** = **`contact.website_email`** — the message is transmitted `from alias.email to contact.website_email` [email_handler.py:1212] via `sl_sendmail(..., contact.website_email, ...)` [email_handler.py:1224-1230]. **The reply is delivered to the contact's real external address, never "to a mailbox."**

**Command:** `docker exec sl-work bash -lc 'cd /app && python /tmp/obs_repro.py'` — segment of that run:

```
########## R3: forwarding-destination selection ##########
contact.alias  -> id = 990 email = encode_cosmic143@sl.local (email_handler.py:994)
alias.user     -> id = 595 email = user_noqs7byvg0@mailbox.test (email_handler.py:1004)
get_mailbox_from_mail_from(AUTHORIZED mail_from=userA.email) -> id=710 email=user_noqs7byvg0@mailbox.test
get_mailbox_from_mail_from(UNAUTHORIZED mail_from=unauthorized@gmail.com) -> None
2026-07-08 06:25:27,744 - SL - INFO - 7916 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() -  - DMARC check disabled
2026-07-08 06:25:27,746 - SL - DEBUG - 7916 - "/app/email_handler.py:1051" - handle_reply() -  - Create <EmailLog 373> for <Contact 206 sender@external.test 990>, <User 595 Test User user_noqs7byvg0@mailbox.test>, <Mailbox 710 user_noqs7byvg0@mailbox.test>
2026-07-08 06:25:27,750 - SL - DEBUG - 7916 - "/app/email_handler.py:1171" - handle_reply() -  - From header is encode_cosmic143@sl.local
2026-07-08 06:25:27,751 - SL - DEBUG - 7916 - "/app/email_handler.py:380" - replace_header_when_reply() -  - Replace To header, old: sender_at_external_test_ycnjnjlzcw@sl.local, new: sender@external.test
2026-07-08 06:25:27,751 - SL - DEBUG - 7916 - "/app/email_handler.py:383" - replace_header_when_reply() -  - delete the Cc header. Old value None
2026-07-08 06:25:27,753 - SL - DEBUG - 7916 - "/app/email_handler.py:1314" - replace_original_message_id() -  - create a new sl_message_id <178349192775.7916.967414439118933684.373@sl.local>
2026-07-08 06:25:27,756 - SL - WARNING - 7916 - "/app/email_handler.py:1206" - handle_reply() -  - missing date header, add one
2026-07-08 06:25:27,758 - SL - DEBUG - 7916 - "/app/email_handler.py:1212" - handle_reply() -  - send email from encode_cosmic143@sl.local to sender@external.test, mail_options:[],rcpt_options:[]
2026-07-08 06:25:27,761 - SL - DEBUG - 7916 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'obs', from 'encode_cosmic143@sl.local' to 'sender@external.test'
handle_reply(envelope, msg, R) direct -> (True, '250 Message accepted for delivery')
EmailLog -> id=373 contact_id=206 alias_id=990 user_id=595 mailbox_id=710 is_reply=True
SELECTED mailbox is the AUTHORIZED SENDER + EmailLog.mailbox_id audit; OUTBOUND recipient is contact.website_email = 'sender@external.test'

```

**Reading the output:**
- `contact.alias` → alias `990` (`encode_cosmic143@sl.local`); `alias.user` → user `595`.
- `get_mailbox_from_mail_from(AUTHORIZED mail_from=userA.email)` → mailbox `id=710`; the same call with an **unauthorized** `mail_from` → **`None`** (this `None` is exactly what produces E214 in R5(c) and the guarded-exit matrix).
- The direct `handle_reply(envelope, msg, R)` returns `(True, '250 Message accepted for delivery')` and writes `EmailLog id=373 contact_id=206 alias_id=990 user_id=595 mailbox_id=710 is_reply=True`.
- The send log confirms `send email from encode_cosmic143@sl.local to sender@external.test` — the **outbound recipient is `contact.website_email='sender@external.test'`**, *not* mailbox 710. Mailbox 710 is only the authorized sender / SPF subject / `EmailLog.mailbox_id` audit field, as the final printed line states explicitly.

**Why this matters for the "wrong user" question:** the party who actually *receives* the forwarded reply is `contact.website_email`. So when the lookup resolves to a different contact (R5), the reply is not merely "attributed" to another user — it is **transmitted to that other contact's external email address**, which is the concrete cross-user information-exposure mechanism.

---

## R4 — Behavior across multiple reply events over time (single contact, before any duplicate)

**Direct answer:** while exactly one `Contact` carries `R`, the behavior is fully stable. Three consecutive **full `handle()` reply events** on the identical input each dispatch into `handle_reply()`, resolve the same `contact.id=206 / user_id=595 / alias_id=990`, select mailbox `710`, and write a distinct sequential `EmailLog` (`374`, `375`, `376`). Ten repeated `Contact.get_by(reply_email=R)` lookups all return user `595`. There is no drift — the divergence in R5 is a *consequence of a second row existing*, not of repetition per se.

**Command:** `docker exec sl-work bash -lc 'cd /app && python /tmp/obs_repro.py'` — segment of that run (three complete `handle()` events, unedited):

```
########## R4: repeated FULL handle() reply events (SINGLE contact, before duplicates) ##########
rows WHERE reply_email=R (ctid,id,alias_id,user_id): [('(1,49)', 206, 990, 595)]
2026-07-08 06:25:27,767 - SL - INFO - 7916 - "/app/email_handler.py:1956" - handle() -  - Set CONTENT_TRANSFER_ENCODING
2026-07-08 06:25:27,767 - SL - DEBUG - 7916 - "/app/email_handler.py:1963" - handle() -  - Cannot parse Postfix queue ID from None None
2026-07-08 06:25:27,768 - SL - DEBUG - 7916 - "/app/email_handler.py:1980" - handle() -  - ==>> Handle mail_from:user_noqs7byvg0@mailbox.test, rcpt_tos:['sender_at_external_test_ycnjnjlzcw@sl.local'], header_from:user_noqs7byvg0@mailbox.test, header_to:sender_at_external_test_ycnjnjlzcw@sl.local, cc:None, reply-to:None, message_id:<obs-r4-1@sl.local>, client_ip:None, headers:[('From', 'user_noqs7byvg0@mailbox.test'), ('To', 'sender_at_external_test_ycnjnjlzcw@sl.local'), ('Message-ID', '<obs-r4-1@sl.local>'), ('Subject', 'obs'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-08 06:25:27,771 - SL - DEBUG - 7916 - "/app/email_handler.py:2196" - handle() -  - Reply phase user_noqs7byvg0@mailbox.test(user_noqs7byvg0@mailbox.test) -> sender_at_external_test_ycnjnjlzcw@sl.local
2026-07-08 06:25:27,773 - SL - INFO - 7916 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() -  - DMARC check disabled
2026-07-08 06:25:27,775 - SL - DEBUG - 7916 - "/app/email_handler.py:1051" - handle_reply() -  - Create <EmailLog 374> for <Contact 206 sender@external.test 990>, <User 595 Test User user_noqs7byvg0@mailbox.test>, <Mailbox 710 user_noqs7byvg0@mailbox.test>
2026-07-08 06:25:27,779 - SL - DEBUG - 7916 - "/app/email_handler.py:1171" - handle_reply() -  - From header is encode_cosmic143@sl.local
2026-07-08 06:25:27,781 - SL - DEBUG - 7916 - "/app/email_handler.py:380" - replace_header_when_reply() -  - Replace To header, old: sender_at_external_test_ycnjnjlzcw@sl.local, new: sender@external.test
2026-07-08 06:25:27,781 - SL - DEBUG - 7916 - "/app/email_handler.py:383" - replace_header_when_reply() -  - delete the Cc header. Old value None
2026-07-08 06:25:27,782 - SL - DEBUG - 7916 - "/app/email_handler.py:1314" - replace_original_message_id() -  - create a new sl_message_id <178349192778.7916.7121224385491028202.374@sl.local>
2026-07-08 06:25:27,785 - SL - WARNING - 7916 - "/app/email_handler.py:1206" - handle_reply() -  - missing date header, add one
2026-07-08 06:25:27,787 - SL - DEBUG - 7916 - "/app/email_handler.py:1212" - handle_reply() -  - send email from encode_cosmic143@sl.local to sender@external.test, mail_options:[],rcpt_options:[]
2026-07-08 06:25:27,790 - SL - DEBUG - 7916 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'obs', from 'encode_cosmic143@sl.local' to 'sender@external.test'
  event 1: handle() -> '250 Message accepted for delivery' | resolved contact.id=206 user_id=595 alias_id=990 | EmailLog id=374 user_id=595 mailbox_id=710
2026-07-08 06:25:27,796 - SL - INFO - 7916 - "/app/email_handler.py:1956" - handle() -  - Set CONTENT_TRANSFER_ENCODING
2026-07-08 06:25:27,796 - SL - DEBUG - 7916 - "/app/email_handler.py:1963" - handle() -  - Cannot parse Postfix queue ID from None None
2026-07-08 06:25:27,798 - SL - DEBUG - 7916 - "/app/email_handler.py:1980" - handle() -  - ==>> Handle mail_from:user_noqs7byvg0@mailbox.test, rcpt_tos:['sender_at_external_test_ycnjnjlzcw@sl.local'], header_from:user_noqs7byvg0@mailbox.test, header_to:sender_at_external_test_ycnjnjlzcw@sl.local, cc:None, reply-to:None, message_id:<obs-r4-2@sl.local>, client_ip:None, headers:[('From', 'user_noqs7byvg0@mailbox.test'), ('To', 'sender_at_external_test_ycnjnjlzcw@sl.local'), ('Message-ID', '<obs-r4-2@sl.local>'), ('Subject', 'obs'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-08 06:25:27,801 - SL - DEBUG - 7916 - "/app/email_handler.py:2196" - handle() -  - Reply phase user_noqs7byvg0@mailbox.test(user_noqs7byvg0@mailbox.test) -> sender_at_external_test_ycnjnjlzcw@sl.local
2026-07-08 06:25:27,803 - SL - INFO - 7916 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() -  - DMARC check disabled
2026-07-08 06:25:27,805 - SL - DEBUG - 7916 - "/app/email_handler.py:1051" - handle_reply() -  - Create <EmailLog 375> for <Contact 206 sender@external.test 990>, <User 595 Test User user_noqs7byvg0@mailbox.test>, <Mailbox 710 user_noqs7byvg0@mailbox.test>
2026-07-08 06:25:27,810 - SL - DEBUG - 7916 - "/app/email_handler.py:1171" - handle_reply() -  - From header is encode_cosmic143@sl.local
2026-07-08 06:25:27,811 - SL - DEBUG - 7916 - "/app/email_handler.py:380" - replace_header_when_reply() -  - Replace To header, old: sender_at_external_test_ycnjnjlzcw@sl.local, new: sender@external.test
2026-07-08 06:25:27,812 - SL - DEBUG - 7916 - "/app/email_handler.py:383" - replace_header_when_reply() -  - delete the Cc header. Old value None
2026-07-08 06:25:27,813 - SL - DEBUG - 7916 - "/app/email_handler.py:1314" - replace_original_message_id() -  - create a new sl_message_id <178349192781.7916.4718085369288259671.375@sl.local>
2026-07-08 06:25:27,817 - SL - WARNING - 7916 - "/app/email_handler.py:1206" - handle_reply() -  - missing date header, add one
2026-07-08 06:25:27,820 - SL - DEBUG - 7916 - "/app/email_handler.py:1212" - handle_reply() -  - send email from encode_cosmic143@sl.local to sender@external.test, mail_options:[],rcpt_options:[]
2026-07-08 06:25:27,823 - SL - DEBUG - 7916 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'obs', from 'encode_cosmic143@sl.local' to 'sender@external.test'
  event 2: handle() -> '250 Message accepted for delivery' | resolved contact.id=206 user_id=595 alias_id=990 | EmailLog id=375 user_id=595 mailbox_id=710
2026-07-08 06:25:27,829 - SL - INFO - 7916 - "/app/email_handler.py:1956" - handle() -  - Set CONTENT_TRANSFER_ENCODING
2026-07-08 06:25:27,829 - SL - DEBUG - 7916 - "/app/email_handler.py:1963" - handle() -  - Cannot parse Postfix queue ID from None None
2026-07-08 06:25:27,830 - SL - DEBUG - 7916 - "/app/email_handler.py:1980" - handle() -  - ==>> Handle mail_from:user_noqs7byvg0@mailbox.test, rcpt_tos:['sender_at_external_test_ycnjnjlzcw@sl.local'], header_from:user_noqs7byvg0@mailbox.test, header_to:sender_at_external_test_ycnjnjlzcw@sl.local, cc:None, reply-to:None, message_id:<obs-r4-3@sl.local>, client_ip:None, headers:[('From', 'user_noqs7byvg0@mailbox.test'), ('To', 'sender_at_external_test_ycnjnjlzcw@sl.local'), ('Message-ID', '<obs-r4-3@sl.local>'), ('Subject', 'obs'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-08 06:25:27,834 - SL - DEBUG - 7916 - "/app/email_handler.py:2196" - handle() -  - Reply phase user_noqs7byvg0@mailbox.test(user_noqs7byvg0@mailbox.test) -> sender_at_external_test_ycnjnjlzcw@sl.local
2026-07-08 06:25:27,836 - SL - INFO - 7916 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() -  - DMARC check disabled
2026-07-08 06:25:27,838 - SL - DEBUG - 7916 - "/app/email_handler.py:1051" - handle_reply() -  - Create <EmailLog 376> for <Contact 206 sender@external.test 990>, <User 595 Test User user_noqs7byvg0@mailbox.test>, <Mailbox 710 user_noqs7byvg0@mailbox.test>
2026-07-08 06:25:27,844 - SL - DEBUG - 7916 - "/app/email_handler.py:1171" - handle_reply() -  - From header is encode_cosmic143@sl.local
2026-07-08 06:25:27,845 - SL - DEBUG - 7916 - "/app/email_handler.py:380" - replace_header_when_reply() -  - Replace To header, old: sender_at_external_test_ycnjnjlzcw@sl.local, new: sender@external.test
2026-07-08 06:25:27,846 - SL - DEBUG - 7916 - "/app/email_handler.py:383" - replace_header_when_reply() -  - delete the Cc header. Old value None
2026-07-08 06:25:27,847 - SL - DEBUG - 7916 - "/app/email_handler.py:1314" - replace_original_message_id() -  - create a new sl_message_id <178349192784.7916.689244614737364620.376@sl.local>
2026-07-08 06:25:27,851 - SL - WARNING - 7916 - "/app/email_handler.py:1206" - handle_reply() -  - missing date header, add one
2026-07-08 06:25:27,856 - SL - DEBUG - 7916 - "/app/email_handler.py:1212" - handle_reply() -  - send email from encode_cosmic143@sl.local to sender@external.test, mail_options:[],rcpt_options:[]
2026-07-08 06:25:27,858 - SL - DEBUG - 7916 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'obs', from 'encode_cosmic143@sl.local' to 'sender@external.test'
  event 3: handle() -> '250 Message accepted for delivery' | resolved contact.id=206 user_id=595 alias_id=990 | EmailLog id=376 user_id=595 mailbox_id=710
  repeated lookups (10 identical Contact.get_by): [595, 595, 595, 595, 595, 595, 595, 595, 595, 595]

```

**Reading the output:**
- The pre-check confirms a single physical row: `rows WHERE reply_email=R (ctid,id,alias_id,user_id): [('(1,49)', 206, 990, 595)]`.
- **event 1/2/3**: each `handle()` returns `'250 Message accepted for delivery'`; each resolves `contact.id=206 user_id=595 alias_id=990`; each writes `EmailLog id=374 / 375 / 376` with `user_id=595 mailbox_id=710`; each logs the outbound `send email from encode_cosmic143@sl.local to sender@external.test` (recipient = `contact.website_email`).
- **repeated lookups**: `10 identical Contact.get_by` → `[595, 595, 595, 595, 595, 595, 595, 595, 595, 595]` — unanimous.

(The `EXPLAIN` in R5(a) shows that the plan carries no `ORDER BY`; with a single matching row the unordered `LIMIT 1` is trivially stable, which is why R4 is deterministic and R5 is not.)

---

## R5 — Divergence of a single reply email

### R5(a) — the same reply email resolves to DIFFERENT contacts over time

**Direct answer: yes.** Once a second `Contact` for a **different user** carries the same `reply_email` (permitted because the column is non-unique [app/models.py:1899]), a change in physical row order flips the unordered `.first()` to the other user.

**Command:** `docker exec sl-work bash -lc 'cd /app && python /tmp/obs_repro.py'` — segment of that run:

```
########## R5(a) + R5(c): insert duplicate for DIFFERENT user, reorder, re-resolve ##########
2026-07-08 06:25:28,126 - SL - INFO - 7916 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 06:25:28,140 - SL - DEBUG - 7916 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email cuddle_squirm393@sl.local
2026-07-08 06:25:28,147 - SL - INFO - 7916 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
userB.id = 596 email = user_6xdpa4pon0@mailbox.test
aliasB.id = 992 email = cuddle_squirm393@sl.local
BEFORE duplicate, get_by(reply_email=R) -> contact.id=206 user_id=595
Contact.create(user_id=596, alias_id=992, website_email='other@external.test', reply_email=R) -> SUCCESS contact_B.id=207 (NO IntegrityError => reply_email NOT unique)
rows WHERE reply_email=R now: [('(1,49)', 206, 990, 595), ('(1,50)', 207, 992, 596)]
get_by AFTER insert (heap unchanged) -> contact.id=206 user_id=595
FLIP: deleted Contact 206; reinserted A2 id=208 (same userA=595 / aliasA=990)
rows WHERE reply_email=R after reorder: [('(1,50)', 207, 992, 596), ('(1,51)', 208, 990, 595)]
get_by AFTER flip -> contact.id=207 user_id=596 alias_id=992 website_email=other@external.test
  alias-owner of R's original reverse alias = userA (id=595); resolved user now = 596 => WRONG USER if 596 != 595
raw SQL 'SELECT id,user_id FROM contact WHERE reply_email=R LIMIT 1' (no ORDER BY) -> id=207 user_id=596

```

**Reading the output (before → intermediate → after):**
- **BEFORE**: with only User A's row, `get_by(reply_email=R)` → `contact.id=206 user_id=595`.
- **INTERMEDIATE**: `Contact.create(user_id=596, alias_id=992, website_email='other@external.test', reply_email=R)` **SUCCEEDS** as `contact_B.id=207` with **NO `IntegrityError`** — direct proof that `reply_email` is not unique. Rows are now `[('(1,49)', 206, 990, 595), ('(1,50)', 207, 992, 596)]`. Immediately after the insert (heap order unchanged) `get_by` still returns `206/595` — insertion alone does not flip it.
- **AFTER (heap reorder)**: deleting Contact 206 and reinserting User A's row as `id=208` changes the physical order to `[('(1,50)', 207, 992, 596), ('(1,51)', 208, 990, 595)]` — User B's row `207` is now physically first. `get_by` **flips** to `contact.id=207 user_id=596 alias_id=992 website_email=other@external.test` (**User B**), even though `R` was minted for User A's alias. The raw `SELECT id,user_id FROM contact WHERE reply_email=R LIMIT 1` (no `ORDER BY`) agrees: `id=207 user_id=596`.

**Cross-process confirmation (this is not a per-session cache artifact).** Three independent OS processes, each booting the app fresh and running the lookup once, all resolve to User B:

**Command:**

```
docker exec sl-work bash -lc 'cd /app && for i in 1 2 3; do python /tmp/obs_proc.py; done'
```

**Complete, unedited output:**

```
load config file /app/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/nuequbfemldfemxgwetf
Upload files to local dir
>>> init logging <<<
2026-07-08 06:28:50,674 - SL - DEBUG - 8009 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
PROC pid=8009 Contact.get_by(reply_email='sender_at_external_test_ycnjnjlzcw@sl.local') -> contact.id=207 user_id=596 alias_id=992
load config file /app/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/iscpaxjafbqxlukhzrog
Upload files to local dir
>>> init logging <<<
2026-07-08 06:28:52,846 - SL - DEBUG - 8023 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
PROC pid=8023 Contact.get_by(reply_email='sender_at_external_test_ycnjnjlzcw@sl.local') -> contact.id=207 user_id=596 alias_id=992
load config file /app/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/fxxaqsrxfofvbysliquj
Upload files to local dir
>>> init logging <<<
2026-07-08 06:28:54,915 - SL - DEBUG - 8036 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
PROC pid=8036 Contact.get_by(reply_email='sender_at_external_test_ycnjnjlzcw@sl.local') -> contact.id=207 user_id=596 alias_id=992
```

Three separate pids (`8009`, `8023`, `8036`) each return `contact.id=207 user_id=596 alias_id=992` — the resolution now favors User B for *every* fresh connection, because it is decided by physical heap order, not by session state.

**The planner confirms there is no ordering.** The lookup's `LIMIT 1` has no `ORDER BY` under either access path:

**Command (default plan — sequential scan):**

```
docker exec sl-work bash -lc "PGPASSWORD=test psql -h localhost -p 15432 -U test -d test -c \"EXPLAIN SELECT id FROM contact WHERE reply_email = 'sender_at_external_test_ycnjnjlzcw@sl.local' LIMIT 1;\""
```

**Complete, unedited output:**

```
                                         QUERY PLAN                                          
---------------------------------------------------------------------------------------------
 Limit  (cost=0.00..2.64 rows=1 width=4)
   ->  Seq Scan on contact  (cost=0.00..2.64 rows=1 width=4)
         Filter: ((reply_email)::text = 'sender_at_external_test_ycnjnjlzcw@sl.local'::text)
(3 rows)

```

**Command (forcing the index path):**

```
docker exec sl-work bash -lc "PGPASSWORD=test psql -h localhost -p 15432 -U test -d test -c \"SET enable_seqscan=off; EXPLAIN SELECT id FROM contact WHERE reply_email = 'sender_at_external_test_ycnjnjlzcw@sl.local' LIMIT 1;\""
```

**Complete, unedited output:**

```
SET
                                           QUERY PLAN                                            
-------------------------------------------------------------------------------------------------
 Limit  (cost=0.14..8.16 rows=1 width=4)
   ->  Index Scan using ix_contact_reply_email on contact  (cost=0.14..8.16 rows=1 width=4)
         Index Cond: ((reply_email)::text = 'sender_at_external_test_ycnjnjlzcw@sl.local'::text)
(3 rows)

```

Neither plan contains `ORDER BY`: the default is `Limit → Seq Scan on contact` and the forced path is `Limit → Index Scan using ix_contact_reply_email`. The physical rows behind this decision:

**Command:**

```
docker exec sl-work bash -lc "PGPASSWORD=test psql -h localhost -p 15432 -U test -d test -c \"SELECT ctid,id,alias_id,user_id FROM contact WHERE reply_email='sender_at_external_test_ycnjnjlzcw@sl.local' ORDER BY ctid;\""
```

**Complete, unedited output:**

```
  ctid  | id  | alias_id | user_id 
--------+-----+----------+---------
 (1,50) | 207 |      992 |     596
 (1,51) | 208 |      990 |     595
(2 rows)

```

Contact `207` (User B) occupies the earlier heap slot `(1,50)`, so an unordered `LIMIT 1` returns User B. (The `ORDER BY ctid` here is only to *display* physical order; the application query has no such clause.)

---

### R5(b) — the same reply email temporarily FAILS to resolve

**Direct answer: yes** — there is a window between minting a reverse alias and committing its `Contact`. `generate_reply_email()` [app/email_utils.py:1103] uses `available_sl_email()` [app/models.py:1425] to pick a free string, but the `Contact` row does not exist until it is created and committed. During that window `Contact.get_by(reply_email=R_new)` returns `None`, and a reply arriving then hits the no-contact guard.

**Command:** `docker exec sl-work bash -lc 'cd /app && python /tmp/obs_repro.py'` — segment of that run:

```
########## R5(b): temporary FAILURE to resolve (generate -> before-commit window) ##########
generate_reply_email(...) -> R_new='brandnew_at_external_test_hshjosh@sl.local'
available_sl_email(R_new) = True
Contact.get_by(reply_email=R_new) = None
2026-07-08 06:25:28,261 - SL - WARNING - 7916 - "/app/email_handler.py:988" - handle_reply() -  - No contact with brandnew_at_external_test_hshjosh@sl.local as reverse alias
handle_reply(envelope, msg, R_new) -> (False, '550 SL E502 Email not exist')

```

`R_new='brandnew_at_external_test_hshjosh@sl.local'` is reported free (`available_sl_email=True`) yet has no contact (`get_by=None`), so `handle_reply()` logs `No contact with … as reverse alias` [email_handler.py:988] and returns `(False, '550 SL E502 Email not exist')` [email_handler.py:989] [app/email/status.py:39]. (In the block above, `generate_reply_email(...)` is the observation script's own verbatim print label for the call — the arguments were elided *by the script when printing its label*, not from any log line; the produced value `R_new=…` is shown in full and no captured output was edited.)

### R5(c) — resolves correctly, but the reply would go to a DIFFERENT user

This is the crux of the reported "sometimes wrong user" behavior. It has **two distinct observed modes**, both driven with the **same unchanged input** used by the happy path (`mail_from=userA.email`, `rcpt_to=R`) after the R5(a) flip made the lookup resolve to User B's alias `992`.

#### R5(c)-i — the honest same-input outcome: E214 (User A's own reply is DENIED)

**Command:** `docker exec sl-work bash -lc 'cd /app && python /tmp/obs_repro.py'` — segment of that run:

```
########## R5(c)-i: SAME UNCHANGED INPUT through REAL handle() AFTER flip (mail_from=userA.email) ##########
2026-07-08 06:25:28,171 - SL - INFO - 7916 - "/app/email_handler.py:1956" - handle() -  - Set CONTENT_TRANSFER_ENCODING
2026-07-08 06:25:28,171 - SL - DEBUG - 7916 - "/app/email_handler.py:1963" - handle() -  - Cannot parse Postfix queue ID from None None
2026-07-08 06:25:28,171 - SL - DEBUG - 7916 - "/app/email_handler.py:1980" - handle() -  - ==>> Handle mail_from:user_noqs7byvg0@mailbox.test, rcpt_tos:['sender_at_external_test_ycnjnjlzcw@sl.local'], header_from:user_noqs7byvg0@mailbox.test, header_to:sender_at_external_test_ycnjnjlzcw@sl.local, cc:None, reply-to:None, message_id:<obs-r5c-unchanged@sl.local>, client_ip:None, headers:[('From', 'user_noqs7byvg0@mailbox.test'), ('To', 'sender_at_external_test_ycnjnjlzcw@sl.local'), ('Message-ID', '<obs-r5c-unchanged@sl.local>'), ('Subject', 'obs'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-08 06:25:28,175 - SL - DEBUG - 7916 - "/app/email_handler.py:2196" - handle() -  - Reply phase user_noqs7byvg0@mailbox.test(user_noqs7byvg0@mailbox.test) -> sender_at_external_test_ycnjnjlzcw@sl.local
2026-07-08 06:25:28,178 - SL - INFO - 7916 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() -  - DMARC check disabled
2026-07-08 06:25:28,180 - SL - WARNING - 7916 - "/app/email_handler.py:1393" - handle_unknown_mailbox() -  - Reply email can only be used by mailbox. Actual mail_from: user_noqs7byvg0@mailbox.test. msg from header: user_noqs7byvg0@mailbox.test, reverse-alias sender_at_external_test_ycnjnjlzcw@sl.local, <Alias 992 cuddle_squirm393@sl.local> <User 596 Test User user_6xdpa4pon0@mailbox.test> <Contact 207 other@external.test 992>
2026-07-08 06:25:28,200 - SL - DEBUG - 7916 - "/app/app/email_utils.py:303" - send_email() -  - send email to user_6xdpa4pon0@mailbox.test, subject 'Attempt to use your alias cuddle_squirm393@sl.local from user_noqs7byvg0@mailbox.test'
2026-07-08 06:25:28,204 - SL - DEBUG - 7916 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'Attempt to use your alias cuddle_squirm393@sl.local from user_noqs7byvg0@mailbox.test', from '"noreply@sl.local" <noreply@sl.local>' to 'user_6xdpa4pon0@mailbox.test'
email_handler.handle(env, msg) [UNCHANGED mail_from=userA.email] RETURNED: '250 SL E214 Unauthorized for using reverse alias'

```

**Reading the output:** with the **identical** unchanged reply (`mail_from=user_noqs7byvg0@mailbox.test` — User A's mailbox), `handle()` now resolves the reply email to **User B's** alias `992`/contact `207`. Because User A's `mail_from` is **not** an authorized sender for alias `992`, `get_mailbox_from_mail_from()` returns `None`, `alias.disable_email_spoofing_check` is `False` (default), so control reaches `handle_unknown_mailbox()` [email_handler.py:1393], which emails **User B** a security notice (`Attempt to use your alias cuddle_squirm393@sl.local from user_noqs7byvg0@mailbox.test`) and returns **`'250 SL E214 Unauthorized for using reverse alias'`** [email_handler.py:1034] [app/email/status.py:22]. So the *honest* consequence of the same unchanged input after the flip is that **User A's own reply is refused** (and User B is even told about A's attempt) — a real, observable defect, not a delivery to A's intended recipient.

#### R5(c)-ii — the cross-user DELIVERY mode (real per-alias flag, clearly labeled)

The reply is actually *delivered to the wrong user's contact* when the resolved alias carries the real per-alias setting **`disable_email_spoofing_check=True`** — a genuine product feature stored on the `Alias`, not a test bypass. With that flag set on alias `992`, the same unchanged input takes the `ignore unknown sender to reverse-alias` branch [email_handler.py:1023] instead of E214.

**Command:** `docker exec sl-work bash -lc 'cd /app && python /tmp/obs_repro.py'` — segment of that run:

```
########## R5(c)-ii: cross-user DELIVERY mechanism (real per-alias disable_email_spoofing_check=True on aliasB) ##########
aliasB.disable_email_spoofing_check = True
2026-07-08 06:25:28,209 - SL - INFO - 7916 - "/app/email_handler.py:1956" - handle() -  - Set CONTENT_TRANSFER_ENCODING
2026-07-08 06:25:28,209 - SL - DEBUG - 7916 - "/app/email_handler.py:1963" - handle() -  - Cannot parse Postfix queue ID from None None
2026-07-08 06:25:28,210 - SL - DEBUG - 7916 - "/app/email_handler.py:1980" - handle() -  - ==>> Handle mail_from:user_noqs7byvg0@mailbox.test, rcpt_tos:['sender_at_external_test_ycnjnjlzcw@sl.local'], header_from:user_noqs7byvg0@mailbox.test, header_to:sender_at_external_test_ycnjnjlzcw@sl.local, cc:None, reply-to:None, message_id:<obs-r5c-deliver@sl.local>, client_ip:None, headers:[('From', 'user_noqs7byvg0@mailbox.test'), ('To', 'sender_at_external_test_ycnjnjlzcw@sl.local'), ('Message-ID', '<obs-r5c-deliver@sl.local>'), ('Subject', 'obs'), ('Content-Transfer-Encoding', '7bit')], mail_options:[], rcpt_options:[]
2026-07-08 06:25:28,214 - SL - DEBUG - 7916 - "/app/email_handler.py:2196" - handle() -  - Reply phase user_noqs7byvg0@mailbox.test(user_noqs7byvg0@mailbox.test) -> sender_at_external_test_ycnjnjlzcw@sl.local
2026-07-08 06:25:28,217 - SL - INFO - 7916 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() -  - DMARC check disabled
2026-07-08 06:25:28,218 - SL - WARNING - 7916 - "/app/email_handler.py:1023" - handle_reply() -  - ignore unknown sender to reverse-alias user_noqs7byvg0@mailbox.test: <Alias 992 cuddle_squirm393@sl.local> -> <Contact 207 other@external.test 992>
2026-07-08 06:25:28,220 - SL - DEBUG - 7916 - "/app/email_handler.py:1051" - handle_reply() -  - Create <EmailLog 377> for <Contact 207 other@external.test 992>, <User 596 Test User user_6xdpa4pon0@mailbox.test>, <Mailbox 711 user_6xdpa4pon0@mailbox.test>
2026-07-08 06:25:28,225 - SL - DEBUG - 7916 - "/app/email_handler.py:1171" - handle_reply() -  - From header is cuddle_squirm393@sl.local
2026-07-08 06:25:28,226 - SL - DEBUG - 7916 - "/app/email_handler.py:380" - replace_header_when_reply() -  - Replace To header, old: sender_at_external_test_ycnjnjlzcw@sl.local, new: other@external.test
2026-07-08 06:25:28,226 - SL - DEBUG - 7916 - "/app/email_handler.py:383" - replace_header_when_reply() -  - delete the Cc header. Old value None
2026-07-08 06:25:28,228 - SL - DEBUG - 7916 - "/app/email_handler.py:1314" - replace_original_message_id() -  - create a new sl_message_id <178349192822.7916.3647946367306320093.377@sl.local>
2026-07-08 06:25:28,231 - SL - WARNING - 7916 - "/app/email_handler.py:1206" - handle_reply() -  - missing date header, add one
2026-07-08 06:25:28,233 - SL - DEBUG - 7916 - "/app/email_handler.py:1212" - handle_reply() -  - send email from cuddle_squirm393@sl.local to other@external.test, mail_options:[],rcpt_options:[]
2026-07-08 06:25:28,236 - SL - DEBUG - 7916 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'obs', from 'cuddle_squirm393@sl.local' to 'other@external.test'
email_handler.handle(env, msg) [UNCHANGED mail_from=userA.email, aliasB spoof-check OFF] RETURNED: '250 Message accepted for delivery'
EmailLog -> id=377 contact_id=207 alias_id=992 user_id=596 mailbox_id=711 is_reply=True (user_id=596 is User B, NOT alias-owner userA=595)
OUTBOUND recipient for this delivery = contact_B.website_email = 'other@external.test' (User B's external contact, NOT userA's 'sender@external.test')

```

**Reading the output:** after setting `aliasB.disable_email_spoofing_check = True`, the same unchanged input (`mail_from=userA.email`) now returns **`'250 Message accepted for delivery'`**, writes `EmailLog id=377 contact_id=207 alias_id=992 user_id=596 mailbox_id=711` (**User B**, not alias-owner User A `595`), and — critically — logs `send email from cuddle_squirm393@sl.local to other@external.test`. **The outbound recipient is `contact_B.website_email='other@external.test'` (User B's external contact), NOT User A's intended `sender@external.test`.** That is the concrete cross-user exposure: User A's private reply content is transmitted to User B's contact address. The value that changed was the *resolved contact* (via the non-unique `reply_email` + unordered `.first()`), not the input.

---

## R6 — Race conditions, the uniqueness assumption, and timing

**Direct answer:** the reply path *behaves as if* `reply_email` were unique (a single `.first()` and done), but the schema does **not** enforce it. The only uniqueness on `contact` is `uq_contact(alias_id, website_email)` [app/models.py:1875]; `reply_email` is merely indexed [app/models.py:1899] [migrations/versions/2021_071310_78403c7b8089_.py:22]. Creation is a check-then-insert: `available_sl_email()` [app/models.py:1425] is a read-only existence probe, and nothing between it and the subsequent `Contact.create()` holds a lock or a unique constraint on `reply_email` — a textbook TOCTOU window.

**Command:** `docker exec sl-work bash -lc 'cd /app && python /tmp/obs_repro.py'` — segment of that run:

```
########## R6: TOCTOU / uniqueness assumption ##########
available_sl_email(R) [R used by 2 contacts] = False
available_sl_email('unused_marker_key_9988@sl.local') = True
uq_contact duplicate (alias_id=992, website_email='other@external.test') -> IntegrityError: duplicate key value violates unique constraint "uq_contact"
2026-07-08 06:25:28,546 - SL - INFO - 7916 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 06:25:28,559 - SL - DEBUG - 7916 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email dahlia_smokes819@sl.local
2026-07-08 06:25:28,567 - SL - INFO - 7916 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 06:25:28,826 - SL - INFO - 7916 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 06:25:28,838 - SL - DEBUG - 7916 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email weblog_unhurt346@sl.local
2026-07-08 06:25:28,846 - SL - INFO - 7916 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
worker P available_sl_email(R_toc='toc_at_external_test_kmtvj@sl.local')=True ; worker Q available_sl_email(R_toc)=True  (both see 'free' -> TOCTOU window)
both Contact.create SUCCEEDED -> rows reply_email=R_toc (id,user_id): [(210, 597), (211, 598)]

```

**Reading the output:**
- `available_sl_email(R)` is `False` (R is in use by two contacts) while `available_sl_email('unused_marker_key_9988@sl.local')` is `True` — the probe works, but it only reflects a point-in-time read.
- Attempting a genuine `uq_contact` collision — a second row with the **same `(alias_id=992, website_email='other@external.test')`** — *does* raise `IntegrityError: duplicate key value violates unique constraint "uq_contact"`. This proves the DB enforces `(alias_id, website_email)` but **not** `reply_email`.
- **TOCTOU:** two simulated workers **P** and **Q** both call `available_sl_email(R_toc='toc_at_external_test_kmtvj@sl.local')` and **both see `True`** ("free"); both then `Contact.create(...)` and **both SUCCEED**, leaving two rows `[(210, 597), (211, 598)]` sharing `R_toc`. The `IntegrityError` fallback in `create_contact()` re-fetches by `(alias_id, website_email)` only [app/contact_utils.py:113-118], so even when it fires it cannot detect or prevent a duplicate `reply_email`.

This is precisely how the duplicate rows that drive R5 can arise in production without any error being surfaced.

## R7 — Observed runtime values → behavior

**Command:** `docker exec sl-work bash -lc 'cd /app && python /tmp/obs_repro.py'` — segment of that run:

```
########## R7: CONSOLIDATED OBSERVED VALUES ##########
reply_email R = 'sender_at_external_test_ycnjnjlzcw@sl.local' (normalized == R: True)
User-A original row : contact.id=206 alias_id=990 user_id=595 website_email=sender@external.test
User-A(A2) row      : contact.id=208 alias_id=990 user_id=595
User-B row (dup)    : contact.id=207 alias_id=992 user_id=596 (alias cuddle_squirm393@sl.local, website_email=other@external.test)
post-flip resolved  : contact.id=207 user_id=596 alias_id=992
R5(c)-ii delivery   : EmailLog id=377 user_id=596 mailbox_id=711 outbound-recipient=other@external.test

```

Consolidated, with the causal chain:

| Item | Observed value | Consequence |
|------|----------------|-------------|
| `reply_email` `R` | `sender_at_external_test_ycnjnjlzcw@sl.local` (normalized == R) | the lookup key; minted for User A's alias |
| User-A original contact | `id=206 alias_id=990 user_id=595` (`website_email=sender@external.test`) | correct target while it is the only row |
| User-A reinserted (A2) | `id=208 alias_id=990 user_id=595` | heap slot `(1,51)` after reorder |
| User-B duplicate contact | `id=207 alias_id=992 user_id=596` (`website_email=other@external.test`) | heap slot `(1,50)` — physically first |
| post-flip resolution | `contact.id=207 user_id=596 alias_id=992` | unordered `LIMIT 1` returns User B |
| R5(c)-ii delivery | `EmailLog id=377 user_id=596 mailbox_id=711 outbound-recipient=other@external.test` | User A's reply transmitted to User B's contact |

**How these produce the observed (wrong-user) behavior:** the reply email `R` is unique to the human eye but not to the database. Once row `207` (User B) precedes row `208` (User A) in heap order, `Contact.get_by(reply_email=R).first()` returns `207`; the handler then reads `alias=992`, `user=596`, and forwards to `contact.website_email='other@external.test'`. Same envelope, different physical order → different user, different outbound recipient.

---

## Guarded-exit matrix — every distinct condition of the reply handler

To be exhaustive rather than representative, every guarded exit of `handle_reply()` was exercised through the production function in a single run. The block below is the **complete, unedited** output (including the SL log lines each branch emits and — verbatim — the `NoneType: None` line the logger prints for the E503 exception path). Two branches, **E201 (SPF)** and **E506 (spam)**, are unreachable in the default/canonical config because `ENFORCE_SPF` [app/config.py:132] and `ENABLE_SPAM_ASSASSIN` [app/config.py:450] default to off; those two are explicitly **labeled `[NON-CANONICAL]`** and were forced only to capture their status strings — every other branch is canonical.

**Command:**

```
docker exec sl-work bash -lc 'cd /app && python /tmp/obs_exits.py'
```

**Complete, unedited output:**

```
load config file /app/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/nqcqxbdtewdkrfiemctt
Upload files to local dir
>>> init logging <<<
2026-07-08 06:28:20,153 - SL - DEBUG - 7980 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-08 06:28:21,207 - SL - DEBUG - 7980 - "/app/init_app.py:42" - add_sl_domains() -  - d1.test is already a SL domain
2026-07-08 06:28:21,208 - SL - DEBUG - 7980 - "/app/init_app.py:42" - add_sl_domains() -  - d2.test is already a SL domain
2026-07-08 06:28:21,209 - SL - DEBUG - 7980 - "/app/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-08 06:28:21,210 - SL - DEBUG - 7980 - "/app/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
=== GUARDED-EXIT MATRIX (all via email_handler.handle_reply, the production fn invoked by handle() at :2199) ===
2026-07-08 06:28:21,507 - SL - INFO - 7980 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 06:28:21,523 - SL - DEBUG - 7980 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email uproar_lathes459@sl.local
2026-07-08 06:28:21,531 - SL - INFO - 7980 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 06:28:21,556 - SL - DEBUG - 7980 - "/app/app/contact_utils.py:110" - create_contact() -  - Created contact <Contact 212 peer@external.test 998> for alias <Alias 998 uproar_lathes459@sl.local> with email peer@external.test invalid_email=False
2026-07-08 06:28:21,559 - SL - WARNING - 7980 - "/app/email_handler.py:980" - handle_reply() -  - Reply email randabc123@unknown-domain.example has wrong domain
E501 (bad reply domain): rcpt_to='randabc123@unknown-domain.example' -> (False, '550 SL E501')   [return email_handler.py:981 | status.py:38]
   is_reverse_alias('nonexistent_reverse_xyz@sl.local') = False
2026-07-08 06:28:21,560 - SL - WARNING - 7980 - "/app/email_handler.py:988" - handle_reply() -  - No contact with nonexistent_reverse_xyz@sl.local as reverse alias
E502 (no contact): rcpt_to='nonexistent_reverse_xyz@sl.local' -> (False, '550 SL E502 Email not exist')   [return :989 | status.py:39]
2026-07-08 06:28:21,817 - SL - INFO - 7980 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 06:28:21,829 - SL - DEBUG - 7980 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email badman_perked335@sl.local
2026-07-08 06:28:21,837 - SL - INFO - 7980 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 06:28:21,861 - SL - DEBUG - 7980 - "/app/app/contact_utils.py:110" - create_contact() -  - Created contact <Contact 213 peer@external.test 1000> for alias <Alias 1000 badman_perked335@sl.local> with email peer@external.test invalid_email=False
   user.id=600 delete_on(future)=2026-08-07T06:28:21.862086+00:00 -> is_active()=False
2026-07-08 06:28:21,868 - SL - WARNING - 7980 - "/app/email_handler.py:991" - handle_reply() -  - User <User 600 Test User user_u0yl50sl9d@mailbox.test> has been soft deleted
E502 (inactive user): reply_email='peer_at_external_test_jerhie@sl.local' -> (False, '550 SL E502 Email not exist')   [return :992 | status.py:39]
2026-07-08 06:28:22,124 - SL - INFO - 7980 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 06:28:22,137 - SL - DEBUG - 7980 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email pinged_dilate521@sl.local
2026-07-08 06:28:22,144 - SL - INFO - 7980 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 06:28:22,169 - SL - DEBUG - 7980 - "/app/app/contact_utils.py:110" - create_contact() -  - Created contact <Contact 214 peer@external.test 1002> for alias <Alias 1002 pinged_dilate521@sl.local> with email peer@external.test invalid_email=False
   alias.email='orphan_35db6f@removed-domain.example' -> is_valid_alias_address_domain=False
2026-07-08 06:28:22,187 - SL - ERROR - 7980 - "/app/email_handler.py:1001" - handle_reply() -  - <Alias 1002 orphan_35db6f@removed-domain.example> domain isn't known
NoneType: None
E503 (alias domain unknown): reply_email='peer_at_external_test_qruhjww@sl.local' -> (False, '550 SL E503')   [return :1002 | status.py:40]
2026-07-08 06:28:22,443 - SL - INFO - 7980 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 06:28:22,457 - SL - DEBUG - 7980 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email camera_chides534@sl.local
2026-07-08 06:28:22,465 - SL - INFO - 7980 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 06:28:22,489 - SL - DEBUG - 7980 - "/app/app/contact_utils.py:110" - create_contact() -  - Created contact <Contact 215 peer@external.test 1004> for alias <Alias 1004 camera_chides534@sl.local> with email peer@external.test invalid_email=False
2026-07-08 06:28:22,494 - SL - INFO - 7980 - "/app/app/models.py:888" - can_send_or_receive() -  - User <User 602 Test User user_nhn9bshvee@mailbox.test> is disabled. Cannot receive or send emails
   user.id=602 disabled=True is_active()=True can_send_or_receive()=False
2026-07-08 06:28:22,497 - SL - INFO - 7980 - "/app/app/models.py:888" - can_send_or_receive() -  - User <User 602 Test User user_nhn9bshvee@mailbox.test> is disabled. Cannot receive or send emails
2026-07-08 06:28:22,497 - SL - INFO - 7980 - "/app/email_handler.py:1008" - handle_reply() -  - User <User 602 Test User user_nhn9bshvee@mailbox.test> cannot send emails
E504 (account disabled): reply_email='peer_at_external_test_ptymwlrokh@sl.local' -> (False, '550 SL E504 Account disabled')   [return :1009 | status.py:41]
2026-07-08 06:28:22,753 - SL - INFO - 7980 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 06:28:22,766 - SL - DEBUG - 7980 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email tacked_stiles961@sl.local
2026-07-08 06:28:22,774 - SL - INFO - 7980 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 06:28:22,798 - SL - DEBUG - 7980 - "/app/app/contact_utils.py:110" - create_contact() -  - Created contact <Contact 216 peer@external.test 1006> for alias <Alias 1006 tacked_stiles961@sl.local> with email peer@external.test invalid_email=False
   mail_from='stranger@evil.example' authorized? mailbox=None ; alias.disable_email_spoofing_check=False
2026-07-08 06:28:22,806 - SL - INFO - 7980 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() -  - DMARC check disabled
2026-07-08 06:28:22,806 - SL - WARNING - 7980 - "/app/email_handler.py:1393" - handle_unknown_mailbox() -  - Reply email can only be used by mailbox. Actual mail_from: stranger@evil.example. msg from header: stranger@evil.example, reverse-alias peer_at_external_test_dowfinsok@sl.local, <Alias 1006 tacked_stiles961@sl.local> <User 603 Test User user_nvgmcg6oi7@mailbox.test> <Contact 216 peer@external.test 1006>
2026-07-08 06:28:22,825 - SL - DEBUG - 7980 - "/app/app/email_utils.py:303" - send_email() -  - send email to user_nvgmcg6oi7@mailbox.test, subject 'Attempt to use your alias tacked_stiles961@sl.local from stranger@evil.example'
2026-07-08 06:28:22,830 - SL - DEBUG - 7980 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'Attempt to use your alias tacked_stiles961@sl.local from stranger@evil.example', from '"noreply@sl.local" <noreply@sl.local>' to 'user_nvgmcg6oi7@mailbox.test'
E214 (unauthorized mailbox): mail_from='stranger@evil.example' -> (False, '250 SL E214 Unauthorized for using reverse alias')   [return :1034 | status.py:22]
2026-07-08 06:28:23,086 - SL - INFO - 7980 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 06:28:23,099 - SL - DEBUG - 7980 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email aghast_flunky494@sl.local
2026-07-08 06:28:23,106 - SL - INFO - 7980 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 06:28:23,131 - SL - DEBUG - 7980 - "/app/app/contact_utils.py:110" - create_contact() -  - Created contact <Contact 217 peer@external.test 1008> for alias <Alias 1008 aghast_flunky494@sl.local> with email peer@external.test invalid_email=False
   [NON-CANONICAL] ENFORCE_SPF True (default False [app/config.py:132]), mailbox.force_spf=True, spf_pass monkeypatched->False
2026-07-08 06:28:23,138 - SL - INFO - 7980 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() -  - DMARC check disabled
E201 (SPF fail) [NON-CANONICAL]: -> (True, '250 SL E201')   [return :1040 | status.py:3]
2026-07-08 06:28:23,396 - SL - INFO - 7980 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 06:28:23,409 - SL - DEBUG - 7980 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email skycap_expose611@sl.local
2026-07-08 06:28:23,416 - SL - INFO - 7980 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 06:28:23,444 - SL - DEBUG - 7980 - "/app/app/contact_utils.py:110" - create_contact() -  - Created contact <Contact 218 peer@external.test 1010> for alias <Alias 1010 skycap_expose611@sl.local> with email peer@external.test invalid_email=False
   [NON-CANONICAL] ENABLE_SPAM_ASSASSIN True (default False [app/config.py:450]), SPAMASSASSIN_HOST=None, get_spam_info monkeypatched->(True,'forced-spam')
2026-07-08 06:28:23,449 - SL - INFO - 7980 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() -  - DMARC check disabled
2026-07-08 06:28:23,453 - SL - DEBUG - 7980 - "/app/email_handler.py:1051" - handle_reply() -  - Create <EmailLog 378> for <Contact 218 peer@external.test 1010>, <User 605 Test User user_dc0iua41na@mailbox.test>, <Mailbox 720 user_dc0iua41na@mailbox.test>
2026-07-08 06:28:23,458 - SL - WARNING - 7980 - "/app/email_handler.py:1081" - handle_reply() -  - Email detected as spam. Reply phase. <Alias 1010 skycap_expose611@sl.local> -> <Contact 218 peer@external.test 1010>. Spam Score: None, Spam Report: None
2026-07-08 06:28:23,466 - SL - DEBUG - 7980 - "/app/email_handler.py:1724" - handle_spam() -  - Create spam email <Refused Email 8 None 2026-07-15T06:28:23.463434+00:00>
2026-07-08 06:28:23,469 - SL - DEBUG - 7980 - "/app/email_handler.py:1730" - handle_spam() -  - Inform <Mailbox 720 user_dc0iua41na@mailbox.test> (<User 605 Test User user_dc0iua41na@mailbox.test>) about spam email sent from alias <Alias 1010 skycap_expose611@sl.local> to <Contact 218 peer@external.test 1010>. <Refused Email 8 None 2026-07-15T06:28:23.463434+00:00>
2026-07-08 06:28:23,488 - SL - DEBUG - 7980 - "/app/app/email_utils.py:303" - send_email() -  - send email to user_dc0iua41na@mailbox.test, subject 'Email from skycap_expose611@sl.local to peer@external.test is detected as spam'
2026-07-08 06:28:23,492 - SL - DEBUG - 7980 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'Email from skycap_expose611@sl.local to peer@external.test is detected as spam', from '"noreply@sl.local" <noreply@sl.local>' to 'user_dc0iua41na@mailbox.test'
E506 (spam) [NON-CANONICAL]: -> (False, '550 SL E506 Email detected as spam')   [return :1094 | status.py:43]
=== END GUARDED-EXIT MATRIX ===
```

| Condition | Trigger | Return | Return site / status |
|-----------|---------|--------|----------------------|
| **E501** | reply domain not an SL/alias domain | `(False, '550 SL E501')` | [email_handler.py:981] / [app/email/status.py:38] |
| **E502** (no contact) | `Contact.get_by(reply_email)` is `None` (`is_reverse_alias`=False) | `(False, '550 SL E502 Email not exist')` | [email_handler.py:989] / [app/email/status.py:39] |
| **E502** (inactive user) | owner `is_active()`=False (`delete_on` in the future) [app/models.py:766-769] | `(False, '550 SL E502 Email not exist')` | [email_handler.py:992] / [app/email/status.py:39] |
| **E503** | `is_valid_alias_address_domain(alias.email)`=False | `(False, '550 SL E503')` | [email_handler.py:1002] / [app/email/status.py:40] |
| **E504** | owner `can_send_or_receive()`=False (`disabled=True`) [app/models.py:886-895] | `(False, '550 SL E504 Account disabled')` | [email_handler.py:1009] / [app/email/status.py:41] |
| **E214** | `get_mailbox_from_mail_from()`=None and spoof-check on → `handle_unknown_mailbox()` [email_handler.py:1393] | `(False, '250 SL E214 Unauthorized for using reverse alias')` | [email_handler.py:1034] / [app/email/status.py:22] |
| **E201** `[NON-CANONICAL]` | SPF fail with `ENFORCE_SPF` forced on | `(True, '250 SL E201')` | [email_handler.py:1040] / [app/email/status.py:3] |
| **E506** `[NON-CANONICAL]` | spam verdict with SpamAssassin forced on | `(False, '550 SL E506 Email detected as spam')` | [email_handler.py:1094] / [app/email/status.py:43] |

Note the E214 branch returns a `250` SMTP code (the message is *accepted then dropped* with an internal notification), which is why in R5(c)-i the same-input outcome is reported as `250 SL E214` rather than a hard `550`.

## State-transition report (`contact` rows for `reply_email=R`)

**Command:** `docker exec sl-work bash -lc 'cd /app && python /tmp/obs_repro.py'` — closing segment of that run:

```
########## STATE-TRANSITION rows (ctid,id,alias_id,user_id) for reply_email=R ##########
stage summary: userA.id=595 aliasA.id=990 | userB.id=596 aliasB.id=992 | contactA=206 contactA2=208 contactB=207
DONE
```

| Stage | Rows `(ctid, id, alias_id, user_id)` for `reply_email=R` | `get_by(...).first()` resolves to |
|-------|----------------------------------------------------------|-----------------------------------|
| **Before** (seed) | `[('(1,49)', 206, 990, 595)]` | User A (`595`) |
| **Intermediate** (duplicate inserted) | `[('(1,49)', 206, 990, 595), ('(1,50)', 207, 992, 596)]` | User A (`595`) — heap unchanged |
| **After** (heap reorder: delete 206, reinsert as 208) | `[('(1,50)', 207, 992, 596), ('(1,51)', 208, 990, 595)]` | **User B (`596`)** — flipped |

The resolved user transitions `595 → 595 → 596` while the *input never changes* — the state that changed is the physical order of two rows that were never required to be unique.

## Root-cause synthesis

- **Mechanism.** `Contact.get_by(reply_email=…).first()` is a `LIMIT 1` with no `ORDER BY` [app/models.py:83-84] (confirmed by emitted SQL in R2 and by `EXPLAIN` in R5(a)). SQL does not guarantee row order without `ORDER BY`, so the row returned is whichever the chosen plan yields first (heap order for a seq scan, index order for an index scan) — and that can change after inserts/deletes/vacuum.
- **Enabling condition.** `reply_email` is indexed but **not unique** [app/models.py:1899] [migrations/versions/2021_071310_78403c7b8089_.py:22]; the sole `contact` uniqueness is `uq_contact(alias_id, website_email)` [app/models.py:1875]. Duplicates on `reply_email` are legal and can be created without error (R5(a), R6).
- **How duplicates arise.** The check-then-insert in `available_sl_email()` → `Contact.create()` has a TOCTOU window (R6); the `create_contact()` `IntegrityError` fallback only understands `(alias_id, website_email)` [app/contact_utils.py:113-118], so it cannot catch a duplicate `reply_email`. A normalization collision (R1) is a second route to a shared `reply_email`.
- **Effect.** With duplicates present, a reply resolves to whichever row is physically first. That decides `alias` (`contact.alias`), `user` (`alias.user`), the authorized mailbox (`EmailLog.mailbox_id`), and — the part that actually reaches a human — the **outbound recipient `contact.website_email`** [email_handler.py:1212,1224-1230]. If the winning row belongs to another user, the same unchanged reply is either **denied to its rightful owner** (E214, R5(c)-i) or, when the resolved alias has `disable_email_spoofing_check=True`, **delivered to the other user's contact address** (R5(c)-ii).

## Web-research corroboration (secondary — the runtime evidence above is primary)

- PostgreSQL's documentation states that without `ORDER BY`, `SELECT` returns rows in an unspecified, implementation-dependent order, and a `LIMIT` without `ORDER BY` can return different subsets across executions (`postgresql.org/docs/current/sql-select.html`, `.../queries-limit.html`). This matches the observed `EXPLAIN` (no `ORDER BY`) and the cross-process flip.
- The naive check-then-insert (SELECT-then-INSERT) is a recognized TOCTOU race that only a DB unique constraint or explicit locking makes safe. This matches the observed dual-success in R6.

---

## Coverage pass — every named sub-question, answered

Re-reading the question and confirming each named item is addressed with a concrete value, a `file:line`, and observed evidence:

| # | Sub-question | Answered in | Direct answer (with evidence) |
|---|--------------|-------------|-------------------------------|
| R1 | How is the "reply email" derived from an inbound reply? | R1 | `reply_email = rcpt_to` [email_handler.py:972] then `normalize_reply_email()` [:984]; observed `R='sender_at_external_test_ycnjnjlzcw@sl.local'`, `normalize(R)==R` True |
| R2 | How does that reply email identify the `Contact`? | R2 | `Contact.get_by(reply_email=…)` [:986] = `filter_by(...).first()` [app/models.py:83-84]; emitted SQL has `LIMIT` and **no `ORDER BY`**; resolved `id=206` |
| R3 | Which user / alias / mailbox is chosen? | R3 | user=`alias.user`=595 [:1004]; alias=`contact.alias`=990 [:994]; **selected mailbox**=710 (authorized sender + `EmailLog.mailbox_id`); **outbound recipient**=`contact.website_email='sender@external.test'` [:1212,1224-1230] |
| R4 | Behavior across multiple reply events over time? | R4 | 3 full `handle()` events → same 206/595/990, `EmailLog` 374/375/376; 10 lookups all `595` |
| R5(a) | Same reply email → different contacts over time? | R5(a) | flip `595 → 596` after heap reorder; cross-process (3 pids) all `207/596`; `EXPLAIN` shows no `ORDER BY` |
| R5(b) | Same reply email temporarily fails to resolve? | R5(b) | pre-commit window: `available_sl_email=True`, `get_by=None` → `(False,'550 SL E502 Email not exist')` [:988-989] |
| R5(c) | Resolves but forwards to a different user? | R5(c)-i, R5(c)-ii | **same unchanged input** → `250 SL E214` (A denied) [:1034]; with real `disable_email_spoofing_check=True` → delivered to `other@external.test` (User B's contact), `EmailLog 377/user 596` |
| R6 | Race conditions / uniqueness assumption / timing? | R6 | duplicate `reply_email` inserts with **no** error; `uq_contact` fires only on `(alias_id, website_email)`; two workers both pass `available_sl_email` (TOCTOU) |
| R7 | Observed values → behavior? | R7 | consolidated table; `id=207 user_id=596` wins post-flip → reply to `other@external.test` |
| — | Every guarded exit enumerated | Guarded-exit matrix | E501/E502(×2)/E503/E504/E214 canonical + E201/E506 `[NON-CANONICAL]`, each with return string and `file:line` |
| — | State before / intermediate / after | State-transition report | rows `595 → 595 → 596`, input unchanged |
| — | Read-only, repository unchanged, scripts removed | Repository verification & cleanup | `git status --porcelain` shows only the answer doc; temp scripts removed |

Every part of the question is answered by name, with its concrete value, `file:line` grounding, observed evidence, sibling variants, and causal reason.

---

## Repository verification & cleanup (read-only proof)

This investigation was strictly read-only apart from creating this one document. The temporary observation scripts lived in the **container's `/tmp`** (`/tmp/obs_repro.py`, `/tmp/obs_exits.py`, `/tmp/obs_proc.py`, `/tmp/obs_R.txt`) — **outside** the bind-mounted `/app` working tree — so they were never part of the repository; they were removed on completion.

**Command (remove the temporary scripts and confirm):**

```
docker exec sl-work bash -lc 'rm -f /tmp/obs_repro.py /tmp/obs_exits.py /tmp/obs_proc.py /tmp/obs_R.txt && echo removed'
docker exec sl-work bash -lc 'ls -la /tmp/obs_*.py /tmp/obs_*.txt 2>&1 || echo "no such files (clean)"'
```

**Complete, unedited output:**

```
removed
ls: cannot access '/tmp/obs_*.py': No such file or directory
ls: cannot access '/tmp/obs_*.txt': No such file or directory
no such files (clean)
```

**Command (only the answer document is modified in the repo):**

```
git status --porcelain
```

**Complete, unedited output:**

```
 M blitzy/documentation/app_2cd6ee777f8c.md
```

**Command (which file changed, with status — invariant to this document's own size):**

```
git diff --name-status
```

**Complete, unedited output:**

```
M	blitzy/documentation/app_2cd6ee777f8c.md
```

**Command (no temporary observation scripts remain anywhere in the repository tree):**

```
find . -path ./.git -prune -o \( -name 'blitzy_adhoc_test_*' -o -name 'obs_repro.py' -o -name 'obs_exits.py' -o -name 'obs_proc.py' \) -print
```

**Complete, unedited output:** *(empty — no matches)*

```
```

The single tracked change is `M blitzy/documentation/app_2cd6ee777f8c.md`; no source file was modified, no code was added to the source repository, and no temporary artifact remains. This satisfies the read-only scope requirement.
