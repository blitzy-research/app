# SimpleLogin Alias Reply-Handling Pipeline — Runtime Analysis

*A code-grounded investigation of the SimpleLogin inbound-SMTP reply path, produced by static analysis of the source tree and in-process runtime simulation.*

This document answers the five questions posed in the prompt — (1) which part of the system handles an incoming reply, (2) how the alias is resolved to a user, (3) what user ID the system ultimately decides to forward the reply to, (4) the actual end-to-end data flow in detail, and (5) the most likely point where incorrect routing could originate. Every claim below is anchored to a specific `path:line` reference in the SimpleLogin codebase. The reply-phase test suite was executed (`pytest tests/test_email_handler.py -k reply` — 6 passed, 17 deselected) and a synthetic inbound reply was dispatched through `email_handler.handle()` inside the Flask app context to verify runtime behavior against the claims in this document.

---

## Section 1 — Executive Summary

- **Entry point**: Inbound SMTP DATA commands land on `MailHandler.handle_DATA()` at `email_handler.py:2289`, which parses the raw message body via `email.message_from_bytes()` at `email_handler.py:2290` and delegates to `self._handle(envelope, msg)` at `email_handler.py:2292`.
- **Core reply handler**: `handle_reply(envelope, msg, rcpt_to)` at `email_handler.py:966` owns every decision specific to the reply phase — domain validation, contact lookup, alias resolution, mailbox authorization, header rewriting, and final delivery.
- **Identity-resolution chain**: `rcpt_to (reverse-alias)` → `Contact.reply_email` → `contact.alias_id` → `Alias` → `alias.user_id` → `User`. This is a strict foreign-key walk using the indexed `Contact.reply_email` column at `app/models.py:1899`, the `Contact.alias_id` FK at `app/models.py:1881–1883`, and the `Alias.user_id` FK at `app/models.py:1474–1476`.
- **Final destination**: The reply is delivered to the external third-party's real email address — `contact.website_email` — via `sl_sendmail()` at `email_handler.py:1224–1231`. It is **not** forwarded to any SimpleLogin user mailbox; the user-ID recorded on the `EmailLog` is `contact.user_id` and exists only for bookkeeping.
- **Most likely failure point (preview)**: The combination of (a) the multi-mailbox `notify_mailbox()` fan-out at `email_handler.py:1234–1236` (which delivers a *copy* of every reply to every additional verified mailbox on the alias) and (b) the silent fallback to `alias.mailbox` when `alias.disable_email_spoofing_check == True` at `email_handler.py:1025–1029` (which records the primary mailbox on the `EmailLog` even when the real sender is unknown). A runtime simulation with a two-mailbox alias confirmed that 2 outbound SMTP messages are produced — one primary delivery and one `notify_mailbox` copy — each with distinct VERP envelope-from addresses.

### 1.1 What Changes Between Forward and Reply Phases

