# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create a new investigative runtime-behavior document** that traces and explains how SimpleLogin processes multi-step workflows at runtime, answering three specific behavioral questions through code analysis.

- **Category:** Create new documentation
- **Documentation type:** Technical investigative analysis / Runtime behavior trace document

The user's requirements decompose into the following precise documentation objectives:

- **Workflow 1 — Mailbox Verification Code Enforcement:** Document the observable state changes and enforcement mechanisms the running system applies when a mailbox verification code is submitted incorrectly multiple times in succession. Trace the actual runtime behavior to determine what limits exist, how failed attempts are tracked, and what ultimately prevents further submissions.
- **Workflow 2 — Background Task Lifecycle:** Trace the complete lifecycle of background tasks from initial creation through scheduling, pickup, execution, and final completion. Document the recovery and retry behavior applied when a task encounters an error during execution, and what observable state reflects that failure.
- **Workflow 3 — Email Forwarding Bounce Address and Handling:** Document the exact format of the special address (VERP) generated during email forwarding through an alias to handle delivery failures. Trace how the system identifies the original email when a failure notification arrives at this address, what state changes are recorded, and how handling behavior differs depending on whether the original message was in the forward or reply direction.

### 0.1.2 Special Instructions and Constraints

- **Read-only constraint:** The user explicitly states: "source files should not be modified and any temporary artifacts should be cleaned up afterward." This means the output document is an analytical document, not an in-place code documentation effort.
- **Implementation rule (SWE-AtlasQnA-Repo):** The resulting document must be named `<source_branch_name>.md` (resolved to `app_2cd6ee777f8c.md`) and placed in the `blitzy/documentation` directory.
- **No code modifications:** "Do not modify any existing files in the source repository."
- **Code as truth:** "Do not make assumptions, base your answers on the code as the truth."
- **Rationale required:** "Provide thinking / rationale behind the answers."
- **Temporary tools:** Temporary inspection or debugging tools may be used to observe runtime behavior, but must be cleaned up.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **document mailbox verification enforcement**, we will create a section in `blitzy/documentation/app_2cd6ee777f8c.md` that traces the call path through `app/mailbox_utils.py:verify_mailbox_code()`, references the `MailboxActivation` model (`app/models.py:2828`), documents the `MAX_ACTIVATION_TRIES = 3` constant, the 15-minute expiry window, the `tries` counter increment behavior, and the terminal `CannotVerifyError` state with activation code clearing.
- To **document background task lifecycle**, we will create a section tracing `Job.create()` (`app/models.py:2683`), the `JobState` enum (`ready → taken → done`), the polling loop in `job_runner.py`, the `get_jobs_to_run()` query with its 30-minute retry window and 5-attempt maximum, and the state transitions on failure (job remains in `taken` state, re-picked on next eligible poll cycle).
- To **document VERP bounce address format and handling**, we will create a section explaining `generate_verp_email()` (`app/email_utils.py:1438`) with its `{VERP_PREFIX}.{base32_payload}.{base32_signature}@{domain}` structure, the `VerpType` enum (`bounce_forward=0`, `bounce_reply=1`, `transactional=2`), `get_verp_info_from_email()` for parsing, and the divergent handling paths in `handle_bounce_forward_phase()` and `handle_bounce_reply_phase()` within `email_handler.py`.

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following additional documentation needs are inferred:

- The `should_disable()` function in `app/email_utils.py:1166` implements a multi-tier bounce threshold policy (12 bounces/24h, 5+10 bounces/week, 9-day sustained, account-level 10-bounce/4-day) that directly affects the forward-phase bounce workflow. This must be documented as part of Workflow 3.
- The `JobState.error` enum value (3) exists in the model definition but is never explicitly set by the job runner loop, which is a significant behavioral observation that must be documented in Workflow 2.
- The legacy VERP format (using `BOUNCE_PREFIX`/`BOUNCE_SUFFIX` patterns like `bounce+{id}+@domain`) coexists with the newer HMAC-signed VERP format, and the `handle()` function in `email_handler.py` checks both patterns. This dual-format handling must be documented in Workflow 3.
- The `MailboxActivation` record is deleted (not merely marked) upon both successful verification and upon exceeding the maximum tries, meaning the enforcement is destructive. This important behavioral detail must be covered in Workflow 1.


## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a **lightweight, markdown-only documentation structure** with no documentation generator framework in use. The project has no `mkdocs.yml`, `docusaurus.config.js`, `sphinx/conf.py`, or `.readthedocs.yml` configuration. Documentation is hand-authored Markdown files residing at the root level and within the `docs/` directory.

- **Current documentation framework:** None (plain Markdown)
- **API documentation tools:** None detected in `pyproject.toml` dependencies (no Sphinx, MkDocs, or TypeDoc)
- **Diagram tools:** None detected as project dependencies; Mermaid is used within the existing tech spec but not in the repository documentation
- **Documentation hosting/deployment:** Not configured; the `docs/` folder and `README.md` are served directly through the GitHub repository

**Existing documentation inventory:**

| File | Type | Content Summary |
|------|------|-----------------|
| `README.md` | Self-hosting guide | Comprehensive deployment instructions for self-hosted SimpleLogin instances |
| `CONTRIBUTING.md` | Contributor guide | Development setup, PR workflow, and coding conventions |
| `SECURITY.md` | Security policy | Vulnerability disclosure process and contact information |
| `docs/api.md` | API reference | Full REST API endpoint documentation with authentication and error conventions |
| `docs/oauth.md` | OAuth reference | OAuth2/OIDC provider flows, code and implicit grants |
| `docs/build-image.md` | Ops runbook | Multi-architecture Docker image build instructions |
| `docs/upgrade.md` | Ops runbook | Version-to-version upgrade paths and migration steps |
| `docs/ssl.md` | Security guide | TLS certificates, HTTPS, HSTS, MTA-STS configuration |
| `docs/ses.md` | Integration guide | Amazon SES outbound relay configuration |
| `docs/gmail-relay.md` | Integration guide | Gmail SMTP relay setup |
| `docs/postfix-tls.md` | Integration guide | Postfix TLS submission configuration |
| `docs/enforce-spf.md` | Security guide | SPF enforcement with Postfix PCRE rules |
| `docs/ufw.md` | Security guide | Firewall port configuration |
| `docs/troubleshooting.md` | Ops runbook | Diagnostic steps for email delivery issues |
| `docs/code-structure.md` | Architecture note | Brief note about `local_data/` directory (incomplete TODO) |

### 0.2.2 Repository Code Analysis for Documentation

The following source files and directories were examined to identify code paths relevant to the three requested runtime workflows:

**Mailbox verification workflow:**
- `app/mailbox_utils.py` — Core verification logic including `verify_mailbox_code()`, `generate_activation_code()`, `clear_activation_codes_for_mailbox()`, `MAX_ACTIVATION_TRIES`
- `app/models.py:2828` — `MailboxActivation` model with `mailbox_id`, `code`, `tries` columns
- `app/models.py:2710` — `Mailbox` model with `verified` boolean column
- `app/dashboard/views/mailbox.py` — Web dashboard route `/dashboard/mailbox_verify`
- `app/api/views/mailbox.py` — REST API endpoints for mailbox CRUD (no direct code-based verification endpoint)
- `tests/test_mailbox_utils.py` — Test suite validating all verification edge cases

**Background task lifecycle:**
- `job_runner.py` — Main polling loop, `process_job()` dispatcher, `get_jobs_to_run()` query logic
- `app/models.py:2683` — `Job` model with `name`, `payload`, `state`, `attempts`, `taken_at`, `run_at` columns
- `app/models.py:253` — `JobState` enum (`ready=0`, `taken=1`, `done=2`, `error=3`)
- `app/config.py:301-311` — Job name constants (`JOB_ONBOARDING_1` through `JOB_SEND_ALIAS_CREATION_EVENTS`)
- `app/config.py:564-565` — `JOB_MAX_ATTEMPTS = 5`, `JOB_TAKEN_RETRY_WAIT_MINS = 30`
- `app/models.py:612-668` — `User.create()` method scheduling onboarding jobs via `Job.create()`
- `app/mailbox_utils.py:141-156` — Mailbox deletion scheduling via `Job.create()`

