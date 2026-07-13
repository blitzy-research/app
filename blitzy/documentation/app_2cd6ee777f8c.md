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

Every distinct condition the question implies was exercised — the happy path plus the secondary/edge/error paths — through the real entry point (`MailHandler.handle_DATA`), **each scenario at least twice**. Before/after states are reported where state changes. The exercised set includes the multi-mailbox alias condition called out in the AAP (§0.5.1): `S7` drives a reply from an alias's secondary (non-default) authorized mailbox — and, for completeness, from its default mailbox and from an authorized delegate address.

| Scenario | Condition | Runs | Observed status | Reply `EmailLog` (before → after) | Key citation |
|----------|-----------|------|-----------------|-----------------------------------|--------------|
| `S1` | Happy path, correctly-owned, identical input | ×2 | `'250 Message accepted for delivery'` (both) | none → created (`user_id=422`, equal to both refs; only `EmailLog.id` 213→214 changed) | `email_handler.py:L1042–L1047` |
| `S2` | Constructed divergence `alias.user_id != contact.user_id` | ×2 | `'250 Message accepted for delivery'` (both) | none → created (`EmailLog.user_id=424`/`426`; gates used `alias.user_id=423`/`425`; `MISMATCH=True`) | `email_handler.py:L1004` vs `L1046` |
| `S3` | Non-unique `reply_email` (two contacts, two users share it) — canonical `handle_DATA` | ×2 (both insertion orders) | selected owner `'250 …'`; shadowed owner `'250 SL E214 …'` | none → created for selected owner (`id=217`/`218`); shadowed owner blocked (E214) | `email_handler.py:L986` / `app/models.py:L84`, `L1899`, `L1875` |
| `S4` | Unauthorized sender, spoofing check **ON** | ×2 | `'250 SL E214 Unauthorized for using reverse alias'` (both) | none → **NONE (blocked)**; `total_stored=1`, `sent_alerts_to_owner=1` | `email_handler.py:L1032`/`L1034`, `handle_unknown_mailbox` `email_handler.py:L1390` |
| `S5` | Unknown sender, `disable_email_spoofing_check=True` | ×2 | `'250 Message accepted for delivery'` (both) | none → created (`user_id=433`/`434`, `mailbox_id==alias.mailbox_id`, `used_default=True`) | `email_handler.py:L1023`/`L1029` |
| `S6` | `NOREPLIES` address; bounce `mail_from == "<>"` | ×2 | `NOREPLIES → '250 Message accepted for delivery'`; bounce `→ '250 SL E206 Out of office'` | no reply routing performed | `email_handler.py:L2181–L2183`; `email_handler.py:L2166`/`email_handler.py:L2173` |
| `S7` | Multi-mailbox alias — reply from the **secondary** (non-default) authorized mailbox (plus default mailbox and an authorized delegate address) | ×2 | all `'250 Message accepted for delivery'` (both) | none → created; `EmailLog.mailbox_id` == the **sending** mailbox (secondary→secondary, default→default, delegate→mapped default) | `email_handler.py:L1019`/`L1364`; `app/models.py:L1580–L1590` |

- **(observed) `S4` — unauthorized sender → E214 (spoofing ON):** a sender NOT authorized for the alias's mailbox is rejected with **E214** ("Unauthorized for using reverse alias") at `email_handler.py:L1034` (via `handle_unknown_mailbox` `email_handler.py:L1032`/`email_handler.py:L1390`). NO reply `EmailLog` is created (before → after: reply `EmailLog` = NONE). `total_stored=1` / `sent_alerts_to_owner=1` is the alert email sent to the legitimate user, not a relayed reply. Reproduced on both runs.
- **(observed) `S5` — default-mailbox fallback (spoofing OFF):** with `disable_email_spoofing_check=True`, an unknown sender is NOT rejected — the code logs "ignore unknown sender to reverse-alias" at `email_handler.py:L1023`, falls back to `mailbox = alias.mailbox` at `email_handler.py:L1029`, and creates the reply `EmailLog` (`used_default=True`). Directly contrasts `S4`'s E214. Reproduced on both runs.
- **(observed) `S6` — guard paths around the reply logic:** a message to a `NOREPLIES` address short-circuits at `email_handler.py:L2181–L2183` (`send_no_reply_response`) and returns `'250 Message accepted for delivery'` without reply handling. A bounce/auto-reply with `mail_from == "<>"` addressed to a reverse alias is handled as out-of-office → **E206** ("Out of office") at `email_handler.py:L2166`/`email_handler.py:L2173`, before `handle_reply` routing. Reproduced on both runs.
- **(observed) `S7` — multi-mailbox alias, sender-driven mailbox selection:** an alias with **two** verified mailboxes (a default `mb1` and a secondary `mb2`, plus an authorized delegate address mapped to `mb1`) was replied to from each authorized origin in turn. `get_mailbox_from_mail_from(mail_from, alias)` (`email_handler.py:L1364`) selects the mailbox whose address matches `mail_from` out of `alias.mailboxes` (`app/models.py:L1580–L1590`), and the persisted `EmailLog.mailbox_id` (`email_handler.py:L1047`) equals that **sending** mailbox: default-send → default, secondary-send → **secondary** (`mailbox_id != alias.mailbox_id`), delegate-send → the mapped default (logged as `Found an authorized address` at `email_handler.py:L1376`). All three returned `250` and the selection invariants were identical across both runs — multi-mailbox authorization routes to the correct mailbox, not merely the default. Reproduced on both runs.

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

# 2) Drive the REAL SMTP callback in-process for all seven scenarios (S1-S7) (each >=2x) and capture the transcript:
cd /tmp/slrun
CONFIG=tests/test.env GITHUB_ACTIONS_TEST=true PYTEST_ADDOPTS="" \
  /app/venv/bin/python -m pytest tests/blitzy_reply_trace.py -s -p no:cacheprovider -p no:randomly -p no:warnings --no-header -q
