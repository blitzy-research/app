# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create a new, comprehensive investigative analysis document** that examines SimpleLogin's bounce email handling system for potential information leakage vulnerabilities — specifically documenting the security gap between the legacy (unsigned) bounce address format and the newer HMAC-signed VERP format, the exact SMTP response behaviors observable by an external attacker, the bounce detection criteria and their spoofability, and the true security boundaries of the system as evidenced by actual code paths rather than theoretical descriptions.

**Request Category:** Create new documentation

**Documentation Type:** Security investigation / Technical analysis document

**Detailed Requirement Breakdown:**

- **R1 — Dual Bounce Format Analysis:** Document and contrast the two bounce address formats in the codebase — the older plaintext format (`bounce+{email_log_id}+@domain`) and the newer HMAC-signed format (`{VERP_PREFIX}.{base32_payload}.{base32_signature}@domain`) — including how they are generated in `app/email_utils.py:generate_verp_email()` and parsed in `app/email_utils.py:get_verp_info_from_email()` and `app/email_utils.py:parse_id_from_bounce()`.

- **R2 — Bounce Routing Mechanics:** Document the complete routing decision tree in `email_handler.py:handle()` (lines 2034–2117) that determines how inbound emails to bounce addresses are classified and dispatched, including the parallel legacy/VERP format checks for transactional, forward, and reply bounce phases.

- **R3 — SMTP Response Differential Analysis:** Provide evidence-based documentation of the exact SMTP response codes returned under various probing scenarios — valid vs. invalid email log IDs, properly formatted bounces vs. non-bounce messages, signed vs. unsigned addresses — by tracing the code paths in `email_handler.py` and `app/email/status.py`.

- **R4 — Bounce Detection Criteria and Spoofability:** Document the `is_bounce()` function's validation criteria (`envelope.mail_from == "<>"` and `Content-Type == multipart/report`) and analyze whether an attacker controlling the SMTP envelope and headers can satisfy these requirements.

- **R5 — Information Leakage Surface Assessment:** Document the specific information that can be inferred from differential SMTP responses when probing the old-format bounce addresses, including email log ID existence enumeration and activity timing inference.

- **R6 — Security Boundary Mapping:** Document where cryptographic validation exists (new VERP format) versus where it is absent (old format), and what the observable behavioral differences are between the two paths.

**Inferred Documentation Needs:**

- Based on code analysis: The `parse_id_from_bounce()` function in `app/email_utils.py` (line 1258) performs no cryptographic verification, extracting the integer directly from the address string — this requires detailed documentation of the security implications.
- Based on structure: The bounce handling spans `email_handler.py`, `app/email_utils.py`, `app/config.py`, `app/email/status.py`, `app/errors.py`, and `app/models.py` — requiring a consolidated cross-module analysis document.
- Based on the SPF downgrade mechanism: The `_handle()` wrapper (lines 2357–2365) that converts 5xx responses to E216 when SPF fails needs documentation as it partially masks response differentials.
- Based on the rate limiting state: Rate limiting is currently disabled (`app/email/rate_limit.py` line 97, `return False`), meaning there is no throttle on probing attempts — this warrants documentation.

### 0.1.2 Special Instructions and Constraints

**Critical Directives:**
- **No source file modifications:** The user explicitly states "Don't modify any source files in the repository." All analysis must be purely observational.
- **Test scripts allowed but must be cleaned up:** "You can create test scripts to observe the actual behavior, but clean them up when finished." — however, running live tests is infeasible without a full database/SMTP stack, so documentation must trace code paths with citations.
- **Evidence-based, not theoretical:** "I don't want theoretical explanations of what the code should do, I want to see actual evidence of how the system behaves in practice." — documentation must reference specific line numbers, functions, and code-path analysis.

**Implementation Rule (SWE-AtlasQnA-Repo):**
- Create a new markdown document named `app_2cd6ee777f8c.md` (matching the source branch name)
- Place the document in the `blitzy/documentation` directory
- Provide thinking and rationale behind the answers
- Base all answers on the code as the truth, do not make assumptions
- Do not modify any existing files in the source repository

**Style Preferences:**
- Deep technical analysis with specific code citations (file:line references)
- Actual code-path tracing over theoretical speculation
- Clear labeling of security-relevant behavior differentials
- Mermaid diagrams for flow visualization where appropriate

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **document the dual bounce format** (R1), we will **create** a section in `blitzy/documentation/app_2cd6ee777f8c.md` that traces the generation path in `app/email_utils.py:generate_verp_email()` (lines 1438–1464) and both parsing paths: `parse_id_from_bounce()` (line 1258–1259) for the legacy format and `get_verp_info_from_email()` (lines 1467–1498) for the signed format, contrasting the presence vs. absence of cryptographic validation.

- To **document bounce routing** (R2), we will **create** a detailed walkthrough of the `handle()` function's VERP routing region (lines 2034–2117) including the three parallel detection blocks (transactional, forward, reply), the iCloud special case, and the priority ordering of signed vs. unsigned address checks.

