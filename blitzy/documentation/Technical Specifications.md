# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create a new investigative documentation artifact** that traces the exact runtime behavior of SimpleLogin's email forwarding pipeline—capturing specific log messages, generated identifiers, header transformations, and database records produced during a single alias forward operation.

**Category:** Create new documentation
**Documentation Type:** Runtime behavior trace / Investigative Q&A document

The user is investigating a production issue where emails forwarded through SimpleLogin aliases exhibit inconsistent behavior. Rather than modifying code or speculating from static analysis, the user requires a precise, code-grounded trace that answers four specific runtime questions:

- **Q1 – Log Messages:** What is the exact log message text that appears when an email is successfully forwarded versus when it fails due to a non-existent alias?
- **Q2 – SL Message-ID Generation:** What specific SL Message-ID gets generated during forwarding, and how does it differ from the original Message-ID?
- **Q3 – From Header Transformation:** What is the exact `From` header value in the forwarded email after transformation, including the reply-email address format?
- **Q4 – Database Records:** What database records are created during a single forward operation—show actual record IDs and timestamps?

The user explicitly requests **actual generated values** derived from tracing through the code paths, not abstract descriptions of what the logic "should" produce. The output document must be named `app_2cd6ee777f8c.md` and placed in the `blitzy/documentation` directory of the destination repository.

### 0.1.2 Special Instructions and Constraints

- **CRITICAL: No source file modifications.** The user explicitly states: "Just don't modify any source files while investigating." This aligns with the implementation rule `SWE-AtlasQnA-Repo` which mandates: "Do not modify any existing files in the source repository."
- **Clean up test artifacts:** Any test containers or database instances spun up during investigation must be cleaned up.
- **Output format:** A single markdown document named `app_2cd6ee777f8c.md` placed in `blitzy/documentation/`.
- **Answers must be grounded in code as truth.** Per the implementation rule: "Do not make assumptions, base your answers on the code as the truth."
- **Include reasoning/rationale.** Per the implementation rule: "Provide thinking / rationale behind the answers."
- **Actual generated values:** The user wants to see concrete example values (e.g., actual Message-ID strings, actual log line formats, actual SQL record shapes) that the code path produces, derived deterministically from code analysis—not speculative descriptions.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **answer Q1 (log messages)**, we will trace the `handle_forward()` function in `email_handler.py` (lines 536–676) and `forward_email_to_mailbox()` (lines 679–928), identifying every `LOG.d()`, `LOG.i()`, and `LOG.w()` call on both the success path (returns `status.E200 = "250 Message accepted for delivery"`) and the failure path for non-existent aliases (returns `status.E515 = "550 SL E515 Email not exist"`). The log format is defined in `app/log.py` as `"%(asctime)s - %(name)s - %(levelname)s - %(process)d - \"%(pathname)s:%(lineno)d\" - %(funcName)s() - %(message_id)s - %(message)s"`.
- To **answer Q2 (SL Message-ID)**, we will trace `replace_sl_message_id_by_original_message_id()` (forward phase, lines 931–963) and `replace_original_message_id()` (reply phase, lines 1296–1361), documenting how `make_msgid(str(email_log.id), domain)` from Python's `email.utils` generates the SL Message-ID and how `MessageIDMatching` records map between original and SL identifiers.
- To **answer Q3 (From header transformation)**, we will trace `Contact.new_addr()` in `app/models.py` (lines 2008–2046) and the `generate_reply_email()` function in `app/email_utils.py` (lines 1103–1153) to document the exact format of the reverse-alias address and the `From` header rewrite at `email_handler.py` line 866.
- To **answer Q4 (database records)**, we will catalog every `Model.create()` call in the forward path: `Contact.create()` / `contact_utils.create_contact()`, `EmailLog.create()` (line 732), and any `MessageIDMatching.create()` or `UserAuditLog` entries, including their column schemas and auto-populated fields (`id`, `created_at`, `updated_at`).

### 0.1.4 Inferred Documentation Needs

Based on code analysis:

- The forward phase calls `replace_sl_message_id_by_original_message_id()` at line 860, which **restores** original Message-IDs in `In-Reply-To` and `References` headers—this is the forward direction (replacing SL IDs back to originals). The SL Message-ID is generated during the **reply** phase in `replace_original_message_id()`. This distinction is critical and must be documented clearly to avoid confusion about when SL Message-IDs are created versus when they are consumed.
- The `Contact` record creation involves a `reply_email` field generated by `generate_reply_email()`, which produces a randomized reverse-alias address. This address appears in the `From` header of the forwarded email. The format varies depending on the user's `include_sender_in_reverse_alias` setting and `sender_format` preference.
- The `EmailLog` record is the primary audit trail for each forwarded email, linking `contact_id`, `alias_id`, `user_id`, `mailbox_id`, and `message_id`. Its `id` field is auto-incrementing and its `created_at` is set to `arrow.utcnow()`.
- The VERP envelope sender (`generate_verp_email()`) is also a generated value that encodes the `VerpType`, `email_log.id`, and a timestamp, signed with HMAC—this is relevant for understanding the complete forwarding operation.


## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a documentation structure concentrated in the `docs/` directory with operational runbooks, no automated documentation generator, and no existing runtime-trace documentation.

**Documentation files discovered:**

| File | Type | Relevance |
|------|------|-----------|
| `docs/troubleshooting.md` | Operational runbook | Directly relevant — covers email forwarding diagnostics but only at the infrastructure level (Postfix connectivity, container health), not at the application code-path level |
| `docs/api.md` | API reference | Tangentially relevant — documents REST API endpoints, not internal email processing |
| `docs/ses.md` | Relay configuration | Low relevance — Amazon SES outbound relay setup |
| `docs/gmail-relay.md` | Relay configuration | Low relevance — Gmail SMTP relay setup |
| `docs/enforce-spf.md` | Security hardening | Low relevance — SPF enforcement configuration |
| `docs/ssl.md` | Security guide | Low relevance — TLS/certificate configuration |
| `docs/postfix-tls.md` | Security guide | Low relevance — Postfix TLS configuration |
| `docs/build-image.md` | Build guide | Low relevance — Docker image build process |
| `docs/upgrade.md` | Upgrade runbook | Low relevance — Version upgrade procedures |
| `docs/code-structure.md` | Code structure note | Marginally relevant — describes `local_data/` only |
| `docs/oauth.md` | Protocol reference | Low relevance — OAuth2/OIDC flows |
| `docs/ufw.md` | Firewall note | No relevance |
| `README.md` | Project overview | Context only — describes the project at a high level |
| `CONTRIBUTING.md` | Contributor guide | Context only |
| `SECURITY.md` | Vulnerability disclosure | No relevance |

**Current documentation framework:** None detected. The `docs/` directory contains standalone Markdown files with no documentation generator configuration (no `mkdocs.yml`, `docusaurus.config.js`, `sphinx/conf.py`, or `.readthedocs.yml` found).

**API documentation tools in use:** No automated API doc generation detected (no JSDoc, Sphinx, or similar configurations). The `docs/api.md` file is hand-authored.

**Diagram tools detected:** No Mermaid or PlantUML configurations in the repository. The tech spec (section 4.2) uses Mermaid diagrams, but these are in the specification layer, not in the repository's own docs.

**Key finding:** There is no existing documentation that traces the internal email processing flow at the code-path level. The `docs/troubleshooting.md` file addresses infrastructure-level diagnostics (Postfix connectivity, container logs) but does not document what happens inside `email_handler.py` during forward operations. This confirms that the requested document is entirely new content.

### 0.2.2 Repository Code Analysis for Documentation

Search patterns used to identify code relevant to the documentation:

- **Primary email handler:** `email_handler.py` (root level, 2405 lines) — the SMTP inbound processor containing `handle()`, `handle_forward()`, `forward_email_to_mailbox()`, and all message transformation logic
- **Email status codes:** `app/email/status.py` — all SMTP response codes (E200–E525) used during forwarding
- **Email headers constants:** `app/email/headers.py` — all header name constants including `MESSAGE_ID`, `FROM`, `TO`, `SL_DIRECTION`, `SL_EMAIL_LOG_ID`
- **Data models:** `app/models.py` — `Alias` (line 1469), `Contact` (line 1863), `EmailLog` (line 2060), `MessageIDMatching` (line 3365), `Mailbox` (line 2710), `Bounce` (line 3290), `ModelMixin` (line 62 — defines `id`, `created_at`, `updated_at` fields)
- **Contact creation:** `app/contact_utils.py` — `create_contact()` function with `ContactCreateResult` dataclass
- **Email utilities:** `app/email_utils.py` — `generate_reply_email()` (line 1103), `generate_verp_email()` (line 1438), `sl_formataddr()` (line 1501)
- **Logging infrastructure:** `app/log.py` — `LOG` singleton, `_log_format` template, `set_message_id()` correlation, logger level shortcuts (`d`=debug, `i`=info, `w`=warning, `e`=exception)
- **Mail sender:** `app/mail_sender.py` — `sl_sendmail()` function (line 270) wrapping SMTP delivery
- **Alias auto-creation:** `app/alias_utils.py` — `try_auto_create()` (line 202) for catch-all/directory aliases
- **String utilities:** `app/utils.py` — `random_string()` (line 41), `convert_to_id()` (line 50), `convert_to_alphanumeric()` (line 62)
- **Configuration:** `app/config.py` — `EMAIL_DOMAIN`, `BOUNCE_PREFIX`, `VERP_PREFIX`, `NOREPLY`, and related constants
- **Test files:** `tests/test_email_handler.py` — test patterns for forward/reply phases