**Email forwarding bounce handling:**
- `email_handler.py:880-928` — Forward phase VERP envelope generation
- `email_handler.py:1220-1260` — Reply phase VERP envelope generation
- `email_handler.py:1432-1592` — `handle_bounce_forward_phase()` implementation
- `email_handler.py:1595-1687` — `handle_bounce_reply_phase()` implementation
- `email_handler.py:1813-1914` — `is_bounce()`, `handle_transactional_bounce()`, `handle_bounce()` dispatcher
- `email_handler.py:2034-2098` — VERP routing in the main `handle()` function
- `app/email_utils.py:1438-1498` — `generate_verp_email()` and `get_verp_info_from_email()` (VERP format)
- `app/email_utils.py:1166-1255` — `should_disable()` multi-tier bounce threshold logic
- `app/models.py:247-251` — `VerpType` enum (`bounce_forward=0`, `bounce_reply=1`, `transactional=2`)
- `app/models.py:2060-2150` — `EmailLog` model with `bounced`, `bounced_mailbox_id`, `refused_email_id` columns
- `app/models.py:3290-3297` — `Bounce` model for bounce tracking records
- `app/config.py:99-117` — `BOUNCE_PREFIX`, `BOUNCE_SUFFIX`, `BOUNCE_PREFIX_FOR_REPLY_PHASE`, `TRANSACTIONAL_BOUNCE_PREFIX/SUFFIX`
- `app/config.py:499-507` — `VERP_PREFIX`, `VERP_EMAIL_SECRET`, `VERP_MESSAGE_LIFETIME`

### 0.2.3 Web Search Research Conducted

No external web search was required for this documentation task. All three workflows are fully documented through direct code analysis of the repository source files. The codebase serves as the single source of truth per the user's instruction: "base your answers on the code as the truth."


## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

**Module: `app/mailbox_utils.py` — Mailbox Verification Enforcement**
- Public APIs: `verify_mailbox_code()`, `generate_activation_code()`, `clear_activation_codes_for_mailbox()`, `create_mailbox()`
- Current documentation: No runtime behavior documentation exists
- Documentation needed: Detailed trace of the verification state machine, enforcement limits, and observable state transitions

**Module: `app/models.py` — Data Models for All Three Workflows**
- Key models: `MailboxActivation` (lines 2828–2836), `Job` (lines 2683–2707), `JobState` (lines 253–257), `VerpType` (lines 247–251), `EmailLog` (lines 2060–2150), `Bounce` (lines 3290–3297), `RefusedEmail` (lines 2858–2870)
- Current documentation: No behavioral documentation; only code comments
- Documentation needed: State enumeration values, column semantics, and transition rules

**Module: `job_runner.py` — Background Task Lifecycle**
- Public APIs: `process_job()`, `get_jobs_to_run()`, main polling loop
- Current documentation: File-level docstring ("Not meant for running job at precise time (+- 1h)")
- Documentation needed: Full lifecycle trace from Job.create() through final state, retry semantics, failure recovery

**Module: `email_handler.py` — Bounce Handling Workflows**
- Public APIs: `handle_bounce_forward_phase()`, `handle_bounce_reply_phase()`, `handle_bounce()`, `handle_transactional_bounce()`
- Current documentation: Inline comments only
- Documentation needed: VERP address format specification, bounce detection logic, divergent forward-vs-reply handling, auto-disable threshold rules

**Module: `app/email_utils.py` — VERP Generation and Bounce Thresholds**
- Public APIs: `generate_verp_email()`, `get_verp_info_from_email()`, `should_disable()`, `parse_id_from_bounce()`
- Current documentation: Inline docstrings only
- Documentation needed: Address format breakdown, HMAC signing mechanics, multi-tier disable heuristics

**Module: `app/config.py` — Configuration Constants**
- Key constants: `JOB_MAX_ATTEMPTS`, `JOB_TAKEN_RETRY_WAIT_MINS`, `VERP_PREFIX`, `VERP_EMAIL_SECRET`, `VERP_MESSAGE_LIFETIME`, `BOUNCE_PREFIX`, `BOUNCE_SUFFIX`, `MAX_ACTIVATION_TRIES` (in mailbox_utils)
- Current documentation: Inline comments and `example.env`
- Documentation needed: Referenced within the behavioral traces as governing parameters

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, the following documentation gaps are being addressed by this deliverable:

- **No existing runtime workflow documentation:** The repository contains deployment guides, API references, and operational runbooks but no document that traces runtime behavior of internal multi-step processes. This deliverable fills that gap for the three specified workflows.
- **No mailbox verification behavioral specification:** While the API reference (`docs/api.md`) documents the mailbox creation endpoint, it does not describe the verification code enforcement mechanism, attempt limits, or expiry behavior.
- **No job system architectural documentation:** The `docs/code-structure.md` file is a stub (TODO) and does not describe the job system. No existing documentation covers the `Job` model's state machine, retry semantics, or the `job_runner.py` processing loop.
- **No VERP format specification:** Despite VERP being central to bounce handling, no documentation describes the address format, the HMAC signing algorithm, or how bounces are attributed back to original email deliveries.
- **No bounce threshold documentation:** The `should_disable()` function implements a complex, multi-tier bounce threshold policy, but no documentation describes the specific thresholds or their rationale.


## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The deliverable is a single comprehensive Markdown document placed at `blitzy/documentation/app_2cd6ee777f8c.md` per the implementation rules. The document structure is organized around the three workflows requested:

```
blitzy/documentation/
└── app_2cd6ee777f8c.md
    ├── Introduction (scope statement and methodology)
    ├── Workflow 1: Mailbox Verification Code Enforcement
    │   ├── Activation Code Generation
    │   ├── Verification Attempt Processing
    │   ├── State Transitions and Enforcement Limits
    │   ├── Terminal States and Recovery
    │   └── Mermaid: Verification State Machine Diagram
    ├── Workflow 2: Background Task Lifecycle
    │   ├── Job Creation and Scheduling
    │   ├── Job Eligibility Query
    │   ├── Job Processing Loop
    │   ├── Error Recovery and Retry Behavior
    │   ├── Observable State Reflecting Failure
    │   └── Mermaid: Job State Machine Diagram
    ├── Workflow 3: Email Forwarding Bounce Address and Handling
    │   ├── VERP Address Format Specification
    │   ├── Bounce Detection and Routing
    │   ├── Forward-Phase Bounce Handling
    │   ├── Reply-Phase Bounce Handling
    │   ├── Alias Auto-Disable Threshold Rules
    │   └── Mermaid: Bounce Handling Flow Diagram
    └── Source References
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**
- Extract verification enforcement logic from `app/mailbox_utils.py:verify_mailbox_code()` by tracing each conditional branch
- Extract job lifecycle state transitions from `job_runner.py` main loop and `get_jobs_to_run()` query
- Extract VERP format from `app/email_utils.py:generate_verp_email()` by analyzing the encoding algorithm
- Extract bounce handling divergence from `email_handler.py:handle_bounce_forward_phase()` and `handle_bounce_reply_phase()`
- Validate behavioral claims against test cases in `tests/test_mailbox_utils.py` and `tests/test_email_utils.py`

**Documentation Standards:**
- Markdown formatting with proper hierarchical headers (`#`, `##`, `###`)
- Mermaid diagram integration using fenced code blocks for state machines and flowcharts
- Code references using the format: `Source: /path/to/file.py:LineNumber`
- Tables for structured data (state enumerations, threshold rules, configuration constants)
- Consistent terminology aligned with the codebase (e.g., "activation" not "verification code", "VERP" not "bounce address")

### 0.4.3 Diagram and Visual Strategy

The following Mermaid diagrams will be included in the deliverable document:

- **Mailbox Verification State Machine** — A stateDiagram-v2 showing transitions between code-generated, attempt-incremented, expired, max-tries-reached, and verified states
- **Job Lifecycle State Machine** — A stateDiagram-v2 showing the `JobState` transitions (`ready → taken → done`) with retry loops and the implicit stale-taken recovery path
- **Bounce Handling Flowchart** — A flowchart showing the VERP routing decision tree and the divergent forward-phase vs. reply-phase handling paths, including S3 archival, state persistence, and auto-disable evaluation


## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | CREATE | `app/mailbox_utils.py`, `job_runner.py`, `email_handler.py`, `app/email_utils.py`, `app/models.py`, `app/config.py` | Comprehensive runtime behavior analysis answering all three workflow questions with code-traced rationale, Mermaid diagrams, and source citations |