- To **document SMTP response differentials** (R3), we will **create** a comprehensive table mapping each probe scenario to its exact code path and resulting SMTP status code from `app/email/status.py`, including the SPF-based downgrade in `MailHandler._handle()` (lines 2357–2365).

- To **document bounce detection criteria** (R4), we will **create** analysis of `is_bounce()` (lines 1813–1818) and `is_automatic_out_of_office()` (lines 1793–1810), documenting how each criterion can be externally satisfied.

- To **document the information leakage surface** (R5), we will **create** a security assessment section documenting the enumeration attack surface via the old format, including what an attacker learns from each distinct response code.

- To **document security boundaries** (R6), we will **create** a boundary map contrasting the HMAC-protected path (`get_verp_info_from_email()`) with the unprotected path (`parse_id_from_bounce()`).

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a **minimal, operationally-focused documentation structure** with no existing documentation covering bounce handling security, VERP implementation details, or SMTP response behavior analysis.

**Documentation File Discovery:**

| Path | Type | Relevance to This Task |
|------|------|------------------------|
| `README.md` | Self-hosting guide | Low — covers deployment setup, not bounce internals |
| `SECURITY.md` | Vulnerability disclosure policy | Medium — establishes security reporting but contains no technical analysis |
| `CONTRIBUTING.md` | Contributor guidelines | Low — development workflow only |
| `docs/api.md` | REST API reference | Low — HTTP API, not SMTP behavior |
| `docs/troubleshooting.md` | Operational troubleshooting | Medium — includes SMTP diagnostic steps but no bounce analysis |
| `docs/enforce-spf.md` | SPF hardening guide | Medium — related SPF enforcement topic |
| `docs/ssl.md` | TLS/certificate setup | Low — transport security, not bounce logic |
| `docs/ses.md` | Amazon SES relay configuration | Low — outbound relay setup |
| `docs/gmail-relay.md` | Gmail SMTP relay setup | Low — outbound relay setup |
| `docs/code-structure.md` | Code organization notes | Low — incomplete TODO-style note about `local_data/` |
| `docs/upgrade.md` | Version upgrade runbook | Low — container lifecycle, not email handling |
| `docs/build-image.md` | Docker image build instructions | Low — build process only |
| `docs/oauth.md` | OAuth/OIDC documentation | Low — authentication, not email |

**Documentation Infrastructure:**
- No documentation generator (no `mkdocs.yml`, `docusaurus.config.js`, or `sphinx.conf.py` detected)
- No automated API documentation tools (no JSDoc, Sphinx autodoc, or TypeDoc configuration)
- Documentation is manually authored Markdown files
- No existing diagram tools configured (no Mermaid CLI, PlantUML, etc.)
- The `blitzy/documentation/` directory exists but is currently empty — this is the designated output location per the implementation rules

**Key Finding:** There is no existing documentation covering bounce handling, VERP implementation, SMTP response behaviors, or the security implications of the dual address format. This is a net-new documentation creation task.

### 0.2.2 Repository Code Analysis for Documentation

**Search Patterns Used for Bounce-Related Code:**

| Search Pattern | Files Found | Key Findings |
|----------------|-------------|---------------|
| Bounce address format handling | `email_handler.py` (lines 2034–2117) | Dual-format parallel checking: old prefix/suffix AND new signed VERP |
| VERP generation and parsing | `app/email_utils.py` (lines 1438–1498) | `generate_verp_email()` creates signed addresses; `get_verp_info_from_email()` validates them |
| Legacy ID extraction | `app/email_utils.py` (line 1258–1259) | `parse_id_from_bounce()` — no validation, simple string extraction |
| Bounce detection | `email_handler.py` (lines 1813–1818) | `is_bounce()` checks mail_from == `<>` AND Content-Type == `multipart/report` |
| Bounce config constants | `app/config.py` (lines 99–118) | `BOUNCE_PREFIX`, `BOUNCE_SUFFIX`, `BOUNCE_PREFIX_FOR_REPLY_PHASE`, `TRANSACTIONAL_BOUNCE_PREFIX/SUFFIX` |
| VERP signing config | `app/config.py` (lines 498–508) | `VERP_PREFIX`, `VERP_EMAIL_SECRET` (min 32 chars), `VERP_MESSAGE_LIFETIME` (5 days) |
| SMTP status codes | `app/email/status.py` (lines 1–65) | Complete mapping of E200–E525 status codes |
| VERP type enumeration | `app/models.py` (lines 247–250) | `VerpType`: `bounce_forward=0`, `bounce_reply=1`, `transactional=2` |
| Bounce record persistence | `app/models.py` (lines 3290–3298) | `Bounce` table — email + info fields, 7-day retention |
| Email log model | `app/models.py` (lines 2060–2150) | `EmailLog` — bounce tracking fields: `bounced`, `bounced_mailbox_id`, `refused_email_id` |
| Bounce-related error classes | `app/errors.py` (lines 42–57) | `VERPTransactional`, `VERPForward`, `VERPReply` exceptions |
| OOO detection | `email_handler.py` (lines 1793–1810) | `is_automatic_out_of_office()` — checks `Auto-Submitted` header |
| SPF downgrade mechanism | `email_handler.py` (lines 2357–2365) | Converts 5xx to E216 when SPF fails, prevents backscatter |
| Alias disable logic | `app/email_utils.py` (lines 1166–1255) | `should_disable()` — multi-tier bounce-count thresholds |
| Rate limiting status | `app/email/rate_limit.py` (line 97) | Rate limiting is **disabled** (`return False`) |

