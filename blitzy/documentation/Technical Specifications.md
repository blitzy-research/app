# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create a new investigative analysis document** that comprehensively explains how SimpleLogin's inbound reply-phase email handler resolves a reply email address to a Contact record, derives the associated Alias and User, and forwards the reply to its destination — and whether this resolution logic exhibits race conditions, uniqueness assumption violations, or timing-related inconsistencies that could cause a reply to be routed to the wrong user.

- **Documentation Category:** Create new documentation
- **Documentation Type:** Technical investigation / Q&A analysis document
- **Output Artifact:** A Markdown file named `app_2cd6ee777f8c.md` placed in the `blitzy/documentation/` directory of the destination repository

The requirements translate to the following concrete documentation deliverables:

- **Reply Resolution Anatomy:** A detailed walkthrough of the code path from SMTP envelope receipt through `handle_reply()` to final delivery, with exact file and line references
- **Runtime Value Trace:** An explanation of the runtime values at each step — the raw `rcpt_to`, the normalized `reply_email`, the Contact record (or absence thereof), the derived Alias and User, and the selected Mailbox
- **Cross-Event Consistency Analysis:** An assessment of whether the same `reply_email` can resolve to different Contacts over time, fail to resolve temporarily, or resolve correctly but forward to a different user than the alias owner
- **Race Condition and Uniqueness Assessment:** A code-level analysis of the `Contact.reply_email` column's index/uniqueness constraints, the TOCTOU window in `generate_reply_email`, the `.first()` behavior of `Contact.get_by()`, and the absence of distributed locking in the SMTP processing path
- **Conclusions and Root Cause Hypothesis:** A synthesized explanation of how the observed code structure could or could not lead to wrong-user routing, grounded in specific code evidence

### 0.1.2 Special Instructions and Constraints

- **CRITICAL — No Source Code Modifications:** The user explicitly requires that no repository source files be modified. The output is a read-only analysis document. If temporary scripts or tooling are needed to observe behavior, they must be cleaned up, leaving the repository unchanged.
- **Implementation Rule — SWE-AtlasQnA-Repo:** The user's implementation rules specify:
  - Create a new markdown document named `<source_branch_name>.md` — i.e., `app_2cd6ee777f8c.md`
  - Provide thinking and rationale behind the answers
  - Do not make assumptions; base answers on the code as the truth
  - Do not modify any existing files in the source repository
  - Place the generated document in the `blitzy/documentation` directory in the destination repo
- **Evidence-Based Analysis:** All claims must cite specific source files and line numbers
- **No assumptions about runtime behavior:** The document must distinguish between what the code guarantees versus what it probabilistically avoids

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **document the reply resolution flow**, we will trace the code path from `MailHandler.handle_DATA()` → `_handle()` → `handle()` → `handle_reply()` in `email_handler.py`, documenting each decision point, lookup, and value transformation
- To **document the Contact lookup mechanism**, we will analyze `Contact.get_by(reply_email=...)` in `app/models.py` (the `ModelMixin.get_by` method using `.first()`), the `contact` table's index and constraint definitions, and the migration history for the `reply_email` column
- To **document the reply email generation and uniqueness**, we will trace `generate_reply_email()` in `app/email_utils.py`, `available_sl_email()` in `app/models.py`, and `normalize_reply_email()` in `app/email_validation.py`
- To **assess race conditions**, we will analyze the concurrency model of `aiosmtpd.Controller`, the absence of distributed locking in the SMTP handler path (compared to the `parallel_limiter.py` used in the web path), and the IntegrityError handling in `app/contact_utils.py`

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following implicit documentation needs have been identified:

- **Database constraint gap documentation:** The `contact.reply_email` column has an index (`ix_contact_reply_email`, `unique=False`) but no UNIQUE constraint — this is a critical finding that must be documented with its implications for the `.first()` query behavior
- **Concurrency model documentation:** The SMTP handler uses `aiosmtpd.Controller` which runs an asyncio event loop in a separate thread; multiple simultaneous connections can invoke `handle_reply()` concurrently without distributed locking
- **Normalization collision documentation:** The `normalize_reply_email()` function replaces non-allowed characters with underscores, which could theoretically map distinct reply emails to the same normalized form
- **IntegrityError recovery documentation:** The `create_contact()` function in `app/contact_utils.py` catches the `(alias_id, website_email)` unique constraint violation but has no mechanism to catch a hypothetical `reply_email` collision


## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a documentation structure consisting of operational runbooks, deployment guides, and API reference material housed in a flat `docs/` directory alongside top-level community documents (`README.md`, `CONTRIBUTING.md`, `SECURITY.md`). There is no structured documentation framework (no `mkdocs.yml`, `docusaurus.config.js`, or `sphinx/conf.py`), no documentation generator configuration, and no existing documentation that specifically covers the reply-resolution logic or the contact lookup mechanism.

- **Documentation framework:** None (flat Markdown files only)
- **Documentation generator:** Not detected
- **API documentation tools:** None in the codebase (no JSDoc, Sphinx, or Godoc configuration)
- **Diagram tools:** None preconfigured (Mermaid will be used inline in the output document)
- **Existing documentation for reply handling:** Not found — no existing document covers the reply-phase flow, Contact.reply_email resolution, or race condition analysis
- **`blitzy/documentation/` directory:** Does not exist yet; must be created

Existing documentation files found:

| File | Content Summary |
|------|----------------|
| `docs/api.md` | REST API reference covering aliases, mailboxes, custom domains, contacts, notifications |
| `docs/code-structure.md` | Brief note on local_data/ directory and JWT keys |
| `docs/troubleshooting.md` | Diagnostic steps for welcome-email and alias forwarding issues |
| `docs/ssl.md` | TLS/HTTPS certificate and MTA-STS configuration guide |
| `docs/enforce-spf.md` | SPF enforcement with Postfix PCRE rules |
| `docs/ses.md` | Amazon SES relay configuration |
| `docs/gmail-relay.md` | Gmail SMTP relay setup |
| `docs/postfix-tls.md` | Postfix submission TLS configuration |
| `docs/build-image.md` | Docker multi-arch image build instructions |
| `docs/upgrade.md` | Version upgrade runbook |
| `docs/ufw.md` | UFW firewall port rules |
| `docs/oauth.md` | OAuth2/OIDC flows and test URLs |
| `README.md` | Project overview and self-hosting guide |
| `CONTRIBUTING.md` | Contributor workflow |
| `SECURITY.md` | Vulnerability disclosure policy |

### 0.2.2 Repository Code Analysis for Documentation

The following source files were systematically examined to build the analysis foundation:

**Primary reply-handling code path:**
- `email_handler.py` — Central SMTP inbound processor; contains `handle()`, `handle_reply()`, `handle_forward()`, `get_or_create_contact()`, `replace_header_when_reply()`, `get_mailbox_from_mail_from()`, `handle_unknown_mailbox()`, and `MailHandler` class
- `app/email_utils.py` — `generate_reply_email()`, `is_reverse_alias()`, `normalize_reply_email()` (delegated to `app/email_validation.py`), VERP generation and validation
- `app/email_validation.py` — `normalize_reply_email()` and `is_valid_email()`
- `app/contact_utils.py` — `create_contact()` workflow with IntegrityError handling
- `app/models.py` — `Contact` model (lines 1863–2058), `ModelMixin.get_by()` (line 83–84), `available_sl_email()` (lines 1425–1432), `Alias` model, `EmailLog` model, `SLDomain` model

**Supporting infrastructure code:**
- `app/utils.py` — `random_string()`, `convert_to_id()`, `convert_to_alphanumeric()`, `canonicalize_email()`
- `app/parallel_limiter.py` — Redis distributed locks (used only in Flask web path, NOT in SMTP handler)
- `app/email/status.py` — SMTP status codes returned by handlers

**Database schema evidence:**
- `migrations/versions/2021_071310_78403c7b8089_.py` — Creates `ix_contact_reply_email` index with `unique=False`
- `migrations/versions/2020_031711_0809266d08ca_.py` — Creates `uq_contact` unique constraint on `(alias_id, website_email)`

**Test coverage for reply handling:**
- `tests/test_email_handler.py` — Tests for `test_replace_contacts_and_user_in_reply_phase`, `test_send_email_from_non_canonical_address_on_reply`, `test_dmarc_reply_quarantine`
- `tests/test_contact_utils.py` — Tests for contact creation, update, IntegrityError handling

