# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create a new investigative analysis document** that comprehensively traces the alias reply-handling flow in the SimpleLogin email aliasing system, identifies the runtime data flow for inbound email replies, and pinpoints the most likely origin of incorrect routing decisions.

**Category:** Create new documentation
**Documentation Type:** Technical investigation and data-flow analysis document

The user is experiencing unexpected behavior in the alias reply-handling pipeline. Specifically:
- When a user replies to a forwarded email via a reverse-alias, some replies appear to be routed to the wrong user
- The logs indicate the alias is recognized correctly, yet the final forwarding destination is incorrect
- The user wants an end-to-end trace of the runtime flow to understand where routing diverges

The documentation requirements, restated with enhanced clarity, are:

- **Requirement 1 — End-to-end data flow documentation:** Trace an inbound email reply from SMTP reception through the `MailHandler.handle_DATA()` entry point, to the `handle()` routing function, through `is_reverse_alias()` classification, into `handle_reply()`, and all the way to final `sl_sendmail()` delivery — documenting every decision point, database lookup, and header rewrite
- **Requirement 2 — Alias-to-user resolution analysis:** Explain how the system resolves a reverse-alias address (`reply_email` on the `Contact` model) to a specific `Alias`, then to a `User`, and then to the authorized `Mailbox` — including the `get_mailbox_from_mail_from()` authorization logic and the `canonicalize_email()` fallback
- **Requirement 3 — Incorrect routing root-cause analysis:** Based on the traced pipeline, identify the most likely points where an incorrect routing decision could originate, including `Contact.get_by(reply_email=...)` lookups, `alias.mailboxes` property resolution, `get_mailbox_from_mail_from()` authorization matching, email canonicalization edge cases, and the `disable_email_spoofing_check` fallback to default mailbox
- **Requirement 4 — Clean codebase policy:** Any temporary scripts or instrumentation used during the analysis must be cleaned up; the final deliverable is a standalone Markdown document placed in `blitzy/documentation/`

### 0.1.2 Special Instructions and Constraints

- **Implementation Rule (SWE-AtlasQnA-Repo):** Create a new markdown document named `app_2cd6ee777f8c.md` that comprehensively answers the questions posed in the prompt. Provide thinking and rationale behind the answers. Do not make assumptions — base answers on the code as the truth. Do not modify any existing files in the source repository. Place the generated document in the `blitzy/documentation` directory in the destination repo.
- **No existing file modification:** The codebase must remain untouched; all output is a single new document
- **Code-as-truth principle:** All analysis must be grounded in actual source code from the repository, not theoretical assumptions
- **Branch name for the output file:** `app_2cd6ee777f8c` (the current source branch)

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **document the end-to-end reply flow**, we will create a new file `blitzy/documentation/app_2cd6ee777f8c.md` that traces the execution path starting from `MailHandler.handle_DATA()` in `email_handler.py` (line 2289) through `_handle()` (line 2335), into `handle()` (line 1945), through the `is_reverse_alias()` check (line 2195), and into `handle_reply()` (line 966)
- To **document alias-to-user resolution**, we will detail the `Contact.get_by(reply_email=reply_email)` lookup (line 986), the `contact.alias` and `contact.user` traversal (lines 994–1004), the `get_mailbox_from_mail_from()` authorization (line 1019 and lines 1364–1387), and the `Alias.mailboxes` property (lines 1580–1589 in `app/models.py`)
- To **identify incorrect routing origins**, we will analyze the `normalize_reply_email()` normalization (line 984 of `email_handler.py`), the mailbox resolution fallback via `canonicalize_email()` (line 1387), the `disable_email_spoofing_check` fallback to default mailbox (line 1029), and the `Alias.mailboxes` property filtering by verified status (line 1586 of `app/models.py`)

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following implicit documentation needs are surfaced:

- The `Contact` model stores `reply_email` as a unique reverse-alias per contact-alias pair (`app/models.py:1899`). If two contacts for different aliases have overlapping reply email patterns due to normalization, this is a potential routing ambiguity that must be documented
- The `get_mailbox_from_mail_from()` function (lines 1364–1387) performs a two-pass check: first against the raw `mail_from`, then against the canonicalized version. The canonicalization logic in `canonicalize_email()` (`app/utils.py:78–94`) strips dots and plus-suffixes only for Gmail, ProtonMail, Proton.me, and PM.me domains — all other domains are returned as-is. This selective canonicalization is a potential source of mismatches
- The `Alias.mailboxes` property (lines 1580–1589) filters for verified mailboxes only, but uses an identity comparison (`m.id is not self.mailbox.id`) rather than value equality for deduplication, which could produce unexpected results with detached ORM objects. This is a significant finding that warrants documentation
- The `disable_email_spoofing_check` flag on aliases (line 1021) silently falls back to `alias.mailbox` (the primary/default mailbox), bypassing the authorized-address check entirely — this means that if an unauthorized sender sends to a reverse-alias with this flag enabled, the reply will be attributed to the default mailbox owner regardless of the actual sender

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a flat documentation structure under `docs/` with operational runbooks and infrastructure guides, but no existing documentation that explains the internal email-processing pipeline at a code-flow level.

**Documentation files discovered:**

| File | Type | Relevance to This Task |
|------|------|------------------------|
| `README.md` | Self-hosting deployment guide | Low — covers DNS/Docker setup, not internal pipeline logic |
| `CONTRIBUTING.md` | Contributor guidelines | Low — development workflow, not architecture |
| `SECURITY.md` | Vulnerability disclosure | Not relevant |
| `docs/troubleshooting.md` | Operational troubleshooting | Medium — covers forward-phase debugging with swaks/telnet/docker, but has zero coverage of reply-phase routing |
| `docs/api.md` | REST API reference | Low — documents HTTP API endpoints, not SMTP handler internals |
| `docs/oauth.md` | OAuth flow documentation | Not relevant |
| `docs/code-structure.md` | Code layout notes | Low — only a TODO stub documenting `local_data/` directory and JWT key generation |
| `docs/ssl.md` | TLS/certificate guide | Not relevant |
| `docs/enforce-spf.md` | SPF enforcement setup | Low — covers Postfix SPF config, not internal reply routing |
| `docs/ses.md` | Amazon SES relay setup | Not relevant |
| `docs/gmail-relay.md` | Gmail relay setup | Not relevant |
| `docs/postfix-tls.md` | Postfix TLS setup | Not relevant |
| `docs/build-image.md` | Docker image build | Not relevant |
| `docs/upgrade.md` | Version upgrade runbook | Not relevant |
| `docs/ufw.md` | Firewall rules | Not relevant |

**Documentation framework:** None detected — the project uses plain Markdown files with no documentation generator configuration (no `mkdocs.yml`, `docusaurus.config.js`, `sphinx/conf.py`, or `.readthedocs.yml`)

**API documentation tools:** None in use — `docs/api.md` is manually maintained

**Diagram tools:** None detected in the repository; Mermaid diagrams are supported in GitHub-rendered Markdown

**Critical gap identified:** There is no internal architecture documentation explaining the email forwarding and reply pipeline at a code level. The `email_handler.py` file has a brief docstring at lines 1–31 describing the forward and reply phases at a conceptual level, but no standalone document traces the actual runtime flow with code citations.

### 0.2.2 Repository Code Analysis for Documentation

The following source files and directories were examined to understand the reply-handling pipeline:

**Primary handler files:**
- `email_handler.py` (2404 lines) — Central SMTP handler implementing `MailHandler`, `handle()`, `handle_forward()`, `handle_reply()`, and all bounce/complaint handlers
- `app/handler/` — Sub-handlers for DMARC, provider complaints, spamd results, and unsubscribe processing

**Core utility modules:**
- `app/email_utils.py` — Email composition, header manipulation, `generate_reply_email()`, `is_reverse_alias()`, VERP generation
- `app/email_validation.py` — `normalize_reply_email()` for handling non-ASCII and control characters
- `app/contact_utils.py` — `create_contact()` which generates unique `reply_email` values via `generate_reply_email()`
- `app/utils.py` — `canonicalize_email()` and `sanitize_email()` functions

**Data model definitions:**
- `app/models.py` — `Alias` (line 1469), `Contact` (line 1863), `Mailbox` (line 2710), `User` (line 336), `EmailLog` (line 2060), `AuthorizedAddress` (line 3190), `SLDomain` (line 3116), `MessageIDMatching` (line 3365)

**Status and header constants:**
- `app/email/status.py` — SMTP status code registry (E200–E525)
- `app/email/headers.py` — Header name constants
- `app/email/rate_limit.py` — Rate limiting helpers (currently disabled via hardcoded `return False`)

