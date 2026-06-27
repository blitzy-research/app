# Investigation — Why Some Alias Replies Are Routed to the Wrong User

> **Repository:** SimpleLogin backend &nbsp;•&nbsp; **Branch:** `app_2cd6ee777f8c` &nbsp;•&nbsp; **HEAD:** `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`
>
> **Type:** Code-grounded investigation (analysis only — no source code was modified and no fix was implemented).
>
> Every factual claim about system behavior carries an exact `file:line` citation. Behavior that was **confirmed by running the code** is labelled **OBSERVED (executed)**; behavior derived from reading the code is labelled **INFERRED (static)**.

---

## Executive Summary

When a mailbox replies to an email that was forwarded through an alias, the reply is addressed to a **reverse-alias** address (`Contact.reply_email`). The backend must (1) **receive** the message, (2) **identify** which alias/contact the reply belongs to, and (3) **relay** it to the correct external recipient. This investigation traces that pipeline end-to-end on a running development stack and pinpoints where a wrong-user routing decision can originate.

**The single most important finding:** the user a reply is attributed to and delivered for is decided by **one** database lookup — `contact = Contact.get_by(reply_email=reply_email)` at `email_handler.py:L986`. Everything downstream (`alias = contact.alias` at `email_handler.py:L994`, `user = alias.user` at `email_handler.py:L1004`, `user_id=contact.user_id` on the `EmailLog` at `email_handler.py:L1042-L1050`, and physical delivery to `contact.website_email` at `email_handler.py:L1224-L1226`) is a **deterministic consequence of which `Contact` row that one lookup returns**. The chain is internally consistent; all the routing risk is concentrated in that single lookup.

**Why the logs can show "alias recognized" while delivery still goes to the wrong user:** recognition and resolution are **two separate, independent, unordered `.first()` lookups**. Recognition happens first inside `is_reverse_alias(rcpt_to)` (`email_handler.py:L2195` → `app/email_utils.py:L1158`) and immediately emits the **"Reply phase"** log (`email_handler.py:L2196-L2197`) — the exact line the user reads as "the alias was recognized." The *decisive* resolution runs **later**, at `email_handler.py:L986`. Because the recognition log fires **before** the decisive lookup, and because the two lookups are independent, the alias can look correctly recognized in the logs while the subsequently selected `Contact` (and therefore the user) is wrong.

**Most likely point of incorrect routing:** `Contact.get_by(reply_email=reply_email)` at `email_handler.py:L986`, enabled by four compounding structural conditions in the data model and resolution path — an unordered `.first()` (`app/models.py:L83-L84`), no database uniqueness on `Contact.reply_email` (`app/models.py:L1899`), a racy application-layer uniqueness check (`app/email_utils.py:L1103`, `app/models.py:L1425-L1432`, `app/contact_utils.py:L42`), and a normalization/case divergence between the lookup key and the stored value (`app/email_validation.py:L25-L38`). These are demonstrated against a live database in §7 and §8.

**Why this is a confidentiality/privacy risk, not just a delivery bug:** because both attribution (`EmailLog.create(... user_id=contact.user_id ...)` at `email_handler.py:L1042-L1050`) and delivery (`contact.website_email` at `email_handler.py:L1224-L1226`) are read off the mis-resolved `Contact`, a cross-user `reply_email` collision can record one user's reply under **another** user's account and/or relay its content to **another** user's external correspondent. The live trace in §8 even observed a collision leaking one user's mailbox address to an unrelated user on the rejection path. The full impact is detailed in §8 ("Confidentiality / privacy impact").

A remediation is **recommended** in §9 but, per the task constraints, was **not** implemented.

---

## 1. Question and Scope

The reported scenario, restated precisely:

> *When a user replies to an email that was forwarded through an alias, the backend should receive the incoming message, identify which alias it belongs to, and relay it back to the correct recipient. In local tests, some replies appear to be routed to the wrong user — even though the logs show the alias being recognized.*

The investigation answers six questions:

| # | Question | Section |
|---|----------|---------|
| O1 | Which component receives the inbound message? | §3 |
| O2 | How is the reply classified and the alias "recognized"? | §4 |
| O3 | How does the reverse-alias resolve to a contact, alias, and user? | §5 |
| O4 | Which user ID is the reply attributed to / forwarded to? | §6 |
| O5 | What is the actual data flow from a live execution trace? | §7 |
| O6 | What is the most likely point of incorrect routing, and why? | §8 |

**In scope:** the inbound **reply** path — the SMTP router `handle()`, the reverse-alias recognizer `is_reverse_alias()`, the reply handler `handle_reply()`, and the ORM resolution/attribution it performs.

**Out of scope (contextual contrast only):** the forward path `handle_forward()` (`email_handler.py:L536`), the bounce/complaint path `handle_bounce()` (`email_handler.py:L1851`), and the DMARC/SPF/PGP/spam subsystems. These are mentioned only where they intersect the reply path.

---

## 2. Method and Environment (O5 setup)

**Environment used for the live trace.** The analysis was produced against an actually-running SimpleLogin development stack, not by static reading alone:

- **Runtime:** Python **3.10.18** inside the provided container. The project declares `python = "^3.10"` at `pyproject.toml:L61`, the `Dockerfile` builds on `python:3.10` at `Dockerfile:L8`, and CI pins Python `3.10` at `.github/workflows/main.yml:L17` (lint job) and `.github/workflows/main.yml:L40` (test matrix). `CONTRIBUTING.md:L22-L25` states the project requires "Python 3.10 and poetry to manage dependencies" and "Postgres 13+", so the trace deliberately uses the container's Python 3.10 interpreter rather than the bare host interpreter. The ORM is **SQLAlchemy 1.3.24**, pinned at `pyproject.toml:L116` and verified in the running container (`python -c "import sqlalchemy; print(sqlalchemy.__version__)"` → `1.3.24`, OBSERVED) — this is the exact ORM version backing the `Contact`/`Alias`/`User`/`EmailLog` models and the `ModelMixin.get_by(reply_email=...)` lookup at the center of this analysis.
- **Datastore:** PostgreSQL **13** with all Alembic migrations applied (`alembic current` → head `32f25cbf12f6`), plus Redis **6**. This matches CI — `postgres:13` at `.github/workflows/main.yml:L47`, exposed on port `15432` at `.github/workflows/main.yml:L60`, and Redis `redis-version: 6` at `.github/workflows/main.yml:L92-L94` — and the `scripts/run-test.sh:L4-L16` harness, which provisions `postgres:13` on port `15432` at `scripts/run-test.sh:L7`, runs `alembic upgrade head` at `scripts/run-test.sh:L13`, then the test suite at `scripts/run-test.sh:L16`.
- **Configuration:** `CONFIG=tests/test.env`, where `NOT_SEND_EMAIL=true` (`tests/test.env:L7`) makes the send path log/print rather than transmit, `EMAIL_DOMAIN=sl.local` (`tests/test.env:L8`), the database is `DB_URI=postgresql://test:test@localhost:15432/test` (`tests/test.env:L17`), and the cache is `MEM_STORE_URI=redis://localhost` (`tests/test.env:L78`). This lets an inbound reply be simulated and traced safely without sending outbound mail.

