# SimpleLogin Alias Reply-Handling Pipeline — Runtime Analysis

*A code-grounded investigation of the SimpleLogin inbound-SMTP reply path, produced by static analysis of the source tree and in-process runtime simulation.*

This document answers the five questions posed in the prompt — (1) which part of the system handles an incoming reply, (2) how the alias is resolved to a user, (3) what user ID the system ultimately decides to forward the reply to, (4) the actual end-to-end data flow in detail, and (5) the most likely point where incorrect routing could originate. Every claim below is anchored to a specific `path:line` reference in the SimpleLogin codebase. The reply-phase test suite was executed (`pytest tests/test_email_handler.py -k reply` — 6 passed, 17 deselected) and a synthetic inbound reply was dispatched through `email_handler.handle()` inside the Flask app context to verify runtime behavior against the claims in this document.

---

## Section 1 — Executive Summary

- **Entry point**: Inbound SMTP DATA commands land on `MailHandler.handle_DATA()` at `email_handler.py:2289`, which parses the raw message body via `email.message_from_bytes()` at `email_handler.py:2290` and delegates to `self._handle(envelope, msg)` at `email_handler.py:2292`.
- **Core reply handler**: `handle_reply(envelope, msg, rcpt_to)` at `email_handler.py:966` owns every decision specific to the reply phase — domain validation, contact lookup, alias resolution, mailbox authorization, header rewriting, and final delivery.
- **Identity-resolution chain**: `rcpt_to (reverse-alias)` → `Contact.reply_email` → `contact.alias_id` → `Alias` → `alias.user_id` → `User`. This is a strict foreign-key walk using the indexed `Contact.reply_email` column at `app/models.py:1899`, the `Contact.alias_id` FK at `app/models.py:1881–1883`, and the `Alias.user_id` FK at `app/models.py:1474–1476`.
- **Final destination**: The reply is delivered to the external third-party's real email address — `contact.website_email` — via `sl_sendmail()` at `email_handler.py:1224–1231`. It is **not** forwarded to any SimpleLogin user mailbox; the user-ID recorded on the `EmailLog` is `contact.user_id` and exists only for bookkeeping.
- **Most likely failure point (preview)**: The combination of (a) the multi-mailbox `notify_mailbox()` fan-out at `email_handler.py:1234–1236` (which delivers a *copy* of every reply to every additional verified mailbox on the alias) and (b) the silent fallback to `alias.mailbox` when `alias.disable_email_spoofing_check == True` at `email_handler.py:1025–1029` (which records the primary mailbox on the `EmailLog` even when the real sender is unknown). A runtime simulation with a two-mailbox alias confirmed that 2 outbound SMTP messages are produced — one primary delivery and one `notify_mailbox` copy — each with distinct VERP envelope-from addresses.

---

## Section 2 — Which Part of the System Handles the Incoming Reply?

SimpleLogin splits its workload across five independent processes: the web app (Flask), the SMTP handler, the cron scheduler, the job runner, and the event listener. The **SMTP handler is a standalone process** defined in `email_handler.py` at the root of the repository. All inbound reply routing happens inside that process — the Flask web app is NOT involved in per-message dispatch (although a minimal Flask context IS established for ORM access, as described below).

### 2.1 Process Entry — aiosmtpd Controller on Port 20381

The SMTP handler runs as an `aiosmtpd` controller bound to `0.0.0.0:20381` by default. `main(port: int)` at `email_handler.py:2381` constructs `controller = Controller(MailHandler(), hostname="0.0.0.0", port=port)` at `email_handler.py:2383` and starts it. The CLI declaration `parser.add_argument("-p", "--port", ..., default=20381)` at `email_handler.py:2399` establishes the default port. Postfix (the upstream MTA) is configured to relay SMTP content to this port, and aiosmtpd accepts the DATA payload on behalf of SimpleLogin.

### 2.2 Per-Message Entry — `MailHandler.handle_DATA`

The `MailHandler` class is defined at `email_handler.py:2288`. Its async method `handle_DATA(server, session, envelope: Envelope)` at `email_handler.py:2289` is the aiosmtpd callback invoked for every SMTP DATA command. The method:

1. Parses the raw bytes via `msg = email.message_from_bytes(envelope.original_content)` at `email_handler.py:2290`.
2. Calls the synchronous `ret = self._handle(envelope, msg)` at `email_handler.py:2292` to perform the business logic.
3. Wraps the whole call in exception handlers that return `E524` (wrong reverse-alias use), `E213` (unknown email ignored), or `E404` (unexpected error) depending on the exception class (`email_handler.py:2297–2332`).

### 2.3 Observability & Flask Context — `_handle`

`_handle(self, envelope, msg)` at `email_handler.py:2335` is decorated with `@newrelic.agent.background_task()` at `email_handler.py:2334` for APM instrumentation. It:

1. Assigns a fresh correlation id: `message_id = str(uuid.uuid4())` at `email_handler.py:2339` and calls `set_message_id(message_id)` at `email_handler.py:2340`, which makes every downstream `LOG` line tagged with this UUID for per-message correlation.
2. Opens a Flask application context via `with create_light_app().app_context():` at `email_handler.py:2352`. `create_light_app()` is defined at `server.py:127` and is a *minimal* Flask application wired ONLY for SQLAlchemy (`app.config["SQLALCHEMY_DATABASE_URI"] = DB_URI` at `server.py:129`). It is deliberately different from the web application factory `create_app()` at `server.py:139` — the SMTP handler does not need the web middleware or route blueprints, only the ORM session.
3. Inside that context, calls `return_status = handle(envelope, msg)` at `email_handler.py:2353`, which is the main dispatcher.

### 2.4 Main Dispatcher — `handle`

`handle(envelope: Envelope, msg: Message)` is defined at `email_handler.py:1945`. Its responsibilities:

1. **Sanitize the envelope**: `mail_from = sanitize_email(envelope.mail_from)` at `email_handler.py:1949` and `rcpt_tos = [sanitize_email(rcpt_to) for rcpt_to in envelope.rcpt_tos]` at `email_handler.py:1950`. `sanitize_email()` is at `app/utils.py:97` and strips whitespace, lowercases (unless `not_lower=True`), and removes the right-to-left mark `\u200f`.
2. **Sanitize the headers**: `sanitize_header(msg, headers.FROM/TO/CC/REPLY_TO/MESSAGE_ID)` at `email_handler.py:1974–1978`.
3. **Classify each recipient**: Iterate `for rcpt_index, rcpt_to in enumerate(rcpt_tos):` at `email_handler.py:2180`. For each one, at `email_handler.py:2195` check `is_reverse_alias(rcpt_to)`.
4. **Dispatch**: If classified as a reply, call `handle_reply(envelope, copy_msg, rcpt_to)` at `email_handler.py:2199`. Otherwise, fall through to the forward-phase branch.

### 2.5 Reply Classification — `is_reverse_alias`

`is_reverse_alias(address: str) -> bool` is defined at `app/email_utils.py:1156`. It returns `True` if **either**:

- The address already exists in the `Contact` table as a reverse-alias: `if Contact.get_by(reply_email=address): return True` at `app/email_utils.py:1158–1159`. This is the authoritative check — it handles modern reverse-aliases that have no fixed prefix.
- OR the address matches the legacy pattern: `return address.endswith(f"@{config.EMAIL_DOMAIN}") and (address.startswith("reply+") or address.startswith("ra+"))` at `app/email_utils.py:1161–1163`. This is the fallback for legacy addresses whose database rows may have been deleted but whose format is still recognizable.

### 2.6 Reply Handler — `handle_reply`

Once the dispatcher classifies a recipient as a reverse-alias, all subsequent reply-phase logic is delegated to `handle_reply(envelope, msg, rcpt_to)` at `email_handler.py:966`. This is the single function that implements every reply-specific behavior: domain gate, contact lookup, alias resolution, user resolution, DMARC check, mailbox authorization, EmailLog creation, header rewriting, DKIM signing, primary delivery, and multi-mailbox notification fan-out.