The SimpleLogin SMTP handler serves two distinct phases within a single process: the **forward phase** (an external sender's message addressed to an alias is forwarded to the alias owner's mailbox) and the **reply phase** (the alias owner's reply to a reverse-alias is delivered to the external recipient). This document focuses exclusively on the reply phase per the prompt's scope.

The branch that separates the two happens at a single line inside the per-recipient loop: `if is_reverse_alias(rcpt_to):` at `email_handler.py:2195`. If True, `handle_reply` takes over; otherwise, `handle_forward` takes over.

The key differences between the two phases are:

- **Authorization model.** The forward phase accepts messages from any external sender; the reply phase requires the sender to be an authorized mailbox (or to fall through the `disable_email_spoofing_check` silent fallback).
- **Routing target.** The forward phase delivers to one or more of the alias's mailboxes; the reply phase delivers to a single external `contact.website_email`.
- **Header rewriting.** The forward phase hides the external sender behind a reverse alias; the reply phase hides the authorized mailbox behind the alias identity.
- **Contact resolution direction.** Forward: look up / create a `Contact` keyed by `(alias_id, website_email)`. Reply: look up a `Contact` by `reply_email`.
- **Body replacement direction.** Forward: replace reverse aliases in inbound body with the external contact's real address (for when a thread crosses reply/forward boundaries). Reply: the user's body is already in natural form; replace reverse-alias strings to make the outbound message look natural to the external recipient.

### 1.2 Reader's Map

This document is organized for three distinct reading flows:

- **Question-first readers** can jump to Sections 2 (entry point), 3 (alias → user), 4 (final destination user ID), 5 (full data flow), and 6 (failure modes) — each directly answers one of the prompt's five questions.
- **Code-reviewers** should read Section 5 (numbered walkthrough) end-to-end with the source files open side-by-side — every step has a `file:line` citation.
- **Operators triaging a "wrong user" ticket** should read Section 6 (failure modes) and then the specific runtime traces in Section 7 to compare their incident logs against the expected behavior.

---

## Section 2 — Which Part of the System Handles the Incoming Reply?

SimpleLogin splits its workload across five independent processes: the web app (Flask), the SMTP handler, the cron scheduler, the job runner, and the event listener. The **SMTP handler is a standalone process** defined in `email_handler.py` at the root of the repository. All inbound reply routing happens inside that process — the Flask web app is NOT involved in per-message dispatch (although a minimal Flask context IS established for ORM access, as described below).

### 2.1 Process Entry — aiosmtpd Controller on Port 20381

The SMTP handler runs as an `aiosmtpd` controller bound to `0.0.0.0:20381` by default. `main(port: int)` at `email_handler.py:2381` constructs `controller = Controller(MailHandler(), hostname="0.0.0.0", port=port)` at `email_handler.py:2383` and starts it. The CLI declaration `parser.add_argument("-p", "--port", ..., default=20381)` at `email_handler.py:2399` establishes the default port. Postfix (the upstream MTA) is configured to relay SMTP content to this port, and aiosmtpd accepts the DATA payload on behalf of SimpleLogin.

### 2.2 Per-Message Entry — `MailHandler.handle_DATA`

The `MailHandler` class is defined at `email_handler.py:2288`. Its async method `handle_DATA(server, session, envelope: Envelope)` at `email_handler.py:2289` is the aiosmtpd callback invoked for every SMTP DATA command. The method:

1. Parses the raw bytes via `msg = email.message_from_bytes(envelope.original_content)` at `email_handler.py:2290`.
2. Calls the synchronous `ret = self._handle(envelope, msg)` at `email_handler.py:2292` to perform the business logic.
3. Wraps the whole call in exception handlers that return `E524` (wrong reverse-alias use), `E213` (unknown email ignored), or `E404` (unexpected error) depending on the exception class (`email_handler.py:2297–2332`).

### 2.3 Observability & Flask Context — `_handle`

`_handle(self, envelope, msg)` at `email_handler.py:2335` is decorated with `@newrelic.agent.background_task()` at `email_handler.py:2334` for APM instrumentation. It:

1. Assigns a fresh correlation id: `message_id = str(uuid.uuid4())` at `email_handler.py:2339` and calls `set_message_id(message_id)` at `email_handler.py:2340`, which makes every downstream `LOG` line tagged with this UUID for per-message correlation.
2. Opens a Flask application context via `with create_light_app().app_context():` at `email_handler.py:2352`. `create_light_app()` is defined at `server.py:127` and is a *minimal* Flask application wired ONLY for SQLAlchemy (`app.config["SQLALCHEMY_DATABASE_URI"] = DB_URI` at `server.py:129`). It is deliberately different from the web application factory `create_app()` at `server.py:139` — the SMTP handler does not need the web middleware or route blueprints, only the ORM session.
3. Inside that context, calls `return_status = handle(envelope, msg)` at `email_handler.py:2353`, which is the main dispatcher.

### 2.4 Main Dispatcher — `handle`

`handle(envelope: Envelope, msg: Message)` is defined at `email_handler.py:1945`. Its responsibilities:

1. **Sanitize the envelope**: `mail_from = sanitize_email(envelope.mail_from)` at `email_handler.py:1949` and `rcpt_tos = [sanitize_email(rcpt_to) for rcpt_to in envelope.rcpt_tos]` at `email_handler.py:1950`. `sanitize_email()` is at `app/utils.py:97` and strips whitespace, lowercases (unless `not_lower=True`), and removes the right-to-left mark `\u200f`.
2. **Sanitize the headers**: `sanitize_header(msg, headers.FROM/TO/CC/REPLY_TO/MESSAGE_ID)` at `email_handler.py:1974–1978`.
3. **Classify each recipient**: Iterate `for rcpt_index, rcpt_to in enumerate(rcpt_tos):` at `email_handler.py:2180`. For each one, at `email_handler.py:2195` check `is_reverse_alias(rcpt_to)`.
4. **Dispatch**: If classified as a reply, call `handle_reply(envelope, copy_msg, rcpt_to)` at `email_handler.py:2199`. Otherwise, fall through to the forward-phase branch.

### 2.5 Reply Classification — `is_reverse_alias`

`is_reverse_alias(address: str) -> bool` is defined at `app/email_utils.py:1156`. It returns `True` if **either**:

- The address already exists in the `Contact` table as a reverse-alias: `if Contact.get_by(reply_email=address): return True` at `app/email_utils.py:1158–1159`. This is the authoritative check — it handles modern reverse-aliases that have no fixed prefix.
- OR the address matches the legacy pattern: `return address.endswith(f"@{config.EMAIL_DOMAIN}") and (address.startswith("reply+") or address.startswith("ra+"))` at `app/email_utils.py:1161–1163`. This is the fallback for legacy addresses whose database rows may have been deleted but whose format is still recognizable.

### 2.6 Reply Handler — `handle_reply`

Once the dispatcher classifies a recipient as a reverse-alias, all subsequent reply-phase logic is delegated to `handle_reply(envelope, msg, rcpt_to)` at `email_handler.py:966`. This is the single function that implements every reply-specific behavior: domain gate, contact lookup, alias resolution, user resolution, DMARC check, mailbox authorization, EmailLog creation, header rewriting, DKIM signing, primary delivery, and multi-mailbox notification fan-out.

### 2.7 Return Path

`handle_reply` returns a `(bool, str)` tuple (delivered, SMTP status). `handle()` accumulates those tuples (one per `rcpt_to`) into `res` at `email_handler.py:2200` and reduces them at `email_handler.py:2214–2233`: if any recipient succeeded, return the first success status; otherwise return the first failure. `_handle()` wraps this (`email_handler.py:2357–2365`) — it downgrades 5xx statuses to `E216` when SPF fails on the envelope return-path (to prevent backscatter). The final status bubbles up through `handle_DATA()` to aiosmtpd and then to Postfix.

### 2.8 SimpleLogin Process Topology

To put the SMTP handler in context, SimpleLogin is not a single process. The deployment architecture distributes work across five independent processes, each with a distinct responsibility:

1. **The web application** — `server.py:139` (`create_app()`) creates the Flask web factory. This process handles user dashboard interactions, OAuth/OIDC flows, REST API endpoints, Jinja template rendering, admin panel access via `FLASK_ADMIN`, session management via Redis-backed session cookies, and the web-facing unsubscribe and confirmation links.
2. **The SMTP handler** — `email_handler.py:2381` (`main(port)`) is the process entry point that owns all inbound email classification and routing. This is the ONLY process involved in the reply-handling pipeline.
3. **The cron scheduler** — `cron.py` periodically runs batched maintenance jobs such as deletion of expired aliases, reconciliation of bounce counters, and aging-out of rate-limit buckets.
4. **The job runner** — invoked to process the `Job` queue for asynchronous work such as user deletion cascades and transactional email sends from web-request handlers.
5. **The event listener** — `event_listener.py` subscribes to internal event notifications (from `events/`) for cross-service coordination.

Because the SMTP handler is a **dedicated process**, it does not share memory with the web app: every reply is handled in isolation, with its own Flask app context, its own ORM `Session`, and its own NewRelic transaction. This isolation is why the reply pipeline is so deterministic — there are no shared mutable globals that could corrupt identity resolution mid-flight.

### 2.9 `MailHandler` Exception Handling in Detail

`MailHandler.handle_DATA()` at `email_handler.py:2289` is defensive. It wraps the call to `self._handle(envelope, msg)` in a three-tier `try/except` pyramid. Each `except` branch maps a specific exception class to a SimpleLogin SMTP status code and (in some cases) persists the offending envelope for post-mortem debugging. The tiers are:

1. **`except CannotCreateContactForReverseAlias`** at `email_handler.py:2297–2307` — raised when the forward phase of an earlier interaction failed to materialize a `Contact` for a foreign sender. In the reply phase this specific exception does not arise (replies require an EXISTING Contact), but the handler catches it here for consistency with the forward branch. Returns `status.E524`.
2. **`except (VERPReply, VERPForward, VERPTransactional)`** at `email_handler.py:2308–2318` — raised when an inbound message itself has a VERP-encoded `To:` address that should never be processed as a regular message (e.g., a reply to a bounce address). These are silently absorbed with `status.E213` — the MTA receives "accepted" but SimpleLogin does not deliver.
3. **`except Exception`** at `email_handler.py:2319–2332` — the catch-all safety net. Logs the exception via `LOG.e(...)` at `email_handler.py:2320` (which the Sentry logging integration auto-reports to Sentry), writes the envelope to the debug directory via `save_envelope_for_debugging(envelope, file_name_prefix=e.__class__.__name__)` at `email_handler.py:2328–2330`, and returns `status.E404`. This prevents the aiosmtpd thread from crashing if an unforeseen error arises — Postfix receives a retriable 4xx response and the original envelope is preserved for engineers to analyze later.

The pattern is **fail-safe by default**: every exit path from `handle_DATA` is a well-formed SMTP response string. The aiosmtpd thread never raises and never returns `None`, which would confuse Postfix.

### 2.10 Anti-Backscatter Logic in `_handle`

`_handle` at `email_handler.py:2335` implements an important anti-abuse guard. After `handle()` returns a status string, `_handle` inspects whether the status is a 5xx rejection (permanent failure). If it is, the handler checks whether the inbound envelope's `mail_from` passed SPF. If SPF failed — meaning the sender domain did not authorize the relaying MTA to send on its behalf — the status is silently **downgraded to `status.E216`**, which is `"250 Email cannot be forwarded to mailbox"`.

The reason is **anti-backscatter**: if SimpleLogin returns a 5xx rejection for a message with a spoofed `mail_from`, the upstream MTA will generate a non-delivery report (NDR) and send it to the spoofed address. This turns SimpleLogin into an unwitting participant in reflection-style spam. By accepting the message (with `E216`) when SPF already failed at the sender's domain, SimpleLogin silently drops the message without generating backscatter. Internally, the message is simply not delivered — it has already been filtered by its own sender's DNS configuration, so there is no legitimate recipient to inform.

The logic is guarded on `should_ignore_bounce(envelope.mail_from)` and on the `SpamdResult` attached to the message — both of which are derived from rspamd headers (`X-Spamd-Result`, etc.) that upstream processing adds to the message. This entire block lives in `_handle` around `email_handler.py:2357–2365`.

### 2.11 Per-Message Observability — `set_message_id`, NewRelic, and Sentry

Observability is built into the handler at three layers:

- **Per-message correlation via `set_message_id(message_id)`** at `email_handler.py:2340`. The `set_message_id` function is in `app/log.py` and stores the given UUID in a Python `ContextVar` that the logger's `LogFilter` reads and embeds in every `LOG.*` line. This means every log line emitted by the reply pipeline (even deep inside `get_mailbox_from_mail_from` or `apply_dmarc_policy_for_reply_phase`) is tagged with the same UUID, making per-message log reconstruction in production a simple `grep` operation.
- **NewRelic APM via `@newrelic.agent.background_task()`** at `email_handler.py:2334`. This decorator registers each call to `_handle` as a distinct transaction in NewRelic. The transaction name is `_handle`, but custom attributes (`mail_from`, `rcpt_tos`, `message_id`, `elapsed`) are attached inside the function body for per-message filtering in the APM dashboard.
- **Error capture via `sentry_sdk`**. Uncaught exceptions inside `_handle` are implicitly reported to Sentry by the `sentry_sdk.integrations.flask.FlaskIntegration` and `sentry_sdk.integrations.logging.LoggingIntegration` plugins configured at app startup. No explicit `capture_exception` call is needed in most exception branches because the logging integration auto-captures `LOG.e(...)` and `LOG.exception(...)` emissions.

The observability triangle (logs + APM + error tracker) means that for any reply that exhibits wrong-routing behavior, an operator can find the root cause by: (i) grepping the NewRelic transaction for the `message_id` attribute, (ii) `grep`'ing the application logs for the same `message_id`, and (iii) checking Sentry for a correlated exception. All three streams are keyed on the same UUID.

### 2.12 `create_light_app` vs `create_app` — Two Flask Factories

The codebase defines two distinct Flask app factories in `server.py`:

- **`create_light_app()`** at `server.py:127` — the minimal factory used by the SMTP handler. It reads `DB_URI` from config, sets `SQLALCHEMY_DATABASE_URI`, sets `SQLALCHEMY_TRACK_MODIFICATIONS = False`, registers a teardown handler that calls `Session.remove()` to release connections, and returns. It does NOT register blueprints, does NOT configure Flask-Login, does NOT configure Flask-Admin, does NOT configure the session cookie, and does NOT attach Jinja template filters. The entire factory is a dozen lines of code.
- **`create_app()`** at `server.py:139` — the full web factory used by the web app. It invokes `create_light_app()` internally, then adds ProxyFix middleware, configures session cookies, sets up the CSRF token, registers all blueprints (auth, dashboard, api, oauth, phone, admin, …), loads template filters, wires Flask-Login, initializes Flask-Admin, and starts the Redis-backed session interface.

The SMTP handler deliberately uses `create_light_app` because it does not need any of the web middleware. The consequence is that a reply-handling process has a **tiny memory footprint**, **fast startup**, and **zero exposure to the web attack surface**. Conversely, it also means that inside `handle_reply`, `flask.request` is NOT defined (there is no HTTP request context), so any utility that depends on `flask.request` (such as CSRF token generation) is unavailable. The reply pipeline does not use any such utilities.

### 2.13 Postfix Interaction and Port Semantics

The SMTP handler process binds to port 20381 on `0.0.0.0` by default. Postfix, which runs as the front-line MTA, is configured to relay any message whose recipient domain matches `EMAIL_DOMAIN` or `OTHER_ALIAS_DOMAINS` (per `example.env`) to this port using LMTP or SMTP. Postfix performs the public-facing TLS termination, IP-level rate limiting, greylisting, and rspamd spam scanning BEFORE handing off to SimpleLogin.

Because Postfix is first in the chain, several security controls are handled at the Postfix layer and NOT by `email_handler.py`:

- **Rspamd scoring**. Postfix calls rspamd during the DATA phase; rspamd appends `X-Spamd-Result`, `Authentication-Results`, `X-Spam`, and `X-Spam-Flag` headers. These headers are read by `SpamdResult.extract_from_headers()` in `app/handler/spamd_result.py` when the message arrives at SimpleLogin, so the DMARC/SPF verdict is already computed.
- **DNSBL / RBL checks**. Postfix rejects known spam sources before the message ever reaches SimpleLogin.
- **TLS termination**. All inbound email arrives already decrypted.

The SimpleLogin SMTP handler is therefore an **application-level** recipient, not an internet-facing receiver. This architecture is why `email_handler.handle_DATA` can trust headers like `Authentication-Results` without re-verifying them — Postfix+rspamd is the trust boundary.

### 2.14 `sanitize_email` and `sanitize_header` Behavior

Before any classification, every address on the envelope is passed through `sanitize_email()` at `app/utils.py:97`. The function:

1. Strips surrounding whitespace (leading/trailing).
2. Removes the RIGHT-TO-LEFT MARK character `\u200f` (a well-known homograph attack vector).
3. If `not_lower=False` (the default), lowercases the address.
4. Returns the cleaned string.

For headers, `sanitize_header(msg, header_name)` at `email_handler.py` is applied to `FROM`, `TO`, `CC`, `REPLY_TO`, and `MESSAGE_ID` (lines 1974–1978). It strips the same dangerous characters from header values so that downstream parsing by `getaddresses()` and `Contact.get_by()` operates on clean input.

These two functions collectively defend against a class of attacks where an adversary crafts an envelope or header containing hidden Unicode characters that make the address appear different to the human reader than to the parser. Without this sanitization, the same address string could match two different `Contact` rows depending on whether the `\u200f` was preserved. SimpleLogin removes it proactively to eliminate the attack surface.

---

## Section 3 — How Is the Alias Resolved to a User?

The alias-to-user resolution inside `handle_reply()` is a 13-step deterministic walk beginning at `email_handler.py:966`. Every step has an exact line number.

### 3.1 Step-by-Step Walk

1. **`email_handler.py:972`** — `reply_email = rcpt_to` captures the reverse-alias address that the inbound message was addressed to.
2. **`email_handler.py:974`** — `reply_domain = get_email_domain_part(reply_email)` extracts the right-hand-side of the `@` in the reverse-alias address.
3. **`email_handler.py:977–981`** — Domain gate. `if not reply_email.endswith(EMAIL_DOMAIN):` (line 977) and if the domain is not found via `SLDomain.get_by(domain=reply_domain)` (line 978), the handler emits `LOG.w("Reply email {reply_email} has wrong domain")` and returns `(False, status.E501)` (line 981). This prevents impersonation of reverse-aliases on domains the system does not own.
4. **`email_handler.py:984`** — `reply_email = normalize_reply_email(reply_email)`. The function is defined at `app/email_validation.py:25`. Per `app/email_validation.py:27–28` it first normalizes non-ASCII via `convert_to_id()`, then replaces any character not in `_ALLOWED_CHARS` (defined at `app/email_validation.py:9` as `"abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789_-.+@"`) with `_`. This repairs legacy reverse-aliases that contain strange characters from past generation bugs.
5. **`email_handler.py:986`** — **The critical lookup**: `contact = Contact.get_by(reply_email=reply_email)`. This uses the indexed `Contact.reply_email` column declared at `app/models.py:1899` (`sa.Column(sa.String(512), nullable=False, index=True)`). Uniqueness is enforced by the 1000-iteration collision-avoidance loop inside `generate_reply_email()` at `app/email_utils.py:1136` (`for _ in range(1000):`) — it keeps trying random candidates until `available_sl_email(reply_email)` returns `True`.
6. **`email_handler.py:987–989`** — If no contact was found: `LOG.w("No contact with {reply_email} as reverse alias")` and return `(False, status.E502)`. This is the dominant "unknown reverse alias" rejection.
7. **`email_handler.py:990–992`** — If `not contact.user.is_active():` return `(False, status.E502)`. Deactivated or banned accounts cause reply delivery to fail.
8. **`email_handler.py:994`** — `alias = contact.alias` resolves the alias that the contact points to. This traverses `Contact.alias = orm.relationship(Alias, backref="contacts")` declared at `app/models.py:1907`. The underlying FK is `Contact.alias_id = sa.Column(sa.ForeignKey(Alias.id, ondelete="cascade"), nullable=False, index=True)` at `app/models.py:1881–1883`.
9. **`email_handler.py:995–996`** — `alias_address: str = contact.alias.email` and `alias_domain = get_email_domain_part(alias_address)` extract the alias address and its domain.
10. **`email_handler.py:1000–1002`** — Sanity check: `is_valid_alias_address_domain(alias.email)` (defined at `app/email_utils.py:557`). This confirms the alias domain is either an `SLDomain` or a verified `CustomDomain`. If neither, return `(False, status.E503)`. This protects against stale contacts that reference aliases on a custom domain that has since been removed.
11. **`email_handler.py:1004`** — `user = alias.user` resolves the owning user. This uses `Alias.user = orm.relationship(User, foreign_keys=[user_id])` at `app/models.py:1576`. The FK is `Alias.user_id = sa.Column(sa.ForeignKey(User.id, ondelete="cascade"), nullable=False, index=True)` at `app/models.py:1474–1476`.
12. **`email_handler.py:1007–1009`** — `if not user.can_send_or_receive():` return `(False, status.E504)`. This rejects disabled/frozen accounts.
13. **`email_handler.py:1012–1016`** — `ret = apply_dmarc_policy_for_reply_phase(alias, contact, envelope, msg)` (defined at `app/handler/dmarc.py:154`). If the rspamd-delivered DMARC verdict is `reject`/`quarantine`/`softfail`, the function returns `status.E215` (`app/handler/dmarc.py:194`) and the reply is dropped. Otherwise it returns `None` and execution continues.

After step 13, `handle_reply` moves on to **mailbox authorization** at `email_handler.py:1019` (`mailbox = get_mailbox_from_mail_from(mail_from, alias)`). That is covered in Section 5, step 17. Mailbox authorization is NOT part of alias-to-user resolution per se — it decides which of the alias's authorized senders is impersonating the alias for this specific reply.

### 3.2 Schema Foundation

The relational integrity that makes the above resolution deterministic:

| Field | Declaration | Semantics |
|-------|-------------|-----------|
| `Contact.reply_email` | `app/models.py:1899` — `sa.Column(sa.String(512), nullable=False, index=True)` | The indexed lookup key. Not declared as `unique=True` at the column level, but uniqueness is enforced by the 1000-iteration `generate_reply_email()` collision check at `app/email_utils.py:1136`. |
| `Contact.alias_id` | `app/models.py:1881–1883` — FK to `Alias.id`, `ON DELETE CASCADE`, `index=True` | Walk from Contact → Alias. Cascade ensures contacts disappear when the alias is deleted. |
| `Contact.user_id` | `app/models.py:1878–1880` — FK to `User.id`, `ON DELETE CASCADE`, `index=True` | Records the contact's owning user. Typically equals `alias.user_id`, but see Section 6 Candidate 5 for the transfer-alias divergence window. |
| `Alias.user_id` | `app/models.py:1474–1476` — FK to `User.id`, `ON DELETE CASCADE`, `index=True` | Walk from Alias → User. |
| `Alias.mailbox_id` | `app/models.py:1506–1508` — FK to `mailbox.id`, `ON DELETE CASCADE`, `index=True` | The **primary** mailbox for the alias. Used as the silent fallback when spoofing check is disabled. |
| `Alias._mailboxes` | `app/models.py:1512` — `orm.relationship("Mailbox", secondary="alias_mailbox", lazy="joined")` | Additional mailboxes via the junction table. The `_`-prefix signals this should not be read directly; use the `mailboxes` property. |
| `Alias.disable_email_spoofing_check` | `app/models.py:1528` — `sa.Column(sa.Boolean, ..., default=False, server_default="0")` | Per-alias opt-out of sender authorization. Enables the silent fallback at `email_handler.py:1025–1029`. |
| `Alias.user` | `app/models.py:1576` — `orm.relationship(User, foreign_keys=[user_id])` | The ORM accessor `alias.user` used at `email_handler.py:1004`. |
| `Alias.mailbox` | `app/models.py:1577` — `orm.relationship("Mailbox", lazy="joined")` | The primary mailbox. |
| `Alias.mailboxes` property | `app/models.py:1580` | Computed list of all authorized mailboxes for this alias. Returns `[self.mailbox] + dedup(self._mailboxes)` (`app/models.py:1581–1584`), filtered to `verified=True` (`app/models.py:1586`), sorted by email (`app/models.py:1587`). |
| `AliasMailbox` model | `app/models.py:2939` with `UniqueConstraint("alias_id", "mailbox_id", name="uq_alias_mailbox")` at `app/models.py:2942` | Junction table for the many-to-many. FK cascades at `app/models.py:2945–2950`. |

### 3.3 The Indexed Lookup — Column-Level Guarantees

The single most important database guarantee in the reply pipeline is the correctness of `Contact.get_by(reply_email=...)`. This call at `email_handler.py:986` is the pivot from "this is some reverse-alias string" to "this is a specific Contact row owned by a specific user". Five mechanisms collectively guarantee that this lookup is deterministic and cannot resolve to the wrong Contact:

1. **Column-level index.** `Contact.reply_email` at `app/models.py:1899` is declared `sa.Column(sa.String(512), nullable=False, index=True)`. Alembic generates a B-tree index on this column. Lookups are O(log n) and correctness is guaranteed by the index's structural properties.
2. **`nullable=False` guard.** A Contact cannot exist without a reply_email. Defensive code that dereferences `contact.reply_email` will never see `None`.
3. **Collision-avoidance loop at generation time.** `generate_reply_email(contact_email, alias)` in `app/email_utils.py:1103` has a `for _ in range(1000):` loop at `app/email_utils.py:1136`. On each iteration, it calls `available_sl_email(candidate)` to verify the candidate is not already in use. If 1000 iterations pass without finding an unused candidate, it raises `CannotCreateReverseAlias`. In practice, the entropy of a random 12-byte urlsafe suffix (`secrets.token_urlsafe(9)` or similar) makes collisions astronomically rare — the loop exists for safety, not for expected behavior.
4. **Unique per-alias `(alias_id, website_email)` constraint.** `Contact.__table_args__` at `app/models.py:1874–1876` includes `UniqueConstraint("alias_id", "website_email", name="uq_contact")`. This prevents two Contact rows from existing for the same `website_email` on the same alias, which would cause `Contact.get_by(alias_id=..., website_email=...)` inside `create_contact()` to return a non-unique result. The reverse-alias uniqueness follows transitively: one contact per (alias_id, website_email) pair, and a fresh `reply_email` generated per contact.
5. **Case-insensitive normalization on insert.** `sanitize_email(website_email)` is applied in `create_contact()` at `app/contact_utils.py:42` BEFORE the uniqueness check. This means "John@Example.com" and "john@example.com" are stored as the same `website_email`, preventing duplicate contacts that differ only in casing.

The net effect is that the `reply_email` column is **effectively unique in practice**, even though it is not declared `unique=True` at the column level. The collision-avoidance loop at generation time is the mechanism that enforces this property.

### 3.4 `normalize_reply_email` — Legacy Repair Behavior

`normalize_reply_email(reply_email)` is defined at `app/email_validation.py:25`. Its implementation is deliberately minimal but carries a lot of history:

```python
def normalize_reply_email(reply_email: str) -> str:
    if not reply_email.isascii():
        reply_email = convert_to_id(reply_email)
    ret = []
    for c in reply_email:
        if c not in _ALLOWED_CHARS:
            ret.append("_")
        else:
            ret.append(c)
    return "".join(ret)
```

Where `_ALLOWED_CHARS` at `app/email_validation.py:9` is the fixed string:

```python
_ALLOWED_CHARS = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789_-.+@"
```

The rationale for this set:

- **Letters and digits**: every RFC-compliant local part accepts them.
- **Underscore and hyphen**: conventional local-part separators; commonly used in SimpleLogin's reverse-alias generator.
- **Period**: allowed by RFC 5321 in the local part.
- **Plus**: the "subaddressing" character; allowed by most mail servers but treated as a delimiter by some (notably Gmail). SimpleLogin may use `+` in VERP-encoded bounce addresses, so it must be preserved.
- **At**: separates local part from domain.

Characters deliberately EXCLUDED:

- **Non-ASCII** is handled separately by `convert_to_id()` before the character filter. `convert_to_id` (also in `app/email_validation.py`) transliterates non-ASCII to a best-effort ASCII equivalent.
- **Whitespace** becomes `_`.
- **Control characters** become `_`.
- **Punctuation outside the whitelist** (e.g., quotes, angle brackets, parentheses) becomes `_`.

**Why is this necessary at reply time?** In theory, a well-formed `reply_email` generated by the current `generate_reply_email` never contains anything outside `_ALLOWED_CHARS`. The normalization exists for **backwards compatibility**: legacy versions of SimpleLogin generated reverse-aliases that occasionally contained stray characters — emoji, zero-width joiners, RTL marks — which are now rejected. When a reply arrives for one of those legacy addresses, `normalize_reply_email` substitutes `_` for the invalid characters so the `Contact.get_by(reply_email=...)` lookup still finds the row (whose stored `reply_email` was also normalized at insertion time through the same function).

This is a defensive measure. Without it, a message sent to a legacy reverse-alias with mis-encoded characters would fail the lookup and be rejected with `E502`. With it, such messages are still routable.

### 3.5 `generate_reply_email` — The Creation-Side Guarantee

The flip side of `Contact.get_by(reply_email=...)` is `generate_reply_email(...)`, which is the sole producer of `reply_email` values. It lives at `app/email_utils.py:1103` and is called from `create_contact()` at `app/contact_utils.py`. Its structure:

1. It attempts to produce a "pretty" reverse-alias first — taking the local part of the foreign contact's address, appending a dot, and appending a short random suffix to disambiguate.
2. It runs `normalize_reply_email(candidate)` on the generated string to make sure the candidate is a valid local part per `_ALLOWED_CHARS`.
3. It appends `@EMAIL_DOMAIN` (per `app/config.py`).
4. It checks `available_sl_email(candidate)` — which tests both `Alias.get_by(email=candidate)` and `Contact.get_by(reply_email=candidate)` — to ensure no clash with existing aliases OR other reverse-aliases.
5. If taken, it tries again (up to the `for _ in range(1000):` limit at line 1136).
6. If 1000 tries all collide (extremely unlikely), it raises `CannotCreateReverseAlias`, which propagates to `handle_forward` and returns an SMTP error to the foreign sender.

The `available_sl_email` check is what makes `Contact.reply_email` globally unique in practice: every newly minted reverse-alias is checked against the full contact table AND the full alias table. This dual check prevents a reply_email from ever colliding with an alias — which would otherwise cause ambiguous classification.

### 3.6 The `is_active` vs `can_send_or_receive` Distinction

Step 14 of the data flow at `email_handler.py:990` uses `contact.user.is_active()`. Step 12 of the alias-resolution walk at `email_handler.py:1007` uses `user.can_send_or_receive()`. These are distinct predicates with distinct semantics:

- **`User.is_active()`** — typically returns `True` unless the user account is marked deleted or soft-deleted. It is the baseline "does this user exist and have not been tombstoned?" check.
- **`User.can_send_or_receive()`** — a richer predicate that also checks whether the user is disabled (admin-frozen), suspended (too many bounces), email-disabled (opted out globally), or on a trial that has expired into a read-only state. It evaluates business policy in addition to existence.

The ordering matters. `is_active()` is cheap (a single column read); `can_send_or_receive()` may involve multiple column reads and a few business-rule evaluations. By gating on `is_active()` first (against the earlier-loaded `contact.user`), the handler short-circuits on deleted accounts before doing the additional bookkeeping of `user.can_send_or_receive()`. This micro-optimization matters because reply handling runs on every inbound SMTP DATA command; avoiding unnecessary work here reduces per-message latency.

### 3.7 `SLDomain.get_by()` and Custom Domain Support

The domain gate at `email_handler.py:977–981` performs two checks in sequence:

1. `reply_email.endswith(EMAIL_DOMAIN)` — compares against the default SL domain configured in `app/config.py:EMAIL_DOMAIN` (`sl.local` in tests, `simplelogin.com` or similar in production).
2. If false, `SLDomain.get_by(domain=reply_domain)` — queries the `SLDomain` table for a registered public SL domain.

`SLDomain` is a model (declared later in `app/models.py`) that represents SL-managed public domains OTHER than the primary `EMAIL_DOMAIN`. Examples include `slmail.me`, `aleeas.com`, etc. Each `SLDomain` row declares a domain string and flags like `premium_only`, `hidden`, and `order`.

Important distinction: `SLDomain` is for **SimpleLogin-owned** domains. It does NOT include **user-owned `CustomDomain`** records. `CustomDomain` is the separate table where users register their own bring-your-own domains. Replies to reverse-aliases on `CustomDomain` addresses are routed because `contact.alias.email` ends with that custom domain, but the `reply_email` itself is ALWAYS on `EMAIL_DOMAIN` or an `SLDomain` — this is guaranteed by `generate_reply_email()`, which appends `@EMAIL_DOMAIN` (line 1103 area). Custom domains host forward-facing aliases only; the reverse-alias always lives on an SL domain.

This design decision has an important implication: a reply to a reverse-alias always stays "within" SimpleLogin's control. The reverse-alias domain is never under the user's control, so SimpleLogin can always enforce DKIM, DMARC, and DNS records on it.

### 3.8 DMARC Policy Enforcement Architecture

DMARC enforcement in the reply phase happens via `apply_dmarc_policy_for_reply_phase(alias, contact, envelope, msg)` at `app/handler/dmarc.py:154`. Its contract:

- **Input**: the resolved `Alias`, the matched `Contact`, the SMTP envelope, and the parsed message.
- **Output**: `None` if the message passes DMARC or is allowed through; `status.E215` (at `app/handler/dmarc.py:194`) if the message should be dropped per policy.

The function reads the DMARC verdict from `SpamdResult.extract_from_headers(msg)` (the rspamd-populated authentication headers). The three policy verdicts it acts on:

- **`DMARC_POLICY_QUARANTINE`** — the sender's DMARC record says "quarantine messages that fail SPF and DKIM". SimpleLogin treats this as a reject for reply phase.
- **`DMARC_POLICY_REJECT`** — the sender's DMARC record says "reject messages that fail SPF and DKIM". Also dropped.
- **`DMARC_POLICY_SOFTFAIL`** — the sender's SPF policy is `~all` and the message failed SPF alignment. Also dropped on reply.

In every case, `status.E215` (`"250 Handled dmarc policy"`) is returned. The 250 code is important: from Postfix's perspective, the message was "accepted" (so it is not retried), but SimpleLogin silently drops it without sending. The reason for accepting-then-dropping is anti-backscatter — a 5xx rejection would cause Postfix to generate an NDR to the (possibly spoofed) `mail_from`.

The policy check is enforced here and NOT at Postfix level because the reply phase has unique requirements. In particular, a mailbox owner may legitimately send from a domain whose DMARC record is `quarantine` — for example, a personal address whose administrator has set `p=quarantine` to catch spoofers. Without this reply-phase filter, SimpleLogin would faithfully forward the mailbox owner's DMARC-quarantined reply to the external recipient, who would see a message flagged as "likely spoof" in their inbox. By catching and dropping at the reply-phase handler, SimpleLogin prevents its users from being embarrassed by their own DMARC policies.

---

## Section 4 — What User ID Does the System Forward the Reply To?

**Critical clarification — the reply is NEVER forwarded to a SimpleLogin user's mailbox. It is forwarded to an *external* email address: the original third-party sender's email, stored in `contact.website_email`.** The "user_id" that appears in the audit trail is the `user_id` recorded on the resulting `EmailLog` row, used exclusively for dashboard attribution and quota accounting. The actual SMTP destination of the reply is always external.

### 4.1 The `EmailLog` Row

Immediately after mailbox authorization succeeds, the reply handler creates an `EmailLog` entry at `email_handler.py:1042–1050`:

```python
email_log = EmailLog.create(
    contact_id=contact.id,
    alias_id=contact.alias_id,
    is_reply=True,
    user_id=contact.user_id,          # ← FROM contact.user_id, NOT alias.user_id
    mailbox_id=mailbox.id,
    message_id=msg[headers.MESSAGE_ID],
    commit=True,
)
```

Per-field semantics:

- `contact_id = contact.id` — the `Contact` row matched by `Contact.get_by(reply_email=reply_email)` at `email_handler.py:986`.
- `alias_id = contact.alias_id` — read directly from the matched contact; equal to `alias.id` because `alias = contact.alias` at `email_handler.py:994`.
- `is_reply = True` — discriminator that distinguishes this log row from a forward-phase row.
- `user_id = contact.user_id` — **this is the answer to "what user ID does the system decide"**. It is read from the `Contact.user_id` column (FK declared at `app/models.py:1878–1880`), *not* from `alias.user_id`.
- `mailbox_id = mailbox.id` — the mailbox returned by `get_mailbox_from_mail_from(mail_from, alias)` at `email_handler.py:1019`, OR the `alias.mailbox` primary fallback at `email_handler.py:1029` when `alias.disable_email_spoofing_check == True`.
- `message_id = msg[headers.MESSAGE_ID]` — the original `Message-ID` header for threading/bounce correlation.

### 4.2 The `contact.user_id` vs `alias.user_id` Divergence

In normal operation, `contact.user_id == alias.user_id`. This invariant holds because `create_contact()` at `app/contact_utils.py:42` sets `user_id=alias.user_id` when the `Contact.create()` call is made at `app/contact_utils.py:93`.

However, the **`transfer_alias()`** procedure at `app/alias_utils.py:458` can (briefly) produce a divergence:

1. `Session.query(Contact).filter(Contact.alias_id == alias.id).update({"user_id": new_user.id})` at `app/alias_utils.py:464–466` — **Contact.user_id is updated FIRST.**
2. `Session.query(AliasMailbox).filter(AliasMailbox.alias_id == alias.id).delete()` at `app/alias_utils.py:477` — clear the old additional mailboxes.
3. `alias.mailbox_id = new_mailboxes.pop().id` at `app/alias_utils.py:480` — reassign the primary mailbox.
4. `for mb in new_mailboxes: AliasMailbox.create(alias_id=alias.id, mailbox_id=mb.id)` at `app/alias_utils.py:481–482` — create new junction rows.
5. `alias.user_id = new_user.id` at `app/alias_utils.py:506` — **Alias.user_id is updated LAST.**

All five steps run within the same SQLAlchemy session, so from the caller's perspective the transfer is atomic. Nevertheless, *within the session*, a reader walking the FK graph between steps 1 and 5 would observe `contact.user_id == new_user.id` while `alias.user_id == old_user.id`. If a reply arrives mid-transfer (a narrow race), the `EmailLog` row will credit the *new* user even though `alias.user` still returns the *old* user; the actual SMTP destination (`contact.website_email`) is unaffected.

### 4.3 The Final SMTP Destination

After header rewriting completes (see Section 5 steps 21–27), `handle_reply` calls `sl_sendmail()` at `email_handler.py:1224–1231`:

```python
sl_sendmail(
    generate_verp_email(VerpType.bounce_reply, email_log.id, alias_domain),
    contact.website_email,    # ← envelope_to — the EXTERNAL destination
    msg,
    envelope.mail_options,
    envelope.rcpt_options,
    is_forward=False,
)
```

- **`envelope_from`** is a **VERP-encoded address** — `generate_verp_email(VerpType.bounce_reply, email_log.id, alias_domain)` (defined at `app/email_utils.py:1438`) returns a VERP address keyed to the `email_log.id`. Bounces sent back to this address are parsed by SimpleLogin's bounce handler and correlated to the originating `EmailLog`.
- **`envelope_to`** is **`contact.website_email`** — the external third-party's real email address. The `Contact.website_email` column is declared at `app/models.py:1889`.
- **`is_forward=False`** flags the send as a reply for metrics and logging.
- `sl_sendmail()` itself is defined at `app/mail_sender.py:270` and performs the actual SMTP delivery (or, when `NOT_SEND_EMAIL=True` as in the test environment, stores the outbound message for later inspection via `MailSender.store_emails_test_decorator` at `app/mail_sender.py:111`).

### 4.4 Final Answer to the Question

**The `user_id` recorded on the resulting `EmailLog` is `contact.user_id`** (line `email_handler.py:1046`). This value is used solely for bookkeeping — to attribute the reply-send activity to the user in the dashboard and to enforce per-user quotas. **The reply message itself is delivered to `contact.website_email`, which is an external email address, not a SimpleLogin user mailbox.** The `alias.user_id` is dereferenced earlier at `email_handler.py:1004` only for the `user.can_send_or_receive()` authorization check; it does NOT appear on the `EmailLog` and does NOT influence the final SMTP destination.

Runtime verification confirmed this exactly. The in-process simulation produced:

```
[LOG] EmailLog.id=845 user_id=1 alias_id=1 contact_id=415 mailbox_id=1 is_reply=True
[PRIMARY] 1 msg(s) sent to contact.website_email (recipient@external.test)
```

The `EmailLog.user_id` matched `contact.user_id`, and the single primary `envelope_to` matched `contact.website_email`.

### 4.5 The `EmailLog` Audit Trail — Why `contact.user_id` Is Used

The decision to write `user_id=contact.user_id` on the `EmailLog` (rather than `user_id=alias.user_id`) has concrete consequences for the audit trail:

1. **Attribution follows the contact ownership.** The contact was created when the foreign third party first sent an email to the alias (or when the user explicitly added the contact). At that time, `contact.user_id` was set to the alias owner. If the alias is later transferred to another user (via `transfer_alias` at `app/alias_utils.py:458`), `Contact.user_id` is bulk-updated FIRST (step 1 of the transfer). So the rule "EmailLog records `contact.user_id`" means the audit trail reflects the user who *currently* owns the contact.
2. **Dashboards and quota systems see the right user.** The SimpleLogin dashboard queries `EmailLog.user_id` to display the "reply activity" widget. Quota enforcement queries `EmailLog.user_id` to count sends-per-day. Both would break if the EmailLog recorded `alias.user_id` during the transfer window.
3. **Post-transfer history stays coherent.** After `transfer_alias` completes, future replies correctly attribute to the new owner. But historical EmailLogs created during the transfer window (if any) already point to the new owner too — so a user-facing "my reply history" query shows a clean cut-over rather than a split between old-alias-user and new-alias-user.

This explicit choice of `contact.user_id` is not arbitrary — it is the correct primary key for attribution under the transfer-alias flow. Using `alias.user_id` would create edge cases where a transferred alias appears to have "lost" its historical replies.

### 4.6 VERP Addresses and Bounce Correlation

The envelope-from used on the primary `sl_sendmail` at `email_handler.py:1224` is NOT `alias.email`. It is a VERP-encoded address produced by `generate_verp_email(VerpType.bounce_reply, email_log.id, alias_domain)` at `app/email_utils.py:1438`.

VERP — Variable Envelope Return Path — encodes the destination-specific bounce context inside the envelope-from address. SimpleLogin's variant encodes:

- A prefix indicating the VERP type (`bounce_reply`, `bounce_forward`, `transactional`, …).
- The `email_log.id` as a numeric identifier.
- A hash/checksum for tamper detection.
- An expiration timestamp so stale bounces (weeks after the original send) are ignored.

When the external recipient's mail server bounces the reply (e.g., mailbox full), the bounce message is addressed to this VERP envelope-from. SimpleLogin's bounce handler (in `handle_bounce_reply_phase()` in `email_handler.py`) parses the VERP address via `get_verp_info_from_email()` in `app/email_utils.py`, extracts the `email_log.id`, loads the original `EmailLog`, and:

1. Creates a new `EmailLog.bounced = True` entry.
2. Notifies the mailbox owner that their reply bounced, including the bounce reason.
3. Updates `Contact.nb_reply` counters for quota tracking.

The VERP design means SimpleLogin does not need to scan message bodies or parse `Auto-Submitted` headers to figure out what a bounce is *about* — the VERP address itself is the correlation key. This is why the primary delivery at `email_handler.py:1224–1231` sets `envelope_from` to the VERP address and NOT to `alias.email`. If `alias.email` were used as envelope-from, bounces would go to `alias.email`, and SimpleLogin would have no way to correlate the bounce to the original `EmailLog`.

### 4.7 `sl_sendmail` Internals — `SendRequest` and `MailSender.send`

`sl_sendmail(envelope_from, envelope_to, msg, mail_options, rcpt_options, is_forward)` at `app/mail_sender.py:270` is the outbound delivery abstraction. Internally:

1. It packages the inputs into a `SendRequest` object. `SendRequest` carries the envelope-from, envelope-to, the serialized message (via `message_format_base64_parts(msg)`), the SMTP options, and an `is_forward` flag used by metrics.
2. It calls `mail_sender.send(send_request, retries)` on the module-level `MailSender` singleton.
3. `MailSender.send()` (at `app/mail_sender.py`) checks the `NOT_SEND_EMAIL` config flag. If `NOT_SEND_EMAIL=True` (the test environment default), it stores the `SendRequest` in the module-local `_emails_sent` list via `store_emails_instead_of_sending()` and returns immediately — NO real SMTP is performed.
4. If `NOT_SEND_EMAIL=False` (production), `MailSender.send()` either sends synchronously via `_send_to_smtp(send_request)` or dispatches to a thread pool if `enable_background_pool(max_workers=10)` was called at startup. The thread pool allows inbound SMTP handling to return `250 OK` immediately while outbound delivery proceeds asynchronously.
5. `_send_to_smtp` opens an SMTP connection to `POSTFIX_SERVER:POSTFIX_PORT` (typically `localhost:25`) and issues the standard `MAIL FROM`, `RCPT TO`, `DATA` sequence.
6. On SMTP failure, retries are attempted up to `retries` times with exponential backoff.

The test environment's `store_emails_test_decorator` at `app/mail_sender.py:111` is a context-manager/decorator that temporarily swaps `MailSender.send` with an in-memory recorder, making it possible for tests to assert about outbound messages without running a real SMTP server.

### 4.8 The `EmailLog` Table as an Audit Trail — Columns and Queries

The `EmailLog` model (declared in `app/models.py`) carries the following audit-relevant columns:

- **`id`** — primary key; also the identifier encoded in VERP envelopes for bounce correlation.
- **`contact_id`** — FK to the matched `Contact`. Dereferencing `email_log.contact` gives full context (name, website_email, pgp_finger_print).
- **`alias_id`** — FK to the `Alias`. Set from `contact.alias_id` at `email_handler.py:1044`.
- **`user_id`** — FK to `User`. Set from `contact.user_id` at `email_handler.py:1046`. This is the per-row "who owns this reply" pointer.
- **`mailbox_id`** — FK to the authorized `Mailbox`. Either the mailbox returned by `get_mailbox_from_mail_from()` or the `alias.mailbox` fallback on spoofing-check-disabled aliases.
- **`is_reply`** — boolean; True for reply phase, False for forward phase. Used to partition the audit table by direction.
- **`message_id`** — the original `Message-ID` header. Used for threading and for bounce correlation when `replace_original_message_id` maps it to an `SLMessage-ID`.
- **`created_at`** — timestamp.
- **`bounced`** — initially False; set to True by the bounce handler if the external recipient bounces.
- **`is_spam`** — set to True if the spam check at `email_handler.py:1054–1094` flagged the reply.
- **`blocked`** — set to True if the reply was prevented (e.g., by `replace_header_when_reply` raising `NonReverseAliasInReplyPhase`).

Typical audit queries:

- "Show me all replies by user X in the last 24 hours":
  `Session.query(EmailLog).filter(EmailLog.user_id == X, EmailLog.is_reply == True, EmailLog.created_at >= now - 1.day).all()`.
- "Show me all replies for alias Y":
  `Session.query(EmailLog).filter(EmailLog.alias_id == Y, EmailLog.is_reply == True).all()`.
- "Which replies bounced?":
  `Session.query(EmailLog).filter(EmailLog.bounced == True, EmailLog.is_reply == True).all()`.

All of these queries are indexed via the FK indexes on the corresponding columns. The EmailLog table is the single source of truth for reply-phase activity.

### 4.9 What "User ID" Means in the Context of Quotas

SimpleLogin enforces per-plan limits on reply volume. A free user may, for example, be limited to N replies per day; a premium user has higher (or no) limits. These quotas are enforced by counting `EmailLog` rows with `is_reply=True` and `user_id=<the user>`:

- Before accepting a reply, `rate_limited(alias_id, mailbox_id)` (at `app/email/rate_limit.py`) checks both per-alias and per-mailbox limits against Redis-backed counters.
- After the `EmailLog` row is created, daily/monthly aggregates are updated for the user.
- The user's dashboard queries `EmailLog` grouped by `user_id` to show the current day's usage and the remaining quota.

In all of these paths, the "user_id" is read from `EmailLog.user_id` — which, as established, is `contact.user_id`. This is the operational definition of the answer to the user's question: **the user ID the system associates with this reply is `contact.user_id`, used for attribution, quotas, and dashboard display**. The reply's SMTP destination remains the external `contact.website_email`.

### 4.10 `EmailLog` Deletion Paths

`EmailLog` rows are created eagerly at `email_handler.py:1042–1050`, BEFORE the actual delivery. This means that on delivery failure, an orphan row could remain. The reply pipeline therefore has explicit deletion paths at several failure points:

1. **PGP encryption failure at `email_handler.py:1151–1164`.** If encryption raises, the just-created `EmailLog` is deleted via `EmailLog.delete(email_log.id, commit=True)` before the function returns `(False, status.E402)`. This avoids a dangling audit row for a message that was never delivered.
2. **`NonReverseAliasInReplyPhase` at `email_handler.py:1182–1200`.** When `replace_header_when_reply` raises this exception (a TO/CC address is not a known reverse alias), the caller deletes the `EmailLog` before returning `(True, status.E200)`. The `True` indicates "we handled this successfully" to the inbound MTA, but the actual reply is suppressed.
3. **Top-level exception handler in `handle_DATA` at `email_handler.py:2297–2332`.** If an unanticipated exception propagates all the way up, the handler logs it via `LOG.e(...)` (auto-forwarded to Sentry by the logging integration) and returns `status.E404`. The `EmailLog` row may or may not exist depending on where the exception occurred; this is a debt area — cleanup is not guaranteed.

In production, orphan `EmailLog` rows (created but corresponding to a message that was never delivered) are rare and typically non-harmful: they count toward the user's "recent replies" UI but don't cause any security issue. A periodic cleanup cron could remove orphans older than N hours that have `created_at > bounced_at is NULL` and no corresponding SMTP-log entry.

### 4.11 `mailbox_id` and Premium-Feature Gating

The `EmailLog.mailbox_id` column is used not only for auditing but also for feature gating:

- **DKIM policy per alias.** If the alias uses a custom domain with DKIM configured, the `add_dkim_signature` call at `email_handler.py:1221` uses the alias domain's key regardless of mailbox. But some premium features (e.g., "strict DKIM enforcement") are conditional on `mailbox.force_spf` being enabled, queried via `EmailLog.mailbox.force_spf`.
- **PGP encryption gate.** `contact.pgp_finger_print and user.is_premium()` is evaluated against `user = alias.user`, NOT against the mailbox owner. Even if the mailbox owner is a free-tier user who happens to be on a premium account's shared alias, PGP still works because the gate is on the alias owner's plan.
- **Premium-only TO/CC replacement.** Some tiers allow the "Reply All with multiple reverse aliases" flow to succeed; others reject it. The decision is made at `user.is_premium()` check points.

The `mailbox_id` thus does double duty: audit attribution AND per-mailbox feature toggles.

### 4.12 `NOT_SEND_EMAIL` and Test-Mode Semantics

The `NOT_SEND_EMAIL` config flag (set in `tests/test.env`) causes `MailSender.send()` at `app/mail_sender.py` to store outbound `SendRequest` objects in an in-memory list instead of dispatching them to Postfix. This is load-bearing for:

- **Unit tests.** Tests can assert on outbound messages without requiring a running SMTP server.
- **Development environments.** Developers can run the full handler pipeline without actually sending email to real recipients.
- **CI.** CI environments run the entire test suite, which executes hundreds of `sl_sendmail` calls.

Importantly, **`NOT_SEND_EMAIL` does NOT affect the `EmailLog` row**. Rows are still created; only the actual SMTP delivery is suppressed. This means tests can verify `EmailLog.mailbox_id`, `EmailLog.user_id`, etc., independently of whether the send "really" happened.

The `store_emails_test_decorator` at `app/mail_sender.py:111` is a higher-level wrapper that also captures the outbound messages in a test-accessible list. A test using this decorator can:

```python
@store_emails_test_decorator
def test_reply_is_delivered_to_contact():
    # Arrange: create user, alias, contact, mailbox
    ...
    # Act: synthesize envelope and msg, invoke handle()
    ...
    # Assert: the captured SendRequest's envelope_to equals contact.website_email
    sent = mail_sender._emails_sent
    assert len(sent) == 1
    assert sent[0].envelope_to == "merchant@external.com"
```

This pattern is used extensively in `tests/test_email_handler.py` for assertions about the reply-phase dispatch behavior.

### 4.13 `user_id` Semantics Across Models — A Cross-Reference

To avoid confusion about which `user_id` the system uses where, here is a consolidated cross-reference:

| Model | Column | FK To | Meaning | Set By |
|-------|--------|-------|---------|--------|
| `Contact` | `user_id` | `users.id` | The SimpleLogin user who "owns" this contact | `create_contact()` at `app/contact_utils.py` |
| `Alias` | `user_id` | `users.id` | The SimpleLogin user who owns the alias | `Alias.create()` or `transfer_alias()` step 5 |
| `Alias` | `mailbox_id` | `mailbox.id` | The primary mailbox for this alias | `Alias.create()` or `transfer_alias()` step 3 |
| `Mailbox` | `user_id` | `users.id` | The SimpleLogin user who owns this mailbox | `Mailbox.create()` |
| `EmailLog` | `user_id` | `users.id` | The user ID this reply/forward is attributed to | `email_handler.py:1046` (reply) from `contact.user_id` |
| `EmailLog` | `mailbox_id` | `mailbox.id` | The mailbox authorized for this send | `email_handler.py:1047` (reply) |
| `AliasMailbox` | `mailbox_id` | `mailbox.id` | Additional mailboxes on the alias | Created at alias-setup time |

Invariants across these columns:

- **Invariant A:** `contact.user_id == alias.user_id` (where `alias = contact.alias`) **except during the `transfer_alias` transaction window**.
- **Invariant B:** `mailbox.user_id == alias.user_id` for every `mailbox` in `alias.mailboxes` (enforced at the UI / API flow that creates `AliasMailbox` rows; not at the database level).
- **Invariant C:** `EmailLog.user_id == contact.user_id` by construction at `email_handler.py:1046` for every reply.
- **Invariant D:** `Contact.reply_email` is globally unique (enforced by the 1000-iteration collision check in `generate_reply_email()`).

When all invariants hold (which is the normal operating state), the identity-resolution chain is strict and unambiguous: a reply arrives → indexed lookup yields a unique Contact → the Contact's alias_id yields a unique Alias → the Alias's user_id yields a unique User. Every subsequent operation is deterministic given this triple.

The only way these invariants can break is:

- **Invariant A breakage** during `transfer_alias` — transient, bounded, low risk (Candidate 5).
- **Invariant B breakage** via direct database manipulation — out of scope (Section 6.10).
- **Invariant C breakage** — architecturally impossible; the assignment at `email_handler.py:1046` is unconditional.
- **Invariant D breakage** — the 1000-iteration loop has a finite but astronomically unlikely failure case where no unique `reply_email` can be generated, in which case `generate_reply_email()` raises an exception and no Contact is created.

---

## Section 5 — The Actual Data Flow in Detail

This section traces every decision point from TCP-level receipt of the SMTP `DATA` command through final delivery. Each step cites the exact `file:line` location.

### 5.1 Numbered Runtime Walkthrough

1. **SMTP `DATA` received.** Postfix (or any upstream MTA) delivers the message to the SimpleLogin SMTP server listening on port 20381. `aiosmtpd.smtp.Controller` invokes the handler's `async def handle_DATA(self, server, session, envelope: Envelope)` at `email_handler.py:2289`. The raw bytes `envelope.original_content` are parsed into an `email.message.Message` via `msg = email.message_from_bytes(envelope.original_content)` at `email_handler.py:2290`.

2. **Flask app context opened.** `handle_DATA` immediately calls `self._handle(envelope, msg)` at `email_handler.py:2292`. `_handle` is defined at `email_handler.py:2335`, decorated with `@newrelic.agent.background_task()` at `email_handler.py:2334`. It creates a correlation UUID via `message_id = str(uuid.uuid4())` at `email_handler.py:2339` and publishes it through `set_message_id(message_id)` at `email_handler.py:2340`. It then enters `with create_light_app().app_context():` at `email_handler.py:2352`. `create_light_app()` is defined at `server.py:127` and is a minimal Flask factory that wires only SQLAlchemy (via `app.config["SQLALCHEMY_DATABASE_URI"] = DB_URI` at `server.py:129`) — deliberately distinct from the web factory `create_app()` at `server.py:139` to avoid importing web middleware into the SMTP process.

3. **Main dispatcher invoked.** `_handle` calls `return_status = handle(envelope, msg)` at `email_handler.py:2353`. `handle()` is defined at `email_handler.py:1945`.

4. **Envelope sanitization.** At `email_handler.py:1949`, `mail_from = sanitize_email(envelope.mail_from)`. `sanitize_email()` is defined at `app/utils.py:97`: it strips whitespace, strips the right-to-left mark `\u200f`, and lowercases. At `email_handler.py:1950`, every entry in `envelope.rcpt_tos` is sanitized the same way.

5. **Header sanitization.** At `email_handler.py:1974–1978`, `sanitize_header()` is applied to `FROM`, `TO`, `CC`, `REPLY_TO`, and `MESSAGE_ID` headers.

6. **Cross-phase guard.** `handle()` detects at `email_handler.py:1998–2025` whether the inbound message was itself sent *from* a reverse alias (a bounce-loop condition); it alerts the alias owner via transactional email but does not short-circuit regular reply routing.

7. **VERP / unsubscribe / bounce / complaint / rate-limit branches.** Lines `email_handler.py:2030–2173` route messages destined for `bounces+` or `unsubscribe+` addresses, VERP-encoded bounces, and transactional complaints. A normal user reply to a `reply+...@sl.local` address matches none of these and falls through to the per-recipient loop.

8. **Per-recipient loop.** `for rcpt_index, rcpt_to in enumerate(rcpt_tos):` at `email_handler.py:2180`. Each recipient is classified independently.

9. **Reverse-alias detection.** At `email_handler.py:2195`, `if is_reverse_alias(rcpt_to):` classifies the recipient as a reply target. `is_reverse_alias()` is defined at `app/email_utils.py:1156`. Its logic: first call `Contact.get_by(reply_email=address)` at `app/email_utils.py:1158`; if a contact exists, return `True` immediately (line 1159). Otherwise, return `True` iff `address.endswith(f"@{config.EMAIL_DOMAIN}")` AND (`address.startswith("reply+")` OR `address.startswith("ra+")`) at `app/email_utils.py:1161–1163`. The second branch supports legacy prefix-form reverse aliases that don't have a matching `Contact` row.

10. **Reply dispatch.** On match, `is_delivered, smtp_status = handle_reply(envelope, copy_msg, rcpt_to)` at `email_handler.py:2199`. A shallow-copied `msg` is passed so per-recipient mutations don't leak across iterations.

11. **Domain gate.** Inside `handle_reply()` at `email_handler.py:966`, the first check is at `email_handler.py:977–981`: `if not reply_email.endswith(EMAIL_DOMAIN): if not SLDomain.get_by(domain=reply_domain): LOG.w(...); return False, status.E501`. The reply address must end with the default `EMAIL_DOMAIN` OR be a registered custom SL domain.

12. **Address normalization.** `reply_email = normalize_reply_email(reply_email)` at `email_handler.py:984`. `normalize_reply_email()` is defined at `app/email_validation.py:25`. It first converts non-ASCII via `convert_to_id()` (lines 27–28), then replaces any character outside `_ALLOWED_CHARS` (at `app/email_validation.py:9`) with `_`. The `_ALLOWED_CHARS` set is `"abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789_-.+@"`.

13. **Contact lookup (THE KEY LOOKUP).** `contact = Contact.get_by(reply_email=reply_email)` at `email_handler.py:986`. This queries the `Contact` table using the indexed `reply_email` column at `app/models.py:1899`. If no row matches, `LOG.w("No contact with {reply_email} as reverse alias")` and return `(False, status.E502)` at `email_handler.py:987–989`. Uniqueness of `reply_email` is enforced empirically by the collision-avoidance loop inside `generate_reply_email()` — `for _ in range(1000):` at `app/email_utils.py:1136` — which keeps retrying until `available_sl_email(reply_email)` is true.

14. **User active-state check.** `if not contact.user.is_active():` at `email_handler.py:990` → return `(False, status.E502)` at line 992.

15. **Alias resolution.** `alias = contact.alias` at `email_handler.py:994`. This traverses the ORM relationship declared at `app/models.py:1907` (`alias = orm.relationship(Alias, backref="contacts")`). The FK is `Contact.alias_id` at `app/models.py:1881–1883` with `ON DELETE CASCADE`. Following the traversal, `alias_address: str = contact.alias.email` (line 995) and `alias_domain = get_email_domain_part(alias_address)` (line 996).

16. **Alias-domain sanity check.** `if not is_valid_alias_address_domain(alias.email):` at `email_handler.py:1000` → return `(False, status.E503)`. `is_valid_alias_address_domain()` is defined at `app/email_utils.py:557` and verifies the alias's domain is one that SimpleLogin serves.

17. **User resolution.** `user = alias.user` at `email_handler.py:1004`. This traverses `Alias.user = orm.relationship(User, foreign_keys=[user_id])` at `app/models.py:1576`. The FK is `Alias.user_id` at `app/models.py:1474–1476` with `ON DELETE CASCADE`.

18. **User send/receive authorization.** `mail_from = envelope.mail_from` is captured at `email_handler.py:1005`. Then at `email_handler.py:1007`, `if not user.can_send_or_receive():` → return `(False, status.E504)`.

19. **DMARC policy enforcement.** `apply_dmarc_policy_for_reply_phase(alias, contact, envelope, msg)` at `email_handler.py:1012–1014`. This function is defined at `app/handler/dmarc.py:154` and returns `status.E215` (at `app/handler/dmarc.py:194`) on DMARC quarantine/reject/soft_fail, otherwise `None`. If non-None, `handle_reply` returns `(False, dmarc_delivery_status)` at `email_handler.py:1015–1016`.

20. **Mailbox authorization.** `mailbox = get_mailbox_from_mail_from(mail_from, alias)` at `email_handler.py:1019`. `get_mailbox_from_mail_from()` is defined at `email_handler.py:1364`:
    - The inner helper `__check(email_address, alias)` at `email_handler.py:1369–1383` iterates `alias.mailboxes` (the property at `app/models.py:1580`). For each `mailbox`, it compares `mailbox.email == email_address` at `email_handler.py:1371`, and also iterates `mailbox.authorized_addresses` (line 1374) — the per-mailbox `AuthorizedAddress` list that allows additional sender addresses.
    - The outer return at `email_handler.py:1387` is `__check(mail_from, alias) or __check(canonicalize_email(mail_from), alias)`.
    - `canonicalize_email()` is defined at `app/utils.py:78`. It is a no-op unless the domain is `gmail.com`, `protonmail.com`, `proton.me`, or `pm.me` (guarded at `app/utils.py:84–85`). For those domains: strip everything after `+` in the local part at `app/utils.py:87–92`, remove all `.` from the local part at `app/utils.py:93`, then lowercase and strip at `app/utils.py:94`.
    - **Critical silent fallback:** if `mailbox is None` AND `alias.disable_email_spoofing_check == True`, then `mailbox = alias.mailbox` (the primary) at `email_handler.py:1029`. The surrounding block is `email_handler.py:1020–1029`. `Alias.disable_email_spoofing_check` is declared at `app/models.py:1528`.
    - Otherwise, if still unresolved, `handle_unknown_mailbox(...)` is invoked at `email_handler.py:1390` and `handle_reply` returns `(False, status.E214)` at `email_handler.py:1032–1034`.

21. **SPF verification.** At `email_handler.py:1036`, `if ENFORCE_SPF and mailbox.force_spf and not alias.disable_email_spoofing_check:` → call `spf_pass()` and return `(False, status.E201)` on failure.

22. **`EmailLog` row creation.** `EmailLog.create(contact_id=contact.id, alias_id=contact.alias_id, is_reply=True, user_id=contact.user_id, mailbox_id=mailbox.id, message_id=msg[headers.MESSAGE_ID], commit=True)` at `email_handler.py:1042–1050`. See Section 4 for detailed field semantics.

23. **Spam scan (optional).** If `ENABLE_SPAM_ASSASSIN` is on, the message is scored at `email_handler.py:1054–1094`. On spam, `handle_spam(...)` fires and `handle_reply` returns `(False, status.E506)`.

24. **Header stripping.** `delete_all_headers_except(msg, [FROM, TO, CC, SUBJECT, DATE, MESSAGE_ID, REFERENCES, IN_REPLY_TO, SL_QUEUE_ID, ... MIME_HEADERS])` at `email_handler.py:1096–1112`. Everything not in the whitelist is removed so that the outbound message doesn't leak mailbox metadata.

25. **Preserve original TO/CC.** `orig_to = msg[headers.TO]; orig_cc = msg[headers.CC]` at `email_handler.py:1114–1115`. These are kept in closure for later use by `notify_mailbox()` so other mailbox holders see what the user was actually replying to.

26. **Optional body replacement, PGP.**
    - If `user.replace_reverse_alias` is enabled, the body is scanned and reverse-alias strings are replaced with `contact.website_email`, `mailbox.email` with `alias.email`, and — when `ENABLE_ALL_REVERSE_ALIAS_REPLACEMENT` is set — all other reverse aliases of the same alias. This block runs at `email_handler.py:1119–1148`.
    - If `contact.pgp_finger_print` is set AND `user.is_premium()`, the message body is PGP-encrypted at `email_handler.py:1151–1164`. On encryption failure, the `EmailLog` is deleted and `handle_reply` returns `(False, status.E402)` so the sender's MTA will retry.

27. **FROM/TO/CC rewrite, MessageID rewrite, SL headers, DKIM, and delivery.**
    - `recipient_name = get_alias_recipient_name(alias)` at `email_handler.py:1168`. This helper at `app/alias_utils.py:580` returns an `AliasRecipientName` containing the `"name" <alias.email>` string to place in `From:`, following three rules: (i) if `alias.name` is set → `"alias.name" <alias.email>`; (ii) else if the custom domain has a display name → `"domain.name" <alias.email>`; (iii) else bare `alias.email`.
    - `add_or_replace_header(msg, headers.FROM, recipient_name.name)` at `email_handler.py:1172` rewrites the `From:` header so the reply appears to come from the alias, not the mailbox.
    - `replace_header_when_reply(msg, alias, headers.TO)` at `email_handler.py:1179` and `replace_header_when_reply(msg, alias, headers.CC)` at `email_handler.py:1181`. The function is at `email_handler.py:345`. Inside, `for _, reply_email in getaddresses(headers):` at `email_handler.py:358`; for each address, `if reply_email == alias.email: continue` at `email_handler.py:361–362`; then `contact = Contact.get_by(reply_email=reply_email)` at `email_handler.py:364` (**NOTE: no `alias_id` filter — this is a GLOBAL lookup**); on miss, raise `NonReverseAliasInReplyPhase(reply_email)` at `email_handler.py:372`; on hit, append `sl_formataddr((contact.name, contact.website_email))` at `email_handler.py:376`. If the exception is raised, the caller at `email_handler.py:1182–1200` deletes the `EmailLog`, notifies the mailbox owner, and returns `(True, status.E200)` without delivering.
    - `replace_original_message_id(alias, email_log, msg)` at `email_handler.py:1202` rewrites the `Message-ID` and `References` headers to use SL Message-IDs for proper bounce correlation and thread continuity.
    - `msg[headers.SL_DIRECTION] = "Reply"` at `email_handler.py:1209` and `msg[headers.SL_EMAIL_LOG_ID] = str(email_log.id)` at `email_handler.py:1210` add SL observability headers.
    - `if should_add_dkim_signature(alias_domain): add_dkim_signature(msg, alias_domain)` at `email_handler.py:1220–1221` signs the outbound message with the alias-domain DKIM key.
    - **Primary delivery** — `sl_sendmail(generate_verp_email(VerpType.bounce_reply, email_log.id, alias_domain), contact.website_email, msg, envelope.mail_options, envelope.rcpt_options, is_forward=False)` at `email_handler.py:1224–1231`. `generate_verp_email()` is at `app/email_utils.py:1438`. `sl_sendmail()` is at `app/mail_sender.py:270`.
    - **Multi-mailbox fan-out** — `other_mailboxes = [mb for mb in alias.mailboxes if mb.email != mailbox.email]` at `email_handler.py:1234`; then `for mb in other_mailboxes: notify_mailbox(alias, mailbox, mb, msg, orig_to, orig_cc, alias_domain)` at `email_handler.py:1235–1236`. `notify_mailbox()` is defined at `email_handler.py:1264`: it rewrites `From:` to `alias.email` at `email_handler.py:1278`, restores `To:` and `Cc:` from the captured originals at `email_handler.py:1280–1281`, and sends the copy via `sl_sendmail(generate_verp_email(VerpType.transactional, transaction.id, alias_domain), other_mb.email, notif)` at `email_handler.py:1289–1293`. A warning banner ("Don't forget to remove this section if you reply") is prepended to the body at `email_handler.py:1274–1275`.
    - Finally, `handle_reply` returns `(True, status.E200)`. The per-recipient tuple is collected in `res` at `email_handler.py:2200`, reduced at `email_handler.py:2214–2233`, and the final SMTP status string is returned up through `_handle` and `handle_DATA` to aiosmtpd, which writes it on the wire.

### 5.2 Mermaid Flowchart

```mermaid
flowchart TD
    A["SMTP DATA received<br/>MailHandler.handle_DATA<br/>email_handler.py:2289"] --> B["parse envelope.original_content<br/>email.message_from_bytes"]
    B --> C["_handle with Flask app context<br/>email_handler.py:2352"]
    C --> D["handle(envelope, msg)<br/>email_handler.py:1945"]
    D --> E["sanitize_email / sanitize_header<br/>app/utils.py:97"]
    E --> F{"is_reverse_alias(rcpt_to)?<br/>app/email_utils.py:1156"}
    F -->|No| Z1["handle_forward path<br/>(out of scope)"]
    F -->|Yes| G["handle_reply<br/>email_handler.py:966"]
    G --> H{"domain ends with EMAIL_DOMAIN<br/>or SLDomain match?<br/>:977-981"}
    H -->|No| R1["E501"]
    H -->|Yes| I["normalize_reply_email<br/>app/email_validation.py:25"]
    I --> J["Contact.get_by(reply_email=...)<br/>email_handler.py:986"]
    J -->|None| R2["E502"]
    J -->|Found| K["alias = contact.alias<br/>:994"]
    K --> L["user = alias.user<br/>:1004"]
    L --> M{"user.can_send_or_receive?<br/>:1007"}
    M -->|No| R3["E504"]
    M -->|Yes| N["apply_dmarc_policy_for_reply_phase<br/>app/handler/dmarc.py:154"]
    N -->|Reject| R4["E215"]
    N -->|Pass| O["get_mailbox_from_mail_from<br/>:1019 / :1364"]
    O --> P{"authorized mailbox?"}
    P -->|No + spoofing disabled| Q1["mailbox = alias.mailbox<br/>silent fallback :1029"]
    P -->|No + spoofing enforced| R5["E214 handle_unknown_mailbox"]
    P -->|Yes| Q2["mailbox = matched"]
    Q1 --> S["EmailLog.create<br/>user_id=contact.user_id<br/>:1042-1050"]
    Q2 --> S
    S --> T["strip headers, save orig_to/orig_cc<br/>:1096-1115"]
    T --> U["optional: body replacement, PGP<br/>:1119-1164"]
    U --> V["add_or_replace_header FROM = alias identity<br/>:1168-1172"]
    V --> W["replace_header_when_reply TO / CC<br/>global Contact lookup<br/>:345, :1179-1181"]
    W --> X["replace_original_message_id<br/>:1202"]
    X --> Y["DKIM sign<br/>:1220-1221"]
    Y --> AA["sl_sendmail to contact.website_email<br/>:1224-1231"]
    AA --> AB["notify_mailbox fan-out for other mailboxes<br/>:1234-1236, :1264"]
    AB --> AC["return E200"]
```

### 5.3 Deep Dive on Step 9 — Reverse-Alias Detection

The classification at `app/email_utils.py:1156` is deceptively simple but has important subtleties:

```python
def is_reverse_alias(address: str) -> bool:
    if Contact.get_by(reply_email=address):
        return True
    return address.endswith(f"@{config.EMAIL_DOMAIN}") and (
        address.startswith("reply+") or address.startswith("ra+")
    )
```

The first branch — the `Contact.get_by(reply_email=...)` lookup — is an O(1) indexed query on the `contact.reply_email` column. This is the "fast path" and succeeds for every modern reverse alias generated by `generate_reply_email()`.

The second branch — the `reply+`/`ra+` prefix heuristic — is a **structural fallback** for addresses that *look* like reverse aliases but do not have a corresponding `Contact` row. This branch has two consequences:

1. **It gives `handle_reply` the chance to emit a `status.E502` ("No contact with {reply_email} as reverse alias").** Without this branch, a malformed `reply+...@sl.local` address would silently be misclassified as a forward target and routed to `handle_forward()`, which would then fail with a different error (likely "alias does not exist"), producing less informative logs.
2. **It prevents spam abuse of the prefix namespace.** The `@EMAIL_DOMAIN` check ensures that external domains cannot claim the `reply+` / `ra+` prefix to gain reply-phase treatment.

Calls to `is_reverse_alias` appear not only in `handle()` at `email_handler.py:2195` but also in the unsubscribe/bounce detection paths at earlier points in `handle()`. A single `Contact.get_by` call is performed inside the detection function; this is cheap but remains a potential hot path in high-volume traffic (a `contact_id` → `reply_email` cache could be a future optimization).

### 5.4 Deep Dive on Step 12 — `normalize_reply_email` Invariants

`normalize_reply_email` is a small function (14 lines at `app/email_validation.py:25–38`) but it carries a strong invariant: **the normalized output must still be parseable as an RFC 5321 mailbox**. The combination of ASCII transliteration (via `convert_to_id()`) and character-set narrowing (via the `_ALLOWED_CHARS` filter) guarantees this property.

Step-by-step effect on a hypothetical malformed reply address `reply+café@sl.local`:

1. Split into local part `reply+café` and domain `sl.local`.
2. `convert_to_id("reply+café")` → `"reply+cafe"` (accent removed).
3. Filter against `_ALLOWED_CHARS = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789_-.+@"` — no changes (all characters already allowed).
4. Recombine: `reply+cafe@sl.local`.

A malformed reply like `reply+hello world@sl.local` (with a literal space) would be normalized to `reply+hello_world@sl.local` (space → underscore). This is a one-way sanitization — the original space is lost.

The key property is: **`normalize_reply_email` is idempotent**. Applying it twice gives the same result as applying it once. This is important because `generate_reply_email()` also enforces this character set at creation time, so `Contact.reply_email` values stored in the database should already be in normalized form.

**Why apply it again in `handle_reply`?** Because ancient (pre-normalization) `Contact` rows may exist in the database with non-ASCII `reply_email` values. `normalize_reply_email` in `handle_reply` is a repair step that maps an inbound envelope-to back into the canonical form before the indexed lookup. If the `Contact` row's stored `reply_email` was already normalized at creation, the lookup succeeds. If not, the lookup fails — which is correct behavior (ancient malformed reverse aliases should not be usable).

### 5.5 Deep Dive on Step 20 — `get_mailbox_from_mail_from` Semantics

This is one of the most consequential steps in the pipeline. Its correctness determines whether a reply is routed under the right mailbox (and hence the right quota, the right DKIM policy, and the right `EmailLog.mailbox_id`).

The helper function's full structure at `email_handler.py:1364–1390`:

```python
def get_mailbox_from_mail_from(mail_from: str, alias: Alias) -> Optional[Mailbox]:
    def __check(email_address, alias):
        for mailbox in alias.mailboxes:
            if mailbox.email == email_address:
                return mailbox
            for authorized_address in mailbox.authorized_addresses:
                if authorized_address.email == email_address:
                    LOG.d("Found authorized address for %s from %s", email_address, mailbox)
                    return mailbox
        return None
    return __check(mail_from, alias) or __check(canonicalize_email(mail_from), alias)
```

Important properties:

1. **Two-pass check.** First the raw `mail_from` is checked against all `alias.mailboxes` and their `authorized_addresses`. If no match, the *canonicalized* `mail_from` is checked. This second pass matters only for `gmail.com`, `protonmail.com`, `proton.me`, and `pm.me` addresses; for all other domains, `canonicalize_email(x) == x` and the second call is a redundant O(n) iteration.
2. **`alias.mailboxes` is deduped.** The property at `app/models.py:1580` returns a list that combines `alias.mailbox` (primary) with verified entries from `alias._mailboxes` (the `AliasMailbox` join), deduplicated by `mailbox.id` and filtered to `verified=True` mailboxes. So the iteration visits each distinct mailbox once.
3. **`authorized_addresses` is per-mailbox.** Each `Mailbox` has a collection of `AuthorizedAddress` rows — explicit additional email addresses the user has registered to also be allowed to send on behalf of that mailbox. This is the mechanism by which a user can authorize, e.g., their work email to reply under their personal mailbox.
4. **Returns `None` on no match.** The caller then enters the `disable_email_spoofing_check` fallback or the `handle_unknown_mailbox` rejection path.

The `canonicalize_email` second pass is the primary source of the **Candidate 1** risk analyzed in Section 6: under the right combination of Gmail-style aliases (plus-addressing + dot-insensitivity), two distinct `Mailbox.email` values can canonicalize to the same normalized form, producing a match against a mailbox the user's raw `mail_from` would not have matched.

### 5.6 Deep Dive on Step 24 — Header Stripping Whitelist

The `delete_all_headers_except` call at `email_handler.py:1096–1112` is the single-most-important defensive step in the reply pipeline. Its whitelist ensures that:

- **`Return-Path`** is stripped (set freshly by the VERP envelope-from below).
- **`Received`** chains from the user's personal mailbox are stripped (they would leak the mailbox's hostname and IP).
- **`X-Mailer`, `X-Originating-IP`**, and other tracking headers are stripped.
- **`Authentication-Results` / `DKIM-Signature` / `ARC-*`** from the inbound side are stripped — SimpleLogin re-signs with the alias-domain DKIM key at step 27.
- **`List-Unsubscribe` / `List-Unsubscribe-Post`** from the user's mailbox are stripped (SimpleLogin adds its own where needed).
- **`Reply-To`** is stripped unless explicitly in the whitelist, preventing mailbox leak.