No existing files are updated or deleted. The deliverable is a single new Markdown file.

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/app_2cd6ee777f8c.md
Type: Technical investigative analysis / Runtime behavior trace
Source Code:
  - app/mailbox_utils.py (verification logic)
  - app/models.py (MailboxActivation, Job, JobState, VerpType, EmailLog, Bounce, RefusedEmail)
  - app/config.py (JOB_MAX_ATTEMPTS, JOB_TAKEN_RETRY_WAIT_MINS, VERP_PREFIX, BOUNCE_PREFIX, etc.)
  - job_runner.py (job polling loop, process_job dispatcher, get_jobs_to_run query)
  - email_handler.py (handle_bounce_forward_phase, handle_bounce_reply_phase, VERP routing)
  - app/email_utils.py (generate_verp_email, get_verp_info_from_email, should_disable)
  - app/dashboard/views/mailbox.py (web route for mailbox_verify)
  - tests/test_mailbox_utils.py (behavioral validation of verification enforcement)

Sections:
  - Introduction: Scope and methodology statement
  - Workflow 1 — Mailbox Verification Code Enforcement:
    - How activation codes are generated (random 6-digit or token_urlsafe)
    - The verify_mailbox_code() call path with each conditional branch
    - MAX_ACTIVATION_TRIES=3 enforcement, tries counter increment
    - 15-minute expiry window
    - Terminal states: verified=True or activation deleted
    - State machine diagram
  - Workflow 2 — Background Task Lifecycle:
    - Job.create() with ready state and run_at scheduling
    - get_jobs_to_run() eligibility query (ready OR stale-taken with <5 attempts)
    - Processing loop: taken → process_job() → done
    - Error behavior: job stays in taken state, re-eligible after 30 mins
    - JOB_MAX_ATTEMPTS=5 as terminal failure threshold
    - JobState.error exists but is never set by runner
    - State machine diagram
  - Workflow 3 — Email Forwarding Bounce Handling:
    - VERP format: sl.{b32_payload}.{b32_signature}@{domain}
    - Payload encoding: [verp_type, email_log_id, time_minutes]
    - HMAC-SHA3-224 signing with VERP_EMAIL_SECRET
    - Bounce detection: empty mail_from + multipart/report
    - Forward-phase handling: Bounce record, S3 upload, RefusedEmail, EmailLog.bounced, should_disable() evaluation
    - Reply-phase handling: Bounce record, S3 upload, RefusedEmail, EmailLog.bounced, notification only (no auto-disable)
    - Auto-disable thresholds (4 tiers)
    - Flowchart diagram
  - Source References: Full list of files examined

Diagrams:
  - stateDiagram-v2: Mailbox verification state machine
  - stateDiagram-v2: Job lifecycle state machine
  - flowchart TD: Bounce handling with forward vs. reply divergence

Key Citations:
  - app/mailbox_utils.py:43 (MAX_ACTIVATION_TRIES)
  - app/mailbox_utils.py:166-220 (verify_mailbox_code)
  - job_runner.py:307-326 (get_jobs_to_run)
  - job_runner.py:329-347 (main loop)
  - app/config.py:564-565 (JOB_MAX_ATTEMPTS, JOB_TAKEN_RETRY_WAIT_MINS)
  - app/email_utils.py:1438-1498 (VERP generation and parsing)
  - app/email_utils.py:1166-1255 (should_disable)
  - email_handler.py:1432-1592 (handle_bounce_forward_phase)
  - email_handler.py:1595-1687 (handle_bounce_reply_phase)
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration files need to be updated. The project does not use a documentation generator framework. The new file is placed in the `blitzy/documentation/` directory as a standalone Markdown document per the implementation rules.

### 0.5.4 Cross-Documentation Dependencies

- **No cross-document dependencies:** The deliverable is a self-contained analysis document. It does not require navigation links to or from existing documentation files.
- **No table of contents updates:** The existing `README.md` and `docs/` files do not reference a central index that would need updating.
- **No glossary updates:** The document will define all domain terms (VERP, activation code, bounce, etc.) inline where first used.


## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

No documentation tooling packages need to be installed for this task. The deliverable is a hand-authored Markdown file with embedded Mermaid diagram syntax. Mermaid rendering is handled natively by GitHub's Markdown renderer and does not require a build step.

The following project runtime dependencies are **referenced** within the documentation (not installed for documentation generation):