### 2.7 Return Path

`handle_reply` returns a `(bool, str)` tuple (delivered, SMTP status). `handle()` accumulates those tuples (one per `rcpt_to`) into `res` at `email_handler.py:2200` and reduces them at `email_handler.py:2214–2233`: if any recipient succeeded, return the first success status; otherwise return the first failure. `_handle()` wraps this (`email_handler.py:2357–2365`) — it downgrades 5xx statuses to `E216` when SPF fails on the envelope return-path (to prevent backscatter). The final status bubbles up through `handle_DATA()` to aiosmtpd and then to Postfix.

---

## Section 3 — How Is the Alias Resolved to a User?

The alias-to-user resolution inside `handle_reply()` is a 13-step deterministic walk beginning at `email_handler.py:966`. Every step has an exact line number.

### 3.1 Step-by-Step Walk

1. **`email_handler.py:972`** — `reply_email = rcpt_to` captures the reverse-alias address that the inbound message was addressed to.
2. **`email_handler.py:974`** — `reply_domain = get_email_domain_part(reply_email)` extracts the right-hand-side of the `@` in the reverse-alias address.
3. **`email_handler.py:977–981`** — Domain gate. `if not reply_email.endswith(EMAIL_DOMAIN):` (line 977) and if the domain is not found via `SLDomain.get_by(domain=reply_domain)` (line 978), the handler emits `LOG.w("Reply email {reply_email} has wrong domain")` and returns `(False, status.E501)` (line 981). This prevents impersonation of reverse-aliases on domains the system does not own.
4. **`email_handler.py:984`** — `reply_email = normalize_reply_email(reply_email)`. The function is defined at `app/email_validation.py:25`. Per `app/email_validation.py:27–28` it first normalizes non-ASCII via `convert_to_id()`, then replaces any character not in `_ALLOWED_CHARS` (defined at `app/email_validation.py:9` as `"abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789_-.+@"`) with `_`. This repairs legacy reverse-aliases that contain strange characters from past generation bugs.
5. **`email_handler.py:986`** — **The critical lookup**: `contact = Contact.get_by(reply_email=reply_email)`. This uses the indexed `Contact.reply_email` column declared at `app/models.py:1899` (`sa.Column(sa.String(512), nullable=False, index=True)`). Uniqueness is enforced by the 1000-iteration collision-avoidance loop inside `generate_reply_email()` at `app/email_utils.py:1136` (`for _ in range(1000):`) — it keeps trying random candidates until `available_sl_email(reply_email)` returns `True`.
6. **`email_handler.py:987–989`** — If no contact was found: `LOG.w("No contact with {reply_email} as reverse alias")` and return `(False, status.E502)`. This is the dominant "unknown reverse alias" rejection.
7. **`email_handler.py:990–992`** — If `not contact.user.is_active():` return `(False, status.E502)`. Deactivated or banned accounts cause reply delivery to fail.
8. **`email_handler.py:994`** — `alias = contact.alias` resolves the alias that the contact points to. This traverses `Contact.alias = orm.relationship(Alias, backref="contacts")` declared at `app/models.py:1907`. The underlying FK is `Contact.alias_id = sa.Column(sa.ForeignKey(Alias.id, ondelete="cascade"), nullable=False, index=True)` at `app/models.py:1881–1883`.
9. **`email_handler.py:995–996`** — `alias_address: str = contact.alias.email` and `alias_domain = get_email_domain_part(alias_address)` extract the alias address and its domain.
10. **`email_handler.py:1000–1002`** — Sanity check: `is_valid_alias_address_domain(alias.email)` (defined at `app/email_utils.py:557`). This confirms the alias domain is either an `SLDomain` or a verified `CustomDomain`. If neither, return `(False, status.E503)`. This protects against stale contacts that reference aliases on a custom domain that has since been removed.
11. **`email_handler.py:1004`** — `user = alias.user` resolves the owning user. This uses `Alias.user = orm.relationship(User, foreign_keys=[user_id])` at `app/models.py:1576`. The FK is `Alias.user_id = sa.Column(sa.ForeignKey(User.id, ondelete="cascade"), nullable=False, index=True)` at `app/models.py:1474–1476`.
12. **`email_handler.py:1007–1009`** — `if not user.can_send_or_receive():` return `(False, status.E504)`. This rejects disabled/frozen accounts.
13. **`email_handler.py:1012–1016`** — `ret = apply_dmarc_policy_for_reply_phase(alias, contact, envelope, msg)` (defined at `app/handler/dmarc.py:154`). If the rspamd-delivered DMARC verdict is `reject`/`quarantine`/`softfail`, the function returns `status.E215` (`app/handler/dmarc.py:194`) and the reply is dropped. Otherwise it returns `None` and execution continues.

After step 13, `handle_reply` moves on to **mailbox authorization** at `email_handler.py:1019` (`mailbox = get_mailbox_from_mail_from(mail_from, alias)`). That is covered in Section 5, step 17. Mailbox authorization is NOT part of alias-to-user resolution per se — it decides which of the alias's authorized senders is impersonating the alias for this specific reply.

### 3.2 Schema Foundation

The relational integrity that makes the above resolution deterministic:

| Field | Declaration | Semantics |
|-------|-------------|-----------|
| `Contact.reply_email` | `app/models.py:1899` — `sa.Column(sa.String(512), nullable=False, index=True)` | The indexed lookup key. Not declared as `unique=True` at the column level, but uniqueness is enforced by the 1000-iteration `generate_reply_email()` collision check at `app/email_utils.py:1136`. |
| `Contact.alias_id` | `app/models.py:1881–1883` — FK to `Alias.id`, `ON DELETE CASCADE`, `index=True` | Walk from Contact → Alias. Cascade ensures contacts disappear when the alias is deleted. |
| `Contact.user_id` | `app/models.py:1878–1880` — FK to `User.id`, `ON DELETE CASCADE`, `index=True` | Records the contact's owning user. Typically equals `alias.user_id`, but see Section 6 Candidate 5 for the transfer-alias divergence window. |
| `Alias.user_id` | `app/models.py:1474–1476` — FK to `User.id`, `ON DELETE CASCADE`, `index=True` | Walk from Alias → User. |
| `Alias.mailbox_id` | `app/models.py:1506–1508` — FK to `mailbox.id`, `ON DELETE CASCADE`, `index=True` | The **primary** mailbox for the alias. Used as the silent fallback when spoofing check is disabled. |
| `Alias._mailboxes` | `app/models.py:1512` — `orm.relationship("Mailbox", secondary="alias_mailbox", lazy="joined")` | Additional mailboxes via the junction table. The `_`-prefix signals this should not be read directly; use the `mailboxes` property. |
| `Alias.disable_email_spoofing_check` | `app/models.py:1528` — `sa.Column(sa.Boolean, ..., default=False, server_default="0")` | Per-alias opt-out of sender authorization. Enables the silent fallback at `email_handler.py:1025–1029`. |
| `Alias.user` | `app/models.py:1576` — `orm.relationship(User, foreign_keys=[user_id])` | The ORM accessor `alias.user` used at `email_handler.py:1004`. |
| `Alias.mailbox` | `app/models.py:1577` — `orm.relationship("Mailbox", lazy="joined")` | The primary mailbox. |
| `Alias.mailboxes` property | `app/models.py:1580` | Computed list of all authorized mailboxes for this alias. Returns `[self.mailbox] + dedup(self._mailboxes)` (`app/models.py:1581–1584`), filtered to `verified=True` (`app/models.py:1586`), sorted by email (`app/models.py:1587`). |
| `AliasMailbox` model | `app/models.py:2939` with `UniqueConstraint("alias_id", "mailbox_id", name="uq_alias_mailbox")` at `app/models.py:2942` | Junction table for the many-to-many. FK cascades at `app/models.py:2945–2950`. |

