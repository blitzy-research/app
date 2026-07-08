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
| Timestamp check | **none** | future-dating clamp: rejects a timestamp **more than 5 days in the future** (it does **not** expire old addresses) |
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
| **State side-effect (fresh)** | `414` / `415` | user 810 active; forward / reply, both `bounced=False` initially | `bounce+414+@sl.local` / `bounce+415+@sl.local` |

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
   $ docker exec -w /workspace sl-app /app/venv/bin/python blitzy_adhoc_test_probe_wire.py 20381 "bounce+408+@sl.local" blitzy_adhoc_test_dsn.eml
   FINAL -> 250 SL E212 Bounce Reply phase handled
   $ docker exec -w /workspace sl-app /app/venv/bin/python blitzy_adhoc_test_probe_wire.py 20381 "bounce_reply+408+@sl.local" blitzy_adhoc_test_dsn.eml
   FINAL -> 250 SL E212 Bounce Reply phase handled
   ```
   (full MAIL/RCPT/FINAL transcripts for these are rows 3 and 3b in §7.1.1.)

2. **Both formats converge on `EmailLog.get(id)`**, so both are gated identically by id existence
   (`E512` when absent). The only difference is *how the id is obtained* — plaintext slice vs. verified
   signature — which is the crux of Q3.

### 3.3 Routing flow diagram

```mermaid
flowchart TD
    A[Inbound message: handle_DATA] --> B{Recipient shape?}
    B -->|"old: bounce+ID+@domain"| C["parse_id_from_bounce()<br/>plaintext slice, NO HMAC"]
    B -->|"new: sl.payload.sig@domain"| D["get_verp_info_from_email()<br/>HMAC-SHA3-224 + future-dating clamp"]
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