**Existing Tests Found:**

| Test File | Coverage |
|-----------|----------|
| `tests/test_email_utils.py:test_parse_id_from_bounce` (line 789) | Verifies basic parsing of `bounces+1234+@local` |
| `tests/test_email_utils.py:test_generate_verp_email` (line 848) | Verifies VERP generation and round-trip verification |
| `tests/test_email_utils.py:test_generate_verp_email_forward_reply_phase` (line 858) | Verifies VerpType encoding across all three types |
| `tests/test_email_utils.py:test_should_ignore_bounce` (line 807) | Tests `IgnoreBounceSender` integration |
| `tests/test_email_utils.py:test_get_orig_message_from_bounce` (line 819) | Parses original message from bounce report EML fixture |
| `tests/test_email_utils.py:test_get_mailbox_bounce_info` (line 828) | Extracts bounce info from bounce report |
| `tests/test_email_handler.py:test_prevent_5xx_from_spf` (line 115) | Tests SPF-based 5xx downgrade to E216 |
| `tests/test_email_handler.py:test_preserve_5xx_with_valid_spf` (line 130) | Confirms 5xx preserved when SPF passes |

### 0.2.3 Web Search Research Conducted

No web search is required for this documentation task. The investigation is entirely code-driven and all technical conclusions are derived from the source code as the ground truth, as mandated by the user's instructions. The codebase contains all necessary evidence to document the bounce handling behavior, SMTP response differentials, and security boundary analysis.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

**Module: `email_handler.py` (Central SMTP Handler)**
- Public APIs/Functions requiring documentation:
  - `handle()` (line 1945) — Main routing hub, bounce VERP detection region (lines 2034–2117)
  - `is_bounce()` (line 1813) — Bounce detection criteria
  - `is_automatic_out_of_office()` (line 1793) — OOO detection criteria
  - `handle_bounce()` (line 1851) — Central bounce dispatch
  - `handle_bounce_forward_phase()` (line 1432) — Forward bounce processing
  - `handle_bounce_reply_phase()` (line 1595) — Reply bounce processing
  - `handle_transactional_bounce()` (line 1821) — Transactional bounce processing
  - `MailHandler._handle()` (line 2335) — SPF-based 5xx downgrade wrapper
- Current documentation: **Missing** — No dedicated documentation of bounce handling security properties
- Documentation needed: Complete security-focused analysis with code-path tracing

**Module: `app/email_utils.py` (Email Utilities)**
- Functions requiring documentation:
  - `generate_verp_email()` (line 1438) — New signed VERP format generation
  - `get_verp_info_from_email()` (line 1467) — Signed VERP parsing and validation
  - `parse_id_from_bounce()` (line 1258) — Legacy plaintext ID extraction (no validation)
  - `should_ignore_bounce()` (line 1361) — Bounce sender ignore list
  - `should_disable()` (line 1166) — Alias auto-disable based on bounce thresholds
  - `get_orig_message_from_bounce()` (line 683) — Original message extraction from bounce report
  - `get_mailbox_bounce_info()` (line 699) — Bounce metadata extraction
- Current documentation: **Missing** — No security analysis of unsigned vs. signed parsing
- Documentation needed: Comparative security analysis of the two format handlers

**Module: `app/config.py` (Configuration)**
- Configuration options requiring documentation:
  - `BOUNCE_PREFIX` (line 100) — Default: `"bounce+"`
  - `BOUNCE_SUFFIX` (line 101) — Default: `"+@{EMAIL_DOMAIN}"`
  - `BOUNCE_PREFIX_FOR_REPLY_PHASE` (line 108) — Default: `"bounce_reply"`
  - `TRANSACTIONAL_BOUNCE_PREFIX` (line 113) — Default: `"transactional+"`
  - `TRANSACTIONAL_BOUNCE_SUFFIX` (line 117) — Default: `"+@{EMAIL_DOMAIN}"`
  - `VERP_PREFIX` (line 500) — Default: `"sl"`
  - `VERP_EMAIL_SECRET` (line 502) — Min 32 chars, derived from `FLASK_SECRET` as fallback
  - `VERP_MESSAGE_LIFETIME` (line 499) — 5 days (432000 seconds)
- Current documentation: `example.env` documents some env vars but lacks bounce-specific coverage
- Documentation needed: Complete config documentation for bounce/VERP system

**Module: `app/email/status.py` (SMTP Status Codes)**
- Status codes relevant to bounce handling:
  - `E205` — "250 SL E205 bounce handled" (transactional)
  - `E206` — "250 SL E206 Out of office"
  - `E211` — "250 SL E211 Bounce Forward phase handled"
  - `E212` — "250 SL E212 Bounce Reply phase handled"
  - `E213` — "250 SL E213 Unknown email ignored"
  - `E216` — "250 SL E216 Handled spf policy" (5xx downgrade)
  - `E510` — "550 SL E510 so such user" (inactive user)
  - `E512` — "550 SL E512 No such email log" (invalid ID)
