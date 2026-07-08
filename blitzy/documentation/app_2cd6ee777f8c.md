# SimpleLogin Inbound Bounce Handling — Runtime-Verified Security Investigation

> **Scope of this document.** This is a **runtime-verified** security investigation of how
> SimpleLogin's inbound SMTP handler (`email_handler.py`) routes *bounce* email across its two
> Variable Envelope Return Path (VERP) address formats, and whether the **older, plaintext**
> format constitutes an information-disclosure / enumeration **oracle** exploitable by an external
> attacker who only observes SMTP responses. Every substantive claim below is backed by **actual
> captured output** from running the real code path **and/or** a precise `file:line` citation.
> No repository source file was modified; the temporary observation scripts used to elicit the
> evidence were deleted on completion (see the Appendix `git status` proof).

---

## 1. Executive summary & threat model

**Finding (confirmed at runtime): the OLD plaintext VERP bounce address is an email-log-identifier
enumeration oracle.** An external sender who crafts a bounce message to the legacy address shape
`bounce+{id}+@{domain}` learns, purely from the returned SMTP status line, **whether an internal
`email_log` id exists**, and — when it does — **what processing phase it is in and whether its owning
account is active**. The differential is stark and stable:

- A **non-existent** id returns `550 SL E512 No such email log`.
- An **existing** id returns a `250 SL E21x` (accepted) or a different `550 SL E510` (inactive user).

This is the same class of vulnerability as classic SMTP `RCPT TO` recipient enumeration (an accepted
recipient returns a success code, an unknown one a `550`); here the enumerated secret is an internal
`email_log` primary key rather than a mailbox. The repository's **observed runtime behavior is the
authoritative source** for everything below; the analogy to recipient enumeration is only framing.

The root cause is a missing cryptographic gate on the legacy path:

| | OLD format `bounce+{id}+@domain` | NEW format `sl.{payload}.{sig}@domain` |
|---|---|---|
| Id source | `parse_id_from_bounce()` — a raw `int()` slice | `get_verp_info_from_email()` — decode signed payload |
| Integrity check | **none** | HMAC-SHA3-224 recomputed & compared |
| Freshness check | **none** | 5-day lifetime bound |
| Attacker can forge a chosen id? | **yes** | no (needs `VERP_EMAIL_SECRET`) |

Both `is_bounce()` acceptance criteria (envelope `MAIL FROM:<>` **and** MIME
`Content-Type: multipart/report`) are fully attacker-controllable, so any external sender can drive
the message all the way to the id-existence gate on the legacy path.

**The six questions, answered at a glance (all runtime-verified below):**

| # | Question | Verified answer |
|---|---|---|
| Q1 | How are bounces routed across the two formats? | `handle()` selects a branch by recipient shape: legacy prefix `bounce+…+@domain` → plaintext `parse_id_from_bounce`; signed `sl.…@domain` → HMAC-verified `get_verp_info_from_email`. Both converge on `EmailLog.get(id)`. |
| Q2 | Can an attacker probe via SMTP responses? | **Yes.** Crafting bounces to legacy addresses and reading the reply line reveals id existence and more — demonstrated on the real aiosmtpd daemon. |
| Q3 | What slips through without crypto validation? | The **entire legacy branch**: `parse_id_from_bounce` is a pure integer slice with no signature/lifetime. Tampering a *signed* address is rejected (`None`); the legacy path accepts any integer verbatim. |
| Q4 | Exact SMTP responses per scenario? | Verbatim matrix in §7 — `E512`, `E211`, `E212`, `E510` (literal typo *"so such user"*), `E213`, `E515`, `E404`. |
| Q5 | What makes a message a "bounce", and is it spoofable? | `is_bounce()` = `mail_from == "<>"` **and** `Content-Type: multipart/report`. Both attacker-controllable → fully spoofable. Pass → `handle_bounce`; fail → `raise VERP*` → `E213`. |
| Q6 | Does the oracle leak more than existence? | **Yes.** Beyond `250`-vs-`550`, the distinct strings leak processing **phase** (forward/reply) and **account state** (active/inactive). |

---

## 2. Environment & methodology

### 2.1 Run-first principle

Per the governing rule and the explicit user directive — *"I don't want theoretical explanations of
what the code should do, I want to see actual evidence of how the system behaves in practice"* — the
investigation was conducted by **building and running the real code paths first**, capturing output,
and only then writing this document. The canonical inbound entry point is the aiosmtpd server in
`email_handler.py`; it was exercised as a live daemon, not reimplemented.

### 2.2 Canonical runtime

The canonical runtime is the provided Docker image
`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (SimpleLogin `app`). All code ran
inside that container against a real PostgreSQL 13 database.

| Component | Value (verified at runtime) | Source |
|---|---|---|
| Python | 3.10.18 | `pyproject.toml:61` (`python = "^3.10"`) |
| SMTP framework | `aiosmtpd` 1.4.2 | `pyproject.toml:87` (`aiosmtpd = "^1.2"`) |
| Database | PostgreSQL, `postgresql://test:test@localhost:15432/test`, migrations at head `32f25cbf12f6` | `tests/test.env:17` |
| `EMAIL_DOMAIN` | `sl.local` | `tests/test.env:8` |
| `FLASK_SECRET` | `secret` | `tests/test.env:20` |
| `VERP_EMAIL_SECRET` | **36 chars** (default `FLASK_SECRET` + `"pleasegenerateagoodrandomtoken"`) — passes the ≥32 guard | `app/config.py:502-508` |
| `NOT_SEND_EMAIL` | `true` | `tests/test.env:7` |