**Test files examined:**
- `tests/test_email_handler.py` — Tests for DMARC reply quarantine, reverse-alias replacement in reply phase, sending from non-canonical addresses

### 0.2.3 Web Search Research Conducted

No external web search was required for this task. The documentation is entirely derived from the source code, which serves as the ground truth per the implementation rule "Do not make assumptions, base your answers on the code as the truth." All technical conclusions are grounded in file-level code inspection of the repository.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The following modules require documentation as part of the alias reply-handling flow analysis:

**Module: `email_handler.py` (root-level)**
- Public APIs: `MailHandler.handle_DATA()`, `handle()`, `handle_forward()`, `handle_reply()`, `get_mailbox_from_mail_from()`, `handle_unknown_mailbox()`, `replace_header_when_reply()`, `forward_email_to_mailbox()`, `get_or_create_contact()`, `replace_original_message_id()`, `notify_mailbox()`
- Current documentation: Minimal — brief module-level docstring (lines 1–31) and sparse function docstrings
- Documentation needed: Complete end-to-end flow trace with line-level citations, decision tree analysis, data flow diagram with Mermaid

**Module: `app/contact_utils.py`**
- Public APIs: `create_contact()` — generates unique `reply_email` via `generate_reply_email()`
- Current documentation: No standalone documentation
- Documentation needed: How contacts are created with unique reverse-alias addresses, the `reply_email` uniqueness guarantee, and the `IntegrityError` retry mechanism

**Module: `app/email_utils.py`**
- Public APIs: `is_reverse_alias()`, `generate_reply_email()`, `normalize_reply_email()`, `generate_verp_email()`, `get_verp_info_from_email()`
- Current documentation: No standalone documentation
- Documentation needed: Reverse-alias detection logic, reply email generation algorithm, domain-selection for reverse aliases

**Module: `app/utils.py`**
- Public APIs: `canonicalize_email()`, `sanitize_email()`
- Current documentation: No standalone documentation
- Documentation needed: Email canonicalization rules (Gmail/Proton-specific dot-stripping and plus-removal) and their impact on mailbox matching

**Module: `app/models.py`**
- Key classes: `Alias.mailboxes` property (line 1580), `Contact.reply_email` field (line 1899), `Contact.new_addr()` method (line 2008), `Mailbox.authorized_addresses` backref (line 3205), `AuthorizedAddress` model (line 3190)
- Current documentation: Inline docstrings only
- Documentation needed: Relationship traversal from `reply_email → Contact → Alias → User → Mailbox`, the verified-only mailbox filtering, and the identity comparison potential issue

**Module: `app/email_validation.py`**
- Public APIs: `normalize_reply_email()` (line 25)
- Current documentation: Brief function docstring
- Documentation needed: Character normalization behavior for non-ASCII reply addresses

**Module: `app/email/status.py`**
- Constants: E200–E525 status codes used throughout the reply flow
- Current documentation: Inline comments only
- Documentation needed: Status code mapping to reply-phase decision points

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

**Undocumented internal flows:**
- No document traces the complete reply-phase data flow from SMTP reception to outbound delivery
- No document explains the alias-to-user resolution chain (`reply_email` → `Contact` → `Alias` → `User`)
- No document covers the mailbox authorization logic in `get_mailbox_from_mail_from()` and its canonicalization fallback
- No document explains the `disable_email_spoofing_check` bypass and its routing implications

**Missing error scenario documentation:**
- The behavior when `Contact.get_by(reply_email=...)` returns `None` (E502) is not documented in any guide
- The `handle_unknown_mailbox()` alert flow is not documented for operators debugging reply failures
- The `NonReverseAliasInReplyPhase` exception path during `replace_header_when_reply()` is not documented

**Undocumented edge cases relevant to misrouting:**
- The `Alias.mailboxes` property uses `m.id is not self.mailbox.id` (identity check) rather than `m.id != self.mailbox.id` (value equality) at line 1583 — behavior difference with detached ORM sessions
- The `canonicalize_email()` function only normalizes Gmail and Proton domains; all other domains pass through unchanged, creating asymmetric matching behavior
- The `normalize_reply_email()` function replaces non-allowed characters with underscores, which could cause a modified reply email to no longer match any stored `Contact.reply_email`

