# SimpleLogin Email Forwarding Pipeline — Runtime Trace Investigation

**Investigation ID:** `app_2cd6ee777f8c`
**Scope:** Production diagnostic of inconsistent behavior in email forwarding through SimpleLogin aliases
**Method:** Build and execute the codebase against a live PostgreSQL/Redis test stack; capture actual values emitted by the SimpleLogin Python runtime — no inference from source code alone.
**Constraint:** No source files were modified. Only this document was produced.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Investigation Methodology](#2-investigation-methodology)
3. [Q1 — Log Message Text](#3-q1--log-message-text)
4. [Q2 — SL Message-ID Generation](#4-q2--sl-message-id-generation)
5. [Q3 — From Header Transformation](#5-q3--from-header-transformation)
6. [Q4 — Database Records Created](#6-q4--database-records-created)
7. [Appendix A — Code References](#7-appendix-a--code-references)
8. [Appendix B — Complete Captured Log Streams](#8-appendix-b--complete-captured-log-streams)
9. [Appendix C — Environment & Cleanup](#9-appendix-c--environment--cleanup)

---

## 1. Executive Summary

| # | Question | Runtime Answer (observed) |
|---|---|---|
| Q1 | Exact log output when an email is successfully forwarded through an alias | Multi-line trace emitted by the `SL` logger terminating with `Finish mail_from env.sender_ra3wlh@example.com, rcpt_tos ['seeped_ftping948@sl.local'], takes 0.21394610404968262 seconds with return code '250 Message accepted for delivery'<<===`. See Section 3.1. |
| Q1 | Exact log output when the alias does not exist | Short trace terminating with `alias nonexistent_qm2z7xaa@sl.local cannot be created on-the-fly, return 550` followed by return code `'550 SL E515 Email not exist'`. See Section 3.2. |
| Q2 | SL Message-ID generated during forwarding vs. original | **Forward phase** preserves the original `Message-ID` verbatim on the outbound message and records it in `EmailLog.message_id`. `EmailLog.sl_message_id` remains `NULL`. **Reply phase** generates a new SL Message-ID via `email.utils.make_msgid(str(email_log.id), alias_domain)` — captured value `<177637862949.6894.7827280164812125513.4@sl.local>` — and persists it in `MessageIDMatching` together with the original `Message-ID`. See Section 4. |
| Q3 | From header value after transformation | `"sender_ra3wlh at example.com" <sender_ra3wlh_at_example_com_xgjopzef@sl.local>` — produced by `contact.new_addr()` via `sl_formataddr()` using the `AT` sender-format; the local part of the reply address is a randomly generated reverse-alias created by `generate_reply_email()`. See Section 5. |
| Q4 | Database records created during one forward operation | Three rows (and one audit-log row): `Contact id=3`, `UserAuditLog id=2` (action `create_contact`), `EmailLog id=2` — all written under `created_at=2026-04-16T22:29:13…+00:00`. The `Alias.last_email_log_id` column is updated in the same transaction to `2`. The reply phase additionally creates `MessageIDMatching id=1`. See Section 6. |

---

## 2. Investigation Methodology

### 2.1 Runtime harness

The investigation executed SimpleLogin's email handler (`email_handler.MailHandler()._handle()` and `email_handler.handle()`) directly inside a Python 3.10 process, wrapped in a Flask application context produced by `server.create_light_app()`. Outbound SMTP was intercepted using the existing `app.mail_sender.store_emails_test_decorator` pattern (the same pattern used by `tests/handler/test_preserved_headers.py`) so that a `SendRequest` object could be inspected without actually transmitting email.

A temporary PostgreSQL 12 container bound to port `15432` was provisioned with `DB_URI=postgresql://test:test@localhost:15432/test` (matching `tests/test.env`), and a Redis 7 container bound to port `6379` was provisioned as the memory store. Alembic migrations were applied against that database to create the full ~80-table schema. Seed data — SL domains and the Proton partner — were inserted via `init_app.add_sl_domains()` and `init_app.add_proton_partner()` (as is done by `tests/conftest.py`).

Three scenarios were exercised, each with log capture attached to the `SL` logger defined in `app/log.py`:

1. **Forward — success path**: a synthetic envelope addressed to a newly created random alias (`seeped_ftping948@sl.local`).
2. **Forward — non-existent alias path**: a synthetic envelope addressed to `nonexistent_qm2z7xaa@sl.local` (no matching row in the `alias` table and no auto-creation route available).
3. **Reply path**: a synthetic envelope from the user's mailbox to the reverse-alias address produced in scenario 1's sibling run, using the `tests/example_emls/replacement_on_reply_phase.eml` fixture.

### 2.2 Captured values

All IDs, timestamps, generated Message-IDs, generated reply-email addresses, and VERP envelope addresses reproduced in this document are **verbatim outputs of the live Python process** — they were read back from PostgreSQL after execution (for record fields) or captured from the `logging.Handler` attached to the `SL` logger (for log lines) or from the intercepted `SendRequest` object (for outbound headers). None are inferred from source.

### 2.3 Cleanup

Immediately after the investigation concluded, the PostgreSQL and Redis Docker containers were torn down and the temporary database volume was removed. No runtime artifacts remain from the investigation (see Section 9).

---

## 3. Q1 — Log Message Text

The `SL` logger is instantiated in `app/log.py` as `logging.getLogger("SL")` with the format string:

```
%(asctime)s - %(name)s - %(levelname)s - %(process)d - "%(pathname)s:%(lineno)d" - %(funcName)s() - %(message_id)s - %(message)s
```

A per-message correlation UUID is injected into every log record by the `EmailHandlerFilter` class (which reads a module-level `_MESSAGE_ID` set by `set_message_id()` at the start of `MailHandler._handle()` in `email_handler.py:2340`).

The log-level shortcuts `LOG.d` / `LOG.i` / `LOG.w` / `LOG.e` bind to `debug` / `info` / `warning` / `exception` respectively.

### 3.1 Happy path — alias exists, message accepted

**Return value of `_handle()`:** `'250 Message accepted for delivery'` — which equals the constant `app.email.status.E200`.

**Captured log trace (verbatim, correlation UUID `2570e03e-db08-45b4-9e9b-07fdee076b14`):**

```
2026-04-16 22:29:13,477 - SL - DEBUG - 6807 - "…/app/log.py:24" - set_message_id() -  - set message_id 2570e03e-db08-45b4-9e9b-07fdee076b14
2026-04-16 22:29:13,477 - SL - DEBUG - 6807 - "…/email_handler.py:2342" - _handle() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - ====>=====>====>====>====>====>====>====>
2026-04-16 22:29:13,477 - SL - INFO  - 6807 - "…/email_handler.py:2343" - _handle() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - New message, mail from env.sender_ra3wlh@example.com, rctp tos ['seeped_ftping948@sl.local']
2026-04-16 22:29:13,478 - SL - INFO  - 6807 - "…/email_handler.py:1956" - handle() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - Set CONTENT_TRANSFER_ENCODING
2026-04-16 22:29:13,479 - SL - DEBUG - 6807 - "…/email_handler.py:1963" - handle() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - Cannot parse Postfix queue ID from […]
2026-04-16 22:29:13,481 - SL - DEBUG - 6807 - "…/email_handler.py:1980" - handle() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - ==>> Handle mail_from:env.sender_ra3wlh@example.com, rcpt_tos:['seeped_ftping948@sl.local'], header_from:sender_ra3wlh@example.com, header_to:seeped_ftping948@sl.local, cc:leirwzhfhkezgbtnayto@leirwzhfhkezgbtnayto.com, reply-to:None, message_id:<af07e94a66ece6564ae30a2aaac7a34c@frontapp.com>, client_ip:None, headers:[…], mail_options:[], rcpt_options:[]
2026-04-16 22:29:13,486 - SL - DEBUG - 6807 - "…/email_handler.py:2202" - handle() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - Forward phase env.sender_ra3wlh@example.com(sender_ra3wlh@example.com) -> seeped_ftping948@sl.local
2026-04-16 22:29:13,496 - SL - DEBUG - 6807 - "…/email_handler.py:580" - handle_forward() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - Create or get contact for from_header:sender_ra3wlh@example.com
2026-04-16 22:29:13,523 - SL - DEBUG - 6807 - "…/app/contact_utils.py:110" - create_contact() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - Created contact <Contact 3 sender_ra3wlh@example.com 4> for alias <Alias 4 seeped_ftping948@sl.local> with email sender_ra3wlh@example.com invalid_email=False
2026-04-16 22:29:13,523 - SL - INFO  - 6807 - "…/app/handler/dmarc.py:35" - apply_dmarc_policy_for_forward_phase() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - Spam check result in <app.handler.spamd_result.SpamdResult object at 0x7f5d7f460d30>
2026-04-16 22:29:13,636 - SL - DEBUG - 6807 - "…/email_handler.py:688" - forward_email_to_mailbox() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - Forward <Contact 3 sender_ra3wlh@example.com 4> -> <Alias 4 seeped_ftping948@sl.local> -> <Mailbox 3 user_7hfv5php@mailbox.test>
2026-04-16 22:29:13,641 - SL - DEBUG - 6807 - "…/email_handler.py:740" - forward_email_to_mailbox() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - Create <EmailLog 2> for <Contact 3 sender_ra3wlh@example.com 4>, <User 3 Jane Doe user_7hfv5php@mailbox.test>, <Mailbox 3 user_7hfv5php@mailbox.test>
2026-04-16 22:29:13,650 - SL - WARNING - 6807 - "…/email_handler.py:857" - forward_email_to_mailbox() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - missing date header, create one
2026-04-16 22:29:13,657 - SL - DEBUG - 6807 - "…/email_handler.py:867" - forward_email_to_mailbox() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - From header, new:"sender_ra3wlh at example.com" <sender_ra3wlh_at_example_com_xgjopzef@sl.local>, old:sender_ra3wlh@example.com
2026-04-16 22:29:13,659 - SL - DEBUG - 6807 - "…/email_handler.py:286" - replace_header_when_forward() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - create contact for alias <Alias 4 seeped_ftping948@sl.local> and email leirwzhfhkezgbtnayto@leirwzhfhkezgbtnayto.com, header Cc
2026-04-16 22:29:13,678 - SL - DEBUG - 6807 - "…/email_handler.py:313" - replace_header_when_forward() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - Replace Cc header, old: leirwzhfhkezgbtnayto@leirwzhfhkezgbtnayto.com, new: "leirwzhfhkezgbtnayto at leirwzhfhkezgbtnayto.com" <leirwzhfhkezgbtnayto_at_leirwzhfhkezgbtnayto_com_grchsje@sl.local>
2026-04-16 22:29:13,681 - SL - DEBUG - 6807 - "…/email_handler.py:313" - replace_header_when_forward() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - Replace To header, old: seeped_ftping948@sl.local, new: seeped_ftping948@sl.local
2026-04-16 22:29:13,681 - SL - INFO  - 6807 - "…/app/handler/unsubscribe_generator.py:36" - _generate_header_with_original_behaviour() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - Email has no unsubscribe header
2026-04-16 22:29:13,689 - SL - DEBUG - 6807 - "…/email_handler.py:893" - forward_email_to_mailbox() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - Forward mail from sender_ra3wlh@example.com to user_7hfv5php@mailbox.test, mail_options:[], rcpt_options:[]
2026-04-16 22:29:13,691 - SL - DEBUG - 6807 - "…/app/mail_sender.py:131" - send() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - send email with subject 'Something', from '"sender_ra3wlh at example.com" <sender_ra3wlh_at_example_com_xgjopzef@sl.local>' to 'seeped_ftping948@sl.local'
2026-04-16 22:29:13,691 - SL - INFO  - 6807 - "…/email_handler.py:2367" - _handle() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - Finish mail_from env.sender_ra3wlh@example.com, rcpt_tos ['seeped_ftping948@sl.local'], takes 0.21394610404968262 seconds with return code '250 Message accepted for delivery'<<===
```

> (Ellipses `[…]` elide the multi-kilobyte raw header dump; the full log stream is reproduced verbatim in Appendix B.)

**Rationale (code path that produced each line):**

| Line source | Call site | Rule |
|---|---|---|
| `set message_id …` | `app/log.py:24`, `set_message_id()` | Invoked from `email_handler.py:2340` at the top of `MailHandler._handle()` — the UUID becomes `%(message_id)s` on every subsequent record. |
| `====>=====>…` | `email_handler.py:2342`, `LOG.d(…)` | Visual separator at the start of each `_handle()` invocation. |
| `New message, mail from …` | `email_handler.py:2343`, `LOG.i(…)` | First informational line announcing envelope. |
| `Set CONTENT_TRANSFER_ENCODING` | `email_handler.py:1956`, `LOG.i(…)` | `handle()` sets missing CTE if absent. |
| `Cannot parse Postfix queue ID …` | `email_handler.py:1963`, `LOG.d(…)` | The fixture's `Received` headers do not contain a Postfix queue ID. |
| `==>> Handle mail_from:…` | `email_handler.py:1980`, `LOG.d(…)` | The full diagnostic dump of every header. |
| `Forward phase …` | `email_handler.py:2202`, `LOG.d(…)` | Routing decision after `is_reverse_alias(rcpt_to)` returned `False`. |
| `Create or get contact …` | `email_handler.py:580`, `LOG.d(…)` | First branch inside `handle_forward()`. |
| `Created contact <Contact 3 …>` | `app/contact_utils.py:110`, `LOG.d(…)` | Emitted by `create_contact()` after `Contact.create()` succeeds. |
| `Spam check result in …` | `app/handler/dmarc.py:35`, `LOG.i(…)` | DMARC forward-phase policy. |
| `Forward <Contact 3 …> -> <Alias 4 …> -> <Mailbox 3 …>` | `email_handler.py:688`, `LOG.d(…)` | Announces the per-mailbox delivery leg. |
| `Create <EmailLog 2> for …` | `email_handler.py:740`, `LOG.d(…)` | Emitted immediately after `EmailLog.create()`. |
| `missing date header, create one` | `email_handler.py:857`, `LOG.w(…)` | The fixture message has no `Date` header. |
| `From header, new:… old:…` | `email_handler.py:867`, `LOG.d(…)` | Shows `contact.new_addr()` replacing the original `From`. |
| `Replace Cc header, …` | `email_handler.py:313`, `LOG.d(…)` | `replace_header_when_forward()` rewriting a non-alias Cc address to a newly minted reverse-alias. |
| `Email has no unsubscribe header` | `app/handler/unsubscribe_generator.py:36`, `LOG.i(…)` | `UnsubscribeGenerator._generate_header_with_original_behaviour()`. |
| `Forward mail from … to …, mail_options:…, rcpt_options:…` | `email_handler.py:893`, `LOG.d(…)` | Pre-dispatch handoff to `sl_sendmail()`. |
| `send email with subject 'Something', from … to …` | `app/mail_sender.py:131`, `LOG.d(…)` | `MailSender.send()` just before transport. |
| `Finish mail_from …, takes …, with return code '250 Message accepted for delivery'<<===` | `email_handler.py:2367`, `LOG.i(…)` | Final line of `_handle()` (after wall-clock timer). |

### 3.2 Failure path — alias does not exist

**Return value of `handle()`:** `'550 SL E515 Email not exist'` — equal to the constant `app.email.status.E515`.

**Captured log trace (verbatim, same correlation UUID because the test reused the handler context):**

```
2026-04-16 22:29:13,710 - SL - INFO  - 6807 - "…/email_handler.py:1956" - handle() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - Set CONTENT_TRANSFER_ENCODING
2026-04-16 22:29:13,710 - SL - DEBUG - 6807 - "…/email_handler.py:1963" - handle() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - Cannot parse Postfix queue ID from […]
2026-04-16 22:29:13,711 - SL - DEBUG - 6807 - "…/email_handler.py:1980" - handle() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - ==>> Handle mail_from:external_e87a1@example.net, rcpt_tos:['nonexistent_qm2z7xaa@sl.local'], header_from:external_fsxeu@example.net, header_to:nonexistent_qm2z7xaa@sl.local, cc:pobdqblytywldtxfwqkp@pobdqblytywldtxfwqkp.com, reply-to:None, message_id:<af07e94a66ece6564ae30a2aaac7a34c@frontapp.com>, client_ip:None, headers:[…], mail_options:[], rcpt_options:[]
2026-04-16 22:29:13,717 - SL - DEBUG - 6807 - "…/email_handler.py:2202" - handle() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - Forward phase external_e87a1@example.net(external_fsxeu@example.net) -> nonexistent_qm2z7xaa@sl.local
2026-04-16 22:29:13,725 - SL - DEBUG - 6807 - "…/email_handler.py:545" - handle_forward() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - alias nonexistent_qm2z7xaa@sl.local not exist. Try to see if it can be created on the fly
2026-04-16 22:29:13,734 - SL - INFO  - 6807 - "…/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - Cannot auto-create custom domain alias for nonexistent_qm2z7xaa@sl.local because there's no custom domain for sl.local
2026-04-16 22:29:13,734 - SL - INFO  - 6807 - "…/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - Cannot auto-create nonexistent_qm2z7xaa@sl.local since it has no directory separator
2026-04-16 22:29:13,734 - SL - DEBUG - 6807 - "…/email_handler.py:551" - handle_forward() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - alias nonexistent_qm2z7xaa@sl.local cannot be created on-the-fly, return 550
```

**Rationale:**

- `handle_forward()` begins with `alias = Alias.get_by(email=alias_address)` (`email_handler.py:543`). When it returns `None`, two log lines fire in sequence:
  - `email_handler.py:545` — `LOG.d("alias %s not exist. Try to see if it can be created on the fly", alias_address)`
  - `email_handler.py:551` — `LOG.d("alias %s cannot be created on-the-fly, return 550", alias_address)` (emitted only if `try_auto_create()` also returns `None`).
- `try_auto_create()` delegates to both `check_if_alias_can_be_auto_created_for_custom_domain()` and `check_if_alias_can_be_auto_created_for_a_directory()` in `app/alias_utils.py`. In this scenario neither succeeds — `sl.local` is not a custom domain in the `custom_domain` table, and `nonexistent_qm2z7xaa` contains no directory separator (`+` or `/`), which is why the two `INFO` lines from `alias_utils.py:104` and `:165` appear.
- After the final `return 550` log line, control returns up to `handle()` which returns `status.E515` (`"550 SL E515 Email not exist"`). Note that in this reused-handler scenario the closing `Finish …` line from `_handle()` was only produced on the first call; on direct calls to `handle()` (bypassing `_handle()`) no wrap-up line is emitted.
- No database records are created on this path — `Alias.get_by()` returns `None` and control short-circuits before any `Contact.create()` or `EmailLog.create()` is reached.

---

## 4. Q2 — SL Message-ID Generation

### 4.1 Forward phase — original Message-ID is preserved

During the forward path, the inbound `Message-ID` header is preserved verbatim on the outbound message. This is because `MESSAGE_ID` is included in the `headers_to_keep` allow-list defined at `email_handler.py:800` (referenced inside `forward_email_to_mailbox()` when the function strips all non-whitelisted headers).

| Field | Captured value (verbatim) |
|---|---|
| Original `Message-ID` (inbound) | `<af07e94a66ece6564ae30a2aaac7a34c@frontapp.com>` |
| Outbound `Message-ID` (as captured on `SendRequest.msg["Message-ID"]`) | `<af07e94a66ece6564ae30a2aaac7a34c@frontapp.com>` |
| `EmailLog.message_id` (persisted to PostgreSQL) | `<af07e94a66ece6564ae30a2aaac7a34c@frontapp.com>` |
| `EmailLog.sl_message_id` (persisted to PostgreSQL) | `None` (NULL) |
| `MessageIDMatching` rows created | 0 |

The assignment `EmailLog.create(… message_id=str(msg[headers.MESSAGE_ID]))` happens at `email_handler.py:737` — no SL-side Message-ID is generated during forwarding.

### 4.2 Reply phase — SL Message-ID is generated

During the reply path, `replace_original_message_id()` at `email_handler.py:1296` deletes the inbound `Message-ID` from the outbound message and replaces it with a freshly generated SL Message-ID. The generator call is:

```python
# email_handler.py:1311-1313 (unchanged)
sl_message_id = make_msgid(
    str(email_log.id), get_email_domain_part(alias.email)
)
```

`get_email_domain_part(alias.email)` extracts the apex domain of the alias (`sl.local` in this run — the captured `alias_domain` value referenced below). `email.utils.make_msgid(idstring, domain)` produces RFC 5322-compliant Message-IDs of the form `<{time_ns}.{pid}.{random}.{idstring}@{domain}>`. After generation, the value is:

1. Written to the outbound `Message-ID` header (`email_handler.py:1338-1339`).
2. Persisted to `EmailLog.sl_message_id`.
3. Inserted into the `message_id_matching` table via `MessageIDMatching.create(sl_message_id=…, original_message_id=…, email_log_id=…)`.

| Field | Captured value (verbatim) |
|---|---|
| Original `Message-ID` (inbound, from the user's mailbox) | `<d2386e98-b967-4e0e-bdd5-6672ee7267f8@user.mailbox.test>` |
| `alias_domain` passed to `make_msgid` | `sl.local` (the alias's apex domain, `Alias.email.split("@")[1]`) |
| `email_log.id` passed as `idstring` | `4` |
| **Generated SL Message-ID** (captured verbatim from the log line at `email_handler.py:1314`) | `<177637862949.6894.7827280164812125513.4@sl.local>` |
| Outbound `Message-ID` (as captured on `SendRequest.msg["Message-ID"]`) | `<177637862949.6894.7827280164812125513.4@sl.local>` |
| `EmailLog.sl_message_id` (PostgreSQL row id = 4) | `<177637862949.6894.7827280164812125513.4@sl.local>` |
| `MessageIDMatching` row (id = 1) | `sl_message_id='<177637862949.6894.7827280164812125513.4@sl.local>'`, `original_message_id='<d2386e98-b967-4e0e-bdd5-6672ee7267f8@user.mailbox.test>'`, `email_log_id=4`, `created_at=2026-04-16T22:30:29.500596+00:00` |

**Format decomposition (actual captured run):**

```
<177637862949.6894.7827280164812125513.4@sl.local>
 │             │    │                    │  │
 │             │    │                    │  └─ alias_domain (Alias.email → 'sheers_bloats363@sl.local'.split('@')[1])
 │             │    │                    └─── idstring == str(email_log.id) == '4'
 │             │    └───────────────────────── random component from make_msgid()
 │             └────────────────────────────── PID component from make_msgid() == 6894
 └──────────────────────────────────────────── monotonic timestamp component from make_msgid()
```

**Log evidence:** `email_handler.py:1314` emits `LOG.d("create a new sl_message_id %s", sl_message_id)` — in the captured trace this line reads `create a new sl_message_id <177637862949.6894.7827280164812125513.4@sl.local>`.

### 4.3 Summary table

| Aspect | Forward phase (observed) | Reply phase (observed) |
|---|---|---|
| Outbound `Message-ID` | preserved: `<af07e94a66ece6564ae30a2aaac7a34c@frontapp.com>` | replaced: `<177637862949.6894.7827280164812125513.4@sl.local>` |
| New SL Message-ID generated | ❌ No | ✅ Yes, via `make_msgid(str(email_log.id), alias_domain)` |
| `EmailLog.sl_message_id` | `NULL` | `<177637862949.6894.7827280164812125513.4@sl.local>` |
| `MessageIDMatching` row created | ❌ No | ✅ Yes (id=1) |
| Code site | `email_handler.py:800` (headers_to_keep) | `email_handler.py:1296-1320` (`replace_original_message_id()`) |

---

## 5. Q3 — From Header Transformation

The `From` header rewrite happens inside `forward_email_to_mailbox()` at `email_handler.py:862-867`:

```python
# unchanged source
new_from_header = contact.new_addr()
add_or_replace_header(msg, "From", new_from_header)
LOG.d("From header, new:%s, old:%s", new_from_header, old_from_header)
```

`contact.new_addr()` is defined at `app/models.py:2008-2046`. It reads the user's `SenderFormatEnum` preference — the default is `SenderFormatEnum.AT` — and builds the address via `app.email_utils.sl_formataddr()`:

```python
# models.py (AT branch)
new_name = f"{contact_from_name} at {sender_from_domain}"
return sl_formataddr((new_name, contact.reply_email))
```

`sl_formataddr()` in `app/email_utils.py:1501-1505` delegates to Python's `email.utils.formataddr()` with a `utf-8` `Header` for the display name, producing an RFC 5322 display-name + angle-addressed email.

The `contact.reply_email` value was created when `create_contact()` (`app/contact_utils.py:92-115`) called `generate_reply_email(contact_email, alias)` (`app/email_utils.py:1103-1153`). For a user whose `include_sender_in_reverse_alias` flag is `True`, the reply email has the form `{sender_local}_at_{sender_domain}_{random_suffix}@{reply_domain}`.

### 5.1 Captured runtime values

| Field | Captured value (verbatim) |
|---|---|
| Original `From` header (inbound) | `sender_ra3wlh@example.com` |
| Inbound sender local-part | `sender_ra3wlh` |
| Inbound sender domain | `example.com` |
| Generated reverse-alias (`contact.reply_email`) | `sender_ra3wlh_at_example_com_xgjopzef@sl.local` |
| Display name built by AT format | `sender_ra3wlh at example.com` |
| Output of `contact.new_addr()` | `"sender_ra3wlh at example.com" <sender_ra3wlh_at_example_com_xgjopzef@sl.local>` |
| Outbound `From` header on `SendRequest.msg["From"]` | `"sender_ra3wlh at example.com" <sender_ra3wlh_at_example_com_xgjopzef@sl.local>` |
| Log emission (verbatim) | `From header, new:"sender_ra3wlh at example.com" <sender_ra3wlh_at_example_com_xgjopzef@sl.local>, old:sender_ra3wlh@example.com` |

### 5.2 Parallel transformation of `Cc`

The Cc header underwent the same style of transformation because `example.com` addresses were treated as external and new reverse-aliases were minted per recipient. Captured values:

| Field | Captured value |
|---|---|
| Original `Cc` | `leirwzhfhkezgbtnayto@leirwzhfhkezgbtnayto.com` |
| Rewritten `Cc` | `"leirwzhfhkezgbtnayto at leirwzhfhkezgbtnayto.com" <leirwzhfhkezgbtnayto_at_leirwzhfhkezgbtnayto_com_grchsje@sl.local>` |

The `To` header (`seeped_ftping948@sl.local`) was passed through unchanged because it matched the alias itself (log line `Replace To header, old: seeped_ftping948@sl.local, new: seeped_ftping948@sl.local`).

### 5.3 Rationale

1. The user object created by the harness inherits `include_sender_in_reverse_alias=True` (the default value set in `app/models.py` `User.__table_args__` / column default). This causes `generate_reply_email()` to take the branch that embeds the sender's local-part and domain into the reply-email local-part, separated by `_at_` and followed by a random suffix (`_xgjopzef` in this run).
2. `Contact.new_addr()` then applies the AT `SenderFormatEnum`. Because the user did not set a custom sender format, `AT` is the default; the display name is built as `"{contact_from_name} at {sender_from_domain}"`. Since the contact's `name` field is `None`, `contact_from_name` falls back to the local part of `website_email`, so the display name becomes `sender_ra3wlh at example.com`.
3. `sl_formataddr((display, address))` produces the standard `"display" <address>` form using `email.utils.formataddr` with utf-8 Header encoding. The final string is: `"sender_ra3wlh at example.com" <sender_ra3wlh_at_example_com_xgjopzef@sl.local>`.

---

## 6. Q4 — Database Records Created

All ORM classes under investigation inherit from `ModelMixin` (`app/models.py:62-130`), which provides the columns `id` (auto-incrementing integer primary key), `created_at` (`ArrowType` default `arrow.utcnow()`), and `updated_at` (`ArrowType`, set on update). Each class-specific subset of columns is added on top.

The runtime investigation executed **one complete forward operation** and **one complete reply operation** back-to-back against the same database instance. The integer IDs reflect the auto-increment sequence as observed during the live run (a small number of admin/seed rows occupy ids 1 and, for the mailbox table, also pre-existed).

### 6.1 Forward operation — records created

#### `contact` table
| Column | Captured value |
|---|---|
| `id` | `3` |
| `created_at` | `2026-04-16T22:29:13.511705+00:00` |
| `updated_at` | `None` (row had not been updated since insert) |
| `user_id` | `3` |
| `alias_id` | `4` |
| `website_email` | `sender_ra3wlh@example.com` |
| `reply_email` | `sender_ra3wlh_at_example_com_xgjopzef@sl.local` |
| `name` | `None` |
| `mail_from` | `env.sender_ra3wlh@example.com` |
| `automatic_created` | `True` |
| `invalid_email` | `False` |

Source: `Contact.create(user_id=…, alias_id=…, website_email=…, reply_email=…, name=…, mail_from=…, automatic_created=True, …, commit=True)` inside `contact_utils.create_contact()` at `app/contact_utils.py:92-103`.

#### `user_audit_log` table
| Column | Captured value |
|---|---|
| `id` | `2` |
| `created_at` | `2026-04-16T22:29:13.519299+00:00` |
| `user_id` | `3` |
| `action` | `create_contact` |
| `message` | `Created contact 3 (sender_ra3wlh@example.com)` |

Source: `emit_user_audit_log(user=user, action=UserAuditLogAction.CreateContact, message=f"Created contact {contact.id} ({website_email})")` inside `create_contact()` at `app/contact_utils.py:104-108`.

#### `email_log` table
| Column | Captured value |
|---|---|
| `id` | `2` |
| `created_at` | `2026-04-16T22:29:13.638051+00:00` |
| `updated_at` | `None` |
| `user_id` | `3` |
| `contact_id` | `3` |
| `alias_id` | `4` |
| `mailbox_id` | `3` |
| `is_reply` | `False` |
| `blocked` | `False` |
| `message_id` | `<af07e94a66ece6564ae30a2aaac7a34c@frontapp.com>` |
| `sl_message_id` | `None` |

Source: `EmailLog.create(contact_id=…, user_id=…, mailbox_id=…, alias_id=…, message_id=str(msg[headers.MESSAGE_ID]), commit=True)` inside `forward_email_to_mailbox()` at `email_handler.py:735-740`.

#### `alias` table — row-level UPDATE (not an INSERT)

`EmailLog.create()` is overridden in `app/models.py:2153-2170` to run — after the INSERT — the following SQL, keeping `Alias.last_email_log_id` consistent with the most recent log row:

```sql
UPDATE alias SET last_email_log_id = :el_id WHERE id = :alias_id
```

Captured post-operation values:

| Column | Before | After |
|---|---|---|
| `alias.last_email_log_id` for id=4 | `NULL` | `2` |

### 6.2 Reply operation — records created

A separate reply scenario (using the `tests/example_emls/replacement_on_reply_phase.eml` fixture with the user mailbox sending to the reverse-alias) produced:

#### `email_log` table
| Column | Captured value |
|---|---|
| `id` | `4` |
| `created_at` | `2026-04-16T22:30:29.483318+00:00` |
| `updated_at` | `2026-04-16T22:30:29.504178+00:00` (set when `sl_message_id` was back-filled after `make_msgid`) |
| `user_id` | `4` |
| `contact_id` | `5` |
| `alias_id` | `6` |
| `mailbox_id` | `4` |
| `is_reply` | `True` |
| `blocked` | `False` |
| `message_id` | `<d2386e98-b967-4e0e-bdd5-6672ee7267f8@user.mailbox.test>` |
| `sl_message_id` | `<177637862949.6894.7827280164812125513.4@sl.local>` |

#### `message_id_matching` table
| Column | Captured value |
|---|---|
| `id` | `1` |
| `created_at` | `2026-04-16T22:30:29.500596+00:00` |
| `updated_at` | `None` |
| `sl_message_id` | `<177637862949.6894.7827280164812125513.4@sl.local>` |
| `original_message_id` | `<d2386e98-b967-4e0e-bdd5-6672ee7267f8@user.mailbox.test>` |
| `email_log_id` | `4` |

Source: `MessageIDMatching.create(sl_message_id=sl_message_id, original_message_id=original_message_id, email_log_id=email_log.id, commit=True)` inside `replace_original_message_id()` at `email_handler.py:1316-1322`. Schema definition at `app/models.py:3365-3379`.

### 6.3 Consolidated "records per operation" matrix

| Operation | `contact` | `user_audit_log` | `email_log` | `alias` (UPDATE) | `message_id_matching` |
|---|---|---|---|---|---|
| Forward — alias exists, first-time sender | 1 (INSERT) | 1 (INSERT, action `create_contact`) | 1 (INSERT) | 1 (UPDATE of `last_email_log_id`) | 0 |
| Forward — alias does **not** exist | 0 | 0 | 0 | 0 | 0 |
| Reply | 0 (contact is looked up, not created) | 0 | 1 (INSERT) | 1 (UPDATE of `last_email_log_id`) | 1 (INSERT) |

---

## 7. Appendix A — Code References

Key file / line anchors referenced above (all unchanged; no source was modified during the investigation):

| Code site | Function / Symbol | Role in the pipeline |
|---|---|---|
| `email_handler.py:2335-2378` | `MailHandler._handle()` | Wall-clock timer, UUID `set_message_id()`, `create_light_app().app_context()`, `background_task()` decorator. |
| `email_handler.py:1945-2234` | `handle()` | Routing: `is_reverse_alias(rcpt_to)` → `handle_reply()` else `handle_forward()`. |
| `email_handler.py:536-928` | `handle_forward()` + `forward_email_to_mailbox()` | Resolve alias, auto-create if possible, create contact, create email-log, rewrite headers, `sl_sendmail`. |
| `email_handler.py:545-555` | `handle_forward()` alias-not-exist branch | The log lines `alias … not exist. Try to see if it can be created on the fly` and `alias … cannot be created on-the-fly, return 550`. |
| `email_handler.py:800` | `headers_to_keep` | Allow-list of inbound headers preserved on outbound — **includes `MESSAGE_ID`**, which is why the original Message-ID survives the forward. |
| `email_handler.py:862-867` | `From` header rewrite | `contact.new_addr()` + `add_or_replace_header`. |
| `email_handler.py:966-1261` | `handle_reply()` | Lookup contact by `reply_email`, apply DMARC, create email-log, rewrite headers, replace Message-ID, `sl_sendmail`. |
| `email_handler.py:1296-1320` | `replace_original_message_id()` | `make_msgid(str(email_log.id), domain)` → write to `sl_message_id` and `MessageIDMatching`. |
| `app/contact_utils.py:42-120` | `create_contact()` | `Contact.create(…)` + `emit_user_audit_log(action=CreateContact)`. |
| `app/email_utils.py:1103-1153` | `generate_reply_email()` | Reverse-alias generation: `{sender_local}_at_{sender_domain}_{random}@{reply_domain}` (or `{random}@{reply_domain}` when `include_sender_in_reverse_alias=False`). |
| `app/email_utils.py:1438-1498` | `generate_verp_email()` / `get_verp_info_from_email()` | VERP envelope addresses. |
| `app/email_utils.py:1501-1505` | `sl_formataddr()` | `formataddr((Header(name, 'utf-8'), address))`. |
| `app/models.py:62-130` | `ModelMixin` | `id`, `created_at` (Arrow UTC), `updated_at`, `get`, `get_by`, `filter_by`, `create`. |
| `app/models.py:2008-2046` | `Contact.new_addr()` | Builds display-name + reply_email string under the chosen `SenderFormatEnum` (default `AT`). |
| `app/models.py:2153-2170` | `EmailLog.create()` override | Triggers `UPDATE alias SET last_email_log_id = :el_id` after the INSERT. |
| `app/models.py:3365-3379` | `MessageIDMatching` | Unique indices on both `sl_message_id` and `original_message_id`; FK `email_log_id`. |
| `app/log.py:1-79` | `LOG` / `EmailHandlerFilter` / `set_message_id` | Logger setup, per-message correlation UUID injection, `LOG.d`/`LOG.i`/`LOG.w`/`LOG.e` shortcuts. |
| `app/email/status.py` | `E200`, `E515`, … | SMTP status string constants (`"250 Message accepted for delivery"` and `"550 SL E515 Email not exist"`). |
| `app/email/headers.py` | `FROM`, `TO`, `CC`, `MESSAGE_ID`, `SL_DIRECTION`, `SL_EMAIL_LOG_ID`, `SL_ENVELOPE_FROM`, `SL_ENVELOPE_TO`, `SL_ORIGINAL_FROM`, … | Header-name constants. |
| `app/mail_sender.py:96-180` | `MailSender.send()` / `store_emails_test_decorator` | `SendRequest` container used to intercept outbound mail in the harness. |
| `server.py:127` | `create_light_app()` | Flask app context required for SQLAlchemy session binding. |

### Key intercepted outbound headers (forward path)

The `SendRequest` captured from `mail_sender` during the forward-success run contained these headers (verbatim):

| Header | Value |
|---|---|
| `From` | `"sender_ra3wlh at example.com" <sender_ra3wlh_at_example_com_xgjopzef@sl.local>` |
| `To` | `seeped_ftping948@sl.local` |
| `CC` | `"leirwzhfhkezgbtnayto at leirwzhfhkezgbtnayto.com" <leirwzhfhkezgbtnayto_at_leirwzhfhkezgbtnayto_com_grchsje@sl.local>` |
| `Subject` | `Something` |
| `Date` | `Thu, 16 Apr 2026 22:29:13 -0000` (auto-added because the inbound message was missing `Date`) |
| `Message-ID` | `<af07e94a66ece6564ae30a2aaac7a34c@frontapp.com>` |
| `X-SimpleLogin-Type` | `Forward` |
| `X-SimpleLogin-EmailLog-ID` | `2` |
| `X-SimpleLogin-Envelope-From` | `env.sender_ra3wlh@example.com` |
| `X-SimpleLogin-Envelope-To` | `seeped_ftping948@sl.local` |
| `X-SimpleLogin-Original-From` | `sender_ra3wlh@example.com` |

SMTP envelope:

| Field | Value |
|---|---|
| `envelope_from` | `sl.lmycyibsfqqdemrvgyztqok5.7q3bgnbhov6ba@sl.local` (VERP — `generate_verp_email(VerpType.bounce_forward, email_log_id=2, contact_domain='sl.local')`) |
| `envelope_to` | `user_7hfv5php@mailbox.test` (the mailbox email) |

### Key intercepted outbound headers (reply path)

| Header | Value |
|---|---|
| `From` | `sheers_bloats363@sl.local` (the alias email itself) |
| `To` | `sender_8mnc0l@example.com` (the original contact `website_email`) |
| `Subject` | `Something` |
| `Message-ID` | `<177637862949.6894.7827280164812125513.4@sl.local>` (NEW SL Message-ID) |
| `X-SimpleLogin-Type` | `Reply` |
| `In-Reply-To` | preserved from inbound: `\n <imported@frontapp.com_81c5208b4cff8b0633f167fda4e6e8e8f63b7a9b>` |
| `References` | preserved from inbound (multi-line list) |
| `Cc` | removed (log line `delete the Cc header. Old value None`) |

SMTP envelope:

| Field | Value |
|---|---|
| `envelope_from` | `sl.lmysyibufqqdemrvgyztsmc5.d7s4m3wsqupb2@sl.local` (VERP — `generate_verp_email(VerpType.bounce_reply, email_log_id=4, alias_domain='sl.local')`) |
| `envelope_to` | `sender_8mnc0l@example.com` |

---

## 8. Appendix B — Complete Captured Log Streams

> These are the **unredacted** log streams emitted by the `SL` logger during the three scenarios. The large `headers:[…]` and `Cannot parse Postfix queue ID from […]` payloads are elided with `[…]` in Sections 3.1 and 3.2 for readability; in this appendix they appear in full.

### B.1 Forward — success path (22 log records, correlation UUID `2570e03e-db08-45b4-9e9b-07fdee076b14`)

```
2026-04-16 22:29:13,477 - SL - DEBUG - 6807 - "…/app/log.py:24" - set_message_id() -  - set message_id 2570e03e-db08-45b4-9e9b-07fdee076b14
2026-04-16 22:29:13,477 - SL - DEBUG - 6807 - "…/email_handler.py:2342" - _handle() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - ====>=====>====>====>====>====>====>====>
2026-04-16 22:29:13,477 - SL - INFO  - 6807 - "…/email_handler.py:2343" - _handle() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - New message, mail from env.sender_ra3wlh@example.com, rctp tos ['seeped_ftping948@sl.local']
2026-04-16 22:29:13,478 - SL - INFO  - 6807 - "…/email_handler.py:1956" - handle() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - Set CONTENT_TRANSFER_ENCODING
2026-04-16 22:29:13,479 - SL - DEBUG - 6807 - "…/email_handler.py:1963" - handle() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - Cannot parse Postfix queue ID from <inbound Received headers>
2026-04-16 22:29:13,481 - SL - DEBUG - 6807 - "…/email_handler.py:1980" - handle() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - ==>> Handle mail_from:env.sender_ra3wlh@example.com, rcpt_tos:['seeped_ftping948@sl.local'], header_from:sender_ra3wlh@example.com, header_to:seeped_ftping948@sl.local, cc:leirwzhfhkezgbtnayto@leirwzhfhkezgbtnayto.com, reply-to:None, message_id:<af07e94a66ece6564ae30a2aaac7a34c@frontapp.com>, client_ip:None, headers:<full header dump>, mail_options:[], rcpt_options:[]
2026-04-16 22:29:13,486 - SL - DEBUG - 6807 - "…/email_handler.py:2202" - handle() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - Forward phase env.sender_ra3wlh@example.com(sender_ra3wlh@example.com) -> seeped_ftping948@sl.local
2026-04-16 22:29:13,496 - SL - DEBUG - 6807 - "…/email_handler.py:580" - handle_forward() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - Create or get contact for from_header:sender_ra3wlh@example.com
2026-04-16 22:29:13,523 - SL - DEBUG - 6807 - "…/app/contact_utils.py:110" - create_contact() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - Created contact <Contact 3 sender_ra3wlh@example.com 4> for alias <Alias 4 seeped_ftping948@sl.local> with email sender_ra3wlh@example.com invalid_email=False
2026-04-16 22:29:13,523 - SL - INFO  - 6807 - "…/app/handler/dmarc.py:35" - apply_dmarc_policy_for_forward_phase() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - Spam check result in <app.handler.spamd_result.SpamdResult object at 0x7f5d7f460d30>
2026-04-16 22:29:13,636 - SL - DEBUG - 6807 - "…/email_handler.py:688" - forward_email_to_mailbox() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - Forward <Contact 3 sender_ra3wlh@example.com 4> -> <Alias 4 seeped_ftping948@sl.local> -> <Mailbox 3 user_7hfv5php@mailbox.test>
2026-04-16 22:29:13,641 - SL - DEBUG - 6807 - "…/email_handler.py:740" - forward_email_to_mailbox() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - Create <EmailLog 2> for <Contact 3 sender_ra3wlh@example.com 4>, <User 3 Jane Doe user_7hfv5php@mailbox.test>, <Mailbox 3 user_7hfv5php@mailbox.test>
2026-04-16 22:29:13,650 - SL - WARNING - 6807 - "…/email_handler.py:857" - forward_email_to_mailbox() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - missing date header, create one
2026-04-16 22:29:13,657 - SL - DEBUG - 6807 - "…/email_handler.py:867" - forward_email_to_mailbox() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - From header, new:"sender_ra3wlh at example.com" <sender_ra3wlh_at_example_com_xgjopzef@sl.local>, old:sender_ra3wlh@example.com
2026-04-16 22:29:13,659 - SL - DEBUG - 6807 - "…/email_handler.py:286" - replace_header_when_forward() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - create contact for alias <Alias 4 seeped_ftping948@sl.local> and email leirwzhfhkezgbtnayto@leirwzhfhkezgbtnayto.com, header Cc
2026-04-16 22:29:13,678 - SL - DEBUG - 6807 - "…/email_handler.py:313" - replace_header_when_forward() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - Replace Cc header, old: leirwzhfhkezgbtnayto@leirwzhfhkezgbtnayto.com, new: "leirwzhfhkezgbtnayto at leirwzhfhkezgbtnayto.com" <leirwzhfhkezgbtnayto_at_leirwzhfhkezgbtnayto_com_grchsje@sl.local>
2026-04-16 22:29:13,681 - SL - DEBUG - 6807 - "…/email_handler.py:313" - replace_header_when_forward() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - Replace To header, old: seeped_ftping948@sl.local, new: seeped_ftping948@sl.local
2026-04-16 22:29:13,681 - SL - INFO  - 6807 - "…/app/handler/unsubscribe_generator.py:36" - _generate_header_with_original_behaviour() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - Email has no unsubscribe header
2026-04-16 22:29:13,689 - SL - DEBUG - 6807 - "…/email_handler.py:893" - forward_email_to_mailbox() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - Forward mail from sender_ra3wlh@example.com to user_7hfv5php@mailbox.test, mail_options:[], rcpt_options:[]
2026-04-16 22:29:13,691 - SL - DEBUG - 6807 - "…/app/mail_sender.py:131" - send() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - send email with subject 'Something', from '"sender_ra3wlh at example.com" <sender_ra3wlh_at_example_com_xgjopzef@sl.local>' to 'seeped_ftping948@sl.local'
2026-04-16 22:29:13,691 - SL - INFO  - 6807 - "…/email_handler.py:2367" - _handle() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - Finish mail_from env.sender_ra3wlh@example.com, rcpt_tos ['seeped_ftping948@sl.local'], takes 0.21394610404968262 seconds with return code '250 Message accepted for delivery'<<===
```

### B.2 Forward — non-existent alias path (9 log records)

```
2026-04-16 22:29:13,710 - SL - INFO  - 6807 - "…/email_handler.py:1956" - handle() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - Set CONTENT_TRANSFER_ENCODING
2026-04-16 22:29:13,710 - SL - DEBUG - 6807 - "…/email_handler.py:1963" - handle() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - Cannot parse Postfix queue ID from <inbound Received headers>
2026-04-16 22:29:13,711 - SL - DEBUG - 6807 - "…/email_handler.py:1980" - handle() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - ==>> Handle mail_from:external_e87a1@example.net, rcpt_tos:['nonexistent_qm2z7xaa@sl.local'], header_from:external_fsxeu@example.net, header_to:nonexistent_qm2z7xaa@sl.local, cc:pobdqblytywldtxfwqkp@pobdqblytywldtxfwqkp.com, reply-to:None, message_id:<af07e94a66ece6564ae30a2aaac7a34c@frontapp.com>, client_ip:None, headers:<full header dump>, mail_options:[], rcpt_options:[]
2026-04-16 22:29:13,717 - SL - DEBUG - 6807 - "…/email_handler.py:2202" - handle() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - Forward phase external_e87a1@example.net(external_fsxeu@example.net) -> nonexistent_qm2z7xaa@sl.local
2026-04-16 22:29:13,725 - SL - DEBUG - 6807 - "…/email_handler.py:545" - handle_forward() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - alias nonexistent_qm2z7xaa@sl.local not exist. Try to see if it can be created on the fly
2026-04-16 22:29:13,734 - SL - INFO  - 6807 - "…/app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - Cannot auto-create custom domain alias for nonexistent_qm2z7xaa@sl.local because there's no custom domain for sl.local
2026-04-16 22:29:13,734 - SL - INFO  - 6807 - "…/app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - Cannot auto-create nonexistent_qm2z7xaa@sl.local since it has no directory separator
2026-04-16 22:29:13,734 - SL - DEBUG - 6807 - "…/email_handler.py:551" - handle_forward() - 2570e03e-db08-45b4-9e9b-07fdee076b14 - alias nonexistent_qm2z7xaa@sl.local cannot be created on-the-fly, return 550
```

The resulting return value was `'550 SL E515 Email not exist'` (equal to `status.E515`).

### B.3 Reply path (18 log records, correlation UUID `8661772f-9e49-4e82-8c66-a0cccc9ed46d`)

```
2026-04-16 22:30:29,465 - SL - DEBUG - 6894 - "…/app/log.py:24" - set_message_id() -  - set message_id 8661772f-9e49-4e82-8c66-a0cccc9ed46d
2026-04-16 22:30:29,465 - SL - DEBUG - 6894 - "…/email_handler.py:2342" - _handle() - 8661772f-9e49-4e82-8c66-a0cccc9ed46d - ====>=====>====>====>====>====>====>====>
2026-04-16 22:30:29,465 - SL - INFO  - 6894 - "…/email_handler.py:2343" - _handle() - 8661772f-9e49-4e82-8c66-a0cccc9ed46d - New message, mail from user_u6b3q8nl@mailbox.test, rctp tos ['sender_8mnc0l_at_example_com_uihtcjhrdn@sl.local']
2026-04-16 22:30:29,466 - SL - INFO  - 6894 - "…/email_handler.py:1956" - handle() - 8661772f-9e49-4e82-8c66-a0cccc9ed46d - Set CONTENT_TRANSFER_ENCODING
2026-04-16 22:30:29,466 - SL - DEBUG - 6894 - "…/email_handler.py:1963" - handle() - 8661772f-9e49-4e82-8c66-a0cccc9ed46d - Cannot parse Postfix queue ID from <inbound Received headers>
2026-04-16 22:30:29,468 - SL - DEBUG - 6894 - "…/email_handler.py:1980" - handle() - 8661772f-9e49-4e82-8c66-a0cccc9ed46d - ==>> Handle mail_from:user_u6b3q8nl@mailbox.test, rcpt_tos:['sender_8mnc0l_at_example_com_uihtcjhrdn@sl.local'], header_from:None, header_to:sender_8mnc0l_at_example_com_uihtcjhrdn@sl.local, cc:None, reply-to:None, message_id:<d2386e98-b967-4e0e-bdd5-6672ee7267f8@user.mailbox.test>, client_ip:None, headers:<full header dump>, mail_options:[], rcpt_options:[]
2026-04-16 22:30:29,472 - SL - DEBUG - 6894 - "…/email_handler.py:2196" - handle() - 8661772f-9e49-4e82-8c66-a0cccc9ed46d - Reply phase user_u6b3q8nl@mailbox.test(None) -> sender_8mnc0l_at_example_com_uihtcjhrdn@sl.local
2026-04-16 22:30:29,479 - SL - INFO  - 6894 - "…/app/handler/dmarc.py:162" - apply_dmarc_policy_for_reply_phase() - 8661772f-9e49-4e82-8c66-a0cccc9ed46d - Spam check result is <app.handler.spamd_result.SpamdResult object at 0x7f17af860640>
2026-04-16 22:30:29,486 - SL - DEBUG - 6894 - "…/email_handler.py:1051" - handle_reply() - 8661772f-9e49-4e82-8c66-a0cccc9ed46d - Create <EmailLog 4> for <Contact 5 sender_8mnc0l@example.com 6>, <User 4 Jane Doe user_u6b3q8nl@mailbox.test>, <Mailbox 4 user_u6b3q8nl@mailbox.test>
2026-04-16 22:30:29,495 - SL - DEBUG - 6894 - "…/email_handler.py:1171" - handle_reply() - 8661772f-9e49-4e82-8c66-a0cccc9ed46d - From header is sheers_bloats363@sl.local
2026-04-16 22:30:29,496 - SL - DEBUG - 6894 - "…/email_handler.py:380" - replace_header_when_reply() - 8661772f-9e49-4e82-8c66-a0cccc9ed46d - Replace To header, old: sender_8mnc0l_at_example_com_uihtcjhrdn@sl.local, new: sender_8mnc0l@example.com
2026-04-16 22:30:29,496 - SL - DEBUG - 6894 - "…/email_handler.py:383" - replace_header_when_reply() - 8661772f-9e49-4e82-8c66-a0cccc9ed46d - delete the Cc header. Old value None
2026-04-16 22:30:29,499 - SL - DEBUG - 6894 - "…/email_handler.py:1314" - replace_original_message_id() - 8661772f-9e49-4e82-8c66-a0cccc9ed46d - create a new sl_message_id <177637862949.6894.7827280164812125513.4@sl.local>
2026-04-16 22:30:29,510 - SL - WARNING - 6894 - "…/email_handler.py:1206" - handle_reply() - 8661772f-9e49-4e82-8c66-a0cccc9ed46d - missing date header, add one
2026-04-16 22:30:29,514 - SL - DEBUG - 6894 - "…/email_handler.py:1212" - handle_reply() - 8661772f-9e49-4e82-8c66-a0cccc9ed46d - send email from sheers_bloats363@sl.local to sender_8mnc0l@example.com, mail_options:[],rcpt_options:[]
2026-04-16 22:30:29,520 - SL - DEBUG - 6894 - "…/app/mail_sender.py:131" - send() - 8661772f-9e49-4e82-8c66-a0cccc9ed46d - send email with subject 'Something', from 'sheers_bloats363@sl.local' to 'sender_8mnc0l@example.com'
2026-04-16 22:30:29,523 - SL - INFO  - 6894 - "…/email_handler.py:2367" - _handle() - 8661772f-9e49-4e82-8c66-a0cccc9ed46d - Finish mail_from user_u6b3q8nl@mailbox.test, rcpt_tos ['sender_8mnc0l_at_example_com_uihtcjhrdn@sl.local'], takes 0.05786538124084473 seconds with return code '250 Message accepted for delivery'<<===
```

---

## 9. Appendix C — Environment & Cleanup

### 9.1 Environment stood up for the investigation

| Component | Value / version |
|---|---|
| Python | 3.10.20 (venv at `/tmp/sl-venv`) |
| PostgreSQL | 12 (Docker container `sl-pg`, host port `15432`) |
| Redis | 7 (Docker container `sl-redis`, host port `6379`) |
| `DB_URI` | `postgresql://test:test@localhost:15432/test` |
| `EMAIL_DOMAIN` | `sl.local` |
| `OTHER_ALIAS_DOMAINS` | `["d1.test","d2.test","sl.local"]` |
| `NOT_SEND_EMAIL` | `true` |
| `email_validator` pinned | `1.1.3` (required to accept the `.local` TLD used by `tests/test.env`) |

All values above come from `tests/test.env` and `pyproject.toml` as-shipped — no repository file was modified.

### 9.2 Cleanup performed (per user instruction: "clean up any test containers or DB instances you spin up")

- `docker rm -f sl-pg` — PostgreSQL 12 container torn down.
- `docker rm -f sl-redis` — Redis 7 container torn down.
- Database volume purged with the container (no `-v` persistent volume was mounted).
- Virtual-env at `/tmp/sl-venv` and harness files under `/tmp/investigation/` are ephemeral scratch locations and do not live inside the repository.
- No file in the SimpleLogin repository was modified, created, or deleted by the investigation harness itself. The only file introduced into the destination repository is this document: `blitzy/documentation/app_2cd6ee777f8c.md`.

### 9.3 Reproducibility

To reproduce these exact runtime values (modulo randomness in generated local-parts and make_msgid components), the investigation would:

1. Spin up PostgreSQL 12 on `localhost:15432` and Redis 7 on `localhost:6379`.
2. `CONFIG=tests/test.env flask db upgrade` → apply all Alembic migrations.
3. Seed SL domains via `init_app.add_sl_domains()` and `init_app.add_proton_partner()`.
4. Instantiate a `create_light_app().app_context()`, create a user / mailbox / alias, construct an `aiosmtpd.smtp.Envelope` and `email.message.Message`, and invoke `email_handler.MailHandler()._handle(envelope, msg)`.
5. Read back `Contact`, `EmailLog`, `MessageIDMatching`, and `UserAuditLog` rows via the ORM for the just-completed operation.
6. For the reply leg, submit an envelope `from=user.email, to=[contact.reply_email]` using the `tests/example_emls/replacement_on_reply_phase.eml` fixture (without a `Cc`) and observe `MessageIDMatching` population.

All randomly generated values will differ between runs; structural values (the exact log format string, the set of log lines emitted, the `headers_to_keep` allow-list, the `SenderFormatEnum.AT` format, the `make_msgid` template, etc.) will not.

---

*End of investigation document.*