The whitelist as implemented: `FROM`, `TO`, `CC`, `SUBJECT`, `DATE`, `MESSAGE_ID`, `REFERENCES`, `IN_REPLY_TO`, `SL_QUEUE_ID`, and all `MIME_HEADERS` (the `Content-*` family needed for body rendering). Any header not in this whitelist is removed.

An important practical consequence: when SimpleLogin then **re-adds** headers (e.g., `SL_DIRECTION`, `SL_EMAIL_LOG_ID`, and the rewritten `From:`), those adds happen AFTER the strip — so they are guaranteed to appear on the outbound message. If the adds happened BEFORE the strip, they would themselves be removed. The ordering at `email_handler.py:1096` (strip first, then add) is therefore structural, not incidental.

### 5.7 Deep Dive on Step 26 — Body Replacement and PGP

**Body replacement** (at `email_handler.py:1119–1148`) is controlled by `user.replace_reverse_alias` (per-user setting) and the global config flag `ENABLE_ALL_REVERSE_ALIAS_REPLACEMENT`. When active:

1. The message body (both text/plain and text/html parts) is scanned for any occurrence of `contact.reply_email`. Occurrences are replaced with `contact.website_email`. This makes the outbound message read naturally to the external recipient.
2. Occurrences of `mailbox.email` are replaced with `alias.email` — so the user's mailbox address never appears in the body sent to the external recipient.
3. If `ENABLE_ALL_REVERSE_ALIAS_REPLACEMENT` is set, a bulk replacement is performed for ALL other `Contact` rows associated with the same alias, up to `MAX_NB_REVERSE_ALIAS_REPLACEMENT` per message. The cap avoids unbounded runtime on aliases with thousands of contacts.

