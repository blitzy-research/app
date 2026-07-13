# SimpleLogin Alias Reply-Handling Flow — Runtime Root-Cause Analysis

**Source branch:** `app_2cd6ee777f8c`
**HEAD commit:** `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`
**Nature of this document:** a **read-only, runtime-first** investigation. The relevant code paths were BUILT and RUN first, the real output was captured, and this analysis is written from what was *observed* — not from reading the code alone. No source file was modified; the only artifact produced is this document. Temporary observation scripts lived in a scratch location outside the repository and were removed, leaving the working tree git-clean.

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

All citations use the form `file:line` (e.g., `email_handler.py:L986`) and were verified against the source at the HEAD commit above. All runtime numbers, log lines, and status strings are transcribed EXACTLY as emitted — nothing is rounded, paraphrased, or "cleaned up."

---

## (a) Verdict — real bug vs. misunderstanding (bottom line up front)

**Direct answer:** *Both interpretations are partly correct, and which one applies depends entirely on the data.* In the default, correctly-owned configuration the reply flow routes **correctly and deterministically**, so a "wrong user" report for the ordinary case is most likely a **misunderstanding** of the reverse-alias model. However, the pipeline contains **two real, reproducible latent defects** that genuinely mis-route replies under specific-but-plausible data conditions — so the behavior *can* be a real bug, not merely a misunderstanding.

- **(observed)** In the DEFAULT, correctly-owned configuration (`contact.user_id == alias.user_id`, and a `reply_email` that maps to exactly one `Contact`), the reply flow routes **correctly and deterministically**. Scenario `S1` was run twice on identical-shaped input: both runs reported `agree=True`, the persisted `EmailLog.user_id` matched **both** user references, `handle_DATA` returned `'250 Message accepted for delivery'`, and the message was relayed to the correct external contact with the user's real mailbox hidden. There was **no run-to-run inconsistency** on identical input.
- **(inferred)** For that common case, a report of "the reply went to the wrong user" is most likely a **misunderstanding of the reverse-alias model**: a reply to a reverse alias legitimately egresses *outward* to the external contact's `website_email`, with the `From` header rewritten to the alias. SimpleLogin **relays the reply outward**; it does **not** deposit the reply into another SimpleLogin user's inbox. So "wrong user" only has a concrete technical meaning as either (i) the *wrong `Contact`* being resolved (→ wrong external recipient and wrong owning `user_id`), or (ii) a *divergence* between the user that authorizes the send (`alias.user`) and the user that gets recorded as owner (`contact.user_id`).
- **(observed + inferred)** The pipeline nevertheless contains **two reproducible latent defects** that DO cause mis-routing when the data allows it:
  - **Q5b — non-unique `reply_email` resolved by `.first()`.** Two `Contact` rows can share the same `reply_email` (the column is indexed but **not** unique — `app/models.py:L1899`; the only uniqueness is `uq_contact(alias_id, website_email)` — `app/models.py:L1875`). The resolver `Contact.get_by(reply_email=…)` returns `.first()` with no correctness `ORDER BY` (`app/models.py:L84`). Reproduced at runtime in `S2`/`S3`: `.first()` deterministically returns the lowest-PK / first-inserted row (`ids=[5, 5, 5, 5, 5]` and `ids=[7, 7, 7, 7, 7]`), which single-handedly dictates the destination `website_email` AND the owning `user_id`.
  - **Q5a — the dual user reference.** Authorization/DMARC/mailbox selection use `user = alias.user` (`email_handler.py:L1004`), while the persisted row is written with `user_id=contact.user_id` (`email_handler.py:L1046`). Reproduced in `S2`: with `alias.user_id=4` but `contact.user_id=5`, the reply was accepted (`250`) yet `EmailLog.user_id=5` while authorization ran against user `4` — `MISMATCH=True`.