The `VERP_EMAIL_SECRET` ≥32-character guard is enforced at import time; the process aborts with
`RuntimeError` otherwise (`app/config.py:505-508`). Verified length at runtime:

```
$ docker exec -e CONFIG=/workspace/tests/test.env -e GITHUB_ACTIONS_TEST=true -w /workspace \
    sl-app /app/venv/bin/python -c "from app import config; print(len(config.VERP_EMAIL_SECRET))"
36
```

### 2.3 Two entry-path strategies

Two complementary strategies drove the **same** code, and their results agree everywhere (§7.2):

- **Path A — canonical wire path (primary evidence).** The real daemon was started with the
  canonical command and driven with a raw SMTP client (`smtplib`). The reply to the terminating `.`
  of `DATA` **is** `MailHandler.handle_DATA`'s return value (`email_handler.py:2289-2293`), i.e. the
  verbatim SMTP wire status line.

  ```
  # start the canonical daemon (aiosmtpd Controller, email_handler.py:2381-2383, default port 20381:2399)
  CONFIG=/workspace/tests/test.env GITHUB_ACTIONS_TEST=true \
    /app/venv/bin/python email_handler.py -p 20381
  # log line confirms: "Start mail controller 0.0.0.0 20381"
  ```

- **Path B — handler harness (corroborating).** Mirrors `tests/test_email_handler.py:81-85`: build an
  `Envelope()`, construct the `Message` via `email.message_from_bytes(...)` (the exact call
  `load_eml_file` makes at `tests/utils.py:87`), set `mail_from`/`rcpt_tos`, and invoke the
  module-level `email_handler.handle(...)` the test suite itself uses.

### 2.4 The 5XX→E216 SPF-rewrite nuance (controlled for)

`_handle` runs `handle()` inside `create_light_app().app_context()` and then **rewrites a 5XX status
to `E216` *only if*** a `SpamdResult` is present in the headers **and** its SPF result is
`fail`/`soft_fail` (`email_handler.py:2356-2365`; the header key is `X-Spamd-Result`,
`app/email/headers.py:16`). All crafted inputs deliberately **omit** any `X-Spamd-Result` header, so
`spamd_result` is `None` and every `550` passes through **unchanged** — which is exactly what was
observed (clean `550 SL E512` / `550 SL E510`, never `E216`).

### 2.5 Seeded ground truth

Using the repository's own helpers (`tests/utils.py:create_new_user`, `Alias.create_new_random`,
`Contact.create`, `EmailLog.create`) inside `create_app()`'s app context, the following ground-truth
rows were seeded (verified via direct `psql`):

| Purpose | `email_log.id` | Properties | Probe address |
|---|---|---|---|
| **Valid, forward phase** | `407` | user 807 active, `is_reply=False` | `bounce+407+@sl.local` |
| **Valid, reply phase** | `408` | user 807 active, `is_reply=True` | `bounce+408+@sl.local` / `bounce_reply+408+@sl.local` |
| **Valid, inactive user** | `409` | user 808 `delete_on` in the future ⇒ `is_active()==False` | `bounce+409+@sl.local` |
| **Known-absent** | `10000409` | `EmailLog.get(10000409) is None` ⇒ `True` | `bounce+10000409+@sl.local` |
| **State side-effect (fresh)** | `410` / `411` | forward / reply, both `bounced=False` initially | `bounce+410+@sl.local` / `bounce+411+@sl.local` |

`User.is_active()` returns `False` when `delete_on` is set to a future date
(`if self.delete_on is None: return True; return self.delete_on < arrow.now()`,
`app/models.py:766-769`) — that is the "inactive user" condition exercised for `E510`.

---

## 3. Q1 — Routing across the two VERP formats

### 3.1 The two address shapes

- **OLD (plaintext).** Built as `BOUNCE_PREFIX + str(email_log.id) + BOUNCE_SUFFIX`
  (`app/config.py:99`), i.e. `bounce+{email_log.id}+@{EMAIL_DOMAIN}`. With the test config that is
  `bounce+{id}+@sl.local` (`BOUNCE_PREFIX = "bounce+"` `app/config.py:100`;
  `BOUNCE_SUFFIX = "+@{EMAIL_DOMAIN}"` `app/config.py:101`). The reply-phase variant uses
  `BOUNCE_PREFIX_FOR_REPLY_PHASE = "bounce_reply"` (`app/config.py:108-110`), i.e.
  `bounce_reply+{id}+@sl.local`. **The id travels in clear text.**