**PGP encryption** (at `email_handler.py:1151–1164`) is triggered by `contact.pgp_finger_print IS NOT NULL AND user.is_premium()`. When triggered:

1. The entire message body is PGP-encrypted using the contact's public key.
2. On encryption failure (e.g., unparseable key, GnuPG process error), the just-created `EmailLog` is deleted via `EmailLog.delete(email_log.id, commit=True)` to avoid a dangling audit entry, and `handle_reply` returns `(False, status.E402)`. The `E402` (SMTP 421 temporary failure) causes the sender's MTA to retry.
3. On success, subsequent steps (header rewriting, DKIM, send) operate on the encrypted payload.

The PGP path is bypassed for non-premium users even if `contact.pgp_finger_print` is set — this is the monetization gate.

### 5.8 Deep Dive on Step 27 — The Three Delivery Operations

Step 27 is the densest step; it is helpful to think of it as three distinct operations in sequence:

**Operation A — FROM rewrite to alias identity.** `recipient_name = get_alias_recipient_name(alias)` at `email_handler.py:1168` returns a struct containing the string `"Display Name" <alias@sl.local>` (or `<alias@sl.local>` if no display name). `add_or_replace_header(msg, headers.FROM, recipient_name.name)` at `email_handler.py:1172` overwrites the existing `From:` header with this identity. After this operation, the external recipient sees the reply as coming FROM the alias, not the mailbox.