**How the reply was simulated.** The trace mirrors the project's own canonical reply-simulation pattern in `tests/test_email_handler.py:L165-L188`: create a user, create a random alias, create a `Contact` with a `website_email` and a generated `reply_email`, build an `email.message.Message`, construct an `aiosmtpd` `Envelope` with `envelope.rcpt_tos = [contact.reply_email]`, and call `email_handler.handle(envelope, msg)`. The application context was bootstrapped exactly as `tests/conftest.py` does (`create_app()` at `tests/conftest.py:L23`, `add_sl_domains()` at `tests/conftest.py:L38`, `add_proton_partner()` at `tests/conftest.py:L39`). Every seeded row was created inside a single `connection.begin()` transaction (`tests/conftest.py:L61`) that was rolled back at the end (`tests/conftest.py:L75-L77`) — the same external-transaction pattern the test fixtures use — so every seeded row was removed when the transaction rolled back (the pre-trace baseline row counts equalled the post-rollback counts, with no residual marker rows; see Appendix C and Appendix D). PostgreSQL id sequences, however, are non-transactional: a `nextval` consumed by a seeded insert is *not* reclaimed when the transaction rolls back, so per-table id sequences can advance past the current `max(id)` even though no seeded row persists (this is expected PostgreSQL behavior and leaves no row-level residue — quantified in Appendix D). The full trace narrative and raw log output are in §7 and Appendix C.