**Outdated references in existing troubleshooting docs:**
- `docs/troubleshooting.md` covers only the forward phase (alias → mailbox) and has no section on reply-phase (mailbox → reverse-alias → contact) debugging

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The deliverable is a single comprehensive Markdown document. The planned structure is:

```
blitzy/
└── documentation/
    └── app_2cd6ee777f8c.md
        ├── Overview and Context
        ├── End-to-End Reply Flow Trace
        │   ├── SMTP Reception (MailHandler.handle_DATA)
        │   ├── Central Routing (handle())
        │   ├── Reply Phase Entry (is_reverse_alias check)
        │   └── Reply Handler (handle_reply())
        ├── Alias-to-User Resolution Chain
        │   ├── Contact Lookup by reply_email
        │   ├── Alias and User Traversal
        │   ├── Mailbox Authorization (get_mailbox_from_mail_from)
        │   └── Email Canonicalization Behavior
        ├── Header Rewriting and Delivery
        │   ├── FROM Header Rewrite to Alias Identity
        │   ├── TO/CC Restoration via replace_header_when_reply
        │   ├── Message-ID Replacement
        │   └── VERP-based SMTP Delivery
        ├── Likely Points of Incorrect Routing
        │   ├── Point 1: Identity vs. Equality in Alias.mailboxes
        │   ├── Point 2: Selective Email Canonicalization
        │   ├── Point 3: disable_email_spoofing_check Fallback
        │   ├── Point 4: normalize_reply_email Character Replacement
        │   ├── Point 5: Multi-Mailbox Notification Flow
        │   └── Summary of Risk Points
        └── Conclusion and Recommendations
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**
- Extract the complete `handle_reply()` execution path from `email_handler.py` (lines 966–1261) using line-by-line code analysis
- Extract the `get_mailbox_from_mail_from()` two-pass check logic from lines 1364–1387
- Extract the `Alias.mailboxes` property from `app/models.py` (lines 1580–1589) and analyze ORM identity behavior
- Extract `canonicalize_email()` from `app/utils.py` (lines 78–94) and map domain-specific behavior
- Extract `normalize_reply_email()` from `app/email_validation.py` (lines 25–38) and document transformation rules
- Cross-reference with `is_reverse_alias()` from `app/email_utils.py` (lines 1156–1163) for the entry classification logic
- Generate Mermaid diagrams by mapping the function call graph and decision points in `handle_reply()`

**Documentation Standards:**
- Markdown formatting with hierarchical headers (`#`, `##`, `###`)
- Mermaid diagram integration for flow visualization
- Code citations using format: `Source: /path/to/file.py:LineNumber`
- Tables for decision point summaries and status code mappings
- Consistent terminology: "reverse-alias" (not "reply address"), "forward phase" / "reply phase" (not "send" / "receive")

### 0.4.3 Diagram and Visual Strategy

The following Mermaid diagrams will be created within the documentation:

- **Reply Phase End-to-End Flowchart:** A `flowchart TD` showing the complete decision tree from `handle()` entry through `handle_reply()` to `sl_sendmail()`, including every conditional branch (domain validation, contact lookup, user status, DMARC, mailbox authorization, SPF, spam check, header rewrite, PGP, delivery)
- **Alias Resolution Sequence Diagram:** A `sequenceDiagram` showing the database lookup chain: `reply_email → Contact.get_by() → contact.alias → alias.user → get_mailbox_from_mail_from() → mailbox`
- **Routing Risk Point Map:** A `flowchart LR` highlighting the five identified risk points inline with the pipeline stages where they occur

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | CREATE | `email_handler.py`, `app/models.py`, `app/contact_utils.py`, `app/email_utils.py`, `app/email_validation.py`, `app/utils.py`, `app/email/status.py`, `app/email/headers.py`, `app/handler/dmarc.py` | Complete investigative analysis document tracing the alias reply-handling flow end-to-end, explaining alias-to-user resolution, and identifying likely points of incorrect routing with code-grounded evidence |

**Transformation Modes Applied:**
- **CREATE** — `blitzy/documentation/app_2cd6ee777f8c.md` is a new file; the `blitzy/documentation/` directory does not yet exist in the repository and must be created