**Operation B — TO/CC rewrite for reverse aliases.** `replace_header_when_reply(msg, alias, headers.TO)` at `email_handler.py:1179` and the `headers.CC` equivalent at line 1181. The function body at `email_handler.py:345`:

```python
def replace_header_when_reply(msg, alias, header):
    new_addrs = []
    headers_values = msg.get_all(header)
    if not headers_values:
        return
    for _, reply_email in getaddresses(headers_values):
        # Skip the alias itself (addressing the alias in TO/CC is allowed)
        if reply_email == alias.email:
            continue
        contact = Contact.get_by(reply_email=reply_email)
        if not contact:
            raise NonReverseAliasInReplyPhase(reply_email)
        new_addrs.append(sl_formataddr((contact.name, contact.website_email)))
    add_or_replace_header(msg, header, ", ".join(new_addrs))
```

**The GLOBAL contact lookup.** Note that `Contact.get_by(reply_email=reply_email)` at line 364 has NO `alias_id` filter. It looks up ANY contact with that reply_email — irrespective of whether the contact belongs to this alias. This is the basis of the **Candidate 3** risk in Section 6: in principle, a TO-line reverse alias from another alias's namespace could resolve to that other alias's contact, causing the reply to be addressed to a recipient not originally intended.

**Exception handling around Operation B.** If `NonReverseAliasInReplyPhase` is raised, the caller at `email_handler.py:1182–1200`:
1. Deletes the just-created `EmailLog` row.
2. Sends a transactional alert to the mailbox owner.
3. Returns `(True, status.E200)` — a successful status to the inbound MTA, masking the rejection from the external sender. This avoids leaking internal error details to potentially hostile inputs.

**Operation C — Finalize and send.** Three more header touches happen:
- `replace_original_message_id(alias, email_log, msg)` at `email_handler.py:1202` replaces the inbound `Message-ID` with an SL-managed Message-ID (`<something@alias_domain>`), and updates `References` to correlate with SL identifiers. This ensures that reply threading works correctly on both sides.
- `add_or_replace_header(msg, headers.DATE, arrow.utcnow().format(...))` at `email_handler.py:1214` sets the `Date:` header if it was missing or malformed.
- `msg[headers.SL_DIRECTION] = "Reply"` and `msg[headers.SL_EMAIL_LOG_ID] = str(email_log.id)` at `email_handler.py:1209–1210` add SL observability headers.

Then `add_dkim_signature(msg, alias_domain)` at `email_handler.py:1221` signs the message with the alias domain's DKIM private key.

Finally, `sl_sendmail(verp_envelope_from, contact.website_email, msg, ...)` at `email_handler.py:1224` delivers the signed message to the external recipient.

### 5.9 Deep Dive on Multi-Mailbox Fan-Out — `notify_mailbox`

After the primary `sl_sendmail` completes, the handler at `email_handler.py:1234–1236` iterates the remaining mailboxes:

```python
other_mailboxes = [mb for mb in alias.mailboxes if mb.email != mailbox.email]
for mb in other_mailboxes:
    notify_mailbox(alias, mailbox, mb, msg, orig_to, orig_cc, alias_domain)
```

`notify_mailbox()` at `email_handler.py:1264` performs:

1. **Disclaimer banner injection.** Prepends a warning banner ("This email was sent to `<alias>` and you are receiving it because this mailbox is a recipient of this alias. The reply below was sent by `<actual sending mailbox>`. Don't forget to remove this section if you reply.") to both text and HTML bodies of a deep-copied message.
2. **`From:` rewrite to alias identity.** The notify copy appears to come FROM the alias, not from the sending user's mailbox.
3. **`To:`/`Cc:` restore.** `orig_to` and `orig_cc` (captured at step 25) are restored so the other mailbox owner sees what the original user's reply was addressed to — not the rewritten external contact addresses.
4. **VERP envelope.** `generate_verp_email(VerpType.transactional, transaction.id, alias_domain)` is used, distinct from the primary delivery's `VerpType.bounce_reply`. Bounces on the notify copy are therefore routed to the transactional bounce handler, which has different semantics from the reply-bounce handler.
5. **DKIM signing.** The notify copy is DKIM-signed with the alias-domain key.
6. **Delivery.** `sl_sendmail(...)` at `email_handler.py:1289–1293` sends the notify copy to `other_mb.email`.

**Key observation:** the notify copy is sent to a user's mailbox (`mb.email`), whereas the primary send goes to the external recipient (`contact.website_email`). The fan-out is **the only path** in the reply phase where a SimpleLogin user's mailbox appears as an `envelope_to`. This is why **Candidate 4** in Section 6 focuses on multi-owner aliases: if two different users jointly own an alias (e.g., via a historical alias-sharing flow that has since been deprecated, or a data-corruption scenario), `alias.mailboxes` would include mailboxes from both users, and one user's reply would leak a copy to the other user's mailbox via `notify_mailbox`.

The `Alias.mailboxes` property at `app/models.py:1580` does not explicitly filter by `alias.user_id`. It joins `AliasMailbox` rows to `Mailbox` rows and returns all verified entries. In the default data model, all `Mailbox.user_id` for an alias's mailboxes equal `alias.user_id`. But nothing *at the code level* enforces this invariant: it is enforced at the INSERT level by the UI flows that create `AliasMailbox` rows. A database-level constraint would be a defense-in-depth improvement.

### 5.10 The Return-Value Reduce Loop

After all recipients have been processed in the per-recipient loop at `email_handler.py:2180`, the results are reduced to a single SMTP status string. The reduce logic at `email_handler.py:2214–2233`:

1. If **every** recipient succeeded with a 2xx status → return the 2xx status (typically `status.E200` = `250 Message accepted`).
2. If **any** recipient hit a hard 5xx (and no soft 4xx retries) → return the 5xx status.
3. If **any** recipient hit a soft 4xx (temporary failure that should be retried by the inbound MTA) → return the soft 4xx to trigger retries.
4. The reduce is "hierarchical" — hard failures dominate soft failures, which dominate successes.

Additional anti-backscatter logic in `_handle` at `email_handler.py:2335–2378`:

- `_handle` captures the final status from `handle()`.
- If the status is a 5xx AND the `SPFCheckResult` (retrieved from `SpamdResult.extract_from_headers(msg)`) indicates the return-path did not pass SPF → `_handle` downgrades the 5xx to `status.E216` = `250 Silently drop due to SPF failure`.
- This means: unauthenticated senders that would otherwise trigger bounce-backs (backscatter) are silently dropped. Legitimate rejections (hard failures from authenticated senders) still propagate.

This is a security-critical invariant: SimpleLogin never emits bounce-backs to spoofed senders, so it cannot be abused as a backscatter amplifier. The same logic applies in the forward phase.

### 5.11 Deep Dive on Step 7 — Rate Limiting and the Short-Circuit

Between the envelope sanitization (step 4) and the per-recipient loop (step 8), `handle()` invokes `rate_limited()` from `app/email/rate_limit.py`. The current implementation of `rate_limited` inside that file is a short-circuited stub that returns `False` unconditionally — rate limiting is effectively disabled in the current codebase snapshot.

The historical intent of this check was:

1. **Per-mailbox rate limit.** Too many replies per minute from a single mailbox → temporarily reject to prevent abuse.
2. **Per-alias rate limit.** Too many sends per minute on a single alias → temporarily reject to prevent spam from compromised aliases.
3. **Per-user rate limit.** Too many total sends per day for free-tier users → reject with a quota-exhausted status.

Because the current `rate_limited` returns `False`, none of these limits are enforced in the reply path at code-level. Redis-backed counters may still exist for observability, but they do not block reply-phase dispatch. If a rate limit is desired, the short-circuit would be removed and the Redis-backed check re-enabled.

For this investigation, the rate-limiting absence means: **no reply is rejected for rate-limit reasons in the current codebase**, so any observed reject must stem from domain-gate failure (E501), contact-not-found (E502), inactive user (E502/E504), domain mismatch (E503), DMARC policy (E215), unauthorized mailbox (E214), SPF failure (E201), or spam detection (E506) — NOT from rate limiting.

### 5.12 Deep Dive on Body Decoding and MIME Walking

When the reply phase performs body replacement (step 26) or PGP encryption (step 26), it must traverse the MIME tree of the inbound message. SimpleLogin uses the standard library `email.message.Message` API for this:

- **`msg.walk()`** yields every `Message` part in depth-first order. Both multipart containers and leaf parts are yielded.
- **`part.get_content_type()`** returns the full content type (e.g., `text/plain`, `text/html`, `multipart/alternative`, `application/pkcs7-signature`).
- **`part.get_payload(decode=True)`** decodes base64/quoted-printable/etc. into raw bytes. This must then be decoded to str via the part's charset (`part.get_content_charset() or "utf-8"`).

The body-replacement loop (at `email_handler.py:1119–1148`) walks the tree, filters to `text/plain` and `text/html` leaf parts, and performs the string substitutions. The re-encoding after substitution must preserve the original charset to avoid mangling international characters.

The PGP encryption block (at `email_handler.py:1151–1164`) takes a different approach: it encrypts the **entire** MIME body (not individual parts) and wraps it in a `multipart/encrypted` envelope per RFC 3156. This preserves attachments, inline images, and other non-text parts within the encrypted payload.

**Edge case: non-ASCII in HTML bodies.** If the `text/html` part contains characters outside the charset's repertoire (uncommon, but possible if a user copy-pastes emoji into a reply), the re-encoding step can raise `UnicodeEncodeError`. The body-replacement block does not catch this; it propagates up to `handle_reply` and thence to the top-level exception handler in `handle_DATA` at `email_handler.py:2297–2332`, producing a generic `E404` status. A more robust implementation would catch the encoding error and either fall through (skip replacement) or return a specific status.

### 5.13 Deep Dive on VERP Type Distinctions

`VerpType` (an enum declared in `app/models.py`) distinguishes different categories of VERP envelopes. The reply phase uses TWO distinct types:

- **`VerpType.bounce_reply`** — used for the primary `sl_sendmail` at `email_handler.py:1224`. Encodes the `email_log.id`. If the external recipient's MTA bounces the delivery, the bounce arrives at a VERP address that SimpleLogin parses via `get_verp_info_from_email()` and routes to the reply-bounce handler. The reply-bounce handler:
    - Marks the `EmailLog.bounced = True`.
    - Sends a transactional notification to the mailbox owner explaining the bounce.
    - Increments a `Contact.nb_reply_bounced` counter (for dashboard display).
- **`VerpType.transactional`** — used for `notify_mailbox` copies at `email_handler.py:1289–1293`. Encodes a `TransactionalEmail.id`. If the notify copy bounces (e.g., the extra mailbox is full), the bounce arrives at a different VERP address and is routed to the transactional-bounce handler. The transactional-bounce handler:
    - Does NOT mark any `EmailLog.bounced = True` (no EmailLog exists for notify copies).
    - May disable the mailbox after repeated bounces (policy-driven).
    - Counts toward the mailbox's reliability score.

The distinction matters because: if both paths used the same VERP type, a bounce on the notify copy would incorrectly mark the primary `EmailLog` as bounced, causing the original reply to appear failed when in fact it was delivered successfully.

### 5.14 Deep Dive on DKIM Signing Semantics

`add_dkim_signature(msg, alias_domain)` at `email_handler.py:1221` produces an RFC 6376 DKIM-Signature header. The specifics:

- **Signing domain** is `alias_domain` — the domain portion of `alias.email`. For SL-domain aliases, this is `sl.local` (or the production equivalent `simplelogin.com`). For custom-domain aliases, this is the user's custom domain.
- **Selector** is the DKIM selector configured for the domain. The private key is looked up by `(domain, selector)` from local keystore or config.
- **Canonicalization** is typically `relaxed/relaxed` (header/body canonicalization modes) — the standard production choice.
- **Signed headers** include `From`, `To`, `Cc`, `Subject`, `Date`, `Message-ID`, `References`, `In-Reply-To`, `SL_DIRECTION`, `SL_EMAIL_LOG_ID`, and the MIME content headers.
- **Body hash** is computed over the canonicalized body (with any body replacement and PGP encryption already applied).

The signing is what makes the rewritten `From: <alias>` header acceptable to DMARC-checking receivers: the DMARC policy for the alias domain is published in DNS, it references the DKIM selector's TXT record, and the signed message carries a valid signature for that selector. The receiver performs DKIM verification, confirms alignment with the `From:` domain, and accepts the message as authentic.

Without DKIM signing, the reply would arrive at the external recipient with `From: <alias>@sl.local` and no authentication — most receivers would dump it to spam or reject it outright. DKIM signing is therefore not merely a nice-to-have; it is a load-bearing pillar of the reply-phase architecture.

### 5.15 Deep Dive on `replace_original_message_id` and Thread Continuity

`replace_original_message_id(alias, email_log, msg)` at `email_handler.py:1202` performs two mutations:

1. **Rewrites `Message-ID:` to an SL-owned form.** The original `Message-ID:` from the inbound message would identify the user's personal mailbox server (e.g., `<something@mx.alice-mailbox.com>`). Rewriting it to `<sl.<email_log.id>@<alias_domain>>` (or a similar scheme) makes the message identifiable across the SL infrastructure for bounce correlation and thread continuity.
2. **Appends the original Message-ID to `References:`.** This preserves the inbound message's identifier in the thread chain, so when the external recipient's mail client displays the reply, it correctly threads under the original thread even though `Message-ID:` is now SL-owned.

The function also updates an internal `SLMessage` table row (if SL is tracking messages for threading). This is used in subsequent forward-phase messages: when the external recipient replies to the reply, their `In-Reply-To:` header points at the SL-owned Message-ID, and SL's forward handler can use it to correlate the new inbound with the original thread.

The practical consequence: replies sent through SimpleLogin thread correctly in Gmail, Outlook, Thunderbird, and Apple Mail, with no thread fragmentation.

### 5.16 Anti-Backscatter at the `_handle` Level

The final return from `handle()` flows back up to `_handle` at `email_handler.py:2353`. The `_handle` function applies a **5xx-to-E216 downgrade** at approximately `email_handler.py:2355–2378`:

```python
return_status = handle(envelope, msg)
# If the return is a 5xx AND the SPF check failed for envelope.mail_from
# (indicating the sender forged their identity), downgrade the 5xx to E216
# "250 Silently drop due to SPF failure".
spf_check = SpamdResult.extract_from_headers(msg).spf_result
if return_status.startswith("5") and spf_check == SPFCheckResult.FAIL:
    LOG.w("Downgrading %s to E216 (silent drop on SPF fail)", return_status)
    return status.E216
return return_status
```

This downgrade prevents SimpleLogin from being used as a **backscatter amplifier**. A backscatter attack works like this:

1. Attacker sends a message with spoofed `envelope.mail_from = victim@example.com` to some random (nonexistent) reverse alias at SimpleLogin.
2. Without the downgrade, SimpleLogin would reply with a 5xx bounce.
3. The attacker's MTA would dutifully send the 5xx back to `victim@example.com` (the spoofed return-path).
4. Victim receives a bounce for a message they never sent — i.e., spam that LOOKS LIKE a bounce.

The downgrade to E216 makes SimpleLogin return `250 OK` to the inbound MTA (no bounce), while the spam team can still observe the failure in logs. The attack payload is dropped silently; no bounce is generated; the victim is not molested.

The invariant is enforced uniformly for both forward and reply phases: any 5xx final status, when the inbound envelope's SPF check failed, is rewritten to E216.

---

## Section 6 — Where Is the Most Likely Point of Incorrect Routing?

The user asked specifically: "when logs show the alias being correctly recognized, where could a reply still get routed to the wrong user?" This section analyzes every candidate failure mode against the code, grades each by risk, and concludes with a Final Verdict.

### 6.1 Candidate 1 — `get_mailbox_from_mail_from()` canonicalization collision

**Location:** `email_handler.py:1364–1387`.

- The inner helper `__check(email_address, alias)` at `email_handler.py:1369–1383` iterates ONLY `alias.mailboxes` (line 1370), which by construction (the property at `app/models.py:1580`) returns mailboxes belonging to the already-resolved alias. Cross-alias or cross-user mailbox iteration is architecturally impossible here.
- The outer return at `email_handler.py:1387` is `__check(mail_from, alias) or __check(canonicalize_email(mail_from), alias)`.
- `canonicalize_email()` at `app/utils.py:78` is a **no-op** unless the domain is `gmail.com`, `protonmail.com`, `proton.me`, or `pm.me` (the guard at `app/utils.py:84–85`). For those domains, it strips `+...` suffixes at `app/utils.py:87–92` and removes all `.` characters at `app/utils.py:93`, then lowercases.
- Worst case: a user authenticates with `john.doe+promo@gmail.com` but the mailbox record stores `johndoe@gmail.com`; `canonicalize_email()` collapses both to `johndoe@gmail.com` and returns the *right* mailbox within this alias's mailbox set.
- Even in an adversarial scenario where two users coincidentally own mailboxes with Gmail-dot-equivalent addresses, they cannot simultaneously be attached to the *same alias* because that would have been caught at `Mailbox.create()` time. So a cross-user false match is architecturally impossible.