**Reaching the final print-mode send (DKIM note).** In the running dev container used for this trace, the DKIM private key materialized at the configured path (`DKIM_PRIVATE_KEY_PATH=local_data/dkim.key`, `tests/test.env:L15`) was in a PEM encoding (`-----BEGIN PRIVATE KEY-----`, PKCS#8) that this stack's `dkimpy` cannot parse; the key committed at that path in the repository is itself a parseable `-----BEGIN RSA PRIVATE KEY-----` (PKCS#1) key (verified with `dkim.crypto.parse_pem_private_key`), so the unparseable encoding is a property of the container's runtime key material, **not** the committed source — a runtime/environment artifact, not a repository defect. With default configuration the reply path calls `add_dkim_signature(msg, alias_domain)` at its call site `email_handler.py:L1220-L1221`; that function — defined at `app/email_utils.py:L457-L480` — logs a "DKIM fail" warning per header set at `app/email_utils.py:L468` and then **raises `Exception("Cannot create DKIM signature")` at `app/email_utils.py:L480`** (confirmed by direct probe — Appendix C). That failure occurs **after** the routing decision is already made and committed — after `EmailLog.create(...)` (`email_handler.py:L1042-L1050`) and after the "send email from … to …" decision is logged (`email_handler.py:L1212-L1215`) — and only just **before** the final physical `sl_sendmail(...)` (`email_handler.py:L1224-L1226`); it is therefore immaterial to *which user* a reply is routed to. To let the trace exercise the final transport and **observe** the `NOT_SEND_EMAIL=true` print branch, DKIM signing was delegated to rspamd via the standard `RSPAMD_SIGN_DKIM` configuration toggle (`app/config.py:L482`), which makes `add_dkim_signature()` return early at `app/email_utils.py:L458-L461`. With that toggle the reply reaches `sl_sendmail(...)` and the print-mode branch `if config.NOT_SEND_EMAIL:` at `app/mail_sender.py:L130-L137`, which **logs** the final "send email with subject …" line instead of transmitting (OBSERVED — §7 and Appendix C). No source file was modified; `RSPAMD_SIGN_DKIM` is a runtime/environment setting only.

---

## 3. O1 — Which component receives the inbound message

Inbound SMTP is handled by an `aiosmtpd`-based server. The callback that receives each message is `MailHandler.handle_DATA` (`class MailHandler` at `email_handler.py:L2288`; `async def handle_DATA(self, server, session, envelope)` at `email_handler.py:L2289`). It hands the parsed envelope and message to the central router `handle(envelope, msg)` (`email_handler.py:L1945`).

`handle()` first sanitizes the addresses — `mail_from = sanitize_email(envelope.mail_from)` (`email_handler.py:L1949`) and `rcpt_tos = [sanitize_email(rcpt_to) for rcpt_to in envelope.rcpt_tos]` (`email_handler.py:L1950`) — then iterates the recipients with `for rcpt_index, rcpt_to in enumerate(rcpt_tos):` (`email_handler.py:L2180`). `sanitize_email` lowercases by default (`app/utils.py:L97`, `def sanitize_email(email_address, not_lower=False)`), so the recipient key entering the recognizer is already case-folded for the *lookup* — a detail that matters in §8.

**OBSERVED (executed).** In the trace, `handle()` received the simulated reply and logged the intake at `email_handler.py:L1980`:

```text
"/app/email_handler.py:1980" - handle() - ==>> Handle mail_from:user_a_tkcncjvw@mailbox.test,
  rcpt_tos:['external-recipient_at_example_com_mnrotdcjq@sl.local'], header_from:..., message_id:<traceA@mailbox.test>, ...
```

This confirms O1: the `aiosmtpd` `handle_DATA` callback (`email_handler.py:L2289`) is the component that receives the message, and it immediately delegates to the central `handle()` router (`email_handler.py:L1945`).

---

## 4. O2 — How the reply is classified and the alias "recognized"

For each recipient, the router decides **reply vs forward** with a single predicate: `if is_reverse_alias(rcpt_to):` (`email_handler.py:L2195`). When it returns true, the router logs the **"Reply phase"** line and dispatches to the reply handler:

- `LOG.d("Reply phase %s(%s) -> %s", mail_from, copy_msg[headers.FROM], rcpt_to)` — call at `email_handler.py:L2196`, format string at `email_handler.py:L2197`. **This is precisely the log the user reads as "the alias was recognized."**
- `is_delivered, smtp_status = handle_reply(envelope, copy_msg, rcpt_to)` — dispatch at `email_handler.py:L2199`.

The recognizer itself is `def is_reverse_alias(address)` (`app/email_utils.py:L1156`). Its **first** action is a database lookup: `if Contact.get_by(reply_email=address): return True` (`app/email_utils.py:L1158`); only if that misses does it fall back to a prefix/domain heuristic. This is the crucial structural fact for O6: **recognition already performs a `Contact.get_by(reply_email=...)` lookup**, and that lookup is *separate from* the decisive one performed later inside `handle_reply()` (§5).

**OBSERVED (executed).** The trace emitted the recognition log exactly as the user describes, immediately before any resolution:

```text
"/app/email_handler.py:2196" - handle() - Reply phase user_a_tkcncjvw@mailbox.test(user_a_tkcncjvw@mailbox.test)
  -> external-recipient_at_example_com_mnrotdcjq@sl.local
```

So O2 is answered: classification is `is_reverse_alias(rcpt_to)` (`email_handler.py:L2195`), the "recognized" log is `email_handler.py:L2196-L2197`, and the message is then dispatched to `handle_reply()` (`email_handler.py:L2199`).

---

## 5. O3 — How the reverse-alias resolves to a contact, alias, and user

All resolution happens inside `def handle_reply(envelope, msg, rcpt_to)` (`email_handler.py:L966`). The relevant steps, in order:

1. **Domain guard.** The reply address must end with `EMAIL_DOMAIN` (or a known SL reverse-alias domain), otherwise it returns `status.E501` (`email_handler.py:L976-L981`).
2. **Key normalization.** `reply_email = normalize_reply_email(reply_email)` (`email_handler.py:L984`). `normalize_reply_email` is defined at `app/email_validation.py:L25-L38`: it converts non-ASCII via `convert_to_id`, maps any character not in `_ALLOWED_CHARS` to `"_"`, and returns the result. **It does not lowercase.** (Contrast `app/utils.py:L78` `canonicalize_email` and `app/utils.py:L97` `sanitize_email`, both of which *do* lowercase by default.)
3. **The decisive lookup.** `contact = Contact.get_by(reply_email=reply_email)` (`email_handler.py:L986`). If it misses, the handler logs `"No contact with {reply_email} as reverse alias"` and returns `status.E502` (`email_handler.py:L987-L989`).
4. **Derive the alias and user from the matched contact.** `alias = contact.alias` (`email_handler.py:L994`) and `user = alias.user` (`email_handler.py:L1004`).
5. **Select the sending mailbox (anti-spoofing).** `mailbox = get_mailbox_from_mail_from(mail_from, alias)` (`email_handler.py:L1019`; defined at `email_handler.py:L1364`). If the envelope sender is not an authorized mailbox of the resolved alias, this returns `None`, and (absent `disable_email_spoofing_check`) the reply is handled as an unknown mailbox and rejected with `status.E214` (`email_handler.py:L1019-L1035`).

The critical observation for O6: **steps 4 and 5 read everything off the `Contact` chosen in step 3.** The identity of the user is *not* derived from who actually sent the reply; it is derived entirely from `reply_email → Contact`.

**OBSERVED (executed).** The trace resolved the reverse-alias to a specific contact/alias/user/mailbox, visible in the `EmailLog` creation log (`email_handler.py:L1051`):

```text
"/app/email_handler.py:1051" - handle_reply() - Create <EmailLog 298> for
  <Contact 178 external-recipient@example.com 840>, <User 479 Trace User A user_a_tkcncjvw@mailbox.test>,
  <Mailbox 561 user_a_tkcncjvw@mailbox.test>
```

Here `Contact 178` → `alias 840` → `User 479` → `Mailbox 561`, exactly the chain `email_handler.py:L986 → L994 → L1004 → L1019`.

---

## 6. O4 — Which user ID the reply is attributed to and forwarded to

**Attribution.** The reply is recorded with an `EmailLog` whose `user_id` is taken straight from the matched contact:

```python
email_log = EmailLog.create(
    contact_id=contact.id,
    alias_id=contact.alias_id,
    is_reply=True,
    user_id=contact.user_id,     # <- the attributed user
    mailbox_id=mailbox.id,
    message_id=msg[headers.MESSAGE_ID],
    commit=True,
)
```

That block is `email_handler.py:L1042-L1050`, with the confirmation log `LOG.d("Create %s for %s, %s, %s", ...)` at `email_handler.py:L1051`. `EmailLog` is defined at `app/models.py:L2060`.

**Delivery.** The reply is relayed **from the alias** to the contact's real address:

- Decision log: `LOG.d("send email from %s to %s, ...", alias.email, contact.website_email, ...)` — `email_handler.py:L1212-L1215` (`alias.email` is the FROM at `L1214`; `contact.website_email` is the TO at `L1215`).
- Physical send: `sl_sendmail(generate_verp_email(VerpType.bounce_reply, email_log.id, alias_domain), contact.website_email, msg, ...)` — `email_handler.py:L1224-L1226`. The return-path is a VERP `bounce_reply` address; the actual recipient is `contact.website_email`.

**Conclusion (O4):** the user a reply is attributed to (`EmailLog.user_id`) and the external address it is delivered to (`contact.website_email`) are **both** properties of the single `Contact` returned at `email_handler.py:L986`. There is no independent re-derivation of the user — `user_id=contact.user_id`.

**OBSERVED (executed).** After the call returned, the persisted `EmailLog` row was inspected directly from the database:

```text
[emaillog] id=298 is_reply=True user_id=479 alias_id=840 contact_id=178
[verdict A] attributed user_id=479 (== contactA.user_id=479); delivered_to(website_email)=external-recipient@example.com
```

and the delivery decision was logged as:

```text
"/app/email_handler.py:1212" - handle_reply() - send email from word_word197@sl.local
  to external-recipient@example.com, mail_options:[],rcpt_options:[]
```

The physical send then reached the `NOT_SEND_EMAIL=true` **print branch** (`app/mail_sender.py:L130-L137`), which prints the message instead of transmitting it, after which `handle()` returned success:

```text
"/app/app/mail_sender.py:131" - send() - send email with subject 'Re: hello from external',
  from 'word_word197@sl.local' to 'External A <external-recipient@example.com>'
# handle() -> 250 Message accepted for delivery
```

i.e. delivered **from the alias** (`word_word197@sl.local`) **to the contact's website_email** (`external-recipient@example.com`), attributed to `user_id=479` — the matched contact's user. Because the run delegated DKIM signing to rspamd via `RSPAMD_SIGN_DKIM` (early-return at `app/email_utils.py:L458-L461`; config flag at `app/config.py:L482`), execution proceeded past the signing step into the actual `sl_sendmail(...)` call (`email_handler.py:L1224-L1226`) and the `NOT_SEND_EMAIL` print branch — confirming the full receive → attribute → relay path end-to-end, not merely the pre-send decision. *(O4 — delivery, OBSERVED through the `NOT_SEND_EMAIL` print branch.)*

---

## 7. O5 — The actual data flow, from a live execution trace

This section narrates the **OBSERVED (executed)** run. A throwaway script (deleted afterwards — see Appendix D) seeded a user, alias, and contact, then called `email_handler.handle(envelope, msg)` with `envelope.rcpt_tos = [contact.reply_email]`, mirroring `tests/test_email_handler.py:L165-L188`.

Seed values (from the database):

```text
userA.id=479       userA.email=user_a_tkcncjvw@mailbox.test
aliasA.id=840      aliasA.email=word_word197@sl.local
contactA.id=178    user_id=479  alias_id=840
                   website_email=external-recipient@example.com
                   reply_email =external-recipient_at_example_com_mnrotdcjq@sl.local
```

Step-by-step, each step tied to the code that produced it:

1. **Receive** — `handle()` intakes the envelope and logs `==>> Handle mail_from:user_a_tkcncjvw@mailbox.test, rcpt_tos:['…mnrotdcjq@sl.local'], …` (`email_handler.py:L1980`, after sanitizing at `email_handler.py:L1949-L1950`). *(O1)*
2. **Recognize** — `is_reverse_alias(rcpt_to)` (`email_handler.py:L2195`) returns true and the router logs `Reply phase user_a_tkcncjvw@mailbox.test(…) -> external-recipient_at_example_com_mnrotdcjq@sl.local` (`email_handler.py:L2196`). *(O2 — the "alias recognized" log.)*
3. **Dispatch** — control passes to `handle_reply(envelope, copy_msg, rcpt_to)` (`email_handler.py:L2199`).
4. **Normalize + resolve** — `normalize_reply_email(...)` (`email_handler.py:L984`) returns the key unchanged, then `Contact.get_by(reply_email=...)` (`email_handler.py:L986`) selects `Contact 178`. *(O3 — the decisive lookup.)*
5. **DMARC gate** — `apply_dmarc_policy_for_reply_phase(...)` returns `None`: it logged `DMARC check disabled` (`app/handler/dmarc.py:L159`, because the plain message carried no spam headers) and returned `None` (`app/handler/dmarc.py:L160`). The caller's guard `if dmarc_delivery_status is not None:` (`email_handler.py:L1015-L1016`) is therefore not taken, so routing continues.
6. **Attribute** — `EmailLog.create(...)` logs `Create <EmailLog 298> for <Contact 178 …>, <User 479 …>, <Mailbox 561 …>` (`email_handler.py:L1051`); the persisted row is `user_id=479, alias_id=840, contact_id=178`. *(O4 — attribution.)*
7. **Relay** — the handler logs `send email from word_word197@sl.local to external-recipient@example.com` (`email_handler.py:L1212`) — FROM the alias, TO `contact.website_email`. With DKIM signing delegated to rspamd (`RSPAMD_SIGN_DKIM`; early-return at `app/email_utils.py:L458-L461`), execution proceeded into the physical `sl_sendmail(...)` at `email_handler.py:L1224-L1226` and reached the `NOT_SEND_EMAIL=true` **print branch**, which logged `send email with subject 'Re: hello from external', from 'word_word197@sl.local' to 'External A <external-recipient@example.com>'` (`app/mail_sender.py:L131`; branch `app/mail_sender.py:L130-L137`); `handle()` then returned `250 Message accepted for delivery`. *(O4 — delivery, OBSERVED through the print branch.)*

The sequence below summarizes the observed flow and marks the single decision point (step 4) where a wrong-user outcome originates:

```mermaid
sequenceDiagram
    participant SMTP as MailHandler.handle_DATA<br/>email_handler.py:L2289
    participant Router as handle()<br/>email_handler.py:L1945
    participant RevAlias as is_reverse_alias()<br/>app/email_utils.py:L1156-L1158
    participant Reply as handle_reply()<br/>email_handler.py:L966
    participant Norm as normalize_reply_email()<br/>app/email_validation.py:L25-L38
    participant DB as Contact.get_by(reply_email)<br/>email_handler.py:L986 (app/models.py:L83-L84)
    participant Log as EmailLog.create()<br/>email_handler.py:L1042-L1050
    participant Out as deliver to contact.website_email<br/>email_handler.py:L1212-L1226

    SMTP->>Router: envelope, msg
    Router->>RevAlias: is_reverse_alias(rcpt_to) [L2195]
    RevAlias-->>Router: true (1st unordered .first()) -> "Reply phase" log [L2196-L2197]
    Router->>Reply: handle_reply(envelope, copy_msg, rcpt_to) [L2199]
    Reply->>Norm: normalize_reply_email(rcpt_to) [L984]
    Norm-->>Reply: normalized reply_email (no lowercasing)
    Reply->>DB: DECISIVE get_by(reply_email).first() [L986] (2nd unordered .first())
    DB-->>Reply: Contact (arbitrary if duplicate) -> alias [L994], user [L1004]
    Reply->>Log: user_id = contact.user_id, is_reply=True [L1042-L1050]
    Reply->>Out: send FROM alias TO contact.website_email [L1212-L1226]
```

The diagram makes the crux visible: the **"Reply phase" recognition log fires at step "true → Reply phase log"** (`email_handler.py:L2196-L2197`), which is *before* the **decisive** `get_by(reply_email).first()` at `email_handler.py:L986`. The two lookups are independent, so recognition can succeed in the logs while the decisive lookup later selects a different — wrong — `Contact`.

---

## 8. O6 — The most likely point of incorrect routing (with rationale)

**Localization.** The single decision that selects the user is `contact = Contact.get_by(reply_email=reply_email)` at **`email_handler.py:L986`**. Because every downstream value (`alias`, `user`, `EmailLog.user_id`, `contact.website_email`) is read off that one `Contact`, a wrong result here — and only here — produces a wrong-user reply while leaving the rest of the chain internally consistent. This is the most likely origin of the reported behavior.

**Why this lookup can return the wrong row.** Four compounding, code-level conditions — all properties of the **data model and resolution path**, not logic bugs inside `handle_reply()`:

### Condition 1 — The lookup is an unordered `.first()`

`ModelMixin.get_by` is implemented as:

```python
def get_by(cls, **kw):                                  # app/models.py:L83
    return Session.query(cls).filter_by(**kw).first()   # app/models.py:L84
```

There is **no `order_by`**. If two or more `Contact` rows share the same `reply_email`, SQL is free to return *any* of them; the row chosen is arbitrary and can belong to a different alias and a different user. The recognizer `is_reverse_alias` (`app/email_utils.py:L1158`) uses the *same* unordered `get_by`, which is why recognition and resolution are two independent, individually-arbitrary lookups.

### Condition 2 — There is no database uniqueness on `reply_email`

`Contact.reply_email` is indexed but **not unique**:

```python
reply_email = sa.Column(sa.String(512), nullable=False, index=True)   # app/models.py:L1899
```

The **only** uniqueness constraint on `Contact` is on a *different* pair of columns:

```python
__table_args__ = (
    sa.UniqueConstraint("alias_id", "website_email", name="uq_contact"),   # app/models.py:L1874-L1876
)
```

So two contacts that belong to **different aliases (and therefore potentially different users)** can physically hold the **same** `reply_email` — the database will not stop them, because `uq_contact` keys on `(alias_id, website_email)`, not on `reply_email`.

### Condition 3 — Application-layer uniqueness is racy (TOCTOU) and not backed by a constraint

Uniqueness of `reply_email` is enforced only *before* insertion, in Python:

- `generate_reply_email(contact_email, alias)` (`app/email_utils.py:L1103`) loops over candidate addresses and tests each with `available_sl_email(...)`.
- `available_sl_email(email)` (`app/models.py:L1425-L1432`) returns `False` if an `Alias`, `Contact`, or `DeletedAlias` already uses the value — a pure read-then-decide check.
- `create_contact(...)` (`app/contact_utils.py:L42`) calls `generate_reply_email` (`app/contact_utils.py:L89`) and inserts via `Contact.create(..., reply_email=reply_email, ..., commit=True)` (`app/contact_utils.py:L92-L102`). Its only integrity recovery is `except IntegrityError: Session.rollback()` (`app/contact_utils.py:L113-L114`) followed by a refetch keyed on `(alias_id, website_email)` (`app/contact_utils.py:L118`).

This is a classic time-of-check/time-of-use gap: because no database constraint backs `reply_email`, two concurrent contact creations can both pass `available_sl_email` and both insert the same `reply_email`. And critically, the `IntegrityError` handler only ever catches the **`uq_contact`** violation — **a duplicate `reply_email` raises no `IntegrityError` at all and persists silently.**

### Condition 4 — Normalization / case divergence between the lookup key and the stored value

The lookup key is transformed by `normalize_reply_email` (`app/email_validation.py:L25-L38`) at `email_handler.py:L984` before the lookup. That function maps disallowed characters to `"_"` and converts non-ASCII via `convert_to_id`, but **does not lowercase**. Meanwhile the value stored at creation time is the raw generated address, and within `create_contact` the surrounding `website_email` is sanitized with `sanitize_email(email, not_lower=True)` (`app/contact_utils.py:L74`) — i.e. case-preserving. Because there is no normalized or case-insensitive uniqueness in the database, a normalized lookup key can match a *different* stored contact than intended, or fail to match a contact that is logically the same. (Contrast `app/utils.py:L78` `canonicalize_email` and `app/utils.py:L97` `sanitize_email`, which lowercase by default — the normalization used at storage/recognition time is not consistent with a single canonical form.)

### OBSERVED (executed) demonstration of the wrong-user mechanism

Against the live database, a second contact for a **different** user/alias was created carrying the **same** `reply_email` as the contact from §7:

```text
[check]  available_sl_email(dup) -> False        # the app-layer guard says "taken"...
[result] Contact.create with DUPLICATE reply_email SUCCEEDED, contactB.id=179 contactB.user_id=480
         -> NO IntegrityError raised             # ...but the DB has no constraint to enforce it
[state]  contacts sharing reply_email='external-recipient_at_example_com_mnrotdcjq@sl.local':
           contact.id=178 user_id=479 alias_id=840 website_email=external-recipient@example.com
           contact.id=179 user_id=480 alias_id=842 website_email=external-recipient-B@example.com
[resolve] Contact.get_by(reply_email).first() -> contact.id=178 user_id=479
[verdict] reply_email maps to 2 contacts across users [479, 480];
          resolver collapses to a SINGLE user_id=479. The other user (480) is shadowed.
```

This directly confirms Conditions 1–3: the duplicate inserted with **no `IntegrityError`** (no DB uniqueness), and `Contact.get_by(reply_email=...)` returned exactly **one** of the two rows via an unordered `.first()`. The *intended* recipient of the shadowed user's correspondence was `Contact 179` (user 480, `website_email=external-recipient-B@example.com`), but the resolver selected `Contact 178` (user 479) instead:

```text
[intended] shadowed user_id=480, intended contact.id=179, website=external-recipient-B@example.com
[resolved] Contact.get_by(reply_email).first() -> contact.id=178 user_id=479   # WRONG user / contact
```

**OBSERVED (executed) shadowed-user route.** A reply was then driven from the *shadowed* user's own mailbox (`user_b_tbaspabw@mailbox.test`, user 480) to the duplicated `reply_email`. The alias was **recognized** (the user's "alias recognized" log fired exactly as in the user's report), yet the decisive lookup resolved to the **other** user's contact/alias, so the anti-spoofing guard rejected the reply *under the wrong user's alias*:

```text
"/app/email_handler.py:2196" - handle()                 - Reply phase user_b_tbaspabw@mailbox.test(...) ->
    external-recipient_at_example_com_mnrotdcjq@sl.local            # RECOGNIZED — "alias recognized" log fires
"/app/email_handler.py:1393" - handle_unknown_mailbox() - Reply email can only be used by mailbox.
    Actual mail_from: user_b_tbaspabw@mailbox.test. ... reverse-alias ...mnrotdcjq@sl.local,
    <Alias 840 word_word197@sl.local> <User 479 Trace User A ...> <Contact 178 ...>   # processed under WRONG user 479
# handle() -> 250 SL E214 Unauthorized for using reverse alias        (no EmailLog created for this reply)
```

This is the wrong-user mechanism executed end-to-end: the shadowed user 480's legitimate reply was recognized, but `Contact.get_by(reply_email=...)` (`email_handler.py:L986`) returned `Contact 178` / alias 840 / user 479 (note the log lists the *other* user's `<Alias 840>`, `<User 479>`, `<Contact 178>`); because user 480's mailbox is not an authorized mailbox of the resolved alias 840, `get_mailbox_from_mail_from(mail_from, alias)` (`email_handler.py:L1019`) returned `None`, control fell into the `else` branch that calls `handle_unknown_mailbox` (`email_handler.py:L1032`; def at `email_handler.py:L1390`, log at `email_handler.py:L1393`), and the caller returned `status.E214` (`email_handler.py:L1034`, surfaced to the SMTP client as `250 SL E214 Unauthorized for using reverse alias`). The shadowed user's reply thus failed **under the other user's alias** — this is the wrong-user outcome **observed in execution**, not merely inferred.

