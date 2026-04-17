# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

Based on the prompt, the Blitzy platform understands that the new feature requirement is to produce a comprehensive runtime-trace investigation document that answers four specific diagnostic questions about SimpleLogin's email forwarding pipeline, all without modifying any existing source code.

### 0.1.1 Core Feature Objective

The user is investigating a production issue involving inconsistent behavior when emails are forwarded through SimpleLogin aliases. The deliverable is a standalone Markdown document (`app_2cd6ee777f8c.md`) placed in `blitzy/documentation/` that provides actual runtime values captured by building and executing the codebase against a test database. The four investigation questions are:

- **Log Message Text**: Identify the exact log output when an email is successfully forwarded through an alias (happy path) versus when it fails due to a non-existent alias (failure path). The successful path terminates with SMTP status `"250 Message accepted for delivery"` (status code `E200` in `app/email/status.py`), while the non-existent alias path returns `"550 SL E515 Email not exist"` (`E515`). The precise log lines are emitted by `LOG.d()` calls at `email_handler.py` lines 545–555.

- **SL Message-ID Generation**: Determine what SL Message-ID is generated during forwarding and how it differs from the original `Message-ID`. In the forward phase, the original `Message-ID` is preserved on the outbound message (see `headers_to_keep` list at line 800). The SL Message-ID is only generated during the **reply phase** by `replace_original_message_id()` at line 1296, using `email.utils.make_msgid(str(email_log.id), alias_domain)` and stored in the `MessageIDMatching` table. The investigation must capture both paths.

- **From Header Transformation**: Capture the exact `From` header value after transformation, which uses the `contact.new_addr()` method (line 865 of `email_handler.py`). This formats the address as `"sender at example.com" <{reply_email}>` using `sl_formataddr()`, where `reply_email` is a randomly generated reverse-alias address.

- **Database Records Created**: Identify and capture actual record IDs and timestamps for all database records created during a single forward operation: `Contact` record, `EmailLog` record, and (in the reply phase) `MessageIDMatching` record. Each record inherits `id` (auto-increment integer), `created_at` (Arrow UTC timestamp), and `updated_at` from `ModelMixin`.

### 0.1.2 Special Instructions and Constraints

- **No Source Modification**: The user explicitly requires: "Just don't modify any source files while investigating." All existing files in the repository must remain untouched. Only a new Markdown document may be created per the `SWE-AtlasQnA-Repo` rule.
- **Cleanup Required**: The user states: "clean up any test containers or DB instances you spin up." Any PostgreSQL containers, Redis instances, or temporary databases must be torn down after the investigation completes.
- **Actual Runtime Values**: The user emphasizes: "I need to see actual generated values, not what the code logic suggests should happen." This requires building and running the application against a real database to capture concrete IDs, timestamps, message-IDs, and log output.
- **Implementation Rule — SWE-AtlasQnA-Repo**: Create a new markdown document named `app_2cd6ee777f8c.md` in the `blitzy/documentation` directory that comprehensively answers all posed questions, with rationale, based on code-as-truth, without modifying any existing source files.

### 0.1.3 Technical Interpretation

These investigation requirements translate to the following technical implementation strategy:

- To **capture exact log messages**, we will stand up a test PostgreSQL database, configure the test environment using `tests/test.env`, create a user/alias/contact via the ORM, construct a synthetic `aiosmtpd.smtp.Envelope` and `email.message.Message`, and invoke `email_handler.handle()` directly, intercepting log output from the `SL` logger defined in `app/log.py`.
- To **observe SL Message-ID generation**, we will run both a forward-phase invocation (where the original Message-ID is preserved) and a reply-phase invocation (where `replace_original_message_id()` generates a new SL Message-ID via `make_msgid()`), then query the `MessageIDMatching` table for the persisted mapping.
- To **capture the From header transformation**, we will use `mail_sender.store_emails_test_decorator` to intercept the outbound `SendRequest` and extract `request.msg[headers.FROM]`, which contains the fully formatted reverse-alias address produced by `Contact.new_addr()`.
- To **retrieve database records**, we will query the ORM after each operation to capture `Contact.id`, `Contact.created_at`, `Contact.reply_email`, `EmailLog.id`, `EmailLog.created_at`, `EmailLog.message_id`, and `MessageIDMatching.sl_message_id`/`original_message_id` with their actual auto-generated values.