| Registry | Package Name | Version | Relevance to Documentation |
|----------|--------------|---------|---------------------------|
| PyPI | flask | ^1.1.2 | Web framework hosting the `/dashboard/mailbox_verify` route |
| PyPI | sqlalchemy | 1.3.24 | ORM layer for Job, MailboxActivation, EmailLog, Bounce models |
| PyPI | arrow | ^0.16.0 | Timestamp arithmetic in verification expiry and job scheduling |
| PyPI | aiosmtpd | ^1.2 | SMTP server handling inbound email and bounce reception |
| PyPI | dkimpy | ^1.0.5 | DKIM signing applied before VERP envelope delivery |
| PyPI | boto3 | ^1.15.9 | S3 uploads for bounce report and original message archival |
| PyPI | sentry_sdk | ^2.16.0 | Error tracking integrated into job runner context |
| PyPI | newrelic | 8.8.0 | APM instrumentation in email handler and job processing |
| PyPI | psycopg2-binary | ^2.9.3 | PostgreSQL driver for all database state transitions |
| PyPI | redis | ^4.5.3 | Session and rate-limiting backend referenced in email handling |

### 0.6.2 Documentation Reference Updates

No documentation link updates are required. The new document (`blitzy/documentation/app_2cd6ee777f8c.md`) is a standalone deliverable and does not introduce or modify hyperlinks in any existing documentation files.


## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

The scope of this document is a targeted investigation of three specific runtime workflows. Coverage is measured against the user's explicit questions:

| Workflow | Question Coverage | Target |
|----------|------------------|--------|
| Mailbox verification enforcement | Limits, tracking, prevention of further submissions | 100% of user questions answered |
| Background task lifecycle | Creation → scheduling → pickup → execution → completion, error/retry | 100% of user questions answered |
| Email forwarding bounce handling | VERP format, original email identification, state changes, direction-dependent handling | 100% of user questions answered |

**Current coverage of these workflows in existing documentation: 0%** — No existing document in the repository addresses any of these runtime behaviors.

**Target coverage: 100%** — Every sub-question posed by the user must be answered with code-traced evidence.

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every behavioral claim must cite the specific source file and line number
- All state transitions must be enumerated, not summarized
- All threshold values must be quoted as exact constants from the code
- All conditional branches in the traced call paths must be documented

**Accuracy validation:**
- Behavioral claims are cross-referenced against test cases in `tests/test_mailbox_utils.py` (e.g., `test_verify_fail`, `test_verify_too_may`, `test_verify_too_old_code`, `test_verify_ok`)
- VERP format claims are validated against `tests/test_email_utils.py` (e.g., `test_generate_verp_email`, `test_generate_verp_email_forward_reply_phase`)
- Job retry semantics are validated against the `get_jobs_to_run()` query logic with exact SQLAlchemy filter conditions

**Clarity standards:**
- Technical accuracy with accessible language for developers unfamiliar with the SimpleLogin codebase
- Progressive disclosure: overview statement before detailed code trace
- Consistent terminology: use code identifiers (e.g., `MailboxActivation.tries`, `JobState.taken`) to match source

**Maintainability:**
- Source citations include file paths and line numbers for traceability
- Mermaid diagrams can be updated independently of prose

### 0.7.3 Example and Diagram Requirements

- Minimum 1 Mermaid state diagram per workflow (3 total)
- Minimum 1 summary table per workflow documenting key state values and transitions
- Code references must use the citation format: `Source: path/to/file.py:LineNumber`
- No executable code examples are required (this is an analysis document, not an API guide)


## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `blitzy/documentation/app_2cd6ee777f8c.md` — The sole deliverable

**Source files analyzed (read-only):**
- `app/mailbox_utils.py` — Mailbox verification logic
- `app/models.py` — All relevant ORM models (MailboxActivation, Job, JobState, VerpType, EmailLog, Bounce, RefusedEmail, Mailbox)
- `app/config.py` — All relevant configuration constants
- `job_runner.py` — Background task processing loop
- `email_handler.py` — Inbound email handling, VERP routing, bounce processing
- `app/email_utils.py` — VERP generation/parsing, bounce thresholds, email utilities
- `app/errors.py` — Exception hierarchy (VERPTransactional, VERPForward, VERPReply, CannotVerifyError)
- `app/dashboard/views/mailbox.py` — Web dashboard verification route
- `app/api/views/mailbox.py` — REST API mailbox endpoints
- `app/handler/*.py` — Handler subpackage (DMARC, complaints, unsubscribe)
- `tests/test_mailbox_utils.py` — Test suite for verification behavior validation
- `tests/test_email_utils.py` — Test suite for VERP and bounce threshold validation
- `cron.py` — Scheduled maintenance operations (context for job system)
- `app/s3.py` — S3 storage operations referenced in bounce archival

