# SimpleLogin Alias Reply-Handling Flow — Runtime Root-Cause Analysis

**Source branch:** `app_2cd6ee777f8c`
**HEAD commit:** `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`
**Nature of this document:** a **read-only, runtime-first** investigation. The relevant code paths were BUILT and RUN first, the real output was captured, and this analysis is written from what was *observed* — not from reading the code alone. No source file was modified. The only artifact added to the repository is this document. All temporary observation scaffolding (a single pytest module, `tests/blitzy_reply_trace.py`, and its captured transcripts) lived **inside the run container** under `/tmp/slrun`, never inside the repository working tree, so the working tree stayed git-clean throughout (verified in [§ (k)](#k-cleanup--repository-state-proof)).

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
| **(observed)** | Came directly from captured runtime output (the raw scenario blocks `S1`–`S11` and `RUNTIME-01`/`-02`/`-INFO-01` in [§ (h)](#h-commands-run--raw-runtime-output)) **or** a direct, quoted line of source code with a `file:line` citation verified at HEAD `2cd6ee777f8c`. |
| **(inferred)** | A logical deduction drawn from the observed facts above. Inferences are conclusions, not measurements. |

All citations use the form `file:line` (e.g., `email_handler.py:L986`). Every runtime value, status string, and integer ID quoted in the narrative below is transcribed EXACTLY as emitted by a **single, coherent harness run** whose complete transcript is `tests/blitzy_reply_trace.py` → [§ (h)](#h-commands-run--raw-runtime-output); nothing is rounded, paraphrased, or adjusted toward an expected answer. To keep the document readable, the raw-output section reproduces **one full-fidelity, byte-for-byte unedited sample** (scenario `S1`, from interpreter start through the end of `S1`, including the complete timestamped SimpleLogin log stream); every **other** scenario block below it is explicitly labeled as **filtered** — it retains the `[...]` observation lines, the routing-relevant `LOG>` lines, and every `ASSERT PASS` line, but omits the repeated raw timestamped stream, which the harness re-emits identically for each scenario. Each embedded numeric result additionally carries an explicit in-harness assertion (`ASSERT PASS: …`), so the values are machine-verified, not hand-copied.

---

## (a) Verdict — real bug vs. misunderstanding (bottom line up front)

**Direct answer:** *Both interpretations are partly correct, and which one applies depends entirely on the data.* In the default, correctly-owned configuration the reply flow routes **correctly and deterministically**, so a "wrong user" report for the ordinary case is most likely a **misunderstanding** of the reverse-alias model. However, the pipeline contains **real, reproducible latent fragilities** that genuinely mis-route the reply *decision* under specific-but-plausible data conditions — so the behavior *can* be a real bug, not merely a misunderstanding. Beyond the two latent fragilities, this run also captured **two concrete, reproducible reply-body defects** (`RUNTIME-01`, `RUNTIME-02`) in which the alias is recognized yet the reply either **discloses a second user's external contact** or is **silently dropped** — see [§ (f)](#f-q5--most-likely-point-where-an-incorrect-routing-decision-originates) and [§ (j)](#j-security--privacy-note).

- **(observed)** In the DEFAULT, correctly-owned configuration (`contact.user_id == alias.user_id`, and a `reply_email` that maps to exactly one `Contact`), the reply flow routes **correctly and deterministically**. Scenario `S1` was run twice on the **exact same fixture and the exact same message bytes**: both runs returned `'250 Message accepted for delivery'`, both persisted `EmailLog.user_id=47` (equal to both `contact.user_id` and `alias.user_id`), and both relayed to the same external contact (`ext-s1_kdpnyi@external.example`) with `msg.From` rewritten to the alias (`cuppas_workup671@sl.local`). Among the compared routing-decision fields (status, `user_id`, `envelope_to`, rewritten `From`), the only value that changed between the two runs was the autoincrement `EmailLog.id` (`27` → `28`); the per-message `Message-ID` is independently regenerated each reply (`make_msgid(str(email_log.id), …)` — `email_handler.py:L1333`) and does not affect routing. Distribution of the routing-decision tuple over the two identical-input runs: **identical** (`ASSERT PASS: S1 routing invariants identical across 2 identical-input runs`) — **no run-to-run inconsistency** in the routing decision.
- **(inferred)** For that common case, a report of "the reply went to the wrong user" is most likely a **misunderstanding of the reverse-alias model**: a reply to a reverse alias legitimately egresses *outward* to the external contact's `website_email`, with the `From` header rewritten to the alias. SimpleLogin **relays the reply outward**; it does **not** deposit the reply into another SimpleLogin user's inbox. So "wrong user" only has a concrete technical meaning as either (i) the *wrong `Contact`* being resolved (→ wrong external recipient and wrong owning `user_id`), or (ii) a *divergence* between the user that authorizes the send (`alias.user`) and the user that gets recorded as owner (`contact.user_id`).
- **(observed + inferred)** The pipeline nevertheless contains **two reproducible latent fragilities** that affect the routing decision when the data allows it:
  - **Q5b — non-unique `reply_email` resolved by `Contact.get_by(...).first()`.** Two `Contact` rows can share the same `reply_email` (the column is indexed but **not** unique — `app/models.py:L1899`; the only uniqueness is `uq_contact(alias_id, website_email)` — `app/models.py:L1875`). The resolver `Contact.get_by(reply_email=…)` returns `.first()` with **no `ORDER BY`** (`app/models.py:L84`). Reproduced canonically **through `handle_DATA`** in `S3` (and, on a fixed unchanged fixture, `S3b`): with two contacts (owned by two different users) sharing one `reply_email`, `get_by(...).first()` selected exactly ONE contact for **every** reply to that reverse alias; the selected contact's owner replied successfully (`250`, routed to the selected contact) while the *other* (shadowed) owner's reply resolved to the selected owner's contact and was rejected with `E214`. In both insertion orders (`S3`) and across five repeats on one unchanged fixture (`S3b`) the selection was **observed** to be the first-inserted / lowest-PK row, but this ordering is **not guaranteed** by the query.
  - **Q5a — the dual user reference.** The reply handler computes **two independent references to "the user"** on the same run: the *authorizing* user `user = alias.user` (`email_handler.py:L1004`) drives the permission/mailbox gates, while the *persisted* owner is written as `user_id=contact.user_id` (`email_handler.py:L1046`). Reproduced in `S2`: with `alias.user_id=48` but `contact.user_id=49`, the reply was accepted (`250`) yet `EmailLog.user_id=49` while the authorization gates ran against `alias.user_id=48` — `MISMATCH(recorded_vs_authorizing)=True`.
- **(observed)** Two **concrete reply-body defects** were additionally reproduced, both with the alias fully "recognized":
  - **`RUNTIME-01` — cross-user recipient disclosure.** A reply whose `To` header lists two reverse aliases belonging to **different users** was relayed to user A's external contact with a rewritten `To` header of `extA_mjcrhr@aaa.example,extB_hrmrqa@bbb.example` — i.e., user **B's** external contact address was injected into user **A's** outbound reply, because `replace_header_when_reply` resolves each `To`-header reverse alias through a **global, un-scoped** `Contact.get_by` (`email_handler.py:L364`).
  - **`RUNTIME-02` — silent non-delivery on letter-case.** A reply whose `To`-header reverse alias differs only by **domain letter case** (`ra+up_shjmxq@SL.LOCAL` vs the envelope's `ra+up_shjmxq@sl.local`) was accepted at SMTP (`250`) but **never relayed** — the `EmailLog` was created then deleted and an internal notice was sent instead, because the same case-sensitive `Contact.get_by` (`email_handler.py:L364`) misses.
- **(inferred) Single most likely origin (see [§ (f)](#f-q5--most-likely-point-where-an-incorrect-routing-decision-originates)):** the **contact-resolution step `Contact.get_by(reply_email=…).first()` at `email_handler.py:L986`**, combined with the non-unique `reply_email` schema (`app/models.py:L1899`) and the `.first()`-without-`ORDER BY` semantics (`app/models.py:L84`). This is the single pivot from which the alias, the user, and the destination all hang — and it requires no abnormal ownership state to misfire, only two contacts that happen to share a `reply_email`. The dual user reference (Q5a) is the closely-related structural fragility that lets the *authorizing* user and the *recorded* user diverge; the un-scoped `Contact.get_by` in `replace_header_when_reply` (`email_handler.py:L364`) is the *same resolution weakness* surfacing in the body-rewrite step, producing `RUNTIME-01`/`-02`.

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
LOG> _handle:2343 New message, mail from user_juvpu3izq5@mailbox.test, rctp tos ['ra+s1_ahnude@sl.local']
LOG> handle:2196 Reply phase user_juvpu3izq5@mailbox.test(user_juvpu3izq5@mailbox.test) -> ra+s1_ahnude@sl.local
```

**(observed)** The `Reply phase …` line confirms that `handle()` classified `ra+s1_ahnude@sl.local` as a reverse alias and entered the reply branch, which then calls `handle_reply(envelope, copy_msg, rcpt_to)` (`email_handler.py:L2199`).

---

## (c) Q2 — How is the alias resolved to a user? (resolution chain)

**Direct answer (observed / code):** the alias is resolved **indirectly through the `Contact` reverse-alias record**. The recipient's `reply_email` is looked up to a `Contact`, and the owning user/alias are read off that contact: `reply_email` → `Contact` → `Contact.alias` → `Alias.user`. There is no direct `reply_email → User` lookup; the `Contact` row is the pivot.

Step-by-step, as exercised at runtime:

1. **Classification** — `handle()` calls `is_reverse_alias(rcpt_to)` (`app/email_utils.py:L1156`). The function returns `True` if `Contact.get_by(reply_email=address)` exists (`app/email_utils.py:L1158`), otherwise it returns whether the address ends with `@EMAIL_DOMAIN` **and** starts with `reply+` or `ra+` (`app/email_utils.py:L1161–L1163`). **(observed)** In `S1`, `is_reverse_alias(reply_email)=True`.
2. **Dispatch** — the reverse-alias branch calls `handle_reply(envelope, copy_msg, rcpt_to)` (`email_handler.py:L2199`); the reply handler is `def handle_reply(envelope, msg, rcpt_to) -> (bool, str)` (`email_handler.py:L966`).
3. **Normalize** — `reply_email = normalize_reply_email(reply_email)` (`email_handler.py:L984`), after a reply-domain validation gate (`email_handler.py:L977–L981`; a wrong reply domain returns `status.E501` — reproduced in `S8a`).
4. **Contact lookup (the pivot)** — `contact = Contact.get_by(reply_email=reply_email)` (`email_handler.py:L986`). `get_by` is `Session.query(cls).filter_by(**kw).first()` (`app/models.py:L83–L84`) — i.e., it returns exactly one row via `.first()`, with **no `ORDER BY`**. If no contact matches, the handler logs `No contact …` and returns `status.E502` (reproduced in `S8b`).
5. **Active-user gate** — `if not contact.user.is_active():` (`email_handler.py:L990`); a soft-deleted user yields `E502` (reproduced in `S8c`). **Note:** this gate reads `contact.user`, i.e., the *contact's* owning user.
6. **Alias from contact** — `alias = contact.alias` (`email_handler.py:L994`), followed by an alias-domain sanity check that yields `E503` on failure (reproduced in `S8d`).
7. **User from alias** — `user = alias.user` (`email_handler.py:L1004`); a user that `can_send_or_receive()` is `False` yields `E504` (`email_handler.py:L1007`; reproduced in `S8e`). **Note:** this gate reads `alias.user`, i.e., the *alias's* owning user — a *different* reference from step 5.

**(observed) The resolved chain in `S1` (run 1):** `contact.id=35` → `alias.id=85` (`cuppas_workup671@sl.local`) → `user.id=47`. The relationships used are `Contact.alias` (`app/models.py:L1907`) and `Contact.user` / `Alias.user` (`app/models.py:L1908`); the contact's own `user_id` column is `app/models.py:L1878`, and its `reply_email` column is `app/models.py:L1899`.

**(observed → M6, selection semantics)** The lookup at step 4 uses `.first()` with no `ORDER BY` (`app/models.py:L84`). When `reply_email` maps to exactly one `Contact` (the normal case, `S1`), the result is unambiguous. When two contacts share a `reply_email` (`S3`/`S3b`), `.first()` still returns exactly one row; the *specific* row it returns was **observed** to be the first-inserted / lowest-PK contact in every run, but the query gives **no ordering guarantee**, so this is an observation of those runs, not a contract.

---

## (d) Q3 — The concrete integer `user_id` the system decides

**Direct answer (observed):** on the happy path (`S1`) the system decided **`user_id = 47`** — and, critically, it computes **two independent user references** on the same run: the *authorizing* user `user = alias.user` (`email_handler.py:L1004`; `alias.user_id=47` in `S1`) and the *persisted* owner `EmailLog.user_id = contact.user_id` (`email_handler.py:L1046`; value `47` in `S1`). In `S1` they are equal; in `S2` they were deliberately made to diverge and did NOT agree.

- **(observed) Authorization user** — `user = alias.user` (`email_handler.py:L1004`) drives the permission / mailbox-authorization gates. In `S1`, `alias.user_id=47`.
- **(observed) Persisted user** — the reply is recorded by `EmailLog.create(...)` (`email_handler.py:L1042`) with these fields: `alias_id=contact.alias_id` (`email_handler.py:L1044`), `user_id=contact.user_id` (`email_handler.py:L1046`), `mailbox_id=mailbox.id` (`email_handler.py:L1047`), `is_reply=True`, `commit=True`. The `EmailLog` model columns are `class EmailLog` (`app/models.py:L2060`) → `user_id` (`app/models.py:L2064`), `contact_id`, `alias_id`, `is_reply`, `mailbox_id`.
- **(observed) The persisted row in `S1` run 1:**

```text
[RUN 1] EmailLog={'id': 27, 'user_id': 47, 'mailbox_id': 48, 'alias_id': 85, 'contact_id': 35, 'is_reply': True}
```

- **(observed) Confirming log line** (`LOG.d("Create %s for %s, %s, %s", email_log, contact, user, mailbox)` at `email_handler.py:L1051`):

```text
LOG> handle_reply:1051 Create <EmailLog 27> for <Contact 35 ext-s1_kdpnyi@external.example 85>, <User 47 Test User user_juvpu3izq5@mailbox.test>, <Mailbox 48 user_juvpu3izq5@mailbox.test>
```

- **(observed) Captured outbound** (via `mail_sender.get_stored_emails()` — `app/mail_sender.py:L108`): `envelope_to=ext-s1_kdpnyi@external.example`, `msg.From=cuppas_workup671@sl.local`. That is, the reply is relayed to `contact.website_email` with the `From` header rewritten to the alias.
- **(observed) Stability of the decided value** — the identical input was submitted twice; the decided `user_id` was `47` on both runs (the autoincrement `EmailLog.id` incremented, `27` → `28`; the per-message `Message-ID` also regenerates each run but does not affect the decided value). The routing-decision tuple was asserted identical across both runs.
- **(inferred) The crux:** two independent user references coexist. They are computed independently (`alias.user` vs `contact.user_id`), are normally equal, but **can diverge** — proven at runtime in `S2`, where `EmailLog.user_id=49` while the authorization gates used `alias.user_id=48` (`MISMATCH=True`). See [§ (f)](#f-q5--most-likely-point-where-an-incorrect-routing-decision-originates) hypothesis (a).

---

## (e) Q4 — Actual observed end-to-end data flow

**Direct answer:** the observed path, from SMTP `DATA` to the outbound relayed message, is:

`handle_DATA` → `_handle` → `handle` → `is_reverse_alias` → `handle_reply` → `Contact.get_by(reply_email).first()` → `contact.user.is_active()` gate → `alias = contact.alias` / `user = alias.user` + `user.can_send_or_receive()` gate → `apply_dmarc_policy_for_reply_phase(alias, contact, …)` → `get_mailbox_from_mail_from(mail_from, alias)` → `EmailLog.create(user_id=contact.user_id, mailbox_id=mailbox.id, is_reply=True)` → rewrite `From` to alias, relay to `contact.website_email`.

**(observed / code) Narrative with citations and the `S1`/`S2` evidence tying each node to a captured value:**

1. **SMTP `DATA`** → `MailHandler.handle_DATA()` (`email_handler.py:L2289`). *Observed:* the run began by driving this callback directly.
2. **Flask app context** → `_handle()` (`email_handler.py:L2335`). *Observed:* `LOG> _handle:2343 New message, mail from user_juvpu3izq5@mailbox.test, rctp tos ['ra+s1_ahnude@sl.local']`.
3. **Routing hub** → `handle()` (`email_handler.py:L1945`), per-recipient loop (`email_handler.py:L2180`).
4. **Classify recipient** → `is_reverse_alias(rcpt_to)` (`app/email_utils.py:L1156`). *Observed:* `is_reverse_alias(reply_email)=True`; reverse-alias branch at `email_handler.py:L2195`, `LOG> handle:2196 Reply phase … -> ra+s1_ahnude@sl.local`.
5. **Reply handler** → `handle_reply()` (`email_handler.py:L966`, called at `email_handler.py:L2199`).
6. **Resolve contact (pivot)** → `contact = Contact.get_by(reply_email=reply_email)` (`email_handler.py:L986`) = `filter_by(...).first()` (`app/models.py:L84`). *Observed:* `contact.id=35`.
7. **Gate 1 — contact's user active?** → `if not contact.user.is_active()` (`email_handler.py:L990`). *Observed (S2 divergence):* this gate reads **`contact.user`** — `contact.user.id=49` in `S2` run 1.
8. **Resolve alias & Gate 2 — alias's user can send?** → `alias = contact.alias` (`email_handler.py:L994`), `user = alias.user` (`email_handler.py:L1004`), `if not user.can_send_or_receive()` (`email_handler.py:L1007`). *Observed:* `alias.id=85`, `alias.user_id=47` in `S1`; in `S2` this gate reads **`alias.user`** — `alias.user.id=48`, a *different* reference from Gate 1.
9. **Gate 3 — DMARC policy (gets BOTH references)** → `dmarc_delivery_status = apply_dmarc_policy_for_reply_phase(alias, contact, envelope, msg)` (`email_handler.py:L1012`; function at `app/handler/dmarc.py:L154`, signature `(alias_from: Alias, contact_recipient: Contact, envelope, msg)`). *Observed:* on every reply-path run the function logged `DMARC check disabled` (`app/handler/dmarc.py:L159`) and returned `None` (non-blocking); see the DMARC note below and [§ the ledger](#i-observed-vs-inferred--key-claim-ledger).
10. **Gate 4 — authorize sending mailbox (alias-scoped)** → `mailbox = get_mailbox_from_mail_from(mail_from, alias)` (`email_handler.py:L1019`; function at `email_handler.py:L1364`, matching `mail_from` against **`alias.mailboxes`** and their `authorized_addresses`, raw then canonicalized). *Observed:* `mailbox.id=48` in `S1`. This gate is scoped to the *alias's* mailboxes — i.e., to `alias.user`, not `contact.user`.
11. **Persist the decision (records contact's user)** → `EmailLog.create(user_id=contact.user_id, mailbox_id=mailbox.id, is_reply=True, …)` (`email_handler.py:L1042–L1047`). *Observed:* `EmailLog{'id': 27, 'user_id': 47, 'mailbox_id': 48, 'alias_id': 85, 'contact_id': 35, 'is_reply': True}`. This records **`contact.user_id`**, closing the loop back to Gate 1's reference rather than Gate 2/4's `alias.user`.
12. **Rewrite & relay** → `From` rewritten to the alias, message relayed to `contact.website_email`. *Observed:* `OUT envelope_to=ext-s1_kdpnyi@external.example | msg.From=cuppas_workup671@sl.local`.

**(observed) DMARC note (grounds the ledger nuance):** the reply was **not** blocked by DMARC, but this is because DMARC evaluation is **non-blocking** for these messages, not because of a "clean pass." `apply_dmarc_policy_for_reply_phase` first extracts `spam_result = SpamdResult.extract_from_headers(msg, Phase.reply)` (`app/handler/dmarc.py:L157`); when `not DMARC_CHECK_ENABLED or not spam_result` it logs `DMARC check disabled` (`app/handler/dmarc.py:L159`) and returns `None` (`app/handler/dmarc.py:L160`). The synthetic messages carry no spamd headers, so `spam_result` is falsy and the function short-circuits to `None` on every run. The **observed** outcome is therefore "DMARC returned a non-blocking result", not "DMARC passed."

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
    M --> N["Rewrite From to alias, relay to<br/>contact.website_email<br/>(To-header rewrite: replace_header_when_reply L364 &lt;-- RUNTIME-01/-02)"]
```

---

## (f) Q5 — Most likely point where an incorrect routing decision originates

Code-grounded hypotheses were tested at runtime. Each is presented with its reproduced result, then the single most likely origin is named. Hypotheses (a)–(c) are the *routing-decision* fragilities; hypothesis (d) is the *reply-body rewrite* defect reproduced in `RUNTIME-01`/`RUNTIME-02`.

### Hypothesis (a) — Dual user reference (`alias.user` vs `contact.user_id`)

- **(observed / code)** The reply handler uses **distinct user references at distinct gates** — it is *not* "`alias.user` everywhere":
  - Gate 1, active-user check, reads **`contact.user`**: `if not contact.user.is_active()` (`email_handler.py:L990`).
  - Gate 2, send-permission check, reads **`alias.user`**: `user = alias.user` (`email_handler.py:L1004`), `if not user.can_send_or_receive()` (`email_handler.py:L1007`).
  - Gate 3, DMARC, receives **both** the alias and the contact: `apply_dmarc_policy_for_reply_phase(alias, contact, envelope, msg)` (`email_handler.py:L1012`).
  - Gate 4, mailbox authorization, is scoped to the **alias's** mailboxes: `get_mailbox_from_mail_from(mail_from, alias)` (`email_handler.py:L1019` / `email_handler.py:L1364`).
  - Persistence records the **contact's** user: `user_id=contact.user_id` (`email_handler.py:L1046`).
- **(observed) Reproduced in `S2` (×2):** with `alias.user_id=48` and `contact.user_id=49` (run 1), the reply was accepted (`'250 Message accepted for delivery'`), yet `EmailLog.user_id=49` while the send/mailbox gates ran against `alias.user_id=48` — `MISMATCH(recorded_vs_authorizing)=True`. Run 2 (`alias.user_id=50`, `contact.user_id=51`) reproduced the same mismatch (`EmailLog.user_id=51`). The confirming `Create` log even prints the alias's user object while the row stores the contact's user: `Create <EmailLog 29> for <Contact 36 …>, <User 48 …>` — i.e., `<User 48>` (alias.user) is logged while `EmailLog.user_id=49` (contact.user_id).
- **(observed) Context:** the standard ownership-transfer path `transfer_alias()` (`app/alias_utils.py:L458–L540`) updates `Contact.user_id` (`app/alias_utils.py:L464–L466`) AND `alias.user_id` (`app/alias_utils.py:L506`) **together**.
- **(inferred)** Because the normal transfer path keeps the two references in sync, a divergence between `alias.user_id` and `contact.user_id` is an **abnormal/inconsistent data state**, not a normal one. When it does occur, the *authorizing* user and the *recorded owner* refer to different accounts. **(inferred, sibling variant)** The same dual reference exists in the forward path — `user = alias.user` (`email_handler.py:L557`) vs `user_id=contact.user_id` (`email_handler.py:L600` and `email_handler.py:L734`) — so this is a pipeline-wide pattern, not a one-off in the reply handler.

### Hypothesis (b) — Non-unique `reply_email` resolved by `.first()`

- **(observed / code)** The `reply_email` column is indexed but **NOT unique** — `reply_email = sa.Column(sa.String(512), nullable=False, index=True)` (`app/models.py:L1899`). The **only** uniqueness constraint on `Contact` is `uq_contact(alias_id, website_email)` (`app/models.py:L1875`). The resolver `Contact.get_by(reply_email=…)` (`email_handler.py:L986`) is `Session.query(cls).filter_by(**kw).first()` (`app/models.py:L84`) — a `.first()` with **no correctness `ORDER BY`**.
- **(observed) Reproduced canonically through `handle_DATA` in `S3` (×2, both insertion orders):** two `Contact` rows — owned by **two different users** — were created sharing one `reply_email`, and the DB accepted both (`DB row count for that reply_email = 2`), confirming non-uniqueness. Then a real inbound reply was driven through `MailHandler.handle_DATA` for each owner:
  - `order=A_then_B` (shared `reply_email=ra+dup_a_then_b_isgulexe@sl.local`): `Contact.get_by(reply_email).first()` **selected `contact.id=38` (user 52, `ext-a-dup_a_then_b_xduyem@external.example`)**; `selected == first-inserted-contact? True`. Reply from the **selected** owner's mailbox → `'250 Message accepted for delivery'`, `EmailLog{'id': 31, 'user_id': 52, …, 'contact_id': 38}`, relayed to the selected contact. Reply from the **shadowed** owner's mailbox → `'250 SL E214 Unauthorized for using reverse alias'` (`new_reply_EmailLog_created=False`) — the shadowed owner's identical `reply_email` resolves to the *other* user's `contact.id=38`, whose alias the shadowed mailbox is not authorized on, so their reply is **not** delivered to their intended contact.
  - `order=B_then_A` (shared `reply_email=ra+dup_b_then_a_unyszbyf@sl.local`): `get_by(...).first()` **selected `contact.id=40` (user 54, `ext-b-dup_b_then_a_haufjc@external.example`)**; `selected == first-inserted-contact? True`. Selected owner → `250`, `EmailLog{'id': 32, 'user_id': 54, …, 'contact_id': 40}`. Shadowed owner → `E214`.
  - Summary line: `[S3] OBSERVED selection==first-inserted in both runs: True (run1=True run2=True). NOTE: get_by()=filter_by().first() has NO ORDER BY (models.py:L84); this selection was OBSERVED, not guaranteed.`
- **(observed) Same-fixture reproduction — `S3b` (×5 on the ONE unchanged fixture, per the "reproduce reported inconsistency directly" rule):** two contacts (`id=42`, `id=43`) share `reply_email=ra+dupfix_weavrgpl@sl.local`; `get_by(...).first()` selected `id=42`. Driving the **selected**-owner reply five times on that identical fixture produced the identical tuple every time — `('250 Message accepted for delivery', 42, 'ext-fixA_wiskvc@external.example')` ×5 — and the **shadowed**-owner reply produced `('250 SL E214 Unauthorized for using reverse alias', True)` ×5. So on a fixed collision the winner is **stable** within a process (not flapping run-to-run); the reported "sometimes wrong user" is explained by *which* contact wins the collision (data-dependent), not by per-request nondeterminism on one fixture.
- **(inferred)** Because `.first()` has **no `ORDER BY` tied to correctness**, *which* contact "wins" is not determined by which contact the reply was meant for — it is determined by whatever single row the database returns first (observed here to be the first-inserted/lowest-PK row, but not guaranteed to be). Whichever contact wins dictates the alias, the recorded `user_id`, and the destination `website_email` for **every** reply to that shared `reply_email`. The observed consequence is the reported symptom exactly: the alias is still "recognized" (`is_reverse_alias=True`), yet one user's reply is routed/attributed via the *other* user's contact — succeeding for the selected owner and being rejected (`E214`) for the shadowed owner.

### Hypothesis (c) — `disable_email_spoofing_check` default-mailbox fallback

- **(observed / code)** When `get_mailbox_from_mail_from` returns no mailbox (`if not mailbox:` — `email_handler.py:L1020`), the code branches on `if alias.disable_email_spoofing_check:` (`email_handler.py:L1021`). If the check is disabled, it logs "ignore unknown sender to reverse-alias" (`email_handler.py:L1023`) and falls back to `mailbox = alias.mailbox` (`email_handler.py:L1029`). Otherwise it calls `handle_unknown_mailbox(...)` (`email_handler.py:L1032`) and returns `status.E214` (`email_handler.py:L1034`).
- **(observed) Reproduced as a controlled differential in `S9` (one fixture, one stranger sender, only the flag toggled):** with the **same** alias (`id=115`, default `mailbox_id=64`), the **same** stranger `mail_from=stranger-s9_atvwcp@nowhere.example`, and the **same** reverse alias `ra+s9_hhfcum@sl.local`:
  - `spoofing_check=ENABLED` (`disable_email_spoofing_check=False`) → `'250 SL E214 Unauthorized for using reverse alias'`, `reply_EmailLog=None`, `total_stored=1` (the owner alert only), `relay_to_contact=no`.
  - `spoofing_check=DISABLED` (`disable_email_spoofing_check=True`) → `'250 Message accepted for delivery'`, `reply_EmailLog={'id': 39, 'user_id': 63, 'mailbox_id': 64, 'alias_id': 115, 'contact_id': 49, …}`, `used_default_mailbox=True` (`mailbox_id == alias.mailbox_id`, `64 == 64`), `relay_to_contact=yes`.
  - `DIFFERENTIAL: identical stranger+rcpt; ONLY the spoofing flag changed the outcome E214 -> E200(default-mailbox fallback)`.
- **(inferred)** This fallback widens *who* can trigger a relay through the reverse alias, but it uses the **correct owning mailbox** (`alias.mailbox`) in the normal-ownership case, so it changes *who can send*, not *which user the reply is attributed to*. It is a routing-relevant behavior toggle rather than a mis-attribution defect on its own.

### Hypothesis (d) — Un-scoped `Contact.get_by` in the To-header rewrite (`RUNTIME-01`/`RUNTIME-02`)

- **(observed / code)** After the routing decision, `handle_reply` rewrites the outbound `To`/`Cc` via `replace_header_when_reply(msg, alias, "To")` (`email_handler.py:L1177–L1180`). For **each** address in that header, `replace_header_when_reply` resolves `contact = Contact.get_by(reply_email=reply_email)` (`email_handler.py:L364`) — a **global** lookup that is **not scoped** to the current alias/user; if it returns `None` it raises `NonReverseAliasInReplyPhase` (`email_handler.py:L370`), and `handle_reply` then **deletes the `EmailLog`** and sends an internal notice instead of relaying (`email_handler.py:L1183–L1198`).
- **(observed) `RUNTIME-01` — cross-user recipient disclosure.** A reply from user **A**'s mailbox with `To: ra+a_iawqrq@sl.local, ra+b_qfowxc@sl.local` (A's reverse alias + **B's** reverse alias) was accepted (`'250 Message accepted for delivery'`). A's leg relayed to `extA_mjcrhr@aaa.example`, but the rewritten `To` header became `extA_mjcrhr@aaa.example,extB_hrmrqa@bbb.example` — user **B's** external contact (`extB_hrmrqa@bbb.example`) was injected into user **A's** outbound reply, because `replace_header_when_reply` resolved B's reverse alias through the un-scoped `Contact.get_by` (`email_handler.py:L364`). (B's own leg → `E214`.)
- **(observed) `RUNTIME-02` — silent non-delivery on letter-case.** A reply whose envelope recipient was `ra+up_shjmxq@sl.local` (lowercase) but whose `To` header carried `ra+up_shjmxq@SL.LOCAL` (uppercase domain) was accepted (`250`), yet `reply_EmailLog_after=None` and `relay_to_contact=no`: the case-sensitive `Contact.get_by` in `replace_header_when_reply` (`email_handler.py:L364`) missed, `NonReverseAliasInReplyPhase` fired, the `EmailLog` was deleted, and an internal notice (`'Email sent to ext-up_wltmkt@external.example contains non reverse-alias addresses'`) was stored instead of the reply. The reply is **silently dropped** despite the SMTP `250`.
- **(inferred)** This is the *same resolution weakness* as hypothesis (b) — a `Contact.get_by` on `reply_email` that is neither uniqueness-guaranteed nor scoped to the owning user — but surfacing in the **body-rewrite** step rather than the routing pivot. It converts the recognized-alias reply into either a **confidentiality breach** (`RUNTIME-01`) or a **lost message** (`RUNTIME-02`).

### Conclusion — the single most likely origin

- **(inferred, evidence-led)** The **most likely origin** of an *incorrect routing decision that still shows "alias recognized"* is the **contact-resolution step `Contact.get_by(reply_email=…).first()` at `email_handler.py:L986`**, combined with the non-unique `reply_email` schema (`app/models.py:L1899`) and the `.first()`-without-`ORDER BY` semantics (`app/models.py:L84`) — **Hypothesis (b)**. Cause → effect: `reply_email` is not unique, so multiple `Contact` rows can carry it; `.first()` then returns one row with no correctness ordering; and because *everything downstream hangs off that single contact* — the alias (`contact.alias`), the recorded user (`contact.user_id`), and the destination (`contact.website_email`) — resolving to the wrong contact simultaneously mis-routes/rejects the reply and mis-attributes the user, all while `is_reverse_alias` still reports the alias as "recognized." This needs **no abnormal ownership state** and is reachable whenever two contacts share a `reply_email` (reproduced canonically in `S3`, and shown stable-on-a-fixed-fixture in `S3b`).
- **(inferred)** **Hypothesis (a)** (the dual user reference, `alias.user` at `email_handler.py:L1004` vs `contact.user_id` at `email_handler.py:L1046`) is the closely-related **structural fragility** that makes the *authorizing* user and the *recorded* user diverge once the two `user_id`s are inconsistent (reproduced in `S2`). It amplifies (b) but, on its own, requires the abnormal divergent-ownership state that `transfer_alias()` normally prevents.
- **(inferred)** **Hypothesis (d)** shows the *same* `Contact.get_by`-on-`reply_email` weakness in the To-header rewrite (`email_handler.py:L364`) producing concrete, alias-recognized defects with no abnormal ownership at all — a **disclosure** (`RUNTIME-01`) and a **silent drop** (`RUNTIME-02`). It reinforces that the reverse-alias→contact resolution (unscoped, non-unique, case-sensitive) is the pipeline's central routing weakness.
- **(inferred)** **Hypothesis (c)** is a secondary, configuration-gated widening of sender acceptance; it does not by itself send the reply to the wrong user under normal ownership.

---

## (g) Edge-condition results

Every distinct condition the question implies was exercised — the happy path plus the secondary/edge/error paths — through the real entry point (`MailHandler.handle_DATA`), **each scenario at least twice** (or, for the guard/differential scenarios, once per distinct branch with the branch explicitly toggled). Before/after states are reported where state changes. Every embedded value carries an explicit in-harness assertion (`ASSERT PASS: …`) in the raw transcript, [§ (h)](#h-commands-run--raw-runtime-output). The exercised set includes the multi-mailbox alias condition called out in the AAP (§0.5.1): `S10` drives replies from an alias's default mailbox, its secondary (non-default) authorized mailbox, an authorized delegate address, **and** an unauthorized stranger, and additionally verifies the `notify_mailbox` fan-out.

| Scenario | Condition | Runs | Observed status | Reply `EmailLog` (before → after) | Key citation |
|----------|-----------|------|-----------------|-----------------------------------|--------------|
| `S1` | Happy path, correctly-owned, identical input | ×2 | `'250 Message accepted for delivery'` (both) | none → created (`user_id=47`, equal to both refs; `EmailLog.id` 27→28, row fields otherwise identical) | `email_handler.py:L1042–L1047` |
| `S2` | Constructed divergence `alias.user_id != contact.user_id` | ×2 | `'250 Message accepted for delivery'` (both) | none → created (`EmailLog.user_id=49`/`51`; gates used `alias.user_id=48`/`50`; `MISMATCH=True`) | `email_handler.py:L1004` vs `L1046` |
| `S3` | Non-unique `reply_email` (two contacts, two users share it) — canonical `handle_DATA`, both insertion orders | ×2 | selected owner `'250 …'`; shadowed owner `'250 SL E214 …'` | none → created for selected owner (`id=31`/`32`); shadowed owner blocked (E214) | `email_handler.py:L986` / `app/models.py:L84`, `L1899`, `L1875` |
| `S3b` | Non-unique `reply_email`, **same unchanged fixture**, repeated | ×5 | selected owner `'250 …'` ×5 (contact `42`); shadowed owner `'250 SL E214 …'` ×5 | created for selected owner each time; shadowed blocked each time — **stable** | `email_handler.py:L986` / `app/models.py:L84` |
| `S8a` | Reply-domain not `EMAIL_DOMAIN`/no `SLDomain` | ×1 | `'550 SL E501'` | none → **NONE** (rejected before contact lookup) | `email_handler.py:L977–L981` |
| `S8b` | Reverse-alias shape, **no `Contact`** | ×1 | `'550 SL E502 Email not exist'` | none → **NONE** | `email_handler.py:L986–L989` |
| `S8c` | Contact's user soft-deleted (`is_active=False`) | ×1 | `'550 SL E502 Email not exist'` | none → **NONE** | `email_handler.py:L990–L992` |
| `S8d` | Alias on an unmanaged domain (`is_valid_alias_address_domain=False`) | ×1 | `'550 SL E503'` | none → **NONE** | `email_handler.py:L1000–L1002` |
| `S8e` | Alias user disabled (`can_send_or_receive=False`, `is_active=True`) | ×1 | `'550 SL E504 Account disabled'` | none → **NONE** | `email_handler.py:L1007–L1009` |
| `S8f` | Reverse alias with a `\x01` control char in the local part (normalization) | ×1 | `'250 Message accepted for delivery'` | none → created (`id=38`, relayed to `ext-norm_igxgdd@external.example`) | `email_handler.py:L984`, `app/email_validation.py:L25–L39` |
| `S9` | `disable_email_spoofing_check` OFF→ON, **same** stranger + rcpt (controlled differential) | ×2 (ENABLED, DISABLED) | ENABLED → `'250 SL E214 …'`; DISABLED → `'250 Message accepted for delivery'` | ENABLED: none → **NONE** (owner alert only); DISABLED: none → created (`id=39`, `used_default_mailbox=True`) | `email_handler.py:L1021`/`L1029`/`L1034` |
| `S10` | Multi-mailbox alias — default / secondary / authorized-delegate / **unauthorized** senders + `notify_mailbox` | ×4 senders | default/secondary/authorized → `'250 …'`; unauthorized → `'250 SL E214 …'` | created for the three authorized senders (`EmailLog.mailbox_id` == the **sending** mailbox `65`/`66`/`65`); unauthorized blocked | `email_handler.py:L1019`/`L1364`/`L1047`; notify `email_handler.py:L1264–L1293` |
| `S11` | `NOREPLIES` address; bounce `mail_from == "<>"` | ×2 | `NOREPLIES → '250 Message accepted for delivery'` (auto-response); bounce `→ '250 SL E206 Out of office'` | `EmailLog_delta=0` (no reply routing performed) | `email_handler.py:L2181–L2183`; `email_handler.py:L2166` |
| `RUNTIME-01` | `To` header lists **two** reverse aliases owned by **different users** | ×1 | overall `'250 Message accepted for delivery'` | A's leg created & relayed (with B's contact disclosed); B's leg → E214 | `email_handler.py:L364` / `L1177–L1180` |
| `RUNTIME-02` | `To`-header reverse alias differs only by domain **letter case** | ×1 | `'250 Message accepted for delivery'` (but not relayed) | created → **deleted** (`reply_EmailLog_after=None`); internal notice stored instead | `email_handler.py:L364` / `L1183–L1198` |
| `RUNTIME-INFO-01` | Identical message bytes (same `Message-ID`) driven twice | ×2 | `'250 Message accepted for delivery'` (both) | two distinct rows created (`id=45`, `id=46`); **both relayed** | `email_handler.py:L1042–L1046` |

- **(observed) `S8` — `handle_reply` guard paths (before → after):** each guard was reached through the real `handle_DATA` and returned its exact SMTP status with **no reply `EmailLog` created**, except the normalization case which succeeds:
  - `S8a` `reply_email=ra+e501_mpvjur@notsl.example` (domain not `EMAIL_DOMAIN`, not an `SLDomain`), `is_reverse_alias=True` → `'550 SL E501'`, `new_EmailLog_created=False` (`email_handler.py:L977–L981`).
  - `S8b` `ra+nocontact_ckirsngi@sl.local` (no `Contact` row), `is_reverse_alias=True` (matched via the `@EMAIL_DOMAIN`+`ra+` branch) → `'550 SL E502 Email not exist'` (`email_handler.py:L986–L989`).
  - `S8c` before `user.is_active=True` → set `delete_on=FUTURE` → after `user.is_active=False` → `'550 SL E502 Email not exist'` (`email_handler.py:L990–L992`; `is_active` at `app/models.py:L766–L769`).
  - `S8d` before `alias.email=longer_doomed380@sl.local` → after `alias.email=orphan_brgezx@unmanaged.example` (domain `unmanaged.example`) → `'550 SL E503'` (`email_handler.py:L1000–L1002`; `is_valid_alias_address_domain` at `app/email_utils.py:L557–L566`).
  - `S8e` before `can_send_or_receive=True` → set `disabled=True, delete_on=None` → after `can_send_or_receive=False`, `is_active=True` → `'550 SL E504 Account disabled'` (`email_handler.py:L1007–L1009`; `can_send_or_receive` at `app/models.py:L886–L889`).
  - `S8f` stored `reply_email='ra+norm_kixgja@sl.local'` but the envelope recipient carried a `\x01` control byte (`'ra+norm\x01kixgja@sl.local'`); `is_reverse_alias(raw)=True` (via the `@EMAIL_DOMAIN`+`ra+` branch since `Contact.get_by(raw)=None`); `normalize_reply_email` (`email_handler.py:L984`; `app/email_validation.py:L25–L39`) mapped the byte back so the contact resolved → `'250 Message accepted for delivery'`, `EmailLog{'id': 38, …}`, relayed to `ext-norm_igxgdd@external.example`.
- **(observed) `S9` — controlled spoofing differential:** identical stranger sender and identical reverse-alias recipient; **only** `disable_email_spoofing_check` was toggled. ENABLED → `E214`, no reply `EmailLog`, one stored email (the owner alert). DISABLED → `250`, reply `EmailLog{'id': 39, 'user_id': 63, 'mailbox_id': 64, …}` with `used_default_mailbox=True` (`mailbox_id == alias.mailbox_id == 64`), relayed to the contact. This is the same behavior as the former separate S4/S5 pair, but now proven as a *single-input differential* so the flag is the only variable.
- **(observed) `S10` — multi-mailbox alias, sender-driven selection + `notify_mailbox`:** an alias with two verified mailboxes (default `mb1.id=65`, secondary `mb2.id=66`; `alias.mailboxes=[66, 65]`) and an authorized delegate address mapped to `mb1`. `get_mailbox_from_mail_from(mail_from, alias)` (`email_handler.py:L1364`) selected the mailbox matching `mail_from`, and the persisted `EmailLog.mailbox_id` (`email_handler.py:L1047`) equaled that **sending** mailbox: default-send → `65`, secondary-send → **`66`** (`!= alias.mailbox_id`), delegate-send → mapped default `65`. Each authorized reply additionally emitted a **`notify_mailbox`** message to the alias's *other* verified mailbox (`email_handler.py:L1264–L1293`): the default-send `stored_envelope_to=['ext-s10_uoybzx@external.example', 'moaltkkpasbtwhuuundh@moaltkkpasbtwhuuundh.com']` (relay + notify to `mb2`); the secondary-send `stored_envelope_to=['ext-s10_uoybzx@external.example', 'user_p9pty8v2qr@mailbox.test']` (relay + notify to the default). The **unauthorized** stranger (`stranger-s10_ccaqgz@evil.example`) → `E214`, `stored_envelope_to=['user_p9pty8v2qr@mailbox.test']` (owner alert only). Multi-mailbox authorization routes to the **correct sending mailbox**, not merely the default — so it is **not** a source of the reported mis-routing.
- **(observed) `S11` — `NOREPLIES` + bounce, with `EmailLog` deltas and stored bodies:** a message to `noreply@sl.local` short-circuited via `send_no_reply_response` (`email_handler.py:L2181–L2183`; `app/email_utils.py`), returning `'250 Message accepted for delivery'` with `EmailLog_delta=0` and an **auto-response** captured — `stored_subjects=['Auto: hello s11r1']` (run 2 `['Auto: hello s11r2']`). A bounce/auto-reply with `mail_from == "<>"` addressed to a reverse alias was handled as out-of-office → `'250 SL E206 Out of office'` (`email_handler.py:L2166`) with `EmailLog_delta=0` and `total_stored=0` — i.e., no reply routing at all. Reproduced on both runs.
- **(observed) `RUNTIME-01` / `RUNTIME-02` / `RUNTIME-INFO-01`:** detailed under [§ (f) hypothesis (d)](#hypothesis-d--un-scoped-contactget_by-in-the-to-header-rewrite-runtime-01runtime-02) and [§ (j)](#j-security--privacy-note). `RUNTIME-INFO-01` additionally shows the pipeline is **at-least-once with no `Message-ID` de-duplication**: two byte-identical inbound replies (same `Message-ID`) produced two distinct `EmailLog` rows (`id=45`, `id=46`) and **both** were relayed.

---

## (h) Commands run & raw runtime output

### Environment (observed — canonical / default configuration)

The investigation ran inside the **user-provided canonical execution platform**: the container built from the GHCR image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (image id `ea242796bbce`), which ships the project at commit `2cd6ee77` with its own virtualenv (`/app/venv`), PostgreSQL, and Redis. The exact commands and their outputs (complete for these short commands):

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
- **(observed) Infrastructure version deviation from the CI recipe.** `.github/workflows/main.yml` pins the CI matrix to **PostgreSQL 13** and **Redis 6** (with Python 3.10). The canonical image ships **PostgreSQL 15.13** and **Redis 7.0.15** (Python 3.10.18). This is a benign infrastructure deviation: the schema is built by the same `alembic upgrade head` recipe (reaching head `32f25cbf12f6`, 77 tables) and the reply-handler code paths under test are database-version-agnostic. The deviation is reported here rather than hidden.
- **(observed) Runtime toolchain deviation (harness only).** Two further deviations exist and neither touches the reply pipeline: (1) the image venv's **pytest is `8.4.1`** whereas `poetry.lock` pins **`7.3.1`** — a test-runner override that affects only how the temporary observation module is executed, not any product code path; the harness uses no pytest-8-specific API. (2) The venv ships **`google-re2` (`1.1.20250805`)** rather than the locked `pyre2`, and the image applies a one-line `import re2 as re` → `import re` shim in `app/spamassassin_utils.py`. That module is only reached when `ENABLE_SPAM_ASSASSIN=True`; the effective config has **`ENABLE_SPAM_ASSASSIN=False`**, so the reply path never imports it. Both deviations are disclosed for integrity and have **zero** effect on any cited `file:line` or observed value.

### Existing-suite regression check (`tests/test_email_handler.py`)

Before and after the investigation, the project's own reply-handler suite was run through the same canonical venv to confirm the environment is sound and nothing was perturbed. It passed identically on two consecutive runs:

```text
$ CONFIG=tests/test.env GITHUB_ACTIONS_TEST=true /app/venv/bin/python -m pytest tests/test_email_handler.py --timeout=60 -q    # run 1
...
23 passed, 18 warnings in 6.41s
$ echo $?
0

$ CONFIG=tests/test.env GITHUB_ACTIONS_TEST=true /app/venv/bin/python -m pytest tests/test_email_handler.py --timeout=60 -q    # run 2
...
23 passed, 18 warnings in 6.55s
$ echo $?
0
```

### Configuration precedence & migration idempotence

- **(observed) Effective (in-process) configuration.** With `CONFIG=tests/test.env`, the values actually loaded into `app.config` at runtime were `NOT_SEND_EMAIL=True`, `EMAIL_DOMAIN='sl.local'`, `DB_URI='postgresql://test:test@localhost:15432/test'`, `NOREPLIES=['noreply@sl.local']`, `ENABLE_SPAM_ASSASSIN=False` — matching the `tests/test.env` source, confirming the canonical config is the one exercised (relevant to `S11`'s `NOREPLIES` short-circuit and to the `re2`-shim non-impact above).
- **(observed) Migration idempotence.** Re-running the schema recipe a second time is a no-op — the database is already at head:

  ```text
  $ CONFIG=tests/test.env /app/venv/bin/alembic upgrade head       # second invocation
  INFO  [alembic.runtime.migration] Context impl PostgreSQLImpl.
  INFO  [alembic.runtime.migration] Will assume transactional DDL.
  $ echo $?
  0
  $ CONFIG=tests/test.env /app/venv/bin/alembic current
  32f25cbf12f6 (head)
  ```

### Dependency integrity & baseline advisories

- **(observed) Manifests unchanged.** No dependency was added, upgraded, or removed. In the repository working tree, `git diff --stat -- pyproject.toml poetry.lock` produced **no output** (empty), and the file digests are:

  ```text
  $ sha256sum pyproject.toml poetry.lock
  187bc159f3ce1031b556f8d376fef66bae543fe0e2ddffa04c2f5229473e9692  pyproject.toml
  8544d66814881926eccf3f51b4fb5892112f4f33218688b9eaab1a294944b909  poetry.lock
  ```
- **(observed / inferred) Baseline advisories (pre-existing, not path-exploitable here).** The pinned versions carry known public advisories that are **pre-existing in the frozen dependency set** and are **not** introduced or exercised by this read-only investigation: `aiosmtpd 1.4.2` → **CVE-2024-27305** (SMTP smuggling; fixed 1.4.5) and **CVE-2024-34083** (STARTTLS command injection / MiTM; fixed 1.4.6); `Flask 1.1.2` → **CVE-2023-30861** (response caching of a session cookie). **(inferred)** None is reachable in this trace: the reply path is driven **in-process via `handle_DATA`** with no live SMTP listener, no `STARTTLS`, and no HTTP/session layer — the network-egress proof below confirms no SMTP port and no non-loopback socket is ever opened. They are reported for completeness, not because the investigation triggers them.

### Network no-egress proof (real email never leaves)

A dedicated probe installed a `sys.addaudithook` capturing every `socket.connect` and `socket.getaddrinfo` for the **whole** process (import + fixtures + a real reply drive) with outbound capture enabled. The complete tail:

```text
[EGRESS] reply drive status='250 Message accepted for delivery' (store_emails ON, NOT_SEND_EMAIL)
[EGRESS] CUMULATIVE socket.connect endpoints for the WHOLE process (import + fixtures + drive) (host,port -> count):
[EGRESS]   CUM ::1:6379 x1
[EGRESS] socket.connect endpoints DURING the drive delta = []
[EGRESS] socket.getaddrinfo (DNS) DURING drive = []
[EGRESS] NON-LOOPBACK endpoints (whole process) = [] (count=0)
[EGRESS] SMTP-PORT endpoints (whole process)    = [] (count=0)
[EGRESS] PROVEN: the ONLY socket connections in the whole run are loopback (Postgres/Redis); no SMTP port opened; no non-loopback egress.
```

- **(observed)** The only Python-visible socket connection in the entire process was loopback Redis (`::1:6379`); zero non-loopback endpoints and zero SMTP-port (25/465/587/2525) endpoints were opened. **(inferred)** The PostgreSQL connection is made by `libpq` at the C layer (below the Python socket audit hook) but is likewise loopback by construction (`DB_URI=…@localhost:15432/…`). Combined with `NOT_SEND_EMAIL=true` and the `store_emails_instead_of_sending` capture seam, no real email left the environment.

### Conceptual reference (background only)

- **(background, not a behavioral claim)** SimpleLogin's official documentation describes the reverse-alias reply model this investigation traced in code: replying to a forwarded email actually sends to the **reverse alias**, and SimpleLogin relays the message **from the alias** while the real mailbox stays hidden; a reverse alias is created per `(alias, contact)` pair — see `https://simplelogin.io/docs/getting-started/reverse-alias/` and `https://simplelogin.io/faq/`. This is used only to frame the model; every behavioral claim in this document traces to a `file:line` citation or observed runtime output, not to the documentation.

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

# 2) Drive the REAL SMTP callback in-process for every scenario (each >=2x, or once per toggled
#    branch for the guard/differential scenarios) and capture the full transcript:
cd /tmp/slrun
CONFIG=tests/test.env GITHUB_ACTIONS_TEST=true PYTEST_ADDOPTS="" \
  /app/venv/bin/python -m pytest tests/blitzy_reply_trace.py -s -p no:cacheprovider -p no:randomly -p no:warnings --no-header -q
#   -> 11 passed in 8.99s   (104 embedded "ASSERT PASS" checks, 0 failures)
```

`tests/blitzy_reply_trace.py` was a temporary observation module that lived **only** inside the run container under `/tmp/slrun/tests/` (a copy of the committed tree exported via `git archive HEAD`); it was never added to the repository and does not exist in the working tree. `/tmp/slrun` is entirely inside the container, so the repository working tree remained git-clean (verified in [§ (k)](#k-cleanup--repository-state-proof)).

Its **complete source is reproduced verbatim below**, so the command above is runnable as-is. **Every numeric value quoted anywhere in this document comes from this single harness run** (its complete transcript follows in [§ Raw scenario output](#raw-scenario-output)); nothing is stitched together from multiple runs. As `S1` (run twice on identical bytes) and `S3b` (run five times on one fixture) demonstrate, the **routing-decision invariants — status, decided `user_id`, outbound `envelope_to`, and rewritten `From` — are stable**. Two classes of value legitimately differ between messages without affecting the routing decision, and are called out wherever they appear: (1) the Postgres-sequence autoincrement IDs (`EmailLog`/`Contact`/`Alias`/`Mailbox`/`User`), which increment monotonically as fixtures are created; and (2) the per-message `Message-ID`, which the reply path regenerates for every message via `replace_original_message_id` → `make_msgid(str(email_log.id), …)` (`email_handler.py:L1202`, `email_handler.py:L1333`) — embedding the autoincrement `EmailLog.id` plus a random 64-bit component. In this run both identical-input `S1` messages serialized to `747` bytes, but the serialized byte length is **not** guaranteed stable in general, because the random `Message-ID` component's digit-length can vary run-to-run.

(The only environment fixes applied inside the container — converting `local_data/dkim.key` to PKCS#1 and shimming `re2`→`re` in `app/spamassassin_utils.py`, both noted in the setup — are confined to the container image and touch neither the reply-pipeline code nor any cited `file:line`.)

---

### Observation harness source (`tests/blitzy_reply_trace.py`)

The temporary module below is reproduced **verbatim**. It defines the eleven scenario functions (`S1`, `S2`, `S3`, `S3b`, `S8`, `S9`, `S10`, `S11`, `RUNTIME-01`, `RUNTIME-02`, `RUNTIME-INFO-01`) and contains **104 explicit `_ck(...)` assertions** (printed as `ASSERT PASS: …`), one for every numeric result quoted in this document, so the values are machine-verified rather than hand-copied.

> **Citation note.** The harness's inline comments and printed `OBSERVED` lines use the informal shorthand `(L363)` for the `Contact.get_by(reply_email=…)` call inside `replace_header_when_reply`; that call's executable statement is precisely at `email_handler.py:L364` (`L363` is the blank line directly above it), which is the line the analytical sections of this document cite. The embedded source and captured transcript below are reproduced **verbatim** and therefore retain the harness's `(L363)` shorthand, so they faithfully reflect exactly what ran.

```python
"""
blitzy_reply_trace.py — TEMPORARY, read-only runtime observation harness for the
SimpleLogin alias reply-handling root-cause analysis (Q1-Q5).

It drives SimpleLogin's *canonical* inbound-SMTP entry point
``MailHandler.handle_DATA()`` (email_handler.py:L2289) -> ``_handle()`` (L2334,
wraps handle() in create_light_app().app_context()) -> routing hub ``handle()``
(L1945). Outbound relay is captured non-intrusively through the production seam
``mail_sender.store_emails_instead_of_sending()`` / ``get_stored_emails()``
(app/mail_sender.py:L102/L108) with ``NOT_SEND_EMAIL=true`` (tests/test.env).

Nothing here is a mock, monkeypatch, or synthetic bypass: every routing
decision, is_reverse_alias classification, Contact.get_by lookup, user/mailbox
selection and EmailLog write is produced by the REAL handler. The module is a
pytest module so it inherits the canonical ``flask_client`` app-context +
DB-session (transaction/rollback) fixture from tests/conftest.py.

Every claimed status / id / mailbox / recipient / header / state is backed by an
explicit ``assert`` (helper ``_ck``) so a divergence in source behavior fails the
run loudly (P5-DOC-06). Status codes are asserted against the real constants
imported from ``app.email.status`` (never string literals).

Run (inside the canonical GHCR image container; module lives only under the
container run dir /tmp/slrun and is NEVER committed):

    cd /tmp/slrun
    CONFIG=tests/test.env GITHUB_ACTIONS_TEST=true PYTEST_ADDOPTS="" \
      /app/venv/bin/python -m pytest tests/blitzy_reply_trace.py \
      -s -p no:cacheprovider -p no:randomly -p no:warnings --no-header -q

Absolute integer IDs (user/alias/contact/mailbox/EmailLog) are assigned by the
Postgres sequences at run time and therefore differ run-to-run; only the
behaviors, status strings, and structural invariants are stable.
"""

import asyncio
import logging
from email.message import EmailMessage

import arrow
from aiosmtpd.smtp import Envelope

import email_handler
from email_handler import MailHandler
from app import config
from app.config import EMAIL_DOMAIN
from app.db import Session
from app.email import headers
from app.email import status
from app.email.status import E200, E201, E206, E214, E402, E501, E502, E503, E504
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
# Assertion helper: print a PASS/FAIL ledger line AND hard-assert the invariant.
# Every behavioral claim in the deliverable is anchored to one of these.
# --------------------------------------------------------------------------- #
def _ck(label, cond, detail=""):
    print(f"ASSERT {'PASS' if cond else 'FAIL'}: {label}"
          + (f" -- {detail}" if detail else ""))
    assert cond, f"{label} :: {detail}"


# --------------------------------------------------------------------------- #
# Logging capture: record every "SL" logger emission as (funcName, lineno, msg).
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

_LOG_FUNCS_OF_INTEREST = {
    "_handle",
    "handle",
    "apply_dmarc_policy_for_reply_phase",
    "handle_reply",
    "handle_unknown_mailbox",
    "replace_header_when_reply",
    "send_no_reply_response",
    "__check",  # nested in get_mailbox_from_mail_from -> "Found an authorized address"
}


def _captured_logs():
    out = []
    for func, lineno, msg in _CAP.records:
        if func in _LOG_FUNCS_OF_INTEREST:
            out.append(f"LOG> {func}:{lineno} {msg}")
    return out


def _print_logs(prefix, lines):
    for ln in lines:
        print(f"{prefix}{ln}")


def _emaillog_dict(el):
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
    """Drive the REAL async SMTP DATA callback with a SINGLE recipient."""
    envelope = Envelope()
    envelope.mail_from = mail_from
    envelope.rcpt_tos = [rcpt_to]
    envelope.original_content = msg.as_bytes()
    _CAP.clear()
    return asyncio.run(MailHandler().handle_DATA(None, None, envelope))


def drive_multi(mail_from, rcpt_tos, msg: EmailMessage) -> str:
    """Drive the REAL async SMTP DATA callback with MULTIPLE recipients
    (used to observe the per-recipient dispatch loop, RUNTIME-01)."""
    envelope = Envelope()
    envelope.mail_from = mail_from
    envelope.rcpt_tos = list(rcpt_tos)
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


def build_reply_msg_to(from_addr, to_header_value, subject, body,
                       message_id=None) -> EmailMessage:
    """Like build_reply_msg but the To header is set INDEPENDENTLY of the
    envelope recipient (used for RUNTIME-01 mixed-To and RUNTIME-02 uppercase
    domain). Optional fixed Message-ID for the dedup probe (RUNTIME-INFO-01)."""
    msg = EmailMessage()
    msg[headers.FROM] = from_addr
    msg[headers.TO] = to_header_value
    msg[headers.SUBJECT] = subject
    if message_id is not None:
        msg[headers.MESSAGE_ID] = message_id
    msg.set_content(body)
    return msg


def _stored():
    return mail_sender.get_stored_emails()


def _find_relay_to(stored, website_email):
    """Return the first stored SendRequest whose envelope_to == website_email."""
    for sr in stored:
        if sr.envelope_to == website_email:
            return sr
    return None


# =========================================================================== #
# S1 — Happy path, correctly-owned, SAME UNCHANGED INPUT x2 (Q1/Q2/Q3/Q4 +
# privacy). Asserts: 250 E200, EmailLog written with is_reply, dual user refs
# EQUAL, relay to contact.website, From rewritten to alias, real mailbox hidden.
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
    _ck("S1 dispatch: is_reverse_alias(reply_email) True",
        is_reverse_alias(reply_email) is True, reply_email)
    _ck("S1 happy fixture: alias.user_id == contact.user_id",
        alias_user_id == contact_user_id,
        f"alias.user_id={alias_user_id} contact.user_id={contact_user_id}")

    distribution = []
    for run in (1, 2):
        mail_sender.purge_stored_emails()
        st = drive(mailbox_email, reply_email,
                   build_reply_msg(mailbox_email, reply_email,
                                   "re: same input", "identical body"))
        logs = _captured_logs()
        el_d = _latest_reply_log_dict(contact_id)
        out = _stored()
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
        _ck(f"S1[run{run}] status == E200", st == E200, f"st={st!r}")
        _ck(f"S1[run{run}] reply EmailLog written & is_reply",
            el_d is not None and el_d["is_reply"] is True, str(el_d))
        _ck(f"S1[run{run}] EmailLog.user_id == contact.user_id == alias.user_id",
            el_d["user_id"] == contact_user_id == alias_user_id,
            f"el.user_id={el_d['user_id']} contact={contact_user_id} alias={alias_user_id}")
        _ck(f"S1[run{run}] relayed to contact.website_email",
            out_to == website, f"out_to={out_to} website={website}")
        _ck(f"S1[run{run}] From rewritten to alias (alias.email present)",
            alias_email in str(msg_from), f"msg.From={msg_from} alias={alias_email}")
        _ck(f"S1[run{run}] PRIVACY: real mailbox NOT present anywhere in message",
            present is False, f"mailbox={mailbox_email}")
        distribution.append((st, el_d["user_id"], out_to, msg_from))

    print(f"[S1] identical-input distribution over 2 runs: {distribution}")
    _ck("S1 routing invariants identical across 2 identical-input runs",
        distribution[0] == distribution[1], str(distribution))
    mail_sender.store_emails_instead_of_sending(False)


# =========================================================================== #
# S2 — Constructed divergence alias.user_id != contact.user_id (Q5a). Delivery
# is authorized against alias.user but the EmailLog is attributed to
# contact.user_id. Asserts the MISMATCH is real and observed at runtime x2.
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
        website = contact.website_email
        print(f"[RUN {run}] BEFORE alias.id={alias_id} alias.user_id={alias_user_id} "
              f"contact.id={contact_id} contact.user_id={contact_user_id} "
              f"dual-ref-EQUAL={alias_user_id == contact_user_id}")
        _ck(f"S2[run{run}] constructed divergence alias.user_id != contact.user_id",
            alias_user_id != contact_user_id,
            f"alias.user_id={alias_user_id} contact.user_id={contact_user_id}")
        mail_sender.purge_stored_emails()
        st = drive(mailbox_email, reply_email,
                   build_reply_msg(mailbox_email, reply_email,
                                   f"re: divergence s2r{run}", "body"))
        logs = _captured_logs()
        el_d = _latest_reply_log_dict(contact_id)
        out = _stored()
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
        _ck(f"S2[run{run}] status == E200 (authorized via alias.user)", st == E200,
            f"st={st!r}")
        _ck(f"S2[run{run}] EmailLog.user_id == contact.user_id (recorded owner)",
            el_d["user_id"] == contact_user_id,
            f"el.user_id={el_d['user_id']} contact.user_id={contact_user_id}")
        _ck(f"S2[run{run}] EmailLog.user_id != alias.user_id (MISMATCH, Q5a crux)",
            el_d["user_id"] != alias_user_id,
            f"el.user_id={el_d['user_id']} alias.user_id={alias_user_id}")
        _ck(f"S2[run{run}] relayed to contact.website_email", out_to == website,
            f"out_to={out_to} website={website}")
    mail_sender.store_emails_instead_of_sending(False)


# =========================================================================== #
# S3 — Non-unique reply_email driven through handle_DATA (Q5b), BOTH insertion
# orders. Two Contact rows (two users, two aliases) share ONE reply_email.
# =========================================================================== #
def _s3_one_order(order_label, first_ext_tag, second_ext_tag):
    # reverse-alias local part MUST be lower-case (real reverse-aliases are
    # generated lower-case and Contact.reply_email is matched case-sensitively).
    tag = order_label.lower()
    shared_reply = f"ra+dup_{tag}_{random_string(8)}@{EMAIL_DOMAIN}"
    u1 = create_new_user()
    a1 = Alias.create_new_random(u1)
    Session.flush()
    c1 = Contact.create(
        user_id=u1.id, alias_id=a1.id,
        website_email=f"ext-{first_ext_tag.lower()}-dup_{tag}_{random_string(6)}@external.example",
        reply_email=shared_reply, flush=True,
    )
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
    a1_mb_email = a1.mailbox.email
    a2_mb_email = a2.mailbox.email

    print(f"[order={order_label}] two Contact rows share reply_email={shared_reply} "
          f"(DB accepted both -> reply_email NOT unique):")
    print(f"   contact.id={c1_id} user_id={c1_user} alias_id={c1_alias} "
          f"website_email={c1_web}")
    print(f"   contact.id={c2_id} user_id={c2_user} alias_id={c2_alias} "
          f"website_email={c2_web}")
    db_count = Contact.filter_by(reply_email=shared_reply).count()
    print(f"[order={order_label}] DB row count for that reply_email = {db_count}")
    _ck(f"S3[{order_label}] reply_email NOT unique: 2 Contact rows accepted",
        db_count == 2, f"db_count={db_count}")

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
    _ck(f"S3[{order_label}] get_by(reply_email) selected one of the 2 rows",
        sel_id in (c1_id, c2_id), f"sel_id={sel_id} ids=({c1_id},{c2_id})")

    selected_mb = a1_mb_email if sel_id == c1_id else a2_mb_email
    shadowed_mb = a2_mb_email if sel_id == c1_id else a1_mb_email

    mail_sender.purge_stored_emails()
    st_sel = drive(selected_mb, shared_reply,
                   build_reply_msg(selected_mb, shared_reply,
                                   f"re: dup {order_label} selected", "body"))
    logs_sel = _captured_logs()
    el_sel = _latest_reply_log_dict(sel_id)
    out_sel = _stored()
    out_to_sel = out_sel[0].envelope_to if out_sel else None
    from_sel = out_sel[0].msg[headers.FROM] if out_sel else None
    print(f"[order={order_label}] reply FROM selected owner mailbox={selected_mb} "
          f"-> status={st_sel!r}")
    print(f"[order={order_label}]   EmailLog={el_sel}")
    print(f"[order={order_label}]   OUT envelope_to={out_to_sel} msg.From={from_sel} "
          f"(relayed to SELECTED contact website={sel_web}, user_id={sel_user})")
    _print_logs(f"[order={order_label}]   ", logs_sel)
    _ck(f"S3[{order_label}] selected owner reply status == E200", st_sel == E200,
        f"st={st_sel!r}")
    _ck(f"S3[{order_label}] EmailLog.contact_id == SELECTED contact",
        el_sel is not None and el_sel["contact_id"] == sel_id,
        f"el={el_sel} sel_id={sel_id}")
    _ck(f"S3[{order_label}] relayed to SELECTED contact website",
        out_to_sel == sel_web, f"out_to={out_to_sel} sel_web={sel_web}")

    baseline = _max_email_log_id()
    mail_sender.purge_stored_emails()
    st_shadow = drive(shadowed_mb, shared_reply,
                      build_reply_msg(shadowed_mb, shared_reply,
                                      f"re: dup {order_label} shadowed", "body"))
    logs_shadow = _captured_logs()
    new_max = _max_email_log_id()
    shadow_web = c2_web if sel_id == c1_id else c1_web
    shadow_relay = _find_relay_to(_stored(), shadow_web)
    print(f"[order={order_label}] reply FROM shadowed owner mailbox={shadowed_mb} "
          f"-> status={st_shadow!r} (the shadowed owner's identical reply_email "
          f"resolves to the SELECTED contact.id={sel_id} on a DIFFERENT user's "
          f"alias, so the shadowed mailbox is UNAUTHORIZED there; "
          f"new_reply_EmailLog_created={new_max > baseline})")
    _print_logs(f"[order={order_label}]   ", logs_shadow)
    _ck(f"S3[{order_label}] shadowed owner reply -> E214 (unauthorized on the "
        f"resolved SELECTED alias)", st_shadow == E214, f"st={st_shadow!r}")
    _ck(f"S3[{order_label}] shadowed owner NOT delivered to their own contact",
        shadow_relay is None, f"shadow_web={shadow_web}")
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
# S3b — SAME-FIXTURE repeats (P5-DOC-02). ONE shared reply_email fixture; drive
# the SELECTED owner's reply x5 and the SHADOWED owner's reply x5. Asserts the
# selection is STABLE (same selected contact every time) and the observed
# routing/privacy impact is deterministic on the unchanged fixture.
# =========================================================================== #
def test_s3b_same_fixture_repeats(flask_client):
    mail_sender.store_emails_instead_of_sending(True)
    print("==================== S3b: NON-UNIQUE reply_email — SAME FIXTURE, "
          "REPEATED DRIVES x5 (P5-DOC-02) ====================")
    shared_reply = f"ra+dupfix_{random_string(8)}@{EMAIL_DOMAIN}"
    u1 = create_new_user()
    a1 = Alias.create_new_random(u1)
    Session.flush()
    c1 = Contact.create(
        user_id=u1.id, alias_id=a1.id,
        website_email=f"ext-fixA_{random_string(6)}@external.example",
        reply_email=shared_reply, flush=True,
    )
    u2 = create_new_user()
    a2 = Alias.create_new_random(u2)
    Session.flush()
    c2 = Contact.create(
        user_id=u2.id, alias_id=a2.id,
        website_email=f"ext-fixB_{random_string(6)}@external.example",
        reply_email=shared_reply, flush=True,
    )
    Session.commit()
    c1_id, c2_id = c1.id, c2.id
    a1_mb, a2_mb = a1.mailbox.email, a2.mailbox.email
    db_count = Contact.filter_by(reply_email=shared_reply).count()
    selected = Contact.get_by(reply_email=shared_reply)
    sel_id, sel_web = selected.id, selected.website_email
    selected_mb = a1_mb if sel_id == c1_id else a2_mb
    shadowed_mb = a2_mb if sel_id == c1_id else a1_mb
    shadow_web = c2.website_email if sel_id == c1_id else c1.website_email
    print(f"[S3b] shared_reply={shared_reply} db_count={db_count} "
          f"contacts=({c1_id},{c2_id}) SELECTED={sel_id} selected_web={sel_web}")
    _ck("S3b reply_email NOT unique (2 rows)", db_count == 2, f"db_count={db_count}")

    sel_dist = []
    for i in range(5):
        mail_sender.purge_stored_emails()
        st = drive(selected_mb, shared_reply,
                   build_reply_msg(selected_mb, shared_reply,
                                   f"re: dupfix selected {i}", "body"))
        el = _latest_reply_log_dict(sel_id)
        out = _stored()
        out_to = out[0].envelope_to if out else None
        sel_dist.append((st, el["contact_id"] if el else None, out_to))
    print(f"[S3b] SELECTED-owner repeat distribution (status,EmailLog.contact_id,"
          f"out_to) x5 = {sel_dist}")
    _ck("S3b selected-owner selection STABLE across 5 identical drives",
        all(d == (E200, sel_id, sel_web) for d in sel_dist), str(sel_dist))

    shadow_dist = []
    for i in range(5):
        mail_sender.purge_stored_emails()
        st = drive(shadowed_mb, shared_reply,
                   build_reply_msg(shadowed_mb, shared_reply,
                                   f"re: dupfix shadowed {i}", "body"))
        relay_to_shadow = _find_relay_to(_stored(), shadow_web)
        shadow_dist.append((st, relay_to_shadow is None))
    print(f"[S3b] SHADOWED-owner repeat distribution (status,not_delivered_to_own) "
          f"x5 = {shadow_dist}")
    _ck("S3b shadowed-owner rejected E214 & never delivered to own contact (x5)",
        all(d == (E214, True) for d in shadow_dist), str(shadow_dist))
    print("[S3b] OBSERVED privacy/routing impact: on the unchanged fixture the "
          "shared reverse-alias ALWAYS resolves to the SELECTED contact "
          f"(id={sel_id}); the shadowed owner's reply is deterministically "
          "unauthorized (E214) on the other user's alias -> reply mis-routed by "
          "collision, not delivered to the intended contact.")
    mail_sender.store_emails_instead_of_sending(False)


# =========================================================================== #
# S8 — GUARD PATHS in handle_reply, each exercised through the CANONICAL entry
# point and asserted against the real status constant (P5-DOC-01).
#   (a) E501  wrong reply domain            email_handler.py:L977-981
#   (b) E502  no Contact for reply_email    email_handler.py:L986-989
#   (c) E502  contact.user soft-deleted     email_handler.py:L990-992
#   (d) E503  alias domain unmanaged        email_handler.py:L1000-1002
#   (e) E504  owning user disabled          email_handler.py:L1007-1009
#   (f) normalize_reply_email control char  email_handler.py:L984
# =========================================================================== #
def test_s8_guard_paths(flask_client):
    mail_sender.store_emails_instead_of_sending(True)
    print("==================== S8: handle_reply GUARD PATHS "
          "(E501/E502x2/E503/E504/normalize) ====================")

    # (a) E501 — reply_email on a non-SL domain (dispatch via Contact match) ---
    u = create_new_user()
    a = Alias.create_new_random(u)
    Session.flush()
    reply_501 = f"ra+e501_{random_string(6)}@notsl.example"
    c = Contact.create(user_id=u.id, alias_id=a.id,
                       website_email=f"ext-e501_{random_string(6)}@external.example",
                       reply_email=reply_501, flush=True)
    Session.commit()
    mb = a.mailbox.email
    cid = c.id
    base = _max_email_log_id()
    print(f"[S8a] reply_email={reply_501} (domain notsl.example NOT EMAIL_DOMAIN, "
          f"not an SLDomain); is_reverse_alias={is_reverse_alias(reply_501)}")
    st = drive(mb, reply_501, build_reply_msg(mb, reply_501, "re: e501", "b"))
    print(f"[S8a] status={st!r} new_EmailLog_created={_max_email_log_id() > base}")
    _print_logs("[S8a] ", _captured_logs())
    _ck("S8a dispatch via is_reverse_alias(Contact match) True",
        is_reverse_alias(reply_501) is True, reply_501)
    _ck("S8a status == E501 (wrong reply domain)", st == E501, f"st={st!r}")
    _ck("S8a NO reply EmailLog created", _latest_reply_log_dict(cid) is None,
        "expected None")

    # (b) E502 — no Contact for a ra+ @EMAIL_DOMAIN address --------------------
    reply_502 = f"ra+nocontact_{random_string(8)}@{EMAIL_DOMAIN}"
    print(f"[S8b] reply_email={reply_502} (NO Contact row); "
          f"is_reverse_alias={is_reverse_alias(reply_502)} (via @EMAIL_DOMAIN+ra+)")
    st = drive(f"someone_{random_string(5)}@wherever.example", reply_502,
               build_reply_msg("someone@wherever.example", reply_502, "re: e502b", "b"))
    print(f"[S8b] status={st!r}")
    _print_logs("[S8b] ", _captured_logs())
    _ck("S8b dispatch via @EMAIL_DOMAIN+ra+ branch True",
        is_reverse_alias(reply_502) is True, reply_502)
    _ck("S8b status == E502 (no Contact)", st == E502, f"st={st!r}")

    # (c) E502 — contact.user soft-deleted (is_active False) ------------------
    u = create_new_user()
    a = Alias.create_new_random(u)
    Session.flush()
    reply_i = f"ra+inact_{random_string(6)}@{EMAIL_DOMAIN}"
    c = Contact.create(user_id=u.id, alias_id=a.id,
                       website_email=f"ext-inact_{random_string(6)}@external.example",
                       reply_email=reply_i, flush=True)
    mb = a.mailbox.email
    active_before = u.is_active()
    u.delete_on = arrow.now().shift(days=30)   # FUTURE -> is_active() False
    Session.flush()
    Session.commit()
    active_after = u.is_active()
    print(f"[S8c] BEFORE user.is_active={active_before} -> set delete_on=FUTURE -> "
          f"AFTER user.is_active={active_after}")
    st = drive(mb, reply_i, build_reply_msg(mb, reply_i, "re: e502c", "b"))
    print(f"[S8c] status={st!r}")
    _print_logs("[S8c] ", _captured_logs())
    _ck("S8c BEFORE is_active True", active_before is True, str(active_before))
    _ck("S8c AFTER is_active False (delete_on FUTURE)", active_after is False,
        str(active_after))
    _ck("S8c status == E502 (soft-deleted user)", st == E502, f"st={st!r}")

    # (d) E503 — alias domain not managed by SimpleLogin ----------------------
    u = create_new_user()
    a = Alias.create_new_random(u)
    Session.flush()
    reply_503 = f"ra+e503_{random_string(6)}@{EMAIL_DOMAIN}"
    c = Contact.create(user_id=u.id, alias_id=a.id,
                       website_email=f"ext-e503_{random_string(6)}@external.example",
                       reply_email=reply_503, flush=True)
    mb = a.mailbox.email
    alias_email_before = a.email
    a.email = f"orphan_{random_string(6)}@unmanaged.example"   # unmanaged domain
    Session.flush()
    Session.commit()
    alias_email_after = a.email
    print(f"[S8d] BEFORE alias.email={alias_email_before} -> AFTER "
          f"alias.email={alias_email_after} (domain unmanaged.example)")
    st = drive(mb, reply_503, build_reply_msg(mb, reply_503, "re: e503", "b"))
    print(f"[S8d] status={st!r}")
    _print_logs("[S8d] ", _captured_logs())
    _ck("S8d status == E503 (alias domain unmanaged)", st == E503, f"st={st!r}")

    # (e) E504 — owning user disabled (can_send_or_receive False) --------------
    u = create_new_user()
    a = Alias.create_new_random(u)
    Session.flush()
    reply_504 = f"ra+e504_{random_string(6)}@{EMAIL_DOMAIN}"
    c = Contact.create(user_id=u.id, alias_id=a.id,
                       website_email=f"ext-e504_{random_string(6)}@external.example",
                       reply_email=reply_504, flush=True)
    mb = a.mailbox.email
    csor_before = u.can_send_or_receive()
    u.disabled = True
    u.delete_on = None      # keep is_active() True so E502 does not fire first
    Session.flush()
    Session.commit()
    csor_after = u.can_send_or_receive()
    active_still = u.is_active()
    print(f"[S8e] BEFORE can_send_or_receive={csor_before} -> set disabled=True,"
          f"delete_on=None -> AFTER can_send_or_receive={csor_after} "
          f"is_active={active_still}")
    st = drive(mb, reply_504, build_reply_msg(mb, reply_504, "re: e504", "b"))
    print(f"[S8e] status={st!r}")
    _print_logs("[S8e] ", _captured_logs())
    _ck("S8e is_active still True (so E502 gate passes)", active_still is True,
        str(active_still))
    _ck("S8e can_send_or_receive False after disable", csor_after is False,
        str(csor_after))
    _ck("S8e status == E504 (account disabled)", st == E504, f"st={st!r}")

    # (f) normalize_reply_email — control char maps to stored reply_email ------
    u = create_new_user()
    a = Alias.create_new_random(u)
    Session.flush()
    token = random_string(6)
    stored_reply = f"ra+norm_{token}@{EMAIL_DOMAIN}"          # underscore (stored)
    sent_reply = f"ra+norm\x01{token}@{EMAIL_DOMAIN}"         # \x01 control char
    c = Contact.create(user_id=u.id, alias_id=a.id,
                       website_email=f"ext-norm_{random_string(6)}@external.example",
                       reply_email=stored_reply, flush=True)
    Session.commit()
    mb = a.mailbox.email
    cid = c.id
    website = c.website_email
    print(f"[S8f] stored reply_email={stored_reply!r} ; envelope rcpt (raw)="
          f"{sent_reply!r} (contains \\x01)")
    print(f"[S8f] is_reverse_alias(raw sent)={is_reverse_alias(sent_reply)} "
          f"(Contact.get_by(raw)=None, matched via @EMAIL_DOMAIN+ra+ branch)")
    mail_sender.purge_stored_emails()
    # envelope rcpt carries the \x01 control char (normalized at handle_reply
    # L984 -> matches the stored contact); the To header carries the already-
    # normalized stored address so the reply-body header rewrite (which does NOT
    # normalize, L363) still matches and a real relay occurs.
    st = drive(mb, sent_reply,
               build_reply_msg_to(mb, stored_reply, "re: norm", "b"))
    el = _latest_reply_log_dict(cid)
    out = _stored()
    out_to = out[0].envelope_to if out else None
    print(f"[S8f] status={st!r} EmailLog={el} OUT envelope_to={out_to}")
    _print_logs("[S8f] ", _captured_logs())
    _ck("S8f status == E200 (control char normalized to stored reply_email)",
        st == E200, f"st={st!r}")
    _ck("S8f reply EmailLog created for the stored contact",
        el is not None and el["contact_id"] == cid, str(el))
    _ck("S8f relayed to the stored contact website", out_to == website,
        f"out_to={out_to} website={website}")
    mail_sender.store_emails_instead_of_sending(False)


# =========================================================================== #
# S9 — CONTROLLED spoofing differential (P5-DOC-03): ONE fixture, ONE stranger
# mail_from; the ONLY variable is alias.disable_email_spoofing_check.
#   flag=False (spoofing check ENABLED, default) -> E214, owner alert, 0 relay
#   flag=True  (spoofing check DISABLED)         -> E200 default-mailbox fallback
# =========================================================================== #
def test_s9_spoofing_controlled(flask_client):
    mail_sender.store_emails_instead_of_sending(True)
    print("==================== S9: CONTROLLED spoofing ON/OFF on ONE input "
          "(P5-DOC-03) ====================")
    user = create_new_user()
    alias = Alias.create_new_random(user)
    Session.flush()
    contact = Contact.create(
        user_id=user.id, alias_id=alias.id,
        website_email=f"ext-s9_{random_string(6)}@external.example",
        reply_email=f"ra+s9_{random_string(6)}@{EMAIL_DOMAIN}",
        flush=True,
    )
    Session.commit()
    reply_email = contact.reply_email
    website = contact.website_email
    contact_id = contact.id
    alias_id = alias.id
    alias_default_mb_id = alias.mailbox_id
    stranger = f"stranger-s9_{random_string(6)}@nowhere.example"
    print(f"[S9] fixture alias.id={alias.id} default_mailbox_id={alias_default_mb_id} "
          f"reply_email={reply_email} SINGLE stranger mail_from={stranger}")

    # ---- ON: spoofing check ENABLED (flag False) ----------------------------
    # re-fetch a LIVE instance (the fixture `alias` expires after each drive's
    # app-context teardown, so attribute writes on it would not persist).
    Alias.get(alias_id).disable_email_spoofing_check = False
    Session.commit()
    mail_sender.purge_stored_emails()
    st_on = drive(stranger, reply_email,
                  build_reply_msg(stranger, reply_email, "re: s9 controlled", "b"))
    logs_on = _captured_logs()
    el_on = _latest_reply_log_dict(contact_id)
    stored_on = _stored()
    relay_on = _find_relay_to(stored_on, website)
    print(f"[S9][spoofing_check=ENABLED flag=False] status={st_on!r} "
          f"reply_EmailLog={el_on} total_stored={len(stored_on)} "
          f"relay_to_contact={'yes' if relay_on else 'no'}")
    _print_logs("[S9][ENABLED] ", logs_on)
    _ck("S9 ENABLED status == E214", st_on == E214, f"st={st_on!r}")
    _ck("S9 ENABLED no reply EmailLog", el_on is None, str(el_on))
    _ck("S9 ENABLED no relay to contact website", relay_on is None, website)
    _ck("S9 ENABLED owner alerted (>=1 stored)", len(stored_on) >= 1,
        f"total_stored={len(stored_on)}")

    # ---- OFF: spoofing check DISABLED (flag True) — SAME stranger, SAME rcpt --
    Alias.get(alias_id).disable_email_spoofing_check = True
    Session.commit()
    mail_sender.purge_stored_emails()
    st_off = drive(stranger, reply_email,
                   build_reply_msg(stranger, reply_email, "re: s9 controlled", "b"))
    logs_off = _captured_logs()
    el_off = _latest_reply_log_dict(contact_id)
    stored_off = _stored()
    relay_off = _find_relay_to(stored_off, website)
    used_default = el_off["mailbox_id"] == alias_default_mb_id if el_off else None
    print(f"[S9][spoofing_check=DISABLED flag=True] status={st_off!r} "
          f"reply_EmailLog={el_off} used_default_mailbox={used_default} "
          f"relay_to_contact={'yes' if relay_off else 'no'}")
    _print_logs("[S9][DISABLED] ", logs_off)
    _ck("S9 DISABLED status == E200", st_off == E200, f"st={st_off!r}")
    _ck("S9 DISABLED reply EmailLog created", el_off is not None, str(el_off))
    _ck("S9 DISABLED used DEFAULT mailbox (mailbox_id == alias.mailbox_id)",
        used_default is True,
        f"el.mailbox_id={el_off['mailbox_id']} default={alias_default_mb_id}")
    _ck("S9 DISABLED relay to contact website", relay_off is not None, website)
    print("[S9] DIFFERENTIAL: identical stranger+rcpt; ONLY the spoofing flag "
          "changed the outcome E214 -> E200(default-mailbox fallback).")
    mail_sender.store_emails_instead_of_sending(False)


# =========================================================================== #
# S10 — MULTI-MAILBOX alias, FULL (P5-DOC-04): default / secondary / authorized-
# address sends (each asserts selected mailbox AND the notify_mailbox to the
# OTHER mailbox), plus an UNAUTHORIZED sender -> E214. All get_stored_emails()
# surfaced. notify_mailbox relays to other_mb.email (email_handler.py:L1289-1292).
# =========================================================================== #
def test_s10_multi_mailbox_full(flask_client):
    mail_sender.store_emails_instead_of_sending(True)
    print("==================== S10: MULTI-MAILBOX FULL (default/secondary/"
          "authorized/UNAUTHORIZED + notify_mailbox) ====================")
    user = create_new_user()
    mb1 = user.default_mailbox
    mb2 = Mailbox.create(user_id=user.id, email=random_email(), verified=True,
                         flush=True)
    alias = Alias.create_new_random(user)
    Session.flush()
    AliasMailbox.create(alias_id=alias.id, mailbox_id=mb2.id, flush=True)
    authorized = f"authorized-s10_{random_string(6)}@delegate.example"
    AuthorizedAddress.create(user_id=user.id, mailbox_id=mb1.id, email=authorized,
                             flush=True)
    contact = Contact.create(
        user_id=user.id, alias_id=alias.id,
        website_email=f"ext-s10_{random_string(6)}@external.example",
        reply_email=f"ra+s10_{random_string(6)}@{EMAIL_DOMAIN}",
        flush=True,
    )
    Session.commit()
    mb1_id, mb1_email = mb1.id, mb1.email
    mb2_id, mb2_email = mb2.id, mb2.email
    contact_id = contact.id
    reply_email = contact.reply_email
    website = contact.website_email
    print(f"[S10] alias.id={alias.id} default_mb1.id={mb1_id} secondary_mb2.id="
          f"{mb2_id} alias.mailboxes={[m.id for m in alias.mailboxes]} "
          f"authorized_address_on_mb1={authorized}")

    def _subcase(tag, mail_from, expect_mb_id, other_mb_email):
        mail_sender.purge_stored_emails()
        st = drive(mail_from, reply_email,
                   build_reply_msg(mail_from, reply_email, f"re: s10 {tag}", "b"))
        logs = _captured_logs()
        el = _latest_reply_log_dict(contact_id)
        stored = _stored()
        relay = _find_relay_to(stored, website)
        notify = _find_relay_to(stored, other_mb_email)
        allto = [s.envelope_to for s in stored]
        print(f"[S10][{tag}] mail_from={mail_from} status={st!r} "
              f"EmailLog.mailbox_id={el['mailbox_id'] if el else None} "
              f"(expect {expect_mb_id}) stored_envelope_to={allto}")
        _print_logs(f"[S10][{tag}] ", logs)
        _ck(f"S10[{tag}] status == E200", st == E200, f"st={st!r}")
        _ck(f"S10[{tag}] selected mailbox_id == {expect_mb_id}",
            el is not None and el["mailbox_id"] == expect_mb_id, str(el))
        _ck(f"S10[{tag}] relay to contact website present", relay is not None,
            website)
        _ck(f"S10[{tag}] notify_mailbox to OTHER mailbox ({other_mb_email}) present",
            notify is not None, f"stored_to={allto}")

    # (a) default mailbox -> mb1 selected, notify mb2
    _subcase("default", mb1_email, mb1_id, mb2_email)
    # (b) secondary mailbox -> mb2 selected, notify mb1
    _subcase("secondary", mb2_email, mb2_id, mb1_email)
    # (c) authorized address mapped to mb1 -> mb1 selected, notify mb2
    _subcase("authorized", authorized, mb1_id, mb2_email)

    # (d) UNAUTHORIZED sender -> E214 (not in mailboxes, not authorized)
    stranger = f"stranger-s10_{random_string(6)}@evil.example"
    mail_sender.purge_stored_emails()
    st_d = drive(stranger, reply_email,
                 build_reply_msg(stranger, reply_email, "re: s10 unauth", "b"))
    logs_d = _captured_logs()
    el_d = _latest_reply_log_dict(contact_id)
    stored_d = _stored()
    relay_d = _find_relay_to(stored_d, website)
    print(f"[S10][unauthorized] mail_from={stranger} status={st_d!r} "
          f"stored_envelope_to={[s.envelope_to for s in stored_d]}")
    _print_logs("[S10][unauthorized] ", logs_d)
    _ck("S10[unauthorized] status == E214", st_d == E214, f"st={st_d!r}")
    _ck("S10[unauthorized] no relay to contact website", relay_d is None, website)
    _ck("S10[unauthorized] owner alerted (>=1 stored)", len(stored_d) >= 1,
        f"total_stored={len(stored_d)}")
    mail_sender.store_emails_instead_of_sending(False)


# =========================================================================== #
# S11 — NOREPLIES short-circuit + bounce mail_from == "<>" (P5-DOC-05), with
# EmailLog deltas + auto-response + stored inventory. Fresh user per NOREPLIES
# run to avoid the send_email_at_most_times rate-limit.
# =========================================================================== #
def test_s11_noreplies_and_bounce(flask_client):
    mail_sender.store_emails_instead_of_sending(True)
    print("==================== S11: NOREPLIES short-circuit + bounce "
          "mail_from=='<>' (P5-DOC-05) ====================")
    print(f"config.NOREPLIES={config.NOREPLIES}")
    for run in (1, 2):
        # ---- NOREPLIES short-circuit (fresh known active mailbox) -----------
        noreply_rcpt = config.NOREPLIES[0]
        user = create_new_user()
        sender_mb = user.default_mailbox.email
        base = _max_email_log_id()
        mail_sender.purge_stored_emails()
        st_nr = drive(sender_mb, noreply_rcpt,
                      build_reply_msg(sender_mb, noreply_rcpt,
                                      f"hello s11r{run}", "body"))
        delta_nr = _max_email_log_id() - base
        stored_nr = _stored()
        subjects = [str(s.msg[headers.SUBJECT]) for s in stored_nr]
        auto = any(s.startswith("Auto:") for s in subjects)
        print(f"[RUN {run}] NOREPLIES rcpt={noreply_rcpt} sender={sender_mb} -> "
              f"status={st_nr!r} EmailLog_delta={delta_nr} "
              f"stored_subjects={subjects}")
        _print_logs(f"[RUN {run}][noreply] ", _captured_logs())
        _ck(f"S11[run{run}] NOREPLIES status == E200", st_nr == E200, f"st={st_nr!r}")
        _ck(f"S11[run{run}] NOREPLIES no EmailLog created (delta==0)",
            delta_nr == 0, f"delta={delta_nr}")
        _ck(f"S11[run{run}] NOREPLIES auto-response 'Auto:' stored",
            auto is True, f"subjects={subjects}")

        # ---- bounce mail_from == "<>" to a reverse alias --------------------
        user2 = create_new_user()
        alias = Alias.create_new_random(user2)
        Session.flush()
        contact = Contact.create(
            user_id=user2.id, alias_id=alias.id,
            website_email=f"ext-s11r{run}_{random_string(6)}@external.example",
            reply_email=f"ra+s11r{run}_{random_string(6)}@{EMAIL_DOMAIN}",
            flush=True,
        )
        Session.commit()
        reply_email = contact.reply_email
        base_b = _max_email_log_id()
        mail_sender.purge_stored_emails()
        st_bounce = drive("<>", reply_email,
                          build_reply_msg("bounce@mta.example", reply_email,
                                          f"auto-reply s11r{run}", "ooo"))
        delta_b = _max_email_log_id() - base_b
        stored_b = _stored()
        print(f"[RUN {run}] BOUNCE mail_from='<>' rcpt=reverse-alias({reply_email}) "
              f"-> status={st_bounce!r} EmailLog_delta={delta_b} "
              f"total_stored={len(stored_b)}")
        _print_logs(f"[RUN {run}][bounce] ", _captured_logs())
        _ck(f"S11[run{run}] BOUNCE status == E206 (out of office)",
            st_bounce == E206, f"st={st_bounce!r}")
        _ck(f"S11[run{run}] BOUNCE no EmailLog created (delta==0)",
            delta_b == 0, f"delta={delta_b}")
        _ck(f"S11[run{run}] BOUNCE nothing relayed (stored empty)",
            len(stored_b) == 0, f"total_stored={len(stored_b)}")
    mail_sender.store_emails_instead_of_sending(False)


# =========================================================================== #
# RUNTIME-01 — Mixed-recipient reply: the To-header rewrite
# (replace_header_when_reply, email_handler.py:L345-383) resolves EVERY reverse
# alias in the header via a GLOBAL Contact.get_by(reply_email) that is NOT scoped
# to the sending alias/user (L363). A reply whose To lists two reverse aliases
# owned by DIFFERENT users therefore rewrites BOTH to their website_emails, so
# user A's outbound reply discloses user B's contact address (cross-user leak).
# =========================================================================== #
def test_runtime01_mixed_to_cross_user_disclosure(flask_client):
    mail_sender.store_emails_instead_of_sending(True)
    print("==================== RUNTIME-01: MIXED-To cross-user disclosure "
          "====================")
    ua = create_new_user()
    aa = Alias.create_new_random(ua)
    Session.flush()
    ca = Contact.create(user_id=ua.id, alias_id=aa.id,
                        website_email=f"extA_{random_string(6)}@aaa.example",
                        reply_email=f"ra+a_{random_string(6)}@{EMAIL_DOMAIN}",
                        flush=True)
    ub = create_new_user()
    ab = Alias.create_new_random(ub)
    Session.flush()
    cb = Contact.create(user_id=ub.id, alias_id=ab.id,
                        website_email=f"extB_{random_string(6)}@bbb.example",
                        reply_email=f"ra+b_{random_string(6)}@{EMAIL_DOMAIN}",
                        flush=True)
    Session.commit()
    a_mbox = aa.mailbox.email          # authorized sender for alias A only
    revA, extA = ca.reply_email, ca.website_email
    revB, extB = cb.reply_email, cb.website_email
    print(f"[RT01] userA alias={aa.email} mailbox={a_mbox} revA={revA} extA={extA}")
    print(f"[RT01] userB alias={ab.email} revB={revB} extB={extB}")
    print(f"[RT01] driving mail_from=A_mailbox envelope rcpt_tos=[revA,revB] "
          f"To header='{revA}, {revB}'")
    mail_sender.purge_stored_emails()
    msg = build_reply_msg_to(a_mbox, f"{revA}, {revB}", "re: mixed", "body")
    st = drive_multi(a_mbox, [revA, revB], msg)
    logs = _captured_logs()
    stored = _stored()
    relayA = _find_relay_to(stored, extA)
    to_hdr_A = str(relayA.msg[headers.TO]) if relayA else None
    allto = [s.envelope_to for s in stored]
    print(f"[RT01] overall status={st!r} stored_envelope_to={allto}")
    print(f"[RT01] A's relay (to extA={extA}) rewritten To header = {to_hdr_A!r}")
    _print_logs("[RT01] ", logs)
    _ck("RT01 overall status == E200 (revA leg succeeds)", st == E200, f"st={st!r}")
    _ck("RT01 A's reply relayed to A's own contact (extA)", relayA is not None, extA)
    _ck("RT01 A's rewritten To CONTAINS extA (own contact)",
        to_hdr_A is not None and extA in to_hdr_A, f"To={to_hdr_A}")
    _ck("RT01 CROSS-USER DISCLOSURE: A's To header CONTAINS user B's extB",
        to_hdr_A is not None and extB in to_hdr_A, f"To={to_hdr_A} extB={extB}")
    print("[RT01] OBSERVED: replace_header_when_reply resolved revB via a global, "
          "un-scoped Contact.get_by (email_handler.py:L363) and injected user B's "
          "external contact address into user A's outbound reply To header.")
    mail_sender.store_emails_instead_of_sending(False)


# =========================================================================== #
# RUNTIME-02 — Uppercase-domain reverse alias in the To header. The envelope
# recipient is the lowercase reverse alias (Contact matches, EmailLog created),
# but the message To header carries the SAME reverse alias with an UPPERCASE
# domain. replace_header_when_reply does a case-sensitive Contact.get_by (L363)
# which misses, raising NonReverseAliasInReplyPhase: the EmailLog is DELETED
# (L1185), the owner is alerted (L1187) and E200 is returned (L1198) — the reply
# is silently NOT relayed despite the 250 status.
# =========================================================================== #
def test_runtime02_uppercase_domain_silent_rejection(flask_client):
    mail_sender.store_emails_instead_of_sending(True)
    print("==================== RUNTIME-02: UPPERCASE-domain To -> silent "
          "non-delivery ====================")
    user = create_new_user()
    alias = Alias.create_new_random(user)
    Session.flush()
    token = random_string(6)
    reply_lower = f"ra+up_{token}@{EMAIL_DOMAIN}"        # stored + envelope rcpt
    reply_upper = f"ra+up_{token}@{EMAIL_DOMAIN.upper()}"  # To header (UPPER domain)
    contact = Contact.create(user_id=user.id, alias_id=alias.id,
                             website_email=f"ext-up_{random_string(6)}@external.example",
                             reply_email=reply_lower, flush=True)
    Session.commit()
    owner_mbox = alias.mailbox.email
    contact_id = contact.id
    website = contact.website_email
    print(f"[RT02] envelope rcpt (lowercase)={reply_lower} ; To header "
          f"(UPPERCASE domain)={reply_upper}")
    mail_sender.purge_stored_emails()
    msg = build_reply_msg_to(owner_mbox, reply_upper, "re: upper", "body")
    st = drive(owner_mbox, reply_lower, msg)
    logs = _captured_logs()
    reply_log_after = _latest_reply_log_dict(contact_id)
    stored = _stored()
    relay = _find_relay_to(stored, website)
    subjects = [str(s.msg[headers.SUBJECT]) for s in stored]
    print(f"[RT02] status={st!r} reply_EmailLog_after={reply_log_after} "
          f"relay_to_contact={'yes' if relay else 'no'} stored_subjects={subjects}")
    _print_logs("[RT02] ", logs)
    _ck("RT02 status == E200 (250 returned to sender)", st == E200, f"st={st!r}")
    _ck("RT02 reply EmailLog was created THEN deleted (now None)",
        reply_log_after is None, str(reply_log_after))
    _ck("RT02 reply NOT relayed to contact website (silent non-delivery)",
        relay is None, website)
    _ck("RT02 owner alerted about non reverse-alias address (>=1 stored)",
        len(stored) >= 1, f"subjects={subjects}")
    print("[RT02] OBSERVED: a reply whose To-header reverse alias differs only by "
          "domain LETTER CASE is accepted at SMTP (250) but silently dropped — "
          "the case-sensitive Contact.get_by in replace_header_when_reply (L363) "
          "misses and the delivery is aborted after the EmailLog is deleted.")
    mail_sender.store_emails_instead_of_sending(False)


# =========================================================================== #
# RUNTIME-INFO-01 — No Message-ID de-duplication. The SAME message (identical
# fixed Message-ID) driven twice yields TWO distinct EmailLog rows and TWO
# relays: the handler delivers at-least-once with no idempotency on Message-ID.
# =========================================================================== #
def test_runtime_info01_no_message_id_dedup(flask_client):
    mail_sender.store_emails_instead_of_sending(True)
    print("==================== RUNTIME-INFO-01: no Message-ID de-dup "
          "====================")
    user = create_new_user()
    alias = Alias.create_new_random(user)
    Session.flush()
    contact = Contact.create(user_id=user.id, alias_id=alias.id,
                             website_email=f"ext-dedup_{random_string(6)}@external.example",
                             reply_email=f"ra+dedup_{random_string(6)}@{EMAIL_DOMAIN}",
                             flush=True)
    Session.commit()
    mbox = alias.mailbox.email
    reply_email = contact.reply_email
    website = contact.website_email
    contact_id = contact.id
    fixed_mid = f"<fixed-{random_string(10)}@mta.example>"
    print(f"[RTI01] fixed Message-ID={fixed_mid} driven x2 (identical bytes)")

    mail_sender.purge_stored_emails()
    st1 = drive(mbox, reply_email,
                build_reply_msg_to(mbox, reply_email, "re: dedup", "body",
                                   message_id=fixed_mid))
    el1 = _latest_reply_log_dict(contact_id)
    relay1 = _find_relay_to(_stored(), website)

    mail_sender.purge_stored_emails()
    st2 = drive(mbox, reply_email,
                build_reply_msg_to(mbox, reply_email, "re: dedup", "body",
                                   message_id=fixed_mid))
    el2 = _latest_reply_log_dict(contact_id)
    relay2 = _find_relay_to(_stored(), website)

    print(f"[RTI01] drive1 status={st1!r} EmailLog.id={el1['id'] if el1 else None} "
          f"relay={'yes' if relay1 else 'no'}")
    print(f"[RTI01] drive2 status={st2!r} EmailLog.id={el2['id'] if el2 else None} "
          f"relay={'yes' if relay2 else 'no'}")
    _ck("RTI01 drive1 == E200", st1 == E200, f"st1={st1!r}")
    _ck("RTI01 drive2 == E200", st2 == E200, f"st2={st2!r}")
    _ck("RTI01 TWO distinct EmailLog rows (no dedup)",
        el1 and el2 and el1["id"] != el2["id"],
        f"id1={el1['id'] if el1 else None} id2={el2['id'] if el2 else None}")
    _ck("RTI01 BOTH drives relayed (at-least-once, no idempotency)",
        relay1 is not None and relay2 is not None, "both relays expected")
    mail_sender.store_emails_instead_of_sending(False)
```

---

### Raw scenario output

The transcript below is from the **single harness run** in the reproduction command above (`11 passed in 8.99s`). To stay both faithful and readable: scenario `S1` is reproduced **completely and byte-for-byte** — from interpreter start (config load, domain seeding) through the end of `S1`, including the full timestamped SimpleLogin log stream and every `ASSERT PASS`. Every **subsequent** scenario block is **filtered** — it retains the `[...]` observation lines, the routing-relevant `[RUN …] LOG> …` lines, and every `ASSERT PASS` line, but omits the repeated raw timestamped SL stream, which the handler re-emits in the same shape as `S1` for each scenario. No shown line is altered.

#### S1 — Happy-path reply, SAME UNCHANGED INPUT ×2 (COMPLETE, UNEDITED — full-fidelity sample)

```text
load config file /tmp/slrun/tests/test.env
>>> URL: http://localhost
WARNING: Use a temp directory for GNUPGHOME /tmp/moigyvgnrjtzqvnxtqxy
Upload files to local dir
>>> init logging <<<
2026-07-14 01:32:12,669 - SL - DEBUG - 768 - "/tmp/slrun/app/utils.py:17" - <module>() -  - load words file: /tmp/slrun/local_data/test_words.txt
>>> pg_trgm can't be dropped, ignore
2026-07-14 01:32:13,905 - SL - DEBUG - 768 - "/tmp/slrun/init_app.py:42" - add_sl_domains() -  - d1.test is already a SL domain
2026-07-14 01:32:13,906 - SL - DEBUG - 768 - "/tmp/slrun/init_app.py:42" - add_sl_domains() -  - d2.test is already a SL domain
2026-07-14 01:32:13,906 - SL - DEBUG - 768 - "/tmp/slrun/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
2026-07-14 01:32:13,907 - SL - DEBUG - 768 - "/tmp/slrun/init_app.py:42" - add_sl_domains() -  - sl.local is already a SL domain
==================== S1: HAPPY PATH — SAME UNCHANGED INPUT x2 ====================
2026-07-14 01:32:14,287 - SL - INFO - 768 - "/tmp/slrun/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-14 01:32:14,304 - SL - DEBUG - 768 - "/tmp/slrun/app/models.py:1459" - generate_random_alias_email() -  - generate email cuppas_workup671@sl.local
2026-07-14 01:32:14,314 - SL - INFO - 768 - "/tmp/slrun/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
FIX user.id=47 alias.id=85 alias.email=cuppas_workup671@sl.local alias.user_id=47 mailbox.email=user_juvpu3izq5@mailbox.test contact.id=35 contact.user_id=47 website=ext-s1_kdpnyi@external.example reply_email=ra+s1_ahnude@sl.local dual-ref-EQUAL=True
is_reverse_alias(reply_email)=True
ASSERT PASS: S1 dispatch: is_reverse_alias(reply_email) True -- ra+s1_ahnude@sl.local
ASSERT PASS: S1 happy fixture: alias.user_id == contact.user_id -- alias.user_id=47 contact.user_id=47
2026-07-14 01:32:14,333 - SL - DEBUG - 768 - "/tmp/slrun/app/log.py:24" - set_message_id() -  - set message_id 882e104d-8f8a-4080-b549-cdcfa4d64239
2026-07-14 01:32:14,333 - SL - DEBUG - 768 - "/tmp/slrun/email_handler.py:2342" - _handle() - 882e104d-8f8a-4080-b549-cdcfa4d64239 - ====>=====>====>====>====>====>====>====>
2026-07-14 01:32:14,333 - SL - INFO - 768 - "/tmp/slrun/email_handler.py:2343" - _handle() - 882e104d-8f8a-4080-b549-cdcfa4d64239 - New message, mail from user_juvpu3izq5@mailbox.test, rctp tos ['ra+s1_ahnude@sl.local'] 
2026-07-14 01:32:14,335 - SL - DEBUG - 768 - "/tmp/slrun/email_handler.py:1963" - handle() - 882e104d-8f8a-4080-b549-cdcfa4d64239 - Cannot parse Postfix queue ID from None None
2026-07-14 01:32:14,337 - SL - DEBUG - 768 - "/tmp/slrun/email_handler.py:1980" - handle() - 882e104d-8f8a-4080-b549-cdcfa4d64239 - ==>> Handle mail_from:user_juvpu3izq5@mailbox.test, rcpt_tos:['ra+s1_ahnude@sl.local'], header_from:user_juvpu3izq5@mailbox.test, header_to:ra+s1_ahnude@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'user_juvpu3izq5@mailbox.test'), ('To', 'ra+s1_ahnude@sl.local'), ('Subject', 're: same input'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
2026-07-14 01:32:14,341 - SL - DEBUG - 768 - "/tmp/slrun/email_handler.py:2196" - handle() - 882e104d-8f8a-4080-b549-cdcfa4d64239 - Reply phase user_juvpu3izq5@mailbox.test(user_juvpu3izq5@mailbox.test) -> ra+s1_ahnude@sl.local
2026-07-14 01:32:14,343 - SL - INFO - 768 - "/tmp/slrun/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() - 882e104d-8f8a-4080-b549-cdcfa4d64239 - DMARC check disabled
2026-07-14 01:32:14,347 - SL - DEBUG - 768 - "/tmp/slrun/email_handler.py:1051" - handle_reply() - 882e104d-8f8a-4080-b549-cdcfa4d64239 - Create <EmailLog 27> for <Contact 35 ext-s1_kdpnyi@external.example 85>, <User 47 Test User user_juvpu3izq5@mailbox.test>, <Mailbox 48 user_juvpu3izq5@mailbox.test>
2026-07-14 01:32:14,431 - SL - DEBUG - 768 - "/tmp/slrun/email_handler.py:1171" - handle_reply() - 882e104d-8f8a-4080-b549-cdcfa4d64239 - From header is cuppas_workup671@sl.local
2026-07-14 01:32:14,433 - SL - DEBUG - 768 - "/tmp/slrun/email_handler.py:380" - replace_header_when_reply() - 882e104d-8f8a-4080-b549-cdcfa4d64239 - Replace To header, old: ra+s1_ahnude@sl.local, new: ext-s1_kdpnyi@external.example
2026-07-14 01:32:14,433 - SL - DEBUG - 768 - "/tmp/slrun/email_handler.py:383" - replace_header_when_reply() - 882e104d-8f8a-4080-b549-cdcfa4d64239 - delete the Cc header. Old value None
2026-07-14 01:32:14,434 - SL - DEBUG - 768 - "/tmp/slrun/email_handler.py:1336" - replace_original_message_id() - 882e104d-8f8a-4080-b549-cdcfa4d64239 - no original_message_id, create a new sl_message_id <178399273443.768.3810980954112942338.27@sl.local>
2026-07-14 01:32:14,436 - SL - WARNING - 768 - "/tmp/slrun/email_handler.py:1206" - handle_reply() - 882e104d-8f8a-4080-b549-cdcfa4d64239 - missing date header, add one
2026-07-14 01:32:14,439 - SL - DEBUG - 768 - "/tmp/slrun/email_handler.py:1212" - handle_reply() - 882e104d-8f8a-4080-b549-cdcfa4d64239 - send email from cuppas_workup671@sl.local to ext-s1_kdpnyi@external.example, mail_options:[],rcpt_options:[]
2026-07-14 01:32:14,444 - SL - DEBUG - 768 - "/tmp/slrun/app/mail_sender.py:131" - send() - 882e104d-8f8a-4080-b549-cdcfa4d64239 - send email with subject 're: same input', from 'cuppas_workup671@sl.local' to 'ext-s1_kdpnyi@external.example'
2026-07-14 01:32:14,446 - SL - INFO - 768 - "/tmp/slrun/email_handler.py:2367" - _handle() - 882e104d-8f8a-4080-b549-cdcfa4d64239 - Finish mail_from user_juvpu3izq5@mailbox.test, rcpt_tos ['ra+s1_ahnude@sl.local'], takes 0.11238288879394531 seconds with return code '250 Message accepted for delivery'<<===
[RUN 1] status='250 Message accepted for delivery'
[RUN 1] EmailLog={'id': 27, 'user_id': 47, 'mailbox_id': 48, 'alias_id': 85, 'contact_id': 35, 'is_reply': True}
[RUN 1] OUT envelope_from=sl.lmysyibsg4wcamrthazteojslu.mqv52bnlgl7aa@sl.local envelope_to=ext-s1_kdpnyi@external.example msg.From=cuppas_workup671@sl.local msg.To=ext-s1_kdpnyi@external.example
[RUN 1] LOG> _handle:2342 ====>=====>====>====>====>====>====>====>
[RUN 1] LOG> _handle:2343 New message, mail from user_juvpu3izq5@mailbox.test, rctp tos ['ra+s1_ahnude@sl.local'] 
[RUN 1] LOG> handle:1963 Cannot parse Postfix queue ID from None None
[RUN 1] LOG> handle:1980 ==>> Handle mail_from:user_juvpu3izq5@mailbox.test, rcpt_tos:['ra+s1_ahnude@sl.local'], header_from:user_juvpu3izq5@mailbox.test, header_to:ra+s1_ahnude@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'user_juvpu3izq5@mailbox.test'), ('To', 'ra+s1_ahnude@sl.local'), ('Subject', 're: same input'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
[RUN 1] LOG> handle:2196 Reply phase user_juvpu3izq5@mailbox.test(user_juvpu3izq5@mailbox.test) -> ra+s1_ahnude@sl.local
[RUN 1] LOG> apply_dmarc_policy_for_reply_phase:159 DMARC check disabled
[RUN 1] LOG> handle_reply:1051 Create <EmailLog 27> for <Contact 35 ext-s1_kdpnyi@external.example 85>, <User 47 Test User user_juvpu3izq5@mailbox.test>, <Mailbox 48 user_juvpu3izq5@mailbox.test>
[RUN 1] LOG> handle_reply:1171 From header is cuppas_workup671@sl.local
[RUN 1] LOG> replace_header_when_reply:380 Replace To header, old: ra+s1_ahnude@sl.local, new: ext-s1_kdpnyi@external.example
[RUN 1] LOG> replace_header_when_reply:383 delete the Cc header. Old value None
[RUN 1] LOG> handle_reply:1206 missing date header, add one
[RUN 1] LOG> handle_reply:1212 send email from cuppas_workup671@sl.local to ext-s1_kdpnyi@external.example, mail_options:[],rcpt_options:[]
[RUN 1] LOG> _handle:2367 Finish mail_from user_juvpu3izq5@mailbox.test, rcpt_tos ['ra+s1_ahnude@sl.local'], takes 0.11238288879394531 seconds with return code '250 Message accepted for delivery'<<===
[RUN 1] FULL-MSG-BYTES len=747 real_mailbox(user_juvpu3izq5@mailbox.test)_present_anywhere=False
[RUN 1] OUT all header names=['Subject', 'Content-Type', 'Content-Transfer-Encoding', 'MIME-Version', 'From', 'To', 'Message-ID', 'Date', 'X-SimpleLogin-Type', 'X-SimpleLogin-EmailLog-ID', 'DKIM-Signature']
ASSERT PASS: S1[run1] status == E200 -- st='250 Message accepted for delivery'
ASSERT PASS: S1[run1] reply EmailLog written & is_reply -- {'id': 27, 'user_id': 47, 'mailbox_id': 48, 'alias_id': 85, 'contact_id': 35, 'is_reply': True}
ASSERT PASS: S1[run1] EmailLog.user_id == contact.user_id == alias.user_id -- el.user_id=47 contact=47 alias=47
ASSERT PASS: S1[run1] relayed to contact.website_email -- out_to=ext-s1_kdpnyi@external.example website=ext-s1_kdpnyi@external.example
ASSERT PASS: S1[run1] From rewritten to alias (alias.email present) -- msg.From=cuppas_workup671@sl.local alias=cuppas_workup671@sl.local
ASSERT PASS: S1[run1] PRIVACY: real mailbox NOT present anywhere in message -- mailbox=user_juvpu3izq5@mailbox.test
2026-07-14 01:32:14,452 - SL - DEBUG - 768 - "/tmp/slrun/app/log.py:24" - set_message_id() - 882e104d-8f8a-4080-b549-cdcfa4d64239 - set message_id 9ed60711-9389-4ec9-88e1-510e9d69feb1
2026-07-14 01:32:14,452 - SL - DEBUG - 768 - "/tmp/slrun/email_handler.py:2342" - _handle() - 9ed60711-9389-4ec9-88e1-510e9d69feb1 - ====>=====>====>====>====>====>====>====>
2026-07-14 01:32:14,452 - SL - INFO - 768 - "/tmp/slrun/email_handler.py:2343" - _handle() - 9ed60711-9389-4ec9-88e1-510e9d69feb1 - New message, mail from user_juvpu3izq5@mailbox.test, rctp tos ['ra+s1_ahnude@sl.local'] 
2026-07-14 01:32:14,453 - SL - DEBUG - 768 - "/tmp/slrun/email_handler.py:1963" - handle() - 9ed60711-9389-4ec9-88e1-510e9d69feb1 - Cannot parse Postfix queue ID from None None
2026-07-14 01:32:14,455 - SL - DEBUG - 768 - "/tmp/slrun/email_handler.py:1980" - handle() - 9ed60711-9389-4ec9-88e1-510e9d69feb1 - ==>> Handle mail_from:user_juvpu3izq5@mailbox.test, rcpt_tos:['ra+s1_ahnude@sl.local'], header_from:user_juvpu3izq5@mailbox.test, header_to:ra+s1_ahnude@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'user_juvpu3izq5@mailbox.test'), ('To', 'ra+s1_ahnude@sl.local'), ('Subject', 're: same input'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
2026-07-14 01:32:14,460 - SL - DEBUG - 768 - "/tmp/slrun/email_handler.py:2196" - handle() - 9ed60711-9389-4ec9-88e1-510e9d69feb1 - Reply phase user_juvpu3izq5@mailbox.test(user_juvpu3izq5@mailbox.test) -> ra+s1_ahnude@sl.local
2026-07-14 01:32:14,469 - SL - INFO - 768 - "/tmp/slrun/app/handler/dmarc.py:159" - apply_dmarc_policy_for_reply_phase() - 9ed60711-9389-4ec9-88e1-510e9d69feb1 - DMARC check disabled
2026-07-14 01:32:14,472 - SL - DEBUG - 768 - "/tmp/slrun/email_handler.py:1051" - handle_reply() - 9ed60711-9389-4ec9-88e1-510e9d69feb1 - Create <EmailLog 28> for <Contact 35 ext-s1_kdpnyi@external.example 85>, <User 47 Test User user_juvpu3izq5@mailbox.test>, <Mailbox 48 user_juvpu3izq5@mailbox.test>
2026-07-14 01:32:14,478 - SL - DEBUG - 768 - "/tmp/slrun/email_handler.py:1171" - handle_reply() - 9ed60711-9389-4ec9-88e1-510e9d69feb1 - From header is cuppas_workup671@sl.local
2026-07-14 01:32:14,479 - SL - DEBUG - 768 - "/tmp/slrun/email_handler.py:380" - replace_header_when_reply() - 9ed60711-9389-4ec9-88e1-510e9d69feb1 - Replace To header, old: ra+s1_ahnude@sl.local, new: ext-s1_kdpnyi@external.example
2026-07-14 01:32:14,479 - SL - DEBUG - 768 - "/tmp/slrun/email_handler.py:383" - replace_header_when_reply() - 9ed60711-9389-4ec9-88e1-510e9d69feb1 - delete the Cc header. Old value None
2026-07-14 01:32:14,481 - SL - DEBUG - 768 - "/tmp/slrun/email_handler.py:1336" - replace_original_message_id() - 9ed60711-9389-4ec9-88e1-510e9d69feb1 - no original_message_id, create a new sl_message_id <178399273448.768.2098206441123118155.28@sl.local>
2026-07-14 01:32:14,482 - SL - WARNING - 768 - "/tmp/slrun/email_handler.py:1206" - handle_reply() - 9ed60711-9389-4ec9-88e1-510e9d69feb1 - missing date header, add one
2026-07-14 01:32:14,486 - SL - DEBUG - 768 - "/tmp/slrun/email_handler.py:1212" - handle_reply() - 9ed60711-9389-4ec9-88e1-510e9d69feb1 - send email from cuppas_workup671@sl.local to ext-s1_kdpnyi@external.example, mail_options:[],rcpt_options:[]
2026-07-14 01:32:14,490 - SL - DEBUG - 768 - "/tmp/slrun/app/mail_sender.py:131" - send() - 9ed60711-9389-4ec9-88e1-510e9d69feb1 - send email with subject 're: same input', from 'cuppas_workup671@sl.local' to 'ext-s1_kdpnyi@external.example'
2026-07-14 01:32:14,492 - SL - INFO - 768 - "/tmp/slrun/email_handler.py:2367" - _handle() - 9ed60711-9389-4ec9-88e1-510e9d69feb1 - Finish mail_from user_juvpu3izq5@mailbox.test, rcpt_tos ['ra+s1_ahnude@sl.local'], takes 0.03950905799865723 seconds with return code '250 Message accepted for delivery'<<===
[RUN 2] status='250 Message accepted for delivery'
[RUN 2] EmailLog={'id': 28, 'user_id': 47, 'mailbox_id': 48, 'alias_id': 85, 'contact_id': 35, 'is_reply': True}
[RUN 2] OUT envelope_from=sl.lmysyibshawcamrthazteojslu.ejjtor6veiyhi@sl.local envelope_to=ext-s1_kdpnyi@external.example msg.From=cuppas_workup671@sl.local msg.To=ext-s1_kdpnyi@external.example
[RUN 2] LOG> _handle:2342 ====>=====>====>====>====>====>====>====>
[RUN 2] LOG> _handle:2343 New message, mail from user_juvpu3izq5@mailbox.test, rctp tos ['ra+s1_ahnude@sl.local'] 
[RUN 2] LOG> handle:1963 Cannot parse Postfix queue ID from None None
[RUN 2] LOG> handle:1980 ==>> Handle mail_from:user_juvpu3izq5@mailbox.test, rcpt_tos:['ra+s1_ahnude@sl.local'], header_from:user_juvpu3izq5@mailbox.test, header_to:ra+s1_ahnude@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'user_juvpu3izq5@mailbox.test'), ('To', 'ra+s1_ahnude@sl.local'), ('Subject', 're: same input'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
[RUN 2] LOG> handle:2196 Reply phase user_juvpu3izq5@mailbox.test(user_juvpu3izq5@mailbox.test) -> ra+s1_ahnude@sl.local
[RUN 2] LOG> apply_dmarc_policy_for_reply_phase:159 DMARC check disabled
[RUN 2] LOG> handle_reply:1051 Create <EmailLog 28> for <Contact 35 ext-s1_kdpnyi@external.example 85>, <User 47 Test User user_juvpu3izq5@mailbox.test>, <Mailbox 48 user_juvpu3izq5@mailbox.test>
[RUN 2] LOG> handle_reply:1171 From header is cuppas_workup671@sl.local
[RUN 2] LOG> replace_header_when_reply:380 Replace To header, old: ra+s1_ahnude@sl.local, new: ext-s1_kdpnyi@external.example
[RUN 2] LOG> replace_header_when_reply:383 delete the Cc header. Old value None
[RUN 2] LOG> handle_reply:1206 missing date header, add one
[RUN 2] LOG> handle_reply:1212 send email from cuppas_workup671@sl.local to ext-s1_kdpnyi@external.example, mail_options:[],rcpt_options:[]
[RUN 2] LOG> _handle:2367 Finish mail_from user_juvpu3izq5@mailbox.test, rcpt_tos ['ra+s1_ahnude@sl.local'], takes 0.03950905799865723 seconds with return code '250 Message accepted for delivery'<<===
[RUN 2] FULL-MSG-BYTES len=747 real_mailbox(user_juvpu3izq5@mailbox.test)_present_anywhere=False
[RUN 2] OUT all header names=['Subject', 'Content-Type', 'Content-Transfer-Encoding', 'MIME-Version', 'From', 'To', 'Message-ID', 'Date', 'X-SimpleLogin-Type', 'X-SimpleLogin-EmailLog-ID', 'DKIM-Signature']
ASSERT PASS: S1[run2] status == E200 -- st='250 Message accepted for delivery'
ASSERT PASS: S1[run2] reply EmailLog written & is_reply -- {'id': 28, 'user_id': 47, 'mailbox_id': 48, 'alias_id': 85, 'contact_id': 35, 'is_reply': True}
ASSERT PASS: S1[run2] EmailLog.user_id == contact.user_id == alias.user_id -- el.user_id=47 contact=47 alias=47
ASSERT PASS: S1[run2] relayed to contact.website_email -- out_to=ext-s1_kdpnyi@external.example website=ext-s1_kdpnyi@external.example
ASSERT PASS: S1[run2] From rewritten to alias (alias.email present) -- msg.From=cuppas_workup671@sl.local alias=cuppas_workup671@sl.local
ASSERT PASS: S1[run2] PRIVACY: real mailbox NOT present anywhere in message -- mailbox=user_juvpu3izq5@mailbox.test
[S1] identical-input distribution over 2 runs: [('250 Message accepted for delivery', 47, 'ext-s1_kdpnyi@external.example', 'cuppas_workup671@sl.local'), ('250 Message accepted for delivery', 47, 'ext-s1_kdpnyi@external.example', 'cuppas_workup671@sl.local')]
ASSERT PASS: S1 routing invariants identical across 2 identical-input runs -- [('250 Message accepted for delivery', 47, 'ext-s1_kdpnyi@external.example', 'cuppas_workup671@sl.local'), ('250 Message accepted for delivery', 47, 'ext-s1_kdpnyi@external.example', 'cuppas_workup671@sl.local')]
```

#### S2 — Constructed divergence `alias.user_id != contact.user_id` (Q5a) ×2 — filtered (observation + `LOG>` + `ASSERT PASS` lines)

```text
[RUN 1] BEFORE alias.id=87 alias.user_id=48 contact.id=36 contact.user_id=49 dual-ref-EQUAL=False
ASSERT PASS: S2[run1] constructed divergence alias.user_id != contact.user_id -- alias.user_id=48 contact.user_id=49
[RUN 1] status='250 Message accepted for delivery'
[RUN 1] EmailLog={'id': 29, 'user_id': 49, 'mailbox_id': 49, 'alias_id': 87, 'contact_id': 36, 'is_reply': True}
[RUN 1] PERSIST: EmailLog.user_id=49 == contact.user_id=49 (recorded owner) ; authorizing alias.user_id=48 ; MISMATCH(recorded_vs_authorizing)=True
[RUN 1] OUT envelope_to=ext-div-s2r1_qcjyrb@external.example msg.From=tubule_closed490@sl.local
[RUN 1] LOG> _handle:2342 ====>=====>====>====>====>====>====>====>
[RUN 1] LOG> _handle:2343 New message, mail from user_965qao8l4w@mailbox.test, rctp tos ['ra+s2r1_cedsek@sl.local'] 
[RUN 1] LOG> handle:1963 Cannot parse Postfix queue ID from None None
[RUN 1] LOG> handle:1980 ==>> Handle mail_from:user_965qao8l4w@mailbox.test, rcpt_tos:['ra+s2r1_cedsek@sl.local'], header_from:user_965qao8l4w@mailbox.test, header_to:ra+s2r1_cedsek@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'user_965qao8l4w@mailbox.test'), ('To', 'ra+s2r1_cedsek@sl.local'), ('Subject', 're: divergence s2r1'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
[RUN 1] LOG> handle:2196 Reply phase user_965qao8l4w@mailbox.test(user_965qao8l4w@mailbox.test) -> ra+s2r1_cedsek@sl.local
[RUN 1] LOG> apply_dmarc_policy_for_reply_phase:159 DMARC check disabled
[RUN 1] LOG> handle_reply:1051 Create <EmailLog 29> for <Contact 36 ext-div-s2r1_qcjyrb@external.example 87>, <User 48 Test User user_965qao8l4w@mailbox.test>, <Mailbox 49 user_965qao8l4w@mailbox.test>
[RUN 1] LOG> handle_reply:1171 From header is tubule_closed490@sl.local
[RUN 1] LOG> replace_header_when_reply:380 Replace To header, old: ra+s2r1_cedsek@sl.local, new: ext-div-s2r1_qcjyrb@external.example
[RUN 1] LOG> replace_header_when_reply:383 delete the Cc header. Old value None
[RUN 1] LOG> handle_reply:1206 missing date header, add one
[RUN 1] LOG> handle_reply:1212 send email from tubule_closed490@sl.local to ext-div-s2r1_qcjyrb@external.example, mail_options:[],rcpt_options:[]
[RUN 1] LOG> _handle:2367 Finish mail_from user_965qao8l4w@mailbox.test, rcpt_tos ['ra+s2r1_cedsek@sl.local'], takes 0.025372028350830078 seconds with return code '250 Message accepted for delivery'<<===
ASSERT PASS: S2[run1] status == E200 (authorized via alias.user) -- st='250 Message accepted for delivery'
ASSERT PASS: S2[run1] EmailLog.user_id == contact.user_id (recorded owner) -- el.user_id=49 contact.user_id=49
ASSERT PASS: S2[run1] EmailLog.user_id != alias.user_id (MISMATCH, Q5a crux) -- el.user_id=49 alias.user_id=48
ASSERT PASS: S2[run1] relayed to contact.website_email -- out_to=ext-div-s2r1_qcjyrb@external.example website=ext-div-s2r1_qcjyrb@external.example
[RUN 2] BEFORE alias.id=90 alias.user_id=50 contact.id=37 contact.user_id=51 dual-ref-EQUAL=False
ASSERT PASS: S2[run2] constructed divergence alias.user_id != contact.user_id -- alias.user_id=50 contact.user_id=51
[RUN 2] status='250 Message accepted for delivery'
[RUN 2] EmailLog={'id': 30, 'user_id': 51, 'mailbox_id': 51, 'alias_id': 90, 'contact_id': 37, 'is_reply': True}
[RUN 2] PERSIST: EmailLog.user_id=51 == contact.user_id=51 (recorded owner) ; authorizing alias.user_id=50 ; MISMATCH(recorded_vs_authorizing)=True
[RUN 2] OUT envelope_to=ext-div-s2r2_yrzemn@external.example msg.From=sector_morels418@sl.local
[RUN 2] LOG> _handle:2342 ====>=====>====>====>====>====>====>====>
[RUN 2] LOG> _handle:2343 New message, mail from user_zi55z2xotw@mailbox.test, rctp tos ['ra+s2r2_llthds@sl.local'] 
[RUN 2] LOG> handle:1963 Cannot parse Postfix queue ID from None None
[RUN 2] LOG> handle:1980 ==>> Handle mail_from:user_zi55z2xotw@mailbox.test, rcpt_tos:['ra+s2r2_llthds@sl.local'], header_from:user_zi55z2xotw@mailbox.test, header_to:ra+s2r2_llthds@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'user_zi55z2xotw@mailbox.test'), ('To', 'ra+s2r2_llthds@sl.local'), ('Subject', 're: divergence s2r2'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
[RUN 2] LOG> handle:2196 Reply phase user_zi55z2xotw@mailbox.test(user_zi55z2xotw@mailbox.test) -> ra+s2r2_llthds@sl.local
[RUN 2] LOG> apply_dmarc_policy_for_reply_phase:159 DMARC check disabled
[RUN 2] LOG> handle_reply:1051 Create <EmailLog 30> for <Contact 37 ext-div-s2r2_yrzemn@external.example 90>, <User 50 Test User user_zi55z2xotw@mailbox.test>, <Mailbox 51 user_zi55z2xotw@mailbox.test>
[RUN 2] LOG> handle_reply:1171 From header is sector_morels418@sl.local
[RUN 2] LOG> replace_header_when_reply:380 Replace To header, old: ra+s2r2_llthds@sl.local, new: ext-div-s2r2_yrzemn@external.example
[RUN 2] LOG> replace_header_when_reply:383 delete the Cc header. Old value None
[RUN 2] LOG> handle_reply:1206 missing date header, add one
[RUN 2] LOG> handle_reply:1212 send email from sector_morels418@sl.local to ext-div-s2r2_yrzemn@external.example, mail_options:[],rcpt_options:[]
[RUN 2] LOG> _handle:2367 Finish mail_from user_zi55z2xotw@mailbox.test, rcpt_tos ['ra+s2r2_llthds@sl.local'], takes 0.0353245735168457 seconds with return code '250 Message accepted for delivery'<<===
ASSERT PASS: S2[run2] status == E200 (authorized via alias.user) -- st='250 Message accepted for delivery'
ASSERT PASS: S2[run2] EmailLog.user_id == contact.user_id (recorded owner) -- el.user_id=51 contact.user_id=51
ASSERT PASS: S2[run2] EmailLog.user_id != alias.user_id (MISMATCH, Q5a crux) -- el.user_id=51 alias.user_id=50
ASSERT PASS: S2[run2] relayed to contact.website_email -- out_to=ext-div-s2r2_yrzemn@external.example website=ext-div-s2r2_yrzemn@external.example
```

#### S3 — Non-unique `reply_email` through `handle_DATA` (Q5b) ×2, both insertion orders — filtered (observation + `LOG>` + `ASSERT PASS` lines)

```text
[order=A_then_B] two Contact rows share reply_email=ra+dup_a_then_b_isgulexe@sl.local (DB accepted both -> reply_email NOT unique):
[order=A_then_B] DB row count for that reply_email = 2
ASSERT PASS: S3[A_then_B] reply_email NOT unique: 2 Contact rows accepted -- db_count=2
[order=A_then_B] Contact.get_by(reply_email).first() SELECTED contact.id=38 user_id=52 alias_id=93 website_email=ext-a-dup_a_then_b_xduyem@external.example
[order=A_then_B] selected == first-inserted-contact? True (selected.id=38 first_inserted.id=38)
ASSERT PASS: S3[A_then_B] get_by(reply_email) selected one of the 2 rows -- sel_id=38 ids=(38,39)
[order=A_then_B] reply FROM selected owner mailbox=user_sfswjaf0x3@mailbox.test -> status='250 Message accepted for delivery'
[order=A_then_B]   EmailLog={'id': 31, 'user_id': 52, 'mailbox_id': 53, 'alias_id': 93, 'contact_id': 38, 'is_reply': True}
[order=A_then_B]   OUT envelope_to=ext-a-dup_a_then_b_xduyem@external.example msg.From=porker_cement137@sl.local (relayed to SELECTED contact website=ext-a-dup_a_then_b_xduyem@external.example, user_id=52)
[order=A_then_B]   LOG> _handle:2342 ====>=====>====>====>====>====>====>====>
[order=A_then_B]   LOG> _handle:2343 New message, mail from user_sfswjaf0x3@mailbox.test, rctp tos ['ra+dup_a_then_b_isgulexe@sl.local'] 
[order=A_then_B]   LOG> handle:1963 Cannot parse Postfix queue ID from None None
[order=A_then_B]   LOG> handle:1980 ==>> Handle mail_from:user_sfswjaf0x3@mailbox.test, rcpt_tos:['ra+dup_a_then_b_isgulexe@sl.local'], header_from:user_sfswjaf0x3@mailbox.test, header_to:ra+dup_a_then_b_isgulexe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'user_sfswjaf0x3@mailbox.test'), ('To', 'ra+dup_a_then_b_isgulexe@sl.local'), ('Subject', 're: dup A_then_B selected'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
[order=A_then_B]   LOG> handle:2196 Reply phase user_sfswjaf0x3@mailbox.test(user_sfswjaf0x3@mailbox.test) -> ra+dup_a_then_b_isgulexe@sl.local
[order=A_then_B]   LOG> apply_dmarc_policy_for_reply_phase:159 DMARC check disabled
[order=A_then_B]   LOG> handle_reply:1051 Create <EmailLog 31> for <Contact 38 ext-a-dup_a_then_b_xduyem@external.example 93>, <User 52 Test User user_sfswjaf0x3@mailbox.test>, <Mailbox 53 user_sfswjaf0x3@mailbox.test>
[order=A_then_B]   LOG> handle_reply:1171 From header is porker_cement137@sl.local
[order=A_then_B]   LOG> replace_header_when_reply:380 Replace To header, old: ra+dup_a_then_b_isgulexe@sl.local, new: ext-a-dup_a_then_b_xduyem@external.example
[order=A_then_B]   LOG> replace_header_when_reply:383 delete the Cc header. Old value None
[order=A_then_B]   LOG> handle_reply:1206 missing date header, add one
[order=A_then_B]   LOG> handle_reply:1212 send email from porker_cement137@sl.local to ext-a-dup_a_then_b_xduyem@external.example, mail_options:[],rcpt_options:[]
[order=A_then_B]   LOG> _handle:2367 Finish mail_from user_sfswjaf0x3@mailbox.test, rcpt_tos ['ra+dup_a_then_b_isgulexe@sl.local'], takes 0.02624201774597168 seconds with return code '250 Message accepted for delivery'<<===
ASSERT PASS: S3[A_then_B] selected owner reply status == E200 -- st='250 Message accepted for delivery'
ASSERT PASS: S3[A_then_B] EmailLog.contact_id == SELECTED contact -- el={'id': 31, 'user_id': 52, 'mailbox_id': 53, 'alias_id': 93, 'contact_id': 38, 'is_reply': True} sel_id=38
ASSERT PASS: S3[A_then_B] relayed to SELECTED contact website -- out_to=ext-a-dup_a_then_b_xduyem@external.example sel_web=ext-a-dup_a_then_b_xduyem@external.example
[order=A_then_B] reply FROM shadowed owner mailbox=user_h45ht67qgt@mailbox.test -> status='250 SL E214 Unauthorized for using reverse alias' (the shadowed owner's identical reply_email resolves to the SELECTED contact.id=38 on a DIFFERENT user's alias, so the shadowed mailbox is UNAUTHORIZED there; new_reply_EmailLog_created=False)
[order=A_then_B]   LOG> _handle:2342 ====>=====>====>====>====>====>====>====>
[order=A_then_B]   LOG> _handle:2343 New message, mail from user_h45ht67qgt@mailbox.test, rctp tos ['ra+dup_a_then_b_isgulexe@sl.local'] 
[order=A_then_B]   LOG> handle:1963 Cannot parse Postfix queue ID from None None
[order=A_then_B]   LOG> handle:1980 ==>> Handle mail_from:user_h45ht67qgt@mailbox.test, rcpt_tos:['ra+dup_a_then_b_isgulexe@sl.local'], header_from:user_h45ht67qgt@mailbox.test, header_to:ra+dup_a_then_b_isgulexe@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'user_h45ht67qgt@mailbox.test'), ('To', 'ra+dup_a_then_b_isgulexe@sl.local'), ('Subject', 're: dup A_then_B shadowed'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
[order=A_then_B]   LOG> handle:2196 Reply phase user_h45ht67qgt@mailbox.test(user_h45ht67qgt@mailbox.test) -> ra+dup_a_then_b_isgulexe@sl.local
[order=A_then_B]   LOG> apply_dmarc_policy_for_reply_phase:159 DMARC check disabled
[order=A_then_B]   LOG> handle_unknown_mailbox:1393 Reply email can only be used by mailbox. Actual mail_from: user_h45ht67qgt@mailbox.test. msg from header: user_h45ht67qgt@mailbox.test, reverse-alias ra+dup_a_then_b_isgulexe@sl.local, <Alias 93 porker_cement137@sl.local> <User 52 Test User user_sfswjaf0x3@mailbox.test> <Contact 38 ext-a-dup_a_then_b_xduyem@external.example 93>
[order=A_then_B]   LOG> _handle:2367 Finish mail_from user_h45ht67qgt@mailbox.test, rcpt_tos ['ra+dup_a_then_b_isgulexe@sl.local'], takes 0.03758358955383301 seconds with return code '250 SL E214 Unauthorized for using reverse alias'<<===
ASSERT PASS: S3[A_then_B] shadowed owner reply -> E214 (unauthorized on the resolved SELECTED alias) -- st='250 SL E214 Unauthorized for using reverse alias'
ASSERT PASS: S3[A_then_B] shadowed owner NOT delivered to their own contact -- shadow_web=ext-b-dup_a_then_b_wmncye@external.example
[order=B_then_A] two Contact rows share reply_email=ra+dup_b_then_a_unyszbyf@sl.local (DB accepted both -> reply_email NOT unique):
[order=B_then_A] DB row count for that reply_email = 2
ASSERT PASS: S3[B_then_A] reply_email NOT unique: 2 Contact rows accepted -- db_count=2
[order=B_then_A] Contact.get_by(reply_email).first() SELECTED contact.id=40 user_id=54 alias_id=97 website_email=ext-b-dup_b_then_a_haufjc@external.example
[order=B_then_A] selected == first-inserted-contact? True (selected.id=40 first_inserted.id=40)
ASSERT PASS: S3[B_then_A] get_by(reply_email) selected one of the 2 rows -- sel_id=40 ids=(40,41)
[order=B_then_A] reply FROM selected owner mailbox=user_ubvnr8j4il@mailbox.test -> status='250 Message accepted for delivery'
[order=B_then_A]   EmailLog={'id': 32, 'user_id': 54, 'mailbox_id': 55, 'alias_id': 97, 'contact_id': 40, 'is_reply': True}
[order=B_then_A]   OUT envelope_to=ext-b-dup_b_then_a_haufjc@external.example msg.From=stairs_ogress426@sl.local (relayed to SELECTED contact website=ext-b-dup_b_then_a_haufjc@external.example, user_id=54)
[order=B_then_A]   LOG> _handle:2342 ====>=====>====>====>====>====>====>====>
[order=B_then_A]   LOG> _handle:2343 New message, mail from user_ubvnr8j4il@mailbox.test, rctp tos ['ra+dup_b_then_a_unyszbyf@sl.local'] 
[order=B_then_A]   LOG> handle:1963 Cannot parse Postfix queue ID from None None
[order=B_then_A]   LOG> handle:1980 ==>> Handle mail_from:user_ubvnr8j4il@mailbox.test, rcpt_tos:['ra+dup_b_then_a_unyszbyf@sl.local'], header_from:user_ubvnr8j4il@mailbox.test, header_to:ra+dup_b_then_a_unyszbyf@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'user_ubvnr8j4il@mailbox.test'), ('To', 'ra+dup_b_then_a_unyszbyf@sl.local'), ('Subject', 're: dup B_then_A selected'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
[order=B_then_A]   LOG> handle:2196 Reply phase user_ubvnr8j4il@mailbox.test(user_ubvnr8j4il@mailbox.test) -> ra+dup_b_then_a_unyszbyf@sl.local
[order=B_then_A]   LOG> apply_dmarc_policy_for_reply_phase:159 DMARC check disabled
[order=B_then_A]   LOG> handle_reply:1051 Create <EmailLog 32> for <Contact 40 ext-b-dup_b_then_a_haufjc@external.example 97>, <User 54 Test User user_ubvnr8j4il@mailbox.test>, <Mailbox 55 user_ubvnr8j4il@mailbox.test>
[order=B_then_A]   LOG> handle_reply:1171 From header is stairs_ogress426@sl.local
[order=B_then_A]   LOG> replace_header_when_reply:380 Replace To header, old: ra+dup_b_then_a_unyszbyf@sl.local, new: ext-b-dup_b_then_a_haufjc@external.example
[order=B_then_A]   LOG> replace_header_when_reply:383 delete the Cc header. Old value None
[order=B_then_A]   LOG> handle_reply:1206 missing date header, add one
[order=B_then_A]   LOG> handle_reply:1212 send email from stairs_ogress426@sl.local to ext-b-dup_b_then_a_haufjc@external.example, mail_options:[],rcpt_options:[]
[order=B_then_A]   LOG> _handle:2367 Finish mail_from user_ubvnr8j4il@mailbox.test, rcpt_tos ['ra+dup_b_then_a_unyszbyf@sl.local'], takes 0.04280495643615723 seconds with return code '250 Message accepted for delivery'<<===
ASSERT PASS: S3[B_then_A] selected owner reply status == E200 -- st='250 Message accepted for delivery'
ASSERT PASS: S3[B_then_A] EmailLog.contact_id == SELECTED contact -- el={'id': 32, 'user_id': 54, 'mailbox_id': 55, 'alias_id': 97, 'contact_id': 40, 'is_reply': True} sel_id=40
ASSERT PASS: S3[B_then_A] relayed to SELECTED contact website -- out_to=ext-b-dup_b_then_a_haufjc@external.example sel_web=ext-b-dup_b_then_a_haufjc@external.example
[order=B_then_A] reply FROM shadowed owner mailbox=user_ldaj8qncg8@mailbox.test -> status='250 SL E214 Unauthorized for using reverse alias' (the shadowed owner's identical reply_email resolves to the SELECTED contact.id=40 on a DIFFERENT user's alias, so the shadowed mailbox is UNAUTHORIZED there; new_reply_EmailLog_created=False)
[order=B_then_A]   LOG> _handle:2342 ====>=====>====>====>====>====>====>====>
[order=B_then_A]   LOG> _handle:2343 New message, mail from user_ldaj8qncg8@mailbox.test, rctp tos ['ra+dup_b_then_a_unyszbyf@sl.local'] 
[order=B_then_A]   LOG> handle:1963 Cannot parse Postfix queue ID from None None
[order=B_then_A]   LOG> handle:1980 ==>> Handle mail_from:user_ldaj8qncg8@mailbox.test, rcpt_tos:['ra+dup_b_then_a_unyszbyf@sl.local'], header_from:user_ldaj8qncg8@mailbox.test, header_to:ra+dup_b_then_a_unyszbyf@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'user_ldaj8qncg8@mailbox.test'), ('To', 'ra+dup_b_then_a_unyszbyf@sl.local'), ('Subject', 're: dup B_then_A shadowed'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
[order=B_then_A]   LOG> handle:2196 Reply phase user_ldaj8qncg8@mailbox.test(user_ldaj8qncg8@mailbox.test) -> ra+dup_b_then_a_unyszbyf@sl.local
[order=B_then_A]   LOG> apply_dmarc_policy_for_reply_phase:159 DMARC check disabled
[order=B_then_A]   LOG> handle_unknown_mailbox:1393 Reply email can only be used by mailbox. Actual mail_from: user_ldaj8qncg8@mailbox.test. msg from header: user_ldaj8qncg8@mailbox.test, reverse-alias ra+dup_b_then_a_unyszbyf@sl.local, <Alias 97 stairs_ogress426@sl.local> <User 54 Test User user_ubvnr8j4il@mailbox.test> <Contact 40 ext-b-dup_b_then_a_haufjc@external.example 97>
[order=B_then_A]   LOG> _handle:2367 Finish mail_from user_ldaj8qncg8@mailbox.test, rcpt_tos ['ra+dup_b_then_a_unyszbyf@sl.local'], takes 0.044944047927856445 seconds with return code '250 SL E214 Unauthorized for using reverse alias'<<===
ASSERT PASS: S3[B_then_A] shadowed owner reply -> E214 (unauthorized on the resolved SELECTED alias) -- st='250 SL E214 Unauthorized for using reverse alias'
ASSERT PASS: S3[B_then_A] shadowed owner NOT delivered to their own contact -- shadow_web=ext-a-dup_b_then_a_hplugs@external.example
[S3] OBSERVED selection==first-inserted in both runs: True (run1=True run2=True). NOTE: get_by()=filter_by().first() has NO ORDER BY (models.py:L84); this selection was OBSERVED, not guaranteed.
```

#### S3b — Non-unique `reply_email`, SAME unchanged fixture, repeated ×5 — filtered (observation + `LOG>` + `ASSERT PASS` lines)

```text
[S3b] shared_reply=ra+dupfix_weavrgpl@sl.local db_count=2 contacts=(42,43) SELECTED=42 selected_web=ext-fixA_wiskvc@external.example
ASSERT PASS: S3b reply_email NOT unique (2 rows) -- db_count=2
[S3b] SELECTED-owner repeat distribution (status,EmailLog.contact_id,out_to) x5 = [('250 Message accepted for delivery', 42, 'ext-fixA_wiskvc@external.example'), ('250 Message accepted for delivery', 42, 'ext-fixA_wiskvc@external.example'), ('250 Message accepted for delivery', 42, 'ext-fixA_wiskvc@external.example'), ('250 Message accepted for delivery', 42, 'ext-fixA_wiskvc@external.example'), ('250 Message accepted for delivery', 42, 'ext-fixA_wiskvc@external.example')]
ASSERT PASS: S3b selected-owner selection STABLE across 5 identical drives -- [('250 Message accepted for delivery', 42, 'ext-fixA_wiskvc@external.example'), ('250 Message accepted for delivery', 42, 'ext-fixA_wiskvc@external.example'), ('250 Message accepted for delivery', 42, 'ext-fixA_wiskvc@external.example'), ('250 Message accepted for delivery', 42, 'ext-fixA_wiskvc@external.example'), ('250 Message accepted for delivery', 42, 'ext-fixA_wiskvc@external.example')]
[S3b] SHADOWED-owner repeat distribution (status,not_delivered_to_own) x5 = [('250 SL E214 Unauthorized for using reverse alias', True), ('250 SL E214 Unauthorized for using reverse alias', True), ('250 SL E214 Unauthorized for using reverse alias', True), ('250 SL E214 Unauthorized for using reverse alias', True), ('250 SL E214 Unauthorized for using reverse alias', True)]
ASSERT PASS: S3b shadowed-owner rejected E214 & never delivered to own contact (x5) -- [('250 SL E214 Unauthorized for using reverse alias', True), ('250 SL E214 Unauthorized for using reverse alias', True), ('250 SL E214 Unauthorized for using reverse alias', True), ('250 SL E214 Unauthorized for using reverse alias', True), ('250 SL E214 Unauthorized for using reverse alias', True)]
[S3b] OBSERVED privacy/routing impact: on the unchanged fixture the shared reverse-alias ALWAYS resolves to the SELECTED contact (id=42); the shadowed owner's reply is deterministically unauthorized (E214) on the other user's alias -> reply mis-routed by collision, not delivered to the intended contact.
```

#### S8 — `handle_reply` guard paths (E501 / E502×2 / E503 / E504 / normalization) — filtered (observation + `LOG>` + `ASSERT PASS` lines)

```text
[S8a] reply_email=ra+e501_mpvjur@notsl.example (domain notsl.example NOT EMAIL_DOMAIN, not an SLDomain); is_reverse_alias=True
[S8a] status='550 SL E501' new_EmailLog_created=False
[S8a] LOG> _handle:2342 ====>=====>====>====>====>====>====>====>
[S8a] LOG> _handle:2343 New message, mail from user_pnbyv5bl6g@mailbox.test, rctp tos ['ra+e501_mpvjur@notsl.example'] 
[S8a] LOG> handle:1963 Cannot parse Postfix queue ID from None None
[S8a] LOG> handle:1980 ==>> Handle mail_from:user_pnbyv5bl6g@mailbox.test, rcpt_tos:['ra+e501_mpvjur@notsl.example'], header_from:user_pnbyv5bl6g@mailbox.test, header_to:ra+e501_mpvjur@notsl.example, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'user_pnbyv5bl6g@mailbox.test'), ('To', 'ra+e501_mpvjur@notsl.example'), ('Subject', 're: e501'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
[S8a] LOG> handle:2196 Reply phase user_pnbyv5bl6g@mailbox.test(user_pnbyv5bl6g@mailbox.test) -> ra+e501_mpvjur@notsl.example
[S8a] LOG> handle_reply:980 Reply email ra+e501_mpvjur@notsl.example has wrong domain
[S8a] LOG> _handle:2367 Finish mail_from user_pnbyv5bl6g@mailbox.test, rcpt_tos ['ra+e501_mpvjur@notsl.example'], takes 0.007425785064697266 seconds with return code '550 SL E501'<<===
ASSERT PASS: S8a dispatch via is_reverse_alias(Contact match) True -- ra+e501_mpvjur@notsl.example
ASSERT PASS: S8a status == E501 (wrong reply domain) -- st='550 SL E501'
ASSERT PASS: S8a NO reply EmailLog created -- expected None
[S8b] reply_email=ra+nocontact_ckirsngi@sl.local (NO Contact row); is_reverse_alias=True (via @EMAIL_DOMAIN+ra+)
[S8b] status='550 SL E502 Email not exist'
[S8b] LOG> _handle:2342 ====>=====>====>====>====>====>====>====>
[S8b] LOG> _handle:2343 New message, mail from someone_gnlep@wherever.example, rctp tos ['ra+nocontact_ckirsngi@sl.local'] 
[S8b] LOG> handle:1963 Cannot parse Postfix queue ID from None None
[S8b] LOG> handle:1980 ==>> Handle mail_from:someone_gnlep@wherever.example, rcpt_tos:['ra+nocontact_ckirsngi@sl.local'], header_from:someone@wherever.example, header_to:ra+nocontact_ckirsngi@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'someone@wherever.example'), ('To', 'ra+nocontact_ckirsngi@sl.local'), ('Subject', 're: e502b'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
[S8b] LOG> handle:2196 Reply phase someone_gnlep@wherever.example(someone@wherever.example) -> ra+nocontact_ckirsngi@sl.local
[S8b] LOG> handle_reply:988 No contact with ra+nocontact_ckirsngi@sl.local as reverse alias
[S8b] LOG> _handle:2367 Finish mail_from someone_gnlep@wherever.example, rcpt_tos ['ra+nocontact_ckirsngi@sl.local'], takes 0.006922245025634766 seconds with return code '550 SL E502 Email not exist'<<===
ASSERT PASS: S8b dispatch via @EMAIL_DOMAIN+ra+ branch True -- ra+nocontact_ckirsngi@sl.local
ASSERT PASS: S8b status == E502 (no Contact) -- st='550 SL E502 Email not exist'
[S8c] BEFORE user.is_active=True -> set delete_on=FUTURE -> AFTER user.is_active=False
[S8c] status='550 SL E502 Email not exist'
[S8c] LOG> _handle:2342 ====>=====>====>====>====>====>====>====>
[S8c] LOG> _handle:2343 New message, mail from user_lmnnrwtar0@mailbox.test, rctp tos ['ra+inact_wxdayi@sl.local'] 
[S8c] LOG> handle:1963 Cannot parse Postfix queue ID from None None
[S8c] LOG> handle:1980 ==>> Handle mail_from:user_lmnnrwtar0@mailbox.test, rcpt_tos:['ra+inact_wxdayi@sl.local'], header_from:user_lmnnrwtar0@mailbox.test, header_to:ra+inact_wxdayi@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'user_lmnnrwtar0@mailbox.test'), ('To', 'ra+inact_wxdayi@sl.local'), ('Subject', 're: e502c'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
[S8c] LOG> handle:2196 Reply phase user_lmnnrwtar0@mailbox.test(user_lmnnrwtar0@mailbox.test) -> ra+inact_wxdayi@sl.local
[S8c] LOG> handle_reply:991 User <User 59 Test User user_lmnnrwtar0@mailbox.test> has been soft deleted
[S8c] LOG> _handle:2367 Finish mail_from user_lmnnrwtar0@mailbox.test, rcpt_tos ['ra+inact_wxdayi@sl.local'], takes 0.007950067520141602 seconds with return code '550 SL E502 Email not exist'<<===
ASSERT PASS: S8c BEFORE is_active True -- True
ASSERT PASS: S8c AFTER is_active False (delete_on FUTURE) -- False
ASSERT PASS: S8c status == E502 (soft-deleted user) -- st='550 SL E502 Email not exist'
[S8d] BEFORE alias.email=longer_doomed380@sl.local -> AFTER alias.email=orphan_brgezx@unmanaged.example (domain unmanaged.example)
[S8d] status='550 SL E503'
[S8d] LOG> _handle:2342 ====>=====>====>====>====>====>====>====>
[S8d] LOG> _handle:2343 New message, mail from user_q8ex9d5zc3@mailbox.test, rctp tos ['ra+e503_wvgnbb@sl.local'] 
[S8d] LOG> handle:1963 Cannot parse Postfix queue ID from None None
[S8d] LOG> handle:1980 ==>> Handle mail_from:user_q8ex9d5zc3@mailbox.test, rcpt_tos:['ra+e503_wvgnbb@sl.local'], header_from:user_q8ex9d5zc3@mailbox.test, header_to:ra+e503_wvgnbb@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'user_q8ex9d5zc3@mailbox.test'), ('To', 'ra+e503_wvgnbb@sl.local'), ('Subject', 're: e503'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
[S8d] LOG> handle:2196 Reply phase user_q8ex9d5zc3@mailbox.test(user_q8ex9d5zc3@mailbox.test) -> ra+e503_wvgnbb@sl.local
[S8d] LOG> handle_reply:1001 <Alias 109 orphan_brgezx@unmanaged.example> domain isn't known
[S8d] LOG> _handle:2367 Finish mail_from user_q8ex9d5zc3@mailbox.test, rcpt_tos ['ra+e503_wvgnbb@sl.local'], takes 0.015285253524780273 seconds with return code '550 SL E503'<<===
ASSERT PASS: S8d status == E503 (alias domain unmanaged) -- st='550 SL E503'
[S8e] BEFORE can_send_or_receive=True -> set disabled=True,delete_on=None -> AFTER can_send_or_receive=False is_active=True
[S8e] status='550 SL E504 Account disabled'
[S8e] LOG> _handle:2342 ====>=====>====>====>====>====>====>====>
[S8e] LOG> _handle:2343 New message, mail from user_kkzpgh6282@mailbox.test, rctp tos ['ra+e504_fbqyys@sl.local'] 
[S8e] LOG> handle:1963 Cannot parse Postfix queue ID from None None
[S8e] LOG> handle:1980 ==>> Handle mail_from:user_kkzpgh6282@mailbox.test, rcpt_tos:['ra+e504_fbqyys@sl.local'], header_from:user_kkzpgh6282@mailbox.test, header_to:ra+e504_fbqyys@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'user_kkzpgh6282@mailbox.test'), ('To', 'ra+e504_fbqyys@sl.local'), ('Subject', 're: e504'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
[S8e] LOG> handle:2196 Reply phase user_kkzpgh6282@mailbox.test(user_kkzpgh6282@mailbox.test) -> ra+e504_fbqyys@sl.local
[S8e] LOG> handle_reply:1008 User <User 61 Test User user_kkzpgh6282@mailbox.test> cannot send emails
[S8e] LOG> _handle:2367 Finish mail_from user_kkzpgh6282@mailbox.test, rcpt_tos ['ra+e504_fbqyys@sl.local'], takes 0.010132312774658203 seconds with return code '550 SL E504 Account disabled'<<===
ASSERT PASS: S8e is_active still True (so E502 gate passes) -- True
ASSERT PASS: S8e can_send_or_receive False after disable -- False
ASSERT PASS: S8e status == E504 (account disabled) -- st='550 SL E504 Account disabled'
[S8f] stored reply_email='ra+norm_kixgja@sl.local' ; envelope rcpt (raw)='ra+norm\x01kixgja@sl.local' (contains \x01)
[S8f] is_reverse_alias(raw sent)=True (Contact.get_by(raw)=None, matched via @EMAIL_DOMAIN+ra+ branch)
[S8f] status='250 Message accepted for delivery' EmailLog={'id': 38, 'user_id': 62, 'mailbox_id': 63, 'alias_id': 113, 'contact_id': 48, 'is_reply': True} OUT envelope_to=ext-norm_igxgdd@external.example
[S8f] LOG> _handle:2342 ====>=====>====>====>====>====>====>====>
[S8f] LOG> _handle:2343 New message, mail from user_ks89kbkr1t@mailbox.test, rctp tos ['ra+norm\x01kixgja@sl.local'] 
[S8f] LOG> handle:1963 Cannot parse Postfix queue ID from None None
[S8f] LOG> handle:1980 ==>> Handle mail_from:user_ks89kbkr1t@mailbox.test, rcpt_tos:['ra+norm\x01kixgja@sl.local'], header_from:user_ks89kbkr1t@mailbox.test, header_to:ra+norm_kixgja@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'user_ks89kbkr1t@mailbox.test'), ('To', 'ra+norm_kixgja@sl.local'), ('Subject', 're: norm'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
[S8f] LOG> handle:2196 Reply phase user_ks89kbkr1t@mailbox.test(user_ks89kbkr1t@mailbox.test) -> ra+normkixgja@sl.local
[S8f] LOG> apply_dmarc_policy_for_reply_phase:159 DMARC check disabled
[S8f] LOG> handle_reply:1051 Create <EmailLog 38> for <Contact 48 ext-norm_igxgdd@external.example 113>, <User 62 Test User user_ks89kbkr1t@mailbox.test>, <Mailbox 63 user_ks89kbkr1t@mailbox.test>
[S8f] LOG> handle_reply:1171 From header is shoats_piling926@sl.local
[S8f] LOG> replace_header_when_reply:380 Replace To header, old: ra+norm_kixgja@sl.local, new: ext-norm_igxgdd@external.example
[S8f] LOG> replace_header_when_reply:383 delete the Cc header. Old value None
[S8f] LOG> handle_reply:1206 missing date header, add one
[S8f] LOG> handle_reply:1212 send email from shoats_piling926@sl.local to ext-norm_igxgdd@external.example, mail_options:[],rcpt_options:[]
[S8f] LOG> _handle:2367 Finish mail_from user_ks89kbkr1t@mailbox.test, rcpt_tos ['ra+norm\x01kixgja@sl.local'], takes 0.02641129493713379 seconds with return code '250 Message accepted for delivery'<<===
ASSERT PASS: S8f status == E200 (control char normalized to stored reply_email) -- st='250 Message accepted for delivery'
ASSERT PASS: S8f reply EmailLog created for the stored contact -- {'id': 38, 'user_id': 62, 'mailbox_id': 63, 'alias_id': 113, 'contact_id': 48, 'is_reply': True}
ASSERT PASS: S8f relayed to the stored contact website -- out_to=ext-norm_igxgdd@external.example website=ext-norm_igxgdd@external.example
```

#### S9 — Controlled `disable_email_spoofing_check` ON/OFF differential on ONE input (Q5c) — filtered (observation + `LOG>` + `ASSERT PASS` lines)

```text
[S9] fixture alias.id=115 default_mailbox_id=64 reply_email=ra+s9_hhfcum@sl.local SINGLE stranger mail_from=stranger-s9_atvwcp@nowhere.example
[S9][spoofing_check=ENABLED flag=False] status='250 SL E214 Unauthorized for using reverse alias' reply_EmailLog=None total_stored=1 relay_to_contact=no
[S9][ENABLED] LOG> _handle:2342 ====>=====>====>====>====>====>====>====>
[S9][ENABLED] LOG> _handle:2343 New message, mail from stranger-s9_atvwcp@nowhere.example, rctp tos ['ra+s9_hhfcum@sl.local'] 
[S9][ENABLED] LOG> handle:1963 Cannot parse Postfix queue ID from None None
[S9][ENABLED] LOG> handle:1980 ==>> Handle mail_from:stranger-s9_atvwcp@nowhere.example, rcpt_tos:['ra+s9_hhfcum@sl.local'], header_from:stranger-s9_atvwcp@nowhere.example, header_to:ra+s9_hhfcum@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'stranger-s9_atvwcp@nowhere.example'), ('To', 'ra+s9_hhfcum@sl.local'), ('Subject', 're: s9 controlled'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
[S9][ENABLED] LOG> handle:2196 Reply phase stranger-s9_atvwcp@nowhere.example(stranger-s9_atvwcp@nowhere.example) -> ra+s9_hhfcum@sl.local
[S9][ENABLED] LOG> apply_dmarc_policy_for_reply_phase:159 DMARC check disabled
[S9][ENABLED] LOG> handle_unknown_mailbox:1393 Reply email can only be used by mailbox. Actual mail_from: stranger-s9_atvwcp@nowhere.example. msg from header: stranger-s9_atvwcp@nowhere.example, reverse-alias ra+s9_hhfcum@sl.local, <Alias 115 slangy_tuneup098@sl.local> <User 63 Test User user_3u1fsyuo0i@mailbox.test> <Contact 49 ext-s9_dlhpyg@external.example 115>
[S9][ENABLED] LOG> _handle:2367 Finish mail_from stranger-s9_atvwcp@nowhere.example, rcpt_tos ['ra+s9_hhfcum@sl.local'], takes 0.03501105308532715 seconds with return code '250 SL E214 Unauthorized for using reverse alias'<<===
ASSERT PASS: S9 ENABLED status == E214 -- st='250 SL E214 Unauthorized for using reverse alias'
ASSERT PASS: S9 ENABLED no reply EmailLog -- None
ASSERT PASS: S9 ENABLED no relay to contact website -- ext-s9_dlhpyg@external.example
ASSERT PASS: S9 ENABLED owner alerted (>=1 stored) -- total_stored=1
[S9][spoofing_check=DISABLED flag=True] status='250 Message accepted for delivery' reply_EmailLog={'id': 39, 'user_id': 63, 'mailbox_id': 64, 'alias_id': 115, 'contact_id': 49, 'is_reply': True} used_default_mailbox=True relay_to_contact=yes
[S9][DISABLED] LOG> _handle:2342 ====>=====>====>====>====>====>====>====>
[S9][DISABLED] LOG> _handle:2343 New message, mail from stranger-s9_atvwcp@nowhere.example, rctp tos ['ra+s9_hhfcum@sl.local'] 
[S9][DISABLED] LOG> handle:1963 Cannot parse Postfix queue ID from None None
[S9][DISABLED] LOG> handle:1980 ==>> Handle mail_from:stranger-s9_atvwcp@nowhere.example, rcpt_tos:['ra+s9_hhfcum@sl.local'], header_from:stranger-s9_atvwcp@nowhere.example, header_to:ra+s9_hhfcum@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'stranger-s9_atvwcp@nowhere.example'), ('To', 'ra+s9_hhfcum@sl.local'), ('Subject', 're: s9 controlled'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
[S9][DISABLED] LOG> handle:2196 Reply phase stranger-s9_atvwcp@nowhere.example(stranger-s9_atvwcp@nowhere.example) -> ra+s9_hhfcum@sl.local
[S9][DISABLED] LOG> apply_dmarc_policy_for_reply_phase:159 DMARC check disabled
[S9][DISABLED] LOG> handle_reply:1023 ignore unknown sender to reverse-alias stranger-s9_atvwcp@nowhere.example: <Alias 115 slangy_tuneup098@sl.local> -> <Contact 49 ext-s9_dlhpyg@external.example 115>
[S9][DISABLED] LOG> handle_reply:1051 Create <EmailLog 39> for <Contact 49 ext-s9_dlhpyg@external.example 115>, <User 63 Test User user_3u1fsyuo0i@mailbox.test>, <Mailbox 64 user_3u1fsyuo0i@mailbox.test>
[S9][DISABLED] LOG> handle_reply:1171 From header is slangy_tuneup098@sl.local
[S9][DISABLED] LOG> replace_header_when_reply:380 Replace To header, old: ra+s9_hhfcum@sl.local, new: ext-s9_dlhpyg@external.example
[S9][DISABLED] LOG> replace_header_when_reply:383 delete the Cc header. Old value None
[S9][DISABLED] LOG> handle_reply:1206 missing date header, add one
[S9][DISABLED] LOG> handle_reply:1212 send email from slangy_tuneup098@sl.local to ext-s9_dlhpyg@external.example, mail_options:[],rcpt_options:[]
[S9][DISABLED] LOG> _handle:2367 Finish mail_from stranger-s9_atvwcp@nowhere.example, rcpt_tos ['ra+s9_hhfcum@sl.local'], takes 0.047827720642089844 seconds with return code '250 Message accepted for delivery'<<===
ASSERT PASS: S9 DISABLED status == E200 -- st='250 Message accepted for delivery'
ASSERT PASS: S9 DISABLED reply EmailLog created -- {'id': 39, 'user_id': 63, 'mailbox_id': 64, 'alias_id': 115, 'contact_id': 49, 'is_reply': True}
ASSERT PASS: S9 DISABLED used DEFAULT mailbox (mailbox_id == alias.mailbox_id) -- el.mailbox_id=64 default=64
ASSERT PASS: S9 DISABLED relay to contact website -- ext-s9_dlhpyg@external.example
[S9] DIFFERENTIAL: identical stranger+rcpt; ONLY the spoofing flag changed the outcome E214 -> E200(default-mailbox fallback).
```

#### S10 — Multi-mailbox alias: default / secondary / authorized / UNAUTHORIZED + `notify_mailbox` — filtered (observation + `LOG>` + `ASSERT PASS` lines)

```text
[S10] alias.id=117 default_mb1.id=65 secondary_mb2.id=66 alias.mailboxes=[66, 65] authorized_address_on_mb1=authorized-s10_xaczmd@delegate.example
[S10][default] mail_from=user_p9pty8v2qr@mailbox.test status='250 Message accepted for delivery' EmailLog.mailbox_id=65 (expect 65) stored_envelope_to=['ext-s10_uoybzx@external.example', 'moaltkkpasbtwhuuundh@moaltkkpasbtwhuuundh.com']
[S10][default] LOG> _handle:2342 ====>=====>====>====>====>====>====>====>
[S10][default] LOG> _handle:2343 New message, mail from user_p9pty8v2qr@mailbox.test, rctp tos ['ra+s10_zkelpf@sl.local'] 
[S10][default] LOG> handle:1963 Cannot parse Postfix queue ID from None None
[S10][default] LOG> handle:1980 ==>> Handle mail_from:user_p9pty8v2qr@mailbox.test, rcpt_tos:['ra+s10_zkelpf@sl.local'], header_from:user_p9pty8v2qr@mailbox.test, header_to:ra+s10_zkelpf@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'user_p9pty8v2qr@mailbox.test'), ('To', 'ra+s10_zkelpf@sl.local'), ('Subject', 're: s10 default'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
[S10][default] LOG> handle:2196 Reply phase user_p9pty8v2qr@mailbox.test(user_p9pty8v2qr@mailbox.test) -> ra+s10_zkelpf@sl.local
[S10][default] LOG> apply_dmarc_policy_for_reply_phase:159 DMARC check disabled
[S10][default] LOG> handle_reply:1051 Create <EmailLog 40> for <Contact 50 ext-s10_uoybzx@external.example 117>, <User 64 Test User user_p9pty8v2qr@mailbox.test>, <Mailbox 65 user_p9pty8v2qr@mailbox.test>
[S10][default] LOG> handle_reply:1171 From header is affray_browse444@sl.local
[S10][default] LOG> replace_header_when_reply:380 Replace To header, old: ra+s10_zkelpf@sl.local, new: ext-s10_uoybzx@external.example
[S10][default] LOG> replace_header_when_reply:383 delete the Cc header. Old value None
[S10][default] LOG> handle_reply:1206 missing date header, add one
[S10][default] LOG> handle_reply:1212 send email from affray_browse444@sl.local to ext-s10_uoybzx@external.example, mail_options:[],rcpt_options:[]
[S10][default] LOG> _handle:2367 Finish mail_from user_p9pty8v2qr@mailbox.test, rcpt_tos ['ra+s10_zkelpf@sl.local'], takes 0.027179956436157227 seconds with return code '250 Message accepted for delivery'<<===
ASSERT PASS: S10[default] status == E200 -- st='250 Message accepted for delivery'
ASSERT PASS: S10[default] selected mailbox_id == 65 -- {'id': 40, 'user_id': 64, 'mailbox_id': 65, 'alias_id': 117, 'contact_id': 50, 'is_reply': True}
ASSERT PASS: S10[default] relay to contact website present -- ext-s10_uoybzx@external.example
ASSERT PASS: S10[default] notify_mailbox to OTHER mailbox (moaltkkpasbtwhuuundh@moaltkkpasbtwhuuundh.com) present -- stored_to=['ext-s10_uoybzx@external.example', 'moaltkkpasbtwhuuundh@moaltkkpasbtwhuuundh.com']
[S10][secondary] mail_from=moaltkkpasbtwhuuundh@moaltkkpasbtwhuuundh.com status='250 Message accepted for delivery' EmailLog.mailbox_id=66 (expect 66) stored_envelope_to=['ext-s10_uoybzx@external.example', 'user_p9pty8v2qr@mailbox.test']
[S10][secondary] LOG> _handle:2342 ====>=====>====>====>====>====>====>====>
[S10][secondary] LOG> _handle:2343 New message, mail from moaltkkpasbtwhuuundh@moaltkkpasbtwhuuundh.com, rctp tos ['ra+s10_zkelpf@sl.local'] 
[S10][secondary] LOG> handle:1963 Cannot parse Postfix queue ID from None None
[S10][secondary] LOG> handle:1980 ==>> Handle mail_from:moaltkkpasbtwhuuundh@moaltkkpasbtwhuuundh.com, rcpt_tos:['ra+s10_zkelpf@sl.local'], header_from:moaltkkpasbtwhuuundh@moaltkkpasbtwhuuundh.com, header_to:ra+s10_zkelpf@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'moaltkkpasbtwhuuundh@moaltkkpasbtwhuuundh.com'), ('To', 'ra+s10_zkelpf@sl.local'), ('Subject', 're: s10 secondary'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
[S10][secondary] LOG> handle:2196 Reply phase moaltkkpasbtwhuuundh@moaltkkpasbtwhuuundh.com(moaltkkpasbtwhuuundh@moaltkkpasbtwhuuundh.com) -> ra+s10_zkelpf@sl.local
[S10][secondary] LOG> apply_dmarc_policy_for_reply_phase:159 DMARC check disabled
[S10][secondary] LOG> handle_reply:1051 Create <EmailLog 41> for <Contact 50 ext-s10_uoybzx@external.example 117>, <User 64 Test User user_p9pty8v2qr@mailbox.test>, <Mailbox 66 moaltkkpasbtwhuuundh@moaltkkpasbtwhuuundh.com>
[S10][secondary] LOG> handle_reply:1171 From header is affray_browse444@sl.local
[S10][secondary] LOG> replace_header_when_reply:380 Replace To header, old: ra+s10_zkelpf@sl.local, new: ext-s10_uoybzx@external.example
[S10][secondary] LOG> replace_header_when_reply:383 delete the Cc header. Old value None
[S10][secondary] LOG> handle_reply:1206 missing date header, add one
[S10][secondary] LOG> handle_reply:1212 send email from affray_browse444@sl.local to ext-s10_uoybzx@external.example, mail_options:[],rcpt_options:[]
[S10][secondary] LOG> _handle:2367 Finish mail_from moaltkkpasbtwhuuundh@moaltkkpasbtwhuuundh.com, rcpt_tos ['ra+s10_zkelpf@sl.local'], takes 0.02780938148498535 seconds with return code '250 Message accepted for delivery'<<===
ASSERT PASS: S10[secondary] status == E200 -- st='250 Message accepted for delivery'
ASSERT PASS: S10[secondary] selected mailbox_id == 66 -- {'id': 41, 'user_id': 64, 'mailbox_id': 66, 'alias_id': 117, 'contact_id': 50, 'is_reply': True}
ASSERT PASS: S10[secondary] relay to contact website present -- ext-s10_uoybzx@external.example
ASSERT PASS: S10[secondary] notify_mailbox to OTHER mailbox (user_p9pty8v2qr@mailbox.test) present -- stored_to=['ext-s10_uoybzx@external.example', 'user_p9pty8v2qr@mailbox.test']
[S10][authorized] mail_from=authorized-s10_xaczmd@delegate.example status='250 Message accepted for delivery' EmailLog.mailbox_id=65 (expect 65) stored_envelope_to=['ext-s10_uoybzx@external.example', 'moaltkkpasbtwhuuundh@moaltkkpasbtwhuuundh.com']
[S10][authorized] LOG> _handle:2342 ====>=====>====>====>====>====>====>====>
[S10][authorized] LOG> _handle:2343 New message, mail from authorized-s10_xaczmd@delegate.example, rctp tos ['ra+s10_zkelpf@sl.local'] 
[S10][authorized] LOG> handle:1963 Cannot parse Postfix queue ID from None None
[S10][authorized] LOG> handle:1980 ==>> Handle mail_from:authorized-s10_xaczmd@delegate.example, rcpt_tos:['ra+s10_zkelpf@sl.local'], header_from:authorized-s10_xaczmd@delegate.example, header_to:ra+s10_zkelpf@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'authorized-s10_xaczmd@delegate.example'), ('To', 'ra+s10_zkelpf@sl.local'), ('Subject', 're: s10 authorized'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
[S10][authorized] LOG> handle:2196 Reply phase authorized-s10_xaczmd@delegate.example(authorized-s10_xaczmd@delegate.example) -> ra+s10_zkelpf@sl.local
[S10][authorized] LOG> apply_dmarc_policy_for_reply_phase:159 DMARC check disabled
[S10][authorized] LOG> __check:1376 Found an authorized address for <Alias 117 affray_browse444@sl.local> <Mailbox 65 user_p9pty8v2qr@mailbox.test> <AuthorizedAddress 3 authorized-s10_xaczmd@delegate.example 65>
[S10][authorized] LOG> handle_reply:1051 Create <EmailLog 42> for <Contact 50 ext-s10_uoybzx@external.example 117>, <User 64 Test User user_p9pty8v2qr@mailbox.test>, <Mailbox 65 user_p9pty8v2qr@mailbox.test>
[S10][authorized] LOG> handle_reply:1171 From header is affray_browse444@sl.local
[S10][authorized] LOG> replace_header_when_reply:380 Replace To header, old: ra+s10_zkelpf@sl.local, new: ext-s10_uoybzx@external.example
[S10][authorized] LOG> replace_header_when_reply:383 delete the Cc header. Old value None
[S10][authorized] LOG> handle_reply:1206 missing date header, add one
[S10][authorized] LOG> handle_reply:1212 send email from affray_browse444@sl.local to ext-s10_uoybzx@external.example, mail_options:[],rcpt_options:[]
[S10][authorized] LOG> _handle:2367 Finish mail_from authorized-s10_xaczmd@delegate.example, rcpt_tos ['ra+s10_zkelpf@sl.local'], takes 0.038594722747802734 seconds with return code '250 Message accepted for delivery'<<===
ASSERT PASS: S10[authorized] status == E200 -- st='250 Message accepted for delivery'
ASSERT PASS: S10[authorized] selected mailbox_id == 65 -- {'id': 42, 'user_id': 64, 'mailbox_id': 65, 'alias_id': 117, 'contact_id': 50, 'is_reply': True}
ASSERT PASS: S10[authorized] relay to contact website present -- ext-s10_uoybzx@external.example
ASSERT PASS: S10[authorized] notify_mailbox to OTHER mailbox (moaltkkpasbtwhuuundh@moaltkkpasbtwhuuundh.com) present -- stored_to=['ext-s10_uoybzx@external.example', 'moaltkkpasbtwhuuundh@moaltkkpasbtwhuuundh.com']
[S10][unauthorized] mail_from=stranger-s10_ccaqgz@evil.example status='250 SL E214 Unauthorized for using reverse alias' stored_envelope_to=['user_p9pty8v2qr@mailbox.test']
[S10][unauthorized] LOG> _handle:2342 ====>=====>====>====>====>====>====>====>
[S10][unauthorized] LOG> _handle:2343 New message, mail from stranger-s10_ccaqgz@evil.example, rctp tos ['ra+s10_zkelpf@sl.local'] 
[S10][unauthorized] LOG> handle:1963 Cannot parse Postfix queue ID from None None
[S10][unauthorized] LOG> handle:1980 ==>> Handle mail_from:stranger-s10_ccaqgz@evil.example, rcpt_tos:['ra+s10_zkelpf@sl.local'], header_from:stranger-s10_ccaqgz@evil.example, header_to:ra+s10_zkelpf@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'stranger-s10_ccaqgz@evil.example'), ('To', 'ra+s10_zkelpf@sl.local'), ('Subject', 're: s10 unauth'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
[S10][unauthorized] LOG> handle:2196 Reply phase stranger-s10_ccaqgz@evil.example(stranger-s10_ccaqgz@evil.example) -> ra+s10_zkelpf@sl.local
[S10][unauthorized] LOG> apply_dmarc_policy_for_reply_phase:159 DMARC check disabled
[S10][unauthorized] LOG> handle_unknown_mailbox:1393 Reply email can only be used by mailbox. Actual mail_from: stranger-s10_ccaqgz@evil.example. msg from header: stranger-s10_ccaqgz@evil.example, reverse-alias ra+s10_zkelpf@sl.local, <Alias 117 affray_browse444@sl.local> <User 64 Test User user_p9pty8v2qr@mailbox.test> <Contact 50 ext-s10_uoybzx@external.example 117>
[S10][unauthorized] LOG> _handle:2367 Finish mail_from stranger-s10_ccaqgz@evil.example, rcpt_tos ['ra+s10_zkelpf@sl.local'], takes 0.03926563262939453 seconds with return code '250 SL E214 Unauthorized for using reverse alias'<<===
ASSERT PASS: S10[unauthorized] status == E214 -- st='250 SL E214 Unauthorized for using reverse alias'
ASSERT PASS: S10[unauthorized] no relay to contact website -- ext-s10_uoybzx@external.example
ASSERT PASS: S10[unauthorized] owner alerted (>=1 stored) -- total_stored=1
```

#### S11 — `NOREPLIES` short-circuit + bounce `mail_from == "<>"` ×2 — filtered (observation + `LOG>` + `ASSERT PASS` lines)

```text
[RUN 1] NOREPLIES rcpt=noreply@sl.local sender=user_thxk6o92vx@mailbox.test -> status='250 Message accepted for delivery' EmailLog_delta=0 stored_subjects=['Auto: hello s11r1']
[RUN 1][noreply] LOG> _handle:2342 ====>=====>====>====>====>====>====>====>
[RUN 1][noreply] LOG> _handle:2343 New message, mail from user_thxk6o92vx@mailbox.test, rctp tos ['noreply@sl.local'] 
[RUN 1][noreply] LOG> handle:1963 Cannot parse Postfix queue ID from None None
[RUN 1][noreply] LOG> handle:1980 ==>> Handle mail_from:user_thxk6o92vx@mailbox.test, rcpt_tos:['noreply@sl.local'], header_from:user_thxk6o92vx@mailbox.test, header_to:noreply@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'user_thxk6o92vx@mailbox.test'), ('To', 'noreply@sl.local'), ('Subject', 'hello s11r1'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
[RUN 1][noreply] LOG> handle:2182 email sent to noreply@sl.local address from user_thxk6o92vx@mailbox.test
[RUN 1][noreply] LOG> _handle:2367 Finish mail_from user_thxk6o92vx@mailbox.test, rcpt_tos ['noreply@sl.local'], takes 0.013583898544311523 seconds with return code '250 Message accepted for delivery'<<===
ASSERT PASS: S11[run1] NOREPLIES status == E200 -- st='250 Message accepted for delivery'
ASSERT PASS: S11[run1] NOREPLIES no EmailLog created (delta==0) -- delta=0
ASSERT PASS: S11[run1] NOREPLIES auto-response 'Auto:' stored -- subjects=['Auto: hello s11r1']
[RUN 1] BOUNCE mail_from='<>' rcpt=reverse-alias(ra+s11r1_emhkwd@sl.local) -> status='250 SL E206 Out of office' EmailLog_delta=0 total_stored=0
[RUN 1][bounce] LOG> _handle:2342 ====>=====>====>====>====>====>====>====>
[RUN 1][bounce] LOG> _handle:2343 New message, mail from <>, rctp tos ['ra+s11r1_emhkwd@sl.local'] 
[RUN 1][bounce] LOG> handle:1963 Cannot parse Postfix queue ID from None None
[RUN 1][bounce] LOG> handle:1980 ==>> Handle mail_from:<>, rcpt_tos:['ra+s11r1_emhkwd@sl.local'], header_from:bounce@mta.example, header_to:ra+s11r1_emhkwd@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'bounce@mta.example'), ('To', 'ra+s11r1_emhkwd@sl.local'), ('Subject', 'auto-reply s11r1'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
[RUN 1][bounce] LOG> handle:2168 out-of-office email to reverse alias <Contact 51 ext-s11r1_ewppjc@external.example 120>. Saved to 
[RUN 1][bounce] LOG> _handle:2367 Finish mail_from <>, rcpt_tos ['ra+s11r1_emhkwd@sl.local'], takes 0.005245685577392578 seconds with return code '250 SL E206 Out of office'<<===
ASSERT PASS: S11[run1] BOUNCE status == E206 (out of office) -- st='250 SL E206 Out of office'
ASSERT PASS: S11[run1] BOUNCE no EmailLog created (delta==0) -- delta=0
ASSERT PASS: S11[run1] BOUNCE nothing relayed (stored empty) -- total_stored=0
[RUN 2] NOREPLIES rcpt=noreply@sl.local sender=user_77wdgc1s8u@mailbox.test -> status='250 Message accepted for delivery' EmailLog_delta=0 stored_subjects=['Auto: hello s11r2']
[RUN 2][noreply] LOG> _handle:2342 ====>=====>====>====>====>====>====>====>
[RUN 2][noreply] LOG> _handle:2343 New message, mail from user_77wdgc1s8u@mailbox.test, rctp tos ['noreply@sl.local'] 
[RUN 2][noreply] LOG> handle:1963 Cannot parse Postfix queue ID from None None
[RUN 2][noreply] LOG> handle:1980 ==>> Handle mail_from:user_77wdgc1s8u@mailbox.test, rcpt_tos:['noreply@sl.local'], header_from:user_77wdgc1s8u@mailbox.test, header_to:noreply@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'user_77wdgc1s8u@mailbox.test'), ('To', 'noreply@sl.local'), ('Subject', 'hello s11r2'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
[RUN 2][noreply] LOG> handle:2182 email sent to noreply@sl.local address from user_77wdgc1s8u@mailbox.test
[RUN 2][noreply] LOG> _handle:2367 Finish mail_from user_77wdgc1s8u@mailbox.test, rcpt_tos ['noreply@sl.local'], takes 0.013060808181762695 seconds with return code '250 Message accepted for delivery'<<===
ASSERT PASS: S11[run2] NOREPLIES status == E200 -- st='250 Message accepted for delivery'
ASSERT PASS: S11[run2] NOREPLIES no EmailLog created (delta==0) -- delta=0
ASSERT PASS: S11[run2] NOREPLIES auto-response 'Auto:' stored -- subjects=['Auto: hello s11r2']
[RUN 2] BOUNCE mail_from='<>' rcpt=reverse-alias(ra+s11r2_nmzhsk@sl.local) -> status='250 SL E206 Out of office' EmailLog_delta=0 total_stored=0
[RUN 2][bounce] LOG> _handle:2342 ====>=====>====>====>====>====>====>====>
[RUN 2][bounce] LOG> _handle:2343 New message, mail from <>, rctp tos ['ra+s11r2_nmzhsk@sl.local'] 
[RUN 2][bounce] LOG> handle:1963 Cannot parse Postfix queue ID from None None
[RUN 2][bounce] LOG> handle:1980 ==>> Handle mail_from:<>, rcpt_tos:['ra+s11r2_nmzhsk@sl.local'], header_from:bounce@mta.example, header_to:ra+s11r2_nmzhsk@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'bounce@mta.example'), ('To', 'ra+s11r2_nmzhsk@sl.local'), ('Subject', 'auto-reply s11r2'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
[RUN 2][bounce] LOG> handle:2168 out-of-office email to reverse alias <Contact 52 ext-s11r2_lskbct@external.example 123>. Saved to 
[RUN 2][bounce] LOG> _handle:2367 Finish mail_from <>, rcpt_tos ['ra+s11r2_nmzhsk@sl.local'], takes 0.006707429885864258 seconds with return code '250 SL E206 Out of office'<<===
ASSERT PASS: S11[run2] BOUNCE status == E206 (out of office) -- st='250 SL E206 Out of office'
ASSERT PASS: S11[run2] BOUNCE no EmailLog created (delta==0) -- delta=0
ASSERT PASS: S11[run2] BOUNCE nothing relayed (stored empty) -- total_stored=0
```

#### RUNTIME-01 — Mixed-`To` cross-user recipient disclosure — filtered (observation + `LOG>` + `ASSERT PASS` lines)

```text
[RT01] userA alias=loiter_swerve171@sl.local mailbox=user_6gwfvealbd@mailbox.test revA=ra+a_iawqrq@sl.local extA=extA_mjcrhr@aaa.example
[RT01] userB alias=strati_abases836@sl.local revB=ra+b_qfowxc@sl.local extB=extB_hrmrqa@bbb.example
[RT01] driving mail_from=A_mailbox envelope rcpt_tos=[revA,revB] To header='ra+a_iawqrq@sl.local, ra+b_qfowxc@sl.local'
[RT01] overall status='250 Message accepted for delivery' stored_envelope_to=['extA_mjcrhr@aaa.example', 'user_lewdpaqbs0@mailbox.test']
[RT01] A's relay (to extA=extA_mjcrhr@aaa.example) rewritten To header = 'extA_mjcrhr@aaa.example,extB_hrmrqa@bbb.example'
[RT01] LOG> _handle:2342 ====>=====>====>====>====>====>====>====>
[RT01] LOG> _handle:2343 New message, mail from user_6gwfvealbd@mailbox.test, rctp tos ['ra+a_iawqrq@sl.local', 'ra+b_qfowxc@sl.local'] 
[RT01] LOG> handle:1963 Cannot parse Postfix queue ID from None None
[RT01] LOG> handle:1980 ==>> Handle mail_from:user_6gwfvealbd@mailbox.test, rcpt_tos:['ra+a_iawqrq@sl.local', 'ra+b_qfowxc@sl.local'], header_from:user_6gwfvealbd@mailbox.test, header_to:ra+a_iawqrq@sl.local, ra+b_qfowxc@sl.local, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'user_6gwfvealbd@mailbox.test'), ('To', 'ra+a_iawqrq@sl.local, ra+b_qfowxc@sl.local'), ('Subject', 're: mixed'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
[RT01] LOG> handle:2189 copy message for rcpt ra+a_iawqrq@sl.local
[RT01] LOG> handle:2196 Reply phase user_6gwfvealbd@mailbox.test(user_6gwfvealbd@mailbox.test) -> ra+a_iawqrq@sl.local
[RT01] LOG> apply_dmarc_policy_for_reply_phase:159 DMARC check disabled
[RT01] LOG> handle_reply:1051 Create <EmailLog 43> for <Contact 53 extA_mjcrhr@aaa.example 125>, <User 69 Test User user_6gwfvealbd@mailbox.test>, <Mailbox 71 user_6gwfvealbd@mailbox.test>
[RT01] LOG> handle_reply:1171 From header is loiter_swerve171@sl.local
[RT01] LOG> replace_header_when_reply:380 Replace To header, old: ra+a_iawqrq@sl.local, ra+b_qfowxc@sl.local, new: extA_mjcrhr@aaa.example,extB_hrmrqa@bbb.example
[RT01] LOG> replace_header_when_reply:383 delete the Cc header. Old value None
[RT01] LOG> handle_reply:1206 missing date header, add one
[RT01] LOG> handle_reply:1212 send email from loiter_swerve171@sl.local to extA_mjcrhr@aaa.example, mail_options:[],rcpt_options:[]
[RT01] LOG> handle:2196 Reply phase user_6gwfvealbd@mailbox.test(user_6gwfvealbd@mailbox.test) -> ra+b_qfowxc@sl.local
[RT01] LOG> apply_dmarc_policy_for_reply_phase:159 DMARC check disabled
[RT01] LOG> handle_unknown_mailbox:1393 Reply email can only be used by mailbox. Actual mail_from: user_6gwfvealbd@mailbox.test. msg from header: user_6gwfvealbd@mailbox.test, reverse-alias ra+b_qfowxc@sl.local, <Alias 127 strati_abases836@sl.local> <User 70 Test User user_lewdpaqbs0@mailbox.test> <Contact 54 extB_hrmrqa@bbb.example 127>
[RT01] LOG> handle:2225 some deliveries fail and some success, user_6gwfvealbd@mailbox.test, ['ra+a_iawqrq@sl.local', 'ra+b_qfowxc@sl.local'], [(True, '250 Message accepted for delivery'), (False, '250 SL E214 Unauthorized for using reverse alias')]
[RT01] LOG> _handle:2367 Finish mail_from user_6gwfvealbd@mailbox.test, rcpt_tos ['ra+a_iawqrq@sl.local', 'ra+b_qfowxc@sl.local'], takes 0.05263781547546387 seconds with return code '250 Message accepted for delivery'<<===
ASSERT PASS: RT01 overall status == E200 (revA leg succeeds) -- st='250 Message accepted for delivery'
ASSERT PASS: RT01 A's reply relayed to A's own contact (extA) -- extA_mjcrhr@aaa.example
ASSERT PASS: RT01 A's rewritten To CONTAINS extA (own contact) -- To=extA_mjcrhr@aaa.example,extB_hrmrqa@bbb.example
ASSERT PASS: RT01 CROSS-USER DISCLOSURE: A's To header CONTAINS user B's extB -- To=extA_mjcrhr@aaa.example,extB_hrmrqa@bbb.example extB=extB_hrmrqa@bbb.example
[RT01] OBSERVED: replace_header_when_reply resolved revB via a global, un-scoped Contact.get_by (email_handler.py:L363) and injected user B's external contact address into user A's outbound reply To header.
```

#### RUNTIME-02 — Uppercase-domain `To` → silent non-delivery — filtered (observation + `LOG>` + `ASSERT PASS` lines)

```text
[RT02] envelope rcpt (lowercase)=ra+up_shjmxq@sl.local ; To header (UPPERCASE domain)=ra+up_shjmxq@SL.LOCAL
[RT02] status='250 Message accepted for delivery' reply_EmailLog_after=None relay_to_contact=no stored_subjects=['Email sent to ext-up_wltmkt@external.example contains non reverse-alias addresses']
[RT02] LOG> _handle:2342 ====>=====>====>====>====>====>====>====>
[RT02] LOG> _handle:2343 New message, mail from user_rlek6197u2@mailbox.test, rctp tos ['ra+up_shjmxq@sl.local'] 
[RT02] LOG> handle:1963 Cannot parse Postfix queue ID from None None
[RT02] LOG> handle:1980 ==>> Handle mail_from:user_rlek6197u2@mailbox.test, rcpt_tos:['ra+up_shjmxq@sl.local'], header_from:user_rlek6197u2@mailbox.test, header_to:ra+up_shjmxq@SL.LOCAL, cc:None, reply-to:None, message_id:None, client_ip:None, headers:[('From', 'user_rlek6197u2@mailbox.test'), ('To', 'ra+up_shjmxq@SL.LOCAL'), ('Subject', 're: upper'), ('Content-Type', 'text/plain; charset="utf-8"'), ('Content-Transfer-Encoding', '7bit'), ('MIME-Version', '1.0')], mail_options:[], rcpt_options:[]
[RT02] LOG> handle:2196 Reply phase user_rlek6197u2@mailbox.test(user_rlek6197u2@mailbox.test) -> ra+up_shjmxq@sl.local
[RT02] LOG> apply_dmarc_policy_for_reply_phase:159 DMARC check disabled
[RT02] LOG> handle_reply:1051 Create <EmailLog 44> for <Contact 55 ext-up_wltmkt@external.example 129>, <User 71 Test User user_rlek6197u2@mailbox.test>, <Mailbox 73 user_rlek6197u2@mailbox.test>
[RT02] LOG> handle_reply:1171 From header is yawing_jumped041@sl.local
[RT02] LOG> replace_header_when_reply:366 email ra+up_shjmxq@SL.LOCAL contained in To header in reply phase must be reply emails. headers:['ra+up_shjmxq@SL.LOCAL']
[RT02] LOG> handle_reply:1183 non reverse-alias in reply NonReverseAliasInReplyPhase ra+up_shjmxq@SL.LOCAL <Contact 55 ext-up_wltmkt@external.example 129> <Alias 129 yawing_jumped041@sl.local>
[RT02] LOG> _handle:2367 Finish mail_from user_rlek6197u2@mailbox.test, rcpt_tos ['ra+up_shjmxq@sl.local'], takes 0.03614068031311035 seconds with return code '250 Message accepted for delivery'<<===
ASSERT PASS: RT02 status == E200 (250 returned to sender) -- st='250 Message accepted for delivery'
ASSERT PASS: RT02 reply EmailLog was created THEN deleted (now None) -- None
ASSERT PASS: RT02 reply NOT relayed to contact website (silent non-delivery) -- ext-up_wltmkt@external.example
ASSERT PASS: RT02 owner alerted about non reverse-alias address (>=1 stored) -- subjects=['Email sent to ext-up_wltmkt@external.example contains non reverse-alias addresses']
[RT02] OBSERVED: a reply whose To-header reverse alias differs only by domain LETTER CASE is accepted at SMTP (250) but silently dropped — the case-sensitive Contact.get_by in replace_header_when_reply (L363) misses and the delivery is aborted after the EmailLog is deleted.
```

#### RUNTIME-INFO-01 — No `Message-ID` de-duplication (at-least-once) — filtered (observation + `LOG>` + `ASSERT PASS` lines)

```text
[RTI01] fixed Message-ID=<fixed-fjrykkosrr@mta.example> driven x2 (identical bytes)
[RTI01] drive1 status='250 Message accepted for delivery' EmailLog.id=45 relay=yes
[RTI01] drive2 status='250 Message accepted for delivery' EmailLog.id=46 relay=yes
ASSERT PASS: RTI01 drive1 == E200 -- st1='250 Message accepted for delivery'
ASSERT PASS: RTI01 drive2 == E200 -- st2='250 Message accepted for delivery'
ASSERT PASS: RTI01 TWO distinct EmailLog rows (no dedup) -- id1=45 id2=46
ASSERT PASS: RTI01 BOTH drives relayed (at-least-once, no idempotency) -- both relays expected
```


---

## (i) Observed vs. inferred — key-claim ledger

The distinction is annotated inline throughout. The most consequential claims are consolidated here; every "observed" row is backed by a labeled `ASSERT PASS` line in the single-run transcript, [§ (h)](#h-commands-run--raw-runtime-output).

| # | Claim | Basis |
|---|-------|-------|
| 1 | Reply ingress is `handle_DATA` → `_handle` → `handle` (`email_handler.py:L2289`, `L2335`, `L1945`) | **observed** (code + `S1` log lines `_handle:2343`, `handle:2196`) |
| 2 | The alias is resolved indirectly via `Contact` (`reply_email` → `Contact` → `Contact.alias` → `Alias.user`) | **observed** (code `email_handler.py:L986`/`L994`/`L1004`; `S1` chain `contact.id=35 → alias.id=85 → user.id=47`) |
| 3 | On the happy path the decided `user_id` is `47`, and `EmailLog.user_id == contact.user_id == alias.user_id` | **observed** (`S1`, both runs) |
| 4 | The handler uses DISTINCT user references at distinct gates: Gate 1 `contact.user` (`L990`); Gate 2 `alias.user` (`L1004`/`L1007`); Gate 3 both (`L1012`); Gate 4 alias-scoped mailbox (`L1019`); persist `contact.user_id` (`L1046`) | **observed** (code + `S2` `GATES-INPUT` line) |
| 5 | Those references CAN diverge, yielding `EmailLog.user_id=49`/`51` while gates used `alias.user_id=48`/`50` | **observed** (`S2`, both runs, `MISMATCH=True`) |
| 6 | `reply_email` is not unique; two contacts (two users) can share it; `.first()` returns one row with NO `ORDER BY` | **observed** (`app/models.py:L1899`/`L1875`/`L84`; `S3` `DB row count = 2`) |
| 7 | In `S3`/`S3b` the selected row was the first-inserted/lowest-PK in every run, but this is not guaranteed by the query | **observed** (`S3`/`S3b` `selected == first-inserted-contact? True`) / **explicitly labeled not-guaranteed** (`app/models.py:L84`) |
| 8 | Selected owner's reply is delivered (`250`) while the shadowed owner's reply is rejected (`E214`); on a fixed fixture this is STABLE across 5 repeats | **observed** (`S3` both insertion orders; `S3b` ×5) |
| 9 | The happy-path reply egresses OUTWARD to `contact.website_email` with `From` rewritten to the alias; the real mailbox is absent from the WHOLE 747-byte message | **observed** (`S1` `OUT`, `real_mailbox…present_anywhere=False`, both runs) |
| 10 | Guard paths return exact statuses with no reply `EmailLog`: E501 (bad reply domain), E502 (no contact / inactive user), E503 (unmanaged alias domain), E504 (disabled user); a `\x01` control byte is normalized and the reply succeeds | **observed** (`S8a`–`S8f`) |
| 11 | Unauthorized sender with spoofing ON → E214 (no reply `EmailLog`, owner alert); with spoofing OFF → default-mailbox fallback, reply created (`used_default_mailbox=True`) — proven as a single-input differential | **observed** (`S9`) |
| 12 | Multi-mailbox alias: `EmailLog.mailbox_id` == the **sending** mailbox (default/secondary/delegate); each authorized reply also emits a `notify_mailbox` to the alias's *other* mailbox; an unauthorized sender → E214 | **observed** (`S10`; `email_handler.py:L1364`/`L1047`/`L1264–L1293`) |
| 13 | `NOREPLIES` → `250` with an `Auto:` auto-response and `EmailLog_delta=0`; bounce `<>` → E206 with `EmailLog_delta=0`, `total_stored=0` | **observed** (`S11`, both runs) |
| 14 | DMARC did not block, but only because it is NON-BLOCKING here (`DMARC check disabled` → return `None`), not a "clean pass" | **observed** (`app/handler/dmarc.py:L157–L160`; `DMARC check disabled` on every reply run) |
| 15 | `RUNTIME-01`: a `To` header with two reverse aliases of different users relays user A's reply with user **B's** external contact injected into the `To` header (`extA…,extB…`) | **observed** (`RUNTIME-01`; `email_handler.py:L364` un-scoped `Contact.get_by`, `L1177–L1180`) |
| 16 | `RUNTIME-02`: a `To`-header reverse alias differing only by domain letter case is accepted (`250`) but silently dropped — `EmailLog` created then deleted, internal notice sent instead | **observed** (`RUNTIME-02`; `email_handler.py:L364`, `L1183–L1198`) |
| 17 | `RUNTIME-INFO-01`: two byte-identical inbound replies (same `Message-ID`) create two `EmailLog` rows (`45`, `46`) and both are relayed — at-least-once, no de-duplication | **observed** (`RUNTIME-INFO-01`) |
| 18 | For the correctly-owned common case, a "wrong user" report is most likely a misunderstanding of the outward-relay model | **inferred** (from claims 3 & 9) |
| 19 | The single most likely mis-routing origin is `Contact.get_by(reply_email=…).first()` (`email_handler.py:L986`) + non-unique schema + `.first()` — Q5b, reproduced in `S3`/`S3b` | **inferred** (from claims 2, 6, 7, 8; no abnormal ownership needed) |
| 20 | The same un-scoped/non-unique `Contact.get_by`-on-`reply_email` weakness appears in the To-header rewrite (`email_handler.py:L364`), producing `RUNTIME-01`/`-02` | **observed** (code + `RUNTIME-01`/`-02`) / **inferred** (that it is the same weakness) |
| 21 | The dual user reference is a pipeline-wide pattern (forward path uses it too: `L557` vs `L600`/`L734`) | **observed** (code) / **inferred** (that it is the same pattern) |
| 22 | Divergent `alias.user_id`/`contact.user_id` is an abnormal state because `transfer_alias()` updates both together | **observed** (`app/alias_utils.py:L464–L466`, `L506`) / **inferred** (that divergence is therefore abnormal) |

---

## (j) Security / privacy note

**(observed)** The reverse-alias mechanism exists to keep the user's real mailbox hidden. This was verified in `S1` by serializing the **entire** outbound message (`SendRequest.msg` via `message_to_bytes`, `747` bytes) and searching the whole byte stream for the user's real mailbox address `user_juvpu3izq5@mailbox.test`: the search returned `real_mailbox(user_juvpu3izq5@mailbox.test)_present_anywhere=False`. The message's complete header set was `['Subject', 'Content-Type', 'Content-Transfer-Encoding', 'MIME-Version', 'From', 'To', 'Message-ID', 'Date', 'X-SimpleLogin-Type', 'X-SimpleLogin-EmailLog-ID', 'DKIM-Signature']`, exposing only the alias (`From: cuppas_workup671@sl.local`) and the external contact (`To: ext-s1_kdpnyi@external.example`); the real mailbox appears in no header (including the DKIM signature and message-id) or body. This was consistent across both identical-input runs (`ASSERT PASS: S1[run1]` and `S1[run2] PRIVACY: real mailbox NOT present anywhere in message`).

**(observed) The privacy guarantee is violated in two reproduced cases where the alias is still recognized:**

- **`RUNTIME-01` — cross-user recipient disclosure.** When a reply's `To` header lists reverse aliases belonging to **different users**, `replace_header_when_reply` resolves each via the un-scoped `Contact.get_by(reply_email=…)` (`email_handler.py:L364`) and rewrites them to their external contacts. User A's outbound reply left with `To: extA_mjcrhr@aaa.example,extB_hrmrqa@bbb.example` — i.e., user **B's** external contact was disclosed to user **A's** recipient. This is an unintended-recipient disclosure that crosses a user boundary while `is_reverse_alias` reports both aliases as "recognized."
- **`RUNTIME-02` — silent loss.** A reply whose `To`-header reverse alias differs only by domain letter case is accepted (`250`) but never relayed (the `EmailLog` is created then deleted and an internal notice is stored instead — `email_handler.py:L1183–L1198`). A silently-dropped reply is an availability/confidentiality concern: the sender believes the reply was delivered.

**(inferred)** Therefore, any mis-routing that resolves the **wrong `Contact`/user** — Hypothesis (b), reproduced in `S3`/`S3b`, and the body-rewrite variant Hypothesis (d), reproduced in `RUNTIME-01`/`-02` — is not merely a correctness bug but a **privacy-relevant defect**: because the alias, the recorded `user_id`, and the destination `website_email` all hang off the single contact `.first()` returns (and each `To`-header address off a single un-scoped `Contact.get_by`), resolving to the wrong contact routes/attributes/discloses the reply via a *different* user's contact while the alias still appears "recognized." In `S3` the shadowed owner was rejected with `E214` (fail-closed); but `RUNTIME-01` shows the same resolution weakness failing **open** into a cross-user disclosure. Because the whole point of the reverse alias is confidentiality, a wrong-contact resolution is a confidentiality risk, not just an attribution bug.

---

## (k) Cleanup & repository-state proof

Per the user's explicit instruction ("create temporary scripts or logs … clean them up and leave the codebase as you found it") and the read-only rule, all observation scaffolding was transient and external to the repository working tree:

- **(observed) Where the scaffolding lived.** The harness (`blitzy_reply_trace.py`), the egress probe, and every captured `.log` transcript resided under `/tmp/slrun/` (a `git archive HEAD` export) **inside the run container**, never under the repository working tree. Nothing was written into the source repo by the investigation except this one document.
- **(observed) Working-tree cleanliness.** The only change in the repository is this deliverable. Observed at investigation completion (the document written but not yet committed), the verification commands and their results:

  ```text
  $ git status --porcelain
   M blitzy/documentation/app_2cd6ee777f8c.md

  $ git diff --stat -- pyproject.toml poetry.lock
  (no output)

  $ git diff --name-status -- email_handler.py app/ tests/
  (no output)
  ```

  The ` M` (modified) status — rather than a new-file `??` — reflects that a prior documentation-only commit (`49d585cd`, HEAD) already tracked this exact path; the reconstruction therefore registers as a modification of the deliverable, never a source change. Either way it is the sole differing path: no source file (`email_handler.py`, anything under `app/`, or any file under `tests/`) is modified, added, or deleted; `pyproject.toml`/`poetry.lock` are byte-identical (digests in [§ (h)](#dependency-integrity--baseline-advisories)); and the sole changed path is `blitzy/documentation/app_2cd6ee777f8c.md`.
- **(observed) Container-only environment fixes.** The two setup fixes (`local_data/dkim.key` → PKCS#1; `re2`→`re` shim in `app/spamassassin_utils.py`) were applied only inside the container image's copy, not in the repository working tree, and touch no cited `file:line`. They do not appear in `git status` of the working tree.

The repository is therefore left as found, with this document as the only added artifact.

---

## Coverage recap (final pass)

- **Q1 — Entry point:** answered in [§ (b)](#b-q1--which-component-handles-the-incoming-reply-entry-point) — `MailHandler.handle_DATA()` (`email_handler.py:L2289`) → `_handle()` (`L2335`) → `handle()` (`L1945`); production ingress at `127.0.0.1:20381` (`README.md:L367`).
- **Q2 — Resolution:** answered in [§ (c)](#c-q2--how-is-the-alias-resolved-to-a-user-resolution-chain) — `is_reverse_alias` (`app/email_utils.py:L1156`) → `handle_reply` (`L966`) → `Contact.get_by(reply_email).first()` (`L986` / `app/models.py:L84`) → `contact.alias` (`L994`) → `alias.user` (`L1004`).
- **Q3 — Decided `user_id`:** answered in [§ (d)](#d-q3--the-concrete-integer-user_id-the-system-decides) — happy-path value `47` (stable across two identical-input runs); persisted via `EmailLog.create(user_id=contact.user_id …)` (`L1046`); two independent references (`alias.user` vs `contact.user_id`).
- **Q4 — Data flow:** answered in [§ (e)](#e-q4--actual-observed-end-to-end-data-flow) — full gate-by-gate narrative + diagram, each node tied to `S1`/`S2` evidence; DMARC shown to be non-blocking.
- **Q5 — Most likely origin:** answered in [§ (f)](#f-q5--most-likely-point-where-an-incorrect-routing-decision-originates) — four hypotheses reproduced (Q5a in `S2`, Q5b canonically in `S3` + fixed-fixture `S3b`, Q5c as a controlled differential in `S9`, and the body-rewrite variant in `RUNTIME-01`/`-02`); single most likely origin = `Contact.get_by(reply_email=…).first()` (`email_handler.py:L986`) with the non-unique `reply_email` schema (`app/models.py:L1899`) and `.first()`-without-`ORDER BY` semantics (`app/models.py:L84`).
- **Edge conditions:** exercised in [§ (g)](#g-edge-condition-results) — happy path (`S1`), divergence (`S2`), non-unique collision (`S3`, `S3b`), all `handle_reply` guards (`S8a`–`S8f`: E501/E502/E503/E504/normalization), spoofing ON/OFF differential (`S9`), **multi-mailbox alias** with default/secondary/authorized/unauthorized senders + `notify_mailbox` (`S10`; AAP §0.5.1), `NOREPLIES`/bounce (`S11`), plus the two reply-body defects (`RUNTIME-01`, `RUNTIME-02`) and the no-dedup note (`RUNTIME-INFO-01`) — each with explicit `ASSERT PASS` checks.
- **Integrity:** existing suite green ×2 (`23 passed`), migrations idempotent, dependency manifests unchanged, and a network audit hook proving no non-loopback / no SMTP egress — all in [§ (h)](#h-commands-run--raw-runtime-output); repository left git-clean apart from this document ([§ (k)](#k-cleanup--repository-state-proof)).

*End of analysis. The only artifact added to the repository is this document; no source file was modified, and all temporary observation scaffolding lived inside the run container and was removed.*