## 0.2 Repository Scope Discovery

### 0.2.1 Comprehensive File Analysis

The email forwarding pipeline spans a well-defined set of modules across the repository. Every file listed below was inspected to map the investigation scope.

**Core Email Processing Chain (files that implement the flow under investigation):**

| File Path | Role in Forward Pipeline | Lines of Interest |
|---|---|---|
| `email_handler.py` | Central SMTP handler: `handle()`, `handle_forward()`, `forward_email_to_mailbox()`, `handle_reply()`, `replace_original_message_id()` | 536–928 (forward), 966–1261 (reply), 1296–1362 (Message-ID), 1945–2234 (routing) |
| `app/email_utils.py` | `generate_reply_email()`, `sl_formataddr()`, `generate_verp_email()`, `get_verp_info_from_email()`, `is_reverse_alias()` | 1103–1153 (reply-email gen), 1438–1498 (VERP), 1501–1505 (formataddr) |
| `app/contact_utils.py` | `create_contact()` — creates Contact records with reply_email, emits audit logs | 42–120 |
| `app/mail_sender.py` | `MailSender.send()`, `store_emails_test_decorator` for intercepting outbound mail in tests | 96–180 |
| `app/models.py` | ORM definitions: `Alias`, `Contact`, `EmailLog`, `MessageIDMatching`, `Mailbox`, `User`, `ModelMixin` (id, created_at, updated_at) | 62–130 (ModelMixin), 1469–1770 (Alias), 1863–2058 (Contact), 2060–2168 (EmailLog), 3365–3379 (MessageIDMatching) |
| `app/log.py` | Logger `LOG` with `SL` namespace, `set_message_id()`, `EmailHandlerFilter` for message-id correlation | 1–79 |
| `app/email/status.py` | All SMTP status constants: `E200`, `E515`, `E502`, etc. | 1–64 |
| `app/email/headers.py` | Header name constants: `MESSAGE_ID`, `FROM`, `TO`, `SL_DIRECTION`, `SL_EMAIL_LOG_ID`, etc. | 1–59 |
| `app/config.py` | `EMAIL_DOMAIN`, `BOUNCE_PREFIX`, `VERP_PREFIX`, `NOREPLY`, `DB_URI` | 92, 100–117, 435, 499–507 |
| `app/db.py` | SQLAlchemy engine, connection, scoped Session | 1–18 |

**Supporting Infrastructure (needed for test execution context):**

| File Path | Purpose |
|---|---|
| `server.py` | `create_light_app()` — lightweight Flask app context for email handler | line 127 |
| `init_app.py` | `add_sl_domains()`, `add_proton_partner()` — seed SLDomain and Partner records | 39–66 |
| `app/alias_utils.py` | `try_auto_create()` — auto-create alias for catch-all/directory domains | referenced at `email_handler.py:549` |
| `app/handler/dmarc.py` | `apply_dmarc_policy_for_forward_phase()`, `apply_dmarc_policy_for_reply_phase()` | invoked at `email_handler.py:615`, `email_handler.py:1012` |
| `app/handler/spamd_result.py` | `SpamdResult.extract_from_headers()` — parses rspamd results | invoked at `email_handler.py:2356` |
| `app/handler/unsubscribe_generator.py` | `UnsubscribeGenerator.add_header_to_message()` | invoked at `email_handler.py:889` |

**Test Infrastructure (patterns for the investigation harness):**

| File Path | Purpose |
|---|---|
| `tests/conftest.py` | Flask test app creation, database setup, `flask_client` fixture with transactional rollback | 1–77 |
| `tests/test.env` | Test environment variables: `EMAIL_DOMAIN=sl.local`, `DB_URI=postgresql://test:test@localhost:15432/test` | 1–80 |
| `tests/utils.py` | `create_new_user()`, `load_eml_file()`, `random_email()`, `random_token()` | 1–92 |
| `tests/test_email_handler.py` | Existing tests for `handle_forward()`, `handle_reply()`, DMARC, bounce handling | 1–280+ |
| `tests/handler/test_preserved_headers.py` | Pattern for using `mail_sender.store_emails_test_decorator` to capture outbound messages | 1–75 |
| `tests/example_emls/replacement_on_forward_phase.eml` | Jinja2-templated EML fixture with rspamd headers for forward testing | complete file |