**Behavioral topics exhaustively documented:**
- Mailbox verification code generation, submission, enforcement limits, expiry, and terminal states
- Job creation, scheduling, polling, eligibility, processing, state transitions, retry, and terminal failure
- VERP address format (encoding, signing, parsing), bounce detection, forward-phase handling, reply-phase handling, auto-disable thresholds

### 0.8.2 Explicitly Out of Scope

- **Source code modifications:** No existing files in the source repository will be modified per the user's explicit instruction and the implementation rule
- **Docstring additions:** Adding or modifying inline documentation comments within source files is out of scope
- **Test file modifications:** No test files will be created or modified
- **Feature additions or code refactoring:** This is purely a documentation exercise
- **Deployment configuration changes:** No Docker, Postfix, or infrastructure changes
- **Documentation of unrelated workflows:** Only the three specified workflows (mailbox verification, job lifecycle, bounce handling) are documented. Other workflows such as alias creation, PGP encryption, DMARC enforcement, spam handling, OAuth flows, and subscription management are out of scope
- **Performance analysis:** No benchmarking or performance profiling of the traced workflows
- **Security vulnerability disclosure:** Runtime behavior is described factually; no security recommendations are made
- **Existing documentation updates:** The `README.md`, `docs/api.md`, and other existing documentation files are not modified


## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** Not applicable — the deliverable is a standalone Markdown file with no build step
- **Documentation preview command:** View `blitzy/documentation/app_2cd6ee777f8c.md` directly in any Markdown renderer (GitHub, VS Code, etc.)
- **Diagram generation command:** Mermaid diagrams are embedded inline using fenced code blocks and render natively in GitHub; no external generation tool is required
- **Documentation deployment command:** Not applicable — the file is committed directly to the repository
- **Default format:** Markdown with embedded Mermaid diagrams
- **Citation requirement:** Every behavioral claim must reference the source file path and line number
- **Style guide:** Follow the analytical prose style, with clear section headings, summary tables, and embedded diagrams. Use code identifiers from the source (e.g., `MailboxActivation.tries`, `JobState.taken`) consistently
- **Documentation validation:** Manual review against the user's original questions to verify every sub-question is answered with code-traced evidence

### 0.9.2 Output File Naming Convention

Per the implementation rule (SWE-AtlasQnA-Repo), the document is named after the source branch:

- **Source branch name:** `app_2cd6ee777f8c`
- **Output file path:** `blitzy/documentation/app_2cd6ee777f8c.md`
- **Output directory:** `blitzy/documentation/` (to be created if it does not exist)


## 0.10 Rules for Documentation

The following rules are derived from the user's explicit instructions and the implementation rule set:

- **Do not modify any existing files in the source repository.** The deliverable is a new file only. No source files, test files, configuration files, or existing documentation files may be changed.
- **Do not make assumptions; base all answers on the code as the truth.** Every behavioral claim in the document must be traceable to a specific line of source code. Speculative or inferred behavior that cannot be verified in the codebase must not be presented as fact.
- **Provide thinking and rationale behind the answers.** The document must explain not just what the code does, but why each behavioral observation follows from the code structure. For example, explain why the job runner's failure mode leaves jobs in `taken` state rather than setting `error`, and what implication this has for retry behavior.
- **Temporary inspection or debugging tools may be used to observe runtime behavior, but source files should not be modified and any temporary artifacts should be cleaned up afterward.** If any temporary scripts or tools are created during analysis, they must be removed before the task is complete.
- **Place the generated document in the `blitzy/documentation` directory.** The output file must be at `blitzy/documentation/app_2cd6ee777f8c.md`.
- **Name the document `<source_branch_name>.md`.** The resolved file name is `app_2cd6ee777f8c.md`.
- **Comprehensively answer all questions posed in the prompt.** Every sub-question across all three workflows must receive a direct, evidence-backed answer. No question may be deferred or left unanswered.


## 0.11 References