**Key directories examined:** root (`/`), `app/`, `app/email/`, `app/handler/`, `docs/`, `tests/`

### 0.2.3 Web Search Research Conducted

No web search was required for this documentation task. The investigation is entirely code-grounded per the user's explicit instruction: "base your answers on the code as the truth." All runtime values and behaviors are derived directly from static analysis of the repository source code.


## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The documentation must trace four distinct aspects of the forward operation. Each maps to specific code modules:

**Module: `email_handler.py` (root)**
- Public functions: `handle()`, `handle_forward()`, `forward_email_to_mailbox()`, `get_or_create_contact()`, `replace_sl_message_id_by_original_message_id()`, `replace_header_when_forward()`
- Current documentation: None at the code-trace level
- Documentation needed: Complete runtime trace with exact log messages, branching decisions, and generated values for both success and failure paths

**Module: `app/email/status.py`**
- Constants: `E200` ("250 Message accepted for delivery"), `E515` ("550 SL E515 Email not exist"), `E207` ("250 SL E207 No bounce report")
- Current documentation: Inline comments only
- Documentation needed: Mapping of which status codes appear on which code paths during the forward operation

**Module: `app/models.py` — `Contact` class (line 1863)**
- Methods: `new_addr()` (line 2008) — generates the transformed `From` header value
- Fields: `id`, `user_id`, `alias_id`, `website_email`, `reply_email`, `name`, `created_at`
- Current documentation: Docstrings only
- Documentation needed: Exact format of `new_addr()` output with concrete examples based on `SenderFormatEnum` values

**Module: `app/models.py` — `EmailLog` class (line 2060)**
- Fields: `id`, `contact_id`, `user_id`, `alias_id`, `mailbox_id`, `message_id`, `sl_message_id`, `is_reply`, `blocked`, `bounced`, `created_at`
- Current documentation: Inline comments only
- Documentation needed: Complete record shape with example values showing what gets created during a forward operation

**Module: `app/models.py` — `MessageIDMatching` class (line 3365)**
- Fields: `id`, `sl_message_id`, `original_message_id`, `email_log_id`, `created_at`
- Current documentation: One-line docstring
- Documentation needed: Clarification that this record is created during the **reply** phase, not the forward phase, with explanation of how it is consumed during forward phase via `replace_sl_message_id_by_original_message_id()`

**Module: `app/contact_utils.py`**
- Function: `create_contact()` (line 42) — orchestrates Contact record creation with reply email generation
- Current documentation: No external documentation
- Documentation needed: Trace of the contact creation flow including `generate_reply_email()` call and `UserAuditLog` emission

**Module: `app/email_utils.py`**
- Functions: `generate_reply_email()` (line 1103), `generate_verp_email()` (line 1438), `sl_formataddr()` (line 1501)
- Current documentation: Docstrings only
- Documentation needed: Concrete examples of generated values with format patterns

**Module: `app/log.py`**
- Log format: `"%(asctime)s - %(name)s - %(levelname)s - %(process)d - \"%(pathname)s:%(lineno)d\" - %(funcName)s() - %(message_id)s - %(message)s"`
- Logger shortcuts: `LOG.d` = debug, `LOG.i` = info, `LOG.w` = warning, `LOG.e` = exception
- Current documentation: None
- Documentation needed: Exact log line format with example output

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **No runtime trace documentation exists** for any code path in the email handler. The existing `docs/troubleshooting.md` addresses infrastructure-level issues (Postfix connectivity) but never discusses application-level behavior such as log messages, header transformations, or database record creation.
- **No documentation of SL Message-ID lifecycle** — how Message-IDs are generated, stored in `MessageIDMatching`, and swapped between phases is completely undocumented.
- **No documentation of the `From` header transformation** — the `Contact.new_addr()` method and its dependence on `SenderFormatEnum` and `generate_reply_email()` is undocumented outside of inline docstrings.
- **No documentation of database records created per forward operation** — the combination of `Contact`, `EmailLog`, and optionally `UserAuditLog` records created during a single forward is not documented anywhere.
- **No documentation mapping log messages to specific code paths** — the relationship between log level/message text and the success/failure outcome of a forward operation is entirely undocumented.


## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output is a single investigative Q&A markdown document structured to trace through the email forward operation and answer each of the user's four questions with code-grounded evidence.

```
blitzy/documentation/
└── app_2cd6ee777f8c.md
    ├── Introduction (investigation context and approach)
    ├── Q1: Log Messages (success vs failure paths)
    │   ├── Successful Forward — exact log lines
    │   ├── Failed Forward (non-existent alias) — exact log lines
    │   └── Log format reference
    ├── Q2: SL Message-ID Generation
    │   ├── Forward phase — Message-ID passthrough behavior
    │   ├── Reply phase — SL Message-ID creation via make_msgid()
    │   ├── MessageIDMatching record lifecycle
    │   └── Example generated values
    ├── Q3: From Header Transformation
    │   ├── Contact creation and reply-email generation
    │   ├── Contact.new_addr() format by SenderFormatEnum
    │   ├── From header rewrite in forward_email_to_mailbox()
    │   └── Example transformed From header values
    ├── Q4: Database Records Created
    │   ├── Contact record (schema and example)
    │   ├── EmailLog record (schema and example)
    │   ├── UserAuditLog record (schema and example)
    │   ├── VERP envelope sender (generated value)
    │   └── Complete record timeline for a single forward
    └── Summary and Rationale
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**
- Extract exact log messages from `LOG.d()`, `LOG.i()`, and `LOG.w()` calls in `email_handler.py` by tracing the code path line-by-line for both the success and failure scenarios
- Generate concrete example values by applying `generate_reply_email()` logic (from `app/email_utils.py:1103`) with known inputs: a contact email, an alias, and the `random_string()` function's character set (`string.ascii_lowercase`, length 20–50)
- Derive SL Message-ID format from `email.utils.make_msgid(str(email_log.id), domain)` which produces `<email_log.id.random_hex@domain>` per Python stdlib
- Reconstruct `Contact.new_addr()` output by applying `SenderFormatEnum.AT` (default) formatting logic to a sample contact
- Derive database record shapes from `ModelMixin` (auto-increment `id`, `created_at` as Arrow/UTC, nullable `updated_at`) combined with each model's column definitions

**Documentation Standards:**
- Markdown formatting with proper headers (`#`, `##`, `###`)
- Code examples using fenced code blocks with language hints
- Source citations as inline references: `Source: email_handler.py:line_number`
- Tables for database record schemas and field values
- Mermaid sequence diagram for the complete forward operation flow

### 0.4.3 Diagram and Visual Strategy

**Mermaid diagrams to create:**

- **Sequence diagram:** End-to-end forward operation showing the interaction between `handle()` → `handle_forward()` → `forward_email_to_mailbox()` → `sl_sendmail()`, with database write points and log emission points annotated
- **Decision flowchart:** The alias resolution branch showing the path to success (`E200`) versus the path to `E515` (non-existent alias)

These diagrams will be embedded directly in the output markdown document using fenced `mermaid` code blocks.


## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | CREATE | `email_handler.py`, `app/models.py`, `app/email/status.py`, `app/email/headers.py`, `app/email_utils.py`, `app/contact_utils.py`, `app/log.py`, `app/mail_sender.py`, `app/utils.py`, `app/config.py`, `app/alias_utils.py` | Complete investigative Q&A document tracing the email forward operation with exact log messages, SL Message-ID generation, From header transformation, and database record creation with concrete example values |

**Transformation Modes Used:** CREATE only. No existing documentation files are updated or deleted. No existing source files are modified.

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/app_2cd6ee777f8c.md
Type: Investigative Q&A / Runtime Behavior Trace
Source Code:
  - email_handler.py (primary — handle(), handle_forward(), forward_email_to_mailbox())
  - app/models.py (Contact, EmailLog, MessageIDMatching, Alias, Mailbox, ModelMixin)
  - app/email/status.py (E200, E515, E207 status codes)
  - app/email/headers.py (MESSAGE_ID, FROM, SL_DIRECTION, SL_EMAIL_LOG_ID constants)
  - app/email_utils.py (generate_reply_email, generate_verp_email, sl_formataddr)
  - app/contact_utils.py (create_contact, ContactCreateResult)
  - app/log.py (LOG format, set_message_id, EmailHandlerFilter)
  - app/mail_sender.py (sl_sendmail, SendRequest)
  - app/utils.py (random_string, convert_to_id, convert_to_alphanumeric)
  - app/config.py (EMAIL_DOMAIN, VERP_PREFIX, BOUNCE_PREFIX)
  - app/alias_utils.py (try_auto_create)