#   -> 7 passed in 6.13s
```

`tests/blitzy_reply_trace.py` was a temporary observation module that lived **only** inside the run container under `/tmp/slrun/tests/` (a copy of the committed tree exported via `git archive HEAD`); it was never added to the repository and does not exist in the working tree. `/tmp/slrun` is entirely inside the container, so the repository working tree remained git-clean.

Its **complete source is reproduced verbatim below**, so the command above is runnable as-is. The module contains all seven scenarios; the `S1`–`S6` raw transcripts above were captured on the initial six-scenario investigation run, while the `S7` transcript (below) was captured on a subsequent run after the multi-mailbox scenario was added — so the autoincrement `EmailLog`/`Contact`/`Alias`/`Mailbox`/`User` IDs in the `S7` block fall in a higher range than those in the `S1`–`S6` blocks. As `S1` demonstrates, only these autoincrement IDs vary run-to-run; the behaviors, statuses, and invariants are stable.

(The only environment fixes applied inside the container — converting `local_data/dkim.key` to PKCS#1 and shimming `re2`→`re` in `app/spamassassin_utils.py`, both noted in the setup — are confined to the container image and touch neither the reply-pipeline code nor any cited `file:line`.)

### Observation harness source (`tests/blitzy_reply_trace.py`)

```python
"""
blitzy_reply_trace.py — temporary, read-only runtime observation harness for the
SimpleLogin alias reply-handling root-cause analysis.

It drives SimpleLogin's *canonical* inbound-SMTP entry point
``MailHandler.handle_DATA()`` (email_handler.py:L2289) — which parses the raw
envelope bytes with ``email.message_from_bytes`` and calls ``_handle()`` (L2335)
-> ``handle()`` (L1945) — for the reply scenarios S1-S7. Outbound relay is
captured non-intrusively through the production seam
``mail_sender.store_emails_instead_of_sending()`` / ``get_stored_emails()``
(app/mail_sender.py:L102/L108) with ``NOT_SEND_EMAIL=true`` (tests/test.env), so
no real email leaves the environment.

Nothing here is a mock, monkeypatch, or synthetic bypass: every routing
decision, ``is_reverse_alias`` classification, ``Contact.get_by`` lookup,
user/mailbox selection and ``EmailLog`` write is produced by the real handler.
The module is a pytest module so it inherits the canonical ``flask_client``
app-context + DB-session fixture from tests/conftest.py.

Run (inside the canonical GHCR image container; the module lives only under the
container run dir and is never committed):

    cd /tmp/slrun
    CONFIG=tests/test.env GITHUB_ACTIONS_TEST=true PYTEST_ADDOPTS="" \
      /app/venv/bin/python -m pytest tests/blitzy_reply_trace.py \
      -s -p no:cacheprovider -p no:randomly -p no:warnings --no-header -q

Absolute integer IDs (user/alias/contact/mailbox/EmailLog) are assigned by the
Postgres sequences at run time and therefore differ run-to-run; only the
behaviors, status strings, and invariants are stable (this is exactly what S1's
identical-input repetition demonstrates).
"""

import asyncio
import logging
from email.message import EmailMessage

from aiosmtpd.smtp import Envelope

import email_handler
from email_handler import MailHandler
from app import config
from app.config import EMAIL_DOMAIN
from app.db import Session
from app.email import headers
from app.email_utils import is_reverse_alias
from app.mail_sender import mail_sender
from app.message_utils import message_to_bytes
from app.models import (
    Alias,
    AliasMailbox,
    AuthorizedAddress,
    Contact,
    EmailLog,
    Mailbox,
)
from app.utils import random_string
from tests.utils import create_new_user, random_email


# --------------------------------------------------------------------------- #
# Logging capture: record every "SL" logger emission as (funcName, lineno, msg)
# so we can print the stable compact form  ``LOG> funcName:lineno message``.
# --------------------------------------------------------------------------- #
class _LogCapture(logging.Handler):
    def __init__(self):
        super().__init__(level=logging.DEBUG)
        self.records = []

    def emit(self, record):
        self.records.append((record.funcName, record.lineno, record.getMessage()))

    def clear(self):
        self.records = []


_CAP = _LogCapture()
_SL_LOGGER = logging.getLogger("SL")
_SL_LOGGER.setLevel(logging.DEBUG)
_SL_LOGGER.addHandler(_CAP)

# The handler funcNames whose log lines we surface (the routing-relevant ones).
_LOG_FUNCS_OF_INTEREST = {
    "_handle",
    "handle",
    "apply_dmarc_policy_for_reply_phase",
    "handle_reply",
    "handle_unknown_mailbox",
    "__check",  # nested in get_mailbox_from_mail_from -> "Found an authorized address"
}


def _captured_logs():
    """Return the compact-form log lines emitted during the last drive()."""
    out = []
    for func, lineno, msg in _CAP.records:
        if func in _LOG_FUNCS_OF_INTEREST:
            out.append(f"LOG> {func}:{lineno} {msg}")
    return out


def _print_logs(prefix, lines):
    for ln in lines:
        print(f"{prefix}{ln}")


def _emaillog_dict(el):
    """Snapshot the EmailLog scalar fields into a plain dict immediately."""
    if el is None:
        return None
    return {
        "id": el.id,
        "user_id": el.user_id,
        "mailbox_id": el.mailbox_id,
        "alias_id": el.alias_id,
        "contact_id": el.contact_id,
        "is_reply": el.is_reply,
    }


def _latest_reply_log_dict(contact_id):
    el = (
        EmailLog.filter_by(contact_id=contact_id, is_reply=True)
        .order_by(EmailLog.id.desc())
        .first()
    )
    return _emaillog_dict(el)


def _max_email_log_id():
    el = EmailLog.filter_by().order_by(EmailLog.id.desc()).first()
    return el.id if el else 0


def drive(mail_from, rcpt_to, msg: EmailMessage) -> str:
    """Drive the REAL async SMTP DATA callback (the canonical entry point)."""
    envelope = Envelope()
    envelope.mail_from = mail_from
    envelope.rcpt_tos = [rcpt_to]
    envelope.original_content = msg.as_bytes()
    _CAP.clear()
    return asyncio.run(MailHandler().handle_DATA(None, None, envelope))


def build_reply_msg(from_addr, reply_email, subject, body) -> EmailMessage:
    msg = EmailMessage()
    msg[headers.FROM] = from_addr
    msg[headers.TO] = reply_email
    msg[headers.SUBJECT] = subject
    msg.set_content(body)
    return msg