- **(inferred) Single most likely origin (see [§ (f)](#f-q5--most-likely-point-where-an-incorrect-routing-decision-originates)):** the **contact-resolution step `Contact.get_by(reply_email=…).first()` at `email_handler.py:L986`**, combined with the non-unique `reply_email` schema (`app/models.py:L1899`) and the `.first()` semantics (`app/models.py:L84`). This is the single pivot from which the alias, the user, and the destination all hang — and it requires no abnormal ownership state to misfire, only two contacts that happen to share a `reply_email`. The dual user reference (Q5a) is the closely-related structural fragility that lets the *authorizing* user and the *recorded* user diverge once resolution goes wrong (or once the two `user_id`s differ).

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
LOG> _handle:2343 New message, mail from user_glv8jsf45q@mailbox.test, rctp tos ['ra+s1r1_233293235@sl.local']
LOG> handle:2196 Reply phase user_glv8jsf45q@mailbox.test(user_glv8jsf45q@mailbox.test) -> ra+s1r1_233293235@sl.local
```

**(observed)** The `Reply phase …` line confirms that `handle()` classified `ra+s1r1_233293235@sl.local` as a reverse alias and entered the reply branch, which then calls `handle_reply(envelope, copy_msg, rcpt_to)` (`email_handler.py:L2199`).

---

## (c) Q2 — How is the alias resolved to a user? (resolution chain)

**Direct answer (observed / code):** the alias is resolved **indirectly through the `Contact` reverse-alias record**. The recipient's `reply_email` is looked up to a `Contact`, and the owning user/alias are read off that contact: `reply_email` → `Contact` → `Contact.alias` → `Alias.user`. There is no direct `reply_email → User` lookup; the `Contact` row is the pivot.

Step-by-step, as exercised at runtime:

1. **Classification** — `handle()` calls `is_reverse_alias(rcpt_to)` (`app/email_utils.py:L1156`). The function returns `True` if `Contact.get_by(reply_email=address)` exists (`app/email_utils.py:L1158`), otherwise it returns whether the address ends with `@EMAIL_DOMAIN` **and** starts with `reply+` or `ra+` (`app/email_utils.py:L1161–L1163`). **(observed)** In `S1`, `is_reverse_alias(reply_email)=True`.
2. **Dispatch** — the reverse-alias branch calls `handle_reply(envelope, copy_msg, rcpt_to)` (`email_handler.py:L2199`); the reply handler is `def handle_reply(envelope, msg, rcpt_to) -> (bool, str)` (`email_handler.py:L966`).
3. **Normalize** — `reply_email = normalize_reply_email(reply_email)` (`email_handler.py:L984`), after a reply-domain validation gate (`email_handler.py:L977–L981`; a wrong reply domain returns `status.E501`).
4. **Contact lookup (the pivot)** — `contact = Contact.get_by(reply_email=reply_email)` (`email_handler.py:L986`). `get_by` is `Session.query(cls).filter_by(**kw).first()` (`app/models.py:L83–L84`) — i.e., it returns exactly one row via `.first()`, with **no `ORDER BY`**. If no contact matches, the handler logs `No contact …` and returns `status.E502`.
5. **Active-user gate** — `if not contact.user.is_active():` (`email_handler.py:L990`); a soft-deleted user yields `E502`.
6. **Alias from contact** — `alias = contact.alias` (`email_handler.py:L994`), followed by an alias-domain sanity check that yields `E503` on failure (`email_handler.py:L997–L1002`).
7. **User from alias** — `user = alias.user` (`email_handler.py:L1004`); a user that `can_send_or_receive()` is `False` yields `E504` (`email_handler.py:L1007`).

**(observed) The resolved chain in `S1` (run 1):** `contact.id=2` → `alias.id=4` (`gamuts_papaya338@sl.local`) → `user.id=2`. The relationships used are `Contact.alias` (`app/models.py:L1907`) and `Contact.user` / `Alias.user`; the contact's own `user_id` column is `app/models.py:L1877–L1878`, and its `reply_email` column is `app/models.py:L1899`.

---

## (d) Q3 — The concrete integer `user_id` the system decides

**Direct answer (observed):** on the happy path (`S1` run 1) the system decided **`user_id = 2`** — and, critically, it computes **two independent user references** on the same run: the *authorizing* user `user = alias.user` (`email_handler.py:L1004`, `alias.user_id=2` in `S1`) and the *persisted* owner `EmailLog.user_id = contact.user_id` (`email_handler.py:L1046`, value `2` in `S1`). In `S1` they `agree=True`; in `S2` they were deliberately made to diverge and did NOT agree.

- **(observed) Authorization user** — `user = alias.user` (`email_handler.py:L1004`) drives the permission / DMARC / mailbox-authorization checks. In `S1` run 1, `alias.user_id=2`.
- **(observed) Persisted user** — the reply is recorded by `EmailLog.create(...)` (`email_handler.py:L1042`) with these fields: `contact_id=contact.id` (L1043), `alias_id=contact.alias_id` (L1044), `is_reply=True` (`email_handler.py:L1045`), `user_id=contact.user_id` (`email_handler.py:L1046`), `mailbox_id=mailbox.id` (`email_handler.py:L1047`), `message_id=…` (L1048), `commit=True` (L1049). The `EmailLog` model columns are `class EmailLog` (`app/models.py:L2060`) → `user_id` (L2064), `contact_id` (L2067), `alias_id` (L2070), `is_reply` (L2075), `mailbox_id` (L2103).
- **(observed) The persisted row in `S1` run 1:**

```text
EmailLog.id=2 user_id=2 mailbox_id=2 alias_id=4 contact_id=2 is_reply=True
[Q3] EmailLog.user_id=2 (contact.user_id=2, alias.user_id=2) agree=True
```

- **(observed) Confirming log line** (`LOG.d("Create %s for %s, %s, %s", email_log, contact, user, mailbox)` at `email_handler.py:L1051`):

```text
LOG> handle_reply:1051 Create <EmailLog 2> for <Contact 2 ext-13474@external.example 4>, <User 2 Test User user_glv8jsf45q@mailbox.test>, <Mailbox 2 user_glv8jsf45q@mailbox.test>
```

- **(observed) Captured outbound** (via `mail_sender.get_stored_emails()` — `app/mail_sender.py:L108`): `envelope_to=ext-13474@external.example`, `msg.From=gamuts_papaya338@sl.local`. That is, the reply is relayed to `contact.website_email` with the `From` header rewritten to the alias.
- **(inferred) The crux:** two independent user references coexist. They are computed independently (`alias.user` vs `contact.user_id`), are normally equal, but **can diverge** — proven at runtime in `S2`, where `EmailLog.user_id=5` while authorization used `alias.user_id=4` (`MISMATCH=True`). See [§ (f)](#f-q5--most-likely-point-where-an-incorrect-routing-decision-originates) hypothesis (a).

---

## (e) Q4 — Actual observed end-to-end data flow

**Direct answer:** the observed path, from SMTP `DATA` to the outbound relayed message, is:

`handle_DATA` → `_handle` → `handle` → `is_reverse_alias` → `handle_reply` → `Contact.get_by(reply_email).first()` → `alias = contact.alias` / `user = alias.user` → `apply_dmarc_policy_for_reply_phase` → `get_mailbox_from_mail_from` → `EmailLog.create(user_id=contact.user_id, mailbox_id=mailbox.id, is_reply=True)` → rewrite `From` to alias, relay to `contact.website_email`.

**(observed / code) Narrative with citations and the `S1` evidence tying each node to a captured value:**

1. **SMTP `DATA`** → `MailHandler.handle_DATA()` (`email_handler.py:L2289`). *Observed:* the run began by driving this callback directly.
2. **Flask app context** → `_handle()` (`email_handler.py:L2335`). *Observed:* `LOG> _handle:2343 New message, mail from user_glv8jsf45q@mailbox.test, rctp tos ['ra+s1r1_233293235@sl.local']`.
3. **Routing hub** → `handle()` (`email_handler.py:L1945`), per-recipient loop (`email_handler.py:L2180`).
4. **Classify recipient** → `is_reverse_alias(rcpt_to)` (`app/email_utils.py:L1156`). *Observed:* `is_reverse_alias(reply_email)=True`; reverse-alias branch at `email_handler.py:L2195`, `LOG> handle:2196 Reply phase … -> ra+s1r1_233293235@sl.local`.
5. **Reply handler** → `handle_reply()` (`email_handler.py:L966`, called at `email_handler.py:L2199`).
6. **Resolve contact (pivot)** → `contact = Contact.get_by(reply_email=reply_email)` (`email_handler.py:L986`) = `filter_by(...).first()` (`app/models.py:L84`). *Observed:* `contact.id=2`.
7. **Resolve alias & user** → `alias = contact.alias` (`email_handler.py:L994`), `user = alias.user` (`email_handler.py:L1004`). *Observed:* `alias.id=4`, `alias.user_id=2`.
8. **DMARC policy** → `dmarc_delivery_status = apply_dmarc_policy_for_reply_phase(alias, contact, envelope, msg)` (`email_handler.py:L1012`; function at `app/handler/dmarc.py:L154`). *Observed:* the reply passed cleanly (final status `250`).
9. **Authorize sending mailbox** → `mailbox = get_mailbox_from_mail_from(mail_from, alias)` (`email_handler.py:L1019`; function at `email_handler.py:L1364`, matching `mail_from` against `alias.mailboxes` and their `authorized_addresses`, raw then canonicalized). *Observed:* `mailbox.id=2`.
10. **Persist the decision** → `EmailLog.create(user_id=contact.user_id, mailbox_id=mailbox.id, is_reply=True, …)` (`email_handler.py:L1042–L1050`). *Observed:* `EmailLog.id=2 user_id=2 mailbox_id=2 alias_id=4 contact_id=2 is_reply=True`.
11. **Rewrite & relay** → `From` rewritten to the alias, message relayed to `contact.website_email`. *Observed:* `OUT envelope_to=ext-13474@external.example | msg.From=gamuts_papaya338@sl.local`.

**(observed) Data-flow diagram of the exercised path** (the `Q5` candidate origins are annotated):

```mermaid
flowchart TD
    A["SMTP DATA<br/>MailHandler.handle_DATA()<br/>email_handler.py:L2289"] --> B["_handle()<br/>Flask app context<br/>email_handler.py:L2335"]
    B --> C["handle() routing hub<br/>email_handler.py:L1945<br/>per-recipient loop L2180"]
    C --> D{"is_reverse_alias(rcpt_to)?<br/>app/email_utils.py:L1156"}
    D -->|"No"| E["handle_forward()<br/>email_handler.py:L536"]
    D -->|"Yes (REPLY) L2195"| F["handle_reply()<br/>email_handler.py:L966 (called L2199)"]
    F --> G["contact = Contact.get_by(reply_email).first()<br/>email_handler.py:L986 / models.py:L84  &lt;-- Q5b pivot"]
    G --> H["alias = contact.alias  L994<br/>user = alias.user  L1004  &lt;-- Q5a authz user"]
    H --> DM["apply_dmarc_policy_for_reply_phase()<br/>email_handler.py:L1012 / dmarc.py:L154"]
    DM --> I["mailbox = get_mailbox_from_mail_from(mail_from, alias)<br/>email_handler.py:L1019 / L1364"]
    I --> J{"mailbox found?"}
    J -->|"No + spoofing check ON"| K["handle_unknown_mailbox()<br/>return E214 — L1032/L1034"]
    J -->|"No + spoofing check OFF"| L["mailbox = alias.mailbox (default)<br/>email_handler.py:L1029  &lt;-- Q5c fallback"]
    J -->|"Yes"| M["EmailLog.create(user_id=contact.user_id,<br/>mailbox_id=mailbox.id, is_reply=True)<br/>email_handler.py:L1042-L1050  &lt;-- Q5a recorded user"]
    L --> M
    M --> N["Rewrite From to alias, relay to<br/>contact.website_email"]
```

---

## (f) Q5 — Most likely point where an incorrect routing decision originates

Three code-grounded hypotheses were tested at runtime. Each is presented with its reproduced result, then the single most likely origin is named.

### Hypothesis (a) — Dual user reference (`alias.user` vs `contact.user_id`)

- **(observed / code)** Authorization/DMARC/mailbox selection use `user = alias.user` (`email_handler.py:L1004`), but the persisted row uses `user_id=contact.user_id` (`email_handler.py:L1046`). These are two independently-computed references to "the user."
- **(observed) Reproduced in `S2`:** with `alias.user_id=4` and `contact.user_id=5`, the reply was accepted (`'250 Message accepted for delivery'`), yet `EmailLog.user_id=5` while permission checks ran against `alias.user` (id `4`) — `MISMATCH=True`.
- **(observed) Context:** the standard ownership-transfer path `transfer_alias()` (`app/alias_utils.py:L458–L540`) updates `Contact.user_id` (`app/alias_utils.py:L464–L466`) AND `alias.user_id` (`app/alias_utils.py:L506`) **together**.
- **(inferred)** Because the normal transfer path keeps the two references in sync, a divergence between `alias.user_id` and `contact.user_id` is an **abnormal/inconsistent data state**, not a normal one. When it does occur, the *authorizing* user and the *recorded owner* refer to different accounts. **(inferred, sibling variant)** The exact same dual reference is used in the forward path — `user = alias.user` (`email_handler.py:L557`) vs `user_id=contact.user_id` (`email_handler.py:L600` and `email_handler.py:L734`) — so this is a pipeline-wide pattern, not a one-off in the reply handler.

### Hypothesis (b) — Non-unique `reply_email` resolved by `.first()`

- **(observed / code)** The `reply_email` column is indexed but **NOT unique** — `reply_email = sa.Column(sa.String(512), nullable=False, index=True)` (`app/models.py:L1899`). The **only** uniqueness constraint on `Contact` is `uq_contact(alias_id, website_email)` (`app/models.py:L1875`). The resolver `Contact.get_by(reply_email=…)` (`email_handler.py:L986`) is `Session.query(cls).filter_by(**kw).first()` (`app/models.py:L84`) — a `.first()` with no correctness `ORDER BY`.
- **(observed) Reproduced in `S3`:** two `Contact` rows were created sharing one `reply_email` (the DB accepted both — confirming non-uniqueness). Over 5 identical calls, `Contact.get_by(reply_email).first()` returned a **stable** result: `ids=[5, 5, 5, 5, 5]` for insertion order A-then-B, and `ids=[7, 7, 7, 7, 7]` for order B-then-A. In both cases it returned the row with the **lowest primary key = the first-inserted contact**.
- **(inferred)** Because `.first()` has no `ORDER BY` tied to correctness, the "winner" is decided purely by **insertion order**, not by which contact the reply was actually meant for. Whichever contact was created first "captures" that `reply_email`, and **its** `user_id` and **its** `website_email` are used. A reply can therefore be relayed to the *wrong external recipient* and attributed to the *wrong user*, while the alias is still "recognized" — exactly matching the reported symptom.

### Hypothesis (c) — `disable_email_spoofing_check` default-mailbox fallback

- **(observed / code)** When `get_mailbox_from_mail_from` returns no mailbox (`if not mailbox:` — `email_handler.py:L1020`), the code branches on `if alias.disable_email_spoofing_check:` (`email_handler.py:L1021`). If the check is disabled, it logs "ignore unknown sender to reverse-alias" (`email_handler.py:L1023`) and falls back to `mailbox = alias.mailbox` (`email_handler.py:L1029`). Otherwise it calls `handle_unknown_mailbox(...)` (`email_handler.py:L1032`) and returns `status.E214` (`email_handler.py:L1034`).
- **(observed) Reproduced in `S5`:** with `disable_email_spoofing_check=True`, an unknown sender (`stranger@nowhere.example`) was NOT rejected; the reply `EmailLog` was created with `user_id=11`, `mailbox_id=11 == alias.mailbox_id`, `used_default=True`.
- **(inferred)** This fallback widens *who* can trigger a relay through the reverse alias, but it uses the **correct owning mailbox** in the normal-ownership case, so it changes *who can send*, not *which user the reply is attributed to*. It is a routing-relevant behavior toggle rather than a mis-attribution defect on its own.

### Conclusion — the single most likely origin

- **(inferred, evidence-led)** The **most likely origin** of an *incorrect routing decision that still shows "alias recognized"* is the **contact-resolution step `Contact.get_by(reply_email=…).first()` at `email_handler.py:L986`**, combined with the non-unique `reply_email` schema (`app/models.py:L1899`) and the `.first()` semantics (`app/models.py:L84`) — **Hypothesis (b)**. Cause → effect: `reply_email` is not unique, so multiple `Contact` rows can carry it; `.first()` then selects one row by insertion order (no correctness ordering); and because *everything downstream hangs off that single contact* — the alias (`contact.alias`), the recorded user (`contact.user_id`), and the destination (`contact.website_email`) — choosing the wrong contact simultaneously mis-routes the recipient and mis-attributes the user, all while `is_reverse_alias` still reports the alias as "recognized." This needs **no abnormal ownership state** and is reachable whenever two contacts share a `reply_email`.
- **(inferred)** **Hypothesis (a)** (the dual user reference, `alias.user` at `email_handler.py:L1004` vs `contact.user_id` at `email_handler.py:L1046`) is the closely-related **structural fragility** that makes the *authorizing* user and the *recorded* user diverge once resolution goes wrong, or once the two `user_id`s are inconsistent (as reproduced in `S2`). It amplifies (b) but, on its own, requires the abnormal divergent-ownership state that `transfer_alias()` normally prevents.
- **(inferred)** **Hypothesis (c)** is a secondary, configuration-gated widening of sender acceptance; it does not by itself send the reply to the wrong user under normal ownership.

---

## (g) Edge-condition results

Every distinct condition the question implies was exercised — the happy path plus the secondary/edge/error paths — through the real entry point. Before/after states are reported where state changes.

| Scenario | Condition | Observed status | Reply `EmailLog` (before → after) | Key citation |
|----------|-----------|-----------------|-----------------------------------|--------------|
| `S1` | Happy path, correctly-owned (×2) | `'250 Message accepted for delivery'` | none → created (`user_id` matches both refs, `agree=True`) | `email_handler.py:L1042–L1050` |
| `S2` | Constructed divergence `alias.user_id != contact.user_id` | `'250 Message accepted for delivery'` | none → created (`EmailLog.user_id=5`, authz used `alias.user_id=4`, `MISMATCH=True`) | `email_handler.py:L1004` vs `L1046` |
| `S3` | Non-unique `reply_email` (two contacts share it) | n/a (resolver probe) | n/a — `.first()` returns lowest-PK contact, stable over 5 calls | `email_handler.py:L986` / `app/models.py:L84`, `L1899` |
| `S4` | Unauthorized sender, spoofing check **ON** | `'250 SL E214 Unauthorized for using reverse alias'` | none → **NONE (blocked)**; `total_stored=1` is the alert to the legitimate user | `email_handler.py:L1032`/`L1034`, `handle_unknown_mailbox` `email_handler.py:L1390` |
| `S5` | Unknown sender, `disable_email_spoofing_check=True` | `'250 Message accepted for delivery'` | none → created (`user_id=11`, `mailbox_id=11==alias.mailbox_id`, `used_default=True`) | `email_handler.py:L1023`/`L1029` |
| `S6` | `NOREPLIES` address; bounce `mail_from == "<>"` | `NOREPLIES → '250 Message accepted for delivery'`; bounce `→ '250 SL E206 Out of office'` | no reply routing performed | `email_handler.py:L2181`; `email_handler.py:L2166`/`email_handler.py:L2173` |

- **(observed) `S4` — unauthorized sender → E214 (spoofing ON):** a sender NOT authorized for the alias's mailbox is rejected with **E214** ("Unauthorized for using reverse alias") at `email_handler.py:L1034` (via `handle_unknown_mailbox` `email_handler.py:L1032`/`email_handler.py:L1390`). NO reply `EmailLog` is created (before → after: reply `EmailLog` = NONE). The `total_stored=1` is the alert email sent to the legitimate user, not a relayed reply.
- **(observed) `S5` — default-mailbox fallback (spoofing OFF):** with `disable_email_spoofing_check=True`, an unknown sender is NOT rejected — the code logs "ignore unknown sender to reverse-alias" at `email_handler.py:L1023`, falls back to `mailbox = alias.mailbox` at `email_handler.py:L1029`, and creates the reply `EmailLog` (`user_id=11`, `mailbox_id=11 == alias.mailbox_id`, `used_default=True`). Directly contrasts `S4`'s E214.
- **(observed) `S6` — guard paths around the reply logic:** a message to a `NOREPLIES` address short-circuits at `email_handler.py:L2181` and returns `'250 Message accepted for delivery'` without reply handling. A bounce/auto-reply with `mail_from == "<>"` addressed to a reverse alias is handled as out-of-office → **E206** ("Out of office") at `email_handler.py:L2166`/`email_handler.py:L2173`, before `handle_reply` routing.

---

## (h) Commands run & raw runtime output

### Environment (observed — canonical / default configuration)

The investigation was run in the canonical configuration a normal user / the CI recipe would use:

- **Runtime:** Python **3.10.20** (constraint `python = "^3.10"` in `pyproject.toml:L61`).
- **Database:** PostgreSQL **13.23** (`PostgreSQL 13.23 (Debian 13.23-1.pgdg13+1) on x86_64-pc-linux-gnu`).
- **Cache/broker:** Redis **6.2.22**.
- **Key libraries** (from `poetry.lock`, resolved & imported): `aiosmtpd 1.4.2`, `SQLAlchemy 1.3.24`, `Flask 1.1.2`.
- **Schema build (canonical):** `CONFIG=tests/test.env poetry run alembic upgrade head` → **77 tables** created (exit 0). This mirrors `.github/workflows/main.yml` (PostgreSQL 13, Redis 6, `alembic upgrade head`).
- **Config used (`tests/test.env`):** `NOT_SEND_EMAIL=true` (`tests/test.env:L7`), `EMAIL_DOMAIN=sl.local` (`tests/test.env:L8`), `DB_URI=postgresql://test:test@localhost:15432/test` (`tests/test.env:L17`), `DMARC_CHECK_ENABLED=true` (`tests/test.env:L66`).
- **Simulation vehicle (canonical entry point — no mocks/hooks):** an in-process pytest module that constructs an `aiosmtpd.smtp.Envelope` and drives `MailHandler().handle_DATA(None, None, envelope)` → `_handle()` → `handle()`, exactly mirroring the pattern in `tests/test_email_handler.py::test_dmarc_reply_quarantine` (`tests/test_email_handler.py:L165`; `Envelope` import at `tests/test_email_handler.py:L6`). Outbound relay was captured non-intrusively via `mail_sender.store_emails_instead_of_sending(True)` + `get_stored_emails()` (`app/mail_sender.py:L102`, `app/mail_sender.py:L108`), so no real email left the environment. `MailSender.send` appends to the capture list before the `NOT_SEND_EMAIL` check (`app/mail_sender.py:L126`), so capture works even with `NOT_SEND_EMAIL=true`.
- **Test result:** `6 passed, 18 warnings in 3.61s`.
- **(observed) Methodology / integrity note:** the user-provided canonical Docker image `andrewparkscaleai/coding-agent:simple-login__app__2cd6ee777f8c…` was NOT pullable (it requires `docker login`); an equivalent environment was therefore built at the exact pinned versions from `poetry.lock`. All values reported here were obtained through the **real** entry point `handle_DATA` / `_handle` / `handle` — none from a bypass, mock, or synthetic stand-in — so all reported values are **canonical observations**. The temporary pytest module (`<temp_investigation_module>` below) lived only in a scratch copy outside the repository and was deleted afterward, leaving the working tree git-clean.

### Reproduction commands

```bash
# 1) Services (canonical: Postgres 13 + Redis 6)
#    postgres:13 published on host 15432; redis:6 on 6379
# 2) Install deps at pinned versions from poetry.lock (Python 3.10)
# 3) Build schema via the canonical CI recipe:
CONFIG=tests/test.env alembic upgrade head        # -> 77 tables, exit 0
# 4) Drive the REAL SMTP callback in-process and capture output:
CONFIG=tests/test.env python -m pytest tests/<temp_investigation_module>.py -s
#    -> 6 passed, 18 warnings in 3.61s
```

`<temp_investigation_module>` was a temporary, scratch-only pytest file that drove `MailHandler.handle_DATA`; it was deleted after the run, leaving the repository git-clean.

### Raw scenario output (verbatim)

#### S1 — Happy-path reply (run twice for stability)

```text
==================== S1: HAPPY-PATH REPLY (x2 for stability) ====================
[RUN 1] BEFORE:
  user.id=2 user.email=user_glv8jsf45q@mailbox.test
  mailbox.id=2 mailbox.email=user_glv8jsf45q@mailbox.test
  alias.id=4 alias.email=gamuts_papaya338@sl.local alias.user_id=2 alias.mailbox_id=2
  contact.id=2 contact.user_id=2 contact.alias_id=4 website_email=ext-13474@external.example reply_email=ra+s1r1_233293235@sl.local
  is_reverse_alias(reply_email)=True
  [dual-ref] alias.user_id=2 contact.user_id=2 EQUAL=True
2026-... - SL - DEBUG - "/code/email_handler.py:2342" - _handle() - ... - ====>=====>====>====>====>====>====>====>
2026-... - SL - DEBUG - "/code/email_handler.py:1051" - handle_reply() - ... - Create <EmailLog 2> for <Contact 2 ext-13474@external.example 4>, <User 2 Test User user_glv8jsf45q@mailbox.test>, <Mailbox 2 user_glv8jsf45q@mailbox.test>
[RUN 1] AFTER:
  handle_DATA return status = '250 Message accepted for delivery'
  EmailLog.id=2 user_id=2 mailbox_id=2 alias_id=4 contact_id=2 is_reply=True
  [Q3] EmailLog.user_id=2 (contact.user_id=2, alias.user_id=2) agree=True
  stored outbound count = 1
  OUT envelope_from=sl.lmysyibsfqqdemzygi3dsok5.pdkhwd4jeungw@sl.local envelope_to=ext-13474@external.example | msg.From=gamuts_papaya338@sl.local msg.To=Ext Contact <ext-13474@external.example>
  LOG> _handle:2343 New message, mail from user_glv8jsf45q@mailbox.test, rctp tos ['ra+s1r1_233293235@sl.local']
  LOG> handle:2196 Reply phase user_glv8jsf45q@mailbox.test(user_glv8jsf45q@mailbox.test) -> ra+s1r1_233293235@sl.local
  LOG> handle_reply:1051 Create <EmailLog 2> for <Contact 2 ext-13474@external.example 4>, <User 2 Test User user_glv8jsf45q@mailbox.test>, <Mailbox 2 user_glv8jsf45q@mailbox.test>
[RUN 2] BEFORE:
  user.id=3 user.email=user_kbuh7a0kwa@mailbox.test
  mailbox.id=3 mailbox.email=user_kbuh7a0kwa@mailbox.test
  alias.id=6 alias.email=bisect_anyone339@sl.local alias.user_id=3 alias.mailbox_id=3
  contact.id=3 contact.user_id=3 contact.alias_id=6 website_email=ext-133902@external.example reply_email=ra+s1r2_505975812@sl.local
  is_reverse_alias(reply_email)=True
  [dual-ref] alias.user_id=3 contact.user_id=3 EQUAL=True
[RUN 2] AFTER:
  handle_DATA return status = '250 Message accepted for delivery'
  EmailLog.id=3 user_id=3 mailbox_id=3 alias_id=6 contact_id=3 is_reply=True
  [Q3] EmailLog.user_id=3 (contact.user_id=3, alias.user_id=3) agree=True
  stored outbound count = 1
  OUT envelope_from=sl.lmysyibtfqqdemzygi3dsok5.6fjuqjl7joosq@sl.local envelope_to=ext-133902@external.example | msg.From=bisect_anyone339@sl.local msg.To=Ext Contact <ext-133902@external.example>
  LOG> _handle:2343 New message, mail from user_kbuh7a0kwa@mailbox.test, rctp tos ['ra+s1r2_505975812@sl.local']
  LOG> handle:2196 Reply phase user_kbuh7a0kwa@mailbox.test(user_kbuh7a0kwa@mailbox.test) -> ra+s1r2_505975812@sl.local
  LOG> handle_reply:1051 Create <EmailLog 3> for <Contact 3 ext-133902@external.example 6>, <User 3 Test User user_kbuh7a0kwa@mailbox.test>, <Mailbox 3 user_kbuh7a0kwa@mailbox.test>
```

**(observed)** Both runs are identical in shape; `handle_DATA` returns `'250 Message accepted for delivery'`; the persisted `EmailLog.user_id` equals BOTH `contact.user_id` and `alias.user_id` (`agree=True`); the message is relayed to `contact.website_email` with `msg.From` rewritten to the alias; the user's real mailbox address (`user_glv8jsf45q@mailbox.test`) does NOT appear in the outbound headers. Stable across 2 runs — no run-to-run inconsistency on identical input.

#### S2 — Constructed divergence `alias.user_id != contact.user_id` (Q5a)

```text
==================== S2: CONSTRUCTED DIVERGENCE alias.user_id != contact.user_id (Q5a) ====================
BEFORE:
  alias.id=9 alias.user_id=4 contact.id=4 contact.user_id=5
  [dual-ref] alias.user_id=4 contact.user_id=5 EQUAL=False
AFTER:
  handle_DATA return status = '250 Message accepted for delivery'
  EmailLog.user_id=5 mailbox_id=4 alias_id=9 contact_id=4
  [Q5a] permission-user alias.user_id=4 ; recorded EmailLog.user_id=5 ; MISMATCH=True
  OUT envelope_to=ext-div-205245@external.example msg.From=widely_towhee311@sl.local
  LOG> _handle:2343 New message, mail from user_foz5if00t3@mailbox.test, rctp tos ['ra+div519003664@sl.local']
  LOG> handle:2196 Reply phase user_foz5if00t3@mailbox.test(user_foz5if00t3@mailbox.test) -> ra+div519003664@sl.local
  LOG> handle_reply:1051 Create <EmailLog 4> for <Contact 4 ext-div-205245@external.example 9>, <User 4 Test User user_foz5if00t3@mailbox.test>, <Mailbox 4 user_foz5if00t3@mailbox.test>
```

**(observed)** When `alias.user_id=4` but `contact.user_id=5`, the reply is ACCEPTED (`250`), permission checks run against `alias.user` (id **4**), yet the persisted `EmailLog.user_id` is **5** (`contact.user_id`). `MISMATCH=True` — this REPRODUCES "alias recognized but the recorded/owning user is different from the authorizing user." **(inferred)** This is the dual-reference defect: `user = alias.user` (`email_handler.py:L1004`) governs authorization/DMARC/mailbox, but `EmailLog.create(..., user_id=contact.user_id ...)` (`email_handler.py:L1046`) attributes the reply to a different account.

#### S3 — Non-unique `reply_email` resolved by `.first()` (Q5b)

```text
==================== S3: NON-UNIQUE reply_email -> get_by().first() (Q5b) ====================
[order=A_then_B] two Contact rows share reply_email=ra+dup_A_then_B_731270689@sl.local :
   contact.id=5 user_id=6 alias_id=12 website_email=ext-A-957750@external.example
   contact.id=6 user_id=7 alias_id=13 website_email=ext-B-257759@external.example
  Contact.get_by(reply_email).first() over 5 calls -> ids=[5, 5, 5, 5, 5]
  [Q5b] .first() CHOSE contact.id=5 user_id=6 website_email=ext-A-957750@external.example  (cA.id=5 userA=6 / cB.id=6 userB=7)
[order=B_then_A] two Contact rows share reply_email=ra+dup_B_then_A_212753284@sl.local :
   contact.id=7 user_id=9 alias_id=17 website_email=ext-B-450950@external.example
   contact.id=8 user_id=8 alias_id=16 website_email=ext-A-69216@external.example
  Contact.get_by(reply_email).first() over 5 calls -> ids=[7, 7, 7, 7, 7]
  [Q5b] .first() CHOSE contact.id=7 user_id=9 website_email=ext-B-450950@external.example  (cA.id=8 userA=8 / cB.id=7 userB=9)
```

**(observed)** Two `Contact` rows CAN share the same `reply_email` (the DB accepts it — `reply_email` is indexed but NOT unique, `app/models.py:L1899`; only `uq_contact(alias_id, website_email)`, `app/models.py:L1875`, is enforced). `Contact.get_by(reply_email).first()` (`email_handler.py:L986` → `app/models.py:L84`) returns exactly ONE row, and over 5 identical calls it is **stable** (`ids=[5, 5, 5, 5, 5]` and `[7, 7, 7, 7, 7]`) — it consistently returns the row with the **lowest primary key = the first-inserted contact**. **(inferred)** Because `.first()` has no `ORDER BY` tied to correctness, the "winner" is decided purely by insertion order, NOT by which contact the reply was actually meant for. Whichever contact was created first "captures" that `reply_email`, and its `user_id` and `website_email` are used — so a reply can be relayed to the wrong external recipient and attributed to the wrong user, while the alias is still "recognized."

#### S4 — Unauthorized sender → E214 (edge; spoofing check ON)

```text
==================== S4: UNAUTHORIZED mail_from -> E214 (edge) ====================
BEFORE: alias.disable_email_spoofing_check=False
AFTER: handle_DATA status='250 SL E214 Unauthorized for using reverse alias'  EmailLog(reply)=NONE  total_stored=1
  LOG> _handle:2343 New message, mail from attacker@evil.example, rctp tos ['ra+unauth999342638@sl.local']
  LOG> handle:2196 Reply phase attacker@evil.example(attacker@evil.example) -> ra+unauth999342638@sl.local
```

**(observed)** A sender NOT authorized for the alias's mailbox is rejected with **E214** ("Unauthorized for using reverse alias") (`email_handler.py:L1034` via `handle_unknown_mailbox` `email_handler.py:L1032`/`email_handler.py:L1390`); NO reply `EmailLog` is created; `total_stored=1` is the alert email sent to the legitimate user (not a relayed reply). Before/after state change: reply `EmailLog` = NONE (blocked).

#### S5 — `disable_email_spoofing_check` default-mailbox fallback (Q5c)

```text
==================== S5: disable_email_spoofing_check DEFAULT-MAILBOX FALLBACK (Q5c) ====================
BEFORE: alias.disable_email_spoofing_check=True alias.mailbox_id=11
AFTER: handle_DATA status='250 Message accepted for delivery'
  [Q5c] EmailLog created user_id=11 mailbox_id=11 (alias.mailbox_id=11) used_default=True
  OUT envelope_to=ext-472432@external.example msg.From=pewees_crones570@sl.local
  LOG> handle:2196 Reply phase stranger@nowhere.example(stranger@nowhere.example) -> ra+fb518685544@sl.local
  LOG> handle_reply:1023 ignore unknown sender to reverse-alias stranger@nowhere.example: <Alias 21 pewees_crones570@sl.local> -> <Contact 10 ext-472432@external.example 21>
  LOG> handle_reply:1051 Create <EmailLog 5> for <Contact 10 ext-472432@external.example 21>, <User 11 Test User user_9tdjc159yl@mailbox.test>, <Mailbox 11 user_9tdjc159yl@mailbox.test>
```

**(observed)** With `disable_email_spoofing_check=True`, an UNKNOWN sender (`stranger@nowhere.example`) is NOT rejected — the code logs "ignore unknown sender to reverse-alias" at `email_handler.py:L1023`, falls back to `mailbox = alias.mailbox` (the default) at `email_handler.py:L1029`, and proceeds to create the reply `EmailLog` (`user_id=11`, `mailbox_id=11 == alias.mailbox_id`, `used_default=True`). Contrast with `S4` (E214). **(inferred)** This fallback widens who can trigger a relay through the reverse alias — a routing-relevant behavior toggle, though it uses the correct owning mailbox in the normal-ownership case.

#### S6 — NOREPLIES short-circuit + bounce `mail_from == "<>"` (edge)

```text
==================== S6: NOREPLIES short-circuit + bounce mail_from=='<>' (edge) ====================
  config.NOREPLIES=['noreply@sl.local']
  [NOREPLIES] rcpt_to=noreply@sl.local -> status='250 Message accepted for delivery'
  [BOUNCE mail_from='<>'] rcpt_to=reverse-alias -> status='250 SL E206 Out of office'
```

**(observed)** A message to a `NOREPLIES` address short-circuits at `email_handler.py:L2181` (returns a 250 without reply handling). A bounce/auto-reply with `mail_from == "<>"` addressed to a reverse alias is handled as out-of-office → **E206** ("Out of office") at `email_handler.py:L2166`/`email_handler.py:L2173`, before `handle_reply` routing. These confirm the guard paths that precede/around the reply logic.

---

## (i) Observed vs. inferred — key-claim ledger

The distinction is annotated inline throughout. The most consequential claims are consolidated here:

| # | Claim | Basis |
|---|-------|-------|
| 1 | Reply ingress is `handle_DATA` → `_handle` → `handle` (`email_handler.py:L2289`, `L2335`, `L1945`) | **observed** (code + `S1` log lines `_handle:2343`, `handle:2196`) |
| 2 | The alias is resolved indirectly via `Contact` (`reply_email` → `Contact` → `Contact.alias` → `Alias.user`) | **observed** (code `email_handler.py:L986`/`L994`/`L1004`; `S1` chain `contact.id=2 → alias.id=4 → user.id=2`) |
| 3 | On the happy path the decided `user_id` is `2`, and `EmailLog.user_id == contact.user_id == alias.user_id` (`agree=True`) | **observed** (`S1`) |
| 4 | The system keeps TWO independent user references: `alias.user` (authz, `L1004`) and `contact.user_id` (persisted, `L1046`) | **observed** (code) |
| 5 | Those two references CAN diverge, yielding `EmailLog.user_id=5` while authz used `alias.user_id=4` | **observed** (`S2`, `MISMATCH=True`) |
| 6 | `reply_email` is not unique; two contacts can share it; `.first()` returns the lowest-PK/first-inserted row, stably | **observed** (`app/models.py:L1899`/`L1875`/`L84`; `S3` `ids=[5,5,5,5,5]`/`[7,7,7,7,7]`) |
| 7 | The happy-path reply egresses OUTWARD to `contact.website_email` with `From` rewritten to the alias; the real mailbox is hidden | **observed** (`S1` `OUT envelope_to=…`, `msg.From=gamuts_papaya338@sl.local`; real mailbox absent) |
| 8 | Unauthorized sender (spoofing ON) → E214, no reply `EmailLog` | **observed** (`S4`) |
| 9 | `disable_email_spoofing_check=True` → default-mailbox fallback, reply created | **observed** (`S5`) |
| 10 | `NOREPLIES` → 250 short-circuit; bounce `<>` → E206 | **observed** (`S6`) |
| 11 | For the correctly-owned common case, a "wrong user" report is most likely a misunderstanding of the outward-relay model | **inferred** (from claims 3 & 7) |
| 12 | The single most likely mis-routing origin is `Contact.get_by(reply_email=…).first()` (`email_handler.py:L986`) + non-unique schema + `.first()` | **inferred** (from claims 2 & 6; no abnormal ownership needed) |
| 13 | The dual user reference is a pipeline-wide pattern (forward path uses it too: `L557` vs `L600`/`L734`) | **observed** (code) / **inferred** (that it is the same pattern) |
| 14 | Divergent `alias.user_id`/`contact.user_id` is an abnormal state because `transfer_alias()` updates both together | **observed** (`app/alias_utils.py:L464–L466`, `L506`) / **inferred** (that divergence is therefore abnormal) |

---

## (j) Security / privacy note

**(observed)** The reverse-alias mechanism exists to keep the user's real mailbox hidden. This was confirmed in `S1`: the outbound headers expose only the alias (`msg.From=gamuts_papaya338@sl.local`) and the external contact (`envelope_to=ext-13474@external.example`), and the user's real mailbox address (`user_glv8jsf45q@mailbox.test`) does **not** appear anywhere in the relayed message.

**(inferred)** Therefore, any mis-routing that resolves the **wrong `Contact`/user** — Hypothesis (b), reproduced in `S3` — is not merely a correctness bug but a **privacy-relevant defect**: a reply could egress to the *wrong external party* (the first-inserted contact's `website_email`) and/or be attributed to the *wrong account* (`contact.user_id`), while the alias still appears "recognized." Because the whole point of the reverse alias is confidentiality, a wrong-contact resolution risks disclosing reply content to an unintended recipient.

---

## Coverage recap (final pass)

- **Q1 — Entry point:** answered in [§ (b)](#b-q1--which-component-handles-the-incoming-reply-entry-point) — `MailHandler.handle_DATA()` (`email_handler.py:L2289`) → `_handle()` (`L2335`) → `handle()` (`L1945`); production ingress at `127.0.0.1:20381` (`README.md:L367`).
- **Q2 — Resolution:** answered in [§ (c)](#c-q2--how-is-the-alias-resolved-to-a-user-resolution-chain) — `is_reverse_alias` (`app/email_utils.py:L1156`) → `handle_reply` (`L966`) → `Contact.get_by(reply_email).first()` (`L986` / `app/models.py:L84`) → `contact.alias` (`L994`) → `alias.user` (`L1004`).
- **Q3 — Decided `user_id`:** answered in [§ (d)](#d-q3--the-concrete-integer-user_id-the-system-decides) — happy-path value `2`; persisted via `EmailLog.create(user_id=contact.user_id …)` (`L1046`); two independent references (`alias.user` vs `contact.user_id`).
- **Q4 — Data flow:** answered in [§ (e)](#e-q4--actual-observed-end-to-end-data-flow) — full narrative + diagram, each node tied to `S1` evidence.
- **Q5 — Most likely origin:** answered in [§ (f)](#f-q5--most-likely-point-where-an-incorrect-routing-decision-originates) — three hypotheses reproduced (`S2`/`S3`/`S5`); single most likely origin = `Contact.get_by(reply_email=…).first()` (`email_handler.py:L986`) with the non-unique `reply_email` schema (`app/models.py:L1899`) and `.first()` semantics (`app/models.py:L84`).

*End of analysis. This document is the sole artifact produced; no source file was modified.*