Sections:
  - Introduction (investigation context, methodology, codebase overview)
  - Q1: Log Messages for Success vs Failure
    - Exact log format template from app/log.py
    - Line-by-line trace of LOG calls on success path
    - Line-by-line trace of LOG calls on failure path (non-existent alias)
  - Q2: SL Message-ID Generation and Lifecycle
    - Forward phase: replace_sl_message_id_by_original_message_id() behavior
    - Reply phase: replace_original_message_id() with make_msgid()
    - MessageIDMatching record creation and lookup
    - Concrete example values
  - Q3: From Header Transformation
    - Contact creation via contact_utils.create_contact()
    - reply_email generation via generate_reply_email()
    - Contact.new_addr() formatting by SenderFormatEnum
    - Concrete example From header values
  - Q4: Database Records Created During Forward
    - Contact record schema and example
    - EmailLog record schema and example
    - UserAuditLog record (emitted by contact_utils)
    - VERP envelope sender example
    - Timeline of all writes in a single forward operation
  - Summary and Rationale
Diagrams:
  - Sequence diagram: complete forward operation with DB writes and log points
  - Flowchart: alias resolution success vs failure paths
Key Citations:
  - email_handler.py:536-928 (handle_forward, forward_email_to_mailbox)
  - app/models.py:1863-2058 (Contact), :2060-2167 (EmailLog), :3365-3379 (MessageIDMatching)
  - app/email_utils.py:1103-1153 (generate_reply_email)
  - app/log.py:12-15 (log format)
  - app/email/status.py:1-64 (all status codes)
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration files require updates. The repository uses no documentation generator framework. The output file is a standalone Markdown document placed in the `blitzy/documentation/` directory.

### 0.5.4 Cross-Documentation Dependencies

- **No shared content/includes:** The output document is self-contained.
- **No navigation links:** No documentation navigation system exists in the repository.
- **No table of contents updates:** No centralized documentation index exists.
- **No index/glossary updates:** No glossary or index file exists.
- The document will reference source file paths throughout for traceability, but these are citations, not hyperlinks requiring maintenance.


## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

This documentation task is a code-analysis exercise that produces a standalone Markdown file. No documentation generation tools, build systems, or external packages are required for the output artifact. However, the following project dependencies are relevant to understanding the code being documented:

| Registry | Package Name | Version | Purpose in Documentation Context |
|----------|--------------|---------|----------------------------------|
| pip (Poetry) | flask | ^1.1.2 | Application framework — provides the `app_context()` used during email handling |
| pip (Poetry) | aiosmtpd | ^1.2 | SMTP server — provides `Envelope` class used in `handle()` and `handle_forward()` |
| pip (Poetry) | SQLAlchemy | 1.3.24 | ORM — defines all model classes (`Contact`, `EmailLog`, `MessageIDMatching`) being documented |
| pip (Poetry) | arrow | ^0.16.0 | Timestamp library — `created_at` and `updated_at` fields use `ArrowType` with `arrow.utcnow` |
| pip (Poetry) | dkimpy | ^1.0.5 | DKIM signing — `add_dkim_signature()` called in the forward path |
| pip (Poetry) | flanker | ^0.9.11 | Email address parsing — used in `replace_header_when_forward()` |
| pip (Poetry) | email_validator | ^1.1.1 | Email validation — used in contact creation and alias validation |
| pip (Poetry) | newrelic | 8.8.0 | APM — `@newrelic.agent.background_task()` decorates `_handle()` and custom metrics are recorded |
| pip (Poetry) | python-gnupg | ^0.4.6 | PGP encryption — optional path in forward, documented as conditional branch |
| pip (Poetry) | PGPy | 0.5.4 | PGP fallback — used when python-gnupg fails |
| stdlib | email.utils | (stdlib) | `make_msgid()` — generates SL Message-IDs in the format `<idstring.randomhex@domain>` |
| stdlib | uuid | (stdlib) | `uuid.uuid4()` — generates correlation message IDs in `_handle()` |

**Python version:** ^3.10 (from `pyproject.toml` `tool.poetry.dependencies.python`)