No existing files are updated, deleted, or used as style references per the implementation rule "Do not modify any existing files in the source repository."

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/app_2cd6ee777f8c.md
Type: Technical Investigation / Data-Flow Analysis
Source Code Files:
  - email_handler.py (primary — lines 966–1261 for handle_reply, lines 1364–1387 for mailbox auth, lines 1945–2234 for handle routing)
  - app/models.py (Alias model lines 1469–1660, Contact model lines 1863–2057, Mailbox model lines 2710–2825, AuthorizedAddress model lines 3190–3208)
  - app/contact_utils.py (create_contact function lines 42–120)
  - app/email_utils.py (generate_reply_email lines 1103–1153, is_reverse_alias lines 1156–1163)
  - app/email_validation.py (normalize_reply_email lines 25–38)
  - app/utils.py (canonicalize_email lines 78–94, sanitize_email lines 97–102)
  - app/email/status.py (E200–E525 status code definitions)
  - app/handler/dmarc.py (apply_dmarc_policy_for_reply_phase)
Sections:
  - Overview and Context (purpose, scope, problem statement)
  - End-to-End Reply Flow Trace (SMTP reception → routing → reply handler → delivery)
  - Alias-to-User Resolution Chain (Contact lookup → Alias → User → Mailbox authorization)
  - Header Rewriting and Delivery (FROM/TO/CC rewrite, Message-ID replacement, VERP delivery)
  - Likely Points of Incorrect Routing (5 identified risk points with code evidence)
  - Conclusion and Recommendations (summary of findings)
Diagrams:
  - Reply Phase End-to-End Flowchart (Mermaid flowchart TD)
  - Alias Resolution Sequence Diagram (Mermaid sequenceDiagram)
  - Routing Risk Point Map (Mermaid flowchart LR)
Key Citations:
  - email_handler.py:966-1261 (handle_reply)
  - email_handler.py:1364-1387 (get_mailbox_from_mail_from)
  - email_handler.py:1945-2234 (handle routing)
  - email_handler.py:2288-2378 (MailHandler class)
  - app/models.py:1580-1589 (Alias.mailboxes property)
  - app/models.py:1863-1899 (Contact.reply_email)
  - app/contact_utils.py:89 (reply_email generation)
  - app/email_utils.py:1103-1153 (generate_reply_email)
  - app/email_utils.py:1156-1163 (is_reverse_alias)
  - app/email_validation.py:25-38 (normalize_reply_email)
  - app/utils.py:78-94 (canonicalize_email)
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration updates are required. The project does not use a documentation generator framework. The output is a standalone Markdown file that requires no build system integration.

### 0.5.4 Cross-Documentation Dependencies

- **No shared content or includes:** The new document is self-contained
- **No navigation links required:** No documentation site navigation to update
- **No table of contents updates:** The project has no centralized documentation index
- **No glossary updates needed:** The document will define terms inline where used

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

No external documentation tooling is required for this task. The deliverable is a plain Markdown file with embedded Mermaid diagram blocks that render natively on GitHub. No documentation generator, build tool, or diagram rendering pipeline is needed.

For reference, the following are the key runtime dependencies of the system being documented (verified from `pyproject.toml`):

| Registry | Package Name | Version | Relevance to Documentation |
|----------|--------------|---------|---------------------------|
| PyPI | python | ^3.10 | Runtime version target for all code cited in the analysis |
| PyPI | flask | ^1.1.2 | Web framework — provides `app_context()` used in `_handle()` |
| PyPI | aiosmtpd | ^1.2 | SMTP server — `MailHandler` inherits from aiosmtpd's `Controller` |
| PyPI | sqlalchemy | 1.3.24 | ORM — all model queries and the `Alias.mailboxes` property behavior depend on this specific version |
| PyPI | psycopg2-binary | ^2.9.3 | PostgreSQL driver — database connectivity for `Contact.get_by()` lookups |
| PyPI | email_validator | ^1.1.1 | Email validation — used in forward-phase contact creation |
| PyPI | flanker | ^0.9.11 | Address parsing — used in `replace_header_when_forward()` |
| PyPI | dkimpy | ^1.0.5 | DKIM signing — `add_dkim_signature()` in the reply delivery path |
| PyPI | dnspython | ^2.0.0 | DNS utilities — used for domain validation checks |
| PyPI | newrelic | 8.8.0 | APM instrumentation — `_handle()` is decorated with `@newrelic.agent.background_task()` |
| PyPI | redis | ^4.5.3 | Session and rate-limiting backend |