**Configuration and deployment:**
- `pyproject.toml` — Python 3.10+, SQLAlchemy 1.3.24 (pinned), aiosmtpd ^1.2
- `Dockerfile` — Python 3.10 base, Gunicorn for web (2 workers), email handler is a separate process

### 0.2.3 Web Search Research Conducted

No external web search was required for this analysis. The investigation is entirely code-based, as specified by the user's directive to "base answers on the code as the truth." The codebase contains all information necessary to trace the reply-resolution logic, identify the database constraints, and assess concurrency risks.


## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The following modules and functions are directly relevant to the reply-resolution analysis and form the backbone of the documentation:

- **Module: `email_handler.py` (root)**
  - Public functions: `handle()`, `handle_reply()`, `handle_forward()`, `get_or_create_contact()`, `get_or_create_reply_to_contact()`, `replace_header_when_reply()`, `get_mailbox_from_mail_from()`, `handle_unknown_mailbox()`, `replace_original_message_id()`
  - Current documentation: File-level docstring (lines 1–31) describes forward/reply phases conceptually. No inline documentation of the Contact lookup by `reply_email` or its uniqueness assumptions.
  - Documentation needed: Step-by-step trace of the reply resolution path, runtime value documentation at each decision point, assessment of the `.first()` query behavior when `reply_email` is not uniquely constrained

- **Module: `app/email_utils.py`**
  - Key functions: `generate_reply_email()` (lines 1103–1153), `is_reverse_alias()` (lines 1156–1163)
  - Current documentation: Brief docstring on `generate_reply_email` — "generate a reply_email (aka reverse-alias), make sure it isn't used by any contact"
  - Documentation needed: Analysis of the TOCTOU gap between `available_sl_email()` check and `Contact.create()`, entropy assessment of `random_string(20-50)`, analysis of the `include_sender_in_reverse_alias` path

- **Module: `app/email_validation.py`**
  - Key function: `normalize_reply_email()` (lines 25–38)
  - Current documentation: Docstring — "Handle the case where reply email contains *strange* char that was wrongly generated in the past"
  - Documentation needed: Analysis of the character-replacement rules and whether distinct reply emails can normalize to the same value, creating a lookup collision

- **Module: `app/contact_utils.py`**
  - Key function: `create_contact()` (lines 42–120)
  - Current documentation: No docstring beyond dataclass definitions
  - Documentation needed: Documentation of the IntegrityError recovery path when `(alias_id, website_email)` collides, and the fact that no analogous recovery exists for `reply_email` collisions

- **Module: `app/models.py`**
  - Key entities: `Contact` (lines 1863–2058), `ModelMixin.get_by()` (line 83–84), `available_sl_email()` (lines 1425–1432)
  - Current documentation: Docstring on Contact — "Store configuration of sender (website-email) and alias."
  - Documentation needed: Analysis of the `reply_email` column definition (`index=True` but no unique constraint), the `.first()` behavior of `get_by()`, and the `Contact.create()` override that checks for reverse-alias self-reference

- **Module: `app/parallel_limiter.py`**
  - Key class: `_InnerLock` — Redis-based distributed lock
  - Current documentation: Minimal
  - Documentation needed: Explanation of why this locking mechanism applies only to Flask web endpoints (uses `current_user` and `request.remote_addr`) and is NOT available in the SMTP handler path

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **Undocumented reply resolution internals:** No existing documentation covers the `handle_reply()` → `Contact.get_by(reply_email=...)` → alias → user → mailbox resolution chain
- **Undocumented database constraint design decision:** The choice to make `contact.reply_email` indexed but NOT unique is undocumented and has direct implications for the correctness of `.first()` lookups
- **Undocumented concurrency model:** No documentation describes how the `aiosmtpd.Controller` processes concurrent SMTP connections and the absence of distributed locking in the email handler path
- **Undocumented reply_email generation assumptions:** The assumption that `random_string(20-50)` provides sufficient entropy to avoid collision, combined with the application-level `available_sl_email()` check, is undocumented
- **Undocumented normalization behavior:** The `normalize_reply_email()` function's character replacement rules and their potential to create lookup collisions are undocumented
- **Missing architecture docs for reply phase:** While the tech spec contains flowcharts for the forward and reply phases (Section 4.2.3), no document provides a code-level trace with runtime values


## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output is a single Markdown file placed at `blitzy/documentation/app_2cd6ee777f8c.md`. The document structure follows a logical progression from mechanism description through runtime trace to root cause assessment:

```
blitzy/
└── documentation/
    └── app_2cd6ee777f8c.md
        ├── Overview / Question Statement
        ├── Reply Resolution Mechanism
        │   ├── Entry Point: MailHandler and Routing
        │   ├── Reply Email Derivation
        │   ├── Contact Lookup by reply_email
        │   ├── Alias, User, and Mailbox Derivation
        │   └── Delivery to Contact
        ├── Reply Email Generation and Uniqueness
        │   ├── generate_reply_email() Internals
        │   ├── available_sl_email() Check
        │   ├── Entropy and Collision Probability
        │   └── normalize_reply_email() Behavior
        ├── Database Constraints and .first() Behavior
        │   ├── Contact Table Schema and Indexes
        │   ├── reply_email Index (Non-Unique)
        │   ├── ModelMixin.get_by() and .first() Semantics
        │   └── Implications for Multi-Row Matches
        ├── Concurrency and Timing Analysis
        │   ├── aiosmtpd Concurrency Model
        │   ├── TOCTOU Window in Reply Email Generation
        │   ├── IntegrityError Recovery in create_contact()
        │   ├── Absence of Distributed Locking in SMTP Path
        │   └── Multiple Forward Events Creating Contacts Simultaneously
        ├── Cross-Event Consistency Assessment
        │   ├── Can the Same reply_email Resolve to Different Contacts?
        │   ├── Can reply_email Resolution Fail Temporarily?
        │   └── Can a Correct Resolution Forward to the Wrong User?
        ├── Root Cause Assessment
        │   └── How Code Structure Leads to Observed Behavior
        └── Conclusions
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**
- Extract the reply-phase code path from `email_handler.py` (lines 966–1261), documenting each function call, database query, and conditional branch with exact line references
- Extract the `Contact` model schema from `app/models.py` (lines 1863–1899), including the `reply_email` column definition and the `__table_args__` unique constraint
- Extract the `generate_reply_email()` logic from `app/email_utils.py` (lines 1103–1153), documenting the random string generation, the `available_sl_email()` check, and the loop structure
- Extract the `normalize_reply_email()` logic from `app/email_validation.py` (lines 25–38), documenting the character replacement rules
- Extract the `create_contact()` workflow from `app/contact_utils.py` (lines 42–120), documenting the IntegrityError handling
- Extract migration evidence from `migrations/versions/2021_071310_78403c7b8089_.py` confirming `unique=False` on the `reply_email` index

**Documentation Standards:**
- Markdown formatting with proper headers (`#`, `##`, `###`)
- Mermaid diagrams for the reply resolution flow and the TOCTOU timing window
- Code snippets using fenced code blocks with `python` syntax highlighting — kept brief (2–3 lines per snippet)
- Source citations in the format `Source: /path/to/file.py:LineNumber`
- Tables for parameter descriptions, constraint analysis, and runtime value traces

### 0.4.3 Diagram and Visual Strategy

Mermaid diagrams to create within the output document:

- **Reply Resolution Sequence Diagram:** Shows the step-by-step flow from SMTP receipt through `handle_reply()` to delivery, including the Contact lookup, Alias derivation, Mailbox authorization, and final SMTP send — with runtime values annotated at each step
- **Reply Email Generation Flow:** Shows the `generate_reply_email()` → `available_sl_email()` → `Contact.create()` path with the TOCTOU gap highlighted
- **Concurrency Timing Diagram:** Shows two concurrent SMTP connections processing reply-phase emails, illustrating how the absence of distributed locking could theoretically allow a race condition


## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | CREATE | `email_handler.py`, `app/email_utils.py`, `app/email_validation.py`, `app/contact_utils.py`, `app/models.py`, `app/parallel_limiter.py`, `app/utils.py`, `app/email/status.py`, `migrations/versions/2021_071310_78403c7b8089_.py`, `migrations/versions/2020_031711_0809266d08ca_.py` | Complete investigative analysis of reply-resolution logic, runtime values, race conditions, and uniqueness assumptions — answering all questions posed in the user prompt |