### 0.6.2 Documentation Reference Updates

Not applicable. No existing documentation files require link updates. The output document is a new standalone file with no inbound references from existing documentation.


## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Current coverage analysis:**
- Email forward success path documented: 0/1 (0%) — no existing documentation traces the success path through `handle_forward()` → `forward_email_to_mailbox()` → `E200`
- Email forward failure path documented: 0/1 (0%) — no existing documentation traces the non-existent alias path to `E515`
- SL Message-ID lifecycle documented: 0/1 (0%) — no existing documentation describes the `MessageIDMatching` table or `make_msgid()` usage
- From header transformation documented: 0/1 (0%) — no existing documentation covers `Contact.new_addr()` formatting
- Database record creation per forward documented: 0/1 (0%) — no existing documentation catalogs the records created in a single forward operation

**Target coverage:** 100% — all four user questions must be answered completely with code-grounded evidence.

**Coverage gaps to address:**

| Area | Current | Target | Focus |
|------|---------|--------|-------|
| Forward success log trace | 0% | 100% | Every `LOG.*()` call on the success path with exact message text |
| Forward failure log trace | 0% | 100% | Every `LOG.*()` call on the `E515` failure path with exact message text |
| SL Message-ID lifecycle | 0% | 100% | Generation format, storage, and cross-phase lookup |
| From header format | 0% | 100% | `Contact.new_addr()` output for each `SenderFormatEnum` value |
| Database records per forward | 0% | 100% | Complete catalog of records with schemas and example values |

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every user question (Q1–Q4) is answered with specific code references and concrete example values
- All code paths are traced to their terminal status code
- All database writes are identified with table name, column list, and example values
- All log messages include the exact format string and substituted values

**Accuracy validation:**
- Every claim references a specific file and line number in the repository
- Example values are derived from the actual code logic (e.g., `random_string()` character set, `make_msgid()` format, `SenderFormatEnum` formatting rules), not invented
- Status codes are quoted verbatim from `app/email/status.py`
- Log format is quoted verbatim from `app/log.py`

**Clarity standards:**
- Technical accuracy with the reasoning/rationale behind each answer (per implementation rule)
- Progressive structure: each question answered independently with self-contained context
- Code snippets kept short (2–3 lines) to illustrate key transformation points
- Mermaid diagrams to visualize the forward operation sequence

**Maintainability:**
- Source citations as `Source: filename:line_number` throughout
- Clear section structure matching the four user questions

### 0.7.3 Example and Diagram Requirements

- **Minimum examples per answer:** At least one concrete example value per question (e.g., a sample log line, a sample Message-ID, a sample From header, a sample database record)
- **Diagram types required:** Sequence diagram (forward operation flow), flowchart (alias resolution success vs failure)
- **Code example scope:** Short inline snippets showing the exact code that produces each value (2–3 lines maximum)


## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `blitzy/documentation/app_2cd6ee777f8c.md` — the sole output artifact

**Source code analyzed (read-only, for documentation derivation):**
- `email_handler.py` — primary subject: `handle()`, `handle_forward()`, `forward_email_to_mailbox()`, `get_or_create_contact()`, `replace_sl_message_id_by_original_message_id()`, `replace_header_when_forward()`
- `app/models.py` — `ModelMixin`, `Alias`, `Contact`, `EmailLog`, `MessageIDMatching`, `Mailbox`, `SenderFormatEnum`, `VerpType`, `BlockBehaviourEnum`
- `app/email/status.py` — all SMTP status codes referenced in forward paths
- `app/email/headers.py` — all header constants used in header rewriting
- `app/email_utils.py` — `generate_reply_email()`, `generate_verp_email()`, `sl_formataddr()`, `is_reverse_alias()`, `add_or_replace_header()`, `delete_all_headers_except()`
- `app/contact_utils.py` — `create_contact()`, `ContactCreateResult`
- `app/log.py` — `LOG`, `_log_format`, `set_message_id()`, `EmailHandlerFilter`
- `app/mail_sender.py` — `sl_sendmail()`, `SendRequest`
- `app/utils.py` — `random_string()`, `convert_to_id()`, `convert_to_alphanumeric()`, `sanitize_email()`
- `app/config.py` — `EMAIL_DOMAIN`, `VERP_PREFIX`, `BOUNCE_PREFIX`, `BOUNCE_SUFFIX`, `VERP_EMAIL_SECRET`
- `app/alias_utils.py` — `try_auto_create()`, `try_auto_create_directory()`, `try_auto_create_via_domain()`
- `app/handler/unsubscribe_generator.py` — `UnsubscribeGenerator.add_header_to_message()` (called in forward path)
- `app/handler/dmarc.py` — `apply_dmarc_policy_for_forward_phase()` (called in forward path)
- `app/user_audit_log_utils.py` — `emit_user_audit_log()` (called during contact creation)
- `pyproject.toml` — dependency versions and Python version constraint
- `tests/test_email_handler.py` — test patterns confirming expected behavior