### 0.6.2 Documentation Reference Updates

Not applicable. No existing documentation links need to be updated, as no existing documentation files are being modified and the new document is self-contained within the `blitzy/documentation/` directory.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Current coverage analysis of the reply-handling flow:**

| Documentation Area | Files Involved | Currently Documented | Target Coverage |
|--------------------|---------------|---------------------|-----------------|
| Reply flow entry (SMTP → routing) | `email_handler.py:2288-2378, 1945-2234` | 0% — no standalone document | 100% — full trace |
| Reply handler logic | `email_handler.py:966-1261` | 0% — only inline docstring | 100% — every branch documented |
| Mailbox authorization | `email_handler.py:1364-1387` | 0% — no documentation | 100% — both passes documented |
| Contact/Alias resolution | `app/models.py`, `app/contact_utils.py` | 0% — no documentation | 100% — full chain traced |
| Email canonicalization | `app/utils.py:78-94` | 0% — no documentation | 100% — domain-specific rules |
| Reply email normalization | `app/email_validation.py:25-38` | 0% — brief docstring only | 100% — character handling |
| Reverse-alias detection | `app/email_utils.py:1156-1163` | 0% — no documentation | 100% — both detection paths |
| Header rewriting in replies | `email_handler.py:1096-1210` | 0% — no documentation | 100% — all header transformations |
| Status codes for reply phase | `app/email/status.py` | 0% — inline comments only | 100% — all reply-relevant codes |

**Target coverage:** 100% of all decision points, database lookups, and header transformations in the reply-handling pipeline, as this is the core scope of the investigative document.

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every function in the reply-phase call chain has a documented purpose, inputs, outputs, and decision branches
- Every conditional branch in `handle_reply()` is documented with the corresponding status code outcome
- All five identified routing risk points include code-line citations, explanation of the mechanism, and conditions under which misrouting could occur
- The alias-to-user resolution chain is traced with concrete field references (model name, column name, line number)

**Accuracy validation:**
- All code citations reference specific line numbers verified against the current repository state (branch `app_2cd6ee777f8c`)
- All function signatures and parameter names match the actual source code
- All status codes match the definitions in `app/email/status.py`
- Mermaid diagrams accurately reflect the branching logic in the source code

**Clarity standards:**
- Technical accuracy with accessible explanations — each routing risk point explains both the code mechanism and the user-visible symptom
- Progressive disclosure — overview first, then detailed trace, then root-cause analysis
- Consistent terminology aligned with the codebase: "reverse-alias," "forward phase," "reply phase," "mailbox," "contact," "alias"

**Maintainability:**
- Source citations in format `Source: file_path:LineNumber` for traceability
- Section structure mirrors the code execution order for easy cross-referencing
- Self-contained document with no external dependencies

### 0.7.3 Example and Diagram Requirements

- **Minimum diagrams:** 3 Mermaid diagrams (reply flow, resolution sequence, risk point map)
- **Code example testing:** Not applicable — this is an analysis document, not a tutorial. Code snippets are citations from existing source, not new executable examples
- **Diagram accuracy:** Each diagram must be verified against the actual branching logic in the source code

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `blitzy/documentation/app_2cd6ee777f8c.md` — The sole deliverable: a comprehensive investigative analysis document

**Source files analyzed for documentation content (read-only):**
- `email_handler.py` — Central SMTP handler with `handle()`, `handle_reply()`, `handle_forward()`, `get_mailbox_from_mail_from()`, `handle_unknown_mailbox()`, `replace_header_when_reply()`, `forward_email_to_mailbox()`, `replace_original_message_id()`, `notify_mailbox()`
- `app/models.py` — `Alias`, `Contact`, `Mailbox`, `User`, `EmailLog`, `AuthorizedAddress`, `SLDomain`, `MessageIDMatching` model definitions and properties
- `app/contact_utils.py` — `create_contact()` function and `ContactCreateResult` dataclass
- `app/email_utils.py` — `generate_reply_email()`, `is_reverse_alias()`, `generate_verp_email()`, `get_verp_info_from_email()`, `parse_full_address()`, `sl_formataddr()`
- `app/email_validation.py` — `normalize_reply_email()`, `is_valid_email()`
- `app/utils.py` — `canonicalize_email()`, `sanitize_email()`
- `app/email/status.py` — SMTP status code constants
- `app/email/headers.py` — Header name constants
- `app/email/rate_limit.py` — Rate limiting logic (currently disabled)
- `app/handler/dmarc.py` — `apply_dmarc_policy_for_reply_phase()`
- `app/handler/spamd_result.py` — `SpamdResult` and `SPFCheckResult`
- `app/config.py` — Configuration constants referenced in the reply flow