### Confidentiality / privacy impact

This wrong-user resolution is not merely a delivery nuisance — it is a **confidentiality/privacy risk**, because both the *attribution* and the *delivery target* of a reply are derived from the mis-resolved `Contact`:

- **Mis-attribution.** The reply is logged against the wrong account: `EmailLog.create(... user_id=contact.user_id ...)` (`email_handler.py:L1042-L1050`) records `user_id` from the resolved contact, so one user's reply activity can be persisted under **another user's** identity.
- **Mis-delivery.** When resolution reaches the send step, the message is relayed to `contact.website_email` of the *resolved* contact (`email_handler.py:L1224-L1226`) — i.e. potentially to **another user's** external correspondent rather than the intended one. The content of a private reply could therefore be disclosed to a third party who happens to share the duplicated `reply_email`.
- **Observed information disclosure.** In the executed shadowed-user run above, the rejection did not stay private: `handle_unknown_mailbox` alerts the *resolved* alias's owner via `send_email_with_rate_control(user, ALERT_REVERSE_ALIAS_UNKNOWN_MAILBOX, user.email, f"Attempt to use your alias {alias.email} from {envelope.mail_from}", ...)` (`email_handler.py:L1408-L1412`) — the recipient is `user.email` (the resolved user, A) and the subject embeds `envelope.mail_from` (the shadowed sender, B). The trace observed `app/mail_sender.py:L131` emit `send email with subject 'Attempt to use your alias word_word197@sl.local from user_b_tbaspabw@mailbox.test' ... to user_a_tkcncjvw@mailbox.test` — i.e. the shadowed user **B's mailbox address was disclosed to user A**, who has no legitimate relationship to user B. Even on the *rejection* path, the collision leaked one user's identifier to another.