**Configuration and Build Files:**

| File Path | Purpose |
|---|---|
| `pyproject.toml` | Poetry dependency manifest: Python ^3.10, Flask ^1.1.2, SQLAlchemy 1.3.24, aiosmtpd ^1.2 | 60–119 |
| `Dockerfile` | Multi-stage build: Node for frontend, Poetry for Python backend, Gunicorn on port 7777 | complete file |
| `example.env` | Template for all runtime environment variables | complete file |
| `alembic.ini` | Alembic migration configuration pointing to DB_URI | complete file |

### 0.2.2 Integration Point Discovery

- **Database interaction**: All record creation flows through `app/db.py`'s `Session` (scoped SQLAlchemy session bound to `engine`). The `ModelMixin.create()` method adds objects to the Session and optionally commits.
- **SMTP delivery**: `app/mail_sender.py`'s `sl_sendmail()` dispatches through `MailSender.send()` which, in test mode with `NOT_SEND_EMAIL=true`, logs but does not actually deliver. The `store_emails_test_decorator` captures `SendRequest` objects for inspection.
- **Logging**: All log output flows through `app/log.py`'s `LOG` logger (named `SL`), which uses `EmailHandlerFilter` to inject `message_id` into every log record. The format is: `%(asctime)s - %(name)s - %(levelname)s - %(process)d - "%(pathname)s:%(lineno)d" - %(funcName)s() - %(message_id)s - %(message)s`.

### 0.2.3 New File Requirements

- **CREATE**: `blitzy/documentation/app_2cd6ee777f8c.md` — The investigation results document containing actual runtime-captured values for all four diagnostic questions, with rationale and evidence from code execution. This is the sole new file per the `SWE-AtlasQnA-Repo` rule.

## 0.3 Dependency Inventory

### 0.3.1 Key Packages for Investigation

All packages listed are from the project's `pyproject.toml` and are required to execute the email forwarding code paths under investigation. No new dependencies need to be added.

| Registry | Package | Version | Purpose in Investigation |
|---|---|---|---|
| PyPI | python | ^3.10 (target: 3.10) | Runtime — `pyproject.toml` specifies `python = "^3.10"`, `target-version = ['py310']` |
| PyPI | flask | ^1.1.2 | App context required by `create_light_app()` for `email_handler._handle()` |
| PyPI | SQLAlchemy | 1.3.24 (pinned) | ORM for all database models: `Alias`, `Contact`, `EmailLog`, `MessageIDMatching` |
| PyPI | psycopg2-binary | ^2.9.3 | PostgreSQL driver for `DB_URI` connection |
| PyPI | aiosmtpd | ^1.2 | Provides `Envelope` class used to construct test email envelopes |
| PyPI | arrow | ^0.16.0 | `ArrowType` columns for `created_at`/`updated_at` in all ORM models |
| PyPI | flanker | ^0.9.11 | Email address parsing in `replace_header_when_forward()` |
| PyPI | email_validator | ^1.1.1 | `validate_email()` for contact email validation |
| PyPI | dkimpy | ^1.0.5 | DKIM signing via `add_dkim_signature()` |
| PyPI | newrelic | 8.8.0 (pinned) | `@newrelic.agent.background_task()` decorator on `_handle()` |
| PyPI | pyre2 | ^0.3.6 | `re2` regex backend used in `app/email_utils.py` |
| PyPI | sentry_sdk | ^2.16.0 | Error tracking (configured but non-blocking in test) |
| PyPI | redis | ^4.5.3 | Session store and rate limiter backend |
| PyPI | pytest | ^7.0.0 | Test framework for running investigation harness |
| PyPI | coloredlogs | ^14.0 | Log formatting for `app/log.py` |
| PyPI | Flask-Migrate | ^2.5.3 | Alembic integration for database schema management |
| PyPI | PGPy | 0.5.4 (pinned) | PGP encryption support (not exercised in basic forward, but imported) |
| PyPI | cryptography | 37.0.1 (pinned) | Cryptographic primitives used by dkimpy and PGPy |

### 0.3.2 Import Chain for Forward Phase