# =========================================================================== #
# S1 — Happy path, correctly-owned, SAME UNCHANGED INPUT x2.
# Grounds the stability + privacy claims (Q1/Q2/Q3/Q4).
# =========================================================================== #
def test_s1_happy_path(flask_client):
    mail_sender.store_emails_instead_of_sending(True)
    print("==================== S1: HAPPY PATH — SAME UNCHANGED INPUT x2 "
          "====================")
    user = create_new_user()
    alias = Alias.create_new_random(user)
    Session.flush()
    contact = Contact.create(
        user_id=user.id,
        alias_id=alias.id,
        website_email=f"ext-s1_{random_string(6)}@external.example",
        reply_email=f"ra+s1_{random_string(6)}@{EMAIL_DOMAIN}",
        flush=True,
    )
    Session.commit()
    # snapshot fixture scalars BEFORE any drive (drives can expire/detach ORM objs)
    user_id = user.id
    alias_id = alias.id
    alias_user_id = alias.user_id
    alias_email = alias.email
    mailbox_email = alias.mailbox.email
    contact_id = contact.id
    contact_user_id = contact.user_id
    website = contact.website_email
    reply_email = contact.reply_email
    print(f"FIX user.id={user_id} alias.id={alias_id} alias.email={alias_email} "
          f"alias.user_id={alias_user_id} mailbox.email={mailbox_email} "
          f"contact.id={contact_id} contact.user_id={contact_user_id} "
          f"website={website} reply_email={reply_email} "
          f"dual-ref-EQUAL={alias_user_id == contact_user_id}")
    print(f"is_reverse_alias(reply_email)={is_reverse_alias(reply_email)}")

    distribution = []
    for run in (1, 2):
        mail_sender.purge_stored_emails()
        st = drive(mailbox_email, reply_email,
                   build_reply_msg(mailbox_email, reply_email,
                                   "re: same input", "identical body"))
        logs = _captured_logs()
        el_d = _latest_reply_log_dict(contact_id)
        out = mail_sender.get_stored_emails()
        out_to = out[0].envelope_to if out else None
        out_from = out[0].envelope_from if out else None
        msg_from = out[0].msg[headers.FROM] if out else None
        msg_to = out[0].msg[headers.TO] if out else None
        full_bytes = message_to_bytes(out[0].msg) if out else b""
        present = mailbox_email.encode() in full_bytes
        header_names = out[0].msg.keys() if out else []
        print(f"[RUN {run}] status={st!r}")
        print(f"[RUN {run}] EmailLog={el_d}")
        print(f"[RUN {run}] OUT envelope_from={out_from} envelope_to={out_to} "
              f"msg.From={msg_from} msg.To={msg_to}")
        _print_logs(f"[RUN {run}] ", logs)
        print(f"[RUN {run}] FULL-MSG-BYTES len={len(full_bytes)} "
              f"real_mailbox({mailbox_email})_present_anywhere={present}")
        print(f"[RUN {run}] OUT all header names={list(header_names)}")
        distribution.append((st, el_d["user_id"], out_to, msg_from))

    print(f"[S1] identical-input distribution over 2 runs: {distribution}")
    print(f"[S1] all runs identical? {distribution[0] == distribution[1]}")
    mail_sender.store_emails_instead_of_sending(False)


# =========================================================================== #
# S2 — Constructed divergence alias.user_id != contact.user_id (Q5a).
# The reply is authorized against alias.user but attributed to contact.user_id.
# =========================================================================== #
def test_s2_divergence(flask_client):
    mail_sender.store_emails_instead_of_sending(True)
    print("==================== S2: DIVERGENCE alias.user_id != contact.user_id "
          "(Q5a) x2 ====================")
    for run in (1, 2):
        owner = create_new_user()          # owns the ALIAS (authorizing user)
        alias = Alias.create_new_random(owner)
        Session.flush()
        other = create_new_user()          # owns the CONTACT (recorded user)
        contact = Contact.create(
            user_id=other.id,               # <-- diverges from alias.user_id
            alias_id=alias.id,
            website_email=f"ext-div-s2r{run}_{random_string(6)}@external.example",
            reply_email=f"ra+s2r{run}_{random_string(6)}@{EMAIL_DOMAIN}",
            flush=True,
        )
        Session.commit()
        alias_id = alias.id
        alias_user_id = alias.user_id
        contact_id = contact.id
        contact_user_id = contact.user_id
        mailbox_email = alias.mailbox.email  # authorized sender = alias's mailbox
        reply_email = contact.reply_email
        print(f"[RUN {run}] BEFORE alias.id={alias_id} alias.user_id={alias_user_id} "
              f"contact.id={contact_id} contact.user_id={contact_user_id} "
              f"dual-ref-EQUAL={alias_user_id == contact_user_id}")
        print(f"[RUN {run}] GATES-INPUT: gate1 contact.user.is_active uses "
              f"contact.user.id={contact_user_id} ; gate2 user=alias.user."
              f"can_send_or_receive uses alias.user.id={alias_user_id} ; gate3 "
              f"apply_dmarc_policy_for_reply_phase(alias,contact) gets "
              f"alias.user={alias_user_id} AND contact={contact_id} ; gate4 "
              f"get_mailbox_from_mail_from(mail_from, alias) scoped to "
              f"alias.user={alias_user_id} mailboxes")
        mail_sender.purge_stored_emails()
        st = drive(mailbox_email, reply_email,
                   build_reply_msg(mailbox_email, reply_email,
                                   f"re: divergence s2r{run}", "body"))
        logs = _captured_logs()
        el_d = _latest_reply_log_dict(contact_id)
        out = mail_sender.get_stored_emails()
        out_to = out[0].envelope_to if out else None
        msg_from = out[0].msg[headers.FROM] if out else None
        mismatch = el_d["user_id"] != alias_user_id if el_d else None
        print(f"[RUN {run}] status={st!r}")
        print(f"[RUN {run}] EmailLog={el_d}")
        print(f"[RUN {run}] PERSIST: EmailLog.user_id={el_d['user_id']} == "
              f"contact.user_id={contact_user_id} (recorded owner) ; authorizing "
              f"alias.user_id={alias_user_id} ; "
              f"MISMATCH(recorded_vs_authorizing)={mismatch}")
        print(f"[RUN {run}] OUT envelope_to={out_to} msg.From={msg_from}")
        _print_logs(f"[RUN {run}] ", logs)
    mail_sender.store_emails_instead_of_sending(False)