---

## Section 4 — What User ID Does the System Forward the Reply To?

**Critical clarification — the reply is NEVER forwarded to a SimpleLogin user's mailbox. It is forwarded to an *external* email address: the original third-party sender's email, stored in `contact.website_email`.** The "user_id" that appears in the audit trail is the `user_id` recorded on the resulting `EmailLog` row, used exclusively for dashboard attribution and quota accounting. The actual SMTP destination of the reply is always external.

### 4.1 The `EmailLog` Row

Immediately after mailbox authorization succeeds, the reply handler creates an `EmailLog` entry at `email_handler.py:1042–1050`:

```python
email_log = EmailLog.create(
    contact_id=contact.id,
    alias_id=contact.alias_id,
    is_reply=True,
    user_id=contact.user_id,          # ← FROM contact.user_id, NOT alias.user_id
    mailbox_id=mailbox.id,
    message_id=msg[headers.MESSAGE_ID],
    commit=True,
)
```

Per-field semantics:

- `contact_id = contact.id` — the `Contact` row matched by `Contact.get_by(reply_email=reply_email)` at `email_handler.py:986`.
- `alias_id = contact.alias_id` — read directly from the matched contact; equal to `alias.id` because `alias = contact.alias` at `email_handler.py:994`.
- `is_reply = True` — discriminator that distinguishes this log row from a forward-phase row.
- `user_id = contact.user_id` — **this is the answer to "what user ID does the system decide"**. It is read from the `Contact.user_id` column (FK declared at `app/models.py:1878–1880`), *not* from `alias.user_id`.
- `mailbox_id = mailbox.id` — the mailbox returned by `get_mailbox_from_mail_from(mail_from, alias)` at `email_handler.py:1019`, OR the `alias.mailbox` primary fallback at `email_handler.py:1029` when `alias.disable_email_spoofing_check == True`.
- `message_id = msg[headers.MESSAGE_ID]` — the original `Message-ID` header for threading/bounce correlation.

### 4.2 The `contact.user_id` vs `alias.user_id` Divergence

In normal operation, `contact.user_id == alias.user_id`. This invariant holds because `create_contact()` at `app/contact_utils.py:42` sets `user_id=alias.user_id` when the `Contact.create()` call is made at `app/contact_utils.py:93`.

However, the **`transfer_alias()`** procedure at `app/alias_utils.py:458` can (briefly) produce a divergence:

1. `Session.query(Contact).filter(Contact.alias_id == alias.id).update({"user_id": new_user.id})` at `app/alias_utils.py:464–466` — **Contact.user_id is updated FIRST.**
2. `Session.query(AliasMailbox).filter(AliasMailbox.alias_id == alias.id).delete()` at `app/alias_utils.py:477` — clear the old additional mailboxes.
3. `alias.mailbox_id = new_mailboxes.pop().id` at `app/alias_utils.py:480` — reassign the primary mailbox.
4. `for mb in new_mailboxes: AliasMailbox.create(alias_id=alias.id, mailbox_id=mb.id)` at `app/alias_utils.py:481–482` — create new junction rows.
5. `alias.user_id = new_user.id` at `app/alias_utils.py:506` — **Alias.user_id is updated LAST.**

All five steps run within the same SQLAlchemy session, so from the caller's perspective the transfer is atomic. Nevertheless, *within the session*, a reader walking the FK graph between steps 1 and 5 would observe `contact.user_id == new_user.id` while `alias.user_id == old_user.id`. If a reply arrives mid-transfer (a narrow race), the `EmailLog` row will credit the *new* user even though `alias.user` still returns the *old* user; the actual SMTP destination (`contact.website_email`) is unaffected.

### 4.3 The Final SMTP Destination

After header rewriting completes (see Section 5 steps 21–27), `handle_reply` calls `sl_sendmail()` at `email_handler.py:1224–1231`:

```python
sl_sendmail(
    generate_verp_email(VerpType.bounce_reply, email_log.id, alias_domain),
    contact.website_email,    # ← envelope_to — the EXTERNAL destination
    msg,
    envelope.mail_options,
    envelope.rcpt_options,
    is_forward=False,
)
```

- **`envelope_from`** is a **VERP-encoded address** — `generate_verp_email(VerpType.bounce_reply, email_log.id, alias_domain)` (defined at `app/email_utils.py:1438`) returns a VERP address keyed to the `email_log.id`. Bounces sent back to this address are parsed by SimpleLogin's bounce handler and correlated to the originating `EmailLog`.
- **`envelope_to`** is **`contact.website_email`** — the external third-party's real email address. The `Contact.website_email` column is declared at `app/models.py:1889`.
- **`is_forward=False`** flags the send as a reply for metrics and logging.
- `sl_sendmail()` itself is defined at `app/mail_sender.py:270` and performs the actual SMTP delivery (or, when `NOT_SEND_EMAIL=True` as in the test environment, stores the outbound message for later inspection via `MailSender.store_emails_test_decorator` at `app/mail_sender.py:111`).

### 4.4 Final Answer to the Question

**The `user_id` recorded on the resulting `EmailLog` is `contact.user_id`** (line `email_handler.py:1046`). This value is used solely for bookkeeping — to attribute the reply-send activity to the user in the dashboard and to enforce per-user quotas. **The reply message itself is delivered to `contact.website_email`, which is an external email address, not a SimpleLogin user mailbox.** The `alias.user_id` is dereferenced earlier at `email_handler.py:1004` only for the `user.can_send_or_receive()` authorization check; it does NOT appear on the `EmailLog` and does NOT influence the final SMTP destination.

Runtime verification confirmed this exactly. The in-process simulation produced:

```
[LOG] EmailLog.id=845 user_id=1 alias_id=1 contact_id=415 mailbox_id=1 is_reply=True
[PRIMARY] 1 msg(s) sent to contact.website_email (recipient@external.test)
```

The `EmailLog.user_id` matched `contact.user_id`, and the single primary `envelope_to` matched `contact.website_email`.

---

## Section 5 — The Actual Data Flow in Detail

This section traces every decision point from TCP-level receipt of the SMTP `DATA` command through final delivery. Each step cites the exact `file:line` location.

### 5.1 Numbered Runtime Walkthrough

1. **SMTP `DATA` received.** Postfix (or any upstream MTA) delivers the message to the SimpleLogin SMTP server listening on port 20381. `aiosmtpd.smtp.Controller` invokes the handler's `async def handle_DATA(self, server, session, envelope: Envelope)` at `email_handler.py:2289`. The raw bytes `envelope.original_content` are parsed into an `email.message.Message` via `msg = email.message_from_bytes(envelope.original_content)` at `email_handler.py:2290`.

2. **Flask app context opened.** `handle_DATA` immediately calls `self._handle(envelope, msg)` at `email_handler.py:2292`. `_handle` is defined at `email_handler.py:2335`, decorated with `@newrelic.agent.background_task()` at `email_handler.py:2334`. It creates a correlation UUID via `message_id = str(uuid.uuid4())` at `email_handler.py:2339` and publishes it through `set_message_id(message_id)` at `email_handler.py:2340`. It then enters `with create_light_app().app_context():` at `email_handler.py:2352`. `create_light_app()` is defined at `server.py:127` and is a minimal Flask factory that wires only SQLAlchemy (via `app.config["SQLALCHEMY_DATABASE_URI"] = DB_URI` at `server.py:129`) — deliberately distinct from the web factory `create_app()` at `server.py:139` to avoid importing web middleware into the SMTP process.

3. **Main dispatcher invoked.** `_handle` calls `return_status = handle(envelope, msg)` at `email_handler.py:2353`. `handle()` is defined at `email_handler.py:1945`.