- **NEW (signed).** Built by `generate_verp_email()` (`app/email_utils.py:1438-1464`) as
  `"{VERP_PREFIX}.{b32(payload)}.{b32(hmac_sha3_224(secret,payload)[:8])}@{domain}".lower()`, i.e.
  `sl.{payload}.{sig}@sl.local` (`VERP_PREFIX = "sl"`, `app/config.py:500`). The payload
  `[verp_type, object_id, minutes]` is **HMAC-signed**.

### 3.2 Branch selection in `handle()`

`handle()` (`email_handler.py:1945`) first computes `verp_info = get_verp_info_from_email(rcpt_tos[0])`
(`email_handler.py:2035`) and then selects a branch by **recipient shape**:

- **Forward-VERP branch** (`email_handler.py:2059-2074`): taken when
  `rcpt_tos[0].startswith(BOUNCE_PREFIX) and rcpt_tos[0].endswith(BOUNCE_SUFFIX)`
  **or** `verp_info[0] == VerpType.bounce_forward`. Then:
  - `email_log_id = (verp_info and verp_info[1]) or parse_id_from_bounce(rcpt_tos[0])` (`:2062`)
  - `email_log = EmailLog.get(email_log_id)` (`:2063`)
  - **existence gate:** `if not email_log: return status.E512` (`:2065-2067`)
  - `if is_bounce(...): return handle_bounce(...)` (`:2069-2070`) **else** `raise VERPForward` (`:2074`)
- **Reply-VERP branch** (`email_handler.py:2079-2095`): taken when
  `rcpt_tos[0].startswith(f"{BOUNCE_PREFIX_FOR_REPLY_PHASE}+")` **or**
  `verp_info[0] == VerpType.bounce_reply`; identical existence gate → `E512` (`:2087`); on non-bounce
  `raise VERPReply` (`:2095`).

Two subtleties confirmed at runtime:

1. **Branch vs. phase are independent.** The *branch* (forward-VERP vs. reply-VERP) is chosen by the
   **address prefix** (`bounce+` vs. `bounce_reply+`), whereas the **`E211` vs. `E212`** outcome is
   decided *inside* `handle_bounce` by the stored `email_log.is_reply`
   (`email_handler.py:1910-1914`). Probing the reply-phase id `408` at the **forward**-prefix address
   `bounce+408+@sl.local` still yields `E212`, because `handle_bounce` keys on `is_reply`:

   ```
   $ ... probe_wire.py 20381 "bounce+408+@sl.local"       ... dsn.eml  → FINAL -> 250 SL E212 Bounce Reply phase handled
   $ ... probe_wire.py 20381 "bounce_reply+408+@sl.local" ... dsn.eml  → FINAL -> 250 SL E212 Bounce Reply phase handled
   ```

2. **Both formats converge on `EmailLog.get(id)`**, so both are gated identically by id existence
   (`E512` when absent). The only difference is *how the id is obtained* — plaintext slice vs. verified
   signature — which is the crux of Q3.

### 3.3 Routing flow diagram

```mermaid
flowchart TD
    A[Inbound message: handle_DATA] --> B{Recipient shape?}
    B -->|"old: bounce+ID+@domain"| C["parse_id_from_bounce()<br/>plaintext slice, NO HMAC"]
    B -->|"new: sl.payload.sig@domain"| D["get_verp_info_from_email()<br/>HMAC-SHA3-224 + 5-day lifetime"]
    C --> E["EmailLog.get(id)"]
    D --> E
    E -->|not found| F["550 SL E512 No such email log"]
    E -->|found| G{"is_bounce?<br/>mail_from == '<>' AND<br/>Content-Type multipart/report"}
    G -->|no| H["raise VERPForward/VERPReply<br/>-> caught -> 250 SL E213"]
    G -->|yes| I{"handle_bounce()"}
    I -->|user inactive| J["550 SL E510 so such user"]
    I -->|reply phase| K["250 SL E212 Bounce Reply phase handled"]
    I -->|forward phase| L["250 SL E211 Bounce Forward phase handled"]
```

Each diagram terminal was **observed** on the live daemon (see the verbatim matrix in §7).

---

## 4. Q2 — Attacker probing via SMTP responses

An external sender who can open an SMTP connection to the inbound handler can craft bounces to the
legacy address and **read the internal state out of the reply line**. Below are the *actual* wire
transcripts against the running daemon (`email_handler.py -p 20381`). The client is a plain
`smtplib` session; `s.mail("")` emits the null reverse-path `MAIL FROM:<>` and `s.data(...)` returns
the reply to the terminating `.`.

**Probe a known-absent id (attacker guesses an id that does not exist):**

```
$ docker exec -w /workspace sl-app /app/venv/bin/python \
    blitzy_adhoc_test_probe_wire.py 20381 "bounce+10000409+@sl.local" blitzy_adhoc_test_dsn.eml
MAIL  -> 250 OK
RCPT  -> 250 OK
FINAL -> 550 SL E512 No such email log
```

**Probe a real (seeded) id (attacker hits an id that exists):**