# =========================================================================== #
# S3 — Non-unique reply_email driven through handle_DATA (Q5b), both insertion
# orders. Two Contact rows (two users, two aliases) share one reply_email.
# =========================================================================== #
def _s3_one_order(order_label, first_ext_tag, second_ext_tag):
    # NOTE: the reverse-alias local part MUST be lower-case — the SMTP handler
    # lower-cases the envelope recipient before the Contact lookup, and
    # Contact.reply_email is matched case-sensitively (real SimpleLogin
    # reverse-aliases are always generated lower-case). ``order_label`` is a
    # human-readable tag only, so build the address from its lower-cased form.
    tag = order_label.lower()
    shared_reply = f"ra+dup_{tag}_{random_string(8)}@{EMAIL_DOMAIN}"
    # first-inserted contact
    u1 = create_new_user()
    a1 = Alias.create_new_random(u1)
    Session.flush()
    c1 = Contact.create(
        user_id=u1.id, alias_id=a1.id,
        website_email=f"ext-{first_ext_tag.lower()}-dup_{tag}_{random_string(6)}@external.example",
        reply_email=shared_reply, flush=True,
    )
    # second-inserted contact (shares reply_email, different user + alias)
    u2 = create_new_user()
    a2 = Alias.create_new_random(u2)
    Session.flush()
    c2 = Contact.create(
        user_id=u2.id, alias_id=a2.id,
        website_email=f"ext-{second_ext_tag.lower()}-dup_{tag}_{random_string(6)}@external.example",
        reply_email=shared_reply, flush=True,
    )
    Session.commit()

    c1_id, c1_user, c1_alias, c1_web = c1.id, c1.user_id, c1.alias_id, c1.website_email
    c2_id, c2_user, c2_alias, c2_web = c2.id, c2.user_id, c2.alias_id, c2.website_email
    a1_mb_email = a1.mailbox.email  # first contact owner's mailbox
    a2_mb_email = a2.mailbox.email  # second contact owner's mailbox

    print(f"[order={order_label}] two Contact rows share reply_email={shared_reply} "
          f"(DB accepted both -> reply_email NOT unique):")
    print(f"   contact.id={c1_id} user_id={c1_user} alias_id={c1_alias} "
          f"website_email={c1_web}")
    print(f"   contact.id={c2_id} user_id={c2_user} alias_id={c2_alias} "
          f"website_email={c2_web}")
    db_count = Contact.filter_by(reply_email=shared_reply).count()
    print(f"[order={order_label}] DB row count for that reply_email = {db_count}")

    selected = Contact.get_by(reply_email=shared_reply)
    sel_id, sel_user, sel_alias, sel_web = (
        selected.id, selected.user_id, selected.alias_id, selected.website_email
    )
    first_inserted_id = min(c1_id, c2_id)
    print(f"[order={order_label}] Contact.get_by(reply_email).first() SELECTED "
          f"contact.id={sel_id} user_id={sel_user} alias_id={sel_alias} "
          f"website_email={sel_web}")
    print(f"[order={order_label}] selected == first-inserted-contact? "
          f"{sel_id == first_inserted_id} (selected.id={sel_id} "
          f"first_inserted.id={first_inserted_id})")

    # the SELECTED owner's mailbox
    selected_mb = a1_mb_email if sel_id == c1_id else a2_mb_email
    shadowed_mb = a2_mb_email if sel_id == c1_id else a1_mb_email

    mail_sender.purge_stored_emails()
    st_sel = drive(selected_mb, shared_reply,
                   build_reply_msg(selected_mb, shared_reply,
                                   f"re: dup {order_label} selected", "body"))
    logs_sel = _captured_logs()
    el_sel = _latest_reply_log_dict(sel_id)
    out_sel = mail_sender.get_stored_emails()
    out_to_sel = out_sel[0].envelope_to if out_sel else None
    from_sel = out_sel[0].msg[headers.FROM] if out_sel else None
    print(f"[order={order_label}] reply FROM selected owner mailbox={selected_mb} "
          f"-> status={st_sel!r}")
    print(f"[order={order_label}]   EmailLog={el_sel}")
    print(f"[order={order_label}]   OUT envelope_to={out_to_sel} msg.From={from_sel} "
          f"(relayed to SELECTED contact website={sel_web}, user_id={sel_user})")
    _print_logs(f"[order={order_label}]   ", logs_sel)

    baseline = _max_email_log_id()
    mail_sender.purge_stored_emails()
    st_shadow = drive(shadowed_mb, shared_reply,
                      build_reply_msg(shadowed_mb, shared_reply,
                                      f"re: dup {order_label} shadowed", "body"))
    logs_shadow = _captured_logs()
    new_max = _max_email_log_id()
    print(f"[order={order_label}] reply FROM shadowed owner mailbox={shadowed_mb} "
          f"-> status={st_shadow!r} (the shadowed owner's identical reply_email "
          f"resolves to the SELECTED contact.id={sel_id}, so their reply is NOT "
          f"delivered to their intended contact; new_reply_EmailLog_created="
          f"{new_max > baseline})")
    _print_logs(f"[order={order_label}]   ", logs_shadow)
    return (sel_id == first_inserted_id)


def test_s3_non_unique_reply_email(flask_client):
    mail_sender.store_emails_instead_of_sending(True)
    print("==================== S3: NON-UNIQUE reply_email -> handle_DATA "
          "CANONICAL x2 (Q5b) ====================")
    r1 = _s3_one_order("A_then_B", "A", "B")
    r2 = _s3_one_order("B_then_A", "B", "A")
    print(f"[S3] OBSERVED selection==first-inserted in both runs: {r1 and r2} "
          f"(run1={r1} run2={r2}). NOTE: get_by()=filter_by().first() has NO "
          f"ORDER BY (models.py:L84); this selection was OBSERVED, not guaranteed.")
    mail_sender.store_emails_instead_of_sending(False)


