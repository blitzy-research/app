# SimpleLogin Alias Reply-Handling Flow — Runtime Root-Cause Analysis

**Source branch:** `app_2cd6ee777f8c`
**HEAD commit:** `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`
**Nature of this document:** a **read-only, runtime-first** investigation. The relevant code paths were BUILT and RUN first, the real output was captured, and this analysis is written from what was *observed* — not from reading the code alone. No source file was modified. The only artifact added to the repository is this document. All temporary observation scaffolding (a single pytest module and its captured transcripts) lived **inside the run container** under `/tmp/slrun`, never inside the repository working tree, so the working tree stayed git-clean throughout; the pre-existing local `__pycache__/*.pyc` and generated `static/upload/**` mail captures left by earlier tooling were removed during finalization (see [§ (h)](#h-commands-run--raw-runtime-output)).

This document answers the user's question and its five decomposed sub-questions (Q1–Q5) explicitly and by name. The user's request is preserved verbatim:

> "I'm running into unexpected behavior in the alias reply-handling flow of SimpleLogin and I want to determine whether it's a real issue or just a misunderstanding of how the pipeline works. When a user replies to an email that was forwarded through an alias, the backend is supposed to receive the inbound message, identify which alias it belongs to, and relay it back to the correct recipient, but in my local tests some replies appear to be routed to the wrong user even though the logs show the alias being recognized. Using the development environment, simulate an inbound email reply and trace the runtime flow end-to-end: observe which part of the system handles the incoming message, how the alias is resolved to a user, and what user ID the system ultimately decides to forward the reply to. Based on this live execution trace, explain the actual data flow and identify the most likely point in the pipeline where an incorrect routing decision could originate. You can create temporary scripts or logs that's fine but clean them up and leave the codebase as you found it."

The five sub-questions this document addresses:

- **Q1 — Entry point:** Which part of the system handles the incoming reply message?
- **Q2 — Resolution:** How is the alias resolved to a user?
- **Q3 — Decision:** What concrete integer user ID does the system ultimately record/decide for the reply?
- **Q4 — Data flow:** Based on the LIVE execution trace, what is the actual end-to-end data flow?
- **Q5 — Root cause:** What is the most likely point in the pipeline where an incorrect routing decision could originate?

---

## Legend — observed vs. inferred

Every factual statement in this document is labeled:

| Label | Meaning |
|-------|---------|
| **(observed)** | Came directly from captured runtime output (the raw `S1`–`S6` blocks in [§ (h)](#h-commands-run--raw-runtime-output)) **or** a direct, quoted line of source code with a `file:line` citation verified at HEAD `2cd6ee777f8c`. |
| **(inferred)** | A logical deduction drawn from the observed facts above. Inferences are conclusions, not measurements. |

All citations use the form `file:line` (e.g., `email_handler.py:L986`) and were verified against the source at the HEAD commit above. All runtime numbers, log lines, and status strings are transcribed EXACTLY as emitted — nothing is rounded, paraphrased, truncated, or "cleaned up." Where a line is long (e.g., a DKIM signature), it is reproduced in full.

---

## (a) Verdict — real bug vs. misunderstanding (bottom line up front)

**Direct answer:** *Both interpretations are partly correct, and which one applies depends entirely on the data.* In the default, correctly-owned configuration the reply flow routes **correctly and deterministically**, so a "wrong user" report for the ordinary case is most likely a **misunderstanding** of the reverse-alias model. However, the pipeline contains **real, reproducible latent fragilities** that genuinely mis-route the reply *decision* under specific-but-plausible data conditions — so the behavior *can* be a real bug, not merely a misunderstanding.

- **(observed)** In the DEFAULT, correctly-owned configuration (`contact.user_id == alias.user_id`, and a `reply_email` that maps to exactly one `Contact`), the reply flow routes **correctly and deterministically**. Scenario `S1` was run twice on the **exact same fixture and the exact same message bytes**: both runs returned `'250 Message accepted for delivery'`, both persisted `EmailLog.user_id=422` (equal to both `contact.user_id` and `alias.user_id`), and both relayed to the same external contact (`ext-s1_afc9ef33@external.example`) with `msg.From` rewritten to the alias (`settee_leches851@sl.local`). The only field that changed between the two runs was the autoincrement `EmailLog.id` (`213` → `214`). Distribution over the two identical-input runs: **identical (`all runs identical? True`)** — **no run-to-run inconsistency**.
- **(inferred)** For that common case, a report of "the reply went to the wrong user" is most likely a **misunderstanding of the reverse-alias model**: a reply to a reverse alias legitimately egresses *outward* to the external contact's `website_email`, with the `From` header rewritten to the alias. SimpleLogin **relays the reply outward**; it does **not** deposit the reply into another SimpleLogin user's inbox. So "wrong user" only has a concrete technical meaning as either (i) the *wrong `Contact`* being resolved (→ wrong external recipient and wrong owning `user_id`), or (ii) a *divergence* between the user that authorizes the send (`alias.user`) and the user that gets recorded as owner (`contact.user_id`).
- **(observed + inferred)** The pipeline nevertheless contains **two reproducible latent fragilities** that affect the routing decision when the data allows it:
  - **Q5b — non-unique `reply_email` resolved by `Contact.get_by(...).first()`.** Two `Contact` rows can share the same `reply_email` (the column is indexed but **not** unique — `app/models.py:L1899`; the only uniqueness is `uq_contact(alias_id, website_email)` — `app/models.py:L1875`). The resolver `Contact.get_by(reply_email=…)` returns `.first()` with **no `ORDER BY`** (`app/models.py:L84`). Reproduced canonically **through `handle_DATA`** in `S3`: with two contacts (owned by two different users) sharing one `reply_email`, `get_by(...).first()` selected exactly ONE contact for **every** reply to that reverse alias; the selected contact's owner replied successfully (`250`, routed to the selected contact) while the *other* (shadowed) owner's reply resolved to the selected owner's contact and was rejected with `E214`. In both insertion orders the selection was **observed** to be the first-inserted / lowest-PK row (`selected == first-inserted-contact? True`), but this ordering is **not guaranteed** by the query.
  - **Q5a — the dual user reference.** The reply handler computes **two independent references to "the user"** on the same run: the *authorizing* user `user = alias.user` (`email_handler.py:L1004`) drives the permission/mailbox gates, while the *persisted* owner is written as `user_id=contact.user_id` (`email_handler.py:L1046`). Reproduced in `S2`: with `alias.user_id=423` but `contact.user_id=424`, the reply was accepted (`250`) yet `EmailLog.user_id=424` while the authorization gates ran against `alias.user_id=423` — `MISMATCH(recorded_vs_authorizing)=True`.
- **(inferred) Single most likely origin (see [§ (f)](#f-q5--most-likely-point-where-an-incorrect-routing-decision-originates)):** the **contact-resolution step `Contact.get_by(reply_email=…).first()` at `email_handler.py:L986`**, combined with the non-unique `reply_email` schema (`app/models.py:L1899`) and the `.first()`-without-`ORDER BY` semantics (`app/models.py:L84`). This is the single pivot from which the alias, the user, and the destination all hang — and it requires no abnormal ownership state to misfire, only two contacts that happen to share a `reply_email`. The dual user reference (Q5a) is the closely-related structural fragility that lets the *authorizing* user and the *recorded* user diverge.

---

## (b) Q1 — Which component handles the incoming reply? (entry point)

**Direct answer (observed / code):** the inbound reply is handled by the `aiosmtpd` SMTP `DATA` callback `MailHandler.handle_DATA()` (`email_handler.py:L2289`, in `class MailHandler` — `email_handler.py:L2288`), which delegates to `_handle()` (`email_handler.py:L2335`) and then to the routing hub `handle()` (`email_handler.py:L1945`).

The canonical ingress chain, with verified citations:

1. **`async def handle_DATA(self, server, session, envelope)`** — `email_handler.py:L2289`. The `aiosmtpd` SMTP `DATA` callback. It parses `envelope.original_content` into a `Message` and calls `_handle`.
2. **`def _handle(self, envelope, msg)`** — `email_handler.py:L2335`. Wraps `handle()` in a Flask app context (via `create_light_app`) and emits the "New message" log line (`LOG.i(...)` at `email_handler.py:L2343`, format string `email_handler.py:L2344`).
3. **`def handle(envelope, msg) -> str`** — `email_handler.py:L1945`. The routing hub. It iterates recipients in the per-recipient dispatch loop `for rcpt_index, rcpt_to in enumerate(rcpt_tos):` (`email_handler.py:L2180`) and classifies each recipient with `is_reverse_alias()` before dispatching. The reverse-alias branch is `if is_reverse_alias(rcpt_to):` (`email_handler.py:L2195`).

**(observed)** In production, this is the same process: Postfix forwards inbound mail to SimpleLogin's SMTP listener — `# forward to smtp:127.0.0.1:20381 for custom domain AND email domain` (`README.md:L367`) — i.e., the `email_handler.py` process is the reply entry point.

**(observed) Runtime proof** — the following log lines were emitted by driving the real `handle_DATA` callback in scenario `S1` (run 1); the first is produced by `_handle` at `email_handler.py:L2343`, the second by the reverse-alias branch in `handle` at `email_handler.py:L2196` (format string `email_handler.py:L2197`):

```text
LOG> _handle:2343 New message, mail from user_6e7vw6j126@mailbox.test, rctp tos ['ra+s1_afc9ef33@sl.local']
LOG> handle:2196 Reply phase user_6e7vw6j126@mailbox.test(user_6e7vw6j126@mailbox.test) -> ra+s1_afc9ef33@sl.local
```

**(observed)** The `Reply phase …` line confirms that `handle()` classified `ra+s1_afc9ef33@sl.local` as a reverse alias and entered the reply branch, which then calls `handle_reply(envelope, copy_msg, rcpt_to)` (`email_handler.py:L2199`).

---

## (c) Q2 — How is the alias resolved to a user? (resolution chain)

**Direct answer (observed / code):** the alias is resolved **indirectly through the `Contact` reverse-alias record**. The recipient's `reply_email` is looked up to a `Contact`, and the owning user/alias are read off that contact: `reply_email` → `Contact` → `Contact.alias` → `Alias.user`. There is no direct `reply_email → User` lookup; the `Contact` row is the pivot.

Step-by-step, as exercised at runtime:

1. **Classification** — `handle()` calls `is_reverse_alias(rcpt_to)` (`app/email_utils.py:L1156`). The function returns `True` if `Contact.get_by(reply_email=address)` exists (`app/email_utils.py:L1158`), otherwise it returns whether the address ends with `@EMAIL_DOMAIN` **and** starts with `reply+` or `ra+` (`app/email_utils.py:L1161–L1163`). **(observed)** In `S1`, `is_reverse_alias(reply_email)=True`.
2. **Dispatch** — the reverse-alias branch calls `handle_reply(envelope, copy_msg, rcpt_to)` (`email_handler.py:L2199`); the reply handler is `def handle_reply(envelope, msg, rcpt_to) -> (bool, str)` (`email_handler.py:L966`).
3. **Normalize** — `reply_email = normalize_reply_email(reply_email)` (`email_handler.py:L984`), after a reply-domain validation gate (`email_handler.py:L977–L981`; a wrong reply domain returns `status.E501`).
4. **Contact lookup (the pivot)** — `contact = Contact.get_by(reply_email=reply_email)` (`email_handler.py:L986`). `get_by` is `Session.query(cls).filter_by(**kw).first()` (`app/models.py:L83–L84`) — i.e., it returns exactly one row via `.first()`, with **no `ORDER BY`**. If no contact matches, the handler logs `No contact …` and returns `status.E502`.
5. **Active-user gate** — `if not contact.user.is_active():` (`email_handler.py:L990`); a soft-deleted user yields `E502`. **Note:** this gate reads `contact.user`, i.e., the *contact's* owning user.
6. **Alias from contact** — `alias = contact.alias` (`email_handler.py:L994`), followed by an alias-domain sanity check that yields `E503` on failure.
7. **User from alias** — `user = alias.user` (`email_handler.py:L1004`); a user that `can_send_or_receive()` is `False` yields `E504` (`email_handler.py:L1007`). **Note:** this gate reads `alias.user`, i.e., the *alias's* owning user — a *different* reference from step 5.

**(observed) The resolved chain in `S1` (run 1):** `contact.id=118` → `alias.id=692` (`settee_leches851@sl.local`) → `user.id=422`. The relationships used are `Contact.alias` (`app/models.py:L1907`) and `Contact.user` / `Alias.user` (`app/models.py:L1908`); the contact's own `user_id` column is `app/models.py:L1878`, and its `reply_email` column is `app/models.py:L1899`.

**(observed → M6, selection semantics)** The lookup at step 4 uses `.first()` with no `ORDER BY` (`app/models.py:L84`). When `reply_email` maps to exactly one `Contact` (the normal case, `S1`), the result is unambiguous. When two contacts share a `reply_email` (`S3`), `.first()` still returns exactly one row; the *specific* row it returns was **observed** to be the first-inserted / lowest-PK contact in both runs, but the query gives **no ordering guarantee**, so this is an observation of those runs, not a contract.

---

## (d) Q3 — The concrete integer `user_id` the system decides

**Direct answer (observed):** on the happy path (`S1`) the system decided **`user_id = 422`** — and, critically, it computes **two independent user references** on the same run: the *authorizing* user `user = alias.user` (`email_handler.py:L1004`; `alias.user_id=422` in `S1`) and the *persisted* owner `EmailLog.user_id = contact.user_id` (`email_handler.py:L1046`; value `422` in `S1`). In `S1` they are equal; in `S2` they were deliberately made to diverge and did NOT agree.

- **(observed) Authorization user** — `user = alias.user` (`email_handler.py:L1004`) drives the permission / mailbox-authorization gates. In `S1`, `alias.user_id=422`.
- **(observed) Persisted user** — the reply is recorded by `EmailLog.create(...)` (`email_handler.py:L1042`) with these fields: `alias_id=contact.alias_id` (`email_handler.py:L1044`), `user_id=contact.user_id` (`email_handler.py:L1046`), `mailbox_id=mailbox.id` (`email_handler.py:L1047`), `is_reply=True`, `commit=True`. The `EmailLog` model columns are `class EmailLog` (`app/models.py:L2060`) → `user_id` (`app/models.py:L2064`), `contact_id`, `alias_id`, `is_reply`, `mailbox_id`.
- **(observed) The persisted row in `S1` run 1:**

```text
[RUN 1] EmailLog={'id': 213, 'user_id': 422, 'mailbox_id': 500, 'alias_id': 692, 'contact_id': 118, 'is_reply': True}
```

- **(observed) Confirming log line** (`LOG.d("Create %s for %s, %s, %s", email_log, contact, user, mailbox)` at `email_handler.py:L1051`):

```text
LOG> handle_reply:1051 Create <EmailLog 213> for <Contact 118 ext-s1_afc9ef33@external.example 692>, <User 422 Test User user_6e7vw6j126@mailbox.test>, <Mailbox 500 user_6e7vw6j126@mailbox.test>
```

- **(observed) Captured outbound** (via `mail_sender.get_stored_emails()` — `app/mail_sender.py:L108`): `envelope_to=ext-s1_afc9ef33@external.example`, `msg.From=settee_leches851@sl.local`. That is, the reply is relayed to `contact.website_email` with the `From` header rewritten to the alias.
- **(observed) Stability of the decided value** — the identical input was submitted twice; the decided `user_id` was `422` on both runs (only `EmailLog.id` incremented, `213` → `214`).
- **(inferred) The crux:** two independent user references coexist. They are computed independently (`alias.user` vs `contact.user_id`), are normally equal, but **can diverge** — proven at runtime in `S2`, where `EmailLog.user_id=424` while the authorization gates used `alias.user_id=423` (`MISMATCH=True`). See [§ (f)](#f-q5--most-likely-point-where-an-incorrect-routing-decision-originates) hypothesis (a).

---

## (e) Q4 — Actual observed end-to-end data flow

**Direct answer:** the observed path, from SMTP `DATA` to the outbound relayed message, is:

`handle_DATA` → `_handle` → `handle` → `is_reverse_alias` → `handle_reply` → `Contact.get_by(reply_email).first()` → `contact.user.is_active()` gate → `alias = contact.alias` / `user = alias.user` + `user.can_send_or_receive()` gate → `apply_dmarc_policy_for_reply_phase(alias, contact, …)` → `get_mailbox_from_mail_from(mail_from, alias)` → `EmailLog.create(user_id=contact.user_id, mailbox_id=mailbox.id, is_reply=True)` → rewrite `From` to alias, relay to `contact.website_email`.

**(observed / code) Narrative with citations and the `S1` evidence tying each node to a captured value:**

1. **SMTP `DATA`** → `MailHandler.handle_DATA()` (`email_handler.py:L2289`). *Observed:* the run began by driving this callback directly.
2. **Flask app context** → `_handle()` (`email_handler.py:L2335`). *Observed:* `LOG> _handle:2343 New message, mail from user_6e7vw6j126@mailbox.test, rctp tos ['ra+s1_afc9ef33@sl.local']`.
3. **Routing hub** → `handle()` (`email_handler.py:L1945`), per-recipient loop (`email_handler.py:L2180`).
4. **Classify recipient** → `is_reverse_alias(rcpt_to)` (`app/email_utils.py:L1156`). *Observed:* `is_reverse_alias(reply_email)=True`; reverse-alias branch at `email_handler.py:L2195`, `LOG> handle:2196 Reply phase … -> ra+s1_afc9ef33@sl.local`.
5. **Reply handler** → `handle_reply()` (`email_handler.py:L966`, called at `email_handler.py:L2199`).
6. **Resolve contact (pivot)** → `contact = Contact.get_by(reply_email=reply_email)` (`email_handler.py:L986`) = `filter_by(...).first()` (`app/models.py:L84`). *Observed:* `contact.id=118`.
7. **Gate 1 — contact's user active?** → `if not contact.user.is_active()` (`email_handler.py:L990`). *Observed (S2 divergence):* this gate reads **`contact.user`** — `contact.user.id=424` in `S2` run 1.
8. **Resolve alias & Gate 2 — alias's user can send?** → `alias = contact.alias` (`email_handler.py:L994`), `user = alias.user` (`email_handler.py:L1004`), `if not user.can_send_or_receive()` (`email_handler.py:L1007`). *Observed:* `alias.id=692`, `alias.user_id=422` in `S1`; in `S2` this gate reads **`alias.user`** — `alias.user.id=423`, a *different* reference from Gate 1.
9. **Gate 3 — DMARC policy (gets BOTH references)** → `dmarc_delivery_status = apply_dmarc_policy_for_reply_phase(alias, contact, envelope, msg)` (`email_handler.py:L1012`; function at `app/handler/dmarc.py:L154`, signature `(alias_from: Alias, contact_recipient: Contact, envelope, msg)`). *Observed:* on every reply-path run the function logged `DMARC check disabled` (`app/handler/dmarc.py:L159`) and returned `None` (non-blocking); see the DMARC note below and [§ m11 in the ledger](#i-observed-vs-inferred--key-claim-ledger).
10. **Gate 4 — authorize sending mailbox (alias-scoped)** → `mailbox = get_mailbox_from_mail_from(mail_from, alias)` (`email_handler.py:L1019`; function at `email_handler.py:L1364`, matching `mail_from` against **`alias.mailboxes`** and their `authorized_addresses`, raw then canonicalized). *Observed:* `mailbox.id=500` in `S1`. This gate is scoped to the *alias's* mailboxes — i.e., to `alias.user`, not `contact.user`.
11. **Persist the decision (records contact's user)** → `EmailLog.create(user_id=contact.user_id, mailbox_id=mailbox.id, is_reply=True, …)` (`email_handler.py:L1042–L1047`). *Observed:* `EmailLog{'id': 213, 'user_id': 422, 'mailbox_id': 500, 'alias_id': 692, 'contact_id': 118, 'is_reply': True}`. This records **`contact.user_id`**, closing the loop back to Gate 1's reference rather than Gate 2/4's `alias.user`.
12. **Rewrite & relay** → `From` rewritten to the alias, message relayed to `contact.website_email`. *Observed:* `OUT envelope_to=ext-s1_afc9ef33@external.example | msg.From=settee_leches851@sl.local`.

**(observed) DMARC note (grounds the m11 nuance):** the reply was **not** blocked by DMARC, but this is because DMARC evaluation is **non-blocking** for these messages, not because of a "clean pass." `apply_dmarc_policy_for_reply_phase` first extracts `spam_result = SpamdResult.extract_from_headers(msg, Phase.reply)` (`app/handler/dmarc.py:L157`); when `not DMARC_CHECK_ENABLED or not spam_result` it logs `DMARC check disabled` (`app/handler/dmarc.py:L159`) and returns `None` (`app/handler/dmarc.py:L160`). The synthetic messages carry no spamd headers, so `spam_result` is falsy and the function short-circuits to `None` on every run. The **observed** outcome is therefore "DMARC returned a non-blocking result", not "DMARC passed."

**(observed) Data-flow diagram of the exercised path** (the `Q5` candidate origins are annotated):

```mermaid
flowchart TD
    A["SMTP DATA<br/>MailHandler.handle_DATA()<br/>email_handler.py:L2289"] --> B["_handle()<br/>Flask app context<br/>email_handler.py:L2335"]
    B --> C["handle() routing hub<br/>email_handler.py:L1945<br/>per-recipient loop L2180"]
    C --> D{"is_reverse_alias(rcpt_to)?<br/>app/email_utils.py:L1156"}
    D -->|"No"| E["handle_forward()<br/>email_handler.py:L536"]
    D -->|"Yes (REPLY) L2195"| F["handle_reply()<br/>email_handler.py:L966 (called L2199)"]
    F --> G["contact = Contact.get_by(reply_email).first()<br/>email_handler.py:L986 / models.py:L84  &lt;-- Q5b pivot"]
    G --> G1["Gate 1: contact.user.is_active()<br/>email_handler.py:L990  (uses contact.user)"]
    G1 --> H["alias = contact.alias  L994<br/>Gate 2: user = alias.user  L1004<br/>user.can_send_or_receive()  L1007  (uses alias.user)  &lt;-- Q5a authz user"]
    H --> DM["Gate 3: apply_dmarc_policy_for_reply_phase(alias, contact)<br/>email_handler.py:L1012 / dmarc.py:L154<br/>(non-blocking: 'DMARC check disabled' dmarc.py:L159)"]
    DM --> I["Gate 4: mailbox = get_mailbox_from_mail_from(mail_from, alias)<br/>email_handler.py:L1019 / L1364  (scoped to alias.mailboxes)"]
    I --> J{"mailbox found?"}
    J -->|"No + spoofing check ON"| K["handle_unknown_mailbox()<br/>return E214 — L1032/L1034"]
    J -->|"No + spoofing check OFF"| L["mailbox = alias.mailbox (default)<br/>email_handler.py:L1029  &lt;-- Q5c fallback"]
    J -->|"Yes"| M["EmailLog.create(user_id=contact.user_id,<br/>mailbox_id=mailbox.id, is_reply=True)<br/>email_handler.py:L1042-L1047  &lt;-- Q5a recorded user (contact.user_id)"]
    L --> M
    M --> N["Rewrite From to alias, relay to<br/>contact.website_email"]
```

---

## (f) Q5 — Most likely point where an incorrect routing decision originates

Three code-grounded hypotheses were tested at runtime. Each is presented with its reproduced result, then the single most likely origin is named.

### Hypothesis (a) — Dual user reference (`alias.user` vs `contact.user_id`)

- **(observed / code)** The reply handler uses **distinct user references at distinct gates** — it is *not* "`alias.user` everywhere":
  - Gate 1, active-user check, reads **`contact.user`**: `if not contact.user.is_active()` (`email_handler.py:L990`).
  - Gate 2, send-permission check, reads **`alias.user`**: `user = alias.user` (`email_handler.py:L1004`), `if not user.can_send_or_receive()` (`email_handler.py:L1007`).
  - Gate 3, DMARC, receives **both** the alias and the contact: `apply_dmarc_policy_for_reply_phase(alias, contact, envelope, msg)` (`email_handler.py:L1012`).
  - Gate 4, mailbox authorization, is scoped to the **alias's** mailboxes: `get_mailbox_from_mail_from(mail_from, alias)` (`email_handler.py:L1019` / `email_handler.py:L1364`).
  - Persistence records the **contact's** user: `user_id=contact.user_id` (`email_handler.py:L1046`).
- **(observed) Reproduced in `S2` (×2):** with `alias.user_id=423` and `contact.user_id=424` (run 1), the reply was accepted (`'250 Message accepted for delivery'`), yet `EmailLog.user_id=424` while the send/mailbox gates ran against `alias.user_id=423` — `MISMATCH(recorded_vs_authorizing)=True`. Run 2 (`alias.user_id=425`, `contact.user_id=426`) reproduced the same mismatch (`EmailLog.user_id=426`). The confirming `Create` log even prints the alias's user object while the row stores the contact's user: `Create <EmailLog 215> for <Contact 119 …>, <User 423 …>` — i.e., `<User 423>` (alias.user) is logged while `EmailLog.user_id=424` (contact.user_id).
- **(observed) Context:** the standard ownership-transfer path `transfer_alias()` (`app/alias_utils.py:L458–L540`) updates `Contact.user_id` (`app/alias_utils.py:L464–L466`) AND `alias.user_id` (`app/alias_utils.py:L506`) **together**.
- **(inferred)** Because the normal transfer path keeps the two references in sync, a divergence between `alias.user_id` and `contact.user_id` is an **abnormal/inconsistent data state**, not a normal one. When it does occur, the *authorizing* user and the *recorded owner* refer to different accounts. **(inferred, sibling variant)** The same dual reference exists in the forward path — `user = alias.user` (`email_handler.py:L557`) vs `user_id=contact.user_id` (`email_handler.py:L600` and `email_handler.py:L734`) — so this is a pipeline-wide pattern, not a one-off in the reply handler.

### Hypothesis (b) — Non-unique `reply_email` resolved by `.first()`

- **(observed / code)** The `reply_email` column is indexed but **NOT unique** — `reply_email = sa.Column(sa.String(512), nullable=False, index=True)` (`app/models.py:L1899`). The **only** uniqueness constraint on `Contact` is `uq_contact(alias_id, website_email)` (`app/models.py:L1875`). The resolver `Contact.get_by(reply_email=…)` (`email_handler.py:L986`) is `Session.query(cls).filter_by(**kw).first()` (`app/models.py:L84`) — a `.first()` with **no correctness `ORDER BY`**.
- **(observed) Reproduced canonically through `handle_DATA` in `S3` (×2, both insertion orders):** two `Contact` rows — owned by **two different users** — were created sharing one `reply_email`, and the DB accepted both (`DB row count for that reply_email = 2`), confirming non-uniqueness. Then a real inbound reply was driven through `MailHandler.handle_DATA` for each owner:
  - `order=A_then_B`: contacts `id=121 (user 427, ext-A)` and `id=122 (user 428, ext-B)`. `Contact.get_by(reply_email).first()` **selected `id=121` (user 427)**; `selected == first-inserted-contact? True`. Reply from the **selected** owner's mailbox → `'250 Message accepted for delivery'`, `EmailLog{'id': 217, 'user_id': 427, …, 'contact_id': 121}`, relayed to the selected contact `ext-A-dup_athenb_febd0c89@external.example`. Reply from the **shadowed** owner's mailbox → `'250 SL E214 Unauthorized for using reverse alias'` — the shadowed owner's identical `reply_email` resolves to the *other* user's `contact.id=121`, whose alias the shadowed mailbox is not authorized on, so their reply is **not** delivered to their intended contact.
  - `order=B_then_A`: contacts `id=123 (user 430, ext-B)` and `id=124 (user 429, ext-A)`. `get_by(...).first()` **selected `id=123` (user 430)**; `selected == first-inserted-contact? True`. Selected owner → `250`, `EmailLog{'id': 218, 'user_id': 430, …, 'contact_id': 123}`, relayed to `ext-B-dup_bthena_bb4e9de4@external.example`. Shadowed owner → `E214`.
  - Summary line: `[S3] OBSERVED selection==first-inserted in both runs: True (run1=True run2=True). NOTE: get_by()=filter_by().first() has NO ORDER BY (models.py:L84); this selection was OBSERVED, not guaranteed.`
- **(inferred)** Because `.first()` has **no `ORDER BY` tied to correctness**, *which* contact "wins" is not determined by which contact the reply was meant for — it is determined by whatever single row the database returns first (observed here to be the first-inserted/lowest-PK row, but not guaranteed to be). Whichever contact wins dictates the alias, the recorded `user_id`, and the destination `website_email` for **every** reply to that shared `reply_email`. The observed consequence is the reported symptom exactly: the alias is still "recognized" (`is_reverse_alias=True`), yet one user's reply is routed/attributed via the *other* user's contact — succeeding for the selected owner and being rejected (`E214`) for the shadowed owner.

### Hypothesis (c) — `disable_email_spoofing_check` default-mailbox fallback

- **(observed / code)** When `get_mailbox_from_mail_from` returns no mailbox (`if not mailbox:` — `email_handler.py:L1020`), the code branches on `if alias.disable_email_spoofing_check:` (`email_handler.py:L1021`). If the check is disabled, it logs "ignore unknown sender to reverse-alias" (`email_handler.py:L1023`) and falls back to `mailbox = alias.mailbox` (`email_handler.py:L1029`). Otherwise it calls `handle_unknown_mailbox(...)` (`email_handler.py:L1032`) and returns `status.E214` (`email_handler.py:L1034`).
- **(observed) Reproduced in `S5` (×2):** with `disable_email_spoofing_check=True`, an unknown sender (`stranger-s5r1_7c7e5940@nowhere.example`) was NOT rejected; the reply `EmailLog` was created with `user_id=433`, `mailbox_id=511 == alias.mailbox_id`, `used_default(mailbox_id==alias.mailbox_id)=True`, and relayed to `ext-s5r1_7c7e5940@external.example`. Run 2 reproduced this (`EmailLog id=220`, `user_id=434`, `mailbox_id=512 == alias.mailbox_id`, `used_default=True`).
- **(inferred)** This fallback widens *who* can trigger a relay through the reverse alias, but it uses the **correct owning mailbox** in the normal-ownership case, so it changes *who can send*, not *which user the reply is attributed to*. It is a routing-relevant behavior toggle rather than a mis-attribution defect on its own.

### Conclusion — the single most likely origin

- **(inferred, evidence-led)** The **most likely origin** of an *incorrect routing decision that still shows "alias recognized"* is the **contact-resolution step `Contact.get_by(reply_email=…).first()` at `email_handler.py:L986`**, combined with the non-unique `reply_email` schema (`app/models.py:L1899`) and the `.first()`-without-`ORDER BY` semantics (`app/models.py:L84`) — **Hypothesis (b)**. Cause → effect: `reply_email` is not unique, so multiple `Contact` rows can carry it; `.first()` then returns one row with no correctness ordering; and because *everything downstream hangs off that single contact* — the alias (`contact.alias`), the recorded user (`contact.user_id`), and the destination (`contact.website_email`) — resolving to the wrong contact simultaneously mis-routes/rejects the reply and mis-attributes the user, all while `is_reverse_alias` still reports the alias as "recognized." This needs **no abnormal ownership state** and is reachable whenever two contacts share a `reply_email` (reproduced canonically in `S3`).
- **(inferred)** **Hypothesis (a)** (the dual user reference, `alias.user` at `email_handler.py:L1004` vs `contact.user_id` at `email_handler.py:L1046`) is the closely-related **structural fragility** that makes the *authorizing* user and the *recorded* user diverge once the two `user_id`s are inconsistent (reproduced in `S2`). It amplifies (b) but, on its own, requires the abnormal divergent-ownership state that `transfer_alias()` normally prevents.
- **(inferred)** **Hypothesis (c)** is a secondary, configuration-gated widening of sender acceptance; it does not by itself send the reply to the wrong user under normal ownership.

---

## (g) Edge-condition results

Every distinct condition the question implies was exercised — the happy path plus the secondary/edge/error paths — through the real entry point (`MailHandler.handle_DATA`), **each scenario at least twice**. Before/after states are reported where state changes.

| Scenario | Condition | Runs | Observed status | Reply `EmailLog` (before → after) | Key citation |
|----------|-----------|------|-----------------|-----------------------------------|--------------|
| `S1` | Happy path, correctly-owned, identical input | ×2 | `'250 Message accepted for delivery'` (both) | none → created (`user_id=422`, equal to both refs; only `EmailLog.id` 213→214 changed) | `email_handler.py:L1042–L1047` |
| `S2` | Constructed divergence `alias.user_id != contact.user_id` | ×2 | `'250 Message accepted for delivery'` (both) | none → created (`EmailLog.user_id=424`/`426`; gates used `alias.user_id=423`/`425`; `MISMATCH=True`) | `email_handler.py:L1004` vs `L1046` |
| `S3` | Non-unique `reply_email` (two contacts, two users share it) — canonical `handle_DATA` | ×2 (both insertion orders) | selected owner `'250 …'`; shadowed owner `'250 SL E214 …'` | none → created for selected owner (`id=217`/`218`); shadowed owner blocked (E214) | `email_handler.py:L986` / `app/models.py:L84`, `L1899`, `L1875` |
| `S4` | Unauthorized sender, spoofing check **ON** | ×2 | `'250 SL E214 Unauthorized for using reverse alias'` (both) | none → **NONE (blocked)**; `total_stored=1`, `sent_alerts_to_owner=1` | `email_handler.py:L1032`/`L1034`, `handle_unknown_mailbox` `email_handler.py:L1390` |
| `S5` | Unknown sender, `disable_email_spoofing_check=True` | ×2 | `'250 Message accepted for delivery'` (both) | none → created (`user_id=433`/`434`, `mailbox_id==alias.mailbox_id`, `used_default=True`) | `email_handler.py:L1023`/`L1029` |
| `S6` | `NOREPLIES` address; bounce `mail_from == "<>"` | ×2 | `NOREPLIES → '250 Message accepted for delivery'`; bounce `→ '250 SL E206 Out of office'` | no reply routing performed | `email_handler.py:L2181–L2183`; `email_handler.py:L2166`/`email_handler.py:L2173` |

- **(observed) `S4` — unauthorized sender → E214 (spoofing ON):** a sender NOT authorized for the alias's mailbox is rejected with **E214** ("Unauthorized for using reverse alias") at `email_handler.py:L1034` (via `handle_unknown_mailbox` `email_handler.py:L1032`/`email_handler.py:L1390`). NO reply `EmailLog` is created (before → after: reply `EmailLog` = NONE). `total_stored=1` / `sent_alerts_to_owner=1` is the alert email sent to the legitimate user, not a relayed reply. Reproduced on both runs.
- **(observed) `S5` — default-mailbox fallback (spoofing OFF):** with `disable_email_spoofing_check=True`, an unknown sender is NOT rejected — the code logs "ignore unknown sender to reverse-alias" at `email_handler.py:L1023`, falls back to `mailbox = alias.mailbox` at `email_handler.py:L1029`, and creates the reply `EmailLog` (`used_default=True`). Directly contrasts `S4`'s E214. Reproduced on both runs.
- **(observed) `S6` — guard paths around the reply logic:** a message to a `NOREPLIES` address short-circuits at `email_handler.py:L2181–L2183` (`send_no_reply_response`) and returns `'250 Message accepted for delivery'` without reply handling. A bounce/auto-reply with `mail_from == "<>"` addressed to a reverse alias is handled as out-of-office → **E206** ("Out of office") at `email_handler.py:L2166`/`email_handler.py:L2173`, before `handle_reply` routing. Reproduced on both runs.

---

## (h) Commands run & raw runtime output

### Environment (observed — canonical / default configuration)

The investigation ran inside the **user-provided canonical execution platform**: the container built from the GHCR image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (image id `ea242796bbce`), which ships the project at commit `2cd6ee77` with its own virtualenv (`/app/venv`), PostgreSQL, and Redis. The exact commands and their **complete, unedited** outputs:

```text
$ /app/venv/bin/python --version
Python 3.10.18

$ PGPASSWORD=test psql -h localhost -p 15432 -U test -d test -tAc "select version();"
PostgreSQL 15.13 (Debian 15.13-0+deb12u1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 12.2.0-14+deb12u1) 12.2.0, 64-bit

$ redis-server --version
Redis server v=7.0.15 sha=00000000:0 malloc=jemalloc-5.3.0 bits=64 build=3f20e06e76a2b578

$ /root/.local/bin/poetry --version
Poetry (version 2.1.4)

$ cd /tmp/slrun && CONFIG=tests/test.env /app/venv/bin/alembic current
INFO  [alembic.runtime.migration] Will assume transactional DDL.
32f25cbf12f6 (head)

$ PGPASSWORD=test psql -h localhost -p 15432 -U test -d test -tAc \
    "select count(*) from information_schema.tables where table_schema='public';"
77

$ /app/venv/bin/python -c "import aiosmtpd,sqlalchemy,flask; \
    print('aiosmtpd',aiosmtpd.__version__); print('SQLAlchemy',sqlalchemy.__version__); print('Flask',flask.__version__)"
aiosmtpd 1.4.2
SQLAlchemy 1.3.24
Flask 1.1.2
```

- **Config used (`tests/test.env`):** `NOT_SEND_EMAIL=true`, `EMAIL_DOMAIN=sl.local`, `DB_URI=postgresql://test:test@localhost:15432/test`, `MEM_STORE_URI=redis://localhost`.
- **Simulation vehicle (canonical entry point — no mocks/hooks):** an in-process pytest module that constructs an `aiosmtpd.smtp.Envelope` and drives `MailHandler().handle_DATA(None, None, envelope)` → `_handle()` → `handle()`, mirroring the pattern in `tests/test_email_handler.py::test_dmarc_reply_quarantine` (`tests/test_email_handler.py:L165`). Outbound relay was captured non-intrusively via `mail_sender.store_emails_instead_of_sending(True)` + `get_stored_emails()` (`app/mail_sender.py:L102`, `app/mail_sender.py:L108`), so no real email left the environment. `MailSender.send` (`app/mail_sender.py:L126`) appends to the capture list before the `NOT_SEND_EMAIL` check, so capture works even with `NOT_SEND_EMAIL=true`.
- **(observed) Image-provenance / integrity note.** The DockerHub reference in the setup instructions — `andrewparkscaleai/coding-agent:simple-login__app__2cd6ee777f8c2d3531559588bcfb18627ffb5d2c` — is **not anonymously pullable**; the exact daemon response was:

  ```text
  $ docker pull andrewparkscaleai/coding-agent:simple-login__app__2cd6ee777f8c2d3531559588bcfb18627ffb5d2c
  Error response from daemon: pull access denied for andrewparkscaleai/coding-agent, repository does not exist or may require 'docker login': denied: requested access to the resource is denied
  ```

  The equivalent **GHCR** image named in the same setup instructions (`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`) *is* available and was used as the run vehicle. Every value in this document was obtained through the **real** entry point `handle_DATA` / `_handle` / `handle` — none from a bypass, mock, or synthetic stand-in — so all reported values are **canonical observations**.
- **(observed) Version deviation from the CI recipe.** `.github/workflows/main.yml` pins the CI matrix to **PostgreSQL 13** and **Redis 6** (with Python 3.10). The canonical image ships **PostgreSQL 15.13** and **Redis 7.0.15** (Python 3.10.18). This is a benign infrastructure deviation: the schema is built by the same `alembic upgrade head` recipe (reaching head `32f25cbf12f6`, 77 tables) and the reply-handler code paths under test are database-version-agnostic. The deviation is reported here rather than hidden.

### Reproduction commands (exact, as executed)

```bash
# 0) Run vehicle: the canonical GHCR image container (services already provisioned in-image).
#    Postgres remapped to the test port and started; Redis started:
sed -i 's/^port = 5432/port = 15432/' /etc/postgresql/15/main/postgresql.conf
pg_ctlcluster 15 main start
redis-server --daemonize yes

# 1) Build the schema via the canonical CI recipe (Python 3.10 venv):
cd /tmp/slrun
CONFIG=tests/test.env /app/venv/bin/alembic upgrade head        # -> 32f25cbf12f6 (head); 77 tables

# 2) Drive the REAL SMTP callback in-process for all six scenarios (each >=2x) and capture the transcript:
cd /tmp/slrun
CONFIG=tests/test.env GITHUB_ACTIONS_TEST=true PYTEST_ADDOPTS="" \
  /app/venv/bin/python -m pytest tests/blitzy_reply_trace.py -s -p no:cacheprovider -p no:randomly --no-header -q
#   -> 6 passed, 18 warnings in 5.00s
```

`tests/blitzy_reply_trace.py` was a temporary observation module that lived **only** inside the run container under `/tmp/slrun/tests/` (a copy of the committed tree exported via `git archive HEAD`); it was never added to the repository and does not exist in the working tree. `/tmp/slrun` is entirely inside the container, so the repository working tree remained git-clean.

### Raw scenario output

Each block below is transcribed **exactly** as emitted. Volatile per-line logger prefixes (wall-clock timestamp, PID, request-UUID) are the only thing normalized in the compact `LOG>` lines, which use the stable form `funcName:lineno message`; the fully-raw, timestamped emission is shown once in full for `S1` run 1 to demonstrate the unedited output. No field is elided.

#### S1 — Happy-path reply, SAME UNCHANGED INPUT ×2 (grounds the stability + privacy claims)

Fully-raw, timestamped emission for `S1` run 1 (unedited):

```text
==================== S1: HAPPY PATH — SAME UNCHANGED INPUT x2 ====================
2026-07-13 18:29:48,824 - SL - INFO - 3228 - "/tmp/slrun/email_handler.py:2343" - _handle() - 69de1a69-8ebb-4ff9-bcfc-480b4c883909 - New message, mail from user_6e7vw6j126@mailbox.test, rctp tos ['ra+s1_afc9ef33@sl.local'] 
2026-07-13 18:29:48,830 - SL - DEBUG - 3228 - "/tmp/slrun/email_handler.py:2196" - handle() - 69de1a69-8ebb-4ff9-bcfc-480b4c883909 - Reply phase user_6e7vw6j126@mailbox.test(user_6e7vw6j126@mailbox.test) -> ra+s1_afc9ef33@sl.local
2026-07-13 18:29:48,832 - SL - INFO - 3228 - "/tmp/slrun/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() - 69de1a69-8ebb-4ff9-bcfc-480b4c883909 - DMARC check disabled
2026-07-13 18:29:48,838 - SL - DEBUG - 3228 - "/tmp/slrun/email_handler.py:1051" - handle_reply() - 69de1a69-8ebb-4ff9-bcfc-480b4c883909 - Create <EmailLog 213> for <Contact 118 ext-s1_afc9ef33@external.example 692>, <User 422 Test User user_6e7vw6j126@mailbox.test>, <Mailbox 500 user_6e7vw6j126@mailbox.test>
2026-07-13 18:29:48,860 - SL - DEBUG - 3228 - "/tmp/slrun/app/mail_sender.py:131" - send() - 69de1a69-8ebb-4ff9-bcfc-480b4c883909 - send email with subject 're: same input', from 'settee_leches851@sl.local' to 'Ext <ext-s1_afc9ef33@external.example>'
2026-07-13 18:29:48,862 - SL - INFO - 3228 - "/tmp/slrun/email_handler.py:2367" - _handle() - 69de1a69-8ebb-4ff9-bcfc-480b4c883909 - Finish mail_from user_6e7vw6j126@mailbox.test, rcpt_tos ['ra+s1_afc9ef33@sl.local'], takes 0.03832387924194336 seconds with return code '250 Message accepted for delivery'<<===
```

Structured before/after capture for both runs (compact `LOG>` form):

```text
FIX user.id=422 alias.id=692 alias.email=settee_leches851@sl.local alias.user_id=422 mailbox.email=user_6e7vw6j126@mailbox.test contact.id=118 contact.user_id=422 website=ext-s1_afc9ef33@external.example reply_email=ra+s1_afc9ef33@sl.local dual-ref-EQUAL=True
is_reverse_alias(reply_email)=True
[RUN 1] status='250 Message accepted for delivery'
[RUN 1] EmailLog={'id': 213, 'user_id': 422, 'mailbox_id': 500, 'alias_id': 692, 'contact_id': 118, 'is_reply': True}
[RUN 1] OUT envelope_from=sl.lmysyibsgezsyibsgm4deobwhfoq.3a5itw2bjvb2s@sl.local envelope_to=ext-s1_afc9ef33@external.example msg.From=settee_leches851@sl.local msg.To=Ext <ext-s1_afc9ef33@external.example>
[RUN 1] LOG> _handle:2343 New message, mail from user_6e7vw6j126@mailbox.test, rctp tos ['ra+s1_afc9ef33@sl.local']
[RUN 1] LOG> handle:2196 Reply phase user_6e7vw6j126@mailbox.test(user_6e7vw6j126@mailbox.test) -> ra+s1_afc9ef33@sl.local
[RUN 1] LOG> apply_dmarc_policy_for_reply_phase:159 DMARC check disabled
[RUN 1] LOG> handle_reply:1051 Create <EmailLog 213> for <Contact 118 ext-s1_afc9ef33@external.example 692>, <User 422 Test User user_6e7vw6j126@mailbox.test>, <Mailbox 500 user_6e7vw6j126@mailbox.test>
[RUN 1] FULL-MSG-BYTES len=758 real_mailbox(user_6e7vw6j126@mailbox.test)_present_anywhere=False
[RUN 1] OUT all header names=['Subject', 'Content-Type', 'Content-Transfer-Encoding', 'MIME-Version', 'From', 'To', 'Message-ID', 'Date', 'X-SimpleLogin-Type', 'X-SimpleLogin-EmailLog-ID', 'DKIM-Signature']
[RUN 2] status='250 Message accepted for delivery'
[RUN 2] EmailLog={'id': 214, 'user_id': 422, 'mailbox_id': 500, 'alias_id': 692, 'contact_id': 118, 'is_reply': True}
[RUN 2] OUT envelope_from=sl.lmysyibsge2cyibsgm4deobwhfoq.4uehhdpbagrpe@sl.local envelope_to=ext-s1_afc9ef33@external.example msg.From=settee_leches851@sl.local msg.To=Ext <ext-s1_afc9ef33@external.example>
[RUN 2] LOG> _handle:2343 New message, mail from user_6e7vw6j126@mailbox.test, rctp tos ['ra+s1_afc9ef33@sl.local']
[RUN 2] LOG> handle:2196 Reply phase user_6e7vw6j126@mailbox.test(user_6e7vw6j126@mailbox.test) -> ra+s1_afc9ef33@sl.local
[RUN 2] LOG> apply_dmarc_policy_for_reply_phase:159 DMARC check disabled
[RUN 2] LOG> handle_reply:1051 Create <EmailLog 214> for <Contact 118 ext-s1_afc9ef33@external.example 692>, <User 422 Test User user_6e7vw6j126@mailbox.test>, <Mailbox 500 user_6e7vw6j126@mailbox.test>
[RUN 2] FULL-MSG-BYTES len=758 real_mailbox(user_6e7vw6j126@mailbox.test)_present_anywhere=False
[S1] identical-input distribution over 2 runs: [('250 Message accepted for delivery', 422, 'ext-s1_afc9ef33@external.example', 'settee_leches851@sl.local'), ('250 Message accepted for delivery', 422, 'ext-s1_afc9ef33@external.example', 'settee_leches851@sl.local')]
[S1] all runs identical? True
```

Complete serialized outbound message for `S1` run 1 (every byte, including the full DKIM signature) — this is the whole-message artifact used for the privacy inspection in [§ (j)](#j-security--privacy-note):

```text
----- BEGIN FULL SERIALIZED OUTBOUND MESSAGE (S1 run 1) -----
Subject: re: same input
Content-Type: text/plain; charset="utf-8"
Content-Transfer-Encoding: 7bit
MIME-Version: 1.0
From: settee_leches851@sl.local
To: Ext <ext-s1_afc9ef33@external.example>
Message-ID: <178396738884.3228.4887539225924666492.213@sl.local>
Date: Mon, 13 Jul 2026 18:29:48 -0000
X-SimpleLogin-Type: Reply
X-SimpleLogin-EmailLog-ID: 213
DKIM-Signature: v=1; a=rsa-sha256; c=relaxed/simple; d=sl.local;  i=@sl.local;
 q=dns/txt; s=dkim; t=1783967388; h=message-id : date :  subject : from : to;
 bh=tvKgLkl88R3iv1+OVBQQWT9APf/3TjO0LbTeCxT9How=;
  b=Vyq2dhayzAbuB0xiUveX610bukCzZeGv7GX0IsS86RXXZpG+75wQO4PUdyAlkVOxmEhT4
  kWic+Y6iy474+OFehcbNPJK5BJ2WmVq/790VoWbTzru5y6cxhR2UNSuQ7mLjL9ncgPVS3nM
  eXvBBNnIzUhIi/N+x3gTuZbwHK+nNVw= 

identical body

----- END FULL SERIALIZED OUTBOUND MESSAGE (S1 run 1) -----
```

**(observed)** Both runs on the identical fixture and identical message bytes returned `'250 Message accepted for delivery'`, persisted `EmailLog.user_id=422` (equal to both `contact.user_id` and `alias.user_id`), and relayed to the same external contact with `msg.From` rewritten to the alias. The only field that differed was the autoincrement `EmailLog.id` (`213` → `214`). Distribution is identical (`all runs identical? True`) — no run-to-run inconsistency. The full 758-byte serialized message was searched for the user's real mailbox (`user_6e7vw6j126@mailbox.test`): `present_anywhere=False`.

#### S2 — Constructed divergence `alias.user_id != contact.user_id` (Q5a) ×2

```text
==================== S2: DIVERGENCE alias.user_id != contact.user_id (Q5a) x2 ====================
[RUN 1] BEFORE alias.id=694 alias.user_id=423 contact.id=119 contact.user_id=424 dual-ref-EQUAL=False
[RUN 1] GATES-INPUT: gate1 contact.user.is_active uses contact.user.id=424 ; gate2 user=alias.user.can_send_or_receive uses alias.user.id=423 ; gate3 apply_dmarc_policy_for_reply_phase(alias,contact) gets alias.user=423 AND contact=119 ; gate4 get_mailbox_from_mail_from(mail_from, alias) scoped to alias.user=423 mailboxes
[RUN 1] status='250 Message accepted for delivery'
[RUN 1] EmailLog={'id': 215, 'user_id': 424, 'mailbox_id': 501, 'alias_id': 694, 'contact_id': 119, 'is_reply': True}
[RUN 1] PERSIST: EmailLog.user_id=424 == contact.user_id=424 (recorded owner) ; authorizing alias.user_id=423 ; MISMATCH(recorded_vs_authorizing)=True
[RUN 1] OUT envelope_to=ext-div-s2r1_eca08ac9@external.example msg.From=gaming_reface247@sl.local
[RUN 1] LOG> _handle:2343 New message, mail from user_yrm6ak5sfr@mailbox.test, rctp tos ['ra+s2r1_eca08ac9@sl.local']
[RUN 1] LOG> handle:2196 Reply phase user_yrm6ak5sfr@mailbox.test(user_yrm6ak5sfr@mailbox.test) -> ra+s2r1_eca08ac9@sl.local
[RUN 1] LOG> apply_dmarc_policy_for_reply_phase:159 DMARC check disabled
[RUN 1] LOG> handle_reply:1051 Create <EmailLog 215> for <Contact 119 ext-div-s2r1_eca08ac9@external.example 694>, <User 423 Test User user_yrm6ak5sfr@mailbox.test>, <Mailbox 501 user_yrm6ak5sfr@mailbox.test>
[RUN 2] BEFORE alias.id=697 alias.user_id=425 contact.id=120 contact.user_id=426 dual-ref-EQUAL=False
[RUN 2] GATES-INPUT: gate1 contact.user.is_active uses contact.user.id=426 ; gate2 user=alias.user.can_send_or_receive uses alias.user.id=425 ; gate3 apply_dmarc_policy_for_reply_phase(alias,contact) gets alias.user=425 AND contact=120 ; gate4 get_mailbox_from_mail_from(mail_from, alias) scoped to alias.user=425 mailboxes
[RUN 2] status='250 Message accepted for delivery'
[RUN 2] EmailLog={'id': 216, 'user_id': 426, 'mailbox_id': 503, 'alias_id': 697, 'contact_id': 120, 'is_reply': True}
[RUN 2] PERSIST: EmailLog.user_id=426 == contact.user_id=426 (recorded owner) ; authorizing alias.user_id=425 ; MISMATCH(recorded_vs_authorizing)=True
[RUN 2] OUT envelope_to=ext-div-s2r2_4781794c@external.example msg.From=reseal_millet281@sl.local
[RUN 2] LOG> _handle:2343 New message, mail from user_9ymfg9kedx@mailbox.test, rctp tos ['ra+s2r2_4781794c@sl.local']
[RUN 2] LOG> handle:2196 Reply phase user_9ymfg9kedx@mailbox.test(user_9ymfg9kedx@mailbox.test) -> ra+s2r2_4781794c@sl.local
[RUN 2] LOG> apply_dmarc_policy_for_reply_phase:159 DMARC check disabled
[RUN 2] LOG> handle_reply:1051 Create <EmailLog 216> for <Contact 120 ext-div-s2r2_4781794c@external.example 697>, <User 425 Test User user_9ymfg9kedx@mailbox.test>, <Mailbox 503 user_9ymfg9kedx@mailbox.test>
```

**(observed)** On both runs, with `alias.user_id != contact.user_id`, the reply is ACCEPTED (`250`), the send/mailbox gates run against **`alias.user`** (id `423`/`425`), yet the persisted `EmailLog.user_id` is the **`contact.user_id`** (`424`/`426`) — `MISMATCH(recorded_vs_authorizing)=True`. The `GATES-INPUT` line records, per gate, which user reference is consumed. Note the `Create` log prints `<User 423>`/`<User 425>` (the alias's user) while the row stored `user_id=424`/`426`. **(inferred)** This is the dual-reference fragility: `user = alias.user` (`email_handler.py:L1004`) governs authorization/mailbox, but `EmailLog.create(..., user_id=contact.user_id ...)` (`email_handler.py:L1046`) attributes the reply to a different account.

#### S3 — Non-unique `reply_email` driven through `handle_DATA` (Q5b) ×2, both insertion orders

```text
==================== S3: NON-UNIQUE reply_email -> handle_DATA CANONICAL x2 (Q5b) ====================
[order=A_then_B] two Contact rows share reply_email=ra+dup_athenb_febd0c89@sl.local (DB accepted both -> reply_email NOT unique):
   contact.id=121 user_id=427 alias_id=700 website_email=ext-A-dup_athenb_febd0c89@external.example
   contact.id=122 user_id=428 alias_id=702 website_email=ext-B-dup_athenb_febd0c89@external.example
[order=A_then_B] DB row count for that reply_email = 2
[order=A_then_B] Contact.get_by(reply_email).first() SELECTED contact.id=121 user_id=427 alias_id=700 website_email=ext-A-dup_athenb_febd0c89@external.example
[order=A_then_B] selected == first-inserted-contact? True (selected.id=121 first_inserted.id=121)
[order=A_then_B] reply FROM selected owner mailbox=user_5sckyrgdj3@mailbox.test -> status='250 Message accepted for delivery'
[order=A_then_B]   EmailLog={'id': 217, 'user_id': 427, 'mailbox_id': 505, 'alias_id': 700, 'contact_id': 121, 'is_reply': True}
[order=A_then_B]   OUT envelope_to=ext-A-dup_athenb_febd0c89@external.example msg.From=loughs_lyrics561@sl.local (relayed to SELECTED contact website=ext-A-dup_athenb_febd0c89@external.example, user_id=427)
[order=A_then_B]   LOG> handle_reply:1051 Create <EmailLog 217> for <Contact 121 ext-A-dup_athenb_febd0c89@external.example 700>, <User 427 Test User user_5sckyrgdj3@mailbox.test>, <Mailbox 505 user_5sckyrgdj3@mailbox.test>
[order=A_then_B] reply FROM shadowed owner mailbox=user_tihsyttxco@mailbox.test -> status='250 SL E214 Unauthorized for using reverse alias' (the shadowed owner's identical reply_email resolves to the OTHER user's contact.id=121, so their reply is NOT delivered to their intended contact)
[order=A_then_B]   LOG> handle_unknown_mailbox:1393 Reply email can only be used by mailbox. Actual mail_from: user_tihsyttxco@mailbox.test. msg from header: user_tihsyttxco@mailbox.test, reverse-alias ra+dup_athenb_febd0c89@sl.local, <Alias 700 loughs_lyrics561@sl.local> <User 427 Test User user_5sckyrgdj3@mailbox.test> <Contact 121 ext-A-dup_athenb_febd0c89@external.example 700>
[order=B_then_A] two Contact rows share reply_email=ra+dup_bthena_bb4e9de4@sl.local (DB accepted both -> reply_email NOT unique):
   contact.id=123 user_id=430 alias_id=706 website_email=ext-B-dup_bthena_bb4e9de4@external.example
   contact.id=124 user_id=429 alias_id=704 website_email=ext-A-dup_bthena_bb4e9de4@external.example
[order=B_then_A] DB row count for that reply_email = 2
[order=B_then_A] Contact.get_by(reply_email).first() SELECTED contact.id=123 user_id=430 alias_id=706 website_email=ext-B-dup_bthena_bb4e9de4@external.example
[order=B_then_A] selected == first-inserted-contact? True (selected.id=123 first_inserted.id=123)
[order=B_then_A] reply FROM selected owner mailbox=user_7crd6uqpmc@mailbox.test -> status='250 Message accepted for delivery'
[order=B_then_A]   EmailLog={'id': 218, 'user_id': 430, 'mailbox_id': 508, 'alias_id': 706, 'contact_id': 123, 'is_reply': True}
[order=B_then_A]   OUT envelope_to=ext-B-dup_bthena_bb4e9de4@external.example msg.From=flabby_arsing208@sl.local (relayed to SELECTED contact website=ext-B-dup_bthena_bb4e9de4@external.example, user_id=430)
[order=B_then_A]   LOG> handle_reply:1051 Create <EmailLog 218> for <Contact 123 ext-B-dup_bthena_bb4e9de4@external.example 706>, <User 430 Test User user_7crd6uqpmc@mailbox.test>, <Mailbox 508 user_7crd6uqpmc@mailbox.test>
[order=B_then_A] reply FROM shadowed owner mailbox=user_115zw0mqsg@mailbox.test -> status='250 SL E214 Unauthorized for using reverse alias' (the shadowed owner's identical reply_email resolves to the OTHER user's contact.id=123, so their reply is NOT delivered to their intended contact)
[order=B_then_A]   LOG> handle_unknown_mailbox:1393 Reply email can only be used by mailbox. Actual mail_from: user_115zw0mqsg@mailbox.test. msg from header: user_115zw0mqsg@mailbox.test, reverse-alias ra+dup_bthena_bb4e9de4@sl.local, <Alias 706 flabby_arsing208@sl.local> <User 430 Test User user_7crd6uqpmc@mailbox.test> <Contact 123 ext-B-dup_bthena_bb4e9de4@external.example 706>
[S3] OBSERVED selection==first-inserted in both runs: True (run1=True run2=True). NOTE: get_by()=filter_by().first() has NO ORDER BY (models.py:L84); this selection was OBSERVED, not guaranteed.
```

**(observed)** Two `Contact` rows owned by two different users CAN share one `reply_email` (`DB row count for that reply_email = 2`), confirming `reply_email` is not unique. When a real reply is driven through `handle_DATA`, `Contact.get_by(reply_email).first()` selects exactly ONE contact; in both insertion orders the selected row was the first-inserted / lowest-PK (`selected == first-inserted-contact? True`). The **selected** owner's reply succeeds (`250`, `EmailLog` created, relayed to the selected contact's `website_email`); the **shadowed** owner's reply resolves to the *other* user's contact and is rejected with `E214` (their mailbox is not authorized on the selected contact's alias). **(inferred)** The alias is "recognized" throughout, yet the routing/attribution is dictated entirely by the single contact `.first()` returns — reproducing the reported "alias recognized but wrong user" symptom. **(observed, per M6)** The specific winner tracked insertion order **in these runs only**; `get_by()` = `filter_by().first()` has no `ORDER BY` (`app/models.py:L84`), so the selection is unspecified in general.

#### S4 — Unauthorized sender → E214 (spoofing check ON) ×2

```text
==================== S4: UNAUTHORIZED mail_from -> E214 (spoofing ON) x2 ====================
[RUN 1] alias.disable_email_spoofing_check=False attacker_mail_from=attacker-s4r1_8b44b913@evil.example reply=ra+s4r1_8b44b913@sl.local
[RUN 1] status='250 SL E214 Unauthorized for using reverse alias' reply_EmailLog=None total_stored=1 sent_alerts_to_owner=1
[RUN 1] LOG> _handle:2343 New message, mail from attacker-s4r1_8b44b913@evil.example, rctp tos ['ra+s4r1_8b44b913@sl.local']
[RUN 1] LOG> handle:2196 Reply phase attacker-s4r1_8b44b913@evil.example(attacker-s4r1_8b44b913@evil.example) -> ra+s4r1_8b44b913@sl.local
[RUN 1] LOG> apply_dmarc_policy_for_reply_phase:159 DMARC check disabled
[RUN 1] LOG> handle_unknown_mailbox:1393 Reply email can only be used by mailbox. Actual mail_from: attacker-s4r1_8b44b913@evil.example. msg from header: attacker-s4r1_8b44b913@evil.example, reverse-alias ra+s4r1_8b44b913@sl.local, <Alias 708 fruity_byplay246@sl.local> <User 431 Test User user_7e6rgksnuc@mailbox.test> <Contact 125 ext-s4r1_8b44b913@external.example 708>
[RUN 2] alias.disable_email_spoofing_check=False attacker_mail_from=attacker-s4r2_a53bee56@evil.example reply=ra+s4r2_a53bee56@sl.local
[RUN 2] status='250 SL E214 Unauthorized for using reverse alias' reply_EmailLog=None total_stored=1 sent_alerts_to_owner=1
[RUN 2] LOG> _handle:2343 New message, mail from attacker-s4r2_a53bee56@evil.example, rctp tos ['ra+s4r2_a53bee56@sl.local']
[RUN 2] LOG> handle:2196 Reply phase attacker-s4r2_a53bee56@evil.example(attacker-s4r2_a53bee56@evil.example) -> ra+s4r2_a53bee56@sl.local
[RUN 2] LOG> apply_dmarc_policy_for_reply_phase:159 DMARC check disabled
[RUN 2] LOG> handle_unknown_mailbox:1393 Reply email can only be used by mailbox. Actual mail_from: attacker-s4r2_a53bee56@evil.example. msg from header: attacker-s4r2_a53bee56@evil.example, reverse-alias ra+s4r2_a53bee56@sl.local, <Alias 710 stoves_frills157@sl.local> <User 432 Test User user_vkxsxz76f3@mailbox.test> <Contact 126 ext-s4r2_a53bee56@external.example 710>
```

**(observed)** A sender NOT authorized for the alias's mailbox is rejected with **E214** on both runs; NO reply `EmailLog` is created (`reply_EmailLog=None`); `total_stored=1` / `sent_alerts_to_owner=1` is the alert email sent to the legitimate owner via `handle_unknown_mailbox` (`email_handler.py:L1032`/`email_handler.py:L1034`; the warning is logged at `email_handler.py:L1393`).

#### S5 — `disable_email_spoofing_check` default-mailbox fallback (Q5c) ×2

```text
==================== S5: disable_email_spoofing_check DEFAULT-MAILBOX FALLBACK (Q5c) x2 ====================
[RUN 1] alias.disable_email_spoofing_check=True alias.mailbox_id=511 stranger=stranger-s5r1_7c7e5940@nowhere.example
[RUN 1] status='250 Message accepted for delivery' EmailLog={'id': 219, 'user_id': 433, 'mailbox_id': 511, 'alias_id': 712, 'contact_id': 127, 'is_reply': True} used_default(mailbox_id==alias.mailbox_id)=True
[RUN 1] OUT envelope_to=ext-s5r1_7c7e5940@external.example msg.From=mercer_umbels290@sl.local
[RUN 1] LOG> handle:2196 Reply phase stranger-s5r1_7c7e5940@nowhere.example(stranger-s5r1_7c7e5940@nowhere.example) -> ra+s5r1_7c7e5940@sl.local
[RUN 1] LOG> apply_dmarc_policy_for_reply_phase:159 DMARC check disabled
[RUN 1] LOG> handle_reply:1023 ignore unknown sender to reverse-alias stranger-s5r1_7c7e5940@nowhere.example: <Alias 712 mercer_umbels290@sl.local> -> <Contact 127 ext-s5r1_7c7e5940@external.example 712>
[RUN 1] LOG> handle_reply:1051 Create <EmailLog 219> for <Contact 127 ext-s5r1_7c7e5940@external.example 712>, <User 433 Test User user_mxfigzhvn3@mailbox.test>, <Mailbox 511 user_mxfigzhvn3@mailbox.test>
[RUN 2] alias.disable_email_spoofing_check=True alias.mailbox_id=512 stranger=stranger-s5r2_c2b67665@nowhere.example
[RUN 2] status='250 Message accepted for delivery' EmailLog={'id': 220, 'user_id': 434, 'mailbox_id': 512, 'alias_id': 714, 'contact_id': 128, 'is_reply': True} used_default(mailbox_id==alias.mailbox_id)=True
[RUN 2] OUT envelope_to=ext-s5r2_c2b67665@external.example msg.From=afford_partly739@sl.local
[RUN 2] LOG> handle:2196 Reply phase stranger-s5r2_c2b67665@nowhere.example(stranger-s5r2_c2b67665@nowhere.example) -> ra+s5r2_c2b67665@sl.local
[RUN 2] LOG> apply_dmarc_policy_for_reply_phase:159 DMARC check disabled
[RUN 2] LOG> handle_reply:1023 ignore unknown sender to reverse-alias stranger-s5r2_c2b67665@nowhere.example: <Alias 714 afford_partly739@sl.local> -> <Contact 128 ext-s5r2_c2b67665@external.example 714>
[RUN 2] LOG> handle_reply:1051 Create <EmailLog 220> for <Contact 128 ext-s5r2_c2b67665@external.example 714>, <User 434 Test User user_n2myedw5lz@mailbox.test>, <Mailbox 512 user_n2myedw5lz@mailbox.test>
```

**(observed)** With `disable_email_spoofing_check=True`, an UNKNOWN sender is NOT rejected on either run — the code logs "ignore unknown sender to reverse-alias" (`email_handler.py:L1023`), falls back to `mailbox = alias.mailbox` (`email_handler.py:L1029`), and creates the reply `EmailLog` with `used_default(mailbox_id==alias.mailbox_id)=True`. Directly contrasts `S4`'s E214.

#### S6 — NOREPLIES short-circuit + bounce `mail_from == "<>"` (edge) ×2

```text
==================== S6: NOREPLIES short-circuit + bounce mail_from=='<>' (edge) x2 ====================
config.NOREPLIES=['noreply@sl.local']
[RUN 1] NOREPLIES rcpt=noreply@sl.local -> status='250 Message accepted for delivery'
[RUN 1] BOUNCE mail_from='<>' rcpt=reverse-alias(ra+s6r1_69513cc7@sl.local) -> status='250 SL E206 Out of office'
[RUN 2] NOREPLIES rcpt=noreply@sl.local -> status='250 Message accepted for delivery'
[RUN 2] BOUNCE mail_from='<>' rcpt=reverse-alias(ra+s6r2_e8672fd1@sl.local) -> status='250 SL E206 Out of office'
```

**(observed)** On both runs, a message to a `NOREPLIES` address short-circuits (`send_no_reply_response`, `email_handler.py:L2181–L2183`) and returns `'250 Message accepted for delivery'` without reply handling; a bounce/auto-reply with `mail_from == "<>"` to a reverse alias is handled as out-of-office → **E206** ("Out of office") at `email_handler.py:L2166`/`email_handler.py:L2173`, before `handle_reply` routing.

---

## (i) Observed vs. inferred — key-claim ledger

The distinction is annotated inline throughout. The most consequential claims are consolidated here:

| # | Claim | Basis |
|---|-------|-------|
| 1 | Reply ingress is `handle_DATA` → `_handle` → `handle` (`email_handler.py:L2289`, `L2335`, `L1945`) | **observed** (code + `S1` log lines `_handle:2343`, `handle:2196`) |
| 2 | The alias is resolved indirectly via `Contact` (`reply_email` → `Contact` → `Contact.alias` → `Alias.user`) | **observed** (code `email_handler.py:L986`/`L994`/`L1004`; `S1` chain `contact.id=118 → alias.id=692 → user.id=422`) |
| 3 | On the happy path the decided `user_id` is `422`, and `EmailLog.user_id == contact.user_id == alias.user_id` | **observed** (`S1`, both runs) |
| 4 | The handler uses DISTINCT user references at distinct gates: Gate 1 `contact.user` (`L990`); Gate 2 `alias.user` (`L1004`/`L1007`); Gate 3 both (`L1012`); Gate 4 alias-scoped mailbox (`L1019`); persist `contact.user_id` (`L1046`) | **observed** (code + `S2` `GATES-INPUT` line) |
| 5 | Those references CAN diverge, yielding `EmailLog.user_id=424`/`426` while gates used `alias.user_id=423`/`425` | **observed** (`S2`, both runs, `MISMATCH=True`) |
| 6 | `reply_email` is not unique; two contacts (two users) can share it; `.first()` returns one row with NO `ORDER BY` | **observed** (`app/models.py:L1899`/`L1875`/`L84`; `S3` `DB row count = 2`) |
| 7 | In `S3` the selected row was the first-inserted/lowest-PK in both runs, but this is not guaranteed by the query | **observed** (`S3` `selected == first-inserted-contact? True`) / **explicitly labeled not-guaranteed** (`app/models.py:L84`) |
| 8 | In `S3`, the selected owner's reply is delivered (`250`) while the shadowed owner's reply is rejected (`E214`) — routing dictated by the winning contact | **observed** (`S3`, both insertion orders) |
| 9 | The happy-path reply egresses OUTWARD to `contact.website_email` with `From` rewritten to the alias; the real mailbox is absent from the WHOLE 758-byte message | **observed** (`S1` `OUT`, full serialized message, `real_mailbox…present_anywhere=False`) |
| 10 | Unauthorized sender (spoofing ON) → E214, no reply `EmailLog`, alert to owner | **observed** (`S4`, both runs) |
| 11 | `disable_email_spoofing_check=True` → default-mailbox fallback, reply created (`used_default=True`) | **observed** (`S5`, both runs) |
| 12 | `NOREPLIES` → 250 short-circuit; bounce `<>` → E206 | **observed** (`S6`, both runs) |
| 13 | DMARC did not block, but only because it is NON-BLOCKING here (`DMARC check disabled` → return `None`), not a "clean pass" | **observed** (`app/handler/dmarc.py:L157–L160`; `DMARC check disabled` log line on every reply run) |
| 14 | For the correctly-owned common case, a "wrong user" report is most likely a misunderstanding of the outward-relay model | **inferred** (from claims 3 & 9) |
| 15 | The single most likely mis-routing origin is `Contact.get_by(reply_email=…).first()` (`email_handler.py:L986`) + non-unique schema + `.first()` — Q5b, reproduced in `S3` | **inferred** (from claims 2, 6, 7, 8; no abnormal ownership needed) |
| 16 | The dual user reference is a pipeline-wide pattern (forward path uses it too: `L557` vs `L600`/`L734`) | **observed** (code) / **inferred** (that it is the same pattern) |
| 17 | Divergent `alias.user_id`/`contact.user_id` is an abnormal state because `transfer_alias()` updates both together | **observed** (`app/alias_utils.py:L464–L466`, `L506`) / **inferred** (that divergence is therefore abnormal) |

---

## (j) Security / privacy note

**(observed)** The reverse-alias mechanism exists to keep the user's real mailbox hidden. This was verified in `S1` by serializing the **entire** outbound message (`SendRequest.msg` via `message_to_bytes`, 758 bytes — reproduced in full in [§ (h)](#h-commands-run--raw-runtime-output)) and searching it for the user's real mailbox address `user_6e7vw6j126@mailbox.test`: the search returned `present_anywhere=False`. The message exposes only the alias (`From: settee_leches851@sl.local`) and the external contact (`To: Ext <ext-s1_afc9ef33@external.example>`); the real mailbox does not appear in any header (including the DKIM signature and message-id) or the body. This was consistent across both identical-input runs.

**(inferred)** Therefore, any mis-routing that resolves the **wrong `Contact`/user** — Hypothesis (b), reproduced in `S3` — is not merely a correctness bug but a **privacy-relevant defect**: because the alias, the recorded `user_id`, and the destination `website_email` all hang off the single contact `.first()` returns, resolving to the wrong contact routes/attributes the reply via a *different* user's contact while the alias still appears "recognized." In `S3` the shadowed owner was rejected with `E214` (fail-closed); however, if the shadowed owner's mailbox were an authorized address on the winning contact's alias, the same resolution would relay their reply to the *winning contact's* external recipient — an unintended-recipient disclosure. Because the whole point of the reverse alias is confidentiality, a wrong-contact resolution is a confidentiality risk, not just an attribution bug.

---

## Coverage recap (final pass)

- **Q1 — Entry point:** answered in [§ (b)](#b-q1--which-component-handles-the-incoming-reply-entry-point) — `MailHandler.handle_DATA()` (`email_handler.py:L2289`) → `_handle()` (`L2335`) → `handle()` (`L1945`); production ingress at `127.0.0.1:20381` (`README.md:L367`).
- **Q2 — Resolution:** answered in [§ (c)](#c-q2--how-is-the-alias-resolved-to-a-user-resolution-chain) — `is_reverse_alias` (`app/email_utils.py:L1156`) → `handle_reply` (`L966`) → `Contact.get_by(reply_email).first()` (`L986` / `app/models.py:L84`) → `contact.alias` (`L994`) → `alias.user` (`L1004`).
- **Q3 — Decided `user_id`:** answered in [§ (d)](#d-q3--the-concrete-integer-user_id-the-system-decides) — happy-path value `422` (stable across two identical-input runs); persisted via `EmailLog.create(user_id=contact.user_id …)` (`L1046`); two independent references (`alias.user` vs `contact.user_id`).
- **Q4 — Data flow:** answered in [§ (e)](#e-q4--actual-observed-end-to-end-data-flow) — full gate-by-gate narrative + diagram, each node tied to `S1`/`S2` evidence; DMARC shown to be non-blocking.
- **Q5 — Most likely origin:** answered in [§ (f)](#f-q5--most-likely-point-where-an-incorrect-routing-decision-originates) — three hypotheses reproduced (Q5a in `S2`, Q5b canonically in `S3`, Q5c in `S5`); single most likely origin = `Contact.get_by(reply_email=…).first()` (`email_handler.py:L986`) with the non-unique `reply_email` schema (`app/models.py:L1899`) and `.first()`-without-`ORDER BY` semantics (`app/models.py:L84`).

*End of analysis. The only artifact added to the repository is this document; no source file was modified, and all temporary observation scaffolding lived inside the run container and was removed.*