No existing files are updated or deleted. No documentation configuration files need modification (no documentation framework exists in the repository).

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/app_2cd6ee777f8c.md
Type: Technical Investigation / Q&A Analysis
Source Code:
  - email_handler.py (primary: lines 966-1261, 1364-1430, 1945-2234, 2288-2405)
  - app/email_utils.py (lines 1103-1163)
  - app/email_validation.py (lines 25-38)
  - app/contact_utils.py (lines 42-120)
  - app/models.py (lines 79-84, 1425-1432, 1863-1962)
  - app/parallel_limiter.py (lines 1-73)
  - app/utils.py (lines 41-47)
  - migrations/versions/2021_071310_78403c7b8089_.py
  - migrations/versions/2020_031711_0809266d08ca_.py
Sections:
  - Overview and Question Statement
  - Reply Resolution Mechanism (entry point, derivation, lookup, delivery)
  - Reply Email Generation and Uniqueness (entropy, collision, normalization)
  - Database Constraints and .first() Behavior (schema, index, implications)
  - Concurrency and Timing Analysis (aiosmtpd model, TOCTOU, IntegrityError, locking)
  - Cross-Event Consistency Assessment (same reply_email → different contacts?, temporary failures?, wrong-user forwarding?)
  - Root Cause Assessment
  - Conclusions
Diagrams:
  - Reply resolution sequence diagram (Mermaid)
  - Reply email generation flow with TOCTOU gap (Mermaid)
  - Concurrency timing diagram (Mermaid)
Key Citations:
  - email_handler.py:966-1261 (handle_reply)
  - email_handler.py:1945-2234 (handle routing)
  - app/models.py:1863-1899 (Contact model)
  - app/models.py:83-84 (get_by using .first())
  - app/email_utils.py:1103-1153 (generate_reply_email)
  - app/email_validation.py:25-38 (normalize_reply_email)
  - app/contact_utils.py:42-120 (create_contact with IntegrityError)
  - migrations/versions/2021_071310_78403c7b8089_.py (reply_email index, unique=False)
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration updates are required. The repository does not use a documentation framework. The output file is a standalone Markdown document placed in a new `blitzy/documentation/` directory.

### 0.5.4 Cross-Documentation Dependencies

- **No shared content or includes:** The output document is self-contained
- **No navigation links:** No documentation index or table of contents to update
- **No glossary updates:** No existing glossary in the repository
- **Reference to existing docs:** The document may reference `docs/api.md` for context on the Contact API, but does not modify it


## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

No documentation tools or packages are required for this task. The output is a plain Markdown file with inline Mermaid diagram syntax. No documentation site generator, diagram renderer, or API doc extractor is needed.

The following project dependencies are relevant to the **analysis** (not to the documentation toolchain) and are cited as evidence in the output document:

| Registry | Package Name | Version | Relevance to Analysis |
|----------|--------------|---------|----------------------|
| pip | SQLAlchemy | 1.3.24 (pinned) | Defines the ORM query behavior of `Session.query(cls).filter_by(**kw).first()` used by `Contact.get_by()` |
| pip | aiosmtpd | ^1.2 | Defines the SMTP server concurrency model via `Controller` (separate thread with asyncio event loop) |
| pip | psycopg2-binary | ^2.9.3 | PostgreSQL driver; determines how `.first()` interacts with non-unique indexed columns |
| pip | Flask-Migrate | ^2.5.3 | Alembic wrapper managing the migration that created the `ix_contact_reply_email` index with `unique=False` |
| pip | email_validator | ^1.1.1 | Used in `is_valid_email()` and `normalize_reply_email()` for input validation |

### 0.6.2 Documentation Reference Updates

No documentation link updates are required. The new file at `blitzy/documentation/app_2cd6ee777f8c.md` is a standalone artifact that does not require integration into any existing documentation navigation or cross-reference system.


## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

The user's prompt poses a multi-part investigative question. Coverage is measured by whether each sub-question is fully answered with code-grounded evidence.