The investigation script must ensure all transitive imports resolve. The critical import chain for `email_handler.handle_forward()` is:

- `email_handler` → `app.config` (requires `CONFIG` env var pointing to `.env` file)
- `email_handler` → `app.db.Session` (requires `DB_URI` environment variable)
- `email_handler` → `app.models.*` (requires database tables to exist)
- `email_handler` → `app.email_utils.generate_reply_email` → `app.models.SLDomain`
- `email_handler` → `app.mail_sender.sl_sendmail` → `app.mail_sender.MailSender`
- `email_handler` → `app.handler.dmarc` → `app.handler.spamd_result.SpamdResult`
- `email_handler` → `server.create_light_app` (Flask app context)
- `email_handler` → `init_app.load_pgp_public_keys`

### 0.3.3 External Reference Updates

No external reference updates are required. The investigation produces only a new documentation file and does not alter any configuration, build, or CI/CD files.

## 0.4 Integration Analysis

### 0.4.1 Existing Code Touchpoints

The investigation traces the full forward-phase and reply-phase code paths without modification. The following touchpoints represent the exact execution sequence that the investigation must exercise and observe:

**Forward Phase Execution Chain** (`email_handler.py`):

- `MailHandler._handle()` (line 2335) → generates `uuid4()` message_id, creates Flask app context via `create_light_app()`
- `handle()` (line 1945) → sanitizes `mail_from`/`rcpt_tos`, extracts Postfix queue ID, checks ignore list, sanitizes headers, logs the full `==>> Handle` diagnostic line
- `handle_forward()` (line 536) → resolves alias via `Alias.get_by(email=alias_address)`, attempts `try_auto_create()` if not found, checks user eligibility
- `get_or_create_contact()` (line 180) → calls `contact_utils.create_contact()` which creates `Contact` record with `reply_email` from `generate_reply_email()`
- `forward_email_to_mailbox()` (line 679) → creates `EmailLog` record (line 732), strips headers, sets `SL_DIRECTION=Forward`, replaces `FROM` with `contact.new_addr()`, adds DKIM signature, calls `sl_sendmail()`

**Reply Phase Execution Chain** (`email_handler.py`):

- `handle_reply()` (line 966) → looks up `Contact` by `reply_email`, validates mailbox authorization
- `replace_original_message_id()` (line 1296) → generates SL Message-ID via `make_msgid(str(email_log.id), alias_domain)`, creates `MessageIDMatching` record
- Delivers via `sl_sendmail()` with `VerpType.bounce_reply`

### 0.4.2 Database Records Created per Forward Operation

A single successful forward operation creates the following records in sequence:

| Record | Table | Key Fields | Created By |
|---|---|---|---|
| `Contact` | `contact` | `id`, `created_at`, `user_id`, `alias_id`, `website_email`, `reply_email`, `name`, `mail_from`, `automatic_created=True` | `contact_utils.create_contact()` → `Contact.create()` at `app/contact_utils.py:92` |
| `UserAuditLog` | `user_audit_log` | `id`, `created_at`, `user_id`, `action=CreateContact`, `message` | `emit_user_audit_log()` at `app/contact_utils.py:104` |
| `EmailLog` | `email_log` | `id`, `created_at`, `contact_id`, `user_id`, `mailbox_id`, `alias_id`, `message_id`, `blocked=False`, `is_reply=False` | `EmailLog.create()` at `email_handler.py:732` |
| `Alias.last_email_log_id` update | `alias` | `last_email_log_id` set to new `EmailLog.id` | `EmailLog.create()` SQL update at `app/models.py:2158` |

If the alias does not exist and auto-creation fails, **no records are created** — only log messages are emitted and `E515` is returned.

### 0.4.3 Log Message Flow During Forward

The logging sequence during a successful forward uses the `LOG` logger (namespace `SL`) defined in `app/log.py` with level shortcuts (`LOG.d` = debug, `LOG.i` = info, `LOG.w` = warning). Key log messages in order:

- `"====>=====>====>====>====>====>====>====>"` — visual separator (line 2342)
- `"New message, mail from %s, rctp tos %s"` — initial receipt (line 2343)
- `"==>> Handle mail_from:%s, rcpt_tos:%s, header_from:%s, ..."` — full diagnostic dump (line 1980)
- `"Forward phase %s(%s) -> %s"` — phase identification (line 2202)
- `"Create or get contact for from_header:%s"` — contact resolution (line 580)
- `"Created contact %s for alias %s with email %s"` — new contact creation (app/contact_utils.py:111)
- `"Create %s for %s, %s, %s"` — EmailLog creation (line 740)
- `"Forward %s -> %s -> %s"` — per-mailbox forwarding (line 688)
- `"From header, new:%s, old:%s"` — FROM transformation (line 867)
- `"Forward mail from %s to %s, mail_options:%s, rcpt_options:%s"` — pre-delivery (line 893)
- `"Finish mail_from %s, rcpt_tos %s, takes %s seconds with return code '%s'<<===="` — completion (line 2367)

For a non-existent alias, the key messages are:
- `"alias %s not exist. Try to see if it can be created on the fly"` (line 545)
- `"alias %s cannot be created on-the-fly, return 550"` (line 551)

### 0.4.4 Message-ID Handling Differences

| Phase | Original Message-ID | SL Message-ID | Storage |
|---|---|---|---|
| **Forward** | Preserved in outbound `Message-ID` header (kept in `headers_to_keep` at line 800) | Not generated — only stored in `EmailLog.message_id` field (line 737) | `EmailLog.message_id` = original value |
| **Reply** | Replaced by SL Message-ID in outbound header (line 1338–1339) | Generated via `make_msgid(str(email_log.id), alias_domain)` (line 1311–1312) | `MessageIDMatching(sl_message_id, original_message_id, email_log_id)` + `EmailLog.sl_message_id` |

The SL Message-ID format follows Python's `email.utils.make_msgid()`: `<{email_log_id}.{timestamp}.{random}@{alias_domain}>`.

## 0.5 Technical Implementation

### 0.5.1 File-by-File Execution Plan

The investigation produces exactly one new file. All existing files are read-only.

**Group 1 — Investigation Output (CREATE):**

- **CREATE**: `blitzy/documentation/app_2cd6ee777f8c.md` — Comprehensive Markdown document answering all four diagnostic questions with actual runtime values, code-path rationale, and captured artifacts.

**Group 2 — Existing Files to Read and Execute (NO MODIFICATION):**

- **READ/EXECUTE**: `email_handler.py` — Invoke `MailHandler()._handle()` and `handle()` directly to trace forward and reply flows
- **READ/EXECUTE**: `app/email_utils.py` — Observe `generate_reply_email()` output and `generate_verp_email()` VERP address construction
- **READ/EXECUTE**: `app/contact_utils.py` — Observe `create_contact()` creating Contact records with generated reply addresses
- **READ/EXECUTE**: `app/models.py` — Query ORM objects post-execution to capture actual IDs, timestamps, and field values
- **READ/EXECUTE**: `app/mail_sender.py` — Use `store_emails_test_decorator` pattern to intercept outbound messages
- **READ/EXECUTE**: `app/log.py` — Capture and parse log output from the `SL` logger
- **READ/EXECUTE**: `tests/conftest.py` — Reuse test app setup pattern (Flask app, database, SL domains seeding)
- **READ/EXECUTE**: `tests/utils.py` — Reuse `create_new_user()`, `load_eml_file()` helpers
- **READ/EXECUTE**: `tests/test.env` — Test environment configuration
- **READ/EXECUTE**: `tests/example_emls/replacement_on_forward_phase.eml` — EML fixture for forward-phase testing

### 0.5.2 Implementation Approach

The investigation follows this approach to capture actual runtime values:

**Step 1 — Environment Provisioning:**
- Spin up a temporary PostgreSQL container on port 15432 with credentials `test:test` and database `test` (matching `tests/test.env`)
- Spin up a temporary Redis container (matching `MEM_STORE_URI=redis://localhost`)
- Run Alembic migrations to create the full schema
- Seed SL domains and Proton partner via `init_app.add_sl_domains()` and `add_proton_partner()`

