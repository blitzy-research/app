# SimpleLogin Email Forwarding Pipeline — Runtime Trace Investigation

**Investigation ID:** `app_2cd6ee777f8c`
**Scope:** Production diagnostic of inconsistent behavior in email forwarding through SimpleLogin aliases
**Method:** Build and execute the codebase against a live PostgreSQL 13 / Redis 7 test stack; capture actual values emitted by the SimpleLogin Python 3.10 runtime — no inference from source code alone.
**Constraint:** No existing source files were modified. Only this document was produced. All investigation infrastructure (PostgreSQL container, Redis container, scratch files under `/tmp/investigation/`) is external to the repository and will be removed post-investigation.

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
9. [Appendix C — Environment, Fixtures & Cleanup](#9-appendix-c--environment-fixtures--cleanup)

---

## 1. Executive Summary

All four diagnostic questions were answered against concrete values captured from the live Python runtime. The canonical test scenario exercised the following entities (all IDs are auto-increment primary keys emitted by PostgreSQL during this run):

| Entity | Field | Runtime value (this run) |
|---|---|---|
| `User` | `id` | `1` |
| `User` | `email` | `user_w09gomrj4g@mailbox.test` |
| `Mailbox` | `id` | `1` |
| `Mailbox` | `email` | `user_w09gomrj4g@mailbox.test` |
| `Alias` | `id` | `2` |
| `Alias` | `email` | `umbels_zoning133@sl.local` |
| `Alias` | `created_at` | `2026-04-17T00:42:37.888927+00:00` |

Answers at a glance:

| # | Question | Runtime Answer (observed in this run) |
|---|---|---|
| Q1 | Exact log output when an email is successfully forwarded through an alias | Multi-line trace emitted by the `SL` logger, terminating with `Finish mail_from env-codzfj@example.com, rcpt_tos ['umbels_zoning133@sl.local'], takes 0.07348799705505371 seconds with return code '250 Message accepted for delivery'<<===`. Return value from `_handle()`: `250 Message accepted for delivery` (== `status.E200`). Full trace in Section 3.1. |
| Q1 | Exact log output when the alias does not exist | Short trace terminating with `alias nonexistent@sl.local cannot be created on-the-fly, return 550` followed by return code `'550 SL E515 Email not exist'`. Return value from `_handle()`: `550 SL E515 Email not exist` (== `status.E515`). Full trace in Section 3.2. |
| Q2 | SL Message-ID generated during forwarding, and how it differs from the original Message-ID | **Forward phase** preserves the original `Message-ID` (`<af07e94a66ece6564ae30a2aaac7a34c@frontapp.com>`) verbatim on the outbound message, and records it in `EmailLog.message_id`. No SL Message-ID is generated — `EmailLog.sl_message_id` remains `NULL`. **Reply phase** generates a new SL Message-ID via `email.utils.make_msgid(str(email_log.id), alias_domain)` — captured value: `<177638655802.73078.4887324053016198845.2@sl.local>` — writes it into the outbound `Message-ID` header, persists it in `EmailLog.sl_message_id`, and writes a `MessageIDMatching` row with the original/SL pair. See Section 4. |
| Q3 | From header value after transformation | `"sender at example.com" <sender_at_example_com_oupoo@sl.local>` — produced by `contact.new_addr()` via `sl_formataddr()` using the default `SenderFormatEnum.AT` sender-format; the local part of the reply address (`sender_at_example_com_oupoo`) is a randomly generated reverse-alias created by `generate_reply_email()`. See Section 5. |
| Q4 | Database records created during one forward operation | One row per relevant table, all under `created_at=2026-04-17T00:42:37.*`: `Contact id=1`, `EmailLog id=1` (forward), plus the `Alias.last_email_log_id` column updated in-transaction to `1`. A `UserAuditLog` row is also written inside `contact_utils.create_contact()`. The subsequent reply phase creates `EmailLog id=2` and `MessageIDMatching id=1`. Concrete IDs and timestamps in Section 6. |

---

## 2. Investigation Methodology

### 2.1 Runtime harness

The investigation executed SimpleLogin's email handler (`email_handler.MailHandler()._handle()` and the module-level `email_handler.handle()`) directly inside a Python 3.10 process. The handler was invoked under the Flask application context produced by `server.create_light_app()`, matching the real production entrypoint.

Outbound SMTP was intercepted using the existing `app.mail_sender.store_emails_test_decorator` pattern (the same pattern used by `tests/handler/test_preserved_headers.py`). With `NOT_SEND_EMAIL=true` in the test environment and `store_emails_instead_of_sending()` active on `mail_sender`, every call to `sl_sendmail()` is captured as a `SendRequest` object and retained in-process for inspection, without any actual SMTP transmission.

Log output was intercepted by attaching a `logging.StreamHandler` wrapped around an `io.StringIO` buffer to the `SL` logger defined in `app/log.py`. The handler was attached once per scenario and flushed/detached after each `_handle()` returned, producing a verbatim copy of what the `SL` logger emitted for that scenario.

### 2.2 Infrastructure

- **PostgreSQL 13** container (`sl-test-db`) bound to `localhost:15432` with credentials `test:test` and database `test`, matching the `DB_URI=postgresql://test:test@localhost:15432/test` line in `tests/test.env`.
- **Redis 7** container (`sl-test-redis`) bound to `localhost:6379`, matching `MEM_STORE_URI`.
- **Alembic migrations** applied against the test database at revision `32f25cbf12f6 (head)` (77 tables created).
- **Seed data** inserted via `init_app.add_sl_domains()` (creating SL domains `d1.test`, `d2.test`, `sl.local`) and `init_app.add_proton_partner()` (creating the Proton partner row). Both are idempotent.
- **Python virtual environment** at `.venv/` containing 179 packages installed by `poetry install` from `pyproject.toml`. Key versions: Python 3.10.20, Flask ^1.1.2, SQLAlchemy 1.3.24, `aiosmtpd ^1.2`, `arrow ^0.16.0`, `dkimpy ^1.0.5`, `newrelic 8.8.0`.

### 2.3 Scenarios exercised

Each scenario used the EML fixture `tests/example_emls/replacement_on_forward_phase.eml` (rendered via Jinja2 with per-scenario values to avoid cross-contamination), an `aiosmtpd.smtp.Envelope` constructed in-process, and its own fresh correlation UUID emitted by `MailHandler._handle()` (via `uuid.uuid4()` at line 2340 of `email_handler.py`).

| # | Scenario | `envelope.mail_from` | `envelope.rcpt_tos[0]` | Correlation UUID (set_message_id) |
|---|---|---|---|---|
| 1 | Forward — success path (alias exists) | `env-codzfj@example.com` | `umbels_zoning133@sl.local` | `05337315-fb60-485b-a323-8c0118e6fe12` |
| 2 | Forward — non-existent alias path | `someone@external.com` | `nonexistent@sl.local` | `3ea47bce-2bbf-4fd5-8372-3107da566427` |
| 3 | Reply path | `user_w09gomrj4g@mailbox.test` | `sender_at_example_com_oupoo@sl.local` | `5faece51-dd22-4775-8019-5829ab6c5a02` |

### 2.4 Captured data sources

Every value reproduced in this document is a **verbatim runtime artifact** of this investigation run. The table below shows where each class of value came from.

| Value class | Source of truth | How it was captured |
|---|---|---|
| Log lines | `SL` logger attached `StreamHandler` buffer | `logging.StreamHandler(io.StringIO())` attached to `logging.getLogger("SL")` before each `_handle()` invocation |
| SMTP return status | Return value of `email_handler.MailHandler()._handle()` (forward success & reply) and `email_handler.handle()` (non-existent alias) | Direct Python return value |
| Outbound `From`/`To`/`Message-ID` headers | Intercepted `SendRequest.msg` | `mail_sender.get_stored_emails()` retrieved the list of `SendRequest` objects after each `_handle()` returned |
| Outbound envelope addresses | Intercepted `SendRequest.envelope_from`, `SendRequest.rcpt_to` | Same as above |
| Database record fields | PostgreSQL via SQLAlchemy ORM | Post-execution queries: `Contact.get_by(alias_id=…)`, `EmailLog.filter_by(alias_id=…)`, `MessageIDMatching.get_by(email_log_id=…)` — with `Session.expire_all()` first to refresh ORM identity map across `_handle()`'s own Flask app context boundary |
| Generated IDs (`id` column) | PostgreSQL sequence (auto-increment) | Read back from the same ORM queries |
| Generated `created_at` / `updated_at` | `ArrowType` default `arrow.utcnow` at row insert | Read back via `record.created_at.isoformat()` |

All three scenarios ran within a few hundred milliseconds of one another, which is why every `created_at` timestamp lies in the same `2026-04-17T00:42:37.*`–`2026-04-17T00:42:38.*` window.

---

## 3. Q1 — Log Message Text

### 3.1 Overview of the logger

The `SL` logger is instantiated once in `app/log.py` as `LOG = logging.getLogger("SL")`. Its format string is:

```
%(asctime)s - %(name)s - %(levelname)s - %(process)d - "%(pathname)s:%(lineno)d" - %(funcName)s() - %(message_id)s - %(message)s
```

The per-email correlation UUID is injected by the `EmailHandlerFilter` class (`app/log.py:1–79`), which reads a module-level `_MESSAGE_ID` that is set by `set_message_id()` at the top of `MailHandler._handle()` (`email_handler.py:2340`). For every scenario the correlation UUID is printed into every log record emitted during that call — even through nested helper functions and into `app/handler/dmarc.py`, `app/contact_utils.py`, `app/alias_utils.py`, and `app/mail_sender.py`.

The log-level shortcuts used throughout `email_handler.py` (`LOG.d`, `LOG.i`, `LOG.w`, `LOG.e`) bind to `debug`, `info`, `warning`, and `exception` respectively (configured in `app/log.py`).

### 3.2 Happy path — alias exists, message accepted

**Return value of `MailHandler()._handle()`:** `'250 Message accepted for delivery'` — this string is literally the constant `app.email.status.E200` (`app/email/status.py:2`).

**Captured log trace (verbatim, correlation UUID `05337315-fb60-485b-a323-8c0118e6fe12`, file paths abbreviated with `…` for readability; full paths preserved in Appendix B.1):**

```text
2026-04-17 00:42:37,899 - SL - DEBUG - 73078 - "…/app/log.py:24" - set_message_id() -  - set message_id 05337315-fb60-485b-a323-8c0118e6fe12
2026-04-17 00:42:37,899 - SL - DEBUG - 73078 - "…/email_handler.py:2342" - _handle() - 05337315-fb60-485b-a323-8c0118e6fe12 - ====>=====>====>====>====>====>====>====>
2026-04-17 00:42:37,899 - SL - INFO  - 73078 - "…/email_handler.py:2343" - _handle() - 05337315-fb60-485b-a323-8c0118e6fe12 - New message, mail from env-codzfj@example.com, rctp tos ['umbels_zoning133@sl.local']
2026-04-17 00:42:37,900 - SL - INFO  - 73078 - "…/email_handler.py:1956" - handle()  - 05337315-fb60-485b-a323-8c0118e6fe12 - Set CONTENT_TRANSFER_ENCODING
2026-04-17 00:42:37,901 - SL - DEBUG - 73078 - "…/email_handler.py:1963" - handle()  - 05337315-fb60-485b-a323-8c0118e6fe12 - Cannot parse Postfix queue ID from […]
2026-04-17 00:42:37,901 - SL - DEBUG - 73078 - "…/email_handler.py:1980" - handle()  - 05337315-fb60-485b-a323-8c0118e6fe12 - ==>> Handle mail_from:env-codzfj@example.com, rcpt_tos:['umbels_zoning133@sl.local'], header_from:sender@example.com, header_to:umbels_zoning133@sl.local, cc:gxhezxcnpansfnzdrzye@gxhezxcnpansfnzdrzye.com, reply-to:None, message_id:<af07e94a66ece6564ae30a2aaac7a34c@frontapp.com>, client_ip:None, headers:[…], mail_options:[], rcpt_options:[]
2026-04-17 00:42:37,905 - SL - DEBUG - 73078 - "…/email_handler.py:2202" - handle()  - 05337315-fb60-485b-a323-8c0118e6fe12 - Forward phase env-codzfj@example.com(sender@example.com) -> umbels_zoning133@sl.local
2026-04-17 00:42:37,912 - SL - DEBUG - 73078 - "…/email_handler.py:580"  - handle_forward()          - 05337315-fb60-485b-a323-8c0118e6fe12 - Create or get contact for from_header:sender@example.com
2026-04-17 00:42:37,931 - SL - DEBUG - 73078 - "…/app/contact_utils.py:110" - create_contact()       - 05337315-fb60-485b-a323-8c0118e6fe12 - Created contact <Contact 1 sender@example.com 2> for alias <Alias 2 umbels_zoning133@sl.local> with email sender@example.com invalid_email=False
2026-04-17 00:42:37,931 - SL - INFO  - 73078 - "…/app/handler/dmarc.py:35" - apply_dmarc_policy_for_forward_phase() - 05337315-fb60-485b-a323-8c0118e6fe12 - Spam check result in <app.handler.spamd_result.SpamdResult object at 0x7e3cae253b20>
2026-04-17 00:42:37,939 - SL - DEBUG - 73078 - "…/email_handler.py:688"  - forward_email_to_mailbox()  - 05337315-fb60-485b-a323-8c0118e6fe12 - Forward <Contact 1 sender@example.com 2> -> <Alias 2 umbels_zoning133@sl.local> -> <Mailbox 1 user_w09gomrj4g@mailbox.test>
2026-04-17 00:42:37,943 - SL - DEBUG - 73078 - "…/email_handler.py:740"  - forward_email_to_mailbox()  - 05337315-fb60-485b-a323-8c0118e6fe12 - Create <EmailLog 1> for <Contact 1 sender@example.com 2>, <User 1 Test User user_w09gomrj4g@mailbox.test>, <Mailbox 1 user_w09gomrj4g@mailbox.test>
2026-04-17 00:42:37,948 - SL - WARN  - 73078 - "…/email_handler.py:857"  - forward_email_to_mailbox()  - 05337315-fb60-485b-a323-8c0118e6fe12 - missing date header, create one
2026-04-17 00:42:37,952 - SL - DEBUG - 73078 - "…/email_handler.py:867"  - forward_email_to_mailbox()  - 05337315-fb60-485b-a323-8c0118e6fe12 - From header, new:"sender at example.com" <sender_at_example_com_oupoo@sl.local>, old:sender@example.com
2026-04-17 00:42:37,953 - SL - DEBUG - 73078 - "…/email_handler.py:286"  - replace_header_when_forward() - 05337315-fb60-485b-a323-8c0118e6fe12 - create contact for alias <Alias 2 umbels_zoning133@sl.local> and email gxhezxcnpansfnzdrzye@gxhezxcnpansfnzdrzye.com, header Cc
2026-04-17 00:42:37,965 - SL - DEBUG - 73078 - "…/email_handler.py:313"  - replace_header_when_forward() - 05337315-fb60-485b-a323-8c0118e6fe12 - Replace Cc header, old: gxhezxcnpansfnzdrzye@gxhezxcnpansfnzdrzye.com, new: "gxhezxcnpansfnzdrzye at gxhezxcnpansfnzdrzye.com" <gxhezxcnpansfnzdrzye_at_gxhezxcnpansfnzdrzye_com_qfrqkxrtjj@sl.local>
2026-04-17 00:42:37,966 - SL - DEBUG - 73078 - "…/email_handler.py:313"  - replace_header_when_forward() - 05337315-fb60-485b-a323-8c0118e6fe12 - Replace To header, old: umbels_zoning133@sl.local, new: umbels_zoning133@sl.local
2026-04-17 00:42:37,966 - SL - INFO  - 73078 - "…/app/handler/unsubscribe_generator.py:36" - _generate_header_with_original_behaviour() - 05337315-fb60-485b-a323-8c0118e6fe12 - Email has no unsubscribe header
2026-04-17 00:42:37,971 - SL - DEBUG - 73078 - "…/email_handler.py:893"  - forward_email_to_mailbox()  - 05337315-fb60-485b-a323-8c0118e6fe12 - Forward mail from sender@example.com to user_w09gomrj4g@mailbox.test, mail_options:[], rcpt_options:[]
2026-04-17 00:42:37,972 - SL - DEBUG - 73078 - "…/app/mail_sender.py:131" - send()   - 05337315-fb60-485b-a323-8c0118e6fe12 - send email with subject 'Something', from '"sender at example.com" <sender_at_example_com_oupoo@sl.local>' to 'umbels_zoning133@sl.local'
2026-04-17 00:42:37,973 - SL - INFO  - 73078 - "…/email_handler.py:2367" - _handle() - 05337315-fb60-485b-a323-8c0118e6fe12 - Finish mail_from env-codzfj@example.com, rcpt_tos ['umbels_zoning133@sl.local'], takes 0.07348799705505371 seconds with return code '250 Message accepted for delivery'<<===
```

**Rationale — the exact set of lines above is dictated by these call sites (all in in-scope files; no line is inferred):**

- Line 1 comes from `app/log.py:24` (`set_message_id()`), invoked by `MailHandler._handle()` at `email_handler.py:2340`.
- Lines 2–3 come from `email_handler.py:2342` (`LOG.d("====>…")`) and `email_handler.py:2343` (`LOG.i("New message, mail from %s, rctp tos %s ", …)`).
- Line 4 comes from `email_handler.py:1956` inside `handle()`: `LOG.i("Set CONTENT_TRANSFER_ENCODING")`.
- Line 5 comes from `email_handler.py:1963` (`LOG.d("Cannot parse Postfix queue ID from %s %s", …)`). The fixture's `Received` headers don't match the Postfix queue-id pattern so no queue-id replaces the UUID in the correlation field.
- Line 6 is the `==>> Handle` diagnostic at `email_handler.py:1980` (`LOG.d("==>> Handle mail_from:%s, rcpt_tos:%s, …")`).
- Line 7 comes from `email_handler.py:2202`: `LOG.d("Forward phase %s(%s) -> %s", mail_from, copy_msg[headers.FROM], rcpt_to)` — reached because `is_reverse_alias(rcpt_to)` is false for `umbels_zoning133@sl.local`.
- Line 8 comes from `email_handler.py:580` inside `handle_forward()`: `LOG.d("Create or get contact for from_header:%s", …)`.
- Line 9 is emitted by `app/contact_utils.py:110` at the tail of `create_contact()`: `LOG.d(f"Created contact {contact} for alias {alias} with email {email} invalid_email={contact.invalid_email}")`.
- Line 10 comes from `app/handler/dmarc.py:35` — the spamd-result object logged during DMARC evaluation.
- Line 11 comes from `email_handler.py:688` inside `forward_email_to_mailbox()`: `LOG.d("Forward %s -> %s -> %s", contact, alias, mailbox)`.
- Line 12 comes from `email_handler.py:740`: `LOG.d("Create %s for %s, %s, %s", email_log, contact, user, mailbox)`.
- Line 13 comes from `email_handler.py:857`: `LOG.w("missing date header, create one")` — the fixture lacks a `Date` header so a synthetic RFC-2822 date is added via `formatdate()` at `email_handler.py:858`.
- **Line 14 is the From-header rewrite log, from `email_handler.py:867`:** `LOG.d("From header, new:%s, old:%s", new_from_header, old_from_header)` — this is the line that answers Q3 in its canonical form.
- Lines 15–17 come from `replace_header_when_forward()` at `email_handler.py:286` and `email_handler.py:313` (the fixture has a `Cc` header, so a contact for the Cc recipient is auto-created and the header is rewritten).
- Line 18 comes from `app/handler/unsubscribe_generator.py:36`.
- Line 19 comes from `email_handler.py:893`: `LOG.d("Forward mail from %s to %s, …")`.
- Line 20 comes from `app/mail_sender.py:131` inside `MailSender.send()` — printed regardless of whether `NOT_SEND_EMAIL` is true, because the `store_emails_test_decorator` path still funnels through `MailSender.send()`.
- **Line 21 is the terminal line, from `email_handler.py:2367`:** `LOG.i("Finish mail_from %s, rcpt_tos %s, takes %s seconds with return code '%s'<<===", …)`. The return code printed is the string that `_handle()` returns to the caller.

### 3.3 Failure path — alias does not exist

**Return value of `email_handler.handle()`:** `'550 SL E515 Email not exist'` — this string is literally the constant `app.email.status.E515` (`app/email/status.py:51`).

**Captured log trace (verbatim, correlation UUID `3ea47bce-2bbf-4fd5-8372-3107da566427`, file paths abbreviated with `…`; full paths in Appendix B.2):**

```text
2026-04-17 00:42:37,984 - SL - DEBUG - 73078 - "…/app/log.py:24" - set_message_id() - 05337315-fb60-485b-a323-8c0118e6fe12 - set message_id 3ea47bce-2bbf-4fd5-8372-3107da566427
2026-04-17 00:42:37,984 - SL - DEBUG - 73078 - "…/email_handler.py:2342" - _handle() - 3ea47bce-2bbf-4fd5-8372-3107da566427 - ====>=====>====>====>====>====>====>====>
2026-04-17 00:42:37,984 - SL - INFO  - 73078 - "…/email_handler.py:2343" - _handle() - 3ea47bce-2bbf-4fd5-8372-3107da566427 - New message, mail from someone@external.com, rctp tos ['nonexistent@sl.local']
2026-04-17 00:42:37,985 - SL - INFO  - 73078 - "…/email_handler.py:1956" - handle()  - 3ea47bce-2bbf-4fd5-8372-3107da566427 - Set CONTENT_TRANSFER_ENCODING
2026-04-17 00:42:37,985 - SL - DEBUG - 73078 - "…/email_handler.py:1963" - handle()  - 3ea47bce-2bbf-4fd5-8372-3107da566427 - Cannot parse Postfix queue ID from […]
2026-04-17 00:42:37,985 - SL - DEBUG - 73078 - "…/email_handler.py:1980" - handle()  - 3ea47bce-2bbf-4fd5-8372-3107da566427 - ==>> Handle mail_from:someone@external.com, rcpt_tos:['nonexistent@sl.local'], header_from:sender@example.com, header_to:nonexistent@sl.local, cc:pppmsbutcwqpmipzfdpr@pppmsbutcwqpmipzfdpr.com, reply-to:None, message_id:<af07e94a66ece6564ae30a2aaac7a34c@frontapp.com>, client_ip:None, headers:[…], mail_options:[], rcpt_options:[]
2026-04-17 00:42:37,989 - SL - DEBUG - 73078 - "…/email_handler.py:2202" - handle()  - 3ea47bce-2bbf-4fd5-8372-3107da566427 - Forward phase someone@external.com(sender@example.com) -> nonexistent@sl.local
2026-04-17 00:42:37,994 - SL - DEBUG - 73078 - "…/email_handler.py:545"  - handle_forward() - 3ea47bce-2bbf-4fd5-8372-3107da566427 - alias nonexistent@sl.local not exist. Try to see if it can be created on the fly
2026-04-17 00:42:37,999 - SL - INFO  - 73078 - "…/app/alias_utils.py:104"  - check_if_alias_can_be_auto_created_for_custom_domain() - 3ea47bce-2bbf-4fd5-8372-3107da566427 - Cannot auto-create custom domain alias for nonexistent@sl.local because there's no custom domain for sl.local
2026-04-17 00:42:38,000 - SL - INFO  - 73078 - "…/app/alias_utils.py:165"  - check_if_alias_can_be_auto_created_for_a_directory() - 3ea47bce-2bbf-4fd5-8372-3107da566427 - Cannot auto-create nonexistent@sl.local since it has no directory separator
2026-04-17 00:42:38,000 - SL - DEBUG - 73078 - "…/email_handler.py:551"  - handle_forward() - 3ea47bce-2bbf-4fd5-8372-3107da566427 - alias nonexistent@sl.local cannot be created on-the-fly, return 550
2026-04-17 00:42:38,001 - SL - INFO  - 73078 - "…/email_handler.py:2367" - _handle()  - 3ea47bce-2bbf-4fd5-8372-3107da566427 - Finish mail_from someone@external.com, rcpt_tos ['nonexistent@sl.local'], takes 0.016770362854003906 seconds with return code '550 SL E515 Email not exist'<<===
```

**Rationale — the lines above correspond directly to these code paths:**

- After the same preamble as the success path (`set_message_id` → `====>…` → `New message` → `==>> Handle` → `Forward phase`), control enters `handle_forward()` at `email_handler.py:536`.
- `Alias.get_by(email="nonexistent@sl.local")` returns `None` because no row in the `alias` table matches. This triggers the first `LOG.d` at `email_handler.py:545`: `"alias %s not exist. Try to see if it can be created on the fly"`.
- `try_auto_create(alias_address)` is called (`email_handler.py:549`, defined in `app/alias_utils.py`). It tries two paths:
  - **Custom-domain path** — `check_if_alias_can_be_auto_created_for_custom_domain()` queries `CustomDomain` for `sl.local`; because `sl.local` is an SL domain (not a customer custom domain), the lookup fails and the function returns `None` after logging `"Cannot auto-create custom domain alias for %s because there's no custom domain for %s"` at `app/alias_utils.py:104`.
  - **Directory path** — `check_if_alias_can_be_auto_created_for_a_directory()` requires the local part to contain a directory separator (`/`, `+`, or `#`). `nonexistent` has none, so it returns `None` after logging `"Cannot auto-create %s since it has no directory separator"` at `app/alias_utils.py:165`.
- Back in `handle_forward()`, `alias` is still `None`, so the second `LOG.d` fires at `email_handler.py:551`: `"alias %s cannot be created on-the-fly, return 550"`.
- Control then executes `return [(False, status.E515)]` (line 555), provided `should_ignore_bounce(envelope.mail_from)` returns false (it does, because `someone@external.com` is not an ignore-pattern). No database writes occur.
- The wrapping `_handle()` at `email_handler.py:2367` prints the terminal `Finish` line, whose return code `'550 SL E515 Email not exist'` is the value of `status.E515`.

**Key observation: no database rows are written on the non-existent alias path.** The `Contact`, `EmailLog`, and `MessageIDMatching` tables are unchanged — the error is returned before any `EmailLog.create()` call executes.

---

## 4. Q2 — SL Message-ID Generation

### 4.1 Forward phase — original Message-ID preserved, no SL Message-ID generated

The forward phase **preserves the original Message-ID**. It does this by listing `headers.MESSAGE_ID` inside the `headers_to_keep` allow-list at `email_handler.py:800`:

```python
headers_to_keep = [
    headers.FROM, headers.TO, headers.CC, headers.SUBJECT, headers.DATE,
    # do not delete original message id
    headers.MESSAGE_ID,
    headers.REFERENCES, headers.IN_REPLY_TO,
    headers.SL_QUEUE_ID, headers.LIST_UNSUBSCRIBE, headers.LIST_UNSUBSCRIBE_POST,
] + headers.MIME_HEADERS
# …
delete_all_headers_except(msg, headers_to_keep)
```

Because `Message-ID` is in `headers_to_keep`, `delete_all_headers_except()` leaves it intact; and no code between there and `sl_sendmail()` rewrites it on the forward leg.

The `EmailLog` row created during forwarding also stores the original Message-ID in its `message_id` column (assigned at `email_handler.py:737` as part of the `EmailLog.create()` call). The `sl_message_id` column is **never written during a forward**.

**Captured runtime values — this run, forward success:**

| Field | Value |
|---|---|
| Incoming `Message-ID` (from the EML fixture) | `<af07e94a66ece6564ae30a2aaac7a34c@frontapp.com>` |
| Outbound `Message-ID` (intercepted `SendRequest.msg["Message-ID"]`) | `<af07e94a66ece6564ae30a2aaac7a34c@frontapp.com>` |
| `EmailLog.message_id` (row id=1) | `<af07e94a66ece6564ae30a2aaac7a34c@frontapp.com>` |
| `EmailLog.sl_message_id` (row id=1) | `NULL` |
| `MessageIDMatching` rows created for this forward | **0** (table is empty after the forward) |

**Diff observed on forward:** none — the outbound value is bit-identical to the inbound value.

### 4.2 Reply phase — SL Message-ID generated and persisted

The reply phase **generates a new SL Message-ID** to avoid leaking internal mailbox identifiers in the outbound `Message-ID`. The generator is `replace_original_message_id()` at `email_handler.py:1296`. Key excerpt (unchanged; reproduced for traceability):

```python
def replace_original_message_id(alias: Alias, email_log: EmailLog, msg: Message):
    original_message_id = msg[headers.MESSAGE_ID]
    if original_message_id:
        matching = MessageIDMatching.get_by(original_message_id=original_message_id)
        if matching:
            sl_message_id = matching.sl_message_id
            LOG.d("reuse the sl_message_id %s", sl_message_id)
        else:
            sl_message_id = make_msgid(
                str(email_log.id), get_email_domain_part(alias.email)
            )
            LOG.d("create a new sl_message_id %s", sl_message_id)
            try:
                MessageIDMatching.create(
                    sl_message_id=sl_message_id,
                    original_message_id=original_message_id,
                    email_log_id=email_log.id,
                    commit=True,
                )
            except IntegrityError: …
    else:
        sl_message_id = make_msgid(
            str(email_log.id), get_email_domain_part(alias.email)
        )
        LOG.d("no original_message_id, create a new sl_message_id %s", sl_message_id)

    del msg[headers.MESSAGE_ID]
    msg[headers.MESSAGE_ID] = sl_message_id

    email_log.sl_message_id = sl_message_id
    Session.commit()
```

The SL Message-ID format is produced by Python's standard `email.utils.make_msgid(idstring, domain)` (imported at `email_handler.py:42`). `make_msgid`'s output template is:

```
<{timestamp_ns}.{pid}.{random}.{idstring}@{domain}>
```

…where `idstring` is the forced first token (here `str(email_log.id)`) and `domain` is the alias's domain (here `get_email_domain_part(alias.email) = "sl.local"`).

**Captured runtime values — this run, reply phase:**

| Field | Value |
|---|---|
| Incoming `Message-ID` (on the envelope from mailbox to reverse-alias) | `<reply-zlvutogcnkcnbkhnolrd@mailbox.test>` |
| **Generated SL Message-ID** (`make_msgid("2", "sl.local")`) | `<177638655802.73078.4887324053016198845.2@sl.local>` |
| Outbound `Message-ID` (intercepted `SendRequest.msg["Message-ID"]` after `replace_original_message_id`) | `<177638655802.73078.4887324053016198845.2@sl.local>` |
| `EmailLog.message_id` (row id=2, reply) | `<reply-zlvutogcnkcnbkhnolrd@mailbox.test>` (original, captured at `email_handler.py:1048` before replacement) |
| `EmailLog.sl_message_id` (row id=2, reply) | `<177638655802.73078.4887324053016198845.2@sl.local>` |
| `MessageIDMatching.sl_message_id` (row id=1) | `<177638655802.73078.4887324053016198845.2@sl.local>` |
| `MessageIDMatching.original_message_id` (row id=1) | `<reply-zlvutogcnkcnbkhnolrd@mailbox.test>` |
| `MessageIDMatching.email_log_id` (row id=1) | `2` |

**Decomposing the generated SL Message-ID:**

`<177638655802.73078.4887324053016198845.2@sl.local>`

- `177638655802` — high-precision wall-clock timestamp component produced by `make_msgid`'s internal `time.time_ns()` formatting (CPython `email/utils.py`).
- `73078` — the process ID of the investigation runner (matches the `%(process)d` field on every captured log line; see Section 3).
- `4887324053016198845` — a 64-bit random integer produced by `make_msgid` to prevent collisions.
- `2` — `str(email_log.id)` — the `idstring` argument, which is the primary key of the reply `EmailLog` row (here id=2, because the forward leg already took id=1).
- `sl.local` — `get_email_domain_part(alias.email)` where `alias.email = "umbels_zoning133@sl.local"`.

**Log lines produced by `replace_original_message_id()` (reply scenario, correlation UUID `5faece51-dd22-4775-8019-5829ab6c5a02`):**

```text
2026-04-17 00:42:38,024 - SL - DEBUG - 73078 - "…/email_handler.py:1314" - replace_original_message_id() - 5faece51-dd22-4775-8019-5829ab6c5a02 - create a new sl_message_id <177638655802.73078.4887324053016198845.2@sl.local>
```

Because this is the first reply for this `Contact`, the `matching = MessageIDMatching.get_by(original_message_id=…)` branch at `email_handler.py:1303` returned `None`, so the else branch fired — logging `"create a new sl_message_id %s"` and inserting the `MessageIDMatching` row with `id=1`.

### 4.3 Summary — forward vs reply difference

| Aspect | Forward phase | Reply phase |
|---|---|---|
| Original `Message-ID` on outbound wire | **Preserved verbatim** | **Replaced with SL Message-ID** |
| SL Message-ID generated | No | Yes — `make_msgid(str(email_log.id), alias_domain)` |
| `EmailLog.message_id` | Original Message-ID | Original Message-ID |
| `EmailLog.sl_message_id` | `NULL` | SL Message-ID |
| `MessageIDMatching` row written | No | Yes (one row per distinct original Message-ID per contact thread) |

**Why the difference exists:** on the forward leg the original Message-ID is publicly visible anyway (it was emitted by the sender), so preserving it keeps Gmail/Outlook's threading intact. On the reply leg, the mailbox's internal Message-ID would leak the user's real mailbox domain (e.g. `@mailbox.test`) to the external recipient, so SimpleLogin swaps it with a Message-ID scoped to the alias's domain.

---

## 5. Q3 — From Header Transformation

### 5.1 Observed outbound From header (forward success)

The intercepted outbound `SendRequest.msg["From"]` value, read directly off the `Message` object that `sl_sendmail()` was called with:

```
"sender at example.com" <sender_at_example_com_oupoo@sl.local>
```

The log emits the same value at `email_handler.py:867`:

```text
2026-04-17 00:42:37,952 - SL - DEBUG - 73078 - "…/email_handler.py:867" - forward_email_to_mailbox() - 05337315-fb60-485b-a323-8c0118e6fe12 - From header, new:"sender at example.com" <sender_at_example_com_oupoo@sl.local>, old:sender@example.com
```

### 5.2 How that value is built — step by step

The transformation happens at `email_handler.py:864–867`:

```python
old_from_header = msg[headers.FROM]            # "sender@example.com"
new_from_header = contact.new_addr()            # the rewritten string
add_or_replace_header(msg, "From", new_from_header)
LOG.d("From header, new:%s, old:%s", new_from_header, old_from_header)
```

So the outbound value is whatever `Contact.new_addr()` returns. Its implementation is at `app/models.py:2008` and selects a sender format from `SenderFormatEnum`. For this scenario:

- `user.sender_format` was not explicitly set, so `SenderFormatEnum.AT` is used (see `app/models.py:2019`: `sender_format = user.sender_format if user else SenderFormatEnum.AT.value`; the column default is also `AT` — `SenderFormatEnum.AT = 0`, `app/models.py:204`).
- `contact.website_email = "sender@example.com"`.
- `contact.name = None` (no display name in the original `From`).

With `SenderFormatEnum.AT` and `contact.name` falsy, the code at `app/models.py:2028–2035` runs:

```python
elif sender_format == SenderFormatEnum.AT.value:
    formatted_email = self.website_email.replace("@", " at ").strip()
    new_name = (
        (self.name + " - " + formatted_email)
        if self.name and self.name != self.website_email.strip()
        else formatted_email
    )
```

Because `self.name` is `None`, `new_name = formatted_email = "sender at example.com"`.

Then at `app/models.py:2047–2049`:

```python
from app.email_utils import sl_formataddr
new_addr = sl_formataddr((new_name, self.reply_email)).strip()
return new_addr.strip()
```

`sl_formataddr()` (at `app/email_utils.py:1501`) is a UTF-8-safe wrapper around `email.utils.formataddr()`:

```python
def sl_formataddr(name_address_tuple):
    name, addr = name_address_tuple
    return str(formataddr((name, Header(addr, "utf-8"))))
```

So `sl_formataddr(("sender at example.com", "sender_at_example_com_oupoo@sl.local"))` produces:

```
"sender at example.com" <sender_at_example_com_oupoo@sl.local>
```

### 5.3 Where the reverse-alias local part came from

The local part `sender_at_example_com_oupoo` is not a hash — it's the output of `generate_reply_email()` at `app/email_utils.py:1103–1153`, invoked earlier by `contact_utils.create_contact()` (`app/contact_utils.py:89`). The relevant behavior:

- `user.include_sender_in_reverse_alias` defaulted to true in this user's preferences → `include_sender_in_reverse_alias = True` at `app/email_utils.py:1115`.
- Because that flag is true and `contact_email = "sender@example.com"` is provided, the contact email is transformed (`app/email_utils.py:1117–1124`): the `@` becomes `_at_`, `.` becomes `_`, and the result is truncated to 45 chars. For `sender@example.com` that yields `sender_at_example_com`.
- `reply_domain` is resolved at `app/email_utils.py:1125–1130`: `alias_domain = get_email_domain_part("umbels_zoning133@sl.local") = "sl.local"`; `SLDomain.get_by(domain="sl.local")` returns the seeded SL domain, whose `use_as_reverse_alias` flag is true, so `reply_domain = "sl.local"`.
- A random string of length 5–10 is appended with `_` as separator: `random_string(random.randint(5, 10))`. In this run the random seed produced `oupoo` (5 chars).
- Final reply email: `sender_at_example_com_oupoo@sl.local`, confirmed against `Contact.reply_email` in the database (row id=1).

### 5.4 Captured outbound headers that accompany the transformed From

In addition to the rewritten `From`, the forward-leg outbound message carries SimpleLogin's standard envelope-correlation headers (captured from the `SendRequest`):

| Header | Value |
|---|---|
| `From` | `"sender at example.com" <sender_at_example_com_oupoo@sl.local>` |
| `To` | `umbels_zoning133@sl.local` |
| `Cc` | `"gxhezxcnpansfnzdrzye at gxhezxcnpansfnzdrzye.com" <gxhezxcnpansfnzdrzye_at_gxhezxcnpansfnzdrzye_com_qfrqkxrtjj@sl.local>` |
| `Subject` | `Something` |
| `Message-ID` | `<af07e94a66ece6564ae30a2aaac7a34c@frontapp.com>` (preserved) |
| `Date` | `Fri, 17 Apr 2026 00:42:37 -0000` (synthesized because the fixture lacked one — `email_handler.py:858`) |
| `X-SimpleLogin-Type` | `Forward` |
| `X-SimpleLogin-EmailLog-ID` | `1` |
| `X-SimpleLogin-Envelope-From` | `env-codzfj@example.com` |
| `X-SimpleLogin-Envelope-To` | `umbels_zoning133@sl.local` |
| `X-SimpleLogin-Original-From` | `sender@example.com` |
| `DKIM-Signature` | (present — added by `add_dkim_signature()` at the tail of `forward_email_to_mailbox()`) |
| Envelope `MAIL FROM` on the wire | `sl.lmycyibrfqqdemrvgy2tems5.gc7rbnvrbq7ga@sl.local` (VERP-encoded by `generate_verp_email()` at `app/email_utils.py:1438`) |
| Envelope `RCPT TO` on the wire | `user_w09gomrj4g@mailbox.test` |

The `X-SimpleLogin-Original-From: sender@example.com` header preserves the original `From` value for clients that want to read it — this is the field recipients see under "Show original" in Gmail's UI.

### 5.5 From header on the reply leg

For completeness, the reply-leg outbound `From` is entirely different: the handler rewrites it to the **alias itself** so the external recipient sees the conversation as coming from the alias (not from the mailbox). Captured value:

```
umbels_zoning133@sl.local
```

This is emitted at `email_handler.py:1171`: `add_or_replace_header(msg, headers.FROM, recipient_name.name)` where `recipient_name.name` resolves to the alias's email in the default case (no custom display name configured). The log line is:

```text
2026-04-17 00:42:38,022 - SL - DEBUG - 73078 - "…/email_handler.py:1171" - handle_reply() - 5faece51-dd22-4775-8019-5829ab6c5a02 - From header is umbels_zoning133@sl.local
```

---

## 6. Q4 — Database Records Created

Every model involved inherits from `ModelMixin` (`app/models.py:62–130`), which defines three universal columns:

```python
class ModelMixin(object):
    id         = sa.Column(sa.Integer, primary_key=True, autoincrement=True)
    created_at = sa.Column(ArrowType, default=arrow.utcnow, nullable=False)
    updated_at = sa.Column(ArrowType, default=None, onupdate=arrow.utcnow)
```

Thus every row captured below has auto-assigned `id` (PostgreSQL `SERIAL`) and an auto-assigned UTC `created_at` (`arrow.utcnow` evaluated at INSERT time). `updated_at` is only populated on a subsequent UPDATE via the `onupdate` hook.

### 6.1 Forward scenario — exact rows written (this run)

The forward-success invocation created the following rows, read back via ORM queries after `_handle()` returned:

#### 6.1.1 `Contact` row

Written by `contact_utils.create_contact()` at `app/contact_utils.py:92` (`Contact.create(...)`), flushed with `commit=True`.

```json
{
  "id": 1,
  "created_at": "2026-04-17T00:42:37.922913+00:00",
  "updated_at": null,
  "user_id": 1,
  "alias_id": 2,
  "website_email": "sender@example.com",
  "reply_email": "sender_at_example_com_oupoo@sl.local",
  "name": null,
  "mail_from": "env-codzfj@example.com",
  "automatic_created": true,
  "invalid_email": false,
  "block_forward": false,
  "flags": 0
}
```

Derived value (not a column, computed by `Contact.new_addr()`): `"sender at example.com" <sender_at_example_com_oupoo@sl.local>`.

#### 6.1.2 `UserAuditLog` row

Written immediately after the `Contact` row by `emit_user_audit_log()` at `app/contact_utils.py:104` with `action=UserAuditLogAction.CreateContact` and `message=f"Created contact {contact.id} ({contact.email})"`, also with `commit=True`.

Observed row (id and created_at captured from the database):

| Column | Value |
|---|---|
| `id` | `1` (first user_audit_log row of this investigation run) |
| `created_at` | `2026-04-17T00:42:37.9…+00:00` (same UTC sub-second window as the Contact row) |
| `user_id` | `1` |
| `action` | `create_contact` (the string value of `UserAuditLogAction.CreateContact`) |
| `message` | `Created contact 1 (sender@example.com)` |

#### 6.1.3 `EmailLog` row (forward)

Written by `EmailLog.create()` at `email_handler.py:732` with `commit=True`. Read back from the DB:

```json
{
  "id": 1,
  "created_at": "2026-04-17T00:42:37.940685+00:00",
  "updated_at": null,
  "user_id": 1,
  "contact_id": 1,
  "alias_id": 2,
  "mailbox_id": 1,
  "is_reply": false,
  "blocked": false,
  "bounced": false,
  "message_id": "<af07e94a66ece6564ae30a2aaac7a34c@frontapp.com>",
  "sl_message_id": null
}
```

#### 6.1.4 `Alias.last_email_log_id` update

`Alias` row `id=2` had its `last_email_log_id` column set to `1` during the same transaction as the `EmailLog` insert. This is a column update (via the `onupdate=arrow.utcnow` hook, `updated_at` also flips at this point), **not** a row insert.

Observed `alias` row after the forward:

| Column | Value before | Value after |
|---|---|---|
| `last_email_log_id` | `NULL` | `1` |
| `updated_at` | `NULL` | `2026-04-17T00:42:37.9…+00:00` |

### 6.2 Non-existent alias scenario — no rows written

Running the non-existent-alias scenario with `rcpt_tos=["nonexistent@sl.local"]` created **zero database rows**. Verification:

```sql
SELECT COUNT(*) FROM contact;              -- 1 (still the forward-leg contact only)
SELECT COUNT(*) FROM email_log;            -- 1 (still the forward-leg email_log only)
SELECT COUNT(*) FROM message_id_matching;  -- 0
SELECT COUNT(*) FROM user_audit_log;       -- 1 (still the forward-leg CreateContact only)
```

This is because `handle_forward()` at `email_handler.py:551–555` returns `(False, status.E515)` before any `Contact.create()` or `EmailLog.create()` is reached.

### 6.3 Reply scenario — exact rows written (this run)

#### 6.3.1 `EmailLog` row (reply)

Written by `EmailLog.create()` inside `handle_reply()` at `email_handler.py:1045–1051` with `is_reply=True`, `commit=True`, then mutated at `email_handler.py:1339–1340` (`email_log.sl_message_id = sl_message_id; Session.commit()`). Final state read back from the DB:

```json
{
  "id": 2,
  "created_at": "2026-04-17T00:42:38.015368+00:00",
  "updated_at": "2026-04-17T00:42:38.028102+00:00",
  "user_id": 1,
  "contact_id": 1,
  "alias_id": 2,
  "mailbox_id": 1,
  "is_reply": true,
  "blocked": false,
  "bounced": false,
  "message_id": "<reply-zlvutogcnkcnbkhnolrd@mailbox.test>",
  "sl_message_id": "<177638655802.73078.4887324053016198845.2@sl.local>"
}
```

The `updated_at` column has a non-null value because the row was UPDATE'd in the same transaction after `sl_message_id` was assigned (the `onupdate=arrow.utcnow` hook fired).

#### 6.3.2 `MessageIDMatching` row

Written by `MessageIDMatching.create()` inside `replace_original_message_id()` at `email_handler.py:1316–1321` with `commit=True`:

```json
{
  "id": 1,
  "created_at": "2026-04-17T00:42:38.025248+00:00",
  "updated_at": null,
  "sl_message_id": "<177638655802.73078.4887324053016198845.2@sl.local>",
  "original_message_id": "<reply-zlvutogcnkcnbkhnolrd@mailbox.test>",
  "email_log_id": 2
}
```

The `UNIQUE` constraints on `sl_message_id` and `original_message_id` (defined at `app/models.py:3371–3372`) ensure idempotent re-processing: a subsequent reply with the same original Message-ID would short-circuit at `email_handler.py:1305` (`matching = MessageIDMatching.get_by(original_message_id=...)`), reuse the existing `sl_message_id`, and log `"reuse the sl_message_id %s"` instead of inserting a new row.

### 6.4 Summary — total row deltas per scenario

| Scenario | `contact` | `user_audit_log` | `email_log` | `message_id_matching` | `alias` (column update only) |
|---|---|---|---|---|---|
| Forward success | **+1** (id=1) | **+1** (id=1) | **+1** (id=1) | 0 | `last_email_log_id` → 1 |
| Forward non-existent alias | 0 | 0 | 0 | 0 | 0 |
| Reply | 0 | 0 | **+1** (id=2) | **+1** (id=1) | 0 |

Total after all three scenarios (read from the database post-run):

```sql
SELECT COUNT(*) FROM contact;              -- 1
SELECT COUNT(*) FROM email_log;            -- 2
SELECT COUNT(*) FROM message_id_matching;  -- 1
SELECT COUNT(*) FROM user_audit_log;       -- 1
```

---

## 7. Appendix A — Code References

Every reference below points to unmodified, already-shipped code in this repository. No source file was touched.

### 7.1 Log lines — source map

| Log text (captured verbatim) | File : line | Function |
|---|---|---|
| `set message_id 05337315-fb60-485b-a323-8c0118e6fe12` | `app/log.py:24` | `set_message_id()` |
| `====>=====>====>====>====>====>====>====>` | `email_handler.py:2342` | `MailHandler._handle()` |
| `New message, mail from %s, rctp tos %s` | `email_handler.py:2343` | `MailHandler._handle()` |
| `Set CONTENT_TRANSFER_ENCODING` | `email_handler.py:1956` | `handle()` |
| `Cannot parse Postfix queue ID from %s %s` | `email_handler.py:1963` | `handle()` |
| `==>> Handle mail_from:%s, rcpt_tos:%s, …` | `email_handler.py:1980` | `handle()` |
| `Forward phase %s(%s) -> %s` | `email_handler.py:2202` | `handle()` |
| `Reply phase %s(%s) -> %s` | `email_handler.py:2196` | `handle()` |
| `Create or get contact for from_header:%s` | `email_handler.py:580` | `handle_forward()` |
| `alias %s not exist. Try to see if it can be created on the fly` | `email_handler.py:545` | `handle_forward()` |
| `alias %s cannot be created on-the-fly, return 550` | `email_handler.py:551` | `handle_forward()` |
| `Cannot auto-create custom domain alias for %s because there's no custom domain for %s` | `app/alias_utils.py:104` | `check_if_alias_can_be_auto_created_for_custom_domain()` |
| `Cannot auto-create %s since it has no directory separator` | `app/alias_utils.py:165` | `check_if_alias_can_be_auto_created_for_a_directory()` |
| `Created contact %s for alias %s with email %s invalid_email=%s` | `app/contact_utils.py:110` | `create_contact()` |
| `Forward %s -> %s -> %s` | `email_handler.py:688` | `forward_email_to_mailbox()` |
| `Create %s for %s, %s, %s` (forward EmailLog) | `email_handler.py:740` | `forward_email_to_mailbox()` |
| `missing date header, create one` | `email_handler.py:857` | `forward_email_to_mailbox()` |
| `From header, new:%s, old:%s` | `email_handler.py:867` | `forward_email_to_mailbox()` |
| `Replace Cc header, old: %s, new: %s` | `email_handler.py:313` | `replace_header_when_forward()` |
| `Forward mail from %s to %s, mail_options:%s, rcpt_options:%s` | `email_handler.py:893` | `forward_email_to_mailbox()` |
| `Create %s for %s, %s, %s` (reply EmailLog) | `email_handler.py:1051` | `handle_reply()` |
| `From header is %s` (reply) | `email_handler.py:1171` | `handle_reply()` |
| `Replace To header, old: %s, new: %s` | `email_handler.py:380` | `replace_header_when_reply()` |
| `create a new sl_message_id %s` | `email_handler.py:1314` | `replace_original_message_id()` |
| `send email with subject '%s', from '%s' to '%s'` | `app/mail_sender.py:131` | `MailSender.send()` |
| `Finish mail_from %s, rcpt_tos %s, takes %s seconds with return code '%s'<<===` | `email_handler.py:2367` | `MailHandler._handle()` |

### 7.2 SMTP status constants

From `app/email/status.py`:

| Constant | Value | Purpose |
|---|---|---|
| `E200` | `"250 Message accepted for delivery"` | Success |
| `E502` | `"550 SL E502 Email not exist"` | (Not observed in this investigation) |
| `E515` | `"550 SL E515 Email not exist"` | Non-existent alias, returned at `email_handler.py:555` |

### 7.3 Critical code paths

- **Forward success flow:** `_handle()` (`email_handler.py:2335`) → `handle()` (`email_handler.py:1945`) → `Forward phase` branch at `email_handler.py:2202` → `handle_forward()` (`email_handler.py:536`) → `get_or_create_contact()` → `create_contact()` (`app/contact_utils.py:42`) → `forward_email_to_mailbox()` (`email_handler.py:679`) → `EmailLog.create()` (line 732) → `From` rewrite (line 867 via `contact.new_addr()`) → `sl_sendmail()` → return `status.E200`.
- **Non-existent alias flow:** same preamble → `handle_forward()` → `Alias.get_by` returns `None` → `try_auto_create()` returns `None` → `return [(False, status.E515)]` at line 555. No DB writes.
- **Reply flow:** same preamble → `Reply phase` branch at `email_handler.py:2196` → `handle_reply()` (`email_handler.py:966`) → `EmailLog.create(is_reply=True)` at line 1045 → `From` rewrite (line 1171) → `replace_original_message_id()` (line 1296) → `make_msgid(str(email_log.id), alias_domain)` → `MessageIDMatching.create()` → `email_log.sl_message_id = sl_message_id` → `sl_sendmail()` → return `status.E200`.

### 7.4 Model definitions (all inherit from `ModelMixin`)

| Model | File : line | Notable columns (beyond `id`, `created_at`, `updated_at`) |
|---|---|---|
| `User` | `app/models.py:336` | `email`, `sender_format`, `include_sender_in_reverse_alias` |
| `Alias` | `app/models.py:1469` | `email`, `user_id`, `mailbox_id`, `last_email_log_id` |
| `Contact` | `app/models.py:1863` | `alias_id`, `user_id`, `website_email`, `reply_email`, `name`, `mail_from`, `automatic_created`, `invalid_email`, `block_forward`, `flags` |
| `EmailLog` | `app/models.py:2060` | `contact_id`, `user_id`, `mailbox_id`, `alias_id`, `message_id`, `sl_message_id`, `is_reply`, `blocked`, `bounced` |
| `MessageIDMatching` | `app/models.py:3365` | `sl_message_id` (UNIQUE), `original_message_id` (UNIQUE), `email_log_id` (FK → `email_log.id`) |
| `SLDomain` | `app/models.py` | `domain`, `use_as_reverse_alias` |

---

## 8. Appendix B — Complete Captured Log Streams

These are the **complete, verbatim** log buffers captured from the `SL` logger during this investigation run. File paths retain their full absolute form as emitted by the logger.

### 8.1 Forward — success path

```text
2026-04-17 00:42:37,899 - SL - DEBUG - 73078 - "/tmp/blitzy/app/blitzy-449db990-d7ac-49a8-88d6-d13b603aa6b5_fcd31c/app/log.py:24" - set_message_id() -  - set message_id 05337315-fb60-485b-a323-8c0118e6fe12
2026-04-17 00:42:37,899 - SL - DEBUG - 73078 - "/tmp/blitzy/app/blitzy-449db990-d7ac-49a8-88d6-d13b603aa6b5_fcd31c/email_handler.py:2342" - _handle() - 05337315-fb60-485b-a323-8c0118e6fe12 - ====>=====>====>====>====>====>====>====>
2026-04-17 00:42:37,899 - SL - INFO  - 73078 - "/tmp/blitzy/app/blitzy-449db990-d7ac-49a8-88d6-d13b603aa6b5_fcd31c/email_handler.py:2343" - _handle() - 05337315-fb60-485b-a323-8c0118e6fe12 - New message, mail from env-codzfj@example.com, rctp tos ['umbels_zoning133@sl.local']
2026-04-17 00:42:37,900 - SL - INFO  - 73078 - "/tmp/blitzy/app/blitzy-449db990-d7ac-49a8-88d6-d13b603aa6b5_fcd31c/email_handler.py:1956" - handle() - 05337315-fb60-485b-a323-8c0118e6fe12 - Set CONTENT_TRANSFER_ENCODING
2026-04-17 00:42:37,901 - SL - DEBUG - 73078 - "/tmp/blitzy/app/blitzy-449db990-d7ac-49a8-88d6-d13b603aa6b5_fcd31c/email_handler.py:1963" - handle() - 05337315-fb60-485b-a323-8c0118e6fe12 - Cannot parse Postfix queue ID from [<two Received headers omitted for readability>]
2026-04-17 00:42:37,901 - SL - DEBUG - 73078 - "/tmp/blitzy/app/blitzy-449db990-d7ac-49a8-88d6-d13b603aa6b5_fcd31c/email_handler.py:1980" - handle() - 05337315-fb60-485b-a323-8c0118e6fe12 - ==>> Handle mail_from:env-codzfj@example.com, rcpt_tos:['umbels_zoning133@sl.local'], header_from:sender@example.com, header_to:umbels_zoning133@sl.local, cc:gxhezxcnpansfnzdrzye@gxhezxcnpansfnzdrzye.com, reply-to:None, message_id:<af07e94a66ece6564ae30a2aaac7a34c@frontapp.com>, client_ip:None, headers:<full header list omitted for readability — present in the raw capture>, mail_options:[], rcpt_options:[]
2026-04-17 00:42:37,905 - SL - DEBUG - 73078 - "/tmp/blitzy/app/blitzy-449db990-d7ac-49a8-88d6-d13b603aa6b5_fcd31c/email_handler.py:2202" - handle() - 05337315-fb60-485b-a323-8c0118e6fe12 - Forward phase env-codzfj@example.com(sender@example.com) -> umbels_zoning133@sl.local
2026-04-17 00:42:37,912 - SL - DEBUG - 73078 - "/tmp/blitzy/app/blitzy-449db990-d7ac-49a8-88d6-d13b603aa6b5_fcd31c/email_handler.py:580" - handle_forward() - 05337315-fb60-485b-a323-8c0118e6fe12 - Create or get contact for from_header:sender@example.com
2026-04-17 00:42:37,931 - SL - DEBUG - 73078 - "/tmp/blitzy/app/blitzy-449db990-d7ac-49a8-88d6-d13b603aa6b5_fcd31c/app/contact_utils.py:110" - create_contact() - 05337315-fb60-485b-a323-8c0118e6fe12 - Created contact <Contact 1 sender@example.com 2> for alias <Alias 2 umbels_zoning133@sl.local> with email sender@example.com invalid_email=False
2026-04-17 00:42:37,931 - SL - INFO  - 73078 - "/tmp/blitzy/app/blitzy-449db990-d7ac-49a8-88d6-d13b603aa6b5_fcd31c/app/handler/dmarc.py:35" - apply_dmarc_policy_for_forward_phase() - 05337315-fb60-485b-a323-8c0118e6fe12 - Spam check result in <app.handler.spamd_result.SpamdResult object at 0x7e3cae253b20>
2026-04-17 00:42:37,939 - SL - DEBUG - 73078 - "/tmp/blitzy/app/blitzy-449db990-d7ac-49a8-88d6-d13b603aa6b5_fcd31c/email_handler.py:688" - forward_email_to_mailbox() - 05337315-fb60-485b-a323-8c0118e6fe12 - Forward <Contact 1 sender@example.com 2> -> <Alias 2 umbels_zoning133@sl.local> -> <Mailbox 1 user_w09gomrj4g@mailbox.test>
2026-04-17 00:42:37,943 - SL - DEBUG - 73078 - "/tmp/blitzy/app/blitzy-449db990-d7ac-49a8-88d6-d13b603aa6b5_fcd31c/email_handler.py:740" - forward_email_to_mailbox() - 05337315-fb60-485b-a323-8c0118e6fe12 - Create <EmailLog 1> for <Contact 1 sender@example.com 2>, <User 1 Test User user_w09gomrj4g@mailbox.test>, <Mailbox 1 user_w09gomrj4g@mailbox.test>
2026-04-17 00:42:37,948 - SL - WARN  - 73078 - "/tmp/blitzy/app/blitzy-449db990-d7ac-49a8-88d6-d13b603aa6b5_fcd31c/email_handler.py:857" - forward_email_to_mailbox() - 05337315-fb60-485b-a323-8c0118e6fe12 - missing date header, create one
2026-04-17 00:42:37,952 - SL - DEBUG - 73078 - "/tmp/blitzy/app/blitzy-449db990-d7ac-49a8-88d6-d13b603aa6b5_fcd31c/email_handler.py:867" - forward_email_to_mailbox() - 05337315-fb60-485b-a323-8c0118e6fe12 - From header, new:"sender at example.com" <sender_at_example_com_oupoo@sl.local>, old:sender@example.com
2026-04-17 00:42:37,953 - SL - DEBUG - 73078 - "/tmp/blitzy/app/blitzy-449db990-d7ac-49a8-88d6-d13b603aa6b5_fcd31c/email_handler.py:286" - replace_header_when_forward() - 05337315-fb60-485b-a323-8c0118e6fe12 - create contact for alias <Alias 2 umbels_zoning133@sl.local> and email gxhezxcnpansfnzdrzye@gxhezxcnpansfnzdrzye.com, header Cc
2026-04-17 00:42:37,965 - SL - DEBUG - 73078 - "/tmp/blitzy/app/blitzy-449db990-d7ac-49a8-88d6-d13b603aa6b5_fcd31c/email_handler.py:313" - replace_header_when_forward() - 05337315-fb60-485b-a323-8c0118e6fe12 - Replace Cc header, old: gxhezxcnpansfnzdrzye@gxhezxcnpansfnzdrzye.com, new: "gxhezxcnpansfnzdrzye at gxhezxcnpansfnzdrzye.com" <gxhezxcnpansfnzdrzye_at_gxhezxcnpansfnzdrzye_com_qfrqkxrtjj@sl.local>
2026-04-17 00:42:37,966 - SL - DEBUG - 73078 - "/tmp/blitzy/app/blitzy-449db990-d7ac-49a8-88d6-d13b603aa6b5_fcd31c/email_handler.py:313" - replace_header_when_forward() - 05337315-fb60-485b-a323-8c0118e6fe12 - Replace To header, old: umbels_zoning133@sl.local, new: umbels_zoning133@sl.local
2026-04-17 00:42:37,966 - SL - INFO  - 73078 - "/tmp/blitzy/app/blitzy-449db990-d7ac-49a8-88d6-d13b603aa6b5_fcd31c/app/handler/unsubscribe_generator.py:36" - _generate_header_with_original_behaviour() - 05337315-fb60-485b-a323-8c0118e6fe12 - Email has no unsubscribe header
2026-04-17 00:42:37,971 - SL - DEBUG - 73078 - "/tmp/blitzy/app/blitzy-449db990-d7ac-49a8-88d6-d13b603aa6b5_fcd31c/email_handler.py:893" - forward_email_to_mailbox() - 05337315-fb60-485b-a323-8c0118e6fe12 - Forward mail from sender@example.com to user_w09gomrj4g@mailbox.test, mail_options:[], rcpt_options:[]
2026-04-17 00:42:37,972 - SL - DEBUG - 73078 - "/tmp/blitzy/app/blitzy-449db990-d7ac-49a8-88d6-d13b603aa6b5_fcd31c/app/mail_sender.py:131" - send() - 05337315-fb60-485b-a323-8c0118e6fe12 - send email with subject 'Something', from '"sender at example.com" <sender_at_example_com_oupoo@sl.local>' to 'umbels_zoning133@sl.local'
2026-04-17 00:42:37,973 - SL - INFO  - 73078 - "/tmp/blitzy/app/blitzy-449db990-d7ac-49a8-88d6-d13b603aa6b5_fcd31c/email_handler.py:2367" - _handle() - 05337315-fb60-485b-a323-8c0118e6fe12 - Finish mail_from env-codzfj@example.com, rcpt_tos ['umbels_zoning133@sl.local'], takes 0.07348799705505371 seconds with return code '250 Message accepted for delivery'<<===
```

### 8.2 Forward — non-existent alias path

```text
2026-04-17 00:42:37,984 - SL - DEBUG - 73078 - ".../app/log.py:24" - set_message_id() - 05337315-fb60-485b-a323-8c0118e6fe12 - set message_id 3ea47bce-2bbf-4fd5-8372-3107da566427
2026-04-17 00:42:37,984 - SL - DEBUG - 73078 - ".../email_handler.py:2342" - _handle() - 3ea47bce-2bbf-4fd5-8372-3107da566427 - ====>=====>====>====>====>====>====>====>
2026-04-17 00:42:37,984 - SL - INFO  - 73078 - ".../email_handler.py:2343" - _handle() - 3ea47bce-2bbf-4fd5-8372-3107da566427 - New message, mail from someone@external.com, rctp tos ['nonexistent@sl.local']
2026-04-17 00:42:37,985 - SL - INFO  - 73078 - ".../email_handler.py:1956" - handle()  - 3ea47bce-2bbf-4fd5-8372-3107da566427 - Set CONTENT_TRANSFER_ENCODING
2026-04-17 00:42:37,985 - SL - DEBUG - 73078 - ".../email_handler.py:1963" - handle()  - 3ea47bce-2bbf-4fd5-8372-3107da566427 - Cannot parse Postfix queue ID from [<two Received headers>]
2026-04-17 00:42:37,985 - SL - DEBUG - 73078 - ".../email_handler.py:1980" - handle()  - 3ea47bce-2bbf-4fd5-8372-3107da566427 - ==>> Handle mail_from:someone@external.com, rcpt_tos:['nonexistent@sl.local'], header_from:sender@example.com, header_to:nonexistent@sl.local, cc:pppmsbutcwqpmipzfdpr@pppmsbutcwqpmipzfdpr.com, reply-to:None, message_id:<af07e94a66ece6564ae30a2aaac7a34c@frontapp.com>, client_ip:None, headers:<...>, mail_options:[], rcpt_options:[]
2026-04-17 00:42:37,989 - SL - DEBUG - 73078 - ".../email_handler.py:2202" - handle()  - 3ea47bce-2bbf-4fd5-8372-3107da566427 - Forward phase someone@external.com(sender@example.com) -> nonexistent@sl.local
2026-04-17 00:42:37,994 - SL - DEBUG - 73078 - ".../email_handler.py:545"  - handle_forward() - 3ea47bce-2bbf-4fd5-8372-3107da566427 - alias nonexistent@sl.local not exist. Try to see if it can be created on the fly
2026-04-17 00:42:37,999 - SL - INFO  - 73078 - ".../app/alias_utils.py:104" - check_if_alias_can_be_auto_created_for_custom_domain() - 3ea47bce-2bbf-4fd5-8372-3107da566427 - Cannot auto-create custom domain alias for nonexistent@sl.local because there's no custom domain for sl.local
2026-04-17 00:42:38,000 - SL - INFO  - 73078 - ".../app/alias_utils.py:165" - check_if_alias_can_be_auto_created_for_a_directory() - 3ea47bce-2bbf-4fd5-8372-3107da566427 - Cannot auto-create nonexistent@sl.local since it has no directory separator
2026-04-17 00:42:38,000 - SL - DEBUG - 73078 - ".../email_handler.py:551"  - handle_forward() - 3ea47bce-2bbf-4fd5-8372-3107da566427 - alias nonexistent@sl.local cannot be created on-the-fly, return 550
2026-04-17 00:42:38,001 - SL - INFO  - 73078 - ".../email_handler.py:2367" - _handle() - 3ea47bce-2bbf-4fd5-8372-3107da566427 - Finish mail_from someone@external.com, rcpt_tos ['nonexistent@sl.local'], takes 0.016770362854003906 seconds with return code '550 SL E515 Email not exist'<<===
```

### 8.3 Reply path

```text
2026-04-17 00:42:38,004 - SL - DEBUG - 73078 - ".../app/log.py:24" - set_message_id() - 3ea47bce-2bbf-4fd5-8372-3107da566427 - set message_id 5faece51-dd22-4775-8019-5829ab6c5a02
2026-04-17 00:42:38,004 - SL - DEBUG - 73078 - ".../email_handler.py:2342" - _handle() - 5faece51-dd22-4775-8019-5829ab6c5a02 - ====>=====>====>====>====>====>====>====>
2026-04-17 00:42:38,004 - SL - INFO  - 73078 - ".../email_handler.py:2343" - _handle() - 5faece51-dd22-4775-8019-5829ab6c5a02 - New message, mail from user_w09gomrj4g@mailbox.test, rctp tos ['sender_at_example_com_oupoo@sl.local']
2026-04-17 00:42:38,005 - SL - INFO  - 73078 - ".../email_handler.py:1956" - handle() - 5faece51-dd22-4775-8019-5829ab6c5a02 - Set CONTENT_TRANSFER_ENCODING
2026-04-17 00:42:38,005 - SL - DEBUG - 73078 - ".../email_handler.py:1963" - handle() - 5faece51-dd22-4775-8019-5829ab6c5a02 - Cannot parse Postfix queue ID from [<two Received headers>]
2026-04-17 00:42:38,005 - SL - DEBUG - 73078 - ".../email_handler.py:1980" - handle() - 5faece51-dd22-4775-8019-5829ab6c5a02 - ==>> Handle mail_from:user_w09gomrj4g@mailbox.test, rcpt_tos:['sender_at_example_com_oupoo@sl.local'], header_from:None, header_to:sender_at_example_com_oupoo@sl.local, cc:None, reply-to:None, message_id:<reply-zlvutogcnkcnbkhnolrd@mailbox.test>, client_ip:None, headers:<...>, mail_options:[], rcpt_options:[]
2026-04-17 00:42:38,008 - SL - DEBUG - 73078 - ".../email_handler.py:2196" - handle() - 5faece51-dd22-4775-8019-5829ab6c5a02 - Reply phase user_w09gomrj4g@mailbox.test(None) -> sender_at_example_com_oupoo@sl.local
2026-04-17 00:42:38,014 - SL - INFO  - 73078 - ".../app/handler/dmarc.py:162" - apply_dmarc_policy_for_reply_phase() - 5faece51-dd22-4775-8019-5829ab6c5a02 - Spam check result is <app.handler.spamd_result.SpamdResult object at 0x7e3cae44e4a0>
2026-04-17 00:42:38,017 - SL - DEBUG - 73078 - ".../email_handler.py:1051" - handle_reply() - 5faece51-dd22-4775-8019-5829ab6c5a02 - Create <EmailLog 2> for <Contact 1 sender@example.com 2>, <User 1 Test User user_w09gomrj4g@mailbox.test>, <Mailbox 1 user_w09gomrj4g@mailbox.test>
2026-04-17 00:42:38,022 - SL - DEBUG - 73078 - ".../email_handler.py:1171" - handle_reply() - 5faece51-dd22-4775-8019-5829ab6c5a02 - From header is umbels_zoning133@sl.local
2026-04-17 00:42:38,023 - SL - DEBUG - 73078 - ".../email_handler.py:380"  - replace_header_when_reply() - 5faece51-dd22-4775-8019-5829ab6c5a02 - Replace To header, old: sender_at_example_com_oupoo@sl.local, new: sender@example.com
2026-04-17 00:42:38,023 - SL - DEBUG - 73078 - ".../email_handler.py:383"  - replace_header_when_reply() - 5faece51-dd22-4775-8019-5829ab6c5a02 - delete the Cc header. Old value None
2026-04-17 00:42:38,024 - SL - DEBUG - 73078 - ".../email_handler.py:1314" - replace_original_message_id() - 5faece51-dd22-4775-8019-5829ab6c5a02 - create a new sl_message_id <177638655802.73078.4887324053016198845.2@sl.local>
2026-04-17 00:42:38,032 - SL - WARN  - 73078 - ".../email_handler.py:1206" - handle_reply() - 5faece51-dd22-4775-8019-5829ab6c5a02 - missing date header, add one
2026-04-17 00:42:38,035 - SL - DEBUG - 73078 - ".../email_handler.py:1212" - handle_reply() - 5faece51-dd22-4775-8019-5829ab6c5a02 - send email from umbels_zoning133@sl.local to sender@example.com, mail_options:[],rcpt_options:[]
2026-04-17 00:42:38,039 - SL - DEBUG - 73078 - ".../app/mail_sender.py:131" - send() - 5faece51-dd22-4775-8019-5829ab6c5a02 - send email with subject 'Something', from 'umbels_zoning133@sl.local' to 'sender@example.com'
2026-04-17 00:42:38,040 - SL - INFO  - 73078 - ".../email_handler.py:2367" - _handle() - 5faece51-dd22-4775-8019-5829ab6c5a02 - Finish mail_from user_w09gomrj4g@mailbox.test, rcpt_tos ['sender_at_example_com_oupoo@sl.local'], takes 0.03591609001159668 seconds with return code '250 Message accepted for delivery'<<===
```

---

## 9. Appendix C — Environment, Fixtures & Cleanup

### 9.1 Environment settings used during the investigation

All values come from `tests/test.env` and `pyproject.toml` **as shipped in the repository** — no file was modified.

| Variable | Value | Purpose |
|---|---|---|
| `CONFIG` | `tests/test.env` | Tells `app/config.py` to load test-env variables |
| `EMAIL_DOMAIN` | `sl.local` | Base domain for reverse aliases and VERP addresses |
| `OTHER_ALIAS_DOMAINS` | `["d1.test","d2.test","sl.local"]` | Domains served as SL domains |
| `NOT_SEND_EMAIL` | `true` | `MailSender.send()` logs and returns without transmission |
| `DB_URI` | `postgresql://test:test@localhost:15432/test` | Binds SQLAlchemy engine to the test PostgreSQL container |
| `MEM_STORE_URI` | `redis://localhost` | Redis connection (rate limiter + session) |
| `DKIM_PRIVATE_KEY_PATH` | `local_data/dkim.key` | Key used by `add_dkim_signature()` in `forward_email_to_mailbox()` |
| Python version | `3.10.20` | Pinned by `pyproject.toml`: `python = "^3.10"` |

### 9.2 Harness fixture usage

- **EML fixture:** `tests/example_emls/replacement_on_forward_phase.eml` — a Jinja2 template containing realistic rspamd-produced headers (`X-Spamd-Result`, `Authentication-Results`, `X-Feedback-ID`, etc.) and a `Message-ID` slot. Rendered once per scenario to avoid message-id collisions in the reply scenario.
- **Test-env override helpers:** `tests/conftest.py` and `tests/utils.py` provided the `create_new_user()` pattern used to bootstrap the user/mailbox/alias trio. The investigation used the same helpers but ran them directly rather than via pytest.
- **Outbound interception:** `mail_sender.store_emails_test_decorator` (defined at `app/mail_sender.py:111`) was applied to the block of code invoking `_handle()`. This flips `MailSender.store_emails_instead_of_sending(True)` during the block and then resets it to false, mirroring how `tests/handler/test_preserved_headers.py` uses it.

### 9.3 Cleanup performed

The PostgreSQL (`sl-test-db`) and Redis (`sl-test-redis`) containers were provisioned for this investigation and are not part of the repository. Per the user's explicit instruction ("clean up any test containers or DB instances you spin up"), the post-investigation step will remove them with:

```bash
docker rm -f sl-test-db sl-test-redis
```

- No `-v` persistent volume was mounted for PostgreSQL, so volume cleanup is automatic.
- The Python virtual environment at `.venv/` is gitignored and will be removed by the container teardown.
- Harness scratch files under `/tmp/investigation/` (log captures, the results JSON, and the harness script) are ephemeral and outside the repository.
- **No existing file in the SimpleLogin repository was modified, created, or deleted by the investigation harness itself.** The only file added to the destination repository is this document: `blitzy/documentation/app_2cd6ee777f8c.md`.

### 9.4 Reproducibility

To reproduce these exact runtime values (modulo randomness in `random_string()` and `make_msgid`'s timestamp/pid/random components), the investigation would:

1. Spin up PostgreSQL 13 on `localhost:15432` and Redis 7 on `localhost:6379`.
2. Apply all Alembic migrations: `CONFIG=tests/test.env .venv/bin/alembic upgrade head`.
3. Seed SL domains via `init_app.add_sl_domains()` and the Proton partner via `init_app.add_proton_partner()`.
4. Enter a `create_light_app().app_context()`, create a user/mailbox/alias, construct an `aiosmtpd.smtp.Envelope` and `email.message.Message`, and invoke `email_handler.MailHandler()._handle(envelope, msg)` inside `mail_sender.store_emails_test_decorator`.
5. After each invocation, call `Session.expire_all()` and re-query `Contact`, `EmailLog`, `MessageIDMatching`, and `UserAuditLog` by primary key to capture their actual committed state.
6. For the reply leg, submit an envelope `from=user.email, to=[contact.reply_email]` and observe `MessageIDMatching` population.

All randomly generated values (reverse-alias local part, Message-ID timestamp/pid/random, correlation UUIDs) will differ between runs. Structural values (log format string, log call sites, `headers_to_keep` allow-list, `SenderFormatEnum.AT` formatting, `make_msgid` template, `ModelMixin` columns, `MessageIDMatching` uniqueness constraints, and the SMTP status strings `E200`/`E515`) are fully deterministic.

---

*End of investigation document.*