| Question from User Prompt | Target Coverage | Source Files |
|--------------------------|----------------|--------------|
| How does the system derive the reply email address from an inbound reply message? | 100% — trace from `rcpt_to` through `normalize_reply_email()` to the lookup value | `email_handler.py:972-984`, `app/email_validation.py:25-38` |
| How does it use the reply email to identify the associated contact? | 100% — document `Contact.get_by(reply_email=...)` and `ModelMixin.get_by()` | `email_handler.py:986`, `app/models.py:83-84` |
| What are the runtime values involved in resolution? | 100% — enumerate `rcpt_to`, `reply_email` (normalized), `contact`, `alias`, `user`, `mailbox` at each step | `email_handler.py:966-1050` |
| Does the same reply email ever resolve to different contacts over time? | 100% — analyze `reply_email` uniqueness (index not unique, `.first()` non-deterministic) | `app/models.py:1899`, `migrations/versions/2021_071310_78403c7b8089_.py` |
| Can resolution fail temporarily? | 100% — analyze transaction isolation, Session lifecycle, commit timing | `app/contact_utils.py:90-119`, `app/db.py` |
| Can a correct resolution forward to a different user than expected? | 100% — analyze Contact → Alias → User chain and what breaks it | `email_handler.py:986-1004` |
| Does the contact-by-reply-email lookup exhibit race conditions? | 100% — analyze TOCTOU in `generate_reply_email`, concurrency model, missing locks | `app/email_utils.py:1103-1153`, `app/parallel_limiter.py` |
| Does it exhibit uniqueness assumption violations? | 100% — contrast the application-level uniqueness check with the missing DB-level unique constraint | `app/models.py:1899`, `app/models.py:1425-1432` |
| Does it exhibit timing-related inconsistencies? | 100% — analyze whether concurrent forward-phase contact creation can produce duplicate `reply_email` values | `app/contact_utils.py:89-119` |

**Target documentation coverage: 100% of user-posed questions, grounded in code evidence.**

### 0.7.2 Documentation Quality Criteria

- **Completeness:** Every sub-question in the user prompt is answered with a dedicated section
- **Accuracy:** All code citations include exact file paths and line numbers; all claims are verifiable by reading the cited source
- **Clarity:** Technical analysis is presented progressively — mechanism first, then edge cases, then assessment
- **Traceability:** Every conclusion cites the specific code that supports it using `Source: file:line` notation
- **No assumptions:** The document explicitly separates what the code guarantees from what it probabilistically avoids, per the user's instruction
- **Diagrams:** Mermaid sequence/flow diagrams illustrate the reply resolution path and the TOCTOU timing window

### 0.7.3 Example and Diagram Requirements

- **Minimum diagrams:** 3 (reply resolution flow, generation/uniqueness flow, concurrency timing)
- **Code example style:** Brief 2–3 line extracts showing the exact function signatures and query patterns
- **Diagram format:** Mermaid (inline in Markdown, no external rendering required)
- **Visual content freshness:** All diagrams reflect the current codebase state as analyzed


## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

- **New documentation file:**
  - `blitzy/documentation/app_2cd6ee777f8c.md` — the sole output artifact

- **Code analysis scope (read-only, no modifications):**
  - `email_handler.py` — reply-phase handler, forward-phase handler, main routing logic, MailHandler class
  - `app/email_utils.py` — `generate_reply_email()`, `is_reverse_alias()`, VERP handling
  - `app/email_validation.py` — `normalize_reply_email()`, `is_valid_email()`
  - `app/contact_utils.py` — `create_contact()`, IntegrityError recovery
  - `app/models.py` — `Contact` model, `ModelMixin.get_by()`, `available_sl_email()`, `Alias` model, `EmailLog` model, `SLDomain` model
  - `app/parallel_limiter.py` — Redis distributed lock (to document its absence in SMTP path)
  - `app/utils.py` — `random_string()`, `sanitize_email()`, `canonicalize_email()`
  - `app/db.py` — Session management and scoped session configuration
  - `app/email/status.py` — SMTP status codes
  - `migrations/versions/2021_071310_78403c7b8089_.py` — `ix_contact_reply_email` index creation (unique=False)
  - `migrations/versions/2020_031711_0809266d08ca_.py` — `uq_contact` unique constraint on (alias_id, website_email)
  - `tests/test_email_handler.py` — Reply-phase test coverage
  - `tests/test_contact_utils.py` — Contact creation test coverage
  - `pyproject.toml` — Dependency versions (SQLAlchemy 1.3.24, aiosmtpd ^1.2)
  - `Dockerfile` — Deployment model (separate processes for web and email handler)