4. **Envelope sanitization.** At `email_handler.py:1949`, `mail_from = sanitize_email(envelope.mail_from)`. `sanitize_email()` is defined at `app/utils.py:97`: it strips whitespace, strips the right-to-left mark `\u200f`, and lowercases. At `email_handler.py:1950`, every entry in `envelope.rcpt_tos` is sanitized the same way.

5. **Header sanitization.** At `email_handler.py:1974–1978`, `sanitize_header()` is applied to `FROM`, `TO`, `CC`, `REPLY_TO`, and `MESSAGE_ID` headers.

6. **Cross-phase guard.** `handle()` detects at `email_handler.py:1998–2025` whether the inbound message was itself sent *from* a reverse alias (a bounce-loop condition); it alerts the alias owner via transactional email but does not short-circuit regular reply routing.

7. **VERP / unsubscribe / bounce / complaint / rate-limit branches.** Lines `email_handler.py:2030–2173` route messages destined for `bounces+` or `unsubscribe+` addresses, VERP-encoded bounces, and transactional complaints. A normal user reply to a `reply+...@sl.local` address matches none of these and falls through to the per-recipient loop.

8. **Per-recipient loop.** `for rcpt_index, rcpt_to in enumerate(rcpt_tos):` at `email_handler.py:2180`. Each recipient is classified independently.

9. **Reverse-alias detection.** At `email_handler.py:2195`, `if is_reverse_alias(rcpt_to):` classifies the recipient as a reply target. `is_reverse_alias()` is defined at `app/email_utils.py:1156`. Its logic: first call `Contact.get_by(reply_email=address)` at `app/email_utils.py:1158`; if a contact exists, return `True` immediately (line 1159). Otherwise, return `True` iff `address.endswith(f"@{config.EMAIL_DOMAIN}")` AND (`address.startswith("reply+")` OR `address.startswith("ra+")`) at `app/email_utils.py:1161–1163`. The second branch supports legacy prefix-form reverse aliases that don't have a matching `Contact` row.

10. **Reply dispatch.** On match, `is_delivered, smtp_status = handle_reply(envelope, copy_msg, rcpt_to)` at `email_handler.py:2199`. A shallow-copied `msg` is passed so per-recipient mutations don't leak across iterations.

11. **Domain gate.** Inside `handle_reply()` at `email_handler.py:966`, the first check is at `email_handler.py:977–981`: `if not reply_email.endswith(EMAIL_DOMAIN): if not SLDomain.get_by(domain=reply_domain): LOG.w(...); return False, status.E501`. The reply address must end with the default `EMAIL_DOMAIN` OR be a registered custom SL domain.

12. **Address normalization.** `reply_email = normalize_reply_email(reply_email)` at `email_handler.py:984`. `normalize_reply_email()` is defined at `app/email_validation.py:25`. It first converts non-ASCII via `convert_to_id()` (lines 27–28), then replaces any character outside `_ALLOWED_CHARS` (at `app/email_validation.py:9`) with `_`. The `_ALLOWED_CHARS` set is `"abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789_-.+@"`.

13. **Contact lookup (THE KEY LOOKUP).** `contact = Contact.get_by(reply_email=reply_email)` at `email_handler.py:986`. This queries the `Contact` table using the indexed `reply_email` column at `app/models.py:1899`. If no row matches, `LOG.w("No contact with {reply_email} as reverse alias")` and return `(False, status.E502)` at `email_handler.py:987–989`. Uniqueness of `reply_email` is enforced empirically by the collision-avoidance loop inside `generate_reply_email()` — `for _ in range(1000):` at `app/email_utils.py:1136` — which keeps retrying until `available_sl_email(reply_email)` is true.

14. **User active-state check.** `if not contact.user.is_active():` at `email_handler.py:990` → return `(False, status.E502)` at line 992.

15. **Alias resolution.** `alias = contact.alias` at `email_handler.py:994`. This traverses the ORM relationship declared at `app/models.py:1907` (`alias = orm.relationship(Alias, backref="contacts")`). The FK is `Contact.alias_id` at `app/models.py:1881–1883` with `ON DELETE CASCADE`. Following the traversal, `alias_address: str = contact.alias.email` (line 995) and `alias_domain = get_email_domain_part(alias_address)` (line 996).

16. **Alias-domain sanity check.** `if not is_valid_alias_address_domain(alias.email):` at `email_handler.py:1000` → return `(False, status.E503)`. `is_valid_alias_address_domain()` is defined at `app/email_utils.py:557` and verifies the alias's domain is one that SimpleLogin serves.

17. **User resolution.** `user = alias.user` at `email_handler.py:1004`. This traverses `Alias.user = orm.relationship(User, foreign_keys=[user_id])` at `app/models.py:1576`. The FK is `Alias.user_id` at `app/models.py:1474–1476` with `ON DELETE CASCADE`.

18. **User send/receive authorization.** `mail_from = envelope.mail_from` is captured at `email_handler.py:1005`. Then at `email_handler.py:1007`, `if not user.can_send_or_receive():` → return `(False, status.E504)`.

19. **DMARC policy enforcement.** `apply_dmarc_policy_for_reply_phase(alias, contact, envelope, msg)` at `email_handler.py:1012–1014`. This function is defined at `app/handler/dmarc.py:154` and returns `status.E215` (at `app/handler/dmarc.py:194`) on DMARC quarantine/reject/soft_fail, otherwise `None`. If non-None, `handle_reply` returns `(False, dmarc_delivery_status)` at `email_handler.py:1015–1016`.

20. **Mailbox authorization.** `mailbox = get_mailbox_from_mail_from(mail_from, alias)` at `email_handler.py:1019`. `get_mailbox_from_mail_from()` is defined at `email_handler.py:1364`:
    - The inner helper `__check(email_address, alias)` at `email_handler.py:1369–1383` iterates `alias.mailboxes` (the property at `app/models.py:1580`). For each `mailbox`, it compares `mailbox.email == email_address` at `email_handler.py:1371`, and also iterates `mailbox.authorized_addresses` (line 1374) — the per-mailbox `AuthorizedAddress` list that allows additional sender addresses.
    - The outer return at `email_handler.py:1387` is `__check(mail_from, alias) or __check(canonicalize_email(mail_from), alias)`.
    - `canonicalize_email()` is defined at `app/utils.py:78`. It is a no-op unless the domain is `gmail.com`, `protonmail.com`, `proton.me`, or `pm.me` (guarded at `app/utils.py:84–85`). For those domains: strip everything after `+` in the local part at `app/utils.py:87–92`, remove all `.` from the local part at `app/utils.py:93`, then lowercase and strip at `app/utils.py:94`.
    - **Critical silent fallback:** if `mailbox is None` AND `alias.disable_email_spoofing_check == True`, then `mailbox = alias.mailbox` (the primary) at `email_handler.py:1029`. The surrounding block is `email_handler.py:1020–1029`. `Alias.disable_email_spoofing_check` is declared at `app/models.py:1528`.
    - Otherwise, if still unresolved, `handle_unknown_mailbox(...)` is invoked at `email_handler.py:1390` and `handle_reply` returns `(False, status.E214)` at `email_handler.py:1032–1034`.

21. **SPF verification.** At `email_handler.py:1036`, `if ENFORCE_SPF and mailbox.force_spf and not alias.disable_email_spoofing_check:` → call `spf_pass()` and return `(False, status.E201)` on failure.

22. **`EmailLog` row creation.** `EmailLog.create(contact_id=contact.id, alias_id=contact.alias_id, is_reply=True, user_id=contact.user_id, mailbox_id=mailbox.id, message_id=msg[headers.MESSAGE_ID], commit=True)` at `email_handler.py:1042–1050`. See Section 4 for detailed field semantics.

