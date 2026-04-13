# Investigative Analysis: SimpleLogin Inbound Reply-Email Resolution Logic

## Table of Contents

- [1. Executive Summary](#1-executive-summary)
- [2. Forward-Phase: Reply-Email Generation Path](#2-forward-phase-reply-email-generation-path)
  - [2.1 Entry Point — `get_or_create_contact()`](#21-entry-point--get_or_create_contact)
  - [2.2 Core Creation — `create_contact()`](#22-core-creation--create_contact)
  - [2.3 Reply-Email Generation — `generate_reply_email()`](#23-reply-email-generation--generate_reply_email)
  - [2.4 Application-Level Uniqueness Check — `available_sl_email()`](#24-application-level-uniqueness-check--available_sl_email)
  - [2.5 Parallel Creation Path — `replace_header_when_forward()`](#25-parallel-creation-path--replace_header_when_forward)
  - [2.6 Contact ORM Override — `Contact.create()`](#26-contact-orm-override--contactcreate)
- [3. Reply-Phase: Contact Resolution Data Flow](#3-reply-phase-contact-resolution-data-flow)
  - [3.1 SMTP Entry Point](#31-smtp-entry-point)
  - [3.2 Routing Dispatcher — `handle()`](#32-routing-dispatcher--handle)
  - [3.3 Reverse-Alias Detection — `is_reverse_alias()`](#33-reverse-alias-detection--is_reverse_alias)
  - [3.4 Critical Resolution — `handle_reply()`](#34-critical-resolution--handle_reply)
- [4. Runtime Values at Each Resolution Step](#4-runtime-values-at-each-resolution-step)
  - [4.1 Happy-Path Runtime Values](#41-happy-path-runtime-values)
  - [4.2 Failure Scenarios](#42-failure-scenarios)
- [5. Database Schema and Constraint Analysis](#5-database-schema-and-constraint-analysis)
  - [5.1 Contact Model Definition](#51-contact-model-definition)
  - [5.2 Migration Evidence](#52-migration-evidence)
  - [5.3 Implication Summary](#53-implication-summary)
- [6. Race Condition and TOCTOU Vulnerability Analysis](#6-race-condition-and-toctou-vulnerability-analysis)
  - [6.1 The TOCTOU Window](#61-the-toctou-window)
  - [6.2 Session and Connection Architecture](#62-session-and-connection-architecture)
  - [6.3 App Context Lifecycle](#63-app-context-lifecycle)
  - [6.4 Collision Probability Assessment](#64-collision-probability-assessment)
  - [6.5 Missing Distributed Locking](#65-missing-distributed-locking)
- [7. Cross-Contact and Cross-User Mismatch Scenarios](#7-cross-contact-and-cross-user-mismatch-scenarios)
  - [7.1 Can the Same `reply_email` Resolve to Different Contacts Over Time?](#71-can-the-same-reply_email-resolve-to-different-contacts-over-time)
  - [7.2 Can Resolution Fail Temporarily?](#72-can-resolution-fail-temporarily)
  - [7.3 Can Resolution Succeed but Forward to a Different User?](#73-can-resolution-succeed-but-forward-to-a-different-user)
- [8. Behavioral Conclusions](#8-behavioral-conclusions)
- [9. Appendix: Code References](#9-appendix-code-references)

---

## 1. Executive Summary

This document presents a deep investigative analysis of SimpleLogin's inbound reply-email resolution logic — specifically how the system derives the reply email address from an inbound SMTP envelope, resolves it to an associated `Contact` record, and selects the forwarding destination (alias and user).

### Investigation Scope

The investigation traces two interrelated code paths:

1. **Forward phase (contact creation)**: When an email arrives for an alias, the system creates a `Contact` record with a generated `reply_email` — a reverse-alias address. This occurs in `get_or_create_contact()` (`email_handler.py` line 180) and `replace_header_when_forward()` (`email_handler.py` line 239), which both delegate to `generate_reply_email()` (`app/email_utils.py` line 1103).

2. **Reply phase (contact resolution)**: When a user replies to a reverse-alias, the system normalizes the `rcpt_to` address, looks up the `Contact` via `Contact.get_by(reply_email=...)`, traverses the contact → alias → user relationships, and delivers the message to the contact's `website_email`. This occurs in `handle_reply()` (`email_handler.py` line 966).

### Key Questions Investigated

| Question | Answer |
|----------|--------|
| Can the same `reply_email` resolve to different contacts over time? | **Yes, in theory** — if a TOCTOU race during `generate_reply_email()` produces duplicate `reply_email` values across two `Contact` records (see [Section 7.1](#71-can-the-same-reply_email-resolve-to-different-contacts-over-time)) |
| Can resolution fail temporarily (return `None`)? | **No, under normal operation** — once a `Contact` is committed, `Contact.get_by(reply_email=...)` will find it at READ COMMITTED isolation (see [Section 7.2](#72-can-resolution-fail-temporarily)) |
| Can resolution succeed but forward to a wrong user? | **Yes, if duplicate records exist** — though the mailbox authorization check at `get_mailbox_from_mail_from()` provides an imperfect safety net (see [Section 7.3](#73-can-resolution-succeed-but-forward-to-a-different-user)) |

### Critical Finding

The `Contact.reply_email` column (`app/models.py` line 1899) is defined with `index=True` but **not** `unique=True`. The migration `2021_071310_78403c7b8089_.py` (line 22) explicitly creates the index with `unique=False`. The only unique constraint on the `contact` table is `uq_contact` covering `(alias_id, website_email)` — **not** `reply_email`. This means the application-level uniqueness check in `available_sl_email()` (`app/models.py` line 1425) is the sole guard against duplicate `reply_email` values, and it is susceptible to a time-of-check-to-time-of-use (TOCTOU) race under concurrent multi-process operation.

---

## 2. Forward-Phase: Reply-Email Generation Path

This section documents the complete forward-phase creation chain that generates `reply_email` values stored on `Contact` records. Understanding this path is essential context for the reply phase, because the values generated here are what the reply phase later resolves.

### 2.1 Entry Point — `get_or_create_contact()`

**File**: `email_handler.py`, lines 180–211

The `get_or_create_contact()` function is the primary entry point for contact creation during the forward phase. It is called from `handle_forward()` when an inbound email arrives destined for an alias.

**Observed behavior**:
- Lines 184–187: Parses the contact name and email from the RFC 2047 `from_header` via `parse_full_address(from_header)`. If parsing fails, both default to empty strings.
- Lines 190–191: Truncates `contact_name` to `Contact.MAX_NAME_LENGTH` (512 characters, defined at `app/models.py` line 1868).
- Lines 193–201: If the parsed `contact_email` is not a valid email (checked via `is_valid_email()` from `app/email_validation.py` line 12), falls back to using the SMTP envelope `mail_from`.
- **Line 202**: Delegates to `contact_utils.create_contact()` with parameters including `email=contact_email`, `alias=alias`, `name=contact_name`, `mail_from=mail_from`, `allow_empty_email=True`, and `automatic_created=True`.
- Line 211: Returns `contact_result.contact` — the `Contact` object (newly created or existing).

**Rationale**: This function's role is to ensure that every sender who emails an alias has a corresponding `Contact` record. The contact's `reply_email` is what enables the alias owner to reply back to the original sender. The `allow_empty_email=True` parameter allows creating contacts even when the sender's email cannot be parsed, ensuring no emails are silently dropped.

### 2.2 Core Creation — `create_contact()`

**File**: `app/contact_utils.py`, lines 42–120

This is the core contact creation function. Each step is documented below with its implications for `reply_email` integrity.

**Step-by-step execution**:

1. **Lines 52–55**: Permission check — if the user cannot create contacts and this is not an automatic creation, returns `ContactCreateError.NotAllowed`.

2. **Lines 57–61**: Parses the email again via `parse_full_address(email)` to extract a clean email address and display name.

3. **Lines 63–72**: Name normalization — uses parsed name if no explicit name given, removes null bytes from name.

4. **Line 74**: Sanitizes the email via `sanitize_email(email, not_lower=True)` (`app/utils.py` line 97), which strips whitespace and replaces Unicode control characters.

5. **Lines 75–83**: If the sanitized email is not valid, either returns an error (when `allow_empty_email=False`) or allows creation with `email = ""`.

6. **Line 85**: **Existing contact check** — `Contact.get_by(alias_id=alias.id, website_email=email)`. This uses the fields covered by the `uq_contact` unique constraint. If a contact already exists for this alias+email pair, returns it (with possible name/mail_from update via `__update_contact_if_needed()` at lines 28–39).

7. **Line 89**: **Reply email generation** — `reply_email = generate_reply_email(email, alias)`. This is the critical step where the reverse-alias address is created. See [Section 2.3](#23-reply-email-generation--generate_reply_email) for the full algorithm.

8. **Lines 90–103**: Creates the new `Contact` via `Contact.create()` with `reply_email=reply_email, commit=True`. The `commit=True` parameter causes an immediate `Session.commit()` (see `Contact.create()` in `app/models.py` line 1956–1957).

9. **Lines 113–119**: **`IntegrityError` handler** — catches constraint violations from the INSERT+COMMIT.
   - **Critical finding**: This handler is designed to catch violations of the `uq_contact` constraint on `(alias_id, website_email)`. On `IntegrityError`, it rolls back the session and fetches the existing contact by `Contact.get_by(alias_id=alias.id, website_email=email)` at line 118.
   - **This handler does NOT detect or handle duplicate `reply_email` values**, because there is no unique constraint on `reply_email`. If two concurrent processes generate the same `reply_email`, both INSERT operations will succeed without raising `IntegrityError`, and two `Contact` records with identical `reply_email` values will exist in the database.

### 2.3 Reply-Email Generation — `generate_reply_email()`

**File**: `app/email_utils.py`, lines 1103–1153

This function generates a random reverse-alias email address and performs an application-level uniqueness check before returning it.

**Algorithm**:

1. **Line 1112**: Initializes `include_sender_in_reverse_alias = False`.

2. **Lines 1114–1117**: Reads the user preference via `alias.user.include_sender_in_reverse_alias`. If the user has explicitly set this option, uses their preference.

3. **Lines 1119–1127**: If including sender info, preprocesses the `contact_email`:
   - Converts to ASCII-safe form via `convert_to_id()` (`app/utils.py` line 50)
   - Sanitizes via `sanitize_email()`
   - Truncates to 45 characters
   - Replaces `@` with `_at_` and `.` with `_`
   - Converts to alphanumeric via `convert_to_alphanumeric()`

4. **Lines 1129–1133**: Selects the reply domain:
   - Default: `config.EMAIL_DOMAIN` (from environment variable, `app/config.py` line 92)
   - If the alias's domain is an `SLDomain` with `use_as_reverse_alias=True`, uses the alias domain instead

5. **Lines 1136–1151**: Loops up to 1000 times to find an available reply email:
   - **With sender info** (lines 1137–1142): Generates `f"{contact_email}_{random_string(random_length)}@{reply_domain}"` where `random_length = random.randint(5, 10)`
   - **Without sender info** (lines 1144–1148): Generates `f"{random_string(random_length)}@{reply_domain}"` where `random_length = random.randint(20, 50)`
   - **Line 1150**: Calls `available_sl_email(reply_email)` to check if the generated address is unused
   - **Line 1151**: Returns the first available reply email

6. **Line 1153**: If no available address is found after 1000 attempts, raises `Exception("Cannot generate reply email")`.

**Key observation about `random_string()`** (`app/utils.py` lines 41–47): Uses `secrets.choice()` from `string.ascii_lowercase` (26 lowercase letters). This means:
- Without sender: random component is 20–50 characters from a 26-letter alphabet → ~26^20 to 26^50 possible values
- With sender: random component is 5–10 characters → ~26^5 to 26^10 possible values, but prefixed with a sender-specific string

### 2.4 Application-Level Uniqueness Check — `available_sl_email()`

**File**: `app/models.py`, lines 1425–1432

```python
def available_sl_email(email: str) -> bool:
    if (
        Alias.get_by(email=email)
        or Contact.get_by(reply_email=email)
        or DeletedAlias.get_by(email=email)
    ):
        return False
    return True
```

This function checks three tables to determine if a given email address is "available":
- `Alias.get_by(email=email)` — checks the alias table
- `Contact.get_by(reply_email=email)` — checks for existing contacts with this reply email
- `DeletedAlias.get_by(email=email)` — checks deleted aliases

**Critical finding**: This is an **application-level SELECT check only**. There is no database-level guarantee that the `reply_email` remains available between the SELECT query and the subsequent INSERT+COMMIT in `Contact.create()`. In a multi-process deployment, another process can insert a `Contact` with the same `reply_email` after the `available_sl_email()` check returns `True` but before the calling process commits its own INSERT. Since there is no UNIQUE constraint on `reply_email`, both INSERTs will succeed.

### 2.5 Parallel Creation Path — `replace_header_when_forward()`

**File**: `email_handler.py`, lines 239–317

This function provides a **second code path** for creating contacts with generated `reply_email` values. It is called during the forward phase to rewrite CC and To headers.

**Key behavior** (lines 293–307):

```python
try:
    contact = Contact.create(
        user_id=alias.user_id,
        alias_id=alias.id,
        website_email=contact_email,
        name=contact_name,
        reply_email=generate_reply_email(contact_email, alias),
        is_cc=header.lower() == "cc",
        automatic_created=True,
    )
    Session.commit()
except IntegrityError:
    LOG.w("Contact %s %s already exist", alias, contact_email)
    Session.rollback()
    contact = Contact.get_by(alias_id=alias.id, website_email=contact_email)
```

- Line 299: Calls `generate_reply_email(contact_email, alias)` inline within the `Contact.create()` call — the same generation function with the same TOCTOU vulnerability as in `create_contact()`.
- Lines 304–307: The `IntegrityError` handler catches `uq_contact` violations only — the same limitation as `create_contact()`. Duplicate `reply_email` values will not trigger an `IntegrityError`.

This parallel path means there are at least two independent code locations where `reply_email` values are generated and stored, both subject to the same race condition.

### 2.6 Contact ORM Override — `Contact.create()`

**File**: `app/models.py`, lines 1938–1962

The `Contact` class overrides the base `ModelMixin.create()` method with additional validation:

- Lines 1939–1940: Extracts `commit` and `flush` flags from keyword arguments.
- Line 1942: Instantiates the new `Contact` object.
- Lines 1944–1946: Sanitizes `website_email` via `sanitize_email()`.
- **Lines 1948–1952**: Anti-reverse-alias check — verifies that the `website_email` is not itself a reverse alias by calling `Contact.get_by(reply_email=website_email)`. If a match is found, raises `CannotCreateContactForReverseAlias` (`app/errors.py` line 29). This prevents circular references where a reverse-alias address is used as a contact's "real" email.
- Line 1954: Adds the new contact to the session via `Session.add(new_contact)`.
- Lines 1956–1957: If `commit=True`, calls `Session.commit()` — this is where the INSERT is flushed to the database.

**Note**: There is no uniqueness validation on the `reply_email` value being stored. The anti-reverse-alias check at line 1950 only verifies the `website_email` field, not the `reply_email` field.

---

## 3. Reply-Phase: Contact Resolution Data Flow

This section traces the exact resolution sequence when an inbound email arrives with `rcpt_to` set to a reverse-alias address.

### 3.1 SMTP Entry Point

**`MailHandler.handle_DATA()`** — `email_handler.py` line 2289

The async SMTP handler receives the raw SMTP DATA and initiates processing:

- Line 2290: Parses the raw bytes into an `email.message.Message` object: `msg = email.message_from_bytes(envelope.original_content)`
- Line 2292: Calls `ret = self._handle(envelope, msg)` — note this is a **synchronous call** (no `await`), which blocks the aiosmtpd event loop for the duration of processing.

**`_handle()`** — `email_handler.py` line 2335

- Line 2334: Decorated with `@newrelic.agent.background_task()` for performance monitoring.
- Lines 2336–2350: Generates a UUID message ID and records custom metrics.
- **Line 2352**: Creates a Flask app context and calls the main handler:
  ```python
  with create_light_app().app_context():
      return_status = handle(envelope, msg)
  ```
  The `create_light_app()` function (`server.py` line 127) creates a minimal Flask app. Its teardown callback at line 133–134 calls `Session.remove()` to clean up the database session when the app context exits.

**Concurrency implication**: Because `_handle()` is called synchronously (line 2292), the aiosmtpd event loop is blocked during processing. This means that within a single `email_handler` process, email processing is serialized — only one email is processed at a time. However, in a multi-process deployment (multiple `email_handler` instances), each process runs its own event loop and database connection, enabling concurrent processing.

### 3.2 Routing Dispatcher — `handle()`

**File**: `email_handler.py`, lines 1945–2234

The `handle()` function is the central routing dispatcher. Key steps relevant to reply resolution:

1. **Lines 1948–1952**: Sanitizes `mail_from` and `rcpt_tos` via `sanitize_email()`.

2. **Lines 1996–2011**: **Pre-check for reverse-alias abuse** — detects if the SMTP `mail_from` or the email's FROM header is itself a reverse alias:
   - Line 1998: `contact = Contact.get_by(reply_email=mail_from)` — checks if the sender address is a reverse alias
   - Lines 2003–2011: Parses the FROM header and checks `Contact.get_by(reply_email=from_header_address)` — same check for the header-level sender
   - If either matches, sets `email_sent_from_reverse_alias = True`, which triggers abuse handling (not the primary resolution path)

3. **Lines 2165–2173**: **Out-of-office handling** — when `rcpt_to` is a reverse alias and `mail_from` is `"<>"` (null sender):
   - Line 2166: `is_reverse_alias(rcpt_tos[0])` — calls `Contact.get_by(reply_email=rcpt_tos[0])` (see [Section 3.3](#33-reverse-alias-detection--is_reverse_alias))
   - Line 2167: `contact = Contact.get_by(reply_email=rcpt_tos[0])` — performs the lookup again
   - Returns `E206` ("Out of office") without delivering the message

4. **Line 2195**: **Reply phase dispatch** — the critical routing decision:
   ```python
   if is_reverse_alias(rcpt_to):
   ```
   If the recipient address is identified as a reverse alias, calls `handle_reply(envelope, copy_msg, rcpt_to)` at line 2199.

### 3.3 Reverse-Alias Detection — `is_reverse_alias()`

**File**: `app/email_utils.py`, lines 1156–1163

```python
def is_reverse_alias(address: str) -> bool:
    # to take into account the new reverse-alias that doesn't start with "ra+"
    if Contact.get_by(reply_email=address):
        return True

    return address.endswith(f"@{config.EMAIL_DOMAIN}") and (
        address.startswith("reply+") or address.startswith("ra+")
    )
```

- **Line 1158**: Performs `Contact.get_by(reply_email=address)` — if any contact has this address as its `reply_email`, returns `True`.
- **Lines 1161–1162**: Fallback for legacy reverse aliases that start with `reply+` or `ra+` prefix.

**Important observation**: This function performs the **same** `Contact.get_by(reply_email=...)` lookup that `handle_reply()` will perform again at line 986. This means the contact lookup happens **twice** for every reply email: once in `is_reverse_alias()` (for routing) and once in `handle_reply()` (for processing). Both calls use `ModelMixin.get_by()` which calls `.first()`, and both are subject to the same non-determinism risk if duplicates exist.

### 3.4 Critical Resolution — `handle_reply()`

**File**: `email_handler.py`, lines 966–1261

This is the core resolution function. Each step is documented with the exact runtime values it produces.

**Step 1: Raw assignment** (line 972):
```python
reply_email = rcpt_to
```
The raw reverse-alias address from the SMTP envelope is assigned directly. No transformation yet.

**Step 2: Domain validation** (lines 974–981):
```python
reply_domain = get_email_domain_part(reply_email)
if not reply_email.endswith(EMAIL_DOMAIN):
    sl_domain: SLDomain = SLDomain.get_by(domain=reply_domain)
    if sl_domain is None:
        LOG.w(f"Reply email {reply_email} has wrong domain")
        return False, status.E501
```
Verifies the reply email's domain is either the primary `EMAIL_DOMAIN` or a registered `SLDomain`. Returns `E501` ("550 SL E501") if the domain is unrecognized.

**Step 3: Normalization** (line 984):
```python
reply_email = normalize_reply_email(reply_email)
```
Calls `normalize_reply_email()` from `app/email_validation.py` (lines 25–38):
- If the reply email contains non-ASCII characters, converts to ASCII via `convert_to_id()` (`app/utils.py` line 50)
- Replaces any character not in `_ALLOWED_CHARS` (`abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789_-.+@`) with `_`

**Step 4: THE CRITICAL LOOKUP** (line 986):
```python
contact = Contact.get_by(reply_email=reply_email)
```
This is the single most important line in the reply resolution chain. It calls `ModelMixin.get_by()` (`app/models.py` lines 82–84):
```python
@classmethod
def get_by(cls, **kw):
    return Session.query(cls).filter_by(**kw).first()
```

The `.first()` call returns the first row matching the filter, or `None` if no match exists.

**Critical risk**: If multiple `Contact` rows share the same `reply_email`, `.first()` returns whichever row the database engine returns first. In PostgreSQL, this is typically (but not guaranteed to be) the row with the lowest primary key. The SQL standard does not define the order for `LIMIT 1` without an explicit `ORDER BY`. This means:
- If duplicates exist, the returned `Contact` is **non-deterministic**
- The returned `Contact` may belong to a different alias and different user than the one who generated the reverse alias
- Lines 987–989: If `contact` is `None`, returns `E502` ("550 SL E502 Email not exist")

**Step 5: User activity check** (lines 990–992):
```python
if not contact.user.is_active():
    LOG.w(f"User {contact.user} has been soft deleted")
    return False, status.E502
```
Verifies the resolved user has not been soft-deleted.

**Step 6: Alias derivation** (line 994):
```python
alias = contact.alias
alias_address: str = contact.alias.email
```
Traverses the SQLAlchemy relationship from `Contact` to `Alias` (defined at `app/models.py` line 1907: `alias = orm.relationship(Alias, backref="contacts")`).

**Step 7: User derivation** (line 1004):
```python
user = alias.user
```
Traverses from `Alias` to `User`. This is the user the reply will be attributed to.

**Step 8: Mailbox authorization** (line 1019):
```python
mailbox = get_mailbox_from_mail_from(mail_from, alias)
```
Calls `get_mailbox_from_mail_from()` (`email_handler.py` lines 1364–1387) which checks if the SMTP `mail_from` matches any of the alias's authorized mailboxes (or their authorized addresses).

If no matching mailbox is found:
- **Line 1021**: If `alias.disable_email_spoofing_check` is `True`, falls back to `alias.mailbox` (the default mailbox) at line 1029
- **Line 1030–1034**: If `disable_email_spoofing_check` is `False`, calls `handle_unknown_mailbox()` (line 1390) which sends an alert to the user, then returns `E214` ("250 SL E214 Unauthorized for using reverse alias")

**Step 9: DMARC enforcement** (lines 1012–1016):
```python
dmarc_delivery_status = apply_dmarc_policy_for_reply_phase(alias, contact, envelope, msg)
if dmarc_delivery_status is not None:
    return False, dmarc_delivery_status
```
Applies DMARC policy checks before delivery.

**Step 10: Header rewriting** (lines 1168–1181):
- Line 1172: Sets the FROM header to the alias's recipient name
- Line 1179: `replace_header_when_reply(msg, alias, headers.TO)` — rewrites TO header
- Line 1181: `replace_header_when_reply(msg, alias, headers.CC)` — rewrites CC header

**Step 11: `replace_header_when_reply()`** (`email_handler.py` lines 345–384):
- **Line 364**: For each email in TO/CC headers, calls `Contact.get_by(reply_email=reply_email)` — yet another lookup by `reply_email`, subject to the same non-determinism.
- Line 372: If no contact found for a reply email in the header, raises `NonReverseAliasInReplyPhase` (`app/errors.py` line 36).

**Step 12: Outbound delivery** (lines 1224–1231):
```python
sl_sendmail(
    generate_verp_email(VerpType.bounce_reply, email_log.id, alias_domain),
    contact.website_email,
    msg,
    envelope.mail_options,
    envelope.rcpt_options,
    is_forward=False,
)
```
Sends the reply to `contact.website_email` with a VERP return path for bounce tracking.

---

## 4. Runtime Values at Each Resolution Step

### 4.1 Happy-Path Runtime Values

The following table shows concrete runtime values for a typical reply scenario where User Alice (alice@gmail.com) replies to a contact John (john@example.com) through her alias (myalias@simplelogin.co):

| Step | Variable | Example Runtime Value | Source Code Location |
|------|----------|----------------------|---------------------|
| 1 | `rcpt_to` (raw envelope) | `john_doe_at_example_com_abc123def@simplelogin.co` | SMTP envelope, passed to `handle_reply()` at line 2199 |
| 2 | `reply_domain` | `simplelogin.co` | `get_email_domain_part()` at line 974 |
| 3 | `reply_email` (after normalize) | `john_doe_at_example_com_abc123def@simplelogin.co` | `normalize_reply_email()` at line 984 |
| 4 | `contact` | `Contact(id=42, alias_id=7, website_email='john@example.com', reply_email='john_doe_at_example_com_abc123def@simplelogin.co')` | `Contact.get_by(reply_email=reply_email)` at line 986 |
| 5 | `contact.alias` / `alias` | `Alias(id=7, email='myalias@simplelogin.co', user_id=1)` | ORM relationship traversal at line 994 |
| 6 | `alias.user` / `user` | `User(id=1, email='alice@gmail.com')` | ORM relationship traversal at line 1004 |
| 7 | `mail_from` | `alice@gmail.com` | SMTP envelope at line 1005 |
| 8 | `mailbox` | `Mailbox(id=3, email='alice@gmail.com', user_id=1)` | `get_mailbox_from_mail_from()` at line 1019 |
| 9 | DMARC check | `None` (pass) | `apply_dmarc_policy_for_reply_phase()` at line 1012 |
| 10 | FROM header (rewritten) | `myalias@simplelogin.co` | `add_or_replace_header()` at line 1172 |
| 11 | Final delivery target | `contact.website_email` = `john@example.com` | `sl_sendmail()` at line 1224–1226 |

**Value derivation chain**: `rcpt_to` → `reply_email` (normalized) → `Contact` (via DB lookup) → `Alias` (via ORM relationship) → `User` (via ORM relationship) → `Mailbox` (via authorization check) → delivery to `contact.website_email`.

### 4.2 Failure Scenarios

| Scenario | Trigger Condition | Return Value | Code Location |
|----------|------------------|--------------|---------------|
| Invalid domain | `reply_email` domain is not `EMAIL_DOMAIN` and not a registered `SLDomain` | `E501` ("550 SL E501") | Lines 977–981 |
| Contact not found | `Contact.get_by(reply_email=reply_email)` returns `None` | `E502` ("550 SL E502 Email not exist") | Lines 987–989 |
| User soft-deleted | `contact.user.is_active()` returns `False` | `E502` ("550 SL E502 Email not exist") | Lines 990–992 |
| User cannot send | `user.can_send_or_receive()` returns `False` | `E504` ("550 SL E504 Account disabled") | Lines 1007–1009 |
| DMARC rejection | `apply_dmarc_policy_for_reply_phase()` returns a status | The DMARC status code | Lines 1012–1016 |
| Mailbox unauthorized | `get_mailbox_from_mail_from()` returns `None` and `disable_email_spoofing_check` is `False` | `E214` ("250 SL E214 Unauthorized for using reverse alias") | Lines 1019–1034 |
| Invalid alias domain | `is_valid_alias_address_domain()` returns `False` | `E503` ("550 SL E503") | Lines 1000–1002 |

---

## 5. Database Schema and Constraint Analysis

### 5.1 Contact Model Definition

**File**: `app/models.py`, lines 1863–1962

The `Contact` class is defined as follows (relevant columns only):

```python
class Contact(Base, ModelMixin):
    __tablename__ = "contact"

    __table_args__ = (
        sa.UniqueConstraint("alias_id", "website_email", name="uq_contact"),
    )

    user_id = sa.Column(sa.ForeignKey(User.id, ondelete="cascade"), nullable=False, index=True)
    alias_id = sa.Column(sa.ForeignKey(Alias.id, ondelete="cascade"), nullable=False, index=True)
    website_email = sa.Column(sa.String(512), nullable=False)
    reply_email = sa.Column(sa.String(512), nullable=False, index=True)
```

**Key observations**:

1. **`reply_email` column** (line 1899): Defined as `sa.Column(sa.String(512), nullable=False, index=True)`.
   - `index=True` creates a B-tree index for query performance
   - **There is NO `unique=True`** — the index allows duplicate values
   - The column is `nullable=False`, so every contact must have a `reply_email`

2. **`__table_args__`** (lines 1874–1876): `sa.UniqueConstraint("alias_id", "website_email", name="uq_contact")`
   - The **only** unique constraint on the `contact` table covers `(alias_id, website_email)`
   - This ensures each alias can have at most one contact per sender email address
   - **`reply_email` is NOT part of any unique constraint**

3. **Relationships** (lines 1907–1908):
   - `alias = orm.relationship(Alias, backref="contacts")` — many-to-one from Contact to Alias
   - `user = orm.relationship(User)` — many-to-one from Contact to User

### 5.2 Migration Evidence

Three migrations provide authoritative evidence about the `reply_email` column and constraints:

**Migration 1: `5fa68bafae72_.py`** (revision `5fa68bafae72`, created 2019-11-07)

This migration creates the original `forward_email` table (later renamed to `contact`):
- **Line 28**: `sa.Column('reply_email', sa.String(length=128), nullable=False)` — original column definition with no uniqueness constraint
- **Line 31**: `sa.UniqueConstraint('gen_email_id', 'website_email', name='uq_forward_email')` — the unique constraint has always been on the alias+email pair (here using the old column name `gen_email_id`), never on `reply_email`

**Migration 2: `0809266d08ca_.py`** (revision `0809266d08ca`, created 2020-03-17)

This migration renames columns and recreates constraints:
- **Line 44**: Renames `gen_email_id` to `alias_id` in the contact table
- **Line 45**: `op.create_unique_constraint("uq_contact", "contact", ["alias_id", "website_email"])` — creates the `uq_contact` constraint on the renamed columns
- **Line 46**: `op.drop_constraint("uq_forward_email", "contact", type_="unique")` — drops the old unique constraint
- **No unique constraint is created for `reply_email`**

**Migration 3: `78403c7b8089_.py`** (revision `78403c7b8089`, created 2021-07-13)

This migration adds the index on `reply_email`:
- **Line 22**: `op.create_index(op.f('ix_contact_reply_email'), 'contact', ['reply_email'], unique=False)`
- **Explicit `unique=False`** — the developer consciously chose NOT to make this a unique index. The index exists solely for query performance (speeding up the `Contact.get_by(reply_email=...)` lookup), not for data integrity.

### 5.3 Implication Summary

The schema analysis reveals the following implications:

1. **PostgreSQL will accept multiple Contact rows with identical `reply_email` values** without raising any error. The database provides no enforcement of `reply_email` uniqueness.

2. **The application-level `available_sl_email()` check** (`app/models.py` line 1425) **is the only guard** against duplicate `reply_email` values.

3. **There is no `ON CONFLICT` clause, advisory lock, or compare-and-swap mechanism** for `reply_email` in any contact creation path.

4. **The `IntegrityError` handler** in `create_contact()` (line 113) and `replace_header_when_forward()` (line 304) will never fire for duplicate `reply_email` values because there is no constraint to violate. It only catches `uq_contact` violations on `(alias_id, website_email)`.

---

## 6. Race Condition and TOCTOU Vulnerability Analysis

### 6.1 The TOCTOU Window

A Time-of-Check-to-Time-of-Use (TOCTOU) race condition exists in the reply-email generation path. The vulnerability arises because the uniqueness check (SELECT) and the insertion (INSERT+COMMIT) are separate operations with no atomic guarantee.

**Detailed timeline of the race**:

1. **Time T1 — Process A checks availability**:
   - `generate_reply_email()` (line 1136) generates a candidate: `"abc123xyz@simplelogin.co"`
   - Calls `available_sl_email("abc123xyz@simplelogin.co")` (line 1150)
   - `available_sl_email()` calls `Contact.get_by(reply_email="abc123xyz@simplelogin.co")` (line 1428)
   - SELECT query returns no matching Contact → function returns `True`
   - `generate_reply_email()` returns `"abc123xyz@simplelogin.co"` as the generated reply email

2. **Time T2 — Process B checks availability** (concurrent with Process A):
   - `generate_reply_email()` generates the same candidate: `"abc123xyz@simplelogin.co"`
   - Calls `available_sl_email("abc123xyz@simplelogin.co")`
   - SELECT query returns no matching Contact (Process A has not committed yet)
   - Function returns `True` — Process B also considers this address available

3. **Time T3 — Process A inserts**:
   - `Contact.create(reply_email="abc123xyz@simplelogin.co", commit=True)` at `create_contact()` line 92–103
   - INSERT+COMMIT succeeds — the Contact row is now in the database

4. **Time T4 — Process B inserts**:
   - `Contact.create(reply_email="abc123xyz@simplelogin.co", commit=True)`
   - INSERT+COMMIT **also succeeds** because there is no UNIQUE constraint on `reply_email`
   - No `IntegrityError` is raised

5. **Result**: Two `Contact` records exist with `reply_email = "abc123xyz@simplelogin.co"`, potentially belonging to different aliases and different users.

### 6.2 Session and Connection Architecture

**File**: `app/db.py`, lines 1–19 (complete file)

```python
engine = create_engine(
    config.DB_URI, connect_args={"application_name": config.DB_CONN_NAME}
)
connection = engine.connect()

Session = scoped_session(sessionmaker(bind=connection))
```

**Architecture**:
- **Line 9–11**: Creates a SQLAlchemy engine connected to the PostgreSQL database.
- **Line 12**: Establishes a **single** database connection for the entire process: `connection = engine.connect()`.
- **Line 14**: Creates a `scoped_session` bound to that single connection. The `scoped_session` provides thread-local session proxies, but since all sessions share the same underlying connection, database operations within a single process are effectively serialized at the connection level.

**Implications for concurrency**:

1. **Within a single `email_handler` process**: The `aiosmtpd` library calls `_handle()` synchronously at line 2292 (`ret = self._handle(envelope, msg)`) — there is no `await`, so this call blocks the event loop. Combined with the single database connection, this means email processing is **fully serialized** within one process. Two emails cannot be processed concurrently within the same process, which eliminates intra-process TOCTOU races.

2. **Across multiple `email_handler` processes**: Each process creates its own `engine`, `connection`, and `Session` (since `db.py` is module-level). Multiple processes have independent database connections and can execute overlapping transactions. The TOCTOU race described in [Section 6.1](#61-the-toctou-window) is possible when two different `email_handler` processes create contacts concurrently.

3. **Web application + email handler**: The web application (`server.py`) uses the same `Session` from `app/db.py`. If the web dashboard has endpoints that create contacts (e.g., creating contacts manually), these could also race with `email_handler` processes.

### 6.3 App Context Lifecycle

**File**: `server.py`, lines 127–136

```python
def create_light_app() -> Flask:
    app = Flask(__name__)
    app.config["SQLALCHEMY_DATABASE_URI"] = DB_URI
    app.config["SQLALCHEMY_TRACK_MODIFICATIONS"] = False

    @app.teardown_appcontext
    def shutdown_session(response_or_exc):
        Session.remove()

    return app
```

Each `_handle()` invocation (line 2352) creates a fresh Flask app context via `with create_light_app().app_context()`. When the context exits, `Session.remove()` is called, which returns the session's connection to the pool and clears the thread-local session state. This ensures clean session boundaries per email processing cycle.

### 6.4 Collision Probability Assessment

The practical probability of the TOCTOU race producing a collision depends on the random space of generated reply emails.

**`random_string()` implementation** (`app/utils.py` lines 41–47):
```python
def random_string(length=10, include_digits=False):
    letters = string.ascii_lowercase
    if include_digits:
        letters += string.digits
    return "".join(secrets.choice(letters) for _ in range(length))
```

Uses `secrets.choice()` for cryptographically secure randomness, drawing from 26 lowercase ASCII letters.

**Collision probability calculations**:

| Mode | Random Length | Alphabet Size | Possible Values | Collision Probability (per generation) |
|------|-------------|--------------|----------------|---------------------------------------|
| Without sender info | 20–50 chars | 26 | 26^20 ≈ 1.9 × 10^28 to 26^50 ≈ 6.6 × 10^70 | Astronomically low (~10^-28 or less) |
| With sender info | 5–10 chars | 26 | 26^5 ≈ 1.2 × 10^7 to 26^10 ≈ 1.4 × 10^14 | Very low for unique prefixes, but the prefix is sender-specific |

**Practical assessment**: Even with the shorter random component (5–10 characters when including sender), the contact email prefix makes the full reply email nearly unique per sender-alias pair. For two independent processes to generate the same `reply_email`, they would need to:
1. Be processing emails for the same alias from the same sender, OR
2. Generate identical random strings independently

The random space is large enough that accidental collisions are **astronomically unlikely** under normal operation. The TOCTOU vulnerability exists in principle and could theoretically be exploited by a deliberate attack, but it is not a practical concern for routine email processing.

### 6.5 Missing Distributed Locking

The codebase provides no distributed locking for reply-email generation:

- **No `SELECT FOR UPDATE`**: The `available_sl_email()` function at `app/models.py` line 1425 performs a plain SELECT without row-level locking.
- **No advisory lock**: There is no PostgreSQL advisory lock (`pg_advisory_lock`) around the check-and-insert sequence.
- **No Redis-based lock**: While the codebase uses Redis-based distributed locking for other operations (e.g., alias creation via the `parallel_limiter` pattern), no equivalent mechanism exists for reply-email generation.
- **No optimistic concurrency control**: There is no version column or compare-and-swap pattern for `reply_email` on the `Contact` model.

The absence of distributed locking means that the only protection against the TOCTOU race is the probabilistic uniqueness provided by the large random space (see [Section 6.4](#64-collision-probability-assessment)).

---

## 7. Cross-Contact and Cross-User Mismatch Scenarios

### 7.1 Can the Same `reply_email` Resolve to Different Contacts Over Time?

**Answer: Yes, in theory.**

**Reasoning**: If the TOCTOU race described in [Section 6.1](#61-the-toctou-window) produces two `Contact` records with the same `reply_email`, then `Contact.get_by(reply_email=...).first()` will return whichever record PostgreSQL returns first for the `LIMIT 1` query without an `ORDER BY` clause. While PostgreSQL typically returns rows in heap order (which correlates with insertion order and primary key for freshly inserted data), this behavior is:

1. **Not guaranteed by the SQL standard** — there is no ordering contract for `.first()` without `ORDER BY`
2. **Subject to change after table maintenance** — `VACUUM`, `CLUSTER`, or page splits can alter physical row order
3. **Subject to change after row deletion** — if the currently-returned Contact is deleted, the next lookup will return the other Contact

**Evidence chain**:
- `ModelMixin.get_by()` at `app/models.py` lines 82–84 uses `.first()` which maps to SQL `LIMIT 1` without `ORDER BY`
- No unique constraint on `reply_email` (migration `78403c7b8089_.py` line 22: `unique=False`)
- The `ix_contact_reply_email` B-tree index may impose a secondary sort by the indexed column, but when multiple rows have the same indexed value, the tie-breaking order is undefined

**Practical likelihood**: Extremely low under normal operation due to the large random space (see [Section 6.4](#64-collision-probability-assessment)). This scenario requires either a successful TOCTOU race or manual database manipulation.

### 7.2 Can Resolution Fail Temporarily?

**Answer: No, under normal operation.**

**Reasoning**: Once a `Contact` record is created and committed to the database, the `Contact.get_by(reply_email=...)` lookup will find it. PostgreSQL's default transaction isolation level is `READ COMMITTED`, which guarantees that any committed data is visible to all subsequent transactions.

**Evidence**:
- `Contact.create()` with `commit=True` (called at `create_contact()` line 102 and `replace_header_when_forward()` line 303) performs `Session.commit()` which flushes the INSERT and commits the transaction at `app/models.py` line 1956–1957
- `Contact.get_by()` at `app/models.py` line 83 performs a new SELECT within the calling transaction, which will see all committed data
- SQLAlchemy's `scoped_session` (`app/db.py` line 14) ensures proper session management

**Edge case**: If a `Contact` is being deleted (e.g., a user clears their contact list via the web dashboard) concurrently with a reply arriving, the `Contact.get_by()` could return `None` if the DELETE commits between the forward-phase creation and the reply-phase lookup. This would result in `E502` ("Email not exist") at line 989. However, this is not a transient failure — it reflects the actual state of the database (the contact no longer exists). The user would need to re-send the original email to regenerate the contact.

### 7.3 Can Resolution Succeed but Forward to a Different User?

**Answer: Yes, if duplicate `reply_email` records exist, but with an important caveat.**

**Scenario**: Assume two `Contact` records share the same `reply_email` due to a TOCTOU race:
- `Contact_A`: `alias_id=7` (owned by User Alice), `reply_email="abc@sl.co"`
- `Contact_B`: `alias_id=15` (owned by User Bob), `reply_email="abc@sl.co"`

When User Alice replies to `abc@sl.co`:

1. **Step 4** (line 986): `Contact.get_by(reply_email="abc@sl.co")` returns `Contact_A` or `Contact_B` non-deterministically
2. **If `Contact_A` is returned** (correct): Normal processing — Alice's mailbox matches Alias 7's authorized mailboxes, delivery succeeds
3. **If `Contact_B` is returned** (incorrect):
   - Line 994: `alias = Contact_B.alias` → Alias 15 (Bob's alias)
   - Line 1004: `user = alias.user` → User Bob
   - Line 1019: `mailbox = get_mailbox_from_mail_from(mail_from, alias)` checks if Alice's email matches **Bob's** alias's authorized mailboxes
   - **This check will almost certainly FAIL** because Alice's mailbox is not authorized for Bob's alias
   - Line 1030–1034: `handle_unknown_mailbox()` is called, returns `E214`
   - **The email is NOT delivered** — the mailbox authorization check acts as an implicit safety net

**The critical caveat — `disable_email_spoofing_check`**:
- Line 1021: If `alias.disable_email_spoofing_check` is `True` on the **wrong** contact's alias:
  ```python
  if alias.disable_email_spoofing_check:
      mailbox = alias.mailbox
  ```
  Line 1029 falls back to the alias's default mailbox — which is **Bob's** mailbox
- In this case, the reply would be processed as if it came from Bob's mailbox, potentially exposing Alice's reply content to Bob's infrastructure

**Summary**: The mailbox authorization check at `get_mailbox_from_mail_from()` (lines 1364–1387) provides a **practical barrier** against cross-user misrouting in most cases. However:
1. It was not designed for this purpose (it's an anti-spoofing measure)
2. It can be bypassed by the `disable_email_spoofing_check` flag
3. If both users happen to share the same mailbox email address (unlikely but theoretically possible), the check would pass

---

## 8. Behavioral Conclusions

Based on the evidence gathered from the source code, database schema, migration history, and code flow analysis, the following conclusions are established:

### Conclusion 1: `reply_email` Lacks a Database-Level Unique Constraint

The `Contact.reply_email` column at `app/models.py` line 1899 is defined with `index=True` but **not** `unique=True`. Migration `78403c7b8089_.py` (line 22) explicitly creates the index with `unique=False`. The only unique constraint on the `contact` table is `uq_contact` covering `(alias_id, website_email)` (defined at `app/models.py` lines 1874–1876 and created by migration `0809266d08ca_.py` line 45). PostgreSQL will accept multiple rows with identical `reply_email` values without raising any error.

### Conclusion 2: Application-Level Uniqueness Check Has a TOCTOU Window

The `generate_reply_email()` function (`app/email_utils.py` line 1103) calls `available_sl_email()` (`app/models.py` line 1425) which performs a SELECT query to check if a reply email is unused. Between this SELECT and the subsequent INSERT+COMMIT in `Contact.create()` (via `create_contact()` at `app/contact_utils.py` line 92–103 or `replace_header_when_forward()` at `email_handler.py` line 294–303), another process can insert a Contact with the same `reply_email`. Since there is no database-level unique constraint, both INSERTs succeed, producing duplicate `reply_email` records.

### Conclusion 3: `.first()` Produces Non-Deterministic Results for Duplicate Records

`ModelMixin.get_by()` at `app/models.py` lines 82–84 uses `Session.query(cls).filter_by(**kw).first()`, which maps to SQL `LIMIT 1` without an explicit `ORDER BY` clause. If multiple `Contact` rows share the same `reply_email`, the returned row depends on the database engine's internal ordering (heap position, index traversal order), which is not guaranteed to be stable across transactions, table maintenance operations, or PostgreSQL versions.

### Conclusion 4: Single-Process Serialization Mitigates but Does Not Eliminate the Risk

Within a single `email_handler` process, the `aiosmtpd` framework calls `_handle()` synchronously at line 2292 (`ret = self._handle(envelope, msg)`) without using `await`. This blocks the event loop and serializes email processing, preventing intra-process TOCTOU races. The single database connection architecture in `app/db.py` (line 12: `connection = engine.connect()`) further reinforces this serialization within one process. However, multi-process deployments (multiple `email_handler` instances with separate database connections) break this serialization and enable inter-process concurrent contact creation.

### Conclusion 5: The `IntegrityError` Handler Does Not Guard `reply_email`

The `try/except IntegrityError` blocks in `create_contact()` (`app/contact_utils.py` lines 113–119) and `replace_header_when_forward()` (`email_handler.py` lines 304–307) are designed to catch violations of the `uq_contact` unique constraint on `(alias_id, website_email)`. Since there is no unique constraint on `reply_email`, an `IntegrityError` will **never** be raised for duplicate `reply_email` values. The handler performs a fallback lookup by `Contact.get_by(alias_id=..., website_email=...)` which is correct for the `uq_contact` case but entirely irrelevant to `reply_email` duplication.

### Conclusion 6: The Mailbox Authorization Check Provides an Imperfect Safety Net

The `get_mailbox_from_mail_from()` function (`email_handler.py` lines 1364–1387) verifies that the SMTP `mail_from` address matches one of the alias's authorized mailboxes. In the cross-user mismatch scenario (where the wrong Contact is returned due to duplicate `reply_email`), this check will typically **fail** because the replying user's mailbox will not be authorized for the other user's alias. This failure triggers `handle_unknown_mailbox()` (line 1390) and returns `E214`, preventing actual message delivery to the wrong recipient.

However, this safety net has limitations:
1. **Not by design**: The mailbox authorization check is an anti-spoofing measure, not a uniqueness guard. Its protective effect in the cross-user scenario is incidental.
2. **Bypassable**: If the wrongly-resolved alias has `disable_email_spoofing_check = True` (line 1021), the check is skipped and the system falls back to the alias's default mailbox (`alias.mailbox` at line 1029), potentially processing the reply under the wrong user's context.
3. **Shared mailbox emails**: If two users share the same mailbox email address (e.g., both use alice@gmail.com), the authorization check could pass even for the wrong user's alias.

---

## 9. Appendix: Code References

### Complete Reference Table

| File | Function/Element | Lines | Role in Resolution Chain |
|------|-----------------|-------|--------------------------|
| `email_handler.py` | `handle_reply()` | 966–1261 | Core reply-phase resolution function |
| `email_handler.py` | `handle()` | 1945–2234 | Central routing dispatcher |
| `email_handler.py` | `get_or_create_contact()` | 180–211 | Forward-phase contact creation entry point |
| `email_handler.py` | `replace_header_when_forward()` | 239–317 | CC/To header contact creation (parallel path) |
| `email_handler.py` | `replace_header_when_reply()` | 345–384 | Reply-phase header rewriting with contact lookup |
| `email_handler.py` | `get_mailbox_from_mail_from()` | 1364–1387 | Mailbox authorization check |
| `email_handler.py` | `handle_unknown_mailbox()` | 1390–1429 | Unknown sender handling and user notification |
| `email_handler.py` | `MailHandler.handle_DATA()` | 2289 | Async SMTP entry point |
| `email_handler.py` | `_handle()` | 2335 | App context creation and dispatch |
| `app/contact_utils.py` | `create_contact()` | 42–120 | Contact creation with IntegrityError handling |
| `app/contact_utils.py` | `__update_contact_if_needed()` | 28–39 | Contact name/mail_from update on existing contact |
| `app/email_utils.py` | `generate_reply_email()` | 1103–1153 | Reply email generation with random string and uniqueness check |
| `app/email_utils.py` | `is_reverse_alias()` | 1156–1163 | Reverse alias detection via Contact.get_by() |
| `app/email_validation.py` | `normalize_reply_email()` | 25–38 | Non-ASCII and control character normalization |
| `app/models.py` | `Contact` class | 1863–1962 | ORM model with `reply_email` column (line 1899) and `uq_contact` constraint (lines 1874–1876) |
| `app/models.py` | `Contact.create()` | 1938–1962 | ORM override with anti-reverse-alias check |
| `app/models.py` | `ModelMixin.get_by()` | 82–84 | Generic `.filter_by(**kw).first()` lookup |
| `app/models.py` | `available_sl_email()` | 1425–1432 | Application-level uniqueness check (SELECT only) |
| `app/db.py` | Engine, connection, Session | 1–19 | Single connection, scoped session architecture |
| `server.py` | `create_light_app()` | 127–136 | Flask app context factory with session teardown |
| `app/errors.py` | `CannotCreateContactForReverseAlias` | 29–33 | Exception for reverse-alias abuse in contact creation |
| `app/errors.py` | `NonReverseAliasInReplyPhase` | 36–38 | Exception for non-reverse-alias in reply headers |
| `app/email/status.py` | `E501` | Line 38 | "550 SL E501" — invalid reply email domain |
| `app/email/status.py` | `E502` | Line 39 | "550 SL E502 Email not exist" — contact not found |
| `app/email/status.py` | `E503` | Line 40 | "550 SL E503" — invalid alias domain |
| `app/email/status.py` | `E504` | Line 41 | "550 SL E504 Account disabled" |
| `app/email/status.py` | `E214` | Line 22 | "250 SL E214 Unauthorized for using reverse alias" |
| `app/email/rate_limit.py` | `rate_limited_reply_phase()` | 86–93 | Rate limiting using `Contact.get_by(reply_email=...)` |
| `app/utils.py` | `random_string()` | 41–47 | Cryptographic random string generation (lowercase letters) |
| `app/utils.py` | `sanitize_email()` | 97–102 | Email sanitization (strip, lowercase, remove control chars) |
| `app/utils.py` | `convert_to_id()` | 50–56 | Unicode to ASCII conversion |
| `app/config.py` | `EMAIL_DOMAIN` | Line 92 | Primary email domain from environment variable |

### Migration References

| Migration File | Revision ID | Date | Key Change |
|---------------|-------------|------|------------|
| `migrations/versions/5fa68bafae72_.py` | `5fa68bafae72` | 2019-11-07 | Original `forward_email` table with `reply_email` column (line 28) and `uq_forward_email` on `(gen_email_id, website_email)` (line 31) — no unique constraint on `reply_email` |
| `migrations/versions/2020_031711_0809266d08ca_.py` | `0809266d08ca` | 2020-03-17 | Renames to `alias_id`, creates `uq_contact` on `(alias_id, website_email)` (line 45) — still no unique constraint on `reply_email` |
| `migrations/versions/2021_071310_78403c7b8089_.py` | `78403c7b8089` | 2021-07-13 | Creates `ix_contact_reply_email` index with explicit `unique=False` (line 22) — performance index only, not a uniqueness guard |