```
$ docker exec -w /workspace sl-app /app/venv/bin/python \
    blitzy_adhoc_test_probe_wire.py 20381 "bounce+407+@sl.local" blitzy_adhoc_test_dsn.eml
MAIL  -> 250 OK
RCPT  -> 250 OK
FINAL -> 250 SL E211 Bounce Forward phase handled
```

Same address shape, same crafted body, **only the id differs** — and the reply flips from
`550 SL E512` to `250 SL E211`. That single bit (`5xx` vs `2xx`) is a **yes/no existence oracle** for
internal `email_log` primary keys, harvestable by iterating ids. The daemon's own log records the
transaction from the server's perspective, confirming the null sender is seen literally as `<>`:

```
... _handle() ... New message, mail from <>, rctp tos ['bounce+407+@sl.local']
... _handle() ... Finish mail_from <>, rcpt_tos ['bounce+407+@sl.local'], ... with return code '250 SL E211 Bounce Forward phase handled'<<===
```

Because ids are sequential integers (a SQL primary key; seeded ids here were `407`–`411`, and the
user-supplied `12345` mapped cleanly — see §9), an attacker can enumerate the keyspace linearly and
map out which `email_log` rows exist. The full differential (existence **plus** phase/state) is in
§7–§8.


---

## 5. Q3 — The missing cryptographic gate (what slips through)

### 5.1 The two extraction functions, side by side

**OLD path — `parse_id_from_bounce` (`app/email_utils.py:1258-1259`):**

```python
def parse_id_from_bounce(email_address: str) -> int:
    return int(email_address[email_address.find("+") : email_address.rfind("+")])
```

This is a **pure integer slice** between the first and last `+`. There is **no signature, no HMAC,
no lifetime, no secret** — any attacker-chosen integer is accepted verbatim. Verified:

```
$ ... blitzy_adhoc_test_signed.py   # excerpt
PLAINTEXT_407          407
PLAINTEXT_10000409     10000409
PLAINTEXT_USEREXAMPLE_12345  12345
```

Every input — a seeded id, a guaranteed-absent id, and the user's literal `12345` — is returned
as-is. The function performs **zero validation**.

**NEW path — `get_verp_info_from_email` (`app/email_utils.py:1467-1499`):**

```python
expected_signature = hmac.new(
    config.VERP_EMAIL_SECRET.encode("utf-8"), payload, VERP_HMAC_ALGO   # "sha3-224"
).digest()[:8]
if expected_signature != signature:      # app/email_utils.py:1490
    return None
...
if data[2] > (time.time() + config.VERP_MESSAGE_LIFETIME - VERP_TIME_START) / 60:  # :1496
    return None
return VerpType(data[0]), data[1]
```

The signature is **recomputed with the server secret and compared**; a mismatch returns `None`, and a
5-day lifetime bound (`VERP_MESSAGE_LIFETIME = 5 * 86400`, `app/config.py:499`) is enforced.

### 5.2 Tamper test (runtime proof the gate exists on NEW, absent on OLD)

Using `generate_verp_email(VerpType.bounce_forward, 407)` to mint a **valid** signed address, then
flipping one character:

```
ADDR_VALID              sl.lmycyibuga3syibsgm3tiobthfoq.x57guln4txoz4@sl.local
VERIFY_VALID            (<VerpType.bounce_forward: 0>, 407)     # accepted, id recovered
ADDR_TAMPERED           sl.lmycyibuga3syibsgm3tiobthfoq.x57guln4txoza@sl.local
VERIFY_TAMPERED         None                                    # rejected: HMAC mismatch (:1490)
ADDR_TAMPERED_PAYLOAD   sl.lmycyibuga3syibsgm3tiobthfoa.x57guln4txoz4@sl.local
VERIFY_TAMPERED_PAYLOAD None                                    # rejected: signature no longer matches
ADDR_EXPIRED            sl.lmycyibuga3syibsgm4tembthfoq.asuil4ou7umi2@sl.local
VERIFY_EXPIRED          None                                    # correctly signed but future-dated -> lifetime gate (:1496)
```

The same tamper, observed **on the wire**, shows the security consequence end-to-end. A **valid**
signed address is treated as a bounce (routes via `verp_info[0] == VerpType.bounce_forward`,
`email_handler.py:2061`) and reaches `handle_bounce` → `E211`. A **tampered** signed address yields
`verp_info == None`, matches **no** bounce branch, and falls through to ordinary alias handling,
which returns `550 SL E515 Email not exist` (`app/email/status.py:51`):

```
$ ... probe_wire.py 20381 "sl.lmycyibuga3syibsgm3tiobthfoq.x57guln4txoz4@sl.local" dsn.eml
FINAL -> 250 SL E211 Bounce Forward phase handled          # valid signature -> accepted as bounce

$ ... probe_wire.py 20381 "sl.lmycyibuga3syibsgm3tiobthfoq.x57guln4txoza@sl.local" dsn.eml
FINAL -> 550 SL E515 Email not exist                       # tampered signature -> not a bounce at all
```

### 5.3 What "slips through"