- Current documentation: **Missing** — no analysis of information leakage via differential responses
- Documentation needed: Mapping of probe scenarios to response codes with leakage analysis

**Module: `app/models.py` (Data Models)**
- Models requiring documentation: `VerpType` (line 247), `EmailLog` (line 2060), `Bounce` (line 3290), `TransactionalEmail` (line 3300)
- Current documentation: **Missing** — model schemas exist but security implications undocumented
- Documentation needed: How model lookups enable or prevent ID enumeration

**Module: `app/email/rate_limit.py` (Rate Limiting)**
- Current state: Rate limiting is **disabled** (line 97, `return False`)
- Documentation needed: Impact of disabled rate limiting on probing attacks

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **Undocumented security property:** The coexistence of unsigned and signed bounce address formats and the information leakage implications of the legacy format — no documentation anywhere in the repository addresses this.
- **Undocumented SMTP response differential:** The system returns distinct SMTP codes for valid vs. invalid email log IDs when probing old-format bounce addresses, enabling blind enumeration — this is entirely undocumented.
- **Undocumented bounce detection spoofability:** The `is_bounce()` criteria (null sender + multipart/report) are externally satisfiable by any SMTP client — this has no documentation.
- **Undocumented SPF downgrade interaction:** The SPF-based 5xx-to-250 downgrade partially masks the enumeration signal, but only when SPF fails — this interaction is not documented.
- **Undocumented rate limiting gap:** Rate limiting is disabled, removing any throttle on probing — no documentation addresses this implication.
- **No cross-module security flow documentation:** The security-relevant code path spans 6+ modules with no consolidated view of how they interact during a bounce probe scenario.

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output document `blitzy/documentation/app_2cd6ee777f8c.md` will follow this structure:

```
blitzy/documentation/app_2cd6ee777f8c.md
├── # Bounce Email Handling Security Analysis
│   ├── ## Executive Summary
│   ├── ## 1. Dual Bounce Address Format Analysis
│   │   ├── ### 1.1 Legacy Format (Unsigned)
│   │   ├── ### 1.2 New Signed VERP Format
│   │   └── ### 1.3 Comparative Security Assessment
│   ├── ## 2. Bounce Routing Decision Tree
│   │   ├── ### 2.1 VERP Detection Region
│   │   ├── ### 2.2 Parallel Format Check Logic
│   │   ├── ### 2.3 Priority: Signed Over Unsigned
│   │   └── ### 2.4 iCloud Special Case
│   ├── ## 3. SMTP Response Differential Analysis
│   │   ├── ### 3.1 Response Code Inventory
│   │   ├── ### 3.2 Probe Scenario Matrix
│   │   ├── ### 3.3 SPF Downgrade Interaction
│   │   └── ### 3.4 Observable Information Per Scenario
│   ├── ## 4. Bounce Detection Criteria & Spoofability
│   │   ├── ### 4.1 is_bounce() Criteria
│   │   ├── ### 4.2 is_automatic_out_of_office() Criteria
│   │   ├── ### 4.3 External Satisfiability Assessment
│   │   └── ### 4.4 Criteria Pass vs. Fail Behavior
│   ├── ## 5. Information Leakage Surface
│   │   ├── ### 5.1 Email Log ID Enumeration Attack
│   │   ├── ### 5.2 What Response Codes Reveal
│   │   ├── ### 5.3 Rate Limiting Gap
│   │   └── ### 5.4 Timing and Activity Inference
│   ├── ## 6. Security Boundary Map
│   │   ├── ### 6.1 Cryptographically Protected Paths
│   │   ├── ### 6.2 Unprotected Paths
│   │   └── ### 6.3 Boundary Summary Diagram
│   └── ## 7. Source References
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**
- Extract bounce address formats from `app/config.py` (lines 99–118, 498–508) and `app/email_utils.py` (lines 1258–1259, 1438–1498)
- Trace routing logic from `email_handler.py:handle()` (lines 2034–2117) with line-by-line code-path analysis
- Map all SMTP responses from `app/email/status.py` to their triggering conditions in `email_handler.py`
- Analyze `is_bounce()` (line 1813) criteria against RFC 3464 (DSN format) for spoofability assessment
- Document the SPF downgrade from `MailHandler._handle()` (lines 2357–2365)
- Verify rate limiting state from `app/email/rate_limit.py` (line 97)

**Documentation Standards:**
- Every technical claim must include a source citation in the format `Source: /path/to/file.py:LineNumber`
- Code examples kept to 2–3 lines for clarity, referencing the source for full context
- Mermaid diagrams for the routing decision tree and security boundary visualization
- Tables for the SMTP response differential matrix and probe scenario analysis
- Consistent use of `"old format"` / `"legacy format"` for the unsigned pattern and `"new format"` / `"signed VERP"` for the HMAC-authenticated pattern

### 0.4.3 Diagram and Visual Strategy

**Mermaid diagrams to create:**

- **Bounce Routing Decision Flowchart:** Documenting the full decision tree from inbound email reception through the VERP detection region, showing how both old and new formats are checked in parallel and which code paths they activate. Based on `email_handler.py` lines 2034–2117.

- **SMTP Response Differential Matrix Diagram:** A visual representation of the different response codes an external prober would observe based on: address format (old vs. new), email log ID validity, bounce criteria satisfaction, and SPF status.

- **Security Boundary Map:** A diagram showing which components in the bounce processing pipeline are protected by cryptographic validation and which are not, with the boundary clearly delineated between `get_verp_info_from_email()` (protected) and `parse_id_from_bounce()` (unprotected).

- **Bounce Detection Criteria Flow:** A small decision diagram showing the `is_bounce()` and `is_automatic_out_of_office()` checks and their interaction with the bounce handler dispatch.

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | CREATE | `email_handler.py`, `app/email_utils.py`, `app/config.py`, `app/email/status.py`, `app/models.py`, `app/errors.py`, `app/email/rate_limit.py`, `app/handler/spamd_result.py` | Complete security investigation document analyzing bounce email handling information leakage, dual address format comparison, SMTP response differentials, bounce detection criteria spoofability, and security boundary mapping. All findings based on code-path analysis with file:line citations. |

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/app_2cd6ee777f8c.md
Type: Security Investigation / Technical Analysis
Source Code:
  - email_handler.py (primary — bounce routing, handle(), is_bounce(), handle_bounce(), MailHandler._handle())
  - app/email_utils.py (VERP generation/parsing, legacy ID extraction, bounce threshold logic)
  - app/config.py (BOUNCE_PREFIX, BOUNCE_SUFFIX, VERP_PREFIX, VERP_EMAIL_SECRET, VERP_MESSAGE_LIFETIME)
  - app/email/status.py (all E-codes returned during bounce processing)
  - app/models.py (VerpType enum, EmailLog model, Bounce model, TransactionalEmail model)
  - app/errors.py (VERPTransactional, VERPForward, VERPReply exception classes)
  - app/email/rate_limit.py (rate_limited() — currently disabled)
  - app/handler/spamd_result.py (SpamdResult, SPFCheckResult — used in 5xx downgrade logic)
Sections:
  - Executive Summary (overview of findings)
  - Dual Bounce Address Format Analysis (old vs. new format generation and parsing)
  - Bounce Routing Decision Tree (handle() VERP detection region walkthrough)
  - SMTP Response Differential Analysis (probe scenario → response code mapping)
  - Bounce Detection Criteria & Spoofability (is_bounce() analysis)
  - Information Leakage Surface (enumeration attack, response code semantics, rate limit gap)
  - Security Boundary Map (cryptographic vs. unprotected paths)
  - Source References (all files examined with line ranges)
Diagrams:
  - Bounce routing decision flowchart (Mermaid)
  - Security boundary map diagram (Mermaid)
  - SMTP response differential matrix (table)
  - Bounce detection criteria flow (Mermaid)
Key Citations:
  - email_handler.py:1813-1818 (is_bounce)
  - email_handler.py:1258-1259 (parse_id_from_bounce)
  - email_handler.py:2034-2117 (VERP routing region)
  - email_handler.py:2357-2365 (SPF downgrade)
  - app/email_utils.py:1438-1464 (generate_verp_email)
  - app/email_utils.py:1467-1498 (get_verp_info_from_email)
  - app/config.py:99-118 (bounce config constants)
  - app/config.py:498-508 (VERP config)
  - app/email/status.py:1-65 (all status codes)
  - app/email/rate_limit.py:95-97 (disabled rate limiting)
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration updates are needed. The project does not use a documentation generator or build system. The output document is a standalone Markdown file placed directly in `blitzy/documentation/`.

### 0.5.4 Cross-Documentation Dependencies

- **No shared content or includes:** The output document is self-contained.
- **No navigation links required:** The `blitzy/documentation/` directory is an output-only location.
- **No table of contents updates:** No documentation index or site generator exists.
- **Internal cross-references:** The document will internally link between its own sections (e.g., the Routing section referencing the Response Differential section).

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

This documentation task does not require any additional documentation tooling installation. The output is a standalone Markdown file authored directly. No documentation generators, diagram rendering tools, or API documentation extractors are needed at build time.

The following project dependencies are referenced in the analysis but are not documentation tools — they are the application dependencies whose behavior is being documented:

| Registry | Package Name | Version | Purpose in Analysis |
|----------|--------------|---------|---------------------|
| pip | flask | ^1.1.2 | Web framework providing the application context used in `MailHandler._handle()` |
| pip | aiosmtpd | (from pyproject.toml) | SMTP server framework; `MailHandler`, `Controller`, and `Envelope` used in bounce handling |
| pip | sqlalchemy | (from pyproject.toml) | ORM for `EmailLog.get()`, `Bounce.create()`, `TransactionalEmail.get()` lookups |
| pip | arrow | ^0.16.0 | Time calculations in `should_disable()` bounce threshold logic |
| pip | newrelic | (from pyproject.toml) | `@background_task()` decorator on `_handle()`, SPF downgrade metric recording |
| stdlib | hmac | Python 3.10 | HMAC computation in `generate_verp_email()` and `get_verp_info_from_email()` |
| stdlib | base64 | Python 3.10 | Base32 encoding/decoding in VERP address format |
| stdlib | json | Python 3.10 | JSON payload serialization in VERP generation |
| stdlib | hashlib (sha3_224) | Python 3.10 | `VERP_HMAC_ALGO = "sha3-224"` used for HMAC signing |

### 0.6.2 Documentation Reference Updates

Not applicable. This is a net-new document creation with no existing links to update. The document will be self-contained in `blitzy/documentation/app_2cd6ee777f8c.md` with all references pointing to source code file paths within the repository.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Current coverage analysis:**

| Coverage Area | Documented | Total | Percentage |
|---------------|-----------|-------|-----------|
| Bounce address formats (old + new) | 0 | 2 | 0% |
| Bounce routing code paths in `handle()` | 0 | 4 (transactional, forward, reply, iCloud) | 0% |
| SMTP response codes returned during bounce processing | 0 | 10 (E205, E206, E211, E212, E213, E216, E404, E510, E512, plus VERPForward/Reply/Transactional→E213) | 0% |
| Bounce detection criteria functions | 0 | 2 (`is_bounce`, `is_automatic_out_of_office`) | 0% |
| Security boundary analysis (signed vs. unsigned) | 0 | 1 | 0% |
| Information leakage scenarios | 0 | 5+ (valid ID, invalid ID, non-bounce to VERP, OOO to VERP, SPF downgrade interaction) | 0% |

**Target coverage: 100%** — The output document must address every bounce-related code path, response code, detection criterion, and security boundary identified in the codebase.

**Coverage gaps to address:**
- `email_handler.py` bounce routing region: Currently 0% documented, target 100%
- `app/email_utils.py` VERP functions: Currently 0% documented for security properties, target 100%
- `app/email/status.py` bounce-related codes: Currently 0% documented in security context, target 100%
- `app/email/rate_limit.py` disabled state: Currently 0% documented for security implications, target 100%

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every bounce-related function must have its behavior traced with file:line citations
- Every distinct SMTP response code reachable through bounce probing must be documented with its triggering condition
- The probe scenario matrix must cover: valid ID / invalid ID × bounce format / non-bounce format × old address / new address × SPF pass / SPF fail
- Both the forward and reply bounce paths must be fully traced
- The iCloud special case (lines 2100–2116) must be documented

**Accuracy validation:**
- All code-path claims must reference specific line numbers verified through `read_file` tool inspection
- SMTP response codes must exactly match `app/email/status.py` definitions
- Configuration defaults must match values in `app/config.py`
- The HMAC algorithm, key length, and encoding details must match `app/email_utils.py` implementation

**Clarity standards:**
- Technical accuracy with precise code citations — no vague references
- Progressive disclosure: executive summary → detailed analysis → source references
- Each finding must state: what happens, why it happens (code path), and what an attacker learns
- Consistent terminology: "legacy format" / "signed VERP format" used throughout

**Maintainability:**
- Source citations (file:line) allow future verification against code changes
- Modular section structure allows individual sections to be updated independently
- Mermaid diagrams are embedded and renderable in any Markdown viewer

### 0.7.3 Example and Diagram Requirements

- **Minimum examples per finding:** At least one concrete address example for each format (e.g., `bounce+12345+@sl.local` vs. `sl.giytemjugu3c4mzr.mfqxa5bofvvhk4tvnfuca.@sl.local`)
- **Diagram types required:**
  - Flowchart: Bounce routing decision tree
  - Flowchart: Security boundary map
  - Table: SMTP response differential matrix (not a diagram, but serves the same visual purpose)
- **Code example verification:** All code snippets are direct quotations from the source with line numbers — they are inherently verified
- **Visual content freshness:** Diagrams reflect the current state of code as of the analysis date; line numbers may shift with future commits

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation file:**
- `blitzy/documentation/app_2cd6ee777f8c.md` — Complete security investigation document

**Source code modules analyzed (read-only, for documentation purposes):**
- `email_handler.py` — Central SMTP handler: bounce routing (lines 2034–2117), `handle_bounce()` (line 1851), `handle_bounce_forward_phase()` (line 1432), `handle_bounce_reply_phase()` (line 1595), `handle_transactional_bounce()` (line 1821), `is_bounce()` (line 1813), `is_automatic_out_of_office()` (line 1793), `MailHandler._handle()` SPF downgrade (lines 2357–2365), `handle()` main routing (line 1945)
- `app/email_utils.py` — VERP generation (`generate_verp_email()` line 1438), VERP validation (`get_verp_info_from_email()` line 1467), legacy ID extraction (`parse_id_from_bounce()` line 1258), bounce threshold (`should_disable()` line 1166), `should_ignore_bounce()` (line 1361), `get_orig_message_from_bounce()` (line 683), `get_mailbox_bounce_info()` (line 699)
- `app/config.py` — Bounce configuration constants (lines 99–118), VERP configuration (lines 498–508)
- `app/email/status.py` — All SMTP status codes (lines 1–65)
- `app/models.py` — `VerpType` (line 247), `EmailLog` (line 2060), `Bounce` (line 3290), `TransactionalEmail` (line 3300)
- `app/errors.py` — `VERPTransactional` (line 42), `VERPForward` (line 48), `VERPReply` (line 54)
- `app/email/rate_limit.py` — `rate_limited()` disabled state (line 97)
- `app/handler/spamd_result.py` — `SpamdResult`, `SPFCheckResult` used in 5xx downgrade logic
- `app/email/headers.py` — Header constants used in bounce processing

**Existing test files analyzed (read-only, for documentation context):**
- `tests/test_email_utils.py` — Tests for `parse_id_from_bounce`, `generate_verp_email`, `get_verp_info_from_email`, `should_ignore_bounce`, `get_orig_message_from_bounce`, `get_mailbox_bounce_info`
- `tests/test_email_handler.py` — Tests for SPF downgrade, VERP bounce handling

**Existing documentation files analyzed (read-only):**
- `SECURITY.md` — Vulnerability disclosure policy
- `README.md` — Project overview and self-hosting guide
- `docs/troubleshooting.md` — Operational diagnostics
- `docs/enforce-spf.md` — SPF enforcement guide

### 0.8.2 Explicitly Out of Scope

- **Source code modifications:** No files in the repository will be modified. The user explicitly prohibits this.
- **Test file modifications:** No existing test files will be modified.
- **Feature additions or code refactoring:** This is a documentation-only task. No patches, fixes, or mitigations will be implemented.
- **Deployment configuration changes:** No Docker, Postfix, or infrastructure changes.
- **Documentation unrelated to bounce handling:** The `docs/api.md`, `docs/oauth.md`, `docs/ssl.md`, etc. will not be updated.
- **Forward-phase and reply-phase email processing:** Only the bounce-handling aspects of forward/reply are in scope — the full forward and reply email flows are out of scope.
- **PGP encryption, DKIM signing, and spam detection:** These systems are not part of the bounce security investigation unless they directly interact with bounce routing.
- **User interface or dashboard documentation:** No web UI aspects are in scope.
- **Database migration or schema documentation:** Data models are referenced for context only.

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** Not applicable — standalone Markdown file, no build system
- **Documentation preview command:** Any Markdown renderer (e.g., `grip blitzy/documentation/app_2cd6ee777f8c.md` or GitHub/GitLab preview)
- **Diagram generation command:** Not applicable — Mermaid diagrams are embedded inline and rendered by Markdown viewers with Mermaid support
- **Documentation deployment command:** Not applicable — document is committed directly to the repository
- **Default format:** Markdown with embedded Mermaid diagrams
- **Citation requirement:** Every technical claim must reference source files with line numbers in the format `Source: path/to/file.py:LineNumber` or `Source: path/to/file.py:StartLine-EndLine`
- **Style guide:** Evidence-based technical analysis style — code-path tracing with citations, no theoretical speculation; findings stated as "the code does X" not "the code should do X"
- **Documentation validation:** Manual review of all file:line citations against the current codebase; verify all SMTP status code strings match `app/email/status.py`; confirm all configuration defaults match `app/config.py`

### 0.9.2 Output File Specification

- **File path:** `blitzy/documentation/app_2cd6ee777f8c.md`
- **File name derivation:** Source branch name `app_2cd6ee777f8c` + `.md` extension, per SWE-AtlasQnA-Repo implementation rule
- **Output directory:** `blitzy/documentation/` — per implementation rule specification
- **Encoding:** UTF-8
- **Maximum length:** No artificial limit; completeness takes priority over brevity

## 0.10 Rules for Documentation

The following rules are explicitly mandated by the user's instructions and the implementation rule configuration:

- **Do not modify any existing files in the source repository.** The investigation is read-only. All output goes exclusively to `blitzy/documentation/app_2cd6ee777f8c.md`.
- **Base all answers on the code as the truth, do not make assumptions.** Every claim in the document must be verifiable by reading the cited source code. No speculation about intended behavior — only document what the code actually does.
- **Provide thinking and rationale behind the answers.** The document must not just state findings but explain the reasoning — why a particular code path leads to a particular response, what the implications are, and how the conclusion was derived.
- **Place the generated document in the `blitzy/documentation` directory.** The file must exist at `blitzy/documentation/app_2cd6ee777f8c.md`.
- **Test scripts may be created to observe behavior but must be cleaned up when finished.** If any test scripts are created during the investigation, they must be removed before completion. In practice, the investigation is conducted through static code analysis rather than runtime testing, as the system requires a full database and SMTP stack.
- **Document actual evidence, not theoretical explanations.** The user explicitly states: "I don't want theoretical explanations of what the code should do, I want to see actual evidence of how the system behaves in practice." All code-path analysis serves as the "actual evidence" in lieu of live runtime testing.
- **Address all user questions comprehensively.** The document must cover: dual format comparison, bounce routing mechanics, SMTP response differentials for various probing scenarios, bounce detection criteria and spoofability, information revealed by response codes, and the exact location of security boundaries.

## 0.11 References

### 0.11.1 Files and Folders Searched Across the Codebase

**Primary Source Files (read in full or critical sections):**

| File Path | Lines Read | Purpose in Analysis |
|-----------|-----------|---------------------|
| `email_handler.py` | 1–2405 (full file) | Central SMTP handler — bounce routing, VERP detection, `is_bounce()`, `handle_bounce()`, `handle_bounce_forward_phase()`, `handle_bounce_reply_phase()`, `handle_transactional_bounce()`, SPF downgrade in `_handle()` |
| `app/email_utils.py` | 55–75, 683–770, 1156–1260, 1361–1367, 1430–1506 | VERP generation/parsing (`generate_verp_email`, `get_verp_info_from_email`), legacy ID extraction (`parse_id_from_bounce`), bounce utilities (`should_disable`, `should_ignore_bounce`, `get_orig_message_from_bounce`, `get_mailbox_bounce_info`) |
| `app/config.py` | 90–118, 490–520 | Bounce configuration (`BOUNCE_PREFIX`, `BOUNCE_SUFFIX`, `BOUNCE_PREFIX_FOR_REPLY_PHASE`, `TRANSACTIONAL_BOUNCE_PREFIX/SUFFIX`), VERP configuration (`VERP_PREFIX`, `VERP_EMAIL_SECRET`, `VERP_MESSAGE_LIFETIME`) |
| `app/email/status.py` | 1–65 (full file) | All SMTP status codes: E200–E525 |
| `app/models.py` | 247–250, 2060–2150, 3290–3330 | `VerpType` enum, `EmailLog` model, `Bounce` model, `TransactionalEmail` model |
| `app/errors.py` | 1–131 (full file) | `VERPTransactional`, `VERPForward`, `VERPReply` exception classes |
| `app/email/rate_limit.py` | 1–110 (full file) | `rate_limited()` function — confirmed disabled at line 97 |
| `app/email/headers.py` | 1–60 (full file) | Header constants: `AUTO_SUBMITTED`, `SPAMD_RESULT`, MIME headers |

**Test Files Examined:**

| File Path | Lines Read | Purpose in Analysis |
|-----------|-----------|---------------------|
| `tests/test_email_utils.py` | 789–870 | Tests for `parse_id_from_bounce`, `generate_verp_email`, `get_verp_info_from_email`, `should_ignore_bounce`, `get_orig_message_from_bounce`, `get_mailbox_bounce_info` |
| `tests/test_email_handler.py` | 1–414 (full file) | Tests for SPF downgrade (5xx→E216), VERP bounce handling, DMARC processing |
| `tests/test.env` | Key variables | Test environment configuration: `EMAIL_DOMAIN=sl.local`, `FLASK_SECRET=secret` |

**Documentation Files Examined:**

| File Path | Purpose |
|-----------|---------|
| `README.md` | Project overview, self-hosting instructions — no bounce documentation |
| `SECURITY.md` | Vulnerability disclosure policy |
| `CONTRIBUTING.md` | Contributor workflow guidelines |
| `docs/troubleshooting.md` | Operational diagnostics (swaks, telnet, docker logs) |
| `docs/enforce-spf.md` | SPF enforcement guide |
| `docs/api.md` | REST API reference |

**Folders Explored:**

| Folder Path | Depth | Purpose |
|-------------|-------|---------|
| (root) | Level 0 | Repository structure discovery |
| `app/` | Level 1 | Application package structure, core modules |
| `app/handler/` | Level 2 | Handler subpackage: DMARC, provider complaints, unsubscribe, spamd |
| `app/email/` | Level 2 | Email subpackage: status codes, rate limiting, headers, spam |
| `tests/` | Level 1 | Test suite structure |
| `tests/handler/` | Level 2 | Handler-specific tests |
| `docs/` | Level 1 | Existing documentation inventory |
| `blitzy/documentation/` | Level 1 | Output directory verification |

**Configuration Files Examined:**

| File Path | Purpose |
|-----------|---------|
| `pyproject.toml` | Poetry dependencies, Python version (^3.10) |
| `example.env` | Environment variable documentation |
| `tests/test.env` | Test environment defaults |

### 0.11.2 Attachments

No attachments were provided by the user for this project.

### 0.11.3 Figma Screens

No Figma URLs or design screens were provided for this project.

### 0.11.4 Tech Spec Sections Referenced

| Section | Content Used |
|---------|-------------|
| 4.2 Core Email Processing Workflows | Bounce routing flowcharts, forward/reply phase flows, VERP tracking documentation |
| 4.3 Email Security Processing | DMARC enforcement logic, spam detection thresholds, SPF downgrade mechanism |
| 6.4 Security Architecture | VERP token architecture, key management (VERP_EMAIL_SECRET), security control matrix, data retention policies for bounce records |

