# SimpleLogin Email-Forwarding Flow — Forensic Q&A (branch `app_2cd6ee777f8c`)

> A runtime-grounded investigation of SimpleLogin's inbound email-forwarding flow.
> Every log line, `Message-ID`, `From` value, reverse-alias address, database row id, and
> timestamp reported below is a **genuine value captured from a live run** of the application —
> not a value inferred by reading code alone. Source-line citations such as `[email_handler.py:L928]`
> point only at the *origin* of each captured value so it is traceable back to the code.

---

## Table of contents

- [(0) Introduction & Methodology](#0-introduction--methodology)
- [(1) Q1 — Log messages: successful forward vs. non-existent alias](#1-q1--log-messages-successful-forward-vs-non-existent-alias)
- [(2) Q2 — SL Message-ID vs. the sender's original Message-ID](#2-q2--sl-message-id-vs-the-senders-original-message-id)
- [(3) Q3 — Transformed `From` header and reverse-alias format](#3-q3--transformed-from-header-and-reverse-alias-format)
- [(4) Q4 — Database records created by one forward](#4-q4--database-records-created-by-one-forward)
- [Reproduction & cleanup](#reproduction--cleanup)

---

## (0) Introduction & Methodology

### The questions under investigation

This document answers four questions about what SimpleLogin actually does, at runtime, when an
inbound email is delivered to an alias:

1. **(Q1)** What is the exact log message when an email is (a) successfully forwarded to its
   mailbox, versus (b) rejected because the alias does not exist?
2. **(Q2)** What SimpleLogin `Message-ID` is generated during processing, and how does it differ
   from the sender's original `Message-ID` header?
3. **(Q3)** What is the exact `From` header on the outbound (forwarded) email after transformation,
   and what is the format of the reverse-alias / reply address that replaces the original sender?
4. **(Q4)** Which database records are created during a single forward operation (with their actual
   primary-key ids and `created_at` timestamps)?

### Investigated commit

- **Branch:** `app_2cd6ee777f8c`
- **HEAD:** `2cd6ee77` — *"chore: emit some missing contact audit logs (#2269)"*
- **Working tree:** verified clean at runtime (`git status --porcelain` produced no output); this
  exact commit was checked out for the entire investigation. No source file was modified — the only
  artifact produced anywhere is **this document**.

### Runtime harness

The values below were captured by **building and running** SimpleLogin, not by static reasoning:

- A **`postgres:13`** database container, standing up the exact pattern used by the project's own
  test runner `scripts/run-test.sh`: container named **`sl-test-db`**, host port **`15432`** mapped to
  container port `5432`, with `POSTGRES_USER=test` / `POSTGRES_PASSWORD=test` / `POSTGRES_DB=test`.
- **Python 3.10** (the version pinned by `Dockerfile` `FROM python:3.10` `[Dockerfile:L8]` and by CI),
  with dependencies pinned from `poetry.lock` — notably **`SQLAlchemy 1.3.24`**, **`Flask 1.1.2`**,
  **`aiosmtpd 1.4.2`**, **`flanker 0.9.11`**, and **`arrow 0.16.0`**.
- Schema materialized via `CONFIG=tests/test.env poetry run alembic upgrade head` (Alembic head
  `32f25cbf12f6`).
- Runtime configuration loaded from **`tests/test.env`**:
  `EMAIL_DOMAIN=sl.local` `[tests/test.env:L8]`, `NOT_SEND_EMAIL=true` `[tests/test.env:L7]`,
  `DB_URI=postgresql://test:test@localhost:15432/test` `[tests/test.env:L17]`, and
  `MEM_STORE_URI=redis://localhost` `[tests/test.env:L78]`.

### How the flow was driven

The inbound SMTP handler `email_handler.handle(envelope, msg)` was invoked **directly**, mirroring the
pattern already used throughout `tests/test_email_handler.py` (build an `aiosmtpd` `Envelope`, set
`envelope.mail_from` and `envelope.rcpt_tos`, build the MIME `Message`, then call
`email_handler.handle(...)`) `[tests/test_email_handler.py:L273-L307]`. The outbound **transformed**
message was captured **in-process** through the mail-sender store-mode — the
`@mail_sender.store_emails_test_decorator` decorator `[app/mail_sender.py:L111]` together with
`mail_sender.get_stored_emails()` `[app/mail_sender.py:L108]` — so that **no real SMTP delivery
occurred** (consistent with `NOT_SEND_EMAIL=true`). Database rows were read from the live PostgreSQL
instance during the probe.

Three scenarios were exercised against this single coherent snapshot:

- **(A)** a forward to a **valid** alias from a **previously-unseen sender** (drives contact creation —
  the most informative case for Q3 and Q4);
- **(B)** a forward to a **non-existent** alias (drives the Q1 failure branch); and
- **(C)** a single **reply** to a contact's reverse-alias — included **only** to contrast the
  `Message-ID` behavior for Q2.

### Transparency note (environment substitutions that do not affect any reported value)

In full honesty: two dependencies were substituted **in the throwaway test environment only**, and
neither affects any value reported in this document. `pyre2` (a libre2 binding) was replaced by a
thin standard-library `re` shim — the regular-expression *engine* is irrelevant to email routing and
header transformation logic — and `cbor2` resolved to `5.6.4` (used solely by the FIDO2/WebAuthn code
paths, which are unrelated to email forwarding). **No source file was modified.** The probe code ran
from a writable working copy with the repository source treated strictly read-only, and any probe
code lived **outside** the repository tree. `git status --porcelain` confirmed the source tree was
byte-for-byte unchanged after the run.

### Disambiguation: the per-email trace UUID is NOT the email `Message-ID`

A point that is easy to conflate, and that matters for Q2: the transport-layer entry point
`MailHandler._handle()` mints a fresh UUID at the start of every message —
`message_id = str(uuid.uuid4())` `[email_handler.py:L2339]` — and registers it via
`set_message_id(message_id)` `[email_handler.py:L2340]`. This UUID is used **purely for log
correlation** (it prefixes the log lines for a given message so concurrent deliveries can be told
apart). It is **not** the email's `Message-ID` header, and it is **not** the "SL Message-ID" minted on
the reply path (Q2). This document never conflates the two.

---

## (1) Q1 — Log messages: successful forward vs. non-existent alias

### 1.1 Success path — forward accepted (`250`)

**Scenario:** a forward to a **valid** alias `rotary_carafe092@sl.local` from a **new** sender
`bob_skaliz@external-example.com` (display name `Bob Sender`), owned by user *Alice Owner* whose
mailbox is `owner_gyhgrvmq@mailbox.test`.

**Final SMTP return code (captured):**

```text
250 Message accepted for delivery
```

This is `status.E200`, defined exactly as `E200 = "250 Message accepted for delivery"`
`[app/email/status.py:L2]`. It is returned by `forward_email_to_mailbox()` as
`return True, status.E200` `[email_handler.py:L928]`, immediately after `Session.commit()`
`[email_handler.py:L927]`.

**Captured log lines, in the exact order emitted** (SimpleLogin's module-level `LOG`, at DEBUG/INFO
level). Each line is reproduced verbatim:

```text
==>> Handle mail_from:bob_skaliz@external-example.com, rcpt_tos:['rotary_carafe092@sl.local'], header_from:Bob Sender <bob_skaliz@external-example.com>, header_to:rotary_carafe092@sl.local, cc:None, reply-to:None, message_id:<orig.8e0ff55d69e242e795d10f30b9bc4f8a@external-example.com>, ...
```
↳ emitted by `handle()` `[email_handler.py:L1980]` (the `==>> Handle ...` format string sits on the
following line, `[email_handler.py:L1981]`).

```text
Forward phase bob_skaliz@external-example.com(Bob Sender <bob_skaliz@external-example.com>) -> rotary_carafe092@sl.local
```
↳ `handle()` `[email_handler.py:L2202]`.

```text
Create or get contact for from_header:Bob Sender <bob_skaliz@external-example.com>
```
↳ `handle_forward()` `[email_handler.py:L580]`.

```text
Created contact <Contact 1 bob_skaliz@external-example.com 2> for alias <Alias 2 rotary_carafe092@sl.local> with email bob_skaliz@external-example.com invalid_email=False
```
↳ `create_contact()` `[app/contact_utils.py:L110]`.

```text
Forward <Contact 1 bob_skaliz@external-example.com 2> -> <Alias 2 rotary_carafe092@sl.local> -> <Mailbox 1 owner_gyhgrvmq@mailbox.test>
```
↳ `forward_email_to_mailbox()` `[email_handler.py:L688]`.

```text
Create <EmailLog 1> for <Contact 1 bob_skaliz@external-example.com 2>, <User 1 Alice Owner owner_gyhgrvmq@mailbox.test>, <Mailbox 1 owner_gyhgrvmq@mailbox.test>
```
↳ `forward_email_to_mailbox()` `[email_handler.py:L740]`.

```text
From header, new:"Bob Sender - bob_skaliz at external-example.com" <bob_skaliz_at_external-example_com_vukwpsmrkk@sl.local>, old:Bob Sender <bob_skaliz@external-example.com>
```
↳ `forward_email_to_mailbox()` `[email_handler.py:L867]`.

```text
Forward mail from bob_skaliz@external-example.com to owner_gyhgrvmq@mailbox.test, mail_options:[], rcpt_options:[]
```
↳ `forward_email_to_mailbox()` `[email_handler.py:L893]`.

```text
send email with subject 'Hello through my alias', from '"Bob Sender - bob_skaliz at external-example.com" <bob_skaliz_at_external-example_com_vukwpsmrkk@sl.local>' to 'rotary_carafe092@sl.local'
```
↳ `mail_sender.send()` `[app/mail_sender.py:L131]`.

> A single DMARC-related INFO line — `LOG.i("DMARC check disabled")` `[app/handler/dmarc.py:L33]` — also
> appears incidentally in the captured forward log. Although `tests/test.env` sets `DMARC_CHECK_ENABLED=true`
> `[tests/test.env:L66]`, this line fires via the **second** half of the guard
> `if not DMARC_CHECK_ENABLED or not spam_result:` `[app/handler/dmarc.py:L32]`: the bare test message
> carries no spam-check headers, so `spam_result` is `None` (`[app/handler/spamd_result.py:L83-L85]`),
> making `not spam_result` true. It is noted here only for completeness; DMARC handling is out of scope
> for this investigation.

### 1.2 Failure path — alias does not exist (`550`)

**Scenario:** a forward to a **non-existent** alias `nope_xqlqxkko@sl.local`.

**Final SMTP return code (captured):**

```text
550 SL E515 Email not exist
```

This is `status.E515`, defined exactly as `E515 = "550 SL E515 Email not exist"`
`[app/email/status.py:L51]`, returned as `return [(False, status.E515)]` `[email_handler.py:L555]`.

**Captured log lines, in the exact order emitted** (verbatim):

```text
alias nope_xqlqxkko@sl.local not exist. Try to see if it can be created on the fly
```
↳ `handle_forward()` `[email_handler.py:L545]`.

```text
Cannot auto-create custom domain alias for nope_xqlqxkko@sl.local because there's no custom domain for sl.local
```
↳ `check_if_alias_can_be_auto_created_for_custom_domain()` `[app/alias_utils.py:L104]`.

```text
Cannot auto-create nope_xqlqxkko@sl.local since it has no directory separator
```
↳ `check_if_alias_can_be_auto_created_for_a_directory()` `[app/alias_utils.py:L165]`.

```text
alias nope_xqlqxkko@sl.local cannot be created on-the-fly, return 550
```
↳ `handle_forward()` `[email_handler.py:L551]`.

### 1.3 Rationale / How we know (Q1)

- **The success terminus is explicit in the code.** `forward_email_to_mailbox()` ends the happy path
  with `Session.commit()` `[email_handler.py:L927]` followed by `return True, status.E200`
  `[email_handler.py:L928]`. Because `status.E200` is literally the string
  `"250 Message accepted for delivery"` `[app/email/status.py:L2]`, that exact string is the SMTP
  reply the sending MTA receives. The nine log lines above are the genuine, ordered output observed
  for the accepted forward; each is anchored to the `LOG.x(...)` call that produced it.
- **The failure terminus is the alias-resolution branch.** In `handle_forward()`, the alias is looked
  up with `Alias.get_by(email=alias_address)` `[email_handler.py:L543]`; when it is missing, the code
  logs *"alias … not exist. Try to see if it can be created on the fly"* `[email_handler.py:L545]`,
  attempts on-the-fly creation via `try_auto_create()` `[email_handler.py:L549]`, and — because the
  test domain `sl.local` is a SimpleLogin alias domain with **no** matching custom-domain rule
  `[app/alias_utils.py:L104]` and **no** directory separator in the local part `[app/alias_utils.py:L165]` —
  that attempt returns nothing. The handler then logs
  *"alias … cannot be created on-the-fly, return 550"* `[email_handler.py:L551]` and returns
  `status.E515` `[email_handler.py:L555]`, i.e. `"550 SL E515 Email not exist"`
  `[app/email/status.py:L51]`.
- **Authoring note — transport-layer boundary logs.** When `email_handler.handle()` is driven
  *directly* (as it is here, and as it is throughout `tests/test_email_handler.py`), it does **not**
  itself emit the outermost boundary logs `New message, mail from …` `[email_handler.py:L2343]` and
  `Finish mail_from …, takes … seconds with return code '%s'<<===` `[email_handler.py:L2367]`. Those two
  lines belong to the `aiosmtpd` **transport layer** — `MailHandler._handle()`, whose class is defined
  at `[email_handler.py:L2288]` — which *wraps* `handle()` in production. In a production run those
  framing lines would surround the per-message logs above, reporting the final return code (`'250'`
  on success, `'550'` on the failure path) `[email_handler.py:L2367-L2373]`. They are presented here as
  transport-layer **framing**; the `L1980` / `L545` / `L551` / etc. lines above are the genuinely
  captured per-message logs from the direct-`handle()` probe.


---

## (2) Q2 — SL Message-ID vs. the sender's original Message-ID

The headline result corrects a common misconception:

> **A pure *forward* does NOT generate a new `Message-ID` — it preserves the sender's original.
> A new "SL Message-ID" is minted only on the *reply* path.**

### 2.1 Captured — a forward PRESERVES the original `Message-ID`

For the forward scenario, the inbound original `Message-ID`:

```text
<orig.8e0ff55d69e242e795d10f30b9bc4f8a@external-example.com>
```

appeared **identically** on the outbound forwarded message, and `EmailLog #1.message_id` stored that
same original value. The forwarder explicitly carries the sender's `Message-ID` through:
`forward_email_to_mailbox()` calls `EmailLog.create(..., message_id=str(msg[headers.MESSAGE_ID]), ...)`
`[email_handler.py:L732-L739]` — it persists the *original* header and only rewrites `In-Reply-To` /
`References` for threading. It does **not** mint a new `Message-ID`, and **no `message_id_matching`
row is created on a forward.**

### 2.2 Captured — a reply MINTS a new "SL Message-ID"

Driving a reply (from the mailbox back to the contact's reverse-alias) produced a new SimpleLogin
`Message-ID`:

```text
inbound original Message-ID : <reply.orig.7e1a27db3d354ee1a2e667e8e5c08211@mailbox.test>
minted SL Message-ID        : <178250299191.8674.9648925825288615843.2@sl.local>
```

The minting was visible in the log:

```text
create a new sl_message_id <178250299191.8674.9648925825288615843.2@sl.local>
```
↳ `replace_original_message_id()` `[email_handler.py:L1314]`.

…and the correlation was persisted as a `MessageIDMatching` row (model at `[app/models.py:L3365]`):

```text
MessageIDMatching:
  id                  = 1
  sl_message_id       = <178250299191.8674.9648925825288615843.2@sl.local>
  original_message_id = <reply.orig.7e1a27db3d354ee1a2e667e8e5c08211@mailbox.test>
  email_log_id        = 2
  created_at          = 2026-06-26T19:43:11.917109+00:00
```

### 2.3 Rationale / How we know — how it differs (Q2)

- **Where the SL Message-ID comes from.** On the reply path,
  `replace_original_message_id()` generates it with
  `make_msgid(str(email_log.id), get_email_domain_part(alias.email))` `[email_handler.py:L1311-L1313]`.
  Python's `email.utils.make_msgid(idstring, domain)` produces a value of the shape
  `<{timestamp}.{pid}.{random}.{idstring}@{domain}>`. Decoding the captured
  `<178250299191.8674.9648925825288615843.2@sl.local>` against that template gives
  `idstring = "2"` — i.e. `str(email_log.id)`, the reply's `EmailLog` id — and `domain = sl.local`,
  the alias's domain.
- **How it differs from the original.** The SL Message-ID therefore (a) **embeds the `EmailLog` id**
  as its trailing dotted component, and (b) is **rooted at the SimpleLogin alias domain
  (`sl.local`)**. The sender's original `Message-ID`, by contrast, is rooted at the sender's / mailbox's
  own domain — `external-example.com` for the forward, `mailbox.test` for the reply — and carries no
  SimpleLogin-internal id. That is the precise, structural difference.
- **Why a correlation record exists.** The original and the SL Message-ID are linked through the
  `message_id_matching` table so SimpleLogin can later map a reply or a bounce back to the original
  message. If a matching row already exists for an `original_message_id`, the existing `sl_message_id`
  is **reused** rather than re-minted `[email_handler.py:L1307-L1309]`; otherwise a new one is minted
  and recorded `[email_handler.py:L1316-L1321]`.
- **The pivotal nuance (stated explicitly).** *Forwarding does not generate a new `Message-ID`; it keeps
  the sender's original. The SL Message-ID is minted only on the reply path.* This is exactly why the
  reply scenario had to be exercised in addition to the forward — there is no SL Message-ID to observe
  on a pure forward.
- **Not to be confused with the trace UUID.** The SL Message-ID is unrelated to the per-email
  log-correlation UUID minted at `[email_handler.py:L2339]` (see §0's disambiguation).


---

## (3) Q3 — Transformed `From` header and reverse-alias format

### 3.1 Captured

**Relevant runtime user attributes** for the alias owner (*Alice Owner*):

```text
sender_format                   = 0      # = SenderFormatEnum.AT  [app/models.py:L204]
include_sender_in_reverse_alias = True   # Python model default   [app/models.py:L455]
```

**Transformed `From` header on the stored outbound forwarded message:**

```text
"Bob Sender - bob_skaliz at external-example.com" <bob_skaliz_at_external-example_com_vukwpsmrkk@sl.local>
```

**Reverse-alias / reply address (`Contact.reply_email`):**

```text
bob_skaliz_at_external-example_com_vukwpsmrkk@sl.local
```

### 3.2 Rationale / How we know — format explanation (Q3)

- **How the `From` phrase is built.** `Contact.new_addr()` `[app/models.py:L2008-L2046]` constructs the
  rewritten `From` according to the owner's `sender_format`. Here `sender_format == SenderFormatEnum.AT.value == 0`
  `[app/models.py:L204]`, so the **AT** branch runs `[app/models.py:L2028-L2034]`:
  it computes `formatted_email = website_email.replace("@", " at ").strip()` and, because the contact
  has a display name (`Bob Sender`) distinct from its email, sets
  `new_name = "{name} - {formatted_email}"`, then returns `sl_formataddr((new_name, reply_email))`
  `[app/models.py:L2045]`. Concretely:
  - display name → `Bob Sender`
  - `bob_skaliz@external-example.com` → `bob_skaliz at external-example.com` (the `@` becomes ` at `)
  - phrase → `Bob Sender - bob_skaliz at external-example.com`
  - addressed at the reverse-alias → the captured `From` header above.

- **How the reverse-alias is generated.** The reverse-alias is produced by `generate_reply_email()`
  `[app/email_utils.py:L1103-L1153]`. Because the owner's `include_sender_in_reverse_alias` is `True`,
  the **sender-embedding** branch runs `[app/email_utils.py:L1137-L1143]`. The sender address is first
  normalized `[app/email_utils.py:L1119-L1127]` — `convert_to_id` → `sanitize_email` → truncate to 45
  chars → replace `@` with `_at_` → replace `.` with `_` → `convert_to_alphanumeric` — and then a random
  suffix is appended: `f"{contact_email}_{random_string(random_length)}@{reply_domain}"` with
  `random_length = random.randint(5, 10)`. For our sender this composes as:

  ```text
  bob_skaliz  +  _at_  +  external-example_com  +  _vukwpsmrkk  +  @sl.local
  └ local part ─────────────────────────────┘  └ random(5..10) ┘  └ EMAIL_DOMAIN ┘
  = bob_skaliz_at_external-example_com_vukwpsmrkk@sl.local
  ```

  The random suffix `vukwpsmrkk` is 10 characters — within the `random.randint(5, 10)` range. There is
  **no `ra+` / `reply+` prefix on the generated reverse-alias**: in `generate_reply_email()` the legacy
  prefixed-generation is commented out — the include-sender branch at `[app/email_utils.py:L1140-L1141]`
  and the bare-random branch at `[app/email_utils.py:L1146-L1147]`, **both referencing only `ra+`** — so
  the live generation lines (`[app/email_utils.py:L1142]` and `[app/email_utils.py:L1148]`) emit a
  prefix-less local part. The `ra+` and `reply+` tokens survive elsewhere **only** as backward-compat
  *recognition* tokens in `is_reverse_alias()` `[app/email_utils.py:L1162]`
  (`address.startswith("reply+") or address.startswith("ra+")`); neither is generated by the modern code.

- **Transparency — this diverges from the static assumption in AAP §0.5.4.** The plan's static reading
  assumed a "prefix-less random local part" with the sender **not** embedded. The *runtime* truth is
  different: because `include_sender_in_reverse_alias = True`, the sender **is** embedded in the
  reverse-alias. The governing rule is to report the **actual generated** address, which is the
  sender-embedding form captured above — not the legacy/assumed bare-random form. The reason the
  runtime default is `True` is subtle and worth stating: the column is declared
  `include_sender_in_reverse_alias = sa.Column(sa.Boolean, default=True, nullable=False, server_default="0")`
  `[app/models.py:L455-L457]`. For a `User` created through the ORM (as in the harness), the SQLAlchemy
  **Python-side `default=True`** is what is applied; the column's `server_default="0"` only governs a
  raw SQL insert that bypasses the ORM. Hence the ORM-created owner has `include_sender_in_reverse_alias = True`.

- **The alternative branch, for completeness.** Had `include_sender_in_reverse_alias` been `False`, the
  else-branch `[app/email_utils.py:L1144-L1148]` would instead yield a bare
  `f"{random_string(random_length)}@{reply_domain}"` with `random_length = random.randint(20, 50)` — i.e.
  a prefix-less, sender-less random local part at `@sl.local`. That path was **not** taken in this run.


---

## (4) Q4 — Database records created by one forward

### 4.1 Captured — exactly 3 rows for a forward from a NEW sender

A single forward from a previously-unseen sender created **exactly three** database rows. Listed in
creation order (by `created_at`):

```text
1) Contact
   id              = 1
   website_email   = bob_skaliz@external-example.com
   reply_email     = bob_skaliz_at_external-example_com_vukwpsmrkk@sl.local
   automatic_created = True
   created_at      = 2026-06-26T19:43:11.844665+00:00
   origin: create_contact() -> Contact.create(...)   [app/contact_utils.py:L92-L103]

2) UserAuditLog
   id          = 1
   action      = create_contact
   message     = "Created contact 1 (bob_skaliz@external-example.com)"
   created_at  = 2026-06-26T19:43:11.850017+00:00
   origin: emit_user_audit_log(action=UserAuditLogAction.CreateContact,
                               message=f"Created contact {contact.id} ({contact.email})")
           [app/contact_utils.py:L104-L109]

3) EmailLog
   id          = 1
   message_id  = <orig.8e0ff55d69e242e795d10f30b9bc4f8a@external-example.com>
   is_reply    = False
   contact_id  = 1
   created_at  = 2026-06-26T19:43:11.861468+00:00
   origin: EmailLog.create(..., message_id=str(msg[headers.MESSAGE_ID]), commit=True)
           [email_handler.py:L732-L739]
```

### 4.2 Rationale / How we know (Q4)

- **Why these three, in this order.** On the **first** contact from a sender, `create_contact()`
  inserts the `Contact` row with `automatic_created=True` `[app/contact_utils.py:L92-L103]` and an
  accompanying `UserAuditLog` of action `CreateContact` `[app/contact_utils.py:L104-L109]`; the forward
  then inserts one `EmailLog` `[email_handler.py:L732-L739]`. The `message` of the audit row,
  `"Created contact 1 (bob_skaliz@external-example.com)"`, is produced verbatim by the f-string
  `f"Created contact {contact.id} ({contact.email})"` at `[app/contact_utils.py:L107]`.
- **Why the ids and timestamps look the way they do.** All three models inherit `ModelMixin`, which
  defines an autoincrement integer primary key
  `id = sa.Column(sa.Integer, primary_key=True, autoincrement=True)` `[app/models.py:L63]` and
  `created_at = sa.Column(ArrowType, default=arrow.utcnow, nullable=False)` `[app/models.py:L64]`.
  Because `arrow.utcnow()` is evaluated at insert time, the timestamps are strictly increasing in
  insert order — Contact (`…844665`) → UserAuditLog (`…850017`) → EmailLog (`…861468`) — which is
  exactly what was observed.
- **Contrast — a forward from an *existing* contact creates only ONE row.** When the sender's contact
  already exists, `create_contact()` short-circuits to the existing contact (no new `Contact`, no new
  `UserAuditLog`) and the forward inserts only a new `EmailLog`. The separately-captured **reply**
  scenario corroborates this: it added `EmailLog id=2` plus `MessageIDMatching id=1` (see Q2) and
  created **no** new `Contact` and **no** new `UserAuditLog`, precisely because the contact already
  existed. So the per-forward record count is **3 for a new sender, 1 for a known sender.**
- **Authoring caveat — how the live ids/timestamps were read.** The `flask_client` fixture in
  `tests/conftest.py` wraps each test in `connection.begin()` `[tests/conftest.py:L61]` and **rolls
  back** at teardown (`transaction.rollback()` `[tests/conftest.py:L75]` / `Session.rollback()`
  `[tests/conftest.py:L76]`), so probe rows do not persist after the test. The ids and `created_at`
  values above were therefore read **during** the probe (before rollback). They are genuine live-DB
  autoincrement values: on a freshly migrated, empty database the first forward yields `Contact id=1`
  / `UserAuditLog id=1` / `EmailLog id=1`, consistent with the snapshot above.


---

## Reproduction & cleanup

**Reproduce** by mirroring the project's own test-DB lifecycle in `scripts/run-test.sh`:

```bash
# 1) Stand up an ephemeral PostgreSQL 13 (same image/port/credentials as scripts/run-test.sh)
docker run -d --name sl-test-db \
  -e POSTGRES_PASSWORD=test -e POSTGRES_USER=test -e POSTGRES_DB=test \
  -p 15432:5432 postgres:13

# 2) Give the container a moment to accept connections
sleep 3

# 3) Materialize the schema at Alembic head, using the test config
CONFIG=tests/test.env poetry run alembic upgrade head

# 4) Drive email_handler.handle(envelope, msg) under mail_sender store-mode
#    (the @mail_sender.store_emails_test_decorator + mail_sender.get_stored_emails()
#     pattern from tests/test_email_handler.py), then read:
#      - the emitted LOG.d / LOG.i lines              (Q1)
#      - inbound vs. outbound Message-ID + matching    (Q2)
#      - the stored outbound message's From header     (Q3)
#      - the Contact / UserAuditLog / EmailLog rows    (Q4)
```

Because `tests/test.env` sets `NOT_SEND_EMAIL=true`, no real SMTP delivery occurs; the transformed
outbound message is retained in-process for inspection.

**Cleanup** (honoring the constraint *"clean up any test containers or DB instances you spin up"*):

```bash
docker rm -f sl-test-db        # remove the ephemeral PostgreSQL container
git status --porcelain         # confirm the source tree is byte-for-byte unchanged (no output)
```

After teardown, `git status --porcelain` reports no source-tree changes — the only artifact produced
by this investigation is **this document**, `blitzy/documentation/app_2cd6ee777f8c.md`.

---

### Citation index (origins of the captured values)

| Concern | Primary source citations |
|---|---|
| SMTP status strings (`E200`, `E515`) | `app/email/status.py:L2`, `app/email/status.py:L51` |
| Forward success/return path & logs | `email_handler.py:L580`, `:L688`, `:L740`, `:L867`, `:L893`, `:L927`, `:L928` |
| Alias-not-found failure branch | `email_handler.py:L545`, `:L551`, `:L555`; `app/alias_utils.py:L104`, `:L165` |
| Forward preserves original Message-ID | `email_handler.py:L732-L739` |
| Reply mints SL Message-ID + matching | `email_handler.py:L1311-L1313`, `:L1314`, `:L1316-L1321`; `app/models.py:L3365` |
| Trace-UUID vs. Message-ID disambiguation | `email_handler.py:L2339`, `:L2340` |
| Transport-layer boundary logs | `email_handler.py:L2288`, `:L2343`, `:L2367-L2373` |
| Rewritten `From` header (AT format) | `app/models.py:L2008-L2046`, `:L204` |
| Reverse-alias generation & recognition | `app/email_utils.py:L1103-L1153` (sender-embed `:L1137-L1143`); recognition `is_reverse_alias` `:L1156-L1163` |
| `include_sender_in_reverse_alias` default | `app/models.py:L455` |
| Contact + audit-log creation | `app/contact_utils.py:L92-L103`, `:L104-L109`, `:L110` |
| `ModelMixin` id / `created_at` | `app/models.py:L63`, `:L64` |
| Capture mechanism (store-mode) | `app/mail_sender.py:L108`, `:L111`, `:L131` |
| Harness pattern & rollback fixture | `tests/test_email_handler.py:L273-L307`; `tests/conftest.py:L61`, `:L75`, `:L76` |
| Runtime config | `tests/test.env:L7`, `:L8`, `:L17`, `:L66`, `:L78` |
| Build/run + teardown pattern | `scripts/run-test.sh` |

*All values reported above are genuine values captured from a single coherent live-run snapshot
(timestamps ~`2026-06-26T19:43:11Z`). Re-running the harness yields different non-deterministic
random suffixes and timestamps; the code-level structure and behavior, however, are invariant and are
what the citations point to.*