**Documentation topics covered:**
- Exact log messages on forward success path (`E200`)
- Exact log messages on forward failure path for non-existent alias (`E515`)
- SL Message-ID generation mechanics and format
- Original vs SL Message-ID distinction across forward and reply phases
- From header transformation via `Contact.new_addr()` with all `SenderFormatEnum` variants
- Reply-email address format from `generate_reply_email()`
- Complete catalog of database records created during a single forward operation
- VERP envelope sender format from `generate_verp_email()`

### 0.8.2 Explicitly Out of Scope

- **Source code modifications:** No files in the repository will be modified, per user instruction ("don't modify any source files") and implementation rule ("Do not modify any existing files in the source repository")
- **Reply phase tracing:** The user specifically asks about the **forward** phase; the reply phase is referenced only to explain SL Message-ID creation context
- **Bounce handling tracing:** Bounce processing (`handle_bounce_forward_phase()`, `handle_bounce_reply_phase()`) is not traced
- **Spam filtering details:** SpamAssassin scoring is mentioned as a conditional branch but not deeply traced
- **PGP encryption details:** PGP-related code paths are noted as conditional but not the focus of the trace
- **DMARC policy details:** DMARC enforcement is noted as a gate in the forward path but not deeply traced
- **Infrastructure-level diagnostics:** Postfix configuration, container networking, and SMTP relay setup are out of scope (already covered by `docs/troubleshooting.md`)
- **Test file modifications:** No test files are modified
- **Feature additions or refactoring:** No code changes of any kind
- **Documentation for other email phases:** Only the forward phase (alias-to-mailbox) is traced in depth
- **Deployment or CI/CD changes:** No operational changes


## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** Not applicable — the output is a standalone Markdown file requiring no build step
- **Documentation preview command:** Any Markdown renderer (e.g., `grip blitzy/documentation/app_2cd6ee777f8c.md`, GitHub's built-in renderer, or VS Code preview)
- **Diagram generation command:** Mermaid diagrams are embedded as fenced code blocks; they render natively in GitHub, GitLab, and most modern Markdown renderers without a separate generation step
- **Documentation deployment command:** Not applicable — file is committed directly to the repository
- **Default format:** Markdown with Mermaid diagrams
- **Citation requirement:** Every technical claim references a source file and line number
- **Style guide:** The document follows the investigative Q&A format specified by the `SWE-AtlasQnA-Repo` implementation rule:
  - Comprehensive answers with reasoning/rationale
  - Grounded in the code as truth
  - No assumptions
  - No modifications to existing files
- **Documentation validation:** Manual review — verify that all cited line numbers correspond to the claimed code, and that all example values are consistent with the code logic
- **Cleanup requirement:** Any test containers or database instances created during investigation must be torn down. Since this is a code-analysis documentation task (no runtime execution required), no cleanup is expected to be necessary


## 0.10 Rules for Documentation

The following rules govern the documentation generation for this task, derived from user instructions and implementation rules:

- **Do not modify any existing files in the source repository.** The output is a new file only (`blitzy/documentation/app_2cd6ee777f8c.md`). No source code, configuration, test, or existing documentation files may be altered.
- **Do not make assumptions — base answers on the code as truth.** Every claim in the output document must be traceable to a specific file and line number in the repository. Speculative behavior ("the code should do X") is not acceptable; only demonstrable behavior ("the code at line Y does X") is permitted.
- **Provide thinking and rationale behind the answers.** Each answer must explain *why* the code produces the documented behavior, not just *what* it produces. This means tracing the call chain, explaining branching conditions, and citing the controlling logic.
- **Show actual generated values, not code logic descriptions.** The user explicitly states: "I need to see actual generated values, not what the code logic suggests should happen." The document must include concrete example values (e.g., a fully rendered log line, a complete Message-ID string, a formatted From header, database record JSON) derived deterministically from the code.
- **Clean up any test containers or database instances.** If any runtime environment is created during investigation, it must be torn down. Since this task is purely analytical (static code tracing), no runtime artifacts are expected.
- **Place the generated document in `blitzy/documentation/`.** The file must be named `app_2cd6ee777f8c.md` per the implementation rule naming convention (`<source_branch_name>.md`).
- **Use only dashes (`-`) for bullet points in the output.** No numbered bullets.


## 0.11 References

### 0.11.1 Repository Files and Folders Searched

The following files and folders were retrieved and analyzed to derive the conclusions in this Agent Action Plan:

**Primary source files (read in full):**

| File | Lines Read | Purpose |
|------|------------|---------|
| `email_handler.py` | 1–2405 | Primary email processing handler — `handle()`, `handle_forward()`, `forward_email_to_mailbox()`, `replace_sl_message_id_by_original_message_id()`, all log messages, status codes, and header transformations |
| `app/email/status.py` | 1–64 | All SMTP status code constants used in email handler returns |
| `app/email/headers.py` | 1–59 | All email header name constants including SL custom headers |
| `app/log.py` | 1–79 | Log format string, logger initialization, `set_message_id()` correlation, level shortcuts |
| `app/models.py` | 62–100 | `ModelMixin` — base model with `id`, `created_at`, `updated_at` fields |
| `app/models.py` | 203–251 | `SenderFormatEnum`, `VerpType`, `BlockBehaviourEnum` enumerations |
| `app/models.py` | 1469–1600 | `Alias` model — fields, `mailboxes` property, `authorized_addresses()` |
| `app/models.py` | 1863–2058 | `Contact` model — fields, `create()`, `new_addr()`, `website_send_to()` |
| `app/models.py` | 2060–2167 | `EmailLog` model — fields, `create()` override, `get_phase()`, `get_action()` |
| `app/models.py` | 2710–2825 | `Mailbox` model — fields, `pgp_enabled()` |
| `app/models.py` | 3290–3320 | `Bounce` and `TransactionalEmail` models |
| `app/models.py` | 3365–3430 | `MessageIDMatching` model — `sl_message_id`, `original_message_id`, `email_log_id` |
| `app/email_utils.py` | 1103–1153 | `generate_reply_email()` — reverse-alias address generation with randomization |
| `app/email_utils.py` | 1438–1498 | `generate_verp_email()` — VERP envelope sender with HMAC signing |
| `app/email_utils.py` | 1501–1506 | `sl_formataddr()` — RFC-compliant address formatting |
| `app/contact_utils.py` | 1–121 | `create_contact()` — full contact creation flow with audit logging |
| `app/mail_sender.py` | 1–60, 270–290 | `SendRequest` dataclass and `sl_sendmail()` entry point |
| `app/alias_utils.py` | 202–280 | `try_auto_create()`, `try_auto_create_directory()`, `try_auto_create_via_domain()` |
| `app/utils.py` | 41–71 | `random_string()`, `convert_to_id()`, `convert_to_alphanumeric()` |
| `app/config.py` | (grep) | `EMAIL_DOMAIN`, `BOUNCE_PREFIX`, `BOUNCE_SUFFIX`, `VERP_PREFIX`, `VERP_EMAIL_SECRET`, `NOREPLY` |
| `pyproject.toml` | 1–134 | Python version (^3.10), all dependency versions |
| `tests/test_email_handler.py` | 1–35, 262–340 | Test patterns for forward/reply phases confirming expected behavior |

**Folders explored:**

| Folder | Depth | Purpose |
|--------|-------|---------|
| `/` (root) | Level 0 | Repository structure, top-level files |
| `app/` | Level 1 | Core application package — all models, utilities, configuration |
| `app/email/` | Level 2 | Email subsystem — status codes, headers, rate limiting, spam |
| `app/handler/` | Level 2 | Handler subsystem — DMARC, unsubscribe, provider complaints |
| `docs/` | Level 1 | Existing documentation — runbooks, API reference, troubleshooting |
| `tests/` | Level 1 | Test suite — `test_email_handler.py`, handler tests |
| `tests/handler/` | Level 2 | Handler-specific tests |

### 0.11.2 Attachments Provided

No attachments were provided by the user. No Figma screens, images, or supplementary files were included.

### 0.11.3 External References

No external URLs or Figma screens were specified. The entire investigation is grounded in the repository source code.

### 0.11.4 Tech Spec Sections Referenced

- **Section 4.2: Core Email Processing Workflows** — retrieved via `get_tech_spec_section` to cross-reference the forward phase flow documentation against the actual code, confirming alignment between the specification's flowcharts and the code-level trace in `email_handler.py`.