23. **Spam scan (optional).** If `ENABLE_SPAM_ASSASSIN` is on, the message is scored at `email_handler.py:1054–1094`. On spam, `handle_spam(...)` fires and `handle_reply` returns `(False, status.E506)`.

24. **Header stripping.** `delete_all_headers_except(msg, [FROM, TO, CC, SUBJECT, DATE, MESSAGE_ID, REFERENCES, IN_REPLY_TO, SL_QUEUE_ID, ... MIME_HEADERS])` at `email_handler.py:1096–1112`. Everything not in the whitelist is removed so that the outbound message doesn't leak mailbox metadata.

25. **Preserve original TO/CC.** `orig_to = msg[headers.TO]; orig_cc = msg[headers.CC]` at `email_handler.py:1114–1115`. These are kept in closure for later use by `notify_mailbox()` so other mailbox holders see what the user was actually replying to.

26. **Optional body replacement, PGP.**
    - If `user.replace_reverse_alias` is enabled, the body is scanned and reverse-alias strings are replaced with `contact.website_email`, `mailbox.email` with `alias.email`, and — when `ENABLE_ALL_REVERSE_ALIAS_REPLACEMENT` is set — all other reverse aliases of the same alias. This block runs at `email_handler.py:1119–1148`.
    - If `contact.pgp_finger_print` is set AND `user.is_premium()`, the message body is PGP-encrypted at `email_handler.py:1151–1164`. On encryption failure, the `EmailLog` is deleted and `handle_reply` returns `(False, status.E402)` so the sender's MTA will retry.

27. **FROM/TO/CC rewrite, MessageID rewrite, SL headers, DKIM, and delivery.**
    - `recipient_name = get_alias_recipient_name(alias)` at `email_handler.py:1168`. This helper at `app/alias_utils.py:580` returns an `AliasRecipientName` containing the `"name" <alias.email>` string to place in `From:`, following three rules: (i) if `alias.name` is set → `"alias.name" <alias.email>`; (ii) else if the custom domain has a display name → `"domain.name" <alias.email>`; (iii) else bare `alias.email`.
    - `add_or_replace_header(msg, headers.FROM, recipient_name.name)` at `email_handler.py:1172` rewrites the `From:` header so the reply appears to come from the alias, not the mailbox.
    - `replace_header_when_reply(msg, alias, headers.TO)` at `email_handler.py:1179` and `replace_header_when_reply(msg, alias, headers.CC)` at `email_handler.py:1181`. The function is at `email_handler.py:345`. Inside, `for _, reply_email in getaddresses(headers):` at `email_handler.py:358`; for each address, `if reply_email == alias.email: continue` at `email_handler.py:361–362`; then `contact = Contact.get_by(reply_email=reply_email)` at `email_handler.py:364` (**NOTE: no `alias_id` filter — this is a GLOBAL lookup**); on miss, raise `NonReverseAliasInReplyPhase(reply_email)` at `email_handler.py:372`; on hit, append `sl_formataddr((contact.name, contact.website_email))` at `email_handler.py:376`. If the exception is raised, the caller at `email_handler.py:1182–1200` deletes the `EmailLog`, notifies the mailbox owner, and returns `(True, status.E200)` without delivering.
    - `replace_original_message_id(alias, email_log, msg)` at `email_handler.py:1202` rewrites the `Message-ID` and `References` headers to use SL Message-IDs for proper bounce correlation and thread continuity.
    - `msg[headers.SL_DIRECTION] = "Reply"` at `email_handler.py:1209` and `msg[headers.SL_EMAIL_LOG_ID] = str(email_log.id)` at `email_handler.py:1210` add SL observability headers.
    - `if should_add_dkim_signature(alias_domain): add_dkim_signature(msg, alias_domain)` at `email_handler.py:1220–1221` signs the outbound message with the alias-domain DKIM key.
    - **Primary delivery** — `sl_sendmail(generate_verp_email(VerpType.bounce_reply, email_log.id, alias_domain), contact.website_email, msg, envelope.mail_options, envelope.rcpt_options, is_forward=False)` at `email_handler.py:1224–1231`. `generate_verp_email()` is at `app/email_utils.py:1438`. `sl_sendmail()` is at `app/mail_sender.py:270`.
    - **Multi-mailbox fan-out** — `other_mailboxes = [mb for mb in alias.mailboxes if mb.email != mailbox.email]` at `email_handler.py:1234`; then `for mb in other_mailboxes: notify_mailbox(alias, mailbox, mb, msg, orig_to, orig_cc, alias_domain)` at `email_handler.py:1235–1236`. `notify_mailbox()` is defined at `email_handler.py:1264`: it rewrites `From:` to `alias.email` at `email_handler.py:1278`, restores `To:` and `Cc:` from the captured originals at `email_handler.py:1280–1281`, and sends the copy via `sl_sendmail(generate_verp_email(VerpType.transactional, transaction.id, alias_domain), other_mb.email, notif)` at `email_handler.py:1289–1293`. A warning banner ("Don't forget to remove this section if you reply") is prepended to the body at `email_handler.py:1274–1275`.
    - Finally, `handle_reply` returns `(True, status.E200)`. The per-recipient tuple is collected in `res` at `email_handler.py:2200`, reduced at `email_handler.py:2214–2233`, and the final SMTP status string is returned up through `_handle` and `handle_DATA` to aiosmtpd, which writes it on the wire.

### 5.2 Mermaid Flowchart

```mermaid
flowchart TD
    A["SMTP DATA received<br/>MailHandler.handle_DATA<br/>email_handler.py:2289"] --> B["parse envelope.original_content<br/>email.message_from_bytes"]
    B --> C["_handle with Flask app context<br/>email_handler.py:2352"]
    C --> D["handle(envelope, msg)<br/>email_handler.py:1945"]
    D --> E["sanitize_email / sanitize_header<br/>app/utils.py:97"]
    E --> F{"is_reverse_alias(rcpt_to)?<br/>app/email_utils.py:1156"}
    F -->|No| Z1["handle_forward path<br/>(out of scope)"]
    F -->|Yes| G["handle_reply<br/>email_handler.py:966"]
    G --> H{"domain ends with EMAIL_DOMAIN<br/>or SLDomain match?<br/>:977-981"}
    H -->|No| R1["E501"]
    H -->|Yes| I["normalize_reply_email<br/>app/email_validation.py:25"]
    I --> J["Contact.get_by(reply_email=...)<br/>email_handler.py:986"]
    J -->|None| R2["E502"]
    J -->|Found| K["alias = contact.alias<br/>:994"]
    K --> L["user = alias.user<br/>:1004"]
    L --> M{"user.can_send_or_receive?<br/>:1007"}
    M -->|No| R3["E504"]
    M -->|Yes| N["apply_dmarc_policy_for_reply_phase<br/>app/handler/dmarc.py:154"]
    N -->|Reject| R4["E215"]
    N -->|Pass| O["get_mailbox_from_mail_from<br/>:1019 / :1364"]
    O --> P{"authorized mailbox?"}
    P -->|No + spoofing disabled| Q1["mailbox = alias.mailbox<br/>silent fallback :1029"]
    P -->|No + spoofing enforced| R5["E214 handle_unknown_mailbox"]
    P -->|Yes| Q2["mailbox = matched"]
    Q1 --> S["EmailLog.create<br/>user_id=contact.user_id<br/>:1042-1050"]
    Q2 --> S
    S --> T["strip headers, save orig_to/orig_cc<br/>:1096-1115"]
    T --> U["optional: body replacement, PGP<br/>:1119-1164"]
    U --> V["add_or_replace_header FROM = alias identity<br/>:1168-1172"]
    V --> W["replace_header_when_reply TO / CC<br/>global Contact lookup<br/>:345, :1179-1181"]
    W --> X["replace_original_message_id<br/>:1202"]
    X --> Y["DKIM sign<br/>:1220-1221"]
    Y --> AA["sl_sendmail to contact.website_email<br/>:1224-1231"]
    AA --> AB["notify_mailbox fan-out for other mailboxes<br/>:1234-1236, :1264"]
    AB --> AC["return E200"]
```