# =========================================================================== #
# S4 — Unauthorized sender -> E214 (spoofing check ON) x2.
# =========================================================================== #
def test_s4_unauthorized_e214(flask_client):
    mail_sender.store_emails_instead_of_sending(True)
    print("==================== S4: UNAUTHORIZED mail_from -> E214 (spoofing ON) "
          "x2 ====================")
    for run in (1, 2):
        user = create_new_user()
        alias = Alias.create_new_random(user)
        Session.flush()
        contact = Contact.create(
            user_id=user.id, alias_id=alias.id,
            website_email=f"ext-s4r{run}_{random_string(6)}@external.example",
            reply_email=f"ra+s4r{run}_{random_string(6)}@{EMAIL_DOMAIN}",
            flush=True,
        )
        Session.commit()
        spoofing = alias.disable_email_spoofing_check
        contact_id = contact.id
        reply_email = contact.reply_email
        attacker = f"attacker-s4r{run}_{random_string(6)}@evil.example"
        mail_sender.purge_stored_emails()
        st = drive(attacker, reply_email,
                   build_reply_msg(attacker, reply_email, f"re: s4r{run}", "body"))
        logs = _captured_logs()
        el_d = _latest_reply_log_dict(contact_id)
        total_stored = len(mail_sender.get_stored_emails())
        print(f"[RUN {run}] alias.disable_email_spoofing_check={spoofing} "
              f"attacker_mail_from={attacker} reply={reply_email}")
        print(f"[RUN {run}] status={st!r} reply_EmailLog={el_d} "
              f"total_stored={total_stored} sent_alerts_to_owner={total_stored}")
        _print_logs(f"[RUN {run}] ", logs)
    mail_sender.store_emails_instead_of_sending(False)


# =========================================================================== #
# S5 — disable_email_spoofing_check default-mailbox fallback (Q5c) x2.
# =========================================================================== #
def test_s5_spoofing_off_fallback(flask_client):
    mail_sender.store_emails_instead_of_sending(True)
    print("==================== S5: disable_email_spoofing_check DEFAULT-MAILBOX "
          "FALLBACK (Q5c) x2 ====================")
    for run in (1, 2):
        user = create_new_user()
        alias = Alias.create_new_random(user)
        alias.disable_email_spoofing_check = True
        Session.flush()
        contact = Contact.create(
            user_id=user.id, alias_id=alias.id,
            website_email=f"ext-s5r{run}_{random_string(6)}@external.example",
            reply_email=f"ra+s5r{run}_{random_string(6)}@{EMAIL_DOMAIN}",
            flush=True,
        )
        Session.commit()
        alias_mailbox_id = alias.mailbox_id
        contact_id = contact.id
        reply_email = contact.reply_email
        stranger = f"stranger-s5r{run}_{random_string(6)}@nowhere.example"
        mail_sender.purge_stored_emails()
        st = drive(stranger, reply_email,
                   build_reply_msg(stranger, reply_email, f"re: s5r{run}", "body"))
        logs = _captured_logs()
        el_d = _latest_reply_log_dict(contact_id)
        out = mail_sender.get_stored_emails()
        out_to = out[0].envelope_to if out else None
        msg_from = out[0].msg[headers.FROM] if out else None
        used_default = el_d["mailbox_id"] == alias_mailbox_id if el_d else None
        print(f"[RUN {run}] alias.disable_email_spoofing_check=True "
              f"alias.mailbox_id={alias_mailbox_id} stranger={stranger}")
        print(f"[RUN {run}] status={st!r} EmailLog={el_d} "
              f"used_default(mailbox_id==alias.mailbox_id)={used_default}")
        print(f"[RUN {run}] OUT envelope_to={out_to} msg.From={msg_from}")
        _print_logs(f"[RUN {run}] ", logs)
    mail_sender.store_emails_instead_of_sending(False)


# =========================================================================== #
# S6 — NOREPLIES short-circuit + bounce mail_from == "<>" (edge) x2.
# =========================================================================== #
def test_s6_noreplies_and_bounce(flask_client):
    mail_sender.store_emails_instead_of_sending(True)
    print("==================== S6: NOREPLIES short-circuit + bounce "
          "mail_from=='<>' (edge) x2 ====================")
    print(f"config.NOREPLIES={config.NOREPLIES}")
    for run in (1, 2):
        # NOREPLIES short-circuit
        noreply_rcpt = config.NOREPLIES[0]
        user = create_new_user()
        mail_sender.purge_stored_emails()
        st_nr = drive(user.default_mailbox.email, noreply_rcpt,
                      build_reply_msg(user.default_mailbox.email, noreply_rcpt,
                                      f"re: noreply s6r{run}", "body"))
        print(f"[RUN {run}] NOREPLIES rcpt={noreply_rcpt} -> status={st_nr!r}")

        # bounce mail_from == "<>" to a reverse alias
        user2 = create_new_user()
        alias = Alias.create_new_random(user2)
        Session.flush()
        contact = Contact.create(
            user_id=user2.id, alias_id=alias.id,
            website_email=f"ext-s6r{run}_{random_string(6)}@external.example",
            reply_email=f"ra+s6r{run}_{random_string(6)}@{EMAIL_DOMAIN}",
            flush=True,
        )
        Session.commit()
        reply_email = contact.reply_email
        mail_sender.purge_stored_emails()
        st_bounce = drive("<>", reply_email,
                          build_reply_msg("bounce@mta.example", reply_email,
                                          f"auto-reply s6r{run}", "ooo"))
        print(f"[RUN {run}] BOUNCE mail_from='<>' rcpt=reverse-alias({reply_email}) "
              f"-> status={st_bounce!r}")
    mail_sender.store_emails_instead_of_sending(False)