In short, a cross-user `reply_email` collision can attribute and/or deliver a reply under the wrong user/contact, and can leak one user's address to another — a genuine confidentiality concern, not just a routing inconvenience.

### Tying it back to the user's report

The user observes that "the logs show the alias being recognized" yet "some replies are routed to the wrong user." Both halves are explained by the same structural fact:

- **"Alias recognized"** is the `Reply phase` log at `email_handler.py:L2196-L2197`, emitted right after the **recognition** lookup `is_reverse_alias` → `Contact.get_by(reply_email=...)` (`app/email_utils.py:L1158`).
- **"Wrong user"** is decided later, by the **independent** decisive lookup `Contact.get_by(reply_email=...)` at `email_handler.py:L986`.

Because recognition fires first and the two unordered `.first()` lookups are independent, the alias can look correctly recognized in the logs while the decisive lookup selects a different `Contact` — and thus a different user — whenever a duplicate or normalization-divergent `reply_email` exists. That is the most likely point of incorrect routing.

---

## 9. Rationale and Recommendation (analysis only — not implemented)

**Is this a real defect or a misunderstanding of the pipeline?** The evidence indicates the wrong-user behavior is a **genuine structural defect in the resolution layer**, not a misreading of the logs. The reply pipeline's *intent* is correct (and matches SimpleLogin's documented reverse-alias design: a reply to the reverse-alias is relayed out from the alias to the contact's real address while the user's mailbox stays hidden). The defect is that the routing key — `Contact.reply_email` — is treated by the application as if it were globally unique and deterministically resolvable, while the schema and the resolver provide **neither** guarantee:

- the resolver uses an **unordered `.first()`** (`app/models.py:L83-L84`), so a duplicate yields an arbitrary winner;
- the column has **no `UNIQUE` constraint** (`app/models.py:L1899`); the only constraint is `uq_contact(alias_id, website_email)` (`app/models.py:L1874-L1876`);
- uniqueness is enforced only by a **racy, unbacked application check** (`app/email_utils.py:L1103`, `app/models.py:L1425-L1432`), and the one `IntegrityError` recovery path keys on `uq_contact`, never on `reply_email` (`app/contact_utils.py:L113-L118`);
- the lookup key normalization is **case- and form-divergent** from storage (`app/email_validation.py:L25-L38` vs `app/utils.py:L78`/`L97`).

The live trace in §8 demonstrated that a duplicate `reply_email` across two users **inserts without any `IntegrityError`** and that the resolver then **collapses both users onto one**, with attribution and delivery following whichever row `.first()` happens to return. Because attribution (`email_handler.py:L1042-L1050`) and delivery (`email_handler.py:L1224-L1226`) both follow that resolved contact, the defect carries a **confidentiality/privacy** dimension — a reply can be recorded under, or delivered on behalf of, the wrong user, and (as observed) a collision can disclose one user's mailbox address to another. That raises the severity of the defect from a delivery error to a cross-user data-exposure risk.

**Recommendation (for the maintainers to consider — deliberately NOT implemented here):**

1. **Make the routing key unique at the database level** — add a `UNIQUE` constraint (and matching migration) on `Contact.reply_email` so the duplicate condition that enables mis-resolution becomes impossible to persist, and so the existing check-then-insert in `create_contact` (`app/contact_utils.py:L42`) is backstopped by a real `IntegrityError` it can catch.
2. **Make the lookup deterministic and normalization-consistent** — store and look up `reply_email` in a single canonical (e.g. case-folded) form, and/or give `ModelMixin.get_by` a deterministic ordering for this resolution, so that recognition (`app/email_utils.py:L1158`) and resolution (`email_handler.py:L986`) cannot diverge.

These two changes together would remove the ambiguity at its source — the single lookup at `email_handler.py:L986`. **This document does not implement them**; per the task constraints it only identifies and explains the problem and records the recommendation.

---

## 10. Appendix

### A. Observed-vs-Inferred ledger