**Step 2 — Forward Phase Trace (Success Path):**
- Create a test user via `User.create()` and a random alias via `Alias.create_new_random()`
- Construct an `Envelope` with `mail_from="sender@external.com"` and `rcpt_tos=[alias.email]`
- Construct a `Message` from the `replacement_on_forward_phase.eml` fixture
- Capture log output by attaching a handler to the `SL` logger
- Invoke `email_handler.MailHandler()._handle(envelope, msg)` using `mail_sender.store_emails_test_decorator`
- Assert the return status is `E200`
- Extract and record: the captured log lines, the outbound `SendRequest.msg[headers.FROM]` value, the `EmailLog` record (id, created_at, message_id), and the `Contact` record (id, created_at, reply_email)

**Step 3 — Forward Phase Trace (Non-Existent Alias Path):**
- Construct an `Envelope` with `rcpt_tos=["nonexistent@sl.local"]`
- Invoke `email_handler.handle(envelope, msg)` within a Flask app context
- Assert the return status is `E515` ("550 SL E515 Email not exist")
- Capture and record the specific log lines showing alias lookup failure

**Step 4 — Reply Phase Trace (SL Message-ID Generation):**
- Using the Contact created in Step 2, construct an `Envelope` with `mail_from=user.email` and `rcpt_tos=[contact.reply_email]`
- Invoke `email_handler.handle(envelope, msg)`
- Query `MessageIDMatching` to capture the generated `sl_message_id` and its relationship to `original_message_id`
- Capture `EmailLog.sl_message_id` to confirm persistence

**Step 5 — Assemble Output Document:**
- Collate all captured runtime values into the investigation document
- Include actual IDs, timestamps, log text, header values, and Message-ID comparisons
- Provide rationale linking each answer to the specific code path

**Step 6 — Cleanup:**
- Tear down the PostgreSQL and Redis containers
- Remove any temporary files created during the investigation

### 0.5.3 Key Runtime Values to Capture

| Value | Source | Extraction Method |
|---|---|---|
| Success log messages | `SL` logger output during `handle()` | Attach `logging.Handler` to `LOG` |
| Failure log messages | `SL` logger output for non-existent alias | Same handler capturing "alias %s not exist" and "cannot be created on-the-fly" |
| SMTP status (success) | Return value of `_handle()` | `== "250 Message accepted for delivery"` |
| SMTP status (failure) | Return value of `handle()` | `== "550 SL E515 Email not exist"` |
| Generated reply_email | `Contact.reply_email` | `Contact.get_by(alias_id=alias.id).reply_email` |
| Transformed From header | `SendRequest.msg[headers.FROM]` | `mail_sender.get_stored_emails()[0].msg["From"]` |
| EmailLog record | `EmailLog` ORM object | `EmailLog.filter_by(alias_id=alias.id).first()` |
| Contact record | `Contact` ORM object | `Contact.get_by(alias_id=alias.id)` |
| SL Message-ID | `MessageIDMatching.sl_message_id` | `MessageIDMatching.get_by(email_log_id=email_log.id)` |
| Original Message-ID | `msg["Message-ID"]` header | Direct header extraction |
| Record timestamps | `created_at` fields | `record.created_at.isoformat()` |

## 0.6 Scope Boundaries

### 0.6.1 Exhaustively In Scope

**Output Artifact:**
- `blitzy/documentation/app_2cd6ee777f8c.md` — sole created file

**Source Files Executed (read-only, no modifications):**
- `email_handler.py` — complete forward/reply/bounce handler chain
- `app/email_utils.py` — reply-email generation, VERP, formataddr, header manipulation
- `app/contact_utils.py` — contact creation with reply_email generation
- `app/models.py` — all ORM models: `User`, `Alias`, `Contact`, `EmailLog`, `MessageIDMatching`, `Mailbox`, `SLDomain`, `ModelMixin`
- `app/config.py` — `EMAIL_DOMAIN`, `BOUNCE_*`, `VERP_*`, `NOREPLY`, `DB_URI`
- `app/db.py` — SQLAlchemy session and engine
- `app/log.py` — `LOG` logger, `set_message_id()`, `EmailHandlerFilter`
- `app/mail_sender.py` — `MailSender`, `sl_sendmail()`, `store_emails_test_decorator`
- `app/email/status.py` — all `E*` status constants
- `app/email/headers.py` — all header name constants
- `app/handler/dmarc.py` — DMARC policy enforcement
- `app/handler/spamd_result.py` — Rspamd result parsing
- `app/handler/unsubscribe_generator.py` — List-Unsubscribe header generation
- `app/alias_utils.py` — `try_auto_create()` for auto-alias creation
- `app/utils.py` — `sanitize_email()`, `random_string()`
- `app/email_validation.py` — `is_valid_email()`, `normalize_reply_email()`
- `server.py` — `create_light_app()` for Flask context
- `init_app.py` — `add_sl_domains()`, `add_proton_partner()` for database seeding