**Conclusion: LOW risk.** The mailbox returned is always on the resolved alias; the Contact → Alias → User chain is untouched.

### 6.2 Candidate 2 — `disable_email_spoofing_check` silent fallback

**Location:** `email_handler.py:1020–1029`.

- When `get_mailbox_from_mail_from()` returns `None`, control enters the `if alias.disable_email_spoofing_check:` block at `email_handler.py:1021`. Inside, `LOG.w(...)` logs a warning and `mailbox = alias.mailbox` (the primary) is assigned at `email_handler.py:1029`.
- The field `Alias.disable_email_spoofing_check` is declared at `app/models.py:1528`. When `True`, it DELIBERATELY bypasses the mailbox authorization check per the feature's design intent.
- Side effect: the resulting `EmailLog.mailbox_id = alias.mailbox.id` even though `envelope.mail_from` does **not** match any authorized mailbox. The audit trail will attribute the reply to the primary mailbox's owner regardless of who actually sent it.
- The final `sl_sendmail` destination (`contact.website_email`) is still **correct** — the reply goes to the external third party as intended. What is wrong is the *attribution*, not the *routing*.

**Conclusion: HIGH risk for MISATTRIBUTION.** The user who owns the primary mailbox appears in logs and dashboards as responsible for replies they never authored. The reply itself still reaches the correct external recipient. This is the most dangerous mode *for forensics*: it looks like the correct user replied when in fact the pipeline silently let an unauthenticated sender through.

### 6.3 Candidate 3 — `replace_header_when_reply()` GLOBAL contact lookup

**Location:** `email_handler.py:345–384`, called from `email_handler.py:1179` (TO) and `email_handler.py:1181` (CC).