**The boundary that slips through is the total absence of any cryptographic gate on the legacy
branch.** On the NEW path an attacker cannot even reach the `EmailLog.get(id)` existence gate without
a valid HMAC (tampering drops them into alias handling, `E515`). On the OLD path there is **nothing to
tamper** — the attacker writes the id in clear text (`parse_id_from_bounce`), so they walk straight up
to the existence gate and read the oracle. The signed format was clearly introduced to close exactly
this hole, but the plaintext branch remains fully live and reachable
(`email_handler.py:2059-2074`).

---

## 6. Q5 — Bounce-detection criteria & spoofability

### 6.1 What makes the system treat a message as a bounce

`is_bounce` (`email_handler.py:1813-1818`):

```python
def is_bounce(envelope: Envelope, msg: Message):
    """Detect whether an email is a Delivery Status Notification"""
    return (
        envelope.mail_from == "<>"
        and msg.get_content_type().lower() == "multipart/report"
    )
```

Two criteria, **both attacker-controllable**:

1. **Envelope `MAIL FROM:<>`** — the SMTP null reverse-path. A sending client chooses this freely. On
   the real aiosmtpd 1.4.2 server, `MAIL FROM:<>` parses to the literal two-character string `"<>"`
   (`_getaddr("<>") == ("<>", "")`), so `envelope.mail_from == "<>"` is **True** on the wire — the
   daemon log above shows `mail from <>`.
2. **Top-level `Content-Type: multipart/report`** — an ordinary MIME header the sender writes into the
   message body.

Neither criterion involves authentication, DKIM, SPF enforcement, or the server secret, so **any
external sender can satisfy both at will.**

### 6.2 Pass path vs. fail path (observed)

- **Pass** (`is_bounce` true) → `handle_bounce(...)` → `E211`/`E212` (probe rows 2/3, §7).
- **Fail** (`is_bounce` false) → the branch executes `raise VERPForward` / `raise VERPReply`
  (`email_handler.py:2074` / `:2095`), which `MailHandler.handle_DATA` catches
  (`email_handler.py:2308-2318`) and maps to `status.E213`.

Observed with an otherwise-identical probe that only swaps the body to `text/plain`:

```
$ ... probe_harness.py "bounce+407+@sl.local" blitzy_adhoc_test_plain.eml
content_type -> text/plain
HARNESS -> handle() raised VERPForward; handle_DATA maps -> 250 SL E213 Unknown email ignored

$ ... probe_wire.py 20381 "bounce+407+@sl.local" blitzy_adhoc_test_plain.eml
FINAL -> 250 SL E213 Unknown email ignored
```

The harness (Path B) shows the raw control flow — `handle()` **raises** `VERPForward` — while the wire
(Path A) shows the **mapped** `250 SL E213`, exactly as `handle_DATA` produces it. They are two views
of the same behavior.


---

## 7. Q4 — Exact SMTP responses per scenario (verbatim matrix)

### 7.1 The response matrix

Every reply below is the **unedited** status line returned by the running daemon (Path A), confirmed
**stable across at least two runs**. All probes used `MAIL FROM:<>`; the body is the crafted
`multipart/report` DSN (`blitzy_adhoc_test_dsn.eml`) unless the row says otherwise. The status
strings match `app/email/status.py` byte-for-byte — **including the literal typo in `E510`**.

| # | Scenario | Address / input | Producing command (abbrev.) | **Verbatim reply** | Manifesting code |
|---|---|---|---|---|---|
| 1 | Non-existent id | `bounce+10000409+@sl.local` | `probe_wire.py 20381 "bounce+10000409+@sl.local" dsn.eml` | `550 SL E512 No such email log` | gate `email_handler.py:2065-2067`; `app/email/status.py:49` |
| 2 | Existing id, forward phase | `bounce+407+@sl.local` | `probe_wire.py 20381 "bounce+407+@sl.local" dsn.eml` | `250 SL E211 Bounce Forward phase handled` | `email_handler.py:1913-1914`; `app/email/status.py:19` |
| 3 | Existing id, reply phase | `bounce+408+@sl.local` | `probe_wire.py 20381 "bounce+408+@sl.local" dsn.eml` | `250 SL E212 Bounce Reply phase handled` | `email_handler.py:1910-1911`; `app/email/status.py:20` |
| 3b | Reply-VERP branch (prefix) | `bounce_reply+408+@sl.local` | `probe_wire.py 20381 "bounce_reply+408+@sl.local" dsn.eml` | `250 SL E212 Bounce Reply phase handled` | reply branch `email_handler.py:2079-2091` |
| 4 | Existing id, inactive user | `bounce+409+@sl.local` | `probe_wire.py 20381 "bounce+409+@sl.local" dsn.eml` | `550 SL E510 so such user` | `email_handler.py:1869-1871`; `app/email/status.py:47` |
| 5 | Existing id, non-DSN body | `bounce+407+@sl.local` + `text/plain` | `probe_wire.py 20381 "bounce+407+@sl.local" plain.eml` | `250 SL E213 Unknown email ignored` | raise `email_handler.py:2074` → catch `:2308-2318`; `app/email/status.py:21` |
| 6a | Valid **signed** address | `sl.lmycyibuga3syibsgm3tiobthfoq.x57guln4txoz4@sl.local` | `probe_wire.py 20381 "<valid-signed>" dsn.eml` | `250 SL E211 Bounce Forward phase handled` | verified route `email_handler.py:2061`; `handle_bounce :1914` |
| 6b | Tampered **signed** address | `sl.lmycyibuga3syibsgm3tiobthfoq.x57guln4txoza@sl.local` | `probe_wire.py 20381 "<tampered-signed>" dsn.eml` | `550 SL E515 Email not exist` | HMAC reject `app/email_utils.py:1490`; `app/email/status.py:51` |
| 7 | Unexpected error | `bounce++@sl.local` | `probe_wire.py 20381 "bounce++@sl.local" dsn.eml` | `421 SL E404 Unexpected error - Retry later` | `int("+")` `ValueError` in `parse_id_from_bounce` → generic catch `email_handler.py:2319-2332`; `app/email/status.py:32` |

