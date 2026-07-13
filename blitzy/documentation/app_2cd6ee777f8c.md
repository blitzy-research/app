# SimpleLogin Reply-Resolution Investigation — How an Inbound Reply Resolves to a Contact and Forwarding Destination (and Why It Can Forward to the Wrong User)

> **Scope:** Read-only root-cause investigation. This document is the *only* persistent artifact produced. No product/source/migration/test/configuration file was modified, and no remediation was performed. All temporary observation scripts were removed afterward (see §12 Repository Invariant).
>
> **Methodology:** Run-first. Every behavioral claim below is backed by the *actual, complete, unedited* output of a command that exercised the **canonical** inbound reply entry point `email_handler.handle_reply(envelope, msg, rcpt_to)` in the canonical Docker runtime (Python 3.10 + PostgreSQL 15 + Redis). Factual claims are grounded in a specific `file:line`. Statements not directly observed are labeled **`[inferred]`**; values obtained by bypassing the entry point are labeled **`[non-canonical]`**.

---

## 1. Lead Answer (Executive Summary)

**How an inbound reply resolves.** When a user replies to a *reverse-alias*, the inbound SMTP recipient (`rcpt_to`) is taken verbatim as the reply address (`reply_email = rcpt_to` — `email_handler.py:972`). It is gated against the service domain (`if not reply_email.endswith(EMAIL_DOMAIN)` — `email_handler.py:977`; falling back to an `SLDomain` lookup — `email_handler.py:978`; else `status.E501` — `email_handler.py:981`), then normalized (`reply_email = normalize_reply_email(reply_email)` — `email_handler.py:984`). The normalized value is used to resolve a single `Contact`: `contact = Contact.get_by(reply_email=reply_email)` (`email_handler.py:986`). The **forwarding destination is derived entirely from that resolved contact**: `alias = contact.alias` (`email_handler.py:994`), `user = alias.user` (`email_handler.py:1004`), and `mailbox = get_mailbox_from_mail_from(mail_from, alias)` (`email_handler.py:1019`). On success the resolution is persisted as `EmailLog.create(contact_id=contact.id, alias_id=contact.alias_id, is_reply=True, user_id=contact.user_id, mailbox_id=mailbox.id, ...)` (`email_handler.py:1042-1050`).

**Why it can forward to the wrong user.** The lookup `Contact.get_by(reply_email=...)` delegates to `ModelMixin.get_by()`, whose entire body is `return Session.query(cls).filter_by(**kw).first()` — a `.first()` with **no `ORDER BY`** (`app/models.py:83-84`). The `reply_email` column carries **no `UNIQUE` constraint**: the only unique constraint on `Contact` is `uq_contact` on `(alias_id, website_email)` (`app/models.py:1875`); `reply_email` is merely `index=True` (`app/models.py:1899`), and the migration that created its index sets `unique=False` (`migrations/versions/2021_071310_78403c7b8089_.py:22`). Generation-time uniqueness is only a best-effort, non-atomic TOCTOU check (`available_sl_email()` — `app/models.py:1425-1432`) that non-generator write paths bypass entirely. Consequently, **two `Contact` rows owned by different users can share one `reply_email`**, and on such a multi-row match `.first()` returns an **arbitrary, physical/plan-order-dependent** row. Because the alias and user are derived from that row, the reply is routed to — and logged against — **whichever user's contact the database happened to return first**, not necessarily the alias owner whose mailbox actually sent the reply.

**Headline observed evidence.** Seeding two contacts that share one `reply_email` and then driving the *identical* canonical input `N=100` times:

- Insertion order **A-then-B** → **100/100** resolved to the first-inserted contact, `user_A`.
- Insertion order **B-then-A** (identical `mail_from = user_A`'s mailbox, identical `rcpt_to`) → **100/100** resolved to `user_B` — **the different user**. The persisted `EmailLog.user_id` recorded `user_B` for all 100 events.

The resolved user is governed **purely by physical row order**, not by the alias owner or the sender. This is the wrong-user condition, reproduced directly. Full outputs are in §7; the causal chain is in §6.

---

## 2. Environment & Methodology

### 2.1 Canonical runtime

All observations were produced inside the canonical Docker container `sl_app` (image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`), which contains the SimpleLogin repository at commit `2cd6ee777f8c` — the same commit the assigned filename encodes.

**Command:**

```
docker exec sl_app bash -lc 'python --version 2>&1; echo "---"; . /app/venv/bin/activate; python --version 2>&1'
docker exec sl_app bash -lc 'pg_isready 2>&1; su postgres -c "psql -tAc \"select version();\"" 2>/dev/null | head -1'
docker exec sl_app bash -lc 'redis-cli ping 2>&1'
```

**Output (complete, unedited):**

```
Python 3.10.18
---
Python 3.10.18
--- postgres ---
/var/run/postgresql:5432 - accepting connections
PostgreSQL 15.13 (Debian 15.13-0+deb12u1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 12.2.0-14+deb12u1) 12.2.0, 64-bit
--- redis ---
PONG
```

This matches the AAP's canonical configuration (Python 3.10, PostgreSQL, Redis).

### 2.2 Canonical configuration values

The reply path depends on `EMAIL_DOMAIN` (the reply-domain gate) and `DB_URI`. These are sourced from the environment recipe `/tmp/sl_env.sh` derived from `tests/test.env`.

**Command:**

```
docker exec sl_app bash -lc 'set -a && . /tmp/sl_env.sh && set +a; echo "CONFIG=$CONFIG"; echo "DB_URI=$DB_URI"; echo "EMAIL_DOMAIN=$EMAIL_DOMAIN"; echo "MEM_STORE_URI=$MEM_STORE_URI"'
docker exec sl_app bash -lc 'cd /app; grep -n "^EMAIL_DOMAIN" tests/test.env; grep -n "^DB_URI" tests/test.env; grep -n "EMAIL_DOMAIN=sl.local" example.env'
```

**Output (complete, unedited):**

```
CONFIG=/app/tests/test.env
DB_URI=postgresql://test:test@localhost:5432/test
EMAIL_DOMAIN=sl.local
MEM_STORE_URI=redis://localhost
8:EMAIL_DOMAIN=sl.local
17:DB_URI=postgresql://test:test@localhost:15432/test
22:EMAIL_DOMAIN=sl.local
```

- `EMAIL_DOMAIN=sl.local` is declared at `tests/test.env:8` and `example.env:22`.
- **Note on `DB_URI`:** `tests/test.env:17` declares port **15432**, but the runtime recipe `/tmp/sl_env.sh` exports `DB_URI=...@localhost:5432/test`. Because `app/config.py` loads the dotenv **without override**, the already-exported `5432` value wins, so the app connects directly to the in-container PostgreSQL on `5432` (no `socat` bridge needed). This is a runtime detail of the harness, not a code change.

### 2.3 How the app was booted (mirroring `tests/conftest.py`)

Each temporary harness boots a *real* Flask app exactly as the canonical test fixture does:

1. `CONFIG` is set to `tests/test.env` (via `/tmp/sl_env.sh`) **before** importing app modules (`tests/conftest.py:8`).
2. `from server import create_app` (`tests/conftest.py:20`; `server.py` `def create_app() -> Flask`), then `app = create_app()` (`tests/conftest.py:23`).
3. The `pg_trgm` extension is (re)created (`tests/conftest.py:28-35`); dropping it while dependent objects exist prints `pg_trgm can't be dropped, ignore` and is harmless (mirrors conftest).
4. `from init_app import add_sl_domains, add_proton_partner` (`tests/conftest.py:21`); `add_sl_domains()` (`tests/conftest.py:38`) seeds the `SLDomain` rows the domain gate needs (including `sl.local`); `add_proton_partner()` (`tests/conftest.py:39`).
5. The ORM session is `from app.db import Session` (`shell.py:7`); all seeding/observation runs inside `app.app_context()`.
6. `mail_sender.store_emails_instead_of_sending(True)` (`app/mail_sender.py:102`) captures outbound mail instead of performing real SMTP on the success path.

### 2.4 Seeding the real data condition

Using the canonical helpers exactly as the tests do (`tests/test_email_handler.py`, `tests/utils.py`):

- `create_new_user()` creates a `User` whose **default mailbox email equals the user's email** (`tests/utils.py`), so `get_mailbox_from_mail_from(user.email, alias)` matches that mailbox.
- `Alias.create_new_random(user)` creates one alias per user.
- Two `Contact` rows are created on aliases owned by **different** users but with the **same** `reply_email`, via direct `Contact.create(...)`. This is accepted because there is no `UNIQUE` constraint on `reply_email` (proof in §6) and because `Contact.create` (`app/models.py:1938`) only guards that `website_email` is not itself a reverse-alias — it does **not** check `reply_email` uniqueness.

### 2.5 Canonical vs non-canonical, and scale

- **Canonical** observations drive `email_handler.handle_reply(envelope, msg, rcpt_to)` (`email_handler.py:966`) directly — the real inbound reply entry point. Routing to this handler is confirmed via `is_reverse_alias()` (`app/email_utils.py:1156-1158`), which is how `email_handler.handle()` dispatches replies.
- **`[non-canonical]`** corroboration (a direct `Contact.get_by(reply_email=...)` call) is explicitly labeled wherever used; it demonstrates the underlying `.first()` mechanism but bypasses the entry point.
- **Scale:** the cross-event experiment runs the identical input at **N=100** and is repeated across **≥2 independent runs** (§7). Where within-dataset stability is observed, the *cause* of that stability is labeled `[inferred]`; the A-vs-B ordering-dependence itself is directly **observed**.

### 2.6 Temporary harness scripts

Five throwaway scripts were used, all living **outside** the repository at `/tmp/observe_*.py` and all removed afterward (§12). Their full source is reproduced in the Evidence Appendix (§10) so the results remain reproducible after deletion. The invocation pattern was:

```
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a && . /tmp/sl_env.sh && set +a; python /tmp/observe_<name>.py [args]'
```

---

## 3. Reply-address derivation

The reply address is the inbound SMTP recipient, taken verbatim, then domain-gated and normalized. The exact code (canonical source, re-confirmed at runtime):

```
email_handler.py:966   def handle_reply(envelope, msg: Message, rcpt_to: str) -> (bool, str):
email_handler.py:972       reply_email = rcpt_to
email_handler.py:974       reply_domain = get_email_domain_part(reply_email)
email_handler.py:977       if not reply_email.endswith(EMAIL_DOMAIN):
email_handler.py:978           sl_domain: SLDomain = SLDomain.get_by(domain=reply_domain)
email_handler.py:981               return False, status.E501
email_handler.py:984       reply_email = normalize_reply_email(reply_email)
email_handler.py:986       contact = Contact.get_by(reply_email=reply_email)
```

- **Extraction:** `reply_email = rcpt_to` (`email_handler.py:972`). The reply address is *literally* the envelope recipient — no transformation at this step.
- **Domain gate:** `if not reply_email.endswith(EMAIL_DOMAIN)` (`email_handler.py:977`). If the address does not end with `EMAIL_DOMAIN` (`sl.local`), the handler falls back to an `SLDomain` lookup on the domain part (`email_handler.py:978`) and returns `status.E501` if that also fails (`email_handler.py:981`). See §8 for the observed E501.
- **Normalization:** `reply_email = normalize_reply_email(reply_email)` (`email_handler.py:984`). `normalize_reply_email()` (`app/email_validation.py:25`) first routes non-ASCII input through `convert_to_id()` (`app/email_validation.py:27-28`), then replaces any character not in `_ALLOWED_CHARS` (`app/email_validation.py:9`) with `_` (`app/email_validation.py:33`). This is a **many-to-one** mapping (see §8 collision).

### 3.1 Observed extracted/normalized value (canonical)

From the canonical single-call run (harness `observe_reply.py`, token `5jvdsvfg`):

**Command:**

```
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a && . /tmp/sl_env.sh && set +a; python /tmp/observe_reply.py 2>&1'
```

**Relevant excerpt of the (complete, unedited) output — full output in §4.2:**

```
=== reply-address derivation values ===
rcpt_to (raw)              = 'dup-reply-5jvdsvfg@sl.local'
endswith(EMAIL_DOMAIN)     = True
normalize_reply_email(...) = 'dup-reply-5jvdsvfg@sl.local'
```

- **Extracted `rcpt_to`:** `'dup-reply-5jvdsvfg@sl.local'`.
- **Domain gate passes:** `endswith(EMAIL_DOMAIN) = True`.
- **Normalized value:** `'dup-reply-5jvdsvfg@sl.local'` — unchanged here, because every character is already in `_ALLOWED_CHARS`. (The normalization *does* alter and collide other inputs; see §8.)

---

## 4. Contact resolution & `.first()` semantics

### 4.1 The lookup and its underlying `.first()`

The normalized reply address is resolved to a single `Contact`:

```
email_handler.py:986   contact = Contact.get_by(reply_email=reply_email)
email_handler.py:989       return False, status.E502            # when no contact is found
email_handler.py:990   if not contact.user.is_active():
email_handler.py:992       return False, status.E502
```

`Contact.get_by(...)` is the inherited `ModelMixin.get_by()` helper. Its **entire body** is a `.first()` with **no `ORDER BY`**:

```
app/models.py:82      @classmethod
app/models.py:83      def get_by(cls, **kw):
app/models.py:84          return Session.query(cls).filter_by(**kw).first()
```

**Consequence:** when more than one `Contact` row matches the given `reply_email`, `.first()` returns whichever row the database yields first. Because the query carries no `ORDER BY`, that row is **arbitrary and ordering/plan-dependent** — not a code-guaranteed choice. **`[inferred]`** (framework semantics): under SQLAlchemy 1.3.24 (pinned), `Query.first()` emits `LIMIT 1` with no ordering, so the returned row is not guaranteed stable across executions or across physical layouts. The direct runtime consequence (which row is returned, and that reversing insertion order flips it) is **observed** in §7.

### 4.2 Observed resolved contact (canonical, full output)

The canonical single-call run seeds two contacts sharing one `reply_email` (owned by different users), proves the duplicate persisted, then drives `handle_reply` once.

**Command:**

```
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a && . /tmp/sl_env.sh && set +a; python /tmp/observe_reply.py 2>&1'
```

**Output (complete, unedited):**

```
load config file /app/tests/test.env
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-13 17:26:52,517 - SL - DEBUG - 4553 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
>>> pg_trgm can't be dropped, ignore
2026-07-13 17:26:53,312 - SL - DEBUG - 4553 - "/app/init_app.py:42" - add_sl_domains() -  - d1.test is already a SL domain
2026-07-13 17:26:53,313 - SL - DEBUG - 4553 - "/app/init_app.py:42" - add_sl_domains() -  - d2.test is already a SL domain
2026-07-13 17:26:53,314 - SL - DEBUG - 4553 - "/app/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-13 17:26:53,315 - SL - DEBUG - 4553 - "/app/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
BOOT OK: <Flask 'server'>
DB_URI: postgresql://test:test@localhost:5432/test
2026-07-13 17:26:53,612 - SL - INFO - 4553 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-13 17:26:53,872 - SL - INFO - 4553 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-13 17:26:53,887 - SL - DEBUG - 4553 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email word_list209@sl.local
2026-07-13 17:26:53,894 - SL - INFO - 4553 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-13 17:26:53,904 - SL - DEBUG - 4553 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email list_word440@sl.local
2026-07-13 17:26:53,911 - SL - INFO - 4553 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
=== SEED ===
EMAIL_DOMAIN='sl.local'
shared_reply='dup-reply-5jvdsvfg@sl.local'
user_a.id=27 email='user_zom563s8s3@mailbox.test' alias_a.id=55 alias_a.email='word_list209@sl.local'
user_b.id=28 email='user_dokyueg5wz@mailbox.test' alias_b.id=56 alias_b.email='list_word440@sl.local'
contact_a.id=28 (alias_a, user_a)   contact_b.id=29 (alias_b, user_b)
=== PROVE DUPLICATE PERSISTED (no UNIQUE constraint blocked it) ===
rows sharing reply_email='dup-reply-5jvdsvfg@sl.local': count=2
  Contact id=28 alias_id=55 user_id=27 website_email='a@nowhere.net'
  Contact id=29 alias_id=56 user_id=28 website_email='b@nowhere.net'
=== reply-address derivation values ===
rcpt_to (raw)              = 'dup-reply-5jvdsvfg@sl.local'
endswith(EMAIL_DOMAIN)     = True
normalize_reply_email(...) = 'dup-reply-5jvdsvfg@sl.local'
=== routing predicate ===
is_reverse_alias('dup-reply-5jvdsvfg@sl.local') = True
=== [non-canonical] direct Contact.get_by(reply_email=...) ===
[non-canonical] returned Contact id=28 alias_id=55 user_id=27
=== CANONICAL CALL: email_handler.handle_reply(envelope, msg, shared_reply) ===
envelope.mail_from='user_zom563s8s3@mailbox.test' envelope.rcpt_tos=['dup-reply-5jvdsvfg@sl.local'] Message-ID='<obs-5jvdsvfg-0@sl.local>'
2026-07-13 17:26:53,931 - SL - INFO - 4553 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() -  - DMARC check disabled
2026-07-13 17:26:53,937 - SL - DEBUG - 4553 - "/app/email_handler.py:1051" - handle_reply() -  - Create <EmailLog 507> for <Contact 28 a@nowhere.net 55>, <User 27 Test User user_zom563s8s3@mailbox.test>, <Mailbox 27 user_zom563s8s3@mailbox.test>
2026-07-13 17:26:53,942 - SL - DEBUG - 4553 - "/app/email_handler.py:1171" - handle_reply() -  - From header is word_list209@sl.local
2026-07-13 17:26:53,943 - SL - DEBUG - 4553 - "/app/email_handler.py:383" - replace_header_when_reply() -  - delete the To header. Old value word_list209@sl.local
2026-07-13 17:26:53,943 - SL - DEBUG - 4553 - "/app/email_handler.py:383" - replace_header_when_reply() -  - delete the Cc header. Old value None
2026-07-13 17:26:53,945 - SL - DEBUG - 4553 - "/app/email_handler.py:1314" - replace_original_message_id() -  - create a new sl_message_id <178396361394.4553.11592256382378005378.507@sl.local>
2026-07-13 17:26:53,949 - SL - WARNING - 4553 - "/app/email_handler.py:1206" - handle_reply() -  - missing date header, add one
2026-07-13 17:26:53,951 - SL - DEBUG - 4553 - "/app/email_handler.py:1212" - handle_reply() -  - send email from word_list209@sl.local to a@nowhere.net, mail_options:[],rcpt_options:[]
2026-07-13 17:26:53,955 - SL - DEBUG - 4553 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'reply subject', from 'word_list209@sl.local' to 'None'
RESULT delivered=True code='250 Message accepted for delivery'
code==status.E200? True  code==status.E214? False  code==status.E502? False
PERSISTED EmailLog id=507 contact_id=28 alias_id=55 user_id=27 mailbox_id=27 is_reply=True message_id='<obs-5jvdsvfg-0@sl.local>'
  -> forwarding user_id=27 (user_a.id=27, user_b.id=28)
NEW SentAlert rows during call: 0
DONE
```

**Reading the output:**

- **Duplicate persisted:** `count=2` rows share `reply_email='dup-reply-5jvdsvfg@sl.local'` — `Contact id=28 (alias 55, user 27)` and `Contact id=29 (alias 56, user 28)`. No `UNIQUE` constraint blocked the second insert.
- **Resolved `Contact` (`id`, `alias_id`, `user_id`):** the canonical path resolved to **`Contact id=28`, `alias_id=55`, `user_id=27`** (the first-inserted / lowest-id row). The `[non-canonical]` direct `Contact.get_by` returned the *same* row (`id=28`), corroborating the `.first()` mechanism.
- The resolved contact's `reply_email` is the shared value `'dup-reply-5jvdsvfg@sl.local'`.

---

## 5. Forwarding-destination selection

Once the contact is resolved, the alias, user, and mailbox are derived from it:

```
email_handler.py:994    alias = contact.alias
email_handler.py:995    alias_address: str = contact.alias.email
email_handler.py:1004   user = alias.user
email_handler.py:1005   mail_from = envelope.mail_from
email_handler.py:1019   mailbox = get_mailbox_from_mail_from(mail_from, alias)
email_handler.py:1032       handle_unknown_mailbox(envelope, msg, reply_email, user, alias, contact)
email_handler.py:1034       return False, status.E214
```

- `alias = contact.alias` (`email_handler.py:994`) — the forwarding alias is the resolved contact's alias.
- `user = alias.user` (`email_handler.py:1004`) — the forwarding user is that alias's owner. The `Alias.user` relationship is `orm.relationship(User, foreign_keys=[user_id])` (`app/models.py:1576`).
- `mailbox = get_mailbox_from_mail_from(mail_from, alias)` (`email_handler.py:1019`, defined at `email_handler.py:1364`) — iterates `alias.mailboxes` and returns the mailbox whose email (or authorized address) matches `mail_from`, else `None`. When it returns `None` (unauthorized sender, spoof-check on), `handle_unknown_mailbox(...)` (`email_handler.py:1032`) is called and the handler returns `status.E214` (`email_handler.py:1034`).
- On success, the resolution is persisted:

```
email_handler.py:1042   email_log = EmailLog.create(
email_handler.py:1043       contact_id=contact.id,
email_handler.py:1044       alias_id=contact.alias_id,
email_handler.py:1045       is_reply=True,
email_handler.py:1046       user_id=contact.user_id,
email_handler.py:1047       mailbox_id=mailbox.id,
email_handler.py:1048       message_id=msg[headers.MESSAGE_ID],
email_handler.py:1049       commit=True,
email_handler.py:1050   )
```

### 5.1 Observed forwarding destination (canonical)

From the same canonical run (§4.2, token `5jvdsvfg`), the internal LOG line and the persisted `EmailLog` reveal the full destination:

- Internal creation LOG (`email_handler.py:1051`): `Create <EmailLog 507> for <Contact 28 a@nowhere.net 55>, <User 27 Test User user_zom563s8s3@mailbox.test>, <Mailbox 27 user_zom563s8s3@mailbox.test>`.
- Rewritten `From` header (`email_handler.py:1171`): `From header is word_list209@sl.local` (i.e. `alias_a.email`).
- Outbound send (`email_handler.py:1212`): `send email from word_list209@sl.local to a@nowhere.net` (the alias to the contact's real address).
- **Persisted `EmailLog`:** `id=507 contact_id=28 alias_id=55 user_id=27 mailbox_id=27 is_reply=True`.

So the forwarding destination is **alias `word_list209@sl.local` (id 55), user `27`, mailbox `27`** — all derived from resolved `Contact id=28`. `RESULT delivered=True code='250 Message accepted for delivery'` (`== status.E200`), and **0** new `SentAlert` rows (clean success path). In this dataset the first-inserted contact happens to be `user_A`'s, so the resolution is correct; §7 shows that reversing the row order makes the *identical* input resolve to a different user.


---

## 6. Uniqueness / race / timing root cause

This section states the causal chain, each link grounded in `file:line` and/or observed output. The wrong-user behavior is **not** a vague environmental effect — it is a concrete consequence of an unenforced uniqueness invariant combined with an unordered `.first()`.

### 6.1 Proof: `reply_email` has NO `UNIQUE` constraint

**Model (`app/models.py`).** The only unique constraint on `Contact` is on `(alias_id, website_email)`, and `reply_email` is only indexed:

```
app/models.py:1875           sa.UniqueConstraint("alias_id", "website_email", name="uq_contact"),
app/models.py:1899       reply_email = sa.Column(sa.String(512), nullable=False, index=True)
```

- `uq_contact` (`app/models.py:1875`) is on `(alias_id, website_email)` — **not** `reply_email`.
- `reply_email` (`app/models.py:1899`) is `index=True` but **not** `unique=True`.

**Migration (`migrations/versions/2021_071310_78403c7b8089_.py`).** The index that backs `reply_email` was created with `unique=False`:

```
migrations/versions/2021_071310_78403c7b8089_.py:22       op.create_index(op.f('ix_contact_reply_email'), 'contact', ['reply_email'], unique=False)
```

No migration anywhere adds a `UNIQUE` constraint on `reply_email`.

**Observed consequence.** The §4.2 output shows two rows with the same `reply_email` persisting successfully (`count=2`), which is only possible because no `UNIQUE` constraint exists.

### 6.2 The generation-time guard is best-effort TOCTOU (and bypassable)

Uniqueness is only enforced *at generation time*, non-atomically:

```
app/models.py:1425   def available_sl_email(email: str) -> bool:
app/models.py:1426-1432   return not (
                              Alias.get_by(email=email)
                              or Contact.get_by(reply_email=email)
                              or DeletedAlias.get_by(email=email)
                              or ...
                          )
```

- `available_sl_email()` (`app/models.py:1425-1432`) is a read-only OR-check with no locking — a classic **time-of-check-to-time-of-use (TOCTOU)** pattern. **`[inferred]`** (from the code + the confirmed absence of a `UNIQUE` constraint): under concurrency, two generator calls can both observe "available" and both insert the same `reply_email`; nothing at the DB level prevents it. A deterministic live reproduction of this concurrency window was not attempted (labeled inferred).
- It is invoked only from the generator `generate_reply_email()` (`app/email_utils.py:1103`) at the check `if available_sl_email(reply_email):` (`app/email_utils.py:1150`).
- **Non-generator write paths bypass it entirely.** Direct `Contact.create(reply_email=...)` never calls `available_sl_email()`. **Observed** (§8): `available_sl_email('ra_x...@sl.local') = False` for an address already in use, yet a direct `Contact.create` with that same address still succeeded — the guard is simply not on that path.

### 6.3 The lookup returns an arbitrary row

`Contact.get_by(reply_email=...)` (`email_handler.py:986`) → `ModelMixin.get_by()` → `Session.query(cls).filter_by(**kw).first()` with **no `ORDER BY`** (`app/models.py:83-84`). On a multi-row match the returned row is arbitrary/ordering-dependent — **observed** in §7 (reversing insertion order flips the result 100/100).

### 6.4 A wrong contact yields a wrong alias, user, and persisted log

- `alias = contact.alias` (`email_handler.py:994`) and `user = alias.user` (`email_handler.py:1004`) are derived *directly* from the resolved contact.
- The resolved user is persisted: `EmailLog.create(..., user_id=contact.user_id, ...)` (`email_handler.py:1042-1050`). So a reply is not only *routed* under the wrong user, it is *recorded* against that user. **Observed** in §7 (BA run: `EmailLog.user_id = user_B` for all 100 events).

### 6.5 The same non-unique lookup gates multiple decisions

`Contact.get_by(reply_email=...)` is reused at several decision points, all inheriting the same arbitrary-on-multi-match behavior:

- `replace_header_when_reply()` at `email_handler.py:364` (restores original contact addresses in CC/To) — **observed** executing in §8.
- Routing/bounce lookups: `email_handler.py:1998` (`reply_email=mail_from`), `email_handler.py:2009` (`reply_email=from_header_address`), `email_handler.py:2167` (`reply_email=rcpt_tos[0]`). **`[inferred]`** (grounded by `file:line`): these call the identical `ModelMixin.get_by().first()` and therefore exhibit the same arbitrary selection on a multi-row match; they were not each driven to a multi-row condition at runtime.
- `is_reverse_alias()` at `app/email_utils.py:1158` (`if Contact.get_by(reply_email=address): return True`) — the routing predicate that dispatches to `handle_reply` — **observed** returning `True` in §4.2 and §8.

### 6.6 Normalization widens the collision surface

`normalize_reply_email()` (`app/email_validation.py:25-38`) is many-to-one: distinct stored `reply_email` values can collapse to the same lookup key, increasing the chance of a multi-row match at lookup time. **Observed** in §8 (three distinct inputs normalize to one key; a canonical reply whose `rcpt_to` normalizes onto a stored value resolves to it).

### 6.7 Forward-phase minting (where `reply_email` originates)

A contact's `reply_email` is originally minted in the forward phase:

```
email_handler.py:299    reply_email=generate_reply_email(contact_email, alias),   # inside Contact.create(...)
```

`generate_reply_email()` (`app/email_utils.py:1103`) loops (up to 1000 attempts) calling the TOCTOU `available_sl_email()` (`app/email_utils.py:1150`). The seeded scenario uses direct `Contact.create(reply_email=...)`, which — like any non-generator path — does **not** pass through this guard, which is precisely why duplicates are possible.

---

## 7. Cross-event behavior (distribution across ≥2 runs)

**Question:** does the *same* reply email ever resolve to *different* contacts/users? **Answer (observed):** within a fixed dataset the resolution is stable at the first-inserted / lowest-`ctid` row (100/100); but the resolved **user is determined purely by physical row order** — reversing the insertion order of the two duplicate rows flips the resolved user from A to B (100/100) under the *identical* canonical input.

The harness `observe_dist.py` seeds two users/aliases, sets `disable_email_spoofing_check = True` on both aliases **only so that every iteration reaches `EmailLog.create` and records the chosen `user_id`** (this does **not** influence which row `.first()` selects — it merely lets the choice be *recorded* each time), inserts two `Contact` rows sharing `shared_reply` in a given order, then issues **N=100 identical** `handle_reply` calls (same `mail_from = user_A`'s mailbox, same `rcpt_to = shared_reply`; only the transient `Message-ID` varies). It reads the resolved identity from the `EmailLog` persisted by each call and reports a `Counter` distribution. It also prints `EXPLAIN ANALYZE` of the exact equality lookup (`LIMIT 1`, no `ORDER BY`).

### 7.1 Run 1 — insertion order A-then-B (N=100)

**Command:**

```
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a && . /tmp/sl_env.sh && set +a; python /tmp/observe_dist.py ab 100 2>/dev/null'
```

**Output (complete, unedited):**

```
load config file /app/tests/test.env
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-13 17:27:06,696 - SL - DEBUG - 4574 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-13 17:27:07,479 - SL - DEBUG - 4574 - "/app/init_app.py:42" - add_sl_domains() -  - d1.test is already a SL domain
2026-07-13 17:27:07,480 - SL - DEBUG - 4574 - "/app/init_app.py:42" - add_sl_domains() -  - d2.test is already a SL domain
2026-07-13 17:27:07,480 - SL - DEBUG - 4574 - "/app/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-13 17:27:07,481 - SL - DEBUG - 4574 - "/app/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-13 17:27:07,777 - SL - INFO - 4574 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-13 17:27:08,036 - SL - INFO - 4574 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-13 17:27:08,050 - SL - DEBUG - 4574 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email test_list367@sl.local
2026-07-13 17:27:08,057 - SL - INFO - 4574 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-13 17:27:08,066 - SL - DEBUG - 4574 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email word_word010@sl.local
2026-07-13 17:27:08,073 - SL - INFO - 4574 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
=== DIST EXPERIMENT order=ab N=100 tok=ceyu31ut ===
shared_reply='dup-reply-ceyu31ut@sl.local'
mail_from (IDENTICAL every call) = user_a.email = 'user_k1g14fv1aj@mailbox.test'
user_a.id=29 alias_a.id=59 | user_b.id=30 alias_b.id=60
INSERTION ORDER: 1st=Contact id=30 user_id=29 ; 2nd=Contact id=31 user_id=30
--- EXPLAIN ANALYZE of the equality lookup that .first() runs (LIMIT 1, NO ORDER BY) ---
    Limit  (cost=0.14..8.16 rows=1 width=1954) (actual time=0.006..0.007 rows=1 loops=1)
      ->  Index Scan using ix_contact_reply_email on contact  (cost=0.14..8.16 rows=1 width=1954) (actual time=0.006..0.006 rows=1 loops=1)
            Index Cond: ((reply_email)::text = 'dup-reply-ceyu31ut@sl.local'::text)
    Planning Time: 0.030 ms
    Execution Time: 0.015 ms
--- DISTRIBUTION over N=100 IDENTICAL calls (mail_from='user_k1g14fv1aj@mailbox.test', rcpt_to='dup-reply-ceyu31ut@sl.local') ---
resolved Contact.id : {(30, 'FIRST_INSERTED'): 100}
resolved user_id    : {(29, 'user_A(alias-owner-matching-mail_from)'): 100}
DONE
```

**Result:** all **100/100** resolved to `Contact id=30` (FIRST_INSERTED), `user_id=29` (user_A). The lookup plan is `Index Scan using ix_contact_reply_email` under `Limit ... rows=1` — **no `Sort` node**.

### 7.2 Run 2 — insertion order A-then-B, independent re-seed (N=100)

**Command:**

```
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a && . /tmp/sl_env.sh && set +a; python /tmp/observe_dist.py ab 100 2>/dev/null'
```

**Output (complete, unedited):**

```
load config file /app/tests/test.env
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-13 17:27:11,548 - SL - DEBUG - 4594 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-13 17:27:12,342 - SL - DEBUG - 4594 - "/app/init_app.py:42" - add_sl_domains() -  - d1.test is already a SL domain
2026-07-13 17:27:12,343 - SL - DEBUG - 4594 - "/app/init_app.py:42" - add_sl_domains() -  - d2.test is already a SL domain
2026-07-13 17:27:12,344 - SL - DEBUG - 4594 - "/app/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-13 17:27:12,345 - SL - DEBUG - 4594 - "/app/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-13 17:27:12,641 - SL - INFO - 4594 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-13 17:27:12,901 - SL - INFO - 4594 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-13 17:27:12,915 - SL - DEBUG - 4594 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email test_list950@sl.local
2026-07-13 17:27:12,922 - SL - INFO - 4594 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-13 17:27:12,931 - SL - DEBUG - 4594 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email test_list373@sl.local
2026-07-13 17:27:12,939 - SL - INFO - 4594 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
=== DIST EXPERIMENT order=ab N=100 tok=8k5kuet1 ===
shared_reply='dup-reply-8k5kuet1@sl.local'
mail_from (IDENTICAL every call) = user_a.email = 'user_0ec7i060fk@mailbox.test'
user_a.id=31 alias_a.id=63 | user_b.id=32 alias_b.id=64
INSERTION ORDER: 1st=Contact id=32 user_id=31 ; 2nd=Contact id=33 user_id=32
--- EXPLAIN ANALYZE of the equality lookup that .first() runs (LIMIT 1, NO ORDER BY) ---
    Limit  (cost=0.14..8.16 rows=1 width=1954) (actual time=0.008..0.009 rows=1 loops=1)
      ->  Index Scan using ix_contact_reply_email on contact  (cost=0.14..8.16 rows=1 width=1954) (actual time=0.008..0.008 rows=1 loops=1)
            Index Cond: ((reply_email)::text = 'dup-reply-8k5kuet1@sl.local'::text)
    Planning Time: 0.047 ms
    Execution Time: 0.021 ms
--- DISTRIBUTION over N=100 IDENTICAL calls (mail_from='user_0ec7i060fk@mailbox.test', rcpt_to='dup-reply-8k5kuet1@sl.local') ---
resolved Contact.id : {(32, 'FIRST_INSERTED'): 100}
resolved user_id    : {(31, 'user_A(alias-owner-matching-mail_from)'): 100}
DONE
```

**Result:** again **100/100** → `Contact id=32` (FIRST_INSERTED), `user_id=31` (user_A). **Stable across the two independent A-then-B runs.**

### 7.3 Run 3 — REVERSED insertion order B-then-A, identical input (N=100) — the wrong-user proof

**Command:**

```
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a && . /tmp/sl_env.sh && set +a; python /tmp/observe_dist.py ba 100 2>/dev/null'
```

**Output (complete, unedited):**

```
load config file /app/tests/test.env
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-13 17:27:24,239 - SL - DEBUG - 4616 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-13 17:27:25,019 - SL - DEBUG - 4616 - "/app/init_app.py:42" - add_sl_domains() -  - d1.test is already a SL domain
2026-07-13 17:27:25,020 - SL - DEBUG - 4616 - "/app/init_app.py:42" - add_sl_domains() -  - d2.test is already a SL domain
2026-07-13 17:27:25,021 - SL - DEBUG - 4616 - "/app/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-13 17:27:25,022 - SL - DEBUG - 4616 - "/app/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-13 17:27:25,317 - SL - INFO - 4616 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-13 17:27:25,578 - SL - INFO - 4616 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-13 17:27:25,592 - SL - DEBUG - 4616 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email word_word807@sl.local
2026-07-13 17:27:25,599 - SL - INFO - 4616 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-13 17:27:25,608 - SL - DEBUG - 4616 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email test_word965@sl.local
2026-07-13 17:27:25,615 - SL - INFO - 4616 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
=== DIST EXPERIMENT order=ba N=100 tok=7bxcb1wz ===
shared_reply='dup-reply-7bxcb1wz@sl.local'
mail_from (IDENTICAL every call) = user_a.email = 'user_hpc4r2ln0n@mailbox.test'
user_a.id=33 alias_a.id=67 | user_b.id=34 alias_b.id=68
INSERTION ORDER: 1st=Contact id=34 user_id=34 ; 2nd=Contact id=35 user_id=33
--- EXPLAIN ANALYZE of the equality lookup that .first() runs (LIMIT 1, NO ORDER BY) ---
    Limit  (cost=0.14..8.16 rows=1 width=1954) (actual time=0.006..0.006 rows=1 loops=1)
      ->  Index Scan using ix_contact_reply_email on contact  (cost=0.14..8.16 rows=1 width=1954) (actual time=0.005..0.005 rows=1 loops=1)
            Index Cond: ((reply_email)::text = 'dup-reply-7bxcb1wz@sl.local'::text)
    Planning Time: 0.029 ms
    Execution Time: 0.014 ms
--- DISTRIBUTION over N=100 IDENTICAL calls (mail_from='user_hpc4r2ln0n@mailbox.test', rcpt_to='dup-reply-7bxcb1wz@sl.local') ---
resolved Contact.id : {(34, 'FIRST_INSERTED'): 100}
resolved user_id    : {(34, 'user_B(different-user)'): 100}
DONE
```

**Result — WRONG USER:** with the **identical** input (`mail_from = user_A`'s mailbox `user_hpc4r2ln0n@mailbox.test`, `rcpt_to = dup-reply-7bxcb1wz@sl.local`), all **100/100** resolved to `Contact id=34`, whose `user_id=34` is **`user_B` — the different user**, not the sender/alias-owner `user_A` (id 33). The only thing changed versus §7.1–7.2 is the *pre-existing physical order* of the two duplicate rows. `EmailLog.user_id` recorded `user_B` for all 100 events.

### 7.4 psql corroboration: physical order (`ctid`) and plan-dependence

To show *why* `.first()` returned that row, the persisted BA dataset was inspected directly.

**Command:**

```
docker exec sl_app bash -lc 'su postgres -c "psql -d test <<SQL
\pset pager off
SELECT id, alias_id, user_id, website_email, ctid FROM contact WHERE reply_email = ''dup-reply-7bxcb1wz@sl.local'' ORDER BY ctid;
EXPLAIN (ANALYZE, COSTS OFF) SELECT * FROM contact WHERE reply_email = ''dup-reply-7bxcb1wz@sl.local'' LIMIT 1;
SELECT id, user_id FROM contact WHERE reply_email = ''dup-reply-7bxcb1wz@sl.local'' LIMIT 1;
BEGIN;
SET LOCAL enable_indexscan = off;
SET LOCAL enable_bitmapscan = off;
EXPLAIN (ANALYZE, COSTS OFF) SELECT * FROM contact WHERE reply_email = ''dup-reply-7bxcb1wz@sl.local'' LIMIT 1;
COMMIT;
SQL"'
```

**Output (complete, unedited):**

```
Pager usage is off.
 id | alias_id | user_id | website_email |  ctid  
----+----------+---------+---------------+--------
 34 |       68 |      34 | b@nowhere.net | (0,34)
 35 |       67 |      33 | a@nowhere.net | (0,35)
(2 rows)

                                             QUERY PLAN                                             
----------------------------------------------------------------------------------------------------
 Limit (actual time=0.007..0.007 rows=1 loops=1)
   ->  Index Scan using ix_contact_reply_email on contact (actual time=0.006..0.006 rows=1 loops=1)
         Index Cond: ((reply_email)::text = 'dup-reply-7bxcb1wz@sl.local'::text)
 Planning Time: 0.044 ms
 Execution Time: 0.016 ms
(5 rows)

 id | user_id 
----+---------
 34 |      34
(1 row)

BEGIN
SET
SET
                                 QUERY PLAN                                  
-----------------------------------------------------------------------------
 Limit (actual time=0.012..0.012 rows=1 loops=1)
   ->  Seq Scan on contact (actual time=0.011..0.011 rows=1 loops=1)
         Filter: ((reply_email)::text = 'dup-reply-7bxcb1wz@sl.local'::text)
         Rows Removed by Filter: 33
 Planning Time: 0.026 ms
 Execution Time: 0.019 ms
(6 rows)

COMMIT
```

**Interpretation:**

- `ctid (0,34)` for `id=34` and `(0,35)` for `id=35` show that **physical row order equals insertion order** (in the BA run, `user_B`'s contact `id=34` was inserted first).
- The **default** plan is `Index Scan using ix_contact_reply_email` under `Limit rows=1`, and it returns `id=34, user_id=34` — matching the 100/100 distribution.
- **Plan-dependence:** forcing `enable_indexscan=off; enable_bitmapscan=off` makes the planner switch to `Seq Scan on contact` for the *same* query. This demonstrates the row is served by whatever plan/physical order the engine chooses — **nothing pins the order because there is no `ORDER BY`**.

### 7.5 Verdict

- **Within a fixed dataset:** resolution is **stable** (100/100) at the first-inserted / lowest-`ctid` row; confirmed across two independent A-then-B runs (§7.1, §7.2).
- **Across datasets:** the resolved **user is governed purely by physical row order**, not by the alias owner or `mail_from`. Reversing only the insertion order of the two duplicate rows flips the resolved user **A → B (100/100)** under the identical canonical input (§7.3). This is the directly-reproducible, ordering-dependent wrong-user behavior.
- **`[inferred]`** nuance: the within-dataset stability is an implementation artifact of PostgreSQL serving the equal-key lookup via the index in lowest-`ctid`-first order on an append-only static heap; because `ModelMixin.get_by` emits `LIMIT 1` with **no `ORDER BY`** (`app/models.py:83-84`), a different physical layout (`UPDATE`/`HOT`/`VACUUM`/bloat), a different plan (the forced `Seq Scan` above shows the planner *will* switch), or a different PG/SQLAlchemy build could return the other row. The "could differ under other physical/plan conditions" statement is `[inferred]` from PostgreSQL/SQLAlchemy 1.3.24 semantics plus the observed absence of a `Sort` node; the A-vs-B ordering-dependence is **observed**.


---

## 8. Edge / secondary conditions

Every implied branch of the reply path was exercised. The consolidated harness `observe_edge.py` (source in §10) covers E501, E502 (no-contact and inactive-user), E214 (wrong-user alert), the `normalize_reply_email()` collision, the `available_sl_email()` guard, and `is_reverse_alias()`. Two focused harnesses (`observe_norm.py`, `observe_second.py`) capture the normalization-collision `EmailLog` persistence and the second lookup site.

### 8.1 Consolidated edge harness

**Command:**

```
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a && . /tmp/sl_env.sh && set +a; python /tmp/observe_edge.py 2>/dev/null'
```

**Output (complete, unedited):**

```
load config file /app/tests/test.env
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-13 17:28:00,286 - SL - DEBUG - 4649 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-13 17:28:01,076 - SL - DEBUG - 4649 - "/app/init_app.py:42" - add_sl_domains() -  - d1.test is already a SL domain
2026-07-13 17:28:01,077 - SL - DEBUG - 4649 - "/app/init_app.py:42" - add_sl_domains() -  - d2.test is already a SL domain
2026-07-13 17:28:01,078 - SL - DEBUG - 4649 - "/app/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-13 17:28:01,078 - SL - DEBUG - 4649 - "/app/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
========== E501: reply domain not EMAIL_DOMAIN and not an SLDomain ==========
rcpt_to='whatever@not-sl-domain.example' -> delivered=False code='550 SL E501'  (status.E501='550 SL E501') match=True
========== E502(a): rcpt_to ends with EMAIL_DOMAIN but NO Contact ==========
rcpt_to='no-contact-dclmiu@sl.local' Contact.get_by->None -> delivered=False code='550 SL E502 Email not exist' (E502='550 SL E502 Email not exist') match=True
========== E502(b): contact exists but contact.user is INACTIVE ==========
user.id=35 is_active()=False (delete_on=2026-07-13T18:28:01.407555+00:00)
rcpt_to='inactive-dclmiu@sl.local' -> delivered=False code='550 SL E502 Email not exist' (E502='550 SL E502 Email not exist') match=True
========== E214 + WRONG-USER alert: duplicate reply_email, resolves to user_B, mail_from is user_A ==========
mail_from(user_A)='user_px2u3iyq3x@mailbox.test'  user_A.id=36 user_B.id=37
.first() resolved Contact id=37 alias_id=74 user_id=37 (=user_B)
-> delivered=False code='250 SL E214 Unauthorized for using reverse alias' (E214='250 SL E214 Unauthorized for using reverse alias') match=True
   SentAlert id=4 user_id=37 to_email='user_vv5oe7dlsd@mailbox.test' alert_type='reverse_alias_unknown_mailbox'
   ^ alert was sent to user_B though the reply came from user_A's mailbox => WRONG USER
========== normalize_reply_email(): many-to-one collision ==========
normalize_reply_email('ra#xdclmiu@sl.local')   = 'ra_xdclmiu@sl.local'
normalize_reply_email('ra%xdclmiu@sl.local')   = 'ra_xdclmiu@sl.local'
normalize_reply_email('ra_xdclmiu@sl.local') = 'ra_xdclmiu@sl.local'
all three collide to same key? True
canonical handle_reply(rcpt_to='ra#xdclmiu@sl.local') -> delivered=True code='250 Message accepted for delivery'; resolved EmailLog.contact_id=39 (stored contact id=39) match=True
========== available_sl_email() best-effort guard (bypassed by direct Contact.create) ==========
available_sl_email('ra_xdclmiu@sl.local') now that a Contact uses it = False  (False => generator would avoid it)
available_sl_email('brand-new-dclmiu@sl.local') = True  (True => free)
NOTE: direct Contact.create(reply_email=...) NEVER calls available_sl_email(); only generate_reply_email() does.
========== is_reverse_alias() routing predicate on the shared key ==========
is_reverse_alias('e214-dclmiu@sl.local') = True
DONE
```

**Reading each condition (token `dclmiu`):**

- **E501** (`email_handler.py:977-981`): `rcpt_to='whatever@not-sl-domain.example'` does not end with `EMAIL_DOMAIN` and is not an `SLDomain` → `delivered=False code='550 SL E501'` (`match=True`).
- **E502(a) — no contact** (`email_handler.py:986-989`): `rcpt_to='no-contact-dclmiu@sl.local'` ends with `EMAIL_DOMAIN` but `Contact.get_by->None` → `delivered=False code='550 SL E502 Email not exist'` (`match=True`).
- **E502(b) — inactive user** (`email_handler.py:990-992`; `User.is_active` at `app/models.py:766`): the contact exists but its user's `delete_on` is set to a future time, so `is_active()=False` → `code='550 SL E502 Email not exist'` (`match=True`).
- **E214 — wrong-user alert** (`email_handler.py:1019-1034`): two contacts share `reply_email='e214-dclmiu@sl.local'` with `user_B`'s inserted first, so `.first()` resolved `Contact id=37, alias_id=74, user_id=37` (`user_B`). The sender `mail_from = user_A`'s mailbox is **not** authorized for `user_B`'s alias, so with the spoof-check ON, `get_mailbox_from_mail_from` returned `None`, `handle_unknown_mailbox` fired, and the handler returned `code='250 SL E214'` (`match=True`). Critically, `SentAlert id=4 user_id=37 to_email='user_vv5oe7dlsd@mailbox.test' alert_type='reverse_alias_unknown_mailbox'` — **the alert was sent to `user_B` even though the reply came from `user_A`'s mailbox**. This is a second, independent manifestation of the wrong-user condition (information about `user_A`'s reply attempt is surfaced to `user_B`).
- **`normalize_reply_email()` many-to-one** (`app/email_validation.py:9,25-38`): `'ra#xdclmiu@sl.local'`, `'ra%xdclmiu@sl.local'`, and `'ra_xdclmiu@sl.local'` all normalize to `'ra_xdclmiu@sl.local'` (`#` and `%` are not in `_ALLOWED_CHARS`, so they become `_`). `all three collide to same key? True`. Driving the canonical `handle_reply(rcpt_to='ra#xdclmiu@sl.local')` delivered successfully and resolved `EmailLog.contact_id=39`, exactly the stored contact whose `reply_email` is `'ra_xdclmiu@sl.local'` (`match=True`) — i.e. an inbound address spelled with a disallowed character resolves onto a *different* stored value after normalization.
- **`available_sl_email()`** (`app/models.py:1425-1432`): returns `False` for the in-use address and `True` for a brand-new one. Confirms the guard *would* steer the generator away from duplicates — but direct `Contact.create(reply_email=...)` never calls it.
- **`is_reverse_alias()`** (`app/email_utils.py:1156-1158`): returns `True` on the shared key — the routing predicate that dispatches to `handle_reply`.

### 8.2 Focused normalization-collision persistence (`observe_norm.py`)

**Command:**

```
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a && . /tmp/sl_env.sh && set +a; python /tmp/observe_norm.py 2>/dev/null'
```

**Output (complete, unedited):**

```
load config file /app/tests/test.env
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-13 17:28:14,011 - SL - DEBUG - 4669 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-13 17:28:14,790 - SL - DEBUG - 4669 - "/app/init_app.py:42" - add_sl_domains() -  - d1.test is already a SL domain
2026-07-13 17:28:14,791 - SL - DEBUG - 4669 - "/app/init_app.py:42" - add_sl_domains() -  - d2.test is already a SL domain
2026-07-13 17:28:14,792 - SL - DEBUG - 4669 - "/app/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-13 17:28:14,793 - SL - DEBUG - 4669 - "/app/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
stored contact id 40 reply_email ra_xgw1mlc@sl.local
normalize(in1)= ra_xgw1mlc@sl.local  Contact.get_by(normalized)= <Contact 40 cc-gw1mlc@nowhere.net 78>
before max EmailLog.id = 808
handle_reply(rcpt_to=in1) -> True '250 Message accepted for delivery'
after max EmailLog.id = 809
  NEW EmailLog id=809 contact_id=40 alias_id=78 user_id=39 message_id='<norm-gw1mlc@sl.local>'
DONE
```

The stored contact (id 40) has `reply_email='ra_xgw1mlc@sl.local'`. An inbound `rcpt_to='ra#xgw1mlc@sl.local'` normalizes to `'ra_xgw1mlc@sl.local'`, and the canonical `handle_reply` persisted `EmailLog id=809 contact_id=40` — the collision resolved onto the stored contact through the real entry point.

### 8.3 Second lookup site — `replace_header_when_reply()` (`observe_second.py`)

**Command:**

```
docker exec sl_app bash -lc 'cd /app; . venv/bin/activate; set -a && . /tmp/sl_env.sh && set +a; python /tmp/observe_second.py 2>&1'
```

**Output (complete, unedited):**

```
load config file /app/tests/test.env
>>> URL: http://localhost
Upload files to local dir
>>> init logging <<<
2026-07-13 17:28:16,200 - SL - DEBUG - 4690 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-13 17:28:16,990 - SL - DEBUG - 4690 - "/app/init_app.py:42" - add_sl_domains() -  - d1.test is already a SL domain
2026-07-13 17:28:16,991 - SL - DEBUG - 4690 - "/app/init_app.py:42" - add_sl_domains() -  - d2.test is already a SL domain
2026-07-13 17:28:16,992 - SL - DEBUG - 4690 - "/app/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-13 17:28:16,993 - SL - DEBUG - 4690 - "/app/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-13 17:28:17,290 - SL - INFO - 4690 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
2026-07-13 17:28:17,306 - SL - DEBUG - 4690 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email test_test155@sl.local
2026-07-13 17:28:17,313 - SL - INFO - 4690 - "/app/app/events/event_dispatcher.py:58" - send_event() -  - Not sending events because webhook is disabled
alias='test_test155@sl.local' c1.id=41(reply=rt-jicgid@sl.local) c2.id=42(reply=other-jicgid@sl.local)
driving handle_reply: rcpt_to='rt-jicgid@sl.local'; To header contains reverse-alias r2='other-jicgid@sl.local'
2026-07-13 17:28:17,329 - SL - INFO - 4690 - "/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() -  - DMARC check disabled
2026-07-13 17:28:17,335 - SL - DEBUG - 4690 - "/app/email_handler.py:1051" - handle_reply() -  - Create <EmailLog 810> for <Contact 41 target-jicgid@nowhere.net 80>, <User 40 Test User user_j53j94nmd9@mailbox.test>, <Mailbox 40 user_j53j94nmd9@mailbox.test>
2026-07-13 17:28:17,340 - SL - DEBUG - 4690 - "/app/email_handler.py:1171" - handle_reply() -  - From header is test_test155@sl.local
2026-07-13 17:28:17,342 - SL - DEBUG - 4690 - "/app/email_handler.py:380" - replace_header_when_reply() -  - Replace To header, old: other-jicgid@sl.local, new: Cced <cced-jicgid@nowhere.net>
2026-07-13 17:28:17,342 - SL - DEBUG - 4690 - "/app/email_handler.py:383" - replace_header_when_reply() -  - delete the Cc header. Old value None
2026-07-13 17:28:17,344 - SL - DEBUG - 4690 - "/app/email_handler.py:1314" - replace_original_message_id() -  - create a new sl_message_id <178396369734.4690.6661809572661413553.810@sl.local>
2026-07-13 17:28:17,348 - SL - WARNING - 4690 - "/app/email_handler.py:1206" - handle_reply() -  - missing date header, add one
2026-07-13 17:28:17,350 - SL - DEBUG - 4690 - "/app/email_handler.py:1212" - handle_reply() -  - send email from test_test155@sl.local to target-jicgid@nowhere.net, mail_options:[],rcpt_options:[]
2026-07-13 17:28:17,354 - SL - DEBUG - 4690 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 's', from 'test_test155@sl.local' to 'Cced <cced-jicgid@nowhere.net>'
result delivered=True code='250 Message accepted for delivery'
```

The reply's `rcpt_to='rt-jicgid@sl.local'` resolves the primary contact (`EmailLog 810` for `Contact 41`), while the `To` header carries a *different* reverse-alias `other-jicgid@sl.local`. The LOG at `email_handler.py:380` — `Replace To header, old: other-jicgid@sl.local, new: Cced <cced-jicgid@nowhere.net>` — proves `replace_header_when_reply()`'s `Contact.get_by(reply_email=...)` at `email_handler.py:364` executed during the canonical reply and resolved that second reverse-alias to its contact via the same non-unique lookup.

---

## 9. SimpleLogin reverse-alias concept (intended invariant vs. schema reality)

Background from the official SimpleLogin documentation (paraphrased; short quotations only):

- A reverse-alias is generated **per `(alias, contact)` pair** — the docs state a reverse-alias is created "for each alias you want to send email from and each contact you want to send email to" (simplelogin.io).
- It is "dynamically generated for each sender" and, on reply, SimpleLogin relays the message back so it appears to come from the alias while "your real mailbox address stays hidden" (simplelogin.io reverse-alias docs).
- The forwarding service "redirects your response back to the original sender and replace your real email address with the email alias" (simplelogin.io/email-forwarding).

**Intended invariant:** one `reply_email` (reverse-alias) should map to exactly one `(alias, contact)` pair, hence exactly one owning user.

**Schema reality (the crux):** the database does **not** enforce this. There is no `UNIQUE` constraint on `contact.reply_email` (§6.1), so two contacts on aliases owned by different users can share one `reply_email`, and the unordered `.first()` lookup (§6.3) then resolves an arbitrary one. The web-confirmed design intent is precisely the invariant the schema fails to guarantee — which is the root of the wrong-user question. (Web facts are background only; the primary evidence is the runtime observations and `file:line` references above.)


---

## 10. Per-claim Evidence Appendix

### 10.1 Factual claims → `file:line` (re-confirmed at runtime, canonical source in `/app`)

| Claim | Reference |
|-------|-----------|
| Reply entry point | `email_handler.py:966` `def handle_reply(envelope, msg: Message, rcpt_to: str)` |
| Reply address = recipient | `email_handler.py:972` `reply_email = rcpt_to` |
| Domain part | `email_handler.py:974` `reply_domain = get_email_domain_part(reply_email)` |
| Domain gate | `email_handler.py:977` `if not reply_email.endswith(EMAIL_DOMAIN):` |
| SLDomain fallback | `email_handler.py:978` `sl_domain: SLDomain = SLDomain.get_by(domain=reply_domain)` |
| E501 return | `email_handler.py:981` `return False, status.E501` |
| Normalization | `email_handler.py:984` `reply_email = normalize_reply_email(reply_email)` |
| Contact resolution | `email_handler.py:986` `contact = Contact.get_by(reply_email=reply_email)` |
| E502 (no contact) | `email_handler.py:989` `return False, status.E502` |
| Inactive-user gate | `email_handler.py:990` `if not contact.user.is_active():` → `:992` E502 |
| Alias from contact | `email_handler.py:994` `alias = contact.alias` |
| Alias address | `email_handler.py:995` `alias_address: str = contact.alias.email` |
| User from alias | `email_handler.py:1004` `user = alias.user` |
| mail_from | `email_handler.py:1005` `mail_from = envelope.mail_from` |
| Mailbox selection | `email_handler.py:1019` `mailbox = get_mailbox_from_mail_from(mail_from, alias)` (def `:1364`) |
| Unknown-mailbox alert | `email_handler.py:1032` `handle_unknown_mailbox(envelope, msg, reply_email, user, alias, contact)` |
| E214 return | `email_handler.py:1034` `return False, status.E214` |
| Persisted resolution | `email_handler.py:1042-1050` `EmailLog.create(contact_id=..., alias_id=..., is_reply=True, user_id=contact.user_id, mailbox_id=..., ...)` |
| `.first()` no ORDER BY | `app/models.py:83-84` `return Session.query(cls).filter_by(**kw).first()` |
| Only unique constraint | `app/models.py:1875` `sa.UniqueConstraint("alias_id", "website_email", name="uq_contact")` |
| reply_email indexed, not unique | `app/models.py:1899` `reply_email = sa.Column(sa.String(512), nullable=False, index=True)` |
| Non-unique index (migration) | `migrations/versions/2021_071310_78403c7b8089_.py:22` `create_index(..., ['reply_email'], unique=False)` |
| TOCTOU guard | `app/models.py:1425-1432` `def available_sl_email(email)` OR-read |
| Alias.user relationship | `app/models.py:1576` `orm.relationship(User, foreign_keys=[user_id])` |
| Alias.mailbox relationship | `app/models.py:1577` `orm.relationship("Mailbox", lazy="joined")` |
| User.is_active | `app/models.py:766` |
| Generator loop | `app/email_utils.py:1103` `def generate_reply_email(...)`; guard `:1150` `if available_sl_email(reply_email):` |
| Routing predicate | `app/email_utils.py:1156-1158` `def is_reverse_alias(...)`; `if Contact.get_by(reply_email=address): return True` |
| Normalization chars/loop | `app/email_validation.py:9` `_ALLOWED_CHARS`; `:25` `def normalize_reply_email`; `:27-28` non-ascii→convert_to_id; `:33` disallowed→`_` |
| Second lookup site | `email_handler.py:345` `def replace_header_when_reply(...)`; `:364` `Contact.get_by(reply_email=reply_email)`; observed LOG `:380` |
| Extra reuse sites | `email_handler.py:1998`, `:2009`, `:2167` `Contact.get_by(reply_email=...)` |
| Forward-phase minting | `email_handler.py:299` `reply_email=generate_reply_email(contact_email, alias)` |
| EMAIL_DOMAIN config | `tests/test.env:8`, `example.env:22` |
| DB_URI config | `tests/test.env:17` |
| Bootstrap | `tests/conftest.py:8,20,21,23,28-35,38,39`; `server.py` `create_app`; `shell.py:7` `from app.db import Session` |

### 10.2 Behavioral claims → command + output (section cross-reference)

| Behavioral claim | Where the command + complete unedited output appears |
|------------------|-------------------------------------------------------|
| Canonical runtime (Py 3.10 / PG 15 / Redis) | §2.1 |
| Canonical config values | §2.2 |
| Duplicate `reply_email` persists (no UNIQUE) | §4.2 (`count=2`) |
| Extracted/normalized reply value | §3.1, §4.2 |
| `is_reverse_alias` → True (routing) | §4.2, §8.1 |
| Canonical resolution → contact/alias/user/mailbox + EmailLog | §4.2, §5.1 |
| `[non-canonical]` direct `Contact.get_by` corroboration | §4.2 |
| Stable 100/100 within dataset (A-then-B), ≥2 runs | §7.1, §7.2 |
| Wrong-user 100/100 on reversed order (B-then-A) | §7.3 |
| `ctid`/physical order + Index Scan + forced Seq Scan | §7.4 |
| E501 | §8.1 |
| E502 (no contact) | §8.1 |
| E502 (inactive user) | §8.1 |
| E214 + wrong-user alert (`SentAlert` to user_B) | §8.1 |
| normalize many-to-one collision + canonical resolve | §8.1, §8.2 |
| `available_sl_email` guard (bypassed by direct create) | §8.1 |
| Second lookup site executes (LOG `:380`) | §8.3 |

### 10.3 Temporary harness source (reproduced so results survive script deletion)

All scripts lived at `/tmp/observe_*.py` (outside the repository) and were deleted afterward (§12).

**`/tmp/observe_reply.py`** (canonical single call — §3.1, §4.2, §5.1):

```python
import os, sys, random, string
from app.db import Session, engine
from server import create_app
from init_app import add_sl_domains, add_proton_partner
app = create_app()
app.config["TESTING"] = True
app.config["WTF_CSRF_ENABLED"] = False
app.config["SERVER_NAME"] = "sl.test"
import sqlalchemy
from psycopg2 import errors
from psycopg2.errorcodes import DEPENDENT_OBJECTS_STILL_EXIST
with engine.connect() as conn:
    try:
        conn.execute("DROP EXTENSION if exists pg_trgm")
        conn.execute("CREATE EXTENSION pg_trgm")
    except sqlalchemy.exc.InternalError as e:
        if isinstance(e.orig, errors.lookup(DEPENDENT_OBJECTS_STILL_EXIST)):
            print(">>> pg_trgm can't be dropped, ignore")
        conn.execute("Rollback")
add_sl_domains()
add_proton_partner()
print("BOOT OK:", app)
print("DB_URI:", os.environ.get("DB_URI"))
from aiosmtpd.smtp import Envelope
from email.message import EmailMessage
import email_handler
from app.models import Alias, Contact, EmailLog, SentAlert
from app.email import status, headers
from app.email_utils import is_reverse_alias
from app.email_validation import normalize_reply_email
from app.mail_sender import mail_sender
from app.config import EMAIL_DOMAIN
from tests.utils import create_new_user
mail_sender.store_emails_instead_of_sending(True)
with app.app_context():
    tok = "".join(random.choices(string.ascii_lowercase + string.digits, k=8))
    user_a = create_new_user()
    user_b = create_new_user()
    Session.commit()
    alias_a = Alias.create_new_random(user_a)
    alias_b = Alias.create_new_random(user_b)
    Session.commit()
    shared_reply = f"dup-reply-{tok}@{EMAIL_DOMAIN}"
    ca = Contact.create(user_id=alias_a.user_id, alias_id=alias_a.id,
                        website_email="a@nowhere.net", name="A", reply_email=shared_reply)
    cb = Contact.create(user_id=alias_b.user_id, alias_id=alias_b.id,
                        website_email="b@nowhere.net", name="B", reply_email=shared_reply)
    Session.commit()
    print("=== SEED ===")
    print(f"EMAIL_DOMAIN={EMAIL_DOMAIN!r}")
    print(f"shared_reply={shared_reply!r}")
    print(f"user_a.id={user_a.id} email={user_a.email!r} alias_a.id={alias_a.id} alias_a.email={alias_a.email!r}")
    print(f"user_b.id={user_b.id} email={user_b.email!r} alias_b.id={alias_b.id} alias_b.email={alias_b.email!r}")
    print(f"contact_a.id={ca.id} (alias_a, user_a)   contact_b.id={cb.id} (alias_b, user_b)")
    print("=== PROVE DUPLICATE PERSISTED (no UNIQUE constraint blocked it) ===")
    rows = Contact.filter_by(reply_email=shared_reply).all()
    print(f"rows sharing reply_email={shared_reply!r}: count={len(rows)}")
    for r in rows:
        print(f"  Contact id={r.id} alias_id={r.alias_id} user_id={r.user_id} website_email={r.website_email!r}")
    print("=== reply-address derivation values ===")
    print(f"rcpt_to (raw)              = {shared_reply!r}")
    print(f"endswith(EMAIL_DOMAIN)     = {shared_reply.endswith(EMAIL_DOMAIN)}")
    print(f"normalize_reply_email(...) = {normalize_reply_email(shared_reply)!r}")
    print("=== routing predicate ===")
    print(f"is_reverse_alias({shared_reply!r}) = {is_reverse_alias(shared_reply)}")
    print("=== [non-canonical] direct Contact.get_by(reply_email=...) ===")
    d = Contact.get_by(reply_email=shared_reply)
    print(f"[non-canonical] returned Contact id={d.id} alias_id={d.alias_id} user_id={d.user_id}")
    def make_msg(i):
        m = EmailMessage()
        m["From"] = "a@nowhere.net"; m["To"] = alias_a.email
        m["Message-ID"] = f"<obs-{tok}-{i}@sl.local>"; m["Subject"] = "reply subject"
        m.set_content("hello body"); return m
    envelope = Envelope(); envelope.mail_from = user_a.email; envelope.rcpt_tos = [shared_reply]
    mid = f"<obs-{tok}-0@sl.local>"; msg = make_msg(0)
    alert_max_before = Session.query(sqlalchemy.func.max(SentAlert.id)).scalar() or 0
    print("=== CANONICAL CALL: email_handler.handle_reply(envelope, msg, shared_reply) ===")
    print(f"envelope.mail_from={envelope.mail_from!r} envelope.rcpt_tos={envelope.rcpt_tos!r} Message-ID={mid!r}")
    delivered, code = email_handler.handle_reply(envelope, msg, shared_reply)
    print(f"RESULT delivered={delivered!r} code={code!r}")
    print(f"code==status.E200? {code==status.E200}  code==status.E214? {code==status.E214}  code==status.E502? {code==status.E502}")
    el = EmailLog.filter_by(message_id=mid).order_by(EmailLog.id.desc()).first()
    if el:
        print(f"PERSISTED EmailLog id={el.id} contact_id={el.contact_id} alias_id={el.alias_id} "
              f"user_id={el.user_id} mailbox_id={el.mailbox_id} is_reply={el.is_reply} message_id={el.message_id!r}")
        print(f"  -> forwarding user_id={el.user_id} (user_a.id={user_a.id}, user_b.id={user_b.id})")
    else:
        print("No EmailLog for this message_id (non-success path).")
    new_alerts = SentAlert.filter_by().filter(SentAlert.id > alert_max_before).all()
    print(f"NEW SentAlert rows during call: {len(new_alerts)}")
    for a in new_alerts:
        print(f"  SentAlert id={a.id} user_id={a.user_id} to_email={a.to_email!r} alert_type={a.alert_type!r}")
    print("DONE")
```

**`/tmp/observe_dist.py`** (cross-event distribution — §7; usage `observe_dist.py <ab|ba> <N>`). Key logic: seed two users/aliases with `disable_email_spoofing_check=True`, insert two contacts sharing `shared_reply` in the given order, print `EXPLAIN ANALYZE` of `SELECT * FROM contact WHERE reply_email=:r LIMIT 1`, then issue `N` identical `handle_reply` calls (identical `mail_from`/`rcpt_to`, only `Message-ID` varies), reading the resolved identity from the newest `EmailLog` per call into a `Counter`:

```python
import os, sys, random, string, logging
from collections import Counter
from app.db import Session, engine
from server import create_app
from init_app import add_sl_domains, add_proton_partner
app = create_app()
app.config["TESTING"] = True; app.config["WTF_CSRF_ENABLED"] = False; app.config["SERVER_NAME"] = "sl.test"
import sqlalchemy
from sqlalchemy import func
from psycopg2 import errors
from psycopg2.errorcodes import DEPENDENT_OBJECTS_STILL_EXIST
with engine.connect() as conn:
    try:
        conn.execute("DROP EXTENSION if exists pg_trgm"); conn.execute("CREATE EXTENSION pg_trgm")
    except sqlalchemy.exc.InternalError as e:
        if isinstance(e.orig, errors.lookup(DEPENDENT_OBJECTS_STILL_EXIST)):
            pass
        conn.execute("Rollback")
add_sl_domains(); add_proton_partner()
from aiosmtpd.smtp import Envelope
from email.message import EmailMessage
import email_handler
from app.models import Alias, Contact, EmailLog
from app.email import status, headers
from app.mail_sender import mail_sender
from app.config import EMAIL_DOMAIN
from tests.utils import create_new_user
mail_sender.store_emails_instead_of_sending(True)
order = sys.argv[1] if len(sys.argv) > 1 else "ab"
N = int(sys.argv[2]) if len(sys.argv) > 2 else 100
with app.app_context():
    tok = "".join(random.choices(string.ascii_lowercase + string.digits, k=8))
    user_a = create_new_user(); user_b = create_new_user(); Session.commit()
    alias_a = Alias.create_new_random(user_a); alias_b = Alias.create_new_random(user_b)
    alias_a.disable_email_spoofing_check = True; alias_b.disable_email_spoofing_check = True
    Session.commit()
    shared_reply = f"dup-reply-{tok}@{EMAIL_DOMAIN}"
    if order == "ab":
        first = Contact.create(user_id=alias_a.user_id, alias_id=alias_a.id, website_email="a@nowhere.net", name="A", reply_email=shared_reply)
        second = Contact.create(user_id=alias_b.user_id, alias_id=alias_b.id, website_email="b@nowhere.net", name="B", reply_email=shared_reply)
    else:
        first = Contact.create(user_id=alias_b.user_id, alias_id=alias_b.id, website_email="b@nowhere.net", name="B", reply_email=shared_reply)
        second = Contact.create(user_id=alias_a.user_id, alias_id=alias_a.id, website_email="a@nowhere.net", name="A", reply_email=shared_reply)
    Session.commit()
    owner_label = {user_a.id: "user_A(alias-owner-matching-mail_from)", user_b.id: "user_B(different-user)"}
    cid_label = {first.id: "FIRST_INSERTED", second.id: "SECOND_INSERTED"}
    print(f"=== DIST EXPERIMENT order={order} N={N} tok={tok} ===")
    print(f"shared_reply={shared_reply!r}")
    print(f"mail_from (IDENTICAL every call) = user_a.email = {user_a.email!r}")
    print(f"user_a.id={user_a.id} alias_a.id={alias_a.id} | user_b.id={user_b.id} alias_b.id={alias_b.id}")
    print(f"INSERTION ORDER: 1st=Contact id={first.id} user_id={first.user_id} ; 2nd=Contact id={second.id} user_id={second.user_id}")
    print("--- EXPLAIN ANALYZE of the equality lookup that .first() runs (LIMIT 1, NO ORDER BY) ---")
    res = Session.execute(sqlalchemy.text("EXPLAIN ANALYZE SELECT * FROM contact WHERE reply_email = :r LIMIT 1"), {"r": shared_reply})
    for line in res:
        print("   ", line[0])
    logging.disable(logging.CRITICAL)
    cid_dist = Counter(); user_dist = Counter()
    for i in range(N):
        before = Session.query(func.max(EmailLog.id)).scalar() or 0
        m = EmailMessage(); m["From"] = user_a.email; m["To"] = shared_reply
        m["Message-ID"] = f"<dist-{tok}-{i}@sl.local>"; m["Subject"] = "s"; m.set_content("b")
        env = Envelope(); env.mail_from = user_a.email; env.rcpt_tos = [shared_reply]
        delivered, code = email_handler.handle_reply(env, m, shared_reply)
        el = (Session.query(EmailLog).filter(EmailLog.id > before).order_by(EmailLog.id.desc()).first())
        if el:
            cid_dist[(el.contact_id, cid_label.get(el.contact_id, "?"))] += 1
            user_dist[(el.user_id, owner_label.get(el.user_id, "?"))] += 1
        else:
            cid_dist[("NO_EMAILLOG", code)] += 1
    logging.disable(logging.NOTSET)
    print(f"--- DISTRIBUTION over N={N} IDENTICAL calls (mail_from={user_a.email!r}, rcpt_to={shared_reply!r}) ---")
    print(f"resolved Contact.id : {dict(cid_dist)}")
    print(f"resolved user_id    : {dict(user_dist)}")
    print("DONE")
```

**`/tmp/observe_edge.py`** (E501/E502×2/E214/normalize/available_sl_email/is_reverse_alias — §8.1). Boots the app once; for each condition builds a minimal `EmailMessage`, an `aiosmtpd` `Envelope`, and calls `email_handler.handle_reply(env, msg, rcpt_to)`, asserting on the returned `(delivered, code)`, the newest `EmailLog`, and new `SentAlert` rows. The E214 block inserts `user_B`'s contact first (so `.first()` resolves `user_B`) with the spoof-check left ON and `mail_from = user_A`'s mailbox. The normalize block creates a contact whose stored `reply_email` contains `_` and drives `handle_reply` with a `rcpt_to` spelled using a disallowed character that normalizes onto it.

**`/tmp/observe_norm.py`** (focused normalize persistence — §8.2) and **`/tmp/observe_second.py`** (second lookup site — §8.3) follow the same bootstrap and drive `handle_reply` canonically; `observe_second.py` puts a second reverse-alias of the same alias in the `To` header to trigger `replace_header_when_reply()`.

---

## 11. Coverage Pass

Every requirement and named item from the prompt/AAP, mapped to the section that answers it:

| Requirement / named item | Section(s) | Status |
|---------------------------|------------|--------|
| Reply-address derivation (`rcpt_to` → domain gate → `normalize_reply_email`) | §3, §3.1 | ✅ |
| Contact resolution via `Contact.get_by(reply_email=...)` | §4.1, §4.2 | ✅ |
| `ModelMixin.get_by()` `.first()` no-`ORDER BY` semantics | §4.1, §6.3, §7.5 | ✅ |
| Extracted/normalized reply value (actual runtime value) | §3.1, §4.2 | ✅ |
| Resolved `Contact` `id` / `alias_id` / `user_id` | §4.2 | ✅ |
| Forwarding destination (alias / user / mailbox) | §5, §5.1 | ✅ |
| Resulting `EmailLog` (`contact_id`/`alias_id`/`user_id`/`mailbox_id`) | §5.1, §4.2 | ✅ |
| Cross-event behavior: same reply → different users over time | §7.1–§7.3, §7.5 | ✅ |
| Root cause — uniqueness (no `UNIQUE` on `reply_email`) | §6.1 | ✅ |
| Root cause — race/TOCTOU (`available_sl_email`) | §6.2 | ✅ (concurrency window `[inferred]`) |
| Root cause — timing/ordering (`.first()` arbitrary row) | §6.3, §7.4, §7.5 | ✅ (observed) |
| Wrong-user causally explained (cause → effect) | §6.3–§6.4, §7.3 | ✅ |
| E501 (reply domain not `EMAIL_DOMAIN`/`SLDomain`) | §8.1 | ✅ |
| E502 — no contact | §8.1 | ✅ |
| E502 — inactive/soft-deleted user | §8.1 | ✅ |
| E214 — unknown mailbox + alert to resolved owner | §8.1 | ✅ |
| `normalize_reply_email()` many-to-one collision | §8.1, §8.2 | ✅ |
| Second lookup site `replace_header_when_reply()` (`:364`/`:380`) | §8.3 | ✅ |
| Routing/bounce lookups (`:1998`, `:2009`, `:2167`) | §6.5 | ✅ (`file:line`; `[inferred]` reuse) |
| Forward-phase minting (`:299`) + `generate_reply_email` | §6.7 | ✅ |
| `is_reverse_alias()` routing predicate | §4.2, §8.1, §6.5 | ✅ |
| Uniqueness proof (model `:1875`/`:1899` + migration `:22`) | §6.1 | ✅ |
| Reverse-alias intended invariant (web) vs schema reality | §9 | ✅ |
| Canonical vs non-canonical distinction | §2.5, §4.2 | ✅ |
| Scale stated (N=100) + stability across ≥2 runs | §2.5, §7.1–§7.3 | ✅ |
| Repository left unchanged (only this doc) | §12 | ✅ |

**Labeling summary:** the ordering-dependent wrong-user behavior, the duplicate persistence, the resolved values, all error codes, the normalization collision, and the second lookup are **observed**. The TOCTOU *concurrency* window and the "could resolve differently under other physical/plan conditions" nuance are **`[inferred]`** (grounded in code + confirmed absence of a `UNIQUE` constraint + the forced-plan `EXPLAIN`). The direct `Contact.get_by` corroboration in §4.2 is **`[non-canonical]`**.


---

## 12. Repository Invariant

The investigation was read-only. All temporary observation scripts (`/tmp/observe_reply.py`, `/tmp/observe_dist.py`, `/tmp/observe_edge.py`, `/tmp/observe_norm.py`, `/tmp/observe_second.py`) lived **outside** the repository tree and were deleted after the observations were captured. No product/source/migration/test/configuration file was modified, and no remediation (no `UNIQUE` constraint, no `ORDER BY`, no atomic guard, no migration, no refactor) was performed.

The closing check below confirms the working tree contains **only** this new answer document.

**Command:**

```
git status --porcelain -uall
```

**Output (complete, unedited):**

```
?? blitzy/documentation/app_2cd6ee777f8c.md
```

`git status` shows a single untracked file — `blitzy/documentation/app_2cd6ee777f8c.md` — and nothing else. (`-uall` lists individual untracked files; the empty `blitzy/screenshots/` and `blitzy/screen_recordings/` directories contain no files and are therefore not reported by git.) The repository is otherwise pristine; the temporary observation scripts under `/tmp/` were removed.