**Documentation scope topics:**
- End-to-end reply-phase data flow from SMTP reception to outbound delivery
- Alias-to-user resolution chain through ORM relationships
- Mailbox authorization and email canonicalization behavior
- Header rewriting mechanics in the reply phase
- Identification and explanation of potential misrouting points

### 0.8.2 Explicitly Out of Scope

- **Source code modifications:** No existing files in the repository will be modified, per the implementation rule
- **Forward phase analysis:** The forward phase (`handle_forward()`, `forward_email_to_mailbox()`) is referenced for context but is not the focus of this investigation
- **Bounce and complaint handling:** `handle_bounce_forward_phase()`, `handle_bounce_reply_phase()`, `handle_hotmail_complaint()`, `handle_yahoo_complaint()` are out of scope
- **Unsubscribe processing:** `UnsubscribeHandler` and related encode/decode/generator modules are not relevant to the reply routing question
- **PGP encryption internals:** PGP encryption is mentioned as a pipeline step but its internal implementation in `app/pgp_utils.py` is not analyzed
- **Test file modifications:** No test files are created or modified
- **Deployment configuration changes:** No Docker, Postfix, or infrastructure changes
- **Documentation for other features:** Authentication, billing, OAuth, custom domain verification, and all non-email-handler subsystems
- **REST API documentation:** The `app/api/` endpoints are not relevant to SMTP reply handling
- **Frontend/dashboard documentation:** The `app/dashboard/` views are not relevant
- **Items excluded by user:** No additional explicit exclusions from user instructions

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** Not applicable — output is a standalone Markdown file
- **Documentation preview command:** Any Markdown renderer (e.g., GitHub web view, VS Code Markdown Preview)
- **Diagram generation command:** Mermaid diagrams are embedded inline in the Markdown file and render natively on GitHub and compatible viewers; no separate generation step is needed
- **Documentation deployment command:** Not applicable — the file is committed directly to the `blitzy/documentation/` directory
- **Default format:** GitHub-Flavored Markdown with Mermaid diagram blocks
- **Citation requirement:** Every technical claim must reference specific source files and line numbers in the format `Source: path/to/file.py:LineNumber`
- **Style guide:** No repository-specific style guide exists; the document follows standard technical writing conventions with hierarchical headers, tables for structured data, and Mermaid for visual flow representation
- **Documentation validation:** Manual review for accuracy of code citations, correctness of Mermaid diagram syntax, and completeness of flow coverage

### 0.9.2 Output File Specification

- **Target path:** `blitzy/documentation/app_2cd6ee777f8c.md`
- **File naming convention:** Derived from the source branch name `app_2cd6ee777f8c` per the `SWE-AtlasQnA-Repo` implementation rule
- **Directory creation:** The `blitzy/documentation/` directory must be created as it does not currently exist in the repository
- **Encoding:** UTF-8
- **No modifications to existing files:** Enforced by the implementation rule — the deliverable is exclusively the new file

## 0.10 Rules for Documentation

The following rules are derived from the user's explicit instructions and the `SWE-AtlasQnA-Repo` implementation rule:

- **Do not modify any existing files in the source repository.** The deliverable is exclusively the new file `blitzy/documentation/app_2cd6ee777f8c.md`. No source code, test files, configuration files, or existing documentation files may be changed.
- **Do not make assumptions — base answers on the code as the truth.** All technical conclusions, data flow descriptions, and routing risk assessments must be directly grounded in inspected source code with specific file and line number citations. No external assumptions about how the system "should" work are permitted.
- **Provide thinking and rationale behind the answers.** The document must not merely describe what the code does, but explain why each decision point could or could not lead to incorrect routing, with reasoning visible to the reader.
- **Create the document named `app_2cd6ee777f8c.md`.** The filename must exactly match the source branch name per the implementation rule convention.
- **Place the document in `blitzy/documentation/`.** This directory must be created if it does not exist.
- **Clean up temporary artifacts.** Any temporary scripts, logs, or instrumentation created during the investigation must be removed before completion, leaving the codebase exactly as found.
- **Comprehensive answers required.** The document must comprehensively answer all questions posed: (1) which part of the system handles incoming reply messages, (2) how the alias is resolved to a user, (3) what user ID the system forwards the reply to, and (4) the most likely point where incorrect routing could originate.
- **Include Mermaid diagrams for flow visualization.** The reply-phase data flow is complex enough to warrant visual representation through embedded Mermaid diagrams.
- **Use source code citations for all technical details.** Every referenced function, model, field, or decision point must include a citation in the format `Source: path/to/file.py:LineNumber`.

