# SimpleLogin — How an inbound reply resolves to a Contact + forwarding destination, and why a reply can go to the WRONG user

> **Investigation type:** read-only, **RUN-FIRST** codebase investigation. Every behavioral claim below is backed by the **exact command executed** and its **complete, unedited output**, captured at runtime in the canonical container. Claims derived only from reading source are explicitly labeled **(inferred)**. Values obtained via a forced flag or monkeypatch are labeled **NON-CANONICAL** (the real branch is still exercised).

---

## TL;DR — direct answer

An inbound reply's destination is derived **entirely from the SMTP envelope recipient**. `handle_reply()` sets `reply_email = rcpt_to` [`email_handler.py:972`], normalizes it with `normalize_reply_email()` [`email_handler.py:984` → `app/email_validation.py:25`], then looks up the owning contact with `contact = Contact.get_by(reply_email=reply_email)` [`email_handler.py:986`]. `ModelMixin.get_by` is literally `return Session.query(cls).filter_by(**kw).first()` [`app/models.py:83-84`] — a **`LIMIT 1` query with NO `ORDER BY`** (proven by captured SQL below). The forwarding **user** is then `contact.alias.user` [`email_handler.py:994,1004`], the **alias** is `contact.alias`, and the delivery **mailbox** is `get_mailbox_from_mail_from(mail_from, alias)` [`email_handler.py:1019`, def `:1364`].

**Why a reply can be delivered to a user other than the alias owner:** the `reply_email` column is **indexed but NOT unique** — `sa.Column(sa.String(512), nullable=False, index=True)` [`app/models.py:1899`], created by `op.create_index(..., ['reply_email'], unique=False)` [`migrations/versions/2021_071310_78403c7b8089_.py:22`]. The **only** uniqueness on the `contact` table is `uq_contact(alias_id, website_email)` [`app/models.py:1875`]. Contact creation is a **check-then-insert** (`available_sl_email()` read-only check [`app/models.py:1425`] → `generate_reply_email()` [`app/email_utils.py:1103`] → `Contact.create()`), with an `IntegrityError` handler that recovers **only** the `uq_contact` violation [`app/contact_utils.py:113-118`]. Consequently, **two contacts belonging to different users can share the same `reply_email`**, and the unordered `.first()` then returns an **arbitrary** matching row whose physical (heap) position can change over time — so the very same reply address resolves to **User A** at one moment and **User B** at another. This was reproduced at runtime: 25/25 lookups → User A before a duplicate existed and a heap reorder occurred; **6/6 separate processes → User B** afterward; a real `email_handler.handle(...)` call then delivered the reply to **User B's** mailbox and wrote `EmailLog` proving it.

---

## Environment & Build / Invocation commands