### 0.8.2 Explicitly Out of Scope

- **Source code modifications** — explicitly excluded by user instruction ("Don't modify any repository source files")
- **Test file modifications** — no test changes
- **Feature additions or code refactoring** — no code changes of any kind
- **Forward-phase analysis** — covered only insofar as it creates Contact records with `reply_email` values; the forwarding flow itself is not the subject of the investigation
- **Bounce handling analysis** — not part of the user's question
- **PGP encryption analysis** — not part of the user's question
- **DMARC/SPF enforcement analysis** — not part of the user's question (mentioned only as context for the reply-phase flow)
- **Web dashboard or API endpoint analysis** — not part of the user's question
- **Unsubscribe handling** — not part of the user's question
- **Provider complaint handling** — not part of the user's question
- **Deployment configuration changes** — no infrastructure modifications
- **Documentation for any module outside the reply-resolution code path**


## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** Not applicable — the output is a standalone Markdown file
- **Documentation preview command:** Any Markdown renderer (e.g., `grip`, GitHub preview, VS Code Markdown preview)
- **Diagram generation command:** Not applicable — Mermaid diagrams are embedded inline and renderable by any Mermaid-compatible viewer (GitHub, GitLab, VS Code with Mermaid extension)
- **Documentation deployment command:** Not applicable — the file is placed directly in the repository
- **Default format:** Markdown with inline Mermaid diagrams
- **Citation requirement:** Every technical claim must reference the source file and line number
- **Style guide:** Follows the user's implementation rule: "Provide thinking / rationale behind the answers. Do not make assumptions, base your answers on the code as the truth."
- **Documentation validation:** The document should be parseable as valid Markdown; all Mermaid diagram blocks should be syntactically correct

### 0.9.2 Output File Specification

| Attribute | Value |
|-----------|-------|
| File path | `blitzy/documentation/app_2cd6ee777f8c.md` |
| File name derivation | Branch name `app_2cd6ee777f8c` + `.md` extension |
| Format | GitHub-Flavored Markdown |
| Diagram syntax | Mermaid (```mermaid code blocks) |
| Code syntax | Python (```python code blocks) |
| Max heading depth | `####` (4 levels) |
| Source citation format | `Source: path/to/file.py:LineNumber` or `Source: path/to/file.py:StartLine-EndLine` |


## 0.10 Rules for Documentation

The following rules are derived from the user's explicit instructions and implementation rules:

- **Do not modify any existing files in the source repository.** The only permitted write operation is creating the new file `blitzy/documentation/app_2cd6ee777f8c.md`.
- **Base all answers on the code as the truth.** Every claim must be traceable to a specific file and line in the repository. No speculative assertions about behavior that cannot be confirmed by reading the source.
- **Provide thinking and rationale behind the answers.** The document must not merely state conclusions — it must walk the reader through the code evidence that supports each conclusion.
- **Do not make assumptions.** Where the code is ambiguous or where behavior depends on external factors (e.g., PostgreSQL query planner decisions for `.first()` on a non-unique index), the document must explicitly state the ambiguity rather than assuming a particular outcome.
- **If temporary scripts or tooling are needed to observe behavior, clean them up afterward and leave the repository unchanged.** For this analysis, no temporary scripts are needed — the investigation is purely code-reading-based.
- **Place the generated document in the `blitzy/documentation` directory.** The directory must be created if it does not exist.
- **Name the document `app_2cd6ee777f8c.md`** — derived from the source branch name `app_2cd6ee777f8c`.


## 0.11 References

### 0.11.1 Files and Folders Searched

The following files were read and analyzed to derive the conclusions in this Agent Action Plan:

**Primary Reply-Handling Code Path:**