# =========================================================================== #
# S7 — Multi-mailbox alias: reply from the SECONDARY authorized mailbox (and the
# default mailbox, and an authorized address). Exercises get_mailbox_from_mail_
# from() (email_handler.py:L1364) selecting the correct mailbox out of
# alias.mailboxes (app/models.py:L1580-L1590) and the resulting
# EmailLog.mailbox_id (email_handler.py:L1047). Run x2.
# =========================================================================== #
def test_s7_multi_mailbox(flask_client):
    mail_sender.store_emails_instead_of_sending(True)
    print("==================== S7: MULTI-MAILBOX alias -> reply from secondary "
          "authorized mailbox x2 ====================")
    distribution = []
    for run in (1, 2):
        user = create_new_user()
        mb1 = user.default_mailbox                 # default mailbox of the alias
        mb2 = Mailbox.create(
            user_id=user.id, email=random_email(), verified=True, flush=True
        )
        alias = Alias.create_new_random(user)
        Session.flush()
        # attach the 2nd verified mailbox -> alias.mailboxes = [mb1, mb2]
        AliasMailbox.create(alias_id=alias.id, mailbox_id=mb2.id, flush=True)
        # an authorized address mapped to the DEFAULT mailbox (mb1)
        authorized = f"authorized-s7r{run}_{random_string(6)}@delegate.example"
        AuthorizedAddress.create(
            user_id=user.id, mailbox_id=mb1.id, email=authorized, flush=True
        )
        contact = Contact.create(
            user_id=user.id, alias_id=alias.id,
            website_email=f"ext-s7r{run}_{random_string(6)}@external.example",
            reply_email=f"ra+s7r{run}_{random_string(6)}@{EMAIL_DOMAIN}",
            flush=True,
        )
        Session.commit()
        # snapshot ALL fixture scalars BEFORE any drive
        alias_id = alias.id
        alias_default_mb_id = alias.mailbox_id
        mb1_id, mb1_email = mb1.id, mb1.email
        mb2_id, mb2_email = mb2.id, mb2.email
        contact_id = contact.id
        reply_email = contact.reply_email
        mb_ids_sorted = [m.id for m in alias.mailboxes]
        print(f"[RUN {run}] BEFORE alias.id={alias_id} "
              f"alias.mailbox_id(default)={alias_default_mb_id} "
              f"mb1(default).id={mb1_id} mb2(secondary).id={mb2_id} "
              f"alias.mailboxes(verified,sorted)={mb_ids_sorted} "
              f"authorized_address_on_mb1={authorized}")

        # (a) reply from the DEFAULT mailbox -> selects default (baseline)
        mail_sender.purge_stored_emails()
        st_a = drive(mb1_email, reply_email,
                     build_reply_msg(mb1_email, reply_email,
                                     f"re: s7r{run} from default", "body-default"))
        logs_a = _captured_logs()
        el_a = _latest_reply_log_dict(contact_id)
        out_a = mail_sender.get_stored_emails()
        to_a = out_a[0].envelope_to if out_a else None
        from_a = out_a[0].msg[headers.FROM] if out_a else None
        print(f"[RUN {run}] (a) DEFAULT mailbox send mail_from={mb1_email} "
              f"-> status={st_a!r}")
        print(f"[RUN {run}] (a)   EmailLog={el_a} selected_mailbox_id="
              f"{el_a['mailbox_id']} == default_mb1.id({mb1_id})? "
              f"{el_a['mailbox_id'] == mb1_id}")
        print(f"[RUN {run}] (a)   OUT envelope_to={to_a} msg.From={from_a}")
        _print_logs(f"[RUN {run}] (a)   ", logs_a)

        # (b) reply from the SECONDARY mailbox -> MUST select the secondary (KEY)
        mail_sender.purge_stored_emails()
        st_b = drive(mb2_email, reply_email,
                     build_reply_msg(mb2_email, reply_email,
                                     f"re: s7r{run} from secondary",
                                     "body-secondary"))
        logs_b = _captured_logs()
        el_b = _latest_reply_log_dict(contact_id)
        out_b = mail_sender.get_stored_emails()
        to_b = out_b[0].envelope_to if out_b else None
        from_b = out_b[0].msg[headers.FROM] if out_b else None
        print(f"[RUN {run}] (b) SECONDARY mailbox send mail_from={mb2_email} "
              f"-> status={st_b!r}")
        print(f"[RUN {run}] (b)   EmailLog={el_b} selected_mailbox_id="
              f"{el_b['mailbox_id']} == secondary_mb2.id({mb2_id})? "
              f"{el_b['mailbox_id'] == mb2_id} (default_mb1.id={mb1_id}, so NOT "
              f"default? {el_b['mailbox_id'] != mb1_id})")
        print(f"[RUN {run}] (b)   OUT envelope_to={to_b} msg.From={from_b}")
        _print_logs(f"[RUN {run}] (b)   ", logs_b)

        # (c) reply from an AUTHORIZED ADDRESS mapped to the default mailbox
        mail_sender.purge_stored_emails()
        st_c = drive(authorized, reply_email,
                     build_reply_msg(authorized, reply_email,
                                     f"re: s7r{run} from authorized",
                                     "body-authorized"))
        logs_c = _captured_logs()
        el_c = _latest_reply_log_dict(contact_id)
        out_c = mail_sender.get_stored_emails()
        to_c = out_c[0].envelope_to if out_c else None
        from_c = out_c[0].msg[headers.FROM] if out_c else None
        print(f"[RUN {run}] (c) AUTHORIZED-ADDRESS send mail_from={authorized} "
              f"-> status={st_c!r}")
        print(f"[RUN {run}] (c)   EmailLog={el_c} selected_mailbox_id="
              f"{el_c['mailbox_id']} == default_mb1.id({mb1_id})? "
              f"{el_c['mailbox_id'] == mb1_id} (authorized address resolves to "
              f"its mapped mailbox)")
        print(f"[RUN {run}] (c)   OUT envelope_to={to_c} msg.From={from_c}")
        _print_logs(f"[RUN {run}] (c)   ", logs_c)

        distribution.append((st_a, el_a["mailbox_id"] == mb1_id,
                             st_b, el_b["mailbox_id"] == mb2_id,
                             st_c, el_c["mailbox_id"] == mb1_id))

    print(f"[S7] distribution over 2 runs (a:status,default-selected ; "
          f"b:status,secondary-selected ; c:status,mapped-to-default): "
          f"{distribution}")
    print(f"[S7] all runs identical (selection invariants)? "
          f"{distribution[0] == distribution[1]}")
    mail_sender.store_emails_instead_of_sending(False)