> **`E510` typo — quoted verbatim.** The source literally defines
> `E510 = "550 SL E510 so such user"` (`app/email/status.py:47`) — *"so such user"*, not *"no such
> user"*. The observed reply reproduces the typo exactly; it is **not** corrected here because doing so
> would misrepresent the real behavior.

### 7.2 Path A (wire) vs. Path B (harness) agreement

Both entry paths were run against the same inputs; they agree on every row (Row 5 differs only in
*presentation* — the harness surfaces the raised `VERPForward` that the wire path catches and maps to
`E213`):

```
SCENARIO (addr / input)                | PATH A (wire, handle_DATA)                | PATH B (harness, handle)
---------------------------------------+-------------------------------------------+------------------------------------------
bounce+10000409+@sl.local (dsn)        | 550 SL E512 No such email log             | 550 SL E512 No such email log
bounce+407+@sl.local (dsn)             | 250 SL E211 Bounce Forward phase handled  | 250 SL E211 Bounce Forward phase handled
bounce+408+@sl.local (dsn)             | 250 SL E212 Bounce Reply phase handled    | 250 SL E212 Bounce Reply phase handled
bounce+409+@sl.local (dsn, inactive)   | 550 SL E510 so such user                  | 550 SL E510 so such user
bounce+407+@sl.local (plain)           | 250 SL E213 Unknown email ignored         | handle() raised VERPForward; maps -> 250 SL E213 Unknown email ignored
```

### 7.3 State side-effects (before / during / after)

For the accepted rows the terminal string is not the whole story — a `Bounce` row is logged and
`email_log.bounced` is flipped. Captured with fresh ids `410` (forward) and `411` (reply) via `psql`:

```
BEFORE:  email_log 410.bounced=f, 411.bounced=f ;  bounce table rows = 11
probe bounce+410+@sl.local (dsn) -> 250 SL E211 ;  AFTER: 410.bounced=t ; bounce rows = 12  (+1)
probe bounce+411+@sl.local (dsn) -> 250 SL E212 ;  AFTER: 411.bounced=t ; bounce rows = 13  (+1)
probe bounce+409+@sl.local (dsn) -> 550 SL E510 ;  bounce rows = 13 -> 13 (NO change); 409.bounced=f
```

The forward/reply phases create a `Bounce` row (`handle_bounce_forward_phase`,
`email_handler.py:1449-1454`) and set `email_log.bounced`. The **inactive-user** path returns `E510`
**before** any bounce logging, because `is_active()` is checked at the very top of `handle_bounce`
(`email_handler.py:1869-1871`) — so `E510` is even "cheaper" for an attacker (no persisted side effect
at all).

---

## 8. Q6 — The oracle leaks more than existence

Assembling the full matrix, the responses stratify into a **six-way** classifier of internal state,
not a mere existence bit:

| Observed reply | What the attacker learns about the probed id |
|---|---|
| `550 SL E512 No such email log` | The `email_log` id **does not exist**. |
| `250 SL E211 Bounce Forward phase handled` | Id **exists**, owner **active**, currently in the **forward** phase (`is_reply=False`). |
| `250 SL E212 Bounce Reply phase handled` | Id **exists**, owner **active**, currently in the **reply** phase (`is_reply=True`). |
| `550 SL E510 so such user` | Id **exists**, but the owning **user is inactive** (scheduled for deletion / disabled). |
| `250 SL E213 Unknown email ignored` | Id **exists** (reached the branch) but the message failed `is_bounce`. |
| `550 SL E515 Email not exist` | A **signed** address whose signature did not verify (not treated as a bounce). |

So beyond the coarse `250`-vs-`550` existence signal:

- **Processing phase leaks:** `E211` vs. `E212` discloses whether the mail flow for that id is in the
  forward or reply phase — internal lifecycle state (`EmailLog.get_phase()`, `app/models.py:2143-2147`;
  `is_reply` column `app/models.py:2075`).
- **Account state leaks:** `E510` uniquely flags that a *valid* id belongs to an **inactive** account
  (`User.is_active()`, `app/models.py:766-769`), distinguishing "active user's id" from "pending-
  deletion user's id."