## 0.11 References

### 0.11.1 Codebase Files and Folders Searched

The following files and folders were systematically searched and inspected across the codebase to derive all conclusions in this Agent Action Plan:

**Root-level files inspected:**
- `email_handler.py` — Central SMTP handler (2404 lines); inspected lines 1–176 (imports/docstring), 180–318 (contact creation/header rewrite helpers), 345–534 (reply header replacement, PGP preparation, cycle handling), 536–928 (forward phase), 966–1261 (reply phase), 1296–1387 (message ID replacement, mailbox authorization), 1390–1430 (unknown mailbox handling), 1945–2234 (central routing function), 2236–2287 (out-of-office handlers), 2288–2404 (MailHandler class and main)
- `pyproject.toml` — Dependency manifest (135 lines); full file inspected for Python version (^3.10), all runtime and dev dependencies with exact version constraints
- `README.md` — Self-hosting guide (lines 1–80 inspected for documentation structure)

**`app/` package files inspected:**
- `app/models.py` — ORM models; inspected `Alias` (lines 1469–1660), `Contact` (lines 1863–2057), `EmailLog` (lines 2060+), `Mailbox` (lines 2710–2825), `AuthorizedAddress` (lines 3190–3208)
- `app/contact_utils.py` — Contact creation logic (121 lines); full file inspected
- `app/email_utils.py` — Email utilities; inspected `generate_reply_email()` (lines 1103–1153), `is_reverse_alias()` (lines 1156–1163)
- `app/email_validation.py` — Email validation; inspected `normalize_reply_email()` (lines 25–38)
- `app/utils.py` — General utilities; inspected `canonicalize_email()` (lines 78–94), `sanitize_email()` (lines 97–102)
- `app/email/status.py` — Full file inspected (65 lines of status codes)
- `app/email/__init__.py`, `app/email/rate_limit.py`, `app/email/spam.py`, `app/email/headers.py` — Folder contents reviewed via `get_source_folder_contents`

**`app/handler/` package inspected:**
- `app/handler/__init__.py`, `app/handler/dmarc.py`, `app/handler/provider_complaint.py`, `app/handler/spamd_result.py`, `app/handler/unsubscribe_encoder.py`, `app/handler/unsubscribe_generator.py`, `app/handler/unsubscribe_handler.py` — Folder contents reviewed via `get_source_folder_contents`

**`docs/` folder inspected:**
- `docs/troubleshooting.md` — Full file inspected (66 lines)
- `docs/code-structure.md` — Full file inspected (10 lines)
- All other `docs/*.md` files — Summaries reviewed via `get_source_folder_contents`

**`tests/` folder inspected:**
- `tests/test_email_handler.py` — Reply-related test function names identified via grep
- `tests/` folder structure reviewed via `get_source_folder_contents`

**Folders reviewed via `get_source_folder_contents`:**
- Root folder (`""`) — Full children list with summaries
- `app/` — Full children list with summary
- `app/email/` — Full children list with summary
- `app/handler/` — Full children list with summary
- `docs/` — Full children list with summary
- `tests/` — Full children list with summary

**Tech spec sections retrieved:**
- Section 4.2 "Core Email Processing Workflows" — Provided existing flow documentation for cross-reference
- Section 1.3 "Scope" — Provided system scope definition and in-scope/out-of-scope boundaries

### 0.11.2 Attachments and External Resources

- **User attachments:** None — the user provided 0 attachments
- **Figma designs:** None provided
- **External URLs:** None referenced by the user
- **Environment files:** No environment setup files provided (`/tmp/environments_files/` is empty)