Because ids are sequential integers (a SQL primary key; seeded ids here were `407`–`415`, and the
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
no lifetime, no secret** — any attacker-chosen integer is accepted verbatim. Verified against the
real function (the same `parse_id_from_bounce` the OLD routing branch calls at
`email_handler.py:2062`):

```
$ docker exec -e CONFIG=/workspace/tests/test.env -e GITHUB_ACTIONS_TEST=true -w /workspace \
    sl-app /app/venv/bin/python -c "
from app.email_utils import parse_id_from_bounce
for a in ('bounce+407+@sl.local', 'bounce+10000409+@sl.local', 'bounce+12345+@domain'):
    print('PLAINTEXT %-28s -> %d' % (a, parse_id_from_bounce(a)))
"
[... app config-load banner (timestamp/PID/GNUPGHOME tempdir vary per run) elided ...]
PLAINTEXT bounce+407+@sl.local         -> 407
PLAINTEXT bounce+10000409+@sl.local    -> 10000409
PLAINTEXT bounce+12345+@domain         -> 12345
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

The signature is **recomputed with the server secret and compared**; a mismatch returns `None`. In
addition, the timestamp field `data[2]` (encoded in minutes at generation time,
`app/email_utils.py:1449`) is checked at `app/email_utils.py:1496`:

```python
# app/email_utils.py:1496  — VERP_MESSAGE_LIFETIME = 5*86400 (app/config.py:499); VERP_TIME_START = 1640995200 (:68)
if data[2] > (time.time() + config.VERP_MESSAGE_LIFETIME - VERP_TIME_START) / 60:
    return None
```

**Precise semantics of this check (runtime-verified, see §5.2).** The right-hand side equals
"now, in minutes, **plus** `VERP_MESSAGE_LIFETIME` (5 days)". The condition therefore rejects only a
payload whose timestamp is **more than 5 days in the future** — a future-dating clamp. It is **not**
an expiry/freshness check: an address whose timestamp is *old* (even years old) is **not** rejected,
because an old `data[2]` is *less than* the threshold. This was confirmed at runtime:

```
$ docker exec -e CONFIG=/workspace/tests/test.env -e GITHUB_ACTIONS_TEST=true -w /workspace \
    sl-app /app/venv/bin/python -c "
import time
from app import config
START=1640995200; now_min=(time.time()-START)/60
thr=(time.time()+config.VERP_MESSAGE_LIFETIME-START)/60
print('reject-if data2 >', round(thr), '(= now_min', round(now_min), '+ 7200 min = 5 days ahead)')
print('fresh   (data2=now_min)        rejected?', now_min          > thr)
print('OLD     (data2=now_min-14400)  rejected?', (now_min-14400)  > thr, ' <- old addresses PASS')
print('FUTURE  (data2=now_min+14400)  rejected?', (now_min+14400)  > thr, ' <- only >5d-future rejected')
"
reject-if data2 > 2382099 (= now_min 2374899 + 7200 min = 5 days ahead)
fresh   (data2=now_min)        rejected? False
OLD     (data2=now_min-14400)  rejected? False  <- old addresses PASS
FUTURE  (data2=now_min+14400)  rejected? True  <- only >5d-future rejected
```

> The `now_min`/threshold integers advance ~1 per minute (they are `time.time()`-derived), so an
> exact re-run shows slightly larger numbers; the **boolean verdicts are invariant** — old addresses
> always pass, only a timestamp >5 days in the future is rejected.

### 5.2 Tamper & future-dating test (runtime proof the gate exists on NEW, absent on OLD)

`blitzy_adhoc_test_signed.py` (full source in Appendix §11.6) mints a **valid** signed address with
`generate_verp_email(VerpType.bounce_forward, 407)`, then derives three broken variants and verifies
each with the real `get_verp_info_from_email`. Tampering flips the **first** base32 character of the
target segment (the trailing char can hold only padding bits, so flipping it may decode to identical
bytes — the first char always carries meaningful bits). The future-dated variant is minted by
patching `app.email_utils.time.time` to `now + 10 days` **during generation only**, so it is
*correctly signed* over a future timestamp; verification runs on the real, unmocked clock.

```
$ docker exec -e CONFIG=/workspace/tests/test.env -e GITHUB_ACTIONS_TEST=true -w /workspace \
    sl-app /app/venv/bin/python blitzy_adhoc_test_signed.py 407 10000409
[... app config-load banner + the "=== OLD plaintext parse_id_from_bounce (NO crypto) ==="
     section (its three PLAINTEXT lines are shown verbatim in §5.1) elided ...]
=== NEW signed generate_verp_email + get_verp_info_from_email ===
ADDR_VALID               sl.lmycyibuga3syibsgm3tiojqgfoq.pqexaworaxcai@sl.local
VERIFY_VALID             (<VerpType.bounce_forward: 0>, 407)    # accepted, id recovered
ADDR_TAMPERED_SIG        sl.lmycyibuga3syibsgm3tiojqgfoq.aqexaworaxcai@sl.local
VERIFY_TAMPERED_SIG      None                                   # rejected: HMAC mismatch (:1490)
ADDR_TAMPERED_PAYLOAD    sl.amycyibuga3syibsgm3tiojqgfoq.pqexaworaxcai@sl.local
VERIFY_TAMPERED_PAYLOAD  None                                   # rejected: signature no longer matches
ADDR_FUTUREDATED         sl.lmycyibuga3syibsgm4dsmzqgfoq.332emhuur3iau@sl.local
VERIFY_FUTUREDATED       None                                   # correctly signed but future-dated -> clamp (:1496)
  future payload_minute = 2389301 > reject-above = 2382102 (now_min = 2374902 )
```

The `pqexaworaxcai` payload/signature bytes are **time-dependent** (the payload encodes the current
minute), so re-running `generate_verp_email` produces a different byte string with identical
verify/route behavior; the exact address above is the one captured for this run's wire probes below.

**Observed on the wire, end-to-end (Path A).** A **valid** signed address is treated as a bounce
(routes via `verp_info[0] == VerpType.bounce_forward`, `email_handler.py:2061`) and reaches
`handle_bounce` → `E211`. A **tampered** signed address (signature *or* payload) and a **future-dated**
signed address both yield `verp_info == None`, match **no** bounce branch, and fall through to
ordinary alias handling, returning `550 SL E515 Email not exist` (`app/email/status.py:51`). All three
were confirmed stable across two runs:

```
$ docker exec -w /workspace sl-app /app/venv/bin/python \
    blitzy_adhoc_test_probe_wire.py 20381 "sl.lmycyibuga3syibsgm3tiojqgfoq.pqexaworaxcai@sl.local" blitzy_adhoc_test_dsn.eml
FINAL -> 250 SL E211 Bounce Forward phase handled          # valid signature -> accepted as bounce

$ docker exec -w /workspace sl-app /app/venv/bin/python \
    blitzy_adhoc_test_probe_wire.py 20381 "sl.lmycyibuga3syibsgm3tiojqgfoq.aqexaworaxcai@sl.local" blitzy_adhoc_test_dsn.eml
FINAL -> 550 SL E515 Email not exist                       # tampered signature -> not a bounce at all

$ docker exec -w /workspace sl-app /app/venv/bin/python \
    blitzy_adhoc_test_probe_wire.py 20381 "sl.lmycyibuga3syibsgm4dsmzqgfoq.332emhuur3iau@sl.local" blitzy_adhoc_test_dsn.eml
FINAL -> 550 SL E515 Email not exist                       # future-dated (>5d ahead) -> clamp (:1496) -> not a bounce
```

The **future-dated** case is the runtime proof of the lifetime clamp on the *real SMTP entry path*:
the address carries a valid HMAC, so it clears the signature check (`:1490`); it is rejected **solely**
by the future-dating condition at `:1496`, and that rejection is observable on the wire as the same
`550 SL E515` a tampered signature produces (because both make `get_verp_info_from_email` return
`None`). Note this is **not** an expiry of old addresses — as §5.1 shows, an old timestamp passes.

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
$ docker exec -e CONFIG=/workspace/tests/test.env -e GITHUB_ACTIONS_TEST=true -w /workspace \
    sl-app /app/venv/bin/python blitzy_adhoc_test_probe_harness.py "bounce+407+@sl.local" blitzy_adhoc_test_plain.eml
content_type -> text/plain
HARNESS -> handle() raised VERPForward

$ docker exec -w /workspace sl-app /app/venv/bin/python \
    blitzy_adhoc_test_probe_wire.py 20381 "bounce+407+@sl.local" blitzy_adhoc_test_plain.eml
FINAL -> 250 SL E213 Unknown email ignored
```

The harness (Path B) shows the raw control flow — `handle()` **raises** `VERPForward` (the last line
above is the script's verbatim output) — while the wire (Path A) shows the **mapped** `250 SL E213`
that `MailHandler.handle_DATA` produces from that same raised exception (`email_handler.py:2308-2318`).
They are two views of the same behavior.


---

## 7. Q4 — Exact SMTP responses per scenario (verbatim matrix)

### 7.1 The response matrix

Every reply below is the **unedited** status line returned by the running daemon (Path A). Each probe
was run **twice** and returned an identical reply both times (per-probe two-run transcripts in §11.3).
All probes used `MAIL FROM:<>`; the body is the crafted `multipart/report` DSN
(`blitzy_adhoc_test_dsn.eml`) unless the row says otherwise. The status strings match
`app/email/status.py` byte-for-byte — **including the literal typo in `E510`**. The **exact producing
command** for every row (no abbreviation, no placeholder) with its full unedited transcript follows in
§7.1.1.

| # | Scenario | Address / input | **Verbatim reply** | Manifesting code |
|---|---|---|---|---|
| 1 | Non-existent id | `bounce+10000409+@sl.local` + DSN | `550 SL E512 No such email log` | gate `email_handler.py:2065-2067`; `app/email/status.py:49` |
| 2 | Existing id, forward phase | `bounce+407+@sl.local` + DSN | `250 SL E211 Bounce Forward phase handled` | `email_handler.py:1913-1914`; `app/email/status.py:19` |
| 3 | Existing id, reply phase | `bounce+408+@sl.local` + DSN | `250 SL E212 Bounce Reply phase handled` | `email_handler.py:1910-1911`; `app/email/status.py:20` |
| 3b | Reply-VERP branch (prefix) | `bounce_reply+408+@sl.local` + DSN | `250 SL E212 Bounce Reply phase handled` | reply branch `email_handler.py:2079-2091` |
| 4 | Existing id, inactive user | `bounce+409+@sl.local` + DSN | `550 SL E510 so such user` | `email_handler.py:1869-1871`; `app/email/status.py:47` |
| 5 | Existing id, non-DSN body | `bounce+407+@sl.local` + `text/plain` | `250 SL E213 Unknown email ignored` | raise `email_handler.py:2074` → catch `:2308-2318`; `app/email/status.py:21` |
| 6a | Valid **signed** address | `sl.lmycyibuga3syibsgm3tiojqgfoq.pqexaworaxcai@sl.local` + DSN | `250 SL E211 Bounce Forward phase handled` | verified route `email_handler.py:2061`; `handle_bounce :1914` |
| 6b | Tampered **signed** signature | `sl.lmycyibuga3syibsgm3tiojqgfoq.aqexaworaxcai@sl.local` + DSN | `550 SL E515 Email not exist` | HMAC reject `app/email_utils.py:1490`; `app/email/status.py:51` |
| 6c | Future-dated **signed** address | `sl.lmycyibuga3syibsgm4dsmzqgfoq.332emhuur3iau@sl.local` + DSN | `550 SL E515 Email not exist` | future-dating clamp `app/email_utils.py:1496`; `app/email/status.py:51` |
| 7 | Unexpected error | `bounce++@sl.local` + DSN | `421 SL E404 Unexpected error - Retry later` | `int("+")` `ValueError` in `parse_id_from_bounce` → generic catch `email_handler.py:2319-2332`; `app/email/status.py:32` |

#### 7.1.1 Exact producing command + unedited transcript per row

Every wire probe shares the identical, fully-literal prefix
`docker exec -w /workspace sl-app /app/venv/bin/python blitzy_adhoc_test_probe_wire.py 20381`
(the daemon was started with `… email_handler.py -p 20381`, §2.3). Below is the complete command for
each row and the verbatim reply captured (source of the probe script is inlined in §11.6; the DSN /
plain `.eml` bodies are inlined in §11.7):

```
# Row 1 — non-existent id
$ docker exec -w /workspace sl-app /app/venv/bin/python blitzy_adhoc_test_probe_wire.py 20381 "bounce+10000409+@sl.local" blitzy_adhoc_test_dsn.eml
MAIL  -> 250 OK
RCPT  -> 250 OK
FINAL -> 550 SL E512 No such email log

# Row 2 — existing id, forward phase
$ docker exec -w /workspace sl-app /app/venv/bin/python blitzy_adhoc_test_probe_wire.py 20381 "bounce+407+@sl.local" blitzy_adhoc_test_dsn.eml
MAIL  -> 250 OK
RCPT  -> 250 OK
FINAL -> 250 SL E211 Bounce Forward phase handled

# Row 3 — existing id, reply phase (forward-prefix address; phase keyed on stored is_reply)
$ docker exec -w /workspace sl-app /app/venv/bin/python blitzy_adhoc_test_probe_wire.py 20381 "bounce+408+@sl.local" blitzy_adhoc_test_dsn.eml
MAIL  -> 250 OK
RCPT  -> 250 OK
FINAL -> 250 SL E212 Bounce Reply phase handled

# Row 3b — reply-VERP branch via the bounce_reply+ prefix
$ docker exec -w /workspace sl-app /app/venv/bin/python blitzy_adhoc_test_probe_wire.py 20381 "bounce_reply+408+@sl.local" blitzy_adhoc_test_dsn.eml
MAIL  -> 250 OK
RCPT  -> 250 OK
FINAL -> 250 SL E212 Bounce Reply phase handled

# Row 4 — existing id, inactive user
$ docker exec -w /workspace sl-app /app/venv/bin/python blitzy_adhoc_test_probe_wire.py 20381 "bounce+409+@sl.local" blitzy_adhoc_test_dsn.eml
MAIL  -> 250 OK
RCPT  -> 250 OK
FINAL -> 550 SL E510 so such user

# Row 5 — existing id, non-DSN body (text/plain) fails is_bounce
$ docker exec -w /workspace sl-app /app/venv/bin/python blitzy_adhoc_test_probe_wire.py 20381 "bounce+407+@sl.local" blitzy_adhoc_test_plain.eml
MAIL  -> 250 OK
RCPT  -> 250 OK
FINAL -> 250 SL E213 Unknown email ignored

# Row 6a — valid signed address (generated per §5.2)
$ docker exec -w /workspace sl-app /app/venv/bin/python blitzy_adhoc_test_probe_wire.py 20381 "sl.lmycyibuga3syibsgm3tiojqgfoq.pqexaworaxcai@sl.local" blitzy_adhoc_test_dsn.eml
MAIL  -> 250 OK
RCPT  -> 250 OK
FINAL -> 250 SL E211 Bounce Forward phase handled

# Row 6b — tampered signature (first sig char flipped, §5.2)
$ docker exec -w /workspace sl-app /app/venv/bin/python blitzy_adhoc_test_probe_wire.py 20381 "sl.lmycyibuga3syibsgm3tiojqgfoq.aqexaworaxcai@sl.local" blitzy_adhoc_test_dsn.eml
MAIL  -> 250 OK
RCPT  -> 250 OK
FINAL -> 550 SL E515 Email not exist

# Row 6c — future-dated signed address (>5d ahead; rejected by clamp :1496, §5.2)
$ docker exec -w /workspace sl-app /app/venv/bin/python blitzy_adhoc_test_probe_wire.py 20381 "sl.lmycyibuga3syibsgm4dsmzqgfoq.332emhuur3iau@sl.local" blitzy_adhoc_test_dsn.eml
MAIL  -> 250 OK
RCPT  -> 250 OK
FINAL -> 550 SL E515 Email not exist

# Row 7 — unexpected error: parse_id_from_bounce("bounce++@sl.local") does int("+") -> ValueError
$ docker exec -w /workspace sl-app /app/venv/bin/python blitzy_adhoc_test_probe_wire.py 20381 "bounce++@sl.local" blitzy_adhoc_test_dsn.eml
MAIL  -> 250 OK
RCPT  -> 250 OK
FINAL -> 421 SL E404 Unexpected error - Retry later
```

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

Signed-format rows (the exact §5.2 address set) likewise agree on both paths — the valid address
verifies and routes to the forward bounce, while the tampered (first sig char flipped) and
future-dated (>5 days ahead) addresses both fail `get_verp_info_from_email` and fall through to
`550 SL E515` (`A:` = Path A wire, `B:` = Path B harness):

```
sl.lmycyibuga3syibsgm3tiojqgfoq.pqexaworaxcai@sl.local  (valid)    | A: 250 SL E211 Bounce Forward phase handled | B: 250 SL E211 Bounce Forward phase handled
sl.lmycyibuga3syibsgm3tiojqgfoq.aqexaworaxcai@sl.local  (tampered) | A: 550 SL E515 Email not exist              | B: 550 SL E515 Email not exist
sl.lmycyibuga3syibsgm4dsmzqgfoq.332emhuur3iau@sl.local  (future)   | A: 550 SL E515 Email not exist              | B: 550 SL E515 Email not exist
```

### 7.3 State side-effects (before / during / after)

For the accepted rows the terminal string is not the whole story — a `Bounce` row is logged and
`email_log.bounced` is flipped. This was captured with three exact inspection commands against the
same PostgreSQL the daemon writes to, using the **fresh, pristine** ids `414` (forward) and `415`
(reply) — both `bounced=False` at the start (§2.5) — plus inactive id `409`. The inspection commands
(run verbatim, `psql -tAc` = tuples-only/unaligned so each row is `id|value`):

```bash
# (S) email_log.bounced snapshot for the three ids
docker exec -w /workspace sl-app bash -lc \
  'PGPASSWORD=test psql -h localhost -p 15432 -U test -d test -tAc \
   "SELECT id, bounced FROM email_log WHERE id IN (409,414,415) ORDER BY id;"'
# (C) bounce-table row count
docker exec -w /workspace sl-app bash -lc \
  'PGPASSWORD=test psql -h localhost -p 15432 -U test -d test -tAc "SELECT count(*) FROM bounce;"'
# (N) newest bounce row (its keyed email reveals which code path created it)
docker exec -w /workspace sl-app bash -lc \
  'PGPASSWORD=test psql -h localhost -p 15432 -U test -d test -tAc \
   "SELECT id, email FROM bounce ORDER BY id DESC LIMIT 1;"'
# (P) the state-changing probe itself (Path A wire); run once per id below with the concrete
#     address shown in the transcript — "bounce+414+@sl.local", "bounce+415+@sl.local",
#     "bounce+409+@sl.local" (each is the exact string passed as argv[2]):
docker exec -w /workspace sl-app /app/venv/bin/python \
  blitzy_adhoc_test_probe_wire.py 20381 "bounce+414+@sl.local" blitzy_adhoc_test_dsn.eml
```

Unedited outputs, captured **atomically** in one before → during → after sequence. Each `$ (X)` line
re-runs command `(X)` from the block above; the line(s) directly beneath it are that command's
**verbatim stdout** (no annotation added inside the output — the deltas are called out in the prose
that follows):

```
############ BEFORE (no probes yet) ############
$ (S)
409|f
414|f
415|f
$ (C)
47
$ (N)
47|user_1io4qxbmot@mailbox.test

############ PROBE 414 — forward ############
$ (P) addr="bounce+414+@sl.local"
FINAL -> 250 SL E211 Bounce Forward phase handled
$ (S)
409|f
414|t
415|f
$ (C)
48
$ (N)
48|user_fjvnjvopyf@mailbox.test

############ PROBE 415 — reply ############
$ (P) addr="bounce+415+@sl.local"
FINAL -> 250 SL E212 Bounce Reply phase handled
$ (S)
409|f
414|t
415|t
$ (C)
49
$ (N)
49|contact@example.com

############ PROBE 409 — inactive user ############
$ (P) addr="bounce+409+@sl.local"
FINAL -> 550 SL E510 so such user
$ (S)
409|f
414|t
415|t
$ (C)
49
```

Reading the raw stdout above: `(S)` shows `email_log.bounced` for the three ids as `id|bounced`; `(C)`
is the total `bounce` row count; `(N)` is `id|email` of the newest `bounce` row.

Two independent side effects are proven per accepted bounce: `email_log.bounced` flips `f`→`t`, **and**
one `Bounce` row is appended (count `47`→`48`→`49`). Crucially, the **keyed email of the new row
differs by phase** — `user_fjvnjvopyf@mailbox.test` (the recipient **mailbox** email) for the forward
probe versus `contact@example.com` (the reply **contact**'s `website_email`, set by the seed script in §11.6) for the reply probe.
That difference is the direct runtime fingerprint of **two separate `Bounce.create` call sites**:

- **Forward phase** — `handle_bounce_forward_phase` creates the row with `email=mailbox.email`
  (`email_handler.py:1449` and, when no bounce-info is parsed, `:1454`; block `:1447-1454`).
- **Reply phase** — `handle_bounce_reply_phase` creates the row with
  `email=sanitize_email(contact.website_email, …)` (`email_handler.py:1609` and `:1616`; block
  `:1607-1618`).

The **inactive-user** path returns `E510` **before** any bounce logging, because `is_active()` is
checked at the very top of `handle_bounce` (`email_handler.py:1869-1871`) — so `E510` persists **no**
side effect at all (count and `409.bounced` both unchanged above), making it the "cheapest" signal for
an attacker.

The `bounced` flip is **one-shot per row** (it is a boolean already set to `t` afterwards); to
reproduce the `f`→`t` transition, seed a fresh id pair with the §11.6 seed script and probe those.
Re-probing an *already-bounced* id still returns the identical reply and still appends a `Bounce` row
(the count keeps incrementing — this is why the two-run stability capture in §11.3, taken after the
snapshot above, advances the count further), only `email_log.bounced` is already `t`.

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
| `550 SL E515 Email not exist` | A **signed** address that failed `get_verp_info_from_email` — a bad HMAC **or** a timestamp outside the allowed window (future-dated) — so it was not treated as a bounce (§5.2). |

So beyond the coarse `250`-vs-`550` existence signal:

- **Processing phase leaks:** `E211` vs. `E212` discloses whether the mail flow for that id is in the
  forward or reply phase — internal lifecycle state (`EmailLog.get_phase()`, `app/models.py:2143-2147`;
  `is_reply` column `app/models.py:2075`).
- **Account state leaks:** `E510` uniquely flags that a *valid* id belongs to an **inactive** account
  (`User.is_active()`, `app/models.py:766-769`), distinguishing "active user's id" from "pending-
  deletion user's id."
- **Enumeration is stable and side-effect-aware:** responses are identical across repeated probes
  (per-probe two-run evidence in §11.3), and the cheapest signals (`E512`, `E510`) leave **no**
  persisted trace (§7.3), so probing is quiet.

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

Confirmed at runtime (§5.1 shows the line `PLAINTEXT bounce+12345+@domain         -> 12345`). So
`bounce+12345+@domain` denotes **`email_log` id 12345**, extracted with no validation whatsoever.

Because the runtime uses `EMAIL_DOMAIN=sl.local` (`tests/test.env:8`), the corresponding real probe
address is `bounce+12345+@sl.local`. Id `12345` does not exist in the seeded database, so the oracle
returns "absent"; a seeded id returns "present" — the exact enumeration differential the user asked
about, using their own example shape:

```
$ docker exec -e CONFIG=/workspace/tests/test.env -e GITHUB_ACTIONS_TEST=true -w /workspace \
    sl-app /app/venv/bin/python -c "
from server import create_light_app
from app.models import EmailLog
with create_light_app().app_context():
    print('EmailLog.get(12345) is None ->', EmailLog.get(12345) is None)
    print('EmailLog.get(407) is None   ->', EmailLog.get(407) is None)
"
[... app config-load banner (timestamp/PID/GNUPGHOME tempdir vary per run) elided ...]
EmailLog.get(12345) is None -> True
EmailLog.get(407) is None   -> False

# absent id (the user's example shape) -> existence gate returns E512
$ docker exec -w /workspace sl-app /app/venv/bin/python \
    blitzy_adhoc_test_probe_wire.py 20381 "bounce+12345+@sl.local" blitzy_adhoc_test_dsn.eml
FINAL -> 550 SL E512 No such email log

# seeded id 407 -> accepted bounce (two-run stability for both in §11.3)
$ docker exec -w /workspace sl-app /app/venv/bin/python \
    blitzy_adhoc_test_probe_wire.py 20381 "bounce+407+@sl.local" blitzy_adhoc_test_dsn.eml
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
| Timestamp bound | **none** | future-dating clamp (`:1496`): rejects a timestamp **>5 days in the future**; does **not** expire old addresses (runtime-proven §5.2) — `VERP_MESSAGE_LIFETIME=5d` `app/config.py:499` |
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

Steps 3 and 4 below show the **command shape**; the tokens `<addr>` and `<eml>` are metavariables.
The **fully-literal, per-row invocations** (with the concrete address and `.eml` filename substituted)
are given for every matrix row in §7.1.1, the probe-script sources are inlined in §11.6, and the two
crafted `.eml` bodies are inlined in §11.7 — so every transcript is reproducible from this document
alone, with no reliance on the (deleted) ephemeral files.

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
| Forward state-demo (fresh) | `414` | user 810 active; `bounced` f→t, +1 `Bounce` row (§7.3) |
| Reply state-demo (fresh) | `415` | user 810 active; `bounced` f→t, +1 `Bounce` row (§7.3) |
| Valid signed | `sl.lmycyibuga3syibsgm3tiojqgfoq.pqexaworaxcai@sl.local` | `generate_verp_email(bounce_forward, 407)`; **time-dependent** — the payload encodes the generation minute, so a fresh run emits a different (equally valid) string that still verifies to `(bounce_forward, 407)` (§5.2) |

### 11.3 Stability

Every matrix row was executed **twice**, back-to-back, via the identical §7.1.1 wire command
(`… blitzy_adhoc_test_probe_wire.py 20381 "<addr>" <eml>`). Both runs returned a byte-identical
`FINAL` line for **every** row — no flakiness, no ordering dependence. The per-probe run-1/run-2
`FINAL` lines are shown in full below (not merely asserted):

| # | Address / input | Run 1 `FINAL` | Run 2 `FINAL` |
|---|---|---|---|
| 1 | `bounce+10000409+@sl.local` (DSN) | `550 SL E512 No such email log` | `550 SL E512 No such email log` |
| 2 | `bounce+407+@sl.local` (DSN) | `250 SL E211 Bounce Forward phase handled` | `250 SL E211 Bounce Forward phase handled` |
| 3 | `bounce+408+@sl.local` (DSN) | `250 SL E212 Bounce Reply phase handled` | `250 SL E212 Bounce Reply phase handled` |
| 3b | `bounce_reply+408+@sl.local` (DSN) | `250 SL E212 Bounce Reply phase handled` | `250 SL E212 Bounce Reply phase handled` |
| 4 | `bounce+409+@sl.local` (DSN) | `550 SL E510 so such user` | `550 SL E510 so such user` |
| 5 | `bounce+407+@sl.local` (`text/plain`) | `250 SL E213 Unknown email ignored` | `250 SL E213 Unknown email ignored` |
| 6a | `sl.lmycyibuga3syibsgm3tiojqgfoq.pqexaworaxcai@sl.local` (DSN) | `250 SL E211 Bounce Forward phase handled` | `250 SL E211 Bounce Forward phase handled` |
| 6b | `sl.lmycyibuga3syibsgm3tiojqgfoq.aqexaworaxcai@sl.local` (DSN) | `550 SL E515 Email not exist` | `550 SL E515 Email not exist` |
| 6c | `sl.lmycyibuga3syibsgm4dsmzqgfoq.332emhuur3iau@sl.local` (DSN) | `550 SL E515 Email not exist` | `550 SL E515 Email not exist` |
| 7 | `bounce++@sl.local` (DSN) | `421 SL E404 Unexpected error - Retry later` | `421 SL E404 Unexpected error - Retry later` |

All ten rows: **Run 1 ≡ Run 2** (stable). The signed rows (6a–6c) use the exact §5.2 address set; that
set is time-dependent at *generation*, but for a *fixed* address the verify/route outcome is invariant
(a valid signature always routes, a tampered/future one always falls through to `E515`).

### 11.4 Read-only guarantee

The investigation created only **ephemeral** artifacts (all prefixed `blitzy_adhoc_test_`: the seed
scripts, the crafted `.eml` inputs, the probe scripts, and the daemon log). These were **deleted**
before completion, and the running daemon was stopped. The **only** change this investigation makes to
the repository is this single documentation file — no source, test, config, dependency, migration, or
CI file is touched, consistent with the user directive *"Don't modify any source files in the
repository."*

Immediately before committing this document (with all temporary artifacts already removed), the
working tree shows **only this file** as changed and **no** stray `blitzy_adhoc_test_*` files:

```
$ git status --porcelain
 M blitzy/documentation/app_2cd6ee777f8c.md
```

The durable proof that no source file was ever touched is the cumulative diff since the
pre-investigation baseline — commit `2cd6ee777f8c`, the commit the source branch `app_2cd6ee777f8c`
was cut from — which is exactly **one** file:

```
$ git diff --name-status 2cd6ee777f8c..HEAD
A	blitzy/documentation/app_2cd6ee777f8c.md
```

No entry for any tracked source path (` M`, `MM`, ` D`, `D `, or an `A`/`M` for anything other than
this document) appears. The single path added to the repository across the entire investigation is
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
| `get_verp_info_from_email` (HMAC + future-dating clamp) | `app/email_utils.py:1467-1499` (HMAC compare `:1490`, future-dating clamp `:1496`) |
| `BOUNCE_PREFIX` / `BOUNCE_SUFFIX` / reply prefix | `app/config.py:100` / `:101` / `:108-110` |
| `VERP_MESSAGE_LIFETIME` / `VERP_PREFIX` / secret guard | `app/config.py:499` / `:500` / `:505-508` |
| status strings (`E211/E212/E213/E404/E510/E512/E515`) | `app/email/status.py:19,20,21,32,47,49,51` |
| `VerpType` enum | `app/models.py:247-250` |
| `EmailLog.is_reply` / `get_phase` / `User.is_active` | `app/models.py:2075` / `:2143-2147` / `:766-769` |
| `VERPForward` / `VERPReply` / `VERPTransactional` | `app/errors.py:42-57` |

### 11.6 Observation scripts (inlined for self-contained reproducibility)

The scripts below are the exact, unedited sources of the ephemeral probe drivers referenced by
§5.2 and §7.1.1. They were created in `/workspace` with the `blitzy_adhoc_test_` prefix and
**deleted** on completion (§11.4); they are reproduced here verbatim so every command in this
document can be re-run from scratch without any file that survives in the repository.

`blitzy_adhoc_test_probe_wire.py` — Path A (canonical wire) driver:

```python
#!/usr/bin/env python3
"""Path A (canonical wire path) probe driver.

Opens a raw SMTP session to the running aiosmtpd daemon (email_handler.py),
emits the null reverse-path MAIL FROM:<>, RCPT TO:<addr>, then DATA with the
crafted .eml body. Prints the verbatim reply lines. The reply to the
terminating '.' of DATA is MailHandler.handle_DATA's return value
(email_handler.py:2289-2293) -- i.e. the exact SMTP wire status line.

Usage: blitzy_adhoc_test_probe_wire.py <port> <rcpt_addr> <eml_file>
"""
import sys
import smtplib


def fmt(code, msg):
    text = msg.decode() if isinstance(msg, (bytes, bytearray)) else str(msg)
    return "{} {}".format(code, text)


def main():
    port = int(sys.argv[1])
    rcpt = sys.argv[2]
    eml = sys.argv[3]
    with open(eml, "rb") as fd:
        data = fd.read()
    s = smtplib.SMTP("localhost", port, timeout=30)
    try:
        s.ehlo("probe.local")
        code_m, msg_m = s.mail("")          # MAIL FROM:<>  (null reverse-path)
        print("MAIL  ->", fmt(code_m, msg_m))
        code_r, msg_r = s.rcpt(rcpt)        # RCPT TO:<rcpt>
        print("RCPT  ->", fmt(code_r, msg_r))
        code_d, msg_d = s.data(data)        # DATA ... <CRLF>.<CRLF>
        print("FINAL ->", fmt(code_d, msg_d))
    finally:
        try:
            s.quit()
        except Exception:
            pass


if __name__ == "__main__":
    main()
```

`blitzy_adhoc_test_probe_harness.py` — Path B (corroborating harness) driver:

```python
#!/usr/bin/env python3
"""Path B (corroborating harness) probe driver.

Mirrors tests/test_email_handler.py:81-85: build an Envelope(), set the null
reverse-path mail_from and the bounce rcpt_to, parse the crafted .eml, and call
the module-level email_handler.handle(...) directly -- the routing/detection/
response core. Wrapped in create_light_app().app_context(), the exact context
the real wire path uses (email_handler._handle, email_handler.py:2352).

Usage: blitzy_adhoc_test_probe_harness.py <rcpt_addr> <eml_file>
"""
import sys
import email
from aiosmtpd.smtp import Envelope

import email_handler
from server import create_light_app


def main():
    rcpt = sys.argv[1]
    eml = sys.argv[2]
    with open(eml, "rb") as fd:
        msg = email.message_from_bytes(fd.read())
    envelope = Envelope()
    envelope.mail_from = "<>"               # null reverse-path, as the wire yields
    envelope.rcpt_tos = [rcpt]
    print("content_type ->", msg.get_content_type())
    with create_light_app().app_context():
        try:
            result = email_handler.handle(envelope, msg)
            print("HARNESS -> handle() returned ->", result)
        except Exception as e:
            print("HARNESS -> handle() raised", type(e).__name__)


if __name__ == "__main__":
    main()
```

`blitzy_adhoc_test_signed.py` — signed-VERP generation/verification + plaintext extraction
(the exact producer of the addresses probed in §5.2 rows 6a–6c):

```python
#!/usr/bin/env python3
"""Signed-VERP generation/verification + plaintext extraction evidence.

- Plaintext OLD path: parse_id_from_bounce (app/email_utils.py:1258-1259) -- no crypto.
- Signed NEW path: generate_verp_email (:1438-1464) + get_verp_info_from_email (:1467-1499):
    * valid signature          -> accepted (id recovered)
    * tampered signature       -> HMAC compare fails (:1490)  -> None
    * tampered payload         -> signature no longer matches -> None
    * future-dated (>5d ahead) -> future-dating clamp (:1496) -> None
      (generated by patching app.email_utils.time.time to a future value DURING
       generation only; verification uses the real, unmocked clock -- a
       deliberately future-dated address a normal generator would never emit.)

Tampering flips the FIRST base32 char of the target segment (always meaningful
bits; the trailing char can hold only padding bits) and the four addresses are
written to blitzy_adhoc_test_addrs.txt so the wire probes reuse this same set.

Usage: blitzy_adhoc_test_signed.py <known_valid_id> <absent_id>
"""
import sys
import time as _time
import base64
import json
from unittest import mock

from app import config
from app.models import VerpType
from app.email_utils import (
    parse_id_from_bounce,
    generate_verp_email,
    get_verp_info_from_email,
)

VERP_TIME_START = 1640995200
B32 = "abcdefghijklmnopqrstuvwxyz234567"


def tamper_first(seg):
    for c in B32:
        if c != seg[0]:
            return c + seg[1:]
    return seg


def payload_minute(addr):
    seg = addr.split("@")[0].split(".")[1]
    pad = (8 - (len(seg) % 8)) % 8
    return json.loads(base64.b32decode(seg.encode().upper() + b"=" * pad))[2]


def main():
    known = int(sys.argv[1])
    absent = int(sys.argv[2])

    print("=== OLD plaintext parse_id_from_bounce (NO crypto) ===")
    for addr in ("bounce+%d+@sl.local" % known,
                 "bounce+%d+@sl.local" % absent,
                 "bounce+12345+@domain"):
        print("PLAINTEXT %-28s -> %d" % (addr, parse_id_from_bounce(addr)))

    print("\n=== NEW signed generate_verp_email + get_verp_info_from_email ===")
    valid = generate_verp_email(VerpType.bounce_forward, known)
    u, dom = valid.split("@")
    p, pay, sig = u.split(".")
    tampered_sig = "%s.%s.%s@%s" % (p, pay, tamper_first(sig), dom)
    tampered_pay = "%s.%s.%s@%s" % (p, tamper_first(pay), sig, dom)
    with mock.patch("app.email_utils.time.time", return_value=_time.time() + 10 * 86400):
        future = generate_verp_email(VerpType.bounce_forward, known)

    now_min = (_time.time() - VERP_TIME_START) / 60
    thr = (_time.time() + config.VERP_MESSAGE_LIFETIME - VERP_TIME_START) / 60

    print("ADDR_VALID              ", valid)
    print("VERIFY_VALID            ", get_verp_info_from_email(valid), "   # accepted, id recovered")
    print("ADDR_TAMPERED_SIG       ", tampered_sig)
    print("VERIFY_TAMPERED_SIG     ", get_verp_info_from_email(tampered_sig), "                                  # rejected: HMAC mismatch (:1490)")
    print("ADDR_TAMPERED_PAYLOAD   ", tampered_pay)
    print("VERIFY_TAMPERED_PAYLOAD ", get_verp_info_from_email(tampered_pay), "                                  # rejected: signature no longer matches")
    print("ADDR_FUTUREDATED        ", future)
    print("VERIFY_FUTUREDATED      ", get_verp_info_from_email(future),
          "                                  # correctly signed but future-dated -> clamp (:1496)")
    print("  future payload_minute =", payload_minute(future),
          "> reject-above =", round(thr), "(now_min =", round(now_min), ")")

    with open("/workspace/blitzy_adhoc_test_addrs.txt", "w") as f:
        f.write("VALID=%s\nTAMPERED=%s\nTAMPERED_PAYLOAD=%s\nFUTURE=%s\n"
                % (valid, tampered_sig, tampered_pay, future))


if __name__ == "__main__":
    main()
```

`blitzy_adhoc_test_seed_state.py` — seeds the two fresh, pristine state-demo rows (`414` forward /
`415` reply) on a fresh **active** user, both `bounced=False`, using the repository's own helpers.
This is where the reply contact's `website_email` (`contact@example.com`, seen keyed on the reply
`Bounce` row in §7.3) is set:

```python
#!/usr/bin/env python3
"""Seed two FRESH EmailLog rows (forward + reply) on a fresh ACTIVE user, both
with bounced=False, so a clean before/during/after state transition can be
observed. Uses the repository's own helpers/models. Prints the new ids.
"""
from tests.utils import create_new_user
from app.models import Alias, Contact, EmailLog
from app.db import Session
from server import create_light_app

with create_light_app().app_context():
    user = create_new_user()
    alias = Alias.create_new_random(user)
    Session.commit()
    contact = Contact.create(
        user_id=user.id,
        alias_id=alias.id,
        website_email="contact@example.com",
        reply_email="rep@sl.local",
        commit=True,
    )
    fwd = EmailLog.create(
        user_id=user.id, contact_id=contact.id, alias_id=contact.alias_id, commit=True
    )
    rpl = EmailLog.create(
        user_id=user.id, contact_id=contact.id, alias_id=contact.alias_id,
        is_reply=True, commit=True,
    )
    print("SEED_USER_ID", user.id, "is_active", user.is_active())
    print("SEED_FORWARD_ID", fwd.id, "is_reply", fwd.is_reply, "bounced", fwd.bounced)
    print("SEED_REPLY_ID", rpl.id, "is_reply", rpl.is_reply, "bounced", rpl.bounced)
```

The exact run that seeded this session's ids printed:

```
SEED_USER_ID 810 is_active True
SEED_FORWARD_ID 414 is_reply False bounced False
SEED_REPLY_ID 415 is_reply True bounced False
```

### 11.7 Crafted `.eml` inputs (inlined)

The two crafted message bodies referenced throughout §5–§7. `blitzy_adhoc_test_dsn.eml` is the
`multipart/report` DSN that satisfies `is_bounce` (§6.1); `blitzy_adhoc_test_plain.eml` is the
`text/plain` message that fails it (row 5 of the matrix). Both were deleted on completion (§11.4).

`blitzy_adhoc_test_dsn.eml` — `Content-Type: multipart/report` (passes `is_bounce`):

```
From: mailer-daemon@relay.example.com
To: postmaster@sl.local
Subject: Delivery Status Notification (Failure)
MIME-Version: 1.0
Content-Type: multipart/report; report-type=delivery-status; boundary="BOUNDARY_DSN"

--BOUNDARY_DSN
Content-Type: text/plain; charset=us-ascii

This is an automatically generated Delivery Status Notification.
Delivery to the following recipient failed permanently.

--BOUNDARY_DSN
Content-Type: message/delivery-status

Reporting-MTA: dns; relay.example.com
Final-Recipient: rfc822; recipient@example.com
Action: failed
Status: 5.1.1
Diagnostic-Code: smtp; 550 5.1.1 user unknown

--BOUNDARY_DSN
Content-Type: message/rfc822

From: sender@example.com
To: recipient@example.com
Subject: test message

original message body
--BOUNDARY_DSN--
```

`blitzy_adhoc_test_plain.eml` — `Content-Type: text/plain` (fails `is_bounce` ⇒ row 5 `E213`):

```
From: someone@example.com
To: postmaster@sl.local
Subject: not a bounce
MIME-Version: 1.0
Content-Type: text/plain; charset=us-ascii

This is an ordinary plain-text message, NOT a multipart/report DSN.
```

---

*All evidence in this document was produced by executing the real code paths inside the canonical
Docker runtime; status strings are quoted byte-for-byte from `app/email/status.py` and from captured
SMTP replies. No repository source file was modified.*