| Claim | Status | Evidence |
|-------|--------|----------|
| `handle_DATA` → `handle()` receives the message | OBSERVED | `==>> Handle …` at `email_handler.py:L1980` |
| `is_reverse_alias` true → "Reply phase" log (the "recognized" log) | OBSERVED | `Reply phase …` at `email_handler.py:L2196` |
| `Contact.get_by(reply_email)` resolves contact→alias→user | OBSERVED | `Create <EmailLog 298> for <Contact 178 …>, <User 479 …>` at `email_handler.py:L1051` |
| `EmailLog.user_id == contact.user_id`; delivery to `contact.website_email` | OBSERVED | `EmailLog 298 user_id=479`; `send email … to external-recipient@example.com` at `email_handler.py:L1212` |
| Duplicate `reply_email` (different users) persists with no `IntegrityError` | OBSERVED | `contactB.id=179 … NO IntegrityError raised` |
| `get_by(reply_email).first()` returns one arbitrary row, collapsing two users | OBSERVED | `resolve … contact.id=178 user_id=479`; 2 contacts across users `[479, 480]` |
| Shadowed user's reply is recognized, then fails anti-spoofing under the **other** user's alias (`E214`) | OBSERVED | shadowed user 480 → `Reply phase` at `email_handler.py:L2196`, then `handle_unknown_mailbox` log at `email_handler.py:L1393` lists `<Alias 840> <User 479> <Contact 178>`; caller returns `status.E214` at `email_handler.py:L1034` → `250 SL E214 …` (def `email_handler.py:L1390`) |
| Cross-user collision leaks one user's mailbox address to another (confidentiality) | OBSERVED | unauthorized-use alert `'Attempt to use your alias word_word197@sl.local from user_b_tbaspabw@mailbox.test' … to user_a_tkcncjvw@mailbox.test` at `app/mail_sender.py:L131` |
| DKIM signing (container's runtime key) raises *after* the routing decision; bypassed via `RSPAMD_SIGN_DKIM` to reach print-mode | OBSERVED | `Exception("Cannot create DKIM signature")` raised at `app/email_utils.py:L480` (function `app/email_utils.py:L457-L480`; called from `email_handler.py:L1220-L1221`), after `email_handler.py:L1042-L1050` and `L1212` |
| Final transport reaches the `NOT_SEND_EMAIL=true` print branch | OBSERVED | `send email with subject 'Re: hello from external' … to 'External A <external-recipient@example.com>'` at `app/mail_sender.py:L131` (branch `app/mail_sender.py:L130-L137`); `handle()` returned `250 Message accepted for delivery` |
| Runtime ORM is **SQLAlchemy 1.3.24** (the version backing every model lookup, incl. `ModelMixin.get_by`) | OBSERVED | `import sqlalchemy; sqlalchemy.__version__ == '1.3.24'` in the running container; pinned at `pyproject.toml:L116` |
| PostgreSQL id sequences are non-transactional (a rolled-back insert advances the sequence but persists no row) | OBSERVED | `users_id_seq.last_value = 489` vs `max(users.id) = 478` after rolled-back seeds; temp-sequence proof: a `nextval` taken inside a rolled-back txn is not reclaimed on the next `nextval` |

### B. Environment and commands

```bash
# Runtime: Python 3.10.18, SQLAlchemy 1.3.24, PostgreSQL 13, Redis 6
#   per pyproject.toml:L61 (python = "^3.10"), pyproject.toml:L116 (SQLAlchemy = "1.3.24"), Dockerfile:L8 (FROM python:3.10),
#   .github/workflows/main.yml:L17,L40 (Python 3.10), :L47 (postgres:13), :L60 (15432:5432), :L92-L94 (redis 6)
# Config: CONFIG=tests/test.env -> NOT_SEND_EMAIL=true (tests/test.env:L7), EMAIL_DOMAIN=sl.local (tests/test.env:L8),
#   DB_URI=postgresql://test:test@localhost:15432/test (tests/test.env:L17), MEM_STORE_URI=redis://localhost (tests/test.env:L78)

# Migrations already at head (provisioning matches scripts/run-test.sh:L7,L13,L16):
CONFIG=tests/test.env alembic current        # -> 32f25cbf12f6 (head)

# Run the (temporary) trace inside the Python 3.10 dev container, delegating DKIM signing to rspamd
# (RSPAMD_SIGN_DKIM; app/config.py:L482) so the final NOT_SEND_EMAIL print-mode send (app/mail_sender.py:L130-L137) is reached:
CONFIG=tests/test.env PYTHONPATH=<repo> RSPAMD_SIGN_DKIM=1 python <temporary_trace_script>.py
```

### C. Raw observed log lines (key markers, verbatim)

```text
# Part 1 — primary reply (user A's mailbox -> contact A's reply_email), DKIM delegated to rspamd:
"/app/email_handler.py:1980" - handle()      - ==>> Handle mail_from:user_a_tkcncjvw@mailbox.test,
    rcpt_tos:['external-recipient_at_example_com_mnrotdcjq@sl.local'], ... message_id:<traceA@mailbox.test> ...
"/app/email_handler.py:2196" - handle()      - Reply phase user_a_tkcncjvw@mailbox.test(...) ->
    external-recipient_at_example_com_mnrotdcjq@sl.local
"/app/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() - DMARC check disabled
"/app/email_handler.py:1051" - handle_reply() - Create <EmailLog 298> for
    <Contact 178 external-recipient@example.com 840>, <User 479 Trace User A user_a_tkcncjvw@mailbox.test>,
    <Mailbox 561 user_a_tkcncjvw@mailbox.test>
"/app/email_handler.py:1212" - handle_reply() - send email from word_word197@sl.local to
    external-recipient@example.com, mail_options:[],rcpt_options:[]
"/app/app/email_utils.py:459" - add_dkim_signature() - DKIM signature will be added by rspamd
"/app/app/mail_sender.py:131" - send()        - send email with subject 'Re: hello from external',
    from 'word_word197@sl.local' to 'External A <external-recipient@example.com>'
# handle() -> 250 Message accepted for delivery ;  EmailLog 298 is_reply=True user_id=479 alias_id=840 contact_id=178

# Part 2 — wrong-user demonstration (duplicate reply_email across users 479 and 480):
available_sl_email(dup) -> False
Contact.create with DUPLICATE reply_email SUCCEEDED, contactB.id=179 contactB.user_id=480 -> NO IntegrityError raised
contacts sharing reply_email:
    contact.id=178 user_id=479 alias_id=840 website_email=external-recipient@example.com
    contact.id=179 user_id=480 alias_id=842 website_email=external-recipient-B@example.com
Contact.get_by(reply_email).first() -> contact.id=178 user_id=479   # collapses users [479, 480] to one
[intended] shadowed user_id=480, intended contact.id=179, website=external-recipient-B@example.com

# Driving the reply from the SHADOWED user 480's mailbox -> recognized, then rejected under user 479's alias:
"/app/email_handler.py:2196" - handle()                 - Reply phase user_b_tbaspabw@mailbox.test(...) ->
    external-recipient_at_example_com_mnrotdcjq@sl.local
"/app/email_handler.py:1393" - handle_unknown_mailbox() - Reply email can only be used by mailbox.
    Actual mail_from: user_b_tbaspabw@mailbox.test. ... reverse-alias ...mnrotdcjq@sl.local,
    <Alias 840 word_word197@sl.local> <User 479 Trace User A ...> <Contact 178 ...>
# handle() -> 250 SL E214 Unauthorized for using reverse alias   (NO EmailLog created for the shadowed reply)
"/app/app/mail_sender.py:131" - send()        - send email with subject
    'Attempt to use your alias word_word197@sl.local from user_b_tbaspabw@mailbox.test'
    ... to user_a_tkcncjvw@mailbox.test     # confidentiality leak: user B's address disclosed to user A
```

### D. Temporary artifacts and source-tree cleanliness

The investigation used one temporary trace script and seeded rows in a disposable test database, and the **temporary trace execution left no repository artifact**: the trace script lived under `/tmp` (never inside the repository tree) and was deleted afterwards. The seeded `User`/`Alias`/`Contact`/`EmailLog`/`Mailbox` rows were created inside a single `connection.begin()` transaction and undone with `transaction.rollback()` (the same wrapper the project's own `tests/conftest.py:L61,L75-L77` uses), so **every seeded row was removed** when the transaction rolled back. Row counts confirmed equality across the rollback (`baseline == after_rollback -> True`) and a final marker check found no residual test rows. One nuance is stated precisely, because the rule is that code/runtime is the source of truth: PostgreSQL id sequences are **non-transactional**, so a `nextval` consumed by a rolled-back insert is **not** reclaimed and a per-table id sequence can advance beyond the current `max(id)` even though no seeded row persists. This was OBSERVED directly — e.g., after the rolled-back traces `users_id_seq.last_value = 489` while `max(users.id) = 478` (likewise `forward_email_id_seq` (contact) `188` vs `175`, and `gen_email_id_seq` (alias) `860` vs `838`); a zero-residue temporary sequence confirmed the mechanism (a `nextval` taken inside a transaction that is then rolled back is not given back on the next `nextval`). This is expected PostgreSQL behavior, not row-level residue. No file in the SimpleLogin source tree was created, modified, or deleted at any point.

The only change this task makes to the repository is the addition of this one document. Comparing the deliverable commit against the upstream SimpleLogin `HEAD` (`2cd6ee77`) shows exactly one added file and zero source changes:

```text
$ git diff --name-status 2cd6ee77 HEAD
A       blitzy/documentation/app_2cd6ee777f8c.md

$ git status --porcelain          # after commit: clean working tree (no pending changes)
                                  # (empty output)

$ git ls-files | grep -E '__pycache__|\.pytest_cache'
                                  # (empty: bytecode/pytest caches are git-ignored and never tracked)
```

Interpretation: the single tracked change introduced is this deliverable document; **zero** SimpleLogin source files were created, modified, or deleted (tracked-source changes = 0). Python bytecode caches (`__pycache__/`, `.pytest_cache/`) are listed in `.gitignore`, so they never appear as tracked or untracked changes and do not pollute `git status`; they are build artifacts of the interpreter, not part of the deliverable. The live trace's temporary script and seeded data left no residue (script deleted; DB transaction rolled back).

The sole artifact produced by this task is **this document**, `blitzy/documentation/app_2cd6ee777f8c.md`, whose filename matches the source branch name `app_2cd6ee777f8c`.

### E. Citation index

| Symbol / fact | Location |
|---------------|----------|
| `handle(envelope, msg)` router | `email_handler.py:L1945` |
| `sanitize_email(envelope.mail_from)` / `rcpt_tos` | `email_handler.py:L1949-L1950` |
| recipient loop | `email_handler.py:L2180` |
| `is_reverse_alias(rcpt_to)` (classifier) | `email_handler.py:L2195` |
| "Reply phase" log (the "recognized" log) | `email_handler.py:L2196-L2197` |
| dispatch to `handle_reply()` | `email_handler.py:L2199` |
| `MailHandler` / `handle_DATA` (SMTP ingress) | `email_handler.py:L2288-L2289` |
| `handle_reply(envelope, msg, rcpt_to)` | `email_handler.py:L966` |
| `normalize_reply_email(reply_email)` call | `email_handler.py:L984` |
| **decisive** `Contact.get_by(reply_email=...)` | `email_handler.py:L986` |
| no-contact → `E502` | `email_handler.py:L987-L989` |
| `alias = contact.alias` | `email_handler.py:L994` |
| `user = alias.user` | `email_handler.py:L1004` |
| `get_mailbox_from_mail_from(...)` (anti-spoof) | `email_handler.py:L1019` (def `email_handler.py:L1364`) |
| `EmailLog.create(... user_id=contact.user_id ...)` | `email_handler.py:L1042-L1050` |
| "Create … for …" log | `email_handler.py:L1051` |
| "send email from … to …" decision | `email_handler.py:L1212-L1215` |
| physical `sl_sendmail(... contact.website_email ...)` | `email_handler.py:L1224-L1226` |
| `ModelMixin.get_by` (unordered `.first()`) | `app/models.py:L83-L84` |
| `uq_contact(alias_id, website_email)` | `app/models.py:L1874-L1876` |
| `website_email` column | `app/models.py:L1889` |
| `reply_email` column (`index=True`, not unique) | `app/models.py:L1899` |
| `available_sl_email(email)` | `app/models.py:L1425-L1432` |
| `EmailLog` model | `app/models.py:L2060` |
| `is_reverse_alias` (def) + its `Contact.get_by` | `app/email_utils.py:L1156, L1158` |
| `generate_reply_email` (candidate loop) | `app/email_utils.py:L1103` |
| `normalize_reply_email` (no lowercasing) | `app/email_validation.py:L25-L38` |
| `create_contact` (check-then-insert) | `app/contact_utils.py:L42` |
| `sanitize_email(email, not_lower=True)` in create | `app/contact_utils.py:L74` |
| pre-check / `generate_reply_email` / `Contact.create` | `app/contact_utils.py:L85, L89, L92-L102` |
| `except IntegrityError` → rollback → refetch (`uq_contact` only) | `app/contact_utils.py:L113-L118` |
| `canonicalize_email` / `sanitize_email` (lowercase) | `app/utils.py:L78, L97` |
| reply-simulation harness | `tests/test_email_handler.py:L165-L188` |
| app-context bootstrap (`create_app`, `add_sl_domains`) | `tests/conftest.py:L23, L38-L39` |
| print-mode config (`NOT_SEND_EMAIL`, `EMAIL_DOMAIN`, `DB_URI`, `MEM_STORE_URI`) | `tests/test.env:L7-L8, L17, L78` |
| DKIM key path (`DKIM_PRIVATE_KEY_PATH`) | `tests/test.env:L15` |
| postgres + migrations + tests harness | `scripts/run-test.sh:L4-L16` (provision `:L7`, migrate `:L13`, test `:L16`) |
| CI runtime/service versions (Python 3.10, `postgres:13`, redis 6) | `.github/workflows/main.yml:L17, L40, L47, L60, L92-L94` |
| project requirements (Python 3.10, Postgres 13+) | `CONTRIBUTING.md:L22-L25` |
| SQLAlchemy ORM pin (`1.3.24`) | `pyproject.toml:L116` |
| `add_dkim_signature` (def + raise) / rspamd early-return | `app/email_utils.py:L457-L480` (raise `:L480`; rspamd `:L458-L461`); call site `email_handler.py:L1220-L1221` |
| `RSPAMD_SIGN_DKIM` config flag | `app/config.py:L482` |
| `NOT_SEND_EMAIL` print-mode send branch | `app/mail_sender.py:L130-L137` (log line `:L131`) |
| DMARC reply-phase check (disabled under test) | `app/handler/dmarc.py:L158-L160` |
| external-transaction rollback pattern (clean DB) | `tests/conftest.py:L61, L75-L77` |
| forward path (contrast) | `email_handler.py:L536` |
| bounce path (contrast) | `email_handler.py:L1851` |
| SMTP process entry point | `email_handler.py:L2381` |