- **Enumeration is stable and side-effect-aware:** responses are identical across repeated probes
  (§7.1), and the cheapest signals (`E512`, `E510`) leave **no** persisted trace, so probing is quiet.

This is precisely an **information-disclosure / enumeration oracle**: the same class as SMTP `RCPT TO`
recipient enumeration, but here it reveals internal `email_log` ids **and** their phase/account state.


---

## 9. The user's example — `bounce+12345+@domain`

The user's literal example maps to the OLD plaintext construction
`BOUNCE_PREFIX + str(email_log.id) + BOUNCE_SUFFIX` (`app/config.py:99-101`). Tracing
`parse_id_from_bounce("bounce+12345+@domain")` (`app/email_utils.py:1258-1259`) index-by-index:

```
address = "bounce+12345+@domain"
           0123456789...
address.find("+")   = 6      # first '+'  (after "bounce")
address.rfind("+")  = 12     # last  '+'  (before "@domain")
address[6:12]       = "+12345"
int("+12345")       = 12345
```

Confirmed at runtime (`PLAINTEXT_USEREXAMPLE_12345  12345`, §5.1). So `bounce+12345+@domain` denotes
**`email_log` id 12345**, extracted with no validation whatsoever.

Because the runtime uses `EMAIL_DOMAIN=sl.local` (`tests/test.env:8`), the corresponding real probe
address is `bounce+12345+@sl.local`. Id `12345` does not exist in the seeded database, so the oracle
returns "absent"; a seeded id returns "present" — the exact enumeration differential the user asked
about, using their own example shape:

```
$ ... python -c "... print(EmailLog.get(12345) is None)"
EmailLog.get(12345) is None -> True

$ ... probe_wire.py 20381 "bounce+12345+@sl.local" blitzy_adhoc_test_dsn.eml   # absent id
FINAL -> 550 SL E512 No such email log        # (stable across 2 runs)

$ ... probe_wire.py 20381 "bounce+407+@sl.local"   blitzy_adhoc_test_dsn.eml   # seeded id
FINAL -> 250 SL E211 Bounce Forward phase handled
```

---

## 10. Security-boundary contrast & observations

### 10.1 Plaintext vs. signed — the boundary in one view

| Aspect | OLD `bounce+{id}+@domain` | NEW `sl.{payload}.{sig}@domain` |
|---|---|---|
| Function | `parse_id_from_bounce` (`app/email_utils.py:1258-1259`) | `get_verp_info_from_email` (`app/email_utils.py:1467-1499`) |
| Id derivation | `int()` slice between first/last `+` | base32-decode signed payload |
| Integrity | **none** | HMAC-SHA3-224 recomputed & compared (`:1490`) |
| Freshness | **none** | 5-day lifetime (`:1496`, `VERP_MESSAGE_LIFETIME` `app/config.py:499`) |
| Attacker forges chosen id? | **yes — trivially** | no (requires `VERP_EMAIL_SECRET`) |
| Reaches existence gate `EmailLog.get`? | **yes** | only with a valid signature |
| Tamper outcome (observed) | n/a (nothing to tamper) | `verp_info=None` → not a bounce → `550 SL E515` |

The routing dispatcher keeps the plaintext branch fully live
(`email_handler.py:2059-2074`), so the presence of the hardened signed format does **not** neutralize
the legacy oracle — both are reachable through the same `handle()` entry.

### 10.2 Impact (observation only — not remediated)

The oracle lets an unauthenticated external sender: (a) confirm existence of internal `email_log`
ids by linear enumeration; (b) classify each existing id by processing phase (`E211`/`E212`) and by
owner account state (`E510`); and (c) do so quietly, since the cheapest signals persist no state
(§7.3). Per the task's read-only constraint, **no remediation was implemented**. For completeness (and
strictly as an observation, not a change made here), the standard mitigations for this vulnerability
class are: retire/authenticate the plaintext branch (require the signed HMAC format for all inbound
bounces), and **unify** the responses so that existent/non-existent/phase/state cases return an
indistinguishable status — collapsing the differential the attacker relies on.

---

## 11. Appendix — reproducibility

### 11.1 Canonical commands

```bash
# Runtime = provided Docker image; all code runs inside container "sl-app" against PostgreSQL 13.
# Interpreter is the container venv (host python lacks the pinned deps):
INTERP=/app/venv/bin/python
ENVV="CONFIG=/workspace/tests/test.env GITHUB_ACTIONS_TEST=true"

# 1) Verify runtime + VERP secret length (>=32 guard, app/config.py:505-508)
docker exec -e CONFIG=/workspace/tests/test.env -e GITHUB_ACTIONS_TEST=true -w /workspace \
  sl-app $INTERP -c "from app import config; print(len(config.VERP_EMAIL_SECRET))"      # -> 36

# 2) Start the CANONICAL entry point (aiosmtpd Controller, email_handler.py:2381-2383)
docker exec -d -e CONFIG=/workspace/tests/test.env -e GITHUB_ACTIONS_TEST=true -w /workspace \
  sl-app bash -lc "$INTERP email_handler.py -p 20381"     # log: "Start mail controller 0.0.0.0 20381"

# 3) Path A — raw SMTP client probe (MAIL FROM:<>, RCPT TO:<addr>, DATA); FINAL line = wire reply
docker exec -w /workspace sl-app $INTERP blitzy_adhoc_test_probe_wire.py 20381 "<addr>" <eml>

# 4) Path B — module-level handler harness (mirrors tests/test_email_handler.py:81-85)
docker exec -e CONFIG=/workspace/tests/test.env -e GITHUB_ACTIONS_TEST=true -w /workspace \
  sl-app $INTERP blitzy_adhoc_test_probe_harness.py "<addr>" <eml>
```