All observations ran inside the user-specified canonical Docker container `sl-work` (image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`), which bind-mounts the repository at `/app`. The host is non-canonical (Python 3.13, no psycopg2/aiosmtpd, PostgreSQL not on the required port), so the app cannot run there; the container supplies Python 3.10 + the Poetry dependency set + PostgreSQL on port `15432`.

**Runtime + branch + PostgreSQL (exact commands and output):**

```
$ docker exec sl-work bash -lc 'python --version; python -c "import sqlalchemy, psycopg2, aiosmtpd; print(\"sqlalchemy\", sqlalchemy.__version__, \"| psycopg2\", psycopg2.__version__.split()[0], \"| aiosmtpd\", aiosmtpd.__version__)"; cat /app/.version; cd /app && git rev-parse --abbrev-ref HEAD && git rev-parse HEAD; pg_lsclusters'
Python 3.10.18
sqlalchemy 1.3.24 | psycopg2 2.9.3 | aiosmtpd 1.4.2
dev
blitzy-fa8339d5-38ba-4a32-a6f3-20c43c289bf8
2cd6ee777f8c2d3531559588bcfb18627ffb5d2c
Ver Cluster Port   Status Owner    Data directory              Log file
15  main    15432  online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

**Canonical configuration** is `tests/test.env` (`CONFIG=/app/tests/test.env`): `NOT_SEND_EMAIL=true` [`tests/test.env:7`] (emails are printed, not sent — ideal for observation), `EMAIL_DOMAIN=sl.local` [`tests/test.env:8`], `OTHER_ALIAS_DOMAINS=["d1.test","d2.test","sl.local"]` [`tests/test.env:9`], `DB_URI=postgresql://test:test@localhost:15432/test` [`tests/test.env:17`], `DMARC_CHECK_ENABLED=true` [`tests/test.env:66`].

**Observation harness bootstrap** mirrors `tests/conftest.py` (set `CONFIG` **before** importing app modules; build the app with `server.create_app()`; enable `pg_trgm`; seed SL domains + Proton partner; enter `app.app_context()`). The consolidated observation scripts (`/tmp/obs_doc_main.py`, `/tmp/obs_doc_exits.py`, `/tmp/obs_doc_wrong.py`, `/tmp/obs_doc_proc.py`) were written to the **container's** `/tmp` (never inside the repo) and deleted afterward (see the read-only verification at the end). Their common preamble is:

```python
import os, sys
sys.path.insert(0, "/app")
os.environ["CONFIG"] = "/app/tests/test.env"   # BEFORE importing app modules (tests/conftest.py:8-10)
import sqlalchemy
from app.db import Session, engine
from server import create_app
from init_app import add_sl_domains, add_proton_partner
app = create_app()                              # tests/conftest.py:23
with engine.connect() as conn:                  # pg_trgm (tests/conftest.py:28-36)
    try:
        conn.execute("DROP EXTENSION if exists pg_trgm"); conn.execute("CREATE EXTENSION pg_trgm")
    except Exception:
        conn.execute("Rollback")
add_sl_domains(); add_proton_partner()          # tests/conftest.py:38-39
with app.app_context():
    ...  # all observations run here
```

**Boot proof + effective config (exact command and output; SL DEBUG log lines included verbatim):**

```
$ docker exec sl-work bash -lc 'cd /app && python /tmp/obs_doc_main.py 2>&1'
load config file /app/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/dvryfrzplnfciylylzlq
Upload files to local dir
>>> init logging <<<
2026-07-08 05:38:35,883 - SL - DEBUG - 7343 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-08 05:38:36,929 - SL - DEBUG - 7343 - "/app/init_app.py:42" - add_sl_domains() -  - d1.test is already a SL domain
2026-07-08 05:38:36,930 - SL - DEBUG - 7343 - "/app/init_app.py:42" - add_sl_domains() -  - d2.test is already a SL domain
2026-07-08 05:38:36,931 - SL - DEBUG - 7343 - "/app/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-08 05:38:36,933 - SL - DEBUG - 7343 - "/app/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
BOOT_OK python 3.10.18 sqlalchemy 1.3.24
EMAIL_DOMAIN = sl.local | NOT_SEND_EMAIL = True | DMARC_CHECK_ENABLED = True | ENFORCE_SPF = False | ENABLE_SPAM_ASSASSIN = False
```

The remaining output of `obs_doc_main.py` (the SEED + R1 + R2 + R3 sections) is reproduced verbatim under R1–R3 below.

---

## R1 — Reply-email derivation

**Direct answer:** The reply email is taken **directly from the SMTP envelope recipient** (`rcpt_to`, i.e. `RCPT TO`). `handle_reply()` assigns `reply_email = rcpt_to` [`email_handler.py:972`], then normalizes it with `reply_email = normalize_reply_email(reply_email)` [`email_handler.py:984`]. It is **envelope-driven, not header-driven**: `handle()` passes `envelope.rcpt_tos` down to `handle_reply(envelope, copy_msg, rcpt_to)` [`email_handler.py:2199`], reached only when `is_reverse_alias(rcpt_to)` is true [`email_handler.py:2195`]. The function performing the work is `email_handler.handle_reply` (def [`email_handler.py:966`]) and `app.email_validation.normalize_reply_email` (def [`app/email_validation.py:25`]).

The observation seeds `User A → Mailbox → Alias → Contact` and lets `app.contact_utils.create_contact()` [`app/contact_utils.py:42`] internally call `generate_reply_email()` [`app/email_utils.py:1103`], producing a **real** reverse alias `R` on `sl.local`. Then it drives the **real entry point** `email_handler.handle(envelope, msg)`. Relevant payload lines from `/tmp/obs_doc_main.py`:

```python
user_A = create_new_user()                                   # tests/utils.py:17
alias_A = Alias.create_new_random(user_A); Session.commit()
res = contact_utils.create_contact(email="sender@external.test", alias=alias_A); Session.commit()
contact_A = res.contact; R = contact_A.reply_email
msg = mk_msg(user_A.email, R, "<obs-r1@sl.local>")           # From=mailbox addr, To=R
env = mk_env(user_A.email, R)                                # envelope.mail_from=mailbox, rcpt_tos=[R]
result = email_handler.handle(env, msg)                      # real dispatch -> handle_reply [2195,2199]
print("normalize_reply_email(R) =", repr(normalize_reply_email(R)), "| equal to R:", normalize_reply_email(R) == R)
```

**Command + complete, unedited output** (continuation of `obs_doc_main.py`; SL log lines are the app's own runtime logging):

```
$ docker exec sl-work bash -lc 'cd /app && python /tmp/obs_doc_main.py 2>&1'
...
########## SEED (User A -> Mailbox -> Alias -> Contact) ##########
2026-07-08 05:38:37,252 - SL - INFO - 7343 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 05:38:37,267 - SL - DEBUG - 7343 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email susses_booger149@sl.local
2026-07-08 05:38:37,276 - SL - INFO - 7343 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 05:38:37,311 - SL - DEBUG - 7343 - "/app/app/contact_utils.py:110" - create_contact() -  - Created contact <Contact 181 sender@external.test 950> for alias <Alias 950 susses_booger149@sl.local> with email sender@external.test invalid_email=False
user_A.id = 575 email = user_frnysobzgt@mailbox.test
alias_A.id = 950 email = susses_booger149@sl.local
create_contact -> created = True error = None
contact_A.id = 181 alias_id = 950 user_id = 575 website_email = sender@external.test
R (contact_A.reply_email) = 'sender_at_external_test_nimoezhdz@sl.local'
alias_A.mailbox.id = 690 email = user_frnysobzgt@mailbox.test

########## R1: reply-email derivation (real entry point handle()) ##########
envelope.mail_from = 'user_frnysobzgt@mailbox.test'
envelope.rcpt_tos  = ['sender_at_external_test_nimoezhdz@sl.local']
2026-07-08 05:38:37,318 - SL - DEBUG - 7343 - "/app/email_handler.py:1963" - handle() -  - Cannot parse Postfix queue ID from None None
2026-07-08 05:38:37,319 - SL - DEBUG - 7343 - "/app/email_handler.py:1980" - handle() -  - ==>> Handle mail_from:user_frnysobzgt@mailbox.test, rcpt_tos:['sender_at_external_test_nimoezhdz@sl.local'], header_from:user_frnysobzgt@mailbox.test, header_to:sender_at_external_test_nimoezhdz@sl.local, cc:None, reply-to:None, message_id:<obs-r1@sl.local>, client_ip:None, headers:[('From', 'user_frnysobzgt@mailbox.test'), ('To', 'sender_at_external_test_nimoezhdz@sl.local'), ('Message-ID', '<obs-r1@sl.local>'), ('Subject', 'obs'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
2026-07-08 05:38:37,323 - SL - DEBUG - 7343 - "/app/email_handler.py:2196" - handle() -  - Reply phase user_frnysobzgt@mailbox.test(user_frnysobzgt@mailbox.test) -> sender_at_external_test_nimoezhdz@sl.local
2026-07-08 05:38:37,325 - SL - INFO - 7343 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() -  - DMARC check disabled
2026-07-08 05:38:37,344 - SL - DEBUG - 7343 - "/app/email_handler.py:1051" - handle_reply() -  - Create <EmailLog 362> for <Contact 181 sender@external.test 950>, <User 575 Test User user_frnysobzgt@mailbox.test>, <Mailbox 690 user_frnysobzgt@mailbox.test>
2026-07-08 05:38:37,353 - SL - DEBUG - 7343 - "/app/email_handler.py:380" - replace_header_when_reply() -  - Replace To header, old: sender_at_external_test_nimoezhdz@sl.local, new: sender@external.test
2026-07-08 05:38:37,366 - SL - DEBUG - 7343 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'obs', from 'susses_booger149@sl.local' to 'sender@external.test'
email_handler.handle(env, msg) RETURNED: '250 Message accepted for delivery'
normalize_reply_email(R) = 'sender_at_external_test_nimoezhdz@sl.local' | equal to R: True
```

**What this proves (cause → effect):** the `handle()` dispatch logged `Reply phase ... -> sender_at_external_test_nimoezhdz@sl.local` from `email_handler.py:2196`, confirming `is_reverse_alias(rcpt_to)` routed the envelope into the reply phase. The reply email used downstream is exactly the envelope recipient `R = 'sender_at_external_test_nimoezhdz@sl.local'`; because `R` contains only allowed characters, `normalize_reply_email(R) == R`.

### R1 sibling variant — normalization collision

`normalize_reply_email()` [`app/email_validation.py:25`] first passes non-ASCII input through `convert_to_id` [`app/email_validation.py:27-28`], then replaces **every** character not in `_ALLOWED_CHARS` (`"abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789_-.+@"` [`app/email_validation.py:9`]) with `"_"` [`app/email_validation.py:33-34`]. Two distinct reverse-alias strings differing only in disallowed characters at the same positions therefore **collapse to one lookup key**.

**Command + complete, unedited output** (from `obs_doc_main.py`):

```
---- R1 normalization collision (normalize_reply_email) ----
'ab#cd@sl.local' -> 'ab_cd@sl.local'
'ab$cd@sl.local' -> 'ab_cd@sl.local'
'ab cd@sl.local' -> 'ab_cd@sl.local'
```

**Cause → effect:** three distinct inputs (`#`, `$`, and space are all outside `_ALLOWED_CHARS`) each map to a single normalized key `ab_cd@sl.local`. Since the contact lookup keys on the **normalized** value [`email_handler.py:984,986`], distinct reverse aliases can be funneled to the same `Contact.get_by(reply_email=...)` key — an additional path (besides duplicate rows) by which the reply-email space is not one-to-one.

---

## R2 — Contact identification by `reply_email`

**Direct answer:** The normalized reply email is resolved to a contact with `contact = Contact.get_by(reply_email=reply_email)` [`email_handler.py:986`]. `ModelMixin.get_by` (the method doing the work) is defined as `return Session.query(cls).filter_by(**kw).first()` [`app/models.py:83-84`] — i.e. a **`LIMIT 1`** query with **NO `ORDER BY`**. When more than one row matches, PostgreSQL is free to return any one of them.

**Resolved ids (from `obs_doc_main.py`):**

```
########## R2: contact identification by reply_email ##########
Contact.get_by(reply_email=R) -> id = 181 alias_id = 950 user_id = 575
```

**Proof of the SQL shape** — captured via a SQLAlchemy `before_cursor_execute` event listener wrapped around the real `Contact.get_by(reply_email=R)` call (payload + complete output):

```python
captured = []
def _bce(conn, cursor, statement, parameters, context, executemany):
    captured.append((statement, parameters))
event.listen(engine, "before_cursor_execute", _bce)
_ = Contact.get_by(reply_email=R)
event.remove(engine, "before_cursor_execute", _bce)
```

```
---- R2 actual emitted SQL (before_cursor_execute listener) ----
SQL: SELECT contact.id AS contact_id, contact.created_at AS contact_created_at, contact.updated_at AS contact_updated_at, contact.user_id AS contact_user_id, contact.alias_id AS contact_alias_id, contact.name AS contact_name, contact.website_email AS contact_website_email, contact.website_from AS contact_website_from, contact.reply_email AS contact_reply_email, contact.is_cc AS contact_is_cc, contact.pgp_public_key AS contact_pgp_public_key, contact.pgp_finger_print AS contact_pgp_finger_print, contact.mail_from AS contact_mail_from, contact.invalid_email AS contact_invalid_email, contact.block_forward AS contact_block_forward, contact.automatic_created AS contact_automatic_created, contact.flags AS contact_flags FROM contact WHERE contact.reply_email = %(reply_email_1)s LIMIT %(param_1)s
PARAMS: {'reply_email_1': 'sender_at_external_test_nimoezhdz@sl.local', 'param_1': 1}
CONTAINS 'ORDER BY': False
CONTAINS 'LIMIT': True
```

**Cause → effect:** the emitted statement ends in `... WHERE contact.reply_email = %(reply_email_1)s LIMIT %(param_1)s` with `param_1 = 1` and contains **no `ORDER BY`**. This is the exact `.first()` behavior of `ModelMixin.get_by` [`app/models.py:84`]. With a unique `reply_email` this is harmless; with duplicates it selects an unspecified row (see R4–R7).

**System-wide pattern (read-only observation).** The same non-unique-`reply_email` + `.first()` assumption appears at sibling call sites (all use `Contact.get_by(reply_email=...)`): `replace_header_when_reply` [`email_handler.py:364`], `handle_reply` [`email_handler.py:986`], the bounce/complaint phases [`email_handler.py:1998,2009,2167`], `is_reverse_alias` [`app/email_utils.py:1156`], and `available_sl_email` [`app/models.py:1425`]. `is_reverse_alias` was observed returning `False` for an unknown reverse alias in the E502 case below.

---

## R3 — Forwarding-destination (user / alias / mailbox) selection

**Direct answer:** The destination **alias** is `contact.alias` (`alias = contact.alias` [`email_handler.py:994`]); the destination **user** is `alias.user` (`user = alias.user` [`email_handler.py:1004`]); the delivery **mailbox** is `get_mailbox_from_mail_from(mail_from, alias)` [`email_handler.py:1019`, def `:1364`]. On success an `EmailLog` row is written [`email_handler.py:1043`] carrying `contact_id`, `alias_id`, `user_id`, `mailbox_id` — the audit trail of exactly which user/mailbox the reply resolved to.

`get_mailbox_from_mail_from` (def [`email_handler.py:1364`]) matches `mail_from` against `alias.mailboxes` and each mailbox's `authorized_addresses`, first with the raw value then with `canonicalize_email(mail_from)`; it returns `None` when nothing matches **(inferred from reading `email_handler.py:1364-1387`, and confirmed by the authorized/unauthorized runtime results below)**.

**Command + complete, unedited output** (from `obs_doc_main.py`):

```
########## R3: forwarding-destination selection ##########
contact.alias  -> id = 950 email = susses_booger149@sl.local (email_handler.py:994)
alias.user     -> id = 575 email = user_frnysobzgt@mailbox.test (email_handler.py:1004)
get_mailbox_from_mail_from(AUTHORIZED mail_from=user_A.email) -> id=690 email=user_frnysobzgt@mailbox.test
get_mailbox_from_mail_from(UNAUTHORIZED mail_from=unauthorized@gmail.com) -> None
2026-07-08 05:38:37,378 - SL - INFO - 7343 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() -  - DMARC check disabled
2026-07-08 05:38:37,381 - SL - DEBUG - 7343 - "/app/email_handler.py:1051" - handle_reply() -  - Create <EmailLog 363> for <Contact 181 sender@external.test 950>, <User 575 Test User user_frnysobzgt@mailbox.test>, <Mailbox 690 user_frnysobzgt@mailbox.test>
2026-07-08 05:38:37,400 - SL - DEBUG - 7343 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'obs', from 'susses_booger149@sl.local' to 'sender@external.test'
handle_reply(envelope, msg, R) direct -> (True, '250 Message accepted for delivery')
EmailLog created -> id = 363 contact_id = 181 alias_id = 950 user_id = 575 mailbox_id = 690 is_reply = True
```

**Destination triple (happy path):** user **575** (`user_frnysobzgt@mailbox.test`) / alias **950** (`susses_booger149@sl.local`) / mailbox **690** (`user_frnysobzgt@mailbox.test`). The authorized `mail_from` (the mailbox's own address) returns mailbox **690**; the unauthorized `mail_from` (`unauthorized@gmail.com`) returns **None** — matching the two branches of `get_mailbox_from_mail_from`. The successful reply returns `(True, '250 Message accepted for delivery')` and writes **`EmailLog 363`** (`contact_id=181, alias_id=950, user_id=575, mailbox_id=690, is_reply=True`). Because `NOT_SEND_EMAIL=true` [`tests/test.env:7`], the message is **printed** (`mail_sender.py:131 ... send email ... from 'susses_booger149@sl.local' to 'sender@external.test'`) rather than transmitted.

> Note: the `250 Message accepted for delivery` string is `status.E200` [`app/email/status.py:2`]. The reply handler's own success return for the SPF-forced branch is `status.E201`; the normal successful path returns the delivery-accepted status observed here.

---

## R4 — Behavior across multiple reply events over time

**Direct answer:** With a **single** contact per `reply_email`, repeated identical reply events resolve **stably** to the same contact/user, because the one matching row is the only candidate the unordered `.first()` can return. The scripts `/tmp/obs_doc_wrong.py` (in-process) and `/tmp/obs_doc_proc.py` (separate processes) drive the SAME unchanged input `R` repeatedly.

**BEFORE (single Contact for User A) — command + complete, unedited output:**

```
$ docker exec sl-work bash -lc 'cd /app && python /tmp/obs_doc_wrong.py 2>/dev/null'
R = 'sender_at_external_test_nimoezhdz@sl.local' | normalize==R: True
########## R4 BEFORE (single Contact for User A) ##########
rows WHERE reply_email=R (ctid,id,alias_id,user_id): [('(1,24)', 181, 950, 575)]
  run 1: Contact.get_by(reply_email=R) -> id=181 user_id=575
  run 2: Contact.get_by(reply_email=R) -> id=181 user_id=575
  run 3: Contact.get_by(reply_email=R) -> id=181 user_id=575
```

**AFTER a duplicate exists (25 identical in-process lookups) — command + complete, unedited output:**

```
########## R4 AFTER within-process: 25 identical get_by ##########
distribution of resolved user_id over 25 identical Contact.get_by(reply_email=R): {575: 25}
EXPLAIN (default plan):
    Limit  (cost=0.00..2.60 rows=1 width=4)
      ->  Seq Scan on contact  (cost=0.00..2.60 rows=1 width=4)
            Filter: ((reply_email)::text = 'sender_at_external_test_nimoezhdz@sl.local'::text)
EXPLAIN (enable_seqscan=off -> index path):
    Limit  (cost=0.14..8.16 rows=1 width=4)
      ->  Index Scan using ix_contact_reply_email on contact  (cost=0.14..8.16 rows=1 width=4)
            Index Cond: ((reply_email)::text = 'sender_at_external_test_nimoezhdz@sl.local'::text)
```

**Cause → effect:** with two matching rows present, 25 identical lookups within one connection all returned `user_id=575` — the resolution is **stable within a fixed physical layout / plan**, but neither `EXPLAIN` plan contains an `ORDER BY`: the default plan is a `Seq Scan` (returns the first tuple in **heap order**), and even the index path (`Index Scan using ix_contact_reply_email`) imposes no ordering on equal keys. Stability here is incidental to the current heap layout, not a guarantee — which R5 demonstrates by changing that layout.

---

## R5 — Divergence of a single reply email (cases a, b, c)

### R5(a) — the same reply email resolves to DIFFERENT contacts over time

**Direct answer: YES — observed.** After inserting a second contact for a different user with the same `reply_email` and then changing the heap layout, `Contact.get_by(reply_email=R)` returned a **different** `contact.id` (hence a different user).

**Command + complete, unedited output** (INTERMEDIATE insert + FLIP, from `obs_doc_wrong.py`):

```
########## INTERMEDIATE: insert 2nd Contact for a DIFFERENT user with SAME reply_email ##########
Contact.create(user_id=586, alias_id=972, website_email='other@external.test', reply_email=R, commit=True) -> SUCCESS contact_B.id=192 (NO IntegrityError => reply_email NOT unique)
rows WHERE reply_email=R now: [('(1,24)', 181, 950, 575), ('(1,35)', 192, 972, 586)]
########## FLIP (heap reorder: delete A, reinsert A2) -> AFTER ##########
get_by BEFORE flip -> id=181 user_id=575
deleted original Contact 181; reinserted A2 id=193 (same user_A=575 / alias_A=950)
rows WHERE reply_email=R after reorder: [('(1,35)', 192, 972, 586), ('(1,36)', 193, 950, 575)]
get_by AFTER flip -> id=192 user_id=586 alias_id=972
  alias-owner of R's original reverse alias = User A (id=575); resolved user now = 586 => WRONG USER if 586 != 575
########## R5(a) same reply email -> DIFFERENT contacts over time ##########
BEFORE flip get_by -> contact.id=181 (user 575) ; AFTER flip get_by -> contact.id=192 (user 586)
```

**Cross-process confirmation (6 SEPARATE processes, SAME unchanged input, post-flip) — command + complete, unedited output:**

```
$ docker exec sl-work bash -lc 'cd /app && for i in 1 2 3 4 5 6; do python /tmp/obs_doc_proc.py 2>/dev/null | grep PROC; done'
PROC pid=7497 Contact.get_by(reply_email='sender_at_external_test_nimoezhdz@sl.local') -> contact.id=192 user_id=586
PROC pid=7511 Contact.get_by(reply_email='sender_at_external_test_nimoezhdz@sl.local') -> contact.id=192 user_id=586
PROC pid=7525 Contact.get_by(reply_email='sender_at_external_test_nimoezhdz@sl.local') -> contact.id=192 user_id=586
PROC pid=7539 Contact.get_by(reply_email='sender_at_external_test_nimoezhdz@sl.local') -> contact.id=192 user_id=586
PROC pid=7553 Contact.get_by(reply_email='sender_at_external_test_nimoezhdz@sl.local') -> contact.id=192 user_id=586
PROC pid=7567 Contact.get_by(reply_email='sender_at_external_test_nimoezhdz@sl.local') -> contact.id=192 user_id=586
```

**Observed distribution of the SAME unchanged input `R` across the timeline:** **before** the duplicate + heap change → **25/25 in-process → User A (575)**; **after** → **6/6 separate processes → User B (586)** (and the in-process `get_by` after the flip also → User B). The physical trigger for the change: deleting the original contact `181` (heap slot `(1,24)`) and re-inserting `A2` appended it at slot `(1,36)`, **behind** User B's row at `(1,35)`, so the `Seq Scan` now returns User B's row first. Same query, same parameter, different result — the reported "sometimes X, sometimes Y" reproduced.

### R5(b) — temporary FAILURE to resolve

**Direct answer: YES — observed.** Between the moment `generate_reply_email()` returns a fresh reverse alias `R_new` and the moment a `Contact` row for it is committed, `Contact.get_by(reply_email=R_new)` returns `None`, and a reply to `R_new` fails with **E502**.

**Command + complete, unedited output** (from `obs_doc_wrong.py`, SL log lines filtered by `grep -v` for readability of the assertions; the E502 tuple is the handler's real return):

```
########## R5(b) temporary FAILURE to resolve (generate -> before commit window) ##########
generate_reply_email(...) -> R_new='brandnew_at_external_test_tnhlaylda@sl.local'
available_sl_email(R_new) = True
Contact.get_by(reply_email=R_new) = None
handle_reply(envelope, msg, R_new) -> (False, '550 SL E502 Email not exist')
```

**Cause → effect:** `generate_reply_email()` [`app/email_utils.py:1103`] returns the first candidate `R_new` for which `available_sl_email(R_new)` is `True` [`app/email_utils.py:1150`]; in the window before `Contact.create()` commits, no row exists, so `Contact.get_by(reply_email=R_new)` is `None` and `handle_reply` takes the no-contact branch `return False, status.E502` [`email_handler.py:989`].

### R5(c) — resolves correctly but forwards to a DIFFERENT user

**Direct answer: YES — observed.** Once the duplicate exists and the heap favors User B, a real reply driven through `email_handler.handle(...)` is **accepted and delivered to User B's mailbox**, not the alias owner (User A). The `EmailLog` proves the wrong-user delivery.

**Command + complete, unedited output** (real entry point `handle()` with `mail_from=user_B.email`, from `obs_doc_wrong.py`):

```
########## R5(c) resolves-but-WRONG-user via REAL entry point handle() ##########
email_handler.handle(env, msg) [mail_from=user_B.email] RETURNED: '250 Message accepted for delivery'
EmailLog -> id=365 contact_id=192 alias_id=972 user_id=586 mailbox_id=701 (delivered to User B=586, NOT alias owner A=575)
```

**Cause → effect:** `handle()` saw `is_reverse_alias(R)` true (a contact for `R` exists), routed to `handle_reply`, which resolved `contact = 192` (User B) via the unordered `.first()`, set `user = contact.alias.user = 586` [`email_handler.py:1004`], selected User B's mailbox `701`, and wrote **`EmailLog 365`** (`user_id=586, mailbox_id=701`). The reply intended for User A's reverse alias was delivered to **User B**.

---

## R6 — Race conditions, uniqueness assumption, and timing

**Direct answer:** The reply-handling code **assumes `reply_email` is unique** (it resolves with a single-row `.first()` [`app/models.py:84`]), but the **schema does NOT enforce it**: `reply_email = sa.Column(sa.String(512), nullable=False, index=True)` [`app/models.py:1899`] is indexed only, created `unique=False` [`migrations/versions/2021_071310_78403c7b8089_.py:22`]; the only uniqueness is `uq_contact(alias_id, website_email)` [`app/models.py:1875`]. Creation is a **check-then-insert (TOCTOU)**: `available_sl_email()` is a read-only existence check [`app/models.py:1425`], and `create_contact()`'s `IntegrityError` handler recovers **only** the `uq_contact` violation by re-fetching `Contact.get_by(alias_id=alias.id, website_email=email)` [`app/contact_utils.py:113-118`] — it cannot catch or prevent a duplicate `reply_email`.

**Command + complete, unedited output** (from `obs_doc_wrong.py`):

```
########## R6 TOCTOU / uniqueness assumption ##########
available_sl_email(R) [R used by 2 contacts] = False
available_sl_email('unused_marker_key_9988@sl.local') = True
uq_contact duplicate (alias_id=972, website_email='other@external.test') -> IntegrityError: duplicate key value violates unique constraint "uq_contact"
worker P available_sl_email(R_toc='toc_at_external_test_kadcwnoefh@sl.local')=True ; worker Q available_sl_email(R_toc)=True  (both see 'free' -> TOCTOU window)
both Contact.create SUCCEEDED -> rows reply_email=R_toc (id,user_id): [(195, 587), (196, 588)] => duplicate reply_email, NO error
```

**Cause → effect, three grounded facts:**
1. **`reply_email` is not guarded.** Inserting a second contact with the same `reply_email` (for a different user/alias) **succeeded with no error** — see the INTERMEDIATE output in R5(a): `Contact.create(... reply_email=R ...) -> SUCCESS contact_B.id=192 (NO IntegrityError)`. The DB accepts duplicates because the index is `unique=False` [`migration:22`; `app/models.py:1899`].
2. **`uq_contact` *is* enforced** — and is the *only* uniqueness. Attempting a duplicate `(alias_id=972, website_email='other@external.test')` raised `IntegrityError: duplicate key value violates unique constraint "uq_contact"` [`app/models.py:1875`]. This is precisely (and only) what `create_contact`'s `except IntegrityError` recovers [`app/contact_utils.py:113-118`].
3. **The check-then-insert window is real.** Two "workers" P and Q both observed `available_sl_email(R_toc) == True` (the read-only check at `app/models.py:1425`) and then **both** `Contact.create()` succeeded, yielding two rows `(195, 587)` and `(196, 588)` with the same `reply_email` and **no error**. `available_sl_email()` provides no locking and there is no DB constraint to backstop it, so concurrent generation can mint duplicate reply emails — the same duplication that makes `.first()` non-deterministic.

---

## R7 — Observed runtime values → behavior

**Direct answer:** the concrete values captured across the run, and how they produce the wrong-user behavior.

| Item | Observed value | Grounding |
|------|----------------|-----------|
| Extracted `reply_email` `R` | `sender_at_external_test_nimoezhdz@sl.local` | `contact_A.reply_email`; `reply_email = rcpt_to` [`email_handler.py:972`] |
| Normalized value | `sender_at_external_test_nimoezhdz@sl.local` (== `R`) | `normalize_reply_email` [`email_handler.py:984`] |
| **User-A** row (original) | `contact.id=181`, `alias_id=950`, `user_id=575` | `obs_doc_main.py` R2 output |
| User A alias / mailbox | alias `950` `susses_booger149@sl.local` / mailbox `690` `user_frnysobzgt@mailbox.test` | R3 output |
| User-A row after flip (`A2`) | `contact.id=193`, `alias_id=950`, `user_id=575` | `obs_doc_wrong.py` FLIP output |
| **User-B** row (duplicate `reply_email=R`) | `contact.id=192`, `alias_id=972`, `user_id=586` | `obs_doc_wrong.py` INTERMEDIATE/R7 output; alias `972` `decals_spacey297@sl.local` |
| Forwarding **before** duplicate | user `575` / alias `950` / mailbox `690` (`EmailLog 363`) | R3 output |
| Forwarding **after** flip | user `586` / alias `972` / mailbox `701` (`EmailLog 365`) | R5(c) output |
| Lookup distribution (same `R`) | pre-flip 25/25 → User A `575`; post-flip 6/6 procs → User B `586` | R4 + R5(a) outputs |

```
########## R7 CONSOLIDATED OBSERVED VALUES ##########
reply_email R = 'sender_at_external_test_nimoezhdz@sl.local' (normalized == R)
User-B row : contact.id=192 alias_id=972 user_id=586 (alias decals_spacey297@sl.local)
User-A(A2) row : contact.id=193 alias_id=950 user_id=575
post-flip resolved: user_id=586 alias_id=972 (EmailLog id=365 mailbox_id=701)
```

**How these values produce the behavior (cause → effect):** the reply address `R` is non-unique across contacts `181/193` (User A) and `192` (User B). `handle_reply` resolves the destination purely from whichever single row `Contact.get_by(reply_email=R).first()` returns [`email_handler.py:986`; `app/models.py:84`]; since the query has no `ORDER BY`, PostgreSQL returns the first row in the current physical/scan order. When that order favored User A's row, the reply went to user `575`/mailbox `690`; after the heap changed, it favored User B's row, and the very same reply went to user `586`/mailbox `701` — a delivery to a user **other than the alias owner**.

---

## Guarded-exit matrix (every distinct condition)

Each row was reproduced through the real reply handler `email_handler.handle_reply(...)` — the production function invoked by `handle()` at `email_handler.py:2199`. Calling it directly is required for E501 and the no-contact E502 (for those, `is_reverse_alias(rcpt_to)` is false, so `handle()` would route to the forward phase; the direct call exercises the exact reply guard). All status strings match `app/email/status.py` exactly.

**Command:** `docker exec sl-work bash -lc 'cd /app && python /tmp/obs_doc_exits.py > /tmp/obs_doc_exits.out 2>&1'` — **complete, unedited assertion output:**

```
=== GUARDED-EXIT MATRIX (all via email_handler.handle_reply, the production fn invoked by handle() at :2199) ===
E501 (bad reply domain): rcpt_to='randabc123@unknown-domain.example' -> (False, '550 SL E501')   [return email_handler.py:981 | status.py:38]
   is_reverse_alias('nonexistent_reverse_xyz@sl.local') = False
E502 (no contact): rcpt_to='nonexistent_reverse_xyz@sl.local' -> (False, '550 SL E502 Email not exist')   [return :989 | status.py:39]
   user.id=580 delete_on(future)=2026-08-07T05:40:38.048831+00:00 -> is_active()=False
E502 (inactive user): reply_email='peer_at_external_test_wtuwsem@sl.local' -> (False, '550 SL E502 Email not exist')   [return :992 | status.py:39]
   alias.email='orphan_e7d78d4b@removed-7a01924e.example' -> is_valid_alias_address_domain=False
E503 (alias domain unknown): reply_email='peer_at_external_test_emcza@sl.local' -> (False, '550 SL E503')   [return :1002 | status.py:40]
   user.id=582 disabled=True is_active()=True can_send_or_receive()=False
E504 (account disabled): reply_email='peer_at_external_test_ouzijn@sl.local' -> (False, '550 SL E504 Account disabled')   [return :1009 | status.py:41]
   mail_from='stranger@evil.example' authorized? mailbox=None ; alias.disable_email_spoofing_check=False
E214 (unauthorized mailbox): mail_from='stranger@evil.example' -> (False, '250 SL E214 Unauthorized for using reverse alias')   [return :1034 | status.py:22]
   [NON-CANONICAL] ENFORCE_SPF True (default False), mailbox.force_spf=True, spf_pass monkeypatched->False
E201 (SPF fail) [NON-CANONICAL]: -> (True, '250 SL E201')   [return :1040 | status.py:3]
   [NON-CANONICAL] ENABLE_SPAM_ASSASSIN True (default False), SPAMASSASSIN_HOST=None, get_spam_info monkeypatched->(True,'forced-spam')
E506 (spam) [NON-CANONICAL]: -> (False, '550 SL E506 Email detected as spam')   [return :1094 | status.py:43]
```

| Exit | Condition / setup (observed) | Observed `(bool, status)` | return `file:line` | status const |
|------|------------------------------|---------------------------|--------------------|--------------|
| **E501** | `rcpt_to` domain ≠ `EMAIL_DOMAIN` and no matching `SLDomain` (`randabc123@unknown-domain.example`) | `(False, '550 SL E501')` | `email_handler.py:981` | `app/email/status.py:38` |
| **E502** (no contact) | well-formed `@sl.local` reverse alias with no `Contact`; `is_reverse_alias(...) = False` | `(False, '550 SL E502 Email not exist')` | `email_handler.py:989` | `app/email/status.py:39` |
| **E502** (inactive user) | contact exists but owner soft-deleted: `delete_on` in the future → `is_active()=False` (`User.is_active` [`app/models.py:766-769`]) | `(False, '550 SL E502 Email not exist')` | `email_handler.py:992` | `app/email/status.py:39` |
| **E503** | alias on a domain SL doesn't manage → `is_valid_alias_address_domain(alias.email)=False` (`app/email_utils.py:557`); **CONTROLLED**: simulated de-managed alias domain | `(False, '550 SL E503')` | `email_handler.py:1002` | `app/email/status.py:40` |
| **E504** | owner `disabled=True`, `delete_on=None` → `is_active()=True` but `can_send_or_receive()=False` (`app/models.py:886-895`) | `(False, '550 SL E504 Account disabled')` | `email_handler.py:1009` | `app/email/status.py:41` |
| **E214** | `mail_from` not a mailbox/authorized address; `alias.disable_email_spoofing_check=False` → `get_mailbox_from_mail_from`=None → `handle_unknown_mailbox` | `(False, '250 SL E214 Unauthorized for using reverse alias')` | `email_handler.py:1034` | `app/email/status.py:22` |
| **E201** *(NON-CANONICAL)* | `ENFORCE_SPF=True` (default False [`app/config.py:132`]), `mailbox.force_spf=True`, `spf_pass` monkeypatched → False | `(True, '250 SL E201')` | `email_handler.py:1040` | `app/email/status.py:3` |
| **E506** *(NON-CANONICAL)* | `ENABLE_SPAM_ASSASSIN=True` (default False [`app/config.py:450`]), `SPAMASSASSIN_HOST=None`, `get_spam_info` monkeypatched → `(True,'forced-spam')` | `(False, '550 SL E506 Email detected as spam')` | `email_handler.py:1094` | `app/email/status.py:43` |

**NON-CANONICAL labeling:** E201 and E506 are gated by env-derived flags that are **off** in the default configuration (`ENFORCE_SPF` / `ENABLE_SPAM_ASSASSIN` are `"<NAME>" in os.environ` [`app/config.py:132,450`]). To exercise the **real branches** at `email_handler.py:1040` and `:1094`, those module flags were forced on and the leaf functions (`spf_pass`, `get_spam_info`) were monkeypatched to return failing values; each was **restored** immediately after the case. Every other exit (E501/E502×2/E503/E504/E214) is fully canonical. `is_valid_alias_address_domain` returning False for E503 was produced by pointing the alias at an unmanaged domain (the exact scenario the sanity check guards).

**DMARC note.** Because `DMARC_CHECK_ENABLED=true` [`tests/test.env:66`], `apply_dmarc_policy_for_reply_phase(...)` runs at `email_handler.py:1012` (def `app/handler/dmarc.py:154`) **before** the mailbox stage. For the plain crafted replies it returned `None` (pass), letting execution reach the mailbox check — observed in the R1/R3 runs as the log line `apply_dmarc_policy_for_reply_phase() ... DMARC check disabled` (`dmarc.py:159`) immediately preceding `EmailLog.create`. So DMARC did **not** block the crafted reply.

---

## State-transition report

**Wrong-user reproduction — `contact` rows matching `reply_email=R` (ctid, id, alias_id, user_id) and the resolved user at each stage:**

| Stage | `contact` rows for `R` | `Contact.get_by(reply_email=R)` resolves to | forwarding user |
|-------|------------------------|---------------------------------------------|-----------------|
| **Before** (single contact) | `[(1,24) id181 alias950 user575]` | contact `181` → user `575` (25/25 in-process) | **User A (575)** — correct |
| **Intermediate** (duplicate inserted) | `[(1,24) id181 alias950 user575, (1,35) id192 alias972 user586]` | contact `181` → user `575` (still, heap unchanged) | User A (575) |
| **After** (heap reorder: delete `181`, reinsert `A2`=`193`) | `[(1,35) id192 alias972 user586, (1,36) id193 alias950 user575]` | contact `192` → user `586` (in-process **and** 6/6 processes) | **User B (586)** — WRONG |

**Contact-creation window (R5(b)) — resolution of a freshly generated reverse alias `R_new`:**

| Stage | `available_sl_email(R_new)` | `Contact.get_by(reply_email=R_new)` | `handle_reply(...)` |
|-------|-----------------------------|-------------------------------------|---------------------|
| After `generate_reply_email()`, before `Contact.create()` commit | `True` | `None` | `(False, '550 SL E502 Email not exist')` |
| After commit **(inferred)** | `False` | the new `Contact` | proceeds past the no-contact guard |

> The "after commit" row is **(inferred)** from the code path (`available_sl_email` returns False once any contact uses the address [`app/models.py:1425`], and `get_by` would then find the row); the un-committed window itself was observed directly (row above).

---

## Root-cause synthesis

The wrong-user delivery is the product of three independently observed facts that compose into one defect:

1. **Non-unique key.** `Contact.reply_email` is `index=True` but **not unique** [`app/models.py:1899`], created `unique=False` [`migrations/versions/2021_071310_78403c7b8089_.py:22`]. The only uniqueness on `contact` is `uq_contact(alias_id, website_email)` [`app/models.py:1875`] — which does **not** constrain `reply_email`. → *Observed:* a second contact for a different user with the same `reply_email` inserts with **no error** (R5(a) INTERMEDIATE; R6).
2. **Unordered single-row lookup.** The reply handler resolves the contact with `Contact.get_by(reply_email=...)` = `Session.query(cls).filter_by(**kw).first()` [`email_handler.py:986`; `app/models.py:83-84`] — `LIMIT 1`, **no `ORDER BY`**. → *Observed:* captured SQL has `LIMIT` and no `ORDER BY` (R2); `EXPLAIN` shows a `Seq Scan`/`Index Scan` with no ordering (R4).
3. **Check-then-insert with no backstop.** Reverse-alias minting is `available_sl_email()` (read-only check [`app/models.py:1425`]) → `generate_reply_email()` [`app/email_utils.py:1103`] → `Contact.create()`, and the `IntegrityError` handler recovers only `uq_contact` [`app/contact_utils.py:113-118`]. → *Observed:* two workers both pass the check and both insert the same `reply_email` (R6 TOCTOU).

**Composition:** (1) permits duplicate `reply_email` rows across different users; (2) then selects an **arbitrary** one whose identity tracks the physical heap order; and heap order changes with insert order, delete/reinsert, updates, and `VACUUM`/autovacuum. So a reply to a single reverse alias can resolve to `contact.alias.user` [`email_handler.py:1004`] of the **wrong** user, and — as observed — be delivered to that user's mailbox with a corroborating `EmailLog` (R5(c): `EmailLog 365`, `user_id=586`, `mailbox_id=701`). The code's implicit assumption that `reply_email` uniquely identifies a contact is not backed by the schema.

*(No remediation is proposed or applied — per the read-only scope of this investigation.)*

---

## Web-research corroboration (secondary — the runtime evidence above is primary)

The two database mechanisms behind the observed non-determinism are documented behavior, corroborating the runtime findings (the app runs on PostgreSQL 15.13):

- **No row order without `ORDER BY`.** PostgreSQL states that without sorting, `"the rows will be returned in an unspecified order"` and depends on the scan/plan and on-disk order (`postgresql.org/docs/current/queries-order.html`). For `LIMIT`, it warns that `"SQL does not promise to deliver the results of a query in any particular order unless ORDER BY is used"`, and that repeated executions of the same `LIMIT` query can return different subsets (`postgresql.org/docs/15/queries-limit.html`, `postgresql.org/docs/current/sql-select.html`). This is exactly the `.first()` (`LIMIT 1`, no `ORDER BY`) shape captured in R2.
- **Check-then-insert (TOCTOU) is race-unsafe without a DB constraint.** The naive SELECT-then-INSERT pattern lets two workers both observe "absent" and both insert; only a unique constraint or explicit locking makes it safe. This matches the `available_sl_email()` → `Contact.create()` window observed in R6, where `reply_email` has no unique constraint.

---

## Coverage pass

Re-reading the question, every named item is answered with a concrete value, a `file:line`, observed evidence, sibling variants, and a causal reason:

- **R1 — reply-email derivation** ✅ `reply_email = rcpt_to` [`email_handler.py:972`] then `normalize_reply_email` [`:984`; `app/email_validation.py:25`]; `R='sender_at_external_test_nimoezhdz@sl.local'`, `normalize(R)==R`. Sibling: normalization collision (`ab#cd`/`ab$cd`/`ab cd` → `ab_cd@sl.local`). Envelope-driven (`handle()` `:2195,2199`).
- **R2 — contact identification** ✅ `Contact.get_by(reply_email=...)` [`email_handler.py:986`] = `filter_by(**kw).first()` [`app/models.py:83-84`]; captured SQL proves `LIMIT 1`, no `ORDER BY`; resolved `contact.id=181/alias 950/user 575`. Siblings: `:364,1998,2009,2167`, `email_utils.py:1156`, `models.py:1425`.
- **R3 — forwarding destination** ✅ alias `contact.alias` [`:994`], user `alias.user` [`:1004`], mailbox `get_mailbox_from_mail_from` [`:1019,:1364`]; triple = user 575 / alias 950 / mailbox 690; `EmailLog 363`; authorized→690, unauthorized→None.
- **R4 — behavior across multiple events** ✅ single contact → stable 25/25 → User A; `EXPLAIN` shows no `ORDER BY`.
- **R5(a) different contacts over time** ✅ before `181`(575) → after `192`(586); 6/6 processes → 586.
- **R5(b) temporary failure to resolve** ✅ `get_by(R_new)=None` → `(False,'550 SL E502 Email not exist')` [`:989`].
- **R5(c) resolves but wrong user** ✅ real `handle()` delivered to User B: `EmailLog 365` `user_id=586 mailbox_id=701`.
- **R6 — race / uniqueness / timing** ✅ duplicate `reply_email` inserts with no error [`models.py:1899`, `migration:22`]; only `uq_contact` enforced [`models.py:1875`]; TOCTOU window observed (P & Q both insert `R_toc`); `IntegrityError` handler recovers only `uq_contact` [`contact_utils.py:113-118`].
- **R7 — observed values → behavior** ✅ consolidated table + cause→effect chain.
- **Guarded exits** ✅ E501, E502 (no-contact), E502 (inactive user), E503, E504, E214, E201 (NON-CANONICAL), E506 (NON-CANONICAL) — each with observed `(bool,status)` and `file:line`.
- **State transitions** ✅ before / intermediate / after for the wrong-user reproduction and the contact-creation window.

**Environment:** canonical container `sl-work`, Python 3.10.18, SQLAlchemy 1.3.24, PostgreSQL 15.13 on `:15432`, `CONFIG=tests/test.env`. **Scope:** read-only; no source file modified; no defect remediation; temporary observation scripts live under the container's `/tmp` and are removed after the run (see repository verification below).