| File | Lines Read | Purpose |
|------|-----------|---------|
| `email_handler.py` | 1–200, 200–350, 345–400, 536–750, 966–1150, 1150–1400, 1390–1440, 1945–2100, 2100–2270, 2288–2405 | Central SMTP handler; `handle()`, `handle_reply()`, `handle_forward()`, `get_or_create_contact()`, `replace_header_when_reply()`, `get_mailbox_from_mail_from()`, `handle_unknown_mailbox()`, `MailHandler` |
| `app/email_utils.py` | 1103–1200 | `generate_reply_email()`, `is_reverse_alias()` |
| `app/email_validation.py` | 1–38 | `normalize_reply_email()`, `is_valid_email()`, allowed character set |
| `app/contact_utils.py` | 1–120 (entire file) | `create_contact()` workflow, `ContactCreateResult`, IntegrityError handling |
| `app/models.py` | 79–100, 1425–1460, 1469–1630, 1863–2020, 2060–2170, 3116–3180 | `ModelMixin.get_by()`, `available_sl_email()`, `Alias` model, `Contact` model, `EmailLog` model, `SLDomain` model |

**Supporting Infrastructure:**

| File | Lines Read | Purpose |
|------|-----------|---------|
| `app/parallel_limiter.py` | 1–73 (entire file) | Redis distributed lock implementation; confirms it is only used in Flask web endpoints, not in SMTP handler |
| `app/utils.py` | 1–80 | `random_string()`, `convert_to_id()`, `convert_to_alphanumeric()`, `canonicalize_email()` |
| `app/email/status.py` | 1–50 (entire file) | SMTP status code definitions (E200, E501, E502, etc.) |
| `Dockerfile` | 1–48 (entire file) | Python 3.10 base image, Gunicorn web server (2 workers), separate process model |
| `pyproject.toml` | 1–135 (entire file) | Python ^3.10, SQLAlchemy 1.3.24 (pinned), aiosmtpd ^1.2, psycopg2-binary ^2.9.3 |

**Database Migration Evidence:**

| File | Search Method | Purpose |
|------|--------------|---------|
| `migrations/versions/2021_071310_78403c7b8089_.py` | grep for `reply_email` across migrations/ | Confirms `ix_contact_reply_email` created with `unique=False` |
| `migrations/versions/2020_031711_0809266d08ca_.py` | grep for `uq_contact` across migrations/ | Confirms `uq_contact` unique constraint on `(alias_id, website_email)` |

**Test Files:**

| File | Lines Read | Purpose |
|------|-----------|---------|
| `tests/test_email_handler.py` | 274–345 | Reply-phase tests: `test_replace_contacts_and_user_in_reply_phase`, `test_send_email_from_non_canonical_address_on_reply` |
| `tests/test_contact_utils.py` | 1–197 (entire file) | Contact creation tests, IntegrityError handling, name/mail_from update tests |

**Existing Documentation Surveyed:**

| File | Purpose |
|------|---------|
| `docs/api.md` | REST API reference (no reply-handling internals documented) |
| `docs/code-structure.md` | Minimal code structure note |
| `docs/troubleshooting.md` | Diagnostic steps (no reply-resolution coverage) |
| `README.md` | Project overview |
| `CONTRIBUTING.md` | Contributor workflow |
| `SECURITY.md` | Vulnerability disclosure policy |

**Folders Explored:**

| Folder | Method | Purpose |
|--------|--------|---------|
| `` (root) | `get_source_folder_contents` | Repository structure overview |
| `app/` | `get_source_folder_contents` | Application package structure |
| `app/handler/` | `get_source_folder_contents` | Handler subpackage (DMARC, complaints, spamd, unsubscribe) |
| `docs/` | `get_source_folder_contents` | Existing documentation inventory |
| `tests/` | `bash find` | Test file discovery for reply and contact handling |
| `migrations/versions/` | `bash grep` | Migration history for Contact table constraints |

**Tech Spec Sections Retrieved:**

| Section | Purpose |
|---------|---------|
| 4.2 Core Email Processing Workflows | Reply-phase and forward-phase flowcharts, routing decision tables |
| 6.2 Database Design | Contact table schema, index strategy, cascade behavior, constraint definitions |
| 6.4 Security Architecture | Concurrency control (parallel_limiter), authentication/authorization, privacy-preserving design |

### 0.11.2 Attachments

No attachments were provided by the user. No Figma URLs or external design resources are referenced.

### 0.11.3 External Resources

No external web searches were performed. The analysis is entirely code-based per the user's instruction to "base answers on the code as the truth."


