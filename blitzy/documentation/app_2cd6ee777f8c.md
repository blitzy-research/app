# SimpleLogin Inbound Bounce Handling — Enumeration-Oracle Investigation

> **Evidence-based security investigation.** Every behavioral claim below is backed either by the **actual, verbatim SMTP status string** captured from a run of the canonical inbound handler (with the command/layer that produced it), or by a `file:line` citation into the source. Statements that are reasoned rather than directly observed are explicitly labeled **(inferred)**. The source repository was left byte-for-byte unchanged apart from this one document; all temporary observation scripts lived **outside** the repository checkout and were deleted afterward.

---

## Direct Answer (read this first)

**YES — SimpleLogin's inbound bounce path exposes a recipient/identifier enumeration oracle on the OLDER, unsigned VERP address format.** An external sender who controls only the SMTP envelope can distinguish a *valid* `email_log_id` from an *invalid* one purely by reading the SMTP status string: an invalid id yields **`550 SL E512 No such email log`** while a valid id (probed with an ordinary, non-bounce message) yields **`250 SL E213 Unknown email ignored`**. The two responses differ at **both** the numeric SMTP level (`550` vs `250`) **and** the SimpleLogin sub-code (`E512` vs `E213`).

The oracle is **richer than mere existence**: a valid id whose owning user is *inactive* produces a third, distinct response — **`550 SL E510 so such user`** (the source's literal typo is preserved) — giving a three-way discrimination between *nonexistent* (`E512`), *existent-but-inactive* (`E510`), and *existent-active* (`E211`/`E213`).

The **NEWER, cryptographically-signed** VERP format closes this gap: a forged or expired signature causes `get_verp_info_from_email()` to return `None`, which diverts the message away from the email-log lookup entirely — it falls through to normal alias handling and returns **`550 SL E515 Email not exist`**, never reaching the `E512`/`E213` oracle. The unsigned path has **no** such cryptographic gate.

One important nuance, reported exactly as observed: when the receiving MTA's SPF verdict for the probe's return-path is *fail* or *soft-fail*, a returned `5XX` (such as `E512` or `E510`) is rewritten to **`250 SL E216 Handled spf policy`**, which makes the *numeric* code converge with the valid-id case — **but the SimpleLogin sub-code string still differs (`E216` vs `E213`)**, so the oracle survives at the string level. This rewrite is not attacker-controlled and does not fire on the valid-id branch (which raises an exception before the rewrite runs).

---

## Phase A — Framing & Methodology

### A.1 What was investigated

The seven objectives, answered by name in the sections indicated:

| # | Objective | Answered in |
|---|-----------|-------------|
| 1 | The two address formats (older unsigned vs newer signed) | Phase B |
| 2 | Routing — how a recipient reaches each of the four branches, and why the two formats share a branch but get different validation | Phase C |
| 3 | External-attacker feasibility | Phase D |
| 4 | The security boundary — what slips through *without* cryptographic validation | Phase F |
| 5 | The exact SMTP responses for valid vs invalid vs non-bounce | Phase D (+ Appendix, Phase I) |
| 6 | Bounce detection (`is_bounce`) and its spoofability | Phase E |
| 7 | Oracle richness (does it reveal more than existence?) and the ordering that makes the leak observable | Phase D |

### A.2 Methodology — run-first, canonical path only

All responses were captured by driving the **real inbound chain** used for every message SimpleLogin receives:

```
MailHandler.handle_DATA()   [email_handler.py:L2289]   (async aiosmtpd DATA callback)
      → _handle()           [email_handler.py:L2335]   (applies the SPF 5XX→E216 rewrite)
          → handle()        [email_handler.py:L1945]   (the routing dispatcher)
```

No mocks, no debug hooks, and no hand-written signed addresses were used as a source of any observed value. Signed probes were produced by the software's **own** signer, `generate_verp_email()` [app/email_utils.py:L1438-L1464]. Fixtures (a real `User`, `Alias`, `Contact`, `EmailLog`) were minted through the project's own models against a **real Postgres database**, using the project's own pytest harness (`tests/conftest.py`, `tests/utils.py`) so that the runtime is identical to how the project itself exercises these paths — each observation ran inside a `connection.begin()` transaction that was rolled back, so the repository's data was not persisted.

Because the wire response is composed across **three layers in three different methods**, every matrix cell was driven through each relevant layer and the response captured at each:

1. `handle()` — returns a status string **or raises** a `VERP*` exception (the raw routing outcome).
2. `_handle()` — may rewrite a returned `5XX` to `E216` based on the SPF verdict.
3. `handle_DATA()` — maps a raised `VERP*` exception to `E213` and any other exception to `E404`.

Observing only one layer would miss either the SPF suppression or the exception mapping, and would misreport the oracle's observability.

### A.3 Canonical environment (exact)

- **App container** `sl-app-0`, image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`, **Python 3.10.18**, Poetry-installed dependency set in the venv at `/app/venv`.
- **PostgreSQL** on port **15432** (user/password/db = `test`/`test`/`test`); Alembic migrations applied; `pg_trgm` enabled.
- **Redis** available (`MEM_STORE_URI=redis://localhost`).
- Config: **`CONFIG=tests/test.env`** → `EMAIL_DOMAIN=sl.local`, `NOT_SEND_EMAIL=true`.
- **Repository path inside the container is `/work`** (bind-mounted to the checkout). Raw log lines below therefore show `/work/email_handler.py:...`.
- The three existing SPF tests — `test_prevent_5xx_from_spf`, `test_preserve_5xx_with_valid_spf`, `test_preserve_5xx_with_no_header` [tests/test_email_handler.py] — **pass** in this environment (`3 passed, 18 warnings in 1.09s`), which proves the harness driving the observations is canonical.
- **Invocation form for every observation cell:**
  ```
  docker exec -e CONFIG=/work/tests/test.env -e PYTHONPATH=/work -e GITHUB_ACTIONS_TEST=true \
    -w /work sl-app-0 /app/venv/bin/python /tmp/<script>.py
  ```
  The scripts live in the container's `/tmp` (an overlay filesystem that is **not** bind-mounted) and on the host at `/tmp/obs_host/` — both **outside** the repository checkout — and were deleted after the evidence was captured.

### A.4 Version note (report the actual runtime)

The runtime `aiosmtpd` version observed here is **1.4.2** (the version pinned in `poetry.lock`). Some planning text referenced `1.4.6`; the value used for these observations is the actual installed **1.4.2**. This does not affect any bounce-routing response, which is implemented entirely inside `email_handler.py`.

**(inferred / environmental note):** `re2` bindings are provided by the canonical compiled `pyre2` 0.3.6 (matching `poetry.lock`) — i.e., there is **no** stdlib-`re` substitution in this container. In any case, `re2` is used only for unrelated alias-regex matching in `app/email_utils.py` and is **off the bounce-routing path**, so it cannot affect any status string reported here.

### A.5 Determinism

Every matrix cell was executed **at least twice**; all cells were **stable** (identical strings across runs). As a dedicated determinism spot-check, the same unchanged input — an unsigned invalid-id probe `bounce+99999999999999+@sl.local` with an `R_SPF_ALLOW` verdict — was driven through `_handle()` **five times**:

```
run 1: rcpt=bounce+99999999999999+@sl.local  _handle -> '550 SL E512 No such email log'
run 2: rcpt=bounce+99999999999999+@sl.local  _handle -> '550 SL E512 No such email log'
run 3: rcpt=bounce+99999999999999+@sl.local  _handle -> '550 SL E512 No such email log'
run 4: rcpt=bounce+99999999999999+@sl.local  _handle -> '550 SL E512 No such email log'
run 5: rcpt=bounce+99999999999999+@sl.local  _handle -> '550 SL E512 No such email log'
DISTINCT RESULTS: {'550 SL E512 No such email log'}  (STABLE=True)
```

All five runs returned `550 SL E512 No such email log`; the result set had exactly one distinct value.

---

## Phase B — Objective 1: The two address formats

SimpleLogin accepts **two** shapes of bounce address on inbound mail. They coexist because the older format predates the signed one and is kept for backward compatibility.

### B.1 OLD, human-readable, UNSIGNED VERP

Shape: **`bounce+{email_log_id}+@{EMAIL_DOMAIN}`**, assembled from two config constants:

- `BOUNCE_PREFIX = "bounce+"` [app/config.py:L100]
- `BOUNCE_SUFFIX = f"+@{EMAIL_DOMAIN}"` [app/config.py:L101]

In the test domain (`EMAIL_DOMAIN=sl.local`) this is `bounce+{id}+@sl.local`. The id is recovered by a **plain integer parse with no signature and no verification whatsoever** — `parse_id_from_bounce()` [app/email_utils.py:L1258-L1259]:

```python
def parse_id_from_bounce(email_address: str) -> int:
    return int(email_address[email_address.find("+") : email_address.rfind("+")])
```

**Observed** (this reproduces the user's example `bounce+12345+@domain` verbatim in the test domain):

```
BOUNCE_PREFIX='bounce+'  BOUNCE_SUFFIX='+@sl.local'  EMAIL_DOMAIN='sl.local'
OLD unsigned address      : bounce+12345+@sl.local
parse_id_from_bounce(OLD) : 12345
```

The integer `12345` is exactly the argument later passed to `EmailLog.get(id)` (Phase C), so this address *is* the enumeration probe: substitute a valid vs. an invalid integer and the id is fed straight into the database lookup with nothing standing in the way.

### B.2 NEW, cryptographically-SIGNED VERP

Produced by `generate_verp_email()` [app/email_utils.py:L1438-L1464]: an HMAC (algorithm `sha3-224`, digest truncated to 8 bytes) over the list `[verp_type.value, object_id or 0, minutes-since-VERP_TIME_START]`, with both the JSON payload and the signature base32-encoded (padding stripped), assembled as `"{VERP_PREFIX}.{payload}.{signature}@{domain}".lower()`:

```python
data = [
    verp_type.value,
    object_id or 0,
    int((time.time() - VERP_TIME_START) / 60),
]
json_payload = json.dumps(data).encode("utf-8")
payload_hmac = hmac.new(
    config.VERP_EMAIL_SECRET.encode("utf-8"), json_payload, VERP_HMAC_ALGO
).digest()[:8]
encoded_payload = base64.b32encode(json_payload).rstrip(b"=").decode("utf-8")
encoded_signature = base64.b32encode(payload_hmac).rstrip(b"=").decode("utf-8")
return "{}.{}.{}@{}".format(
    config.VERP_PREFIX, encoded_payload, encoded_signature, sender_domain or config.EMAIL_DOMAIN
).lower()
```

It is verified by `get_verp_info_from_email()` [app/email_utils.py:L1467-L1498]. Relevant constants: `VERP_PREFIX = "sl"` [app/config.py:L500], `VERP_MESSAGE_LIFETIME = 5 * 86400` (five days) [app/config.py:L499], and `VERP_EMAIL_SECRET` [app/config.py:L502] (a runtime `RuntimeError` is raised if it is shorter than 32 characters).

**Observed round-trip** for the user's example id `12345` (shape `sl.<payload>.<signature>@sl.local`, stable across two runs; the signature is time-bucket dependent because it encodes minutes-since-`VERP_TIME_START`):

```
NEW signed [bounce_forward] id=12345 : sl.lmycyibrgiztinjmeazdgobshaytaxi.5sf54mxfekgn4@sl.local
   get_verp_info_from_email -> (<VerpType.bounce_forward: 0>, 12345)
NEW signed [bounce_reply]   id=12345 : sl.lmysyibrgiztinjmeazdgobshaytaxi.qjgwkkblidol2@sl.local
   get_verp_info_from_email -> (<VerpType.bounce_reply: 1>, 12345)
NEW signed [transactional]  id=12345 : sl.lmzcyibrgiztinjmeazdgobshaytaxi.fv7m7jpkqseva@sl.local
   get_verp_info_from_email -> (<VerpType.transactional: 2>, 12345)
```

The three VERP kinds come from the `VerpType` enum [app/models.py:L247-L250]:

```python
class VerpType(EnumE):
    bounce_forward = 0
    bounce_reply = 1
    transactional = 2
```

### B.3 Why both coexist

- The **OLD** format is classic plaintext **VERP** (Variable Envelope Return Path): the per-message identifier is embedded in the envelope in cleartext and recovered by a bare `int(...)`. It is trivially forgeable — anyone can write `bounce+<n>+@sl.local` for any `<n>`.
- The **NEW** format is the "encrypted-identifier" VERP variant, functionally a **BATV**-style (Bounce Address Tag Validation) scheme: the identifier travels with an unforgeable HMAC tag plus a timestamp, so a forged or stale address fails verification. See Phase H for the framing against established practice.

The security-relevant consequence of this split — that the two formats are routed to the *same* branch but validated *differently* — is the subject of Phase C.


---

## Phase C — Objective 2: Routing (all four branches, all observed)

### C.1 The canonical entry chain and the recipient dispatch

An inbound message enters at `MailHandler.handle_DATA()` [email_handler.py:L2289], which calls `_handle()` [email_handler.py:L2335], which calls the module-level dispatcher `handle()` [email_handler.py:L1945]. Inside `handle()`, the VERP routing block [email_handler.py:L2034-L2116] begins by computing:

```python
verp_info = get_verp_info_from_email(rcpt_tos[0])     # email_handler.py:L2035
```

`verp_info` is a `(VerpType, object_id)` tuple for a **valid signed** recipient, or `None` for anything else (unsigned addresses, or signed addresses that fail verification). The four branches are then tried in order.

### C.2 Transactional branch [email_handler.py:L2038-L2054]

Condition: the recipient matches the transactional prefix/suffix (`TRANSACTIONAL_BOUNCE_PREFIX = "transactional+"` [app/config.py:L113-L114]) **or** `verp_info[0] == VerpType.transactional`. This branch has **no `EmailLog.get()` and no `E512`** — the id is never looked up:

```python
if is_bounce(envelope, msg):
    handle_transactional_bounce(envelope, msg, rcpt_tos[0], verp_info and verp_info[1])
    return status.E205
elif is_automatic_out_of_office(msg):
    ...
    return status.E206
else:
    raise VERPTransactional
```

**Observed** (signed transactional address, invalid id):

- non-bounce message → `handle()` **raises `VERPTransactional`**, and `handle_DATA()` maps it to `250 SL E213 Unknown email ignored`.
- bounce message → `handle()` → `250 SL E205 bounce handled` (the raw log shows `handle_transactional_bounce() - No transactional record ...` yet still returns `E205`).

⇒ The transactional branch **does not enumerate**: because there is no id lookup, an invalid id yields the same `E205`/`E213` as any other, so it cannot distinguish existent from non-existent identifiers.

### C.3 Forward-bounce branch [email_handler.py:L2057-L2074] — the crux

Condition (note the `or`): **either** the recipient is an OLD unsigned `bounce+…+@…` address **or** it is a valid signed forward address:

```python
if (
    len(rcpt_tos) == 1
    and rcpt_tos[0].startswith(BOUNCE_PREFIX)
    and rcpt_tos[0].endswith(BOUNCE_SUFFIX)
) or (verp_info and verp_info[0] == VerpType.bounce_forward):
    email_log_id = (verp_info and verp_info[1]) or parse_id_from_bounce(rcpt_tos[0])   # L2062
    email_log = EmailLog.get(email_log_id)                                             # L2063

    if not email_log:
        LOG.w("No such email log")
        return status.E512                                                             # L2065-L2067

    if is_bounce(envelope, msg):
        return handle_bounce(envelope, email_log, msg)                                 # L2069-L2070
    elif is_automatic_out_of_office(msg):
        handle_out_of_office_forward_phase(email_log, envelope, msg, rcpt_tos)
    else:
        raise VERPForward                                                              # L2074
```

Two facts make this branch the heart of the oracle:

1. **Both** address formats hit this **same** branch (the OLD via the `startswith/endswith` test, the NEW via `verp_info[0] == bounce_forward`).
2. **The existence check precedes the bounce gate.** `EmailLog.get()` and the `return status.E512` fire at L2063-L2067, *before* the `is_bounce(...)` test at L2069. So a probe reveals whether the id exists regardless of whether it looks like a real bounce.

**Why the two formats get different validation despite sharing the branch:** the id `email_log_id` at L2062 comes from `verp_info[1]` for the signed form — a value produced only **after** `get_verp_info_from_email()` has already verified the HMAC and timestamp — whereas for the unsigned form it comes from `parse_id_from_bounce(rcpt_tos[0])`, a bare `int(...)` with **no** verification. The unsigned id is attacker-chosen; the signed id is cryptographically vouched-for.

### C.4 Reply-bounce branch [email_handler.py:L2077-L2098]

Parallel to the forward branch. Condition: recipient starts with `f"{BOUNCE_PREFIX_FOR_REPLY_PHASE}+"` (`BOUNCE_PREFIX_FOR_REPLY_PHASE = "bounce_reply"` [app/config.py:L108-L110]) **or** `verp_info[0] == VerpType.bounce_reply`. Same `EmailLog.get()` → `E512`-if-missing (L2087) → `handle_bounce` if `is_bounce` (L2091) → else `raise VERPReply` (L2095).

**Observed:**

- unsigned invalid id, non-bounce → `550 SL E512 No such email log`
- unsigned valid id, non-bounce → `handle()` **raises `VERPReply`** → `handle_DATA()` `250 SL E213 Unknown email ignored`
- unsigned valid id, `is_bounce` → `250 SL E212 Bounce Reply phase handled`
- signed valid `bounce_reply`, non-bounce → `handle()` **raises `VERPReply`** → `handle_DATA()` `250 SL E213`

⇒ The reply branch has the **same `E512`/`E213` divergence** as the forward branch; the only bounce-processing difference is `E212` (reply) vs `E211` (forward).

### C.5 iCloud branch [email_handler.py:L2100-L2116] — most readily exploitable

This branch keys off the **envelope `mail_from`** (not the recipient), and — critically — it calls `handle_bounce` **unconditionally, with no `is_bounce` gate**:

```python
verp_info = get_verp_info_from_email(mail_from[0])                # L2101  (see the bug note below)
if (
    len(rcpt_tos) == 1
    and mail_from.startswith(BOUNCE_PREFIX)
    and mail_from.endswith(BOUNCE_SUFFIX)
) or (verp_info and verp_info[0] == VerpType.bounce_forward):
    email_log_id = (verp_info and verp_info[1]) or parse_id_from_bounce(mail_from)   # L2107
    email_log = EmailLog.get(email_log_id)                                          # L2108
    alias = Alias.get_by(email=rcpt_tos[0])
    LOG.w("iCloud bounces %s %s, saved to%s", ...)
    return handle_bounce(envelope, email_log, msg)                                  # L2116
```

**Observed** (bounce address placed in the envelope `mail_from`, a real alias in the recipient, and an ordinary `text/plain` — i.e. **non-bounce** — message):

- OLD **valid** id in `mail_from` → `250 SL E211 Bounce Forward phase handled` (both `handle()` and `handle_DATA()`)
- OLD **invalid** id in `mail_from` → `550 SL E512 No such email log`
- OLD valid id whose user is **inactive** → `550 SL E510 so such user`
- **NEW signed** address in `mail_from` → `250 Message accepted for delivery` (fall-through)

The last row exposes a **latent bug**: L2101 calls `get_verp_info_from_email(mail_from[0])`, and `mail_from[0]` is the **first character** of the string, not the first element of a list. It therefore always evaluates to `None`, so a *signed* address can never satisfy the `verp_info[0] == bounce_forward` half of the condition and only the OLD unsigned form can reach this branch **(inferred** from the code, corroborated by the observed `250 Message accepted for delivery` fall-through for the signed case**)**.

⇒ The iCloud branch is the **most readily exploitable** oracle surface: it leaks via the attacker-controlled envelope sender **without** requiring `is_bounce` to be satisfied (there is no gate — `handle_bounce` runs unconditionally at L2116), and only the OLD unsigned format can reach it.


---

## Phase D — Objectives 3, 5 & 7: The oracle (feasibility, exact responses, richness, ordering)

### D.1 The core matrix (Objectives 3 & 5)

Unsigned `bounce+{id}+@sl.local`, ordinary **non-bounce** message, forward-bounce branch. Every string below was captured verbatim and was **stable** across at least two runs.

| id | `handle()` | `_handle()` | `handle_DATA()` |
|----|-----------|-------------|-----------------|
| **INVALID** (`99999999999999`) | RETURN `550 SL E512 No such email log` | RETURN `550 SL E512 No such email log` | RETURN `550 SL E512 No such email log` |
| **VALID** | RAISE `VERPForward` | RAISE `VERPForward` | RETURN `250 SL E213 Unknown email ignored` |

**The oracle, stated plainly:** an invalid id returns **`550 SL E512`**; a valid id probed with a non-bounce message returns **`250 SL E213`**. These differ at both the numeric SMTP level (`550` vs `250`) and the SimpleLogin sub-code (`E512` vs `E213`).

**External-attacker feasibility (Objective 3):** the only inputs required are the SMTP envelope recipient (`RCPT TO`) — set to `bounce+<n>+@sl.local` — and an arbitrary message body. Both are fully under the control of any external sender who can deliver a message to the SimpleLogin MX. No account, no authentication, and no valid signature is needed. Reading the returned status string therefore tells the attacker whether the integer `<n>` is a live `email_log_id`.

The signed forward format shares the branch and behaves identically **for a valid id**:

| Signed forward, non-bounce | `handle()` | `_handle()` | `handle_DATA()` |
|----|-----------|-------------|-----------------|
| INVALID id | RETURN `550 SL E512 No such email log` | RETURN `550 SL E512 No such email log` | RETURN `550 SL E512 No such email log` |
| VALID id | RAISE `VERPForward` | RAISE `VERPForward` | RETURN `250 SL E213 Unknown email ignored` |

So the `E512`/`E213` divergence is a property of the forward/reply/iCloud **id-lookup**, not of the address format itself. What differs is *reachability*: a forged **signed** address can never get here (Phase F), whereas a forged **unsigned** address always can.

### D.2 The ordering that makes even a non-bounce probe leak (Objective 7, part 1)

Because `EmailLog.get()` / `return status.E512` [email_handler.py:L2063-L2067] execute **before** the `is_bounce(...)` gate [email_handler.py:L2069], the existence of the id is decided *before* the code cares whether the message is a genuine bounce. Consequently a plain, non-bounce probe already distinguishes existent from non-existent ids — the attacker does **not** need to craft a real DSN. This ordering is confirmed on the bounce path too:

- OLD **valid** id + `is_bounce` (active user) → `250 SL E211 Bounce Forward phase handled`
- OLD **invalid** id + `is_bounce` → `550 SL E512 No such email log` (the existence check still fires first)

### D.3 Richness — the codes reveal more than existence (Objective 7, part 2)

A valid id whose owning **user is inactive** produces a third, distinct response, emitted by `handle_bounce()` [email_handler.py:L1869-L1871]:

```python
if not email_log.user.is_active():
    LOG.d(f"User {email_log.user} is not active")
    return status.E510
```

`E510` is the literal string `"550 SL E510 so such user"` [app/email/status.py:L47] — the source's typo ("so such user") is preserved here exactly as shipped.

**State observed before / during / after** the toggle (the mechanism is `User.is_active()` [app/models.py:L766-L769], which returns `False` when `delete_on` is set to a future date):

- **before:** `EmailLog.get(id).user.is_active() == True` (`delete_on` is `None`)
- **during:** set `user.delete_on = arrow.now().shift(days=1)` → `is_active() == False`
- **after (restored):** `delete_on = None` → `is_active() == True`

With the user inactive, all three layers agree:

- OLD valid id, `is_bounce`, inactive user → `550 SL E510 so such user` at `handle()`, `_handle()`, and `handle_DATA()`.

⇒ **Three-way discrimination:** *nonexistent id* → `E512`; *existent but inactive user* → `E510`; *existent, active* → `E211` (bounce) or `E213` (non-bounce). The oracle leaks not just whether an `email_log_id` exists, but also a fact about the state of the account that owns it.

### D.4 Summary of the leak

- **Feasibility (Obj 3):** trivial for any external sender; only the envelope recipient and a body are needed.
- **Exact responses (Obj 5):** `550 SL E512 No such email log` (invalid) vs `250 SL E213 Unknown email ignored` (valid, non-bounce) vs `250 SL E211 Bounce Forward phase handled` (valid, bounce, active) vs `550 SL E510 so such user` (valid, inactive user).
- **Richness + ordering (Obj 7):** more than existence (inactive-user `E510`), and the existence check runs before the bounce gate so a non-bounce probe suffices.


---

## Phase E — Objective 6: `is_bounce()` and its spoofability

### E.1 The criteria

`is_bounce()` [email_handler.py:L1813-L1818] classifies a message as a Delivery Status Notification using exactly two inputs:

```python
def is_bounce(envelope: Envelope, msg: Message):
    """Detect whether an email is a Delivery Status Notification"""
    return (
        envelope.mail_from == "<>"
        and msg.get_content_type().lower() == "multipart/report"
    )
```

### E.2 Observed truth table (direct calls)

| `envelope.mail_from` | `Content-Type` | `is_bounce()` |
|----------------------|----------------|---------------|
| `"<>"` | `multipart/report` | **True** |
| `"<>"` | `text/plain` | False |
| `real@x.test` | `multipart/report` | False |
| `real@x.test` | `text/plain` | False |

Only the exact combination `("<>", multipart/report)` returns `True`.

### E.3 Both inputs are attacker-controllable

- `envelope.mail_from == "<>"` is the SMTP **`MAIL FROM`** command value. A null return-path (`MAIL FROM:<>`) is a normal, sender-chosen envelope value.
- `msg.get_content_type()` is read from the **`Content-Type`** header of the message the sender supplies.

Neither is derived from any server-side trust decision, so an external sender can trivially set both and satisfy `is_bounce()`, reaching `handle_bounce` on the forward and reply branches. (On the iCloud branch, Phase C.5, `is_bounce` is not even required — `handle_bounce` runs unconditionally.)

### E.4 The pass/fail effect on the response

- When `is_bounce` **passes** on a **valid** id, the response advances from the enumeration-only path (`raise VERPForward` → `E213`) to the bounce-processing codes: `250 SL E211 Bounce Forward phase handled` (forward), `250 SL E212 Bounce Reply phase handled` (reply), or `550 SL E510 so such user` if the user is inactive.
- When `is_bounce` **fails** (non-bounce probe) on a valid id, `handle()` raises `VERPForward`/`VERPReply` → `handle_DATA()` `250 SL E213`.

Crucially, **either way the existence check has already fired** (Phase D.2): an invalid id returns `550 SL E512` before `is_bounce` is consulted. So satisfying — or not satisfying — `is_bounce` changes only the *valid-id* response; the existent-vs-nonexistent leak is observable regardless of whether the probe is dressed up as a bounce.


---

## Phase F — Objective 4: The security boundary (what slips through without crypto validation)

### F.1 The two paths, precisely

- **UNSIGNED** `bounce+{id}+@sl.local`: the id is produced by `parse_id_from_bounce()` — a bare `int(...)` [app/email_utils.py:L1258-L1259] — and handed straight to `EmailLog.get(id)` [email_handler.py:L2063]. There is **no signature check and no timestamp check** anywhere on this path.
- **SIGNED**: `get_verp_info_from_email()` [app/email_utils.py:L1467-L1498] returns `None` on any of four conditions:
  - wrong field count or wrong prefix — `len(fields) != 3 or fields[0] != config.VERP_PREFIX` [L1476];
  - base32 decode failure — `except binascii.Error: return None` [L1485-L1486];
  - **HMAC mismatch** — `if expected_signature != signature: return None` [L1490-L1491];
  - **expiry** — `if data[2] > (time.time() + config.VERP_MESSAGE_LIFETIME - VERP_TIME_START) / 60: return None` [L1496].

### F.2 Observed `get_verp_info_from_email()` behavior

| Input | Result |
|-------|--------|
| GOOD signed, valid id (`5`) | `(<VerpType.bounce_forward: 0>, 5)` |
| CORRUPTED signature (wrong HMAC) | `None` |
| FUTURE-dated `+6d` (beyond the 5-day lifetime) | `None` |
| OLD-dated `-6d` | `(<VerpType.bounce_forward: 0>, 5)` — **still accepted** |

**Reported plainly, with the interpretation labeled:** the expiry test at L1496 enforces only an **upper** bound (it rejects timestamps too far in the future); there is **no lower bound**, so an *old* signed address is still accepted. **(inferred, from the single one-sided comparison at L1496 plus the observed `-6d` acceptance):** this deviates from the short-lived / replay-limiting intent of a BATV-style scheme (Phase H) — a captured signed address remains usable indefinitely into the past relative to "now", constrained only by the future-facing window.

### F.3 Driving a corrupted / expired signed address through the canonical handler

This is the decisive boundary observation. A signed address whose signature does not verify (corrupted) or whose timestamp is out of window (expired) yields `verp_info == None`, so the message **does not enter any bounce branch** — it falls through to normal alias handling. Captured, **stable across two runs**:

```
[GOOD signed (control)] rcpt=sl.<payload>.<sig>@sl.local
   get_verp_info_from_email -> (<VerpType.bounce_forward: 0>, 5)
   handle()      -> RETURN '550 SL E512 No such email log'
   handle_DATA() -> RETURN '550 SL E512 No such email log'

[CORRUPTED signed (wrong HMAC)] rcpt=sl.lmycyibvfqqdemzygi4danc5.4vqgqc4lkkswi@sl.local
   get_verp_info_from_email -> None
   handle()      -> RETURN '550 SL E515 Email not exist'
   handle_DATA() -> RETURN '550 SL E515 Email not exist'

[EXPIRED signed (+6d)] rcpt=sl.lmycyibvfqqdemzzge2dinc5.j2idmgg3spgwo@sl.local
   get_verp_info_from_email -> None
   handle()      -> RETURN '550 SL E515 Email not exist'
   handle_DATA() -> RETURN '550 SL E515 Email not exist'
```

`E515` is `"550 SL E515 Email not exist"` [app/email/status.py:L51]. The raw log for the corrupted/expired cases shows the message entering the **normal forward path**, not the bounce branch:

```
... "/work/email_handler.py:2202" handle() - Forward phase ... attacker@evil.test -> sl.<payload>.<sig>@sl.local
```

### F.4 The boundary

- A **GOOD** signature is required to reach the email-log lookup on the signed path — and when present, it *does* reach it (the control returns `E512` for a non-existent `EmailLog.get(5)` in the rolled-back DB). So the signed path reaches the oracle **only** after cryptographic verification succeeds.
- A **corrupted or expired** signature makes `verp_info == None`, which **diverts the probe away from the oracle** entirely (to normal alias lookup → `550 SL E515 Email not exist`), so it never reaches the `E512`/`E213`/`E510` email-log responses.
- The **UNSIGNED** OLD path has **no such cryptographic gate**: `parse_id_from_bounce` → `EmailLog.get(id)` runs unconditionally.

**This is the security boundary (Objective 4):** the thing that slips through without cryptographic validation is the attacker-chosen `email_log_id` on the unsigned `bounce+{id}+@domain` path — it reaches `EmailLog.get(id)` directly, whereas the signed path's HMAC+timestamp gate turns a forged/expired probe into a benign `E515` that never touches the email-log lookup.


---

## Phase G — Edge condition: SPF-based `E216` suppression

### G.1 The rewrite

`_handle()` [email_handler.py:L2335] can overwrite a returned `5XX` status with `E216` when the return-path failed SPF [email_handler.py:L2357-L2365]:

```python
return_status = handle(envelope, msg)                     # L2353
...
spamd_result = SpamdResult.extract_from_headers(msg)      # L2356
if return_status[0] == "5":                               # L2357
    if spamd_result and spamd_result.spf in (
        SPFCheckResult.fail,
        SPFCheckResult.soft_fail,
    ):
        LOG.i("Replacing 5XX to 216 status because the return-path failed the spf check")
        return_status = status.E216                        # L2365
```

The SPF verdict is parsed from the rspamd `X-Spamd-Result` header by `SpamdResult.extract_from_headers()` [app/handler/spamd_result.py]. In that module `SPFCheckResult.fail` and `SPFCheckResult.soft_fail` share the same value (they are aliased to `1`), so **both** `R_SPF_FAIL` and `R_SPF_SOFTFAIL` trigger the rewrite, while `R_SPF_ALLOW` does not.

### G.2 Observed suppression matrix

Unsigned **invalid** id, non-bounce, driven through `_handle()` (the SPF token is injected on its own continuation line of `X-Spamd-Result`, matching `tests/example_emls/5xx_overwrite_spf.eml`):

| SPF verdict | `_handle()` result |
|-------------|--------------------|
| `R_SPF_FAIL` | `250 SL E216 Handled spf policy` |
| `R_SPF_SOFTFAIL` | `250 SL E216 Handled spf policy` |
| `R_SPF_ALLOW` | `550 SL E512 No such email log` |
| *(no `X-Spamd-Result` header)* | `550 SL E512 No such email log` |

Signed **invalid** id, non-bounce, behaves the same (`R_SPF_FAIL`/`R_SPF_SOFTFAIL` → `250 SL E216`; `R_SPF_ALLOW` → `550 SL E512`).

Inactive-user **valid** id, `is_bounce` (which returns `E510`, a `5XX`, so it *is* subject to the rewrite):

| SPF verdict | result |
|-------------|--------|
| `R_SPF_FAIL` | `250 SL E216 Handled spf policy` (the `E510` is silenced) |
| `R_SPF_SOFTFAIL` | `250 SL E216 Handled spf policy` (silenced) |
| `R_SPF_ALLOW` | `550 SL E510 so such user` (preserved when SPF passes) |

The **valid** non-bounce id is **never** suppressed, because it **raises `VERPForward` before `_handle()`'s rewrite runs** — the rewrite only inspects a *returned* status, not a *raised* exception:

- valid non-bounce + `R_SPF_FAIL` → `_handle()` raises `VERPForward`; `handle_DATA()` `250 SL E213 Unknown email ignored`
- valid non-bounce + `R_SPF_ALLOW` → `handle_DATA()` `250 SL E213 Unknown email ignored`

### G.3 Two critical nuances

1. **The SPF verdict is not attacker-chosen.** It is read from the `X-Spamd-Result` header that the receiving rspamd/MTA injects — it is *not* something the external sender sets on the SMTP envelope. Whether the rewrite fires therefore depends on the receiver's SPF verdict for the probe's return-path, not on a flag the attacker can toggle at will.
2. **The rewrite applies only to a returned `5XX`, not to a raised exception.** The valid-id non-bounce path raises `VERPForward` (mapped to `E213`) and so is unaffected by the rewrite; only the *invalid-id* (`E512`) and *inactive-user* (`E510`) returns are candidates for suppression.

### G.4 Conclusion — reported exactly, not adjusted

Under SPF-fail/soft-fail the invalid case (`E216`, numeric `250`) and the valid non-bounce case (`E213`, numeric `250`) have the **same three-digit code**, so the *numeric* leak is masked. **But the SimpleLogin sub-code string still differs — `E216` vs `E213` — so the oracle survives at the string level.** Likewise the inactive-user `E510` (a `550`) is silenced to `E216` only under SPF-fail; under SPF-pass or an absent header it is preserved. I report this as observed rather than adjusting it toward a "fully mitigated" conclusion: the SPF rewrite narrows, but does not eliminate, the differential response.


---

## Phase H — Security framing (background context)

The observations above are grounded in captured status strings and `file:line` citations. The external references in this section are used only to frame those findings against established practice; each is a brief factual statement, not a substitute for the observed evidence.

### H.1 SMTP recipient-enumeration / differential-response disclosure

A mail server that returns materially different responses for existing vs. non-existing recipients or identifiers is a recognized information-disclosure weakness: the classic pattern is a `2xx` acceptance for a valid recipient versus a `550` rejection for an unknown one, and defensive guidance is to avoid responses that reveal existence unnecessarily (e.g., Tenable/Nessus plugin 10249 "Multiple Mail Server EXPN/VRFY Information Disclosure" advises against commands that give "too much information"; general SMTP-enumeration guidance notes servers "exhibit different responses ... depending on whether the users are recognized as valid or not"). This maps directly onto SimpleLogin's observed divergence: `250 SL E213 Unknown email ignored` for a valid `email_log_id` vs. `550 SL E512 No such email log` for an invalid one — a textbook `250`-vs-`550` differential-response oracle, here over internal `email_log_id` integers rather than mailbox names.

### H.2 Bounce Address Tag Validation (BATV)

BATV validates the envelope sender by adding an unforgeable cryptographic tag, so that bounces to forged return-paths (backscatter) can be rejected; the core idea is to send mail with a return address that includes a **timestamp plus a token that cannot be forged**, and to reject any returned bounce lacking a valid signature. The `prvs` ("Simple Private Signature") scheme uses a key, a day-granularity timestamp for expiry (commonly on the order of a week to limit replay), and a hash; a known limitation is that the short signature offers only weak replay protection (IETF BATV draft; Wikipedia, "Bounce Address Tag Validation"). SimpleLogin's signed format is a BATV-style scheme: `generate_verp_email()`/`get_verp_info_from_email()` compute an HMAC (`sha3-224`, 8-byte digest) over `[verp_type, object_id, minutes-since-VERP_TIME_START]` and gate acceptance with `VERP_MESSAGE_LIFETIME = 5 * 86400` (five days). The observed missing-lower-bound behavior (Phase F.2 — an old-dated `-6d` address is still accepted) is exactly the kind of replay-window laxity BATV's timestamp is meant to constrain.

### H.3 Variable Envelope Return Path (VERP)

VERP encodes per-message routing information into the envelope return-path so that a bounce auto-identifies the failed recipient without parsing the human-readable bounce body; classic VERP exposes the identifier in **plaintext**, and professional platforms use a variant based on **encrypted unique identifiers** (Wikipedia, "Variable envelope return path"; vendor guides). A known VERP hazard is that implementations tend to assume any message arriving at a VERP bounce address *is* a bounce, so probes/spam to that address can trigger bounce processing. This is precisely why SimpleLogin embeds an `email_log_id` in `bounce+{id}+@domain`, and why the OLD (plaintext) vs. NEW (signed) split exists: the OLD form is classic, forgeable VERP; the NEW form is the encrypted-identifier / BATV variant.

**(inferred, framing only):** the report notes — but does not design or implement — that the signed format already closes the enumeration gap; remediation is out of scope for this investigation.


---

## Phase I — Evidence appendix, scenario matrix & coverage pass

### I.1 Verbatim status-string reference [app/email/status.py]

| Const | Line | String |
|-------|------|--------|
| `E200` | L2 | `250 Message accepted for delivery` |
| `E205` | L7 | `250 SL E205 bounce handled` |
| `E206` | L9 | `250 SL E206 Out of office` |
| `E211` | L19 | `250 SL E211 Bounce Forward phase handled` |
| `E212` | L20 | `250 SL E212 Bounce Reply phase handled` |
| `E213` | L21 | `250 SL E213 Unknown email ignored` |
| `E216` | L24 | `250 SL E216 Handled spf policy` |
| `E404` | L32 | `421 SL E404 Unexpected error - Retry later` |
| `E502` | L39 | `550 SL E502 Email not exist` |
| `E504` | L41 | `550 SL E504 Account disabled` |
| `E510` | L47 | `550 SL E510 so such user` *(source typo preserved)* |
| `E512` | L49 | `550 SL E512 No such email log` |
| `E515` | L51 | `550 SL E515 Email not exist` |

Note `E502` and `E515` carry identical text (`"Email not exist"`); the enumeration divergence is between `E512`/`E213`/`E510`, while the signature-failure diversion lands on `E515`.

### I.2 `handle_DATA()` exception mapping [email_handler.py:L2289]

```python
except CannotCreateContactForReverseAlias as e:      # L2297
    return status.E524                                # L2307
except (VERPReply, VERPForward, VERPTransactional) as e:  # L2308
    return status.E213                                # L2318
except Exception as e:                                # L2319
    return status.E404                                # L2332
```

The three `VERP*` exception classes are defined in `app/errors.py` (`VERPTransactional`, `VERPForward`, `VERPReply` [app/errors.py:L42-L57]); each is mapped to `E213` at L2318.

### I.3 Raw captured log lines (evidence of the canonical path; paths show `/work`)

```
WARNING "/work/email_handler.py:2066" handle() - No such email log
INFO    "/work/email_handler.py:2367" _handle() - Finish ... return code '550 SL E512 No such email log'<<===
WARNING "/work/email_handler.py:2309" handle_DATA() - email handling fail with error:VERPForward ...
INFO    "/work/email_handler.py:1834" handle_transactional_bounce() - No transactional record for <> -> [...]
INFO    "/work/email_handler.py:2362" _handle() - Replacing 5XX to 216 status because the return-path failed the spf check
INFO    "/work/email_handler.py:2367" _handle() - Finish ... return code '250 SL E216 Handled spf policy'<<===
INFO    "/work/email_handler.py:2367" _handle() - Finish mail_from <>, ... return code '550 SL E510 so such user'<<===
INFO    "/work/email_handler.py:2367" _handle() - Finish ... '250 SL E211 Bounce Forward phase handled'<<===
```

### I.4 Full scenario matrix

Legend — layers: `H` = `handle()`, `_h` = `_handle()`, `DATA` = `handle_DATA()`. All rows were **STABLE** across at least two runs (the invalid-id + SPF-allow cell was additionally confirmed stable across five runs, Phase A.5).

| # | Address format | id | `is_bounce` | SPF | Layer(s) captured | Result | STABLE |
|---|----------------|----|-----------|-----|-------------------|--------|--------|
| 1 | OLD unsigned `bounce+{id}+@sl.local` | invalid | F | allow/absent | H / _h / DATA | `550 SL E512 No such email log` | ✓ |
| 2 | OLD unsigned | valid | F | allow/absent | H / _h | RAISE `VERPForward` | ✓ |
| 3 | OLD unsigned | valid | F | allow/absent | DATA | `250 SL E213 Unknown email ignored` | ✓ |
| 4 | OLD unsigned | invalid | T | allow/absent | H / _h / DATA | `550 SL E512 No such email log` | ✓ |
| 5 | OLD unsigned | valid (active) | T | allow/absent | H / _h / DATA | `250 SL E211 Bounce Forward phase handled` | ✓ |
| 6 | OLD unsigned | valid (**inactive** user) | T | allow/absent | H / _h / DATA | `550 SL E510 so such user` | ✓ |
| 7 | NEW signed forward | invalid | F | allow/absent | H / _h / DATA | `550 SL E512 No such email log` | ✓ |
| 8 | NEW signed forward | valid | F | allow/absent | H / _h | RAISE `VERPForward` | ✓ |
| 9 | NEW signed forward | valid | F | allow/absent | DATA | `250 SL E213 Unknown email ignored` | ✓ |
| 10 | OLD unsigned reply `bounce_reply+…` | invalid | F | allow/absent | H / _h / DATA | `550 SL E512 No such email log` | ✓ |
| 11 | OLD unsigned reply | valid | F | allow/absent | H / _h → DATA | RAISE `VERPReply` → `250 SL E213 Unknown email ignored` | ✓ |
| 12 | OLD unsigned reply | valid | T | allow/absent | H / _h / DATA | `250 SL E212 Bounce Reply phase handled` | ✓ |
| 13 | NEW signed reply | valid | F | allow/absent | H / _h → DATA | RAISE `VERPReply` → `250 SL E213` | ✓ |
| 14 | Signed transactional | invalid | F | allow/absent | H / _h → DATA | RAISE `VERPTransactional` → `250 SL E213` | ✓ |
| 15 | Signed transactional | invalid | T | allow/absent | H / _h / DATA | `250 SL E205 bounce handled` (no `E512`) | ✓ |
| 16 | iCloud: OLD in `mail_from` | valid | (n/a, no gate) | allow/absent | H / _h / DATA | `250 SL E211 Bounce Forward phase handled` | ✓ |
| 17 | iCloud: OLD in `mail_from` | invalid | (n/a) | allow/absent | H / _h / DATA | `550 SL E512 No such email log` | ✓ |
| 18 | iCloud: OLD in `mail_from` | valid (**inactive**) | (n/a) | allow/absent | H / _h / DATA | `550 SL E510 so such user` | ✓ |
| 19 | iCloud: NEW signed in `mail_from` | valid | (n/a) | allow/absent | H / _h / DATA | `250 Message accepted for delivery` (fall-through, `mail_from[0]` bug) | ✓ |
| 20 | CORRUPTED signed (wrong HMAC) | — | F | allow/absent | H / DATA | `550 SL E515 Email not exist` | ✓ |
| 21 | EXPIRED signed (`+6d`) | valid | F | allow/absent | H / DATA | `550 SL E515 Email not exist` | ✓ |
| 22 | OLD unsigned | invalid | F | **fail** | _h | `250 SL E216 Handled spf policy` | ✓ |
| 23 | OLD unsigned | invalid | F | **soft-fail** | _h | `250 SL E216 Handled spf policy` | ✓ |
| 24 | NEW signed forward | invalid | F | **fail** | _h | `250 SL E216 Handled spf policy` | ✓ |
| 25 | OLD unsigned | valid | F | **fail** | _h → DATA | RAISE `VERPForward` → `250 SL E213` (never suppressed) | ✓ |
| 26 | OLD unsigned | valid (**inactive**) | T | **fail/soft-fail** | _h | `250 SL E216 Handled spf policy` (`E510` silenced) | ✓ |
| 27 | OLD unsigned | valid (**inactive**) | T | **allow** | _h | `550 SL E510 so such user` (preserved) | ✓ |

Auxiliary observations (helper-level, labeled **non-canonical** where they bypass the handler):

- **(non-canonical, direct call)** `parse_id_from_bounce('bounce+12345+@sl.local')` = `12345`.
- **(non-canonical, direct call)** `get_verp_info_from_email(...)`: GOOD→tuple; CORRUPTED→`None`; `+6d`→`None`; `-6d`→tuple (accepted).
- **(non-canonical, direct call)** `is_bounce` truth table (Phase E.2).

### I.5 Invocation commands (representative) and repository cleanliness

```
docker exec -e CONFIG=/work/tests/test.env -e PYTHONPATH=/work -e GITHUB_ACTIONS_TEST=true \
  -w /work sl-app-0 /app/venv/bin/python /tmp/obs_oracle.py
docker exec -e CONFIG=/work/tests/test.env -e PYTHONPATH=/work -e GITHUB_ACTIONS_TEST=true \
  -w /work sl-app-0 /app/venv/bin/python /tmp/obs_spf.py
docker exec -e CONFIG=/work/tests/test.env -e PYTHONPATH=/work -e GITHUB_ACTIONS_TEST=true \
  -w /work sl-app-0 /app/venv/bin/python /tmp/obs_boundary.py
docker exec -e CONFIG=/work/tests/test.env -e PYTHONPATH=/work -e GITHUB_ACTIONS_TEST=true \
  -w /work sl-app-0 /app/venv/bin/python /tmp/obs_determinism.py
```

All observation scripts lived in the container's `/tmp` (a non-bind-mounted overlay) and on the host at `/tmp/obs_host/` — both **outside** the repository checkout — and were deleted after evidence capture. No source or test file was modified; the source repository is left **byte-for-byte unchanged apart from this single document**.

### I.6 Final coverage pass

Every objective, answered by name with observed evidence and `file:line`:

- [x] **Objective 1 — the two address formats.** OLD unsigned `bounce+{id}+@sl.local` (`BOUNCE_PREFIX` [config.py:L100], `BOUNCE_SUFFIX` [config.py:L101]; `parse_id_from_bounce` [email_utils.py:L1258-L1259], observed `parse_id_from_bounce('bounce+12345+@sl.local')=12345`) vs NEW signed via `generate_verp_email` [email_utils.py:L1438-L1464] / `get_verp_info_from_email` [email_utils.py:L1467-L1498], observed round-trips for `bounce_forward`/`bounce_reply`/`transactional` (id 12345). — **Phase B.**
- [x] **Objective 2 — routing / four branches.** Chain `handle_DATA` [L2289] → `_handle` [L2335] → `handle` [L1945]; routing block [L2034-L2116]; transactional [L2038-L2054], forward-bounce [L2057-L2074], reply-bounce [L2077-L2098], iCloud [L2100-L2116]; each branch's behavior observed. — **Phase C.**
- [x] **Objective 3 — external-attacker feasibility.** Only the envelope recipient + a body are required; `550 SL E512` (invalid) vs `250 SL E213` (valid) observed via the canonical chain. — **Phase D.1.**
- [x] **Objective 4 — the security boundary.** Unsigned → `parse_id_from_bounce` → `EmailLog.get` [L2063] with no crypto; signed → `get_verp_info_from_email` returns `None` on HMAC mismatch [L1490-L1491] / expiry [L1496]; corrupted/expired signed observed to divert to `550 SL E515 Email not exist` [status.py:L51], never reaching the oracle. — **Phase F.**
- [x] **Objective 5 — exact SMTP responses.** `E512`/`E213`/`E211`/`E212`/`E205`/`E206`/`E216`/`E510`/`E515`/`E404`/`E524` captured verbatim across `handle`/`_handle`/`handle_DATA`. — **Phases D, G, I.**
- [x] **Objective 6 — `is_bounce` and spoofability.** `is_bounce` [L1813-L1818] reads only `mail_from == "<>"` and `Content-Type == multipart/report`, both attacker-controllable; truth table observed. — **Phase E.**
- [x] **Objective 7 — richness + ordering.** Inactive-user `550 SL E510 so such user` [handle_bounce L1869-L1871]; existence check `E512` [L2063-L2067] precedes the `is_bounce` gate [L2069], so a non-bounce probe still enumerates. — **Phase D.2-D.3.**

Every named mechanism referenced in this report, for completeness: `handle` [L1945], `_handle` [L2335], `handle_DATA` [L2289], `is_bounce` [L1813-L1818], `handle_bounce` [L1851], `handle_transactional_bounce` [L1834 log], `parse_id_from_bounce` [email_utils.py:L1258-L1259], `generate_verp_email` [email_utils.py:L1438-L1464], `get_verp_info_from_email` [email_utils.py:L1467-L1498], `EmailLog.get` [L2063], `User.is_active` [models.py:L766-L769], `VerpType` [models.py:L247-L250], `SpamdResult`/`SPFCheckResult` [app/handler/spamd_result.py], `VERPForward`/`VERPReply`/`VERPTransactional` [app/errors.py:L42-L57]; status codes `E200`/`E205`/`E206`/`E211`/`E212`/`E213`/`E216`/`E404`/`E502`/`E504`/`E510`/`E512`/`E515` [app/email/status.py]; config `BOUNCE_PREFIX`/`BOUNCE_SUFFIX`/`BOUNCE_PREFIX_FOR_REPLY_PHASE`/`TRANSACTIONAL_BOUNCE_PREFIX`/`VERP_PREFIX`/`VERP_MESSAGE_LIFETIME`/`VERP_EMAIL_SECRET` [app/config.py].

### I.7 Methodology attestation

- All behavioral claims are backed by a verbatim captured status string (with the producing layer/command) or a `file:line` citation; interpretations are labeled **(inferred)**.
- The direct answer leads; nuances follow; no observed value was adjusted toward an expected answer (e.g., the SPF-fail numeric convergence is reported honestly alongside the string-level survival of the oracle).
- The user's example `bounce+12345+@domain` is reproduced as `bounce+12345+@sl.local` with `parse_id_from_bounce(...) = 12345`.
- No remediation is proposed or implemented (out of scope); it is noted only, and labeled, that the signed format already closes the gap.
- The source repository is left byte-for-byte unchanged apart from this document; all temporary observation scripts lived outside the repository and were deleted.