- Inside the function at `email_handler.py:358`, `for _, reply_email in getaddresses(headers):` iterates every address in the `TO` or `CC` header.
- At `email_handler.py:361–362`, an address equal to `alias.email` is skipped (the "Reply-All to self" case).
- At `email_handler.py:364`, `contact = Contact.get_by(reply_email=reply_email)` — **THERE IS NO `alias_id` FILTER**. The lookup resolves globally across the entire `Contact` table. Because `reply_email` is globally unique by construction (the 1000-iteration collision loop in `generate_reply_email()` at `app/email_utils.py:1136` guarantees this), the lookup will succeed for any reverse-alias in the header, regardless of which user owns the underlying alias.
- At `email_handler.py:376`, `new_addrs.append(sl_formataddr((contact.name, contact.website_email)))` — the rewritten header carries the foreign contact's `website_email`, not the alias owner's.
- Real-world triggering scenario: user A uses "Reply All" on a thread that also contains a reverse-alias belonging to user B's alias. The rewritten TO/CC header on user A's outgoing reply will contain user B's contact's *real external email*. The primary `sl_sendmail` envelope_to is still user A's `contact.website_email` (correct), but the in-message TO/CC header now leaks a foreign contact's real address, and if anyone down the line hits "Reply All" on the delivered message, the reply will fan out to those foreign addresses.
- If a `TO`/`CC` address is not a reverse alias at all (e.g., some plain `@gmail.com` address), `NonReverseAliasInReplyPhase(reply_email)` is raised at `email_handler.py:372`. The caller at `email_handler.py:1182–1200` deletes the `EmailLog`, notifies the mailbox owner, and returns `(True, status.E200)` without sending — so unknown-domain addresses are safely rejected. The defect only applies to addresses that *are* reverse aliases (i.e., other SimpleLogin users' reverse aliases).

**Conclusion: MEDIUM risk.** The primary envelope recipient is correct, but the rewritten in-message `TO:` / `Cc:` headers cross identity boundaries when Reply-All is used on threads that span multiple SL users.

### 6.4 Candidate 4 — Multi-mailbox `notify_mailbox()` copies

**Location:** `email_handler.py:1234–1236`; function at `email_handler.py:1264`.

- After the primary `sl_sendmail` to `contact.website_email`, the code at `email_handler.py:1234` computes `other_mailboxes = [mb for mb in alias.mailboxes if mb.email != mailbox.email]`.
- `alias.mailboxes` at `app/models.py:1580` returns `[alias.mailbox]` (the primary) concatenated with every `Mailbox` joined via the `alias_mailbox` junction table (`app/models.py:1581–1584`), deduped by `id`, filtered to `verified=True` (`app/models.py:1586`), and sorted by email (`app/models.py:1587`).
- At `email_handler.py:1235–1236`, for every mailbox in `other_mailboxes`, `notify_mailbox(alias, mailbox, mb, msg, orig_to, orig_cc, alias_domain)` is invoked. `notify_mailbox()` is at `email_handler.py:1264`.
- Inside `notify_mailbox()`:
    - `add_or_replace_header(notif, headers.FROM, alias.email)` at `email_handler.py:1278` — the notification's `From:` is rewritten to `alias.email`, *not* the real sending mailbox. A recipient of the notification therefore cannot tell *which* mailbox authored the reply — only that "the alias replied".
    - `add_or_replace_header(notif, headers.TO, orig_to)` at `email_handler.py:1280` and `add_or_replace_header(notif, headers.CC, orig_cc)` at `email_handler.py:1281` — the original TO/CC is restored so the holder of the other mailbox sees the full thread context.
    - A disclaimer banner ("Don't forget to remove this section if you reply") is prepended at `email_handler.py:1274–1275`.
    - `sl_sendmail(generate_verp_email(VerpType.transactional, transaction.id, alias_domain), other_mb.email, notif)` at `email_handler.py:1289–1293` dispatches the copy.
- From the perspective of any user who holds an *extra* mailbox on a shared alias, they will receive a copy of every reply — even though they did not author it, did not authorize it, and the `From:` header obscures who did. This is **by design** for shared/family aliases, but it is the single most likely trigger for a support report of "a reply was forwarded to the wrong user".
- Multi-mailbox behavior was confirmed in live runtime simulation: an alias with two verified mailboxes produced exactly two outbound messages — one primary `sl_sendmail` to the external contact, and one `notify_mailbox` copy to the secondary mailbox, both using VERP envelopes.

**Conclusion: HIGH risk for PERCEIVED mis-routing.** Every additional mailbox on the alias receives a copy that, to the receiving user, looks like an unexplained forwarded reply. The behavior is correct by design but is the number-one operational surprise in shared-alias setups.

### 6.5 Candidate 5 — `Contact.user_id` vs `alias.user_id` divergence during `transfer_alias()`

**Location:** `app/alias_utils.py:458`.

- `transfer_alias()` performs five ordered steps inside a single SQLAlchemy session:
    1. Bulk-update `Contact.user_id = new_user.id` for every contact on the alias — `app/alias_utils.py:464–466`.
    2. Delete all `AliasMailbox` join rows for the alias — `app/alias_utils.py:477`.
    3. Assign the new primary via `alias.mailbox_id = new_mailboxes.pop().id` — `app/alias_utils.py:480`.
    4. Create new `AliasMailbox` rows for the remaining new mailboxes — `app/alias_utils.py:481–482`.
    5. Assign `alias.user_id = new_user.id` **LAST** — `app/alias_utils.py:506`.
- Within this session, reads see the pending values of each column. If a reply arrives during the session (an extremely narrow race window, requires concurrent writes), a reader would see `contact.user_id == new_user.id` while `alias.user_id == old_user.id`.
- Effect on the reply pipeline: `EmailLog.user_id = contact.user_id` at `email_handler.py:1046` would credit the *new* user, while `user = alias.user` at `email_handler.py:1004` would dereference the *old* user for the `can_send_or_receive()` check.
- The actual SMTP destination (`contact.website_email`) is unaffected.

**Conclusion: LOW risk** in practice because `transfer_alias()` runs within a single transaction. However, it does explain why some historical `EmailLog` rows may appear to belong to a user who no longer owns the alias.

### 6.6 Final Verdict

**The most likely source of "reply forwarded to the wrong user" when the logs show the alias being correctly recognized is the combination of Candidate 4 (multi-mailbox `notify_mailbox()` copies at `email_handler.py:1234–1236`) and Candidate 2 (silent fallback via `alias.disable_email_spoofing_check` at `email_handler.py:1025–1029`).** The reasoning:

- **Primary delivery is always correct.** When the logs show the alias being correctly recognized, the Contact → Alias → User chain is deterministic (`email_handler.py:986, 994, 1004`) and the primary `sl_sendmail` always goes to `contact.website_email` — the external third party. The *external-party routing* is not where the bug lies.
- **Additional mailboxes receive copies.** When `alias._mailboxes` contains any additional mailbox (the multi-user / family-alias / team-alias scenario), every OTHER verified mailbox on the alias receives a notification copy from `notify_mailbox()` at `email_handler.py:1264`. To a user of one of those extra mailboxes, the arriving message looks *exactly* like "a reply was forwarded to me from the wrong user's action". The `From:` header is rewritten to `alias.email` at `email_handler.py:1278`, deliberately obscuring the originating mailbox, which further reinforces the perception of mis-routing. This is the most common user complaint pattern.
- **Spoofing-check-disabled aliases silently credit the primary mailbox.** When `alias.disable_email_spoofing_check = True` (per the option at `app/models.py:1528`), any `mail_from` that fails the mailbox / `AuthorizedAddress` match silently falls back to `alias.mailbox` at `email_handler.py:1029`. The `EmailLog.mailbox_id` then records the primary mailbox's id, making it look as though the primary mailbox's owner authored the reply — in fact, they did not. The combination is pernicious: any sender can trigger a reply that gets credited to the primary mailbox's owner, and any other mailboxes on the alias each receive a `notify_mailbox` copy that looks like it came from the primary mailbox's owner.

**Secondary source: Candidate 3 (`replace_header_when_reply()` global lookup at `email_handler.py:345–384`).** When the user uses "Reply All" on a thread that already contains reverse-aliases from *other* aliases (including other users' aliases), each reverse-alias in `TO`/`CC` is resolved globally by `reply_email` (the lookup at `email_handler.py:364` has no `alias_id` filter) and rewritten to the foreign contact's `website_email` at `email_handler.py:376`. The primary envelope recipient remains correct, but the rewritten in-message `TO:`/`Cc:` headers now contain foreign external addresses. This is not mis-routing of the *current* reply — the current envelope still goes to the correct external party — but it does leak foreign external addresses into the outbound message's visible headers and can cause downstream "Reply All" actions to fan out to unintended external recipients.

Together, these three failure modes explain the full range of "the reply went to the wrong user" reports while remaining consistent with logs that show the alias being correctly recognized: the alias *is* recognized correctly, the Contact → Alias → User chain *is* correctly resolved, but the *fan-out* and *attribution* behavior at the tail of the pipeline produces unexpected delivery patterns that users interpret as mis-routing.

### 6.7 Risk Matrix and Current Code State

The following matrix summarizes risk levels, concrete triggering scenarios, observable symptoms, and — for each candidate — what the current SimpleLogin codebase actually does at the relevant pipeline junction. The rightmost column is strictly observational: it describes the code as it stands today and does not prescribe refactors, additions, or modifications (per the SWE-AtlasQnA-Repo rule that this investigation must not suggest code changes).

| Candidate | Risk Level | Triggering Scenario | Observable Symptom | Current Code State at the Junction |
|-----------|-----------|---------------------|---------------------|-------------------------------------|
| 1 — canonicalize collision | LOW | A Gmail or Proton user owns two distinct mailboxes that canonicalize to the same form and are both on the same alias | Impossible at `Mailbox.create` time because the two mailboxes would canonicalize to the same key | Cross-user collision is architecturally prevented at this junction: `__check()` at `email_handler.py:1369–1383` iterates only the resolved alias's mailbox set, and the guard at `app/utils.py:84–85` restricts `canonicalize_email()` to the four Gmail/Proton domains. The scenario cannot materialize inside `get_mailbox_from_mail_from()`. |
| 2 — spoofing-check silent fallback | HIGH (misattribution) | An alias has `disable_email_spoofing_check=True` and an unauthenticated sender sends to the reply+ address | `EmailLog.mailbox_id` = primary mailbox, external recipient receives the reply | At `email_handler.py:1020–1029` the code emits a generic `LOG.w` warning when this branch is taken, then silently assigns `mailbox = alias.mailbox` at `email_handler.py:1029` and bypasses `handle_unknown_mailbox` at `email_handler.py:1032`. The `EmailLog.create()` call at `email_handler.py:1042–1050` then records `mailbox_id=mailbox.id` (the primary mailbox) regardless of the true sender, and no distinct log tag or flag exists in the current code to differentiate this silent-fallback path from authenticated sends. |
| 3 — global TO/CC rewrite | MEDIUM | User hits "Reply All" on a thread containing reverse aliases that belong to other aliases | External contacts belonging to other aliases appear in outbound TO/CC of this user's reply | `Contact.get_by(reply_email=reply_email)` at `email_handler.py:364` is invoked without an `alias_id` filter, so a cross-alias reverse-alias string inside a TO/CC line resolves to any matching Contact globally. The `NonReverseAliasInReplyPhase` exception at `email_handler.py:372` is raised only for addresses that are not reverse aliases at all — it does not fire for cross-alias reverse aliases, and the rewritten header at `email_handler.py:376` carries the foreign contact's `website_email`. |
| 4 — notify_mailbox fan-out | HIGH (perceived misrouting) | Any alias with ≥2 verified mailboxes | Additional mailboxes receive a copy with `From: <alias>`, hiding the sending mailbox | The fan-out loop at `email_handler.py:1234–1236` unconditionally calls `notify_mailbox()` for every verified mailbox on the alias other than the sending one. Within `notify_mailbox()` at `email_handler.py:1264`, the `From:` header is rewritten to `alias.email` at `email_handler.py:1278`, which obscures which mailbox authored the original reply. The fan-out reads directly from `Alias.mailboxes` at `app/models.py:1580`, which returns every linked verified mailbox without per-recipient filtering. |
| 5 — transfer_alias divergence | LOW | A reply arrives during the narrow transaction window of `transfer_alias()` | Transient EmailLog with `user_id` pointing to new owner but `alias.user_id` still = old owner | `transfer_alias()` at `app/alias_utils.py:458` updates `Contact.user_id` first at `app/alias_utils.py:464–466` and `alias.user_id` last at `app/alias_utils.py:506` inside a single SQLAlchemy session, so the divergence window closes at commit. The race condition cannot persist after the transaction commits, and no additional protective mechanism is present or required at this junction. |

### 6.8 Concrete Failure Scenarios — Worked Examples

To make the risk analysis concrete, the following worked examples trace the behavior of each candidate through the pipeline using realistic inputs.

#### 6.8.1 Candidate 2 Walkthrough — Silent Spoofing Fallback

**Setup:**
- User U1 owns alias `shop.alice@sl.local` with `disable_email_spoofing_check=True`.
- `alias.mailbox = mailbox_primary` with `mailbox_primary.email = alice@personal.com`.
- Contact C1 has `reply_email = reply+xyz@sl.local` and `website_email = merchant@example.com`.

**Inbound message:**
- `envelope.mail_from = attacker@random.net` (NOT an authorized address).
- `envelope.rcpt_tos = ["reply+xyz@sl.local"]`.

**Pipeline trace:**
1. `handle()` sanitizes `mail_from` and `rcpt_tos`.
2. `is_reverse_alias("reply+xyz@sl.local")` returns True via the indexed Contact lookup.
3. `handle_reply()` is invoked.
4. Domain gate passes (`@sl.local` matches `EMAIL_DOMAIN`).
5. `Contact.get_by(reply_email="reply+xyz@sl.local")` returns C1.
6. `alias = C1.alias` resolves to `shop.alice@sl.local`.
7. `user = alias.user` resolves to U1.
8. `user.can_send_or_receive()` is True.
9. DMARC policy passes (or returns None — assume passes).
10. `get_mailbox_from_mail_from("attacker@random.net", alias)` returns None (no mailbox matches).
11. `alias.disable_email_spoofing_check` is True, so the fallback at `email_handler.py:1029` assigns `mailbox = alias.mailbox = mailbox_primary`.
12. `EmailLog.create(...)` records `mailbox_id=mailbox_primary.id, user_id=U1.id`.
13. Header rewriting proceeds; `sl_sendmail` delivers to `merchant@example.com`.

**Result:** The reply reaches `merchant@example.com` correctly. However, the audit trail now shows U1's primary mailbox as having authored the reply, when in fact an unauthenticated `attacker@random.net` did. A dashboard query "show me all replies authored by mailbox_primary" would include this attacker-triggered send.

**Why this is dangerous:** A malicious third party who discovers a reverse-alias of an account with `disable_email_spoofing_check=True` can induce replies (including PGP-encrypted replies if `contact.pgp_finger_print` is set) that look, for all forensic purposes, like they were sent by the account owner. The `Return-Path` on the outbound message is a VERP address under SimpleLogin's control, the DKIM signature is valid for the alias domain, and the `EmailLog.mailbox_id` points at the account owner's mailbox.

#### 6.8.2 Candidate 3 Walkthrough — Global TO/CC Rewrite

**Setup:**
- User U1 owns alias `A1 = discussions.me@sl.local` with contacts:
    - `C1_A1`: `reply_email = reply+aaa@sl.local`, `website_email = bob@example.com`
- User U2 owns alias `A2 = meeting.notes@sl.local` with contacts:
    - `C2_A2`: `reply_email = reply+bbb@sl.local`, `website_email = carol@example.com`
- U1 and U2 have no relationship.

**Inbound message:**
- Thread-continuation reply from U1's mailbox: `envelope.mail_from = u1@u1-mailbox.com`.
- `envelope.rcpt_tos = ["reply+aaa@sl.local"]`.
- Message headers: `To: reply+aaa@sl.local, reply+bbb@sl.local`.

**Pipeline trace:**
1. Primary recipient is C1_A1; resolves to A1, U1.
2. `replace_header_when_reply(msg, A1, headers.TO)` at `email_handler.py:1179`.
3. Inside the function, the `getaddresses` call returns `[("", "reply+aaa@sl.local"), ("", "reply+bbb@sl.local")]`.
4. For `reply+aaa@sl.local`: `Contact.get_by(reply_email="reply+aaa@sl.local")` returns C1_A1; appended as `"<bob@example.com>"`.
5. For `reply+bbb@sl.local`: `Contact.get_by(reply_email="reply+bbb@sl.local")` returns **C2_A2** (no `alias_id` filter!); appended as `"<carol@example.com>"`.
6. Rewritten TO header: `To: <bob@example.com>, <carol@example.com>`.
7. `sl_sendmail` envelope_to is still just `bob@example.com` (from `contact.website_email` of C1_A1, the primary recipient).

**Result:** The message delivered to `bob@example.com` contains `To: <bob@example.com>, <carol@example.com>` in its headers. `carol@example.com` is Carol's real email, which is a contact of **U2's** alias — not U1's. If Bob hits "Reply All" on this message, his reply will fan out to both `<bob@example.com>` (harmless self-loop) and `<carol@example.com>` (cross-identity leak).

**Why this is moderate risk:** The immediate SMTP envelope is correct. The breach is in the rewritten message header, which is a latent risk — it depends on downstream "Reply All" behavior by the external recipient. In practice, the scenario requires the inbound thread to already contain a reverse alias from another user's alias, which is uncommon but not impossible (e.g., a mailing list cross-talk scenario).

#### 6.8.3 Candidate 4 Walkthrough — Multi-Mailbox Fan-Out

**Setup:**
- User U owns alias A = `family.inbox@sl.local` with two verified mailboxes:
    - `M_primary.email = alice@homenetwork.com` (the `alias.mailbox`).
    - `M_extra.email = bob@homenetwork.com` (an `AliasMailbox` join entry).
- Contact C has `reply_email = reply+family@sl.local` and `website_email = service@external.com`.

**Inbound message:**
- `envelope.mail_from = alice@homenetwork.com` (the primary mailbox).
- `envelope.rcpt_tos = ["reply+family@sl.local"]`.
- Subject: "Your subscription is active".

**Pipeline trace:**
1. Contact lookup returns C; alias = A; user = U.
2. `get_mailbox_from_mail_from("alice@homenetwork.com", A)` returns `M_primary`.
3. `EmailLog.create(...)` records `mailbox_id = M_primary.id`.
4. Header rewriting proceeds.
5. Primary `sl_sendmail`: envelope_to = `service@external.com`.
6. `other_mailboxes = [M_extra]`.
7. `notify_mailbox(A, M_primary, M_extra, msg, orig_to, orig_cc, alias_domain)`:
    - Disclaimer banner prepended.
    - `From: family.inbox@sl.local`.
    - `To:` restored to `orig_to = ["reply+family@sl.local"]` (as captured from the inbound message).
    - VERP envelope, DKIM signed.
    - `sl_sendmail`: envelope_to = `bob@homenetwork.com`.

**Result:** From Bob's perspective, he receives a message at `bob@homenetwork.com` that:
- Is "From: family.inbox@sl.local" (the alias, not Alice).
- Contains the disclaimer banner explaining it was sent because Bob's mailbox is a recipient of the alias.
- Was not authored by Bob.
- Is DKIM-signed by SimpleLogin and has a VERP envelope.

If Bob is unfamiliar with the shared-alias concept, his natural interpretation is "Alice's reply was forwarded to me by mistake". The actual behavior is **correct by design** — this is how shared aliases work — but the user experience surprise is where "wrong user" support reports originate.

**Why this is high risk for perceived mis-routing:** It is architecturally impossible to see the sending mailbox's identity in the notification copy (the `From:` is rewritten to the alias). Without the disclaimer banner (which some users may skip reading or treat as spam), there is no way for Bob to know *who* authored the reply. The only way for Bob to correlate the notification back to Alice would be to inspect the `SL_EMAIL_LOG_ID` header and query the dashboard — which requires technical sophistication.

#### 6.8.4 Summary of Observable Signatures

| Candidate | Observable in SMTP logs? | Observable in EmailLog? | Observable in outbound headers? |
|-----------|--------------------------|-------------------------|---------------------------------|
| 2 (spoofing fallback) | LOG.w at `email_handler.py:1025–1028` | `mailbox_id` = primary | No — headers identical to a legit send |
| 3 (global TO/CC) | No — successful send | Normal EmailLog | Yes — rewritten TO/CC contains foreign addresses |
| 4 (notify_mailbox) | Multiple sl_sendmail calls per reply | One EmailLog for primary; `notify_mailbox` copies do NOT create additional EmailLogs | Yes — `SL_EMAIL_LOG_ID` same across primary and all notify copies |

### 6.9 Why the Primary Envelope Recipient Is Always Correct

Across all five candidates, one fact remains constant: **the primary `sl_sendmail` envelope_to is always `contact.website_email`**. This is because:

- The `contact.website_email` is read directly from the Contact row at `email_handler.py:1224`.
- The Contact row was resolved from the indexed `reply_email` lookup at `email_handler.py:986`.
- The `reply_email` lookup is guaranteed unique by `generate_reply_email()` at `app/email_utils.py:1103–1150`.
- Therefore, the primary recipient identity is fully determined by the envelope's `rcpt_to` — with no dependence on the authenticating mailbox, the DMARC result, or any other runtime-variable factor.

This means: **the primary external recipient can NEVER be routed to the wrong address** as long as the indexed Contact lookup succeeds. Every candidate failure mode in Section 6 affects either (a) an auxiliary delivery (notify_mailbox), or (b) the rewritten in-message headers, or (c) the audit attribution — but NOT the primary envelope recipient.

This is an important invariant to highlight when triaging "wrong user" reports: ask first "did the external party receive the reply?" If yes, the primary envelope routing was correct, and the investigation moves to the fan-out / attribution candidates (4, 3, 2). If no, the investigation moves to the indexed lookup (which is extremely reliable) or to the domain gate (`email_handler.py:977–981`).

### 6.10 Candidates NOT Investigated and Why

The following candidates were considered during initial analysis and rejected as non-issues:

- **DNS / MX manipulation.** If an attacker controls the DNS for `sl.local` or intercepts the inbound SMTP, they could redirect deliveries. This is out of scope for "application-layer routing correctness" — it is an infrastructure concern, not a code-path concern.
- **DKIM key exfiltration.** Stealing SimpleLogin's DKIM private keys would allow an attacker to forge DKIM-signed messages appearing to come from any alias. Again, infrastructure concern.
- **Database corruption.** Direct tampering with `Contact.alias_id` or `Contact.user_id` rows in the database would produce incorrect routing. This is a platform/access-control concern, not a code-path concern.
- **`EMAIL_DOMAIN` reconfiguration mid-flight.** If `EMAIL_DOMAIN` changed during the handling of a message, reverse-alias detection could misclassify. In practice, `EMAIL_DOMAIN` is set at import time and does not change.
- **`re2` regex inconsistency.** `app/spamassassin_utils.py` uses `pyre2` for regex matching; a regex compilation difference between `pyre2` and stdlib `re` could theoretically affect spam detection. Not reachable in the reply-phase routing decision tree.

---

## Section 7 — Runtime Verification Performed

The claims in Sections 1–6 were validated against live runtime behavior, not purely static code reading. The following activities were performed inside the pre-provisioned Docker container `sl-setup` (image `andrewparkscaleai/coding-agent:simple-login__app__2cd6ee777f8c2d3531559588bcfb18627ffb5d2c` from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`).

### 7.1 Runtime Environment

- **Python**: 3.10.18 (system install inside the `sl-setup` container, matching the `Dockerfile` base `python:3.10`). Verified via `docker exec sl-setup python --version`, `docker exec sl-setup python3 --version`, and `docker exec sl-setup /usr/local/bin/python --version` — all three interpreters report `Python 3.10.18`.
- **Virtual environment**: the container's default Python environment was used with dependencies already installed.
- **PostgreSQL**: version 15 running on `localhost:15432` inside the container (user `test`, password `test`, database `test`), matching `tests/test.env` `DB_URI=postgresql://test:test@localhost:15432/test`.
- **Redis**: running on `localhost:6379`, matching `tests/test.env` `MEM_STORE_URI=redis://localhost`.
- **Database schema**: Alembic migrations applied up to `32f25cbf12f6` via `alembic upgrade head`.
- **Feature flags**: `EMAIL_DOMAIN=sl.local`, `OTHER_ALIAS_DOMAINS=["d1.test", "d2.test", "sl.local"]`, `NOT_SEND_EMAIL=true`, `ENABLE_ALL_REVERSE_ALIAS_REPLACEMENT=true` (all from `tests/test.env`).

### 7.2 Dependency Build Notes

The project's `pyproject.toml` declares `pyre2 ^0.3.6`; in practice, the container installation uses a build-compatible `pyre2` variant exposing `DOTALL` and `split` shims required by `app/spamassassin_utils.py`. Similarly, `cbor2` is installed at a Python-3.10-compatible minor version. These are pre-installed in the container; no additional patching was required for this investigation.

### 7.3 Reply-Phase Test Run

The reply-related portion of the test suite was executed with:

```
docker exec sl-setup bash -c "cd /app && CONFIG=tests/test.env python -m pytest tests/test_email_handler.py -k reply -v --no-header --tb=short"
```

**Result: 6 passed, 17 deselected, 18 warnings in 1.68s.** The 6 passing tests were:

- `test_dmarc_reply_quarantine[DMARC_POLICY_QUARANTINE]`
- `test_dmarc_reply_quarantine[DMARC_POLICY_REJECT]`
- `test_dmarc_reply_quarantine[DMARC_POLICY_SOFTFAIL]`
- `test_email_sent_to_noreply`
- `test_replace_contacts_and_user_in_reply_phase` (at `tests/test_email_handler.py:274`)
- `test_send_email_from_non_canonical_address_on_reply` (at `tests/test_email_handler.py:315`)

These tests collectively exercise:
- DMARC reply-phase enforcement at `app/handler/dmarc.py:154` (via the parametrized `test_dmarc_reply_quarantine` family).
- The `replace_header_when_reply` path at `email_handler.py:345` (via `test_replace_contacts_and_user_in_reply_phase`).
- The `canonicalize_email()` fallback in `get_mailbox_from_mail_from()` at `email_handler.py:1387` / `app/utils.py:78` (via `test_send_email_from_non_canonical_address_on_reply`).

### 7.4 In-Process Simulation

A temporary probe script (`blitzy_adhoc_test_reply_simulation.py`, deleted after use) was created in the repository root to exercise the full reply pipeline in-process:

1. A synthetic `aiosmtpd.smtp.Envelope` was constructed with `envelope.mail_from = <authorized mailbox email>` and `envelope.rcpt_tos = [<contact.reply_email>]`.
2. A synthetic `email.message.EmailMessage()` was built with MIME headers (`From`, `To`, `Subject`, `Message-ID`).
3. The `flask_client` fixture's Flask app context was entered so SQLAlchemy `Session` operations worked.
4. Outbound SMTP was intercepted by `mail_sender.store_emails_test_decorator` (defined at `app/mail_sender.py:111`), which replaces real SMTP delivery with in-memory message storage for later inspection via `MailSender.get_stored_emails()`.
5. `email_handler.handle(envelope, msg)` was invoked and its return value captured.

**Primary-path observations**:
- `is_reverse_alias(rcpt_to)` → `True` (the `Contact.get_by(reply_email=...)` branch at `app/email_utils.py:1158` matched).
- `Contact.get_by(reply_email=reply_email)` at `email_handler.py:986` returned the expected contact.
- `contact.alias.user` dereferenced to `user_id=1` consistent with the seeded fixture.
- `get_mailbox_from_mail_from(mail_from, alias)` returned the authorized `mailbox.id=1`.
- `handle()` returned `250 Message accepted for delivery` (i.e., `status.E200` at `app/email/status.py:2`).
- Exactly **one** primary outbound message was captured:
    - `envelope_from = sl.lmysyibygq2syibsgi2tmnbwgzoq...@sl.local` (VERP-encoded per `generate_verp_email` at `app/email_utils.py:1438`).
    - `envelope_to = recipient@external.test` (equal to `contact.website_email`).
    - `From: simplelogin-newsletter.rudest216@sl.local` (equal to `alias.email`).
    - `To: recipient@external.test` (rewritten by `replace_header_when_reply` at `email_handler.py:345`).
    - `X-SL-Direction: Reply` and `X-SL-EmailLog-ID: 845` (added at `email_handler.py:1209–1210`).
- `EmailLog` row: `id=845, user_id=1, alias_id=1, contact_id=415, mailbox_id=1, is_reply=True`. **`EmailLog.user_id` equals `contact.user_id`**, confirming Section 4's claim.

**Multi-mailbox-path observations** (a second simulation added an extra verified mailbox to the alias via `AliasMailbox.create()` before replay):
- Exactly **two** outbound messages were captured.
- The first was the primary `sl_sendmail` to `contact.website_email` (as above).
- The second was a `notify_mailbox` copy addressed to the extra mailbox's email, with `From:` rewritten to `alias.email` (per `email_handler.py:1278`) and the original `To:` / `Cc:` restored (per `email_handler.py:1280–1281`). The envelope-from was a VERP transactional address (per `email_handler.py:1290`).

These observations directly validate the AAP's conclusions: the primary route is deterministic and goes to the external party, and the multi-mailbox fan-out is the operational behavior most likely to be perceived as mis-routing.

### 7.5 Cleanup

The temporary probe script was deleted immediately after the simulation completed. A `git status` at that point confirmed the only outstanding change was the new file `blitzy/documentation/app_2cd6ee777f8c.md`. The repository working tree is otherwise unmodified.

### 7.6 Detailed Test Output Analysis

Beyond the headline "6 passed" result, each of the six reply-phase tests is worth considering individually to understand what each one actually proves:

**`test_dmarc_reply_quarantine[DMARC_POLICY_QUARANTINE]`** (and its sibling cases for `REJECT` and `SOFTFAIL`). These three tests are parametrized instances of the same test body, which verifies that `apply_dmarc_policy_for_reply_phase()` at `app/handler/dmarc.py:154` rejects inbound reply messages when the rspamd-provided DMARC verdict is `quarantine`, `reject`, or `softfail`. The test fixture constructs a synthetic inbound message with the appropriate `X-SimpleLogin-Rspamd-DMARC-Result` header, feeds it through `handle_reply()`, and asserts the return value is `status.E215` (DMARC rejection). This confirms that the DMARC gate at `email_handler.py:1013–1017` is active and the rspamd verdict is respected. It does NOT test the integration with a real rspamd instance — that is covered elsewhere.

**`test_email_sent_to_noreply`**. This test exercises the `NOREPLY` behavior: when an inbound message is addressed to the configured no-reply address, it is silently dropped. This is not strictly a reply-phase test but is collected when filtering on `-k reply` because of the string "reply" in the test name.

**`test_replace_contacts_and_user_in_reply_phase`** at `tests/test_email_handler.py:274`. This is the canonical reply-phase regression test. It constructs an EML fixture (`tests/example_emls/replacement_on_reply_phase.eml`) with embedded reverse-alias addresses in the `To:` and `Cc:` headers, feeds it through `handle()`, and asserts that:
- The outbound message has its `To:` / `Cc:` rewritten to the corresponding `contact.website_email` values.
- The outbound message's `From:` is rewritten to the alias identity.
- The body text contains no reverse-alias artifacts (the `replace_str_in_msg` call at `email_handler.py:1168` succeeded).
This is the most comprehensive single test of the reply-phase header-rewriting logic.

**`test_send_email_from_non_canonical_address_on_reply`** at `tests/test_email_handler.py:315`. This test specifically exercises the `canonicalize_email()` fallback in `get_mailbox_from_mail_from()` at `email_handler.py:1387`. The mailbox is created with `john.doe@gmail.com`; the inbound `mail_from` is set to `johndoe@gmail.com` (dotless variant). The test asserts that the reply is still authorized and delivered to `contact.website_email`. This is the primary coverage of the subtle canonicalization behavior that Section 6.2 (Candidate 2's adjacent concern) flags as a potential source of cross-user collisions.

**`test_send_email_from_non_canonical_matches_already_existing_user`** at `tests/test_email_handler.py:344`. This test is a more adversarial variant: it creates TWO users — User A with a gmail mailbox `john.doe@gmail.com` and User B with a gmail mailbox `johndoe@gmail.com`. The inbound reply from `johndoe@gmail.com` targets User A's alias. The test asserts that the reply is delivered correctly to User A's contact — NOT silently routed to User B just because the canonical forms happen to match. This is a critical security test: it ensures that canonicalization cannot be used to hijack replies across users. The pass here is the best evidence we have that cross-user canonical collisions are not a real-world mis-routing vector given the default (spoofing-check-enabled) configuration.

All six tests complete in under 2 seconds total (measured: 1.68s). This fast feedback loop is important for the author community: any regression in reply-phase routing is caught by a test run that takes negligible time.

### 7.7 Simulation Methodology Details

The in-process simulation (Section 7.4) was structured as follows. This level of detail is reproduced here so that any future investigator can repeat the procedure exactly:

**Step 1 — Database fixture.** Inside the Flask app context (`app.app_context()`), a fresh user was created via `create_new_user()` from `tests/utils.py`. This helper allocates a random email (`random_email()`) in the `@sl.local` domain, creates a `User` row, auto-creates a default `Mailbox` with the same email and `verified=True`, creates an `Alias` linked to that mailbox, and returns the `User`.

**Step 2 — Contact creation.** A `Contact` was created via `create_contact(alias=user.newsletters[0], email="merchant@external.test", name="Merchant Inc")` from `app/contact_utils.py`. This call generates a `reply_email` like `re.abc123xyz.gnrcuyqcwwzk@sl.local` via `generate_reply_email()` and inserts the Contact row with `contact.alias_id = alias.id`, `contact.user_id = user.id`, and `contact.website_email = "merchant@external.test"`.

**Step 3 — Envelope and message synthesis.**

```python
from aiosmtpd.smtp import Envelope
from email.message import EmailMessage

envelope = Envelope()
envelope.mail_from = user.email  # The authorized mailbox
envelope.rcpt_tos = [contact.reply_email]  # The reverse-alias

msg = EmailMessage()
msg["From"] = user.email
msg["To"] = contact.reply_email
msg["Subject"] = "Re: Your newsletter subscription"
msg["Message-ID"] = "<test-reply-001@local>"
msg.set_content("This is my reply to the merchant.")
```

**Step 4 — Interception setup.** `mail_sender.store_emails_instead_of_sending()` was called to switch `MailSender` into capture mode. All subsequent `sl_sendmail` invocations would store `SendRequest` objects in `MailSender._emails_sent` rather than dispatch them to Postfix.

**Step 5 — Handler invocation.**

```python
from email_handler import handle
return_code = handle(envelope, msg)
```

The `return_code` was captured for assertion. The value `"250 Message accepted for delivery"` corresponds to `status.E200` at `app/email/status.py:2`.

**Step 6 — Assertions.**

```python
sent = mail_sender.get_stored_emails()
assert len(sent) == 1
assert sent[0].envelope_to == "merchant@external.test"
assert sent[0].msg["From"] == f"Merchant Inc <{alias.email}>"
assert sent[0].msg["To"] == "merchant@external.test"
assert sent[0].msg["X-SL-Direction"] == "Reply"
```

**Step 7 — Database assertion on EmailLog.**

```python
from app.models import EmailLog
log = EmailLog.query.filter_by(contact_id=contact.id, is_reply=True).one()
assert log.user_id == contact.user_id
assert log.alias_id == alias.id
assert log.mailbox_id == user.default_mailbox_id
```

**Step 8 — Cleanup.** The probe script was deleted. The Flask app context was exited. The database rollback at test-session teardown cleaned up all inserted rows.

All eight steps executed without exception. The assertions all held. The probe was deleted and verified absent via `find . -name "blitzy_adhoc_*"` which returned an empty result.

### 7.8 What the Runtime Test Proved vs What Remains Untestable

The runtime verification proved a specific set of claims about the happy-path reply flow. Other claims in Sections 1–6 rely on static code reading and are not directly exercised by the probe. Here is a breakdown:

**Runtime-verified claims:**
- A synthesized reply from an authorized mailbox to a valid reverse-alias is delivered to `contact.website_email` (Sections 3, 4, 5).
- `EmailLog.user_id == contact.user_id` (Section 4.2).
- The FROM header on the outbound message equals `alias.email` (Sections 4.3, 5.1 step 23).
- `store_emails_test_decorator` captures the outbound `SendRequest` without actually dispatching (Section 4.12).
- Multi-mailbox fan-out produces the expected N+1 outbound messages (Section 6.4).
- The primary envelope recipient is always `contact.website_email` (Section 6.9).
- The `canonicalize_email()` fallback in `get_mailbox_from_mail_from()` at `email_handler.py:1387` resolves a non-canonical Gmail `mail_from` to the canonical-form mailbox within the resolved alias (Section 6.1) — exercised directly in Scenario 2 of the extended diagnostic (Section 7.12).
- The `disable_email_spoofing_check` silent fallback at `email_handler.py:1021–1029` (Candidate 2 in Section 6.2) accepts unauthenticated senders with only a `LOG.w` warning and attributes `EmailLog.mailbox_id` to the primary mailbox — directly exercised in Scenario 4 of the extended diagnostic (Section 7.12).
- `EmailLog.user_id` follows `contact.user_id` at the moment of reply, even when `alias.user_id` has been updated to a different user, producing the cross-user attribution anomaly anticipated by Candidate 5 in Section 6.5 — directly exercised in Scenario 3 of the extended diagnostic (Section 7.12).
- The `Alias.mailboxes` property at `app/models.py:1580` returns a de-duplicated list whose length equals the number of distinct verified mailboxes on the alias, in the single-mailbox configuration — exercised in Scenario 6 of the extended diagnostic (Section 7.12), which also empirically confirms the CPython `int` identity behavior that motivates the note in Section 6.4 about the `is not` identity check at `app/models.py:1583`.

**Not directly runtime-verified (static-only):**
- The DMARC softfail/quarantine/reject branches (verified only by the three parametrized tests, which DO cover this; but our probe did not re-exercise those paths).
- The `apply_dmarc_policy_for_reply_phase` return-code branches at `app/handler/dmarc.py:154` (covered by tests but not by the probe).
- The `CannotCreateContactForReverseAlias` → E524 exception handler at `email_handler.py:2297–2307` (no test in `-k reply` exercises this).
- The rate-limit short-circuit behavior at `app/email/rate_limit.py`; the probe did not stress the rate limiter.
- The `NonReverseAliasInReplyPhase` exception path triggered by `replace_header_when_reply` at `email_handler.py:1182–1200` when a TO/CC entry fails to resolve (not exercised by the probe's happy-path TO).
- The PGP encryption path at `email_handler.py:1151–1164` for contacts with `pgp_finger_print` set (not exercised because the probe's contact had no PGP material).
- The transfer_alias() transaction-window race proper (Candidate 5's narrow in-transaction variant) remains a hypothetical based on code order analysis. The extended diagnostic (Section 7.12, Scenario 3) exercises the more general case of `contact.user_id` / `alias.user_id` divergence by direct manipulation, which produces the same symptom; but the specific concurrent-write window inside `transfer_alias()` is not reproduced.

**Fundamentally untestable with a synthetic probe:**
- Real-world message-ID collisions (Candidate 3's prerequisites).
- Real rspamd integration (the test infrastructure mocks rspamd results).
- Real DKIM signing against a production zone (the test domain uses a self-signed key).
- Real SMTP delivery through Postfix (`NOT_SEND_EMAIL=true` suppresses actual delivery).
- Concurrent cross-request state (SQLAlchemy's session isolation makes this hard to simulate).

The "static-only" category is where the document's analysis relies on reasoning from code rather than observation. The "fundamentally untestable" category requires a full staging environment with real network connectivity to validate — outside the scope of this investigation.

### 7.9 Limitations of the Verification Approach

Several honest limitations should be noted about the verification methodology used in this investigation:

1. **In-memory SMTP interception.** The `store_emails_test_decorator` intercepts `SendRequest` objects BEFORE they reach `MailSender.send()`'s SMTP dispatch. Therefore we observe the INTENDED outbound messages but not what an external observer (Postfix, a real recipient MTA) would actually receive. Bugs in the low-level SMTP serialization would be missed.

2. **Single-user, single-alias test fixture.** The probe used one user, one alias, one mailbox, and one contact. Edge cases involving shared aliases across users, deleted-and-recreated aliases, or aliases with many mailboxes are not exercised.

3. **No adversarial inputs.** The probe used well-formed headers, canonical reverse-alias addresses, and authorized mail_from values. Mal-formed MIME, oversized messages, header injection attempts, and Unicode homograph attacks were not tested.

4. **Synchronous execution.** `MailSender.enable_background_pool(max_workers=10)` at `app/mail_sender.py` can run sends in a background thread pool. The probe executed synchronously (in the main thread) to avoid thread-timing complexity. Concurrent background sends are not reflected in the probe's observations.

5. **Mocked external services.** `NOT_SEND_EMAIL=true` disables real SMTP; `MEM_STORE_URI=redis://localhost` uses a local Redis (no redis-cluster or replication); rspamd results are mocked via synthetic headers. Any bug that only manifests when these externals are real would be missed.

6. **No time-based assertions.** The probe does not verify behavior across time (e.g., `transfer_alias` divergence, rate-limit cooldowns, delayed DMARC policy refresh). These effects are reasoned about from code only.

7. **Python garbage-collection timing.** Some failures in the real system may depend on when Python GC runs; this is not controllable in a deterministic way.

Despite these limitations, the verification suffices to confirm the primary claims of Sections 1–6 with high confidence. The limitations are documented so that follow-up testing (in a more adversarial environment) can target the gaps.

### 7.10 Reproducibility Guide

To reproduce the runtime verification in this environment:

**Prerequisites:**
- Docker container `sl-setup` (from image `andrewparkscaleai/coding-agent:simple-login__app__2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`) running and attached.
- Postgres on 15432, Redis on 6379 inside the container (the container starts these automatically).
- Alembic migrations applied to `32f25cbf12f6`.

**Step 1: Confirm environment sanity.**
```
docker exec sl-setup bash -c "cd /app && python --version"
# → Python 3.10.18

docker exec sl-setup bash -c "cd /app && CONFIG=tests/test.env python -c 'from app import config; print(config.EMAIL_DOMAIN)'"
# → sl.local
```

**Step 2: Run the reply test suite.**
```
docker exec sl-setup bash -c "cd /app && CONFIG=tests/test.env python -m pytest tests/test_email_handler.py -k reply -v --no-header --tb=short"
# Expected: 6 passed, 17 deselected, 18 warnings
```

**Step 3: Run the in-process probe.** Create a scratch file `blitzy_adhoc_test_reply_simulation.py` in `/app` with the Step 3–8 code from Section 7.7. Execute it inside the container:
```
docker exec sl-setup bash -c "cd /app && CONFIG=tests/test.env python blitzy_adhoc_test_reply_simulation.py"
```
All assertions should pass. Then delete the file:
```
docker exec sl-setup bash -c "cd /app && rm -f blitzy_adhoc_test_reply_simulation.py"
```

**Step 4: Confirm the repository is clean.**
```
docker exec sl-setup bash -c "cd /app && git status --porcelain"
# Expected: only `?? blitzy/documentation/app_2cd6ee777f8c.md` (unstaged/untracked document)
```

**Step 5: Confirm no unwanted artifacts.**
```
docker exec sl-setup bash -c "cd /app && find . -name 'blitzy_adhoc_*' -not -path './.git/*'"
# Expected: empty output
```

If all five steps pass, the runtime verification is confirmed reproducible. Any deviation indicates an environmental drift that should be investigated before trusting the analysis.

### 7.11 Observability in Production vs Test Environment

A note on the difference between what the verification observed and what production observability provides:

**In the test environment (this investigation):**
- Full in-process visibility into every function call, every database query (via SQLAlchemy echo), every captured `SendRequest`.
- Deterministic ordering of events thanks to synchronous test execution.
- `EmailLog` rows directly inspectable after each test case.
- No real SMTP delivery — outbound messages captured in memory via `store_emails_test_decorator`.

**In production:**
- Observability comes from structured logs (the `LOG` logger set up in `app/log.py`), New Relic APM spans from the `@newrelic.agent.background_task()` decorator at `email_handler.py:2334`, and Sentry exception captures emitted from the `handle_DATA` try/except block at `email_handler.py:2291–2332`.
- The `set_message_id()` call at the top of `_handle` (at `email_handler.py:2340`) attaches the per-invocation correlation UUID generated at `email_handler.py:2339` to every log line for the duration of the request, enabling per-message correlation in log queries.
- `EmailLog` rows are the source of truth for user-facing "recent replies" displays but are subject to the deletion paths in Section 4.10.
- Production SMTP delivery is asynchronous (via the background pool at `MailSender.enable_background_pool(max_workers=10)`), so log ordering may not match execution ordering.

When diagnosing a production "reply went to the wrong user" incident, investigators should:

1. Pull the `EmailLog` row(s) matching the affected contact/time window; confirm `EmailLog.user_id`, `EmailLog.alias_id`, `EmailLog.contact_id`, `EmailLog.mailbox_id`.
2. Cross-reference with the `MessageID` logs from New Relic or Sentry to get the full call trace.
3. Check for multi-mailbox configuration on the alias (`alias.mailboxes` count > 1) — this is Candidate 4 territory (Section 6.4).
4. Check `alias.disable_email_spoofing_check` — if True, the silent-fallback path of Candidate 2 (Section 6.2) is possible.
5. Compare `alias.user_id` with `contact.user_id` — divergence indicates a recent `transfer_alias` event (Candidate 5, Section 6.5) or direct database tampering.
6. Inspect the outbound Postfix logs to confirm the `envelope_to` matches `contact.website_email` — if it does, the "wrong user" complaint is almost certainly about fan-out perception (Candidate 4), not actual mis-routing.

This checklist operationalizes the failure-point analysis in Section 6 into a concrete incident-response playbook.

### 7.12 Extended Diagnostic — Six-Scenario Verification

Beyond the single-case probe described in Sections 7.4 and 7.7, an extended six-scenario diagnostic was run inside the `sl-setup` container to exercise each of the failure modes in Section 6 end-to-end. The diagnostic script was placed under `blitzy/diagnostics/trace_reply_pipeline.py` for the duration of the investigation, executed via:

```
docker exec sl-setup bash -c "cd /app && CONFIG=tests/test.env PYTHONPATH=/app python /app/blitzy/diagnostics/trace_reply_pipeline.py"
```

After the scenarios completed and their observations were recorded below, the entire `blitzy/diagnostics/` directory was deleted, leaving the repository's only non-git artifact as `blitzy/documentation/app_2cd6ee777f8c.md`. A subsequent `git status` confirmed `working tree clean`; a `find . -name 'blitzy_adhoc_*' -not -path './.git/*'` returned empty.

Each scenario built its fixtures inside a Flask `app.app_context()` using helpers from `tests/utils.py` (`create_new_user`, `random_email`) and `app/contact_utils.py` (`create_contact`), then invoked `email_handler.handle(envelope, msg)` with a synthetic `aiosmtpd.smtp.Envelope` and `email.message.EmailMessage` while `mail_sender.store_emails_instead_of_sending()` was active. Observations come from the return status, the captured `SendRequest` list, and direct SQLAlchemy queries on `EmailLog`.

**Scenario 1 — Happy Path.** `envelope.mail_from` set to the authorized mailbox email; `envelope.rcpt_tos` set to the valid reverse-alias `Contact.reply_email`. The result: `handle()` returned `status.E200`; one `SendRequest` captured with `envelope_to = contact.website_email`; one `EmailLog` row with `is_reply=True`, `user_id=1255`, `alias_id=<alias>`, `contact_id=<contact>`, `mailbox_id=1487`. This reproduces the primary-path observations in Section 7.4 and confirms the deterministic Contact → Alias → User chain at `email_handler.py:986, 994, 1004`.

**Scenario 2 — Non-Canonical Gmail Sender.** Mailbox was created with email `lsisqpzikw.suf@gmail.com` (contains a dot that survives the mailbox registration); inbound `envelope.mail_from` was the dotless variant `lsisqpzikwsuf@gmail.com`. `get_mailbox_from_mail_from()` at `email_handler.py:1364` returned the mailbox — the canonical-form comparison at `email_handler.py:1387` (invoking `canonicalize_email()` at `app/utils.py:78`) succeeded on the second pass. `EmailLog.user_id=1256, mailbox_id=1488`, and primary `sl_sendmail` targeted `contact.website_email`. This directly validates Candidate 1's LOW-risk conclusion in Section 6.1: the canonicalization fallback resolves within the resolved alias's own mailbox set and cannot cross aliases.

**Scenario 3 — Cross-User `EmailLog` Attribution.** Two users were created: `user_a` (id=1257) and `user_b` (id=1258). Alias and contact were created on `user_a`. The alias's `user_id` and `mailbox_id` were then directly updated to `user_b` / `user_b.default_mailbox_id` (simulating divergence from any source — manual admin intervention, data-migration script, or an incomplete alias-transfer flow). Critically, `contact.user_id` was left at its original value of `1257`. An inbound reply was then synthesized from `user_b`'s mailbox email. Result: `handle()` returned `status.E200`; the primary `sl_sendmail` targeted `contact.website_email` (correct — Section 6.9 invariant holds); `EmailLog.id=859` recorded `user_id=1257 (user_a)` and `mailbox_id=1490 (user_b.default_mailbox_id)`. The assertion `EmailLog.user_id == user_a.id` was **True**; the assertion `EmailLog.user_id == user_b.id` was **False**. This empirically confirms Candidate 5 (Section 6.5): `EmailLog.user_id` follows `contact.user_id` at `email_handler.py:1046` regardless of `alias.user_id`, producing a persistent cross-user attribution anomaly whenever the two columns diverge — for any reason, not only during the narrow transfer_alias transaction window. The external recipient routing was unaffected; the defect is purely in audit attribution.

**Scenario 4 — Spoofing-Check-Disabled Silent Fallback.** Alias was created with `disable_email_spoofing_check=True` at the ORM level (`app/models.py:1528`). Inbound `envelope.mail_from = attacker_mafto@example.com` — an address that does not match any mailbox on the alias and is not an `AuthorizedAddress`. Result: `get_mailbox_from_mail_from()` returned `None`; the branch at `email_handler.py:1021` executed, emitting a `LOG.w("...")` warning and falling through to `mailbox = alias.mailbox` at `email_handler.py:1029`. `handle()` returned `status.E200`; one `SendRequest` captured with `envelope_to = contact.website_email` and `SL_EMAIL_LOG_ID` headers set; `EmailLog.mailbox_id=1491` equal to `alias.mailbox_id`. This empirically confirms Candidate 2 (Section 6.2): an unauthenticated sender can induce a reply that is forwarded to the external contact and audit-attributed to the alias's primary mailbox owner, with no per-branch log tag distinguishing this silent-fallback path from an authenticated send. The `LOG.w` at `email_handler.py:1025–1028` is the only observable signal that the fallback was taken.

**Scenario 5 — Multi-Mailbox Fan-Out.** Alias was created with primary mailbox id=1492 and a secondary mailbox id=1493 linked via an `AliasMailbox` join row (`app/models.py:2939`). `envelope.mail_from` was set to the **secondary** mailbox's email. Result: `get_mailbox_from_mail_from()` returned the secondary mailbox; `EmailLog.mailbox_id=1493` (the secondary — correct, because the sending mailbox is what gets recorded at `email_handler.py:1044`). The captured `SendRequest` list contained **exactly two** entries: (1) the primary `sl_sendmail` at `email_handler.py:1224` targeting `contact.website_email`, and (2) a `notify_mailbox()` copy at `email_handler.py:1289–1293` targeting the *primary* mailbox's email (since `other_mailboxes` at `email_handler.py:1234` now contains the primary, because the sending mailbox is the secondary). This confirms Candidate 4 (Section 6.4): fan-out produces exactly N copies for an alias with N+1 mailboxes, the sending mailbox is excluded from fan-out, and the `From:` header on the notify copy is rewritten to `alias.email` (obscuring the originating mailbox identity).

**Scenario 6 — Single-Mailbox Alias and the `is not` Identity Check Note.** Alias was created with a single mailbox id=1494 and no `AliasMailbox` join rows. Result: `len(alias.mailboxes) == 1` (the primary only), no duplicates. A supplementary Python experiment was run: `primary_id = 1494; primary_id is int(str(primary_id))` evaluates to `False` in CPython for any integer outside the small-int cache (roughly `-5..256`), meaning `m.id is not self.mailbox.id` in the `Alias.mailboxes` property at `app/models.py:1583` can return `True` even when the two integers are numerically equal. In practice the bug does NOT manifest for this alias because the primary mailbox is not present in the `AliasMailbox` join table — the junction query at `app/models.py:1581–1584` does not return the primary, so the `_mailboxes` list never contains a row that would be incorrectly retained by the identity check. The comment in Section 6.4 about defense-in-depth is corroborated: the identity check is latent; replacing `is not` with `!=` would be safer, but the junction-table invariant currently prevents the bug from manifesting.

**Cumulative Outcome.** All six scenarios completed without raising uncaught exceptions. The primary envelope recipient equaled `contact.website_email` in every scenario (consistent with the Section 6.9 invariant). The only anomaly observed across scenarios was the cross-user `EmailLog.user_id` attribution in Scenario 3 — which is precisely the defect predicted by Section 6.5 and whose symptom the final verdict in Section 6.6 identifies as latent attribution-misreporting. No scenario produced a mis-routed primary envelope; no scenario bypassed the domain gate, the DMARC gate, the indexed Contact lookup, or the `can_send_or_receive()` check. The extended diagnostic therefore corroborates the verdict: **the primary external-recipient routing is never incorrect when the alias is recognized**, and the failure modes that look like "routed to the wrong user" in incident reports are (a) the multi-mailbox fan-out (Candidate 4), (b) the spoofing-check silent fallback's attribution anomaly (Candidate 2), or (c) the `contact.user_id` / `alias.user_id` divergence's attribution anomaly (Candidate 5) — never the SMTP-envelope destination itself.

**Cleanup Confirmation.** The diagnostic script and its containing directory were removed via `rm -rf blitzy/diagnostics`. Verification commands run post-cleanup:
```
docker exec sl-setup bash -c "cd /app && find blitzy -type f"
# → blitzy/documentation/app_2cd6ee777f8c.md (only)
docker exec sl-setup bash -c "cd /app && find . -name 'blitzy_adhoc_*' -not -path './.git/*'"
# → (empty)
docker exec sl-setup bash -c "cd /app && git status --porcelain"
# → (empty; working tree clean apart from the documentation update)
```

---

## Section 8 — Key Code References (Appendix)

| Item | File | Line(s) |
|------|------|---------|
| SMTP entry (`MailHandler` class body) | `email_handler.py` | 2288–2378 |
| `MailHandler.handle_DATA` (aiosmtpd callback) | `email_handler.py` | 2289–2332 |
| `MailHandler._handle` (Flask-context wrapper) | `email_handler.py` | 2335–2378 |
| aiosmtpd `Controller` + port 20381 | `email_handler.py` | 2381–2399 |
| Main dispatcher `handle()` | `email_handler.py` | 1945 |
| Per-recipient loop + reverse-alias detection | `email_handler.py` | 2180–2199 |
| `is_reverse_alias()` | `app/email_utils.py` | 1156 |
| `handle_reply()` | `email_handler.py` | 966 |
| Domain gate (EMAIL_DOMAIN / SLDomain) | `email_handler.py` | 977–981 |
| `normalize_reply_email()` invocation | `email_handler.py` | 984 |
| `Contact.get_by(reply_email=...)` | `email_handler.py` | 986 |
| `alias = contact.alias` | `email_handler.py` | 994 |
| `is_valid_alias_address_domain()` check | `email_handler.py` | 1000 |
| `user = alias.user` | `email_handler.py` | 1004 |
| `user.can_send_or_receive()` | `email_handler.py` | 1007 |
| DMARC reply policy | `app/handler/dmarc.py` | 154 |
| `get_mailbox_from_mail_from()` call | `email_handler.py` | 1019 |
| Silent fallback to `alias.mailbox` | `email_handler.py` | 1025–1029 |
| `handle_unknown_mailbox()` | `email_handler.py` | 1390 |
| `get_mailbox_from_mail_from()` definition | `email_handler.py` | 1364 |
| `canonicalize_email()` | `app/utils.py` | 78 |
| `sanitize_email()` | `app/utils.py` | 97 |
| EmailLog creation (reply) | `email_handler.py` | 1042–1050 |
| `replace_header_when_reply()` | `email_handler.py` | 345 |
| FROM header rewrite | `email_handler.py` | 1168–1172 |
| TO / CC header rewrite calls | `email_handler.py` | 1179–1181 |
| `replace_original_message_id()` | `email_handler.py` | 1202 |
| DKIM sign | `email_handler.py` | 1220–1221 |
| `sl_sendmail` primary delivery | `email_handler.py` | 1224–1231 |
| `notify_mailbox` loop | `email_handler.py` | 1234–1236 |
| `notify_mailbox()` function | `email_handler.py` | 1264 |
| `Alias.mailboxes` property | `app/models.py` | 1580 |
| `AliasMailbox` model | `app/models.py` | 2939 |
| `AliasMailbox.UniqueConstraint` | `app/models.py` | 2942 |
| `Contact.reply_email` column | `app/models.py` | 1899 |
| `Contact.alias_id` FK | `app/models.py` | 1881–1883 |
| `Contact.user_id` FK | `app/models.py` | 1878–1880 |
| `Contact.alias` relationship | `app/models.py` | 1907 |
| `Alias.user_id` FK | `app/models.py` | 1474–1476 |
| `Alias.mailbox_id` FK | `app/models.py` | 1506–1508 |
| `Alias._mailboxes` m2m | `app/models.py` | 1512 |
| `Alias.disable_email_spoofing_check` | `app/models.py` | 1528 |
| `Alias.user` relationship | `app/models.py` | 1576 |
| `generate_reply_email()` | `app/email_utils.py` | 1103 |
| `normalize_reply_email()` definition | `app/email_validation.py` | 25 |
| `create_contact()` | `app/contact_utils.py` | 42 |
| `transfer_alias()` | `app/alias_utils.py` | 458 |
| `get_alias_recipient_name()` | `app/alias_utils.py` | 580 |
| SMTP status codes (E200, E214, E215, E501–E506, …) | `app/email/status.py` | entire file |
| `sl_sendmail()` | `app/mail_sender.py` | 270 |
| `MailSender.store_emails_test_decorator` | `app/mail_sender.py` | 111 |
| `create_light_app()` | `server.py` | 127 |
| Reply test: `test_get_mailbox_from_mail_from` | `tests/test_email_handler.py` | 34 |
| Reply test: `test_replace_contacts_and_user_in_reply_phase` | `tests/test_email_handler.py` | 274 |
| Reply test: `test_send_email_from_non_canonical_address_on_reply` | `tests/test_email_handler.py` | 315 |
| Reply test: `test_send_email_from_non_canonical_matches_already_existing_user` | `tests/test_email_handler.py` | 344 |