### 11.2 Seeded ids

| Role | id | Notes |
|---|---|---|
| Forward-phase, active | `407` | `is_reply=False`, user 807 active |
| Reply-phase, active | `408` | `is_reply=True`, user 807 active |
| Inactive user | `409` | user 808 `delete_on` future ⇒ `is_active()==False` |
| Known-absent | `10000409` | `EmailLog.get(10000409) is None` |
| Forward state-demo (fresh) | `410` | `bounced` f→t, +1 `Bounce` row |
| Reply state-demo (fresh) | `411` | `bounced` f→t, +1 `Bounce` row |
| Valid signed | `sl.lmycyibuga3syibsgm3tiobthfoq.x57guln4txoz4@sl.local` | `generate_verp_email(bounce_forward, 407)` |

### 11.3 Stability

Every probe in §7 was executed **at least twice** and returned an **identical** reply each time; the
tables above report the stable value. (Representative example: `bounce+10000409+@sl.local` returned
`550 SL E512 No such email log` on both run 1 and run 2.)

### 11.4 Read-only guarantee

The investigation created only **ephemeral** artifacts (all prefixed `blitzy_adhoc_test_`: the seed
scripts, the crafted `.eml` inputs, the two probe scripts, and the daemon log/pid). These were
**deleted** before completion, and the running daemon was stopped. The only file added to the
repository is this document. The final working-tree state confirms it:

```
$ git status --porcelain
?? blitzy/

$ git status --porcelain --untracked-files=all
?? blitzy/documentation/app_2cd6ee777f8c.md
```

The default `--porcelain` collapses the untracked directory to `?? blitzy/`; expanding it with
`--untracked-files=all` shows the single added file. No entry with a modified/deleted marker
(` M`, `MM`, ` D`, `D `) appears for any tracked path — verified separately:

```
$ git status --porcelain | grep -E '^ ?M|^MM|^ D|^D '
(no output — no tracked file modified or deleted)
```

No tracked source file was modified, created, or deleted — consistent with the user directive
*"Don't modify any source files in the repository."* The only path added to the repository is
`blitzy/documentation/app_2cd6ee777f8c.md`.

### 11.5 Key `file:line` reference index

| Symbol / behavior | Location |
|---|---|
| `is_bounce` (detection criteria) | `email_handler.py:1813-1818` |
| `handle` (routing) | `email_handler.py:1945`; forward branch `:2059-2074`; reply branch `:2079-2095` |
| existence gate → `E512` | `email_handler.py:2065-2067` / `:2087` |
| `handle_bounce` (E510/E212/E211) | `email_handler.py:1851-1914` (E510 `:1869-1871`, E212 `:1910-1911`, E211 `:1913-1914`) |
| `MailHandler.handle_DATA` (return = wire reply) | `email_handler.py:2289-2293` |
| exception → status mapping | VERP* → `E213` `:2308-2318`; generic → `E404` `:2319-2332` |
| 5XX→`E216` SPF rewrite (controlled for) | `email_handler.py:2356-2365` |
| `main` / `Controller` / default port | `email_handler.py:2381-2383` / `:2399` |
| `parse_id_from_bounce` (plaintext, no crypto) | `app/email_utils.py:1258-1259` |
| `generate_verp_email` (signed) | `app/email_utils.py:1438-1464` |
| `get_verp_info_from_email` (HMAC + lifetime) | `app/email_utils.py:1467-1499` (compare `:1490`, lifetime `:1496`) |
| `BOUNCE_PREFIX` / `BOUNCE_SUFFIX` / reply prefix | `app/config.py:100` / `:101` / `:108-110` |
| `VERP_MESSAGE_LIFETIME` / `VERP_PREFIX` / secret guard | `app/config.py:499` / `:500` / `:505-508` |
| status strings (`E211/E212/E213/E404/E510/E512/E515`) | `app/email/status.py:19,20,21,32,47,49,51` |
| `VerpType` enum | `app/models.py:247-250` |
| `EmailLog.is_reply` / `get_phase` / `User.is_active` | `app/models.py:2075` / `:2143-2147` / `:766-769` |
| `VERPForward` / `VERPReply` / `VERPTransactional` | `app/errors.py:42-57` |

---

*All evidence in this document was produced by executing the real code paths inside the canonical
Docker runtime; status strings are quoted byte-for-byte from `app/email/status.py` and from captured
SMTP replies. No repository source file was modified.*