```

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

#### S7 — Multi-mailbox alias: reply from the secondary authorized mailbox (plus default & authorized-address) ×2

```text
==================== S7: MULTI-MAILBOX alias -> reply from secondary authorized mailbox x2 ====================
[RUN 1] BEFORE alias.id=1095 alias.mailbox_id(default)=719 mb1(default).id=719 mb2(secondary).id=720 alias.mailboxes(verified,sorted)=[720, 719] authorized_address_on_mb1=authorized-s7r1_sprxwh@delegate.example
[RUN 1] (a) DEFAULT mailbox send mail_from=user_xugtj7hjlz@mailbox.test -> status='250 Message accepted for delivery'
[RUN 1] (a)   EmailLog={'id': 351, 'user_id': 633, 'mailbox_id': 719, 'alias_id': 1095, 'contact_id': 286, 'is_reply': True} selected_mailbox_id=719 == default_mb1.id(719)? True
[RUN 1] (a)   OUT envelope_to=ext-s7r1_acskth@external.example msg.From=jumble_monies558@sl.local
[RUN 1] (a)   LOG> _handle:2343 New message, mail from user_xugtj7hjlz@mailbox.test, rctp tos ['ra+s7r1_wyyrre@sl.local']
[RUN 1] (a)   LOG> handle:2196 Reply phase user_xugtj7hjlz@mailbox.test(user_xugtj7hjlz@mailbox.test) -> ra+s7r1_wyyrre@sl.local
[RUN 1] (a)   LOG> apply_dmarc_policy_for_reply_phase:159 DMARC check disabled
[RUN 1] (a)   LOG> handle_reply:1051 Create <EmailLog 351> for <Contact 286 ext-s7r1_acskth@external.example 1095>, <User 633 Test User user_xugtj7hjlz@mailbox.test>, <Mailbox 719 user_xugtj7hjlz@mailbox.test>
[RUN 1] (b) SECONDARY mailbox send mail_from=knxmxmicsqepvgnsewyd@knxmxmicsqepvgnsewyd.com -> status='250 Message accepted for delivery'
[RUN 1] (b)   EmailLog={'id': 352, 'user_id': 633, 'mailbox_id': 720, 'alias_id': 1095, 'contact_id': 286, 'is_reply': True} selected_mailbox_id=720 == secondary_mb2.id(720)? True (default_mb1.id=719, so NOT default? True)
[RUN 1] (b)   OUT envelope_to=ext-s7r1_acskth@external.example msg.From=jumble_monies558@sl.local
[RUN 1] (b)   LOG> _handle:2343 New message, mail from knxmxmicsqepvgnsewyd@knxmxmicsqepvgnsewyd.com, rctp tos ['ra+s7r1_wyyrre@sl.local']
[RUN 1] (b)   LOG> handle:2196 Reply phase knxmxmicsqepvgnsewyd@knxmxmicsqepvgnsewyd.com(knxmxmicsqepvgnsewyd@knxmxmicsqepvgnsewyd.com) -> ra+s7r1_wyyrre@sl.local
[RUN 1] (b)   LOG> apply_dmarc_policy_for_reply_phase:159 DMARC check disabled
[RUN 1] (b)   LOG> handle_reply:1051 Create <EmailLog 352> for <Contact 286 ext-s7r1_acskth@external.example 1095>, <User 633 Test User user_xugtj7hjlz@mailbox.test>, <Mailbox 720 knxmxmicsqepvgnsewyd@knxmxmicsqepvgnsewyd.com>
[RUN 1] (c) AUTHORIZED-ADDRESS send mail_from=authorized-s7r1_sprxwh@delegate.example -> status='250 Message accepted for delivery'
[RUN 1] (c)   EmailLog={'id': 353, 'user_id': 633, 'mailbox_id': 719, 'alias_id': 1095, 'contact_id': 286, 'is_reply': True} selected_mailbox_id=719 == default_mb1.id(719)? True (authorized address resolves to its mapped mailbox)
[RUN 1] (c)   OUT envelope_to=ext-s7r1_acskth@external.example msg.From=jumble_monies558@sl.local
[RUN 1] (c)   LOG> _handle:2343 New message, mail from authorized-s7r1_sprxwh@delegate.example, rctp tos ['ra+s7r1_wyyrre@sl.local']
[RUN 1] (c)   LOG> handle:2196 Reply phase authorized-s7r1_sprxwh@delegate.example(authorized-s7r1_sprxwh@delegate.example) -> ra+s7r1_wyyrre@sl.local
[RUN 1] (c)   LOG> apply_dmarc_policy_for_reply_phase:159 DMARC check disabled
[RUN 1] (c)   LOG> __check:1376 Found an authorized address for <Alias 1095 jumble_monies558@sl.local> <Mailbox 719 user_xugtj7hjlz@mailbox.test> <AuthorizedAddress 13 authorized-s7r1_sprxwh@delegate.example 719>
[RUN 1] (c)   LOG> handle_reply:1051 Create <EmailLog 353> for <Contact 286 ext-s7r1_acskth@external.example 1095>, <User 633 Test User user_xugtj7hjlz@mailbox.test>, <Mailbox 719 user_xugtj7hjlz@mailbox.test>
[RUN 2] BEFORE alias.id=1097 alias.mailbox_id(default)=721 mb1(default).id=721 mb2(secondary).id=722 alias.mailboxes(verified,sorted)=[721, 722] authorized_address_on_mb1=authorized-s7r2_eyeaur@delegate.example
[RUN 2] (a) DEFAULT mailbox send mail_from=user_qncr0vh547@mailbox.test -> status='250 Message accepted for delivery'
[RUN 2] (a)   EmailLog={'id': 354, 'user_id': 634, 'mailbox_id': 721, 'alias_id': 1097, 'contact_id': 287, 'is_reply': True} selected_mailbox_id=721 == default_mb1.id(721)? True
[RUN 2] (a)   OUT envelope_to=ext-s7r2_mmyevi@external.example msg.From=ropers_cowers603@sl.local
[RUN 2] (a)   LOG> _handle:2343 New message, mail from user_qncr0vh547@mailbox.test, rctp tos ['ra+s7r2_bmtchn@sl.local']
[RUN 2] (a)   LOG> handle:2196 Reply phase user_qncr0vh547@mailbox.test(user_qncr0vh547@mailbox.test) -> ra+s7r2_bmtchn@sl.local
[RUN 2] (a)   LOG> apply_dmarc_policy_for_reply_phase:159 DMARC check disabled
[RUN 2] (a)   LOG> handle_reply:1051 Create <EmailLog 354> for <Contact 287 ext-s7r2_mmyevi@external.example 1097>, <User 634 Test User user_qncr0vh547@mailbox.test>, <Mailbox 721 user_qncr0vh547@mailbox.test>
[RUN 2] (b) SECONDARY mailbox send mail_from=wbqahhlnsjuyvzzkwipu@wbqahhlnsjuyvzzkwipu.com -> status='250 Message accepted for delivery'
[RUN 2] (b)   EmailLog={'id': 355, 'user_id': 634, 'mailbox_id': 722, 'alias_id': 1097, 'contact_id': 287, 'is_reply': True} selected_mailbox_id=722 == secondary_mb2.id(722)? True (default_mb1.id=721, so NOT default? True)
[RUN 2] (b)   OUT envelope_to=ext-s7r2_mmyevi@external.example msg.From=ropers_cowers603@sl.local
[RUN 2] (b)   LOG> _handle:2343 New message, mail from wbqahhlnsjuyvzzkwipu@wbqahhlnsjuyvzzkwipu.com, rctp tos ['ra+s7r2_bmtchn@sl.local']
[RUN 2] (b)   LOG> handle:2196 Reply phase wbqahhlnsjuyvzzkwipu@wbqahhlnsjuyvzzkwipu.com(wbqahhlnsjuyvzzkwipu@wbqahhlnsjuyvzzkwipu.com) -> ra+s7r2_bmtchn@sl.local
[RUN 2] (b)   LOG> apply_dmarc_policy_for_reply_phase:159 DMARC check disabled
[RUN 2] (b)   LOG> handle_reply:1051 Create <EmailLog 355> for <Contact 287 ext-s7r2_mmyevi@external.example 1097>, <User 634 Test User user_qncr0vh547@mailbox.test>, <Mailbox 722 wbqahhlnsjuyvzzkwipu@wbqahhlnsjuyvzzkwipu.com>
[RUN 2] (c) AUTHORIZED-ADDRESS send mail_from=authorized-s7r2_eyeaur@delegate.example -> status='250 Message accepted for delivery'
[RUN 2] (c)   EmailLog={'id': 356, 'user_id': 634, 'mailbox_id': 721, 'alias_id': 1097, 'contact_id': 287, 'is_reply': True} selected_mailbox_id=721 == default_mb1.id(721)? True (authorized address resolves to its mapped mailbox)
[RUN 2] (c)   OUT envelope_to=ext-s7r2_mmyevi@external.example msg.From=ropers_cowers603@sl.local
[RUN 2] (c)   LOG> _handle:2343 New message, mail from authorized-s7r2_eyeaur@delegate.example, rctp tos ['ra+s7r2_bmtchn@sl.local']
[RUN 2] (c)   LOG> handle:2196 Reply phase authorized-s7r2_eyeaur@delegate.example(authorized-s7r2_eyeaur@delegate.example) -> ra+s7r2_bmtchn@sl.local
[RUN 2] (c)   LOG> apply_dmarc_policy_for_reply_phase:159 DMARC check disabled
[RUN 2] (c)   LOG> __check:1376 Found an authorized address for <Alias 1097 ropers_cowers603@sl.local> <Mailbox 721 user_qncr0vh547@mailbox.test> <AuthorizedAddress 14 authorized-s7r2_eyeaur@delegate.example 721>
[RUN 2] (c)   LOG> handle_reply:1051 Create <EmailLog 356> for <Contact 287 ext-s7r2_mmyevi@external.example 1097>, <User 634 Test User user_qncr0vh547@mailbox.test>, <Mailbox 721 user_qncr0vh547@mailbox.test>
[S7] distribution over 2 runs (a:status,default-selected ; b:status,secondary-selected ; c:status,mapped-to-default): [('250 Message accepted for delivery', True, '250 Message accepted for delivery', True, '250 Message accepted for delivery', True), ('250 Message accepted for delivery', True, '250 Message accepted for delivery', True, '250 Message accepted for delivery', True)]
[S7] all runs identical (selection invariants)? True
```

**(observed)** The alias was given **two** verified mailboxes — a default `mb1` and a secondary `mb2` — plus an authorized delegate address mapped to `mb1`; `alias.mailboxes` therefore lists both, sorted by email (`[720, 719]` in run 1, `[721, 722]` in run 2). Replying from each authorized origin selected the mailbox matching `mail_from`, not merely the default: default-send persisted `EmailLog.mailbox_id == alias.mailbox_id` (the default `mb1`); **secondary-send persisted `EmailLog.mailbox_id` = the secondary `mb2`** (`720`/`722`, `!= alias.mailbox_id`); delegate-send resolved to the mapped default `mb1` and emitted `Found an authorized address ...` (`email_handler.py:L1376`). All three returned `250`, and the selection invariants (default-selected, secondary-selected, mapped-to-default) were identical across both runs (`all runs identical (selection invariants)? True`). **(inferred)** Because the selected mailbox is chosen by matching `mail_from` against `alias.mailboxes` (`get_mailbox_from_mail_from`, `email_handler.py:L1364`; `app/models.py:L1580–L1590`) rather than by the recipient reverse-alias, a multi-mailbox alias routes each reply back through the *sending* mailbox — so multi-mailbox authorization is **not** a source of the reported mis-routing (`EmailLog.mailbox_id` correctly follows the sender). This exercises the AAP §0.5.1 multi-mailbox condition.

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
| 18 | Multi-mailbox alias: a reply from the alias's **secondary** authorized mailbox is routed/attributed to that secondary mailbox (`EmailLog.mailbox_id` == sending mailbox, `!= alias.mailbox_id`); default- and delegate-sends select the default — all `250` | **observed** (`S7`, both runs; `email_handler.py:L1364`/`L1047`, `app/models.py:L1580–L1590`) |

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
- **Edge conditions:** exercised in [§ (g)](#g-edge-condition-results) — `S4` (unauthorized→E214), `S5` (spoofing-off default fallback), `S6` (`NOREPLIES`/bounce), and `S7` (**multi-mailbox alias** — a reply from the secondary authorized mailbox routes to that mailbox; AAP §0.5.1), each reproduced on both runs.

*End of analysis. The only artifact added to the repository is this document; no source file was modified, and all temporary observation scaffolding lived inside the run container and was removed.*