**Test Infrastructure (used as patterns, not modified):**
- `tests/conftest.py` — test app setup pattern
- `tests/test.env` — test environment configuration
- `tests/utils.py` — helper functions for user/alias creation
- `tests/example_emls/replacement_on_forward_phase.eml` — email fixture template
- `tests/handler/test_preserved_headers.py` — pattern for `store_emails_test_decorator` usage
- `tests/test_email_handler.py` — existing forward/reply test patterns

**Database Schema (read and queried, not altered):**
- `migrations/` — Alembic migrations applied to create schema in test database
- Tables: `users`, `alias`, `contact`, `email_log`, `message_id_matching`, `mailbox`, `sl_domain`, `partner`, `user_audit_log`

**Temporary Infrastructure (created and destroyed):**
- PostgreSQL container (port 15432, database `test`)
- Redis container (default port 6379)

### 0.6.2 Explicitly Out of Scope

- Modification of any existing source file in the repository
- Reply-phase PGP encryption testing (not part of the four diagnostic questions)
- Bounce handling investigation (`handle_bounce_forward_phase()`, `handle_bounce_reply_phase()`)
- Provider complaint processing (Hotmail/Yahoo complaint handlers)
- DMARC quarantine/reject behavior testing (only pass-through is relevant)
- SpamAssassin integration testing (spam scoring disabled in test config)
- Unsubscribe flow investigation
- Performance optimization or code refactoring
- Any changes to Dockerfile, CI/CD, or deployment configuration
- Frontend/UI components
- API endpoint testing (the investigation uses direct Python function invocation)

## 0.7 Rules for Feature Addition

### 0.7.1 SWE-AtlasQnA-Repo Rule

The governing implementation rule `SWE-AtlasQnA-Repo` mandates:

- **Create** a new markdown document named `app_2cd6ee777f8c.md` (derived from the source branch name `app_2cd6ee777f8c`) that comprehensively answers all question(s) posed in the prompt
- **Build and run** the source code to analyze the repository behavior as needed — do not make assumptions; base answers on the code as the truth
- **Provide thinking/rationale** behind the answers
- **Do not modify** any existing files in the source repository
- **Do not add** any other code in the source repository besides the above requested document
- **Place** the generated document in the `blitzy/documentation` directory in the destination repo

### 0.7.2 User-Specified Operational Constraints

- **No Source Modification**: "Just don't modify any source files while investigating" — Zero changes to any `.py`, `.eml`, `.env`, `.toml`, `.yml`, or any other existing file
- **Cleanup Requirement**: "clean up any test containers or DB instances you spin up" — All Docker containers, temporary databases, and runtime artifacts must be destroyed after the investigation completes
- **Actual Runtime Values Required**: "I need to see actual generated values, not what the code logic suggests should happen" — The document must contain real auto-increment IDs, real UTC timestamps, real generated reply-email strings, and real Message-ID values produced by actual code execution
- **Trace Through Actual Operation**: "Trace through an actual email forward operation and capture these specific runtime values" — The investigation must execute `email_handler.handle()` against a real database, not merely describe code paths

### 0.7.3 Investigation Integrity Rules

- Every value reported in the output document must be captured from actual Python runtime execution, not inferred from source code reading
- Database record IDs and timestamps must be queried from PostgreSQL after actual record creation
- Log messages must be captured from the `SL` logger's actual output stream
- The `From` header value must be extracted from the actual `SendRequest` captured by `mail_sender`
- The SL Message-ID must be queried from the actual `MessageIDMatching` database record
- All code paths must be executed within a proper Flask application context (via `create_light_app()`) with a real PostgreSQL backend

## 0.8 References

### 0.8.1 Repository Files and Folders Searched

The following files and folders were systematically inspected to derive the conclusions in this Agent Action Plan:

**Root-Level Files:**
- `email_handler.py` — Full content reviewed (2404 lines): SMTP handler, forward/reply/bounce logic, MailHandler class
- `pyproject.toml` — Full content reviewed: Poetry dependencies, Python ^3.10, all package versions
- `server.py` — Searched for `create_light_app()` definition (line 127)
- `init_app.py` — Full content reviewed (74 lines): SL domain seeding, PGP key loading, Proton partner setup
- `example.env` — Referenced for environment variable documentation
- `alembic.ini` — Referenced for migration configuration

**Application Core (`app/`):**
- `app/config.py` — Reviewed for `EMAIL_DOMAIN`, `BOUNCE_PREFIX`, `BOUNCE_SUFFIX`, `VERP_PREFIX`, `VERP_EMAIL_SECRET`, `NOREPLY`, `DB_URI`, `POSTMASTER` (lines 79–507)
- `app/models.py` — Reviewed for `ModelMixin` (lines 62–130), `VerpType` (lines 247–250), `User` (lines 336+), `Alias` (lines 1469–1770), `Contact` (lines 1863–2058), `EmailLog` (lines 2060–2168), `Mailbox` (lines 2710–2826), `MessageIDMatching` (lines 3365–3379)
- `app/email_utils.py` — Reviewed for `generate_reply_email()` (lines 1103–1153), `sl_formataddr()` (lines 1501–1505), `generate_verp_email()` (lines 1438–1464), `get_verp_info_from_email()` (lines 1467–1498), `is_reverse_alias()` (lines 1156–1163)
- `app/contact_utils.py` — Full content reviewed (121 lines): `create_contact()`, `ContactCreateResult`, audit log emission
- `app/mail_sender.py` — Reviewed for `MailSender` class (lines 96–180), `store_emails_test_decorator` (lines 111–121), `sl_sendmail()` (line 270)
- `app/log.py` — Full content reviewed (79 lines): Logger setup, `set_message_id()`, `EmailHandlerFilter`, log format string
- `app/db.py` — Full content reviewed (18 lines): SQLAlchemy engine, connection, scoped Session
- `app/email/status.py` — Full content reviewed (64 lines): All E-codes (E200, E515, E502, etc.)
- `app/email/headers.py` — Full content reviewed (59 lines): All header constants including SL custom headers

**Handler Subpackage (`app/handler/`):**
- `app/handler/__init__.py` — Empty package marker
- `app/handler/dmarc.py` — Summary reviewed: DMARC policy engine for forward and reply phases
- `app/handler/spamd_result.py` — Summary reviewed: Rspamd header parsing and DMARC verdict extraction
- `app/handler/unsubscribe_generator.py` — Summary reviewed: List-Unsubscribe header generation
- `app/handler/unsubscribe_handler.py` — Summary reviewed: Inbound unsubscribe request processing
- `app/handler/provider_complaint.py` — Summary reviewed: Yahoo/Hotmail complaint routing

**Tests (`tests/`):**
- `tests/conftest.py` — Full content reviewed (77 lines): Test app setup, `flask_client` fixture, database rollback
- `tests/test.env` — Full content reviewed (80 lines): Test environment variables
- `tests/utils.py` — Full content reviewed (92 lines): `create_new_user()`, `load_eml_file()`, `random_email()`
- `tests/test_email_handler.py` — Reviewed lines 1–280: Forward/reply test patterns, DMARC tests
- `tests/handler/test_preserved_headers.py` — Full content reviewed (75 lines): `store_emails_test_decorator` pattern
- `tests/handler/` — Folder summary reviewed: Test coverage for DMARC, PGP, complaints, unsubscribe
- `tests/example_emls/replacement_on_forward_phase.eml` — Full content reviewed: Templated EML fixture

**Folders Explored:**
- Root (`""`) — Full folder listing with summary
- `app/` — Full folder listing with detailed summary
- `app/handler/` — Full folder listing with summary
- `tests/` — Full folder listing with summary
- `tests/handler/` — Full folder listing with summary
- `tests/example_emls/` — File listing

### 0.8.2 Tech Spec Sections Referenced

- **4.2 Core Email Processing Workflows** — Retrieved and reviewed for forward phase flow (4.2.2), reply phase flow (4.2.3), and bounce handling (4.2.4)

### 0.8.3 Attachments

No user attachments were provided for this project. No Figma screens or external URLs were referenced.