### 0.11.1 Files and Folders Searched

The following files were retrieved and analyzed during context gathering to derive the conclusions in this Agent Action Plan:

**Core workflow source files:**
- `app/mailbox_utils.py` — Mailbox creation, verification code generation, `verify_mailbox_code()`, `MAX_ACTIVATION_TRIES`, `clear_activation_codes_for_mailbox()`
- `job_runner.py` — Background job polling loop, `process_job()` dispatcher, `get_jobs_to_run()` eligibility query, all job handler functions
- `email_handler.py` — Inbound SMTP handler, VERP routing in `handle()`, `handle_bounce_forward_phase()`, `handle_bounce_reply_phase()`, `handle_bounce()`, `handle_transactional_bounce()`, `is_bounce()`, forward-phase and reply-phase VERP envelope generation
- `app/email_utils.py` — `generate_verp_email()`, `get_verp_info_from_email()`, `should_disable()`, `parse_id_from_bounce()`, `get_orig_message_from_bounce()`, `get_mailbox_bounce_info()`
- `app/models.py` — `MailboxActivation`, `Mailbox`, `Job`, `JobState`, `VerpType`, `EmailLog`, `Bounce`, `RefusedEmail`, `TransactionalEmail`, `User.create()` onboarding job scheduling
- `app/config.py` — `JOB_MAX_ATTEMPTS`, `JOB_TAKEN_RETRY_WAIT_MINS`, `VERP_PREFIX`, `VERP_EMAIL_SECRET`, `VERP_MESSAGE_LIFETIME`, `BOUNCE_PREFIX`, `BOUNCE_SUFFIX`, `BOUNCE_PREFIX_FOR_REPLY_PHASE`, `TRANSACTIONAL_BOUNCE_PREFIX`, `TRANSACTIONAL_BOUNCE_SUFFIX`, `ALIAS_AUTOMATIC_DISABLE`, all `JOB_*` name constants
- `app/errors.py` — `SLException`, `VERPTransactional`, `VERPForward`, `VERPReply`, `CannotCreateContactForReverseAlias`

**Dashboard and API routes:**
- `app/dashboard/views/mailbox.py` — `/dashboard/mailbox_verify` route, `verify_with_signed_secret()` legacy path
- `app/api/views/mailbox.py` — REST API endpoints for mailbox CRUD operations

**Test files (for behavioral validation):**
- `tests/test_mailbox_utils.py` — Tests for `verify_mailbox_code()` including `test_verify_fail`, `test_verify_too_may`, `test_verify_too_old_code`, `test_verify_ok`, `test_verify_non_existing_mailbox`, `test_verify_already_verified_mailbox`, `test_verify_other_users_mailbox`
- `tests/test_email_utils.py` — Tests for VERP generation (`test_generate_verp_email`, `test_generate_verp_email_forward_reply_phase`) and bounce thresholds (`test_should_disable_bounces_every_day`, `test_should_disable_bounces_account`, `test_should_disable_bounce_consecutive_days`)

**Supporting infrastructure files:**
- `app/handler/` (folder) — Handler subpackage for DMARC, provider complaints, spamd, unsubscribe
- `cron.py` — Scheduled maintenance operations providing context for the job system
- `app/s3.py` — S3 storage abstraction referenced in bounce archival
- `app/mail_sender.py` — SMTP delivery and VERP-based `sl_sendmail()` entry point

**Documentation and configuration files:**
- `README.md` — Self-hosting guide (reviewed for existing documentation coverage)
- `CONTRIBUTING.md` — Contributor guidelines (reviewed for documentation style)
- `docs/` (entire folder) — All 13 documentation files reviewed for existing coverage of the three workflows
- `pyproject.toml` — Dependency manifest reviewed for runtime versions and documentation tool detection
- `docs/code-structure.md` — Incomplete architecture note (confirmed gap)

**Folder structures explored:**
- Root (`/`) — Repository root structure and all first-level children
- `app/` — Main application package and all subpackages
- `app/handler/` — Email handler subpackage
- `app/dashboard/` — Dashboard blueprint and views
- `docs/` — Existing documentation files

### 0.11.2 Attachments

No attachments were provided by the user. No Figma screens or external design files are referenced.

### 0.11.3 External URLs

No external URLs were referenced or required. All analysis is based solely on the repository source code.