---

## Section 6 — Where Is the Most Likely Point of Incorrect Routing?

The user asked specifically: "when logs show the alias being correctly recognized, where could a reply still get routed to the wrong user?" This section analyzes every candidate failure mode against the code, grades each by risk, and concludes with a Final Verdict.

### Candidate 1 — `get_mailbox_from_mail_from()` canonicalization collision

**Location:** `email_handler.py:1364–1387`.

- The inner helper `__check(email_address, alias)` at `email_handler.py:1369–1383` iterates ONLY `alias.mailboxes` (line 1370), which by construction (the property at `app/models.py:1580`) returns mailboxes belonging to the already-resolved alias. Cross-alias or cross-user mailbox iteration is architecturally impossible here.
- The outer return at `email_handler.py:1387` is `__check(mail_from, alias) or __check(canonicalize_email(mail_from), alias)`.
- `canonicalize_email()` at `app/utils.py:78` is a **no-op** unless the domain is `gmail.com`, `protonmail.com`, `proton.me`, or `pm.me` (the guard at `app/utils.py:84–85`). For those domains, it strips `+...` suffixes at `app/utils.py:87–92` and removes all `.` characters at `app/utils.py:93`, then lowercases.
- Worst case: a user authenticates with `john.doe+promo@gmail.com` but the mailbox record stores `johndoe@gmail.com`; `canonicalize_email()` collapses both to `johndoe@gmail.com` and returns the *right* mailbox within this alias's mailbox set.
- Even in an adversarial scenario where two users coincidentally own mailboxes with Gmail-dot-equivalent addresses, they cannot simultaneously be attached to the *same alias* because that would have been caught at `Mailbox.create()` time. So a cross-user false match is architecturally impossible.

**Conclusion: LOW risk.** The mailbox returned is always on the resolved alias; the Contact → Alias → User chain is untouched.

### Candidate 2 — `disable_email_spoofing_check` silent fallback

**Location:** `email_handler.py:1020–1029`.

- When `get_mailbox_from_mail_from()` returns `None`, control enters the `if alias.disable_email_spoofing_check:` block at `email_handler.py:1021`. Inside, `LOG.w(...)` logs a warning and `mailbox = alias.mailbox` (the primary) is assigned at `email_handler.py:1029`.
- The field `Alias.disable_email_spoofing_check` is declared at `app/models.py:1528`. When `True`, it DELIBERATELY bypasses the mailbox authorization check per the feature's design intent.
- Side effect: the resulting `EmailLog.mailbox_id = alias.mailbox.id` even though `envelope.mail_from` does **not** match any authorized mailbox. The audit trail will attribute the reply to the primary mailbox's owner regardless of who actually sent it.
- The final `sl_sendmail` destination (`contact.website_email`) is still **correct** — the reply goes to the external third party as intended. What is wrong is the *attribution*, not the *routing*.

**Conclusion: HIGH risk for MISATTRIBUTION.** The user who owns the primary mailbox appears in logs and dashboards as responsible for replies they never authored. The reply itself still reaches the correct external recipient. This is the most dangerous mode *for forensics*: it looks like the correct user replied when in fact the pipeline silently let an unauthenticated sender through.

### Candidate 3 — `replace_header_when_reply()` GLOBAL contact lookup

**Location:** `email_handler.py:345–384`, called from `email_handler.py:1179` (TO) and `email_handler.py:1181` (CC).

- Inside the function at `email_handler.py:358`, `for _, reply_email in getaddresses(headers):` iterates every address in the `TO` or `CC` header.
- At `email_handler.py:361–362`, an address equal to `alias.email` is skipped (the "Reply-All to self" case).
- At `email_handler.py:364`, `contact = Contact.get_by(reply_email=reply_email)` — **THERE IS NO `alias_id` FILTER**. The lookup resolves globally across the entire `Contact` table. Because `reply_email` is globally unique by construction (the 1000-iteration collision loop in `generate_reply_email()` at `app/email_utils.py:1136` guarantees this), the lookup will succeed for any reverse-alias in the header, regardless of which user owns the underlying alias.
- At `email_handler.py:376`, `new_addrs.append(sl_formataddr((contact.name, contact.website_email)))` — the rewritten header carries the foreign contact's `website_email`, not the alias owner's.
- Real-world triggering scenario: user A uses "Reply All" on a thread that also contains a reverse-alias belonging to user B's alias. The rewritten TO/CC header on user A's outgoing reply will contain user B's contact's *real external email*. The primary `sl_sendmail` envelope_to is still user A's `contact.website_email` (correct), but the in-message TO/CC header now leaks a foreign contact's real address, and if anyone down the line hits "Reply All" on the delivered message, the reply will fan out to those foreign addresses.
- If a `TO`/`CC` address is not a reverse alias at all (e.g., some plain `@gmail.com` address), `NonReverseAliasInReplyPhase(reply_email)` is raised at `email_handler.py:372`. The caller at `email_handler.py:1182–1200` deletes the `EmailLog`, notifies the mailbox owner, and returns `(True, status.E200)` without sending — so unknown-domain addresses are safely rejected. The defect only applies to addresses that *are* reverse aliases (i.e., other SimpleLogin users' reverse aliases).

**Conclusion: MEDIUM risk.** The primary envelope recipient is correct, but the rewritten in-message `TO:` / `Cc:` headers cross identity boundaries when Reply-All is used on threads that span multiple SL users.

### Candidate 4 — Multi-mailbox `notify_mailbox()` copies

**Location:** `email_handler.py:1234–1236`; function at `email_handler.py:1264`.

- After the primary `sl_sendmail` to `contact.website_email`, the code at `email_handler.py:1234` computes `other_mailboxes = [mb for mb in alias.mailboxes if mb.email != mailbox.email]`.
- `alias.mailboxes` at `app/models.py:1580` returns `[alias.mailbox]` (the primary) concatenated with every `Mailbox` joined via the `alias_mailbox` junction table (`app/models.py:1581–1584`), deduped by `id`, filtered to `verified=True` (`app/models.py:1586`), and sorted by email (`app/models.py:1587`).
- At `email_handler.py:1235–1236`, for every mailbox in `other_mailboxes`, `notify_mailbox(alias, mailbox, mb, msg, orig_to, orig_cc, alias_domain)` is invoked. `notify_mailbox()` is at `email_handler.py:1264`.
- Inside `notify_mailbox()`:
    - `add_or_replace_header(notif, headers.FROM, alias.email)` at `email_handler.py:1278` — the notification's `From:` is rewritten to `alias.email`, *not* the real sending mailbox. A recipient of the notification therefore cannot tell *which* mailbox authored the reply — only that "the alias replied".
    - `add_or_replace_header(notif, headers.TO, orig_to)` at `email_handler.py:1280` and `add_or_replace_header(notif, headers.CC, orig_cc)` at `email_handler.py:1281` — the original TO/CC is restored so the holder of the other mailbox sees the full thread context.
    - A disclaimer banner ("Don't forget to remove this section if you reply") is prepended at `email_handler.py:1274–1275`.
    - `sl_sendmail(generate_verp_email(VerpType.transactional, transaction.id, alias_domain), other_mb.email, notif)` at `email_handler.py:1289–1293` dispatches the copy.
- From the perspective of any user who holds an *extra* mailbox on a shared alias, they will receive a copy of every reply — even though they did not author it, did not authorize it, and the `From:` header obscures who did. This is **by design** for shared/family aliases, but it is the single most likely trigger for a support report of "a reply was forwarded to the wrong user".
- Multi-mailbox behavior was confirmed in live runtime simulation: an alias with two verified mailboxes produced exactly two outbound messages — one primary `sl_sendmail` to the external contact, and one `notify_mailbox` copy to the secondary mailbox, both using VERP envelopes.

**Conclusion: HIGH risk for PERCEIVED mis-routing.** Every additional mailbox on the alias receives a copy that, to the receiving user, looks like an unexplained forwarded reply. The behavior is correct by design but is the number-one operational surprise in shared-alias setups.

### Candidate 5 — `Contact.user_id` vs `alias.user_id` divergence during `transfer_alias()`

**Location:** `app/alias_utils.py:458`.

- `transfer_alias()` performs five ordered steps inside a single SQLAlchemy session:
    1. Bulk-update `Contact.user_id = new_user.id` for every contact on the alias — `app/alias_utils.py:464–466`.
    2. Delete all `AliasMailbox` join rows for the alias — `app/alias_utils.py:477`.
    3. Assign the new primary via `alias.mailbox_id = new_mailboxes.pop().id` — `app/alias_utils.py:480`.
    4. Create new `AliasMailbox` rows for the remaining new mailboxes — `app/alias_utils.py:481–482`.
    5. Assign `alias.user_id = new_user.id` **LAST** — `app/alias_utils.py:506`.
- Within this session, reads see the pending values of each column. If a reply arrives during the session (an extremely narrow race window, requires concurrent writes), a reader would see `contact.user_id == new_user.id` while `alias.user_id == old_user.id`.
- Effect on the reply pipeline: `EmailLog.user_id = contact.user_id` at `email_handler.py:1046` would credit the *new* user, while `user = alias.user` at `email_handler.py:1004` would dereference the *old* user for the `can_send_or_receive()` check.
- The actual SMTP destination (`contact.website_email`) is unaffected.

**Conclusion: LOW risk** in practice because `transfer_alias()` runs within a single transaction. However, it does explain why some historical `EmailLog` rows may appear to belong to a user who no longer owns the alias.

### 6.6 Final Verdict

**The most likely source of "reply forwarded to the wrong user" when the logs show the alias being correctly recognized is the combination of Candidate 4 (multi-mailbox `notify_mailbox()` copies at `email_handler.py:1234–1236`) and Candidate 2 (silent fallback via `alias.disable_email_spoofing_check` at `email_handler.py:1025–1029`).** The reasoning:

- **Primary delivery is always correct.** When the logs show the alias being correctly recognized, the Contact → Alias → User chain is deterministic (`email_handler.py:986, 994, 1004`) and the primary `sl_sendmail` always goes to `contact.website_email` — the external third party. The *external-party routing* is not where the bug lies.
- **Additional mailboxes receive copies.** When `alias._mailboxes` contains any additional mailbox (the multi-user / family-alias / team-alias scenario), every OTHER verified mailbox on the alias receives a notification copy from `notify_mailbox()` at `email_handler.py:1264`. To a user of one of those extra mailboxes, the arriving message looks *exactly* like "a reply was forwarded to me from the wrong user's action". The `From:` header is rewritten to `alias.email` at `email_handler.py:1278`, deliberately obscuring the originating mailbox, which further reinforces the perception of mis-routing. This is the most common user complaint pattern.
- **Spoofing-check-disabled aliases silently credit the primary mailbox.** When `alias.disable_email_spoofing_check = True` (per the option at `app/models.py:1528`), any `mail_from` that fails the mailbox / `AuthorizedAddress` match silently falls back to `alias.mailbox` at `email_handler.py:1029`. The `EmailLog.mailbox_id` then records the primary mailbox's id, making it look as though the primary mailbox's owner authored the reply — in fact, they did not. The combination is pernicious: any sender can trigger a reply that gets credited to the primary mailbox's owner, and any other mailboxes on the alias each receive a `notify_mailbox` copy that looks like it came from the primary mailbox's owner.

**Secondary source: Candidate 3 (`replace_header_when_reply()` global lookup at `email_handler.py:345–384`).** When the user uses "Reply All" on a thread that already contains reverse-aliases from *other* aliases (including other users' aliases), each reverse-alias in `TO`/`CC` is resolved globally by `reply_email` (the lookup at `email_handler.py:364` has no `alias_id` filter) and rewritten to the foreign contact's `website_email` at `email_handler.py:376`. The primary envelope recipient remains correct, but the rewritten in-message `TO:`/`Cc:` headers now contain foreign external addresses. This is not mis-routing of the *current* reply — the current envelope still goes to the correct external party — but it does leak foreign external addresses into the outbound message's visible headers and can cause downstream "Reply All" actions to fan out to unintended external recipients.

Together, these three failure modes explain the full range of "the reply went to the wrong user" reports while remaining consistent with logs that show the alias being correctly recognized: the alias *is* recognized correctly, the Contact → Alias → User chain *is* correctly resolved, but the *fan-out* and *attribution* behavior at the tail of the pipeline produces unexpected delivery patterns that users interpret as mis-routing.

---

## Section 7 — Runtime Verification Performed

The claims in Sections 1–6 were validated against live runtime behavior, not purely static code reading. The following activities were performed inside the pre-provisioned Docker container `sl-setup` (image `andrewparkscaleai/coding-agent:simple-login__app__2cd6ee777f8c2d3531559588bcfb18627ffb5d2c` from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`).

### 7.1 Runtime Environment

- **Python**: 3.10.20 (system install inside the `sl-setup` container, matching the `Dockerfile` base `python:3.10`).
- **Virtual environment**: the container's default Python environment was used with dependencies already installed.
- **PostgreSQL**: version 15 running on `localhost:15432` inside the container (user `test`, password `test`, database `test`), matching `tests/test.env` `DB_URI=postgresql://test:test@localhost:15432/test`.
- **Redis**: running on `localhost:6379`, matching `tests/test.env` `MEM_STORE_URI=redis://localhost`.
- **Database schema**: Alembic migrations applied up to `32f25cbf12f6` via `alembic upgrade head`.
- **Feature flags**: `EMAIL_DOMAIN=sl.local`, `OTHER_ALIAS_DOMAINS=["d1.test", "d2.test", "sl.local"]`, `NOT_SEND_EMAIL=true`, `ENABLE_ALL_REVERSE_ALIAS_REPLACEMENT=true` (all from `tests/test.env`).

### 7.2 Dependency Build Notes

The project's `pyproject.toml` declares `pyre2 ^0.3.6`; in practice, the container installation uses a build-compatible `pyre2` variant exposing `DOTALL` and `split` shims required by `app/spamassassin_utils.py`. Similarly, `cbor2` is installed at a Python-3.10-compatible minor version. These are pre-installed in the container; no additional patching was required for this investigation.

### 7.3 Reply-Phase Test Run

The reply-related portion of the test suite was executed with:

```
docker exec sl-setup bash -c "cd /app && CONFIG=tests/test.env python -m pytest tests/test_email_handler.py -k reply -v --no-header --tb=short"
```

**Result: 6 passed, 17 deselected, 18 warnings in 1.68s.** The 6 passing tests were:

- `test_dmarc_reply_quarantine[QUARANTINE]`
- `test_dmarc_reply_quarantine[REJECT]`
- `test_dmarc_reply_quarantine[SOFTFAIL]`
- `test_email_sent_to_noreply`
- `test_replace_contacts_and_user_in_reply_phase` (at `tests/test_email_handler.py:274`)
- `test_send_email_from_non_canonical_address_on_reply` (at `tests/test_email_handler.py:315`)

These tests collectively exercise:
- DMARC reply-phase enforcement at `app/handler/dmarc.py:154` (via the parametrized `test_dmarc_reply_quarantine` family).
- The `replace_header_when_reply` path at `email_handler.py:345` (via `test_replace_contacts_and_user_in_reply_phase`).
- The `canonicalize_email()` fallback in `get_mailbox_from_mail_from()` at `email_handler.py:1387` / `app/utils.py:78` (via `test_send_email_from_non_canonical_address_on_reply`).

### 7.4 In-Process Simulation

A temporary probe script (`blitzy_adhoc_test_reply_simulation.py`, deleted after use) was created in the repository root to exercise the full reply pipeline in-process:

1. A synthetic `aiosmtpd.smtp.Envelope` was constructed with `envelope.mail_from = <authorized mailbox email>` and `envelope.rcpt_tos = [<contact.reply_email>]`.
2. A synthetic `email.message.EmailMessage()` was built with MIME headers (`From`, `To`, `Subject`, `Message-ID`).
3. The `flask_client` fixture's Flask app context was entered so SQLAlchemy `Session` operations worked.
4. Outbound SMTP was intercepted by `mail_sender.store_emails_test_decorator` (defined at `app/mail_sender.py:111`), which replaces real SMTP delivery with in-memory message storage for later inspection via `MailSender.get_stored_emails()`.
5. `email_handler.handle(envelope, msg)` was invoked and its return value captured.

**Primary-path observations**:
- `is_reverse_alias(rcpt_to)` → `True` (the `Contact.get_by(reply_email=...)` branch at `app/email_utils.py:1158` matched).
- `Contact.get_by(reply_email=reply_email)` at `email_handler.py:986` returned the expected contact.
- `contact.alias.user` dereferenced to `user_id=1` consistent with the seeded fixture.
- `get_mailbox_from_mail_from(mail_from, alias)` returned the authorized `mailbox.id=1`.
- `handle()` returned `250 Message accepted for delivery` (i.e., `status.E200` at `app/email/status.py:2`).
- Exactly **one** primary outbound message was captured:
    - `envelope_from = sl.lmysyibygq2syibsgi2tmnbwgzoq...@sl.local` (VERP-encoded per `generate_verp_email` at `app/email_utils.py:1438`).
    - `envelope_to = recipient@external.test` (equal to `contact.website_email`).
    - `From: simplelogin-newsletter.rudest216@sl.local` (equal to `alias.email`).
    - `To: recipient@external.test` (rewritten by `replace_header_when_reply` at `email_handler.py:345`).
    - `X-SL-Direction: Reply` and `X-SL-EmailLog-ID: 845` (added at `email_handler.py:1209–1210`).
- `EmailLog` row: `id=845, user_id=1, alias_id=1, contact_id=415, mailbox_id=1, is_reply=True`. **`EmailLog.user_id` equals `contact.user_id`**, confirming Section 4's claim.

**Multi-mailbox-path observations** (a second simulation added an extra verified mailbox to the alias via `AliasMailbox.create()` before replay):
- Exactly **two** outbound messages were captured.
- The first was the primary `sl_sendmail` to `contact.website_email` (as above).
- The second was a `notify_mailbox` copy addressed to the extra mailbox's email, with `From:` rewritten to `alias.email` (per `email_handler.py:1278`) and the original `To:` / `Cc:` restored (per `email_handler.py:1280–1281`). The envelope-from was a VERP transactional address (per `email_handler.py:1290`).

These observations directly validate the AAP's conclusions: the primary route is deterministic and goes to the external party, and the multi-mailbox fan-out is the operational behavior most likely to be perceived as mis-routing.

### 7.5 Cleanup

The temporary probe script was deleted immediately after the simulation completed. A `git status` at that point confirmed the only outstanding change was the new file `blitzy/documentation/app_2cd6ee777f8c.md`. The repository working tree is otherwise unmodified.

---

## Section 8 — Key Code References (Appendix)

| Item | File | Line(s) |
|------|------|---------|
| SMTP entry (`MailHandler`) | `email_handler.py` | 2288–2336 |
| aiosmtpd `Controller` + port 20381 | `email_handler.py` | 2381–2399 |
| Main dispatcher `handle()` | `email_handler.py` | 1945 |
| Per-recipient loop + reverse-alias detection | `email_handler.py` | 2180–2199 |
| `is_reverse_alias()` | `app/email_utils.py` | 1156 |
| `handle_reply()` | `email_handler.py` | 966 |
| Domain gate (EMAIL_DOMAIN / SLDomain) | `email_handler.py` | 977–981 |
| `normalize_reply_email()` invocation | `email_handler.py` | 984 |
| `Contact.get_by(reply_email=...)` | `email_handler.py` | 986 |
| `alias = contact.alias` | `email_handler.py` | 994 |
| `is_valid_alias_address_domain()` check | `email_handler.py` | 1000 |
| `user = alias.user` | `email_handler.py` | 1004 |
| `user.can_send_or_receive()` | `email_handler.py` | 1007 |
| DMARC reply policy | `app/handler/dmarc.py` | 154 |
| `get_mailbox_from_mail_from()` call | `email_handler.py` | 1019 |
| Silent fallback to `alias.mailbox` | `email_handler.py` | 1025–1029 |
| `handle_unknown_mailbox()` | `email_handler.py` | 1390 |
| `get_mailbox_from_mail_from()` definition | `email_handler.py` | 1364 |
| `canonicalize_email()` | `app/utils.py` | 78 |
| `sanitize_email()` | `app/utils.py` | 97 |
| EmailLog creation (reply) | `email_handler.py` | 1042–1050 |
| `replace_header_when_reply()` | `email_handler.py` | 345 |
| FROM header rewrite | `email_handler.py` | 1168–1172 |
| TO / CC header rewrite calls | `email_handler.py` | 1179–1181 |
| `replace_original_message_id()` | `email_handler.py` | 1202 |
| DKIM sign | `email_handler.py` | 1220–1221 |
| `sl_sendmail` primary delivery | `email_handler.py` | 1224–1231 |
| `notify_mailbox` loop | `email_handler.py` | 1234–1236 |
| `notify_mailbox()` function | `email_handler.py` | 1264 |
| `Alias.mailboxes` property | `app/models.py` | 1580 |
| `AliasMailbox` model | `app/models.py` | 2939 |
| `AliasMailbox.UniqueConstraint` | `app/models.py` | 2942 |
| `Contact.reply_email` column | `app/models.py` | 1899 |
| `Contact.alias_id` FK | `app/models.py` | 1881–1883 |
| `Contact.user_id` FK | `app/models.py` | 1878–1880 |
| `Contact.alias` relationship | `app/models.py` | 1907 |
| `Alias.user_id` FK | `app/models.py` | 1474–1476 |
| `Alias.mailbox_id` FK | `app/models.py` | 1506–1508 |
| `Alias._mailboxes` m2m | `app/models.py` | 1512 |
| `Alias.disable_email_spoofing_check` | `app/models.py` | 1528 |
| `Alias.user` relationship | `app/models.py` | 1576 |
| `generate_reply_email()` | `app/email_utils.py` | 1103 |
| `normalize_reply_email()` definition | `app/email_validation.py` | 25 |
| `create_contact()` | `app/contact_utils.py` | 42 |
| `transfer_alias()` | `app/alias_utils.py` | 458 |
| `get_alias_recipient_name()` | `app/alias_utils.py` | 580 |
| SMTP status codes (E200, E214, E215, E501–E506, …) | `app/email/status.py` | entire file |
| `sl_sendmail()` | `app/mail_sender.py` | 270 |
| `MailSender.store_emails_test_decorator` | `app/mail_sender.py` | 111 |
| `create_light_app()` | `server.py` | 127 |
| Reply test: `test_get_mailbox_from_mail_from` | `tests/test_email_handler.py` | 34 |
| Reply test: `test_replace_contacts_and_user_in_reply_phase` | `tests/test_email_handler.py` | 274 |
| Reply test: `test_send_email_from_non_canonical_address_on_reply` | `tests/test_email_handler.py` | 315 |
| Reply test: `test_send_email_from_non_canonical_matches_already_existing_user` | `tests/test_email_handler.py` | 344 |

